# 🔐 Глава 29: Sharded locks — разбиение блокировок

**Что вы узнаете:**
- Что такое sharded locks и какую задачу они решают.
- Почему один мьютекс становится bottleneck.
- Как разбить один мьютекс на N шардов.
- Как выбрать число шардов.
- Как избежать проблем с hash-функцией.
- Как комбинировать sharded locks с RWMutex, atomic, sync.Map.
- Как измерять эффект от шардирования.
- Где sharded locks не помогают.

**После прочтения вы сможете:**
- Построить sharded map с нуля.
- Разбить любой мьютекс на шарды.
- Выбрать число шардов под нагрузку.
- Измерять contention до и после шардирования.
- Понимать, где sharded locks уместны, а где — нет.

---

## Содержание

- [29.0 Пролог: один мьютекс на всё](#290-пролог-один-мьютекс-на-всё)
- [29.1 Что такое sharded locks](#291-что-такое-sharded-locks)
- [29.2 Sharded map с нуля](#292-sharded-map-с-нуля)
- [29.3 Выбор числа шардов](#293-выбор-числа-шардов)
- [29.4 Hash-функция: как не сломать шардирование](#294-hash-функция-как-не-сломать-шардирование)
- [29.5 Шардированный RWMutex](#295-шардированный-rwmutex)
- [29.6 Sharded locks vs sync.Map](#296-sharded-locks-vs-syncmap)
- [29.7 Измерение эффекта](#297-измерение-эффекта)
- [29.8 В связке с другими паттернами](#298-в-связке-с-другими-паттернами)
- [29.9 Практика Go: sharded map с метриками](#299-практика-go-sharded-map-с-метриками)
- [29.10 Выводы и типичные ошибки](#2910-выводы-и-типичные-ошибки)
- [29.11 Для быстрого повторения](#2911-для-быстрого-повторения)
- [29.12 Вопросы для самопроверки](#2912-вопросы-для-самопроверки)
- [29.13 Ответы](#2913-ответы)
- [29.14 Куда идти дальше?](#2914-куда-идти-дальше)
- [29.15 Чек-лист](#2915-чек-лист)

---

## 29.0 Пролог: один мьютекс на всё

У нас есть сервис с **кэшем пользователей**. 1000 горутин постоянно читают и пишут:

```go
type Cache struct {
    mu      sync.Mutex
    users   map[int]User
    orders  map[int]Order
    metrics map[string]int64
}

func (c *Cache) GetUser(id int) User {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.users[id]
}

func (c *Cache) SetUser(u User) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.users[u.ID] = u
}
```

Работает. Но в CPU-профиле видно странное:

```
flat  flat%   sum%        cum   cum%
 8.5s 45.00% 45.00%      8.5s 45.00%  sync.(*Mutex).Lock
 2.5s 13.00% 58.00%      2.5s 13.00%  sync.(*Mutex).Unlock
```

**45% CPU** уходит на `Mutex.Lock` — contention. Все 1000 горутин **конкурируют** за один мьютекс. Даже если они работают с **разными** пользователями — они **ждут** друг друга.

Хочется: разбить **один** мьютекс на **несколько**. Тогда горутины, работающие с **разными** пользователями, **не мешают** друг другу.

Это и есть **sharded locks** — разбиение блокировки на шарды.

> **Мост к следующим главам:** sharded locks — важный паттерн для high-concurrency кэшей. Он часто используется вместе с `sync.Map` (Глава 31), `atomic` (Глава 4) и lock-free структурами (Глава 30). Понимание sharded locks даёт понимание, **как масштабировать кэши**.

---

## 29.1 Что такое sharded locks

**Sharded locks** — паттерн, при котором **один мьютекс** заменяется на **N мьютексов** (шардов). Каждый шард защищает **свою часть** данных.

### Идея

Один мьютекс на всё → **contention** → **медленно**.

N мьютексов → меньше contention → **быстрее**.

**Ключевое:** если горутина A работает с `users[1]`, а горутина B — с `users[2]`, они **не мешают** друг другу, если `1` и `2` попали в **разные шарды**.

### Схема

```
Один мьютекс:

  G1 ──┐
  G2 ──┤──► [Mutex] ──► Map
  G3 ──┤       │
  G4 ──┘       │
            все ждут
            по очереди

Sharded locks (4 шарда):

  G1 ──► [Mutex 0] ──► Map[0]
  G2 ──► [Mutex 1] ──► Map[1]
  G3 ──► [Mutex 2] ──► Map[2]
  G4 ──► [Mutex 3] ──► Map[3]
  
  Каждая горутина работает со своим шардом.
  Contention ↓ в 4 раза.
```

### Когда использовать sharded locks

**1. Высокий contention на одном мьютексе.**

- CPU-профиль показывает `Mutex.Lock` в top.
- Много горутин конкурируют за один ресурс.

**2. Кэши, реестры, map.**

- Ключи **независимы**.
- Операции **короткие**.

**3. Read-heavy нагрузка.**

- Много чтений, мало записей.
- `RWMutex` может помочь, но sharding — сильнее.

### Когда НЕ использовать

**1. Мало горутин.**

Если горутин 2–5 — contention незначителен. Overhead sharding больше выгоды.

**2. Операции над несколькими ключами.**

Если транзакция работает с несколькими ключами — шардирование может дать **deadlock** (если ключи в разных шардах).

**3. Маленькая map.**

Если элементов 100 — overhead sharding больше.

### Аналогия: кассы в супермаркете

**Один мьютекс** — как **одна касса** на весь супермаркет. Все покупатели стоят в одной очереди.

**Sharded locks** — как **10 касс**. Каждая обслуживает своих покупателей. Очереди короче.

### 💡 Практика: как думать о sharded locks

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Sharded locks — при высоком contention.**
2. **N шардов = 2× GOMAXPROCS или больше.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Измерь contention** до sharding.
4. **Используй `atomic` для метрик.**

**❌ НЕ ДЕЛАЙ:**

5. **Не используй sharded locks для 5 горутин.**
6. **Не работай с несколькими шардами в одной транзакции.**

---

## 29.2 Sharded map с нуля

Начнём с **классики** — sharded map.

### Идея

- N шардов.
- Каждый шард — свой `Mutex` + своя `map`.
- Ключ → шард через hash-функцию.

### Реализация

```go
const numShards = 32

type shard struct {
    mu sync.RWMutex
    m  map[string]interface{}
}

type ShardedMap struct {
    shards [numShards]*shard
}

func NewShardedMap() *ShardedMap {
    sm := &ShardedMap{}
    for i := 0; i < numShards; i++ {
        sm.shards[i] = &shard{
            m: make(map[string]interface{}),
        }
    }
    return sm
}

func (sm *ShardedMap) getShard(key string) *shard {
    h := fnv.New32a()
    h.Write([]byte(key))
    return sm.shards[h.Sum32()%numShards]
}

func (sm *ShardedMap) Get(key string) (interface{}, bool) {
    s := sm.getShard(key)
    s.mu.RLock()
    defer s.mu.RUnlock()
    v, ok := s.m[key]
    return v, ok
}

func (sm *ShardedMap) Set(key string, value interface{}) {
    s := sm.getShard(key)
    s.mu.Lock()
    defer s.mu.Unlock()
    s.m[key] = value
}

func (sm *ShardedMap) Delete(key string) {
    s := sm.getShard(key)
    s.mu.Lock()
    defer s.mu.Unlock()
    delete(s.m, key)
}
```

**Что происходит:**

- `getShard` — hash ключа → индекс шарда.
- Каждый шард — свой `RWMutex` + своя map.
- Операции с **разными** ключами → **разные** шарды → **нет contention**.

### Схема

```
ShardedMap:
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ shard[0]     │  │ shard[1]     │  │ shard[2]     │
  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
  │ │RWMutex   │ │  │ │RWMutex   │ │  │ │RWMutex   │ │
  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
  │ │map[...]  │ │  │ │map[...]  │ │  │ │map[...]  │ │
  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
  └──────────────┘  └──────────────┘  └──────────────┘
  
  Key "user-42" → hash → shard[1] → RWMutex[1] → map[1]
  Key "user-43" → hash → shard[2] → RWMutex[2] → map[2]
  
  Разные ключи → разные шарды → нет contention.
```

### Потребитель

```go
func main() {
    sm := NewShardedMap()
    
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        i := i
        wg.Add(1)
        go func() {
            defer wg.Done()
            key := fmt.Sprintf("key-%d", i)
            sm.Set(key, i)
            v, _ := sm.Get(key)
            _ = v
        }()
    }
    wg.Wait()
}
```

### Стоимость

| Операция | Time |
|:---|:---|
| `getShard` (hash) | ~50–100 нс |
| `RLock` | ~20–30 нс |
| `Lock` | ~25–40 нс |
| Get/Set | ~100–200 нс |

### 💡 Практика: как писать sharded map

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **32–256 шардов.**
2. **`RWMutex` в каждом шарде** — для read-heavy.
3. **Hash-функция — FNV или xxhash.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики по шардам** — для диагностики.

**❌ НЕ ДЕЛАЙ:**

5. **Не делай 1 шард** — это обычный мьютекс.
6. **Не делай 10 000 шардов** — overhead.

---

## 29.3 Выбор числа шардов

Число шардов — **ключевой** параметр.

### Формула

**Минимум:** `2 × GOMAXPROCS`.

**Оптимум:** `4 × GOMAXPROCS` или больше.

**Максимум:** `256–1024`.

### Почему не 1

**1 шард** = обычный мьютекс. Contention **тот же**.

### Почему не 10 000

**10 000 шардов:**

- Память: 10 000 × (RWMutex + map) ≈ **несколько МБ**.
- Overhead: **больше cache miss** при обходе.
- **Нет выгоды** — contention уже низкий.

### Как выбрать

**Шаг 1: измерь contention.**

```go
runtime.SetMutexProfileFraction(1)
```

**Шаг 2: выбери N.**

- CPU 8 ядер → N = 32.
- CPU 16 ядер → N = 64.
- CPU 32 ядра → N = 128.

**Шаг 3: измерь после sharding.**

**Шаг 4: подстрой N.**

### Пример

```go
func BenchmarkShards(b *testing.B) {
    for _, n := range []int{1, 4, 16, 64, 256} {
        b.Run(fmt.Sprintf("shards=%d", n), func(b *testing.B) {
            sm := NewShardedMapN(n)
            b.RunParallel(func(pb *testing.PB) {
                i := 0
                for pb.Next() {
                    key := fmt.Sprintf("key-%d", i%1000)
                    sm.Set(key, i)
                    i++
                }
            })
        })
    }
}
```

**Пример вывода:**

```
shards=1:    5000000    250 ns/op
shards=4:   15000000     85 ns/op
shards=16:  25000000     50 ns/op
shards=64:  30000000     42 ns/op
shards=256: 30000000     40 ns/op
```

**Что видно:**

- 1 → 4 шарда: **3x быстрее**.
- 4 → 16: **1.7x**.
- 16 → 256: **почти нет выгоды**.

**Оптимум:** 32–64 шарда для 8-ядерного CPU.

### Схема

```
Contention vs Число шардов:

  Contention
      │
   100%│●
      │  ●
    50%│    ●
      │      ●●
    25%│        ●●●
      │           ●●●●●●●●●
     0%│                    ●●●●●●●●●●●
      └───────────────────────────────► Шарды
       1  4  8  16  32  64 128 256 512
       
  Оптимум: 32–64.
```

### 💡 Практика: как выбирать число шардов

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Минимум 2× GOMAXPROCS.**
2. **Оптимум 4× GOMAXPROCS.**
3. **Максимум 256–1024.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Бенчмаркай разные N.**
5. **Мониторь contention.**

**❌ НЕ ДЕЛАЙ:**

6. **Не делай 1 шард.**
7. **Не делай 10 000 шардов.**

---

## 29.4 Hash-функция: как не сломать шардирование

Hash-функция определяет, **как ключи распределяются** по шардам. Плохая hash → **все ключи в одном шарде** → contention.

### Требования к hash

**1. Равномерность.**

Каждый шард получает **примерно одинаковое** число ключей.

**2. Быстрота.**

Hash вызывается на **каждой** операции. Медленный hash → slowdown.

**3. Детерминизм.**

Один и тот же ключ → **один и тот же** шард.

### Плохой пример: `len(key) % numShards`

```go
func getShard(key string) int {
    return len(key) % numShards
}
```

**Проблема:** все ключи **одинаковой длины** попадут в **один шард**.

**Пример:**

- `"user-0001"`, `"user-0002"`, `"user-0003"` — все длины 9 → **один шард**.

### Хороший пример: FNV-1a

```go
import "hash/fnv"

func getShard(key string) int {
    h := fnv.New32a()
    h.Write([]byte(key))
    return int(h.Sum32()) % numShards
}
```

**Что даёт:**

- **Равномерное** распределение.
- **Быстро** (~30–50 нс).
- **Детерминированно**.

### Альтернатива: xxhash

**xxhash** — быстрее FNV, но требует внешней библиотеки.

```go
import "github.com/cespare/xxhash/v2"

func getShard(key string) int {
    return int(xxhash.Sum64String(key)) % numShards
}
```

**Быстрее** FNV в 2–3 раза. **Рекомендуется** для highload.

### Альтернатива: maphash

**`hash/maphash`** (Go 1.14+) — встроенная быстрая hash:

```go
import "hash/maphash"

var seed = maphash.MakeSeed()

func getShard(key string) int {
    return int(maphash.String(seed, key)) % numShards
}
```

**Быстрее** FNV. **Встроена** в Go.

### Пример: сравнение hash

```go
func BenchmarkHash(b *testing.B) {
    key := "user-12345"
    
    b.Run("fnv", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            h := fnv.New32a()
            h.Write([]byte(key))
            _ = h.Sum32()
        }
    })
    
    b.Run("maphash", func(b *testing.B) {
        seed := maphash.MakeSeed()
        for i := 0; i < b.N; i++ {
            _ = maphash.String(seed, key)
        }
    })
}
```

**Пример вывода:**

```
fnv:       50000000    30 ns/op
maphash:  100000000    15 ns/op
```

**Что видно:** `maphash` в 2 раза быстрее.

### Плохой пример: hash по указателю

```go
func getShard(key *string) int {
    ptr := uintptr(unsafe.Pointer(key))
    return int(ptr) % numShards
}
```

**Проблема:** разные указатели на **одну строку** → **разные шарды**. Нарушает детерминизм.

### 💡 Практика: как выбрать hash

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`hash/maphash`** — встроено, быстро.
2. **FNV-1a** — просто, стандартно.
3. **xxhash** — если нужна максимальная скорость.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Бенчмаркай hash** — если hot path.

**❌ НЕ ДЕЛАЙ:**

5. **Не используй `len(key) % N`** — плохо распределяет.
6. **Не используй hash по указателю** — не детерминирован.

---

## 29.5 Шардированный RWMutex

Иногда данные **связаны** и требуют **RWMutex**. Разберём шардированный RWMutex.

### Проблема: связанные данные

```go
type Cache struct {
    mu      sync.RWMutex
    users   map[int]User
    orders  map[int]Order
}
```

**Что делать, если `users` и `orders` **связаны** логически?**

**Вариант 1:** один RWMutex — contention.

**Вариант 2:** разные RWMutex — нет **атомарности** между `users` и `orders`.

### Решение: шардировать по **ключу**

```go
type shard struct {
    mu     sync.RWMutex
    users  map[int]User
    orders map[int]Order
}
```

**Что даёт:** для одного пользователя `users[id]` и `orders[id]` — в **одном** шарде. Атомарность сохраняется.

### Пример

```go
const numShards = 32

type UserShard struct {
    mu     sync.RWMutex
    users  map[int]User
    orders map[int]Order
}

type UserCache struct {
    shards [numShards]*UserShard
}

func NewUserCache() *UserCache {
    c := &UserCache{}
    for i := 0; i < numShards; i++ {
        c.shards[i] = &UserShard{
            users:  make(map[int]User),
            orders: make(map[int]Order),
        }
    }
    return c
}

func (c *UserCache) getShard(id int) *UserShard {
    return c.shards[id%numShards]
}

func (c *UserCache) GetUser(id int) (User, bool) {
    s := c.getShard(id)
    s.mu.RLock()
    defer s.mu.RUnlock()
    u, ok := s.users[id]
    return u, ok
}

func (c *UserCache) SetUser(u User) {
    s := c.getShard(u.ID)
    s.mu.Lock()
    defer s.mu.Unlock()
    s.users[u.ID] = u
}

func (c *UserCache) GetOrders(id int) []Order {
    s := c.getShard(id)
    s.mu.RLock()
    defer s.mu.RUnlock()
    // ...
    return nil
}
```

### Схема

```
UserCache (32 шарда):
  shard[0]  → users[0], orders[0], users[32], orders[32], ...
  shard[1]  → users[1], orders[1], users[33], orders[33], ...
  ...
  
  Для id=42: shard[10] (42%32=10) → users[42], orders[42] в одном шарде.
  Атомарность сохраняется.
```

### Проблема: операции над несколькими ключами

**Что если нужно перевести заказы от пользователя 42 к пользователю 43?**

- `42 % 32 = 10` → shard[10]
- `43 % 32 = 11` → shard[11]

**Два шарда** — нужен **lock на оба**. Это **deadlock**, если порядок разный.

**Решение:** всегда **сортируй** шарды перед блокировкой:

```go
func transfer(from, to int) {
    s1 := c.getShard(from)
    s2 := c.getShard(to)
    
    // Сортируем по адресу указателя
    if s1 > s2 {
        s1, s2 = s2, s1
    }
    
    s1.mu.Lock()
    s2.mu.Lock()
    defer s1.mu.Unlock()
    defer s2.mu.Unlock()
    
    // ... transfer ...
}
```

**Что даёт:** все горутины берут локи в **одном порядке** → нет deadlock.

### 💡 Практика: как шардировать RWMutex

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Шардируй по ключу** — чтобы связанные данные были в одном шарде.
2. **Сортируй шарды** перед multi-lock.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Избегай операций над несколькими ключами**, если можно.

**❌ НЕ ДЕЛАЙ:**

5. **Не бери шарды в разном порядке.**
6. **Не шардируй связанные данные по разным ключам.**

---

## 29.6 Sharded locks vs sync.Map

Разберём **разницу** между sharded locks и `sync.Map`.

### sync.Map

**`sync.Map`** (Глава 3, 31) — встроенная конкурентная map с lock-free чтением.

**Плюсы:**

- **Встроена** в Go.
- **Lock-free чтение**.
- **Простой API**.

**Минусы:**

- **Медленная запись**.
- **Не всегда быстрее** sharded map.

### Sharded locks

**Плюсы:**

- **Быстрее** для write-heavy.
- **Гибкость** — можно шардировать что угодно.

**Минусы:**

- **Больше кода**.
- **Нужно выбирать N шардов**.
- **Сложнее** с multi-key операциями.

### Сравнение

| Аспект | sync.Map | Sharded locks |
|:---|:---|:---|
| Чтение | Lock-free | RWMutex.RLock |
| Запись | Mutex | shard.Mutex |
| API | Простой | Свой |
| Гибкость | Только map | Любые данные |
| Код | Минимум | Больше |

### Когда что

**`sync.Map`:**

- **Read-heavy**.
- **Ключи стабильны**.
- **Простой случай**.

**Sharded locks:**

- **Write-heavy**.
- **Связанные данные**.
- **Нужна гибкость**.

### Бенчмарк

```go
func BenchmarkSyncMap(b *testing.B) {
    var m sync.Map
    b.RunParallel(func(pb *testing.PB) {
        i := 0
        for pb.Next() {
            key := fmt.Sprintf("key-%d", i%1000)
            m.Store(key, i)
            i++
        }
    })
}

func BenchmarkShardedMap(b *testing.B) {
    sm := NewShardedMap()
    b.RunParallel(func(pb *testing.PB) {
        i := 0
        for pb.Next() {
            key := fmt.Sprintf("key-%d", i%1000)
            sm.Set(key, i)
            i++
        }
    })
}
```

**Пример вывода (write-heavy):**

```
SyncMap:     5000000    250 ns/op
ShardedMap: 15000000     85 ns/op
```

**Что видно:** sharded map в 3 раза быстрее для write-heavy.

### 💡 Практика: как выбирать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **sync.Map** — для read-heavy, простых случаев.
2. **Sharded locks** — для write-heavy, сложных случаев.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Бенчмаркай** перед выбором.

**❌ НЕ ДЕЛАЙ:**

4. **Не используй sharded map для read-only.**

---

## 29.7 Измерение эффекта

Sharded locks **нужно** измерять. Иначе не видно эффекта.

### Метрики

**1. Mutex profile.**

**До sharding:**

```
flat  flat%   sum%        cum   cum%
 8.5s 45.00% 45.00%      8.5s 45.00%  sync.(*Mutex).Lock
```

**После sharding:**

```
flat  flat%   sum%        cum   cum%
 0.5s  5.00%  5.00%      0.5s  5.00%  sync.(*Mutex).Lock
```

**Что видно:** contention упал с 45% до 5%.

**2. CPU profile.**

**Что искать:** `Mutex.Lock` в top.

**3. Бенчмарк.**

**До/после:**

```
Before: 250 ns/op
After:   85 ns/op
```

### Пример

```go
func BenchmarkContention(b *testing.B) {
    for _, n := range []int{1, 4, 16, 64} {
        b.Run(fmt.Sprintf("shards=%d", n), func(b *testing.B) {
            sm := NewShardedMapN(n)
            b.RunParallel(func(pb *testing.PB) {
                i := 0
                for pb.Next() {
                    key := fmt.Sprintf("key-%d", i%1000)
                    sm.Set(key, i)
                    i++
                }
            })
        })
    }
}
```

**Пример вывода:**

```
shards=1:    5000000    250 ns/op
shards=4:   15000000     85 ns/op
shards=16:  25000000     50 ns/op
shards=64:  30000000     42 ns/op
```

### Mutex profile

**Включение:**

```go
runtime.SetMutexProfileFraction(1)
```

**Анализ:**

```bash
go test -mutexprofile=mutex.prof -bench=. ./...
go tool pprof mutex.prof
```

### 💡 Практика: как измерять sharding

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Mutex profile** — до и после.
2. **Бенчмарк** с разными N.
3. **CPU profile** — contention в top.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики в production.**

**❌ НЕ ДЕЛАЙ:**

5. **Не используй sharding без измерений.**

---

## 29.8 В связке с другими паттернами

Sharded locks редко используются **в одиночку**. Разберём связки.

### Sharded locks + atomic

**Счётчики по шардам:**

```go
type ShardedCounter struct {
    shards [numShards]atomic.Int64
}

func (c *ShardedCounter) Add(n int) {
    shard := n % numShards
    c.shards[shard].Add(1)
}

func (c *ShardedCounter) Total() int64 {
    var total int64
    for i := range c.shards {
        total += c.shards[i].Load()
    }
    return total
}
```

**Что даёт:** нет contention на счётчике.

### Sharded locks + sync.Pool

**Пул по шардам:**

```go
type ShardedPool struct {
    pools [numShards]sync.Pool
}

func (p *ShardedPool) Get(id int) interface{} {
    shard := id % numShards
    return p.pools[shard].Get()
}
```

**Что даёт:** меньше contention на `sync.Pool`.

### Sharded locks + worker pool

**Worker pool с шардированными очередями:**

```go
type ShardedWorkerPool struct {
    tasksChs [numShards]chan Task
}

func (p *ShardedWorkerPool) Submit(task Task) {
    shard := task.ID % numShards
    p.tasksChs[shard] <- task
}
```

**Что даёт:** меньше contention на канале задач.

### Полная схема

```
Запрос
   │
   ▼
hash(key) → shard[i]
   │
   ▼
┌──────────────────────┐
│ shard[i]             │
│  ┌────────────────┐  │
│  │ RWMutex        │  │
│  └────────────────┘  │
│  ┌────────────────┐  │
│  │ map            │  │
│  └────────────────┘  │
└──────────────────────┘
```

### 💡 Практика: как комбинировать sharded locks

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Sharded locks + atomic** — счётчики.
2. **Sharded locks + sync.Pool** — пулы.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Sharded locks + worker pool** — очереди.

**❌ НЕ ДЕЛАЙ:**

4. **Не забывай про hash.**

---

## 29.9 Практика Go: sharded map с метриками

Разберём **sharded map с метриками**.

### Полный код

```go
package main

import (
    "fmt"
    "hash/maphash"
    "sync"
    "sync/atomic"
    "time"
)

const numShards = 32

type Metrics struct {
    Gets   atomic.Int64
    Sets   atomic.Int64
    Deletes atomic.Int64
    Contention atomic.Int64
}

type shard struct {
    mu sync.RWMutex
    m  map[string]interface{}
    gets atomic.Int64
    sets atomic.Int64
}

type ShardedMap struct {
    shards [numShards]*shard
    seed   maphash.Seed
    metrics *Metrics
}

func NewShardedMap() *ShardedMap {
    sm := &ShardedMap{
        seed:    maphash.MakeSeed(),
        metrics: &Metrics{},
    }
    for i := 0; i < numShards; i++ {
        sm.shards[i] = &shard{
            m: make(map[string]interface{}),
        }
    }
    return sm
}

func (sm *ShardedMap) getShard(key string) *shard {
    h := maphash.String(sm.seed, key)
    return sm.shards[h%numShards]
}

func (sm *ShardedMap) Get(key string) (interface{}, bool) {
    s := sm.getShard(key)
    s.gets.Add(1)
    sm.metrics.Gets.Add(1)
    
    s.mu.RLock()
    defer s.mu.RUnlock()
    v, ok := s.m[key]
    return v, ok
}

func (sm *ShardedMap) Set(key string, value interface{}) {
    s := sm.getShard(key)
    s.sets.Add(1)
    sm.metrics.Sets.Add(1)
    
    s.mu.Lock()
    defer s.mu.Unlock()
    s.m[key] = value
}

func (sm *ShardedMap) Delete(key string) {
    s := sm.getShard(key)
    sm.metrics.Deletes.Add(1)
    
    s.mu.Lock()
    defer s.mu.Unlock()
    delete(s.m, key)
}

func (sm *ShardedMap) Metrics() *Metrics {
    return sm.metrics
}

func (sm *ShardedMap) ShardStats() []string {
    stats := make([]string, numShards)
    for i, s := range sm.shards {
        stats[i] = fmt.Sprintf("shard[%d]: gets=%d sets=%d size=%d",
            i, s.gets.Load(), s.sets.Load(), len(s.m))
    }
    return stats
}

func main() {
    sm := NewShardedMap()
    
    var wg sync.WaitGroup
    start := time.Now()
    
    for i := 0; i < 100; i++ {
        i := i
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < 10000; j++ {
                key := fmt.Sprintf("key-%d", (i*10000+j)%1000)
                if j%2 == 0 {
                    sm.Set(key, j)
                } else {
                    sm.Get(key)
                }
            }
        }()
    }
    wg.Wait()
    elapsed := time.Since(start)
    
    fmt.Printf("Elapsed: %v\n", elapsed)
    fmt.Printf("Ops/sec: %.0f\n", float64(sm.Metrics().Gets.Load()+sm.Metrics().Sets.Load())/elapsed.Seconds())
    fmt.Printf("Gets:    %d\n", sm.Metrics().Gets.Load())
    fmt.Printf("Sets:    %d\n", sm.Metrics().Sets.Load())
    
    // Распределение по шардам
    fmt.Println("\nShard distribution:")
    for _, s := range sm.ShardStats()[:5] {
        fmt.Println(s)
    }
}
```

**Пример вывода:**

```
Elapsed: 45ms
Ops/sec: 22222222
Gets:    500000
Sets:    500000

Shard distribution:
shard[0]: gets=15630 sets=15625 size=32
shard[1]: gets=15625 sets=15625 size=31
shard[2]: gets=15625 sets=15625 size=32
shard[3]: gets=15625 sets=15625 size=31
shard[4]: gets=15625 sets=15625 size=32
```

**Что видно:**

- **22M ops/sec** — высокая пропускная способность.
- **Равномерное распределение** по шардам (~15 625 ops на шард).

### Сравнение: 1 шард vs 32 шарда

```go
func Benchmark1Shard(b *testing.B) {
    sm := NewShardedMapN(1)
    b.RunParallel(func(pb *testing.PB) {
        i := 0
        for pb.Next() {
            key := fmt.Sprintf("key-%d", i%1000)
            sm.Set(key, i)
            i++
        }
    })
}

func Benchmark32Shards(b *testing.B) {
    sm := NewShardedMapN(32)
    b.RunParallel(func(pb *testing.PB) {
        i := 0
        for pb.Next() {
            key := fmt.Sprintf("key-%d", i%1000)
            sm.Set(key, i)
            i++
        }
    })
}
```

**Пример вывода:**

```
1Shard:    5000000    250 ns/op
32Shards: 30000000     42 ns/op
```

**Что видно:** 6x быстрее с 32 шардами.

### 💡 Практика: как измерять sharded map

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Метрики:** gets, sets, deletes.
2. **Distribution** по шардам.
3. **Ops/sec.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Бенчмарк** с разными N.
5. **Mutex profile** — contention.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `Mutex` для метрик.**
7. **Не игнорируй distribution.**

---

## 29.10 Выводы и типичные ошибки

**Что мы узнали?**

Sharded locks — разбиение одного мьютекса на N шардов. **Простейший** — sharded map. **Число шардов:** 2–4× GOMAXPROCS. **Hash:** FNV, maphash, xxhash. **Sharded RWMutex** — для связанных данных. **Sharded locks vs sync.Map:** write-heavy vs read-heavy. **Измерение:** mutex profile, бенчмарк. Комбинируется с atomic, sync.Pool, worker pool.

**Типичные ошибки:**

- ❌ **1 шард.** Это обычный мьютекс.
- ❌ **10 000 шардов.** Overhead.
- ❌ **Плохой hash.** Все ключи в одном шарде.
- ❌ **`len(key) % N`.** Плохое распределение.
- ❌ **Hash по указателю.** Не детерминирован.
- ❌ **Multi-key операции без сортировки шардов.** Deadlock.
- ❌ **Sharding без измерений.**
- ❌ **Sharded map для read-only.**
- ❌ **Sharding для 5 горутин.**
- ❌ **Не мониторить distribution.**

---

## 29.11 Для быстрого повторения

- **Sharded locks** — N мьютексов вместо одного.
- **Sharded map** — классический пример.
- **Число шардов:** 2–4× GOMAXPROCS, максимум 256–1024.
- **Hash:** FNV, maphash, xxhash.
- **`maphash`** — встроен, быстрее FNV.
- **Sharded RWMutex** — для связанных данных.
- **Multi-key:** сортируй шарды перед lock.
- **Sharded vs sync.Map:** write-heavy vs read-heavy.
- **Метрики:** mutex profile, бенчмарк.
- **Distribution** по шардам — важно.
- **6x быстрее** с 32 шардами (пример).
- **+ atomic** — счётчики.
- **+ sync.Pool** — пулы.

---

## 29.12 Вопросы для самопроверки

1. Что такое sharded locks? Какую задачу решают?
2. Как построить sharded map?
3. Сколько шардов выбрать?
4. Почему плохой hash ломает шардирование?
5. Как шардировать RWMutex?
6. Чем sharded locks отличаются от sync.Map?
7. Как измерять эффект от sharding?
8. Что делать с multi-key операциями?

---

## 29.13 Ответы

### Ответ 1

**Sharded locks** — паттерн разбиения одного мьютекса на N шардов. Решает задачу **contention**: горутины, работающие с **разными** ключами, попадают в **разные** шарды и **не мешают** друг другу.

**Пример:** cache пользователей с 1000 горутин — один мьютекс → 45% CPU в `Mutex.Lock`. 32 шарда → 5% CPU.

### Ответ 2

```go
const numShards = 32

type shard struct {
    mu sync.RWMutex
    m  map[string]interface{}
}

type ShardedMap struct {
    shards [numShards]*shard
}

func (sm *ShardedMap) getShard(key string) *shard {
    h := fnv.New32a()
    h.Write([]byte(key))
    return sm.shards[h.Sum32()%numShards]
}

func (sm *ShardedMap) Get(key string) (interface{}, bool) {
    s := sm.getShard(key)
    s.mu.RLock()
    defer s.mu.RUnlock()
    v, ok := s.m[key]
    return v, ok
}
```

### Ответ 3

**Число шардов:**

- **Минимум:** 2× GOMAXPROCS.
- **Оптимум:** 4× GOMAXPROCS.
- **Максимум:** 256–1024.

**Пример:** 8-ядерный CPU → 32 шарда.

**Бенчмаркай** разные N — оптимум зависит от нагрузки.

### Ответ 4

**Плохой hash** → все ключи в **одном** шарде → contention **тот же**, что с одним мьютексом.

**Пример плохого hash:** `len(key) % numShards` — все ключи **одинаковой длины** в одном шарде.

**Хороший hash:** FNV, maphash, xxhash — равномерное распределение.

### Ответ 5

**Шардирование RWMutex:**

```go
type shard struct {
    mu     sync.RWMutex
    users  map[int]User
    orders map[int]Order
}

func (c *Cache) getShard(id int) *shard {
    return c.shards[id%numShards]
}

// users[id] и orders[id] — в одном шарде. Атомарность сохраняется.
```

**Для multi-key операций:** сортируй шарды перед lock.

```go
s1 := c.getShard(from)
s2 := c.getShard(to)
if s1 > s2 {
    s1, s2 = s2, s1
}
s1.mu.Lock()
s2.mu.Lock()
```

### Ответ 6

**sync.Map:**

- Встроена в Go.
- **Lock-free чтение.**
- **Медленная запись.**
- Простой API.

**Sharded locks:**

- **Быстрее** для write-heavy.
- **Гибкость** — любые данные.
- Больше кода.

**sync.Map** для read-heavy. **Sharded locks** для write-heavy.

### Ответ 7

**Измерение:**

1. **Mutex profile** — `SetMutexProfileFraction(1)`.
2. **Бенчмарк** с разными N.
3. **CPU profile** — contention в top.

**До/после:**

```
Before: 45% CPU in Mutex.Lock
After:   5% CPU in Mutex.Lock

Before: 250 ns/op
After:   42 ns/op
```

### Ответ 8

**Multi-key операции:**

- **Избегай**, если можно.
- Если нельзя — **сортируй шарды** перед lock.

```go
s1 := getShard(key1)
s2 := getShard(key2)
if s1 > s2 {
    s1, s2 = s2, s1
}
s1.mu.Lock()
s2.mu.Lock()
defer s1.mu.Unlock()
defer s2.mu.Unlock()
```

**Что даёт:** все горутины берут локи в **одном порядке** → нет deadlock.

---

## 29.14 Куда идти дальше?

Мы разобрали sharded locks — разбиение блокировок. Теперь мы умеем масштабировать кэши.

Но остаётся **фундаментальный вопрос**: можно ли вообще **избежать** блокировок? Как построить **lock-free** структуры?

- **Как построить lock-free структуры?** → **Глава 30: Lock-free структуры.**
- **Как использовать `sync.Pool`?** → **Глава 31: sync.Pool и аллокатор.**
- **Как работает GC?** → **Глава 32: GC и конкурентный код.**

---

## 29.15 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Sharded locks** | N мьютексов | Вместо одного |
| **Sharded map** | Классика | 32–256 шардов |
| **Число шардов** | 2–4× GOMAXPROCS | Максимум 256–1024 |
| **Hash** | FNV, maphash, xxhash | Равномерность |
| **maphash** | Встроен | Быстрее FNV |
| **Sharded RWMutex** | По ключу | Связанные данные |
| **Multi-key** | Сортируй шарды | Избегай deadlock |
| **vs sync.Map** | write-heavy vs read-heavy | — |
| **Метрики** | Mutex profile | До/после |
| **Distribution** | Равномерность | Важно |
| **+ atomic** | Счётчики | — |
| **+ sync.Pool** | Пулы | — |
| **+ worker pool** | Очереди | — |

🔐 **Ключевая идея:** Sharded locks — разбиение одного мьютекса на N шардов. **Sharded map** — классический пример. **Число шардов:** 2–4× GOMAXPROCS, максимум 256–1024. **Hash:** FNV, maphash, xxhash; `maphash` встроен и быстрее FNV. **Sharded RWMutex** — для связанных данных (шардируй по ключу). **Multi-key операции** — сортируй шарды перед lock. **Sharded locks vs sync.Map:** write-heavy vs read-heavy. **Измеряй** через mutex profile и бенчмарки. Комбинируется с atomic, sync.Pool, worker pool. **6x быстрее** с 32 шардами. Не используй sharding для 5 горутин и read-only.