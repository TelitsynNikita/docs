# 🌍 Глава 14: Реальные сценарии — HTTP, очереди, ETL, scraping

**Что вы узнаете:**
- Как построить **production-ready HTTP-сервер** с ограничением параллелизма.
- Как обрабатывать **очереди** (Kafka, NATS, RabbitMQ) с worker pool.
- Как построить **ETL-пайплайн** с backpressure и batch.
- Как написать **scraper** с rate limiter и circuit breaker.
- Как комбинировать все паттерны в **одном сервисе**.
- Как **диагностировать** production-проблемы.
- Как **тестировать** реальные сценарии.
- Как **не повторять** анти-паттерны из Главы 13.

**После прочтения вы сможете:**
- Построить HTTP-сервер с graceful shutdown и rate limiting.
- Обработать очередь с worker pool и drain.
- Построить ETL с backpressure и batch.
- Написать scraper с rate limiter и circuit breaker.
- Комбинировать паттерны в одном сервисе.
- Диагностировать утечки, contention, backpressure.
- Писать тесты для реальных сценариев.

---

## Содержание

- [14.0 Пролог: сервис, который выжил в production](#140-пролог-сервис-который-выжил-в-production)
- [14.1 HTTP-сервер с ограничением параллелизма](#141-http-сервер-с-ограничением-параллелизма)
- [14.2 Очереди: Kafka, NATS, RabbitMQ](#142-очереди-kafka-nats-rabbitmq)
- [14.3 ETL-пайплайн с batch и backpressure](#143-etl-пайплайн-с-batch-и-backpressure)
- [14.4 Scraper с rate limiter и circuit breaker](#144-scraper-с-rate-limiter-и-circuit-breaker)
- [14.5 Комбинированный сервис: всё вместе](#145-комбинированный-сервис-всё-вместе)
- [14.6 Диагностика production-проблем](#146-диагностика-production-проблем)
- [14.7 Тестирование реальных сценариев](#147-тестирование-реальных-сценариев)
- [14.8 Выводы и типичные ошибки](#148-выводы-и-типичные-ошибки)
- [14.9 Для быстрого повторения](#149-для-быстрого-повторения)
- [14.10 Вопросы для самопроверки](#1410-вопросы-для-самопроверки)
- [14.11 Ответы](#1411-ответы)
- [14.12 Куда идти дальше?](#1412-куда-идти-дальше)
- [14.13 Чек-лист](#1413-чек-лист)

---

## 14.0 Пролог: сервис, который выжил в production

Ты пишешь production-ready сервис. Требования:

- **HTTP API** для клиентов.
- **Очередь** (Kafka) для событий.
- **ETL** для обработки данных.
- **Scraper** для внешних источников.
- **Graceful shutdown** по `SIGTERM`.
- **Метрики** в Prometheus.
- **Логи** в stdout.
- **Traces** в Jaeger.

**Что нужно:**

- HTTP-сервер с ограничением параллелизма.
- Worker pool для очереди.
- Pipeline для ETL.
- Rate limiter и circuit breaker для scraper.
- Graceful shutdown для всего.
- Метрики, логи, трейсы.

**Это глава о том, как собрать всё вместе.**

> **Важный мост:** эта глава — **синтез** всех предыдущих. HTTP-сервер (Глава 11), очереди (Глава 7), ETL (Глава 8), scraping (Глава 9). Все паттерны комбинируются.

---

## 14.1 HTTP-сервер с ограничением параллелизма

Построим **production-ready HTTP-сервер**.

### Требования

1. **Graceful shutdown** по `SIGTERM`/`SIGINT`.
2. **Ограничение параллелизма** — не более N одновременных запросов.
3. **Таймауты** на чтение, запись, idle.
4. **Rate limiting** на клиента.
5. **Метрики** (Prometheus).
6. **Structured logging** (`slog`).
7. **Health checks** (`/healthz`, `/readyz`).

### Шаг 1: структура

```go
type Server struct {
    httpSrv *http.Server
    sem     chan struct{}      // семафор для ограничения
    limiter *rate.Limiter      // rate limiter
}
```

### Шаг 2: создание сервера

```go
func NewServer(addr string, maxConcurrent int, rps int) *Server {
    s := &Server{
        sem:     make(chan struct{}, maxConcurrent),
        limiter: rate.NewLimiter(rate.Limit(rps), rps*2),
    }

    mux := http.NewServeMux()
    mux.HandleFunc("/", s.handleRoot)
    mux.HandleFunc("/healthz", s.handleHealth)
    mux.HandleFunc("/readyz", s.handleReady)
    mux.Handle("/metrics", promhttp.Handler())

    s.httpSrv = &http.Server{
        Addr:         addr,
        Handler:      s.middleware(mux),
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    return s
}
```

### Шаг 3: middleware

```go
func (s *Server) middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Rate limiting
        if !s.limiter.Allow() {
            http.Error(w, "too many requests", http.StatusTooManyRequests)
            return
        }

        // Ограничение параллелизма
        select {
        case s.sem <- struct{}{}:
            defer func() { <-s.sem }()
        case <-r.Context().Done():
            return
        case <-time.After(1 * time.Second):
            http.Error(w, "server busy", http.StatusServiceUnavailable)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

### Шаг 4: обработчики

```go
func (s *Server) handleRoot(w http.ResponseWriter, r *http.Request) {
    select {
    case <-r.Context().Done():
        return
    case <-time.After(100 * time.Millisecond):
        fmt.Fprintln(w, "hello")
    }
}

func (s *Server) handleHealth(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    fmt.Fprintln(w, "ok")
}

func (s *Server) handleReady(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    fmt.Fprintln(w, "ready")
}
```

### Шаг 5: запуск и shutdown

```go
func (s *Server) Run(ctx context.Context) error {
    errCh := make(chan error, 1)
    go func() {
        slog.Info("server starting", "addr", s.httpSrv.Addr)
        if err := s.httpSrv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            errCh <- err
        }
        close(errCh)
    }()

    select {
    case <-ctx.Done():
        slog.Info("shutdown signal received")
    case err := <-errCh:
        return err
    }

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
    defer cancel()

    if err := s.httpSrv.Shutdown(shutdownCtx); err != nil {
        slog.Error("shutdown error", "error", err)
        s.httpSrv.Close()
        return err
    }
    return nil
}
```

### Шаг 6: main

```go
func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    slog.SetDefault(logger)

    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    srv := NewServer(":8080", 100, 1000)

    if err := srv.Run(ctx); err != nil {
        slog.Error("run failed", "error", err)
        os.Exit(1)
    }
    slog.Info("shutdown complete")
}
```

### Что демонстрирует

1. **Семафор** — не более 100 одновременных запросов.
2. **Rate limiter** — не более 1000 запросов/сек.
3. **Graceful shutdown** — по сигналу.
4. **Health checks** — `/healthz`, `/readyz`.
5. **Метрики** — `/metrics`.
6. **Structured logging** — `slog`.

### Аннотация сложности

| Компонент | Time | Space |
|:---|:---|:---|
| Middleware | ~100-200 нс | 0 |
| Семафор | ~30-70 нс | 0 |
| Rate limiter | ~50-100 нс | 0 |
| Обработчик | зависит | зависит |
| Shutdown | 0-25 сек | 0 |

### 💡 Практика: как строить production HTTP

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Graceful shutdown** — по сигналу.
2. **Семафор** — для ограничения параллелизма.
3. **Rate limiter** — для защиты от DDoS.
4. **Таймауты** — Read, Write, Idle.
5. **Health checks.**

**👍 СТОИТ СДЕЛАТЬ:**

6. **Метрики** — Prometheus.
7. **Structured logging** — `slog`.
8. **Middleware** — для cross-cutting concerns.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

9. **Tracing** — OpenTelemetry.

**❌ НЕ ДЕЛАЙ:**

10. **Не блокируй хэндлер** — `select` + `default`.
11. **Не забывай `ctx.Done()`** в долгих обработчиках.
12. **Не используй `os.Exit`** в Run.

### Ключевые выводы подглавы 14.1

- **Семафор** — ограничение параллелизма.
- **Rate limiter** — защита от DDoS.
- **Graceful shutdown** — по сигналу.
- **Health checks** — `/healthz`, `/readyz`.
- **Метрики, логи** — обязательны.

---

## 14.2 Очереди: Kafka, NATS, RabbitMQ

Построим **consumer** для очереди.

### Требования

1. **Worker pool** — N воркеров.
2. **Graceful shutdown** — drain перед закрытием.
3. **Batch commit** — для производительности.
4. **Retry** — для временных ошибок.
5. **Dead letter queue** — для постоянных ошибок.
6. **Метрики** — сколько обработано, сколько упало.

### Шаг 1: структура

```go
type Consumer struct {
    consumer  MessageConsumer
    workers   int
    batchSize int
    metrics   *Metrics
}

type Message struct {
    ID   string
    Body []byte
}

type MessageConsumer interface {
    Read(ctx context.Context) (Message, error)
    Commit(ctx context.Context, msgs []Message) error
    Close() error
}
```

### Шаг 2: запуск

```go
func (c *Consumer) Run(ctx context.Context) error {
    messagesCh := make(chan Message, c.batchSize*2)

    var wg sync.WaitGroup
    for i := 0; i < c.workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            c.worker(ctx, messagesCh)
        }()
    }

    // Reader
    readerDone := make(chan struct{})
    go func() {
        defer close(readerDone)
        defer close(messagesCh)
        c.reader(ctx, messagesCh)
    }()

    <-ctx.Done()
    slog.Info("shutdown signal received, draining")

    <-readerDone
    wg.Wait()

    if err := c.consumer.Close(); err != nil {
        slog.Error("close error", "error", err)
        return err
    }
    return nil
}
```

### Шаг 3: reader

```go
func (c *Consumer) reader(ctx context.Context, out chan<- Message) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
        }

        msg, err := c.consumer.Read(ctx)
        if err != nil {
            if errors.Is(err, context.Canceled) {
                return
            }
            slog.Error("read error", "error", err)
            time.Sleep(100 * time.Millisecond)
            continue
        }

        select {
        case out <- msg:
            c.metrics.Read.Add(1)
        case <-ctx.Done():
            return
        }
    }
}
```

### Шаг 4: worker

```go
func (c *Consumer) worker(ctx context.Context, in <-chan Message) {
    batch := make([]Message, 0, c.batchSize)

    commit := func() {
        if len(batch) == 0 {
            return
        }
        if err := c.consumer.Commit(ctx, batch); err != nil {
            slog.Error("commit error", "error", err, "batch_size", len(batch))
        }
        batch = batch[:0]
    }
    defer commit()

    for msg := range in {
        if err := c.process(ctx, msg); err != nil {
            c.metrics.Failed.Add(1)
            slog.Error("process error", "error", err, "msg_id", msg.ID)

            if isPermanent(err) {
                c.sendToDLQ(ctx, msg)
                batch = append(batch, msg)  // коммитим, чтобы не зацикливаться
            }
            // временные ошибки — не коммитим, будет retry
            continue
        }

        c.metrics.Processed.Add(1)
        batch = append(batch, msg)

        if len(batch) >= c.batchSize {
            commit()
        }
    }
}
```

### Шаг 5: обработка

```go
func (c *Consumer) process(ctx context.Context, msg Message) error {
    // Обработка с таймаутом
    processCtx, cancel := context.WithTimeout(ctx, 10*time.Second)
    defer cancel()

    return doProcess(processCtx, msg)
}

func isPermanent(err error) bool {
    var permErr *PermanentError
    return errors.As(err, &permErr)
}

func (c *Consumer) sendToDLQ(ctx context.Context, msg Message) {
    // Отправка в dead letter queue
    slog.Warn("sending to DLQ", "msg_id", msg.ID)
}
```

### Шаг 6: main

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    consumer := NewConsumer(kafkaConsumer, 10, 100)

    if err := consumer.Run(ctx); err != nil {
        slog.Error("run failed", "error", err)
        os.Exit(1)
    }
}
```

### Что демонстрирует

1. **Worker pool** — 10 воркеров.
2. **Batch commit** — коммит каждые 100 сообщений.
3. **Graceful drain** — при сигнале.
4. **Retry** — временные ошибки.
5. **DLQ** — постоянные ошибки.
6. **Метрики** — сколько обработано.

### Аннотация сложности

| Компонент | Time | Space |
|:---|:---|:---|
| Read | ~1-10 мс | 0 |
| Process | зависит | зависит |
| Batch commit | ~10-100 мс | 0 |
| Drain | 0-30 сек | 0 |

### 💡 Практика: как строить consumer

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Worker pool** — N воркеров.
2. **Batch commit** — для производительности.
3. **Graceful drain** — при сигнале.
4. **Retry** — для временных ошибок.
5. **DLQ** — для постоянных.

**👍 СТОИТ СДЕЛАТЬ:**

6. **Метрики** — read, processed, failed.
7. **Таймауты** на обработку.

**❌ НЕ ДЕЛАЙ:**

8. **Не коммить до обработки.**
9. **Не зацикливайся на плохих сообщениях.**
10. **Не забывай drain.**

### Ключевые выводы подглавы 14.2

- **Worker pool** — для параллелизма.
- **Batch commit** — производительность.
- **Graceful drain** — при сигнале.
- **Retry + DLQ** — для ошибок.
- **Метрики** — обязательны.

---

## 14.3 ETL-пайплайн с batch и backpressure

Построим **ETL-пайплайн**.

### Требования

1. **Source** — читает данные.
2. **Transform** — обрабатывает.
3. **Load** — пишет.
4. **Batch** — группирует.
5. **Backpressure** — между стадиями.
6. **Метрики** — на каждой стадии.

### Шаг 1: структура

```go
type Pipeline struct {
    source    Source
    transform Transformer
    sink      Sink
    batchSize int
    metrics   *Metrics
}

type Source interface {
    Read(ctx context.Context) (Record, error)
    Close() error
}

type Transformer interface {
    Transform(ctx context.Context, rec Record) (Record, error)
}

type Sink interface {
    WriteBatch(ctx context.Context, batch []Record) error
    Close() error
}
```

### Шаг 2: стадия Source

```go
func (p *Pipeline) source(ctx context.Context) <-chan Record {
    out := make(chan Record, 100)

    go func() {
        defer close(out)
        for {
            rec, err := p.source.Read(ctx)
            if err != nil {
                if errors.Is(err, io.EOF) || errors.Is(err, context.Canceled) {
                    return
                }
                slog.Error("source read error", "error", err)
                continue
            }

            select {
            case out <- rec:
                p.metrics.SourceCount.Add(1)
            case <-ctx.Done():
                return
            }
        }
    }()

    return out
}
```

### Шаг 3: стадия Transform (fan-out)

```go
func (p *Pipeline) transform(ctx context.Context, in <-chan Record, workers int) <-chan Record {
    out := make(chan Record, 100)

    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for rec := range in {
                transformed, err := p.transformOne(ctx, rec)
                if err != nil {
                    p.metrics.TransformFailed.Add(1)
                    continue
                }

                select {
                case out <- transformed:
                    p.metrics.TransformCount.Add(1)
                case <-ctx.Done():
                    return
                }
            }
        }()
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}

func (p *Pipeline) transformOne(ctx context.Context, rec Record) (Record, error) {
    transformCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    return p.transform.Transform(transformCtx, rec)
}
```

### Шаг 4: стадия Sink (batch)

```go
func (p *Pipeline) sink(ctx context.Context, in <-chan Record) error {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    batch := make([]Record, 0, p.batchSize)

    flush := func() error {
        if len(batch) == 0 {
            return nil
        }
        if err := p.sink.WriteBatch(ctx, batch); err != nil {
            p.metrics.SinkFailed.Add(int64(len(batch)))
            return err
        }
        p.metrics.SinkCount.Add(int64(len(batch)))
        batch = batch[:0]
        return nil
    }

    for {
        select {
        case <-ctx.Done():
            return flush()
        case <-ticker.C:
            if err := flush(); err != nil {
                slog.Error("flush error", "error", err)
            }
        case rec, ok := <-in:
            if !ok {
                return flush()
            }
            batch = append(batch, rec)
            if len(batch) >= p.batchSize {
                if err := flush(); err != nil {
                    slog.Error("flush error", "error", err)
                }
            }
        }
    }
}
```

### Шаг 5: сборка

```go
func (p *Pipeline) Run(ctx context.Context) error {
    sourceCh := p.source(ctx)
    transformCh := p.transform(ctx, sourceCh, 10)

    if err := p.sink(ctx, transformCh); err != nil {
        return err
    }

    p.source.Close()
    p.sink.Close()
    return nil
}
```

### Шаг 6: main

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    pipeline := NewPipeline(source, transformer, sink, 1000)

    if err := pipeline.Run(ctx); err != nil {
        slog.Error("run failed", "error", err)
        os.Exit(1)
    }
}
```

### Что демонстрирует

1. **Pipeline** — source → transform → sink.
2. **Fan-out** — 10 воркеров на transform.
3. **Batch** — 1000 записей или 1 секунда.
4. **Backpressure** — через буферизованные каналы.
5. **Graceful shutdown** — flush при сигнале.
6. **Метрики** — на каждой стадии.

### Аннотация сложности

| Стадия | Time | Space |
|:---|:---|:---|
| Source | ~1-10 мс | 0 |
| Transform | ~1-10 мс × 10 | 0 |
| Sink batch | ~10-100 мс | ~1000 записей |

### 💡 Практика: как строить ETL

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Pipeline** — source → transform → sink.
2. **Fan-out** для медленных стадий.
3. **Batch** для sink.
4. **Backpressure** — буфер 100-1000.
5. **Graceful shutdown** — flush.

**👍 СТОИТ СДЕЛАТЬ:**

6. **Метрики** на каждой стадии.
7. **Таймауты** на transform.

**❌ НЕ ДЕЛАЙ:**

8. **Не забывай flush при shutdown.**
9. **Не делай большие буферы.**
10. **Не игнорируй ошибки transform.**

### Ключевые выводы подглавы 14.3

- **Pipeline** — source → transform → sink.
- **Fan-out** — для transform.
- **Batch** — для sink.
- **Backpressure** — через каналы.
- **Graceful shutdown** — flush.

---

## 14.4 Scraper с rate limiter и circuit breaker

Построим **scraper** для внешних источников.

### Требования

1. **Rate limiter** — не превышать лимиты.
2. **Circuit breaker** — защита от сбоев.
3. **Retry** — временные ошибки.
4. **Concurrency limit** — не более N одновременных.
5. **Graceful shutdown.**
6. **Метрики.**

### Шаг 1: структура

```go
type Scraper struct {
    client   *http.Client
    limiter  *rate.Limiter
    breaker  *gobreaker.CircuitBreaker
    sem      chan struct{}
    workers  int
    metrics  *Metrics
}
```

### Шаг 2: создание

```go
func NewScraper(workers, rps, maxConcurrent int) *Scraper {
    return &Scraper{
        client: &http.Client{Timeout: 10 * time.Second},
        limiter: rate.NewLimiter(rate.Limit(rps), rps*2),
        breaker: gobreaker.NewCircuitBreaker(gobreaker.Settings{
            Name:        "scraper",
            MaxRequests: 3,
            Interval:    10 * time.Second,
            Timeout:     30 * time.Second,
            ReadyToTrip: func(counts gobreaker.Counts) bool {
                return counts.ConsecutiveFailures > 5
            },
        }),
        sem:     make(chan struct{}, maxConcurrent),
        workers: workers,
    }
}
```

### Шаг 3: fetch

```go
func (s *Scraper) fetch(ctx context.Context, url string) ([]byte, error) {
    // 1. Rate limiter
    if err := s.limiter.Wait(ctx); err != nil {
        return nil, err
    }

    // 2. Circuit breaker
    body, err := s.breaker.Execute(func() (interface{}, error) {
        // 3. Semaphore
        select {
        case s.sem <- struct{}{}:
            defer func() { <-s.sem }()
        case <-ctx.Done():
            return nil, ctx.Err()
        }

        // 4. HTTP
        req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
        if err != nil {
            return nil, err
        }
        resp, err := s.client.Do(req)
        if err != nil {
            return nil, err
        }
        defer resp.Body.Close()

        if resp.StatusCode >= 500 {
            return nil, fmt.Errorf("server error: %d", resp.StatusCode)
        }
        if resp.StatusCode >= 400 {
            return nil, &PermanentError{Code: resp.StatusCode}
        }

        return io.ReadAll(resp.Body)
    })
    if err != nil {
        return nil, err
    }

    return body.([]byte), nil
}
```

### Шаг 4: run

```go
func (s *Scraper) Run(ctx context.Context, urls []string) error {
    urlCh := make(chan string, len(urls))
    for _, url := range urls {
        urlCh <- url
    }
    close(urlCh)

    var wg sync.WaitGroup
    for i := 0; i < s.workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            s.worker(ctx, urlCh)
        }()
    }

    wg.Wait()
    return ctx.Err()
}

func (s *Scraper) worker(ctx context.Context, urlCh <-chan string) {
    for url := range urlCh {
        select {
        case <-ctx.Done():
            return
        default:
        }

        body, err := s.retry(ctx, url)
        if err != nil {
            s.metrics.Failed.Add(1)
            slog.Error("fetch error", "url", url, "error", err)
            continue
        }

        s.metrics.Success.Add(1)
        s.process(ctx, url, body)
    }
}
```

### Шаг 5: retry

```go
func (s *Scraper) retry(ctx context.Context, url string) ([]byte, error) {
    const maxAttempts = 3

    var lastErr error
    for attempt := 1; attempt <= maxAttempts; attempt++ {
        body, err := s.fetch(ctx, url)
        if err == nil {
            return body, nil
        }

        var permErr *PermanentError
        if errors.As(err, &permErr) {
            return nil, err  // не retry
        }

        if errors.Is(err, gobreaker.ErrOpenState) {
            return nil, err  // circuit open
        }

        lastErr = err

        if attempt < maxAttempts {
            backoff := time.Duration(attempt) * 100 * time.Millisecond
            select {
            case <-ctx.Done():
                return nil, ctx.Err()
            case <-time.After(backoff):
            }
        }
    }

    return nil, fmt.Errorf("after %d attempts: %w", maxAttempts, lastErr)
}
```

### Шаг 6: main

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    scraper := NewScraper(10, 100, 20)

    urls := loadURLs()
    if err := scraper.Run(ctx, urls); err != nil && !errors.Is(err, context.Canceled) {
        slog.Error("run failed", "error", err)
        os.Exit(1)
    }
}
```

### Что демонстрирует

1. **Rate limiter** — 100 запросов/сек.
2. **Circuit breaker** — защита от сбоев.
3. **Semaphore** — не более 20 одновременных.
4. **Retry** — 3 попытки с backoff.
5. **PermanentError** — не retry.
6. **Graceful shutdown.**

### Аннотация сложности

| Компонент | Time |
|:---|:---|
| Rate limiter | ~50-100 нс |
| Circuit breaker | ~100-200 нс |
| Semaphore | ~30-70 нс |
| HTTP | ~10-1000 мс |
| Retry | × 3 |

### 💡 Практика: как строить scraper

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Rate limiter** — первым.
2. **Circuit breaker** — вторым.
3. **Semaphore** — третьим.
4. **Retry** — для временных ошибок.
5. **PermanentError** — не retry.

**👍 СТОИТ СДЕЛАТЬ:**

6. **Backoff** — между попытками.
7. **Метрики** — success, failed.

**❌ НЕ ДЕЛАЙ:**

8. **Не retry на permanent ошибки.**
9. **Не игнорируй circuit breaker.**
10. **Не превышай rate limit.**

### Ключевые выводы подглавы 14.4

- **Порядок:** rate limiter → circuit breaker → semaphore.
- **Retry** — с backoff.
- **PermanentError** — не retry.
- **Graceful shutdown.**

---

## 14.5 Комбинированный сервис: всё вместе

Соберём **всё вместе**.

### Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                       Service                                │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │ HTTP Server  │───▶│   Kafka      │───▶│   ETL        │  │
│  │ (requests)   │    │  Consumer    │    │  Pipeline    │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│         │                    │                    │         │
│         ▼                    ▼                    ▼         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   Scraper    │    │  Worker Pool │    │   Batch      │  │
│  │ (external)   │    │  (process)   │    │   Sink       │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│                                                              │
│  Всё работает с общим ctx                                   │
└─────────────────────────────────────────────────────────────┘
```

### Структура

```go
type Service struct {
    httpServer *http.Server
    consumer   *Consumer
    pipeline   *Pipeline
    scraper    *Scraper
    
    shutdownTimeout time.Duration
}
```

### Run

```go
func (s *Service) Run(ctx context.Context) error {
    g, ctx := errgroup.WithContext(ctx)

    // HTTP-сервер
    g.Go(func() error {
        return s.runHTTP(ctx)
    })

    // Kafka consumer
    g.Go(func() error {
        return s.consumer.Run(ctx)
    })

    // ETL pipeline
    g.Go(func() error {
        return s.pipeline.Run(ctx)
    })

    // Scraper
    g.Go(func() error {
        return s.scraper.Run(ctx, s.urls)
    })

    return g.Wait()
}
```

### main

```go
func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    slog.SetDefault(logger)

    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    svc := NewService(...)

    start := time.Now()
    if err := svc.Run(ctx); err != nil && !errors.Is(err, context.Canceled) {
        slog.Error("run failed", "error", err)
        os.Exit(1)
    }
    slog.Info("shutdown complete", "elapsed", time.Since(start))
}
```

### Что демонстрирует

1. **errgroup** — все компоненты с общим `ctx`.
2. **Graceful shutdown** — при сигнале.
3. **Комбинация** — HTTP + Kafka + ETL + Scraper.
4. **Метрики** — на каждом компоненте.

### Аннотация сложности

| Компонент | Time |
|:---|:---|
| HTTP | зависит |
| Kafka | зависит |
| ETL | зависит |
| Scraper | зависит |
| Shutdown | 0-25 сек |

### 💡 Практика: как комбинировать компоненты

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Общий `ctx`** — для graceful shutdown.
2. **`errgroup`** — для запуска.
3. **Метрики** на каждом компоненте.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Health checks** — `/healthz`, `/readyz`.
5. **Tracing** — OpenTelemetry.

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `ctx.Done()`** в компонентах.
7. **Не игнорируй ошибки компонентов.**

### Ключевые выводы подглавы 14.5

- **errgroup** — для запуска компонентов.
- **Общий `ctx`** — для graceful shutdown.
- **Метрики** на каждом компоненте.
- **Health checks** — обязательны.

---

## 14.6 Диагностика production-проблем

Разберём **диагностику** production.

### Проблема 1: утечка горутин

**Симптомы:**

- `runtime.NumGoroutine` растёт.
- Память растёт.
- Через час — OOM.

**Диагностика:**

```bash
curl http://localhost:6060/debug/pprof/goroutine?debug=2
```

**Что искать:**

- Много горутин в `chan send`/`chan receive`.
- Много горутин в `IO wait`.
- Много горутин в `semacquire`.

### Проблема 2: contention на мьютексе

**Симптомы:**

- CPU высокий, но throughput низкий.
- Latency растёт.

**Диагностика:**

```go
runtime.SetMutexProfileFraction(1)
```

```bash
curl http://localhost:6060/debug/pprof/mutex
go tool pprof mutex.prof
```

**Что искать:**

- Функции с высоким contention.
- `sync.(*Mutex).Lock` в top.

### Проблема 3: backpressure не работает

**Симптомы:**

- Очередь растёт.
- Память растёт.
- Producer быстрее consumer.

**Диагностика:**

- Метрики длины очереди.
- Метрики latency consumer.

**Что делать:**

- Увеличить число воркеров.
- Уменьшить размер буфера.
- Добавить rate limiter.

### Проблема 4: медленный внешний сервис

**Симптомы:**

- Timeout ошибки.
- Latency растёт.

**Диагностика:**

- Метрики latency по внешним вызовам.
- Circuit breaker состояние.

**Что делать:**

- Circuit breaker.
- Retry с backoff.
- Timeout.

### Проблема 5: deadlock

**Симптомы:**

- Сервис не отвечает.
- CPU низкий.
- Goroutine dump показывает горутины в `semacquire`.

**Диагностика:**

```bash
kill -QUIT <pid>
```

**Что искать:**

- Горутины в `semacquire` на одном мьютексе.
- Горутины в `chan send`/`chan receive`.

### Проблема 6: GC паузы

**Симптомы:**

- Latency spikes.
- pprof показывает много GC.

**Диагностика:**

```bash
GODEBUG=gctrace=1 ./app
```

**Что делать:**

- Уменьшить аллокации.
- `sync.Pool` для частых объектов.

### Шаг: мониторинг

**Обязательные метрики:**

- **Goroutines:** `runtime.NumGoroutine()`.
- **Memory:** `runtime.ReadMemStats`.
- **GC:** паузы, частота.
- **HTTP:** requests, latency, errors.
- **Kafka:** lag, processed.
- **Внешние сервисы:** latency, errors, circuit state.

### Аннотация сложности

| Проблема | Диагностика |
|:---|:---|
| Утечка горутин | pprof goroutine |
| Contention | mutex profile |
| Backpressure | метрики очереди |
| Медленный сервис | метрики latency |
| Deadlock | goroutine dump |
| GC паузы | gctrace |

### 💡 Практика: как диагностировать production

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **pprof в production (localhost).**
2. **Метрики:** goroutines, memory, GC, HTTP, Kafka.
3. **Circuit breaker состояние.**
4. **Логи с контекстом.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Tracing** — OpenTelemetry.
6. **Алерты** на метрики.

**❌ НЕ ДЕЛАЙ:**

7. **Не открывай pprof наружу.**
8. **Не игнорируй рост горутин.**
9. **Не забывай про circuit breaker.**

### Ключевые выводы подглавы 14.6

- **pprof** — для утечек, contention.
- **Метрики** — для backpressure, latency.
- **Goroutine dump** — для deadlock.
- **gctrace** — для GC пауз.
- **Мониторинг** — обязателен.

---

## 14.7 Тестирование реальных сценариев

Разберём **тестирование**.

### HTTP-сервер

```go
func TestServer(t *testing.T) {
    defer goleak.VerifyNone(t)

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    srv := NewServer(":0", 10, 100)

    done := make(chan error, 1)
    go func() {
        done <- srv.Run(ctx)
    }()

    // Даём серверу запуститься
    time.Sleep(100 * time.Millisecond)

    // Тест запросов
    resp, err := http.Get("http://localhost:8080/healthz")
    if err != nil {
        t.Fatal(err)
    }
    if resp.StatusCode != http.StatusOK {
        t.Errorf("status = %d, want 200", resp.StatusCode)
    }
    resp.Body.Close()

    cancel()

    select {
    case err := <-done:
        if err != nil {
            t.Fatal(err)
        }
    case <-time.After(5 * time.Second):
        t.Fatal("shutdown timeout")
    }
}
```

### Consumer

```go
type MockConsumer struct {
    messages []Message
    idx      int
    mu       sync.Mutex
}

func (m *MockConsumer) Read(ctx context.Context) (Message, error) {
    m.mu.Lock()
    defer m.mu.Unlock()
    if m.idx >= len(m.messages) {
        return Message{}, io.EOF
    }
    msg := m.messages[m.idx]
    m.idx++
    return msg, nil
}

func (m *MockConsumer) Commit(ctx context.Context, msgs []Message) error {
    return nil
}

func (m *MockConsumer) Close() error {
    return nil
}

func TestConsumer(t *testing.T) {
    defer goleak.VerifyNone(t)

    consumer := &MockConsumer{
        messages: []Message{
            {ID: "1", Body: []byte("a")},
            {ID: "2", Body: []byte("b")},
            {ID: "3", Body: []byte("c")},
        },
    }

    c := NewConsumer(consumer, 2, 10)

    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()

    if err := c.Run(ctx); err != nil && !errors.Is(err, context.DeadlineExceeded) {
        t.Fatal(err)
    }

    // Проверяем, что все обработаны
    if c.metrics.Processed.Load() != 3 {
        t.Errorf("processed = %d, want 3", c.metrics.Processed.Load())
    }
}
```

### Scraper

```go
func TestScraper(t *testing.T) {
    defer goleak.VerifyNone(t)

    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("hello"))
    }))
    defer srv.Close()

    scraper := NewScraper(5, 100, 10)

    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    urls := []string{srv.URL, srv.URL, srv.URL}
    if err := scraper.Run(ctx, urls); err != nil && !errors.Is(err, context.Canceled) {
        t.Fatal(err)
    }

    if scraper.metrics.Success.Load() != 3 {
        t.Errorf("success = %d, want 3", scraper.metrics.Success.Load())
    }
}
```

### Stress-тесты

```go
func TestServerStress(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping stress test")
    }
    defer goleak.VerifyNone(t)

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    srv := NewServer(":0", 100, 10000)

    done := make(chan error, 1)
    go func() {
        done <- srv.Run(ctx)
    }()

    time.Sleep(100 * time.Millisecond)

    // 1000 запросов параллельно
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            resp, err := http.Get("http://localhost:8080/")
            if err != nil {
                return
            }
            resp.Body.Close()
        }()
    }
    wg.Wait()

    cancel()
    <-done
}
```

### Запуск

```bash
# Обычные тесты
go test ./...

