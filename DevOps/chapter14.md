# ⚙️ Глава 14: Kubernetes — контроллеры и операторы

**Что вы узнаете:**
- Что такое reconciliation loop и как он работает на уровне кода.
- Что такое controller pattern и зачем он нужен.
- Что такое Custom Resource Definition (CRD) и зачем расширять API.
- Что такое operator pattern и чем он отличается от обычного контроллера.
- Как устроены informers и work queues.
- Что такое ownership и finalizers.
- Как написать собственный контроллер на Go через Kubebuilder.
- Что такое controller-runtime и как он упрощает разработку.
- Как работают готовые операторы: Prometheus Operator, Postgres Operator, Strimzi.
- Как отлаживать операторы.
- Когда писать свой оператор, а когда — использовать готовый.

**После прочтения вы сможете:**
- Объяснить, как Kubernetes обеспечивает desired state через контроллеры.
- Написать CRD и зарегистрировать его в кластере.
- Написать простой контроллер на Go через controller-runtime.
- Использовать Kubebuilder для генерации скелета оператора.
- Понимать, как работают Prometheus Operator, Strimzi, Postgres Operator.
- Отлаживать операторы через логи, events, status.
- Решить, нужен ли свой оператор для конкретной задачи.

---

## Содержание

- [14.0 Пролог: Kubernetes не знает про твой сервис](#140-пролог-kubernetes-не-знает-про-твой-сервис)
- [14.1 Reconciliation loop изнутри](#141-reconciliation-loop-изнутри)
- [14.2 Controller pattern](#142-controller-pattern)
- [14.3 Custom Resource Definition (CRD)](#143-custom-resource-definition-crd)
- [14.4 Operator pattern](#144-operator-pattern)
- [14.5 Informers и work queues](#145-informers-и-work-queues)
- [14.6 Ownership и finalizers](#146-ownership-и-finalizers)
- [14.7 controller-runtime](#147-controller-runtime)
- [14.8 Kubebuilder: пишем свой оператор](#148-kubebuilder-пишем-свой-оператор)
- [14.9 Готовые операторы](#149-готовые-операторы)
- [14.10 Отладка операторов](#1410-отладка-операторов)
- [14.11 Когда писать свой оператор](#1411-когда-писать-свой-оператор)
- [14.12 Диагностика проблем](#1412-диагностика-проблем)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 14.0 Пролог: Kubernetes не знает про твой сервис

Ты — DevOps-инженер. Ты разворачиваешь **PostgreSQL** в Kubernetes. Тебе нужно:

- **StatefulSet** с 3 репликами.
- **Service** для доступа.
- **PVC** для каждого Pod'а.
- **ConfigMap** с настройками.
- **Secret** с паролем.
- **Replication** между репликами (streaming replication).
- **Failover** при падении master'а.
- **Backup** раз в сутки.
- **Monitoring** через `postgres_exporter`.

**Вопрос:** как всё это связать?

**Вариант 1: Руками.**

Ты пишешь YAML-манифесты. Применяешь. Настраиваешь replication вручную. Пишешь скрипт для failover. CronJob для backup. ServiceMonitor для monitoring.

**Проблема:** если master упадёт — failover **не произойдёт** автоматически. Нужен человек.

**Вариант 2: Написать автоматизацию.**

Ты пишешь скрипт, который следит за состоянием кластера. Если master упал — повышает slave. Если нужен backup — запускает. Если что-то не так — исправляет.

**Проблема:** этот скрипт — **внешняя система**. Он не часть Kubernetes. Он не имеет доступа к внутреннему состоянию. Он работает через API.

**Вариант 3: Operator.**

Ты пишешь **operator** — программу, которая работает **внутри** Kubernetes. Она знает про PostgreSQL. Она следит за состоянием. Она автоматически делает failover, backup, upgrades.

**Что даёт:**

- **Автоматизация.** Failover без человека.
- **Declarative.** Ты говоришь «хочу PostgreSQL с 3 репликами», operator делает.
- **Custom API.** Ты создаёшь CRD `PostgreSQL`, а operator следит за ним.
- **Встроено в Kubernetes.** Operator — часть кластера.

**Это — следующий уровень Kubernetes.**

**Контроллеры и операторы** — как Kubernetes реализует **самовосстановление** и **автоматизацию** для **специфичных** приложений.

В этой главе мы разберём:

- **Как работает reconciliation loop** изнутри.
- **Что такое CRD** и как расширять API.
- **Что такое operator** и чем отличается от контроллера.
- **Как написать свой оператор** на Go.
- **Готовые операторы** (Prometheus, Strimzi, Postgres).

Это — **вершина Kubernetes**. Понимание контроллеров = понимание всей системы.

---

## 14.1 Reconciliation loop изнутри

### 🔌 Проблема: как Kubernetes обеспечивает desired state

В Главе 8 мы узнали: Kubernetes — **level-triggered**. Он приводит систему к desired state.

**Но как именно** это работает на уровне кода?

**Ответ:** контроллеры.

### 📊 Что такое reconciliation loop

**Reconciliation loop** — бесконечный цикл:

```go
for {
    desired := getDesiredState()      // из spec
    current := getCurrentState()      // из status
    if desired != current {
        reconcile(desired, current)   // сделать шаг
    }
    waitForEvent()                    // ждать изменений
}
```

**Ключевые принципы:**

1. **Level-triggered.** Смотрит на **состояние**, не на события.
2. **Idempotent.** Повторный запуск = тот же результат.
3. **Eventually consistent.** В конечном счёте сойдётся.
4. **Continuous.** Работает **постоянно**, не один раз.

### 🎯 Пример: ReplicaSet Controller

**Desired state:**

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
    # ...
```

**Что делает ReplicaSet Controller:**

```go
func (c *ReplicaSetController) reconcile(key string) error {
    // 1. Получить ReplicaSet из API
    rs := c.getReplicaSet(key)
    
    // 2. Получить текущие Pod'ы
    pods := c.getPodsForReplicaSet(rs)
    
    // 3. Сравнить
    desired := *rs.Spec.Replicas
    current := len(pods)
    
    // 4. Действовать
    if current < desired {
        // Создать недостающие Pod'ы
        for i := current; i < desired; i++ {
            c.createPod(rs)
        }
    } else if current > desired {
        // Удалить лишние Pod'ы
        for i := desired; i < current; i++ {
            c.deletePod(pods[i])
        }
    }
    
    return nil
}
```

**Что происходит:**

- **Желаемое:** 3 Pod'а.
- **Текущее:** 2 Pod'а.
- **Действие:** создать 1 Pod.

**И так — бесконечно.**

### 🎯 Level-triggered vs edge-triggered

**Edge-triggered:** реагирует на **события**.

```
Событие: Pod упал → создать новый
```

**Проблема:** если событие пропущено (например, apiserver был недоступен) — Pod не восстановится.

**Level-triggered:** смотрит на **состояние**.

```
Состояние: 2 Pod'а вместо 3 → создать новый
```

**Что даёт:** даже если событие пропущено — в следующий reconcile (или periodic resync) система сойдётся.

**Kubernetes использует level-triggered.**

### 🎯 Resync period

**Resync** — периодическое перечитывание состояния.

```go
// controller-runtime
mgr, _ := ctrl.NewManager(config, ctrl.Options{
    SyncPeriod: &resyncPeriod,  // по умолчанию 10 часов
})
```

**Что даёт:**

- **Гарантия.** Даже если событие потеряно — periodic resync обнаружит drift.
- **Производительность.** Меньше событий — меньше нагрузки на API.

**По умолчанию:** 10 часов.

### 🎯 Пример: полный цикл

**Сценарий:** Pod упал.

```
t=0:   Pod "myapp-abc123" переходит в Failed
t=1:   kubelet отправляет обновление в apiserver
t=2:   apiserver обновляет Pod в etcd
t=3:   apiserver отправляет событие подписчикам (watch)
t=4:   ReplicaSet Controller получает событие
t=5:   Controller добавляет Pod в work queue
t=6:   Controller обрабатывает: "current=2, desired=3"
t=7:   Controller создаёт новый Pod через apiserver
t=8:   apiserver создаёт Pod в etcd
t=9:   Scheduler назначает ноду
t=10:  kubelet запускает контейнер
t=11:  Pod Running
```

**Итого:** ~10 секунд от падения до восстановления.

### 🎯 Что происходит при ошибке

**Сценарий:** controller упал во время reconcile.

```
t=0:   Controller получает событие
t=1:   Controller добавляет Pod в queue
t=2:   Controller падает
t=3:   Controller перезапускается
t=4:   Controller читает очередь (или перечитывает состояние)
t=5:   Controller обрабатывает — видит, что Pod'ов 2
t=6:   Создаёт новый
```

**Что даёт:** controller может упасть в любой момент — система восстановится.

### 🎯 Level-triggered в коде

**Правильная реализация:**

```go
func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Прочитать объект
    obj := &MyResource{}
    if err := r.Get(ctx, req.NamespacedName, obj); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 2. Определить desired state
    desired := computeDesired(obj)
    
    // 3. Получить текущее состояние
    current := getCurrent(obj)
    
    // 4. Сравнить и привести
    if !reflect.DeepEqual(desired, current) {
        if err := applyChanges(ctx, desired); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // 5. Обновить status
    if err := r.Status().Update(ctx, obj); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{}, nil
}
```

**Ключевое:**

- **Не хранить состояние** в памяти контроллера.
- **Каждый reconcile** читает объект заново.
- **Идемпотентность** — повторный reconcile = тот же результат.

### 💡 Практика: что важно понять

**✅ ОБЯЗАТЕЛЬНО:**

1. **Reconciliation loop** — основа Kubernetes.
2. **Level-triggered**, не edge-triggered.
3. **Idempotent** — можно запускать много раз.
4. **Eventually consistent.**

**👍 СТОИТ:**

4. **Не хранить состояние** в контроллере.
5. **Resync period** для гарантии.
6. **Обрабатывать ошибки** (retry).

**❌ НЕ ДЕЛАЙ:**

7. **Не полагайся на порядок событий.**
8. **Не делай reconcile не-idempotent.**
9. **Не забывай про status.**

### Где мы сейчас

Мы разобрали reconciliation loop. Теперь — **controller pattern**.

---

## 14.2 Controller pattern

### 🔌 Проблема: как написать контроллер

Reconciliation loop — общая идея. Как реализовать конкретный контроллер?

**Ответ:** controller pattern.

### 📊 Что такое controller pattern

**Controller pattern** — архитектурный паттерн для реализации reconciliation.

**Компоненты:**

1. **Informer** — кэш объектов и watch на API.
2. **Work queue** — очередь объектов для обработки.
3. **Reconciler** — функция обработки.
4. **Event handlers** — что делать при событиях.
5. **Metrics** — для observability.

### 🎯 Архитектура

```
┌─────────────────────────────────────────┐
│                APISERVER                 │
└──────────────┬──────────────────────────┘
               │ watch
               ▼
┌─────────────────────────────────────────┐
│              INFORMER                    │
│                                          │
│  - Watch API                             │
│  - Кэш объектов                          │
│  - Event handlers                        │
└──────────────┬──────────────────────────┘
               │ events
               ▼
┌─────────────────────────────────────────┐
│             WORK QUEUE                   │
│                                          │
│  - Queue объектов                        │
│  - Rate limiting                         │
│  - Retries                               │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│             RECONCILER                   │
│                                          │
│  - Reconcile(obj)                        │
│  - Создать/обновить/удалить              │
│  - Обновить status                       │
└─────────────────────────────────────────┘
```

### 🎯 Informer

**Informer** — кэш объектов из API.

**Что делает:**

- **Watch** API на изменения.
- **Кэширует** объекты в памяти.
- **Отправляет** события обработчикам (Add, Update, Delete).

**Что даёт:**

- **Быстрое чтение** — из кэша, не из API.
- **Меньше нагрузки** на API.
- **Согласованность** — все контроллеры видят одно состояние.

**Пример:**

```go
informer := informers.NewSharedInformerFactory(clientset, 0)

podInformer := informer.Core().V1().Pods().Informer()

podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj interface{}) {
        pod := obj.(*v1.Pod)
        queue.Add(pod)
    },
    UpdateFunc: func(oldObj, newObj interface{}) {
        pod := newObj.(*v1.Pod)
        queue.Add(pod)
    },
    DeleteFunc: func(obj interface{}) {
        pod := obj.(*v1.Pod)
        queue.Add(pod)
    },
})

go informer.Start(stopCh)
```

### 🎯 Work queue

**Work queue** — очередь объектов для обработки.

**Что даёт:**

- **Rate limiting** — не спамить API.
- **Retries** — повторить при ошибке.
- **Дедупликация** — не обрабатывать один объект дважды.
- **Порядок** — по возможности.

**Пример:**

```go
queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

// Добавить в очередь
queue.Add(key)

// Обработать
for {
    key, shutdown := queue.Get()
    if shutdown {
        return
    }
    
    err := reconcile(key)
    if err != nil {
        queue.AddRateLimited(key)  // повторить с backoff
    } else {
        queue.Forget(key)  // успех
    }
    
    queue.Done(key)
}
```

### 🎯 Rate limiting

**Экспоненциальный backoff:**

```
Попытка 1: через 5 мс
Попытка 2: через 10 мс
Попытка 3: через 20 мс
Попытка 4: через 40 мс
...
Максимум: 1000 секунд (16 минут)
```

**Что даёт:** не молотить по API, если что-то сломано.

### 🎯 Retries

**При ошибке reconcile:**

1. Объект возвращается в очередь.
2. Обрабатывается снова с backoff.
3. После N попыток — сдаётся (или логирует).

**Правило:** reconcile должен быть **идемпотентен**. Retry безопасен.

### 🎯 Metrics

**Controller экспортирует метрики:**

- **workqueue_depth** — длина очереди.
- **workqueue_adds_total** — всего добавлено.
- **workqueue_retries_total** — всего retries.
- **reconcile_errors_total** — ошибки.
- **reconcile_duration_seconds** — длительность.

**Что даёт:** мониторинг здоровья контроллера.

### 🎯 Полный пример

```go
package main

import (
    "context"
    "fmt"
    "time"
    
    v1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/fields"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/util/workqueue"
)

type Controller struct {
    clientset kubernetes.Interface
    queue     workqueue.RateLimitingInterface
    informer  cache.SharedIndexInformer
}

func NewController(clientset kubernetes.Interface) *Controller {
    queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())
    
    // ListWatch
    listWatcher := cache.NewListWatchFromClient(
        clientset.CoreV1().RESTClient(),
        "pods",
        metav1.NamespaceAll,
        fields.Everything(),
    )
    
    informer := cache.NewSharedIndexInformer(
        listWatcher,
        &v1.Pod{},
        0,
        cache.Indexers{},
    )
    
    c := &Controller{
        clientset: clientset,
        queue:     queue,
        informer:  informer,
    }
    
    informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc:    c.addPod,
        UpdateFunc: c.updatePod,
        DeleteFunc: c.deletePod,
    })
    
    return c
}

func (c *Controller) addPod(obj interface{}) {
    pod := obj.(*v1.Pod)
    key, _ := cache.MetaNamespaceKeyFunc(pod)
    c.queue.Add(key)
}

func (c *Controller) updatePod(oldObj, newObj interface{}) {
    c.addPod(newObj)
}

func (c *Controller) deletePod(obj interface{}) {
    c.addPod(obj)
}

func (c *Controller) Run(workers int, stopCh <-chan struct{}) {
    defer c.queue.ShutDown()
    
    go c.informer.Run(stopCh)
    
    if !cache.WaitForCacheSync(stopCh, c.informer.HasSynced) {
        return
    }
    
    for i := 0; i < workers; i++ {
        go c.worker()
    }
    
    <-stopCh
}

func (c *Controller) worker() {
    for c.processNextItem() {
    }
}

func (c *Controller) processNextItem() bool {
    key, shutdown := c.queue.Get()
    if shutdown {
        return false
    }
    defer c.queue.Done(key)
    
    err := c.reconcile(key.(string))
    if err != nil {
        c.queue.AddRateLimited(key)
        return true
    }
    
    c.queue.Forget(key)
    return true
}

func (c *Controller) reconcile(key string) error {
    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    if err != nil {
        return err
    }
    
    pod, err := c.informer.GetIndexer().ByIndex("namespace", namespace)
    if err != nil {
        return err
    }
    
    fmt.Printf("Reconciling Pod %s/%s\n", namespace, name)
    _ = pod
    return nil
}

func main() {
    // ...
}
```

### 💡 Практика: как правильно писать контроллер

**✅ ОБЯЗАТЕЛЬНО:**

1. **Informer** для чтения.
2. **Work queue** для обработки.
3. **Rate limiting** для retries.
4. **Idempotent reconcile.**

**👍 СТОИТ:**

4. **Метрики** для observability.
5. **Несколько workers** для параллелизма.
6. **Логирование** для отладки.

**❌ НЕ ДЕЛАЙ:**

7. **Не читай из API** напрямую — используй кэш.
8. **Не делай reconcile не-idempotent.**
9. **Не игнорируй ошибки.**

### Где мы сейчас

Мы разобрали controller pattern. Теперь — **CRD** — как расширять API.

---

## 14.3 Custom Resource Definition (CRD)

### 🔌 Проблема: Kubernetes не знает про твой ресурс

Kubernetes знает про Pod, Service, Deployment. Но что если тебе нужен свой ресурс?

**Пример:** ты хочешь ресурс `PostgreSQL` с 3 репликами, backup, failover.

**Решение:** CRD.

### 📊 Что такое CRD

**Custom Resource Definition (CRD)** — расширение API Kubernetes.

**Что даёт:**

- **Свой ресурс** (kind: PostgreSQL).
- **Свой API** (`/apis/database.example.com/v1/postgresqls`).
- **Свой контроллер** (который следит за ресурсом).
- **Встроено** в Kubernetes.

### 🎯 Пример CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresqls.database.example.com
spec:
  group: database.example.com
  names:
    kind: PostgreSQL
    plural: postgresqls
    singular: postgresql
    shortNames:
      - pg
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [replicas]
              properties:
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 5
                version:
                  type: string
                  default: "16"
                storage:
                  type: string
                  default: "10Gi"
                backup:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: true
                    schedule:
                      type: string
                      default: "0 2 * * *"
            status:
              type: object
              properties:
                ready:
                  type: boolean
                master:
                  type: string
                replicas:
                  type: integer
      subresources:
        status: {}
```

**Что даёт:**

- **kind: PostgreSQL** — новый ресурс.
- **spec.replicas** — количество реплик.
- **spec.version** — версия PostgreSQL.
- **spec.backup** — настройки backup.
- **status** — состояние.

**После применения:**

```bash
kubectl get crd
# NAME                          CREATED AT
# postgresqls.database.example.com   2026-01-15T10:00:00Z

kubectl get postgresqls
# No resources found.

kubectl api-resources | grep postgresql
# postgresqls    pg    database.example.com/v1    true    PostgreSQL
```

### 🎯 Использование CRD

```yaml
apiVersion: database.example.com/v1
kind: PostgreSQL
metadata:
  name: my-postgres
  namespace: production
spec:
  replicas: 3
  version: "16"
  storage: 100Gi
  backup:
    enabled: true
    schedule: "0 2 * * *"
```

**Что произойдёт:**

- **Ничего!** Без контроллера.

**Проблема:** CRD — просто **схема**. Kubernetes знает, что такой ресурс **существует**, но **не знает**, что с ним делать.

**Решение:** контроллер (или оператор).

### 🎯 OpenAPI Schema

**Schema** определяет:

- **Типы** полей.
- **Обязательные** поля.
- **Значения по умолчанию.**
- **Валидацию** (minimum, maximum, pattern).
- **Enum** (допустимые значения).

**Пример валидации:**

```yaml
schema:
  openAPIV3Schema:
    type: object
    properties:
      spec:
        type: object
        required: [replicas]
        properties:
          replicas:
            type: integer
            minimum: 1
            maximum: 5
          version:
            type: string
            enum: ["15", "16", "17"]
          storage:
            type: string
            pattern: "^[0-9]+(Gi|Mi)$"
```

**Что даёт:** Kubernetes **валидирует** ресурс при создании.

**Попытка создать с `replicas: 10`:**

```bash
kubectl apply -f postgres.yaml
# Error: PostgreSQL.database.example.com "my-postgres" is invalid:
# spec.replicas: Invalid value: 10: spec.replicas in body should be less than or equal to 5
```

### 🎯 Subresources

**Subresources** — дополнительные API для CRD.

**`status`:**

```yaml
subresources:
  status: {}
```

**Что даёт:**

- **Отдельный endpoint** для status.
- **Контроллер обновляет status** через `Status().Update()`.
- **Пользователь не может** менять status.

**`scale`:**

```yaml
subresources:
  scale:
    specReplicasPath: .spec.replicas
    statusReplicasPath: .status.replicas
```

**Что даёт:**

- **`kubectl scale`** работает.
- **HPA** может масштабировать CRD.

### 🎯 Printer Columns

**Дополнительные колонки в `kubectl get`:**

```yaml
additionalPrinterColumns:
  - name: Replicas
    type: integer
    jsonPath: .spec.replicas
  - name: Ready
    type: string
    jsonPath: .status.ready
  - name: Master
    type: string
    jsonPath: .status.master
  - name: Age
    type: date
    jsonPath: .metadata.creationTimestamp
```

**Что даёт:**

```bash
kubectl get postgresqls
# NAME          REPLICAS   READY   MASTER      AGE
# my-postgres   3          true    pg-0        5m
```

### 🎯 Версионирование CRD

**Несколько версий:**

```yaml
spec:
  versions:
    - name: v1alpha1
      served: true
      storage: false
    - name: v1
      served: true
      storage: true
```

**Что даёт:**

- **v1alpha1** — для тестирования.
- **v1** — стабильная.
- **Storage version** — в какой версии хранится в etcd.

**Conversion webhooks** для конвертации между версиями.

### 🎯 Namespaced vs Cluster-scoped

**Namespaced:**

```yaml
spec:
  scope: Namespaced
```

**Cluster-scoped:**

```yaml
spec:
  scope: Cluster
```

**Что выбрать:**

- **Namespaced** — если ресурс принадлежит namespace (PostgreSQL, приложения).
- **Cluster** — если ресурс глобальный (ClusterPolicy, ClusterIssuer).

### 🔬 Практика: CRD

```bash
# 1. Создать CRD
cat > crd.yaml <<EOF
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresqls.database.example.com
spec:
  group: database.example.com
  names:
    kind: PostgreSQL
    plural: postgresqls
    singular: postgresql
    shortNames: [pg]
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      additionalPrinterColumns:
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
        - name: Ready
          type: boolean
          jsonPath: .status.ready
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [replicas]
              properties:
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 5
                version:
                  type: string
                  default: "16"
                storage:
                  type: string
                  default: "10Gi"
            status:
              type: object
              properties:
                ready:
                  type: boolean
                master:
                  type: string
      subresources:
        status: {}
EOF
kubectl apply -f crd.yaml

# 2. Проверить
kubectl get crd postgresqls.database.example.com
kubectl api-resources | grep postgresql

# 3. Создать экземпляр
cat > instance.yaml <<EOF
apiVersion: database.example.com/v1
kind: PostgreSQL
metadata:
  name: my-postgres
  namespace: default
spec:
  replicas: 3
  version: "16"
  storage: 100Gi
EOF
kubectl apply -f instance.yaml

# 4. Проверить
kubectl get postgresqls
# NAME          REPLICAS   READY
# my-postgres   3          <none>

kubectl get pg
kubectl describe postgresql my-postgres

# 5. Попробовать невалидное значение
cat > invalid.yaml <<EOF
apiVersion: database.example.com/v1
kind: PostgreSQL
metadata:
  name: invalid
spec:
  replicas: 10
EOF
kubectl apply -f invalid.yaml
# Error: spec.replicas: Invalid value: 10: should be less than or equal to 5
```

### 💡 Практика: как правильно создавать CRD

**✅ ОБЯЗАТЕЛЬНО:**

1. **OpenAPI Schema** с валидацией.
2. **Status subresource.**
3. **Printer columns** для удобства.
4. **Версионирование.**

**👍 СТОИТ:**

4. **`scale` subresource** для HPA.
5. **Short names** для удобства.
6. **Документация** для CRD.

**❌ НЕ ДЕЛАЙ:**

7. **Не создавай CRD без контроллера.**
8. **Не забывай про валидацию.**
9. **Не используй `x-kubernetes-preserve-unknown-fields`** без понимания.

### Где мы сейчас

Мы разобрали CRD. Теперь — **operator pattern**.

---

## 14.4 Operator pattern

### 🔌 Проблема: контроллер для своего ресурса

CRD расширяет API. Но нужен **контроллер**, который обрабатывает CRD.

**Обычный контроллер** — для встроенных ресурсов (Pod, Deployment).

**Оператор** — контроллер для **кастомных** ресурсов + **доменные знания**.

### 📊 Что такое operator

**Operator** — контроллер для CRD, который:

1. **Следит** за CRD.
2. **Знает** предметную область (PostgreSQL, Kafka, Prometheus).
3. **Управляет** сложными приложениями.

**Ключевое отличие от контроллера:** operator содержит **доменные знания**.

**Пример:**

- **Контроллер** ReplicaSet знает только про Pod'ы.
- **Оператор** PostgreSQL знает про replication, failover, backup.

### 🎯 Что делает оператор

**1. Provisioning.**

Создание StatefulSet, Service, ConfigMap, Secret, PVC.

**2. Configuration.**

Настройка replication, параметров PostgreSQL.

**3. Backup.**

CronJob для `pg_dump`, загрузка в S3.

**4. Failover.**

Обнаружение падения master'а, promotion slave.

**5. Upgrade.**

Обновление версии PostgreSQL.

**6. Scaling.**

Увеличение/уменьшение реплик.

**7. Monitoring.**

Создание ServiceMonitor, PrometheusRule.

**8. Restore.**

Восстановление из backup.

### 🎯 Capability Levels

**Operator Capability Model** от Red Hat:

**Level 1: Basic Install.**

- Создание StatefulSet, Service.
- Базовое развёртывание.

**Level 2: Seamless Upgrades.**

- Обновление версии.
- Миграции.

**Level 3: Full Lifecycle.**

- Backup/restore.
- Failover.
- Scaling.

**Level 4: Deep Insights.**

- Metrics.
- Alerts.
- Logs.

**Level 5: Auto Pilot.**

- Автоматическое scaling.
- Auto-tuning.
- Self-healing.

**Цель:** Level 5 — «автопилот».

### 🎯 Пример: PostgreSQL Operator

**CRD:**

```yaml
apiVersion: database.example.com/v1
kind: PostgreSQL
metadata:
  name: my-postgres
spec:
  replicas: 3
  version: "16"
  storage: 100Gi
  backup:
    enabled: true
    schedule: "0 2 * * *"
```

**Что делает оператор:**

1. **Создаёт StatefulSet** с 3 репликами.
2. **Создаёт Service** (headless).
3. **Создаёт ConfigMap** с настройками.
4. **Создаёт Secret** с паролем.
5. **Настраивает streaming replication** между Pod'ами.
6. **Создаёт CronJob** для backup.
7. **Создаёт ServiceMonitor** для monitoring.
8. **Следит за состоянием.**
9. **При падении master'а** — promotion slave.
10. **Обновляет status** CRD.

**Разработчик:** создаёт CRD, оператор делает остальное.

### 🎯 Как работает оператор

```
┌─────────────────────────────────────────┐
│           DEVELOPER                      │
│                                          │
│  kubectl apply -f postgres.yaml         │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│           APISERVER                      │
│                                          │
│  Создаёт PostgreSQL CRD instance        │
└──────────────┬──────────────────────────┘
               │ watch
               ▼
┌─────────────────────────────────────────┐
│           OPERATOR                       │
│                                          │
│  1. Reconcile(PostgreSQL)                │
│  2. Создать StatefulSet                  │
│  3. Создать Service                      │
│  4. Настроить replication                │
│  5. Обновить status                      │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│           KUBERNETES                     │
│                                          │
│  StatefulSet, Service, ConfigMap, ...   │
└─────────────────────────────────────────┘
```

### 🎯 Пример оператора на Go

**Reconciler:**

```go
package controllers

import (
    "context"
    
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/reconcile"
    
    databasev1 "github.com/example/postgres-operator/api/v1"
)

type PostgreSQLReconciler struct {
    client.Client
}

func (r *PostgreSQLReconciler) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
    // 1. Получить PostgreSQL instance
    pg := &databasev1.PostgreSQL{}
    if err := r.Get(ctx, req.NamespacedName, pg); err != nil {
        if errors.IsNotFound(err) {
            return reconcile.Result{}, nil
        }
        return reconcile.Result{}, err
    }
    
    // 2. Проверить/создать StatefulSet
    sts := &appsv1.StatefulSet{}
    err := r.Get(ctx, types.NamespacedName{
        Name:      pg.Name + "-statefulset",
        Namespace: pg.Namespace,
    }, sts)
    
    if errors.IsNotFound(err) {
        // Создать StatefulSet
        sts = r.newStatefulSet(pg)
        if err := r.Create(ctx, sts); err != nil {
            return reconcile.Result{}, err
        }
    } else if err != nil {
        return reconcile.Result{}, err
    }
    
    // 3. Проверить/создать Service
    svc := &corev1.Service{}
    err = r.Get(ctx, types.NamespacedName{
        Name:      pg.Name,
        Namespace: pg.Namespace,
    }, svc)
    
    if errors.IsNotFound(err) {
        svc = r.newService(pg)
        if err := r.Create(ctx, svc); err != nil {
            return reconcile.Result{}, err
        }
    } else if err != nil {
        return reconcile.Result{}, err
    }
    
    // 4. Обновить status
    pg.Status.Ready = sts.Status.ReadyReplicas == *pg.Spec.Replicas
    pg.Status.Replicas = sts.Status.ReadyReplicas
    if err := r.Status().Update(ctx, pg); err != nil {
        return reconcile.Result{}, err
    }
    
    return reconcile.Result{}, nil
}

func (r *PostgreSQLReconciler) newStatefulSet(pg *databasev1.PostgreSQL) *appsv1.StatefulSet {
    // Создать StatefulSet с параметрами из pg.Spec
    // ...
    return &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      pg.Name + "-statefulset",
            Namespace: pg.Namespace,
        },
        Spec: appsv1.StatefulSetSpec{
            Replicas: pg.Spec.Replicas,
            // ...
        },
    }
}

func (r *PostgreSQLReconciler) newService(pg *databasev1.PostgreSQL) *corev1.Service {
    // ...
    return &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      pg.Name,
            Namespace: pg.Namespace,
        },
        Spec: corev1.ServiceSpec{
            // ...
        },
    }
}
```

**Что делает:**

1. Читает PostgreSQL instance.
2. Проверяет/создаёт StatefulSet.
3. Проверяет/создаёт Service.
4. Обновляет status.

**Всё идемпотентно.**

### 💡 Практика: что важно понять

**✅ ОБЯЗАТЕЛЬНО:**

1. **Operator** = CRD + контроллер + доменные знания.
2. **Level 5** — цель.
3. **Идемпотентность** обязательна.

**👍 СТОИТ:**

4. **Готовые операторы** перед написанием своего.
5. **controller-runtime** для разработки.
6. **Тестирование** операторов.

**❌ НЕ ДЕЛАЙ:**

7. **Не пиши оператор** для простых задач.
8. **Не забывай про идемпотентность.**
9. **Не игнорируй status.**

### Где мы сейчас

Мы разобрали operator pattern. Теперь — **informers и work queues** подробнее.

---

## 14.5 Informers и work queues

### 🔌 Проблема: как эффективно читать API

Контроллеру нужно читать объекты из API. Если читать **напрямую** — много запросов, медленно, дорого.

**Решение:** informers.

### 📊 Что такое informer

**Informer** — локальный кэш объектов из API.

**Что делает:**

- **Watch** API на изменения.
- **Кэширует** объекты в памяти.
- **Отправляет** события обработчикам.

**Что даёт:**

- **Быстрое чтение** — из кэша.
- **Меньше нагрузки** на API.
- **Согласованность.**

### 🎯 Shared informer

**Shared informer** — informer, разделяемый между несколькими контроллерами.

**Что даёт:**

- **Один watch** для всех.
- **Один кэш** для всех.
- **Меньше памяти.**

**Пример:**

```go
factory := informers.NewSharedInformerFactory(clientset, 0)

podInformer := factory.Core().V1().Pods().Informer()
podInformer.AddEventHandler(...)

// Все informers запускаются вместе
factory.Start(stopCh)
factory.WaitForCacheSync(stopCh)
```

### 🎯 Event handlers

**Что можно обработать:**

- **AddFunc** — добавление объекта.
- **UpdateFunc** — обновление.
- **DeleteFunc** — удаление.

**Пример:**

```go
podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj interface{}) {
        pod := obj.(*v1.Pod)
        log.Printf("Pod added: %s/%s", pod.Namespace, pod.Name)
        queue.Add(pod)
    },
    UpdateFunc: func(oldObj, newObj interface{}) {
        oldPod := oldObj.(*v1.Pod)
        newPod := newObj.(*v1.Pod)
        if oldPod.ResourceVersion == newPod.ResourceVersion {
            return  // одинаковые
        }
        queue.Add(newPod)
    },
    DeleteFunc: func(obj interface{}) {
        pod, ok := obj.(*v1.Pod)
        if !ok {
            // DeletedFinalStateUnknown
            tombstone := obj.(cache.DeletedFinalStateUnknown)
            pod = tombstone.Obj.(*v1.Pod)
        }
        queue.Add(pod)
    },
})
```

**Ключевое:** update — **сравнивать** resourceVersion, чтобы не обрабатывать одно и то же.

### 🎯 Work queue

**Work queue** — очередь объектов для обработки.

**Что даёт:**

- **Rate limiting.**
- **Retries.**
- **Дедупликация.**
- **Порядок.**

**Пример:**

```go
queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

// Добавить
queue.Add(key)

// Обработать
for {
    key, shutdown := queue.Get()
    if shutdown {
        return
    }
    
    err := reconcile(key)
    if err != nil {
        queue.AddRateLimited(key)  // retry
    } else {
        queue.Forget(key)  // успех
    }
    
    queue.Done(key)
}
```

### 🎯 Типы очередей

**1. RateLimitingQueue.**

Rate limiting + retries.

**2. DelayingQueue.**

Отложенная обработка.

**3. TypedRateLimitingQueue.**

Типизированная.

**4. PriorityQueue.**

С приоритетами.

### 🎯 Rate limiter

**DefaultControllerRateLimiter:**

- **Exponential backoff** для retries.
- **Bucket rate limiter** для новых объектов.

**Настройка:**

```go
queue := workqueue.NewRateLimitingQueue(&workqueue.BucketRateLimiter{
    Limiter: rate.NewLimiter(rate.Limit(10), 100),
})
```

**Что даёт:** не спамить API.

### 🎯 Метрики

**Controller-runtime экспортирует:**

```
workqueue_depth{name="mycontroller"}                    5
workqueue_adds_total{name="mycontroller"}               1000
workqueue_queue_duration_seconds_bucket{name="mycontroller",le="1e-08"}  100
workqueue_work_duration_seconds_bucket{name="mycontroller",le="1e-08"}   100
workqueue_retries_total{name="mycontroller"}            42
workqueue_longest_running_processor_seconds{name="mycontroller"}  0.5
workqueue_unfinished_work_seconds{name="mycontroller"}  0
```

**Что даёт:** мониторинг здоровья контроллера.

### 🎯 Работа с informer

**Получить объект:**

```go
obj, exists, err := informer.GetStore().GetByKey(key)
```

**Получить все объекты:**

```go
for _, obj := range informer.GetStore().List() {
    // ...
}
```

**Получить по label:**

```go
objs, err := informer.GetIndexer().ByIndex(
    cache.NamespaceIndex,  // индекс
    "default",             // значение
)
```

**Что даёт:** быстрое чтение без API.

### 🎯 Синхронизация кэша

**При старте контроллера** — нужно дождаться синхронизации кэша:

```go
if !cache.WaitForCacheSync(stopCh, informer.HasSynced) {
    return
}
```

**Что даёт:** контроллер не начнёт работу, пока кэш не готов.

### 🔬 Практика: informer

```go
package main

import (
    "fmt"
    "time"
    
    v1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/informers"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/tools/clientcmd"
)

func main() {
    config, _ := clientcmd.BuildConfigFromFlags("", "~/.kube/config")
    clientset, _ := kubernetes.NewForConfig(config)
    
    factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)
    
    podInformer := factory.Core().V1().Pods().Informer()
    
    podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            pod := obj.(*v1.Pod)
            fmt.Printf("ADD: %s/%s (%s)\n", pod.Namespace, pod.Name, pod.Status.Phase)
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            pod := newObj.(*v1.Pod)
            fmt.Printf("UPDATE: %s/%s (%s)\n", pod.Namespace, pod.Name, pod.Status.Phase)
        },
        DeleteFunc: func(obj interface{}) {
            pod := obj.(*v1.Pod)
            fmt.Printf("DELETE: %s/%s\n", pod.Namespace, pod.Name)
        },
    })
    
    stopCh := make(chan struct{})
    defer close(stopCh)
    
    factory.Start(stopCh)
    factory.WaitForCacheSync(stopCh)
    
    fmt.Println("Informer synced, watching...")
    select {}
}
```

### 💡 Практика: как правильно использовать informers

**✅ ОБЯЗАТЕЛЬНО:**

1. **Shared informer** для нескольких контроллеров.
2. **WaitForCacheSync** перед работой.
3. **Сравнивать resourceVersion** в update.

**👍 СТОИТ:**

4. **Метрики** для мониторинга.
5. **Rate limiter** настроить.
6. **Несколько workers.**

**❌ НЕ ДЕЛАЙ:**

7. **Не читай API** напрямую.
8. **Не игнорируй ошибки.**
9. **Не забывай про tombstone.**

### Где мы сейчас

Мы разобрали informers. Теперь — **ownership и finalizers**.

---

## 14.6 Ownership и finalizers

### 🔌 Проблема: как связать ресурсы

Оператор создаёт StatefulSet, Service, ConfigMap. Кто отвечает за них?

- **Кто удалит** StatefulSet, если PostgreSQL удалён?
- **Что делать**, если нужно выполнить что-то **перед** удалением?

**Решение:** ownership и finalizers.

### 📊 Owner References

**Owner Reference** — ссылка на родительский объект.

**Пример:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-postgres-statefulset
  ownerReferences:
    - apiVersion: database.example.com/v1
      kind: PostgreSQL
      name: my-postgres
      uid: abc-123
      controller: true
      blockOwnerDeletion: true
```

**Что даёт:**

- **Каскадное удаление.** При удалении PostgreSQL → StatefulSet тоже удаляется.
- **Garbage collector** Kubernetes следит за этим.

### 🎯 Как установить ownership

**Через controller-runtime:**

```go
import "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"

// Создать StatefulSet с owner reference
sts := &appsv1.StatefulSet{
    ObjectMeta: metav1.ObjectMeta{
        Name:      pg.Name + "-statefulset",
        Namespace: pg.Namespace,
    },
    // ...
}

if err := controllerutil.SetControllerReference(pg, sts, r.Scheme); err != nil {
    return err
}

if err := r.Create(ctx, sts); err != nil {
    return err
}
```

**Что произойдёт:**

- `ownerReferences` будет установлена автоматически.
- При удалении `pg` → `sts` тоже удалится.

### 🎯 Finalizers

**Finalizer** — строка в `metadata.finalizers`, которая **блокирует** удаление объекта.

**Как работает:**

1. Пользователь удаляет объект.
2. Kubernetes ставит `deletionTimestamp`, но **не удаляет**.
3. Контроллер видит `deletionTimestamp` и выполняет **cleanup**.
4. Контроллер удаляет finalizer из `metadata.finalizers`.
5. Kubernetes удаляет объект.

**Пример:**

```yaml
apiVersion: database.example.com/v1
kind: PostgreSQL
metadata:
  name: my-postgres
  finalizers:
    - database.example.com/cleanup
```

**Что даёт:** можно выполнить cleanup **перед** удалением.

### 🎯 Пример finalizer

```go
const myFinalizer = "database.example.com/cleanup"

func (r *PostgreSQLReconciler) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
    pg := &databasev1.PostgreSQL{}
    if err := r.Get(ctx, req.NamespacedName, pg); err != nil {
        return reconcile.Result{}, client.IgnoreNotFound(err)
    }
    
    // Проверяем, удаляется ли объект
    if pg.DeletionTimestamp != nil {
        if controllerutil.ContainsFinalizer(pg, myFinalizer) {
            // Выполнить cleanup
            if err := r.cleanup(ctx, pg); err != nil {
                return reconcile.Result{}, err
            }
            
            // Удалить finalizer
            controllerutil.RemoveFinalizer(pg, myFinalizer)
            if err := r.Update(ctx, pg); err != nil {
                return reconcile.Result{}, err
            }
        }
        return reconcile.Result{}, nil
    }
    
    // Добавить finalizer, если ещё нет
    if !controllerutil.ContainsFinalizer(pg, myFinalizer) {
        controllerutil.AddFinalizer(pg, myFinalizer)
        if err := r.Update(ctx, pg); err != nil {
            return reconcile.Result{}, err
        }
    }
    
    // Обычный reconcile
    // ...
    
    return reconcile.Result{}, nil
}

func (r *PostgreSQLReconciler) cleanup(ctx context.Context, pg *databasev1.PostgreSQL) error {
    // 1. Сделать backup
    if err := r.backup(ctx, pg); err != nil {
        return err
    }
    
    // 2. Удалить внешние ресурсы
    if err := r.deleteExternalResources(ctx, pg); err != nil {
        return err
    }
    
    return nil
}
```

**Что даёт:**

- **Backup перед удалением.**
- **Удаление внешних ресурсов** (например, DNS-записи).
- **Graceful cleanup.**

### 🎯 Когда использовать finalizers

**✅ Использовать:**

- **Backup** перед удалением.
- **Cleanup внешних ресурсов** (cloud, DNS).
- **Удаление данных** в правильном порядке.
- **Уведомления.**

**❌ Не использовать:**

- **Для блокировки** удаления (это не то).
- **Без cleanup** (бессмысленно).
- **Если cleanup не нужен.**

### 🎯 Опасности finalizers

**1. Забытый finalizer.**

Если контроллер **не удаляет** finalizer — объект **никогда** не удалится.

**Симптом:**

```bash
kubectl delete postgresql my-postgres
kubectl get postgresql my-postgres
# NAME          AGE
# my-postgres   5m       ← не удаляется!

kubectl get postgresql my-postgres -o yaml | grep finalizers
# finalizers:
#   - database.example.com/cleanup
```

**Решение:** вручную удалить finalizer:

```bash
kubectl patch postgresql my-postgres -p '{"metadata":{"finalizers":null}}' --type=merge
```

**Правило:** finalizer должен быть удалён **всегда** — даже при ошибке cleanup.

**2. Долгий cleanup.**

Если cleanup занимает минуты — объект висит.

**Решение:** timeout в cleanup.

**3. Циклические finalizers.**

Объект A зависит от B, B зависит от A. Оба не удаляются.

**Решение:** избегать циклов.

### 🎯 Owner References vs Finalizers

| Аспект | Owner References | Finalizers |
|:---|:---|:---|
| **Что** | Ссылка на родителя | Блокировка удаления |
| **Когда** | При создании | При удалении |
| **Кто удаляет** | Kubernetes GC | Контроллер |
| **Для чего** | Каскадное удаление | Cleanup |

**Обычно используют вместе:**

- Owner references — для каскадного удаления дочерних ресурсов.
- Finalizers — для cleanup перед удалением родителя.

### 🔬 Практика: ownership и finalizers

```go
// В Reconcile:

// 1. Установить owner reference
if err := controllerutil.SetControllerReference(pg, sts, r.Scheme); err != nil {
    return err
}

// 2. Добавить finalizer
if !controllerutil.ContainsFinalizer(pg, myFinalizer) {
    controllerutil.AddFinalizer(pg, myFinalizer)
    if err := r.Update(ctx, pg); err != nil {
        return err
    }
}

// 3. Проверить удаление
if pg.DeletionTimestamp != nil {
    if controllerutil.ContainsFinalizer(pg, myFinalizer) {
        if err := r.cleanup(ctx, pg); err != nil {
            return err
        }
        controllerutil.RemoveFinalizer(pg, myFinalizer)
        if err := r.Update(ctx, pg); err != nil {
            return err
        }
    }
    return reconcile.Result{}, nil
}
```

### 💡 Практика: как правильно использовать ownership и finalizers

**✅ ОБЯЗАТЕЛЬНО:**

1. **Owner references** для дочерних ресурсов.
2. **Finalizers** для cleanup.
3. **Всегда удалять finalizer.**

**👍 СТОИТ:**

4. **Timeout в cleanup.**
5. **Логирование.**
6. **Тестирование удаления.**

**❌ НЕ ДЕЛАЙ:**

7. **Не забывай удалять finalizer.**
8. **Не используй для блокировки.**
9. **Не делай циклические зависимости.**

### Где мы сейчас

Мы разобрали ownership и finalizers. Теперь — **controller-runtime**.

---

## 14.7 controller-runtime

### 🔌 Проблема: писать контроллер с нуля долго

Нужно писать informer, work queue, metrics, лидерство. Много boilerplate.

**Решение:** controller-runtime.

### 📊 Что такое controller-runtime

**controller-runtime** — библиотека для написания контроллеров.

**Что даёт:**

- **Manager** — управляет контроллерами.
- **Builder** — DSL для создания контроллера.
- **Client** — удобный клиент API.
- **Reconciler** — интерфейс для реализации.
- **Metrics, leader election, webhooks** — из коробки.

**Используется в Kubebuilder и Operator SDK.**

### 🎯 Manager

**Manager** — управляет всем.

```go
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    Scheme:                 scheme,
    MetricsBindAddress:     ":8080",
    HealthProbeBindAddress: ":8081",
    LeaderElection:         true,
    LeaderElectionID:       "myoperator-leader",
})
```

**Что даёт:**

- **Кэш** informers.
- **Клиент** API.
- **Metrics** endpoint.
- **Leader election.**
- **Webhooks.**

### 🎯 Reconciler

**Reconciler** — интерфейс с одним методом.

```go
type Reconciler interface {
    Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error)
}
```

**Реализация:**

```go
type MyReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // ...
    return ctrl.Result{}, nil
}
```

### 🎯 Builder

**Builder** — DSL для создания контроллера.

```go
err := ctrl.NewControllerManagedBy(mgr).
    For(&databasev1.PostgreSQL{}).
    Owns(&appsv1.StatefulSet{}).
    Owns(&corev1.Service{}).
    Complete(r)
```

**Что даёт:**

- **`For`** — за каким ресурсом следить.
- **`Owns`** — за какими дочерними ресурсами следить.
- **`Watches`** — за какими дополнительными ресурсами следить.
- **`WithEventFilter`** — фильтр событий.

### 🎯 Client

**Client** — удобный клиент API.

```go
// Get
err := r.Get(ctx, types.NamespacedName{Name: "my-pod", Namespace: "default"}, pod)

// List
err := r.List(ctx, &podList, client.InNamespace("default"))

// Create
err := r.Create(ctx, pod)

// Update
err := r.Update(ctx, pod)

// Delete
err := r.Delete(ctx, pod)

// Patch
err := r.Patch(ctx, pod, client.Merge)
```

**Что даёт:**

- **Чтение из кэша.**
- **Запись в API.**
- **Типизация.**

### 🎯 ctrl.Result

**Result** — что делать после reconcile.

```go
// Успех, ничего не делать
return ctrl.Result{}, nil

// Повторить через 5 минут
return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil

// Повторить сразу
return ctrl.Result{Requeue: true}, nil

// Ошибка — retry с backoff
return ctrl.Result{}, err
```

**Когда что:**

- **`{}`** — всё сделано.
- **`RequeueAfter`** — периодическая проверка (например, backup).
- **`Requeue`** — нужно перезапустить reconcile.
- **`err`** — ошибка, retry.

### 🎯 Полный пример

```go
package main

import (
    "context"
    "os"
    
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/runtime"
    utilruntime "k8s.io/apimachinery/pkg/util/runtime"
    clientgoscheme "k8s.io/client-go/kubernetes/scheme"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
    
    databasev1 "github.com/example/postgres-operator/api/v1"
    "github.com/example/postgres-operator/controllers"
)

var (
    scheme   = runtime.NewScheme()
    setupLog = ctrl.Log.WithName("setup")
)

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))
    utilruntime.Must(databasev1.AddToScheme(scheme))
}

