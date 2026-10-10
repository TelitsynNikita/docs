# 🎛️ Глава 18: Debounce и Throttle — ограничение частоты событий

**Что вы узнаете:**
- Что такое debounce и throttle и какую задачу они решают.
- Чем debounce отличается от throttle и rate limiter.
- Как построить debounce с нуля.
- Как построить throttle с нуля.
- Как добавить отмену через `context`.
- Как комбинировать debounce и throttle с pipeline, worker pool, rate limiter.

**После прочтения вы сможете:**
- Построить debounce и throttle с нуля.
- Ограничивать частоту событий.
- Понимать, где debounce, где throttle, а где rate limiter.
- Комбинировать их с другими паттернами.

---

## Содержание

- [18.0 Пролог: слишком много событий](#180-пролог-слишком-много-событий)
- [18.1 Что такое debounce и throttle](#181-что-такое-debounce-и-throttle)
- [18.2 Debounce с нуля](#182-debounce-с-нуля)
- [18.3 Throttle с нуля](#183-throttle-с-нуля)
- [18.4 Debounce и throttle с context](#184-debounce-и-throttle-с-context)
- [18.5 Debounce vs throttle vs rate limiter](#185-debounce-vs-throttle-vs-rate-limiter)
- [18.6 В связке с другими паттернами](#186-в-связке-с-другими-паттернами)
- [18.7 Практика Go: debounce и throttle](#187-практика-go-debounce-и-throttle)
- [18.8 Выводы и типичные ошибки](#188-выводы-и-типичные-ошибки)
- [18.9 Для быстрого повторения](#189-для-быстрого-повторения)
- [18.10 Вопросы для самопроверки](#1810-вопросы-для-самопроверки)
- [18.11 Ответы](#1811-ответы)
- [18.12 Куда идти дальше?](#1812-куда-идти-дальше)
- [18.13 Чек-лист](#1813-чек-лист)

---

## 18.0 Пролог: слишком много событий

У нас есть сервис с HTTP-API, к которому подключаются клиенты. Иногда клиенты **сыплют запросами**: например, при вводе текста в поисковую строку фронтенд шлёт запрос на **каждую букву**. 5 букв — 5 запросов. 10 букв — 10 запросов.

Каждый запрос — обращение к БД с тяжёлым `LIKE`-поиском. При 1000 пользователей это **тысячи запросов в секунду**. База захлёбывается.

Простейшая мысль: **не обрабатывать все запросы**, а реагировать **только на последний**. Если пользователь печатает «hello», обработать только «hello», а не «h», «he», «hel», «hell».

Это и есть **debounce** — откладываем обработку до тех пор, пока события **не прекратятся** на N времени.

Другая ситуация: у нас есть поток событий от сенсора — 1000 событий в секунду. Мы хотим **обрабатывать не более 10 в секунду** — но обрабатывать **первое** из каждой «пачки», а не все. Это **throttle**.

Оба паттерна — про **ограничение частоты событий**, но с разной семантикой.

> **Мост к следующим главам:** debounce и throttle — специфические паттерны для потоков событий. Они часто комбинируются с pipeline (Глава 11), worker pool (Глава 13) и rate limiter (Глава 14). Понимание debounce/throttle даёт понимание, **как управлять потоком событий**.

---

## 18.1 Что такое debounce и throttle

Оба паттерна ограничивают частоту обработки, но **по-разному**.

### Debounce

**Debounce** — откладывает обработку до тех пор, пока события **не прекратятся** на N времени.

```
События:  x x x x x         (пауза)         x x x
Обработка:         ▲ (после паузы)             ▲
```

**Что происходит:**

- Событие 1 — ждём N времени.
- Событие 2 — сбрасываем таймер, ждём ещё N.
- Событие 3 — сбрасываем таймер.
- ...
- Пауза N времени — **обрабатываем**.

**Пример:** поиск в UI. Пользователь печатает «hello». Обработка только после того, как он перестал печатать на 300 мс.

### Throttle

**Throttle** — обрабатывает **не чаще** чем раз в N времени.

```
События:  x x x x x x x x x x
Обработка: ▲       ▲       ▲       ▲
           (каждые N времени)
```

**Что происходит:**

- Первое событие — обрабатываем **сразу**.
- Остальные в течение N времени — **игнорируем**.
- Через N времени — снова можно обработать.

**Пример:** сенсор 1000 событий/сек. Обрабатываем не чаще 10/сек.

### Сравнение

| Аспект | Debounce | Throttle |
|:---|:---|:---|
| Что делает | Откладывает до паузы | Обрабатывает раз в N |
| Первое событие | Откладывается | Обрабатывается сразу |
| Последнее событие | Обрабатывается | Может быть проигнорировано |
| Latency | N (до обработки) | 0 (первое сразу) |
| Пропускная способность | Максимум 1 за N | Максимум 1 за N |

### Когда использовать debounce

**1. Поиск в UI.**

Пользователь печатает — не нужно обрабатывать каждую букву.

**2. Автосохранение.**

Пользователь редактирует текст — сохранять после паузы.

**3. Валидация формы.**

Проверять после того, как пользователь закончил ввод.

**4. Обновление кэша.**

Не обновлять на каждое изменение — только после паузы.

### Когда использовать throttle

**1. Сенсоры.**

1000 событий/сек — обрабатывать не чаще 10/сек.

**2. Scroll-события.**

Обрабатывать не чаще 60/сек (по кадрам).

**3. Логи.**

Писать не чаще 10 строк/сек, чтобы не забить диск.

**4. Метрики.**

Отправлять не чаще 1/сек.

### Когда НЕ использовать debounce и throttle

**1. Все события важны.**

Если событие нельзя пропустить — не используй.

**2. Реакция должна быть мгновенной.**

Debounce добавляет latency.

**3. Простой поток.**

Если событий мало — не нужно ограничивать.

### 💡 Практика: как думать о debounce и throttle

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Debounce — для UI-событий.**
2. **Throttle — для потоков.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Debounce с окном 100–500 мс.**
4. **Throttle с окном 1/rate.**

**❌ НЕ ДЕЛАЙ:**

5. **Не используй debounce, если важны все события.**
6. **Не используй throttle, если нужна реакция на каждое.**

---

## 18.2 Debounce с нуля

Начнём с простейшего debounce.

### Идея

- Каждое событие **сбрасывает таймер**.
- Через N времени **без событий** — обрабатываем.

### Реализация

```go
func debounce(ctx context.Context, input <-chan int, delay time.Duration) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        
        var lastValue int
        var hasValue bool
        var timer *time.Timer
        
        for {
            var timerC <-chan time.Time
            if timer != nil {
                timerC = timer.C
            }
            
            select {
            case <-ctx.Done():
                if timer != nil {
                    timer.Stop()
                }
                return
                
            case v, ok := <-input:
                if !ok {
                    if timer != nil {
                        timer.Stop()
                    }
                    if hasValue {
                        select {
                        case out <- lastValue:
                        case <-ctx.Done():
                        }
                    }
                    return
                }
                
                lastValue = v
                hasValue = true
                
                if timer != nil {
                    timer.Stop()
                }
                timer = time.NewTimer(delay)
                
            case <-timerC:
                if hasValue {
                    select {
                    case out <- lastValue:
                    case <-ctx.Done():
                        return
                    }
                    hasValue = false
                }
                timer = nil
            }
        }
    }()
    
    return out
}
```

**Что происходит:**

- При новом событии — сбрасываем таймер.
- При срабатывании таймера — обрабатываем последнее значение.
- При закрытии входа — обрабатываем последнее значение и завершаемся.

### Потребитель

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    input := make(chan int, 100)
    
    // Симулируем ввод
    go func() {
        defer close(input)
        for i := 1; i <= 5; i++ {
            input <- i
            time.Sleep(50 * time.Millisecond)
        }
        time.Sleep(500 * time.Millisecond)  // пауза
        input <- 6
        time.Sleep(50 * time.Millisecond)
        input <- 7
    }()
    
    for v := range debounce(ctx, input, 300*time.Millisecond) {
        fmt.Printf("[%v] debounced: %d\n", time.Now().Format("15:04:05.000"), v)
    }
}
```

**Пример вывода:**

```
[15:04:05.450] debounced: 5
[15:04:06.150] debounced: 7
```

**Что видно:**

- Значения 1–5 пришли за 250 мс — обработано только 5 (последнее перед паузой).
- Через 500 мс пришло 6 и 7 — обработано только 7.

### Схема

```
События: 1 2 3 4 5                   6 7
         │ │ │ │ │                   │ │
         └─┴─┴─┴─┘                   └─┘
         таймер сбрасывается         таймер сбрасывается
              │                            │
         пауза 300 мс                 пауза 300 мс
              │                            │
              ▼                            ▼
         обработка 5                  обработка 7
```

### Обработка последнего значения

**Важно:** при **закрытии** входа debounce должен **обработать** последнее значение.

```go
case v, ok := <-input:
    if !ok {
        if hasValue {
            select {
            case out <- lastValue:
            case <-ctx.Done():
            }
        }
        return
    }
    // ...
```

**Почему:** если пользователь закончил ввод и закрыл соединение — последнее значение **не должно потеряться**.

### 💡 Практика: как писать debounce

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`time.NewTimer`** — для таймера.
2. **Сброс таймера** при каждом событии.
3. **Обработка последнего значения** при закрытии.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`ctx` для отмены.**
5. **Обёртка в структуру.**

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `timer.Stop()`.**
7. **Не теряй последнее значение.**

---

## 18.3 Throttle с нуля

Разберём throttle.

### Идея

- Первое событие — **обрабатываем сразу**.
- Остальные в течение N — **игнорируем**.
- Через N — можно снова.

### Реализация

```go
func throttle(ctx context.Context, input <-chan int, interval time.Duration) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        
        var lastTime time.Time
        
        for {
            select {
            case <-ctx.Done():
                return
                
            case v, ok := <-input:
                if !ok {
                    return
                }
                
                now := time.Now()
                if now.Sub(lastTime) >= interval {
                    lastTime = now
                    select {
                    case out <- v:
                    case <-ctx.Done():
                        return
                    }
                }
                // иначе — игнорируем
            }
        }
    }()
    
    return out
}
```

**Что происходит:**

- При новом событии — проверяем, прошло ли `interval` с последней обработки.
- Если прошло — обрабатываем, обновляем `lastTime`.
- Если нет — **игнорируем**.

### Потребитель

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    input := make(chan int, 100)
    
    // Быстрый поток событий
    go func() {
        defer close(input)
        for i := 1; i <= 20; i++ {
            input <- i
            time.Sleep(50 * time.Millisecond)
        }
    }()
    
    for v := range throttle(ctx, input, 200*time.Millisecond) {
        fmt.Printf("[%v] throttled: %d\n", time.Now().Format("15:04:05.000"), v)
    }
}
```

**Пример вывода:**

```
[15:04:05.000] throttled: 1
[15:04:05.200] throttled: 5
[15:04:05.400] throttled: 9
[15:04:05.600] throttled: 13
[15:04:05.800] throttled: 17
```

**Что видно:**

- Первое событие — сразу.
- Остальные — каждые 200 мс.
- 20 событий за 1000 мс → обработано 5.

### Схема

```
События: 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20
Обработка: ▲       ▲       ▲       ▲       ▲
           (каждые 200 мс)
```

### Throttle с обработкой последнего

**Проблема:** throttle **игнорирует** последнее событие, если оно попало в «окно».

**Пример:** события 1, 2. Throttle обработал 1. Событие 2 **игнорируется**. Но если 2 — это **последнее** значение, оно **потеряется**.

**Решение:** «trailing throttle» — обработать последнее после окна.

```go
func throttleTrailing(ctx context.Context, input <-chan int, interval time.Duration) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        
        var (
            lastTime    time.Time
            pendingVal  int
            hasPending  bool
            timer       *time.Timer
        )
        
        for {
            var timerC <-chan time.Time
            if timer != nil {
                timerC = timer.C
            }
            
            select {
            case <-ctx.Done():
                if timer != nil {
                    timer.Stop()
                }
                return
                
            case v, ok := <-input:
                if !ok {
                    if timer != nil {
                        timer.Stop()
                    }
                    return
                }
                
                now := time.Now()
                if now.Sub(lastTime) >= interval {
                    lastTime = now
                    select {
                    case out <- v:
                    case <-ctx.Done():
                        return
                    }
                } else {
                    pendingVal = v
                    hasPending = true
                    
                    if timer == nil {
                        wait := interval - now.Sub(lastTime)
                        timer = time.NewTimer(wait)
                    }
                }
                
            case <-timerC:
                if hasPending {
                    select {
                    case out <- pendingVal:
                    case <-ctx.Done():
                        return
                    }
                    hasPending = false
                    lastTime = time.Now()
                }
                timer = nil
            }
        }
    }()
    
    return out
}
```

**Что происходит:** throttle обрабатывает первое **и** последнее событие в окне.

### 💡 Практика: как писать throttle

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Первое событие — сразу.**
2. **`lastTime` для отслеживания.**
3. **Игнорировать события в окне.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Trailing throttle** — если важны последние.
5. **`ctx` для отмены.**

**❌ НЕ ДЕЛАЙ:**

6. **Не теряй последнее событие без причины.**

---

## 18.4 Debounce и throttle с context

`context` — критичен для debounce и throttle. Без него горутины могут **зависнуть**.

### Debounce с context

Мы уже добавили `ctx` в реализацию debounce. Разберём подробнее.

```go
case <-ctx.Done():
    if timer != nil {
        timer.Stop()
    }
    return
```

**Что происходит:** при отмене — останавливаем таймер, выходим.

### Throttle с context

Тоже добавлен:

```go
case <-ctx.Done():
    return
```

### Потребитель

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    input := make(chan int, 100)
    
    go func() {
        defer close(input)
        for i := 1; i <= 100; i++ {
            select {
            case input <- i:
            case <-ctx.Done():
                return
            }
            time.Sleep(50 * time.Millisecond)
        }
    }()
    
    count := 0
    for v := range debounce(ctx, input, 300*time.Millisecond) {
        fmt.Printf("[%v] debounced: %d\n", time.Now().Format("15:04:05.000"), v)
        count++
        if count >= 3 {
            cancel()
            break
        }
    }
    
    time.Sleep(50 * time.Millisecond)
    fmt.Println("done")
}
```

**Что происходит:** после 3 обработанных значений — `cancel()`. Debounce останавливается.

### Остановка таймера

**Важно:** при отмене `ctx` или закрытии входа — **остановить таймер**.

```go
if timer != nil {
    timer.Stop()
}
```

**Почему:** без `Stop` таймер продолжит работать до срабатывания. **Утечка.**

### Схема

```
Debounce с ctx:

  loop:
    select {
    case <-ctx.Done():     ← отмена
      timer.Stop()
      return
    case v := <-input:
      timer.Reset(delay)
    case <-timer.C:
      out <- v
    }
```

### 💡 Практика: как добавить context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx` — первый аргумент.**
2. **`select` с `ctx.Done()`.**
3. **`timer.Stop()` при отмене.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`defer cancel()` у потребителя.**

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай `timer.Stop()`.**
6. **Не блокируйся без `ctx`.**

---

## 18.5 Debounce vs throttle vs rate limiter

Разберём **разницу** между тремя паттернами.

### Сравнение

| Аспект | Debounce | Throttle | Rate limiter |
|:---|:---|:---|:---|
| Что делает | Откладывает до паузы | Обрабатывает раз в N | Ограничивает скорость |
| Что обрабатывает | Последнее | Первое (и последнее) | Все (с задержкой) |
| Latency | N | 0 (первое сразу) | Зависит от RPS |
| Пропускная способность | ≤ 1 за N | ≤ 1 за N | = RPS |
| Когда | UI-события | Потоки событий | Внешние API |
| Пропуск событий | Да (кроме последнего) | Да | Нет |

### Пример на одних данных

**События:** 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 — по 50 мс между ними.

**Debounce с окном 200 мс:**

- Событие 1 → таймер 200 мс.
- Событие 2 → сброс таймера.
- ...
- Событие 5 → таймер.
- Пауза 200 мс → **обработка 5**.
- События 6–10 → **обработка 10**.

**Результат:** 5, 10.

**Throttle с окном 200 мс:**

- Событие 1 → **обработка 1**.
- События 2–4 → игнорируются.
- Событие 5 → **обработка 5**.
- ...

**Результат:** 1, 5, 9, ...

**Rate limiter 10/сек:**

- Все события проходят, но с задержкой.
- Каждое событие ждёт своей «очереди».

**Результат:** все 1–10, но с интервалом 100 мс.

### Как выбрать

**Debounce:**

- **UI-события** (ввод, скролл, resize).
- **Важна реакция на последнее.**
- **Пауза — норма.**

**Throttle:**

- **Потоки событий** (сенсоры, метрики).
- **Важна реакция на первое.**
- **Пропуск событий — норма.**

**Rate limiter:**

- **Внешние API** с лимитом.
- **Важны все события.**
- **Задержка — норма.**

### Когда использовать вместе

**Debounce + rate limiter:**

Поиск в UI (debounce) + запрос к внешнему API с лимитом (rate limiter).

**Throttle + rate limiter:**

Метрики (throttle) + отправка в внешний сервис (rate limiter).

### Схема

```
Debounce:              Throttle:                Rate limiter:
  x x x x x              x x x x x                x x x x x
        │                    │                    │ │ │ │ │
        ▼                    ▼                    ▼ ▼ ▼ ▼ ▼
        ●                    ●   ●                ● ● ● ● ●
   (последнее)             (каждые N)              (все)
```

### 💡 Практика: как выбирать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Debounce — для UI.**
2. **Throttle — для потоков.**
3. **Rate limiter — для API.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Debounce + rate limiter** — UI + внешний API.
5. **Throttle + rate limiter** — метрики.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй debounce, если важны все события.**
7. **Не используй throttle для API с лимитом.**

---

## 18.6 В связке с другими паттернами

Debounce и throttle редко используются **в одиночку**. Разберём связки.

### Debounce + HTTP-хендлер

**Поиск в UI:**

```go
func main() {
    ctx := context.Background()
    
    // Канал событий от пользователя
    input := make(chan string, 100)
    debounced := debounceString(ctx, input, 300*time.Millisecond)
    
    // Обработка
    for query := range debounced {
        results := search(query)
        fmt.Printf("Search results for %q: %d\n", query, len(results))
    }
}

func debounceString(ctx context.Context, input <-chan string, delay time.Duration) <-chan string {
    out := make(chan string)
    
    go func() {
        defer close(out)
        
        var lastValue string
        var hasValue bool
        var timer *time.Timer
        
        for {
            var timerC <-chan time.Time
            if timer != nil {
                timerC = timer.C
            }
            
            select {
            case <-ctx.Done():
                if timer != nil {
                    timer.Stop()
                }
                return
            case v, ok := <-input:
                if !ok {
                    if timer != nil {
                        timer.Stop()
                    }
                    if hasValue {
                        select {
                        case out <- lastValue:
                        case <-ctx.Done():
                        }
                    }
                    return
                }
                lastValue = v
                hasValue = true
                if timer != nil {
                    timer.Stop()
                }
                timer = time.NewTimer(delay)
            case <-timerC:
                if hasValue {
                    select {
                    case out <- lastValue:
                    case <-ctx.Done():
                        return
                    }
                    hasValue = false
                }
                timer = nil
            }
        }
    }()
    return out
}
```

### Throttle + worker pool

**Метрики:**

```go
func main() {
    ctx := context.Background()
    
    metricsCh := make(chan Metric, 1000)
    throttled := throttleMetric(ctx, metricsCh, 100*time.Millisecond)
    
    // Worker pool для обработки
    var wg sync.WaitGroup
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for m := range throttled {
                processMetric(m)
            }
        }()
    }
    wg.Wait()
}
```

### Debounce + pipeline

**Debounce как стадия pipeline:**

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 100)
    debounced := debounceStage(ctx, source, 200*time.Millisecond)
    
    for v := range debounced {
        fmt.Println(v)
    }
}

func debounceStage(ctx context.Context, input <-chan int, delay time.Duration) <-chan int {
    // ... реализация debounce
    return out
}
```

### Throttle + rate limiter

**Throttle для событий, rate limiter для API:**

```go
func main() {
    ctx := context.Background()
    
    eventsCh := make(chan Event, 1000)
    throttled := throttle(ctx, eventsCh, 100*time.Millisecond)
    
    limiter := rate.NewLimiter(rate.Limit(10), 1)
    
    for e := range throttled {
        if err := limiter.Wait(ctx); err != nil {
            break
        }
        sendToAPI(e)
    }
}
```

### Полная защита

```
UI-события
   │
   ▼
┌──────────────┐
│   Debounce   │  ← откладывает до паузы
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Rate limiter │  ← ограничивает скорость
└──────┬───────┘
       │
       ▼
   Внешний API
```

### 💡 Практика: как комбинировать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Debounce + HTTP** — для UI.
2. **Throttle + worker pool** — для метрик.
3. **Debounce + rate limiter** — UI + API.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Debounce как стадия pipeline.**
5. **Throttle + rate limiter** — для потоков.

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `ctx`.**

---

## 18.7 Практика Go: debounce и throttle

Разберём **три примера**.

### Пример 1: debounce

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func debounce(ctx context.Context, input <-chan int, delay time.Duration) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        
        var lastValue int
        var hasValue bool
        var timer *time.Timer
        
        for {
            var timerC <-chan time.Time
            if timer != nil {
                timerC = timer.C
            }
            
            select {
            case <-ctx.Done():
                if timer != nil {
                    timer.Stop()
                }
                return
            case v, ok := <-input:
                if !ok {
                    if timer != nil {
                        timer.Stop()
                    }
                    if hasValue {
                        select {
                        case out <- lastValue:
                        case <-ctx.Done():
                        }
                    }
                    return
                }
                lastValue = v
                hasValue = true
                if timer != nil {
                    timer.Stop()
                }
                timer = time.NewTimer(delay)
            case <-timerC:
                if hasValue {
                    select {
                    case out <- lastValue:
                    case <-ctx.Done():
                        return
                    }
                    hasValue = false
                }
                timer = nil
            }
        }
    }()
    return out
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    input := make(chan int, 100)
    
    go func() {
        defer close(input)
        for i := 1; i <= 5; i++ {
            input <- i
            time.Sleep(50 * time.Millisecond)
        }
        time.Sleep(500 * time.Millisecond)
        input <- 6
        time.Sleep(50 * time.Millisecond)
        input <- 7
    }()
    
    for v := range debounce(ctx, input, 300*time.Millisecond) {
        fmt.Printf("[%v] debounced: %d\n", time.Now().Format("15:04:05.000"), v)
    }
}
```

**Пример вывода:**

```
[15:04:05.450] debounced: 5
[15:04:06.150] debounced: 7
```

### Пример 2: throttle

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func throttle(ctx context.Context, input <-chan int, interval time.Duration) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        
        var lastTime time.Time
        
        for {
            select {
            case <-ctx.Done():
                return
            case v, ok := <-input:
                if !ok {
                    return
                }
                
                now := time.Now()
                if now.Sub(lastTime) >= interval {
                    lastTime = now
                    select {
                    case out <- v:
                    case <-ctx.Done():
                        return
                    }
                }
            }
        }
    }()
    
    return out
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    input := make(chan int, 100)
    
    go func() {
        defer close(input)
        for i := 1; i <= 20; i++ {
            input <- i
            time.Sleep(50 * time.Millisecond)
        }
    }()
    
    for v := range throttle(ctx, input, 200*time.Millisecond) {
        fmt.Printf("[%v] throttled: %d\n", time.Now().Format("15:04:05.000"), v)
    }
}
```

**Пример вывода:**

```
[15:04:05.000] throttled: 1
[15:04:05.200] throttled: 5
[15:04:05.400] throttled: 9
[15:04:05.600] throttled: 13
[15:04:05.800] throttled: 17
```

### Пример 3: debounce vs throttle на одних данных

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Функция для создания одинакового входа
    makeInput := func() <-chan int {
        ch := make(chan int, 100)
        go func() {
            defer close(ch)
            for i := 1; i <= 10; i++ {
                ch <- i
                time.Sleep(50 * time.Millisecond)
            }
        }()
        return ch
    }
    
    fmt.Println("=== Debounce (200ms) ===")
    for v := range debounce(ctx, makeInput(), 200*time.Millisecond) {
        fmt.Println(v)
    }
    
    fmt.Println("\n=== Throttle (200ms) ===")
    for v := range throttle(ctx, makeInput(), 200*time.Millisecond) {
        fmt.Println(v)
    }
}
```

**Пример вывода:**

```
=== Debounce (200ms) ===
10

=== Throttle (200ms) ===
1
5
9
```

**Что видно:**

- **Debounce** обработал только **последнее** значение.
- **Throttle** обработал **каждое 4-е**.

### 💡 Практика: как использовать debounce и throttle

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Debounce — для UI.**
2. **Throttle — для потоков.**
3. **`ctx` — для отмены.**
4. **`timer.Stop()` при отмене.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Обёртка в структуру.**
6. **Trailing throttle** — если важны последние.

**❌ НЕ ДЕЛАЙ:**

7. **Не забывай `timer.Stop()`.**
8. **Не теряй последнее событие в debounce.**

---

## 18.8 Выводы и типичные ошибки

**Что мы узнали?**

Debounce откладывает обработку до паузы. Throttle обрабатывает не чаще чем раз в N. Оба используют `time.NewTimer` и `context`. Debounce обрабатывает **последнее**, throttle — **первое** (и последнее, если trailing). Rate limiter — ограничивает **скорость**, обрабатывает **все**. Debounce + rate limiter для UI + API, throttle + worker pool для метрик.

**Типичные ошибки:**

- ❌ **Забыть `timer.Stop()`.** Утечка таймера.
- ❌ **Потерять последнее значение в debounce.** Не обработать при закрытии.
- ❌ **Не использовать `ctx`.** Зависание.
- ❌ **Использовать debounce для потоков.** Пропуск событий.
- ❌ **Использовать throttle для API с лимитом.** Пропуск событий.
- ❌ **`time.After` в цикле.** Утечка.
- ❌ **Не различать debounce и throttle.**
- ❌ **Путать throttle и rate limiter.**

---

## 18.9 Для быстрого повторения

- **Debounce** — откладывает до паузы. Обрабатывает **последнее**.
- **Throttle** — обрабатывает раз в N. Обрабатывает **первое** (и последнее при trailing).
- **Debounce** — для UI-событий (ввод, скролл).
- **Throttle** — для потоков (сенсоры, метрики).
- **Rate limiter** — для внешних API.
- **`time.NewTimer`** — для таймеров.
- **`timer.Stop()`** при отмене.
- **`ctx`** для отмены.
- **Debounce обрабатывает последнее** при закрытии.
- **Throttle trailing** — обрабатывает последнее.
- **Debounce + rate limiter** — UI + API.
- **Throttle + worker pool** — метрики.
- **Debounce vs throttle:** последнее vs первое.
- **Throttle vs rate limiter:** пропуск vs задержка.

---

## 18.10 Вопросы для самопроверки

1. Что такое debounce? Какую задачу решает?
2. Что такое throttle? Какую задачу решает?
3. Чем debounce отличается от throttle?
4. Чем throttle отличается от rate limiter?
5. Как построить debounce с нуля?
6. Как построить throttle с нуля?
7. Зачем `timer.Stop()` в debounce?
8. Как комбинировать debounce с rate limiter?

---

## 18.11 Ответы

### Ответ 1

**Debounce** — откладывает обработку до тех пор, пока события **не прекратятся** на N времени.

**Решает задачу:** не обрабатывать **каждое** событие, а реагировать на **последнее**.

**Пример:** поиск в UI. Пользователь печатает — обрабатывать после паузы.

### Ответ 2

**Throttle** — обрабатывает события **не чаще** чем раз в N времени.

**Решает задачу:** ограничить частоту обработки, но обрабатывать **первое** из каждой «пачки».

**Пример:** сенсор 1000 событий/сек — обрабатывать не более 10/сек.

### Ответ 3

**Debounce:**

- Обрабатывает **последнее** событие в серии.
- Latency N (до обработки).

**Throttle:**

- Обрабатывает **первое** событие (и последнее при trailing).
- Latency 0.

**Debounce** — когда важен результат после паузы. **Throttle** — когда важна реакция на первое.

### Ответ 4

**Throttle** — обрабатывает 1 из N событий, **остальные пропускает**.

**Rate limiter** — обрабатывает **все** события, но с **задержкой**.

**Throttle** — для потоков, где пропуск норма. **Rate limiter** — для API, где важны все.

### Ответ 5

**Debounce:**

```go
func debounce(ctx context.Context, input <-chan int, delay time.Duration) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        var lastValue int
        var hasValue bool
        var timer *time.Timer
        
        for {
            var timerC <-chan time.Time
            if timer != nil {
                timerC = timer.C
            }
            
            select {
            case <-ctx.Done():
                if timer != nil { timer.Stop() }
                return
            case v, ok := <-input:
                if !ok {
                    if timer != nil { timer.Stop() }
                    if hasValue { out <- lastValue }
                    return
                }
                lastValue = v
                hasValue = true
                if timer != nil { timer.Stop() }
                timer = time.NewTimer(delay)
            case <-timerC:
                if hasValue { out <- lastValue; hasValue = false }
                timer = nil
            }
        }
    }()
    return out
}
```

### Ответ 6

**Throttle:**

```go
func throttle(ctx context.Context, input <-chan int, interval time.Duration) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        var lastTime time.Time
        for {
            select {
            case <-ctx.Done():
                return
            case v, ok := <-input:
                if !ok { return }
                now := time.Now()
                if now.Sub(lastTime) >= interval {
                    lastTime = now
                    select {
                    case out <- v:
                    case <-ctx.Done():
                        return
                    }
                }
            }
        }
    }()
    return out
}
```

### Ответ 7

**`timer.Stop()` в debounce** останавливает старый таймер, когда приходит новое событие (сбрасываем) или при отмене `ctx`.

**Без `Stop`:** старые таймеры продолжат работать до срабатывания. **Утечка.**

### Ответ 8

**Debounce + rate limiter:**

```go
ctx := context.Background()
input := make(chan string)
debounced := debounceString(ctx, input, 300*time.Millisecond)
limiter := rate.NewLimiter(rate.Limit(10), 1)

for q := range debounced {
    if err := limiter.Wait(ctx); err != nil { break }
    results := search(q)
    _ = results
}
```

Debounce — до паузы. Rate limiter — ограничение скорости к API.

---

## 18.12 Куда идти дальше?

Мы разобрали debounce и throttle — ограничение частоты событий. Теперь мы умеем управлять потоком.

Но иногда нужно **разветвить** один поток на несколько. Например, данные идут в лог **и** в обработку.

- **Как разветвить поток?** → **Глава 19: Tee и bridge каналы.**
- **Как построить pipeline?** → **Глава 11: Pipeline.**
- **Как ограничить скорость?** → **Глава 14: Rate limiter.**

---

## 18.13 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Debounce** | Откладывает до паузы | Обрабатывает последнее |
| **Throttle** | Обрабатывает раз в N | Обрабатывает первое |
| **Rate limiter** | Ограничивает скорость | Обрабатывает все |
| **`time.NewTimer`** | Таймер | `Stop` при отмене |
| **`ctx`** | Отмена | `select` с `ctx.Done()` |
| **Debounce latency** | N | До обработки |
| **Throttle latency** | 0 | Первое сразу |
| **Trailing throttle** | Обрабатывает последнее | После окна |
| **Debounce + rate limiter** | UI + API | — |
| **Throttle + worker pool** | Метрики | — |

🎛️ **Ключевая идея:** Debounce откладывает обработку до **паузы** — обрабатывает **последнее**. Throttle обрабатывает **не чаще раз в N** — обрабатывает **первое**. Rate limiter ограничивает **скорость** — обрабатывает **все** с задержкой. Debounce для UI, throttle для потоков, rate limiter для API. `time.NewTimer` + `timer.Stop()` — обязательно. `ctx` для отмены. Debounce + rate limiter для UI + API. Throttle + worker pool для метрик. Trailing throttle — если важны последние.