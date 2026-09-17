# 🔌 Глава 5: Context — отмена, таймауты, дедлайны

**Что вы узнаете:**
- Почему «убить горутину» нельзя, и что делать вместо этого.
- Что такое `context.Context` и какие проблемы он решает.
- Как устроен `Context` внутри: `cancelCtx`, `timerCtx`, `valueCtx`, дерево отмены.
- Как работает `WithCancel`, `WithTimeout`, `WithDeadline`.
- Почему `WithValue` — не для передачи параметров, и когда он всё-таки нужен.
- Что происходит при отмене: `Done()`, `Err()`, распространение по дереву.
- Как связаны `Context` и каналы, `Context` и `happens-before` (Глава 4).
- Как не допустить утечек горутин через `Context`.
- Как правильно комбинировать `Context` с `Mutex`, `WaitGroup`, `select`.
- Как использовать `Context` в HTTP, БД, gRPC, Kafka.

**После прочтения вы сможете:**
- Объяснить, почему отмена — это **кооперация**, а не «убийство».
- Проектировать API, который принимает `context.Context` первым аргументом.
- Понимать, что происходит внутри `cancelCtx` при отмене.
- Правильно использовать `WithTimeout` и `WithDeadline` без утечек таймеров.
- Избегать анти-паттернов `WithValue`.
- Диагностировать утечки горутин, связанные с `Context`.
- Писать код, который корректно отменяется на всех уровнях.

---

## Содержание