func main() {
    ctrl.SetLogger(zap.New(zap.UseDevMode(true)))
    
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme: scheme,
        MetricsBindAddress: ":8080",
    })
    if err != nil {
        setupLog.Error(err, "unable to start manager")
        os.Exit(1)
    }
    
    if err = (&controllers.PostgreSQLReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create controller")
        os.Exit(1)
    }
    
    setupLog.Info("starting manager")
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        setupLog.Error(err, "problem running manager")
        os.Exit(1)
    }
}
```

**Reconciler:**

```go
package controllers

import (
    "context"
    "time"
    
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/types"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/log"
    
    databasev1 "github.com/example/postgres-operator/api/v1"
)

const myFinalizer = "database.example.com/cleanup"

type PostgreSQLReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *PostgreSQLReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)
    
    // 1. Получить PostgreSQL
    pg := &databasev1.PostgreSQL{}
    if err := r.Get(ctx, req.NamespacedName, pg); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 2. Добавить finalizer
    if !controllerutil.ContainsFinalizer(pg, myFinalizer) {
        controllerutil.AddFinalizer(pg, myFinalizer)
        if err := r.Update(ctx, pg); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // 3. Проверить удаление
    if pg.DeletionTimestamp != nil {
        if controllerutil.ContainsFinalizer(pg, myFinalizer) {
            log.Info("cleanup before delete")
            controllerutil.RemoveFinalizer(pg, myFinalizer)
            if err := r.Update(ctx, pg); err != nil {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{}, nil
    }
    
    // 4. Проверить/создать StatefulSet
    sts := &appsv1.StatefulSet{}
    err := r.Get(ctx, types.NamespacedName{
        Name:      pg.Name + "-sts",
        Namespace: pg.Namespace,
    }, sts)
    
    if errors.IsNotFound(err) {
        sts = r.newStatefulSet(pg)
        if err := controllerutil.SetControllerReference(pg, sts, r.Scheme); err != nil {
            return ctrl.Result{}, err
        }
        if err := r.Create(ctx, sts); err != nil {
            return ctrl.Result{}, err
        }
        log.Info("created StatefulSet", "name", sts.Name)
    } else if err != nil {
        return ctrl.Result{}, err
    }
    
    // 5. Проверить/создать Service
    svc := &corev1.Service{}
    err = r.Get(ctx, types.NamespacedName{
        Name:      pg.Name,
        Namespace: pg.Namespace,
    }, svc)
    
    if errors.IsNotFound(err) {
        svc = r.newService(pg)
        if err := controllerutil.SetControllerReference(pg, svc, r.Scheme); err != nil {
            return ctrl.Result{}, err
        }
        if err := r.Create(ctx, svc); err != nil {
            return ctrl.Result{}, err
        }
        log.Info("created Service", "name", svc.Name)
    } else if err != nil {
        return ctrl.Result{}, err
    }
    
    // 6. Обновить status
    pg.Status.Ready = sts.Status.ReadyReplicas == *pg.Spec.Replicas
    pg.Status.Replicas = sts.Status.ReadyReplicas
    if err := r.Status().Update(ctx, pg); err != nil {
        return ctrl.Result{}, err
    }
    
    // 7. Периодическая проверка
    return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil
}

func (r *PostgreSQLReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&databasev1.PostgreSQL{}).
        Owns(&appsv1.StatefulSet{}).
        Owns(&corev1.Service{}).
        Complete(r)
}

func (r *PostgreSQLReconciler) newStatefulSet(pg *databasev1.PostgreSQL) *appsv1.StatefulSet {
    // ...
    return &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      pg.Name + "-sts",
            Namespace: pg.Namespace,
        },
        Spec: appsv1.StatefulSetSpec{
            Replicas: pg.Spec.Replicas,
            // ...
        },
    }
}

