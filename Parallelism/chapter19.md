# 🔀 Глава 19: Tee и bridge каналы — разветвление и склейка

**Что вы узнаете:**
- Что такое tee-канал и какую задачу он решает.
- Что такое bridge-канал и какую задачу он решает.
- Чем tee отличается от fan-out.
- Как построить tee с нуля.
- Как построить bridge с нуля.
- Как добавить отмену через `context`.
- Как комбинировать tee и bridge с pipeline, fan-in, worker pool.

**После прочтения вы сможете:**
- Построить tee-канал с нуля.
- Построить bridge-канал с нуля.
- Понимать, где tee, где fan-out, где bridge.
- Комбинировать их с другими паттернами.
- Избегать утечек и блокировок.

---

## Содержание

- [19.0 Пролог: один поток — несколько потребителей](#190-пролог-один-поток--несколько-потребителей)
- [19.1 Что такое tee и bridge](#191-что-такое-tee-и-bridge)
- [19.2 Tee-канал с нуля](#192-tee-канал-с-нуля)
- [19.3 Bridge-канал с нуля](#193-bridge-канал-с-нуля)
- [19.4 Tee и bridge с context](#194-tee-и-bridge-с-context)
- [19.5 Tee vs fan-out](#195-tee-vs-fan-out)
- [19.6 В связке с другими паттернами](#196-в-связке-с-другими-паттернами)
- [19.7 Практика Go: tee и bridge](#197-практика-go-tee-и-bridge)
- [19.8 Выводы и типичные ошибки](#198-выводы-и-типичные-ошибки)
- [19.9 Для быстрого повторения](#199-для-быстрого-повторения)
- [19.10 Вопросы для самопроверки](#1910-вопросы-для-самопроверки)
- [19.11 Ответы](#1911-ответы)
- [19.12 Куда идти дальше?](#1912-куда-идти-дальше)
- [19.13 Чек-лист](#1913-чек-лист)

---

## 19.0 Пролог: один поток — несколько потребителей

У нас есть сервис, который читает данные из Kafka. Каждое сообщение нужно **и** записать в ClickHouse для аналитики, **и** обработать бизнес-логикой. Данные одни и те же — но обработка разная.

Простейшая мысль: **разветвить** поток на два канала.

```go
for msg := range kafkaCh {
    // Хочется передать msg И в analytics, И в business
}
```

Попробуем **fan-out** (Глава 10): два воркера читают из **одного** канала.

```go
go analyticsWorker(kafkaCh)  // ← заберёт часть
go businessWorker(kafkaCh)   // ← заберёт другую часть
```

**Проблема:** fan-out **распределяет** — каждое сообщение идёт **одному** воркеру. А нам нужно, чтобы **каждое** сообщение попало **в оба** обработчика.

Нужен другой паттерн — **tee-канал**. Он **дублирует** каждое сообщение в оба выходных канала.

Другая ситуация: у нас есть **канал каналов** — `chan chan int`. Это результат динамической генерации источников или bridge для склейки. Хочется **развернуть** его в один плоский поток. Это **bridge-канал**.

Оба паттерна — специфические, но полезные.

> **Мост к следующим главам:** tee и bridge дополняют fan-in (Глава 9) и fan-out (Глава 10). Tee дублирует, fan-out распределяет, fan-in сливает. Bridge разворачивает. Понимание этих паттернов даёт полный контроль над потоками.

---

## 19.1 Что такое tee и bridge

Разберём оба паттерна.

### Tee-канал

**Tee-канал** — дублирует **каждое** значение в **два** канала.

```
              ┌──────────┐
         ┌───▶│ Channel1 │
┌──────┐ │    └──────────┘
│Input │─┤
└──────┘ │    ┌──────────┐
         └───▶│ Channel2 │
              └──────────┘
```

**Что происходит:**

- Каждое значение из `input` идёт в **оба** канала.
- `out1` и `out2` получают **одинаковые** данные.

**Пример:** одно сообщение — в аналитику и в бизнес-логику.

**Название:** от Unix-утилиты `tee`, которая дублирует вывод.

### Bridge-канал

**Bridge-канал** — разворачивает `chan chan T` в `chan T`.

```
┌──────────────────┐
│ chan chan T      │
│ ┌──────────────┐ │
│ │ chan T       │ │
│ └──────────────┘ │
└──────────────────┘
        │
        ▼
┌──────────────────┐
│ chan T           │
│ (все элементы)   │
└──────────────────┘
```

**Что происходит:**

- Вход — канал каналов.
- Для каждого вложенного канала — читаем из него.
- Все значения идут в один выход.

**Пример:** динамическая генерация источников. Каждый источник — канал. Bridge склеивает их в один.

### Когда использовать tee

**1. Дублирование для разных обработчиков.**

- Аналитика + бизнес-логика.
- Логирование + обработка.
- Метрики + основной поток.

**2. Одно событие — несколько реакций.**

- Заказ — уведомление клиенту + обновление склада.
- Событие — метрика + запись в БД.

### Когда использовать bridge

**1. Динамическая генерация источников.**

- Пользователь выбирает N источников.
- Каждый источник — канал.
- Bridge склеивает их.

**2. Пагинация с параллельными запросами.**

- N страниц — N каналов.
- Bridge объединяет в один.

**3. Рекурсивные структуры.**

- Дерево — каждый узел даёт канал.
- Bridge разворачивает всё в плоский поток.

### Когда НЕ использовать

**Tee:**

- **Если нужны разные данные** — не дублируй, разделяй.
- **Если важна память** — tee держит несколько горутин.

**Bridge:**

- **Если канал один** — не нужен.
- **Если нужен fan-in** — используй `merge`.

### 💡 Практика: как думать о tee и bridge

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Tee — для дублирования.**
2. **Bridge — для разворачивания.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Tee + worker pool** — дублирование обработки.
4. **Bridge + pipeline** — динамические стадии.

**❌ НЕ ДЕЛАЙ:**

5. **Не путай tee с fan-out.** Tee дублирует, fan-out распределяет.
6. **Не путай bridge с merge.** Bridge разворачивает `chan chan T`, merge сливает N каналов.

---

## 19.2 Tee-канал с нуля

Начнём с простейшего tee.

### Идея

- Читаем из `input`.
- Пишем в `out1`.
- Пишем **то же** в `out2`.
- Закрываем оба при завершении.

### Реализация

```go
func tee(ctx context.Context, input <-chan int) (<-chan int, <-chan int) {
    out1 := make(chan int)
    out2 := make(chan int)
    
    go func() {
        defer close(out1)
        defer close(out2)
        
        for v := range input {
            // Дублируем в оба канала
            select {
            case out1 <- v:
            case <-ctx.Done():
                return
            }
            select {
            case out2 <- v:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    return out1, out2
}
```

**Что происходит:**

1. Создаём два выходных канала.
2. Читаем из входа.
3. Пишем значение в `out1` (ждём, пока прочитают).
4. Пишем значение в `out2` (ждём, пока прочитают).
5. Оба получают **одинаковые** данные.

### Полный пример

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

func tee(ctx context.Context, input <-chan int) (<-chan int, <-chan int) {
    out1 := make(chan int)
    out2 := make(chan int)
    
    go func() {
        defer close(out1)
        defer close(out2)
        
        for v := range input {
            select {
            case out1 <- v:
            case <-ctx.Done():
                return
            }
            select {
            case out2 <- v:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    return out1, out2
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    input := make(chan int, 10)
    go func() {
        defer close(input)
        for i := 1; i <= 5; i++ {
            input <- i
        }
    }()
    
    out1, out2 := tee(ctx, input)
    
    var wg sync.WaitGroup
    wg.Add(2)
    
    go func() {
        defer wg.Done()
        for v := range out1 {
            fmt.Printf("[out1] %d\n", v)
            time.Sleep(50 * time.Millisecond)
        }
    }()
    
    go func() {
        defer wg.Done()
        for v := range out2 {
            fmt.Printf("[out2] %d\n", v)
            time.Sleep(100 * time.Millisecond)
        }
    }()
    
    wg.Wait()
}
```

**Пример вывода:**

```
[out1] 1
[out2] 1
[out1] 2
[out2] 2
[out1] 3
[out2] 3
[out1] 4
[out2] 4
[out1] 5
[out2] 5
```

**Что видно:** каждое значение приходит в **оба** канала.

### Схема

```
input ──► tee ──► out1 (получает 1, 2, 3, ...)
              └─► out2 (получает 1, 2, 3, ...)
              
Медленный out2 → tee ждёт → input тоже ждёт
```

### Проблема: медленный потребитель

**Что если `out2` читается медленно?**

- Tee пишет в `out1`, ждёт, пока прочитают.
- Потом пишет в `out2`, ждёт, пока прочитают.
- Если `out2` медленный — tee ждёт.

**Результат:** `out1` тоже блокируется — tee последовательный.

### Решение: буферизованные каналы

```go
out1 := make(chan int, 10)
out2 := make(chan int, 10)
```

**Что даёт:** tee может писать в оба быстрее. Но если буферы полны — всё равно блокируется.

### Решение: отдельная горутина для каждого канала

```go
func tee(ctx context.Context, input <-chan int) (<-chan int, <-chan int) {
    out1 := make(chan int)
    out2 := make(chan int)
    
    go func() {
        defer close(out1)
        defer close(out2)
        
        for v := range input {
            v1, v2 := v, v
            
            select {
            case out1 <- v1:
            case <-ctx.Done():
                return
            }
            select {
            case out2 <- v2:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    return out1, out2
}
```

Это **та же реализация**. Проблема в том, что tee **последовательный**. Если один канал медленный — второй ждёт.

### Альтернатива: два независимых буфера

```go
func teeBuffered(ctx context.Context, input <-chan int, bufferSize int) (<-chan int, <-chan int) {
    out1 := make(chan int, bufferSize)
    out2 := make(chan int, bufferSize)
    
    go func() {
        defer close(out1)
        defer close(out2)
        
        for v := range input {
            select {
            case out1 <- v:
            case <-ctx.Done():
                return
            }
            select {
            case out2 <- v:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    return out1, out2
}
```

**Плюс:** можно буферизовать.

### 💡 Практика: как писать tee

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Два выходных канала.**
2. **Пишем в оба.**
3. **Закрываем оба.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Буфер для медленных потребителей.**
5. **`ctx` для отмены.**

**❌ НЕ ДЕЛАЙ:**

6. **Не пиши только в один — потеряешь данные.**
7. **Не забывай `close`.**

---

## 19.3 Bridge-канал с нуля

Разберём bridge.

### Идея

- Вход — `chan chan T` (канал каналов).
- Для каждого вложенного канала — читаем из него.
- Все значения пишем в **один** выходной канал.

### Реализация

```go
func bridge(ctx context.Context, chanCh <-chan (<-chan int)) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        
        for {
            var ch <-chan int
            select {
            case c, ok := <-chanCh:
                if !ok {
                    return
                }
                ch = c
            case <-ctx.Done():
                return
            }
            
            for v := range ch {
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

1. Читаем из `chanCh` — получаем **канал**.
2. Читаем из этого канала — получаем значения.
3. Пишем в `out`.
4. Когда внутренний канал закрыт — берём следующий из `chanCh`.
5. Когда `chanCh` закрыт — завершаемся.

### Полный пример

```go
package main

import (
    "context"
    "fmt"
)

func bridge(ctx context.Context, chanCh <-chan (<-chan int)) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        
        for {
            var ch <-chan int
            select {
            case c, ok := <-chanCh:
                if !ok {
                    return
                }
                ch = c
            case <-ctx.Done():
                return
            }
            
            for v := range ch {
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

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    chanCh := make(chan (<-chan int), 10)
    
    // Производим 3 канала
    go func() {
        defer close(chanCh)
        for i := 1; i <= 3; i++ {
            ch := make(chan int)
            chanCh <- ch
            
            go func(id int) {
                defer close(ch)
                for j := 0; j < 3; j++ {
                    ch <- id*10 + j
                }
            }(i)
        }
    }()
    
    for v := range bridge(ctx, chanCh) {
        fmt.Println(v)
    }
}
```

**Пример вывода:**

```
10
11
12
20
21
22
30
31
32
```

**Что видно:** значения из трёх вложенных каналов идут **последовательно** в один поток.

### Схема

```
chanCh ──► ch1 ──► 10 11 12
       ├─► ch2 ──► 20 21 22
       └─► ch3 ──► 30 31 32
              │
              ▼
          bridge
              │
              ▼
       10 11 12 20 21 22 30 31 32
       (все значения)
```

### Разница с merge

**Merge** (fan-in, Глава 9):

- Вход — **N каналов**.
- Все каналы читаются **параллельно**.
- Порядок **не гарантирован**.

**Bridge:**

- Вход — **один канал каналов**.
- Каналы читаются **последовательно** (один за другим).
- Порядок **гарантирован** по каналам.

### Когда что использовать

**Merge:**

- N каналов известно **заранее**.
- Нужна **параллельная** обработка.

**Bridge:**

- Каналы появляются **динамически**.
- Нужна **последовательная** обработка.

### Пример: bridge для дерева

```go
type Node struct {
    Value    int
    Children []*Node
}

func walkTree(ctx context.Context, node *Node) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        
        out <- node.Value
        
        // Дети — каналы каналов
        chanCh := make(chan (<-chan int), len(node.Children))
        for _, child := range node.Children {
            chanCh <- walkTree(ctx, child)
        }
        close(chanCh)
        
        // Bridge разворачивает
        for v := range bridge(ctx, chanCh) {
            out <- v
        }
    }()
    
    return out
}
```

**Что даёт:** обход дерева в плоский поток.

### 💡 Практика: как писать bridge

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Читаем `chanCh` в цикле.**
2. **Для каждого канала — читаем значения.**
3. **Пишем в `out`.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`ctx` для отмены.**

**❌ НЕ ДЕЛАЙ:**

5. **Не путай bridge с merge.**
6. **Не забывай `close(chanCh)`.**

---

## 19.4 Tee и bridge с context

`context` — критичен для tee и bridge. Без него горутины могут **зависнуть**.

### Tee с context

Мы уже добавили `ctx` в реализацию. Разберём подробнее.

```go
for v := range input {
    select {
    case out1 <- v:
    case <-ctx.Done():
        return
    }
    select {
    case out2 <- v:
    case <-ctx.Done():
        return
    }
}
```

**Что происходит:**

- При отмене — выходим.
- `defer close(out1)` и `defer close(out2)` закрывают каналы.

### Bridge с context

Тоже добавлен:

```go
for {
    var ch <-chan int
    select {
    case c, ok := <-chanCh:
        if !ok {
            return
        }
        ch = c
    case <-ctx.Done():
        return
    }
    
    for v := range ch {
        select {
        case out <- v:
        case <-ctx.Done():
            return
        }
    }
}
```

**Что происходит:**

- При отмене — выходим.
- `defer close(out)` закрывает канал.

### Полный пример с отменой

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
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
    
    out1, out2 := tee(ctx, input)
    
    var wg sync.WaitGroup
    wg.Add(2)
    
    go func() {
        defer wg.Done()
        for v := range out1 {
            fmt.Printf("[out1] %d\n", v)
        }
    }()
    
    go func() {
        defer wg.Done()
        for v := range out2 {
            fmt.Printf("[out2] %d\n", v)
        }
    }()
    
    wg.Wait()
    fmt.Println("done")
}
```

**Что происходит:** через 500 мс `ctx` отменяется. Tee останавливается, оба канала закрываются.

### Схема

```
Tee:
  for v := range input {
    select {
    case out1 <- v:      ← отмена через ctx
    case <-ctx.Done():
      return
    }
    select {
    case out2 <- v:
    case <-ctx.Done():
      return
    }
  }

Bridge:
  for {
    select {
    case c := <-chanCh:  ← отмена через ctx
    case <-ctx.Done():
      return
    }
    for v := range ch {
      select {
      case out <- v:
      case <-ctx.Done():
        return
      }
    }
  }
```

### 💡 Практика: как добавить context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ctx` — первый аргумент.**
2. **`select` с `ctx.Done()`.**
3. **`defer close`.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`defer cancel()` у потребителя.**

**❌ НЕ ДЕЛАЙ:**

5. **Не блокируйся без `ctx`.**

---

## 19.5 Tee vs fan-out

Разберём **разницу** между tee и fan-out.

### Fan-out (Глава 10)

**Fan-out** — **распределение** работы.

```
input ──► worker1 (получает 1, 3, 5)
       └─► worker2 (получает 2, 4, 6)

Каждое значение идёт ОДНОМУ воркеру.
```

### Tee

**Tee** — **дублирование**.

```
input ──► out1 (получает 1, 2, 3, ...)
       └─► out2 (получает 1, 2, 3, ...)

Каждое значение идёт ОБОИМ каналам.
```

### Сравнение

| Аспект | Fan-out | Tee |
|:---|:---|:---|
| Что делает | Распределяет | Дублирует |
| Каждое значение | Одному | Всем |
| Параллелизм | Да | Последовательно |
| Backpressure | Через общий канал | Через каждый канал |
| Use case | Worker pool | Логирование + обработка |

### Как выбрать

**Fan-out:**

- **Распределить работу** между воркерами.
- **Ускорить** обработку.
- **Задачи однотипны.**

**Tee:**

- **Дублировать** для разных обработчиков.
- **Одно событие — несколько реакций.**
- **Обработки разные.**

### Пример: fan-out + tee

**Fan-out** для параллельной обработки, **tee** для логирования:

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Источник
    source := numbersCtx(ctx, 100)
    
    // Tee: одна ветка в лог, другая в обработку
    logCh, workCh := tee(ctx, source)
    
    // Лог
    go func() {
        for v := range logCh {
            log.Printf("processing %d", v)
        }
    }()
    
    // Fan-out: 5 воркеров
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for v := range workCh {
                process(v)
            }
        }()
    }
    wg.Wait()
}
```

**Что происходит:**

- Одно сообщение идёт **в лог** (tee).
- **То же** сообщение идёт в **один из 5 воркеров** (fan-out).

### Схема

```
source ──► tee ──► logCh (все)
              └─► workCh ──► fan-out ──► 5 воркеров
```

### 💡 Практика: как выбирать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Fan-out — для параллельной обработки.**
2. **Tee — для дублирования.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Tee + fan-out** — для логирования + параллельной обработки.

**❌ НЕ ДЕЛАЙ:**

4. **Не путай tee с fan-out.**
5. **Не используй fan-out для дублирования.**

---

## 19.6 В связке с другими паттернами

Tee и bridge редко используются **в одиночку**. Разберём связки.

### Tee + pipeline

**Tee как стадия pipeline:**

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    source := numbersCtx(ctx, 100)
    
    // Стадия обработки
    processed := doubleStage(ctx, source)
    
    // Tee: одна ветка в лог, другая в запись
    logCh, writeCh := tee(ctx, processed)
    
    // Лог
    go func() {
        for v := range logCh {
            log.Printf("writing %d", v)
        }
    }()
    
    // Запись
    for v := range writeCh {
        save(v)
    }
}
```

### Tee + fan-out

**Tee для разветвления, fan-out для параллельной обработки:**

```go
source := numbersCtx(ctx, 100)
analyticsCh, businessCh := tee(ctx, source)

// Аналитика: 1 воркер
go analyticsWorker(analyticsCh)

// Бизнес: 5 воркеров (fan-out)
var wg sync.WaitGroup
for i := 0; i < 5; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for v := range businessCh {
            process(v)
        }
    }()
}
wg.Wait()
```

### Bridge + pipeline

**Bridge как стадия pipeline:**

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Динамически генерируем источники
    sourcesCh := make(chan (<-chan int), 10)
    go func() {
        defer close(sourcesCh)
        for i := 0; i < 5; i++ {
            sourcesCh <- numbersCtx(ctx, 10)
        }
    }()
    
    // Bridge склеивает
    merged := bridge(ctx, sourcesCh)
    
    // Дальше — обычная обработка
    doubled := doubleStage(ctx, merged)
    
    for v := range doubled {
        fmt.Println(v)
    }
}
```

### Bridge + fan-out

**Bridge + fan-out для параллельной обработки:**

```go
sourcesCh := make(chan (<-chan int), 10)
// ... заполняем канал каналов

merged := bridge(ctx, sourcesCh)

// Fan-out: 5 воркеров
var wg sync.WaitGroup
for i := 0; i < 5; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for v := range merged {
            process(v)
        }
    }()
}
wg.Wait()
```

### Полная схема

```
Источники ──► bridge ──► tee ──┬──► log
                                └──► fan-out ──► workers
```

### 💡 Практика: как комбинировать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Tee + pipeline** — для логирования.
2. **Tee + fan-out** — для разных обработчиков.
3. **Bridge + pipeline** — для динамических источников.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Bridge + fan-out** — для параллельной обработки.

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай `ctx`.**

---

## 19.7 Практика Go: tee и bridge

Разберём **три примера**.

### Пример 1: tee

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

func tee(ctx context.Context, input <-chan int) (<-chan int, <-chan int) {
    out1 := make(chan int)
    out2 := make(chan int)
    
    go func() {
        defer close(out1)
        defer close(out2)
        
        for v := range input {
            select {
            case out1 <- v:
            case <-ctx.Done():
                return
            }
            select {
            case out2 <- v:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    return out1, out2
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    input := make(chan int, 10)
    go func() {
        defer close(input)
        for i := 1; i <= 5; i++ {
            input <- i
        }
    }()
    
    out1, out2 := tee(ctx, input)
    
    var wg sync.WaitGroup
    wg.Add(2)
    
    go func() {
        defer wg.Done()
        for v := range out1 {
            fmt.Printf("[out1] %d\n", v)
            time.Sleep(50 * time.Millisecond)
        }
    }()
    
    go func() {
        defer wg.Done()
        for v := range out2 {
            fmt.Printf("[out2] %d\n", v)
            time.Sleep(100 * time.Millisecond)
        }
    }()
    
    wg.Wait()
}
```

**Пример вывода:**

```
[out1] 1
[out2] 1
[out1] 2
[out2] 2
[out1] 3
[out2] 3
...
```

### Пример 2: bridge

```go
package main

import (
    "context"
    "fmt"
)

func bridge(ctx context.Context, chanCh <-chan (<-chan int)) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        
        for {
            var ch <-chan int
            select {
            case c, ok := <-chanCh:
                if !ok {
                    return
                }
                ch = c
            case <-ctx.Done():
                return
            }
            
            for v := range ch {
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

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    chanCh := make(chan (<-chan int), 10)
    
    go func() {
        defer close(chanCh)
        for i := 1; i <= 3; i++ {
            ch := make(chan int, 3)
            chanCh <- ch
            
            for j := 0; j < 3; j++ {
                ch <- i*10 + j
            }
            close(ch)
        }
    }()
    
    for v := range bridge(ctx, chanCh) {
        fmt.Println(v)
    }
}
```

**Пример вывода:**

```
10
11
12
20
21
22
30
31
32
```

### Пример 3: tee + fan-out

```go
package main

import (
    "context"
    "fmt"
    "sync"
)

func tee(ctx context.Context, input <-chan int) (<-chan int, <-chan int) {
    out1 := make(chan int)
    out2 := make(chan int)
    
    go func() {
        defer close(out1)
        defer close(out2)
        
        for v := range input {
            select {
            case out1 <- v:
            case <-ctx.Done():
                return
            }
            select {
            case out2 <- v:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    return out1, out2
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Источник
    input := make(chan int, 100)
    go func() {
        defer close(input)
        for i := 1; i <= 20; i++ {
            input <- i
        }
    }()
    
    // Tee: одна ветка в лог, другая в обработку
    logCh, workCh := tee(ctx, input)
    
    // Лог
    var logWg sync.WaitGroup
    logWg.Add(1)
    go func() {
        defer logWg.Done()
        count := 0
        for range logCh {
            count++
        }
        fmt.Printf("Logged: %d\n", count)
    }()
    
    // Fan-out: 5 воркеров
    var wg sync.WaitGroup
    var processed int64
    var mu sync.Mutex
    
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for v := range workCh {
                mu.Lock()
                processed++
                mu.Unlock()
                _ = v
            }
        }(i)
    }
    wg.Wait()
    logWg.Wait()
    
    fmt.Printf("Processed: %d\n", processed)
}
```

**Пример вывода:**

```
Logged: 20
Processed: 20
```

**Что видно:** каждое значение попало **и** в лог, **и** в обработку.

### 💡 Практика: как использовать tee и bridge

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Tee — для дублирования.**
2. **Bridge — для разворачивания.**
3. **`ctx` — для отмены.**
4. **`defer close` — обязательно.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Tee + fan-out** — для разных обработчиков.
6. **Bridge + pipeline** — для динамических источников.

**❌ НЕ ДЕЛАЙ:**

7. **Не путай tee с fan-out.**
8. **Не путай bridge с merge.**

---

## 19.8 Выводы и типичные ошибки

**Что мы узнали?**

Tee дублирует каждое значение в два канала. Bridge разворачивает `chan chan T` в `chan T`. Tee отличается от fan-out: fan-out распределяет, tee дублирует. Bridge отличается от merge: merge параллельный, bridge последовательный. `context` для отмены. Tee + fan-out для логирования + параллельной обработки. Bridge + pipeline для динамических источников.

**Типичные ошибки:**

- ❌ **Путать tee с fan-out.** Tee дублирует, fan-out распределяет.
- ❌ **Путать bridge с merge.** Bridge разворачивает, merge сливает.
- ❌ **Забыть `close` в tee/bridge.** Утечка.
- ❌ **Не использовать `ctx`.** Зависание.
- ❌ **Медленный потребитель в tee.** Блокирует всё.
- ❌ **Не закрывать вложенные каналы.** Bridge зависнет.
- ❌ **Не закрывать `chanCh`.** Bridge зависнет.

---

## 19.9 Для быстрого повторения

- **Tee** — дублирует в два канала.
- **Bridge** — разворачивает `chan chan T`.
- **Tee ≠ fan-out:** дублирование vs распределение.
- **Bridge ≠ merge:** последовательный vs параллельный.
- **Tee:** 1 горутина, 2 канала, `defer close` оба.
- **Bridge:** 1 горутина, читаем `chanCh`, читаем вложенные.
- **`ctx`** для отмены.
- **Tee + fan-out:** логирование + параллельная обработка.
- **Bridge + pipeline:** динамические источники.
- **Tee для:** аналитика + бизнес-логика.
- **Bridge для:** динамическая генерация источников.

---

## 19.10 Вопросы для самопроверки

1. Что такое tee-канал? Какую задачу решает?
2. Что такое bridge-канал? Какую задачу решает?
3. Чем tee отличается от fan-out?
4. Чем bridge отличается от merge?
5. Как построить tee с нуля?
6. Как построить bridge с нуля?
7. Зачем `ctx` в tee и bridge?
8. Как комбинировать tee с fan-out?

---

## 19.11 Ответы

### Ответ 1

**Tee-канал** — дублирует **каждое** значение в **два** канала. Решает задачу: **одно событие — несколько реакций**.

**Пример:** сообщение — в аналитику **и** в бизнес-логику.

### Ответ 2

**Bridge-канал** — разворачивает `chan chan T` в `chan T`. Решает задачу: **динамическая генерация источников**.

**Пример:** пользователь выбрал N источников — каждый канал. Bridge склеивает в один.

### Ответ 3

**Fan-out** — **распределяет**. Каждое значение идёт **одному** воркеру.

**Tee** — **дублирует**. Каждое значение идёт **обоим** каналам.

### Ответ 4

**Merge** — слияние **N каналов** в один. Каналы читаются **параллельно**.

**Bridge** — разворачивание **канала каналов**. Каналы читаются **последовательно**.

### Ответ 5

**Tee:**

```go
func tee(ctx context.Context, input <-chan int) (<-chan int, <-chan int) {
    out1 := make(chan int)
    out2 := make(chan int)
    
    go func() {
        defer close(out1)
        defer close(out2)
        
        for v := range input {
            select {
            case out1 <- v:
            case <-ctx.Done():
                return
            }
            select {
            case out2 <- v:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    return out1, out2
}
```

### Ответ 6

**Bridge:**

```go
func bridge(ctx context.Context, chanCh <-chan (<-chan int)) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        
        for {
            var ch <-chan int
            select {
            case c, ok := <-chanCh:
                if !ok {
                    return
                }
                ch = c
            case <-ctx.Done():
                return
            }
            
            for v := range ch {
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

### Ответ 7

**`ctx`** позволяет остановить tee/bridge при отмене. Без него горутины могут **зависнуть**, если потребитель перестал читать.

### Ответ 8

**Tee + fan-out:**

```go
source := numbersCtx(ctx, 100)
logCh, workCh := tee(ctx, source)

// Лог
go func() {
    for v := range logCh {
        log.Printf("processing %d", v)
    }
}()

// Fan-out: 5 воркеров
var wg sync.WaitGroup
for i := 0; i < 5; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for v := range workCh {
            process(v)
        }
    }()
}
wg.Wait()
```

Tee дублирует: одна ветка в лог, другая — в fan-out.

---

## 19.12 Куда идти дальше?

Мы разобрали tee и bridge — разветвление и склейку. Теперь мы умеем работать со сложными структурами каналов.

Но остаются **продвинутые паттерны**: future/promise, bulkhead, leader election.

- **Как построить future/promise?** → **Глава 20: Future/Promise.**
- **Как построить bulkhead?** → **Глава 22: Bulkhead.**
- **Как построить leader election?** → **Глава 24: Leader election.**

---

## 19.13 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Tee** | Дублирование | 2 канала, одинаковые данные |
| **Bridge** | Разворачивание | `chan chan T` → `chan T` |
| **Tee ≠ fan-out** | Дублирование vs распределение | — |
| **Bridge ≠ merge** | Последовательный vs параллельный | — |
| **Tee горутина** | 1 | Пишет в 2 канала |
| **Bridge горутина** | 1 | Читает chanCh + вложенные |
| **`ctx`** | Отмена | `select` с `ctx.Done()` |
| **`defer close`** | Закрытие | Обязательно |
| **Tee + fan-out** | Логирование + обработка | — |
| **Bridge + pipeline** | Динамические источники | — |
| **Tee для** | Аналитика + бизнес | — |
| **Bridge для** | Динамические источники | — |

🔀 **Ключевая идея:** Tee **дублирует** каждое значение в два канала. Bridge **разворачивает** `chan chan T` в `chan T`. Tee ≠ fan-out: дублирование vs распределение. Bridge ≠ merge: последовательный vs параллельный. `ctx` для отмены, `defer close` обязательно. Tee + fan-out для логирования + параллельной обработки. Bridge + pipeline для динамических источников. Tee для аналитика + бизнес-логика. Bridge для динамической генерации источников.