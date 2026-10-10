# 🧱 Глава 22: Bulkhead — изоляция ресурсов

**Что вы узнаете:**
- Что такое bulkhead и какую задачу он решает.
- Чем bulkhead отличается от semaphore и worker pool.
- Как построить bulkhead с нуля.
- Как разделить ресурсы между разными типами запросов.
- Как добавить отмену через `context`.
- Как обрабатывать переполнение (overflow).
- Как комбинировать bulkhead с rate limiter, circuit breaker, retry.

**После прочтения вы сможете:**
- Построить bulkhead с нуля.
- Изолировать ресурсы между разными частями системы.
- Понимать, где bulkhead уместен, а где — нет.
- Комбинировать bulkhead с другими паттернами.
- Избегать каскадных отказов.

---

## Содержание

- [21.0 Пролог: один медленный сервис кладёт всё](#210-пролог-один-медленный-сервис-кладёт-всё)
- [21.1 Что такое bulkhead](#211-что-такое-bulkhead)
- [21.2 Bulkhead с нуля](#212-bulkhead-с-нуля)
- [21.3 Bulkhead с context](#213-bulkhead-с-context)
- [21.4 Overflow: что делать, когда всё занято](#214-overflow-что-делать-когда-всё-занято)
- [21.5 Bulkhead для разных типов запросов](#215-bulkhead-для-разных-типов-запросов)
- [21.6 Bulkhead vs semaphore vs worker pool](#216-bulkhead-vs-semaphore-vs-worker-pool)
- [21.7 В связке с другими паттернами](#217-в-связке-с-другими-паттернами)
- [21.8 Практика Go: bulkhead с метриками](#218-практика-go-bulkhead-с-метриками)
- [21.9 Выводы и типичные ошибки](#219-выводы-и-типичные-ошибки)
- [21.10 Для быстрого повторения](#2110-для-быстрого-повторения)
- [21.11 Вопросы для самопроверки](#2111-вопросы-для-самопроверки)
- [21.12 Ответы](#2112-ответы)
- [21.13 Куда идти дальше?](#2113-куда-идти-дальше)
- [21.14 Чек-лист](#2114-чек-лист)

---

## 21.0 Пролог: один медленный сервис кладёт всё

У нас есть API-сервис с несколькими эндпоинтами:
- `/users` — читает из БД (быстро).
- `/orders` — читает из БД + вызывает сервис заказов (медленно).
- `/recommendations` — вызывает ML-сервис (очень медленно).

Пишем просто:

```go
func main() {
    // Общий пул воркеров для всего
    tasksCh := make(chan Task, 100)
    
    for i := 0; i < 50; i++ {
        go worker(tasksCh)
    }
    
    http.HandleFunc("/users", handleUsers(tasksCh))
    http.HandleFunc("/orders", handleOrders(tasksCh))
    http.HandleFunc("/recommendations", handleRecommendations(tasksCh))
    
    http.ListenAndServe(":8080", nil)
}
```

Работает. Пока ML-сервис не начал **тормозить**. Каждый запрос к `/recommendations` занимает **30 секунд**. 50 воркеров **заняты** этими запросами. Запросы к `/users` — **в очереди**. Пользователи жалуются: сайт не грузится.

Хочется **изолировать** ресурсы: медленный ML-сервис не должен **занимать** воркеров, которые нужны для `/users`.

Это и есть **bulkhead** — изоляция ресурсов между разными частями системы.

> **Мост к следующим главам:** bulkhead — важный паттерн для production. Он отличается от semaphore (Глава 12) и worker pool (Глава 13) тем, что **разделяет** ресурсы между разными типами запросов. Понимание bulkhead даёт понимание, **как не дать одному сервису положить всё**.

---

## 21.1 Что такое bulkhead

**Bulkhead** — паттерн, при котором **ресурсы разделены** между разными частями системы.

### Идея

Название от **«переборок» на корабле** — водонепроницаемых отсеков. Если один отсек затопило, остальные **остаются сухими**. Корабль не тонет.

**Bulkhead в ПО:** разделяем **пулы воркеров** (или семафоры) между разными частями системы.

```
┌─────────────────────────────────────────┐
│  API-сервис                              │
│                                          │
│  ┌──────────────┐  ┌──────────────┐    │
│  │ Пул /users   │  │ Пул /orders  │    │
│  │ 10 воркеров  │  │ 10 воркеров  │    │
│  └──────────────┘  └──────────────┘    │
│                                          │
│  ┌──────────────┐                        │
│  │ Пул /recs    │                        │
│  │ 20 воркеров  │                        │
│  └──────────────┘                        │
│                                          │
└─────────────────────────────────────────┘
```

**Ключевое:** медленный `/recommendations` **не занимает** воркеров для `/users`. Даже если `/recommendations` полностью забит — `/users` работает.

### Когда использовать bulkhead

**1. Разные типы запросов.**

- Быстрые + медленные.
- Критичные + некритичные.
- Разные SLA.

**2. Разные внешние зависимости.**

- Разные сервисы.
- Разные БД.
- Разные API.

**3. Защита от каскадных отказов.**

- Один сервис не должен ронять весь.
- Изоляция от медленных соседей.

### Когда НЕ использовать bulkhead

**1. Однотипные запросы.**

Если все запросы одинаковы — bulkhead не нужен. Достаточно worker pool.

**2. Мало ресурсов.**

Если у тебя 4 воркера — делить их на 4 части плохо.

**3. Высокая пропускная способность.**

Если все запросы быстрые — bulkhead добавляет overhead.

### Bulkhead vs semaphore

| Аспект | Semaphore | Bulkhead |
|:---|:---|:---|
| Что ограничивает | Параллелизм | Параллелизм для **группы** |
| Гранулярность | Один пул | Несколько пулов |
| Изоляция | Нет | Да |
| Overflow | Ждёт | Можно reject |

### 💡 Практика: как думать о bulkhead

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Bulkhead — для разных типов запросов.**
2. **Bulkhead — для изоляции от медленных соседей.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Разные пулы для разных эндпоинтов.**
4. **Метрики по каждому пулу.**

**❌ НЕ ДЕЛАЙ:**

5. **Не используй bulkhead для однотипных запросов.**
6. **Не дели 4 воркера на 4 части.**

---

## 21.2 Bulkhead с нуля

Начнём с простейшего — **bulkhead на семафоре**.

### Идея

- У каждого типа запроса — **свой семафор**.
- Быстрые запросы — семафор на N.
- Медленные запросы — семафор на M.
- Не пересекаются.

### Реализация

```go
type Bulkhead struct {
    sem chan struct{}
}

func NewBulkhead(maxConcurrent int) *Bulkhead {
    return &Bulkhead{
        sem: make(chan struct{}, maxConcurrent),
    }
}

func (b *Bulkhead) Execute(ctx context.Context, fn func() error) error {
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
    case <-ctx.Done():
        return ctx.Err()
    }
    return fn()
}
```

**Что происходит:**

- `Execute` захватывает слот.
- Вызывает `fn`.
- Освобождает слот.

### Потребитель

```go
var (
    usersBulkhead = NewBulkhead(10)
    ordersBulkhead = NewBulkhead(10)
    recsBulkhead = NewBulkhead(20)
)

func handleUsers(w http.ResponseWriter, r *http.Request) {
    err := usersBulkhead.Execute(r.Context(), func() error {
        return fetchUsers(r.Context())
    })
    if err != nil {
        http.Error(w, err.Error(), 503)
        return
    }
    w.WriteHeader(http.StatusOK)
}

func handleOrders(w http.ResponseWriter, r *http.Request) {
    err := ordersBulkhead.Execute(r.Context(), func() error {
        return fetchOrders(r.Context())
    })
    // ...
}

func handleRecommendations(w http.ResponseWriter, r *http.Request) {
    err := recsBulkhead.Execute(r.Context(), func() error {
        return fetchRecommendations(r.Context())
    })
    // ...
}
```

**Что происходит:** каждый эндпоинт имеет **свой** семафор. Медленные `/recommendations` **не занимают** воркеров `/users`.

### Схема

```
Запрос /users ──► usersBulkhead (10) ──► БД
Запрос /orders ──► ordersBulkhead (10) ──► БД + API
Запрос /recs ───► recsBulkhead (20) ──► ML-сервис

/users и /recs НЕ пересекаются.
```

### Проблема: размер пулов

**Как выбрать размер?**

- **CPU-bound:** GOMAXPROCS для каждого пула.
- **I/O-bound:** 10–100.
- **Медленные:** меньше (не занимать ресурсы).
- **Быстрые:** больше.

**Пример:**

- `/users` — 20 воркеров (быстро).
- `/orders` — 15 воркеров (средне).
- `/recommendations` — 5 воркеров (медленно, но ограниченно).

### Проблема: overflow

**Что если все слоты заняты?**

С текущей реализацией — **ждём** `ctx.Done()` или слот освободится.

**Но:** если клиент **уже ушёл**, ждать не нужно. Можно **reject** сразу.

**Решение:** см. 21.4.

### 💡 Практика: как писать простой bulkhead

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Отдельный семафор** для каждой группы.
2. **`ctx` для отмены.**
3. **`defer` освобождать слот.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики** на каждый bulkhead.
5. **Overflow handling** (см. 21.4).

**❌ НЕ ДЕЛАЙ:**

6. **Не используй один bulkhead для всего.**
7. **Не делай пулы слишком маленькими.**

---

## 21.3 Bulkhead с context

`context` — критичен для bulkhead. Без него запросы могут **зависнуть**.

### Проблема

```go
select {
case b.sem <- struct{}{}:
    // ...
}
```

**Что происходит:** если все слоты заняты и клиент ушёл — ждём впустую.

### Решение: context

Мы уже добавили `ctx` в `Execute`:

```go
func (b *Bulkhead) Execute(ctx context.Context, fn func() error) error {
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
    case <-ctx.Done():
        return ctx.Err()
    }
    return fn()
}
```

**Что происходит:** при отмене `ctx` — возвращаем ошибку. Слот не захватываем.

### Потребитель с таймаутом

```go
func handleUsers(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()
    
    err := usersBulkhead.Execute(ctx, func() error {
        return fetchUsers(ctx)
    })
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "timeout", http.StatusGatewayTimeout)
            return
        }
        http.Error(w, err.Error(), 503)
        return
    }
    w.WriteHeader(http.StatusOK)
}
```

**Что происходит:** через 5 секунд `ctx` отменяется. Запрос либо завершается, либо возвращает timeout.

### Полная реализация с ошибками

```go
type Bulkhead struct {
    sem     chan struct{}
    timeout time.Duration
}

func NewBulkhead(maxConcurrent int, timeout time.Duration) *Bulkhead {
    return &Bulkhead{
        sem:     make(chan struct{}, maxConcurrent),
        timeout: timeout,
    }
}

func (b *Bulkhead) Execute(ctx context.Context, fn func(ctx context.Context) error) error {
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
    case <-ctx.Done():
        return ctx.Err()
    }
    
    callCtx, cancel := context.WithTimeout(ctx, b.timeout)
    defer cancel()
    
    return fn(callCtx)
}
```

**Что даёт:**

- Захват слота с `ctx`.
- Таймаут на выполнение.
- `ctx` в `fn`.

### Схема

```
Execute(ctx, fn):
  select {
  case sem <- {}:           ← захват слота
    defer <-sem             ← освобождение
  case <-ctx.Done():        ← отмена при захвате
    return ctx.Err()
  }
  
  callCtx := WithTimeout(ctx, timeout)
  return fn(callCtx)        ← вызов с таймаутом
```

### 💡 Практика: как добавить context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx` — первый аргумент `Execute`.**
2. **`select` с `ctx.Done()`** при захвате.
3. **`fn(ctx)`** — передавай ctx.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Таймаут на выполнение.**

**❌ НЕ ДЕЛАЙ:**

5. **Не блокируйся без `ctx`.**
6. **Не забывай про таймаут.**

---

## 21.4 Overflow: что делать, когда всё занято

**Overflow** — ситуация, когда все слоты bulkhead заняты. Что делать?

### Три стратегии

**1. Блокироваться (blocking).**

```go
select {
case b.sem <- struct{}{}:
    // ...
case <-ctx.Done():
    return ctx.Err()
}
```

**Что делает:** ждём слот. Если `ctx` отменён — возвращаем ошибку.

**Плюсы:** не теряем запросы.

**Минусы:** клиент ждёт. Может зависнуть.

**2. Отклонять (reject).**

```go
select {
case b.sem <- struct{}{}:
    // ...
default:
    return ErrBulkheadFull
}
```

**Что делает:** если слот занят — **сразу** возвращаем ошибку.

**Плюсы:** клиент быстро получает ошибку.

**Минусы:** теряем запросы.

**3. Комбинация: таймаут + reject.**

```go
select {
case b.sem <- struct{}{}:
    // ...
case <-time.After(100 * time.Millisecond):
    return ErrBulkheadFull
case <-ctx.Done():
    return ctx.Err()
}
```

**Что делает:** ждём 100 мс, потом reject.

**Плюсы:** компромисс.

**Минусы:** сложнее.

### Когда что использовать

**Blocking:**

- **Критичные запросы.**
- **Нет альтернативы.**
- **Можно ждать.**

**Reject:**

- **Некритичные запросы.**
- **Есть fallback.**
- **Клиент может повторить.**

**Timeout + reject:**

- **Большинство случаев.**
- **Баланс между ожиданием и rejection.**

### Пример: reject

```go
var ErrBulkheadFull = errors.New("bulkhead is full")

type Bulkhead struct {
    sem chan struct{}
}

func (b *Bulkhead) Execute(ctx context.Context, fn func() error) error {
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
    case <-ctx.Done():
        return ctx.Err()
    default:
        return ErrBulkheadFull
    }
    return fn()
}
```

**Что происходит:** если слот занят — **сразу** возвращаем ошибку.

### Потребитель

```go
func handleRecommendations(w http.ResponseWriter, r *http.Request) {
    err := recsBulkhead.Execute(r.Context(), func() error {
        return fetchRecommendations(r.Context())
    })
    if err != nil {
        if errors.Is(err, ErrBulkheadFull) {
            // Fallback: показать кэшированные рекомендации
            showCachedRecommendations(w)
            return
        }
        http.Error(w, err.Error(), 503)
        return
    }
    w.WriteHeader(http.StatusOK)
}
```

**Что происходит:** при переполнении — fallback на кэш.

### Схема

```
Запрос:
  │
  ├── Слот свободен ──► выполнить
  │
  └── Слот занят
        ├── Blocking: ждём
        ├── Reject: сразу ошибка
        └── Timeout: ждём N мс, потом ошибка
```

### 💡 Практика: как обрабатывать overflow

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Reject** для некритичных.
2. **Blocking** для критичных.
3. **Timeout + reject** для баланса.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Fallback** при reject.
5. **Метрики** на reject.

**❌ НЕ ДЕЛАЙ:**

6. **Не блокируйся без таймаута.**
7. **Не теряй данные без fallback.**

---

## 21.5 Bulkhead для разных типов запросов

Разберём **разделение** ресурсов между разными запросами.

### Пример: API с эндпоинтами

```go
type API struct {
    usersBulkhead  *Bulkhead
    ordersBulkhead *Bulkhead
    recsBulkhead   *Bulkhead
}

func NewAPI() *API {
    return &API{
        usersBulkhead:  NewBulkhead(20, 2*time.Second),
        ordersBulkhead: NewBulkhead(15, 5*time.Second),
        recsBulkhead:   NewBulkhead(5, 10*time.Second),
    }
}

func (a *API) HandleUsers(w http.ResponseWriter, r *http.Request) {
    err := a.usersBulkhead.Execute(r.Context(), func(ctx context.Context) error {
        return fetchUsers(ctx)
    })
    // ...
}

func (a *API) HandleOrders(w http.ResponseWriter, r *http.Request) {
    err := a.ordersBulkhead.Execute(r.Context(), func(ctx context.Context) error {
        return fetchOrders(ctx)
    })
    // ...
}

func (a *API) HandleRecommendations(w http.ResponseWriter, r *http.Request) {
    err := a.recsBulkhead.Execute(r.Context(), func(ctx context.Context) error {
        return fetchRecommendations(ctx)
    })
    // ...
}
```

**Что даёт:**

- `/users` — 20 слотов, 2 сек timeout.
- `/orders` — 15 слотов, 5 сек.
- `/recommendations` — 5 слотов, 10 сек.

**Размеры подобраны** под нагрузку каждого эндпоинта.

### Пример: разные внешние сервисы

```go
type Services struct {
    dbBulkhead   *Bulkhead  // 20 слотов
    apiBulkhead  *Bulkhead  // 10 слотов
    mlBulkhead   *Bulkhead  // 3 слота
}

func (s *Services) FetchFromDB(ctx context.Context, id int) (User, error) {
    var user User
    err := s.dbBulkhead.Execute(ctx, func(ctx context.Context) error {
        var err error
        user, err = db.GetUser(ctx, id)
        return err
    })
    return user, err
}

func (s *Services) FetchFromAPI(ctx context.Context, id int) (Profile, error) {
    var profile Profile
    err := s.apiBulkhead.Execute(ctx, func(ctx context.Context) error {
        var err error
        profile, err = api.GetProfile(ctx, id)
        return err
    })
    return profile, err
}
```

**Что даёт:** каждый сервис имеет **свой** пул. Медленный ML **не занимает** слоты для БД.

### Размеры пулов

**Рекомендации:**

| Сервис | Слотов | Timeout |
|:---|:---|:---|
| БД (быстрая) | 20–50 | 1–5 сек |
| БД (аналитика) | 5–10 | 30–300 сек |
| HTTP API (внутренний) | 10–30 | 1–5 сек |
| HTTP API (внешний) | 5–20 | 5–30 сек |
| ML-сервис | 3–10 | 10–60 сек |
| Redis | 50–100 | 100 мс – 1 сек |

### Схема

```
БД: 20 слотов ────► много быстрых запросов
API: 10 слотов ───► средне
ML: 3 слота ──────► мало, медленно

ML занял все 3 слота. БД работает с 20.
```

### 💡 Практика: как разделять ресурсы

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Отдельный bulkhead** для каждого типа запроса.
2. **Размеры под нагрузку.**
3. **Timeout** для каждого.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики** на каждый bulkhead.
5. **Fallback** при переполнении.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй один пул для всего.**
7. **Не делай ML-пул большим.**

---

## 21.6 Bulkhead vs semaphore vs worker pool

Разберём **разницу** между тремя паттернами.

### Semaphore

**Semaphore** — ограничивает параллелизм **одной группы**.

```
Запросы ──► semaphore (20) ──► БД
```

**Все** запросы идут через **один** семафор.

### Worker pool

**Worker pool** — N воркеров обрабатывают **однотипные** задачи.

```
tasksCh ──► worker pool (20) ──► БД
```

**Все** задачи идут через **один** пул.

### Bulkhead

**Bulkhead** — несколько **независимых** пулов.

```
/users ──► bulkhead (10) ──► БД
/orders ──► bulkhead (15) ──► БД + API
/recs ───► bulkhead (5) ──► ML
```

Каждый тип запроса имеет **свой** пул.

### Сравнение

| Аспект | Semaphore | Worker pool | Bulkhead |
|:---|:---|:---|:---|
| Групп | 1 | 1 | N |
| Изоляция | Нет | Нет | Да |
| Компонент | Семафор | Воркеры + канал | N семафоров |
| Overflow | Ждёт | Ждёт | Можно reject |
| Use case | Один ресурс | Однотипные задачи | Разные типы |

### Когда что использовать

**Semaphore:**

- **Один ресурс.**
- **Однотипные запросы.**
- **Простота.**

**Worker pool:**

- **Много задач.**
- **Однотипные.**
- **Переиспользование воркеров.**

**Bulkhead:**

- **Разные типы запросов.**
- **Изоляция важна.**
- **Разные SLA.**

### Пример: semaphore + bulkhead

**Semaphore** для БД, **bulkhead** для разных эндпоинтов:

```go
type API struct {
    usersBulkhead  *Bulkhead
    ordersBulkhead *Bulkhead
    dbSemaphore    *Bulkhead  // общий для БД
}

func (a *API) HandleUsers(w http.ResponseWriter, r *http.Request) {
    err := a.usersBulkhead.Execute(r.Context(), func(ctx context.Context) error {
        return a.dbSemaphore.Execute(ctx, func(ctx context.Context) error {
            return fetchUsers(ctx)
        })
    })
    // ...
}
```

**Что даёт:** `/users` и `/orders` изолированы друг от друга, но **оба** ограничены общим семафором БД.

### 💡 Практика: как выбирать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Semaphore — для одного ресурса.**
2. **Worker pool — для однотипных задач.**
3. **Bulkhead — для разных типов.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Bulkhead + semaphore** — изоляция + общий ресурс.

**❌ НЕ ДЕЛАЙ:**

5. **Не путай bulkhead с semaphore.**
6. **Не используй bulkhead для однотипных.**

---

## 21.7 В связке с другими паттернами

Bulkhead редко используется **в одиночку**. Разберём связки.

### Bulkhead + circuit breaker

**Bulkhead** изолирует. **Circuit breaker** защищает от сбоев.

```go
type Service struct {
    bulkhead *Bulkhead
    breaker  *CircuitBreaker
}

func (s *Service) Fetch(ctx context.Context) error {
    return s.bulkhead.Execute(ctx, func(ctx context.Context) error {
        return s.breaker.Call(ctx, func(ctx context.Context) error {
            return callExternal(ctx)
        })
    })
}
```

**Порядок:**

1. **Bulkhead** — захватить слот.
2. **Circuit breaker** — проверить состояние.
3. **Внешний вызов**.

### Bulkhead + rate limiter

**Bulkhead** изолирует. **Rate limiter** ограничивает скорость.

```go
func (s *Service) Fetch(ctx context.Context) error {
    return s.bulkhead.Execute(ctx, func(ctx context.Context) error {
        if err := s.limiter.Wait(ctx); err != nil {
            return err
        }
        return callExternal(ctx)
    })
}
```

### Bulkhead + retry

**Bulkhead** изолирует. **Retry** повторяет.

```go
func (s *Service) Fetch(ctx context.Context) error {
    return s.bulkhead.Execute(ctx, func(ctx context.Context) error {
        return retryCtx(ctx, 3, 100*time.Millisecond, func(ctx context.Context) error {
            return callExternal(ctx)
        })
    })
}
```

### Bulkhead + timeout

**Bulkhead** изолирует. **Timeout** ограничивает время.

```go
func (s *Service) Fetch(ctx context.Context) error {
    return s.bulkhead.Execute(ctx, func(ctx context.Context) error {
        callCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
        defer cancel()
        return callExternal(callCtx)
    })
}
```

### Полная защита

```
Запрос
   │
   ▼
┌──────────────┐
│   Bulkhead   │  ← изоляция
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Timeout    │  ← 5 сек
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Rate limiter│  ← не более RPS
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

### 💡 Практика: как комбинировать bulkhead

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Bulkhead + circuit breaker** — изоляция + защита.
2. **Bulkhead + timeout** — изоляция + ограничение времени.
3. **Bulkhead + rate limiter** — изоляция + скорость.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Bulkhead + retry** — изоляция + повторные попытки.

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай порядок.**
6. **Не смешивай слишком много.**

---

## 21.8 Практика Go: bulkhead с метриками

Разберём **bulkhead с метриками**.

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

var ErrBulkheadFull = errors.New("bulkhead is full")

type Metrics struct {
    Accepted atomic.Int64
    Rejected atomic.Int64
    Canceled atomic.Int64
    Active   atomic.Int64
}

type Bulkhead struct {
    name    string
    sem     chan struct{}
    timeout time.Duration
    metrics *Metrics
}

func NewBulkhead(name string, maxConcurrent int, timeout time.Duration) *Bulkhead {
    return &Bulkhead{
        name:    name,
        sem:     make(chan struct{}, maxConcurrent),
        timeout: timeout,
        metrics: &Metrics{},
    }
}

func (b *Bulkhead) Execute(ctx context.Context, fn func(ctx context.Context) error) error {
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
    case <-ctx.Done():
        b.metrics.Canceled.Add(1)
        return ctx.Err()
    default:
        b.metrics.Rejected.Add(1)
        return ErrBulkheadFull
    }
    
    b.metrics.Accepted.Add(1)
    b.metrics.Active.Add(1)
    defer b.metrics.Active.Add(-1)
    
    callCtx, cancel := context.WithTimeout(ctx, b.timeout)
    defer cancel()
    
    return fn(callCtx)
}

func (b *Bulkhead) Metrics() *Metrics {
    return b.metrics
}

func (b *Bulkhead) Name() string {
    return b.name
}

func main() {
    // Три bulkhead'а
    users := NewBulkhead("users", 5, 2*time.Second)
    orders := NewBulkhead("orders", 3, 3*time.Second)
    recs := NewBulkhead("recs", 2, 5*time.Second)
    
    ctx := context.Background()
    
    // Симулируем нагрузку
    for i := 0; i < 10; i++ {
        go func(id int) {
            users.Execute(ctx, func(ctx context.Context) error {
                time.Sleep(500 * time.Millisecond)
                return nil
            })
        }(i)
        
        go func(id int) {
            orders.Execute(ctx, func(ctx context.Context) error {
                time.Sleep(1 * time.Second)
                return nil
            })
        }(i)
        
        go func(id int) {
            recs.Execute(ctx, func(ctx context.Context) error {
                time.Sleep(2 * time.Second)
                return nil
            })
        }(i)
    }
    
    time.Sleep(3 * time.Second)
    
    printMetrics(users)
    printMetrics(orders)
    printMetrics(recs)
}

func printMetrics(b *Bulkhead) {
    m := b.Metrics()
    fmt.Printf("\n=== %s ===\n", b.Name())
    fmt.Printf("Accepted: %d\n", m.Accepted.Load())
    fmt.Printf("Rejected: %d\n", m.Rejected.Load())
    fmt.Printf("Canceled: %d\n", m.Canceled.Load())
    fmt.Printf("Active:   %d\n", m.Active.Load())
}
```

**Пример вывода:**

```
=== users ===
Accepted: 5
Rejected: 5
Canceled: 0
Active:   0

=== orders ===
Accepted: 3
Rejected: 7
Canceled: 0
Active:   0

=== recs ===
Accepted: 2
Rejected: 8
Canceled: 0
Active:   0
```

**Что видно:**

- `users` (5 слотов) — 5 accepted, 5 rejected.
- `orders` (3 слота) — 3 accepted, 7 rejected.
- `recs` (2 слота) — 2 accepted, 8 rejected.

**Bulkhead работает:** каждый пул изолирован, overflow — rejected.

### 💡 Практика: как измерять bulkhead

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики:** accepted, rejected, canceled, active.
2. **`atomic.Int64`** для счётчиков.
3. **Экспорт в Prometheus.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Алерт при высоком reject rate.**
5. **Логирование reject.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `Mutex` для метрик.**
7. **Не игнорируй reject.**

---

## 21.9 Выводы и типичные ошибки

**Что мы узнали?**

Bulkhead — паттерн изоляции ресурсов. Разделяем **пулы** между разными типами запросов. **Простейший** — семафор на группу. **`ctx`** для отмены. **Overflow:** blocking, reject, timeout + reject. **Размеры пулов** под нагрузку каждого типа. Bulkhead ≠ semaphore: изоляция vs один пул. Bulkhead + circuit breaker / rate limiter / retry / timeout для полной защиты.

**Типичные ошибки:**

- ❌ **Один пул для всего.** Медленный сервис кладёт всё.
- ❌ **Маленькие пулы.** Много reject'ов.
- ❌ **Большие пулы.** Занимают ресурсы.
- ❌ **Не использовать `ctx`.** Зависание.
- ❌ **Blocking без таймаута.** Ждём вечно.
- ❌ **Reject без fallback.** Теряем данные.
- ❌ **Не различать разные типы.**
- ❌ **Не мониторить reject.**
- ❌ **Bulkhead для однотипных запросов.**
- ❌ **Не комбинировать с другими паттернами.**

---

## 21.10 Для быстрого повторения

- **Bulkhead** — изоляция ресурсов между группами.
- **Аналогия:** переборки на корабле.
- **Простейший** — семафор на группу.
- **Разные пулы** для разных типов запросов.
- **Размеры под нагрузку.**
- **`ctx`** для отмены.
- **Overflow:** blocking, reject, timeout + reject.
- **Bulkhead ≠ semaphore:** изоляция vs один пул.
- **Bulkhead + circuit breaker** — изоляция + защита.
- **Bulkhead + rate limiter** — изоляция + скорость.
- **Bulkhead + retry** — изоляция + повторные попытки.
- **Bulkhead + timeout** — изоляция + ограничение времени.
- **Метрики:** accepted, rejected, canceled, active.

---

## 21.11 Вопросы для самопроверки

1. Что такое bulkhead? Какую задачу решает?
2. Чем bulkhead отличается от semaphore?
3. Как построить простейший bulkhead?
4. Зачем `ctx` в bulkhead?
5. Что такое overflow? Три стратегии?
6. Как разделить ресурсы между разными типами запросов?
7. Как комбинировать bulkhead с circuit breaker?
8. Что выбрать — bulkhead или semaphore?

---

## 21.12 Ответы

### Ответ 1

**Bulkhead** — паттерн изоляции ресурсов между разными частями системы. **Аналогия:** переборки на корабле.

**Решает задачу:** медленный сервис **не занимает** ресурсы, нужные для быстрых.

**Пример:** медленный `/recommendations` не должен блокировать `/users`.

### Ответ 2

**Semaphore** — один пул для **всех** запросов.

**Bulkhead** — **несколько** пулов для **разных** типов.

**Semaphore** ограничивает параллелизм. **Bulkhead** изолирует.

### Ответ 3

```go
type Bulkhead struct {
    sem chan struct{}
}

func NewBulkhead(maxConcurrent int) *Bulkhead {
    return &Bulkhead{sem: make(chan struct{}, maxConcurrent)}
}

func (b *Bulkhead) Execute(ctx context.Context, fn func() error) error {
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
    case <-ctx.Done():
        return ctx.Err()
    }
    return fn()
}
```

### Ответ 4

**`ctx`** позволяет не блокироваться, если все слоты заняты. Без него запрос **зависнет** или будет ждать вечно.

**Решение:** `select` с `ctx.Done()`.

### Ответ 5

**Overflow** — все слоты заняты.

**Три стратегии:**
1. **Blocking** — ждём слот.
2. **Reject** — сразу ошибка.
3. **Timeout + reject** — ждём N мс, потом ошибка.

**Blocking** для критичных. **Reject** для некритичных. **Timeout** — баланс.

### Ответ 6

**Разделение:**

```go
var (
    usersBulkhead = NewBulkhead(20, 2*time.Second)
    ordersBulkhead = NewBulkhead(15, 5*time.Second)
    recsBulkhead = NewBulkhead(5, 10*time.Second)
)
```

Каждый эндпоинт — **свой** bulkhead. Размеры под нагрузку.

### Ответ 7

**Bulkhead + circuit breaker:**

```go
func (s *Service) Fetch(ctx context.Context) error {
    return s.bulkhead.Execute(ctx, func(ctx context.Context) error {
        return s.breaker.Call(ctx, func(ctx context.Context) error {
            return callExternal(ctx)
        })
    })
}
```

Bulkhead — снаружи (захват слота). Circuit breaker — внутри (проверка состояния).

### Ответ 8

**Semaphore:**

- Один ресурс.
- Однотипные запросы.

**Bulkhead:**

- Разные типы запросов.
- Изоляция важна.
- Разные SLA.

**Bulkhead** для сложных систем, **semaphore** для простых.

---

## 21.13 Куда идти дальше?

Мы разобрали bulkhead — изоляцию ресурсов. Теперь мы умеем не дать одному сервису положить всё.

Но остаются **продвинутые паттерны**: leader election, sharded locks, lock-free структуры.

- **Как построить leader election?** → **Глава 22: Leader election.**
- **Как разбить лок на шарды?** → **Глава 23: Sharded locks.**
- **Как построить lock-free структуры?** → **Глава 24: Lock-free структуры.**

---

## 21.14 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Bulkhead** | Изоляция ресурсов | Несколько пулов |
| **Аналогия** | Переборки на корабле | Один затопило — остальные сухие |
| **Простейший** | Семафор на группу | `make(chan struct{}, N)` |
| **Разные пулы** | Для разных типов | `/users`, `/orders`, `/recs` |
| **Размеры** | Под нагрузку | Быстрые — больше, медленные — меньше |
| **`ctx`** | Отмена | `select` с `ctx.Done()` |
| **Overflow** | Все занято | Blocking, reject, timeout+reject |
| **Reject** | Сразу ошибка | `default` в `select` |
| **Fallback** | При reject | Кэш, дефолт |
| **Bulkhead ≠ semaphore** | Изоляция vs один пул | — |
| **+ circuit breaker** | Изоляция + защита | Bulkhead снаружи |
| **+ rate limiter** | Изоляция + скорость | Внутри bulkhead |
| **+ retry** | Изоляция + повтор | Внутри bulkhead |
| **+ timeout** | Изоляция + ограничение | Внутри bulkhead |
| **Метрики** | accepted, rejected, canceled, active | `atomic.Int64` |

🧱 **Ключевая идея:** Bulkhead — паттерн изоляции ресурсов между разными частями системы. **Аналогия** — переборки на корабле: одно затопило, остальные сухие. **Простейший** — семафор на группу. **Разные пулы** для разных типов запросов (быстрые/медленные, критичные/некритичные). **Размеры под нагрузку**. **Overflow:** blocking, reject, timeout + reject. **Bulkhead ≠ semaphore:** изоляция vs один пул. Комбинируется с circuit breaker (снаружи), rate limiter / retry / timeout (внутри). Метрики: accepted, rejected, canceled, active.