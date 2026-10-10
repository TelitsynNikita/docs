# 📤 Глава 10: Fan-out — распределение работы

**Что вы узнаете:**
- Что такое fan-out и какую задачу он решает.
- Чем fan-out отличается от дублирования.
- Как построить простейший fan-out.
- Две реализации: N каналов и один общий.
- Как добавить отмену через `context`.
- Как избежать утечек и перекоса нагрузки.
- Как комбинировать fan-out с fan-in, generator, pipeline.

**После прочтения вы сможете:**
- Построить fan-out с нуля.
- Распределять работу между N воркерами.
- Выбирать между N каналами и одним общим.
- Правильно завершать воркеры.
- Комбинировать fan-out с другими паттернами.
- Понимать, где fan-out уместен, а где — нет.

---

## Содержание

- [10.0 Пролог: одна очередь, несколько обработчиков](#100-пролог-одна-очередь-несколько-обработчиков)
- [10.1 Что такое fan-out](#101-что-такое-fan-out)
- [10.2 Простейший fan-out](#102-простейший-fan-out)
- [10.3 Реализация A: N каналов](#103-реализация-a-n-каналов)
- [10.4 Реализация B: один общий канал](#104-реализация-b-один-общий-канал)
- [10.5 Fan-out с context](#105-fan-out-с-context)
- [10.6 Перекос нагрузки и залипание](#106-перекос-нагрузки-и-залипание)
- [10.7 В связке с другими паттернами](#107-в-связке-с-другими-паттернами)
- [10.8 Практика Go: распределение работы](#108-практика-go-распределение-работы)
- [10.9 Выводы и типичные ошибки](#109-выводы-и-типичные-ошибки)
- [10.10 Для быстрого повторения](#1010-для-быстрого-повторения)
- [10.11 Вопросы для самопроверки](#1011-вопросы-для-самопроверки)
- [10.12 Ответы](#1012-ответы)
- [10.13 Куда идти дальше?](#1013-куда-идти-дальше)
- [10.14 Чек-лист](#1014-чек-лист)

---

## 10.0 Пролог: одна очередь, несколько обработчиков

У нас есть сервис, который обрабатывает заказы. Заказы приходят в **одну очередь**, а обработка включает **внешние вызовы** — проверку склада, отправку уведомления. Каждый вызов по 50–100 мс.

Обрабатываем последовательно:

```go
func processAll(orders <-chan Order) {
    for order := range orders {
        process(order)
    }
}
```

Работает. Но пропускная способность — **10–20 заказов в секунду**. При этом CPU почти не загружен — мы ждём ответа от внешних сервисов. Хочется **обрабатывать параллельно**.

Наивное решение — запускать горутину на каждый заказ:

```go
func processAll(orders <-chan Order) {
    for order := range orders {
        go process(order)  // ← теперь нужно думать про ограничения
    }
}
```

И сразу появляются вопросы:

- **Сколько горутин запускать?** Если заказов 10 000 — 10 000 горутин? Внешние сервисы не выдержат.
- **Как ограничить параллелизм?** Нужен пул воркеров.
- **Что делать при отмене?** Как остановить всех?
- **Как собрать ошибки?**

Хочется **распределить** заказы между фиксированным числом воркеров. Именно эту задачу решает **fan-out**.

> **Мост к следующим главам:** fan-out — зеркало fan-in (Глава 9). Fan-out распределяет, fan-in сливает. Вместе они — основа для worker pool (Глава 13) и pipeline (Глава 11). Понимание fan-out даёт понимание, **как ограничивать параллелизм**.

---

## 10.1 Что такое fan-out

**Fan-out** — это распределение работы от **одного источника** к **нескольким** обработчикам.

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

### Ключевое: распределение, не дублирование

**Fan-out** — это **распределение**. Каждое значение идёт **одному** воркеру.

```
input: [1, 2, 3, 4, 5, 6]

worker 0: [1, 4]
worker 1: [2, 5]
worker 2: [3, 6]

→ каждое значение ТОЛЬКО ОДНОМУ воркеру
```

Это **не** дублирование. Дублирование — это **tee-канал** (Глава 19): каждое значение идёт **во все** каналы.

### Как работает распределение

Когда N воркеров читают из **одного** канала, runtime **автоматически** распределяет задачи:

```go
for i := 0; i < N; i++ {
    go func() {
        for task := range tasksCh {  // ← каждый воркер читает свою задачу
            process(task)
        }
    }()
}
```

**Что происходит:**

- Producer пишет `Task{1}` в канал.
- **Один** из воркеров (случайный) забирает его.
- Producer пишет `Task{2}`.
- **Другой** воркер забирает его.
- И так далее.

**Распределение автоматическое.** Не нужно вручную решать, какой воркер какую задачу возьмёт.

### Когда использовать fan-out

**1. Много задач, фиксированное число ресурсов.**

- 10 000 HTTP-запросов, 20 воркеров.
- 1000 заказов, 10 обработчиков.
- Много файлов, 8 потоков.

**2. Задачи с I/O.**

- Внешние API, БД, файлы.
- CPU-bound тоже работает, но чаще нужен `GOMAXPROCS` воркеров.

**3. Ограничение параллелизма.**

- Не 10 000 горутин одновременно, а 20.
- Контроль над file descriptors, памятью, внешними лимитами.

### Когда НЕ использовать fan-out

**1. Задачи зависимы.**

Если задача B требует результата A — fan-out не подойдёт. Нужен pipeline.

**2. Нужен строгий порядок.**

Fan-out не сохраняет порядок. Если порядок важен — добавляй сортировку.

**3. Задачи разнородные.**

Fan-out подходит для **однотипных** задач. Если типы разные — используй разные каналы.

### 💡 Практика: как думать о fan-out

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Для распределения работы — fan-out.**
2. **Один канал — N воркеров.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **N = GOMAXPROCS** для CPU-bound. **N = 10–100** для I/O-bound.

**❌ НЕ ДЕЛАЙ:**

4. **Не путай fan-out с tee.** Tee дублирует, fan-out распределяет.
5. **Не создавай горутину на каждую задачу** без ограничения.

---

## 10.2 Простейший fan-out

Начнём с самого простого — N воркеров читают из одного канала.

### Первая версия

```go
func fanOut(input <-chan int, n int) {
    var wg sync.WaitGroup
    
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for v := range input {
                process(v)
            }
        }()
    }
    
    wg.Wait()
}
```

**Что происходит:**

- Запускаем N горутин.
- Каждая читает из `input` в цикле.
- Обрабатывает значение.
- Завершается, когда `input` закрыт.

### Потребитель

```go
func main() {
    input := make(chan int, 100)
    
    // Запускаем fan-out в фоне
    done := make(chan struct{})
    go func() {
        defer close(done)
        fanOut(input, 5)
    }()
    
    // Producer
    for i := 0; i < 20; i++ {
        input <- i
    }
    close(input)
    
    <-done
    fmt.Println("done")
}
```

**Пример вывода:**

```
worker: process 3
worker: process 2
worker: process 1
worker: process 0
worker: process 4
...
```

**Что видно:** значения обрабатываются параллельно, порядок не гарантирован.

### Проблема: воркеры не возвращают результаты

`fanOut` **ничего не возвращает**. Если нужно обработать результаты — придётся добавлять канал результатов.

### Проблема: непонятно, кто обрабатывает

Внутри воркера мы не знаем **ID воркера**. Для отладки и метрик это полезно:

```go
func fanOut(input <-chan int, n int) {
    var wg sync.WaitGroup
    
    for i := 0; i < n; i++ {
        i := i
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for v := range input {
                process(id, v)
            }
        }(i)
    }
    
    wg.Wait()
}
```

**Что изменилось:** каждый воркер знает свой `id`.

### Схема

```
input ──► ┌──────────┐ ──► process(id, v)
          │ worker 0 │
          └──────────┘
          
input ──► ┌──────────┐ ──► process(id, v)
          │ worker 1 │
          └──────────┘
          
input ──► ┌──────────┐ ──► process(id, v)
          │ worker 2 │
          └──────────┘
```

### 💡 Практика: как писать простой fan-out

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **N воркеров читают из одного канала.**
2. **`defer wg.Done()`** в каждой горутине.
3. **`wg.Wait()`** в конце.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Передавай `id` воркера** — для отладки и метрик.

**❌ НЕ ДЕЛАЙ:**

5. **Не создавай канал на каждого воркера** — для fan-out нужен **один** вход.

---

## 10.3 Реализация A: N каналов

Иногда нужно **отделить** результаты разных воркеров. Тогда каждый воркер пишет в **свой** канал.

### Идея

```go
func fanOut(input <-chan int, n int) []<-chan int {
    outputs := make([]<-chan int, n)
    for i := 0; i < n; i++ {
        outputs[i] = worker(input)
    }
    return outputs
}

func worker(input <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range input {
            out <- v * 2
        }
    }()
    return out
}
```

**Что происходит:**

- N воркеров читают из **общего** `input`.
- Каждый пишет в **свой** канал.
- Возвращаем слайс каналов.

### Схема

```
          ┌──────────┐
      ┌──▶│ worker 0 │──▶ outputs[0]
      │   └──────────┘
      │   ┌──────────┐
input ────▶│ worker 1 │──▶ outputs[1]
      │   └──────────┘
      │   ┌──────────┐
      └──▶│ worker 2 │──▶ outputs[2]
          └──────────┘
```

### Потребитель

```go
func main() {
    input := make(chan int, 20)
    for i := 0; i < 20; i++ {
        input <- i
    }
    close(input)
    
    outputs := fanOut(input, 3)
    
    var wg sync.WaitGroup
    for i, out := range outputs {
        wg.Add(1)
        go func(id int, ch <-chan int) {
            defer wg.Done()
            for v := range ch {
                fmt.Printf("Worker %d: %d\n", id, v)
            }
        }(i, out)
    }
    wg.Wait()
}
```

**Пример вывода:**

```
Worker 1: 4
Worker 0: 2
Worker 2: 6
Worker 0: 8
Worker 1: 10
...
```

**Что видно:** каждый воркер имеет **свой** канал результатов.

### Когда использовать реализацию A

**1. Разная обработка результатов.**

Если результаты разных воркеров требуют **разной** обработки — A подходит.

**2. Нужно знать, какой воркер что сделал.**

`outputs[i]` — прямое соответствие «воркер i → его результаты».

**3. Отладка и тесты.**

Легче отлаживать, когда знаешь, откуда пришли данные.

### Проблема: N+1 каналов

С реализацией A получается **N + 1 каналов**: 1 вход + N выходов.

- **Больше памяти.** N каналов вместо 1.
- **Больше горутин.** N воркеров + N читателей.
- **Сложнее код.** Нужно обрабатывать N каналов.

**Решение:** реализация B (один общий канал).

### 💡 Практика: как использовать реализацию A

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Реализация A — если нужна гибкость.**
2. **Возвращай `[]<-chan T`.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Комбинируй с fan-in** для слияния результатов.

**❌ НЕ ДЕЛАЙ:**

4. **Не используй A, если результаты обрабатываются одинаково.** B проще.

---

## 10.4 Реализация B: один общий канал

Более простой и распространённый вариант — **один общий канал результатов**.

### Идея

```go
func fanOut(input <-chan int, n int) <-chan int {
    out := make(chan int)
    
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for v := range input {
                out <- v * 2
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

**Что происходит:**

- N воркеров читают из **общего** `input`.
- Все пишут в **один** `out`.
- `wg.Wait()` + `close(out)` в отдельной горутине.

### Схема

```
          ┌──────────┐
      ┌──▶│ worker 0 │──┐
      │   └──────────┘  │
      │   ┌──────────┐  │
input ────▶│ worker 1 │──┤──▶ out
      │   └──────────┘  │
      │   ┌──────────┐  │
      └──▶│ worker 2 │──┘
          └──────────┘
```

### Потребитель

```go
func main() {
    input := make(chan int, 20)
    for i := 0; i < 20; i++ {
        input <- i
    }
    close(input)
    
    for v := range fanOut(input, 3) {
        fmt.Println(v)
    }
}
```

**Пример вывода:**

```
2
0
6
4
10
8
...
```

**Что видно:** все результаты в **одном** канале. Порядок не гарантирован.

### Сравнение A и B

| Аспект | A: N каналов | B: один канал |
|:---|:---|:---|
| Каналов результатов | N | 1 |
| Горутин | N (воркеры) + N (читатели) | N (воркеры) + 1 (close) |
| Сложность | Средняя | Простая |
| Гибкость | Высокая | Низкая |
| Contention | Высокий | Низкий |

**Рекомендация:** для большинства случаев — **B**. A — только если нужна гибкость.

### 💡 Практика: как использовать реализацию B

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Реализация B (один канал) — для большинства случаев.**
2. **`wg.Wait()` + `close(out)` в отдельной горутине.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Буферизованный `out`** — для снижения contention.

**❌ НЕ ДЕЛАЙ:**

4. **Не используй A, если результаты одинаковые.**

---

## 10.5 Fan-out с context

Простейший fan-out не умеет останавливаться. Если потребитель перестал читать — воркеры зависнут.

### Проблема

```go
func main() {
    input := make(chan int)
    out := fanOut(input, 5)
    
    for v := range out {
        if v > 10 {
            break  // ← вышли
        }
        fmt.Println(v)
    }
    // Воркеры зависли на out <- v
}
```

**Что происходит:**

- `main` вышел из цикла.
- Воркеры продолжают писать в `out`.
- `out` небуферизованный — воркеры блокируются.
- **Утечка.**

### Решение: context

```go
func fanOutCtx(ctx context.Context, input <-chan int, n int) <-chan int {
    out := make(chan int)
    
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for v := range input {
                select {
                case out <- v * 2:
                case <-ctx.Done():
                    return
                }
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

**Что изменилось:** каждая отправка в `out` проходит через `select`. Если `ctx` отменён — воркер завершается.

### Потребитель

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    input := numbersCtx(ctx, 1_000_000)
    out := fanOutCtx(ctx, input, 5)
    
    count := 0
    for v := range out {
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

- После 10 значений — `cancel()`.
- Все воркеры видят `ctx.Done()` и завершаются.
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

func fanOutCtx(ctx context.Context, input <-chan int, n int) <-chan int {
    out := make(chan int)
    
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for v := range input {
                select {
                case out <- v * 2:
                case <-ctx.Done():
                    return
                }
            }
        }()
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
    
    input := numbersCtx(ctx, 1_000_000)
    out := fanOutCtx(ctx, input, 5)
    
    count := 0
    for v := range out {
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

### Схема

```
fanOutCtx(ctx, input, 3):

  worker 0:                    worker 1:                    worker 2:
    for v := range input {       for v := range input {       for v := range input {
      select {                     select {                     select {
      case out <- v:               case out <- v:               case out <- v:
      case <-ctx.Done():           case <-ctx.Done():           case <-ctx.Done():
        return                       return                       return
      }                            }                            }
    }                            }                            }
    
  Потребитель:
    cancel() ─────► ctx.Done() закрыт
                    │
                    ▼
              все воркеры завершаются
```

### 💡 Практика: как добавить context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx context.Context` — первый аргумент:**
   ```go
   func fanOutCtx(ctx context.Context, input <-chan T, n int) <-chan T
   ```

2. **`select` с `ctx.Done()` при отправке.**

3. **`defer cancel()` у потребителя.**

**❌ НЕ ДЕЛАЙ:**

4. **Не пиши в `out` без `select`** в долгоживущем fan-out.

---

## 10.6 Перекос нагрузки и залипание

Fan-out с одним каналом имеет **тонкость**: если один воркер медленный — он может «залипнуть» на одной задаче, и распределение станет неравномерным.

### Проблема

```go
input := make(chan Task)

// 5 воркеров
for i := 0; i < 5; i++ {
    go func() {
        for task := range input {
            process(task)  // ← разное время обработки
        }
    }()
}

// Producer
for _, task := range tasks {
    input <- task
}
```

**Что происходит:**

- Задачи распределяются **случайно**.
- Один воркер может получить **10 медленных** задач подряд.
- Другой — 10 быстрых.
- **Перекос нагрузки.**

### Как это исправить

**1. Разные воркеры для разных задач.**

Если задачи разной сложности — раздели их:

```go
fastCh := make(chan Task)
slowCh := make(chan Task)

// Быстрые задачи — 2 воркера
for i := 0; i < 2; i++ {
    go func() {
        for task := range fastCh {
            process(task)
        }
    }()
}

// Медленные задачи — 8 воркеров
for i := 0; i < 8; i++ {
    go func() {
        for task := range slowCh {
            process(task)
        }
    }()
}
```

**2. Динамическое число воркеров.**

Если нагрузка переменная — можно адаптировать число воркеров (см. Главу 13).

**3. Мониторинг.**

Метрики на каждого воркера покажут перекос:

```go
var processed [5]atomic.Int64

for i := 0; i < 5; i++ {
    i := i
    go func() {
        for v := range input {
            process(v)
            processed[i].Add(1)
        }
    }()
}
```

**Что искать:** если один воркер обработал в 10 раз больше — перекос.

### Залипание на закрытом канале

**❌ Плохо:**

```go
for {
    select {
    case v, ok := <-input:
        if !ok {
            return  // ← вышли
        }
        process(v)
    }
}
```

Это работает. Но если **случайно** забыть `ok`:

```go
for {
    select {
    case v := <-input:  // ← при закрытии v = zero
        process(v)      // ← обрабатываем zero
    }
}
```

**Что происходит:** при закрытии `input` `<-input` возвращает `zero` немедленно. Цикл крутится на 100% CPU.

**Правильно:**

```go
for v := range input {  // ← завершится при close
    process(v)
}
```

или:

```go
for {
    select {
    case v, ok := <-input:
        if !ok {
            return
        }
        process(v)
    case <-ctx.Done():
        return
    }
}
```

### 💡 Практика: как избежать перекоса и залипания

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`for v := range input`** — самое простое и надёжное.
2. **Проверяй `ok`**, если используешь `select`.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Мониторь processed по воркерам.**
4. **Разные каналы для разных типов задач.**

**❌ НЕ ДЕЛАЙ:**

5. **Не игнорируй `ok` при получении из канала.**
6. **Не полагайся на случайное распределение медленных задач.**

---

## 10.7 В связке с другими паттернами

Fan-out — центральный паттерн для распределения работы. Разберём связки.

### Fan-out + fan-in

**Классическая связка:** распределить работу, собрать результаты.

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
    
    // Fan-in
    merged := mergeCtx(ctx, outputs...)
    
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

**Что происходит:** это классический worker pool через fan-out + fan-in.

### Generator + fan-out

**Generator как источник, fan-out как распределитель:**

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 100)  // generator
    out := fanOutCtx(ctx, source, 5)  // fan-out
    
    for v := range out {
        fmt.Println(v)
    }
}
```

### Fan-out + pipeline

**Fan-out как стадия pipeline:**

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 100)
    
    // Стадия с fan-out
    doubled := fanOutCtx(ctx, source, 5)
    
    // Следующая стадия
    filtered := filterEvenStage(ctx, doubled)
    
    for v := range filtered {
        fmt.Println(v)
    }
}
```

### Fan-out + rate limiter

**Распределение с ограничением скорости:**

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    limiter := rate.NewLimiter(rate.Limit(100), 10)
    
    input := numbersCtx(ctx, 1_000_000)
    out := fanOutCtx(ctx, input, 10)
    
    for v := range out {
        if err := limiter.Wait(ctx); err != nil {
            break
        }
        fmt.Println(v)
    }
}
```

### Схема: worker pool через fan-out + fan-in

```
┌──────────┐   ┌──────────┐   ┌──────────┐
│Generator │──▶│ Fan-out  │──▶│  Fan-in  │──▶ Потребитель
│          │   │ (5 ворк.)│   │          │
└──────────┘   └──────────┘   └──────────┘
```

### 💡 Практика: как комбинировать fan-out

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Fan-out + fan-in — worker pool.**
2. **Generator + fan-out — распределение из источника.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Fan-out + pipeline — стадия распределения.**
4. **Fan-out + rate limiter — ограничение скорости.**

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай `context`.**

---

## 10.8 Практика Go: распределение работы

Разберём **три примера**.

### Пример 1: простой fan-out

```go
package main

import (
    "fmt"
    "sync"
)

func fanOut(input <-chan int, n int) {
    var wg sync.WaitGroup
    
    for i := 0; i < n; i++ {
        i := i
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for v := range input {
                fmt.Printf("Worker %d: %d\n", id, v)
            }
        }(i)
    }
    
    wg.Wait()
}

func main() {
    input := make(chan int, 20)
    for i := 0; i < 20; i++ {
        input <- i
    }
    close(input)
    
    fanOut(input, 3)
}
```

**Пример вывода:**

```
Worker 0: 0
Worker 1: 1
Worker 2: 2
Worker 0: 3
Worker 1: 4
...
```

### Пример 2: fan-out с context

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

func fanOutCtx(ctx context.Context, input <-chan int, n int) <-chan int {
    out := make(chan int)
    
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for v := range input {
                select {
                case out <- v * 2:
                case <-ctx.Done():
                    return
                }
            }
        }()
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
    
    input := numbersCtx(ctx, 1_000_000)
    out := fanOutCtx(ctx, input, 5)
    
    count := 0
    for v := range out {
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
    
    source := numbersCtx(ctx, 20)
    
    // Fan-out
    const workers = 5
    outputs := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        outputs[i] = worker(ctx, source)
    }
    
    // Fan-in
    merged := mergeCtx(ctx, outputs...)
    
    for v := range merged {
        fmt.Println(v)
    }
}
```

**Что демонстрирует:** классический worker pool через fan-out + fan-in.

### 💡 Практика: как писать fan-out

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **N воркеров читают из одного канала.**
2. **`defer wg.Done()`** в каждой горутине.
3. **`select` с `ctx.Done()` при отправке.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Fan-out + fan-in — worker pool.**
5. **Мониторь перекос нагрузки.**

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `close`.**
7. **Не создавай канал на каждого воркера без причины.**

---

## 10.9 Выводы и типичные ошибки

**Что мы узнали?**

Fan-out — распределение работы от одного источника к N обработчикам. **Две реализации:** A (N каналов) и B (один общий канал). Простейший fan-out — N воркеров читают из одного канала. `context` для отмены. Перекос нагрузки — реальная проблема при разных задачах. Fan-out + fan-in — классический worker pool. Generator + fan-out — распределение из источника. Fan-out + pipeline — стадия распределения.

**Типичные ошибки:**

- ❌ **Путать fan-out с tee.** Tee дублирует, fan-out распределяет.
- ❌ **Создавать канал на каждого воркера без причины.** Используй реализацию B.
- ❌ **Не использовать `context`.** Утечка при выходе потребителя.
- ❌ **Писать в `out` без `select` с `ctx.Done()`.**
- ❌ **Игнорировать `ok` при получении из канала.** Busy loop на закрытом.
- ❌ **Не мониторить перекос нагрузки.** Один воркер может залипнуть.
- ❌ **Использовать fan-out для разнородных задач.** Разделяй каналы.
- ❌ **Забыть `close(out)`.** Потребитель зависнет.
- ❌ **`close(out)` в горутине-воркере.** Паника.

---

## 10.10 Для быстрого повторения

- **Fan-out** — распределение работы от одного источника к N обработчикам.
- **Не дублирование.** Каждое значение идёт одному воркеру.
- **Реализация A:** N каналов результатов. Гибко, но больше горутин.
- **Реализация B:** один канал результатов. Проще и быстрее.
- **N воркеров читают из одного канала.**
- **`select` с `ctx.Done()`** — для отмены.
- **`for v := range input`** — самое простое.
- **Перекос нагрузки** — реальная проблема при разных задачах.
- **Fan-out + fan-in** — worker pool.
- **Generator + fan-out** — распределение из источника.
- **N = GOMAXPROCS** для CPU-bound. **N = 10–100** для I/O-bound.

---

## 10.11 Вопросы для самопроверки

1. Что такое fan-out? Какую задачу решает?
2. Чем fan-out отличается от tee?
3. Назови две реализации fan-out.
4. Какую выбрать — A или B?
5. Зачем `context` в fan-out?
6. Что такое перекос нагрузки?
7. Как комбинировать fan-out + fan-in?
8. Что будет, если потребитель перестанет читать из fan-out без `context`?

---

## 10.12 Ответы

### Ответ 1

**Fan-out** — распределение работы от одного источника к N обработчикам. Решает задачу: **много задач, фиксированное число ресурсов**. Один канал — N воркеров. Каждое значение идёт одному воркеру.

### Ответ 2

**Tee** дублирует **каждое** значение в **оба** канала. **Fan-out** распределяет — каждое значение идёт **одному** воркеру.

### Ответ 3

**Две реализации:**
- **A:** N каналов результатов (гибко, но больше горутин).
- **B:** один общий канал (проще, быстрее).

### Ответ 4

**B (один канал)** — для большинства случаев. **A (N каналов)** — если нужна гибкость (разная обработка результатов, отладка).

### Ответ 5

**`context`** позволяет остановить fan-out досрочно. Без него, если потребитель перестал читать, воркеры **зависнут** на `out <- v`. **Утечка.**

### Ответ 6

**Перекос нагрузки** — ситуация, когда один воркер получает больше работы, чем другие. Возникает при **разной сложности задач**. Решение: разные каналы для разных задач, мониторинг.

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

### Ответ 8

**Без `context`:** воркеры **зависнут** на `out <- v`. Горутины не завершатся. **Утечка.**

**С `context`:** воркеры видят `ctx.Done()` и завершаются. `defer close(out)` закрывает канал.

---

## 10.13 Куда идти дальше?

Мы разобрали fan-out — распределение работы. Теперь мы умеем распределять задачи между N обработчиками.

Но что если нужно **связать несколько стадий**? Одна стадия читает, другая обрабатывает, третья пишет?

- **Как построить конвейер?** → **Глава 11: Pipeline.**
- **Как ограничить параллелизм?** → **Глава 12: Semaphore.**
- **Как построить worker pool?** → **Глава 13: Worker pool.**

---

## 10.14 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Fan-out** | Распределение работы | Один вход, N воркеров |
| **Не дублирование** | Распределение | Tee дублирует, fan-out распределяет |
| **Реализация A** | N каналов | Гибко, но N+1 каналов |
| **Реализация B** | Один канал | Проще и быстрее |
| **N воркеров** | Читают из одного канала | Автоматическое распределение |
| **`for v := range input`** | Простой цикл | Завершится при close |
| **`select` с `ctx.Done()`** | Отмена | При отправке |
| **`wg.Wait()` + `close(out)`** | Закрытие | В отдельной горутине |
| **Перекос нагрузки** | Один воркер больше | Разные каналы, мониторинг |
| **Fan-out + fan-in** | Worker pool | Классика |
| **Generator + fan-out** | Источник + распределение | N воркеров |
| **N = GOMAXPROCS** | CPU-bound | Число ядер |
| **N = 10–100** | I/O-bound | Зависит от latency |

📤 **Ключевая идея:** Fan-out — распределение работы от одного источника к N обработчикам. **Не дублирование.** Две реализации: A (N каналов, гибко) и B (один канал, просто). N воркеров читают из одного канала — распределение автоматическое. `context` — для отмены; без него утечка при выходе потребителя. Перекос нагрузки — реальная проблема при разных задачах. Fan-out + fan-in — классический worker pool. Generator + fan-out — распределение из источника. Fan-out + pipeline — стадия распределения. N = GOMAXPROCS для CPU-bound, N = 10–100 для I/O-bound.