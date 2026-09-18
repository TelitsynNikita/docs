# 🚫 Глава 13: Анти-паттерны и типичные ошибки

**Что вы узнаете:**
- Почему **утечки горутин** — самая частая проблема Go-сервисов.
- Как распознать **deadlock**, **livelock** и **starvation**.
- Что такое **гонки** и почему они не всегда проявляются.
- Почему **блокировка на канале без читателя** — утечка.
- Что такое **busy loop** и как его избежать.
- Почему **закрытие канала на читателе** — ошибка.
- Что такое **context.Value** для параметров и почему это плохо.
- Как **не блокировать** HTTP-хэндлер.
- Что такое **thundering herd** и как его избежать.
- Как **не создавать горутины без ограничения**.
- Что такое **ложное совместное использование** (false sharing).
- Как **не копировать** мьютексы и `WaitGroup`.

**После прочтения вы сможете:**
- Распознавать утечки горутин в своём коде.
- Находить deadlock и livelock.
- Избегать гонок через синхронизацию.
- Правильно закрывать каналы.
- Понимать, когда `context.Value` — плохо.
- Не блокировать HTTP-хэндлеры.
- Избегать thundering herd.
- Ограничивать параллелизм.
- Писать код без типичных ошибок.

---

## Содержание

- [13.0 Пролог: сервис, который умирает через час](#130-пролог-сервис-который-умирает-через-час)
- [13.1 Утечки горутин](#131-утечки-горутин)
- [13.2 Deadlock, livelock, starvation](#132-deadlock-livelock-starvation)
- [13.3 Гонки и непредсказуемое поведение](#133-гонки-и-непредсказуемое-поведение)
- [13.4 Блокировка на канале без читателя](#134-блокировка-на-канале-без-читателя)
- [13.5 Busy loop: как не жечь CPU](#135-busy-loop-как-не-жечь-cpu)
- [13.6 Закрытие канала на читателе](#136-закрытие-канала-на-читателе)
- [13.7 context.Value для параметров](#137-contextvalue-для-параметров)
- [13.8 Блокировка HTTP-хэндлера](#138-блокировка-http-хэндлера)
- [13.9 Thundering herd](#139-thundering-herd)
- [13.10 Неограниченный параллелизм](#1310-неограниченный-параллелизм)
- [13.11 Копирование мьютексов и WaitGroup](#1311-копирование-мьютексов-и-waitgroup)
- [13.12 False sharing](#1312-false-sharing)
- [13.13 Практика Go: находим анти-паттерны](#1313-практика-go-находим-анти-паттерны)
- [13.14 Выводы и типичные ошибки](#1314-выводы-и-типичные-ошибки)
- [13.15 Для быстрого повторения](#1315-для-быстрого-повторения)
- [13.16 Вопросы для самопроверки](#1316-вопросы-для-самопроверки)
- [13.17 Ответы](#1317-ответы)
- [13.18 Куда идти дальше?](#1318-куда-идти-дальше)
- [13.19 Чек-лист](#1319-чек-лист)

---

## 13.0 Пролог: сервис, который умирает через час

Ты пишешь сервис, который обрабатывает запросы из Kafka. Всё работает. Пока через час не начинают приходить алерты:

- **Память растёт** с 500 МБ до 8 ГБ.
- **Число горутин растёт** с 50 до 10 000.
- **CPU** в норме.
- **HTTP** отвечает медленно.

Ты смотришь на код:

```go
func consume(ctx context.Context, consumer *kafka.Consumer) {
    for {
        msg, err := consumer.Read()
        if err != nil {
            continue
        }
        go func() {
            process(msg)
        }()
    }
}
```

❓ **Что не так?** Ты запускаешь горутину на **каждое сообщение**. Если `process` тормозит (например, внешний API), горутины **накапливаются**. Через час — 10 000 горутин.

💡 **Решение:** **worker pool** (Глава 7). N воркеров, задачи через канал.

**Это утечка горутин** — самая частая проблема Go-сервисов.

> **Важный мост:** эта глава — **каталог ошибок**. Каждая подглава — конкретный анти-паттерн. Ты будешь узнавать свой код.

---

## 13.1 Утечки горутин

**Утечка горутин** — горутина, которая **никогда не завершится**.

### Причины

**1. Блокировка на канале без читателя/писателя.**

```go
go func() {
    ch <- 42  // ← если никто не читает — вечная блокировка
}()
```

**2. Range по незакрытому каналу.**

```go
go func() {
    for v := range ch {  // ← если ch не закрыт — вечный цикл
        process(v)
    }
}()
```

**3. Ожидание HTTP/БД без таймаута.**

```go
go func() {
    resp, _ := http.Get("https://slow.example.com")  // ← может висеть вечно
    _ = resp
}()
```

**4. `select` без `ctx.Done()`.**

```go
go func() {
    for {
        select {
        case task := <-tasksCh:
            process(task)
        // ← нет ctx.Done() — не завершится при отмене
        }
    }
}()
```

**5. Забытый `cancel()`.**

```go
func worker() {
    ctx, _ := context.WithCancel(context.Background())  // ← cancel потерян
    <-ctx.Done()  // ← никогда не сработает
}
```

### Как обнаружить

**1. `runtime.NumGoroutine()`:**

```go
go func() {
    for range time.Tick(10 * time.Second) {
        log.Printf("goroutines: %d", runtime.NumGoroutine())
    }
}()
```

**Что искать:** рост без падения.

**2. pprof goroutine dump:**

```bash
curl http://localhost:6060/debug/pprof/goroutine?debug=2
```

**Что искать:** много горутин в одном состоянии (`chan send`, `chan receive`).

**3. `goleak` в тестах:**

```go
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```

### Как исправить

**1. Буферизованный канал для «отправь и забудь».**

```go
// ❌ Плохо
ch := make(chan int)
go func() { ch <- 42 }()

// ✅ Хорошо
ch := make(chan int, 1)
go func() { ch <- 42 }()
```

**2. `context` для отмены.**

```go
go func() {
    for {
        select {
        case <-ctx.Done():
            return
        case task := <-tasksCh:
            process(task)
        }
    }
}()
```

**3. Таймауты на внешние вызовы.**

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()

req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
resp, err := http.DefaultClient.Do(req)
```

**4. `defer cancel()`.**

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()  // ← обязательно
```

### Пример: утечка и исправление

**❌ Утечка:**

```go
func processAll(items []Item) {
    for _, item := range items {
        go func(it Item) {
            result := process(it)
            resultsCh <- result  // ← если никто не читает — вечная блокировка
        }(item)
    }
}
```

**✅ Исправление:**

```go
func processAll(ctx context.Context, items []Item) ([]Result, error) {
    resultsCh := make(chan Result, len(items))  // ← буфер на все результаты

    for _, item := range items {
        item := item
        go func() {
            select {
            case resultsCh <- process(item):
            case <-ctx.Done():
                return
            }
        }()
    }

    var results []Result
    for i := 0; i < len(items); i++ {
        select {
        case r := <-resultsCh:
            results = append(results, r)
        case <-ctx.Done():
            return nil, ctx.Err()
        }
    }
    return results, nil
}
```

### Аннотация сложности

| Причина | Время до обнаружения |
|:---|:---|
| Канал без читателя | Секунды |
| Range без close | Секунды |
| HTTP без таймаута | Часы |
| Забытый cancel | Часы |

### 💡 Практика: как не допустить утечек

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`context` для всех долгоживущих горутин.**
2. **`defer cancel()`.**
3. **Таймауты на внешние вызовы.**
4. **`runtime.NumGoroutine()` в мониторинге.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **`goleak` в тестах.**
6. **pprof в production (localhost).**

**❌ НЕ ДЕЛАЙ:**

7. **Не создавай горутину без способа завершить.**
8. **Не блокируйся на канале без `ctx.Done()`.**
9. **Не забывай `close` для range.**

### Ключевые выводы подглавы 13.1

- **Утечка горутин** — горутина, которая не завершится.
- **Причины:** канал без читателя, range без close, HTTP без таймаута, забытый cancel.
- **Обнаружение:** `NumGoroutine`, pprof, `goleak`.
- **Исправление:** `context`, буфер, таймауты.

---

## 13.2 Deadlock, livelock, starvation

Разберём **три вида взаимной блокировки**.

### Deadlock

**Deadlock** — горутины **ждут друг друга** и **никогда** не завершатся.

**Пример 1: два мьютекса в разном порядке.**

```go
var mu1, mu2 sync.Mutex

// Горутина A
go func() {
    mu1.Lock()
    mu2.Lock()  // ← ждёт mu2
    mu2.Unlock()
    mu1.Unlock()
}()

// Горутина B
go func() {
    mu2.Lock()
    mu1.Lock()  // ← ждёт mu1
    mu1.Unlock()
    mu2.Unlock()
}()
```

**Что происходит:**

- A взял mu1, ждёт mu2.
- B взял mu2, ждёт mu1.
- **Deadlock.**

**Пример 2: каналы в кольце.**

```go
ch1 := make(chan int)
ch2 := make(chan int)

go func() { ch1 <- <-ch2 }()
go func() { ch2 <- <-ch1 }()

<-ch1  // ← ждёт ch1, который ждёт ch2, который ждёт ch1
```

**Runtime обнаруживает:**

```
fatal error: all goroutines are asleep - deadlock!
```

**Когда runtime НЕ обнаруживает:**

- Если есть **другие** активные горутины (например, `main` спит).
- Если задействованы **внешние** ресурсы (БД, HTTP).

### Livelock

**Livelock** — горутины **активны**, но **не продвигаются**.

**Пример: две горутины уступают друг другу.**

```go
for {
    if tryLock(&mu1) {
        if tryLock(&mu2) {
            // работа
            unlock(&mu2)
            unlock(&mu1)
            break
        }
        unlock(&mu1)
        // уступаем
        runtime.Gosched()
    }
    runtime.Gosched()
}
```

**Что происходит:** обе горутины берут mu1, видят, что mu2 занят, отпускают, повторяют. **Бесконечный цикл.**

**Как обнаружить:** CPU высокий, но работа не делается.

### Starvation

**Starvation** — одна горутина **никогда** не получает ресурс.

**Пример: `select` с приоритетом.**

```go
for {
    select {
    case v := <-highPriorityCh:
        process(v)
    case v := <-lowPriorityCh:  // ← может голодать
        process(v)
    }
}
```

**Что происходит:** если `highPriorityCh` всегда готов, `lowPriorityCh` голодает.

**В Go `select` случайный** — starvation маловероятен. Но **starvation на мьютексе** возможен (Глава 3, starvation mode).

### Как обнаружить

**1. Goroutine dump:**

```bash
kill -QUIT <pid>
```

**Что искать:** горутины в `semacquire`, `chan send`, `chan receive`.

**2. `-race`:**

Иногда находит гонки, которые приводят к deadlock.

**3. Таймауты:**

Deadlock обычно проявляется как **зависание**. Таймаут покажет.

### Как исправить

**1. Deadlock на мьютексах:** одинаковый порядок захвата.

```go
// ❌ Плохо: разный порядок
mu1.Lock()
mu2.Lock()

// ✅ Хорошо: одинаковый порядок везде
mu1.Lock()
mu2.Lock()
```

**2. Deadlock на каналах:** не создавать кольца.

**3. Livelock:** использовать блокирующие примитивы.

**4. Starvation:** fair locks, random selection.

### Аннотация сложности

| Проблема | Время до обнаружения |
|:---|:---|
| Deadlock | Секунды-минуты |
| Livelock | Минуты |
| Starvation | Часы |

### 💡 Практика: как избежать deadlock и livelock

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Одинаковый порядок захвата мьютексов.**
2. **`context` для отмены.**
3. **Таймауты на всё.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Goroutine dump при зависании.**
5. **Мониторинг CPU (livelock).**

**❌ НЕ ДЕЛАЙ:**

6. **Не создавай кольца каналов.**
7. **Не используй `tryLock` в цикле.**
8. **Не полагайся на порядок в `select`.**

### Ключевые выводы подглавы 13.2

- **Deadlock** — горутины ждут друг друга.
- **Livelock** — горутины активны, но не продвигаются.
- **Starvation** — одна горутина голодает.
- **Deadlock на мьютексах:** одинаковый порядок захвата.
- **Deadlock на каналах:** не создавать кольца.

---

## 13.3 Гонки и непредсказуемое поведение

Разберём **гонки** (data race) подробнее.

### Что такое data race

**Data race** — две горутины обращаются к **одной** памяти, хотя бы **одна пишет**, **без синхронизации**.

**Формально:** undefined behavior по спецификации Go.

### Почему гонки опасны

**1. Непроявляющиеся гонки.**

На x86 гонка может **не проявляться**. На ARM — проявляться. Код «работает» на ноутбуке, падает на проде.

**2. Непредсказуемость.**

Гонка может:

- Работать «правильно».
- Возвращать мусор.
- Падать с segmentation fault.
- Зацикливаться.

**3. Не воспроизводится.**

Гонка может проявляться **раз в час**. Сложно отладить.

### Примеры

**1. Счётчик без синхронизации.**

```go
var counter int

for i := 0; i < 10; i++ {
    go func() { counter++ }()  // ← гонка
}
```

**2. Запись в map.**

```go
m := make(map[int]int)

for i := 0; i < 10; i++ {
    go func(i int) { m[i] = i }(i)  // ← гонка
}

// fatal error: concurrent map writes
```

**3. Ленивая инициализация.**

```go
var config *Config

func getConfig() *Config {
    if config == nil {  // ← гонка
        config = loadConfig()
    }
    return config
}
```

**4. Флаг готовности.**

```go
var ready bool

go func() {
    data = "hello"
    ready = true  // ← гонка
}()

for !ready {}  // ← гонка
```

### Как обнаружить

**1. `-race`:**

```bash
go test -race ./...
go run -race main.go
```

**2. Тесты на ARM:**

ARM маскирует меньше гонок.

**3. Код-ревью:**

Искать разделяемые переменные без синхронизации.

### Как исправить

**1. `Mutex` для разделяемых данных.**

```go
var (
    mu      sync.Mutex
    counter int
)

mu.Lock()
counter++
mu.Unlock()
```

**2. `atomic` для простых случаев.**

```go
var counter atomic.Int64
counter.Add(1)
```

**3. Каналы для передачи владения.**

```go
resultsCh <- result  // один владелец
```

**4. `sync.Once` для инициализации.**

```go
var once sync.Once

once.Do(func() {
    config = loadConfig()
})
```

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `Mutex.Lock/Unlock` | ~15-25 нс |
| `atomic.Add` | ~5-10 нс |
| Канал | ~50-100 нс |

### 💡 Практика: как не допустить гонок

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`-race` в CI.**
2. **Синхронизация для разделяемых данных.**
3. **`sync.Once` для инициализации.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Тесты на ARM.**
5. **Код-ревью на гонки.**

**❌ НЕ ДЕЛАЙ:**

6. **Не полагайся на «работает у меня».**
7. **Не используй `bool` для флага без atomic.**
8. **Не пиши в map без `Mutex`.**

### Ключевые выводы подглавы 13.3

- **Data race** — undefined behavior.
- **Гонки могут не проявляться.**
- **x86 маскирует, ARM проявляет.**
- **`-race` — обязателен.**
- **Синхронизация: `Mutex`, `atomic`, каналы, `Once`.**

---

## 13.4 Блокировка на канале без читателя

Разберём **утечку через канал без читателя**.

### Проблема

```go
func process() {
    ch := make(chan int)
    go func() {
        ch <- 42  // ← вечная блокировка
    }()
    // функция возвращается, не прочитав
}
```

**Что происходит:**

- Горутина пишет в канал.
- Читателя нет.
- **Вечная блокировка.**

### Другие примеры

**1. Результат не нужен.**

```go
for _, item := range items {
    ch := make(chan Result)
    go func(it Item) {
        ch <- process(it)  // ← утечка
    }(item)
}
```

**2. Отправка в буфер, который полон.**

```go
ch := make(chan int, 10)

for i := 0; i < 1000; i++ {
    go func(i int) {
        ch <- i  // ← когда буфер полон — вечная блокировка
    }(i)
}
// никто не читает ch
```

**3. Ошибка не читается.**

```go
errCh := make(chan error)
go func() {
    errCh <- doWork()  // ← утечка
}()
// никто не читает errCh
```

### Как исправить

**1. Буферизованный канал для «отправь и забыть».**

```go
ch := make(chan int, 1)
go func() {
    ch <- 42  // не блокируется
}()
```

**2. `select` с `default`.**

```go
select {
case ch <- result:
default:
    // никто не читает — пропускаем
}
```

**3. `select` с `ctx.Done()`.**

```go
select {
case ch <- result:
case <-ctx.Done():
    return
}
```

**4. Всегда читать из канала.**

```go
go func() { ch <- 42 }()
result := <-ch  // читаем
```

### Паттерн: правильная передача результата

```go
func process(ctx context.Context, items []Item) ([]Result, error) {
    resultsCh := make(chan Result, len(items))  // ← буфер

    for _, item := range items {
        item := item
        go func() {
            select {
            case resultsCh <- process(item):
            case <-ctx.Done():
            }
        }()
    }

    results := make([]Result, 0, len(items))
    for i := 0; i < len(items); i++ {
        select {
        case r := <-resultsCh:
            results = append(results, r)
        case <-ctx.Done():
            return nil, ctx.Err()
        }
    }
    return results, nil
}
```

### Аннотация сложности

| Ситуация | Время до обнаружения |
|:---|:---|
| Один канал | Секунды |
| Много каналов | Минуты-часы |

### 💡 Практика: как не блокироваться на канале

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Буферизованный канал для «отправь и забыть».**
2. **`select` с `ctx.Done()`.**
3. **Всегда читать из канала.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`select` с `default`** — для неблокирующей отправки.

**❌ НЕ ДЕЛАЙ:**

5. **Не отправляй в небуферизованный канал без читателя.**
6. **Не забывай про `ctx`.**

### Ключевые выводы подглавы 13.4

- **Канал без читателя** — утечка.
- **Буферизованный канал** — для «отправь и забыть».
- **`select` с `ctx.Done()`** — для отмены.
- **Всегда читать из канала.**

---

## 13.5 Busy loop: как не жечь CPU

**Busy loop** — цикл, который **крутится без работы**.

### Проблема

**1. Ожидание флага.**

```go
var ready bool

go func() { ready = true }()

for !ready {}  // ← busy loop, 100% CPU
```

**Что происходит:** цикл крутится, `ready` может никогда не обновиться (кешируется в регистре).

**2. `select` без `default` на закрытом канале.**

```go
ch := make(chan int)
close(ch)

for {
    select {
    case v := <-ch:  // ← всегда готов, v = 0, ok = false
        process(v)
    }
}
```

**3. Ожидание с `time.Sleep`.**

```go
for !ready {
    time.Sleep(time.Microsecond)  // ← частое пробуждение
}
```

### Как исправить

**1. Канал для сигнала.**

```go
readyCh := make(chan struct{})
go func() {
    // ... работа ...
    close(readyCh)
}()

<-readyCh  // блокируется, пока не закроют
```

**2. `sync.Cond`.**

```go
var (
    mu    sync.Mutex
    cond  = sync.NewCond(&mu)
    ready bool
)

go func() {
    mu.Lock()
    ready = true
    cond.Signal()
    mu.Unlock()
}()

mu.Lock()
for !ready {
    cond.Wait()  // ← блокируется, не busy loop
}
mu.Unlock()
```

**3. `atomic` для флага.**

```go
var ready atomic.Bool

go func() { ready.Store(true) }()

for !ready.Load() {
    runtime.Gosched()  // ← добровольно отдать P
}
```

**4. `select` с проверкой `ok`.**

```go
for {
    select {
    case v, ok := <-ch:
        if !ok {
            return  // канал закрыт
        }
        process(v)
    }
}
```

**5. `time.Sleep` с разумной длительностью.**

```go
for !ready {
    time.Sleep(100 * time.Millisecond)  // ← реже пробуждение
}
```

### Пример: исправление busy loop

**❌ Busy loop:**

```go
func waitForReady() {
    for !ready {
        // busy loop
    }
}
```

**✅ Хорошо:**

```go
func waitForReady(ctx context.Context) error {
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
            if ready.Load() {
                return nil
            }
        }
    }
}
```

### Аннотация сложности

| Подход | CPU |
|:---|:---|
| Busy loop | 100% |
| `runtime.Gosched` | 100% (но отдаёт P) |
| `time.Sleep(1ms)` | ~1% |
| Канал | ~0% |

### 💡 Практика: как не делать busy loop

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Канал для сигнала.**
2. **`sync.Cond`** — для условий.
3. **`atomic` + `runtime.Gosched`** — для флагов.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`time.Sleep` с разумной длительностью.**
5. **`select` с проверкой `ok`.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `for !flag {}` без синхронизации.**
7. **Не крути `select` на закрытом канале.**
8. **Не спи слишком часто.**

### Ключевые выводы подглавы 13.5

- **Busy loop** — 100% CPU без работы.
- **Канал** — для сигнала.
- **`sync.Cond`** — для условий.
- **`time.Sleep`** — с разумной длительностью.
- **`select` с `ok`** — для закрытого канала.

---

## 13.6 Закрытие канала на читателе

**Правило:** канал закрывает **тот, кто пишет**.

### Проблема

**❌ Плохо:**

```go
func reader(ch chan int) {
    defer close(ch)  // ← закрываем на читателе
    for v := range ch {
        process(v)
    }
}

func writer(ch chan int) {
    ch <- 1  // ← паника: send on closed channel
}
```

**Что происходит:** читатель закрывает канал, писатель пишет → **паника**.

### Почему это ошибка

**1. Паника при отправке.**

Если канал закрыт, `ch <- v` → `panic: send on closed channel`.

**2. Нет способа «дописать».**

Читатель не знает, когда писатель закончит. Он может закрыть канал **до** последней записи.

**3. Множественные писатели.**

Если писателей несколько — читатель не может закрыть канал, потому что **другой писатель** может ещё писать.

### Как исправить

**1. Писатель закрывает.**

```go
func writer(ch chan int) {
    defer close(ch)  // ← писатель
    for i := 0; i < 10; i++ {
        ch <- i
    }
}

func reader(ch chan int) {
    for v := range ch {  // ← читатель только читает
        process(v)
    }
}
```

**2. Несколько писателей: `WaitGroup`.**

```go
func writeAll(ch chan int, writers int) {
    var wg sync.WaitGroup
    for i := 0; i < writers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 10; j++ {
                ch <- id*10 + j
            }
        }(i)
    }

    go func() {
        wg.Wait()
        close(ch)  // ← после всех писателей
    }()
}
```

**3. `context` для отмены.**

Если не нужно ждать всех писателей:

```go
ctx, cancel := context.WithCancel(context.Background())

go func() {
    select {
    case <-ctx.Done():
        return
    case <-done:
    }
}()
```

### Паттерн: правильное закрытие

```go
func producer(ch chan<- int, done <-chan struct{}) {
    defer close(ch)  // ← писатель закрывает
    for i := 0; ; i++ {
        select {
        case ch <- i:
        case <-done:
            return
        }
    }
}

func consumer(ch <-chan int) {
    for v := range ch {  // ← читатель читает до close
        process(v)
    }
}
```

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `close(ch)` | ~50-100 нс |
| `ch <- v` после close | panic |

### 💡 Практика: как правильно закрывать каналы

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Писатель закрывает канал.**
2. **`defer close(ch)`** — сразу после `make`.
3. **Несколько писателей: `wg.Wait()` + `close`.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`context` для отмены.**
5. **Направленные каналы** (`chan<-`, `<-chan`).

**❌ НЕ ДЕЛАЙ:**

6. **Не закрывай на читателе.**
7. **Не закрывай дважды.**
8. **Не пиши в закрытый канал.**

### Ключевые выводы подглавы 13.6

- **Писатель закрывает канал.**
- **Несколько писателей: `wg.Wait()` + `close`.**
- **Не закрывать на читателе.**
- **Не закрывать дважды.**
- **Не писать в закрытый канал.**

---

## 13.7 context.Value для параметров

**Анти-паттерн:** использовать `context.Value` для параметров.

### Проблема

**❌ Плохо:**

```go
func handler(ctx context.Context, w http.ResponseWriter, r *http.Request) {
    pageSize := ctx.Value("page_size").(int)  // ← неявный параметр
    users := db.ListUsers(pageSize)
    // ...
}
```

**Проблемы:**

1. **Нетипизированно.** `"page_size"` — строка, `.(int)` — type assertion.
2. **Неявно.** Не видно в сигнатуре.
3. **Паника.** Если `page_size` не установлен — `panic`.
4. **Не проверяется компилятором.** Опечатка → ошибка в runtime.

### Что должно быть в context.Value

**Только request-scoped данные:**

- **Trace ID** (для распределённого трейсинга).
- **Request ID** (для корреляции логов).
- **User ID** (после аутентификации).
- **Correlation ID.**

**Что НЕ должно:**

- **Опциональные параметры** (page_size, limit).
- **Конфигурация** (db_host).
- **Зависимости** (logger, db).
- **Обязательные параметры** (user_id, если он нужен всем).

### Как исправить

**1. Явные параметры.**

```go
// ❌ Плохо
func handler(ctx context.Context, w http.ResponseWriter, r *http.Request) {
    pageSize := ctx.Value("page_size").(int)
}

// ✅ Хорошо
func handler(ctx context.Context, w http.ResponseWriter, r *http.Request) {
    pageSize, _ := strconv.Atoi(r.URL.Query().Get("page_size"))
    if pageSize == 0 {
        pageSize = 10  // default
    }
}
```

**2. Структура параметров.**

```go
type ListParams struct {
    PageSize int
    Offset   int
}

func ListUsers(ctx context.Context, params ListParams) ([]User, error) {
    // ...
}
```

**3. Типизированные ключи для context.Value.**

Если **действительно** нужно `context.Value`:

```go
type contextKey int

const (
    userIDKey contextKey = iota
)

func WithUserID(ctx context.Context, id int) context.Context {
    return context.WithValue(ctx, userIDKey, id)
}

func UserID(ctx context.Context) (int, bool) {
    id, ok := ctx.Value(userIDKey).(int)
    return id, ok
}
```

### Пример: правильное использование

```go
// Middleware: user ID из JWT
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        userID := extractUserID(r)
        ctx := WithUserID(r.Context(), userID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Handler: user ID из ctx
func handler(w http.ResponseWriter, r *http.Request) {
    userID, ok := UserID(r.Context())
    if !ok {
        http.Error(w, "unauthorized", 401)
        return
    }
    // ...
}
```

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `ctx.Value` | O(depth) |
| Явный параметр | 0 |

### 💡 Практика: как использовать context.Value

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Только для request-scoped данных** (trace ID, user ID).
2. **Типизированные ключи.**
3. **Функции-обёртки** (`WithUserID`, `UserID`).

**👍 СТОИТ СДЕЛАТЬ:**

4. **Явные параметры** для всего остального.

**❌ НЕ ДЕЛАЙ:**

5. **Не передавай опциональные параметры.**
6. **Не передавай конфигурацию.**
7. **Не передавай зависимости.**
8. **Не используй строковые ключи.**

### Ключевые выводы подглавы 13.7

- **`context.Value`** — только для request-scoped данных.
- **Типизированные ключи + функции-обёртки.**
- **Явные параметры** для всего остального.
- **Не передавай конфигурацию, зависимости.**

---

## 13.8 Блокировка HTTP-хэндлера

**Анти-паттерн:** блокировка хэндлера на канале.

### Проблема

**❌ Плохо:**

```go
var tasksCh = make(chan Task, 100)

func handler(w http.ResponseWriter, r *http.Request) {
    task := parseTask(r)
    tasksCh <- task  // ← если буфер полон — хэндлер виснет
    w.WriteHeader(http.StatusAccepted)
}
```

**Что происходит:**

- Если буфер полон (100 задач), `tasksCh <- task` **блокируется**.
- Хэндлер **висит**.
- Клиент **ждёт**.
- 1000 запросов → 1000 висящих соединений.

### Как исправить

**1. `select` с `default`.**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    task := parseTask(r)
    select {
    case tasksCh <- task:
        w.WriteHeader(http.StatusAccepted)
    default:
        http.Error(w, "queue full", http.StatusServiceUnavailable)
    }
}
```

**Что происходит:** если буфер полон — возвращаем **503**.

**2. `select` с `ctx.Done()`.**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    task := parseTask(r)
    select {
    case tasksCh <- task:
        w.WriteHeader(http.StatusAccepted)
    case <-r.Context().Done():
        return  // клиент отключился
    case <-time.After(1 * time.Second):
        http.Error(w, "timeout", http.StatusRequestTimeout)
    }
}
```

**3. Worker pool внутри хэндлера.**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    result, err := processWithTimeout(r.Context(), parseTask(r))
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    json.NewEncoder(w).Encode(result)
}
```

**4. Семафор для ограничения.**

```go
var sem = make(chan struct{}, 100)

func handler(w http.ResponseWriter, r *http.Request) {
    select {
    case sem <- struct{}{}:
        defer func() { <-sem }()
    case <-time.After(1 * time.Second):
        http.Error(w, "too many requests", http.StatusTooManyRequests)
        return
    }
    // обработка
}
```

### Асинхронная обработка

**Паттерн:** хэндлер возвращает **202 Accepted** и `task_id`, клиент опрашивает статус.

```go
func handler(w http.ResponseWriter, r *http.Request) {
    task := parseTask(r)
    taskID := uuid.New()

    select {
    case tasksCh <- TaskWithID{ID: taskID, Task: task}:
        json.NewEncoder(w).Encode(map[string]string{
            "task_id": taskID,
            "status":  "pending",
        })
    default:
        http.Error(w, "queue full", 503)
    }
}

func getStatus(w http.ResponseWriter, r *http.Request) {
    taskID := r.URL.Query().Get("task_id")
    status := getTaskStatus(taskID)
    json.NewEncoder(w).Encode(status)
}
```

### Аннотация сложности

| Подход | Blocking |
|:---|:---|
| `tasksCh <- task` | Да |
| `select` + `default` | Нет |
| `select` + `ctx.Done()` | Нет |
| Семафор | Нет (с таймаутом) |

### 💡 Практика: как не блокировать хэндлер

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`select` с `default`** — для неблокирующей отправки.
2. **`select` с `ctx.Done()`** — для отмены.
3. **Семафор** — для ограничения.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Асинхронная обработка** — 202 + task_id.
5. **Таймауты** на всё.

**❌ НЕ ДЕЛАЙ:**

5. **Не отправляй в канал без `select`.**
6. **Не блокируй хэндлер на I/O.**
7. **Не забывай `ctx.Done()`.**

### Ключевые выводы подглавы 13.8

- **Блокировка хэндлера** — висящие соединения.
- **`select` с `default`** — неблокирующая отправка.
- **`select` с `ctx.Done()`** — отмена.
- **Семафор** — ограничение.
- **Асинхронная обработка** — 202 + task_id.

---

## 13.9 Thundering herd

**Thundering herd** — много горутин просыпаются одновременно и конкурируют за один ресурс.

### Проблема

**Пример: закрытие канала.**

```go
done := make(chan struct{})

for i := 0; i < 1000; i++ {
    go func() {
        <-done  // ← все 1000 горутин ждут
        doWork()  // ← все просыпаются одновременно
    }()
}

close(done)  // ← все 1000 просыпаются
```

**Что происходит:**

- 1000 горутин ждут `<-done`.
- `close(done)` разблокирует **все**.
- Все 1000 **одновременно** делают `doWork`.
- **Конкуренция за CPU, БД, сеть.**

### Другие примеры

**1. `sync.Cond.Broadcast`.**

```go
cond.Broadcast()  // ← все ждущие просыпаются
```

**2. Много горутин на одном канале.**

```go
for i := 0; i < 1000; i++ {
    go func() {
        for v := range ch {  // ← 1000 горутин конкурируют за чтение
            process(v)
        }
    }()
}
```

**3. Кэш с истечением.**

```go
func get(key string) Value {
    if cached, ok := cache[key]; ok {
        return cached
    }
    // ← 1000 горутин одновременно кэшируют
    value := fetchFromDB(key)
    cache[key] = value
    return value
}
```

### Как исправить

**1. Ограничить число горутин.**

```go
sem := make(chan struct{}, 10)  // ← 10 одновременно

for i := 0; i < 1000; i++ {
    go func() {
        sem <- struct{}{}
        defer func() { <-sem }()
        doWork()
    }()
}
```

**2. `singleflight` для кэша.**

```go
import "golang.org/x/sync/singleflight"

var group singleflight.Group

func get(key string) (Value, error) {
    v, err, _ := group.Do(key, func() (interface{}, error) {
        return fetchFromDB(key)
    })
    return v.(Value), err
}
```

**Что делает:** 1000 горутин — **один** запрос к БД, остальные ждут результат.

**3. Рандомизация пробуждения.**

```go
close(done)

// Вместо:
doWork()

// Лучше:
time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
doWork()
```

**4. `select` с `default`.**

```go
select {
case <-done:
    doWork()
default:
    // не сейчас
}
```

### Пример: singleflight

**❌ Плохо:**

```go
func getUser(id int) (*User, error) {
    if u, ok := cache.Get(id); ok {
        return u, nil
    }
    u, err := db.GetUser(id)  // ← 1000 горутин — 1000 запросов
    if err != nil {
        return nil, err
    }
    cache.Set(id, u)
    return u, nil
}
```

**✅ Хорошо:**

```go
import "golang.org/x/sync/singleflight"

var group singleflight.Group

func getUser(id int) (*User, error) {
    if u, ok := cache.Get(id); ok {
        return u, nil
    }
    v, err, _ := group.Do(fmt.Sprintf("user-%d", id), func() (interface{}, error) {
        u, err := db.GetUser(id)  // ← только один запрос
        if err != nil {
            return nil, err
        }
        cache.Set(id, u)
        return u, nil
    })
    if err != nil {
        return nil, err
    }
    return v.(*User), nil
}
```

### Аннотация сложности

| Подход | Проблема |
|:---|:---|
| `close(done)` | Все просыпаются |
| `cond.Broadcast` | Все просыпаются |
| Семафор | Ограничивает |
| `singleflight` | Один запрос |

### 💡 Практика: как избежать thundering herd

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Семафор** — для ограничения.
2. **`singleflight`** — для кэша.
3. **Ограничивать число горутин** на одном ресурсе.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Рандомизация** — для распределения.
5. **`select` с `default`** — для неблокирующих проверок.

**❌ НЕ ДЕЛАЙ:**

6. **Не `close` канал, если 1000 горутин ждут.**
7. **Не `Broadcast` без ограничения.**
8. **Не кэшируй без `singleflight`.**

### Ключевые выводы подглавы 13.9

- **Thundering herd** — много горутин просыпаются одновременно.
- **Семафор** — для ограничения.
- **`singleflight`** — для кэша.
- **Рандомизация** — для распределения.
- **Не `close` на 1000 горутин.**

---

## 13.10 Неограниченный параллелизм

**Анти-паттерн:** создавать горутину на каждую задачу без ограничения.

### Проблема

**❌ Плохо:**

```go
func processAll(urls []string) {
    for _, url := range urls {
        go func(u string) {
            resp, _ := http.Get(u)
            _ = resp
        }(url)
    }
}
```

**Что происходит:**

- 10 000 URL → 10 000 горутин.
- 10 000 TCP-соединений.
- 10 000 file descriptors (лимит 1024).
- **OOM** или **too many open files**.

### Другие примеры

**1. Обработка сообщений.**

```go
for {
    msg := consumer.Read()
    go process(msg)  // ← неограниченно
}
```

**2. Запросы к БД.**

```go
for _, id := range ids {
    go func(id int) {
        db.Query(id)  // ← 10 000 одновременных запросов
    }(id)
}
```

**3. Вычисления.**

```go
for _, item := range items {
    go compute(item)  // ← CPU-bound задачи
}
```

### Как исправить

**1. Worker pool.**

```go
func processAll(urls []string, numWorkers int) {
    urlCh := make(chan string, len(urls))

    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for url := range urlCh {
                resp, _ := http.Get(url)
                _ = resp
            }
        }()
    }

    for _, url := range urls {
        urlCh <- url
    }
    close(urlCh)
    wg.Wait()
}
```

**2. Семафор.**

```go
sem := make(chan struct{}, 20)

for _, url := range urls {
    sem <- struct{}{}
    go func(u string) {
        defer func() { <-sem }()
        resp, _ := http.Get(u)
        _ = resp
    }(url)
}
```

**3. `errgroup.SetLimit`.**

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(20)  // ← не более 20 горутин

for _, url := range urls {
    url := url
    g.Go(func() error {
        resp, err := http.Get(url)
        if err != nil {
            return err
        }
        defer resp.Body.Close()
        return nil
    })
}

if err := g.Wait(); err != nil {
    log.Fatal(err)
}
```

### Как выбрать N

**CPU-bound:** `N = GOMAXPROCS`.

**I/O-bound:** `N = 10-100`.

**Формула:** `N = GOMAXPROCS × (1 + wait_time / compute_time)`.

### Аннотация сложности

| Подход | Горутин | Память |
|:---|:---|:---|
| `go func` на задачу | 10 000 | ~23 МБ |
| Worker pool N=20 | 20 | ~50 КБ |
| Семафор N=20 | 20 + задачи | ~50 КБ |

### 💡 Практика: как ограничивать параллелизм

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Worker pool** — для многих задач.
2. **Семафор** — для ограничения.
3. **`errgroup.SetLimit`** — для простоты.

**👍 СТОИТ СДЕЛАТЬ:**

4. **N = GOMAXPROCS** — для CPU-bound.
5. **N = 10-100** — для I/O-bound.

**❌ НЕ ДЕЛАЙ:**

6. **Не создавай горутину на каждую задачу.**
7. **Не игнорируй file descriptors.**
8. **Не забывай про внешние лимиты.**

### Ключевые выводы подглавы 13.10

- **Неограниченный параллелизм** — OOM, лимит fd.
- **Worker pool** — для многих задач.
- **Семафор** — для ограничения.
- **`errgroup.SetLimit`** — для простоты.
- **N = GOMAXPROCS** для CPU, **10-100** для I/O.

---

## 13.11 Копирование мьютексов и WaitGroup

**Анти-паттерн:** копировать `sync.Mutex` или `sync.WaitGroup`.

### Проблема

**❌ Плохо:**

```go
type Counter struct {
    mu sync.Mutex
    n  int
}

func (c Counter) Inc() {  // ← c копируется
    c.mu.Lock()  // ← блокирует копию, не оригинал
    c.n++
    c.mu.Unlock()
}
```

**Что происходит:**

- `c` копируется при вызове метода.
- `c.mu.Lock()` блокирует **копию** мьютекса.
- Оригинал **не защищён**.
- **Data race.**

**То же для `WaitGroup`:**

```go
func process(wg sync.WaitGroup) {  // ← копия
    defer wg.Done()  // ← уменьшает копию
}
```

### Как исправить

**1. Указатель.**

```go
func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.n++
}
```

**2. `go vet`.**

```bash
go vet ./...
```

**Что находит:** `copylocks` — копирование `Mutex`, `WaitGroup`, `RWMutex`.

**3. `noCopy`.**

```go
type WaitGroup struct {
    noCopy noCopy
    // ...
}
```

**Что делает:** `go vet` видит `noCopy` и выдаёт ошибку.

### Пример: `go vet`

```bash
$ go vet ./...
./main.go:10:6: Inc passes lock by value: main.Counter contains sync.Mutex
```

### Проверка через `unsafe`

**Некоторые структуры имеют `noCopy`:**

- `sync.Mutex`
- `sync.RWMutex`
- `sync.WaitGroup`
- `sync.Once`
- `sync.Cond`
- `sync.Map`

**Все они не должны копироваться.**

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `go vet` | ~100 мс |
| Копирование `Mutex` | ~10-50 нс (но ломает) |

### 💡 Практика: как не копировать мьютексы

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Указатели для методов.**
2. **`go vet ./...` в CI.**
3. **`noCopy` в структурах.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Код-ревью на копирование.**

**❌ НЕ ДЕЛАЙ:**

5. **Не передавай `Mutex` по значению.**
6. **Не передавай `WaitGroup` по значению.**
7. **Не копируй структуры с мьютексами.**

### Ключевые выводы подглавы 13.11

- **Копирование `Mutex`** — ломает синхронизацию.
- **Копирование `WaitGroup`** — то же.
- **Указатели для методов.**
- **`go vet`** — находит копирование.
- **`noCopy`** — для структур.

---

## 13.12 False sharing

**False sharing** — разные переменные в одной кэш-линии, которые пишутся разными ядрами.

### Проблема

```go
type Counters struct {
    a atomic.Int64  // ← в одной кэш-линии
    b atomic.Int64  // ← с a
}

var counters Counters

// Горутина 1
go func() {
    for i := 0; i < 1e9; i++ {
        counters.a.Add(1)
    }
}()

// Горутина 2
go func() {
    for i := 0; i < 1e9; i++ {
        counters.b.Add(1)
    }
}()
```

**Что происходит:**

- `a` и `b` — в одной кэш-линии (64 байта).
- Горутина 1 пишет в `a`.
- Горутина 2 пишет в `b`.
- Кэш-линия **инвалидируется** при каждой записи.
- **Производительность падает в разы.**

### Как исправить

**1. Padding.**

```go
type Counters struct {
    a atomic.Int64
    _ [56]byte  // ← padding до 64 байт
    b atomic.Int64
    _ [56]byte
}
```

**Что делает:** `a` и `b` — в **разных** кэш-линиях.

**2. Разные структуры.**

```go
var (
    counterA atomic.Int64
    counterB atomic.Int64
)
```

**Что делает:** компилятор разместит их в разных местах памяти (обычно).

**3. `align` теги.**

```go
type Counters struct {
    a atomic.Int64 `align:"64"`
    b atomic.Int64 `align:"64"`
}
```

### Как обнаружить

**1. Бенчмарки с разными GOMAXPROCS.**

```bash
go test -bench=. -cpu=1,2,4,8 ./...
```

**Что искать:** резкое падение производительности с ростом `GOMAXPROCS`.

**2. pprof CPU.**

Если много времени в `atomic.Add` — возможно, false sharing.

### Когда это важно

**False sharing** критичен для **горячих** счётчиков.

**Не критичен**, если:

- Счётчики не в hot path.
- Записи редки.

### Аннотация сложности

| Подход | Performance |
|:---|:---|
| Без padding | ~10x медленнее |
| С padding | Baseline |

### 💡 Практика: как избежать false sharing

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Padding** для hot counters.
2. **Разные структуры** для независимых счётчиков.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Бенчмарки с разными GOMAXPROCS.**
4. **pprof CPU.**

**❌ НЕ ДЕЛАЙ:**

5. **Не оптимизируй преждевременно.** False sharing важен только для hot path.

### Ключевые выводы подглавы 13.12

- **False sharing** — разные переменные в одной кэш-линии.
- **Padding** — для hot counters.
- **Разные структуры** — для независимых счётчиков.
- **Бенчмарки с разными GOMAXPROCS** — для обнаружения.

---

## 13.13 Практика Go: находим анти-паттерны

Напишем **код с анти-паттернами** и **исправления**.

### Анти-паттерн 1: утечка горутин

```go
// ❌ Утечка
func badLeak() {
    ch := make(chan int)
    go func() {
        ch <- 42  // ← никто не читает
    }()
}

// ✅ Хорошо
func goodLeak(ctx context.Context) {
    ch := make(chan int, 1)
    go func() {
        select {
        case ch <- 42:
        case <-ctx.Done():
        }
    }()
}
```

### Анти-паттерн 2: busy loop

```go
// ❌ Busy loop
func badWait(ready *atomic.Bool) {
    for !ready.Load() {
        // 100% CPU
    }
}

// ✅ Хорошо
func goodWait(ctx context.Context, ready *atomic.Bool) error {
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
            if ready.Load() {
                return nil
            }
        }
    }
}
```

### Анти-паттерн 3: закрытие канала на читателе

```go
// ❌ Паника
func badClose(ch chan int) {
    defer close(ch)  // читатель закрывает
    for range ch {
    }
}

// ✅ Хорошо
func goodClose(ch chan int, done <-chan struct{}) {
    for {
        select {
        case <-done:
            return
        case v, ok := <-ch:
            if !ok {
                return
            }
            _ = v
        }
    }
}

func goodProducer(ch chan int, done <-chan struct{}) {
    defer close(ch)
    for i := 0; i < 10; i++ {
        select {
        case ch <- i:
        case <-done:
            return
        }
    }
}
```

### Анти-паттерн 4: неограниченный параллелизм

```go
// ❌ 10 000 горутин
func badParallel(urls []string) {
    for _, url := range urls {
        go func(u string) {
            http.Get(u)
        }(url)
    }
}

// ✅ Worker pool
func goodParallel(ctx context.Context, urls []string, numWorkers int) {
    urlCh := make(chan string, len(urls))

    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for url := range urlCh {
                req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
                resp, _ := http.DefaultClient.Do(req)
                if resp != nil {
                    resp.Body.Close()
                }
            }
        }()
    }

    for _, url := range urls {
        urlCh <- url
    }
    close(urlCh)
    wg.Wait()
}
```

### Анти-паттерн 5: копирование Mutex

```go
// ❌ Копирование
type badCounter struct {
    mu sync.Mutex
    n  int
}

func (c badCounter) Inc() {
    c.mu.Lock()
    c.n++
    c.mu.Unlock()
}

// ✅ Указатель
type goodCounter struct {
    mu sync.Mutex
    n  int
}

func (c *goodCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.n++
}
```

### Анти-паттерн 6: context.Value для параметров

```go
// ❌ Неявный параметр
func badHandler(ctx context.Context, w http.ResponseWriter, r *http.Request) {
    pageSize := ctx.Value("page_size").(int)
    _ = pageSize
}

// ✅ Явный параметр
func goodHandler(w http.ResponseWriter, r *http.Request) {
    pageSize, _ := strconv.Atoi(r.URL.Query().Get("page_size"))
    if pageSize == 0 {
        pageSize = 10
    }
    _ = pageSize
}
```

### Анти-паттерн 7: блокировка HTTP-хэндлера

```go
// ❌ Блокировка
func badHandlerBlocking(w http.ResponseWriter, r *http.Request) {
    tasksCh <- parseTask(r)  // ← виснет, если буфер полон
}

// ✅ Неблокирующая
func goodHandlerNonBlocking(w http.ResponseWriter, r *http.Request) {
    select {
    case tasksCh <- parseTask(r):
        w.WriteHeader(http.StatusAccepted)
    case <-r.Context().Done():
        return
    default:
        http.Error(w, "queue full", http.StatusServiceUnavailable)
    }
}
```

### Анти-паттерн 8: thundering herd

```go
// ❌ 1000 горутин конкурируют
func badHerd(keys []string) []Value {
    results := make([]Value, len(keys))
    for i, key := range keys {
        go func(i int, k string) {
            results[i] = fetchFromDB(k)  // ← 1000 запросов
        }(i, key)
    }
    return results
}

// ✅ singleflight
var group singleflight.Group

func goodHerd(keys []string) []Value {
    results := make([]Value, len(keys))
    var wg sync.WaitGroup
    for i, key := range keys {
        wg.Add(1)
        go func(i int, k string) {
            defer wg.Done()
            v, _, _ := group.Do(k, func() (interface{}, error) {
                return fetchFromDB(k), nil
            })
            results[i] = v.(Value)
        }(i, key)
    }
    wg.Wait()
    return results
}
```

### Полный пример: до и после

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "sync"
    "sync/atomic"
    "time"

    "golang.org/x/sync/singleflight"
)

// ============ ПЛОХО ============

func badService(ctx context.Context, urls []string) {
    for _, url := range urls {
        go func(u string) {
            resp, _ := http.Get(u)  // ← нет ctx
            if resp != nil {
                resp.Body.Close()
            }
        }(url)
    }
}

func badCounter() {
    var counter int
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++  // ← гонка
        }()
    }
    wg.Wait()
}

// ============ ХОРОШО ============

func goodService(ctx context.Context, urls []string, numWorkers int) {
    urlCh := make(chan string, len(urls))

    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for url := range urlCh {
                req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
                if err != nil {
                    continue
                }
                resp, err := http.DefaultClient.Do(req)
                if err != nil {
                    continue
                }
                resp.Body.Close()
            }
        }()
    }

    for _, url := range urls {
        urlCh <- url
    }
    close(urlCh)
    wg.Wait()
}

func goodCounter() {
    var counter atomic.Int64
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Add(1)
        }()
    }
    wg.Wait()
    fmt.Println("counter:", counter.Load())
}

var sfGroup singleflight.Group

func goodCache(keys []string) []string {
    results := make([]string, len(keys))
    var wg sync.WaitGroup
    for i, key := range keys {
        wg.Add(1)
        go func(i int, k string) {
            defer wg.Done()
            v, _, _ := sfGroup.Do(k, func() (interface{}, error) {
                time.Sleep(100 * time.Millisecond)  // имитация БД
                return "value-" + k, nil
            })
            results[i] = v.(string)
        }(i, key)
    }
    wg.Wait()
    return results
}

func main() {
    ctx := context.Background()

    // Хороший сервис
    urls := []string{
        "https://example.com",
        // ... 1000 URL
    }
    goodService(ctx, urls, 20)

    // Хороший счётчик
    goodCounter()

    // Хороший кэш
    keys := []string{"a", "b", "c", "a", "b", "c"}
    results := goodCache(keys)
    fmt.Println(results)
}
```

### Аннотация сложности

| Анти-паттерн | Fix |
|:---|:---|
| Утечка горутин | Буфер + `ctx` |
| Busy loop | `time.Ticker` |
| Закрытие на читателе | Писатель закрывает |
| Неограниченный параллелизм | Worker pool |
| Копирование Mutex | Указатель |
| context.Value для параметров | Явные параметры |
| Блокировка хэндлера | `select` + `default` |
| Thundering herd | `singleflight` |

### 💡 Практика: как не допустить анти-паттернов

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`-race` в CI.**
2. **`go vet` в CI.**
3. **`goleak` в тестах.**
4. **Код-ревью.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **pprof в production.**
6. **Мониторинг `NumGoroutine`.**
7. **Stress-тесты.**

**❌ НЕ ДЕЛАЙ:**

8. **Не игнорируй предупреждения.**
9. **Не полагайся на «работает у меня».**

### Ключевые выводы подглавы 13.13

- **8 анти-паттернов** — утечки, busy loop, закрытие на читателе, неограниченный параллелизм, копирование Mutex, context.Value, блокировка хэндлера, thundering herd.
- **Каждый** имеет исправление.
- **`-race`, `go vet`, `goleak`** — инструменты.

---

## 13.14 Выводы и типичные ошибки

**Что мы узнали?**

Утечки горутин — самая частая проблема: канал без читателя, range без close, HTTP без таймаута, забытый cancel. Deadlock — горутины ждут друг друга. Livelock — активны, но не продвигаются. Starvation — одна голодает. Гонки — undefined behavior, x86 маскирует, ARM проявляет. Busy loop — 100% CPU. Закрытие канала на читателе — паника. `context.Value` для параметров — плохо. Блокировка HTTP-хэндлера — висящие соединения. Thundering herd — много горутин просыпаются. Неограниченный параллелизм — OOM. Копирование `Mutex` — ломает. False sharing — кэш-линии.

**Типичные ошибки:**

- ❌ **Утечка горутин.** `context`, буфер, таймауты.
- ❌ **Deadlock на мьютексах.** Одинаковый порядок захвата.
- ❌ **Гонки.** `-race`, синхронизация.
- ❌ **Канал без читателя.** Буфер или `ctx`.
- ❌ **Busy loop.** Канал, `time.Ticker`.
- ❌ **Закрытие канала на читателе.** Писатель закрывает.
- ❌ **`context.Value` для параметров.** Явные параметры.
- ❌ **Блокировка HTTP-хэндлера.** `select` + `default`.
- ❌ **Thundering herd.** `singleflight`, семафор.
- ❌ **Неограниченный параллелизм.** Worker pool.
- ❌ **Копирование `Mutex`.** Указатели.
- ❌ **False sharing.** Padding.
- ❌ **Не использовать `-race`.** Гонки в prod.
- ❌ **Не использовать `go vet`.** Копирование.

---

## 13.15 Для быстрого повторения

- **Утечка горутин** — самая частая проблема. `context`, буфер, таймауты.
- **Deadlock** — горутины ждут друг друга. Одинаковый порядок захвата.
- **Livelock** — активны, но не продвигаются.
- **Starvation** — одна голодает.
- **Гонки** — undefined behavior. x86 маскирует, ARM проявляет.
- **Busy loop** — 100% CPU. Канал, `time.Ticker`.
- **Закрытие канала на читателе** — паника. Писатель закрывает.
- **`context.Value` для параметров** — плохо. Явные параметры.
- **Блокировка HTTP-хэндлера** — висящие соединения. `select` + `default`.
- **Thundering herd** — много горутин просыпаются. `singleflight`.
- **Неограниченный параллелизм** — OOM. Worker pool.
- **Копирование `Mutex`** — ломает. `go vet`.
- **False sharing** — кэш-линии. Padding.
- **`-race` в CI.**
- **`go vet` в CI.**
- **`goleak` в тестах.**

---

## 13.16 Вопросы для самопроверки

1. Что такое утечка горутин? Назови три причины.
2. Как обнаружить утечку?
3. Что такое deadlock? Как избежать?
4. Что такое livelock?
5. Что такое starvation?
6. Что такое data race? Почему опасен?
7. Почему гонка не всегда проявляется?
8. Что такое блокировка на канале без читателя?
9. Что такое busy loop? Как избежать?
10. Почему закрытие канала на читателе — ошибка?
11. Почему `context.Value` для параметров — плохо?
12. Что такое блокировка HTTP-хэндлера?
13. Что такое thundering herd? Как избежать?
14. Что такое неограниченный параллелизм?
15. Почему копирование `Mutex` — ошибка?
16. Что такое false sharing?
17. Как избежать false sharing?
18. Какие инструменты помогут найти анти-паттерны?

---

## 13.17 Ответы

### Ответ 1

**Утечка горутин** — горутина, которая никогда не завершится.

**Три причины:**
1. Канал без читателя.
2. Range по незакрытому каналу.
3. HTTP без таймаута.

### Ответ 2

**Обнаружение:**
1. `runtime.NumGoroutine()` — рост без падения.
2. pprof goroutine dump.
3. `goleak` в тестах.

### Ответ 3

**Deadlock** — горутины ждут друг друга.

**Избежать:**
- Одинаковый порядок захвата мьютексов.
- Не создавать кольца каналов.
- Таймауты.

### Ответ 4

**Livelock** — горутины активны, но не продвигаются.

### Ответ 5

**Starvation** — одна горутина голодает.

### Ответ 6

**Data race** — две горутины обращаются к одной памяти, хотя бы одна пишет, без синхронизации.

**Опасен:** undefined behavior.

### Ответ 7

**Гонка не всегда проявляется**, потому что x86 маскирует гонки через строгую модель памяти.

### Ответ 8

**Блокировка на канале без читателя** — утечка. Горутина пишет в канал, никто не читает.

**Fix:** буфер, `select` с `ctx.Done()`.

### Ответ 9

**Busy loop** — цикл без работы.

**Избежать:** канал, `time.Ticker`, `sync.Cond`.

### Ответ 10

**Закрытие канала на читателе** — ошибка, потому что писатель может ещё писать. Паника.

### Ответ 11

**`context.Value` для параметров** — плохо, потому что:
- Нетипизированно.
- Неявно.
- Не проверяется компилятором.

### Ответ 12

**Блокировка HTTP-хэндлера** — хэндлер виснет на канале. Висящие соединения.

### Ответ 13

**Thundering herd** — много горутин просыпаются одновременно.

**Избежать:** семафор, `singleflight`.

### Ответ 14

**Неограниченный параллелизм** — горутина на каждую задачу. OOM.

### Ответ 15

**Копирование `Mutex`** — ошибка, потому что блокирует копию, а не оригинал.

### Ответ 16

**False sharing** — разные переменные в одной кэш-линии.

### Ответ 17

**Padding** — до 64 байт.

### Ответ 18

**Инструменты:** `-race`, `go vet`, `goleak`, pprof.

---

## 13.18 Куда идти дальше?

Мы разобрали анти-паттерны: утечки, deadlock, гонки, busy loop, неограниченный параллелизм. Теперь мы знаем, **чего не делать**.

Но остаётся **финальная тема**: как **применить** всё это в **реальных сценариях**?

- **HTTP-сервер:** ограничение параллелизма, graceful shutdown.
- **Очереди:** Kafka, NATS, worker pool.
- **ETL:** pipeline, batch, backpressure.
- **Scraping:** rate limiter, circuit breaker, retry.

→ **Глава 14: Реальные сценарии.**

---

## 13.19 Чек-лист

| Анти-паттерн | Fix |
|:---|:---|
| **Утечка горутин** | `context`, буфер, таймауты |
| **Deadlock** | Одинаковый порядок захвата |
| **Livelock** | Блокирующие примитивы |
| **Starvation** | Fair locks |
| **Data race** | `Mutex`, `atomic`, каналы |
| **Канал без читателя** | Буфер, `select` |
| **Busy loop** | Канал, `time.Ticker` |
| **Закрытие на читателе** | Писатель закрывает |
| **`context.Value` для параметров** | Явные параметры |
| **Блокировка хэндлера** | `select` + `default` |
| **Thundering herd** | `singleflight`, семафор |
| **Неограниченный параллелизм** | Worker pool |
| **Копирование `Mutex`** | Указатели |
| **False sharing** | Padding |
| **`-race`** | CI |
| **`go vet`** | CI |
| **`goleak`** | Тесты |
| **pprof** | Production (localhost) |

🚫 **Ключевая идея:** Анти-паттерны — каталог ошибок. **Утечки горутин** — самые частые: канал без читателя, range без close, HTTP без таймаута, забытый cancel. **Deadlock** — горутины ждут друг друга; одинаковый порядок захвата мьютексов. **Livelock** — активны, но не продвигаются. **Гонки** — undefined behavior; `-race` обязателен. **Busy loop** — 100% CPU; канал, `time.Ticker`. **Закрытие канала на читателе** — паника; писатель закрывает. **`context.Value` для параметров** — плохо; явные параметры. **Блокировка HTTP-хэндлера** — висящие соединения; `select` + `default`. **Thundering herd** — `singleflight`. **Неограниченный параллелизм** — OOM; worker pool. **Копирование `Mutex`** — ломает; указатели. **False sharing** — padding. **Инструменты:** `-race`, `go vet`, `goleak`, pprof.