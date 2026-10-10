# 👑 Глава 24: Leader election — выбор лидера

**Что вы узнаете:**
- Что такое leader election и какую задачу он решает.
- Зачем нужен лидер в распределённой системе.
- Как построить leader election с нуля.
- Как использовать TTL и heartbeat.
- Как обрабатывать потерю лидера и split-brain.
- Как использовать `etcd` / `consul` для production.
- Как комбинировать leader election с worker pool, graceful shutdown.

**После прочтения вы сможете:**
- Построить leader election с нуля.
- Понимать, где leader election уместен, а где — нет.
- Использовать готовые решения (`etcd`, `consul`).
- Обрабатывать переход лидерства.
- Комбинировать leader election с другими паттернами.

---

## Содержание

- [24.0 Пролог: одна задача — один исполнитель](#240-пролог-одна-задача--один-исполнитель)
- [24.1 Что такое leader election](#241-что-такое-leader-election)
- [24.2 Leader election на канале](#242-leader-election-на-канале)
- [24.3 Leader election с TTL и heartbeat](#243-leader-election-с-ttl-и-heartbeat)
- [24.4 Leader election с context](#244-leader-election-с-context)
- [24.5 Split-brain: что это и как избежать](#245-split-brain-что-это-и-как-избежать)
- [24.6 etcd и consul: production-ready](#246-etcd-и-consul-production-ready)
- [24.7 В связке с другими паттернами](#247-в-связке-с-другими-паттернами)
- [24.8 Практика Go: leader election с метриками](#248-практика-go-leader-election-с-метриками)
- [24.9 Выводы и типичные ошибки](#249-выводы-и-типичные-ошибки)
- [24.10 Для быстрого повторения](#2410-для-быстрого-повторения)
- [24.11 Вопросы для самопроверки](#2411-вопросы-для-самопроверки)
- [24.12 Ответы](#2412-ответы)
- [24.13 Куда идти дальше?](#2413-куда-идти-дальше)
- [24.14 Чек-лист](#2414-чек-лист)

---

## 24.0 Пролог: одна задача — один исполнитель

У нас есть сервис, который запущен в **5 репликах** в Kubernetes. У него есть **фоновая задача**: раз в минуту синхронизировать данные с внешним API.

Пишем наивно:

```go
func main() {
    go syncLoop()  // ← в каждой реплике
    http.ListenAndServe(":8080", nil)
}

func syncLoop() {
    ticker := time.NewTicker(1 * time.Minute)
    defer ticker.Stop()
    for range ticker.C {
        syncWithExternalAPI()
    }
}
```

Работает. Но замечаем проблему: **5 реплик** делают синхронизацию **одновременно**. Внешний API получает **5 запросов** вместо 1. Мы тратим ресурсы и получаем **дубликаты** данных.

Хочется: только **одна** реплика выполняет фоновую задачу. Остальные — ждут. Если лидер упал — другая реплика становится лидером.

Это и есть **leader election** — выбор одного лидера среди нескольких участников.

> **Мост к следующим главам:** leader election — важный паттерн для распределённых систем. Он часто используется вместе с graceful shutdown (Глава 11), worker pool (Глава 13) и circuit breaker (Глава 15). Понимание leader election даёт понимание, **как делать периодические задачи в распределённой системе**.

---

## 24.1 Что такое leader election

**Leader election** — паттерн, при котором **один участник** становится лидером, а остальные — **follower'ами**.

### Идея

В распределённой системе **несколько реплик** одного сервиса. Некоторые задачи должны выполняться **только одним** экземпляром:

- **Фоновые задачи** (синхронизация, чистка).
- **Периодические задачи** (отчёты, метрики).
- **Эксклюзивные операции** (миграции, удаление).

**Leader election** выбирает одного лидера, который выполняет эти задачи. Остальные ждут.

### Схема

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Реплика 1│   │ Реплика 2│   │ Реплика 3│   │ Реплика 4│
│ (лидер)  │   │ (follower│   │ (follower│   │ (follower│
│          │   │  )       │   │  )       │   │  )       │
└────┬─────┘   └──────────┘   └──────────┘   └──────────┘
     │
     │ Выполняет фоновые задачи
     │
     ▼
┌──────────────────┐
│ Внешний API      │
└──────────────────┘
```

### Когда использовать leader election

**1. Фоновые задачи в Kubernetes.**

- 5 реплик, но задача — только у одной.
- Синхронизация, чистка.

**2. Периодические задачи.**

- Отчёты раз в час.
- Метрики раз в минуту.

**3. Эксклюзивные операции.**

- Миграции БД.
- Удаление старых данных.

**4. Кластерные системы.**

- Kafka, Zookeeper, etcd — используют leader election внутри.

### Когда НЕ использовать leader election

**1. Stateless-сервисы.**

Если задача **идемпотентна** — можно делать в каждой реплике.

**2. Быстрые задачи.**

Для задач < 1 сек — overhead больше пользы.

**3. Нет распределённости.**

Если сервис в одной реплике — leader election не нужен.

### Компоненты leader election

**1. Общее хранилище.**

- etcd.
- Consul.
- Redis.
- Zookeeper.

**2. Лидер.**

- Захватывает ресурс (lock).
- Выполняет задачи.

**3. TTL.**

- Лидер периодически обновляет TTL.
- Если не обновляет — лидерство теряется.

**4. Heartbeat.**

- Лидер шлёт сигнал «я жив».
- Если heartbeat не приходит — follower'ы выбирают нового лидера.

### 💡 Практика: как думать о leader election

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Leader election — для фоновых задач.**
2. **Leader election — для эксклюзивных операций.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **TTL 10–30 секунд.**
4. **Heartbeat каждые 3–5 секунд.**

**❌ НЕ ДЕЛАЙ:**

5. **Не используй leader election для stateless.**
6. **Не забывай про split-brain.**

---

## 24.2 Leader election на канале

Начнём с **простейшей** — in-process leader election. Полезно для понимания механики.

### Идея

- Есть **канал** (роль lock).
- Первый, кто записал в канал, становится лидером.
- Остальные ждут.

### Реализация

```go
type LeaderElection struct {
    leaderCh chan struct{}
    isLeader atomic.Bool
}

func NewLeaderElection() *LeaderElection {
    return &LeaderElection{
        leaderCh: make(chan struct{}, 1),
    }
}

func (le *LeaderElection) TryBecomeLeader() bool {
    select {
    case le.leaderCh <- struct{}{}:
        le.isLeader.Store(true)
        return true
    default:
        return false
    }
}

func (le *LeaderElection) IsLeader() bool {
    return le.isLeader.Load()
}

func (le *LeaderElection) Resign() {
    if le.isLeader.CompareAndSwap(true, false) {
        <-le.leaderCh
    }
}
```

**Что происходит:**

- `TryBecomeLeader` — пытаемся записать в канал.
- Если канал пуст — становимся лидером.
- Если занят — нет.

### Потребитель

```go
func main() {
    le := NewLeaderElection()
    
    // 5 реплик
    for i := 0; i < 5; i++ {
        i := i
        go func() {
            for {
                if le.TryBecomeLeader() {
                    fmt.Printf("Replica %d: became leader\n", i)
                    
                    // Выполняем фоновые задачи
                    for le.IsLeader() {
                        time.Sleep(1 * time.Second)
                        fmt.Printf("Replica %d: doing work\n", i)
                    }
                }
                time.Sleep(100 * time.Millisecond)
            }
        }()
    }
    
    time.Sleep(5 * time.Second)
}
```

**Что происходит:** одна из 5 реплик становится лидером. Остальные **пытаются** каждые 100 мс.

### Проблема: in-process

**Что если реплики — разные процессы?**

In-process election работает **только внутри одного процесса**. Для распределённой системы нужно **общее хранилище**.

### Проблема: нет TTL

**Что если лидер упал?**

`isLeader` останется `true`, но задачи не выполняются. Нужен **TTL**.

### Проблема: нет heartbeat

**Что если лидер завис?**

Без heartbeat follower'ы не знают, что лидер мёртв.

### Схема

```
Replica 1: TryBecomeLeader() ──► успех (лидер)
Replica 2: TryBecomeLeader() ──► fail
Replica 3: TryBecomeLeader() ──► fail
Replica 4: TryBecomeLeader() ──► fail
Replica 5: TryBecomeLeader() ──► fail

Replica 1: выполняет задачи
Replica 2-5: ждут

Replica 1: Resign()
Replica 2: TryBecomeLeader() ──► успех (новый лидер)
```

### 💡 Практика: как писать in-process election

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Канал как lock.**
2. **`atomic.Bool` для состояния.**
3. **`Resign()` для отказа от лидерства.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Для распределённых систем — etcd/consul.**

**❌ НЕ ДЕЛАЙ:**

5. **Не используй in-process для распределённых систем.**
6. **Не забывай про TTL.**

---

## 24.3 Leader election с TTL и heartbeat

**TTL** и **heartbeat** — основа настоящего leader election.

### Идея

- Лидер **захватывает** ресурс с **TTL** (например, 10 секунд).
- Лидер периодически **обновляет** TTL (heartbeat).
- Если TTL истёк — follower'ы могут захватить.

### Реализация на общем хранилище (упрощённо)

```go
type Store interface {
    // SetNX — set if not exists, с TTL.
    SetNX(ctx context.Context, key, value string, ttl time.Duration) (bool, error)
    // Refresh — обновить TTL, если значение совпадает.
    Refresh(ctx context.Context, key, value string, ttl time.Duration) (bool, error)
    // Delete — удалить, если значение совпадает.
    Delete(ctx context.Context, key, value string) error
}

type LeaderElection struct {
    store    Store
    key      string
    id       string
    ttl      time.Duration
    isLeader atomic.Bool
}

func NewLeaderElection(store Store, key, id string, ttl time.Duration) *LeaderElection {
    return &LeaderElection{
        store: store,
        key:   key,
        id:    id,
        ttl:   ttl,
    }
}

func (le *LeaderElection) Run(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }
        
        if le.isLeader.Load() {
            // Heartbeat
            ok, err := le.store.Refresh(ctx, le.key, le.id, le.ttl)
            if err != nil || !ok {
                le.isLeader.Store(false)
                continue
            }
            time.Sleep(le.ttl / 3)
        } else {
            // Попытка стать лидером
            ok, err := le.store.SetNX(ctx, le.key, le.id, le.ttl)
            if err == nil && ok {
                le.isLeader.Store(true)
                fmt.Printf("Became leader: %s\n", le.id)
            }
            time.Sleep(le.ttl / 3)
        }
    }
}

func (le *LeaderElection) IsLeader() bool {
    return le.isLeader.Load()
}
```

**Что происходит:**

- Если **лидер** — периодически обновляем TTL.
- Если **follower** — пытаемся захватить.
- TTL истечёт через `ttl`, если лидер не обновит.

### TTL vs heartbeat

**TTL** — время жизни блокировки.

**Heartbeat** — интервал обновления TTL.

**Рекомендации:**

- **TTL = 10–30 сек.**
- **Heartbeat = TTL / 3.**

**Почему heartbeat меньше TTL:** чтобы при кратковременных сбоях не терять лидерство.

### Пример: TTL = 10 сек

```
t=0:     Лидер захватил TTL=10
t=3.3:   Heartbeat (TTL обновлён до 13.3)
t=6.6:   Heartbeat (TTL до 16.6)
t=10:    Heartbeat (TTL до 20)
...

Если лидер упал на t=7:
  TTL истёк на t=16.6
  Follower'ы могут захватить с t=16.6
```

### Проблема: сетевые задержки

**Что если heartbeat задерживается?**

- Лидер думает, что он лидер.
- TTL истекает.
- Follower захватывает.
- **Два лидера** — split-brain.

**Решение:** fencing tokens (см. 24.5).

### Схема

```
Лидер:
  t=0:   SetNX(key, id, ttl=10) → успех
  t=3.3: Refresh(key, id, ttl=10) → успех
  t=6.6: Refresh → успех
  ...

Follower:
  t=0:   SetNX → fail
  t=3.3: SetNX → fail
  ...
  t=16.6: SetNX → успех (лидер упал)
```

### 💡 Практика: как настроить TTL и heartbeat

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **TTL 10–30 сек.**
2. **Heartbeat = TTL / 3.**
3. **Проверяй ошибки обновления.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Логируй переходы.**

**❌ НЕ ДЕЛАЙ:**

5. **Не ставь TTL 1 сек.** Сеть дрогнет — потеряешь лидерство.
6. **Не ставь heartbeat = TTL.** Не успеешь обновить.

---

## 24.4 Leader election с context

`context` — критичен для leader election. Без него лидер не сможет **корректно завершиться**.

### Проблема

```go
for {
    // Бесконечный цикл
}
```

**Что происходит:** при shutdown лидер не завершится.

### Решение: context

```go
func (le *LeaderElection) Run(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            if le.isLeader.Load() {
                le.store.Delete(context.Background(), le.key, le.id)
                le.isLeader.Store(false)
            }
            return ctx.Err()
        default:
        }
        
        // ...
    }
}
```

**Что происходит:**

- При отмене `ctx` — освобождаем лидерство.
- Возвращаем ошибку.

### Полная реализация

```go
func (le *LeaderElection) Run(ctx context.Context) error {
    ticker := time.NewTicker(le.ttl / 3)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            if le.isLeader.Load() {
                // Освобождаем лидерство
                le.store.Delete(context.Background(), le.key, le.id)
                le.isLeader.Store(false)
                fmt.Printf("Resigned leadership: %s\n", le.id)
            }
            return ctx.Err()
        case <-ticker.C:
            if le.isLeader.Load() {
                // Heartbeat
                ok, err := le.store.Refresh(ctx, le.key, le.id, le.ttl)
                if err != nil || !ok {
                    le.isLeader.Store(false)
                    continue
                }
            } else {
                // Попытка стать лидером
                ok, err := le.store.SetNX(ctx, le.key, le.id, le.ttl)
                if err == nil && ok {
                    le.isLeader.Store(true)
                    fmt.Printf("Became leader: %s\n", le.id)
                }
            }
        }
    }
}
```

**Что происходит:**

- `ticker` — heartbeat/попытка каждые TTL/3.
- При `ctx.Done()` — освобождаем лидерство.

### Потребитель

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
    defer stop()
    
    le := NewLeaderElection(store, "leader", "replica-1", 10*time.Second)
    
    go func() {
        if err := le.Run(ctx); err != nil && !errors.Is(err, context.Canceled) {
            log.Fatal(err)
        }
    }()
    
    // Фоновая работа
    for {
        select {
        case <-ctx.Done():
            return
        default:
        }
        
        if le.IsLeader() {
            doWork()
        } else {
            time.Sleep(1 * time.Second)
        }
    }
}
```

**Что происходит:** при SIGTERM лидер **освобождает** лидерство и корректно завершается.

### Схема

```
Run(ctx):
  ticker := time.NewTicker(ttl/3)
  
  for {
    select {
    case <-ctx.Done():                    ← отмена
      if isLeader:
        store.Delete(key, id)             ← освободить
      return
    case <-ticker.C:
      if isLeader:
        store.Refresh(key, id, ttl)       ← heartbeat
      else:
        store.SetNX(key, id, ttl)         ← попытка
    }
  }
```

### 💡 Практика: как добавить context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx` — первый аргумент.**
2. **`select` с `ctx.Done()`.**
3. **Освобождение лидерства при отмене.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Логирование переходов.**

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай про graceful shutdown.**

---

## 24.5 Split-brain: что это и как избежать

**Split-brain** — ситуация, когда **два лидера** одновременно.

### Проблема

```
t=0:   Лидер 1 захватил TTL=10
t=3:   Лидер 1 обновляет TTL → t=13
t=6:   Сеть дрогнула — heartbeat не дошёл
t=10:  TTL истёк
t=10:  Лидер 2 захватывает TTL=10 → t=20
t=11:  Лидер 1 снова онлайн, думает что он лидер
       → Два лидера!
```

**Что происходит:** оба лидера выполняют задачи. **Дублирование.**

### Причины split-brain

**1. Сетевые задержки.**

Heartbeat не дошёл — TTL истёк — второй лидер захватил.

**2. Пауза в работе лидера.**

GC пауза, CPU contention — heartbeat задержался.

**3. Часы.**

Разное время между узлами.

### Как избежать split-brain

**1. Fencing tokens.**

Каждое действие лидера помечается **монотонно возрастающим числом** (fencing token). Ресурс (БД) **проверяет** токен. Если токен **меньше** последнего — **отклоняет**.

**Пример:**

```
t=0:   Лидер 1 захватил, получил token=1
t=10:  Лидер 2 захватил, получил token=2
t=11:  Лидер 1 пишет в БД с token=1 → БД отклоняет (последний token=2)
```

**2. Quorum.**

Нужно **большинство** голосов для лидерства. Если сеть разделилась на две части — лидер только в **одной** (где больше узлов).

**Пример:** 5 узлов. Quorum = 3. Сеть разделилась на 2 + 3. Лидер только в группе из 3.

**3. Lease.**

Лидер работает **только** пока TTL активен. Если TTL истёк — лидер **немедленно** останавливается.

**Пример:**

```go
if !le.IsLeader() {
    return ErrLostLeadership
}
doWork()
```

**4. Monotonic clock.**

Использовать **монотонные** часы для измерения TTL.

### Пример: fencing tokens

```go
type Store interface {
    SetNX(ctx context.Context, key, value string, ttl time.Duration) (int64, bool, error)
    // Возвращает token (монотонно возрастающий номер).
}

func (le *LeaderElection) Run(ctx context.Context) error {
    var fencingToken int64
    ticker := time.NewTicker(le.ttl / 3)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
            if le.isLeader.Load() {
                // Heartbeat — возвращает новый token
                token, ok, err := le.store.Refresh(ctx, le.key, le.id, le.ttl)
                if err != nil || !ok {
                    le.isLeader.Store(false)
                    continue
                }
                fencingToken = token
            } else {
                token, ok, err := le.store.SetNX(ctx, le.key, le.id, le.ttl)
                if err == nil && ok {
                    le.isLeader.Store(true)
                    fencingToken = token
                }
            }
        }
    }
}

func (le *LeaderElection) Token() int64 {
    return le.fencingToken
}
```

**Что даёт:** каждое действие лидера помечено token. БД проверяет.

### Схема

```
Лидер 1: SetNX → token=1
Лидер 1: действия с token=1 → БД OK
Лидер 2: SetNX → token=2
Лидер 1: действия с token=1 → БД отклоняет (уже видел token=2)
```

### 💡 Практика: как избежать split-brain

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Fencing tokens** для критичных операций.
2. **Quorum** — если возможно.
3. **Lease** — лидер работает только пока TTL активен.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Мониторь переходы лидерства.**
5. **Логируй split-brain.**

**❌ НЕ ДЕЛАЙ:**

6. **Не работай без fencing, если операция критична.**
7. **Не полагайся только на TTL.**

---

## 24.6 etcd и consul: production-ready

Для production — используй **etcd** или **consul**.

### etcd

**etcd** — распределённое key-value хранилище. Использует **Raft** для консенсуса. Поддерживает **lease** и **watch**.

**Библиотека:** `go.etcd.io/etcd/client/v3/concurrency`.

```go
import (
    clientv3 "go.etcd.io/etcd/client/v3"
    "go.etcd.io/etcd/client/v3/concurrency"
)

func main() {
    client, _ := clientv3.New(clientv3.Config{
        Endpoints:   []string{"localhost:2379"},
        DialTimeout: 5 * time.Second,
    })
    defer client.Close()
    
    session, _ := concurrency.NewSession(client, concurrency.WithTTL(10))
    defer session.Close()
    
    election := concurrency.NewElection(session, "/my-election/")
    
    ctx := context.Background()
    
    // Участвовать в выборах
    go func() {
        if err := election.Campaign(ctx, "replica-1"); err != nil {
            log.Fatal(err)
        }
        fmt.Println("Became leader")
    }()
    
    // Ждать лидерства
    for {
        resp, err := election.Leader(ctx)
        if err != nil {
            time.Sleep(100 * time.Millisecond)
            continue
        }
        fmt.Printf("Current leader: %s\n", string(resp.Kvs[0].Value))
        time.Sleep(1 * time.Second)
    }
}
```

**Что даёт:**

- **Session** — TTL и heartbeat автоматически.
- **Campaign** — стать лидером.
- **Observe** — наблюдать за лидером.
- **Resign** — отказаться.

### consul

**Consul** — сервис-меш с leader election. Использует **Raft**.

**Библиотека:** `github.com/hashicorp/consul/api`.

```go
import "github.com/hashicorp/consul/api"

func main() {
    client, _ := api.NewClient(api.DefaultConfig())
    
    key := "service/my-service/leader"
    
    // Попытка захватить lock
    opts := &api.LockOptions{
        Key:         key,
        Value:       []byte("replica-1"),
        SessionTTL:  10 * time.Second,
        LockDelay:   100 * time.Millisecond,
    }
    
    lock, _ := client.LockOpts(opts)
    
    stopCh := make(chan struct{})
    leaderCh, _ := lock.Lock(stopCh)
    
    go func() {
        for {
            select {
            case <-leaderCh:
                fmt.Println("Became leader")
                doWork()
            case <-stopCh:
                return
            }
        }
    }()
    
    time.Sleep(10 * time.Second)
    close(stopCh)
}
```

**Что даёт:** consul автоматически управляет сессией, TTL, leader election.

### Сравнение

| Аспект | etcd | consul |
|:---|:---|:---|
| Консенсус | Raft | Raft |
| Leader election | ✅ | ✅ |
| Watch | ✅ | ✅ |
| KV | ✅ | ✅ |
| Service discovery | ❌ | ✅ |
| Сложность | Простой | Более функциональный |

### Когда использовать

**etcd:**

- **Kubernetes** (использует etcd внутри).
- **Простой KV + election.**
- **Минимум зависимостей.**

**consul:**

- **Service mesh**.
- **Больше функций** (discovery, health checks).
- **Мультидатацентр.**

### 💡 Практика: как использовать etcd/consul

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **etcd / consul** для production.
2. **Session TTL = 10–30 сек.**
3. **Graceful shutdown** — `Close()` session.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Мониторинг лидера** — `Observe`.

**❌ НЕ ДЕЛАЙ:**

5. **Не пиши свой leader election** для production.
6. **Не забывай про `Close()`.**

---

## 24.7 В связке с другими паттернами

Leader election редко используется **в одиночку**. Разберём связки.

### Leader election + worker pool

**Лидер запускает worker pool для фоновых задач:**

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM)
    defer stop()
    
    le := NewLeaderElection(store, "leader", "replica-1", 10*time.Second)
    go le.Run(ctx)
    
    var wg sync.WaitGroup
    for {
        select {
        case <-ctx.Done():
            wg.Wait()
            return
        default:
        }
        
        if le.IsLeader() {
            // Запускаем worker pool
            wg.Add(1)
            go func() {
                defer wg.Done()
                runWorkerPool(ctx)
            }()
            
            // Ждём, пока лидерство не потеряно
            for le.IsLeader() {
                time.Sleep(100 * time.Millisecond)
            }
        } else {
            time.Sleep(1 * time.Second)
        }
    }
}
```

### Leader election + graceful shutdown

**Лидер освобождает лидерство при shutdown:**

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM)
    defer stop()
    
    le := NewLeaderElection(store, "leader", "replica-1", 10*time.Second)
    go le.Run(ctx)
    
    <-ctx.Done()
    
    // Освобождаем лидерство
    if le.IsLeader() {
        le.Resign()
    }
    
    // Graceful shutdown остальных компонентов
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
    defer cancel()
    
    srv.Shutdown(shutdownCtx)
}
```

### Leader election + circuit breaker

**Лидер использует circuit breaker для внешних вызовов:**

```go
func (l *Leader) doWork(ctx context.Context) {
    err := l.breaker.Call(ctx, func(ctx context.Context) error {
        return syncWithExternalAPI(ctx)
    })
    if err != nil {
        log.Printf("sync failed: %v", err)
    }
}
```

### Leader election + metrics

**Метрики по лидерству:**

```go
var (
    isLeaderGauge atomic.Int64
    leaderChanges atomic.Int64
)

func (le *LeaderElection) onBecameLeader() {
    isLeaderGauge.Store(1)
    leaderChanges.Add(1)
}

func (le *LeaderElection) onLostLeadership() {
    isLeaderGauge.Store(0)
}
```

### Схема

```
Сервис (5 реплик):
  │
  ├─ Leader election
  │   └─ Одна реплика — лидер
  │
  ├─ HTTP-сервер (все)
  │
  ├─ Фоновые задачи (только лидер)
  │   └─ Worker pool
  │
  └─ Graceful shutdown
      └─ Освободить лидерство
```

### 💡 Практика: как комбинировать leader election

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Leader election + worker pool** — для фоновых задач.
2. **Leader election + graceful shutdown** — освободить лидерство.
3. **Метрики лидерства.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Leader election + circuit breaker** — для внешних вызовов.

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай про graceful shutdown.**

---

## 24.8 Практика Go: leader election с метриками

Разберём **leader election с метриками** на etcd.

### Полный код

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "log"
    "os/signal"
    "sync/atomic"
    "syscall"
    "time"
    
    clientv3 "go.etcd.io/etcd/client/v3"
    "go.etcd.io/etcd/client/v3/concurrency"
)

type Metrics struct {
    IsLeader       atomic.Int64
    BecameLeader   atomic.Int64
    LostLeadership atomic.Int64
    WorkCycles     atomic.Int64
}

func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
    defer stop()
    
    client, err := clientv3.New(clientv3.Config{
        Endpoints:   []string{"localhost:2379"},
        DialTimeout: 5 * time.Second,
    })
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()
    
    session, err := concurrency.NewSession(client, concurrency.WithTTL(10))
    if err != nil {
        log.Fatal(err)
    }
    defer session.Close()
    
    election := concurrency.NewElection(session, "/my-service/leader")
    
    metrics := &Metrics{}
    
    go runLeaderElection(ctx, election, metrics)
    go runWork(ctx, metrics)
    
    <-ctx.Done()
    fmt.Println("Shutting down...")
    
    // Освобождаем лидерство
    if metrics.IsLeader.Load() == 1 {
        resignCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()
        election.Resign(resignCtx)
    }
    
    printMetrics(metrics)
}

func runLeaderElection(ctx context.Context, election *concurrency.Election, metrics *Metrics) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
        }
        
        if metrics.IsLeader.Load() == 0 {
            // Пытаемся стать лидером
            campaignCtx, cancel := context.WithCancel(ctx)
            
            go func() {
                if err := election.Campaign(campaignCtx, "replica-1"); err != nil {
                    if !errors.Is(err, context.Canceled) {
                        log.Printf("Campaign error: %v", err)
                    }
                    return
                }
                
                metrics.IsLeader.Store(1)
                metrics.BecameLeader.Add(1)
                fmt.Println("Became leader")
                
                // Heartbeat через keep-alive
                <-campaignCtx.Done()
                metrics.IsLeader.Store(0)
                metrics.LostLeadership.Add(1)
                fmt.Println("Lost leadership")
            }()
            
            // Ждём, пока станем лидером или ctx отменён
            for metrics.IsLeader.Load() == 0 {
                select {
                case <-ctx.Done():
                    cancel()
                    return
                case <-time.After(100 * time.Millisecond):
                }
            }
        }
        
        select {
        case <-ctx.Done():
            return
        case <-time.After(100 * time.Millisecond):
        }
    }
}

func runWork(ctx context.Context, metrics *Metrics) {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            if metrics.IsLeader.Load() == 1 {
                metrics.WorkCycles.Add(1)
                fmt.Printf("Doing work (cycle %d)\n", metrics.WorkCycles.Load())
            }
        }
    }
}

