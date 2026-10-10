# 📨 Глава 35: Очереди и consumer

**Что вы узнаете:**
- Как построить **production-ready consumer** для очередей (Kafka, NATS, RabbitMQ).
- Как организовать **worker pool** для параллельной обработки сообщений.
- Как делать **batch commit** для производительности.
- Как обрабатывать **retry** и **dead letter queue (DLQ)**.
- Как корректно **дренировать** очередь при shutdown.
- Как **мониторить** lag и throughput.
- Как избежать **потери сообщений** и **дубликатов**.
- Как комбинировать с rate limiter, circuit breaker, backpressure.

**После прочтения вы сможете:**
- Построить consumer с worker pool.
- Настроить batch commit и at-least-once доставку.
- Реализовать retry с backoff и DLQ.
- Корректно завершать consumer по `SIGTERM`.
- Мониторить lag, throughput, ошибки.
- Понимать, где очереди уместны, а где — нет.

---

## Содержание

- [35.0 Пролог: consumer, который потерял сообщения](#350-пролог-consumer-который-потерял-сообщения)
- [35.1 Базовая структура consumer](#351-базовая-структура-consumer)
- [35.2 Worker pool для параллельной обработки](#352-worker-pool-для-параллельной-обработки)
- [35.3 Batch commit: производительность](#353-batch-commit-производительность)
- [35.4 Retry и Dead Letter Queue](#354-retry-и-dead-letter-queue)
- [35.5 Graceful shutdown: drain очереди](#355-graceful-shutdown-drain-очереди)
- [35.6 Мониторинг: lag, throughput, errors](#356-мониторинг-lag-throughput-errors)
- [35.7 At-least-once vs exactly-once](#357-at-least-once-vs-exactly-once)
- [35.8 Полный production-ready consumer](#358-полный-production-ready-consumer)
- [35.9 Выводы и типичные ошибки](#359-выводы-и-типичные-ошибки)
- [35.10 Для быстрого повторения](#3510-для-быстрого-повторения)
- [35.11 Вопросы для самопроверки](#3511-вопросы-для-самопроверки)
- [35.12 Ответы](#3512-ответы)
- [35.13 Куда идти дальше?](#3513-куда-идти-дальше)
- [35.14 Чек-лист](#3514-чек-лист)

---

## 35.0 Пролог: consumer, который потерял сообщения

У нас есть сервис, который читает заказы из Kafka и обрабатывает их. Пишем наивно:

```go
func main() {
    consumer, _ := kafka.NewConsumer(&kafka.ConfigMap{
        "bootstrap.servers": "localhost:9092",
        "group.id":          "orders",
        "auto.offset.reset": "earliest",
    })
    consumer.SubscribeTopics([]string{"orders"}, nil)

    for {
        msg, err := consumer.ReadMessage(-1)
        if err != nil {
            continue
        }

        if err := process(msg); err != nil {
            log.Println("error:", err)
        }

        consumer.CommitMessage(msg)
    }
}
```

Работает. Пока не замечаем: **иногда сообщения обрабатываются дважды**. А иногда — **теряются**.

**Что происходит:**

- `process` может зависнуть — consumer не читает.
- `CommitMessage` может упасть — сообщение **потеряно** (at-most-once).
- `process` может вернуть ошибку — мы всё равно коммитим (**потеря**).
- `process` долгий — Kafka выкидывает consumer из группы.

Хочется: **надёжный** consumer, который:

1. **Не теряет** сообщения.
2. **Обрабатывает параллельно** через worker pool.
3. **Делает batch commit** для производительности.
4. **Ретраит** временные ошибки.
5. **Отправляет в DLQ** постоянные.
6. **Дренирует** очередь при shutdown.
7. **Мониторит** lag и throughput.

> **Мост к следующим главам:** consumer — синтез паттернов: worker pool (Глава 13), batch (Глава 21), retry (Глава 16), graceful shutdown (Глава 26), backpressure (Глава 27), rate limiter (Глава 14). Понимание consumer'а даёт понимание, **как обрабатывать потоки в production**.

---

## 35.1 Базовая структура consumer

Начнём с **правильной** структуры.

### Интерфейс

**Абстрагируемся от конкретной очереди:**

```go
type Message struct {
    ID    string
    Body  []byte
    Topic string
    Offset int64
}

type Consumer interface {
    Read(ctx context.Context) (Message, error)
    Commit(ctx context.Context, msg Message) error
    Close() error
}
```

**Что даёт:**

- **Kafka, NATS, RabbitMQ** — разные реализации.
- **Тестирование** — mock-реализация.

### Consumer для Kafka

```go
type KafkaConsumer struct {
    consumer *kafka.Consumer
}

func NewKafkaConsumer(brokers, groupID string, topics []string) (*KafkaConsumer, error) {
    c, err := kafka.NewConsumer(&kafka.ConfigMap{
        "bootstrap.servers": brokers,
        "group.id":          groupID,
        "auto.offset.reset": "earliest",
        "enable.auto.commit": false,  // ← отключаем auto-commit
    })
    if err != nil {
        return nil, err
    }

    if err := c.SubscribeTopics(topics, nil); err != nil {
        return nil, err
    }

    return &KafkaConsumer{consumer: c}, nil
}

func (kc *KafkaConsumer) Read(ctx context.Context) (Message, error) {
    msg, err := kc.consumer.ReadMessage(100 * time.Millisecond)
    if err != nil {
        if errors.Is(err, kafka.ErrTimedOut) {
            return Message{}, ErrTimeout
        }
        return Message{}, err
    }

    return Message{
        ID:     string(msg.Key),
        Body:   msg.Value,
        Topic:  *msg.TopicPartition.Topic,
        Offset: int64(msg.TopicPartition.Offset),
    }, nil
}

func (kc *KafkaConsumer) Commit(ctx context.Context, msg Message) error {
    _, err := kc.consumer.CommitMessage(&kafka.Message{
        TopicPartition: kafka.TopicPartition{
            Topic:     &msg.Topic,
            Partition: kafka.PartitionAny,
            Offset:    kafka.Offset(msg.Offset + 1),
        },
    })
    return err
}

func (kc *KafkaConsumer) Close() error {
    return kc.consumer.Close()
}
```

**Ключевое:**

- **`enable.auto.commit: false`** — коммитим **вручную**.
- **`ReadMessage` с таймаутом** — не блокируемся навсегда.
- **`Commit` после обработки** — at-least-once.

### Обработчик

```go
type Handler interface {
    Handle(ctx context.Context, msg Message) error
}
```

**Что даёт:**

- **Разные обработчики** для разных топиков.
- **Тестирование** — mock-обработчик.

### Consumer loop

```go
type Consumer struct {
    consumer Consumer
    handler  Handler
}

func (c *Consumer) Run(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }

        msg, err := c.consumer.Read(ctx)
        if err != nil {
            if errors.Is(err, ErrTimeout) {
                continue
            }
            return err
        }

        if err := c.handler.Handle(ctx, msg); err != nil {
            log.Printf("handler error: %v", err)
            continue  // ← не коммитим, будет retry
        }

        if err := c.consumer.Commit(ctx, msg); err != nil {
            log.Printf("commit error: %v", err)
        }
    }
}
```

**Что происходит:**

- При ошибке обработчика — **не коммитим** → Kafka **переотправит**.
- При успехе — коммитим.

### 💡 Практика: как построить базовый consumer

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Интерфейс `Consumer`** — абстракция.
2. **`enable.auto.commit: false`** — ручной commit.
3. **`ReadMessage` с таймаутом.**
4. **Commit только после успеха.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Интерфейс `Handler`.**
6. **Mock-реализации** для тестов.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `auto.commit: true`.** Потеря сообщений.
8. **Не коммить до обработки.**
9. **Не блокируйся на `ReadMessage(-1)`.**

---

## 35.2 Worker pool для параллельной обработки

Один consumer **не успевает**. Добавим worker pool.

### Идея

- **Reader** читает из Kafka.
- **Канал сообщений** между reader и воркерами.
- **N воркеров** обрабатывают параллельно.
- **Commit** — после обработки.

### Схема

```
Kafka ──► Reader ──► messagesCh ──► N воркеров
                                        │
                                        ├── process
                                        ├── commit
                                        └── errors
```

### Реализация

```go
type Consumer struct {
    consumer  Consumer
    handler   Handler
    workers   int
    bufferSize int
}

func (c *Consumer) Run(ctx context.Context) error {
    messagesCh := make(chan Message, c.bufferSize)

    var wg sync.WaitGroup
    for i := 0; i < c.workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            c.worker(ctx, messagesCh)
        }()
    }

    // Reader
    readerErr := make(chan error, 1)
    go func() {
        readerErr <- c.reader(ctx, messagesCh)
        close(messagesCh)
    }()

    select {
    case <-ctx.Done():
    case err := <-readerErr:
        if err != nil && !errors.Is(err, context.Canceled) {
            wg.Wait()
            return err
        }
    }

    wg.Wait()
    return ctx.Err()
}

func (c *Consumer) reader(ctx context.Context, out chan<- Message) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }

        msg, err := c.consumer.Read(ctx)
        if err != nil {
            if errors.Is(err, ErrTimeout) {
                continue
            }
            return err
        }

        select {
        case out <- msg:
        case <-ctx.Done():
            return ctx.Err()
        }
    }
}

func (c *Consumer) worker(ctx context.Context, in <-chan Message) {
    for msg := range in {
        select {
        case <-ctx.Done():
            return
        default:
        }

        if err := c.handler.Handle(ctx, msg); err != nil {
            log.Printf("handler error: %v", err)
            continue
        }

        if err := c.consumer.Commit(ctx, msg); err != nil {
            log.Printf("commit error: %v", err)
        }
    }
}
```

### Что происходит

1. **Reader** читает из Kafka и пишет в канал.
2. **N воркеров** читают из канала.
3. Каждый воркер **обрабатывает** и **коммитит**.
4. При `ctx.Done()` — reader закрывает канал.
5. Воркеры **дренируют** оставшиеся сообщения.
6. `wg.Wait()` ждёт завершения.

### Проблема: commit по одному

**Что происходит при 1000 сообщений/сек:**

- 1000 commit'ов/сек.
- Каждый commit — сетевой вызов.
- **Медленно.**

**Решение:** batch commit (см. 35.3).

### Сколько воркеров

**Рекомендации:**

- **CPU-bound:** `GOMAXPROCS`.
- **I/O-bound:** 10–100.
- **С БД:** размер пула БД.

**Пример:** 50 соединений к БД → 50 воркеров.

### Backpressure

**Канал сообщений** даёт **backpressure**:

- Если все воркеры заняты — канал **заполняется**.
- Reader **блокируется**.
- Kafka **накапливает** сообщения.
- **Память стабильна.**

**Размер буфера:** 100–1000.

### 💡 Практика: как построить worker pool для consumer

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Reader** + **N воркеров**.
2. **Канал сообщений** с буфером.
3. **Commit после обработки** в воркере.
4. **Backpressure** через канал.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Размер буфера 100–1000.**
6. **Число воркеров = размер пула БД.**

**❌ НЕ ДЕЛАЙ:**

7. **Не делай буфер 100 000+.**
8. **Не коммить в reader'е.** Только в воркере.

---

## 35.3 Batch commit: производительность

Batch commit — **ключевая оптимизация**.

### Проблема

**1000 commit'ов/сек** — каждое сообщение коммитится отдельно. Медленно.

### Решение: batch commit

**Идея:** накапливаем N обработанных сообщений, потом коммитим **одним вызовом**.

```go
func (c *Consumer) workerWithBatch(ctx context.Context, in <-chan Message) {
    batch := make([]Message, 0, c.batchSize)

    commit := func() {
        if len(batch) == 0 {
            return
        }
        if err := c.consumer.CommitBatch(ctx, batch); err != nil {
            log.Printf("commit error: %v", err)
        }
        batch = batch[:0]
    }
    defer commit()

    for msg := range in {
        select {
        case <-ctx.Done():
            return
        default:
        }

        if err := c.handler.Handle(ctx, msg); err != nil {
            log.Printf("handler error: %v", err)
            continue
        }

        batch = append(batch, msg)
        if len(batch) >= c.batchSize {
            commit()
        }
    }
}
```

### CommitBatch в интерфейсе

```go
type Consumer interface {
    Read(ctx context.Context) (Message, error)
    Commit(ctx context.Context, msg Message) error
    CommitBatch(ctx context.Context, msgs []Message) error
    Close() error
}
```

**Для Kafka:**

```go
func (kc *KafkaConsumer) CommitBatch(ctx context.Context, msgs []Message) error {
    // Коммитим последний offset в каждом партишене
    offsets := map[string]kafka.TopicPartition{}

    for _, msg := range msgs {
        key := msg.Topic
        offsets[key] = kafka.TopicPartition{
            Topic:     &msg.Topic,
            Partition: kafka.PartitionAny,
            Offset:    kafka.Offset(msg.Offset + 1),
        }
    }

    _, err := kc.consumer.CommitOffsets(offsets)
    return err
}
```

**Что даёт:**

- **Один commit** на N сообщений.
- **В 10–100 раз быстрее.**

### Размер batch

**Рекомендации:**

- **Batch 100–1000 сообщений.**
- **Batch по времени** (1 сек) — если поток медленный.

**Комбинация:**

```go
ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop()

for {
    select {
    case msg, ok := <-in:
        if !ok {
            commit()
            return
        }
        batch = append(batch, msg)
        if len(batch) >= c.batchSize {
            commit()
        }
    case <-ticker.C:
        commit()
    case <-ctx.Done():
        return
    }
}
```

### Что происходит при ошибке

**Если commit упал** — сообщения **не коммитятся**. Kafka **переотправит** их.

**Если обработка упала** — не добавляем в batch. **Не коммитим.**

**Результат:** at-least-once. Некоторые сообщения **могут быть обработаны дважды**.

### 💡 Практика: как делать batch commit

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Batch commit** — 100–1000 сообщений.
2. **Batch по времени** — 1 сек.
3. **Commit после обработки.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Комбинация:** размер + время.
5. **Метрики** — batch size.

**❌ НЕ ДЕЛАЙ:**

6. **Не коммить batch до обработки.**
7. **Не забывай commit при shutdown.**

---

## 35.4 Retry и Dead Letter Queue

Не все ошибки **временные**. Разберём retry и DLQ.

### Классификация ошибок

**Временные:**

- Сетевые ошибки.
- БД недоступна.
- 5xx от внешних сервисов.

**Постоянные:**

- Невалидные данные.
- 4xx от внешних сервисов.
- Ошибки схемы.

### Retry для временных

```go
func (c *Consumer) processWithRetry(ctx context.Context, msg Message) error {
    const maxAttempts = 3

    var lastErr error
    for attempt := 1; attempt <= maxAttempts; attempt++ {
        err := c.handler.Handle(ctx, msg)
        if err == nil {
            return nil
        }

        // Permanent — не retry
        var permErr *PermanentError
        if errors.As(err, &permErr) {
            return err
        }

        lastErr = err

        if attempt < maxAttempts {
            backoff := time.Duration(attempt) * 100 * time.Millisecond
            select {
            case <-ctx.Done():
                return ctx.Err()
            case <-time.After(backoff):
            }
        }
    }
    return fmt.Errorf("after %d attempts: %w", maxAttempts, lastErr)
}
```

### Dead Letter Queue

**Если retry не помог** — отправляем в DLQ.

```go
func (c *Consumer) worker(ctx context.Context, in <-chan Message) {
    for msg := range in {
        err := c.processWithRetry(ctx, msg)
        if err != nil {
            if isPermanent(err) {
                // Отправляем в DLQ
                if dlqErr := c.sendToDLQ(ctx, msg, err); dlqErr != nil {
                    log.Printf("DLQ error: %v", dlqErr)
                    continue  // не коммитим — повторим
                }
            } else {
                // Временная ошибка после retry — не коммитим
                log.Printf("retry exhausted: %v", err)
                continue
            }
        }

        // Коммитим только если всё ок или DLQ
        c.consumer.Commit(ctx, msg)
    }
}

func (c *Consumer) sendToDLQ(ctx context.Context, msg Message, err error) error {
    // Отправляем в отдельный топик
    return c.dlqProducer.Send(ctx, DLQMessage{
        OriginalTopic: msg.Topic,
        OriginalID:    msg.ID,
        Body:          msg.Body,
        Error:         err.Error(),
        Timestamp:     time.Now(),
    })
}
```

### Схема

```
Сообщение:
  │
  ├── Обработка ──► успех ──► commit
  │
  ├── Временная ошибка ──► retry (3 попытки)
  │   │
  │   ├── успех ──► commit
  │   └── ошибка ──► не commit (повторим)
  │
  └── Постоянная ошибка ──► DLQ ──► commit
```

### Что происходит в Kafka

**При ошибке без commit:**

- Kafka **не считает** сообщение обработанным.
- Consumer **переподключится** и **перечитает** сообщение.
- **At-least-once.**

**При ошибке с DLQ:**

- Сообщение отправлено в **другой топик**.
- Основной топик коммитится.
- **Original не зацикливается.**

### 💡 Практика: как обрабатывать ошибки

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Retry** для временных.
2. **DLQ** для постоянных.
3. **Commit только после успеха или DLQ.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Backoff** между попытками.
5. **Метрики** — retry, DLQ.

**❌ НЕ ДЕЛАЙ:**

6. **Не коммить при ошибке без DLQ.**
7. **Не retry permanent ошибки.**

---

## 35.5 Graceful shutdown: drain очереди

Drain очереди (Глава 26) — **обязателен**.

### Проблема

**Что если SIGTERM приходит во время обработки?**

- Воркеры **прерываются**.
- Сообщения **не коммитятся**.
- Kafka **переотправит**.
- **Дубликаты.**

### Решение: drain

```go
func (c *Consumer) Run(ctx context.Context) error {
    messagesCh := make(chan Message, c.bufferSize)

    var wg sync.WaitGroup
    for i := 0; i < c.workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            c.worker(ctx, messagesCh)
        }()
    }

    readerDone := make(chan struct{})
    go func() {
        defer close(readerDone)
        defer close(messagesCh)
        c.reader(ctx, messagesCh)
    }()

    <-ctx.Done()
    log.Println("shutdown signal received")

    // Ждём reader
    <-readerDone

    // Воркеры дренируют оставшиеся сообщения
    wg.Wait()

    // Закрываем consumer
    return c.consumer.Close()
}
```

### Что происходит

1. `ctx.Done()` — SIGTERM.
2. **Reader** прекращает чтение, закрывает `messagesCh`.
3. **Воркеры** дренируют канал — обрабатывают оставшиеся.
4. **Commit** — все обработанные.
5. **Consumer.Close** — закрываем соединение.

### Порядок

**Правильный порядок:**

1. **Прекратить** читать из Kafka.
2. **Дождаться** обработки оставшихся.
3. **Commit** все.
4. **Закрыть** соединение.

**Неправильный порядок:**

1. **Закрыть** соединение.
2. **Потерять** сообщения в буфере.

### Схема

```
SIGTERM:
  │
  ├── Reader: прекратить чтение, close(messagesCh)
  │
  ├── Workers: drain messagesCh, обработать, commit
  │
  ├── wg.Wait()
  │
  └── Consumer.Close()
```

### 💡 Практика: как дренировать очередь

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx.Done()`** — сигнал.
2. **Reader закрывает канал.**
3. **Воркеры дренируют.**
4. **Commit после дренажа.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Таймаут** на drain.
6. **Метрики** — сколько успели.

**❌ НЕ ДЕЛАЙ:**

7. **Не закрывай consumer до дренажа.**
8. **Не прерывай воркеры.**

---

## 35.6 Мониторинг: lag, throughput, errors

Мониторинг — **обязателен**.

### Ключевые метрики

| Метрика | Что измеряет |
|:---|:---|
| **Consumer lag** | Отставание от конца очереди |
| **Throughput** | Сообщений/сек |
| **Errors** | Ошибок/сек |
| **Retries** | Retry/сек |
| **DLQ** | DLQ/сек |
| **Batch size** | Средний размер batch |
| **Processing time** | Время обработки |

### Consumer lag

**Что это:** разница между **последним offset в очереди** и **текущим offset consumer'а**.

**Как получить в Kafka:**

```go
// Текущий offset consumer
committed, _ := consumer.Committed(partitions, 1000)

// Последний offset в топике
_, high, _ := consumer.QueryWatermarkOffsets(topic, partition, 1000)

lag := high - committed.Offset
```

**Что означает:**

- **Lag растёт** — consumer **не успевает**.
- **Lag = 0** — consumer **успевает**.
- **Lag падает** — догоняет.

### Через sarama

```go
import "github.com/IBM/sarama"

client, _ := sarama.NewClient(brokers, config)
offset, _ := client.GetOffset(topic, partition, sarama.OffsetNewest)

// lag = offset - committed
```

### Метрики

```go
type Metrics struct {
    Processed      atomic.Int64
    Failed         atomic.Int64
    Retried        atomic.Int64
    DLQ            atomic.Int64
    BatchSize      atomic.Int64
    ProcessingTime atomic.Int64
}

func (m *Metrics) Record(processed, failed, retried, dlq int64, duration time.Duration) {
    m.Processed.Add(processed)
    m.Failed.Add(failed)
    m.Retried.Add(retried)
    m.DLQ.Add(dlq)
    m.ProcessingTime.Add(int64(duration))
}
```

### Экспорт в Prometheus

```go
var (
    consumerLag = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{Name: "kafka_consumer_lag"},
        []string{"topic", "partition"},
    )

    processedTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "messages_processed_total"},
        []string{"topic", "status"},
    )
)

func updateLag(client sarama.Client, topics []string) {
    for _, topic := range topics {
        partitions, _ := client.Partitions(topic)
        for _, partition := range partitions {
            offset, _ := client.GetOffset(topic, partition, sarama.OffsetNewest)
            committed, _ := client.GetOffset(topic, partition, sarama.OffsetOldest)
            lag := offset - committed
            consumerLag.WithLabelValues(topic, strconv.Itoa(int(partition))).Set(float64(lag))
        }
    }
}
```

### Алерты

**Что алертить:**

- **Lag > N** — consumer отстаёт.
- **Errors > N/сек** — много ошибок.
- **DLQ > 0** — есть постоянные ошибки.
- **Processing time растёт** — что-то тормозит.

### 💡 Практика: как мониторить consumer

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Consumer lag** — ключевая метрика.
2. **Throughput** — processed/сек.
3. **Errors, retries, DLQ.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Processing time** — p50, p95, p99.
5. **Batch size.**

**❌ НЕ ДЕЛАЙ:**

6. **Не игнорируй lag.**
7. **Не игнорируй DLQ.**

---

## 35.7 At-least-once vs exactly-once

Разберём **семантики доставки**.

### At-most-once

**Что это:** сообщение обрабатывается **не более одного раза**.

**Как реализуется:** commit **до** обработки.

```go
consumer.Commit(msg)
process(msg)  // ← если упадёт — сообщение потеряно
```

**Плюсы:** нет дубликатов.

**Минусы:** сообщения **могут теряться**.

**Когда использовать:** некритичные данные (метрики, логи).

### At-least-once

**Что это:** сообщение обрабатывается **хотя бы один раз**.

**Как реализуется:** commit **после** обработки.

```go
process(msg)
consumer.Commit(msg)  // ← если упадёт после process — дубликат
```

**Плюсы:** нет потерь.

**Минусы:** возможны **дубликаты**.

**Когда использовать:** большинство случаев (заказы, платежи).

### Exactly-once

**Что это:** сообщение обрабатывается **ровно один раз**.

**Как реализуется:**

- **Идемпотентная обработка.**
- **Транзакции** (Kafka transactions).
- **Дедупликация** (по ID).

**Плюсы:** нет потерь и дубликатов.

**Минусы:** **сложно** и **дорого**.

**Когда использовать:** критичные данные (финансы).

### Как добиться exactly-once

**Способ 1: идемпотентная обработка.**

```go
func (h *Handler) Handle(ctx context.Context, msg Message) error {
    // Проверяем, обрабатывали ли уже
    if h.store.Exists(msg.ID) {
        return nil  // уже обработано
    }

    // Обрабатываем
    if err := h.doHandle(ctx, msg); err != nil {
        return err
    }

    // Записываем ID
    h.store.Mark(msg.ID)
    return nil
}
```

**Что даёт:** повторная обработка **не влияет**.

**Способ 2: дедупликация по ID.**

**Таблица `processed_messages`:**

```sql
CREATE TABLE processed_messages (
    id VARCHAR(255) PRIMARY KEY,
    processed_at TIMESTAMP NOT NULL
);
```

**Обработка:**

```go
tx, _ := db.BeginTx(ctx, nil)
defer tx.Rollback()

// Проверяем, обрабатывали ли
var exists bool
tx.QueryRow("SELECT EXISTS(SELECT 1 FROM processed_messages WHERE id = $1)", msg.ID).Scan(&exists)
if exists {
    return nil  // уже
}

// Обрабатываем
if err := processInTx(tx, msg); err != nil {
    return err
}

// Записываем ID
tx.Exec("INSERT INTO processed_messages (id, processed_at) VALUES ($1, $2)", msg.ID, time.Now())

tx.Commit()
```

**Способ 3: Kafka transactions.**

```go
producer, _ := kafka.NewProducer(&kafka.ConfigMap{
    "transactional.id": "my-tx-id",
})
producer.InitTransactions(ctx)
producer.BeginTransaction()

// Produce + commit offset в одной транзакции
producer.Produce(msg)
consumer.CommitOffsets(offsets)
producer.CommitTransaction(ctx)
```

**Что даёт:** exactly-once в рамках Kafka.

### Сравнение

| Семантика | Потери | Дубликаты | Сложность |
|:---|:---|:---|:---|
| **At-most-once** | Да | Нет | Простая |
| **At-least-once** | Нет | Да | Средняя |
| **Exactly-once** | Нет | Нет | Сложная |

### 💡 Практика: как выбрать семантику

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **At-least-once** — по умолчанию.
2. **Идемпотентная обработка** — для критичных.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Дедупликация** — по ID.
4. **Kafka transactions** — если возможно.

**❌ НЕ ДЕЛАЙ:**

5. **Не используй at-most-once для критичных.**
6. **Не полагайся на exactly-once без идемпотентности.**

---

## 35.8 Полный production-ready consumer

Соберём всё вместе.

### Полный код

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "log/slog"
    "os"
    "os/signal"
    "sync"
    "sync/atomic"
    "syscall"
    "time"

    "github.com/IBM/sarama"
)

type Message struct {
    ID     string
    Body   []byte
    Topic  string
    Offset int64
}

type Handler interface {
    Handle(ctx context.Context, msg Message) error
}

type Metrics struct {
    Processed atomic.Int64
    Failed    atomic.Int64
    Retried   atomic.Int64
    DLQ       atomic.Int64
}

type Consumer struct {
    consumer    sarama.ConsumerGroup
    handler     Handler
    dlqProducer sarama.SyncProducer
    workers     int
    batchSize   int
    metrics     *Metrics
}

func NewConsumer(
    brokers []string,
    groupID string,
    handler Handler,
    workers int,
    batchSize int,
) (*Consumer, error) {
    config := sarama.NewConfig()
    config.Consumer.Return.Errors = true
    config.Consumer.Offsets.Initial = sarama.OffsetOldest
    config.Version = sarama.V2_8_0_0

    group, err := sarama.NewConsumerGroup(brokers, groupID, config)
    if err != nil {
        return nil, err
    }

    producer, err := sarama.NewSyncProducer(brokers, config)
    if err != nil {
        return nil, err
    }

    return &Consumer{
        consumer:    group,
        handler:     handler,
        dlqProducer: producer,
        workers:     workers,
        batchSize:   batchSize,
        metrics:     &Metrics{},
    }, nil
}

func (c *Consumer) Run(ctx context.Context, topics []string) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }

        if err := c.consumer.Consume(ctx, topics, &groupHandler{
            handler:     c.handler,
            dlqProducer: c.dlqProducer,
            workers:     c.workers,
            batchSize:   c.batchSize,
            metrics:     c.metrics,
        }); err != nil {
            slog.Error("consume error", "error", err)
            return err
        }
    }
}

func (c *Consumer) Close() error {
    if err := c.consumer.Close(); err != nil {
        return err    }
    return c.dlqProducer.Close()
}

// groupHandler реализует sarama.ConsumerGroupHandler.
type groupHandler struct {
    handler     Handler
    dlqProducer sarama.SyncProducer
    workers     int
    batchSize   int
    metrics     *Metrics
}

func (h *groupHandler) Setup(session sarama.ConsumerGroupSession) error {
    return nil
}

func (h *groupHandler) Cleanup(session sarama.ConsumerGroupSession) error {
    return nil
}

func (h *groupHandler) ConsumeClaim(session sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
    // Канал сообщений
    messagesCh := make(chan Message, h.workers*2)

    // Воркеры
    var wg sync.WaitGroup
    for i := 0; i < h.workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            h.worker(session.Context(), messagesCh)
        }()
    }

    // Reader
    readerDone := make(chan struct{})
    go func() {
        defer close(readerDone)
        defer close(messagesCh)
        for msg := range claim.Messages() {
            select {
            case messagesCh <- Message{
                ID:     string(msg.Key),
                Body:   msg.Value,
                Topic:  msg.Topic,
                Offset: msg.Offset,
            }:
            case <-session.Context().Done():
                return
            }
        }
    }()

    <-readerDone
    wg.Wait()
    return nil
}

func (h *groupHandler) worker(ctx context.Context, in <-chan Message) {
    batch := make([]Message, 0, h.batchSize)

    commit := func() {
        if len(batch) == 0 {
            return
        }
        // Commit offset в текущей сессии
        // (в реальной реализации — через session.MarkMessage)
        batch = batch[:0]
    }
    defer commit()

    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case msg, ok := <-in:
            if !ok {
                return
            }
            if err := h.processWithRetry(ctx, msg); err != nil {
                h.metrics.Failed.Add(1)
                continue
            }
            h.metrics.Processed.Add(1)
            batch = append(batch, msg)
            if len(batch) >= h.batchSize {
                commit()
            }
        case <-ticker.C:
            commit()
        }
    }
}

func (h *groupHandler) processWithRetry(ctx context.Context, msg Message) error {
    const maxAttempts = 3

    var lastErr error
    for attempt := 1; attempt <= maxAttempts; attempt++ {
        err := h.handler.Handle(ctx, msg)
        if err == nil {
            return nil
        }

        var permErr *PermanentError
        if errors.As(err, &permErr) {
            h.metrics.DLQ.Add(1)
            if dlqErr := h.sendToDLQ(ctx, msg, err); dlqErr != nil {
                return dlqErr
            }
            return nil  // DLQ обработана — считаем успехом
        }

        lastErr = err
        h.metrics.Retried.Add(1)

        if attempt < maxAttempts {
            backoff := time.Duration(attempt) * 100 * time.Millisecond
            select {
            case <-ctx.Done():
                return ctx.Err()
            case <-time.After(backoff):
            }
        }
    }
    return fmt.Errorf("after %d attempts: %w", maxAttempts, lastErr)
}

func (h *groupHandler) sendToDLQ(ctx context.Context, msg Message, err error) error {
    _, _, dlqErr := h.dlqProducer.SendMessage(&sarama.ProducerMessage{
        Topic: msg.Topic + ".dlq",
        Key:   sarama.StringEncoder(msg.ID),
        Value: sarama.ByteEncoder(msg.Body),
        Headers: []sarama.RecordHeader{
            {Key: []byte("error"), Value: []byte(err.Error())},
            {Key: []byte("original_topic"), Value: []byte(msg.Topic)},
        },
    })
    return dlqErr
}

type PermanentError struct {
    Reason string
}

func (e *PermanentError) Error() string {
    return e.Reason
}

// Реальный handler
type OrderHandler struct {
    db *sql.DB
}

func (h *OrderHandler) Handle(ctx context.Context, msg Message) error {
    var order Order
    if err := json.Unmarshal(msg.Body, &order); err != nil {
        return &PermanentError{Reason: "invalid json"}
    }

    if order.ID == "" {
        return &PermanentError{Reason: "missing ID"}
    }

    return h.db.SaveOrder(ctx, order)
}

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    slog.SetDefault(logger)

    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    handler := &OrderHandler{db: db}

    consumer, err := NewConsumer(
        []string{"localhost:9092"},
        "orders-consumer",
        handler,
        50,     // 50 воркеров
        1000,   // batch 1000
    )
    if err != nil {
        slog.Error("consumer error", "error", err)
        os.Exit(1)
    }
    defer consumer.Close()

    if err := consumer.Run(ctx, []string{"orders"}); err != nil && !errors.Is(err, context.Canceled) {
        slog.Error("run error", "error", err)
        os.Exit(1)
    }
}
```

### Что демонстрирует

1. **Worker pool** — 50 воркеров.
2. **Batch commit** — 1000 или 1 секунда.
3. **Retry** — 3 попытки с backoff.
4. **DLQ** — для permanent ошибок.
5. **Graceful shutdown** — drain.
6. **Метрики** — processed, failed, retried, DLQ.
7. **Structured logging** — `slog`.

### 💡 Практика: как строить production consumer

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Worker pool** — 50 воркеров.
2. **Batch commit** — 1000 или 1 сек.
3. **Retry + DLQ.**
4. **Graceful shutdown.**
5. **Метрики.**

**👍 СТОИТ СДЕЛАТЬ:**

6. **Consumer lag** — мониторинг.
7. **Идемпотентность** — для критичных.

**❌ НЕ ДЕЛАЙ:**

8. **Не коммить до обработки.**
9. **Не закрывай consumer до дренажа.**

---

## 35.9 Выводы и типичные ошибки

**Что мы узнали?**

Consumer — синтез паттернов. **Worker pool** — параллельная обработка. **Batch commit** — производительность. **Retry + DLQ** — обработка ошибок. **Graceful shutdown** — drain очереди. **At-least-once** — стандарт. **Exactly-once** — через идемпотентность. **Метрики:** lag, throughput, errors, retries, DLQ.

**Типичные ошибки:**

- ❌ **Auto-commit в Kafka.** Потеря сообщений.
- ❌ **Commit до обработки.** At-most-once.
- ❌ **Нет retry.** Временные ошибки теряются.
- ❌ **Нет DLQ.** Постоянные ошибки зацикливаются.
- ❌ **Нет worker pool.** Один consumer не успевает.
- ❌ **Нет batch commit.** Медленно.
- ❌ **Нет drain при shutdown.** Дубликаты.
- ❌ **Нет метрик.** Не видно проблем.
- ❌ **Не мониторить lag.**
- ❌ **Игнорировать DLQ.**
- ❌ **Не использовать идемпотентность** для критичных.
- ❌ **Не делать graceful shutdown.**

---

## 35.10 Для быстрого повторения

- **Consumer** — синтез паттернов.
- **Worker pool** — параллельная обработка.
- **Batch commit** — 1000 или 1 сек.
- **Retry + DLQ** — обработка ошибок.
- **At-least-once** — стандарт.
- **Exactly-once** — через идемпотентность.
- **Graceful shutdown** — drain очереди.
- **Метрики:** lag, throughput, errors.
- **Consumer lag** — ключевая метрика.
- **`enable.auto.commit: false`** — ручной commit.
- **Commit после обработки.**
- **DLQ для permanent ошибок.**
- **Backoff между retry.**
- **Размер буфера 100–1000.**
- **Число воркеров = размер пула БД.**

---

## 35.11 Вопросы для самопроверки

1. Почему `auto-commit: true` — плохо?
2. Как организовать worker pool в consumer'е?
3. Что такое batch commit?
4. Чем retry отличается от DLQ?
5. Как правильно дренировать очередь?
6. Что такое consumer lag?
7. Чем at-least-once отличается от exactly-once?
8. Как добиться exactly-once?

---

## 35.12 Ответы

### Ответ 1

**`auto-commit: true`** — Kafka коммитит offset **автоматически** каждые 5 секунд (по умолчанию). Если consumer падает между коммитами — сообщения **переотправляются**. Если коммит после обработки — at-least-once. Если **до** — at-most-once (потеря).

**Ручной commit** даёт контроль.

### Ответ 2

**Worker pool:**

```
Reader (Kafka) ──► messagesCh ──► N воркеров
                                       │
                                       ├── process
                                       └── commit
```

Reader читает, воркеры обрабатывают. **Backpressure** через канал.

### Ответ 3

**Batch commit** — коммит **группы** сообщений одним вызовом.

**Что даёт:** меньше сетевых вызовов. В 10–100 раз быстрее.

**Размер:** 100–1000 сообщений или 1 секунда.

### Ответ 4

**Retry** — повтор **временных** ошибок (сеть, БД). **DLQ** — отправка **постоянных** ошибок (невалидные данные) в отдельный топик.

**Retry** → DLQ → commit.

### Ответ 5

**Drain:**

1. `ctx.Done()` — сигнал.
2. **Reader** прекращает чтение, `close(messagesCh)`.
3. **Воркеры** дренируют канал.
4. **Commit** все.
5. **Consumer.Close.**

### Ответ 6

**Consumer lag** — разница между **последним offset в топике** и **текущим offset consumer'а**.

**Что означает:**
- **Lag = 0** — consumer успевает.
- **Lag растёт** — consumer отстаёт.

### Ответ 7

**At-least-once** — сообщение обрабатывается **хотя бы один раз**. Возможны **дубликаты**.

**Exactly-once** — обрабатывается **ровно один раз**. Нет потерь и дубликатов.

**At-least-once** — стандарт. **Exactly-once** — сложно и дорого.

### Ответ 8

**Three ways:**

1. **Идемпотентная обработка** — проверка «уже обработано».
2. **Дедупликация** — таблица `processed_messages`.
3. **Kafka transactions** — атомарный produce + commit.

---

## 35.13 Куда идти дальше?

Мы разобрали consumer — обработку очередей. Теперь мы умеем строить надёжные consumer'ы.

Но остаются **другие сценарии**: ETL, scraper, WebSocket, безопасность.

- **Как построить ETL?** → **Глава 36: ETL.**
- **Как написать scraper?** → **Глава 37: Scraper.**
- **Как работать с WebSocket?** → **Глава 38: WebSocket и streaming.**
- **Как обеспечить безопасность?** → **Глава 39: Безопасность конкурентного кода.**

---

## 35.14 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Consumer** | Обработка очереди | Синтез паттернов |
| **Worker pool** | Параллельная обработка | 50 воркеров |
| **Batch commit** | Производительность | 1000 или 1 сек |
| **Retry** | Временные ошибки | 3 попытки с backoff |
| **DLQ** | Постоянные ошибки | Отдельный топик |
| **At-least-once** | Стандарт | Нет потерь, есть дубликаты |
| **Exactly-once** | Идемпотентность | Сложно |
| **Graceful shutdown** | Drain | Через `ctx.Done()` |
| **Consumer lag** | Ключевая метрика | Разница offset'ов |
| **Throughput** | Сообщений/сек | — |
| **`enable.auto.commit: false`** | Ручной commit | Обязательно |
| **Commit после обработки** | At-least-once | — |
| **Буфер 100–1000** | Backpressure | — |
| **Воркеров = пул БД** | — | — |

📨 **Ключевая идея:** Consumer — синтез паттернов: worker pool (параллелизм), batch commit (производительность), retry + DLQ (ошибки), graceful shutdown (drain), at-least-once (семантика), метрики (lag, throughput, errors). **`enable.auto.commit: false`** и commit после обработки. **Worker pool** — N воркеров читают из канала. **Batch commit** — 1000 или 1 сек. **Retry** 3 попытки с backoff, **DLQ** для permanent. **Drain** при shutdown: reader закрывает канал, воркеры дренируют, commit, close. **Метрики:** consumer lag, processed, failed, retried, DLQ. **At-least-once** — стандарт, **exactly-once** — через идемпотентность.