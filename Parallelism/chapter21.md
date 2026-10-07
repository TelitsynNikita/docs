# 🔒 Глава 21: Безопасность конкурентного кода

**Что вы узнаете:**
- Почему **race conditions** — это не только баг, но и **уязвимость**.
- Что такое **TOCTOU** (Time-of-Check to Time-of-Use) и как он эксплуатируется.
- Что такое **race condition в security** и как он приводит к **privilege escalation**.
- Что такое **timing attacks** и как они работают через конкурентный код.
- Что такое **DoS через горутины** (goroutine exhaustion).
- Что такое **memory exhaustion** через каналы и как его избежать.
- Что такое **side-channel attacks** через shared state.
- Как **логировать** безопасно.
- Как **использовать crypto** в конкурентном коде.
- Как **защититься** от атак через конкурентность.

**После прочтения вы сможете:**
- Распознавать TOCTOU в своём коде.
- Понимать, как race conditions приводят к уязвимостям.
- Защищаться от timing attacks.
- Ограничивать горутины против DoS.
- Ограничивать память через каналы.
- Безопасно работать с crypto в конкурентном коде.
- Логировать без утечки данных.
- Проводить security review конкурентного кода.

---

## Содержание

- [21.0 Пролог: баг, который стал уязвимостью](#210-пролог-баг-который-стал-уязвимостью)
- [21.1 Race conditions как уязвимость](#211-race-conditions-как-уязвимость)
- [21.2 TOCTOU: Time-of-Check to Time-of-Use](#212-toctou-time-of-check-to-time-of-use)
- [21.3 Race condition в security: примеры](#213-race-condition-в-security-примеры)
- [21.4 Timing attacks](#214-timing-attacks)
- [21.5 DoS через горутины](#215-dos-через-горутины)
- [21.6 Memory exhaustion через каналы](#216-memory-exhaustion-через-каналы)
- [21.7 Side-channel attacks через shared state](#217-side-channel-attacks-через-shared-state)
- [21.8 Безопасное логирование](#218-безопасное-логирование)
- [21.9 Crypto в конкурентном коде](#219-crypto-в-конкурентном-коде)
- [21.10 Практика Go: защита от DoS и TOCTOU](#2110-практика-go-защита-от-dos-и-toctou)
- [21.11 Выводы и типичные ошибки](#2111-выводы-и-типичные-ошибки)
- [21.12 Для быстрого повторения](#2112-для-быстрого-повторения)
- [21.13 Вопросы для самопроверки](#2113-вопросы-для-самопроверки)
- [21.14 Ответы](#2114-ответы)
- [21.15 Куда идти дальше?](#2115-куда-идти-дальше)
- [21.16 Чек-лист](#2116-чек-лист)

---

## 21.0 Пролог: баг, который стал уязвимостью

Ты пишешь сервис для **сброса пароля**. Логика:

```go
func resetPassword(userID int, token string, newPassword string) error {
    // 1. Проверяем токен
    valid, err := db.CheckToken(userID, token)
    if err != nil {
        return err
    }
    if !valid {
        return errors.New("invalid token")
    }

    // 2. Обновляем пароль
    return db.UpdatePassword(userID, newPassword)
}
```

Всё работает. Но **race condition**:

```
t=0:    Злоумышленник A с валидным токеном вызывает resetPassword(42, "valid", "hacked")
t=0:    Злоумышленник B с ТЕМ ЖЕ токеном вызывает resetPassword(42, "valid", "hacked2")

t=0:    A: CheckToken → valid
t=0:    B: CheckToken → valid

t=1:    A: UpdatePassword("hacked")
t=1:    B: UpdatePassword("hacked2")

Результат: оба запроса прошли. Токен использован дважды.
```

**Что произошло?** **TOCTOU** (Time-of-Check to Time-of-Use). Между **проверкой** токена и **использованием** — окно, где состояние меняется.

**Последствия:**

- **Одноразовый токен** использован дважды.
- **Злоумышленник** может сбросить пароль **дважды**.
- **Race condition → уязвимость.**

❓ **Что делать?** Проверка и использование должны быть **атомарными**.

```go
func resetPassword(userID int, token string, newPassword string) error {
    // Атомарная проверка + использование
    return db.ResetPasswordAtomic(userID, token, newPassword)
}
```

**SQL:**

```sql
UPDATE users
SET password = $1, reset_token = NULL
WHERE id = $2 AND reset_token = $3 AND reset_token_expires > NOW()
RETURNING id
```

**Что изменилось:** проверка и обновление — **одна** атомарная операция. Race condition невозможен.

**Это security через конкурентность.** В этой главе — как race conditions становятся уязвимостями, и как защищаться.

> **Важный мост:** Глава 4 (Memory model) — почему race conditions — UB. Глава 3 (Синхронизация) — как защищаться. Эта глава — **security-перспектива**. Глава 22-28 (Распределённые системы) — security в distributed.

---

## 21.1 Race conditions как уязвимость

Прежде чем разбирать TOCTOU, поймём **почему race conditions — уязвимость**.

### Что такое race condition

**Race condition** — ситуация, когда результат зависит от **порядка** выполнения операций.

**В конкурентном коде:**

```go
var balance int

func withdraw(amount int) bool {
    if balance >= amount {  // ← check
        balance -= amount   // ← use
        return true
    }
    return false
}
```

**Race condition:** между check и use — окно.

```
t=0:    Горутина A: balance = 100, check 100 >= 100 → true
t=0:    Горутина B: balance = 100, check 100 >= 100 → true
t=1:    Горутина A: balance -= 100 → 0
t=1:    Горутина B: balance -= 100 → -100

Результат: balance = -100. Деньги украдены.
```

### Race condition vs data race

**Data race** (Глава 4) — **технически** две горутины обращаются к одной памяти без синхронизации. **UB** по спецификации.

**Race condition** — **логически** результат зависит от порядка. Может быть **без** data race (если есть синхронизация, но неправильная).

**Пример race condition без data race:**

```go
var mu sync.Mutex
var balance int

func withdraw(amount int) bool {
    mu.Lock()
    if balance >= amount {
        mu.Unlock()  // ← отпустили
        mu.Lock()    // ← взяли снова
        balance -= amount
        mu.Unlock()
        return true
    }
    mu.Unlock()
    return false
}
```

**Data race:** нет (Mutex защищает).

**Race condition:** да (между `Unlock` и `Lock` — окно).

### Почему это уязвимость

**1. Финансовые потери.**

Двойное списание, отрицательный баланс.

**2. Privilege escalation.**

TOCTOU в проверке прав.

**3. Обход аутентификации.**

Race в проверке токена.

**4. Утечка данных.**

Race в проверке доступа.

**5. DoS.**

Race в лимитах.

### Как эксплуатируется

**1. Одновременные запросы.**

Злоумышленник отправляет **100 запросов** одновременно.

**2. Медленные операции.**

Если операция медленная — окно больше.

**3. Повторение.**

Race может не сработать с первого раза. Злоумышленник **повторяет**.

### Пример: двойное списание

```
1. Злоумышленник имеет $100.
2. Отправляет 100 запросов на списание $100 одновременно.
3. Race condition: несколько запросов проходят.
4. Баланс: -$10000.
```

### Пример: обход лимита

```go
func createAccount(userID int) error {
    count, _ := db.CountAccounts(userID)
    if count >= 10 {
        return errors.New("limit exceeded")
    }
    return db.CreateAccount(userID)
}
```

**Race:** 100 запросов одновременно. Все видят count < 10. Все создают аккаунт. Результат: 100 аккаунтов вместо 10.

### Аннотация сложности

| Тип race | Data race | Уязвимость |
|:---|:---|:---|
| Без синхронизации | ✅ Да | ✅ Да |
| С неправильной синхронизацией | ❌ Нет | ✅ Да |
| С правильной синхронизацией | ❌ Нет | ❌ Нет |

### 💡 Практика: как защититься от race conditions

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Атомарные операции** — check + use вместе.
2. **Транзакции** в БД.
3. **Уникальные индексы** в БД.
4. **`-race`** в CI.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Security review** — искать race.
6. **Stress-тесты** — одновременные запросы.

**❌ НЕ ДЕЛАЙ:**

7. **Не делай check и use раздельно.**
8. **Не полагайся на «редко срабатывает».**

### Ключевые выводы подглавы 21.1

- **Race condition** — результат зависит от порядка.
- **Может быть без data race.**
- **Уязвимость:** финансовые потери, privilege escalation, обход.
- **Эксплуатация:** одновременные запросы.
- **Защита:** атомарные операции, транзакции.

---

## 21.2 TOCTOU: Time-of-Check to Time-of-Use

**TOCTOU** — классическая уязвимость race condition.

### Что такое TOCTOU

**TOCTOU** — между **проверкой** (check) и **использованием** (use) состояние **меняется**.

**Схема:**

```
Check → [окно] → Use
```

**В окне** злоумышленник может изменить состояние.

### Пример: файл

**Уязвимый код:**

```go
func readFile(path string) ([]byte, error) {
    // 1. Проверяем права
    info, err := os.Stat(path)
    if err != nil {
        return nil, err
    }
    if info.Mode().Perm() != 0644 {
        return nil, errors.New("invalid permissions")
    }

    // 2. Открываем
    return os.ReadFile(path)
}
```

**Атака:**

```
t=0:    Проверка: file.txt имеет права 0644
t=0:    Злоумышленник: mv file.txt file.bak; ln -s /etc/passwd file.txt
t=1:    Открытие: os.ReadFile("file.txt") → читает /etc/passwd
```

**Что произошло:** между `Stat` и `ReadFile` — окно. Злоумышленник заменил файл на симлинк.

### Пример: сброс пароля

**Уязвимый код:**

```go
func resetPassword(userID int, token string) error {
    valid := db.CheckToken(userID, token)
    if !valid {
        return errors.New("invalid token")
    }
    db.DeleteToken(token)
    return db.UpdatePassword(userID, newPassword)
}
```

**Атака:** 100 одновременных запросов.

```
t=0:    Запрос A: CheckToken → valid
t=0:    Запрос B: CheckToken → valid
t=1:    Запрос A: DeleteToken
t=1:    Запрос B: DeleteToken
t=2:    Оба: UpdatePassword

Результат: пароль обновлён дважды. Токен использован дважды.
```

### Пример: создание аккаунта

**Уязвимый код:**

```go
func createUser(email string) error {
    exists := db.UserExists(email)
    if exists {
        return errors.New("user exists")
    }
    return db.CreateUser(email)
}
```

**Атака:** два одновременных запроса.

```
t=0:    Запрос A: UserExists("a@b.com") → false
t=0:    Запрос B: UserExists("a@b.com") → false
t=1:    Запрос A: CreateUser("a@b.com")
t=1:    Запрос B: CreateUser("a@b.com")

Результат: два пользователя с одним email.
```

### Как исправить

**1. Атомарные операции.**

```sql
-- Вместо:
-- SELECT * FROM users WHERE email = 'a@b.com';
-- if not exists: INSERT INTO users ...

-- Используй:
INSERT INTO users (email) VALUES ('a@b.com')
ON CONFLICT (email) DO NOTHING;
```

**Уникальный индекс** на `email` — гарантия.

**2. Транзакции с правильным isolation level.**

```sql
BEGIN;
SELECT ... FOR UPDATE;  -- блокировка
-- check
-- use
COMMIT;
```

**3. File operations: `O_EXCL`.**

```go
f, err := os.OpenFile(path, os.O_CREATE|os.O_EXCL|os.O_WRONLY, 0644)
```

**`O_EXCL`** — открыть, **только если не существует**.

**4. `os.Open` + `fstat`.**

```go
f, err := os.Open(path)
if err != nil {
    return err
}
defer f.Close()

info, err := f.Stat()  // stat уже открытого файла
if err != nil {
    return err
}
// проверка прав
```

**Ключевое:** `Stat` **открытого** файла, а не по пути.

### Аннотация сложности

| Атака | Окно | Защита |
|:---|:---|:---|
| Файл | ~мкс-мс | `O_EXCL`, `fstat` |
| Токен | ~мкс-мс | Атомарный SQL |
| Аккаунт | ~мкс-мс | Уникальный индекс |

### 💡 Практика: как защититься от TOCTOU

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Атомарные SQL** — `INSERT ON CONFLICT`, `UPDATE ... WHERE`.
2. **Уникальные индексы.**
3. **`O_EXCL`** для файлов.
4. **`fstat`** открытого файла.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Транзакции** с правильным isolation.
6. **Security review** на TOCTOU.

**❌ НЕ ДЕЛАЙ:**

7. **Не делай check и use раздельно.**
8. **Не полагайся на `Stat` по пути.**

### Ключевые выводы подглавы 21.2

- **TOCTOU** — окно между check и use.
- **Атаки:** файл, токен, аккаунт.
- **Защита:** атомарные операции, `O_EXCL`, `fstat`.
- **Уникальные индексы** — гарантия.

---

## 21.3 Race condition в security: примеры

Разберём **реальные** примеры.

### Пример 1: двойное списание

**Уязвимый код:**

```go
func withdraw(accountID int, amount int) error {
    balance := db.GetBalance(accountID)
    if balance < amount {
        return errors.New("insufficient funds")
    }
    return db.UpdateBalance(accountID, balance-amount)
}
```

**Атака:** 100 одновременных запросов на $100, когда баланс $100.

**Результат:** баланс -$9900.

**Защита:**

```sql
UPDATE accounts
SET balance = balance - $1
WHERE id = $2 AND balance >= $1;
```

**Проверка `balance >= $1`** — в **том же** SQL.

### Пример 2: обход лимита запросов

**Уязвимый код:**

```go
func checkRateLimit(userID int) error {
    count := redis.Get(fmt.Sprintf("rate:%d", userID))
    if count >= 100 {
        return errors.New("rate limit")
    }
    redis.Incr(fmt.Sprintf("rate:%d", userID))
    return nil
}
```

**Атака:** 1000 одновременных запросов. Все видят count < 100.

**Защита:**

```lua
-- Lua script (атомарный)
local count = redis.call("GET", KEYS[1])
if count and tonumber(count) >= 100 then
    return 0
end
redis.call("INCR", KEYS[1])
redis.call("EXPIRE", KEYS[1], 60)
return 1
```

**Lua** — атомарный.

### Пример 3: обход 2FA

**Уязвимый код:**

```go
func verify2FA(userID int, code string) error {
    valid := db.Check2FACode(userID, code)
    if !valid {
        return errors.New("invalid code")
    }
    db.Delete2FACode(userID)
    return db.Mark2FAVerified(userID)
}
```

**Атака:** 2 одновременных запроса с одним кодом.

**Защита:**

```sql
DELETE FROM two_factor_codes
WHERE user_id = $1 AND code = $2
RETURNING id;
```

**Атомарный** `DELETE ... RETURNING`.

### Пример 4: гонка в кэше

**Уязвимый код:**

```go
var cache = map[string]User{}

func getUser(id string) User {
    if u, ok := cache[id]; ok {
        return u
    }
    u := db.GetUser(id)
    cache[id] = u
    return u
}
```

**Уязвимости:**

- **Data race** на map (падение).
- **Утечка данных** — кэш без TTL.

**Защита:** `sync.Map` или `RWMutex` + TTL.

### Пример 5: гонка в JWT

**Уязвимый код:**

```go
var revokedTokens = map[string]bool{}

func verifyJWT(token string) error {
    if revokedTokens[token] {
        return errors.New("revoked")
    }
    // проверка подписи
    return nil
}
```

**Атака:** проверка revocation и использование — не атомарны.

**Защита:** атомарная проверка + использование.

### Сводная таблица

| Атака | Окно | Защита |
|:---|:---|:---|
| Двойное списание | check + use | Атомарный SQL |
| Rate limit | check + incr | Lua script |
| 2FA | check + delete | Атомарный SQL |
| Кэш | data race | `sync.Map` |
| JWT | check + use | Атомарная проверка |

### Аннотация сложности

| Атака | Time |
|:---|:---|
| 2FA | ~мс |
| Rate limit | ~мс |
| Двойное списание | ~мс |

### 💡 Практика: как защититься от security race

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Атомарные SQL** — `UPDATE ... WHERE`, `DELETE ... RETURNING`.
2. **Уникальные индексы.**
3. **Lua scripts** в Redis.
4. **`sync.Map`** или `RWMutex` для кэша.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Security review** на race.
6. **Stress-тесты** с одновременными запросами.

**❌ НЕ ДЕЛАЙ:**

7. **Не делай check и use раздельно.**
8. **Не полагайся на «редко срабатывает».**

### Ключевые выводы подглавы 21.3

- **Двойное списание** — атомарный SQL.
- **Rate limit** — Lua.
- **2FA** — `DELETE ... RETURNING`.
- **Кэш** — `sync.Map`.
- **JWT** — атомарная проверка.

---

## 21.4 Timing attacks

**Timing attack** — атака по **времени** выполнения.

### Что это

**Timing attack** — злоумышленник измеряет **время** операции и делает выводы о **данных**.

**Классический пример:** сравнение пароля.

```go
func checkPassword(input, stored string) bool {
    if len(input) != len(stored) {
        return false
    }
    for i := 0; i < len(input); i++ {
        if input[i] != stored[i] {
            return false  // ← ранний выход
        }
    }
    return true
}
```

**Уязвимость:** если первый символ не совпал — выход **быстрее**. Злоумышленник подбирает посимвольно.

### Атака

```
1. Злоумышленник пробует "a..." → быстро (первый символ не совпал)
2. Пробует "b..." → быстро
3. Пробует "p..." → медленнее (первый символ совпал)
4. Знает: пароль начинается с "p"
5. Повторяет для второго символа.
```

**Сложность:** O(n × alphabet) вместо O(alphabet^n).

### Constant-time comparison

**Решение:** сравнивать за **одинаковое** время.

```go
import "crypto/subtle"

func checkPassword(input, stored []byte) bool {
    return subtle.ConstantTimeCompare(input, stored) == 1
}
```

**`subtle.ConstantTimeCompare`** — сравнение за **фиксированное** время.

### Timing attacks в конкурентном коде

**1. Мьютекс с разным временем.**

```go
var mu sync.Mutex

func process(data []byte) {
    mu.Lock()
    defer mu.Unlock()
    if len(data) < 10 {
        return  // ← быстро
    }
    // долгая обработка
}
```

**Утечка:** длина данных.

**2. Канал с разным временем.**

```go
select {
case v := <-secretCh:
    if v == 42 {
        // ...
    }
case <-time.After(100 * time.Millisecond):
    // ← timeout
}
```

**Утечка:** наличие данных в канале.

**3. Cache timing.**

```go
var cache = map[string]User{}

func getUser(id string) User {
    if u, ok := cache[id]; ok {
        return u  // ← быстро
    }
    return db.GetUser(id)  // ← медленно
}
```

**Утечка:** наличие в кэше.

### Как защититься

**1. Constant-time сравнения.**

```go
subtle.ConstantTimeCompare(a, b)
subtle.ConstantTimeSelect(v, x, y)
subtle.ConstantTimeByteEq(x, y)
```

**2. Фиксированное время операций.**

Не делай ранних выходов в security-sensitive коде.

**3. Random delays.**

```go
time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
```

**Проблема:** может не помочь. Constant-time лучше.

**4. Одинаковый путь для всех.**

Все запросы идут по одному пути, независимо от данных.

### Аннотация сложности

| Операция | Time |
|:---|:---|
| Обычное сравнение | ~1 нс на байт |
| `subtle.ConstantTimeCompare` | ~1 нс на байт (но фиксировано) |
| Ранний выход | ~1 нс |
| Полное сравнение | ~n нс |

### 💡 Практика: как защититься от timing attacks

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`subtle.ConstantTimeCompare`** для паролей, токенов.
2. **Фиксированное время** для security-sensitive.
3. **Одинаковый путь** для всех.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Аудит** security-sensitive кода.
5. **Тесты** на timing.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `==` для паролей.**
7. **Не делай ранних выходов** в security.

### Ключевые выводы подглавы 21.4

- **Timing attack** — по времени.
- **Классика:** сравнение пароля.
- **`subtle.ConstantTimeCompare`** — защита.
- **Фиксированное время** для security.
- **Одинаковый путь** для всех.

---

## 21.5 DoS через горутины

**DoS через горутины** — атака, при которой злоумышленник создаёт **много** горутин.

### Проблема

**Горутины дешёвые** (~2.3 КБ). Но **миллионы** горутин — это:

- **Память:** 1 000 000 × 2.3 КБ = 2.3 ГБ.
- **GC:** обход миллиона горутин.
- **Планировщик:** переключение.
- **CPU:** конкуренция.

### Пример: HTTP-хэндлер

**Уязвимый код:**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    go process(r)  // ← неограниченно
    w.WriteHeader(http.StatusAccepted)
}
```

**Атака:** 1 000 000 запросов. 1 000 000 горутин.

**Результат:** OOM.

### Пример: WebSocket

**Уязвимый код:**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    conn, _ := websocket.Accept(w, r, nil)
    for {
        _, msg, err := conn.Read(ctx)
        if err != nil {
            return
        }
        go process(msg)  // ← неограниченно
    }
}
```

**Атака:** клиент отправляет 1 000 000 сообщений.

**Результат:** 1 000 000 горутин.

### Защита

**1. Worker pool.**

```go
var taskCh = make(chan Task, 1000)

func init() {
    for i := 0; i < 10; i++ {
        go worker(taskCh)
    }
}

func handler(w http.ResponseWriter, r *http.Request) {
    select {
    case taskCh <- Task{...}:
        w.WriteHeader(http.StatusAccepted)
    default:
        http.Error(w, "busy", http.StatusServiceUnavailable)
    }
}
```

**2. Semaphore.**

```go
var sem = make(chan struct{}, 100)

func handler(w http.ResponseWriter, r *http.Request) {
    select {
    case sem <- struct{}{}:
        defer func() { <-sem }()
    case <-time.After(1 * time.Second):
        http.Error(w, "busy", http.StatusServiceUnavailable)
        return
    }
    process(r)
}
```

**3. `errgroup.SetLimit`.**

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(100)

for _, task := range tasks {
    task := task
    g.Go(func() error {
        return process(ctx, task)
    })
}
```

**4. Лимит на соединения.**

```go
var connSem = make(chan struct{}, 10000)

func handler(w http.ResponseWriter, r *http.Request) {
    select {
    case connSem <- struct{}{}:
        defer func() { <-connSem }()
    default:
        http.Error(w, "too many connections", http.StatusServiceUnavailable)
        return
    }
    // ...
}
```

**5. Лимит на горутины.**

```go
func init() {
    go func() {
        for range time.Tick(1 * time.Second) {
            n := runtime.NumGoroutine()
            if n > 10000 {
                log.Printf("WARNING: %d goroutines", n)
            }
        }
    }()
}
```

### Мониторинг

**Метрики:**

- **`runtime.NumGoroutine()`** — число горутин.
- **Memory** — heap.
- **GC** — частота.

**Алерты:**

- **> 10 000 горутин** — warning.
- **> 100 000 горутин** — critical.

### Аннотация сложности

| Атака | Горутин | Память |
|:---|:---|:---|
| 1000 запросов | 1000 | 2.3 МБ |
| 1 000 000 запросов | 1 000 000 | 2.3 ГБ |
| OOM | — | — |

### 💡 Практика: как защититься от DoS через горутины

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Worker pool** для задач.
2. **Semaphore** для ограничений.
3. **`errgroup.SetLimit`.**
4. **Мониторинг** `NumGoroutine`.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Лимит на соединения.**
6. **Алерты** на аномалии.

**❌ НЕ ДЕЛАЙ:**

7. **Не создавай горутину на каждый запрос.**
8. **Не игнорируй рост горутин.**

### Ключевые выводы подглавы 21.5

- **DoS через горутины** — миллионы горутин.
- **OOM** через память.
- **Защита:** worker pool, semaphore, `SetLimit`.
- **Мониторинг** `NumGoroutine`.
- **Алерты** на аномалии.

---

## 21.6 Memory exhaustion через каналы

**Memory exhaustion** — атака через **заполнение каналов**.

### Проблема

**Каналы с буфером** занимают память.

```go
ch := make(chan []byte, 1000000)  // ← 1 000 000 × размер
```

**Если размер элемента 1 КБ:** 1 ГБ памяти.

### Пример: HTTP-хэндлер

**Уязвимый код:**

```go
var eventsCh = make(chan Event, 1000000)

func handler(w http.ResponseWriter, r *http.Request) {
    eventsCh <- parseEvent(r)  // ← неограниченно
}
```

**Атака:** 1 000 000 запросов. Канал заполнен. 1 ГБ памяти.

### Пример: WebSocket broadcast

**Уязвимый код:**

```go
func (h *Hub) broadcast(msg []byte) {
    for client := range h.clients {
        client.send <- msg  // ← блокировка, если клиент медленный
    }
}
```

**Атака:** медленный клиент. Буфер растёт.

### Защита

**1. Ограниченный буфер.**

```go
ch := make(chan Event, 1000)  // ← ограничено
```

**2. Non-blocking send.**

```go
select {
case ch <- event:
default:
    // Буфер полон — drop или error
}
```

**3. Drop oldest.**

```go
select {
case ch <- event:
default:
    <-ch  // удалить старый
    ch <- event
}
```

**4. Мониторинг длины канала.**

```go
go func() {
    for range time.Tick(1 * time.Second) {
        log.Printf("channel len: %d", len(eventsCh))
    }
}()
```

**5. Лимит на размер элемента.**

```go
const maxEventSize = 4096

func handler(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(io.LimitReader(r.Body, maxEventSize+1))
    if len(body) > maxEventSize {
        http.Error(w, "too large", http.StatusRequestEntityTooLarge)
        return
    }
    // ...
}
```

### Пример: защита в hub

```go
func (h *Hub) broadcast(msg []byte) {
    for client := range h.clients {
        select {
        case client.send <- msg:
            // OK
        default:
            // Буфер полон — disconnect
            close(client.send)
            delete(h.clients, client)
        }
    }
}
```

**Что делает:** если буфер полон — клиент **отключается**.

### Мониторинг

**Метрики:**

- **Длина канала** — `len(ch)`.
- **Емкость** — `cap(ch)`.
- **Заполненность** — `len(ch) / cap(ch)`.

**Алерты:**

- **> 80% заполнен** — warning.
- **100% в течение минуты** — critical.

### Аннотация сложности

| Атака | Память |
|:---|:---|
| 1000 событий | ~1 МБ |
| 1 000 000 событий | ~1 ГБ |
| OOM | — |

### 💡 Практика: как защититься от memory exhaustion

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Ограниченный буфер** канала.
2. **Non-blocking send** с `default`.
3. **Drop или disconnect** при заполнении.
4. **Мониторинг** длины канала.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Лимит на размер элемента.**
6. **Алерты** на заполненность.

**❌ НЕ ДЕЛАЙ:**

7. **Не делай буфер безграничным.**
8. **Не игнорируй заполненные каналы.**

### Ключевые выводы подглавы 21.6

- **Memory exhaustion** через каналы.
- **Ограниченный буфер.**
- **Non-blocking send.**
- **Drop или disconnect.**
- **Мониторинг** длины.

---

## 21.7 Side-channel attacks через shared state

**Side-channel attack** — атака через **побочные** каналы.

### Что это

**Side-channel** — информация, которая **не должна** быть доступна, но **утекает** через:

- **Время.**
- **Потребление CPU.**
- **Потребление памяти.**
- **Сетевой трафик.**
- **Кэш.**

### Пример: shared state

```go
var cache = map[string]User{}
var mu sync.RWMutex

func getUser(id string) (User, bool) {
    mu.RLock()
    defer mu.RUnlock()
    u, ok := cache[id]
    return u, ok
}
```

**Side-channel:** время ответа зависит от наличия в кэше.

**Атака:** злоумышленник измеряет время и делает выводы о **наличии** пользователя.

### Пример: timing по мьютексу

```go
var mu sync.Mutex

func process(data []byte) {
    mu.Lock()
    defer mu.Unlock()
    if len(data) < 100 {
        return  // ← быстро
    }
    // долгая обработка
}
```

**Side-channel:** время зависит от длины данных.

### Пример: cache timing

```go
var cache = map[string]bool{}

func check(key string) bool {
    return cache[key]  // ← O(1) если есть, O(n) если нет? Нет, map всегда O(1)
}
```

**Проблема:** map O(1) в среднем, но **худший случай** O(n).

**Атака:** злоумышленник может измерить разницу.

### Защита

**1. Constant-time operations.**

```go
import "crypto/subtle"

// Всегда одинаковое время
```

**2. Fixed-time operations.**

Не делай ранних выходов.

**3. Random delays.**

```go
time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
```

**Проблема:** может не помочь.

**4. Одинаковый путь для всех.**

Все запросы идут по одному пути.

**5. Separate cache per user.**

```go
type UserCache struct {
    mu    sync.RWMutex
    cache map[string]map[string]User  // user → cache
}
```

**6. Rate limiting.**

```go
limiter := rate.NewLimiter(rate.Limit(10), 5)
```

**Что делает:** ограничивает число запросов. Злоумышленник не может собрать статистику.

### Пример: защита от timing

```go
func checkKey(key string) bool {
    start := time.Now()

    exists := cache[key]

    // Минимальное время ответа
    elapsed := time.Since(start)
    if elapsed < 1*time.Millisecond {
        time.Sleep(1*time.Millisecond - elapsed)
    }

    return exists
}
```

**Что делает:** фиксированное минимальное время.

**Проблема:** не решает полностью.

### Аннотация сложности

| Side-channel | Time |
|:---|:---|
| Cache hit | ~100 нс |
| Cache miss | ~1 мс |
| Разница | ~1 мс |

### 💡 Практика: как защититься от side-channel

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Constant-time** для security-sensitive.
2. **Fixed-time** для сравнений.
3. **Rate limiting** против статистики.
4. **Одинаковый путь** для всех.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Минимальное время** ответа.
6. **Separate cache** per user.

**❌ НЕ ДЕЛАЙ:**

7. **Не делай ранних выходов** в security.
8. **Не игнорируй timing.**

### Ключевые выводы подглавы 21.7

- **Side-channel** — утечка через побочные каналы.
- **Timing** — самый частый.
- **Constant-time** — защита.
- **Rate limiting** — против статистики.
- **Одинаковый путь** для всех.

---

## 21.8 Безопасное логирование

**Логирование** — источник утечек.

### Проблема

**Уязвимый код:**

```go
func login(w http.ResponseWriter, r *http.Request) {
    var req LoginRequest
    json.NewDecoder(r.Body).Decode(&req)

    log.Printf("login attempt: user=%s password=%s", req.Username, req.Password)  // ← утечка
}
```

**Что утекло:** пароль в логах.

### Что не логировать

- **Пароли.**
- **Токены.**
- **API keys.**
- **Кредитные карты.**
- **PII** (персональные данные).
- **Сессии.**
- **JWT.**

### Что логировать

- **ID пользователя** (не email).
- **Действие.**
- **Результат** (success/failure).
- **IP** (с осторожностью).
- **Request ID.**

### Пример

```go
func login(w http.ResponseWriter, r *http.Request) {
    var req LoginRequest
    json.NewDecoder(r.Body).Decode(&req)

    slog.Info("login attempt",
        "user_id", req.Username,  // OK
        "ip", r.RemoteAddr,        // OK (с осторожностью)
        "request_id", r.Header.Get("X-Request-ID"),
        // НЕ логируем password
    )
}
```

### Structured logging

**`slog`** (Go 1.21+):

```go
import "log/slog"

slog.Info("login",
    "user_id", userID,
    "success", true,
)
```

**Что даёт:** структурированные логи, легко парсить.

### Redaction

**Redact** — замена чувствительных данных.

```go
func redact(s string) string {
    if len(s) < 4 {
        return "***"
    }
    return s[:2] + "***" + s[len(s)-2:]
}

slog.Info("login",
    "user_id", userID,
    "token", redact(token),
)
```

### Concurrent logging

**Проблема:** несколько горутин пишут в лог.

**`slog`** — потокобезопасен.

**`log`** — тоже потокобезопасен.

**Но:** порядок не гарантирован.

### Логи и race conditions

**Не логируй shared state без синхронизации:**

```go
// ❌ Плохо
log.Printf("counter: %d", counter)  // ← data race

// ✅ Хорошо
log.Printf("counter: %d", counter.Load())
```

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `slog.Info` | ~1-10 мкс |
| `log.Printf` | ~1-10 мкс |
| Redaction | ~100 нс |

### 💡 Практика: как логировать безопасно

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Не логируй** пароли, токены, PII.
2. **Логируй ID**, не email.
3. **`slog`** для structured.
4. **Redaction** для чувствительных.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Review логов** на утечки.
6. **Аудит** доступа к логам.

**❌ НЕ ДЕЛАЙ:**

7. **Не логируй body запросов** целиком.
8. **Не логируй headers** с токенами.

### Ключевые выводы подглавы 21.8

- **Логирование** — источник утечек.
- **Не логируй** пароли, токены, PII.
- **`slog`** для structured.
- **Redaction** для чувствительных.
- **Потокобезопасность** важна.

---

## 21.9 Crypto в конкурентном коде

Разберём **crypto** в конкурентном коде.

### Проблемы

**1. `crypto/rand` — потокобезопасен.**

```go
import "crypto/rand"

b := make([]byte, 32)
rand.Read(b)  // ← потокобезопасен
```

**Но:** медленнее, чем `math/rand`.

**2. `math/rand` — не потокобезопасен.**

```go
import "math/rand"

rand.Int()  // ← data race при конкурентном использовании
```

**Решение:** `rand.New(rand.NewSource(seed))` для каждой горутины.

**3. `crypto/aes` — потокобезопасен.**

**Но:** **ключ** должен быть защищён.

**4. `crypto/tls` — потокобезопасен.**

**Но:** **сессии** — общие.

### Пример: правильное использование

```go
import (
    "crypto/rand"
    "encoding/base64"
)

func generateToken() (string, error) {
    b := make([]byte, 32)
    if _, err := rand.Read(b); err != nil {
        return "", err
    }
    return base64.URLEncoding.EncodeToString(b), nil
}
```

**Что делает:** криптографически стойкий токен.

### Пример: неправильное использование

```go
import "math/rand"

func generateToken() string {
    b := make([]byte, 32)
    rand.Read(b)  // ← math/rand, не crypto
    return base64.URLEncoding.EncodeToString(b)
}
```

**Проблема:** `math/rand` **предсказуем**. Злоумышленник может вычислить токен.

### Race conditions в crypto

**1. Общий ключ.**

```go
var key []byte  // ← shared

func encrypt(data []byte) []byte {
    return aes.Encrypt(key, data)  // ← key без синхронизации
}
```

**Проблема:** если `key` меняется — data race.

**Решение:** `atomic.Pointer` или `sync.RWMutex`.

**2. Общая сессия.**

```go
var session *tls.Config

func connect() {
    tls.Dial("tcp", addr, session)  // ← shared
}
```

**Проблема:** `tls.Config` может меняться.

**Решение:** не менять после создания.

### Crypto и timing

**1. Сравнение MAC.**

```go
// ❌ Плохо
if !hmac.Equal(expected, actual) {
    return errors.New("invalid")
}

// ✅ Хорошо
if !hmac.Equal(expected, actual) {
    return errors.New("invalid")
}
```

**`hmac.Equal`** — constant-time.

**2. Сравнение подписей.**

```go
// ❌ Плохо
if bytes.Equal(sig1, sig2) {
    // ...
}

// ✅ Хорошо
if subtle.ConstantTimeCompare(sig1, sig2) == 1 {
    // ...
}
```

### Хранение ключей

**1. Не храни в коде.**

```go
// ❌ Плохо
const apiKey = "secret123"
```

**2. Environment variables.**

```go
// ✅ Хорошо
apiKey := os.Getenv("API_KEY")
```

**3. Secret management.**

- **Vault.**
- **AWS Secrets Manager.**
- **Kubernetes Secrets.**

**4. Не логируй ключи.**

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `crypto/rand.Read` | ~1-10 мкс |
| `math/rand.Int` | ~10-50 нс |
| `subtle.ConstantTimeCompare` | ~100 нс |
| `aes.Encrypt` | ~1-10 мкс |

### 💡 Практика: как использовать crypto

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`crypto/rand`** для токенов.
2. **`subtle.ConstantTimeCompare`** для сравнений.
3. **Secret management** для ключей.
4. **Не логируй** ключи.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Аудит** crypto-кода.
6. **Ротация** ключей.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `math/rand`** для security.
8. **Не храни ключи** в коде.
9. **Не сравнивай** через `==`.

### Ключевые выводы подглавы 21.9

- **`crypto/rand`** — потокобезопасен, стойкий.
- **`math/rand`** — не для security.
- **`subtle.ConstantTimeCompare`** — для сравнений.
- **Secret management** для ключей.
- **Не логируй** ключи.

---

## 21.10 Практика Go: защита от DoS и TOCTOU

Напишем **защищённый сервис**.

### Код

```go
package main

import (
    "context"
    "crypto/subtle"
    "database/sql"
    "errors"
    "log/slog"
    "net/http"
    "sync"
    "sync/atomic"
    "time"

    "golang.org/x/time/rate"
)

// 1. Защита от DoS через горутины
type ProtectedServer struct {
    httpSrv *http.Server
    taskSem chan struct{}
    limiter *rate.Limiter
    metrics *Metrics
}

type Metrics struct {
    Requests atomic.Int64
    Dropped  atomic.Int64
    Goroutines atomic.Int64
}

func NewProtectedServer() *ProtectedServer {
    s := &ProtectedServer{
        taskSem: make(chan struct{}, 100),   // не более 100 горутин
        limiter: rate.NewLimiter(rate.Limit(1000), 100),  // 1000 req/s
        metrics: &Metrics{},
    }

    mux := http.NewServeMux()
    mux.HandleFunc("/reset-password", s.handleResetPassword)
    mux.HandleFunc("/health", s.handleHealth)

    s.httpSrv = &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    return s
}

func (s *ProtectedServer) handleResetPassword(w http.ResponseWriter, r *http.Request) {
    s.metrics.Requests.Add(1)

    // Rate limiting
    if !s.limiter.Allow() {
        s.metrics.Dropped.Add(1)
        http.Error(w, "too many requests", http.StatusTooManyRequests)
        return
    }

    // Semaphore для ограничения горутин
    select {
    case s.taskSem <- struct{}{}:
        defer func() { <-s.taskSem }()
    case <-time.After(1 * time.Second):
        s.metrics.Dropped.Add(1)
        http.Error(w, "server busy", http.StatusServiceUnavailable)
        return
    }

    // TOCTOU-safe reset
    if err := s.resetPasswordAtomic(r.Context(), r); err != nil {
        slog.Error("reset failed", "error", err)
        http.Error(w, "failed", http.StatusInternalServerError)
        return
    }

    w.WriteHeader(http.StatusOK)
}

// Атомарный reset (защита от TOCTOU)
func (s *ProtectedServer) resetPasswordAtomic(ctx context.Context, r *http.Request) error {
    var req struct {
        UserID   int    `json:"user_id"`
        Token    string `json:"token"`
        Password string `json:"password"`
    }
    // ... decode req ...

    // Атомарный SQL
    result, err := db.ExecContext(ctx, `
        UPDATE users
        SET password = $1, reset_token = NULL
        WHERE id = $2
          AND reset_token = $3
          AND reset_token_expires > NOW()
    `, hashPassword(req.Password), req.UserID, req.Token)
    if err != nil {
        return err
    }

    rows, _ := result.RowsAffected()
    if rows == 0 {
        return errors.New("invalid or expired token")
    }
    return nil
}

// Constant-time сравнение (защита от timing)
func constantTimeCompare(a, b string) bool {
    return subtle.ConstantTimeCompare([]byte(a), []byte(b)) == 1
}

func (s *ProtectedServer) handleHealth(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    slog.Info("health",
        "goroutines", s.metrics.Goroutines.Load(),
        "requests", s.metrics.Requests.Load(),
        "dropped", s.metrics.Dropped.Load(),
    )
}

// Мониторинг горутин
func (s *ProtectedServer) monitorGoroutines(ctx context.Context) {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            n := runtime.NumGoroutine()
            s.metrics.Goroutines.Store(int64(n))
            if n > 10000 {
                slog.Warn("high goroutine count", "count", n)
            }
        }
    }
}
```

### Что демонстрирует

1. **Rate limiting** — защита от DoS.
2. **Semaphore** — ограничение горутин.
3. **Атомарный SQL** — защита от TOCTOU.
4. **Constant-time** — защита от timing.
5. **Мониторинг** горутин.
6. **Structured logging** без утечек.

### Аннотация сложности

| Защита | Time |
|:---|:---|
| Rate limiter | ~50-100 нс |
| Semaphore | ~30-70 нс |
| Атомарный SQL | ~1-10 мс |
| Constant-time | ~100 нс |

### 💡 Практика: как строить защищённые сервисы

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Rate limiting.**
2. **Semaphore** для горутин.
3. **Атомарные SQL.**
4. **Constant-time** для сравнений.
5. **Мониторинг** горутин.

**👍 СТОИТ СДЕЛАТЬ:**

6. **Structured logging** без утечек.
7. **Алерты** на аномалии.

**❌ НЕ ДЕЛАЙ:**

8. **Не делай check + use раздельно.**
9. **Не создавай горутину на запрос.**
10. **Не логируй чувствительные данные.**

### Ключевые выводы подглавы 21.10

- **Rate limiting + semaphore** — защита от DoS.
- **Атомарный SQL** — защита от TOCTOU.
- **Constant-time** — защита от timing.
- **Мониторинг** горутин.
- **Structured logging** без утечек.

---

## 21.11 Выводы и типичные ошибки

**Что мы узнали?**

Race conditions — уязвимость. TOCTOU — окно между check и use. Атаки: файл, токен, аккаунт. Timing attacks — по времени. DoS через горутины. Memory exhaustion через каналы. Side-channel через shared state. Логирование — источник утечек. Crypto в конкурентном коде.

**Типичные ошибки:**

- ❌ **Check + use раздельно.** TOCTOU.
- ❌ **`==` для паролей.** Timing attack.
- ❌ **Ранние выходы** в security. Timing.
- ❌ **Горутина на запрос.** DoS.
- ❌ **Безграничный буфер** канала. Memory exhaustion.
- ❌ **Логирование паролей.** Утечка.
- ❌ **`math/rand` для токенов.** Предсказуемо.
- ❌ **Хранение ключей** в коде. Утечка.
- ❌ **Shared state** без синхронизации. Data race.
- ❌ **Игнорирование race** в security.
- ❌ **Нет rate limiting.** DoS.
- ❌ **Нет мониторинга** горутин.

---

## 21.12 Для быстрого повторения

- **Race condition** — уязвимость.
- **TOCTOU** — окно между check и use.
- **Защита:** атомарные SQL, `O_EXCL`, `fstat`.
- **Timing attack** — по времени.
- **`subtle.ConstantTimeCompare`** — защита.
- **DoS через горутины** — миллионы горутин.
- **Защита:** worker pool, semaphore, `SetLimit`.
- **Memory exhaustion** — через каналы.
- **Защита:** ограниченный буфер, non-blocking.
- **Side-channel** — через побочные каналы.
- **Логирование** — источник утечек.
- **Не логируй** пароли, токены, PII.
- **`slog`** для structured.
- **`crypto/rand`** для токенов.
- **`math/rand`** — не для security.
- **`subtle.ConstantTimeCompare`** — для сравнений.
- **Secret management** для ключей.
- **Rate limiting** против DoS.

---

## 21.13 Вопросы для самопроверки

1. Что такое race condition?
2. Чем race condition отличается от data race?
3. Что такое TOCTOU?
4. Примеры TOCTOU?
5. Как защититься от TOCTOU?
6. Что такое timing attack?
7. Как защититься от timing attack?
8. Что такое DoS через горутины?
9. Как защититься от DoS?
10. Что такое memory exhaustion?
11. Как защититься от memory exhaustion?
12. Что такое side-channel attack?
13. Как защититься от side-channel?
14. Что не логировать?
15. Что такое structured logging?
16. Что такое `crypto/rand`?
17. Почему `math/rand` не для security?
18. Что такое `subtle.ConstantTimeCompare`?

---

## 21.14 Ответы

### Ответ 1

**Race condition** — результат зависит от порядка выполнения.

### Ответ 2

**Data race** — технически доступ к одной памяти без синхронизации. **Race condition** — логически результат зависит от порядка. Может быть **без** data race.

### Ответ 3

**TOCTOU** — окно между **проверкой** (check) и **использованием** (use). Состояние меняется в окне.

### Ответ 4

**Примеры TOCTOU:**
- Файл: между `Stat` и `ReadFile`.
- Токен: между `CheckToken` и `DeleteToken`.
- Аккаунт: между `UserExists` и `CreateUser`.

### Ответ 5

**Защита:**
- Атомарные SQL: `UPDATE ... WHERE`, `DELETE ... RETURNING`.
- Уникальные индексы.
- `O_EXCL` для файлов.
- `fstat` открытого файла.

### Ответ 6

**Timing attack** — атака по времени выполнения.

### Ответ 7

**Защита:**
- `subtle.ConstantTimeCompare`.
- Фиксированное время.
- Одинаковый путь для всех.

### Ответ 8

**DoS через горутины** — миллионы горутин, OOM.

### Ответ 9

**Защита:**
- Worker pool.
- Semaphore.
- `errgroup.SetLimit`.
- Лимит на соединения.

### Ответ 10

**Memory exhaustion** — атака через заполнение каналов.

### Ответ 11

**Защита:**
- Ограниченный буфер.
- Non-blocking send.
- Drop или disconnect.
- Мониторинг длины.

### Ответ 12

**Side-channel attack** — атака через побочные каналы (время, CPU, память).

### Ответ 13

**Защита:**
- Constant-time.
- Fixed-time.
- Rate limiting.
- Одинаковый путь.

### Ответ 14

**Не логировать:**
- Пароли.
- Токены.
- API keys.
- PII.
- Кредитные карты.

### Ответ 15

**Structured logging** — логи в формате key-value. `slog` в Go.

### Ответ 16

**`crypto/rand`** — криптографически стойкий генератор. Потокобезопасен.

### Ответ 17

**`math/rand` не для security**, потому что **предсказуем**. Злоумышленник может вычислить.

### Ответ 18

**`subtle.ConstantTimeCompare`** — сравнение за **фиксированное** время. Защита от timing.

---

## 21.15 Куда идти дальше?

Мы разобрали безопасность: TOCTOU, timing attacks, DoS, memory exhaustion, side-channel, logging, crypto. Теперь мы умеем писать безопасный конкурентный код.

Но остаётся **фундаментальный вопрос**: как строить **распределённые** системы?

- **Как строить распределённые системы?** CAP, consistency, ordering. → **Глава 22: Распределённые системы — введение.**
- **Как работает Raft?** → **Глава 23: Consensus — Raft и Paxos.**
- **Как делать distributed locks?** → **Глава 24: Distributed locks и leader election.**

---

## 21.16 Чек-лист

| Уязвимость | Защита |
|:---|:---|
| **Race condition** | Атомарные операции |
| **TOCTOU** | `UPDATE ... WHERE`, `O_EXCL` |
| **Timing attack** | `subtle.ConstantTimeCompare` |
| **DoS через горутины** | Worker pool, semaphore |
| **Memory exhaustion** | Ограниченный буфер |
| **Side-channel** | Constant-time, rate limiting |
| **Логирование** | Не логируй PII |
| **`crypto/rand`** | Для токенов |
| **`math/rand`** | Не для security |
| **Secret management** | Vault, K8s Secrets |
| **Rate limiting** | `x/time/rate` |
| **Мониторинг** | `NumGoroutine` |

🔒 **Ключевая идея:** Race conditions — **уязвимость**, не только баг. **TOCTOU** — окно между check и use; защита — **атомарные операции** (`UPDATE ... WHERE`, `O_EXCL`, `fstat`). **Timing attacks** — по времени; защита — `subtle.ConstantTimeCompare`. **DoS через горутины** — миллионы горутин; защита — worker pool, semaphore, `SetLimit`. **Memory exhaustion** — через каналы; защита — ограниченный буфер, non-blocking send. **Side-channel** — через побочные каналы; защита — constant-time, rate limiting. **Логирование** — источник утечек; не логируй пароли, токены, PII. **`crypto/rand`** для токенов, `math/rand` — не для security. **Secret management** для ключей. **Rate limiting** против DoS. **Мониторинг** горутин.