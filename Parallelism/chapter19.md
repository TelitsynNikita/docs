# 🛡️ Глава 19: Продвинутые паттерны — bulkhead, retry, leader election

**Что вы узнаете:**
- Что такое **bulkhead** и как он изолирует ресурсы.
- Чем bulkhead отличается от **circuit breaker** и **rate limiter**.
- Как построить **retry с exponential backoff и jitter**.
- Почему **jitter** критичен для распределённых систем.
- Что такое **leader election** и зачем он нужен.
- Как реализовать leader election через **etcd** и **Consul**.
- Что такое **split-brain** и как его избежать.
- Что такое **fencing tokens** и зачем они нужны.
- Что такое **distributed locks** и как их строить.
- Что такое **Redlock** и почему он спорный.
- Что такое **idempotency** и зачем она нужна.

**После прочтения вы сможете:**
- Построить bulkhead для изоляции ресурсов.
- Написать retry с exponential backoff и jitter.
- Реализовать leader election через etcd.
- Понимать, как избежать split-brain.
- Использовать fencing tokens для безопасных distributed locks.
- Реализовать distributed lock через Redis или etcd.
- Понимать, когда Redlock оправдан, а когда — нет.
- Обеспечить idempotency для retry.

---

## Содержание

- [19.0 Пролог: retry, который положил сервис](#190-пролог-retry-который-положил-сервис)
- [19.1 Bulkhead: изоляция ресурсов](#191-bulkhead-изоляция-ресурсов)
- [19.2 Retry с exponential backoff](#192-retry-с-exponential-backoff)
- [19.3 Jitter: почему случайность критична](#193-jitter-почему-случайность-критична)
- [19.4 Leader election: зачем и как](#194-leader-election-зачем-и-как)
- [19.5 Leader election через etcd](#195-leader-election-через-etcd)
- [19.6 Split-brain и fencing tokens](#196-split-brain-и-fencing-tokens)
- [19.7 Distributed locks: Redis и etcd](#197-distributed-locks-redis-и-etcd)
- [19.8 Redlock: спорный алгоритм](#198-redlock-спорный-алгоритм)
- [19.9 Idempotency: ключ к безопасному retry](#199-idempotency-ключ-к-безопасному-retry)
- [19.10 Практика Go: retry с backoff и jitter](#1910-практика-go-retry-с-backoff-и-jitter)
- [19.11 Выводы и типичные ошибки](#1911-выводы-и-типичные-ошибки)
- [19.12 Для быстрого повторения](#1912-для-быстрого-повторения)
- [19.13 Вопросы для самопроверки](#1913-вопросы-для-самопроверки)
- [19.14 Ответы](#1914-ответы)
- [19.15 Куда идти дальше?](#1915-куда-идти-дальше)
- [19.16 Чек-лист](#1916-чек-лист)

---

## 19.0 Пролог: retry, который положил сервис

Ты пишешь HTTP-клиент к внешнему API. API иногда возвращает 500. Ты добавляешь retry:

```go
func fetch(url string) (*Response, error) {
    for i := 0; i < 3; i++ {
        resp, err := http.Get(url)
        if err == nil && resp.StatusCode < 500 {
            return resp, nil
        }
        time.Sleep(100 * time.Millisecond)  // фиксированная задержка
    }
    return nil, errors.New("max retries")
}
```

Всё работает. Пока внешний API не начинает **тормозить**. Через минуту твой сервис **умирает**:

- **CPU 100%.**
- **Тысячи горутин** ждут retry.
- **Внешний API** получает **в 3 раза больше** запросов.
- **Каскадный отказ.**

❓ **Что произошло?** Ты используешь **фиксированную задержку** retry. Все горутины retry **одновременно**. Внешний API получает **thundering herd** (Глава 13).

💡 **Решение:** **exponential backoff с jitter**.

```go
func fetch(ctx context.Context, url string) (*Response, error) {
    backoff := 100 * time.Millisecond
    for i := 0; i < 3; i++ {
        resp, err := http.Get(url)
        if err == nil && resp.StatusCode < 500 {
            return resp, nil
        }

        // Exponential backoff с jitter
        jitter := time.Duration(rand.Int63n(int64(backoff)))
        select {
        case <-time.After(backoff + jitter):
        case <-ctx.Done():
            return nil, ctx.Err()
        }
        backoff *= 2  // 100ms → 200ms → 400ms
    }
    return nil, errors.New("max retries")
}
```

**Что изменилось:**

- **Backoff растёт** — 100 → 200 → 400 мс.
- **Jitter** — случайная добавка.
- **Горутины retry не одновременно.**

**Это retry с backoff и jitter.** В этой главе — как строить отказоустойчивые системы: bulkhead, retry, leader election, distributed locks, idempotency.

> **Важный мост:** retry — часть **отказоустойчивости**. Bulkhead изолирует ресурсы. Leader election координирует. Distributed locks защищают. Idempotency — ключ к безопасному retry. Глава 22-28 (Распределённые системы) — продолжение.

---

## 19.1 Bulkhead: изоляция ресурсов

**Bulkhead** — паттерн изоляции ресурсов, чтобы отказ одной части не уронил всю систему.

### Идея

**Bulkhead** (переборка) — как **отсеки** на корабле. Если один отсек затоплен — остальные **не тонут**.

**Пример из жизни:**

- **Без bulkhead:** одна медленная зависимость блокирует **весь** пул потоков.
- **С bulkhead:** каждая зависимость имеет **свой** пул.

### Проблема: общий пул

```go
var client = &http.Client{
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 100,
    },
}

func callServiceA() { /* использует client */ }
func callServiceB() { /* использует client */ }
func callServiceC() { /* использует client */ }
```

**Что происходит:** если `ServiceA` **тормозит**, все 100 соединений заняты `ServiceA`. `ServiceB` и `ServiceC` **ждут**.

### Решение: bulkhead

```go
type Bulkhead struct {
    client *http.Client
    sem    chan struct{}
}

func NewBulkhead(maxConcurrent int) *Bulkhead {
    return &Bulkhead{
        client: &http.Client{Timeout: 10 * time.Second},
        sem:    make(chan struct{}, maxConcurrent),
    }
}

func (b *Bulkhead) Call(ctx context.Context, url string) (*http.Response, error) {
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
    case <-ctx.Done():
        return nil, ctx.Err()
    }
    return b.client.Get(url)
}
```

**Что делает:** не более `maxConcurrent` одновременных запросов. Остальные ждут или отклоняются.

### Bulkhead на сервис

```go
type ServiceClients struct {
    serviceA *Bulkhead
    serviceB *Bulkhead
    serviceC *Bulkhead
}

func NewServiceClients() *ServiceClients {
    return &ServiceClients{
        serviceA: NewBulkhead(50),
        serviceB: NewBulkhead(30),
        serviceC: NewBulkhead(20),
    }
}
```

**Что делает:** у каждой зависимости **свой** пул.

**Результат:** если `ServiceA` тормозит, `ServiceB` и `ServiceC` работают.

### Bulkhead vs circuit breaker vs rate limiter

| Паттерн | Что делает | Когда использовать |
|:---|:---|:---|
| **Bulkhead** | Изолирует ресурсы | Разные зависимости |
| **Circuit breaker** | Прекращает вызовы | Зависимость падает |
| **Rate limiter** | Ограничивает скорость | Внешний лимит |

**Все три нужны.** Они **дополняют** друг друга.

### Bulkhead + circuit breaker

```go
type Client struct {
    bulkhead *Bulkhead
    breaker  *gobreaker.CircuitBreaker
}

func (c *Client) Call(ctx context.Context, url string) (*http.Response, error) {
    var resp *http.Response
    _, err := c.breaker.Execute(func() (interface{}, error) {
        var err error
        resp, err = c.bulkhead.Call(ctx, url)
        return nil, err
    })
    return resp, err
}
```

**Что делает:**

- **Bulkhead** — изоляция.
- **Circuit breaker** — быстрый отказ.

### Bulkhead + rate limiter

```go
type Client struct {
    bulkhead *Bulkhead
    limiter  *rate.Limiter
}

func (c *Client) Call(ctx context.Context, url string) (*http.Response, error) {
    if err := c.limiter.Wait(ctx); err != nil {
        return nil, err
    }
    return c.bulkhead.Call(ctx, url)
}
```

### Аннотация сложности

| Операция | Time |
|:---|:---|
| Bulkhead (семафор) | ~30-70 нс |
| Circuit breaker | ~100-200 нс |
| Rate limiter | ~50-100 нс |

### 💡 Практика: как использовать bulkhead

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Отдельный bulkhead** для каждой зависимости.
2. **Разные размеры** пулов.
3. **`context`** для отмены.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Bulkhead + circuit breaker.**
5. **Bulkhead + rate limiter.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй общий пул** для всех зависимостей.
7. **Не делай bulkhead слишком маленьким.**

### Ключевые выводы подглавы 19.1

- **Bulkhead** — изоляция ресурсов.
- **Свой пул** для каждой зависимости.
- **Bulkhead vs circuit breaker vs rate limiter** — разные паттерны.
- **Все три нужны** для отказоустойчивости.

---

## 19.2 Retry с exponential backoff

Разберём **retry с exponential backoff**.

### Проблема: фиксированная задержка

```go
for i := 0; i < 3; i++ {
    resp, err := http.Get(url)
    if err == nil {
        return resp, nil
    }
    time.Sleep(100 * time.Millisecond)  // ← фиксированная
}
```

**Проблемы:**

- **Thundering herd:** все retry одновременно.
- **Не адаптируется:** если сервис тормозит, retry не помогает.
- **Каскадный отказ:** retry нагружает сервис.

### Решение: exponential backoff

**Идея:** задержка **растёт** с каждой попыткой.

```go
backoff := 100 * time.Millisecond
for i := 0; i < 5; i++ {
    resp, err := http.Get(url)
    if err == nil {
        return resp, nil
    }
    time.Sleep(backoff)
    backoff *= 2  // 100ms → 200ms → 400ms → 800ms → 1600ms
}
```

**Что делает:**

- **Быстро** при первой попытке.
- **Медленнее** при повторных.
- **Даёт сервису время** восстановиться.

### Формула

```
backoff = base * 2^attempt
```

**Пример:** base = 100 мс.

| Attempt | Backoff |
|:---|:---|
| 1 | 100 мс |
| 2 | 200 мс |
| 3 | 400 мс |
| 4 | 800 мс |
| 5 | 1600 мс |

### Cap

**Проблема:** backoff растёт **бесконечно**.

**Решение:** **cap** (максимум).

```go
backoff := 100 * time.Millisecond
maxBackoff := 30 * time.Second

for i := 0; i < 10; i++ {
    // ...
    time.Sleep(backoff)
    backoff *= 2
    if backoff > maxBackoff {
        backoff = maxBackoff
    }
}
```

### Context

**Важно:** учитывать `context`.

```go
select {
case <-time.After(backoff):
case <-ctx.Done():
    return nil, ctx.Err()
}
```

### Условия retry

**Не все ошибки retry:**

**Retry:**

- Сетевые ошибки.
- 5xx (500, 502, 503, 504).
- Таймауты.
- Connection refused.

**НЕ retry:**

- 4xx (400, 401, 403, 404).
- Ошибки валидации.
- Permanent ошибки.

```go
func isRetryable(err error) bool {
    if err == nil {
        return false
    }
    var netErr net.Error
    if errors.As(err, &netErr) && netErr.Timeout() {
        return true
    }
    return false
}

func isRetryableStatus(code int) bool {
    return code >= 500 && code < 600
}
```

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `time.After` | ~100-200 нс + таймер |
| `backoff *= 2` | ~1 нс |
| Retry (5 попыток) | ~3 сек |

### 💡 Практика: как делать retry

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Exponential backoff.**
2. **Cap** на максимум.
3. **`context`** для отмены.
4. **Retry только retryable ошибки.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Jitter** (см. 19.3).
6. **Метрики** — сколько retry.

**❌ НЕ ДЕЛАЙ:**

7. **Не retry 4xx.**
8. **Не retry без backoff.**
9. **Не retry бесконечно.**

### Ключевые выводы подглавы 19.2

- **Exponential backoff** — задержка растёт.
- **Cap** — максимум.
- **`context`** — отмена.
- **Retry только retryable ошибки.**
- **5 попыток** обычно достаточно.

---

## 19.3 Jitter: почему случайность критична

**Jitter** — случайная добавка к backoff.

### Проблема: синхронные retry

**Без jitter:**

```
t=0:    100 горутин делают retry
t=100ms: все 100 retry одновременно
t=300ms: все 100 retry одновременно
t=700ms: все 100 retry одновременно
```

**Что происходит:** **thundering herd** (Глава 13).

### Решение: jitter

**С jitter:**

```
t=0:    100 горутин делают retry
t=100-200ms: retry размазаны
t=300-500ms: retry размазаны
t=700-1100ms: retry размазаны
```

**Что делает:** retry **размазаны** по времени.

### Full jitter

**Формула:**

```
sleep = random(0, backoff)
```

**Пример:**

```go
sleep := time.Duration(rand.Int63n(int64(backoff)))
time.Sleep(sleep)
```

**Плюсы:** максимальное размазывание.

**Минусы:** может быть 0 — retry сразу.

### Equal jitter

**Формула:**

```
sleep = backoff/2 + random(0, backoff/2)
```

**Пример:**

```go
half := backoff / 2
sleep := half + time.Duration(rand.Int63n(int64(half)))
time.Sleep(sleep)
```

**Плюсы:** минимум `backoff/2`.

### Decorrelated jitter

**Формула:**

```
sleep = random(base, prev*3)
```

**Пример:**

```go
sleep := time.Duration(rand.Int63n(int64(prev*3-base))) + base
prev = sleep
```

**Плюсы:** лучше для распределённых систем.

### Сравнение

| Тип | Формула | Плюсы | Минусы |
|:---|:---|:---|:---|
| **Full** | `random(0, backoff)` | Максимум размазывания | Может быть 0 |
| **Equal** | `backoff/2 + random(0, backoff/2)` | Минимум `backoff/2` | Меньше размазывание |
| **Decorrelated** | `random(base, prev*3)` | Лучше для distributed | Сложнее |

### Пример

```go
func retryWithJitter(ctx context.Context, fn func() error) error {
    const maxAttempts = 5
    base := 100 * time.Millisecond
    maxBackoff := 10 * time.Second

    var prev time.Duration
    for attempt := 0; attempt < maxAttempts; attempt++ {
        err := fn()
        if err == nil {
            return nil
        }

        // Decorrelated jitter
        var sleep time.Duration
        if attempt == 0 {
            sleep = base
        } else {
            sleep = time.Duration(rand.Int63n(int64(prev*3-base))) + base
        }
        if sleep > maxBackoff {
            sleep = maxBackoff
        }
        prev = sleep

        select {
        case <-time.After(sleep):
        case <-ctx.Done():
            return ctx.Err()
        }
    }
    return errors.New("max retries")
}
```

### Аннотация сложности

| Тип jitter | Time |
|:---|:---|
| Без jitter | ~100 нс |
| Full | ~150-200 нс |
| Equal | ~150-200 нс |
| Decorrelated | ~200-300 нс |

### 💡 Практика: как использовать jitter

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Jitter** для распределённых систем.
2. **Full jitter** — простое.
3. **Decorrelated** — для production.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Cap** на максимум.
5. **`context`** для отмены.

**❌ НЕ ДЕЛАЙ:**

6. **Не retry без jitter** при 100+ горутинах.
7. **Не используй фиксированный jitter.**

### Ключевые выводы подглавы 19.3

- **Jitter** — случайная добавка.
- **Full, Equal, Decorrelated.**
- **Decorrelated** — лучший для distributed.
- **Без jitter** — thundering herd.

---

## 19.4 Leader election: зачем и как

**Leader election** — выбор **одного** узла как лидера.

### Зачем

**Сценарии:**

- **Singleton задачи:** только один узел делает работу.
- **Координация:** лидер решает, что делать.
- **Распределённые блокировки:** лидер держит lock.

**Примеры:**

- **Kafka:** partition leader.
- **Kubernetes:** controller leader.
- **etcd:** Raft leader.
- **Scheduler:** один планировщик.

### Свойства

**Leader election должен быть:**

1. **Safety:** не более **одного** лидера.
2. **Liveness:** в итоге **один** лидер есть.
3. **Fault-tolerant:** при отказе лидера — новый.

### CAP и leader election

**Leader election требует:**

- **Consensus** (согласие).
- **Quorum** (большинство).
- **Strong consistency.**

**Поэтому:** leader election через **etcd**, **Consul**, **ZooKeeper** — они дают **consensus**.

### Простая реализация (некорректная)

```go
var leader atomic.Bool

func tryBecomeLeader() bool {
    return leader.CompareAndSwap(false, true)
}
```

**Проблема:** **split-brain**. Два узла могут стать лидерами.

### Правильная реализация

**Через etcd:**

```go
import (
    clientv3 "go.etcd.io/etcd/client/v3"
    "go.etcd.io/etcd/client/v3/concurrency"
)

func main() {
    client, _ := clientv3.New(clientv3.Config{
        Endpoints: []string{"localhost:2379"},
    })
    defer client.Close()

    session, _ := concurrency.NewSession(client)
    defer session.Close()

    election := concurrency.NewElection(session, "/my-election/")

    ctx := context.Background()
    if err := election.Campaign(ctx, "my-node"); err != nil {
        log.Fatal(err)
    }

    log.Println("I am the leader!")

    // Работаем как лидер
    <-session.Done()
    log.Println("Lost leadership")
}
```

**Что делает:**

1. Создаёт session.
2. Создаёт election.
3. Campaign — пытается стать лидером.
4. Если лидер — работает.
5. `session.Done()` — при потере.

### Как это работает

**etcd election:**

1. **Узел создаёт ключ** `/my-election/{lease-id}` с TTL.
2. **Ключ с минимальным ID** — лидер.
3. **Остальные** — followers.
4. **При отказе лидера** — lease истекает, следующий становится лидером.
5. **Heartbeat** — лидер продлевает lease.

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `Campaign` | ~10-100 мс |
| Heartbeat | ~1-10 мс |
| Смена лидера | ~1-10 сек |

### 💡 Практика: как использовать leader election

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **etcd/Consul/ZooKeeper** — для consensus.
2. **Session** — для lease.
3. **Heartbeat** — для liveness.

**👍 СТОИТ СДЕЛАТЬ:**

4. **OnStartedLeading** / **OnStoppedLeading** — callbacks.
5. **Fencing tokens** — для безопасности (см. 19.6).

**❌ НЕ ДЕЛАЙ:**

6. **Не делай leader election без consensus.**
7. **Не игнорируй split-brain.**

### Ключевые выводы подглавы 19.4

- **Leader election** — выбор одного лидера.
- **Safety, liveness, fault-tolerant.**
- **Требует consensus.**
- **etcd/Consul/ZooKeeper.**
- **Session + lease.**

---

## 19.5 Leader election через etcd

Разберём **leader election через etcd** подробно.

### Session

**Session** — lease с TTL.

```go
session, err := concurrency.NewSession(client,
    concurrency.WithTTL(10),  // 10 секунд
)
if err != nil {
    log.Fatal(err)
}
defer session.Close()
```

**Что делает:**

- Создаёт lease с TTL 10 секунд.
- Автоматически продлевает (keep-alive).
- При потере — `session.Done()`.

### Election

**Election** — выбор лидера.

```go
election := concurrency.NewElection(session, "/my-election/")
```

**Что делает:** создаёт election с префиксом.

### Campaign

**Campaign** — попытка стать лидером.

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

if err := election.Campaign(ctx, "my-node"); err != nil {
    log.Fatal(err)
}

log.Println("I am the leader!")
```

**Что делает:**

1. Создаёт ключ `/my-election/{lease-id}`.
2. Ждёт, пока станет лидером.
3. **Блокируется**, пока не станет.
4. Возвращается, когда стал.

### Работа лидера

```go
for {
    select {
    case <-session.Done():
        log.Println("Lost leadership")
        return
    case <-time.After(1 * time.Second):
        log.Println("Still leader, doing work")
    }
}
```

**Что делает:** работает, пока session активна.

### Observer

**Observer** — следит за лидером.

```go
ch := election.Observe(ctx)
for resp := range ch {
    for _, kv := range resp.Kvs {
        log.Printf("Leader is %s", kv.Value)
    }
}
```

**Что делает:** получает уведомления о смене лидера.

### Resign

**Resign** — добровольная сдача лидерства.

```go
if err := election.Resign(ctx); err != nil {
    log.Fatal(err)
}
```

### Полный пример

```go
package main

import (
    "context"
    "log"
    "time"

    clientv3 "go.etcd.io/etcd/client/v3"
    "go.etcd.io/etcd/client/v3/concurrency"
)

func main() {
    client, err := clientv3.New(clientv3.Config{
        Endpoints:   []string{"localhost:2379"},
        DialTimeout: 5 * time.Second,
    })
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()

    session, err := concurrency.NewSession(client, concurrency.WithTTL(10))
    if err != nil {
        log.Fatal(err)
    }
    defer session.Close()

    election := concurrency.NewElection(session, "/my-election/")

    ctx := context.Background()

    log.Println("Campaigning...")
    if err := election.Campaign(ctx, "my-node"); err != nil {
        log.Fatal(err)
    }

    log.Println("I am the leader!")

    // Работаем
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-session.Done():
            log.Println("Lost leadership")
            return
        case <-ticker.C:
            log.Println("Working as leader")
        }
    }
}
```

### Fencing tokens

**Проблема:** старый лидер может **не знать**, что он больше не лидер.

**Решение:** **fencing tokens**.

**etcd election** возвращает **revision** при campaign:

```go
resp, err := election.Leader(ctx)
// resp.Header.Revision — fencing token
```

**Что делать:** использовать revision как **monotonic token**. Все операции с меньшим token **отклоняются**.

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `NewSession` | ~10-100 мс |
| `Campaign` | ~10-100 мс |
| Heartbeat | ~1-10 мс |
| `Resign` | ~10-100 мс |

### 💡 Практика: как использовать etcd election

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **TTL** разумный (10-30 сек).
2. **Heartbeat** автоматически.
3. **Fencing tokens** для безопасности.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Observer** для followers.
5. **Graceful resign** при shutdown.

**❌ НЕ ДЕЛАЙ:**

6. **Не игнорируй `session.Done()`.**
7. **Не делай работу без проверки лидерства.**

### Ключевые выводы подглавы 19.5

- **Session** — lease с TTL.
- **Election** — выбор лидера.
- **Campaign** — попытка стать.
- **Fencing tokens** — для безопасности.
- **Observer** — для followers.

---

## 19.6 Split-brain и fencing tokens

**Split-brain** — ситуация, когда **два** узла думают, что они лидеры.

### Проблема

**Сценарий:**

1. Узел A — лидер.
2. Сеть разделилась.
3. A не может связаться с etcd.
4. Но A **не знает** об этом.
5. A продолжает работу.
6. B становится лидером.
7. **Два лидера.**

### Последствия

- **Двойная работа.**
- **Конфликты данных.**
- **Неконсистентность.**

### Решение: fencing tokens

**Идея:** каждое действие лидера **токенизируется**.

**Схема:**

```
1. Лидер A получает token = 5.
2. Лидер A делает операцию с token = 5.
3. Хранилище проверяет: token ≥ last_token?
4. Если да — выполняет, last_token = 5.
5. Если нет — отклоняет.

6. Сеть разделилась.
7. B становится лидером, token = 6.
8. B делает операцию с token = 6.
9. Хранилище обновляет last_token = 6.

10. Сеть восстановилась.
11. A пытается сделать операцию с token = 5.
12. Хранилище: 5 < 6 → отклоняет.
13. A понимает, что больше не лидер.
```

### Реализация

**etcd election** возвращает revision:

```go
resp, _ := election.Leader(ctx)
fencingToken := resp.Header.Revision
```

**Хранилище:**

```go
type Storage struct {
    mu          sync.Mutex
    lastToken   int64
}

func (s *Storage) Write(ctx context.Context, token int64, data []byte) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    if token <= s.lastToken {
        return fmt.Errorf("stale token: %d <= %d", token, s.lastToken)
    }

    s.lastToken = token
    // запись данных
    return nil
}
```

### Комбинация с etcd

```go
func (l *Leader) DoWork(ctx context.Context) error {
    // Получаем token
    resp, err := l.election.Leader(ctx)
    if err != nil {
        return err
    }
    token := resp.Header.Revision

    // Делаем операцию с token
    return l.storage.Write(ctx, token, data)
}
```

### Аннотация сложности

| Операция | Time |
|:---|:---|
| Получение token | ~1-10 мс |
| Проверка token | ~10-50 нс |

### 💡 Практика: как избежать split-brain

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Fencing tokens** для всех операций.
2. **Проверка token** в хранилище.
3. **Monotonic tokens.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **etcd revision** как token.
5. **Логирование** stale token.

**❌ НЕ ДЕЛАЙ:**

6. **Не делай операции без token.**
7. **Не игнорируй stale token.**

### Ключевые выводы подглавы 19.6

- **Split-brain** — два лидера.
- **Fencing tokens** — для безопасности.
- **Monotonic tokens** — проверка.
- **etcd revision** — пример.

---

## 19.7 Distributed locks: Redis и etcd

**Distributed locks** — блокировки между узлами.

### Зачем

**Сценарии:**

- **Защита ресурса:** только один узел работает с ресурсом.
- **Leader election:** лидер держит lock.
- **Критическая секция:** операция должна быть атомарной.

### Redis lock

**Простейший:**

```go
func acquireLock(client *redis.Client, key, value string, ttl time.Duration) (bool, error) {
    return client.SetNX(context.Background(), key, value, ttl).Result()
}

func releaseLock(client *redis.Client, key, value string) error {
    script := `
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
    `
    return client.Eval(context.Background(), script, []string{key}, value).Err()
}
```

**Что делает:**

- `SetNX` — установить, если не существует.
- `TTL` — автоматическое освобождение.
- `Eval` — атомарное освобождение (только владелец).

### Lua script

**Почему Lua:** атомарность.

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

**Что делает:** проверяет, что value совпадает, потом удаляет.

**Без Lua:** race condition между `GET` и `DEL`.

### etcd lock

**Через etcd:**

```go
session, _ := concurrency.NewSession(client)
mutex := concurrency.NewMutex(session, "/my-lock/")

if err := mutex.Lock(ctx); err != nil {
    log.Fatal(err)
}
defer mutex.Unlock(ctx)

// Критическая секция
```

**Что делает:**

- Создаёт ключ `/my-lock/{lease-id}`.
- Ждёт, пока станет первым.
- **Блокируется**, пока не получит.
- `Unlock` — удаляет ключ.

### Сравнение

| Аспект | Redis | etcd |
|:---|:---|:---|
| Consensus | ❌ Нет | ✅ Raft |
| Split-brain | Возможен | Нет |
| Производительность | Высокая | Средняя |
| Простота | Простая | Средняя |
| Consistency | Eventual | Strong |

### Когда что

**Redis:**

- **Производительность** важна.
- **Split-brain** не критичен.
- **Cache locks.**

**etcd:**

- **Consistency** важна.
- **Split-brain** недопустим.
- **Leader election.**

### Аннотация сложности

| Операция | Redis | etcd |
|:---|:---|:---|
| Acquire | ~1-5 мс | ~10-100 мс |
| Release | ~1-5 мс | ~10-100 мс |

### 💡 Практика: как использовать distributed locks

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **TTL** для автоматического освобождения.
2. **Lua script** для атомарного освобождения.
3. **Fencing tokens** для безопасности.

**👍 СТОИТ СДЕЛАТЬ:**

4. **etcd** для критичных случаев.
5. **Метрики** — сколько locks.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй Redis lock** без TTL.
7. **Не игнорируй split-brain.**

### Ключевые выводы подглавы 19.7

- **Distributed locks** — между узлами.
- **Redis:** `SetNX` + Lua.
- **etcd:** Raft-based.
- **Redis** — быстрее. **etcd** — надёжнее.
- **Fencing tokens** — для безопасности.

---

## 19.8 Redlock: спорный алгоритм

**Redlock** — алгоритм distributed lock от Redis.

### Идея

**Redlock** использует **N независимых Redis** (обычно 5).

**Алгоритм:**

1. Получить **текущее время**.
2. Попытаться получить lock в **каждом** Redis с **одним** TTL.
3. Если **большинство** (N/2+1) ответили — lock получен.
4. **Проверить**, что время не истекло.
5. Если да — lock получен.
6. Если нет — освободить в **всех**.

### Пример

```go
func redlock(clients []*redis.Client, key, value string, ttl time.Duration) (bool, error) {
    quorum := len(clients)/2 + 1
    acquired := 0

    for _, client := range clients {
        ok, err := client.SetNX(context.Background(), key, value, ttl).Result()
        if err == nil && ok {
            acquired++
        }
    }

    if acquired >= quorum {
        return true, nil
    }

    // Освободить в тех, где получили
    for _, client := range clients {
        releaseLock(client, key, value)
    }
    return false, nil
}
```

### Критика

**Martin Kleppmann** (автор «Designing Data-Intensive Applications») критикует Redlock:

1. **Не даёт fencing tokens.** Без token нельзя гарантировать безопасность.
2. **Зависит от часов.** Если часы расходятся — lock может быть получен дважды.
3. **Не решает split-brain.** Если сеть разделилась — два узла могут получить lock.
4. **Не даёт consistency.** В отличие от etcd/ZooKeeper.

### Ответ antirez

**Salvatore Sanfilippo** (автор Redis) отвечает:

1. **Redlock — не для всех случаев.** Для критичных — etcd.
2. **Fencing tokens** — можно добавить.
3. **Часы** — можно синхронизировать (NTP).

### Вердикт

**Redlock оправдан:**

- **Производительность** важна.
- **Split-brain** маловероятен.
- **Не критичные данные.**

**Redlock НЕ оправдан:**

- **Consistency** критична.
- **Split-brain** недопустим.
- **Критичные данные.**

**Для критичных случаев** — **etcd** или **ZooKeeper**.

### Аннотация сложности

| Операция | Redlock | etcd |
|:---|:---|:---|
| Acquire | ~5-50 мс | ~10-100 мс |
| Safety | ⚠️ Спорная | ✅ Доказана |

### 💡 Практика: как использовать Redlock

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **N = 5** независимых Redis.
2. **TTL** разумный.
3. **Fencing tokens** если возможно.

**👍 СТОИТ СДЕЛАТЬ:**

4. **etcd** для критичных случаев.
5. **Метрики** — сколько lock получен.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй Redlock** для критичных данных.
7. **Не игнорируй критику.**

### Ключевые выводы подглавы 19.8

- **Redlock** — distributed lock от Redis.
- **N = 5** Redis, quorum.
- **Критика:** нет fencing, часы, split-brain.
- **Для критичных** — etcd/ZooKeeper.
- **Для производительности** — Redlock.

---

## 19.9 Idempotency: ключ к безопасному retry

**Idempotency** — свойство операции: **повторный вызов** даёт **тот же результат**.

### Зачем

**Retry** может вызвать операцию **дважды**:

- **Network timeout** — запрос дошёл, ответ нет.
- **Retry** — операция выполнена дважды.
- **Дубликаты.**

**Idempotency** решает: повторный вызов **безопасен**.

### Примеры

**Idempotent:**

- `GET /users/123` — чтение.
- `PUT /users/123` — установка значения.
- `DELETE /users/123` — удаление.

**Не idempotent:**

- `POST /users` — создание (каждый раз новый).
- `POST /orders` — создание заказа.
- `POST /transfer` — перевод денег.

### Idempotency key

**Идея:** клиент передаёт **уникальный ключ** для операции.

```
POST /orders
Idempotency-Key: abc-123
```

**Сервер:**

1. Проверяет, есть ли `abc-123` в БД.
2. Если да — возвращает **сохранённый** результат.
3. Если нет — выполняет, сохраняет результат.

### Реализация

```go
type IdempotencyStore struct {
    mu     sync.Mutex
    store  map[string]Result
}

func (s *IdempotencyStore) Do(key string, fn func() Result) Result {
    s.mu.Lock()
    if r, ok := s.store[key]; ok {
        s.mu.Unlock()
        return r
    }
    s.mu.Unlock()

    r := fn()

    s.mu.Lock()
    s.store[key] = r
    s.mu.Unlock()

    return r
}
```

**Проблема:** race condition. Два запроса с одним ключом.

**Решение:** singleflight:

```go
var group singleflight.Group

func (s *IdempotencyStore) Do(key string, fn func() Result) Result {
    v, _, _ := group.Do(key, func() (interface{}, error) {
        return fn(), nil
    })
    return v.(Result)
}
```

### Хранение

**Redis:**

```go
func (s *IdempotencyStore) Do(ctx context.Context, key string, fn func() Result) (Result, error) {
    // Проверяем
    if cached, err := s.redis.Get(ctx, key).Result(); err == nil {
        var r Result
        json.Unmarshal([]byte(cached), &r)
        return r, nil
    }

    // Выполняем
    r := fn()

    // Сохраняем с TTL
    data, _ := json.Marshal(r)
    s.redis.Set(ctx, key, data, 24*time.Hour)

    return r, nil
}
```

**Что делает:**

- Проверяет Redis.
- Если есть — возвращает.
- Если нет — выполняет, сохраняет.

### TTL

**Важно:** idempotency key должен **истекать**.

- **Слишком долго** — память растёт.
- **Слишком мало** — дубликаты.

**Рекомендация:** 24 часа — 7 дней.

### Аннотация сложности

| Операция | Time |
|:---|:---|
| Проверка в Redis | ~1-5 мс |
| Сохранение | ~1-5 мс |
| singleflight | ~100-200 нс |

### 💡 Практика: как использовать idempotency

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Idempotency-Key** для POST.
2. **TTL** для ключей.
3. **singleflight** для race.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Redis** для хранения.
5. **Метрики** — сколько дубликатов.

**❌ НЕ ДЕЛАЙ:**

6. **Не делай POST без idempotency.**
7. **Не храни ключи вечно.**

### Ключевые выводы подглавы 19.9

- **Idempotency** — повторный вызов = тот же результат.
- **Idempotency-Key** — уникальный ключ.
- **Redis** для хранения.
- **singleflight** для race.
- **TTL** — 24 часа — 7 дней.

---

## 19.10 Практика Go: retry с backoff и jitter

Напишем **полный retry**.

### Код

```go
package retry

import (
    "context"
    "errors"
    "math/rand"
    "time"
)

type Config struct {
    MaxAttempts int
    BaseDelay   time.Duration
    MaxDelay    time.Duration
    Multiplier  float64
    Jitter      JitterType
}

type JitterType int

const (
    NoJitter JitterType = iota
    FullJitter
    EqualJitter
    DecorrelatedJitter
)

func Do(ctx context.Context, cfg Config, fn func() error) error {
    if cfg.MaxAttempts <= 0 {
        cfg.MaxAttempts = 3
    }
    if cfg.BaseDelay <= 0 {
        cfg.BaseDelay = 100 * time.Millisecond
    }
    if cfg.MaxDelay <= 0 {
        cfg.MaxDelay = 30 * time.Second
    }
    if cfg.Multiplier <= 0 {
        cfg.Multiplier = 2.0
    }

    var lastErr error
    var prevDelay time.Duration

    for attempt := 0; attempt < cfg.MaxAttempts; attempt++ {
        lastErr = fn()
        if lastErr == nil {
            return nil
        }

        if !isRetryable(lastErr) {
            return lastErr
        }

        if attempt == cfg.MaxAttempts-1 {
            break
        }

        delay := calculateDelay(cfg, attempt, prevDelay)
        prevDelay = delay

        select {
        case <-time.After(delay):
        case <-ctx.Done():
            return ctx.Err()
        }
    }

    return lastErr
}

func calculateDelay(cfg Config, attempt int, prev time.Duration) time.Duration {
    var delay time.Duration

    switch cfg.Jitter {
    case NoJitter:
        delay = cfg.BaseDelay * time.Duration(pow(cfg.Multiplier, attempt))
    case FullJitter:
        base := cfg.BaseDelay * time.Duration(pow(cfg.Multiplier, attempt))
        delay = time.Duration(rand.Int63n(int64(base)))
    case EqualJitter:
        base := cfg.BaseDelay * time.Duration(pow(cfg.Multiplier, attempt))
        half := base / 2
        delay = half + time.Duration(rand.Int63n(int64(half)))
    case DecorrelatedJitter:
        if prev == 0 {
            delay = cfg.BaseDelay
        } else {
            upper := prev * 3
            if upper > cfg.MaxDelay {
                upper = cfg.MaxDelay
            }
            delay = cfg.BaseDelay + time.Duration(rand.Int63n(int64(upper-cfg.BaseDelay)))
        }
    }

    if delay > cfg.MaxDelay {
        delay = cfg.MaxDelay
    }

    return delay
}

func pow(base float64, exp int) float64 {
    result := 1.0
    for i := 0; i < exp; i++ {
        result *= base
    }
    return result
}

func isRetryable(err error) bool {
    if err == nil {
        return false
    }
    var retryable interface{ Retryable() bool }
    if errors.As(err, &retryable) {
        return retryable.Retryable()
    }
    return true
}
```

### Использование

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    cfg := retry.Config{
        MaxAttempts: 5,
        BaseDelay:   100 * time.Millisecond,
        MaxDelay:    10 * time.Second,
        Multiplier:  2.0,
        Jitter:      retry.DecorrelatedJitter,
    }

    err := retry.Do(ctx, cfg, func() error {
        return fetchFromAPI()
    })
    if err != nil {
        log.Fatal(err)
    }
}
```

### Что демонстрирует

1. **Exponential backoff.**
2. **Cap** на максимум.
3. **Jitter** (4 типа).
4. **`context`** для отмены.
5. **`isRetryable`** для фильтрации.

### Аннотация сложности

| Компонент | Time |
|:---|:---|
| `Do` | ~100 нс + retry |
| `calculateDelay` | ~50-100 нс |
| `time.After` | ~100-200 нс + таймер |

### 💡 Практика: как использовать retry

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Exponential backoff.**
2. **Jitter** для distributed.
3. **Cap** на максимум.
4. **`context`** для отмены.

**👍 СТОИТ СДЕЛАТЬ:**

5. **`isRetryable`** для фильтрации.
6. **Метрики** — сколько retry.

**❌ НЕ ДЕЛАЙ:**

7. **Не retry 4xx.**
8. **Не retry без jitter.**
9. **Не retry бесконечно.**

### Ключевые выводы подглавы 19.10

- **Config** — параметры retry.
- **4 типа jitter.**
- **Decorrelated** — лучший.
- **`context`** для отмены.
- **`isRetryable`** для фильтрации.

---

## 19.11 Выводы и типичные ошибки

**Что мы узнали?**

Bulkhead изолирует ресурсы. Retry с exponential backoff и jitter. Jitter критичен для distributed. Leader election через etcd. Split-brain и fencing tokens. Distributed locks через Redis/etcd. Redlock спорный. Idempotency — ключ к безопасному retry.

**Типичные ошибки:**

- ❌ **Retry без backoff.** Thundering herd.
- ❌ **Retry без jitter.** Синхронные retry.
- ❌ **Retry 4xx.** Не поможет.
- ❌ **Retry бесконечно.** OOM.
- ❌ **Общий пул** для всех зависимостей. Bulkhead нужен.
- ❌ **Leader election без consensus.** Split-brain.
- ❌ **Distributed lock без TTL.** Deadlock.
- ❌ **Distributed lock без fencing.** Stale operations.
- ❌ **Redlock для критичных.** etcd лучше.
- ❌ **POST без idempotency.** Дубликаты.
- ❌ **Idempotency key без TTL.** Память растёт.
- ❌ **Не использовать `context`.** Retry не отменяется.

---

## 19.12 Для быстрого повторения

- **Bulkhead** — изоляция ресурсов.
- **Bulkhead vs circuit breaker vs rate limiter** — разные паттерны.
- **Retry с exponential backoff** — задержка растёт.
- **Cap** — максимум.
- **Jitter** — случайная добавка.
- **Full, Equal, Decorrelated** — 3 типа jitter.
- **Leader election** — выбор лидера.
- **Safety, liveness, fault-tolerant.**
- **etcd election** — session + lease.
- **Split-brain** — два лидера.
- **Fencing tokens** — monotonic token.
- **Distributed locks** — Redis (SetNX + Lua), etcd (Raft).
- **Redlock** — N=5 Redis, спорный.
- **Idempotency** — повторный вызов = тот же результат.
- **Idempotency-Key** — уникальный ключ.
- **`singleflight`** — для race.
- **`context`** — для отмены retry.

---

## 19.13 Вопросы для самопроверки

1. Что такое bulkhead?
2. Чем bulkhead отличается от circuit breaker?
3. Что такое exponential backoff?
4. Зачем cap?
5. Что такое jitter?
6. Какие 3 типа jitter?
7. Почему jitter критичен?
8. Что такое leader election?
9. Какие свойства у leader election?
10. Как работает etcd election?
11. Что такое split-brain?
12. Что такое fencing tokens?
13. Что такое distributed locks?
14. Как работает Redis lock?
15. Как работает etcd lock?
16. Что такое Redlock?
17. Почему Redlock спорный?
18. Что такое idempotency?
19. Что такое Idempotency-Key?
20. Что такое singleflight?

---

## 19.14 Ответы

### Ответ 1

**Bulkhead** — паттерн изоляции ресурсов. Свой пул для каждой зависимости.

### Ответ 2

**Bulkhead** изолирует ресурсы. **Circuit breaker** прекращает вызовы при сбое.

### Ответ 3

**Exponential backoff** — задержка растёт с каждой попыткой: 100ms → 200ms → 400ms.

### Ответ 4

**Cap** — максимум задержки. Иначе backoff растёт бесконечно.

### Ответ 5

**Jitter** — случайная добавка к backoff.

### Ответ 6

**Три типа:** Full, Equal, Decorrelated.

### Ответ 7

**Jitter критичен**, потому что без него retry **синхронные** — thundering herd.

### Ответ 8

**Leader election** — выбор одного узла как лидера.

### Ответ 9

**Свойства:** safety (не более одного), liveness (в итоге один), fault-tolerant.

### Ответ 10

**etcd election:** session + lease + campaign.

### Ответ 11

**Split-brain** — два узла думают, что они лидеры.

### Ответ 12

**Fencing tokens** — monotonic token для проверки, что лидер актуален.

### Ответ 13

**Distributed locks** — блокировки между узлами.

### Ответ 14

**Redis lock:** `SetNX` + `TTL` + Lua для release.

### Ответ 15

**etcd lock:** `concurrency.NewMutex`, Raft-based.

### Ответ 16

**Redlock** — distributed lock от Redis. N=5 Redis, quorum.

### Ответ 17

**Redlock спорный**, потому что нет fencing tokens, зависит от часов, не решает split-brain.

### Ответ 18

**Idempotency** — повторный вызов даёт тот же результат.

### Ответ 19

**Idempotency-Key** — уникальный ключ для операции.

### Ответ 20

**singleflight** — библиотека для дедупликации одновременных запросов.

---

## 19.15 Куда идти дальше?

Мы разобрали bulkhead, retry, leader election, distributed locks, idempotency. Теперь мы умеем строить отказоустойчивые системы.

Но остаётся **следующая тема**: как обрабатывать **real-time** соединения?

- **Как обрабатывать WebSocket?** → **Глава 20: WebSocket, SSE, gRPC streaming.**
- **Как писать безопасный конкурентный код?** TOCTOU, timing attacks. → **Глава 21: Безопасность конкурентного кода.**
- **Как строить распределённые системы?** CAP, consensus. → **Глава 22: Распределённые системы — введение.**

---

## 19.16 Чек-лист

| Паттерн | Что делает | Ключевые факты |
|:---|:---|:---|
| **Bulkhead** | Изоляция ресурсов | Свой пул на зависимость |
| **Circuit breaker** | Прекращает вызовы | Три состояния |
| **Rate limiter** | Ограничивает скорость | Token bucket |
| **Retry** | Повтор при ошибке | Exponential backoff |
| **Backoff** | Задержка растёт | 100 → 200 → 400 мс |
| **Cap** | Максимум | 10-30 сек |
| **Jitter** | Случайная добавка | Full, Equal, Decorrelated |
| **Leader election** | Выбор лидера | etcd, Consul |
| **Session** | Lease с TTL | 10-30 сек |
| **Campaign** | Попытка стать | Блокируется |
| **Fencing tokens** | Безопасность | Monotonic |
| **Split-brain** | Два лидера | Fencing tokens |
| **Redis lock** | SetNX + Lua | Быстро |
| **etcd lock** | Raft-based | Надёжно |
| **Redlock** | N=5 Redis | Спорный |
| **Idempotency** | Повтор = тот же результат | Idempotency-Key |
| **singleflight** | Дедупликация | Для race |
| **`context`** | Отмена | Везде |

🛡️ **Ключевая идея:** **Bulkhead** изолирует ресурсы — свой пул на зависимость. **Retry с exponential backoff** — задержка растёт. **Cap** — максимум. **Jitter** — случайная добавка; Full, Equal, Decorrelated; без него thundering herd. **Leader election** через etcd — session + lease + campaign. **Split-brain** — два лидера; **fencing tokens** для безопасности. **Distributed locks** — Redis (SetNX + Lua), etcd (Raft). **Redlock** спорный — для критичных etcd лучше. **Idempotency** — повторный вызов = тот же результат; Idempotency-Key + `singleflight`. **`context`** — везде для отмены.