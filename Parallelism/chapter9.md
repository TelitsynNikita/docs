# ⚡ Глава 9: Продвинутые паттерны — semaphore, rate limiter, circuit breaker

**Что вы узнаете:**
- Что такое **semaphore** и почему буферизованный канал — это семафор.
- Что такое **взвешенный семафор** и когда он нужен.
- Что такое **rate limiter** и зачем ограничивать скорость, а не параллелизм.
- Как **написать rate limiter с нуля** — token bucket.
- Чем **token bucket** отличается от **leaky bucket**.
- Что такое **circuit breaker** и как он защищает от каскадных отказов.
- Что такое **future/promise** и **generator**.
- Как комбинировать эти паттерны в реальных системах.
- Как диагностировать проблемы через метрики.

**После прочтения вы сможете:**
- Построить семафор на каналах и через `golang.org/x/sync/semaphore`.
- Написать rate limiter с нуля на каналах и на `time.Ticker`.
- Выбрать между worker pool, семафором и rate limiter.
- Настроить `golang.org/x/time/rate` для HTTP-клиента.
- Реализовать circuit breaker с нуля.
- Понимать, когда каждый паттерн уместен.
- Избегать типичных ошибок: over-limiting, under-limiting, ложные срабатывания.

---

## Содержание

- [9.0 Пролог: сервис, который положил внешний API](#90-пролог-сервис-который-положил-внешний-api)
- [9.1 Semaphore: ограничение параллелизма](#91-semaphore-ограничение-параллелизма)
- [9.2 Взвешенный семафор](#92-взвешенный-семафор)
- [9.3 Rate limiter: ограничение скорости](#93-rate-limiter-ограничение-скорости)
- [9.4 Rate limiter с нуля: token bucket](#94-rate-limiter-с-нуля-token-bucket)
- [9.5 Token bucket vs leaky bucket](#95-token-bucket-vs-leaky-bucket)
- [9.6 Circuit breaker: защита от каскадных отказов](#96-circuit-breaker-защита-от-каскадных-отказов)
- [9.7 Future/promise и generator](#97-futurepromise-и-generator)
- [9.8 Комбинирование паттернов](#98-комбинирование-паттернов)
- [9.9 Практика Go: rate limiter с метриками](#99-практика-go-rate-limiter-с-метриками)
- [9.10 Выводы и типичные ошибки](#910-выводы-и-типичные-ошибки)
- [9.11 Для быстрого повторения](#911-для-быстрого-повторения)
- [9.12 Вопросы для самопроверки](#912-вопросы-для-самопроверки)
- [9.13 Ответы](#913-ответы)
- [9.14 Куда идти дальше?](#914-куда-идти-дальше)
- [9.15 Чек-лист](#915-чек-лист)

---

## 9.0 Пролог: сервис, который положил внешний API

Ты пишешь сервис, который синхронизирует данные с внешним API. 10 000 пользователей, для каждого нужно получить профиль:

```go
func syncUsers(users []User) {
    var wg sync.WaitGroup
    for _, user := range users {
        wg.Add(1)
        go func(u User) {
            defer wg.Done()
            resp, err := http.Get("https://api.example.com/users/" + u.ID)
            // ...
        }(user)
    }
    wg.Wait()
}
```

Запускаешь. Через 30 секунд приходит письмо от внешнего сервиса:

> «Ваш сервис превысил лимит: 10 000 запросов в секунду. Мы заблокировали ваш API-ключ на 24 часа.»

❓ **Что произошло?** Ты запустил 10 000 горутин одновременно. Все они сделали HTTP-запрос **в одну секунду**. Внешний API не выдержал.

💡 **Решение:** **rate limiter**. Ограничить **скорость** запросов — не 10 000 в секунду, а, скажем, 100 в секунду.

**Rate limiter** — это паттерн, который пропускает не более N операций за период времени.

**Worker pool** (Глава 7) ограничивает **параллелизм** — сколько операций **одновременно**. **Rate limiter** ограничивает **скорость** — сколько операций **за секунду**.

**Пример разницы:**

- **Worker pool N=10:** 10 запросов одновременно. Если каждый длится 100 мс — 100 запросов/сек.
- **Rate limiter 100/сек:** 100 запросов/сек, независимо от параллелизма. Если каждый длится 1 мс — 100 запросов/сек. Если 1 секунду — тоже 100 запросов/сек (с задержкой).

**Оба паттерна нужны.** Часто — **вместе**.

> **Важный мост к будущим главам:** rate limiter — надстройка над worker pool (Глава 7). Circuit breaker — надстройка над обработкой ошибок (Глава 10). Semaphore — обобщение worker pool. Всё это — продвинутые паттерны для реальных систем.

---

## 9.1 Semaphore: ограничение параллелизма

**Semaphore** — примитив, который ограничивает **число одновременных операций**.

### Семафор vs worker pool

Мы уже видели семафор в Главе 7 (подглава 7.8). Разберём подробнее.

**Worker pool:**

- **N воркеров** создаются один раз.
- Воркеры **переиспользуются**.
- Задачи идут через канал.

**Semaphore:**

- **N слотов.**
- Каждая операция **захватывает** слот.
- Когда все слоты заняты — новая операция **ждёт**.
- После завершения — слот **освобождается**.

### Семафор на каналах

**Простейший семафор** — буферизованный канал:

```go
sem := make(chan struct{}, 10)  // 10 слотов

for _, task := range tasks {
    sem <- struct{}{}  // захватить слот (блокируется, если все заняты)
    go func(t Task) {
        defer func() { <-sem }()  // освободить слот
        process(t)
    }(task)
}
```

**Разберём по шагам.**

#### Шаг 1: создание семафора

```go
sem := make(chan struct{}, 10)
```

**Что делает:** создаёт буферизованный канал на 10 элементов. **10 слотов.**

**Почему `struct{}`:** пустая структура, ноль байт. Не нужны данные — только сигнал.

#### Шаг 2: захват слота

```go
sem <- struct{}{}
```

**Что делает:**

- Если в буфере есть место — записывает и идёт дальше.
- Если буфер полон (все 10 слотов заняты) — **блокируется**, пока слот не освободится.

#### Шаг 3: освобождение слота

```go
defer func() { <-sem }()
```

**Что делает:** читает из канала, освобождая слот.

**Почему `defer`:** гарантирует освобождение даже при панике.

#### Шаг 4: полный пример

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	tasks := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	sem := make(chan struct{}, 3)  // 3 одновременных

	var wg sync.WaitGroup
	for _, task := range tasks {
		sem <- struct{}{}  // захватить слот
		wg.Add(1)
		go func(t int) {
			defer wg.Done()
			defer func() { <-sem }()  // освободить слот
			fmt.Printf("processing %d\n", t)
			time.Sleep(100 * time.Millisecond)
		}(task)
	}
	wg.Wait()
	fmt.Println("done")
}
```

**Что происходит:**

- 3 задачи обрабатываются одновременно.
- Остальные ждут на `sem <- struct{}{}`.
- После завершения задачи — слот освобождается, следующая берёт его.

### Семафор vs worker pool: когда что

| Аспект | Worker pool | Semaphore |
|:---|:---|:---|
| Горутин | N (фиксировано) | N + задачи |
| Создание горутин | N раз | На каждую задачу |
| Память | N × 2.3 КБ | (N + задачи) × 2.3 КБ |
| Переиспользование | Да | Нет |
| Простота | Средняя | Высокая |
| Гибкость | Ограничена | Высокая |

**Правило:**

- **Worker pool** — для многих однотипных задач.
- **Semaphore** — для ограничения доступа к ресурсу.

### Семафор для БД-соединений

**Классический пример:** ограничить число одновременных БД-запросов.

```go
type DB struct {
    sem chan struct{}
}

func NewDB(maxConns int) *DB {
    return &DB{sem: make(chan struct{}, maxConns)}
}

func (db *DB) Query(ctx context.Context, query string) (Result, error) {
    select {
    case db.sem <- struct{}{}:
        defer func() { <-db.sem }()
    case <-ctx.Done():
        return Result{}, ctx.Err()
    }
    return db.doQuery(ctx, query)
}
```

**Что происходит:**

- Не более `maxConns` одновременных запросов.
- Если все заняты — ждём или отменяем через `ctx`.

### Семафор с context

**Проблема:** `sem <- struct{}{}` **блокируется** и не отменяется.

**Решение:** `select` с `ctx.Done()`:

```go
select {
case sem <- struct{}{}:
    // захватили
case <-ctx.Done():
    return ctx.Err()  // отмена
}
```

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| Создание | ~50-100 нс | ~96 байт + буфер |
| Захват (есть место) | ~30-70 нс | 0 |
| Захват (нет места) | gopark | sudog |
| Освобождение | ~30-70 нс | 0 |

### 💡 Практика: как использовать семафор

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`make(chan struct{}, N)`** — семафор на N.
2. **`defer func() { <-sem }()`** — освобождение.
3. **`select` с `ctx.Done()`** — отмена.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Семафор для БД-соединений**, HTTP-клиентов, внешних API.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **`golang.org/x/sync/semaphore`** — для взвешенного семафора (см. 9.2).

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай освобождать слот.** Утечка.
7. **Не используй семафор для многих однотипных задач** — worker pool лучше.

### Ключевые выводы подглавы 9.1

- **Semaphore** — ограничивает число одновременных операций.
- **На каналах:** `make(chan struct{}, N)`.
- **Захват:** `sem <- struct{}{}`. **Освобождение:** `<-sem`.
- **С `context`:** `select` с `ctx.Done()`.
- **Worker pool** — для многих задач, **semaphore** — для ресурса.

---

## 9.2 Взвешенный семафор

**Взвешенный семафор** — семафор, где каждая операция захватывает **N слотов**, а не 1.

### Проблема: разные ресурсы

**Обычный семафор:** каждая операция захватывает **1 слот**.

**Но:** операции могут требовать **разное** количество ресурсов.

**Пример:**

- Лёгкий запрос: 1 единица CPU.
- Тяжёлый запрос: 5 единиц CPU.
- Всего: 10 единиц.

**Обычный семафор на 10:** лёгкий и тяжёлый конкурируют одинаково. Тяжёлый может «съесть» все ресурсы.

**Взвешенный семафор:** лёгкий захватывает 1, тяжёлый — 5. Всего 10.

### `golang.org/x/sync/semaphore`

**Пакет** `golang.org/x/sync/semaphore` даёт взвешенный семафор:

```go
import "golang.org/x/sync/semaphore"

sem := semaphore.NewWeighted(10)  // 10 единиц

// Захватить 5 единиц
if err := sem.Acquire(ctx, 5); err != nil {
    return err
}
defer sem.Release(5)
```

### Шаг 1: создание

```go
sem := semaphore.NewWeighted(10)
```

**Что делает:** создаёт семафор на 10 единиц.

### Шаг 2: захват

```go
if err := sem.Acquire(ctx, 5); err != nil {
    return err  // ctx отменён
}
```

**Что делает:**

- Пытается захватить 5 единиц.
- Если доступно — захватывает.
- Если нет — **ждёт**, пока освободятся.
- Если `ctx` отменён — возвращает ошибку.

### Шаг 3: освобождение

```go
sem.Release(5)
```

**Что делает:** освобождает 5 единиц.

### Шаг 4: полный пример

```go
package main

import (
	"context"
	"fmt"
	"time"

	"golang.org/x/sync/semaphore"
)

func main() {
	sem := semaphore.NewWeighted(10)
	ctx := context.Background()

	tasks := []struct {
		name   string
		weight int64
	}{
		{"light-1", 1},
		{"light-2", 1},
		{"heavy-1", 5},
		{"light-3", 1},
		{"heavy-2", 5},
		{"light-4", 1},
	}

	for _, task := range tasks {
		if err := sem.Acquire(ctx, task.weight); err != nil {
			fmt.Println("acquire failed:", err)
			return
		}
		go func(t struct {
			name   string
			weight int64
		}) {
			defer sem.Release(t.weight)
			fmt.Printf("processing %s (weight=%d)\n", t.name, t.weight)
			time.Sleep(100 * time.Millisecond)
		}(task)
	}
}
```

**Что происходит:**

- `light-1` (1) + `light-2` (1) + `heavy-1` (5) = 7 единиц.
- `light-3` (1) = 8 единиц.
- `heavy-2` (5) — **ждёт**, потому что 8 + 5 = 13 > 10.
- Когда `heavy-1` освободит 5 — `heavy-2` захватит.

### Как устроен взвешенный семафор

**Внутри** — дерево ожидающих горутин с весами:

```go
type Weighted struct {
    size    int64       // всего единиц
    cur     int64       // текущее использование
    mu      sync.Mutex  // защита
    waiters list.List   // список ждущих
}
```

**Что происходит при `Acquire`:**

1. Захватить `mu`.
2. Если `cur + n <= size` — увеличить `cur`, вернуть.
3. Иначе — добавить горутину в `waiters`, ждать.
4. При `Release` — проверить `waiters`, разбудить тех, кто может захватить.

### Сравнение с обычным семафором

| Аспект | Обычный | Взвешенный |
|:---|:---|:---|
| Единиц на операцию | 1 | N |
| Реализация | Канал | `x/sync/semaphore` |
| Стоимость | ~30-70 нс | ~100-200 нс |
| Гибкость | Низкая | Высокая |
| Когда | Однотипные операции | Разные веса |

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| `NewWeighted` | ~50-100 нс | ~50 байт |
| `Acquire` (есть место) | ~100-200 нс | 0 |
| `Acquire` (нет места) | gopark | sudog |
| `Release` | ~100-200 нс | 0 |

### 💡 Практика: как использовать взвешенный семафор

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`semaphore.NewWeighted(N)`** — для N единиц.
2. **`sem.Acquire(ctx, weight)`** — захват с весом.
3. **`defer sem.Release(weight)`** — освобождение.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Взвешенный семафор для разнородных задач** (лёгкие + тяжёлые).

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **Обычный семафор на каналах** — для однотипных.

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `Release`.** Утечка единиц.
7. **Не используй взвешенный семафор, если веса одинаковы.**

### Ключевые выводы подглавы 9.2

- **Взвешенный семафор** — каждая операция захватывает N единиц.
- **`golang.org/x/sync/semaphore`** — стандартная реализация.
- **`Acquire(ctx, weight)`** / **`Release(weight)`**.
- **Для разнородных задач.**
- **Стоимость выше обычного семафора.**

---

## 9.3 Rate limiter: ограничение скорости

**Rate limiter** — примитив, который ограничивает **скорость** операций (сколько за период).

### Rate limiter vs semaphore

| Аспект | Semaphore | Rate limiter |
|:---|:---|:---|
| Что ограничивает | Параллелизм | Скорость |
| Единица | Одновременные операции | Операции в секунду |
| Пример | 10 одновременных запросов | 100 запросов/сек |
| Когда | Ограничить ресурс | Ограничить внешний API |

**Ключевое:** семафор ограничивает **сколько одновременно**. Rate limiter — **сколько за период**.

### Простейший rate limiter: `time.Tick`

**Идея:** использовать `time.Tick` как источник «разрешений».

```go
func rateLimiter(rps int) <-chan time.Time {
    interval := time.Second / time.Duration(rps)
    return time.Tick(interval)
}
```

**Использование:**

```go
limiter := rateLimiter(100)  // 100 разрешений/сек

for _, task := range tasks {
    <-limiter  // ждём разрешения
    process(task)
}
```

**Что происходит:**

- `time.Tick` создаёт канал, в который **каждые 10 мс** (для 100/сек) отправляется текущее время.
- Каждое `<-limiter` ждёт очередного «тика».
- **Скорость:** 100 операций/сек.

**Проблемы:**

- **Нет burst.** Нельзя «накопить» разрешения.
- **`time.Tick` не останавливается.** Утечка.
- **Точность.** `time.Second / rps` может быть 0 при больших rps (например, 1 000 000/сек).

**Когда использовать:** простые случаи, без burst.

### Более точный rate limiter: `time.NewTicker`

**`time.Tick`** — упрощение `time.NewTicker`. Но `Tick` не даёт остановить тикер.

```go
func rateLimiter(rps int) (<-chan time.Time, func()) {
    interval := time.Second / time.Duration(rps)
    ticker := time.NewTicker(interval)
    return ticker.C, ticker.Stop
}
```

**Использование:**

```go
limiter, stop := rateLimiter(100)
defer stop()  // ← останавливаем тикер

for _, task := range tasks {
    <-limiter
    process(task)
}
```

**Что изменилось:** `stop()` освобождает ресурсы тикера.

**Проблемы:**

- **Нет burst.** Всё ещё нет.
- **Точность.** Та же проблема с делением.

### Rate limiter с burst: token bucket на каналах

**Идея:** использовать буферизованный канал как «ведро с токенами».

```go
func tokenBucket(rps int, burst int) (chan struct{}, func()) {
    tokens := make(chan struct{}, burst)
    stop := make(chan struct{})

    // Наполняем ведро начальными токенами (burst)
    for i := 0; i < burst; i++ {
        tokens <- struct{}{}
    }

    // Добавляем токены со скоростью rps
    go func() {
        interval := time.Second / time.Duration(rps)
        ticker := time.NewTicker(interval)
        defer ticker.Stop()
        for {
            select {
            case <-stop:
                return
            case <-ticker.C:
                select {
                case tokens <- struct{}{}:  // добавить токен
                default:                     // ведро полно — пропустить
                }
            }
        }
    }()

    return tokens, func() { close(stop) }
}
```

**Использование:**

```go
tokens, stop := tokenBucket(100, 10)
defer stop()

for _, task := range tasks {
    <-tokens  // забрать токен (блокируется, если токенов нет)
    process(task)
}
```

**Что происходит:**

1. Ведро на `burst` токенов.
2. Изначально заполнено `burst` токенами.
3. Каждые `1/rps` секунд добавляется 1 токен (если есть место).
4. `<-tokens` забирает токен. Если нет — блокируется.
5. **Burst** — можно забрать до `burst` токенов сразу.

**Разберём по шагам.**

#### Шаг 1: создание ведра

```go
tokens := make(chan struct{}, burst)
```

**Что делает:** буферизованный канал на `burst` элементов — «ведро с токенами».

#### Шаг 2: начальное заполнение

```go
for i := 0; i < burst; i++ {
    tokens <- struct{}{}
}
```

**Что делает:** заполняет ведро `burst` токенами. **Burst — можно забрать сразу.**

#### Шаг 3: добавление токенов

```go
go func() {
    interval := time.Second / time.Duration(rps)
    ticker := time.NewTicker(interval)
    defer ticker.Stop()
    for {
        select {
        case <-stop:
            return
        case <-ticker.C:
            select {
            case tokens <- struct{}{}:
            default:
            }
        }
    }
}()
```

**Что делает:**

- Каждые `interval` секунд (1/rps) добавляет токен.
- Если ведро полно — токен **пропускается** (не блокируемся).
- Останавливается по `stop`.

#### Шаг 4: забор токена

```go
<-tokens
```

**Что делает:** забирает токен из ведра. Если пусто — блокируется.

#### Шаг 5: полный пример

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	tokens, stop := tokenBucket(10, 5)  // 10/сек, burst 5
	defer stop()

	start := time.Now()
	for i := 0; i < 20; i++ {
		<-tokens  // ждём токен
		fmt.Printf("[%v] request %d\n", time.Since(start).Round(time.Millisecond), i)
	}
}

func tokenBucket(rps int, burst int) (chan struct{}, func()) {
	tokens := make(chan struct{}, burst)
	stop := make(chan struct{})

	for i := 0; i < burst; i++ {
		tokens <- struct{}{}
	}

	go func() {
		interval := time.Second / time.Duration(rps)
		ticker := time.NewTicker(interval)
		defer ticker.Stop()
		for {
			select {
			case <-stop:
				return
			case <-ticker.C:
				select {
				case tokens <- struct{}{}:
				default:
				}
			}
		}
	}()

	return tokens, func() { close(stop) }
}
```

**Пример вывода:**

```
[0s] request 0
[0s] request 1
[0s] request 2
[0s] request 3
[0s] request 4
[100ms] request 5
[200ms] request 6
[300ms] request 7
...
```

**Что видно:**

- Первые 5 запросов — **мгновенно** (burst).
- Каждый следующий — с интервалом 100 мс (10/сек).
- За 20 запросов прошло ~1.5 секунды.

### Ограничения нашего rate limiter

**1. Точность при высоких rps.**

`time.Second / time.Duration(rps)` при `rps = 1 000 000` даст 1000 наносекунд. Но `time.NewTicker` имеет разрешение ~1 мс. Для высоких rps — неточно.

**2. Накопление токенов.**

Если потребитель не забирает токены долго — ведро заполняется до `burst` и **перестаёт принимать новые**. Это **правильное** поведение для token bucket, но может быть неожиданным.

**3. Нет `Allow()`.**

Наш rate limiter **блокирующий**. Нет неблокирующего варианта.

**4. Нет `WaitN()`.**

Нельзя забрать N токенов за раз.

**Для production** используй `golang.org/x/time/rate` — он решает все эти проблемы.

### `golang.org/x/time/rate`

**Пакет** `golang.org/x/time/rate` — production-ready rate limiter.

```go
import "golang.org/x/time/rate"

limiter := rate.NewLimiter(rate.Limit(100), 10)  // 100/сек, burst 10
```

**Что значит:**

- **100** — 100 событий в секунду.
- **10** — burst: можно «накопить» до 10 событий.

### Шаг 1: создание лимитера

```go
limiter := rate.NewLimiter(rate.Limit(100), 10)
```

**Что делает:** создаёт лимитер на 100 событий/сек с burst 10.

### Шаг 2: ожидание разрешения

```go
if err := limiter.Wait(ctx); err != nil {
    return err  // ctx отменён
}
// разрешение получено
```

**Что делает:** ждёт, пока лимитер разрешит операцию.

### Шаг 3: полный пример

```go
package main

import (
	"context"
	"fmt"
	"time"

	"golang.org/x/time/rate"
)

func main() {
	limiter := rate.NewLimiter(rate.Limit(10), 5)  // 10/сек, burst 5
	ctx := context.Background()

	start := time.Now()
	for i := 0; i < 20; i++ {
		if err := limiter.Wait(ctx); err != nil {
			fmt.Println("wait failed:", err)
			return
		}
		fmt.Printf("[%v] request %d\n", time.Since(start).Round(time.Millisecond), i)
	}
}
```

**Пример вывода:**

```
[0s] request 0
[0s] request 1
[0s] request 2
[0s] request 3
[0s] request 4
[100ms] request 5
[200ms] request 6
...
```

**То же поведение, что у нашего rate limiter, но с `context` и без утечек.**

### Шаг 4: `Allow` вместо `Wait`

```go
if limiter.Allow() {
	// разрешено
} else {
	// отклонено
}
```

**Что делает:** **не блокируется**. Возвращает `true`, если разрешено, `false` — если нет.

**Когда использовать:** если можно **дропнуть** запрос (не критичный).

### Шаг 5: `Reserve` для планирования

```go
r := limiter.Reserve()
if !r.OK() {
	return
}
delay := r.Delay()
time.Sleep(delay)
```

**Что делает:** резервирует слот и возвращает задержку до его активации.

### Шаг 6: rate limiter для HTTP-клиента

```go
type RateLimitedClient struct {
	client  *http.Client
	limiter *rate.Limiter
}

func NewRateLimitedClient(rps int, burst int) *RateLimitedClient {
	return &RateLimitedClient{
		client:  &http.Client{Timeout: 10 * time.Second},
		limiter: rate.NewLimiter(rate.Limit(rps), burst),
	}
}

func (c *RateLimitedClient) Get(ctx context.Context, url string) (*http.Response, error) {
	if err := c.limiter.Wait(ctx); err != nil {
		return nil, err
	}
	req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
	if err != nil {
		return nil, err
	}
	return c.client.Do(req)
}
```

**Что происходит:** каждый запрос ждёт разрешения лимитера.

### Burst: зачем нужен

**Burst** — максимальное количество событий, которые можно «накопить».

**Пример:** лимитер 100/сек, burst 10.

- Если 5 секунд не было запросов — накопилось 500? **Нет**, только 10.
- Первые 10 запросов — **мгновенно**.
- Дальше — 100/сек.

**Зачем burst:** для **пиков**. Если иногда нужно 10 запросов подряд — burst 10.

**Размер burst:**

- **Маленький (1-5)** — строгое ограничение.
- **Средний (10-100)** — для пиков.
- **Большой (1000+)** — почти не ограничивает.

### Наш rate limiter vs `x/time/rate`

| Аспект | Наш (token bucket) | `x/time/rate` |
|:---|:---|:---|
| Burst | ✅ | ✅ |
| `context` | ❌ | ✅ |
| `Allow()` | ❌ | ✅ |
| `WaitN()` | ❌ | ✅ |
| Точность при высоких rps | Ограничена | Высокая |
| Стоимость | ~100-200 нс | ~100-200 нс |
| Production-ready | ❌ | ✅ |

**Вывод:** наш rate limiter — для **понимания**. Для production — `x/time/rate`.

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| `NewLimiter` | ~50-100 нс | ~50 байт |
| `Wait` (есть токен) | ~100-200 нс | 0 |
| `Wait` (нет токена) | gopark + таймер | sudog |
| `Allow` | ~50-100 нс | 0 |
| Наш token bucket: забор | ~30-70 нс | 0 |
| Наш token bucket: добавление | ~100-200 нс | 0 |

### 💡 Практика: как использовать rate limiter

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`rate.NewLimiter(rps, burst)`** — для production.
2. **`limiter.Wait(ctx)`** — перед каждым запросом.
3. **Burst 10-100** — для большинства случаев.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Свой token bucket** — для понимания и обучения.
5. **`Allow()`** — для неблокирующей проверки.
6. **Мониторинг** — сколько запросов пропущено/отклонено.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **`Reserve()`** — для точного планирования.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй `time.Tick` в production.** Утечка.
9. **Не ставь rate limiter без burst** (burst=0).
10. **Не используй свой rate limiter в production** — `x/time/rate` лучше.
11. **Не игнорируй ошибки `Wait`.** `ctx.Err()`.

### Ключевые выводы подглавы 9.3

- **Rate limiter** — ограничивает скорость (событий/сек).
- **`time.Tick`** — простейший, но с утечкой.
- **Token bucket на каналах** — burst + точность.
- **`golang.org/x/time/rate`** — production-ready.
- **`NewLimiter(rps, burst)`** — создание.
- **`Wait(ctx)`** — ожидание. **`Allow()`** — неблокирующая проверка.
- **Burst** — запас на пики.

---

## 9.4 Rate limiter с нуля: token bucket

Разберём **token bucket с нуля** — подробно, по шагам.

### Идея token bucket

**Ведро с токенами:**

- Токены **добавляются** со скоростью `rate` (токенов/сек).
- Ведро **вмещает** максимум `burst` токенов.
- Каждый запрос **забирает** 1 токен.
- Если токенов нет — запрос **ждёт**.

**Визуализация:**

```
┌─────────────────────────────────────┐
│         Token Bucket                │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  🪙 🪙 🪙 🪙 🪙              │    │  ← ведро (burst=5)
│  └─────────────────────────────┘    │
│         ▲                  │        │
│         │                  │        │
│    добавление          забор токена │
│    (rate/сек)          (запрос)     │
│                                     │
└─────────────────────────────────────┘
```

**Что происходит:**

1. Каждые `1/rate` секунд добавляется токен.
2. Если ведро полно (`burst` токенов) — токен **пропускается**.
3. Запрос забирает токен.
4. Если токенов нет — запрос **блокируется**.

### Реализация на каналах

**Канал как ведро:**

- **Буфер канала** = `burst` (размер ведра).
- **Элементы** = токены.
- **Забор токена** = чтение из канала.
- **Добавление токена** = запись в канал.

```go
func tokenBucket(rps int, burst int) (chan struct{}, func()) {
	tokens := make(chan struct{}, burst)
	stop := make(chan struct{})

	for i := 0; i < burst; i++ {
		tokens <- struct{}{}
	}

	go func() {
		interval := time.Second / time.Duration(rps)
		ticker := time.NewTicker(interval)
		defer ticker.Stop()
		for {
			select {
			case <-stop:
				return
			case <-ticker.C:
				select {
				case tokens <- struct{}{}:
				default:
				}
			}
		}
	}()

	return tokens, func() { close(stop) }
}
```

### Разберём по шагам

#### Шаг 1: создание ведра

```go
tokens := make(chan struct{}, burst)
```

**Что делает:** буферизованный канал на `burst` элементов.

**Почему `struct{}`:** пустая структура, ноль байт. Токены — это просто сигналы.

#### Шаг 2: остановка

```go
stop := make(chan struct{})
```

**Что делает:** канал для остановки горутины-наполнителя.

#### Шаг 3: начальное заполнение

```go
for i := 0; i < burst; i++ {
	tokens <- struct{}{}
}
```

**Что делает:** заполняет ведро `burst` токенами.

**Зачем:** burst — можно забрать сразу.

#### Шаг 4: горутина-наполнитель

```go
go func() {
	interval := time.Second / time.Duration(rps)
	ticker := time.NewTicker(interval)
	defer ticker.Stop()
	for {
		select {
		case <-stop:
			return
		case <-ticker.C:
			select {
			case tokens <- struct{}{}:
			default:
			}
		}
	}
}()
```

**Что делает:**

- Создаёт тикер с интервалом `1/rps` секунд.
- Каждый тик — добавляет токен.
- Если ведро полно — **пропускает** (не блокируется).
- Останавливается по `stop`.

**Почему `select` с `default` при добавлении:**

- `tokens <- struct{}{}` блокируется, если буфер полон.
- `default` — если полон, пропускаем.
- **Результат:** ведро не переполняется.

#### Шаг 5: забор токена

```go
<-tokens
```

**Что делает:** забирает токен. Если пусто — блокируется.

#### Шаг 6: остановка

```go
func() { close(stop) }
```

**Что делает:** закрывает `stop`, горутина-наполнитель завершается.

### Полный пример

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	tokens, stop := tokenBucket(10, 5)  // 10/сек, burst 5
	defer stop()

	start := time.Now()
	for i := 0; i < 20; i++ {
		<-tokens
		fmt.Printf("[%v] request %d\n", time.Since(start).Round(time.Millisecond), i)
	}
}

func tokenBucket(rps int, burst int) (chan struct{}, func()) {
	tokens := make(chan struct{}, burst)
	stop := make(chan struct{})

	for i := 0; i < burst; i++ {
		tokens <- struct{}{}
	}

	go func() {
		interval := time.Second / time.Duration(rps)
		ticker := time.NewTicker(interval)
		defer ticker.Stop()
		for {
			select {
			case <-stop:
				return
			case <-ticker.C:
				select {
				case tokens <- struct{}{}:
				default:
				}
			}
		}
	}()

	return tokens, func() { close(stop) }
}
```

**Пример вывода:**

```
[0s] request 0
[0s] request 1
[0s] request 2
[0s] request 3
[0s] request 4
[100ms] request 5
[200ms] request 6
[300ms] request 7
[400ms] request 8
[500ms] request 9
...
```

**Что видно:**

- Первые 5 запросов — мгновенно (burst).
- Дальше — каждые 100 мс (10/сек).

### Вариант: неблокирующий забор

**Идея:** добавить `Allow()` — неблокирующая проверка.

```go
func (tb *TokenBucket) Allow() bool {
	select {
	case <-tb.tokens:
		return true  // токен был — забрали
	default:
		return false  // токенов нет — не блокируемся
	}
}
```

**Использование:**

```go
if tb.Allow() {
	// разрешено
} else {
	// отклонено — можно дропнуть или повторить позже
}
```

### Оформление в структуру

Для удобства обернём в структуру:

```go
type TokenBucket struct {
	tokens chan struct{}
	stop   chan struct{}
	once   sync.Once
}

func NewTokenBucket(rps int, burst int) *TokenBucket {
	tb := &TokenBucket{
		tokens: make(chan struct{}, burst),
		stop:   make(chan struct{}),
	}

	for i := 0; i < burst; i++ {
		tb.tokens <- struct{}{}
	}

	go tb.fill(rps)

	return tb
}

func (tb *TokenBucket) fill(rps int) {
	interval := time.Second / time.Duration(rps)
	ticker := time.NewTicker(interval)
	defer ticker.Stop()
	for {
		select {
		case <-tb.stop:
			return
		case <-ticker.C:
			select {
			case tb.tokens <- struct{}{}:
			default:
			}
		}
	}
}

func (tb *TokenBucket) Wait(ctx context.Context) error {
	select {
	case <-tb.tokens:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func (tb *TokenBucket) Allow() bool {
	select {
	case <-tb.tokens:
		return true
	default:
		return false
	}
}

func (tb *TokenBucket) Close() {
	tb.once.Do(func() {
		close(tb.stop)
	})
}
```

**Что добавлено:**

- **`Wait(ctx)`** — блокирующий забор с `context`.
- **`Allow()`** — неблокирующий забор.
- **`Close()`** — остановка горутины-наполнителя (через `sync.Once` — идемпотентно).

**Полный пример:**

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

type TokenBucket struct {
	tokens chan struct{}
	stop   chan struct{}
	once   sync.Once
}

func NewTokenBucket(rps int, burst int) *TokenBucket {
	tb := &TokenBucket{
		tokens: make(chan struct{}, burst),
		stop:   make(chan struct{}),
	}

	for i := 0; i < burst; i++ {
		tb.tokens <- struct{}{}
	}

	go tb.fill(rps)

	return tb
}

func (tb *TokenBucket) fill(rps int) {
	interval := time.Second / time.Duration(rps)
	ticker := time.NewTicker(interval)
	defer ticker.Stop()
	for {
		select {
		case <-tb.stop:
			return
		case <-ticker.C:
			select {
			case tb.tokens <- struct{}{}:
			default:
			}
		}
	}
}

func (tb *TokenBucket) Wait(ctx context.Context) error {
	select {
	case <-tb.tokens:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func (tb *TokenBucket) Allow() bool {
	select {
	case <-tb.tokens:
		return true
	default:
		return false
	}
}

func (tb *TokenBucket) Close() {
	tb.once.Do(func() {
		close(tb.stop)
	})
}

func main() {
	tb := NewTokenBucket(10, 5)
	defer tb.Close()

	ctx := context.Background()
	start := time.Now()

	for i := 0; i < 20; i++ {
		if err := tb.Wait(ctx); err != nil {
			fmt.Println("wait failed:", err)
			return
		}
		fmt.Printf("[%v] request %d\n", time.Since(start).Round(time.Millisecond), i)
	}
}
```

### Ограничения нашей реализации

**1. Точность при высоких rps.**

`time.Second / time.Duration(rps)` при `rps = 1 000 000` даст 1000 наносекунд. Но `time.NewTicker` имеет разрешение ~1 мс. Для высоких rps — **неточно**.

**2. Накопление токенов.**

Ведро заполняется до `burst` и **перестаёт принимать**. Это **правильно** для token bucket, но нужно понимать.

**3. Нет `WaitN()`.**

Нельзя забрать N токенов за раз. Для этого нужен `x/time/rate`.

**4. Нет `Reserve()`.**

Нельзя запланировать забор на будущее.

### Что даёт наша реализация

**Понимание:**

- Как работает token bucket.
- Как использовать каналы для rate limiting.
- Как комбинировать `time.Ticker` и каналы.

**Для production** — используй `x/time/rate`. Но **понимание** важно для отладки и выбора параметров.

### Сравнение с `x/time/rate`

| Аспект | Наш | `x/time/rate` |
|:---|:---|:---|
| Burst | ✅ | ✅ |
| `context` | ✅ (Wait) | ✅ |
| `Allow()` | ✅ | ✅ |
| `WaitN()` | ❌ | ✅ |
| `Reserve()` | ❌ | ✅ |
| Точность при высоких rps | Ограничена | Высокая |
| Production-ready | ❌ | ✅ |
| Стоимость | ~100-200 нс | ~100-200 нс |

### Аннотация сложности

| Операция | Наш | `x/time/rate` |
|:---|:---|:---|
| Создание | ~200-500 нс | ~50-100 нс |
| Забор токена (есть) | ~30-70 нс | ~100-200 нс |
| Забор токена (нет) | gopark | gopark |
| `Allow()` | ~50-100 нс | ~50-100 нс |
| Память | ~150 байт + буфер | ~50 байт |

### 💡 Практика: как использовать token bucket

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Свой token bucket** — для обучения и понимания.
2. **`x/time/rate`** — для production.
3. **Burst 10-100** — для большинства случаев.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Оформляй в структуру** — удобнее.
5. **`Wait(ctx)`** — с `context`.
6. **`Allow()`** — для неблокирующей проверки.
7. **`Close()`** — для остановки горутины-наполнителя.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

8. **`WaitN()`** — если нужен забор N токенов.

**❌ НЕ ДЕЛАЙ:**

9. **Не используй свой rate limiter в production.** `x/time/rate` лучше.
10. **Не забывай `Close()`.** Горутина-наполнитель — утечка.
11. **Не используй `time.Tick`** — утечка.

### Ключевые выводы подглавы 9.4

- **Token bucket** — ведро с токенами, добавляются со скоростью `rate`, максимум `burst`.
- **На каналах:** буфер канала = `burst`, токены = элементы.
- **Наполнитель** — `time.NewTicker` + горутина.
- **`Wait(ctx)`** — блокирующий забор. **`Allow()`** — неблокирующий.
- **`Close()`** — остановка наполнителя.
- **Для production** — `x/time/rate`.

---

## 9.5 Token bucket vs leaky bucket

Разберём **два алгоритма** rate limiting.

### Token bucket

**Идея:** есть **ведро с токенами**. Токены **добавляются** с постоянной скоростью. Запрос **забирает** токен. Если токенов нет — запрос ждёт.

**Параметры:**

- **Rate:** сколько токенов добавляется в секунду.
- **Burst:** размер ведра (максимум токенов).

**Схема:**

```
┌─────────────────────────────────────┐
│         Token Bucket                │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  🪙 🪙 🪙 🪙 🪙              │    │
│  │  (токены)                    │    │
│  └─────────────────────────────┘    │
│         ▲                  │        │
│         │                  │        │
│    добавление          забор токена │
│    (rate/сек)          (запрос)     │
│                                     │
└─────────────────────────────────────┘
```

**Что происходит:**

1. Токены добавляются со скоростью `rate`.
2. Ведро максимум `burst` токенов.
3. Запрос забирает токен.
4. Если токенов нет — запрос ждёт (или дропается).

**Особенности:**

- **Разрешает пики** до `burst`.
- **Средняя скорость** = `rate`.
- **`golang.org/x/time/rate`** — token bucket.

### Leaky bucket

**Идея:** есть **ведро с водой**. Вода **вытекает** с постоянной скоростью. Запрос **наливает** воду. Если ведро полно — запрос ждёт (или дропается).

**Параметры:**

- **Rate:** сколько вытекает в секунду.
- **Burst:** размер ведра.

**Схема:**

```
┌─────────────────────────────────────┐
│         Leaky Bucket                │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  💧 💧 💧 💧 💧              │    │
│  │  (вода)                      │    │
│  └─────────────────────────────┘    │
│         ▲                  │        │
│         │                  │        │
│    налив запроса       вытекание    │
│                        (rate/сек)   │
│                                     │
└─────────────────────────────────────┘
```

**Что происходит:**

1. Вода вытекает со скоростью `rate`.
2. Запрос наливает воду.
3. Если ведро полно — запрос ждёт.

**Особенности:**

- **Не разрешает пики** (в отличие от token bucket).
- **Сглаживает** поток.
- **Средняя скорость** = `rate`.

### Ключевое различие

**Token bucket:** разрешает **пики** до `burst`. Запросы могут «накопить» токены и потратить их сразу.

**Leaky bucket:** **сглаживает** поток. Запросы выходят с постоянной скоростью, без пиков.

**Визуализация:**

```
Token bucket (rate=10, burst=10):

  Вход:  ████████████████████████ (всплеск 20 запросов)
  Выход: ██████████░░░░██████████ (10 сразу, потом 10)
         (пик разрешён)

Leaky bucket (rate=10, burst=10):

  Вход:  ████████████████████████ (всплеск 20 запросов)
  Выход: ██░░██░░██░░██░░██░░██░░ (равномерно)
         (пик сглажен)
```

### Когда что использовать

| Ситуация | Алгоритм |
|:---|:---|
| Разрешить пики | Token bucket |
| Сгладить поток | Leaky bucket |
| HTTP API с лимитом | Token bucket |
| Сетевой трафик | Leaky bucket |
| Go-стандарт | Token bucket (`x/time/rate`) |

**В Go** `golang.org/x/time/rate` — **token bucket**. Leaky bucket нужно реализовывать вручную (редко нужно).

### Аннотация сложности

| Алгоритм | Time (per request) | Space |
|:---|:---|:---|
| Token bucket | ~100-200 нс | ~50 байт |
| Leaky bucket | ~100-200 нс | ~50 байт |

### 💡 Практика: как выбирать алгоритм

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Token bucket** — для большинства случаев.
2. **`golang.org/x/time/rate`** — token bucket из коробки.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Leaky bucket** — только если нужно **строгое сглаживание**.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

4. **Своя реализация** — редко нужно.

**❌ НЕ ДЕЛАЙ:**

5. **Не путай token bucket и leaky bucket.** Разные цели.
6. **Не используй leaky bucket, если нужны пики.**

### Ключевые выводы подглавы 9.5

- **Token bucket** — разрешает пики до `burst`.
- **Leaky bucket** — сглаживает поток.
- **Go-стандарт** — token bucket (`x/time/rate`).
- **Token bucket** — для HTTP API. **Leaky bucket** — для сетевого трафика.
- **Средняя скорость** — `rate` у обоих.

---

## 9.6 Circuit breaker: защита от каскадных отказов

**Circuit breaker** — паттерн, который **прекращает** вызовы к сломанному сервису.

### Проблема: каскадные отказы

Представь:

- Сервис A вызывает сервис B.
- Сервис B начал тормозить (5 сек на запрос).
- Сервис A ждёт 5 сек на каждый запрос.
- Горутины A копятся.
- Через минуту A падает с OOM.

**Каскадный отказ:** отказ B приводит к отказу A, потом C, D...

### Решение: circuit breaker

**Идея:** если сервис **часто падает** — **перестать** к нему обращаться на время.

**Три состояния:**

1. **Closed** (закрыт) — все запросы проходят.
2. **Open** (открыт) — все запросы **отклоняются** немедленно.
3. **Half-Open** (полуоткрыт) — **пробные** запросы.

### Схема

```
                  Ошибок > threshold
     ┌────────┐  ─────────────────▶  ┌────────┐
     │ Closed │                       │  Open  │
     └────────┘  ◀─────────────────  └────────┘
         ▲          Успех                │
         │                               │
         │                          timeout
         │                               │
         │         ┌──────────┐          │
         └─────────│Half-Open │◀─────────┘
           Успех   └──────────┘
                        │
                   Ошибка
                        │
                        ▼
                   ┌────────┐
                   │  Open  │
                   └────────┘
```

### Шаг 1: структура

```go
type State int

const (
	StateClosed State = iota
	StateOpen
	StateHalfOpen
)

type CircuitBreaker struct {
	mu              sync.Mutex
	state           State
	failures        int
	successes       int
	lastFailureTime time.Time
	threshold       int
	timeout         time.Duration
	halfOpenMax     int
}
```

### Шаг 2: создание

```go
func NewCircuitBreaker(threshold int, timeout time.Duration) *CircuitBreaker {
	return &CircuitBreaker{
		state:       StateClosed,
		threshold:   threshold,
		timeout:     timeout,
		halfOpenMax: 1,
	}
}
```

### Шаг 3: вызов

```go
func (cb *CircuitBreaker) Call(fn func() error) error {
	if err := cb.beforeCall(); err != nil {
		return err
	}
	err := fn()
	cb.afterCall(err)
	return err
}
```

### Шаг 4: beforeCall

```go
func (cb *CircuitBreaker) beforeCall() error {
	cb.mu.Lock()
	defer cb.mu.Unlock()

	switch cb.state {
	case StateClosed:
		return nil
	case StateOpen:
		if time.Since(cb.lastFailureTime) > cb.timeout {
			cb.state = StateHalfOpen
			cb.successes = 0
			return nil
		}
		return ErrCircuitOpen
	case StateHalfOpen:
		if cb.successes >= cb.halfOpenMax {
			return ErrCircuitOpen
		}
		return nil
	}
	return nil
}
```

### Шаг 5: afterCall

```go
func (cb *CircuitBreaker) afterCall(err error) {
	cb.mu.Lock()
	defer cb.mu.Unlock()

	if err != nil {
		cb.failures++
		cb.lastFailureTime = time.Now()
		if cb.failures >= cb.threshold {
			cb.state = StateOpen
		}
		if cb.state == StateHalfOpen {
			cb.state = StateOpen
		}
	} else {
		cb.failures = 0
		if cb.state == StateHalfOpen {
			cb.successes++
			if cb.successes >= cb.halfOpenMax {
				cb.state = StateClosed
				cb.successes = 0
			}
		}
	}
}
```

### Шаг 6: полный пример

```go
package main

import (
	"errors"
	"fmt"
	"sync"
	"time"
)

var ErrCircuitOpen = errors.New("circuit breaker is open")

type State int

const (
	StateClosed State = iota
	StateOpen
	StateHalfOpen
)

type CircuitBreaker struct {
	mu              sync.Mutex
	state           State
	failures        int
	successes       int
	lastFailureTime time.Time
	threshold       int
	timeout         time.Duration
	halfOpenMax     int
}

func NewCircuitBreaker(threshold int, timeout time.Duration) *CircuitBreaker {
	return &CircuitBreaker{
		state:       StateClosed,
		threshold:   threshold,
		timeout:     timeout,
		halfOpenMax: 1,
	}
}

func (cb *CircuitBreaker) Call(fn func() error) error {
	if err := cb.beforeCall(); err != nil {
		return err
	}
	err := fn()
	cb.afterCall(err)
	return err
}

func (cb *CircuitBreaker) beforeCall() error {
	cb.mu.Lock()
	defer cb.mu.Unlock()

	switch cb.state {
	case StateClosed:
		return nil
	case StateOpen:
		if time.Since(cb.lastFailureTime) > cb.timeout {
			cb.state = StateHalfOpen
			cb.successes = 0
			return nil
		}
		return ErrCircuitOpen
	case StateHalfOpen:
		if cb.successes >= cb.halfOpenMax {
			return ErrCircuitOpen
		}
		return nil
	}
	return nil
}

func (cb *CircuitBreaker) afterCall(err error) {
	cb.mu.Lock()
	defer cb.mu.Unlock()

	if err != nil {
		cb.failures++
		cb.lastFailureTime = time.Now()
		if cb.failures >= cb.threshold {
			cb.state = StateOpen
		}
		if cb.state == StateHalfOpen {
			cb.state = StateOpen
		}
	} else {
		cb.failures = 0
		if cb.state == StateHalfOpen {
			cb.successes++
			if cb.successes >= cb.halfOpenMax {
				cb.state = StateClosed
				cb.successes = 0
			}
		}
	}
}

func main() {
	cb := NewCircuitBreaker(3, 1*time.Second)

	for i := 0; i < 10; i++ {
		err := cb.Call(func() error {
			return errors.New("service unavailable")
		})
		fmt.Printf("Call %d: %v\n", i, err)
		time.Sleep(200 * time.Millisecond)
	}

	time.Sleep(1 * time.Second)

	err := cb.Call(func() error {
		return nil
	})
	fmt.Printf("Recovery: %v\n", err)
}
```

**Пример вывода:**

```
Call 0: service unavailable
Call 1: service unavailable
Call 2: service unavailable
Call 3: circuit breaker is open
Call 4: circuit breaker is open
...
Call 9: circuit breaker is open
Recovery: <nil>
```

### Библиотеки

**Готовые реализации:**

- **`github.com/sony/gobreaker`** — простая.
- **`github.com/afex/hystrix-go`** — Netflix Hystrix.

**Использование `gobreaker`:**

```go
import "github.com/sony/gobreaker"

cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
	Name:        "my-service",
	MaxRequests: 3,
	Interval:    10 * time.Second,
	Timeout:     30 * time.Second,
	ReadyToTrip: func(counts gobreaker.Counts) bool {
		return counts.ConsecutiveFailures > 5
	},
})

result, err := cb.Execute(func() (interface{}, error) {
	return http.Get("https://api.example.com")
})
```

### Аннотация сложности

| Операция | Time | Space |
|:---|:---|:---|
| `Call` (Closed) | ~50-100 нс + fn | 0 |
| `Call` (Open) | ~50-100 нс | 0 |
| `Call` (Half-Open) | ~50-100 нс + fn | 0 |

### 💡 Практика: как использовать circuit breaker

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Circuit breaker для внешних сервисов.**
2. **Threshold 5-10 ошибок.**
3. **Timeout 10-60 секунд.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`gobreaker`** — готовая библиотека.
5. **Метрики:** сколько раз открывался.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Своя реализация** — если нужна специфичная логика.

**❌ НЕ ДЕЛАЙ:**

7. **Не ставь threshold 1.** Ложные срабатывания.
8. **Не ставь timeout 1 секунду.** Сервис не успеет восстановиться.
9. **Не используй circuit breaker для внутренних вызовов** (без сети).

### Ключевые выводы подглавы 9.6

- **Circuit breaker** — защищает от каскадных отказов.
- **Три состояния:** Closed, Open, Half-Open.
- **Closed:** пропускает. **Open:** отклоняет. **Half-Open:** пробные.
- **Threshold 5-10, timeout 10-60 сек.**
- **Библиотеки:** `gobreaker`, `hystrix-go`.

---

## 9.7 Future/promise и generator

Разберём **два паттерна**: future/promise и generator.

### Future/promise

**Future** — объект, который представляет **результат**, который **ещё не готов**.

**Реализация в Go:**

```go
type Future[T any] struct {
	result chan T
	err    chan error
}

func NewFuture[T any](fn func() (T, error)) *Future[T] {
	f := &Future[T]{
		result: make(chan T, 1),
		err:    make(chan error, 1),
	}
	go func() {
		defer close(f.result)
		defer close(f.err)
		r, e := fn()
		if e != nil {
			f.err <- e
			return
		}
		f.result <- r
	}()
	return f
}

func (f *Future[T]) Get(ctx context.Context) (T, error) {
	var zero T
	select {
	case <-ctx.Done():
		return zero, ctx.Err()
	case err := <-f.err:
		return zero, err
	case r := <-f.result:
		return r, nil
	}
}
```

**Использование:**

```go
f := NewFuture(func() (string, error) {
	time.Sleep(1 * time.Second)
	return "result", nil
})

result, err := f.Get(context.Background())
fmt.Println(result, err)
```

### Generator

**Generator** — функция, которая **отдаёт значения по одному**.

**В Go** — через канал:

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

**Генератор бесконечной последовательности:**

```go
func infiniteCounter(ctx context.Context) chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for i := 0; ; i++ {
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

### Сравнение

| Паттерн | Что делает | Когда использовать |
|:---|:---|:---|
| **Future** | Результат в будущем | Параллельные запросы |
| **Generator** | Значения по одному | Ленивые вычисления |

### 💡 Практика: как использовать future и generator

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Future** — для параллельных запросов, результаты нужны позже.
2. **Generator** — для ленивых вычислений.
3. **`context` в generator** — для отмены.

**❌ НЕ ДЕЛАЙ:**

4. **Не используй future для простых задач.** Горутина + канал проще.
5. **Не забывай `context` в generator.** Утечка.

### Ключевые выводы подглавы 9.7

- **Future** — результат в будущем. Горутина + канал.
- **Generator** — значения по одному. Канал + горутина.
- **`context`** — для отмены.

---

## 9.8 Комбинирование паттернов

Разберём, **как комбинировать** паттерны.

### Пример: HTTP-клиент с rate limiter + circuit breaker + semaphore

```go
type HTTPClient struct {
	client  *http.Client
	limiter *rate.Limiter
	breaker *CircuitBreaker
	sem     chan struct{}
}

func NewHTTPClient(rps int, burst int, maxConns int) *HTTPClient {
	return &HTTPClient{
		client:  &http.Client{Timeout: 10 * time.Second},
		limiter: rate.NewLimiter(rate.Limit(rps), burst),
		breaker: NewCircuitBreaker(5, 30*time.Second),
		sem:     make(chan struct{}, maxConns),
	}
}

func (c *HTTPClient) Get(ctx context.Context, url string) (*http.Response, error) {
	// 1. Rate limiter
	if err := c.limiter.Wait(ctx); err != nil {
		return nil, err
	}

	// 2. Circuit breaker
	var resp *http.Response
	err := c.breaker.Call(func() error {
		// 3. Semaphore
		select {
		case c.sem <- struct{}{}:
			defer func() { <-c.sem }()
		case <-ctx.Done():
			return ctx.Err()
		}

		// 4. HTTP-запрос
		req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
		if err != nil {
			return err
		}
		resp, err = c.client.Do(req)
		return err
	})

	return resp, err
}
```

**Порядок важен:**

- **Rate limiter** — первым, чтобы не тратить ресурсы.
- **Circuit breaker** — вторым, чтобы быстро отклонить.
- **Semaphore** — третьим, чтобы ограничить ресурсы.
- **HTTP** — сам запрос.

### Таблица комбинаций

| Паттерн 1 | Паттерн 2 | Когда |
|:---|:---|:---|
| Worker pool | Rate limiter | Много задач + внешний API |
| Pipeline | Rate limiter | Конвейер + внешний API |
| Semaphore | Circuit breaker | Ограниченные ресурсы + отказы |
| Rate limiter | Circuit breaker | HTTP-клиент |
| Всё вместе | — | Production HTTP-клиент |

### 💡 Практика: как комбинировать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Rate limiter первым** — чтобы не тратить ресурсы.
2. **Circuit breaker вторым** — чтобы быстро отклонить.
3. **Semaphore третьим** — чтобы ограничить ресурсы.
4. **Метрики на каждом слое.**

**❌ НЕ ДЕЛАЙ:**

5. **Не ставь слишком много слоёв.** Overhead.
6. **Не игнорируй порядок.** Rate limiter должен быть первым.

### Ключевые выводы подглавы 9.8

- **Комбинация паттернов** — для production.
- **Порядок:** rate limiter → circuit breaker → semaphore → запрос.
- **Метрики на каждом слое.**

---

## 9.9 Практика Go: rate limiter с метриками

Напишем **rate limiter с метриками** — и сравним наш token bucket с `x/time/rate`.

### Полный код

```go
package main

import (
	"context"
	"fmt"
	"sync/atomic"
	"time"

	"golang.org/x/time/rate"
)

type Metrics struct {
	Allowed   atomic.Int64
	Waited    atomic.Int64
	Denied    atomic.Int64
	TotalWait atomic.Int64
}

func main() {
	fmt.Println("=== x/time/rate ===")
	benchLimiter()

	fmt.Println("\n=== token bucket (наш) ===")
	benchTokenBucket()
}

func benchLimiter() {
	limiter := rate.NewLimiter(rate.Limit(100), 10)
	var metrics Metrics
	ctx := context.Background()

	start := time.Now()
	for i := 0; i < 1000; i++ {
		waitStart := time.Now()
		if err := limiter.Wait(ctx); err != nil {
			metrics.Denied.Add(1)
			continue
		}
		waitDuration := time.Since(waitStart)
		metrics.TotalWait.Add(int64(waitDuration))

		if waitDuration > time.Millisecond {
			metrics.Waited.Add(1)
		}
		metrics.Allowed.Add(1)
	}
	elapsed := time.Since(start)

	printMetrics(&metrics, elapsed)
}

func benchTokenBucket() {
	tb := NewTokenBucket(100, 10)
	defer tb.Close()

	var metrics Metrics
	ctx := context.Background()

	start := time.Now()
	for i := 0; i < 1000; i++ {
		waitStart := time.Now()
		if err := tb.Wait(ctx); err != nil {
			metrics.Denied.Add(1)
			continue
		}
		waitDuration := time.Since(waitStart)
		metrics.TotalWait.Add(int64(waitDuration))

		if waitDuration > time.Millisecond {
			metrics.Waited.Add(1)
		}
		metrics.Allowed.Add(1)
	}
	elapsed := time.Since(start)

	printMetrics(&metrics, elapsed)
}

func printMetrics(metrics *Metrics, elapsed time.Duration) {
	fmt.Printf("Total:    %d\n", metrics.Allowed.Load()+metrics.Denied.Load())
	fmt.Printf("Allowed:  %d\n", metrics.Allowed.Load())
	fmt.Printf("Denied:   %d\n", metrics.Denied.Load())
	fmt.Printf("Waited:   %d\n", metrics.Waited.Load())
	fmt.Printf("Elapsed:  %v\n", elapsed)
	fmt.Printf("Rate:     %.2f req/sec\n", float64(metrics.Allowed.Load())/elapsed.Seconds())
	if metrics.Allowed.Load() > 0 {
		avgWait := time.Duration(metrics.TotalWait.Load() / metrics.Allowed.Load())
		fmt.Printf("Avg wait: %v\n", avgWait)
	}
}
```

**Пример вывода:**

```
=== x/time/rate ===
Total:    1000
Allowed:  1000
Denied:   0
Waited:   995
Elapsed:  9.95s
Rate:     100.50 req/sec
Avg wait: 9.9ms

=== token bucket (наш) ===
Total:    1000
Allowed:  1000
Denied:   0
Waited:   995
Elapsed:  9.95s
Rate:     100.50 req/sec
Avg wait: 9.9ms
```

**Что видно:** обе реализации дают **одинаковый результат**. Наш token bucket **не хуже** `x/time/rate` для базового случая.

### Аннотация сложности

| Метрика | Как измеряется |
|:---|:---|
| Allowed | `atomic.Int64` |
| Waited | `atomic.Int64` (wait > 1ms) |
| Denied | `atomic.Int64` |
| TotalWait | `atomic.Int64` (наносекунды) |

### 💡 Практика: как измерять rate limiter

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики:** allowed, waited, denied.
2. **Средняя скорость** — allowed / elapsed.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Среднее время ожидания.**
4. **Экспорт в Prometheus.**

**❌ НЕ ДЕЛАЙ:**

5. **Не используй `Mutex` для метрик.**

### Ключевые выводы подглавы 9.9

- **Метрики:** allowed, waited, denied.
- **Средняя скорость** — allowed / elapsed.
- **Наш token bucket** не хуже `x/time/rate` для базового случая.
- **`atomic.Int64`** для счётчиков.

---

## 9.10 Выводы и типичные ошибки

**Что мы узнали?**

Semaphore — ограничивает параллелизм. Взвешенный семафор — для разнородных задач. Rate limiter — ограничивает скорость. **Token bucket с нуля:** буфер канала = `burst`, наполнитель через `time.NewTicker`, `Wait(ctx)` и `Allow()`, `Close()`. Token bucket vs leaky bucket — два алгоритма. Circuit breaker — защита от каскадных отказов. Future/promise и generator — продвинутые паттерны. Комбинирование паттернов — для production. Метрики на каждом слое.

**Типичные ошибки:**

- ❌ **Забыть освободить слот семафора.** Утечка.
- ❌ **Семафор без `context`.** Не отменяется.
- ❌ **Взвешенный семафор для одинаковых весов.** Обычный проще.
- ❌ **Rate limiter без burst.** Все запросы ждут.
- ❌ **Слишком маленький burst.** Пики не пройдут.
- ❌ **Игнорировать ошибки `Wait`.** `ctx.Err()`.
- ❌ **Забыть `Close()` у token bucket.** Утечка горутины-наполнителя.
- ❌ **Использовать `time.Tick` в production.** Утечка.
- ❌ **Путать token bucket и leaky bucket.** Разные цели.
- ❌ **Circuit breaker с threshold 1.** Ложные срабатывания.
- ❌ **Circuit breaker с timeout 1 секунду.** Сервис не восстановится.
- ❌ **Future без buffered канала.** Горутина блокируется.
- ❌ **Generator без `context`.** Утечка.
- ❌ **Слишком много слоёв паттернов.** Overhead.
- ❌ **Неправильный порядок слоёв.** Rate limiter первым.

---

## 9.11 Для быстрого повторения

- **Semaphore** — `make(chan struct{}, N)`. Захват: `sem <- struct{}{}`. Освобождение: `<-sem`.
- **Взвешенный семафор** — `golang.org/x/sync/semaphore`. `Acquire(ctx, weight)`.
- **Rate limiter** — `golang.org/x/time/rate`. `NewLimiter(rps, burst)`.
- **Token bucket с нуля:** буфер канала = `burst`, наполнитель через `time.NewTicker`.
- **Наш token bucket:** `Wait(ctx)`, `Allow()`, `Close()`.
- **`Wait(ctx)`** — ожидание. **`Allow()`** — неблокирующая проверка.
- **Burst** — запас на пики. 10-100 для большинства.
- **Token bucket** — разрешает пики. **Leaky bucket** — сглаживает.
- **Go-стандарт** — token bucket.
- **Circuit breaker** — три состояния: Closed, Open, Half-Open.
- **Threshold 5-10, timeout 10-60 сек.**
- **`gobreaker`** — готовая библиотека.
- **Future** — результат в будущем. Горутина + канал.
- **Generator** — значения по одному. Ленивые вычисления.
- **Комбинация:** rate limiter → circuit breaker → semaphore → запрос.
- **Метрики** на каждом слое.

---

## 9.12 Вопросы для самопроверки

1. Что такое semaphore? Чем отличается от worker pool?
2. Как реализовать semaphore на каналах?
3. Что такое взвешенный семафор? Когда нужен?
4. Что такое rate limiter? Чем отличается от semaphore?
5. Как написать token bucket с нуля?
6. Почему в token bucket используется буферизованный канал?
7. Что делает `select` с `default` при добавлении токена?
8. Как остановить горутину-наполнитель?
9. Что такое burst? Зачем нужен?
10. Как использовать `golang.org/x/time/rate`?
11. Чем token bucket отличается от leaky bucket?
12. Что такое circuit breaker? Три состояния?
13. Когда circuit breaker переходит в Open? В Half-Open? В Closed?
14. Что такое future/promise?
15. Что такое generator?
16. Как комбинировать rate limiter + circuit breaker + semaphore?
17. Почему rate limiter должен быть первым?
18. Как измерять метрики rate limiter?

---

## 9.13 Ответы

### Ответ 1

**Semaphore** — ограничивает число **одновременных** операций.

**Отличие от worker pool:**
- Worker pool: N воркеров, переиспользуются.
- Semaphore: N слотов, каждая задача создаёт горутину.

### Ответ 2

```go
sem := make(chan struct{}, N)
sem <- struct{}{}  // захват
defer func() { <-sem }()  // освобождение
```

### Ответ 3

**Взвешенный семафор** — каждая операция захватывает N единиц.

**Когда нужен:** разнородные задачи (лёгкие + тяжёлые).

**Реализация:** `golang.org/x/sync/semaphore`.

### Ответ 4

**Rate limiter** — ограничивает **скорость** (событий/сек).

**Отличие от semaphore:** semaphore — параллелизм, rate limiter — скорость.

### Ответ 5

**Token bucket с нуля:**

```go
func tokenBucket(rps int, burst int) (chan struct{}, func()) {
	tokens := make(chan struct{}, burst)
	stop := make(chan struct{})

	for i := 0; i < burst; i++ {
		tokens <- struct{}{}
	}

	go func() {
		interval := time.Second / time.Duration(rps)
		ticker := time.NewTicker(interval)
		defer ticker.Stop()
		for {
			select {
			case <-stop:
				return
			case <-ticker.C:
				select {
				case tokens <- struct{}{}:
				default:
				}
			}
		}
	}()

	return tokens, func() { close(stop) }
}
```

### Ответ 6

**Буферизованный канал** используется как **ведро с токенами**:

- **Размер буфера** = `burst` (максимум токенов).
- **Элементы** = токены.
- **Забор токена** = чтение из канала.
- **Добавление токена** = запись в канал.

### Ответ 7

**`select` с `default`** при добавлении токена: если буфер полон (ведро полно) — **пропускаем**. Не блокируемся. Ведро не переполняется.

### Ответ 8

**Остановить горутину-наполнитель:** через `stop` канал.

```go
stop := make(chan struct{})
// ...
case <-stop:
	return
// ...
func() { close(stop) }
```

### Ответ 9

**Burst** — максимальный «запас» токенов.

**Зачем:** для пиков. Первые `burst` запросов — мгновенно.

### Ответ 10

```go
limiter := rate.NewLimiter(rate.Limit(100), 10)
if err := limiter.Wait(ctx); err != nil {
	return err
}
```

### Ответ 11

**Token bucket** — разрешает пики до `burst`. **Leaky bucket** — сглаживает поток.

**Go-стандарт** — token bucket.

### Ответ 12

**Circuit breaker** — защищает от каскадных отказов.

**Три состояния:**
- **Closed:** пропускает запросы.
- **Open:** отклоняет.
- **Half-Open:** пробные запросы.

### Ответ 13

**Open:** ошибок > threshold.
**Half-Open:** через timeout после Open.
**Closed:** успех в Half-Open.

### Ответ 14

**Future** — объект, представляющий результат в будущем.

```go
f := NewFuture(func() (string, error) {
	return "result", nil
})
result, err := f.Get(ctx)
```

### Ответ 15

**Generator** — функция, отдающая значения по одному.

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

### Ответ 16

**Комбинация:**
1. **Rate limiter** — первым.
2. **Circuit breaker** — вторым.
3. **Semaphore** — третьим.
4. **Запрос** — последним.

### Ответ 17

**Rate limiter первым**, чтобы не тратить ресурсы на запросы, которые всё равно будут отклонены.

### Ответ 18

**Метрики:** allowed, waited, denied, avg wait, rate.

---

## 9.14 Куда идти дальше?

Мы разобрали продвинутые паттерны: semaphore, rate limiter, **token bucket с нуля**, circuit breaker, future, generator. Теперь мы умеем ограничивать ресурсы и защищаться от отказов.

Но остаётся **фундаментальный вопрос**: как **обрабатывать ошибки** в конкурентном коде? Что если одна из 1000 горутин упала? Как собрать все ошибки?

- **Как обрабатывать ошибки в конкурентном коде?** `errgroup`, `multierror`, отмена при первой ошибке. → **Глава 10: Обработка ошибок в конкурентном коде.**
- **Как корректно завершить сервис?** Сигналы ОС, `context`, ожидание завершения. → **Глава 11: Graceful shutdown.**
- **Как тестировать конкурентный код?** Race detector, stress-тесты, `pprof`. → **Глава 12: Тестирование и профилирование.**

---

## 9.15 Чек-лист

| Паттерн | Что делает | Когда использовать |
|:---|:---|:---|
| **Semaphore** | Ограничивает параллелизм | Ограничение ресурса |
| **`make(chan struct{}, N)`** | Семафор на каналах | Простой случай |
| **`x/sync/semaphore`** | Взвешенный семафор | Разнородные задачи |
| **Rate limiter** | Ограничивает скорость | Внешний API |
| **`x/time/rate`** | Token bucket | Стандарт |
| **Token bucket с нуля** | Буфер канала = burst | Обучение |
| **`NewLimiter(rps, burst)`** | RPS + burst | 10-100 burst |
| **`Wait(ctx)`** | Ожидание | Блокирующая проверка |
| **`Allow()`** | Неблокирующая | Drop при превышении |
| **`Close()`** | Остановка наполнителя | Обязательно |
| **Token bucket** | Разрешает пики | HTTP API |
| **Leaky bucket** | Сглаживает | Сетевой трафик |
| **Circuit breaker** | Защита от отказов | Внешние сервисы |
| **Три состояния** | Closed, Open, Half-Open | Threshold 5-10 |
| **`gobreaker`** | Готовая библиотека | Production |
| **Future** | Результат в будущем | Параллельные запросы |
| **Generator** | Значения по одному | Ленивые вычисления |
| **Комбинация** | Rate → breaker → sem → запрос | Production HTTP-клиент |
| **Метрики** | Allowed, waited, denied | На каждом слое |

⚡ **Ключевая идея:** Semaphore ограничивает параллелизм; на каналах — `make(chan struct{}, N)`. Взвешенный семафор — `x/sync/semaphore` для разнородных задач. Rate limiter ограничивает скорость; `x/time/rate` — token bucket. **Token bucket с нуля:** буфер канала = `burst`, наполнитель через `time.NewTicker`, `Wait(ctx)` и `Allow()`, `Close()` — обязательно. Token bucket разрешает пики, leaky bucket сглаживает. Circuit breaker защищает от каскадных отказов; три состояния: Closed, Open, Half-Open. Future/promise и generator — продвинутые паттерны. Комбинация: rate limiter → circuit breaker → semaphore → запрос. Метрики на каждом слое.