func printMetrics(metrics *Metrics) {
    fmt.Println("\n=== Metrics ===")
    fmt.Printf("Is leader:       %d\n", metrics.IsLeader.Load())
    fmt.Printf("Became leader:   %d\n", metrics.BecameLeader.Load())
    fmt.Printf("Lost leadership: %d\n", metrics.LostLeadership.Load())
    fmt.Printf("Work cycles:     %d\n", metrics.WorkCycles.Load())
}
```

**Пример вывода:**

```
Became leader
Doing work (cycle 1)
Doing work (cycle 2)
Doing work (cycle 3)
Shutting down...

=== Metrics ===
Is leader:       0
Became leader:   1
Lost leadership: 0
Work cycles:     3
```

**Что демонстрирует:**

- **`Campaign`** — стал лидером.
- **`Work cycles`** — выполнял работу.
- **`Resign`** — освободил лидерство при shutdown.

### Пример: consul

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"
    
    "github.com/hashicorp/consul/api"
)

type Metrics struct {
    IsLeader       atomic.Int64
    BecameLeader   atomic.Int64
    LostLeadership atomic.Int64
}

func main() {
    client, err := api.NewClient(api.DefaultConfig())
    if err != nil {
        log.Fatal(err)
    }
    
    opts := &api.LockOptions{
        Key:         "service/my-service/leader",
        Value:       []byte("replica-1"),
        SessionTTL:  10 * time.Second,
        LockDelay:   100 * time.Millisecond,
    }
    
    lock, err := client.LockOpts(opts)
    if err != nil {
        log.Fatal(err)
    }
    
    metrics := &Metrics{}
    
    stopCh := make(chan struct{})
    leaderCh, err := lock.Lock(stopCh)
    if err != nil {
        log.Fatal(err)
    }
    
    go func() {
        for {
            select {
            case <-leaderCh:
                metrics.IsLeader.Store(1)
                metrics.BecameLeader.Add(1)
                fmt.Println("Became leader")
                runWork(metrics)
            case <-stopCh:
                return
            }
        }
    }()
    
    time.Sleep(10 * time.Second)
    close(stopCh)
    metrics.IsLeader.Store(0)
    metrics.LostLeadership.Add(1)
    
    fmt.Println("\n=== Metrics ===")
    fmt.Printf("Became leader:   %d\n", metrics.BecameLeader.Load())
    fmt.Printf("Lost leadership: %d\n", metrics.LostLeadership.Load())
}

func runWork(metrics *Metrics) {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()
    
    for metrics.IsLeader.Load() == 1 {
        select {
        case <-ticker.C:
            fmt.Println("Doing work")
        }
    }
}
```

