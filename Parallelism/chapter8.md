# 🌱 Глава 8: Generator — источник данных

**Что вы узнаете:**
- Что такое generator и какую задачу он решает.
- Как построить простейший generator.
- Как добавить отмену через `context`.
- Как добавить обработку ошибок.
- Как сделать generator с backpressure.
- Как комбинировать generator с другими паттернами.
- Как избежать утечек горутин в generator.

**После прочтения вы сможете:**
- Построить generator с нуля.
- Останавливать generator через `context`.
- Передавать ошибки из generator.
- Использовать generator в pipeline.
- Понимать, где generator уместен, а где — нет.

---

## Содержание

- [8.0 Пролог: откуда брать данные](#80-пролог-откуда-брать-данные)
- [8.1 Что такое generator](#81-что-такое-generator)
- [8.2 Простейший generator](#82-простейший-generator)
- [8.3 Generator с контекстом](#83-generator-с-контекстом)
- [8.4 Generator с ошибками](#84-generator-с-ошибками)
- [8.5 Generator с backpressure](#85-generator-с-backpressure)
- [8.6 В связке с другими паттернами](#86-в-связке-с-другими-паттернами)
- [8.7 Практика Go: генераторы данных](#87-практика-go-генераторы-данных)
- [8.8 Выводы и типичные ошибки](#88-выводы-и-типичные-ошибки)
- [8.9 Для быстрого повторения](#89-для-быстрого-повторения)
- [8.10 Вопросы для самопроверки](#810-вопросы-для-самопроверки)
- [8.11 Ответы](#811-ответы)
- [8.12 Куда идти дальше?](#812-куда-идти-дальше)
- [8.13 Чек-лист](#813-чек-лист)

---

## 8.0 Пролог: откуда брать данные

У нас есть сервис, который читает данные из источника и обрабатывает их. Источником может быть файл, база данных, внешний API, Kafka — что угодно. Работает так: читаем следующую запись, обрабатываем, читаем следующую.

```go
func processAll() error {
    for {
        record, err := source.Read()
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }
        
        if err := process(record); err != nil {
            return err
        }
    }
    return nil
}
```

Работает. Но есть проблема: обработка одного сообщения может быть **долгой**. Пока обрабатываем — источник простаивает. И если обработка включает fan-out, worker pool или rate limiter — придётся **дополнительно** продумывать, как связать источник с обработчиками.

Например, хочется запустить 10 воркеров параллельно:

```go
func processAll() error {
    for {
        record, err := source.Read()
        if err == io.EOF {
            break
        }
        go process(record)  // ← теперь надо думать про WaitGroup, ошибки, backpressure
    }
}
```

Теперь нужно: ограничить параллелизм (10 воркеров, не 10 000), собрать ошибки из воркеров, понять, когда все завершатся. Код растёт, смешивается.

Хочется вынести **чтение** в отдельный компонент: он знает, **откуда** брать данные, и отдаёт их в канал. А дальше — как хочешь: воркеры, pipeline, что угодно. Этот компонент — **generator**.

> **Мост к следующим главам:** generator — самый простой паттерн. Он лежит в основе fan-in (Глава 9), fan-out (Глава 10), pipeline (Глава 11). Понимание generator даёт понимание, как строить конвейеры данных.

---

## 8.1 Что такое generator

**Generator** — это функция, которая отдаёт данные в канал. Она сама решает, **откуда** брать данные и **когда** остановиться.

### Сигнатура

```go
func generator(...) <-chan T
```

**Ключевые свойства:**

- Возвращает **receive-only** канал — потребитель не может в него писать.
- Запускает **свою горутину** для чтения данных.
- **Закрывает канал**, когда данные закончились.
- Создаёт **одну точку**, где определяется «откуда данные».

### Схема

```
┌─────────────────────────────┐
│      Generator               │
│                              │
│  func generator():           │
│    1. Создать канал          │
│    2. Запустить горутину     │
│    3. Читать из источника    │
│    4. Писать в канал         │
│    5. close(out) при EOF     │
│    6. Вернуть out            │
│                              │
└──────────────┬──────────────┘
               │
               │ ←chan T
               ▼
        Потребитель
        for v := range ch {
           process(v)
        }
```

### Когда использовать generator

**1. Источник данных с логикой.**

- Чтение из файла построчно.
- Пагинация API.
- Обход дерева.
- Генерация последовательностей.

**2. Унификация источника.**

- Один интерфейс — много источников.
- Легко подменить Kafka на файл.

**3. Развязка чтения и обработки.**

- Generator знает только «как читать».
- Потребитель знает только «как обрабатывать».

### Когда НЕ использовать generator

**1. Одиночное чтение.**

Если нужно прочитать один объект — функция возвращает его напрямую, без канала.

**2. Маленький объём.**

Для 100 записей overhead канала может быть избыточным. Но обычно не критично.

**3. Данные уже в памяти.**

Если у тебя `[]Item` — просто верни слайс. Не нужно оборачивать в канал.

### 💡 Практика: как думать о generator

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Возвращай `<-chan T`**, не `chan T`.
2. **Закрывай канал, когда данные закончились.**
3. **Запускай горутину внутри generator.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Принимай `context`**, если чтение может быть долгим.

**❌ НЕ ДЕЛАЙ:**

5. **Не создавай generator без закрытия канала.** Утечка.
6. **Не используй generator, если данные уже в памяти.**

---

## 8.2 Простейший generator

Начнём с самого простого.

### Пример: генератор чисел

```go
func numbers(n int) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        for i := 0; i < n; i++ {
            out <- i
        }
    }()
    
    return out
}
```

**Что происходит:**

1. Создаём канал.
2. Запускаем горутину.
3. В горутине: пишем числа 0..n-1 в канал.
4. Когда закончили — `defer close(out)`.
5. Возвращаем канал.

### Потребитель

```go
func main() {
    for v := range numbers(10) {
        fmt.Println(v)
    }
}
```

**Что происходит:**

- `range` читает, пока канал открыт.
- Когда generator закрыл канал — цикл завершается.

### Полный код

```go
package main

import "fmt"

func numbers(n int) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        for i := 0; i < n; i++ {
            out <- i
        }
    }()
    
    return out
}

func main() {
    for v := range numbers(10) {
        fmt.Println(v)
    }
    fmt.Println("done")
}
```

**Пример вывода:**

```
0
1
2
3
4
5
6
7
8
9
done
```

### Generator из слайса

```go
func fromSlice(items []string) <-chan string {
    out := make(chan string)
    
    go func() {
        defer close(out)
        for _, item := range items {
            out <- item
        }
    }()
    
    return out
}
```

**Что делает:** оборачивает слайс в канал. Полезно, когда нужно единообразно работать с источниками — не важно, откуда данные, все они приходят через канал.

### Generator строк из файла

```go
func linesFromFile(path string) (<-chan string, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    
    out := make(chan string)
    
    go func() {
        defer close(out)
        defer f.Close()
        
        scanner := bufio.NewScanner(f)
        for scanner.Scan() {
            out <- scanner.Text()
        }
    }()
    
    return out, nil
}
```

**Что делает:** построчно читает файл и пишет в канал.

### Аналогия: конвейер

Generator — как **входная часть конвейера**. Он подхватывает сырьё (данные из источника) и кладёт на ленту (канал). Дальше лента несёт сырьё к обработчикам.

### 💡 Практика: как писать простой generator

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`defer close(out)`** сразу после `make(chan)`.
2. **Возвращай `<-chan T`** — receive-only.
3. **Пиши в канал в отдельной горутине.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Для файлов — `defer f.Close()` в той же горутине.**

**❌ НЕ ДЕЛАЙ:**

5. **Не пиши в канал из вызывающей горутины** — получится deadlock.
6. **Не забывай `close`.**

---

## 8.3 Generator с контекстом

Простейший generator не умеет останавливаться. Если потребитель перестал читать — горутина generator зависнет.

### Проблема

```go
func main() {
    for v := range numbers(1_000_000) {
        if v > 10 {
            break  // ← вышли из цикла
        }
        fmt.Println(v)
    }
}
```

**Что происходит:**

- `main` вышел из цикла после `v == 10`.
- Generator продолжает писать в канал.
- Канал небуферизованный — generator блокируется на `out <- v`.
- **Утечка горутины.**

### Решение: context

Добавляем `context` для отмены:

```go
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
```

**Что происходит:**

- Каждая отправка в канал проходит через `select`.
- Если `ctx` отменён — generator завершается.
- `defer close(out)` закрывает канал.

### Потребитель

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    for v := range numbersCtx(ctx, 1_000_000) {
        if v > 10 {
            cancel()  // отменяем generator
            break
        }
        fmt.Println(v)
    }
    
    // Даём время завершиться
    time.Sleep(10 * time.Millisecond)
}
```

**Что происходит:**

- При `v > 10` вызываем `cancel()`.
- Generator видит `ctx.Done()` и завершается.
- Утечки нет.

### Паттерн: таймаут

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()
    
    for v := range numbersCtx(ctx, 1_000_000) {
        fmt.Println(v)
    }
    // Через 1 секунду generator остановится
}
```

### Полный код

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

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    for v := range numbersCtx(ctx, 1_000_000) {
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

**Пример вывода:**

```
0
1
...
10
done
```

### Схема

```
Generator:
  for i := 0; i < n; i++ {
    select {
    case out <- i:        ← отправили
    case <-ctx.Done():    ← отмена
      return
    }
  }

Потребитель:
  for v := range ch {
    if v > 10 {
      cancel()  ────────► Generator видит ctx.Done()
      break
    }
  }
```

### 💡 Практика: как добавить context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx context.Context` — первый аргумент:**
   ```go
   func generator(ctx context.Context, ...) <-chan T
   ```

2. **`select` с `ctx.Done()` при отправке:**
   ```go
   select {
   case out <- v:
   case <-ctx.Done():
       return
   }
   ```

**👍 СТОИТ СДЕЛАТЬ:**

3. **`defer cancel()` у потребителя.**

**❌ НЕ ДЕЛАЙ:**

4. **Не пиши в канал без `select` в долгоживущем generator.**
5. **Не забывай `cancel()`.**

---

## 8.4 Generator с ошибками

Generator может столкнуться с ошибкой при чтении. Как её передать потребителю?

### Простой способ: Result с ошибкой

```go
type Result[T any] struct {
    Value T
    Err   error
}

func generatorWithErr(ctx context.Context) <-chan Result[string] {
    out := make(chan Result[string])
    
    go func() {
        defer close(out)
        for {
            v, err := readNext(ctx)
            select {
            case out <- Result[string]{Value: v, Err: err}:
            case <-ctx.Done():
                return
            }
            if err != nil {
                return
            }
        }
    }()
    
    return out
}
```

**Что происходит:**

- Каждый элемент канала — `Result{Value, Err}`.
- Потребитель проверяет `Err`.
- При ошибке generator **завершается** (после отправки).

### Потребитель

```go
for r := range generatorWithErr(ctx) {
    if r.Err != nil {
        log.Println("error:", r.Err)
        break
    }
    process(r.Value)
}
```

**Что происходит:**

- Если ошибка — логируем и выходим.
- Иначе — обрабатываем значение.

### Схема

```
Generator:
  for {
    v, err := readNext(ctx)
    out <- Result{v, err}
    if err != nil {
      return
    }
  }

Потребитель:
  for r := range ch {
    if r.Err != nil {
      // обработка
      break
    }
    process(r.Value)
  }
```

### Альтернатива: отдельный канал ошибок

```go
func generatorWithErrCh(ctx context.Context) (<-chan string, <-chan error) {
    out := make(chan string)
    errCh := make(chan error, 1)
    
    go func() {
        defer close(out)
        defer close(errCh)
        for {
            v, err := readNext(ctx)
            if err != nil {
                errCh <- err
                return
            }
            select {
            case out <- v:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    return out, errCh
}
```

**Что происходит:**

- Данные идут в `out`.
- Ошибки идут в `errCh`.
- Потребитель читает из обоих каналов.

**Плюсы:** чище — значения не заворачиваются в `Result`.

**Минусы:** нужно читать из двух каналов — либо через `select`, либо в отдельных горутинах.

### Что выбрать

| Способ | Когда |
|:---|:---|
| `Result[T]{Value, Err}` | Простой случай, ошибок мало |
| Отдельный `errCh` | Часто ошибки, сложная логика |

**Рекомендация:** для простых generator — `Result`. Для сложных pipeline — отдельные каналы.

### 💡 Практика: как передавать ошибки

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`Result[T]{Value, Err}` для простых случаев.**
2. **При ошибке — `return` после отправки.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Отдельный `errCh` для сложных pipeline.**

**❌ НЕ ДЕЛАЙ:**

4. **Не игнорируй ошибки в generator.**
5. **Не паникуй при ошибке — передавай через канал.**

---

## 8.5 Generator с backpressure

Backpressure — это когда генератор **не может** писать быстрее, чем потребитель читает. В Go это работает **автоматически** через небуферизованный канал.

### Небуферизованный канал = backpressure

```go
out := make(chan int)  // ← небуферизованный
```

**Что происходит:**

- Generator пытается писать в канал.
- Если потребитель не готов читать — generator **блокируется**.
- Generator работает со скоростью потребителя.

**Это и есть backpressure.**

### Буферизованный канал = нет backpressure

```go
out := make(chan int, 100)  // ← буфер на 100
```

**Что происходит:**

- Generator пишет в буфер.
- Пока буфер не полон — generator **не блокируется**.
- Только когда буфер полон — generator блокируется.

**Backpressure отложен.** Если producer быстрее consumer — буфер заполняется. Когда полон — generator всё равно блокируется.

### Пример: быстрый generator, медленный потребитель

```go
func fastGen() <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for i := 0; i < 100; i++ {
            out <- i  // ← блокируется, пока не прочитают
        }
    }()
    return out
}

func main() {
    for v := range fastGen() {
        time.Sleep(100 * time.Millisecond)  // ← медленный потребитель
        fmt.Println(v)
    }
}
```

**Что происходит:**

- Generator пишет число, блокируется.
- Потребитель спит 100 мс, читает, обрабатывает.
- Generator пишет следующее.
- **Generator работает со скоростью потребителя.**

### Пример: буферизованный generator

```go
func bufferedGen() <-chan int {
    out := make(chan int, 10)  // ← буфер 10
    go func() {
        defer close(out)
        for i := 0; i < 100; i++ {
            out <- i
        }
    }()
    return out
}
```

**Что происходит:**

- Generator пишет 10 чисел в буфер и продолжает.
- Когда буфер полон — блокируется.
- Потребитель медленный — буфер остаётся полным.
- **Отличие:** генератор имеет «запас» на 10 чисел.

### Когда какой буфер

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

## 8.6 В связке с другими паттернами

Generator — базовая часть pipeline. Разберём, как он комбинируется с другими паттернами.

### Generator + fan-out

Generator отдаёт данные, fan-out распределяет их между воркерами:

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 100)
    
    const workers = 5
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for v := range source {  // ← читают из одного канала
                fmt.Printf("Worker %d: %d\n", id, v)
            }
        }(i)
    }
    
    wg.Wait()
}
```

**Что происходит:**

- Generator отдаёт числа.
- 5 воркеров читают из **одного** канала.
- Распределение автоматическое.

### Generator + fan-in

Несколько generator'ов сливаются в один канал:

```go
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
    
    ch1 := numbersCtx(ctx, 5)
    ch2 := numbersCtx(ctx, 5)
    
    for v := range merge(ctx, ch1, ch2) {
        fmt.Println(v)
    }
}
```

**Что происходит:**

- Два generator'а отдают числа.
- `merge` сливает в один канал.
- Потребитель читает из одного канала.

### Generator + pipeline

Generator — первая стадия pipeline:

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 10)
    doubled := doubleStage(ctx, source)
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

**Что происходит:**

- `numbersCtx` — generator.
- `doubleStage` — стадия pipeline.
- `filterEvenStage` — стадия pipeline.
- Потребитель читает из последней стадии.

### Схема

```
┌──────────┐   ┌────────┐   ┌──────────┐
│Generator │──▶│ Double │──▶│  Filter  │──▶ Потребитель
└──────────┘   └────────┘   └──────────┘
```

### 💡 Практика: как комбинировать generator

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Generator — первая стадия pipeline.**
2. **`context` — через все стадии.**
3. **`close` — в каждой стадии.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Разбивай логику на маленькие стадии.**

**❌ НЕ ДЕЛАЙ:**

5. **Не смешивай generator и обработку в одной функции.**

---

## 8.7 Практика Go: генераторы данных

Разберём **четыре примера**.

### Пример 1: генератор Фибоначчи

```go
func fibonacci(ctx context.Context) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        a, b := 0, 1
        for {
            select {
            case out <- a:
                a, b = b, a+b
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
    
    for v := range fibonacci(ctx) {
        if v > 100 {
            cancel()
            break
        }
        fmt.Println(v)
    }
}
```

**Пример вывода:**

```
0
1
1
2
3
5
8
13
21
34
55
89
```

**Что видно:** generator отдаёт **бесконечную** последовательность. Потребитель прерывает через `cancel()`.

### Пример 2: генератор из API с пагинацией

```go
func paginatedAPI(ctx context.Context, baseURL string) <-chan User {
    out := make(chan User)
    
    go func() {
        defer close(out)
        page := 1
        for {
            users, err := fetchPage(ctx, baseURL, page)
            if err != nil {
                return
            }
            if len(users) == 0 {
                return
            }
            for _, u := range users {
                select {
                case out <- u:
                case <-ctx.Done():
                    return
                }
            }
            page++
        }
    }()
    
    return out
}
```

**Что демонстрирует:** generator сам управляет пагинацией. Потребитель читает пользователей одного за другим.

### Пример 3: генератор с ошибками

```go
type Result struct {
    Line string
    Err  error
}

func fileLines(ctx context.Context, path string) <-chan Result {
    out := make(chan Result)
    
    go func() {
        defer close(out)
        f, err := os.Open(path)
        if err != nil {
            out <- Result{Err: err}
            return
        }
        defer f.Close()
        
        scanner := bufio.NewScanner(f)
        for scanner.Scan() {
            select {
            case out <- Result{Line: scanner.Text()}:
            case <-ctx.Done():
                return
            }
        }
        if err := scanner.Err(); err != nil {
            select {
            case out <- Result{Err: err}:
            case <-ctx.Done():
            }
        }
    }()
    
    return out
}

func main() {
    ctx := context.Background()
    for r := range fileLines(ctx, "data.txt") {
        if r.Err != nil {
            log.Println("error:", r.Err)
            break
        }
        fmt.Println(r.Line)
    }
}
```

**Что демонстрирует:** ошибка передаётся через тот же канал. Потребитель её видит и завершается.

### Пример 4: generator + worker pool

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 100)
    
    // Fan-out: 5 воркеров
    const workers = 5
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for v := range source {
                // обработка
                time.Sleep(10 * time.Millisecond)
                fmt.Printf("Worker %d: %d\n", id, v)
            }
        }(i)
    }
    
    wg.Wait()
    fmt.Println("done")
}
```

**Что демонстрирует:** generator + fan-out. 5 воркеров читают из одного канала.

### 💡 Практика: как писать generator

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx` для отмены.**
2. **`defer close(out)` сразу после `make(chan)`.**
3. **`select` с `ctx.Done()` при отправке.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`Result{Value, Err}` для передачи ошибок.**
5. **Проверяй `scanner.Err()` для файлов.**

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `close`.**
7. **Не паникуй при ошибке — передавай через канал.**

---

## 8.8 Выводы и типичные ошибки

**Что мы узнали?**

Generator — это функция, которая отдаёт данные в канал. Возвращает `<-chan T`, запускает горутину, закрывает канал при завершении. Простейший generator не умеет останавливаться — для этого нужен `context`. Ошибки передаются через `Result{Value, Err}` или отдельный канал. Backpressure работает **автоматически** через небуферизованный канал. Generator — первая стадия pipeline; комбинируется с fan-out, fan-in, pipeline.

**Типичные ошибки:**

- ❌ **Забыть `defer close(out)`.** Потребитель зависнет.
- ❌ **Не использовать `context`.** Утечка при выходе потребителя.
- ❌ **Писать в канал без `select` с `ctx.Done()`.** Generator не остановится.
- ❌ **Возвращать `chan T` вместо `<-chan T`.** Потребитель может писать.
- ❌ **Игнорировать ошибки.** Теряются.
- ❌ **Паниковать при ошибке.** Роняет программу.
- ❌ **Огромный буфер.** Маскирует медленного потребителя.
- ❌ **Не проверять `scanner.Err()`.** Ошибки чтения теряются.
- ❌ **Смешивать generator и обработку.** Сложно тестировать.

---

## 8.9 Для быстрого повторения

- **Generator** — функция, возвращающая `<-chan T`.
- **Запускает горутину**, пишет в канал, закрывает по завершении.
- **`defer close(out)`** — сразу после `make(chan)`.
- **`context`** — для отмены. `select` с `ctx.Done()` при отправке.
- **Ошибки:** `Result{Value, Err}` или отдельный канал.
- **Backpressure** — автоматически через небуферизованный канал.
- **Буфер 10–100** — для небольшого запаса.
- **Generator — первая стадия pipeline.**
- **Fan-out** — N воркеров читают из одного канала generator.
- **Fan-in** — merge нескольких generator'ов.
- **Возвращай `<-chan T`**, не `chan T`.

---

## 8.10 Вопросы для самопроверки

1. Что такое generator? Какую задачу решает?
2. Как построить простейший generator?
3. Зачем нужен `context` в generator?
4. Как передавать ошибки из generator?
5. Как работает backpressure в generator?
6. Как комбинировать generator с fan-out?
7. Что будет, если потребитель перестанет читать из generator без `context`?

---

## 8.11 Ответы

### Ответ 1

**Generator** — функция, которая отдаёт данные в канал. Решает задачу **развязки** источника и обработчика: generator знает только «как читать», потребитель — «как обрабатывать».

### Ответ 2

```go
func numbers(n int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for i := 0; i < n; i++ {
            out <- i
        }
    }()
    return out
}
```

### Ответ 3

**`context`** позволяет остановить generator досрочно. Без него, если потребитель перестал читать, generator **зависнет** на отправке в канал. **Утечка горутины.**

### Ответ 4

**Два способа:**
1. `Result{Value, Err}` — каждый элемент несёт ошибку.
2. Отдельный канал `errCh` — данные в одном канале, ошибки в другом.

### Ответ 5

**Backpressure** работает **автоматически** через небуферизованный канал. Generator пишет в канал, блокируется, пока потребитель не прочитает. Generator работает со скоростью потребителя.

### Ответ 6

**Generator + fan-out:**

```go
source := numbersCtx(ctx, 100)
for i := 0; i < 5; i++ {
    go func() {
        for v := range source {  // читают из одного канала
            process(v)
        }
    }()
}
```

5 воркеров читают из **одного** канала generator.

### Ответ 7

**Без `context`:** generator **зависнет** на `out <- v`. Горутина не завершится. **Утечка.** Канал не закроется.

**С `context`:** generator видит `ctx.Done()` и завершается. `defer close(out)` закрывает канал.

---

## 8.12 Куда идти дальше?

Мы разобрали generator — источник данных. Теперь мы умеем читать данные из любого источника через канал.

Но что если нужно **ждать несколько источников** одновременно? Как слить результаты из нескольких generator'ов в один?

- **Как слить каналы?** → **Глава 9: Fan-in — слияние каналов.**
- **Как распределить работу?** → **Глава 10: Fan-out — распределение работы.**
- **Как построить конвейер?** → **Глава 11: Pipeline.**

---

## 8.13 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Generator** | Источник данных | `func() <-chan T` |
| **Горутина внутри** | Чтение данных | `go func() { ... }()` |
| **`defer close(out)`** | Закрытие канала | Сразу после `make(chan)` |
| **`<-chan T`** | Receive-only | Потребитель не может писать |
| **`context`** | Отмена | `select` с `ctx.Done()` |
| **`Result{Value, Err}`** | Передача ошибок | Или отдельный канал |
| **Backpressure** | Автоматически | Небуферизованный канал |
| **Буфер 10–100** | Небольшой запас | Сгладить пики |
| **Fan-out** | N воркеров | Читают из одного канала |
| **Fan-in** | Слияние | `merge` каналов |
| **Pipeline** | Первая стадия | Generator → стадии → потребитель |

🌱 **Ключевая идея:** Generator — функция, возвращающая `<-chan T`. Запускает горутину, пишет в канал, закрывает по завершении. `context` — для отмены; без него утечка при выходе потребителя. Ошибки передаются через `Result{Value, Err}` или отдельный канал. Backpressure работает автоматически через небуферизованный канал. Generator — первая стадия pipeline; комбинируется с fan-out, fan-in, pipeline. Не забывай `defer close(out)` и `select` с `ctx.Done()`.