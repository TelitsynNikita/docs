# 🔁 Глава 15: Каналы — динамический select и приоритеты

**Что вы узнаете:**
- Почему `select` работает только с **фиксированным** набором case'ов.
- Что такое **`reflect.Select`** и как он решает проблему.
- Как построить **динамический fan-in** для переменного числа каналов.
- Как эмулировать **приоритетный select**.
- Что такое **starvation** в `select` и как его избежать.
- Как **`reflect.Select`** соотносится с обычным `select` по производительности.
- Когда **`reflect.Select`** — правильный выбор, а когда — нет.

**После прочтения вы сможете:**
- Использовать `reflect.Select` для динамического набора каналов.
- Построить приоритетный `select` через двойной `select`.
- Понимать, когда `reflect.Select` оправдан, а когда — нет.
- Избегать starvation при приоритетах.
- Писать fan-in с переменным числом каналов.

---

## Содержание

- [15.0 Пролог: каналы, число которых известно только в runtime](#150-пролог-каналы-число-которых-известно-только-в-runtime)
- [15.1 Ограничение select: фиксированный набор case'ов](#151-ограничение-select-фиксированный-набор-caseов)
- [15.2 reflect.Select: динамический select](#152-reflectselect-динамический-select)
- [15.3 Динамический fan-in](#153-динамический-fan-in)
- [15.4 Приоритетный select](#154-приоритетный-select)
- [15.5 Starvation в select и как его избежать](#155-starvation-в-select-и-как-его-избежать)
- [15.6 Производительность: reflect.Select vs N горутин](#156-производительность-reflectselect-vs-n-горутин)
- [15.7 Практика Go: динамический fan-in с метриками](#157-практика-go-динамический-fan-in-с-метриками)
- [15.8 Выводы и типичные ошибки](#158-выводы-и-типичные-ошибки)
- [15.9 Для быстрого повторения](#159-для-быстрого-повторения)
- [15.10 Вопросы для самопроверки](#1510-вопросы-для-самопроверки)
- [15.11 Ответы](#1511-ответы)
- [15.12 Куда идти дальше?](#1512-куда-идти-дальше)
- [15.13 Чек-лист](#1513-чек-лист)

---

## 15.0 Пролог: каналы, число которых известно только в runtime

Ты пишешь сервис, который агрегирует данные из **нескольких источников**. Источники приходят **динамически** — из конфига, из service discovery, из API.

```go
func mergeAll(sources []<-chan Event) <-chan Event {
    out := make(chan Event)
    // Как написать select, если число каналов неизвестно?
    // select { case v := <-sources[0]: ... case v := <-sources[1]: ... }
    // Но sources может быть 3, 10, 100...
    return out
}
```

❓ **Проблема:** `select` требует **фиксированного** набора case'ов. Нельзя написать «`select` по N каналам», если N известно только в runtime.

💡 **Решение:** **`reflect.Select`**. Он принимает **слайс** case'ов и динамически выбирает готовый.

```go
func mergeAll(sources []<-chan Event) <-chan Event {
    out := make(chan Event)

    go func() {
        defer close(out)

        cases := make([]reflect.SelectCase, len(sources))
        for i, ch := range sources {
            cases[i] = reflect.SelectCase{
                Dir:  reflect.SelectRecv,
                Chan: reflect.ValueOf(ch),
            }
        }

        for len(cases) > 0 {
            i, v, ok := reflect.Select(cases)
            if !ok {
                // Канал i закрыт — удаляем из cases
                cases = append(cases[:i], cases[i+1:]...)
                continue
            }
            out <- v.Interface().(Event)
        }
    }()

    return out
}
```

**Что делает:**

1. Создаёт `reflect.SelectCase` для каждого канала.
2. `reflect.Select` выбирает готовый case.
3. Если канал закрыт — удаляет его из `cases`.
4. Продолжает, пока есть активные каналы.

**Это динамический fan-in.** Работает для любого числа каналов.

> **Важный мост к будущим главам:** `reflect.Select` — инструмент для **динамических** сценариев. Он редко нужен, но когда нужен — незаменим. Глава 16 (Lock-free) — про другой способ работы с динамикой. Глава 19 (Паттерны) — про distributed locks и leader election.

---

## 15.1 Ограничение select: фиксированный набор case'ов

Прежде чем разбирать `reflect.Select`, поймём **ограничение** обычного `select`.

### Синтаксис select

```go
select {
case v := <-ch1:
    // получение из ch1
case ch2 <- 42:
    // отправка в ch2
case <-done:
    // закрытие done
default:
    // ни один case не готов
}
```

**Ключевое:** case'ы **фиксированы** на этапе компиляции. Компилятор генерирует код для **каждого** case'а.

### Что нельзя сделать

**1. Динамическое число каналов.**

```go
// ❌ Нельзя:
channels := []<-chan int{ch1, ch2, ch3}
select {
case v := <-channels[i]:  // i — переменная
    // ...
}
```

**Компилятор не знает**, сколько каналов и какие. `select` не работает с индексами.

**2. Переменные в case.**

```go
// ❌ Нельзя:
for i, ch := range channels {
    select {
    case v := <-ch:  // ch — переменная, но case'ов всё равно N
        // ...
    }
}
```

**Это N отдельных `select`**, а не один. Если `ch` не готов — `select` блокируется на **одном** канале, а не ждёт **все**.

**3. Case'ы, известные только в runtime.**

```go
// ❌ Нельзя:
func merge(channels []<-chan int) <-chan int {
    select {
    case v := <-channels[0]:  // а если channels пуст?
    case v := <-channels[1]:  // а если длина 3?
    // ...
    }
}
```

### Обходной путь: N горутин

**Для fan-in с фиксированным числом каналов** мы использовали N горутин (Глава 8):

```go
func merge(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
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

**Что делает:** для **каждого** канала — **отдельная** горутина. Каждая читает из своего канала и пишет в общий `out`.

**Плюсы:**

- Просто.
- Работает для любого числа каналов.

**Минусы:**

- **N + 1 горутин** — для 1000 каналов это 1001 горутина.
- **Память:** (N + 1) × 2.3 КБ = ~2.3 МБ для 1000 каналов.
- **Contention:** все горутины пишут в **один** `out`.

### Обходной путь: фиксированный select

**Для малого числа каналов** можно использовать фиксированный `select`:

```go
func merge3(ch1, ch2, ch3 <-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        for {
            select {
            case v, ok := <-ch1:
                if !ok {
                    ch1 = nil
                } else {
                    out <- v
                }
            case v, ok := <-ch2:
                if !ok {
                    ch2 = nil
                } else {
                    out <- v
                }
            case v, ok := <-ch3:
                if !ok {
                    ch3 = nil
                } else {
                    out <- v
                }
            }

            if ch1 == nil && ch2 == nil && ch3 == nil {
                return
            }
        }
    }()

    return out
}
```

**Что делает:** `ch1 = nil` отключает case (nil-канал блокируется навсегда, Глава 2).

**Плюсы:**

- **Одна горутина** вместо N.
- **Меньше contention.**

**Минусы:**

- **Фиксированное число каналов** — код генерируется для 3.
- **Много кода** для каждого N.

### Сравнение

| Подход | Горутин | Каналов | Код |
|:---|:---|:---|:---|
| N горутин | N + 1 | Любое | Простой |
| Фиксированный select | 1 | Фиксированное | Много |
| `reflect.Select` | 1 | Любое | Средний |

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| `select` (фиксированный) | ~100-200 нс | 0 |
| N горутин | ~100-200 нс на элемент | N × 2.3 КБ |
| `reflect.Select` | ~500-1000 нс | ~100 байт на case |

### 💡 Практика: как обходить ограничение select

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Для фиксированного числа каналов** — обычный `select`.
2. **Для многих каналов** — N горутин или `reflect.Select`.
3. **Для динамического числа** — `reflect.Select`.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`ch = nil`** для отключения case.

**❌ НЕ ДЕЛАЙ:**

5. **Не пиши `select` с индексами** — не работает.
6. **Не создавай N горутин** для тысяч каналов.

### Ключевые выводы подглавы 15.1

- **`select` фиксирован** на этапе компиляции.
- **N горутин** — обходной путь для многих каналов.
- **`ch = nil`** — отключение case.
- **`reflect.Select`** — для динамических каналов.

---

## 15.2 reflect.Select: динамический select

**`reflect.Select`** — динамический аналог `select`.

### API

```go
import "reflect"

type SelectCase struct {
    Dir  SelectDir   // SelectRecv, SelectSend, SelectDefault
    Chan Value       // канал
    Send Value       // значение для отправки (для SelectSend)
}

func Select(cases []SelectCase) (chosen int, recv Value, recvOK bool)
```

**Что делает:**

- Принимает **слайс** case'ов.
- Возвращает **индекс** выбранного case'а.
- Для `SelectRecv` — полученное значение и `ok`.
- Для `SelectSend` — `recv` = zero Value.
- Для `SelectDefault` — выбирается, если ни один не готов.

### Три типа case'ов

| Dir | Значение |
|:---|:---|
| `SelectRecv` | Получение из канала |
| `SelectSend` | Отправка в канал |
| `SelectDefault` | Default case |

### Пример: получение из N каналов

```go
func merge(channels []<-chan int) <-chan int {
    out := make(chan int)

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
                // Канал i закрыт — удаляем
                cases = append(cases[:i], cases[i+1:]...)
                continue
            }
            out <- int(v.Int())
        }
    }()

    return out
}
```

**Разберём по шагам.**

#### Шаг 1: подготовка cases

```go
cases := make([]reflect.SelectCase, len(channels))
for i, ch := range channels {
    cases[i] = reflect.SelectCase{
        Dir:  reflect.SelectRecv,
        Chan: reflect.ValueOf(ch),
    }
}
```

**Что делает:** создаёт case для каждого канала.

**`reflect.ValueOf(ch)`** — оборачивает канал в `reflect.Value`.

#### Шаг 2: цикл select

```go
for len(cases) > 0 {
    i, v, ok := reflect.Select(cases)
    // ...
}
```

**Что делает:** `reflect.Select` выбирает **готовый** case (случайно из готовых, как обычный `select`).

#### Шаг 3: обработка закрытия

```go
if !ok {
    cases = append(cases[:i], cases[i+1:]...)
    continue
}
```

**Что делает:** если канал `i` закрыт — удаляем его из `cases`.

**Почему `append(cases[:i], cases[i+1:]...)`:** удаляет элемент по индексу `i`.

#### Шаг 4: запись в out

```go
out <- int(v.Int())
```

**Что делает:** `v` — `reflect.Value`, `v.Int()` — `int64`, приводим к `int`.

### Пример: отправка

```go
func broadcast(value int, channels []chan<- int) {
    cases := make([]reflect.SelectCase, len(channels))
    for i, ch := range channels {
        cases[i] = reflect.SelectCase{
            Dir:  reflect.SelectSend,
            Chan: reflect.ValueOf(ch),
            Send: reflect.ValueOf(value),
        }
    }

    for len(cases) > 0 {
        i, _, ok := reflect.Select(cases)
        if !ok {
            // Канал i закрыт — отправка не удалась
            cases = append(cases[:i], cases[i+1:]...)
            continue
        }
        // Успешно отправили в канал i
        // Можно удалить, если не нужно отправлять ещё
        cases = append(cases[:i], cases[i+1:]...)
    }
}
```

**Что делает:** отправляет `value` в **каждый** канал из `channels`.

**`SelectSend`:** `reflect.Select` вернёт `i` того канала, в который **удалось** отправить.

### Пример: default

```go
func tryRecv(ch <-chan int) (int, bool) {
    cases := []reflect.SelectCase{
        {
            Dir:  reflect.SelectRecv,
            Chan: reflect.ValueOf(ch),
        },
        {
            Dir: reflect.SelectDefault,
        },
    }

    i, v, ok := reflect.Select(cases)
    if i == 1 {
        // default — канал не готов
        return 0, false
    }
    if !ok {
        // канал закрыт
        return 0, false
    }
    return int(v.Int()), true
}
```

**Что делает:** неблокирующая проверка канала.

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `reflect.ValueOf` | ~50-100 нс |
| `reflect.Select` (1 case) | ~500-1000 нс |
| `reflect.Select` (10 cases) | ~500-1000 нс |
| `reflect.Select` (100 cases) | ~500-1000 нс |

**Ключевое:** `reflect.Select` **не зависит** от числа case'ов линейно. Он использует ту же логику, что обычный `select` (случайный выбор из готовых).

**Но:** `reflect` **медленнее** обычного `select` в ~5-10 раз из-за динамической типизации.

### 💡 Практика: как использовать reflect.Select

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`reflect.SelectCase`** для каждого канала.
2. **Удаляй закрытые каналы** из `cases`.
3. **`reflect.ValueOf(ch)`** для обёртки канала.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Проверяй `ok`** после `reflect.Select`.
5. **Используй для динамических случаев** — N неизвестно.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Для фиксированных случаев** — обычный `select` быстрее.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `reflect.Select` в hot path.** Он медленнее.
8. **Не забывай удалять закрытые каналы.** Иначе бесконечный цикл.

### Ключевые выводы подглавы 15.2

- **`reflect.Select`** — динамический `select`.
- **`SelectCase`** — case для каждого канала.
- **`SelectRecv`, `SelectSend`, `SelectDefault`** — три типа.
- **Удаляй закрытые каналы** из `cases`.
- **Медленнее обычного `select`** в 5-10 раз.

---

## 15.3 Динамический fan-in

Разберём **динамический fan-in** — слияние переменного числа каналов.

### Проблема

**Дано:** слайс каналов `[]<-chan T` неизвестной длины.

**Нужно:** один канал, в который идут значения из всех.

### Решение через N горутин

```go
func fanInGoroutines(channels []<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
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

**Плюсы:**

- Просто.
- Работает для любого N.

**Минусы:**

- **N + 1 горутин.**
- **Contention** на `out`.

### Решение через reflect.Select

```go
func fanInReflect(channels []<-chan int) <-chan int {
    out := make(chan int)

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
            out <- int(v.Int())
        }
    }()

    return out
}
```

**Плюсы:**

- **Одна горутина.**
- **Меньше contention.**
- **Меньше памяти.**

**Минусы:**

- **`reflect` медленнее.**
- **Сложнее код.**

### Сравнение

| Подход | Горутин | Память | Time (per element) |
|:---|:---|:---|:---|
| N горутин | N + 1 | (N + 1) × 2.3 КБ | ~50-100 нс |
| `reflect.Select` | 1 | ~100 байт | ~500-1000 нс |

**Когда что:**

- **N < 100** — N горутин (проще).
- **N > 1000** — `reflect.Select` (меньше горутин).

### Fan-in с context

**Добавим отмену:**

```go
func fanInReflectCtx(ctx context.Context, channels []<-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        cases := make([]reflect.SelectCase, len(channels)+1)
        for i, ch := range channels {
            cases[i] = reflect.SelectCase{
                Dir:  reflect.SelectRecv,
                Chan: reflect.ValueOf(ch),
            }
        }
        // Добавляем ctx.Done() как case
        cases[len(channels)] = reflect.SelectCase{
            Dir:  reflect.SelectRecv,
            Chan: reflect.ValueOf(ctx.Done()),
        }

        ctxIdx := len(channels)

        for len(cases) > 1 {
            i, v, ok := reflect.Select(cases)
            if i == ctxIdx {
                // ctx.Done() — отмена
                return
            }
            if !ok {
                cases = append(cases[:i], cases[i+1:]...)
                if i < ctxIdx {
                    ctxIdx--
                }
                continue
            }
            out <- int(v.Int())
        }
    }()

    return out
}
```

**Что добавилось:**

- `ctx.Done()` как **дополнительный** case.
- При срабатывании — выход.
- Индекс `ctxIdx` корректируется при удалении каналов.

### Полный пример

```go
package main

import (
    "context"
    "fmt"
    "reflect"
    "time"
)

func main() {
    // Создаём N каналов
    const numChannels = 5
    channels := make([]<-chan int, numChannels)

    for i := 0; i < numChannels; i++ {
        ch := make(chan int, 10)
        channels[i] = ch
        go func(id int, c chan<- int) {
            defer close(c)
            for j := 0; j < 3; j++ {
                c <- id*10 + j
                time.Sleep(10 * time.Millisecond)
            }
        }(i, ch)
    }

    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()

    out := fanInReflectCtx(ctx, channels)

    for v := range out {
        fmt.Println(v)
    }
    fmt.Println("done")
}

func fanInReflectCtx(ctx context.Context, channels []<-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        cases := make([]reflect.SelectCase, len(channels)+1)
        for i, ch := range channels {
            cases[i] = reflect.SelectCase{
                Dir:  reflect.SelectRecv,
                Chan: reflect.ValueOf(ch),
            }
        }
        cases[len(channels)] = reflect.SelectCase{
            Dir:  reflect.SelectRecv,
            Chan: reflect.ValueOf(ctx.Done()),
        }
        ctxIdx := len(channels)

        for len(cases) > 1 {
            i, v, ok := reflect.Select(cases)
            if i == ctxIdx {
                return
            }
            if !ok {
                cases = append(cases[:i], cases[i+1:]...)
                if i < ctxIdx {
                    ctxIdx--
                }
                continue
            }
            out <- int(v.Int())
        }
    }()

    return out
}
```

**Пример вывода:**

```
0
10
20
30
40
1
11
21
...
done
```

**Что видно:** значения из всех 5 каналов, вперемешку.

### Аннотация сложности

| Подход | Горутин | Time (per element) |
|:---|:---|:---|
| N горутин | N + 1 | ~50-100 нс |
| `reflect.Select` | 1 | ~500-1000 нс |
| `reflect.Select` + ctx | 1 | ~500-1000 нс |

### 💡 Практика: как строить динамический fan-in

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`reflect.Select`** для N > 1000.
2. **N горутин** для N < 100.
3. **Удаляй закрытые каналы.**
4. **`ctx.Done()`** как дополнительный case.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Буферизованный `out`** для снижения contention.
6. **Логирование** — сколько каналов активно.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `reflect.Select` для 2-3 каналов** — обычный `select` проще.
8. **Не забывай удалять закрытые каналы.**

### Ключевые выводы подглавы 15.3

- **Динамический fan-in** — слияние N каналов.
- **N горутин** — для N < 100.
- **`reflect.Select`** — для N > 1000.
- **`ctx.Done()`** как дополнительный case.
- **Удаляй закрытые каналы.**

---

## 15.4 Приоритетный select

**Приоритетный select** — `select`, где один case имеет **приоритет**.

### Проблема

**Обычный `select` случаен.** Если несколько case'ов готовы, выбирается **случайный** (Глава 2).

```go
for {
    select {
    case v := <-highPriorityCh:
        processHigh(v)
    case v := <-lowPriorityCh:
        processLow(v)  // ← может голодать
    }
}
```

**Что происходит:** если `highPriorityCh` всегда готов, `lowPriorityCh` может **голодать**.

### Решение: двойной select

**Идея:** сначала проверить **приоритетный** канал, потом — все остальные.

```go
for {
    // 1. Проверяем приоритетный канал
    select {
    case v := <-highPriorityCh:
        processHigh(v)
        continue
    default:
    }

    // 2. Если приоритетный не готов — ждём любой
    select {
    case v := <-highPriorityCh:
        processHigh(v)
    case v := <-lowPriorityCh:
        processLow(v)
    }
}
```

**Что делает:**

1. **Первый `select`** с `default` — неблокирующая проверка приоритетного.
2. Если готов — обрабатываем.
3. Если нет — **второй `select`** ждёт любой.
4. Во втором `select` приоритетный всё равно участвует — но уже честно.

### Альтернатива: weighted random

**Если приоритет не строгий** — можно использовать вероятности:

```go
for {
    select {
    case v := <-highPriorityCh:
        processHigh(v)
    case v := <-lowPriorityCh:
        processLow(v)
    case <-time.After(100 * time.Millisecond):
        // Периодически обрабатываем low, даже если high готов
        select {
        case v := <-lowPriorityCh:
            processLow(v)
        default:
        }
    }
}
```

**Что делает:** раз в 100 мс даёт шанс `lowPriorityCh`.

### Паттерн: три уровня приоритета

```go
func processWithPriority(high, medium, low <-chan int) {
    for {
        // 1. High
        select {
        case v := <-high:
            processHigh(v)
            continue
        default:
        }

        // 2. Medium
        select {
        case v := <-high:
            processHigh(v)
            continue
        case v := <-medium:
            processMedium(v)
            continue
        default:
        }

        // 3. Low (блокирующий)
        select {
        case v := <-high:
            processHigh(v)
        case v := <-medium:
            processMedium(v)
        case v := <-low:
            processLow(v)
        }
    }
}
```

**Что делает:**

1. High — проверяется первым.
2. Medium — если high нет.
3. Low — если ни high, ни medium нет.
4. Но low **всё равно** может сработать, если high/medium не готовы.

### Starvation в приоритетном select

**Проблема:** если high **всегда** готов, low **никогда** не обработается.

**Решение:** **weighted** или **периодический** доступ.

```go
for i := 0; ; i++ {
    // Каждые 100 итераций — обрабатываем low
    if i%100 == 0 {
        select {
        case v := <-low:
            processLow(v)
            continue
        default:
        }
    }

    select {
    case v := <-high:
        processHigh(v)
    case v := <-low:
        processLow(v)
    }
}
```

### Аннотация сложности

| Подход | Time |
|:---|:---|
| Обычный select | ~100-200 нс |
| Двойной select | ~200-400 нс |
| Weighted random | ~200-400 нс |

### 💡 Практика: как строить приоритетный select

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Двойной select** — для строгого приоритета.
2. **Weighted random** — для мягкого.
3. **Периодический доступ** — против starvation.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики** — сколько из каждого канала.

**❌ НЕ ДЕЛАЙ:**

5. **Не полагайся на порядок в `select`** — он случаен.
6. **Не делай строгий приоритет без периодического low.**

### Ключевые выводы подглавы 15.4

- **`select` случаен** — нет приоритетов.
- **Двойной select** — для строгого приоритета.
- **Weighted random** — для мягкого.
- **Периодический доступ** — против starvation.

---

## 15.5 Starvation в select и как его избежать

**Starvation** — один канал **никогда** не обрабатывается.

### Причины

**1. Готовый канал всегда есть.**

```go
for {
    select {
    case v := <-fastCh:  // ← всегда готов
        process(v)
    case v := <-slowCh:  // ← голодает
        process(v)
    }
}
```

**Что происходит:** `fastCh` всегда готов, `select` его выбирает. `slowCh` голодает.

**2. Приоритет без периодического доступа.**

См. 15.4.

**3. Голодание в `reflect.Select`.**

```go
for len(cases) > 0 {
    i, v, ok := reflect.Select(cases)
    // ...
}
```

`reflect.Select` тоже **случаен** — starvation маловероятен, но возможен.

### Как обнаружить

**1. Метрики по каналам.**

```go
var metrics = map[string]atomic.Int64{
    "fast": {},
    "slow": {},
}

for {
    select {
    case v := <-fastCh:
        metrics["fast"].Add(1)
    case v := <-slowCh:
        metrics["slow"].Add(1)
    }
}
```

**Что искать:** один канал **резко** отстаёт.

**2. Логирование.**

```go
select {
case v := <-fastCh:
    log.Println("fast")
case v := <-slowCh:
    log.Println("slow")
}
```

**Что искать:** один канал не появляется в логах.

### Как избежать

**1. Weighted random.**

```go
for {
    select {
    case v := <-fastCh:
        process(v)
    case v := <-slowCh:
        process(v)
    }
    // ...
}
```

**Проблема:** Go `select` случаен, но **не weighted**. Оба канала равны.

**2. Round-robin.**

```go
channels := []<-chan int{fastCh, slowCh}
i := 0

for {
    select {
    case v := <-channels[i]:
        process(v)
    }
    i = (i + 1) % len(channels)
}
```

**Проблема:** если канал не готов — блокировка.

**3. Периодический доступ.**

```go
for i := 0; ; i++ {
    if i%10 == 0 {
        // Каждые 10 итераций — slow
        select {
        case v := <-slowCh:
            process(v)
            continue
        default:
        }
    }

    select {
    case v := <-fastCh:
        process(v)
    case v := <-slowCh:
        process(v)
    }
}
```

**Что делает:** раз в 10 итераций — попытка slow.

**4. Разные горутины.**

```go
go func() {
    for v := range fastCh {
        process(v)
    }
}()

go func() {
    for v := range slowCh {
        process(v)
    }
}()
```

**Что делает:** каждый канал обрабатывается **своей** горутиной. Starvation **невозможен**.

**Минус:** contention на общих ресурсах.

### Сравнение

| Подход | Starvation | Contention |
|:---|:---|:---|
| Обычный select | Возможен | Низкий |
| Двойной select | Возможен | Низкий |
| Периодический | Маловероятен | Низкий |
| Разные горутины | Невозможен | Высокий |

### Аннотация сложности

| Подход | Time |
|:---|:---|
| Обычный select | ~100-200 нс |
| Периодический | ~200-400 нс |
| Разные горутины | ~100-200 нс + contention |

### 💡 Практика: как избежать starvation

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики** по каналам.
2. **Периодический доступ** к голодающим.
3. **Разные горутины** для независимых каналов.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Алерты** на starvation.
5. **Round-robin** для справедливости.

**❌ НЕ ДЕЛАЙ:**

6. **Не полагайся на случайность `select`.**
7. **Не игнорируй отстающий канал.**

### Ключевые выводы подглавы 15.5

- **Starvation** — один канал голодает.
- **Причины:** готовый канал всегда есть, приоритет без периодичности.
- **Обнаружение:** метрики, логирование.
- **Решение:** периодический доступ, разные горутины.

---

## 15.6 Производительность: reflect.Select vs N горутин

Разберём **производительность**.

### Бенчмарк

```go
func BenchmarkFanInGoroutines(b *testing.B) {
    channels := make([]<-chan int, 100)
    for i := range channels {
        ch := make(chan int, b.N)
        channels[i] = ch
        for j := 0; j < b.N; j++ {
            ch <- j
        }
        close(ch)
    }

    b.ResetTimer()
    out := fanInGoroutines(channels)
    count := 0
    for range out {
        count++
    }
}

func BenchmarkFanInReflect(b *testing.B) {
    channels := make([]<-chan int, 100)
    for i := range channels {
        ch := make(chan int, b.N)
        channels[i] = ch
        for j := 0; j < b.N; j++ {
            ch <- j
        }
        close(ch)
    }

    b.ResetTimer()
    out := fanInReflect(channels)
    count := 0
    for range out {
        count++
    }
}
```

### Пример вывода

```
BenchmarkFanInGoroutines-8    1000    1234567 ns/op
BenchmarkFanInReflect-8       1000    5678901 ns/op
```

**Что видно:** `reflect.Select` **в 5 раз медленнее** для 100 каналов.

### Когда `reflect.Select` быстрее

**1. Очень много каналов (1000+).**

N горутин = 1001 горутина. Планировщик тратит время на переключение.

`reflect.Select` = 1 горутина. Планировщик не тратит время.

**2. Мало данных.**

Если данных мало, N горутин **создаются и уничтожаются** — overhead.

`reflect.Select` = 1 горутина на всё время.

**3. Много памяти.**

N горутин = N × 2.3 КБ. Для 10 000 каналов — 23 МБ.

`reflect.Select` = ~100 байт на case. Для 10 000 — 1 МБ.

### Когда N горутин быстрее

**1. Мало каналов (< 100).**

N горутин = 3-100. Overhead минимален.

`reflect.Select` = 500-1000 нс на элемент. Дорого.

**2. Много данных.**

N горутин обрабатывают **параллельно**. `reflect.Select` — **последовательно**.

**3. Простой код.**

N горутин — просто. `reflect.Select` — сложно.

### Сравнение

| N каналов | N горутин | reflect.Select |
|:---|:---|:---|
| 2-10 | Быстрее | Медленнее |
| 100 | Быстрее | Медленнее |
| 1000 | Сравнимо | Сравнимо |
| 10000 | Медленнее (память) | Быстрее |

### Аннотация сложности

| N | N горутин | reflect.Select |
|:---|:---|:---|
| 10 | ~50-100 нс | ~500-1000 нс |
| 100 | ~50-100 нс | ~500-1000 нс |
| 1000 | ~100-200 нс | ~500-1000 нс |
| 10000 | ~1-2 мкс + 23 МБ | ~500-1000 нс + 1 МБ |

### 💡 Практика: как выбирать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **N < 100** — N горутин.
2. **N > 1000** — `reflect.Select`.
3. **Бенчмаркай** для своего случая.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики** — сколько времени тратится.

**❌ НЕ ДЕЛАЙ:**

5. **Не используй `reflect.Select` для 2-3 каналов.**
6. **Не создавай 10 000 горутин.**

### Ключевые выводы подглавы 15.6

- **`reflect.Select`** медленнее для малых N.
- **N горутин** медленнее для больших N (память).
- **Порог:** ~100-1000 каналов.
- **Бенчмаркай** для своего случая.

---

## 15.7 Практика Go: динамический fan-in с метриками

Напишем **динамический fan-in** с метриками.

### Полный код

```go
package main

import (
    "context"
    "fmt"
    "reflect"
    "sync/atomic"
    "time"
)

type Metrics struct {
    Received atomic.Int64
    Active   atomic.Int64
    Closed   atomic.Int64
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // Создаём 10 каналов
    const numChannels = 10
    channels := make([]<-chan int, numChannels)

    for i := 0; i < numChannels; i++ {
        ch := make(chan int, 100)
        channels[i] = ch
        go producer(ctx, i, ch)
    }

    var metrics Metrics
    metrics.Active.Store(int64(numChannels))

    out := fanInReflect(ctx, channels, &metrics)

    for v := range out {
        _ = v
    }

    fmt.Printf("Received: %d\n", metrics.Received.Load())
    fmt.Printf("Active:   %d\n", metrics.Active.Load())
    fmt.Printf("Closed:   %d\n", metrics.Closed.Load())
}

func producer(ctx context.Context, id int, ch chan<- int) {
    defer close(ch)
    for i := 0; i < 10; i++ {
        select {
        case ch <- id*100 + i:
            time.Sleep(10 * time.Millisecond)
        case <-ctx.Done():
            return
        }
    }
}

func fanInReflect(ctx context.Context, channels []<-chan int, metrics *Metrics) <-chan int {
    out := make(chan int, 100)

    go func() {
        defer close(out)

        cases := make([]reflect.SelectCase, len(channels)+1)
        for i, ch := range channels {
            cases[i] = reflect.SelectCase{
                Dir:  reflect.SelectRecv,
                Chan: reflect.ValueOf(ch),
            }
        }
        ctxIdx := len(channels)
        cases[ctxIdx] = reflect.SelectCase{
            Dir:  reflect.SelectRecv,
            Chan: reflect.ValueOf(ctx.Done()),
        }

        for len(cases) > 1 {
            i, v, ok := reflect.Select(cases)
            if i == ctxIdx {
                return
            }
            if !ok {
                cases = append(cases[:i], cases[i+1:]...)
                if i < ctxIdx {
                    ctxIdx--
                }
                metrics.Active.Add(-1)
                metrics.Closed.Add(1)
                continue
            }
            metrics.Received.Add(1)
            out <- int(v.Int())
        }
    }()

    return out
}
```

### Пример вывода

```
Received: 100
Active:   0
Closed:   10
```

**Что видно:**

- 100 сообщений получено.
- 10 каналов закрыто.
- 0 активных.

### Аннотация сложности

| Метрика | Как измеряется |
|:---|:---|
| Received | `atomic.Int64` |
| Active | `atomic.Int64` |
| Closed | `atomic.Int64` |

### 💡 Практика: как измерять fan-in

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Received** — сколько получено.
2. **Active** — сколько каналов активно.
3. **Closed** — сколько закрыто.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Экспорт в Prometheus.**
5. **Алерты** на аномалии.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `Mutex` для метрик.**

### Ключевые выводы подглавы 15.7

- **Received, Active, Closed** — основные метрики.
- **`atomic.Int64`** — для счётчиков.
- **Экспорт в Prometheus** — для production.

---

## 15.8 Выводы и типичные ошибки

**Что мы узнали?**

`select` фиксирован на этапе компиляции. Для динамических каналов — `reflect.Select`. `reflect.SelectCase` — case для каждого канала. `SelectRecv`, `SelectSend`, `SelectDefault` — три типа. Динамический fan-in: N горутин для малых N, `reflect.Select` для больших. Приоритетный select через двойной `select`. Starvation возможен в приоритетном select. `reflect.Select` медленнее обычного `select` в 5-10 раз, но экономит память для больших N.

**Типичные ошибки:**

- ❌ **Пытаться использовать `select` с индексами.** Не работает.
- ❌ **Забыть удалить закрытый канал** из `cases`. Бесконечный цикл.
- ❌ **Использовать `reflect.Select` для 2-3 каналов.** Медленнее.
- ❌ **Создавать 10 000 горутин** для 10 000 каналов.
- ❌ **Полагать, что `select` даёт приоритет.** Он случаен.
- ❌ **Не использовать `ctx.Done()`** в `reflect.Select`.
- ❌ **Игнорировать starvation.**
- ❌ **Не бенчмаркать** `reflect.Select` vs N горутин.
- ❌ **Забыть про `reflect.ValueOf(ch)`.** Паника.
- ❌ **Не проверять `ok`** после `reflect.Select`.

---

## 15.9 Для быстрого повторения

- **`select` фиксирован** на этапе компиляции.
- **`reflect.Select`** — динамический `select`.
- **`SelectCase`** — case для каждого канала.
- **Три типа:** `SelectRecv`, `SelectSend`, `SelectDefault`.
- **Динамический fan-in:** N горутин для N < 100, `reflect.Select` для N > 1000.
- **Приоритетный select** — двойной `select`.
- **Starvation** — один канал голодает.
- **`reflect.Select`** медленнее в 5-10 раз, но экономит память.
- **Порог:** ~100-1000 каналов.
- **`ctx.Done()`** как дополнительный case.
- **Удаляй закрытые каналы** из `cases`.

---

## 15.10 Вопросы для самопроверки

1. Почему `select` не работает с динамическими каналами?
2. Что такое `reflect.Select`?
3. Какие три типа `SelectCase`?
4. Как удалить закрытый канал из `cases`?
5. Что такое динамический fan-in?
6. Когда N горутин, когда `reflect.Select`?
7. Что такое приоритетный select?
8. Как построить приоритетный select?
9. Что такое starvation в select?
10. Как избежать starvation?
11. Насколько `reflect.Select` медленнее обычного `select`?
12. Когда `reflect.Select` быстрее N горутин?
13. Как добавить `ctx.Done()` в `reflect.Select`?
14. Какие метрики для fan-in?
15. Почему `ch = nil` отключает case?
16. Что произойдёт, если не удалить закрытый канал?
17. Почему `reflect.ValueOf(ch)` нужен?
18. Что вернёт `reflect.Select` при закрытом канале?

---

## 15.11 Ответы

### Ответ 1

**`select` не работает с динамическими каналами**, потому что компилятор должен знать **все** case'ы на этапе компиляции. Индексы и переменные в case'ах не поддерживаются.

### Ответ 2

**`reflect.Select`** — динамический аналог `select`, который принимает **слайс** case'ов и возвращает индекс готового.

### Ответ 3

**Три типа:**
1. `SelectRecv` — получение.
2. `SelectSend` — отправка.
3. `SelectDefault` — default.

### Ответ 4

**Удалить закрытый канал:**

```go
cases = append(cases[:i], cases[i+1:]...)
```

### Ответ 5

**Динамический fan-in** — слияние переменного числа каналов в один.

### Ответ 6

- **N < 100** — N горутин.
- **N > 1000** — `reflect.Select`.

### Ответ 7

**Приоритетный select** — `select`, где один case имеет приоритет.

### Ответ 8

**Двойной select:**

```go
select {
case v := <-high:
    process(v)
    continue
default:
}

select {
case v := <-high:
    process(v)
case v := <-low:
    process(v)
}
```

### Ответ 9

**Starvation** — один канал никогда не обрабатывается.

### Ответ 10

**Избежать:**
- Периодический доступ к голодающим.
- Разные горутины для независимых каналов.
- Weighted random.

### Ответ 11

**`reflect.Select` медленнее в 5-10 раз** для малых N.

### Ответ 12

**`reflect.Select` быстрее** для N > 1000 из-за экономии памяти и горутин.

### Ответ 13

**Добавить `ctx.Done()`:**

```go
cases[len(channels)] = reflect.SelectCase{
    Dir:  reflect.SelectRecv,
    Chan: reflect.ValueOf(ctx.Done()),
}
```

### Ответ 14

**Метрики:** received, active, closed.

### Ответ 15

**`ch = nil`** отключает case, потому что приём из nil-канала блокируется навсегда, и `select` игнорирует этот case.

### Ответ 16

**Бесконечный цикл** — закрытый канал всегда готов (возвращает zero, false).

### Ответ 17

**`reflect.ValueOf(ch)`** нужен для обёртки канала в `reflect.Value`, потому что `SelectCase.Chan` — это `reflect.Value`.

### Ответ 18

**При закрытом канале** `reflect.Select` вернёт `i` этого канала, `recv` = zero Value, `recvOK` = `false`.

---

## 15.12 Куда идти дальше?

Мы разобрали `reflect.Select`, динамический fan-in, приоритетный select, starvation. Теперь мы умеем работать с динамическими каналами.

Но остаётся **следующая тема**: как уменьшить contention через **шардирование** и как писать **lock-free структуры**?

- **Как уменьшить contention?** Sharded locks, lock-free stack, lock-free queue. → **Глава 16: Sharded locks и lock-free структуры.**
- **Как переиспользовать объекты?** `sync.Pool`, аллокатор памяти. → **Глава 17: sync.Pool и аллокатор памяти.**
- **Как GC влияет на конкурентный код?** Tri-color, write barrier, STW паузы. → **Глава 18: GC и его влияние на конкурентный код.**

---

## 15.13 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **`select`** | Мультиплексирование | Фиксированный набор case'ов |
| **`reflect.Select`** | Динамический select | Слайс case'ов |
| **`SelectCase`** | Case | Dir, Chan, Send |
| **`SelectRecv`** | Получение | Dir для recv |
| **`SelectSend`** | Отправка | Dir для send |
| **`SelectDefault`** | Default | Dir для default |
| **Динамический fan-in** | N каналов в один | N горутин или reflect |
| **N горутин** | Для N < 100 | Просто, но много горутин |
| **`reflect.Select`** | Для N > 1000 | Одна горутина |
| **Приоритетный select** | Двойной select | Проверка high → любой |
| **Starvation** | Один канал голодает | Периодический доступ |
| **`ctx.Done()` в reflect** | Дополнительный case | Для отмены |
| **Удаление закрытых** | `append(cases[:i], cases[i+1:]...)` | Обязательно |
| **Метрики** | received, active, closed | `atomic.Int64` |

🔁 **Ключевая идея:** `select` фиксирован на этапе компиляции. Для динамических каналов — `reflect.Select`. `SelectCase` для каждого канала. Три типа: `SelectRecv`, `SelectSend`, `SelectDefault`. Динамический fan-in: N горутин для малых N, `reflect.Select` для больших. Приоритетный select через двойной `select`. Starvation возможен в приоритетном select. `reflect.Select` медленнее обычного `select` в 5-10 раз, но экономит память. Порог: ~100-1000 каналов. `ctx.Done()` как дополнительный case. **Удаляй закрытые каналы** из `cases`.