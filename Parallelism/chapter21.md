# 📦 Глава 21: Batch — группировка операций

**Что вы узнаете:**
- Что такое batch и какую задачу он решает.
- Чем batch по размеру отличается от batch по времени.
- Как построить batch с нуля.
- Как сделать гибридный batch (размер + время).
- Как добавить отмену через `context`.
- Как делать flush при shutdown.
- Как обрабатывать backpressure.
- Как комбинировать batch с worker pool, pipeline, rate limiter.

**После прочтения вы сможете:**
- Построить batch с нуля.
- Группировать операции по размеру и времени.
- Правильно делать flush при завершении.
- Комбинировать batch с другими паттернами.
- Понимать, где batch уместен, а где — нет.

---

## Содержание

- [21.0 Пролог: слишком много мелких операций](#210-пролог-слишком-много-мелких-операций)
- [21.1 Что такое batch](#211-что-такое-batch)
- [21.2 Batch по размеру](#212-batch-по-размеру)
- [21.3 Batch по времени](#213-batch-по-времени)
- [21.4 Гибридный batch (размер + время)](#214-гибридный-batch-размер--время)
- [21.5 Batch с context](#215-batch-с-context)
- [21.6 Flush при shutdown](#216-flush-при-shutdown)
- [21.7 Backpressure в batch](#217-backpressure-в-batch)
- [21.8 В связке с другими паттернами](#218-в-связке-с-другими-паттернами)
- [21.9 Практика Go: batch с метриками](#219-практика-go-batch-с-метриками)
- [21.10 Выводы и типичные ошибки](#2110-выводы-и-типичные-ошибки)
- [21.11 Для быстрого повторения](#2111-для-быстрого-повторения)
- [21.12 Вопросы для самопроверки](#2112-вопросы-для-самопроверки)
- [21.13 Ответы](#2113-ответы)
- [21.14 Куда идти дальше?](#2114-куда-идти-дальше)
- [21.15 Чек-лист](#2115-чек-лист)

---

## 21.0 Пролог: слишком много мелких операций

У нас есть сервис, который обрабатывает события и пишет их в ClickHouse. События приходят потоком — **10 000 в секунду**. Пишем наивно:

```go
func (s *Service) HandleEvent(ctx context.Context, event Event) error {
    return s.clickhouse.Insert(ctx, event)  // ← INSERT на каждое событие
}
```

Работает. Но ClickHouse **не любит** мелкие INSERT. Каждый INSERT — это отдельная транзакция, отдельная запись в WAL, отдельный fsync. При 10 000 событий в секунду — **10 000 INSERT'ов**. ClickHouse захлёбывается.

Хочется: **накапливать** события и писать **батчами** — 1000 событий за раз. Тогда 10 000/сек = 10 INSERT'ов в секунду. **В 1000 раз меньше нагрузки**.

Но есть тонкость: если накапливать 1000 событий, а события идут медленно (10 в секунду), то батч соберётся только через **100 секунд**. Данные задерживаются.

Хочется: писать батчами **по 1000 событий** ИЛИ **раз в 1 секунду**, что раньше наступит.

Это и есть **batch** — группировка операций.

> **Мост к следующим главам:** batch — важный паттерн для работы с БД, Kafka, ClickHouse, внешними API. Он часто используется вместе с worker pool (Глава 13), pipeline (Глава 11) и graceful shutdown (Глава 11). Понимание batch даёт понимание, **как оптимизировать I/O**.

---

## 21.1 Что такое batch

**Batch** — паттерн, при котором **несколько мелких операций** объединяются в **одну крупную**.

### Идея

Многие системы **оптимизированы под крупные операции**:

- **ClickHouse** — batch INSERT эффективнее в 1000 раз.
- **Kafka** — batch produce эффективнее в 10–100 раз.
- **HTTP API** — bulk endpoint эффективнее, чем N запросов.
- **БД** — batch INSERT через `COPY` или `INSERT ... VALUES (...), (...)`.

**Batch** накапливает операции и выполняет их **группой**.

### Схема

```
Без batch:
  event1 ──► INSERT
  event2 ──► INSERT
  event3 ──► INSERT
  ...
  10 000 INSERT'ов в секунду

С batch:
  event1 ──┐
  event2 ──┤
  event3 ──┼──► [batch 1000] ──► INSERT
  ...      │
  event1000┘
  10 INSERT'ов в секунду
```

### Когда использовать batch

**1. Много мелких операций.**

- События в секунду.
- Логи.
- Метрики.
- Трейсы.

**2. Системы с дорогим I/O.**

- ClickHouse.
- Kafka.
- S3.
- Внешние API с bulk endpoint.

**3. Дорогая транзакция.**

- БД с fsync.
- Распределённые транзакции.

### Когда НЕ использовать batch

**1. Мало операций.**

Если операций 10 в час — batch не нужен.

**2. Низкая latency критична.**

Batch **добавляет задержку** — ждём накопления.

**3. Операции независимы по времени.**

Если каждая операция должна быть выполнена **немедленно** — batch не подходит.

### Два типа batch

**Batch по размеру:**

- Накапливаем N элементов.
- При достижении N — flush.
- Простой, но может ждать долго при низкой нагрузке.

**Batch по времени:**

- Накапливаем T времени.
- Через T — flush.
- Простой, но может быть маленьким батч.

**Гибридный (размер + время):**

- Flush при N элементов **или** через T времени.
- Лучший вариант для production.

### 💡 Практика: как думать о batch

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Batch — для многих мелких операций.**
2. **Batch по размеру + по времени.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Размер батча 100–10 000.**
4. **Время 100 мс – 5 сек.**

**❌ НЕ ДЕЛАЙ:**

5. **Не используй batch, если latency критична.**
6. **Не делай батч слишком большим.**

---

## 21.2 Batch по размеру

Начнём с простейшего — batch по размеру.

### Идея

- Накапливаем элементы в буфере.
- При достижении N — flush.
- Простой и предсказуемый.

### Реализация

```go
type Batcher[T any] struct {
    size   int
    buffer []T
    flush  func([]T) error
    mu     sync.Mutex
}

func NewBatcher[T any](size int, flush func([]T) error) *Batcher[T] {
    return &Batcher[T]{
        size:   size,
        buffer: make([]T, 0, size),
        flush:  flush,
    }
}

func (b *Batcher[T]) Add(item T) error {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    b.buffer = append(b.buffer, item)
    if len(b.buffer) >= b.size {
        return b.flushLocked()
    }
    return nil
}

func (b *Batcher[T]) flushLocked() error {
    if len(b.buffer) == 0 {
        return nil
    }
    batch := b.buffer
    b.buffer = make([]T, 0, b.size)
    return b.flush(batch)
}

func (b *Batcher[T]) Flush() error {
    b.mu.Lock()
    defer b.mu.Unlock()
    return b.flushLocked()
}
```

**Что происходит:**

- `Add` добавляет элемент.
- При `len >= size` — flush.
- `Flush` — ручной flush.

### Потребитель

```go
func main() {
    batcher := NewBatcher(100, func(items []int) error {
        fmt.Printf("Flushing %d items: %v\n", len(items), items[:5])
        return nil
    })
    
    for i := 1; i <= 250; i++ {
        batcher.Add(i)
    }
    
    // Остаток
    batcher.Flush()
}
```

**Пример вывода:**

```
Flushing 100 items: [1 2 3 4 5]
Flushing 100 items: [101 102 103 104 105]
Flushing 50 items: [201 202 203 204 205]
```

**Что видно:** 250 элементов → 2 полных батча + 1 остаток.

### Проблема: низкая нагрузка

**Что если элементов 10 в час?**

- Batch на 100 не соберётся **никогда**.
- Данные **задерживаются** на часы.

**Решение:** batch по времени.

### Схема

```
Add(1) → buffer=[1]
Add(2) → buffer=[1,2]
...
Add(100) → buffer=[1..100] → flush!
```

### 💡 Практика: как писать batch по размеру

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Буфер с capacity `size`.**
2. **Flush при достижении `size`.**
3. **Ручной `Flush` для остатка.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`Mutex` для потокобезопасности.**
5. **Размер 100–10 000.**

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай про остаток.**
7. **Не используй для низкой нагрузки.**

---

## 21.3 Batch по времени

Решим проблему низкой нагрузки — flush по времени.

### Идея

- Накапливаем элементы.
- Через T времени — flush.
- Даже если батч маленький.

### Реализация

```go
type TimeBatcher[T any] struct {
    interval time.Duration
    buffer   []T
    flush    func([]T) error
    mu       sync.Mutex
    stopCh   chan struct{}
    wg       sync.WaitGroup
}

func NewTimeBatcher[T any](interval time.Duration, flush func([]T) error) *TimeBatcher[T] {
    b := &TimeBatcher[T]{
        interval: interval,
        buffer:   make([]T, 0),
        flush:    flush,
        stopCh:   make(chan struct{}),
    }
    
    b.wg.Add(1)
    go b.run()
    return b
}

func (b *TimeBatcher[T]) run() {
    defer b.wg.Done()
    ticker := time.NewTicker(b.interval)
    defer ticker.Stop()
    
    for {
        select {
        case <-b.stopCh:
            return
        case <-ticker.C:
            b.Flush()
        }
    }
}

func (b *TimeBatcher[T]) Add(item T) {
    b.mu.Lock()
    b.buffer = append(b.buffer, item)
    b.mu.Unlock()
}

func (b *TimeBatcher[T]) Flush() error {
    b.mu.Lock()
    if len(b.buffer) == 0 {
        b.mu.Unlock()
        return nil
    }
    batch := b.buffer
    b.buffer = make([]T, 0)
    b.mu.Unlock()
    return b.flush(batch)
}

func (b *TimeBatcher[T]) Close() error {
    close(b.stopCh)
    b.wg.Wait()
    return b.Flush()
}
```

**Что происходит:**

- `run` — тикер каждые T времени.
- При тике — flush.
- `Add` не блокируется.

### Потребитель

```go
func main() {
    batcher := NewTimeBatcher(1*time.Second, func(items []int) error {
        fmt.Printf("[%v] Flushing %d items\n", time.Now().Format("15:04:05.000"), len(items))
        return nil
    })
    defer batcher.Close()
    
    for i := 1; i <= 25; i++ {
        batcher.Add(i)
        time.Sleep(100 * time.Millisecond)
    }
    
    time.Sleep(1 * time.Second)
}
```

**Пример вывода:**

```
[15:04:05.000] Flushing 10 items
[15:04:06.000] Flushing 10 items
[15:04:07.000] Flushing 5 items
```

**Что видно:** каждую секунду — flush того, что накопилось.

### Проблема: маленькие батчи

**Что если нагрузка высокая?**

- За 1 секунду накапливается 10 000 элементов.
- Батч на 10 000 — большой.
- Память тратится.

**Решение:** гибридный batch.

### Схема

```
t=0:    buffer=[1, 2, 3]
t=1s:   flush → [1, 2, 3]
t=1.5:  buffer=[4, 5]
t=2s:   flush → [4, 5]
```

### 💡 Практика: как писать batch по времени

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Тикер для flush.**
2. **`stopCh` для остановки.**
3. **`Close()` для финального flush.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Интервал 100 мс – 5 сек.**
5. **`Mutex` для буфера.**

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `Close()`.**
7. **Не используй для высоких нагрузок без размера.**

---

## 21.4 Гибридный batch (размер + время)

**Гибридный batch** — flush при N элементов **или** через T времени.

### Идея

Лучшее из двух:

- **По размеру** — не даёт батчу расти бесконечно.
- **По времени** — не даёт данным задерживаться.

### Реализация

```go
type Batcher[T any] struct {
    size     int
    interval time.Duration
    buffer   []T
    flush    func([]T) error
    mu       sync.Mutex
    stopCh   chan struct{}
    wg       sync.WaitGroup
}

func NewBatcher[T any](size int, interval time.Duration, flush func([]T) error) *Batcher[T] {
    b := &Batcher[T]{
        size:     size,
        interval: interval,
        buffer:   make([]T, 0, size),
        flush:    flush,
        stopCh:   make(chan struct{}),
    }
    
    b.wg.Add(1)
    go b.run()
    return b
}

func (b *Batcher[T]) run() {
    defer b.wg.Done()
    ticker := time.NewTicker(b.interval)
    defer ticker.Stop()
    
    for {
        select {
        case <-b.stopCh:
            return
        case <-ticker.C:
            b.Flush()
        }
    }
}

func (b *Batcher[T]) Add(item T) error {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    b.buffer = append(b.buffer, item)
    if len(b.buffer) >= b.size {
        return b.flushLocked()
    }
    return nil
}

func (b *Batcher[T]) flushLocked() error {
    if len(b.buffer) == 0 {
        return nil
    }
    batch := b.buffer
    b.buffer = make([]T, 0, b.size)
    return b.flush(batch)
}

func (b *Batcher[T]) Flush() error {
    b.mu.Lock()
    defer b.mu.Unlock()
    return b.flushLocked()
}

func (b *Batcher[T]) Close() error {
    close(b.stopCh)
    b.wg.Wait()
    return b.Flush()
}
```

**Что происходит:**

- `Add` — при достижении `size` → flush.
- `run` — тикер каждые `interval` → flush.
- `Close` — остановка тикера + финальный flush.

### Потребитель

```go
func main() {
    batcher := NewBatcher(10, 1*time.Second, func(items []int) error {
        fmt.Printf("[%v] Flushing %d items\n", time.Now().Format("15:04:05.000"), len(items))
        return nil
    })
    defer batcher.Close()
    
    // Быстрая нагрузка: 25 элементов сразу
    fmt.Println("=== Fast load ===")
    for i := 1; i <= 25; i++ {
        batcher.Add(i)
    }
    time.Sleep(2 * time.Second)
    
    // Медленная нагрузка: 3 элемента за 5 секунд
    fmt.Println("=== Slow load ===")
    for i := 1; i <= 3; i++ {
        batcher.Add(i)
        time.Sleep(1 * time.Second)
    }
}
```

**Пример вывода:**

```
=== Fast load ===
[15:04:05.000] Flushing 10 items
[15:04:05.000] Flushing 10 items
[15:04:05.000] Flushing 5 items
=== Slow load ===
[15:04:06.000] Flushing 1 items
[15:04:07.000] Flushing 1 items
[15:04:08.000] Flushing 1 items
```

**Что видно:**

- **Быстрая нагрузка:** flush по размеру (10, 10, 5).
- **Медленная нагрузка:** flush по времени (1, 1, 1).

**Гибридный batch работает.**

### Схема

```
При быстрой нагрузке:
  Add x10 → flush
  Add x10 → flush
  Add x5  → flush

При медленной нагрузке:
  Add x1  → тик 1s → flush
  Add x1  → тик 1s → flush
```

### 💡 Практика: как писать гибридный batch

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Size + interval.**
2. **`Add` — блокирующий при flush.**
3. **`Close` — остановка + финальный flush.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики:** размер батчей, частота.
5. **`context` для отмены.**

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `Close`.**
7. **Не делай батч слишком большим.**

---

## 21.5 Batch с context

`context` — критичен для batch. Без него batch может **зависнуть** или **потерять данные**.

### Проблема

```go
batcher := NewBatcher(100, 1*time.Second, flushFn)
// При shutdown batcher.Close() не вызывается
```

**Что происходит:** при shutdown буфер **теряется**.

### Решение: context

```go
type Batcher[T any] struct {
    size     int
    interval time.Duration
    buffer   []T
    flush    func(context.Context, []T) error
    mu       sync.Mutex
    stopCh   chan struct{}
    wg       sync.WaitGroup
}

func (b *Batcher[T]) Add(ctx context.Context, item T) error {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    b.buffer = append(b.buffer, item)
    if len(b.buffer) >= b.size {
        return b.flushLocked(ctx)
    }
    return nil
}

func (b *Batcher[T]) flushLocked(ctx context.Context) error {
    if len(b.buffer) == 0 {
        return nil
    }
    batch := b.buffer
    b.buffer = make([]T, 0, b.size)
    return b.flush(ctx, batch)
}

func (b *Batcher[T]) Flush(ctx context.Context) error {
    b.mu.Lock()
    defer b.mu.Unlock()
    return b.flushLocked(ctx)
}

func (b *Batcher[T]) Close(ctx context.Context) error {
    close(b.stopCh)
    b.wg.Wait()
    
    // Финальный flush с отдельным ctx (без отмены)
    finalCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    return b.Flush(finalCtx)
}
```

**Что происходит:**

- `flush` принимает `ctx`.
- `Close` использует **отдельный** `ctx` для финального flush.

### Потребитель

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM)
    defer stop()
    
    batcher := NewBatcher(100, 1*time.Second, func(ctx context.Context, items []int) error {
        return saveToDB(ctx, items)
    })
    
    // Обработка событий
    go func() {
        for {
            select {
            case <-ctx.Done():
                return
            default:
            }
            
            event := receiveEvent()
            if err := batcher.Add(ctx, event); err != nil {
                log.Printf("add error: %v", err)
            }
        }
    }()
    
    <-ctx.Done()
    fmt.Println("Shutting down...")
    
    // Финальный flush
    if err := batcher.Close(context.Background()); err != nil {
        log.Printf("close error: %v", err)
    }
}
```

**Что происходит:** при SIGTERM `batcher.Close` делает **финальный flush** всех накопленных данных.

### Схема

```
Add(ctx, item):
  ├── append
  └── if len >= size: flush(ctx, batch)

Close(ctx):
  ├── close(stopCh)
  ├── wg.Wait()
  └── flush(finalCtx, buffer)  ← финальный flush
```

### 💡 Практика: как добавить context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx` — первый аргумент `Add` и `flush`.**
2. **`Close` с отдельным `ctx`.**
3. **`defer batcher.Close()`.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Логирование финального flush.**
5. **Метрики размера батчей.**

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай финальный flush.**
7. **Не используй отменённый `ctx` в `Close`.**

---

## 21.6 Flush при shutdown

**Flush при shutdown** — критичен для batch. Без него данные **теряются**.

### Проблема

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM)
    defer stop()
    
    batcher := NewBatcher(100, 1*time.Second, flushFn)
    // ...
    
    <-ctx.Done()
    // Программа выходит — буфер теряется!
}
```

**Что происходит:** 50 элементов в буфере — **потеряны**.

### Решение: graceful shutdown

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM)
    defer stop()
    
    batcher := NewBatcher(100, 1*time.Second, flushFn)
    
    // ... работа ...
    
    <-ctx.Done()
    fmt.Println("Shutting down...")
    
    // Финальный flush
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    if err := batcher.Close(shutdownCtx); err != nil {
        log.Printf("close error: %v", err)
    }
    
    fmt.Println("Shutdown complete")
}
```

**Что происходит:** при shutdown — финальный flush с таймаутом 30 секунд.

### Порядок операций

**Правильный порядок:**

1. **Прекратить** приём новых событий.
2. **Дождаться** завершения обработки.
3. **Flush** batch.
4. **Закрыть** ресурсы.

**Неправильный порядок:**

1. **Закрыть** БД.
2. **Flush** batch → ошибка (БД закрыта).

### Схема

```
Shutdown:
  1. Прекратить приём событий
  2. Дождаться обработки
  3. Flush batch
  4. Закрыть ресурсы
  
Если закрыть ресурсы раньше — flush не сработает.
```

### 💡 Практика: как делать flush при shutdown

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`defer batcher.Close()`** или явный вызов.
2. **Отдельный `shutdownCtx` с таймаутом.**
3. **Правильный порядок** — flush до закрытия ресурсов.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Логирование** — сколько элементов во flush.
5. **Метрики** — потерянные данные.

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай про flush.**
7. **Не закрывай ресурсы до flush.**

---

## 21.7 Backpressure в batch

**Backpressure** — что делать, если batch не успевает.

### Проблема

**Что если flush медленный?**

- События приходят быстрее, чем мы пишем батчами.
- Буфер растёт.
- Память растёт.

**Что если flush быстрый, но producer очень быстрый?**

- Буфер заполняется мгновенно.
- `Add` блокируется на `flush`.

### Решение: буферизованный канал

```go
type Batcher[T any] struct {
    size     int
    interval time.Duration
    in       chan T
    flush    func([]T) error
    stopCh   chan struct{}
    wg       sync.WaitGroup
}

func NewBatcher[T any](size int, interval time.Duration, bufferSize int, flush func([]T) error) *Batcher[T] {
    b := &Batcher[T]{
        size:     size,
        interval: interval,
        in:       make(chan T, bufferSize),
        flush:    flush,
        stopCh:   make(chan struct{}),
    }
    
    b.wg.Add(1)
    go b.run()
    return b
}

func (b *Batcher[T]) run() {
    defer b.wg.Done()
    
    ticker := time.NewTicker(b.interval)
    defer ticker.Stop()
    
    buffer := make([]T, 0, b.size)
    
    flush := func() {
        if len(buffer) == 0 {
            return
        }
        batch := make([]T, len(buffer))
        copy(batch, buffer)
        buffer = buffer[:0]
        
        if err := b.flush(batch); err != nil {
            log.Printf("flush error: %v", err)
        }
    }
    
    for {
        select {
        case <-b.stopCh:
            flush()
            return
        case item := <-b.in:
            buffer = append(buffer, item)
            if len(buffer) >= b.size {
                flush()
            }
        case <-ticker.C:
            flush()
        }
    }
}

func (b *Batcher[T]) Add(ctx context.Context, item T) error {
    select {
    case b.in <- item:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func (b *Batcher[T]) Close(ctx context.Context) error {
    close(b.stopCh)
    b.wg.Wait()
    return nil
}
```

**Что происходит:**

- `in` — буферизованный канал.
- `Add` пишет в канал.
- `run` читает из канала, накапливает, flush.
- **Backpressure:** если канал полон — `Add` блокируется.

### Схема

```
Add(ctx, item) ──► in (буфер 1000) ──► run ──► buffer ──► flush
                                      │
                                      │ при size или interval
                                      ▼
                                    flush
```

### Размер буфера

**Маленький буфер (10–100):**

- Backpressure быстрый.
- `Add` блокируется часто.

**Большой буфер (1000–10 000):**

- Backpressure отложенный.
- Больше памяти.

**Рекомендация:** буфер = 10× size батча.

### 💡 Практика: как обрабатывать backpressure

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Буферизованный канал.**
2. **`select` с `ctx.Done()`.**
3. **Мониторинг длины буфера.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики** — сколько Add'ов отвергнуто.
5. **Буфер 10× size.**

**❌ НЕ ДЕЛАЙ:**

6. **Не делай буфер бесконечным.**
7. **Не игнорируй растущий буфер.**

---

## 21.8 В связке с другими паттернами

Batch редко используется **в одиночку**. Разберём связки.

### Batch + worker pool

**Worker pool обрабатывает события, batch пишет в БД:**

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM)
    defer stop()
    
    batcher := NewBatcher(1000, 1*time.Second, 10000, flushToDB)
    defer batcher.Close(context.Background())
    
    eventsCh := make(chan Event, 1000)
    
    // Worker pool
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for event := range eventsCh {
                processed := process(event)
                if err := batcher.Add(ctx, processed); err != nil {
                    log.Printf("add error: %v", err)
                }
            }
        }()
    }
    
    // Producer
    for _, event := range events {
        eventsCh <- event
    }
    close(eventsCh)
    wg.Wait()
}
```

### Batch + pipeline

**Batch как стадия pipeline:**

```go
func batchStage(ctx context.Context, in <-chan Event, size int, interval time.Duration) <-chan []Event {
    out := make(chan []Event)
    
    go func() {
        defer close(out)
        
        buffer := make([]Event, 0, size)
        ticker := time.NewTicker(interval)
        defer ticker.Stop()
        
        flush := func() {
            if len(buffer) == 0 {
                return
            }
            batch := make([]Event, len(buffer))
            copy(batch, buffer)
            buffer = buffer[:0]
            
            select {
            case out <- batch:
            case <-ctx.Done():
            }
        }
        
        for {
            select {
            case <-ctx.Done():
                return
            case event, ok := <-in:
                if !ok {
                    flush()
                    return
                }
                buffer = append(buffer, event)
                if len(buffer) >= size {
                    flush()
                }
            case <-ticker.C:
                flush()
            }
        }
    }()
    
    return out
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := eventSource(ctx)
    batched := batchStage(ctx, source, 100, 1*time.Second)
    
    for batch := range batched {
        saveToDB(batch)
    }
}
```

### Batch + rate limiter

**Batch + rate limiter для внешних API:**

```go
func (s *Service) Flush(ctx context.Context, items []Item) error {
    if err := s.limiter.Wait(ctx); err != nil {
        return err
    }
    return s.api.BulkInsert(ctx, items)
}
```

**Что происходит:** rate limiter ограничивает частоту батчей.

### Batch + circuit breaker

**Batch + circuit breaker для внешних вызовов:**

```go
func (s *Service) Flush(ctx context.Context, items []Item) error {
    _, err := s.breaker.Execute(func() (interface{}, error) {
        return nil, s.api.BulkInsert(ctx, items)
    })
    return err
}
```

### Схема

```
Events ──► worker pool ──► batcher ──► DB
                              │
                              │ при size или interval
                              ▼
                           flush
```

### 💡 Практика: как комбинировать batch

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Batch + worker pool** — обработка + запись.
2. **Batch как стадия pipeline.**
3. **Batch + rate limiter** — ограничение скорости.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Batch + circuit breaker** — защита от сбоев.

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай про flush при shutdown.**

---

## 21.9 Практика Go: batch с метриками

Разберём **batch с метриками**.

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

type Metrics struct {
    Added     atomic.Int64
    Flushed   atomic.Int64
    Batches   atomic.Int64
    Errors    atomic.Int64
    FlushTime atomic.Int64
}

type Batcher[T any] struct {
    size     int
    interval time.Duration
    buffer   []T
    flush    func([]T) error
    mu       sync.Mutex
    stopCh   chan struct{}
    wg       sync.WaitGroup
    metrics  *Metrics
}

func NewBatcher[T any](size int, interval time.Duration, flush func([]T) error) *Batcher[T] {
    b := &Batcher[T]{
        size:     size,
        interval: interval,
        buffer:   make([]T, 0, size),
        flush:    flush,
        stopCh:   make(chan struct{}),
        metrics:  &Metrics{},
    }
    
    b.wg.Add(1)
    go b.run()
    return b
}

func (b *Batcher[T]) run() {
    defer b.wg.Done()
    ticker := time.NewTicker(b.interval)
    defer ticker.Stop()
    
    for {
        select {
        case <-b.stopCh:
            return
        case <-ticker.C:
            b.Flush()
        }
    }
}

func (b *Batcher[T]) Add(item T) error {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    b.buffer = append(b.buffer, item)
    b.metrics.Added.Add(1)
    
    if len(b.buffer) >= b.size {
        return b.flushLocked()
    }
    return nil
}

func (b *Batcher[T]) flushLocked() error {
    if len(b.buffer) == 0 {
        return nil
    }
    
    batch := b.buffer
    b.buffer = make([]T, 0, b.size)
    
    start := time.Now()
    err := b.flush(batch)
    elapsed := time.Since(start)
    
    b.metrics.Batches.Add(1)
    b.metrics.Flushed.Add(int64(len(batch)))
    b.metrics.FlushTime.Add(int64(elapsed))
    
    if err != nil {
        b.metrics.Errors.Add(1)
        return err
    }
    return nil
}

func (b *Batcher[T]) Flush() error {
    b.mu.Lock()
    defer b.mu.Unlock()
    return b.flushLocked()
}

func (b *Batcher[T]) Close() error {
    close(b.stopCh)
    b.wg.Wait()
    return b.Flush()
}

func (b *Batcher[T]) Metrics() *Metrics {
    return b.metrics
}

func main() {
    batcher := NewBatcher(100, 500*time.Millisecond, func(items []int) error {
        time.Sleep(10 * time.Millisecond)
        return nil
    })
    
    // Быстрая нагрузка
    for i := 0; i < 10000; i++ {
        batcher.Add(i)
    }
    
    // Медленная нагрузка
    for i := 0; i < 5; i++ {
        batcher.Add(i)
        time.Sleep(200 * time.Millisecond)
    }
    
    batcher.Close()
    
    m := batcher.Metrics()
    fmt.Println("=== Metrics ===")
    fmt.Printf("Added:     %d\n", m.Added.Load())
    fmt.Printf("Flushed:   %d\n", m.Flushed.Load())
    fmt.Printf("Batches:   %d\n", m.Batches.Load())
    fmt.Printf("Errors:    %d\n", m.Errors.Load())
    fmt.Printf("Avg size:  %.1f\n", float64(m.Flushed.Load())/float64(m.Batches.Load()))
    fmt.Printf("Avg flush: %v\n", time.Duration(m.FlushTime.Load()/m.Batches.Load()))
}
```

**Пример вывода:**

```
=== Metrics ===
Added:     10005
Flushed:   10005
Batches:   103
Errors:    0
Avg size:  97.1
Avg flush: 10.2ms
```

**Что демонстрирует:**

- **Добавлено** 10 005 элементов.
- **Flushed** столько же — ничего не потеряно.
- **103 батча** — по 97 в среднем.
- **Среднее время flush** — 10.2 мс.

### Пример: batch с backpressure

```go
func main() {
    batcher := NewBatcherWithBuffer(100, 500*time.Millisecond, 1000, func(items []int) error {
        time.Sleep(50 * time.Millisecond)
        return nil
    })
    defer batcher.Close()
    
    ctx := context.Background()
    
    var rejected atomic.Int64
    var added atomic.Int64
    
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        i := i
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < 1000; j++ {
                select {
                case <-ctx.Done():
                    return
                default:
                }
                if err := batcher.Add(ctx, i*1000+j); err != nil {
                    rejected.Add(1)
                    return
                }
                added.Add(1)
            }
        }()
    }
    wg.Wait()
    
    fmt.Printf("Added:    %d\n", added.Load())
    fmt.Printf("Rejected: %d\n", rejected.Load())
}
```

**Что демонстрирует:** при переполнении буфера `Add` блокируется или возвращает ошибку.

### 💡 Практика: как измерять batch

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики:** added, flushed, batches, errors.
2. **Avg size** — средний размер батча.
3. **Avg flush time** — среднее время flush.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Экспорт в Prometheus.**
5. **Алерт при большом errors.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `Mutex` для метрик.**
7. **Не игнорируй errors.**

---

## 21.10 Выводы и типичные ошибки

**Что мы узнали?**

Batch — паттерн группировки мелких операций в крупные. **Batch по размеру** — flush при N. **Batch по времени** — flush через T. **Гибридный** — при N **или** T (лучший для production). **`context`** — для отмены. **Flush при shutdown** — обязательно, иначе данные теряются. **Backpressure** — через буферизованный канал. Batch комбинируется с worker pool, pipeline, rate limiter, circuit breaker.

**Типичные ошибки:**

- ❌ **Забыть `Close()`.** Данные в буфере теряются.
- ❌ **Не использовать `context`.** Зависание.
- ❌ **Flush при shutdown после закрытия ресурсов.** Ошибка.
- ❌ **Не забывай про остаток.** Последний батч может быть маленьким.
- ❌ **Огромный буфер.** Память растёт.
- ❌ **Маленький буфер.** Много reject'ов.
- ❌ **Batch для latency-критичных операций.**
- ❌ **Один батч без размера.** Ждёт долго при низкой нагрузке.
- ❌ **Один батч без времени.** Большие батчи при высокой нагрузке.
- ❌ **Не мониторить размер батчей.**

---

## 21.11 Для быстрого повторения

- **Batch** — группировка мелких операций в крупные.
- **Batch по размеру** — flush при N.
- **Batch по времени** — flush через T.
- **Гибридный** — при N **или** T.
- **Размер 100–10 000.**
- **Интервал 100 мс – 5 сек.**
- **`context`** — для отмены.
- **`Close()`** — обязательно, иначе данные теряются.
- **Flush при shutdown** — до закрытия ресурсов.
- **Backpressure** — буферизованный канал.
- **Буфер = 10× size.**
- **Batch + worker pool** — обработка + запись.
- **Batch как стадия pipeline.**
- **Batch + rate limiter** — ограничение скорости.
- **Batch + circuit breaker** — защита от сбоев.
- **Метрики:** added, flushed, batches, errors.

---

## 21.12 Вопросы для самопроверки

1. Что такое batch? Какую задачу решает?
2. Чем batch по размеру отличается от batch по времени?
3. Почему гибридный batch лучше?
4. Зачем `context` в batch?
5. Почему `Close()` критичен?
6. Что такое backpressure в batch?
7. Как комбинировать batch с worker pool?
8. Что будет, если не делать flush при shutdown?

---

## 21.13 Ответы

### Ответ 1

**Batch** — паттерн группировки мелких операций в крупные. Решает задачу: **оптимизация I/O** — вместо N мелких операций делаем 1 крупную.

**Пример:** 10 000 событий в ClickHouse — 10 000 INSERT'ов вместо 10 батчей по 1000.

### Ответ 2

**Batch по размеру** — flush при N элементов. Простой, но ждёт при низкой нагрузке.

**Batch по времени** — flush через T времени. Простой, но маленькие батчи при низкой нагрузке.

**Гибридный** — flush при N **или** T. Лучший для production.

### Ответ 3

**Гибридный batch** решает обе проблемы:

- **По размеру** — не даёт батчу расти бесконечно.
- **По времени** — не даёт данным задерживаться.

При быстрой нагрузке — flush по размеру. При медленной — по времени.

### Ответ 4

**`context`** позволяет отменить flush при отмене. Без него flush может **зависнуть**.

**Решение:** `flush(ctx, batch)` + `Close` с отдельным `ctx`.

### Ответ 5

**`Close()` критичен**, потому что при shutdown буфер **теряется**. Без `Close` — 50 элементов в буфере **потеряны**.

**Решение:** `Close()` делает **финальный flush** всех накопленных данных.

### Ответ 6

**Backpressure в batch** — что делать, если flush медленный или producer быстрый.

**Решение:** буферизованный канал между `Add` и `run`. Если канал полон — `Add` блокируется.

### Ответ 7

**Batch + worker pool:**

```go
// Worker pool обрабатывает события
for event := range eventsCh {
    processed := process(event)
    batcher.Add(ctx, processed)  // ← batch пишет в БД
}
```

Worker pool — обработка. Batch — запись.

### Ответ 8

**Без flush при shutdown:** данные в буфере **теряются**. 50 элементов — потеряны.

**Решение:** `defer batcher.Close()` или явный вызов с `shutdownCtx`.

---

## 21.14 Куда идти дальше?

Мы разобрали batch — группировку операций. Теперь мы умеем оптимизировать I/O.

Но иногда нужно **изолировать ресурсы** между разными частями системы. Медленный сервис не должен занимать воркеров для быстрых.

- **Как изолировать ресурсы?** → **Глава 22: Bulkhead.**
- **Как выбрать case в select?** → **Глава 23: Динамический select и приоритеты.**
- **Как выбрать лидера?** → **Глава 24: Leader election.**

---

## 21.15 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Batch** | Группировка операций | N мелких → 1 крупная |
| **Batch по размеру** | Flush при N | Простой, но ждёт |
| **Batch по времени** | Flush через T | Простой, но маленькие |
| **Гибридный** | N или T | Лучший для production |
| **Размер** | 100–10 000 | — |
| **Интервал** | 100 мс – 5 сек | — |
| **`context`** | Отмена | `flush(ctx, batch)` |
| **`Close()`** | Финальный flush | Обязательно |
| **Flush при shutdown** | До закрытия ресурсов | Иначе потеря данных |
| **Backpressure** | Буферизованный канал | Буфер = 10× size |
| **Batch + worker pool** | Обработка + запись | — |
| **Batch как стадия pipeline** | — | — |
| **Batch + rate limiter** | Ограничение скорости | — |
| **Batch + circuit breaker** | Защита от сбоев | — |
| **Метрики** | added, flushed, batches, errors | `atomic.Int64` |

📦 **Ключевая идея:** Batch — паттерн группировки мелких операций в крупные. **Batch по размеру** — flush при N. **Batch по времени** — flush через T. **Гибридный** — при N **или** T. **`context`** для отмены. **`Close()`** обязателен — иначе данные теряются. **Flush при shutdown** — до закрытия ресурсов. **Backpressure** — буферизованный канал (буфер = 10× size). Комбинируется с worker pool, pipeline, rate limiter, circuit breaker. Метрики: added, flushed, batches, errors.