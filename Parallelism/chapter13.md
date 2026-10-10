# 🏊 Глава 13: Worker pool — ограниченный параллелизм

**Что вы узнаете:**
- Что такое worker pool и какую задачу он решает.
- Чем worker pool отличается от semaphore.
- Как построить простейший worker pool.
- Как добавить отмену через `context`.
- Как собирать результаты — три подхода.
- Как обрабатывать ошибки.
- Как динамически менять число воркеров.
- Как комбинировать worker pool с generator, pipeline, rate limiter.

**После прочтения вы сможете:**
- Построить worker pool с нуля.
- Собирать результаты из N воркеров.
- Правильно закрывать каналы.
- Обрабатывать ошибки и отмену.
- Выбирать между worker pool и semaphore.
- Комбинировать worker pool с другими паттернами.

---

## Содержание

- [13.0 Пролог: 10 000 задач, 20 воркеров](#130-пролог-10-000-задач-20-воркеров)
- [13.1 Что такое worker pool](#131-что-такое-worker-pool)
- [13.2 Простейший worker pool](#132-простейший-worker-pool)
- [13.3 Worker pool с context](#133-worker-pool-с-context)
- [13.4 Сбор результатов](#134-сбор-результатов)
- [13.5 Обработка ошибок](#135-обработка-ошибок)
- [13.6 Динамическое число воркеров](#136-динамическое-число-воркеров)
- [13.7 Worker pool vs semaphore](#137-worker-pool-vs-semaphore)
- [13.8 В связке с другими паттернами](#138-в-связке-с-другими-паттернами)
- [13.9 Практика Go: worker pool с метриками](#139-практика-go-worker-pool-с-метриками)
- [13.10 Выводы и типичные ошибки](#1310-выводы-и-типичные-ошибки)
- [13.11 Для быстрого повторения](#1311-для-быстрого-повторения)
- [13.12 Вопросы для самопроверки](#1312-вопросы-для-самопроверки)
- [13.13 Ответы](#1313-ответы)
- [13.14 Куда идти дальше?](#1314-куда-идти-дальше)
- [13.15 Чек-лист](#1315-чек-лист)

---

## 13.0 Пролог: 10 000 задач, 20 воркеров

У нас есть сервис, который обрабатывает 10 000 URL. Каждый URL — HTTP-запрос, обработка тела, запись в БД. Всё это может занять **секунды**.

Пишем наивно:

```go
func processAll(urls []string) {
    var wg sync.WaitGroup
    for _, url := range urls {
        wg.Add(1)
        go func(u string) {
            defer wg.Done()
            process(u)
        }(url)
    }
    wg.Wait()
}
```

Запускаем. Через 30 секунд — OOM. Потому что **10 000 горутин** одновременно делают HTTP-запросы. 10 000 открытых соединений, 10 000 буферов, 10 000 стеков. Внешние сервисы отвечают 429. File descriptors исчерпаны.

Хочется обрабатывать **20 URL одновременно**. Остальные — в очереди.

**Первое решение:** semaphore (Глава 12).

```go
sem := make(chan struct{}, 20)

for _, url := range urls {
    sem <- struct{}{}
    go func(u string) {
        defer func() { <-sem }()
        process(u)
    }(url)
}
```

Работает. Но каждая задача **создаёт горутину**. 10 000 задач → 10 000 горутин. Пусть не одновременно, но всё равно много.

Хочется **переиспользовать** воркеров. Создать **20 горутин** один раз и пусть они читают задачи из очереди.

Это и есть **worker pool**.

> **Мост к следующим главам:** worker pool — обобщение semaphore (Глава 12) и основа для pipeline (Глава 11). Понимание worker pool даёт понимание, **как строить production-обработчики**.

---

## 13.1 Что такое worker pool

**Worker pool** — это паттерн, при котором **N воркеров** (горутин) обрабатывают задачи из **общего канала**.

### Схема

```
┌─────────────────────────────────────────────────────────────┐
│                    Producer (главная горутина)               │
│                                                              │
│  for _, task := range tasks {                               │
│      tasksCh <- task                                        │
│  }                                                           │
│  close(tasksCh)                                             │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       │ tasksCh (канал задач)
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    Worker Pool (N воркеров)                  │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      ┌──────────┐ │
│  │ Worker 1 │  │ Worker 2 │  │ Worker 3 │ ...  │ Worker N │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘      └────┬─────┘ │
│       │             │             │                  │       │
│       └─────────────┴─────────────┴──────────────────┘       │
│                              │                                │
│                              │ resultsCh (канал результатов)  │
│                              ▼                                │
└─────────────────────────────────────────────────────────────┘
                       │
                       │ resultsCh
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    Collector (главная горутина)              │
│                                                              │
│  for result := range resultsCh {                            │
│      process(result)                                        │
│  }                                                           │
└─────────────────────────────────────────────────────────────┘
```

### Ключевые свойства

1. **Ограниченный параллелизм.** N воркеров.
2. **Переиспользование.** Воркеры не создаются на каждую задачу.
3. **Backpressure.** Если канал задач полон — producer ждёт.
4. **Контроль ресурсов.** N TCP-соединений, N буферов, N file descriptors.

### Сколько воркеров

**Зависит от задачи:**

| Тип задачи | N воркеров | Почему |
|:---|:---|:---|
| CPU-bound | `GOMAXPROCS` | Больше — конкуренция за CPU |
| I/O-bound | 10–100 | Зависит от latency внешнего сервиса |
| Смешанная | `GOMAXPROCS` × 2–4 | Компромисс |

**Эмпирическое правило:**

```
N = GOMAXPROCS × (1 + wait_time / compute_time)
```

**Пример:** HTTP-запрос: 1 мс CPU + 100 мс ожидания сети.

```
N = 8 × (1 + 100/1) = 808
```

**Но:** это теоретический максимум. На практике — 20–100, потому что внешний сервис может не выдержать 800 одновременных запросов.

### Когда использовать worker pool

**1. Много задач, ограничение параллелизма.**

- 10 000 URL, 20 воркеров.
- 1000 заказов, 10 обработчиков.

**2. Однотипные задачи.**

- Все задачи — HTTP-запросы.
- Все задачи — чтение файлов.

**3. Задачи I/O-bound.**

- HTTP, БД, файлы.
- CPU-bound тоже работает, но часто semaphore проще.

### Когда НЕ использовать worker pool

**1. Мало задач (< 100).**

Semaphore проще.

**2. Разнородные задачи.**

Разные типы задач — разные каналы.

**3. Задачи зависимы.**

Если задача B требует результата A — нужен pipeline.

### 💡 Практика: как думать о worker pool

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Worker pool — для многих однотипных задач.**
2. **N по типу задачи.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **N = GOMAXPROCS** для CPU-bound. **N = 10–100** для I/O-bound.

**❌ НЕ ДЕЛАЙ:**

4. **Не создавай горутину на каждую задачу.**
5. **Не используй worker pool для 100 задач** — semaphore проще.

---

## 13.2 Простейший worker pool

Начнём с самого простого — N воркеров читают из канала, ничего не возвращают.

### Реализация

```go
func workerPool(tasks []Task, numWorkers int) {
    tasksCh := make(chan Task, len(tasks))
    
    // Запускаем воркеров
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for task := range tasksCh {
                process(task)
            }
        }()
    }
    
    // Producer
    for _, task := range tasks {
        tasksCh <- task
    }
    close(tasksCh)
    
    wg.Wait()
}
```

**Что происходит:**

1. Создаём канал задач.
2. Запускаем N воркеров.
3. Каждый воркер читает из `tasksCh` в цикле `for range`.
4. Producer пишет задачи.
5. `close(tasksCh)` — воркеры завершаются.
6. `wg.Wait()` — ждём всех.

### Полный код

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Task struct {
    ID int
}

func workerPool(tasks []Task, numWorkers int) {
    tasksCh := make(chan Task, len(tasks))
    
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        i := i
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            for task := range tasksCh {
                fmt.Printf("Worker %d: processing %d\n", workerID, task.ID)
                time.Sleep(100 * time.Millisecond)
            }
        }(i)
    }
    
    for _, task := range tasks {
        tasksCh <- task
    }
    close(tasksCh)
    
    wg.Wait()
}

func main() {
    tasks := make([]Task, 10)
    for i := range tasks {
        tasks[i] = Task{ID: i}
    }
    
    start := time.Now()
    workerPool(tasks, 3)
    fmt.Printf("Elapsed: %v\n", time.Since(start))
}
```

**Пример вывода:**

```
Worker 2: processing 2
Worker 0: processing 0
Worker 1: processing 1
Worker 2: processing 3
Worker 0: processing 4
...
Elapsed: 400ms
```

**Что видно:** одновременно 3 задачи. Общее время — 400 мс.

### Схема

```
Producer:
  tasksCh <- Task{0}
  tasksCh <- Task{1}
  ...
  close(tasksCh)

Workers:
  Worker 0: for task := range tasksCh { process(task) }
  Worker 1: for task := range tasksCh { process(task) }
  Worker 2: for task := range tasksCh { process(task) }

Producer закрывает tasksCh → воркеры завершаются
```

### Проблема: producer блокируется

```go
for _, task := range tasks {
    tasksCh <- task  // ← блокируется, если буфер полон и воркеры заняты
}
```

**Что происходит:**

- Канал `tasksCh` имеет буфер `len(tasks)`.
- Producer пишет все задачи **мгновенно**.
- Не блокируется.

**Но:** если буфер маленький, producer **блокируется**. Это **backpressure** — producer ждёт, пока воркеры заберут задачи.

### 💡 Практика: как писать простой worker pool

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`tasksCh := make(chan Task, len(tasks))`** — буфер на все задачи.
2. **N воркеров читают из одного канала.**
3. **`close(tasksCh)` после producer.**
4. **`wg.Wait()` в конце.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Передавай `workerID`** — для отладки и метрик.

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `close(tasksCh)`.**
7. **Не создавай канал на каждого воркера.**

---

## 13.3 Worker pool с context

Простейший worker pool не умеет отменяться. Если внешний сервис завис — воркеры ждут.

### Проблема

```go
for task := range tasksCh {
    process(task)  // ← может зависнуть
}
```

**Что происходит:** если `process` зависнет, воркер не завершится. `wg.Wait()` зависнет.

### Решение: context

```go
func workerPoolCtx(ctx context.Context, tasks []Task, numWorkers int) error {
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
                    process(ctx, task)
                }
            }
        }()
    }
    
    // Producer с отменой
    for _, task := range tasks {
        select {
        case tasksCh <- task:
        case <-ctx.Done():
            close(tasksCh)
            wg.Wait()
            return ctx.Err()
        }
    }
    close(tasksCh)
    
    wg.Wait()
    return ctx.Err()
}
```

**Что происходит:**

- Воркеры проверяют `ctx.Done()` в `select`.
- Producer проверяет `ctx.Done()` при отправке.
- При отмене — воркеры завершаются, `wg.Wait()` возвращается.

### Полный код

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

type Task struct {
    ID int
}

func workerPoolCtx(ctx context.Context, tasks []Task, numWorkers int) error {
    tasksCh := make(chan Task, len(tasks))
    
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        i := i
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case task, ok := <-tasksCh:
                    if !ok {
                        return
                    }
                    fmt.Printf("Worker %d: processing %d\n", workerID, task.ID)
                    process(ctx, task)
                }
            }
        }(i)
    }
    
    for _, task := range tasks {
        select {
        case tasksCh <- task:
        case <-ctx.Done():
            close(tasksCh)
            wg.Wait()
            return ctx.Err()
        }
    }
    close(tasksCh)
    
    wg.Wait()
    return ctx.Err()
}

func process(ctx context.Context, task Task) {
    select {
    case <-ctx.Done():
        return
    case <-time.After(100 * time.Millisecond):
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 250*time.Millisecond)
    defer cancel()
    
    tasks := make([]Task, 100)
    for i := range tasks {
        tasks[i] = Task{ID: i}
    }
    
    if err := workerPoolCtx(ctx, tasks, 3); err != nil {
        fmt.Println("error:", err)
    }
}
```

**Пример вывода:**

```
Worker 0: processing 0
Worker 1: processing 1
Worker 2: processing 2
...
error: context deadline exceeded
```

### Схема

```
Producer:                       Workers:
  select {                        select {
  case tasksCh <- task:           case <-ctx.Done():
  case <-ctx.Done():                return
    close(tasksCh)                case task, ok := <-tasksCh:
    wg.Wait()                       if !ok { return }
    return                          process(ctx, task)
  }                               }
```

### 💡 Практика: как добавить context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`select` с `ctx.Done()` в воркерах.**
2. **`select` с `ctx.Done()` в producer.**
3. **`close(tasksCh)` при отмене.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Проверять `ok` при получении из канала.**

**❌ НЕ ДЕЛАЙ:**

5. **Не блокируйся на `tasksCh <- task` без `select`.**
6. **Не забывай `close` при отмене.**

---

## 13.4 Сбор результатов

Worker pool без результатов — редкость. Обычно нужно **собирать результаты** от воркеров.

### Подход 1: канал результатов (рекомендуемый)

```go
func workerPoolWithResults(ctx context.Context, tasks []Task, numWorkers int) <-chan Result {
    tasksCh := make(chan Task, len(tasks))
    resultsCh := make(chan Result, len(tasks))
    
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
                    result := process(ctx, task)
                    select {
                    case resultsCh <- result:
                    case <-ctx.Done():
                        return
                    }
                }
            }
        }()
    }
    
    // Producer
    go func() {
        defer close(tasksCh)
        for _, task := range tasks {
            select {
            case tasksCh <- task:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    // Закрытие resultsCh после всех воркеров
    go func() {
        wg.Wait()
        close(resultsCh)
    }()
    
    return resultsCh
}
```

**Что происходит:**

- Воркеры пишут результат в `resultsCh`.
- Collector читает из `resultsCh`.
- `close(resultsCh)` — после всех воркеров.

### Потребитель

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    tasks := make([]Task, 10)
    for i := range tasks {
        tasks[i] = Task{ID: i}
    }
    
    for result := range workerPoolWithResults(ctx, tasks, 3) {
        fmt.Printf("Result: %+v\n", result)
    }
}
```

### Схема

```
Producer ──► tasksCh ──► Workers ──► resultsCh ──► Collector
                                                    
                                              wg.Wait() + close
```

### Подход 2: Mutex + слайс

```go
var (
    mu      sync.Mutex
    results []Result
)

// В воркерах:
result := process(task)
mu.Lock()
results = append(results, result)
mu.Unlock()
```

**Плюсы:** просто.

**Минусы:** Mutex contention при N воркерах. Все результаты в памяти.

### Подход 3: atomic + предварительно выделенный слайс

```go
results := make([]Result, len(tasks))
var idx atomic.Int64

// В воркерах:
result := process(task)
i := idx.Add(1) - 1
results[i] = result  // разные индексы, нет гонки
```

**Плюсы:** нет блокировок.

**Минусы:** нужно знать число задач заранее. Порядок не гарантирован.

### Сравнение

| Подход | Гонки | Backpressure | Память | Порядок |
|:---|:---|:---|:---|:---|
| Канал результатов | Нет | Да | O(1) в буфере | Не гарантирован |
| Mutex + слайс | Нет (Mutex) | Нет | O(N) | Не гарантирован |
| Atomic + слайс | Нет | Нет | O(N) | Не гарантирован |

**Рекомендация:** **канал результатов** — для большинства случаев.

### 💡 Практика: как собирать результаты

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Канал результатов** — для большинства случаев.
2. **`close(resultsCh)`** после `wg.Wait()`.
3. **Отдельная горутина** для `close`.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Mutex + слайс** — если нужно обработать после.
5. **Atomic + слайс** — для производительности.

**❌ НЕ ДЕЛАЙ:**

6. **Не пиши в общий слайс без синхронизации.**

---

## 13.5 Обработка ошибок

Разберём **обработку ошибок** в worker pool.

### Простой способ: Result с ошибкой

```go
type Result struct {
    TaskID int
    Value  string
    Err    error
}

// В воркерах:
result := Result{TaskID: task.ID}
result.Value, result.Err = process(task)
resultsCh <- result
```

**Что происходит:** каждая задача несёт ошибку. Collector проверяет.

### Потребитель

```go
for result := range workerPoolWithResults(ctx, tasks, 3) {
    if result.Err != nil {
        log.Printf("task %d failed: %v", result.TaskID, result.Err)
        continue
    }
    processResult(result.Value)
}
```

### Отмена при первой ошибке

```go
func workerPoolCtx(ctx context.Context, tasks []Task, numWorkers int) error {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    
    tasksCh := make(chan Task, len(tasks))
    errCh := make(chan error, numWorkers)
    
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
                    if err := process(task); err != nil {
                        select {
                        case errCh <- err:
                            cancel()  // отменяем всех
                        case <-ctx.Done():
                        }
                        return
                    }
                }
            }
        }()
    }
    
    // Producer
    go func() {
        defer close(tasksCh)
        for _, task := range tasks {
            select {
            case tasksCh <- task:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    wg.Wait()
    close(errCh)
    
    for err := range errCh {
        return err  // первая ошибка
    }
    return nil
}
```

**Что происходит:**

- При первой ошибке — `cancel()`.
- Все воркеры видят `ctx.Done()` и завершаются.
- Возвращаем первую ошибку.

### Сбор всех ошибок

```go
var (
    mu   sync.Mutex
    errs []error
)

// В воркерах:
if err := process(task); err != nil {
    mu.Lock()
    errs = append(errs, fmt.Errorf("task %d: %w", task.ID, err))
    mu.Unlock()
}

// После wg.Wait():
return errors.Join(errs...)
```

**Что происходит:** собираем все ошибки через `errors.Join`.

### 💡 Практика: как обрабатывать ошибки

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`Result{Value, Err}`** — для каждой задачи.
2. **`cancel()` при первой ошибке** — если критично.
3. **`errors.Join`** — для сбора всех ошибок.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`errCh`** — для ошибок из воркеров.

**❌ НЕ ДЕЛАЙ:**

5. **Не игнорируй ошибки в воркерах.**
6. **Не пиши в `errs` без `Mutex`.**

---

## 13.6 Динамическое число воркеров

Иногда нагрузка переменная: днём — много, ночью — мало. Число воркеров хочется адаптировать.

### Подход: добавление воркеров

```go
type Pool struct {
    tasksCh chan Task
    ctx     context.Context
    wg      sync.WaitGroup
    mu      sync.Mutex
    workers int
}

func (p *Pool) AddWorker() {
    p.mu.Lock()
    defer p.mu.Unlock()
    
    p.wg.Add(1)
    go func() {
        defer p.wg.Done()
        for {
            select {
            case <-p.ctx.Done():
                return
            case task, ok := <-p.tasksCh:
                if !ok {
                    return
                }
                process(task)
            }
        }
    }()
    p.workers++
}
```

**Что происходит:** добавляем воркеров по мере необходимости.

### Уменьшение воркеров: stopCh

**Проблема:** уменьшить число воркеров сложнее. Воркеры «спят» на `<-tasksCh` — их не разбудить.

**Решение:** отдельный канал `stopCh`:

```go
type Pool struct {
    tasksCh chan Task
    stopCh  chan struct{}  // сигнал остановки
    ctx     context.Context
    wg      sync.WaitGroup
    mu      sync.Mutex
    workers int
}

func (p *Pool) RemoveWorker() {
    p.mu.Lock()
    defer p.mu.Unlock()
    
    if p.workers > 0 {
        p.stopCh <- struct{}{}  // сигнал одному воркеру
        p.workers--
    }
}

func (p *Pool) worker() {
    defer p.wg.Done()
    for {
        select {
        case <-p.ctx.Done():
            return
        case <-p.stopCh:
            return
        case task, ok := <-p.tasksCh:
            if !ok {
                return
            }
            process(task)
        }
    }
}
```

**Что происходит:** `RemoveWorker` отправляет сигнал в `stopCh`. Один воркер видит `stopCh` и завершается.

### Схема

```
Pool:
  workers = 5
  stopCh = make(chan struct{})

RemoveWorker():
  stopCh <- struct{}{}  ──► один воркер завершается
  workers = 4

Worker:
  select {
  case <-ctx.Done(): return
  case <-stopCh: return       ← сюда
  case task := <-tasksCh: process(task)
  }
```

### Когда динамическое N оправдано

**Оправдано:**

- **Нагрузка сильно варьируется.**
- **Внешний сервис имеет rate limit.**

**Не оправдано:**

- **Стабильная нагрузка.** Фиксированное N проще.
- **Малая система.** Overhead не окупается.

### 💡 Практика: как делать динамическое N

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`AddWorker`** — через `go p.worker()`.
2. **`RemoveWorker`** — через `stopCh`.
3. **`ctx` — для массовой отмены.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики нагрузки** — CPU, длина очереди.
5. **Пороги** — когда добавлять/удалять.

**❌ НЕ ДЕЛАЙ:**

6. **Не меняй N слишком часто.**
7. **Не забывай про backpressure.**

---

## 13.7 Worker pool vs semaphore

Разберём **когда что использовать**.

### Сравнение

| Аспект | Worker pool | Semaphore |
|:---|:---|:---|
| Горутин | N (фиксировано) | N + задачи |
| Создание горутин | N раз | На каждую задачу |
| Память | N × 2.3 КБ | (N + задачи) × 2.3 КБ |
| Переиспользование | Да | Нет |
| Канал задач | Нужен | Не нужен |
| Backpressure | Через канал | Через слоты |
| Сложность | Средняя | Простая |

### Что выбрать

**Worker pool:**

- **Много задач** (1000+).
- **Однотипные задачи.**
- **Важна память.**
- **CPU-bound задачи.**

**Semaphore:**

- **Умеренное число задач** (100–1000).
- **Разнородные задачи.**
- **Простота кода.**
- **Ограничение доступа к ресурсу.**

**Практическое правило:**

- **< 1000 задач** — semaphore.
- **> 1000 задач** — worker pool.

### Пример: HTTP-запросы

**Semaphore:**

```go
sem := make(chan struct{}, 20)
for _, url := range urls {
    sem <- struct{}{}
    go func(u string) {
        defer func() { <-sem }()
        http.Get(u)
    }(url)
}
```

**Worker pool:**

```go
urlCh := make(chan string, 100)
for i := 0; i < 20; i++ {
    go func() {
        for url := range urlCh {
            http.Get(url)
        }
    }()
}
for _, url := range urls {
    urlCh <- url
}
close(urlCh)
```

**Оба работают.** Semaphore проще для 100 URL. Worker pool эффективнее для 10 000.

### 💡 Практика: как выбирать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Semaphore — для < 1000 задач.**
2. **Worker pool — для > 1000 задач.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Semaphore — для разнородных задач.**
4. **Worker pool — для CPU-bound.**

**❌ НЕ ДЕЛАЙ:**

5. **Не используй semaphore для миллионов задач.**
6. **Не используй worker pool для 100 задач.**

---

## 13.8 В связке с другими паттернами

Worker pool — центральный паттерн для production-обработчиков. Разберём связки.

### Generator + worker pool

Generator отдаёт задачи, worker pool обрабатывает:

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 100)  // generator
    results := workerPoolSource(ctx, source, 5)  // worker pool
    
    for r := range results {
        fmt.Println(r)
    }
}

func workerPoolSource(ctx context.Context, source <-chan int, numWorkers int) <-chan int {
    resultsCh := make(chan int)
    
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for v := range source {
                select {
                case resultsCh <- v * 2:
                case <-ctx.Done():
                    return
                }
            }
        }()
    }
    
    go func() {
        wg.Wait()
        close(resultsCh)
    }()
    
    return resultsCh
}
```

### Worker pool + fan-in

Результаты от нескольких worker pool'ов сливаются:

```go
pool1 := workerPoolSource(ctx, source1, 5)
pool2 := workerPoolSource(ctx, source2, 5)
merged := mergeCtx(ctx, pool1, pool2)

for v := range merged {
    fmt.Println(v)
}
```

### Worker pool + pipeline

Worker pool как стадия pipeline:

```go
source := numbersCtx(ctx, 100)
parsed := parseStage(ctx, source)
enriched := workerPoolStage(ctx, parsed, 10)  // worker pool
written := writeStage(ctx, enriched)
```

### Worker pool + rate limiter

Ограничение скорости в воркерах:

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

### Worker pool + circuit breaker

Circuit breaker защищает от сбоев внешнего сервиса:

```go
func worker(ctx context.Context, tasksCh <-chan Task, breaker *gobreaker.CircuitBreaker) {
    for {
        select {
        case <-ctx.Done():
            return
        case task, ok := <-tasksCh:
            if !ok {
                return
            }
            _, err := breaker.Execute(func() (interface{}, error) {
                return nil, process(ctx, task)
            })
            if err != nil {
                // обработка ошибки
            }
        }
    }
}
```

### 💡 Практика: как комбинировать worker pool

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Generator + worker pool** — источник + обработка.
2. **Worker pool + fan-in** — слияние результатов.
3. **Worker pool + pipeline** — стадия pipeline.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Worker pool + rate limiter** — ограничение скорости.
5. **Worker pool + circuit breaker** — защита от сбоев.

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `context`.**

---

## 13.9 Практика Go: worker pool с метриками

Разберём **полный worker pool с метриками**.

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

type Task struct {
    ID int
}

type Metrics struct {
    Processed atomic.Int64
    Failed    atomic.Int64
    Active    atomic.Int64
}

func workerPoolWithMetrics(ctx context.Context, tasks []Task, numWorkers int) *Metrics {
    metrics := &Metrics{}
    tasksCh := make(chan Task, len(tasks))
    
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        i := i
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case task, ok := <-tasksCh:
                    if !ok {
                        return
                    }
                    metrics.Active.Add(1)
                    start := time.Now()
                    err := process(ctx, task)
                    duration := time.Since(start)
                    metrics.Active.Add(-1)
                    
                    if err != nil {
                        metrics.Failed.Add(1)
                        continue
                    }
                    metrics.Processed.Add(1)
                    _ = duration
                }
            }
        }(i)
    }
    
    // Producer
    go func() {
        defer close(tasksCh)
        for _, task := range tasks {
            select {
            case tasksCh <- task:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    // Мониторинг
    go func() {
        ticker := time.NewTicker(500 * time.Millisecond)
        defer ticker.Stop()
        for range ticker.C {
            fmt.Printf("Processed: %d, Failed: %d, Active: %d\n",
                metrics.Processed.Load(),
                metrics.Failed.Load(),
                metrics.Active.Load())
        }
    }()
    
    return metrics
}

func process(ctx context.Context, task Task) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-time.After(100 * time.Millisecond):
        if task.ID%10 == 0 {
            return fmt.Errorf("task %d failed", task.ID)
        }
        return nil
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    tasks := make([]Task, 100)
    for i := range tasks {
        tasks[i] = Task{ID: i}
    }
    
    start := time.Now()
    metrics := workerPoolWithMetrics(ctx, tasks, 10)
    
    time.Sleep(2 * time.Second)
    cancel()
    
    fmt.Printf("\nTotal time: %v\n", time.Since(start))
    fmt.Printf("Processed: %d\n", metrics.Processed.Load())
    fmt.Printf("Failed: %d\n", metrics.Failed.Load())
}
```

### Пример вывода

```
Processed: 25, Failed: 3, Active: 10
Processed: 50, Failed: 5, Active: 10
Processed: 70, Failed: 7, Active: 10

Total time: 2.001s
Processed: 90
Failed: 10
```

**Что видно:**

- 10 воркеров, 10 активных горутин.
- 90 обработано, 10 упало (каждая 10-я задача).
- Мониторинг в реальном времени.

### 💡 Практика: как измерять worker pool

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики:** processed, failed, active.
2. **`atomic.Int64`** для счётчиков.
3. **Мониторинг в реальном времени.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Экспорт в Prometheus** для production.
5. **Percentile latency** (p50, p95, p99).

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `Mutex` для метрик.**
7. **Не игнорируй метрики.**

---

## 13.10 Выводы и типичные ошибки

**Что мы узнали?**

Worker pool — паттерн для ограничения параллелизма с переиспользованием воркеров. N воркеров читают из одного канала задач. `context` для отмены. Сбор результатов — через канал результатов (рекомендуется), Mutex + слайс или atomic + слайс. Обработка ошибок — `Result{Err}`, `cancel()` при первой ошибке, `errors.Join` для всех. Динамическое N — через `AddWorker` / `RemoveWorker`. Worker pool vs semaphore: < 1000 задач — semaphore, > 1000 — worker pool. Комбинируется с generator, fan-in, pipeline, rate limiter, circuit breaker.

**Типичные ошибки:**

- ❌ **Забыть `close(tasksCh)`.** Воркеры не завершатся.
- ❌ **Не использовать `context`.** Утечка при отмене.
- ❌ **Писать в `resultsCh` без `select`.** Воркеры зависнут.
- ❌ **`close(resultsCh)` в основной горутине.** Deadlock.
- ❌ **Писать в общий слайс без синхронизации.** Гонка.
- ❌ **Забыть `wg.Wait()`.** Воркеры не завершатся.
- ❌ **Динамическое N без `stopCh`.** Воркеры не завершатся.
- ❌ **Использовать worker pool для 100 задач.** Semaphore проще.
- ❌ **Использовать semaphore для миллиона задач.** Worker pool эффективнее.
- ❌ **Не мониторить метрики.**

---

## 13.11 Для быстрого повторения

- **Worker pool:** N воркеров + канал задач + канал результатов.
- **N:** CPU-bound → `GOMAXPROCS`, I/O-bound → 10–100.
- **`tasksCh := make(chan Task, len(tasks))`** — буфер на все.
- **`close(tasksCh)`** после producer.
- **`select` с `ctx.Done()`** в воркерах.
- **Сбор результатов:** канал (рекомендуется), Mutex + слайс, atomic + слайс.
- **`close(resultsCh)`** после `wg.Wait()`, в отдельной горутине.
- **Обработка ошибок:** `Result{Err}`, `cancel()`, `errors.Join`.
- **Динамическое N:** `AddWorker` / `RemoveWorker` через `stopCh`.
- **Worker pool vs semaphore:** < 1000 — semaphore, > 1000 — worker pool.
- **Generator + worker pool** — источник + обработка.
- **Worker pool + pipeline** — стадия pipeline.
- **Метрики:** processed, failed, active.

---

## 13.12 Вопросы для самопроверки

1. Что такое worker pool? Какую задачу решает?
2. Чем worker pool отличается от semaphore?
3. Как построить простейший worker pool?
4. Зачем `close(tasksCh)`?
5. Почему `close(resultsCh)` в отдельной горутине?
6. Назови три подхода к сбору результатов.
7. Как обрабатывать ошибки в worker pool?
8. Что выбрать — worker pool или semaphore?

---

## 13.13 Ответы

### Ответ 1

**Worker pool** — паттерн: N воркеров (горутин) обрабатывают задачи из общего канала. Решает задачу: **много задач, ограничение параллелизма, переиспользование воркеров**.

### Ответ 2

**Worker pool:** N воркеров, переиспользуются, задачи через канал. **Semaphore:** N слотов, каждая задача — своя горутина.

**Worker pool** эффективнее для больших объёмов. **Semaphore** проще.

### Ответ 3

```go
tasksCh := make(chan Task, len(tasks))

var wg sync.WaitGroup
for i := 0; i < N; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for task := range tasksCh {
            process(task)
        }
    }()
}

for _, task := range tasks {
    tasksCh <- task
}
close(tasksCh)
wg.Wait()
```

### Ответ 4

**`close(tasksCh)`** сигнализирует воркерам, что задачи закончились. Воркеры видят закрытие в `for range` и завершаются.

**Без `close`:** воркеры зависнут на чтении из канала.

### Ответ 5

**`wg.Wait()` блокируется.** Если `close(resultsCh)` в основной горутине — она не вернёт `resultsCh`. Потребитель не сможет читать. **Deadlock.**

**Отдельная горутина:** основная возвращает `resultsCh` сразу, отдельная ждёт завершения воркеров и закрывает.

### Ответ 6

**Три подхода:**
1. **Канал результатов** — рекомендуется.
2. **Mutex + слайс** — просто.
3. **Atomic + слайс** — для производительности.

### Ответ 7

**Обработка ошибок:**

- `Result{Value, Err}` — для каждой задачи.
- `cancel()` при первой ошибке — если критично.
- `errors.Join` — для сбора всех ошибок.
- `errCh` — отдельный канал.

### Ответ 8

**Worker pool:**

- > 1000 задач.
- Однотипные задачи.
- Важна память.
- CPU-bound.

**Semaphore:**

- < 1000 задач.
- Разнородные задачи.
- Простота кода.

---

## 13.14 Куда идти дальше?

Мы разобрали worker pool — ограниченный параллелизм с переиспользованием воркеров. Теперь мы умеем строить production-обработчики.

Но иногда нужно **ограничить скорость**, а не параллелизм. Например, внешний сервис разрешает 100 запросов в секунду.

- **Как ограничить скорость?** → **Глава 14: Rate limiter.**
- **Как защититься от отказов?** → **Глава 15: Circuit breaker.**
- **Как повторять неудачные операции?** → **Глава 16: Retry.**

---

## 13.15 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Worker pool** | N воркеров + канал | Ограничение параллелизма |
| **`tasksCh`** | Канал задач | Буфер на все задачи |
| **N воркеров** | Читают из одного канала | Распределение автоматическое |
| **`close(tasksCh)`** | После producer | Воркеры завершаются |
| **`select` с `ctx.Done()`** | Отмена | В воркерах и producer |
| **`resultsCh`** | Канал результатов | Рекомендуется |
| **`close(resultsCh)`** | После `wg.Wait()` | В отдельной горутине |
| **Сбор результатов** | 3 подхода | Канал, Mutex, atomic |
| **Обработка ошибок** | `Result{Err}` | `cancel()`, `errors.Join` |
| **Динамическое N** | `AddWorker` / `RemoveWorker` | Через `stopCh` |
| **Worker pool vs semaphore** | > 1000 vs < 1000 | Эффективность vs простота |
| **N = GOMAXPROCS** | CPU-bound | Число ядер |
| **N = 10–100** | I/O-bound | Зависит от latency |
| **Метрики** | processed, failed, active | `atomic.Int64` |

🏊 **Ключевая идея:** Worker pool — N воркеров + канал задач + канал результатов. N воркеров читают из одного канала — распределение автоматическое. `close(tasksCh)` после producer. `context` для отмены. Сбор результатов — канал (рекомендуется), Mutex, atomic. `close(resultsCh)` в отдельной горутине после `wg.Wait()`. Обработка ошибок — `Result{Err}`, `cancel()` при первой ошибке, `errors.Join`. Динамическое N — `AddWorker` / `RemoveWorker`. **Worker pool** — для > 1000 задач; **semaphore** — для < 1000. N = GOMAXPROCS для CPU-bound, N = 10–100 для I/O-bound.