### 💡 Практика: как измерять leader election

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики:** is_leader, became_leader, lost_leadership.
2. **`atomic.Int64`** для счётчиков.
3. **Экспорт в Prometheus.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Алерт при частых сменах лидера.**
5. **Логирование переходов.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `Mutex` для метрик.**
7. **Не игнорируй потерю лидерства.**

---

## 24.9 Выводы и типичные ошибки

**Что мы узнали?**

Leader election — паттерн выбора одного лидера среди нескольких реплик. **In-process** — на канале. **С TTL и heartbeat** — через общее хранилище. **`context`** — для отмены и освобождения лидерства. **Split-brain** — два лидера одновременно; избегаем через fencing tokens, quorum, lease. **`etcd` и `consul`** — production-ready. Комбинируется с worker pool, graceful shutdown, circuit breaker.

**Типичные ошибки:**

- ❌ **In-process для распределённых систем.** Не работает.
- ❌ **TTL = 1 сек.** Сеть дрогнет — потеряешь лидерство.
- ❌ **Heartbeat = TTL.** Не успеешь обновить.
- ❌ **Нет fencing tokens** для критичных операций. Split-brain.
- ❌ **Не освобождать лидерство при shutdown.** Другой ждёт TTL.
- ❌ **Забыть `Close()` session.** Утечка.
- ❌ **Писать свой leader election** для production.
- ❌ **Не мониторить переходы лидерства.**
- ❌ **Не логировать split-brain.**
- ❌ **Использовать leader election для stateless.**

