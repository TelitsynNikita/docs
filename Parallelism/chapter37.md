# 🕷️ Глава 37: Scraper — сбор данных из внешних источников

**Что вы узнаете:**
- Что такое scraper и какую задачу он решает.
- Как построить **production-ready scraper** на Go.
- Как использовать **worker pool** для параллельного сбора.
- Как добавить **rate limiter** для защиты внешних сервисов.
- Как реализовать **retry** с backoff и **circuit breaker**.
- Как обрабатывать **robots.txt**, **User-Agent** и **прокси**.
- Как **дренировать** scraper при shutdown.
- Как **мониторить** throughput, errors, rate limits.
- Как избежать типичных ошибок: банов, утечек, DDoS.

**После прочтения вы сможете:**
- Построить scraper с worker pool.
- Настроить rate limiter под конкретный сайт.
- Реализовать retry и circuit breaker.
- Обрабатывать ошибки и DLQ.
- Корректно завершать scraper.
- Мониторить throughput, errors, rate limits.

---

## Содержание

- [37.0 Пролог: scraper, который забанили](#370-пролог-scraper-который-забанили)
- [37.1 Что такое scraper](#371-что-такое-scraper)
- [37.2 Базовая структура](#372-базовая-структура)
- [37.3 Worker pool для параллельного сбора](#373-worker-pool-для-параллельного-сбора)
- [37.4 Rate limiter: защита внешних сервисов](#374-rate-limiter-защита-внешних-сервисов)
- [37.5 Retry и circuit breaker](#375-retry-и-circuit-breaker)
- [37.6 Robots.txt, User-Agent, прокси](#376-robotstxt-user-agent-прокси)
- [37.7 Graceful shutdown: drain scraper](#377-graceful-shutdown-drain-scraper)
- [37.8 Мониторинг: throughput, errors, rate limits](#378-мониторинг-throughput-errors-rate-limits)
- [37.9 Полный production-ready scraper](#379-полный-production-ready-scraper)
- [37.10 Выводы и типичные ошибки](#3710-выводы-и-типичные-ошибки)
- [37.11 Для быстрого повторения](#3711-для-быстрого-повторения)
- [37.12 Вопросы для самопроверки](#3712-вопросы-для-самопроверки)
- [37.13 Ответы](#3713-ответы)
- [37.14 Куда идти дальше?](#3714-куда-идти-дальше)
- [37.15 Чек-лист](#3715-чек-лист)

---

## 37.0 Пролог: scraper, который забанили

У нас есть сервис, который собирает данные с внешнего сайта. 10 000 страниц, каждая — HTTP-запрос. Пишем наивно:

```go
func main() {
    for _, url := range urls {
        go fetch(url)  // ← 10 000 горутин
    }
    time.Sleep(1 * time.Hour)
}

func fetch(url string) {
    resp, err := http.Get(url)
    if err != nil {
        return
    }
    defer resp.Body.Close()
    // обработка
}
```

Работает. Но через 30 секунд приходит письмо от администратора сайта:

> «Ваш IP заблокирован за DDoS. 10 000 запросов в секунду.»

**Что произошло:**

- 10 000 одновременных запросов.
- Сайт не выдержал.
- **Нас забанили.**

Хочется: **вежливый** scraper, который:

1. **Ограничивает** параллелизм (worker pool).
2. **Ограничивает** скорость (rate limiter).
3. **Уважает** `robots.txt`.
4. **Ретраит** временные ошибки.
5. **Останавливается** при бане (circuit breaker).
6. **Корректно завершается** по сигналу.
7. **Мониторится** через метрики.

> **Мост к следующим главам:** scraper — синтез паттернов: worker pool (Глава 13), rate limiter (Глава 14), retry (Глава 16), circuit breaker (Глава 15), bulkhead (Глава 22), graceful shutdown (Глава 26), backpressure (Глава 27). Понимание scraper'а даёт понимание, **как работать с внешними сервисами вежливо**.

---

## 37.1 Что такое scraper

**Scraper** — программа, которая **собирает данные** с внешних источников (сайтов, API).

### Идея

- **Получаем список URL.**
- **Скачиваем** каждый URL.
- **Парсим** данные.
- **Сохраняем** результат.

### Компоненты

**1. Источник URL.**

- Список в файле.
- Из БД.
- Из sitemap.
- Обход сайта.

**2. HTTP-клиент.**

- С таймаутами.
- С retry.
- С прокси.

**3. Парсер.**

- HTML, JSON, XML.
- Извлечение данных.

**4. Приёмник.**

- БД.
- Файл.
- Kafka.

### Когда использовать scraper

**1. Сбор данных.**

- Цены конкурентов.
- Новости.
- Вакансии.

**2. Мониторинг.**

- Изменения на сайте.
- Новые объявления.

**3. Исследования.**

- Публичные данные.
- Аналитика.

### Когда НЕ использовать scraper

**1. Есть API.**

Если у сайта есть API — используй его.

**2. Запрещено robots.txt.**

Не нарушай правила.

**3. Платные данные.**

Не воруй контент.

### Правовые аспекты

**Robots.txt** — правила для ботов.

**User-Agent** — идентификация бота.

**Terms of Service** — условия использования.

**Rate limiting** — вежливость.

### 💡 Практика: как думать о scraper

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Worker pool** — ограничение параллелизма.
2. **Rate limiter** — вежливость.
3. **Robots.txt** — уважение.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Retry + circuit breaker.**
5. **Метрики.**

**❌ НЕ ДЕЛАЙ:**

6. **Не делай 10 000 одновременных запросов.**
7. **Не игнорируй бан.**

---

## 37.2 Базовая структура

Начнём с простейшего scraper.

### Типы

```go
type URL struct {
    ID   int
    Link string
}

type Result struct {
    URL      URL
    Body     []byte
    Status   int
    Err      error
    Duration time.Duration
}
```

### HTTP-клиент

```go
type HTTPClient struct {
    client *http.Client
}

func NewHTTPClient(timeout time.Duration) *HTTPClient {
    return &HTTPClient{
        client: &http.Client{
            Timeout: timeout,
            Transport: &http.Transport{
                MaxIdleConns:        100,
                MaxIdleConnsPerHost: 10,
                IdleConnTimeout:     90 * time.Second,
                DialContext: (&net.Dialer{
                    Timeout:   5 * time.Second,
                    KeepAlive: 30 * time.Second,
                }).DialContext,
                TLSHandshakeTimeout: 5 * time.Second,
            },
        },
    }
}

func (c *HTTPClient) Fetch(ctx context.Context, url string) ([]byte, int, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, 0, err
    }
    req.Header.Set("User-Agent", "MyScraper/1.0 (+https://example.com/bot)")

    resp, err := c.client.Do(req)
    if err != nil {
        return nil, 0, err
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(io.LimitReader(resp.Body, 10*1024*1024))  // 10 MB лимит
    if err != nil {
        return nil, resp.StatusCode, err
    }

    return body, resp.StatusCode, nil
}
```

**Что важно:**

- **Timeout** — защита от зависаний.
- **MaxIdleConns** — переиспользование соединений.
- **User-Agent** — идентификация.
- **Лимит** на размер тела — защита от больших ответов.

### Простой scraper

```go
func main() {
    client := NewHTTPClient(10 * time.Second)
    urls := []URL{
        {ID: 1, Link: "https://example.com/1"},
        {ID: 2, Link: "https://example.com/2"},
    }

    for _, u := range urls {
        body, status, err := client.Fetch(context.Background(), u.Link)
        if err != nil {
            log.Printf("error fetching %s: %v", u.Link, err)
            continue
        }
        log.Printf("fetched %s: status=%d, len=%d", u.Link, status, len(body))
    }
}
```

**Проблема:** последовательно. Для 10 000 URL — медленно.

### 💡 Практика: как построить базовый scraper

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **HTTP-клиент с таймаутами.**
2. **User-Agent.**
3. **Лимит на размер тела.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Переиспользование соединений** (MaxIdleConns).
5. **Контекст** для отмены.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `http.DefaultClient`.** Нет таймаутов.
7. **Не читай тело без лимита.**

---

## 37.3 Worker pool для параллельного сбора

Параллелим сбор через worker pool.

### Идея

- **Producer** пишет URL в канал.
- **N воркеров** читают URL и скачивают.
- **Результаты** — в канал результатов.

### Реализация

```go
type Scraper struct {
    client  *HTTPClient
    workers int
    metrics *Metrics
}

func (s *Scraper) Run(ctx context.Context, urls []URL) (<-chan Result, error) {
    urlsCh := make(chan URL, len(urls))
    resultsCh := make(chan Result, len(urls))

    var wg sync.WaitGroup
    for i := 0; i < s.workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            s.worker(ctx, urlsCh, resultsCh)
        }()
    }

    // Producer
    go func() {
        defer close(urlsCh)
        for _, url := range urls {
            select {
            case urlsCh <- url:
            case <-ctx.Done():
                return
            }
        }
    }()

    // Закрытие результатов
    go func() {
        wg.Wait()
        close(resultsCh)
    }()

    return resultsCh, nil
}

func (s *Scraper) worker(ctx context.Context, in <-chan URL, out chan<- Result) {
    for url := range in {
        select {
        case <-ctx.Done():
            return
        default:
        }

        start := time.Now()
        body, status, err := s.client.Fetch(ctx, url.Link)
        duration := time.Since(start)

        result := Result{
            URL:      url,
            Body:     body,
            Status:   status,
            Err:      err,
            Duration: duration,
        }

        s.metrics.Record(result)

        select {
        case out <- result:
        case <-ctx.Done():
            return
        }
    }
}
```

### Потребитель

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    client := NewHTTPClient(10 * time.Second)
    scraper := NewScraper(client, 20)  // 20 воркеров

    urls := loadURLs()
    results, _ := scraper.Run(ctx, urls)

    for result := range results {
        if result.Err != nil {
            log.Printf("error: %v", result.Err)
            continue
        }
        log.Printf("fetched %s: status=%d, len=%d, duration=%v",
            result.URL.Link, result.Status, len(result.Body), result.Duration)
    }
}
```

### Схема

```
URLs ──► urlsCh ──► N воркеров ──► resultsCh
           │            │                │
           │            ├── fetch        │
           │            └── metrics      │
           │                             │
           └── producer                  └── consumer
```

### Сколько воркеров

**Рекомендации:**

- **Много маленьких сайтов:** 20–50.
- **Один большой сайт:** 5–10 (вежливость).
- **С rate limiter:** больше (rate limiter ограничит).

**Правило:** воркеры **не больше**, чем rate limiter позволяет.

### Backpressure

**Результаты** — в канал с буфером. Если consumer медленный — воркеры **блокируются**.

**Размер буфера:** `len(urls)` или 1000.

### 💡 Практика: как построить worker pool для scraper

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **N воркеров** — параллельный сбор.
2. **Канал URL** — вход.
3. **Канал результатов** — выход.
4. **`select` с `ctx.Done()`.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Размер буфера `len(urls)`.**
6. **Метрики** в воркере.

**❌ НЕ ДЕЛАЙ:**

7. **Не делай 10 000 воркеров.**
8. **Не игнорируй backpressure.**

---

## 37.4 Rate limiter: защита внешних сервисов

Rate limiter — **критичен** для scraper.

### Проблема

**10 000 запросов в секунду** — сайт не выдержит. **Забанят.**

### Решение: rate limiter

**Глава 14** — rate limiter. Для scraper — **на каждый домен**.

```go
type DomainLimiters struct {
    limiters sync.Map  // map[string]*rate.Limiter
}

func (d *DomainLimiters) Get(domain string) *rate.Limiter {
    if l, ok := d.limiters.Load(domain); ok {
        return l.(*rate.Limiter)
    }

    // 10 запросов/сек на домен
    limiter := rate.NewLimiter(rate.Limit(10), 20)
    d.limiters.Store(domain, limiter)
    return limiter
}

func extractDomain(url string) string {
    u, err := neturl.Parse(url)
    if err != nil {
        return ""
    }
    return u.Host
}
```

### Использование в воркере

```go
func (s *Scraper) worker(ctx context.Context, in <-chan URL, out chan<- Result) {
    for url := range in {
        domain := extractDomain(url.Link)
        limiter := s.limiters.Get(domain)

        // Ждём разрешения
        if err := limiter.Wait(ctx); err != nil {
            return
        }

        start := time.Now()
        body, status, err := s.client.Fetch(ctx, url.Link)
        // ...
    }
}
```

**Что происходит:**

- Каждый домен имеет **свой** rate limiter.
- Запросы к **одному** домену — **не более 10/сек**.
- Запросы к **разным** доменам — **параллельно**.

### Настройка rate limiter

**Рекомендации:**

- **Маленькие сайты:** 1–5/сек.
- **Средние:** 5–10/сек.
- **Большие (API):** 10–100/сек.
- **С уважением:** всегда проверяй `robots.txt`.

### Robots.txt

**Читаем robots.txt** перед началом.

```go
type RobotsChecker struct {
    cache sync.Map  // map[string]*robotstxt.RobotsData
}

func (r *RobotsChecker) CanFetch(ctx context.Context, url string) (bool, error) {
    u, err := neturl.Parse(url)
    if err != nil {
        return false, err
    }

    // Проверяем кэш
    if data, ok := r.cache.Load(u.Host); ok {
        return data.(*robotstxt.RobotsData).TestAgent("MyScraper"), nil
    }

    // Загружаем robots.txt
    robotsURL := fmt.Sprintf("%s://%s/robots.txt", u.Scheme, u.Host)
    resp, err := http.Get(robotsURL)
    if err != nil {
        // Нет robots.txt — можно всё
        r.cache.Store(u.Host, &robotstxt.RobotsData{})
        return true, nil
    }
    defer resp.Body.Close()

    data, err := robotstxt.FromResponse(resp)
    if err != nil {
        return true, nil
    }

    r.cache.Store(u.Host, data)
    return data.TestAgent("MyScraper"), nil
}
```

**В воркере:**

```go
canFetch, _ := s.robots.CanFetch(ctx, url.Link)
if !canFetch {
    log.Printf("robots.txt запрещает: %s", url.Link)
    continue
}
```

### Схема

```
URL ──► Robots Check ──► Rate Limiter (per domain) ──► Fetch
          │                    │
          │                    ├── 10/сек для example.com
          │                    ├── 5/сек для google.com
          │                    └── 100/сек для api.example.com
          │
          └── robots.txt для каждого домена
```

### 💡 Практика: как использовать rate limiter

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Rate limiter на каждый домен.**
2. **Robots.txt** — уважение.
3. **User-Agent** — идентификация.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Разные лимиты** для разных доменов.
5. **Кэш robots.txt.**

**❌ НЕ ДЕЛАЙ:**

6. **Не игнорируй robots.txt.**
7. **Не делай > 100/сек без разрешения.**

---

## 37.5 Retry и circuit breaker

Ошибки — нормальны. Retry и circuit breaker.

### Retry

**Глава 16** — retry для временных ошибок.

```go
func (s *Scraper) fetchWithRetry(ctx context.Context, url string) ([]byte, int, error) {
    const maxAttempts = 3

    var lastErr error
    for attempt := 1; attempt <= maxAttempts; attempt++ {
        body, status, err := s.client.Fetch(ctx, url)
        if err == nil {
            return body, status, nil
        }

        // Permanent — не retry
        if status >= 400 && status < 500 && status != 429 {
            return nil, status, &PermanentError{Code: status}
        }

        lastErr = err
        s.metrics.Retries.Add(1)

        if attempt < maxAttempts {
            backoff := time.Duration(attempt) * 100 * time.Millisecond
            select {
            case <-ctx.Done():
                return nil, 0, ctx.Err()
            case <-time.After(backoff):
            }
        }
    }
    return nil, 0, fmt.Errorf("after %d attempts: %w", maxAttempts, lastErr)
}
```

**Что retry:**

- **Сетевые ошибки.**
- **5xx.**
- **429 Too Many Requests.**

**Что НЕ retry:**

- **4xx (кроме 429).**
- **Permanent ошибки.**

### Circuit breaker

**Глава 15** — circuit breaker для защиты от сбоев.

```go
type CircuitBreakers struct {
    breakers sync.Map  // map[string]*gobreaker.CircuitBreaker
}

func (c *CircuitBreakers) Get(domain string) *gobreaker.CircuitBreaker {
    if b, ok := c.breakers.Load(domain); ok {
        return b.(*gobreaker.CircuitBreaker)
    }

    b := gobreaker.NewCircuitBreaker(gobreaker.Settings{
        Name:        domain,
        MaxRequests: 3,
        Timeout:     30 * time.Second,
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            return counts.ConsecutiveFailures > 10
        },
    })
    c.breakers.Store(domain, b)
    return b
}
```

**В воркере:**

```go
domain := extractDomain(url.Link)
breaker := s.breakers.Get(domain)

_, err := breaker.Execute(func() (interface{}, error) {
    return nil, s.fetchWithRetry(ctx, url.Link)
})

if errors.Is(err, gobreaker.ErrOpenState) {
    // Circuit открыт — домен недоступен
    log.Printf("circuit open for %s", domain)
    return
}
```

### Схема

```
URL ──► Circuit Breaker ──► Retry ──► Fetch
          │                    │
          │                    ├── 3 попытки
          │                    └── backoff
          │
          └── если > 10 ошибок — открыт на 30 сек
```

### 💡 Практика: как использовать retry и circuit breaker

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Retry** для временных.
2. **Circuit breaker** для доменов.
3. **Backoff** между попытками.

**👍 СТОИТ СДЕЛАТЬ:**

4. **PermanentError** — не retry.
5. **Метрики** — retries, circuit open.

**❌ НЕ ДЕЛАЙ:**

6. **Не retry 4xx.**
7. **Не игнорируй circuit open.**

---

## 37.6 Robots.txt, User-Agent, прокси

Правовые и технические аспекты.

### User-Agent

**Обязательно** — идентификация.

```go
req.Header.Set("User-Agent", "MyScraper/1.0 (+https://example.com/bot)")
```

**Что включает:**

- **Имя бота.**
- **Версия.**
- **URL** — где можно связаться.

### Robots.txt

**Читай перед началом.**

```go
data, err := robotstxt.FromResponse(resp)
canFetch := data.TestAgent("MyScraper")
```

**Кэшируй** для каждого домена.

### Прокси

**Для распределения нагрузки** — прокси.

```go
proxyURL, _ := url.Parse("http://proxy:8080")
transport := &http.Transport{
    Proxy: http.ProxyURL(proxyURL),
    // ...
}
```

**Ротация прокси:**

```go
type ProxyPool struct {
    proxies []*url.URL
    idx     atomic.Int64
}

func (p *ProxyPool) Next() *url.URL {
    i := p.idx.Add(1) - 1
    return p.proxies[int(i)%len(p.proxies)]
}
```

**В клиенте:**

```go
func (c *HTTPClient) FetchWithProxy(ctx context.Context, url string, proxy *url.URL) ([]byte, int, error) {
    transport := &http.Transport{
        Proxy: http.ProxyURL(proxy),
        // ...
    }
    client := &http.Client{
        Timeout:   10 * time.Second,
        Transport: transport,
    }
    // ...
}
```

### Задержки

**Между запросами:**

```go
select {
case <-ctx.Done():
    return
case <-time.After(100 * time.Millisecond):
}
```

**Что даёт:** вежливость.

### 💡 Практика: как быть вежливым

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **User-Agent** с контактами.
2. **Robots.txt** — уважение.
3. **Задержки** между запросами.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Прокси** — если нужно.
5. **Разные IP.**

**❌ НЕ ДЕЛАЙ:**

6. **Не скрывайся** без причины.
7. **Не нарушай robots.txt.**

---

## 37.7 Graceful shutdown: drain scraper

Graceful shutdown (Глава 26) — **обязателен**.

### Что должно произойти

1. **Producer** — прекратить отправку URL.
2. **Воркеры** — дренировать канал URL.
3. **Результаты** — обработать оставшиеся.
4. **Закрыть** HTTP-клиент.

### Реализация

```go
func (s *Scraper) Run(ctx context.Context, urls []URL) error {
    urlsCh := make(chan URL, len(urls))
    resultsCh := make(chan Result, len(urls))

    var wg sync.WaitGroup
    for i := 0; i < s.workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            s.worker(ctx, urlsCh, resultsCh)
        }()
    }

    // Producer
    go func() {
        defer close(urlsCh)
        for _, url := range urls {
            select {
            case urlsCh <- url:
            case <-ctx.Done():
                return
            }
        }
    }()

    // Закрытие результатов
    go func() {
        wg.Wait()
        close(resultsCh)
    }()

    // Consumer
    for result := range resultsCh {
        if result.Err != nil {
            s.metrics.Failed.Add(1)
            continue
        }
        s.metrics.Success.Add(1)
        s.process(result)
    }

    return ctx.Err()
}
```

### Что происходит

```
SIGTERM:
  │
  ├── Producer: close(urlsCh)
  │
  ├── Workers: drain urlsCh, обработать
  │
  ├── close(resultsCh)
  │
  └── Consumer: drain resultsCh, обработать
```

**Ничего не теряется.**

### Таймаут

**Что если не завершается за 30 секунд?**

```go
shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

// Запускаем Run с shutdownCtx
```

### 💡 Практика: как дренировать scraper

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx.Done()`** — сигнал.
2. **Producer закрывает канал.**
3. **Воркеры дренируют.**
4. **Consumer обрабатывает результаты.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Таймаут** на shutdown.
6. **Метрики** — сколько успели.

**❌ НЕ ДЕЛАЙ:**

7. **Не прерывай воркеры.**
8. **Не закрывай HTTP-клиент до дренажа.**

---

## 37.8 Мониторинг: throughput, errors, rate limits

Мониторинг — **обязателен**.

### Ключевые метрики

| Метрика | Что измеряет |
|:---|:---|
| **Throughput** | URL/сек |
| **Success rate** | Успешных запросов |
| **Error rate** | Ошибок/сек |
| **Latency** | Время запроса |
| **Rate limit waits** | Ожидание rate limiter |
| **Retries** | Retry/сек |
| **Circuit state** | Открыт/закрыт |

### Метрики

```go
type Metrics struct {
    Fetched    atomic.Int64
    Success    atomic.Int64
    Failed     atomic.Int64
    Retries    atomic.Int64
    DLQ        atomic.Int64
    TotalTime  atomic.Int64
}

func (m *Metrics) Record(result Result) {
    m.Fetched.Add(1)
    m.TotalTime.Add(int64(result.Duration))

    if result.Err != nil {
        m.Failed.Add(1)
    } else {
        m.Success.Add(1)
    }
}
```

### Prometheus

```go
var (
    scrapedTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "scraper_requests_total"},
        []string{"domain", "status"},
    )
    scrapeDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{Name: "scraper_request_duration_seconds"},
        []string{"domain"},
    )
    rateLimitWait = prometheus.NewHistogram(
        prometheus.HistogramOpts{Name: "scraper_rate_limit_wait_seconds"},
    )
)
```

### Что алертить

- **Error rate > 10%** — проблемы.
- **Circuit открыт** — домен недоступен.
- **Rate limit waits растут** — слишком агрессивно.
- **Latency растёт** — сайт тормозит.

### Схема

```
Каждый запрос:
  │
  ├── Rate limit wait ──► метрика
  │
  ├── Circuit breaker state ──► метрика
  │
  ├── Fetch ──► latency, status
  │
  └── Retry ──► метрика
```

### 💡 Практика: как мониторить scraper

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Throughput** — URL/сек.
2. **Success rate** — % успешных.
3. **Error rate** — по типам.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Latency** — p50, p95, p99.
5. **Circuit state.**
6. **Rate limit waits.**

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй error rate.**
8. **Не игнорируй circuit open.**

---

## 37.9 Полный production-ready scraper

Соберём всё вместе.

### Полный код

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "io"
    "log/slog"
    "net"
    neturl "net/url"
    "os"
    "os/signal"
    "sync"
    "sync/atomic"
    "syscall"
    "time"

    "github.com/sony/gobreaker"
    "golang.org/x/time/rate"
)

type URL struct {
    ID   int
    Link string
}

type Result struct {
    URL      URL
    Body     []byte
    Status   int
    Err      error
    Duration time.Duration
}

type Metrics struct {
    Fetched   atomic.Int64
    Success   atomic.Int64
    Failed    atomic.Int64
    Retries   atomic.Int64
    TotalTime atomic.Int64
}

type HTTPClient struct {
    client *http.Client
}

func NewHTTPClient(timeout time.Duration) *HTTPClient {
    return &HTTPClient{
        client: &http.Client{
            Timeout: timeout,
            Transport: &http.Transport{
                MaxIdleConns:        100,
                MaxIdleConnsPerHost: 10,
                IdleConnTimeout:     90 * time.Second,
                DialContext: (&net.Dialer{
                    Timeout:   5 * time.Second,
                    KeepAlive: 30 * time.Second,
                }).DialContext,
                TLSHandshakeTimeout: 5 * time.Second,
            },
        },
    }
}

func (c *HTTPClient) Fetch(ctx context.Context, url string) ([]byte, int, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, 0, err
    }
    req.Header.Set("User-Agent", "MyScraper/1.0 (+https://example.com/bot)")

    resp, err := c.client.Do(req)
    if err != nil {
        return nil, 0, err
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(io.LimitReader(resp.Body, 10*1024*1024))
    if err != nil {
        return nil, resp.StatusCode, err
    }

    return body, resp.StatusCode, nil
}

type Scraper struct {
    client    *HTTPClient
    workers   int
    metrics   *Metrics
    limiters  sync.Map
    breakers  sync.Map
}

func NewScraper(client *HTTPClient, workers int) *Scraper {
    return &Scraper{
        client:  client,
        workers: workers,
        metrics: &Metrics{},
    }
}

func (s *Scraper) Run(ctx context.Context, urls []URL) error {
    urlsCh := make(chan URL, len(urls))
    resultsCh := make(chan Result, len(urls))

    var wg sync.WaitGroup
    for i := 0; i < s.workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            s.worker(ctx, urlsCh, resultsCh)
        }()
    }

    go func() {
        defer close(urlsCh)
        for _, url := range urls {
            select {
            case urlsCh <- url:
            case <-ctx.Done():
                return
            }
        }
    }()

    go func() {
        wg.Wait()
        close(resultsCh)
    }()

    for result := range resultsCh {
        s.metrics.Record(result)
        if result.Err != nil {
            slog.Error("fetch failed",
                "url", result.URL.Link,
                "error", result.Err,
                "status", result.Status,
            )
            continue
        }
        slog.Info("fetch succeeded",
            "url", result.URL.Link,
            "status", result.Status,
            "size", len(result.Body),
            "duration", result.Duration,
        )
    }

    return ctx.Err()
}

func (s *Scraper) worker(ctx context.Context, in <-chan URL, out chan<- Result) {
    for url := range in {
        select {
        case <-ctx.Done():
            return
        default:
        }

        domain := extractDomain(url.Link)

        // Rate limiter
        limiter := s.getLimiter(domain)
        if err := limiter.Wait(ctx); err != nil {
            return
        }

        // Circuit breaker
        breaker := s.getBreaker(domain)

        start := time.Now()
        var body []byte
        var status int
        var err error

        _, cbErr := breaker.Execute(func() (interface{}, error) {
            body, status, err = s.fetchWithRetry(ctx, url.Link)
            return nil, err
        })

        if cbErr != nil {
            err = cbErr
        }

        result := Result{
            URL:      url,
            Body:     body,
            Status:   status,
            Err:      err,
            Duration: time.Since(start),
        }

        select {
        case out <- result:
        case <-ctx.Done():
            return
        }
    }
}

func (s *Scraper) fetchWithRetry(ctx context.Context, url string) ([]byte, int, error) {
    const maxAttempts = 3

    var lastErr error
    for attempt := 1; attempt <= maxAttempts; attempt++ {
        body, status, err := s.client.Fetch(ctx, url)
        if err == nil {
            return body, status, nil
        }

        if status >= 400 && status < 500 && status != 429 {
            return nil, status, err
        }

        lastErr = err
        s.metrics.Retries.Add(1)

        if attempt < maxAttempts {
            backoff := time.Duration(attempt) * 100 * time.Millisecond
            select {
            case <-ctx.Done():
                return nil, 0, ctx.Err()
            case <-time.After(backoff):
            }
        }
    }
    return nil, 0, fmt.Errorf("after %d attempts: %w", maxAttempts, lastErr)
}

func (s *Scraper) getLimiter(domain string) *rate.Limiter {
    if l, ok := s.limiters.Load(domain); ok {
        return l.(*rate.Limiter)
    }
    limiter := rate.NewLimiter(rate.Limit(10), 20)
    s.limiters.Store(domain, limiter)
    return limiter
}

func (s *Scraper) getBreaker(domain string) *gobreaker.CircuitBreaker {
    if b, ok := s.breakers.Load(domain); ok {
        return b.(*gobreaker.CircuitBreaker)
    }
    b := gobreaker.NewCircuitBreaker(gobreaker.Settings{
        Name:        domain,
        MaxRequests: 3,
        Timeout:     30 * time.Second,
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            return counts.ConsecutiveFailures > 10
        },
    })
    s.breakers.Store(domain, b)
    return b
}

func extractDomain(link string) string {
    u, err := neturl.Parse(link)
    if err != nil {
        return ""
    }
    return u.Host
}

func (m *Metrics) Record(result Result) {
    m.Fetched.Add(1)
    m.TotalTime.Add(int64(result.Duration))
    if result.Err != nil {
        m.Failed.Add(1)
    } else {
        m.Success.Add(1)
    }
}

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    slog.SetDefault(logger)

    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    client := NewHTTPClient(10 * time.Second)
    scraper := NewScraper(client, 20)

    urls := []URL{
        {ID: 1, Link: "https://example.com"},
        {ID: 2, Link: "https://example.org"},
    }

    if err := scraper.Run(ctx, urls); err != nil && !errors.Is(err, context.Canceled) {
        slog.Error("run failed", "error", err)
        os.Exit(1)
    }

    slog.Info("done",
        "fetched", scraper.metrics.Fetched.Load(),
        "success", scraper.metrics.Success.Load(),
        "failed", scraper.metrics.Failed.Load(),
        "retries", scraper.metrics.Retries.Load(),
    )
}
```

### Что демонстрирует

1. **Worker pool** — 20 воркеров.
2. **Rate limiter** — 10/сек на домен.
3. **Circuit breaker** — защита от сбоев.
4. **Retry** — 3 попытки с backoff.
5. **Graceful shutdown** — drain.
6. **Метрики** — fetched, success, failed, retries.
7. **User-Agent** — идентификация.
8. **Лимит на размер тела** — 10 MB.

### 💡 Практика: как строить production scraper

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Worker pool** — 20 воркеров.
2. **Rate limiter** — на каждый домен.
3. **Retry + circuit breaker.**
4. **Graceful shutdown.**
5. **User-Agent.**

**👍 СТОИТ СДЕЛАТЬ:**

6. **Метрики** — throughput, errors.
7. **Robots.txt.**

**❌ НЕ ДЕЛАЙ:**

8. **Не делай 10 000 воркеров.**
9. **Не игнорируй rate limit.**

---

## 37.10 Выводы и типичные ошибки

**Что мы узнали?**

Scraper — синтез паттернов: worker pool (параллелизм), rate limiter (вежливость), retry (временные ошибки), circuit breaker (защита), graceful shutdown (drain). **User-Agent** и **robots.txt** — обязательны. **Метрики:** throughput, success rate, latency, retries. **Прокси** — для распределения.

**Типичные ошибки:**

- ❌ **10 000 одновременных запросов.** DDoS.
- ❌ **Нет rate limiter.** Забанят.
- ❌ **Игнорировать robots.txt.** Правовые проблемы.
- ❌ **Нет User-Agent.** Анонимный бот.
- ❌ **Нет retry.** Временные ошибки теряются.
- ❌ **Нет circuit breaker.** Зацикливание на сбоях.
- ❌ **Нет graceful shutdown.** Потеря данных.
- ❌ **Нет метрик.** Не видно проблем.
- ❌ **Нет таймаутов.** Зависание.
- ❌ **Нет лимита на размер тела.** OOM.
- ❌ **Один rate limiter на всё.** Не учитывает домены.
- ❌ **Не игнорируй rate limit waits.**

---

## 37.11 Для быстрого повторения

- **Scraper** — синтез паттернов.
- **Worker pool** — 20 воркеров.
- **Rate limiter** — на каждый домен.
- **Retry** — 3 попытки с backoff.
- **Circuit breaker** — защита от сбоев.
- **User-Agent** — идентификация.
- **Robots.txt** — уважение.
- **Graceful shutdown** — drain.
- **Метрики:** throughput, success, latency, retries.
- **Прокси** — для распределения.
- **Таймауты** — на HTTP-клиент.
- **Лимит на размер тела** — 10 MB.
- **Rate limit 10/сек** — вежливость.
- **Circuit open после 10 ошибок.**

---

## 37.12 Вопросы для самопроверки

1. Почему 10 000 одновременных запросов — плохо?
2. Что такое worker pool? Зачем?
3. Что такое rate limiter? Зачем?
4. Зачем robots.txt?
5. Что такое circuit breaker?
6. Как правильно дренировать scraper?
7. Что мониторить в scraper?
8. Как обрабатывать retry?

---

## 37.13 Ответы

### Ответ 1

**10 000 одновременных запросов** — DDoS на сайт. Сайт не выдержит, нас забанят.

**Решение:** worker pool + rate limiter.

### Ответ 2

**Worker pool** — N воркеров обрабатывают URL.

**Зачем:** параллелизм с ограничением.

### Ответ 3

**Rate limiter** — ограничение скорости запросов.

**Зачем:** вежливость. 10/сек на домен.

### Ответ 4

**Robots.txt** — правила для ботов. Указывает, что можно скачивать, а что нет.

**Зачем:** уважение к сайту. Правовые аспекты.

### Ответ 5

**Circuit breaker** — защита от сбоев.

**Зачем:** если домен недоступен — перестать его вызывать. Не зацикливаться.

### Ответ 6

**Drain scraper:**

1. `ctx.Done()`.
2. Producer → `close(urlsCh)`.
3. Workers → drain urlsCh.
4. `close(resultsCh)`.
5. Consumer → drain resultsCh.

### Ответ 7

**Мониторинг scraper:**

- **Throughput** — URL/сек.
- **Success rate** — %.
- **Error rate** — по типам.
- **Latency** — p50, p95, p99.
- **Retries** — retry/сек.
- **Circuit state** — открыт/закрыт.

### Ответ 8

**Retry для временных ошибок:**

- Сетевые ошибки.
- 5xx.
- 429.

**НЕ retry:**

- 4xx (кроме 429).
- Permanent ошибки.

3 попытки с backoff (100 мс, 200 мс, 400 мс).

---

## 37.14 Куда идти дальше?

Мы разобрали scraper — сбор данных вежливо. Теперь мы умеем строить production-ready scraper.

Но остаются **другие сценарии**: WebSocket и streaming, безопасность.

- **Как работать с WebSocket?** → **Глава 38: WebSocket и streaming.**
- **Как обеспечить безопасность?** → **Глава 39: Безопасность конкурентного кода.**

---

## 37.15 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Scraper** | Сбор данных | Синтез паттернов |
| **Worker pool** | Параллельный сбор | 20 воркеров |
| **Rate limiter** | Вежливость | 10/сек на домен |
| **Retry** | Временные ошибки | 3 попытки с backoff |
| **Circuit breaker** | Защита от сбоев | После 10 ошибок |
| **User-Agent** | Идентификация | С контактами |
| **Robots.txt** | Уважение | Кэш по домену |
| **Graceful shutdown** | Drain | Через `ctx.Done()` |
| **Метрики** | Throughput, errors | Prometheus |
| **Прокси** | Распределение | Ротация |
| **Таймауты** | Защита | 10 сек |
| **Лимит тела** | Защита | 10 MB |
| **Backoff** | Retry | 100, 200, 400 мс |

🕷️ **Ключевая идея:** Scraper — синтез паттернов: worker pool (параллелизм), rate limiter (вежливость), retry (временные ошибки), circuit breaker (защита), graceful shutdown (drain). **User-Agent** с контактами, **robots.txt** — уважение. **Rate limiter** на каждый домен (10/сек). **Retry** 3 попытки с backoff (100–400 мс) для 5xx и 429, **не retry** для 4xx. **Circuit breaker** открывается после 10 ошибок. **Drain** при shutdown: producer закрывает urlsCh, воркеры дренируют, consumer обрабатывает. **Метрики:** throughput, success, failed, retries, latency, circuit state. **Прокси** для распределения. **Таймауты** и **лимит тела** обязательны.