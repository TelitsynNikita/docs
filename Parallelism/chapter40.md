# 🏭 Глава 40: Проект — конкурентный брокер сообщений на Go

**Что вы узнаете:**
- Как применить **все** изученные паттерны в одном проекте.
- Как спроектировать **broker** с топиками, партициями, consumer groups.
- Как использовать **worker pool**, **pipeline**, **backpressure** для обработки сообщений.
- Как реализовать **replication** с **leader election** и **fencing tokens**.
- Как защититься от **DoS**, **deadlock**, **split-brain**.
- Как организовать **graceful shutdown** всего кластера.
- Как тестировать **такой** проект: race detector, stress, integration.

**После прочтения вы сможете:**
- Спроектировать конкурентную систему с нуля.
- Применить паттерны в правильных местах.
- Понять **trade-off** между сложностью и надёжностью.
- Написать production-готовый broker (упрощённый).

---

## Содержание

- [40.0 Пролог: свой Kafka на Go](#400-пролог-свой-kafka-на-go)
- [40.1 Архитектура брокера](#401-архитектура-брокера)
- [40.2 Хранилище: топики, партиции, сегменты](#402-хранилище-топики-партиции-сегменты)
- [40.3 Producer: приём сообщений](#403-producer-приём-сообщений)
- [40.4 Consumer: чтение сообщений](#404-consumer-чтение-сообщений)
- [40.5 Consumer groups: координация](#405-consumer-groups-координация)
- [40.6 Репликация: leader election и fencing tokens](#406-репликация-leader-election-и-fencing-tokens)
- [40.7 Метаданные: registry и discovery](#407-метаданные-registry-и-discovery)
- [40.8 Защита от DoS и deadlock](#408-защита-от-dos-и-deadlock)
- [40.9 Graceful shutdown кластера](#409-graceful-shutdown-кластера)
- [40.10 Тестирование брокера](#4010-тестирование-брокера)
- [40.11 Практика Go: полный код](#4011-практика-go-полный-код)
- [40.12 Выводы и типичные ошибки](#4012-выводы-и-типичные-ошибки)
- [40.13 Для быстрого повторения](#4013-для-быстрого-повторения)
- [40.14 Вопросы для самопроверки](#4014-вопросы-для-самопроверки)
- [40.15 Ответы](#4015-ответы)
- [40.16 Куда идти дальше?](#4016-куда-идти-дальше)
- [40.17 Чек-лист](#4017-чек-лист)

---

## 40.0 Пролог: свой Kafka на Go

Мы разобрали **все** паттерны: горутины, каналы, синхронизация, context, планировщик, memory model, generator, fan-in, fan-out, pipeline, semaphore, worker pool, rate limiter, circuit breaker, retry, timeout, debounce, throttle, tee, bridge, future, batch, bulkhead, приоритеты, leader election, обработка ошибок, graceful shutdown, backpressure, тестирование, sharded locks, lock-free, sync.Pool, GC, анти-паттерны, HTTP-сервер, очереди, ETL, scraper, WebSocket, безопасность.

Пришло время **собрать всё вместе**. Мы построим **упрощённый брокер сообщений** — как Kafka, но на Go.

**Зачем?**

- **Применить** паттерны в реальном проекте.
- **Понять**, где какие паттерны **нужны**.
- **Увидеть trade-off** между простотой и надёжностью.

**Что построим:**

- **Топики** с **партициями**.
- **Producer** — приём сообщений.
- **Consumer** — чтение сообщений.
- **Consumer groups** — координация.
- **Replication** — 3 реплики с leader election.
- **Graceful shutdown** всего кластера.

> **Мост к предыдущим главам:** этот проект **интегрирует** все паттерны, которые мы разбирали. Каждая секция ссылается на нужную главу.

---

## 40.1 Архитектура брокера

Начнём с **общей картины**.

### Компоненты

```
┌─────────────────────────────────────────────────────────┐
│                    Broker Cluster                        │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Broker 1    │  │  Broker 2    │  │  Broker 3    │  │
│  │  (leader)    │  │  (follower)  │  │  (follower)  │  │
│  │              │  │              │  │              │  │
│  │  Topics:     │  │  Topics:     │  │  Topics:     │  │
│  │  - orders    │  │  - orders    │  │  - orders    │  │
│  │  - events    │  │  - events    │  │  - events    │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                 │                 │           │
│         └─────────────────┴─────────────────┘           │
│                     Metadata (etcd)                     │
│                                                          │
└─────────────────────────────────────────────────────────┘
              ▲                              ▲
              │ Produce                      │ Consume
         ┌────┴────┐                    ┌────┴────┐
         │Producer │                    │Consumer │
         └─────────┘                    └─────────┘
```

### Ключевые сущности

**Topic** — логический поток сообщений. Например, `orders`.

**Partition** — часть топика. Сообщения распределяются по партициям. Например, `orders-0`, `orders-1`, `orders-2`.

**Segment** — файл на диске с сообщениями партиции. Например, `orders-0/000000.log`.

**Offset** — порядковый номер сообщения в партиции. Монотонно возрастающий.

**Consumer Group** — группа consumer'ов, которая читает топик **совместно**. Каждая партиция назначается **одному** consumer'у в группе.

**Leader/Follower** — для каждой партиции один брокер — **лидер**, остальные — **follower'ы**. Только лидер принимает записи.

### Паттерны в архитектуре

| Компонент | Паттерны |
|:---|:---|
| **HTTP-сервер** | Rate limiter, semaphore, timeout, graceful shutdown |
| **Producer** | Batch, backpressure, worker pool |
| **Consumer** | Fan-out, pipeline, backpressure |
| **Consumer groups** | Leader election, bulkhead |
| **Replication** | Leader election, fencing tokens, retry, circuit breaker |
| **Metadata** | Batch, retry, circuit breaker |
| **Storage** | Batch, sync.Pool, sharded locks |
| **Защита** | Rate limiter, semaphore, bulkhead, timeout |

### План проекта

1. **Storage** — топики, партиции, сегменты.
2. **Producer** — HTTP API для записи.
3. **Consumer** — HTTP API для чтения.
4. **Consumer groups** — координация.
5. **Replication** — leader/follower, fencing tokens.
6. **Metadata** — registry (etcd).
7. **Защита** — middleware.
8. **Graceful shutdown**.

### 💡 Практика: как думать об архитектуре

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Разделяй** компоненты по **ответственности**.
2. **Определяй** паттерны **до** кода.
3. **Думай** о **failure modes**.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Начинай** с простого — добавляй паттерны **по мере необходимости**.

**❌ НЕ ДЕЛАЙ:**

5. **Не пытайся** применить **все** паттерны сразу.

---

## 40.2 Хранилище: топики, партиции, сегменты

Начнём с **хранилища**.

### Структура на диске

```
data/
├── orders/
│   ├── 0/
│   │   ├── 000000.log
│   │   ├── 000001.log
│   │   └── index
│   ├── 1/
│   │   └── 000000.log
│   └── 2/
│       └── 000000.log
└── events/
    ├── 0/
    │   └── 000000.log
    └── 1/
        └── 000000.log
```

### Структура в памяти

```go
type Broker struct {
    topics   map[string]*Topic
    mu       sync.RWMutex   // ← Глава 4
}

type Topic struct {
    name       string
    partitions []*Partition
    mu         sync.RWMutex
}

type Partition struct {
    id       int
    leaderID string         // ← Глава 24
    segments []*Segment
    nextOffset atomic.Int64  // ← Глава 4
    
    // Backpressure (Глава 27)
    writeCh  chan *Message
    readCh   chan *Message
    
    // Sharded locks (Глава 29)
    shards   [16]sync.Mutex
}
```

### Запись сообщения

```go
type Message struct {
    Offset    int64
    Key       []byte
    Value     []byte
    Timestamp time.Time
    Headers   map[string][]byte
}

func (p *Partition) Append(ctx context.Context, msg *Message) (int64, error) {
    // 1. Выделить offset (atomic — Глава 4)
    offset := p.nextOffset.Add(1) - 1
    msg.Offset = offset
    
    // 2. Записать в WAL (batch — Глава 21)
    if err := p.segment.Append(ctx, msg); err != nil {
        return 0, err
    }
    
    // 3. Уведомить consumer'ов (fan-out — Глава 10)
    select {
    case p.notify <- struct{}{}:
    case <-ctx.Done():
        return 0, ctx.Err()
    case <-time.After(100 * time.Millisecond):
        // не блокируемся (reject — Глава 27)
    }
    
    return offset, nil
}
```

### Чтение сообщений

```go
func (p *Partition) Read(ctx context.Context, offset int64) (*Message, error) {
    // 1. Найти сегмент (sharded lock — Глава 29)
    shard := p.shards[offset%16]
    shard.Lock()
    defer shard.Unlock()
    
    segment := p.findSegment(offset)
    if segment == nil {
        return nil, ErrOffsetOutOfRange
    }
    
    // 2. Прочитать сообщение
    return segment.Read(ctx, offset)
}
```

### Паттерны в storage

| Паттерн | Где |
|:---|:---|
| **RWMutex** | Защита topics map |
| **atomic** | nextOffset |
| **Batch** | Запись в WAL |
| **Sharded locks** | 16 шардов на partition |
| **Backpressure** | writeCh, readCh |
| **Fan-out** | Уведомление consumer'ов |
| **Timeout** | Не блокироваться на notify |

### 💡 Практика: как писать storage

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **atomic** для offset.
2. **Batch** для записи.
3. **Sharded locks** для read/write.
4. **Timeout** для notify.

**👍 СТОИТ СДЕЛАТЬ:**

5. **RWMutex** для map.
6. **Backpressure** через каналы.

**❌ НЕ ДЕЛАЙ:**

7. **Не пиши по одному сообщению.**
8. **Не используй один mutex на всё.**

---

## 40.3 Producer: приём сообщений

**Producer** присылает сообщения через HTTP.

### HTTP API

```go
func (b *Broker) handleProduce(w http.ResponseWriter, r *http.Request) {
    // 1. Rate limiter (Глава 14)
    if !b.limiter.Allow() {
        http.Error(w, "too many requests", 429)
        return
    }
    
    // 2. Semaphore (Глава 12)
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
    case <-r.Context().Done():
        return
    default:
        http.Error(w, "server busy", 503)
        return
    }
    
    // 3. Timeout (Глава 17)
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()
    
    // 4. Parse
    var req ProduceRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "bad request", 400)
        return
    }
    
    // 5. Find topic/partition
    topic := b.topics[req.Topic]
    if topic == nil {
        http.Error(w, "topic not found", 404)
        return
    }
    partition := topic.partitions[req.Partition]
    
    // 6. Append
    offset, err := partition.Append(ctx, &req.Message)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    
    // 7. Response
    json.NewEncoder(w).Encode(ProduceResponse{Offset: offset})
}
```

### Batch-режим

**Для производительности** — приём **батчами** (Глава 21).

```go
func (b *Broker) handleProduceBatch(w http.ResponseWriter, r *http.Request) {
    // ... rate limiter, semaphore, timeout ...
    
    var req ProduceBatchRequest
    json.NewDecoder(r.Body).Decode(&req)
    
    topic := b.topics[req.Topic]
    partition := topic.partitions[req.Partition]
    
    // Batch append
    offsets, err := partition.AppendBatch(ctx, req.Messages)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    
    json.NewEncoder(w).Encode(ProduceBatchResponse{Offsets: offsets})
}
```

### Pipeline для обработки

**Приём сообщений** — это **pipeline** (Глава 11):

```
HTTP Request → Parse → Validate → Append → WAL → Response
```

```go
func (b *Broker) processMessage(ctx context.Context, raw []byte) (*Message, error) {
    msg, err := parse(raw)
    if err != nil {
        return nil, err
    }
    
    if err := validate(msg); err != nil {
        return nil, err
    }
    
    return msg, nil
}
```

### Worker pool для параллельной записи

**Для высокой нагрузки** — worker pool (Глава 13).

```go
type ProducerPool struct {
    tasksCh chan *ProduceTask
    resultsCh chan *ProduceResult
    workers int
}

func (p *ProducerPool) Start(ctx context.Context) {
    var wg sync.WaitGroup
    for i := 0; i < p.workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case task := <-p.tasksCh:
                    result := p.process(ctx, task)
                    select {
                    case p.resultsCh <- result:
                    case <-ctx.Done():
                        return
                    }
                }
            }
        }()
    }
    wg.Wait()
}
```

### Паттерны в producer

| Паттерн | Где |
|:---|:---|
| **Rate limiter** | HTTP middleware |
| **Semaphore** | Ограничение параллелизма |
| **Timeout** | Context |
| **Batch** | Batch produce |
| **Pipeline** | Parse → Validate → Append |
| **Worker pool** | Параллельная запись |
| **Backpressure** | Channel для worker pool |

### 💡 Практика: как писать producer

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Rate limiter** для эндпоинтов.
2. **Semaphore** для ограничения.
3. **Timeout** для операций.
4. **Batch** для производительности.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Worker pool** для параллельной записи.
6. **Pipeline** для обработки.

**❌ НЕ ДЕЛАЙ:**

7. **Не оставляй эндпоинт без rate limit.**

---

## 40.4 Consumer: чтение сообщений

**Consumer** читает сообщения через HTTP.

### HTTP API

```go
func (b *Broker) handleFetch(w http.ResponseWriter, r *http.Request) {
    // ... rate limiter, semaphore, timeout ...
    
    topicName := r.URL.Query().Get("topic")
    partitionID := parseInt(r.URL.Query().Get("partition"))
    offset := parseInt64(r.URL.Query().Get("offset"))
    maxBytes := parseInt(r.URL.Query().Get("max_bytes"))
    
    topic := b.topics[topicName]
    partition := topic.partitions[partitionID]
    
    messages, err := partition.Fetch(ctx, offset, maxBytes)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    
    json.NewEncoder(w).Encode(FetchResponse{Messages: messages})
}
```

### Long polling

**Для низкой latency** — long polling (Глава 3).

```go
func (b *Broker) handleFetchLongPoll(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 30*time.Second)
    defer cancel()
    
    topic := b.topics[r.URL.Query().Get("topic")]
    partition := topic.partitions[parseInt(r.URL.Query().Get("partition"))]
    offset := parseInt64(r.URL.Query().Get("offset"))
    
    // 1. Проверяем, есть ли данные
    messages, _ := partition.Fetch(ctx, offset, 1024*1024)
    if len(messages) > 0 {
        json.NewEncoder(w).Encode(FetchResponse{Messages: messages})
        return
    }
    
    // 2. Ждём новых данных (select — Глава 3)
    select {
    case <-partition.notify:
        messages, _ := partition.Fetch(ctx, offset, 1024*1024)
        json.NewEncoder(w).Encode(FetchResponse{Messages: messages})
    case <-ctx.Done():
        json.NewEncoder(w).Encode(FetchResponse{})
    }
}
```

### Pipeline для чтения

**Чтение** — это **pipeline** (Глава 11):

```
Offset → Segment → Read → Parse → Response
```

### Fan-out для consumer'ов

**Несколько consumer'ов** читают из одной партиции (fan-out — Глава 10).

```go
type PartitionReader struct {
    partition *Partition
    consumers []*Consumer
}

func (r *PartitionReader) Notify(ctx context.Context) {
    // Уведомить всех consumer'ов
    for _, c := range r.consumers {
        select {
        case c.notify <- struct{}{}:
        case <-ctx.Done():
            return
        default:
            // consumer занят — пропускаем
        }
    }
}
```

### Паттерны в consumer

| Паттерн | Где |
|:---|:---|
| **Rate limiter** | HTTP middleware |
| **Semaphore** | Ограничение параллелизма |
| **Timeout** | Context |
| **Long polling** | Select |
| **Pipeline** | Offset → Read → Parse |
| **Fan-out** | Уведомление consumer'ов |
| **Backpressure** | Каналы |

### 💡 Практика: как писать consumer

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Long polling** для низкой latency.
2. **Rate limiter** для эндпоинтов.
3. **Timeout** для операций.
4. **Fan-out** для уведомления.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Pipeline** для чтения.
6. **Backpressure** через каналы.

**❌ НЕ ДЕЛАЙ:**

7. **Не блокируйся без timeout.**

---

## 40.5 Consumer groups: координация

**Consumer groups** — координация между consumer'ами.

### Идея

**Topic** имеет N партиций. **Consumer group** имеет M consumer'ов. Каждая партиция назначается **одному** consumer'у в группе.

```
Topic: orders (3 partitions)
Consumer Group: order-processors (2 consumers)

Assignment:
  Consumer 1: orders-0, orders-1
  Consumer 2: orders-2
```

### Координация через etcd

```go
type GroupCoordinator struct {
    etcd      *clientv3.Client
    groupID   string
    consumerID string
    session   *concurrency.Session
    election  *concurrency.Election
}

func (g *GroupCoordinator) Join(ctx context.Context) error {
    // 1. Leader election для группы (Глава 24)
    session, _ := concurrency.NewSession(g.etcd, concurrency.WithTTL(10))
    g.session = session
    
    election := concurrency.NewElection(session, "/groups/"+g.groupID+"/leader")
    g.election = election
    
    // 2. Пытаемся стать лидером группы
    if err := election.Campaign(ctx, g.consumerID); err != nil {
        return err
    }
    
    // 3. Регистрируем себя
    key := "/groups/" + g.groupID + "/members/" + g.consumerID
    g.etcd.Put(ctx, key, "", clientv3.WithLease(session.Lease()))
    
    return nil
}
```

### Назначение партиций

**Лидер группы** назначает партиции.

```go
func (g *GroupCoordinator) Assign(ctx context.Context) ([]int, error) {
    // 1. Получить список members
    resp, _ := g.etcd.Get(ctx, "/groups/"+g.groupID+"/members/", clientv3.WithPrefix())
    
    var members []string
    for _, kv := range resp.Kvs {
        members = append(members, path.Base(string(kv.Key)))
    }
    
    // 2. Отсортировать
    sort.Strings(members)
    
    // 3. Назначить партиции (round-robin)
    numPartitions := 3  // ← из metadata
    assignments := make(map[string][]int)
    for i := 0; i < numPartitions; i++ {
        member := members[i%len(members)]
        assignments[member] = append(assignments[member], i)
    }
    
    return assignments[g.consumerID], nil
}
```

### Rebalance

**Когда consumer присоединяется или уходит** — rebalance.

```go
func (g *GroupCoordinator) WatchMembers(ctx context.Context) {
    watchCh := g.etcd.Watch(ctx, "/groups/"+g.groupID+"/members/", clientv3.WithPrefix())
    
    for resp := range watchCh {
        for _, ev := range resp.Events {
            switch ev.Type {
            case clientv3.EventTypePut:
                log.Printf("member joined: %s", ev.Kv.Key)
                g.triggerRebalance(ctx)
            case clientv3.EventTypeDelete:
                log.Printf("member left: %s", ev.Kv.Key)
                g.triggerRebalance(ctx)
            }
        }
    }
}
```

### Паттерны в consumer groups

| Паттерн | Где |
|:---|:---|
| **Leader election** | Выбор лидера группы |
| **Watch** | Отслеживание members |
| **Bulkhead** | Изоляция групп |
| **Graceful shutdown** | Leave группы |

### 💡 Практика: как писать consumer groups

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Leader election** для группы.
2. **Watch** для members.
3. **Rebalance** при изменениях.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Graceful shutdown** при уходе.

**❌ НЕ ДЕЛАЙ:**

5. **Не назначай партиции** без координации.

---

## 40.6 Репликация: leader election и fencing tokens

**Репликация** — копирование данных между брокерами.

### Идея

**Каждая партиция** имеет **лидера** и **follower'ов**.

```
Partition orders-0:
  Leader:   Broker 1
  Follower: Broker 2
  Follower: Broker 3

Producer пишет в Leader.
Leader реплицирует на Followers.
```

### Leader election для партиции

```go
type PartitionReplication struct {
    partitionID string
    brokerID    string
    etcd        *clientv3.Client
    isLeader    atomic.Bool
}

func (r *PartitionReplication) Campaign(ctx context.Context) error {
    session, _ := concurrency.NewSession(r.etcd, concurrency.WithTTL(10))
    election := concurrency.NewElection(session, "/partitions/"+r.partitionID+"/leader")
    
    if err := election.Campaign(ctx, r.brokerID); err != nil {
        return err
    }
    
    r.isLeader.Store(true)
    return nil
}
```

### Fencing tokens

**При записи** — использовать fencing token (Глава 39).

```go
func (r *PartitionReplication) Append(ctx context.Context, msg *Message) error {
    // 1. Получить fencing token
    token := r.session.Lease()
    
    // 2. Проверить, что мы всё ещё лидер
    if !r.isLeader.Load() {
        return ErrNotLeader
    }
    
    // 3. Записать с token
    return r.storage.Append(ctx, msg, token)
}
```

### Репликация на followers

**Leader реплицирует** на followers.

```go
func (r *PartitionReplication) Replicate(ctx context.Context, msg *Message) error {
    // Fan-out на followers (Глава 10)
    g, ctx := errgroup.WithContext(ctx)
    
    for _, follower := range r.followers {
        follower := follower
        g.Go(func() error {
            return retryCtx(ctx, 3, 100*time.Millisecond, func(ctx context.Context) error {
                return follower.Append(ctx, msg)
            })
        })
    }
    
    return g.Wait()
}
```

### Quorum

**Для надёжности** — quorum.

```go
func (r *PartitionReplication) AppendWithQuorum(ctx context.Context, msg *Message) error {
    // 1. Записать локально
    if err := r.storage.Append(ctx, msg); err != nil {
        return err
    }
    
    // 2. Реплицировать на followers
    acks := 1  // ← мы
    var mu sync.Mutex
    
    g, ctx := errgroup.WithContext(ctx)
    for _, follower := range r.followers {
        follower := follower
        g.Go(func() error {
            if err := follower.Append(ctx, msg); err != nil {
                return err
            }
            mu.Lock()
            acks++
            mu.Unlock()
            return nil
        })
    }
    
    g.Wait()
    
    // 3. Проверить quorum
    if acks < r.quorum {
        return ErrQuorumNotReached
    }
    return nil
}
```

### Паттерны в репликации

| Паттерн | Где |
|:---|:---|
| **Leader election** | Выбор лидера партиции |
| **Fencing tokens** | Защита от split-brain |
| **Retry** | Повтор при ошибке |
| **Circuit breaker** | Защита от сбоев follower'ов |
| **Fan-out** | Репликация на followers |
| **errgroup** | Ожидание всех followers |
| **Quorum** | Большинство для надёжности |

### 💡 Практика: как писать репликацию

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Leader election** для партиции.
2. **Fencing tokens** для защиты.
3. **Retry** для followers.
4. **Quorum** для надёжности.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Circuit breaker** для сбоев follower'ов.

**❌ НЕ ДЕЛАЙ:**

6. **Не пиши без quorum** для критичных данных.

---

## 40.7 Метаданные: registry и discovery

**Metadata** — информация о топиках, партициях, лидерах.

### Идея

**Metadata registry** хранит:

- Список топиков.
- Список партиций.
- Лидеры партиций.
- Follower'ы.

**Хранится в etcd.**

### Структура

```
/metadata/
├── topics/
│   ├── orders/
│   │   ├── partitions: 3
│   │   └── replication_factor: 3
│   └── events/
│       ├── partitions: 2
│       └── replication_factor: 2
└── partitions/
    ├── orders-0/
    │   ├── leader: broker-1
    │   └── followers: [broker-2, broker-3]
    ├── orders-1/
    │   ├── leader: broker-2
    │   └── followers: [broker-1, broker-3]
    └── ...
```

### Metadata client

```go
type MetadataClient struct {
    etcd *clientv3.Client
    cache sync.Map  // ← кэш (Глава 20)
}

func (c *MetadataClient) GetTopic(ctx context.Context, name string) (*TopicMetadata, error) {
    // 1. Проверить кэш
    if v, ok := c.cache.Load(name); ok {
        return v.(*TopicMetadata), nil
    }
    
    // 2. Singleflight (Глава 20)
    v, _, _ := group.Do(name, func() (interface{}, error) {
        return c.fetchTopic(ctx, name)
    })
    
    return v.(*TopicMetadata), nil
}

func (c *MetadataClient) fetchTopic(ctx context.Context, name string) (*TopicMetadata, error) {
    resp, err := c.etcd.Get(ctx, "/metadata/topics/"+name)
    if err != nil {
        return nil, err
    }
    
    var meta TopicMetadata
    json.Unmarshal(resp.Kvs[0].Value, &meta)
    
    c.cache.Store(name, &meta)
    return &meta, nil
}
```

### Discovery: как клиент находит лидера

**Producer** должен знать, куда писать.

```go
func (p *Producer) Produce(ctx context.Context, msg *Message) error {
    // 1. Получить metadata
    meta, err := p.metadata.GetTopic(ctx, msg.Topic)
    if err != nil {
        return err
    }
    
    // 2. Выбрать партицию (по ключу)
    partition := hash(msg.Key) % meta.Partitions
    
    // 3. Найти лидера партиции
    leader, err := p.metadata.GetLeader(ctx, msg.Topic, partition)
    if err != nil {
        return err
    }
    
    // 4. Отправить лидеру
    return retryCtx(ctx, 3, 100*time.Millisecond, func(ctx context.Context) error {
        return p.sendToLeader(ctx, leader, msg)
    })
}
```

### Паттерны в metadata

| Паттерн | Где |
|:---|:---|
| **Batch** | Обновление metadata |
| **Retry** | При сбое etcd |
| **Circuit breaker** | Защита от сбоев etcd |
| **Cache** | Кэш metadata |
| **Singleflight** | Один запрос для нескольких |
| **Watch** | Отслеживание изменений |

### 💡 Практика: как писать metadata

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Cache** для metadata.
2. **Singleflight** для избежания дублирования.
3. **Retry** при сбое etcd.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Watch** для отслеживания изменений.
5. **Circuit breaker** для etcd.

**❌ НЕ ДЕЛАЙ:**

6. **Не читай metadata** на каждое сообщение.

---

## 40.8 Защита от DoS и deadlock

**Безопасность** — критична для брокера.

### Middleware для защиты

```go
func secureMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 1. Rate limiter (Глава 14)
        if !limiter.Allow() {
            http.Error(w, "too many requests", 429)
            return
        }
        
        // 2. Semaphore (Глава 12)
        select {
        case sem <- struct{}{}:
            defer func() { <-sem }()
        case <-r.Context().Done():
            return
        default:
            http.Error(w, "server busy", 503)
            return
        }
        
        // 3. Timeout (Глава 17)
        ctx, cancel := context.WithTimeout(r.Context(), 30*time.Second)
        defer cancel()
        
        // 4. Max body size
        r.Body = http.MaxBytesReader(w, r.Body, 10*1024*1024)
        
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### Bulkhead для изоляции

**Разные bulkhead** для разных операций (Глава 22).

```go
type Broker struct {
    produceBulkhead *Bulkhead  // 100 слотов
    fetchBulkhead   *Bulkhead  // 200 слотов
    adminBulkhead   *Bulkhead  // 10 слотов
}
```

**Что даёт:** медленный admin **не занимает** слоты produce.

### Защита от deadlock

**Один порядок захвата мьютексов.**

```go
func (b *Broker) processTopicAndPartition(topicName string, partitionID int) error {
    // 1. Сначала topic
    b.topicsMu.RLock()
    topic := b.topics[topicName]
    b.topicsMu.RUnlock()
    
    // 2. Потом partition
    partition := topic.partitions[partitionID]
    partition.mu.Lock()
    defer partition.mu.Unlock()
    
    // ... работа ...
    return nil
}
```

**Что даёт:** нет циклической зависимости.

### Защита от goroutine exhaustion

**Semaphore на все горутины.**

```go
type Broker struct {
    goroutineSem chan struct{}  // 10 000 слотов
}

func (b *Broker) SpawnWorker(fn func()) error {
    select {
    case b.goroutineSem <- struct{}{}:
        go func() {
            defer func() { <-b.goroutineSem }()
            fn()
        }()
        return nil
    default:
        return ErrTooManyGoroutines
    }
}
```

### Паттерны в защите

| Паттерн | Где |
|:---|:---|
| **Rate limiter** | HTTP middleware |
| **Semaphore** | Ограничение параллелизма |
| **Timeout** | Context |
| **Bulkhead** | Изоляция операций |
| **Deadlock prevention** | Один порядок захвата |
| **Goroutine limit** | Semaphore на горутины |

### 💡 Практика: как защищать брокер

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Rate limiter** для всех эндпоинтов.
2. **Semaphore** для ограничения.
3. **Timeout** для операций.
4. **Bulkhead** для изоляции.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Один порядок захвата мьютексов.**
6. **Goroutine limit.**

**❌ НЕ ДЕЛАЙ:**

7. **Не оставляй эндпоинты без защиты.**

---

## 40.9 Graceful shutdown кластера

**Graceful shutdown** — корректное завершение (Глава 26).

### Последовательность

```
1. SIGTERM → Broker
   ↓
2. Прекратить принимать новые соединения
   ↓
3. Дождаться завершения активных запросов
   ↓
4. Финальный flush WAL
   ↓
5. Leave consumer groups
   ↓
6. Остановить replication
   ↓
7. Закрыть etcd session
   ↓
8. Выход
```

### Код

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()
    
    broker := NewBroker()
    
    // Запуск компонентов
    g, ctx := errgroup.WithContext(ctx)
    g.Go(func() error { return broker.RunHTTP(ctx) })
    g.Go(func() error { return broker.RunReplication(ctx) })
    g.Go(func() error { return broker.RunMetadata(ctx) })
    
    <-ctx.Done()
    log.Println("shutdown signal received")
    
    // Shutdown timeout (Глава 26)
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
    defer cancel()
    
    // 1. Прекратить принимать HTTP
    if err := broker.HTTPServer.Shutdown(shutdownCtx); err != nil {
        log.Printf("http shutdown error: %v", err)
    }
    
    // 2. Финальный flush WAL
    if err := broker.FlushAll(shutdownCtx); err != nil {
        log.Printf("flush error: %v", err)
    }
    
    // 3. Leave consumer groups
    if err := broker.LeaveGroups(shutdownCtx); err != nil {
        log.Printf("leave error: %v", err)
    }
    
    // 4. Остановить replication
    broker.StopReplication()
    
    // 5. Закрыть etcd
    broker.Close()
    
    // Ждём завершения всех горутин
    if err := g.Wait(); err != nil && !errors.Is(err, context.Canceled) {
        log.Printf("run error: %v", err)
    }
    
    log.Println("shutdown complete")
}
```

### Drain WAL

**Финальный flush** (Глава 21, batch).

```go
func (b *Broker) FlushAll(ctx context.Context) error {
    var wg sync.WaitGroup
    var mu sync.Mutex
    var errs []error
    
    b.topicsMu.RLock()
    for _, topic := range b.topics {
        for _, partition := range topic.partitions {
            wg.Add(1)
            partition := partition
            go func() {
                defer wg.Done()
                if err := partition.Flush(ctx); err != nil {
                    mu.Lock()
                    errs = append(errs, err)
                    mu.Unlock()
                }
            }()
        }
    }
    b.topicsMu.RUnlock()
    
    wg.Wait()
    return errors.Join(errs...)
}
```

### Leave consumer groups

**Отписаться** от групп.

```go
func (b *Broker) LeaveGroups(ctx context.Context) error {
    b.groupsMu.RLock()
    defer b.groupsMu.RUnlock()
    
    for _, coordinator := range b.groups {
        if err := coordinator.Leave(ctx); err != nil {
            return err
        }
    }
    return nil
}
```

### Паттерны в shutdown

| Паттерн | Где |
|:---|:---|
| **signal.NotifyContext** | Перехват сигналов |
| **errgroup** | Запуск компонентов |
| **Timeout** | Shutdown timeout |
| **WaitGroup** | Ожидание компонентов |
| **Flush** | Финальный flush |
| **errors.Join** | Сбор ошибок |

### 💡 Практика: как делать shutdown

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **signal.NotifyContext** для сигналов.
2. **errgroup** для компонентов.
3. **Shutdown timeout.**
4. **Финальный flush.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Leave consumer groups.**
6. **errors.Join** для сбора ошибок.

**❌ НЕ ДЕЛАЙ:**

7. **Не забывай про flush.**
8. **Не жди вечно.**

---

## 40.10 Тестирование брокера

**Тестирование** — критично (Глава 28).

### Unit-тесты

```go
func TestPartitionAppend(t *testing.T) {
    defer goleak.VerifyNone(t)
    
    p := NewPartition(0, "test")
    ctx := context.Background()
    
    msg := &Message{Key: []byte("k"), Value: []byte("v")}
    offset, err := p.Append(ctx, msg)
    if err != nil {
        t.Fatal(err)
    }
    if offset != 0 {
        t.Errorf("offset = %d, want 0", offset)
    }
    
    msg2 := &Message{Key: []byte("k"), Value: []byte("v")}
    offset, err = p.Append(ctx, msg2)
    if err != nil {
        t.Fatal(err)
    }
    if offset != 1 {
        t.Errorf("offset = %d, want 1", offset)
    }
}
```

### Stress-тесты

```go
func TestPartitionConcurrentAppend(t *testing.T) {
    defer goleak.VerifyNone(t)
    
    p := NewPartition(0, "test")
    ctx := context.Background()
    
    const N = 10000
    const workers = 100
    
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        i := i
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < N/workers; j++ {
                msg := &Message{
                    Key:   []byte(fmt.Sprintf("k-%d-%d", i, j)),
                    Value: []byte("v"),
                }
                if _, err := p.Append(ctx, msg); err != nil {
                    t.Errorf("append error: %v", err)
                    return
                }
            }
        }()
    }
    wg.Wait()
    
    if p.NextOffset() != N {
        t.Errorf("next offset = %d, want %d", p.NextOffset(), N)
    }
}
```

### Интеграционные тесты

```go
func TestProduceConsume(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }
    defer goleak.VerifyNone(t)
    
    // Запуск брокера
    broker := NewBroker()
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    go broker.Run(ctx)
    defer broker.Stop()
    
    // Ждём запуска
    time.Sleep(100 * time.Millisecond)
    
    // Produce
    resp, err := http.Post("http://localhost:8080/produce",
        "application/json",
        strings.NewReader(`{"topic":"orders","partition":0,"key":"k1","value":"hello"}`))
    if err != nil {
        t.Fatal(err)
    }
    resp.Body.Close()
    
    // Consume
    resp, err = http.Get("http://localhost:8080/fetch?topic=orders&partition=0&offset=0")
    if err != nil {
        t.Fatal(err)
    }
    defer resp.Body.Close()
    
    var result FetchResponse
    json.NewDecoder(resp.Body).Decode(&result)
    
    if len(result.Messages) != 1 {
        t.Fatalf("got %d messages, want 1", len(result.Messages))
    }
    if string(result.Messages[0].Value) != "hello" {
        t.Errorf("value = %q, want 'hello'", result.Messages[0].Value)
    }
}
```

### Race detector

```bash
go test -race ./...
```

### Бенчмарки

```go
func BenchmarkPartitionAppend(b *testing.B) {
    p := NewPartition(0, "test")
    ctx := context.Background()
    
    b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            msg := &Message{Key: []byte("k"), Value: []byte("v")}
            p.Append(ctx, msg)
        }
    })
}
```

### Паттерны в тестировании

| Паттерн | Где |
|:---|:---|
| **goleak** | Проверка утечек |
| **Stress-тесты** | Параллельные |
| **Интеграционные** | HTTP API |
| **Race detector** | `-race` |
| **Бенчмарки** | `b.RunParallel` |

### 💡 Практика: как тестировать брокер

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Unit-тесты** для компонентов.
2. **Stress-тесты** для конкурентности.
3. **Интеграционные** для HTTP.
4. **`-race` в CI.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **`goleak`** для утечек.
6. **Бенчмарки.**

**❌ НЕ ДЕЛАЙ:**

7. **Не полагайся на один запуск.**

---

## 40.11 Практика Go: полный код

Соберём **упрощённый брокер**.

### Основные структуры

```go
package broker

import (
    "context"
    "encoding/json"
    "errors"
    "log"
    "net/http"
    "sync"
    "sync/atomic"
    "time"
    
    "golang.org/x/sync/errgroup"
    "golang.org/x/time/rate"
)

type Message struct {
    Offset    int64             `json:"offset"`
    Key       []byte            `json:"key"`
    Value     []byte            `json:"value"`
    Timestamp time.Time         `json:"timestamp"`
    Headers   map[string][]byte `json:"headers"`
}

type Partition struct {
    id         int
    topic      string
    nextOffset atomic.Int64
    messages   []*Message
    mu         sync.RWMutex
    notify     chan struct{}
}

func NewPartition(id int, topic string) *Partition {
    return &Partition{
        id:       id,
        topic:    topic,
        messages: make([]*Message, 0, 10000),
        notify:   make(chan struct{}, 1),
    }
}

func (p *Partition) Append(ctx context.Context, msg *Message) (int64, error) {
    p.mu.Lock()
    defer p.mu.Unlock()
    
    offset := p.nextOffset.Add(1) - 1
    msg.Offset = offset
    msg.Timestamp = time.Now()
    p.messages = append(p.messages, msg)
    
    // Notify
    select {
    case p.notify <- struct{}{}:
    default:
    }
    
    return offset, nil
}

func (p *Partition) Fetch(ctx context.Context, offset int64, maxBytes int) ([]*Message, error) {
    p.mu.RLock()
    defer p.mu.RUnlock()
    
    if offset >= int64(len(p.messages)) {
        return nil, nil
    }
    
    result := make([]*Message, 0)
    size := 0
    for i := offset; i < int64(len(p.messages)); i++ {
        msg := p.messages[i]
        msgSize := len(msg.Key) + len(msg.Value)
        if size+msgSize > maxBytes {
            break
        }
        result = append(result, msg)
        size += msgSize
    }
    return result, nil
}

func (p *Partition) Notify() <-chan struct{} {
    return p.notify
}

type Topic struct {
    name       string
    partitions []*Partition
}

func NewTopic(name string, numPartitions int) *Topic {
    t := &Topic{name: name, partitions: make([]*Partition, numPartitions)}
    for i := 0; i < numPartitions; i++ {
        t.partitions[i] = NewPartition(i, name)
    }
    return t
}

type Broker struct {
    topics   map[string]*Topic
    topicsMu sync.RWMutex
    
    limiter  *rate.Limiter
    sem      chan struct{}
    httpSrv  *http.Server
}

func NewBroker() *Broker {
    b := &Broker{
        topics:  make(map[string]*Topic),
        limiter: rate.NewLimiter(rate.Limit(10000), 100),
        sem:     make(chan struct{}, 1000),
    }
    
    // Создаём топик по умолчанию
    b.topics["orders"] = NewTopic("orders", 3)
    
    return b
}

func (b *Broker) handleProduce(w http.ResponseWriter, r *http.Request) {
    // 1. Rate limiter
    if !b.limiter.Allow() {
        http.Error(w, "too many requests", 429)
        return
    }
    
    // 2. Semaphore
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
    case <-r.Context().Done():
        return
    default:
        http.Error(w, "server busy", 503)
        return
    }
    
    // 3. Timeout
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()
    
    // 4. Parse
    var req struct {
        Topic     string  `json:"topic"`
        Partition int     `json:"partition"`
        Key       string  `json:"key"`
        Value     string  `json:"value"`
    }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "bad request", 400)
        return
    }
    
    // 5. Find topic
    b.topicsMu.RLock()
    topic := b.topics[req.Topic]
    b.topicsMu.RUnlock()
    
    if topic == nil {
        http.Error(w, "topic not found", 404)
        return
    }
    
    if req.Partition < 0 || req.Partition >= len(topic.partitions) {
        http.Error(w, "invalid partition", 400)
        return
    }
    
    // 6. Append
    partition := topic.partitions[req.Partition]
    offset, err := partition.Append(ctx, &Message{
        Key:   []byte(req.Key),
        Value: []byte(req.Value),
    })
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    
    // 7. Response
    json.NewEncoder(w).Encode(map[string]int64{"offset": offset})
}

func (b *Broker) handleFetch(w http.ResponseWriter, r *http.Request) {
    // ... rate limiter, semaphore ...
    
    ctx, cancel := context.WithTimeout(r.Context(), 30*time.Second)
    defer cancel()
    
    topicName := r.URL.Query().Get("topic")
    partitionID := 0
    offset := int64(0)
    
    b.topicsMu.RLock()
    topic := b.topics[topicName]
    b.topicsMu.RUnlock()
    
    if topic == nil {
        http.Error(w, "topic not found", 404)
        return
    }
    
    partition := topic.partitions[partitionID]
    
    // Long polling
    messages, _ := partition.Fetch(ctx, offset, 1024*1024)
    if len(messages) == 0 {
        select {
        case <-partition.Notify():
            messages, _ = partition.Fetch(ctx, offset, 1024*1024)
        case <-ctx.Done():
        }
    }
    
    json.NewEncoder(w).Encode(map[string]interface{}{"messages": messages})
}

func (b *Broker) Run(ctx context.Context) error {
    mux := http.NewServeMux()
    mux.HandleFunc("/produce", b.handleProduce)
    mux.HandleFunc("/fetch", b.handleFetch)
    
    b.httpSrv = &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  60 * time.Second,
    }
    
    g, ctx := errgroup.WithContext(ctx)
    
    g.Go(func() error {
        log.Println("HTTP server starting on :8080")
        if err := b.httpSrv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            return err
        }
        return nil
    })
    
    <-ctx.Done()
    
    // Shutdown
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    return b.httpSrv.Shutdown(shutdownCtx)
}

func main() {
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()
    
    broker := NewBroker()
    
    if err := broker.Run(ctx); err != nil {
        log.Fatal(err)
    }
    
    log.Println("shutdown complete")
}
```

### Что демонстрирует

1. **Rate limiter** — `rate.NewLimiter`.
2. **Semaphore** — `sem chan struct{}`.
3. **Timeout** — `context.WithTimeout`.
4. **Long polling** — `select` на `Notify()`.
5. **Graceful shutdown** — `signal.NotifyContext` + `Shutdown`.
6. **RWMutex** — для topics.
7. **atomic** — для offset.
8. **Fan-out** — notify.

### 💡 Практика: как собирать всё вместе

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Начинай** с минимального рабочего.
2. **Добавляй** паттерны **по мере необходимости**.
3. **Тестируй** каждый компонент.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Документируй** решения.
5. **Мониторь** метрики.

**❌ НЕ ДЕЛАЙ:**

6. **Не пытайся** применить **все** паттерны сразу.

---

## 40.12 Выводы и типичные ошибки

**Что мы узнали?**

Мы построили **упрощённый брокер сообщений** и применили **все** паттерны:

- **Storage** — atomic, batch, sharded locks, RWMutex.
- **Producer** — rate limiter, semaphore, timeout, batch, worker pool.
- **Consumer** — long polling, fan-out, pipeline.
- **Consumer groups** — leader election, watch, bulkhead.
- **Replication** — leader election, fencing tokens, retry, quorum.
- **Metadata** — cache, singleflight, watch.
- **Защита** — rate limiter, semaphore, bulkhead, timeout.
- **Shutdown** — signal.NotifyContext, errgroup, flush.
- **Тестирование** — goleak, stress, integration, race detector.

**Типичные ошибки:**

- ❌ **Применять паттерны без необходимости.**
- ❌ **Один mutex на всё.**
- ❌ **Не использовать atomic для offset.**
- ❌ **Не делать batch.**
- ❌ **Без rate limiter.**
- ❌ **Без quorum.**
- ❌ **Без fencing tokens.**
- ❌ **Не делать flush при shutdown.**
- ❌ **Не тестировать с `-race`.**
- ❌ **Не использовать goleak.**
- ❌ **Сложный код без документации.**

---

## 40.13 Для быстрого повторения

- **Брокер** — упрощённый Kafka на Go.
- **Топики** — логический поток.
- **Партиции** — часть топика.
- **Segments** — файлы на диске.
- **Offset** — порядковый номер.
- **Consumer groups** — координация.
- **Leader/Follower** — для каждой партиции.
- **Storage:** atomic, batch, sharded locks, RWMutex.
- **Producer:** rate limiter, semaphore, timeout, batch.
- **Consumer:** long polling, fan-out, pipeline.
- **Groups:** leader election, watch, bulkhead.
- **Replication:** leader election, fencing, retry, quorum.
- **Metadata:** cache, singleflight, watch.
- **Защита:** rate limiter, semaphore, bulkhead, timeout.
- **Shutdown:** signal.NotifyContext, errgroup, flush.
- **Тесты:** goleak, stress, integration, `-race`.

---

## 40.14 Вопросы для самопроверки

1. Какие паттерны применяются в storage?
2. Какие паттерны в producer?
3. Какие паттерны в consumer?
4. Как работает consumer group?
5. Что такое fencing tokens?
6. Какие паттерны в metadata?
7. Как защититься от DoS?
8. Как делать graceful shutdown?
9. Как тестировать брокер?
10. Какие анти-паттерны?

---

## 40.15 Ответы

### Ответ 1

**В storage:**

- **atomic** — offset.
- **batch** — запись в WAL.
- **sharded locks** — 16 шардов.
- **RWMutex** — topics map.
- **backpressure** — каналы.

### Ответ 2

**В producer:**

- **rate limiter** — HTTP.
- **semaphore** — параллелизм.
- **timeout** — context.
- **batch** — группировка.
- **worker pool** — параллельная запись.
- **pipeline** — parse → validate.

### Ответ 3

**В consumer:**

- **long polling** — select.
- **fan-out** — notify.
- **pipeline** — offset → read.
- **backpressure** — каналы.

### Ответ 4

**Consumer group:**

1. Leader election для группы.
2. Лидер назначает партиции.
3. Watch members.
4. Rebalance при изменениях.

### Ответ 5

**Fencing tokens** — монотонные номера для защиты от split-brain.

```sql
UPDATE data SET token = $2
WHERE resource = $3 AND token < $2;
```

### Ответ 6

**В metadata:**

- **cache** — локальный кэш.
- **singleflight** — один запрос.
- **watch** — отслеживание.
- **retry** — при сбое.
- **circuit breaker** — защита.

### Ответ 7

**От DoS:**

- **rate limiter.**
- **semaphore.**
- **timeout.**
- **bulkhead.**
- **goroutine limit.**

### Ответ 8

**Graceful shutdown:**

1. signal.NotifyContext.
2. errgroup для компонентов.
3. Shutdown timeout.
4. Финальный flush.
5. Leave groups.
6. Close etcd.

### Ответ 9

**Тестирование:**

- **unit-тесты** для компонентов.
- **stress-тесты** для конкурентности.
- **интеграционные** для HTTP.
- **`-race` в CI.**
- **goleak** для утечек.
- **бенчмарки.**

### Ответ 10

**Анти-паттерны:**

- Паттерны без необходимости.
- Один mutex на всё.
- Без atomic для offset.
- Без batch.
- Без rate limiter.
- Без quorum.
- Без fencing.
- Без flush.
- Без тестов.
- Сложный код без документации.

---

## 40.16 Куда идти дальше?

Мы **завершили** книгу. Мы прошли путь от **горутин** до **сложного проекта**.

**Что дальше:**

- **Применяй** знания в своих проектах.
- **Читай** исходники Go (runtime, sync, net).
- **Изучай** другие системы (Kafka, etcd, NATS).
- **Участвуй** в open source.

**Приложения:**

- **Приложение A:** Шпаргалка по каналам.
- **Приложение B:** Шпаргалка по sync.
- **Приложение C:** Шпаргалка по context.
- **Приложение D:** Шпаргалка по планировщику.
- **Приложение E:** Инструменты и метрики.

**Спасибо за чтение!**

---

## 40.17 Чек-лист

| Компонент | Паттерны |
|:---|:---|
| **Storage** | atomic, batch, sharded locks, RWMutex |
| **Producer** | rate limiter, semaphore, timeout, batch, worker pool |
| **Consumer** | long polling, fan-out, pipeline |
| **Groups** | leader election, watch, bulkhead |
| **Replication** | leader election, fencing, retry, quorum |
| **Metadata** | cache, singleflight, watch |
| **Защита** | rate limiter, semaphore, bulkhead, timeout |
| **Shutdown** | signal.NotifyContext, errgroup, flush |
| **Тесты** | goleak, stress, integration, `-race` |

🏭 **Ключевая идея:** Мы построили **упрощённый брокер сообщений** и применили **все** паттерны из книги. **Storage** использует atomic, batch, sharded locks. **Producer** — rate limiter, semaphore, timeout, batch, worker pool. **Consumer** — long polling, fan-out, pipeline. **Consumer groups** — leader election, watch, bulkhead. **Replication** — leader election, fencing tokens, retry, quorum. **Metadata** — cache, singleflight, watch. **Защита** — rate limiter, semaphore, bulkhead, timeout. **Shutdown** — signal.NotifyContext, errgroup, flush. **Тесты** — goleak, stress, integration, `-race`. Не применяй паттерны без необходимости. Начинай с простого. Тестируй каждый компонент. Документируй решения.