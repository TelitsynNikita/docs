# 🧭 Глава 5: Context — отмена, таймауты, дедлайны

**Что вы узнаете:**
- Почему «убить горутину» нельзя и что делать вместо этого.
- Что такое `context.Context` и какие задачи он решает.
- Как устроен `cancelCtx` и дерево отмены.
- Как работают `WithCancel`, `WithTimeout`, `WithDeadline`.
- Что такое `WithValue` и почему его не любят.
- Как правильно проверять отмену в горутинах.
- Как избежать утечек через `context`.
- Как комбинировать `context` с `select`, `Mutex`, `WaitGroup`.

**После прочтения вы сможете:**
- Объяснить, почему отмена — это **кооперация**, а не «убийство».
- Проектировать API, принимающий `context.Context` первым аргументом.
- Понимать, что происходит внутри `cancelCtx` при отмене.
- Правильно использовать `WithTimeout` и `WithDeadline` без утечек таймеров.
- Избегать анти-паттернов `WithValue`.

---

## Содержание

- [5.0 Пролог: операция, которую надо уметь прервать](#50-пролог-операция-которую-надо-уметь-прервать)
- [5.1 Почему нельзя убить горутину](#51-почему-нельзя-убить-горутину)
- [5.2 Context: интерфейс и контракт](#52-context-интерфейс-и-контракт)
- [5.3 Дерево отмены: как устроен cancelCtx](#53-дерево-отмены-как-устроен-cancelctx)
- [5.4 WithCancel: ручная отмена](#54-withcancel-ручная-отмена)
- [5.5 WithTimeout и WithDeadline](#55-withtimeout-и-withdeadline)
- [5.6 WithValue: что это и почему его не любят](#56-withvalue-что-это-и-почему-его-не-любят)
- [5.7 Как работает отмена: Done(), Err(), распространение](#57-как-работает-отмена-done-err-распространение)
- [5.8 Context и happens-before](#58-context-и-happens-before)
- [5.9 Context и другие примитивы](#59-context-и-другие-примитивы)
- [5.10 Практика Go: диагностика Context](#510-практика-go-диагностика-context)
- [5.11 Выводы и типичные ошибки](#511-выводы-и-типичные-ошибки)
- [5.12 Для быстрого повторения](#512-для-быстрого-повторения)
- [5.13 Вопросы для самопроверки](#513-вопросы-для-самопроверки)
- [5.14 Ответы](#514-ответы)
- [5.15 Куда идти дальше?](#515-куда-идти-дальше)
- [5.16 Чек-лист](#516-чек-лист)

---

## 5.0 Пролог: операция, которую надо уметь прервать

У нас есть сервис, который ходит за обогащением в другой сервис. Работает так: клиент отправляет запрос, наш сервис делает HTTP-вызов к сервису-обогатителю, получает данные, отдаёт клиенту.

```go
func fetchUser(id int) (*User, error) {
    resp, err := http.Get(fmt.Sprintf("https://api.example.com/users/%d", id))
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    var user User
    if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
        return nil, err
    }
    return &user, nil
}
```

Всё работает. Пока внешний API не начинает тормозить.

Запрос висит 30 секунд. Клиент уже давно ушёл. А наша горутина всё ещё держит соединение, ждёт ответа от API. За минуту таких горутин накапливаются сотни. Соединения не освобождаются. Память растёт.

Хочется уметь сказать: «хватит, прекрати эту операцию». Но `http.Get` не даёт такой возможности.

Можно было бы просто «убить» горутину:

```go
go fetchUser(42)
// через 5 секунд убить?
```

Но в Go **нет способа убить горутину извне**. И это не случайность — это **осознанное решение**. Разберёмся, почему.

> **Мост к следующим главам:** `Context` — это надстройка над каналами (Глава 2) и `select` (Глава 3). Он использует канал `Done()` для сигнала отмены. Понимание `Context` критично для worker pool, pipeline, graceful shutdown.

---

## 5.1 Почему нельзя убить горутину

Прежде чем разбирать `Context`, поймём, **почему** в Go нет `runtime.KillGoroutine(g)`.

### Проблема: горутина держит ресурсы

Представим, что горутина выполняет:

```go
func process() {
    mu.Lock()
    defer mu.Unlock()
    
    data := readFromDB()
    modified := transform(data)
    writeToDB(modified)
}
```

Если «убить» горутину в середине:

- **Мьютекс захвачен** — никто не может его получить. **Deadlock.**
- **Данные в БД** — частично записаны. **Несогласованность.**
- **Транзакция открыта** — останется висеть.
- **Файл открыт** — file descriptor утечёт.
- **Канал** — если горутина пишет, читатель может ждать вечно.

**Ключевое:** горутина не изолирована. Она взаимодействует с системой. Принудительное завершение оставляет систему в **неизвестном состоянии**.

### Кооперативная отмена

Идея: горутина **сама** проверяет сигнал отмены и завершается корректно.

```go
func process(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()  // корректное завершение
        case task := <-tasks:
            if err := handle(task); err != nil {
                return err
            }
        }
    }
}
```

**Что происходит:**

1. Внешний код вызывает `cancel()`.
2. `ctx.Done()` закрывается.
3. Горутина видит `<-ctx.Done()` в `select`.
4. Горутина **сама** возвращает управление.
5. `defer` выполняются. Ресурсы освобождаются.

**Ключевое:** горутина **решает**, когда завершиться. `Context` даёт ей **сигнал**, но не **принуждение**.

### Аналогия: просьба vs приказ

**Принудительное завершение** — как выдернуть провод из работающего компьютера. Быстро, но данные потеряны, файлы повреждены.

**Кооперативная отмена** — как нажать кнопку «Сохранить и выключить». Компьютер завершает работу корректно.

**Go выбрал второе.** `Context` — это кнопка.

### Практическое следствие

**Три следствия:**

1. **Горутина должна проверять `ctx.Done()`.** Если не проверяет — отмена не сработает.
2. **Горутина должна завершаться быстро.** Если она в середине долгой операции — отмена не мгновенна.
3. **Горутина должна освобождать ресурсы.** `defer` выполняются, если горутина завершается корректно.

### 💡 Практика: как проектировать отменяемый код

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Принимай `ctx context.Context` первым аргументом:**
   ```go
   func Process(ctx context.Context, data []byte) error
   ```

2. **Проверяй `ctx.Done()` в циклах и долгих операциях:**
   ```go
   select {
   case <-ctx.Done():
       return ctx.Err()
   default:
   }
   ```

3. **Передавай `ctx` дальше** — во все функции, которые могут долго работать.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Для HTTP-запросов — `http.NewRequestWithContext`.**
5. **Для БД — `db.QueryContext`.**

**❌ НЕ ДЕЛАЙ:**

6. **Не игнорируй `ctx.Done()`.** Если функция долгая — она должна проверять отмену.
7. **Не пытайся «убить» горутину.** В Go нет такого API.
8. **Не используй `time.Sleep` в отменяемом коде.** Используй `select` с `ctx.Done()`.

---

## 5.2 Context: интерфейс и контракт

Разберём, что такое `Context`.

### Интерфейс

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

**Четыре метода:**

| Метод | Что возвращает |
|:---|:---|
| `Deadline()` | Дедлайн (если есть) |
| `Done()` | Канал, закрывающийся при отмене |
| `Err()` | Причина отмены |
| `Value(key)` | Значение по ключу |

### Базовые контексты

**`context.Background()`:**

- Корневой контекст.
- Никогда не отменяется.
- Не имеет дедлайна.
- Используется в `main`, `init`, тестах, как база для `With*`.

**`context.TODO()`:**

- Аналогичен `Background()`.
- Используется, когда не ясно, какой контекст использовать (заглушка).

Разница — только в семантике.

### Производные контексты

**`WithCancel(parent)`** — возвращает `ctx, cancel`. `cancel()` отменяет `ctx` и всех потомков.

**`WithTimeout(parent, d)`** — `ctx` отменяется через `d` или при `cancel()`.

**`WithDeadline(parent, t)`** — `ctx` отменяется в `t` или при `cancel()`.

**`WithValue(parent, key, val)`** — `ctx.Value(key)` возвращает `val`.

### Контракт Context

**Правила использования:**

1. **`Context` — первый аргумент функции** (если функция его принимает).
2. **`Context` не хранится в структурах:**
   ```go
   // ❌ Плохо
   type Service struct {
       ctx context.Context
   }
   
   // ✅ Хорошо
   func (s *Service) Process(ctx context.Context) error
   ```
3. **`Context` не передаётся как `nil`:**
   ```go
   Process(context.Background())  // ✅
   // Process(nil)  // ❌
   ```
4. **`cancel()` всегда вызывается:**
   ```go
   ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
   defer cancel()
   ```
5. **`Context` — неизменяемый.** `With*` создаёт новый, не меняет родителя.
6. **`Context` потокобезопасен.** Можно передавать между горутинами.

### Дерево контекстов

```
context.Background()
    │
    ├── WithCancel → ctx1, cancel1
    │       │
    │       ├── WithTimeout(5s) → ctx2, cancel2
    │       │       │
    │       │       └── WithValue("user_id", 42) → ctx3
    │       │
    │       └── WithCancel → ctx4, cancel4
    │
    └── WithTimeout(10s) → ctx5, cancel5
```

**Свойства:**

- **Отмена родителя отменяет всех потомков.**
- **Отмена потомка не отменяет родителя.**
- **`Deadline` родителя наследуется потомками.**
- **`Value` родителя доступно потомкам.**

### 💡 Практика: как использовать Context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`Context` — первый аргумент.**
2. **`defer cancel()`** — всегда.
3. **`context.Background()` для корня.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`WithValue` только для request-scoped данных.**

**❌ НЕ ДЕЛАЙ:**

5. **Не храни `Context` в структуре.**
6. **Не передавай `nil`.**
7. **Не забывай `cancel()`.**

---

## 5.3 Дерево отмены: как устроен cancelCtx

Разберём внутреннее устройство `cancelCtx`.

### Структура cancelCtx

```go
type cancelCtx struct {
    Context                    // родитель (встроенный)
    
    mu       sync.Mutex
    done     atomic.Value      // chan struct{} — закрывается при отмене
    children map[canceler]struct{}  // потомки
    err      error             // причина отмены
}
```

**Поля:**

| Поле | Назначение |
|:---|:---|
| `Context` | Родитель |
| `mu` | Защита |
| `done` | Канал отмены |
| `children` | Потомки |
| `err` | Причина (`Canceled`, `DeadlineExceeded`) |

### Как создаётся cancelCtx

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    c := &cancelCtx{}
    c.Context = parent
    
    if p, ok := parentCancelCtx(parent); ok {
        p.mu.Lock()
        if p.err != nil {
            c.cancel(true, Canceled, nil)
        } else {
            p.children[c] = struct{}{}
        }
        p.mu.Unlock()
    }
    
    return c, func() { c.cancel(true, Canceled, nil) }
}
```

**Ключевое:** `WithCancel` **регистрируется у родителя**. Если родитель — `cancelCtx`, новый контекст добавляется в его `children`.

### Дерево отмены

```
context.Background()
    │
    ├── cancelCtx A (children: {B, D})
    │       │
    │       ├── cancelCtx B (children: {C})
    │       │       │
    │       │       └── cancelCtx C (children: {})
    │       │
    │       └── cancelCtx D (children: {})
    │
    └── cancelCtx E (children: {})
```

**Что происходит при отмене A:**

1. `A.cancel()` вызывается.
2. A закрывает свой `done`.
3. A проходит по `children` (B, D) и вызывает их `cancel()`.
4. B закрывает свой `done` и проходит по своим `children` (C).
5. C закрывает свой `done`.
6. D закрывает свой `done`.

**Результат:** все потомки A отменены.

### Как cancel работает

```go
func (c *cancelCtx) cancel(removeFromParent bool, err, cause error) {
    c.mu.Lock()
    
    if c.err != nil {
        c.mu.Unlock()
        return  // уже отменён
    }
    
    c.err = err
    close(c.done)
    
    for child := range c.children {
        child.cancel(false, err, cause)
    }
    c.children = nil
    
    c.mu.Unlock()
    
    if removeFromParent {
        removeChild(c.Context, c)
    }
}
```

**Ключевые шаги:**

1. **Проверка `c.err != nil`** — идемпотентность. Повторный `cancel()` — no-op.
2. **`close(c.done)`** — закрывает канал. Все, кто ждёт `<-ctx.Done()`, разблокируются.
3. **Рекурсивная отмена потомков.**
4. **Удаление из родителя.**

### Идемпотентность cancel

**`cancel()` можно вызывать несколько раз.** Повторные вызовы — no-op.

```go
ctx, cancel := context.WithCancel(context.Background())
cancel()
cancel()  // ничего не происходит
cancel()  // тоже
```

**Почему:** `c.err != nil` — если уже отменён, выходим. Это защита от паники при `close(c.done)` дважды.

### 💡 Практика: как использовать cancelCtx

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`defer cancel()`** — всегда.
2. **Помни: отмена родителя отменяет потомков.** Обратно — нет.

**❌ НЕ ДЕЛАЙ:**

3. **Не вызывай `cancel()` из горутины-потомка.** Только из создателя.
4. **Не забывай `cancel()`.** Даже если контекст с таймаутом.

---

## 5.4 WithCancel: ручная отмена

Разберём **ручную отмену** через `WithCancel`.

### Простой пример

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    go worker(ctx)
    
    time.Sleep(1 * time.Second)
    cancel()  // отменяем
    
    time.Sleep(100 * time.Millisecond)
}

func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("worker: canceled")
            return
        default:
            fmt.Println("worker: working")
            time.Sleep(100 * time.Millisecond)
        }
    }
}
```

**Что происходит:**

1. `WithCancel` создаёт `ctx` и `cancel`.
2. `worker` запускается, проверяет `ctx.Done()` в цикле.
3. Через 1 секунду вызывается `cancel()`.
4. `ctx.Done()` закрывается.
5. `worker` видит `<-ctx.Done()`, печатает `canceled`, возвращается.

### Паттерн: отмена нескольких горутин

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            worker(ctx, id)
        }(i)
    }
    
    time.Sleep(1 * time.Second)
    cancel()  // отменяем всех
    
    wg.Wait()
    fmt.Println("all workers stopped")
}
```

**Что происходит:**

1. 10 горутин запускаются с одним `ctx`.
2. Через 1 секунду `cancel()` отменяет все.
3. Все горутины видят `ctx.Done()` и завершаются.
4. `wg.Wait()` ждёт завершения.

### Паттерн: отмена по сигналу

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Отменяем при получении сигнала
    go func() {
        sigCh := make(chan os.Signal, 1)
        signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
        <-sigCh
        fmt.Println("received signal, canceling")
        cancel()
    }()
    
    // Работаем, пока не отменят
    worker(ctx)
}
```

**Что происходит:** `Ctrl+C` (SIGINT) вызывает `cancel()`, все горутины завершаются. Это основа **graceful shutdown**.

### Паттерн: вложенные контексты

```go
func main() {
    parent, parentCancel := context.WithCancel(context.Background())
    defer parentCancel()
    
    child1, child1Cancel := context.WithCancel(parent)
    defer child1Cancel()
    
    child2, child2Cancel := context.WithCancel(parent)
    defer child2Cancel()
    
    go worker("parent", parent)
    go worker("child1", child1)
    go worker("child2", child2)
    
    time.Sleep(1 * time.Second)
    child1Cancel()  // отменяет только child1
    
    time.Sleep(1 * time.Second)
    parentCancel()  // отменяет parent, child2
}
```

**Что происходит:**

1. `child1Cancel()` отменяет только `child1`. `parent` и `child2` продолжают.
2. `parentCancel()` отменяет `parent` и всех потомков (`child2`). `child1` уже отменён.

**Ключевое:** отмена **потомка** не отменяет **родителя**. Отмена **родителя** отменяет всех **потомков**.

### 💡 Практика: как использовать WithCancel

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`defer cancel()`** — даже если отменяешь вручную.
2. **Передавай `ctx` во все горутины**, которые должны отменяться.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Для отмены по сигналу — `signal.Notify` + `cancel()`.**
4. **Для вложенных контекстов — отменяй родителя, чтобы отменить всех.**

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай `cancel()`.**
6. **Не отменяй родителя, если хочешь отменить только потомка.**

---

## 5.5 WithTimeout и WithDeadline

Разберём **отмену по времени**.

### WithTimeout vs WithDeadline

**`WithTimeout(parent, d)`** — отмена через `d` от **текущего момента**.

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
```

**`WithDeadline(parent, t)`** — отмена в **абсолютный момент** `t`.

```go
deadline := time.Now().Add(5 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
```

**Связь:** `WithTimeout(parent, d)` эквивалентно `WithDeadline(parent, time.Now().Add(d))`.

### Структура timerCtx

```go
type timerCtx struct {
    cancelCtx
    timer    *time.Timer
    deadline time.Time
}
```

**`timerCtx`** — надстройка над `cancelCtx`. Добавляет таймер и дедлайн.

### Как работает WithTimeout

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        return WithCancel(parent)  // родитель раньше — используем его
    }
    
    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  d,
    }
    
    propagateCancel(parent, c)
    
    dur := time.Until(d)
    if dur <= 0 {
        c.cancel(true, DeadlineExceeded, nil)
        return c, func() { c.cancel(false, Canceled, nil) }
    }
    
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.err == nil {
        c.timer = time.AfterFunc(dur, func() {
            c.cancel(true, DeadlineExceeded, nil)
        })
    }
    
    return c, func() { c.cancel(true, Canceled, nil) }
}
```

**Ключевые шаги:**

1. **Проверка родителя:** если у родителя более ранний дедлайн — используем его.
2. **Создание `timerCtx`.**
3. **Проверка дедлайна:** если уже истёк — отменяем сразу.
4. **Запуск таймера:** `time.AfterFunc` вызывает `cancel` через `dur`.

### Что происходит при таймауте

```
t=0:     WithTimeout(ctx, 5s)
         - Создан timerCtx
         - Запущен таймер на 5 секунд

t=0..5s: Горутина работает
         - ctx.Done() открыт
         - ctx.Err() == nil

t=5s:    Таймер срабатывает
         - c.cancel(true, DeadlineExceeded, nil)
         - close(ctx.Done())
         - ctx.Err() == DeadlineExceeded

t=5s+:   Горутина видит <-ctx.Done()
         - select case <-ctx.Done()
         - return ctx.Err()
```

### Важно: cancel() освобождает таймер

**`cancel()`** не только отменяет контекст, но и **останавливает таймер**:

```go
func (c *timerCtx) cancel(removeFromParent bool, err, cause error) {
    c.cancelCtx.cancel(false, err, cause)
    if removeFromParent {
        removeChild(c.cancelCtx.Context, c)
    }
    
    c.mu.Lock()
    if c.timer != nil {
        c.timer.Stop()  // освобождаем таймер
        c.timer = nil
    }
    c.mu.Unlock()
}
```

**Почему это важно:** без `cancel()` таймер остаётся в памяти до срабатывания. Если контекстов много — утечка.

**Правило:** `defer cancel()` — обязательно, даже если контекст с таймаутом.

### Пример: HTTP-запрос с таймаутом

```go
func fetchUser(ctx context.Context, id int) (*User, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    req, err := http.NewRequestWithContext(ctx, "GET", 
        fmt.Sprintf("https://api.example.com/users/%d", id), nil)
    if err != nil {
        return nil, err
    }
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err  // context.DeadlineExceeded
    }
    defer resp.Body.Close()
    
    var user User
    if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
        return nil, err
    }
    return &user, nil
}
```

**Что происходит:**

1. `WithTimeout(ctx, 5s)` — создаёт контекст с таймаутом.
2. `http.Do` видит `ctx.Done()` — если таймаут истёк, `Do` вернёт `DeadlineExceeded`.
3. `defer cancel()` — освобождает таймер, даже если запрос завершился раньше.

### Паттерн: вложенные таймауты

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Общий таймаут на весь handler
    ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
    defer cancel()
    
    // Отдельный таймаут на БД
    dbCtx, dbCancel := context.WithTimeout(ctx, 2*time.Second)
    defer dbCancel()
    
    // Отдельный таймаут на внешний API
    apiCtx, apiCancel := context.WithTimeout(ctx, 5*time.Second)
    defer apiCancel()
    
    user, err := db.GetUser(dbCtx, 42)
    profile, err := api.GetProfile(apiCtx, user.ID)
}
```

**Что происходит:**

- **Родительский `ctx`** — 10 секунд на всё.
- **`dbCtx`** — 2 секунды.
- **`apiCtx`** — 5 секунд.

**Если `ctx` отменится раньше** — все потомки отменятся.

### 💡 Практика: как использовать таймауты

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`defer cancel()`** — даже если контекст с таймаутом. Иначе таймер утечёт.
2. **Для HTTP — `http.NewRequestWithContext`.**
3. **Для БД — `db.QueryContext`.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Вложенные таймауты** — общий + на каждую операцию.
5. **`WithDeadline`** — если есть абсолютный момент.

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `cancel()`.**
7. **Не используй `time.After` вместо `Context`.**
8. **Не ставь таймаут больше, чем у родителя.**

---

## 5.6 WithValue: что это и почему его не любят

Разберём `WithValue` — самый спорный метод `Context`.

### Что делает WithValue

```go
ctx = context.WithValue(ctx, "user_id", 42)
userID := ctx.Value("user_id").(int)
```

**Что происходит:** создаётся `valueCtx`, который хранит пару `(key, value)`. При `Value(key)` ищется по цепочке контекстов.

### Структура valueCtx

```go
type valueCtx struct {
    Context              // родитель
    key, val any
}

func (c *valueCtx) Value(key any) any {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)
}
```

**Поиск:** `Value(key)` идёт по цепочке **от текущего к корню**.

### Почему WithValue не любят

**1. Нетипизированные ключи.**

```go
ctx = context.WithValue(ctx, "user_id", 42)  // key — строка
userID := ctx.Value("user_id").(int)         // type assertion
```

**Проблемы:**

- **Ключ — `any`.** Легко ошибиться: `"user_id"` vs `"userId"`.
- **Значение — `any`.** Нужен type assertion, который может паниковать.
- **Нет проверки на этапе компиляции.**

**2. Неявные зависимости.**

```go
func process(ctx context.Context) error {
    userID := ctx.Value("user_id").(int)  // откуда?
}
```

**Проблема:** функция `process` зависит от `user_id`, но это **не видно в сигнатуре**. Легко вызвать без `user_id` — паника.

**3. «Магические» данные.**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    userID := r.Context().Value("user_id").(int)  // где установлено?
}
```

**Проблема:** неясно, **где** и **кем** установлен `user_id`.

**4. Проблемы с типизацией.**

```go
ctx = context.WithValue(ctx, "count", 42)
count := ctx.Value("count").(int)     // OK
count := ctx.Value("count").(int64)   // panic!
```

### Правильный способ: типизированные ключи

**Плохо:**

```go
ctx = context.WithValue(ctx, "user_id", 42)
userID := ctx.Value("user_id").(int)
```

**Хорошо:**

```go
type contextKey int

const (
    userIDKey contextKey = iota
)

func WithUserID(ctx context.Context, id int) context.Context {
    return context.WithValue(ctx, userIDKey, id)
}

func UserID(ctx context.Context) (int, bool) {
    id, ok := ctx.Value(userIDKey).(int)
    return id, ok
}

ctx = WithUserID(ctx, 42)
if id, ok := UserID(ctx); ok {
    // ...
}
```

**Что это даёт:**

- **Типизированный ключ** — нельзя перепутать.
- **Функции-обёртки** — скрывают type assertion.
- **Безопасность** — `ok` вместо паники.

### Когда WithValue нужен

**Только для request-scoped данных:**

- **Trace ID** (для распределённого трейсинга).
- **Request ID** (для корреляции логов).
- **User ID** (после аутентификации).
- **Correlation ID.**

**Что НЕ должно:**

- **Опциональные параметры** (page_size, limit).
- **Конфигурация** (db_host).
- **Зависимости** (logger, db).
- **Обязательные параметры** (user_id, если он нужен всем).

### 💡 Практика: как использовать WithValue

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Типизированные ключи:**
   ```go
   type contextKey int
   const userIDKey contextKey = iota
   ```

2. **Функции-обёртки:**
   ```go
   func WithUserID(ctx context.Context, id int) context.Context
   func UserID(ctx context.Context) (int, bool)
   ```

**👍 СТОИТ СДЕЛАТЬ:**

3. **Только для request-scoped данных.**

**❌ НЕ ДЕЛАЙ:**

4. **Не используй строковые ключи.**
5. **Не передавай конфигурацию, зависимости, опциональные параметры.**
6. **Не создавай длинные цепочки `WithValue`.**

---

## 5.7 Как работает отмена: Done(), Err(), распространение

Разберём **механику отмены**.

### Done() — канал отмены

```go
select {
case <-ctx.Done():
    return ctx.Err()
case result := <-resultCh:
    return result
}
```

**`ctx.Done()`** возвращает `<-chan struct{}`:

- **Открыт** — контекст активен.
- **Закрыт** — контекст отменён.

**Закрытие канала** — сигнал **всем** горутинам, которые ждут `<-ctx.Done()`. Это **broadcast** (как `close` в Главе 2).

### Err() — причина отмены

```go
if err := ctx.Err(); err != nil {
    // Canceled или DeadlineExceeded
}
```

| Значение | Когда |
|:---|:---|
| `nil` | Контекст не отменён |
| `context.Canceled` | Вызван `cancel()` |
| `context.DeadlineExceeded` | Истёк таймаут/дедлайн |

**Важно:** `Err()` возвращает **не nil** только **после** закрытия `Done()`.

### Распространение отмены

```
parent (cancelCtx)
    │
    ├── child1 (cancelCtx)
    │       │
    │       └── grandchild1 (cancelCtx)
    │
    └── child2 (cancelCtx)

parent.cancel():
    1. parent.err = Canceled
    2. close(parent.done)
    3. Для каждого child:
       - child1.cancel():
         - child1.err = Canceled
         - close(child1.done)
         - grandchild1.cancel():
           - grandchild1.err = Canceled
           - close(grandchild1.done)
       - child2.cancel():
         - child2.err = Canceled
         - close(child2.done)
```

**Ключевое:** отмена **рекурсивна**. Родитель отменяет потомков.

### Что происходит с горутинами

**Горутина, которая ждёт `<-ctx.Done()`:**

```go
func worker(ctx context.Context) {
    select {
    case <-ctx.Done():
        return  // разблокируется при отмене
    case task := <-tasks:
        // ...
    }
}
```

**Что происходит при отмене:**

1. `ctx.Done()` закрывается.
2. `select` видит готовый case `<-ctx.Done()`.
3. Горутина выполняет `return`.
4. `defer` выполняются.
5. Горутина завершается.

**Горутина, которая НЕ ждёт `<-ctx.Done()`:**

```go
func badWorker(ctx context.Context) {
    for i := 0; i < 1_000_000_000; i++ {
        // долгий цикл, не проверяет ctx
    }
}
```

**Что происходит при отмене:** **ничего**. Горутина продолжит работу.

**Вывод:** горутина должна **проверять** `ctx.Done()`.

### Как проверять отмену

**1. В `select`:**

```go
select {
case <-ctx.Done():
    return ctx.Err()
case result := <-resultCh:
    return result
}
```

**2. В цикле:**

```go
for {
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
    }
    
    if err := doWork(); err != nil {
        return err
    }
}
```

**3. Перед долгой операцией:**

```go
if err := ctx.Err(); err != nil {
    return err
}
```

**4. Во внешних вызовах — передавай `ctx`:**

```go
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
resp, err := http.DefaultClient.Do(req)
```

### 💡 Практика: как проверять отмену

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **В `select` — `case <-ctx.Done()`.**
2. **В долгих циклах — проверка.**
3. **Передавай `ctx` во внешние вызовы.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Проверять `ctx.Err()` после `<-ctx.Done()`** — для диагностики.

**❌ НЕ ДЕЛАЙ:**

5. **Не игнорируй `ctx.Done()` в долгих операциях.**
6. **Не используй `time.Sleep` в отменяемом коде.**

---

## 5.8 Context и happens-before

Разберём связь `Context` с happens-before (Глава 7).

### Context устанавливает happens-before

**`close(ctx.Done())` → `<-ctx.Done()`:**

`close` канала **happens-before** получения zero-значения из закрытого канала.

```go
var data string

// Горутина A:
data = "hello"     // (1)
cancel()           // (2) закрывает ctx.Done()

// Горутина B:
<-ctx.Done()       // (3) получает zero
fmt.Println(data)  // (4)
```

**Что гарантируется:**

- (2) happens-before (3) — close → recv.
- (1) happens-before (4) — транзитивно.
- **Результат:** B увидит `data == "hello"`.

**Важно:** это работает, потому что `cancel()` **закрывает** `ctx.Done()`, а не отправляет в него. Закрытие канала — broadcast с happens-before.

### Context и race detector

**Race detector** понимает `Context`:

```go
var data string

// Горутина A:
data = "hello"
cancel()

// Горутина B:
<-ctx.Done()
fmt.Println(data)
```

**Race detector:** **не найдёт** гонку. Потому что `close` → `recv` устанавливает happens-before.

### Context и Acquire/Release

**`cancel()`** — это **release** операция. **`<-ctx.Done()`** — **acquire** операция.

```
cancel() (release):
  - Все записи до cancel видны после <-Done()
  - Аналог release барьера

<-ctx.Done() (acquire):
  - Все записи до cancel видны после получения
  - Аналог acquire барьера
```

**Связь с `Mutex`:** `Unlock` (release) → `Lock` (acquire). Аналогично `cancel` → `<-Done()`.

### 💡 Практика: как использовать happens-before Context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Помни: `cancel()` → `<-ctx.Done()` устанавливает happens-before.**

2. **Не нужна дополнительная синхронизация** для данных, переданных через `Context`.

**❌ НЕ ДЕЛАЙ:**

3. **Не полагайся на happens-before без `Context`.** Обычные переменные — race.

---

## 5.9 Context и другие примитивы

Разберём, как комбинировать `Context` с другими примитивами.

### Context и select

**Основной паттерн:**

```go
select {
case <-ctx.Done():
    return ctx.Err()
case result := <-resultCh:
    return result
case <-time.After(5 * time.Second):
    return errors.New("timeout")
}
```

**Что происходит:**

1. Если `ctx.Done()` закрыт — отмена.
2. Если `resultCh` готов — результат.
3. Если таймаут — ошибка.

### Context и Mutex

**❌ Плохо: держать мьютекс при отмене**

```go
func process(ctx context.Context) error {
    mu.Lock()
    defer mu.Unlock()
    
    select {
    case <-ctx.Done():
        return ctx.Err()
    case result := <-resultCh:
        return result
    }
}
```

**Проблема:** если `resultCh` долго не готов, мьютекс держится **всё время ожидания**.

**✅ Хорошо: не держать мьютекс при долгом ожидании**

```go
func process(ctx context.Context) error {
    mu.Lock()
    data := getData()
    mu.Unlock()
    
    select {
    case <-ctx.Done():
        return ctx.Err()
    case result := <-resultCh:
        mu.Lock()
        updateData(result)
        mu.Unlock()
        return nil
    }
}
```

**Что изменилось:** мьютекс держится только на время работы с данными.

### Context и WaitGroup

**Паттерн: отмена + ожидание**

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            worker(ctx, id)
        }(i)
    }
    
    time.Sleep(1 * time.Second)
    cancel()  // отменяем всех
    wg.Wait() // ждём завершения
}
```

**Что происходит:**

1. 10 горутин запускаются с `ctx`.
2. `cancel()` отменяет все.
3. `wg.Wait()` ждёт завершения.

**Важно:** `WaitGroup` **не** отменяется. Только `Context`.

### Context и time.After

**❌ Плохо: `time.After` вместо `Context`**

```go
select {
case result := <-resultCh:
    return result
case <-time.After(5 * time.Second):
    return errors.New("timeout")
}
```

**Проблема:** `time.After` **не отменяет** операцию. Если `resultCh` не готов, горутина продолжит ждать.

**✅ Хорошо: `Context` с таймаутом**

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()

select {
case result := <-resultCh:
    return result
case <-ctx.Done():
    return ctx.Err()
}
```

### Context и sync.Once

**Проблема:** `once.Do` **не отменяется**. Если `connect(ctx)` завис, `once.Do` заблокирует всех.

**Решение:** использовать `Mutex` + флаг:

```go
type Service struct {
    mu   sync.Mutex
    conn *Connection
    done bool
}

func (s *Service) GetConn(ctx context.Context) (*Connection, error) {
    s.mu.Lock()
    if s.done {
        conn := s.conn
        s.mu.Unlock()
        return conn, nil
    }
    s.mu.Unlock()
    
    conn, err := connect(ctx)
    
    s.mu.Lock()
    s.conn = conn
    s.done = true
    s.mu.Unlock()
    
    return conn, err
}
```

### 💡 Практика: как комбинировать Context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`select` с `ctx.Done()`** — основной паттерн.
2. **Не держи мьютекс при долгом ожидании.**

**👍 СТОИТ СДЕЛАТЬ:**

3. **`cancel()` + `wg.Wait()`** — для массовой отмены.
4. **Для ленивой инициализации с отменой — `Mutex` + флаг.**

**❌ НЕ ДЕЛАЙ:**

5. **Не используй `time.After` вместо `Context`.**
6. **Не используй `once.Do` для отменяемой инициализации.**

---

## 5.10 Практика Go: диагностика Context

Разберём **утечку через Context и как её найти**.

### Утечка горутин без cancel

```go
package main

import (
    "context"
    "fmt"
    "runtime"
    "time"
)

func main() {
    fmt.Printf("goroutines before: %d\n", runtime.NumGoroutine())
    
    // ❌ Утечка: cancel не вызывается
    for i := 0; i < 1000; i++ {
        ctx, _ := context.WithCancel(context.Background())  // cancel потерян!
        go worker(ctx)
    }
    
    time.Sleep(100 * time.Millisecond)
    fmt.Printf("goroutines after: %d\n", runtime.NumGoroutine())
}

func worker(ctx context.Context) {
    <-ctx.Done()  // ждёт вечно
}
```

**Пример вывода:**

```
goroutines before: 1
goroutines after: 1001
```

**Что видно:** 1000 горутин утекли, потому что `cancel` не вызывается.

**Исправление:**

```go
for i := 0; i < 1000; i++ {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()  // ✅
    go worker(ctx)
}
```

### Отмена через WithCancel

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            worker(ctx, id)
        }(i)
    }
    
    time.Sleep(500 * time.Millisecond)
    fmt.Println("canceling...")
    cancel()
    
    wg.Wait()
    fmt.Println("all workers stopped")
}

func worker(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("worker %d: %v\n", id, ctx.Err())
            return
        default:
            time.Sleep(100 * time.Millisecond)
        }
    }
}
```

**Пример вывода:**

```
worker 0: context canceled
worker 1: context canceled
...
canceling...
worker 9: context canceled
all workers stopped
```

**Что видно:** все 10 горутин завершились после `cancel()`.

### Таймаут через WithTimeout

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
    defer cancel()
    
    start := time.Now()
    err := slowOperation(ctx)
    fmt.Printf("elapsed: %v, err: %v\n", time.Since(start), err)
}

func slowOperation(ctx context.Context) error {
    select {
    case <-time.After(2 * time.Second):
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

**Пример вывода:**

```
elapsed: 500ms, err: context deadline exceeded
```

**Что видно:** операция прервалась через 500 мс, а не через 2 секунды.

### Дерево контекстов

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    parent, parentCancel := context.WithCancel(context.Background())
    defer parentCancel()
    
    child1, child1Cancel := context.WithCancel(parent)
    defer child1Cancel()
    
    child2, child2Cancel := context.WithCancel(parent)
    defer child2Cancel()
    
    go worker("parent", parent)
    go worker("child1", child1)
    go worker("child2", child2)
    
    time.Sleep(500 * time.Millisecond)
    fmt.Println("canceling child1...")
    child1Cancel()
    
    time.Sleep(500 * time.Millisecond)
    fmt.Println("canceling parent...")
    parentCancel()
    
    time.Sleep(500 * time.Millisecond)
}

func worker(name string, ctx context.Context) {
    <-ctx.Done()
    fmt.Printf("%s: %v\n", name, ctx.Err())
}
```

**Пример вывода:**

```
canceling child1...
child1: context canceled
canceling parent...
parent: context canceled
child2: context canceled
```

**Что видно:**

- `child1Cancel()` отменил только `child1`.
- `parentCancel()` отменил `parent` и `child2`.

### Мониторинг горутин

```go
package main

import (
    "context"
    "fmt"
    "runtime"
    "time"
)

func main() {
    // Мониторинг горутин
    go func() {
        for range time.Tick(100 * time.Millisecond) {
            fmt.Printf("goroutines: %d\n", runtime.NumGoroutine())
        }
    }()
    
    // ✅ Хорошо: cancel вызывается
    for i := 0; i < 100; i++ {
        ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
        go func() {
            defer cancel()
            <-ctx.Done()
        }()
    }
    
    time.Sleep(200 * time.Millisecond)
    fmt.Println("done")
}
```

**Пример вывода:**

```
goroutines: 102
goroutines: 3
goroutines: 2
done
```

**Что видно:** горутины создаются, потом завершаются. Утечки нет.

### 💡 Практика: как диагностировать Context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`defer cancel()`** — всегда.
2. **Мониторь `runtime.NumGoroutine()`.**
3. **pprof goroutine dump** при подозрении.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`goleak` в тестах** — ловит утечки.

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай `cancel()`.**
6. **Не игнорируй растущее число горутин.**

---

## 5.11 Выводы и типичные ошибки

**Что мы узнали?**

Горутину нельзя «убить» — она может держать ресурсы. Отмена — **кооперация**: горутина сама проверяет `ctx.Done()`. `Context` — интерфейс с четырьмя методами: `Deadline`, `Done`, `Err`, `Value`. `Background()` — корень, `TODO()` — заглушка. `WithCancel`, `WithTimeout`, `WithDeadline`, `WithValue` — производные. `cancelCtx` — структура с `done`, `children`, `err`. Отмена **рекурсивна**: родитель отменяет потомков. `cancel()` **идемпотентен**. `WithTimeout` использует `timerCtx` с таймером; `cancel()` **останавливает таймер**. `WithValue` — для request-scoped данных; используй типизированные ключи. `cancel()` → `<-ctx.Done()` устанавливает happens-before. `Context` комбинируется с `select`, `Mutex`, `WaitGroup`.

**Типичные ошибки:**

- ❌ **Забывать `cancel()`.** Утечка горутин и таймеров.
- ❌ **Использовать `context.Background()` в HTTP-хэндлере.** Используй `r.Context()`.
- ❌ **Хранить `Context` в структуре.** Передавай явно.
- ❌ **Передавать `nil` как `Context`.** Используй `Background()` или `TODO()`.
- ❌ **Игнорировать `ctx.Done()` в долгих операциях.**
- ❌ **Использовать `time.Sleep` в отменяемом коде.**
- ❌ **Использовать строковые ключи в `WithValue`.**
- ❌ **Передавать через `Context` опциональные параметры.**
- ❌ **Использовать `time.After` вместо `Context`.**
- ❌ **Держать `Mutex` при долгом ожидании.**
- ❌ **Использовать `once.Do` для отменяемой инициализации.**

---

## 5.12 Для быстрого повторения

- **Горутину нельзя убить.** Отмена — кооперация.
- **`Context` — интерфейс:** `Deadline`, `Done`, `Err`, `Value`.
- **`Background()`** — корень. **`TODO()`** — заглушка.
- **`WithCancel`, `WithTimeout`, `WithDeadline`, `WithValue`** — производные.
- **`cancelCtx`** — `done` (канал), `children` (потомки), `err` (причина).
- **Отмена рекурсивна:** родитель отменяет потомков. Обратно — нет.
- **`cancel()` идемпотентен.**
- **`WithTimeout`** — `timerCtx` с таймером. **`cancel()` останавливает таймер.**
- **`WithValue`** — request-scoped данные. Типизированные ключи.
- **`cancel()` → `<-ctx.Done()`** устанавливает happens-before.
- **`select` с `ctx.Done()`** — основной паттерн.
- **`Mutex`** — освобождай до `select`.
- **`WaitGroup`** — `cancel()` + `wg.Wait()`.
- **`time.After`** не отменяет — используй `Context`.
- **`defer cancel()`** — всегда.

---

## 5.13 Вопросы для самопроверки

1. Почему нельзя «убить» горутину? Что использовать вместо этого?
2. Что такое кооперативная отмена?
3. Какие четыре метода у `Context`?
4. Что такое дерево контекстов? Что происходит при отмене родителя?
5. Что делает `cancel()` с таймером в `WithTimeout`?
6. Почему `WithValue` не любят? Как использовать правильно?
7. Как `Context` связан с happens-before?

---

## 5.14 Ответы

### Ответ 1

**Нельзя убить горутину**, потому что она может держать ресурсы: мьютексы, транзакции, файлы, каналы. Принудительное завершение оставит систему в несогласованном состоянии.

**Вместо этого:** `context.Context`. Горутина **сама** проверяет `ctx.Done()` и завершается корректно.

### Ответ 2

**Кооперативная отмена** — горутина сама решает, когда завершиться, проверяя сигнал отмены.

```go
for {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case task := <-tasks:
        process(task)
    }
}
```

Горутина видит `<-ctx.Done()`, возвращает управление, `defer` выполняются.

### Ответ 3

**Четыре метода:**
1. `Deadline()` — дедлайн.
2. `Done()` — канал отмены.
3. `Err()` — причина отмены.
4. `Value(key)` — значение по ключу.

### Ответ 4

**Дерево контекстов** — иерархия, где каждый `With*` создаёт потомка.

**При отмене родителя:** все потомки отменяются рекурсивно.

**Обратно:** отмена потомка **не** отменяет родителя.

### Ответ 5

**`cancel()` останавливает таймер** (`c.timer.Stop()`). Без `cancel()` таймер останется в памяти до срабатывания. Если контекстов много — утечка.

**Поэтому `defer cancel()` обязателен.**

### Ответ 6

**`WithValue` не любят**, потому что:
- Нетипизированные ключи (`any`).
- Неявные зависимости.
- «Магические» данные.
- Проблемы с типизацией.

**Правильно:** типизированные ключи + функции-обёртки. Только для request-scoped данных.

### Ответ 7

**`cancel()` → `<-ctx.Done()`** устанавливает happens-before. Аналог `close(ch)` → `<-ch`.

- `cancel()` — release.
- `<-ctx.Done()` — acquire.

Все записи до `cancel()` видны после `<-Done()`.

---

## 5.15 Куда идти дальше?

Мы разобрали `Context`: отмену, таймауты, дедлайны, дерево отмены, happens-before. Теперь мы умеем **корректно отменять** операции.

Но остаётся **фундаментальный вопрос**: как **планировщик** Go выбирает, какую горутину выполнять? Как работает G-M-P модель? Что такое work stealing?

- **Как планировщик выбирает горутину?** → **Глава 6: Планировщик Go — G-M-P, work stealing, preemption.**
- **Почему data race — undefined behavior?** → **Глава 7: Memory model и data race.**

---

## 5.16 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **`Context`** | Интерфейс отмены | `Deadline`, `Done`, `Err`, `Value` |
| **`Background()`** | Корневой контекст | Никогда не отменяется |
| **`TODO()`** | Заглушка | Аналог `Background()` |
| **`WithCancel`** | Ручная отмена | `ctx, cancel` |
| **`WithTimeout`** | Отмена через `d` | `timerCtx` с таймером |
| **`WithDeadline`** | Отмена в момент `t` | Аналог `WithTimeout` |
| **`WithValue`** | Request-scoped данные | Типизированные ключи |
| **`cancelCtx`** | Структура отмены | `done`, `children`, `err` |
| **`timerCtx`** | Структура с таймером | `cancelCtx` + `timer` + `deadline` |
| **`cancel()`** | Отмена | Идемпотентен. Останавливает таймер |
| **`<-ctx.Done()`** | Проверка отмены | Закрывается при отмене. Broadcast |
| **`ctx.Err()`** | Причина отмены | `Canceled`, `DeadlineExceeded` |
| **Дерево отмены** | Иерархия контекстов | Родитель отменяет потомков |
| **Happens-before** | `cancel()` → `<-Done()` | Как `close(ch)` → `<-ch` |
| **`select` с `ctx.Done()`** | Основной паттерн | Отмена + результат |
| **`defer cancel()`** | Обязательно | Утечка без него |

🧭 **Ключевая идея:** Горутину нельзя «убить» — она может держать ресурсы. Отмена — **кооперация**: горутина сама проверяет `ctx.Done()`. `Context` — интерфейс с четырьмя методами. `Background()` — корень, `TODO()` — заглушка. `WithCancel`, `WithTimeout`, `WithDeadline`, `WithValue` — производные. `cancelCtx` — `done`, `children`, `err`. Отмена **рекурсивна**: родитель отменяет потомков. `cancel()` **идемпотентен** и **останавливает таймер**. `WithValue` — только для request-scoped данных. `cancel()` → `<-ctx.Done()` устанавливает happens-before (как `close` канала). Комбинируется с `select`, `Mutex`, `WaitGroup`. **`defer cancel()` — всегда.**