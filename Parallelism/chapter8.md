# 🔀 Глава 8: Fan-in, Fan-out, Pipeline — конвейеры данных

**Что вы узнаете:**
- Что такое **fan-out** и зачем распределять работу между несколькими горутинами.
- Что такое **fan-in** и как слить результаты из нескольких каналов в один.
- Что такое **pipeline** и как построить многостадийную обработку.
- Как работает **backpressure** между стадиями pipeline.
- Что такое **tee-канал** и **bridge-канал**.
- Как **отменять** pipeline через `context.Context`.
- Как **закрывать** каналы в pipeline без утечек и deadlock.
- Как диагностировать проблемы pipeline через pprof и метрики.
- Как комбинировать pipeline с worker pool (Глава 7).

**После прочтения вы сможете:**
- Построить fan-out/fan-in для распределения и сбора работы.
- Построить pipeline из N стадий с backpressure.
- Правильно закрывать каналы в каждой стадии.
- Отменять pipeline через `context`.
- Реализовать tee-канал (разветвление) и bridge-канал (склейка).
- Диагностировать утечки и блокировки в pipeline.
- Измерять пропускную способность и latency каждой стадии.

---

## Содержание

- [8.0 Пролог: конвейер, который встал](#80-пролог-конвейер-который-встал)
- [8.1 Fan-out: распределение работы](#81-fan-out-распределение-работы)
- [8.2 Fan-in: слияние результатов](#82-fan-in-слияние-результатов)
- [8.3 Fan-out + Fan-in: полная схема](#83-fan-out--fan-in-полная-схема)
- [8.4 Pipeline: многостадийная обработка](#84-pipeline-многостадийная-обработка)
- [8.5 Backpressure в pipeline](#85-backpressure-в-pipeline)
- [8.6 Закрытие каналов в pipeline](#86-закрытие-каналов-в-pipeline)
- [8.7 Отмена pipeline через context](#87-отмена-pipeline-через-context)
- [8.8 Tee-канал и bridge-канал](#88-tee-канал-и-bridge-канал)
- [8.9 Pipeline vs worker pool: что выбрать](#89-pipeline-vs-worker-pool-что-выбрать)
- [8.10 Практика Go: pipeline с метриками](#810-практика-go-pipeline-с-метриками)
- [8.11 Выводы и типичные ошибки](#811-выводы-и-типичные-ошибки)
- [8.12 Для быстрого повторения](#812-для-быстрого-повторения)
- [8.13 Вопросы для самопроверки](#813-вопросы-для-самопроверки)
- [8.14 Ответы](#814-ответы)
- [8.15 Куда идти дальше?](#815-куда-идти-дальше)
- [8.16 Чек-лист](#816-чек-лист)

---

## 8.0 Пролог: конвейер, который встал

Ты пишешь ETL-пайплайн: читаешь логи из Kafka, парсишь, обогащаешь данными из БД, записываешь в ClickHouse.

Наивная реализация:

```go
func main() {
    for {
        msg := kafka.Read()
        parsed := parse(msg)
        enriched := enrich(parsed)
        writeToClickHouse(enriched)
    }
}
```

Всё работает. Пока не выясняется:

- **Kafka быстрая** — 100 000 сообщений/сек.
- **Парсинг быстрый** — 1 мкс на сообщение.
- **Обогащение медленное** — 10 мс (запрос в БД).
- **ClickHouse быстрый** — 100 мкс на запись.

**Пропускная способность:** 1 / 10 мс = **100 сообщений/сек**. Хотя Kafka даёт 100 000.

❓ **Что произошло?** Ты обрабатываешь сообщения **последовательно**. Обогащение — bottleneck. Пока одно сообщение обогащается, остальные ждут.

💡 **Решение:** **pipeline** + **fan-out**.

- **Стадия 1:** читаем из Kafka.
- **Стадия 2:** парсим (быстро).
- **Стадия 3:** обогащаем — **100 параллельных воркеров** (fan-out).
- **Стадия 4:** пишем в ClickHouse.

**Пропускная способность:** 100 воркеров × (1 / 10 мс) = **10 000 сообщений/сек**. В 100 раз быстрее.

> **Важный мост к будущим главам:** pipeline — это обобщение worker pool (Глава 7). Fan-out — это worker pool для одной стадии. Fan-in — это слияние каналов. Backpressure — из Главы 7. Отмена через `context` — из Главы 5.

---

## 8.1 Fan-out: распределение работы

**Fan-out** — это распределение работы от **одного** источника к **нескольким** обработчикам.

### Схема

```
                  ┌──────────┐
              ┌──▶│ Worker 1 │──┐
              │   └──────────┘  │
              │   ┌──────────┐  │
   ┌──────┐   ├──▶│ Worker 2 │──┤   ┌─────────┐
   │Source│───┤   └──────────┘  ├──▶│ Results │
   └──────┘   │   ┌──────────┐  │   └─────────┘
              ├──▶│ Worker 3 │──┤
              │   └──────────┘  │
              │   ┌──────────┐  │
              └──▶│ Worker N │──┘
                  └──────────┘
```

**Source** пишет в **один канал**. N воркеров читают из него. Каждый воркер обрабатывает **свою** порцию задач.

### Ключевое: канал распределяет задачи

Когда N воркеров читают из **одного** канала, runtime **автоматически** распределяет задачи:

```go
tasksCh := make(chan Task)

// N воркеров читают из ОДНОГО канала
for i := 0; i < N; i++ {
    go func() {
        for task := range tasksCh {  // ← каждый воркер читает свою задачу
            process(task)
        }
    }()
}
```

**Что происходит:**

- Producer пишет Task{1} в канал.
- Один из воркеров (случайный) забирает его.
- Producer пишет Task{2}.
- Другой воркер забирает его.
- И так далее.

**Распределение автоматическое.** Не нужно вручную решать, какой воркер какую задачу возьмёт.

### Fan-out vs worker pool

**Fan-out** — это **та же идея**, что worker pool. Разница в **терминологии**:

- **Worker pool** — акцент на **ограничении** параллелизма.
- **Fan-out** — акцент на **распределении** работы.

**Пример fan-out:**

```go
func fanOut(ctx context.Context, input <-chan Task, n int) []<-chan Result {
    outputs := make([]<-chan Result, n)
    for i := 0; i < n; i++ {
        outputs[i] = worker(ctx, input)
    }
    return outputs
}

func worker(ctx context.Context, input <-chan Task) <-chan Result {
    out := make(chan Result)
    go func() {
        defer close(out)
        for task := range input {
            select {
            case out <- process(task):
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

**Что вернул `fanOut`:** **N каналов** результатов. Каждый — от своего воркера.

**Зачем N каналов:** их можно **слить** через fan-in (см. 8.2).

### Сколько воркеров

**Те же правила, что в Главе 7:**

- CPU-bound → `GOMAXPROCS`.
- I/O-bound → 10-100.

### Визуализация работы

```
t=0:    Producer пишет Task{1} в input
        Воркер 1 забирает Task{1}
        Producer пишет Task{2}
        Воркер 2 забирает Task{2}
        Producer пишет Task{3}
        Воркер 3 забирает Task{3}
        ...

t=10ms: Воркер 1 завершил Task{1}, пишет в output1
        Воркер 2 завершил Task{2}, пишет в output2
        Воркер 1 забирает Task{4}
        ...
```

**Ключевое:** каждый воркер работает **независимо**. Быстрый воркер берёт больше задач.

### Аннотация сложности

| Аспект | Значение |
|:---|:---|
| Воркеров | N |
| Горутин | N + producer |
| Память | N × 2.3 КБ |
| Пропускная способность | min(N × rate_worker, rate_source) |

### 💡 Практика: как использовать fan-out

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Один канал — N воркеров.** Распределение автоматическое.
2. **N по типу задачи:** CPU-bound → `GOMAXPROCS`, I/O-bound → 10-100.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Возвращай N каналов результатов** — для последующего fan-in.
4. **Отмена через `context`** — в `select`.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **Динамическое N** — для переменной нагрузки (см. 7.7).

**❌ НЕ ДЕЛАЙ:**

6. **Не создавай канал на каждого воркера.** Один канал — N воркеров.
7. **Не путай fan-out с fan-in.** Fan-out — распределение, fan-in — слияние.

### Ключевые выводы подглавы 8.1

- **Fan-out** — распределение работы от одного источника к N обработчикам.
- **Один канал — N воркеров.** Распределение автоматическое.
- **N** — по типу задачи.
- **Возвращай N каналов** для fan-in.
- **Fan-out ≈ worker pool** по идее.

---

## 8.2 Fan-in: слияние результатов

**Fan-in** — это слияние **нескольких** каналов в **один**.

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

**N каналов** объединяются в **один**. Все результаты идут в один поток.

### Реализация: `merge`

```go
func merge(ctx context.Context, channels ...<-chan Result) <-chan Result {
    out := make(chan Result)
    var wg sync.WaitGroup
    
    // Для каждого канала — отдельная горутина
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan Result) {
            defer wg.Done()
            for result := range c {
                select {
                case out <- result:
                case <-ctx.Done():
                    return
                }
            }
        }(ch)
    }
    
    // Закрытие out после всех горутин
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

**Что делает `merge`:**

1. Создаёт **один** канал `out`.
2. Для **каждого** входного канала — отдельная горутина.
3. Каждая горутина читает из своего канала и пишет в `out`.
4. Когда все входные каналы закрыты — `close(out)`.

**Ключевое:** `out` получает результаты из **всех** входных каналов.

### Порядок результатов

**Порядок не гарантирован.** Результаты приходят в порядке готовности. Если важен порядок — нужно дополнительное упорядочивание (например, по ID).

### Fan-in + fan-out

**Fan-out + fan-in** — классическая комбинация:

```go
// Fan-out: N воркеров
channels := fanOut(ctx, input, N)

// Fan-in: слияние в один канал
output := merge(ctx, channels...)

// Читаем из одного канала
for result := range output {
    process(result)
}
```

**Схема:**

```
                    Fan-out                    Fan-in
   ┌──────┐    ┌──────────┐                ┌──────────┐
   │Source│───▶│ Worker 1 │───▶ Channel 1 ─┤          │
   └──────┘    └──────────┘                │          │
               ┌──────────┐                │          │
               │ Worker 2 │───▶ Channel 2 ─┤  merge   │───▶ Combined
               └──────────┘                │          │
               ┌──────────┐                │          │
               │ Worker 3 │───▶ Channel 3 ─┤          │
               └──────────┘                └──────────┘
```

### Сколько горутин в merge

**N + 1 горутина:**

- N горутин — по одной на каждый входной канал.
- 1 горутина — `wg.Wait()` + `close(out)`.

**Память:** (N + 1) × 2.3 КБ.

### Альтернатива: `reflect.Select`

**`reflect.Select`** позволяет динамически выбирать из N каналов:

```go
func mergeReflect(channels []<-chan Result) <-chan Result {
    out := make(chan Result)
    go func() {
        defer close(out)
        cases := make([]reflect.SelectCase, len(channels))
        for i, ch := range channels {
            cases[i] = reflect.SelectCase{
                Dir:  reflect.SelectRecv,
                Chan: reflect.ValueOf(ch),
            }
        }
        for len(cases) > 0 {
            i, v, ok := reflect.Select(cases)
            if !ok {
                cases = append(cases[:i], cases[i+1:]...)
                continue
            }
            out <- v.Interface().(Result)
        }
    }()
    return out
}
```

**Плюсы:**

- **Одна горутина**, а не N.
- **Меньше памяти.**

**Минусы:**

- **`reflect` медленнее** (~10x).
- **Сложнее код.**

**Рекомендация:** для N < 100 — простая реализация с N горутинами. Для N > 1000 — `reflect.Select`.

### Аннотация сложности

| Реализация | Горутин | Time (per element) | Сложность |
|:---|:---|:---|:---|
| N горутин | N + 1 | ~50-100 нс | Простая |
| `reflect.Select` | 1 | ~500-1000 нс | Сложная |

### 💡 Практика: как использовать fan-in

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **N горутин + 1 для `close`.** Простая реализация.
2. **`select` с `ctx.Done()`** для отмены.

**👍 СТОИТ СДЕЛАТЬ:**

3. **`reflect.Select`** — если N > 1000.
4. **Порядок результатов** — если важен, сортируй по ID.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **Буферизованный `out`** — для снижения contention.

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `close(out)`.** Иначе collector зависнет.
7. **Не используй `Mutex` для слияния** — канал проще и безопаснее.

### Ключевые выводы подглавы 8.2

- **Fan-in** — слияние N каналов в один.
- **N горутин + 1** для `close`.
- **Порядок не гарантирован.**
- **Fan-out + fan-in** — классическая комбинация.
- **`reflect.Select`** — для больших N.

---

## 8.3 Fan-out + Fan-in: полная схема

Соберём **полную схему** fan-out + fan-in.

### Полный код

```go
package main

import (
    "context"
    "fmt"
    "sync"
)

type Task struct {
    ID  int
    URL string
}

type Result struct {
    TaskID int
    Status int
    Err    error
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Входной канал
    tasksCh := make(chan Task, 100)
    
    // Producer
    go func() {
        defer close(tasksCh)
        for i := 0; i < 100; i++ {
            select {
            case tasksCh <- Task{ID: i, URL: "https://example.com"}:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    // Fan-out: 10 воркеров
    const numWorkers = 10
    workerOutputs := fanOut(ctx, tasksCh, numWorkers)
    
    // Fan-in: слияние в один канал
    merged := merge(ctx, workerOutputs...)
    
    // Collector
    for result := range merged {
        if result.Err != nil {
            fmt.Printf("task %d failed: %v\n", result.TaskID, result.Err)
        }
    }
    
    fmt.Println("done")
}

func fanOut(ctx context.Context, input <-chan Task, n int) []<-chan Result {
    outputs := make([]<-chan Result, n)
    for i := 0; i < n; i++ {
        outputs[i] = worker(ctx, input)
    }
    return outputs
}

func worker(ctx context.Context, input <-chan Task) <-chan Result {
    out := make(chan Result)
    go func() {
        defer close(out)
        for task := range input {
            select {
            case out <- process(ctx, task):
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func merge(ctx context.Context, channels ...<-chan Result) <-chan Result {
    out := make(chan Result)
    var wg sync.WaitGroup
    
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan Result) {
            defer wg.Done()
            for result := range c {
                select {
                case out <- result:
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

func process(ctx context.Context, task Task) Result {
    // имитация работы
    return Result{TaskID: task.ID, Status: 200}
}
```

### Что происходит

```
t=0:    10 воркеров запущены, каждый читает из tasksCh
        Producer пишет задачи
        10 каналов результатов создано (workerOutputs)
        merge запущен — 10 горутин читают из workerOutputs
        Collector читает из merged

t=T:    Producer закончил, close(tasksCh)
        Воркеры дочитывают остатки, close(свои output)
        merge-горутины видят close, завершаются
        wg.Wait() возвращается, close(merged)
        Collector видит close, завершается

t=T+ε:  Все горутины завершены
```

### Сколько горутин

| Компонент | Горутин |
|:---|:---|
| Producer | 1 |
| Воркеры | N |
| Merge (читатели) | N |
| Merge (close) | 1 |
| Collector | 1 |
| **Итого** | **2N + 3** |

**Для N = 10:** 23 горутины. **Для N = 100:** 203 горутины.

### Альтернатива: worker pool с одним каналом результатов

**Fan-out + fan-in** можно упростить, если воркеры пишут в **один** канал результатов:

```go
tasksCh := make(chan Task, 100)
resultsCh := make(chan Result, 100)

var wg sync.WaitGroup
for i := 0; i < N; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for task := range tasksCh {
            resultsCh <- process(task)  // ← один канал
        }
    }()
}

go func() {
    wg.Wait()
    close(resultsCh)
}()

for result := range resultsCh {
    process(result)
}
```

**Что изменилось:** вместо N каналов + merge — **один** канал результатов.

**Плюсы:**

- **Меньше горутин:** N + 2 вместо 2N + 3.
- **Проще код.**
- **Меньше contention** (один канал вместо N+1).

**Минусы:**

- **Нет разделения** по воркерам. Если нужно обработать результаты **разных** воркеров по-разному — не подходит.

**Рекомендация:** для большинства случаев — **один канал результатов**. Fan-out + fan-in — если нужна гибкость (разные обработчики для разных воркеров).

### Сравнение

| Подход | Горутин | Сложность | Гибкость |
|:---|:---|:---|:---|
| Fan-out + fan-in | 2N + 3 | Средняя | Высокая |
| Worker pool с одним каналом | N + 2 | Низкая | Низкая |

### Аннотация сложности

| Подход | Горутин (N=10) | Горутин (N=100) | Time (per element) |
|:---|:---|:---|:---|
| Fan-out + fan-in | 23 | 203 | ~100-200 нс |
| Worker pool | 12 | 102 | ~50-100 нс |

### 💡 Практика: какую схему выбрать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Worker pool с одним каналом** — для большинства случаев.
2. **Fan-out + fan-in** — если нужна гибкость.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Буферизованные каналы** для снижения contention.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

4. **`reflect.Select`** — для больших N.

**❌ НЕ ДЕЛАЙ:**

5. **Не создавай N + 1 каналов**, если можно обойтись одним.
6. **Не путай fan-out и fan-in.**

### Ключевые выводы подглавы 8.3

- **Fan-out + fan-in** — классическая комбинация.
- **Горутин:** 2N + 3 для fan-out + fan-in, N + 2 для worker pool.
- **Worker pool с одним каналом** — проще и быстрее.
- **Fan-out + fan-in** — для гибкости.

---

## 8.4 Pipeline: многостадийная обработка

**Pipeline** — это **цепочка стадий**, где каждая стадия:

1. Читает из входного канала.
2. Обрабатывает.
3. Пишет в выходной канал.

### Схема

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Source  │──▶│  Stage 1 │──▶│  Stage 2 │──▶│  Stage 3 │──▶ Sink
└──────────┘   └──────────┘   └──────────┘   └──────────┘
   (Kafka)      (parse)        (enrich)      (write)
```

**Каждая стадия — это функция**, которая принимает входной канал и возвращает выходной.

### Пример: ETL-пайплайн

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Source: читаем из Kafka
    source := readKafka(ctx)
    
    // Stage 1: парсим
    parsed := parseStage(ctx, source)
    
    // Stage 2: обогащаем (fan-out на 100 воркеров)
    enriched := enrichStage(ctx, parsed, 100)
    
    // Stage 3: записываем в ClickHouse
    written := writeStage(ctx, enriched)
    
    // Sink: ждём завершения
    for range written {
        // ничего не делаем, просто ждём
    }
}
```

### Реализация стадии

**Каждая стадия — функция** с сигнатурой:

```go
func stage(ctx context.Context, input <-chan Input) <-chan Output
```

**Пример: parseStage**

```go
func parseStage(ctx context.Context, input <-chan RawMessage) <-chan ParsedMessage {
    out := make(chan ParsedMessage)
    go func() {
        defer close(out)
        for msg := range input {
            parsed, err := parse(msg)
            if err != nil {
                continue  // пропускаем невалидные
            }
            select {
            case out <- parsed:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

**Что делает:**

1. Создаёт выходной канал.
2. Запускает горутину.
3. Читает из входа, парсит, пишет в выход.
4. При `ctx.Done()` — завершается.
5. При закрытии входа — `close(out)`.

### Стадия с fan-out

**Стадия, которая делает fan-out:**

```go
func enrichStage(ctx context.Context, input <-chan ParsedMessage, n int) <-chan EnrichedMessage {
    // Fan-out
    workers := make([]<-chan EnrichedMessage, n)
    for i := 0; i < n; i++ {
        workers[i] = enrichWorker(ctx, input)
    }
    
    // Fan-in
    return merge(ctx, workers...)
}

func enrichWorker(ctx context.Context, input <-chan ParsedMessage) <-chan EnrichedMessage {
    out := make(chan EnrichedMessage)
    go func() {
        defer close(out)
        for msg := range input {
            enriched := enrich(ctx, msg)
            select {
            case out <- enriched:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func merge(ctx context.Context, channels ...<-chan EnrichedMessage) <-chan EnrichedMessage {
    out := make(chan EnrichedMessage)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan EnrichedMessage) {
            defer wg.Done()
            for msg := range c {
                select {
                case out <- msg:
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

### Полная схема pipeline

```
Source → parseStage → enrichStage (fan-out N) → writeStage → Sink
  1         1             N + N + 1               1          1

Горутин: 1 + 1 + (N + N + 1) + 1 + 1 = 2N + 5
```

### Визуализация

```
t=0:    Source пишет в source
        parseStage читает, парсит, пишет в parsed
        enrichStage: N воркеров читают из parsed
        merge: N горутин читают из workerOutputs
        writeStage читает из merged, пишет в ClickHouse
        Sink читает из written

t=1:    Source пишет 1000 сообщений
        parseStage обрабатывает их за 1 мс
        enrichStage: 100 воркеров × 10 мс = 100 сообщений/мс
        writeStage: 1000 сообщений за 10 мс

Пропускная способность: 100 сообщений/мс = 100 000 сообщений/сек
```

### Backpressure между стадиями

**Каждая стадия — отдельная горутина.** Если стадия медленная — её входной канал заполняется. Предыдущая стадия **блокируется** на `out <- msg`. Это **backpressure**.

```
Source (быстро) → parsed (заполнен) → enrich (медленно)
                                     ↑
                              backpressure
```

**Ключевое:** pipeline **автоматически** балансирует скорость через backpressure.

### Сравнение с последовательной обработкой

**Последовательно:**

```
1000 сообщений × (1 мкс parse + 10 мс enrich + 100 мкс write) = 10.1 сек
```

**Pipeline:**

```
1000 сообщений / 100 воркеров × 10 мс = 100 мс
```

**В 100 раз быстрее.**

### Аннотация сложности

| Аспект | Последовательно | Pipeline |
|:---|:---|:---|
| Пропускная способность | 1 / sum(times) | N / max(time) |
| Latency (per message) | sum(times) | sum(times) |
| Горутин | 1 | 2N + 5 |

### 💡 Практика: как строить pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Каждая стадия — функция** `func(ctx, input) <-chan Output`.
2. **`defer close(out)`** в каждой стадии.
3. **`select` с `ctx.Done()`** для отмены.
4. **Fan-out для медленных стадий.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Буферизованные каналы** между стадиями.
6. **Метрики на каждой стадии** — пропускная способность, latency.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **Динамическое N** — для переменной нагрузки.

**❌ НЕ ДЕЛАЙ:**

8. **Не делай стадии слишком мелкими.** Overhead на каналы.
9. **Не забывай `close`.** Утечка горутин.
10. **Не блокируй стадию без `select`.** Отмена не сработает.

### Ключевые выводы подглавы 8.4

- **Pipeline** — цепочка стадий, каждая читает из входа и пишет в выход.
- **Стадия — функция** `func(ctx, input) <-chan Output`.
- **Fan-out** для медленных стадий.
- **Backpressure** между стадиями автоматически.
- **В 100 раз быстрее** последовательной обработки.

---

## 8.5 Backpressure в pipeline

Разберём **backpressure** между стадиями.

### Как работает backpressure

**Backpressure** — это когда **медленная стадия замедляет быструю**.

```
Source (быстро) → parsed (буфер 100) → enrich (медленно)
```

**Что происходит:**

1. Source пишет в `parsed` быстро.
2. `parsed` заполняется (100 элементов).
3. Source **блокируется** на `parsed <- msg`.
4. Source ждёт, пока enrich не заберёт из `parsed`.
5. Enrich обрабатывает медленно → `parsed` остаётся полным.
6. Source продолжает ждать.

**Результат:** Source работает со скоростью enrich.

### Размер буфера и backpressure

**Маленький буфер (1-10):**

- Backpressure **быстрый**.
- Source блокируется часто.
- Меньше памяти.

**Большой буфер (1000+):**

- Backpressure **отложенный**.
- Source блокируется редко.
- Больше памяти.

**Очень большой буфер (100 000):**

- Backpressure **не работает**.
- Source пишет всё в буфер.
- Память растёт.

### Визуализация

```
Маленький буфер (10):

  Source: ████████░░░░░░░░░░░░░░░░ (блокируется)
  Enrich: ████░░░░░░░░░░░░░░░░░░░░ (медленно)
  
  Source работает со скоростью Enrich.

Большой буфер (1000):

  Source: ████████████████████████ (не блокируется)
  Enrich: ████░░░░░░░░░░░░░░░░░░░░ (медленно)
  
  Буфер заполняется, память растёт.

Очень большой буфер (100 000):

  Source: ████████████████████████
  Enrich: ████░░░░░░░░░░░░░░░░░░░░
  
  OOM через минуту.
```

### Как выбрать размер буфера

**Правило:** буфер должен **сглаживать пики**, но **не скрывать** проблему.

**Рекомендации:**

- **1-10× пропускная способность стадии.** Если стадия обрабатывает 1000/сек, буфер 100-10 000.
- **Не больше 10 000.** Иначе backpressure не работает.
- **Мониторь длину буфера.** Если постоянно полный — стадия медленная.

### Альтернатива: `select` с `default`

**Без блокировки:**

```go
select {
case out <- msg:
case <-ctx.Done():
    return
default:
    // буфер полон — пропускаем или логируем
    metrics.Dropped.Add(1)
}
```

**Что делает:** если буфер полон, сообщение **пропускается**. Это **не backpressure**, а **drop**.

**Когда использовать:** если сообщения **можно терять** (метрики, логи).

**Когда не использовать:** если **нельзя терять** (заказы, платежи).

### Комбинация: backpressure + drop

```go
select {
case out <- msg:
    // отправлено
case <-ctx.Done():
    return
default:
    // буфер полон — попробовать ещё раз с таймаутом
    select {
    case out <- msg:
    case <-time.After(100 * time.Millisecond):
        metrics.Dropped.Add(1)
    case <-ctx.Done():
        return
    }
}
```

**Что делает:** ждёт 100 мс, потом дропает.

### Аннотация сложности

| Размер буфера | Backpressure | Память | Когда |
|:---|:---|:---|:---|
| 1 | Мгновенный | Минимум | Синхронная обработка |
| 10-100 | Быстрый | Мало | Большинство случаев |
| 1000-10000 | Отложенный | Средне | Пики нагрузки |
| 100000+ | Не работает | Много | ❌ Не использовать |

### 💡 Практика: как настроить backpressure

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Буфер 10-1000** для большинства случаев.
2. **Мониторь длину буфера.**
3. **Backpressure** — для критичных данных.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`select` с `default`** — для некритичных данных.
5. **Drop метрики** — сколько потеряно.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Динамический буфер** — редко нужно.

**❌ НЕ ДЕЛАЙ:**

7. **Не ставь буфер 100 000+.** Backpressure не работает.
8. **Не игнорируй заполненный буфер.** Это сигнал.

### Ключевые выводы подглавы 8.5

- **Backpressure** — медленная стадия замедляет быструю.
- **Размер буфера** определяет, как быстро работает backpressure.
- **Маленький буфер** — быстрый backpressure. **Большой** — отложенный.
- **`select` с `default`** — drop вместо backpressure.
- **Мониторь длину буфера.**

---

## 8.6 Закрытие каналов в pipeline

Разберём **правильное закрытие** каналов в pipeline.

### Правило: кто пишет — тот закрывает

**В pipeline** каждая стадия:

- **Читает** из входного канала.
- **Пишет** в выходной канал.
- **Закрывает** свой **выходной** канал.

**Не закрывает входной** — это делает предыдущая стадия.

### Пример

```go
func stage(ctx context.Context, input <-chan Input) <-chan Output {
    out := make(chan Output)
    go func() {
        defer close(out)  // ← закрываем СВОЙ выход
        for msg := range input {  // ← читаем из чужого входа
            select {
            case out <- process(msg):
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

**Что происходит:**

1. Стадия читает из `input`.
2. Когда `input` закрыт — `for range` завершается.
3. `defer close(out)` закрывает выход.
4. Следующая стадия видит `close` и завершается.

### Каскадное закрытие

```
Source закрывает source
    ↓
Stage 1: for range source завершается → close(stage1Out)
    ↓
Stage 2: for range stage1Out завершается → close(stage2Out)
    ↓
Stage 3: for range stage2Out завершается → close(stage3Out)
    ↓
Sink: for range stage3Out завершается
```

**Ключевое:** закрытие **каскадное**. Каждая стадия закрывает свой выход, что триггерит закрытие следующей.

### Проблема: fan-out/fan-in

**Fan-out:** N воркеров читают из **одного** входа. Кто закроет **выходы** воркеров?

**Fan-in:** N каналов сливаются в **один**. Кто закроет **merged**?

**Решение:**

1. Каждый воркер закрывает **свой** выход (`defer close(workerOut)`).
2. `merge` ждёт **все** воркеры (`wg.Wait()`), потом закрывает `merged`.

```go
func merge(ctx context.Context, channels ...<-chan Result) <-chan Result {
    out := make(chan Result)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan Result) {
            defer wg.Done()
            for result := range c {  // ← читаем из чужого канала
                select {
                case out <- result:
                case <-ctx.Done():
                    return
                }
            }
        }(ch)
    }
    go func() {
        wg.Wait()  // ← ждём всех
        close(out)  // ← закрываем
    }()
    return out
}
```

### Проблема: забыть close

**❌ Плохо:**

```go
func stage(input <-chan Input) <-chan Output {
    out := make(chan Output)
    go func() {
        for msg := range input {
            out <- process(msg)
        }
        // забыли close(out)
    }()
    return out
}
```

**Что происходит:** следующая стадия (`for range out`) **никогда не завершится**. **Утечка горутин.**

### Проблема: закрыть вход

**❌ Плохо:**

```go
func stage(input <-chan Input) <-chan Output {
    out := make(chan Output)
    go func() {
        defer close(out)
        defer close(input)  // ← НЕЛЬЗЯ!
        for msg := range input {
            out <- process(msg)
        }
    }()
    return out
}
```

**Что происходит:** `close(input)` закроет канал, который **читает другая стадия**. Паника при отправке в закрытый канал. **Panic.**

### Проблема: двойное закрытие

**❌ Плохо:**

```go
go func() {
    defer close(out)
    for msg := range input {
        out <- process(msg)
    }
    close(out)  // ← паника: close of closed channel
}()
```

**Что происходит:** `defer close(out)` + `close(out)` — двойное закрытие. **Panic.**

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| `close(ch)` | ~50-100 нс | 0 |
| `for range ch` после close | ~50-100 нс на элемент | 0 |
| `wg.Wait()` | ~10-20 нс + ожидание | 0 |

### 💡 Практика: как закрывать каналы в pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Каждая стадия закрывает СВОЙ выход** (`defer close(out)`).
2. **НЕ закрывай входной канал** — это делает предыдущая стадия.
3. **`merge` ждёт всех** (`wg.Wait()`) и закрывает `merged`.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`defer close(out)`** сразу после `make(chan)`.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **`context.Context`** для отмены — в дополнение к close.

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `close(out)`.** Утечка.
7. **Не закрывай входной канал.**
8. **Не закрывай канал дважды.** Паника.
9. **Не пиши в закрытый канал.** Паника.

### Ключевые выводы подглавы 8.6

- **Кто пишет — тот закрывает.** Каждая стадия закрывает **свой** выход.
- **Не закрывай входной канал.**
- **Каскадное закрытие:** закрытие входа → завершение → закрытие выхода.
- **`merge` ждёт всех** и закрывает merged.
- **Забыть close** → утечка. **Двойное close** → паника.

---

## 8.7 Отмена pipeline через context

Разберём **отмену** pipeline через `context.Context`.

### Зачем отмена

**Сценарии:**

- **Таймаут.** Pipeline работает слишком долго.
- **Ошибка.** Критичная ошибка в одной из стадий.
- **Сигнал ОС.** Ctrl+C.
- **Клиент отключился.** HTTP-запрос отменён.

### Паттерн: `context` во всех стадиях

```go
func stage(ctx context.Context, input <-chan Input) <-chan Output {
    out := make(chan Output)
    go func() {
        defer close(out)
        for {
            select {
            case <-ctx.Done():
                return  // отмена
            case msg, ok := <-input:
                if !ok {
                    return  // вход закрыт
                }
                select {
                case out <- process(msg):
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    return out
}
```

**Что происходит:**

1. Проверка `ctx.Done()` в **двух** местах: при чтении из входа и при записи в выход.
2. Если `ctx` отменён — стадия завершается.
3. `defer close(out)` закрывает выход.
4. Следующая стадия видит `close` или `ctx.Done()` и тоже завершается.

### Каскадная отмена

```
ctx отменён
    ↓
Source: select case <-ctx.Done() → return
    ↓
Stage 1: select case <-ctx.Done() → return → close(stage1Out)
    ↓
Stage 2: select case <-ctx.Done() → return → close(stage2Out)
    ↓
Sink: select case <-ctx.Done() → return
```

**Ключевое:** `ctx.Done()` — **broadcast**. Все стадии видят отмену **одновременно**.

### Полный пример с отменой

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    source := readKafka(ctx)
    parsed := parseStage(ctx, source)
    enriched := enrichStage(ctx, parsed, 100)
    written := writeStage(ctx, enriched)
    
    for range written {
        // ждём завершения
    }
    
    if ctx.Err() != nil {
        fmt.Println("pipeline canceled:", ctx.Err())
    }
}

func readKafka(ctx context.Context) <-chan RawMessage {
    out := make(chan RawMessage)
    go func() {
        defer close(out)
        for {
            msg, err := kafka.Read()
            if err != nil {
                return
            }
            select {
            case out <- msg:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

**Что происходит:**

1. `WithTimeout(10s)` — pipeline работает максимум 10 секунд.
2. Через 10 секунд `ctx.Done()` закрывается.
3. Все стадии видят отмену и завершаются.
4. `ctx.Err()` = `DeadlineExceeded`.

### Отмена при ошибке

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := readKafka(ctx)
    parsed := parseStage(ctx, source)
    enriched := enrichStage(ctx, parsed, 100)
    written := writeStage(ctx, enriched)
    
    // Обработка ошибок
    errCh := make(chan error, 1)
    go func() {
        for result := range written {
            if result.Err != nil {
                errCh <- result.Err
                cancel()  // отменяем всё
                return
            }
        }
    }()
    
    select {
    case err := <-errCh:
        fmt.Println("error:", err)
    case <-ctx.Done():
        fmt.Println("done or canceled")
    }
}
```

**Что происходит:**

1. Отдельная горутина читает результаты.
2. При ошибке — `cancel()` отменяет всё.
3. Все стадии завершаются.

### `errgroup` для pipeline

```go
g, ctx := errgroup.WithContext(context.Background())

// Запуск стадий
g.Go(func() error { return runSource(ctx, source) })
g.Go(func() error { return runParse(ctx, source, parsed) })
g.Go(func() error { return runEnrich(ctx, parsed, enriched) })
g.Go(func() error { return runWrite(ctx, enriched) })

if err := g.Wait(); err != nil {
    fmt.Println("error:", err)
}
```

**Что делает `errgroup`:**

- Запускает N горутин.
- При первой ошибке — отменяет `ctx`.
- `g.Wait()` возвращает первую ошибку.

### Аннотация сложности

| Подход | Отмена | Ошибок | Сложность |
|:---|:---|:---|:---|
| `context` + `cancel` | ✅ | Первая | Средняя |
| `errgroup` | ✅ | Первая | Низкая |
| Ручной `errCh` | ✅ | Первая | Средняя |

### 💡 Практика: как отменять pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`context.Context`** во всех стадиях.
2. **`select` с `ctx.Done()`** — при чтении и записи.
3. **`errgroup`** — для обработки ошибок.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`context.WithTimeout`** — для ограничения времени.
5. **`cancel()` при ошибке** — отменяет всё.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **`errCh`** — если нужна своя логика обработки ошибок.

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй `ctx.Done()`.** Отмена не сработает.
8. **Не забывай `cancel()`.** Утечка.

### Ключевые выводы подглавы 8.7

- **`context.Context`** — для отмены pipeline.
- **`select` с `ctx.Done()`** — при чтении и записи.
- **Каскадная отмена:** `ctx.Done()` — broadcast.
- **`errgroup`** — для обработки ошибок.
- **`cancel()` при ошибке** — отменяет всё.

---

## 8.8 Tee-канал и bridge-канал

Разберём **tee-канал** и **bridge-канал** — продвинутые паттерны.

### Tee-канал: разветвление

**Tee-канал** — это канал, который **дублирует** данные в два канала.

**Схема:**

```
                ┌──────────┐
           ┌───▶│ Channel1 │
┌──────┐   │    └──────────┘
│Input │───┤
└──────┘   │    ┌──────────┐
           └───▶│ Channel2 │
                └──────────┘
```

**Реализация:**

```go
func tee(ctx context.Context, input <-chan Message) (<-chan Message, <-chan Message) {
    out1 := make(chan Message)
    out2 := make(chan Message)
    
    go func() {
        defer close(out1)
        defer close(out2)
        for msg := range input {
            // Отправляем в оба канала
            select {
            case out1 <- msg:
            case <-ctx.Done():
                return
            }
            select {
            case out2 <- msg:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    return out1, out2
}
```

**Что делает:**

1. Читает из `input`.
2. Отправляет сообщение в `out1`.
3. Отправляет **то же** сообщение в `out2`.
4. Закрывает оба канала при завершении.

**Ключевое:** каждое сообщение **дублируется**.

**Использование:**

- **Логирование + обработка.** Одно сообщение идёт в лог и в обработчик.
- **Метрики + обработка.** Одно сообщение идёт в метрики и в обработчик.

### Bridge-канал: склейка

**Bridge-канал** — это канал, который **разворачивает** канал каналов в один канал.

**Схема:**

```
┌──────────────────┐
│ Channel of       │
│ Channels         │
│ ┌──────────────┐ │
│ │ chan chan T  │ │
│ └──────────────┘ │
└──────────────────┘
        │
        ▼
┌──────────────────┐
│ Bridge           │
│ (разворачивает)  │
└──────────────────┘
        │
        ▼
┌──────────────────┐
│ chan T           │
│ (все элементы)   │
└──────────────────┘
```

**Реализация:**

```go
func bridge(ctx context.Context, chanCh <-chan <-chan Message) <-chan Message {
    out := make(chan Message)
    go func() {
        defer close(out)
        for {
            var ch <-chan Message
            select {
            case c, ok := <-chanCh:
                if !ok {
                    return
                }
                ch = c
            case <-ctx.Done():
                return
            }
            for msg := range ch {
                select {
                case out <- msg:
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    return out
}
```

**Что делает:**

1. Читает из `chanCh` (канал каналов).
2. Для каждого вложенного канала — читает из него и пишет в `out`.
3. Когда `chanCh` закрыт — завершается.

**Использование:**

- **Динамическое добавление стадий.** Каждая стадия — отдельный канал.
- **Fan-in с переменным N.** N каналов приходят в `chanCh`.

### Пример: bridge для динамического fan-in

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Канал каналов
    chanCh := make(chan (<-chan Result))
    
    // Динамически добавляем каналы
    go func() {
        defer close(chanCh)
        for i := 0; i < 10; i++ {
            workerCh := startWorker(ctx, i)
            chanCh <- workerCh
        }
    }()
    
    // Bridge разворачивает
    merged := bridge(ctx, chanCh)
    
    // Читаем из одного канала
    for result := range merged {
        process(result)
    }
}
```

### Сравнение tee и bridge

| Паттерн | Что делает | Использование |
|:---|:---|:---|
| **Tee** | Дублирует в 2 канала | Логирование + обработка |
| **Bridge** | Разворачивает `chan chan T` | Динамический fan-in |

### Аннотация сложности

| Паттерн | Горутин | Time (per element) |
|:---|:---|:---|
| Tee | 1 | ~100-200 нс |
| Bridge | 1 | ~50-100 нс |

### 💡 Практика: как использовать tee и bridge

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Tee** — для дублирования в 2 канала.
2. **Bridge** — для динамического fan-in.

**👍 СТОИТ СДЕЛАТЬ:**

3. **`context` для отмены** в tee и bridge.
4. **Буферизованные выходы** для снижения contention.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **Tee с N каналов** — обобщение для большего N.

**❌ НЕ ДЕЛАЙ:**

6. **Не путай tee с fan-out.** Tee дублирует, fan-out распределяет.
7. **Не путай bridge с merge.** Bridge разворачивает `chan chan T`, merge сливает N каналов.

### Ключевые выводы подглавы 8.8

- **Tee** — дублирование в 2 канала.
- **Bridge** — разворачивание `chan chan T` в `chan T`.
- **Tee** — для логирования + обработки.
- **Bridge** — для динамического fan-in.
- **Оба используют `context` для отмены.**

---

## 8.9 Pipeline vs worker pool: что выбрать

Разберём **когда что использовать**.

### Worker pool

**Worker pool** — N воркеров обрабатывают **однотипные** задачи.

**Когда использовать:**

- **Однотипная обработка.** Все задачи одинаковые.
- **Одна стадия.** Нет цепочки.
- **Простота важна.**

**Пример:** HTTP-запросы к 10 000 URL.

### Pipeline

**Pipeline** — цепочка стадий, каждая обрабатывает **по-своему**.

**Когда использовать:**

- **Разные стадии.** Парсинг → обогащение → запись.
- **Разные N на стадиях.** Parse — 1 воркер, Enrich — 100.
- **Backpressure между стадиями.**

**Пример:** ETL — Kafka → parse → enrich → ClickHouse.

### Гибрид: pipeline из worker pool'ов

**Каждая стадия pipeline** — это worker pool:

```go
func stage(ctx context.Context, input <-chan Input, n int) <-chan Output {
    // Fan-out: N воркеров
    workers := make([]<-chan Output, n)
    for i := 0; i < n; i++ {
        workers[i] = worker(ctx, input)
    }
    // Fan-in: merge
    return merge(ctx, workers...)
}
```

**Что это даёт:**

- **N воркеров** на каждой стадии.
- **Разные N** для разных стадий.
- **Backpressure** между стадиями.

### Сравнение

| Аспект | Worker pool | Pipeline |
|:---|:---|:---|
| Стадий | 1 | N |
| Тип задач | Однотипные | Разные |
| N | Фиксировано | Разное на стадиях |
| Backpressure | Внутри pool | Между стадиями |
| Сложность | Низкая | Средняя |

### Аннотация сложности

| Подход | Горутин | Сложность | Гибкость |
|:---|:---|:---|:---|
| Worker pool | N + 2 | Низкая | Низкая |
| Pipeline | 2N + 5 | Средняя | Высокая |
| Гибрид | зависит | Высокая | Высокая |

### 💡 Практика: как выбирать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Worker pool** — для однотипной обработки.
2. **Pipeline** — для разных стадий.
3. **Гибрид** — pipeline из worker pool'ов.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Разные N** на разных стадиях — по типу задачи.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **Fan-out + fan-in** — если нужна гибкость.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй pipeline для одной стадии.** Worker pool проще.
7. **Не используй worker pool для разных стадий.** Pipeline лучше.

### Ключевые выводы подглавы 8.9

- **Worker pool** — однотипная обработка, одна стадия.
- **Pipeline** — разные стадии, разное N.
- **Гибрид** — pipeline из worker pool'ов.
- **Разные N** на стадиях.

---

## 8.10 Практика Go: pipeline с метриками

Напишем **pipeline с метриками**.

### Полный код

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "sync/atomic"
    "time"
)

type RawMessage struct {
    ID   int
    Data string
}

type ParsedMessage struct {
    ID    int
    Value int
}

type EnrichedMessage struct {
    ID       int
    Value    int
    Enriched string
}

type Metrics struct {
    SourceCount   atomic.Int64
    ParsedCount   atomic.Int64
    EnrichedCount atomic.Int64
    WrittenCount  atomic.Int64
    
    SourceDuration   atomic.Int64
    ParseDuration    atomic.Int64
    EnrichDuration   atomic.Int64
    WriteDuration    atomic.Int64
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    var metrics Metrics
    
    // Stage 0: Source
    source := readSource(ctx, &metrics)
    
    // Stage 1: Parse
    parsed := parseStage(ctx, source, &metrics)
    
    // Stage 2: Enrich (fan-out 10)
    enriched := enrichStage(ctx, parsed, 10, &metrics)
    
    // Stage 3: Write
    written := writeStage(ctx, enriched, &metrics)
    
    // Sink
    for range written {
        // ждём
    }
    
    fmt.Printf("Source:   %d\n", metrics.SourceCount.Load())
    fmt.Printf("Parsed:   %d\n", metrics.ParsedCount.Load())
    fmt.Printf("Enriched: %d\n", metrics.EnrichedCount.Load())
    fmt.Printf("Written:  %d\n", metrics.WrittenCount.Load())
    fmt.Printf("Source avg:  %v\n", avgDuration(&metrics.SourceDuration, &metrics.SourceCount))
    fmt.Printf("Parse avg:   %v\n", avgDuration(&metrics.ParseDuration, &metrics.ParsedCount))
    fmt.Printf("Enrich avg:  %v\n", avgDuration(&metrics.EnrichDuration, &metrics.EnrichedCount))
    fmt.Printf("Write avg:   %v\n", avgDuration(&metrics.WriteDuration, &metrics.WrittenCount))
}

func avgDuration(total *atomic.Int64, count *atomic.Int64) time.Duration {
    c := count.Load()
    if c == 0 {
        return 0
    }
    return time.Duration(total.Load() / c)
}

func readSource(ctx context.Context, metrics *Metrics) <-chan RawMessage {
    out := make(chan RawMessage, 100)
    go func() {
        defer close(out)
        for i := 0; i < 1000; i++ {
            start := time.Now()
            msg := RawMessage{ID: i, Data: fmt.Sprintf("data-%d", i)}
            select {
            case out <- msg:
                metrics.SourceCount.Add(1)
                metrics.SourceDuration.Add(int64(time.Since(start)))
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func parseStage(ctx context.Context, input <-chan RawMessage, metrics *Metrics) <-chan ParsedMessage {
    out := make(chan ParsedMessage, 100)
    go func() {
        defer close(out)
        for msg := range input {
            start := time.Now()
            parsed := ParsedMessage{ID: msg.ID, Value: len(msg.Data)}
            select {
            case out <- parsed:
                metrics.ParsedCount.Add(1)
                metrics.ParseDuration.Add(int64(time.Since(start)))
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func enrichStage(ctx context.Context, input <-chan ParsedMessage, n int, metrics *Metrics) <-chan EnrichedMessage {
    workers := make([]<-chan EnrichedMessage, n)
    for i := 0; i < n; i++ {
        workers[i] = enrichWorker(ctx, input, metrics)
    }
    return merge(ctx, workers...)
}

func enrichWorker(ctx context.Context, input <-chan ParsedMessage, metrics *Metrics) <-chan EnrichedMessage {
    out := make(chan EnrichedMessage)
    go func() {
        defer close(out)
        for msg := range input {
            start := time.Now()
            time.Sleep(1 * time.Millisecond)  // имитация запроса в БД
            enriched := EnrichedMessage{ID: msg.ID, Value: msg.Value, Enriched: "enriched"}
            select {
            case out <- enriched:
                metrics.EnrichedCount.Add(1)
                metrics.EnrichDuration.Add(int64(time.Since(start)))
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func writeStage(ctx context.Context, input <-chan EnrichedMessage, metrics *Metrics) <-chan struct{} {
    out := make(chan struct{})
    go func() {
        defer close(out)
        for range input {
            start := time.Now()
            // имитация записи в ClickHouse
            metrics.WrittenCount.Add(1)
            metrics.WriteDuration.Add(int64(time.Since(start)))
        }
    }()
    return out
}

func merge(ctx context.Context, channels ...<-chan EnrichedMessage) <-chan EnrichedMessage {
    out := make(chan EnrichedMessage)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan EnrichedMessage) {
            defer wg.Done()
            for msg := range c {
                select {
                case out <- msg:
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

### Что демонстрирует

1. **Pipeline из 4 стадий.**
2. **Fan-out 10** на стадии enrich.
3. **Метрики на каждой стадии.**
4. **Отмена через `context.WithTimeout`.**
5. **Backpressure** через буферизованные каналы.

### Пример вывода

```
Source:   1000
Parsed:   1000
Enriched: 1000
Written:  1000
Source avg:  1.2µs
Parse avg:   800ns
Enrich avg:  1.1ms
Write avg:   500ns
```

**Что видно:**

- **Enrich** — bottleneck (1.1 мс).
- **Fan-out 10** — 10 параллельных enrich → пропускная способность 10 / 1.1 мс ≈ 9000/сек.
- **Latency** каждой стадии измерена.

### Аннотация сложности

| Стадия | Time (per msg) | Пропускная способность |
|:---|:---|:---|
| Source | 1.2 µs | 833 000/сек |
| Parse | 800 нс | 1 250 000/сек |
| Enrich | 1.1 мс × 10 воркеров | 9 000/сек |
| Write | 500 нс | 2 000 000/сек |

### 💡 Практика: как измерять pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики на каждой стадии:** count, duration.
2. **`atomic.Int64`** для метрик.
3. **Мониторинг** в реальном времени.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Экспорт в Prometheus** (для production).
5. **Percentile latency** (p50, p95, p99).

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Tracing** через OpenTelemetry.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `Mutex` для метрик** — `atomic` быстрее.
8. **Не игнорируй метрики.** Они показывают bottleneck.

### Ключевые выводы подглавы 8.10

- **Метрики на каждой стадии:** count, duration.
- **`atomic.Int64`** для счётчиков.
- **Fan-out 10** — 10x пропускная способность.
- **Bottleneck** видно по метрикам.
- **Экспорт в Prometheus** — для production.

---

## 8.11 Выводы и типичные ошибки

**Что мы узнали?**

Fan-out — распределение работы от одного источника к N обработчикам. Fan-in — слияние N каналов в один. Fan-out + fan-in — классическая комбинация. Pipeline — цепочка стадий, каждая читает из входа и пишет в выход. Backpressure — медленная стадия замедляет быструю. Размер буфера определяет, как быстро работает backpressure. Закрытие каналов: кто пишет — тот закрывает. Каскадное закрытие. Отмена через `context.Context`. Tee-канал — дублирование. Bridge-канал — разворачивание `chan chan T`. Worker pool vs pipeline: pool для однотипной обработки, pipeline для разных стадий. Гибрид — pipeline из worker pool'ов. Метрики на каждой стадии.

**Типичные ошибки:**

- ❌ **Забыть `close(out)` в стадии.** Утечка горутин.
- ❌ **Закрыть входной канал.** Паника при отправке.
- ❌ **Двойное закрытие.** Паника.
- ❌ **Писать в закрытый канал.** Паника.
- ❌ **Не использовать `select` с `ctx.Done()`.** Отмена не сработает.
- ❌ **Большие буферы.** Backpressure не работает.
- ❌ **Забыть `wg.Wait()` в merge.** `close(out)` не выполнится.
- ❌ **Путать fan-out и fan-in.** Fan-out распределяет, fan-in сливает.
- ❌ **Путать tee и fan-out.** Tee дублирует, fan-out распределяет.
- ❌ **Путать bridge и merge.** Bridge разворачивает `chan chan T`, merge сливает N каналов.
- ❌ **Использовать pipeline для одной стадии.** Worker pool проще.
- ❌ **Не измерять метрики.** Bottleneck не видно.
- ❌ **Игнорировать `ctx.Err()` после отмены.** Теряешь причину.
- ❌ **Создавать N + 1 каналов, когда можно одним.** Overhead.

---

## 8.12 Для быстрого повторения

- **Fan-out** — распределение работы от одного источника к N обработчикам. Один канал — N воркеров.
- **Fan-in** — слияние N каналов в один. N горутин + 1 для `close`.
- **Fan-out + fan-in** — классическая комбинация. Горутин: 2N + 3.
- **Worker pool с одним каналом** — проще: N + 2 горутин.
- **Pipeline** — цепочка стадий. Каждая — `func(ctx, input) <-chan Output`.
- **Стадия** — читает из входа, пишет в выход, закрывает **свой** выход.
- **Backpressure** — медленная стадия замедляет быструю. Размер буфера определяет скорость.
- **Маленький буфер** — быстрый backpressure. **Большой** — отложенный. **100 000+** — не работает.
- **`select` с `default`** — drop вместо backpressure.
- **Закрытие:** кто пишет — тот закрывает. Каскадное.
- **Отмена:** `context.Context` + `select` с `ctx.Done()`.
- **`errgroup`** — для обработки ошибок.
- **Tee-канал** — дублирование в 2 канала.
- **Bridge-канал** — разворачивание `chan chan T`.
- **Worker pool vs pipeline:** pool для однотипной, pipeline для разных стадий.
- **Гибрид** — pipeline из worker pool'ов.
- **Метрики:** count + duration на каждой стадии.
- **`atomic.Int64`** для метрик.

---

## 8.13 Вопросы для самопроверки

1. Что такое fan-out? Как реализуется?
2. Что такое fan-in? Как реализуется?
3. Чем fan-out + fan-in отличается от worker pool с одним каналом?
4. Что такое pipeline? Как реализуется?
5. Как каждая стадия закрывает свой выход?
6. Почему нельзя закрывать входной канал?
7. Что такое backpressure? Как размер буфера влияет на него?
8. Почему большой буфер маскирует проблему?
9. Что делает `select` с `default`? Когда использовать?
10. Как отменить pipeline через `context`?
11. Что делает `errgroup` для pipeline?
12. Что такое tee-канал? Когда использовать?
13. Что такое bridge-канал? Когда использовать?
14. Чем tee отличается от fan-out?
15. Чем bridge отличается от merge?
16. Когда использовать worker pool, а когда pipeline?
17. Что такое гибрид pipeline из worker pool'ов?
18. Как измерять метрики pipeline?

---

## 8.14 Ответы

### Ответ 1

**Fan-out** — распределение работы от одного источника к N обработчикам.

**Реализация:** один канал, N воркеров читают из него. Распределение автоматическое.

```go
for i := 0; i < N; i++ {
    go func() {
        for task := range tasksCh {
            process(task)
        }
    }()
}
```

### Ответ 2

**Fan-in** — слияние N каналов в один.

**Реализация:** N горутин + 1 для close.

```go
func merge(ctx context.Context, channels ...<-chan Result) <-chan Result {
    out := make(chan Result)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan Result) {
            defer wg.Done()
            for r := range c {
                select {
                case out <- r:
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

### Ответ 3

**Fan-out + fan-in:** N каналов + merge. **Горутин:** 2N + 3.

**Worker pool:** один канал результатов. **Горутин:** N + 2.

**Worker pool** проще, быстрее, меньше contention. **Fan-out + fan-in** — для гибкости.

### Ответ 4

**Pipeline** — цепочка стадий, каждая читает из входа и пишет в выход.

**Реализация:** каждая стадия — функция `func(ctx, input) <-chan Output`.

```go
source := readSource(ctx)
parsed := parseStage(ctx, source)
enriched := enrichStage(ctx, parsed, 100)
written := writeStage(ctx, enriched)
```

### Ответ 5

Каждая стадия закрывает **свой выходной** канал через `defer close(out)`. Когда входной канал закрыт — `for range` завершается, `defer` закрывает выход.

```go
func stage(ctx context.Context, input <-chan Input) <-chan Output {
    out := make(chan Output)
    go func() {
        defer close(out)  // ← закрываем свой выход
        for msg := range input {
            select {
            case out <- process(msg):
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

### Ответ 6

**Нельзя закрывать входной канал**, потому что его **читает другая стадия**. Если закрыть — предыдущая стадия получит панику при отправке.

**Кто пишет — тот закрывает.**

### Ответ 7

**Backpressure** — медленная стадия замедляет быструю.

**Размер буфера:**
- Маленький (1-10) — быстрый backpressure.
- Большой (1000) — отложенный.
- 100 000+ — не работает.

### Ответ 8

**Большой буфер маскирует проблему**, потому что producer и воркеры не блокируются, пока буфер не полон. Но если producer быстрее collector надолго — буферы переполнятся.

### Ответ 9

**`select` с `default`** — если буфер полон, сообщение **пропускается**.

**Когда использовать:** если сообщения **можно терять** (метрики, логи).

### Ответ 10

**Отмена pipeline через `context`:**

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

source := readSource(ctx)
parsed := parseStage(ctx, source)
// ...

// В каждой стадии:
select {
case <-ctx.Done():
    return
case msg := <-input:
    // ...
}
```

### Ответ 11

**`errgroup` для pipeline:**

```go
g, ctx := errgroup.WithContext(context.Background())

g.Go(func() error { return runSource(ctx, ...) })
g.Go(func() error { return runParse(ctx, ...) })
// ...

if err := g.Wait(); err != nil {
    // ошибка
}
```

При **первой** ошибке — отменяет `ctx`.

### Ответ 12

**Tee-канал** — дублирует данные в 2 канала.

**Когда использовать:** логирование + обработка, метрики + обработка.

### Ответ 13

**Bridge-канал** — разворачивает `chan chan T` в `chan T`.

**Когда использовать:** динамический fan-in, динамическое добавление стадий.

### Ответ 14

**Tee** дублирует **каждое** сообщение в **оба** канала.

**Fan-out** распределяет сообщения **между** воркерами (каждое сообщение идёт **одному** воркеру).

### Ответ 15

**Bridge** разворачивает `chan chan T` (канал каналов) в `chan T`.

**Merge** сливает **N каналов** в один.

### Ответ 16

**Worker pool** — для однотипной обработки, одна стадия.

**Pipeline** — для разных стадий, разное N.

**Гибрид** — pipeline из worker pool'ов.

### Ответ 17

**Гибрид pipeline из worker pool'ов** — каждая стадия pipeline реализована как worker pool:

```go
func stage(ctx context.Context, input <-chan Input, n int) <-chan Output {
    workers := make([]<-chan Output, n)
    for i := 0; i < n; i++ {
        workers[i] = worker(ctx, input)
    }
    return merge(ctx, workers...)
}
```

Разные N на разных стадиях.

### Ответ 18

**Метрики pipeline:**
- **Count** на каждой стадии.
- **Duration** на каждой стадии.
- **`atomic.Int64`** для счётчиков.
- **Экспорт в Prometheus** для production.
- **Bottleneck** видно по метрикам.

---

## 8.15 Куда идти дальше?

Мы разобрали pipeline: fan-out, fan-in, стадии, backpressure, отмена, tee, bridge. Теперь мы умеем строить конвейеры данных.

Но остаётся **практический вопрос**: как ограничить **скорость** обработки? Что если внешний сервис имеет rate limit? Что если нужно защититься от каскадных отказов?

- **Как ограничить скорость?** Rate limiter, token bucket, leaky bucket. → **Глава 9: Продвинутые паттерны — semaphore, rate limiter, circuit breaker.**
- **Как обрабатывать ошибки в конкурентном коде?** `errgroup`, `multierror`, отмена при первой ошибке. → **Глава 10: Обработка ошибок в конкурентном коде.**
- **Как корректно завершить сервис?** Сигналы ОС, `context`, ожидание завершения. → **Глава 11: Graceful shutdown.**

---

## 8.16 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Fan-out** | Распределение работы | Один канал — N воркеров |
| **Fan-in** | Слияние каналов | N горутин + 1 для close |
| **Fan-out + fan-in** | Классическая комбинация | 2N + 3 горутин |
| **Worker pool** | N воркеров | N + 2 горутин |
| **Pipeline** | Цепочка стадий | Каждая — `func(ctx, input) <-chan Output` |
| **Стадия** | Обработка | Читает из входа, пишет в выход, закрывает свой выход |
| **Backpressure** | Замедление производителя | Размер буфера определяет скорость |
| **Маленький буфер** | 1-10 | Быстрый backpressure |
| **Большой буфер** | 1000+ | Отложенный backpressure |
| **Очень большой** | 100 000+ | Не работает |
| **`select` с `default`** | Drop | Для некритичных данных |
| **Закрытие** | Кто пишет — тот закрывает | Каскадное |
| **Отмена** | `context.Context` | `select` с `ctx.Done()` |
| **`errgroup`** | Обработка ошибок | Отменяет при первой ошибке |
| **Tee-канал** | Дублирование | В 2 канала |
| **Bridge-канал** | Разворачивание | `chan chan T` → `chan T` |
| **Worker pool vs pipeline** | Однотипное vs разное | Pool для однотипной, pipeline для разных стадий |
| **Гибрид** | Pipeline из worker pool'ов | Разное N на стадиях |
| **Метрики** | Count + duration | `atomic.Int64` |

🔀 **Ключевая идея:** Fan-out — распределение работы, fan-in — слияние каналов. Pipeline — цепочка стадий, каждая читает из входа и пишет в выход. Backpressure — медленная стадия замедляет быструю; размер буфера определяет скорость. Закрытие: кто пишет — тот закрывает; каскадное. Отмена через `context.Context` + `select` с `ctx.Done()`. Tee дублирует, bridge разворачивает. Worker pool для однотипной обработки, pipeline для разных стадий; гибрид — pipeline из worker pool'ов. Метрики: count + duration на каждой стадии.