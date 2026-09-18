# 🛡️ Глава 9: Продвинутые паттерны — semaphore, rate limiter, circuit breaker

**Что вы узнаете:**
- Чем **семафор** отличается от worker pool, и когда его использовать.
- Что такое **взвешенный семафор** (`golang.org/x/sync/semaphore`) и как он работает.
- Что такое **rate limiter** и какие алгоритмы существуют: token bucket, leaky bucket.
- Как устроен `golang.org/x/time/rate` изнутри.
- Что такое **circuit breaker** и как он защищает от каскадных отказов.
- Как комбинировать семафор, rate limiter и circuit breaker.
- Что такое **bulkhead** и как изолировать сбои.
- Как обрабатывать **retry** с backoff и jitter.
- Как измерять и настраивать эти паттерны в production.
- Как диагностировать проблемы через метрики и логи.

**После прочтения вы сможете:**
- Объяснить, когда семафор, а когда worker pool.
- Использовать взвешенный семафор для ресурсов с разной стоимостью.
- Настроить rate limiter для внешнего API.
- Реализовать circuit breaker с нуля или через библиотеку.
- Комбинировать паттерны: rate limiter + circuit breaker + retry.
- Изолировать сбои через bulkhead.
- Измерять эффективность паттернов через метрики.
- Диагностировать проблемы: throttling, отказы, retry storms.

---

## Содержание

