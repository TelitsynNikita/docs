# 🚦 Глава 27: Backpressure — управление потоком

**Что вы узнаете:**
- Что такое backpressure и какую задачу он решает.
- Почему backpressure — это **свойство**, а не паттерн.
- Как backpressure работает в Go **автоматически** через каналы.
- Как его **усилить** или **ослабить** через размер буфера.
- Что такое **drop** и **reject** как альтернативы backpressure.
- Как выбрать между backpressure, drop и reject.
- Как комбинировать backpressure с worker pool, pipeline, rate limiter.
- Как мониторить backpressure в production.

**После прочтения вы сможете:**
- Объяснить, как работает backpressure в Go.
- Настроить размер буфера под свою нагрузку.
- Выбрать между backpressure, drop и reject.
- Мониторить backpressure через метрики.
- Комбинировать backpressure с другими паттернами.
- Понимать, где backpressure уместен, а где — нет.

---

## Содержание

- [27.0 Пролог: consumer не успевает](#270-пролог-consumer-не-успевает)
- [27.1 Что такое backpressure](#271-что-такое-backpressure)
- [27.2 Backpressure в Go: автоматически](#272-backpressure-в-go-автоматически)
- [27.3 Размер буфера: как настроить](#273-размер-буфера-как-настроить)
- [27.4 Три стратегии: block, drop, reject](#274-три-стратегии-block-drop-reject)
- [27.5 Backpressure в pipeline](#275-backpressure-в-pipeline)
- [27.6 Backpressure в worker pool](#276-backpressure-в-worker-pool)
- [27.7 Мониторинг backpressure](#277-мониторинг-backpressure)
- [27.8 В связке с другими паттернами](#278-в-связке-с-другими-паттернами)
- [27.9 Практика Go: backpressure с метриками](#279-практика-go-backpressure-с-метриками)
- [27.10 Выводы и типичные ошибки](#2710-выводы-и-типичные-ошибки)
- [27.11 Для быстрого повторения](#2711-для-быстрого-повторения)
- [27.12 Вопросы для самопроверки](#2712-вопросы-для-самопроверки)
- [27.13 Ответы](#2713-ответы)
- [27.14 Куда идти дальше?](#2714-куда-идти-дальше)
- [27.15 Чек-лист](#2715-чек-лист)

---

## 27.0 Пролог: consumer не успевает

У нас есть сервис, который читает события из Kafka и пишет их в БД. Consumer работает быстро — **10 000 событий в секунду**. Но БД **не успевает** — максимум **1000 событий в секунду**. Что происходит?

Пишем наивно через worker pool:

```go
func main() {
    eventsCh := make(chan Event, 10000)  // ← большой буфер
    
    // Consumer из Kafka
    go func() {
        for {
            event := kafka.Read()
            eventsCh <- event  // ← не блокируется, буфер большой
        }
    }()
    
    // Worker pool для записи в БД
    for i := 0; i < 10; i++ {
        go func() {
            for event := range eventsCh {
                db.Insert(event)  // ← 1000/сек
            }
        }()
    }
}
```

Работает. Но замечаем проблему: **память растёт**. За минуту буфер на 10 000 заполняется. Consumer **продолжает писать** — потому что канал буферизован на 10 000. Он не блокируется.

**Что происходит:**

- Kafka производит 10 000/сек.
- БД обрабатывает 1000/сек.
- Буфер заполняется за **1 секунду**.
- Дальше — consumer **блокируется**? Нет, потому что буфер большой.
- Consumer **продолжает писать** — блокируется только когда буфер полон.
- **Буфер полон** → consumer блокируется → Kafka **не читается**.
- Но Kafka **накапливает** события.
- Если consumer долго не читает — Kafka **выкидывает** его из группы.

Хочется: **не накапливать** события. Consumer должен **замедлиться** до скорости БД. Тогда Kafka будет **буферизовать** события — и это её работа, а не наша.

Это и есть **backpressure** — управление потоком данных.

> **Мост к следующим главам:** backpressure — не паттерн, а **свойство** каналов в Go. Он работает **автоматически**. Понимание backpressure даёт понимание, **как не утонуть в данных**.

---

## 27.1 Что такое backpressure

**Backpressure** — механизм, при котором **замедление consumer'а замедляет producer'а**.

### Идея

В любой системе с потоком данных есть **producer** (создаёт данные) и **consumer** (обрабатывает). Если producer **быстрее** consumer'а — данные **накапливаются** в буфере.

**Без backpressure:**

- Буфер **растёт бесконечно**.
- Память **растёт**.
- Через некоторое время — **OOM**.

**С backpressure:**

- Producer **замедляется** до скорости consumer'а.
- Буфер **ограничен**.
- Память **стабильна**.

### Аналогия: водопровод

**Без backpressure** — как если бы водопроводный кран **всегда открыт на полную**, а сток узкий. Вода накапливается, рано или поздно затапливает квартиру.

**С backpressure** — как если бы кран **закрывался**, когда раковина полна. Вода **не переливается**.

### Где встречается backpressure

**1. Потоки данных.**

- Kafka → consumer.
- БД → app.
- API → app.

**2. Внутренние пайплайны.**

- Стадия A → стадия B.
- Worker pool → collector.

**3. Сетевые протоколы.**

- TCP **имеет** backpressure через **window size**.
- HTTP/2 **имеет** flow control.

**4. Пользовательские интерфейсы.**

- Блокировка кнопки во время загрузки.
- Ограничение частоты клика.

### Backpressure vs rate limiter

**Backpressure** — **замедление** producer'а до скорости consumer'а.

**Rate limiter** — **ограничение** скорости до заданного значения.

**Разница:**

- Backpressure: consumer **медленный** — producer **ждёт**.
- Rate limiter: producer **ограничен** — consumer **ждёт**.

**Вместе:** rate limiter ограничивает producer'а **сверху**, backpressure — **снизу**.

### Backpressure vs buffering

**Buffering** — **сглаживание** пиков. Буфер накапливает данные, если producer временно быстрее.

**Backpressure** — **замедление** producer'а, если он **постоянно** быстрее.

**Разница:**

- Buffer — **временное** решение.
- Backpressure — **постоянное**.

**Вместе:** буфер **небольшой** (сглаживает), backpressure **автоматический** (замедляет).

### 💡 Практика: как думать о backpressure

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Помни: backpressure — свойство каналов.**
2. **Не делай буферы бесконечными.**
3. **Мониторь длину буфера.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Backpressure по умолчанию** — с небуферизованным каналом.
5. **Буфер 10–1000** — для сглаживания пиков.

**❌ НЕ ДЕЛАЙ:**

6. **Не накапливай данные в памяти без ограничения.**
7. **Не игнорируй растущий буфер.**

---

## 27.2 Backpressure в Go: автоматически

В Go backpressure работает **автоматически** через каналы.

### Небуферизованный канал = строгий backpressure

```go
ch := make(chan Event)  // небуферизованный

// Producer
ch <- event  // ← блокируется, пока consumer не прочитает

// Consumer
event := <-ch  // ← блокируется, пока producer не напишет
```

**Что происходит:**

- Producer **не может** писать быстрее, чем consumer читает.
- Каждое событие — **синхронизация**.
- Буфер **нулевой** — данные не накапливаются.

**Это строгий backpressure.**

### Буферизованный канал = ослабленный backpressure

```go
ch := make(chan Event, 100)  // буфер на 100
```

**Что происходит:**

- Producer **может** писать быстрее, пока буфер не полон.
- Буфер **сглаживает** пики.
- При заполнении — producer **блокируется**.

**Это ослабленный backpressure.**

### Пример: автоматический backpressure

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int)  // небуферизованный
    
    // Producer: пишет быстро
    go func() {
        for i := 1; i <= 5; i++ {
            fmt.Printf("[producer] sending %d\n", i)
            ch <- i  // ← блокируется, пока consumer не прочитает
            fmt.Printf("[producer] sent %d\n", i)
        }
        close(ch)
    }()
    
    // Consumer: читает медленно
    for v := range ch {
        fmt.Printf("[consumer] received %d\n", v)
        time.Sleep(500 * time.Millisecond)  // ← медленно
    }
}
```

**Пример вывода:**

```
[producer] sending 1
[consumer] received 1
[producer] sent 1
[producer] sending 2
[consumer] received 2
...
```

**Что видно:** producer **ждёт** consumer'а. Backpressure работает **автоматически**.

### Схема

```
Producer:                    Consumer:
  ch <- 1  ──блокировка──►    <-ch
  (ждёт)                       (получил)
  ch <- 2  ──блокировка──►    <-ch
  ...
```

### Ключевое: backpressure не нужно реализовывать

**В Go backpressure — не паттерн, а свойство.** Каналы **автоматически** замедляют producer'а, если consumer не успевает. Не нужно писать дополнительный код.

**Что нужно:** **выбрать правильный размер буфера**. Об этом — 27.3.

### 💡 Практика: как использовать автоматический backpressure

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Небуферизованный канал** — строгий backpressure.
2. **Буферизованный канал 10–1000** — ослабленный.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Не делай буферы 100 000+** — маскирует проблему.

**❌ НЕ ДЕЛАЙ:**

4. **Не используй канал без backpressure** для потоков данных.

---

## 27.3 Размер буфера: как настроить

Размер буфера — **ключевой** параметр backpressure.

### Маленький буфер (0–10)

**Что даёт:**

- **Строгий backpressure.**
- Producer **часто блокируется**.
- **Минимум памяти.**

**Когда использовать:**

- **Критичные данные** (нельзя терять).
- **Consumer медленный.**
- **Низкая latency** (нет задержки в буфере).

### Средний буфер (10–1000)

**Что даёт:**

- **Сглаживание пиков.**
- Producer **не блокируется** при кратковременных пиках.
- **Умеренная память.**

**Когда использовать:**

- **Большинство случаев.**
- **Пиковая нагрузка.**
- **Сглаживание.**

### Большой буфер (1000–100 000)

**Что даёт:**

- **Маскировка** проблемы.
- Producer **редко блокируется**.
- **Много памяти.**

**Когда использовать:**

- **Крайне редко.**
- **Специальные сценарии.**

### Огромный буфер (100 000+)

**Что даёт:**

- **Backpressure не работает.**
- Producer **не блокируется**.
- **Память растёт.**

**Когда использовать:**

- **Никогда.** Это анти-паттерн.

### Как выбрать

**Правило:** буфер **сглаживает пики**, но **не маскирует** проблему.

**Рекомендации:**

- **Размер = 2–10× средняя пропускная способность**.
- **Не больше 10 000.**
- **Мониторь длину буфера.**

### Пример: выбор буфера

**Сценарий:** Kafka (10 000/сек) → app → БД (1000/сек).

**Без backpressure (буфер 10 000):**

- Буфер заполняется за 1 секунду.
- Память растёт.

**С backpressure (буфер 100):**

- Буфер заполняется за 10 мс.
- Consumer (Kafka) **блокируется** через 10 мс.
- Kafka **накапливает** события.
- App обрабатывает 1000/сек.
- Память **стабильна**.

**Что выбрать:** буфер **100**. Не 10 000.

### Схема

```
Kafka (10 000/сек) ──► ch (100) ──► app (1000/сек)
                       │
                       │ быстро заполняется
                       ▼
                  Kafka ждёт
                  (backpressure)
                  
Буфер НЕ маскирует проблему.
```

### 💡 Практика: как выбирать размер буфера

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Буфер 10–1000** — для большинства.
2. **Мониторь длину буфера.**
3. **Не маскируй проблему большим буфером.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Тестируй с разными размерами.**

**❌ НЕ ДЕЛАЙ:**

5. **Не ставь буфер 100 000+ без причины.**
6. **Не игнорируй растущий буфер.**

---

## 27.4 Три стратегии: block, drop, reject

Когда producer быстрее consumer'а, есть **три** стратегии.

### Стратегия 1: Block (backpressure)

**Producer ждёт**, пока consumer не готов.

```go
ch <- event  // ← блокируется
```

**Плюсы:**

- **Нет потери данных.**
- Backpressure работает автоматически.

**Минусы:**

- Producer **замедляется**.
- Если producer — внешняя система (Kafka), она **накапливает** данные.

**Когда использовать:**

- **Критичные данные.**
- **Внешняя система может буферизовать** (Kafka).

### Стратегия 2: Drop

**Producer пропускает** событие, если consumer не готов.

```go
select {
case ch <- event:
    // отправлено
default:
    // буфер полон — пропускаем
    metrics.Dropped.Add(1)
}
```

**Плюсы:**

- Producer **не блокируется**.
- Pамять **стабильна**.

**Минусы:**

- **Данные теряются.**

**Когда использовать:**

- **Некритичные данные** (метрики, логи).
- **Producer не может ждать** (real-time).

### Стратегия 3: Reject

**Producer возвращает ошибку**, если consumer не готов.

```go
select {
case ch <- event:
    return nil
default:
    return ErrQueueFull
}
```

**Плюсы:**

- Producer **не блокируется**.
- **Ошибка** — сигнал для клиента.

**Минусы:**

- Клиент **должен обработать** ошибку.

**Когда использовать:**

- **HTTP-хэндлеры.**
- **API с bulk endpoint.**
- **Клиент может повторить.**

### Сравнение

| Стратегия | Потеря данных | Блокировка | Use case |
|:---|:---|:---|:---|
| **Block** | Нет | Да | Критичные данные |
| **Drop** | Да | Нет | Метрики, логи |
| **Reject** | Нет | Нет | HTTP, API |

### Комбинация: block + timeout

```go
select {
case ch <- event:
    return nil
case <-time.After(100 * time.Millisecond):
    return ErrQueueFull
case <-ctx.Done():
    return ctx.Err()
}
```

**Что даёт:**

- Ждём **100 мс**.
- Если не получилось — **reject**.

**Когда использовать:**

- **Большинство случаев.**
- **Баланс** между ожиданием и reject.

### Пример: HTTP-хэндлер

**❌ Плохо (blocking):**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    tasksCh <- task  // ← если буфер полон — хэндлер виснет
    w.WriteHeader(http.StatusAccepted)
}
```

**✅ Хорошо (reject):**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    select {
    case tasksCh <- task:
        w.WriteHeader(http.StatusAccepted)
    default:
        http.Error(w, "server busy", http.StatusServiceUnavailable)
    }
}
```

**Что происходит:** если буфер полон — **503 Service Unavailable**.

### Схема

```
Producer:
  select {
  case ch <- event:        ← Block (если буфер полон — блокируется)
    ...
  default:                 ← Drop (пропустить)
    ...
  case <-time.After(100ms): ← Reject (после таймаута)
    return ErrQueueFull
  }
```

### 💡 Практика: как выбрать стратегию

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Block** — для критичных данных.
2. **Drop** — для метрик, логов.
3. **Reject** — для HTTP-хэндлеров.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Block + timeout** — для баланса.

**❌ НЕ ДЕЛАЙ:**

5. **Не используй Drop для критичных данных.**
6. **Не блокируй HTTP-хэндлер без таймаута.**

---

## 27.5 Backpressure в pipeline

В pipeline backpressure работает **между стадиями**.

### Как это работает

```
Source ──► Stage 1 ──► Stage 2 ──► Stage 3
           (быстро)    (медленно)   (быстро)
```

**Что происходит:**

- Stage 1 пишет в канал между Stage 1 и Stage 2.
- Stage 2 **медленный** — канал **заполняется**.
- Stage 1 **блокируется** на отправке.
- Source **блокируется** на отправке в Stage 1.
- Вся цепочка работает со скоростью **Stage 2**.

**Backpressure работает автоматически.**

### Пример

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 1000000)       // быстро
    parsed := parseStage(ctx, source)         // быстро
    enriched := enrichStage(ctx, parsed, 10)  // медленно
    written := writeStage(ctx, enriched)      // быстро
    
    for range written {
    }
}
```

**Что происходит:**

- `enrichStage` медленный.
- Канал между `parseStage` и `enrichStage` **заполняется**.
- `parseStage` **блокируется**.
- `source` **блокируется**.
- Вся цепочка **замедляется**.

### Размер буфера между стадиями

**Маленький буфер:**

- Быстрый backpressure.
- Producer блокируется часто.

**Большой буфер:**

- Отложенный backpressure.
- Больше памяти.

**Рекомендация:** буфер = 10–100.

### Схема

```
Source ──► parseStage ──► enrichStage ──► writeStage
           (быстро)       (медленно)      (быстро)
                │              │
                │              │
                └──── ch(100) ─┘
                       │
                       │ быстро заполняется
                       ▼
                  parseStage ждёт
                  (backpressure)
```

### 💡 Практика: как использовать backpressure в pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Буфер 10–100** между стадиями.
2. **Мониторь длину буфера.**
3. **Медленную стадию — fan-out.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Не делай буферы 100 000+.**

**❌ НЕ ДЕЛАЙ:**

5. **Не игнорируй растущий буфер.**
6. **Не маскируй проблему большим буфером.**

---

## 27.6 Backpressure в worker pool

В worker pool backpressure работает через **канал задач**.

### Как это работает

```
Producer ──► tasksCh (буфер) ──► N воркеров
```

**Что происходит:**

- Producer пишет в `tasksCh`.
- N воркеров читают.
- Если все воркеры заняты — `tasksCh` **заполняется**.
- Producer **блокируется** (backpressure).

### Размер буфера

**Маленький буфер (0–10):**

- Backpressure быстрый.
- Producer блокируется часто.

**Средний буфер (100–1000):**

- Сглаживание пиков.
- Backpressure отложенный.

**Большой буфер (10 000+):**

- Backpressure не работает.
- Память растёт.

**Рекомендация:** буфер = 2–10× число воркеров.

### Пример

```go
func main() {
    tasksCh := make(chan Task, 100)  // ← буфер 100
    
    // 10 воркеров
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
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
        tasksCh <- task  // ← backpressure при заполнении
    }
    close(tasksCh)
    wg.Wait()
}
```

### Worker pool + backpressure + rate limiter

**Разные инструменты для разных целей:**

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

**Что даёт:**

- **Backpressure** — через канал задач.
- **Rate limiter** — ограничение скорости.

### Схема

```
Producer ──► tasksCh (100) ──► 10 воркеров
                │                    │
                │ заполняется        │ медленно
                ▼                    ▼
           Producer ждёт         Воркеры заняты
           (backpressure)
```

### 💡 Практика: как использовать backpressure в worker pool

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Буфер = 2–10× число воркеров.**
2. **Мониторь длину буфера.**
3. **Backpressure + rate limiter.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Не делай буфер 10 000+.**

**❌ НЕ ДЕЛАЙ:**

5. **Не игнорируй растущий буфер.**

---

## 27.7 Мониторинг backpressure

Backpressure нужно **мониторить**. Иначе не видно проблемы.

### Что мониторить

**1. Длина буфера.**

```go
bufferLen := len(tasksCh)
bufferCap := cap(tasksCh)
usage := float64(bufferLen) / float64(bufferCap)
```

**Что означает:**

- **< 50%** — норма.
- **50–90%** — растёт нагрузка.
- **> 90%** — backpressure активен.

**2. Метрики.**

```go
type Metrics struct {
    QueueLength   atomic.Int64
    QueueCapacity atomic.Int64
    Blocked       atomic.Int64  // сколько раз producer блокировался
    Dropped       atomic.Int64  // сколько drop
    Rejected      atomic.Int64  // сколько reject
}
```

**3. Latency.**

**Что измерять:**

- **P50, P95, P99** обработки.
- **Очередь** — сколько событий ждёт.

**4. Throughput.**

**Что измерять:**

- **Producer rate** (события/сек).
- **Consumer rate** (события/сек).
- **Разница** — backpressure.

### Пример: метрики backpressure

```go
type Monitor struct {
    QueueLength  atomic.Int64
    BlockedCount atomic.Int64
    DroppedCount atomic.Int64
}

func (m *Monitor) AddWithBackpressure(ctx context.Context, ch chan<- Event, event Event) error {
    m.QueueLength.Store(int64(len(ch)))
    
    select {
    case ch <- event:
        return nil
    default:
        m.DroppedCount.Add(1)
        return ErrQueueFull
    }
}

func (m *Monitor) AddWithBlock(ctx context.Context, ch chan<- Event, event Event) error {
    m.QueueLength.Store(int64(len(ch)))
    
    select {
    case ch <- event:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

### Алерты

**Что алертить:**

- **Длина буфера > 90%** — backpressure активен.
- **Blocked count растёт** — consumer медленный.
- **Dropped count растёт** — данные теряются.
- **Latency растёт** — consumer не успевает.

### Схема

```
Метрики:
  queue_length: 90% ───► backpressure активен
  blocked_count: 1000/сек ───► consumer медленный
  dropped_count: 50/сек ───► данные теряются
  latency_p99: 5s ───► consumer не успевает
```

### 💡 Практика: как мониторить backpressure

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Длина буфера** — ключевая метрика.
2. **Blocked count.**
3. **Dropped count.**
4. **Latency.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Экспорт в Prometheus.**
6. **Алерты при > 90%.**

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй растущий буфер.**

---

## 27.8 В связке с другими паттернами

Backpressure редко используется **в одиночку**. Разберём связки.

### Backpressure + rate limiter

**Backpressure** — замедление. **Rate limiter** — ограничение.

```go
func worker(ctx context.Context, tasksCh <-chan Task, limiter *rate.Limiter) {
    for task := range tasksCh {
        if err := limiter.Wait(ctx); err != nil {
            return
        }
        process(task)
    }
}
```

**Что даёт:**

- Backpressure — через канал задач.
- Rate limiter — ограничение скорости.

### Backpressure + circuit breaker

**Backpressure** — замедление. **Circuit breaker** — защита.

```go
func worker(ctx context.Context, tasksCh <-chan Task, breaker *CircuitBreaker) {
    for task := range tasksCh {
        err := breaker.Call(ctx, func(ctx context.Context) error {
            return process(task)
        })
        if errors.Is(err, ErrCircuitOpen) {
            // Backpressure: ждём
            time.Sleep(100 * time.Millisecond)
        }
    }
}
```

### Backpressure + batch

**Backpressure** — управление потоком. **Batch** — группировка.

```go
func worker(ctx context.Context, in <-chan Event, batcher *Batcher[Event]) {
    for event := range in {
        if err := batcher.Add(ctx, event); err != nil {
            // Backpressure: batch не успевает
            log.Printf("batch error: %v", err)
        }
    }
}
```

### Backpressure + bulkhead

**Backpressure** — общий поток. **Bulkhead** — изоляция.

```go
type Services struct {
    dbBulkhead  *Bulkhead
    apiBulkhead *Bulkhead
}

func (s *Services) FetchFromDB(ctx context.Context, id int) (User, error) {
    return s.dbBulkhead.Execute(ctx, func(ctx context.Context) error {
        return db.GetUser(ctx, id)
    })
}
```

**Что даёт:**

- Bulkhead — изоляция между сервисами.
- Backpressure — внутри каждого bulkhead.

### Полная защита

```
Producer
   │
   ▼
┌──────────────┐
│  Rate limiter│  ← ограничение скорости
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Bulkhead    │  ← изоляция
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Channel    │  ← backpressure
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Worker pool │  ← обработка
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Circuit     │  ← защита от сбоев
│  breaker     │
└──────┬───────┘
       │
       ▼
     External
```

### 💡 Практика: как комбинировать backpressure

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Backpressure + rate limiter** — замедление + ограничение.
2. **Backpressure + circuit breaker** — замедление + защита.
3. **Backpressure + batch** — управление потоком.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Backpressure + bulkhead** — изоляция.

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай про мониторинг.**

---

## 27.9 Практика Go: backpressure с метриками

Разберём **backpressure с метриками**.

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

var ErrQueueFull = errors.New("queue full")

type Metrics struct {
    Produced   atomic.Int64
    Consumed   atomic.Int64
    Blocked    atomic.Int64
    Dropped    atomic.Int64
    QueueLen   atomic.Int64
    QueueCap   atomic.Int64
}

type Pipeline struct {
    ch      chan int
    metrics *Metrics
}

func NewPipeline(bufferSize int) *Pipeline {
    p := &Pipeline{
        ch:      make(chan int, bufferSize),
        metrics: &Metrics{},
    }
    p.metrics.QueueCap.Store(int64(bufferSize))
    return p
}

// ProduceBlock — блокирующий producer (backpressure).
func (p *Pipeline) ProduceBlock(ctx context.Context, item int) error {
    p.metrics.Produced.Add(1)
    
    select {
    case p.ch <- item:
        p.metrics.QueueLen.Store(int64(len(p.ch)))
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

// ProduceDrop — drop при заполнении.
func (p *Pipeline) ProduceDrop(item int) error {
    p.metrics.Produced.Add(1)
    
    select {
    case p.ch <- item:
        p.metrics.QueueLen.Store(int64(len(p.ch)))
        return nil
    default:
        p.metrics.Dropped.Add(1)
        return ErrQueueFull
    }
}

// ProduceBlockTimeout — block с timeout.
func (p *Pipeline) ProduceBlockTimeout(ctx context.Context, item int, timeout time.Duration) error {
    p.metrics.Produced.Add(1)
    
    select {
    case p.ch <- item:
        p.metrics.QueueLen.Store(int64(len(p.ch)))
        return nil
    case <-time.After(timeout):
        p.metrics.Blocked.Add(1)
        return ErrQueueFull
    case <-ctx.Done():
        return ctx.Err()
    }
}

func (p *Pipeline) Consume(ctx context.Context, workers int) {
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case item, ok := <-p.ch:
                    if !ok {
                        return
                    }
                    p.metrics.Consumed.Add(1)
                    p.metrics.QueueLen.Store(int64(len(p.ch)))
                    process(item)
                }
            }
        }()
    }
    wg.Wait()
}

func (p *Pipeline) Close() {
    close(p.ch)
}

func (p *Pipeline) Metrics() *Metrics {
    return p.metrics
}

func process(item int) {
    time.Sleep(1 * time.Millisecond)
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()
    
    p := NewPipeline(100)  // буфер на 100
    
    // 5 воркеров
    go p.Consume(ctx, 5)
    
    // Producer: пытаемся 10 000 раз
    for i := 0; i < 10000; i++ {
        if err := p.ProduceBlockTimeout(ctx, i, 10*time.Millisecond); err != nil {
            // reject
        }
    }
    
    p.Close()
    <-ctx.Done()
    
    m := p.Metrics()
    fmt.Println("=== Metrics ===")
    fmt.Printf("Produced:  %d\n", m.Produced.Load())
    fmt.Printf("Consumed:  %d\n", m.Consumed.Load())
    fmt.Printf("Blocked:   %d\n", m.Blocked.Load())
    fmt.Printf("Dropped:   %d\n", m.Dropped.Load())
    fmt.Printf("QueueLen:  %d\n", m.QueueLen.Load())
    fmt.Printf("QueueCap:  %d\n", m.QueueCap.Load())
}
```

**Пример вывода:**

```
=== Metrics ===
Produced:  10000
Consumed:  2000
Blocked:   8000
Dropped:   0
QueueLen:  100
QueueCap:  100
```

**Что видно:**

- **Produced** 10 000 — producer пытался.
- **Consumed** 2000 — consumer обработал.
- **Blocked** 8000 — producer заблокировался (backpressure).
- **QueueLen** 100 — буфер полон.

**Backpressure работает:** producer **замедлился** до скорости consumer'а.

### Пример: сравнение стратегий

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()
    
    fmt.Println("=== Block ===")
    testBlock(ctx)
    
    fmt.Println("\n=== Drop ===")
    testDrop(ctx)
    
    fmt.Println("\n=== Block with timeout ===")
    testBlockTimeout(ctx)
}

func testBlock(ctx context.Context) {
    p := NewPipeline(100)
    go p.Consume(ctx, 5)
    defer p.Close()
    
    for i := 0; i < 10000; i++ {
        p.ProduceBlock(ctx, i)
    }
    
    fmt.Printf("Produced: %d, Consumed: %d\n",
        p.Metrics().Produced.Load(), p.Metrics().Consumed.Load())
}

func testDrop(ctx context.Context) {
    p := NewPipeline(100)
    go p.Consume(ctx, 5)
    defer p.Close()
    
    for i := 0; i < 10000; i++ {
        p.ProduceDrop(i)
    }
    
    fmt.Printf("Produced: %d, Consumed: %d, Dropped: %d\n",
        p.Metrics().Produced.Load(),
        p.Metrics().Consumed.Load(),
        p.Metrics().Dropped.Load())
}

func testBlockTimeout(ctx context.Context) {
    p := NewPipeline(100)
    go p.Consume(ctx, 5)
    defer p.Close()
    
    for i := 0; i < 10000; i++ {
        p.ProduceBlockTimeout(ctx, i, 10*time.Millisecond)
    }
    
    fmt.Printf("Produced: %d, Consumed: %d, Blocked: %d\n",
        p.Metrics().Produced.Load(),
        p.Metrics().Consumed.Load(),
        p.Metrics().Blocked.Load())
}
```

**Пример вывода:**

```
=== Block ===
Produced: 10000, Consumed: 2000

=== Drop ===
Produced: 10000, Consumed: 2000, Dropped: 8000

=== Block with timeout ===
Produced: 10000, Consumed: 2000, Blocked: 8000
```

**Что видно:**

- **Block:** producer **заблокировался** — 2000 обработано.
- **Drop:** producer **пропустил** — 8000 потеряно.
- **Block with timeout:** producer **заблокировался** — 8000 reject.

### 💡 Практика: как измерять backpressure

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики:** produced, consumed, blocked, dropped.
2. **Queue length** — ключевая метрика.
3. **Экспорт в Prometheus.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Алерт при queue length > 90%.**
5. **Алерт при dropped > 0.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `Mutex` для метрик.**
7. **Не игнорируй растущий буфер.**

---

## 27.10 Выводы и типичные ошибки

**Что мы узнали?**

Backpressure — механизм, при котором **замедление consumer'а замедляет producer'а**. В Go backpressure работает **автоматически** через каналы. **Небуферизованный канал** — строгий backpressure. **Буферизованный 10–1000** — ослабленный. **Три стратегии:** block, drop, reject. **Block** — для критичных. **Drop** — для метрик. **Reject** — для HTTP. **Мониторинг:** queue length, blocked, dropped. Комбинируется с rate limiter, circuit breaker, batch, bulkhead.

**Типичные ошибки:**

- ❌ **Огромный буфер (100 000+).** Backpressure не работает.
- ❌ **Маленький буфер (0–10).** Producer блокируется часто.
- ❌ **Drop для критичных данных.** Данные теряются.
- ❌ **Block без timeout в HTTP.** Висящие соединения.
- ❌ **Не мониторить длину буфера.**
- ❌ **Не алертить при dropped > 0.**
- ❌ **Игнорировать растущий буфер.**
- ❌ **Один буфер для всего.** Не учитывает разные стадии.
- ❌ **Не различать block/drop/reject.**
- ❌ **Batch без backpressure.** Буфер растёт.

---

## 27.11 Для быстрого повторения

- **Backpressure** — замедление producer'а до скорости consumer'а.
- **В Go** — автоматически через каналы.
- **Небуферизованный канал** — строгий backpressure.
- **Буферизованный 10–1000** — ослабленный.
- **Буфер 100 000+** — backpressure не работает.
- **Три стратегии:** block, drop, reject.
- **Block** — для критичных.
- **Drop** — для метрик, логов.
- **Reject** — для HTTP, API.
- **Block + timeout** — баланс.
- **Мониторинг:** queue length, blocked, dropped.
- **Алерты:** queue > 90%, dropped > 0.
- **Pipeline:** backpressure между стадиями.
- **Worker pool:** буфер = 2–10× число воркеров.
- **Backpressure + rate limiter** — замедление + ограничение.
- **Backpressure + circuit breaker** — замедление + защита.

---

## 27.12 Вопросы для самопроверки

1. Что такое backpressure? Какую задачу решает?
2. Как backpressure работает в Go?
3. Чем небуферизованный канал отличается от буферизованного?
4. Как выбрать размер буфера?
5. Назови три стратегии при переполнении.
6. Чем block отличается от drop?
7. Как работает backpressure в pipeline?
8. Как мониторить backpressure?

---

## 27.13 Ответы

### Ответ 1

**Backpressure** — механизм замедления producer'а до скорости consumer'а. Решает задачу: **не накапливать данные** в памяти бесконечно.

**Пример:** Kafka производит 10 000/сек, БД обрабатывает 1000/сек. Backpressure замедляет Kafka до 1000/сек.

### Ответ 2

**Backpressure в Go работает автоматически** через каналы:

- Небуферизованный канал — producer **блокируется**, пока consumer не прочитает.
- Буферизованный канал — producer блокируется, когда буфер **полон**.

**Не нужно писать дополнительный код** — только выбрать размер буфера.

### Ответ 3

**Небуферизованный:** строгий backpressure. Producer **блокируется** на каждой отправке, пока consumer не готов.

**Буферизованный:** ослабленный backpressure. Producer **не блокируется**, пока буфер не полон. Буфер **сглаживает** пики.

### Ответ 4

**Размер буфера:**

- **Маленький (0–10)** — строгий backpressure, минимум памяти.
- **Средний (10–1000)** — большинство случаев.
- **Большой (1000–100 000)** — маскирует проблему.
- **Огромный (100 000+)** — не работает.

**Рекомендация:** 2–10× средняя пропускная способность, но не больше 10 000.

### Ответ 5

**Три стратегии:**

1. **Block** — producer ждёт.
2. **Drop** — producer пропускает.
3. **Reject** — producer возвращает ошибку.

**Комбинация:** block + timeout — баланс.

### Ответ 6

**Block** — producer **ждёт**, пока consumer не готов. **Нет потери данных.** Producer замедляется.

**Drop** — producer **пропускает** событие. **Данные теряются.** Producer не блокируется.

**Block** — для критичных. **Drop** — для метрик.

### Ответ 7

**Backpressure в pipeline** работает **между стадиями**:

```
Source ──► Stage 1 ──► Stage 2 ──► Stage 3
           (быстро)    (медленно)   (быстро)
```

Stage 2 медленный → канал между Stage 1 и Stage 2 заполняется → Stage 1 блокируется → Source блокируется. Вся цепочка **замедляется** до скорости Stage 2.

### Ответ 8

**Мониторинг backpressure:**

1. **Queue length** — длина буфера.
2. **Blocked count** — сколько producer блокировался.
3. **Dropped count** — сколько потеряно.
4. **Latency** — задержка обработки.

**Алерты:**
- **Queue length > 90%.**
- **Dropped count > 0.**

---

## 27.14 Куда идти дальше?

Мы разобрали backpressure — управление потоком. Теперь мы умеем не тонуть в данных.

Но остаётся **важный вопрос**: какие **анти-паттерны** существуют в конкурентном коде? Что **не надо делать**?

- **Какие анти-паттерны существуют?** → **Глава 28: Анти-паттерны.**
- **Как тестировать конкурентный код?** → **Глава 29: Тестирование конкурентного кода.**
- **Реальные сценарии:** HTTP-сервер, очереди, ETL, scraping. → **Глава 30: Реальные сценарии.**

---

## 27.15 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Backpressure** | Замедление producer'а | До скорости consumer'а |
| **В Go** | Автоматически | Через каналы |
| **Небуферизованный** | Строгий backpressure | Producer блокируется |
| **Буферизованный** | Ослабленный | Буфер сглаживает |
| **Маленький буфер** | 0–10 | Строгий backpressure |
| **Средний буфер** | 10–1000 | Большинство случаев |
| **Огромный буфер** | 100 000+ | Не работает |
| **Block** | Producer ждёт | Критичные данные |
| **Drop** | Producer пропускает | Метрики, логи |
| **Reject** | Producer возвращает ошибку | HTTP, API |
| **Block + timeout** | Баланс | Большинство |
| **Pipeline** | Backpressure между стадиями | Автоматически |
| **Worker pool** | Буфер = 2–10× воркеров | — |
| **Мониторинг** | queue length, blocked, dropped | Ключевые метрики |
| **Алерты** | > 90%, dropped > 0 | — |
| **+ rate limiter** | Замедление + ограничение | — |
| **+ circuit breaker** | Замедление + защита | — |

🚦 **Ключевая идея:** Backpressure — механизм замедления producer'а до скорости consumer'а. **В Go работает автоматически** через каналы: небуферизованный — строгий, буферизованный — ослабленный. **Три стратегии:** block (критичные), drop (метрики), reject (HTTP). **Block + timeout** — баланс. **Размер буфера 10–1000** — для большинства; **100 000+ — не работает**. **Backpressure в pipeline** — автоматически между стадиями. **В worker pool** — через канал задач (буфер 2–10× воркеров). **Мониторинг:** queue length, blocked, dropped; алерты при > 90% и dropped > 0. Комбинируется с rate limiter, circuit breaker, batch, bulkhead. Не игнорируй растущий буфер.