func (r *PostgreSQLReconciler) newService(pg *databasev1.PostgreSQL) *corev1.Service {
    // ...
    return &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      pg.Name,
            Namespace: pg.Namespace,
        },
        Spec: corev1.ServiceSpec{
            // ...
        },
    }
}
```

**Что даёт controller-runtime:**

- **Меньше boilerplate.**
- **Кэш из коробки.**
- **Метрики.**
- **Leader election.**
- **Webhooks.**

### 🔬 Практика: controller-runtime

```bash
# 1. Создать проект
mkdir postgres-operator && cd postgres-operator
go mod init github.com/example/postgres-operator

# 2. Установить зависимости
go get sigs.k8s.io/controller-runtime
go get k8s.io/api k8s.io/apimachinery k8s.io/client-go

# 3. Создать структуру
mkdir -p api/v1 controllers

# 4. API types (api/v1/postgresql_types.go)
# 5. Controllers (controllers/postgresql_controller.go)
# 6. Main (main.go)

# 7. Запустить
go run main.go
```

### 💡 Практика: как правильно использовать controller-runtime

**✅ ОБЯЗАТЕЛЬНО:**

1. **Manager** для всего.
2. **Builder** для контроллера.
3. **Client** для API.
4. **ctrl.Result** для управления.

**👍 СТОИТ:**

4. **Metrics** endpoint.
5. **Leader election** для HA.
6. **Webhooks** для validation.

**❌ НЕ ДЕЛАЙ:**

7. **Не пиши свой informer.** Используй controller-runtime.
8. **Не забывай про Requeue.**
9. **Не игнорируй ошибки.**

### Где мы сейчас

Мы разобрали controller-runtime. Теперь — **Kubebuilder** — пишем свой оператор.

---

## 14.8 Kubebuilder: пишем свой оператор

### 🔌 Проблема: нужен скелет оператора

Написать оператор с нуля — много boilerplate. Нужен инструмент для генерации скелета.

**Решение:** Kubebuilder.

### 📊 Что такое Kubebuilder

**Kubebuilder** — framework для создания операторов.

**Что даёт:**

- **Генерация** проекта.
- **Генерация** CRD из Go-типов.
- **Генерация** RBAC.
- **Генерация** манифестов.
- **Тестирование** через envtest.

### 🎯 Установка

```bash
# macOS
brew install kubebuilder

