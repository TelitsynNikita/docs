# ⏱️ Глава 14: Rate limiter — token bucket

**Что вы узнаете:**
- Что такое rate limiter и какую задачу он решает.
- Чем rate limiter отличается от semaphore и worker pool.
- Как построить простейший rate limiter на `time.Ticker`.
- Как построить token bucket с нуля.
- Что такое burst и зачем он нужен.
- Как использовать `golang.org/x/time/rate`.
- Как добавить отмену через `context`.
- Как комбинировать rate limiter с worker pool, pipeline, circuit breaker.

**После прочтения вы сможете:**
- Построить rate limiter с нуля.
- Ограничивать скорость операций.
- Настраивать burst для пиков.
- Использовать `golang.org/x/time/rate` в production.
- Комбинировать rate limiter с другими паттернами.

---

## Содержание

- [14.0 Пролог: сервис, который положил внешний API](#140-пролог-сервис-который-положил-внешний-api)
- [14.1 Что такое rate limiter](#141-что-такое-rate-limiter)
- [14.2 Простейший rate limiter на time.Ticker](#142-простейший-rate-limiter-на-timeticker)
- [14.3 Token bucket с нуля](#143-token-bucket-с-нуля)
- [14.4 Burst: зачем нужен](#144-burst-зачем-нужен)
- [14.5 golang.org/x/time/rate](#145-golangorgxtimerate)
- [14.6 Rate limiter с context](#146-rate-limiter-с-context)
- [14.7 В связке с другими паттернами](#147-в-связке-с-другими-паттернами)
- [14.8 Практика Go: rate limiter с метриками](#148-практика-go-rate-limiter-с-метриками)
- [14.9 Выводы и типичные ошибки](#149-выводы-и-типичные-ошибки)
- [14.10 Для быстрого повторения](#1410-для-быстрого-повторения)
- [14.11 Вопросы для самопроверки](#1411-вопросы-для-самопроверки)
- [14.12 Ответы](#1412-ответы)
- [14.13 Куда идти дальше?](#1413-куда-идти-дальше)
- [14.14 Чек-лист](#1414-чек-лист)

---

## 14.0 Пролог: сервис, который положил внешний API

У нас есть сервис, который ходит за обогащением в другой сервис. Работает так: получает 10 000 ID, делает HTTP-запрос на каждый.

Пишем через worker pool (Глава 13):

```go
func fetchAll(ids []int, numWorkers int) {
    idsCh := make(chan int, len(ids))
    
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for id := range idsCh {
                resp, err := http.Get(fmt.Sprintf("https://api.example.com/users/%d", id))
                // обработка
            }
        }()
    }
    
    for _, id := range ids {
        idsCh <- id
    }
    close(idsCh)
    wg.Wait()
}
```

Работает. Но через 30 секунд приходит письмо от внешнего сервиса:

> «Ваш сервис превысил лимит: 10 000 запросов в секунду. Мы заблокировали ваш API-ключ на 24 часа.»

Хочется понять, что произошло. У нас **20 воркеров**. Значит, одновременно не более 20 запросов. Но если каждый занимает **1 мс**, то за секунду проходит **20 000** запросов. Внешний сервис **не выдержал**.

Проблема в том, что worker pool ограничивает **параллелизм** (число одновременных операций), но не **скорость** (число операций за период).

Хочется: **не более 100 запросов в секунду**, независимо от latency и числа воркеров.

Это и есть **rate limiter**.

> **Мост к следующим главам:** rate limiter — важный инструмент для защиты внешних сервисов. Он часто используется вместе с worker pool (Глава 13) и circuit breaker (Глава 15). Понимание rate limiter даёт понимание, **как не положить чужие сервисы**.

---

## 14.1 Что такое rate limiter

**Rate limiter** — примитив, который ограничивает **скорость** операций (сколько за период).

### Rate limiter vs semaphore и worker pool

| Аспект | Semaphore | Worker pool | Rate limiter |
|:---|:---|:---|:---|
| Что ограничивает | Параллелизм | Параллелизм | Скорость |
| Единица | Одновременные операции | Число воркеров | Операции в секунду |
| Пример | 10 одновременно | 20 воркеров | 100/сек |
| Когда | Ограничить ресурс | Много задач | Внешний API |

**Ключевое:** semaphore и worker pool ограничивают **сколько одновременно**. Rate limiter — **сколько за период**.

### Пример разницы

**Worker pool N=10:** 10 одновременных запросов. Если каждый длится 100 мс — 100 запросов/сек. Если 1 мс — 10 000 запросов/сек.

**Rate limiter 100/сек:** 100 запросов/сек, независимо от параллелизма. Если каждый длится 1 мс — 100 запросов/сек. Если 1 секунду — тоже 100 запросов/сек.

**Оба паттерна нужны.** Часто — **вместе**.

### Когда использовать rate limiter

**1. Внешний API с лимитом.**

- «Не более 100 запросов в секунду.»
- «Не более 10 000 в час.»

**2. Защита от DDoS.**

- Ограничить запросы от одного клиента.
- Ограничить запросы к внутреннему сервису.

**3. Экономия ресурсов.**

- Не перегружать БД.
- Не перегружать CPU.

**4. Плавный старт.**

- Постепенно увеличивать нагрузку.

### Когда НЕ использовать rate limiter

**1. Внутренние операции.**

Rate limiter добавляет latency. Для внутренних операций — не нужен.

**2. Одиночные операции.**

Для редких операций — overhead.

**3. Если важна пропускная способность.**

Rate limiter **ограничивает** — если нужна максимальная пропускная способность, не подходит.

### Два алгоритма

**Token bucket** — основный в Go (`golang.org/x/time/rate`).

**Leaky bucket** — для строгого сглаживания.

Разберём **token bucket**.

### 💡 Практика: как думать о rate limiter

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Rate limiter — для внешних API.**
2. **Worker pool + rate limiter — для production.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Проверяй лимиты внешнего сервиса.**
4. **Настраивай burst.**

**❌ НЕ ДЕЛАЙ:**

5. **Не путай rate limiter с worker pool.**
6. **Не используй rate limiter для внутренних операций.**

---

## 14.2 Простейший rate limiter на time.Ticker

Начнём с простейшего — `time.Ticker`.

### Идея

`time.Ticker` отправляет значение в канал **каждые N миллисекунд**. Мы используем это как «разрешение».

```go
limiter := time.Tick(100 * time.Millisecond)

for _, task := range tasks {
    <-limiter  // ждём разрешения
    process(task)
}
```

**Что происходит:**

- `time.Tick(100ms)` создаёт канал, куда каждые 100 мс отправляется время.
- `<-limiter` ждёт очередного «тика».
- Скорость: **10 операций в секунду**.

### Полный код

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    limiter := time.Tick(100 * time.Millisecond)
    
    start := time.Now()
    for i := 0; i < 10; i++ {
        <-limiter
        fmt.Printf("[%v] processing %d\n", time.Since(start).Round(time.Millisecond), i)
    }
}
```

**Пример вывода:**

```
[0s] processing 0
[100ms] processing 1
[200ms] processing 2
[300ms] processing 3
...
[900ms] processing 9
```

**Что видно:** одна операция каждые 100 мс = 10 операций в секунду.

### Проблема: time.Tick не останавливается

**❌ Плохо:**

```go
limiter := time.Tick(100 * time.Millisecond)
// ... используем, потом забываем
```

**Что происходит:** `time.Tick` создаёт таймер, который **никогда не останавливается**. Даже если `limiter` больше не используется — таймер живёт. **Утечка.**

### Решение: time.NewTicker

```go
ticker := time.NewTicker(100 * time.Millisecond)
defer ticker.Stop()

for _, task := range tasks {
    <-ticker.C
    process(task)
}
```

**Что изменилось:** `ticker.Stop()` освобождает таймер.

### Проблема: нет burst

**Что если 5 секунд не было операций, а потом нужно 10 сразу?**

`time.Tick` **не накапливает** разрешения. Если ты не читал из канала — разрешения **потеряны**. Можно получить только **одно** разрешение за тик.

**Пример:**

```go
limiter := time.Tick(100 * time.Millisecond)

// Ждём 5 секунд
time.Sleep(5 * time.Second)

// Делаем 10 запросов подряд
for i := 0; i < 10; i++ {
    <-limiter  // ← каждый ждёт 100 мс
    process(i)
}
```

**Что происходит:** 5 секунд простоя **не накапливаются**. Все 10 запросов всё равно ждут по 100 мс.

### Проблема: точность

`time.Second / time.Duration(rps)` — при больших RPS может дать **0**.

```go
rps := 1_000_000
interval := time.Second / time.Duration(rps)  // = 1000 наносекунд
```

**Но:** `time.NewTicker` имеет разрешение ~1 мс. Для высоких RPS — **неточно**.

### 💡 Практика: как использовать time.Ticker

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`time.NewTicker`**, не `time.Tick`.
2. **`defer ticker.Stop()`.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Простые случаи, без burst.**

**❌ НЕ ДЕЛАЙ:**

4. **Не используй `time.Tick` в production.**
5. **Не жди burst от `time.Ticker`.**

---

## 14.3 Token bucket с нуля

Token bucket решает проблемы `time.Ticker`: burst, точность, остановка.

### Идея

**Ведро с токенами:**

- Токены **добавляются** со скоростью `rate` (токенов/сек).
- Ведро **вмещает** максимум `burst` токенов.
- Каждый запрос **забирает** 1 токен.
- Если токенов нет — запрос **ждёт**.

### Схема

```
┌─────────────────────────────────────┐
│         Token Bucket                │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  🪙 🪙 🪙 🪙 🪙              │    │  ← ведро (burst=5)
│  └─────────────────────────────┘    │
│         ▲                  │        │
│         │                  │        │
│    добавление          забор токена │
│    (rate/сек)          (запрос)     │
│                                     │
└─────────────────────────────────────┘
```

**Что происходит:**

1. Каждые `1/rate` секунд добавляется токен.
2. Если ведро полно (`burst` токенов) — токен **пропускается**.
3. Запрос забирает токен.
4. Если токенов нет — запрос **блокируется**.

### Реализация на каналах

**Канал как ведро:**

- **Буфер канала** = `burst` (размер ведра).
- **Элементы** = токены.
- **Забор токена** = чтение из канала.
- **Добавление токена** = запись в канал.

```go
func tokenBucket(rps int, burst int) (chan struct{}, func()) {
    tokens := make(chan struct{}, burst)
    stop := make(chan struct{})
    
    // Наполняем ведро начальными токенами
    for i := 0; i < burst; i++ {
        tokens <- struct{}{}
    }
    
    // Добавляем токены со скоростью rps
    go func() {
        interval := time.Second / time.Duration(rps)
        ticker := time.NewTicker(interval)
        defer ticker.Stop()
        for {
            select {
            case <-stop:
                return
            case <-ticker.C:
                select {
                case tokens <- struct{}{}:
                default:  // ведро полно — пропускаем
                }
            }
        }
    }()
    
    return tokens, func() { close(stop) }
}
```

**Разберём по шагам.**

### Шаг 1: создание ведра

```go
tokens := make(chan struct{}, burst)
```

**Что делает:** буферизованный канал на `burst` элементов — «ведро с токенами».

### Шаг 2: начальное заполнение

```go
for i := 0; i < burst; i++ {
    tokens <- struct{}{}
}
```

**Что делает:** заполняет ведро `burst` токенами. **Burst — можно забрать сразу.**

### Шаг 3: горутина-наполнитель

```go
go func() {
    interval := time.Second / time.Duration(rps)
    ticker := time.NewTicker(interval)
    defer ticker.Stop()
    for {
        select {
        case <-stop:
            return
        case <-ticker.C:
            select {
            case tokens <- struct{}{}:
            default:
            }
        }
    }
}()
```

**Что делает:**

- Каждые `interval` секунд добавляет токен.
- Если ведро полно — токен **пропускается** (не блокируется).
- Останавливается по `stop`.

### Шаг 4: забор токена

```go
<-tokens
```

**Что делает:** забирает токен из ведра. Если пусто — блокируется.

### Шаг 5: полный пример

```go
package main

import (
    "fmt"
    "time"
)

func tokenBucket(rps int, burst int) (chan struct{}, func()) {
    tokens := make(chan struct{}, burst)
    stop := make(chan struct{})
    
    for i := 0; i < burst; i++ {
        tokens <- struct{}{}
    }
    
    go func() {
        interval := time.Second / time.Duration(rps)
        ticker := time.NewTicker(interval)
        defer ticker.Stop()
        for {
            select {
            case <-stop:
                return
            case <-ticker.C:
                select {
                case tokens <- struct{}{}:
                default:
                }
            }
        }
    }()
    
    return tokens, func() { close(stop) }
}

func main() {
    tokens, stop := tokenBucket(10, 5)  // 10/сек, burst 5
    defer stop()
    
    start := time.Now()
    for i := 0; i < 20; i++ {
        <-tokens
        fmt.Printf("[%v] request %d\n", time.Since(start).Round(time.Millisecond), i)
    }
}
```

**Пример вывода:**

```
[0s] request 0
[0s] request 1
[0s] request 2
[0s] request 3
[0s] request 4
[100ms] request 5
[200ms] request 6
[300ms] request 7
...
```

**Что видно:**

- Первые 5 запросов — **мгновенно** (burst).
- Каждый следующий — с интервалом 100 мс (10/сек).

### Схема

```
Ведро (burst=5):
  [🪙][🪙][🪙][🪙][🪙]  ← начальное заполнение

Наполнитель каждые 100 мс:
  tokens <- {}  ← если есть место

Потребитель:
  <-tokens  ← забирает токен, блокируется, если пусто
```

### Ограничения реализации

**1. Точность при высоких rps.**

`time.Second / time.Duration(rps)` при `rps = 1 000 000` даст 1000 наносекунд. Но `time.NewTicker` имеет разрешение ~1 мс. Для высоких rps — **неточно**.

**2. Нет `WaitN`.**

Нельзя забрать N токенов за раз.

**3. Нет `Allow`.**

Нельзя проверить без блокировки.

**4. Нет `Reserve`.**

Нельзя запланировать на будущее.

**Для production** используй `golang.org/x/time/rate`.

### 💡 Практика: как писать token bucket

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Буферизованный канал = ведро.**
2. **Наполнитель через `time.NewTicker`.**
3. **`select` с `default` при добавлении токена.**
4. **`stop` канал для остановки.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Оформляй в структуру.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй в production** — `x/time/rate` лучше.
7. **Не забывай `stop`.**

---

## 14.4 Burst: зачем нужен

**Burst** — максимальное число токенов в ведре. Позволяет накопить «запас» и потратить его сразу.

### Без burst

**Burst = 1:** каждую секунду ровно 1 запрос. Строгое ограничение.

```
t=0:  запрос 0  ──► 1 сек ждём
t=1s: запрос 1  ──► 1 сек ждём
t=2s: запрос 2
...
```

**Проблема:** если первый запрос нужен **сразу**, а следующий — через час, всё равно ждёшь секунду. Burst = 1 **не разрешает пики**.

### С burst = 10

**Что происходит:**

- Ведро на 10 токенов.
- Изначально 10 токенов.
- Первые 10 запросов — **мгновенно**.
- Дальше — 1 в секунду.

```
t=0:    запросы 0-9  ──► мгновенно (10 токенов)
t=1s:   запрос 10    ──► 1 токен добавлен
t=2s:   запрос 11
...
```

**Burst разрешает пики.** Если иногда нужно 10 запросов подряд — burst = 10.

### Аналогия: запас воды

**Burst = 1:** стакан воды. Наливаешь, пьёшь, ждёшь следующую каплю.

**Burst = 10:** ведро воды. Первые 10 глотков — сразу. Дальше — по капле.

### Размер burst

**Маленький (1–5):**

- Строгое ограничение.
- Нет пиков.

**Средний (10–100):**

- Для большинства случаев.
- Небольшие пики.

**Большой (1000+):**

- Почти не ограничивает.
- Большие пики.

**Правило:** burst = максимум **одновременных** запросов, которые сервис может выдержать.

### Пример: API с лимитом 100/сек

**Burst = 1:**

```go
limiter := rate.NewLimiter(100, 1)
```

- 100 запросов/сек.
- Строго по одному.

**Burst = 10:**

```go
limiter := rate.NewLimiter(100, 10)
```

- 100 запросов/сек.
- Первые 10 — мгновенно.
- Дальше по 10 мс.

**Burst = 100:**

```go
limiter := rate.NewLimiter(100, 100)
```

- 100 запросов/сек.
- Первые 100 — мгновенно.

### Что выбрать

| Сценарий | Burst |
|:---|:---|
| Строгий лимит | 1–5 |
| Обычный API | 10–100 |
| Большие пики | 100–1000 |
| Не ограничивать | 10 000+ |

### 💡 Практика: как настраивать burst

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Burst = максимум одновременных запросов.**
2. **Burst 10–100** для большинства API.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Мониторь отклонённые запросы** — если много, увеличить burst.
4. **Мониторь rate limit errors** от внешнего API.

**❌ НЕ ДЕЛАЙ:**

5. **Не ставь burst 1** для всех случаев.
6. **Не ставь burst 10 000** без причины.

---

## 14.5 golang.org/x/time/rate

Для production — используй `golang.org/x/time/rate`. Это **production-ready** rate limiter.

### Установка

```bash
go get golang.org/x/time/rate
```

### Создание

```go
import "golang.org/x/time/rate"

limiter := rate.NewLimiter(rate.Limit(100), 10)  // 100/сек, burst 10
```

**Параметры:**

- **`rate.Limit(100)`** — скорость: 100/сек.
- **`10`** — burst.

### Три метода

**`Wait(ctx)`** — блокирующее ожидание разрешения.

```go
if err := limiter.Wait(ctx); err != nil {
    return err  // ctx отменён
}
// разрешение получено
```

**`Allow()`** — неблокирующая проверка.

```go
if limiter.Allow() {
    // разрешено
} else {
    // отклонено
}
```

**`Reserve()`** — резервирование с задержкой.

```go
r := limiter.Reserve()
if !r.OK() {
    return
}
delay := r.Delay()
time.Sleep(delay)
```

### Полный пример

```go
package main

import (
    "context"
    "fmt"
    "time"
    
    "golang.org/x/time/rate"
)

func main() {
    limiter := rate.NewLimiter(rate.Limit(10), 5)  // 10/сек, burst 5
    ctx := context.Background()
    
    start := time.Now()
    for i := 0; i < 20; i++ {
        if err := limiter.Wait(ctx); err != nil {
            fmt.Println("wait failed:", err)
            return
        }
        fmt.Printf("[%v] request %d\n", time.Since(start).Round(time.Millisecond), i)
    }
}
```

**Пример вывода:**

```
[0s] request 0
[0s] request 1
[0s] request 2
[0s] request 3
[0s] request 4
[100ms] request 5
[200ms] request 6
...
```

### HTTP-клиент с rate limiter

```go
type RateLimitedClient struct {
    client  *http.Client
    limiter *rate.Limiter
}

func NewRateLimitedClient(rps int, burst int) *RateLimitedClient {
    return &RateLimitedClient{
        client:  &http.Client{Timeout: 10 * time.Second},
        limiter: rate.NewLimiter(rate.Limit(rps), burst),
    }
}

func (c *RateLimitedClient) Get(ctx context.Context, url string) (*http.Response, error) {
    if err := c.limiter.Wait(ctx); err != nil {
        return nil, err
    }
    
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    return c.client.Do(req)
}
```

**Что происходит:** каждый запрос ждёт разрешения лимитера.

### Сравнение с нашей реализацией

| Аспект | Наш token bucket | `x/time/rate` |
|:---|:---|:---|
| Burst | ✅ | ✅ |
| `context` | Частично | ✅ |
| `Allow()` | ❌ | ✅ |
| `WaitN()` | ❌ | ✅ |
| `Reserve()` | ❌ | ✅ |
| Точность при высоких rps | Ограничена | Высокая |
| Production-ready | ❌ | ✅ |

**Вывод:** наш token bucket — для **понимания**. Для production — `x/time/rate`.

### 💡 Практика: как использовать x/time/rate

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`rate.NewLimiter(rate.Limit(rps), burst)`.**
2. **`limiter.Wait(ctx)`** перед операцией.
3. **`Allow()`** для неблокирующей проверки.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Оформляй в структуру.**
5. **Мониторь отклонённые запросы.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй свой rate limiter в production.**
7. **Не забывай `context`.**

---

## 14.6 Rate limiter с context

`context` — критичен для rate limiter. Без него `Wait` может **блокироваться навсегда**.

### Проблема

```go
limiter.Wait(ctx)  // ← что если ctx отменён?
```

**С `x/time/rate`:** `Wait(ctx)` **проверяет ctx**. Если отменён — возвращает ошибку.

**С нашим token bucket:** `<-tokens` **не проверяет ctx**. Может блокироваться навсегда.

### Решение для нашего token bucket

```go
func waitToken(ctx context.Context, tokens <-chan struct{}) error {
    select {
    case <-tokens:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

**Что происходит:** `select` с `ctx.Done()`.

### Полный пример

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func tokenBucket(rps int, burst int) (chan struct{}, func()) {
    tokens := make(chan struct{}, burst)
    stop := make(chan struct{})
    
    for i := 0; i < burst; i++ {
        tokens <- struct{}{}
    }
    
    go func() {
        interval := time.Second / time.Duration(rps)
        ticker := time.NewTicker(interval)
        defer ticker.Stop()
        for {
            select {
            case <-stop:
                return
            case <-ticker.C:
                select {
                case tokens <- struct{}{}:
                default:
                }
            }
        }
    }()
    
    return tokens, func() { close(stop) }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 250*time.Millisecond)
    defer cancel()
    
    tokens, stop := tokenBucket(10, 5)
    defer stop()
    
    start := time.Now()
    for i := 0; i < 20; i++ {
        select {
        case <-tokens:
            fmt.Printf("[%v] request %d\n", time.Since(start).Round(time.Millisecond), i)
        case <-ctx.Done():
            fmt.Println("timeout:", ctx.Err())
            return
        }
    }
}
```

**Пример вывода:**

```
[0s] request 0
[0s] request 1
[0s] request 2
[0s] request 3
[0s] request 4
[100ms] request 5
[200ms] request 6
timeout: context deadline exceeded
```

**Что происходит:** через 250 мс `ctx` отменяется, цикл завершается.

### Rate limiter + worker pool + context

```go
func worker(ctx context.Context, tasksCh <-chan Task, limiter *rate.Limiter) {
    for {
        select {
        case <-ctx.Done():
            return
        case task, ok := <-tasksCh:
            if !ok {
                return
            }
            if err := limiter.Wait(ctx); err != nil {
                return  // ctx отменён
            }
            process(task)
        }
    }
}
```

**Что происходит:** worker pool + rate limiter + context.

### 💡 Практика: как добавить context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`select` с `ctx.Done()`** при ожидании токена.
2. **`Wait(ctx)`** в `x/time/rate`.
3. **Проверяй `ctx.Err()`.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`ctx` — первый аргумент.**

**❌ НЕ ДЕЛАЙ:**

5. **Не блокируйся на `<-tokens` без `ctx`.**
6. **Не забывай `ctx`.**

---

## 14.7 В связке с другими паттернами

Rate limiter редко используется **в одиночку**. Разберём связки.

### Worker pool + rate limiter

**Классическая связка:** worker pool ограничивает параллелизм, rate limiter — скорость.

```go
func workerPoolWithRateLimit(ctx context.Context, tasks []Task, numWorkers int, limiter *rate.Limiter) {
    tasksCh := make(chan Task, len(tasks))
    
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case task, ok := <-tasksCh:
                    if !ok {
                        return
                    }
                    if err := limiter.Wait(ctx); err != nil {
                        return
                    }
                    process(task)
                }
            }
        }()
    }
    
    for _, task := range tasks {
        select {
        case tasksCh <- task:
        case <-ctx.Done():
            close(tasksCh)
            wg.Wait()
            return
        }
    }
    close(tasksCh)
    wg.Wait()
}
```

**Что даёт:** одновременно не более N воркеров, и не более RPS запросов/сек.

### Pipeline + rate limiter

**Rate limiter на стадии pipeline:**

```go
func enrichStage(ctx context.Context, input <-chan Parsed, workers int, limiter *rate.Limiter) <-chan Enriched {
    out := make(chan Enriched)
    
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for v := range input {
                if err := limiter.Wait(ctx); err != nil {
                    return
                }
                enriched := enrich(v)
                select {
                case out <- enriched:
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
```

### HTTP-клиент + rate limiter + circuit breaker

**Полная защита внешнего API:**

```go
type HTTPClient struct {
    client  *http.Client
    limiter *rate.Limiter
    breaker *gobreaker.CircuitBreaker
}

func (c *HTTPClient) Get(ctx context.Context, url string) (*http.Response, error) {
    // 1. Rate limiter
    if err := c.limiter.Wait(ctx); err != nil {
        return nil, err
    }
    
    // 2. Circuit breaker
    var resp *http.Response
    _, err := c.breaker.Execute(func() (interface{}, error) {
        req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
        if err != nil {
            return nil, err
        }
        resp, err = c.client.Do(req)
        return resp, err
    })
    if err != nil {
        return nil, err
    }
    return resp, nil
}
```

**Что даёт:**

- Rate limiter — ограничивает скорость.
- Circuit breaker — защищает от сбоев.
- HTTP — сам запрос.

### Rate limiter + semaphore

**Rate limiter + semaphore = скорость + параллелизм:**

```go
func fetchAll(ctx context.Context, urls []string, maxConcurrent int, rps int) {
    sem := make(chan struct{}, maxConcurrent)
    limiter := rate.NewLimiter(rate.Limit(rps), rps*2)
    
    var wg sync.WaitGroup
    for _, url := range urls {
        url := url
        
        // Semaphore
        select {
        case sem <- struct{}{}:
        case <-ctx.Done():
            wg.Wait()
            return
        }
        
        // Rate limiter
        if err := limiter.Wait(ctx); err != nil {
            <-sem
            wg.Wait()
            return
        }
        
        wg.Add(1)
        go func() {
            defer wg.Done()
            defer func() { <-sem }()
            http.Get(url)
        }()
    }
    wg.Wait()
}
```

### Схема: полная защита

```
Запрос
   │
   ▼
┌──────────────┐
│ Rate limiter │  ← не более RPS
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Semaphore  │  ← не более N одновременно
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Circuit    │  ← защита от сбоев
│   breaker    │
└──────┬───────┘
       │
       ▼
     HTTP
```

### 💡 Практика: как комбинировать rate limiter

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Worker pool + rate limiter** — параллелизм + скорость.
2. **Pipeline + rate limiter** — стадия с ограничением.
3. **Rate limiter + circuit breaker** — защита API.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Rate limiter + semaphore** — для гибкости.

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай `context`.**

---

## 14.8 Практика Go: rate limiter с метриками

Разберём **rate limiter с метриками**.

### Полный код

```go
package main

import (
    "context"
    "fmt"
    "sync/atomic"
    "time"
    
    "golang.org/x/time/rate"
)

type Metrics struct {
    Allowed atomic.Int64
    Waited  atomic.Int64
    Denied  atomic.Int64
}

func main() {
    limiter := rate.NewLimiter(rate.Limit(100), 10)
    var metrics Metrics
    ctx := context.Background()
    
    // Симулируем 1000 запросов
    start := time.Now()
    for i := 0; i < 1000; i++ {
        waitStart := time.Now()
        if err := limiter.Wait(ctx); err != nil {
            metrics.Denied.Add(1)
            continue
        }
        waitDuration := time.Since(waitStart)
        if waitDuration > time.Millisecond {
            metrics.Waited.Add(1)
        }
        metrics.Allowed.Add(1)
    }
    elapsed := time.Since(start)
    
    fmt.Printf("Total:   %d\n", metrics.Allowed.Load()+metrics.Denied.Load())
    fmt.Printf("Allowed: %d\n", metrics.Allowed.Load())
    fmt.Printf("Denied:  %d\n", metrics.Denied.Load())
    fmt.Printf("Waited:  %d\n", metrics.Waited.Load())
    fmt.Printf("Elapsed: %v\n", elapsed)
    fmt.Printf("Rate:    %.2f req/sec\n", float64(metrics.Allowed.Load())/elapsed.Seconds())
}
```

**Пример вывода:**

```
Total:   1000
Allowed: 1000
Denied:  0
Waited:  995
Elapsed: 9.95s
Rate:    100.50 req/sec
```

**Что видно:**

- 1000 запросов за 9.95 сек.
- Первые 10 (burst) — мгновенно.
- Остальные — с интервалом 10 мс (100/сек).
- Средняя скорость — 100.50 req/sec.

### Пример: rate limiter в HTTP-клиенте

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "time"
    
    "golang.org/x/time/rate"
)

type Client struct {
    http    *http.Client
    limiter *rate.Limiter
}

func NewClient(rps int, burst int) *Client {
    return &Client{
        http:    &http.Client{Timeout: 10 * time.Second},
        limiter: rate.NewLimiter(rate.Limit(rps), burst),
    }
}

func (c *Client) Get(ctx context.Context, url string) (*http.Response, error) {
    if err := c.limiter.Wait(ctx); err != nil {
        return nil, err
    }
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    return c.http.Do(req)
}

func main() {
    client := NewClient(10, 5)
    ctx := context.Background()
    
    start := time.Now()
    for i := 0; i < 15; i++ {
        resp, err := client.Get(ctx, "https://example.com")
        if err != nil {
            fmt.Printf("[%v] error: %v\n", time.Since(start).Round(time.Millisecond), err)
            continue
        }
        resp.Body.Close()
        fmt.Printf("[%v] request %d OK\n", time.Since(start).Round(time.Millisecond), i)
    }
}
```

**Что демонстрирует:** HTTP-клиент с rate limiter. Не более 10 запросов/сек.

### 💡 Практика: как измерять rate limiter

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики:** allowed, waited, denied.
2. **Средняя скорость** — allowed / elapsed.
3. **Burst size** — число мгновенных запросов.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Экспорт в Prometheus.**
5. **Мониторинг rate limit errors** от внешнего API.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `Mutex` для метрик.**
7. **Не игнорируй растущее число denied.**

---

## 14.9 Выводы и типичные ошибки

**Что мы узнали?**

Rate limiter — примитив для ограничения **скорости** операций. **Простейший** — `time.Ticker` (но без burst). **Token bucket** — ведро с токенами: токены добавляются со скоростью `rate`, максимум `burst`. **Burst** — для пиков. **`golang.org/x/time/rate`** — production-ready. **`context`** — для отмены. Rate limiter комбинируется с worker pool, pipeline, circuit breaker, semaphore.

**Типичные ошибки:**

- ❌ **Использовать `time.Tick` в production.** Утечка.
- ❌ **Не использовать `time.NewTicker` + `Stop`.** Утечка.
- ❌ **Забыть про burst.** Пики не пройдут.
- ❌ **Burst = 1.** Нет пиков.
- ❌ **Burst = 10 000.** Почти не ограничивает.
- ❌ **Блокироваться на `<-tokens` без `ctx`.** Зависание.
- ❌ **Не использовать `x/time/rate` в production.**
- ❌ **Путать rate limiter с worker pool.**
- ❌ **Не мониторить rate limit errors.**

---

## 14.10 Для быстрого повторения

- **Rate limiter** — ограничивает скорость (операций/сек).
- **Worker pool** — параллелизм. **Semaphore** — параллелизм. **Rate limiter** — скорость.
- **`time.Ticker`** — простейший, но без burst и с утечкой.
- **Token bucket** — ведро с токенами.
- **Burst** — запас на пики.
- **Burst 10–100** для большинства API.
- **`golang.org/x/time/rate`** — production-ready.
- **`rate.NewLimiter(rps, burst)`.**
- **`Wait(ctx)`** — блокирующее ожидание.
- **`Allow()`** — неблокирующая проверка.
- **`context`** — для отмены.
- **Worker pool + rate limiter** — параллелизм + скорость.
- **Rate limiter + circuit breaker** — защита API.
- **Метрики:** allowed, waited, denied.

---

## 14.11 Вопросы для самопроверки

1. Что такое rate limiter? Какую задачу решает?
2. Чем rate limiter отличается от worker pool?
3. Что такое token bucket?
4. Что такое burst? Зачем нужен?
5. Почему `time.Tick` — плохо?
6. Как использовать `golang.org/x/time/rate`?
7. Зачем `context` в rate limiter?
8. Как комбинировать rate limiter с worker pool?

---

## 14.12 Ответы

### Ответ 1

**Rate limiter** — примитив для ограничения **скорости** (операций/сек). Решает задачу: **не более N операций за период**. Для внешних API с лимитами.

### Ответ 2

**Worker pool** ограничивает **параллелизм** — сколько операций одновременно. **Rate limiter** — **скорость** — сколько за период.

**Worker pool N=10:** 10 одновременных. Если 1 мс — 10 000/сек.

**Rate limiter 100/сек:** всегда 100/сек, независимо от параллелизма.

### Ответ 3

**Token bucket** — ведро с токенами. Токены добавляются со скоростью `rate`. Ведро максимум `burst`. Запрос забирает токен. Если пусто — ждёт.

**Реализация на каналах:** буфер канала = `burst`, токены = элементы.

### Ответ 4

**Burst** — максимальное число токенов. Позволяет **накопить запас** и потратить сразу.

**Пример:** лимитер 100/сек, burst 10. Первые 10 запросов — мгновенно. Дальше — по 10 мс.

**Размер:** 10–100 для большинства API. 1 — нет пиков. 10 000 — почти не ограничивает.

### Ответ 5

**`time.Tick` — плохо**, потому что:
1. **Не останавливается** — утечка таймера.
2. **Нет burst** — нельзя накопить запас.
3. **Неточен** при высоких rps.

**Решение:** `time.NewTicker` + `Stop`. Или `x/time/rate`.

### Ответ 6

```go
limiter := rate.NewLimiter(rate.Limit(100), 10)
if err := limiter.Wait(ctx); err != nil {
    return err
}
```

- **100** — RPS.
- **10** — burst.
- **`Wait(ctx)`** — ждать разрешения.
- **`Allow()`** — неблокирующая проверка.

### Ответ 7

**`context`** позволяет не блокироваться навсегда, если rate limiter не даёт токенов. Без него — **зависание**.

**`Wait(ctx)`** проверяет `ctx`. Если отменён — возвращает ошибку.

### Ответ 8

**Worker pool + rate limiter:**

```go
func worker(ctx context.Context, tasksCh <-chan Task, limiter *rate.Limiter) {
    for {
        select {
        case <-ctx.Done():
            return
        case task, ok := <-tasksCh:
            if !ok {
                return
            }
            if err := limiter.Wait(ctx); err != nil {
                return
            }
            process(task)
        }
    }
}
```

Worker pool ограничивает параллелизм, rate limiter — скорость.

---

## 14.13 Куда идти дальше?

Мы разобрали rate limiter — ограничение скорости. Теперь мы умеем не перегружать внешние API.

Но что если внешний сервис всё равно **падает**? Что если 50% запросов возвращают 500?

- **Как защититься от отказов?** → **Глава 15: Circuit breaker.**
- **Как повторять неудачные операции?** → **Глава 16: Retry.**
- **Как ограничить время операции?** → **Глава 17: Timeout.**

---

## 14.14 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Rate limiter** | Ограничение скорости | Операций/сек |
| **`time.Ticker`** | Простейший | Без burst, утечка |
| **Token bucket** | Ведро с токенами | Буфер канала = burst |
| **Burst** | Запас на пики | 10–100 для API |
| **`x/time/rate`** | Production-ready | `NewLimiter(rps, burst)` |
| **`Wait(ctx)`** | Блокирующее | Ждёт разрешения |
| **`Allow()`** | Неблокирующее | Проверка без ожидания |
| **`Reserve()`** | Резервирование | С задержкой |
| **`context`** | Отмена | `Wait(ctx)` |
| **Worker pool + rate limiter** | Параллелизм + скорость | Классика |
| **Rate limiter + circuit breaker** | Защита API | — |
| **Метрики** | allowed, waited, denied | `atomic.Int64` |

⏱️ **Ключевая идея:** Rate limiter — ограничение **скорости** операций. **Token bucket** — ведро с токенами: `rate` токенов/сек, максимум `burst`. **Burst** — для пиков (10–100 для API). **`golang.org/x/time/rate`** — production-ready. **`Wait(ctx)`** — блокирующее ожидание, **`Allow()`** — неблокирующее. **`context`** для отмены. Комбинируется с worker pool, pipeline, circuit breaker, semaphore. **Worker pool** ограничивает параллелизм, **rate limiter** — скорость. **Worker pool + rate limiter** — классическая связка для внешних API. `time.Tick` — утечка, не используй.