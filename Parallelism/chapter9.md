# 🔀 Глава 9: Fan-in — слияние каналов

**Что вы узнаете:**
- Что такое fan-in и какую задачу он решает.
- Как построить простейший fan-in.
- Как добавить отмену через `context`.
- Как обрабатывать закрытие всех каналов.
- Как собрать результаты из нескольких источников.
- Как комбинировать fan-in с fan-out и generator.
- Как избежать утечек и deadlock в fan-in.

**После прочтения вы сможете:**
- Построить fan-in с нуля.
- Сливать результаты из нескольких каналов в один.
- Правильно закрывать выходной канал.
- Комбинировать fan-in с другими паттернами.
- Понимать, где fan-in уместен, а где — нет.

---

## Содержание

- [9.0 Пролог: несколько источников, один потребитель](#90-пролог-несколько-источников-один-потребитель)
- [9.1 Что такое fan-in](#91-что-такое-fan-in)
- [9.2 Простейший fan-in](#92-простейший-fan-in)
- [9.3 Fan-in с context](#93-fan-in-с-context)
- [9.4 Fan-in с закрытием](#94-fan-in-с-закрытием)
- [9.5 Fan-in с Result](#95-fan-in-с-result)
- [9.6 В связке с другими паттернами](#96-в-связке-с-другими-паттернами)
- [9.7 Практика Go: слияние каналов](#97-практика-go-слияние-каналов)
- [9.8 Выводы и типичные ошибки](#98-выводы-и-типичные-ошибки)
- [9.9 Для быстрого повторения](#99-для-быстрого-повторения)
- [9.10 Вопросы для самопроверки](#910-вопросы-для-самопроверки)
- [9.11 Ответы](#911-ответы)
- [9.12 Куда идти дальше?](#912-куда-идти-дальше)
- [9.13 Чек-лист](#913-чек-лист)

---

## 9.0 Пролог: несколько источников, один потребитель

У нас есть сервис, который обрабатывает заказы из двух источников: HTTP API и фоновой очереди. Логика обработки одна — создать заказ в БД, отправить уведомление, обновить метрики. Но источники разные.

Хочется написать одного потребителя:

```go
func processAll() {
    for {
        select {
        case order := <-httpOrders:
            process(order)
        case order := <-queueOrders:
            process(order)
        }
    }
}
```

Это работает: `select` берёт заказ из любого канала. Но есть проблема: логика обработки **дублируется** между case'ами. А если добавится третий источник — придётся добавлять третий case. А если четвёртый?

Хочется **объединить** несколько каналов в **один** и написать одного потребителя:

```go
merged := merge(httpOrders, queueOrders, ...)

for order := range merged {
    process(order)
}
```

Эта операция называется **fan-in** — слияние нескольких каналов в один.

> **Мост к следующим главам:** fan-in — зеркало fan-out. Fan-out распределяет работу от одного источника к нескольким обработчикам. Fan-in собирает результаты из нескольких источников в один. Вместе они — основа для worker pool (Глава 13) и pipeline (Глава 11).

---

## 9.1 Что такое fan-in

**Fan-in** — это объединение **нескольких каналов** в **один**.

### Схема

```
   ┌─────────┐
   │Channel 1│──┐
   └─────────┘  │
   ┌─────────┐  │
   │Channel 2│──┤   ┌──────────┐
   └─────────┘  ├──▶│ Combined │
   ┌─────────┐  │   └──────────┘
   │Channel 3│──┤
   └─────────┘  │
   ┌─────────┐  │
   │Channel N│──┘
   └─────────┘
```

**N каналов** объединяются в **один**. Все значения идут в один поток.

### Когда использовать fan-in

**1. Несколько источников данных.**

- HTTP API + очередь.
- Два файла.
- Две БД.

**2. Сбор результатов от нескольких воркеров.**

- Fan-out → N воркеров → fan-in → один канал.
- Классический паттерн worker pool.

**3. Слияние pipeline-стадий.**

- Несколько pipeline'ов идут в один потребитель.

### Когда НЕ использовать fan-in

**1. Один канал.**

Если канал один — fan-in не нужен. Просто `for v := range ch`.

**2. Разная обработка.**

Если разные источники требуют **разной** обработки — не сливай их. Иначе потеряешь контекст «откуда пришло».

**3. Нужен порядок.**

Fan-in **не сохраняет порядок**. Значения приходят в порядке готовности.

### Ключевое свойство

**Порядок не гарантирован.** Значения из разных каналов приходят **в порядке готовности**. Если важен порядок — нужно дополнительное упорядочивание (например, по ID).

### 💡 Практика: как думать о fan-in

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Для слияния — fan-in.**
2. **Помни: порядок не гарантирован.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Проверяй `Result{Value, SourceID}`** — если важно знать источник.

**❌ НЕ ДЕЛАЙ:**

4. **Не сливай каналы с разной обработкой.**
5. **Не полагайся на порядок.**

---

## 9.2 Простейший fan-in

Начнём с самого простого — слияние двух каналов.

### Наивная попытка

Первое, что приходит в голову — читать из двух каналов **по очереди**:

```go
func merge(ch1, ch2 <-chan int) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        for v := range ch1 {
            out <- v
        }
        for v := range ch2 {
            out <- v
        }
    }()
    
    return out
}
```

**Проблема:** если `ch1` долго не закрывается, `ch2` не читается. **Нет параллелизма.**

**Что происходит:**

```
t=0:    Читаем из ch1
        ch1 отдаёт 1, 2, 3, ...
        ch2 ждёт (никто не читает)

t=T:    ch1 закрылся
        Начинаем читать ch2
        ch2 отдаёт 6, 7, 8, ...
```

**Результат:** значения из `ch2` идут **после** всех значений из `ch1`.

### Правильный fan-in: горутина на каждый канал

**Идея:** для **каждого** входного канала — **своя** горутина. Каждая горутина читает из своего канала и пишет в **общий** выходной.

```go
func merge(ch1, ch2 <-chan int) <-chan int {
    out := make(chan int)
    
    var wg sync.WaitGroup
    
    // Горутина для ch1
    wg.Add(1)
    go func() {
        defer wg.Done()
        for v := range ch1 {
            out <- v
        }
    }()
    
    // Горутина для ch2
    wg.Add(1)
    go func() {
        defer wg.Done()
        for v := range ch2 {
            out <- v
        }
    }()
    
    // Закрытие out после всех горутин
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

**Что происходит:**

- `ch1` читается горутиной 1.
- `ch2` читается горутиной 2.
- Обе пишут в `out`.
- Отдельная горутина ждёт завершения обеих и закрывает `out`.

### Универсальный fan-in для N каналов

```go
func merge(channels ...<-chan int) <-chan int {
    out := make(chan int)
    
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

**Ключевое:** `ch` передаётся как **аргумент** (`c <-chan int`), а не захватывается замыканием. Иначе все горутины читали бы из **последнего** `ch`.

### Потребитель

```go
func main() {
    ch1 := make(chan int, 5)
    ch2 := make(chan int, 5)
    
    for i := 1; i <= 5; i++ {
        ch1 <- i
    }
    close(ch1)
    
    for i := 6; i <= 10; i++ {
        ch2 <- i
    }
    close(ch2)
    
    for v := range merge(ch1, ch2) {
        fmt.Println(v)
    }
}
```

**Пример вывода:**

```
1
6
2
7
3
8
4
9
5
10
```

**Что видно:** значения из `ch1` и `ch2` **перемешаны**. Порядок не гарантирован.

### Схема

```
ch1 ──┐
      ├──► goroutine 1 ──┐
      │                   │
      │                   ▼
      │                ┌─────┐
      │                │ out │
      │                └─────┘
      │                   ▲
      │                   │
ch2 ──┤                   │
      │                   │
      └──► goroutine 2 ──┘
```

### 💡 Практика: как писать простой fan-in

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Горутина на каждый канал.**
2. **`ch` передавай как аргумент**, не замыкание.
3. **`wg.Wait()` + `close(out)` в отдельной горутине.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`merge(channels ...<-chan T)` — универсальная сигнатура.**

**❌ НЕ ДЕЛАЙ:**

5. **Не читай каналы последовательно** — нет параллелизма.
6. **Не забывай `close(out)`** — потребитель зависнет.

---

## 9.3 Fan-in с context

Простейший fan-in не умеет останавливаться. Если потребитель перестал читать — горутины зависнут.

### Проблема

```go
func main() {
    ch1 := numbers(1_000_000)
    ch2 := numbers(1_000_000)
    
    for v := range merge(ch1, ch2) {
        if v > 10 {
            break  // ← вышли
        }
        fmt.Println(v)
    }
    // Горутины merge зависли на out <- v
}
```

**Что происходит:**

- `main` вышел из цикла.
- Горутины merge продолжают писать в `out`.
- `out` небуферизованный — горутины блокируются.
- **Утечка.**

### Решение: context

```go
func mergeCtx(ctx context.Context, channels ...<-chan int) <-chan int {
    out := make(chan int)
    
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                select {
                case out <- v:
                case <-ctx.Done():
                    return
                }
            }
        }(ch)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

**Что изменилось:** каждая отправка в `out` проходит через `select`. Если `ctx` отменён — горутина завершается.

### Потребитель

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    ch1 := numbersCtx(ctx, 1_000_000)
    ch2 := numbersCtx(ctx, 1_000_000)
    
    for v := range mergeCtx(ctx, ch1, ch2) {
        if v > 10 {
            cancel()  // отменяем всех
            break
        }
        fmt.Println(v)
    }
    
    time.Sleep(10 * time.Millisecond)
}
```

**Что происходит:**

- При `v > 10` вызываем `cancel()`.
- Все горутины merge видят `ctx.Done()` и завершаются.
- `out` закрывается.
- Утечки нет.

### Полный код

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

func numbersCtx(ctx context.Context, n int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for i := 0; i < n; i++ {
            select {
            case out <- i:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func mergeCtx(ctx context.Context, channels ...<-chan int) <-chan int {
    out := make(chan int)
    
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                select {
                case out <- v:
                case <-ctx.Done():
                    return
                }
            }
        }(ch)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    ch1 := numbersCtx(ctx, 1_000_000)
    ch2 := numbersCtx(ctx, 1_000_000)
    
    for v := range mergeCtx(ctx, ch1, ch2) {
        if v > 10 {
            cancel()
            break
        }
        fmt.Println(v)
    }
    
    time.Sleep(10 * time.Millisecond)
    fmt.Println("done")
}
```

### Схема

```
mergeCtx(ctx, ch1, ch2):

  goroutine 1:                    goroutine 2:
    for v := range ch1 {            for v := range ch2 {
      select {                        select {
      case out <- v:                  case out <- v:
      case <-ctx.Done():              case <-ctx.Done():
        return                          return
      }                               }
    }                               }
    
  Потребитель:
    cancel() ─────► ctx.Done() закрыт
                    │
                    ▼
              все горутины завершаются
```

### 💡 Практика: как добавить context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx context.Context` — первый аргумент:**
   ```go
   func mergeCtx(ctx context.Context, channels ...<-chan T) <-chan T
   ```

2. **`select` с `ctx.Done()` при отправке.**

3. **`defer cancel()` у потребителя.**

**❌ НЕ ДЕЛАЙ:**

4. **Не пиши в `out` без `select`** в долгоживущем fan-in.

---

## 9.4 Fan-in с закрытием

Правильное закрытие `out` — критично. Разберём **почему** и **как**.

### Как закрывается out

```go
go func() {
    wg.Wait()
    close(out)
}()
```

**Что происходит:**

- Отдельная горутина ждёт `wg.Wait()`.
- Когда **все** горутины-читатели завершились — `close(out)`.

### Почему отдельная горутина

**❌ Плохо:**

```go
func merge(channels ...<-chan int) <-chan int {
    out := make(chan int)
    
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }
    
    wg.Wait()  // ← БЛОКИРУЕТСЯ здесь
    close(out)
    return out  // ← никогда не выполнится
}
```

**Что происходит:**

- `wg.Wait()` блокируется.
- Горутины пишут в `out`.
- `close(out)` не выполняется.
- Функция не возвращает `out`.
- **Deadlock.**

**✅ Хорошо:**

```go
go func() {
    wg.Wait()
    close(out)
}()
return out  // ← возвращаемся сразу
```

**Что происходит:**

- `merge` возвращает `out` сразу.
- Потребитель читает из `out`.
- Отдельная горутина закрывает `out`, когда все горутины завершились.

### Почему нельзя закрывать в горутине-читателе

**❌ Плохо:**

```go
for _, ch := range channels {
    wg.Add(1)
    go func(c <-chan int) {
        defer wg.Done()
        defer close(out)  // ← НЕЛЬЗЯ!
        for v := range c {
            out <- v
        }
    }(ch)
}
```

**Что происходит:**

- Каждая горутина закрывает `out`.
- Вторая горутина закрывает уже закрытый канал.
- **Паника: close of closed channel.**

### Схема закрытия

```
ДО ЗАКРЫТИЯ:

  goroutine 1 ──пишет в out──┐
                              │
  goroutine 2 ──пишет в out──┤
                              │
  goroutine 3 ──пишет в out──┤
                              ▼
                          ┌─────┐
                          │ out │
                          └─────┘

ВСЕ ГОРУТИНЫ ЗАВЕРШИЛИСЬ:

  goroutine 1 ──завершилась
  goroutine 2 ──завершилась
  goroutine 3 ──завершилась
  
  Отдельная горутина:
    wg.Wait() ──► возвращается
    close(out) ──► закрывает канал
```

### Что делать при закрытии входных каналов

**Если входной канал закрыт** — `for v := range c` завершится автоматически. Горутина вызовет `wg.Done()`. Когда все горутины завершатся — `close(out)`.

**Если часть каналов закрыта, а часть — нет** — `merge` ждёт **все**. Пока хотя бы один канал открыт — `out` не закроется.

### 💡 Практика: как закрывать out

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Отдельная горутина для `wg.Wait()` + `close(out)`.**
2. **`defer wg.Done()` в каждой горутине-читателе.**

**❌ НЕ ДЕЛАЙ:**

3. **Не закрывай `out` в горутине-читателе.**
4. **Не `wg.Wait()` в основной горутине** — deadlock.

---

## 9.5 Fan-in с Result

Иногда важно знать, **откуда** пришло значение. Или обработать ошибку из конкретного источника.

### Проблема: разные источники

```go
func merge(ch1, ch2 <-chan int) <-chan int {
    // ...
}
```

**Что теряется:** мы не знаем, из какого канала пришло значение.

### Решение: Result с SourceID

```go
type Result struct {
    SourceID string
    Value    int
    Err      error
}

func mergeWithSource(ch1, ch2 <-chan int) <-chan Result {
    out := make(chan Result)
    
    var wg sync.WaitGroup
    wg.Add(2)
    
    go func() {
        defer wg.Done()
        for v := range ch1 {
            out <- Result{SourceID: "ch1", Value: v}
        }
    }()
    
    go func() {
        defer wg.Done()
        for v := range ch2 {
            out <- Result{SourceID: "ch2", Value: v}
        }
    }()
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

**Что происходит:**

- Каждое значение оборачивается в `Result`.
- `SourceID` показывает, откуда пришло.

### Потребитель

```go
for r := range mergeWithSource(ch1, ch2) {
    if r.Err != nil {
        log.Printf("[%s] error: %v", r.SourceID, r.Err)
        continue
    }
    fmt.Printf("[%s] %d\n", r.SourceID, r.Value)
}
```

**Пример вывода:**

```
[ch1] 1
[ch2] 6
[ch1] 2
[ch2] 7
...
```

### Обработка ошибок

Если каждый источник может вернуть ошибку — включаем `Err` в `Result`:

```go
func mergeWithSource(ch1, ch2 <-chan Result) <-chan Result {
    out := make(chan Result)
    
    var wg sync.WaitGroup
    wg.Add(2)
    
    go func() {
        defer wg.Done()
        for r := range ch1 {
            out <- r  // уже Result с Err
        }
    }()
    
    go func() {
        defer wg.Done()
        for r := range ch2 {
            out <- r
        }
    }()
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

### Схема

```
ch1 ──► goroutine 1 ──► Result{Source: "ch1", Value, Err} ──┐
                                                              │
                                                              ▼
                                                          ┌─────┐
                                                          │ out │
                                                          └─────┘
                                                              ▲
                                                              │
ch2 ──► goroutine 2 ──► Result{Source: "ch2", Value, Err} ──┘
```

### 💡 Практика: как использовать Result в fan-in

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`Result{SourceID, Value, Err}`** для сложных случаев.
2. **Проверяй `Err` в потребителе.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **`SourceID` — строковый или числовой.**

**❌ НЕ ДЕЛАЙ:**

4. **Не сливай каналы с разной обработкой** — потеряется контекст.

---

## 9.6 В связке с другими паттернами

Fan-in — центральный паттерн для сбора результатов. Разберём связки.

### Fan-out + fan-in

**Классическая связка:** распределить работу между воркерами, собрать результаты.

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 100)
    
    // Fan-out: 5 воркеров
    outputs := make([]<-chan int, 5)
    for i := 0; i < 5; i++ {
        outputs[i] = worker(ctx, source)
    }
    
    // Fan-in: слияние
    merged := mergeCtx(ctx, outputs...)
    
    // Потребитель
    for v := range merged {
        fmt.Println(v)
    }
}

func worker(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range in {
            select {
            case out <- v * 2:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

**Что происходит:**

- Generator → source.
- 5 воркеров читают из source (fan-out).
- `mergeCtx` сливает выходы воркеров (fan-in).
- Потребитель читает из одного канала.

**Это классический worker pool.**

### Generator + fan-in

**Несколько generator'ов сливаются в один:**

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    ch1 := numbersCtx(ctx, 5)
    ch2 := numbersCtx(ctx, 5)
    ch3 := numbersCtx(ctx, 5)
    
    for v := range mergeCtx(ctx, ch1, ch2, ch3) {
        fmt.Println(v)
    }
}
```

**Что происходит:** три generator'а → один потребитель.

### Fan-in + pipeline

**Fan-in как стадия pipeline:**

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source1 := numbersCtx(ctx, 10)
    source2 := numbersCtx(ctx, 10)
    
    merged := mergeCtx(ctx, source1, source2)
    doubled := doubleStage(ctx, merged)
    filtered := filterEvenStage(ctx, doubled)
    
    for v := range filtered {
        fmt.Println(v)
    }
}

func doubleStage(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range in {
            select {
            case out <- v * 2:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func filterEvenStage(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range in {
            if v%2 == 0 {
                select {
                case out <- v:
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    return out
}
```

**Схема:**

```
Generator 1 ──┐
              ├──► merge ──► double ──► filter ──► Потребитель
Generator 2 ──┘
```

### Fan-in + rate limiter

**Слияние с ограничением скорости:**

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    ch1 := numbersCtx(ctx, 100)
    ch2 := numbersCtx(ctx, 100)
    merged := mergeCtx(ctx, ch1, ch2)
    
    limiter := rate.NewLimiter(rate.Limit(10), 5)
    
    for v := range merged {
        if err := limiter.Wait(ctx); err != nil {
            break
        }
        fmt.Println(v)
    }
}
```

**Что происходит:** значения сливаются, но обработка ограничена 10/сек.

### 💡 Практика: как комбинировать fan-in

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Fan-out + fan-in — классический worker pool.**
2. **Generator + fan-in — несколько источников.**
3. **Fan-in + pipeline — стадия слияния.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`Result{SourceID, ...}` — если важен источник.**

**❌ НЕ ДЕЛАЙ:**

5. **Не смешивай fan-in с разной обработкой.**
6. **Не забывай `context`.**

---

## 9.7 Практика Go: слияние каналов

Разберём **три примера**.

### Пример 1: слияние двух каналов

```go
package main

import (
    "fmt"
    "sync"
)

func makeChan(data []int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, v := range data {
            out <- v
        }
    }()
    return out
}

func merge(channels ...<-chan int) <-chan int {
    out := make(chan int)
    
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}

func main() {
    ch1 := makeChan([]int{1, 2, 3, 4, 5})
    ch2 := makeChan([]int{6, 7, 8, 9, 10})
    
    for v := range merge(ch1, ch2) {
        fmt.Println(v)
    }
}
```

**Пример вывода:**

```
1
6
2
7
3
8
4
9
5
10
```

### Пример 2: fan-in с context

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

func numbersCtx(ctx context.Context, n int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for i := 0; i < n; i++ {
            select {
            case out <- i:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func mergeCtx(ctx context.Context, channels ...<-chan int) <-chan int {
    out := make(chan int)
    
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                select {
                case out <- v:
                case <-ctx.Done():
                    return
                }
            }
        }(ch)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    ch1 := numbersCtx(ctx, 1_000_000)
    ch2 := numbersCtx(ctx, 1_000_000)
    
    count := 0
    for v := range mergeCtx(ctx, ch1, ch2) {
        if count >= 10 {
            cancel()
            break
        }
        fmt.Println(v)
        count++
    }
    
    time.Sleep(10 * time.Millisecond)
    fmt.Println("done")
}
```

**Что демонстрирует:** отмена через `context`. После 10 значений — `cancel()`.

### Пример 3: fan-out + fan-in (worker pool)

```go
package main

import (
    "context"
    "fmt"
    "sync"
)

func numbersCtx(ctx context.Context, n int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for i := 0; i < n; i++ {
            select {
            case out <- i:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func worker(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range in {
            select {
            case out <- v * 2:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func mergeCtx(ctx context.Context, channels ...<-chan int) <-chan int {
    out := make(chan int)
    
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                select {
                case out <- v:
                case <-ctx.Done():
                    return
                }
            }
        }(ch)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Источник
    source := numbersCtx(ctx, 20)
    
    // Fan-out: 5 воркеров
    const workers = 5
    outputs := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        outputs[i] = worker(ctx, source)
    }
    
    // Fan-in
    merged := mergeCtx(ctx, outputs...)
    
    // Потребитель
    for v := range merged {
        fmt.Println(v)
    }
    
    fmt.Println("done")
}
```

**Пример вывода:**

```
0
2
6
4
10
8
14
12
18
16
...
```

**Что видно:** значения удвоены воркерами, порядок не гарантирован. Это классический worker pool через fan-out + fan-in.

### 💡 Практика: как писать fan-in

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Горутина на каждый канал.**
2. **`ch` передавай как аргумент.**
3. **`wg.Wait()` + `close(out)` в отдельной горутине.**
4. **`select` с `ctx.Done()` при отправке.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **`Result{SourceID, ...}` для сложных случаев.**
6. **Fan-out + fan-in — для worker pool.**

**❌ НЕ ДЕЛАЙ:**

7. **Не читай каналы последовательно.**
8. **Не забывай `close`.**
9. **Не закрывай `out` в горутине-читателе.**

---

## 9.8 Выводы и типичные ошибки

**Что мы узнали?**

Fan-in — слияние N каналов в один. Простейшая реализация — горутина на каждый канал + общий выход + `wg.Wait()` + `close(out)`. `context` для отмены. `Result{SourceID, Value, Err}` для сложных случаев. Fan-in — центральный паттерн для сборки результатов. Fan-out + fan-in — классический worker pool. Generator + fan-in — несколько источников. Fan-in + pipeline — стадия слияния.

**Типичные ошибки:**

- ❌ **Читать каналы последовательно.** Нет параллелизма.
- ❌ **`ch` в замыкании, а не аргументом.** Все горутины читают из последнего канала.
- ❌ **Забыть `close(out)`.** Потребитель зависнет.
- ❌ **`close(out)` в горутине-читателе.** Паника.
- ❌ **`wg.Wait()` в основной горутине.** Deadlock.
- ❌ **Не использовать `context`.** Утечка при отмене.
- ❌ **Полагать, что порядок сохранится.** Порядок не гарантирован.
- ❌ **Сливать каналы с разной обработкой.** Потеряется контекст.

---

## 9.9 Для быстрого повторения

- **Fan-in** — слияние N каналов в один.
- **Горутина на каждый канал** — параллельное чтение.
- **`ch` передавай как аргумент**, не замыкание.
- **`wg.Wait()` + `close(out)` в отдельной горутине.**
- **Не закрывай `out` в горутине-читателе.**
- **`select` с `ctx.Done()`** — для отмены.
- **`Result{SourceID, Value, Err}`** — если важен источник.
- **Порядок не гарантирован.**
- **Fan-out + fan-in** — worker pool.
- **Generator + fan-in** — несколько источников.
- **Fan-in + pipeline** — стадия слияния.

---

## 9.10 Вопросы для самопроверки

1. Что такое fan-in? Какую задачу решает?
2. Как построить простейший fan-in?
3. Почему `ch` нужно передавать как аргумент, а не замыкание?
4. Почему `close(out)` в отдельной горутине?
5. Зачем `context` в fan-in?
6. Что теряется при слиянии каналов без `Result`?
7. Как комбинировать fan-out + fan-in?

---

## 9.11 Ответы

### Ответ 1

**Fan-in** — слияние N каналов в один. Решает задачу: **несколько источников — один потребитель**.

### Ответ 2

```go
func merge(channels ...<-chan int) <-chan int {
    out := make(chan int)
    
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

### Ответ 3

**Без аргумента:** все горутины захватят **последнее** значение `ch` из цикла. Все будут читать из одного канала.

**С аргументом:** каждая горутина получает **копию** `ch` — своё значение.

### Ответ 4

**`wg.Wait()` блокируется.** Если `close(out)` в основной горутине — она не вернёт `out`. Потребитель не сможет читать. **Deadlock.**

**Отдельная горутина:** основная горутина возвращает `out` сразу, отдельная ждёт завершения и закрывает.

### Ответ 5

**`context`** позволяет остановить fan-in досрочно. Без него, если потребитель перестал читать, горутины **зависнут** на `out <- v`. **Утечка.**

### Ответ 6

**Без `Result`:** теряется информация, **из какого канала** пришло значение. Нельзя обработать ошибку конкретного источника.

**С `Result{SourceID, Value, Err}`:** потребитель знает источник.

### Ответ 7

**Fan-out + fan-in — worker pool:**

```go
source := generator(ctx)
outputs := make([]<-chan T, workers)
for i := 0; i < workers; i++ {
    outputs[i] = worker(ctx, source)
}
merged := merge(ctx, outputs...)
for v := range merged {
    process(v)
}
```

Generator → N воркеров (fan-out) → merge (fan-in) → потребитель.

---

## 9.12 Куда идти дальше?

Мы разобрали fan-in — слияние каналов. Теперь мы умеем собирать результаты из нескольких источников.

Но остаётся вопрос: как **распределить работу** от одного источника между несколькими обработчиками?

- **Как распределить работу?** → **Глава 10: Fan-out — распределение работы.**
- **Как построить конвейер?** → **Глава 11: Pipeline.**
- **Как ограничить параллелизм?** → **Глава 12: Semaphore.**
- **Как построить worker pool?** → **Глава 13: Worker pool.**

---

## 9.13 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Fan-in** | Слияние N каналов | Один выходной канал |
| **Горутина на каждый канал** | Параллельное чтение | `for _, ch := range channels` |
| **`ch` как аргумент** | Не замыкание | Иначе все читают из последнего |
| **`wg.Wait()` + `close(out)`** | В отдельной горутине | Иначе deadlock |
| **`select` с `ctx.Done()`** | Отмена | При отправке |
| **`Result{SourceID, ...}`** | Знать источник | Для сложных случаев |
| **Порядок не гарантирован** | Приходит в порядке готовности | Не полагайся |
| **Fan-out + fan-in** | Worker pool | Классика |
| **Generator + fan-in** | Несколько источников | N generator'ов |
| **Fan-in + pipeline** | Стадия слияния | В конвейере |

🔀 **Ключевая идея:** Fan-in — слияние N каналов в один. Горутина на каждый канал + общий выход + `wg.Wait()` + `close(out)`. `ch` передавай как аргумент, не замыкание. `context` — для отмены. `Result{SourceID, Value, Err}` — если важен источник. Порядок не гарантирован — приходит в порядке готовности. Fan-out + fan-in — классический worker pool. Generator + fan-in — несколько источников. Fan-in + pipeline — стадия слияния. Не забывай `close(out)` в отдельной горутине и `select` с `ctx.Done()`.