# Linux
curl -L -o kubebuilder https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)
chmod +x kubebuilder
sudo mv kubebuilder /usr/local/bin/
```

### 🎯 Создание проекта

```bash
mkdir postgres-operator && cd postgres-operator
kubebuilder init --domain example.com --repo github.com/example/postgres-operator
```

**Что создаст:**

```
postgres-operator/
├── config/
│   ├── crd/
│   ├── default/
│   ├── manager/
│   ├── rbac/
│   └── samples/
├── hack/
├── api/
├── controllers/
├── main.go
├── go.mod
├── Makefile
├── PROJECT
└── README.md
```

### 🎯 Создание API

```bash
kubebuilder create api --group database --version v1 --kind PostgreSQL
```

**Что создаст:**

- `api/v1/postgresql_types.go` — типы.
- `controllers/postgresql_controller.go` — контроллер.
- Обновит CRD в `config/crd/`.

### 🎯 Типы

**`api/v1/postgresql_types.go`:**

```go
package v1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// PostgreSQLSpec defines the desired state
type PostgreSQLSpec struct {
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=5
    Replicas int32 `json:"replicas"`
    
    // +kubebuilder:default="16"
    Version string `json:"version,omitempty"`
    
    // +kubebuilder:default="10Gi"
    Storage string `json:"storage,omitempty"`
    
    Backup *BackupSpec `json:"backup,omitempty"`
}