---

## 24.10 Для быстрого повторения

- **Leader election** — выбор одного лидера среди реплик.
- **In-process** — на канале.
- **TTL 10–30 сек.**
- **Heartbeat = TTL / 3.**
- **`context`** — для отмены.
- **Освобождать лидерство** при shutdown.
- **Split-brain** — два лидера.
- **Fencing tokens** — для критичных операций.
- **Quorum** — большинство голосов.
- **Lease** — работает пока TTL активен.
- **`etcd` и `consul`** — production-ready.
- **`etcd/concurrency`** — `Session`, `Election`, `Campaign`.
- **`consul/api`** — `Lock`, `LockOpts`.
- **Leader election + worker pool** — фоновые задачи.
- **Leader election + graceful shutdown** — освободить лидерство.
- **Метрики:** is_leader, became_leader, lost_leadership.

---

## 24.11 Вопросы для самопроверки

1. Что такое leader election? Какую задачу решает?
2. Что такое TTL и heartbeat?
3. Зачем `context` в leader election?
4. Что такое split-brain? Как избежать?
5. Что такое fencing tokens?
6. Как использовать etcd для leader election?
7. Как комбинировать leader election с worker pool?
8. Почему нельзя in-process для распределённых систем?

---

## 24.12 Ответы

### Ответ 1

