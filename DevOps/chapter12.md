# 🔐 Глава 12: Kubernetes — конфигурация и секреты

**Что вы узнаете:**
- Что такое ConfigMap и как передавать конфигурацию в Pod'ы.
- Что такое Secret и почему он небезопасен по умолчанию.
- Как работает шифрование Secrets в etcd.
- Как обновлять конфигурацию без перезапуска Pod'ов.
- Что такое external-secrets и как интегрировать Vault, AWS Secrets Manager.
- Что такое Sealed Secrets и SOPS.
- Как безопасно передавать секреты из CI/CD в кластер.
- Как избежать утечек секретов в Git и логи.

**После прочтения вы сможете:**
- Настроить ConfigMap и Secret для приложения.
- Обновлять конфигурацию без простоя.
- Использовать external-secrets для интеграции с Vault.
- Шифровать секреты в Git через Sealed Secrets.
- Проверять безопасность Secrets в кластере.
- Диагностировать проблемы с ConfigMap и Secret.

---

## Содержание

- [12.0 Пролог: пароль в Git](#120-пролог-пароль-в-git)
- [12.1 ConfigMap: конфигурация без пересборки образа](#121-configmap-конфигурация-без-пересборки-образа)
- [12.2 Secret: секреты в Kubernetes](#122-secret-секреты-в-kubernetes)
- [12.3 Secret небезопасен по умолчанию](#123-secret-небезопасен-по-умолчанию)
- [12.4 Шифрование Secrets в etcd](#124-шифрование-secrets-в-etcd)
- [12.5 Обновление конфигурации без перезапуска](#125-обновление-конфигурации-без-перезапуска)
- [12.6 External Secrets: интеграция с Vault и облаками](#126-external-secrets-интеграция-с-vault-и-облаками)
- [12.7 Sealed Secrets и SOPS: секреты в Git](#127-sealed-secrets-и-sops-секреты-в-git)
- [12.8 Секреты из CI/CD в кластер](#128-секреты-из-cicd-в-кластер)
- [12.9 Диагностика проблем с ConfigMap и Secret](#129-диагностика-проблем-с-configmap-и-secret)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 12.0 Пролог: пароль в Git

Ты пишешь приложение. Ему нужны:

- **DATABASE_URL** с паролем к PostgreSQL.
- **API_KEY** для внешнего сервиса.
- **SMTP_PASSWORD** для отправки email.

Ты пишешь манифест:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      env:
        - name: DATABASE_URL
          value: "postgres://user:SuperSecret123@postgres:5432/mydb"
        - name: API_KEY
          value: "sk_live_abc123def456"
        - name: SMTP_PASSWORD
          value: "my-smtp-password"
```

Коммитишь в Git. Push.

**Через день — утечка.** Кто-то из команды форкнул репозиторий. Или репозиторий случайно стал публичным. Или разработчик скопировал манифест на форум, чтобы спросить про ошибку.

**Пароли в Git — это компрометация.**

Даже если ты удалишь пароли в следующем коммите — они останутся в **истории Git**. Любой может достать их через `git log`.

Даже если репозиторий приватный — слишком много людей имеют доступ. Уволенный сотрудник. Подрядчик. Случайный fork.

**Решение:**

1. **ConfigMap** — для нечувствительной конфигурации.
2. **Secret** — для чувствительных данных.
3. **Внешние secrets-менеджеры** — Vault, AWS Secrets Manager.
4. **Шифрование секретов в Git** — Sealed Secrets, SOPS.

В этой главе мы разберём все эти механизмы. И ты научишься **безопасно** работать с секретами в Kubernetes.

Это — основа **безопасности**. В Главе 19 (Безопасность K8s) мы разберём всё вместе. Но основы — здесь.

---

## 12.1 ConfigMap: конфигурация без пересборки образа

### 🔌 Проблема: конфигурация меняется, образ — нет

Твой Go-сервис имеет настройки:

- Порт (8080).
- Уровень логирования (info).
- Таймауты.
- URL других сервисов.

Если эти настройки **в образе** — при каждом изменении нужно пересобирать образ, пушить в registry, передеплоить. Медленно и неудобно.

**Решение:** ConfigMap.

### 📦 Что такое ConfigMap

**ConfigMap** — это объект Kubernetes, который хранит **нечувствительные** данные в формате key-value. Pod'ы могут использовать ConfigMap как:

- **Переменные окружения.**
- **Файлы в volume.**
- **Аргументы командной строки.**

**Ключевое:** ConfigMap **не в образе**. Изменение ConfigMap не требует пересборки образа. Но для применения изменений может понадобиться перезапуск Pod'ов (см. подглаву 12.5).

### 🎯 Создание ConfigMap

**Способ 1: YAML**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  # Простые key-value
  LOG_LEVEL: "info"
  PORT: "8080"
  TIMEOUT: "30s"
  
  # Многострочные значения (файлы)
  app.yaml: |
    server:
      port: 8080
      timeout: 30s
    database:
      host: postgres
      port: 5432
    logging:
      level: info
  
  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://myapp:8080;
      }
    }
```

**Способ 2: kubectl**

```bash
# Из литералов
kubectl create configmap myapp-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=PORT=8080

# Из файлов
kubectl create configmap myapp-config \
  --from-file=app.yaml \
  --from-file=nginx.conf

# Из директории
kubectl create configmap myapp-config --from-file=./config/
```

### 🎯 Использование как переменные окружения

**Способ 1: Все ключи как переменные**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      envFrom:
        - configMapRef:
            name: myapp-config
```

**Что произойдёт:** все ключи из ConfigMap станут переменными окружения в контейнере.

**Способ 2: Конкретные ключи**

```yaml
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: myapp-config
              key: LOG_LEVEL
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: myapp-config
              key: PORT
```

### 🎯 Использование как файлы

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      volumeMounts:
        - name: config
          mountPath: /etc/myapp
          readOnly: true
  volumes:
    - name: config
      configMap:
        name: myapp-config
```

**Что произойдёт:** каждый ключ ConfigMap станет **файлом** в `/etc/myapp`:

```
/etc/myapp/
├── LOG_LEVEL        (содержимое: "info")
├── PORT             (содержимое: "8080")
├── TIMEOUT          (содержимое: "30s")
├── app.yaml         (содержимое: многострочное)
└── nginx.conf       (содержимое: многострочное)
```

### 🎯 Монтирование конкретных ключей

```yaml
volumes:
  - name: config
    configMap:
      name: myapp-config
      items:
        - key: app.yaml
          path: config.yaml           # переименовать
        - key: LOG_LEVEL
          path: log-level.txt
```

**Что произойдёт:** смонтируются только указанные ключи, с переименованием:

```
/etc/myapp/
├── config.yaml    (из ключа app.yaml)
└── log-level.txt  (из ключа LOG_LEVEL)
```

### 🎯 subPath: монтирование одного файла

По умолчанию ConfigMap монтируется как **директория**. Если нужно смонтировать **один файл**:

```yaml
spec:
  containers:
    - name: myapp
      volumeMounts:
        - name: config
          mountPath: /etc/myapp/config.yaml     # файл
          subPath: app.yaml                     # ключ из ConfigMap
  volumes:
    - name: config
      configMap:
        name: myapp-config
```

**Что произойдёт:** в `/etc/myapp/config.yaml` будет содержимое ключа `app.yaml`. **Не перезапишет** другие файлы в `/etc/myapp/`.

**Важно:** с `subPath` ConfigMap **не обновляется** автоматически при изменении. Нужен перезапуск Pod'а.

### 🎯 Практический пример

**ConfigMap:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  DATABASE_HOST: "postgres"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "mydb"
  LOG_LEVEL: "info"
  CACHE_TTL: "300"
```

**Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          envFrom:
            - configMapRef:
                name: myapp-config
          env:
            - name: DATABASE_URL
              value: "postgres://$(DATABASE_HOST):$(DATABASE_PORT)/$(DATABASE_NAME)"
```

**Что произошло:** все ключи стали переменными окружения. `DATABASE_URL` собран из других переменных.

### 🔬 Практика: ConfigMap

```bash
# 1. Создать ConfigMap
kubectl create configmap test-config \
  --from-literal=LOG_LEVEL=debug \
  --from-literal=PORT=8080

# 2. Посмотреть
kubectl get configmap test-config -o yaml

# 3. Pod с ConfigMap
kubectl run test --image=busybox --restart=Never --overrides='
{
  "spec": {
    "containers": [{
      "name": "test",
      "image": "busybox",
      "command": ["sh", "-c", "env | sort; sleep 3600"],
      "envFrom": [{"configMapRef": {"name": "test-config"}}]
    }]
  }
}'

# 4. Проверить переменные
kubectl logs test | grep -E "LOG_LEVEL|PORT"
# LOG_LEVEL=debug
# PORT=8080

# 5. Изменить ConfigMap
kubectl edit configmap test-config
# Изменить LOG_LEVEL на "info"

# 6. Проверить, что Pod видит изменения
kubectl exec test -- env | grep LOG_LEVEL
# LOG_LEVEL=debug  ← НЕ обновилось!
```

**Что произошло:** Pod получил переменные при запуске. Изменение ConfigMap **не обновляет** переменные в работающем Pod'е. Нужен перезапуск (подглава 12.5).

### 💡 Практика: как правильно работать с ConfigMap

**✅ ОБЯЗАТЕЛЬНО:**

1. **Использовать ConfigMap для нечувствительной конфигурации.**
2. **Никогда не хранить секреты в ConfigMap.** Для этого Secret.
3. **Использовать `envFrom` для простоты** или `configMapKeyRef` для конкретных ключей.

**👍 СТОИТ:**

4. **Разделять ConfigMap по назначению** (`app-config`, `nginx-config`).
5. **Версионировать ConfigMap** через labels.

**❌ НЕ ДЕЛАЙ:**

6. **Не храни пароли, токены, ключи в ConfigMap.** Это не безопасно.
7. **Не используй `subPath`, если нужно обновление.** Не обновляется.
8. **Не путай ConfigMap и Secret.** ConfigMap — для обычной конфигурации.

### Где мы сейчас

Мы разобрали ConfigMap. Теперь — **Secret** — для чувствительных данных.

---

## 12.2 Secret: секреты в Kubernetes

### 🔌 Проблема: пароли в ConfigMap небезопасны

ConfigMap хранит данные в открытом виде в etcd. Любой, у кого есть доступ к API, может прочитать ConfigMap. Для паролей это недопустимо.

**Решение:** Secret.

### 📦 Что такое Secret

**Secret** — как ConfigMap, но для **чувствительных данных**:

- Пароли.
- Токены.
- Ключи.
- Сертификаты.

**Отличия от ConfigMap:**

1. **Монтируется как tmpfs** — в памяти, не на диске.
2. **Может быть зашифрован в etcd** (см. подглаву 12.4).
3. **Доступ контролируется RBAC** отдельно.
4. **Не логируется** в `kubectl describe` (только размер).

### 🎯 Типы Secret

| Тип | Для чего |
|:---|:---|
| `Opaque` | Обычные key-value (по умолчанию) |
| `kubernetes.io/dockerconfigjson` | Креденшелы к registry |
| `kubernetes.io/tls` | TLS-сертификаты |
| `kubernetes.io/basic-auth` | Basic auth |
| `kubernetes.io/ssh-auth` | SSH-ключи |
| `bootstrap.kubernetes.io/token` | Токены для bootstrap |

### 🎯 Создание Secret

**Способ 1: kubectl из литералов**

```bash
kubectl create secret generic db-secret \
  --from-literal=username=postgres \
  --from-literal=password=SuperSecret123
```

**Способ 2: kubectl из файлов**

```bash
kubectl create secret generic db-secret \
  --from-file=username=./username.txt \
  --from-file=password=./password.txt
```

**Способ 3: YAML (значения в base64)**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: cG9zdGdyZXM=           # base64 от "postgres"
  password: U3VwZXJTZWNyZXQxMjM=   # base64 от "SuperSecret123"
```

**Важно:** значения в `data` — **base64**, не шифрование! Легко декодируются.

**Способ 4: YAML с `stringData`**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:                        # plain text, Kubernetes сам кодирует в base64
  username: postgres
  password: SuperSecret123
```

**`stringData` удобнее** — не нужно вручную кодировать.

### 🎯 TLS Secret

```bash
kubectl create secret tls my-tls \
  --cert=tls.crt \
  --key=tls.key
```

**Или YAML:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-tls
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

**Использование в Ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  tls:
    - hosts:
        - example.com
      secretName: my-tls
```

### 🎯 Docker Registry Secret

```bash
kubectl create secret docker-registry reg-secret \
  --docker-server=myregistry.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=my@email.com
```

**Использование в Pod:**

```yaml
spec:
  imagePullSecrets:
    - name: reg-secret
  containers:
    - name: myapp
      image: myregistry.com/myapp:1.0
```

**Что произошло:** kubelet использует эти креденшелы для pull образа из приватного registry.

### 🎯 Использование как переменные окружения

```yaml
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      envFrom:
        - secretRef:
            name: db-secret
      # Или конкретные ключи
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
```

### 🎯 Использование как файлы

```yaml
spec:
  containers:
    - name: myapp
      volumeMounts:
        - name: secret
          mountPath: /etc/secret
          readOnly: true
  volumes:
    - name: secret
      secret:
        secretName: db-secret
```

**Что произойдёт:** каждый ключ станет файлом в `/etc/secret/`. Volume монтируется как **tmpfs** (в памяти).

### 🔬 Практика: Secret

```bash
# 1. Создать Secret
kubectl create secret generic test-secret \
  --from-literal=password=SuperSecret123 \
  --from-literal=api-key=sk_live_abc123

# 2. Посмотреть (значения не показываются в describe)
kubectl get secret test-secret
# NAME          TYPE     DATA   AGE
# test-secret   Opaque   2      5s

kubectl describe secret test-secret
# Name:         test-secret
# Namespace:    default
# Type:         Opaque
# Data
# ====
# api-key:   15 bytes
# password:  15 bytes
# (значения не показываются)

# 3. Посмотреть значения в base64
kubectl get secret test-secret -o jsonpath='{.data.password}' | base64 -d
# SuperSecret123

# 4. Pod с Secret
kubectl run test --image=busybox --restart=Never --overrides='
{
  "spec": {
    "containers": [{
      "name": "test",
      "image": "busybox",
      "command": ["sh", "-c", "echo $password; sleep 3600"],
      "envFrom": [{"secretRef": {"name": "test-secret"}}]
    }]
  }
}'

# 5. Проверить
kubectl logs test
# SuperSecret123
```

### 💡 Практика: как правильно работать с Secret

**✅ ОБЯЗАТЕЛЬНО:**

1. **Использовать Secret для всех чувствительных данных.**
2. **Использовать `stringData` в YAML** для удобства.
3. **Монтировать как volume, не как env** (если возможно) — так безопаснее.

**👍 СТОИТ:**

4. **Использовать RBAC** для ограничения доступа к Secret.
5. **Шифровать Secrets в etcd** (подглава 12.4).
6. **Внешние secrets-менеджеры** (Vault, AWS Secrets Manager) для production.

**❌ НЕ ДЕЛАЙ:**

7. **Не коммить Secret YAML с реальными значениями в Git.** Используй Sealed Secrets (подглава 12.7).
8. **Не логировать значения Secret.** Даже в debug.
9. **Не используй Secret в ConfigMap.** Secret — для секретов.
10. **Не забывай про base64 — это НЕ шифрование.**

### Где мы сейчас

Мы разобрали Secret. Но есть проблема — **Secret небезопасен по умолчанию**. Разберём почему.

---

## 12.3 Secret небезопасен по умолчанию

### 🔌 Проблема: Secret — это не «секрет»

Название «Secret» вводит в заблуждение. Многие думают, что Secret **шифруется** автоматически. Это **не так**.

**Что происходит по умолчанию:**

1. Secret хранится в **etcd** в base64.
2. Base64 — это **кодирование**, не шифрование.
3. Любой, у кого есть **доступ к etcd** — может прочитать Secret.
4. Любой, у кого есть **RBAC-права на чтение Secret** — может прочитать.
5. Secret монтируется в Pod как **tmpfs**, но если Pod получит root — может прочитать.

### 📊 Что защищает Secret

**Что делает Kubernetes:**

- **RBAC** — ограничивает, кто может читать Secret через API.
- **Tmpfs** — Secret не пишется на диск ноды (но в etcd — да).
- **Не логирует** значения в `kubectl describe`.

**Что НЕ делает:**

- **Не шифрует** в etcd по умолчанию.
- **Не защищает** от root на ноде.
- **Не защищает** от чтения через API, если есть RBAC-права.

### 🎯 Атаки на Secret

**1. Чтение из etcd.**

Если злоумышленник получил доступ к etcd (например, через backup), он может прочитать все Secrets.

```bash
# Прямое чтение из etcd
etcdctl get /registry/secrets/default/db-secret
# Внутри — base64 значения
```

**2. Чтение через API.**

Если есть RBAC-права:

```bash
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
# SuperSecret123
```

**3. Чтение из Pod'а.**

Если Pod запущен с root и имеет доступ к `/var/run/secrets/`:

```bash
kubectl exec -it myapp -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
# Токен ServiceAccount
```

**4. Чтение через node.**

Если злоумышленник получил root на ноде, он может читать Secrets всех Pod'ов, которые запущены на этой ноде.

**5. Через `kubectl describe` (не напрямую).**

`kubectl describe` не показывает значения, но показывает **размеры** — это уже утечка информации.

### 🎯 Что делать

**1. Включить шифрование Secrets в etcd.**

См. подглаву 12.4.

**2. Использовать RBAC.**

Минимальные права. Только те, кому нужно.

**3. Внешние secrets-менеджеры.**

Vault, AWS Secrets Manager. См. подглаву 12.6.

**4. Sealed Secrets / SOPS.**

Шифрование в Git. См. подглаву 12.7.

**5. Ограничить доступ к etcd.**

Только control plane. Backup — зашифрован.

**6. Ограничить доступ к нодам.**

Только админы. Не запускать произвольные Pod'ы.

**7. Pod Security Standards.**

Не разрешать root, privileged.

**8. Network Policies.**

Ограничить доступ к API.

### 🎯 Практический чек-лист

**Проверка безопасности Secrets:**

```bash
# 1. Проверить, зашифрованы ли Secrets в etcd
# (зависит от настройки cluster)

# 2. Проверить RBAC
kubectl auth can-i get secrets --as=system:serviceaccount:default:myapp
# yes / no

# 3. Проверить, кто имеет доступ
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq '.items[] | select(.roleRef.name | contains("secret")) | {name: .metadata.name, namespace: .metadata.namespace}'

# 4. Проверить Pod Security
kubectl get pod myapp -o jsonpath='{.spec.securityContext}'
```

### 🔬 Практика: демонстрация небезопасности

```bash
# 1. Создать Secret
kubectl create secret generic test-secret \
  --from-literal=password=SuperSecret123

# 2. Прочитать через API (если есть права)
kubectl get secret test-secret -o jsonpath='{.data.password}' | base64 -d
# SuperSecret123

# 3. Прочитать из Pod'а (mount)
kubectl run test --image=busybox --restart=Never --overrides='
{
  "spec": {
    "containers": [{
      "name": "test",
      "image": "busybox",
      "command": ["sh", "-c", "cat /etc/secret/password; sleep 3600"],
      "volumeMounts": [{"name": "secret", "mountPath": "/etc/secret"}]
    }],
    "volumes": [{
      "name": "secret",
      "secret": {"secretName": "test-secret"}
    }]
  }
}'

kubectl logs test
# SuperSecret123

# 4. Любой Pod в namespace может это сделать, если есть RBAC-права
```

**Вывод:** Secret **не защищает** от того, у кого есть доступ. Защита — в RBAC, шифровании etcd, внешних менеджерах.

### 💡 Практика: как правильно защищать Secret

**✅ ОБЯЗАТЕЛЬНО:**

1. **Включить шифрование Secrets в etcd** (подглава 12.4).
2. **RBAC: минимальные права.** Только те, кому нужно.
3. **Не коммить Secrets в Git.** Используй Sealed Secrets или external-secrets.

**👍 СТОИТ:**

4. **Внешние secrets-менеджеры** (Vault, AWS Secrets Manager).
5. **Pod Security Standards** — не разрешать root.
6. **Audit logging** — логировать доступ к Secrets.

**❌ НЕ ДЕЛАЙ:**

7. **Не думай, что Secret защищает автоматически.** Нужны дополнительные меры.
8. **Не давай `get secrets` всем.** Только конкретным ServiceAccount'ам.
9. **Не храни Secrets в Git без шифрования.**

### Где мы сейчас

Мы разобрали, что Secret небезопасен по умолчанию. Теперь — **как включить шифрование в etcd**.

---

## 12.4 Шифрование Secrets в etcd

### 🔌 Проблема: etcd хранит Secrets в открытом виде

По умолчанию etcd хранит Secrets в base64. Любой, кто получит доступ к etcd (backup, дамп), может прочитать их.

**Решение:** включить **encryption at rest** для Secrets в etcd.

### 📊 Что такое Encryption at Rest

**Encryption at Rest** — шифрование данных на диске. Когда Secret записывается в etcd, он шифруется. Когда читается — расшифровывается.

**Ключевое:**

- **etcd хранит зашифрованные данные.**
- **apiserver шифрует/расшифровывает** прозрачно.
- **Приложения работают как обычно.**

### 🎯 Как включить

**1. Создать EncryptionConfiguration.**

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps              # можно и configmaps
    providers:
      # Первый провайдер — для новых записей
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      # Последний — identity (для чтения старых незашифрованных)
      - identity: {}
```

**Сгенерировать ключ:**

```bash
head -c 32 /dev/urandom | base64
# Пример: c2VjcmV0LWtleS0zMi1ieXRlcy1sb25nISEhIQ==
```

**2. Настроить apiserver.**

**Для kubeadm-кластера:**

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
    - command:
        - kube-apiserver
        - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
        # ... другие флаги
      volumeMounts:
        - name: encryption-config
          mountPath: /etc/kubernetes/encryption-config.yaml
          readOnly: true
  volumes:
    - name: encryption-config
      hostPath:
        path: /etc/kubernetes/encryption-config.yaml
```

**3. Перезапустить apiserver.**

kubelet автоматически перезапустит apiserver после изменения манифеста.

**4. Перезаписать существующие Secrets.**

Существующие Secrets в etcd **остаются незашифрованными**. Нужно перезаписать:

```bash
# Перезаписать все Secrets
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

После этого все Secrets зашифрованы.

### 🎯 Провайдеры шифрования

| Провайдер | Ключ | Безопасность |
|:---|:---|:---|
| **identity** | нет | Без шифрования |
| **aescbc** | 32 байта | AES-CBC |
| **aesgcm** | 32 байта | AES-GCM (рекомендуется) |
| **secretbox** | 32 байта | XSalsa20+Poly1305 |
| **kms** | KMS-провайдер | Внешний KMS (AWS KMS, GCP KMS) |

**Рекомендация:** `aesgcm` или `kms`.

**Пример с `aesgcm`:**

```yaml
providers:
  - aesgcm:
      keys:
        - name: key1
          secret: <base64-encoded-32-byte-key>
  - identity: {}
```

**Пример с KMS (AWS):**

```yaml
providers:
  - kms:
      name: aws-kms
      endpoint: unix:///var/run/kmsplugin/socket.sock
      cachesize: 1000
      timeout: 3s
  - identity: {}
```

**Преимущества KMS:** ключ хранится в KMS (AWS KMS, GCP KMS), не в конфиге apiserver.

### 🎯 Проверка шифрования

```bash
# 1. Прочитать Secret напрямую из etcd
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/test-secret

# 2. Если зашифрован — увидишь "k8s:enc:aesgcm:v1:key1:..."
# Если нет — увидишь base64 значения
```

### 🎯 Ротация ключей

**Периодически меняй ключи шифрования.** Как:

1. Добавь новый ключ **первым** в конфиг.
2. Перезапусти apiserver.
3. Перезапиши все Secrets (`kubectl get secrets -A -o json | kubectl replace -f -`).
4. Удали старый ключ из конфига.
5. Перезапусти apiserver.

### 🔬 Практика: encryption at rest

```bash
# 1. Посмотреть конфиг apiserver (если доступен)
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep encryption

# 2. Если пусто — шифрование не включено

# 3. Проверить через etcd (требует доступа к etcd)
sudo ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/test-secret

# Если base64 — не зашифровано
# Если k8s:enc:... — зашифровано
```

### 💡 Практика: как правильно настроить шифрование

**✅ ОБЯЗАТЕЛЬНО:**

1. **Включить encryption at rest** в production.
2. **Использовать `aesgcm` или `kms`.**
3. **Перезаписать существующие Secrets** после включения.

**👍 СТОИТ:**

4. **KMS** для управления ключами (AWS KMS, GCP KMS).
5. **Ротация ключей** раз в год.
6. **Backup encryption config** отдельно.

**❌ НЕ ДЕЛАЙ:**

7. **Не храни ключ шифрования в конфиге apiserver.** Для production — KMS.
8. **Не забывай перезаписать Secrets.** Иначе старые останутся в открытом виде.

### Где мы сейчас

Мы разобрали шифрование. Теперь — **как обновлять конфигурацию без перезапуска**.

---

## 12.5 Обновление конфигурации без перезапуска

### 🔌 Проблема: изменение ConfigMap не применяется

Ты изменил ConfigMap. Но Pod'ы продолжают работать со старым значением. Почему?

**Как работает обновление:**

| Тип использования | Обновляется автоматически? | Задержка |
|:---|:---|:---|
| **envFrom / valueFrom** | ❌ Нет | Нужен перезапуск |
| **Volume (без subPath)** | ✅ Да | 1-2 минуты |
| **Volume (с subPath)** | ❌ Нет | Нужен перезапуск |

**envFrom** — переменные окружения устанавливаются **при запуске контейнера**. Изменение ConfigMap не обновляет их в работающем Pod'е.

**Volume без subPath** — kubelet периодически синхронизирует содержимое ConfigMap с mounted volume. Задержка 1-2 минуты.

### 🎯 Способы обновления

**1. Rolling restart Deployment.**

```bash
kubectl rollout restart deployment/myapp
```

**Что произойдёт:** Kubernetes создаст новые Pod'ы с обновлённым ConfigMap.

**Плюсы:** просто, работает для envFrom.
**Минусы:** перезапуск Pod'ов.

**2. Reloader.**

**Reloader** — инструмент, который автоматически перезапускает Pod'ы при изменении ConfigMap или Secret.

```bash
# Установить Reloader
helm repo add stakater https://stakater.github.io/stakater-charts
helm install reloader stakater/reloader
```

**Аннотация на Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    reloader.stakater.com/auto: "true"    # авто-reload при изменении ConfigMap/Secret
spec:
  ...
```

**Что произойдёт:** Reloader watch на ConfigMap/Secret. При изменении — rolling restart Deployment.

**3. Immutable ConfigMap.**

Можно сделать ConfigMap **immutable** — тогда его нельзя изменить, только создать новый.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config-v2      # новая версия
immutable: true
data:
  ...
```

**Плюсы:** явное версионирование.
**Минусы:** нужно обновлять Deployment вручную.

**4. Version в имени ConfigMap.**

**Паттерн:** добавлять version в имя ConfigMap.

```yaml
# ConfigMap myapp-config-v1
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config-v1
data:
  ...
---
# Deployment использует v1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: myapp
          envFrom:
            - configMapRef:
                name: myapp-config-v1
```

**При обновлении:**

1. Создать `myapp-config-v2`.
2. Обновить Deployment, чтобы использовал `v2`.
3. `kubectl apply` → rolling update.

**Плюсы:** явное версионирование, GitOps-friendly.
**Минусы:** нужно обновлять Deployment вручную или через CI/CD.

**5. Helm checksum annotation.**

Если используешь Helm:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      ...
```

**Что произойдёт:** при изменении ConfigMap меняется checksum → Helm видит изменение в Pod template → rolling restart.

### 🎯 Что выбрать

**Для production:**

- **Reloader** — автоматизация.
- **Helm checksum** — если используешь Helm.
- **Version в имени** — для GitOps.

**Для dev:**

- **`kubectl rollout restart`** — просто.

**Для критичных приложений:**

- **Не полагаться на автоматическое обновление.** Явно управлять версиями.

### 🔬 Практика: Reloader

```bash
# 1. Установить Reloader
helm repo add stakater https://stakater.github.io/stakater-charts
helm repo update
helm install reloader stakater/reloader

# 2. ConfigMap
kubectl create configmap test-config --from-literal=key=value1

# 3. Deployment с аннотацией
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
  annotations:
    reloader.stakater.com/auto: "true"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
        - name: test
          image: busybox
          command: ["sh", "-c", "echo \$key; sleep 3600"]
          envFrom:
            - configMapRef:
                name: test-config
EOF

# 4. Проверить
kubectl get pods -l app=test
kubectl logs <pod>
# value1

# 5. Изменить ConfigMap
kubectl edit configmap test-config
# key: value2

# 6. Reloader автоматически перезапустит Pod'ы через 30 секунд
kubectl get pods -l app=test
# Новые Pod'ы с новым значением

kubectl logs <new-pod>
# value2
```

### 💡 Практика: как правильно обновлять конфигурацию

**✅ ОБЯЗАТЕЛЬНО:**

1. **Понимать, что envFrom не обновляется автоматически.**
2. **Использовать Reloader** для автоматизации.
3. **Тестировать обновление конфигурации** перед production.

**👍 СТОИТ:**

4. **Helm checksum** для Helm-приложений.
5. **Version в имени ConfigMap** для GitOps.
6. **Immutable ConfigMap** для критичных.

**❌ НЕ ДЕЛАЙ:**

7. **Не полагайся на volume без subPath** для env-переменных.
8. **Не забывай, что subPath не обновляется.**
9. **Не обновляй ConfigMap вручную на production без плана.**

### Где мы сейчас

Мы разобрали обновление конфигурации. Теперь — **external-secrets** — интеграция с Vault.

---

## 12.6 External Secrets: интеграция с Vault и облаками

### 🔌 Проблема: Secret в K8s — не единственный источник

Хорошая практика — хранить секреты **вне** Kubernetes:

- **HashiCorp Vault** — централизованный secrets-менеджер.
- **AWS Secrets Manager** — managed в AWS.
- **GCP Secret Manager**.
- **Azure Key Vault**.

**Преимущества:**

- Централизованное управление.
- Аудит доступа.
- Ротация секретов.
- Динамические секреты (Vault).

**Проблема:** Pod'ы не могут читать из Vault напрямую (нужен SDK, аутентификация).

**Решение:** External Secrets Operator.

### 📊 Что такое External Secrets Operator

**External Secrets Operator (ESO)** — оператор, который:

1. Watch на **ExternalSecret** (CR).
2. Читает секрет из **внешнего** источника (Vault, AWS SM).
3. Создаёт обычный **Secret** в Kubernetes.
4. Обновляет Secret при изменении внешнего источника.

**Схема:**

```
┌──────────────────┐
│  Vault / AWS SM  │  ← внешний источник
└────────┬─────────┘
         │
         │ read
         ▼
┌──────────────────────────┐
│  External Secrets        │
│  Operator                │
└────────┬─────────────────┘
         │
         │ create/update
         ▼
┌──────────────────────────┐
│  Kubernetes Secret       │  ← обычный Secret
└────────┬─────────────────┘
         │
         │ mount
         ▼
┌──────────────────────────┐
│  Pod                     │
└──────────────────────────┘
```

**Pod работает с обычным Secret.** Он не знает про Vault.

### 🎯 Установка

```bash
# Helm
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

### 🎯 Настройка SecretStore

**SecretStore** — описывает, как подключиться к внешнему источнику.

**Для HashiCorp Vault:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: default
spec:
  provider:
    vault:
      server: https://vault.example.com
      path: secret
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: myapp-role
          serviceAccountRef:
            name: myapp
```

**Что произошло:**

- ESO аутентифицируется в Vault через ServiceAccount `myapp`.
- Vault проверяет токен и решает, разрешено ли.
- ESO читает секреты из `secret/` пути.

**Для AWS Secrets Manager:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
  namespace: default
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-west-2
      auth:
        jwt:
          serviceAccountRef:
            name: myapp
```

**Для GCP Secret Manager:**

```yaml
spec:
  provider:
    gcpsm:
      projectID: my-project
      auth:
        workloadIdentity:
          clusterLocation: us-west1
          clusterName: my-cluster
          serviceAccountRef:
            name: myapp
```

### 🎯 ExternalSecret

**ExternalSecret** — описывает, **какой** секрет читать и **как** превратить в Kubernetes Secret.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
  namespace: default
spec:
  refreshInterval: 1h                    # как часто обновлять
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: myapp-secrets                  # имя создаваемого Secret
    creationPolicy: Owner
  data:
    - secretKey: DATABASE_PASSWORD       # ключ в K8s Secret
      remoteRef:
        key: myapp/database               # путь в Vault
        property: password                # поле
    - secretKey: API_KEY
      remoteRef:
        key: myapp/api
        property: key
```

**Что произойдёт:**

1. ESO читает `myapp/database.password` и `myapp/api.key` из Vault.
2. Создаёт Kubernetes Secret `myapp-secrets` с ключами `DATABASE_PASSWORD` и `API_KEY`.
3. Каждый час (refreshInterval) обновляет значения.

**Pod использует обычный Secret:**

```yaml
spec:
  containers:
    - name: myapp
      envFrom:
        - secretRef:
            name: myapp-secrets
```

### 🎯 ClusterSecretStore

**ClusterSecretStore** — как SecretStore, но доступен из всех namespace.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-global
spec:
  provider:
    vault:
      server: https://vault.example.com
      ...
```

**Когда использовать:** когда один Vault для всего кластера.

### 🎯 Динамические секреты (Vault)

Vault умеет выдавать **динамические** секреты — временные credentials для БД.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: postgres-creds
spec:
  refreshInterval: 5m                    # обновлять каждые 5 минут
  secretStoreRef:
    name: vault-backend
  target:
    name: postgres-creds
  data:
    - secretKey: username
      remoteRef:
        key: database/creds/myapp-role  # Vault генерирует временные credentials
        property: username
    - secretKey: password
      remoteRef:
        key: database/creds/myapp-role
        property: password
```

**Что произойдёт:**

- Vault создаёт временного пользователя в PostgreSQL с TTL 1 час.
- ESO читает credentials и создаёт Secret.
- Через 5 минут ESO обновляет Secret (новые credentials).
- Pod перезапускается (через Reloader) и использует новые credentials.

**Это — production-grade подход.** Секреты не живут дольше TTL.

### 🔬 Практика: External Secrets Operator

```bash
# 1. Установить ESO
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace

# 2. Создать SecretStore (для теста — fake provider)
kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: fake-store
  namespace: default
spec:
  provider:
    fake:
      data:
        - key: /myapp/db-password
          value: "SuperSecret123"
EOF

# 3. ExternalSecret
kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: fake-store
    kind: SecretStore
  target:
    name: myapp-secrets
  data:
    - secretKey: DATABASE_PASSWORD
      remoteRef:
        key: /myapp/db-password
EOF

# 4. Проверить
kubectl get externalsecret myapp-secrets
kubectl get secret myapp-secrets
kubectl get secret myapp-secrets -o jsonpath='{.data.DATABASE_PASSWORD}' | base64 -d
# SuperSecret123
```

### 💡 Практика: как правильно использовать ESO

**✅ ОБЯЗАТЕЛЬНО:**

1. **Vault или managed secrets** для production.
2. **External Secrets Operator** для интеграции.
3. **RBAC для ServiceAccount** — минимальные права в Vault.
4. **Refresh interval** — разумный (1 час или меньше).

**👍 СТОИТ:**

5. **Динамические секреты** (Vault) для БД.
6. **ClusterSecretStore** для глобального доступа.
7. **Audit logging** в Vault.

**❌ НЕ ДЕЛАЙ:**

8. **Не храни секреты в K8s Secret напрямую** в production. Используй ESO.
9. **Не давай ESO слишком много прав.** Только чтение конкретных путей.
10. **Не забывай про ротацию.** Регулярно обновляй секреты.

### Где мы сейчас

Мы разобрали external-secrets. Теперь — **Sealed Secrets и SOPS** — секреты в Git.

---

## 12.7 Sealed Secrets и SOPS: секреты в Git

### 🔌 Проблема: как хранить секреты в Git

**GitOps** требует, чтобы всё было в Git:

- Deployments.
- Services.
- ConfigMaps.
- Secrets.

Но **секреты в Git** — это компрометация. Даже в приватном репозитории.

**Решение:** шифровать секреты перед коммитом.

**Инструменты:**

- **Sealed Secrets** — от Bitnami.
- **SOPS** — от Mozilla.

### 📊 Sealed Secrets

**Sealed Secrets** — контроллер, который:

1. **Шифрует** секреты **публичным ключом** кластера.
2. Ты коммитишь **зашифрованный** файл (SealedSecret) в Git.
3. Контроллер в кластере **расшифровывает** его **приватным ключом**.
4. Создаёт обычный Secret.

**Схема:**

```
┌─────────────────────────────────────┐
│  Разработчик                         │
│                                      │
│  1. Создать Secret YAML              │
│  2. kubeseal encrypt → SealedSecret  │
│  3. git commit SealedSecret          │
└──────────────────┬───────────────────┘
                   │
                   │ git push
                   ▼
┌─────────────────────────────────────┐
│  Git-репозиторий                     │
│  (SealedSecret в открытом виде)      │
└──────────────────┬───────────────────┘
                   │
                   │ kubectl apply
                   ▼
┌─────────────────────────────────────┐
│  Sealed Secrets Controller           │
│  (в кластере)                        │
│                                      │
│  1. Расшифровать приватным ключом    │
│  2. Создать обычный Secret           │
└─────────────────────────────────────┘
```

### 🎯 Установка Sealed Secrets

```bash
# Helm
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# Или через manifest
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.0/controller.yaml

# Установить CLI
brew install kubeseal  # macOS
# или скачать с GitHub
```

### 🎯 Использование

**1. Создать обычный Secret (не коммитить!):**

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: default
type: Opaque
stringData:
  password: SuperSecret123
  username: postgres
```

**2. Зашифровать:**

```bash
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
```

**Что произойдёт:**

```yaml
# sealed-secret.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-secret
  namespace: default
spec:
  encryptedData:
    password: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...   # зашифровано
    username: AgCtr8Y5cWwvvHsdvGDEq...
  template:
    metadata:
      name: db-secret
      namespace: default
    type: Opaque
```

**3. Коммитить SealedSecret в Git:**

```bash
git add sealed-secret.yaml
git commit -m "Add sealed db secret"
git push
```

**4. Применить:**

```bash
kubectl apply -f sealed-secret.yaml
```

**Что произойдёт:**

1. Контроллер видит SealedSecret.
2. Расшифровывает приватным ключом.
3. Создаёт обычный Secret `db-secret` в namespace `default`.

### 🎯 Область видимости

**SealedSecret привязан к:**

- **Namespace** (по умолчанию).
- **Имени** Secret.

**Можно расширить:**

```bash
# Разрешить использование в любом namespace
kubeseal --scope cluster-wide < secret.yaml > sealed-secret.yaml
```

**Смысл:** злоумышленник не может скопировать SealedSecret в другой namespace и получить доступ.

### 📊 SOPS

**SOPS (Secrets OPerationS)** — инструмент от Mozilla для шифрования файлов.

**Как работает:**

1. Шифрует **значения** в YAML/JSON, оставляя **ключи** открытыми.
2. Использует **KMS** (AWS, GCP, Azure) или **age**/PGP.
3. Файл коммитится в Git.
4. При деплое — расшифровывается (например, в CI/CD).

**Пример зашифрованного файла:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  password: ENC[AES256_GCM,data:abc123...,iv:...,tag:...,type:str]
  username: ENC[AES256_GCM,data:def456...,iv:...,tag:...,type:str]
sops:
  kms: []
  age:
    - recipient: age1abc...
      enc: |
        -----BEGIN AGE ENCRYPTED FILE-----
        ...
```

**Ключи открыты**, значения зашифрованы.

### 🎯 Sealed Secrets vs SOPS

| Аспект | Sealed Secrets | SOPS |
|:---|:---|:---|
| Шифрование | Публичный ключ кластера | KMS / age / PGP |
| Расшифровка | В кластере (контроллер) | Вне кластера (CI/CD) |
| Зависимость от кластера | Да | Нет |
| Работа с Helm | Через плагин | Нативно |
| Rotation | Пересоздать SealedSecret | Перешифровать файл |
| Multi-cluster | Нужен ключ на кластер | Один KMS на все |

**Когда использовать:**

- **Sealed Secrets** — если хочешь расшифровку в кластере.
- **SOPS** — если хочешь расшифровку в CI/CD или multi-cluster.

### 🎯 Sealed Secrets + Helm

**Плагин `helm-secrets`:**

```bash
helm plugin install https://github.com/jkroepke/helm-secrets
```

**Использование:**

```bash
# Зашифровать values.yaml
sops -e -i secrets.yaml

# Деплой
helm secrets install myapp ./chart -f secrets.yaml
```

**Что произойдёт:** helm-secrets расшифрует `secrets.yaml` перед применением.

### 🔬 Практика: Sealed Secrets

```bash
# 1. Установить controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# 2. Установить kubeseal CLI
brew install kubeseal

# 3. Создать Secret
cat > secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: default
stringData:
  password: SuperSecret123
EOF

# 4. Зашифровать
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# 5. Посмотреть
cat sealed-secret.yaml
# Зашифрованные значения

# 6. Удалить оригинал (важно!)
rm secret.yaml

# 7. Применить
kubectl apply -f sealed-secret.yaml

# 8. Проверить, что Secret создан
kubectl get secret db-secret
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
# SuperSecret123

# 9. Коммитить sealed-secret.yaml в Git — безопасно!
git add sealed-secret.yaml
git commit -m "Add sealed secret"
```

### 💡 Практика: как правильно хранить секреты в Git

**✅ ОБЯЗАТЕЛЬНО:**

1. **Никогда не коммить обычные Secret YAML** с реальными значениями.
2. **Использовать Sealed Secrets или SOPS.**
3. **`.gitignore` для `*secret*.yaml`** на всякий случай.

**👍 СТОИТ:**

4. **Sealed Secrets** — если GitOps в кластере (ArgoCD, Flux).
5. **SOPS + age/KMS** — если multi-cluster или CI/CD деплой.
6. **Rotate encryption keys** регулярно.

**❌ НЕ ДЕЛАЙ:**

7. **Не коммить приватный ключ Sealed Secrets** в Git.
8. **Не используй один и тот же ключ для всех кластеров.**
9. **Не забывай, что SealedSecret привязан к namespace.**

### Где мы сейчас

Мы разобрали Sealed Secrets и SOPS. Теперь — **секреты из CI/CD**.

---

## 12.8 Секреты из CI/CD в кластер

### 🔌 Проблема: как передать секреты из CI

В Главе 6 мы разбирали CI/CD. В Главе 7 — деплой в K8s. Но как передать секреты из CI в кластер?

**Варианты:**

1. **Sealed Secrets + GitOps.** CI коммитит зашифрованный секрет.
2. **External Secrets.** CI коммитит ExternalSecret, ESO создаёт Secret.
3. **Vault + CI.** CI читает из Vault и создаёт Secret.
4. **Secrets из CI напрямую.** CI хранит секреты и применяет их.

### 📊 Паттерн 1: Sealed Secrets + GitOps

**CI:**

```yaml
stages:
  - update-secrets

update-sealed-secrets:
  stage: update-secrets
  image: bitnami/kubeseal:latest
  script:
    # Расшифровать секрет из CI variables
    - echo "$DB_PASSWORD" | kubeseal --raw --from-file=/dev/stdin --name db-secret --namespace production > sealed-password.yaml
    
    # Или использовать готовый Secret
    - kubectl create secret generic db-secret --from-literal=password=$DB_PASSWORD --dry-run=client -o yaml | kubeseal -o yaml > sealed-secret.yaml
    
    # Коммитить в GitOps-репозиторий
    - git clone https://oauth2:$GITOPS_TOKEN@gitlab.com/org/gitops.git
    - cp sealed-secret.yaml gitops/production/
    - cd gitops && git add . && git commit -m "Update db-secret" && git push
```

**Что произошло:**

- CI получил секрет из CI variables.
- Зашифровал через `kubeseal`.
- Закоммитил в GitOps-репозиторий.
- ArgoCD применит SealedSecret → создаст Secret.

**Плюсы:**

- Секрет **никогда** не попадает в CI-логи.
- GitOps как единый источник правды.

### 📊 Паттерн 2: External Secrets + CI

**CI:**

```yaml
update-external-secret:
  stage: update-secrets
  image: bitnami/kubectl:latest
  script:
    # Применить ExternalSecret, который читает из Vault
    - kubectl apply -f k8s/external-secret.yaml
```

**ExternalSecret в Git:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
spec:
  secretStoreRef:
    name: vault-backend
  target:
    name: myapp-secrets
  data:
    - secretKey: DATABASE_PASSWORD
      remoteRef:
        key: myapp/database
        property: password
```

**Что произошло:**

- CI применяет ExternalSecret.
- ESO читает из Vault → создаёт Secret.
- **Секрет вообще не в Git.**

**Плюсы:**

- Секрет в Vault, не в Git.
- CI вообще не знает секрет.
- Ротация через Vault.

**Минусы:**

- Нужен Vault в инфраструктуре.

### 📊 Паттерн 3: Vault + CI напрямую

**CI:**

```yaml
deploy:
  stage: deploy
  image: vault:latest
  script:
    # Аутентификация через JWT
    - export VAULT_TOKEN=$(vault write -field=token auth/jwt/login role=myapp jwt=$CI_JOB_JWT)
    
    # Читать секрет
    - export DB_PASSWORD=$(vault kv get -field=password secret/myapp/db)
    
    # Создать Secret в K8s
    - kubectl create secret generic db-secret --from-literal=password=$DB_PASSWORD --dry-run=client -o yaml | kubectl apply -f -
```

**Что произошло:**

- CI аутентифицировался в Vault через JWT.
- Прочитал секрет.
- Создал Secret в K8s.

**Плюсы:**

- Секрет в Vault.
- Нет зависимости от ESO.

**Минусы:**

- CI знает секрет (в переменных).
- Может попасть в логи, если не осторожен.

### 📊 Паттерн 4: Sealed Secrets + CI напрямую

**CI:**

```yaml
deploy:
  stage: deploy
  image: bitnami/kubeseal:latest
  script:
    - echo "$DB_PASSWORD" > /tmp/password
    - kubeseal --raw --from-file=/tmp/password --name db-secret --namespace production > /tmp/sealed
    - kubectl create secret generic db-secret --from-file=password=/tmp/sealed --dry-run=client -o yaml | kubectl apply -f -
    - rm /tmp/password /tmp/sealed
```

**Что произошло:**

- CI зашифровал секрет.
- Применил SealedSecret.
- Контроллер создал Secret.

**Плюсы:**

- Секрет не попадает в Git.
- Sealed Secrets работает.

**Минусы:**

- CI знает секрет.

### 🎯 Что выбрать

| Паттерн | Секрет в Git | Секрет в CI | Сложность | Рекомендация |
|:---|:---|:---|:---|:---|
| **Sealed + GitOps** | Зашифрован | Нет | Средняя | ⭐ GitOps |
| **External Secrets** | Нет | Нет | Высокая | ⭐ Production |
| **Vault + CI** | Нет | Да | Средняя | Если нет ESO |
| **Sealed + CI** | Зашифрован | Да | Низкая | Простые случаи |

**Рекомендация:**

- **Production:** External Secrets + Vault.
- **GitOps:** Sealed Secrets.
- **Простые случаи:** Sealed Secrets через CI.

### 🔬 Практика: Sealed Secrets через CI

```yaml
# .gitlab-ci.yml
stages:
  - deploy

deploy-secrets:
  stage: deploy
  image: bitnami/kubeseal:latest
  variables:
    # DB_PASSWORD из CI/CD Variables (Masked + Protected)
    K8S_NAMESPACE: production
    SECRET_NAME: db-secret
  before_script:
    # Установить kubectl
    - curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    - chmod +x kubectl && mv kubectl /usr/local/bin/
    # Настроить kubeconfig
    - mkdir -p ~/.kube
    - echo "$KUBE_CONFIG" | base64 -d > ~/.kube/config
  script:
    # Создать Secret и зашифровать
    - |
      kubectl create secret generic $SECRET_NAME \
        --from-literal=password=$DB_PASSWORD \
        --namespace=$K8S_NAMESPACE \
        --dry-run=client -o yaml | \
        kubeseal --format yaml > /tmp/sealed-secret.yaml
    # Применить
    - kubectl apply -f /tmp/sealed-secret.yaml
    # Удалить временный файл
    - rm /tmp/sealed-secret.yaml
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

### 💡 Практика: как правильно передавать секреты из CI

**✅ ОБЯЗАТЕЛЬНО:**

1. **CI/CD Variables с Masked + Protected.**
2. **Не логировать секреты.** Даже в debug.
3. **Удалять временные файлы** после использования.
4. **Использовать External Secrets или Sealed Secrets.**

**👍 СТОИТ:**

5. **Vault с OIDC** для CI — без статических токенов.
6. **Разные секреты для разных окружений.**
7. **Ротация секретов** регулярно.

**❌ НЕ ДЕЛАЙ:**

8. **Не коммить секреты в Git** без шифрования.
9. **Не использовать один секрет для всех окружений.**
10. **Не передавать секреты через артефакты.**
11. **Не забывать очищать временные файлы.**

### Где мы сейчас

Мы разобрали передачу секретов из CI. Теперь — **диагностика проблем**.

---

## 12.9 Диагностика проблем с ConfigMap и Secret

### 🔌 Проблема: Pod не запускается

Pod падает с ошибкой. Или не запускается вообще. Возможные причины связаны с ConfigMap или Secret.

### 🔍 Типичные проблемы

**1. ConfigMap не найден.**

**Симптом:**

```bash
kubectl describe pod myapp
# Events:
#   Warning  FailedMount  MountVolume.SetUp failed for volume "config" : configmap "myapp-config" not found
```

**Причины:**

- ConfigMap не создан.
- Неправильное имя.
- Неправильный namespace.

**Решение:**

```bash
# Проверить, что ConfigMap существует
kubectl get configmap myapp-config
kubectl get configmap -A | grep myapp-config

# Проверить namespace
kubectl get configmap myapp-config -n production
```

**2. Ключ не найден в ConfigMap.**

**Симптом:**

```bash
kubectl describe pod myapp
# Events:
#   Warning  CreateContainerConfigError  couldn't find key DATABASE_HOST in ConfigMap default/myapp-config
```

**Причины:**

- Опечатка в имени ключа.
- Ключ отсутствует.

**Решение:**

```bash
kubectl get configmap myapp-config -o yaml
# Проверить, что ключ существует
```

**3. Secret не найден.**

Аналогично ConfigMap.

**4. Pod не может смонтировать Secret.**

```bash
kubectl describe pod myapp
# Events:
#   Warning  FailedMount  MountVolume.SetUp failed for volume "secret" : secret "db-secret" not found
```

**Решение:** проверить, что Secret существует в **том же namespace**, что Pod.

**5. Permission denied при чтении Secret.**

**Симптом:** приложение не может прочитать файл в `/etc/secret`.

**Причины:**

- Неправильные права на файле.
- `readOnly: false` в volume (по умолчанию Secret монтируется read-only).

**Решение:** проверить `volumeMounts` — `readOnly: true`.

**6. Приложение использует старое значение.**

**Причины:**

- envFrom: не обновляется автоматически.
- subPath: не обновляется.

**Решение:** Reloader или rollout restart.

### 🔍 Алгоритм диагностики

**Шаг 1: Проверить Pod**

```bash
kubectl get pods
kubectl describe pod myapp
```

**Шаг 2: Посмотреть Events**

Events в `describe` покажут причину.

**Шаг 3: Проверить ConfigMap/Secret**

```bash
kubectl get configmap myapp-config -o yaml
kubectl get secret db-secret -o yaml
```

**Шаг 4: Проверить namespace**

```bash
kubectl get configmap -A | grep myapp-config
kubectl get secret -A | grep db-secret
```

**Шаг 5: Проверить, что Pod видит данные**

```bash
# Для envFrom
kubectl exec myapp -- env | grep DATABASE

# Для volume
kubectl exec myapp -- ls -la /etc/myapp/
kubectl exec myapp -- cat /etc/myapp/config.yaml
```

**Шаг 6: Логи**

```bash
kubectl logs myapp
kubectl logs myapp --previous    # если падал
```

### 🎯 Расширенная диагностика

**1. Проверить RBAC.**

Если Pod использует ServiceAccount для доступа к API:

```bash
kubectl auth can-i get configmaps --as=system:serviceaccount:default:myapp
kubectl auth can-i get secrets --as=system:serviceaccount:default:myapp
```

**2. Проверить admission webhooks.**

Некоторые admission webhooks могут модифицировать или блокировать Pod'ы с Secret.

```bash
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
```

**3. Проверить ResourceQuota.**

```bash
kubectl get resourcequota -n production
kubectl describe resourcequota -n production
```

**4. Проверить LimitRange.**

```bash
kubectl get limitrange -n production
```

### 🔬 Практика: диагностика

```bash
# 1. Pod с несуществующим ConfigMap
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-bad
spec:
  containers:
    - name: test
      image: busybox
      command: ["sleep", "3600"]
      envFrom:
        - configMapRef:
            name: nonexistent-config
EOF

# 2. Проверить статус
kubectl get pod test-bad
# STATUS: CreateContainerConfigError

# 3. Описать
kubectl describe pod test-bad
# Events:
#   Warning  CreateContainerConfigError  couldn't find key ... in ConfigMap

# 4. Исправить: создать ConfigMap или изменить Pod
kubectl delete pod test-bad
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **`kubectl describe pod`** — Events.
2. **`kubectl get configmap/secret -o yaml`** — проверить содержимое.
3. **`kubectl exec`** — проверить, что Pod видит данные.

**👍 СТОИТ:**

4. **Проверять RBAC.**
5. **Проверять admission webhooks.**
6. **Мониторинг и алерты.**

**❌ НЕ ДЕЛАЙ:**

7. **Не пересоздавай Pod'ы без понимания.**
8. **Не игнорируй `CreateContainerConfigError`.**

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **ConfigMap** | Объект K8s для нечувствительной конфигурации. |
| **Secret** | Объект K8s для чувствительных данных. |
| **stringData** | Поле в Secret для plain text значений. |
| **data** | Поле в Secret для base64 значений. |
| **tmpfs** | Файловая система в памяти. |
| **Encryption at rest** | Шифрование данных на диске (etcd). |
| **EncryptionConfiguration** | Конфиг для apiserver. |
| **aesgcm** | AES-GCM шифрование. |
| **KMS** | Key Management Service. |
| **Reloader** | Инструмент для авто-перезапуска Pod'ов. |
| **ExternalSecret** | CR для интеграции с внешними secrets. |
| **SecretStore** | Описание подключения к внешнему источнику. |
| **ClusterSecretStore** | SecretStore для всего кластера. |
| **External Secrets Operator** | Оператор для external-secrets. |
| **Sealed Secrets** | Инструмент для шифрования секретов в Git. |
| **SealedSecret** | CR с зашифрованными данными. |
| **kubeseal** | CLI для Sealed Secrets. |
| **SOPS** | Secrets OPerationS — инструмент шифрования. |
| **age** | Современный инструмент шифрования. |
| **Vault** | HashiCorp Vault — secrets-менеджер. |
| **RBAC** | Role-Based Access Control. |
| **subPath** | Монтирование одного файла из volume. |

---

## Что мы узнали?

- **ConfigMap** — для нечувствительной конфигурации. Env, files, args.
- **Secret** — для чувствительных данных. Tmpfs, RBAC, типы.
- **Secret небезопасен по умолчанию.** Base64 — не шифрование.
- **Encryption at rest** — шифрование Secrets в etcd через apiserver.
- **Обновление конфигурации:** envFrom не обновляется, volume обновляется через 1-2 минуты. Reloader для автоматизации.
- **External Secrets** — интеграция с Vault, AWS SM, GCP SM.
- **Sealed Secrets** — шифрование секретов в Git через ключ кластера.
- **SOPS** — шифрование файлов через KMS/age/PGP.
- **Секреты из CI:** Sealed Secrets, External Secrets, Vault.
- **Диагностика:** `kubectl describe pod`, Events, `kubectl exec`.

---

## Типичные ошибки

- ❌ **Хранить секреты в ConfigMap.** Используй Secret.
- ❌ **Коммитить Secret YAML с реальными значениями в Git.** Используй Sealed Secrets.
- ❌ **Думать, что base64 — это шифрование.**
- ❌ **Не включать encryption at rest** в production.
- ❌ **Использовать envFrom** и ожидать автоматического обновления.
- ❌ **Использовать subPath** и ожидать обновления.
- ❌ **Давать всем `get secrets`.** RBAC.
- ❌ **Не использовать external-secrets** для production.
- ❌ **Забывать ротировать секреты.**
- ❌ **Логировать значения Secret.**
- ❌ **Использовать один секрет для всех окружений.**
- ❌ **Не проверять Events при проблемах с ConfigMap/Secret.**

---

## Для быстрого повторения

- **ConfigMap:** нечувствительная конфигурация. `envFrom`, `configMapKeyRef`, volume.
- **Secret:** чувствительные данные. `stringData`, `data`, tmpfs.
- **Типы Secret:** Opaque, tls, dockerconfigjson, basic-auth, ssh-auth.
- **Encryption at rest:** EncryptionConfiguration + флаг apiserver.
- **Обновление:** envFrom не обновляется, volume обновляется (1-2 мин). Reloader.
- **External Secrets:** ExternalSecret + SecretStore. Vault, AWS SM, GCP SM.
- **Sealed Secrets:** kubeseal → SealedSecret в Git. Привязка к namespace.
- **SOPS:** шифрует значения, ключи открыты. KMS/age/PGP.
- **Из CI:** Sealed Secrets через CI, External Secrets, Vault + OIDC.
- **Диагностика:** `kubectl describe pod`, `kubectl get configmap/secret -o yaml`, `kubectl exec`.

---

## Вопросы для самопроверки

1. Чем ConfigMap отличается от Secret?
2. Как передать ConfigMap в Pod? Три способа.
3. Что такое base64? Почему это не шифрование?
4. Что такое encryption at rest? Как включить?
5. Что произойдёт при изменении ConfigMap, если Pod использует envFrom?
6. Что произойдёт при изменении ConfigMap, если Pod монтирует volume без subPath?
7. Что такое Reloader? Зачем нужен?
8. Что такое External Secrets Operator? Как работает?
9. Что такое Sealed Secrets? Как использовать?
10. Чем Sealed Secrets отличается от SOPS?
11. Как передать секрет из CI/CD в кластер? Три паттерна.
12. ConfigMap не найден, Pod в CreateContainerConfigError. Как диагностировать?
13. Что произойдёт, если удалить ConfigMap, который использует Pod?
14. Ты закоммитил Secret YAML с паролем в Git. Что делать?
15. Что такое SecretStore vs ClusterSecretStore?

---

## Ответы

**1. ConfigMap vs Secret**

ConfigMap — для нечувствительных данных, хранится в открытом виде. Secret — для чувствительных, монтируется как tmpfs, может быть зашифрован в etcd.

**2. Три способа передачи ConfigMap**

1. `envFrom` — все ключи как переменные.
2. `valueFrom.configMapKeyRef` — конкретные ключи.
3. Volume — как файлы.

**3. base64**

Кодирование, не шифрование. Легко декодируется: `echo "c3VwZXI=" | base64 -d`.

**4. Encryption at rest**

Шифрование данных на диске etcd. Включается через `--encryption-provider-config` в apiserver. Провайдеры: aesgcm, aescbc, secretbox, kms.

**5. envFrom + изменение ConfigMap**

Переменные окружения устанавливаются при запуске контейнера. Изменение ConfigMap **не обновляет** их в работающем Pod'е. Нужен перезапуск.

**6. Volume без subPath**

Kubelet периодически синхронизирует содержимое. Задержка 1-2 минуты. Автоматически обновляется.

**7. Reloader**

Инструмент, который watch на ConfigMap/Secret и автоматически перезапускает Pod'ы при изменении. Через аннотацию `reloader.stakater.com/auto: "true"`.

**8. External Secrets Operator**

Оператор, который читает секреты из внешнего источника (Vault, AWS SM) и создаёт обычный Kubernetes Secret. Через CR: ExternalSecret + SecretStore.

**9. Sealed Secrets**

Инструмент шифрования секретов для Git. Публичный ключ кластера шифрует, приватный (в контроллере) расшифровывает. `kubeseal` CLI.

**10. Sealed Secrets vs SOPS**

Sealed Secrets: шифрование ключом кластера, расшифровка в кластере. SOPS: шифрование через KMS/age/PGP, расшифровка вне кластера.

**11. Три паттерна CI**

1. **Sealed Secrets + GitOps** — CI шифрует, коммитит в Git, ArgoCD применяет.
2. **External Secrets** — CI коммитит ExternalSecret, ESO читает из Vault.
3. **Vault + CI** — CI читает из Vault, создаёт Secret.

**12. Диагностика ConfigMap**

1. `kubectl describe pod` — Events.
2. `kubectl get configmap <name>` — существует?
3. `kubectl get configmap -A | grep <name>` — в правильном namespace?
4. `kubectl get configmap <name> -o yaml` — ключ есть?

**13. Удалить ConfigMap**

Pod продолжит работать с уже загруженными данными (env или файлами в памяти). Но при перезапуске Pod'а — ошибка. И volume может стать недоступным.

**14. Секрет в Git**

1. **Немедленно отозвать** скомпрометированный секрет (сменить пароль, ротировать токен).
2. Удалить из текущей версии файла.
3. **Переписать историю Git** (`git filter-repo` или BFG Repo-Cleaner).
4. Force push.
5. Уведомить всех, кто имел доступ.
6. Включить secret scanning для предотвращения.

**15. SecretStore vs ClusterSecretStore**

SecretStore — в namespace, доступен только в нём. ClusterSecretStore — кластерный, доступен из всех namespace. ClusterSecretStore удобен для одного Vault на кластер.

---

## Куда идти дальше?

Мы разобрали конфигурацию и секреты в Kubernetes. Теперь ты знаешь:

- ConfigMap и Secret.
- Encryption at rest.
- Обновление конфигурации.
- External Secrets.
- Sealed Secrets.
- Передачу секретов из CI.

Но мы пока не разобрали:

- **Helm** — менеджер пакетов для Kubernetes (Глава 13).
- **Terraform** — инфраструктура как код (Глава 14).
- **Ansible** — конфигурационное управление (Глава 15).
- **GitOps** — Git как источник правды (Глава 16).

Эти главы — про **автоматизацию**. Как управлять Kubernetes и инфраструктурой декларативно.

**Глава 13: Helm — менеджер пакетов для Kubernetes.** Погнали. 🚀