type BackupSpec struct {
    // +kubebuilder:default=true
    Enabled bool `json:"enabled,omitempty"`
    
    // +kubebuilder:default="0 2 * * *"
    Schedule string `json:"schedule,omitempty"`
}

// PostgreSQLStatus defines the observed state
type PostgreSQLStatus struct {
    Ready    bool   `json:"ready,omitempty"`
    Replicas int32  `json:"replicas,omitempty"`
    Master   string `json:"master,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Replicas",type=integer,JSONPath=`.spec.replicas`
// +kubebuilder:printcolumn:name="Ready",type=boolean,JSONPath=`.status.ready`
// +kubebuilder:printcolumn:name="Master",type=string,JSONPath=`.status.master`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`
type PostgreSQL struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   PostgreSQLSpec   `json:"spec,omitempty"`
    Status PostgreSQLStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true
type PostgreSQLList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []PostgreSQL `json:"items"`
}

func init() {
    SchemeBuilder.Register(&PostgreSQL{}, &PostgreSQLList{})
}
```

**Что даёт:**

- **Go-типы** для CRD.
- **Kubebuilder markers** для генерации:
  - `+kubebuilder:validation:Minimum` — валидация.
  - `+kubebuilder:default` — default.
  - `+kubebuilder:subresource:status` — status subresource.
  - `+kubebuilder:printcolumn` — printer columns.

### 🎯 Генерация CRD

```bash
make manifests
```

**Что произойдёт:**

- Из Go-типов генерируется **CRD YAML** в `config/crd/bases/`.
- Генерируются RBAC манифесты.

**Проверить:**

```bash
cat config/crd/bases/database.example.com_postgresqls.yaml
```

### 🎯 Контроллер

**`controllers/postgresql_controller.go`:**

```go
package controllers

import (
    "context"
    "time"
    
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/api/resource"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/types"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/log"
    
    databasev1 "github.com/example/postgres-operator/api/v1"
)

const postgresFinalizer = "database.example.com/finalizer"

type PostgreSQLReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=database.example.com,resources=postgresqls,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=database.example.com,resources=postgresqls/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete

func (r *PostgreSQLReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)
    
    // 1. Получить PostgreSQL
    pg := &databasev1.PostgreSQL{}
    if err := r.Get(ctx, req.NamespacedName, pg); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 2. Добавить finalizer
    if !controllerutil.ContainsFinalizer(pg, postgresFinalizer) {
        controllerutil.AddFinalizer(pg, postgresFinalizer)
        if err := r.Update(ctx, pg); err != nil {
            return ctrl.Result{}, err
        }
    }
    
    // 3. Проверить удаление
    if pg.DeletionTimestamp != nil {
        if controllerutil.ContainsFinalizer(pg, postgresFinalizer) {
            log.Info("cleanup before delete")
            controllerutil.RemoveFinalizer(pg, postgresFinalizer)
            if err := r.Update(ctx, pg); err != nil {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{}, nil
    }
    
    // 4. Reconcile StatefulSet
    sts := &appsv1.StatefulSet{}
    err := r.Get(ctx, types.NamespacedName{
        Name:      pg.Name,
        Namespace: pg.Namespace,
    }, sts)
    
    if errors.IsNotFound(err) {
        sts = r.newStatefulSet(pg)
        if err := controllerutil.SetControllerReference(pg, sts, r.Scheme); err != nil {
            return ctrl.Result{}, err
        }
        if err := r.Create(ctx, sts); err != nil {
            return ctrl.Result{}, err
        }
        log.Info("created StatefulSet", "name", sts.Name)
    } else if err != nil {
        return ctrl.Result{}, err
    }
    
    // 5. Reconcile Service
    svc := &corev1.Service{}
    err = r.Get(ctx, types.NamespacedName{
        Name:      pg.Name,
        Namespace: pg.Namespace,
    }, svc)
    
    if errors.IsNotFound(err) {
        svc = r.newService(pg)
        if err := controllerutil.SetControllerReference(pg, svc, r.Scheme); err != nil {
            return ctrl.Result{}, err
        }
        if err := r.Create(ctx, svc); err != nil {
            return ctrl.Result{}, err
        }
        log.Info("created Service", "name", svc.Name)
    } else if err != nil {
        return ctrl.Result{}, err
    }
    
    // 6. Обновить status
    pg.Status.Ready = sts.Status.ReadyReplicas == *pg.Spec.Replicas
    pg.Status.Replicas = sts.Status.ReadyReplicas
    if err := r.Status().Update(ctx, pg); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil
}

func (r *PostgreSQLReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&databasev1.PostgreSQL{}).
        Owns(&appsv1.StatefulSet{}).
        Owns(&corev1.Service{}).
        Complete(r)
}