# С race detector
go test -race ./...

# Stress
go test -race -count=10 ./...

# Short
go test -short ./...

# Бенчмарки
go test -bench=. -benchmem ./...
```

### Аннотация сложности

| Тест | Time |
|:---|:---|
| TestServer | ~500 мс |
| TestConsumer | ~1 сек |
| TestScraper | ~2 сек |
| TestServerStress | ~5 сек |

### 💡 Практика: как тестировать реальные сценарии

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`goleak`** — для утечек.
2. **`-race`** — для гонок.
3. **Mock-интерфейсы** — для внешних сервисов.
4. **Stress-тесты** — для нагрузки.

**👍 СТОИТ СДЕЛАТЬ:**

5. **`testing.Short()`** — для skip.
6. **`httptest`** — для HTTP.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `time.Sleep` для синхронизации.**
8. **Не забывай `-race`.**

### Ключевые выводы подглавы 14.7

- **`goleak`** — для утечек.
- **`-race`** — для гонок.
- **Mock-интерфейсы** — для внешних.
- **Stress-тесты** — для нагрузки.

---

## 14.8 Выводы и типичные ошибки

**Что мы узнали?**

HTTP-сервер с ограничением параллелизма: семафор + rate limiter + graceful shutdown. Consumer: worker pool + batch commit + drain + retry + DLQ. ETL: pipeline + fan-out + batch + backpressure. Scraper: rate limiter → circuit breaker → semaphore + retry. Комбинированный сервис: errgroup + общий ctx. Диагностика: pprof, метрики, goroutine dump. Тестирование: goleak, -race, mock, stress.

**Типичные ошибки:**

- ❌ **Не ограничивать параллелизм.** OOM.
- ❌ **Не использовать rate limiter** для внешних.
- ❌ **Не использовать circuit breaker.**
- ❌ **Не делать drain** для очередей.
- ❌ **Не коммить batch.**
- ❌ **Не flush batch при shutdown.**
- ❌ **Не использовать retry.**
- ❌ **Retry на permanent ошибки.**
- ❌ **Не логировать с контекстом.**
- ❌ **Не мониторить goroutines.**
- ❌ **Не использовать pprof.**
- ❌ **Не тестировать graceful shutdown.**
- ❌ **Не использовать goleak.**
- ❌ **Не использовать -race.**

---

## 14.9 Для быстрого повторения

- **HTTP:** семафор + rate limiter + graceful shutdown.
- **Consumer:** worker pool + batch commit + drain + retry + DLQ.
- **ETL:** pipeline + fan-out + batch + backpressure.
- **Scraper:** rate limiter → circuit breaker → semaphore + retry.
- **Комбинированный:** errgroup + общий ctx.
- **Диагностика:** pprof, метрики, goroutine dump.
- **Тестирование:** goleak, -race, mock, stress.
- **Health checks:** `/healthz`, `/readyz`.
- **Метрики:** goroutines, memory, GC, HTTP, Kafka.
- **Graceful shutdown:** 25 секунд.
- **Retry:** 3 попытки с backoff.
- **DLQ:** для permanent ошибок.

---

## 14.10 Вопросы для самопроверки

1. Как ограничить параллелизм в HTTP-сервере?
2. Как защититься от DDoS?
3. Как обработать Kafka с worker pool?
4. Что такое batch commit?
5. Что такое DLQ?
6. Как построить ETL-пайплайн?
7. Что такое backpressure в ETL?
8. Как построить scraper?
9. Почему rate limiter должен быть первым?
10. Что такое circuit breaker?
11. Как комбинировать компоненты в одном сервисе?
12. Как диагностировать утечку горутин?
13. Как диагностировать contention?
14. Как диагностировать deadlock?
15. Как тестировать HTTP-сервер?
16. Как тестировать consumer?
17. Как тестировать scraper?
18. Какие метрики обязательны?

---

## 14.11 Ответы

### Ответ 1

**Семафор:**

```go
sem := make(chan struct{}, maxConcurrent)