- [5.0 Пролог: горутина, которую нельзя убить](#50-пролог-горутина-которую-нельзя-убить)
- [5.1 Почему нельзя убить горутину](#51-почему-нельзя-убить-горутину)
- [5.2 Context: интерфейс и контракт](#52-context-интерфейс-и-контракт)
- [5.3 Дерево отмены: как устроен cancelCtx](#53-дерево-отмены-как-устроен-cancelctx)
- [5.4 WithCancel: ручная отмена](#54-withcancel-ручная-отмена)
- [5.5 WithTimeout и WithDeadline: отмена по времени](#55-withtimeout-и-withdeadline-отмена-по-времени)
- [5.6 WithValue: что это и почему его не любят](#56-withvalue-что-это-и-почему-его-не-любят)
- [5.7 Как работает отмена: Done(), Err(), распространение](#57-как-работает-отмена-done-err-распространение)
- [5.8 Context и happens-before: связь с Главой 4](#58-context-и-happens-before-связь-с-главой-4)
- [5.9 Context и другие примитивы: select, Mutex, WaitGroup](#59-context-и-другие-примитивы-select-mutex-waitgroup)
- [5.10 Context в реальных сценариях: HTTP, БД, gRPC, Kafka](#510-context-в-реальных-сценариях-http-бд-grpc-kafka)
- [5.11 Практика Go: диагностика Context](#511-практика-go-диагностика-context)
- [5.12 Выводы и типичные ошибки](#512-выводы-и-типичные-ошибки)
- [5.13 Для быстрого повторения](#513-для-быстрого-повторения)
- [5.14 Вопросы для самопроверки](#514-вопросы-для-самопроверки)
- [5.15 Ответы](#515-ответы)
- [5.16 Куда идти дальше?](#516-куда-идти-дальше)
- [5.17 Чек-лист](#517-чек-лист)

---

## 5.0 Пролог: горутина, которую нельзя убить

Ты пишешь сервис, который делает HTTP-запрос к внешнему API:

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

Всё работает. Пока внешний API не начинает **тормозить**. Запрос висит 30 секунд. Ты хочешь его **отменить** — но `http.Get` не даёт такой возможности.

Ты пытаешься «убить» горутину:

```go
go fetchUser(42)
// ... через 5 секунд ...
// как убить горутину? 
```

И обнаруживаешь: **в Go нет способа убить горутину извне**.

❓ **Почему?** Горутина может держать мьютексы, писать в каналы, находиться в середине транзакции. Если её «убить» — можно оставить систему в несогласованном состоянии.

💡 **Решение:** горутина должна **сама** решить, когда завершиться. Ей нужен **сигнал**: «отменись». Этот сигнал — `context.Context`.

```go
func fetchUser(ctx context.Context, id int) (*User, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", 
        fmt.Sprintf("https://api.example.com/users/%d", id), nil)
    if err != nil {
        return nil, err
    }
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err  // сюда придёт context.DeadlineExceeded
    }
    defer resp.Body.Close()
    
    var user User
    if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
        return nil, err
    }
    return &user, nil
}

// Использование:
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
user, err := fetchUser(ctx, 42)
```

Теперь через 5 секунд `http.Do` **сам** вернёт ошибку `context.DeadlineExceeded`. Горутина завершится корректно.

❓ **Но как это работает?** `http.Do` не «убивает» горутину — он **проверяет** `ctx.Done()` и **сам** решает прервать работу. Это **кооперация**, а не принуждение.

> **Важный мост к будущим главам:** `Context` — это надстройка над каналами (Глава 2) и `happens-before` (Глава 4). Он использует канал `Done()` для сигнала отмены. Понимание `Context` критично для worker pool (Глава 7), pipeline (Глава 8), graceful shutdown (Глава 11).

---

## 5.1 Почему нельзя убить горутину

Прежде чем разбирать `Context`, нужно понять, **почему** в Go нет `runtime.KillGoroutine(g)`.

### Проблема: горутина может держать ресурсы

Представь, что горутина выполняет:

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

### Что было бы, если бы «убийство» было

**Гипотетический `runtime.KillGoroutine(g)`:**

```go
// ❌ Не существует в Go:
runtime.KillGoroutine(g)
```

**Проблемы:**

1. **Мьютексы остаются захваченными.**
2. **Каналы остаются в неизвестном состоянии.**
3. **`defer` не выполняется** — ресурсы не освобождаются.
4. **Транзакции не откатываются.**
5. **Стеки не очищаются корректно.**

**Именно поэтому** Go не даёт такого API. **Разработчик должен сам** обеспечить корректное завершение.

### Кооперативная отмена

**Идея:** горутина **сама** проверяет сигнал отмены и завершается корректно.

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

**Принудительное завершение** — как **выдернуть провод** из работающего компьютера. Быстро, но данные потеряны, файлы повреждены.

**Кооперативная отмена** — как **нажать кнопку «Сохранить и выключить»**. Компьютер завершает работу корректно.

**Go выбрал второе.** `Context` — это кнопка.

### Что значит «отмена — это кооперация»

**Три следствия:**

1. **Горутина должна проверять `ctx.Done()`.** Если не проверяет — отмена не сработает.
2. **Горутина должна завершаться быстро.** Если она в середине долгой операции — отмена не мгновенна.
3. **Горутина должна освобождать ресурсы.** `defer` выполняются, если горутина завершается корректно.

**Пример: горутина, которая НЕ проверяет `ctx.Done()`:**

```go
func bad(ctx context.Context) {
    for i := 0; i < 1_000_000_000; i++ {
        // долгий цикл без проверки ctx
        _ = i * i
    }
    // ctx.Done() никогда не проверяется → отмена не сработает
}
```

**Даже если** `cancel()` вызван — горутина продолжит работу. Потому что она **не проверяет** сигнал.

**Пример: горутина, которая проверяет:**

```go
func good(ctx context.Context) {
    for i := 0; i < 1_000_000_000; i++ {
        select {
        case <-ctx.Done():
            return  // отмена
        default:
            // продолжаем
        }
        _ = i * i
    }
}
```

**Теперь** `cancel()` сработает — горутина увидит `ctx.Done()` и завершится.

### Аннотация сложности

| Аспект | Принудительное завершение | Кооперативная отмена |
|:---|:---|:---|
| Скорость | Мгновенно | Зависит от проверок |
| Корректность | ❌ Ресурсы не освобождаются | ✅ `defer` выполняются |
| Deadlock | ✅ Возможен | ❌ Нет |
| Согласованность | ❌ Нарушается | ✅ Сохраняется |
| В Go | ❌ Не существует | ✅ `Context` |

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

4. **Для HTTP-запросов — `http.NewRequestWithContext`:**
   ```go
   req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
   ```

5. **Для БД — `db.QueryContext(ctx, ...)`:**
   ```go
   rows, err := db.QueryContext(ctx, "SELECT ...")
   ```

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Создавать `ctx` в каждой функции** — обычно он приходит извне.

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй `ctx.Done()`.** Если функция долгая — она должна проверять отмену.
8. **Не пытайся «убить» горутину.** В Go нет такого API.
9. **Не используй `time.Sleep` в отменяемом коде.** Используй `select` с `ctx.Done()`.

### Ключевые выводы подглавы 5.1

- **Нельзя «убить» горутину** — она может держать ресурсы.
- **Отмена — кооперация.** Горутина сама проверяет сигнал.
- **`Context` — сигнал отмены**, а не принуждение.
- **Горутина должна проверять `ctx.Done()`.** Иначе отмена не сработает.
- **`defer` выполняются** при корректном завершении.

---

## 5.2 Context: интерфейс и контракт

Разберём, **что такое** `context.Context`.

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

| Метод | Что возвращает | Когда использовать |
|:---|:---|:---|
| `Deadline()` | Дедлайн (если есть) | Для установки таймаутов на операции |
| `Done()` | Канал, закрывающийся при отмене | В `select` для проверки отмены |
| `Err()` | Причина отмены | После `<-Done()` для диагностики |
| `Value(key)` | Значение по ключу | Для передачи request-scoped данных |

### Базовые контексты

**`context.Background()`:**

- Корневой контекст.
- Никогда не отменяется.
- Не имеет дедлайна.
- Не имеет значений.
- **Используется:** в `main`, в `init`, в тестах, как база для `With*`.

**`context.TODO()`:**

- Аналогичен `Background()`.
- **Используется:** когда не ясно, какой контекст использовать (заглушка).
- **Соглашение:** `TODO` — «я знаю, что нужно исправить».

**Разница:** только в **семантике**. `Background` — «я знаю, что делаю». `TODO` — «я пока не знаю».

### Производные контексты

**`WithCancel(parent)`:**

- Возвращает `ctx, cancel`.
- `cancel()` отменяет `ctx` и всех потомков.

**`WithTimeout(parent, d)`:**

- Возвращает `ctx, cancel`.
- `ctx` отменяется через `d` или при `cancel()`.

**`WithDeadline(parent, t)`:**

- Возвращает `ctx, cancel`.
- `ctx` отменяется в `t` или при `cancel()`.

**`WithValue(parent, key, val)`:**

- Возвращает `ctx`.
- `ctx.Value(key)` возвращает `val`.

### Контракт Context

**Правила использования:**

1. **`Context` — первый аргумент функции** (если функция его принимает):
   ```go
   func Process(ctx context.Context, data []byte) error
   ```

2. **`Context` не хранится в структурах:**
   ```go
   // ❌ Плохо:
   type Service struct {
       ctx context.Context
   }
   
   // ✅ Хорошо:
   func (s *Service) Process(ctx context.Context) error
   ```

3. **`Context` не передаётся как `nil`:**
   ```go
   // ❌ Плохо:
   Process(nil)
   
   // ✅ Хорошо:
   Process(context.Background())
   ```

4. **`cancel()` всегда вызывается:**
   ```go
   ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
   defer cancel()  // обязательно!
   ```

5. **`Context` — неизменяемый.** `With*` создаёт новый контекст, не меняет родителя.

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

**Свойства дерева:**

- **Отмена родителя отменяет всех потомков.**
- **Отмена потомка не отменяет родителя.**
- **`Deadline` родителя наследуется потомками** (если не переопределён).
- **`Value` родителя доступно потомкам.**

### Аннотация сложности

| Операция | Time | Space | Use case |
|:---|:---|:---|:---|
| `context.Background()` | 0 | 0 (singleton) | Корень |
| `WithCancel` | ~50-100 нс | ~100 байт | Ручная отмена |
| `WithTimeout` | ~200-500 нс | ~150 байт + таймер | Отмена по времени |
| `WithDeadline` | ~200-500 нс | ~150 байт + таймер | Отмена по дедлайну |
| `WithValue` | ~50-100 нс | ~50 байт | Request-scoped данные |
| `ctx.Done()` | ~1-5 нс | 0 | Проверка отмены |
| `ctx.Err()` | ~1-5 нс | 0 | Причина отмены |

### 💡 Практика: как использовать Context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`Context` — первый аргумент:**
   ```go
   func Process(ctx context.Context, ...) error
   ```

2. **`defer cancel()`** — всегда, даже если таймаут:
   ```go
   ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
   defer cancel()
   ```

3. **`context.Background()` для корня.** Не `nil`.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`context.TODO()`** — как заглушка, если не знаешь, какой контекст.
5. **`WithValue` только для request-scoped данных** (trace ID, user ID).

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Создавать контекст в каждой функции** — обычно приходит извне.

**❌ НЕ ДЕЛАЙ:**

7. **Не храни `Context` в структуре.** Передавай явно.
8. **Не передавай `nil` как `Context`.** Используй `Background()` или `TODO()`.
9. **Не забывай `cancel()`.** Утечка горутин и таймеров.

### Ключевые выводы подглавы 5.2

- **`Context` — интерфейс** с четырьмя методами: `Deadline`, `Done`, `Err`, `Value`.
- **`Background()`** — корень, никогда не отменяется. **`TODO()`** — заглушка.
- **`WithCancel`, `WithTimeout`, `WithDeadline`, `WithValue`** — производные.
- **`Context` — первый аргумент функции.** Не хранится в структурах.
- **`cancel()` всегда вызывается.** Иначе утечка.
- **Дерево:** отмена родителя отменяет потомков. Обратно — нет.

---

## 5.3 Дерево отмены: как устроен cancelCtx

Разберём, **как устроен** `cancelCtx` внутри.

### Структура cancelCtx

```go
type cancelCtx struct {
    Context                    // родитель (встроенный)
    
    mu       sync.Mutex        // защита
    done     atomic.Value      // chan struct{} — закрывается при отмене
    children map[canceler]struct{}  // потомки
    err      error             // причина отмены
}
```

**Поля:**

| Поле | Назначение |
|:---|:---|
| `Context` | Родитель (встроенный интерфейс) |
| `mu` | Защита `children`, `err`, инициализации `done` |
| `done` | Канал, закрывающийся при отмене |
| `children` | Множество потомков, которых нужно отменить |
| `err` | Причина отмены (`Canceled`, `DeadlineExceeded`) |

### Как создаётся cancelCtx

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    c := &cancelCtx{}
    c.Context = parent  // встроенный родитель
    
    // Регистрируемся у родителя (если он cancelCtx)
    if p, ok := parentCancelCtx(parent); ok {
        p.mu.Lock()
        if p.err != nil {
            // Родитель уже отменён — отменяем сразу
            c.cancel(true, Canceled, nil)
        } else {
            // Добавляем себя в children родителя
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
    close(c.done)  // закрываем канал
    
    // Отменяем всех потомков
    for child := range c.children {
        child.cancel(false, err, cause)
    }
    c.children = nil
    
    c.mu.Unlock()
    
    if removeFromParent {
        removeChild(c.Context, c)  // удаляем себя из родителя
    }
}
```

**Ключевые шаги:**

1. **Проверка `c.err != nil`** — идемпотентность. Повторный `cancel()` — no-op.
2. **`close(c.done)`** — закрывает канал. Все, кто ждёт `<-ctx.Done()`, разблокируются.
3. **Рекурсивная отмена потомков** — проходит по `children`, вызывает `cancel` у каждого.
4. **Удаление из родителя** — если `removeFromParent`, удаляет себя из `children` родителя.

### Идемпотентность cancel

**`cancel()` можно вызывать несколько раз.** Повторные вызовы — no-op.

```go
ctx, cancel := context.WithCancel(context.Background())
cancel()
cancel()  // ничего не происходит
cancel()  // тоже
```

**Почему:** `c.err != nil` — если уже отменён, выходим. Это защита от паники при `close(c.done)` дважды.

### Done() и Err()

**`Done()`:**

```go
func (c *cancelCtx) Done() <-chan struct{} {
    c.mu.Lock()
    if c.done == nil {
        c.done = make(chan struct{})
    }
    d := c.done
    c.mu.Unlock()
    return d
}
```

**Ленивая инициализация:** `done` создаётся при первом вызове `Done()`. Если никто не вызывает `Done()`, канал не создаётся.

**`Err()`:**

```go
func (c *cancelCtx) Err() error {
    c.mu.Lock()
    err := c.err
    c.mu.Unlock()
    return err
}
```

**Возвращает:**

- `nil` — если не отменён.
- `context.Canceled` — если вызван `cancel()`.
- `context.DeadlineExceeded` — если истёк таймаут.

### Визуализация отмены

```
ДО ОТМЕНЫ:

  cancelCtx A (err=nil, done=открыт)
      │
      ├── cancelCtx B (err=nil, done=открыт)
      │       │
      │       └── cancelCtx C (err=nil, done=открыт)
      │
      └── cancelCtx D (err=nil, done=открыт)

A.cancel():

  1. A.err = Canceled
  2. close(A.done)  ← все, кто ждёт A.Done(), разблокируются
  3. Для каждого child:
     - B.cancel():
       - B.err = Canceled
       - close(B.done)
       - C.cancel():
         - C.err = Canceled
         - close(C.done)
     - D.cancel():
       - D.err = Canceled
       - close(D.done)

ПОСЛЕ ОТМЕНЫ:

  cancelCtx A (err=Canceled, done=закрыт)
      │
      ├── cancelCtx B (err=Canceled, done=закрыт)
      │       │
      │       └── cancelCtx C (err=Canceled, done=закрыт)
      │
      └── cancelCtx D (err=Canceled, done=закрыт)
```

### Аннотация сложности

| Операция | Time | Space | Use case |
|:---|:---|:---|:---|
| `WithCancel` | ~50-100 нс | ~100 байт | Создание |
| `cancel()` | ~100-500 нс | 0 | Отмена |
| `cancel()` с N потомками | O(N) | 0 | Рекурсивная отмена |
| `Done()` | ~1-5 нс | 0 (ленивая) | Проверка |
| `Err()` | ~1-5 нс | 0 | Причина |

### 💡 Практика: как использовать cancelCtx

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`defer cancel()`** — всегда. Даже если отменяешь вручную.
2. **Помни: отмена родителя отменяет потомков.** Но не наоборот.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Для ручной отмены — `WithCancel`.** Для таймаута — `WithTimeout`.
4. **Проверяй `ctx.Err()` после `<-ctx.Done()`** для диагностики.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **Знать точное устройство `cancelCtx`** — достаточно понимать дерево.

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `cancel()`.** Даже если контекст с таймаутом — таймер нужно освободить.
7. **Не вызывай `cancel()` из горутины-потомка.** Только из создателя.

### Ключевые выводы подглавы 5.3

- **`cancelCtx`** — структура с `done`, `children`, `err`.
- **`WithCancel` регистрируется у родителя** — добавляется в `children`.
- **Отмена родителя отменяет всех потомков** — рекурсивно.
- **`cancel()` идемпотентен** — повторные вызовы no-op.
- **`Done()` ленивый** — канал создаётся при первом вызове.
- **`Err()`** возвращает `Canceled` или `DeadlineExceeded`.

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

func worker(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("worker %d: canceled\n", id)
            return
        default:
            time.Sleep(100 * time.Millisecond)
        }
    }
}
```

**Что происходит:**

1. 10 горутин запускаются с одним `ctx`.
2. Через 1 секунду `cancel()` отменяет все.
3. Все горутины видят `ctx.Done()` и завершаются.
4. `wg.Wait()` ждёт завершения.

### Паттерн: отмена по событию

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

**Что происходит:** `Ctrl+C` (SIGINT) вызывает `cancel()`, все горутины завершаются. Это основа **graceful shutdown** (Глава 11).

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

### Аннотация сложности

| Операция | Time | Space | Use case |
|:---|:---|:---|:---|
| `WithCancel` | ~50-100 нс | ~100 байт | Создание |
| `cancel()` | ~100-500 нс | 0 | Отмена |
| `<-ctx.Done()` | ~1-5 нс (если закрыт) | 0 | Проверка |
| Отмена N горутин | O(N) | 0 | Массовая отмена |

### 💡 Практика: как использовать WithCancel

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`defer cancel()`** — даже если отменяешь вручную.
2. **Передавай `ctx` во все горутины**, которые должны отменяться.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Для отмены по сигналу — `signal.Notify` + `cancel()`.**
4. **Для вложенных контекстов — отменяй родителя, чтобы отменить всех.**

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **Проверять `ctx.Err()` после отмены** — для логирования.

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `cancel()`.** Утечка.
7. **Не отменяй родителя, если хочешь отменить только потомка.** Используй отдельный `WithCancel`.

### Ключевые выводы подглавы 5.4

- **`WithCancel`** — ручная отмена через `cancel()`.
- **`cancel()` идемпотентен.**
- **Отмена родителя отменяет потомков.** Обратно — нет.
- **Для отмены по сигналу — `signal.Notify` + `cancel()`.**
- **`defer cancel()`** — всегда.

---

## 5.5 WithTimeout и WithDeadline: отмена по времени

Разберём **отмену по времени**.

### WithTimeout vs WithDeadline

**`WithTimeout(parent, d)`** — отмена через `d` от **текущего момента**.

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
// ctx отменится через 5 секунд
```

**`WithDeadline(parent, t)`** — отмена в **абсолютный момент** `t`.

```go
deadline := time.Now().Add(5 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
// ctx отменится в deadline
```

**Связь:** `WithTimeout(parent, d)` эквивалентно `WithDeadline(parent, time.Now().Add(d))`.

### Структура timerCtx

```go
type timerCtx struct {
    cancelCtx              // встроенный cancelCtx
    
    timer    *time.Timer   // таймер
    deadline time.Time     // дедлайн
}
```

**`timerCtx`** — надстройка над `cancelCtx`. Добавляет таймер и дедлайн.

### Как работает WithTimeout

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    // Если родитель имеет более ранний дедлайн — используем его
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        return WithCancel(parent)
    }
    
    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  d,
    }
    
    // Регистрируемся у родителя
    propagateCancel(parent, c)
    
    // Проверяем, не истёк ли дедлайн
    dur := time.Until(d)
    if dur <= 0 {
        c.cancel(true, DeadlineExceeded, nil)
        return c, func() { c.cancel(false, Canceled, nil) }
    }
    
    // Запускаем таймер
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
    defer cancel()  // освобождаем таймер
    
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
    
    // БД: до 2 секунд
    user, err := db.GetUser(dbCtx, 42)
    // ...
    
    // API: до 5 секунд
    profile, err := api.GetProfile(apiCtx, user.ID)
    // ...
}
```

**Что происходит:**

- **Родительский `ctx`** — 10 секунд на всё.
- **`dbCtx`** — 2 секунды, наследует от `ctx`.
- **`apiCtx`** — 5 секунд, наследует от `ctx`.

**Если `ctx` отменится раньше** (например, клиент отключился) — все потомки отменятся.

**Если `dbCtx` отменится через 2 секунды** — только БД-операция отменится. `apiCtx` продолжит.

### Аннотация сложности

| Операция | Time | Space | Use case |
|:---|:---|:---|:---|
| `WithTimeout` | ~200-500 нс | ~150 байт + таймер | Создание |
| `WithDeadline` | ~200-500 нс | ~150 байт + таймер | Создание |
| `cancel()` | ~100-500 нс + `timer.Stop()` | 0 | Отмена |
| Таймер срабатывает | ~100-500 нс | 0 | Автоотмена |
| `<-ctx.Done()` | ~1-5 нс | 0 | Проверка |

### 💡 Практика: как использовать таймауты

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`defer cancel()`** — даже если контекст с таймаутом. Иначе таймер утечёт.
2. **Для HTTP — `http.NewRequestWithContext`.**
3. **Для БД — `db.QueryContext`.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Вложенные таймауты** — общий + на каждую операцию.
5. **`WithDeadline`** — если есть абсолютный момент (например, дедлайн запроса).

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Очень короткие таймауты** — могут вызвать ложные срабатывания.

**❌ НЕ ДЕЛАЙ:**

7. **Не забывай `cancel()`.** Таймер утечёт.
8. **Не используй `time.After` вместо `Context`.** `time.After` не отменяет операцию.
9. **Не ставь таймаут больше, чем у родителя.** Он всё равно не сработает.

### Ключевые выводы подглавы 5.5

- **`WithTimeout`** — отмена через `d` от текущего момента.
- **`WithDeadline`** — отмена в абсолютный момент.
- **`timerCtx`** — надстройка над `cancelCtx` с таймером.
- **`cancel()` останавливает таймер** — обязательно для избежания утечек.
- **Вложенные таймауты** — общий + на каждую операцию.
- **`<-ctx.Done()`** — способ проверить отмену.

---

## 5.6 WithValue: что это и почему его не любят

Разберём `WithValue` — самый спорный метод `Context`.

### Что делает WithValue

```go
ctx = context.WithValue(ctx, "user_id", 42)
// ...
userID := ctx.Value("user_id").(int)
```

**Что происходит:** создаётся `valueCtx`, который хранит пару `(key, value)`. При `Value(key)` ищется по цепочке контекстов.

### Структура valueCtx

```go
type valueCtx struct {
    Context              // родитель
    key, val any         // пара
}

func (c *valueCtx) Value(key any) any {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)  // ищем у родителя
}
```

**Поиск:** `Value(key)` идёт по цепочке **от текущего к корню**. Первое совпадение — возвращается.

### Почему WithValue не любят

**1. Нетипизированные ключи.**

```go
ctx = context.WithValue(ctx, "user_id", 42)  // key — строка
userID := ctx.Value("user_id").(int)         // type assertion
```

**Проблемы:**

- **Ключ — `any`.** Легко ошибиться: `"user_id"` vs `"userId"`.
- **Значение — `any`.** Нужен type assertion, который может паниковать.
- **Нет проверки на этапе компиляции.** Опечатка в ключе — ошибка в runtime.

**2. Неявные зависимости.**

```go
func process(ctx context.Context) error {
    userID := ctx.Value("user_id").(int)  // откуда?
    // ...
}
```

**Проблема:** функция `process` зависит от `user_id`, но это **не видно в сигнатуре**. Легко вызвать без `user_id` — паника.

**3. «Магические» данные.**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    userID := ctx.Value("user_id").(int)  // где установлено?
    // ...
}
```

**Проблема:** неясно, **где** и **кем** установлен `user_id`. Приходится искать по всему коду.

**4. Проблемы с типизацией.**

```go
ctx = context.WithValue(ctx, "count", 42)
// ...
count := ctx.Value("count").(int)     // OK
count := ctx.Value("count").(int64)   // panic!
```

**Проблема:** тип не проверяется компилятором. `42` — `int`, а не `int64`.

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

// Использование:
ctx = WithUserID(ctx, 42)
if id, ok := UserID(ctx); ok {
    // ...
}
```

**Что это даёт:**

- **Типизированный ключ** — нельзя перепутать с другим.
- **Функции-обёртки** — скрывают type assertion.
- **Безопасность** — `ok` вместо паники.

### Когда WithValue нужен

**1. Request-scoped данные:**

- Trace ID (для распределённого трейсинга).
- Request ID.
- User ID (после аутентификации).
- Correlation ID.

**2. Данные, которые передаются через много уровней:**

- HTTP-хэндлер → сервис → репозиторий → БД.
- Без `Context` пришлось бы добавлять параметр в каждую функцию.

**3. Данные, которые не влияют на логику:**

- Логирование, метрики, трейсинг.
- Если данные влияют на логику — они должны быть **явными параметрами**.

### Когда WithValue НЕ нужен

**1. Опциональные параметры:**

```go
// ❌ Плохо:
ctx = context.WithValue(ctx, "page_size", 10)

// ✅ Хорошо:
func ListUsers(ctx context.Context, pageSize int) ([]User, error)
```

**2. Конфигурация:**

```go
// ❌ Плохо:
ctx = context.WithValue(ctx, "db_host", "localhost")

// ✅ Хорошо:
type Config struct { DBHost string }
func NewService(cfg Config) *Service
```

**3. Зависимости:**

```go
// ❌ Плохо:
ctx = context.WithValue(ctx, "logger", logger)

// ✅ Хорошо:
type Service struct { logger *slog.Logger }
```

### Аннотация сложности

| Операция | Time | Space | Use case |
|:---|:---|:---|:---|
| `WithValue` | ~50-100 нс | ~50 байт | Создание |
| `Value(key)` | O(depth) | 0 | Поиск |
| `Value(key)` при 10 контекстах | ~10-50 нс | 0 | Поиск |
| `Value(key)` при 100 контекстах | ~100-500 нс | 0 | Поиск |

**Ключевое:** `Value` ищет по **цепочке**. Чем глубже — тем дольше. Не создавай длинные цепочки.

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

3. **Только для request-scoped данных:** trace ID, user ID, request ID.
4. **Никогда — для опциональных параметров.** Они должны быть явными.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **Документировать, какие ключи ожидаются.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй строковые ключи.** Только типизированные.
7. **Не передавай через `Context` конфигурацию, зависимости, опциональные параметры.**
8. **Не создавай длинные цепочки `WithValue`.** Поиск замедляется.

### Ключевые выводы подглавы 5.6

- **`WithValue`** — для request-scoped данных.
- **Проблемы:** нетипизированные ключи, неявные зависимости, «магические» данные.
- **Решение:** типизированные ключи + функции-обёртки.
- **Использовать:** trace ID, user ID, request ID.
- **НЕ использовать:** опциональные параметры, конфигурация, зависимости.
- **Поиск `Value`** — O(depth). Не создавай длинные цепочки.

---

## 5.7 Как работает отмена: Done(), Err(), распространение

Разберём **механику отмены**.

### Done() — канал отмены

```go
select {
case <-ctx.Done():
    // контекст отменён
    return ctx.Err()
case result := <-resultCh:
    // результат получен
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

**Возможные значения:**

| Значение | Когда |
|:---|:---|
| `nil` | Контекст не отменён |
| `context.Canceled` | Вызван `cancel()` |
| `context.DeadlineExceeded` | Истёк таймаут/дедлайн |

**Важно:** `Err()` возвращает **не nil** только **после** закрытия `Done()`. До этого — `nil`.

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

**Что происходит при отмене:** **ничего**. Горутина продолжит работу. `ctx.Done()` не проверяется.

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
        // продолжаем
    }
    
    // работа
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
// долгая операция
```

**4. В `http.NewRequestWithContext`:**

```go
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
resp, err := http.DefaultClient.Do(req)
// Do вернёт ошибку при отмене
```

### Визуализация отмены

```
t=0:     ctx создан, Done() открыт

Горутина A: select { case <-ctx.Done(): ... case <-ch: ... }
Горутина B: select { case <-ctx.Done(): ... case <-ch: ... }
Горутина C: select { case <-ctx.Done(): ... case <-ch: ... }

t=5s:    cancel() вызван
         - close(ctx.Done())
         - ctx.Err() = Canceled

Горутина A: <-ctx.Done() готов → return
Горутина B: <-ctx.Done() готов → return
Горутина C: <-ctx.Done() готов → return

t=5s+:   все горутины завершились
```

**Ключевое:** `close(Done())` разблокирует **всех**, кто ждёт. Это broadcast.

### Аннотация сложности

| Операция | Time | Space | Use case |
|:---|:---|:---|:---|
| `close(ctx.Done())` | ~50-100 нс | 0 | Отмена |
| `<-ctx.Done()` (закрыт) | ~1-5 нс | 0 | Проверка |
| `ctx.Err()` | ~1-5 нс | 0 | Причина |
| Проверка в цикле | ~5-10 нс на итерацию | 0 | Долгие операции |

### 💡 Практика: как проверять отмену

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **В `select` — `case <-ctx.Done()`:**
   ```go
   select {
   case <-ctx.Done():
       return ctx.Err()
   case result := <-resultCh:
       return result
   }
   ```

2. **В долгих циклах — проверка:**
   ```go
   for {
       select {
       case <-ctx.Done():
           return ctx.Err()
       default:
       }
       // работа
   }
   ```

**👍 СТОИТ СДЕЛАТЬ:**

3. **Перед долгой операцией — `if err := ctx.Err(); err != nil`.**

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

4. **Проверять `ctx.Err()` после `<-ctx.Done()`** — для диагностики.

**❌ НЕ ДЕЛАЙ:**

5. **Не игнорируй `ctx.Done()` в долгих операциях.** Отмена не сработает.
6. **Не используй `time.Sleep` в отменяемом коде.** Используй `select` с `ctx.Done()`.

### Ключевые выводы подглавы 5.7

- **`Done()`** — канал, закрывающийся при отмене. **Broadcast** всем.
- **`Err()`** — `nil`, `Canceled`, `DeadlineExceeded`.
- **Отмена рекурсивна** — родитель отменяет потомков.
- **Горутина должна проверять `ctx.Done()`.** Иначе отмена не сработает.
- **`close(Done())` разблокирует всех.**

---

## 5.8 Context и happens-before: связь с Главой 4

В Главе 4 мы разобрали happens-before. Теперь свяжем с `Context`.

### Context устанавливает happens-before

**`close(ctx.Done())` → `<-ctx.Done()`:**

`close` канала **happens-before** получения zero-значения из закрытого канала (Глава 4).

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

### Пример: передача данных через Context

```go
var result string
var done = make(chan struct{})

// Горутина A:
result = "computed"  // (1)
close(done)          // (2)

// Горутина B:
<-done               // (3)
fmt.Println(result)  // (4) — увидит "computed"
```

**Что гарантируется:** (1) → (4), потому что (2) → (3).

**Аналогия с Context:** `close(ctx.Done())` — это `close(done)`.

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

**Без Context:**

```go
var data string
var ready bool

// Горутина A:
data = "hello"
ready = true

// Горутина B:
for !ready {}
fmt.Println(data)
```

**Race detector:** **найдёт** гонку. Нет happens-before.

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

### Context и каналы

**`Context` использует каналы.** `ctx.Done()` — это `<-chan struct{}`.

**Связь:**

- `close(ctx.Done())` — как `close(ch)`.
- `<-ctx.Done()` — как `<-ch` (получение zero).
- Happens-before — как у каналов.

**Ключевое:** `Context` **не заменяет** каналы, а **использует** их.

### Аннотация сложности

| Операция | Happens-before | Аналог |
|:---|:---|:---|
| `cancel()` → `<-ctx.Done()` | ✅ Да | `close(ch)` → `<-ch` |
| `<-ctx.Done()` → `ctx.Err()` | ✅ Да | — |
| `WithValue` → `Value` | ✅ Да (через родителя) | — |
| `WithCancel` → `cancel` | ✅ Да | — |

### 💡 Практика: как использовать happens-before Context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Помни: `cancel()` → `<-ctx.Done()` устанавливает happens-before.** Данные, записанные до `cancel()`, видны после `<-Done()`.

2. **Не нужна дополнительная синхронизация** для данных, переданных через `Context`.

**👍 СТОИТ СДЕЛАТЬ:**

3. **Используй `Context` для передачи данных**, если нужен happens-before.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

4. **Знать точную механику** — достаточно понимать, что happens-before есть.

**❌ НЕ ДЕЛАЙ:**

5. **Не полагайся на happens-before без `Context`.** Обычные переменные — race.

### Ключевые выводы подглавы 5.8

- **`cancel()` → `<-ctx.Done()`** устанавливает happens-before.
- **Аналог:** `close(ch)` → `<-ch`.
- **`cancel()` — release, `<-Done()` — acquire.**
- **Race detector понимает `Context`.**
- **`Context` использует каналы** для happens-before.

---

## 5.9 Context и другие примитивы: select, Mutex, WaitGroup

Разберём, **как комбинировать** `Context` с другими примитивами.

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

**Порядок case'ов** не важен. `select` выберет **случайный** из готовых.

### Context и Mutex

**❌ Плохо: держать мьютекс при отмене**

```go
func process(ctx context.Context) error {
    mu.Lock()
    defer mu.Unlock()
    
    select {
    case <-ctx.Done():
        return ctx.Err()  // мьютекс всё ещё захвачен, но defer освободит
    case result := <-resultCh:
        return result
    }
}
```

**Проблема:** если отмена произошла, `select` вернёт `ctx.Err()`. `defer mu.Unlock()` освободит мьютекс. Это **корректно**.

**Но:** если `resultCh` долго не готов, мьютекс держится **всё время ожидания**. Это блокирует других.

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

**Что изменилось:** мьютекс держится только на время работы с данными, не на время ожидания.

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

**Важно:** `WaitGroup` **не** отменяется. Только `Context`. `WaitGroup` просто ждёт.

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

**Проблема:** `time.After` **не отменяет** операцию. Если `resultCh` не готов, горутина продолжит ждать. `time.After` создаёт **новый таймер** каждый раз.

**✅ Хорошо: `Context` с таймаутом**

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()

select {
case result := <-resultCh:
    return result
case <-ctx.Done():
    return ctx.Err()  // DeadlineExceeded
}
```

**Что изменилось:** `Context` отменяет операцию, если она поддерживает отмену. И освобождает таймер через `cancel()`.

### Context и sync.Once

**Паттерн: ленивая инициализация с отменой**

```go
type Service struct {
    once sync.Once
    conn *Connection
    err  error
}

func (s *Service) GetConn(ctx context.Context) (*Connection, error) {
    s.once.Do(func() {
        s.conn, s.err = connect(ctx)
    })
    return s.conn, s.err
}
```

**Проблема:** `once.Do` **не отменяется**. Если `connect(ctx)` завис, `once.Do` заблокирует всех.

**Решение:** использовать `Mutex` + флаг:

```go
type Service struct {
    mu   sync.Mutex
    conn *Connection
    err  error
    done bool
}

func (s *Service) GetConn(ctx context.Context) (*Connection, error) {
    s.mu.Lock()
    if s.done {
        conn, err := s.conn, s.err
        s.mu.Unlock()
        return conn, err
    }
    s.mu.Unlock()
    
    conn, err := connect(ctx)
    
    s.mu.Lock()
    s.conn = conn
    s.err = err
    s.done = true
    s.mu.Unlock()
    
    return conn, err
}
```

**Что изменилось:** `connect(ctx)` может отмениться. `done` устанавливается только при успехе.

### Аннотация сложности

| Комбинация | Проблема | Решение |
|:---|:---|:---|
| `Context` + `Mutex` | Держать мьютекс при ожидании | Освобождать до `select` |
| `Context` + `WaitGroup` | `WaitGroup` не отменяется | `cancel()` + `wg.Wait()` |
| `Context` + `time.After` | `time.After` не отменяет | `Context` с таймаутом |
| `Context` + `Once` | `once.Do` не отменяется | `Mutex` + флаг |

### 💡 Практика: как комбинировать Context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`select` с `ctx.Done()`** — основной паттерн.
2. **Не держи мьютекс при долгом ожидании** — освобождай до `select`.
3. **Для таймаута — `Context`, не `time.After`.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`cancel()` + `wg.Wait()`** — для массовой отмены.
5. **Для ленивой инициализации с отменой — `Mutex` + флаг.**

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Проверять `ctx.Err()` после отмены** — для диагностики.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `time.After` вместо `Context`.** `time.After` не отменяет.
8. **Не держи мьютекс при ожидании.** Блокирует других.
9. **Не используй `once.Do` для отменяемой инициализации.** `once.Do` не отменяется.

### Ключевые выводы подглавы 5.9

- **`select` с `ctx.Done()`** — основной паттерн.
- **`Mutex`** — освобождай до `select`.
- **`WaitGroup`** — `cancel()` + `wg.Wait()`.
- **`time.After`** не отменяет — используй `Context`.
- **`once.Do`** не отменяется — используй `Mutex` + флаг.

---

## 5.10 Context в реальных сценариях: HTTP, БД, gRPC, Kafka

Разберём **реальные сценарии** использования `Context`.

### HTTP-сервер

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()  // контекст от HTTP-сервера
    // ctx отменяется, если клиент отключился
    
    user, err := db.GetUser(ctx, 42)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    
    profile, err := api.GetProfile(ctx, user.ID)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    
    json.NewEncoder(w).Encode(profile)
}
```

**Что происходит:**

1. `r.Context()` — контекст запроса. Отменяется при отключении клиента.
2. `db.GetUser(ctx, ...)` — БД-запрос отменяется.
3. `api.GetProfile(ctx, ...)` — API-запрос отменяется.
4. Если клиент отключился — все операции прерываются.

**Важно:** `r.Context()` **уже** создан. Не нужно `context.Background()`.

### HTTP-клиент

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

**Что происходит:**

1. `WithTimeout(ctx, 5s)` — добавляет таймаут.
2. `http.NewRequestWithContext` — привязывает контекст к запросу.
3. `http.Do` — отменяется при таймауте или отмене родителя.
4. `defer cancel()` — освобождает таймер.

### База данных

```go
func getUser(ctx context.Context, id int) (*User, error) {
    var user User
    err := db.QueryRowContext(ctx, 
        "SELECT id, name FROM users WHERE id = $1", id).
        Scan(&user.ID, &user.Name)
    if err != nil {
        return nil, err
    }
    return &user, nil
}
```

**Что происходит:** `QueryRowContext` отменяется при отмене `ctx`. Если БД-запрос долгий — он прервётся.

**Важно:** использовать `Context`-версии методов (`QueryContext`, `ExecContext`, `QueryRowContext`).

### gRPC

```go
// Сервер:
func (s *Server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    user, err := s.db.GetUser(ctx, req.Id)
    if err != nil {
        return nil, err
    }
    return &pb.User{Id: user.ID, Name: user.Name}, nil
}

// Клиент:
func (c *Client) GetUser(ctx context.Context, id int) (*pb.User, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    return c.client.GetUser(ctx, &pb.GetUserRequest{Id: int32(id)})
}
```

**Что происходит:** gRPC **автоматически** пробрасывает `Context` через сеть. Если клиент отменяет — сервер видит отмену.

### Kafka

```go
func consume(ctx context.Context, consumer *kafka.Consumer) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }
        
        msg, err := consumer.ReadMessage(100 * time.Millisecond)
        if err != nil {
            if errors.Is(err, kafka.ErrTimedOut) {
                continue
            }
            return err
        }
        
        if err := process(ctx, msg); err != nil {
            return err
        }
    }
}
```

**Что происходит:** consumer проверяет `ctx.Done()` в каждой итерации. При отмене — завершается корректно.

**Важно:** `ReadMessage` с таймаутом, чтобы не блокироваться навсегда.

### Аннотация сложности

| Сценарий | Time | Space | Use case |
|:---|:---|:---|:---|
| HTTP-сервер | ~100-500 нс | ~150 байт | Request-scoped |
| HTTP-клиент | ~200-500 нс | ~150 байт + таймер | Таймаут |
| БД | ~100-500 нс | ~150 байт | Таймаут |
| gRPC | ~200-500 нс | ~150 байт | Проброс через сеть |
| Kafka | ~100-500 нс | ~150 байт | Consumer |

### 💡 Практика: Context в реальных сценариях

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **HTTP-сервер:** `r.Context()` — контекст запроса.
2. **HTTP-клиент:** `http.NewRequestWithContext`.
3. **БД:** `QueryContext`, `ExecContext`, `QueryRowContext`.
4. **gRPC:** пробрасывай `ctx` во все вызовы.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Таймауты** на каждую операцию: `WithTimeout`.
6. **Kafka:** проверяй `ctx.Done()` в цикле.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **Метрики отмены** — сколько запросов отменено.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй `context.Background()` в HTTP-хэндлере.** Используй `r.Context()`.
9. **Не игнорируй `ctx` в БД-запросах.** Используй `Context`-версии.
10. **Не блокируйся на `ReadMessage` без таймаута.**

### Ключевые выводы подглавы 5.10

- **HTTP-сервер:** `r.Context()` — контекст запроса.
- **HTTP-клиент:** `http.NewRequestWithContext` + таймаут.
- **БД:** `QueryContext`, `ExecContext`, `QueryRowContext`.
- **gRPC:** пробрасывает `ctx` автоматически.
- **Kafka:** проверяй `ctx.Done()` в цикле.

---

## 5.11 Практика Go: диагностика Context

Теперь напишем **утилиту на Go**, которая демонстрирует и диагностирует `Context`:

1. **Утечка горутин без `cancel()`.**
2. **Отмена через `WithCancel`.**
3. **Таймаут через `WithTimeout`.**
4. **Дерево контекстов.**
5. **Диагностика утечек через `runtime.NumGoroutine`.**

### Утилита 1: утечка горутин без cancel

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
    // 1000+ горутин висят вечно
}

func worker(ctx context.Context) {
    <-ctx.Done()  // ждёт вечно
}
```

**Вывод:**

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

### Утилита 2: отмена через WithCancel

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

**Вывод:**

```
worker 0: context canceled
worker 1: context canceled
...
canceling...
worker 9: context canceled
all workers stopped
```

**Что видно:** все 10 горутин завершились после `cancel()`.

### Утилита 3: таймаут через WithTimeout

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
        return nil  // успех через 2 секунды
    case <-ctx.Done():
        return ctx.Err()  // отмена через 500 мс
    }
}
```

**Вывод:**

```
elapsed: 500ms, err: context deadline exceeded
```

**Что видно:** операция прервалась через 500 мс, а не через 2 секунды.

### Утилита 4: дерево контекстов

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

**Вывод:**

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

### Утилита 5: диагностика утечек

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

**Вывод:**

```
goroutines: 102
goroutines: 3
goroutines: 2
done
```

**Что видно:** горутины создаются, потом завершаются. Утечки нет.

### Аннотация сложности

| Сценарий | Time | Space | Use case |
|:---|:---|:---|:---|
| Утечка без cancel | ∞ | ~2 КБ на горутину | Баг |
| Отмена через WithCancel | ~мс | 0 | Массовая отмена |
| Таймаут через WithTimeout | ~мс | 0 | Таймаут |
| Дерево контекстов | ~мс | 0 | Вложенные таймауты |

### 💡 Практика: как диагностировать Context

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`defer cancel()`** — всегда. Даже если контекст с таймаутом.
2. **Мониторь `runtime.NumGoroutine()`** — рост без падения = утечка.

**👍 СТОИТ СДЕЛАТЬ:**

3. **pprof goroutine dump** — показывает, где горутины висят.
4. **Тесты с `goleak`** — ловят утечки.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **Логирование отмен** — для диагностики.

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай `cancel()`.** Утечка горутин и таймеров.
7. **Не игнорируй растущее число горутин.**

### Ключевые выводы подглавы 5.11

- **Утечка без `cancel()`** — 1000 горутин висят вечно.
- **`cancel()`** завершает всех, кто ждёт `<-ctx.Done()`.
- **`WithTimeout`** прерывает операцию через таймаут.
- **Дерево:** отмена родителя отменяет потомков. Обратно — нет.
- **Диагностика:** `runtime.NumGoroutine()`, pprof.

---

## 5.12 Выводы и типичные ошибки

**Что мы узнали?**

Горутину нельзя «убить» — она может держать ресурсы. Отмена — **кооперация**: горутина сама проверяет `ctx.Done()`. `Context` — интерфейс с четырьмя методами: `Deadline`, `Done`, `Err`, `Value`. `Background()` — корень, `TODO()` — заглушка. `WithCancel`, `WithTimeout`, `WithDeadline`, `WithValue` — производные. `cancelCtx` — структура с `done`, `children`, `err`. Отмена **рекурсивна**: родитель отменяет потомков. `cancel()` **идемпотентен**. `WithTimeout` использует `timerCtx` с таймером; `cancel()` **останавливает таймер**. `WithValue` — для request-scoped данных; используй типизированные ключи. `cancel()` → `<-ctx.Done()` устанавливает happens-before (как `close` канала). `Context` комбинируется с `select`, `Mutex`, `WaitGroup`. В HTTP, БД, gRPC, Kafka — `Context` пробрасывается. Утечки горутин — главная проблема; `defer cancel()` обязателен.

**Типичные ошибки:**

- ❌ **Забывать `cancel()`.** Утечка горутин и таймеров.
- ❌ **Использовать `context.Background()` в HTTP-хэндлере.** Используй `r.Context()`.
- ❌ **Хранить `Context` в структуре.** Передавай явно.
- ❌ **Передавать `nil` как `Context`.** Используй `Background()` или `TODO()`.
- ❌ **Игнорировать `ctx.Done()` в долгих операциях.** Отмена не сработает.
- ❌ **Использовать `time.Sleep` в отменяемом коде.** Используй `select` с `ctx.Done()`.
- ❌ **Использовать строковые ключи в `WithValue`.** Типизированные ключи.
- ❌ **Передавать через `Context` опциональные параметры.** Явные параметры.
- ❌ **Использовать `time.After` вместо `Context`.** `time.After` не отменяет.
- ❌ **Держать `Mutex` при долгом ожидании.** Освобождай до `select`.
- ❌ **Использовать `once.Do` для отменяемой инициализации.** `once.Do` не отменяется.
- ❌ **Не проверять `ctx.Err()` после `<-ctx.Done()`.** Теряешь причину.
- ❌ **Создавать `Context` без необходимости.** Обычно приходит извне.
- ❌ **Полагать, что `cancel()` «убивает» горутину.** Это сигнал.

---

## 5.13 Для быстрого повторения

- **Горутину нельзя убить.** Отмена — кооперация. Горутина сама проверяет `ctx.Done()`.
- **`Context` — интерфейс:** `Deadline`, `Done`, `Err`, `Value`.
- **`Background()`** — корень. **`TODO()`** — заглушка.
- **`WithCancel`, `WithTimeout`, `WithDeadline`, `WithValue`** — производные.
- **`cancelCtx`** — `done` (канал), `children` (потомки), `err` (причина).
- **Отмена рекурсивна:** родитель отменяет потомков. Обратно — нет.
- **`cancel()` идемпотентен.** Повторные вызовы — no-op.
- **`WithTimeout`** — `timerCtx` с таймером. **`cancel()` останавливает таймер.**
- **`WithValue`** — request-scoped данные. Типизированные ключи + функции-обёртки.
- **`cancel()` → `<-ctx.Done()`** устанавливает happens-before (как `close` канала).
- **`select` с `ctx.Done()`** — основной паттерн.
- **`Mutex`** — освобождай до `select`.
- **`WaitGroup`** — `cancel()` + `wg.Wait()`.
- **`time.After`** не отменяет — используй `Context`.
- **HTTP:** `r.Context()` (сервер), `http.NewRequestWithContext` (клиент).
- **БД:** `QueryContext`, `ExecContext`, `QueryRowContext`.
- **gRPC:** пробрасывает `ctx` автоматически.
- **`defer cancel()`** — всегда. Иначе утечка.

---

## 5.14 Вопросы для самопроверки

1. Почему нельзя «убить» горутину? Что использовать вместо этого?
2. Что такое кооперативная отмена? Приведи пример.
3. Какие четыре метода у `Context`? Что каждый делает?
4. Чем `Background()` отличается от `TODO()`?
5. Что такое дерево контекстов? Что происходит при отмене родителя?
6. Как устроен `cancelCtx`? Какие поля содержит?
7. Почему `cancel()` идемпотентен?
8. Что происходит при отмене родителя? Обратно — нет?
9. Как устроен `timerCtx`? Что делает `cancel()` с таймером?
10. Почему `WithTimeout` нужно `defer cancel()`?
11. Почему `WithValue` не любят? Как использовать правильно?
12. Как `Context` связан с happens-before (Глава 4)?
13. Как комбинировать `Context` с `select`, `Mutex`, `WaitGroup`?
14. Почему `time.After` не заменяет `Context`?
15. Как использовать `Context` в HTTP-сервере? В HTTP-клиенте? В БД? В gRPC?
16. Что произойдёт, если забыть `cancel()`?
17. Как диагностировать утечки горутин, связанные с `Context`?
18. Почему `once.Do` не подходит для отменяемой инициализации?

---

## 5.15 Ответы

### Ответ 1

**Нельзя убить горутину**, потому что она может держать ресурсы: мьютексы, транзакции, файлы, каналы. Принудительное завершение оставит систему в несогласованном состоянии, `defer` не выполнятся, мьютексы останутся захваченными.

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
1. `Deadline()` — дедлайн (если есть).
2. `Done()` — канал, закрывающийся при отмене.
3. `Err()` — причина отмены.
4. `Value(key)` — значение по ключу.

### Ответ 4

**`Background()`** — корневой контекст. «Я знаю, что делаю». Используется в `main`, `init`, тестах.

**`TODO()`** — аналогичен. «Я пока не знаю, какой контекст использовать». Заглушка.

**Разница:** только в семантике.

### Ответ 5

**Дерево контекстов** — иерархия, где каждый `With*` создаёт потомка.

**При отмене родителя:** все потомки отменяются рекурсивно. Каждый потомок закрывает свой `Done()`.

**Обратно:** отмена потомка **не** отменяет родителя.

### Ответ 6

**`cancelCtx`:**
```go
type cancelCtx struct {
    Context
    mu       sync.Mutex
    done     atomic.Value           // chan struct{}
    children map[canceler]struct{}  // потомки
    err      error
}
```

- `Context` — родитель.
- `mu` — защита.
- `done` — канал отмены.
- `children` — потомки.
- `err` — причина.

### Ответ 7

**`cancel()` идемпотентен**, потому что проверяет `c.err != nil`. Если уже отменён — выходит. Это защита от `close(c.done)` дважды (паника).

### Ответ 8

**При отмене родителя:** все потомки отменяются рекурсивно. Родитель проходит по `children`, вызывает `cancel` у каждого.

**При отмене потомка:** родитель **не** отменяется. Только потомок (и его потомки).

### Ответ 9

**`timerCtx`:**
```go
type timerCtx struct {
    cancelCtx
    timer    *time.Timer
    deadline time.Time
}
```

**`cancel()`:** останавливает таймер (`c.timer.Stop()`) и вызывает `cancelCtx.cancel`.

### Ответ 10

**`defer cancel()` нужен**, потому что `cancel()` **останавливает таймер**. Без `cancel()` таймер останется в памяти до срабатывания. Если контекстов много — утечка.

### Ответ 11

**`WithValue` не любят**, потому что:
- Нетипизированные ключи (`any`).
- Неявные зависимости (не видно в сигнатуре).
- «Магические» данные (неясно, откуда).
- Проблемы с типизацией (type assertion).

**Правильно:** типизированные ключи + функции-обёртки. Только для request-scoped данных (trace ID, user ID).

### Ответ 12

**`cancel()` → `<-ctx.Done()`** устанавливает happens-before. Аналог `close(ch)` → `<-ch`.

- `cancel()` — release: всё, что было до, видно после.
- `<-ctx.Done()` — acquire: всё, что было до, видно после.

### Ответ 13

**С `select`:** `case <-ctx.Done(): return ctx.Err()`.

**С `Mutex`:** освобождай мьютекс **до** `select`. Не держи при долгом ожидании.

**С `WaitGroup`:** `cancel()` + `wg.Wait()`. `WaitGroup` не отменяется.

### Ответ 14

**`time.After` не отменяет** операцию. Если `resultCh` не готов, горутина продолжит ждать. `time.After` создаёт **новый таймер** каждый раз.

**Решение:** `Context` с таймаутом. `<-ctx.Done()` отменяет операцию, `cancel()` освобождает таймер.

### Ответ 15

**HTTP-сервер:** `r.Context()` — контекст запроса. Отменяется при отключении клиента.

**HTTP-клиент:** `http.NewRequestWithContext(ctx, ...)`. `Do` отменяется при таймауте.

**БД:** `QueryContext`, `ExecContext`, `QueryRowContext`.

**gRPC:** пробрасывает `ctx` автоматически через сеть.

### Ответ 16

**Если забыть `cancel()`:**
- **Таймер** (для `WithTimeout`) останется в памяти.
- **Горутины**, ждущие `<-ctx.Done()`, не завершатся.
- **Родитель** хранит ссылку на потомка в `children`.
- **Утечка памяти и горутин.**

### Ответ 17

**Диагностика:**
1. `runtime.NumGoroutine()` — рост без падения = утечка.
2. `pprof goroutine dump` — показывает, где горутины висят.
3. `goleak` в тестах — ловит утечки.

### Ответ 18

**`once.Do` не подходит для отменяемой инициализации**, потому что:
- `once.Do` **не отменяется**. Если `connect(ctx)` завис — `once.Do` заблокирует всех.
- После паники `Once` считается выполненным.

**Решение:** `Mutex` + флаг `done`.

---

## 5.16 Куда идти дальше?

Мы разобрали `Context`: отмену, таймауты, дедлайны, дерево отмены, happens-before, комбинацию с другими примитивами.

Но остаётся **фундаментальный вопрос**: как **планировщик** Go выбирает, какую горутину выполнять? Как работает G-M-P модель? Что такое work stealing? Как происходит асинхронное вытеснение?

- **Как планировщик выбирает горутину?** Детально G-M-P, work stealing, preemption, `GOMAXPROCS`. → **Глава 6: Планировщик Go — G-M-P, work stealing, preemption.**
- **Как построить worker pool?** Ограничить параллелизм, распределить задачи, собрать результаты. → **Глава 7: Worker pool — ограничение параллелизма.**
- **Как построить pipeline?** Fan-in, fan-out, стадии обработки. → **Глава 8: Fan-in, Fan-out, Pipeline.**

---

## 5.17 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **`Context`** | Интерфейс отмены | `Deadline`, `Done`, `Err`, `Value` |
| **`Background()`** | Корневой контекст | Никогда не отменяется |
| **`TODO()`** | Заглушка | Аналог `Background()`, но семантически «не знаю» |
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
| **`Mutex` + `Context`** | Освобождай до `select` | Не держи при ожидании |
| **`WaitGroup` + `Context`** | `cancel()` + `wg.Wait()` | `WaitGroup` не отменяется |
| **`time.After`** | НЕ отменяет | Используй `Context` |
| **`once.Do`** | НЕ отменяется | `Mutex` + флаг |
| **HTTP-сервер** | `r.Context()` | Отменяется при отключении клиента |
| **HTTP-клиент** | `http.NewRequestWithContext` | Отменяется при таймауте |
| **БД** | `QueryContext` | Отменяется при отмене |
| **gRPC** | Пробрасывает `ctx` | Автоматически через сеть |
| **`defer cancel()`** | Обязательно | Утечка без него |

🔌 **Ключевая идея:** Горутину нельзя «убить» — она может держать ресурсы. Отмена — **кооперация**: горутина сама проверяет `ctx.Done()`. `Context` — интерфейс с четырьмя методами. `Background()` — корень, `TODO()` — заглушка. `WithCancel`, `WithTimeout`, `WithDeadline`, `WithValue` — производные. `cancelCtx` — `done`, `children`, `err`. Отмена **рекурсивна**: родитель отменяет потомков. `cancel()` **идемпотентен** и **останавливает таймер**. `WithValue` — только для request-scoped данных. `cancel()` → `<-ctx.Done()` устанавливает happens-before (как `close` канала). Комбинируется с `select`, `Mutex`, `WaitGroup`. В HTTP, БД, gRPC, Kafka — `Context` пробрасывается. **`defer cancel()` — всегда.**