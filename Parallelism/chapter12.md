# 🚦 Глава 12: Semaphore — ограничение параллелизма

**Что вы узнаете:**
- Что такое semaphore и какую задачу он решает.
- Чем semaphore отличается от worker pool.
- Как построить простейший semaphore на каналах.
- Как добавить отмену через `context`.
- Что такое взвешенный semaphore и когда он нужен.
- Как избежать утечек и deadlock при использовании semaphore.
- Как комбинировать semaphore с generator, fan-out, pipeline.

**После прочтения вы сможете:**
- Построить semaphore с нуля.
- Ограничивать число одновременных операций.
- Использовать взвешенный semaphore для разнородных задач.
- Правильно освобождать слоты.
- Выбирать между semaphore и worker pool.
- Комбинировать semaphore с другими паттернами.

---

## Содержание

- [12.0 Пролог: слишком много одновременных запросов](#120-пролог-слишком-много-одновременных-запросов)
- [12.1 Что такое semaphore](#121-что-такое-semaphore)
- [12.2 Простейший semaphore](#122-простейший-semaphore)
- [12.3 Semaphore с context](#123-semaphore-с-context)
- [12.4 Semaphore vs worker pool](#124-semaphore-vs-worker-pool)
- [12.5 Взвешенный semaphore](#125-взвешенный-semaphore)
- [12.6 Semaphore как паттерн](#126-semaphore-как-паттерн)
- [12.7 В связке с другими паттернами](#127-в-связке-с-другими-паттернами)
- [12.8 Практика Go: ограничение параллелизма](#128-практика-go-ограничение-параллелизма)
- [12.9 Выводы и типичные ошибки](#129-выводы-и-типичные-ошибки)
- [12.10 Для быстрого повторения](#1210-для-быстрого-повторения)
- [12.11 Вопросы для самопроверки](#1211-вопросы-для-самопроверки)
- [12.12 Ответы](#1212-ответы)
- [12.13 Куда идти дальше?](#1213-куда-идти-дальше)
- [12.14 Чек-лист](#1214-чек-лист)

---

## 12.0 Пролог: слишком много одновременных запросов

У нас есть сервис, который ходит за обогащением в другой сервис. Работает так: получает 10 000 ID, делает HTTP-запрос на каждый.

Пишем наивно:

```go
func fetchAll(ids []int) {
    var wg sync.WaitGroup
    for _, id := range ids {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            resp, err := http.Get(fmt.Sprintf("https://api.example.com/users/%d", id))
            if err != nil {
                return
            }
            defer resp.Body.Close()
            // обработка
        }(id)
    }
    wg.Wait()
}
```

Работает. Но замечаем проблему: **10 000 одновременных соединений**. Внешний сервис начинает отвечать 429 (Too Many Requests). Наш сервис исчерпывает file descriptors. Память растёт.

Хочется **ограничить** число одновременных запросов — не 10 000, а, скажем, 20. Остальные пусть ждут.

Можно было бы запустить **фиксированное число воркеров** (worker pool, Глава 13). Но тогда нужно переделывать логику: канал задач, воркеры, fan-in. Это оправдано, если задач действительно много.

Но здесь другое: **у нас уже есть список задач** (`ids`). Мы просто хотим **взять слот** перед запросом и **освободить** после. Не нужно переделывать архитектуру — нужен **примитив ограничения параллелизма**.

Этот примитив — **semaphore**.

> **Мост к следующим главам:** semaphore — основа worker pool (Глава 13) и pipeline (Глава 11). Он ограничивает параллелизм, но не переделывает архитектуру. Понимание semaphore даёт понимание, **как устроен worker pool изнутри**.

---

## 12.1 Что такое semaphore

**Semaphore** — примитив, который ограничивает **число одновременных операций**.

### Идея

У нас есть **N слотов**. Каждая операция:

1. **Захватывает** слот (если есть свободный).
2. Выполняется.
3. **Освобождает** слот.

Если все слоты заняты — новая операция **ждёт**.

### Схема

```
Семафор N = 3:

  ┌─────────────────────────────┐
  │  Слоты: [1] [2] [3]         │
  └─────────────────────────────┘
  
Операция 1: захватить [1] ──► работает
Операция 2: захватить [2] ──► работает
Операция 3: захватить [3] ──► работает
Операция 4: захватить ──► ждёт (все заняты)
Операция 5: захватить ──► ждёт

Операция 1: освободить [1]
Операция 4: захватить [1] ──► работает
```

### Semaphore vs worker pool

**Worker pool** — N воркеров создаются **один раз** и **переиспользуются**. Задачи идут через канал.

**Semaphore** — N слотов. Каждая операция захватывает слот и создаёт горутину.

| Аспект | Worker pool | Semaphore |
|:---|:---|:---|
| Горутин | N (фиксировано) | N + задачи |
| Создание горутин | N раз | На каждую задачу |
| Память | N × 2.3 КБ | (N + задачи) × 2.3 КБ |
| Переиспользование | Да | Нет |
| Сложность | Средняя | Простая |
| Гибкость | Ограничена | Высокая |

**Правило:**

- **Worker pool** — для многих однотипных задач.
- **Semaphore** — для ограничения доступа к ресурсу.

### Когда использовать semaphore

**1. Много задач, ограничение параллелизма.**

- 10 000 HTTP-запросов, но не больше 20 одновременно.
- 1000 файлов, но не больше 10 читаются одновременно.

**2. Ограничение доступа к ресурсу.**

- Не более N одновременных запросов к БД.
- Не более N одновременных запросов к внешнему API.

**3. Простой код.**

- Не нужно переделывать архитектуру.
- Просто взять слот перед операцией.

### Когда НЕ использовать semaphore

**1. Много однотипных задач с большим объёмом.**

Worker pool эффективнее — переиспользует воркеров.

**2. Нужна очередь задач.**

Semaphore не хранит очередь. Если операция не может захватить слот — она **ждёт**.

**3. Нужны метрики по воркерам.**

Worker pool лучше — метрики на каждого воркера.

### 💡 Практика: как думать о semaphore

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Semaphore — для ограничения доступа к ресурсу.**
2. **Worker pool — для многих однотипных задач.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Для HTTP — semaphore на число соединений.**
4. **Для БД — semaphore на размер пула.**

**❌ НЕ ДЕЛАЙ:**

5. **Не путай semaphore с worker pool.**
6. **Не используй semaphore для многих задач с большим объёмом.**

---

## 12.2 Простейший semaphore

Начнём с самого простого — буферизованный канал как семафор.

### Идея

**Буферизованный канал** — идеальный семафор:

- **Размер буфера** — число слотов.
- **Запись в канал** — захват слота.
- **Чтение из канала** — освобождение слота.

### Реализация

```go
sem := make(chan struct{}, 10)  // 10 слотов

sem <- struct{}{}  // захватить слот
// ... работа ...
<-sem  // освободить слот
```

**Что происходит:**

- `sem <- struct{}{}` блокируется, если все слоты заняты.
- `<-sem` освобождает слот.

### Потребитель

```go
func fetchAll(ids []int, maxConcurrent int) {
    sem := make(chan struct{}, maxConcurrent)
    
    var wg sync.WaitGroup
    for _, id := range ids {
        id := id
        
        sem <- struct{}{}  // захватить слот
        wg.Add(1)
        go func() {
            defer wg.Done()
            defer func() { <-sem }()  // освободить слот
            
            resp, err := http.Get(fmt.Sprintf("https://api.example.com/users/%d", id))
            if err != nil {
                return
            }
            defer resp.Body.Close()
            // обработка
        }()
    }
    wg.Wait()
}
```

**Что происходит:**

- Не более 10 одновременных запросов.
- Каждая горутина захватывает слот перед запросом.
- Освобождает после.

### Полный код

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func processAll(items []int, maxConcurrent int) {
    sem := make(chan struct{}, maxConcurrent)
    
    var wg sync.WaitGroup
    for _, item := range items {
        item := item
        
        sem <- struct{}{}
        wg.Add(1)
        go func() {
            defer wg.Done()
            defer func() { <-sem }()
            
            fmt.Printf("Processing %d\n", item)
            time.Sleep(100 * time.Millisecond)
        }()
    }
    wg.Wait()
}

func main() {
    items := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    
    start := time.Now()
    processAll(items, 3)  // 3 одновременно
    fmt.Printf("Elapsed: %v\n", time.Since(start))
}
```

**Пример вывода:**

```
Processing 1
Processing 3
Processing 2
Processing 4
...
Elapsed: 400ms  ← 10 задач / 3 одновременно × 100 мс ≈ 400 мс
```

**Что видно:** одновременно 3 задачи. Общее время — 400 мс (не 1000 мс).

### Схема

```
sem := make(chan struct{}, 3):

  goroutine 1: sem <- {} ──► работает ──► <-sem
  goroutine 2: sem <- {} ──► работает ──► <-sem
  goroutine 3: sem <- {} ──► работает ──► <-sem
  goroutine 4: sem <- {} ──► ждёт
  goroutine 5: sem <- {} ──► ждёт
  ...
```

### 💡 Практика: как использовать простой semaphore

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`make(chan struct{}, N)`** — семафор на N.
2. **`defer func() { <-sem }()`** — освобождение.
3. **Захват до `go func()`.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **N = число одновременных операций.**

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай освобождать слот.** Утечка.
6. **Не захватывай слот внутри горутины без `defer`.**

---

## 12.3 Semaphore с context

Простейший semaphore не умеет отменяться. Если все слоты заняты и `ctx` отменён — операция **зависнет**.

### Проблема

```go
sem <- struct{}{}  // ← блокируется, пока не будет слот
```

**Что происходит:** если слот никогда не освободится — горутина зависнет навсегда.

### Решение: select с ctx.Done()

```go
select {
case sem <- struct{}{}:
    // захватили слот
case <-ctx.Done():
    return  // отмена
}
```

**Что происходит:**

- Если есть слот — захватываем.
- Если `ctx` отменён — выходим.
- Не блокируемся навсегда.

### Полный пример

```go
func processAllCtx(ctx context.Context, items []int, maxConcurrent int) error {
    sem := make(chan struct{}, maxConcurrent)
    
    var wg sync.WaitGroup
    for _, item := range items {
        item := item
        
        select {
        case sem <- struct{}{}:
        case <-ctx.Done():
            wg.Wait()
            return ctx.Err()
        }
        
        wg.Add(1)
        go func() {
            defer wg.Done()
            defer func() { <-sem }()
            
            process(ctx, item)
        }()
    }
    wg.Wait()
    return nil
}
```

**Что происходит:**

- Захват слота — через `select`.
- Если `ctx` отменён — ждём уже запущенные горутины и возвращаем ошибку.

### Потребитель

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
    defer cancel()
    
    items := make([]int, 100)
    for i := range items {
        items[i] = i
    }
    
    if err := processAllCtx(ctx, items, 3); err != nil {
        fmt.Println("error:", err)
    }
}
```

**Что происходит:**

- Через 500 мс `ctx` отменяется.
- Захват слотов возвращает ошибку.
- Уже запущенные горутины завершаются.

### Схема

```
Захват слота:

  select {
  case sem <- {}:      ← слот захвачен
  case <-ctx.Done():   ← отмена
    return
  }

Если все слоты заняты и ctx отменён:

  ctx.Done() закрыт ──► select идёт в case <-ctx.Done()
                    ──► return
```

### 💡 Практика: как добавить context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`select` с `ctx.Done()` при захвате:**
   ```go
   select {
   case sem <- struct{}{}:
   case <-ctx.Done():
       return ctx.Err()
   }
   ```

2. **`ctx` — первый аргумент.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **`wg.Wait()` перед возвратом ошибки** — дождаться горутин.

**❌ НЕ ДЕЛАЙ:**

4. **Не блокируйся на `sem <- struct{}{}` без `select`.**

---

## 12.4 Semaphore vs worker pool

Разберём, **когда что использовать**.

### Worker pool

**Идея:** N воркеров читают из **одного** канала задач.

```go
tasksCh := make(chan Task, 100)

for i := 0; i < N; i++ {
    go func() {
        for task := range tasksCh {
            process(task)
        }
    }()
}

for _, task := range tasks {
    tasksCh <- task
}
close(tasksCh)
```

**Плюсы:**

- Горутины **переиспользуются**.
- Задачи идут через канал.
- **Backpressure** — через буфер.

**Минусы:**

- Нужен канал задач.
- Сложнее, если задачи разнородные.

### Semaphore

**Идея:** N слотов, каждая задача захватывает слот и создаёт горутину.

```go
sem := make(chan struct{}, N)

for _, task := range tasks {
    sem <- struct{}{}
    go func(task Task) {
        defer func() { <-sem }()
        process(task)
    }(task)
}
```

**Плюсы:**

- **Проще.**
- Не нужен канал задач.
- Каждая задача — своя горутина.

**Минусы:**

- Горутины создаются на каждую задачу.
- Больше памяти.

### Сравнение

| Аспект | Worker pool | Semaphore |
|:---|:---|:---|
| Горутин | N | N + задачи |
| Память | N × 2.3 КБ | (N + задачи) × 2.3 КБ |
| Создание горутин | N раз | На каждую задачу |
| Канал задач | Нужен | Не нужен |
| Backpressure | Через канал | Через слоты |
| Сложность | Средняя | Простая |

### Что выбрать

**Worker pool:**

- **Много задач** (10 000+).
- **Однотипные задачи.**
- **Важна память.**

**Semaphore:**

- **Умеренное число задач** (100–1000).
- **Разнородные задачи.**
- **Простота кода.**

**Практическое правило:**

- **< 1000 задач** — semaphore.
- **> 1000 задач** — worker pool.
- **CPU-bound** — worker pool с N = GOMAXPROCS.

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

**Оба работают.** Semaphore проще, worker pool эффективнее для больших объёмов.

### 💡 Практика: как выбирать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Semaphore — для < 1000 задач.**
2. **Worker pool — для > 1000 задач.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Semaphore — для разнородных задач.**
4. **Worker pool — для CPU-bound.**

**❌ НЕ ДЕЛАЙ:**

5. **Не используй semaphore для миллионов задач.**
6. **Не используй worker pool для 100 задач** — semaphore проще.

---

## 12.5 Взвешенный semaphore

Иногда операции требуют **разного** количества ресурсов. Лёгкий запрос — 1 единица. Тяжёлый — 5 единиц. Всего 10.

**Взвешенный semaphore** — каждая операция захватывает **N слотов**.

### Проблема обычного semaphore

```go
sem := make(chan struct{}, 10)

sem <- struct{}{}  // ← 1 слот
```

**Все операции захватывают 1 слот.** Лёгкий и тяжёлый конкурируют одинаково.

### Решение: golang.org/x/sync/semaphore

```go
import "golang.org/x/sync/semaphore"

sem := semaphore.NewWeighted(10)  // 10 единиц

sem.Acquire(ctx, 5)  // захватить 5 единиц
// ... работа ...
sem.Release(5)  // освободить 5 единиц
```

### API

**`NewWeighted(n)`** — создать семафор на n единиц.

**`Acquire(ctx, weight)`** — захватить weight единиц. Блокируется, если нет. Возвращает ошибку, если `ctx` отменён.

**`Release(weight)`** — освободить weight единиц.

**`TryAcquire(weight)`** — неблокирующая попытка.

### Пример

```go
package main

import (
    "context"
    "fmt"
    "time"
    
    "golang.org/x/sync/semaphore"
)

func main() {
    sem := semaphore.NewWeighted(10)
    ctx := context.Background()
    
    tasks := []struct {
        name   string
        weight int64
    }{
        {"light-1", 1},
        {"light-2", 1},
        {"heavy-1", 5},
        {"light-3", 1},
        {"heavy-2", 5},
    }
    
    var wg sync.WaitGroup
    for _, task := range tasks {
        task := task
        wg.Add(1)
        go func() {
            defer wg.Done()
            
            if err := sem.Acquire(ctx, task.weight); err != nil {
                return
            }
            defer sem.Release(task.weight)
            
            fmt.Printf("Processing %s (weight=%d)\n", task.name, task.weight)
            time.Sleep(100 * time.Millisecond)
        }()
    }
    wg.Wait()
}
```

**Что происходит:**

- `heavy-1` и `heavy-2` занимают по 5 единиц.
- `light-*` занимают по 1 единице.
- Когда `heavy-1` работает (5 единиц), остаётся 5 единиц для `light`.
- `heavy-2` ждёт, пока освободится 5 единиц.

### Как устроен взвешенный semaphore

Внутри — `Weighted` с полями:

```go
type Weighted struct {
    size    int64        // всего единиц
    cur     int64        // текущее использование
    mu      sync.Mutex
    waiters list.List    // ждущие горутины
}
```

**При `Acquire`:**

1. Захватить `mu`.
2. Если `cur + weight <= size` — увеличить `cur`, вернуть.
3. Иначе — добавить горутину в `waiters`, ждать.
4. При `Release` — разбудить тех, кто может захватить.

### Стоимость

| Операция | Time |
|:---|:---|
| `NewWeighted` | ~50–100 нс |
| `Acquire` (есть место) | ~100–200 нс |
| `Acquire` (нет места) | gopark |
| `Release` | ~100–200 нс |

### 💡 Практика: как использовать взвешенный semaphore

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`semaphore.NewWeighted(N)`** — для N единиц.
2. **`Acquire(ctx, weight)`** — захват с весом.
3. **`defer sem.Release(weight)`** — освобождение.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Взвешенный semaphore для разнородных задач.**
5. **Обычный semaphore на каналах** — для однотипных.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй взвешенный semaphore, если веса одинаковы.**

---

## 12.6 Semaphore как паттерн

Semaphore — не просто примитив. Это **паттерн**, который реализуется через каналы. Разберём это.

### Каналы как универсальный примитив

**Буферизованный канал** — это semaphore:

- **Размер буфера** = число слотов.
- **Запись** = захват.
- **Чтение** = освобождение.

Это **идиоматический** способ ограничения параллелизма в Go.

### Паттерн: semaphore на каналах

```go
type Semaphore chan struct{}

func NewSemaphore(n int) Semaphore {
    return make(Semaphore, n)
}

func (s Semaphore) Acquire(ctx context.Context) error {
    select {
    case s <- struct{}{}:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func (s Semaphore) Release() {
    <-s
}
```

**Что даёт:** инкапсуляция логики. Пользователь не знает, что внутри канал.

### Потребитель

```go
sem := NewSemaphore(10)

for _, task := range tasks {
    if err := sem.Acquire(ctx); err != nil {
        break
    }
    go func(task Task) {
        defer sem.Release()
        process(task)
    }(task)
}
```

### Паттерн: semaphore для внешнего API

```go
type APIClient struct {
    client *http.Client
    sem    Semaphore
}

func NewAPIClient(maxConcurrent int) *APIClient {
    return &APIClient{
        client: &http.Client{Timeout: 10 * time.Second},
        sem:    NewSemaphore(maxConcurrent),
    }
}

func (c *APIClient) Get(ctx context.Context, url string) (*http.Response, error) {
    if err := c.sem.Acquire(ctx); err != nil {
        return nil, err
    }
    defer c.sem.Release()
    
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    return c.client.Do(req)
}
```

**Что даёт:** клиент сам ограничивает параллелизм. Пользователь не думает об этом.

### Паттерн: semaphore для БД

```go
type DB struct {
    sem Semaphore
}

func NewDB(maxConns int) *DB {
    return &DB{sem: NewSemaphore(maxConns)}
}

func (db *DB) Query(ctx context.Context, query string) (Result, error) {
    if err := db.sem.Acquire(ctx); err != nil {
        return Result{}, err
    }
    defer db.sem.Release()
    
    return db.doQuery(ctx, query)
}
```

**Что даёт:** не более N одновременных запросов к БД.

### 💡 Практика: как инкапсулировать semaphore

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Инкапсулируй semaphore в структуру.**
2. **`Acquire(ctx)` / `Release()`** — методы.
3. **`ctx` — для отмены.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Semaphore внутри клиента API/БД.**

**❌ НЕ ДЕЛАЙ:**

5. **Не разбрасывай `sem <- struct{}{}` по всему коду.**
6. **Не забывай инкапсулировать логику.**

---

## 12.7 В связке с другими паттернами

Semaphore комбинируется с другими паттернами.

### Generator + semaphore

Generator отдаёт задачи, semaphore ограничивает обработку:

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 100)
    sem := NewSemaphore(10)
    
    var wg sync.WaitGroup
    for v := range source {
        if err := sem.Acquire(ctx); err != nil {
            break
        }
        v := v
        wg.Add(1)
        go func() {
            defer wg.Done()
            defer sem.Release()
            process(v)
        }()
    }
    wg.Wait()
}
```

### Fan-out + semaphore

Semaphore может ограничить fan-out **динамически**:

```go
func fanOutWithSem(ctx context.Context, input <-chan int, maxConcurrent int) <-chan int {
    out := make(chan int)
    sem := NewSemaphore(maxConcurrent)
    
    var wg sync.WaitGroup
    for v := range input {
        if err := sem.Acquire(ctx); err != nil {
            break
        }
        v := v
        wg.Add(1)
        go func() {
            defer wg.Done()
            defer sem.Release()
            
            select {
            case out <- process(v):
            case <-ctx.Done():
            }
        }()
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

### Pipeline + semaphore

Semaphore в стадии pipeline:

```go
func enrichStage(ctx context.Context, input <-chan Parsed, maxConcurrent int) <-chan Enriched {
    out := make(chan Enriched)
    sem := NewSemaphore(maxConcurrent)
    
    var wg sync.WaitGroup
    for v := range input {
        if err := sem.Acquire(ctx); err != nil {
            break
        }
        v := v
        wg.Add(1)
        go func() {
            defer wg.Done()
            defer sem.Release()
            
            enriched := enrich(v)
            select {
            case out <- enriched:
            case <-ctx.Done():
            }
        }()
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

### Semaphore + rate limiter

Semaphore ограничивает **параллелизм**, rate limiter — **скорость**:

```go
func main() {
    sem := NewSemaphore(10)              // не более 10 одновременно
    limiter := rate.NewLimiter(100, 10)  // не более 100/сек
    
    for _, task := range tasks {
        if err := sem.Acquire(ctx); err != nil {
            break
        }
        if err := limiter.Wait(ctx); err != nil {
            sem.Release()
            break
        }
        go func(task Task) {
            defer sem.Release()
            process(task)
        }(task)
    }
}
```

**Что даёт:** одновременно не более 10, и не более 100/сек.

### 💡 Практика: как комбинировать semaphore

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Generator + semaphore** — для чтения с ограничением.
2. **Fan-out + semaphore** — для динамического ограничения.
3. **Pipeline + semaphore** — для стадии с ограничением.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Semaphore + rate limiter** — для внешних API.

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай `ctx`.**

---

## 12.8 Практика Go: ограничение параллелизма

Разберём **три примера**.

### Пример 1: простой semaphore

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func processAll(items []int, maxConcurrent int) {
    sem := make(chan struct{}, maxConcurrent)
    
    var wg sync.WaitGroup
    for _, item := range items {
        item := item
        
        sem <- struct{}{}
        wg.Add(1)
        go func() {
            defer wg.Done()
            defer func() { <-sem }()
            
            fmt.Printf("Processing %d\n", item)
            time.Sleep(100 * time.Millisecond)
        }()
    }
    wg.Wait()
}

func main() {
    items := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    
    start := time.Now()
    processAll(items, 3)
    fmt.Printf("Elapsed: %v\n", time.Since(start))
}
```

**Пример вывода:**

```
Processing 2
Processing 1
Processing 3
Processing 4
Processing 5
Processing 6
Processing 7
Processing 8
Processing 9
Processing 10
Elapsed: 400ms
```

### Пример 2: semaphore с context

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

func processAllCtx(ctx context.Context, items []int, maxConcurrent int) error {
    sem := make(chan struct{}, maxConcurrent)
    
    var wg sync.WaitGroup
    for _, item := range items {
        item := item
        
        select {
        case sem <- struct{}{}:
        case <-ctx.Done():
            wg.Wait()
            return ctx.Err()
        }
        
        wg.Add(1)
        go func() {
            defer wg.Done()
            defer func() { <-sem }()
            
            select {
            case <-ctx.Done():
                return
            case <-time.After(100 * time.Millisecond):
                fmt.Printf("Processing %d\n", item)
            }
        }()
    }
    wg.Wait()
    return nil
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 250*time.Millisecond)
    defer cancel()
    
    items := make([]int, 100)
    for i := range items {
        items[i] = i
    }
    
    if err := processAllCtx(ctx, items, 3); err != nil {
        fmt.Println("error:", err)
    }
}
```

**Пример вывода:**

```
Processing 0
Processing 1
Processing 2
Processing 3
Processing 4
Processing 5
error: context deadline exceeded
```

### Пример 3: взвешенный semaphore

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
    
    "golang.org/x/sync/semaphore"
)

type Task struct {
    Name   string
    Weight int64
}

func main() {
    sem := semaphore.NewWeighted(10)
    ctx := context.Background()
    
    tasks := []Task{
        {"light-1", 1},
        {"light-2", 1},
        {"heavy-1", 5},
        {"light-3", 1},
        {"heavy-2", 5},
    }
    
    var wg sync.WaitGroup
    for _, task := range tasks {
        task := task
        wg.Add(1)
        go func() {
            defer wg.Done()
            
            if err := sem.Acquire(ctx, task.Weight); err != nil {
                return
            }
            defer sem.Release(task.Weight)
            
            fmt.Printf("Processing %s (weight=%d)\n", task.Name, task.Weight)
            time.Sleep(100 * time.Millisecond)
        }()
    }
    wg.Wait()
}
```

**Пример вывода:**

```
Processing light-2 (weight=1)
Processing light-1 (weight=1)
Processing heavy-1 (weight=5)
Processing light-3 (weight=1)
Processing heavy-2 (weight=5)
```

**Что видно:** `heavy-2` начал работу только после того, как `heavy-1` освободил 5 единиц.

### 💡 Практика: как использовать semaphore

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`make(chan struct{}, N)`** — для простого.
2. **`semaphore.NewWeighted(N)`** — для взвешенного.
3. **`select` с `ctx.Done()`** — при захвате.
4. **`defer` освобождать.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Инкапсулируй в структуру.**
6. **Semaphore для внешних API/БД.**

**❌ НЕ ДЕЛАЙ:**

7. **Не забывай освобождать.**
8. **Не блокируйся без `ctx`.**

---

## 12.9 Выводы и типичные ошибки

**Что мы узнали?**

Semaphore — примитив для ограничения числа одновременных операций. **Простейший** — буферизованный канал `make(chan struct{}, N)`. Захват — `sem <- struct{}{}`, освобождение — `<-sem`. `context` для отмены. **Взвешенный** — `golang.org/x/sync/semaphore` для разнородных задач. Semaphore **проще** worker pool, но worker pool **эффективнее** для больших объёмов. Semaphore комбинируется с generator, fan-out, pipeline, rate limiter.

**Типичные ошибки:**

- ❌ **Забыть освободить слот.** Утечка.
- ❌ **Блокироваться на `sem <- struct{}{}` без `ctx`.** Зависание.
- ❌ **Захватить слот внутри горутины** (race).
- ❌ **Не использовать `defer` при освобождении.**
- ❌ **Путать semaphore с worker pool.**
- ❌ **Использовать semaphore для миллионов задач.**
- ❌ **Взвешенный semaphore для одинаковых весов.**
- ❌ **Не инкапсулировать semaphore в структуру.**

---

## 12.10 Для быстрого повторения

- **Semaphore** — ограничивает параллелизм.
- **На каналах:** `make(chan struct{}, N)`.
- **Захват:** `sem <- struct{}{}`. **Освобождение:** `<-sem`.
- **`select` с `ctx.Done()`** — отмена.
- **Взвешенный semaphore:** `golang.org/x/sync/semaphore`.
- **`Acquire(ctx, weight)` / `Release(weight)`.**
- **Semaphore** — для < 1000 задач. **Worker pool** — для > 1000.
- **Semaphore** — проще, worker pool — эффективнее.
- **Инкапсулируй в структуру** для клиента API/БД.
- **Semaphore + generator** — чтение с ограничением.
- **Semaphore + fan-out** — динамическое ограничение.
- **Semaphore + rate limiter** — параллелизм + скорость.

---

## 12.11 Вопросы для самопроверки

1. Что такое semaphore? Какую задачу решает?
2. Чем semaphore отличается от worker pool?
3. Как построить простейший semaphore?
4. Зачем `context` в semaphore?
5. Что такое взвешенный semaphore? Когда нужен?
6. Как комбинировать semaphore с generator?
7. Что будет, если забыть освободить слот?
8. Что выбрать — semaphore или worker pool?

---

## 12.12 Ответы

### Ответ 1

**Semaphore** — примитив для ограничения числа **одновременных** операций. Решает задачу: **много задач, но не больше N одновременно**.

### Ответ 2

**Semaphore:** N слотов, каждая задача — своя горутина. **Worker pool:** N воркеров, переиспользуются, задачи через канал.

**Semaphore** проще, worker pool эффективнее для больших объёмов.

### Ответ 3

```go
sem := make(chan struct{}, N)

sem <- struct{}{}  // захват
defer func() { <-sem }()  // освобождение
```

### Ответ 4

**`context`** позволяет не блокироваться навсегда, если все слоты заняты. Без него, если слот никогда не освободится, горутина **зависнет**.

**Решение:** `select` с `ctx.Done()`.

### Ответ 5

**Взвешенный semaphore** — каждая операция захватывает **N** единиц. Для **разнородных** задач (лёгкие + тяжёлые).

**Реализация:** `golang.org/x/sync/semaphore`.

### Ответ 6

**Semaphore + generator:**

```go
source := numbersCtx(ctx, 100)
sem := NewSemaphore(10)

var wg sync.WaitGroup
for v := range source {
    if err := sem.Acquire(ctx); err != nil {
        break
    }
    v := v
    wg.Add(1)
    go func() {
        defer wg.Done()
        defer sem.Release()
        process(v)
    }()
}
wg.Wait()
```

Generator отдаёт задачи, semaphore ограничивает обработку.

### Ответ 7

**Утечка.** Слот остаётся занятым навсегда. После N забытых освобождений **новые операции зависнут** — все слоты заняты.

### Ответ 8

**Worker pool:**

- > 1000 задач.
- Однотипные задачи.
- Важна память.

**Semaphore:**

- < 1000 задач.
- Разнородные задачи.
- Простота кода.

---

## 12.13 Куда идти дальше?

Мы разобрали semaphore — ограничение параллелизма. Теперь мы умеем ограничивать число одновременных операций.

Но semaphore — это частный случай более общего паттерна — **worker pool**. Как построить worker pool с нуля?

- **Как построить worker pool?** → **Глава 13: Worker pool.**
- **Как ограничить скорость?** → **Глава 14: Rate limiter.**
- **Как защититься от отказов?** → **Глава 15: Circuit breaker.**

---

## 12.14 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Semaphore** | Ограничение параллелизма | N слотов |
| **`make(chan struct{}, N)`** | Простейший | N слотов |
| **`sem <- struct{}{}`** | Захват | Блокируется, если нет |
| **`<-sem`** | Освобождение | Сразу после работы |
| **`select` с `ctx.Done()`** | Отмена | При захвате |
| **Взвешенный** | `x/sync/semaphore` | Для разнородных задач |
| **`Acquire(ctx, weight)`** | Захват с весом | — |
| **`Release(weight)`** | Освобождение | — |
| **Semaphore vs worker pool** | < 1000 vs > 1000 | Простота vs эффективность |
| **Инкапсуляция** | В структуру | Для API/БД |
| **Semaphore + generator** | Чтение с ограничением | — |
| **Semaphore + rate limiter** | Параллелизм + скорость | — |

🚦 **Ключевая идея:** Semaphore — примитив для ограничения параллелизма. **Простейший** — буферизованный канал `make(chan struct{}, N)`. Захват — `sem <- struct{}{}`, освобождение — `<-sem`. **`context`** — для отмены, `select` с `ctx.Done()`. **Взвешенный** — `golang.org/x/sync/semaphore` для разнородных задач. **Semaphore** — для < 1000 задач, **worker pool** — для > 1000. **Инкапсулируй** в структуру для клиента API/БД. Комбинируется с generator, fan-out, pipeline, rate limiter.