select {
case sem <- struct{}{}:
    defer func() { <-sem }()
case <-ctx.Done():
    return
case <-time.After(1 * time.Second):
    http.Error(w, "server busy", 503)
    return
}
```

### Ответ 2

**Rate limiter:**

```go
limiter := rate.NewLimiter(rate.Limit(rps), burst)

if !limiter.Allow() {
    http.Error(w, "too many requests", 429)
    return
}
```

### Ответ 3

**Worker pool:**

```go
messagesCh := make(chan Message, 100)

var wg sync.WaitGroup
for i := 0; i < workers; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for msg := range messagesCh {
            process(msg)
        }
    }()
}
```

### Ответ 4

**Batch commit** — коммит группы сообщений вместо каждого.

**Зачем:** производительность (меньше I/O).

### Ответ 5

**DLQ** (Dead Letter Queue) — очередь для сообщений, которые не удалось обработать.

### Ответ 6

**ETL:**

```go
sourceCh := source(ctx)
transformCh := transform(ctx, sourceCh, 10)
sink(ctx, transformCh)
```

### Ответ 7

**Backpressure** — медленная стадия замедляет быструю. Через буферизованные каналы.

### Ответ 8

**Scraper:**

```go
body, err := scraper.fetch(ctx, url)
```

С rate limiter → circuit breaker → semaphore → HTTP.

### Ответ 9

**Rate limiter первым**, чтобы не тратить ресурсы на запросы, которые всё равно будут отклонены.

### Ответ 10

**Circuit breaker** — защита от каскадных отказов. Три состояния: Closed, Open, Half-Open.

### Ответ 11

**errgroup** + общий ctx:

```go
g, ctx := errgroup.WithContext(ctx)

