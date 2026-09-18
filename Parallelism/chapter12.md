# 🧪 Глава 12: Тестирование и профилирование конкурентного кода

**Что вы узнаете:**
- Почему конкурентный код **сложно тестировать**.
- Что такое **race detector** и как его использовать.
- Как писать **stress-тесты** для поиска гонок.
- Что такое **`go test -race`** и когда он не помогает.
- Как тестировать **таймауты** и **отмену**.
- Что такое **`testing.T.Parallel()`** и когда он ломает тесты.
- Как использовать **pprof** для CPU, памяти, горутин.
- Что такое **block profile** и **mutex profile**.
- Как читать **goroutine dump**.
- Как использовать **`runtime/trace`**.
- Как писать **бенчмарки** конкурентного кода.
- Как использовать **`goleak`** для поиска утечек горутин.

**После прочтения вы сможете:**
- Запускать `go test -race` в CI.
- Писать stress-тесты для конкурентного кода.
- Тестировать `context.Canceled` и таймауты.
- Диагностировать утечки горутин через pprof.
- Читать goroutine dump.
- Профилировать CPU и память конкурентного кода.
- Писать бенчмарки с `b.RunParallel`.
- Использовать `goleak` в тестах.
- Понимать, когда тесты **не находят** гонки.

---

## Содержание

