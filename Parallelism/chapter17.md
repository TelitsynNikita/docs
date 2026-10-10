# ⏳ Глава 17: Timeout — ограничение времени операции

**Что вы узнаете:**
- Что такое timeout и какую задачу он решает.
- Чем timeout отличается от retry и circuit breaker.
- Как построить простейший timeout на `time.After`.
- Как использовать `context.WithTimeout`.
- Как комбинировать timeout с retry, circuit breaker, worker pool.
- Что такое deadline propagation.
- Как избежать утечек таймеров.
- Как настроить timeout для разных операций.

**После прочтения вы сможете:**
- Построить timeout с нуля.
- Ограничивать время операций.
- Правильно использовать `context.WithTimeout`.
- Прокидывать deadline через вызовы.
- Комбинировать timeout с другими паттернами.
- Понимать, где timeout уместен, а где — нет.

---

## Содержание

- [17.0 Пролог: операция, которая зависла](#170-пролог-операция-которая-зависла)
- [17.1 Что такое timeout](#171-что-такое-timeout)
- [17.2 Простейший timeout на time.After](#172-простейший-timeout-на-timeafter)
- [17.3 Timeout на context.WithTimeout](#173-timeout-на-contextwithtimeout)
- [17.4 Deadline propagation](#174-deadline-propagation)
- [17.5 Timeout для разных операций](#175-timeout-для-разных-операций)
- [17.6 Обработка timeout-ошибок](#176-обработка-timeout-ошибок)
- [17.7 В связке с другими паттернами](#177-в-связке-с-другими-паттернами)
- [17.8 Практика Go: timeout с метриками](#178-практика-go-timeout-с-метриками)
- [17.9 Выводы и типичные ошибки](#179-выводы-и-типичные-ошибки)
- [17.10 Для быстрого повторения](#1710-для-быстрого-повторения)
- [17.11 Вопросы для самопроверки](#1711-вопросы-для-самопроверки)
- [17.12 Ответы](#1712-ответы)
- [17.13 Куда идти дальше?](#1713-куда-идти-дальше)
- [17.14 Чек-лист](#1714-чек-лист)

---

## 17.0 Пролог: операция, которая зависла

У нас есть сервис, который ходит за обогащением в другой сервис. Работает так: клиент приходит к нам, мы делаем HTTP-запрос к сервису-обогатителю, получаем данные, отдаём клиенту.

```go
func handler(w http.ResponseWriter, r *http.Request) {
    user, err := fetchFromExternal(r.Context())
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    json.NewEncoder(w).Encode(user)
}
```

Работает. Но иногда внешний сервис **зависает**. Не отвечает ни успехом, ни ошибкой. Просто **молчит**.

Наш `http.Get` ждёт. Клиент ждёт. Горутина висит. Через минуту таких запросов — десятки. Через час — сотни. Память растёт. Соединения не освобождаются.

Хочется: если внешний сервис не ответил за **5 секунд** — **отменить** операцию, вернуть клиенту ошибку.

Это и есть **timeout** — ограничение времени операции.

> **Мост к следующим главам:** timeout — фундаментальный паттерн. Он используется везде: в HTTP, БД, gRPC, worker pool. Понимание timeout даёт понимание, **как не зависать навсегда**.

---

## 17.1 Что такое timeout

**Timeout** — примитив, который **прерывает** операцию, если она длится **дольше N времени**.

### Идея

Каждая операция должна **завершиться** за разумное время. Если не завершилась — **отменяем**.

### Timeout vs retry и circuit breaker

| Аспект | Timeout | Retry | Circuit breaker |
|:---|:---|:---|:---|
| Что делает | Прерывает | Повторяет | Прекращает |
| Когда | Операция зависла | Временные ошибки | Постоянные сбои |
| Длительность | N секунд | N попыток | N ошибок |

**Вместе:**

- **Timeout** — прерывает одну попытку.
- **Retry** — повторяет N раз.
- **Circuit breaker** — прекращает после N ошибок.

### Когда использовать timeout

**1. Внешние вызовы.**

- HTTP-запросы.
- gRPC.
- БД.

**2. Операции с неизвестным временем.**

- Скачивание файла.
- Обработка большого объёма.
- ML-инференс.

**3. Пользовательские запросы.**

- HTTP-хендлеры.
- gRPC-методы.

### Когда НЕ использовать timeout

**1. Внутренние операции.**

- Быстрые функции.
- Локальные вычисления.

**2. Фоновые задачи.**

- Если операция может длиться долго — timeout не подходит.
- Нужен `context.WithCancel` без таймаута.

**3. Критичные операции.**

- Если операция **должна** завершиться — timeout может её оборвать.
- Нужна отмена только по внешнему сигналу.

### Уровни timeout

**Уровень 1: одна операция.**

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
```

**Уровень 2: цепочка операций.**

```go
ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
defer cancel()

// Внутри — отдельные таймауты
dbCtx, cancel := context.WithTimeout(ctx, 2*time.Second)
```

**Уровень 3: HTTP-сервер.**

```go
srv := &http.Server{
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  60 * time.Second,
}
```

### 💡 Практика: как думать о timeout

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Timeout — для внешних вызовов.**
2. **Timeout — для HTTP-хендлеров.**
3. **Timeout — для операций с неизвестным временем.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Deadline propagation** (см. 17.4).
5. **Метрики timeout'ов.**

**❌ НЕ ДЕЛАЙ:**

6. **Не ставь timeout без причины.**
7. **Не используй timeout для фоновых задач.**

---

## 17.2 Простейший timeout на time.After

Начнём с простейшего — `select` с `time.After`.

### Идея

```go
select {
case result := <-resultCh:
    return result
case <-time.After(5 * time.Second):
    return errors.New("timeout")
}
```

**Что делает:**

- Ждёт `resultCh` или таймаут.
- Через 5 секунд — возвращает ошибку.

### Полный пример

```go
package main

import (
    "errors"
    "fmt"
    "time"
)

func slowOperation() <-chan string {
    ch := make(chan string)
    go func() {
        time.Sleep(10 * time.Second)
        ch <- "done"
    }()
    return ch
}

func main() {
    start := time.Now()
    
    select {
    case result := <-slowOperation():
        fmt.Printf("result: %s (elapsed: %v)\n", result, time.Since(start))
    case <-time.After(1 * time.Second):
        fmt.Printf("timeout (elapsed: %v)\n", time.Since(start))
    }
}
```

**Пример вывода:**

```
timeout (elapsed: 1.001s)
```

**Что видно:** через 1 секунду — timeout, не ждём 10 секунд.

### Проблема: горутина не завершается

```go
func slowOperation() <-chan string {
    ch := make(chan string)
    go func() {
        time.Sleep(10 * time.Second)  // ← продолжает работать
        ch <- "done"                    // ← пишет в канал, который никто не читает
    }()
    return ch
}
```

**Что происходит:**

- `main` вернулся через 1 секунду.
- Горутина продолжает работать 10 секунд.
- Потом **блокируется** на `ch <- "done"` (читателя нет).
- **Утечка.**

### Решение: context

Вместо `time.After` — использовать `context`:

```go
func slowOperation(ctx context.Context) string {
    select {
    case <-ctx.Done():
        return ""
    case <-time.After(10 * time.Second):
        return "done"
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()
    
    start := time.Now()
    result := slowOperation(ctx)
    fmt.Printf("result: %q (elapsed: %v)\n", result, time.Since(start))
}
```

**Что происходит:** горутина **видит** `ctx.Done()` и завершается.

### Проблема: time.After в цикле

**❌ Плохо:**

```go
for {
    select {
    case v := <-ch:
        process(v)
    case <-time.After(5 * time.Second):  // ← создаёт НОВЫЙ таймер каждый цикл
        return
    }
}
```

**Проблема:** `time.After` создаёт **новый таймер** на каждой итерации. Старые таймеры не освобождаются до срабатывания. Утечка.

**✅ Хорошо:**

```go
timer := time.NewTimer(5 * time.Second)
defer timer.Stop()

for {
    select {
    case v := <-ch:
        process(v)
        timer.Reset(5 * time.Second)
    case <-timer.C:
        return
    }
}
```

### Схема

```
select {
case result := <-resultCh:
  return result
case <-time.After(5s):       ← создаёт таймер
  return timeout
}

Проблема: горутина, которая пишет в resultCh, не знает про таймаут
```

### 💡 Практика: как использовать time.After

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`time.After` — для простых случаев.**
2. **`time.NewTimer` — для циклов.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **`context.WithTimeout` — для отмены горутин.**

**❌ НЕ ДЕЛАЙ:**

4. **Не используй `time.After` в цикле.**
5. **Не забывай про утечки горутин.**

---

## 17.3 Timeout на context.WithTimeout

Основной способ — `context.WithTimeout`.

### Идея

```go
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
defer cancel()

result, err := doWork(ctx)
if err != nil {
    if errors.Is(err, context.DeadlineExceeded) {
        return errors.New("timeout")
    }
    return err
}
```

**Что делает:**

- Создаёт `ctx` с таймаутом 5 секунд.
- `ctx.Done()` закрывается через 5 секунд.
- `doWork` **видит** отмену.
- `err` = `context.DeadlineExceeded`.

### Полный пример

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "time"
)

func doWork(ctx context.Context) (string, error) {
    select {
    case <-ctx.Done():
        return "", ctx.Err()
    case <-time.After(10 * time.Second):
        return "done", nil
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()
    
    start := time.Now()
    result, err := doWork(ctx)
    fmt.Printf("result: %q, err: %v, elapsed: %v\n", result, err, time.Since(start))
}
```

**Пример вывода:**

```
result: "", err: context deadline exceeded, elapsed: 1.001s
```

### Почему context лучше time.After

**1. Прокидывается через вызовы.**

`ctx` передаётся в `http.NewRequestWithContext`, `db.QueryContext`, `grpc.Call`.

**2. Отменяет горутины.**

Горутина **видит** `ctx.Done()` и завершается. **Нет утечки.**

**3. Комбинируется с другими `ctx`.**

Если родительский `ctx` отменён — дочерний тоже.

**4. Даёт ошибку `context.DeadlineExceeded`.**

Легко отличить timeout от других ошибок.

### Схема

```
context.WithTimeout(parentCtx, 5s):
  │
  ├── ctx.Done() закрывается через 5 сек
  ├── ctx.Err() = DeadlineExceeded
  └── Родительский ctx отменяется → дочерний тоже

doWork(ctx):
  select {
  case <-ctx.Done():    ← отмена
    return ctx.Err()
  case <-time.After(10s):
    return "done"
  }
```

### Правильный HTTP-клиент

```go
func fetchWithTimeout(ctx context.Context, url string) (*http.Response, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    return http.DefaultClient.Do(req)
}
```

**Что происходит:** HTTP-запрос **отменяется** через 5 секунд.

### Стоимость

| Операция | Time |
|:---|:---|
| `context.WithTimeout` | ~200–500 нс |
| `ctx.Done()` проверка | ~1–5 нс |
| Таймер срабатывает | ~100–500 нс |

### 💡 Практика: как использовать context.WithTimeout

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`context.WithTimeout(ctx, N)`** — основной способ.
2. **`defer cancel()`** — обязательно.
3. **Проверяй `ctx.Err()`.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`errors.Is(err, context.DeadlineExceeded)`** — отличить timeout.

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай `cancel()`.**
6. **Не используй `context.Background()` в дочерних.**

---

## 17.4 Deadline propagation

**Deadline propagation** — передача дедлайна через все вызовы.

### Идея

Если клиент поставил дедлайн 5 секунд — **все** внутренние операции должны уложиться в 5 секунд.

### Схема

```
Клиент:                       Наш сервис:              Внешний сервис:
  "Не более 5 сек" ─────────► ctx (5 сек)
                               │
                               ├── dbCtx (2 сек) ─────► БД
                               │
                               └── apiCtx (3 сек) ────► API
                               
Если клиент отменил ─────────► ctx отменён
                               ├── dbCtx отменён
                               └── apiCtx отменён
```

### Пример

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Общий таймаут на весь handler
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()
    
    // Таймаут на БД
    dbCtx, dbCancel := context.WithTimeout(ctx, 2*time.Second)
    defer dbCancel()
    
    user, err := db.GetUser(dbCtx, 42)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    
    // Таймаут на API
    apiCtx, apiCancel := context.WithTimeout(ctx, 3*time.Second)
    defer apiCancel()
    
    profile, err := api.GetProfile(apiCtx, user.ID)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    
    json.NewEncoder(w).Encode(profile)
}
```

**Что происходит:**

- **Общий таймаут** — 5 секунд на всё.
- **БД** — 2 секунды.
- **API** — 3 секунды.
- Если **клиент** отключился — `r.Context()` отменяется → всё отменяется.
- Если **общий таймаут** истёк — всё отменяется.
- Если **БД** не успела за 2 сек — только БД отменяется.

### Правила deadline propagation

**1. Дочерний таймаут меньше родительского.**

Если родитель 5 сек, дочерний **не больше** 5 сек.

**2. Дочерний `ctx` — из родительского.**

```go
childCtx, cancel := context.WithTimeout(parentCtx, N)  // ✅
// childCtx, cancel := context.WithTimeout(context.Background(), N)  // ❌
```

**3. Передавай `ctx` во все вызовы.**

- `http.NewRequestWithContext`.
- `db.QueryContext`.
- `grpc.Call`.

### Что если дочерний больше

```go
parentCtx, _ := context.WithTimeout(context.Background(), 5*time.Second)
childCtx, _ := context.WithTimeout(parentCtx, 10*time.Second)
```

**Что происходит:** дочерний **наследует** родительский дедлайн. Через 5 секунд дочерний тоже отменится.

**Правило:** `WithTimeout` **не может увеличить** дедлайн. Только **уменьшить**.

### Как проверить дедлайн

```go
if deadline, ok := ctx.Deadline(); ok {
    remaining := time.Until(deadline)
    fmt.Printf("Remaining: %v\n", remaining)
}
```

**Что даёт:** оставшееся время до дедлайна.

### Пример: логирование

```go
func doWork(ctx context.Context) {
    if deadline, ok := ctx.Deadline(); ok {
        log.Printf("Deadline: %v (remaining: %v)", deadline, time.Until(deadline))
    }
    // ...
}
```

### Схема

```
Правильное:
  parentCtx (5 сек) ──► childCtx (2 сек)  ← меньше
  parentCtx (5 сек) ──► childCtx (3 сек)  ← меньше

Неправильное:
  parentCtx (5 сек) ──► childCtx (10 сек) ← больше, но всё равно 5
  parentCtx (5 сек) ──► childCtx (context.Background(), 2 сек) ← не связано
```

### 💡 Практика: как делать deadline propagation

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Дочерний `ctx` — из родительского.**
2. **Дочерний таймаут ≤ родительского.**
3. **Передавай `ctx` во все вызовы.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Логируй дедлайн.**
5. **Метрики по стадиям.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `context.Background()` в дочерних.**
7. **Не ставь дочерний таймаут больше родительского.**

---

## 17.5 Timeout для разных операций

Разные операции требуют разных таймаутов.

### Рекомендации

| Операция | Типичный timeout |
|:---|:---|
| **HTTP-запрос к внутреннему сервису** | 1–5 сек |
| **HTTP-запрос к внешнему API** | 5–30 сек |
| **Запрос в БД (OLTP)** | 1–5 сек |
| **Запрос в БД (аналитика)** | 30–300 сек |
| **Redis** | 100 мс – 1 сек |
| **gRPC внутри кластера** | 1–10 сек |
| **Скачивание файла** | 30 сек – 5 мин |
| **ML-инференс** | 1–60 сек |
| **HTTP-хендлер (общий)** | 5–30 сек |

### Как выбирать

**1. Измерь latency.**

Метрики покажут, сколько **обычно** занимает операция. Timeout = P99 × 2–3.

**2. Учитывай SLA.**

Если клиенту обещано 1 секунда — общий timeout ≤ 1 секунда.

**3. Учитывай retry.**

Если retry 3 раза по 1 секунде — общий timeout ≥ 3 секунды.

### Пример: HTTP-сервер с таймаутами

```go
srv := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  5 * time.Second,   // время на чтение запроса
    WriteTimeout: 10 * time.Second,  // время на запись ответа
    IdleTimeout:  60 * time.Second,  // idle-соединения
}
```

### Пример: HTTP-клиент с таймаутами

```go
client := &http.Client{
    Timeout: 10 * time.Second,  // общий таймаут
    Transport: &http.Transport{
        DialContext: (&net.Dialer{
            Timeout: 5 * time.Second,  // TCP handshake
        }).DialContext,
        TLSHandshakeTimeout:   5 * time.Second,
        ResponseHeaderTimeout: 5 * time.Second,
        IdleConnTimeout:       90 * time.Second,
    },
}
```

### Пример: БД с таймаутами

```go
db, _ := sql.Open("postgres", dsn)
db.SetConnMaxLifetime(30 * time.Minute)
db.SetMaxIdleConns(10)
db.SetMaxOpenConns(25)

// В запросе:
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
rows, _ := db.QueryContext(ctx, "SELECT ...")
```

### Ошибка: слишком маленький timeout

**Что происходит:**

- Timeout 100 мс для HTTP-запроса.
- Внешний сервис иногда отвечает за 150 мс.
- 10% запросов — timeout.
- Клиенты получают ошибки.

**Решение:** timeout = P99 × 2–3.

### Ошибка: слишком большой timeout

**Что происходит:**

- Timeout 60 сек для HTTP-запроса.
- Внешний сервис завис.
- 60 секунд ждём.
- Клиент уже ушёл.

**Решение:** timeout = 5–30 сек для HTTP.

### 💡 Практика: как настроить timeout

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Timeout = P99 × 2–3.**
2. **Учитывай SLA.**
3. **Разные timeout для разных операций.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики latency** — для выбора timeout.
5. **Мониторинг timeout errors.**

**❌ НЕ ДЕЛАЙ:**

6. **Не ставь timeout 100 мс для HTTP.**
7. **Не ставь timeout 60 сек для HTTP-хендлера.**

---

## 17.6 Обработка timeout-ошибок

Timeout даёт **конкретную ошибку** — `context.DeadlineExceeded`. Разберём, как её обрабатывать.

### Отличие timeout от отмены

**`context.DeadlineExceeded`** — истёк timeout.

**`context.Canceled`** — отмена (клиент ушёл, родитель отменил).

```go
if err := ctx.Err(); err != nil {
    if errors.Is(err, context.DeadlineExceeded) {
        // timeout
    }
    if errors.Is(err, context.Canceled) {
        // отмена
    }
}
```

### Обработка в HTTP-хендлере

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()
    
    user, err := db.GetUser(ctx, 42)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "request timeout", http.StatusGatewayTimeout)
            return
        }
        if errors.Is(err, context.Canceled) {
            // клиент ушёл — не отвечаем
            return
        }
        http.Error(w, err.Error(), 500)
        return
    }
    json.NewEncoder(w).Encode(user)
}
```

**Что происходит:**

- **Timeout** → 504 Gateway Timeout.
- **Отмена** → ничего (клиент ушёл).
- **Другая ошибка** → 500.

### Обработка с retry

```go
func fetchWithRetryAndTimeout(ctx context.Context, url string) (*http.Response, error) {
    var lastErr error
    for attempt := 0; attempt < 3; attempt++ {
        attemptCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
        
        resp, err := fetch(attemptCtx, url)
        cancel()
        
        if err == nil {
            return resp, nil
        }
        
        if errors.Is(err, context.DeadlineExceeded) {
            lastErr = err
            continue  // retry при timeout
        }
        
        if errors.Is(err, context.Canceled) {
            return nil, err  // родитель отменён — не retry
        }
        
        lastErr = err
    }
    return nil, lastErr
}
```

**Что происходит:**

- **Timeout** одной попытки → retry.
- **Отмена** родителя → не retry.

### Логирование

```go
if errors.Is(err, context.DeadlineExceeded) {
    log.Printf("operation timed out after %v", time.Since(start))
}
```

### Метрики

```go
var timeouts atomic.Int64

if errors.Is(err, context.DeadlineExceeded) {
    timeouts.Add(1)
}
```

### Схема

```
err := doWork(ctx)

if err == nil:
  → успех

if errors.Is(err, context.DeadlineExceeded):
  → timeout: retry, 504, метрика

if errors.Is(err, context.Canceled):
  → отмена: не отвечаем, не retry

else:
  → другая ошибка
```

### 💡 Практика: как обрабатывать timeout

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`errors.Is(err, context.DeadlineExceeded)`** — timeout.
2. **`errors.Is(err, context.Canceled)`** — отмена.
3. **504 Gateway Timeout** для HTTP.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Retry при timeout** одной попытки.
5. **Метрики timeout.**

**❌ НЕ ДЕЛАЙ:**

6. **Не путай timeout и отмену.**
7. **Не retry при отмене родителя.**

---

## 17.7 В связке с другими паттернами

Timeout редко используется **в одиночку**. Разберём связки.

### Timeout + retry

**Timeout** ограничивает одну попытку. **Retry** — повторяет.

```go
func fetchWithRetryAndTimeout(ctx context.Context, url string) (*http.Response, error) {
    var lastErr error
    for attempt := 0; attempt < 3; attempt++ {
        attemptCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
        
        req, _ := http.NewRequestWithContext(attemptCtx, "GET", url, nil)
        resp, err := http.DefaultClient.Do(req)
        cancel()
        
        if err == nil {
            return resp, nil
        }
        
        if errors.Is(err, context.Canceled) {
            return nil, err  // родитель отменён
        }
        
        lastErr = err
        
        select {
        case <-ctx.Done():
            return nil, lastErr
        case <-time.After(time.Duration(attempt+1) * 100 * time.Millisecond):
        }
    }
    return nil, lastErr
}
```

### Timeout + circuit breaker

**Timeout** — на одну операцию. **Circuit breaker** — на серию.

```go
func fetchWithTimeoutAndBreaker(ctx context.Context, cb *CircuitBreaker, url string) (*http.Response, error) {
    var resp *http.Response
    err := cb.Call(ctx, func(ctx context.Context) error {
        callCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
        defer cancel()
        
        req, _ := http.NewRequestWithContext(callCtx, "GET", url, nil)
        var reqErr error
        resp, reqErr = http.DefaultClient.Do(req)
        return reqErr
    })
    return resp, err
}
```

### Timeout + worker pool

**Worker pool** обрабатывает задачи. **Timeout** — внутри воркера.

```go
func worker(ctx context.Context, tasksCh <-chan Task) {
    for {
        select {
        case <-ctx.Done():
            return
        case task, ok := <-tasksCh:
            if !ok {
                return
            }
            
            taskCtx, cancel := context.WithTimeout(ctx, 10*time.Second)
            err := process(taskCtx, task)
            cancel()
            
            if err != nil {
                if errors.Is(err, context.DeadlineExceeded) {
                    log.Printf("task %d timed out", task.ID)
                }
            }
        }
    }
}
```

### Timeout + graceful shutdown

**Общий `ctx`** отменяется по сигналу. **Timeout'ы** внутри — дочерние.

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
    defer stop()
    
    srv := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
    }
    
    go srv.ListenAndServe()
    
    <-ctx.Done()
    
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
    defer cancel()
    
    srv.Shutdown(shutdownCtx)
}
```

### Полная защита

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
│   Timeout    │  ← 5 сек
└──────┬───────┘
       │
       ▼
┌──────────────┐
│    Retry     │  ← 3 попытки
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

### 💡 Практика: как комбинировать timeout

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Timeout + retry** — timeout на попытку, retry на серию.
2. **Timeout + circuit breaker** — timeout внутри breaker.
3. **Timeout + worker pool** — timeout внутри воркера.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Deadline propagation** через все вызовы.
5. **Метрики timeout.**

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `cancel()`.**

---

## 17.8 Практика Go: timeout с метриками

Разберём **timeout с метриками**.

### Полный код

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "sync/atomic"
    "time"
)

type Metrics struct {
    Calls         atomic.Int64
    Successes     atomic.Int64
    Timeouts      atomic.Int64
    Canceled      atomic.Int64
    Failures      atomic.Int64
    TotalDuration atomic.Int64
}

func doWorkWithTimeout(ctx context.Context, timeout time.Duration, fn func(ctx context.Context) error, metrics *Metrics) error {
    callCtx, cancel := context.WithTimeout(ctx, timeout)
    defer cancel()
    
    metrics.Calls.Add(1)
    start := time.Now()
    
    err := fn(callCtx)
    duration := time.Since(start)
    metrics.TotalDuration.Add(int64(duration))
    
    if err == nil {
        metrics.Successes.Add(1)
        return nil
    }
    
    if errors.Is(err, context.DeadlineExceeded) {
        metrics.Timeouts.Add(1)
        return err
    }
    
    if errors.Is(err, context.Canceled) {
        metrics.Canceled.Add(1)
        return err
    }
    
    metrics.Failures.Add(1)
    return err
}

func main() {
    ctx := context.Background()
    metrics := &Metrics{}
    
    // Быстрая операция
    err := doWorkWithTimeout(ctx, 1*time.Second, func(ctx context.Context) error {
        time.Sleep(100 * time.Millisecond)
        return nil
    }, metrics)
    fmt.Printf("Fast: %v\n", err)
    
    // Медленная операция (timeout)
    err = doWorkWithTimeout(ctx, 500*time.Millisecond, func(ctx context.Context) error {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(2 * time.Second):
            return nil
        }
    }, metrics)
    fmt.Printf("Slow: %v\n", err)
    
    // Отмена
    cancelCtx, cancel := context.WithCancel(ctx)
    cancel()
    err = doWorkWithTimeout(cancelCtx, 1*time.Second, func(ctx context.Context) error {
        return ctx.Err()
    }, metrics)
    fmt.Printf("Canceled: %v\n", err)
    
    fmt.Println("\n=== Metrics ===")
    fmt.Printf("Calls:         %d\n", metrics.Calls.Load())
    fmt.Printf("Successes:     %d\n", metrics.Successes.Load())
    fmt.Printf("Timeouts:      %d\n", metrics.Timeouts.Load())
    fmt.Printf("Canceled:      %d\n", metrics.Canceled.Load())
    fmt.Printf("Failures:      %d\n", metrics.Failures.Load())
    fmt.Printf("TotalDuration: %v\n", time.Duration(metrics.TotalDuration.Load()))
}
```

**Пример вывода:**

```
Fast: <nil>
Slow: context deadline exceeded
Canceled: context canceled

=== Metrics ===
Calls:         3
Successes:     1
Timeouts:      1
Canceled:      1
Failures:      0
TotalDuration: 600ms
```

### Что демонстрирует

1. **Три типа ошибок:** success, timeout, canceled.
2. **Метрики** по типам.
3. **Общая длительность** операций.

### 💡 Практика: как измерять timeout

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики:** calls, successes, timeouts, canceled.
2. **Различай timeout и canceled.**
3. **Экспорт в Prometheus.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Алерт при частых timeout'ах.**
5. **Percentile latency.**

**❌ НЕ ДЕЛАЙ:**

6. **Не путай timeout и ошибки бизнес-логики.**

---

## 17.9 Выводы и типичные ошибки

**Что мы узнали?**

Timeout — примитив для ограничения времени операции. **Простейший** — `time.After` в `select` (но утечка горутин). **`context.WithTimeout`** — основной способ, отменяет горутины. **Deadline propagation** — передача дедлайна через все вызовы; дочерний ≤ родительского. **Разные timeout** для разных операций. **Обработка:** `errors.Is(err, context.DeadlineExceeded)` — timeout, `context.Canceled` — отмена. Timeout комбинируется с retry, circuit breaker, worker pool, graceful shutdown.

**Типичные ошибки:**

- ❌ **`time.After` в цикле.** Утечка таймеров.
- ❌ **Не использовать `context.WithTimeout`.** Утечка горутин.
- ❌ **Забыть `cancel()`.** Утечка таймера.
- ❌ **Дочерний timeout больше родительского.** Бесполезно.
- ❌ **`context.Background()` в дочерних.** Не связан с родителем.
- ❌ **Timeout 100 мс для HTTP.** False timeouts.
- ❌ **Timeout 60 сек для HTTP-хендлера.** Клиент ушёл.
- ❌ **Не различать timeout и canceled.**
- ❌ **Retry при отмене родителя.**
- ❌ **Не мониторить timeout.**

---

## 17.10 Для быстрого повторения

- **Timeout** — ограничение времени операции.
- **`time.After` в `select`** — простейший, но утечка.
- **`time.NewTimer`** — для циклов.
- **`context.WithTimeout`** — основной способ.
- **`ctx.Done()`** — отменяет горутины.
- **`ctx.Err()`** — `DeadlineExceeded` или `Canceled`.
- **Deadline propagation** — передача через вызовы.
- **Дочерний timeout ≤ родительского.**
- **Дочерний `ctx` — из родительского.**
- **Timeout = P99 × 2–3.**
- **HTTP:** ReadTimeout, WriteTimeout, IdleTimeout.
- **БД:** `db.QueryContext` с timeout.
- **Обработка:** `errors.Is` для типа ошибки.
- **504 Gateway Timeout** для HTTP.
- **Retry при timeout** одной попытки.
- **Не retry при отмене родителя.**
- **Метрики:** calls, successes, timeouts, canceled.

---

## 17.11 Вопросы для самопроверки

1. Что такое timeout? Какую задачу решает?
2. Чем timeout отличается от retry и circuit breaker?
3. Почему `time.After` в цикле — плохо?
4. Почему `context.WithTimeout` лучше `time.After`?
5. Что такое deadline propagation?
6. Как выбирать timeout для операции?
7. Как отличить timeout от отмены?
8. Как комбинировать timeout с retry?

---

## 17.12 Ответы

### Ответ 1

**Timeout** — примитив для ограничения времени операции. Решает задачу: **не зависать навсегда**, если операция не завершается.

### Ответ 2

**Timeout** прерывает **одну** операцию. **Retry** повторяет **серию**. **Circuit breaker** прекращает после **N ошибок**.

**Вместе:** timeout на попытку, retry на серию, circuit breaker на всё.

### Ответ 3

**`time.After` в цикле** создаёт **новый таймер** на каждой итерации. Старые таймеры не освобождаются. Утечка.

**Решение:** `time.NewTimer` + `timer.Reset`.

### Ответ 4

**`context.WithTimeout` лучше**, потому что:
1. **Прокидывается** через вызовы.
2. **Отменяет горутины** — нет утечки.
3. **Комбинируется** с другими `ctx`.
4. **Даёт `DeadlineExceeded`** — легко отличить.

### Ответ 5

**Deadline propagation** — передача дедлайна через все вызовы. Если клиент поставил 5 секунд — все внутренние операции должны уложиться.

**Правило:** дочерний timeout ≤ родительского, дочерний `ctx` — из родительского.

### Ответ 6

**Timeout = P99 × 2–3.**

- Измерь latency через метрики.
- Учитывай SLA.
- Учитывай retry.

**Типичные:**
- HTTP к внутреннему: 1–5 сек.
- HTTP к внешнему: 5–30 сек.
- БД OLTP: 1–5 сек.
- Redis: 100 мс – 1 сек.

### Ответ 7

**`context.DeadlineExceeded`** — timeout.

**`context.Canceled`** — отмена (клиент ушёл, родитель отменил).

```go
if errors.Is(err, context.DeadlineExceeded) {
    // timeout
}
if errors.Is(err, context.Canceled) {
    // отмена
}
```

### Ответ 8

**Timeout + retry:**

```go
for attempt := 0; attempt < 3; attempt++ {
    attemptCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
    err := fn(attemptCtx)
    cancel()
    
    if err == nil {
        return nil
    }
    if errors.Is(err, context.Canceled) {
        return err  // не retry
    }
    // timeout → retry
}
```

Timeout на попытку, retry на серию. Не retry при отмене родителя.

---

## 17.13 Куда идти дальше?

Мы разобрали timeout — ограничение времени операции. Теперь мы умеем не зависать навсегда.

Но иногда операция вызывается **слишком часто**. Например, пользователь вводит текст — не нужно делать запрос на **каждую** букву.

- **Как ограничить частоту операций?** → **Глава 18: Debounce и Throttle.**
- **Как построить pipeline?** → **Глава 11: Pipeline.**
- **Как ограничить скорость?** → **Глава 14: Rate limiter.**

---

## 17.14 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Timeout** | Ограничение времени | Прерывает операцию |
| **`time.After`** | Простейший | Утечка горутин |
| **`time.NewTimer`** | Для циклов | `Reset` |
| **`context.WithTimeout`** | Основной | Отменяет горутины |
| **`ctx.Done()`** | Отмена | В `select` |
| **`ctx.Err()`** | Причина | `DeadlineExceeded`, `Canceled` |
| **Deadline propagation** | Передача дедлайна | Дочерний ≤ родительского |
| **HTTP** | ReadTimeout, WriteTimeout, IdleTimeout | — |
| **БД** | `QueryContext` | — |
| **Timeout = P99 × 2–3** | Выбор | — |
| **504** | HTTP-ответ | Gateway Timeout |
| **Retry при timeout** | Одной попытки | — |
| **Не retry при canceled** | Родитель отменён | — |
| **Метрики** | calls, timeouts, canceled | — |

⏳ **Ключевая идея:** Timeout — примитив для ограничения времени операции. **`context.WithTimeout`** — основной способ, отменяет горутины. **`time.After` в `select`** — простейший, но утечка горутин и таймеров. **Deadline propagation** — передача дедлайна через все вызовы; дочерний timeout ≤ родительского, дочерний `ctx` — из родительского. **Timeout = P99 × 2–3**. Разные timeout для разных операций. Обработка: `DeadlineExceeded` (timeout) vs `Canceled` (отмена). **504 Gateway Timeout** для HTTP. Retry при timeout одной попытки, не retry при отмене. Комбинируется с retry, circuit breaker, worker pool, graceful shutdown. Метрики: calls, successes, timeouts, canceled.