# 🧩 Глава 16: Sharded locks и lock-free структуры

**Что вы узнаете:**
- Почему один мьютекс на структуру — **bottleneck**.
- Что такое **sharded locks** и как они уменьшают contention.
- Как устроена **sharded map** и когда она быстрее `sync.Map`.
- Что такое **lock-free** структуры и почему они сложнее mutex.
- Как работает **Treiber stack** — lock-free стек.
- Что такое **Michael-Scott queue** — lock-free очередь.
- Что такое **ABA problem** и как с ним бороться.
- Что такое **hazard pointers** и зачем они нужны.
- Когда **lock-free хуже** обычного `Mutex`.

**После прочтения вы сможете:**
- Построить sharded map с N шардами.
- Понимать, когда sharding оправдан, а когда — нет.
- Реализовать Treiber stack и Michael-Scott queue.
- Распознавать ABA problem и знать, как её избежать.
- Осознанно выбирать между `Mutex`, sharding и lock-free.
- Бенчмаркать конкурентные структуры данных.

---

## Содержание

- [16.0 Пролог: map, которая упирается в один мьютекс](#160-пролог-map-которая-упирается-в-один-мьютекс)
- [16.1 Проблема: contention на одном мьютексе](#161-проблема-contention-на-одном-мьютексе)
- [16.2 Sharded locks: идея и реализация](#162-sharded-locks-идея-и-реализация)
- [16.3 Sharded map: полная реализация](#163-sharded-map-полная-реализация)
- [16.4 Когда sharding оправдан](#164-когда-sharding-оправдан)
- [16.5 Lock-free структуры: идея](#165-lock-free-структуры-идея)
- [16.6 Treiber stack: lock-free стек](#166-treiber-stack-lock-free-стек)
- [16.7 Michael-Scott queue: lock-free очередь](#167-michael-scott-queue-lock-free-очередь)
- [16.8 ABA problem](#168-aba-problem)
- [16.9 Hazard pointers](#169-hazard-pointers)
- [16.10 Когда lock-free хуже Mutex](#1610-когда-lock-free-хуже-mutex)
- [16.11 Практика Go: бенчмарки конкурентных структур](#1611-практика-go-бенчмарки-конкурентных-структур)
- [16.12 Выводы и типичные ошибки](#1612-выводы-и-типичные-ошибки)
- [16.13 Для быстрого повторения](#1613-для-быстрого-повторения)
- [16.14 Вопросы для самопроверки](#1614-вопросы-для-самопроверки)
- [16.15 Ответы](#1615-ответы)
- [16.16 Куда идти дальше?](#1616-куда-идти-дальше)
- [16.17 Чек-лист](#1617-чек-лист)

---

## 16.0 Пролог: map, которая упирается в один мьютекс

Ты пишешь кэш на 10 миллионов ключей. 100 горутин читают и пишут одновременно.

```go
type Cache struct {
    mu sync.RWMutex
    m  map[string]Value
}

func (c *Cache) Get(key string) (Value, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.m[key]
    return v, ok
}

func (c *Cache) Set(key string, v Value) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.m[key] = v
}
```

Профилирование показывает: **80% времени в `Mutex.Lock/Unlock`**. Contention убивает производительность.

❓ **Что произошло?** Все 100 горутин конкурируют за **один** мьютекс. Даже если они работают с **разными** ключами, мьютекс сериализует их.

💡 **Решение:** **sharded locks**. Разбить map на N частей, каждая со своим мьютексом. Горутины, работающие с разными частями, **не конкурируют**.

```go
type ShardedCache struct {
    shards [256]struct {
        mu sync.RWMutex
        m  map[string]Value
    }
}

func (c *ShardedCache) shard(key string) *shard {
    h := fnv.New32a()
    h.Write([]byte(key))
    return &c.shards[h.Sum32()%256]
}
```

**Что изменилось:** contention упал в **256 раз** (по числу шардов).

**Это sharded locks.** В этой главе — как их строить, когда применять и чем они отличаются от lock-free.

> **Важный мост:** sharding — **простейший** способ уменьшить contention. Lock-free — сложнее, но даёт больше. Глава 18 (GC) — почему lock-free может быть дороже. Глава 21 (Безопасность) — почему lock-free опасен.

---

## 16.1 Проблема: contention на одном мьютексе

Прежде чем разбирать sharding, поймём **проблему**.

### Что такое contention

**Contention** (конкуренция) — ситуация, когда несколько горутин конкурируют за **один** ресурс (мьютекс, кэш-линию, CPU).

**В Главе 3** мы разбирали contention на `Mutex`: горутины уходят в slow path, spin, `semaRoot`.

**Формула:**

```
Время ожидания ≈ O(N × T_hold)

где N — число горутин, конкурирующих за мьютекс,
    T_hold — среднее время удержания мьютекса.
```

**Пример:** 100 горутин, каждая держит мьютекс 1 мкс. Последняя ждёт 99 мкс.

### Contention на map

**Проблема:** один `RWMutex` на всю map.

```go
type Cache struct {
    mu sync.RWMutex
    m  map[string]Value
}
```

**Что происходит:**

- 100 горутин читают и пишут.
- `RWMutex` позволяет читателям **параллельно**, но писатель **блокирует всех**.
- Если писателей много — contention растёт.
- Если читателей много, но писатель частый — тоже.

### Как измерить contention

**Mutex profile:**

```go
runtime.SetMutexProfileFraction(1)
```

```bash
go tool pprof mutex.prof
```

**Что искать:**

- `sync.(*Mutex).Lock` в top.
- `sync.(*RWMutex).Lock` в top.

**Block profile:**

```go
runtime.SetBlockProfileRate(1)
```

```bash
go tool pprof block.prof
```

**Что искать:**

- Время, проведённое в ожидании.

### Как уменьшить contention

**1. Sharding.**

Разбить структуру на N частей, каждая со своим мьютексом.

**2. Lock-free.**

Использовать `atomic` вместо `Mutex`.

**3. Разделение мьютексов.**

Отдельные мьютексы для разных данных.

**4. Read-only копии.**

Использовать `atomic.Pointer` для снапшотов.

### Аннотация сложности

| Подход | Contention |
|:---|:---|
| Один `Mutex` | O(N) |
| `RWMutex` | O(писателей × N) |
| Sharding на 256 | O(N/256) |
| Lock-free | 0 (но CAS contention) |

### 💡 Практика: как обнаружить contention

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Mutex profile** — `SetMutexProfileFraction(1)`.
2. **Block profile** — `SetBlockProfileRate(1)`.
3. **pprof** — `go tool pprof`.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики** — latency операций.
5. **Бенчмарки** с разными `GOMAXPROCS`.

**❌ НЕ ДЕЛАЙ:**

6. **Не оптимизируй преждевременно.** Contention виден в профиле.
7. **Не игнорируй высокий contention.**

### Ключевые выводы подглавы 16.1

- **Contention** — конкуренция за один ресурс.
- **Формула:** O(N × T_hold).
- **Один мьютекс на map** — bottleneck.
- **Mutex profile** — для обнаружения.
- **Sharding** — простейшее решение.

---

## 16.2 Sharded locks: идея и реализация

**Sharded locks** — разбиение структуры на N частей, каждая со своим мьютексом.

### Идея

```
Без sharding:
  ┌─────────────────────────────────────────────┐
  │              Один RWMutex                    │
  │  ┌─────────────────────────────────────┐    │
  │  │  map (все ключи)                     │    │
  │  └─────────────────────────────────────┘    │
  └─────────────────────────────────────────────┘
  Все горутины конкурируют за один мьютекс.

С sharding (N=4):
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  RWMutex 0   │  │  RWMutex 1   │  │  RWMutex 2   │  │  RWMutex 3   │
  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │
  │  │ map 0  │  │  │  │ map 1  │  │  │  │ map 2  │  │  │  │ map 3  │  │
  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │
  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
  Горутины, работающие с разными шардами, не конкурируют.
```

### Как выбрать шард

**Хэш-функция** определяет, в какой шард попадёт ключ.

```go
func (c *ShardedCache) shard(key string) int {
    h := fnv.New32a()
    h.Write([]byte(key))
    return int(h.Sum32() % uint32(len(c.shards)))
}
```

**Что важно:**

- **Равномерное распределение.** Ключи должны попадать в шарды равномерно.
- **Быстрый хэш.** `fnv`, `xxhash`, `maphash`.
- **Стабильность.** Один и тот же ключ → один и тот же шард.

### Хэш-функции в Go

**1. `hash/fnv`** — простой, быстрый.

```go
import "hash/fnv"

h := fnv.New32a()
h.Write([]byte(key))
sum := h.Sum32()
```

**2. `hash/maphash`** — стандартный, быстрый.

```go
import "hash/maphash"

var seed = maphash.MakeSeed()

func hash(key string) uint64 {
    return maphash.String(seed, key)
}
```

**3. `xxhash`** — очень быстрый, но внешний.

```go
import "github.com/cespare/xxhash/v2"

sum := xxhash.Sum64String(key)
```

**4. `fnv` vs `maphash`:**

- `fnv` — детерминированный, простой.
- `maphash` — использует случайный seed (защита от hash DoS).

### Реализация простого sharded cache

```go
type ShardedCache struct {
    shards []*shard
}

type shard struct {
    mu sync.RWMutex
    m  map[string]Value
}

func NewShardedCache(numShards int) *ShardedCache {
    c := &ShardedCache{
        shards: make([]*shard, numShards),
    }
    for i := range c.shards {
        c.shards[i] = &shard{
            m: make(map[string]Value),
        }
    }
    return c
}

func (c *ShardedCache) shardFor(key string) *shard {
    h := fnv.New32a()
    h.Write([]byte(key))
    return c.shards[h.Sum32()%uint32(len(c.shards))]
}

func (c *ShardedCache) Get(key string) (Value, bool) {
    s := c.shardFor(key)
    s.mu.RLock()
    defer s.mu.RUnlock()
    v, ok := s.m[key]
    return v, ok
}

func (c *ShardedCache) Set(key string, v Value) {
    s := c.shardFor(key)
    s.mu.Lock()
    defer s.mu.Unlock()
    s.m[key] = v
}

func (c *ShardedCache) Delete(key string) {
    s := c.shardFor(key)
    s.mu.Lock()
    defer s.mu.Unlock()
    delete(s.m, key)
}
```

**Что делает:**

1. Создаёт N шардов, каждый со своим `RWMutex` и map.
2. `shardFor(key)` — определяет шард по ключу.
3. Операции работают с **одним** шардом.

### Сколько шардов

**Правило:** **N = 2^k**, где k — степень двойки, близкая к числу CPU.

**Примеры:**

- 8 ядер → 16 шардов.
- 16 ядер → 32 шарда.
- 64 ядра → 128 шардов.

**Почему степень двойки:**

- `h.Sum32() % N` — быстро для N = 2^k.
- Можно заменить на `h & (N - 1)`.

**Слишком много шардов:**

- Больше памяти.
- Больше кэш-промахов.

**Слишком мало:**

- Contention.

### Оптимизация: shardFor через маску

```go
func (c *ShardedCache) shardFor(key string) *shard {
    h := fnv.New32a()
    h.Write([]byte(key))
    return c.shards[h.Sum32()&uint32(len(c.shards)-1)]
}
```

**Что изменилось:** `% N` → `& (N-1)`. Работает для N = 2^k.

### Аннотация сложности

| Операция | Time (N=256) |
|:---|:---|
| `shardFor` | ~20-50 нс |
| `Get` | ~50-100 нс |
| `Set` | ~50-100 нс |

### 💡 Практика: как строить sharded locks

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **N = 2^k**, где k — степень двойки.
2. **N ≈ 2 × число ядер.**
3. **Быстрый хэш** — `fnv`, `maphash`.
4. **Равномерное распределение.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **`& (N-1)`** вместо `% N` для N = 2^k.
6. **Бенчмаркай** разные N.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй sharding для малых структур.** Overhead.
8. **Не забывай про качество хэша.**

### Ключевые выводы подглавы 16.2

- **Sharding** — разбиение на N частей.
- **Хэш-функция** определяет шард.
- **N = 2^k**, близко к числу ядер.
- **`& (N-1)`** вместо `% N`.
- **Быстрый хэш** — `fnv`, `maphash`.

---

## 16.3 Sharded map: полная реализация

Соберём **полную sharded map**.

### Полный код

```go
package sharded

import (
    "hash/fnv"
    "sync"
)

type Map[K comparable, V any] struct {
    shards []*shard[K, V]
    mask   uint32
}

type shard[K comparable, V any] struct {
    mu sync.RWMutex
    m  map[K]V
}

func New[K comparable, V any](numShards int) *Map[K, V] {
    if numShards <= 0 {
        numShards = 1
    }
    // Округляем до степени двойки
    n := 1
    for n < numShards {
        n <<= 1
    }

    m := &Map[K, V]{
        shards: make([]*shard[K, V], n),
        mask:   uint32(n - 1),
    }
    for i := range m.shards {
        m.shards[i] = &shard[K, V]{
            m: make(map[K]V),
        }
    }
    return m
}

func (m *Map[K, V]) shardFor(key K) *shard[K, V] {
    h := fnv.New32a()
    // Для generic K используем fmt
    h.Write([]byte(fmt.Sprintf("%v", key)))
    return m.shards[h.Sum32()&m.mask]
}

func (m *Map[K, V]) Get(key K) (V, bool) {
    s := m.shardFor(key)
    s.mu.RLock()
    defer s.mu.RUnlock()
    v, ok := s.m[key]
    return v, ok
}

func (m *Map[K, V]) Set(key K, value V) {
    s := m.shardFor(key)
    s.mu.Lock()
    defer s.mu.Unlock()
    s.m[key] = value
}

func (m *Map[K, V]) Delete(key K) {
    s := m.shardFor(key)
    s.mu.Lock()
    defer s.mu.Unlock()
    delete(s.m, key)
}

func (m *Map[K, V]) Len() int {
    total := 0
    for _, s := range m.shards {
        s.mu.RLock()
        total += len(s.m)
        s.mu.RUnlock()
    }
    return total
}

func (m *Map[K, V]) Range(fn func(K, V) bool) {
    for _, s := range m.shards {
        s.mu.RLock()
        for k, v := range s.m {
            if !fn(k, v) {
                s.mu.RUnlock()
                return
            }
        }
        s.mu.RUnlock()
    }
}
```

### Разберём по шагам

#### Шаг 1: округление до степени двойки

```go
n := 1
for n < numShards {
    n <<= 1
}
```

**Что делает:** находит ближайшую степень двойки ≥ `numShards`.

**Пример:** `numShards = 100` → `n = 128`.

#### Шаг 2: маска

```go
mask: uint32(n - 1)
```

**Что делает:** `n = 128` → `mask = 127` (0b01111111).

**Использование:** `h.Sum32() & mask` эквивалентно `h.Sum32() % n`.

#### Шаг 3: хэш для generic K

```go
h := fnv.New32a()
h.Write([]byte(fmt.Sprintf("%v", key)))
```

**Проблема:** для generic `K` нельзя напрямую хэшировать.

**Решение:** `fmt.Sprintf("%v", key)` — преобразует в строку.

**Минус:** медленно для не-строк.

**Альтернатива:** использовать интерфейс `Hasher`:

```go
type Hasher interface {
    Hash() uint64
}

func (m *Map[K, V]) shardFor(key K) *shard[K, V] {
    if h, ok := any(key).(Hasher); ok {
        return m.shards[h.Hash()&uint64(m.mask)]
    }
    // fallback
    ...
}
```

#### Шаг 4: Get с RLock

```go
func (m *Map[K, V]) Get(key K) (V, bool) {
    s := m.shardFor(key)
    s.mu.RLock()
    defer s.mu.RUnlock()
    v, ok := s.m[key]
    return v, ok
}
```

**Что делает:** читает из одного шарда.

**Почему `RLock`:** читатели параллельны.

#### Шаг 5: Set с Lock

```go
func (m *Map[K, V]) Set(key K, value V) {
    s := m.shardFor(key)
    s.mu.Lock()
    defer s.mu.Unlock()
    s.m[key] = value
}
```

**Что делает:** пишет в один шард.

**Почему `Lock`:** писатель блокирует шард.

#### Шаг 6: Len

```go
func (m *Map[K, V]) Len() int {
    total := 0
    for _, s := range m.shards {
        s.mu.RLock()
        total += len(s.m)
        s.mu.RUnlock()
    }
    return total
}
```

**Что делает:** суммирует размеры всех шардов.

**Важно:** не атомарно. Между шардами map может измениться.

#### Шаг 7: Range

```go
func (m *Map[K, V]) Range(fn func(K, V) bool) {
    for _, s := range m.shards {
        s.mu.RLock()
        for k, v := range s.m {
            if !fn(k, v) {
                s.mu.RUnlock()
                return
            }
        }
        s.mu.RUnlock()
    }
}
```

**Что делает:** проходит по всем шардам.

**Важно:** блокирует **один** шард за раз. Не весь map.

### Использование

```go
func main() {
    m := New[string, int](256)

    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 1000; j++ {
                key := fmt.Sprintf("key-%d-%d", id, j)
                m.Set(key, id*1000+j)
            }
        }(i)
    }
    wg.Wait()

    fmt.Println("len:", m.Len())

    count := 0
    m.Range(func(k string, v int) bool {
        count++
        return true
    })
    fmt.Println("range:", count)
}
```

### Сравнение с `sync.Map`

| Аспект | `sync.Map` | Sharded map |
|:---|:---|:---|
| Read-heavy | ✅ Хорошо | ✅ Хорошо |
| Write-heavy | ❌ Плохо | ✅ Хорошо |
| Range | Атомарный | Не атомарный |
| Память | ~48 байт | ~N × 100 байт |
| Типизация | `any` | Generic |

**Рекомендация:**

- **Read-heavy** — `sync.Map`.
- **Write-heavy** — sharded map.
- **Смешанная** — бенчмаркай.

### Аннотация сложности

| Операция | Time (N=256) |
|:---|:---|
| `shardFor` | ~30-50 нс |
| `Get` | ~50-100 нс |
| `Set` | ~50-100 нс |
| `Delete` | ~50-100 нс |
| `Len` | ~N × 50 нс |
| `Range` | ~N × 100 нс |

### 💡 Практика: как строить sharded map

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Generic** — для типобезопасности.
2. **N = 2^k** — для быстрой маски.
3. **`& mask`** вместо `% N`.
4. **RWMutex** в каждом шарде.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Hasher-интерфейс** для быстрого хэша.
6. **Len/Range** — для диагностики.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `fmt.Sprintf`** для hot path.
8. **Не забывай про качество хэша.**

### Ключевые выводы подглавы 16.3

- **Sharded map** — N шардов, каждый с `RWMutex`.
- **Generic** — для типобезопасности.
- **`& mask`** — быстро.
- **Write-heavy** — sharded map. **Read-heavy** — `sync.Map`.

---

## 16.4 Когда sharding оправдан

Разберём **когда** sharding **оправдан**.

### Оправдан

**1. Высокий contention.**

Если mutex profile показывает высокий contention — sharding поможет.

**2. Write-heavy нагрузка.**

`RWMutex` не помогает, если писателей много. Sharding — да.

**3. Большая map.**

Для 10 миллионов ключей — sharding уменьшает contention.

**4. Разные ключи.**

Если горутины работают с разными ключами — sharding разведёт их по шардам.

**5. Долгие операции.**

Если операции долгие — contention растёт. Sharding помогает.

### Не оправдан

**1. Низкий contention.**

Если все операции быстрые и contention низкий — sharding добавит overhead.

**2. Маленькая map.**

Для 100 ключей — sharding не поможет.

**3. Read-only нагрузка.**

Если только чтение — `RWMutex` справляется. Sharding не нужен.

**4. Один шард.**

Если все ключи попадают в один шард (плохой хэш) — sharding бесполезен.

**5. Операции над несколькими ключами.**

Если операция захватывает несколько ключей — sharding сложен.

### Сравнение

| Сценарий | Один Mutex | Sharding | sync.Map |
|:---|:---|:---|:---|
| Read-only | ✅ | ✅ | ✅ |
| Read-heavy | ✅ | ✅ | ✅ |
| Write-heavy | ❌ | ✅ | ❌ |
| Мало ключей | ✅ | ❌ | ✅ |
| Много ключей | ❌ | ✅ | ⚠️ |

### Как измерить

**1. Mutex profile.**

```go
runtime.SetMutexProfileFraction(1)
```

**Что искать:** `Mutex.Lock` в top.

**2. Бенчмарки.**

```go
func BenchmarkMap(b *testing.B) {
    m := New[string, int](256)
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            m.Set("key", 1)
            m.Get("key")
        }
    })
}
```

**Что искать:** scaling с `GOMAXPROCS`.

### Аннотация сложности

| Сценарий | Один Mutex | Sharding |
|:---|:---|:---|
| 8 ядер, write-heavy | ~1000 нс | ~100 нс |
| 8 ядер, read-only | ~50 нс | ~50 нс |

### 💡 Практика: когда использовать sharding

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Mutex profile** — для обнаружения contention.
2. **Бенчмаркай** перед sharding.
3. **Write-heavy** — sharding.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Разные N** — для оптимизации.
5. **Метрики** — latency.

**❌ НЕ ДЕЛАЙ:**

6. **Не делай sharding без причины.**
7. **Не забывай про качество хэша.**

### Ключевые выводы подглавы 16.4

- **Sharding оправдан** при высоком contention, write-heavy, большой map.
- **Не оправдан** при низком contention, read-only, малой map.
- **Mutex profile** — для обнаружения.
- **Бенчмаркай** перед sharding.

---

## 16.5 Lock-free структуры: идея

**Lock-free** — структуры данных **без мьютексов**.

### Идея

**Обычный mutex:**

```
Горутина A: Lock → работа → Unlock
Горутина B: ────ЖДЁТ───────Lock → работа → Unlock
```

**Lock-free:**

```
Горутина A: CAS → работа → CAS
Горутина B: CAS → работа → CAS
```

**Ключевое:** обе горутины **не блокируются**. Используют `atomic.CompareAndSwap`.

### Три уровня

**1. Blocking.**

- Мьютексы.
- Горутина **блокируется**, пока другой не отпустит.

**2. Lock-free.**

- Хотя бы **одна** горутина делает прогресс.
- `CAS` retry loop.

**3. Wait-free.**

- **Все** горутины делают прогресс за **ограниченное** число шагов.
- Очень сложно.

### CAS: Compare-And-Swap

**Основа lock-free:**

```go
func CompareAndSwap(addr *T, old, new T) bool
```

**Алгоритм:**

1. Прочитать `*addr`.
2. Если `*addr == old` — записать `new`, вернуть `true`.
3. Иначе — вернуть `false`.

**Всё атомарно.**

### Retry loop

**Классический паттерн:**

```go
for {
    old := atomic.Load(&value)
    new := compute(old)
    if atomic.CompareAndSwap(&value, old, new) {
        return
    }
    // Не удалось — кто-то изменил value
    // Повторяем
}
```

**Что делает:**

1. Читаем текущее значение.
2. Вычисляем новое.
3. Пытаемся заменить через CAS.
4. Если не удалось — повторяем.

### Пример: lock-free счётчик

```go
type Counter struct {
    value atomic.Int64
}

func (c *Counter) Inc() {
    c.value.Add(1)  // atomic.Add — lock-free
}

func (c *Counter) Get() int64 {
    return c.value.Load()
}
```

**Что делает:** `atomic.Add` — lock-free инкремент.

### Пример: lock-free update

```go
func updateIfEqual(addr *atomic.Int64, old, new int64) bool {
    return addr.CompareAndSwap(old, new)
}
```

### Преимущества lock-free

1. **Нет блокировок.** Горутины не ждут.
2. **Нет deadlock.** Нет мьютексов — нет взаимных блокировок.
3. **Нет starvation** (в теории).
4. **Лучше для real-time систем.**

### Недостатки lock-free

1. **CAS contention.** При высокой конкуренции CAS «проваливается».
2. **Сложность.** Трудно писать и отлаживать.
3. **ABA problem.** См. 16.8.
4. **Memory reclamation.** Когда освобождать узлы? См. 16.9.
5. **Может быть медленнее mutex.** При высокой конкуренции.

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `atomic.Load` | ~1-2 нс |
| `atomic.Store` | ~1-2 нс |
| `atomic.CAS` | ~5-15 нс |
| `atomic.Add` | ~5-10 нс |
| `Mutex.Lock/Unlock` | ~15-25 нс |

### 💡 Практика: как использовать lock-free

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`atomic` для простых случаев** — счётчики, флаги, указатели.
2. **CAS retry loop** — для сложных.
3. **Бенчмаркай** против mutex.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Простые структуры** — stack, queue.
5. **Hazard pointers** — для memory reclamation.

**❌ НЕ ДЕЛАЙ:**

6. **Не пиши lock-free без необходимости.**
7. **Не используй lock-free для сложных структур.**
8. **Не игнорируй ABA problem.**

### Ключевые выводы подглавы 16.5

- **Lock-free** — без мьютексов.
- **CAS** — основа.
- **Retry loop** — классический паттерн.
- **Плюсы:** нет блокировок, deadlock, starvation.
- **Минусы:** сложность, ABA, memory reclamation.

---

## 16.6 Treiber stack: lock-free стек

**Treiber stack** — простейшая lock-free структура.

### Идея

**Стек** — LIFO. Операции: `Push`, `Pop`.

**Treiber stack** использует `atomic.Pointer` на голову.

### Структура

```go
type node[T any] struct {
    value T
    next  *node[T]
}

type Stack[T any] struct {
    head atomic.Pointer[node[T]]
}
```

### Push

```go
func (s *Stack[T]) Push(value T) {
    n := &node[T]{value: value}
    for {
        old := s.head.Load()
        n.next = old
        if s.head.CompareAndSwap(old, n) {
            return
        }
        // Кто-то изменил head — повторяем
    }
}
```

**Что делает:**

1. Создаёт новый узел `n`.
2. Читает текущую голову.
3. `n.next = old` — новый узел указывает на старую голову.
4. CAS: если голова не изменилась — устанавливает `n` как новую голову.
5. Если изменилась — повторяет.

### Pop

```go
func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    for {
        old := s.head.Load()
        if old == nil {
            return zero, false
        }
        next := old.next
        if s.head.CompareAndSwap(old, next) {
            return old.value, true
        }
        // Кто-то изменил head — повторяем
    }
}
```

**Что делает:**

1. Читает текущую голову.
2. Если пусто — возвращает `zero, false`.
3. Читает `old.next`.
4. CAS: если голова не изменилась — устанавливает `next` как новую голову.
5. Если изменилась — повторяет.

### Полный пример

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
)

type node[T any] struct {
    value T
    next  *node[T]
}

type Stack[T any] struct {
    head atomic.Pointer[node[T]]
}

func (s *Stack[T]) Push(value T) {
    n := &node[T]{value: value}
    for {
        old := s.head.Load()
        n.next = old
        if s.head.CompareAndSwap(old, n) {
            return
        }
    }
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    for {
        old := s.head.Load()
        if old == nil {
            return zero, false
        }
        next := old.next
        if s.head.CompareAndSwap(old, next) {
            return old.value, true
        }
    }
}

func main() {
    var s Stack[int]

    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 100; j++ {
                s.Push(id*100 + j)
            }
        }(i)
    }
    wg.Wait()

    count := 0
    for {
        _, ok := s.Pop()
        if !ok {
            break
        }
        count++
    }
    fmt.Println("count:", count)  // 10000
}
```

### Проблемы Treiber stack

**1. ABA problem.**

См. 16.8.

**2. Memory reclamation.**

Когда освобождать `old` node? Если сразу — другая горутина может ещё держать указатель.

**3. Contention на head.**

Все операции — на `head`. При высокой конкуренции CAS «проваливается».

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `Push` (без contention) | ~10-20 нс |
| `Push` (с contention) | ~100-1000 нс |
| `Pop` (без contention) | ~10-20 нс |
| `Pop` (с contention) | ~100-1000 нс |

### 💡 Практика: как использовать Treiber stack

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`atomic.Pointer`** для головы.
2. **CAS retry loop.**
3. **Бенчмаркай** против mutex.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Hazard pointers** для memory reclamation.
5. **Метрики** — сколько retry.

**❌ НЕ ДЕЛАЙ:**

6. **Не освобождай узлы сразу** после Pop.
7. **Не игнорируй ABA problem.**

### Ключевые выводы подглавы 16.6

- **Treiber stack** — lock-free стек.
- **`atomic.Pointer`** для головы.
- **CAS retry loop.**
- **Проблемы:** ABA, memory reclamation, contention на head.

---

## 16.7 Michael-Scott queue: lock-free очередь

**Michael-Scott queue** — lock-free очередь.

### Идея

**Очередь** — FIFO. Операции: `Enqueue`, `Dequeue`.

**Michael-Scott** использует **два** указателя: `head` и `tail`.

**Проблема:** два указателя нужно обновлять атомарно.

### Структура

```go
type node[T any] struct {
    value T
    next  atomic.Pointer[node[T]]
}

type Queue[T any] struct {
    head atomic.Pointer[node[T]]
    tail atomic.Pointer[node[T]]
}
```

### Инициализация

```go
func NewQueue[T any]() *Queue[T] {
    q := &Queue[T]{}
    dummy := &node[T]{}
    q.head.Store(dummy)
    q.tail.Store(dummy)
    return q
}
```

**Ключевое:** есть **dummy node**. `head` и `tail` указывают на неё.

### Enqueue

```go
func (q *Queue[T]) Enqueue(value T) {
    n := &node[T]{value: value}
    for {
        tail := q.tail.Load()
        next := tail.next.Load()
        if tail == q.tail.Load() {  // tail не изменился
            if next == nil {
                // tail.next пуст — можно прицепить
                if tail.next.CompareAndSwap(nil, n) {
                    // Успех — пробуем сдвинуть tail
                    q.tail.CompareAndSwap(tail, n)
                    return
                }
            } else {
                // Кто-то уже добавил — сдвигаем tail
                q.tail.CompareAndSwap(tail, next)
            }
        }
    }
}
```

### Dequeue

```go
func (q *Queue[T]) Dequeue() (T, bool) {
    var zero T
    for {
        head := q.head.Load()
        tail := q.tail.Load()
        next := head.next.Load()
        if head == q.head.Load() {  // head не изменился
            if head == tail {
                // Очередь пуста?
                if next == nil {
                    return zero, false
                }
                // tail отстал — сдвигаем
                q.tail.CompareAndSwap(tail, next)
            } else {
                // Читаем значение
                value := next.value
                // Сдвигаем head
                if q.head.CompareAndSwap(head, next) {
                    return value, true
                }
            }
        }
    }
}
```

### Полный пример

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
)

type node[T any] struct {
    value T
    next  atomic.Pointer[node[T]]
}

type Queue[T any] struct {
    head atomic.Pointer[node[T]]
    tail atomic.Pointer[node[T]]
}

func NewQueue[T any]() *Queue[T] {
    q := &Queue[T]{}
    dummy := &node[T]{}
    q.head.Store(dummy)
    q.tail.Store(dummy)
    return q
}

func (q *Queue[T]) Enqueue(value T) {
    n := &node[T]{value: value}
    for {
        tail := q.tail.Load()
        next := tail.next.Load()
        if tail == q.tail.Load() {
            if next == nil {
                if tail.next.CompareAndSwap(nil, n) {
                    q.tail.CompareAndSwap(tail, n)
                    return
                }
            } else {
                q.tail.CompareAndSwap(tail, next)
            }
        }
    }
}

func (q *Queue[T]) Dequeue() (T, bool) {
    var zero T
    for {
        head := q.head.Load()
        tail := q.tail.Load()
        next := head.next.Load()
        if head == q.head.Load() {
            if head == tail {
                if next == nil {
                    return zero, false
                }
                q.tail.CompareAndSwap(tail, next)
            } else {
                value := next.value
                if q.head.CompareAndSwap(head, next) {
                    return value, true
                }
            }
        }
    }
}

func main() {
    q := NewQueue[int]()

    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 100; j++ {
                q.Enqueue(id*100 + j)
            }
        }(i)
    }
    wg.Wait()

    count := 0
    for {
        _, ok := q.Dequeue()
        if !ok {
            break
        }
        count++
    }
    fmt.Println("count:", count)  // 10000
}
```

### Проблемы Michael-Scott queue

**1. Сложность.**

Код сложнее, чем Treiber stack.

**2. ABA problem.**

См. 16.8.

**3. Memory reclamation.**

Когда освобождать узлы?

**4. Contention на head/tail.**

При высокой конкуренции — CAS contention.

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `Enqueue` (без contention) | ~20-50 нс |
| `Enqueue` (с contention) | ~100-1000 нс |
| `Dequeue` (без contention) | ~20-50 нс |
| `Dequeue` (с contention) | ~100-1000 нс |

### 💡 Практика: как использовать Michael-Scott queue

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Dummy node** — для упрощения.
2. **Два указателя** — `head` и `tail`.
3. **CAS** для обоих.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Hazard pointers** для memory reclamation.
5. **Бенчмаркай** против mutex-based queue.

**❌ НЕ ДЕЛАЙ:**

6. **Не пиши lock-free queue с нуля** без необходимости.
7. **Не игнорируй ABA problem.**

### Ключевые выводы подглавы 16.7

- **Michael-Scott queue** — lock-free очередь.
- **Dummy node** для упрощения.
- **Два указателя:** `head` и `tail`.
- **Проблемы:** сложность, ABA, memory reclamation.

---

## 16.8 ABA problem

**ABA problem** — классическая проблема lock-free структур.

### Что такое ABA

**Сценарий:**

1. Горутина A читает `value = X`.
2. Горутина B меняет `value = Y`, потом `value = X`.
3. Горутина A делает CAS: `value == X`, устанавливает `value = Z`.
4. **Успех**, но состояние **изменилось** между чтением и CAS.

**Проблема:** CAS не знает, что между чтением и записью что-то произошло.

### Пример: Treiber stack

**Сценарий:**

```
Начальное состояние: head → A → B → C

1. Горутина 1 читает head = A, next = B.
2. Горутина 1 приостанавливается.

3. Горутина 2 делает Pop: head → B → C.
4. Горутина 2 делает Pop: head → C.
5. Горутина 2 делает Push A: head → A → C.
   (узел A переиспользован)

6. Горутина 1 продолжает: CAS(head, A, B).
7. CAS успешен (head == A), head → B.
8. Но B уже не в стеке! B.next = C, но C может быть изменён.
```

**Результат:** стек повреждён.

### Как обнаружить

**ABA problem** сложно обнаружить:

- Проявляется **редко**.
- Зависит от timing.
- Может **не проявляться** годами.

**Race detector** может **не найти** — это не data race, а **логическая** ошибка.

### Решения

**1. Tagged pointers.**

Добавить **счётчик** к указателю. CAS проверяет и указатель, и счётчик.

```go
type taggedPtr struct {
    ptr   *node
    tag   uint64
}
```

**Проблема:** в Go нет 128-битных атомарных операций.

**2. Hazard pointers.**

См. 16.9.

**3. `atomic.Pointer` с version.**

```go
type versioned[T any] struct {
    value T
    ver   uint64
}
```

**4. Не переиспользовать узлы.**

Если узлы не переиспользуются (только новые) — ABA не возникает.

**Минус:** утечка памяти.

### В Go

**Go не поддерживает** tagged pointers напрямую.

**Альтернативы:**

- **Hazard pointers** — сложно.
- **`sync.Pool`** — не решает ABA.
- **Избегать lock-free** для сложных структур.

### Аннотация сложности

| Решение | Сложность |
|:---|:---|
| Tagged pointers | Высокая (нет в Go) |
| Hazard pointers | Высокая |
| Не переиспользовать | Низкая (но утечка) |
| Избегать lock-free | Низкая |

### 💡 Практика: как избежать ABA

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Понимай ABA** — это не data race.
2. **Избегай lock-free** для сложных структур.
3. **Используй hazard pointers** если нужно.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Тесты** на ABA (сложно).
5. **Код-ревью** lock-free структур.

**❌ НЕ ДЕЛАЙ:**

6. **Не пиши lock-free структуры без понимания ABA.**
7. **Не используй tagged pointers в Go.**

### Ключевые выводы подглавы 16.8

- **ABA problem** — CAS не видит изменения.
- **Сложно обнаружить.**
- **Решения:** tagged pointers, hazard pointers, не переиспользовать.
- **В Go** — избегать lock-free для сложных структур.

---

## 16.9 Hazard pointers

**Hazard pointers** — механизм для безопасного освобождения памяти в lock-free структурах.

### Проблема

**Memory reclamation:** когда освобождать узлы?

**Без hazard pointers:**

- Горутина A читает `head = X`.
- Горутина B удаляет X и освобождает.
- Горутина A обращается к X → **use-after-free**.

### Идея hazard pointers

**Каждая горутина объявляет** указатели, к которым обращается.

**Освобождение:**

- Узел освобождается, только если **никто** его не объявил hazard.

### Схема

```
Горутина A: hazard = X
Горутина B: hazard = Y
Горутина C: hazard = Z

Освобождение:
  - Узел X: НЕ освобождать (A объявил)
  - Узел Y: НЕ освобождать (B объявил)
  - Узел Z: НЕ освобождать (C объявил)
  - Узел W: освободить (никто не объявил)
```

### Реализация

**Упрощённая схема:**

```go
type HazardPointer struct {
    ptrs []atomic.Pointer[node]
}
```

**Каждая горутина:**

1. Объявляет hazard перед чтением.
2. Читает.
3. Снимает hazard после.

**При освобождении:**

1. Собирает все hazard pointers.
2. Если узел не в списке — освобождает.
3. Иначе — откладывает.

### Сложность

**Hazard pointers сложны:**

- Нужно управлять per-goroutine hazard pointers.
- Нужно сканировать hazard pointers при освобождении.
- Overhead на каждую операцию.

**В Go нет встроенной поддержки.**

**Библиотеки:**

- `github.com/romshark/...` — редко.
- Своя реализация — сложно.

### Альтернативы

**1. `sync.Pool`.**

Не решает ABA, но уменьшает аллокации.

**2. Не переиспользовать узлы.**

Утечка, но безопасно.

**3. Garbage collector.**

В Go GC **сам** освобождает память. Если нет явного `free` — ABA problem **не возникает** (потому что узлы не переиспользуются).

**Ключевое:** **в Go GC решает memory reclamation автоматически.** ABA problem **всё ещё возможен**, но **use-after-free** — нет.

### Аннотация сложности

| Механизм | Сложность | Overhead |
|:---|:---|:---|
| Hazard pointers | Высокая | ~10-50% |
| Не переиспользовать | Низкая | Память |
| GC (Go) | — | — |

### 💡 Практика: как использовать hazard pointers

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Понимай, что hazard pointers нужны для C/C++.**
2. **В Go GC решает большую часть проблем.**
3. **ABA остаётся** — используй tagged или избегай.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Библиотеки** если нужны.

**❌ НЕ ДЕЛАЙ:**

5. **Не пиши hazard pointers с нуля.**
6. **Не игнорируй ABA.**

### Ключевые выводы подглавы 16.9

- **Hazard pointers** — для memory reclamation.
- **Сложны** в реализации.
- **В Go GC решает** memory reclamation.
- **ABA остаётся** — используй tagged или избегай.

---

## 16.10 Когда lock-free хуже Mutex

Разберём **когда lock-free хуже**.

### 1. Высокая конкуренция

**Проблема:** CAS retry loop может «проваливаться» много раз.

**Пример:**

```
100 горутин делают CAS на одном указателе.
Только одна выигрывает.
99 повторяют.
```

**Что происходит:** CPU тратится на retry, а не на работу.

**Mutex** в этом случае **быстрее**: горутины спят, а не крутятся.

### 2. Сложные структуры

**Проблема:** lock-free map, tree — очень сложны.

**Пример:** lock-free hash map — сотни строк кода, сложная отладка.

**Mutex-based** map — просто и работает.

### 3. ABA problem

**Проблема:** ABA сложно обнаружить и решить.

**Mutex** не имеет ABA.

### 4. Memory reclamation

**Проблема:** в C/C++ нужно управлять памятью вручную.

**В Go** GC решает, но ABA остаётся.

### 5. Отладка

**Проблема:** lock-free структуры сложно отлаживать.

- Race detector может не найти.
- ABA проявляется редко.
- Логика сложная.

### 6. Бенчмарки

**Часто mutex быстрее.**

**Пример:**

```go
// Mutex-based stack
type MutexStack struct {
    mu sync.Mutex
    head *node
}

// Lock-free stack (Treiber)
type LockFreeStack struct {
    head atomic.Pointer[node]
}
```

**Бенчмарк (8 ядер, 100 горутин):**

```
BenchmarkMutexStack-8      1000000    1500 ns/op
BenchmarkLockFreeStack-8   1000000    2500 ns/op
```

**Что видно:** mutex **быстрее** из-за CAS contention.

### Сравнение

| Аспект | Mutex | Lock-free |
|:---|:---|:---|
| Низкая конкуренция | ~25 нс | ~10 нс |
| Высокая конкуренция | ~100-1000 нс | ~1000-10000 нс |
| Сложность | Низкая | Высокая |
| ABA | Нет | Есть |
| Memory reclamation | Нет | Сложно |
| Отладка | Легко | Сложно |

### Аннотация сложности

| Сценарий | Mutex | Lock-free |
|:---|:---|:---|
| 1 горутина | ~15-25 нс | ~10-20 нс |
| 8 горутин | ~100-200 нс | ~50-100 нс |
| 100 горутин | ~1000 нс | ~5000 нс |

### 💡 Практика: когда lock-free хуже

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Бенчмаркай** перед выбором.
2. **Mutex** для сложных структур.
3. **Lock-free** только для простых.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`atomic`** для счётчиков.
5. **Sharding** для map.

**❌ НЕ ДЕЛАЙ:**

6. **Не пиши lock-free без необходимости.**
7. **Не игнорируй contention.**

### Ключевые выводы подглавы 16.10

- **Lock-free хуже** при высокой конкуренции, сложных структурах, ABA.
- **Mutex** часто быстрее.
- **Бенчмаркай** перед выбором.
- **Lock-free** только для простых структур.

---

## 16.11 Практика Go: бенчмарки конкурентных структур

Напишем **бенчмарки** для сравнения.

### Код

```go
package benchmarks

import (
    "sync"
    "sync/atomic"
    "testing"
)

// Mutex-based counter
type MutexCounter struct {
    mu sync.Mutex
    n  int64
}

func (c *MutexCounter) Inc() {
    c.mu.Lock()
    c.n++
    c.mu.Unlock()
}

// Atomic counter
type AtomicCounter struct {
    n atomic.Int64
}

func (c *AtomicCounter) Inc() {
    c.n.Add(1)
}

// Sharded counter
type ShardedCounter struct {
    shards [256]struct {
        pad [56]byte
        n   atomic.Int64
    }
}

func (c *ShardedCounter) Inc() {
    // Для бенчмарка — просто round-robin
    c.shards[0].n.Add(1)
}

func BenchmarkMutexCounter(b *testing.B) {
    var c MutexCounter
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            c.Inc()
        }
    })
}

func BenchmarkAtomicCounter(b *testing.B) {
    var c AtomicCounter
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            c.Inc()
        }
    })
}

// Mutex-based stack
type MutexStack struct {
    mu   sync.Mutex
    head *node
}

type node struct {
    value int
    next  *node
}

func (s *MutexStack) Push(v int) {
    s.mu.Lock()
    s.head = &node{value: v, next: s.head}
    s.mu.Unlock()
}

// Treiber stack
type TreiberStack struct {
    head atomic.Pointer[node]
}

func (s *TreiberStack) Push(v int) {
    n := &node{value: v}
    for {
        old := s.head.Load()
        n.next = old
        if s.head.CompareAndSwap(old, n) {
            return
        }
    }
}

func BenchmarkMutexStack(b *testing.B) {
    var s MutexStack
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            s.Push(1)
        }
    })
}

func BenchmarkTreiberStack(b *testing.B) {
    var s TreiberStack
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            s.Push(1)
        }
    })
}
```

### Запуск

```bash
go test -bench=. -benchmem -cpu=1,2,4,8 ./...
```

### Пример вывода

```
BenchmarkMutexCounter-1     50000000    25 ns/op
BenchmarkMutexCounter-8     10000000   150 ns/op
BenchmarkAtomicCounter-1   100000000    10 ns/op
BenchmarkAtomicCounter-8    50000000    25 ns/op
BenchmarkMutexStack-1       30000000    40 ns/op
BenchmarkMutexStack-8        5000000   300 ns/op
BenchmarkTreiberStack-1     20000000    60 ns/op
BenchmarkTreiberStack-8      3000000   500 ns/op
```

**Что видно:**

- **`atomic` быстрее `Mutex`** в 6 раз при 8 ядрах.
- **`Mutex` stack быстрее Treiber** при 8 ядрах.

### Анализ

**`atomic` vs `Mutex`:**

- `atomic` — нет блокировки, нет contention (для одного счётчика).
- `Mutex` — contention растёт с `GOMAXPROCS`.

**Mutex stack vs Treiber:**

- Mutex — горутины спят.
- Treiber — CAS retry, contention на head.

### Аннотация сложности

| Структура | 1 ядро | 8 ядер |
|:---|:---|:---|
| Mutex counter | ~25 нс | ~150 нс |
| Atomic counter | ~10 нс | ~25 нс |
| Mutex stack | ~40 нс | ~300 нс |
| Treiber stack | ~60 нс | ~500 нс |

### 💡 Практика: как бенчмаркать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`b.RunParallel`** — для параллельных.
2. **`-cpu=1,2,4,8`** — разные GOMAXPROCS.
3. **`-benchmem`** — память.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Реалистичная нагрузка** — не только 1 операция.
5. **Разные размеры** — 10, 100, 1000 элементов.

**❌ НЕ ДЕЛАЙ:**

6. **Не сравнивай несравнимое.**
7. **Не игнорируй `-benchmem`.**

### Ключевые выводы подглавы 16.11

- **`b.RunParallel`** — для параллельных бенчмарков.
- **`atomic` быстрее `Mutex`** для счётчиков.
- **Mutex stack быстрее Treiber** при высокой конкуренции.
- **`-cpu=1,2,4,8`** — обязательно.

---

## 16.12 Выводы и типичные ошибки

**Что мы узнали?**

Contention на одном мьютексе — bottleneck. Sharding — разбиение на N частей, каждая со своим мьютексом. Sharded map — Generic, N = 2^k, `& mask`. Когда sharding оправдан: высокий contention, write-heavy, большая map. Lock-free — без мьютексов, через CAS. Treiber stack — lock-free стек. Michael-Scott queue — lock-free очередь. ABA problem — CAS не видит изменения. Hazard pointers — для memory reclamation. Lock-free хуже Mutex при высокой конкуренции и сложных структурах.

**Типичные ошибки:**

- ❌ **Один мьютекс на map.** Sharding.
- ❌ **Sharding без причины.** Overhead.
- ❌ **Плохой хэш.** Ключи в один шард.
- ❌ **`fmt.Sprintf` в hot path.** Медленно.
- ❌ **Не использовать `& mask`.** `% N` медленнее.
- ❌ **Писать lock-free без необходимости.** Сложно.
- ❌ **Игнорировать ABA problem.** Сложно отладить.
- ❌ **Не освобождать память в lock-free.** Утечка.
- ❌ **Освобождать сразу.** Use-after-free.
- ❌ **Lock-free для сложных структур.** Mutex проще.
- ❌ **Не бенчмаркать.** Может быть медленнее.
- ❌ **Не использовать `-cpu=1,2,4,8`.**

---

## 16.13 Для быстрого повторения

- **Contention** — конкуренция за один ресурс.
- **Sharding** — N частей, каждая со своим мьютексом.
- **N = 2^k**, близко к числу ядер.
- **`& mask`** вместо `% N`.
- **Хэш:** `fnv`, `maphash`.
- **Sharded map** — Generic, RWMutex в каждом шарде.
- **Когда оправдан:** высокий contention, write-heavy, большая map.
- **Lock-free** — через CAS.
- **Treiber stack** — lock-free стек.
- **Michael-Scott queue** — lock-free очередь.
- **ABA problem** — CAS не видит изменения.
- **Hazard pointers** — для memory reclamation.
- **Lock-free хуже Mutex** при высокой конкуренции.
- **Бенчмаркай** с `-cpu=1,2,4,8`.
- **`atomic` быстрее `Mutex`** для счётчиков.
- **Mutex stack быстрее Treiber** при 8 ядрах.

---

## 16.14 Вопросы для самопроверки

1. Что такое contention?
2. Как обнаружить contention?
3. Что такое sharding?
4. Как выбрать N шардов?
5. Почему `& mask` быстрее `% N`?
6. Какие хэш-функции использовать?
7. Когда sharding оправдан?
8. Когда sharding не нужен?
9. Что такое lock-free?
10. Что такое CAS?
11. Как работает Treiber stack?
12. Как работает Michael-Scott queue?
13. Что такое ABA problem?
14. Как решить ABA problem?
15. Что такое hazard pointers?
16. Когда lock-free хуже Mutex?
17. Как бенчмаркать конкурентные структуры?
18. Почему `atomic` быстрее `Mutex`?

---

## 16.15 Ответы

### Ответ 1

**Contention** — конкуренция за один ресурс. Горутины ждут друг друга.

### Ответ 2

**Mutex profile:**

```go
runtime.SetMutexProfileFraction(1)
```

```bash
go tool pprof mutex.prof
```

### Ответ 3

**Sharding** — разбиение структуры на N частей, каждая со своим мьютексом.

### Ответ 4

**N = 2^k**, близко к числу ядер. Обычно 2 × число ядер.

### Ответ 5

**`& mask` быстрее `% N`**, потому что деление медленнее битовой операции.

### Ответ 6

**Хэш:** `fnv`, `maphash`, `xxhash`.

### Ответ 7

**Sharding оправдан:** высокий contention, write-heavy, большая map.

### Ответ 8

**Не оправдан:** низкий contention, read-only, малая map.

### Ответ 9

**Lock-free** — структуры без мьютексов, через CAS.

### Ответ 10

**CAS** — Compare-And-Swap. Атомарная «замена, если равно».

### Ответ 11

**Treiber stack** — lock-free стек. `atomic.Pointer` на head. CAS retry loop.

### Ответ 12

**Michael-Scott queue** — lock-free очередь. Два указателя: head и tail. Dummy node.

### Ответ 13

**ABA problem** — CAS не видит, что значение изменилось A→B→A.

### Ответ 14

**Решения:** tagged pointers, hazard pointers, не переиспользовать.

### Ответ 15

**Hazard pointers** — механизм для memory reclamation. Каждая горутина объявляет указатели.

### Ответ 16

**Lock-free хуже** при высокой конкуренции, сложных структурах, ABA.

### Ответ 17

**`b.RunParallel`** + **`-cpu=1,2,4,8`** + **`-benchmem`**.

### Ответ 18

**`atomic` быстрее `Mutex`**, потому что нет блокировки и нет contention (для одного счётчика).

---

## 16.16 Куда идти дальше?

Мы разобрали sharded locks и lock-free структуры. Теперь мы умеем уменьшать contention.

Но остаётся **следующая тема**: как переиспользовать объекты и как устроен аллокатор памяти?

- **Как переиспользовать объекты?** `sync.Pool`, escape analysis, аллокатор. → **Глава 17: sync.Pool и аллокатор памяти.**
- **Как GC влияет на конкурентный код?** Tri-color, write barrier, STW паузы. → **Глава 18: GC и его влияние на конкурентный код.**
- **Как строить отказоустойчивые системы?** Bulkhead, retry, leader election. → **Глава 19: Продвинутые паттерны.**

---

## 16.17 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Contention** | Конкуренция за ресурс | O(N × T_hold) |
| **Sharding** | Разбиение на N частей | N = 2^k, `& mask` |
| **Хэш** | Определяет шард | `fnv`, `maphash` |
| **Sharded map** | N шардов | Generic, RWMutex |
| **Когда оправдан** | Высокий contention | Write-heavy |
| **Lock-free** | Без мьютексов | CAS |
| **CAS** | Compare-And-Swap | Атомарная замена |
| **Retry loop** | Паттерн lock-free | Load → compute → CAS |
| **Treiber stack** | Lock-free стек | `atomic.Pointer` head |
| **Michael-Scott** | Lock-free очередь | head + tail + dummy |
| **ABA problem** | CAS не видит изменения | A→B→A |
| **Hazard pointers** | Memory reclamation | Per-goroutine |
| **Lock-free хуже** | При конкуренции | Mutex быстрее |
| **Бенчмарки** | `b.RunParallel` | `-cpu=1,2,4,8` |
| **`atomic`** | Быстрее `Mutex` | ~10 нс vs ~25 нс |

🧩 **Ключевая идея:** Contention на одном мьютексе — bottleneck. **Sharding** — разбиение на N частей, каждая со своим мьютексом. N = 2^k, близко к числу ядер. `& mask` вместо `% N`. Sharded map — Generic, RWMutex в каждом шарде. Когда sharding оправдан: высокий contention, write-heavy, большая map. **Lock-free** — без мьютексов, через CAS. Treiber stack — lock-free стек. Michael-Scott queue — lock-free очередь. **ABA problem** — CAS не видит изменения A→B→A. Решения: tagged pointers, hazard pointers, не переиспользовать. **Hazard pointers** — для memory reclamation. **Lock-free хуже Mutex** при высокой конкуренции и сложных структурах. **Бенчмаркай** с `-cpu=1,2,4,8`. `atomic` быстрее `Mutex` для счётчиков. Mutex stack быстрее Treiber при 8 ядрах.