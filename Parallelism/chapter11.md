# 🔗 Глава 11: Pipeline — конвейеры данных

**Что вы узнаете:**
- Что такое pipeline и какую задачу он решает.
- Как построить простейший pipeline.
- Как добавить отмену через `context`.
- Как работает backpressure между стадиями.
- Как добавлять fan-out и fan-in в стадии pipeline.
- Как правильно закрывать каналы между стадиями.
- Как комбинировать pipeline с generator, worker pool и rate limiter.

**После прочтения вы сможете:**
- Построить pipeline с нуля.
- Разбивать обработку на стадии.
- Понимать, где backpressure срабатывает автоматически.
- Правильно закрывать каналы между стадиями.
- Добавлять fan-out для медленных стадий.
- Комбинировать pipeline с другими паттернами.

---

## Содержание

- [11.0 Пролог: цепочка обработки](#110-пролог-цепочка-обработки)
- [11.1 Что такое pipeline](#111-что-такое-pipeline)
- [11.2 Простейший pipeline](#112-простейший-pipeline)
- [11.3 Pipeline с context](#113-pipeline-с-context)
- [11.4 Backpressure между стадиями](#114-backpressure-между-стадиями)
- [11.5 Закрытие каналов в pipeline](#115-закрытие-каналов-в-pipeline)
- [11.6 Fan-out в стадии pipeline](#116-fan-out-в-стадии-pipeline)
- [11.7 В связке с другими паттернами](#117-в-связке-с-другими-паттернами)
- [11.8 Практика Go: конвейеры](#118-практика-go-конвейеры)
- [11.9 Выводы и типичные ошибки](#119-выводы-и-типичные-ошибки)
- [11.10 Для быстрого повторения](#1110-для-быстрого-повторения)
- [11.11 Вопросы для самопроверки](#1111-вопросы-для-самопроверки)
- [11.12 Ответы](#1112-ответы)
- [11.13 Куда идти дальше?](#1113-куда-идти-дальше)
- [11.14 Чек-лист](#1114-чек-лист)

---

## 11.0 Пролог: цепочка обработки

У нас есть сервис, который обрабатывает логи. Логи приходят в сыром виде, их нужно: **распарсить**, **обогатить** данными из БД и **записать** в ClickHouse.

Простейшая реализация — последовательно:

```go
func processAll(logs <-chan RawLog) error {
    for raw := range logs {
        parsed, err := parse(raw)
        if err != nil {
            continue
        }
        
        enriched, err := enrich(parsed)
        if err != nil {
            continue
        }
        
        if err := writeToClickHouse(enriched); err != nil {
            continue
        }
    }
    return nil
}
```

Работает. Но заметна странность в метриках: **parse — быстро** (1 мкс), **enrich — медленно** (10 мс, запрос в БД), **write — быстро** (100 мкс). При этом пропускная способность всего **100 логов в секунду** — потому что enrich узкое место.

Хочется **ускорить enrich**: запустить 10 параллельных обработчиков. Но тогда придётся делать fan-out внутри цикла. А ещё хочется добавить метрики по стадиям, обработку ошибок, отмену по сигналу.

Код растёт. Смешивается: parse, enrich, write, fan-out, метрики. Сложно тестировать.

Хочется **разбить** обработку на **стадии**: parse — отдельно, enrich — отдельно, write — отдельно. Каждая стадия — отдельная функция, читающая из одного канала и пишущая в другой. Между ними — каналы. Такая цепочка называется **pipeline**.

> **Мост к следующим главам:** pipeline — обобщение generator (Глава 8), fan-in (Глава 9), fan-out (Глава 10). Каждая стадия pipeline — это generator + fan-out + fan-in в одном. Понимание pipeline даёт понимание, как строить сложные конвейеры данных.

---

## 11.1 Что такое pipeline

**Pipeline** — это **цепочка стадий**, где каждая стадия:

1. Читает из **входного** канала.
2. Обрабатывает данные.
3. Пишет в **выходной** канал.

### Схема

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Source  │──▶│  Stage 1 │──▶│  Stage 2 │──▶│  Stage 3 │──▶ Sink
└──────────┘   └──────────┘   └──────────┘   └──────────┘
   (logs)        (parse)        (enrich)      (write)
```

**Каждая стадия** — отдельная горутина. Между стадиями — каналы.

### Ключевые свойства

1. **Каждая стадия — независима.** Своя горутина, свои каналы.
2. **Backpressure автоматический.** Если следующая стадия медленная, входной канал заполняется, предыдущая стадия блокируется.
3. **Отмена — через `context`.** Все стадии получают один `ctx`.
4. **Масштабирование — через fan-out.** Медленная стадия может иметь N воркеров.

### Сигнатура стадии

```go
func stage(ctx context.Context, input <-chan Input) <-chan Output
```

**Что делает:**

- Создаёт выходной канал.
- Запускает горутину.
- Читает из входа, обрабатывает, пишет в выход.
- Закрывает выход при завершении.

### Когда использовать pipeline

**1. Последовательная обработка в несколько шагов.**

- ETL: extract → transform → load.
- Логи: parse → enrich → write.
- Заказы: validate → process → notify.

**2. Разные типы обработки.**

- Каждая стадия делает **своё**.
- Каждая стадия может иметь **своё** число воркеров.

**3. Нужны метрики по стадиям.**

- Легко измерять пропускную способность и latency каждой стадии.

### Когда НЕ использовать pipeline

**1. Одна стадия.**

Если обработка — одна операция, не нужен pipeline. Просто generator.

**2. Задачи зависимы.**

Если задача B требует результата A — fan-out не подойдёт, нужна **упорядоченная** обработка.

**3. Простые задачи.**

Для маленьких задач overhead каналов может быть заметен.

### 💡 Практика: как думать о pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Каждая стадия — функция** `func(ctx, input) <-chan Output`.
2. **`context` — через все стадии.**
3. **`close(out)` в каждой стадии.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Fan-out для медленных стадий.**
5. **Метрики на каждой стадии.**

**❌ НЕ ДЕЛАЙ:**

6. **Не смешивай несколько стадий в одной функции.**
7. **Не используй pipeline для одной стадии.**

---

## 11.2 Простейший pipeline

Начнём с трёх стадий: **умножение на 2**, **фильтр чётных**, **печать**.

### Стадия 1: умножить на 2

```go
func doubleStage(ctx context.Context, input <-chan int) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        for v := range input {
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

### Стадия 2: фильтр чётных

```go
func filterEvenStage(ctx context.Context, input <-chan int) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        for v := range input {
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

### Стадия 3: печать

```go
func printStage(ctx context.Context, input <-chan int) {
    for v := range input {
        select {
        case <-ctx.Done():
            return
        default:
            fmt.Println(v)
        }
    }
}
```

### Сборка

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 10)
    doubled := doubleStage(ctx, source)
    filtered := filterEvenStage(ctx, doubled)
    printStage(ctx, filtered)
}
```

### Полный код

```go
package main

import (
    "context"
    "fmt"
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

func doubleStage(ctx context.Context, input <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range input {
            select {
            case out <- v * 2:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func filterEvenStage(ctx context.Context, input <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range input {
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

func printStage(ctx context.Context, input <-chan int) {
    for v := range input {
        select {
        case <-ctx.Done():
            return
        default:
            fmt.Println(v)
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 10)
    doubled := doubleStage(ctx, source)
    filtered := filterEvenStage(ctx, doubled)
    printStage(ctx, filtered)
}
```

**Пример вывода:**

```
0
2
4
6
8
10
12
14
16
18
```

**Что видно:** pipeline прошёл через 3 стадии. Умножение, фильтр, печать.

### Схема

```
numbersCtx ──▶ doubleStage ──▶ filterEvenStage ──▶ printStage
   (0..9)        (x2)            (чётные)           (печать)
```

### 💡 Практика: как писать простой pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Каждая стадия — функция** `func(ctx, input) <-chan Output`.
2. **`defer close(out)`** сразу после `make(chan)`.
3. **`context` — первый аргумент.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Явно разделяй логику по стадиям.**

**❌ НЕ ДЕЛАЙ:**

5. **Не смешивай несколько стадий в одной функции.**
6. **Не забывай `close(out)`.**

---

## 11.3 Pipeline с context

`context` — критичен для pipeline. Он позволяет **остановить** все стадии одновременно.

### Что происходит без context

```go
func main() {
    source := numbers(1_000_000)
    doubled := doubleStage(source)
    filtered := filterEvenStage(doubled)
    
    // Читаем 10 значений и выходим
    count := 0
    for v := range filtered {
        if count >= 10 {
            break
        }
        fmt.Println(v)
        count++
    }
    // Все стадии зависли
}
```

**Что происходит:**

- `main` вышел из цикла.
- Все стадии продолжают работать.
- Одна из стадий **заблокируется** на отправке в канал.
- **Утечка горутин.**

### Решение: context

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 1_000_000)
    doubled := doubleStage(ctx, source)
    filtered := filterEvenStage(ctx, doubled)
    
    count := 0
    for v := range filtered {
        if count >= 10 {
            cancel()
            break
        }
        fmt.Println(v)
        count++
    }
    
    time.Sleep(10 * time.Millisecond)
}
```

**Что происходит:**

- При `count >= 10` вызывается `cancel()`.
- Все стадии видят `ctx.Done()` и завершаются.
- `close(out)` закрывает каналы.
- Утечки нет.

### Как context распространяется

```
Потребитель вызывает cancel()
    │
    ▼
ctx.Done() закрывается
    │
    ▼
Все стадии видят ctx.Done()
    │
    ├─── Stage 1: return
    ├─── Stage 2: return
    └─── Stage 3: return
    │
    ▼
close(out) в каждой стадии
    │
    ▼
Все каналы закрыты, все горутины завершены
```

**Ключевое:** `ctx.Done()` — **broadcast**. Все стадии видят отмену **одновременно**.

### Схема

```
numbersCtx ──▶ doubleStage ──▶ filterEvenStage ──▶ printStage
   ▲              ▲                 ▲                  ▲
   │              │                 │                  │
   └──────────────┴─────────────────┴──────────────────┘
                        ctx
                        │
                    cancel()
```

Все стадии получают **один** `ctx`.

### 💡 Практика: как использовать context в pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx` — первый аргумент** каждой стадии.
2. **`select` с `ctx.Done()`** при отправке.
3. **`defer cancel()`** у потребителя.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`ctx` — общий для всех стадий.**

**❌ НЕ ДЕЛАЙ:**

5. **Не создавай новый `ctx` в каждой стадии.**
6. **Не забывай `cancel()`.**

---

## 11.4 Backpressure между стадиями

Backpressure в pipeline работает **автоматически** через каналы.

### Как это работает

```
Source (быстро) → parsed (буфер) → enrich (медленно)
```

**Что происходит:**

1. Source пишет в `parsed` быстро.
2. `parsed` заполняется (буфер).
3. Source **блокируется** на `parsed <- msg`.
4. Source ждёт, пока enrich не заберёт.
5. Enrich обрабатывает медленно → `parsed` остаётся полным.
6. Source продолжает ждать.

**Результат:** Source работает со скоростью enrich.

### Пример: быстрый source, медленный enrich

```go
func fastSource(ctx context.Context) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for i := 0; i < 1000; i++ {
            select {
            case out <- i:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func slowStage(ctx context.Context, input <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range input {
            time.Sleep(100 * time.Millisecond)  // ← медленно
            select {
            case out <- v:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    start := time.Now()
    source := fastSource(ctx)
    slow := slowStage(ctx, source)
    
    for range slow {
        fmt.Printf("processed in %v\n", time.Since(start))
    }
}
```

**Что видно:** source генерирует мгновенно, но обрабатывается по 100 мс. Source **ждёт**.

### Размер буфера

**Маленький буфер (0–10):**

- Backpressure быстрый.
- Source блокируется часто.
- Меньше памяти.

**Большой буфер (1000):**

- Backpressure отложенный.
- Source блокируется редко.
- Больше памяти.

**Очень большой буфер (100 000):**

- Backpressure не работает.
- Память растёт.

### Как выбирать буфер

| Сценарий | Буфер |
|:---|:---|
| Важен backpressure | 0 (небуферизованный) |
| Сгладить пики | 10–100 |
| Большой запас | 1000+ |
| Огромный запас | ❌ Маскирует проблему |

**Рекомендация:** начинай с небуферизованного. Если нужен «запас» — добавляй буфер 10–100.

### 💡 Практика: как использовать backpressure

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Небуферизованный канал** — для автоматического backpressure.
2. **Буфер 10–100** — если нужен небольшой запас.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Мониторь длину буфера** — если растёт, потребитель медленный.

**❌ НЕ ДЕЛАЙ:**

4. **Не ставь буфер 100 000+.** Backpressure не работает.
5. **Не игнорируй растущий буфер.**

---

## 11.5 Закрытие каналов в pipeline

Правильное закрытие каналов в pipeline — критично.

### Правило: кто пишет — тот закрывает

**В pipeline** каждая стадия:

- **Читает** из входного канала.
- **Пишет** в выходной канал.
- **Закрывает** свой **выходной** канал.

**Не закрывает входной** — это делает предыдущая стадия.

### Пример

```go
func stage(ctx context.Context, input <-chan int) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)  // ← закрываем СВОЙ выход
        for v := range input {  // ← читаем из чужого входа
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

### Каскадное закрытие

```
Source закрывает source
    │
    ▼
Stage 1: for range source завершается → close(stage1Out)
    │
    ▼
Stage 2: for range stage1Out завершается → close(stage2Out)
    │
    ▼
Stage 3: for range stage2Out завершается → close(stage3Out)
    │
    ▼
Sink: for range stage3Out завершается
```

**Ключевое:** закрытие **каскадное**. Каждая стадия закрывает свой выход, что триггерит закрытие следующей.

### Что НЕ надо делать

**❌ Не закрывай входной канал:**

```go
go func() {
    defer close(out)
    defer close(input)  // ← НЕЛЬЗЯ!
    for v := range input {
        out <- v
    }
}()
```

**Проблема:** `close(input)` закроет канал, который читает предыдущая стадия. Паника при отправке.

**❌ Не закрывай out в горутине-читателе:**

```go
for v := range input {
    go func() {
        out <- v
        close(out)  // ← НЕЛЬЗЯ! Паника при повторе
    }()
}
```

**❌ Не забывай close:**

```go
go func() {
    for v := range input {
        out <- v
    }
    // забыли close(out)
}()
```

**Проблема:** следующая стадия (`for range out`) **никогда не завершится**.

### Схема

```
Stage 1:                          Stage 2:
  out := make(chan int)             out := make(chan int)
  go func() {                        go func() {
    defer close(out)                   defer close(out)
    for v := range input {             for v := range stage1Out {
      out <- v                           out <- v
    }                                  }
  }()                                }()
  
  close(stage1Out) ──────────────────► for range завершится
                                       │
                                       ▼
                                     close(stage2Out)
```

### 💡 Практика: как закрывать каналы

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Каждая стадия закрывает СВОЙ выход** (`defer close(out)`).
2. **НЕ закрывай входной канал.**
3. **`for v := range input`** — автоматическое завершение.

**❌ НЕ ДЕЛАЙ:**

4. **Не забывай `close(out)`.**
5. **Не закрывай канал дважды.**
6. **Не пиши в закрытый канал.**

---

## 11.6 Fan-out в стадии pipeline

Иногда стадия медленная. Можно разбить её на **N параллельных воркеров** — fan-out внутри стадии.

### Идея

Медленная стадия `enrich` читает из входа и пишет в выход. Заменим её на:

1. **Fan-out:** N воркеров читают из входа.
2. **Каждый воркер** обрабатывает.
3. **Fan-in:** слияние в один выход.

### Реализация

```go
func enrichStage(ctx context.Context, input <-chan Parsed, workers int) <-chan Enriched {
    // Fan-out: N воркеров
    outputs := make([]<-chan Enriched, workers)
    for i := 0; i < workers; i++ {
        outputs[i] = enrichWorker(ctx, input)
    }
    
    // Fan-in: слияние
    return merge(ctx, outputs...)
}

func enrichWorker(ctx context.Context, input <-chan Parsed) <-chan Enriched {
    out := make(chan Enriched)
    go func() {
        defer close(out)
        for v := range input {
            enriched := enrich(v)
            select {
            case out <- enriched:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func merge(ctx context.Context, channels ...<-chan Enriched) <-chan Enriched {
    out := make(chan Enriched)
    
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan Enriched) {
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

### Схема

```
input ──► ┌──────────┐ ──┐
          │ enrich 0 │   │
          └──────────┘   │
          ┌──────────┐   │
input ────▶│ enrich 1 │───┼──► merge ──► out
          └──────────┘   │
          ┌──────────┐   │
input ────▶│ enrich 2 │───┘
          └──────────┘
```

### Потребитель

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 100)
    parsed := parseStage(ctx, source)
    enriched := enrichStage(ctx, parsed, 10)  // ← 10 воркеров
    writeStage(ctx, enriched)
}
```

**Что происходит:**

- `parseStage` — 1 воркер.
- `enrichStage` — 10 воркеров (fan-out + fan-in).
- `writeStage` — 1 воркер.

**Разное число воркеров на разных стадиях** — это ключевое преимущество pipeline.

### Сколько воркеров

**Для CPU-bound:** `GOMAXPROCS`.

**Для I/O-bound:** 10–100.

**Формула:** `N = GOMAXPROCS × (1 + wait_time / compute_time)`.

### 💡 Практика: как использовать fan-out в стадии

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Fan-out для медленных стадий.**
2. **Fan-in для слияния результатов.**
3. **Разное число воркеров** для разных стадий.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики по стадиям** — покажут узкое место.

**❌ НЕ ДЕЛАЙ:**

5. **Не используй fan-out для быстрых стадий.**
6. **Не забывай `context`.**

---

## 11.7 В связке с другими паттернами

Pipeline — обобщение всех предыдущих паттернов. Разберём связки.

### Generator + pipeline

Generator — первая стадия pipeline:

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 100)  // generator
    doubled := doubleStage(ctx, source)
    filtered := filterEvenStage(ctx, doubled)
    printStage(ctx, filtered)
}
```

### Pipeline + fan-out + fan-in

Медленная стадия — fan-out + fan-in:

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 100)
    parsed := parseStage(ctx, source)
    enriched := enrichStage(ctx, parsed, 10)  // fan-out + fan-in
    written := writeStage(ctx, enriched)
    for range written {
    }
}
```

### Pipeline + rate limiter

Медленная стадия с rate limit:

```go
func enrichWorker(ctx context.Context, input <-chan Parsed, limiter *rate.Limiter) <-chan Enriched {
    out := make(chan Enriched)
    go func() {
        defer close(out)
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
    return out
}
```

### Pipeline + retry

Стадия с retry:

```go
func enrichWorker(ctx context.Context, input <-chan Parsed) <-chan Enriched {
    out := make(chan Enriched)
    go func() {
        defer close(out)
        for v := range input {
            enriched, err := enrichWithRetry(ctx, v, 3)
            if err != nil {
                // отправить ошибку или пропустить
                continue
            }
            select {
            case out <- enriched:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

### Схема: полный pipeline

```
Generator ──▶ Parse ──▶ Enrich (10 воркеров) ──▶ Write ──▶ Sink
   │             │             │                    │
   └─────────────┴─────────────┴────────────────────┘
                         context
```

### 💡 Практика: как комбинировать pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Generator — первая стадия.**
2. **Fan-out + fan-in — для медленных стадий.**
3. **Rate limiter — для внешних вызовов.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Retry — для временных ошибок.**
5. **Метрики по стадиям.**

**❌ НЕ ДЕЛАЙ:**

6. **Не смешивай логику стадий.**
7. **Не забывай `context`.**

---

## 11.8 Практика Go: конвейеры

Разберём **три примера**.

### Пример 1: простой pipeline

```go
package main

import (
    "context"
    "fmt"
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

func doubleStage(ctx context.Context, input <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range input {
            select {
            case out <- v * 2:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 10)
    doubled := doubleStage(ctx, source)
    
    for v := range doubled {
        fmt.Println(v)
    }
}
```

**Пример вывода:**

```
0
2
4
6
8
10
12
14
16
18
```

### Пример 2: pipeline с отменой

```go
package main

import (
    "context"
    "fmt"
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

func doubleStage(ctx context.Context, input <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range input {
            select {
            case out <- v * 2:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func filterEvenStage(ctx context.Context, input <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range input {
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

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 1_000_000)
    doubled := doubleStage(ctx, source)
    filtered := filterEvenStage(ctx, doubled)
    
    count := 0
    for v := range filtered {
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

### Пример 3: pipeline с fan-out в стадии

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

func fanOutStage(ctx context.Context, input <-chan int, workers int) <-chan int {
    outputs := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        outputs[i] = worker(ctx, input)
    }
    return merge(ctx, outputs...)
}

func worker(ctx context.Context, input <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range input {
            select {
            case out <- v * 2:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func merge(ctx context.Context, channels ...<-chan int) <-chan int {
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
    
    source := numbersCtx(ctx, 20)
    doubled := fanOutStage(ctx, source, 5)
    
    for v := range doubled {
        fmt.Println(v)
    }
}
```

**Что демонстрирует:** стадия pipeline с fan-out + fan-in.

### 💡 Практика: как писать pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Каждая стадия — функция** `func(ctx, input) <-chan Output`.
2. **`context` — через все стадии.**
3. **`defer close(out)` в каждой стадии.**
4. **Fan-out + fan-in** для медленных стадий.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Метрики по стадиям.**
6. **Разное число воркеров.**

**❌ НЕ ДЕЛАЙ:**

7. **Не забывай `close(out)`.**
8. **Не закрывай входной канал.**
9. **Не забывай `context`.**

---

## 11.9 Выводы и типичные ошибки

**Что мы узнали?**

Pipeline — цепочка стадий, каждая читает из входа и пишет в выход. Каждая стадия — функция `func(ctx, input) <-chan Output`. `context` — для отмены; `ctx.Done()` — broadcast. Backpressure между стадиями работает автоматически через каналы. Закрытие каналов — каскадное, каждая стадия закрывает свой выход. Fan-out внутри стадии — для медленных стадий. Pipeline + generator, fan-out, fan-in, rate limiter — классические связки.

**Типичные ошибки:**

- ❌ **Забыть `close(out)` в стадии.** Утечка.
- ❌ **Закрыть входной канал.** Паника при отправке.
- ❌ **Двойное закрытие.** Паника.
- ❌ **Писать в закрытый канал.** Паника.
- ❌ **Не использовать `select` с `ctx.Done()`.** Отмена не сработает.
- ❌ **Большие буферы.** Backpressure не работает.
- ❌ **Забыть `wg.Wait()` в merge.** `close(out)` не выполнится.
- ❌ **Смешивать стадии в одной функции.**
- ❌ **Не использовать fan-out для медленных стадий.**
- ❌ **Не мониторить пропускную способность стадий.**

---

## 11.10 Для быстрого повторения

- **Pipeline** — цепочка стадий. Каждая — `func(ctx, input) <-chan Output`.
- **Стадия** — читает из входа, пишет в выход, закрывает **свой** выход.
- **Backpressure** — автоматически через каналы.
- **Маленький буфер** — быстрый backpressure.
- **Большой буфер** — отложенный backpressure.
- **Закрытие** — каскадное. Кто пишет — тот закрывает.
- **`context`** — для отмены; `ctx.Done()` — broadcast.
- **Fan-out в стадии** — для медленных стадий.
- **Fan-in после fan-out** — слияние.
- **Разное число воркеров** на разных стадиях.
- **Pipeline + generator** — первая стадия.
- **Pipeline + rate limiter** — для внешних вызовов.
- **Метрики по стадиям** — показывают узкое место.

---

## 11.11 Вопросы для самопроверки

1. Что такое pipeline? Какую задачу решает?
2. Как построить простейший pipeline?
3. Зачем `context` в pipeline?
4. Что такое backpressure между стадиями?
5. Почему `close(out)` в отдельной горутине?
6. Как добавить fan-out в стадию pipeline?
7. Что произойдёт, если стадия не закроет свой выход?
8. Как комбинировать pipeline с rate limiter?

---

## 11.12 Ответы

### Ответ 1

**Pipeline** — цепочка стадий. Каждая стадия читает из входа и пишет в выход. Решает задачу: **последовательная обработка в несколько шагов** (parse → enrich → write).

### Ответ 2

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 10)
    doubled := doubleStage(ctx, source)
    filtered := filterEvenStage(ctx, doubled)
    printStage(ctx, filtered)
}
```

Каждая стадия — функция `func(ctx, input) <-chan Output`.

### Ответ 3

**`context`** позволяет остановить все стадии одновременно. Без него, если потребитель перестал читать, стадии **зависнут** на отправке. **Утечка.**

### Ответ 4

**Backpressure** — медленная стадия замедляет быструю. Через небуферизованный канал между стадиями. Если следующая стадия не читает — предыдущая блокируется на отправке.

### Ответ 5

**`wg.Wait()` блокируется.** Если `close(out)` в основной горутине — она не вернёт `out`. Потребитель не сможет читать. **Deadlock.**

**Отдельная горутина:** основная возвращает `out` сразу, отдельная ждёт завершения и закрывает.

### Ответ 6

**Fan-out в стадии:**

```go
func enrichStage(ctx context.Context, input <-chan Parsed, workers int) <-chan Enriched {
    outputs := make([]<-chan Enriched, workers)
    for i := 0; i < workers; i++ {
        outputs[i] = enrichWorker(ctx, input)
    }
    return merge(ctx, outputs...)
}
```

N воркеров читают из **общего** входа, пишут в **свои** каналы, `merge` сливает.

### Ответ 7

**Следующая стадия зависнет** на `for range out`. Она будет ждать, пока канал закроется. **Утечка горутины.**

### Ответ 8

**Pipeline + rate limiter:**

```go
func enrichWorker(ctx context.Context, input <-chan Parsed, limiter *rate.Limiter) <-chan Enriched {
    out := make(chan Enriched)
    go func() {
        defer close(out)
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
    return out
}
```

Rate limiter вызывается перед обработкой. Ограничивает скорость.

---

## 11.13 Куда идти дальше?

Мы разобрали pipeline — цепочки стадий. Теперь мы умеем строить конвейеры данных.

Но иногда pipeline **избыточен**. Что если нужно просто ограничить число одновременных операций, без стадий?

- **Как ограничить параллелизм?** → **Глава 12: Semaphore.**
- **Как построить worker pool?** → **Глава 13: Worker pool.**
- **Как построить rate limiter?** → **Глава 14: Rate limiter.**

---

## 11.14 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Pipeline** | Цепочка стадий | Каждая — `func(ctx, input) <-chan Output` |
| **Стадия** | Обработка | Читает из входа, пишет в выход, закрывает свой выход |
| **`context`** | Отмена | Broadcast всем стадиям |
| **Backpressure** | Автоматически | Через небуферизованный канал |
| **Маленький буфер** | 0–10 | Быстрый backpressure |
| **Большой буфер** | 1000+ | Отложенный backpressure |
| **Закрытие** | Каскадное | Кто пишет — тот закрывает |
| **Fan-out в стадии** | Для медленных | N воркеров + merge |
| **Разное число воркеров** | На разных стадиях | CPU vs I/O |
| **Generator + pipeline** | Первая стадия | Источник данных |
| **Rate limiter + pipeline** | Для внешних | Ограничение скорости |
| **Метрики** | По стадиям | Показывают узкое место |

🔗 **Ключевая идея:** Pipeline — цепочка стадий. Каждая стадия — `func(ctx, input) <-chan Output`. `context` — для отмены; `ctx.Done()` — broadcast всем стадиям. Backpressure между стадиями работает **автоматически** через каналы; размер буфера определяет его скорость. Закрытие каналов **каскадное**: каждая стадия закрывает свой выход. Fan-out внутри стадии — для медленных стадий; fan-in после fan-out — слияние. Разное число воркеров на разных стадиях. Pipeline + generator, rate limiter, retry — классические связки. Метрики по стадиям показывают узкое место.