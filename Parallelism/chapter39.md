# 🔐 Глава 39: Безопасность конкурентного кода

**Что вы узнаете:**
- Почему конкурентный код — это **поверхность атаки**.
- Что такое **race condition** как уязвимость (не только баг).
- Что такое **TOCTOU** (Time-of-Check to Time-of-Use).
- Как **утечки данных** возникают через гонки.
- Что такое **DoS через конкурентность**: goroutine exhaustion, lock contention, thundering herd.
- Как защититься от **deadlock как атаки**.
- Что такое **side-channel атаки** через тайминг.
- Как безопасно работать с **shared state** в HTTP-серверах.
- Что такое **fencing tokens** в контексте безопасности.
- Как проводить **threat modeling** для конкурентного кода.

**После прочтения вы сможете:**
- Понимать, где конкурентный код уязвим.
- Защищаться от TOCTOU-атак.
- Предотвращать DoS через конкурентность.
- Защищать shared state в HTTP-серверах.
- Проводить threat modeling для конкурентного кода.
- Избегать типичных уязвимостей.

---

## Содержание

- [39.0 Пролог: гонка, которая стала уязвимостью](#390-пролог-гонка-которая-стала-уязвимостью)
- [39.1 Почему конкурентный код — поверхность атаки](#391-почему-конкурентный-код--поверхность-атаки)
- [39.2 Race condition как уязвимость](#392-race-condition-как-уязвимость)
- [39.3 TOCTOU: Time-of-Check to Time-of-Use](#393-toctou-time-of-check-to-time-of-use)
- [39.4 Утечки данных через гонки](#394-утечки-данных-через-гонки)
- [39.5 DoS через конкурентность](#395-dos-через-конкурентность)
- [39.6 Deadlock как атака](#396-deadlock-как-атака)
- [39.7 Side-channel атаки через тайминг](#397-side-channel-атаки-через-тайминг)
- [39.8 Защита shared state в HTTP-серверах](#398-защита-shared-state-в-http-серверах)
- [39.9 Fencing tokens и split-brain](#399-fencing-tokens-и-split-brain)
- [39.10 Threat modeling для конкурентного кода](#3910-threat-modeling-для-конкурентного-кода)
- [39.11 В связке с другими паттернами](#3911-в-связке-с-другими-паттернами)
- [39.12 Практика Go: защита от race condition](#3912-практика-go-защита-от-race-condition)
- [39.13 Выводы и типичные ошибки](#3913-выводы-и-типичные-ошибки)
- [39.14 Для быстрого повторения](#3914-для-быстрого-повторения)
- [39.15 Вопросы для самопроверки](#3915-вопросы-для-самопроверки)
- [39.16 Ответы](#3916-ответы)
- [39.17 Куда идти дальше?](#3917-куда-идти-дальше)
- [39.18 Чек-лист](#3918-чек-лист)

---

## 39.0 Пролог: гонка, которая стала уязвимостью

У нас есть сервис, который выдаёт одноразовые купоны. Логика простая: пользователь запрашивает купон, мы проверяем, что купон **не использован**, и помечаем как использованный.

```go
type Coupon struct {
    Code     string
    Used     bool
    UsedBy   string
    UsedAt   time.Time
}

var (
    mu      sync.Mutex
    coupons map[string]*Coupon
)

func RedeemCoupon(code, userID string) error {
    mu.Lock()
    defer mu.Unlock()
    
    coupon, ok := coupons[code]
    if !ok {
        return errors.New("coupon not found")
    }
    if coupon.Used {
        return errors.New("coupon already used")
    }
    
    coupon.Used = true
    coupon.UsedBy = userID
    coupon.UsedAt = time.Now()
    return nil
}
```

Работает. Но заметили странность: **один купон использован дважды**. Двумя разными пользователями. В логах — оба запроса прошли проверку `coupon.Used == false`.

❓ **Что произошло?** Гонка **где-то вне** нашей функции. Сервис запущен в **5 репликах**. Каждая реплика имеет **свой** `coupons` map. Два пользователя попали на **разные реплики** — обе увидели `coupon.Used == false`.

Это **race condition** — и это **уязвимость**. Не баг. Злоумышленник может **специально** отправить параллельные запросы, чтобы использовать купон дважды.

**Что нужно:**

- **Распределённая блокировка** (Redis, etcd).
- **Идемпотентность** на уровне БД.
- **Уникальный индекс** на `Used`.

Это **безопасность конкурентного кода**. Именно об этом глава.

> **Мост к следующим главам:** это **финальная** глава. Она связывает всё, что мы разбирали: гонки (Глава 7), синхронизацию (Глава 4), распределённые системы (Глава 24).

---

## 39.1 Почему конкурентный код — поверхность атаки

Прежде чем разбирать конкретные атаки, поймём, **почему** конкурентный код — это **поверхность атаки**.

### Гонки — не только баг

**Обычно** гонку воспринимают как **баг**: программа падает, тест флакает, прод иногда глючит. Разработчик исправляет — и забывает.

**Но гонка — это уязвимость.** Злоумышленник может **специально** создать условия для гонки:

- Отправить **параллельные** запросы.
- Попасть на **разные** реплики.
- Создать **нагрузку**, которая увеличивает окно гонки.
- Использовать **тайминг** для эксплуатации.

### Три категории атак

**1. Логические атаки.**

- **Race condition** — обход проверок.
- **TOCTOU** — проверка и использование не атомарны.
- **Double-spend** — двойное использование ресурса.

**2. DoS-атаки.**

- **Goroutine exhaustion** — исчерпание горутин.
- **Lock contention** — блокировка мьютексов.
- **Thundering herd** — массовое пробуждение.

**3. Информационные атаки.**

- **Side-channel** — утечка через тайминг.
- **Утечки данных** — через race в shared state.

### Почему Go особенно уязвим

**Горутины дешёвые** — легко создать **много**. Это **плюс** для производительности и **минус** для безопасности: злоумышленник может создать **миллион** горутин и положить сервис.

**Гонки — undefined behavior** — их **сложно** найти. Race detector не находит все. На x86 гонка может **не проявляться**.

**Распределённые системы** — гонки между **репликами**. In-process mutex не помогает.

### Модель угроз

**Активы:**

- **Данные** — пользователи, заказы, платежи.
- **Ресурсы** — CPU, память, соединения.
- **Деньги** — купоны, балансы, скидки.

**Угрозы:**

- **Обход логики** — использование купона дважды.
- **DoS** — положить сервис.
- **Утечка** — увидеть чужие данные.

**Злоумышленник:**

- **Внешний** — через HTTP-API.
- **Внутренний** — другой микросервис.
- **Скомпрометированный** — угнанный токен.

### 💡 Практика: как думать о безопасности

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Помни: гонка — это уязвимость.**
2. **Думай о злоумышленнике**, который может создать нагрузку.
3. **Проверяй распределённые системы** отдельно.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Threat modeling** для конкурентного кода.
5. **Идемпотентность** в критичных операциях.

**❌ НЕ ДЕЛАЙ:**

6. **Не полагайся на in-process mutex** для распределённых систем.
7. **Не игнорируй race detector.**

---

## 39.2 Race condition как уязвимость

**Race condition** — это не только баг, но и **уязвимость**.

### Пример: двойное списание баланса

```go
type Account struct {
    Balance int64
}

func Withdraw(account *Account, amount int64) error {
    if account.Balance < amount {  // ← CHECK
        return errors.New("insufficient funds")
    }
    account.Balance -= amount       // ← USE
    return nil
}
```

**Атака:** злоумышленник отправляет **два** параллельных запроса на списание 100 у.е., когда на балансе 150 у.е.

**Что происходит:**

```
Горутина A: читает Balance = 150
Горутина B: читает Balance = 150
Горутина A: 150 >= 100 → OK
Горутина B: 150 >= 100 → OK
Горутина A: Balance = 50
Горутина B: Balance = -50   ← ушли в минус!
```

**Результат:** списано 200 у.е. вместо 150. Баланс отрицательный.

### Пример: обход лимита

```go
var requests atomic.Int64

func handler(w http.ResponseWriter, r *http.Request) {
    if requests.Load() >= 100 {  // ← CHECK
        http.Error(w, "limit exceeded", 429)
        return
    }
    requests.Add(1)               // ← USE
    // обработка
}
```

**Атака:** злоумышленник отправляет 1000 параллельных запросов. Они **все** видят `requests.Load() == 0`, все проходят проверку, все инкрементируют.

**Результат:** лимит 100 обойдён.

### Пример: обход капчи

```go
var captchaTokens = make(map[string]bool)

func verifyCaptcha(token string) bool {
    if !captchaTokens[token] {   // ← CHECK
        return false
    }
    delete(captchaTokens, token)  // ← USE
    return true
}
```

**Атака:** злоумышленник отправляет **два** запроса с одним токеном параллельно. Оба видят `captchaTokens[token] == true`, оба проходят.

**Результат:** капча обойдена.

### Как защититься

**1. Атомарные операции.**

```go
func Withdraw(account *Account, amount int64) error {
    for {
        old := atomic.LoadInt64(&account.Balance)
        if old < amount {
            return errors.New("insufficient funds")
        }
        if atomic.CompareAndSwapInt64(&account.Balance, old, old-amount) {
            return nil
        }
        // retry
    }
}
```

**Что даёт:** проверка и изменение — **атомарно**.

**2. Мьютекс.**

```go
func Withdraw(account *Account, amount int64) error {
    mu.Lock()
    defer mu.Unlock()
    
    if account.Balance < amount {
        return errors.New("insufficient funds")
    }
    account.Balance -= amount
    return nil
}
```

**Что даёт:** вся секция — **атомарна**.

**3. Идемпотентность в БД.**

```sql
UPDATE accounts 
SET balance = balance - 100 
WHERE id = 42 AND balance >= 100;
-- Если 0 строк — недостаточно средств
```

**Что даёт:** БД гарантирует атомарность.

**4. Distributed lock.**

Для **распределённых** систем — Redis, etcd.

### Схема

```
CHECK ─┐
       ├── Окно гонки
USE  ──┘

Если злоумышленник успел в это окно — уязвимость.
```

### 💡 Практика: как защищаться от race condition

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Атомарные операции** для критичных данных.
2. **Мьютекс** для сложных секций.
3. **Идемпотентность** в БД.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Distributed lock** для распределённых.

**❌ НЕ ДЕЛАЙ:**

5. **Не разделяй CHECK и USE** без синхронизации.
6. **Не полагайся на in-process mutex** для распределённых.

---

## 39.3 TOCTOU: Time-of-Check to Time-of-Use

**TOCTOU** — **частный случай** race condition. **Проверка** и **использование** — **разделены** во времени.

### Классический пример: файл

```go
func processFile(path string) error {
    // CHECK
    if _, err := os.Stat(path); err != nil {
        return err
    }
    if !isAllowed(path) {  // ← CHECK
        return errors.New("access denied")
    }
    
    // ... время проходит ...
    
    // USE
    data, err := os.ReadFile(path)  // ← USE
    if err != nil {
        return err
    }
    // обработка
    return nil
}
```

**Атака:**

1. Злоумышленник создаёт **разрешённый** файл.
2. Программа проходит проверку `isAllowed`.
3. **Между** проверкой и чтением злоумышленник **заменяет** файл на **запрещённый** (symlink).
4. Программа читает **запрещённый** файл.

**Результат:** обход проверки.

### Пример: кэш

```go
var cache sync.Map

func get(key string) (string, bool) {
    if val, ok := cache.Load(key); ok {  // ← CHECK
        return val.(string), true
    }
    
    // ... время проходит ...
    
    // USE
    val := fetchFromDB(key)  // ← USE
    cache.Store(key, val)
    return val, true
}
```

**Атака:** злоумышленник создаёт **много** параллельных запросов на **один** ключ.

**Что происходит:**

- 1000 горутин **все** не находят ключ в кэше.
- **Все** идут в БД.
- БД **перегружена**.
- Это **thundering herd** (см. 39.5).

### Пример: HTTP-запрос

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // CHECK
    if !isAuthorized(r) {
        http.Error(w, "unauthorized", 401)
        return
    }
    
    // ... время проходит ...
    
    // USE
    data := fetchData(r.URL.Query().Get("id"))  // ← USE
    w.Write(data)
}
```

**Атака:**

1. Злоумышленник **аутентифицируется**.
2. Отправляет запрос.
3. **Между** проверкой и использованием — токен **отзывается**.
4. Запрос **всё равно** проходит.

**Результат:** использование отозванного токена.

### Как защититься

**1. Атомарность.**

Проверка и использование — **в одной критической секции**.

```go
func processFile(path string) error {
    mu.Lock()
    defer mu.Unlock()
    
    if !isAllowed(path) {
        return errors.New("access denied")
    }
    data, err := os.ReadFile(path)
    if err != nil {
        return err
    }
    // обработка
    return nil
}
```

**2. Открытие файла один раз.**

```go
func processFile(path string) error {
    f, err := os.Open(path)  // ← один раз
    if err != nil {
        return err
    }
    defer f.Close()
    
    // Проверяем через f
    stat, _ := f.Stat()
    if !isAllowed(stat) {
        return errors.New("access denied")
    }
    
    data, _ := io.ReadAll(f)
    // обработка
    return nil
}
```

**3. Использование токена один раз.**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Атомарно: проверить + пометить использованным
    if !consumeToken(r) {
        http.Error(w, "unauthorized", 401)
        return
    }
    // ...
}
```

**4. Singleflight для кэша.**

```go
import "golang.org/x/sync/singleflight"

var (
    group singleflight.Group
    cache sync.Map
)

func get(key string) (string, bool) {
    if val, ok := cache.Load(key); ok {
        return val.(string), true
    }
    
    val, _, _ := group.Do(key, func() (interface{}, error) {
        v := fetchFromDB(key)
        cache.Store(key, v)
        return v, nil
    })
    return val.(string), true
}
```

**Что даёт:** 1000 параллельных запросов — **один** запрос к БД.

### Схема

```
CHECK ──── окно TOCTOU ──── USE
          │
          │ злоумышленник
          │ меняет состояние
          ▼
       USE с невалидным состоянием
```

### 💡 Практика: как защищаться от TOCTOU

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Атомарность** CHECK и USE.
2. **Singleflight** для кэша.
3. **Одноразовые токены** — атомарно.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Открытие файла один раз**.
5. **Идемпотентность** в критичных операциях.

**❌ НЕ ДЕЛАЙ:**

6. **Не разделяй CHECK и USE** без синхронизации.
7. **Не открывай файл дважды.**

---

## 39.4 Утечки данных через гонки

**Гонки** могут привести к **утечке данных**.

### Пример: shared buffer

```go
var buf = make([]byte, 1024)

func process(data []byte) {
    copy(buf, data)        // ← запись в shared buffer
    result := transform(buf)  // ← чтение из shared buffer
    send(result)
}

func sendToClient(w http.ResponseWriter) {
    w.Write(buf)           // ← чтение из shared buffer
}
```

**Атака:**

- Горутина A **обрабатывает** данные пользователя A.
- Горутина B **отправляет** данные пользователя B.
- Между `copy` и `transform` — горутина B читает **частично** записанные данные.
- **Утечка** данных A к пользователю B.

### Пример: кэш с общим ключом

```go
type Cache struct {
    mu   sync.Mutex
    data map[string][]byte
}

func (c *Cache) Get(key string) []byte {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.data[key]  // ← возвращает slice (ссылку)
}
```

**Атака:**

- Горутина A получает `slice` через `Get`.
- Горутина B **мутирует** тот же `slice`.
- A видит **изменённые** данные.

### Пример: race в логировании

```go
var logEntry LogEntry

func handleRequest(r *http.Request) {
    logEntry.UserID = extractUserID(r)     // ← запись
    logEntry.IP = r.RemoteAddr              // ← запись
    logEntry.Path = r.URL.Path              // ← запись
    logEntry.Timestamp = time.Now()         // ← запись
    logToFile(logEntry)                     // ← чтение
}
```

**Атака:**

- Горутина A пишет `UserID` для запроса A.
- Горутина B **перезаписывает** `UserID` для запроса B.
- A читает **чужой** `UserID`.
- **Утечка** пользовательских данных.

### Как защититься

**1. Локальные буферы.**

```go
func process(data []byte) {
    buf := make([]byte, len(data))  // ← локальный
    copy(buf, data)
    result := transform(buf)
    send(result)
}
```

**2. Копирование при возврате.**

```go
func (c *Cache) Get(key string) []byte {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    data := c.data[key]
    result := make([]byte, len(data))
    copy(result, data)
    return result
}
```

**3. Локальные структуры.**

```go
func handleRequest(r *http.Request) {
    entry := LogEntry{  // ← локальная
        UserID:    extractUserID(r),
        IP:        r.RemoteAddr,
        Path:      r.URL.Path,
        Timestamp: time.Now(),
    }
    logToFile(entry)
}
```

**4. Race detector.**

Найти гонки **до** production.

### Схема

```
Горутина A: пишет данные A в shared buffer
Горутина B: читает shared buffer (данные A + B)

Результат: пользователь B видит данные A.
```

### 💡 Практика: как защищаться от утечек

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Локальные буферы** вместо shared.
2. **Копирование** при возврате slice.
3. **Race detector** в CI.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Локальные структуры** в хендлерах.

**❌ НЕ ДЕЛАЙ:**

5. **Не используй shared buffer** без синхронизации.
6. **Не возвращай slice** из кэша без копии.

---

## 39.5 DoS через конкурентность

**DoS** (Denial of Service) — атака, которая делает сервис **недоступным**.

### Атака 1: Goroutine exhaustion

**Что происходит:** злоумышленник создаёт **много** долгоживущих горутин.

**Пример:**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    go func() {
        time.Sleep(1 * time.Hour)  // ← висит час
    }()
    w.WriteHeader(http.StatusOK)
}
```

**Атака:** злоумышленник отправляет **1 000 000** запросов. Сервис создаёт 1 000 000 горутин. **OOM**.

**Что защищает:**

- **Rate limiter** (Глава 14).
- **Bulkhead** (Глава 22).
- **Semaphore** (Глава 12).
- **Context с таймаутом**.

**Пример защиты:**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()
    
    select {
    case sem <- struct{}{}:
        defer func() { <-sem }()
    case <-ctx.Done():
        http.Error(w, "timeout", 503)
        return
    }
    
    // обработка
}
```

### Атака 2: Lock contention

**Что происходит:** злоумышленник создаёт **нагрузку** на один мьютекс.

**Пример:**

```go
var mu sync.Mutex

func handler(w http.ResponseWriter, r *http.Request) {
    mu.Lock()
    defer mu.Unlock()
    
    // долгая операция под мьютексом
    time.Sleep(100 * time.Millisecond)
}
```

**Атака:** злоумышленник отправляет 1000 параллельных запросов. Все **ждут** мьютекс. Latency растёт.

**Что защищает:**

- **Короткие критические секции.**
- **Разделение мьютексов** (sharding, Глава 29).
- **Bulkhead** для изоляции.

### Атака 3: Thundering herd

**Что происходит:** **много** горутин просыпаются **одновременно**.

**Пример:**

```go
var data atomic.Value

func get(key string) string {
    if v := data.Load(); v != nil {
        return v.(string)
    }
    
    // 1000 горутин одновременно сюда
    v := fetchFromDB(key)
    data.Store(v)
    return v
}
```

**Атака:** злоумышленник отправляет 1000 запросов на **один** ключ. Все 1000 идут в БД.

**Что защищает:**

- **Singleflight** (Глава 20).

```go
var group singleflight.Group

func get(key string) string {
    v, _, _ := group.Do(key, func() (interface{}, error) {
        return fetchFromDB(key), nil
    })
    return v.(string)
}
```

**Что даёт:** 1000 горутин — **один** запрос к БД.

### Атака 4: Unbounded queue

**Что происходит:** канал **растёт** быстрее, чем consumer.

**Пример:**

```go
tasksCh := make(chan Task, 1000000)  // ← огромный буфер
```

**Атака:** злоумышленник заваливает producer'а. Буфер **растёт**. Память **растёт**. **OOM**.

**Что защищает:**

- **Маленький буфер** (Глава 27).
- **Backpressure.**

### Атака 5: Slow loris

**Что происходит:** злоумышленник **медленно** шлёт данные.

**Пример:** HTTP-сервер без `ReadTimeout`.

**Атака:** злоумышленник открывает 10 000 соединений. Шлёт 1 байт в минуту. Сервер **держит** соединения.

**Что защищает:**

- **`ReadTimeout`**.
- **`WriteTimeout`**.
- **`IdleTimeout`**.

```go
srv := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  60 * time.Second,
}
```

### Сводка DoS-атак

| Атака | Защита |
|:---|:---|
| Goroutine exhaustion | Rate limiter, bulkhead, semaphore |
| Lock contention | Короткие секции, sharding |
| Thundering herd | Singleflight |
| Unbounded queue | Маленький буфер, backpressure |
| Slow loris | Timeouts |

### 💡 Практика: как защищаться от DoS

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Rate limiter** для всех публичных эндпоинтов.
2. **Semaphore** или **bulkhead** для ограничения параллелизма.
3. **Timeout** для всех операций.
4. **Singleflight** для кэша.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Маленький буфер** для каналов.
6. **Короткие критические секции.**
7. **`ReadTimeout`, `WriteTimeout`, `IdleTimeout`.**

**❌ НЕ ДЕЛАЙ:**

6. **Не создавай горутины без ограничения.**
7. **Не используй огромные буферы.**
8. **Не оставляй эндпоинты без rate limit.**

---

## 39.6 Deadlock как атака

**Deadlock** — ситуация, когда горутины **ждут** друг друга **вечно**. Это **баг** и **уязвимость**.

### Атака: блокировка мьютексов

**Что происходит:** злоумышленник создаёт **порядок** захвата мьютексов, который ведёт к deadlock.

**Пример:**

```go
var mu1, mu2 sync.Mutex

func handlerA() {
    mu1.Lock()
    defer mu1.Unlock()
    mu2.Lock()  // ← если другой держит mu2 и ждёт mu1
    defer mu2.Unlock()
}

func handlerB() {
    mu2.Lock()
    defer mu2.Unlock()
    mu1.Lock()
    defer mu1.Unlock()
}
```

**Атака:** злоумышленник отправляет **параллельно** запросы на `handlerA` и `handlerB`. **Deadlock**.

### Атака: блокировка каналов

**Что происходит:** горутина **пишет** в канал, никто **не читает**.

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ch := make(chan int)
    go func() {
        ch <- 42  // ← если никто не читает — вечная блокировка
    }()
    // обработка
}
```

**Атака:** злоумышленник создаёт **много** таких запросов. Горутины **утекают**.

### Атака: context без таймаута

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := context.Background()  // ← без таймаута
    result := slowOperation(ctx)  // ← может висеть вечно
    w.Write(result)
}
```

**Атака:** злоумышленник создаёт **много** запросов к медленному сервису. Всё **висит**.

### Что защищает

**1. Порядок захвата мьютексов.**

Всегда **один** порядок.

```go
func handlerA() {
    mu1.Lock()
    defer mu1.Unlock()
    mu2.Lock()
    defer mu2.Unlock()
}

func handlerB() {
    mu1.Lock()  // ← тот же порядок
    defer mu1.Unlock()
    mu2.Lock()
    defer mu2.Unlock()
}
```

**2. Таймауты.**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()
    
    result := slowOperation(ctx)
    w.Write(result)
}
```

**3. Буферизованные каналы.**

```go
ch := make(chan int, 1)
go func() {
    ch <- 42  // ← не блокируется
}()
```

**4. `defer` для `Unlock`.**

```go
mu.Lock()
defer mu.Unlock()  // ← гарантирует освобождение
```

**5. Deadlock detection.**

- **`runtime`** — обнаруживает deadlock при `all goroutines are asleep`.
- **Мониторинг** — рост горутин в `semacquire`.

### Схема

```
Горутина A: держит mu1, ждёт mu2
Горутина B: держит mu2, ждёт mu1

Deadlock.
```

### 💡 Практика: как защищаться от deadlock

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Один порядок захвата мьютексов.**
2. **`defer` для `Unlock`.**
3. **Timeout для всех операций.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Буферизованные каналы** для «отправь и забудь».
5. **Мониторинг goroutine dump.**

**❌ НЕ ДЕЛАЙ:**

6. **Не захватывай мьютексы в разном порядке.**
7. **Не блокируйся без таймаута.**

---

## 39.7 Side-channel атаки через тайминг

**Side-channel атака** — утечка информации через **время** выполнения.

### Атака: timing attack на сравнение

```go
func checkPassword(input, stored string) bool {
    if len(input) != len(stored) {
        return false  // ← быстро
    }
    for i := 0; i < len(input); i++ {
        if input[i] != stored[i] {
            return false  // ← медленно (постепенно)
        }
    }
    return true  // ← медленно
}
```

**Атака:** злоумышленник измеряет **время** ответа. По времени узнаёт, **сколько** символов угадал.

**Результат:** подбор пароля по **таймингу**.

### Атака: timing attack на БД

```go
func login(username, password string) error {
    user := db.FindUser(username)  // ← быстрее, если пользователя нет
    if user == nil {
        return errors.New("invalid credentials")  // ← быстро
    }
    if !checkPassword(password, user.Hash) {
        return errors.New("invalid credentials")  // ← медленно
    }
    return nil
}
```

**Атака:** злоумышленник измеряет **время** ответа. По времени узнаёт, **существует** ли пользователь.

### Атака: timing attack на хеширование

**Что происходит:** злоумышленник отправляет запросы разного размера. По времени узнаёт **алгоритм** хеширования.

### Как защититься

**1. Constant-time сравнение.**

```go
import "crypto/subtle"

func checkPassword(input, stored []byte) bool {
    return subtle.ConstantTimeCompare(input, stored) == 1
}
```

**Что даёт:** время не зависит от совпадения.

**2. Одинаковое время ответа.**

```go
func login(username, password string) error {
    user := db.FindUser(username)
    if user == nil {
        // Имитируем проверку пароля
        _ = bcrypt.CompareHashAndPassword(dummyHash, []byte(password))
        return errors.New("invalid credentials")
    }
    if !checkPassword(password, user.Hash) {
        return errors.New("invalid credentials")
    }
    return nil
}
```

**Что даёт:** время ответа **одинаково**.

**3. Rate limiter.**

Ограничивает **скорость** подбора.

### Схема```
Запрос 1: время = 10 мс  → пароль начинается на 'a'
Запрос 2: время = 20 мс  → пароль начинается на 'ab'
Запрос 3: время = 30 мс  → пароль начинается на 'abc'

Злоумышленник подбирает пароль по времени.
```

### 💡 Практика: как защищаться от timing-атак

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Constant-time сравнение** для паролей, токенов.
2. **Одинаковое время ответа** для login.
3. **Rate limiter** для всех публичных эндпоинтов.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Логирование** подозрительных паттернов.

**❌ НЕ ДЕЛАЙ:**

5. **Не используй обычное `==`** для сравнения секретов.
6. **Не возвращай разные ошибки** для «пользователь не найден» и «неверный пароль».

---

## 39.8 Защита shared state в HTTP-серверах

**HTTP-серверы** — основное место для **shared state**. Разберём, как защищать.

### Проблема: глобальные переменные

**❌ Плохо:**

```go
var (
    requestCount int      // ← гонка
    lastUser     string   // ← гонка
)

func handler(w http.ResponseWriter, r *http.Request) {
    requestCount++
    lastUser = extractUser(r)
}
```

**Атака:** злоумышленник создаёт нагрузку. Гонки **проявляются**.

### Решение 1: atomic для счётчиков

```go
var (
    requestCount atomic.Int64
    lastUser     atomic.Value
)

func handler(w http.ResponseWriter, r *http.Request) {
    requestCount.Add(1)
    lastUser.Store(extractUser(r))
}
```

### Решение 2: мьютекс для структур

```go
var (
    mu   sync.RWMutex
    data map[string]string
)

func handler(w http.ResponseWriter, r *http.Request) {
    mu.RLock()
    val := data[r.URL.Path]
    mu.RUnlock()
    
    w.Write([]byte(val))
}
```

### Решение 3: локальные переменные

**Лучшее решение** — **избегать** shared state.

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Все данные — локальные
    user := extractUser(r)
    path := r.URL.Path
    
    // Обработка
    result := process(user, path)
    w.Write(result)
}
```

### Решение 4: context для request-scoped

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    ctx = context.WithValue(ctx, userIDKey, extractUserID(r))
    
    result := process(ctx)
    w.Write(result)
}
```

### Защита HTTP-сервера: полный список

```go
srv := &http.Server{
    Addr:           ":8080",
    Handler:        middleware(mux),
    ReadTimeout:    5 * time.Second,
    WriteTimeout:   10 * time.Second,
    IdleTimeout:    60 * time.Second,
    MaxHeaderBytes: 1 << 20,  // 1 MB
}
```

**Middleware:**

```go
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 1. Rate limiter
        if !limiter.Allow() {
            http.Error(w, "too many requests", 429)
            return
        }
        
        // 2. Semaphore
        select {
        case sem <- struct{}{}:
            defer func() { <-sem }()
        case <-r.Context().Done():
            return
        default:
            http.Error(w, "server busy", 503)
            return
        }
        
        // 3. Timeout
        ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
        defer cancel()
        
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### Схема

```
Запрос
   │
   ▼
Rate limiter  ← защита от flood
   │
   ▼
Semaphore  ← ограничение параллелизма
   │
   ▼
Timeout  ← ограничение времени
   │
   ▼
Handler  ← локальный state
```

### 💡 Практика: как защищать HTTP-сервер

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Rate limiter** для всех эндпоинтов.
2. **Semaphore** для ограничения параллелизма.
3. **Timeout** для всех операций.
4. **Локальные переменные** вместо shared.

**👍 СТОИТ СДЕЛАТЬ:**

5. **`atomic` для счётчиков.**
6. **`RWMutex` для map/slice.**
7. **`ReadTimeout`, `WriteTimeout`, `IdleTimeout`.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй глобальные переменные** без синхронизации.
7. **Не оставляй эндпоинты без rate limit.**

---

## 39.9 Fencing tokens и split-brain

**Split-brain** — ситуация, когда **два лидера** одновременно. Это **серьёзная уязвимость**.

### Проблема

**Сценарий:**

1. Лидер 1 **захватил** ресурс.
2. Сеть **разделилась**.
3. Лидер 2 **тоже** захватил ресурс.
4. Оба **думают**, что они лидеры.
5. **Дублирование** операций.

**Пример:** синхронизация с внешним API. Оба лидера делают **одну** синхронизацию. Дубликаты данных.

### Что защищает: fencing tokens

**Fencing tokens** — **монотонно возрастающий** номер для действий лидера.

**Как работает:**

1. Лидер получает **token** при захвате.
2. Каждое действие помечается **token**.
3. Ресурс (БД) **проверяет** token.
4. Если token **меньше** последнего — **отклоняет**.

**Пример:**

```
t=0:   Лидер 1 захватил, получил token=1
t=10:  Лидер 2 захватил, получил token=2
t=11:  Лидер 1 пишет с token=1 → БД отклоняет (последний token=2)
```

**Результат:** старый лидер **не может** писать. Split-brain **нейтрализован**.

### Пример в Go

```go
type FencedWriter struct {
    db        *sql.DB
    token     int64
    resource  string
}

func (w *FencedWriter) Write(ctx context.Context, data []byte) error {
    result, err := w.db.ExecContext(ctx,
        "UPDATE resources SET data = $1, fencing_token = $2 WHERE name = $3 AND fencing_token < $2",
        data, w.token, w.resource,
    )
    if err != nil {
        return err
    }
    rows, _ := result.RowsAffected()
    if rows == 0 {
        return errors.New("fencing token too old")
    }
    return nil
}
```

**Что даёт:** БД **проверяет** token. Старый лидер **не может** писать.

### Как получить token

**Через etcd/consul:**

```go
// etcd возвращает revision при захвате
resp, _ := election.Campaign(ctx, "replica-1")
token := resp.Header.Revision  // ← монотонно возрастающий
```

**Через БД:**

```sql
UPDATE leadership
SET holder = $1, token = token + 1
WHERE resource = $2 AND (holder = $1 OR token < $3)
RETURNING token;
```

### Quorum

**Quorum** — большинство голосов.

**Пример:** 5 узлов. Quorum = 3. Сеть разделилась на 2 + 3. Лидер — только в группе из 3.

**Что защищает:** невозможность двух лидеров в **большинстве**.

### Lease

**Lease** — лидер работает **только** пока TTL активен.

**Пример:**

```go
if !le.IsLeader() {
    return ErrLostLeadership
}
doWork()
```

**Что защищает:** лидер **немедленно** останавливается при потере лидерства.

### Схема

```
Лидер 1: token=1, пишет → OK
Лидер 2: token=2, пишет → OK
Лидер 1: token=1, пишет → ОТКЛОНЕНО (уже видели token=2)

Split-brain нейтрализован.
```

### 💡 Практика: как защищаться от split-brain

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Fencing tokens** для критичных операций.
2. **Quorum** для выбора лидера.
3. **Lease** — лидер работает только пока TTL активен.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Мониторинг split-brain.**
5. **Логирование переходов лидерства.**

**❌ НЕ ДЕЛАЙ:**

6. **Не работай без fencing** для критичных операций.
7. **Не полагайся только на TTL.**

---

## 39.10 Threat modeling для конкурентного кода

**Threat modeling** — систематический анализ угроз.

### Шаг 1: определи активы

**Что защищаем:**

- **Данные** — PII, платежи, заказы.
- **Ресурсы** — CPU, память, соединения.
- **Логика** — купоны, балансы, лимиты.

### Шаг 2: определи злоумышленника

**Кто может атаковать:**

- **Внешний пользователь** — через HTTP-API.
- **Внутренний сервис** — через gRPC.
- **Скомпрометированный** — угнанный токен.

### Шаг 3: определи угрозы

**Что может сделать злоумышленник:**

| Угроза | Актив | Защита |
|:---|:---|:---|
| **Double-spend** | Деньги | Идемпотентность |
| **Bypass limit** | Логика | Atomic |
| **DoS** | Ресурсы | Rate limit, semaphore |
| **Data leak** | Данные | Изоляция |
| **Split-brain** | Логика | Fencing tokens |

### Шаг 4: определи точки входа

**Где злоумышленник может влиять:**

- **HTTP-эндпоинты** — публичные.
- **gRPC-методы** — внутренние.
- **Очереди** — Kafka, NATS.
- **Файлы** — конфигурация.
- **ENV** — переменные окружения.

### Шаг 5: определи защиту

**Для каждой угрозы:**

1. **Идентификация** — как узнать атаку.
2. **Предотвращение** — как не допустить.
3. **Реакция** — что делать при атаке.

### Пример: threat modeling для купонов

**Активы:**

- Купоны — деньги.
- `coupons` map — shared state.
- HTTP-эндпоинт `/redeem`.

**Злоумышленник:**

- Внешний пользователь.
- Может отправлять **параллельные** запросы.
- Может попасть на **разные** реплики.

**Угрозы:**

1. **Double-spend купона** — использование дважды.
2. **Race condition** — на `coupon.Used`.
3. **DoS** — заваливание запросами.

**Точки входа:**

- HTTP-эндпоинт `/redeem`.

**Защита:**

1. **Идемпотентность** — уникальный индекс на `Used`.
2. **Distributed lock** — Redis, etcd.
3. **Rate limiter** — ограничить скорость.

### Схема

```
┌─────────────────────────────────────────┐
│  Threat Modeling                         │
│                                          │
│  1. Активы: деньги, данные, ресурсы     │
│  2. Злоумышленник: внешний, внутренний  │
│  3. Угрозы: double-spend, DoS, leak     │
│  4. Точки входа: HTTP, gRPC, очереди    │
│  5. Защита: идемпотентность, rate limit │
│                                          │
└─────────────────────────────────────────┘
```

### 💡 Практика: как делать threat modeling

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Определи активы.**
2. **Определи злоумышленника.**
3. **Определи угрозы.**
4. **Определи защиту.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Регулярно пересматривай.**
6. **Документируй.**

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй распределённые системы.**

---

## 39.11 В связке с другими паттернами

Безопасность конкурентного кода **комбинируется** с другими паттернами.

### Безопасность + rate limiter

**Защита от DoS:**

```go
limiter := rate.NewLimiter(rate.Limit(100), 10)

func handler(w http.ResponseWriter, r *http.Request) {
    if !limiter.Allow() {
        http.Error(w, "too many requests", 429)
        return
    }
    // ...
}
```

### Безопасность + bulkhead

**Изоляция ресурсов:**

```go
usersBulkhead := NewBulkhead(20, 2*time.Second)
recsBulkhead := NewBulkhead(5, 10*time.Second)
```

Медленный ML **не занимает** воркеров для `/users`.

### Безопасность + circuit breaker

**Защита от каскадных отказов:**

```go
cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "external-api",
    MaxRequests: 3,
    Timeout:     30 * time.Second,
})
```

### Безопасность + fencing tokens

**Защита от split-brain:**

```go
func (w *FencedWriter) Write(ctx context.Context, data []byte) error {
    result, _ := w.db.ExecContext(ctx,
        "UPDATE resources SET data = $1, token = $2 WHERE name = $3 AND token < $2",
        data, w.token, w.resource,
    )
    // ...
}
```

### Безопасность + constant-time

**Защита от timing-атак:**

```go
import "crypto/subtle"

if subtle.ConstantTimeCompare(input, stored) != 1 {
    return errors.New("invalid")
}
```

### Полная защита

```
Запрос
   │
   ▼
┌──────────────┐
│ Rate limiter │  ← от DoS
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Bulkhead    │  ← изоляция
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Timeout     │  ← ограничение времени
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Circuit     │  ← защита от сбоев
│  breaker     │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Idempotent   │  ← защита от double-spend
│   handler    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Fencing     │  ← защита от split-brain
│   tokens     │
└──────┬───────┘
       │
       ▼
     Database
```

### 💡 Практика: как комбинировать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Rate limiter** для DoS.
2. **Bulkhead** для изоляции.
3. **Timeout** для ограничения.
4. **Idempotent handler** для double-spend.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Circuit breaker** для внешних.
6. **Fencing tokens** для распределённых.
7. **Constant-time** для секретов.

**❌ НЕ ДЕЛАЙ:**

8. **Не забывай про безопасность.**

---

## 39.12 Практика Go: защита от race condition

Разберём **полный пример**: купоны с защитой.

### Уязвимый код

```go
package main

import (
    "errors"
    "fmt"
    "sync"
)

type Coupon struct {
    Code   string
    Used   bool
    UsedBy string
}

var (
    mu      sync.Mutex
    coupons = map[string]*Coupon{
        "ABC123": {Code: "ABC123"},
    }
)

func RedeemCoupon(code, userID string) error {
    mu.Lock()
    defer mu.Unlock()
    
    coupon, ok := coupons[code]
    if !ok {
        return errors.New("coupon not found")
    }
    if coupon.Used {
        return errors.New("coupon already used")
    }
    coupon.Used = true
    coupon.UsedBy = userID
    return nil
}

func main() {
    var wg sync.WaitGroup
    var redeemed int64
    
    for i := 0; i < 100; i++ {
        i := i
        wg.Add(1)
        go func() {
            defer wg.Done()
            err := RedeemCoupon("ABC123", fmt.Sprintf("user%d", i))
            if err == nil {
                redeemed++
            }
        }()
    }
    
    wg.Wait()
    fmt.Printf("Redeemed %d times\n", redeemed)
}
```

**Проблема:** 100 горутин, но `RedeemCoupon` **правильно** использует `Mutex`. **In-process** всё работает. **Но** в **распределённой** системе — 5 реплик — **каждая** имеет свой map.

### Защита через БД

```go
type CouponStore struct {
    db *sql.DB
}

func (s *CouponStore) Redeem(ctx context.Context, code, userID string) error {
    // Атомарно: пометить использованным, если ещё не использован
    result, err := s.db.ExecContext(ctx, `
        UPDATE coupons
        SET used = true, used_by = $1, used_at = NOW()
        WHERE code = $2 AND used = false
    `, userID, code)
    if err != nil {
        return err
    }
    
    rows, _ := result.RowsAffected()
    if rows == 0 {
        // Либо купон не существует, либо уже использован
        return errors.New("coupon not found or already used")
    }
    return nil
}
```

**Что даёт:**

- **Атомарность** на уровне БД.
- **Работает** в распределённой системе.
- **Уникальный индекс** на `code` гарантирует корректность.

### Защита через Redis

```go
type CouponStore struct {
    redis *redis.Client
}

func (s *CouponStore) Redeem(ctx context.Context, code, userID string) error {
    // SET NX — атомарно
    key := "coupon:" + code
    ok, err := s.redis.SetNX(ctx, key, userID, 0).Result()
    if err != nil {
        return err
    }
    if !ok {
        return errors.New("coupon already used")
    }
    return nil
}
```

**Что даёт:**

- **Атомарность** через `SET NX`.
- **Работает** в распределённой системе.

### Защита через fencing tokens

```go
type FencedCouponStore struct {
    db    *sql.DB
    token int64
}

func (s *FencedCouponStore) Redeem(ctx context.Context, code, userID string) error {
    result, err := s.db.ExecContext(ctx, `
        UPDATE coupons
        SET used = true, used_by = $1, used_at = NOW(), fencing_token = $2
        WHERE code = $3 AND used = false AND fencing_token < $2
    `, userID, s.token, code)
    if err != nil {
        return err
    }
    
    rows, _ := result.RowsAffected()
    if rows == 0 {
        return errors.New("coupon not found, already used, or stale leader")
    }
    return nil
}
```

**Что даёт:**

- **Идемпотентность** на уровне БД.
- **Защита от split-brain** через fencing token.

### Полная защита: middleware

```go
func secureMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 1. Rate limiter
        if !limiter.Allow() {
            http.Error(w, "too many requests", 429)
            return
        }
        
        // 2. Semaphore
        select {
        case sem <- struct{}{}:
            defer func() { <-sem }()
        case <-r.Context().Done():
            return
        default:
            http.Error(w, "server busy", 503)
            return
        }
        
        // 3. Timeout
        ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
        defer cancel()
        
        // 4. Idempotency key
        idempotencyKey := r.Header.Get("Idempotency-Key")
        if idempotencyKey != "" {
            if cached, ok := idempotencyCache.Get(idempotencyKey); ok {
                w.Write(cached.([]byte))
                return
            }
        }
        
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### 💡 Практика: как защищать критические операции

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Атомарность** на уровне БД.
2. **Идемпотентность** через уникальный индекс.
3. **Distributed lock** для распределённых.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Idempotency key** для HTTP.
5. **Fencing tokens** для split-brain.

**❌ НЕ ДЕЛАЙ:**

6. **Не полагайся только на in-process mutex.**
7. **Не разделяй CHECK и USE.**

---

## 39.13 Выводы и типичные ошибки

**Что мы узнали?**

Безопасность конкурентного кода — это **поверхность атаки**. **Race condition** — не только баг, но и уязвимость. **TOCTOU** — проверка и использование не атомарны. **Утечки** через shared buffer. **DoS** — goroutine exhaustion, lock contention, thundering herd, slow loris. **Deadlock** — через разный порядок захвата. **Side-channel** — timing-атаки. **Защита shared state** в HTTP-серверах — атомарность, локальные переменные. **Fencing tokens** — защита от split-brain. **Threat modeling** — систематический анализ.

**Типичные ошибки:**

- ❌ **Полагать, что гонка — только баг.** Это уязвимость.
- ❌ **Разделять CHECK и USE** без синхронизации. TOCTOU.
- ❌ **Использовать shared buffer** без синхронизации. Утечки.
- ❌ **Создавать горутины без ограничения.** Goroutine exhaustion.
- ❌ **Держать мьютекс долго.** Lock contention.
- ❌ **Не использовать singleflight.** Thundering herd.
- ❌ **Огромные буферы.** DoS через OOM.
- ❌ **Без ReadTimeout.** Slow loris.
- ❌ **Разный порядок захвата мьютексов.** Deadlock.
- ❌ **Обычное `==` для секретов.** Timing-атака.
- ❌ **In-process mutex для распределённых.** Не работает.
- ❌ **Без fencing tokens.** Split-brain.
- ❌ **Не делать threat modeling.**

---

## 39.14 Для быстрого повторения

- **Конкурентный код — поверхность атаки.**
- **Race condition** — уязвимость.
- **TOCTOU** — CHECK и USE не атомарны.
- **Утечки** — shared buffer.
- **DoS:** goroutine exhaustion, lock contention, thundering herd, slow loris.
- **Защита DoS:** rate limiter, bulkhead, semaphore, timeout.
- **Deadlock:** один порядок захвата, `defer`, timeout.
- **Side-channel:** constant-time, одинаковое время ответа.
- **Shared state:** atomic, mutex, локальные переменные.
- **HTTP-сервер:** rate limiter + semaphore + timeout.
- **Fencing tokens:** монотонные номера для split-brain.
- **Quorum:** большинство голосов.
- **Lease:** работает пока TTL активен.
- **Threat modeling:** активы, злоумышленник, угрозы, защита.
- **Идемпотентность:** уникальный индекс.
- **Distributed lock:** Redis, etcd.

---

## 39.15 Вопросы для самопроверки

1. Почему конкурентный код — поверхность атаки?
2. Что такое race condition как уязвимость?
3. Что такое TOCTOU? Пример?
4. Как утечки данных возникают через гонки?
5. Назови четыре DoS-атаки через конкурентность.
6. Как защититься от thundering herd?
7. Что такое deadlock как атака?
8. Что такое side-channel атака?
9. Как защищать shared state в HTTP-серверах?
10. Что такое fencing tokens? Зачем?
11. Что такое threat modeling?

---

## 39.16 Ответы

### Ответ 1

**Конкурентный код — поверхность атаки**, потому что:

- **Гонки** — уязвимости, не только баги.
- **Злоумышленник** может создать условия для гонки.
- **Горутины дешёвые** — легко создать миллион.
- **Распределённые системы** — гонки между репликами.

### Ответ 2

**Race condition как уязвимость:**

- **Double-spend** — двойное использование купона.
- **Bypass limit** — обход лимита.
- **Bypass captcha** — обход капчи.

**Злоумышленник** отправляет **параллельные** запросы.

### Ответ 3

**TOCTOU** — **Time-of-Check to Time-of-Use**. Проверка и использование **разделены** во времени.

**Пример:** программа проверяет файл, потом читает. Между этим злоумышленник **заменяет** файл.

### Ответ 4

**Утечки через гонки:**

- **Shared buffer** — данные одного пользователя видны другому.
- **Кэш без копии** — slice мутируется.
- **Глобальные переменные** — данные перезаписываются.

**Защита:** локальные переменные, копирование, race detector.

### Ответ 5

**Четыре DoS-атаки:**

1. **Goroutine exhaustion** — миллион горутин.
2. **Lock contention** — нагрузка на мьютекс.
3. **Thundering herd** — массовое пробуждение.
4. **Slow loris** — медленная отправка данных.

### Ответ 6

**Thundering herd** — защита через **singleflight**.

```go
v, _, _ := group.Do(key, func() (interface{}, error) {
    return fetchFromDB(key), nil
})
```

1000 горутин — **один** запрос.

### Ответ 7

**Deadlock как атака** — злоумышленник создаёт **порядок** захвата мьютексов, который ведёт к deadlock.

**Защита:** один порядок захвата, `defer`, timeout.

### Ответ 8

**Side-channel атака** — утечка через **время** выполнения.

**Пример:** `==` для паролей — время зависит от совпадения.

**Защита:** constant-time (`subtle.ConstantTimeCompare`).

### Ответ 9

**Защита shared state в HTTP-серверах:**

- **Rate limiter** для DoS.
- **Semaphore** для параллелизма.
- **Timeout** для операций.
- **Локальные переменные** вместо shared.
- **Atomic** для счётчиков.
- **RWMutex** для map/slice.

### Ответ 10

**Fencing tokens** — монотонно возрастающий номер для действий лидера. Защита от **split-brain**.

```sql
UPDATE resources
SET data = $1, token = $2
WHERE name = $3 AND token < $2;
```

Если 0 строк — старый лидер отклонён.

### Ответ 11

**Threat modeling:**

1. **Активы** — что защищаем.
2. **Злоумышленник** — кто атакует.
3. **Угрозы** — что может сделать.
4. **Точки входа** — где.
5. **Защита** — как.

---

## 39.17 Куда идти дальше?

Мы разобрали безопасность конкурентного кода. Это **финальная** глава книги.

**Что дальше:**

- **Применяй** знания в реальных проектах.
- **Проводи** threat modeling.
- **Запускай** `-race` в CI.
- **Мониторь** production.

**Приложения:**

- **Приложение A:** Шпаргалка по каналам.
- **Приложение B:** Шпаргалка по sync.
- **Приложение C:** Шпаргалка по context.
- **Приложение D:** Шпаргалка по планировщику.
- **Приложение E:** Инструменты и метрики.

---

## 39.18 Чек-лист

| Атака | Защита |
|:---|:---|
| **Race condition** | Atomic, mutex, идемпотентность |
| **TOCTOU** | Атомарность, singleflight |
| **Утечки данных** | Локальные переменные, копии |
| **Goroutine exhaustion** | Rate limiter, semaphore, bulkhead |
| **Lock contention** | Короткие секции, sharding |
| **Thundering herd** | Singleflight |
| **Slow loris** | Timeouts |
| **Deadlock** | Один порядок, `defer`, timeout |
| **Side-channel** | Constant-time |
| **Shared state HTTP** | Atomic, RWMutex, локальные |
| **Split-brain** | Fencing tokens, quorum, lease |
| **Threat modeling** | Активы, злоумышленник, угрозы |
| **Idempotency** | Уникальный индекс |
| **Distributed lock** | Redis, etcd |

🔐 **Ключевая идея:** Безопасность конкурентного кода — это **поверхность атаки**. **Race condition** — уязвимость, не только баг. **TOCTOU** — CHECK и USE не атомарны. **DoS** — goroutine exhaustion, lock contention, thundering herd, slow loris. **Защита DoS:** rate limiter, bulkhead, semaphore, timeout. **Deadlock** — через разный порядок захвата мьютексов. **Side-channel** — timing-атаки; constant-time. **Shared state** — atomic, RWMutex, локальные. **Fencing tokens** — защита от split-brain. **Threat modeling** — систематический анализ. **Идемпотентность** — уникальный индекс. **Distributed lock** — Redis, etcd. Не разделяй CHECK и USE. Не используй in-process mutex для распределённых. Запускай `-race` в CI. Делай threat modeling.