**Leader election** — паттерн выбора одного лидера среди нескольких реплик. Решает задачу: **фоновая задача должна выполняться только одним экземпляром**.

**Пример:** синхронизация с внешним API в 5 репликах — только одна делает.

### Ответ 2

**TTL** — время жизни блокировки. **Heartbeat** — интервал обновления TTL.

**TTL 10–30 сек.** **Heartbeat = TTL / 3.** Heartbeat меньше TTL — чтобы при кратковременных сбоях не терять лидерство.

### Ответ 3

**`context`** позволяет корректно завершиться. При отмене — **освобождаем лидерство** через `Delete`. Другой follower **сразу** может захватить, не ждать TTL.

### Ответ 4

**Split-brain** — два лидера одновременно.

**Причины:** сетевые задержки, пауза в работе, разные часы.

**Избежать:**
- **Fencing tokens.**
- **Quorum.**
- **Lease.**
- **Monotonic clock.**

### Ответ 5

**Fencing tokens** — монотонно возрастающие номера для действий лидера. Ресурс (БД) **проверяет** токен. Если токен **меньше** последнего — **отклоняет**.

**Пример:**

```
Лидер 1: token=1 → БД OK
Лидер 2: token=2 → БД OK
Лидер 1: token=1 → БД отклоняет
```

