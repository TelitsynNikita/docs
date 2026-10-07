# 💾 Глава 11: Kubernetes — хранилища и Stateful-приложения

**Что вы узнаете:**
- Зачем нужны volumes в Kubernetes.
- Что такое PersistentVolume (PV) и PersistentVolumeClaim (PVC).
- Что такое StorageClass и как работает dynamic provisioning.
- Как работают StatefulSets и почему они нужны для баз данных.
- Что такое headless Service и как он связан со StatefulSet.
- Как делать бэкапы данных в Kubernetes.
- Как запустить PostgreSQL, Kafka, Redis в кластере.

**После прочтения вы сможете:**
- Объяснить разницу между emptyDir, hostPath, PV и PVC.
- Настроить StorageClass для dynamic provisioning.
- Запустить StatefulSet с persistent storage.
- Диагностировать проблемы с PVC в статусе Pending.
- Делать бэкапы и восстанавливать данные.
- Выбрать паттерн запуска базы данных в K8s.

---

## Содержание

- [11.0 Пролог: данные исчезли после перезапуска Pod'а](#110-пролог-данные-исчезли-после-перезапуска-podа)
- [11.1 Volumes: зачем нужны](#111-volumes-зачем-нужны)
- [11.2 emptyDir, hostPath, configMap, secret](#112-emptydir-hostpath-configmap-secret)
- [11.3 PersistentVolume и PersistentVolumeClaim](#113-persistentvolume-и-persistentvolumeclaim)
- [11.4 StorageClass: dynamic provisioning](#114-storageclass-dynamic-provisioning)
- [11.5 StatefulSet: stateful-приложения в K8s](#115-statefulset-stateful-приложения-в-k8s)
- [11.6 Headless Service и DNS для StatefulSet](#116-headless-service-и-dns-для-statefulset)
- [11.7 Паттерны запуска БД в Kubernetes](#117-паттерны-запуска-бд-в-kubernetes)
- [11.8 Бэкапы и восстановление](#118-бэкапы-и-восстановление)
- [11.9 Диагностика: PVC в Pending](#119-диагностика-pvc-в-pending)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 11.0 Пролог: данные исчезли после перезапуска Pod'а

Ты запустил PostgreSQL в Kubernetes:

```bash
kubectl create deployment postgres --image=postgres:16
```

Через неделю накопил данные. Пользователи работают, всё хорошо.

Внезапно Pod падает (например, из-за OOM). Kubernetes создаёт новый Pod. Ты подключаешься:

```bash
kubectl exec -it postgres-xxx -- psql -U postgres
# psql: error: connection to server failed
```

Или хуже — подключаешься, а **данных нет**:

```sql
SELECT * FROM users;
-- ERROR: relation "users" does not exist
```

**Данные исчезли.** Все таблицы, все записи.

**Почему?** Потому что данные были в **файловой системе Pod'а**. Когда Pod удалился — его файловая система (upperdir overlayfs, помнишь из Главы 3?) тоже удалилась.

**Это — фундаментальная проблема.** Pod'ы эфемерны. Их файловая система — тоже. Для stateful-приложений нужны **volumes**.

В этой главе мы разберём **как хранить данные в Kubernetes**. От простых volumes до PersistentVolumeClaims и StatefulSets.

Это — основа для запуска баз данных, брокеров, кэшей в кластере. Без этого Kubernetes — только для stateless-приложений.

---

## 11.1 Volumes: зачем нужны

### 🔌 Проблема: файловая система Pod'а эфемерна

Каждый Pod имеет **свою файловую систему** — overlayfs, собранный из слоёв образа + upperdir. Когда Pod удаляется:

- Upperdir удаляется.
- Все данные, записанные в файловую систему Pod'а, **теряются**.
- При пересоздании Pod'а — чистая файловая система из образа.

**Это — правильно для stateless-приложений.** Они не хранят состояние. Но **проблема для stateful**:

- PostgreSQL пишет в `/var/lib/postgresql/data`.
- Kafka пишет логи в `/var/lib/kafka/data`.
- Redis сохраняет дампы в `/data`.

Если эти данные в файловой системе Pod'а — они теряются.

**Решение:** volumes.

### 📦 Что такое volume

**Volume** — это директория, которая **не зависит от жизненного цикла Pod'а**. Volume **монтируется** в контейнеры Pod'а.

```
┌──────────────────────────────────────────┐
│              Pod                          │
│                                           │
│  ┌────────────────┐  ┌────────────────┐  │
│  │  Container 1   │  │  Container 2   │  │
│  │                │  │                │  │
│  │  /data ◄───────┼──┼──► /shared     │  │
│  └────────────────┘  └────────────────┘  │
│         ▲                                 │
│         │                                 │
│  ┌──────┴──────────────────────────────┐ │
│  │        Volume                        │ │
│  │  (emptyDir, hostPath, PVC, ...)      │ │
│  └──────────────────────────────────────┘ │
└──────────────────────────────────────────┘
```

**Ключевые свойства:**

- **Живёт дольше Pod'а** (если это persistent volume).
- **Может шариться между контейнерами** в Pod'е.
- **Монтируется в путь** внутри контейнера.

### 🎯 Типы volumes

Kubernetes поддерживает много типов volumes. Основные:

| Тип | Живёт | Где данные | Для чего |
|:---|:---|:---|:---|
| **emptyDir** | С Pod'ом | На ноде | Временные данные, кэш |
| **hostPath** | С нодой | На ноде | Доступ к файлам ноды (осторожно!) |
| **configMap** | С ConfigMap | В etcd | Конфигурация |
| **secret** | С Secret | В etcd | Секреты |
| **persistentVolumeClaim** | Дольше Pod'а | Зависит от PV | Stateful-данные |
| **nfs** | Внешний | NFS-сервер | Shared storage |
| **csi** | Внешний | Зависит от драйвера | Любое хранилище |

### 🎯 Монтирование volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: data
          mountPath: /app/data       # куда монтировать в контейнере
        - name: config
          mountPath: /app/config
          readOnly: true              # только для чтения
  volumes:
    - name: data
      emptyDir: {}
    - name: config
      configMap:
        name: app-config
```

**Что произошло:**

- Volume `data` (emptyDir) смонтирован в `/app/data`.
- Volume `config` (ConfigMap) смонтирован в `/app/config` (read-only).

**Важно:** volume описывается в `spec.volumes`, а монтирование — в `spec.containers[].volumeMounts`.

### 💡 Практика: что важно понять про volumes

**✅ ОБЯЗАТЕЛЬНО:**

1. **Volume живёт дольше Pod'а** (если это persistent volume).
2. **Volume описывается в `spec.volumes`, монтируется в `volumeMounts`.**
3. **Один volume можно смонтировать в несколько контейнеров** (shared data).

**👍 СТОИТ:**

4. **`readOnly: true`** для конфигов.
5. **`subPath`** для монтирования конкретного файла, а не директории.

**❌ НЕ ДЕЛАЙ:**

6. **Не храни состояние в файловой системе Pod'а.** Используй volume.
7. **Не используй hostPath для stateful.** Привязывает Pod к ноде.

### Где мы сейчас

Мы разобрали, зачем нужны volumes. Теперь — конкретные типы.

---

## 11.2 emptyDir, hostPath, configMap, secret

### 🔌 Проблема: какой volume выбрать

Kubernetes имеет много типов volumes. Разберём четыре основных:

- **emptyDir** — временное хранилище.
- **hostPath** — директория на ноде.
- **configMap** — конфигурация.
- **secret** — секреты.

### 📊 emptyDir

**emptyDir** — временная директория, которая создаётся при запуске Pod'а и удаляется при его удалении.

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: cache
          mountPath: /app/cache
  volumes:
    - name: cache
      emptyDir: {}
```

**Что произошло:**

- При запуске Pod'а Kubernetes создаёт директорию на ноде.
- Монтирует её в контейнер.
- При удалении Pod'а директория удаляется.

**Вариант с размером:**

```yaml
volumes:
  - name: cache
    emptyDir:
      sizeLimit: 1Gi
```

**Вариант в памяти:**

```yaml
volumes:
  - name: cache
    emptyDir:
      medium: Memory       # tmpfs, быстрее, но ест RAM
```

**Когда использовать:**

- **Кэш** (можно пересоздать).
- **Временные файлы** (не нужны после Pod'а).
- **Shared data между контейнерами Pod'а** (sidecar).

**НЕ использовать:**

- **Stateful-данные.** Потеряются с Pod'ом.

### 📊 hostPath

**hostPath** — монтирует директорию **с ноды** в Pod.

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: host-data
          mountPath: /host-data
  volumes:
    - name: host-data
      hostPath:
        path: /var/lib/myapp/data
        type: DirectoryOrCreate       # создать, если не существует
```

**Что произошло:**

- Директория `/var/lib/myapp/data` на ноде монтируется в Pod.
- Данные сохраняются между перезапусками Pod'а (пока Pod на этой ноде).

**Проблемы hostPath:**

- **Привязка к ноде.** Если Pod переедет на другую ноду — данных там не будет.
- **Безопасность.** Pod может читать/писать файлы ноды. Опасность при компрометации.
- **Multi-tenancy.** Pod'ы разных приложений могут конфликтовать.

**Когда использовать:**

- **Системные Pod'ы** (kubelet, CNI, monitoring agent).
- **Тестирование.**
- **Когда точно знаешь, что Pod останется на ноде** (с nodeSelector).

**НЕ использовать:**

- **Для stateful-приложений** в production. Используй PVC.
- **Для multi-tenant кластеров.**

### 📊 configMap

**configMap** — монтирует ConfigMap как файлы.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.yaml: |
    server:
      port: 8080
      timeout: 30s
  log-level: "info"
---
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: config
          mountPath: /etc/app
          readOnly: true
  volumes:
    - name: config
      configMap:
        name: app-config
```

**Что произошло:**

- Файлы из ConfigMap появляются в `/etc/app`:
  - `/etc/app/app.yaml`
  - `/etc/app/log-level`

**Монтирование отдельного ключа:**

```yaml
volumes:
  - name: config
    configMap:
      name: app-config
      items:
        - key: app.yaml
          path: config.yaml       # переименовать файл
```

**Когда использовать:**

- **Конфигурация приложений.**
- **Скрипты инициализации.**
- **Любые текстовые данные.**

**Важно:** ConfigMap **не обновляется** в Pod'е автоматически (с задержкой 1-2 минуты). Для мгновенного обновления нужен sidecar или reloader.

### 📊 secret

**secret** — как ConfigMap, но для секретов.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: c3VwZXItc2VjcmV0     # base64 от "super-secret"
  username: cG9zdGdyZXM=         # base64 от "postgres"
---
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: secret
          mountPath: /etc/secret
          readOnly: true
  volumes:
    - name: secret
      secret:
        secretName: db-secret
```

**Что произошло:**

- Файлы в `/etc/secret`:
  - `/etc/secret/password` (расшифрованный `super-secret`)
  - `/etc/secret/username`

**Типы Secret:**

| Тип | Для чего |
|:---|:---|
| `Opaque` | Обычные key-value |
| `kubernetes.io/dockerconfigjson` | Креденшелы к registry |
| `kubernetes.io/tls` | TLS-сертификаты |
| `kubernetes.io/service-account-token` | Токен ServiceAccount |

**Как создать:**

```bash
# Из файлов
kubectl create secret generic db-secret \
  --from-file=password=./password.txt \
  --from-file=username=./username.txt

# Из литералов
kubectl create secret generic db-secret \
  --from-literal=password=super-secret \
  --from-literal=username=postgres

# TLS
kubectl create secret tls my-tls \
  --cert=tls.crt --key=tls.key
```

**Важно:** Secret монтируется как **tmpfs** (в памяти), не пишется на диск.

### 🎯 subPath: монтирование одного файла

По умолчанию volume монтируется как директория. Если нужно смонтировать **один файл**:

```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - name: config
          mountPath: /etc/app/config.yaml    # файл
          subPath: app.yaml                  # какой файл из volume
  volumes:
    - name: config
      configMap:
        name: app-config
```

**Без `subPath`:** ConfigMap монтируется как директория `/etc/app/` со всеми файлами.

**С `subPath`:** монтируется **только** `app.yaml` как `/etc/app/config.yaml`.

**Когда использовать:** когда нужно смонтировать один файл, не перезаписывая директорию.

**Осторожно:** `subPath` **не обновляется** при изменении ConfigMap. Нужен перезапуск Pod'а.

### 🔬 Практика: emptyDir и ConfigMap

```bash
# 1. Pod с emptyDir
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-emptydir
spec:
  containers:
    - name: app
      image: busybox
      command: ["sleep", "3600"]
      volumeMounts:
        - name: cache
          mountPath: /cache
  volumes:
    - name: cache
      emptyDir: {}
EOF

# 2. Записать файл
kubectl exec test-emptydir -- sh -c "echo 'data' > /cache/file.txt"
kubectl exec test-emptydir -- cat /cache/file.txt
# data

# 3. Удалить Pod и создать новый (тот же манифест)
kubectl delete pod test-emptydir
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-emptydir
spec:
  containers:
    - name: app
      image: busybox
      command: ["sleep", "3600"]
      volumeMounts:
        - name: cache
          mountPath: /cache
  volumes:
    - name: cache
      emptyDir: {}
EOF

# 4. Данные исчезли
kubectl exec test-emptydir -- cat /cache/file.txt
# cat: /cache/file.txt: No such file or directory
```

**Что произошло:** emptyDir живёт с Pod'ом. При удалении Pod'а — удаляется.

### 💡 Практика: как выбирать volume

**✅ ОБЯЗАТЕЛЬНО:**

1. **emptyDir для кэша и временных данных.**
2. **ConfigMap для конфигурации.**
3. **Secret для секретов.**
4. **PersistentVolumeClaim для stateful.**

**👍 СТОИТ:**

5. **`medium: Memory` для быстрого кэша.**
6. **`subPath` для монтирования одного файла.**

**❌ НЕ ДЕЛАЙ:**

7. **Не используй hostPath для production stateful.** Используй PVC.
8. **Не храни секреты в ConfigMap.** Только в Secret.
9. **Не путай ConfigMap и Secret.** Секреты — для чувствительных данных.

### Где мы сейчас

Мы разобрали базовые volumes. Теперь — **PersistentVolume и PersistentVolumeClaim** — главное для stateful.

---

## 11.3 PersistentVolume и PersistentVolumeClaim

### 🔌 Проблема: данные должны жить дольше Pod'а

emptyDir умирает с Pod'ом. hostPath привязывает к ноде. Нужен способ хранить данные **независимо от Pod'а и ноды**.

**Решение:** PersistentVolume и PersistentVolumeClaim.

### 📊 Два объекта

**PersistentVolume (PV)** — это **физическое хранилище**:

- Создаётся администратором (или динамически).
- Имеет размер, тип доступа, StorageClass.
- Живёт независимо от Pod'ов.
- Может быть локальным диском, NFS, EBS, GCE PD, Ceph, и т.д.

**PersistentVolumeClaim (PVC)** — это **запрос** на хранилище:

- Создаётся разработчиком.
- Запрашивает размер, тип доступа, StorageClass.
- Kubernetes **связывает** PVC с подходящим PV.
- Pod использует PVC.

**Аналогия:**

- **PV** — это квартира.
- **PVC** — это заявка «хочу квартиру 2 комнаты в этом районе».
- **Binding** — это заселение.

```
┌─────────────────────────────────────────────────────┐
│            PV (PersistentVolume)                     │
│                                                      │
│  size: 10Gi                                          │
│  accessModes: [ReadWriteOnce]                        │
│  storageClassName: fast                              │
│  hostPath: /data/vol-1                               │
└────────────────────────┬────────────────────────────┘
                         │
                         │ bound to
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│            PVC (PersistentVolumeClaim)               │
│                                                      │
│  size: 10Gi                                          │
│  accessModes: [ReadWriteOnce]                        │
│  storageClassName: fast                              │
└────────────────────────┬────────────────────────────┘
                         │
                         │ mounted in
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│            Pod                                       │
│  volumeMounts:                                       │
│    - name: data                                      │
│      mountPath: /data                                │
│  volumes:                                            │
│    - name: data                                      │
│      persistentVolumeClaim:                          │
│        claimName: my-pvc                             │
└─────────────────────────────────────────────────────┘
```

### 🎯 PV: пример

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data/my-pv
```

**Поля:**

| Поле | Что означает |
|:---|:---|
| `capacity.storage` | Размер |
| `accessModes` | ReadWriteOnce, ReadOnlyMany, ReadWriteMany |
| `persistentVolumeReclaimPolicy` | Что делать после удаления PVC |
| `storageClassName` | Класс (для matching) |
| `hostPath` / `nfs` / `csi` | Тип хранилища |

### 🎯 PVC: пример

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: manual
```

### 🎯 Pod с PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-pvc
```

**Что произойдёт:**

1. PVC запрашивает 10Gi с `ReadWriteOnce`.
2. Kubernetes находит подходящий PV.
3. Связывает PVC с PV (binding).
4. Pod монтирует PVC в `/data`.
5. Данные в `/data` сохраняются даже после удаления Pod'а.

### 🎯 Access Modes

**Access Mode** определяет, как volume может быть смонтирован:

| Mode | Сокращение | Что означает |
|:---|:---|:---|
| **ReadWriteOnce** | RWO | Только один Pod на чтение и запись |
| **ReadOnlyMany** | ROX | Много Pod'ов на чтение |
| **ReadWriteMany** | RWX | Много Pod'ов на чтение и запись |
| **ReadWriteOncePod** | RWOP | Только **один** Pod (уникально) |

**Что выбрать:**

- **RWO** — для баз данных (PostgreSQL, MySQL).
- **ROX** — для статических файлов.
- **RWX** — для shared data (NFS, CephFS, EFS).
- **RWOP** — для случаев, когда нужна гарантия, что только один Pod.

**Важно:** **не все storage поддерживают все access modes**:

- **EBS (AWS)** — только RWO.
- **EFS (AWS)** — RWX.
- **NFS** — RWX.
- **Local SSD** — RWO.

### 🎯 Reclaim Policy

Что делать с PV после удаления PVC:

| Policy | Что происходит |
|:---|:---|
| **Retain** | PV остаётся, данные сохраняются. Нужно вручную удалить/переиспользовать. |
| **Delete** | PV удаляется, данные удаляются. |
| **Recycle** | Устарело. |

**Default:** зависит от StorageClass. Обычно **Delete**.

**Для баз данных:** обычно **Retain**, чтобы случайно не удалить данные.

### 🎯 Binding: как PV связывается с PVC

**Static binding:**

1. Администратор создаёт PV с `storageClassName: manual`, `capacity: 10Gi`.
2. Разработчик создаёт PVC с `storageClassName: manual`, `storage: 10Gi`.
3. Kubernetes находит подходящий PV и связывает.

**Правила matching:**

- StorageClass должна совпадать.
- Access modes должны совпадать (или PV поддерживает запрошенные).
- Размер PV ≥ запрошенного PVC.
- Если несколько подходящих PV — выбирается наименьший.

**Dynamic binding:**

См. подглаву 11.4 (StorageClass).

### 🎯 Жизненный цикл

```
1. Provisioning       — создание PV
   - Static: администратор создаёт PV вручную
   - Dynamic: StorageClass создаёт автоматически

2. Binding            — связывание PVC с PV

3. Using              — Pod использует PVC

4. Reclaiming         — что делать после удаления PVC
   - Retain
   - Delete
```

### 🔬 Практика: static PV и PVC

```bash
# 1. Создать PV
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /tmp/test-pv
EOF

# 2. Создать PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: manual
EOF

# 3. Проверить binding
kubectl get pv
# NAME      CAPACITY   ACCESS MODES   STATUS   CLAIM
# test-pv   1Gi        RWO            Bound    default/test-pvc

kubectl get pvc
# NAME      STATUS   VOLUME    CAPACITY
# test-pvc  Bound    test-pv   1Gi

# 4. Pod с PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sleep", "3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: test-pvc
EOF

# 5. Записать данные
kubectl exec test-pod -- sh -c "echo 'persistent data' > /data/file.txt"

# 6. Удалить Pod
kubectl delete pod test-pod

# 7. Создать снова
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sleep", "3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: test-pvc
EOF

# 8. Данные сохранились!
kubectl exec test-pod -- cat /data/file.txt
# persistent data
```

### 💡 Практика: как правильно работать с PV и PVC

**✅ ОБЯЗАТЕЛЬНО:**

1. **PVC создаёт разработчик.** PV — администратор (или StorageClass).
2. **Для БД — RWO.**
3. **Для production — `Retain`** reclaim policy, чтобы не удалить данные случайно.

**👍 СТОИТ:**

4. **Динамическое provisioning через StorageClass** (подглава 11.4).
5. **Разные StorageClass для разных нужд** (fast, slow, backup).

**❌ НЕ ДЕЛАЙ:**

6. **Не используй hostPath для production.** Используй CSI-драйверы.
7. **Не забывай про reclaim policy.** `Delete` — данные удалятся с PVC.

### Где мы сейчас

Мы разобрали static PV/PVC. Теперь — **StorageClass** — динамическое создание.

---

## 11.4 StorageClass: dynamic provisioning

### 🔌 Проблема: создавать PV вручную — долго

В static binding администратор должен **вручную** создавать PV для каждого PVC. Для большого кластера это не масштабируется.

**Решение:** StorageClass + dynamic provisioning.

### 📊 Что такое StorageClass

**StorageClass** — это **шаблон** для создания PV. Когда PVC создаётся, Kubernetes использует StorageClass, чтобы **автоматически** создать PV.

**Как работает:**

1. Администратор создаёт StorageClass (один раз).
2. Разработчик создаёт PVC с указанием StorageClass.
3. Kubernetes видит PVC, вызывает **provisioner** (плагин).
4. Provisioner создаёт PV (например, EBS volume в AWS).
5. PV связывается с PVC.

```
┌─────────────────────────────────────────────────────┐
│         StorageClass: fast                           │
│                                                      │
│  provisioner: ebs.csi.aws.com                        │
│  parameters:                                         │
│    type: gp3                                         │
│    iops: "3000"                                      │
└────────────────────────┬────────────────────────────┘
                         │
                         │ uses
                         ▼
┌─────────────────────────────────────────────────────┐
│         PVC                                        │
│                                                      │
│  storageClassName: fast                              │
│  size: 10Gi                                          │
└────────────────────────┬────────────────────────────┘
                         │
                         │ triggers provisioning
                         ▼
┌─────────────────────────────────────────────────────┐
│         PV (создан автоматически)                    │
│                                                      │
│  size: 10Gi                                          │
│  ebs: vol-abc123                                     │
└─────────────────────────────────────────────────────┘
```

### 🎯 Пример StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com         # AWS EBS
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete
allowVolumeExpansion: true           # можно увеличивать размер
volumeBindingMode: WaitForFirstConsumer
```

**Поля:**

| Поле | Что означает |
|:---|:---|
| `provisioner` | Какой драйвер использовать |
| `parameters` | Параметры драйвера (тип диска, IOPS) |
| `reclaimPolicy` | Delete или Retain |
| `allowVolumeExpansion` | Можно ли увеличивать PVC |
| `volumeBindingMode` | Immediate или WaitForFirstConsumer |

### 🎯 Volume Binding Mode

**Immediate:**

- PV создаётся **сразу** при создании PVC.
- Не учитывает, где будет Pod.

**WaitForFirstConsumer:**

- PV создаётся **при первом Pod'е**, который использует PVC.
- Учитывает **топологию**: PV будет в той же зоне, что Pod.
- **Рекомендуется** для облачных провайдеров (EBS привязан к зоне).

**Пример проблемы с Immediate:**

1. PVC создаётся в `us-west-2`.
2. PV создаётся в `us-west-2a`.
3. Pod планируется в `us-west-2b` (scheduler не знает о PV).
4. Pod не может смонтировать PV (не та зона).
5. Pod в Pending.

**WaitForFirstConsumer** решает: PV создаётся в зоне Pod'а.

### 🎯 Provisioner'ы

| Provisioner | Для чего |
|:---|:---|
| `ebs.csi.aws.com` | AWS EBS |
| `pd.csi.storage.gke.io` | GCP Persistent Disk |
| `disk.csi.azure.com` | Azure Disk |
| `rook-ceph.rbd.csi.ceph.com` | Ceph RBD |
| `driver.longhorn.io` | Longhorn |
| `local-path` | Локальные диски (для dev) |
| `nfs.csi.k8s.io` | NFS |

### 🎯 Default StorageClass

Можно пометить StorageClass как **default**. Тогда PVC без `storageClassName` будет использовать её.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
```

**Проверить:**

```bash
kubectl get storageclass
# NAME       PROVISIONER         RECLAIMPOLICY   VOLUMEBINDINGMODE
# standard   ebs.csi.aws.com     Delete          WaitForFirstConsumer   (default)
# fast       ebs.csi.aws.com     Delete          WaitForFirstConsumer
```

**PVC без storageClassName:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  # storageClassName не указан — используется default
```

### 🎯 Расширение PVC

Если `allowVolumeExpansion: true`, можно увеличить PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  resources:
    requests:
      storage: 20Gi        # было 10Gi — увеличили
```

**Что произойдёт:**

1. Kubernetes увеличит PV.
2. Файловая система расширится (иногда нужен restart Pod'а).

**Важно:** **уменьшить** PVC нельзя.

### 🔬 Практика: StorageClass и dynamic provisioning

```bash
# 1. Посмотреть StorageClass'ы
kubectl get storageclass
# NAME                 PROVISIONER
# standard (default)   k8s.io/minikube-hostpath    ← для Minikube
# или
# gp2 (default)        ebs.csi.aws.com             ← для EKS

# 2. Создать PVC без указания storageClassName (использует default)
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce  resources:
    requests:
      storage: 1Gi
EOF

# 3. Проверить — PV создан автоматически!
kubectl get pv
# NAME                                       CAPACITY   STATUS   CLAIM
# pvc-abc123-def456-...                      1Gi        Bound    default/dynamic-pvc

kubectl get pvc
# NAME          STATUS   VOLUME                                     CAPACITY
# dynamic-pvc   Bound    pvc-abc123-def456-...                      1Gi

# 4. Использовать в Pod
kubectl run test --image=busybox --restart=Never --overrides='
{
  "spec": {
    "containers": [{
      "name": "test",
      "image": "busybox",
      "command": ["sleep", "3600"],
      "volumeMounts": [{"name": "data", "mountPath": "/data"}]
    }],
    "volumes": [{
      "name": "data",
      "persistentVolumeClaim": {"claimName": "dynamic-pvc"}
    }]
  }
}'

# 5. Данные сохранятся
kubectl exec test -- sh -c "echo 'dynamic data' > /data/file.txt"
```

### 💡 Практика: как правильно работать с StorageClass

**✅ ОБЯЗАТЕЛЬНО:**

1. **Использовать dynamic provisioning** через StorageClass.
2. **`WaitForFirstConsumer`** для облачных провайдеров.
3. **Default StorageClass** в кластере.

**👍 СТОИТ:**

4. **Разные StorageClass для разных нужд:**
   - `fast` — SSD для БД.
   - `standard` — обычные диски.
   - `backup` — дешёвые диски для бэкапов.
5. **`allowVolumeExpansion: true`** для возможности увеличить.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `Immediate`** в облаках. Проблемы с топологией.
7. **Не забывай про reclaimPolicy.** `Delete` — данные удалятся.

### Где мы сейчас

Мы разобрали StorageClass. Теперь — **StatefulSet** — главный workload для stateful.

---

## 11.5 StatefulSet: stateful-приложения в K8s

### 🔌 Проблема: Deployment не подходит для БД

Deployment создаёт Pod'ы с **случайными именами** (`myapp-7d9f8c6b4d-abc12`). При пересоздании — новое имя, новый IP, новый PVC (если не привязан).

Для БД это не подходит:

- **Имя БД** (`postgres-0`) должно быть стабильным.
- **PVC** должен быть привязан к Pod'у (тот же диск при пересоздании).
- **Порядок запуска** важен (master первый, потом slaves).

**Решение:** StatefulSet.

### 📊 Что такое StatefulSet

**StatefulSet** — workload для **stateful-приложений**. Отличается от Deployment:

1. **Стабильные имена Pod'ов:** `app-0`, `app-1`, `app-2` (не случайные).
2. **Стабильные сетевые идентификаторы:** DNS `<pod>.<service>`.
3. **Стабильное хранилище:** `volumeClaimTemplates` создаёт PVC для каждого Pod'а.
4. **Порядок запуска и остановки:** 0, потом 1, потом 2.
5. **Порядок обновления:** обратный (2, 1, 0).

### 🎯 Пример StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres          # headless Service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:              # PVC для каждого Pod'а
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: fast
        resources:
          requests:
            storage: 100Gi
```

### 🎯 Что создаст StatefulSet

**Pod'ы:**

```
postgres-0
postgres-1
postgres-2
```

**PVC (по одному на Pod):**

```
data-postgres-0
data-postgres-1
data-postgres-2
```

**DNS (через headless Service):**

```
postgres-0.postgres.default.svc.cluster.local
postgres-1.postgres.default.svc.cluster.local
postgres-2.postgres.default.svc.cluster.local
```

**Порядок запуска:**

1. `postgres-0` запускается первым.
2. Ждёт, пока станет Ready.
3. `postgres-1` запускается.
4. Ждёт Ready.
5. `postgres-2` запускается.

**Порядок остановки:** обратный (2, 1, 0).

### 🎯 PVC при пересоздании Pod'а

**Ключевое отличие от Deployment:**

- **Deployment:** Pod пересоздаётся с новым PVC (если используется `volumeClaimTemplates` — вообще не используется).
- **StatefulSet:** Pod `postgres-0` **всегда** использует PVC `data-postgres-0`.

**Что это значит:**

- Если Pod `postgres-0` удалён (упал, evicted), новый Pod `postgres-0` получит **тот же** PVC.
- Данные сохраняются.

**При удалении StatefulSet:**

- Pod'ы удаляются.
- PVC **остаются** (по умолчанию). Их нужно удалять вручную.
- Это защита от случайной потери данных.

**Удалить PVC:**

```bash
kubectl delete pvc data-postgres-0 data-postgres-1 data-postgres-2
```

### 🎯 Headless Service

StatefulSet требует **headless Service** (`clusterIP: None`). Через него Pod'ы получают стабильные DNS-имена.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None            # headless
  selector:
    app: postgres
  ports:
    - port: 5432
```

**Что даёт:**

- DNS `postgres` → все IP Pod'ов.
- DNS `postgres-0.postgres` → IP Pod'а 0.
- DNS `postgres-1.postgres` → IP Pod'а 1.

Разберём подробно в подглаве 11.6.

### 🎯 Когда использовать StatefulSet

**✅ Использовать:**

- **Базы данных** (PostgreSQL, MySQL, MongoDB).
- **Брокеры** (Kafka, RabbitMQ, ZooKeeper).
- **Распределённые системы** (Elasticsearch, Cassandra).
- **Stateful-приложения**, где важен стабильный идентификатор.

**❌ Не использовать:**

- **Stateless-приложения** — Deployment.
- **Одноразовые задачи** — Job.
- **Daemon** — DaemonSet.

### 🎯 Deployment vs StatefulSet

| Характеристика | Deployment | StatefulSet |
|:---|:---|:---|
| Имена Pod'ов | Случайные | `app-0`, `app-1`, ... |
| Порядок запуска | Параллельно | По порядку |
| PVC | Общий (если есть) | Свой на Pod (`volumeClaimTemplates`) |
| DNS | Service | Headless Service + `<pod>.<svc>` |
| Обновление | Rolling (параллельно) | По одному (обратный порядок) |
| Для чего | Stateless | Stateful |

### 🔬 Практика: StatefulSet

```bash
# 1. Headless Service
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  clusterIP: None
  selector:
    app: nginx
  ports:
    - port: 80
EOF

# 2. StatefulSet
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx
spec:
  serviceName: nginx
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
EOF

# 3. Проверить
kubectl get pods -l app=nginx
# NAME      READY   STATUS
# nginx-0   1/1     Running
# nginx-1   1/1     Running
# nginx-2   1/1     Running

kubectl get pvc
# NAME           STATUS   VOLUME
# data-nginx-0   Bound    pvc-...
# data-nginx-1   Bound    pvc-...
# data-nginx-2   Bound    pvc-...

# 4. DNS
kubectl run -it --rm debug --image=nicolaka/netshoot --restart=Never -- bash
# Внутри:
nslookup nginx-0.nginx
# Address: 10.244.1.5
nslookup nginx-1.nginx
# Address: 10.244.2.5

# 5. Записать данные
kubectl exec nginx-0 -- sh -c "echo 'pod 0 data' > /data/file.txt"
kubectl exec nginx-1 -- sh -c "echo 'pod 1 data' > /data/file.txt"

# 6. Удалить Pod
kubectl delete pod nginx-0

# 7. Новый Pod получит тот же PVC
kubectl exec nginx-0 -- cat /data/file.txt
# pod 0 data  ← данные сохранились!
```

### 💡 Практика: как правильно работать со StatefulSet

**✅ ОБЯЗАТЕЛЬНО:**

1. **Headless Service** — обязателен для StatefulSet.
2. **`volumeClaimTemplates`** для PVC каждого Pod'а.
3. **`serviceName`** в StatefulSet — ссылка на headless Service.

**👍 СТОИТ:**

4. **Разные StorageClass для разных StatefulSet'ов.**
5. **Бэкапы PVC** (подглава 11.8).
6. **PodManagementPolicy: Parallel** для быстрого запуска (если порядок не важен).

**❌ НЕ ДЕЛАЙ:**

7. **Не используй StatefulSet для stateless.** Deployment лучше.
8. **Не удаляй PVC StatefulSet'а вручную без понимания.** Данные потеряются.
9. **Не масштабируй StatefulSet'ы БД без понимания репликации.** Нужны настройки кластера.

### Где мы сейчас

Мы разобрали StatefulSet. Теперь — **headless Service** подробнее.

---

## 11.6 Headless Service и DNS для StatefulSet

### 🔌 Проблема: как обратиться к конкретному Pod'у

В обычном Service клиент обращается к Service, и трафик идёт на **случайный** Pod. Но для БД нужно обращаться к **конкретному** Pod'у:

- К master'у PostgreSQL — для записи.
- К определённому broker'у Kafka — для партиции.

Как это сделать?

**Решение:** headless Service.

### 📊 Что такое headless Service

**Headless Service** — Service без ClusterIP (`clusterIP: None`).

**Что происходит:**

- Нет виртуального IP.
- DNS возвращает **IP всех Pod'ов**, а не один ClusterIP.
- Клиент сам выбирает, к какому Pod'у идти.

**DNS-записи:**

```
<service>.<namespace>.svc.cluster.local       → все Pod'ы (A-записи)
<pod>.<service>.<namespace>.svc.cluster.local → конкретный Pod
```

### 🎯 Пример

**Headless Service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
```

**StatefulSet:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  ...
```

**DNS-записи:**

| Запрос | Ответ |
|:---|:---|
| `postgres` | IP всех 3 Pod'ов (A-записи) |
| `postgres-0.postgres` | IP Pod'а postgres-0 |
| `postgres-1.postgres` | IP Pod'а postgres-1 |
| `postgres-2.postgres` | IP Pod'а postgres-2 |

### 🎯 Использование

**Клиент обращается к конкретному Pod'у:**

```bash
# Подключение к master'у (postgres-0)
psql -h postgres-0.postgres.default.svc.cluster.local -U postgres

# Подключение к slave'у (postgres-1)
psql -h postgres-1.postgres.default.svc.cluster.local -U postgres
```

**Peer discovery для Kafka:**

```properties
bootstrap.servers=kafka-0.kafka:9092,kafka-1.kafka:9092,kafka-2.kafka:9092
```

**PostgreSQL replication:**

```
primary_conninfo = 'host=postgres-0.postgres port=5432'
```

### 🎯 SRV-записи

Для headless Service с **именованными портами** DNS возвращает **SRV-записи**:

```yaml
spec:
  ports:
    - name: postgres
      port: 5432
```

**DNS-запрос SRV:**

```bash
dig SRV _postgres._tcp.postgres.default.svc.cluster.local
# Ответ:
# _postgres._tcp.postgres.default.svc.cluster.local. 5 IN SRV 0 50 5432 postgres-0.postgres.default.svc.cluster.local.
# _postgres._tcp.postgres.default.svc.cluster.local. 5 IN SRV 0 50 5432 postgres-1.postgres.default.svc.cluster.local.
```

**Что это даёт:** клиент может обнаружить все Pod'ы и их порты автоматически.

### 🎯 Обычный Service vs Headless

**Обычный Service:**

- Клиент → ClusterIP → случайный Pod.
- Балансировка — на уровне kube-proxy.
- DNS возвращает **один IP** (ClusterIP).

**Headless Service:**

- Клиент → DNS → все Pod'ы → выбирает сам.
- Балансировка — на стороне клиента.
- DNS возвращает **все IP** Pod'ов.

**Когда использовать headless:**

- **StatefulSet** — для прямого доступа к Pod'ам.
- **Peer discovery** — клиенты сами находят друг друга.
- **Кастомная балансировка** — клиент решает, куда идти.

### 🔬 Практика: headless DNS

```bash
# 1. Headless Service + StatefulSet
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: demo
spec:
  clusterIP: None
  selector:
    app: demo
  ports:
    - name: http
      port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: demo
spec:
  serviceName: demo
  replicas: 3
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - name: http
              containerPort: 80
EOF

# 2. DNS-запросы
kubectl run -it --rm debug --image=nicolaka/netshoot --restart=Never -- bash
# Внутри:

# Все Pod'ы
dig +short demo.default.svc.cluster.local
# 10.244.1.5
# 10.244.2.5
# 10.244.3.5

# Конкретный Pod
dig +short demo-0.demo.default.svc.cluster.local
# 10.244.1.5

dig +short demo-1.demo.default.svc.cluster.local
# 10.244.2.5

# SRV-запись
dig SRV _http._tcp.demo.default.svc.cluster.local
```

### 💡 Практика: как правильно работать с headless

**✅ ОБЯЗАТЕЛЬНО:**

1. **Headless Service для StatefulSet.**
2. **Именованные порты для SRV-записей.**
3. **Клиент сам выбирает Pod.**

**👍 СТОИТ:**

4. **Использовать для peer discovery** (Kafka, Cassandra, Elasticsearch).

**❌ НЕ ДЕЛАЙ:**

5. **Не используй headless для stateless.** Обычный Service проще.
6. **Не полагайся на порядок IP** в DNS-ответе.

### Где мы сейчас

Мы разобрали headless Service. Теперь — **паттерны запуска БД в K8s**.

---

## 11.7 Паттерны запуска БД в Kubernetes

### 🔌 Проблема: как запускать БД в K8s

Запустить PostgreSQL в Kubernetes — просто. Запустить **production-ready** PostgreSQL — сложно:

- **Replication** — master + slaves.
- **Failover** — если master упал, slave становится master.
- **Backups** — регулярные бэкапы.
- **Monitoring** — метрики, алерты.
- **Upgrades** — обновления без простоя.

Существует несколько паттернов.

### 🎯 Паттерн 1: Не запускать БД в K8s

**Самый честный вариант.** Использовать **managed БД**:

- **AWS RDS / Aurora** (PostgreSQL, MySQL).
- **GCP Cloud SQL**.
- **Azure Database**.
- **MongoDB Atlas**.
- **ElastiCache** (Redis).

**Плюсы:**

- **Не нужно управлять.** Провайдер делает backup, failover, upgrades.
- **Надёжнее.** Managed БД оптимизированы.
- **Дешевле в сумме.** Не тратишь время команды на управление.

**Минусы:**

- **Дороже по счёту.**
- **Vendor lock-in.**
- **Меньше контроля.**

**Когда использовать:**

- **Компании без dedicated DBA.**
- **Когда надёжность важнее контроля.**
- **Для небольших команд.**

**Для 90% случаев — это лучший выбор.**

### 🎯 Паттерн 2: Kubernetes Operator

**Operator** — это расширение Kubernetes, которое знает, как управлять конкретным приложением.

**Что делает:**

- Watch на Custom Resource (CR).
- Автоматически создаёт StatefulSet, Service, ConfigMap.
- Управляет replication, failover, backups.
- Реагирует на события (Pod упал → promotion slave).

**Популярные операторы:**

| Оператор | Для чего |
|:---|:---|
| **Zalando Postgres Operator** | PostgreSQL |
| **Crunchy Data PGO** | PostgreSQL |
| **Percona Operator** | PostgreSQL, MySQL, MongoDB |
| **Strimzi** | Kafka |
| **Redis Operator** | Redis |
| **Elastic Cloud on K8s (ECK)** | Elasticsearch |

**Пример Zalando Postgres Operator:**

```yaml
apiVersion: acid.zalan.do/v1
kind: postgresql
metadata:
  name: my-postgres
spec:
  teamId: myteam
  volume:
    size: 100Gi
    storageClass: fast
  numberOfInstances: 3
  users:
    myapp:
      - superuser
      - createdb
  databases:
    mydb: myapp
  postgresql:
    version: "16"
    parameters:
      max_connections: "200"
      shared_buffers: "2GB"
```

**Что произойдёт:**

1. Operator создаст StatefulSet с 3 Pod'ами.
2. Настроит replication.
3. Создаст users и databases.
4. Будет делать backups.
5. Обработает failover.

**Плюсы:**

- **Автоматизация.** Всё управляется через CR.
- **Production-ready.** Оператор знает best practices.
- **Failover автоматический.**

**Минусы:**

- **Сложность.** Нужно изучить оператор.
- **Ограничения.** Только то, что оператор умеет.
- **Debugging сложнее.**

**Когда использовать:**

- **Компания готова инвестировать в K8s-нативную БД.**
- **Нужен полный контроль.**
- **Есть dedicated DBA или опыт.**

### 🎯 Паттерн 3: Ручной StatefulSet

**Самый сложный.** Создать StatefulSet + настроить replication сам.

**Что нужно:**

- StatefulSet с PVC.
- Init-контейнеры для настройки replication.
- Sidecar для health checks.
- Скрипты для failover.
- Backup-скрипты.

**Плюсы:**

- **Полный контроль.**
- **Нет зависимости от оператора.**

**Минусы:**

- **Очень сложно.**
- **Легко ошибиться.**
- **Failover вручную.**

**Когда использовать:** почти никогда. Только если есть очень специфичные требования.

### 📊 Сравнение паттернов

| Паттерн | Сложность | Надёжность | Стоимость | Для кого |
|:---|:---|:---|:---|:---|
| **Managed БД** | Низкая | Высокая | Высокая ($) | Большинство |
| **Operator** | Средняя | Высокая | Средняя | Средние команды |
| **Ручной StatefulSet** | Очень высокая | Зависит | Низкая | Эксперты |

### 🎯 Рекомендация

**Правило:**

1. **Если можно — managed БД.** RDS, Cloud SQL, Atlas. Не изобретай.
2. **Если нельзя (on-premise, compliance) — Operator.** Zalando, Percona, Crunchy.
3. **Ручной StatefulSet — только если очень специфичные требования.**

**Для обучения** — начни с managed. Потом разберись с Operator.

### 🔬 Практика: PostgreSQL с Operator

**Установка Zalando Postgres Operator:**

```bash
# 1. Установить оператор
kubectl apply -k github.com/zalando/postgres-operator/manifests

# 2. Проверить
kubectl get pods -n postgres-operator
# NAME                                 READY   STATUS
# postgres-operator-xxxxxxxxx-xxxxx    1/1     Running

# 3. Создать PostgreSQL
kubectl apply -f - <<EOF
apiVersion: acid.zalan.do/v1
kind: postgresql
metadata:
  name: my-postgres
spec:
  teamId: myteam
  volume:
    size: 1Gi
  numberOfInstances: 2
  users:
    myapp:
      - superuser
      - createdb
  databases:
    mydb: myapp
  postgresql:
    version: "16"
EOF

# 4. Проверить
kubectl get postgresql
# NAME          TEAM     VERSION   PODS   VOLUME   CPU-REQUEST   MEMORY-REQUEST   AGE
# my-postgres   myteam   16        2      1Gi                                   30s

kubectl get pods -l application=spilo
# NAME                READY   STATUS
# my-postgres-0       1/1     Running
# my-postgres-1       1/1     Running

# 5. Подключиться
kubectl exec -it my-postgres-0 -- psql -U postgres
```

**Что произошло:**

- Operator создал StatefulSet с 2 Pod'ами.
- Настроил replication (master + slave).
- Создал пользователя `myapp` и БД `mydb`.
- Готов к failover.

### 💡 Практика: как правильно запускать БД

**✅ ОБЯЗАТЕЛЬНО:**

1. **Managed БД для production** (если возможно).
2. **Operator для K8s-native** (если managed нельзя).
3. **Backups** — всегда.
4. **Monitoring** — всегда.

**👍 СТОИТ:**

5. **Отдельный namespace для БД.**
6. **Network Policies** для изоляции.
7. **ResourceQuota** для ограничения.

**❌ НЕ ДЕЛАЙ:**

8. **Не запускай БД в Deployment.** Только StatefulSet или Operator.
9. **Не используй одну БД для всех окружений.**
10. **Не забывай про бэкапы.** Данные — самое дорогое.

### Где мы сейчас

Мы разобрали паттерны запуска БД. Теперь — **бэкапы**.

---

## 11.8 Бэкапы и восстановление

### 🔌 Проблема: данные можно потерять

Даже в Kubernetes данные можно потерять:

- **PVC удалён** по ошибке.
- **StorageClass с `reclaimPolicy: Delete`** — данные удаляются при удалении PVC.
- **Приложение испортило данные** (логическая ошибка).
- **Ransomware** зашифровал данные.
- **Кластер потерян** (disaster recovery).

**Решение:** регулярные бэкапы.

### 📊 Что бэкапить

**1. Данные приложений (PVC).**

**2. Конфигурация Kubernetes:**

- YAML-манифесты (Deployments, Services, ConfigMaps).
- Secrets.
- CRD.

**3. etcd** (для disaster recovery всего кластера).

### 🎯 Velero: стандарт для бэкапов K8s

**Velero** — инструмент для бэкапа и восстановления Kubernetes-кластера.

**Что делает:**

- Бэкап ресурсов (Deployments, Services, ConfigMaps, PVC).
- Восстановление в тот же или другой кластер.
- Snapshot volumes (если storage поддерживает).
- Миграция между кластерами.

**Установка:**

```bash
# 1. Скачать CLI
wget https://github.com/vmware-tanzu/velero/releases/download/v1.14.0/velero-v1.14.0-linux-amd64.tar.gz
tar -xvf velero-v1.14.0-linux-amd64.tar.gz
mv velero-v1.14.0-linux-amd64/velero /usr/local/bin/

# 2. Установить в кластер (с S3 backend)
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.10.0 \
  --bucket my-backups \
  --backup-location-config region=us-west-2 \
  --snapshot-location-config region=us-west-2 \
  --secret-file ./credentials-velero
```

**Что произошло:**

- Velero установлен в namespace `velero`.
- Бэкапы будут храниться в S3 bucket `my-backups`.

### 🎯 Создание бэкапа

**Разовый бэкап:**

```bash
# Бэкап всего namespace
velero backup create my-backup --include-namespaces production

# Бэкап конкретных ресурсов
velero backup create my-backup --include-resources deployments,services --include-namespaces production

# Бэкап с селектором
velero backup create my-backup --selector app=myapp
```

**Расписание:**

```bash
# Ежедневно в 2 ночи
velero schedule create daily-backup --schedule="0 2 * * *" --include-namespaces production

# Каждую неделю
velero schedule create weekly-backup --schedule="0 2 * * 0" --include-namespaces production

# Хранить 30 дней
velero schedule create daily-backup --schedule="0 2 * * *" --ttl 720h
```

### 🎯 Восстановление

```bash
# Посмотреть бэкапы
velero backup get
# NAME          STATUS      CREATED                         EXPIRES
# my-backup     Completed   2026-01-15 02:00:00 +0000 UTC   29d

# Восстановить
velero restore create --from-backup my-backup

# Восстановить в другой namespace
velero restore create --from-backup my-backup --namespace-mappings production:production-restored

# Посмотреть восстановления
velero restore get
```

### 🎯 Что бэкапит Velero

**По умолчанию:**

- Все ресурсы Kubernetes в namespace (кроме некоторых системных).
- PVC — snapshot через CSI-драйвер (если поддерживается).
- Если snapshot не поддерживается — использует **Restic/Kopia** для file-level бэкапа.

**Не бэкапит по умолчанию:**

- Secrets (можно включить).
- CRD definitions.
- Cluster-scoped resources.

### 🎯 Бэкап БД

**Отдельно от Velero** важно делать бэкапы **данных БД**. Velero бэкапит PVC (snapshot), но это **crash-consistent**, не **application-consistent**.

**Для PostgreSQL:**

```bash
# Логический бэкап (pg_dump)
kubectl exec my-postgres-0 -- pg_dump -U postgres mydb > backup.sql

# Или в S3
kubectl exec my-postgres-0 -- sh -c "pg_dump -U postgres mydb | gzip | aws s3 cp - s3://my-backups/db-\$(date +%Y%m%d).sql.gz"
```

**Для production — использовать инструменты:**

- **pgBackRest** — инкрементальные бэкапы, PITR.
- **Barman** — backup and recovery manager.
- **WAL-G** — WAL archiving.

**Operator'ы** (Zalando, Crunchy) имеют встроенные бэкапы.

### 🎯 Тестирование бэкапов

**Ключевое правило:** **бэкап, который не тестировали, — это не бэкап.**

**Что тестировать:**

1. **Восстановление работает.**
2. **Данные целостны.**
3. **Время восстановления приемлемо** (RTO).
4. **Потери данных приемлемы** (RPO).

**Тестирование:**

- **Раз в месяц** — восстановление в тестовый кластер.
- **Раз в квартал** — полный disaster recovery drill.
- **После изменения схемы БД** — проверка, что бэкапы работают.

### 🎯 RTO и RPO

**RTO (Recovery Time Objective)** — сколько времени можно восстанавливать.

**RPO (Recovery Point Objective)** — сколько данных можно потерять.

**Пример:**

- RTO = 1 час — восстановление должно занять ≤ 1 час.
- RPO = 15 минут — можно потерять данные за последние 15 минут.

**Как достичь:**

- **Частые бэкапы** → меньше RPO.
- **Быстрое восстановление** → меньше RTO.
- **Replication + failover** → почти 0 RTO/RPO.

### 🔬 Практика: Velero

```bash
# 1. Установить Velero (упрощённо, для теста — локальный minio)
# В production — S3, GCS, Azure Blob

# 2. Создать бэкап
velero backup create test-backup --include-namespaces default

# 3. Посмотреть
velero backup get
velero backup describe test-backup

# 4. Симулировать проблему
kubectl delete deployment myapp

# 5. Восстановить
velero restore create --from-backup test-backup

# 6. Проверить
kubectl get deployment myapp
```

### 💡 Практика: как правильно делать бэкапы

**✅ ОБЯЗАТЕЛЬНО:**

1. **Velero для ресурсов K8s.**
2. **pg_dump / pgBackRest для БД.**
3. **Тестировать восстановление.** Бэкап без теста — не бэкап.
4. **Хранить бэкапы вне кластера** (S3, другой регион).

**👍 СТОИТ:**

5. **Разные retention policies** — ежедневные 30 дней, еженедельные 1 год.
6. **PITR (Point-in-Time Recovery)** для БД через WAL archiving.
7. **Мониторинг бэкапов** — алерты, если бэкап не прошёл.

**❌ НЕ ДЕЛАЙ:**

8. **Не храни бэкапы только в том же кластере.** Кластер потерян — бэкапы тоже.
9. **Не забывай тестировать.** «Бэкап есть» ≠ «восстановление работает».
10. **Не делай бэкап БД через snapshot PVC.** Нужен application-consistent backup.

### Где мы сейчас

Мы разобрали бэкапы. Теперь — **диагностика PVC в Pending**.

---

## 11.9 Диагностика: PVC в Pending

### 🔌 Проблема: PVC не связывается с PV

Ты создал PVC. Проверяешь:

```bash
kubectl get pvc
# NAME      STATUS    VOLUME   CAPACITY
# my-pvc    Pending
```

**PVC в статусе Pending.** Pod, который его использует, тоже в Pending.

**Почему?** Разберём типичные причины.

### 🔍 Причины Pending

**1. Нет подходящего PV (static binding).**

**Симптом:**

```bash
kubectl describe pvc my-pvc
# Events:
#   Warning  ProvisioningFailed  no persistent volumes available for this claim
```

**Причины:**

- Нет PV с нужным `storageClassName`.
- Нет PV с нужным размером.
- Нет PV с нужным `accessModes`.

**Решение:**

- Создать PV вручную.
- Использовать StorageClass с dynamic provisioning.

**2. StorageClass не существует.**

**Симптом:**

```bash
kubectl describe pvc my-pvc
# Events:
#   Warning  ProvisioningFailed  storageclass.storage.k8s.io "fast" not found
```

**Решение:**

- Проверить, что StorageClass существует: `kubectl get storageclass`.
- Исправить `storageClassName` в PVC.

**3. Provisioner не работает.**

**Симптом:**

```bash
kubectl describe pvc my-pvc
# Events:
#   Warning  ProvisioningFailed  failed to provision volume: rpc error: ...
```

**Причины:**

- CSI-драйвер не установлен.
- Проблемы с облачным провайдером (IAM, квоты).
- Проблемы с сетью.

**Решение:**

- Проверить Pod'ы CSI-драйвера: `kubectl get pods -n kube-system | grep csi`.
- Проверить логи CSI: `kubectl logs -n kube-system <csi-pod>`.
- Проверить IAM-роли.

**4. WaitForFirstConsumer и Pod не создан.**

**Симптом:**

```bash
kubectl describe pvc my-pvc
# Events:
#   Normal  WaitForFirstConsumer  waiting for first consumer to be created before binding
```

**Что означает:** StorageClass использует `WaitForFirstConsumer`, PV создастся, когда появится Pod.

**Решение:** создать Pod, который использует этот PVC. Или изменить `volumeBindingMode` на `Immediate` (если поддерживается).

**5. Несовместимость accessModes.**

**Симптом:**

```bash
kubectl describe pvc my-pvc
# Events:
#   Warning  ProvisioningFailed  access mode is not supported
```

**Пример:** EBS поддерживает только `ReadWriteOnce`, а PVC запрашивает `ReadWriteMany`.

**Решение:** использовать правильный accessMode или другой StorageClass.

**6. PVC уже связан.**

**Симптом:**

```bash
kubectl get pvc
# NAME     STATUS   VOLUME
# my-pvc   Bound    pvc-abc123
```

**Если статус Bound — всё хорошо.**

**7. PV в статусе Released.**

**Симптом:**

```bash
kubectl get pv
# NAME      STATUS     CLAIM
# my-pv     Released   default/old-pvc
```

**Что означает:** PV был связан с PVC, PVC удалён. Reclaim policy — Retain. PV остался, но связан со старым PVC.

**Решение:**

- Удалить PV и создать заново.
- Или вручную очистить claim: `kubectl patch pv my-pv -p '{"spec":{"claimRef":null}}'`.

### 🔍 Алгоритм диагностики

**Шаг 1: Статус PVC**

```bash
kubectl get pvc
kubectl describe pvc my-pvc
```

**Шаг 2: События**

Смотри **Events** в `describe`. Они точно говорят причину.

**Шаг 3: StorageClass**

```bash
kubectl get storageclass
# Есть ли нужный?
# Какой provisioner?
# Default?
```

**Шаг 4: PV**

```bash
kubectl get pv
# Есть ли подходящий?
# Bound или Available?
```

**Шаг 5: CSI-драйвер**

```bash
kubectl get pods -n kube-system | grep csi
# Работают ли Pod'ы?
```

**Шаг 6: Логи CSI**

```bash
kubectl logs -n kube-system <csi-controller-pod>
# Ошибки?
```

**Шаг 7: Pod**

Если `WaitForFirstConsumer`:

```bash
kubectl get pods
# Есть ли Pod, использующий PVC?
```

### 🎯 Расширенная диагностика

**Проверить IAM (для облака):**

```bash
# Для EKS — проверить, что ServiceAccount CSI имеет IAM-роль
kubectl get sa -n kube-system ebs-csi-controller-sa -o yaml
# Есть ли annotation с role ARN?
```

**Проверить квоты:**

```bash
# Для облака — проверить квоты
aws service-quotas get-service-quota --service-code ebs --quota-code L-D18FCD1D
```

**Проверить топологию:**

```bash
# Если WaitForFirstConsumer — Pod в зоне, где есть storage
kubectl get pod my-pod -o jsonpath='{.spec.nodeName}'
kubectl describe node <node> | grep topology
```

### 🔬 Практика: диагностика

```bash
# 1. Создать PVC с несуществующим StorageClass
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: nonexistent
EOF

# 2. Проверить
kubectl get pvc test-pvc
# Pending

kubectl describe pvc test-pvc
# Events:
#   Warning  ProvisioningFailed  storageclass.storage.k8s.io "nonexistent" not found

# 3. Исправить
kubectl delete pvc test-pvc
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  # storageClassName не указан — используется default
EOF

# 4. Проверить
kubectl get pvc test-pvc
# Bound
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **`kubectl describe pvc`** — Events внизу.
2. **Проверить StorageClass.**
3. **Проверить Pod'ы CSI-драйвера.**
4. **Логи CSI-контроллера.**

**👍 СТОИТ:**

5. **Проверить IAM-роли** (для облака).
6. **Проверить квоты.**
7. **Проверить `volumeBindingMode`.**

**❌ НЕ ДЕЛАЙ:**

8. **Не пересоздавай PVC без понимания.** Может стать хуже.
9. **Не игнорируй `WaitForFirstConsumer`.** Это нормально, не баг.

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **Volume** | Директория, монтируемая в Pod. |
| **emptyDir** | Временный volume, живёт с Pod'ом. |
| **hostPath** | Директория на ноде. |
| **configMap** | Volume с содержимым ConfigMap. |
| **secret** | Volume с содержимым Secret. |
| **persistentVolumeClaim** | Запрос на persistent storage. |
| **PV (PersistentVolume)** | Физическое хранилище. |
| **PVC (PersistentVolumeClaim)** | Запрос на хранилище. |
| **StorageClass** | Шаблон для создания PV. |
| **Dynamic provisioning** | Автоматическое создание PV. |
| **Access modes** | RWO, ROX, RWX, RWOP. |
| **Reclaim policy** | Retain, Delete, Recycle. |
| **Volume binding mode** | Immediate или WaitForFirstConsumer. |
| **CSI** | Container Storage Interface — стандарт для storage-драйверов. |
| **StatefulSet** | Workload для stateful-приложений. |
| **volumeClaimTemplates** | Шаблон PVC для StatefulSet. |
| **Headless Service** | Service без ClusterIP. |
| **Operator** | Расширение K8s для управления приложением. |
| **CRD** | Custom Resource Definition. |
| **Velero** | Инструмент для бэкапа K8s. |
| **RTO** | Recovery Time Objective — время восстановления. |
| **RPO** | Recovery Point Objective — потеря данных. |
| **PITR** | Point-in-Time Recovery. |

---

## Что мы узнали?

- **Volumes** живут дольше Pod'а. Типы: emptyDir, hostPath, configMap, secret, PVC.
- **PV** — физическое хранилище. **PVC** — запрос на хранилище. **Binding** — связывание.
- **StorageClass** — шаблон для dynamic provisioning. `WaitForFirstConsumer` для облаков.
- **StatefulSet** — workload для stateful. Стабильные имена, свои PVC, порядок запуска.
- **Headless Service** — DNS для конкретных Pod'ов StatefulSet.
- **Паттерны БД:** managed (RDS), Operator (Zalando), ручной StatefulSet.
- **Backups:** Velero для K8s, pg_dump/pgBackRest для БД. Тестировать восстановление.
- **Диагностика PVC Pending:** `kubectl describe pvc`, StorageClass, CSI-драйвер.

---

## Типичные ошибки

- ❌ **Хранить данные в файловой системе Pod'а.** Потеряются при пересоздании.
- ❌ **Использовать emptyDir для stateful.** Живёт с Pod'ом.
- ❌ **Использовать hostPath в production.** Привязка к ноде.
- ❌ **`reclaimPolicy: Delete` для БД.** Данные удалятся с PVC.
- ❌ **StorageClass без `WaitForFirstConsumer`** в облаке.
- ❌ **Использовать Deployment для БД.** StatefulSet нужен.
- ❌ **Забывать про headless Service для StatefulSet.**
- ❌ **Не делать бэкапы БД.** Данные — самое дорогое.
- ❌ **Не тестировать восстановление.**
- ❌ **Хранить бэкапы только в кластере.**
- ❌ **Игнорировать PVC в Pending.** Pod тоже не запустится.

---

## Для быстрого повторения

- **Volumes:** emptyDir (временный), hostPath (нода), configMap/secret, PVC (persistent).
- **PV/PVC:** PV — хранилище, PVC — запрос. Binding связывает.
- **StorageClass:** шаблон для dynamic PV. `WaitForFirstConsumer` для облаков.
- **Access modes:** RWO (один Pod), ROX (много на чтение), RWX (много на запись).
- **StatefulSet:** `postgres-0`, `postgres-1`, ..., свои PVC, порядок запуска.
- **Headless Service:** `clusterIP: None`. DNS `<pod>.<svc>`.
- **Operator:** Zalando Postgres, Strimzi Kafka, Redis Operator.
- **Backups:** Velero + pg_dump. Тестировать.
- **Диагностика PVC:** `kubectl describe pvc`, Events, StorageClass, CSI logs.

---

## Вопросы для самопроверки

1. Чем volume отличается от файловой системы Pod'а?
2. Назови типы volumes. Когда использовать каждый?
3. Что такое PV и PVC? Чем отличаются?
4. Что такое StorageClass? Зачем нужен?
5. Что такое `WaitForFirstConsumer`? Зачем нужен?
6. Чем StatefulSet отличается от Deployment?
7. Что такое headless Service? Как связан с StatefulSet?
8. Как запустить PostgreSQL в Kubernetes? Три паттерна.
9. Что такое Operator? Зачем нужен?
10. Как делать бэкапы в K8s? Что использовать?
11. Что такое RTO и RPO?
12. PVC в Pending. Как диагностировать?
13. Что произойдёт с PVC при удалении StatefulSet?
14. Можно ли увеличить PVC? Как?
15. Ты хочешь запустить Kafka в K8s. Какой паттерн выбрать?

---

## Ответы

**1. Volume vs файловая система Pod'а**

Volume живёт дольше Pod'а (если persistent). Файловая система Pod'а удаляется вместе с Pod'ом.

**2. Типы volumes**

- **emptyDir** — временный, для кэша.
- **hostPath** — директория ноды, для системных Pod'ов.
- **configMap** — конфигурация.
- **secret** — секреты.
- **PVC** — persistent storage, для stateful.

**3. PV vs PVC**

PV — физическое хранилище, создаётся админом. PVC — запрос на хранилище, создаётся разработчиком. Kubernetes связывает PVC с PV.

**4. StorageClass**

Шаблон для dynamic provisioning. Указывает provisioner, параметры, reclaim policy.

**5. WaitForFirstConsumer**

Volume binding mode. PV создаётся, когда появляется Pod (учитывает топологию). Решает проблему, когда PV в одной зоне, а Pod — в другой.

**6. StatefulSet vs Deployment**

StatefulSet: стабильные имена (`app-0`), свои PVC на Pod, порядок запуска, headless Service. Deployment: случайные имена, общий PVC, параллельный запуск.

**7. Headless Service**

Service с `clusterIP: None`. DNS возвращает все IP Pod'ов. Для StatefulSet — `<pod>.<svc>.<ns>.svc.cluster.local`.

**8. Паттерны БД**

1. **Managed БД** (RDS, Cloud SQL) — для большинства.
2. **Operator** (Zalando, Crunchy) — для K8s-native.
3. **Ручной StatefulSet** — для экспертов.

**9. Operator**

Расширение K8s. Watch на CR, автоматически управляет StatefulSet, Service, replication, failover, backups.

**10. Бэкапы**

- **Velero** — для K8s-ресурсов и PVC.
- **pg_dump/pgBackRest/WAL-G** — для БД.
- **Тестировать восстановление.**

**11. RTO и RPO**

RTO — сколько времени на восстановление. RPO — сколько данных можно потерять. Частые бэкапы → меньше RPO. Быстрое восстановление → меньше RTO.

**12. PVC в Pending**

1. `kubectl describe pvc` — Events.
2. Проверить StorageClass.
3. Проверить CSI-драйвер.
4. Проверить логи CSI.
5. Проверить IAM (облако).
6. Проверить топологию.

**13. PVC при удалении StatefulSet**

По умолчанию PVC **остаются**. Удаляются вручную: `kubectl delete pvc <name>`. Это защита от случайной потери данных.

**14. Увеличить PVC**

Если `allowVolumeExpansion: true` в StorageClass — да:
```yaml
spec:
  resources:
    requests:
      storage: 20Gi    # было 10Gi
```

Уменьшить **нельзя**.

**15. Kafka в K8s**

**Operator (Strimzi)** — оптимально. Или **managed (Confluent Cloud, MSK)**. Ручной StatefulSet — сложно (ZooKeeper/KRaft, партиции, replication).

---

## Куда идти дальше?

Мы разобрали хранилища и Stateful-приложения. Теперь ты знаешь:

- Как работают volumes в K8s.
- PV, PVC, StorageClass.
- StatefulSet и headless Service.
- Паттерны запуска БД.
- Бэкапы.
- Диагностика PVC.

Но мы пока не разобрали:

- **Как передавать конфигурацию.** ConfigMaps, Secrets — мы упоминали, но не разбирали глубоко.
- **Как безопасно хранить секреты.** Vault, external-secrets, sealed-secrets.
- **Как обновлять конфигурацию без перезапуска.**

**Глава 12: Kubernetes — конфигурация и секреты.** Погнали. 🚀