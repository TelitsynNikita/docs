# 🔮 Глава 20: Future/Promise — результат в будущем

**Что вы узнаете:**
- Что такое future и какую задачу он решает.
- Чем future отличается от обычной горутины с каналом.
- Как построить простейший future.
- Как добавить отмену через `context`.
- Как обрабатывать ошибки.
- Как комбинировать несколько future'ов.
- Как избежать утечек и deadlock.

**После прочтения вы сможете:**
- Построить future с нуля.
- Запускать операции в фоне.
- Получать результат позже.
- Обрабатывать ошибки из future.
- Комбинировать future'ы.
- Понимать, где future уместен, а где — нет.

---

## Содержание

- [20.0 Пролог: параллельные запросы](#200-пролог-параллельные-запросы)
- [20.1 Что такое future](#201-что-такое-future)
- [20.2 Простейший future](#202-простейший-future)
- [20.3 Future с context](#203-future-с-context)
- [20.4 Future с ошибками](#204-future-с-ошибками)
- [20.5 Комбинирование future'ов](#205-комбинирование-futureов)
- [20.6 Future vs горутина + канал](#206-future-vs-горутина--канал)
- [20.7 В связке с другими паттернами](#207-в-связке-с-другими-паттернами)
- [20.8 Практика Go: future с метриками](#208-практика-go-future-с-метриками)
- [20.9 Выводы и типичные ошибки](#209-выводы-и-типичные-ошибки)
- [20.10 Для быстрого повторения](#2010-для-быстрого-повторения)
- [20.11 Вопросы для самопроверки](#2011-вопросы-для-самопроверки)
- [20.12 Ответы](#2012-ответы)
- [20.13 Куда идти дальше?](#2013-куда-идти-дальше)
- [20.14 Чек-лист](#2014-чек-лист)

---

## 20.0 Пролог: параллельные запросы

У нас есть HTTP-хендлер, который формирует страницу пользователя. Для этого нужно:
- Прочитать **профиль** из БД.
- Прочитать **историю заказов** из БД.
- Получить **рекомендации** из внешнего сервиса.

Пишем последовательно:

```go
func handler(w http.ResponseWriter, r *http.Request) {
    profile := fetchProfile(r.Context(), userID)          // 50 мс
    orders := fetchOrders(r.Context(), userID)            // 100 мс
    recommendations := fetchRecommendations(r.Context())  // 200 мс
    // ...
}
```

Всё работает. Но 350 мс на запрос — долго. Три запроса **независимы** — их можно делать **параллельно**.

Хочется: запустить все три **одновременно**, подождать **все**, собрать результаты.

Можно через `WaitGroup` и общий слайс:

```go
var wg sync.WaitGroup
var profile Profile
var orders []Order
var recommendations []Recommendation

wg.Add(3)
go func() { defer wg.Done(); profile = fetchProfile(ctx, userID) }()
go func() { defer wg.Done(); orders = fetchOrders(ctx, userID) }()
go func() { defer wg.Done(); recommendations = fetchRecommendations(ctx) }()
wg.Wait()
```

Работает, но **неудобно**: три переменные, три горутины, `WaitGroup`. А если один запрос занимает 1 секунду, а другие 50 мс — ждём всё равно секунду.

Хочется **абстракцию**: «запусти в фоне, дай мне ручку, я потом получу результат». Эта абстракция — **future**.

> **Мост к следующим главам:** future — обобщение горутины + канал. Он часто используется для параллельных запросов, кэширования и pipeline. Понимание future даёт понимание, **как строить асинхронные API**.

---

## 20.1 Что такое future

**Future** — объект, который представляет **результат**, который **ещё не готов**.

### Идея

- **Запускаем** операцию в фоне.
- Получаем **future** — ручку для результата.
- Позже **получаем** результат через future.

```
Запуск:
  future := NewFuture(func() (T, error) { ... })

Позже:
  result, err := future.Get(ctx)
```

### Когда использовать future

**1. Параллельные запросы.**

- Профиль + заказы + рекомендации.
- Запуск одновременно, получение позже.

**2. Асинхронный API.**

- Функция возвращает future.
- Пользователь решает, когда ждать.

**3. Кэширование.**

- Future в кэше — избегает дублирования запросов.

**4. Композиция.**

- Несколько future'ов → один.

### Когда НЕ использовать future

**1. Синхронный код.**

Если результат нужен сразу — future не нужен.

**2. Один вызов.**

Для одного запроса — горутина + канал проще.

**3. Сложные зависимости.**

Если результат нужен в разных местах с разной логикой — может быть сложнее.

### Ключевые свойства

**1. Однократное выполнение.**

Future запускает операцию **один раз**.

**2. Результат доступен после завершения.**

`Get` блокируется, пока результат не готов.

**3. Результат можно получить несколько раз.**

Если future хранит результат — можно получать повторно.

### 💡 Практика: как думать о future

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Future — для параллельных запросов.**
2. **Future — для асинхронного API.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Future с `context` для отмены.**
4. **Future в кэше** — избегает дублирования.

**❌ НЕ ДЕЛАЙ:**

5. **Не используй future для синхронного кода.**
6. **Не забывай про утечки.**

---

## 20.2 Простейший future

Начнём с самого простого — future на канале.

### Идея

- Future содержит канал для результата.
- Горутина пишет результат в канал.
- `Get` читает из канала.

### Реализация

```go
type Future[T any] struct {
    result chan T
    err    chan error
}

func NewFuture[T any](fn func() (T, error)) *Future[T] {
    f := &Future[T]{
        result: make(chan T, 1),
        err:    make(chan error, 1),
    }
    
    go func() {
        defer close(f.result)
        defer close(f.err)
        
        r, e := fn()
        if e != nil {
            f.err <- e
            return
        }
        f.result <- r
    }()
    
    return f
}

func (f *Future[T]) Get(ctx context.Context) (T, error) {
    var zero T
    
    select {
    case <-ctx.Done():
        return zero, ctx.Err()
    case err := <-f.err:
        return zero, err
    case r := <-f.result:
        return r, nil
    }
}
```

**Что происходит:**

1. `NewFuture` создаёт два канала — для результата и ошибки.
2. Запускает горутину, которая вызывает `fn`.
3. Результат или ошибка пишется в соответствующий канал.
4. `Get` ждёт результат, ошибку или отмену `ctx`.

### Потребитель

```go
func main() {
    future := NewFuture(func() (string, error) {
        time.Sleep(1 * time.Second)
        return "result", nil
    })
    
    // Делаем что-то другое
    fmt.Println("doing other work...")
    time.Sleep(500 * time.Millisecond)
    
    // Получаем результат
    result, err := future.Get(context.Background())
    fmt.Printf("result: %s, err: %v\n", result, err)
}
```

**Пример вывода:**

```
doing other work...
result: result, err: <nil>
```

**Что видно:** future запускается сразу. Пока мы делаем другую работу — операция выполняется. Через 500 мс получаем результат (ещё через 500 мс).

### Буферизованные каналы

**Почему `make(chan T, 1)`?**

- Буфер на 1 позволяет горутине **записать результат** и **завершиться**, даже если никто не читал.
- Без буфера — горутина зависнет на `f.result <- r`, если `Get` не вызывается.

### Полный код

```go
package main

import (
    "context"
    "fmt"
    "time"
)

type Future[T any] struct {
    result chan T
    err    chan error
}

func NewFuture[T any](fn func() (T, error)) *Future[T] {
    f := &Future[T]{
        result: make(chan T, 1),
        err:    make(chan error, 1),
    }
    
    go func() {
        defer close(f.result)
        defer close(f.err)
        
        r, e := fn()
        if e != nil {
            f.err <- e
            return
        }
        f.result <- r
    }()
    
    return f
}

func (f *Future[T]) Get(ctx context.Context) (T, error) {
    var zero T
    
    select {
    case <-ctx.Done():
        return zero, ctx.Err()
    case err := <-f.err:
        return zero, err
    case r := <-f.result:
        return r, nil
    }
}

func main() {
    future := NewFuture(func() (string, error) {
        time.Sleep(1 * time.Second)
        return "result", nil
    })
    
    fmt.Println("doing other work...")
    time.Sleep(500 * time.Millisecond)
    
    result, err := future.Get(context.Background())
    fmt.Printf("result: %s, err: %v\n", result, err)
}
```

### Схема

```
NewFuture(fn):
  ├── result := make(chan T, 1)
  ├── err := make(chan error, 1)
  └── go fn()
        ├── r, e := fn()
        ├── if e != nil: err <- e
        └── else: result <- r

Get(ctx):
  select {
  case <-ctx.Done():  ← отмена
  case err := <-err:  ← ошибка
  case r := <-result: ← результат
  }
```

### Проблема: `Get` можно вызвать только один раз

**Что если нужен результат в нескольких местах?**

```go
r1, _ := future.Get(ctx)  // OK
r2, _ := future.Get(ctx)  // ← зависит от реализации
```

**С текущей реализацией:** второй `Get` вернёт `zero`, потому что канал **уже пуст** (значение прочитано).

### Решение: сохранить результат

```go
type Future[T any] struct {
    result chan T
    err    chan error
    done   chan struct{}
    
    once   sync.Once
    value  T
    error  error
}

func (f *Future[T]) Get(ctx context.Context) (T, error) {
    select {
    case <-ctx.Done():
        var zero T
        return zero, ctx.Err()
    case <-f.done:
        return f.value, f.error
    }
}
```

**Что даёт:** результат сохраняется. `Get` можно вызывать многократно.

### 💡 Практика: как писать простой future

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Буферизованные каналы** (буфер 1).
2. **`defer close`** в горутине.
3. **`ctx` в `Get`.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Сохранять результат** — если нужен многократный `Get`.

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай буфер.**
6. **Не блокируйся без `ctx`.**

---

## 20.3 Future с context

`context` — критичен для future. Без него future может **зависнуть**.

### Проблема

```go
future := NewFuture(func() (string, error) {
    time.Sleep(1 * time.Hour)
    return "result", nil
})

result, err := future.Get(context.Background())
// Ждём час
```

**Что происходит:** future нельзя **отменить**. Даже если клиент ушёл — операция продолжается.

### Решение: context в fn

```go
future := NewFuture(func(ctx context.Context) (string, error) {
    select {
    case <-ctx.Done():
        return "", ctx.Err()
    case <-time.After(1 * time.Hour):
        return "result", nil
    }
})

// В Get — передаём ctx
result, err := future.Get(ctx)
```

**Что происходит:** future **видит** `ctx.Done()` и завершается.

### Полная реализация

```go
type Future[T any] struct {
    fn     func(ctx context.Context) (T, error)
    result chan T
    err    chan error
    ctx    context.Context
    cancel context.CancelFunc
}

func NewFuture[T any](parentCtx context.Context, fn func(ctx context.Context) (T, error)) *Future[T] {
    ctx, cancel := context.WithCancel(parentCtx)
    
    f := &Future[T]{
        fn:     fn,
        result: make(chan T, 1),
        err:    make(chan error, 1),
        ctx:    ctx,
        cancel: cancel,
    }
    
    go func() {
        defer close(f.result)
        defer close(f.err)
        
        r, e := fn(ctx)
        if e != nil {
            f.err <- e
            return
        }
        f.result <- r
    }()
    
    return f
}

func (f *Future[T]) Get(ctx context.Context) (T, error) {
    var zero T
    
    select {
    case <-ctx.Done():
        f.cancel()  // отменяем операцию
        return zero, ctx.Err()
    case err := <-f.err:
        return zero, err
    case r := <-f.result:
        return r, nil
    }
}

func (f *Future[T]) Cancel() {
    f.cancel()
}
```

**Что происходит:**

- `NewFuture` создаёт дочерний `ctx` из `parentCtx`.
- В `fn` передаётся этот `ctx`.
- `Get` при отмене вызывает `f.cancel()`.
- Есть отдельный `Cancel()`.

### Потребитель

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
    defer cancel()
    
    future := NewFuture(ctx, func(ctx context.Context) (string, error) {
        select {
        case <-ctx.Done():
            return "", ctx.Err()
        case <-time.After(2 * time.Second):
            return "result", nil
        }
    })
    
    result, err := future.Get(ctx)
    fmt.Printf("result: %q, err: %v\n", result, err)
}
```

**Пример вывода:**

```
result: "", err: context deadline exceeded
```

**Что происходит:** через 500 мс `ctx` отменяется. Future возвращает `DeadlineExceeded`.

### Схема

```
NewFuture(parentCtx, fn):
  ├── ctx, cancel := WithCancel(parentCtx)
  ├── result := make(chan T, 1)
  ├── err := make(chan error, 1)
  └── go fn(ctx) → result или err

Get(ctx):
  select {
  case <-ctx.Done():   ← отмена
    f.cancel()
    return ctx.Err()
  case err := <-err:
  case r := <-result:
  }
```

### 💡 Практика: как добавить context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`parentCtx` — первый аргумент `NewFuture`.**
2. **`fn(ctx)` — передавай ctx в callback.**
3. **`f.cancel()` при отмене `Get`.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метод `Cancel()`** для явной отмены.

**❌ НЕ ДЕЛАЙ:**

5. **Не блокируйся без `ctx`.**
6. **Не забывай `f.cancel()`.**

---

## 20.4 Future с ошибками

Future может завершиться **ошибкой**. Разберём, как её обрабатывать.

### Простой способ: результат + ошибка

Мы уже это делаем: `result` и `err` каналы.

```go
select {
case <-ctx.Done():
    return zero, ctx.Err()
case err := <-f.err:
    return zero, err
case r := <-f.result:
    return r, nil
}
```

### Проблема: читаем из обоих каналов

**Что если `fn` вернул ошибку, но мы читаем из `result`?**

```go
r := <-f.result  // ← получим zero
```

**Решение:** `Get` читает из обоих каналов через `select`.

### Паника в fn

**Что если `fn` паникует?**

```go
func NewFuture[T any](parentCtx context.Context, fn func(ctx context.Context) (T, error)) *Future[T] {
    // ...
    go func() {
        defer close(f.result)
        defer close(f.err)
        
        defer func() {
            if r := recover(); r != nil {
                f.err <- fmt.Errorf("panic: %v", r)
            }
        }()
        
        r, e := fn(ctx)
        // ...
    }()
    // ...
}
```

**Что даёт:** паника превращается в ошибку.

### Ошибки с типом

```go
type Result[T any] struct {
    Value T
    Err   error
}

func (r Result[T]) Ok() bool {
    return r.Err == nil
}
```

**Что даёт:** один канал для результата и ошибки.

### Полный пример

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "time"
)

type Future[T any] struct {
    result chan T
    err    chan error
    ctx    context.Context
    cancel context.CancelFunc
}

func NewFuture[T any](parentCtx context.Context, fn func(ctx context.Context) (T, error)) *Future[T] {
    ctx, cancel := context.WithCancel(parentCtx)
    
    f := &Future[T]{
        result: make(chan T, 1),
        err:    make(chan error, 1),
        ctx:    ctx,
        cancel: cancel,
    }
    
    go func() {
        defer close(f.result)
        defer close(f.err)
        
        defer func() {
            if r := recover(); r != nil {
                f.err <- fmt.Errorf("panic: %v", r)
            }
        }()
        
        r, e := fn(ctx)
        if e != nil {
            f.err <- e
            return
        }
        f.result <- r
    }()
    
    return f
}

func (f *Future[T]) Get(ctx context.Context) (T, error) {
    var zero T
    
    select {
    case <-ctx.Done():
        f.cancel()
        return zero, ctx.Err()
    case err := <-f.err:
        return zero, err
    case r := <-f.result:
        return r, nil
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Успешный future
    f1 := NewFuture(ctx, func(ctx context.Context) (string, error) {
        return "hello", nil
    })
    r, err := f1.Get(ctx)
    fmt.Printf("f1: %q, %v\n", r, err)
    
    // Future с ошибкой
    f2 := NewFuture(ctx, func(ctx context.Context) (string, error) {
        return "", errors.New("something failed")
    })
    r, err = f2.Get(ctx)
    fmt.Printf("f2: %q, %v\n", r, err)
    
    // Future с паникой
    f3 := NewFuture(ctx, func(ctx context.Context) (string, error) {
        panic("boom")
    })
    r, err = f3.Get(ctx)
    fmt.Printf("f3: %q, %v\n", r, err)
}
```

**Пример вывода:**

```
f1: "hello", <nil>
f2: "", something failed
f3: "", panic: boom
```

### 💡 Практика: как обрабатывать ошибки

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Два канала** — для результата и ошибки.
2. **`recover`** в горутине.
3. **`select`** в `Get`.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Тип `Result[T]`** для сложных случаев.

**❌ НЕ ДЕЛАЙ:**

5. **Не игнорируй паники.**
6. **Не путай ошибки с результатом.**

---

## 20.5 Комбинирование future'ов

Иногда нужно **несколько** future'ов объединить.

### WaitAll: все результаты

```go
func WaitAll[T any](ctx context.Context, futures []*Future[T]) ([]T, error) {
    results := make([]T, len(futures))
    
    var wg sync.WaitGroup
    var mu sync.Mutex
    var firstErr error
    
    for i, f := range futures {
        i, f := i, f
        wg.Add(1)
        go func() {
            defer wg.Done()
            r, err := f.Get(ctx)
            if err != nil {
                mu.Lock()
                if firstErr == nil {
                    firstErr = err
                }
                mu.Unlock()
                return
            }
            mu.Lock()
            results[i] = r
            mu.Unlock()
        }()
    }
    
    wg.Wait()
    return results, firstErr
}
```

**Что даёт:** ждём **все** future'ы. Возвращаем все результаты или первую ошибку.

### WaitAny: первый результат

```go
func WaitAny[T any](ctx context.Context, futures []*Future[T]) (T, error) {
    type result struct {
        value T
        err   error
    }
    
    ch := make(chan result, len(futures))
    
    for _, f := range futures {
        f := f
        go func() {
            r, err := f.Get(ctx)
            ch <- result{value: r, err: err}
        }()
    }
    
    select {
    case r := <-ch:
        return r.value, r.err
    case <-ctx.Done():
        var zero T
        return zero, ctx.Err()
    }
}
```

**Что даёт:** ждём **первый** завершившийся future.

### Пример: параллельные запросы

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Запускаем 3 запроса параллельно
    f1 := NewFuture(ctx, func(ctx context.Context) (string, error) {
        time.Sleep(100 * time.Millisecond)
        return "profile", nil
    })
    
    f2 := NewFuture(ctx, func(ctx context.Context) (string, error) {
        time.Sleep(200 * time.Millisecond)
        return "orders", nil
    })
    
    f3 := NewFuture(ctx, func(ctx context.Context) (string, error) {
        time.Sleep(150 * time.Millisecond)
        return "recommendations", nil
    })
    
    start := time.Now()
    results, err := WaitAll(ctx, []*Future[string]{f1, f2, f3})
    fmt.Printf("WaitAll: %v, err: %v, elapsed: %v\n", results, err, time.Since(start))
    
    // Или первый
    f4 := NewFuture(ctx, func(ctx context.Context) (string, error) {
        time.Sleep(50 * time.Millisecond)
        return "fast", nil
    })
    f5 := NewFuture(ctx, func(ctx context.Context) (string, error) {
        time.Sleep(200 * time.Millisecond)
        return "slow", nil
    })
    
    start = time.Now()
    result, err := WaitAny(ctx, []*Future[string]{f4, f5})
    fmt.Printf("WaitAny: %q, err: %v, elapsed: %v\n", result, err, time.Since(start))
}
```

**Пример вывода:**

```
WaitAll: [profile orders recommendations], err: <nil>, elapsed: 200ms
WaitAny: "fast", err: <nil>, elapsed: 50ms
```

**Что видно:**

- `WaitAll` ждёт 200 мс (самый долгий).
- `WaitAny` возвращает через 50 мс (первый).

### 💡 Практика: как комбинировать future'ы

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`WaitAll` — для всех результатов.**
2. **`WaitAny` — для первого.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **`ctx` — для отмены всех.**
4. **`sync.WaitGroup` + `Mutex` для `WaitAll`.**

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай про гонки.**
6. **Не блокируйся без `ctx`.**

---

## 20.6 Future vs горутина + канал

Разберём **разницу** между future и горутина + канал.

### Горутина + канал

```go
resultCh := make(chan string, 1)
go func() {
    r, err := fetch()
    if err != nil {
        resultCh <- ""
        return
    }
    resultCh <- r
}()

// Позже
result := <-resultCh
```

**Что даёт:** запуск в фоне, получение через канал.

### Future

```go
future := NewFuture(ctx, func(ctx context.Context) (string, error) {
    return fetch(ctx)
})

// Позже
result, err := future.Get(ctx)
```

**Что даёт:** абстракция. Инкапсулирует каналы, ошибки, отмену.

### Сравнение

| Аспект | Горутина + канал | Future |
|:---|:---|:---|
| Каналы | Ручные | Скрытые |
| Ошибки | Ручные | Встроенные |
| Отмена | Через `ctx` вручную | Через `Get` или `Cancel` |
| Композиция | Вручную | `WaitAll` / `WaitAny` |
| Многоразовый `Get` | Нет | Да (если сохранить результат) |

### Что выбрать

**Горутина + канал:**

- **Один вызов.**
- **Простая логика.**
- **Не нужна абстракция.**

**Future:**

- **Много вызовов.**
- **Композиция.**
- **Асинхронный API.**

### 💡 Практика: как выбирать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Горутина + канал — для простого.**
2. **Future — для сложного.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Future в кэше** — избегает дублирования.

**❌ НЕ ДЕЛАЙ:**

4. **Не используй future для одного вызова.**

---

## 20.7 В связке с другими паттернами

Future редко используется **в одиночку**. Разберём связки.

### Future + worker pool

**Future для параллельных запросов:**

```go
func fetchAll(ctx context.Context, ids []int) ([]User, error) {
    futures := make([]*Future[User], len(ids))
    for i, id := range ids {
        id := id
        futures[i] = NewFuture(ctx, func(ctx context.Context) (User, error) {
            return fetchUser(ctx, id)
        })
    }
    return WaitAll(ctx, futures)
}
```

### Future + pipeline

**Future как стадия pipeline:**

```go
func enrichStage(ctx context.Context, input <-chan int) <-chan Enriched {
    out := make(chan Enriched)
    
    go func() {
        defer close(out)
        for v := range input {
            future := NewFuture(ctx, func(ctx context.Context) (Enriched, error) {
                return enrich(ctx, v)
            })
            result, err := future.Get(ctx)
            if err != nil {
                continue
            }
            select {
            case out <- result:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    return out
}
```

### Future + cache

**Future в кэше — избегает дублирования:**

```go
type Cache struct {
    mu      sync.Mutex
    futures map[string]*Future[User]
}

func (c *Cache) Get(ctx context.Context, id string) (User, error) {
    c.mu.Lock()
    f, ok := c.futures[id]
    if !ok {
        f = NewFuture(ctx, func(ctx context.Context) (User, error) {
            return fetchUser(ctx, id)
        })
        c.futures[id] = f
    }
    c.mu.Unlock()
    
    return f.Get(ctx)
}
```

**Что даёт:** если два клиента запрашивают одного пользователя **одновременно** — один запрос.

### Схема

```
Первый запрос:
  c.futures["42"] = NewFuture(...)
  → f.Get(ctx)

Второй запрос (одновременно):
  c.futures["42"] уже есть
  → f.Get(ctx) ← тот же future

Результат: один запрос к БД, два клиента получают результат.
```

### 💡 Практика: как комбинировать future

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Future + worker pool** — параллельные запросы.
2. **Future + pipeline** — стадия pipeline.
3. **Future + cache** — избегает дублирования.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`WaitAll` / `WaitAny`** для композиции.

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай `ctx`.**

---

## 20.8 Практика Go: future с метриками

Разберём **future с метриками**.

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

type Metrics struct {
    Started    atomic.Int64
    Completed  atomic.Int64
    Failed     atomic.Int64
    Canceled   atomic.Int64
    TotalTime  atomic.Int64
}

type Future[T any] struct {
    result chan T
    err    chan error
    ctx    context.Context
    cancel context.CancelFunc
    start  time.Time
    metrics *Metrics
}

func NewFuture[T any](parentCtx context.Context, fn func(ctx context.Context) (T, error), metrics *Metrics) *Future[T] {
    ctx, cancel := context.WithCancel(parentCtx)
    
    f := &Future[T]{
        result:  make(chan T, 1),
        err:     make(chan error, 1),
        ctx:     ctx,
        cancel:  cancel,
        start:   time.Now(),
        metrics: metrics,
    }
    
    metrics.Started.Add(1)
    
    go func() {
        defer close(f.result)
        defer close(f.err)
        
        defer func() {
            if r := recover(); r != nil {
                f.err <- fmt.Errorf("panic: %v", r)
            }
        }()
        
        r, e := fn(ctx)
        if e != nil {
            f.err <- e
            return
        }
        f.result <- r
    }()
    
    return f
}

func (f *Future[T]) Get(ctx context.Context) (T, error) {
    var zero T
    
    select {
    case <-ctx.Done():
        f.cancel()
        f.metrics.Canceled.Add(1)
        return zero, ctx.Err()
    case err := <-f.err:
        f.metrics.Failed.Add(1)
        f.metrics.TotalTime.Add(int64(time.Since(f.start)))
        if errors.Is(err, context.Canceled) {
            f.metrics.Canceled.Add(1)
        }
        return zero, err
    case r := <-f.result:
        f.metrics.Completed.Add(1)
        f.metrics.TotalTime.Add(int64(time.Since(f.start)))
        return r, nil
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    metrics := &Metrics{}
    
    // 3 успешных
    futures := []*Future[string]{
        NewFuture(ctx, func(ctx context.Context) (string, error) {
            time.Sleep(100 * time.Millisecond)
            return "a", nil
        }, metrics),
        NewFuture(ctx, func(ctx context.Context) (string, error) {
            time.Sleep(150 * time.Millisecond)
            return "b", nil
        }, metrics),
        NewFuture(ctx, func(ctx context.Context) (string, error) {
            time.Sleep(200 * time.Millisecond)
            return "c", nil
        }, metrics),
    }
    
    // Один с ошибкой
    f4 := NewFuture(ctx, func(ctx context.Context) (string, error) {
        return "", errors.New("failed")
    }, metrics)
    
    for _, f := range futures {
        f.Get(ctx)
    }
    f4.Get(ctx)
    
    fmt.Println("=== Metrics ===")
    fmt.Printf("Started:   %d\n", metrics.Started.Load())
    fmt.Printf("Completed: %d\n", metrics.Completed.Load())
    fmt.Printf("Failed:    %d\n", metrics.Failed.Load())
    fmt.Printf("Canceled:  %d\n", metrics.Canceled.Load())
    fmt.Printf("TotalTime: %v\n", time.Duration(metrics.TotalTime.Load()))
}
```

**Пример вывода:**

```
=== Metrics ===
Started:   4
Completed: 3
Failed:    1
Canceled:  0
TotalTime: 450ms
```

**Что демонстрирует:** метрики по future — started, completed, failed, canceled.

### Пример: WaitAll с метриками

```go
func WaitAll[T any](ctx context.Context, futures []*Future[T]) ([]T, error) {
    results := make([]T, len(futures))
    
    var wg sync.WaitGroup
    var mu sync.Mutex
    var firstErr error
    
    for i, f := range futures {
        i, f := i, f
        wg.Add(1)
        go func() {
            defer wg.Done()
            r, err := f.Get(ctx)
            if err != nil {
                mu.Lock()
                if firstErr == nil {
                    firstErr = err
                }
                mu.Unlock()
                return
            }
            mu.Lock()
            results[i] = r
            mu.Unlock()
        }()
    }
    
    wg.Wait()
    return results, firstErr
}
```

### 💡 Практика: как измерять future

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики:** started, completed, failed, canceled.
2. **TotalTime** — суммарное время.
3. **Экспорт в Prometheus.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Различай failed и canceled.**
5. **Percentile latency.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `Mutex` для метрик.**

---

## 20.9 Выводы и типичные ошибки

**Что мы узнали?**

Future — объект, представляющий результат в будущем. **Простейший** — два канала (result + err), горутина, `Get(ctx)`. **`context`** для отмены. **Ошибки** — через отдельный канал или `Result[T]`. **`recover`** для паник. **`WaitAll` / `WaitAny`** для композиции. Future + cache для избежания дублирования. Future + worker pool для параллельных запросов.

**Типичные ошибки:**

- ❌ **Небуферизованные каналы.** Горутина зависнет, если `Get` не вызывается.
- ❌ **Не использовать `ctx`.** Зависание.
- ❌ **Игнорировать паники.** Future не вернёт ошибку.
- ❌ **Многоразовый `Get` без сохранения результата.**
- ❌ **Не различать failed и canceled.**
- ❌ **Гонки в `WaitAll`.** `Mutex` для `firstErr`.
- ❌ **Future в кэше без синхронизации.**
- ❌ **Использовать future для одного вызова.**

---

## 20.10 Для быстрого повторения

- **Future** — результат в будущем.
- **`NewFuture(ctx, fn)`** — создать.
- **`Get(ctx)`** — получить результат.
- **Буферизованные каналы** (буфер 1).
- **`recover`** — для паник.
- **`ctx`** — для отмены.
- **`WaitAll`** — все результаты.
- **`WaitAny`** — первый.
- **Future + cache** — избегает дублирования.
- **Future + worker pool** — параллельные запросы.
- **Метрики:** started, completed, failed, canceled.

---

## 20.11 Вопросы для самопроверки

1. Что такое future? Какую задачу решает?
2. Как построить простейший future?
3. Почему буферизованные каналы?
4. Зачем `ctx` в future?
5. Как обрабатывать паники в future?
6. Что такое `WaitAll` и `WaitAny`?
7. Как future помогает с кэшем?
8. Что выбрать — future или горутина + канал?

---

## 20.12 Ответы

### Ответ 1

**Future** — объект, представляющий результат в будущем. Решает задачу: **запустить операцию в фоне, получить результат позже**.

**Пример:** параллельные запросы к БД + API.

### Ответ 2

```go
type Future[T any] struct {
    result chan T
    err    chan error
}

func NewFuture[T any](fn func() (T, error)) *Future[T] {
    f := &Future[T]{
        result: make(chan T, 1),
        err:    make(chan error, 1),
    }
    go func() {
        defer close(f.result)
        defer close(f.err)
        r, e := fn()
        if e != nil {
            f.err <- e
            return
        }
        f.result <- r
    }()
    return f
}

func (f *Future[T]) Get(ctx context.Context) (T, error) {
    var zero T
    select {
    case <-ctx.Done():
        return zero, ctx.Err()
    case err := <-f.err:
        return zero, err
    case r := <-f.result:
        return r, nil
    }
}
```

### Ответ 3

**Буферизованные каналы** (буфер 1) позволяют горутине **записать результат** и **завершиться**, даже если никто не читал.

**Без буфера:** горутина зависнет на `f.result <- r`, если `Get` не вызывается. **Утечка.**

### Ответ 4

**`ctx`** позволяет отменить операцию. Без него future может **зависнуть**, даже если клиент ушёл.

**Решение:** `NewFuture(parentCtx, fn)` + `fn(ctx)`.

### Ответ 5

**`recover` в горутине:**

```go
go func() {
    defer close(f.result)
    defer close(f.err)
    
    defer func() {
        if r := recover(); r != nil {
            f.err <- fmt.Errorf("panic: %v", r)
        }
    }()
    
    r, e := fn(ctx)
    // ...
}()
```

### Ответ 6

**`WaitAll`** — ждём **все** future'ы. Возвращаем все результаты или первую ошибку.

**`WaitAny`** — ждём **первый** завершившийся. Возвращаем его результат.

### Ответ 7

**Future в кэше:**

```go
type Cache struct {
    mu      sync.Mutex
    futures map[string]*Future[User]
}

func (c *Cache) Get(ctx context.Context, id string) (User, error) {
    c.mu.Lock()
    f, ok := c.futures[id]
    if !ok {
        f = NewFuture(ctx, func(ctx context.Context) (User, error) {
            return fetchUser(ctx, id)
        })
        c.futures[id] = f
    }
    c.mu.Unlock()
    return f.Get(ctx)
}
```

Если два клиента запрашивают одного пользователя **одновременно** — **один** запрос к БД.

### Ответ 8

**Горутина + канал:**

- Один вызов.
- Простая логика.
- Не нужна абстракция.

**Future:**

- Много вызовов.
- Композиция.
- Асинхронный API.

**Future** для сложного, **горутина + канал** для простого.

---

## 20.13 Куда идти дальше?

Мы разобрали future — результат в будущем. Теперь мы умеем запускать операции в фоне и получать результат позже.

Но остаются **продвинутые паттерны**: bulkhead, leader election, sharded locks.

- **Как построить bulkhead?** → **Глава 22: Bulkhead.**
- **Как построить leader election?** → **Глава 24: Leader election.**
- **Как разбить лок на шарды?** → **Глава 25: Sharded locks.**

---

## 20.14 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Future** | Результат в будущем | `NewFuture(ctx, fn)` |
| **`Get(ctx)`** | Получить результат | Блокируется или `ctx.Done()` |
| **Буферизованные каналы** | Буфер 1 | Иначе утечка |
| **`recover`** | Для паник | В горутине |
| **`ctx`** | Отмена | `parentCtx` в `NewFuture` |
| **`WaitAll`** | Все результаты | `sync.WaitGroup` + `Mutex` |
| **`WaitAny`** | Первый | `select` на канал |
| **Future + cache** | Избегает дублирования | `Mutex` для map |
| **Future + worker pool** | Параллельные запросы | `WaitAll` |
| **Метрики** | started, completed, failed, canceled | `atomic.Int64` |

🔮 **Ключевая идея:** Future — объект, представляющий результат в будущем. **Простейший** — два канала (result + err), горутина, `Get(ctx)`. **Буферизованные каналы** обязательны. **`ctx`** для отмены. **`recover`** для паник. **`WaitAll` / `WaitAny`** для композиции. Future + cache для избежания дублирования (один запрос — много клиентов). Future + worker pool для параллельных запросов. Метрики: started, completed, failed, canceled. Не используй future для одного вызова — горутина + канал проще.