### Ответ 6

```go
client, _ := clientv3.New(clientv3.Config{
    Endpoints: []string{"localhost:2379"},
})
session, _ := concurrency.NewSession(client, concurrency.WithTTL(10))
election := concurrency.NewElection(session, "/my-service/leader")

// Campaign — стать лидером
election.Campaign(ctx, "replica-1")

// Resign — отказаться
election.Resign(ctx)
```

etcd автоматически управляет TTL и heartbeat через session.

### Ответ 7

**Leader election + worker pool:**

```go
if le.IsLeader() {
    runWorkerPool(ctx)
}
```

Лидер запускает worker pool. При потере лидерства — worker pool останавливается.

### Ответ 8

**In-process** работает только **внутри одного процесса**. Для распределённой системы — **разные процессы** — нужно **общее хранилище** (etcd, consul, Redis). Иначе каждый процесс будет думать, что он лидер.

---

## 24.13 Куда идти дальше?

Мы разобрали leader election — выбор лидера. Теперь мы умеем делать фоновые задачи в распределённой системе.

Но остаются **продвинутые темы**: sharded locks, lock-free структуры, sync.Pool, GC.

- **Как разбить лок на шарды?** → **Глава 25: Sharded locks.**
- **Как построить lock-free структуры?** → **Глава 26: Lock-free структуры.**
- **Как использовать `sync.Pool`?** → **Глава 27: sync.Pool и аллокатор.**