- [12.0 Пролог: тест, который прошёл на ноутбуке](#120-пролог-тест-который-прошёл-на-ноутбуке)
- [12.1 Почему конкурентный код сложно тестировать](#121-почему-конкурентный-код-сложно-тестировать)
- [12.2 Race detector: go test -race](#122-race-detector-go-test--race)
- [12.3 Stress-тесты](#123-stress-тесты)
- [12.4 Тестирование таймаутов и отмены](#124-тестирование-таймаутов-и-отмены)
- [12.5 testing.T.Parallel()](#125-testingtparallel)
- [12.6 pprof: CPU, memory, goroutine](#126-pprof-cpu-memory-goroutine)
- [12.7 Block profile и mutex profile](#127-block-profile-и-mutex-profile)
- [12.8 Goroutine dump](#128-goroutine-dump)
- [12.9 runtime/trace](#129-runtimetrace)
- [12.10 Бенчмарки конкурентного кода](#1210-бенчмарки-конкурентного-кода)
- [12.11 goleak: поиск утечек](#1211-goleak-поиск-утечек)
- [12.12 Практика Go: тестирование worker pool](#1212-практика-go-тестирование-worker-pool)
- [12.13 Выводы и типичные ошибки](#1213-выводы-и-типичные-ошибки)
- [12.14 Для быстрого повторения](#1214-для-быстрого-повторения)
- [12.15 Вопросы для самопроверки](#1215-вопросы-для-самопроверки)
- [12.16 Ответы](#1216-ответы)
- [12.17 Куда идти дальше?](#1217-куда-идти-дальше)
- [12.18 Чек-лист](#1218-чек-лист)

---

## 12.0 Пролог: тест, который прошёл на ноутбуке

Ты пишешь worker pool (Глава 7). Покрываешь тестами:

```go
func TestWorkerPool(t *testing.T) {
    tasks := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    results := make([]int, len(tasks))

    var wg sync.WaitGroup
    tasksCh := make(chan int, len(tasks))

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for task := range tasksCh {
                results[task-1] = task * 2  // ← Гонка!
            }
        }()
    }

    for _, task := range tasks {
        tasksCh <- task
    }
    close(tasksCh)
    wg.Wait()

    for i, r := range results {
        if r != (i+1)*2 {
            t.Errorf("results[%d] = %d, want %d", i, r, (i+1)*2)
        }
    }
}
```

**Тест проходит.** На твоём ноутбуке (x86, 8 ядер).

**На CI (ARM, 4 ядра) — тест падает.** Или хуже — **иногда** падает.

❓ **Что произошло?** В тесте **data race** — горутины пишут в `results` без синхронизации. Но на x86 гонка «маскируется» (Глава 4), а на ARM проявляется.

💡 **Решение:** **race detector**. `go test -race` найдёт гонку **на любой архитектуре**.

```bash
go test -race ./...
```

**Вывод:**

```
==================
WARNING: DATA RACE
Write at 0x00c0000b4030 by goroutine 8:
  main.TestWorkerPool.func1()
      /path/to/main_test.go:15 +0x...

Previous write at 0x00c0000b4030 by goroutine 7:
  main.TestWorkerPool.func1()
      /path/to/main_test.go:15 +0x...
==================
```

**Что видно:** две горутины пишут в один адрес. Гонка найдена.

> **Важный мост к будущим главам:** тестирование — основа production. Race detector, pprof, trace — инструменты, без которых **нельзя** выпускать конкурентный код. Глава 13 (анти-паттерны) — что искать. Глава 14 (реальные сценарии) — как применять.

---

## 12.1 Почему конкурентный код сложно тестировать

Прежде чем разбирать инструменты, поймём **проблемы**.

### Проблема 1: недетерминизм

**Последовательный код** детерминирован: один и тот же вход → один и тот же выход.

**Конкурентный код** — нет. Порядок выполнения горутин **непредсказуем**.

```go
go func() { counter++ }()
go func() { counter++ }()
// counter может быть 1 или 2
```

**Что это значит для тестов:**

- Тест может пройти 1000 раз и упасть на 1001-й.
- Нельзя полагаться на «прошло — значит, работает».

### Проблема 2: гонки не всегда проявляются

**Data race** — undefined behavior (Глава 4). На x86 гонка может **не проявляться**, на ARM — проявляться.

```go
// Гонка, но на x86 «работает»
var x int
go func() { x = 1 }()
go func() { _ = x }()
```

**Что это значит:** тест на одной платформе может **не найти** гонку, которая есть.

### Проблема 3: утечки горутин

**Горутина, которая никогда не завершится**, не видна в обычных тестах.

```go
go func() {
    ch := make(chan int)
    ch <- 42  // ← блокируется навсегда
}()
```

**Что это значит:** тест проходит, но горутина **утекает**. Через час — OOM.

### Проблема 4: таймауты недетерминированы

```go
select {
case <-ch:
    // успех
case <-time.After(1 * time.Second):
    // таймаут
}
```

**Что это значит:** на быстрой машине — успех, на медленной — таймаут. Тест **нестабилен**.

### Проблема 5: deadlock не всегда воспроизводится

**Deadlock** может проявляться только при определённом порядке выполнения.

```go
mu1.Lock()
mu2.Lock()  // ← если другой поток взял mu2 и ждёт mu1 → deadlock
```

**Что это значит:** тест может пройти, а в production — зависнуть.

### Сводная таблица

| Проблема | Инструмент |
|:---|:---|
| Недетерминизм | Stress-тесты |
| Гонки | `-race` |
| Утечки горутин | `goleak`, pprof |
| Таймауты | `context` с запасом |
| Deadlock | `-race`, stress-тесты |

### Аннотация сложности

| Проблема | Вероятность в тесте |
|:---|:---|
| Недетерминизм | 100% |
| Гонка на x86 | Низкая |
| Гонка на ARM | Средняя |
| Утечка горутин | 100% (не видна) |
| Deadlock | Зависит от нагрузки |

### 💡 Практика: как тестировать конкурентный код

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`go test -race`** — обязательно.
2. **Stress-тесты** — многократный запуск.
3. **`goleak`** — для утечек.
4. **`context` с запасом** — для таймаутов.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Тесты на разных платформах** (x86 + ARM).
6. **pprof** в тестах с `-cpuprofile`, `-memprofile`.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **Формальная верификация** — редко.

**❌ НЕ ДЕЛАЙ:**

8. **Не полагайся на «тест прошёл».** Конкурентный код — сложен.
9. **Не используй `time.Sleep` в тестах** для синхронизации.
10. **Не игнорируй `-race`.**

### Ключевые выводы подглавы 12.1

- **Недетерминизм** — главная проблема.
- **Гонки** не всегда проявляются.
- **Утечки горутин** не видны в обычных тестах.
- **Таймауты** недетерминированы.
- **`-race`, stress-тесты, `goleak`** — обязательные инструменты.

---

## 12.2 Race detector: go test -race

**Race detector** — инструмент для поиска data race.

### Запуск

```bash
# Тесты:
go test -race ./...

# Программа:
go run -race main.go

# Бинарник:
go build -race -o myapp
./myapp
```

### Как работает

**Race detector** использует **Vector Clock** из ThreadSanitizer.

**Для каждой переменной:**

- История доступов: кто писал, кто читал.
- Вектор часов для каждой горутины.

**При каждом доступе:**

- Проверка: есть ли **параллельный** доступ без синхронизации?
- Если да — **race**.

### Пример

```go
package main

import "sync"

func main() {
    var counter int

    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++  // ← data race
        }()
    }
    wg.Wait()
}
```

**Запуск:**

```bash
go run -race main.go
```

**Вывод:**

```
==================
WARNING: DATA RACE
Write at 0x00c00001e0b8 by goroutine 7:
  main.main.func1()
      /path/to/main.go:12 +0x...

Previous write at 0x00c00001e0b8 by goroutine 6:
  main.main.func1()
      /path/to/main.go:12 +0x...

Goroutine 7 (running) created at:
  main.main()
      /path/to/main.go:10 +0x...

Goroutine 6 (running) created at:
  main.main()
      /path/to/main.go:10 +0x...
==================
Found 1 data race(s)
exit status 66
```

**Что видно:**

- **Адрес** (`0x00c00001e0b8`) — где переменная.
- **Две горутины** — обе пишут.
- **Строки** — где именно.
- **Создание горутин** — где запущены.

### Что находит race detector

**Находит:**

- **Write-write** без синхронизации.
- **Write-read** без синхронизации.
- **Через `sync.Mutex`, каналы, `atomic`** — не считает гонкой.

**Не находит:**

- **Read-read** — не гонка.
- **Логические ошибки** — race detector не знает про логику.
- **Гонки, которые не произошли** — если код не выполнился.
- **Гонки в `unsafe`** — может не найти.

### Ограничения

**1. Замедление.**

- Программа с `-race` работает **в 5-20 раз медленнее**.
- Потребляет **в 5-10 раз больше памяти**.

**2. Не находит все гонки.**

- Находит только те, которые **произошли**.
- Если гонка не «выстрелила» — не найдёт.

**3. Ложные срабатывания.**

- Иногда сообщает о гонке, которой нет (при `unsafe`, ассемблере).

**4. Не работает в production.**

- Замедление слишком велико.

### Race detector в CI

**Обязательно в CI:**

```yaml
# .github/workflows/test.yml
- name: Test with race detector
  run: go test -race ./...
```

**Почему:**

- Ловит гонки на ранней стадии.
- Не зависит от платформы.
- Не даёт попасть в production.

### Аннотация сложности

| Аспект | Значение |
|:---|:---|
| Замедление | 5-20x |
| Потребление памяти | 5-10x |
| Находит все гонки? | ❌ Нет |
| Работает в production? | ❌ Нет |
| Ложные срабатывания? | Иногда |

### 💡 Практика: как использовать race detector

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`go test -race ./...` в CI.**
2. **`go run -race` при отладке.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Stress-тесты с `-race`.**
4. **Тесты на ARM** — если прод на ARM.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **`-race` постоянно** — замедление слишком велико.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `-race` в production.**
7. **Не полагайся только на race detector.**
8. **Не игнорируй предупреждения.**

### Ключевые выводы подглавы 12.2

- **`-race`** — стандартный инструмент.
- **Замедляет 5-20x.** Только для тестов.
- **Не находит все гонки.** Только произошедшие.
- **Обязательно в CI.**
- **Не работает в production.**

---

## 12.3 Stress-тесты

**Stress-тесты** — многократный запуск теста для поиска редких гонок.

### Зачем

**Проблема:** гонка может проявляться **редко**. Один запуск теста — не находит.

**Решение:** запустить тест **1000 раз** подряд.

### Паттерн: `-count`

```bash
# Запустить тест 1000 раз
go test -race -count=1000 -run TestConcurrent ./...
```

**Что делает:** `-count=1000` — 1000 запусков.

**Плюс:** просто.

**Минус:** медленно.

### Паттерн: loop внутри теста

```go
func TestConcurrentStress(t *testing.T) {
    for i := 0; i < 1000; i++ {
        t.Run(fmt.Sprintf("iter-%d", i), func(t *testing.T) {
            testConcurrent(t)
        })
    }
}
```

**Что делает:** 1000 подтестов.

**Плюс:** видно, какой iteration упал.

### Паттерн: параллельный stress

```go
func TestConcurrentParallel(t *testing.T) {
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < 100; j++ {
                testConcurrent(t)
            }
        }()
    }
    wg.Wait()
}
```

**Что делает:** 100 горутин × 100 итераций = 10 000 запусков.

**Плюс:** быстро (параллельно).

**Минус:** сложнее отладка.

### Паттерн: `b.N` в бенчмарке

```go
func BenchmarkConcurrent(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            testConcurrent(nil)
        }
    })
}
```

**Что делает:** `go test -bench=. -benchtime=10s` — много итераций.

**Плюс:** race detector работает с бенчмарками.

### Стресс с разными GOMAXPROCS

```go
func TestConcurrentGOMAXPROCS(t *testing.T) {
    for _, procs := range []int{1, 2, 4, 8} {
        t.Run(fmt.Sprintf("GOMAXPROCS=%d", procs), func(t *testing.T) {
            old := runtime.GOMAXPROCS(procs)
            defer runtime.GOMAXPROCS(old)
            testConcurrent(t)
        })
    }
}
```

**Что делает:** тест с разным числом P.

**Плюс:** находит гонки, которые проявляются только при определённом параллелизме.

### Аннотация сложности

| Подход | Время | Гонок найдено |
|:---|:---|:---|
| `-count=1000` | Долго | Больше |
| Loop внутри | Долго | Больше |
| Параллельный | Быстро | Больше |
| Разные GOMAXPROCS | Средне | Больше |

### 💡 Практика: как писать stress-тесты

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`-count=100` или больше** — для конкурентных тестов.
2. **Параллельный stress** — для скорости.
3. **Разные `GOMAXPROCS`** — для полноты.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`-race` вместе со stress.**
5. **Логирование** — какой iteration упал.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Очень долгие stress-тесты** — только в nightly.

**❌ НЕ ДЕЛАЙ:**

7. **Не полагайся на один запуск.**
8. **Не используй `time.Sleep` в stress-тестах.**

### Ключевые выводы подглавы 12.3

- **Stress-тесты** — многократный запуск.
- **`-count=1000`** — простой способ.
- **Параллельный stress** — быстро.
- **Разные `GOMAXPROCS`** — полнота.
- **`-race` вместе со stress.**

---

## 12.4 Тестирование таймаутов и отмены

Разберём **тестирование** таймаутов и отмены.

### Проблема: `time.Sleep` в тестах

**❌ Плохо:**

```go
func TestTimeout(t *testing.T) {
    start := time.Now()
    err := doWorkWithTimeout(100 * time.Millisecond)
    elapsed := time.Since(start)
    if elapsed < 100*time.Millisecond {
        t.Error("did not wait")
    }
}
```

**Проблема:** `elapsed` может быть 101 мс или 150 мс. Тест нестабилен.

### Паттерн: проверка через `context`

**✅ Хорошо:**

```go
func TestContextCancel(t *testing.T) {
    ctx, cancel := context.WithCancel(context.Background())

    done := make(chan struct{})
    go func() {
        defer close(done)
        <-ctx.Done()
    }()

    cancel()

    select {
    case <-done:
        // ОК: горутина завершилась
    case <-time.After(1 * time.Second):
        t.Fatal("goroutine did not exit")
    }
}
```

**Что проверяет:** `ctx.Done()` разблокировал горутину.

### Паттерн: проверка таймаута

```go
func TestTimeout(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
    defer cancel()

    start := time.Now()
    <-ctx.Done()
    elapsed := time.Since(start)

    // Проверяем, что таймаут сработал
    if !errors.Is(ctx.Err(), context.DeadlineExceeded) {
        t.Fatalf("expected DeadlineExceeded, got %v", ctx.Err())
    }

    // Проверяем, что прошло ~50 мс (с запасом)
    if elapsed < 40*time.Millisecond || elapsed > 200*time.Millisecond {
        t.Errorf("elapsed = %v, expected ~50ms", elapsed)
    }
}
```

**Что проверяет:**

- Таймаут сработал.
- Прошло **примерно** 50 мс (с запасом 40-200 мс).

**Запас:** на медленной машине таймер может сработать позже.

### Паттерн: проверка отмены в обработчике

```go
func TestHandlerCancellation(t *testing.T) {
    ctx, cancel := context.WithCancel(context.Background())

    canceled := make(chan struct{})
    go func() {
        defer close(canceled)
        select {
        case <-ctx.Done():
        case <-time.After(10 * time.Second):
            t.Error("timeout")
        }
    }()

    time.Sleep(50 * time.Millisecond)
    cancel()

    select {
    case <-canceled:
    case <-time.After(1 * time.Second):
        t.Fatal("not canceled")
    }
}
```

**Что проверяет:** отмена работает.

### Паттерн: тестовая функция с timeout

```go
func TestWithTimeout(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // Тест не должен висеть больше 5 секунд
    done := make(chan struct{})
    go func() {
        defer close(done)
        // ... тест ...
    }()

    select {
    case <-done:
    case <-ctx.Done():
        t.Fatal("test timed out")
    }
}
```

**Что проверяет:** тест не виснет.

### Использование `-timeout`

**Встроенный флаг:**

```bash
go test -timeout=30s ./...
```

**Что делает:** если тест не завершился за 30 секунд — **паника**. Показывает стек.

**По умолчанию:** 10 минут.

### Аннотация сложности

| Проверка | Time |
|:---|:---|
| `ctx.Done()` | ~1-5 нс |
| `time.After` | ~100-200 нс + таймер |
| `-timeout` | Настраивается |

### 💡 Практика: как тестировать таймауты

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`context` с запасом** — для таймаутов.
2. **Проверка `ctx.Err()`** — для типа ошибки.
3. **`-timeout` в CI.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Проверка `elapsed`** с запасом.
5. **Тест отмены** — что горутина завершается.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `time.Sleep` для синхронизации.**
7. **Не проверяй `elapsed` точно.** Только с запасом.
8. **Не забывай `-timeout`.**

### Ключевые выводы подглавы 12.4

- **`context` с запасом** — для таймаутов.
- **`ctx.Err()`** — для типа ошибки.
- **`elapsed`** — только с запасом.
- **`-timeout`** — обязательно.
- **`ctx.Done()`** — для синхронизации.

---

## 12.5 testing.T.Parallel()

**`testing.T.Parallel()`** — запускает тесты параллельно.

### Как работает

```go
func TestA(t *testing.T) {
    t.Parallel()
    // ...
}

func TestB(t *testing.T) {
    t.Parallel()
    // ...
}
```

**Что делает:** тесты запускаются **параллельно** (до `GOMAXPROCS` одновременно).

### Плюсы

- **Быстрее** — тесты идут параллельно.
- **Больше нагрузки** — легче найти гонки.

### Минусы

- **Общее состояние** — если тесты используют глобальные переменные, ломается.
- **Недетерминизм** — порядок выполнения не определён.
- **Флаки** — тесты могут падать случайно.

### Когда использовать

**Подходит:**

- Тесты **независимы**.
- Нет **общего состояния**.
- Хорошо изолированы.

**Не подходит:**

- Тесты с **глобальными переменными**.
- Тесты с **файлами** (те же пути).
- Тесты, которые **меняют окружение** (`os.Setenv`).

### Пример: правильное использование

```go
func TestSafe(t *testing.T) {
    t.Parallel()

    // Каждый тест создаёт свои данные
    data := make([]int, 100)
    for i := range data {
        data[i] = i
    }

    result := processData(data)
    if len(result) != 100 {
        t.Errorf("len = %d, want 100", len(result))
    }
}
```

### Пример: неправильное использование

**❌ Плохо:**

```go
var globalCounter int  // ← общее состояние

func TestIncrement(t *testing.T) {
    t.Parallel()
    globalCounter++  // ← race между тестами
}
```

**Что происходит:** тесты пишут в `globalCounter` параллельно. **Data race.**

### Паттерн: `-parallel`

```bash
go test -parallel=8 ./...
```

**Что делает:** не более 8 параллельных тестов.

**По умолчанию:** `GOMAXPROCS`.

### Паттерн: `t.Parallel()` + `-race`

```bash
go test -race -parallel=8 ./...
```

**Что делает:** race detector + параллельные тесты. **Максимальная нагрузка.**

**Плюс:** находит гонки между тестами.

### Паттерн: subtests с `t.Parallel()`

```go
func TestSubtests(t *testing.T) {
    for i := 0; i < 10; i++ {
        i := i
        t.Run(fmt.Sprintf("case-%d", i), func(t *testing.T) {
            t.Parallel()
            // ...
        })
    }
}
```

**Что делает:** 10 подтестов параллельно.

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `t.Parallel()` | ~1-10 нс |
| `-parallel=8` | Настраивается |

### 💡 Практика: как использовать t.Parallel()

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`t.Parallel()`** — если тесты **изолированы**.
2. **`-race` + `-parallel`** — для поиска гонок.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Изоляция** — каждый тест создаёт свои данные.
4. **Subtests с `t.Parallel()`** — для многих кейсов.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **`-parallel=8`** — если нужно ограничить.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `t.Parallel()` с общим состоянием.**
7. **Не используй `os.Setenv` в параллельных тестах.**
8. **Не используй один файл в параллельных тестах.**

### Ключевые выводы подглавы 12.5

- **`t.Parallel()`** — параллельные тесты.
- **Быстрее, но требует изоляции.**
- **`-race` + `-parallel`** — для гонок.
- **Не использовать с общим состоянием.**

---

## 12.6 pprof: CPU, memory, goroutine

**pprof** — инструмент профилирования.

### Виды профилей

| Профиль | Что показывает | Как включить |
|:---|:---|:---|
| **CPU** | Где тратится CPU | `-cpuprofile` |
| **Memory** | Аллокации | `-memprofile` |
| **Goroutine** | Все горутины | `pprof.Lookup("goroutine")` |
| **Block** | Блокировки | `SetBlockProfileRate` |
| **Mutex** | Contention | `SetMutexProfileFraction` |

### CPU profile

**В тестах:**

```bash
go test -cpuprofile=cpu.prof -bench=. ./...
go tool pprof cpu.prof
```

**В программе:**

```go
import "runtime/pprof"

f, _ := os.Create("cpu.prof")
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()

// ... работа ...
```

### Memory profile

**В тестах:**

```bash
go test -memprofile=mem.prof -bench=. ./...
go tool pprof mem.prof
```

### Goroutine profile

**В программе:**

```go
import (
    "net/http"
    _ "net/http/pprof"
)

func main() {
    go http.ListenAndServe("localhost:6060", nil)
    // ... работа ...
}
```

**Затем:**

```bash
# Дамп всех горутин
curl http://localhost:6060/debug/pprof/goroutine?debug=2
```

### Анализ через pprof

```bash
go tool pprof cpu.prof
(pprof) top          # топ функций
(pprof) list main    # код функции
(pprof) web          # визуализация
(pprof) peek main.main  # детали
```

### Пример: утечка горутин

```go
package main

import (
    "net/http"
    _ "net/http/pprof"
    "time"
)

func main() {
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()

    // Утечка: 1000 горутин пишут в канал без читателя
    for i := 0; i < 1000; i++ {
        ch := make(chan int)
        go func() {
            ch <- i
        }()
    }

    time.Sleep(1 * time.Hour)
}
```

**Затем:**

```bash
curl http://localhost:6060/debug/pprof/goroutine?debug=2
```

**Вывод:**

```
goroutine 18 [chan send]:
main.main.func2()
    /path/to/main.go:19 +0x...
... (повторяется 1000 раз)
```

**Что видно:** 1000 горутин в `chan send` — утечка.

### Аннотация сложности

| Профиль | Overhead |
|:---|:---|
| CPU | ~5-10% |
| Memory | ~10-20% |
| Goroutine | ~1-5% |
| Block | ~5-10% |
| Mutex | ~5-10% |

### 💡 Практика: как использовать pprof

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`-cpuprofile`** — для CPU-профиля.
2. **`-memprofile`** — для памяти.
3. **`debug/pprof/goroutine?debug=2`** — для горутин.

**👍 СТОИТ СДЕЛАТЬ:**

4. **pprof в production (localhost).**
5. **Анализ через `go tool pprof`.**

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **`web` визуализация** — для сложных случаев.

**❌ НЕ ДЕЛАЙ:**

7. **Не открывай pprof наружу.**
8. **Не включай CPU profile постоянно.**

### Ключевые выводы подглавы 12.6

- **pprof** — профилирование.
- **CPU, memory, goroutine** — основные.
- **`-cpuprofile`, `-memprofile`.**
- **`debug/pprof/goroutine?debug=2`** — для горутин.
- **Анализ через `go tool pprof`.**

---

## 12.7 Block profile и mutex profile

Разберём **block** и **mutex** профили.

### Block profile

**Block profile** — где горутины **блокируются**.

**Включение:**

```go
import "runtime"

runtime.SetBlockProfileRate(1)  // все события
```

**В тестах:**

```bash
go test -blockprofile=block.prof -bench=. ./...
go tool pprof block.prof
```

**Что показывает:**

- **`sync.Mutex.Lock`** — ожидание мьютекса.
- **Каналы** — ожидание отправки/получения.
- **`select`** — ожидание.

**Пример:**

```go
package main

import (
    "os"
    "runtime"
    "runtime/pprof"
    "sync"
    "time"
)

func main() {
    runtime.SetBlockProfileRate(1)

    var mu sync.Mutex
    var counter int

    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < 10000; j++ {
                mu.Lock()
                counter++
                mu.Unlock()
            }
        }()
    }

    time.Sleep(100 * time.Millisecond)

    f, _ := os.Create("block.prof")
    pprof.Lookup("block").WriteTo(f, 0)
    f.Close()

    wg.Wait()
}
```

**Анализ:**

```bash
go tool pprof block.prof
(pprof) top
```

**Вывод:**

```
Showing nodes accounting for 5s, 100% of 5s total
      flat  flat%   sum%        cum   cum%
       5s   100%   100%       5s   100%  sync.(*Mutex).Lock
```

**Что видно:** горутины проводят 5 секунд в ожидании `Mutex.Lock`. **Contention.**

### Mutex profile

**Mutex profile** — contention на мьютексах.

**Включение:**

```go
runtime.SetMutexProfileFraction(1)
```

**В тестах:**

```bash
go test -mutexprofile=mutex.prof -bench=. ./...
go tool pprof mutex.prof
```

**Пример:**

```go
package main

import (
    "os"
    "runtime"
    "runtime/pprof"
    "sync"
    "time"
)

func main() {
    runtime.SetMutexProfileFraction(1)

    var mu sync.Mutex
    var counter int

    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < 10000; j++ {
                mu.Lock()
                counter++
                mu.Unlock()
            }
        }()
    }

    time.Sleep(100 * time.Millisecond)

    f, _ := os.Create("mutex.prof")
    pprof.Lookup("mutex").WriteTo(f, 0)
    f.Close()

    wg.Wait()
}
```

**Анализ:**

```bash
go tool pprof mutex.prof
(pprof) top
```

**Вывод:**

```
Showing nodes accounting for 1.5s, 100% of 1.5s total
      flat  flat%   sum%        cum   cum%
     1.5s   100%   100%       1.5s   100%  sync.(*Mutex).Unlock
```

**Что видно:** contention на `Mutex.Unlock`.

### Разница между block и mutex

| Аспект | Block | Mutex |
|:---|:---|:---|
| Что показывает | Все блокировки | Только мьютексы |
| Включает | Каналы, `select` | `sync.Mutex`, `RWMutex` |
| Overhead | ~5-10% | ~5-10% |
| Когда использовать | Общая диагностика | Contention на мьютексах |

### Аннотация сложности

| Профиль | Overhead | Что показывает |
|:---|:---|:---|
| Block | ~5-10% | Все блокировки |
| Mutex | ~5-10% | Contention на мьютексах |

### 💡 Практика: как использовать block и mutex profile

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`SetBlockProfileRate(1)`** — для block profile.
2. **`SetMutexProfileFraction(1)`** — для mutex profile.
3. **`go tool pprof`** — для анализа.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Block profile** — для поиска узких мест.
5. **Mutex profile** — для contention.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **`web` визуализация** — для сложных случаев.

**❌ НЕ ДЕЛАЙ:**

7. **Не включай в production постоянно.**
8. **Не игнорируй высокий contention.**

### Ключевые выводы подглавы 12.7

- **Block profile** — все блокировки.
- **Mutex profile** — contention на мьютексах.
- **`SetBlockProfileRate`, `SetMutexProfileFraction`.**
- **Block** — для диагностики. **Mutex** — для contention.

---

## 12.8 Goroutine dump

**Goroutine dump** — снимок всех горутин с их стеками.

### Как получить

**1. `SIGQUIT`:**

```bash
kill -QUIT <pid>
```

**Что происходит:** программа печатает dump **всех** горутин в stderr и **завершается**.

**2. `runtime.Stack`:**

```go
buf := make([]byte, 1<<20)
n := runtime.Stack(buf, true)  // true = все горутины
fmt.Printf("%s", buf[:n])
```

**3. pprof:**

```bash
curl http://localhost:6060/debug/pprof/goroutine?debug=2
```

### Что показывает

**Формат:**

```
goroutine 18 [chan send]:
main.worker()
    /path/to/main.go:15 +0x...
created by main.main in goroutine 1
    /path/to/main.go:10 +0x...
```

**Разбор:**

- **`goroutine 18`** — ID.
- **`[chan send]`** — состояние (что делает).
- **`main.worker()`** — функция.
- **Строка** — где.
- **`created by`** — кто создал.

### Состояния горутин

| Состояние | Что означает |
|:---|:---|
| `[running]` | Выполняется |
| `[chan send]` | Ждёт отправки в канал |
| `[chan receive]` | Ждёт получения из канала |
| `[select]` | Ждёт в `select` |
| `[semacquire]` | Ждёт мьютекс/семафор |
| `[IO wait]` | Ждёт I/O |
| `[sleep]` | Спит |
| `[syscall]` | В syscall |
| `[finalizer wait]` | Ждёт finalizer |

### Пример: утечка

```
goroutine 18 [chan send]:
main.main.func1()
    /path/to/main.go:15 +0x...
... (повторяется 1000 раз)
```

**Что видно:** 1000 горутин в `chan send` — утечка.

### Пример: deadlock

```
goroutine 1 [semacquire]:
sync.runtime_SemacquireMutex(...)
sync.(*Mutex).lockSlow(...)
sync.(*Mutex).Lock(...)
main.main()
    /path/to/main.go:20 +0x...

goroutine 2 [semacquire]:
sync.runtime_SemacquireMutex(...)
sync.(*Mutex).lockSlow(...)
sync.(*Mutex).Lock(...)
main.main()
    /path/to/main.go:25 +0x...
```

**Что видно:** две горутины ждут `Mutex.Lock`. **Deadlock.**

### Анализ dump

**Что искать:**

1. **Много горутин в одном состоянии** — утечка.
2. **Горутины в `semacquire`** — contention.
3. **Горутины в `chan send`/`chan receive`** — deadlock.
4. **Горутины в `IO wait`** — медленный I/O.
5. **Горутины в `[running]`** — активно работают.

### Автоматизация

```go
import (
    "os"
    "os/signal"
    "runtime"
    "syscall"
)

func main() {
    go func() {
        sigCh := make(chan os.Signal, 1)
        signal.Notify(sigCh, syscall.SIGUSR1)
        for range sigCh {
            buf := make([]byte, 1<<20)
            n := runtime.Stack(buf, true)
            os.Stdout.Write(buf[:n])
        }
    }()

    // ... работа ...
}
```

**Что делает:** `SIGUSR1` печатает dump **без завершения**.

### Аннотация сложности

| Метод | Time | Space |
|:---|:---|:---|
| `SIGQUIT` | ~1-100 мс | ~1-10 МБ |
| `runtime.Stack` | ~1-100 мс | ~1-10 МБ |
| pprof | ~10-100 мс | ~1-10 МБ |

### 💡 Практика: как использовать goroutine dump

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`runtime.Stack(buf, true)`** — при подозрении.
2. **pprof goroutine dump** — в production (localhost).
3. **`SIGUSR1`** — для дампа без завершения.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Анализ dump** — искать много горутин в одном состоянии.
5. **Логирование** — что происходит.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Автоматический дамп** при росте горутин.

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй растущее число горутин.**
8. **Не дампи слишком часто** — дорого.

### Ключевые выводы подглавы 12.8

- **Goroutine dump** — снимок всех горутин.
- **`SIGQUIT`, `runtime.Stack`, pprof.**
- **Состояния:** `chan send`, `chan receive`, `semacquire`, `IO wait`.
- **Анализ** — искать много горутин в одном состоянии.
- **`SIGUSR1`** — для дампа без завершения.

---

## 12.9 runtime/trace

**`runtime/trace`** — детальная трассировка выполнения.

### Включение

**В программе:**

```go
import (
    "os"
    "runtime/trace"
)

f, _ := os.Create("trace.out")
defer f.Close()
trace.Start(f)
defer trace.Stop()

// ... работа ...
```

**В тестах:**

```bash
go test -trace=trace.out ./...
```

### Анализ

```bash
go tool trace trace.out
```

**Открывается браузер** с визуализацией:

- **Goroutines** — какие G выполняются.
- **Heap** — память.
- **Threads** — M.
- **Proc** — P.
- **Network** — netpoller.
- **Syscalls** — syscall.

### Что показывает

**1. Timeline выполнения.**

Видно, какие горутины выполняются в каждый момент.

**2. Переключения.**

Видно, когда планировщик переключает горутины.

**3. Блокировки.**

Видно, где горутины блокируются.

**4. GC.**

Видно, когда сработал GC.

### Пример: что искать

**Проблема:** программа работает медленно.

**Анализ через trace:**

1. **Много goroutines?** Может, утечка.
2. **Долгие блокировки?** Может, contention.
3. **Много GC?** Может, много аллокаций.
4. **Простаивающие P?** Может, не хватает работы.

### Ограничения

**1. Overhead.**

Трассировка замедляет программу.

**2. Размер файла.**

Может быть **сотни МБ**.

**3. Только для отладки.**

Не для production.

### Аннотация сложности

| Аспект | Значение |
|:---|:---|
| Overhead | ~10-20% |
| Размер файла | ~10-100 МБ/сек |
| Анализ | Через браузер |

### 💡 Практика: как использовать runtime/trace

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`trace.Start/Stop`** — для трассировки.
2. **`go tool trace`** — для анализа.
3. **Только для отладки.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Короткие сессии** — 1-10 секунд.
5. **Фокус на проблеме** — что искать.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй в production.**
7. **Не записывай долгие сессии.**

### Ключевые выводы подглавы 12.9

- **`runtime/trace`** — детальная трассировка.
- **`go tool trace`** — визуализация.
- **Overhead ~10-20%.**
- **Только для отладки.**
- **Короткие сессии.**

---

## 12.10 Бенчмарки конкурентного кода

Разберём **бенчмарки** для конкурентного кода.

### Базовый бенчмарк

```go
func BenchmarkMutex(b *testing.B) {
    var mu sync.Mutex
    var counter int

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        mu.Lock()
        counter++
        mu.Unlock()
    }
}
```

**Запуск:**

```bash
go test -bench=BenchmarkMutex -benchmem ./...
```

### Бенчмарк с параллелизмом

**`b.RunParallel`** — запускает бенчмарк параллельно.

```go
func BenchmarkMutexParallel(b *testing.B) {
    var mu sync.Mutex
    var counter int

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            mu.Lock()
            counter++
            mu.Unlock()
        }
    })
}
```

**Что делает:** `GOMAXPROCS` горутин параллельно.

### Сравнение Mutex vs Atomic

```go
func BenchmarkMutex(b *testing.B) {
    var mu sync.Mutex
    var counter int64

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            mu.Lock()
            counter++
            mu.Unlock()
        }
    })
}

func BenchmarkAtomic(b *testing.B) {
    var counter atomic.Int64

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            counter.Add(1)
        }
    })
}
```

**Запуск:**

```bash
go test -bench=. -benchmem ./...
```

**Пример вывода:**

```
BenchmarkMutex-8     50000000    25 ns/op
BenchmarkAtomic-8   100000000    10 ns/op
```

**Что видно:** `atomic` быстрее `Mutex` в 2.5 раза.

### Бенчмарк каналов

```go
func BenchmarkChanBuffered(b *testing.B) {
    ch := make(chan int, 100)

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            ch <- 1
            <-ch
        }
    })
}

func BenchmarkChanUnbuffered(b *testing.B) {
    ch := make(chan int)

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            ch <- 1
            <-ch
        }
    })
}
```

### Бенчмарк worker pool

```go
func BenchmarkWorkerPool(b *testing.B) {
    const numWorkers = 10
    tasksCh := make(chan int, b.N)
    var wg sync.WaitGroup

    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for range tasksCh {
                // обработка
            }
        }()
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        tasksCh <- i
    }
    close(tasksCh)
    wg.Wait()
}
```

### Флаги

| Флаг | Что делает |
|:---|:---|
| `-bench=.` | Все бенчмарки |
| `-benchmem` | Память |
| `-benchtime=10s` | Время |
| `-cpu=1,2,4,8` | Разные GOMAXPROCS |
| `-count=10` | Повторы |

### Пример с разными GOMAXPROCS

```bash
go test -bench=. -benchmem -cpu=1,2,4,8 ./...
```

**Вывод:**

```
BenchmarkMutex-1     200000000    8 ns/op
BenchmarkMutex-2     100000000   15 ns/op
BenchmarkMutex-4      50000000   25 ns/op
BenchmarkMutex-8      30000000   40 ns/op
```

**Что видно:** с ростом `GOMAXPROCS` contention растёт, время на операцию увеличивается.

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `b.RunParallel` | ~1-10 нс overhead |
| `-benchtime=10s` | Настраивается |
| `-count=10` | Настраивается |

### 💡 Практика: как писать бенчмарки

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`b.RunParallel`** — для параллельных бенчмарков.
2. **`-benchmem`** — для памяти.
3. **`-cpu=1,2,4,8`** — для разных GOMAXPROCS.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`-count=10`** — для стабильности.
5. **`b.ResetTimer`** — после setup.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **`-benchtime=10s`** — для точности.

**❌ НЕ ДЕЛАЙ:**

7. **Не забывай `b.ResetTimer`.**
8. **Не сравнивай несравнимое.**

### Ключевые выводы подглавы 12.10

- **`b.RunParallel`** — параллельный бенчмарк.
- **`-benchmem`** — память.
- **`-cpu=1,2,4,8`** — разные GOMAXPROCS.
- **`-count=10`** — стабильность.
- **`atomic` быстрее `Mutex`** в 2.5 раза.

---

## 12.11 goleak: поиск утечек

**`goleak`** — библиотека для поиска утечек горутин в тестах.

### Установка

```bash
go get go.uber.org/goleak
```

### Использование

**В `TestMain`:**

```go
import "go.uber.org/goleak"

func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```

**Что делает:** после **всех** тестов проверяет, что не осталось горутин.

**В конкретном тесте:**

```go
func TestWorkerPool(t *testing.T) {
    defer goleak.VerifyNone(t)

    // ... тест ...
}
```

**Что делает:** после **теста** проверяет, что не осталось горутин.

### Пример: утечка

```go
func TestLeak(t *testing.T) {
    defer goleak.VerifyNone(t)

    ch := make(chan int)
    go func() {
        ch <- 42  // ← утечка
    }()
}
```

**Вывод:**

```
found unexpected goroutines:
[Goroutine 18 in state chan send, with main.TestLeak.func1 on top of the stack:
main.TestLeak.func1()
    /path/to/main_test.go:12 +0x...
]
```

**Что видно:** горутина в `chan send` — утечка.

### Фильтрация известных горутин

```go
goleak.VerifyNone(t,
    goleak.IgnoreTopFunction("net/http.(*Server).Serve"),
    goleak.IgnoreCurrent(),
)
```

**Что делает:** игнорирует известные горутины (например, HTTP-сервер).

### Пример: правильный тест

```go
func TestWorkerPoolNoLeak(t *testing.T) {
    defer goleak.VerifyNone(t)

    tasksCh := make(chan int, 10)
    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for range tasksCh {
                // обработка
            }
        }()
    }

    for i := 0; i < 100; i++ {
        tasksCh <- i
    }
    close(tasksCh)
    wg.Wait()
}
```

**Что проверяет:** после теста нет утечек.

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `VerifyNone` | ~1-10 мс |
| `VerifyTestMain` | ~1-10 мс |

### 💡 Практика: как использовать goleak

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`VerifyTestMain`** — для всех тестов.
2. **`VerifyNone`** — для конкретных тестов.
3. **Фильтрация** — для известных горутин.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Тесты для каждого конкурентного компонента.**
5. **`goleak` в CI.**

**❌ НЕ ДЕЛАЙ:**

6. **Не игнорируй утечки.**
7. **Не отключай `goleak` без причины.**

### Ключевые выводы подглавы 12.11

- **`goleak`** — поиск утечек горутин.
- **`VerifyTestMain`** — для всех тестов.
- **`VerifyNone`** — для конкретных.
- **Фильтрация** — для известных.
- **`goleak` в CI.**

---

## 12.12 Практика Go: тестирование worker pool

Напишем **полное тестирование** worker pool.

### Код worker pool

```go
package pool

import (
    "context"
    "sync"
)

type Task struct {
    ID  int
    Val int
}

type Result struct {
    TaskID int
    Val    int
    Err    error
}

func Process(ctx context.Context, tasks []Task, numWorkers int) ([]Result, error) {
    tasksCh := make(chan Task, len(tasks))
    resultsCh := make(chan Result, len(tasks))

    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for task := range tasksCh {
                select {
                case <-ctx.Done():
                    return
                case resultsCh <- process(ctx, task):
                }
            }
        }()
    }

    for _, task := range tasks {
        tasksCh <- task
    }
    close(tasksCh)

    go func() {
        wg.Wait()
        close(resultsCh)
    }()

    var results []Result
    for r := range resultsCh {
        results = append(results, r)
    }

    return results, nil
}

func process(ctx context.Context, task Task) Result {
    select {
    case <-ctx.Done():
        return Result{TaskID: task.ID, Err: ctx.Err()}
    default:
    }
    return Result{TaskID: task.ID, Val: task.Val * 2}
}
```

### Тесты

```go
package pool

import (
    "context"
    "errors"
    "fmt"
    "runtime"
    "sync"
    "testing"
    "time"

    "go.uber.org/goleak"
)

func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}

// Тест 1: базовый
func TestProcess(t *testing.T) {
    tasks := []Task{
        {ID: 1, Val: 1},
        {ID: 2, Val: 2},
        {ID: 3, Val: 3},
    }

    results, err := Process(context.Background(), tasks, 2)
    if err != nil {
        t.Fatal(err)
    }

    if len(results) != 3 {
        t.Fatalf("got %d results, want 3", len(results))
    }

    // Проверяем значения
    for _, r := range results {
        want := r.TaskID * 2
        if r.Val != want {
            t.Errorf("task %d: got %d, want %d", r.TaskID, r.Val, want)
        }
    }
}

// Тест 2: отмена
func TestProcessCancel(t *testing.T) {
    ctx, cancel := context.WithCancel(context.Background())
    cancel()  // сразу отменяем

    tasks := make([]Task, 100)
    for i := range tasks {
        tasks[i] = Task{ID: i, Val: i}
    }

    results, _ := Process(ctx, tasks, 10)

    // Все результаты должны быть с ошибкой
    for _, r := range results {
        if r.Err == nil {
            t.Errorf("task %d: expected error, got nil", r.TaskID)
        }
    }
}

// Тест 3: stress с race detector
func TestProcessStress(t *testing.T) {
    tasks := make([]Task, 1000)
    for i := range tasks {
        tasks[i] = Task{ID: i, Val: i}
    }

    for iter := 0; iter < 10; iter++ {
        t.Run(fmt.Sprintf("iter-%d", iter), func(t *testing.T) {
            results, err := Process(context.Background(), tasks, 20)
            if err != nil {
                t.Fatal(err)
            }
            if len(results) != 1000 {
                t.Fatalf("got %d results, want 1000", len(results))
            }
        })
    }
}

// Тест 4: разные GOMAXPROCS
func TestProcessGOMAXPROCS(t *testing.T) {
    for _, procs := range []int{1, 2, 4, 8} {
        t.Run(fmt.Sprintf("GOMAXPROCS=%d", procs), func(t *testing.T) {
            old := runtime.GOMAXPROCS(procs)
            defer runtime.GOMAXPROCS(old)

            tasks := make([]Task, 100)
            for i := range tasks {
                tasks[i] = Task{ID: i, Val: i}
            }

            results, err := Process(context.Background(), tasks, 10)
            if err != nil {
                t.Fatal(err)
            }
            if len(results) != 100 {
                t.Fatalf("got %d results, want 100", len(results))
            }
        })
    }
}

// Тест 5: таймаут
func TestProcessTimeout(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
    defer cancel()

    tasks := make([]Task, 100)
    for i := range tasks {
        tasks[i] = Task{ID: i, Val: i}
    }

    results, _ := Process(ctx, tasks, 10)

    // Проверяем, что ctx.Err() = DeadlineExceeded
    if !errors.Is(ctx.Err(), context.DeadlineExceeded) {
        t.Fatalf("expected DeadlineExceeded, got %v", ctx.Err())
    }

    // Некоторые результаты могут быть с ошибкой
    t.Logf("got %d results", len(results))
}

// Тест 6: параллельные тесты
func TestProcessParallel(t *testing.T) {
    t.Parallel()

    tasks := make([]Task, 100)
    for i := range tasks {
        tasks[i] = Task{ID: i, Val: i}
    }

    results, err := Process(context.Background(), tasks, 10)
    if err != nil {
        t.Fatal(err)
    }
    if len(results) != 100 {
        t.Fatalf("got %d results, want 100", len(results))
    }
}

// Бенчмарк
func BenchmarkProcess(b *testing.B) {
    tasks := make([]Task, 1000)
    for i := range tasks {
        tasks[i] = Task{ID: i, Val: i}
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        Process(context.Background(), tasks, 10)
    }
}

// Бенчмарк с параллелизмом
func BenchmarkProcessParallel(b *testing.B) {
    tasks := make([]Task, 100)
    for i := range tasks {
        tasks[i] = Task{ID: i, Val: i}
    }

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            Process(context.Background(), tasks, 10)
        }
    })
}
```

### Запуск

```bash
# Обычные тесты
go test ./...

# С race detector
go test -race ./...

# Stress-тесты
go test -race -count=100 ./...

# Разные GOMAXPROCS
go test -race -cpu=1,2,4,8 ./...

# Бенчмарки
go test -bench=. -benchmem ./...

# С профилированием
go test -bench=. -cpuprofile=cpu.prof -memprofile=mem.prof ./...
```

### Что демонстрирует

1. **`goleak`** — проверка утечек.
2. **Stress-тесты** — 10 итераций.
3. **Разные GOMAXPROCS.**
4. **Тест отмены** — `context.Canceled`.
5. **Тест таймаута** — `DeadlineExceeded`.
6. **`t.Parallel()`** — для изолированных тестов.
7. **Бенчмарки** — `b.RunParallel`.

### Аннотация сложности

| Тест | Time |
|:---|:---|
| TestProcess | ~1 мс |
| TestProcessCancel | ~1 мс |
| TestProcessStress (10 iter) | ~10 мс |
| TestProcessGOMAXPROCS | ~10 мс |
| TestProcessTimeout | ~50 мс |
| BenchmarkProcess | ~1 сек |

### 💡 Практика: как тестировать worker pool

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`goleak`** — для утечек.
2. **Stress-тесты** — 10+ итераций.
3. **Разные GOMAXPROCS.**
4. **Тест отмены и таймаута.**
5. **`-race` в CI.**

**👍 СТОИТ СДЕЛАТЬ:**

6. **`t.Parallel()`** — для изолированных.
7. **Бенчмарки.**

**❌ НЕ ДЕЛАЙ:**

8. **Не используй `time.Sleep` для синхронизации.**
9. **Не полагайся на один запуск.**

### Ключевые выводы подглавы 12.12

- **`goleak`** — для утечек.
- **Stress-тесты** — 10+ итераций.
- **Разные GOMAXPROCS.**
- **Тест отмены и таймаута.**
- **Бенчмарки** с `b.RunParallel`.

---

## 12.13 Выводы и типичные ошибки

**Что мы узнали?**

Конкурентный код сложно тестировать: недетерминизм, гонки, утечки. `-race` — обязательный инструмент, замедляет 5-20x, не находит все гонки. Stress-тесты — многократный запуск. `context` с запасом — для таймаутов. `t.Parallel()` — для изолированных. pprof — CPU, memory, goroutine. Block/mutex profile — contention. Goroutine dump — снимок всех горутин. `runtime/trace` — детальная трассировка. Бенчмарки с `b.RunParallel`. `goleak` — поиск утечек.

**Типичные ошибки:**

- ❌ **Не запускать `-race`.** Гонки попадут в production.
- ❌ **Использовать `time.Sleep` в тестах.** Недетерминизм.
- ❌ **Один запуск теста.** Не находит редкие гонки.
- ❌ **Не использовать `goleak`.** Утечки горутин.
- ❌ **Проверять `elapsed` точно.** Флаки.
- ❌ **`t.Parallel()` с общим состоянием.** Гонки между тестами.
- ❌ **Не использовать `-timeout`.** Тесты виснут.
- ❌ **Не проверять `ctx.Err()`.** Непонятно, что произошло.
- ❌ **Игнорировать pprof.** Не видно узких мест.
- ❌ **Включать CPU profile постоянно.** Overhead.
- ❌ **Открывать pprof наружу.** Безопасность.
- ❌ **Не использовать stress-тесты.** Редкие гонки.
- ❌ **Не тестировать разные GOMAXPROCS.** Не находит гонки.
- ❌ **Не использовать `-benchmem`.** Не видно аллокаций.

---

## 12.14 Для быстрого повторения

- **Конкурентный код сложно тестировать:** недетерминизм, гонки, утечки.
- **`-race`** — обязателен. Замедляет 5-20x. Не находит все гонки.
- **Stress-тесты** — `-count=100`, параллельные, разные GOMAXPROCS.
- **`context` с запасом** — для таймаутов.
- **`t.Parallel()`** — для изолированных тестов.
- **pprof:** CPU, memory, goroutine.
- **Block profile** — все блокировки. **Mutex profile** — contention.
- **Goroutine dump** — `SIGQUIT`, `runtime.Stack`, pprof.
- **`runtime/trace`** — детальная трассировка.
- **Бенчмарки:** `b.RunParallel`, `-benchmem`, `-cpu=1,2,4,8`.
- **`goleak`** — поиск утечек.
- **`atomic` быстрее `Mutex`** в 2.5 раза.
- **`-timeout`** — обязательно.
- **`go test -race ./...` в CI.**
- **Не `time.Sleep` в тестах.**
- **Не один запуск.**
- **Не игнорируй pprof.**

---

## 12.15 Вопросы для самопроверки

1. Почему конкурентный код сложно тестировать? Назови три проблемы.
2. Что такое race detector? Как запустить?
3. Что race detector находит? Что не находит?
4. Какие ограничения race detector?
5. Что такое stress-тесты? Как писать?
6. Как тестировать таймауты?
7. Что такое `t.Parallel()`? Когда использовать?
8. Что такое pprof? Какие профили?
9. Что такое block profile? Mutex profile?
10. Как получить goroutine dump?
11. Какие состояния горутин?
12. Что такое `runtime/trace`?
13. Как писать бенчмарки конкурентного кода?
14. Что такое `b.RunParallel`?
15. Что такое `goleak`?
16. Как использовать `goleak` в тестах?
17. Что делать, если тест падает только на CI?
18. Почему `atomic` быстрее `Mutex`?

---

## 12.16 Ответы

### Ответ 1

**Три проблемы:**
1. **Недетерминизм** — порядок горутин непредсказуем.
2. **Гонки** — не всегда проявляются.
3. **Утечки горутин** — не видны в обычных тестах.

### Ответ 2

**Race detector** — инструмент для поиска data race.

```bash
go test -race ./...
go run -race main.go
```

### Ответ 3

**Находит:**
- Write-write без синхронизации.
- Write-read без синхронизации.

**Не находит:**
- Read-read.
- Логические ошибки.
- Гонки, которые не произошли.

### Ответ 4

**Ограничения:**
- Замедление 5-20x.
- Не находит все гонки.
- Ложные срабатывания.
- Не работает в production.

### Ответ 5

**Stress-тесты** — многократный запуск.

```bash
go test -race -count=1000 ./...
```

Или loop внутри теста.

### Ответ 6

**Тестирование таймаутов:**

```go
ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
defer cancel()

start := time.Now()
<-ctx.Done()
elapsed := time.Since(start)

if !errors.Is(ctx.Err(), context.DeadlineExceeded) {
    t.Fatal("wrong error")
}
if elapsed < 40*time.Millisecond || elapsed > 200*time.Millisecond {
    t.Errorf("wrong elapsed: %v", elapsed)
}
```

### Ответ 7

**`t.Parallel()`** — параллельные тесты.

**Когда использовать:** если тесты **изолированы** (нет общего состояния).

### Ответ 8

**pprof** — профилирование.

**Профили:** CPU, memory, goroutine, block, mutex.

### Ответ 9

**Block profile** — все блокировки (каналы, mutex, select).

**Mutex profile** — contention на мьютексах.

### Ответ 10

**Goroutine dump:**

```bash
kill -QUIT <pid>
```

Или:

```go
buf := make([]byte, 1<<20)
n := runtime.Stack(buf, true)
```

Или:

```bash
curl http://localhost:6060/debug/pprof/goroutine?debug=2
```

### Ответ 11

**Состояния:**
- `[running]` — выполняется.
- `[chan send]` — ждёт отправки.
- `[chan receive]` — ждёт получения.
- `[select]` — в select.
- `[semacquire]` — ждёт мьютекс.
- `[IO wait]` — ждёт I/O.
- `[sleep]` — спит.
- `[syscall]` — в syscall.

### Ответ 12

**`runtime/trace`** — детальная трассировка.

```go
trace.Start(f)
defer trace.Stop()
```

```bash
go tool trace trace.out
```

### Ответ 13

**Бенчмарки:**

```go
func BenchmarkMutex(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            // ...
        }
    })
}
```

### Ответ 14

**`b.RunParallel`** — запускает бенчмарк параллельно на `GOMAXPROCS` горутинах.

### Ответ 15

**`goleak`** — библиотека для поиска утечек горутин в тестах.

### Ответ 16

```go
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}

// Или в конкретном тесте:
func TestX(t *testing.T) {
    defer goleak.VerifyNone(t)
    // ...
}
```

### Ответ 17

**Если тест падает только на CI:**
- **Race detector** — `-race`.
- **Разные GOMAXPROCS** — `-cpu=1,2,4,8`.
- **Stress-тесты** — `-count=100`.
- **Разная архитектура** — ARM vs x86.

### Ответ 18

**`atomic` быстрее `Mutex`**, потому что:
- Нет блокировки.
- Одна атомарная операция.
- Нет contention.

В 2.5 раза быстрее для счётчиков.

---

## 12.17 Куда идти дальше?

Мы разобрали тестирование и профилирование: race detector, stress-тесты, pprof, trace, goleak. Теперь мы умеем **находить** проблемы.

Но остаётся **важный вопрос**: какие **анти-паттерны** существуют? Что **не надо делать**?

- **Какие анти-паттерны существуют?** Утечки горутин, deadlock, livelock, гонки. → **Глава 13: Анти-паттерны и типичные ошибки.**
- **Реальные сценарии:** HTTP-сервер, очереди, ETL, scraping. → **Глава 14: Реальные сценарии.**

---

## 12.18 Чек-лист

| Инструмент | Что делает | Когда использовать |
|:---|:---|:---|
| **`-race`** | Race detector | Всегда в CI |
| **Stress-тесты** | Многократный запуск | Для поиска редких гонок |
| **`-count=1000`** | 1000 запусков | Stress |
| **`context` с запасом** | Таймауты | Для тестов таймаутов |
| **`t.Parallel()`** | Параллельные тесты | Для изолированных |
| **pprof CPU** | Где CPU | Для оптимизации |
| **pprof memory** | Аллокации | Для оптимизации |
| **pprof goroutine** | Все горутины | Для утечек |
| **Block profile** | Все блокировки | Для диагностики |
| **Mutex profile** | Contention | Для мьютексов |
| **Goroutine dump** | Снимок горутин | `SIGQUIT`, pprof |
| **`runtime/trace`** | Трассировка | Для сложных случаев |
| **`b.RunParallel`** | Параллельный бенчмарк | Для параллельных |
| **`-benchmem`** | Память | Для бенчмарков |
| **`-cpu=1,2,4,8`** | Разные GOMAXPROCS | Для полноты |
| **`goleak`** | Утечки горутин | Всегда в тестах |
| **`-timeout`** | Таймаут тестов | Всегда |
| **`go test -race ./...`** | CI | Обязательно |

🧪 **Ключевая идея:** Конкурентный код сложно тестировать из-за недетерминизма, гонок, утечек. `-race` обязателен, но не находит все гонки; замедляет 5-20x. Stress-тесты (`-count=1000`, разные `GOMAXPROCS`) находят редкие гонки. `context` с запасом — для таймаутов. `t.Parallel()` — для изолированных. pprof (CPU, memory, goroutine), block/mutex profile — для диагностики. Goroutine dump (`SIGQUIT`, pprof) — для утечек. `runtime/trace` — детальная трассировка. Бенчмарки с `b.RunParallel`. `goleak` — для утечек горутин. `-timeout` — обязательно. `go test -race ./...` в CI. `atomic` быстрее `Mutex` в 2.5 раза.