# ☸️ Глава 8: Kubernetes — архитектура и основные объекты

**Что вы узнаете:**
- Из чего состоит кластер Kubernetes: control plane и worker nodes.
- Что делает каждый компонент: kube-apiserver, etcd, scheduler, controller-manager, kubelet, kube-proxy.
- Как работает модель «желаемое состояние» и reconciliation loop.
- Что такое Pod и почему это минимальная единица деплоя.
- Как работают Deployments, ReplicaSets, DaemonSets, Jobs, CronJobs, StatefulSets.
- Как Services находят Pod'ы и как работает service discovery.
- Как использовать `kubectl` для управления кластером.
- Как устроены labels, selectors, annotations, namespaces.

**После прочтения вы сможете:**
- Объяснить, что происходит от `kubectl apply` до запущенного Pod'а.
- Выбрать правильный тип workload для задачи.
- Написать манифесты для Deployment, Service, ConfigMap.
- Диагностировать проблемы: Pod в Pending, CrashLoopBackOff, ImagePullBackOff.
- Понимать, как Kubernetes приводит систему к желаемому состоянию.

---

## Содержание

- [8.0 Пролог: kubectl apply — и что дальше?](#80-пролог-kubectl-apply--и-что-дальше)
- [8.1 Что такое Kubernetes и зачем он нужен](#81-что-такое-kubernetes-и-зачем-он-нужен)
- [8.2 Архитектура кластера: control plane и worker nodes](#82-архитектура-кластера-control-plane-и-worker-nodes)
- [8.3 Модель «желаемое состояние» и reconciliation loop](#83-модель-желаемое-состояние-и-reconciliation-loop)
- [8.4 Pod: минимальная единица деплоя](#84-pod-минимальная-единица-деплоя)
- [8.5 Labels, selectors, annotations](#85-labels-selectors-annotations)
- [8.6 Workloads: Deployment, ReplicaSet, DaemonSet, Job, CronJob, StatefulSet](#86-workloads-deployment-replicaset-daemonset-job-cronjob-statefulset)
- [8.7 Services: как Pod'ы находят друг друга](#87-services-как-podы-находят-друг-друга)
- [8.8 Namespaces: виртуальные кластеры](#88-namespaces-виртуальные-кластеры)
- [8.9 kubectl: рабочий инструмент](#89-kubectl-рабочий-инструмент)
- [8.10 Диагностика: Pod не запускается](#810-диагностика-pod-не-запускается)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 8.0 Пролог: kubectl apply — и что дальше?

Ты написал манифест:

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
          image: myregistry.com/myapp:v1.0.0
          ports:
            - containerPort: 8080
```

Применяешь:

```bash
kubectl apply -f deployment.yaml
# deployment.apps/myapp created
```

Через 30 секунд проверяешь:

```bash
kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# myapp-7d9f8c6b4d-abc12   1/1     Running   0          28s
# myapp-7d9f8c6b4d-def34   1/1     Running   0          28s
# myapp-7d9f8c6b4d-ghi56   1/1     Running   0          28s
```

Три Pod'а работают. Приложение доступно. Но что произошло **между** `kubectl apply` и запущенными Pod'ами?

**Кто принял решение создать 3 Pod'а?** В манифесте написано `replicas: 3` — но кто это читает?

**Кто выбрал, на каких нодах запустить Pod'ы?** Их три, нод может быть десять. Как Kubernetes решает, где запускать?

**Кто скачал образ `myregistry.com/myapp:v1.0.0`?** Где-то на ноде должен быть Docker или containerd, который это делает.

**Кто следит, что Pod'ы живы?** Если один упадёт — Kubernetes создаст новый. Кто именно?

Ответы на эти вопросы — в **архитектуре Kubernetes**. Это не монолит, а набор компонентов, каждый из которых делает свою работу. В этой главе мы разберём каждый из них.

Это — фундамент для всего остального. Без понимания архитектуры ты будешь использовать `kubectl` как заклинание, не понимая, что происходит под капотом.

---

## 8.1 Что такое Kubernetes и зачем он нужен

### 🔌 Проблема: Docker Compose не масштабируется на production

В Главе 5 мы разобрали Docker Compose. Он отлично работает для локальной разработки: один YAML, одна команда, всё поднимается.

Но у Compose есть **фундаментальные ограничения** для production:

**1. Одна машина.** Compose работает на одном хосте. Если хост падает — всё падает. Нет отказоустойчивости.

**2. Нет автоматического восстановления.** Если контейнер упал — Compose его перезапустит (с `restart: unless-stopped`). Но если хост целиком умер — ничего не поделаешь.

**3. Нет горизонтального масштабирования.** `--scale app=5` запустит 5 контейнеров на **одной** машине. Если нужно 50 — не хватит ресурсов.

**4. Нет автоматического распределения нагрузки.** Нужно вручную настраивать Nginx / HAProxy.

**5. Нет rolling update.** `docker compose up -d` перезапустит контейнеры с простоем.

**6. Нет service discovery.** Контейнеры находят друг друга по именам только в рамках одного проекта.

**7. Нет управления секретами.** Секреты в `.env` или в YAML — небезопасно.

**Все эти проблемы решает Kubernetes.**

### 📦 Что такое Kubernetes

**Kubernetes** (K8s) — это **оркестратор контейнеров**. Он управляет запуском, масштабированием и восстановлением контейнеров в **кластере** машин.

**Ключевые идеи:**

1. **Декларативность.** Ты описываешь **желаемое состояние** («3 Pod'а с myapp:v1.0.0»), Kubernetes приводит систему к нему.
2. **Кластер.** Несколько машин работают как одна. Kubernetes распределяет нагрузку.
3. **Самовосстановление.** Если Pod упал — Kubernetes создаст новый. Если нода упала — Pod'ы переедут на другие.
4. **Автоматическое масштабирование.** Больше нагрузки — больше Pod'ов.
5. **Rolling update.** Обновление без простоя.
6. **Service discovery.** Pod'ы находят друг друга по DNS-именам.

### 📊 Что даёт Kubernetes

| Проблема в Compose | Решение в Kubernetes |
|:---|:---|
| Одна машина | Кластер из N машин |
| Нет отказоустойчивости | Pod'ы перезапускаются, переезжают на другие ноды |
| Нет масштабирования | HPA (Horizontal Pod Autoscaler) |
| Нет балансировки | Service + Ingress |
| Простой при деплое | Rolling update |
| Нет service discovery | CoreDNS |
| Секреты в .env | Secrets API |

### 🎯 Когда использовать Kubernetes

**✅ Использовать:**

- **Production-нагрузки** с требованиями к отказоустойчивости.
- **Много микросервисов** (5+).
- **Нужно масштабирование** (горизонтальное).
- **Multi-cloud или hybrid-cloud**.
- **Команда больше 5 разработчиков.**

**❌ Не использовать:**

- **Маленькие проекты** — 1-2 сервиса. Docker Compose справится.
- **Dev-окружения** — Docker Compose или даже `docker run`.
- **Batch-задачи** — иногда проще cron + systemd.

**Правило:** Kubernetes оправдан, когда его сложность компенсируется решаемыми проблемами. Для 1-2 сервисов он избыточен.

### 🌍 Краткая история

- **2013** — Google выпускает **Borg** (внутренний оркестратор) как **Omega**.
- **2014** — Google выпускает **Kubernetes** как open source.
- **2015** — Kubernetes 1.0. Основан **CNCF** (Cloud Native Computing Foundation).
- **2017** — Kubernetes побеждает в «войне оркестраторов» (Docker Swarm, Mesos).
- **2024** — Kubernetes — стандарт де-факто для оркестрации.

Сегодня Kubernetes используют все крупные компании: Google, Amazon, Microsoft, Netflix, Spotify, Airbnb.

### 💡 Практика: что важно понять про Kubernetes

**✅ ОБЯЗАТЕЛЬНО:**

1. **Kubernetes — это декларативный.** Ты описываешь **что**, а не **как**.
2. **Kubernetes — это кластер.** Не одна машина, а много.
3. **Kubernetes — это самовосстанавливающаяся система.** Он постоянно приводит систему к желаемому состоянию.

**👍 СТОИТ:**

4. **Начинать с managed Kubernetes** (EKS, GKE, AKS) — не поднимать свой.
5. **Понимать, что K8s сложен.** Для маленьких проектов он не нужен.

**❌ НЕ ДЕЛАЙ:**

6. **Не использовать Kubernetes для 1-2 сервисов.** Избыточно.
7. **Не думать, что K8s решит все проблемы.** Он добавляет свои.

### Где мы сейчас

Мы разобрали, зачем нужен Kubernetes. Теперь посмотрим на **архитектуру кластера** — из чего он состоит.

---

## 8.2 Архитектура кластера: control plane и worker nodes

### 🔌 Проблема: как машины работают вместе

Кластер Kubernetes — это **несколько машин**, которые работают как одна. Но как они координируются? Кто решает, где запускать Pod? Кто следит, что Pod'ы живы?

Ответ — в **архитектуре**: есть **control plane** (управляющие компоненты) и **worker nodes** (рабочие машины).

### 📊 Общая схема

```
┌───────────────────────────────────────────────────────────────────┐
│                          CONTROL PLANE                             │
│                      (управляющие компоненты)                      │
│                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────────┐ │
│  │ kube-apiserver│  │    etcd      │  │   kube-scheduler        │ │
│  │              │  │              │  │                         │ │
│  │ REST API     │  │ Ключ-значение│  │ Распределяет Pod'ы      │ │
│  │ для kubectl  │  │ хранилище    │  │ по нодам                │ │
│  └──────────────┘  └──────────────┘  └─────────────────────────┘ │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │              kube-controller-manager                          │ │
│  │                                                               │ │
│  │  • Replication Controller (следит за replicas)                │ │
│  │  • Node Controller (следит за нодами)                        │ │
│  │  • Endpoint Controller (обновляет endpoints)                  │ │
│  │  • Service Account Controller                                 │ │
│  │  • ... ещё десятки контроллеров                               │ │
│  └──────────────────────────────────────────────────────────────┘ │
└──────────────────────────┬────────────────────────────────────────┘
                           │
                           │ REST API / gRPC
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  WORKER NODE 1│  │  WORKER NODE 2│  │  WORKER NODE 3│
│               │  │               │  │               │
│ ┌───────────┐ │  │ ┌───────────┐ │  │ ┌───────────┐ │
│ │  kubelet  │ │  │ │  kubelet  │ │  │ │  kubelet  │ │
│ └───────────┘ │  │ └───────────┘ │  │ └───────────┘ │
│ ┌───────────┐ │  │ ┌───────────┐ │  │ ┌───────────┐ │
│ │kube-proxy │ │  │ │kube-proxy │ │  │ │kube-proxy │ │
│ └───────────┘ │  │ └───────────┘ │  │ └───────────┘ │
│ ┌───────────┐ │  │ ┌───────────┐ │  │ ┌───────────┐ │
│ │containerd │ │  │ │containerd │ │  │ │containerd │ │
│ └───────────┘ │  │ └───────────┘ │  │ └───────────┘ │
│               │  │               │  │               │
│ ┌───┐ ┌───┐   │  │ ┌───┐ ┌───┐   │  │ ┌───┐ ┌───┐   │
│ │Pod│ │Pod│   │  │ │Pod│ │Pod│   │  │ │Pod│ │Pod│   │
│ └───┘ └───┘   │  │ └───┘ └───┘   │  │ └───┘ └───┘   │
└───────────────┘  └───────────────┘  └───────────────┘
```

### 🧠 Control Plane

**Control plane** — это «мозг» кластера. Он принимает решения, хранит состояние, координирует работу нод.

**Компоненты:**

**1. kube-apiserver**

**Единственная точка входа** в кластер. Все операции (через `kubectl`, через другие компоненты) идут через него.

- **REST API** — принимает YAML/JSON манифесты.
- **Аутентификация** — проверяет, кто делает запрос.
- **Авторизация** — проверяет, имеет ли право (RBAC).
- **Validation** — проверяет, что манифест корректен.
- **Admission Controllers** — модифицируют или отклоняют запросы (мутационные и валидационные webhooks).
- **Хранение** — записывает в etcd.

Все остальные компоненты **только** через apiserver. Никто не обращается к etcd напрямую (кроме apiserver).

**2. etcd**

**Распределённое ключ-значение хранилище.** Это **единственный** источник правды о состоянии кластера.

- Хранит все ресурсы: Pod'ы, Service'ы, ConfigMap'ы, Secrets.
- **Raft consensus** — консенсус для consistency между репликами.
- **Watch API** — другие компоненты подписываются на изменения.

**Критически важно:** etcd — самое важное в кластере. Если etcd потеряется — потеряется весь кластер. Поэтому:

- **Replicas.** Минимум 3 реплики для отказоустойчивости.
- **Backups.** Регулярные бэкапы — обязательны.
- **SSD.** Быстрый диск для etcd.

**3. kube-scheduler**

**Решает, на какой ноде запустить Pod.**

**Как работает:**

1. Смотрит на новые Pod'ы без назначенной ноды.
2. Применяет **filters** — ноды, которые подходят по требованиям (ресурсы, affinity).
3. Применяет **scores** — ранжирует ноды (наименее загруженные — лучше).
4. Выбирает лучшую ноду и говорит apiserver: «запусти Pod там».

**Filters (примеры):**

- `NodeResourcesFit` — достаточно ли CPU/RAM.
- `NodeAffinity` — подходит ли нода по affinity.
- `TaintsTolerations` — толерантность к taints.
- `PodTopologySpread` — распределение по зонам.

**Scores (примеры):**

- `LeastAllocated` — меньше загружена — выше score.
- `BalancedAllocation` — балансирует использование ресурсов.
- `ImageLocality` — если образ уже на ноде — выше score.

Подробно разберём в Главе 9.

**4. kube-controller-manager**

**Набор контроллеров**, каждый следит за своим аспектом состояния.

**Основные контроллеры:**

- **Deployment Controller** — управляет ReplicaSet'ами.
- **ReplicaSet Controller** — следит, чтобы было ровно N Pod'ов.
- **Node Controller** — следит за здоровьем нод.
- **Endpoint Controller** — обновляет Endpoints для Service'ов.
- **Service Account Controller** — создаёт ServiceAccount'ы.
- **Namespace Controller** — управляет namespaces.
- **PersistentVolume Controller** — связывает PV и PVC.
- **... десятки других.**

Каждый контроллер — это **reconciliation loop**: смотрит на желаемое состояние, сравнивает с текущим, делает шаги для сближения.

**5. cloud-controller-manager** (в облаке)

Отдельный компонент для интеграции с облачным провайдером:

- Управляет Load Balancer'ами (создаёт ELB в AWS).
- Управляет нодами (удаляет ноды из облака).
- Управляет маршрутами.

### 🖥️ Worker Nodes

**Worker node** — рабочая машина, где запускаются Pod'ы.

**Компоненты:**

**1. kubelet**

**Агент на каждой ноде.** Он:

- Регистрируется в apiserver.
- Получает PodSpec'ы для своей ноды.
- Запускает и останавливает контейнеры через container runtime.
- Выполняет health checks (liveness, readiness).
- Сообщает статус Pod'ов в apiserver.

**kubelet не управляется Kubernetes напрямую.** Он работает как systemd-сервис. Если kubelet упадёт — нода станет NotReady.

**2. kube-proxy**

**Сетевой агент на каждой ноде.** Он:

- Реализует Service'ы (ClusterIP, NodePort, LoadBalancer).
- Настраивает iptables/IPVS правила для маршрутизации трафика.
- Обновляет правила при изменениях Service'ов и Endpoints.

**Как работает:**

1. Service создаёт виртуальный IP (ClusterIP).
2. kube-proxy настраивает правила: «трафик на этот IP → один из Pod'ов».
3. Когда Pod обращается к Service — трафик идёт на реальный Pod.

Разберём подробно в Главе 10.

**3. Container Runtime**

**Программа, которая запускает контейнеры.** В современном Kubernetes:

- **containerd** — самый популярный.
- **CRI-O** — для OpenShift.
- **Docker** — через `dockershim` (удалён в K8s 1.24).

Runtime работает через **CRI (Container Runtime Interface)** — стандартный интерфейс. Любой runtime, реализующий CRI, работает с Kubernetes.

### 🔄 Что происходит при `kubectl apply`

Теперь проследим полный путь:

```
1. kubectl apply -f deployment.yaml
   │
   │ HTTPS (REST API)
   ▼
2. kube-apiserver
   │
   ├─ Аутентификация (кто это?)
   ├─ Авторизация (можно ли?)
   ├─ Validation (корректен ли манифест?)
   ├─ Admission Controllers (мутации/проверки)
   │
   │ запись в etcd
   ▼
3. etcd
   │
   │ Deployment сохранён
   ▼
4. Deployment Controller (в kube-controller-manager)
   │
   │ видит новый Deployment
   │ создаёт ReplicaSet
   ▼
5. kube-apiserver → etcd
   │
   │ ReplicaSet сохранён
   ▼
6. ReplicaSet Controller (в kube-controller-manager)
   │
   │ видит новый ReplicaSet
   │ создаёт 3 Pod'а (без нод)
   ▼
7. kube-apiserver → etcd
   │
   │ Pod'ы сохранены (nodeName: "")
   ▼
8. kube-scheduler
   │
   │ видит Pod'ы без ноды
   │ для каждого выбирает ноду (filters + scores)
   │ обновляет Pod (nodeName: "node-1")
   ▼
9. kubelet на node-1
   │
   │ видит Pod, назначенный на его ноду
   │ скачивает образ
   │ запускает контейнер через containerd
   │ обновляет статус Pod (Running)
   ▼
10. kube-apiserver → etcd
    │
    │ статус Pod сохранён
    ▼
11. kubectl get pods
    │
    │ видит Pod'ы в статусе Running
    ▼
```

**Время:** 10-30 секунд от `kubectl apply` до Running Pod'ов.

### 🎯 Watch API: как компоненты узнают об изменениях

**Ключевой механизм:** компоненты **не опрашивают** apiserver постоянно. Они подписываются на изменения через **Watch API**.

**Как работает:**

1. kubelet подписывается на изменения Pod'ов для своей ноды.
2. apiserver отправляет **событие** при каждом изменении.
3. kubelet получает событие и реагирует.

**Преимущества:**

- **Эффективность.** Нет постоянных запросов.
- **Быстрота.** События приходят сразу.
- **Масштабируемость.** Один apiserver обслуживает тысячи клиентов.

### 💡 Практика: что важно понять про архитектуру

**✅ ОБЯЗАТЕЛЬНО:**

1. **apiserver — единственная точка входа.** Все операции через него.
2. **etcd — единственный источник правды.** Если потеряется — потеряется кластер.
3. **kubelet — агент на каждой ноде.** Запускает контейнеры.
4. **Контроллеры — reconciliation loops.** Приводят систему к желаемому состоянию.

**👍 СТОИТ:**

5. **Понимать разделение control plane / workers.** Control plane — мозг, workers — руки.
6. **Знать про Watch API** — как компоненты узнают об изменениях.

**❌ НЕ ДЕЛАЙ:**

7. **Не размещать рабочие нагрузки на control plane.** Обычно control plane — отдельные машины.
8. **Не игнорировать бэкапы etcd.** Если потеряешь — потеряешь всё.

### Где мы сейчас

Мы разобрали архитектуру кластера. Теперь — **модель «желаемое состояние»** и reconciliation loop — сердце Kubernetes.

---

## 8.3 Модель «желаемое состояние» и reconciliation loop

### 🔌 Проблема: как Kubernetes знает, что делать

В традиционных системах ты говоришь **что делать**:

```bash
docker run -d nginx         # импéративно: запусти контейнер
systemctl start nginx       # импéративно: запусти сервис
```

В Kubernetes ты говоришь **что должно быть**:

```yaml
spec:
  replicas: 3               # декларативно: должно быть 3 Pod'а
```

**Как Kubernetes решает, что делать?** Как он обеспечивает, чтобы всегда было 3 Pod'а, даже если один упал, нода умерла, или ты удалил Pod вручную?

Ответ: **reconciliation loop**.

### 🎯 Декларативный vs императивный

| Подход | Как | Пример |
|:---|:---|:---|
| **Императивный** | «Сделай X» | `docker run nginx` |
| **Декларативный** | «Хочу, чтобы было X» | `replicas: 3` |

**Императивный:**

- Ты даёшь команду.
- Если что-то упадёт — не восстановится само.
- Нужно помнить текущее состояние.

**Декларативный:**

- Ты описываешь желаемое состояние.
- Система сама приводит к нему.
- Не важно текущее состояние — важно желаемое.

### 🔄 Reconciliation loop

**Reconciliation loop** — это бесконечный цикл:

```
1. Получить желаемое состояние (из манифеста)
2. Получить текущее состояние (из etcd)
3. Сравнить
4. Если не совпадают — сделать шаги для сближения
5. Повторить
```

**Пример: ReplicaSet Controller**

```go
for {
    // 1. Желаемое состояние
    desired := getReplicaSetDesiredReplicas()  // 3
    
    // 2. Текущее состояние
    current := countPodsForReplicaSet()        // 2 (один упал)
    
    // 3. Сравнить
    if current < desired {
        // 4. Создать недостающий Pod
        createPod()
    } else if current > desired {
        // Удалить лишний Pod
        deletePod()
    }
    
    // 5. Подождать события
    waitForEvent()
}
```

**Что происходит:**

1. Controller смотрит на ReplicaSet.
2. Видит `replicas: 3`.
3. Считает Pod'ы: 2.
4. Создаёт новый Pod.
5. Снова считает: 3.
6. Всё сходится — ждёт событий.

**Если Pod упадёт:**

1. Событие: Pod перешёл в Failed.
2. Controller просыпается.
3. Считает Pod'ы: 2.
4. Создаёт новый.

**Если ты удалишь Pod вручную:**

1. Событие: Pod удалён.
2. Controller просыпается.
3. Создаёт новый.

**Если нода умерла:**

1. Node Controller видит, что нода NotReady.
2. Помечает Pod'ы на ней как Failed.
3. ReplicaSet Controller видит, что Pod'ов стало меньше.
4. Создаёт новые на других нодах.

**Вот почему Kubernetes самовосстанавливающийся.**

### 🎯 Уровни reconciliation

Reconciliation работает на **нескольких уровнях**:

```
Deployment (replicas: 3)
    │
    │ Deployment Controller
    ▼
ReplicaSet (replicas: 3)
    │
    │ ReplicaSet Controller
    ▼
Pod, Pod, Pod
    │
    │ kubelet
    ▼
Container, Container, Container
```

**Каждый уровень — свой reconciliation loop:**

- **Deployment Controller** — следит, что ReplicaSet соответствует Deployment.
- **ReplicaSet Controller** — следит, что Pod'ов ровно N.
- **kubelet** — следит, что контейнеры в Pod'ах запущены.

### 🎯 Level-triggered vs edge-triggered

**Ключевое различие:**

**Edge-triggered:** реагирует на **события** («Pod упал»).

**Level-triggered:** реагирует на **состояние** («сейчас 2 Pod'а, должно быть 3»).

**Kubernetes — level-triggered.**

**Почему это важно:**

- Если ты пропустил событие (например, apiserver был недоступен) — level-triggered всё равно приведёт к правильному состоянию.
- Reconciliation идёт **постоянно**, не только при событиях.
- «Eventual consistency» — в конечном счёте система сойдётся.

**Пример edge-triggered:** если Pod упал, но controller пропустил событие — Pod не восстановится.

**Пример level-triggered:** если Pod упал, controller в следующий цикл увидит «2 Pod'а вместо 3» и создаст новый. Даже если пропустил событие.

### 🔬 Практика: наблюдаем reconciliation

```bash
# 1. Создай Deployment
kubectl create deployment nginx --image=nginx --replicas=3

# 2. Проверь Pod'ы
kubectl get pods
# NAME                     READY   STATUS
# nginx-7d9f8c6b4d-abc12   1/1     Running
# nginx-7d9f8c6b4d-def34   1/1     Running
# nginx-7d9f8c6b4d-ghi56   1/1     Running

# 3. Удали один Pod вручную
kubectl delete pod nginx-7d9f8c6b4d-abc12

# 4. Через секунду — новый Pod
kubectl get pods
# NAME                     READY   STATUS
# nginx-7d9f8c6b4d-def34   1/1     Running
# nginx-7d9f8c6b4d-ghi56   1/1     Running
# nginx-7d9f8c6b4d-xyz78   1/1     Running    ← новый!

# 5. Проверь ReplicaSet
kubectl get replicaset
# NAME                DESIRED   CURRENT   READY
# nginx-7d9f8c6b4d     3         3         3

# 6. Измени replicas
kubectl scale deployment nginx --replicas=5

# 7. Через секунду — 5 Pod'ов
kubectl get pods
# 5 Pod'ов
```

**Что произошло:**

- ReplicaSet Controller постоянно проверяет: «сколько Pod'ов сейчас vs сколько должно быть».
- Если не совпадает — создаёт или удаляет Pod'ы.
- Это — reconciliation loop в действии.

### 🎯 Как описать reconciliation

**Формула:**

```
Desired State (из манифеста)
        │
        ▼
┌───────────────────────┐
│  Reconciliation Loop  │
│                       │
│  Наблюдение → Сравнение → Действие
│       ▲                      │
│       │                      │
│       └──────────────────────┘
        │
        ▼
Current State (в etcd)
```

**Контроллеры постоянно крутят этот цикл.** Не один раз, а бесконечно.

### 💡 Практика: что важно понять про reconciliation

**✅ ОБЯЗАТЕЛЬНО:**

1. **Kubernetes — декларативный.** Ты говоришь «что», не «как».
2. **Reconciliation loop — level-triggered.** Смотрит на состояние, не на события.
3. **Eventual consistency.** В конечном счёте система сойдётся к желаемому состоянию.

**👍 СТОИТ:**

4. **Понимать, что контроллеры — отдельные процессы.** Они могут падать и перезапускаться.
5. **Знать, что reconciliation занимает время.** Не мгновенно.

**❌ НЕ ДЕЛАЙ:**

6. **Не пытайся «помочь» Kubernetes вручную.** Если он создаёт Pod'ы, а ты их удаляешь — он будет создавать снова.
7. **Не думай, что Kubernetes мгновенный.** 10-30 секунд на реакцию — норма.

### Где мы сейчас

Мы разобрали reconciliation loop — сердце Kubernetes. Теперь — **Pod** — минимальная единица деплоя.

---

## 8.4 Pod: минимальная единица деплоя

### 🔌 Проблема: почему не контейнер

В Docker минимальная единица — контейнер. В Kubernetes — **Pod**.

**Pod** — это группа из одного или нескольких контейнеров, которые:

- **Запускаются вместе** на одной ноде.
- **Разделяют сетевой namespace** (один IP, один loopback).
- **Разделяют storage** (volumes).
- **Разделяют lifecycle** (запускаются и умирают вместе).

### 📦 Что такое Pod

**Pod — это обёртка над контейнерами.** Сам Pod не запускает процессы — он описывает, какие контейнеры запустить.

**Минимальный Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
```

**Что произошло:**

1. Kubernetes создал Pod `nginx`.
2. Внутри — один контейнер `nginx`.
3. Pod получил свой IP.
4. Контейнер запущен.

**Но Pod'ы напрямую не создают.** Обычно Pod'ы создаются через **Deployment** (или другой workload). Потому что Pod сам по себе:

- Не перезапустится при падении.
- Не масштабируется.
- Не обновляется без простоя.

### 🎯 Почему Pod, а не контейнер

**Зачем группировать контейнеры в Pod?**

**Основные причины:**

**1. Sidecar-паттерн.**

Рядом с основным контейнером работает вспомогательный:

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
    - name: log-shipper
      image: fluent-bit:latest
      # читает логи app и отправляет в Loki
```

**2. Init-контейнеры.**

Контейнеры, которые выполняются **до** основного:

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox
      command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 1; done']
  containers:
    - name: app
      image: myapp:1.0
```

**3. Разделение ответственности.**

- app — основное приложение.
- sidecar — прокси, логирование, метрики.
- init — подготовка.

**4. Разделяемые ресурсы.**

- Один network namespace — `localhost` между контейнерами.
- Один volume — общие файлы.

### 📊 Жизненный цикл Pod

```
1. Pending        — Pod создан, но не назначен на ноду
2. ContainerCreating — нода назначена, образ скачивается
3. Running        — контейнеры запущены
4. Succeeded      — все контейнеры завершились успешно (для Job)
5. Failed         — хотя бы один контейнер упал
6. Unknown        — нода недоступна
```

**Диагностические статусы:**

| Статус | Что означает |
|:---|:---|
| `Pending` | Не может быть назначен на ноду |
| `ContainerCreating` | Образ скачивается, контейнеры запускаются |
| `Running` | Работает |
| `CrashLoopBackOff` | Контейнер падает и перезапускается |
| `ImagePullBackOff` | Не может скачать образ |
| `ErrImagePull` | Ошибка при скачивании образа |
| `Completed` | Завершился успешно (для Job) |
| `Error` | Завершился с ошибкой |

### 🎯 Pod и IP-адреса

**Каждый Pod имеет уникальный IP** в кластере.

- Pod'ы видят друг друга по IP (если нет Network Policies).
- IP **не сохраняется** при пересоздании Pod'а.
- Поэтому **нельзя обращаться к Pod'у по IP напрямую** — IP меняется.

**Решение:** Service (подглава 8.7).

### 🎯 Pod и volumes

**Volume** — это директория, доступная контейнерам в Pod'е.

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: data
          mountPath: /app/data
  volumes:
    - name: data
      emptyDir: {}      # временный volume, живёт пока Pod жив
```

**Типы volumes:**

| Тип | Что делает |
|:---|:---|
| `emptyDir` | Временный, удаляется с Pod'ом |
| `hostPath` | Директория на ноде |
| `configMap` | Содержимое ConfigMap |
| `secret` | Содержимое Secret |
| `persistentVolumeClaim` | PVC (Глава 12) |

Подробно разберём в Главе 12.

### 🎯 Sidecar-паттерн

**Sidecar** — контейнер, который работает **рядом** с основным, помогая ему.

**Примеры:**

**1. Логирование:**

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
    - name: log-shipper
      image: fluent-bit:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
  volumes:
    - name: logs
      emptyDir: {}
```

**2. Service Mesh (Istio, Linkerd):**

Sidecar-прокси перехватывает трафик Pod'а для mTLS, retry, metrics.

**3. Адаптер:**

Sidecar преобразует формат данных основного контейнера (например, Prometheus exporter).

### 🔬 Практика: создание Pod'а

```bash
# 1. Создать Pod
kubectl run nginx --image=nginx:1.25 --port=80

# 2. Посмотреть Pod
kubectl get pods
# NAME    READY   STATUS    RESTARTS   AGE
# nginx   1/1     Running   0          10s

# 3. Описание Pod'а
kubectl describe pod nginx

# 4. Логи
kubectl logs nginx

# 5. Зайти в Pod
kubectl exec -it nginx -- bash

# 6. Удалить Pod
kubectl delete pod nginx
```

**Важно:** `kubectl run` создаёт **голый** Pod (без ReplicaSet, без контроллера). Он не перезапустится при падении.

**Правильно:** создавать через Deployment.

### 💡 Практика: что важно понять про Pod'ы

**✅ ОБЯЗАТЕЛЬНО:**

1. **Pod — минимальная единица деплоя в K8s.** Не контейнер.
2. **Не создавай Pod'ы напрямую.** Используй Deployment.
3. **Pod'ы эфемерны.** IP меняется при пересоздании.

**👍 СТОИТ:**

4. **Sidecar-паттерн** для логирования, прокси, метрик.
5. **Init-контейнеры** для подготовки.

**❌ НЕ ДЕЛАЙ:**

6. **Не создавай Pod'ы без контроллера.** Они не восстановятся.
7. **Не обращайся к Pod'ам по IP.** Используй Service.

### Где мы сейчас

Мы разобрали Pod'ы. Теперь — **labels, selectors, annotations** — как K8s находит ресурсы.

---

## 8.5 Labels, selectors, annotations

### 🔌 Проблема: как K8s связывает ресурсы

Kubernetes не использует имена для связи ресурсов. Например, Service находит Pod'ы **не по имени**, а по **labels**.

**Labels** — это key-value пары, которые можно привязать к любому ресурсу.

```yaml
metadata:
  labels:
    app: myapp
    version: v1.0.0
    environment: production
```

### 🎯 Labels

**Labels** — метаданные для **идентификации и выборки**. Используются для:

- Организации ресурсов.
- Селекторов (Service, Deployment).
- `kubectl get -l`.

**Синтаксис:**

```
key: value

Префикс и имя:
  example.com/environment: production
  
Правила:
  - prefix — DNS-поддомен (опционально)
  - name — до 63 символов, начинается и заканчивается буквой/цифрой
  - value — до 63 символов, буквы, цифры, -, _, .
```

**Примеры:**

```yaml
labels:
  app: myapp
  version: v1.2.3
  tier: backend
  environment: production
  team: payments
  app.kubernetes.io/name: myapp
  app.kubernetes.io/version: "1.2.3"
  app.kubernetes.io/component: api
```

**Рекомендуемые labels (Kubernetes conventions):**

| Label | Значение |
|:---|:---|
| `app.kubernetes.io/name` | Имя приложения |
| `app.kubernetes.io/instance` | Уникальное имя экземпляра |
| `app.kubernetes.io/version` | Версия |
| `app.kubernetes.io/component` | Компонент (api, db, cache) |
| `app.kubernetes.io/part-of` | Часть чего |
| `app.kubernetes.io/managed-by` | Кто управляет (helm, kustomize) |

### 🎯 Selectors

**Selector** — это запрос по labels. Используется в:

- **Service** — какие Pod'ы обслуживать.
- **Deployment** — какие Pod'ы управляются.
- **NetworkPolicy** — какие Pod'ы изолировать.
- **kubectl** — фильтрация.

**Equality-based:**

```yaml
selector:
  matchLabels:
    app: myapp
    version: v1.0.0
```

**Set-based:**

```yaml
selector:
  matchExpressions:
    - key: environment
      operator: In
      values: [production, staging]
    - key: tier
      operator: NotIn
      values: [frontend]
    - key: critical
      operator: Exists
```

**Операторы:**

| Оператор | Что означает |
|:---|:---|
| `In` | Значение в списке |
| `NotIn` | Значение не в списке |
| `Exists` | Label существует (любое значение) |
| `DoesNotExist` | Label отсутствует |

### 🎯 Использование в манифестах

**Deployment → ReplicaSet → Pod'ы:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:                    # ← Deployment находит Pod'ы по этим labels
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp             # ← Pod'ы имеют эти labels
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
```

**Как связаны:**

- Deployment имеет `selector: {app: myapp}`.
- Pod'ы имеют `labels: {app: myapp}`.
- Deployment **управляет** Pod'ами с этими labels.

**Service → Pod'ы:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:                    # ← Service находит Pod'ы по этим labels
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
```

**Как связаны:**

- Service имеет `selector: {app: myapp}`.
- Pod'ы имеют `labels: {app: myapp}`.
- Service **направляет трафик** на эти Pod'ы.

**Важно:** Service и Deployment **не знают друг о друге**. Они оба смотрят на labels Pod'ов. Это **слабая связь** — легко менять.

### 🎯 Annotations

**Annotations** — метаданные **для инструментов**, не для идентификации.

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    description: "Main API service"
    contact: "team-payments@example.com"
```

**Отличия от labels:**

| | Labels | Annotations |
|:---|:---|:---|
| Для чего | Идентификация, выборка | Метаданные для инструментов |
| Размер | До 63 символов | До 256 KB |
| Селекторы | Да | Нет |
| Примеры | `app`, `version` | `prometheus.io/scrape`, `nginx.ingress...` |

**Правило:**

- **Labels** — для идентификации. Используются в selectors.
- **Annotations** — для метаданных. Не используются в selectors.

### 🔬 Практика: работа с labels

```bash
# 1. Создать Deployment
kubectl create deployment nginx --image=nginx --replicas=3

# 2. Посмотреть Pod'ы с labels
kubectl get pods --show-labels
# NAME                     READY   STATUS    LABELS
# nginx-7d9f8c6b4d-abc12   1/1     Running   app=nginx,pod-template-hash=7d9f8c6b4d

# 3. Фильтрация по label
kubectl get pods -l app=nginx
kubectl get pods -l app=nginx,version=v1.0.0

# 4. Set-based selector
kubectl get pods -l 'environment in (production,staging)'

# 5. Добавить label Pod'у
kubectl label pod nginx-7d9f8c6b4d-abc12 environment=production

# 6. Удалить label
kubectl label pod nginx-7d9f8c6b4d-abc12 environment-

# 7. Посмотреть labels Deployment
kubectl get deployment nginx -o jsonpath='{.metadata.labels}'
```

### 💡 Практика: как правильно использовать labels и annotations

**✅ ОБЯЗАТЕЛЬНО:**

1. **Используй `app` label** для идентификации приложения:
   ```yaml
   labels:
     app: myapp
   ```

2. **Используй `app.kubernetes.io/*` labels** — это стандарт:
   ```yaml
   labels:
     app.kubernetes.io/name: myapp
     app.kubernetes.io/version: "1.0.0"
     app.kubernetes.io/component: api
   ```

3. **Annotations для метаданных**, не для селекторов:
   ```yaml
   annotations:
     prometheus.io/scrape: "true"
   ```

**👍 СТОИТ:**

4. **Добавляй `version` label** для канареечных деплоев.
5. **Добавляй `environment` label** для разделения окружений.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй имена ресурсов для связи.** Только labels.
7. **Не используй annotations в selectors.** Они для этого не предназначены.
8. **Не дублируй информацию** в labels и annotations.

### Где мы сейчас

Мы разобрали labels, selectors, annotations. Теперь — **workloads** — какие типы ресурсов управляют Pod'ами.

---

## 8.6 Workloads: Deployment, ReplicaSet, DaemonSet, Job, CronJob, StatefulSet

### 🔌 Проблема: Pod'ы не управляются сами

Голый Pod не перезапустится при падении, не масштабируется, не обновляется. Нужны **workloads** — контроллеры, которые управляют Pod'ами.

Kubernetes имеет несколько типов workloads для разных задач.

### 📊 Обзор workloads

| Workload | Для чего | Особенности |
|:---|:---|:---|
| **Deployment** | Stateless-приложения | Rolling update, scaling, rollback |
| **ReplicaSet** | Управление Pod'ами (низкоуровневый) | Используется Deployment'ом |
| **DaemonSet** | Один Pod на каждую ноду | Логирование, мониторинг, сеть |
| **Job** | Одноразовая задача | Завершается после выполнения |
| **CronJob** | Задача по расписанию | Как cron, но в K8s |
| **StatefulSet** | Stateful-приложения | Стабильные имена, PVC на Pod |

### 🚀 Deployment

**Самый популярный workload.** Для stateless-приложений.

**Что делает:**

- Управляет ReplicaSet'ами.
- Обеспечивает rolling update.
- Позволяет масштабировать (`replicas`).
- Позволяет откатываться (`rollout undo`).

**Пример:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
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
          image: myapp:1.0.0
          ports:
            - containerPort: 8080
```

**Иерархия:**

```
Deployment: myapp
    │
    │ создаёт
    ▼
ReplicaSet: myapp-7d9f8c6b4d
    │
    │ создаёт
    ▼
Pod, Pod, Pod
```

**Что делает Deployment при обновлении:**

1. Создаёт **новый** ReplicaSet с новым шаблоном.
2. Постепенно масштабирует новый ReplicaSet.
3. Постепенно уменьшает старый ReplicaSet.
4. Старый ReplicaSet остаётся в истории (для rollback).

Разбирали подробно в Главе 7.

### 🔧 ReplicaSet

**Низкоуровневый workload.** Обеспечивает, что всегда есть N Pod'ов.

**Обычно не используется напрямую.** Deployment создаёт и управляет ReplicaSet'ами.

**Пример:**

```yaml
apiVersion: apps/v1
kind: ReplicaSet
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
          image: myapp:1.0.0
```

**Ограничения по сравнению с Deployment:**

- Нет rolling update.
- Нет rollback.
- Нет истории ревизий.

**Когда использовать:** почти никогда. Используй Deployment.

### 🖥️ DaemonSet

**Один Pod на каждую ноду.**

**Для чего:**

- **Логирование** (fluent-bit, filebeat) — на каждой ноде.
- **Мониторинг** (node-exporter) — на каждой ноде.
- **Сеть** (Calico, Cilium) — CNI-плагины.
- **Storage** (CSI-драйверы).

**Пример:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true       # использовать сеть ноды
      hostPID: true           # видеть процессы ноды
      containers:
        - name: node-exporter
          image: prom/node-exporter:latest
          ports:
            - containerPort: 9100
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
```

**Что произошло:**

- Kubernetes создаст Pod `node-exporter` на **каждой** ноде.
- Новая нода добавится — Pod создастся автоматически.
- Нода удалится — Pod удалится.

**Когда использовать:**

- **Логирование** — собрать логи со всех нод.
- **Мониторинг** — метрики нод.
- **Сеть** — CNI-плагины.
- **Storage** — CSI-драйверы.

### 📋 Job

**Одноразовая задача.** Выполнилась — завершилась.

**Для чего:**

- **Миграции БД.**
- **Batch processing.**
- **Setup/teardown задачи.**

**Пример:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 3           # попыток при ошибке
  completions: 1            # сколько успешных завершений нужно
  parallelism: 1            # сколько Pod'ов одновременно
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: myapp:1.0.0
          command: ["./myapp", "migrate", "up"]
```

**Что произошло:**

- Kubernetes запустит Pod.
- Pod выполнит `./myapp migrate up` и завершится.
- Если упадёт — попробует ещё раз (до `backoffLimit`).
- Когда успешно завершится — Job Completed.

**Статусы:**

- `Pending` — ещё не запущен.
- `Running` — Pod работает.
- `Completed` — успешно завершён.
- `Failed` — упал после всех попыток.

**Когда использовать:**

- Миграции БД.
- Одноразовые задачи.
- Batch processing.

### ⏰ CronJob

**Job по расписанию.** Как cron, но в Kubernetes.

**Пример:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup
spec:
  schedule: "0 2 * * *"      # каждый день в 2 ночи
  concurrencyPolicy: Forbid   # не запускать, если предыдущий ещё работает
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: busybox
              command: ["/bin/sh", "-c", "echo 'Cleanup done'"]
```

**Поля:**

| Поле | Что означает |
|:---|:---|
| `schedule` | Cron-выражение |
| `concurrencyPolicy` | `Allow`, `Forbid`, `Replace` |
| `successfulJobsHistoryLimit` | Сколько успешных Job'ов хранить |
| `failedJobsHistoryLimit` | Сколько неудачных Job'ов хранить |

**Когда использовать:**

- Бэкапы.
- Cleanup старых данных.
- Отчёты по расписанию.
- Периодические проверки.

### 💾 StatefulSet

**Для stateful-приложений.** Обеспечивает:

- **Стабильные имена Pod'ов** (`myapp-0`, `myapp-1`, `myapp-2`).
- **Стабильные сетевые идентификаторы** (headless Service).
- **Стабильное хранилище** (PVC на Pod).
- **Порядок запуска и остановки** (0, 1, 2 — по порядку).

**Пример:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres       # headless service
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
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:        # PVC для каждого Pod'а
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 10Gi
```

**Что произошло:**

- Создадутся Pod'ы: `postgres-0`, `postgres-1`, `postgres-2`.
- Каждый получит свой PVC: `data-postgres-0`, `data-postgres-1`, `data-postgres-2`.
- Headless Service `postgres` даст DNS: `postgres-0.postgres`, `postgres-1.postgres`, `postgres-2.postgres`.
- Pod'ы запускаются **по порядку** (0, потом 1, потом 2).

**Когда использовать:**

- **Базы данных** (PostgreSQL, MySQL, MongoDB).
- **Брокеры** (Kafka, RabbitMQ, Zookeeper).
- **Распределённые системы** (Elasticsearch, Cassandra).

Подробно разберём в Главе 12.

### 📊 Сравнение workloads

| Workload | Pod'ов | Имена | Storage | Порядок |
|:---|:---|:---|:---|:---|
| **Deployment** | N реплик | Случайные | Shared | Параллельно |
| **ReplicaSet** | N реплик | Случайные | Shared | Параллельно |
| **DaemonSet** | 1 на ноду | С суффиксом ноды | Нода | По мере добавления нод |
| **Job** | 1..N | Случайные | Shared | Параллельно/последовательно |
| **CronJob** | 1..N | Случайные | Shared | По расписанию |
| **StatefulSet** | N реплик | `app-0`, `app-1` | Свой PVC | По порядку |

### 💡 Практика: как выбрать workload

**✅ ОБЯЗАТЕЛЬНО:**

1. **Для stateless — Deployment.** 95% случаев.
2. **Для stateful (БД) — StatefulSet.**
3. **Для задач на каждой ноде — DaemonSet.**
4. **Для одноразовых задач — Job.**
5. **Для периодических — CronJob.**

**👍 СТОИТ:**

6. **Не использовать ReplicaSet напрямую.** Deployment делает всё то же + rolling update.

**❌ НЕ ДЕЛАЙ:**

7. **Не использовать Deployment для БД.** Нужны стабильные имена и storage.
8. **Не использовать Pod напрямую.** Без контроллера не восстановится.

### Где мы сейчас

Мы разобрали workloads. Теперь — **Services** — как Pod'ы находят друг друга.

---

## 8.7 Services: как Pod'ы находят друг друга

### 🔌 Проблема: IP Pod'ов меняется

Pod'ы эфемерны. При пересоздании Pod получает **новый IP**. Если приложение A обращается к приложению B по IP — после пересоздания Pod'а B приложение A сломается.

**Решение:** Service.

### 📦 Что такое Service

**Service** — это абстракция, которая предоставляет **стабильный IP** и **DNS-имя** для группы Pod'ов.

```
Без Service:                    С Service:
                                
App ──► 10.1.1.5:8080          App ──► myapp:80 (ClusterIP)
        (IP Pod'а)                      │
        (меняется!)                     │ kube-proxy
                                        ▼
                                 10.1.1.5:8080  (Pod 1)
                                 10.1.1.6:8080  (Pod 2)
                                 10.1.1.7:8080  (Pod 3)
```

**Service:**

- Имеет **стабильный ClusterIP** (виртуальный IP).
- Имеет **DNS-имя** (`myapp.default.svc.cluster.local`).
- Автоматически **находит Pod'ы** по selector.
- **Балансирует** трафик между Pod'ами.

### 🎯 Типы Service

| Тип | Что делает | Когда использовать |
|:---|:---|:---|
| **ClusterIP** | Виртуальный IP внутри кластера | Внутренние сервисы |
| **NodePort** | Порт на каждой ноде | Dev, простые случаи |
| **LoadBalancer** | Внешний LB от облака | Production в облаке |
| **ExternalName** | CNAME на внешний сервис | Интеграция с внешним |

**ClusterIP (по умолчанию):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - port: 80          # порт Service
      targetPort: 8080  # порт Pod'а
```

**Как работает:**

1. Service получает ClusterIP (например, `10.96.0.5`).
2. kube-proxy настраивает правила: «трафик на `10.96.0.5:80` → один из Pod'ов на порт 8080».
3. Pod'ы могут обращаться к `myapp:80` или `10.96.0.5:80`.
4. Балансировка — round-robin по Pod'ам.

**NodePort:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080   # порт на каждой ноде (30000-32767)
```

**Как работает:**

- На каждой ноде открывается порт `30080`.
- `node-1:30080`, `node-2:30080`, `node-3:30080` — все ведут на Pod'ы.

**LoadBalancer:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
```

**Как работает:**

- Kubernetes просит облачного провайдера создать внешний LB.
- LB получает внешний IP.
- Трафик идёт: LB → NodePort → Pod.

**ExternalName:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com
```

**Как работает:**

- DNS-запрос к `external-db` возвращает CNAME на `db.example.com`.
- Нет ClusterIP, нет proxy.

### 🎯 Service discovery

Kubernetes имеет встроенный **DNS** (CoreDNS). Каждый Service получает DNS-имя:

```
<service>.<namespace>.svc.cluster.local
```

**Пример:**

- Service `myapp` в namespace `default`:
  - `myapp` (внутри того же namespace)
  - `myapp.default` (из другого namespace)
  - `myapp.default.svc` (сокращённо)
  - `myapp.default.svc.cluster.local` (полное)

**Внутри Pod'а:**

```bash
# Обратиться к Service по короткому имени
curl http://myapp

# Или по полному
curl http://myapp.default.svc.cluster.local
```

**Как работает:**

1. CoreDNS — DNS-сервер в кластере.
2. Каждый Pod имеет `nameserver` на CoreDNS.
3. CoreDNS резолвит `<service>.<namespace>` в ClusterIP.
4. kube-proxy настраивает правила для маршрутизации.

### 🎯 Headless Service

**Headless Service** — Service без ClusterIP (`clusterIP: None`). Возвращает IP **всех** Pod'ов, не балансирует.

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

**DNS для headless:**

- `postgres.default.svc.cluster.local` → IP **всех** Pod'ов.
- `postgres-0.postgres.default.svc.cluster.local` → IP Pod'а 0 (с StatefulSet).

**Когда использовать:**

- **StatefulSet** (для прямого доступа к Pod'ам).
- **Кастомные балансировщики** (клиент сам выбирает Pod).
- **Peer discovery** (например, Kafka broker discovery).

### 🎯 Endpoints

**Endpoints** — это объект, который хранит **IP'ы Pod'ов**, соответствующих Service'у.

```bash
kubectl get endpoints myapp
# NAME    ENDPOINTS                              AGE
# myapp   10.1.1.5:8080,10.1.1.6:8080,10.1.1.7:8080   5m
```

**Как работают:**

1. Endpoint Controller смотрит на Service и его selector.
2. Находит Pod'ы с соответствующими labels.
3. Записывает их IP в Endpoints.
4. kube-proxy читает Endpoints и настраивает маршрутизацию.

**Endpoints обновляются автоматически** при:
- Добавлении Pod'а.
- Удалении Pod'а.
- Изменении readiness Pod'а.

### 🔬 Практика: Service в действии

```bash
# 1. Создать Deployment
kubectl create deployment nginx --image=nginx --replicas=3

# 2. Создать Service
kubectl expose deployment nginx --port=80 --target-port=80

# 3. Посмотреть Service
kubectl get services
# NAME    TYPE        CLUSTER-IP      PORT(S)
# nginx   ClusterIP   10.96.123.45    80/TCP

# 4. Посмотреть Endpoints
kubectl get endpoints nginx
# NAME    ENDPOINTS                                AGE
# nginx   10.1.1.5:80,10.1.1.6:80,10.1.1.7:80     10s

# 5. Проверить DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
# Внутри:
nslookup nginx
# Server:    10.96.0.10
# Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
# 
# Name:      nginx
# Address 1: 10.96.123.45 nginx.default.svc.cluster.local

# Обратиться к Service
wget -qO- http://nginx
# <!DOCTYPE html>...  (HTML от nginx)
```

### 💡 Практика: как правильно работать с Services

**✅ ОБЯЗАТЕЛЬНО:**

1. **Используй ClusterIP для внутренних сервисов.**
2. **Используй LoadBalancer для внешних** (в облаке).
3. **Используй headless для StatefulSet.**

**👍 СТОИТ:**

4. **Не открывай NodePort в production** — используй LoadBalancer или Ingress.
5. **Используй Ingress** (Глава 10) для HTTP — один IP на много сервисов.

**❌ НЕ ДЕЛАЙ:**

6. **Не обращайся к Pod'ам по IP.** Используй Service.
7. **Не используй `NodePort` в облаке.** LoadBalancer лучше.

### Где мы сейчас

Мы разобрали Services. Теперь — **namespaces** — виртуальные кластеры.

---

## 8.8 Namespaces: виртуальные кластеры

### 🔌 Проблема: как разделить ресурсы между командами

В одном кластере работают:

- Команда A — микросервисы платежей.
- Команда B — микросервисы пользователей.
- Инфраструктура — monitoring, logging, ingress.

Как разделить ресурсы? Как ограничить доступ? Как избежать конфликтов имён?

**Решение:** namespaces.

### 📦 Что такое namespace

**Namespace** — это логическое разделение кластера. Ресурсы внутри namespace изолированы от других namespaces (по именам).

**Дефолтные namespaces:**

| Namespace | Что там |
|:---|:---|
| `default` | Ресурсы по умолчанию |
| `kube-system` | Системные компоненты (kube-dns, kube-proxy) |
| `kube-public` | Публичные ресурсы (cluster-info) |
| `kube-node-lease` | Lease-объекты для нод |

**Создать namespace:**

```bash
kubectl create namespace payments
kubectl create namespace users
kubectl create namespace monitoring
```

**Или через YAML:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
```

### 🎯 Что изолируется

**Внутри namespace:**

- **Имена уникальны.** Два Pod'а с именем `myapp` могут существовать в разных namespaces.
- **DNS.** Service `myapp` в namespace `payments` → `myapp.payments.svc.cluster.local`.
- **RBAC.** Можно ограничить доступ пользователей к namespace.
- **Resource Quotas.** Можно ограничить ресурсы namespace.
- **Network Policies.** Можно изолировать сеть между namespaces.

**Что НЕ изолируется:**

- **Ноды.** Общие для всего кластера.
- **Storage Classes.** Общие.
- **ClusterRole, ClusterRoleBinding.** Кластерные.

### 🎯 Как использовать namespace

**В манифесте:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: payments    # ← namespace
spec:
  containers:
    - name: myapp
      image: myapp:1.0
```

**В kubectl:**

```bash
# По умолчанию — namespace "default"
kubectl get pods

# Явно указать namespace
kubectl get pods -n payments

# Все namespaces
kubectl get pods --all-namespaces

# Сменить namespace для текущего контекста
kubectl config set-context --current --namespace=payments
```

### 🎯 Resource Quotas

**ResourceQuota** ограничивает ресурсы namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payments-quota
  namespace: payments
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"
```

**Что произошло:**

- Namespace `payments` может запросить максимум 100 CPU, 200 GB RAM.
- Максимум 50 Pod'ов.
- Максимум 20 Service'ов.

**Зачем:** предотвращает ситуацию, когда одна команда съедает все ресурсы кластера.

### 🎯 LimitRange

**LimitRange** задаёт дефолтные лимиты для Pod'ов в namespace:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: payments-limits
  namespace: payments
spec:
  limits:
    - default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "2"
        memory: 2Gi
      min:
        cpu: 50m
        memory: 64Mi
      type: Container
```

**Что произошло:**

- Если Pod не указал `resources.limits.cpu` — будет 500m.
- Если не указал `requests` — будет 100m.
- Максимум — 2 CPU.
- Минимум — 50m.

### 🎯 Namespace для окружений

**Типичный паттерн:**

```
default          — ничего (оставляем пустым)
kube-system      — системные компоненты
monitoring       — Prometheus, Grafana, Loki
ingress-nginx    — Ingress Controller
payments-dev     — payments в dev
payments-staging — payments в staging
payments-prod    — payments в production
users-dev        — users в dev
users-prod       — users в production
```

**Каждое окружение — свой namespace.** Изоляция, quotas, RBAC.

### 🔬 Практика: namespace

```bash
# 1. Создать namespace
kubectl create namespace demo

# 2. Запустить Pod в namespace
kubectl run nginx --image=nginx -n demo

# 3. Посмотреть Pod'ы
kubectl get pods -n demo
# NAME    READY   STATUS
# nginx   1/1     Running

# 4. Pod'ы в default его не видят
kubectl get pods
# No resources found in default namespace

# 5. Посмотреть Service'ы во всех namespaces
kubectl get services --all-namespaces

# 6. Удалить namespace (удалит всё внутри!)
kubectl delete namespace demo
```

### 💡 Практика: как правильно использовать namespaces

**✅ ОБЯЗАТЕЛЬНО:**

1. **Один namespace на команду/приложение.**
2. **Один namespace на окружение** (dev, staging, prod).
3. **ResourceQuota** для каждого namespace команды.

**👍 СТОИТ:**

4. **LimitRange** для дефолтных лимитов.
5. **RBAC** для ограничения доступа.
6. **Network Policies** для изоляции трафика.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `default` namespace** для приложений.
8. **Не смешивай окружения в одном namespace.**
9. **Не удаляй namespace без понимания.** Удалит всё внутри.

### Где мы сейчас

Мы разобрали namespaces. Теперь — **kubectl** — основной инструмент.

---

## 8.9 kubectl: рабочий инструмент

### 🔌 Проблема: как управлять кластером

`kubectl` — это CLI для управления Kubernetes. Через него ты:

- Применяешь манифесты.
- Смотришь ресурсы.
- Диагностируешь проблемы.
- Удаляешь ресурсы.

### 📊 Основные команды

**Применение манифестов:**

```bash
# Применить манифест
kubectl apply -f deployment.yaml

# Применить всё в директории
kubectl apply -f k8s/

# Применить по URL
kubectl apply -f https://example.com/manifest.yaml

# Удалить
kubectl delete -f deployment.yaml

# Dry-run (проверка без применения)
kubectl apply -f deployment.yaml --dry-run=client

# Diff — что изменится
kubectl diff -f deployment.yaml
```

**Просмотр ресурсов:**

```bash
# Список Pod'ов
kubectl get pods

# Подробно
kubectl get pods -o wide

# В YAML
kubectl get pod nginx -o yaml

# В JSON
kubectl get pod nginx -o json

# Определённые поля
kubectl get pod nginx -o jsonpath='{.status.podIP}'

# Кастомные колонки
kubectl get pods -o custom-columns=NAME:.metadata.name,IP:.status.podIP

# Watch — обновления в реальном времени
kubectl get pods -w

# Все ресурсы
kubectl get all

# С фильтром по label
kubectl get pods -l app=nginx

# Сортировка
kubectl get pods --sort-by=.metadata.creationTimestamp
```

**Описание ресурсов:**

```bash
# Подробное описание (события, статус)
kubectl describe pod nginx

# События кластера
kubectl get events

# События для конкретного ресурса
kubectl describe pod nginx | grep -A 10 Events
```

**Логи:**

```bash
# Логи Pod'а
kubectl logs nginx

# Follow (в реальном времени)
kubectl logs nginx -f

# Последние N строк
kubectl logs nginx --tail=100

# С таймстампами
kubectl logs nginx --timestamps

# Для предыдущего экземпляра (после падения)
kubectl logs nginx --previous

# Логи конкретного контейнера в Pod'е
kubectl logs nginx -c sidecar

# Логи всех Pod'ов Deployment'а
kubectl logs -l app=nginx --all-containers=true
```

**Exec и debug:**

```bash
# Зайти в Pod
kubectl exec -it nginx -- bash

# Выполнить команду
kubectl exec nginx -- ls -la

# Скопировать файл из Pod'а
kubectl cp nginx:/etc/nginx/nginx.conf ./nginx.conf

# Запустить debug-Pod с нужными инструментами
kubectl debug -it nginx --image=nicolaka/netshoot
```

**Масштабирование:**

```bash
# Изменить replicas
kubectl scale deployment nginx --replicas=5

# Autoscaling
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80
```

**Обновление:**

```bash
# Изменить образ
kubectl set image deployment/nginx nginx=nginx:1.26

# Rollout status
kubectl rollout status deployment/nginx

# История
kubectl rollout history deployment/nginx

# Откат
kubectl rollout undo deployment/nginx

# Откат на конкретную ревизию
kubectl rollout undo deployment/nginx --to-revision=2

# Перезапуск (rolling)
kubectl rollout restart deployment/nginx
```

**Порты:**

```bash
# Проброс порта Pod'а на локальную машину
kubectl port-forward pod/nginx 8080:80

# Проброс Service
kubectl port-forward service/nginx 8080:80

# Проброс Deployment
kubectl port-forward deployment/nginx 8080:80
```

**Контексты:**

```bash
# Список контекстов
kubectl config get-contexts

# Текущий контекст
kubectl config current-context

# Переключить контекст
kubectl config use-context prod-cluster

# Переключить namespace
kubectl config set-context --current --namespace=payments
```

### 🎯 Namespace в командах

```bash
# Явно указать namespace
kubectl get pods -n payments

# Все namespaces
kubectl get pods --all-namespaces
kubectl get pods -A

# Сменить namespace по умолчанию
kubectl config set-context --current --namespace=payments
```

### 🎯 Полезные флаги

| Флаг | Что делает |
|:---|:---|
| `-o wide` | Больше колонок |
| `-o yaml/json` | Вывод в YAML/JSON |
| `-o jsonpath` | Извлечь поле |
| `-o custom-columns` | Свои колонки |
| `-w` | Watch (обновления) |
| `-l` | Фильтр по label |
| `--all-namespaces` | Все namespaces |
| `-n` | Namespace |
| `--dry-run=client` | Проверка без применения |
| `--previous` | Предыдущий экземпляр контейнера |
| `-f` | Из файла |
| `--tail` | Последние N строк |

### 🎯 Namespaces в kubectl

```bash
# Список namespaces
kubectl get namespaces

# Все ресурсы в namespace
kubectl get all -n payments

# Создать namespace
kubectl create namespace payments

# Удалить namespace (со всем содержимым!)
kubectl delete namespace payments
```

### 💡 Практика: как правильно использовать kubectl

**✅ ОБЯЗАТЕЛЬНО:**

1. **`kubectl apply`** для применения, не `create`. `apply` идемпотентен.
2. **`kubectl describe`** для диагностики — показывает события.
3. **`kubectl logs --previous`** для упавших Pod'ов.
4. **`kubectl diff`** перед применением — увидеть, что изменится.

**👍 СТОИТ:**

5. **`kubectl rollout status`** после деплоя.
6. **`-o yaml`** для понимания структуры ресурса.
7. **`kubectl debug`** для отладки (с K8s 1.25+).
8. **`kubectl port-forward`** для доступа к внутренним сервисам.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

9. **K9s, Lens** — GUI для kubectl.

**❌ НЕ ДЕЛАЙ:**

10. **Не используй `kubectl edit` для production.** Изменения не отслеживаются в Git.
11. **Не используй `kubectl delete` без `--dry-run`.** Можно удалить лишнее.
12. **Не игнорируй `-n`.** Можно случайно изменить не то.

### Где мы сейчас

Мы разобрали kubectl. Теперь — **диагностика** проблем с Pod'ами.

---

## 8.10 Диагностика: Pod не запускается

### 🔌 Проблема: Pod в непонятном статусе

Ты применил манифест. Pod не запускается. Статус — что-то непонятное.

Разберём **типичные проблемы** и **алгоритмы диагностики**.

### 📊 Типичные статусы Pod'ов

| Статус | Что означает |
|:---|:---|
| `Pending` | Не назначен на ноду или не может быть запущен |
| `ContainerCreating` | Образ скачивается, контейнер запускается |
| `Running` | Работает |
| `CrashLoopBackOff` | Контейнер падает и перезапускается |
| `ImagePullBackOff` | Не может скачать образ |
| `ErrImagePull` | Ошибка при скачивании образа |
| `Error` | Завершился с ошибкой |
| `Completed` | Завершился успешно (для Job) |
| `Terminating` | Удаляется |
| `Unknown` | Нода недоступна |

### 🔍 Алгоритм диагностики

**Шаг 1: Статус**

```bash
kubectl get pods
kubectl get pods -o wide    # + нода, IP
```

**Шаг 2: Описание**

```bash
kubectl describe pod <name>
```

Смотри:

- **Events** внизу — что происходит.
- **State** контейнера.
- **Conditions** Pod'а.
- **Node** — на какой ноде.

**Шаг 3: Логи**

```bash
kubectl logs <name>
kubectl logs <name> --previous    # если упал
kubectl logs <name> -c <container>  # конкретный контейнер
```

**Шаг 4: События**

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events -n <namespace>
```

### 🎯 Pending

**Причины:**

**1. Недостаточно ресурсов.**

```
Events:
  Type     Reason            Message
  ----     ------            -------
  Warning  FailedScheduling  0/3 nodes are available: 3 Insufficient cpu.
```

**Решение:**

- Уменьшить `resources.requests.cpu`.
- Добавить ноды.
- Удалить другие Pod'ы.

**2. Нет нод с нужными labels.**

```
Events:
  Warning  FailedScheduling  node(s) didn't match node selector.
```

**Решение:** проверить `nodeSelector`, `affinity`.

**3. PVC не готов.**

```
Events:
  Warning  FailedScheduling  pod has unbound immediate PersistentVolumeClaims.
```

**Решение:** проверить PVC — есть ли доступный PV.

**4. Taints/Tolerations.**

```
Events:
  Warning  FailedScheduling  node(s) had taints that the pod didn't tolerate.
```

**Решение:** добавить tolerations.

### 🎯 ImagePullBackOff / ErrImagePull

**Причины:**

**1. Образ не существует.**

```
Events:
  Warning  Failed  Failed to pull image "myregistry.com/myapp:v99": not found
```

**Решение:** проверить имя и тег.

**2. Нет доступа к приватному registry.**

```
Events:
  Warning  Failed  Failed to pull image: unauthorized
```

**Решение:** добавить `imagePullSecrets`.

```yaml
spec:
  imagePullSecrets:
    - name: my-registry-secret
  containers:
    - name: myapp
      image: myregistry.com/myapp:v1.0.0
```

Создать secret:

```bash
kubectl create secret docker-registry my-registry-secret \
  --docker-server=myregistry.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=my@email.com
```

**3. Опечатка в имени registry.**

**4. Сеть недоступна** — нода не может достать registry.

### 🎯 CrashLoopBackOff

**Что означает:** контейнер запускается, падает, Kubernetes перезапускает. Снова падает. Снова перезапускает.

**Причины:**

**1. Ошибка в приложении.**

```bash
kubectl logs <pod> --previous
# panic: runtime error: ...
```

**2. Ошибка в конфигурации.**

```bash
kubectl logs <pod> --previous
# Error: config file not found: /etc/app/config.yaml
```

**3. Не может подключиться к БД.**

```bash
kubectl logs <pod> --previous
# Error: dial tcp postgres:5432: connect: connection refused
```

**4. Проблемы с правами.**

```bash
kubectl logs <pod> --previous
# Permission denied: /data/app.log
```

**Решение:**

- Посмотреть логи.
- Понять, почему приложение падает.
- Исправить код / конфигурацию / окружение.

**Диагностика через debug-Pod:**

```bash
# Если приложение падает слишком быстро
kubectl debug -it <pod> --image=nicolaka/netshoot --copy-to=myapp-debug
```

### 🎯 ContainerCreating (зависает)

**Причины:**

**1. Долго скачивается образ** (большой образ, медленная сеть).

**2. Ожидание mount volume** (например, NFS).

**3. CNI не может назначить IP.**

**Диагностика:**

```bash
kubectl describe pod <name>
# Смотри Events
```

### 🎯 Terminating (зависает)

**Причины:**

**1. Finalizers не снимаются.**

```bash
kubectl get pod <name> -o yaml | grep finalizers
```

**2. `terminationGracePeriodSeconds` слишком большой.**

**3. Проблемы с CNI.**

**Решение (осторожно!):**

```bash
# Форсированное удаление
kubectl delete pod <name> --grace-period=0 --force
```

**Внимание:** может привести к проблемам с сетью и storage.

### 🔬 Практика: диагностика

```bash
# 1. Pod в CrashLoopBackOff
kubectl get pods
# NAME    READY   STATUS             RESTARTS
# myapp   0/1     CrashLoopBackOff   5

# 2. Описание
kubectl describe pod myapp
# Events:
#   Warning  BackOff  Back-off restarting failed container

# 3. Логи предыдущего экземпляра
kubectl logs myapp --previous
# Error: cannot connect to database: dial tcp postgres:5432: connection refused

# 4. Проверить, что Service postgres существует
kubectl get service postgres

# 5. Проверить, что Pod'ы postgres running
kubectl get pods -l app=postgres

# 6. Если нет — проблема с postgres, не с myapp
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Начинай с `kubectl get pods -o wide`.** Видишь ноду, IP, статус.
2. **`kubectl describe pod`** — Events внизу.
3. **`kubectl logs --previous`** — логи упавшего.
4. **`kubectl get events`** — общие события кластера.

**👍 СТОИТ:**

5. **`kubectl debug`** — запустить Pod с инструментами для отладки.
6. **`kubectl exec`** — зайти в работающий Pod.
7. **Мониторинг и логи** — Grafana, Loki, Prometheus.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

8. **K9s, Lens** — GUI для диагностики.

**❌ НЕ ДЕЛАЙ:**

9. **Не удаляй Pod'ы без понимания.** Deployment создаст новые.
10. **Не игнорируй Events.** Они — главный источник информации.
11. **Не удаляй PVC без понимания.** Может быть с данными.

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **Kubernetes** | Оркестратор контейнеров. |
| **Cluster** | Набор машин, работающих как одна. |
| **Control plane** | Управляющие компоненты (apiserver, etcd, scheduler, controller-manager). |
| **Worker node** | Рабочая машина, где запускаются Pod'ы. |
| **kube-apiserver** | REST API, единственная точка входа. |
| **etcd** | Key-value хранилище состояния кластера. |
| **kube-scheduler** | Распределяет Pod'ы по нодам. |
| **kube-controller-manager** | Набор контроллеров. |
| **kubelet** | Агент на ноде, запускает контейнеры. |
| **kube-proxy** | Сетевой агент, реализует Services. |
| **Container Runtime** | containerd, CRI-O — запускает контейнеры. |
| **Pod** | Минимальная единица деплоя. Группа контейнеров. |
| **Deployment** | Workload для stateless-приложений. |
| **ReplicaSet** | Управляет N Pod'ами. Используется Deployment'ом. |
| **DaemonSet** | Один Pod на каждую ноду. |
| **Job** | Одноразовая задача. |
| **CronJob** | Задача по расписанию. |
| **StatefulSet** | Workload для stateful-приложений. |
| **Service** | Стабильный IP и DNS для группы Pod'ов. |
| **ClusterIP** | Внутренний IP Service'а. |
| **NodePort** | Порт на каждой ноде. |
| **LoadBalancer** | Внешний LB от облака. |
| **Headless Service** | Service без ClusterIP. |
| **Endpoints** | IP'ы Pod'ов для Service'а. |
| **Labels** | Key-value для идентификации ресурсов. |
| **Selectors** | Запросы по labels. |
| **Annotations** | Метаданные для инструментов. |
| **Namespace** | Логическое разделение кластера. |
| **ResourceQuota** | Ограничение ресурсов namespace. |
| **LimitRange** | Дефолтные лимиты для Pod'ов. |
| **Reconciliation loop** | Приведение системы к желаемому состоянию. |
| **kubectl** | CLI для управления Kubernetes. |

---

## Что мы узнали?

- **Kubernetes** — оркестратор контейнеров для production. Решает проблемы Compose: отказоустойчивость, масштабирование, rolling update.
- **Архитектура:** control plane (apiserver, etcd, scheduler, controller-manager) + worker nodes (kubelet, kube-proxy, container runtime).
- **Reconciliation loop** — сердце Kubernetes. Level-triggered, приводит систему к желаемому состоянию.
- **Pod** — минимальная единица деплоя. Группа контейнеров с общим network namespace и storage.
- **Labels, selectors, annotations** — как K8s связывает ресурсы. Labels для идентификации, annotations для метаданных.
- **Workloads:** Deployment (stateless), DaemonSet (на каждой ноде), Job (одноразовая), CronJob (по расписанию), StatefulSet (stateful).
- **Service** — стабильный IP и DNS для группы Pod'ов. ClusterIP, NodePort, LoadBalancer, Headless.
- **Namespace** — логическое разделение. ResourceQuota, LimitRange, RBAC.
- **kubectl** — основной инструмент. apply, get, describe, logs, exec, rollout.
- **Диагностика:** Pending, ImagePullBackOff, CrashLoopBackOff — типичные проблемы и алгоритмы.

---

## Типичные ошибки

- ❌ **Создавать Pod'ы напрямую.** Без контроллера не восстановятся.
- ❌ **Использовать `kubectl run` в production.** Только для отладки.
- ❌ **Обращаться к Pod'ам по IP.** IP меняется. Используй Service.
- ❌ **Не указывать `resources.requests/limits`.** Scheduler не знает, сколько нужно.
- ❌ **Использовать `default` namespace для приложений.**
- ❌ **Не настраивать ResourceQuota.** Одна команда может съесть всё.
- ❌ **Игнорировать labels.** Без них Service не найдёт Pod'ы.
- ❌ **Использовать `kubectl edit` в production.** Изменения не в Git.
- ❌ **Не использовать `kubectl describe` при диагностике.** Events — ключ.
- ❌ **Забывать про `-n namespace`.** Можно изменить не то.
- ❌ **Не читать логи упавших Pod'ов через `--previous`.**
- ❌ **Удалять namespace без понимания.** Удалит всё внутри.

---

## Для быстрого повторения

- **Архитектура:** control plane (apiserver, etcd, scheduler, controller-manager) + workers (kubelet, kube-proxy, runtime).
- **Reconciliation loop:** level-triggered, приводит к желаемому состоянию.
- **Pod:** минимальная единица. IP, volumes, sidecar, init.
- **Labels:** `app`, `version`, `app.kubernetes.io/name`. Selectors для связи.
- **Workloads:** Deployment (stateless), DaemonSet (на ноде), Job (одноразовая), CronJob (расписание), StatefulSet (stateful).
- **Service:** ClusterIP (внутренний), NodePort, LoadBalancer, Headless.
- **DNS:** `<service>.<namespace>.svc.cluster.local`.
- **Namespace:** логическое разделение. ResourceQuota, LimitRange.
- **kubectl:** `apply`, `get`, `describe`, `logs`, `exec`, `rollout`, `debug`, `port-forward`.
- **Диагностика:** `get pods -o wide`, `describe`, `logs --previous`, `get events`.

---

## Вопросы для самопроверки

1. Из каких компонентов состоит control plane? Что делает каждый?
2. Что такое worker node? Какие компоненты на ней?
3. Что такое reconciliation loop? Чем отличается level-triggered от edge-triggered?
4. Что произойдёт от `kubectl apply` до запущенного Pod'а? Опиши пошагово.
5. Что такое Pod? Чем отличается от контейнера?
6. Зачем нужны sidecar-контейнеры? Приведи примеры.
7. Что такое labels и selectors? Как Deployment находит свои Pod'ы?
8. Чем labels отличаются от annotations?
9. Назови пять типов workloads. Когда использовать каждый?
10. Что такое Service? Какие типы существуют?
11. Что такое headless Service? Когда использовать?
12. Что такое namespace? Как изолирует ресурсы?
13. Как диагностировать Pod в статусе `CrashLoopBackOff`?
14. Что означает `ImagePullBackOff`? Как исправить?
15. Что такое `kubectl port-forward`? Зачем нужен?

---

## Ответы

**1. Control plane**

- **kube-apiserver** — REST API, единственная точка входа, аутентификация, авторизация, admission.
- **etcd** — key-value хранилище состояния.
- **kube-scheduler** — распределяет Pod'ы по нодам (filters + scores).
- **kube-controller-manager** — набор контроллеров (Deployment, ReplicaSet, Node, Endpoint).

**2. Worker node**

Компоненты: **kubelet** (агент, запускает контейнеры), **kube-proxy** (сетевой агент, реализует Services), **container runtime** (containerd, CRI-O).

**3. Reconciliation loop**

Бесконечный цикл: получить желаемое состояние, получить текущее, сравнить, сделать шаги для сближения. **Level-triggered** — смотрит на состояние, не на события. Даже если событие пропущено — приведёт к правильному состоянию.

**4. Путь от `kubectl apply` до Pod'а**

1. kubectl → apiserver (REST API).
2. apiserver: аутентификация, авторизация, validation, admission.
3. Запись в etcd.
4. Deployment Controller создаёт ReplicaSet.
5. ReplicaSet Controller создаёт Pod'ы (без нод).
6. Scheduler назначает ноды (filters + scores).
7. kubelet на ноде скачивает образ и запускает контейнер.
8. kubelet обновляет статус Pod (Running).
9. `kubectl get pods` показывает Running.

**5. Pod**

Pod — минимальная единица деплоя. Группа из 1+ контейнеров с общим network namespace, storage, lifecycle. Контейнер — процесс. Pod — обёртка над контейнерами.

**6. Sidecar**

Sidecar — контейнер, работающий рядом с основным. Примеры: логирование (fluent-bit), прокси (Istio), адаптер (Prometheus exporter), init-контейнеры (wait-for-db).

**7. Labels и selectors**

Labels — key-value. Selectors — запросы по labels (`matchLabels`, `matchExpressions`). Deployment имеет `selector: {app: myapp}`, Pod'ы имеют `labels: {app: myapp}`. Deployment управляет Pod'ами с этими labels.

**8. Labels vs annotations**

Labels — для идентификации и выборки (до 63 символов, используются в selectors). Annotations — метаданные для инструментов (до 256 KB, не используются в selectors).

**9. Пять workloads**

- **Deployment** — stateless, rolling update.
- **DaemonSet** — один Pod на каждую ноду.
- **Job** — одноразовая задача.
- **CronJob** — задача по расписанию.
- **StatefulSet** — stateful, стабильные имена и storage.

**10. Service**

Стабильный IP и DNS для группы Pod'ов. Типы: **ClusterIP** (внутренний), **NodePort** (порт на ноде), **LoadBalancer** (внешний LB), **ExternalName** (CNAME).

**11. Headless Service**

Service без ClusterIP (`clusterIP: None`). Возвращает IP всех Pod'ов. Для StatefulSet, кастомных балансировщиков, peer discovery.

**12. Namespace**

Логическое разделение кластера. Изолирует имена, DNS, RBAC, quotas. Ноды и storage classes — общие.

**13. CrashLoopBackOff**

1. `kubectl get pods` — увидеть статус.
2. `kubectl describe pod` — Events.
3. `kubectl logs <pod> --previous` — логи упавшего экземпляра.
4. Понять причину (ошибка кода, конфига, окружения).
5. Исправить.

**14. ImagePullBackOff**

Не может скачать образ. Причины: образ не существует, нет доступа к registry, опечатка, сеть. Решение: проверить имя/тег, добавить `imagePullSecrets`, проверить сеть.

**15. `kubectl port-forward`**

Пробрасывает порт Pod'а/Service'а/Deployment'а на локальную машину. Используется для доступа к внутренним сервисам без публикации.

---

## Куда идти дальше?

Мы разобрали архитектуру Kubernetes и основные объекты. Теперь ты знаешь:

- Из чего состоит кластер.
- Как работает reconciliation loop.
- Что такое Pod и workloads.
- Как работают Services и DNS.
- Как использовать namespaces.
- Как диагностировать проблемы.

Но мы пока не разобрали:

- **Как scheduler решает, куда поставить Pod.** Requests/limits, affinity, taints, priority.
- **Как работает сеть в K8s.** CNI, Network Policies, Ingress.
- **Как хранить данные.** PV, PVC, StatefulSets.
- **Как передавать конфигурацию.** ConfigMaps, Secrets.

Следующие главы — про это.

**Глава 9: Kubernetes — планирование и ресурсы.** Погнали. 🚀