g.Go(func() error { return s.runHTTP(ctx) })
g.Go(func() error { return s.consumer.Run(ctx) })

return g.Wait()
```

### Ответ 12

**pprof goroutine dump:**

```bash
curl http://localhost:6060/debug/pprof/goroutine?debug=2
```

### Ответ 13

**Mutex profile:**

```go
runtime.SetMutexProfileFraction(1)
```

```bash
go tool pprof mutex.prof
```

### Ответ 14

**Goroutine dump:**

```bash
kill -QUIT <pid>
```

Искать горутины в `semacquire`.

### Ответ 15

```go
func TestServer(t *testing.T) {
    defer goleak.VerifyNone(t)
    
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    srv := NewServer(":0", 10, 100)
    go srv.Run(ctx)
    
    time.Sleep(100 * time.Millisecond)
    
    resp, _ := http.Get("http://localhost:8080/healthz")
    // ...
}
```

### Ответ 16

Mock-интерфейс:

```go
type MockConsumer struct {
    messages []Message
    idx      int
}
```

### Ответ 17

`httptest.NewServer`:

```go
srv := httptest.NewServer(handler)
defer srv.Close()
```

### Ответ 18

**Обязательные метрики:**
- Goroutines.
- Memory.
- GC.
- HTTP requests, latency, errors.
- Kafka lag.
- Circuit breaker state.

---

## 14.12 Куда идти дальше?

Мы разобрали реальные сценарии: HTTP, очереди, ETL, scraping. Теперь мы умеем **строить production-ready сервисы**.

**Что дальше:**

- **Приложение A:** Шпаргалка по каналам.
- **Приложение B:** Шпаргалка по sync.
- **Приложение C:** Шпаргалка по контексту.
- **Приложение D:** Шпаргалка по планировщику.
- **Приложение E:** Инструменты и метрики.

**Или:** применяй знания в своих проектах.

---

## 14.13 Чек-лист

| Сценарий | Компоненты |
|:---|:---|
| **HTTP** | Семафор + rate limiter + graceful shutdown |
| **Consumer** | Worker pool + batch commit + drain + retry + DLQ |
| **ETL** | Pipeline + fan-out + batch + backpressure |
| **Scraper** | Rate limiter + circuit breaker + semaphore + retry |
| **Комбинированный** | errgroup + общий ctx |
| **Диагностика** | pprof, метрики, goroutine dump |
| **Тестирование** | goleak, -race, mock, stress |
| **Health checks** | `/healthz`, `/readyz` |
| **Метрики** | goroutines, memory, GC, HTTP, Kafka |
| **Graceful shutdown** | 25 секунд |
| **Retry** | 3 попытки с backoff |
| **DLQ** | Для permanent ошибок |

🌍 **Ключевая идея:** Реальные сценарии комбинируют все паттерны. **HTTP:** семафор для ограничения, rate limiter для DDoS, graceful shutdown. **Consumer:** worker pool, batch commit, drain, retry, DLQ. **ETL:** pipeline, fan-out, batch, backpressure. **Scraper:** rate limiter → circuit breaker → semaphore + retry. **Комбинированный:** errgroup + общий ctx. **Диагностика:** pprof, метрики, goroutine dump. **Тестирование:** goleak, -race, mock, stress. **Метрики:** goroutines, memory, GC, HTTP, Kafka. **Graceful shutdown:** 25 секунд. **Retry:** 3 попытки с backoff. **DLQ:** для permanent ошибок.