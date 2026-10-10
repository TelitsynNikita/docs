# 🔓 Глава 30: Lock-free структуры

**Что вы узнаете:**
- Что такое lock-free и чем отличается от mutex-based.
- Что такое **CAS** (Compare-And-Swap) и почему он основа.
- Что такое **ABA-проблема** и как её решают.
- Как построить **lock-free стек** и **очередь**.
- Почему lock-free не всегда быстрее.
- Что такое **lock-free vs wait-free**.
- Как использовать `atomic.Pointer` для lock-free структур.
- Как тестировать lock-free код.

**После прочтения вы сможете:**
- Объяснить разницу между lock-free и mutex-based.
- Построить простой lock-free стек.
- Понимать ABA-проблему и способы её решения.
- Осознанно выбирать между lock-free и mutex.
- Тестировать lock-free код.
- Понимать, где lock-free уместен, а где — нет.

---

## Содержание

- [30.0 Пролог: мьютекс, который тормозит](#300-пролог-мьютекс-который-тормозит)
- [30.1 Что такое lock-free](#301-что-такое-lock-free)
- [30.2 CAS: Compare-And-Swap](#302-cas-compare-and-swap)
- [30.3 Lock-free стек](#303-lock-free-стек)
- [30.4 ABA-проблема](#304-aba-проблема)
- [30.5 Lock-free очередь](#305-lock-free-очередь)
- [30.6 Lock-free vs mutex](#306-lock-free-vs-mutex)
- [30.7 Lock-free vs wait-free](#307-lock-free-vs-wait-free)
- [30.8 Тестирование lock-free](#308-тестирование-lock-free)
- [30.9 В связке с другими паттернами](#309-в-связке-с-другими-паттернами)
- [30.10 Практика Go: lock-free счётчик и стек](#3010-практика-go-lock-free-счётчик-и-стек)
- [30.11 Выводы и типичные ошибки](#3011-выводы-и-типичные-ошибки)
- [30.12 Для быстрого повторения](#3012-для-быстрого-повторения)
- [30.13 Вопросы для самопроверки](#3013-вопросы-для-самопроверки)
- [30.14 Ответы](#3014-ответы)
- [30.15 Куда идти дальше?](#3015-куда-идти-дальше)
- [30.16 Чек-лист](#3016-чек-лист)

---

## 30.0 Пролог: мьютекс, который тормозит

У нас есть сервис с **счётчиком** — 1000 горутин постоянно инкрементируют его:

```go
type Counter struct {
    mu sync.Mutex
    n  int64
}

func (c *Counter) Inc() {
    c.mu.Lock()
    c.n++
    c.mu.Unlock()
}
```

Работает. Но в CPU-профиле видно:

```
flat  flat%   sum%        cum   cum%
 8.5s 45.00% 45.00%      8.5s 45.00%  sync.(*Mutex).Lock
```

**45% CPU** уходит на `Mutex.Lock` — contention. Все 1000 горутин **конкурируют** за один мьютекс. Даже если операция **тривиальная** (`c.n++`).

Хочется: **не блокироваться**. Инкремент — **атомарная** операция. Зачем мьютекс, если можно `atomic.AddInt64`?

```go
type Counter struct {
    n atomic.Int64
}

func (c *Counter) Inc() {
    c.n.Add(1)
}
```

**Что даёт:** нет `Mutex.Lock` вообще. Инкремент — **одна инструкция процессора** (`LOCK XADD`).

**Это и есть lock-free** — операция выполняется **без блокировки**.

> **Мост к следующим главам:** lock-free — продвинутая тема. Она требует понимания memory model (Глава 7) и atomic (Глава 4). В production lock-free используется **редко** — только там, где mutex реально bottleneck. Понимание lock-free даёт понимание, **как работает атомарность на уровне CPU**.

---

## 30.1 Что такое lock-free

**Lock-free** — структура данных, в которой **хотя бы одна** операция всегда **завершается** без блокировки других потоков.

### Формальное определение

**Lock-free** — система **lock-free**, если:
- Хотя бы **один** поток всегда продвигается.
- Нет **взаимной блокировки** (deadlock).
- Нет **голодания** (starvation) — хотя бы один поток завершится.

**Ключевое:** если один поток **завис** — остальные **продолжают** работать.

### Что НЕ lock-free

**Mutex-based структура** — не lock-free. Если поток с мьютексом **завис** — остальные **ждут вечно**.

```go
type Stack struct {
    mu sync.Mutex
    items []int
}

func (s *Stack) Push(v int) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.items = append(s.items, v)
}
```

**Если `Push` зависнет с захваченным мьютексом** — все остальные операции **ждут вечно**.

### Что lock-free

**Atomic-based структура** — lock-free. Даже если один поток **завис** — остальные **продолжают**.

```go
type Counter struct {
    n atomic.Int64
}

func (c *Counter) Inc() {
    c.n.Add(1)  // ← одна инструкция, всегда завершается
}
```

### Схема

```
Mutex-based:                     Lock-free:
  G1: Lock ─── [работа] ── Unlock   G1: CAS ──┐
  G2: Lock ── ждёт ── Lock ── ...            │
  G3: Lock ── ждёт ── ждёт ── ...   G2: CAS ──┼──► успех
  G4: Lock ── ждёт ── ждёт ── ...            │
                                    G3: CAS ──┘
  Если G1 завис — все ждут.         Если G1 завис — G2, G3 продолжают.
```

### Когда использовать lock-free

**1. Очень высокая конкуренция.**

- Тысячи горутин на одной структуре.
- Mutex профиль показывает 40%+ на `Mutex.Lock`.

**2. Тривиальные операции.**

- Инкремент, декремент, swap.
- `atomic` достаточно.

**3. Критичные по latency.**

- Нельзя ждать мьютекс.

### Когда НЕ использовать

**1. Сложные операции.**

- Транзакции над несколькими полями.
- Lock-free код **очень сложно** писать и отлаживать.

**2. Умеренная нагрузка.**

- Mutex **достаточен**.
- Lock-free **не даст** выгоды.

**3. Правильность важнее скорости.**

- Lock-free код **легко сломать**.
- Mutex **проще** и **надёжнее**.

### Аналогия: касса vs самообслуживание

**Mutex** — как **касса с кассиром**. Один человек работает, остальные ждут.

**Lock-free** — как **кассы самообслуживания**. Каждый работает сам, **не блокируя** других.

### 💡 Практика: как думать о lock-free

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Lock-free — для тривиальных операций.**
2. **Atomic — для счётчиков, флагов, указателей.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Mutex — по умолчанию.**
4. **Lock-free — только если mutex bottleneck.**

**❌ НЕ ДЕЛАЙ:**

5. **Не пиши lock-free, если можно mutex.**
6. **Не используй lock-free для сложных операций.**

---

## 30.2 CAS: Compare-And-Swap

**CAS** — основа всех lock-free структур.

### Что такое CAS

**CAS(addr, old, new)** — атомарная операция:

1. Прочитать `*addr`.
2. Если `*addr == old`:
   - `*addr = new`.
   - Вернуть `true`.
3. Иначе:
   - Вернуть `false`.

**Всё атомарно** — одна инструкция процессора (`LOCK CMPXCHG` на x86).

### CAS в Go

```go
var v atomic.Int64
v.Store(10)

swapped := v.CompareAndSwap(10, 20)
// true, если было 10
```

### CAS retry loop

**Классический паттерн:**

```go
var v atomic.Int64

func increment() {
    for {
        old := v.Load()
        new := old + 1
        if v.CompareAndSwap(old, new) {
            return  // успех
        }
        // неудача: кто-то изменил v между Load и CAS
        // повторяем
    }
}
```

**Что происходит:**

1. Читаем текущее значение.
2. Вычисляем новое.
3. Пытаемся заменить через CAS.
4. Если не удалось — **повторяем**.

### Схема

```
CAS retry loop:

  ┌─────────────────────────────┐
  │ old := v.Load()             │
  │ new := old + 1              │
  │ if v.CAS(old, new):         │
  │   return                    │
  │ else:                       │
  │   repeat ───────────────────┤
  └─────────────────────────────┘
  
  Два потока:
    G1: old=10, new=11, CAS(10,11) → OK
    G2: old=10, new=11, CAS(10,11) → FAIL (уже 11)
    G2: old=11, new=12, CAS(11,12) → OK
```

### Стоимость

| Операция | Time |
|:---|:---|
| `atomic.Load` | ~1–2 нс |
| `atomic.Store` | ~1–2 нс |
| `atomic.CompareAndSwap` | ~5–15 нс |
| `Mutex.Lock/Unlock` | ~15–25 нс |

**CAS быстрее мьютекса** в 2–3 раза для простых операций.

### Проблема: retry loop при высокой конкуренции

**Что если 1000 горутин делают CAS на одном адресе?**

- Каждая попытка **проваливается** с вероятностью 999/1000.
- **Много ретраев**.
- CAS может быть **медленнее** мьютекса.

**Решение:** **шардирование** (Глава 29) или **backoff**.

### 💡 Практика: как использовать CAS

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **CAS для lock-free операций.**
2. **Retry loop — стандартный паттерн.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Backoff при высоком contention.**
4. **Шардирование для счётчиков.**

**❌ НЕ ДЕЛАЙ:**

5. **Не используй CAS для сложных операций.**
6. **Не игнорируй contention.**

---

## 30.3 Lock-free стек

Разберём **классический** пример — lock-free стек.

### Идея

**Стек** — LIFO структура. Операции:
- **Push** — добавить в голову.
- **Pop** — удалить из головы.

**Lock-free стек** — на **связном списке** с CAS.

### Реализация

```go
type node struct {
    value int
    next  *node
}

type Stack struct {
    head atomic.Pointer[node]
}

func (s *Stack) Push(value int) {
    n := &node{value: value}
    for {
        old := s.head.Load()
        n.next = old
        if s.head.CompareAndSwap(old, n) {
            return
        }
        // кто-то изменил head — повторяем
    }
}

func (s *Stack) Pop() (int, bool) {
    for {
        old := s.head.Load()
        if old == nil {
            return 0, false  // стек пуст
        }
        next := old.next
        if s.head.CompareAndSwap(old, next) {
            return old.value, true
        }
        // кто-то изменил head — повторяем
    }
}
```

**Что происходит:**

- **Push:** создаём узел, пытаемся заменить `head` через CAS.
- **Pop:** читаем `head`, пытаемся заменить на `head.next` через CAS.
- **Retry loop:** если CAS провалился — повторяем.

### Схема

```
Push(42):

  1. old := head (например, узел 10)
  2. n := &node{42, next: old}
  3. head.CAS(old, n)
  
  ┌─────────┐      ┌─────────┐
  │ head=10 │──►   │ node 42 │──► old
  └─────────┘      └─────────┘
  
  Если CAS успешен:
  ┌─────────┐      ┌─────────┐
  │ head=42 │──►   │ node 10 │──► ...
  └─────────┘      └─────────┘

Pop():

  1. old := head (например, узел 42)
  2. next := old.next (узел 10)
  3. head.CAS(old, next)
  
  Если CAS успешен:
  head=10
```

### Потребитель

```go
func main() {
    var s Stack
    
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        i := i
        wg.Add(1)
        go func() {
            defer wg.Done()
            s.Push(i)
        }()
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
    fmt.Println("count:", count)
}
```

### Проблема: ABA

**CAS проверяет только значение, а не "версию".** Если между `Load` и `CAS` значение **изменилось и вернулось** к старому — CAS **пройдёт**, но состояние **изменилось**.

**Пример:**

```
t=0:  head = A → B → C
t=1:  G1: old = A, next = B
t=2:  G2: Pop A (head = B)
t=3:  G2: Pop B (head = C)
t=4:  G2: Push A (head = A → C)   ← новый A, но значение то же
t=5:  G1: CAS(A, B) → успех!
      Но head должен быть C, а стал B → потеряли C
```

**Это ABA-проблема.** Разберём в 30.4.

### 💡 Практика: как писать lock-free стек

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`atomic.Pointer[node]`** — для head.
2. **Retry loop** — при провале CAS.
3. **Помни про ABA.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Тегированные указатели** — для ABA.
5. **Тесты с race detector.**

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай про ABA.**
7. **Не используй lock-free стек для сложных данных.**

---

## 30.4 ABA-проблема

**ABA** — самая известная проблема lock-free структур.

### Что такое ABA

**CAS(addr, old, new)** проходит, если `*addr == old`. Но если между `Load(old)` и `CAS` значение **изменилось A → B → A**, CAS **пройдёт**, хотя состояние **изменилось**.

### Пример

```
t=0:  head = A
      G1: old = A, next = ?
t=1:  G2: Pop A (head = B)
t=2:  G2: Pop B (head = C)
t=3:  G2: Push A (head = A)   ← новый узел A, но адрес тот же? 
t=4:  G1: CAS(A, ?) → успех!
      Но структура изменилась.
```

**Что происходит:** G1 думает, что структура **не изменилась**, но это не так.

### Решение 1: тегированные указатели

**Идея:** к указателю **добавить** счётчик версии. CAS проверяет **и указатель, и версию**.

```go
type taggedPointer struct {
    ptr uintptr
    tag uint64
}
```

**Проблема:** Go **не даёт** атомарно менять 128 бит (указатель + счётчик). На x86 есть `CMPXCHG16B`, но Go **не использует** его напрямую.

### Решение 2: hazard pointers

**Идея:** каждый поток **регистрирует** указатель, который он читает. Другие потоки **не удаляют** объекты, на которые есть hazard pointer.

**Проблема:** сложно реализовать и **медленно**.

### Решение 3: epoch-based reclamation

**Идея:** объекты **не удаляются сразу**. Удаление происходит, когда **все потоки** прошли "эпоху".

**Проблема:** сложно, требует глобального состояния.

### Решение 4: не удалять вообще

**Идея:** lock-free структура **только добавляет**. Удаление — **логическое** (пометка).

**Проблема:** память **растёт**.

### Решение 5: использовать готовые структуры

**Go не даёт** готовых lock-free структур (кроме `sync.Map` и `atomic`). **В production** lock-free структуры **почти не используют** — слишком сложно.

### Что делать в Go

**1. `sync.Map`** — для map.

**2. `atomic.Pointer`** — для одиночного указателя (без ABA).

**3. `channel`** — вместо lock-free очереди.

**4. `Mutex`** — по умолчанию.

### Схема

```
ABA:
  A → B → A
  
  G1: Load(A)
  G2: A → B → A
  G1: CAS(A, X) → успех
      Но состояние изменилось!

Решение:
  Тегированный указатель: (A, 1) → (B, 2) → (A, 3)
  G1: Load((A, 1))
  G2: A → B → A
  G1: CAS((A, 1), X) → FAIL (уже (A, 3))
```

### 💡 Практика: как избежать ABA

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Помни про ABA** при работе с указателями.
2. **Используй `atomic.Pointer`** для **одиночного** указателя.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Тегированные указатели** — если нужно.
4. **Hazard pointers** — если очень нужно.

**❌ НЕ ДЕЛАЙ:**

5. **Не пиши lock-free структуры с удалением** без решения ABA.
6. **Не используй lock-free для сложных структур.**

---

## 30.5 Lock-free очередь

**Lock-free очередь** — сложнее стека из-за **двух** указателей (head и tail).

### Идея

**Очередь** — FIFO. Нужны **два** указателя: **head** (front) и **tail** (back).

**Michael-Scott queue** — классический алгоритм.

### Реализация (упрощённо)

```go
type node struct {
    value int
    next  atomic.Pointer[node]
}

type Queue struct {
    head atomic.Pointer[node]
    tail atomic.Pointer[node]
}

func NewQueue() *Queue {
    dummy := &node{}
    q := &Queue{}
    q.head.Store(dummy)
    q.tail.Store(dummy)
    return q
}

func (q *Queue) Enqueue(value int) {
    n := &node{value: value}
    for {
        tail := q.tail.Load()
        next := tail.next.Load()
        if tail == q.tail.Load() {  // tail не изменился
            if next == nil {
                if tail.next.CompareAndSwap(nil, n) {
                    // Успех
                    q.tail.CompareAndSwap(tail, n)  // может провалиться — не страшно
                    return
                }
            } else {
                // tail отстал — помогаем
                q.tail.CompareAndSwap(tail, next)
            }
        }
    }
}

func (q *Queue) Dequeue() (int, bool) {
    for {
        head := q.head.Load()
        tail := q.tail.Load()
        next := head.next.Load()
        if head == q.head.Load() {
            if head == tail {
                if next == nil {
                    return 0, false  // очередь пуста
                }
                // tail отстал — помогаем
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
```

**Что происходит:**

- **Enqueue:** добавляем узел в tail через CAS.
- **Dequeue:** удаляем узел из head через CAS.
- **Retry loop:** при провале CAS.
- **Помощь:** если один поток отстал — другой помогает.

### Схема

```
Enqueue(42):

  1. tail := q.tail.Load()
  2. next := tail.next.Load()  ← nil
  3. tail.next.CAS(nil, n) → успех
  4. q.tail.CAS(tail, n) → успех или нет (не страшно)
  
  ┌─────────┐      ┌─────────┐
  │head=dummy│──►  │ node 42 │
  └─────────┘      └─────────┘
       ▲                ▲
      head             tail

Dequeue():

  1. head := q.head.Load() (dummy)
  2. next := head.next.Load() (node 42)
  3. q.head.CAS(dummy, node 42) → успех
  4. return node 42.value
  
  head теперь указывает на node 42.
```

### Проблема: ABA в очереди

Очередь **тоже** страдает от ABA. Решение — **тегированные указатели** или **не удалять dummy-узлы**.

### Когда использовать

**В Go lock-free очередь используется редко.** Вместо неё:

- **Каналы** — встроены, безопасны.
- **`sync.Mutex` + slice** — проще.
- **`container/list` + `Mutex`** — для сложных случаев.

### 💡 Практика: как писать lock-free очередь

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Michael-Scott queue** — классический алгоритм.
2. **Dummy-узел** — упрощает код.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Помни про ABA.**
4. **В Go используй каналы** — проще.

**❌ НЕ ДЕЛАЙ:**

5. **Не пиши lock-free очередь для production.**
6. **Не забывай про помощь** (helping).

---

## 30.6 Lock-free vs mutex

Разберём **разницу** и **когда что**.

### Сравнение

| Аспект | Mutex | Lock-free |
|:---|:---|:---|
| Блокировка | Да | Нет |
| Сложность кода | Низкая | Очень высокая |
| Правильность | Легко | Легко сломать |
| Отладка | Просто | Очень сложно |
| Contention | Зависит | Может быть retry storm |
| Latency | Может быть высокой | Ниже |
| Throughput | Высокий | Может быть выше |
| Use case | Большинство | Только special case |

### Когда mutex лучше

**1. Сложные операции.**

- Транзакции над несколькими полями.
- Структуры данных с инвариантами.

**2. Умеренная нагрузка.**

- Mutex **достаточен**.
- Lock-free **не даст** выгоды.

**3. Правильность критична.**

- Mutex **проще** и **надёжнее**.

### Когда lock-free лучше

**1. Очень высокая конкуренция.**

- Тысячи горутин на одной структуре.
- Mutex профиль показывает 40%+ на `Mutex.Lock`.

**2. Тривиальные операции.**

- Инкремент, декремент, swap.

**3. Критичная latency.**

- Нельзя ждать мьютекс.

### Бенчмарк

```go
func BenchmarkMutexCounter(b *testing.B) {
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

func BenchmarkAtomicCounter(b *testing.B) {
    var counter atomic.Int64
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            counter.Add(1)
        }
    })
}
```

**Пример вывода:**

```
MutexCounter:   50000000    25 ns/op
AtomicCounter: 100000000    10 ns/op
```

**Что видно:** `atomic` в 2.5 раза быстрее.

### Практическое правило

**В Go:** начинай с `Mutex`. Если contention высокий — попробуй `atomic` для простых операций. **Lock-free структуры** для сложных случаев — **очень редко**.

### 💡 Практика: как выбирать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Mutex — по умолчанию.**
2. **Atomic — для простых операций.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Lock-free — только если mutex bottleneck.**
4. **Шардирование** — прежде чем lock-free.

**❌ НЕ ДЕЛАЙ:**

5. **Не пиши lock-free без необходимости.**
6. **Не используй lock-free для сложных структур.**

---

## 30.7 Lock-free vs wait-free

Разберём **два** уровня неблокирующих структур.

### Lock-free

**Lock-free** — хотя бы **один** поток **продвигается**.

**Пример:** CAS retry loop. Если один поток проваливает CAS — другой **пройдёт**.

### Wait-free

**Wait-free** — **каждый** поток **завершается** за **ограниченное** число шагов.

**Пример:** `atomic.AddInt64` — одна инструкция, всегда завершается.

### Сравнение

| Аспект | Lock-free | Wait-free |
|:---|:---|:---|
| Прогресс | Хотя бы один | Каждый |
| Retry | Возможен | Нет |
| Сложность | Высокая | Очень высокая |
| Гарантия | Частичная | Полная |
| Use case | CAS-структуры | Простые atomic |

### Схема

```
Lock-free:

  G1: CAS ──┐
  G2: CAS ──┼──► успех (G2)
  G3: CAS ──┘     G1, G3 retry

Wait-free:

  G1: AddInt64 ──► успех
  G2: AddInt64 ──► успех
  G3: AddInt64 ──► успех
  
  Все завершились за 1 шаг.
```

### В Go

**`atomic.Add`, `atomic.Load`, `atomic.Store`** — wait-free.

**CAS retry loop** — lock-free.

**Lock-free структуры с retry** — lock-free, но **не wait-free**.

### 💡 Практика: как думать о wait-free

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Простые atomic — wait-free.**
2. **CAS retry — lock-free.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Wait-free — если нужна гарантия latency.**

**❌ НЕ ДЕЛАЙ:**

4. **Не пиши wait-free структуры** без необходимости.

---

## 30.8 Тестирование lock-free

**Lock-free код сложно тестировать.** Разберём подходы.

### Проблемы

**1. Недетерминизм.**

- Retry loop может сработать 1000 раз и упасть на 1001-й.

**2. ABA.**

- Проявляется **редко**.

**3. Гонки.**

- Легко пропустить.

### Подходы

**1. Race detector.**

```bash
go test -race ./...
```

**Что находит:**

- Data race на `atomic` полях (если используешь `unsafe`).
- Не находит ABA.

**2. Stress-тесты.**

```go
func TestLockFreeStack(t *testing.T) {
    for iter := 0; iter < 1000; iter++ {
        t.Run(fmt.Sprintf("iter-%d", iter), func(t *testing.T) {
            testStack(t)
        })
    }
}
```

**3. Разные GOMAXPROCS.**

```go
for _, procs := range []int{1, 2, 4, 8} {
    t.Run(fmt.Sprintf("GOMAXPROCS=%d", procs), func(t *testing.T) {
        old := runtime.GOMAXPROCS(procs)
        defer runtime.GOMAXPROCS(old)
        testStack(t)
    })
}
```

**4. Проверка инвариантов.**

```go
func TestStackInvariants(t *testing.T) {
    var s Stack
    var wg sync.WaitGroup
    
    // 100 горутин push
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(v int) {
            defer wg.Done()
            s.Push(v)
        }(i)
    }
    wg.Wait()
    
    // Проверяем: всего 100 элементов
    count := 0
    for {
        _, ok := s.Pop()
        if !ok {
            break
        }
        count++
    }
    if count != 100 {
        t.Errorf("got %d, want 100", count)
    }
}
```

### Модель для тестирования

**`go test -race -count=1000 -cpu=1,2,4,8 ./...`** — максимальная нагрузка.

### 💡 Практика: как тестировать lock-free

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Race detector** — обязательно.
2. **Stress-тесты** — 1000+ итераций.
3. **Разные GOMAXPROCS.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Проверка инвариантов.**

**❌ НЕ ДЕЛАЙ:**

5. **Не полагайся на один запуск.**
6. **Не игнорируй редкие падения.**

---

## 30.9 В связке с другими паттернами

Lock-free редко используется **в одиночку**. Разберём связки.

### Lock-free + sharded locks

**Шардированный atomic-счётчик:**

```go
type ShardedCounter struct {
    shards [32]atomic.Int64
}

func (c *ShardedCounter) Add(n int) {
    shard := n % 32
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

**Что даёт:** нет contention даже на `atomic`.

### Lock-free + worker pool

**Lock-free очередь задач:**

```go
type WorkerPool struct {
    tasks *LockFreeQueue
}

func (wp *WorkerPool) Submit(task Task) {
    wp.tasks.Enqueue(task)
}
```

**Что даёт:** воркеры **не блокируются** при взятии задачи.

**В Go лучше:** использовать каналы.

### Lock-free + rate limiter

**Token bucket с atomic:**

```go
type TokenBucket struct {
    tokens atomic.Int64
    last   atomic.Int64
}

func (tb *TokenBucket) TryAcquire() bool {
    for {
        tokens := tb.tokens.Load()
        if tokens <= 0 {
            return false
        }
        if tb.tokens.CompareAndSwap(tokens, tokens-1) {
            return true
        }
    }
}
```

### Lock-free + circuit breaker

**Circuit breaker с atomic state:**

```go
type CircuitBreaker struct {
    state atomic.Int32
}

func (cb *CircuitBreaker) State() State {
    return State(cb.state.Load())
}

func (cb *CircuitBreaker) SetState(s State) {
    cb.state.Store(int32(s))
}
```

**Что даёт:** нет мьютекса на state.

### 💡 Практика: как комбинировать lock-free

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Lock-free + sharded** — для счётчиков.
2. **Lock-free + circuit breaker** — для state.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Lock-free + rate limiter** — для токенов.

**❌ НЕ ДЕЛАЙ:**

4. **Не пиши lock-free очередь** — используй каналы.

---

## 30.10 Практика Go: lock-free счётчик и стек

Разберём **два примера**.

### Пример 1: lock-free счётчик

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
)

type Counter struct {
    n atomic.Int64
}

func (c *Counter) Inc() {
    c.n.Add(1)
}

func (c *Counter) Value() int64 {
    return c.n.Load()
}

func main() {
    var c Counter
    
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < 10000; j++ {
                c.Inc()
            }
        }()
    }
    wg.Wait()
    
    fmt.Println("Counter:", c.Value())
}
```

**Пример вывода:**

```
Counter: 10000000
```

**Что видно:** 10 000 000 инкрементов без мьютекса.

### Пример 2: lock-free стек

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
)

type node struct {
    value int
    next  *node
}

type Stack struct {
    head atomic.Pointer[node]
}

func (s *Stack) Push(value int) {
    n := &node{value: value}
    for {
        old := s.head.Load()
        n.next = old
        if s.head.CompareAndSwap(old, n) {
            return
        }
    }
}

func (s *Stack) Pop() (int, bool) {
    for {
        old := s.head.Load()
        if old == nil {
            return 0, false
        }
        next := old.next
        if s.head.CompareAndSwap(old, next) {
            return old.value, true
        }
    }
}

func main() {
    var s Stack
    
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        i := i
        wg.Add(1)
        go func() {
            defer wg.Done()
            s.Push(i)
        }()
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
    fmt.Println("Count:", count)
}
```

**Пример вывода:**

```
Count: 1000
```

### Пример 3: сравнение Mutex vs atomic

```go
func BenchmarkMutexCounter(b *testing.B) {
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

func BenchmarkAtomicCounter(b *testing.B) {
    var counter atomic.Int64
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            counter.Add(1)
        }
    })
}
```

**Пример вывода:**

```
MutexCounter:   50000000    25 ns/op
AtomicCounter: 100000000    10 ns/op
```

**Что видно:** `atomic` в 2.5 раза быстрее.

### 💡 Практика: как использовать lock-free

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Atomic для счётчиков и флагов.**
2. **`atomic.Pointer` для указателей.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **Lock-free стек — для понимания.**
4. **Race detector + stress-тесты.**

**❌ НЕ ДЕЛАЙ:**

5. **Не пиши lock-free очередь для production.**
6. **Не забывай про ABA.**

---

## 30.11 Выводы и типичные ошибки

**Что мы узнали?**

Lock-free — структура, в которой хотя бы один поток продвигается без блокировки. **CAS** — основа. **ABA-проблема** — когда значение меняется A→B→A, CAS проходит, но состояние изменилось. Решения: тегированные указатели, hazard pointers, epoch-based reclamation. **Lock-free стек** — на связном списке с CAS. **Lock-free очередь** — Michael-Scott, сложнее. **Lock-free vs mutex:** сложность vs простота. **Lock-free vs wait-free:** хотя бы один vs каждый. **Тестирование:** race detector, stress-тесты, разные GOMAXPROCS.

**Типичные ошибки:**

- ❌ **Писать lock-free без необходимости.** Mutex проще.
- ❌ **Игнорировать ABA.** Стек может сломаться.
- ❌ **Использовать lock-free для сложных структур.**
- ❌ **Не тестировать с race detector.**
- ❌ **Не тестировать с разными GOMAXPROCS.**
- ❌ **Писать lock-free очередь для production.**
- ❌ **Не понимать разницу lock-free vs wait-free.**
- ❌ **Использовать `unsafe` без необходимости.**
- ❌ **Retry loop без backoff при высоком contention.**
- ❌ **Не измерять эффект.**

---

## 30.12 Для быстрого повторения

- **Lock-free** — хотя бы один поток продвигается.
- **CAS** — основа lock-free. `CompareAndSwap(old, new)`.
- **Retry loop** — стандартный паттерн.
- **ABA** — A→B→A. CAS проходит, но состояние изменилось.
- **Решения ABA:** тегированные указатели, hazard pointers, epoch.
- **Lock-free стек** — связный список + CAS.
- **Lock-free очередь** — Michael-Scott, сложнее.
- **Lock-free vs mutex:** сложность vs простота.
- **Lock-free vs wait-free:** хотя бы один vs каждый.
- **`atomic.Add`** — wait-free.
- **`atomic.CompareAndSwap`** — lock-free.
- **В Go:** начинай с Mutex.
- **Atomic** — для простых операций.
- **Lock-free** — только если mutex bottleneck.
- **Тестирование:** race detector, stress-тесты, GOMAXPROCS.

---

## 30.13 Вопросы для самопроверки

1. Что такое lock-free? Чем отличается от mutex?
2. Что такое CAS? Как использовать?
3. Что такое ABA-проблема?
4. Как решить ABA?
5. Как построить lock-free стек?
6. Чем lock-free очередь сложнее стека?
7. Чем lock-free отличается от wait-free?
8. Когда lock-free лучше mutex?
9. Как тестировать lock-free код?
10. Что выбрать — mutex или lock-free?

---

## 30.14 Ответы

### Ответ 1

**Lock-free** — структура, в которой **хотя бы один** поток всегда **продвигается** без блокировки.

**Mutex-based** — если поток с мьютексом **завис**, остальные **ждут вечно**.

**Lock-free** — если один поток **завис**, остальные **продолжают**.

**Пример:** `atomic.AddInt64` — lock-free. `Mutex.Lock` — нет.

### Ответ 2

**CAS(addr, old, new)** — атомарная операция:

1. Прочитать `*addr`.
2. Если `*addr == old` → `*addr = new`, вернуть `true`.
3. Иначе → вернуть `false`.

```go
var v atomic.Int64
v.Store(10)
swapped := v.CompareAndSwap(10, 20)  // true
```

**Retry loop:**

```go
for {
    old := v.Load()
    new := old + 1
    if v.CompareAndSwap(old, new) {
        return
    }
}
```

### Ответ 3

**ABA-проблема:** между `Load(old)` и `CAS(old, new)` значение меняется `A → B → A`. CAS **проходит**, но состояние **изменилось**.

**Пример:**

```
t=0: head = A → B → C
G1: old = A, next = B
G2: Pop A (head = B)
G2: Pop B (head = C)
G2: Push A (head = A → C)
G1: CAS(A, B) → успех, но потеряли C
```

### Ответ 4

**Решения ABA:**

1. **Тегированные указатели** — к указателю добавить счётчик версии.
2. **Hazard pointers** — регистрировать читаемые указатели.
3. **Epoch-based reclamation** — не удалять сразу.
4. **Не удалять** — только добавлять.

**В Go:** `atomic.Pointer` для **одиночного** указателя (без ABA). Сложные структуры — используй `Mutex`.

### Ответ 5

**Lock-free стек:**

```go
type node struct {
    value int
    next  *node
}

type Stack struct {
    head atomic.Pointer[node]
}

func (s *Stack) Push(value int) {
    n := &node{value: value}
    for {
        old := s.head.Load()
        n.next = old
        if s.head.CompareAndSwap(old, n) {
            return
        }
    }
}

func (s *Stack) Pop() (int, bool) {
    for {
        old := s.head.Load()
        if old == nil {
            return 0, false
        }
        next := old.next
        if s.head.CompareAndSwap(old, next) {
            return old.value, true
        }
    }
}
```

### Ответ 6

**Lock-free очередь** сложнее стека из-за:

1. **Двух указателей:** head и tail.
2. **Помощи:** если один поток отстал, другой помогает.
3. **ABA:** сложнее решить.

**Michael-Scott queue** — классический алгоритм.

### Ответ 7

**Lock-free** — хотя бы **один** поток продвигается.

**Wait-free** — **каждый** поток завершается за **ограниченное** число шагов.

**Примеры:**

- `atomic.Add` — wait-free.
- `CAS retry loop` — lock-free.
- Lock-free структуры с retry — lock-free, но не wait-free.

### Ответ 8

**Lock-free лучше mutex, когда:**

1. **Очень высокая конкуренция** (тысячи горутин).
2. **Тривиальные операции** (инкремент).
3. **Критичная latency.**

**В большинстве случаев:** mutex **достаточен** и **проще**.

### Ответ 9

**Тестирование:**

1. **`go test -race`** — обязательно.
2. **Stress-тесты** — `-count=1000`.
3. **Разные GOMAXPROCS** — `-cpu=1,2,4,8`.
4. **Проверка инвариантов.**

**Race detector не находит ABA** — нужны доп. тесты.

### Ответ 10

**Выбор:**

- **Mutex — по умолчанию.**
- **Atomic — для простых операций.**
- **Lock-free — только если mutex bottleneck.**
- **Шардирование** — прежде чем lock-free.

**В Go lock-free структуры** используются **редко** — слишком сложны.

---

## 30.15 Куда идти дальше?

Мы разобрали lock-free структуры — продвинутая тема. Теперь мы понимаем, как работает атомарность на уровне CPU.

Но остаётся **важный вопрос**: как **переиспользовать** объекты без GC? Как снизить давление на аллокатор?

- **Как использовать `sync.Pool`?** → **Глава 31: sync.Pool и аллокатор.**
- **Как работает GC?** → **Глава 32: GC и конкурентный код.**
- **Какие анти-паттерны существуют?** → **Глава 33: Анти-паттерны.**

---

## 30.16 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Lock-free** | Хотя бы один продвигается | Без блокировки |
| **CAS** | Compare-And-Swap | `CompareAndSwap(old, new)` |
| **Retry loop** | Повтор при провале | Стандартный паттерн |
| **ABA** | A→B→A | CAS проходит, но состояние изменилось |
| **Тегированные указатели** | Решение ABA | Указатель + версия |
| **Lock-free стек** | Связный список + CAS | `atomic.Pointer` |
| **Lock-free очередь** | Michael-Scott | Сложнее |
| **Lock-free vs mutex** | Сложность vs простота | Mutex — по умолчанию |
| **Lock-free vs wait-free** | Хотя бы один vs каждый | — |
| **`atomic.Add`** | Wait-free | Одна инструкция |
| **`atomic.CompareAndSwap`** | Lock-free | Retry loop |
| **Тестирование** | Race detector + stress | + разные GOMAXPROCS |
| **В Go** | Редко | Mutex, каналы, atomic |

🔓 **Ключевая идея:** Lock-free — структура, в которой хотя бы один поток продвигается без блокировки. **CAS** — основа: `CompareAndSwap(old, new)`, retry loop. **ABA-проблема:** A→B→A, CAS проходит, но состояние изменилось. Решения: тегированные указатели, hazard pointers, epoch-based reclamation. **Lock-free стек** — связный список + CAS. **Lock-free очередь** — Michael-Scott, сложнее из-за head и tail. **Lock-free vs mutex:** сложность vs простота. **Lock-free vs wait-free:** хотя бы один vs каждый. `atomic.Add` — wait-free. `atomic.CompareAndSwap` — lock-free. **В Go** lock-free используется редко: начинай с Mutex, atomic для простых операций, шардирование прежде чем lock-free. Тестирование: race detector, stress-тесты, разные GOMAXPROCS.