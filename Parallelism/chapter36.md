# 🔄 Глава 36: ETL — Extract, Transform, Load

**Что вы узнаете:**
- Что такое ETL и какую задачу он решает.
- Как построить **pipeline из стадий**: extract → transform → load.
- Как использовать **fan-out** для медленных стадий.
- Как делать **batch** для load-стадии.
- Как обрабатывать **ошибки** на каждой стадии.
- Как реализовать **backpressure** между стадиями.
- Как **дренировать** pipeline при shutdown.
- Как **мониторить** каждую стадию.
- Как избежать типичных ошибок: потери данных, утечек, дубликатов.

**После прочтения вы сможете:**
- Построить ETL-pipeline с нуля.
- Разбить обработку на стадии.
- Добавить fan-out для медленных стадий.
- Сделать batch для load.
- Обрабатывать ошибки на каждой стадии.
- Корректно завершать pipeline.

---

## Содержание

- [36.0 Пролог: ETL, который встал](#360-пролог-etl-который-встал)
- [36.1 Что такое ETL](#361-что-такое-etl)
- [36.2 Базовая структура pipeline](#362-базовая-структура-pipeline)
- [36.3 Стадия Extract](#363-стадия-extract)
- [36.4 Стадия Transform с fan-out](#364-стадия-transform-с-fan-out)
- [36.5 Стадия Load с batch](#365-стадия-load-с-batch)
- [36.6 Обработка ошибок на каждой стадии](#366-обработка-ошибок-на-каждой-стадии)
- [36.7 Backpressure между стадиями](#367-backpressure-между-стадиями)
- [36.8 Graceful shutdown: drain pipeline](#368-graceful-shutdown-drain-pipeline)
- [36.9 Мониторинг каждой стадии](#369-мониторинг-каждой-стадии)
- [36.10 Полный production-ready ETL](#3610-полный-production-ready-etl)
- [36.11 Выводы и типичные ошибки](#3611-выводы-и-типичные-ошибки)
- [36.12 Для быстрого повторения](#3612-для-быстрого-повторения)
- [36.13 Вопросы для самопроверки](#3613-вопросы-для-самопроверки)
- [36.14 Ответы](#3614-ответы)
- [36.15 Куда идти дальше?](#3615-куда-идти-дальше)
- [36.16 Чек-лист](#3616-чек-лист)

---

## 36.0 Пролог: ETL, который встал

У нас есть сервис, который читает логи из Kafka, парсит их, обогащает данными из БД и пишет в ClickHouse. Классический **ETL** — Extract, Transform, Load.

Пишем наивно:

```go
func main() {
    for {
        msg := kafka.Read()
        parsed := parse(msg)
        enriched := enrich(parsed)
        writeToClickHouse(enriched)
    }
}
```

Работает. Пока не замечаем проблему:

- **Kafka быстрая** — 100 000 сообщений/сек.
- **Parse быстрый** — 1 мкс.
- **Enrich медленный** — 10 мс (запрос в БД).
- **ClickHouse быстрый** — 100 мкс.

**Что происходит:**

- Producer пишет 100 000/сек.
- Consumer обрабатывает 1 / 10 мс = **100/сек**.
- **Пропускная способность — 100/сек**, хотя Kafka даёт 100 000.

**Причина:** обработка **последовательная**. Enrich — bottleneck.

Хочется: **распараллелить** enrich. Но не просто запустить 10 000 горутин — нужно **ограничить** параллелизм. И нужно **дренировать** всё при shutdown. И обработать **ошибки** на каждой стадии.

Это и есть **ETL-pipeline** — конвейер из стадий.

> **Мост к следующим главам:** ETL — синтез pipeline (Глава 11), fan-out (Глава 10), batch (Глава 21), backpressure (Глава 27), worker pool (Глава 13), graceful shutdown (Глава 26). Понимание ETL даёт понимание, **как строить конвейеры данных**.

---

## 36.1 Что такое ETL

**ETL** — Extract, Transform, Load. Классический паттерн обработки данных.

### Три стадии

**1. Extract.**

- Читаем данные из источника.
- Kafka, файлы, БД, API.

**2. Transform.**

- Преобразуем данные.
- Парсинг, обогащение, агрегация.

**3. Load.**

- Пишем данные в приёмник.
- ClickHouse, БД, S3, Kafka.

### Схема

```
┌──────────┐   ┌──────────┐   ┌──────────┐
│  Source  │──▶│Transform │──▶│   Sink   │
│ (Kafka)  │   │ (parse)  │   │(ClickHouse)│
└──────────┘   └──────────┘   └──────────┘
```

### ETL vs ELT

**ETL** — Transform **до** Load.

**ELT** — Load **до** Transform. Данные сначала загружаются в приёмник (например, data warehouse), потом трансформируются там.

**В Go** обычно ETL — трансформация в приложении.

### Ключевые свойства

**1. Многостадийность.**

Каждая стадия — отдельная функция.

**2. Разное число воркеров.**

Каждая стадия может иметь своё число воркеров.

**3. Backpressure.**

Автоматический через каналы.

**4. Graceful shutdown.**

Drain всех стадий.

### Когда использовать ETL

**1. Потоковая обработка.**

- Логи → ClickHouse.
- Заказы → аналитика.
- Метрики → хранилище.

**2. Batch-обработка.**

- Ночные отчёты.
- Импорт данных.

**3. Real-time аналитика.**

- События → агрегаты.

### Когда НЕ использовать ETL

**1. Простая передача.**

Если данные не трансформируются — не нужен ETL.

**2. Одиночные операции.**

Для разовых задач — простой скрипт.

**3. Данные в памяти.**

Если всё влезает в память — batch-скрипт.

### 💡 Практика: как думать об ETL

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **ETL — для потоковой обработки.**
2. **Разбивай на стадии.**
3. **Разное число воркеров.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Backpressure между стадиями.**
5. **Graceful shutdown.**

**❌ НЕ ДЕЛАЙ:**

6. **Не смешивай стадии в одной функции.**
7. **Не забывай про drain.**

---

## 36.2 Базовая структура pipeline

Начнём с простейшего pipeline.

### Типы

```go
type RawMessage struct {
    ID   string
    Body []byte
}

type ParsedMessage struct {
    ID    string
    Value int
}

type EnrichedMessage struct {
    ID       string
    Value    int
    Enriched string
}
```

**Три типа — три стадии.**

### Стадии

**Extract:**

```go
func extract(ctx context.Context) <-chan RawMessage {
    out := make(chan RawMessage)
    go func() {
        defer close(out)
        for {
            select {
            case <-ctx.Done():
                return
            default:
            }
            msg, err := kafka.Read(ctx)
            if err != nil {
                continue
            }
            select {
            case out <- msg:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

**Transform (parse):**

```go
func parse(ctx context.Context, in <-chan RawMessage) <-chan ParsedMessage {
    out := make(chan ParsedMessage)
    go func() {
        defer close(out)
        for msg := range in {
            parsed, err := doParse(msg)
            if err != nil {
                continue
            }
            select {
            case out <- parsed:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

**Transform (enrich):**

```go
func enrich(ctx context.Context, in <-chan ParsedMessage, workers int) <-chan EnrichedMessage {
    // Fan-out
    outputs := make([]<-chan EnrichedMessage, workers)
    for i := 0; i < workers; i++ {
        outputs[i] = enrichWorker(ctx, in)
    }
    return merge(ctx, outputs...)
}
```

**Load:**

```go
func load(ctx context.Context, in <-chan EnrichedMessage) {
    for msg := range in {
        writeToClickHouse(ctx, msg)
    }
}
```

### Сборка

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    raw := extract(ctx)
    parsed := parse(ctx, raw)
    enriched := enrich(ctx, parsed, 10)
    load(ctx, enriched)
}
```

### Схема

```
Kafka ──► extract ──► parse ──► enrich (10 воркеров) ──► load ──► ClickHouse
          (1)         (1)       (fan-out + fan-in)       (1)
```

### Размер буферов

**Рекомендации:**

- **Между стадиями:** 100–1000.
- **Больше для медленных стадий:** 1000.

### 💡 Практика: как построить базовый ETL

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Каждая стадия — функция.**
2. **Размер буферов 100–1000.**
3. **`context` через все стадии.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Типы для каждой стадии.**
5. **Fan-out для медленных стадий.**

**❌ НЕ ДЕЛАЙ:**

6. **Не смешивай стадии.**
7. **Не делай буферы 100 000+.**

---

## 36.3 Стадия Extract

Extract — **источник** данных.

### Из Kafka

```go
func extractKafka(ctx context.Context, brokers, topic, groupID string) (<-chan RawMessage, error) {
    consumer, err := kafka.NewConsumer(&kafka.ConfigMap{
        "bootstrap.servers":  brokers,
        "group.id":           groupID,
        "auto.offset.reset":  "earliest",
        "enable.auto.commit": false,
    })
    if err != nil {
        return nil, err
    }
    consumer.SubscribeTopics([]string{topic}, nil)

    out := make(chan RawMessage, 1000)
    go func() {
        defer close(out)
        for {
            select {
            case <-ctx.Done():
                return
            default:
            }
            msg, err := consumer.ReadMessage(100 * time.Millisecond)
            if err != nil {
                if errors.Is(err, kafka.ErrTimedOut) {
                    continue
                }
                return
            }
            select {
            case out <- RawMessage{ID: string(msg.Key), Body: msg.Value}:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out, nil
}
```

### Из файла

```go
func extractFile(ctx context.Context, path string) (<-chan RawMessage, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, err
    }

    out := make(chan RawMessage, 1000)
    go func() {
        defer close(out)
        defer f.Close()
        scanner := bufio.NewScanner(f)
        for scanner.Scan() {
            select {
            case out <- RawMessage{Body: scanner.Bytes()}:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out, nil
}
```

### Из БД с пагинацией

```go
func extractDB(ctx context.Context, db *sql.DB) <-chan RawMessage {
    out := make(chan RawMessage, 1000)
    go func() {
        defer close(out)
        var lastID int64
        for {
            rows, err := db.QueryContext(ctx,
                "SELECT id, body FROM messages WHERE id > $1 ORDER BY id LIMIT 1000", lastID)
            if err != nil {
                return
            }
            count := 0
            for rows.Next() {
                var id int64
                var body []byte
                rows.Scan(&id, &body)
                select {
                case out <- RawMessage{ID: strconv.FormatInt(id, 10), Body: body}:
                case <-ctx.Done():
                    rows.Close()
                    return
                }
                lastID = id
                count++
            }
            rows.Close()
            if count == 0 {
                return
            }
        }
    }()
    return out
}
```

### Backpressure

**Extract** имеет буфер. Если pipeline не успевает — extract **блокируется** на отправке в канал.

**Что происходит:**

- Kafka **накапливает** сообщения.
- Extract не читает.
- Consumer **не коммитит**.
- При shutdown — drain.

### 💡 Практика: как писать extract

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Буфер 100–1000.**
2. **`select` с `ctx.Done()`.**
3. **`defer close(out)`.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Batch чтение** из БД (пагинация).
5. **Backpressure** через буфер.

**❌ НЕ ДЕЛАЙ:**

6. **Не блокируйся на `ReadMessage(-1)`.**
7. **Не делай буфер 100 000+.**

---

## 36.4 Стадия Transform с fan-out

Transform — **обработка** данных. Медленная стадия — **fan-out**.

### Проблема

**Enrich** — 10 мс на сообщение. Один воркер → **100/сек**.

**Нужно:** 100 воркеров → **10 000/сек**.

### Fan-out + fan-in

```go
func enrich(ctx context.Context, in <-chan ParsedMessage, workers int) <-chan EnrichedMessage {
    // Fan-out
    outputs := make([]<-chan EnrichedMessage, workers)
    for i := 0; i < workers; i++ {
        outputs[i] = enrichWorker(ctx, in)
    }
    // Fan-in
    return merge(ctx, outputs...)
}

func enrichWorker(ctx context.Context, in <-chan ParsedMessage) <-chan EnrichedMessage {
    out := make(chan EnrichedMessage)
    go func() {
        defer close(out)
        for msg := range in {
            enriched, err := doEnrich(ctx, msg)
            if err != nil {
                continue
            }
            select {
            case out <- enriched:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func merge(ctx context.Context, channels ...<-chan EnrichedMessage) <-chan EnrichedMessage {
    out := make(chan EnrichedMessage)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan EnrichedMessage) {
            defer wg.Done()
            for msg := range c {
                select {
                case out <- msg:
                case <-ctx.Done():
                    return
                }
            }
        }(ch)
    }
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```

### Схема

```
ParsedMessage ──► enrichWorker 0 ──┐
                 enrichWorker 1 ──┼──► EnrichedMessage
                 ...              │
                 enrichWorker N ──┘
```

### Сколько воркеров

**Рекомендации:**

- **CPU-bound:** `GOMAXPROCS`.
- **I/O-bound (БД, API):** 10–100.
- **Медленные:** 100+.

**Пример:** 10 мс на запрос к БД → 100 воркеров → 10 000/сек.

### Backpressure

**Fan-out** даёт **backpressure**:

- Если воркеры заняты — `in` заполняется.
- Предыдущая стадия **блокируется**.
- **Память стабильна.**

**Размер буфера:** 100–1000.

### Обработка ошибок

**Каждый воркер** обрабатывает ошибки:

```go
for msg := range in {
    enriched, err := doEnrich(ctx, msg)
    if err != nil {
        // Логируем, но не блокируем pipeline
        log.Printf("enrich error: %v", err)
        continue
    }
    // ...
}
```

**Что делать с ошибками:**

- **Логировать.**
- **Метрики.**
- **DLQ** — если критично.

### 💡 Практика: как писать transform с fan-out

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Fan-out** для медленных стадий.
2. **Fan-in** (merge) для сбора.
3. **Число воркеров** под нагрузку.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Backpressure** через буфер.
5. **Обработка ошибок** в каждом воркере.

**❌ НЕ ДЕЛАЙ:**

6. **Не делай 1 воркер для медленной стадии.**
7. **Не делай 10 000 воркеров.**

---

## 36.5 Стадия Load с batch

Load — **запись** данных. Batch — **ключевая** оптимизация.

### Проблема

**ClickHouse** не любит мелкие INSERT. 100 000 INSERT'ов = **плохо**.

### Решение: batch

```go
func load(ctx context.Context, in <-chan EnrichedMessage, batchSize int, interval time.Duration) error {
    batch := make([]EnrichedMessage, 0, batchSize)

    flush := func() error {
        if len(batch) == 0 {
            return nil
        }
        if err := writeBatch(ctx, batch); err != nil {
            return err
        }
        batch = batch[:0]
        return nil
    }

    ticker := time.NewTicker(interval)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return flush()
        case msg, ok := <-in:
            if !ok {
                return flush()
            }
            batch = append(batch, msg)
            if len(batch) >= batchSize {
                if err := flush(); err != nil {
                    log.Printf("flush error: %v", err)
                }
            }
        case <-ticker.C:
            if err := flush(); err != nil {
                log.Printf("flush error: %v", err)
            }
        }
    }
}
```

### Параметры batch

**Размер:**

- **ClickHouse:** 10 000–100 000 строк.
- **PostgreSQL:** 1000–10 000.
- **Kafka:** 100–1000.

**Интервал:**

- **1 сек** — для real-time.
- **5 сек** — для near-real-time.

### Что происходит при ошибке

**Flush упал:**

- **Batch теряется.**
- **Нужно retry.**

**Решение:** retry с backoff.

```go
func flushWithRetry(ctx context.Context, batch []EnrichedMessage) error {
    var lastErr error
    for attempt := 1; attempt <= 3; attempt++ {
        if err := writeBatch(ctx, batch); err != nil {
            lastErr = err
            time.Sleep(time.Duration(attempt) * 100 * time.Millisecond)
            continue
        }
        return nil
    }
    return lastErr
}
```

### Graceful shutdown

**При `ctx.Done()`** — финальный flush:

```go
case <-ctx.Done():
    if err := flushWithRetry(context.Background(), batch); err != nil {
        log.Printf("final flush error: %v", err)
    }
    return err
```

**Важно:** использовать `context.Background()` для финального flush — чтобы не отменился.

### 💡 Практика: как писать load с batch

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Batch по размеру + времени.**
2. **Flush при shutdown.**
3. **Retry при ошибке.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Размер batch под приёмник.**
5. **Метрики** — размер batch, время flush.

**❌ НЕ ДЕЛАЙ:**

6. **Не коммить до обработки.**
7. **Не забывай финальный flush.**

---

## 36.6 Обработка ошибок на каждой стадии

Ошибки **на каждой стадии** — разная стратегия.

### Extract

**Ошибки:**

- Kafka недоступна.
- Ошибки десериализации.

**Стратегия:**

- **Retry** для сетевых.
- **Skip** для невалидных.

### Transform

**Ошибки:**

- Парсинг.
- Обогащение.
- Бизнес-логика.

**Стратегия:**

- **Skip** для невалидных (логируем).
- **Retry** для временных (БД недоступна).
- **DLQ** для постоянных.

### Load

**Ошибки:**

- ClickHouse недоступна.
- Ошибки схемы.

**Стратегия:**

- **Retry** для временных.
- **DLQ** для постоянных.

### Centralized error handling

**Идея:** все ошибки идут в **один канал**.

```go
type ErrorInfo struct {
    Stage   string
    Message string
    Err     error
    MsgID   string
}

func errorHandler(ctx context.Context, errCh <-chan ErrorInfo) {
    for err := range errCh {
        log.Printf("[%s] %s: %v (msg=%s)", err.Stage, err.Message, err.Err, err.MsgID)
        metrics.Errors.WithLabelValues(err.Stage).Inc()
    }
}
```

**В каждой стадии:**

```go
func enrichWorker(ctx context.Context, in <-chan ParsedMessage, errCh chan<- ErrorInfo) <-chan EnrichedMessage {
    out := make(chan EnrichedMessage)
    go func() {
        defer close(out)
        for msg := range in {
            enriched, err := doEnrich(ctx, msg)
            if err != nil {
                select {
                case errCh <- ErrorInfo{Stage: "enrich", MsgID: msg.ID, Err: err}:
                case <-ctx.Done():
                    return
                }
                continue
            }
            // ...
        }
    }()
    return out
}
```

### DLQ для permanent ошибок

```go
func enrichWorker(ctx context.Context, in <-chan ParsedMessage, dlq chan<- RawMessage) <-chan EnrichedMessage {
    out := make(chan EnrichedMessage)
    go func() {
        defer close(out)
        for msg := range in {
            enriched, err := doEnrich(ctx, msg)
            if err != nil {
                var permErr *PermanentError
                if errors.As(err, &permErr) {
                    // DLQ
                    select {
                    case dlq <- msg:
                    case <-ctx.Done():
                        return
                    }
                }
                continue
            }
            // ...
        }
    }()
    return out
}
```

### Метрики

```go
var (
    errorsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "etl_errors_total"},
        []string{"stage", "type"},
    )
)
```

### 💡 Практика: как обрабатывать ошибки в ETL

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Разные стратегии** для разных стадий.
2. **Centralized error channel.**
3. **DLQ** для permanent.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики** по стадиям.
5. **Логирование** с контекстом.

**❌ НЕ ДЕЛАЙ:**

6. **Не игнорируй ошибки.**
7. **Не блокируй pipeline на ошибках.**

---

## 36.7 Backpressure между стадиями

Backpressure (Глава 27) — **критичен** для ETL.

### Как работает

```
Kafka ──► extract ──► ch(1000) ──► parse ──► ch(100) ──► enrich ──► ch(100) ──► load
          (быстро)                    (быстро)            (медленно)          (быстро)
```

**Что происходит:**

- Enrich медленный — `ch(100)` заполняется.
- Parse **блокируется** на отправке.
- Extract **блокируется**.
- Kafka **накапливает** сообщения.

**Backpressure автоматический.**

### Размер буферов

**Правило:** буфер **сглаживает пики**, но **не маскирует** проблему.

**Рекомендации:**

- **Маленький буфер (10–100)** — строгий backpressure.
- **Средний (100–1000)** — большинство случаев.
- **Большой (1000–10 000)** — сглаживание пиков.
- **Огромный (100 000+)** — не работает.

### Мониторинг

**Что мониторить:**

- **Длина буфера** — % заполнения.
- **Latency** — время прохождения через стадию.
- **Throughput** — сообщений/сек.

**Алерты:**

- **Длина буфера > 90%** — backpressure.
- **Latency растёт** — bottleneck.

### Что делать, если backpressure не справляется

**1. Увеличить число воркеров** в медленной стадии.

**2. Увеличить размер batch** в load.

**3. Добавить rate limiter** для producer.

**4. Drop** — если данные некритичны.

### Схема

```
Медленная стадия:
  ch заполнен → предыдущая стадия блокируется → backpressure

Быстрая стадия:
  ch почти пуст → нет backpressure
```

### 💡 Практика: как настроить backpressure

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Буфер 100–1000.**
2. **Мониторинг длины буфера.**
3. **Алерт при > 90%.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Увеличить воркеры** при bottleneck.
5. **Rate limiter** для producer.

**❌ НЕ ДЕЛАЙ:**

6. **Не делай буфер 100 000+.**
7. **Не игнорируй растущий буфер.**

---

## 36.8 Graceful shutdown: drain pipeline

Graceful shutdown (Глава 26) — **обязателен**.

### Что должно произойти

1. **Extract** — прекратить чтение.
2. **Transform** — дренировать оставшиеся.
3. **Load** — финальный flush.
4. **Все стадии** — закрыть.

### Реализация

```go
func Run(ctx context.Context) error {
    raw := extract(ctx)
    parsed := parse(ctx, raw)
    enriched := enrich(ctx, parsed, 10)

    loadErr := make(chan error, 1)
    go func() {
        loadErr <- load(ctx, enriched, 1000, 1*time.Second)
    }()

    select {
    case <-ctx.Done():
    case err := <-loadErr:
        return err
    }

    // Ждём завершения load (финальный flush)
    if err := <-loadErr; err != nil && !errors.Is(err, context.Canceled) {
        return err
    }
    return nil
}
```

### Порядок

**Правильный:**

1. **Signal** — `ctx.Done()`.
2. **Extract** закрывает `raw`.
3. **Parse** закрывает `parsed`.
4. **Enrich** закрывает `enriched`.
5. **Load** дренирует `enriched`, финальный flush.
6. **Все горутины** завершены.

**Автоматически:**

- `close(raw)` → parse видит EOF → `close(parsed)` → ...
- Load видит EOF → финальный flush.

### Что происходит с сообщениями

**В буферах при shutdown:**

- Сообщения в `raw` → обрабатываются parse.
- В `parsed` → обрабатываются enrich.
- В `enriched` → **финальный flush** в load.

**Ничего не теряется.**

### Таймаут

**Что если pipeline не завершается за 30 секунд?**

```go
shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

// Запускаем Run с shutdownCtx
```

**При таймауте:**

- **Force close.**
- **Логирование.**
- **Дамп горутин.**

### Схема

```
SIGTERM:
  │
  ├── Extract: close(raw)
  │
  ├── Parse: close(parsed)
  │
  ├── Enrich: close(enriched)
  │
  ├── Load: final flush
  │
  └── All goroutines done
```

### 💡 Практика: как дренировать pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx.Done()`** — сигнал.
2. **Close** между стадиями.
3. **Финальный flush** в load.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Таймаут** на shutdown (30 сек).
5. **Метрики** — сколько успели обработать.

**❌ НЕ ДЕЛАЙ:**

6. **Не прерывай стадии.**
7. **Не забывай финальный flush.**

---

## 36.9 Мониторинг каждой стадии

Мониторинг — **обязателен**.

### Ключевые метрики

| Метрика | Что измеряет |
|:---|:---|
| **Stage throughput** | Сообщений/сек на стадии |
| **Stage latency** | Время обработки |
| **Buffer length** | Заполнение буфера |
| **Errors** | Ошибок/сек |
| **DLQ** | DLQ/сек |
| **Batch size** | Средний размер batch |

### Метрики по стадиям

```go
type StageMetrics struct {
    Processed atomic.Int64
    Failed    atomic.Int64
    Dropped   atomic.Int64
    TotalTime atomic.Int64
}

type PipelineMetrics struct {
    Extract   StageMetrics
    Parse     StageMetrics
    Enrich    StageMetrics
    Load      StageMetrics
}
```

**В каждой стадии:**

```go
func parse(ctx context.Context, in <-chan RawMessage, metrics *StageMetrics) <-chan ParsedMessage {
    out := make(chan ParsedMessage)
    go func() {
        defer close(out)
        for msg := range in {
            start := time.Now()
            parsed, err := doParse(msg)
            if err != nil {
                metrics.Failed.Add(1)
                continue
            }
            metrics.Processed.Add(1)
            metrics.TotalTime.Add(int64(time.Since(start)))
            select {
            case out <- parsed:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

### Prometheus

```go
var (
    stageProcessed = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "etl_stage_processed_total"},
        []string{"stage"},
    )
    stageDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{Name: "etl_stage_duration_seconds"},
        []string{"stage"},
    )
    bufferLength = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{Name: "etl_buffer_length"},
        []string{"stage"},
    )
)
```

### Алерты

**Что алертить:**

- **Buffer length > 90%** — backpressure.
- **Errors > N/сек** — много ошибок.
- **DLQ > 0** — permanent ошибки.
- **Latency растёт** — bottleneck.

### Схема

```
Extract ──► Parse ──► Enrich ──► Load
  │          │         │         │
  └──────────┴─────────┴─────────┘
             │
             ▼
        Метрики на каждой стадии:
          - processed
          - failed
          - latency
          - buffer length
```

### 💡 Практика: как мониторить ETL

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики** на каждой стадии.
2. **Buffer length** — backpressure.
3. **Latency** — bottleneck.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Алерты** на > 90% и errors.
5. **Grafana** — визуализация.

**❌ НЕ ДЕЛАЙ:**

6. **Не игнорируй backpressure.**
7. **Не игнорируй DLQ.**

---

## 36.10 Полный production-ready ETL

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
)

type RawMessage struct {
    ID   string
    Body []byte
}

type ParsedMessage struct {
    ID    string
    Value int
}

type EnrichedMessage struct {
    ID       string
    Value    int
    Enriched string
}

type StageMetrics struct {
    Processed atomic.Int64
    Failed    atomic.Int64
    TotalTime atomic.Int64
}

type Pipeline struct {
    metrics struct {
        Extract StageMetrics
        Parse   StageMetrics
        Enrich  StageMetrics
        Load    StageMetrics
    }
    buffers struct {
        Raw      int
        Parsed   int
        Enriched int
    }
    workers      int
    batchSize    int
    batchTimeout time.Duration
}

func NewPipeline(workers, batchSize int, batchTimeout time.Duration) *Pipeline {
    return &Pipeline{
        workers:      workers,
        batchSize:    batchSize,
        batchTimeout: batchTimeout,
        buffers: struct {
            Raw      int
            Parsed   int
            Enriched int
        }{
            Raw:      1000,
            Parsed:   1000,
            Enriched: 1000,
        },
    }
}

func (p *Pipeline) Run(ctx context.Context) error {
    raw := p.extract(ctx)
    parsed := p.parse(ctx, raw)
    enriched := p.enrich(ctx, parsed)

    return p.load(ctx, enriched)
}

func (p *Pipeline) extract(ctx context.Context) <-chan RawMessage {
    out := make(chan RawMessage, p.buffers.Raw)
    go func() {
        defer close(out)
        for i := 0; i < 10000; i++ {
            msg := RawMessage{ID: fmt.Sprintf("msg-%d", i), Body: []byte("hello")}
            select {
            case out <- msg:
                p.metrics.Extract.Processed.Add(1)
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func (p *Pipeline) parse(ctx context.Context, in <-chan RawMessage) <-chan ParsedMessage {
    out := make(chan ParsedMessage, p.buffers.Parsed)
    go func() {
        defer close(out)
        for msg := range in {
            start := time.Now()
            parsed := ParsedMessage{ID: msg.ID, Value: len(msg.Body)}
            p.metrics.Parse.Processed.Add(1)
            p.metrics.Parse.TotalTime.Add(int64(time.Since(start)))
            select {
            case out <- parsed:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func (p *Pipeline) enrich(ctx context.Context, in <-chan ParsedMessage) <-chan EnrichedMessage {
    outputs := make([]<-chan EnrichedMessage, p.workers)
    for i := 0; i < p.workers; i++ {
        outputs[i] = p.enrichWorker(ctx, in)
    }
    return p.merge(ctx, outputs...)
}

func (p *Pipeline) enrichWorker(ctx context.Context, in <-chan ParsedMessage) <-chan EnrichedMessage {
    out := make(chan EnrichedMessage)
    go func() {
        defer close(out)
        for msg := range in {
            start := time.Now()
            select {
            case <-ctx.Done():
                return
            case <-time.After(1 * time.Millisecond):
            }
            enriched := EnrichedMessage{ID: msg.ID, Value: msg.Value, Enriched: "enriched"}
            p.metrics.Enrich.Processed.Add(1)
            p.metrics.Enrich.TotalTime.Add(int64(time.Since(start)))
            select {
            case out <- enriched:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func (p *Pipeline) merge(ctx context.Context, channels ...<-chan EnrichedMessage) <-chan EnrichedMessage {
    out := make(chan EnrichedMessage, p.buffers.Enriched)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan EnrichedMessage) {
            defer wg.Done()
            for msg := range c {
                select {
                case out <- msg:
                case <-ctx.Done():
                    return
                }
            }
        }(ch)
    }
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}

func (p *Pipeline) load(ctx context.Context, in <-chan EnrichedMessage) error {
    batch := make([]EnrichedMessage, 0, p.batchSize)

    flush := func(flushCtx context.Context) error {
        if len(batch) == 0 {
            return nil
        }
        start := time.Now()
        select {
        case <-flushCtx.Done():
            return flushCtx.Err()
        case <-time.After(1 * time.Millisecond):
        }
        p.metrics.Load.Processed.Add(int64(len(batch)))
        p.metrics.Load.TotalTime.Add(int64(time.Since(start)))
        slog.Info("flush", "batch_size", len(batch))
        batch = batch[:0]
        return nil
    }

    ticker := time.NewTicker(p.batchTimeout)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            finalCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
            defer cancel()
            return flush(finalCtx)
        case msg, ok := <-in:
            if !ok {
                finalCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
                defer cancel()
                return flush(finalCtx)
            }
            batch = append(batch, msg)
            if len(batch) >= p.batchSize {
                if err := flush(ctx); err != nil {
                    slog.Error("flush error", "error", err)
                }
            }
        case <-ticker.C:
            if err := flush(ctx); err != nil {
                slog.Error("flush error", "error", err)
            }
        }
    }
}

func (p *Pipeline) PrintMetrics() {
    slog.Info("metrics",
        "extract", p.metrics.Extract.Processed.Load(),
        "parse", p.metrics.Parse.Processed.Load(),
        "enrich", p.metrics.Enrich.Processed.Load(),
        "load", p.metrics.Load.Processed.Load(),
    )
}

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    slog.SetDefault(logger)

    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    pipeline := NewPipeline(10, 1000, 1*time.Second)

    if err := pipeline.Run(ctx); err != nil && !errors.Is(err, context.Canceled) {
        slog.Error("run error", "error", err)
        os.Exit(1)
    }

    pipeline.PrintMetrics()
    slog.Info("done")
}
```

### Что демонстрирует

1. **Extract** — источник (симуляция Kafka).
2. **Parse** — быстрое преобразование.
3. **Enrich** — fan-out на 10 воркеров.
4. **Load** — batch 1000 или 1 сек.
5. **Graceful shutdown** — финальный flush.
6. **Метрики** на каждой стадии.
7. **Backpressure** через буферы 1000.
8. **Structured logging** — `slog`.

### 💡 Практика: как строить production ETL

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Стадии** — extract, transform, load.
2. **Fan-out** для медленных.
3. **Batch** для load.
4. **Backpressure** через буфер.
5. **Graceful shutdown** с финальным flush.

**👍 СТОИТ СДЕЛАТЬ:**

6. **Метрики** на каждой стадии.
7. **Retry + DLQ.**

**❌ НЕ ДЕЛАЙ:**

8. **Не смешивай стадии.**
9. **Не забывай drain.**

---

## 36.11 Выводы и типичные ошибки

**Что мы узнали?**

ETL — pipeline из стадий: extract, transform, load. **Fan-out** для медленных стадий. **Batch** для load. **Backpressure** через каналы. **Graceful shutdown** — drain. **Метрики** на каждой стадии. **Обработка ошибок** — retry, skip, DLQ.

**Типичные ошибки:**

- ❌ **Последовательная обработка.** Медленно.
- ❌ **Один воркер для медленной стадии.** Bottleneck.
- ❌ **Нет batch для load.** Мелкие записи.
- ❌ **Нет backpressure.** OOM.
- ❌ **Нет graceful shutdown.** Потеря данных.
- ❌ **Нет финального flush.**
- ❌ **Нет метрик.** Не видно bottleneck.
- ❌ **Игнорировать ошибки.**
- ❌ **Не использовать fan-out.**
- ❌ **Огромные буферы.**
- ❌ **Не мониторить buffer length.**
- ❌ **Не различать permanent и temporary ошибки.**

---

## 36.12 Для быстрого повторения

- **ETL** — Extract, Transform, Load.
- **Стадии** — функции `func(ctx, in) <-chan Out`.
- **Fan-out** — для медленных стадий.
- **Batch** — для load.
- **Backpressure** — через каналы.
- **Graceful shutdown** — drain.
- **Финальный flush** — обязательно.
- **Метрики** на каждой стадии.
- **Buffer length** — backpressure.
- **Latency** — bottleneck.
- **Retry** — временные ошибки.
- **DLQ** — постоянные.
- **Размер буфера 100–1000.**
- **Fan-out 10–100 воркеров.**
- **Batch 1000 или 1 сек.**

---

## 36.13 Вопросы для самопроверки

1. Что такое ETL? Назови три стадии.
2. Почему последовательная обработка — плохо?
3. Что такое fan-out? Зачем?
4. Что такое batch? Зачем?
5. Как работает backpressure между стадиями?
6. Как правильно дренировать pipeline?
7. Как обрабатывать ошибки на разных стадиях?
8. Что мониторить в ETL?

---

## 36.14 Ответы

### Ответ 1

**ETL** — Extract, Transform, Load.

1. **Extract** — читаем из источника (Kafka, файлы, БД).
2. **Transform** — преобразуем (парсинг, обогащение).
3. **Load** — пишем в приёмник (ClickHouse, БД).

### Ответ 2

**Последовательная обработка** — медленная. Если **enrich** 10 мс, то пропускная способность **100/сек**, хотя Kafka даёт 100 000/сек.

**Решение:** fan-out.

### Ответ 3

**Fan-out** — распределение работы между N воркерами.

**Зачем:** ускорить медленные стадии. 10 воркеров × 100/сек = 1000/сек.

### Ответ 4

**Batch** — группировка операций.

**Зачем:** ClickHouse, Kafka, БД не любят мелкие операции. Batch 1000 = в 1000 раз меньше операций.

### Ответ 5

**Backpressure** между стадиями — **автоматически** через каналы.

- Медленная стадия → канал заполняется.
- Предыдущая стадия → блокируется.
- Source → накапливает.

**Память стабильна.**

### Ответ 6

**Drain pipeline:**

1. `ctx.Done()`.
2. Extract → `close(raw)`.
3. Parse → `close(parsed)`.
4. Enrich → `close(enriched)`.
5. Load → **финальный flush**.

**Ничего не теряется.**

### Ответ 7

**Extract:** retry для сетевых, skip для невалидных.

**Transform:** skip для невалидных, retry для временных, DLQ для постоянных.

**Load:** retry для временных, DLQ для постоянных.

**Centralized:** все ошибки → один канал → логирование, метрики.

### Ответ 8

**Мониторинг ETL:**

- **Stage throughput** — сообщений/сек.
- **Stage latency** — время обработки.
- **Buffer length** — % заполнения.
- **Errors** — ошибок/сек.
- **DLQ** — DLQ/сек.
- **Batch size** — средний размер.

---

## 36.15 Куда идти дальше?

Мы разобрали ETL — конвейеры данных. Теперь мы умеем строить production-ready ETL.

Но остаются **другие сценарии**: scraper, WebSocket, безопасность.

- **Как написать scraper?** → **Глава 37: Scraper.**
- **Как работать с WebSocket?** → **Глава 38: WebSocket и streaming.**
- **Как обеспечить безопасность?** → **Глава 39: Безопасность конкурентного кода.**

---

## 36.16 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **ETL** | Extract, Transform, Load | Конвейер данных |
| **Extract** | Источник | Kafka, файлы, БД |
| **Transform** | Обработка | Parse, enrich |
| **Load** | Приёмник | ClickHouse, БД |
| **Fan-out** | Медленная стадия | 10–100 воркеров |
| **Batch** | Load | 1000 или 1 сек |
| **Backpressure** | Через каналы | Буфер 100–1000 |
| **Graceful shutdown** | Drain | Через `ctx.Done()` |
| **Финальный flush** | Обязательно | — |
| **Метрики** | На каждой стадии | processed, latency, buffer |
| **Retry** | Временные ошибки | 3 попытки |
| **DLQ** | Постоянные ошибки | Отдельный канал |
| **Buffer length** | Backpressure | < 90% |
| **Latency** | Bottleneck | p50, p95, p99 |

🔄 **Ключевая идея:** ETL — конвейер из стадий: extract → transform → load. **Fan-out** для медленных стадий (10–100 воркеров). **Batch** для load (1000 или 1 сек). **Backpressure** автоматический через каналы (буфер 100–1000). **Graceful shutdown** — drain, финальный flush. **Метрики** на каждой стадии (processed, latency, buffer length). **Retry** для временных, **DLQ** для постоянных. **Buffer length > 90%** — backpressure. **Latency растёт** — bottleneck. Не смешивай стадии. Не забывай финальный flush.