- [9.0 Пролог: сервис, который положил внешний API](#90-пролог-сервис-который-положил-внешний-api)
- [9.1 Семафор: ограничение доступа к ресурсу](#91-семафор-ограничение-доступа-к-ресурсу)
- [9.2 Взвешенный семафор](#92-взвешенный-семафор)
- [9.3 Rate limiter: token bucket](#93-rate-limiter-token-bucket)
- [9.4 Rate limiter: leaky bucket](#94-rate-limiter-leaky-bucket)
- [9.5 `golang.org/x/time/rate` изнутри](#95-golangorgxtimerate-изнутри)
- [9.6 Circuit breaker: защита от каскадных отказов](#96-circuit-breaker-защита-от-каскадных-отказов)
- [9.7 Bulkhead: изоляция сбоев](#97-bulkhead-изоляция-сбоев)
- [9.8 Retry с backoff и jitter](#98-retry-с-backoff-и-jitter)
- [9.9 Комбинирование паттернов](#99-комбинирование-паттернов)
- [9.10 Практика Go: production-ready клиент](#910-практика-go-production-ready-клиент)
- [9.11 Выводы и типичные ошибки](#911-выводы-и-типичные-ошибки)
- [9.12 Для быстрого повторения](#912-для-быстрого-повторения)
- [9.13 Вопросы для самопроверки](#913-вопросы-для-самопроверки)
- [9.14 Ответы](#914-ответы)
- [9.15 Куда идти дальше?](#915-куда-идти-дальше)
- [9.16 Чек-лист](#916-чек-лист)

---

## 9.0 Пролог: сервис, который положил внешний API

Ты пишешь сервис, который вызывает внешний API для обогащения данных. API имеет **rate limit: 100 запросов в секунду**. Ты запускаешь 1000 горутин:

```go
for _, item := range items {
    go func(it Item) {
        resp, err := api.Call(it)
        // ...
    }(item)
}
```

**Что происходит:**

1. **1000 горутин** одновременно делают запросы.
2. **API получает 1000 запросов** за долю секунды.
3. **API отвечает 429 (Too Many Requests)** на 900 из них.
4. **Твой сервис получает 900 ошибок.**
5. **Retry** — ещё 900 запросов.
6. **API блокирует твой IP** за DDoS.
7. **Сервис падает** — все запросы падают.

❓ **Что произошло?** Ты **не ограничил скорость**. API имеет rate limit, а ты его игнорируешь.

💡 **Решение:** **rate limiter** + **circuit breaker** + **retry с backoff**.

- **Rate limiter** — не больше 100 запросов/сек.
- **Circuit breaker** — если API падает, не долбить его.
- **Retry с backoff** — повторять с увеличивающейся задержкой.

**Результат:** сервис работает **стабильно**, API не блокирует, ошибки обрабатываются корректно.

> **Важный мост к будущим главам:** эта глава — о **защите** от проблем. Семафор — из Главы 3 (базовый). Rate limiter — комбинация каналов (Глава 2) и `time` (Глава 5). Circuit breaker — конечный автомат. Retry — комбинация `context` (Глава 5) и `time`. Все паттерны комбинируются.

---

## 9.1 Семафор: ограничение доступа к ресурсу

**Семафор** — примитив, ограничивающий **доступ к ресурсу**. В Go реализуется через **буферизованный канал**.

### Базовый семафор

```go
sem := make(chan struct{}, 10)  // семафор на 10

for _, task := range tasks {
    sem <- struct{}{}  // захватить слот
    go func(t Task) {
        defer func() { <-sem }()  // освободить
        process(t)
    }(task)
}
```

**Что происходит:**

- 10 слотов.
- Каждая задача **захватывает** слот перед запуском.
- Когда все 10 заняты — новая задача **ждёт** на `sem <- struct{}{}`.
- После завершения — слот **освобождается**.

### Семафор vs worker pool

В Главе 7 мы разбирали worker pool. В Главе 8 — fan-out. Теперь сравним с семафором.

| Аспект | Worker pool | Семафор |
|:---|:---|:---|
| Горутин | N (фиксировано) | N + задачи (создаются и завершаются) |
| Переиспользование | Да | Нет |
| Память | N × 2.3 КБ | N × 2.3 КБ + задачи |
| Overhead | ~100-300 нс на N | ~100-300 нс на задачу |
| Простота | Средняя | Высокая |
| Backpressure | Через канал | Через канал |

### Когда использовать семафор

**✅ Семафор лучше:**

- **Мало задач** (100-1000).
- **Задачи разные** (нельзя переиспользовать воркер).
- **Простота важна.**
- **Нужно ограничить доступ к ресурсу** (БД-соединения, file descriptors).

**✅ Worker pool лучше:**

- **Много задач** (10 000+).
- **Задачи однотипные.**
- **Производительность важна.**
- **Воркеры переиспользуются.**

### Пример: семафор для БД-соединений

```go
type DBPool struct {
    sem chan struct{}
    db  *sql.DB
}

func NewDBPool(db *sql.DB, maxConns int) *DBPool {
    return &DBPool{
        sem: make(chan struct{}, maxConns),
        db:  db,
    }
}

func (p *DBPool) Query(ctx context.Context, query string) (*sql.Rows, error) {
    select {
    case p.sem <- struct{}{}:
        defer func() { <-p.sem }()
    case <-ctx.Done():
        return nil, ctx.Err()
    }
    
    return p.db.QueryContext(ctx, query)
}
```

**Что происходит:**

- Не более `maxConns` одновременных запросов к БД.
- Остальные ждут.
- При отмене — выход.

### Пример: семафор для HTTP-клиента

```go
type LimitedClient struct {
    sem    chan struct{}
    client *http.Client
}

func NewLimitedClient(maxConcurrent int) *LimitedClient {
    return &LimitedClient{
        sem:    make(chan struct{}, maxConcurrent),
        client: &http.Client{Timeout: 10 * time.Second},
    }
}

func (c *LimitedClient) Get(ctx context.Context, url string) (*http.Response, error) {
    select {
    case c.sem <- struct{}{}:
        defer func() { <-c.sem }()
    case <-ctx.Done():
        return nil, ctx.Err()
    }
    
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    return c.client.Do(req)
}
```

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| Захват слота | ~50-100 нс | 0 |
| Освобождение слота | ~50-100 нс | 0 |
| Ожидание слота | зависит | 0 |

### 💡 Практика: как использовать семафор

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Семафор для ограничения доступа к ресурсу** (БД, HTTP, файлы).
2. **`select` с `ctx.Done()`** для отмены.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Буферизованный канал** размером N (число слотов).
4. **`defer func() { <-sem }()`** для освобождения.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **Взвешенный семафор** — если ресурсы имеют разную стоимость (см. 9.2).

**❌ НЕ ДЕЛАЙ:**

6. **Не используй семафор для многих однотипных задач** — worker pool лучше.
7. **Не забывай освобождать слот.** Утечка.
8. **Не захватывай слот без `select` с `ctx.Done()`.** Отмена не сработает.

### Ключевые выводы подглавы 9.1

- **Семафор** — ограничение доступа к ресурсу через буферизованный канал.
- **N слотов** — N одновременных доступов.
- **Семафор vs worker pool:** семафор создаёт горутину на задачу, pool переиспользует.
- **Семафор** — для ресурсов (БД, HTTP, файлы).
- **Worker pool** — для многих однотипных задач.

---

## 9.2 Взвешенный семафор

**Взвешенный семафор** — семафор, где каждая задача **захватывает N слотов** (вес).

### Проблема: ресурсы с разной стоимостью

Представь, что ты ограничиваешь **память**. Задачи разные:

- **Малая задача** — 1 МБ.
- **Средняя задача** — 10 МБ.
- **Большая задача** — 100 МБ.

**Обычный семафор** на 10 слотов:

- 10 малых задач — 10 МБ.
- 10 больших задач — 1000 МБ. **OOM.**

**Взвешенный семафор:**

- Малая задача — вес 1.
- Средняя — вес 10.
- Большая — вес 100.
- Семафор на 100 единиц.
- 100 малых, 10 средних, 1 большая — все укладываются в лимит.

### `golang.org/x/sync/semaphore`

```go
import "golang.org/x/sync/semaphore"

sem := semaphore.NewWeighted(100)  // 100 единиц

// Захватить 10 единиц
if err := sem.Acquire(ctx, 10); err != nil {
    return err
}
defer sem.Release(10)

// работа
```

**API:**

| Метод | Что делает |
|:---|:---|
| `NewWeighted(n)` | Создать семафор на n единиц |
| `Acquire(ctx, n)` | Захватить n единиц (блокируется) |
| `TryAcquire(n)` | Попытаться захватить n единиц (без блокировки) |
| `Release(n)` | Освободить n единиц |

### Как работает внутри

`semaphore.Weighted` использует:

- **`size`** — общее число единиц.
- **`cur`** — текущее занятое число.
- **`waiters`** — очередь ждущих (дерево, как `semaRoot` из Главы 3).
- **`mu`** — мьютекс.

**Acquire:**

```
1. Захватить mu.
2. Если size - cur >= n:
   - cur += n.
   - Освободить mu.
   - return nil.
3. Иначе:
   - Создать waiter.
   - Добавить в очередь.
   - gopark.
```

**Release:**

```
1. Захватить mu.
2. cur -= n.
3. Пока есть waiters и size - cur >= waiter.n:
   - Разбудить waiter.
4. Освободить mu.
```

### Пример: ограничение памяти

```go
var memSem = semaphore.NewWeighted(1000)  // 1000 МБ

func processTask(ctx context.Context, task Task) error {
    weight := estimateMemory(task)  // 1-100 МБ
    
    if err := memSem.Acquire(ctx, weight); err != nil {
        return err
    }
    defer memSem.Release(weight)
    
    return doWork(ctx, task)
}

func estimateMemory(task Task) int64 {
    // оценка памяти для задачи
    return int64(len(task.Data) / 1024 / 1024) + 1
}
```

**Что происходит:**

- Не более 1000 МБ памяти занято одновременно.
- Большие задачи захватывают больше единиц.
- Малые — меньше.

### Пример: ограничение CPU

```go
var cpuSem = semaphore.NewWeighted(int64(runtime.GOMAXPROCS(0) * 100))

func processTask(ctx context.Context, task Task) error {
    weight := estimateCPU(task)  // 1-100 единиц
    
    if err := cpuSem.Acquire(ctx, weight); err != nil {
        return err
    }
    defer cpuSem.Release(weight)
    
    return doWork(ctx, task)
}
```

### Сравнение с обычным семафором

| Аспект | Обычный семафор | Взвешенный семафор |
|:---|:---|:---|
| Единица | 1 задача | N единиц |
| Гибкость | Фиксированная | Переменная |
| Использование | Однотипные задачи | Разные ресурсы |
| Реализация | `chan struct{}` | `semaphore.Weighted` |
| Стоимость | ~50-100 нс | ~200-500 нс |

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| `NewWeighted` | ~50-100 нс | ~100 байт |
| `Acquire` (свободно) | ~100-200 нс | 0 |
| `Acquire` (блокировка) | ~200-500 нс + ожидание | waiter |
| `Release` | ~100-300 нс | 0 |

### 💡 Практика: как использовать взвешенный семафор

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Взвешенный семафор** — для ресурсов с разной стоимостью (память, CPU).
2. **Оценка веса** — функция `estimateWeight(task)`.

**👍 СТОИТ СДЕЛАТЬ:**

3. **`Acquire(ctx, weight)`** — с отменой.
4. **`defer sem.Release(weight)`** — освобождение.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **`TryAcquire`** — если нужно без блокировки.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй взвешенный семафор для однотипных задач.** Обычный проще.
7. **Не забывай `Release`.** Утечка единиц.

### Ключевые выводы подглавы 9.2

- **Взвешенный семафор** — задача захватывает N единиц.
- **`semaphore.NewWeighted(n)`** — семафор на n единиц.
- **`Acquire(ctx, weight)`** — захват с блокировкой.
- **`Release(weight)`** — освобождение.
- **Использование:** ограничение памяти, CPU.

---

## 9.3 Rate limiter: token bucket

**Rate limiter** — примитив, ограничивающий **скорость** операций.

**Token bucket** — самый популярный алгоритм.

### Идея

**Token bucket** — это **ведро с токенами**:

- **Ёмкость** — максимум токенов.
- **Скорость пополнения** — сколько токенов добавляется в секунду.
- **Операция** — берёт 1 токен.
- **Если токенов нет** — операция ждёт.

### Схема

```
┌──────────────────────────────────────┐
│           Token Bucket               │
│                                      │
│  Ёмкость: 10 токенов                 │
│  Скорость: 100 токенов/сек           │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ ● ● ● ● ● ● ● ● ● ● │ 10 токенов│
│  └────────────────────────────────┘ │
│                                      │
│  Операция: взять 1 токен             │
│  Если токенов нет — ждать            │
└──────────────────────────────────────┘
```

### Свойства

- **Burst** — можно взять до `ёмкость` токенов сразу.
- **Rate** — в среднем не больше `скорость` токенов/сек.
- **Smooth** — токены пополняются постепенно.

### Пример

```
Ёмкость: 10, Скорость: 100/сек

t=0:    10 токенов
         10 операций сразу → 0 токенов
t=0.01: 1 токен (100/сек × 0.01 сек)
         1 операция
t=0.02: 1 токен
         1 операция
...
```

**Результат:** burst из 10 операций, потом ~100/сек.

### Реализация с нуля

```go
type TokenBucket struct {
    capacity   int           // ёмкость
    tokens     int           // текущее число токенов
    rate       float64       // токенов в секунду
    lastRefill time.Time     // время последнего пополнения
    mu         sync.Mutex
}

func NewTokenBucket(capacity int, rate float64) *TokenBucket {
    return &TokenBucket{
        capacity:   capacity,
        tokens:     capacity,
        rate:       rate,
        lastRefill: time.Now(),
    }
}

func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()
    
    // Пополнить токены
    now := time.Now()
    elapsed := now.Sub(tb.lastRefill).Seconds()
    tb.tokens += int(elapsed * tb.rate)
    if tb.tokens > tb.capacity {
        tb.tokens = tb.capacity
    }
    tb.lastRefill = now
    
    // Взять токен
    if tb.tokens > 0 {
        tb.tokens--
        return true
    }
    return false
}
```

**Проблема:** `Allow()` возвращает `false`, если токенов нет. Не блокируется.

**Для блокирующего варианта** нужен `Wait(ctx)`:

```go
func (tb *TokenBucket) Wait(ctx context.Context) error {
    for {
        if tb.Allow() {
            return nil
        }
        
        // Ждём немного
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(10 * time.Millisecond):
        }
    }
}
```

**Проблема:** polling. Лучше использовать `golang.org/x/time/rate`.

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| `Allow` | ~50-100 нс | 0 |
| `Wait` | зависит | 0 |

### 💡 Практика: как использовать token bucket

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Token bucket** — для ограничения скорости.
2. **`golang.org/x/time/rate`** — production-ready реализация (см. 9.5).

**👍 СТОИТ СДЕЛАТЬ:**

3. **Burst** = 1-10× rate для сглаживания пиков.
4. **`Wait(ctx)`** — с отменой.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **Своя реализация** — если нужна специфичная логика.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй polling для `Wait`.** Лучше `rate.Limiter`.
7. **Не игнорируй `ctx.Done()`.** Отмена не сработает.

### Ключевые выводы подглавы 9.3

- **Token bucket** — ведро с токенами.
- **Ёмкость** — burst. **Скорость** — rate.
- **`Allow()`** — без блокировки. **`Wait(ctx)`** — с блокировкой.
- **`golang.org/x/time/rate`** — production-ready.

---

## 9.4 Rate limiter: leaky bucket

**Leaky bucket** — альтернативный алгоритм.

### Идея

**Leaky bucket** — это **ведро с дыркой**:

- **Ёмкость** — максимум запросов в очереди.
- **Скорость утечки** — сколько запросов обрабатывается в секунду.
- **Операция** — кладёт запрос в ведро.
- **Если ведро полно** — запрос отбрасывается.

### Схема

```
┌──────────────────────────────────────┐
│           Leaky Bucket               │
│                                      │
│  Ёмкость: 10 запросов                │
│  Скорость утечки: 100/сек            │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ ● ● ● ● ● ● ● ● ● ● │ 10 запросов│
│  └────────────────────────────────┘ │
│         │                            │
│         │ утечка 100/сек             │
│         ▼                            │
│  ┌────────────────────────────────┐ │
│  │ Обработка                      │ │
│  └────────────────────────────────┘ │
└──────────────────────────────────────┘
```

### Отличие от token bucket

| Аспект | Token bucket | Leaky bucket |
|:---|:---|:---|
| Burst | Разрешён (до ёмкости) | Не разрешён |
| Скорость | Средняя = rate | Фиксированная |
| Очередь | Токены | Запросы |
| При переполнении | Ждать токен | Отбросить запрос |

**Token bucket** — burst разрешён. **Leaky bucket** — burst не разрешён, скорость фиксирована.

### Когда использовать

**Token bucket:**

- **Burst допустим** (например, клиент может иногда сделать 10 запросов сразу).
- **Средняя скорость важна.**

**Leaky bucket:**

- **Burst недопустим** (например, видео-стриминг).
- **Фиксированная скорость важна.**

### Реализация с нуля

```go
type LeakyBucket struct {
    capacity int
    queue    chan struct{}
    rate     time.Duration
    stop     chan struct{}
}

func NewLeakyBucket(capacity int, rate time.Duration) *LeakyBucket {
    lb := &LeakyBucket{
        capacity: capacity,
        queue:    make(chan struct{}, capacity),
        rate:     rate,
        stop:     make(chan struct{}),
    }
    go lb.drain()
    return lb
}

func (lb *LeakyBucket) drain() {
    ticker := time.NewTicker(lb.rate)
    defer ticker.Stop()
    
    for {
        select {
        case <-lb.stop:
            return
        case <-ticker.C:
            select {
            case <-lb.queue:
            default:
            }
        }
    }
}

func (lb *LeakyBucket) Allow() bool {
    select {
    case lb.queue <- struct{}{}:
        return true
    default:
        return false
    }
}
```

**Что происходит:**

- `queue` — буфер на `capacity` запросов.
- `drain` — горутина, которая каждые `rate` забирает 1 запрос.
- `Allow` — пытается положить в очередь; если полна — `false`.

### Пример: ограничение скорости обработки

```go
lb := NewLeakyBucket(10, 10*time.Millisecond)  // 100/сек

for _, item := range items {
    if !lb.Allow() {
        // отбросить или подождать
        metrics.Dropped.Add(1)
        continue
    }
    process(item)
}
```

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| `Allow` (есть место) | ~50-100 нс | 0 |
| `Allow` (полно) | ~50-100 нс | 0 |
| `drain` | ~1-10 мкс | 0 |

### 💡 Практика: как использовать leaky bucket

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Leaky bucket** — если burst недопустим.
2. **`Allow()`** — если можно отбрасывать.

**👍 СТОИТ СДЕЛАТЬ:**

3. **`Wait(ctx)`** — если нельзя отбрасывать.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

4. **Своя реализация** — если нужна специфичная логика.

**❌ НЕ ДЕЛАЙ:**

5. **Не путай с token bucket.** Leaky bucket не разрешает burst.
6. **Не забывай про `stop`** для `drain`.

### Ключевые выводы подглавы 9.4

- **Leaky bucket** — ведро с дыркой.
- **Burst не разрешён.** Скорость фиксирована.
- **`Allow()`** — без блокировки. **`Wait(ctx)`** — с блокировкой.
- **Использование:** видео-стриминг, фиксированная скорость.

---

## 9.5 `golang.org/x/time/rate` изнутри

**`golang.org/x/time/rate`** — production-ready rate limiter.

### API

```go
import "golang.org/x/time/rate"

limiter := rate.NewLimiter(rate.Limit(100), 10)  // 100/сек, burst 10

// Allow — без блокировки
if limiter.Allow() {
    // ...
}

// Wait — с блокировкой
if err := limiter.Wait(ctx); err != nil {
    return err
}
```

### Параметры

- **`rate.Limit(100)`** — 100 событий в секунду.
- **`10`** — burst (максимум токенов).

### Основные методы

| Метод | Что делает |
|:---|:---|
| `Allow()` | Проверить без блокировки |
| `AllowN(t, n)` | Проверить n событий в момент t |
| `Wait(ctx)` | Ждать 1 событие |
| `WaitN(ctx, n)` | Ждать n событий |
| `Reserve()` | Зарезервировать (с задержкой) |
| `SetLimit(l)` | Установить новый rate |
| `SetBurst(b)` | Установить новый burst |

### Пример: ограничение HTTP-клиента

```go
var limiter = rate.NewLimiter(rate.Limit(100), 10)

func callAPI(ctx context.Context, url string) (*http.Response, error) {
    if err := limiter.Wait(ctx); err != nil {
        return nil, err
    }
    
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    return http.DefaultClient.Do(req)
}
```

**Что происходит:**

- Не более 100 запросов/сек.
- Burst 10 — можно 10 запросов сразу.
- `Wait` блокируется, если лимит исчерпан.

### Как устроен внутри

`rate.Limiter` использует **token bucket**:

```go
type Limiter struct {
    limit  Limit
    burst  int
    tokens float64
    last   time.Time
    mu     sync.Mutex
}
```

**`Wait`:**

```
1. Захватить mu.
2. Пополнить токены (по времени).
3. Если tokens >= 1:
   - tokens -= 1.
   - Освободить mu.
   - return nil.
4. Иначе:
   - Вычислить задержку.
   - Освободить mu.
   - Ждать задержку (через timer).
```

### Динамическое изменение

```go
limiter.SetLimit(rate.Limit(200))  // увеличить до 200/сек
limiter.SetBurst(20)               // увеличить burst
```

**Использование:** адаптация к нагрузке.

### Пример: адаптивный rate limiter

```go
type AdaptiveLimiter struct {
    limiter *rate.Limiter
    mu      sync.Mutex
}

func (a *AdaptiveLimiter) Adjust(errorRate float64) {
    a.mu.Lock()
    defer a.mu.Unlock()
    
    if errorRate > 0.1 {
        // Много ошибок — снизить
        a.limiter.SetLimit(a.limiter.Limit() / 2)
    } else if errorRate < 0.01 {
        // Мало ошибок — повысить
        a.limiter.SetLimit(a.limiter.Limit() * 2)
    }
}
```

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| `NewLimiter` | ~50-100 нс | ~100 байт |
| `Allow` | ~50-100 нс | 0 |
| `Wait` (есть токен) | ~100-200 нс | 0 |
| `Wait` (нет токена) | ~100-200 нс + задержка | 0 |

### 💡 Практика: как использовать rate.Limiter

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`rate.NewLimiter(rate.Limit(N), burst)`** — для ограничения скорости.
2. **`Wait(ctx)`** — с отменой.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Burst = 1-10× rate** для сглаживания пиков.
4. **`SetLimit` / `SetBurst`** — для адаптации.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **Свой limiter** — если нужна специфичная логика.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `Allow` в цикле** — busy loop.
7. **Не забывай про `ctx`.** Отмена не сработает.

### Ключевые выводы подглавы 9.5

- **`golang.org/x/time/rate`** — production-ready rate limiter.
- **Token bucket** внутри.
- **`Wait(ctx)`** — с блокировкой. **`Allow()`** — без.
- **`SetLimit` / `SetBurst`** — динамическое изменение.
- **Burst = 1-10× rate.**

---

## 9.6 Circuit breaker: защита от каскадных отказов

**Circuit breaker** — паттерн, который **прекращает** вызовы к падающему сервису.

### Проблема: каскадные отказы

Представь:

- **Сервис A** вызывает **сервис B**.
- **Сервис B** начал падать (медленно, ошибки).
- **Сервис A** ждёт B, накапливает горутины.
- **Сервис A** тоже падает.
- **Клиенты A** падают.

**Каскадный отказ.** Один сервис тянет за собой остальные.

### Идея circuit breaker

**Circuit breaker** — **конечный автомат** с тремя состояниями:

```
         ┌──────────┐
         │  Closed  │  ← нормальная работа
         └────┬─────┘
              │ много ошибок
              ▼
         ┌──────────┐
         │   Open   │  ← запросы блокируются
         └────┬─────┘
              │ таймаут
              ▼
         ┌──────────┐
         │ Half-Open│  ← пропускаем 1 запрос
         └────┬─────┘
              │ успех → Closed
              │ ошибка → Open
```

### Состояния

| Состояние | Что делает |
|:---|:---|
| **Closed** | Пропускает запросы. Считает ошибки. |
| **Open** | Блокирует запросы. Ждёт таймаут. |
| **Half-Open** | Пропускает 1 запрос. Проверяет. |

### Переходы

- **Closed → Open:** когда ошибок > threshold.
- **Open → Half-Open:** через timeout.
- **Half-Open → Closed:** если запрос успешен.
- **Half-Open → Open:** если запрос упал.

### Реализация с нуля

```go
type State int

const (
    StateClosed State = iota
    StateOpen
    StateHalfOpen
)

type CircuitBreaker struct {
    mu           sync.Mutex
    state        State
    failures     int
    successes    int
    threshold    int           // ошибок до Open    timeout      time.Duration // время до Half-Open
    lastFailure  time.Time
    halfOpenMax  int           // запросов в Half-Open
}

func NewCircuitBreaker(threshold int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        state:       StateClosed,
        threshold:   threshold,
        timeout:     timeout,
        halfOpenMax: 1,
    }
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    cb.mu.Lock()
    
    switch cb.state {
    case StateOpen:
        if time.Since(cb.lastFailure) > cb.timeout {
            cb.state = StateHalfOpen
            cb.successes = 0
        } else {
            cb.mu.Unlock()
            return errors.New("circuit breaker is open")
        }
    case StateHalfOpen:
        if cb.successes >= cb.halfOpenMax {
            cb.mu.Unlock()
            return errors.New("circuit breaker is half-open (limit reached)")
        }
    }
    
    cb.mu.Unlock()
    
    err := fn()
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()
        
        switch cb.state {
        case StateClosed:
            if cb.failures >= cb.threshold {
                cb.state = StateOpen
            }
        case StateHalfOpen:
            cb.state = StateOpen
        }
        return err
    }
    
    // Успех
    switch cb.state {
    case StateClosed:
        cb.failures = 0
    case StateHalfOpen:
        cb.successes++
        if cb.successes >= cb.halfOpenMax {
            cb.state = StateClosed
            cb.failures = 0
        }
    }
    return nil
}
```

### Пример использования

```go
cb := NewCircuitBreaker(5, 30*time.Second)

for _, url := range urls {
    err := cb.Call(func() error {
        resp, err := http.Get(url)
        if err != nil {
            return err
        }
        defer resp.Body.Close()
        if resp.StatusCode >= 500 {
            return fmt.Errorf("server error: %d", resp.StatusCode)
        }
        return nil
    })
    
    if err != nil {
        log.Printf("error: %v", err)
    }
}
```

**Что происходит:**

- Первые 5 ошибок — пропускаются.
- 5 ошибок → `Open`.
- 30 секунд — запросы блокируются.
- Через 30 секунд → `Half-Open`.
- 1 запрос — если успешен → `Closed`.

### Библиотеки

**`github.com/sony/gobreaker`:**

```go
import "github.com/sony/gobreaker"

cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "my-service",
    MaxRequests: 3,
    Interval:    10 * time.Second,
    Timeout:     30 * time.Second,
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        return counts.ConsecutiveFailures > 5
    },
})

result, err := cb.Execute(func() (interface{}, error) {
    return callAPI()
})
```

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| `Call` (Closed) | ~100-200 нс | 0 |
| `Call` (Open) | ~50-100 нс | 0 |
| `Call` (Half-Open) | ~100-200 нс | 0 |

### 💡 Практика: как использовать circuit breaker

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Circuit breaker** для вызовов внешних сервисов.
2. **Threshold** — 5-10 ошибок.
3. **Timeout** — 10-60 секунд.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`github.com/sony/gobreaker`** — production-ready.
5. **Метрики** — сколько раз открылся, сколько запросов заблокировано.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Half-Open max = 1-3** — для проверки.

**❌ НЕ ДЕЛАЙ:**

7. **Не ставь threshold = 1.** Одна случайная ошибка откроет breaker.
8. **Не ставь timeout = 0.** Breaker никогда не закроется.
9. **Не игнорируй ошибки в Half-Open.** Breaker должен открыться.

### Ключевые выводы подглавы 9.6

- **Circuit breaker** — конечный автомат: Closed, Open, Half-Open.
- **Closed → Open** при threshold ошибок.
- **Open → Half-Open** через timeout.
- **Half-Open → Closed** при успехе.
- **`github.com/sony/gobreaker`** — production-ready.

---

## 9.7 Bulkhead: изоляция сбоев

**Bulkhead** — паттерн, который **изолирует** ресурсы для разных типов задач.

### Проблема: один тип задач блокирует другие

Представь:

- **Сервис** обрабатывает **HTTP-запросы** и **фоновые задачи**.
- **HTTP-запросы** делают запросы к **медленному API**.
- **Медленный API** перегружен.
- **HTTP-горутины** занимают все ресурсы.
- **Фоновые задачи** не могут выполниться.

**Проблема:** один тип задач **блокирует** другой.

### Идея bulkhead

**Bulkhead** — разделение ресурсов на **изолированные пулы**.

```
┌─────────────────────────────────────┐
│             Сервис                   │
│                                      │
│  ┌────────────┐    ┌────────────┐  │
│  │ HTTP Pool  │    │ Bg Pool    │  │
│  │ (100 слотов)│    │ (50 слотов)│  │
│  └────────────┘    └────────────┘  │
│                                      │
│  HTTP-запросы не могут занять        │
│  слоты из Bg Pool                    │
└─────────────────────────────────────┘
```

### Пример

```go
type Bulkhead struct {
    httpSem chan struct{}
    bgSem   chan struct{}
}

func NewBulkhead(httpLimit, bgLimit int) *Bulkhead {
    return &Bulkhead{
        httpSem: make(chan struct{}, httpLimit),
        bgSem:   make(chan struct{}, bgLimit),
    }
}

func (b *Bulkhead) HandleHTTP(ctx context.Context, fn func() error) error {
    select {
    case b.httpSem <- struct{}{}:
        defer func() { <-b.httpSem }()
    case <-ctx.Done():
        return ctx.Err()
    }
    return fn()
}

func (b *Bulkhead) HandleBackground(ctx context.Context, fn func() error) error {
    select {
    case b.bgSem <- struct{}{}:
        defer func() { <-b.bgSem }()
    case <-ctx.Done():
        return ctx.Err()
    }
    return fn()
}
```

**Что происходит:**

- HTTP-запросы используют `httpSem`.
- Фоновые задачи используют `bgSem`.
- Они **не конкурируют** за слоты.

### Пример с разными лимитами

```go
bulkhead := NewBulkhead(100, 50)

// HTTP-хэндлер
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    err := bulkhead.HandleHTTP(r.Context(), func() error {
        // обработка
        return nil
    })
    if err != nil {
        http.Error(w, err.Error(), 503)
    }
})

// Фоновая задача
go func() {
    for {
        err := bulkhead.HandleBackground(context.Background(), func() error {
            // обработка
            return nil
        })
        if err != nil {
            log.Printf("error: %v", err)
        }
    }
}()
```

### Bulkhead vs семафор

| Аспект | Семафор | Bulkhead |
|:---|:---|:---|
| Ресурсов | Один | Несколько |
| Изоляция | Нет | Да |
| Использование | Ограничить один ресурс | Изолировать типы задач |

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| `HandleHTTP` | ~50-100 нс | 0 |
| `HandleBackground` | ~50-100 нс | 0 |

### 💡 Практика: как использовать bulkhead

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Bulkhead** для изоляции типов задач.
2. **Разные лимиты** для разных пулов.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Метрики** — использование каждого пула.
4. **`select` с `ctx.Done()`** — для отмены.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **Динамические лимиты** — для адаптации.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй один пул для всего.** Изоляция важна.
7. **Не ставь слишком маленькие лимиты.** Голодание.

### Ключевые выводы подглавы 9.7

- **Bulkhead** — изоляция ресурсов для разных типов задач.
- **Разные пулы** — HTTP, фоновые, БД.
- **Изоляция** предотвращает каскадные отказы.
- **Разные лимиты** для разных пулов.

---

## 9.8 Retry с backoff и jitter

**Retry** — повторная попытка при ошибке. **Backoff** — увеличение задержки. **Jitter** — случайность для избежания синхронизации.

### Проблема: retry storm

**Наивный retry:**

```go
for i := 0; i < 3; i++ {
    err := call()
    if err == nil {
        return nil
    }
    time.Sleep(1 * time.Second)  // ← все retry одновременно
}
```

**Что происходит:**

- 1000 клиентов делают retry.
- Все ждут 1 секунду.
- Все retry **одновременно**.
- API получает 1000 запросов снова.
- **Retry storm.**

### Решение: backoff + jitter

**Backoff:** задержка растёт экспоненциально.

```
1s → 2s → 4s → 8s → ...
```

**Jitter:** случайное отклонение.

```
1s ± 0.5s → 2s ± 1s → 4s ± 2s → ...
```

**Результат:** retry **размазаны** по времени. API не перегружен.

### Реализация

```go
func retryWithBackoff(ctx context.Context, fn func() error) error {
    const (
        maxRetries = 5
        baseDelay  = 1 * time.Second
        maxDelay   = 30 * time.Second
    )
    
    for i := 0; i < maxRetries; i++ {
        err := fn()
        if err == nil {
            return nil
        }
        
        if i == maxRetries-1 {
            return err
        }
        
        // Exponential backoff
        delay := baseDelay * time.Duration(1<<uint(i))
        if delay > maxDelay {
            delay = maxDelay
        }
        
        // Jitter (full jitter)
        jitter := time.Duration(rand.Int63n(int64(delay)))
        
        select {
        case <-time.After(jitter):
        case <-ctx.Done():
            return ctx.Err()
        }
    }
    return nil
}
```

### Типы jitter

**Full jitter:**

```go
jitter := time.Duration(rand.Int63n(int64(delay)))
```

**Equal jitter:**

```go
half := delay / 2
jitter := half + time.Duration(rand.Int63n(int64(half)))
```

**Decorrelated jitter:**

```go
jitter := time.Duration(rand.Int63n(int64(delay * 3)))
```

**Рекомендация:** **full jitter** — простой и эффективный.

### Пример использования

```go
err := retryWithBackoff(ctx, func() error {
    resp, err := http.Get(url)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode == 429 || resp.StatusCode >= 500 {
        return fmt.Errorf("retryable: %d", resp.StatusCode)
    }
    if resp.StatusCode >= 400 {
        return fmt.Errorf("non-retryable: %d", resp.StatusCode)  // не retry
    }
    return nil
})
```

**Что происходит:**

- `429` (Too Many Requests) — retry.
- `5xx` — retry.
- `4xx` (кроме 429) — не retry.

### Что можно retry, а что нельзя

**Можно retry:**

- **Сетевые ошибки** (timeout, connection refused).
- **5xx** (серверные ошибки).
- **429** (rate limit).
- **Idempotent** операции (GET, PUT, DELETE).

**Нельзя retry:**

- **4xx** (кроме 429) — клиентская ошибка.
- **Non-idempotent** операции (POST) без idempotency key.
- **Ошибки валидации.**

### Библиотеки

**`github.com/cenkalti/backoff`:**

```go
import "github.com/cenkalti/backoff/v4"

operation := func() error {
    return callAPI()
}

err := backoff.Retry(operation, backoff.NewExponentialBackOff())
```

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| Retry с backoff | зависит | 0 |
| Jitter | ~10-50 нс | 0 |

### 💡 Практика: как использовать retry

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Backoff** — экспоненциальный.
2. **Jitter** — full jitter.
3. **Max retries** — 3-5.
4. **`ctx.Done()`** — для отмены.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Retry только для retryable ошибок** (5xx, 429, network).
6. **Idempotency key** — для non-idempotent операций.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **`github.com/cenkalti/backoff`** — production-ready.

**❌ НЕ ДЕЛАЙ:**

8. **Не retry без backoff.** Retry storm.
9. **Не retry без jitter.** Синхронизация.
10. **Не retry non-idempotent операции** без idempotency key.
11. **Не retry 4xx** (кроме 429).

### Ключевые выводы подглавы 9.8

- **Retry** — повторная попытка при ошибке.
- **Backoff** — экспоненциальная задержка.
- **Jitter** — случайность для избежания синхронизации.
- **Full jitter** — рекомендуемый.
- **Retry только retryable ошибок.**

---

## 9.9 Комбинирование паттернов

Разберём, **как комбинировать** паттерны.

### Схема production-клиента

```
┌─────────────────────────────────────────────────────────────────┐
│                    Production HTTP Client                        │
│                                                                  │
│  1. Bulkhead (изоляция)                                          │
│     └─▶ 2. Rate Limiter (ограничение скорости)                  │
│         └─▶ 3. Circuit Breaker (защита от отказов)              │
│             └─▶ 4. Retry с backoff (повтор при ошибке)          │
│                 └─▶ 5. Семафор (ограничение параллелизма)       │
│                     └─▶ HTTP-запрос                              │
└─────────────────────────────────────────────────────────────────┘
```

### Порядок применения

**Снаружи внутрь:**

1. **Bulkhead** — изолировать тип задач.
2. **Rate Limiter** — ограничить скорость.
3. **Circuit Breaker** — не долбить падающий сервис.
4. **Retry** — повторить при временной ошибке.
5. **Семафор** — ограничить одновременные запросы.

### Пример: полный клиент

```go
type ProductionClient struct {
    httpClient     *http.Client
    limiter        *rate.Limiter
    breaker        *gobreaker.CircuitBreaker
    sem            chan struct{}
}

func NewProductionClient() *ProductionClient {
    return &ProductionClient{
        httpClient: &http.Client{Timeout: 10 * time.Second},
        limiter:    rate.NewLimiter(rate.Limit(100), 10),
        breaker:    gobreaker.NewCircuitBreaker(gobreaker.Settings{
            Name:        "api",
            MaxRequests: 3,
            Timeout:     30 * time.Second,
            ReadyToTrip: func(counts gobreaker.Counts) bool {
                return counts.ConsecutiveFailures > 5
            },
        }),
        sem: make(chan struct{}, 50),
    }
}

func (c *ProductionClient) Get(ctx context.Context, url string) (*http.Response, error) {
    // 1. Rate limit
    if err := c.limiter.Wait(ctx); err != nil {
        return nil, err
    }
    
    // 2. Semaphore
    select {
    case c.sem <- struct{}{}:
        defer func() { <-c.sem }()
    case <-ctx.Done():
        return nil, ctx.Err()
    }
    
    // 3. Circuit breaker + retry
    var resp *http.Response
    err := retryWithBackoff(ctx, func() error {
        result, err := c.breaker.Execute(func() (interface{}, error) {
            req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
            if err != nil {
                return nil, err
            }
            r, err := c.httpClient.Do(req)
            if err != nil {
                return nil, err
            }
            if r.StatusCode >= 500 || r.StatusCode == 429 {
                return nil, fmt.Errorf("retryable: %d", r.StatusCode)
            }
            if r.StatusCode >= 400 {
                return nil, fmt.Errorf("non-retryable: %d", r.StatusCode)
            }
            return r, nil
        })
        if err != nil {
            return err
        }
        resp = result.(*http.Response)
        return nil
    })
    
    return resp, err
}
```

### Что происходит

1. **Rate limit** — не более 100/сек.
2. **Semaphore** — не более 50 одновременных.
3. **Circuit breaker** — блокирует при 5 ошибках.
4. **Retry** — повторяет с backoff.
5. **HTTP-запрос.**

### Метрики

```go
type Metrics struct {
    RateLimited    atomic.Int64
    SemaphoreWait  atomic.Int64
    BreakerOpen    atomic.Int64
    Retries        atomic.Int64
    Successes      atomic.Int64
    Failures       atomic.Int64
}
```

### Аннотация сложности

| Паттерн | Overhead |
|:---|:---|
| Bulkhead | ~50-100 нс |
| Rate limiter | ~50-100 нс |
| Circuit breaker | ~100-200 нс |
| Retry | зависит |
| Semaphore | ~50-100 нс |
| **Итого** | ~300-500 нс |

### 💡 Практика: как комбинировать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Bulkhead → Rate Limiter → Circuit Breaker → Retry → Semaphore.**
2. **Метрики на каждом слое.**
3. **`context.Context` во всех слоях.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Логирование** — какие слои сработали.
5. **Tracing** — для диагностики.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Динамическая настройка** — для адаптации.

**❌ НЕ ДЕЛАЙ:**

7. **Не применяй все паттерны бездумно.** Только те, что нужны.
8. **Не забывай метрики.** Без них не видно проблем.

### Ключевые выводы подглавы 9.9

- **Комбинирование:** Bulkhead → Rate Limiter → Circuit Breaker → Retry → Semaphore.
- **Overhead:** ~300-500 нс на все слои.
- **Метрики** на каждом слое.
- **`context.Context`** во всех слоях.

---

## 9.10 Практика Go: production-ready клиент

Напишем **production-ready HTTP-клиент** со всеми паттернами.

### Полный код

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "math/rand"
    "net/http"
    "sync/atomic"
    "time"
    
    "github.com/sony/gobreaker"
    "golang.org/x/time/rate"
)

type Metrics struct {
    Requests       atomic.Int64
    Successes      atomic.Int64
    Failures       atomic.Int64
    RateLimited    atomic.Int64
    SemaphoreWait  atomic.Int64
    BreakerOpen    atomic.Int64
    Retries        atomic.Int64
    TotalLatency   atomic.Int64
}

type Client struct {
    httpClient *http.Client
    limiter    *rate.Limiter
    breaker    *gobreaker.CircuitBreaker
    sem        chan struct{}
    metrics    *Metrics
}

func NewClient(metrics *Metrics) *Client {
    return &Client{
        httpClient: &http.Client{Timeout: 10 * time.Second},
        limiter:    rate.NewLimiter(rate.Limit(100), 10),
        breaker: gobreaker.NewCircuitBreaker(gobreaker.Settings{
            Name:        "api",
            MaxRequests: 3,
            Timeout:     30 * time.Second,
            ReadyToTrip: func(counts gobreaker.Counts) bool {
                return counts.ConsecutiveFailures > 5
            },
            OnStateChange: func(name string, from, to gobreaker.State) {
                fmt.Printf("breaker %s: %s → %s\n", name, from, to)
            },
        }),
        sem:     make(chan struct{}, 50),
        metrics: metrics,
    }
}

func (c *Client) Get(ctx context.Context, url string) (*http.Response, error) {
    c.metrics.Requests.Add(1)
    start := time.Now()
    defer func() {
        c.metrics.TotalLatency.Add(int64(time.Since(start)))
    }()
    
    // 1. Rate limit
    if err := c.limiter.Wait(ctx); err != nil {
        c.metrics.RateLimited.Add(1)
        return nil, err
    }
    
    // 2. Semaphore
    select {
    case c.sem <- struct{}{}:
        defer func() { <-c.sem }()
    case <-ctx.Done():
        c.metrics.SemaphoreWait.Add(1)
        return nil, ctx.Err()
    }
    
    // 3. Circuit breaker + retry
    var resp *http.Response
    err := c.retryWithBackoff(ctx, func() error {
        result, err := c.breaker.Execute(func() (interface{}, error) {
            req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
            if err != nil {
                return nil, err
            }
            r, err := c.httpClient.Do(req)
            if err != nil {
                return nil, err
            }
            if r.StatusCode >= 500 || r.StatusCode == 429 {
                return nil, fmt.Errorf("retryable: %d", r.StatusCode)
            }
            if r.StatusCode >= 400 {
                return nil, fmt.Errorf("non-retryable: %d", r.StatusCode)
            }
            return r, nil
        })
        if err != nil {
            if errors.Is(err, gobreaker.ErrOpenState) {
                c.metrics.BreakerOpen.Add(1)
            }
            return err
        }
        resp = result.(*http.Response)
        return nil
    })
    
    if err != nil {
        c.metrics.Failures.Add(1)
        return nil, err
    }
    c.metrics.Successes.Add(1)
    return resp, nil
}

func (c *Client) retryWithBackoff(ctx context.Context, fn func() error) error {
    const (
        maxRetries = 3
        baseDelay  = 100 * time.Millisecond
        maxDelay   = 5 * time.Second
    )
    
    var lastErr error
    for i := 0; i < maxRetries; i++ {
        if i > 0 {
            c.metrics.Retries.Add(1)
        }
        
        lastErr = fn()
        if lastErr == nil {
            return nil
        }
        
        // Не retry для non-retryable
        if !isRetryable(lastErr) {
            return lastErr
        }
        
        if i == maxRetries-1 {
            break
        }
        
        delay := baseDelay * time.Duration(1<<uint(i))
        if delay > maxDelay {
            delay = maxDelay
        }
        
        // Full jitter
        jitter := time.Duration(rand.Int63n(int64(delay)))
        
        select {
        case <-time.After(jitter):
        case <-ctx.Done():
            return ctx.Err()
        }
    }
    return lastErr
}

func isRetryable(err error) bool {
    return !errors.Is(err, gobreaker.ErrOpenState) &&
        !errors.Is(err, context.Canceled) &&
        !errors.Is(err, context.DeadlineExceeded)
}

func main() {
    metrics := &Metrics{}
    client := NewClient(metrics)
    
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    // Запускаем 1000 запросов
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            resp, err := client.Get(ctx, "https://api.example.com/data")
            if err != nil {
                return
            }
            defer resp.Body.Close()
        }(i)
    }
    wg.Wait()
    
    fmt.Printf("Requests:      %d\n", metrics.Requests.Load())
    fmt.Printf("Successes:     %d\n", metrics.Successes.Load())
    fmt.Printf("Failures:      %d\n", metrics.Failures.Load())
    fmt.Printf("Rate limited:  %d\n", metrics.RateLimited.Load())
    fmt.Printf("Semaphore wait: %d\n", metrics.SemaphoreWait.Load())
    fmt.Printf("Breaker open:  %d\n", metrics.BreakerOpen.Load())
    fmt.Printf("Retries:       %d\n", metrics.Retries.Load())
    if metrics.Requests.Load() > 0 {
        avg := time.Duration(metrics.TotalLatency.Load() / metrics.Requests.Load())
        fmt.Printf("Avg latency:   %v\n", avg)
    }
}
```

### Что демонстрирует

1. **Rate limiter** — 100/сек, burst 10.
2. **Semaphore** — 50 одновременных.
3. **Circuit breaker** — 5 ошибок → Open.
4. **Retry** — 3 попытки с backoff + jitter.
5. **Метрики** на каждом слое.
6. **Отмена** через `context`.

### Аннотация сложности

| Слой | Overhead |
|:---|:---|
| Rate limiter | ~50-100 нс |
| Semaphore | ~50-100 нс |
| Circuit breaker | ~100-200 нс |
| Retry | зависит |
| **Итого** | ~300-500 нс |

### 💡 Практика: как строить production-клиент

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Все слои:** rate limiter, semaphore, circuit breaker, retry.
2. **Метрики** на каждом слое.
3. **`context.Context`** во всех вызовах.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Логирование** переходов circuit breaker.
5. **Экспорт метрик в Prometheus.**

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Адаптивные лимиты** — для переменной нагрузки.

**❌ НЕ ДЕЛАЙ:**

7. **Не применяй все паттерны сразу** — начни с rate limiter.
8. **Не забывай про метрики.**

### Ключевые выводы подглавы 9.10

- **Production-клиент** комбинирует все паттерны.
- **Метрики** на каждом слое.
- **Overhead** ~300-500 нс.
- **`context.Context`** — отмена.

---

## 9.11 Выводы и типичные ошибки

**Что мы узнали?**

Семафор — ограничение доступа к ресурсу через буферизованный канал. Взвешенный семафор — задача захватывает N единиц. Rate limiter — token bucket (burst разрешён) и leaky bucket (burst не разрешён). `golang.org/x/time/rate` — production-ready. Circuit breaker — конечный автомат (Closed, Open, Half-Open) для защиты от каскадных отказов. Bulkhead — изоляция ресурсов для разных типов задач. Retry с backoff + jitter — для временных ошибок. Комбинирование: Bulkhead → Rate Limiter → Circuit Breaker → Retry → Semaphore.

**Типичные ошибки:**

- ❌ **Использовать семафор для многих однотипных задач.** Worker pool лучше.
- ❌ **Забывать освобождать слот семафора.** Утечка.
- ❌ **Не использовать взвешенный семафор для ресурсов с разной стоимостью.** OOM.
- ❌ **Retry без backoff.** Retry storm.
- ❌ **Retry без jitter.** Синхронизация.
- ❌ **Retry non-idempotent операции** без idempotency key.
- ❌ **Retry 4xx** (кроме 429).
- ❌ **Circuit breaker с threshold = 1.** Одна ошибка откроет.
- ❌ **Circuit breaker без timeout.** Никогда не закроется.
- ❌ **Один пул для всех типов задач.** Нет изоляции.
- ❌ **Не использовать `context.Context`.** Отмена не сработает.
- ❌ **Не измерять метрики.** Не видно проблем.
- ❌ **Применять все паттерны бездумно.** Только те, что нужны.
- ❌ **Не логировать переходы circuit breaker.** Не видно проблем.

---

## 9.12 Для быстрого повторения

- **Семафор** — ограничение доступа к ресурсу. `chan struct{}`.
- **Взвешенный семафор** — `semaphore.NewWeighted(n)`. Задача захватывает N единиц.
- **Token bucket** — ведро с токенами. Burst разрешён.
- **Leaky bucket** — ведро с дыркой. Burst не разрешён.
- **`golang.org/x/time/rate`** — production-ready rate limiter.
- **`Wait(ctx)`** — блокируется. **`Allow()`** — не блокируется.
- **Burst** = 1-10× rate.
- **Circuit breaker** — Closed, Open, Half-Open.
- **Threshold** = 5-10 ошибок. **Timeout** = 10-60 сек.
- **Bulkhead** — изоляция ресурсов. Разные пулы.
- **Retry** — backoff (экспоненциальный) + jitter (full).
- **Max retries** = 3-5.
- **Retry только retryable ошибок** (5xx, 429, network).
- **Комбинирование:** Bulkhead → Rate Limiter → Circuit Breaker → Retry → Semaphore.
- **Overhead** всех слоёв: ~300-500 нс.
- **Метрики** на каждом слое.
- **`context.Context`** во всех вызовах.

---

## 9.13 Вопросы для самопроверки

1. Чем семафор отличается от worker pool? Когда что использовать?
2. Что такое взвешенный семафор? Когда его использовать?
3. Что такое token bucket? Что такое burst?
4. Чем leaky bucket отличается от token bucket?
5. Что такое `golang.org/x/time/rate`? Как его использовать?
6. Что такое circuit breaker? Какие состояния?
7. Как работает переход Closed → Open → Half-Open → Closed?
8. Что такое bulkhead? Зачем нужен?
9. Что такое retry с backoff? Зачем jitter?
10. Какие ошибки можно retry, а какие нельзя?
11. Как комбинировать паттерны?
12. Что такое idempotency key? Зачем нужен?
13. Как измерять эффективность паттернов?
14. Что произойдёт, если circuit breaker с threshold = 1?
15. Почему нельзя retry non-idempotent операции без idempotency key?
16. Как настроить rate limiter для внешнего API?
17. Что такое full jitter?
18. Как избежать retry storm?

---

## 9.14 Ответы

### Ответ 1

**Семафор:** создаёт горутину на задачу. **Worker pool:** переиспользует N воркеров.

**Семафор** — для ограничения доступа к ресурсу (БД, HTTP, файлы), мало задач, разные задачи. **Worker pool** — для многих однотипных задач, производительность важна.

### Ответ 2

**Взвешенный семафор** — задача захватывает N единиц.

**Использование:** ресурсы с разной стоимостью (память, CPU). `semaphore.NewWeighted(n)`, `Acquire(ctx, weight)`, `Release(weight)`.

### Ответ 3

**Token bucket** — ведро с токенами. **Burst** — можно взять до ёмкости токенов сразу. **Rate** — средняя скорость.

**Пример:** ёмкость 10, скорость 100/сек. Burst 10, потом 100/сек.

### Ответ 4

**Token bucket:** burst разрешён (до ёмкости). **Leaky bucket:** burst не разрешён, скорость фиксирована.

**Token bucket** — для клиентов, которые могут иногда сделать много запросов. **Leaky bucket** — для видео-стриминга, фиксированной скорости.

### Ответ 5

**`golang.org/x/time/rate`** — production-ready rate limiter на token bucket.

```go
limiter := rate.NewLimiter(rate.Limit(100), 10)
limiter.Wait(ctx)
limiter.Allow()
limiter.SetLimit(rate.Limit(200))
```

### Ответ 6

**Circuit breaker** — конечный автомат для защиты от каскадных отказов.

**Состояния:**
- **Closed** — нормальная работа.
- **Open** — запросы блокируются.
- **Half-Open** — пропускаем 1 запрос.

### Ответ 7

- **Closed → Open:** ошибок > threshold.
- **Open → Half-Open:** через timeout.
- **Half-Open → Closed:** запрос успешен.
- **Half-Open → Open:** запрос упал.

### Ответ 8

**Bulkhead** — изоляция ресурсов для разных типов задач.

**Зачем:** предотвращает каскадные отказы. HTTP-запросы не блокируют фоновые задачи.

### Ответ 9

**Retry с backoff** — повторная попытка с увеличивающейся задержкой.

**Backoff:** 1s → 2s → 4s → 8s.

**Jitter:** случайное отклонение, чтобы retry не синхронизировались.

### Ответ 10

**Можно retry:**
- 5xx (серверные ошибки).
- 429 (rate limit).
- Сетевые ошибки (timeout, connection refused).
- Idempotent операции (GET, PUT, DELETE).

**Нельзя retry:**
- 4xx (кроме 429).
- Non-idempotent (POST) без idempotency key.
- Ошибки валидации.

### Ответ 11

**Комбинирование:** Bulkhead → Rate Limiter → Circuit Breaker → Retry → Semaphore.

**Снаружи внутрь:** изоляция → скорость → защита → retry → параллелизм.

### Ответ 12

**Idempotency key** — уникальный идентификатор запроса. Позволяет безопасно retry non-idempotent операции.

**Зачем:** если запрос выполнился, но ответ потерялся, retry с тем же ключом не выполнит операцию дважды.

### Ответ 13

**Метрики:**
- **Rate limited** — сколько запросов отклонено.
- **Semaphore wait** — сколько ждали.
- **Breaker open** — сколько заблокировано.
- **Retries** — сколько retry.
- **Successes / Failures** — результаты.
- **Avg latency** — среднее время.

### Ответ 14

**Circuit breaker с threshold = 1:** одна случайная ошибка откроет breaker. Сервис будет блокировать запросы из-за временного сбоя.

**Рекомендация:** threshold = 5-10.

### Ответ 15

**Non-idempotent операции** (POST) при retry могут выполниться дважды. Например, POST /orders создаст два заказа.

**Решение:** idempotency key. Сервер проверяет ключ и не выполняет операцию дважды.

### Ответ 16

**Rate limiter для API:**
1. Узнать rate limit API (например, 100/сек).
2. `rate.NewLimiter(rate.Limit(100), 10)`.
3. `limiter.Wait(ctx)` перед каждым запросом.
4. Burst = 1-10× rate.

### Ответ 17

**Full jitter** — случайное значение в диапазоне `[0, delay)`:

```go
jitter := time.Duration(rand.Int63n(int64(delay)))
```

**Рекомендуется** из-за простоты и эффективности.

### Ответ 18

**Retry storm** — все клиенты retry одновременно.

**Избежать:**
1. **Backoff** — экспоненциальная задержка.
2. **Jitter** — случайное отклонение.
3. **Circuit breaker** — не долбить падающий сервис.
4. **Rate limiter** — ограничить скорость.

---

## 9.15 Куда идти дальше?

Мы разобрали продвинутые паттерны: семафор, rate limiter, circuit breaker, bulkhead, retry. Теперь мы умеем защищать сервис от проблем.

Но остаётся **фундаментальный вопрос**: как **обрабатывать ошибки** в конкурентном коде? Что если N горутин возвращают ошибки? Как собрать их все? Как отменить работу при первой ошибке?

- **Как обрабатывать ошибки в конкурентном коде?** `errgroup`, `multierror`, отмена при первой ошибке. → **Глава 10: Обработка ошибок в конкурентном коде.**
- **Как корректно завершить сервис?** Сигналы ОС, `context`, ожидание завершения. → **Глава 11: Graceful shutdown.**
- **Как тестировать конкурентный код?** Race detector, stress-тесты, `pprof`. → **Глава 12: Тестирование и профилирование.**

---

## 9.16 Чек-лист

| Паттерн | Что это | Ключевые факты |
|:---|:---|:---|
| **Семафор** | Ограничение доступа | `chan struct{}`, N слотов |
| **Взвешенный семафор** | Задача захватывает N единиц | `semaphore.NewWeighted(n)` |
| **Token bucket** | Ведро с токенами | Burst разрешён |
| **Leaky bucket** | Ведро с дыркой | Burst не разрешён |
| **`golang.org/x/time/rate`** | Production rate limiter | `NewLimiter`, `Wait`, `Allow` |
| **Burst** | Размер ведра | 1-10× rate |
| **Circuit breaker** | Конечный автомат | Closed, Open, Half-Open |
| **Threshold** | Ошибок до Open | 5-10 |
| **Timeout** | Время до Half-Open | 10-60 сек |
| **Bulkhead** | Изоляция ресурсов | Разные пулы |
| **Retry** | Повтор при ошибке | Max 3-5 |
| **Backoff** | Экспоненциальная задержка | 1s → 2s → 4s |
| **Jitter** | Случайность | Full jitter |
| **Retryable** | 5xx, 429, network | Не 4xx |
| **Idempotency key** | Уникальный ID | Для POST |
| **Комбинирование** | Bulkhead → Rate → Breaker → Retry → Sem | Overhead ~300-500 нс |
| **Метрики** | На каждом слое | `atomic.Int64` |

🛡️ **Ключевая идея:** Семафор — ограничение доступа к ресурсу. Взвешенный семафор — для ресурсов с разной стоимостью. Token bucket — burst разрешён; leaky bucket — нет. `golang.org/x/time/rate` — production-ready. Circuit breaker — Closed, Open, Half-Open; threshold 5-10, timeout 10-60 сек. Bulkhead — изоляция типов задач. Retry с backoff + full jitter — max 3-5, только retryable. Комбинирование: Bulkhead → Rate Limiter → Circuit Breaker → Retry → Semaphore. Метрики на каждом слое. `context.Context` во всех вызовах.