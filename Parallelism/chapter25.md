# ⚠️ Глава 25: Обработка ошибок в конкурентном коде

**Что вы узнаете:**
- Почему ошибки в конкурентном коде — это **сложнее**, чем в последовательном.
- Что такое **`errgroup`** и как он решает большинство задач.
- Как собрать **все** ошибки из N горутин, а не только первую.
- Что такое **`multierror`** и **`errors.Join`**.
- Как **отменять** работу при первой ошибке.
- Как обрабатывать **паники** в горутинах.
- Как логировать ошибки из конкурентного кода.
- Как не потерять ошибки в pipeline и worker pool.

**После прочтения вы сможете:**
- Использовать `errgroup` для конкурентных задач с ошибками.
- Собрать все ошибки через `errors.Join` или `multierror`.
- Отменять работу через `context` при первой ошибке.
- Обрабатывать паники в горутинах через `recover`.
- Логировать ошибки с контекстом (какая горутина, какая задача).
- Избегать типичных ошибок: потеря ошибок, гонки при записи в общий слайс, паника в горутине.

---

## Содержание

- [25.0 Пролог: одна ошибка из тысячи](#250-пролог-одна-ошибка-из-тысячи)
- [25.1 Проблема: ошибки в конкурентном коде](#251-проблема-ошибки-в-конкурентном-коде)
- [25.2 errgroup: стандартное решение](#252-errgroup-стандартное-решение)
- [25.3 Сбор всех ошибок: errors.Join и multierror](#253-сбор-всех-ошибок-errorsjoin-и-multierror)
- [25.4 Отмена при первой ошибке](#254-отмена-при-первой-ошибке)
- [25.5 Паники в горутинах: recover](#255-паники-в-горутинах-recover)
- [25.6 Логирование ошибок в конкурентном коде](#256-логирование-ошибок-в-конкурентном-коде)
- [25.7 Ошибки в pipeline и worker pool](#257-ошибки-в-pipeline-и-worker-pool)
- [25.8 В связке с другими паттернами](#258-в-связке-с-другими-паттернами)
- [25.9 Практика Go: errgroup с метриками](#259-практика-go-errgroup-с-метриками)
- [25.10 Выводы и типичные ошибки](#2510-выводы-и-типичные-ошибки)
- [25.11 Для быстрого повторения](#2511-для-быстрого-повторения)
- [25.12 Вопросы для самопроверки](#2512-вопросы-для-самопроверки)
- [25.13 Ответы](#2513-ответы)
- [25.14 Куда идти дальше?](#2514-куда-идти-дальше)
- [25.15 Чек-лист](#2515-чек-лист)

---

## 25.0 Пролог: одна ошибка из тысячи

Ты пишешь сервис, который синхронизирует данные с внешним API. 1000 пользователей, для каждого — HTTP-запрос:

```go
func syncUsers(ctx context.Context, users []User) error {
    var wg sync.WaitGroup
    var mu sync.Mutex
    var firstErr error

    for _, user := range users {
        wg.Add(1)
        go func(u User) {
            defer wg.Done()
            resp, err := http.Get("https://api.example.com/users/" + u.ID)
            if err != nil {
                mu.Lock()
                if firstErr == nil {
                    firstErr = err
                }
                mu.Unlock()
                return
            }
            defer resp.Body.Close()
            // ...
        }(user)
    }
    wg.Wait()
    return firstErr
}
```

Код работает. Но есть **проблемы**:

- **Только первая ошибка.** Если 100 запросов упали, ты видишь только один.
- **Нет отмены.** Остальные 999 горутин продолжают работу, хотя уже ясно, что что-то не так.
- **Mutex для ошибки.** Гонка при записи в `firstErr` без блокировки.
- **Нет контекста.** Какая задача упала? Какой пользователь?

❓ **Как сделать лучше?**

💡 **Решение:** `errgroup` + `context` + `errors.Join`.

```go
func syncUsers(ctx context.Context, users []User) error {
    g, ctx := errgroup.WithContext(ctx)

    for _, user := range users {
        user := user
        g.Go(func() error {
            resp, err := http.Get("https://api.example.com/users/" + user.ID)
            if err != nil {
                return fmt.Errorf("user %s: %w", user.ID, err)
            }
            defer resp.Body.Close()
            return nil
        })
    }

    return g.Wait()
}
```

**Что изменилось:**

- **Отмена при первой ошибке** — `errgroup.WithContext`.
- **Первая ошибка с контекстом** — `fmt.Errorf("user %s: %w", ...)`.
- **Нет Mutex** — `errgroup` сам синхронизирует.
- **Меньше кода.**

> **Мост к следующим главам:** обработка ошибок — основа graceful shutdown (Глава 11). `errgroup` — стандартный инструмент для конкурентных задач. `errors.Join` — для сбора всех ошибок. Понимание обработки ошибок критично для production-кода.

---

## 25.1 Проблема: ошибки в конкурентном коде

Прежде чем разбирать решения, поймём **проблемы**.

### Проблема 1: гонка при записи ошибки

**❌ Плохо:**

```go
var firstErr error

for _, user := range users {
    go func(u User) {
        if err := process(u); err != nil {
            firstErr = err  // ← ГОНКА!
        }
    }(user)
}
```

**Что происходит:** несколько горутин пишут в `firstErr` одновременно. **Data race** (Глава 7).

**Решение:** `Mutex` или `atomic`:

```go
var (
    mu       sync.Mutex
    firstErr error
)

mu.Lock()
if firstErr == nil {
    firstErr = err
}
mu.Unlock()
```

**Но:** это много кода. Лучше — `errgroup`.

### Проблема 2: потеря ошибок

**❌ Плохо:**

```go
for _, user := range users {
    go func(u User) {
        if err := process(u); err != nil {
            log.Println(err)  // ← только логируем
        }
    }(user)
}
```

**Что происходит:** ошибки **теряются**. Функция возвращает `nil`, хотя задачи упали.

**Решение:** собрать ошибки и вернуть.

### Проблема 3: нет отмены

**❌ Плохо:**

```go
for _, user := range users {
    go func(u User) {
        process(u)  // ← продолжает даже если что-то упало
    }(user)
}
```

**Что происходит:** если первая задача упала — остальные **продолжают** работу. Ресурсы тратятся зря.

**Решение:** `context` + `errgroup` для отмены.

### Проблема 4: нет контекста ошибки

**❌ Плохо:**

```go
if err := process(u); err != nil {
    return err  // ← какая задача? какой пользователь?
}
```

**Что происходит:** ошибка без контекста. Непонятно, **что** упало.

**Решение:** `fmt.Errorf` с контекстом:

```go
return fmt.Errorf("user %s: %w", u.ID, err)
```

### Проблема 5: паника в горутине

**❌ Плохо:**

```go
go func() {
    panic("oops")  // ← программа падает
}()
```

**Что происходит:** паника в горутине **не может быть восстановлена** из main. **Вся программа падает.**

**Решение:** `recover` в каждой горутине (см. 25.5).

### Сводная таблица

| Проблема | Решение |
|:---|:---|
| Гонка при записи ошибки | `Mutex`, `atomic`, `errgroup` |
| Потеря ошибок | Собрать и вернуть |
| Нет отмены | `context` + `errgroup` |
| Нет контекста | `fmt.Errorf` с `%w` |
| Паника в горутине | `recover` |

### 💡 Практика: как не потерять ошибки

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Собирай ошибки** — не логируй и не игнорируй.
2. **Возвращай ошибки** — не теряй.
3. **Добавляй контекст** — `fmt.Errorf("task %d: %w", id, err)`.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`errgroup`** — для большинства случаев.
5. **`recover`** — в каждой горутине.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **`multierror`** — если нужны все ошибки.

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй ошибки.** Логирование — не обработка.
8. **Не записывай ошибку без `Mutex`.** Гонка.
9. **Не паникуй в горутине** без `recover`.

---

## 25.2 errgroup: стандартное решение

**`errgroup`** — пакет для запуска группы горутин с обработкой ошибок.

### Установка

```bash
go get golang.org/x/sync/errgroup
```

### Шаг 1: базовый errgroup

```go
import "golang.org/x/sync/errgroup"

func main() {
    var g errgroup.Group

    g.Go(func() error {
        return process(1)
    })
    g.Go(func() error {
        return process(2)
    })

    if err := g.Wait(); err != nil {
        fmt.Println("error:", err)
    }
}
```

**Что делает:**

- `g.Go(fn)` — запускает горутину.
- `g.Wait()` — ждёт все, возвращает первую ошибку.

**Ключевое:** `errgroup` **синхронизирует** ошибки. Не нужно `Mutex`.

### Шаг 2: errgroup с context

```go
func main() {
    ctx := context.Background()
    g, ctx := errgroup.WithContext(ctx)

    g.Go(func() error {
        return process(ctx, 1)
    })
    g.Go(func() error {
        return process(ctx, 2)
    })

    if err := g.Wait(); err != nil {
        fmt.Println("error:", err)
    }
}
```

**Что изменилось:** `errgroup.WithContext` создаёт **общий** `ctx`.

**Что происходит:**

- При **первой** ошибке `ctx` отменяется.
- Все горутины видят `ctx.Done()` и завершаются.
- `g.Wait()` возвращает первую ошибку.

**Это ключевое преимущество:** автоматическая отмена при ошибке.

### Шаг 3: полный пример

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "time"

    "golang.org/x/sync/errgroup"
)

func main() {
    users := []string{"user1", "user2", "user3", "user4", "user5"}
    ctx := context.Background()

    if err := syncUsers(ctx, users); err != nil {
        fmt.Println("error:", err)
    }
}

func syncUsers(ctx context.Context, users []string) error {
    g, ctx := errgroup.WithContext(ctx)

    for _, user := range users {
        user := user
        g.Go(func() error {
            return processUser(ctx, user)
        })
    }

    return g.Wait()
}

func processUser(ctx context.Context, user string) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-time.After(100 * time.Millisecond):
        if user == "user3" {
            return fmt.Errorf("user %s: %w", user, errors.New("not found"))
        }
        fmt.Printf("processed %s\n", user)
        return nil
    }
}
```

**Что происходит:**

1. 5 горутин запускаются.
2. `user3` возвращает ошибку.
3. `ctx` отменяется.
4. Остальные горутины видят `ctx.Done()` и завершаются.
5. `g.Wait()` возвращает ошибку `user3: not found`.

### Шаг 4: errgroup внутри worker pool

**Классический паттерн:** N воркеров + errgroup.

```go
func processAll(ctx context.Context, tasks []Task, numWorkers int) error {
    g, ctx := errgroup.WithContext(ctx)
    tasksCh := make(chan Task, len(tasks))

    for _, task := range tasks {
        tasksCh <- task
    }
    close(tasksCh)

    for i := 0; i < numWorkers; i++ {
        g.Go(func() error {
            for task := range tasksCh {
                if err := processTask(ctx, task); err != nil {
                    return fmt.Errorf("task %d: %w", task.ID, err)
                }
            }
            return nil
        })
    }

    return g.Wait()
}
```

### Шаг 5: errgroup с ограничением

**`errgroup.Group`** не ограничивает число горутин. Если нужно — используй `SetLimit`:

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(10)  // максимум 10 горутин

for _, task := range tasks {
    task := task
    g.Go(func() error {
        return process(ctx, task)
    })
}

return g.Wait()
```

**Что делает `SetLimit`:**

- Не более 10 горутин **одновременно**.
- `g.Go` **блокируется**, если лимит достигнут.
- Это **семафор** внутри `errgroup`.

### Сравнение errgroup с ручным решением

| Аспект | Ручное | `errgroup` |
|:---|:---|:---|
| Гонка | Нужен `Mutex` | Автоматически |
| Отмена | Ручной `context` | `WithContext` |
| Первая ошибка | Ручная логика | Автоматически |
| Код | Много | Мало |

### 💡 Практика: как использовать errgroup

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`errgroup.WithContext(ctx)`** — для отмены.
2. **`g.Go(func() error { ... })`** — для каждой горутины.
3. **`g.Wait()`** — для ожидания.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`fmt.Errorf("task %d: %w", id, err)`** — для контекста.
5. **`g.SetLimit(N)`** — если задач много.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `errgroup` для задач, которые не возвращают ошибку.** Просто `WaitGroup`.
7. **Не забывай `g.Wait()`.** Утечка.
8. **Не игнорируй возвращённую ошибку.**

---

## 25.3 Сбор всех ошибок: errors.Join и multierror

`errgroup` возвращает **первую** ошибку. Но иногда нужны **все**.

### errors.Join (Go 1.20+)

**`errors.Join`** — объединяет несколько ошибок в одну.

```go
import "errors"

err1 := errors.New("error 1")
err2 := errors.New("error 2")
err3 := errors.New("error 3")

joined := errors.Join(err1, err2, err3)
fmt.Println(joined)
// error 1
// error 2
// error 3
```

**Что делает:**

- Создаёт **составную** ошибку.
- `joined.Error()` возвращает все ошибки через `\n`.
- `joined.Unwrap()` возвращает `[]error` — слайс всех ошибок.

### Шаг 1: структура для сбора

```go
type Collector struct {
    mu   sync.Mutex
    errs []error
}

func (c *Collector) Add(err error) {
    if err == nil {
        return
    }
    c.mu.Lock()
    c.errs = append(c.errs, err)
    c.mu.Unlock()
}

func (c *Collector) Err() error {
    c.mu.Lock()
    defer c.mu.Unlock()
    return errors.Join(c.errs...)
}
```

### Шаг 2: использование в горутинах

```go
func processAll(ctx context.Context, tasks []Task) error {
    var (
        wg        sync.WaitGroup
        collector Collector
    )

    for _, task := range tasks {
        task := task
        wg.Add(1)
        go func() {
            defer wg.Done()
            if err := process(ctx, task); err != nil {
                collector.Add(fmt.Errorf("task %d: %w", task.ID, err))
            }
        }()
    }

    wg.Wait()
    return collector.Err()
}
```

### Шаг 3: полный пример

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "sync"
    "time"
)

type Task struct {
    ID      int
    Failing bool
}

type Collector struct {
    mu   sync.Mutex
    errs []error
}

func (c *Collector) Add(err error) {
    if err == nil {
        return
    }
    c.mu.Lock()
    c.errs = append(c.errs, err)
    c.mu.Unlock()
}

func (c *Collector) Err() error {
    c.mu.Lock()
    defer c.mu.Unlock()
    return errors.Join(c.errs...)
}

func main() {
    tasks := []Task{
        {ID: 1},
        {ID: 2, Failing: true},
        {ID: 3},
        {ID: 4, Failing: true},
        {ID: 5},
    }

    if err := processAll(context.Background(), tasks); err != nil {
        fmt.Println("errors:")
        fmt.Println(err)
    }
}

func processAll(ctx context.Context, tasks []Task) error {
    var (
        wg        sync.WaitGroup
        collector Collector
    )

    for _, task := range tasks {
        task := task
        wg.Add(1)
        go func() {
            defer wg.Done()
            if err := process(ctx, task); err != nil {
                collector.Add(fmt.Errorf("task %d: %w", task.ID, err))
            }
        }()
    }

    wg.Wait()
    return collector.Err()
}

func process(ctx context.Context, task Task) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-time.After(100 * time.Millisecond):
        if task.Failing {
            return errors.New("failed")
        }
        fmt.Printf("task %d: ok\n", task.ID)
        return nil
    }
}
```

**Пример вывода:**

```
task 1: ok
task 3: ok
task 5: ok
errors:
task 2: failed
task 4: failed
```

**Что видно:** все ошибки собраны. Каждая с контекстом задачи.

### multierror (HashiCorp)

**Для Go < 1.20** — `github.com/hashicorp/go-multierror`.

```go
import "github.com/hashicorp/go-multierror"

var result *multierror.Error

result = multierror.Append(result, err1)
result = multierror.Append(result, err2)

return result.ErrorOrNil()
```

**API:**

- `multierror.Append(result, err)` — добавить.
- `result.ErrorOrNil()` — вернуть ошибку или `nil`.
- `result.Errors` — слайс ошибок.

### errgroup + errors.Join

**Комбинация:** `errgroup` + `errors.Join` для сбора **всех** ошибок.

```go
func processAll(ctx context.Context, tasks []Task) error {
    g, ctx := errgroup.WithContext(ctx)

    var (
        mu   sync.Mutex
        errs []error
    )

    for _, task := range tasks {
        task := task
        g.Go(func() error {
            if err := process(ctx, task); err != nil {
                mu.Lock()
                errs = append(errs, fmt.Errorf("task %d: %w", task.ID, err))
                mu.Unlock()
            }
            return nil  // не отменяем
        })
    }

    g.Wait()
    return errors.Join(errs...)
}
```

### 💡 Практика: как собирать все ошибки

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`errors.Join`** (Go 1.20+) — для сбора.
2. **`Mutex`** — для синхронизации `errs`.
3. **`fmt.Errorf("task %d: %w", id, err)`** — для контекста.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`multierror`** — если Go < 1.20.

**❌ НЕ ДЕЛАЙ:**

5. **Не пиши в `errs` без `Mutex`.** Гонка.
6. **Не забывай `errors.Join`.** Без него — только последняя ошибка.

---

## 25.4 Отмена при первой ошибке

Разберём **отмену** работы при первой ошибке.

### Зачем отменять

**Сценарии:**

- Один из 1000 запросов упал — нет смысла продолжать.
- Критичная ошибка — нужно остановить всё.
- Таймаут — работа больше не нужна.

### Паттерн: errgroup.WithContext

**Классический паттерн:**

```go
func processAll(ctx context.Context, tasks []Task) error {
    g, ctx := errgroup.WithContext(ctx)

    for _, task := range tasks {
        task := task
        g.Go(func() error {
            return process(ctx, task)
        })
    }

    return g.Wait()
}
```

**Что происходит:**

1. `WithContext` создаёт **дочерний** `ctx`.
2. Каждая горутина получает `ctx`.
3. При **первой** ошибке — `errgroup` **отменяет** `ctx`.
4. Все горутины видят `ctx.Done()`.
5. `g.Wait()` возвращает первую ошибку.

### Что должна делать горутина

**Ключевое:** горутина должна **проверять `ctx.Done()`**.

```go
func process(ctx context.Context, task Task) error {
    select {
    case <-ctx.Done():
        return ctx.Err()  // отмена
    default:
    }

    // долгая операция
    if err := doWork(ctx, task); err != nil {
        return err
    }

    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
    }

    return nil
}
```

### Полный пример

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "time"

    "golang.org/x/sync/errgroup"
)

func main() {
    tasks := []Task{
        {ID: 1, Duration: 100 * time.Millisecond},
        {ID: 2, Duration: 200 * time.Millisecond, Failing: true},
        {ID: 3, Duration: 300 * time.Millisecond},
        {ID: 4, Duration: 400 * time.Millisecond},
        {ID: 5, Duration: 500 * time.Millisecond},
    }

    start := time.Now()
    if err := processAll(context.Background(), tasks); err != nil {
        fmt.Printf("error: %v (elapsed: %v)\n", err, time.Since(start))
    }
}

type Task struct {
    ID       int
    Duration time.Duration
    Failing  bool
}

func processAll(ctx context.Context, tasks []Task) error {
    g, ctx := errgroup.WithContext(ctx)

    for _, task := range tasks {
        task := task
        g.Go(func() error {
            return process(ctx, task)
        })
    }

    return g.Wait()
}

func process(ctx context.Context, task Task) error {
    fmt.Printf("task %d: starting\n", task.ID)

    select {
    case <-ctx.Done():
        fmt.Printf("task %d: canceled\n", task.ID)
        return ctx.Err()
    case <-time.After(task.Duration):
        if task.Failing {
            fmt.Printf("task %d: failed\n", task.ID)
            return fmt.Errorf("task %d: %w", task.ID, errors.New("failed"))
        }
        fmt.Printf("task %d: done\n", task.ID)
        return nil
    }
}
```

**Пример вывода:**

```
task 5: starting
task 1: starting
task 3: starting
task 2: starting
task 4: starting
task 1: done
task 2: failed
task 5: canceled
task 3: canceled
task 4: canceled
error: task 2: failed (elapsed: 200ms)
```

**Что видно:**

- Задачи 1 и 2 завершились.
- Задача 2 упала — `ctx` отменён.
- Задачи 3, 4, 5 — canceled.
- Общее время — 200 мс (время до ошибки).

**Без отмены** общее время было бы 500 мс.

### Обработка ctx.Err()

**Важно:** `ctx.Err()` — **не ошибка задачи**. Это **отмена**.

```go
if err := process(ctx, task); err != nil {
    if errors.Is(err, context.Canceled) {
        // задача отменена — не считаем ошибкой
        return nil
    }
    return err
}
```

### 💡 Практика: как отменять

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`errgroup.WithContext`** — для отмены.
2. **`select` с `ctx.Done()`** в горутинах.
3. **`errors.Is(err, context.Canceled)`** — для отмены.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`context.WithTimeout`** — для общего таймаута.

**❌ НЕ ДЕЛАЙ:**

5. **Не игнорируй `ctx.Done()`.** Отмена не сработает.
6. **Не считай `context.Canceled` ошибкой.**

---

## 25.5 Паники в горутинах: recover

**Паника в горутине** — особая проблема. `recover` в main её **не поймает**.

### Проблема

```go
func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("recovered:", r)
        }
    }()

    go func() {
        panic("oops")  // ← программа падает
    }()

    time.Sleep(1 * time.Second)
    fmt.Println("done")
}
```

**Что происходит:**

- `recover` в main **не поймает** панику из другой горутины.
- Программа **падает** с `panic: oops`.

### Решение: recover в каждой горутине

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("recovered:", r)
        }
    }()
    panic("oops")
}()
```

### Шаг 1: обёртка для горутины

```go
func safeGo(fn func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("panic in goroutine: %v\n%s", r, debug.Stack())
            }
        }()
        fn()
    }()
}
```

### Шаг 2: использование в errgroup

```go
func processAll(ctx context.Context, tasks []Task) error {
    g, ctx := errgroup.WithContext(ctx)

    for _, task := range tasks {
        task := task
        g.Go(func() (err error) {
            defer func() {
                if r := recover(); r != nil {
                    err = fmt.Errorf("task %d: panic: %v", task.ID, r)
                }
            }()
            return process(ctx, task)
        })
    }

    return g.Wait()
}
```

**Ключевое:** паника **превращается в ошибку**. Не роняет программу.

### Шаг 3: полный пример

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "runtime/debug"
    "sync"

    "golang.org/x/sync/errgroup"
)

type Task struct {
    ID     int
    Panics bool
    Fails  bool
}

func main() {
    tasks := []Task{
        {ID: 1},
        {ID: 2, Fails: true},
        {ID: 3, Panics: true},
        {ID: 4},
    }

    if err := processAll(context.Background(), tasks); err != nil {
        fmt.Println("errors:")
        fmt.Println(err)
    }
}

func processAll(ctx context.Context, tasks []Task) error {
    var (
        mu   sync.Mutex
        errs []error
    )

    g, ctx := errgroup.WithContext(ctx)

    for _, task := range tasks {
        task := task
        g.Go(func() (err error) {
            defer func() {
                if r := recover(); r != nil {
                    err = fmt.Errorf("task %d: panic: %v\n%s",
                        task.ID, r, debug.Stack())
                }
            }()

            if err := process(ctx, task); err != nil {
                mu.Lock()
                errs = append(errs, fmt.Errorf("task %d: %w", task.ID, err))
                mu.Unlock()
            }
            return nil
        })
    }

    g.Wait()
    return errors.Join(errs...)
}

func process(ctx context.Context, task Task) error {
    if task.Panics {
        panic("something went wrong")
    }
    if task.Fails {
        return errors.New("failed")
    }
    fmt.Printf("task %d: ok\n", task.ID)
    return nil
}
```

**Пример вывода:**

```
task 1: ok
task 4: ok
errors:
task 2: failed
task 3: panic: something went wrong
goroutine 20 [running]:
...
```

**Что видно:**

- Задача 2 — ошибка.
- Задача 3 — паника → ошибка.
- Задачи 1, 4 — ок.
- Программа **не упала**.

### Важно: не все паники можно восстановить

**Некоторые паники нельзя восстановить:**

- **`fatal error: concurrent map writes`** — runtime-паника.
- **`runtime: out of memory`** — OOM.
- **`fatal error: all goroutines are asleep - deadlock!`** — deadlock.
- **`runtime.throw`** — низкоуровневые ошибки.

**Что делать:**

- **Не допускать** таких паник.
- **Использовать `Mutex`** для map.
- **Мониторить память.**
- **Тестировать** на deadlock.

### 💡 Практика: как обрабатывать паники

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`defer recover`** в каждой горутине.
2. **Логируй панику** — `debug.Stack()`.
3. **Превращай панику в ошибку** — для `errgroup`.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`safeGo`** — обёртка для горутин.
5. **Метрики** — сколько паник.

**❌ НЕ ДЕЛАЙ:**

6. **Не игнорируй паники.** Логируй и исправляй.
7. **Не полагайся на recover для runtime-паник.** Они не восстанавливаются.
8. **Не используй panic для контроля потока.** Только для багов.

---

## 25.6 Логирование ошибок в конкурентном коде

Разберём **логирование** ошибок.

### Проблема: логи без контекста

**❌ Плохо:**

```go
if err := process(task); err != nil {
    log.Println(err)  // ← какая задача?
}
```

**Что происходит:** в логе непонятно, **что** упало.

### Решение: контекст в ошибке

**✅ Хорошо:**

```go
if err := process(task); err != nil {
    log.Printf("task %d failed: %v", task.ID, err)
}
```

**Или через `fmt.Errorf`:**

```go
if err := process(task); err != nil {
    return fmt.Errorf("task %d: %w", task.ID, err)
}
```

### Контекст из context.Context

**`context.Value`** — для request-scoped данных:

```go
type ctxKey int

const (
    requestIDKey ctxKey = iota
    userIDKey
)

func process(ctx context.Context, task Task) error {
    requestID, _ := ctx.Value(requestIDKey).(string)
    userID, _ := ctx.Value(userIDKey).(int)

    if err := doWork(task); err != nil {
        log.Printf("[request=%s user=%d task=%d] %v",
            requestID, userID, task.ID, err)
        return err
    }
    return nil
}
```

### Structured logging

**`log/slog`** (Go 1.21+):

```go
import "log/slog"

slog.Error("task failed",
    "task_id", task.ID,
    "user_id", userID,
    "error", err,
)
```

**Что даёт:** структурированный лог, легко парсить.

### Агрегация логов

**Для production** — не логируй каждую ошибку отдельно. **Агрегируй**:

```go
type ErrorCollector struct {
    mu    sync.Mutex
    errs  []error
    count int
}

func (c *ErrorCollector) Add(err error) {
    c.mu.Lock()
    c.errs = append(c.errs, err)
    c.count++
    c.mu.Unlock()
}

func (c *ErrorCollector) Summary() string {
    c.mu.Lock()
    defer c.mu.Unlock()
    if len(c.errs) == 0 {
        return "no errors"
    }
    return fmt.Sprintf("%d errors: %v", c.count, errors.Join(c.errs...))
}
```

### Когда логировать, когда возвращать

| Ситуация | Действие |
|:---|:---|
| Критичная ошибка | Вернуть |
| Некритичная ошибка | Логировать |
| Ошибка с контекстом | Вернуть |
| Ошибка в фоне | Логировать |
| Ошибка в API | Вернуть клиенту |

**Правило:** **возвращай** ошибки, которые **можно обработать**. **Логируй** — которые **нельзя**.

### 💡 Практика: как логировать ошибки

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Контекст в ошибке** — `fmt.Errorf("task %d: %w", ...)`.
2. **`slog`** — для structured logging.
3. **Агрегация** — для production.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`context.Value`** — для request-scoped данных (request ID, user ID).
5. **Метрики** — сколько ошибок.

**❌ НЕ ДЕЛАЙ:**

6. **Не логируй каждую ошибку отдельно** в цикле.
7. **Не логируй ошибки без контекста.**
8. **Не используй `fmt.Println`** — используй `log` или `slog`.

---

## 25.7 Ошибки в pipeline и worker pool

Разберём **обработку ошибок** в pipeline и worker pool.

### Ошибки в worker pool

**Классический паттерн:** воркеры возвращают ошибки через канал.

```go
type Result struct {
    TaskID int
    Err    error
}

func processAll(ctx context.Context, tasks []Task, numWorkers int) error {
    tasksCh := make(chan Task, len(tasks))
    resultsCh := make(chan Result, len(tasks))

    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for task := range tasksCh {
                err := process(ctx, task)
                resultsCh <- Result{TaskID: task.ID, Err: err}
            }
        }()
    }

    go func() {
        for _, task := range tasks {
            tasksCh <- task
        }
        close(tasksCh)
    }()

    go func() {
        wg.Wait()
        close(resultsCh)
    }()

    var errs []error
    for result := range resultsCh {
        if result.Err != nil {
            errs = append(errs, fmt.Errorf("task %d: %w", result.TaskID, result.Err))
        }
    }

    return errors.Join(errs...)
}
```

### Ошибки в pipeline

**Паттерн:** каждая стадия возвращает ошибки через выходной канал.

```go
type Message struct {
    ID    int
    Data  string
    Err   error
}

func parseStage(ctx context.Context, input <-chan Message) <-chan Message {
    out := make(chan Message)
    go func() {
        defer close(out)
        for msg := range input {
            parsed, err := parse(msg.Data)
            select {
            case out <- Message{ID: msg.ID, Data: parsed, Err: err}:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

**Что происходит:**

- `Message` содержит `Err`.
- Стадия пишет ошибку в выходной канал.
- Следующая стадия видит ошибку.

### Отмена pipeline при ошибке

**`errgroup` + pipeline:**

```go
func runPipeline(ctx context.Context, data []string) error {
    g, ctx := errgroup.WithContext(ctx)

    source := generator(ctx, data)
    parsed := parseStage(ctx, source)
    enriched := enrichStage(ctx, parsed, 10)

    g.Go(func() error {
        for msg := range enriched {
            if msg.Err != nil {
                return fmt.Errorf("message %d: %w", msg.ID, msg.Err)
            }
        }
        return nil
    })

    return g.Wait()
}
```

### 💡 Практика: как обрабатывать ошибки в pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`Result{Err}` или `Message{Err}`** — для передачи ошибок.
2. **`context`** — для отмены.
3. **`errors.Join`** — для сбора.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`errgroup`** — для отмены при первой ошибке.
5. **Метрики** — сколько ошибок на стадии.

**❌ НЕ ДЕЛАЙ:**

6. **Не игнорируй ошибки в стадиях.**
7. **Не забывай `context`.** Отмена не сработает.

---

## 25.8 В связке с другими паттернами

Обработка ошибок редко используется **в одиночку**. Разберём связки.

### errgroup + worker pool

```go
func processAll(ctx context.Context, tasks []Task, numWorkers int) error {
    g, ctx := errgroup.WithContext(ctx)
    tasksCh := make(chan Task, len(tasks))

    for _, task := range tasks {
        tasksCh <- task
    }
    close(tasksCh)

    for i := 0; i < numWorkers; i++ {
        g.Go(func() error {
            for task := range tasksCh {
                if err := processTask(ctx, task); err != nil {
                    return fmt.Errorf("task %d: %w", task.ID, err)
                }
            }
            return nil
        })
    }

    return g.Wait()
}
```

### errgroup + pipeline

```go
func runPipeline(ctx context.Context, data []string) error {
    g, ctx := errgroup.WithContext(ctx)

    g.Go(func() error {
        return runSource(ctx, data)
    })
    g.Go(func() error {
        return runParse(ctx)
    })
    g.Go(func() error {
        return runEnrich(ctx)
    })

    return g.Wait()
}
```

### errors.Join + pipeline

**Собрать все ошибки из стадии:**

```go
func parseStage(ctx context.Context, input <-chan Message) (<-chan Message, error) {
    out := make(chan Message)
    var errs []error

    go func() {
        defer close(out)
        for msg := range input {
            parsed, err := parse(msg.Data)
            if err != nil {
                errs = append(errs, fmt.Errorf("msg %d: %w", msg.ID, err))
                continue
            }
            select {
            case out <- Message{ID: msg.ID, Data: parsed}:
            case <-ctx.Done():
                return
            }
        }
    }()

    return out, errors.Join(errs...)
}
```

### errgroup + retry

**errgroup + retry с обработкой ошибок:**

```go
func processAll(ctx context.Context, tasks []Task) error {
    g, ctx := errgroup.WithContext(ctx)

    for _, task := range tasks {
        task := task
        g.Go(func() error {
            return retryCtx(ctx, 3, 100*time.Millisecond, func(ctx context.Context) error {
                return process(ctx, task)
            })
        })
    }

    return g.Wait()
}
```

### errgroup + circuit breaker

**errgroup + circuit breaker для внешних вызовов:**

```go
func processAll(ctx context.Context, tasks []Task, breaker *CircuitBreaker) error {
    g, ctx := errgroup.WithContext(ctx)

    for _, task := range tasks {
        task := task
        g.Go(func() error {
            return breaker.Call(ctx, func(ctx context.Context) error {
                return process(ctx, task)
            })
        })
    }

    return g.Wait()
}
```

### Полная защита

```
Задачи
   │
   ▼
errgroup.WithContext
   │
   ├── retry
   │   └── circuit breaker
   │       └── process (with recover)
   │
   ▼
errors.Join
   │
   ▼
Итоговая ошибка
```

### 💡 Практика: как комбинировать обработку ошибок

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`errgroup` + worker pool** — для конкурентных задач.
2. **`errgroup` + pipeline** — для стадий.
3. **`errors.Join`** — для сбора всех ошибок.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`errgroup` + retry** — повторные попытки.
5. **`errgroup` + circuit breaker** — защита от сбоев.

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай про `recover`.**
7. **Не теряй ошибки.**

---

## 25.9 Практика Go: errgroup с метриками

Разберём **errgroup с метриками**.

### Полный код

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "runtime/debug"
    "sync/atomic"
    "time"

    "golang.org/x/sync/errgroup"
)

type Metrics struct {
    Total   atomic.Int64
    Success atomic.Int64
    Failed  atomic.Int64
    Panics  atomic.Int64
}

type Task struct {
    ID       int
    Duration time.Duration
    Failing  bool
    Panics   bool
}

func main() {
    tasks := make([]Task, 100)
    for i := range tasks {
        tasks[i] = Task{
            ID:       i,
            Duration: 10 * time.Millisecond,
            Failing:  i%10 == 0,
            Panics:   i%20 == 0,
        }
    }

    var metrics Metrics
    start := time.Now()

    err := processAll(context.Background(), tasks, &metrics)
    elapsed := time.Since(start)

    fmt.Printf("\n=== Результаты ===\n")
    fmt.Printf("Total:   %d\n", metrics.Total.Load())
    fmt.Printf("Success: %d\n", metrics.Success.Load())
    fmt.Printf("Failed:  %d\n", metrics.Failed.Load())
    fmt.Printf("Panics:  %d\n", metrics.Panics.Load())
    fmt.Printf("Elapsed: %v\n", elapsed)

    if err != nil {
        fmt.Printf("\nErrors:\n%v\n", err)
    }
}

func processAll(ctx context.Context, tasks []Task, metrics *Metrics) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(10)  // максимум 10 горутин

    for _, task := range tasks {
        task := task
        g.Go(func() (err error) {
            metrics.Total.Add(1)

            defer func() {
                if r := recover(); r != nil {
                    metrics.Panics.Add(1)
                    err = fmt.Errorf("task %d: panic: %v\n%s",
                        task.ID, r, debug.Stack())
                }
            }()

            if err := process(ctx, task); err != nil {
                metrics.Failed.Add(1)
                return fmt.Errorf("task %d: %w", task.ID, err)
            }

            metrics.Success.Add(1)
            return nil
        })
    }

    return g.Wait()
}

func process(ctx context.Context, task Task) error {
    if task.Panics {
        panic("simulated panic")
    }

    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-time.After(task.Duration):
        if task.Failing {
            return errors.New("simulated failure")
        }
        return nil
    }
}
```

**Пример вывода:**

```
=== Результаты ===
Total:   100
Success: 81
Failed:  9
Panics:  5
Elapsed: 105ms

Errors:
task 0: panic: simulated panic
goroutine 20 [running]:
...
task 10: simulated failure
task 20: panic: simulated panic
...
```

**Что видно:**

- 100 задач.
- 81 успешных.
- 9 failed (i%10 == 0, но i%20 != 0).
- 5 panics (i%20 == 0).
- Общее время — 105 мс (с лимитом 10 горутин).

### 💡 Практика: как измерять errgroup

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики:** total, success, failed, panics.
2. **`atomic.Int64`** — для счётчиков.
3. **`recover`** в каждой горутине.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`SetLimit`** — для ограничения.
5. **`debug.Stack()`** — для паник.
6. **Экспорт в Prometheus.**

**❌ НЕ ДЕЛАЙ:**

7. **Не забывай `recover`.** Программа упадёт.
8. **Не игнорируй паники.**

---

## 25.10 Выводы и типичные ошибки

**Что мы узнали?**

Обработка ошибок в конкурентном коде сложнее, чем в последовательном. `errgroup` — стандартное решение: `g.Go(fn)`, `g.Wait()`, `WithContext` для отмены, `SetLimit` для ограничения. `errors.Join` (Go 1.20+) — для сбора всех ошибок. `multierror` — для Go < 1.20. Отмена при первой ошибке через `errgroup.WithContext` + `context`. Паники в горутинах — `recover` в каждой горутине. Логирование — с контекстом, агрегация, `slog`. Worker pool и pipeline — `Result{Err}` / `Message{Err}` + `errors.Join`.

**Типичные ошибки:**

- ❌ **Гонка при записи ошибки.** `Mutex` или `errgroup`.
- ❌ **Потеря ошибок.** Собирай и возвращай.
- ❌ **Нет отмены.** `errgroup.WithContext`.
- ❌ **Нет контекста в ошибке.** `fmt.Errorf` с `%w`.
- ❌ **Паника в горутине без `recover`.** Программа падает.
- ❌ **`recover` в main не ловит панику из горутины.**
- ❌ **Игнорирование `ctx.Err()`.** Отмена не сработает.
- ❌ **`context.Canceled` как ошибка.** Это отмена.
- ❌ **Логирование без контекста.**
- ❌ **Логирование каждой ошибки в цикле.**
- ❌ **Забыть `errgroup.SetLimit`.** 1000 горутин.
- ❌ **Не использовать `errors.Join`.** Только последняя ошибка.

---

## 25.11 Для быстрого повторения

- **Проблемы:** гонка, потеря, нет отмены, нет контекста, паника.
- **`errgroup`** — `g.Go(fn)`, `g.Wait()`, `WithContext`, `SetLimit`.
- **`errors.Join`** (Go 1.20+) — для сбора ошибок.
- **`multierror`** — для Go < 1.20.
- **Отмена:** `errgroup.WithContext` + `select` с `ctx.Done()`.
- **`context.Canceled`** — не ошибка, а отмена.
- **`context.WithTimeout`** — для таймаута.
- **`signal.NotifyContext`** — для сигнала ОС.
- **Паника в горутине** — `recover` в каждой горутине.
- **`debug.Stack()`** — для логирования паник.
- **Runtime-паники не восстанавливаются.**
- **Логирование:** с контекстом, агрегация, `slog`.
- **Worker pool:** `Result{Err}` + `errors.Join`.
- **Pipeline:** `Message{Err}` + `context`.
- **Метрики:** total, success, failed, panics.

---

## 25.12 Вопросы для самопроверки

1. Почему ошибки в конкурентном коде сложнее?
2. Назови пять проблем обработки ошибок.
3. Что такое `errgroup`? Как использовать?
4. Что делает `errgroup.WithContext`?
5. Что делает `errgroup.SetLimit`?
6. Как собрать все ошибки?
7. Что такое `errors.Join`? `multierror`?
8. Как отменить работу при первой ошибке?
9. Почему `context.Canceled` — не ошибка?
10. Как обрабатывать паники в горутинах?

---

## 25.13 Ответы

### Ответ 1

**Сложнее, потому что:**
- Гонка при записи ошибки.
- Потеря ошибок.
- Нет отмены.
- Нет контекста.
- Паника в горутине.

### Ответ 2

**Пять проблем:**
1. Гонка при записи.
2. Потеря ошибок.
3. Нет отмены.
4. Нет контекста.
5. Паника в горутине.

### Ответ 3

**`errgroup`** — группа горутин с обработкой ошибок.

```go
g, ctx := errgroup.WithContext(ctx)
g.Go(func() error {
    return process(ctx)
})
return g.Wait()
```

### Ответ 4

**`WithContext`** создаёт дочерний `ctx`. При **первой** ошибке — отменяет `ctx`. Все горутины видят `ctx.Done()` и завершаются.

### Ответ 5

**`SetLimit(N)`** — не более N горутин одновременно. `g.Go` блокируется, если лимит достигнут.

### Ответ 6

**Собрать все ошибки:**

```go
var (
    mu   sync.Mutex
    errs []error
)

// В каждой горутине:
mu.Lock()
errs = append(errs, err)
mu.Unlock()

// В конце:
return errors.Join(errs...)
```

### Ответ 7

**`errors.Join`** (Go 1.20+) — объединяет несколько ошибок в одну.

**`multierror`** — для Go < 1.20.

### Ответ 8

**Отмена при первой ошибке:**

```go
g, ctx := errgroup.WithContext(ctx)
g.Go(func() error {
    return process(ctx)
})
return g.Wait()
```

При ошибке `ctx` отменяется.

### Ответ 9

**`context.Canceled`** — не ошибка, а **отмена**. Это нормальное завершение.

```go
if errors.Is(err, context.Canceled) {
    return nil  // не считаем ошибкой
}
```

### Ответ 10

**`recover` в каждой горутине:**

```go
g.Go(func() (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    return process(ctx)
})
```

---

## 25.14 Куда идти дальше?

Мы разобрали обработку ошибок в конкурентном коде: `errgroup`, `errors.Join`, отмена, `recover`, логирование. Теперь мы умеем собирать и обрабатывать ошибки.

Но остаётся **важный вопрос**: как **корректно завершить** сервис? Что делать при `Ctrl+C`? Как дождаться завершения всех горутин?

- **Как корректно завершить сервис?** → **Глава 11: Graceful shutdown.**
- **Как тестировать конкурентный код?** → **Глава 12: Тестирование конкурентного кода.**
- **Какие анти-паттерны существуют?** → **Глава 13: Анти-паттерны.**

---

## 25.15 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **`errgroup`** | Группа горутин с ошибками | `g.Go`, `g.Wait`, `WithContext` |
| **`WithContext`** | Отмена при ошибке | Автоматически |
| **`SetLimit(N)`** | Ограничение | Не более N горутин |
| **`errors.Join`** | Сбор ошибок | Go 1.20+ |
| **`multierror`** | Сбор ошибок | Go < 1.20 |
| **`Collector`** | Структура для сбора | `Mutex` + слайс |
| **Отмена** | При первой ошибке | `errgroup.WithContext` |
| **`context.Canceled`** | Не ошибка | Отмена |
| **`context.WithTimeout`** | Таймаут | Общий |
| **`signal.NotifyContext`** | Сигнал ОС | Ctrl+C |
| **`recover`** | Паника в горутине | В каждой горутине |
| **`debug.Stack()`** | Логирование паник | Для диагностики |
| **Runtime-паники** | Не восстанавливаются | OOM, deadlock |
| **Логирование** | С контекстом | `slog`, агрегация |
| **Worker pool ошибки** | `Result{Err}` | + `errors.Join` |
| **Pipeline ошибки** | `Message{Err}` | + `context` |
| **Метрики** | total, success, failed, panics | `atomic.Int64` |

⚠️ **Ключевая идея:** Обработка ошибок в конкурентном коде сложнее, чем в последовательном. `errgroup` — стандартное решение: `g.Go`, `g.Wait`, `WithContext` для отмены, `SetLimit` для ограничения. `errors.Join` (Go 1.20+) для сбора всех ошибок. `multierror` — для старых версий. Отмена при первой ошибке через `errgroup.WithContext` + `context`. Паники в горутинах — `recover` в каждой горутине. Логирование — с контекстом, агрегация, `slog`. Worker pool и pipeline — `Result{Err}` / `Message{Err}` + `errors.Join`. Метрики: total, success, failed, panics. `context.Canceled` — не ошибка, а отмена.