func (r *PostgreSQLReconciler) newStatefulSet(pg *databasev1.PostgreSQL) *appsv1.StatefulSet {
    labels := map[string]string{
        "app": pg.Name,
    }
    
    return &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      pg.Name,
            Namespace: pg.Namespace,
            Labels:    labels,
        },
        Spec: appsv1.StatefulSetSpec{
            ServiceName: pg.Name,
            Replicas:    pg.Spec.Replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: labels,
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: labels,
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  "postgres",
                            Image: "postgres:" + pg.Spec.Version,
                            Ports: []corev1.ContainerPort{
                                {ContainerPort: 5432, Name: "postgres"},
                            },
                            VolumeMounts: []corev1.VolumeMount{
                                {
                                    Name:      "data",
                                    MountPath: "/var/lib/postgresql/data",
                                },
                            },
                        },
                    },
                },
            },
            VolumeClaimTemplates: []corev1.PersistentVolumeClaim{
                {
                    ObjectMeta: metav1.ObjectMeta{
                        Name: "data",
                    },
                    Spec: corev1.PersistentVolumeClaimSpec{
                        AccessModes: []corev1.PersistentVolumeAccessMode{
                            corev1.ReadWriteOnce,
                        },
                        Resources: corev1.ResourceRequirements{
                            Requests: corev1.ResourceList{
                                corev1.ResourceStorage: resource.MustParse(pg.Spec.Storage),
                            },
                        },
                    },
                },
            },
        },
    }
}

func (r *PostgreSQLReconciler) newService(pg *databasev1.PostgreSQL) *corev1.Service {
    return &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      pg.Name,
            Namespace: pg.Namespace,
        },
        Spec: corev1.ServiceSpec{
            ClusterIP: "None",  // headless
            Selector: map[string]string{
                "app": pg.Name,
            },
            Ports: []corev1.ServicePort{
                {Port: 5432, Name: "postgres"},
            },
        },
    }
}
```

### 🎯 RBAC

**Kubebuilder markers:**

```go
// +kubebuilder:rbac:groups=database.example.com,resources=postgresqls,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=database.example.com,resources=postgresqls/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
```

**Генерация:**

```bash
make manifests
```

**Что даёт:** RBAC-манифесты в `config/rbac/`.

### 🎯 Установка

```bash
# Установить CRD
make install

# Запустить контроллер локально
make run

# В другом терминале — создать instance
kubectl apply -f config/samples/database_v1_postgresql.yaml
```

### 🎯 Деплой в кластер

```bash
# Собрать образ
make docker-build IMG=myregistry/postgres-operator:v1.0.0

# Запушить
make docker-push IMG=myregistry/postgres-operator:v1.0.0

# Задеплоить
make deploy IMG=myregistry/postgres-operator:v1.0.0
```

### 🎯 Тестирование

**envtest** — тестовый apiserver без кластера.

```bash
make test
```

**Что даёт:** unit-тесты для контроллера.

### 🔬 Практика: Kubebuilder

```bash
# 1. Установить
brew install kubebuilder

# 2. Создать проект
mkdir postgres-operator && cd postgres-operator
kubebuilder init --domain example.com --repo github.com/example/postgres-operator

# 3. Создать API
kubebuilder create api --group database --version v1 --kind PostgreSQL
# Create Resource [y/n]: y
# Create Controller [y/n]: y

# 4. Отредактировать api/v1/postgresql_types.go
# 5. Отредактировать controllers/postgresql_controller.go

# 6. Генерация
make manifests
make generate

# 7. Установить CRD
make install

# 8. Запустить контроллер
make run

# 9. В другом терминале
kubectl apply -f config/samples/database_v1_postgresql.yaml
kubectl get postgresqls

# 10. Деплой
make docker-build IMG=myregistry/postgres-operator:v1.0.0
make deploy IMG=myregistry/postgres-operator:v1.0.0
```

### 💡 Практика: как правильно использовать Kubebuilder

**✅ ОБЯЗАТЕЛЬНО:**

1. **Kubebuilder init** для проекта.
2. **create api** для CRD.
3. **Markers** для генерации.
4. **make manifests** после изменений.

**👍 СТОИТ:**

4. **envtest** для тестов.
5. **Версионирование** CRD.
6. **Webhooks** для validation.

**❌ НЕ ДЕЛАЙ:**

7. **Не пиши вручную** CRD YAML.
8. **Не забывай про RBAC markers.**
9. **Не игнорируй тесты.**

### Где мы сейчас

Мы разобрали Kubebuilder. Теперь — **готовые операторы**.

---

## 14.9 Готовые операторы

### 🔌 Проблема: не всегда нужно писать свой

Ты хочешь PostgreSQL в Kubernetes. Писать свой оператор — недели. Есть ли готовый?

**Ответ:** да.

### 📊 Популярные операторы

| Оператор | Для чего |
|:---|:---|
| **Prometheus Operator** | Prometheus, Alertmanager, ServiceMonitor |
| **Strimzi** | Kafka |
| **Zalando Postgres Operator** | PostgreSQL |
| **Crunchy Data PGO** | PostgreSQL |
| **Percona Operators** | PostgreSQL, MySQL, MongoDB |
| **Redis Operator** | Redis |
| **Elastic Cloud on K8s (ECK)** | Elasticsearch |
| **Cert Manager** | TLS-сертификаты |
| **External Secrets Operator** | Секреты из Vault |
| **Argo CD** | GitOps |
| **KEDA** | Autoscaling |
| **Crossplane** | Infrastructure as Code |

### 🎯 Prometheus Operator

**Что делает:**

- **Управляет** Prometheus, Alertmanager, Thanos.
- **ServiceMonitor** — как опрашивать сервис.
- **PodMonitor** — как опрашивать Pod'ы.
- **PrometheusRule** — правила алертов и recording.
- **Probe** — blackbox checks.

**Пример:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: metrics
      interval: 15s
```

**Разбирали в Главе 22.**

### 🎯 Strimzi

**Что делает:**

- **Управляет** Kafka, Zookeeper, Kafka Connect, MirrorMaker.
- **Топики** как CRD.
- **Пользователи** как CRD.
- **TLS, SASL.**

**Пример:**

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    replicas: 3
    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 100Gi
          deleteClaim: false
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      deleteClaim: false
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

**Что даёт:** Kafka-кластер с 3 брокерами, ZooKeeper, TLS, topic operator, user operator. Одна CRD.

### 🎯 Zalando Postgres Operator

**Что делает:**

- **Управляет** PostgreSQL-кластерами.
- **Replication** streaming.
- **Failover** автоматический.
- **Backup** через WAL-G, pgBackRest.
- **Connection pooler** (PgBouncer).
- **Users, databases** как CRD.

**Пример:**

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
  resources:
    requests:
      cpu: 500m
      memory: 2Gi
    limits:
      cpu: 2
      memory: 4Gi
  patroni:
    initdb:
      encoding: "UTF8"
    pg_hba:
      - host all all 0.0.0.0/0 md5
  backup:
    schedule: "0 2 * * *"
    retention:
      count: 30
```

**Что даёт:**

- **3 реплики** PostgreSQL.
- **Patroni** для failover.
- **Streaming replication.**
- **PgBouncer** для connection pooling.
- **Backup** каждый день в 2 ночи.
- **Retention** 30 дней.
- **User `myapp`** с правами.
- **Database `mydb`.**

**Всё — одна CRD.**

### 🎯 Cert Manager

**Что делает:**

- **Управляет** TLS-сертификатами.
- **Let's Encrypt** интеграция.
- **Автоматический** выпуск и ротация.
- **Issuer, ClusterIssuer, Certificate** как CRD.

**Пример:**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-tls
spec:
  secretName: myapp-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - myapp.example.com
```

**Что даёт:** сертификат от Let's Encrypt для `myapp.example.com`. Автоматически.

### 🎯 External Secrets Operator

**Что делает:**

- **Интеграция** с Vault, AWS SM, GCP SM, Azure KV.
- **ExternalSecret** как CRD.
- **Синхронизация** секретов в K8s.

**Пример:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-password
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault
    kind: SecretStore
  target:
    name: db-password
  data:
    - secretKey: password
      remoteRef:
        key: secret/data/db
        property: password
```

**Разбирали в Главе 12.**

### 🎯 Crossplane

**Что делает:**

- **Infrastructure as Code** через Kubernetes.
- **Управляет** облачными ресурсами (RDS, S3, ...).
- **Composite Resources** — свои абстракции.

**Пример:**

```yaml
apiVersion: database.example.org/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: my-db
spec:
  parameters:
    storageGB: 20
    environment: dev
  writeConnectionSecretToRef:
    name: my-db-conn
```

**Что даёт:** RDS в AWS + Secret с connection string.

### 🎯 Как выбрать

**Правило:**

1. **Есть готовый оператор** → используй его.
2. **Нет готового** → пиши свой.

**Где искать:**

- **OperatorHub.io** — каталог.
- **Artifact Hub** — каталог.
- **GitHub** — поиск.
- **Awesome Operators** — список.

### 🎯 Оценка оператора

**Что проверить:**

1. **Capability level** — Level 5?
2. **Активность** — обновляется?
3. **Документация** — есть?
4. **Сообщество** — есть?
5. **Тесты** — есть?
6. **Production users** — есть?

**Правило:** не используй оператор без production users.

### 🔬 Практика: Prometheus Operator

```bash
# 1. Установить
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack

# 2. ServiceMonitor
cat > servicemonitor.yaml <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: metrics
      interval: 15s
EOF
kubectl apply -f servicemonitor.yaml

# 3. Проверить
kubectl get servicemonitors
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
# http://localhost:9090/targets
```

### 💡 Практика: как правильно использовать готовые операторы

**✅ ОБЯЗАТЕЛЬНО:**

1. **Готовый оператор** перед написанием своего.
2. **Production users** проверить.
3. **Документацию** изучить.

**👍 СТОИТ:**

4. **OperatorHub** для поиска.
5. **Capability level** — Level 4+.
6. **Сообщество** активно.

**❌ НЕ ДЕЛАЙ:**

7. **Не пиши свой** для типовых задач.
8. **Не используй заброшенные** операторы.
9. **Не забывай про обновления.**

### Где мы сейчас

Мы разобрали готовые операторы. Теперь — **отладка операторов**.

---

## 14.10 Отладка операторов

### 🔌 Проблема: оператор не работает

Оператор развёрнут. CRD создан. Instance создан. Но ничего не происходит.

### 🔍 Типичные проблемы

**1. Оператор не запущен.**

```bash
kubectl get pods -n myoperator-system
# NAME                              READY   STATUS
# myoperator-controller-manager-0   1/1     Running  ✓
```

**Если не Running:**

```bash
kubectl logs -n myoperator-system deployment/myoperator-controller-manager
kubectl describe pod -n myoperator-system myoperator-controller-manager-xxx
```

**2. CRD не установлен.**

```bash
kubectl get crd | grep postgresql
# postgresqls.database.example.com   ✓
```

**Если нет:**

```bash
make install
```

**3. Instance не создан.**

```bash
kubectl get postgresqls
# NAME          AGE
# my-postgres   5m
```

**4. Оператор не видит Instance.**

**Диагностика:**

```bash
# Логи оператора
kubectl logs -n myoperator-system deployment/myoperator-controller-manager

# Ищи "Reconciling PostgreSQL"
# Если нет — watch не работает
```

**Причины:**

- **RBAC** — нет прав на watch.
- **Namespace** — оператор смотрит в другом namespace.
- **Label selector** — фильтр.

**5. Reconcile fails.**

**Диагностика:**

```bash
kubectl logs -n myoperator-system deployment/myoperator-controller-manager | grep -i error
```

**Пример:**

```
2026-01-15T10:00:00Z ERROR Reconciler error
  {"controller": "postgresql", "error": "failed to create StatefulSet: ..."}
```

**Причины:**

- **RBAC** — нет прав на create StatefulSet.
- **Ошибка в коде.**
- **Невалидные данные.**

**6. Status не обновляется.**

**Диагностика:**

```bash
kubectl get postgresql my-postgres -o yaml | grep -A 5 status
# status:
#   ready: false
#   replicas: 0
```

**Причины:**

- **RBAC** — нет прав на update status.
- **Subresource** не настроен.
- **Ошибка в коде.**

**7. StatefulSet не создан.**

**Диагностика:**

```bash
kubectl get statefulset
# Если нет — оператор не создал

# Логи оператора
kubectl logs -n myoperator-system deployment/myoperator-controller-manager | grep "StatefulSet"
```

### 🎯 Инструменты отладки

**1. Логи.**

```bash
# Оператор
kubectl logs -n myoperator-system deployment/myoperator-controller-manager -f