---

## 24.14 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Leader election** | Выбор лидера | Одна реплика — лидер |
| **In-process** | На канале | Только внутри процесса |
| **TTL** | 10–30 сек | Время жизни блокировки |
| **Heartbeat** | TTL / 3 | Обновление TTL |
| **`context`** | Отмена | Освобождение лидерства |
| **Split-brain** | Два лидера | Fencing, quorum, lease |
| **Fencing tokens** | Монотонные номера | БД проверяет |
| **etcd** | Хранилище | `concurrency.Election` |
| **consul** | Хранилище | `api.Lock` |
| **+ worker pool** | Фоновые задачи | — |
| **+ graceful shutdown** | Освободить лидерство | — |
| **Метрики** | is_leader, became_leader | `atomic.Int64` |

👑 **Ключевая идея:** Leader election — паттерн выбора одного лидера среди нескольких реплик. **In-process** — на канале (только внутри процесса). **С TTL и heartbeat** — через общее хранилище (TTL 10–30 сек, heartbeat TTL/3). **`context`** — для отмены и освобождения лидерства. **Split-brain** — два лидера; избегаем через fencing tokens, quorum, lease. **`etcd` и `consul`** — production-ready. **Освобождать лидерство при shutdown** — обязательно. Комбинируется с worker pool, graceful shutdown, circuit breaker. Метрики: is_leader, became_leader, lost_leadership.