# 🔁 Глава 16: Retry — повторные попытки

**Что вы узнаете:**
- Что такое retry и какую задачу он решает.
- Чем retry отличается от circuit breaker.
- Как построить простейший retry.
- Как добавить задержку между попытками.
- Что такое экспоненциальный backoff и jitter.
- Как ограничить общее время retry через `context`.
- Как обрабатывать **permanent** ошибки (не повторять).
- Как комбинировать retry с circuit breaker, rate limiter, worker pool.

**После прочтения вы сможете:**
- Построить retry с нуля.
- Настраивать backoff и jitter.
- Понимать, когда retry помогает, а когда — нет.
- Ограничивать retry через `context`.
- Отличать временные ошибки от постоянных.
- Комбинировать retry с другими паттернами.

---

## Содержание

- [16.0 Пролог: временные ошибки](#160-пролог-временные-ошибки)
- [16.1 Что такое retry](#161-что-такое-retry)
- [16.2 Простейший retry](#162-простейший-retry)
- [16.3 Retry с задержкой и backoff](#163-retry-с-задержкой-и-backoff)
- [16.4 Jitter: почему нужен](#164-jitter-почему-нужен)
- [16.5 Retry с context](#165-retry-с-context)
- [16.6 Permanent ошибки: когда не повторять](#166-permanent-ошибки-когда-не-повторять)
- [16.7 В связке с другими паттернами](#167-в-связке-с-другими-паттернами)
- [16.8 Практика Go: retry с метриками](#168-практика-go-retry-с-метриками)
- [16.9 Выводы и типичные ошибки](#169-выводы-и-типичные-ошибки)
- [16.10 Для быстрого повторения](#1610-для-быстрого-повторения)
- [16.11 Вопросы для самопроверки](#1611-вопросы-для-самопроверки)
- [16.12 Ответы](#1612-ответы)
- [16.13 Куда идти дальше?](#1613-куда-идти-дальше)
- [16.14 Чек-лист](#1614-чек-лист)

---

## 16.0 Пролог: временные ошибки

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

Работает. Но иногда внешний сервис возвращает **ошибки**. Не 100% — а, скажем, 2% запросов. По логам видно, что это **временные** ошибки: сервис перегружен, БД на секунду лагнула, сеть дрогнула. Через секунду тот же запрос проходит **успешно**.

Клиенты жалуются: 2% запросов падают. Хочется **не отдавать** клиенту ошибку, если она **временная**.

Простейшая мысль: **повторить** запрос при ошибке.

```go
func fetchWithRetry(ctx context.Context, url string) (*User, error) {
    for i := 0; i < 3; i++ {
        user, err := fetch(url)
        if err == nil {
            return user, nil
        }
        time.Sleep(100 * time.Millisecond)  // ← фиксированная задержка
    }
    return nil, errors.New("max retries")
}
```

Работает. Но если внешний сервис **сильно** перегружен — 3 попытки **подряд** только **ухудшают** ситуацию. Мы **добавляем** ему нагрузку.

Хочется: **повторять**, но с **умной** задержкой, и **не повторять** постоянные ошибки (401, 404). И **не бесконечно** — общий таймаут.

Это и есть **retry** — паттерн повторных попыток.

> **Мост к следующим главам:** retry часто используется вместе с circuit breaker (Глава 15) и rate limiter (Глава 14). Понимание retry даёт понимание, **как не ухудшить ситуацию при повторных попытках**.

---

## 16.1 Что такое retry

**Retry** — паттерн, при котором неудачные операции **повторяются** несколько раз.

### Идея

**Временные ошибки** — нормальны для распределённых систем. Сеть дрогнула, сервис перегружен, БД лагнула. Через секунду та же операция пройдёт.

**Retry** повторяет операцию, давая системе **время** восстановиться.

### Retry vs circuit breaker

| Аспект | Retry | Circuit breaker |
|:---|:---|:---|
| Что делает | Повторяет | Прекращает |
| Когда | Временные ошибки | Постоянные сбои |
| Риск | Ухудшить ситуацию | Никогда не восстановиться |
| Как часто | 1–5 попыток | Открывается при N ошибках |

**Вместе:** retry повторяет, circuit breaker прекращает, если retry не помогает.

### Когда использовать retry

**1. Временные ошибки.**

- HTTP 5xx (кроме 501 Not Implemented).
- Сетевые ошибки (timeout, connection refused).
- БД: deadlock, serialization failure.
- Rate limit 429 (Too Many Requests).

**2. Идемпотентные операции.**

- GET запросы.
- PUT запросы.
- DELETE запросы.

**3. Внешние сервисы.**

- Микросервисы.
- БД.
- Кэши.

### Когда НЕ использовать retry

**1. Постоянные ошибки.**

- 400 Bad Request.
- 401 Unauthorized.
- 403 Forbidden.
- 404 Not Found.
- 501 Not Implemented.

**Retry не поможет.** Эти ошибки повторятся.

**2. Не-идемпотентные операции.**

- POST без idempotency key.
- Платежи без idempotency key.

**Риск:** задвоение платежа.

**3. Высокая нагрузка.**

Если сервис **перегружен**, retry **ухудшает** ситуацию. Нужен circuit breaker.

### Правила retry

**1. Ограниченное число попыток.**

- 3–5 попыток — разумно.
- 10+ — опасно.

**2. Задержка между попытками.**

- Без задержки — DDOS на сервис.
- С задержкой — время восстановиться.

**3. Экспоненциальный backoff.**

- 100 мс, 200 мс, 400 мс, 800 мс...
- Экспоненциальный рост.

**4. Jitter.**

- Случайный разброс.
- Предотвращает «толпу».

**5. Общий таймаут.**

- Не более N секунд всего.
- Через `context`.

### 💡 Практика: как думать о retry

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Retry — для временных ошибок.**
2. **3–5 попыток максимум.**
3. **Задержка + jitter.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Экспоненциальный backoff.**
5. **`context` с общим таймаутом.**

**❌ НЕ ДЕЛАЙ:**

6. **Не retry permanent ошибки.**
7. **Не retry без задержки.**
8. **Не retry не-идемпотентные операции без ключа.**

---

## 16.2 Простейший retry

Начнём с самого простого — N попыток без задержки.

### Реализация

```go
func retry(n int, fn func() error) error {
    var lastErr error
    for i := 0; i < n; i++ {
        err := fn()
        if err == nil {
            return nil
        }
        lastErr = err
    }
    return lastErr
}
```

**Что делает:** вызывает `fn` до N раз. Возвращает последнюю ошибку.

### Потребитель

```go
func main() {
    attempts := 0
    
    err := retry(3, func() error {
        attempts++
        if attempts < 3 {
            return errors.New("temporary error")
        }
        return nil
    })
    
    fmt.Println("attempts:", attempts)
    fmt.Println("error:", err)
}
```

**Пример вывода:**

```
attempts: 3
error: <nil>
```

**Что видно:** третья попытка успешна.

### Проблема: нет задержки

**Что если ошибка из-за перегрузки?** Три попытки **мгновенно** только **ухудшают** ситуацию. Внешний сервис получает **3 запроса** за миллисекунду.

### Проблема: нет ограничения по времени

**Что если каждая попытка занимает 5 секунд?** 3 попытки = 15 секунд. Клиент всё это время ждёт.

### Проблема: нет различения ошибок

**Что если ошибка — 404?** Retry не поможет. Все 3 попытки вернут 404.

### Проблема: нет метрик

**Что если мы не видим, сколько retry делается?** Не знаем, помогает ли retry.

### Схема

```
retry(3, fn):

  attempt 1 ──► err ──► attempt 2 ──► err ──► attempt 3 ──► err
                                                          │
                                                          ▼
                                                       return err
```

### 💡 Практика: как писать простой retry

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **N попыток — 3–5.**
2. **Возвращай последнюю ошибку.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Добавь задержку** (см. 16.3).

**❌ НЕ ДЕЛАЙ:**

4. **Не retry без задержки в production.**
5. **Не retry бесконечно.**

---

## 16.3 Retry с задержкой и backoff

Добавим **задержку** между попытками.

### Фиксированная задержка

```go
func retryWithDelay(n int, delay time.Duration, fn func() error) error {
    var lastErr error
    for i := 0; i < n; i++ {
        err := fn()
        if err == nil {
            return nil
        }
        lastErr = err
        if i < n-1 {
            time.Sleep(delay)
        }
    }
    return lastErr
}
```

**Что делает:** между попытками спит `delay`.

**Проблема:** фиксированная задержка **не адаптируется**. Если сервис восстанавливается за 1 секунду — 100 мс недостаточно. Если за 100 мс — 1 секунда избыточна.

### Экспоненциальный backoff

**Идея:** задержка **удваивается** с каждой попыткой.

```
attempt 1: 100 мс
attempt 2: 200 мс
attempt 3: 400 мс
attempt 4: 800 мс
attempt 5: 1600 мс
```

**Зачем:**

- Первая попытка — быстро.
- Если ошибка временная — следующая попытка через 100 мс.
- Если ошибка долгая — следующая попытка через 1.6 сек.

### Реализация

```go
func retryWithBackoff(n int, baseDelay time.Duration, fn func() error) error {
    var lastErr error
    delay := baseDelay
    
    for i := 0; i < n; i++ {
        err := fn()
        if err == nil {
            return nil
        }
        lastErr = err
        
        if i < n-1 {
            time.Sleep(delay)
            delay *= 2  // удваиваем
        }
    }
    return lastErr
}
```

### Пример

```go
func main() {
    attempts := 0
    start := time.Now()
    
    err := retryWithBackoff(4, 100*time.Millisecond, func() error {
        attempts++
        fmt.Printf("[%v] attempt %d\n", time.Since(start).Round(time.Millisecond), attempts)
        return errors.New("temporary error")
    })
    
    fmt.Println("error:", err)
}
```

**Пример вывода:**

```
[0s] attempt 1
[100ms] attempt 2
[300ms] attempt 3
[700ms] attempt 4
error: temporary error
```

**Что видно:**

- Attempt 1 — сразу.
- Attempt 2 — через 100 мс.
- Attempt 3 — через 200 мс (100+100).
- Attempt 4 — через 400 мс (100+200+400).

### Ограничение максимальной задержки

**Проблема:** экспоненциальный backoff может стать **очень большим**.

```
attempt 10: 51.2 сек
attempt 20: 14 часов
```

**Решение:** **cap** — максимальная задержка.

```go
func retryWithBackoff(n int, baseDelay, maxDelay time.Duration, fn func() error) error {
    var lastErr error
    delay := baseDelay
    
    for i := 0; i < n; i++ {
        err := fn()
        if err == nil {
            return nil
        }
        lastErr = err
        
        if i < n-1 {
            time.Sleep(delay)
            delay *= 2
            if delay > maxDelay {
                delay = maxDelay
            }
        }
    }
    return lastErr
}
```

**Пример с maxDelay = 1 сек:**

```
attempt 1: 0s
attempt 2: 100ms
attempt 3: 300ms
attempt 4: 700ms
attempt 5: 1500ms (не 1500, а 1000 + 700 = 1700... cap на 1000)
```

### Схема

```
Экспоненциальный backoff:

  attempt 1: 0 ms
  attempt 2: 100 ms
  attempt 3: 300 ms (100 + 200)
  attempt 4: 700 ms (100 + 200 + 400)
  attempt 5: 1500 ms (100 + 200 + 400 + 800)
  
  С maxDelay = 500 ms:
  attempt 1: 0 ms
  attempt 2: 100 ms
  attempt 3: 300 ms
  attempt 4: 800 ms (300 + 500 вместо 700)
  attempt 5: 1300 ms (800 + 500)
```

### 💡 Практика: как настроить backoff

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Экспоненциальный backoff** — стандарт.
2. **Base delay 50–200 мс.**
3. **Max delay 1–5 сек.**
4. **N попыток 3–5.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Jitter** (см. 16.4).

**❌ НЕ ДЕЛАЙ:**

6. **Не используй фиксированную задержку.**
7. **Не давай backoff расти бесконечно.**

---

## 16.4 Jitter: почему нужен

**Jitter** — случайный разброс задержки. Предотвращает **thundering herd**.

### Проблема: thundering herd

Представь: 1000 клиентов одновременно получили ошибку. Все начали retry **одновременно**:

```
t=0:    1000 клиентов получили ошибку
t=100ms: все 1000 делают попытку 2
t=300ms: все 1000 делают попытку 3
...
```

**Что происходит:** на сервис идёт **1000 одновременных** запросов. Он **снова** перегружается.

### Решение: jitter

Добавляем **случайный разброс** к задержке:

```
t=0:    1000 клиентов получили ошибку
t=100ms: клиент 1 делает попытку 2
t=110ms: клиент 2 делает попытку 2
t=120ms: клиент 3 делает попытку 2
...
```

**Что происходит:** запросы **распределяются** во времени. Сервис получает **плавную** нагрузку.

### Реализация

**Full jitter:**

```go
delay := time.Duration(rand.Int63n(int64(maxDelay)))
```

**Что делает:** случайная задержка от 0 до maxDelay.

**Equal jitter:**

```go
half := delay / 2
delay = half + time.Duration(rand.Int63n(int64(half)))
```

**Что делает:** задержка от delay/2 до delay.

**Decorrelated jitter:**

```go
delay = min(maxDelay, baseDelay + time.Duration(rand.Int63n(int64(delay*3-baseDelay))))
```

**Что делает:** задержка зависит от предыдущей.

### Простая реализация

```go
func retryWithBackoffJitter(n int, baseDelay, maxDelay time.Duration, fn func() error) error {
    var lastErr error
    delay := baseDelay
    
    for i := 0; i < n; i++ {
        err := fn()
        if err == nil {
            return nil
        }
        lastErr = err
        
        if i < n-1 {
            // Jitter: delay ± 50%
            jitter := time.Duration(rand.Int63n(int64(delay))) - delay/2
            time.Sleep(delay + jitter)
            
            delay *= 2
            if delay > maxDelay {
                delay = maxDelay
            }
        }
    }
    return lastErr
}
```

**Что делает:** задержка `delay ± 50%`.

### Пример

```go
func main() {
    attempts := 0
    start := time.Now()
    
    err := retryWithBackoffJitter(4, 100*time.Millisecond, 1*time.Second, func() error {
        attempts++
        fmt.Printf("[%v] attempt %d\n", time.Since(start).Round(time.Millisecond), attempts)
        return errors.New("temporary error")
    })
    
    fmt.Println("error:", err)
}
```

**Пример вывода:**

```
[0s] attempt 1
[85ms] attempt 2
[251ms] attempt 3
[623ms] attempt 4
error: temporary error
```

**Что видно:** задержки разные (85, 166, 372). Это jitter.

### Схема

```
Без jitter:
  1000 клиентов:
    t=100ms: ████████████████████
    t=300ms: ████████████████████
    t=700ms: ████████████████████

С jitter:
  1000 клиентов:
    t=100-200ms: ████████████████████
    t=200-500ms: ████████████████████
    t=500-1200ms: ████████████████████
```

### 💡 Практика: как использовать jitter

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Jitter для всех retry.**
2. **Full jitter** (`rand.Int63n(delay)`) — просто.
3. **Equal jitter** (`delay/2 + rand.Int63n(delay/2)`) — стандарт.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`math/rand/v2`** в Go 1.22+.

**❌ НЕ ДЕЛАЙ:**

5. **Не retry без jitter при многих клиентах.**

---

## 16.5 Retry с context

`context` — критичен для retry. Без него retry может **блокироваться навсегда**.

### Проблема

```go
for i := 0; i < n; i++ {
    err := fn()
    if err == nil {
        return nil
    }
    time.Sleep(delay)  // ← если ctx отменён — всё равно спим
}
```

**Что происходит:** если `ctx` отменён во время `time.Sleep`, retry **продолжит** работу.

### Решение: select с ctx.Done()

```go
func retryCtx(ctx context.Context, n int, baseDelay time.Duration, fn func(ctx context.Context) error) error {
    var lastErr error
    delay := baseDelay
    
    for i := 0; i < n; i++ {
        select {
        case <-ctx.Done():
            if lastErr != nil {
                return lastErr  // возвращаем последнюю ошибку
            }
            return ctx.Err()
        default:
        }
        
        err := fn(ctx)
        if err == nil {
            return nil
        }
        lastErr = err
        
        if i < n-1 {
            select {
            case <-ctx.Done():
                return lastErr
            case <-time.After(delay):
            }
            delay *= 2
        }
    }
    return lastErr
}
```

**Что происходит:**

- Проверяем `ctx.Done()` в начале каждой попытки.
- Ждём через `select` с `ctx.Done()`.
- Если `ctx` отменён — возвращаем последнюю ошибку.

### Полный пример

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "time"
)

func retryCtx(ctx context.Context, n int, baseDelay time.Duration, fn func(ctx context.Context) error) error {
    var lastErr error
    delay := baseDelay
    
    for i := 0; i < n; i++ {
        select {
        case <-ctx.Done():
            if lastErr != nil {
                return lastErr
            }
            return ctx.Err()
        default:
        }
        
        err := fn(ctx)
        if err == nil {
            return nil
        }
        lastErr = err
        
        if i < n-1 {
            select {
            case <-ctx.Done():
                return lastErr
            case <-time.After(delay):
            }
            delay *= 2
        }
    }
    return lastErr
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
    defer cancel()
    
    attempts := 0
    start := time.Now()
    
    err := retryCtx(ctx, 10, 100*time.Millisecond, func(ctx context.Context) error {
        attempts++
        fmt.Printf("[%v] attempt %d\n", time.Since(start).Round(time.Millisecond), attempts)
        return errors.New("temporary error")
    })
    
    fmt.Println("error:", err)
}
```

**Пример вывода:**

```
[0s] attempt 1
[100ms] attempt 2
[300ms] attempt 3
error: temporary error
```

**Что видно:** на attempt 4 `ctx` отменён (500 мс истекли). Retry останавливается, возвращает последнюю ошибку.

### Схема

```
Retry с ctx:

  for i := 0; i < n; i++ {
    select {
    case <-ctx.Done():  ← отмена
      return lastErr
    default:
    }
    
    err := fn(ctx)
    if err == nil {
      return nil
    }
    lastErr = err
    
    select {
    case <-ctx.Done():  ← отмена
      return lastErr
    case <-time.After(delay):
    }
  }
```

### 💡 Практика: как добавить context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx context.Context` — первый аргумент.**
2. **`fn(ctx)` — передавай ctx в callback.**
3. **`select` с `ctx.Done()` при ожидании.**
4. **Проверяй `ctx.Done()` в начале попытки.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **`context.WithTimeout`** — общий таймаут retry.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `time.Sleep` — используй `select`.**
7. **Не блокируйся навсегда.**

---

## 16.6 Permanent ошибки: когда не повторять

Не все ошибки **временные**. Некоторые — **постоянные**. Retry **не поможет**.

### Какие ошибки permanent

**HTTP:**

- 400 Bad Request.
- 401 Unauthorized.
- 403 Forbidden.
- 404 Not Found.
- 405 Method Not Allowed.
- 501 Not Implemented.

**gRPC:**

- `InvalidArgument`.
- `Unauthenticated`.
- `PermissionDenied`.
- `NotFound`.
- `Unimplemented`.

**БД:**

- Ошибки валидации.
- Ошибки схемы.
- Нарушение уникальности.

### Как отличить permanent от временной

**Способ 1: тип ошибки.**

```go
type PermanentError struct {
    Code int
    Msg  string
}

func (e *PermanentError) Error() string {
    return fmt.Sprintf("permanent error %d: %s", e.Code, e.Msg)
}
```

**В коде:**

```go
if resp.StatusCode >= 400 && resp.StatusCode < 500 && resp.StatusCode != 429 {
    return &PermanentError{Code: resp.StatusCode}
}
```

**При retry:**

```go
err := fn()
if err == nil {
    return nil
}

var permErr *PermanentError
if errors.As(err, &permErr) {
    return err  // не retry
}
```

### Способ 2: HTTP-код

```go
func shouldRetry(statusCode int) bool {
    switch statusCode {
    case 429:              // Too Many Requests — retry
        return true
    }
    if statusCode >= 500 && statusCode != 501 {
        return true  // 5xx — retry
    }
    return false
}
```

**Что retry:**

- 429 — Too Many Requests.
- 500 — Internal Server Error.
- 502 — Bad Gateway.
- 503 — Service Unavailable.
- 504 — Gateway Timeout.

**Что НЕ retry:**

- 400, 401, 403, 404, 405, 501.

### Полная реализация

```go
func retryWithPermanentCheck(ctx context.Context, n int, baseDelay time.Duration, fn func(ctx context.Context) error) error {
    var lastErr error
    delay := baseDelay
    
    for i := 0; i < n; i++ {
        select {
        case <-ctx.Done():
            if lastErr != nil {
                return lastErr
            }
            return ctx.Err()
        default:
        }
        
        err := fn(ctx)
        if err == nil {
            return nil
        }
        
        var permErr *PermanentError
        if errors.As(err, &permErr) {
            return err  // не retry
        }
        
        lastErr = err
        
        if i < n-1 {
            select {
            case <-ctx.Done():
                return lastErr
            case <-time.After(delay):
            }
            delay *= 2
        }
    }
    return lastErr
}
```

### Схема

```
Retry с проверкой permanent:

  err := fn(ctx)
  if err == nil {
    return nil
  }
  
  if isPermanent(err) {  ← проверка
    return err           ← не retry
  }
  
  // retry
```

### 💡 Практика: как обрабатывать permanent ошибки

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`PermanentError` — тип для постоянных.**
2. **`errors.As` — проверка.**
3. **`shouldRetry(statusCode)` — по HTTP-коду.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Логирование permanent ошибок.**
5. **Метрики для permanent.**

**❌ НЕ ДЕЛАЙ:**

6. **Не retry 4xx (кроме 429).**
7. **Не retry ошибки валидации.**

---

## 16.7 В связке с другими паттернами

Retry редко используется **в одиночку**. Разберём связки.

### Retry + circuit breaker

**Retry** повторяет. **Circuit breaker** прекращает, если retry не помогает.

```go
func fetchWithRetryAndBreaker(ctx context.Context, cb *CircuitBreaker, url string) (*http.Response, error) {
    var lastErr error
    for attempt := 0; attempt < 3; attempt++ {
        var resp *http.Response
        err := cb.Call(ctx, func(ctx context.Context) error {
            req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
            if err != nil {
                return err
            }
            var reqErr error
            resp, reqErr = http.DefaultClient.Do(req)
            return reqErr
        })
        if err == nil {
            return resp, nil
        }
        if errors.Is(err, ErrCircuitOpen) {
            return nil, err  // не retry
        }
        lastErr = err
        
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-time.After(time.Duration(attempt+1) * 100 * time.Millisecond):
        }
    }
    return nil, lastErr
}
```

**Порядок:**

1. **Circuit breaker** — оборачивает HTTP.
2. **Retry** — вокруг breaker.
3. **Не retry при `ErrCircuitOpen`.**

### Retry + rate limiter

**Rate limiter** ограничивает скорость. **Retry** — повторяет.

```go
func fetchWithRetryAndLimiter(ctx context.Context, limiter *rate.Limiter, url string) (*http.Response, error) {
    var lastErr error
    for attempt := 0; attempt < 3; attempt++ {
        if err := limiter.Wait(ctx); err != nil {
            return nil, err
        }
        
        resp, err := http.Get(url)
        if err == nil {
            return resp, nil
        }
        lastErr = err
        
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-time.After(time.Duration(attempt+1) * 100 * time.Millisecond):
        }
    }
    return nil, lastErr
}
```

### Retry + worker pool

**Worker pool** обрабатывает задачи. **Retry** — внутри воркера.

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
            err := retryCtx(ctx, 3, 100*time.Millisecond, func(ctx context.Context) error {
                return process(ctx, task)
            })
            if err != nil {
                log.Printf("task %d failed after retries: %v", task.ID, err)
            }
        }
    }
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

**Порядок:**

1. **Rate limiter** — первым.
2. **Retry** — вокруг breaker.
3. **Circuit breaker** — оборачивает HTTP.

### 💡 Практика: как комбинировать retry

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Retry + circuit breaker** — полная защита.
2. **Rate limiter → retry → breaker → HTTP.**
3. **Не retry при open breaker.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Retry + worker pool** — для задач.
5. **Retry + rate limiter** — для внешних API.

**❌ НЕ ДЕЛАЙ:**

6. **Не retry бесконечно.**
7. **Не забывай про jitter.**

---

## 16.8 Практика Go: retry с метриками

Разберём **retry с метриками**.

### Полный код

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "math/rand"
    "sync/atomic"
    "time"
)

type Metrics struct {
    Attempts     atomic.Int64
    Successes    atomic.Int64
    Failures     atomic.Int64
    TotalWait    atomic.Int64
    PermanentErr atomic.Int64
}

type PermanentError struct {
    Code int
}

func (e *PermanentError) Error() string {
    return fmt.Sprintf("permanent error %d", e.Code)
}

func retryCtx(ctx context.Context, n int, baseDelay, maxDelay time.Duration, fn func(ctx context.Context) error, metrics *Metrics) error {
    var lastErr error
    delay := baseDelay
    
    for i := 0; i < n; i++ {
        select {
        case <-ctx.Done():
            if lastErr != nil {
                return lastErr
            }
            return ctx.Err()
        default:
        }
        
        metrics.Attempts.Add(1)
        err := fn(ctx)
        if err == nil {
            metrics.Successes.Add(1)
            return nil
        }
        
        var permErr *PermanentError
        if errors.As(err, &permErr) {
            metrics.PermanentErr.Add(1)
            return err
        }
        
        lastErr = err
        metrics.Failures.Add(1)
        
        if i < n-1 {
            // Jitter: delay ± 50%
            jitter := time.Duration(rand.Int63n(int64(delay))) - delay/2
            wait := delay + jitter
            metrics.TotalWait.Add(int64(wait))
            
            select {
            case <-ctx.Done():
                return lastErr
            case <-time.After(wait):
            }
            
            delay *= 2
            if delay > maxDelay {
                delay = maxDelay
            }
        }
    }
    return lastErr
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    metrics := &Metrics{}
    attempts := 0
    
    start := time.Now()
    err := retryCtx(ctx, 5, 100*time.Millisecond, 1*time.Second, func(ctx context.Context) error {
        attempts++
        if attempts < 3 {
            return errors.New("temporary error")
        }
        return nil
    }, metrics)
    
    fmt.Printf("Error: %v\n", err)
    fmt.Printf("Elapsed: %v\n\n", time.Since(start))
    
    fmt.Println("=== Metrics ===")
    fmt.Printf("Attempts:     %d\n", metrics.Attempts.Load())
    fmt.Printf("Successes:    %d\n", metrics.Successes.Load())
    fmt.Printf("Failures:     %d\n", metrics.Failures.Load())
    fmt.Printf("PermanentErr: %d\n", metrics.PermanentErr.Load())
    fmt.Printf("TotalWait:    %v\n", time.Duration(metrics.TotalWait.Load()))
}
```

**Пример вывода:**

```
Error: <nil>
Elapsed: 235ms

=== Metrics ===
Attempts:     3
Successes:    1
Failures:     2
PermanentErr: 0
TotalWait:    235ms
```

### Пример: retry с permanent ошибкой

```go
func main() {
    ctx := context.Background()
    metrics := &Metrics{}
    
    err := retryCtx(ctx, 5, 100*time.Millisecond, 1*time.Second, func(ctx context.Context) error {
        return &PermanentError{Code: 404}
    }, metrics)
    
    fmt.Printf("Error: %v\n", err)
    fmt.Printf("Attempts: %d\n", metrics.Attempts.Load())
    fmt.Printf("PermanentErr: %d\n", metrics.PermanentErr.Load())
}
```

**Пример вывода:**

```
Error: permanent error 404
Attempts: 1
PermanentErr: 1
```

**Что видно:** попытка **одна** — permanent ошибка **не повторяется**.

### 💡 Практика: как измерять retry

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики:** attempts, successes, failures, permanentErr.
2. **TotalWait** — сколько времени ушло на задержки.
3. **Экспорт в Prometheus.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Логирование каждой попытки.**
5. **Мониторинг permanent ошибок.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `Mutex` для метрик.**
7. **Не игнорируй retry в метриках.**

---

## 16.9 Выводы и типичные ошибки

**Что мы узнали?**

Retry — паттерн повторных попыток для временных ошибок. **Простейший** — N попыток без задержки (плохо). **С задержкой** — фиксированная (плохо) или экспоненциальный backoff (хорошо). **Jitter** — случайный разброс, предотвращает thundering herd. **`context`** — для отмены. **Permanent ошибки** (4xx) — не повторять. Retry комбинируется с circuit breaker, rate limiter, worker pool.

**Типичные ошибки:**

- ❌ **Retry без задержки.** DDOS на сервис.
- ❌ **Фиксированная задержка.** Не адаптируется.
- ❌ **Экспоненциальный backoff без cap.** Растёт бесконечно.
- ❌ **Нет jitter.** Thundering herd.
- ❌ **Нет `context`.** Блокировка навсегда.
- ❌ **Retry permanent ошибки.** 4xx повторятся.
- ❌ **Retry не-идемпотентные операции.** Задвоение.
- ❌ **Retry при open breaker.** Ухудшает ситуацию.
- ❌ **Слишком много попыток.** 10+ — опасно.
- ❌ **Нет метрик.** Не видно эффекта.

---

## 16.10 Для быстрого повторения

- **Retry** — паттерн повторных попыток для временных ошибок.
- **N попыток 3–5** — разумно.
- **Экспоненциальный backoff** — стандарт.
- **Base delay 50–200 мс.** **Max delay 1–5 сек.**
- **Jitter** — случайный разброс, ± 50%.
- **`context`** — для отмены.
- **Permanent ошибки** (4xx, кроме 429) — не retry.
- **Тип `PermanentError`** — для постоянных.
- **`errors.As`** — проверка.
- **Retry + circuit breaker** — полная защита.
- **Порядок:** rate limiter → retry → breaker → HTTP.
- **Метрики:** attempts, successes, failures, permanentErr.

---

## 16.11 Вопросы для самопроверки

1. Что такое retry? Какую задачу решает?
2. Чем retry отличается от circuit breaker?
3. Что такое экспоненциальный backoff?
4. Зачем нужен jitter?
5. Зачем `context` в retry?
6. Какие ошибки не нужно повторять?
7. Как комбинировать retry с circuit breaker?
8. Что будет, если retry без задержки?

---

## 16.12 Ответы

### Ответ 1

**Retry** — паттерн повторных попыток для **временных** ошибок. Решает задачу: **не отдавать клиенту ошибку**, если она может быть временной (сеть, перегрузка, БД).

### Ответ 2

**Retry** повторяет операции. **Circuit breaker** прекращает вызовы, если сервис **гарантированно** не работает.

**Вместе:** retry повторяет до N раз; если N раз упало — circuit breaker открывается.

### Ответ 3

**Экспоненциальный backoff** — задержка **удваивается** с каждой попыткой:

```
attempt 1: 100 мс
attempt 2: 200 мс
attempt 3: 400 мс
attempt 4: 800 мс
```

**Зачем:** первая попытка быстро, следующая через время, чтобы сервис восстановился.

**Cap:** максимальная задержка (1–5 сек), чтобы backoff не рос бесконечно.

### Ответ 4

**Jitter** — случайный разброс задержки. Предотвращает **thundering herd**: когда 1000 клиентов одновременно делают retry — все попадают в одну точку времени.

**С jitter:** запросы **распределяются** во времени.

**Реализация:** `delay ± 50%` или `0..delay`.

### Ответ 5

**`context`** позволяет остановить retry при отмене. Без него retry может **блокироваться навсегда** или работать, даже если клиент ушёл.

**Решение:** `select` с `ctx.Done()` при ожидании, `fn(ctx)` для передачи контекста.

### Ответ 6

**Permanent ошибки:**
- 400 Bad Request.
- 401 Unauthorized.
- 403 Forbidden.
- 404 Not Found.
- 405 Method Not Allowed.
- 501 Not Implemented.

**Retry не поможет.** Эти ошибки повторятся.

### Ответ 7

**Retry + circuit breaker:**

```go
for attempt := 0; attempt < 3; attempt++ {
    err := breaker.Call(ctx, func(ctx context.Context) error {
        return http.Get(url)
    })
    if err == nil {
        return nil
    }
    if errors.Is(err, ErrCircuitOpen) {
        return err  // не retry
    }
    time.Sleep(backoff)
}
```

Retry вокруг breaker. Не retry при открытом breaker.

### Ответ 8

**Retry без задержки:** DDOS на сервис. Если 1000 клиентов получили ошибку — все одновременно делают retry. Сервис **снова** перегружается.

**Решение:** экспоненциальный backoff + jitter.

---

## 16.13 Куда идти дальше?

Мы разобрали retry — повторные попытки. Теперь мы умеем не отдавать клиенту временные ошибки.

Но иногда операция **зависает**. Нужно **ограничить** время выполнения.

- **Как ограничить время операции?** → **Глава 17: Timeout.**
- **Как ограничить частоту операций?** → **Глава 18: Debounce и Throttle.**
- **Как построить pipeline?** → **Глава 11: Pipeline.**

---

## 16.14 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Retry** | Повторные попытки | Для временных ошибок |
| **N попыток** | 3–5 | Разумно |
| **Задержка** | Между попытками | Обязательна |
| **Экспоненциальный backoff** | Удвоение | Стандарт |
| **Base delay** | 50–200 мс | Первая задержка |
| **Max delay** | 1–5 сек | Cap |
| **Jitter** | ± 50% | Предотвращает thundering herd |
| **`context`** | Отмена | `select` с `ctx.Done()` |
| **Permanent ошибки** | 4xx | Не retry |
| **`PermanentError`** | Тип | Для постоянных |
| **`errors.As`** | Проверка | — |
| **Retry + breaker** | Полная защита | Не retry при open |
| **Порядок** | Rate → retry → breaker → HTTP | — |
| **Метрики** | attempts, successes, failures | PermanentErr |

🔁 **Ключевая идея:** Retry — паттерн повторных попыток для временных ошибок. **N попыток 3–5**. **Экспоненциальный backoff** (base 50–200 мс, max 1–5 сек) — стандарт. **Jitter** (± 50%) предотвращает thundering herd. **`context`** для отмены. **Permanent ошибки** (4xx, кроме 429) не retry. Тип `PermanentError` + `errors.As`. Комбинируется с circuit breaker (не retry при open), rate limiter (rate → retry → breaker → HTTP), worker pool. Метрики: attempts, successes, failures, permanentErr. Не retry без задержки — DDOS на сервис.