# С уровнем
kubectl logs -n myoperator-system deployment/myoperator-controller-manager --tail=100
```

**2. Events.**

```bash
kubectl get events --sort-by=.lastTimestamp -A
kubectl get events --field-selector involvedObject.name=my-postgres
```

**3. Describe.**

```bash
kubectl describe postgresql my-postgres
# Events:
#   Normal  Created  StatefulSet my-postgres created
#   Warning Failed   Failed to create Service: ...
```

**4. Status.**

```bash
kubectl get postgresql my-postgres -o yaml
```

**5. Exec в оператор.**

```bash
kubectl exec -it -n myoperator-system deployment/myoperator-controller-manager -- sh
```

**6. Metrics.**

```bash
kubectl port-forward -n myoperator-system deployment/myoperator-controller-manager 8080:8080
curl localhost:8080/metrics | grep controller_runtime
```

**Что смотреть:**

- `controller_runtime_reconcile_total` — всего reconcile.
- `controller_runtime_reconcile_errors_total` — ошибки.
- `controller_runtime_reconcile_time_seconds` — длительность.
- `workqueue_depth` — длина очереди.

### 🎯 Типичные ошибки

**1. RBAC.**

**Симптом:**

```
ERROR Reconciler error
  {"error": "statefulsets.apps is forbidden: User \"system:serviceaccount:myoperator-system:myoperator-controller-manager\" cannot create resource \"statefulsets\""}
```

**Решение:** добавить RBAC markers:

```go
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
```

**2. Webhook.**

**Симптом:**

```
ERROR Failed to create webhook
```

**Решение:** проверить сертификаты, webhook configuration.

**3. Finalizer застрял.**

**Симптом:**

```bash
kubectl get postgresql my-postgres
# NAME          AGE
# my-postgres   1h    ← не удаляется

kubectl get postgresql my-postgres -o yaml | grep finalizers
# finalizers:
#   - database.example.com/finalizer
```

**Решение:** вручную удалить finalizer:

```bash
kubectl patch postgresql my-postgres -p '{"metadata":{"finalizers":null}}' --type=merge
```

**4. Owner reference конфликт.**

**Симптом:**

```
ERROR Reconciler error
  {"error": "Object my-postgres is already owned by another PostgreSQL"}
```

**Решение:** проверить `ownerReferences`.

**5. Version mismatch.**

**Симптом:** CRD версия v1, но оператор ожидает v1beta1.

**Решение:** обновить CRD или оператор.

### 🎯 Local development

**Запуск оператора локально:**

```bash
make run
```

**Что даёт:**

- Оператор запускается на ноутбуке.
- Подключается к кластеру через kubeconfig.
- Легко отлаживать в IDE.

**Преимущества:**

- **Debugger** (Delve, GDB).
- **Breakpoints.**
- **Быстрая итерация.**

### 🎯 envtest

**envtest** — тестовый apiserver + etcd без кластера.

```bash
make test
```

**Что даёт:**

- **Unit-тесты** для контроллера.
- **Быстро.**
- **Без кластера.**

**Пример теста:**

```go
func TestReconcile(t *testing.T) {
    // Создать PostgreSQL instance
    pg := &databasev1.PostgreSQL{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test",
            Namespace: "default",
        },
        Spec: databasev1.PostgreSQLSpec{
            Replicas: 3,
        },
    }
    
    // Reconcile
    _, err := reconciler.Reconcile(ctx, reconcile.Request{
        NamespacedName: types.NamespacedName{
            Name:      "test",
            Namespace: "default",
        },
    })
    assert.NoError(t, err)
    
    // Проверить StatefulSet
    sts := &appsv1.StatefulSet{}
    err = k8sClient.Get(ctx, types.NamespacedName{
        Name:      "test",
        Namespace: "default",
    }, sts)
    assert.NoError(t, err)
    assert.Equal(t, int32(3), *sts.Spec.Replicas)
}
```

### 🔬 Практика: отладка

```bash
# 1. Проверить Pod оператора
kubectl get pods -n myoperator-system

# 2. Логи
kubectl logs -n myoperator-system deployment/myoperator-controller-manager --tail=100

# 3. Events
kubectl get events -A --sort-by=.lastTimestamp | grep -i postgres

# 4. Describe CRD instance
kubectl describe postgresql my-postgres

# 5. Status
kubectl get postgresql my-postgres -o yaml

# 6. Проверить дочерние ресурсы
kubectl get statefulset,service -l app=my-postgres

# 7. Metrics
kubectl port-forward -n myoperator-system deployment/myoperator-controller-manager 8080:8080
curl localhost:8080/metrics | grep controller_runtime

# 8. RBAC
kubectl auth can-i create statefulsets \
  --as=system:serviceaccount:myoperator-system:myoperator-controller-manager

# 9. Локальный запуск
make run

# 10. Тесты
make test
```

### 💡 Практика: как правильно отлаживать операторы

**✅ ОБЯЗАТЕЛЬНО:**

1. **Логи оператора** — первый источник.
2. **Events** на CRD instance.
3. **Describe** CRD instance.
4. **Metrics** для мониторинга.

**👍 СТОИТ:**

4. **Local run** для отладки.
5. **envtest** для тестов.
6. **Debugger** для сложных случаев.

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй ошибки в логах.**
8. **Не забывай про RBAC.**
9. **Не оставляй застрявшие finalizers.**

### Где мы сейчас

Мы разобрали отладку операторов. Теперь — **когда писать свой оператор**.

---

## 14.11 Когда писать свой оператор

### 🔌 Проблема: писать или не писать

У тебя задача. Нужен ли свой оператор?

### 📊 Когда писать

**✅ Писать, если:**

1. **Нет готового** оператора для задачи.
2. **Специфичная** доменная логика.
3. **Сложный** lifecycle (backup, failover).
4. **Интеграция** с внутренними системами.
5. **Много** повторяющихся действий.

**Примеры:**

- **Внутренний сервис** с особыми правилами.
- **Интеграция** с legacy-системой.
- **Кастомный** workflow.

### 📊 Когда не писать

**❌ Не писать, если:**

1. **Есть готовый** оператор.
2. **Простая** задача (ConfigMap, Deployment).
3. **Одноразовая** задача.
4. **Helm-чарт** справится.
5. **Нет команды** для поддержки.

**Примеры:**

- **PostgreSQL** — используй Zalando/Crunchy.
- **Kafka** — используй Strimzi.
- **Prometheus** — используй Prometheus Operator.
- **Простое приложение** — Helm.

### 🎯 Критерии

**Писать оператор, если:**

- **3+ из следующих:**
  - Нет готового оператора.
  - Специфичная логика.
  - Сложный lifecycle.
  - Много ручной работы.
  - Есть команда для поддержки.

**Не писать, если:**

- **1-2 из следующих:**
  - Есть готовый оператор.
  - Простая задача.
  - Нет команды.

### 🎯 Альтернативы

**1. Helm-чарт.**

Для простых приложений. Не требует кода.

**2. Kustomize.**

Для конфигурации.

**3. Скрипты.**

Для одноразовых задач.

**4. CronJob.**

Для периодических задач.

**5. Ansible Operator.**

Для простых операторов без Go.

**6. Helm Operator.**

Для операторов на основе Helm-чартов.

### 🎯 Ansible Operator

**Ansible Operator** — оператор на Ansible.

**Что даёт:**

- **Не нужно** писать Go.
- **Ansible-роли** как логика.
- **Быстрее** разработка.

**Пример:**

```yaml
# watches.yaml
- version: v1
  group: database.example.com
  kind: PostgreSQL
  playbook: playbooks/postgresql.yml
```

**Когда:** для простых операторов, команда знает Ansible.

### 🎯 Helm Operator

**Helm Operator** — оператор на основе Helm-чарта.

**Что даёт:**

- **Helm-чарт** как логика.
- **CRD** для параметров.
- **Быстрее** разработка.

**Пример:**

```yaml
apiVersion: helm.operator-sdk/v1alpha1
kind: PostgreSQL
metadata:
  name: my-postgres
spec:
  replicas: 3
  version: "16"
```

**Когда:** для приложений, которые уже упакованы в Helm.

### 🎯 Оценка стоимости

**Свой оператор:**

- **Разработка:** 2-8 недель.
- **Тестирование:** 1-2 недели.
- **Поддержка:** постоянно.
- **Команда:** 1-2 инженера.

**Готовый оператор:**

- **Установка:** часы.
- **Настройка:** дни.
- **Поддержка:** сообщество.
- **Команда:** 0.

**Правило:** не пиши свой оператор, если есть готовый.

### 🎯 Пример: PostgreSQL

**Вариант 1: Свой оператор.**

- **Разработка:** 4-8 недель.
- **Тестирование:** 2 недели.
- **Поддержка:** постоянно.

**Вариант 2: Zalando Postgres Operator.**

- **Установка:** часы.
- **Настройка:** дни.
- **Production users:** сотни.

**Вывод:** используй готовый.

### 🎯 Пример: внутренний сервис

**Задача:** развернуть внутренний сервис с особыми правилами.

**Вариант 1: Helm-чарт.**

- **Просто.**
- **Быстро.**
- **Но:** нет автоматизации lifecycle.

**Вариант 2: Свой оператор.**

- **Сложнее.**
- **Долго.**
- **Но:** полная автоматизация.

**Вывод:** зависит от требований. Если нужен сложный lifecycle — оператор. Иначе — Helm.

### 🔬 Практика: оценка

**Чек-лист перед написанием оператора:**

- [ ] Нет готового оператора?
- [ ] Специфичная доменная логика?
- [ ] Сложный lifecycle (backup, failover)?
- [ ] Много ручной работы?
- [ ] Есть команда для поддержки?
- [ ] Бюджет 2-8 недель?
- [ ] Есть опыт в Go/Kubebuilder?

**Если 5+ ответов «да» — пиши. Иначе — ищи альтернативы.**

### 💡 Практика: как принимать решение

**✅ ОБЯЗАТЕЛЬНО:**

1. **Искать готовый** оператор.
2. **Оценивать стоимость** своего.
3. **Начинать с Helm** для простого.

**👍 СТОИТ:**

4. **Ansible/Helm Operator** для простых случаев.
5. **Команда** для поддержки.
6. **Production users** у готовых.

**❌ НЕ ДЕЛАЙ:**

7. **Не пиши свой** для типовых задач.
8. **Не недооценивай** стоимость поддержки.
9. **Не забывай про обновления.**

### Где мы сейчас

Мы разобрали, когда писать свой оператор. Теперь — **диагностика проблем**.

---

## 14.12 Диагностика проблем

### 🔌 Проблема: оператор не работает

Оператор развёрнут. CRD создан. Но что-то не так.

### 🔍 Типичные проблемы

**1. CRD не найден.**

```bash
kubectl get crd | grep postgresql
# Нет результата
```

**Решение:**

```bash
make install
# или
kubectl apply -f config/crd/bases/
```

**2. Instance не создаётся.**

```bash
kubectl apply -f instance.yaml
# Error: PostgreSQL.database.example.com "my-postgres" is invalid:
# spec.replicas: Invalid value: 10: should be less than or equal to 5
```

**Диагностика:**

```bash
kubectl describe crd postgresqls.database.example.com
kubectl explain postgresql.spec
```

**3. Оператор не видит Instance.**

**Диагностика:**

```bash
# Логи оператора
kubectl logs -n myoperator-system deployment/myoperator-controller-manager | grep "Reconciling"

