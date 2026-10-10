# ⚡ Глава 15: Circuit breaker — защита от отказов

**Что вы узнаете:**
- Что такое circuit breaker и какую задачу он решает.
- Три состояния circuit breaker: Closed, Open, Half-Open.
- Как построить circuit breaker с нуля.
- Как настроить threshold и timeout.
- Как обрабатывать ошибки и успехи.
- Как комбинировать circuit breaker с retry, rate limiter, worker pool.
- Как использовать `github.com/sony/gobreaker`.

**После прочтения вы сможете:**
- Построить circuit breaker с нуля.
- Настраивать threshold и timeout.
- Обрабатывать переходы между состояниями.
- Использовать circuit breaker в HTTP-клиентах.
- Комбинировать circuit breaker с другими паттернами.
- Понимать, где circuit breaker уместен, а где — нет.

---

## Содержание

- [15.0 Пролог: каскадный отказ](#150-пролог-каскадный-отказ)
- [15.1 Что такое circuit breaker](#151-что-такое-circuit-breaker)
- [15.2 Простейший circuit breaker](#152-простейший-circuit-breaker)
- [15.3 Circuit breaker с threshold и timeout](#153-circuit-breaker-с-threshold-и-timeout)
- [15.4 Half-Open: пробные запросы](#154-half-open-пробные-запросы)
- [15.5 Circuit breaker с context](#155-circuit-breaker-с-context)
- [15.6 gobreaker: production-ready](#156-gobreaker-production-ready)
- [15.7 В связке с другими паттернами](#157-в-связке-с-другими-паттернами)
- [15.8 Практика Go: circuit breaker с метриками](#158-практика-go-circuit-breaker-с-метриками)
- [15.9 Выводы и типичные ошибки](#159-выводы-и-типичные-ошибки)
- [15.10 Для быстрого повторения](#1510-для-быстрого-повторения)
- [15.11 Вопросы для самопроверки](#1511-вопросы-для-самопроверки)
- [15.12 Ответы](#1512-ответы)
- [15.13 Куда идти дальше?](#1513-куда-идти-дальше)
- [15.14 Чек-лист](#1514-чек-лист)

---

## 15.0 Пролог: каскадный отказ

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

Работает. Пока внешний сервис не начал **тормозить**. Запрос висит 5 секунд. Клиенты ждут. Горутины копятся. Память растёт.

Через минуту внешний сервис падает совсем. Все 100% запросов к нему возвращают ошибку **за 5 секунд** (timeout). Наш сервис ждёт 5 секунд на каждый запрос, потом возвращает 500.

**Что происходит:**

- Клиенты ждут 5 секунд впустую.
- Горутины копятся (10 000 одновременно).
- Память растёт.
- Мы **знаем**, что внешний сервис **не работает**, но продолжаем его вызывать.

Хочется **перестать вызывать** внешний сервис, когда он **гарантированно** не работает. Не ждать 5 секунд впустую, а **сразу вернуть ошибку**. И периодически **проверять**, не восстановился ли сервис.

Это и есть **circuit breaker**.

> **Мост к следующим главам:** circuit breaker — важный инструмент для защиты от каскадных отказов. Он часто используется вместе с rate limiter (Глава 14) и retry (Глава 16). Понимание circuit breaker даёт понимание, **как не рушить свой сервис чужими проблемами**.

---

## 15.1 Что такое circuit breaker

**Circuit breaker** — паттерн, который **прекращает** вызовы к сломанному сервису.

### Идея

Как **электрический предохранитель**: при коротком замыкании он **разрывает цепь**, чтобы не сгорела вся система.

### Три состояния

**1. Closed (закрыт).**

Все запросы **проходят** к внешнему сервису. Считаем ошибки.

**2. Open (открыт).**

Все запросы **отклоняются** немедленно. Не ждём внешний сервис. Ждём `timeout`.

**3. Half-Open (полуоткрыт).**

Пропускаем **1 пробный запрос**. Если успех — возвращаемся в Closed. Если ошибка — снова Open.

### Схема

```
                       Ошибок > threshold
     ┌────────┐  ─────────────────────────▶  ┌────────┐
     │ Closed │                               │  Open  │
     └────────┘  ◀─────────────────────────  └────────┘
         ▲                Успех                  │
         │                                       │
         │                                  timeout
         │                                       │
         │          ┌──────────┐               │
         └──────────│Half-Open │◀──────────────┘
            Успех   └──────────┘
                          │
                     Ошибка
                          │
                          ▼
                     ┌────────┐
                     │  Open  │
                     └────────┘
```

### Когда использовать circuit breaker

**1. Внешние сервисы.**

- HTTP API.
- gRPC.
- БД.
- Другие внутренние сервисы.

**2. Медленные сервисы.**

- Если сервис может тормозить — circuit breaker защитит.
- Если сервис быстрый — не нужен.

**3. Критичные пути.**

- Если сбой сервиса ломает весь наш сервис — нужен circuit breaker.

### Когда НЕ использовать circuit breaker

**1. Внутренние операции.**

Для внутренних функций — overhead.

**2. Быстрые сервисы.**

Если сервис отвечает за 1 мс — circuit breaker не нужен.

**3. Одиночные вызовы.**

Для редких операций — overhead.

### Circuit breaker vs retry

**Retry** — повторяет неудачные операции. **Circuit breaker** — прекращает вызовы.

**Вместе:**

- Retry: пробуем N раз.
- Circuit breaker: если N раз упало — прекращаем вызывать.

### 💡 Практика: как думать о circuit breaker

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Circuit breaker — для внешних сервисов.**
2. **Threshold 5–10 ошибок.**
3. **Timeout 10–60 секунд.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Circuit breaker + retry** — полная защита.
5. **Circuit breaker + rate limiter** — для API.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй circuit breaker для внутренних операций.**
7. **Не ставь threshold 1** — ложные срабатывания.

---

## 15.2 Простейший circuit breaker

Начнём с самого простого — circuit breaker, который считает **последовательные** ошибки.

### Идея

- При ошибке — увеличиваем счётчик.
- При успехе — сбрасываем счётчик.
- Если счётчик > threshold — открываем.

### Реализация

```go
type SimpleBreaker struct {
    threshold  int
    failures   int
    state      State
    mu         sync.Mutex
}

type State int

const (
    StateClosed State = iota
    StateOpen
)

var ErrCircuitOpen = errors.New("circuit breaker is open")

func NewSimpleBreaker(threshold int) *SimpleBreaker {
    return &SimpleBreaker{
        threshold: threshold,
        state:     StateClosed,
    }
}

func (b *SimpleBreaker) Call(fn func() error) error {
    b.mu.Lock()
    if b.state == StateOpen {
        b.mu.Unlock()
        return ErrCircuitOpen
    }
    b.mu.Unlock()
    
    err := fn()
    
    b.mu.Lock()
    defer b.mu.Unlock()
    
    if err != nil {
        b.failures++
        if b.failures >= b.threshold {
            b.state = StateOpen
        }
    } else {
        b.failures = 0
    }
    return err
}
```

**Что происходит:**

1. `Call` проверяет состояние.
2. Если Open — возвращает `ErrCircuitOpen`.
3. Иначе вызывает `fn`.
4. При ошибке — счётчик увеличивается.
5. При успехе — счётчик сбрасывается.

### Потребитель

```go
func main() {
    breaker := NewSimpleBreaker(3)
    
    for i := 0; i < 10; i++ {
        err := breaker.Call(func() error {
            return errors.New("service unavailable")
        })
        fmt.Printf("Call %d: %v\n", i, err)
    }
}
```

**Пример вывода:**

```
Call 0: service unavailable
Call 1: service unavailable
Call 2: service unavailable
Call 3: circuit breaker is open
Call 4: circuit breaker is open
Call 5: circuit breaker is open
Call 6: circuit breaker is open
Call 7: circuit breaker is open
Call 8: circuit breaker is open
Call 9: circuit breaker is open
```

**Что видно:** после 3 ошибок circuit breaker **открыт**. Все последующие вызовы **отклоняются** немедленно.

### Проблема: не восстановится

**Что если сервис восстановился?** Circuit breaker останется Open **навсегда**. Нужно **Half-Open**.

### Проблема: нет timeout

Без timeout circuit breaker не знает, когда **пробовать** восстановление.

### Схема

```
Call(fn):

  ┌───────────────────────┐
  │ state == Open?        │
  │   → ErrCircuitOpen    │
  └───────────┬───────────┘
              │
              │ Нет
              ▼
  ┌───────────────────────┐
  │ err := fn()           │
  └───────────┬───────────┘
              │
              ▼
  ┌───────────────────────┐
  │ err != nil?           │
  │   failures++          │
  │   if failures >= N:   │
  │     state = Open      │
  │ else:                 │
  │   failures = 0        │
  └───────────────────────┘
```

### 💡 Практика: как писать простой circuit breaker

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Счётчик ошибок.**
2. **`Mutex` для состояния.**
3. **`ErrCircuitOpen`.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`Call(fn)` — универсальный интерфейс.**

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай про Half-Open.**
6. **Не блокируйся навсегда при Open.**

---

## 15.3 Circuit breaker с threshold и timeout

Добавим timeout — через какое время пробовать **Half-Open**.

### Идея

- **Closed:** считаем ошибки. Если > threshold → Open.
- **Open:** все запросы отклоняются. Ждём `timeout`.
- **После timeout:** переходим в Half-Open.

### Реализация

```go
type CircuitBreaker struct {
    threshold       int
    timeout         time.Duration
    failures        int
    lastFailureTime time.Time
    state           State
    mu              sync.Mutex
}

type State int

const (
    StateClosed State = iota
    StateOpen
    StateHalfOpen
)

var ErrCircuitOpen = errors.New("circuit breaker is open")

func NewCircuitBreaker(threshold int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        threshold: threshold,
        timeout:   timeout,
        state:     StateClosed,
    }
}

func (b *CircuitBreaker) Call(fn func() error) error {
    if err := b.beforeCall(); err != nil {
        return err
    }
    
    err := fn()
    b.afterCall(err)
    return err
}

func (b *CircuitBreaker) beforeCall() error {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    switch b.state {
    case StateClosed:
        return nil
    case StateOpen:
        if time.Since(b.lastFailureTime) > b.timeout {
            b.state = StateHalfOpen
            return nil  // пробный запрос
        }
        return ErrCircuitOpen
    case StateHalfOpen:
        return nil  // один пробный
    }
    return nil
}

func (b *CircuitBreaker) afterCall(err error) {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    if err != nil {
        b.failures++
        b.lastFailureTime = time.Now()
        if b.failures >= b.threshold {
            b.state = StateOpen
        }
        if b.state == StateHalfOpen {
            b.state = StateOpen  // ошибка в Half-Open
        }
    } else {
        b.failures = 0
        if b.state == StateHalfOpen {
            b.state = StateClosed
        }
    }
}
```

**Что происходит:**

- **Closed:** считаем ошибки.
- **Open:** ждём `timeout`.
- **Half-Open:** пробный запрос.
  - Успех → Closed.
  - Ошибка → Open.

### Полный пример

```go
package main

import (
    "errors"
    "fmt"
    "sync"
    "time"
)

type State int

const (
    StateClosed State = iota
    StateOpen
    StateHalfOpen
)

var ErrCircuitOpen = errors.New("circuit breaker is open")

type CircuitBreaker struct {
    threshold       int
    timeout         time.Duration
    failures        int
    lastFailureTime time.Time
    state           State
    mu              sync.Mutex
}

func NewCircuitBreaker(threshold int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        threshold: threshold,
        timeout:   timeout,
        state:     StateClosed,
    }
}

func (b *CircuitBreaker) Call(fn func() error) error {
    if err := b.beforeCall(); err != nil {
        return err
    }
    
    err := fn()
    b.afterCall(err)
    return err
}

func (b *CircuitBreaker) beforeCall() error {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    switch b.state {
    case StateClosed:
        return nil
    case StateOpen:
        if time.Since(b.lastFailureTime) > b.timeout {
            b.state = StateHalfOpen
            return nil
        }
        return ErrCircuitOpen
    case StateHalfOpen:
        return nil
    }
    return nil
}

func (b *CircuitBreaker) afterCall(err error) {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    if err != nil {
        b.failures++
        b.lastFailureTime = time.Now()
        if b.failures >= b.threshold {
            b.state = StateOpen
        }
        if b.state == StateHalfOpen {
            b.state = StateOpen
        }
    } else {
        b.failures = 0
        if b.state == StateHalfOpen {
            b.state = StateClosed
        }
    }
}

func main() {
    breaker := NewCircuitBreaker(3, 1*time.Second)
    
    // Симулируем отказы
    for i := 0; i < 5; i++ {
        err := breaker.Call(func() error {
            return errors.New("service unavailable")
        })
        fmt.Printf("[%v] Call %d: %v\n", time.Now().Format("15:04:05.000"), i, err)
        time.Sleep(200 * time.Millisecond)
    }
    
    // Ждём timeout
    fmt.Println("waiting for recovery...")
    time.Sleep(1 * time.Second)
    
    // Пробный запрос
    err := breaker.Call(func() error {
        return nil  // успех
    })
    fmt.Printf("[%v] Recovery: %v\n", time.Now().Format("15:04:05.000"), err)
}
```

**Пример вывода:**

```
[12:00:00.000] Call 0: service unavailable
[12:00:00.200] Call 1: service unavailable
[12:00:00.400] Call 2: service unavailable
[12:00:00.600] Call 3: circuit breaker is open
[12:00:00.800] Call 4: circuit breaker is open
waiting for recovery...
[12:00:01.800] Recovery: <nil>
```

**Что видно:**

- После 3 ошибок — Open.
- Через 1 секунду — Half-Open.
- Пробный запрос — успех → Closed.

### Схема

```
StateClosed:                StateOpen:                 StateHalfOpen:
  errors++                    return ErrCircuitOpen       пробный запрос
  if errors >= N:             if timeout passed:          ├─ успех → Closed
    state = Open                state = HalfOpen          └─ ошибка → Open
```

### 💡 Практика: как настроить threshold и timeout

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Threshold 5–10** — для большинства сервисов.
2. **Timeout 10–60 сек** — время на восстановление.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Threshold зависит от RPS.** Если 1000 RPS — 10 ошибок мгновенно. Если 1 RPS — 10 ошибок за 10 сек.
4. **Timeout зависит от сервиса.** Быстрый рестарт — 10 сек. Долгий — 60 сек.

**❌ НЕ ДЕЛАЙ:**

5. **Threshold = 1.** Ложные срабатывания.
6. **Timeout = 1 сек.** Сервис не успеет восстановиться.

---

## 15.4 Half-Open: пробные запросы

**Half-Open** — состояние, в котором мы **пробуем** восстановиться.

### Идея

После `timeout` в Open переходим в Half-Open. Пропускаем **1 пробный** запрос:

- **Успех** → Closed (сервис восстановился).
- **Ошибка** → Open (всё ещё не работает).

### Проблема: несколько пробных

Если **несколько** горутин видят Half-Open **одновременно** — все делают пробные запросы. Внешний сервис может **упасть снова**.

### Решение: ограничение пробных

Добавим счётчик успехов в Half-Open:

```go
type CircuitBreaker struct {
    threshold       int
    timeout         time.Duration
    halfOpenMax     int  // сколько пробных в Half-Open
    failures        int
    successes       int
    lastFailureTime time.Time
    state           State
    mu              sync.Mutex
}

func (b *CircuitBreaker) beforeCall() error {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    switch b.state {
    case StateClosed:
        return nil
    case StateOpen:
        if time.Since(b.lastFailureTime) > b.timeout {
            b.state = StateHalfOpen
            b.successes = 0
            return nil
        }
        return ErrCircuitOpen
    case StateHalfOpen:
        if b.successes >= b.halfOpenMax {
            return ErrCircuitOpen  // уже достаточно пробных
        }
        return nil
    }
    return nil
}

func (b *CircuitBreaker) afterCall(err error) {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    if err != nil {
        b.failures++
        b.lastFailureTime = time.Now()
        if b.failures >= b.threshold {
            b.state = StateOpen
        }
        if b.state == StateHalfOpen {
            b.state = StateOpen
            b.successes = 0
        }
    } else {
        b.failures = 0
        if b.state == StateHalfOpen {
            b.successes++
            if b.successes >= b.halfOpenMax {
                b.state = StateClosed
                b.successes = 0
            }
        }
    }
}
```

**Что происходит:**

- В Half-Open пропускаем `halfOpenMax` пробных.
- Если все успешны — Closed.
- Если хотя бы один упал — Open.

### Схема

```
StateHalfOpen:
  successes = 0
  halfOpenMax = 1

Пробный запрос 1:
  ├─ Успех → successes = 1 → Closed
  └─ Ошибка → Open

StateHalfOpen с halfOpenMax = 3:
  Пробный 1: успех → successes = 1
  Пробный 2: успех → successes = 2
  Пробный 3: успех → successes = 3 → Closed
  
  Если пробный 2 упал → Open
```

### Что выбрать

**halfOpenMax = 1:**

- Строгое восстановление.
- Сервис должен ответить на 1 запрос.

**halfOpenMax = 3:**

- Более надёжно.
- Нужно 3 успешных подряд.

**halfOpenMax = 5:**

- Для критичных сервисов.

### 💡 Практика: как настроить Half-Open

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`halfOpenMax = 1`** — минимально.
2. **`halfOpenMax = 3`** — для надёжности.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Сброс `successes` при ошибке.**

**❌ НЕ ДЕЛАЙ:**

4. **Не пропускай много пробных.**
5. **Не сбрасывай `successes` при успехе.**

---

## 15.5 Circuit breaker с context

`context` — критичен для circuit breaker. Без него `Call` может **блокироваться навсегда**.

### Проблема

```go
err := breaker.Call(func() error {
    return slowOperation()  // ← может висеть
})
```

**Что происходит:** если `slowOperation` виснет, `Call` ждёт. Даже если `ctx` отменён.

### Решение: context в Call

```go
func (b *CircuitBreaker) Call(ctx context.Context, fn func(ctx context.Context) error) error {
    if err := b.beforeCall(); err != nil {
        return err
    }
    
    // Проверка context
    select {
    case <-ctx.Done():
        b.afterCall(ctx.Err())
        return ctx.Err()
    default:
    }
    
    err := fn(ctx)
    b.afterCall(err)
    return err
}
```

**Что происходит:**

- `ctx` передаётся в `fn`.
- `fn` сама проверяет `ctx`.
- `Call` проверяет `ctx` до вызова.

### Полный пример

```go
func (b *CircuitBreaker) Call(ctx context.Context, fn func(ctx context.Context) error) error {
    if err := b.beforeCall(); err != nil {
        return err
    }
    
    select {
    case <-ctx.Done():
        b.afterCall(ctx.Err())
        return ctx.Err()
    default:
    }
    
    err := fn(ctx)
    b.afterCall(err)
    return err
}
```

### Потребитель

```go
func main() {
    breaker := NewCircuitBreaker(3, 1*time.Second)
    ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
    defer cancel()
    
    err := breaker.Call(ctx, func(ctx context.Context) error {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(1 * time.Second):
            return nil
        }
    })
    fmt.Println("error:", err)
}
```

**Пример вывода:**

```
error: context deadline exceeded
```

**Что происходит:** `ctx` отменён через 500 мс. `fn` возвращает `ctx.Err()`.

### Circuit breaker + context в HTTP-клиенте

```go
type HTTPClient struct {
    client  *http.Client
    breaker *CircuitBreaker
}

func (c *HTTPClient) Get(ctx context.Context, url string) (*http.Response, error) {
    var resp *http.Response
    err := c.breaker.Call(ctx, func(ctx context.Context) error {
        req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
        if err != nil {
            return err
        }
        var reqErr error
        resp, reqErr = c.client.Do(req)
        return reqErr
    })
    if err != nil {
        return nil, err
    }
    return resp, nil
}
```

### 💡 Практика: как добавить context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx context.Context` — первый аргумент.**
2. **`fn(ctx)` — передавай ctx в callback.**
3. **Проверяй `ctx.Done()` до вызова.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`afterCall(ctx.Err())` при отмене.**

**❌ НЕ ДЕЛАЙ:**

5. **Не вызывай `fn` без `ctx`.**
6. **Не блокируйся навсегда.**

---

## 15.6 gobreaker: production-ready

Для production — используй `github.com/sony/gobreaker`.

### Установка

```bash
go get github.com/sony/gobreaker
```

### Создание

```go
import "github.com/sony/gobreaker"

cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "my-service",
    MaxRequests: 3,               // пробных в Half-Open
    Interval:    10 * time.Second, // окно для подсчёта ошибок
    Timeout:     30 * time.Second, // Open → Half-Open
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        return counts.ConsecutiveFailures > 5
    },
})
```

**Параметры:**

- **`Name`** — имя для метрик.
- **`MaxRequests`** — пробных в Half-Open.
- **`Interval`** — окно подсчёта ошибок (если 0 — не сбрасывать счётчики).
- **`Timeout`** — время в Open.
- **`ReadyToTrip`** — функция, которая решает, когда открывать.

### Использование

```go
result, err := cb.Execute(func() (interface{}, error) {
    return http.Get("https://api.example.com")
})
```

**Что происходит:** `Execute` вызывает функцию, считает ошибки, переключает состояния.

### Пример

```go
package main

import (
    "errors"
    "fmt"
    "time"
    
    "github.com/sony/gobreaker"
)

func main() {
    cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
        Name:        "my-service",
        MaxRequests: 1,
        Timeout:     1 * time.Second,
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            return counts.ConsecutiveFailures > 3
        },
    })
    
    for i := 0; i < 10; i++ {
        _, err := cb.Execute(func() (interface{}, error) {
            return nil, errors.New("service unavailable")
        })
        fmt.Printf("[%v] Call %d: %v (state=%v)\n",
            time.Now().Format("15:04:05.000"), i, err, cb.State())
        time.Sleep(200 * time.Millisecond)
    }
}
```

**Пример вывода:**

```
[12:00:00.000] Call 0: service unavailable (state=closed)
[12:00:00.200] Call 1: service unavailable (state=closed)
[12:00:00.400] Call 2: service unavailable (state=closed)
[12:00:00.600] Call 3: service unavailable (state=closed)
[12:00:00.800] Call 4: circuit breaker is open (state=open)
[12:00:01.000] Call 5: circuit breaker is open (state=open)
...
```

### Настройки ReadyToTrip

**Последовательные ошибки:**

```go
ReadyToTrip: func(counts gobreaker.Counts) bool {
    return counts.ConsecutiveFailures > 5
}
```

**Процент ошибок:**

```go
ReadyToTrip: func(counts gobreaker.Counts) bool {
    failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
    return counts.Requests >= 10 && failureRatio >= 0.6
}
```

**Комбинация:**

```go
ReadyToTrip: func(counts gobreaker.Counts) bool {
    return counts.ConsecutiveFailures > 5 ||
        (counts.Requests >= 100 && counts.TotalFailures > 50)
}
```

### Сравнение с нашей реализацией

| Аспект | Наш | `gobreaker` |
|:---|:---|:---|
| Threshold | Последовательные ошибки | Гибкий |
| Timeout | ✅ | ✅ |
| Half-Open | ✅ | ✅ |
| Метрики | ❌ | ✅ |
| Гибкость | Ограничена | Высокая |
| Production-ready | ❌ | ✅ |

**Вывод:** наш circuit breaker — для **понимания**. Для production — `gobreaker`.

### 💡 Практика: как использовать gobreaker

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`gobreaker.NewCircuitBreaker`** для production.
2. **`ReadyToTrip`** — настрой под свой сервис.
3. **`Timeout` 30–60 сек.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики из `cb.State()`.**
5. **Логирование переходов.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй свою реализацию в production.**
7. **Не забывай про `Interval`.**

---

## 15.7 В связке с другими паттернами

Circuit breaker редко используется **в одиночку**. Разберём связки.

### Circuit breaker + retry

**Retry** повторяет неудачные операции. **Circuit breaker** прекращает вызовы.

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
        
        // Backoff
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-time.After(time.Duration(attempt+1) * 100 * time.Millisecond):
        }
    }
    return nil, lastErr
}
```

**Что происходит:**

- Retry до 3 раз.
- Circuit breaker между попытками.
- Если breaker открыт — не retry.

### Circuit breaker + rate limiter

**Rate limiter** ограничивает скорость. **Circuit breaker** защищает от сбоев.

```go
type HTTPClient struct {
    client  *http.Client
    limiter *rate.Limiter
    breaker *CircuitBreaker
}

func (c *HTTPClient) Get(ctx context.Context, url string) (*http.Response, error) {
    // 1. Rate limiter
    if err := c.limiter.Wait(ctx); err != nil {
        return nil, err
    }
    
    // 2. Circuit breaker
    var resp *http.Response
    err := c.breaker.Call(ctx, func(ctx context.Context) error {
        req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
        if err != nil {
            return err
        }
        var reqErr error
        resp, reqErr = c.client.Do(req)
        return reqErr
    })
    if err != nil {
        return nil, err
    }
    return resp, nil
}
```

**Порядок:**

1. **Rate limiter** — первым.
2. **Circuit breaker** — вторым.

**Почему:** rate limiter не тратит ресурсы, если breaker открыт.

### Circuit breaker + worker pool

**Worker pool** обрабатывает задачи. **Circuit breaker** защищает внешние вызовы.

```go
func worker(ctx context.Context, tasksCh <-chan Task, breaker *CircuitBreaker) {
    for {
        select {
        case <-ctx.Done():
            return
        case task, ok := <-tasksCh:
            if !ok {
                return
            }
            err := breaker.Call(ctx, func(ctx context.Context) error {
                return process(ctx, task)
            })
            if err != nil {
                if errors.Is(err, ErrCircuitOpen) {
                    // breaker открыт — пропускаем задачу или ждём
                }
                continue
            }
        }
    }
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
│   Circuit    │  ← защита от сбоев
│   breaker    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│    Retry     │  ← повтор при временных ошибках
└──────┬───────┘
       │
       ▼
     HTTP
```

### 💡 Практика: как комбинировать circuit breaker

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Circuit breaker + retry** — полная защита.
2. **Rate limiter → circuit breaker → retry → HTTP.**
3. **Circuit breaker + worker pool** — для внешних вызовов.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Разные circuit breaker для разных сервисов.**

**❌ НЕ ДЕЛАЙ:**

5. **Не retry при открытом breaker.**
6. **Не забывай про порядок.**

---

## 15.8 Практика Go: circuit breaker с метриками

Разберём **circuit breaker с метриками**.

### Полный код

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "sync"
    "sync/atomic"
    "time"
)

type State int

const (
    StateClosed State = iota
    StateOpen
    StateHalfOpen
)

func (s State) String() string {
    switch s {
    case StateClosed:
        return "closed"
    case StateOpen:
        return "open"
    case StateHalfOpen:
        return "half-open"
    }
    return "unknown"
}

var ErrCircuitOpen = errors.New("circuit breaker is open")

type Metrics struct {
    Requests      atomic.Int64
    Successes     atomic.Int64
    Failures      atomic.Int64
    Rejected      atomic.Int64
    StateChanges  atomic.Int64
}

type CircuitBreaker struct {
    threshold       int
    timeout         time.Duration
    halfOpenMax     int
    failures        int
    successes       int
    lastFailureTime time.Time
    state           State
    mu              sync.Mutex
    metrics         *Metrics
}

func NewCircuitBreaker(threshold int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        threshold:   threshold,
        timeout:     timeout,
        halfOpenMax: 1,
        state:       StateClosed,
        metrics:     &Metrics{},
    }
}

func (b *CircuitBreaker) Call(ctx context.Context, fn func(ctx context.Context) error) error {
    b.metrics.Requests.Add(1)
    
    if err := b.beforeCall(); err != nil {
        b.metrics.Rejected.Add(1)
        return err
    }
    
    select {
    case <-ctx.Done():
        b.afterCall(ctx.Err())
        return ctx.Err()
    default:
    }
    
    err := fn(ctx)
    b.afterCall(err)
    return err
}

func (b *CircuitBreaker) beforeCall() error {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    switch b.state {
    case StateClosed:
        return nil
    case StateOpen:
        if time.Since(b.lastFailureTime) > b.timeout {
            b.setState(StateHalfOpen)
            b.successes = 0
            return nil
        }
        return ErrCircuitOpen
    case StateHalfOpen:
        if b.successes >= b.halfOpenMax {
            return ErrCircuitOpen
        }
        return nil
    }
    return nil
}

func (b *CircuitBreaker) afterCall(err error) {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    if err != nil {
        b.metrics.Failures.Add(1)
        b.failures++
        b.lastFailureTime = time.Now()
        
        if b.failures >= b.threshold {
            b.setState(StateOpen)
        }
        if b.state == StateHalfOpen {
            b.setState(StateOpen)
            b.successes = 0
        }
    } else {
        b.metrics.Successes.Add(1)
        b.failures = 0
        if b.state == StateHalfOpen {
            b.successes++
            if b.successes >= b.halfOpenMax {
                b.setState(StateClosed)
                b.successes = 0
            }
        }
    }
}

func (b *CircuitBreaker) setState(s State) {
    if b.state != s {
        b.state = s
        b.metrics.StateChanges.Add(1)
        fmt.Printf("[%v] State changed: %v\n", time.Now().Format("15:04:05.000"), s)
    }
}

func (b *CircuitBreaker) State() State {
    b.mu.Lock()
    defer b.mu.Unlock()
    return b.state
}

func main() {
    breaker := NewCircuitBreaker(3, 1*time.Second)
    ctx := context.Background()
    
    // Фаза 1: отказы
    for i := 0; i < 5; i++ {
        err := breaker.Call(ctx, func(ctx context.Context) error {
            return errors.New("service unavailable")
        })
        fmt.Printf("Call %d: %v\n", i, err)
        time.Sleep(100 * time.Millisecond)
    }
    
    // Фаза 2: ждём восстановления
    fmt.Println("waiting...")
    time.Sleep(1 * time.Second)
    
    // Фаза 3: восстановление
    err := breaker.Call(ctx, func(ctx context.Context) error {
        return nil
    })
    fmt.Printf("Recovery: %v\n", err)
    
    // Метрики
    fmt.Println("\n=== Metrics ===")
    fmt.Printf("Requests:     %d\n", breaker.metrics.Requests.Load())
    fmt.Printf("Successes:    %d\n", breaker.metrics.Successes.Load())
    fmt.Printf("Failures:     %d\n", breaker.metrics.Failures.Load())
    fmt.Printf("Rejected:     %d\n", breaker.metrics.Rejected.Load())
    fmt.Printf("StateChanges: %d\n", breaker.metrics.StateChanges.Load())
}
```

**Пример вывода:**

```
[12:00:00.000] State changed: open
Call 0: service unavailable
Call 1: service unavailable
Call 2: service unavailable
[12:00:00.300] State changed: open
Call 3: circuit breaker is open
Call 4: circuit breaker is open
waiting...
[12:00:01.400] State changed: half-open
[12:00:01.400] State changed: closed
Recovery: <nil>

=== Metrics ===
Requests:     6
Successes:    1
Failures:     3
Rejected:     2
StateChanges: 3
```

### Что демонстрирует

1. **3 состояния** с переходами.
2. **Метрики:** requests, successes, failures, rejected, stateChanges.
3. **Логирование** переходов.
4. **Восстановление** через Half-Open.

### 💡 Практика: как измерять circuit breaker

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики:** requests, successes, failures, rejected.
2. **State changes.**
3. **Логирование переходов.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Экспорт в Prometheus.**
5. **Алерт при частых открытиях.**

**❌ НЕ ДЕЛАЙ:**

6. **Не игнорируй circuit breaker в метриках.**
7. **Не забывай про логирование.**

---

## 15.9 Выводы и типичные ошибки

**Что мы узнали?**

Circuit breaker — паттерн для защиты от каскадных отказов. **Три состояния:** Closed (пропускает), Open (отклоняет), Half-Open (пробные запросы). **Threshold** — число ошибок до открытия. **Timeout** — время в Open. **Half-Open** — 1–3 пробных запроса. **`context`** — для отмены. **`gobreaker`** — production-ready. Circuit breaker комбинируется с retry, rate limiter, worker pool.

**Типичные ошибки:**

- ❌ **Threshold = 1.** Ложные срабатывания.
- ❌ **Timeout = 1 сек.** Сервис не восстановится.
- ❌ **Не использовать Half-Open.** Breaker не восстановится.
- ❌ **Блокироваться на `Call` без `ctx`.**
- ❌ **Retry при открытом breaker.**
- ❌ **Circuit breaker для внутренних операций.**
- ❌ **Не мониторить состояние.**
- ❌ **Не логировать переходы.**
- ❌ **Использовать свою реализацию в production.**
- ❌ **Один breaker на все сервисы.**

---

## 15.10 Для быстрого повторения

- **Circuit breaker** — защита от каскадных отказов.
- **Три состояния:** Closed, Open, Half-Open.
- **Closed:** пропускает. **Open:** отклоняет. **Half-Open:** пробные.
- **Threshold 5–10** — ошибок до открытия.
- **Timeout 10–60 сек** — время в Open.
- **Half-Open max 1–3** — пробных запросов.
- **`context`** — для отмены.
- **`ErrCircuitOpen`** — ошибка.
- **`gobreaker`** — production-ready.
- **Circuit breaker + retry** — полная защита.
- **Rate limiter → circuit breaker → retry → HTTP.**
- **Метрики:** requests, successes, failures, rejected, stateChanges.

---

## 15.11 Вопросы для самопроверки

1. Что такое circuit breaker? Какую задачу решает?
2. Назови три состояния.
3. Что такое threshold и timeout?
4. Что такое Half-Open?
5. Зачем `context` в circuit breaker?
6. Как комбинировать circuit breaker с retry?
7. Что будет, если не использовать Half-Open?
8. Как использовать `gobreaker`?

---

## 15.12 Ответы

### Ответ 1

**Circuit breaker** — паттерн для защиты от каскадных отказов. **Прекращает** вызовы к сломанному сервису. Не ждёт таймаут, а **сразу возвращает ошибку**.

**Когда:** внешние сервисы с медленными или частыми отказами.

### Ответ 2

**Три состояния:**
1. **Closed** — пропускает запросы.
2. **Open** — отклоняет запросы немедленно.
3. **Half-Open** — пропускает пробные запросы.

### Ответ 3

**Threshold** — число ошибок до открытия. 5–10 для большинства.

**Timeout** — время в Open до перехода в Half-Open. 10–60 сек.

### Ответ 4

**Half-Open** — состояние после timeout в Open. Пропускает **1–3 пробных** запроса:
- **Успех** → Closed.
- **Ошибка** → Open.

**Зачем:** проверить, восстановился ли сервис.

### Ответ 5

**`context`** позволяет не блокироваться навсегда, если `fn` виснет. Без него `Call` может ждать бесконечно.

**Решение:** `ctx` — первый аргумент, `fn(ctx)`.

### Ответ 6

**Circuit breaker + retry:**

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
    time.Sleep(backoff(attempt))
}
```

Retry до N раз, но не при открытом breaker.

### Ответ 7

**Без Half-Open:** breaker останется Open **навсегда**. Даже если сервис восстановился — все запросы отклоняются.

**С Half-Open:** периодически пробуем восстановиться.

### Ответ 8

```go
cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "my-service",
    MaxRequests: 3,
    Timeout:     30 * time.Second,
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        return counts.ConsecutiveFailures > 5
    },
})

result, err := cb.Execute(func() (interface{}, error) {
    return http.Get("https://api.example.com")
})
```

---

## 15.13 Куда идти дальше?

Мы разобрали circuit breaker — защиту от каскадных отказов. Теперь мы умеем не рушить свой сервис чужими проблемами.

Но иногда временные ошибки **стоит** повторить. Как это делать правильно?

- **Как повторять неудачные операции?** → **Глава 16: Retry.**
- **Как ограничить время операции?** → **Глава 17: Timeout.**
- **Как ограничить частоту операций?** → **Глава 18: Debounce и Throttle.**

---

## 15.14 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Circuit breaker** | Защита от отказов | Три состояния |
| **Closed** | Пропускает | Считает ошибки |
| **Open** | Отклоняет | Ждёт timeout |
| **Half-Open** | Пробные | 1–3 запроса |
| **Threshold** | 5–10 | Ошибок до открытия |
| **Timeout** | 10–60 сек | Время в Open |
| **`ErrCircuitOpen`** | Ошибка | При открытом breaker |
| **`context`** | Отмена | В `Call` |
| **`gobreaker`** | Production | `NewCircuitBreaker` |
| **`ReadyToTrip`** | Функция | Когда открывать |
| **+ retry** | Полная защита | Не retry при open |
| **+ rate limiter** | Порядок | Rate → breaker → retry |
| **Метрики** | requests, failures, rejected | State changes |

⚡ **Ключевая идея:** Circuit breaker — паттерн для защиты от каскадных отказов. **Три состояния:** Closed (пропускает), Open (отклоняет), Half-Open (пробные). **Threshold 5–10**, **timeout 10–60 сек**, **halfOpenMax 1–3**. **`context`** для отмены. **`gobreaker`** — production-ready. Комбинируется с retry (не retry при open), rate limiter (rate → breaker → retry → HTTP), worker pool. Метрики: requests, successes, failures, rejected, stateChanges. Не используй свою реализацию в production — `gobreaker` лучше.