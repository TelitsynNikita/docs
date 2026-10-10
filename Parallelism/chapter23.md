# 🎯 Глава 23: Динамический select и приоритеты

**Что вы узнаете:**
- Почему `select` не поддерживает приоритеты.
- Что такое **starvation** в `select`.
- Как сделать **приоритетный select** через вложенные `select` с `default`.
- Как реализовать **взвешенные приоритеты**.
- Что такое **динамический select** через `reflect.Select`.
- Как добавить отмену через `context` в приоритетный select.
- Как комбинировать приоритеты с worker pool, pipeline, rate limiter.

**После прочтения вы сможете:**
- Понимать, почему `select` выбирает случайный case.
- Строить приоритетный select через `default`.
- Реализовывать взвешенные приоритеты.
- Использовать `reflect.Select` для динамических наборов каналов.
- Комбинировать приоритеты с другими паттернами.

---

## Содержание

- [23.0 Пролог: важные и не очень события](#230-пролог-важные-и-не-очень-события)
- [23.1 Почему select не поддерживает приоритеты](#231-почему-select-не-поддерживает-приоритеты)
- [23.2 Приоритетный select через default](#232-приоритетный-select-через-default)
- [23.3 Взвешенные приоритеты](#233-взвешенные-приоритеты)
- [23.4 Динамический select через reflect.Select](#234-динамический-select-через-reflectselect)
- [23.5 Приоритетный select с context](#235-приоритетный-select-с-context)
- [23.6 В связке с другими паттернами](#236-в-связке-с-другими-паттернами)
- [23.7 Практика Go: приоритетный worker pool](#237-практика-go-приоритетный-worker-pool)
- [23.8 Выводы и типичные ошибки](#238-выводы-и-типичные-ошибки)
- [23.9 Для быстрого повторения](#239-для-быстрого-повторения)
- [23.10 Вопросы для самопроверки](#2310-вопросы-для-самопроверки)
- [23.11 Ответы](#2311-ответы)
- [23.12 Куда идти дальше?](#2312-куда-идти-дальше)
- [23.13 Чек-лист](#2313-чек-лист)

---

## 23.0 Пролог: важные и не очень события

У нас есть сервис, который обрабатывает события из двух источников: **высокоприоритетные** (заказы клиентов) и **низкоприоритетные** (аналитика). События обоих типов приходят в отдельные каналы.

Пишем наивно через `select`:

```go
func worker(ctx context.Context, highCh, lowCh <-chan Event) {
    for {
        select {
        case <-ctx.Done():
            return
        case e := <-highCh:
            processHigh(e)
        case e := <-lowCh:
            processLow(e)
        }
    }
}
```

Работает. Но замечаем проблему: если **lowCh всегда готов** (низкоприоритетных событий много), то `select` может **часто** выбирать `lowCh`. Заказы клиентов обрабатываются **с задержкой**.

Хочется: если есть **высокоприоритетное** событие — обрабатывать его **первым**. Только если высокоприоритетных нет — обрабатывать низкоприоритетные.

Но `select` **не поддерживает приоритеты** — он выбирает **случайный** case из готовых. Значит, нужно как-то **эмулировать** приоритет.

Это и есть **динамический select и приоритеты** — набор техник для управления порядком обработки.

> **Мост к следующим главам:** приоритеты — важный паттерн для production-обработчиков. Он часто используется вместе с worker pool (Глава 13), pipeline (Глава 11) и rate limiter (Глава 14). Понимание приоритетов даёт понимание, **как управлять потоком событий**.

---

## 23.1 Почему select не поддерживает приоритеты

Разберём, **почему** `select` не поддерживает приоритеты.

### Случайный выбор

Как мы разбирали в Главе 3, `select` выбирает **случайный** case из готовых. Это **намеренное** поведение — для **справедливости**.

Если бы `select` всегда выбирал **первый** готовый case, то последние каналы **голодали** бы. Например:

```go
select {
case e := <-highCh:  // ← всегда выбирается первым
    processHigh(e)
case e := <-lowCh:   // ← никогда не выбирается
    processLow(e)
}
```

Если `highCh` всегда готов — `lowCh` **никогда** не обрабатывается. Это **starvation**.

### Как работает select

`select` в runtime:

1. **Перемешивает** порядок case'ов (случайный).
2. Проверяет готовность каждого.
3. Если **несколько** готовы — выбирает **первый в перемешанном порядке**.

**Результат:** порядок case'ов в коде **не влияет** на приоритет.

### Что такое starvation в select

**Starvation** — ситуация, когда один case **никогда не выбирается**, потому что всегда есть более приоритетный.

**Пример:** если `highCh` всегда готов, `lowCh` голодает.

### Что хочется

**Приоритетный select** — выбрать case с **наивысшим приоритетом** среди готовых.

```
Приоритеты:
  1. ctx.Done()  — отмена (наивысший)
  2. highCh      — высокий
  3. lowCh       — низкий
```

**Логика:**

- Если `ctx.Done()` готов — обработать его.
- Иначе если `highCh` готов — обработать его.
- Иначе если `lowCh` готов — обработать его.
- Иначе — ждать любого.

### 💡 Практика: как думать о приоритетах

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Помни: `select` выбирает случайный case.**
2. **Приоритеты — через вложенные `select` с `default`.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **`ctx.Done()` — наивысший приоритет.**
4. **Высокоприоритетные каналы — первыми.**

**❌ НЕ ДЕЛАЙ:**

5. **Не полагайся на порядок case'ов.**
6. **Не забывай про starvation.**

---

## 23.2 Приоритетный select через default

Основной способ — **вложенные `select` с `default`**.

### Идея

Проверяем каналы **в порядке приоритета**:

1. Проверяем **самый приоритетный** канал через `select` с `default`.
2. Если готов — обрабатываем.
3. Иначе проверяем **следующий**.
4. Если ни один не готов — блокируемся на **обычном** `select`.

### Реализация

```go
func worker(ctx context.Context, highCh, lowCh <-chan Event) {
    for {
        // 1. Проверяем ctx.Done() — наивысший приоритет
        select {
        case <-ctx.Done():
            return
        default:
        }
        
        // 2. Проверяем highCh — высокий приоритет
        select {
        case e := <-highCh:
            processHigh(e)
            continue
        default:
        }
        
        // 3. Проверяем lowCh — низкий приоритет
        select {
        case e := <-lowCh:
            processLow(e)
            continue
        default:
        }
        
        // 4. Ни один не готов — блокируемся на обычном select
        select {
        case <-ctx.Done():
            return
        case e := <-highCh:
            processHigh(e)
        case e := <-lowCh:
            processLow(e)
        }
    }
}
```

**Что происходит:**

1. **`ctx.Done()`** — проверяем без блокировки.
2. **`highCh`** — проверяем без блокировки.
3. **`lowCh`** — проверяем без блокировки.
4. **Если ни один не готов** — блокируемся на обычном `select`.

### Схема

```
for {
  ├── ctx.Done() готов?
  │   └── да → return
  │
  ├── highCh готов?
  │   └── да → processHigh → continue
  │
  ├── lowCh готов?
  │   └── да → processLow → continue
  │
  └── ни один не готов
      └── блокируемся на select с ctx.Done(), highCh, lowCh
}
```

### Полный пример

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func worker(ctx context.Context, highCh, lowCh <-chan string) {
    for {
        // 1. ctx.Done() — наивысший
        select {
        case <-ctx.Done():
            return
        default:
        }
        
        // 2. highCh — высокий
        select {
        case e := <-highCh:
            fmt.Printf("[HIGH] %s\n", e)
            continue
        default:
        }
        
        // 3. lowCh — низкий
        select {
        case e := <-lowCh:
            fmt.Printf("[LOW]  %s\n", e)
            continue
        default:
        }
        
        // 4. Ни один не готов — блокируемся
        select {
        case <-ctx.Done():
            return
        case e := <-highCh:
            fmt.Printf("[HIGH] %s\n", e)
        case e := <-lowCh:
            fmt.Printf("[LOW]  %s\n", e)
        }
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()
    
    highCh := make(chan string, 100)
    lowCh := make(chan string, 100)
    
    // Быстро заполняем оба канала
    for i := 0; i < 20; i++ {
        highCh <- fmt.Sprintf("high-%d", i)
        lowCh <- fmt.Sprintf("low-%d", i)
    }
    
    worker(ctx, highCh, lowCh)
}
```

**Пример вывода:**

```
[HIGH] high-0
[HIGH] high-1
...
[HIGH] high-19
[LOW]  low-0
[LOW]  low-1
...
[LOW]  low-19
```

**Что видно:** сначала обрабатываются **все высокоприоритетные**, потом низкоприоритетные. Приоритет работает.

### Проблема: busy loop

**Что если `ctx.Done()` и каналы не готовы?**

- Все три `select` с `default` — идут в `default`.
- Мы попадаем в **блокирующий** `select`.
- OK.

**Но:** если **highCh периодически готов**, а lowCh всегда готов, мы **быстро** проверяем highCh в цикле. При высокой частоте это **busy loop**.

**Решение:** добавить `runtime.Gosched()` или проверить через `time.Sleep` в крайнем случае. Но обычно не нужно — `select` в конце блокируется.

### 💡 Практика: как писать приоритетный select

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Проверяй `ctx.Done()` первым.**
2. **Используй `default` для неблокирующей проверки.**
3. **Блокируйся на обычном `select` в конце.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`continue` после обработки — чтобы снова проверить приоритеты.**

**❌ НЕ ДЕЛАЙ:**

5. **Не полагайся на порядок case'ов в блокирующем `select`.**

---

## 23.3 Взвешенные приоритеты

Иногда нужны **взвешенные** приоритеты: не «строго high, потом low», а «обрабатывать 5 high, потом 1 low».

### Идея

- Считаем число обработанных высокоприоритетных.
- После N — обрабатываем один низкоприоритетный.
- Возвращаемся к high.

### Реализация

```go
func worker(ctx context.Context, highCh, lowCh <-chan Event, highWeight int) {
    highCount := 0
    
    for {
        // Проверяем ctx.Done()
        select {
        case <-ctx.Done():
            return
        default:
        }
        
        // Если обработали highWeight высокоприоритетных — обрабатываем low
        if highCount >= highWeight {
            select {
            case e := <-lowCh:
                processLow(e)
                highCount = 0
                continue
            default:
            }
        }
        
        // Обрабатываем high
        select {
        case e := <-highCh:
            processHigh(e)
            highCount++
            continue
        default:
        }
        
        // Если high пуст — обрабатываем low
        select {
        case e := <-lowCh:
            processLow(e)
            continue
        default:
        }
        
        // Блокируемся
        select {
        case <-ctx.Done():
            return
        case e := <-highCh:
            processHigh(e)
            highCount++
        case e := <-lowCh:
            processLow(e)
        }
    }
}
```

**Что происходит:**

- `highCount` — сколько high обработано подряд.
- При достижении `highWeight` — обрабатываем **один** low.
- Сбрасываем счётчик.

### Схема

```
highWeight = 5:

  1: HIGH
  2: HIGH
  3: HIGH
  4: HIGH
  5: HIGH
  6: LOW    ← после 5 high
  7: HIGH
  8: HIGH
  ...
```

### Пример: взвешенные приоритеты 3:1

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func worker(ctx context.Context, highCh, lowCh <-chan string, highWeight int) {
    highCount := 0
    
    for {
        select {
        case <-ctx.Done():
            return
        default:
        }
        
        if highCount >= highWeight {
            select {
            case e := <-lowCh:
                fmt.Printf("[LOW]  %s\n", e)
                highCount = 0
                continue
            default:
            }
        }
        
        select {
        case e := <-highCh:
            fmt.Printf("[HIGH] %s\n", e)
            highCount++
            continue
        default:
        }
        
        select {
        case e := <-lowCh:
            fmt.Printf("[LOW]  %s\n", e)
            continue
        default:
        }
        
        select {
        case <-ctx.Done():
            return
        case e := <-highCh:
            fmt.Printf("[HIGH] %s\n", e)
            highCount++
        case e := <-lowCh:
            fmt.Printf("[LOW]  %s\n", e)
        }
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()
    
    highCh := make(chan string, 100)
    lowCh := make(chan string, 100)
    
    for i := 0; i < 20; i++ {
        highCh <- fmt.Sprintf("high-%d", i)
        lowCh <- fmt.Sprintf("low-%d", i)
    }
    
    worker(ctx, highCh, lowCh, 3)
}
```

**Пример вывода:**

```
[HIGH] high-0
[HIGH] high-1
[HIGH] high-2
[LOW]  low-0
[HIGH] high-3
[HIGH] high-4
[HIGH] high-5
[LOW]  low-1
...
```

**Что видно:** 3 высокоприоритетных, 1 низкоприоритетный. Взвешенные приоритеты работают.

### 💡 Практика: как делать взвешенные приоритеты

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Счётчик обработанных high.**
2. **Обработка low после `highWeight`.**
3. **Сброс счётчика.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`highWeight` = 3–10.**
5. **Метрики по обоим типам.**

**❌ НЕ ДЕЛАЙ:**

6. **Не делай `highWeight` слишком большим** — low голодает.

---

## 23.4 Динамический select через reflect.Select

Иногда набор каналов **не известен** на этапе компиляции. Например, N каналов из конфигурации.

### Идея

`reflect.Select` позволяет выбирать из **динамического** набора каналов.

### API

```go
import "reflect"

cases := make([]reflect.SelectCase, len(channels))
for i, ch := range channels {
    cases[i] = reflect.SelectCase{
        Dir:  reflect.SelectRecv,
        Chan: reflect.ValueOf(ch),
    }
}

// Выбрать из cases
idx, value, ok := reflect.Select(cases)
```

**Что делает:**

- `reflect.Select` возвращает **индекс** выбранного case.
- Значение в `value`.
- `ok` — канал открыт.

### Пример: динамический набор каналов

```go
package main

import (
    "context"
    "fmt"
    "reflect"
    "time"
)

func worker(ctx context.Context, channels []<-chan string) {
    cases := make([]reflect.SelectCase, len(channels)+1)
    cases[0] = reflect.SelectCase{
        Dir:  reflect.SelectRecv,
        Chan: reflect.ValueOf(ctx.Done()),
    }
    for i, ch := range channels {
        cases[i+1] = reflect.SelectCase{
            Dir:  reflect.SelectRecv,
            Chan: reflect.ValueOf(ch),
        }
    }
    
    for {
        idx, value, ok := reflect.Select(cases)
        if idx == 0 {
            return  // ctx.Done()
        }
        if !ok {
            // Канал закрыт — удаляем из cases
            cases = append(cases[:idx], cases[idx+1:]...)
            if len(cases) == 1 {  // только ctx.Done()
                return
            }
            continue
        }
        fmt.Printf("[ch-%d] %s\n", idx-1, value.String())
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
    defer cancel()
    
    // 3 канала
    ch1 := make(chan string, 10)
    ch2 := make(chan string, 10)
    ch3 := make(chan string, 10)
    
    // Заполняем
    for i := 0; i < 5; i++ {
        ch1 <- fmt.Sprintf("a%d", i)
        ch2 <- fmt.Sprintf("b%d", i)
        ch3 <- fmt.Sprintf("c%d", i)
    }
    
    worker(ctx, []<-chan string{ch1, ch2, ch3})
}
```

**Пример вывода:**

```
[ch-0] a0
[ch-2] c0
[ch-1] b0
[ch-2] c1
...
```

**Что видно:** `select` работает с **динамическим** набором каналов.

### Проблема: медленно

`reflect.Select` **медленнее** обычного `select` в **10 раз**. Overhead от reflection.

**Когда использовать:**

- **N > 100** каналов.
- **Набор каналов неизвестен** на этапе компиляции.
- **Динамическое добавление/удаление.**

**Когда НЕ использовать:**

- **N маленькое** (2–10).
- **Набор известен** — используй обычный `select`.

### Приоритеты с reflect.Select

**Приоритеты через порядок** case'ов:

```go
// cases[0] — наивысший приоритет
// cases[1] — следующий
// ...

for {
    idx, value, ok := reflect.Select(cases)
    // idx: 0 — самый приоритетный из готовых
    // ...
}
```

**Стоп:** `reflect.Select` **тоже случайный**. Он **не** поддерживает приоритеты.

**Решение:** приоритетный select через несколько `reflect.Select` с `default`:

```go
func trySelect(cases []reflect.SelectCase) (int, reflect.Value, bool, bool) {
    // Добавляем default
    withDefault := append(cases, reflect.SelectCase{
        Dir: reflect.SelectDefault,
    })
    idx, value, ok := reflect.Select(withDefault)
    if idx == len(cases) {
        return 0, reflect.Value{}, false, false  // default
    }
    return idx, value, ok, true
}
```

**Что даёт:** неблокирующая проверка.

### 💡 Практика: как использовать reflect.Select

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`reflect.Select` — для динамических наборов.**
2. **N > 100 каналов — только здесь.**
3. **Удаляй закрытые каналы.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Приоритеты через несколько `reflect.Select` с default.**

**❌ НЕ ДЕЛАЙ:**

5. **Не используй `reflect.Select` для 2–10 каналов.**
6. **Не забывай про overhead.**

---

## 23.5 Приоритетный select с context

`context` — критичен для приоритетного select. `ctx.Done()` — **наивысший** приоритет.

### Идея

Всегда проверяем `ctx.Done()` **первым**:

```go
select {
case <-ctx.Done():
    return
default:
}
```

**Что даёт:** при отмене — **немедленно** выходим.

### Полная реализация

```go
func worker(ctx context.Context, highCh, lowCh <-chan Event) {
    for {
        // 1. Проверяем ctx.Done() — наивысший приоритет
        if ctx.Err() != nil {
            return
        }
        
        // 2. Проверяем highCh
        select {
        case e := <-highCh:
            processHigh(e)
            continue
        default:
        }
        
        // 3. Проверяем lowCh
        select {
        case e := <-lowCh:
            processLow(e)
            continue
        default:
        }
        
        // 4. Блокируемся
        select {
        case <-ctx.Done():
            return
        case e := <-highCh:
            processHigh(e)
        case e := <-lowCh:
            processLow(e)
        }
    }
}
```

**Ключевое:** в блокирующем `select` `ctx.Done()` участвует, но не с приоритетом. Если `highCh` и `ctx.Done()` готовы **одновременно** — выберется **случайный**.

**Как сделать `ctx.Done()` приоритетным?** Проверять **до** блокирующего `select`:

```go
if ctx.Err() != nil {
    return
}
```

**Что даёт:** при отмене — выходим **немедленно**.

### Схема

```
for {
  ├── ctx.Err() != nil? → return
  │
  ├── highCh готов? → processHigh → continue
  │
  ├── lowCh готов? → processLow → continue
  │
  └── блокируемся
      ├── ctx.Done() → return
      ├── highCh → processHigh
      └── lowCh → processLow
}
```

### 💡 Практика: как добавить context в приоритетный select

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx.Err()` — в начале каждой итерации.**
2. **`ctx.Done()` — в блокирующем `select`.**
3. **`ctx.Done()` — наивысший приоритет.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Graceful shutdown.**

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай про `ctx.Done()`.**
6. **Не игнорируй `ctx.Err()`.**

---

## 23.6 В связке с другими паттернами

Приоритеты редко используются **в одиночку**. Разберём связки.

### Приоритеты + worker pool

**Worker pool с приоритетами:**

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    highCh := make(chan Task, 100)
    lowCh := make(chan Task, 100)
    
    // 5 воркеров
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            priorityWorker(ctx, highCh, lowCh)
        }()
    }
    
    // Producer: high приоритет
    go func() {
        for _, task := range highTasks {
            highCh <- task
        }
    }()
    
    // Producer: low приоритет
    go func() {
        for _, task := range lowTasks {
            lowCh <- task
        }
    }()
    
    wg.Wait()
}

func priorityWorker(ctx context.Context, highCh, lowCh <-chan Task) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
        }
        
        select {
        case t := <-highCh:
            process(t)
            continue
        default:
        }
        
        select {
        case t := <-lowCh:
            process(t)
            continue
        default:
        }
        
        select {
        case <-ctx.Done():
            return
        case t := <-highCh:
            process(t)
        case t := <-lowCh:
            process(t)
        }
    }
}
```

### Приоритеты + pipeline

**Приоритетный select как стадия pipeline:**

```go
func mergePriority(ctx context.Context, highCh, lowCh <-chan Event) <-chan Event {
    out := make(chan Event)
    
    go func() {
        defer close(out)
        
        for {
            select {
            case <-ctx.Done():
                return
            default:
            }
            
            select {
            case e := <-highCh:
                select {
                case out <- e:
                case <-ctx.Done():
                    return
                }
                continue
            default:
            }
            
            select {
            case e := <-lowCh:
                select {
                case out <- e:
                case <-ctx.Done():
                    return
                }
                continue
            default:
            }
            
            select {
            case <-ctx.Done():
                return
            case e := <-highCh:
                select {
                case out <- e:
                case <-ctx.Done():
                    return
                }
            case e := <-lowCh:
                select {
                case out <- e:
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    
    return out
}
```

### Приоритеты + rate limiter

**Разные rate limiters для разных приоритетов:**

```go
func priorityWorker(ctx context.Context, highCh, lowCh <-chan Event, highLimiter, lowLimiter *rate.Limiter) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
        }
        
        select {
        case e := <-highCh:
            if err := highLimiter.Wait(ctx); err != nil {
                return
            }
            processHigh(e)
            continue
        default:
        }
        
        select {
        case e := <-lowCh:
            if err := lowLimiter.Wait(ctx); err != nil {
                return
            }
            processLow(e)
            continue
        default:
        }
        
        select {
        case <-ctx.Done():
            return
        case e := <-highCh:
            if err := highLimiter.Wait(ctx); err != nil {
                return
            }
            processHigh(e)
        case e := <-lowCh:
            if err := lowLimiter.Wait(ctx); err != nil {
                return
            }
            processLow(e)
        }
    }
}
```

**Что даёт:** разные лимиты для разных приоритетов.

### Схема

```
highCh ──┐
         ├──► priorityWorker ──► process
lowCh ───┘
         │
         └── ctx.Done() — наивысший приоритет
```

### 💡 Практика: как комбинировать приоритеты

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Приоритеты + worker pool** — для критичных задач.
2. **Приоритеты + pipeline** — стадия с приоритетами.
3. **Приоритеты + rate limiter** — разные лимиты.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики по приоритетам.**

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай про `ctx.Done()`.**

---

## 23.7 Практика Go: приоритетный worker pool

Разберём **полный приоритетный worker pool**.

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
    ID       int
    Priority string
}

type Metrics struct {
    HighProcessed atomic.Int64
    LowProcessed  atomic.Int64
    HighStarved   atomic.Int64
    LowStarved    atomic.Int64
}

func priorityWorker(ctx context.Context, highCh, lowCh <-chan Task, highWeight int, metrics *Metrics) {
    highCount := 0
    
    for {
        if ctx.Err() != nil {
            return
        }
        
        // Взвешенный приоритет
        if highCount >= highWeight {
            select {
            case t := <-lowCh:
                fmt.Printf("[LOW]  %d\n", t.ID)
                metrics.LowProcessed.Add(1)
                highCount = 0
                continue
            default:
            }
        }
        
        select {
        case t := <-highCh:
            fmt.Printf("[HIGH] %d\n", t.ID)
            metrics.HighProcessed.Add(1)
            highCount++
            continue
        default:
        }
        
        select {
        case t := <-lowCh:
            fmt.Printf("[LOW]  %d\n", t.ID)
            metrics.LowProcessed.Add(1)
            continue
        default:
        }
        
        select {
        case <-ctx.Done():
            return
        case t := <-highCh:
            fmt.Printf("[HIGH] %d\n", t.ID)
            metrics.HighProcessed.Add(1)
            highCount++
        case t := <-lowCh:
            fmt.Printf("[LOW]  %d\n", t.ID)
            metrics.LowProcessed.Add(1)
        }
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()
    
    highCh := make(chan Task, 1000)
    lowCh := make(chan Task, 1000)
    
    metrics := &Metrics{}
    
    // 5 приоритетных воркеров
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            priorityWorker(ctx, highCh, lowCh, 3, metrics)
        }()
    }
    
    // Producer: 1000 high и 1000 low
    go func() {
        for i := 0; i < 1000; i++ {
            highCh <- Task{ID: i, Priority: "high"}
        }
    }()
    go func() {
        for i := 0; i < 1000; i++ {
            lowCh <- Task{ID: i, Priority: "low"}
        }
    }()
    
    time.Sleep(500 * time.Millisecond)
    cancel()
    wg.Wait()
    
    fmt.Println("\n=== Metrics ===")
    fmt.Printf("High processed: %d\n", metrics.HighProcessed.Load())
    fmt.Printf("Low processed:  %d\n", metrics.LowProcessed.Load())
}
```

**Пример вывода:**

```
[HIGH] 0
[HIGH] 1
[HIGH] 2
[LOW]  0
[HIGH] 3
[HIGH] 4
[HIGH] 5
[LOW]  1
...
=== Metrics ===
High processed: 1200
Low processed:  400
```

**Что видно:** 3 high на 1 low. Метрики показывают пропорции.

### Пример: приоритетный merge

```go
func mergePriority(ctx context.Context, highCh, lowCh <-chan int) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        
        for {
            select {
            case <-ctx.Done():
                return
            default:
            }
            
            select {
            case v := <-highCh:
                select {
                case out <- v:
                case <-ctx.Done():
                    return
                }
                continue
            default:
            }
            
            select {
            case v := <-lowCh:
                select {
                case out <- v:
                case <-ctx.Done():
                    return
                }
                continue
            default:
            }
            
            select {
            case <-ctx.Done():
                return
            case v := <-highCh:
                select {
                case out <- v:
                case <-ctx.Done():
                    return
                }
            case v := <-lowCh:
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

### 💡 Практика: как измерять приоритетный select

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики:** high_processed, low_processed.
2. **Соотношение** — проверка весов.
3. **Экспорт в Prometheus.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Алерт при starvation.**
5. **Логирование редких событий.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `Mutex` для метрик.**

---

## 23.8 Выводы и типичные ошибки

**Что мы узнали?**

`select` не поддерживает приоритеты — выбирает **случайный** case. **Приоритетный select** реализуется через **вложенные `select` с `default`**: проверяем каналы по порядку, если никто не готов — блокируемся. **Взвешенные приоритеты** — обрабатывать N high, потом 1 low. **`reflect.Select`** — для динамических наборов каналов (N > 100). **`ctx.Done()`** — наивысший приоритет. Комбинируется с worker pool, pipeline, rate limiter.

**Типичные ошибки:**

- ❌ **Полагаться на порядок case'ов в `select`.** Выбирается случайный.
- ❌ **Не проверять `ctx.Err()` в начале итерации.** Отмена не сработает.
- ❌ **Busy loop при частых проверках.** Нужен блокирующий `select`.
- ❌ **`reflect.Select` для 2–10 каналов.** Overhead.
- ❌ **Не удалять закрытые каналы из `reflect.Select`.** Ошибки.
- ❌ **`highWeight` слишком большой.** Low голодает.
- ❌ **Не мониторить starvation.**
- ❌ **Не различать приоритеты в метриках.**

---

## 23.9 Для быстрого повторения

- **`select` не поддерживает приоритеты** — случайный выбор.
- **Приоритетный select** — вложенные `select` с `default`.
- **Порядок проверки:** ctx.Done() → high → low → блокировка.
- **`continue` после обработки** — для перепроверки.
- **Взвешенные приоритеты** — N high, потом 1 low.
- **`highWeight` 3–10.**
- **`reflect.Select`** — для динамических наборов.
- **`reflect.Select` в 10 раз медленнее** обычного.
- **`ctx.Done()` — наивысший приоритет.**
- **`ctx.Err()` — в начале итерации.**
- **Приоритеты + worker pool** — для критичных.
- **Приоритеты + pipeline** — стадия.
- **Метрики:** high_processed, low_processed.

---

## 23.10 Вопросы для самопроверки

1. Почему `select` не поддерживает приоритеты?
2. Как сделать приоритетный select?
3. Что такое starvation в select?
4. Как реализовать взвешенные приоритеты?
5. Что такое `reflect.Select`? Когда использовать?
6. Почему `reflect.Select` медленный?
7. Как `context` участвует в приоритетах?
8. Как комбинировать приоритеты с worker pool?

---

## 23.11 Ответы

### Ответ 1

**`select` не поддерживает приоритеты**, потому что выбирает **случайный** case из готовых. Это **намеренное** поведение — для **справедливости**. Если бы выбирался первый готовый, последние каналы **голодали** бы.

### Ответ 2

**Приоритетный select** — через вложенные `select` с `default`:

```go
for {
    select {
    case <-ctx.Done():
        return
    default:
    }
    
    select {
    case e := <-highCh:
        processHigh(e)
        continue
    default:
    }
    
    select {
    case e := <-lowCh:
        processLow(e)
        continue
    default:
    }
    
    select {
    case <-ctx.Done():
        return
    case e := <-highCh:
        processHigh(e)
    case e := <-lowCh:
        processLow(e)
    }
}
```

Проверяем каналы по порядку приоритета. Если никто не готов — блокируемся.

### Ответ 3

**Starvation в select** — ситуация, когда один case **никогда не выбирается**, потому что всегда есть более приоритетный.

**Пример:** если `highCh` всегда готов, `lowCh` голодает.

**В обычном `select`** starvation **невозможен** (случайный выбор). **В приоритетном** — возможен, если `highCh` всегда готов.

**Решение:** взвешенные приоритеты.

### Ответ 4

**Взвешенные приоритеты** — обрабатывать N high, потом 1 low:

```go
if highCount >= highWeight {
    select {
    case e := <-lowCh:
        processLow(e)
        highCount = 0
        continue
    default:
    }
}
```

`highWeight` = 3–10.

### Ответ 5

**`reflect.Select`** — для **динамических** наборов каналов. Когда N каналов **неизвестно** на этапе компиляции.

**Когда использовать:**
- N > 100.
- Динамическое добавление/удаление.

**Пример:**

```go
cases := make([]reflect.SelectCase, len(channels))
for i, ch := range channels {
    cases[i] = reflect.SelectCase{
        Dir:  reflect.SelectRecv,
        Chan: reflect.ValueOf(ch),
    }
}
idx, value, ok := reflect.Select(cases)
```

### Ответ 6

**`reflect.Select` медленный** в **10 раз** из-за overhead от reflection. Каждый вызов — это reflection API, что дороже обычного `select` в runtime.

**Использовать только** для N > 100 или динамических наборов.

### Ответ 7

**`context` в приоритетах:**

- **`ctx.Done()` — наивысший приоритет.** Проверяем **первым**.
- **`ctx.Err()` — в начале итерации.** Для немедленного выхода.

```go
if ctx.Err() != nil {
    return
}
```

**В блокирующем `select`** `ctx.Done()` участвует, но **не с приоритетом**. Если `highCh` и `ctx.Done()` готовы — выберется случайный.

### Ответ 8

**Приоритеты + worker pool:**

```go
for i := 0; i < 5; i++ {
    go priorityWorker(ctx, highCh, lowCh)
}

func priorityWorker(ctx context.Context, highCh, lowCh <-chan Task) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
        }
        
        select {
        case t := <-highCh:
            process(t)
            continue
        default:
        }
        
        select {
        case t := <-lowCh:
            process(t)
            continue
        default:
        }
        
        select {
        case <-ctx.Done():
            return
        case t := <-highCh:
            process(t)
        case t := <-lowCh:
            process(t)
        }
    }
}
```

N воркеров обрабатывают оба канала с приоритетом.

---

## 23.12 Куда идти дальше?

Мы разобрали приоритетный select — управление порядком обработки. Теперь мы умеем не давать низкоприоритетным событиям голодать.

Но остаётся **важный вопрос**: как выбрать **одного лидера** среди нескольких реплик? Как делать фоновые задачи в распределённой системе?

- **Как выбрать лидера?** → **Глава 24: Leader election.**
- **Как разбить лок на шарды?** → **Глава 25: Sharded locks.**
- **Как построить lock-free структуры?** → **Глава 26: Lock-free структуры.**

---

## 23.13 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **`select`** | Мультиплексор | Случайный выбор |
| **Приоритетный select** | Через `default` | Вложенные `select` |
| **Порядок проверки** | ctx → high → low | `continue` после обработки |
| **Starvation** | Один case голодает | Возможен в приоритетном |
| **Взвешенные приоритеты** | N high, 1 low | `highWeight` 3–10 |
| **`reflect.Select`** | Динамический select | N > 100, медленный |
| **`reflect.Select` overhead** | В 10 раз медленнее | — |
| **`ctx.Err()`** | В начале итерации | Немедленный выход |
| **`ctx.Done()`** | Наивысший приоритет | В блокирующем select |
| **Приоритеты + worker pool** | Для критичных | — |
| **Приоритеты + pipeline** | Стадия | — |
| **Метрики** | high_processed, low_processed | — |

🎯 **Ключевая идея:** `select` не поддерживает приоритеты — выбирает **случайный** case. **Приоритетный select** — через вложенные `select` с `default`: проверяем каналы по порядку, блокируемся если никто не готов. **Взвешенные приоритеты** — N high, потом 1 low. **`reflect.Select`** — для динамических наборов (N > 100), в 10 раз медленнее. **`ctx.Done()`** — наивысший приоритет. **`ctx.Err()`** — в начале итерации. Комбинируется с worker pool, pipeline, rate limiter. **Starvation** возможен в приоритетном select — решается взвешенными приоритетами. Метрики: high_processed, low_processed.