# Если нет — watch не работает
kubectl describe pod -n myoperator-system myoperator-controller-manager-xxx
```

**Причины:**

- **RBAC** — нет прав на watch.
- **Namespace** — оператор смотрит в другом namespace.
- **Label selector** — фильтр.

**4. Reconcile fails.**

```bash
kubectl logs -n myoperator-system deployment/myoperator-controller-manager | grep -i error
```

**Примеры:**

- `statefulsets.apps is forbidden` — RBAC.
- `failed to create Service` — ошибка в коде.
- `failed to update status` — subresource.

**5. Finalizer застрял.**

```bash
kubectl get postgresql my-postgres
# NAME          AGE
# my-postgres   1h    ← не удаляется
```

**Решение:**

```bash
kubectl patch postgresql my-postgres -p '{"metadata":{"finalizers":null}}' --type=merge
```

**6. Дочерние ресурсы не создаются.**

```bash
kubectl get statefulset -l app=my-postgres
# Нет результата
```

**Диагностика:**

```bash
kubectl logs -n myoperator-system deployment/myoperator-controller-manager | grep "StatefulSet"
```

**7. Status не обновляется.**

```bash
kubectl get postgresql my-postgres -o yaml | grep -A 5 status
# status: {}    ← пусто
```

**Причины:**

- **Subresource** не настроен.
- **RBAC** — нет прав на status update.

**8. Operator падает.**

```bash
kubectl get pods -n myoperator-system
# NAME                              READY   STATUS             RESTARTS
# myoperator-controller-manager-0   0/1     CrashLoopBackOff   5
```

**Диагностика:**

```bash
kubectl logs -n myoperator-system deployment/myoperator-controller-manager --previous
kubectl describe pod -n myoperator-system myoperator-controller-manager-xxx
```

### 🎯 Алгоритм диагностики

**1. Оператор работает?**

```bash
kubectl get pods -n myoperator-system
```

**2. CRD установлен?**

```bash
kubectl get crd | grep <kind>
```

**3. Instance создан?**

```bash
kubectl get <kind>
```

**4. Логи оператора?**

```bash
kubectl logs -n myoperator-system deployment/myoperator-controller-manager
```

**5. Events?**

```bash
kubectl get events -A --sort-by=.lastTimestamp
```

**6. Describe instance?**

```bash
kubectl describe <kind> <name>
```

**7. Status instance?**

```bash
kubectl get <kind> <name> -o yaml
```

**8. Дочерние ресурсы?**

```bash
kubectl get all -l <label>
```

### 🎯 Инструменты

**1. Логи.**

```bash
kubectl logs -n myoperator-system deployment/myoperator-controller-manager -f
```

**2. Metrics.**

```bash
kubectl port-forward -n myoperator-system deployment/myoperator-controller-manager 8080:8080
curl localhost:8080/metrics | grep controller_runtime
```

**3. Events.**

```bash
kubectl get events -A --field-selector involvedObject.kind=PostgreSQL
```

**4. RBAC check.**

```bash
kubectl auth can-i create statefulsets \
  --as=system:serviceaccount:myoperator-system:myoperator-controller-manager
```

**5. Local run.**

```bash
make run
```

### 🎯 Частые ошибки

**1. RBAC.**

**Симптом:**

```
ERROR Reconciler error
  {"error": "statefulsets.apps is forbidden"}
```

**Решение:** добавить RBAC markers.

**2. Finalizer.**

**Симптом:**

```
kubectl delete postgresql my-postgres
# Ничего не происходит
```

**Решение:** удалить finalizer вручную.

**3. Owner reference.**

**Симптом:**

```
ERROR Object my-postgres is already owned by another PostgreSQL
```

**Решение:** проверить `ownerReferences`.

**4. Version mismatch.**

**Симптом:** CRD v1, оператор ожидает v1beta1.

**Решение:** обновить CRD или оператор.

**5. Webhook.**

**Симптом:**

```
ERROR Failed to create webhook
```

**Решение:** проверить сертификаты, webhook configuration.

### 🔬 Практика: диагностика

```bash
# 1. Оператор работает?
kubectl get pods -n myoperator-system

# 2. CRD?
kubectl get crd | grep postgresql

# 3. Instance?
kubectl get postgresqls

# 4. Логи
kubectl logs -n myoperator-system deployment/myoperator-controller-manager --tail=100

# 5. Events
kubectl get events -A --sort-by=.lastTimestamp | grep -i postgres

# 6. Describe instance
kubectl describe postgresql my-postgres

# 7. Status
kubectl get postgresql my-postgres -o yaml

# 8. Дочерние ресурсы
kubectl get statefulset,service -l app=my-postgres

# 9. Metrics
kubectl port-forward -n myoperator-system deployment/myoperator-controller-manager 8080:8080
curl localhost:8080/metrics | grep controller_runtime

# 10. RBAC
kubectl auth can-i create statefulsets \
  --as=system:serviceaccount:myoperator-system:myoperator-controller-manager

# 11. Local run
make run

# 12. Тесты
make test
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Логи оператора** — первый источник.
2. **Events** на CRD instance.
3. **Describe** CRD instance.
4. **Metrics** для мониторинга.

**👍 СТОИТ:**

4. **Local run** для отладки.
5. **envtest** для тестов.
6. **Debugger** для сложных случаев.

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй ошибки в логах.**
8. **Не забывай про RBAC.**
9. **Не оставляй застрявшие finalizers.**

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **Reconciliation loop** | Цикл приведения к desired state. |
| **Desired state** | Желаемое состояние (из spec). |
| **Current state** | Текущее состояние (из status). |
| **Level-triggered** | Реагирует на состояние, не на события. |
| **Controller pattern** | Паттерн для реализации контроллера. |
| **Informer** | Кэш объектов из API. |
| **Work queue** | Очередь объектов для обработки. |
| **Rate limiter** | Ограничение частоты retries. |
| **CRD** | Custom Resource Definition. |
| **Custom Resource** | Экземпляр CRD. |
| **Operator** | Контроллер для CRD с доменными знаниями. |
| **Capability level** | Уровень зрелости оператора. |
| **Owner reference** | Ссылка на родительский объект. |
| **Finalizer** | Блокировка удаления для cleanup. |
| **controller-runtime** | Библиотека для контроллеров. |
| **Manager** | Управляет контроллерами. |
| **Reconciler** | Интерфейс с методом Reconcile. |
| **Builder** | DSL для создания контроллера. |
| **Kubebuilder** | Framework для операторов. |
| **envtest** | Тестовый apiserver без кластера. |
| **Prometheus Operator** | Оператор для Prometheus. |
| **Strimzi** | Оператор для Kafka. |
| **Zalando Postgres Operator** | Оператор для PostgreSQL. |
| **Cert Manager** | Оператор для TLS. |
| **Crossplane** | IaC через Kubernetes. |
| **Ansible Operator** | Оператор на Ansible. |
| **Helm Operator** | Оператор на Helm. |

---

## Что мы узнали?

- **Reconciliation loop** — основа Kubernetes. Level-triggered, idempotent, eventually consistent.
- **Controller pattern** — informer, work queue, reconciler, metrics.
- **CRD** — расширение API. OpenAPI schema, subresources, printer columns.
- **Operator pattern** — CRD + контроллер + доменные знания. Capability levels 1-5.
- **Informers** — кэш объектов. Shared informers. Event handlers.
- **Work queues** — rate limiting, retries, дедупликация.
- **Ownership** — owner references для каскадного удаления.
- **Finalizers** — cleanup перед удалением. Не забывать удалять.
- **controller-runtime** — Manager, Reconciler, Builder, Client.
- **Kubebuilder** — генерация проекта, CRD, RBAC, манифестов.
- **Готовые операторы** — Prometheus, Strimzi, Zalando, Cert Manager, Crossplane.
- **Отладка** — логи, events, describe, metrics, local run.
- **Когда писать свой** — только если нет готового и есть команда.

---

## Типичные ошибки

- ❌ **Reconcile не идемпотентен.** Retry ломает.
- ❌ **Хранить состояние** в контроллере.
- ❌ **Читать API напрямую** без informer.
- ❌ **Не использовать work queue.** Нет rate limiting.
- ❌ **Забыть про RBAC markers.**
- ❌ **Не удалять finalizer.** Объект не удалится.
- ❌ **Не использовать owner references.** Дочерние ресурсы не удалятся.
- ❌ **Писать свой оператор** для типовой задачи.
- ❌ **Не тестировать** оператор.
- ❌ **Не мониторить** оператор.
- ❌ **Игнорировать ошибки** в логах.
- ❌ **Не использовать RequeueAfter** для периодических задач.

---

## Для быстрого повторения

- **Reconciliation:** desired vs current → reconcile → status update.
- **Level-triggered:** состояние, не события.
- **Controller pattern:** informer → work queue → reconciler.
- **CRD:** OpenAPI schema, subresources, printer columns.
- **Operator:** CRD + контроллер + доменные знания.
- **Capability levels:** 1-5.
- **Ownership:** owner references для каскадного удаления.
- **Finalizers:** cleanup перед удалением.
- **controller-runtime:** Manager, Reconciler, Builder, Client.
- **Kubebuilder:** init, create api, make manifests.
- **Готовые операторы:** Prometheus, Strimzi, Zalando.
- **Отладка:** логи, events, describe, metrics.
- **Когда писать:** нет готового + специфичная логика + команда.

---

## Вопросы для самопроверки

1. Что такое reconciliation loop? Как работает?
2. Чем level-triggered отличается от edge-triggered?
3. Что такое controller pattern? Компоненты?
4. Что такое informer? Зачем нужен?
5. Что такое work queue? Что даёт?
6. Что такое CRD? Как создать?
7. Что такое operator pattern? Чем отличается от контроллера?
8. Что такое capability levels?
9. Что такое owner references? Зачем нужны?
10. Что такое finalizers? Как работают?
11. Что такое controller-runtime? Компоненты?
12. Что такое Kubebuilder? Что генерирует?
13. Какие готовые операторы знаешь?
14. Как отлаживать оператор?
15. Когда писать свой оператор?

---

## Ответы

**1. Reconciliation loop**

Бесконечный цикл: получить desired state, получить current state, сравнить, привести. Level-triggered, idempotent, eventually consistent.

**2. Level vs edge**

Level-triggered — реагирует на состояние. Edge-triggered — на события. Kubernetes использует level-triggered: даже если событие пропущено — periodic resync обнаружит drift.

**3. Controller pattern**

Informer (кэш), work queue (очередь), reconciler (обработка), event handlers, metrics. Informer watch API, queue обрабатывает, reconciler приводит к desired.

**4. Informer**

Кэш объектов из API. Watch API, кэширует в памяти, отправляет события. Быстрое чтение, меньше нагрузки на API.

**5. Work queue**

Очередь объектов для обработки. Rate limiting, retries, дедупликация, порядок. Защита от спама API.

**6. CRD**

Custom Resource Definition. Расширение API. OpenAPI schema, subresources (status, scale), printer columns, версионирование. `kubectl apply -f crd.yaml`.

**7. Operator pattern**

CRD + контроллер + доменные знания. Контроллер знает только про Pod'ы. Operator знает про replication, failover, backup.

**8. Capability levels**

Level 1: Basic Install. Level 2: Seamless Upgrades. Level 3: Full Lifecycle. Level 4: Deep Insights. Level 5: Auto Pilot.

**9. Owner references**

Ссылка на родительский объект. При удалении родителя → дочерние ресурсы удаляются (garbage collector).

**10. Finalizers**

Строка в metadata.finalizers, блокирует удаление. Контроллер видит deletionTimestamp, выполняет cleanup, удаляет finalizer, объект удаляется.

**11. controller-runtime**

Библиотека для контроллеров. Manager (управляет), Reconciler (интерфейс), Builder (DSL), Client (API). Используется в Kubebuilder.

**12. Kubebuilder**

Framework для операторов. Генерирует проект, CRD из Go-типов, RBAC, манифесты. `kubebuilder init`, `kubebuilder create api`, `make manifests`.

**13. Готовые операторы**

Prometheus Operator, Strimzi (Kafka), Zalando Postgres Operator, Cert Manager, External Secrets Operator, Crossplane.

**14. Отладка**

Логи оператора, events на instance, describe instance, status, дочерние ресурсы, metrics (`controller_runtime_*`). Local run (`make run`), envtest (`make test`).

**15. Когда писать свой**

Нет готового + специфичная логика + сложный lifecycle + много ручной работы + есть команда. Иначе — Helm, Ansible Operator, Helm Operator.

---

## Куда идти дальше?

Мы разобрали контроллеры и операторы. Теперь ты знаешь:

- Reconciliation loop.
- Controller pattern.
- CRD.
- Operator pattern.
- Informers и work queues.
- Ownership и finalizers.
- controller-runtime.
- Kubebuilder.
- Готовые операторы.
- Отладку.
- Когда писать свой.

**Kubernetes-блок завершён:**

- Глава 8: Архитектура.
- Глава 9: Планирование и ресурсы.
- Глава 10: Сети.
- Глава 11: Хранилища.
- Глава 12: Конфигурация и секреты.
- Глава 13: Обновления и деплой-стратегии.
- Глава 14: Контроллеры и операторы.

**Следующая — Глава 15: Helm — менеджер пакетов для Kubernetes.** Она уже написана, но требует переименования (была отправлена как Глава 13). Скажи «дальше» — и я отправлю её в правильной нумерации с обновлённым заголовком.