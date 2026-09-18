# 🔀 Глава 8: Fan-in, Fan-out, Pipeline — конвейеры данных

**Что вы узнаете:**
- Что такое **fan-out** и две его реализации: N каналов и один общий.
- Что такое **fan-in** и как слить N каналов в один.
- Что такое **pipeline** и как построить многостадийную обработку.
- Как работает **backpressure** между стадиями pipeline.
- Что такое **tee-канал** и **bridge-канал**.
- Как **отменять** pipeline через `context.Context`.
- Как **закрывать** каналы в pipeline без утечек и deadlock.
- Как диагностировать проблемы pipeline через pprof и метрики.
- Как комбинировать pipeline с worker pool (Глава 7).

**После прочтения вы сможете:**
- Построить fan-out/fan-in для распределения и сбора работы.
- Выбирать между двумя реализациями fan-out.
- Построить pipeline из N стадий с backpressure.
- Правильно закрывать каналы в каждой стадии.
- Отменять pipeline через `context`.
- Реализовать tee-канал (разветвление) и bridge-канал (склейка).
- Диагностировать утечки и блокировки в pipeline.
- Измерять пропускную способность и latency каждой стадии.

---

## Содержание

- [8.0 Пролог: конвейер, который встал](#80-пролог-конвейер-который-встал)
- [8.1 Fan-out: распределение работы](#81-fan-out-распределение-работы)
- [8.2 Fan-in: слияние результатов](#82-fan-in-слияние-результатов)
- [8.3 Fan-out + Fan-in: полная схема](#83-fan-out--fan-in-полная-схема)
- [8.4 Pipeline: многостадийная обработка](#84-pipeline-многостадийная-обработка)
- [8.5 Backpressure в pipeline](#85-backpressure-в-pipeline)
- [8.6 Закрытие каналов в pipeline](#86-закрытие-каналов-в-pipeline)
- [8.7 Отмена pipeline через context](#87-отмена-pipeline-через-context)
- [8.8 Tee-канал и bridge-канал](#88-tee-канал-и-bridge-канал)
- [8.9 Pipeline vs worker pool: что выбрать](#89-pipeline-vs-worker-pool-что-выбрать)
- [8.10 Практика Go: pipeline с метриками](#810-практика-go-pipeline-с-метриками)
- [8.11 Выводы и типичные ошибки](#811-выводы-и-типичные-ошибки)
- [8.12 Для быстрого повторения](#812-для-быстрого-повторения)
- [8.13 Вопросы для самопроверки](#813-вопросы-для-самопроверки)
- [8.14 Ответы](#814-ответы)
- [8.15 Куда идти дальше?](#815-куда-идти-дальше)
- [8.16 Чек-лист](#816-чек-лист)

---

## 8.0 Пролог: конвейер, который встал

Ты пишешь ETL-пайплайн: читаешь логи из Kafka, парсишь, обогащаешь данными из БД, записываешь в ClickHouse.

Наивная реализация:

```go
func main() {
    for {
        msg := kafka.Read()
        parsed := parse(msg)
        enriched := enrich(parsed)
        writeToClickHouse(enriched)
    }
}
```

Всё работает. Пока не выясняется:

- **Kafka быстрая** — 100 000 сообщений/сек.
- **Парсинг быстрый** — 1 мкс на сообщение.
- **Обогащение медленное** — 10 мс (запрос в БД).
- **ClickHouse быстрый** — 100 мкс на запись.

**Пропускная способность:** 1 / 10 мс = **100 сообщений/сек**. Хотя Kafka даёт 100 000.

❓ **Что произошло?** Ты обрабатываешь сообщения **последовательно**. Обогащение — bottleneck. Пока одно сообщение обогащается, остальные ждут.

💡 **Решение:** **pipeline** + **fan-out**.

- **Стадия 1:** читаем из Kafka.
- **Стадия 2:** парсим (быстро).
- **Стадия 3:** обогащаем — **100 параллельных воркеров** (fan-out).
- **Стадия 4:** пишем в ClickHouse.

**Пропускная способность:** 100 воркеров × (1 / 10 мс) = **10 000 сообщений/сек**. В 100 раз быстрее.

> **Важный мост к будущим главам:** pipeline — это обобщение worker pool (Глава 7). Fan-out — это worker pool для одной стадии. Fan-in — это слияние каналов. Backpressure — из Главы 7. Отмена через `context` — из Главы 5.

---

## 8.1 Fan-out: распределение работы

**Fan-out** — это распределение работы от **одного** источника к **нескольким** обработчикам.

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

**Ключевое:** fan-out — это **распределение**, не **дублирование**. Каждое значение идёт **одному** воркеру.

### Шаг 1: generator — источник данных

Начнём с простого: функция, которая отдаёт данные в канал.

```go
func generator(data []int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for _, v := range data {
			out <- v
		}
	}()

	return out
}
```

**Что делает:**

1. Создаёт канал.
2. Запускает горутину, которая пишет данные.
3. Когда данные закончились — `close(out)`.
4. Возвращает канал.

**Использование:**

```go
ch := generator([]int{1, 2, 3, 4, 5})
for v := range ch {
    fmt.Println(v)
}
```

### Шаг 2: worker — обработчик одной задачи

Прежде чем делать fan-out, определим, **что делает один воркер**.

```go
func worker(input <-chan int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for v := range input {
			out <- process(v)
		}
	}()

	return out
}

func process(v int) int {
	return v * 2  // для примера
}
```

**Что делает:**

1. Создаёт **свой** выходной канал.
2. Читает из **общего** входного канала.
3. Обрабатывает каждое значение.
4. Пишет в **свой** выход.
5. При закрытии входа — `close(out)`.

**Ключевое:** каждый воркер **не знает** о других воркерах. Он просто читает из входа.

### Шаг 3: fan-out — N воркеров

Теперь запустим **N воркеров**, каждый читает из **одного** входа.

```go
func fanOut(input <-chan int, n int) []chan int {
	outputs := make([]chan int, n)
	for i := 0; i < n; i++ {
		outputs[i] = worker(input)
	}
	return outputs
}
```

**Что делает:**

1. Создаёт слайс из N каналов.
2. Для каждого — запускает воркер.
3. Возвращает слайс каналов.

**Что происходит:**

- N горутин читают из **одного** `input`.
- Распределение **автоматическое** — какое значение попадёт какому воркеру, решает runtime.
- Каждый воркер пишет в **свой** `outputs[i]`.

**Визуализация:**

```
input: [1, 2, 3, 4, 5, 6, 7, 8]

Воркер 0: [1, 5, 7]      → outputs[0]
Воркер 1: [2, 4, 8]      → outputs[1]
Воркер 2: [3, 6]         → outputs[2]

Распределение случайное. Каждое значение попадает ТОЛЬКО в один output.
```

### Шаг 4: полный main

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	data := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	input := generator(data)

	const numWorkers = 3
	outputs := fanOut(input, numWorkers)

	var wg sync.WaitGroup
	for i, out := range outputs {
		wg.Add(1)
		go func(id int, ch chan int) {
			defer wg.Done()
			for v := range ch {
				fmt.Printf("Worker %d: %d\n", id, v)
			}
		}(i, out)
	}
	wg.Wait()
	fmt.Println("done")
}

func generator(data []int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for _, v := range data {
			out <- v
		}
	}()

	return out
}

func fanOut(input <-chan int, n int) []chan int {
	outputs := make([]chan int, n)
	for i := 0; i < n; i++ {
		outputs[i] = worker(input)
	}
	return outputs
}

func worker(input <-chan int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for v := range input {
			out <- process(v)
		}
	}()

	return out
}

func process(v int) int {
	return v * 2
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

**Что видно:** значения распределились между воркерами. Каждое значение обработано **один раз**.

### Шаг 5: альтернативная реализация — один общий канал результатов

В **реализации A** (шаги 1–4) каждый воркер пишет в **свой** канал. Но чаще воркеры пишут в **один общий** канал.

**Реализация B:**

```go
func fanOutShared(input <-chan int, n int) chan int {
	out := make(chan int)

	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for v := range input {
				out <- process(v)
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

**Что изменилось:**

- **Один** канал результатов вместо N.
- N воркеров пишут в него.
- `wg.Wait()` + `close(out)` — как в fan-in.

**Разберём по частям.**

#### Шаг 5.1: запуск N воркеров

```go
for i := 0; i < n; i++ {
	wg.Add(1)
	go func() {
		defer wg.Done()
		for v := range input {
			out <- process(v)
		}
	}()
}
```

**Что делает:**

- Запускает N горутин.
- Каждая читает из **общего** `input`.
- Каждая пишет в **общий** `out`.

**Ключевое:** `input` и `out` — **одни и те же** каналы для всех воркеров.

#### Шаг 5.2: закрытие out

```go
go func() {
	wg.Wait()
	close(out)
}()
```

**Что делает:**

- Отдельная горутина ждёт `wg.Wait()`.
- Когда **все** воркеры завершились — `close(out)`.

**Почему отдельная горутина:** если `wg.Wait()` и `close(out)` в main — main заблокируется на `Wait`, `out` не закроется, читатели зависнут.

### Шаг 6: полный main для реализации B

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	data := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	input := generator(data)

	const numWorkers = 3
	output := fanOutShared(input, numWorkers)

	for v := range output {
		fmt.Println(v)
	}
	fmt.Println("done")
}

func generator(data []int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for _, v := range data {
			out <- v
		}
	}()

	return out
}

func fanOutShared(input <-chan int, n int) chan int {
	out := make(chan int)

	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for v := range input {
				out <- process(v)
			}
		}()
	}

	go func() {
		wg.Wait()
		close(out)
	}()

	return out
}

func process(v int) int {
	return v * 2
}
```

**Пример вывода:**

```
4
2
6
8
10
12
14
16
18
20
```

**Что видно:** все значения в **одном** канале. Порядок **не гарантирован** — воркеры пишут параллельно.

### Сравнение реализаций

| Аспект | A: N каналов | B: один канал |
|:---|:---|:---|
| Каналов результатов | N | 1 |
| Горутин | N (воркеры) + N (читатели) | N (воркеры) + 1 (close) |
| Сложность | Средняя | Простая |
| Потребитель | Читает N каналов | Читает 1 канал |
| Гибкость | Высокая (разные обработчики) | Низкая |
| Contention | Высокий (N+1 каналов) | Низкий (1 канал) |

**Рекомендация:**

- **Реализация B (один канал)** — для большинства случаев.
- **Реализация A (N каналов)** — если нужно обрабатывать результаты **разных** воркеров по-разному.

### Шаг 7: обработка N каналов через `received` + `ch = nil`

В **реализации A** потребитель читает из N каналов. Наивный подход — `select` в цикле. Но есть **баг**, который часто допускают.

**❌ Неправильно: `i--` при `!ok`**

```go
for i := 0; i < len(data); i++ {
	select {
	case v, ok := <-ch1:
		if !ok {
			i--       // ← БАГ
			continue
		}
		fmt.Println("Channel 1:", v)
	case v, ok := <-ch2:
		if !ok {
			i--
			continue
		}
		fmt.Println("Channel 2:", v)
	case v, ok := <-ch3:
		if !ok {
			i--
			continue
		}
		fmt.Println("Channel 3:", v)
	}
}
```

**Проблема:** `i--` при `!ok` — это **busy loop**. Если `ch1` закрыт, `<-ch1` возвращает `0, false` **немедленно**. Цикл крутится, `i--` компенсирует, но если все каналы закрыты, а `i < len(data)` — **бесконечный цикл**.

**✅ Правильно: `received` + `ch = nil`**

```go
total := len(data)
received := 0

for received < total {
	select {
	case v, ok := <-ch1:
		if ok {
			fmt.Println("Channel 1:", v)
			received++
		} else {
			ch1 = nil  // ← отключаем case
		}
	case v, ok := <-ch2:
		if ok {
			fmt.Println("Channel 2:", v)
			received++
		} else {
			ch2 = nil
		}
	case v, ok := <-ch3:
		if ok {
			fmt.Println("Channel 3:", v)
			received++
		} else {
			ch3 = nil
		}
	}
}
```

**Почему `ch1 = nil` работает:**

- Приём из nil-канала **блокируется навсегда** (Глава 2, подглава 2.8).
- `select` **игнорирует** case с nil-каналом.
- Так закрытые каналы «выключаются» из `select`.

**Полный main для реализации A:**

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	data := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	ch := generator(data)

	ch1 := fanOut(ch)
	ch2 := fanOut(ch)
	ch3 := fanOut(ch)

	total := len(data)
	received := 0

	for received < total {
		select {
		case v, ok := <-ch1:
			if ok {
				fmt.Println("Channel 1:", v)
				received++
			} else {
				ch1 = nil
			}
		case v, ok := <-ch2:
			if ok {
				fmt.Println("Channel 2:", v)
				received++
			} else {
				ch2 = nil
			}
		case v, ok := <-ch3:
			if ok {
				fmt.Println("Channel 3:", v)
				received++
			} else {
				ch3 = nil
			}
		}
	}
}

func generator(data []int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for _, v := range data {
			out <- v
		}
	}()

	return out
}

func fanOut(ch <-chan int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for v := range ch {
			out <- v
		}
	}()

	return out
}
```

**Важное замечание:** в твоём варианте `fanOut(ch)` создаёт **отдельную** горутину для **каждого** вызова. Три вызова → три горутины, конкурирующие за чтение из `ch`. Это **распределение**, не дублирование. Каждое значение попадёт **только в один** из `ch1`, `ch2`, `ch3`.

### Шаг 8: fan-out с отменой через context

Добавим `context` для отмены.

```go
func fanOutCtx(ctx context.Context, input <-chan int, n int) []chan int {
	outputs := make([]chan int, n)
	for i := 0; i < n; i++ {
		outputs[i] = workerCtx(ctx, input)
	}
	return outputs
}

func workerCtx(ctx context.Context, input <-chan int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for v := range input {
			select {
			case out <- process(v):
			case <-ctx.Done():
				return
			}
		}
	}()

	return out
}
```

**Что изменилось:** воркер проверяет `ctx.Done()` при отправке. Если контекст отменён — завершается.

**Аналогично для реализации B:**

```go
func fanOutSharedCtx(ctx context.Context, input <-chan int, n int) chan int {
	out := make(chan int)

	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for v := range input {
				select {
				case out <- process(v):
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

### Сколько воркеров

**Те же правила, что в Главе 7:**

- CPU-bound → `GOMAXPROCS`.
- I/O-bound → 10-100.

### Аннотация сложности

| Аспект | Реализация A (N каналов) | Реализация B (один канал) |
|:---|:---|:---|
| Каналов результатов | N | 1 |
| Горутин | 2N + 1 | N + 2 |
| Память | 2N × 2.3 КБ | (N + 2) × 2.3 КБ |
| Contention | Высокий | Низкий |
| Гибкость | Высокая | Низкая |

### 💡 Практика: как использовать fan-out

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Реализация B (один канал)** — для большинства случаев.
2. **Реализация A (N каналов)** — если нужна гибкость.
3. **N по типу задачи:** CPU-bound → `GOMAXPROCS`, I/O-bound → 10-100.
4. **Отключай закрытые каналы через `ch = nil`**, не через `i--`.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Отмена через `context`** — в `select` при отправке.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Динамическое N** — для переменной нагрузки (см. 7.7).

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `i--` при `!ok`.** Busy loop.
8. **Не путай fan-out с tee.** Fan-out распределяет, tee дублирует.
9. **Не создавай канал на каждого воркера**, если можно обойтись одним.

### Ключевые выводы подглавы 8.1

- **Fan-out** — распределение работы от одного источника к N обработчикам.
- **Реализация A:** N каналов результатов, N воркеров + N читателей. Гибко, но больше горутин.
- **Реализация B:** один канал результатов, N воркеров + 1 для close. Проще и быстрее.
- **Каждое значение идёт одному воркеру** (не дублируется).
- **Отключение закрытых каналов через `ch = nil`**, не через `i--`.
- **N** — по типу задачи.

---

## 8.2 Fan-in: слияние результатов

**Fan-in** — это слияние **нескольких** каналов в **один**.

### Схема

```
   ┌─────────┐
   │Channel 1│──┐
   └─────────┘  │
   ┌─────────┐  │
   │Channel 2│──┤   ┌──────────┐
   └─────────┘  ├──▶│ Combined │
   ┌─────────┐  │   └──────────┘
   │Channel 3│──┤
   └─────────┘  │
   ┌─────────┐  │
   │Channel N│──┘
   └─────────┘
```

**N каналов** объединяются в **один**. Все результаты идут в один поток.

### Шаг 1: makeChan — источники данных

Начнём с простого генератора:

```go
func makeChan(data []int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for _, v := range data {
			out <- v
		}
	}()

	return out
}
```

**Использование:**

```go
ch1 := makeChan([]int{1, 2, 3, 4, 5})
ch2 := makeChan([]int{6, 7, 8, 9, 10})
```

**Что происходит:** два независимых канала, каждый отдаёт свои данные.

### Шаг 2: наивный merge — один канал

**❌ Наивная попытка:** читать из двух каналов в одной горутине.

```go
func naiveMerge(ch1, ch2 <-chan int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for v := range ch1 {
			out <- v
		}
		for v := range ch2 {
			out <- v
		}
	}()

	return out
}
```

**Проблема:** если `ch1` долго не закрывается, `ch2` не читается. **Нет параллелизма.**

**Что происходит:**

```
t=0:    Читаем из ch1
        ch1 отдаёт 1, 2, 3, ...
        ch2 ждёт (никто не читает)

t=T:    ch1 закрылся
        Начинаем читать ch2
        ch2 отдаёт 6, 7, 8, ...
```

**Результат:** значения из `ch2` идут **после** всех значений из `ch1`.

### Шаг 3: fan-in через N горутин

**Правильная реализация:** для **каждого** входного канала — **своя** горутина.

```go
func fanIn(channels ...<-chan int) chan int {
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

**Разберём по частям.**

#### Шаг 3.1: один канал — одна горутина

```go
for _, ch := range channels {
	wg.Add(1)
	go func(c <-chan int) {
		defer wg.Done()
		for v := range c {
			out <- v
		}
	}(ch)
}
```

**Что делает:**

- Для **каждого** входного канала запускается горутина.
- Горутина читает из **своего** канала.
- Пишет в **общий** `out`.

**Ключевое:** `ch` передаётся как **аргумент** (`c <-chan int`), а не захватывается замыканием. Иначе все горутины читали бы из **последнего** `ch`.

**Почему это работает:**

- `ch1` читается горутиной 1.
- `ch2` читается горутиной 2.
- Обе пишут в `out`.
- `out` получает значения из **обоих** каналов.

#### Шаг 3.2: закрытие out

```go
go func() {
	wg.Wait()
	close(out)
}()
```

**Что делает:**

- Отдельная горутина ждёт `wg.Wait()`.
- Когда **все** горутины-читатели завершились — `close(out)`.

**Почему отдельная горутина:**

- Если `wg.Wait()` и `close(out)` в main — main заблокируется на `Wait`.
- `out` не будет закрыт, пока main ждёт.
- Читатели `out` (если они в main) зависнут.

**Почему `close(out)` после `wg.Wait()`:**

- Пока хотя бы одна горутина пишет в `out`, `close` нельзя.
- `wg.Wait()` гарантирует, что **все** горутины завершились.
- Только после этого `close(out)` безопасен.

### Шаг 4: полный main

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	ch1 := makeChan([]int{1, 2, 3, 4, 5})
	ch2 := makeChan([]int{6, 7, 8, 9, 10})

	out := fanIn(ch1, ch2)

	for v := range out {
		fmt.Println(v)
	}
}

func makeChan(data []int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for _, v := range data {
			out <- v
		}
	}()

	return out
}

func fanIn(channels ...<-chan int) chan int {
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

**Пример вывода:**

```
1
6
2
7
3
8
4
9
5
10
```

**Что видно:**

- Значения из `ch1` и `ch2` **перемешаны**.
- Порядок **не гарантирован** — зависит от скорости каналов.
- Все 10 значений прочитаны.

### Шаг 5: fan-in с отменой через context

Добавим `context` для отмены:

```go
func fanInCtx(ctx context.Context, channels ...<-chan int) chan int {
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
```

**Что изменилось:** читатели проверяют `ctx.Done()` при отправке. Если контекст отменён — завершаются.

### Шаг 6: полный main с context

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	ch1 := makeChan(ctx, []int{1, 2, 3, 4, 5})
	ch2 := makeChan(ctx, []int{6, 7, 8, 9, 10})

	out := fanIn(ctx, ch1, ch2)

	for v := range out {
		fmt.Println(v)
	}
}

func makeChan(ctx context.Context, data []int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for _, v := range data {
			select {
			case out <- v:
			case <-ctx.Done():
				return
			}
		}
	}()

	return out
}

func fanIn(ctx context.Context, channels ...<-chan int) chan int {
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
```

### Порядок результатов

**Порядок не гарантирован.** Значения из разных каналов приходят в порядке готовности. Если важен порядок — нужно дополнительное упорядочивание (например, по ID).

### Альтернатива: `reflect.Select`

**`reflect.Select`** позволяет динамически выбирать из N каналов:

```go
func fanInReflect(channels []<-chan int) chan int {
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
			out <- v.Interface().(int)
		}
	}()

	return out
}
```

**Плюсы:**

- **Одна горутина**, а не N.
- **Меньше памяти.**

**Минусы:**

- **`reflect` медленнее** (~10x).
- **Сложнее код.**

**Рекомендация:** для N < 100 — простая реализация с N горутинами. Для N > 1000 — `reflect.Select`.

### Сравнение реализаций

| Реализация | Горутин | Time (per element) | Сложность |
|:---|:---|:---|:---|
| N горутин | N + 1 | ~50-100 нс | Простая |
| `reflect.Select` | 1 | ~500-1000 нс | Сложная |

### Аннотация сложности

| Аспект | Значение |
|:---|:---|
| Горутин | N + 1 |
| Память | (N + 1) × 2.3 КБ |
| Пропускная способность | sum(rate_channels) |

### 💡 Практика: как использовать fan-in

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Для каждого канала — своя горутина.**
2. **`ch` передавай как аргумент**, не замыкание.
3. **`wg.Wait()` + `close(out)` в отдельной горутине.**
4. **`select` с `ctx.Done()`** для отмены.

**👍 СТОИТ СДЕЛАТЬ:**

5. **`reflect.Select`** — если N > 1000.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Буферизованный `out`** — для снижения contention.

**❌ НЕ ДЕЛАЙ:**

7. **Не забывай `close(out)`.** Иначе collector зависнет.
8. **Не используй `Mutex` для слияния** — канал проще и безопаснее.

### Ключевые выводы подглавы 8.2

- **Fan-in** — слияние N каналов в один.
- **N горутин + 1** для `close`.
- **`ch` передавай как аргумент**, не замыкание.
- **Порядок не гарантирован.**
- **`reflect.Select`** — для больших N.
- **`context`** — для отмены.

---

## 8.3 Fan-out + Fan-in: полная схема

Соберём **полную схему** fan-out + fan-in.

### Шаг 1: generator

```go
func generator(data []int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for _, v := range data {
			out <- v
		}
	}()

	return out
}
```

### Шаг 2: fan-out

```go
func fanOut(input <-chan int, n int) []chan int {
	outputs := make([]chan int, n)
	for i := 0; i < n; i++ {
		outputs[i] = worker(input)
	}
	return outputs
}

func worker(input <-chan int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for v := range input {
			out <- process(v)
		}
	}()

	return out
}

func process(v int) int {
	return v * 2
}
```

### Шаг 3: fan-in

```go
func fanIn(channels ...<-chan int) chan int {
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

### Шаг 4: полный main

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	data := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	input := generator(data)

	const numWorkers = 3
	outputs := fanOut(input, numWorkers)
	merged := fanIn(outputs...)

	for v := range merged {
		fmt.Println(v)
	}
	fmt.Println("done")
}

func generator(data []int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for _, v := range data {
			out <- v
		}
	}()

	return out
}

func fanOut(input <-chan int, n int) []chan int {
	outputs := make([]chan int, n)
	for i := 0; i < n; i++ {
		outputs[i] = worker(input)
	}
	return outputs
}

func worker(input <-chan int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for v := range input {
			out <- process(v)
		}
	}()

	return out
}

func process(v int) int {
	return v * 2
}

func fanIn(channels ...<-chan int) chan int {
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

**Что происходит:**

```
t=0:    generator пишет в input
        3 воркера читают из input
        Каждый пишет в свой output
        fanIn: 3 горутины читают из outputs
        fanIn: пишет в merged
        main: читает из merged

t=T:    generator закрыл input
        Воркеры дочитали остатки, закрыли свои outputs
        fanIn-горутины видят close, завершаются
        wg.Wait() вернулся, close(merged)
        main видит close, завершается
```

### Сколько горутин

| Компонент | Горутин |
|:---|:---|
| generator | 1 |
| Воркеры (fan-out) | N |
| fan-in читатели | N |
| fan-in close | 1 |
| main | 1 |
| **Итого** | **2N + 3** |

**Для N = 3:** 9 горутин.

### Альтернатива: fan-out с общим каналом

**Fan-out + fan-in** можно упростить, если воркеры пишут в **один** канал результатов:

```go
func fanOutShared(input <-chan int, n int) chan int {
	out := make(chan int)

	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for v := range input {
				out <- process(v)
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

**Полный main:**

```go
func main() {
	data := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	input := generator(data)
	out := fanOutShared(input, 3)

	for v := range out {
		fmt.Println(v)
	}
}
```

**Что изменилось:** вместо N каналов + fan-in — **один** канал. Горутин: N + 2 вместо 2N + 3.

### Сравнение

| Подход | Горутин (N=3) | Сложность | Гибкость |
|:---|:---|:---|:---|
| Fan-out A + fan-in | 9 | Средняя | Высокая |
| Fan-out B (общий) | 5 | Низкая | Низкая |

### 💡 Практика: какую схему выбрать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Fan-out B (общий канал)** — для большинства случаев.
2. **Fan-out A + fan-in** — если нужна гибкость.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Буферизованные каналы** для снижения contention.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

4. **`reflect.Select`** — для больших N.

**❌ НЕ ДЕЛАЙ:**

5. **Не создавай N + 1 каналов**, если можно обойтись одним.

### Ключевые выводы подглавы 8.3

- **Fan-out + fan-in** — классическая комбинация.
- **Горутин:** 2N + 3 для fan-out A + fan-in, N + 2 для fan-out B.
- **Fan-out B** — проще и быстрее.
- **Fan-out A + fan-in** — для гибкости.

---

## 8.4 Pipeline: многостадийная обработка

**Pipeline** — это **цепочка стадий**, где каждая стадия:

1. Читает из входного канала.
2. Обрабатывает.
3. Пишет в выходной канал.

### Схема

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Source  │──▶│  Stage 1 │──▶│  Stage 2 │──▶│  Stage 3 │──▶ Sink
└──────────┘   └──────────┘   └──────────┘   └──────────┘
   (Kafka)      (parse)        (enrich)      (write)
```

**Каждая стадия — это функция**, которая принимает входной канал и возвращает выходной.

### Шаг 1: простая стадия

**Стадия** — функция с сигнатурой:

```go
func stage(ctx context.Context, input <-chan int) chan int
```

**Пример: parseStage**

```go
func parseStage(ctx context.Context, input <-chan int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for v := range input {
			parsed := parse(v)
			select {
			case out <- parsed:
			case <-ctx.Done():
				return
			}
		}
	}()

	return out
}

func parse(v int) int {
	return v * 2
}
```

**Что делает:**

1. Создаёт выходной канал.
2. Запускает горутину.
3. Читает из входа, парсит, пишет в выход.
4. При `ctx.Done()` — завершается.
5. При закрытии входа — `close(out)`.

### Шаг 2: стадия с fan-out

**Стадия, которая делает fan-out:**

```go
func enrichStage(ctx context.Context, input <-chan int, n int) chan int {
	out := make(chan int)

	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for v := range input {
				enriched := enrich(ctx, v)
				select {
				case out <- enriched:
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

func enrich(ctx context.Context, v int) int {
	// имитация запроса в БД
	time.Sleep(1 * time.Millisecond)
	return v + 100
}
```

**Что делает:**

1. N воркеров читают из **общего** входа.
2. Каждый обогащает значение.
3. Пишут в **общий** выход.
4. `wg.Wait()` + `close(out)`.

### Шаг 3: сборка pipeline

```go
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	source := generator([]int{1, 2, 3, 4, 5})
	parsed := parseStage(ctx, source)
	enriched := enrichStage(ctx, parsed, 10)

	for v := range enriched {
		fmt.Println(v)
	}
}
```

**Что происходит:**

- generator → parseStage → enrichStage → main.
- Каждая стадия — отдельная горутина (или N горутин).
- Backpressure автоматически.

### Шаг 4: полный код

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	data := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

	source := generator(ctx, data)
	parsed := parseStage(ctx, source)
	enriched := enrichStage(ctx, parsed, 5)

	for v := range enriched {
		fmt.Println(v)
	}
	fmt.Println("done")
}

func generator(ctx context.Context, data []int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for _, v := range data {
			select {
			case out <- v:
			case <-ctx.Done():
				return
			}
		}
	}()

	return out
}

func parseStage(ctx context.Context, input <-chan int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for v := range input {
			parsed := parse(v)
			select {
			case out <- parsed:
			case <-ctx.Done():
				return
			}
		}
	}()

	return out
}

func parse(v int) int {
	return v * 2
}

func enrichStage(ctx context.Context, input <-chan int, n int) chan int {
	out := make(chan int)

	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for v := range input {
				enriched := enrich(ctx, v)
				select {
				case out <- enriched:
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

func enrich(ctx context.Context, v int) int {
	time.Sleep(1 * time.Millisecond)
	return v + 100
}
```

### Визуализация pipeline

```
generator → source
              │
              ▼
         parseStage → parsed
              │
              ▼
         enrichStage (5 воркеров) → enriched
              │
              ▼
            main (collector)
```

### Backpressure между стадиями

**Каждая стадия — отдельная горутина.** Если стадия медленная — её входной канал заполняется. Предыдущая стадия **блокируется** на `out <- msg`. Это **backpressure**.

```
Source (быстро) → parsed (заполнен) → enrich (медленно)
                                     ↑
                              backpressure
```

### Сравнение с последовательной обработкой

**Последовательно:**

```
10 сообщений × (1 мкс parse + 1 мс enrich) = 10.1 мс
```

**Pipeline:**

```
10 сообщений / 5 воркеров × 1 мс = 2 мс
```

**В 5 раз быстрее.**

### Аннотация сложности

| Аспект | Последовательно | Pipeline |
|:---|:---|:---|
| Пропускная способность | 1 / sum(times) | N / max(time) |
| Latency (per message) | sum(times) | sum(times) |
| Горутин | 1 | N + 2 |

### 💡 Практика: как строить pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Каждая стадия — функция** `func(ctx, input) chan T`.
2. **`defer close(out)`** в каждой стадии.
3. **`select` с `ctx.Done()`** для отмены.
4. **Fan-out для медленных стадий.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Буферизованные каналы** между стадиями.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Динамическое N** — для переменной нагрузки.

**❌ НЕ ДЕЛАЙ:**

7. **Не делай стадии слишком мелкими.** Overhead на каналы.
8. **Не забывай `close`.** Утечка горутин.

### Ключевые выводы подглавы 8.4

- **Pipeline** — цепочка стадий, каждая читает из входа и пишет в выход.
- **Стадия — функция** `func(ctx, input) chan T`.
- **Fan-out** для медленных стадий.
- **Backpressure** между стадиями автоматически.
- **В N раз быстрее** последовательной обработки.

---

## 8.5 Backpressure в pipeline

Разберём **backpressure** между стадиями.

### Как работает backpressure

**Backpressure** — это когда **медленная стадия замедляет быструю**.

```
Source (быстро) → parsed (буфер 100) → enrich (медленно)
```

**Что происходит:**

1. Source пишет в `parsed` быстро.
2. `parsed` заполняется (100 элементов).
3. Source **блокируется** на `parsed <- msg`.
4. Source ждёт, пока enrich не заберёт из `parsed`.
5. Enrich обрабатывает медленно → `parsed` остаётся полным.
6. Source продолжает ждать.

**Результат:** Source работает со скоростью enrich.

### Размер буфера и backpressure

**Маленький буфер (1-10):**

- Backpressure **быстрый**.
- Source блокируется часто.
- Меньше памяти.

**Большой буфер (1000+):**

- Backpressure **отложенный**.
- Source блокируется редко.
- Больше памяти.

**Очень большой буфер (100 000):**

- Backpressure **не работает**.
- Source пишет всё в буфер.
- Память растёт.

### Визуализация

```
Маленький буфер (10):

  Source: ████████░░░░░░░░░░░░░░░░ (блокируется)
  Enrich: ████░░░░░░░░░░░░░░░░░░░░ (медленно)
  
  Source работает со скоростью Enrich.

Большой буфер (1000):

  Source: ████████████████████████ (не блокируется)
  Enrich: ████░░░░░░░░░░░░░░░░░░░░ (медленно)
  
  Буфер заполняется, память растёт.

Очень большой буфер (100 000):

  Source: ████████████████████████
  Enrich: ████░░░░░░░░░░░░░░░░░░░░
  
  OOM через минуту.
```

### Как выбрать размер буфера

**Правило:** буфер должен **сглаживать пики**, но **не скрывать** проблему.

**Рекомендации:**

- **1-10× пропускная способность стадии.** Если стадия обрабатывает 1000/сек, буфер 100-10 000.
- **Не больше 10 000.** Иначе backpressure не работает.
- **Мониторь длину буфера.** Если постоянно полный — стадия медленная.

### Альтернатива: `select` с `default`

**Без блокировки:**

```go
select {
case out <- msg:
case <-ctx.Done():
    return
default:
    // буфер полон — пропускаем или логируем
    metrics.Dropped.Add(1)
}
```

**Что делает:** если буфер полон, сообщение **пропускается**. Это **не backpressure**, а **drop**.

**Когда использовать:** если сообщения **можно терять** (метрики, логи).

### Аннотация сложности

| Размер буфера | Backpressure | Память | Когда |
|:---|:---|:---|:---|
| 1 | Мгновенный | Минимум | Синхронная обработка |
| 10-100 | Быстрый | Мало | Большинство случаев |
| 1000-10000 | Отложенный | Средне | Пики нагрузки |
| 100000+ | Не работает | Много | ❌ Не использовать |

### 💡 Практика: как настроить backpressure

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Буфер 10-1000** для большинства случаев.
2. **Мониторь длину буфера.**
3. **Backpressure** — для критичных данных.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`select` с `default`** — для некритичных данных.
5. **Drop метрики** — сколько потеряно.

**❌ НЕ ДЕЛАЙ:**

6. **Не ставь буфер 100 000+.** Backpressure не работает.
7. **Не игнорируй заполненный буфер.** Это сигнал.

### Ключевые выводы подглавы 8.5

- **Backpressure** — медленная стадия замедляет быструю.
- **Размер буфера** определяет, как быстро работает backpressure.
- **Маленький буфер** — быстрый backpressure. **Большой** — отложенный.
- **`select` с `default`** — drop вместо backpressure.
- **Мониторь длину буфера.**

---

## 8.6 Закрытие каналов в pipeline

Разберём **правильное закрытие** каналов в pipeline.

### Правило: кто пишет — тот закрывает

**В pipeline** каждая стадия:

- **Читает** из входного канала.
- **Пишет** в выходной канал.
- **Закрывает** свой **выходной** канал.

**Не закрывает входной** — это делает предыдущая стадия.

### Пример

```go
func stage(ctx context.Context, input <-chan int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)  // ← закрываем СВОЙ выход
		for v := range input {  // ← читаем из чужого входа
			select {
			case out <- process(v):
			case <-ctx.Done():
				return
			}
		}
	}()

	return out
}
```

**Что происходит:**

1. Стадия читает из `input`.
2. Когда `input` закрыт — `for range` завершается.
3. `defer close(out)` закрывает выход.
4. Следующая стадия видит `close` и завершается.

### Каскадное закрытие

```
Source закрывает source
    ↓
Stage 1: for range source завершается → close(stage1Out)
    ↓
Stage 2: for range stage1Out завершается → close(stage2Out)
    ↓
Stage 3: for range stage2Out завершается → close(stage3Out)
    ↓
Sink: for range stage3Out завершается
```

**Ключевое:** закрытие **каскадное**. Каждая стадия закрывает свой выход, что триггерит закрытие следующей.

### Проблема: fan-out

**Fan-out:** N воркеров читают из **одного** входа. Кто закроет **выходы** воркеров?

**Решение:**

- Каждый воркер закрывает **свой** выход (`defer close(workerOut)`).
- Если воркеры пишут в **общий** выход — `wg.Wait()` + `close(out)`.

```go
func enrichStage(ctx context.Context, input <-chan int, n int) chan int {
	out := make(chan int)

	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for v := range input {
				select {
				case out <- enrich(ctx, v):
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

### Проблема: забыть close

**❌ Плохо:**

```go
func stage(input <-chan int) chan int {
	out := make(chan int)

	go func() {
		for v := range input {
			out <- process(v)
		}
		// забыли close(out)
	}()

	return out
}
```

**Что происходит:** следующая стадия (`for range out`) **никогда не завершится**. **Утечка горутин.**

### Проблема: закрыть вход

**❌ Плохо:**

```go
func stage(input <-chan int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		defer close(input)  // ← НЕЛЬЗЯ!
		for v := range input {
			out <- process(v)
		}
	}()

	return out
}
```

**Что происходит:** `close(input)` закроет канал, который **читает другая стадия**. Паника при отправке в закрытый канал.

### Проблема: двойное закрытие

**❌ Плохо:**

```go
go func() {
	defer close(out)
	for v := range input {
		out <- process(v)
	}
	close(out)  // ← паника: close of closed channel
}()
```

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| `close(ch)` | ~50-100 нс | 0 |
| `for range ch` после close | ~50-100 нс на элемент | 0 |
| `wg.Wait()` | ~10-20 нс + ожидание | 0 |

### 💡 Практика: как закрывать каналы в pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Каждая стадия закрывает СВОЙ выход** (`defer close(out)`).
2. **НЕ закрывай входной канал.**
3. **Fan-out:** `wg.Wait()` + `close(out)`.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`defer close(out)`** сразу после `make(chan)`.

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай `close(out)`.** Утечка.
6. **Не закрывай входной канал.**
7. **Не закрывай канал дважды.** Паника.

### Ключевые выводы подглавы 8.6

- **Кто пишет — тот закрывает.** Каждая стадия закрывает **свой** выход.
- **Не закрывай входной канал.**
- **Каскадное закрытие.**
- **Fan-out:** `wg.Wait()` + `close(out)`.
- **Забыть close** → утечка. **Двойное close** → паника.

---

## 8.7 Отмена pipeline через context

Разберём **отмену** pipeline через `context.Context`.

### Паттерн: `context` во всех стадиях

```go
func stage(ctx context.Context, input <-chan int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for {
			select {
			case <-ctx.Done():
				return
			case v, ok := <-input:
				if !ok {
					return
				}
				select {
				case out <- process(v):
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

1. Проверка `ctx.Done()` в **двух** местах: при чтении из входа и при записи в выход.
2. Если `ctx` отменён — стадия завершается.
3. `defer close(out)` закрывает выход.

### Каскадная отмена

```
ctx отменён
    ↓
Source: select case <-ctx.Done() → return
    ↓
Stage 1: select case <-ctx.Done() → return → close(stage1Out)
    ↓
Stage 2: select case <-ctx.Done() → return → close(stage2Out)
    ↓
Sink: select case <-ctx.Done() → return
```

**Ключевое:** `ctx.Done()` — **broadcast**. Все стадии видят отмену **одновременно**.

### Полный пример с отменой

```go
func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	source := generator(ctx, []int{1, 2, 3, 4, 5})
	parsed := parseStage(ctx, source)
	enriched := enrichStage(ctx, parsed, 5)

	for range enriched {
		// ждём завершения
	}

	if ctx.Err() != nil {
		fmt.Println("pipeline canceled:", ctx.Err())
	}
}
```

### `errgroup` для pipeline

```go
g, ctx := errgroup.WithContext(context.Background())

g.Go(func() error { return runSource(ctx, ...) })
g.Go(func() error { return runParse(ctx, ...) })
g.Go(func() error { return runEnrich(ctx, ...) })
g.Go(func() error { return runWrite(ctx, ...) })

if err := g.Wait(); err != nil {
	fmt.Println("error:", err)
}
```

**Что делает `errgroup`:**

- Запускает N горутин.
- При **первой** ошибке — отменяет `ctx`.
- `g.Wait()` возвращает первую ошибку.

### Аннотация сложности

| Подход | Отмена | Ошибок | Сложность |
|:---|:---|:---|:---|
| `context` + `cancel` | ✅ | Первая | Средняя |
| `errgroup` | ✅ | Первая | Низкая |
| Ручной `errCh` | ✅ | Первая | Средняя |

### 💡 Практика: как отменять pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`context.Context`** во всех стадиях.
2. **`select` с `ctx.Done()`** — при чтении и записи.
3. **`errgroup`** — для обработки ошибок.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`context.WithTimeout`** — для ограничения времени.

**❌ НЕ ДЕЛАЙ:**

5. **Не игнорируй `ctx.Done()`.** Отмена не сработает.
6. **Не забывай `cancel()`.** Утечка.

### Ключевые выводы подглавы 8.7

- **`context.Context`** — для отмены pipeline.
- **`select` с `ctx.Done()`** — при чтении и записи.
- **Каскадная отмена:** `ctx.Done()` — broadcast.
- **`errgroup`** — для обработки ошибок.

---

## 8.8 Tee-канал и bridge-канал

Разберём **tee-канал** и **bridge-канал**.

### Tee-канал: разветвление

**Tee-канал** — это канал, который **дублирует** данные в два канала.

**Схема:**

```
                ┌──────────┐
           ┌───▶│ Channel1 │
┌──────┐   │    └──────────┘
│Input │───┤
└──────┘   │    ┌──────────┐
           └───▶│ Channel2 │
                └──────────┘
```

**Реализация:**

```go
func tee(ctx context.Context, input <-chan int) (chan int, chan int) {
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

**Что делает:**

1. Читает из `input`.
2. Отправляет сообщение в `out1`.
3. Отправляет **то же** сообщение в `out2`.
4. Закрывает оба канала при завершении.

**Ключевое:** каждое сообщение **дублируется**.

### Bridge-канал: склейка

**Bridge-канал** — это канал, который **разворачивает** канал каналов в один канал.

**Реализация:**

```go
func bridge(ctx context.Context, chanCh <-chan chan int) chan int {
	out := make(chan int)

	go func() {
		defer close(out)
		for {
			var ch chan int
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

**Что делает:**

1. Читает из `chanCh` (канал каналов).
2. Для каждого вложенного канала — читает из него и пишет в `out`.
3. Когда `chanCh` закрыт — завершается.

### Сравнение tee и bridge

| Паттерн | Что делает | Использование |
|:---|:---|:---|
| **Tee** | Дублирует в 2 канала | Логирование + обработка |
| **Bridge** | Разворачивает `chan chan T` | Динамический fan-in |

### Аннотация сложности

| Паттерн | Горутин | Time (per element) |
|:---|:---|:---|
| Tee | 1 | ~100-200 нс |
| Bridge | 1 | ~50-100 нс |

### 💡 Практика: как использовать tee и bridge

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Tee** — для дублирования в 2 канала.
2. **Bridge** — для динамического fan-in.

**👍 СТОИТ СДЕЛАТЬ:**

3. **`context` для отмены** в tee и bridge.

**❌ НЕ ДЕЛАЙ:**

4. **Не путай tee с fan-out.** Tee дублирует, fan-out распределяет.
5. **Не путай bridge с merge.** Bridge разворачивает `chan chan T`, merge сливает N каналов.

### Ключевые выводы подглавы 8.8

- **Tee** — дублирование в 2 канала.
- **Bridge** — разворачивание `chan chan T` в `chan T`.
- **Tee** — для логирования + обработки.
- **Bridge** — для динамического fan-in.

---

## 8.9 Pipeline vs worker pool: что выбрать

Разберём **когда что использовать**.

### Worker pool

**Worker pool** — N воркеров обрабатывают **однотипные** задачи.

**Когда использовать:**

- **Однотипная обработка.**
- **Одна стадия.**
- **Простота важна.**

### Pipeline

**Pipeline** — цепочка стадий, каждая обрабатывает **по-своему**.

**Когда использовать:**

- **Разные стадии.** Парсинг → обогащение → запись.
- **Разные N на стадиях.**
- **Backpressure между стадиями.**

### Гибрид: pipeline из worker pool'ов

**Каждая стадия pipeline** — это worker pool:

```go
func stage(ctx context.Context, input <-chan int, n int) chan int {
	out := make(chan int)

	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for v := range input {
				select {
				case out <- process(ctx, v):
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

### Сравнение

| Аспект | Worker pool | Pipeline |
|:---|:---|:---|
| Стадий | 1 | N |
| Тип задач | Однотипные | Разные |
| N | Фиксировано | Разное на стадиях |
| Backpressure | Внутри pool | Между стадиями |
| Сложность | Низкая | Средняя |

### 💡 Практика: как выбирать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Worker pool** — для однотипной обработки.
2. **Pipeline** — для разных стадий.
3. **Гибрид** — pipeline из worker pool'ов.

**❌ НЕ ДЕЛАЙ:**

4. **Не используй pipeline для одной стадии.** Worker pool проще.
5. **Не используй worker pool для разных стадий.** Pipeline лучше.

### Ключевые выводы подглавы 8.9

- **Worker pool** — однотипная обработка, одна стадия.
- **Pipeline** — разные стадии, разное N.
- **Гибрид** — pipeline из worker pool'ов.

---

## 8.10 Практика Go: pipeline с метриками

Напишем **pipeline с метриками**.

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
	SourceCount   atomic.Int64
	ParsedCount   atomic.Int64
	EnrichedCount atomic.Int64

	SourceDuration   atomic.Int64
	ParseDuration    atomic.Int64
	EnrichDuration   atomic.Int64
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	var metrics Metrics

	data := make([]int, 1000)
	for i := range data {
		data[i] = i
	}

	source := generator(ctx, data, &metrics)
	parsed := parseStage(ctx, source, &metrics)
	enriched := enrichStage(ctx, parsed, 10, &metrics)

	for range enriched {
		// ждём
	}

	fmt.Printf("Source:   %d\n", metrics.SourceCount.Load())
	fmt.Printf("Parsed:   %d\n", metrics.ParsedCount.Load())
	fmt.Printf("Enriched: %d\n", metrics.EnrichedCount.Load())
	fmt.Printf("Source avg:  %v\n", avgDuration(&metrics.SourceDuration, &metrics.SourceCount))
	fmt.Printf("Parse avg:   %v\n", avgDuration(&metrics.ParseDuration, &metrics.ParsedCount))
	fmt.Printf("Enrich avg:  %v\n", avgDuration(&metrics.EnrichDuration, &metrics.EnrichedCount))
}

func avgDuration(total, count *atomic.Int64) time.Duration {
	c := count.Load()
	if c == 0 {
		return 0
	}
	return time.Duration(total.Load() / c)
}

func generator(ctx context.Context, data []int, metrics *Metrics) chan int {
	out := make(chan int, 100)

	go func() {
		defer close(out)
		for _, v := range data {
			start := time.Now()
			select {
			case out <- v:
				metrics.SourceCount.Add(1)
				metrics.SourceDuration.Add(int64(time.Since(start)))
			case <-ctx.Done():
				return
			}
		}
	}()

	return out
}

func parseStage(ctx context.Context, input <-chan int, metrics *Metrics) chan int {
	out := make(chan int, 100)

	go func() {
		defer close(out)
		for v := range input {
			start := time.Now()
			parsed := parse(v)
			select {
			case out <- parsed:
				metrics.ParsedCount.Add(1)
				metrics.ParseDuration.Add(int64(time.Since(start)))
			case <-ctx.Done():
				return
			}
		}
	}()

	return out
}

func parse(v int) int {
	return v * 2
}

func enrichStage(ctx context.Context, input <-chan int, n int, metrics *Metrics) chan int {
	out := make(chan int)

	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for v := range input {
				start := time.Now()
				enriched := enrich(ctx, v)
				select {
				case out <- enriched:
					metrics.EnrichedCount.Add(1)
					metrics.EnrichDuration.Add(int64(time.Since(start)))
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

func enrich(ctx context.Context, v int) int {
	time.Sleep(1 * time.Millisecond)
	return v + 100
}
```

### Пример вывода

```
Source:   1000
Parsed:   1000
Enriched: 1000
Source avg:  1.2µs
Parse avg:   800ns
Enrich avg:  1.1ms
```

### Аннотация сложности

| Стадия | Time (per msg) | Пропускная способность |
|:---|:---|:---|
| Source | 1.2 µs | 833 000/сек |
| Parse | 800 нс | 1 250 000/сек |
| Enrich | 1.1 мс × 10 | 9 000/сек |

### 💡 Практика: как измерять pipeline

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики на каждой стадии:** count, duration.
2. **`atomic.Int64`** для метрик.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Экспорт в Prometheus** (для production).

**❌ НЕ ДЕЛАЙ:**

4. **Не используй `Mutex` для метрик** — `atomic` быстрее.

### Ключевые выводы подглавы 8.10

- **Метрики на каждой стадии:** count, duration.
- **`atomic.Int64`** для счётчиков.
- **Bottleneck** видно по метрикам.

---

## 8.11 Выводы и типичные ошибки

**Что мы узнали?**

Fan-out — распределение работы от одного источника к N обработчикам. **Две реализации:** A (N каналов) и B (один общий канал). Fan-in — слияние N каналов в один. Fan-out + fan-in — классическая комбинация. Pipeline — цепочка стадий. Backpressure — медленная стадия замедляет быструю. Закрытие: кто пишет — тот закрывает. Отмена через `context`. Tee — дублирование. Bridge — разворачивание `chan chan T`. Worker pool для однотипной, pipeline для разных стадий. Гибрид — pipeline из worker pool'ов.

**Типичные ошибки:**

- ❌ **Использовать `i--` при `!ok`.** Busy loop. Используй `received` + `ch = nil`.
- ❌ **Забыть `close(out)` в стадии.** Утечка горутин.
- ❌ **Закрыть входной канал.** Паника при отправке.
- ❌ **Двойное закрытие.** Паника.
- ❌ **Не использовать `select` с `ctx.Done()`.** Отмена не сработает.
- ❌ **Большие буферы.** Backpressure не работает.
- ❌ **Забыть `wg.Wait()` в fan-in.** `close(out)` не выполнится.
- ❌ **Путать fan-out и fan-in.**
- ❌ **Путать tee и fan-out.**
- ❌ **Путать bridge и merge.**
- ❌ **Передавать `ch` в замыкание**, а не аргумент. Все горутины читают из последнего.
- ❌ **Создавать N + 1 каналов, когда можно одним.**

---

## 8.12 Для быстрого повторения

- **Fan-out** — распределение работы от одного источника к N обработчикам.
- **Реализация A:** N каналов результатов. **Горутин:** 2N + 1.
- **Реализация B:** один общий канал. **Горутин:** N + 2.
- **Fan-in** — слияние N каналов в один. **Горутин:** N + 1.
- **Fan-out + fan-in** — классическая комбинация. **Горутин:** 2N + 3.
- **Fan-out B** — проще и быстрее для большинства случаев.
- **Отключение закрытых каналов через `ch = nil`**, не через `i--`.
- **Pipeline** — цепочка стадий. Каждая — `func(ctx, input) chan T`.
- **Backpressure** — медленная стадия замедляет быструю.
- **Маленький буфер** — быстрый backpressure. **Большой** — отложенный.
- **Закрытие:** кто пишет — тот закрывает. Каскадное.
- **Fan-out:** `wg.Wait()` + `close(out)`.
- **Отмена:** `context` + `select` с `ctx.Done()`.
- **Tee** — дублирование в 2 канала.
- **Bridge** — разворачивание `chan chan T`.
- **Worker pool vs pipeline:** pool для однотипной, pipeline для разных стадий.
- **Метрики:** count + duration на каждой стадии.

---

## 8.13 Вопросы для самопроверки

1. Что такое fan-out? Назови две реализации.
2. Чем реализация A отличается от B?
3. Что такое fan-in? Как реализуется?
4. Почему `ch` передаётся как аргумент, а не замыкание?
5. Почему `close(out)` в отдельной горутине?
6. Что такое pipeline? Как реализуется?
7. Что такое backpressure? Как размер буфера влияет на него?
8. Почему большой буфер маскирует проблему?
9. Что делает `select` с `default`?
10. Как отменить pipeline через `context`?
11. Что такое tee-канал?
12. Что такое bridge-канал?
13. Чем tee отличается от fan-out?
14. Чем bridge отличается от merge?
15. Когда worker pool, а когда pipeline?
16. Как исправить баг с `i--`?
17. Почему `ch = nil` отключает case в select?
18. Как измерять метрики pipeline?

---

## 8.14 Ответы

### Ответ 1

**Fan-out** — распределение работы от одного источника к N обработчикам.

**Две реализации:**
- **A:** N каналов результатов (каждый воркер пишет в свой).
- **B:** один общий канал результатов (все воркеры пишут в один).

### Ответ 2

**A:** N каналов, 2N горутин, гибко, высокий contention.

**B:** один канал, N + 2 горутин, просто, низкий contention.

### Ответ 3

**Fan-in** — слияние N каналов в один.

**Реализация:** N горутин + 1 для close.

```go
func fanIn(channels ...<-chan int) chan int {
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

### Ответ 4

**`ch` передаётся как аргумент**, потому что иначе все горутины захватят **последнее** значение `ch` из цикла. Все будут читать из **одного** канала.

```go
// ❌ Плохо:
for _, ch := range channels {
	go func() {
		for v := range ch {  // все читают из последнего ch
			...
		}
	}()
}

// ✅ Хорошо:
for _, ch := range channels {
	go func(c <-chan int) {  // копия ch
		for v := range c {
			...
		}
	}(ch)
}
```

### Ответ 5

**`close(out)` в отдельной горутине**, потому что:
- `wg.Wait()` блокируется.
- Если `close(out)` в main — main ждёт, `out` не закрыт.
- Читатели `out` (в main) зависнут.
- Отдельная горутина ждёт `wg.Wait()` и закрывает `out` **параллельно**.

### Ответ 6

**Pipeline** — цепочка стадий, каждая читает из входа и пишет в выход.

```go
source := generator(ctx, data)
parsed := parseStage(ctx, source)
enriched := enrichStage(ctx, parsed, 5)
```

### Ответ 7

**Backpressure** — медленная стадия замедляет быструю.

**Размер буфера:**
- Маленький (1-10) — быстрый backpressure.
- Большой (1000) — отложенный.
- 100 000+ — не работает.

### Ответ 8

**Большой буфер маскирует проблему**, потому что producer и воркеры не блокируются, пока буфер не полон. Но если producer быстрее collector надолго — буферы переполнятся.

### Ответ 9

**`select` с `default`** — если буфер полон, сообщение **пропускается** (drop). Для некритичных данных.

### Ответ 10

**Отмена pipeline через `context`:**

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

// В каждой стадии:
select {
case <-ctx.Done():
	return
case out <- process(v):
}
```

### Ответ 11

**Tee-канал** — дублирует данные в 2 канала.

```go
func tee(ctx context.Context, input <-chan int) (chan int, chan int) {
	out1 := make(chan int)
	out2 := make(chan int)
	go func() {
		defer close(out1)
		defer close(out2)
		for v := range input {
			out1 <- v
			out2 <- v
		}
	}()
	return out1, out2
}
```

### Ответ 12

**Bridge-канал** — разворачивает `chan chan T` в `chan T`.

### Ответ 13

**Tee** дублирует **каждое** сообщение в **оба** канала.

**Fan-out** распределяет сообщения **между** воркерами.

### Ответ 14

**Bridge** разворачивает `chan chan T` (канал каналов) в `chan T`.

**Merge** сливает **N каналов** в один.

### Ответ 15

**Worker pool** — для однотипной обработки, одна стадия.

**Pipeline** — для разных стадий, разное N.

### Ответ 16

**Баг с `i--`:** при `!ok` (канал закрыт) `<-ch1` возвращает `0, false` **немедленно**. Цикл крутится, `i--` компенсирует, но если все каналы закрыты — **бесконечный цикл**.

**Исправление:** считать `received`, отключать закрытые каналы через `ch = nil`.

### Ответ 17

**`ch = nil` отключает case в select**, потому что приём из nil-канала **блокируется навсегда**. `select` игнорирует case с nil-каналом — он никогда не готов.

### Ответ 18

**Метрики pipeline:**
- **Count** на каждой стадии.
- **Duration** на каждой стадии.
- **`atomic.Int64`** для счётчиков.
- **Bottleneck** видно по метрикам.

---

## 8.15 Куда идти дальше?

Мы разобрали pipeline: fan-out, fan-in, стадии, backpressure, отмена, tee, bridge. Теперь мы умеем строить конвейеры данных.

Но остаётся **практический вопрос**: как ограничить **скорость** обработки? Что если внешний сервис имеет rate limit? Что если нужно защититься от каскадных отказов?

- **Как ограничить скорость?** Rate limiter, token bucket, leaky bucket. → **Глава 9: Продвинутые паттерны — semaphore, rate limiter, circuit breaker.**
- **Как обрабатывать ошибки в конкурентном коде?** `errgroup`, `multierror`, отмена при первой ошибке. → **Глава 10: Обработка ошибок в конкурентном коде.**
- **Как корректно завершить сервис?** Сигналы ОС, `context`, ожидание завершения. → **Глава 11: Graceful shutdown.**

---

## 8.16 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Fan-out A** | N каналов результатов | N воркеров + N читателей. 2N + 1 горутин |
| **Fan-out B** | Один общий канал | N воркеров + 1 close. N + 2 горутин |
| **Fan-in** | Слияние N каналов | N горутин + 1 для close |
| **Fan-out + fan-in** | Классическая комбинация | 2N + 3 горутин |
| **`i--` при `!ok`** | Баг: busy loop | Используй `received` + `ch = nil` |
| **`ch = nil`** | Отключение case | Приём из nil блокируется навсегда |
| **Pipeline** | Цепочка стадий | Каждая — `func(ctx, input) chan T` |
| **Backpressure** | Замедление производителя | Размер буфера определяет скорость |
| **Маленький буфер** | 1-10 | Быстрый backpressure |
| **Большой буфер** | 1000+ | Отложенный backpressure |
| **`select` с `default`** | Drop | Для некритичных данных |
| **Закрытие** | Кто пишет — тот закрывает | Каскадное |
| **Отмена** | `context` + `select` с `ctx.Done()` | Broadcast |
| **`errgroup`** | Обработка ошибок | Отменяет при первой ошибке |
| **Tee-канал** | Дублирование | В 2 канала |
| **Bridge-канал** | Разворачивание | `chan chan T` → `chan T` |
| **Worker pool vs pipeline** | Однотипное vs разное | Pool для однотипной, pipeline для разных стадий |
| **Гибрид** | Pipeline из worker pool'ов | Разное N на стадиях |
| **Метрики** | Count + duration | `atomic.Int64` |

🔀 **Ключевая идея:** Fan-out — распределение работы от одного источника к N обработчикам. **Две реализации:** A (N каналов, гибко) и B (один общий, просто). Fan-in — слияние N каналов в один; N горутин + 1 для close. Fan-out + fan-in — классическая комбинация (2N + 3 горутин). Pipeline — цепочка стадий; каждая — `func(ctx, input) chan T`. Backpressure — медленная стадия замедляет быструю; размер буфера определяет скорость. Закрытие: кто пишет — тот закрывает; каскадное. Отмена через `context` + `select` с `ctx.Done()`. Tee дублирует, bridge разворачивает. Worker pool для однотипной обработки, pipeline для разных стадий. **Не используй `i--` при `!ok`** — это busy loop. Используй `received` + `ch = nil`.