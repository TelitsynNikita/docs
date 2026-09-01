# 🔒 Глава 14: Блокировки и конкурентность — как PostgreSQL управляет параллельным доступом

**Что вы узнаете:**
- Какие бывают блокировки: на строки, таблицы, advisory locks.
- Что происходит при конкурентных UPDATE и DELETE.
- Как работают `FOR UPDATE`, `FOR SHARE`, `FOR NO KEY UPDATE`, `FOR KEY SHARE`.
- Как PostgreSQL выстраивает очередь ожидающих транзакций.
- Что такое deadlock и как PostgreSQL его разрешает.
- Как `SKIP LOCKED` и `NOWAIT` меняют поведение блокировок.
- Как использовать advisory locks для прикладных задач.
- Как мониторить блокировки через `pg_locks` и `pg_stat_activity`.
- Связь с MVCC (Глава 5), VACUUM (Глава 6) и очередями (Глава 12).

**После прочтения вы сможете:**
- Писать конкурентно-безопасные UPDATE без гонок.
- Выбирать правильный режим блокировки (`FOR UPDATE` vs `FOR SHARE`).
- Реализовать очередь с `SKIP LOCKED` без гонок.
- Использовать advisory locks для распределённых блокировок.
- Диагностировать и разрешать deadlock'и.
- Находить блокирующие транзакции через `pg_locks`.
- Понимать, как блокировки влияют на VACUUM и производительность.

---

## Содержание

- [14.0 Пролог: два запроса, одна строка, бесконечное ожидание](#140-пролог-два-запроса-одна-строка-бесконечное-ожидание)
- [14.1 Блокировки строк: FOR UPDATE и FOR SHARE](#141-блокировки-строк-for-update-и-for-share)
- [14.2 Режимы блокировок: FOR NO KEY UPDATE, FOR KEY SHARE](#142-режимы-блокировок-for-no-key-update-for-key-share)
- [14.3 Очередь ожидания и deadlock](#143-очередь-ожидания-и-deadlock)
- [14.4 SKIP LOCKED и NOWAIT: как не ждать](#144-skip-locked-и-nowait-как-не-ждать)
- [14.5 Табличные блокировки: ACCESS SHARE, ACCESS EXCLUSIVE](#145-табличные-блокировки-access-share-access-exclusive)
- [14.6 Advisory locks: блокировки для прикладных задач](#146-advisory-locks-блокировки-для-прикладных-задач)
- [14.7 Мониторинг блокировок: pg_locks и pg_stat_activity](#147-мониторинг-блокировок-pg_locks-и-pg_stat_activity)
- [14.8 Практика Go: конкурентные операции и обработка deadlock](#148-практика-go-конкурентные-операции-и-обработка-deadlock)
- [14.9 Выводы и типичные ошибки](#149-выводы-и-типичные-ошибки)
- [14.10 Для быстрого повторения](#1410-для-быстрого-повторения)
- [14.11 Вопросы для самопроверки](#1411-вопросы-для-самопроверки)
- [14.12 Ответы](#1412-ответы)
- [14.13 Куда идти дальше?](#1413-куда-идти-дальше)

---

## 14.0 Пролог: два запроса, одна строка, бесконечное ожидание

Ты написал API для покупки товара:

```go
func buyItem(db *sql.DB, itemID, userID int64) error {
    tx, _ := db.Begin()
    defer tx.Rollback()

    // Проверяем остаток:
    var quantity int
    tx.QueryRow("SELECT quantity FROM products WHERE id = $1 FOR UPDATE", itemID).Scan(&quantity)
    if quantity < 1 {
        return errors.New("нет в наличии")
    }

    // Уменьшаем остаток:
    tx.Exec("UPDATE products SET quantity = quantity - 1 WHERE id = $1", itemID)

    // Создаём заказ:
    tx.Exec("INSERT INTO orders (user_id, product_id) VALUES ($1, $2)", userID, itemID)

    return tx.Commit()
}
```

**Всё работало, пока не запустили в проде.** Вдруг API начал **виснуть**. Запросы копятся, пользователи жалуются. Ты смотришь в `pg_stat_activity` (Глава 1) и видишь:

```
 pid  | state        | wait_event    | query
------+--------------+---------------+----------------------------------
 1234 | active       | transactionid | UPDATE products SET quantity = ...
 5678 | active       | transactionid | UPDATE products SET quantity = ...
 9012 | active       | tuple         | SELECT ... FOR UPDATE
```

Все ждут **одну строку** — товар с `id = 42`.

❓ **Что произошло?**

💡 Ответ: **блокировка строки**. Первый запрос заблокировал строку через `FOR UPDATE`, остальные встали в **очередь ожидания**. Если первый запрос долго не завершается — остальные висят.

В этой главе разберём, **как работают блокировки**, как их избежать, и как проектировать конкурентно-безопасные запросы.

---

## 14.1 Блокировки строк: FOR UPDATE и FOR SHARE

### Что такое блокировка строки

> **Блокировка строки (row lock)** — механизм, который не даёт другим транзакциям изменять или блокировать ту же строку, пока текущая транзакция не завершится.

**Вспомни главу 5 (MVCC):** читатели не блокируют писателей. Но **писатели блокируют писателей** — два UPDATE одной строки не могут выполняться одновременно.

### FOR UPDATE: блокировка для изменения

```sql
BEGIN;
SELECT * FROM products WHERE id = 42 FOR UPDATE;
-- Строка заблокирована до COMMIT/ROLLBACK.
-- Другие транзакции, которые попытаются FOR UPDATE или UPDATE эту строку, будут ждать.
```

**Что происходит под капотом:**

```
Транзакция A:
  SELECT ... FOR UPDATE → блокирует строку id=42

Транзакция B:
  SELECT ... FOR UPDATE → пытается блокировать id=42
  → Строка уже заблокирована A
  → B встаёт в очередь ожидания (wait_event = 'transactionid')
  → B ждёт, пока A не COMMIT/ROLLBACK

Транзакция C:
  UPDATE products SET ... WHERE id = 42
  → UPDATE тоже требует блокировку
  → C встаёт в очередь за B
```

**Очередь — FIFO:** кто первый встал, тот первый получит блокировку.

### FOR SHARE: блокировка для чтения

```sql
BEGIN;
SELECT * FROM products WHERE id = 42 FOR SHARE;
-- Строка заблокирована для ЧТЕНИЯ.
-- Другие могут читать, но не могут UPDATE/DELETE.
```

**Отличие от FOR UPDATE:**

| Аспект | FOR UPDATE | FOR SHARE |
|:---|:---|:---|
| **Цель** | Строку будут ИЗМЕНЯТЬ | Строку только ЧИТАЮТ, но нужно защитить от изменений |
| **Другие SELECT FOR SHARE** | ❌ Ждут | ✅ Проходят (совместимы) |
| **Другие UPDATE/DELETE** | ❌ Ждут | ❌ Ждут |
| **Когда** | Перед UPDATE/DELETE | Перед проверкой целостности |

**Аналогия:** `FOR UPDATE` — «я буду менять, не мешайте». `FOR SHARE` — «я читаю, не меняйте пока».

### Блокировка строки vs MVCC

Важное различие с главой 5:

- **Обычный SELECT** (MVCC) — видит снимок, **не блокирует** строку.
- **SELECT FOR UPDATE** — **блокирует** строку, видит **актуальную** версию.

```sql
-- MVCC: не блокирует, видит снимок на момент начала транзакции
SELECT * FROM products WHERE id = 42;

-- Блокировка: блокирует строку, видит актуальную версию
SELECT * FROM products WHERE id = 42 FOR UPDATE;
```

### Когда использовать FOR UPDATE

**✅ Правильно:**

```sql
-- Прочитать → изменить → закоммитить:
BEGIN;
SELECT quantity FROM products WHERE id = 42 FOR UPDATE;
-- Строка заблокирована. Никто не изменит quantity, пока мы работаем.
UPDATE products SET quantity = quantity - 1 WHERE id = 42;
COMMIT;
```

**❌ Неправильно:**

```sql
-- FOR UPDATE без последующего UPDATE — просто блокировка без смысла:
BEGIN;
SELECT * FROM products WHERE id = 42 FOR UPDATE;
-- ... долго думаем ...
COMMIT;  -- зря держали блокировку
```

**Альтернатива — атомарный UPDATE (Глава 5):**

```sql
UPDATE products SET quantity = quantity - 1 WHERE id = 42 AND quantity >= 1;
-- Без SELECT FOR UPDATE. Блокировка на время одного UPDATE.
```

**Правило:** если можешь обойтись атомарным UPDATE — используй его. `FOR UPDATE` — когда нужен **несколько шагов** внутри одной транзакции.

---

## 14.2 Режимы блокировок: FOR NO KEY UPDATE, FOR KEY SHARE

В подглаве 14.1 мы разобрали `FOR UPDATE` и `FOR SHARE`. Но PostgreSQL имеет **четыре режима** блокировки строк. Два дополнительных — `FOR NO KEY UPDATE` и `FOR KEY SHARE` — решают **конфликт с внешними ключами**.

### Проблема: блокировка FK

Вспомни главу 3: `REFERENCES` не создаёт индекс, но создаёт **внешний ключ**. При обновлении/удалении родительской строки PostgreSQL проверяет дочерние строки.

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id)  -- FK
);
```

**Что происходит при обновлении `users.id`:**

```
UPDATE users SET id = 100 WHERE id = 42;
-- PostgreSQL должен проверить: есть ли orders с user_id = 42?
-- Если есть — нужно их обновить (ON UPDATE CASCADE) или запретить (RESTRICT).
-- Для этого блокируются ДОЧЕРНИЕ строки в orders.
```

**Проблема:** если ты делаешь `SELECT ... FOR UPDATE` на строку в `orders`, а кто-то обновляет `users.id` — возникают **лишние блокировки**.

### Четыре режима блокировки строк

| Режим | Что блокирует | Конфликтует с |
|:---|:---|:---|
| `FOR UPDATE` | Строку для полного изменения | Всеми другими блокировками |
| `FOR NO KEY UPDATE` | Строку, но не ключ | `FOR UPDATE`, `FOR NO KEY UPDATE` |
| `FOR SHARE` | Строку от изменения | `FOR UPDATE`, `FOR NO KEY UPDATE` |
| `FOR KEY SHARE` | Строку только от изменения ключа | `FOR UPDATE`, `FOR NO KEY UPDATE` |

### FOR NO KEY UPDATE: блокировка без ключа

**Идея:** блокируем строку, но **разрешаем** обновление ключевых колонок.

```sql
BEGIN;
SELECT * FROM orders WHERE id = 42 FOR NO KEY UPDATE;
-- Блокирует строку от обычных UPDATE.
-- НО: FK-проверки могут обновлять ключ (user_id) без конфликта.
```

**Когда использовать:**

```sql
-- Обновляем НЕ ключевые колонки (статус, сумму):
UPDATE orders SET status = 'shipped' WHERE id = 42;
-- Достаточно FOR NO KEY UPDATE.
-- Не мешаем FK-обновлениям родительской таблицы.
```

### FOR KEY SHARE: только защита ключа

**Идея:** блокируем строку **только от изменения ключа**, не блокируем обычные UPDATE.

```sql
BEGIN;
SELECT * FROM orders WHERE id = 42 FOR KEY SHARE;
-- Другие могут UPDATE неключевые колонки (status, total).
-- НО: никто не может изменить user_id (ключ FK).
```

**Когда использовать:**

```sql
-- Проверяем, что родительская строка не будет удалена,
-- но не мешаем обновлять дочерние:
SELECT * FROM users WHERE id = 42 FOR KEY SHARE;
-- Другие могут UPDATE users.name, но не users.id.
```

### Визуализация: совместимость режимов

```
            | FU | FNKU | FS | FKS
------------+----+------+----+-----
FOR UPDATE  | ❌ | ❌   | ❌ | ❌
FOR NO KEY  | ❌ | ❌   | ❌ | ❌
FOR SHARE   | ❌ | ❌   | ✅ | ✅
FOR KEY SHR | ❌ | ❌   | ✅ | ✅
```

- `FOR UPDATE` конфликтует со всеми.
- `FOR SHARE` совместим только с `FOR SHARE` и `FOR KEY SHARE`.
- `FOR KEY SHARE` — самый «мягкий», совместим с `FOR SHARE` и `FOR KEY SHARE`.

### Как PostgreSQL выбирает режим

**Обычный UPDATE** берёт `FOR NO KEY UPDATE`, если обновляются **неключевые** колонки:

```sql
-- Обновляем status (не ключ):
UPDATE orders SET status = 'shipped' WHERE id = 42;
-- Блокировка: FOR NO KEY UPDATE
```

**UPDATE ключевой колонки** берёт `FOR UPDATE`:

```sql
-- Обновляем user_id (ключ FK):
UPDATE orders SET user_id = 100 WHERE id = 42;
-- Блокировка: FOR UPDATE (полная)
```

**DELETE** всегда берёт `FOR UPDATE`:

```sql
DELETE FROM orders WHERE id = 42;
-- Блокировка: FOR UPDATE (строка будет удалена)
```

### Практический пример: почему FK-блокировки бесят

**Сценарий:**

```sql
-- Таблица users (родитель), orders (дочерняя с FK).

-- Транзакция A: обновляет неключевую колонку заказа
BEGIN;
SELECT * FROM orders WHERE id = 42 FOR NO KEY UPDATE;
UPDATE orders SET status = 'shipped' WHERE id = 42;
-- Блокировка: FOR NO KEY UPDATE на строке id=42.

-- Транзакция B: хочет обновить users.id (ключ)
UPDATE users SET id = 100 WHERE id = 1;
-- PostgreSQL проверяет orders с user_id = 1.
-- Находит строку id=42, видит блокировку FOR NO KEY UPDATE.
-- FOR NO KEY UPDATE НЕ конфликтует с проверкой FK!
-- B проходит без ожидания.
```

**Если бы A взял FOR UPDATE:**

```sql
BEGIN;
SELECT * FROM orders WHERE id = 42 FOR UPDATE;
-- Блокировка: FOR UPDATE.

-- Транзакция B:
UPDATE users SET id = 100 WHERE id = 1;
-- Проверка FK находит orders id=42 с FOR UPDATE.
-- FOR UPDATE конфликтует с проверкой FK!
-- B ЖДЁТ, пока A не завершится.
```

**Вывод:** `FOR NO KEY UPDATE` — «мягче», не блокирует FK-проверки. Используй его, когда не меняешь ключевые колонки.

### 💡 Практика

**✅ ОБЯЗАТЕЛЬНО:**

1. **Для UPDATE неключевых колонок** — `FOR NO KEY UPDATE`:

```sql
SELECT * FROM orders WHERE id = 42 FOR NO KEY UPDATE;
```

2. **Для защиты от удаления родителя** — `FOR KEY SHARE`:

```sql
SELECT * FROM users WHERE id = 42 FOR KEY SHARE;
```

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

3. **Для простых атомарных UPDATE** — не нужен явный SELECT FOR UPDATE. PostgreSQL сам возьмёт нужный режим.

**❌ НЕ ДЕЛАЙ:**

4. **Не бери `FOR UPDATE` всегда** — он самый строгий, блокирует FK-проверки.

### Итог подглавы 14.2

- **Четыре режима:** `FOR UPDATE` (полный), `FOR NO KEY UPDATE` (не ключ), `FOR SHARE` (чтение), `FOR KEY SHARE` (только ключ).
- **Обычный UPDATE неключевой колонки** — автоматически `FOR NO KEY UPDATE`.
- **UPDATE ключа / DELETE** — `FOR UPDATE`.
- **FK-проверки конфликтуют** только с `FOR UPDATE` и `FOR NO KEY UPDATE`.
- **Используй мягкие режимы**, чтобы не блокировать лишнего.

---

## 14.3 Очередь ожидания и deadlock

В подглавах 14.1-14.2 мы видели: если строка заблокирована, другая транзакция **ждёт**. Теперь разберём, **как устроена очередь**, и что происходит, когда ожидание заходит в тупик.

### Как устроена очередь ожидания

Когда транзакция пытается заблокировать уже заблокированную строку, она **встаёт в очередь**.

```
Транзакция A: блокирует строку id=42 (FOR UPDATE)

Транзакция B: пытается FOR UPDATE id=42
  → строка занята → B встаёт в очередь за A

Транзакция C: пытается FOR UPDATE id=42
  → строка занята → C встаёт в очередь за B

Очередь: [B, C] (FIFO — первый встал, первый получит)
```

**Когда A завершается (COMMIT/ROLLBACK):**

```
1. A снимает блокировку.
2. PostgreSQL будит B (первый в очереди).
3. B получает блокировку, продолжает работу.
4. C всё ещё ждёт.
5. Когда B завершится — проснётся C.
```

**Что видно в pg_stat_activity (Глава 1):**

```
Транзакция A: state=active, wait_event=NULL (работает)
Транзакция B: state=active, wait_event='transactionid' (ждёт A)
Транзакция C: state=active, wait_event='transactionid' (ждёт B)
```

### Deadlock: тупик

**Deadlock (взаимоблокировка)** — ситуация, когда две транзакции ждут друг друга, и никто не может продолжиться.

**Классический сценарий:**

```
Транзакция A:
  BEGIN;
  UPDATE products SET quantity = 1 WHERE id = 1;  -- блокирует id=1
  UPDATE products SET quantity = 2 WHERE id = 2;  -- хочет id=2

Транзакция B:
  BEGIN;
  UPDATE products SET quantity = 3 WHERE id = 2;  -- блокирует id=2
  UPDATE products SET quantity = 4 WHERE id = 1;  -- хочет id=1

A держит id=1 и ждёт id=2.
B держит id=2 и ждёт id=1.
Никто не может завершиться. Тупик.
```

### Как PostgreSQL разрешает deadlock

**PostgreSQL автоматически обнаруживает deadlock** через **детектор тупиков** (deadlock detector).

**Как работает:**

1. Detector периодически проверяет граф ожиданий.
2. Если находит цикл (A ждёт B, B ждёт A) — **прерывает одну из транзакций**.
3. Прерванная транзакция получает ошибку:

```
ERROR: deadlock detected
DETAIL: Process 1234 waits for ShareLock on transaction 5678; blocked by process 5678.
```

4. Прерванная транзакция откатывается (ROLLBACK).
5. Вторая транзакция продолжает работу.

**Кого прерывают?** Обычно **ту, которая сделала меньше работы** (меньше WAL-записей, дешевле откатывать). Но это не гарантировано — зависит от внутренней эвристики.

### Как избежать deadlock

**Правило 1: блокируй в одинаковом порядке**

```sql
-- ❌ Плохо: A блокирует id=1, потом id=2. B — наоборот.

-- ✅ Хорошо: обе транзакции блокируют id=1, потом id=2:
-- Транзакция A:
UPDATE products SET ... WHERE id = 1;
UPDATE products SET ... WHERE id = 2;

-- Транзакция B:
UPDATE products SET ... WHERE id = 1;  -- ждёт A на id=1
-- До id=2 не доходит, тупика нет.
```

**Правило 2: держи транзакции короткими**

```sql
-- ❌ Долгая транзакция с несколькими UPDATE:
BEGIN;
UPDATE users SET name = 'Alice' WHERE id = 42;
-- ... 5 секунд думаем ...
UPDATE orders SET status = 'shipped' WHERE id = 100;
COMMIT;

-- ✅ Разбей на короткие:
UPDATE users SET name = 'Alice' WHERE id = 42;
-- отдельная транзакция

UPDATE orders SET status = 'shipped' WHERE id = 100;
-- отдельная транзакция
```

**Правило 3: используй атомарные UPDATE вместо SELECT FOR UPDATE**

```sql
-- ❌ SELECT FOR UPDATE → думаем → UPDATE:
BEGIN;
SELECT * FROM products WHERE id = 42 FOR UPDATE;
-- ...
UPDATE products SET quantity = quantity - 1 WHERE id = 42;
COMMIT;

-- ✅ Один атомарный UPDATE:
UPDATE products SET quantity = quantity - 1 WHERE id = 42 AND quantity >= 1;
-- Блокировка держится миллисекунды.
```

### Как найти deadlock в логах

```bash
# В логах PostgreSQL:
grep "deadlock detected" /var/log/postgresql/postgresql.log
```

**Вывод:**

```
2024-01-15 10:30:00 UTC [1234] ERROR: deadlock detected
2024-01-15 10:30:00 UTC [1234] DETAIL: Process 1234 waits for ShareLock on transaction 5678
2024-01-15 10:30:00 UTC [1234] HINT: See server log for query details.
```

### Как обрабатывать deadlock в Go

```go
func updateTwoRows(db *sql.DB) error {
    maxRetries := 3
    for attempt := 0; attempt < maxRetries; attempt++ {
        err := doUpdate(db)
        if err == nil {
            return nil
        }
        if isDeadlock(err) {
            // Deadlock — повторяем транзакцию
            time.Sleep(time.Duration(attempt*100) * time.Millisecond)
            continue
        }
        return err
    }
    return errors.New("deadlock: слишком много попыток")
}

func isDeadlock(err error) bool {
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) {
        return pgErr.Code == "40P01"  // SQLSTATE для deadlock
    }
    return false
}
```

### Визуализация: deadlock vs обычное ожидание

**Обычное ожидание (без тупика):**

```
A: блокирует id=1 ──→ ждёт id=2
B:                    ждёт id=2 → (A завершится) → B получает id=2
B: блокирует id=2 → ждёт id=1 → (C завершится) → B получает id=1

Всё разрешается последовательно.
```

**Deadlock (тупик):**

```
A: блокирует id=1 ──→ ждёт id=2
B: блокирует id=2 ──→ ждёт id=1

Цикл: A → id=2 → B → id=1 → A
Никто не может завершиться.
```

### 💡 Практика

**✅ ОБЯЗАТЕЛЬНО:**

1. **Блокируй строки в одинаковом порядке** во всех транзакциях.
2. **Держи транзакции короткими** — меньше шансов на deadlock.
3. **Обрабатывай deadlock в коде** — SQLSTATE `40P01`, повторяй транзакцию.

**👍 СТОИТ:**

4. **Используй атомарные UPDATE** вместо SELECT FOR UPDATE + UPDATE.

**❌ НЕ ДЕЛАЙ:**

5. **Не держи открытую транзакцию с блокировками** дольше секунды.

### Итог подглавы 14.3

- **Очередь ожидания** — FIFO: первый встал — первый получил блокировку.
- **Deadlock** — цикл: A ждёт B, B ждёт A.
- **PostgreSQL сам обнаруживает deadlock** и прерывает одну из транзакций (SQLSTATE `40P01`).
- **Избегай:** одинаковый порядок блокировок, короткие транзакции, атомарные UPDATE.
- **Обрабатывай в коде:** повторяй транзакцию при deadlock.

---

## 14.4 SKIP LOCKED и NOWAIT: как не ждать

В подглаве 14.3 мы видели: если строка заблокирована, транзакция **ждёт**. Но иногда ждать **нельзя** — лучше быстро получить ошибку или пропустить заблокированную строку.

### NOWAIT: не ждать, а падать

**NOWAIT** — если строка заблокирована, PostgreSQL **сразу** возвращает ошибку вместо ожидания.

```sql
BEGIN;
SELECT * FROM products WHERE id = 42 FOR UPDATE NOWAIT;
```

**Если строка заблокирована другой транзакцией:**

```
ERROR: could not obtain lock on row in relation "products"
SQLSTATE: 55P03
```

**Когда использовать:**

- **Пользовательский интерфейс** — не заставлять пользователя ждать.
- **API** — быстро вернуть «товар занят», чем висеть 10 секунд.

**Go-пример:**

```go
func tryLockProduct(db *sql.DB, productID int64) error {
    tx, _ := db.Begin()
    defer tx.Rollback()

    var id int64
    err := tx.QueryRow(
        "SELECT id FROM products WHERE id = $1 FOR UPDATE NOWAIT",
        productID,
    ).Scan(&id)

    if err != nil {
        if isLockNotAvailable(err) {
            return errors.New("товар занят, попробуйте позже")
        }
        return err
    }
    return tx.Commit()
}

func isLockNotAvailable(err error) bool {
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) {
        return pgErr.Code == "55P03"  // lock_not_available
    }
    return false
}
```

### SKIP LOCKED: пропустить заблокированное

**SKIP LOCKED** — если строка заблокирована, PostgreSQL **пропускает** её и идёт дальше.

```sql
SELECT * FROM tasks
WHERE status = 'pending'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

**Как работает:**

```
Таблица tasks:
  id=1: заблокирована другим воркером
  id=2: свободна
  id=3: свободна

SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1:
  → Пропускает id=1 (заблокирована)
  → Берёт id=2
```

**Мы уже использовали SKIP LOCKED в главе 12 (очередь задач).** Это ключевой механизм для:

- **Очередей** — воркеры берут свободные задачи, не мешая друг другу.
- **Пакетной обработки** — разбирать строки параллельно без конфликтов.

### Сравнение: обычное ожидание vs NOWAIT vs SKIP LOCKED

| Режим | Если строка заблокирована | Когда использовать |
|:---|:---|:---|
| **Обычный** (FOR UPDATE) | Ждёт, пока не освободится | Критично получить именно эту строку |
| **NOWAIT** | Сразу ошибка (55P03) | Нельзя ждать (UI, API) |
| **SKIP LOCKED** | Пропускает | Очереди, пакетная обработка |

### Практический пример: пакетная обработка

**Задача:** обработать 100 000 задач тремя воркерами параллельно.

**Без SKIP LOCKED:**

```sql
-- Каждый воркер:
SELECT * FROM tasks
WHERE status = 'pending'
ORDER BY id
FOR UPDATE
LIMIT 100;
-- Если другой воркер уже заблокировал первую сотню — ждём.
```

**С SKIP LOCKED:**

```sql
-- Каждый воркер:
SELECT * FROM tasks
WHERE status = 'pending'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 100;
-- Каждый воркер берёт СВОЮ сотню незаблокированных задач.
```

**Результат:** воркеры работают параллельно, без ожидания.

### Комбинированный запрос с обновлением

```sql
WITH next_batch AS (
    SELECT id
    FROM tasks
    WHERE status = 'pending'
    ORDER BY id
    FOR UPDATE SKIP LOCKED
    LIMIT 100
)
UPDATE tasks SET status = 'processing'
WHERE id IN (SELECT id FROM next_batch)
RETURNING *;
-- Атомарно: выбрали 100 свободных задач и сразу пометили processing.
```

### 💡 Практика

**✅ ОБЯЗАТЕЛЬНО:**

1. **Для очередей — SKIP LOCKED** (глава 12):

```sql
SELECT * FROM tasks WHERE status = 'pending'
FOR UPDATE SKIP LOCKED LIMIT 1;
```

2. **Для UI/API — NOWAIT**, если не хочешь заставлять пользователя ждать.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

3. **Для критичных транзакций** — обычное ожидание (ждём, пока освободится).

**❌ НЕ ДЕЛАЙ:**

4. **Не используй SKIP LOCKED** там, где нужна конкретная строка — можешь пропустить именно её.

### Итог подглавы 14.4

- **NOWAIT** — не ждать, сразу ошибка `55P03`.
- **SKIP LOCKED** — пропустить заблокированное, взять свободное.
- **Очереди** — SKIP LOCKED для параллельных воркеров.
- **UI/API** — NOWAIT для быстрого отклика.
- **Обычное ожидание** — когда нужна именно эта строка.

---

## 14.5 Табличные блокировки: ACCESS SHARE, ACCESS EXCLUSIVE

До сих пор мы говорили о блокировках **строк**. Но PostgreSQL блокирует и **целые таблицы** — особенно при DDL-операциях (ALTER TABLE, CREATE INDEX, VACUUM FULL).

### Зачем нужны табличные блокировки

**Строковая блокировка** защищает конкретную строку. **Табличная** — защищает **структуру** таблицы.

```
ALTER TABLE users ADD COLUMN phone TEXT;
-- Нельзя менять структуру, пока кто-то читает таблицу.
-- Поэтому PostgreSQL берёт табличную блокировку.
```

### Основные режимы табличных блокировок

| Режим | Кто берёт | Что блокирует |
|:---|:---|:---|
| `ACCESS SHARE` | Обычный SELECT | Почти ничего, совместим со всеми |
| `ROW SHARE` | SELECT FOR UPDATE / FOR SHARE | Только `ACCESS EXCLUSIVE` |
| `ROW EXCLUSIVE` | INSERT, UPDATE, DELETE | Только `ACCESS EXCLUSIVE` |
| `SHARE` | CREATE INDEX (без CONCURRENTLY) | Изменения структуры |
| `ACCESS EXCLUSIVE` | ALTER TABLE, DROP TABLE, VACUUM FULL | **Всё** |

### ACCESS SHARE: обычный SELECT

```sql
SELECT * FROM users;
-- Берёт ACCESS SHARE.
-- Совместим со всеми другими режимами.
-- SELECT'ы не мешают друг другу и не мешают UPDATE.
```

### ROW EXCLUSIVE: обычные изменения

```sql
INSERT INTO users VALUES (...);
UPDATE users SET ...;
DELETE FROM users WHERE ...;
-- Каждая операция берёт ROW EXCLUSIVE на таблицу.
-- Плюс блокировку строки (FOR UPDATE / FOR NO KEY UPDATE).
```

**Совместимость:** ROW EXCLUSIVE совместим с ACCESS SHARE, ROW SHARE, другими ROW EXCLUSIVE. **Не совместим** с ACCESS EXCLUSIVE.

### ACCESS EXCLUSIVE: полная блокировка

```sql
ALTER TABLE users ADD COLUMN phone TEXT;
-- Берёт ACCESS EXCLUSIVE.
-- Никто не может читать или писать, пока ALTER не завершится.
```

**Что происходит:**

```
Транзакция A: SELECT * FROM users;  -- ACCESS SHARE

Транзакция B: ALTER TABLE users ...;  -- хочет ACCESS EXCLUSIVE
→ Блокировка ACCESS EXCLUSIVE несовместима с ACCESS SHARE.
→ B ждёт, пока A завершится.

Если A — длинный SELECT (отчёт на 10 минут),
то B будет ждать 10 минут.
```

### Проблема: длинные запросы блокируют DDL

**Сценарий из жизни:**

```
1. Аналитик запускает SELECT на 10 минут (строит отчёт).
2. Ты запускаешь миграцию: ALTER TABLE users ADD COLUMN phone TEXT.
3. ALTER ждёт, пока SELECT завершится.
4. Все INSERT/UPDATE/DELETE в users — тоже ждут (ACCESS EXCLUSIVE несовместим с ROW EXCLUSIVE).
5. Таблица заблокирована на 10 минут.
6. Пользователи жалуются.
```

**Решение:**

```sql
-- Перед ALTER проверь, есть ли длинные запросы:
SELECT pid, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - xact_start > interval '1 minute';
```

### SHARE: CREATE INDEX

**Обычный `CREATE INDEX`** берёт `SHARE`:

```sql
CREATE INDEX idx_users_email ON users(email);
-- SHARE: SELECT'ы продолжают работать, но INSERT/UPDATE/DELETE — нет.
```

**`CREATE INDEX CONCURRENTLY`** — без блокировки:

```sql
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
-- Не берёт SHARE. Таблица доступна для записи.
-- Медленнее (2-3 раза), но без блокировки.
-- Вспомни главу 3 и главу 11.14.
```

### Как проверить табличные блокировки

```sql
SELECT 
    l.locktype,
    l.relation::regclass AS table_name,
    l.mode,
    l.granted,
    a.pid,
    a.query
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE l.locktype = 'relation'
  AND l.relation::regclass = 'users'::regclass;
```

**Вывод:**

```
 locktype | table_name | mode              | granted | pid  | query
----------+------------+-------------------+---------+------+---------------------------
 relation | users      | AccessShareLock   | t       | 1234 | SELECT * FROM users;
 relation | users      | AccessExclusiveLock| f      | 5678 | ALTER TABLE users ...;
```

- `granted = t` — блокировка получена.
- `granted = f` — ждёт.

### Таблица совместимости (упрощённо)

| | ACCESS SHARE | ROW EXCLUSIVE | SHARE | ACCESS EXCLUSIVE |
|:---|:---:|:---:|:---:|:---:|
| **ACCESS SHARE** | ✅ | ✅ | ✅ | ❌ |
| **ROW EXCLUSIVE** | ✅ | ✅ | ❌ | ❌ |
| **SHARE** | ✅ | ❌ | ✅ | ❌ |
| **ACCESS EXCLUSIVE** | ❌ | ❌ | ❌ | ❌ |

### 💡 Практика

**✅ ОБЯЗАТЕЛЬНО:**

1. **Используй `CREATE INDEX CONCURRENTLY`** — без блокировки записи.
2. **Для ALTER TABLE ADD COLUMN** — в PostgreSQL 11+ это мгновенно и без полной блокировки записи (но всё ещё ACCESS EXCLUSIVE).
3. **Проверяй длинные запросы перед DDL:**

```sql
SELECT pid, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - xact_start > interval '1 minute';
```

**👍 СТОИТ:**

4. **Планируй DDL на часы низкой нагрузки.**

**❌ НЕ ДЕЛАЙ:**

5. **Не запускай `ALTER TABLE ... ALTER COLUMN TYPE` на проде** без проверки — это ACCESS EXCLUSIVE + переписывание таблицы.

### Итог подглавы 14.5

- **Табличные блокировки** защищают структуру таблицы.
- **SELECT** — ACCESS SHARE, совместим со всеми.
- **INSERT/UPDATE/DELETE** — ROW EXCLUSIVE, совместимы между собой.
- **ALTER TABLE / DROP / VACUUM FULL** — ACCESS EXCLUSIVE, блокирует всё.
- **CREATE INDEX CONCURRENTLY** — без блокировки записи.
- **Длинные SELECT'ы** могут блокировать DDL.

---

## 14.6 Advisory locks: блокировки для прикладных задач

Все предыдущие блокировки PostgreSQL ставит **сам** — на строки и таблицы. **Advisory locks** — это блокировки, которые **ты** ставишь явно, для своих прикладных задач.

### Что такое advisory lock

> **Advisory lock (рекомендательная блокировка)** — блокировка на **произвольный ключ** (число или строку), которую приложение ставит и снимает **само**. PostgreSQL не связывает её со строками или таблицами.

**Простыми словами:** это «блокировка по договорённости». PostgreSQL просто хранит, кто какой ключ заблокировал. Смысл ключу придаёт приложение.

### Зачем нужны advisory locks

**Задачи, которые не решаются блокировками строк:**

1. **Распределённая блокировка** — «только один инстанс приложения может выполнять задачу X».
2. **Синхронизация между сервисами** — сервис A и сервис B не должны работать одновременно.
3. **Блокировка на уровне бизнес-логики** — «пользователь 42 не может проводить две операции одновременно».

### Функции advisory locks

```sql
-- Блокировка по целому числу:
SELECT pg_advisory_lock(42);

-- Блокировка по двум числам:
SELECT pg_advisory_lock(42, 100);

-- Блокировка по строке (хешируется внутри):
SELECT pg_advisory_lock('my_lock_key');

-- Снятие блокировки:
SELECT pg_advisory_unlock(42);

-- Попытка без ожидания:
SELECT pg_try_advisory_lock(42);
-- Возвращает true, если блокировка получена, false — если занята.
```

### Как это работает

**Advisory lock — это просто запись в shared memory:**

```
pg_locks:
  locktype = 'advisory'
  objid = 42  ← твой ключ
  pid = 1234  ← кто держит
```

PostgreSQL **не проверяет**, существует ли строка с id=42 или таблица. Это просто «ключ занят процессом 1234».

### Ключевые свойства

**1. Блокировка не связана с транзакцией (session-level):**

```sql
-- Session-level: живёт до явного unlock или до конца сессии
SELECT pg_advisory_lock(42);
-- ... делаем что-то долго ...
SELECT pg_advisory_unlock(42);
-- COMMIT/ROLLBACK НЕ снимает блокировку!
```

**2. Transaction-level (xact):**

```sql
-- Transaction-level: живёт до конца транзакции
BEGIN;
SELECT pg_advisory_xact_lock(42);
-- ... работаем ...
COMMIT;  -- блокировка автоматически снимается
```

**3. try-версия — без ожидания:**

```sql
SELECT pg_try_advisory_lock(42);
-- Если заблокировано — сразу false, без ожидания.
```

### Практический пример: распределённая блокировка между инстансами

**Задача:** три инстанса приложения периодически запускают «сборку отчётов». Нужно, чтобы сборка выполнялась **только одним** инстансом одновременно.

```sql
-- Инстанс пытается взять блокировку:
SELECT pg_try_advisory_lock(100500);
-- Если true — можно выполнять сборку.
-- Если false — другой инстанс уже собирает, пропускаем.
```

**Go-реализация:**

```go
func tryRunReport(db *sql.DB) (bool, error) {
    var gotLock bool
    err := db.QueryRow("SELECT pg_try_advisory_lock(100500)").Scan(&gotLock)
    if err != nil {
        return false, err
    }
    if !gotLock {
        return false, nil  // другой инстанс уже работает
    }

    // Блокировка получена — выполняем отчёт
    defer db.Exec("SELECT pg_advisory_unlock(100500)")
    runReport()
    return true, nil
}
```

### Пример: блокировка на уровне пользователя

**Задача:** пользователь не может проводить два платёжных запроса одновременно.

```sql
-- Ключ = user_id:
SELECT pg_advisory_lock(42);
-- Пока пользователь 42 «заблокирован», другой его запрос будет ждать:
SELECT pg_advisory_lock(42);  -- ждёт...
```

**Go-реализация:**

```go
func processPayment(db *sql.DB, userID int64) error {
    // Блокируем пользователя
    _, err := db.Exec("SELECT pg_advisory_lock($1)", userID)
    if err != nil {
        return err
    }
    defer db.Exec("SELECT pg_advisory_unlock($1)", userID)

    // Проверяем баланс, списываем, коммитим
    // ...
}
```

### Advisory locks vs блокировки строк

| Аспект | Блокировка строки | Advisory lock |
|:---|:---|:---|
| **Что блокирует** | Конкретную строку | Произвольный ключ |
| **Кто ставит** | PostgreSQL автоматически | Приложение явно |
| **Связана с транзакцией** | Да | Session-level или xact-level |
| **Смысл** | Защита данных | Защита бизнес-логики |
| **Конфликты** | С другими строками | Только с тем же ключом |

### Проблемы advisory locks

**1. Легко забыть снять:**

```sql
SELECT pg_advisory_lock(42);
-- ... код ...
-- Забыли pg_advisory_unlock(42)!
-- Блокировка висит до конца сессии.
```

**Решение:** используй `pg_advisory_xact_lock` — снимается автоматически при COMMIT.

**2. Ключи не типизированы:**

```sql
-- pg_advisory_lock(42) и pg_advisory_lock('42') — РАЗНЫЕ ключи!
-- Число 42 и строка '42' — разные.
```

**Решение:** всегда используй числа или всегда строки.

### 💡 Практика

**✅ ОБЯЗАТЕЛЬНО:**

1. **Для распределённых блокировок** — `pg_try_advisory_lock` + проверка.

2. **Для коротких операций** — `pg_advisory_xact_lock` (снимается при COMMIT):

```sql
BEGIN;
SELECT pg_advisory_xact_lock(42);
-- ... работаем ...
COMMIT;  -- блокировка снята
```

3. **Всегда снимай session-level блокировку** через `pg_advisory_unlock`.

**❌ НЕ ДЕЛАЙ:**

4. **Не используй advisory lock для блокировки строк** — для этого есть FOR UPDATE.

5. **Не смешивай числовые и строковые ключи** — это разные блокировки.

### Итог подглавы 14.6

- **Advisory lock** — блокировка на произвольный ключ, ставится приложением.
- **Session-level** (`pg_advisory_lock`) — живёт до unlock или конца сессии.
- **Transaction-level** (`pg_advisory_xact_lock`) — живёт до COMMIT/ROLLBACK.
- **Try-версия** (`pg_try_advisory_lock`) — без ожидания.
- **Использование:** распределённые блокировки, синхронизация между сервисами, блокировка бизнес-логики.

---

## 14.7 Мониторинг блокировок: pg_locks и pg_stat_activity

Когда запросы висят, нужно **найти, кто кого блокирует**. PostgreSQL предоставляет два ключевых инструмента: `pg_locks` и `pg_stat_activity`.

### pg_locks: все блокировки

```sql
SELECT 
    l.locktype,
    l.relation::regclass AS table_name,
    l.mode,
    l.granted,
    l.pid,
    a.usename,
    a.query
FROM pg_locks l
LEFT JOIN pg_stat_activity a ON a.pid = l.pid
ORDER BY l.granted, l.pid;
```

**Вывод:**

```
 locktype | table_name | mode              | granted | pid  | query
----------+------------+-------------------+---------+------+---------------------------
 relation | products   | RowExclusiveLock  | t       | 1234 | UPDATE products SET ...
 relation | products   | RowExclusiveLock  | f       | 5678 | UPDATE products SET ...
 tuple    | products   | ExclusiveLock     | t       | 1234 | UPDATE products SET ...
 transactionid | NULL  | ExclusiveLock     | f       | 5678 | UPDATE products SET ...
```

**Ключевые поля:**

| Поле | Что означает |
|:---|:---|
| `locktype` | Тип блокировки: `relation`, `tuple`, `transactionid`, `advisory` |
| `granted` | `t` — получена, `f` — ожидает |
| `mode` | Режим: `RowExclusiveLock`, `AccessExclusiveLock`, `ExclusiveLock` |
| `pid` | Процесс, который держит/ждёт |

### Как найти блокирующего и ждущего

**Запрос, который показывает цепочку блокировок:**

```sql
SELECT 
    waiting.pid AS waiting_pid,
    waiting.query AS waiting_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    now() - waiting.query_start AS waiting_duration
FROM pg_stat_activity waiting
JOIN pg_locks wl ON wl.pid = waiting.pid AND NOT wl.granted
JOIN pg_locks bl ON bl.locktype = wl.locktype 
    AND bl.database = wl.database 
    AND bl.relation = wl.relation 
    AND bl.pid <> waiting.pid 
    AND bl.granted
JOIN pg_stat_activity blocking ON blocking.pid = bl.pid
WHERE waiting.state = 'active';
```

**Вывод:**

```
 waiting_pid | waiting_query                    | blocking_pid | blocking_query
-------------+----------------------------------+--------------+-------------------------------
 5678        | UPDATE products SET quantity = 1 | 1234         | SELECT ... FOR UPDATE;
```

**Теперь видно: процесс 5678 ждёт, процесс 1234 блокирует.**

### Типы блокировок в pg_locks

| locktype | Что означает | Пример |
|:---|:---|:---|
| `relation` | Блокировка таблицы | ALTER TABLE, CREATE INDEX |
| `tuple` | Блокировка конкретной версии строки | UPDATE с ожиданием |
| `transactionid` | Блокировка по ID транзакции | Ожидание COMMIT другой транзакции |
| `advisory` | Advisory lock | `pg_advisory_lock(42)` |

### Мониторинг через pg_stat_activity

Вспомни главу 1: `pg_stat_activity` показывает **состояние и ожидания**.

```sql
SELECT 
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS query_duration,
    query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_duration DESC;
```

**wait_event для блокировок:**

| wait_event | Что означает |
|:---|:---|
| `transactionid` | Ждёт завершения транзакции (блокировка строки) |
| `tuple` | Ждёт конкретную версию строки |
| `relation` | Ждёт табличную блокировку |
| `client` | Ждёт запрос от клиента |
| `DataFileRead` | Ждёт чтения с диска |

### Практический сценарий: найти виновника

**Ситуация:** API висит, пользователи жалуются.

**Шаг 1:** Найти ждущие запросы:

```sql
SELECT pid, wait_event, query, now() - query_start AS duration
FROM pg_stat_activity
WHERE state = 'active' AND wait_event IS NOT NULL;
```

**Шаг 2:** Найти блокирующего:

```sql
SELECT 
    waiting.pid AS waiting_pid,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    now() - blocking.query_start AS blocking_duration
FROM pg_stat_activity waiting
JOIN pg_locks wl ON wl.pid = waiting.pid AND NOT wl.granted
JOIN pg_locks bl ON bl.pid <> waiting.pid AND bl.granted
    AND bl.locktype = wl.locktype
    AND bl.relation = wl.relation
JOIN pg_stat_activity blocking ON blocking.pid = bl.pid
WHERE waiting.wait_event IS NOT NULL;
```

**Шаг 3:** Прервать блокирующего:

```sql
SELECT pg_terminate_backend(1234);
```

### Готовый запрос: все блокировки в понятном виде

```sql
SELECT 
    COALESCE(blocking.pid, waiting.pid) AS pid,
    COALESCE(blocking.query, waiting.query) AS query,
    CASE 
        WHEN blocking.pid IS NULL THEN 'waiting'
        ELSE 'blocking'
    END AS role,
    waiting.pid AS waiting_pid,
    blocking.pid AS blocking_pid
FROM pg_stat_activity waiting
FULL JOIN pg_locks wl ON wl.pid = waiting.pid AND NOT wl.granted
FULL JOIN pg_locks bl ON bl.locktype = wl.locktype 
    AND bl.relation = wl.relation AND bl.granted AND bl.pid <> waiting.pid
FULL JOIN pg_stat_activity blocking ON blocking.pid = bl.pid
WHERE waiting.pid IS NOT NULL OR blocking.pid IS NOT NULL;
```

### 💡 Практика

**✅ ОБЯЗАТЕЛЬНО:**

1. **Настрой алерт** в Grafana (Приложение E) на `wait_event` — если много `transactionid` ожиданий, что-то блокируется.

2. **Проверяй `pg_locks` при проблемах** — находи блокирующего и ждущего.

3. **Держи запрос на готове** — цепочку блокировок сохрани в закладки.

**👍 СТОИТ:**

4. **Раз в день проверяй:**

```sql
SELECT COUNT(*), wait_event
FROM pg_stat_activity
WHERE state = 'active'
GROUP BY wait_event;
```

**❌ НЕ ДЕЛАЙ:**

5. **Не прерывай транзакции бездумно** — `pg_terminate_backend` откатывает изменения, пользователь может потерять данные.

### Итог подглавы 14.7

- **pg_locks** — все блокировки: `granted` (t/f), `mode`, `pid`.
- **pg_stat_activity** — `wait_event` показывает, чего ждёт процесс.
- **Цепочка блокировок** — JOIN `pg_locks` с `pg_stat_activity`.
- **Прерывание** — `pg_terminate_backend(pid)`.
- **Настрой мониторинг** — алерты на `transactionid` ожидания.

---

## 14.8 Практика Go: конкурентные операции и обработка deadlock

### Полный пример: покупка товара с обработкой всех ошибок

```go
package main

import (
    "database/sql"
    "errors"
    "fmt"
    "log"
    "time"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgconn"
    _ "github.com/jackc/pgx/v5/stdlib"
)

// buyItem — конкурентно-безопасная покупка товара
func buyItem(db *sql.DB, productID, userID int64, quantity int) error {
    maxRetries := 3

    for attempt := 0; attempt < maxRetries; attempt++ {
        err := tryBuyItem(db, productID, userID, quantity)
        if err == nil {
            return nil
        }

        var pgErr *pgconn.PgError
        if errors.As(err, &pgErr) {
            switch pgErr.Code {
            case "40P01": // deadlock_detected
                time.Sleep(time.Duration(attempt*50) * time.Millisecond)
                continue
            case "55P03": // lock_not_available
                return errors.New("товар временно недоступен, попробуйте позже")
            case "23514": // check_violation
                return errors.New("недостаточно товара на складе")
            }
        }
        return err
    }
    return errors.New("слишком много попыток из-за конкуренции")
}

func tryBuyItem(db *sql.DB, productID, userID int64, quantity int) error {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // Атомарный UPDATE: без SELECT FOR UPDATE
    result, err := tx.Exec(`
        UPDATE products 
        SET quantity = quantity - $1 
        WHERE id = $2 AND quantity >= $1
    `, quantity, productID)

    if err != nil {
        return err
    }

    rowsAffected, _ := result.RowsAffected()
    if rowsAffected == 0 {
        return errors.New("товар закончился")
    }

    // Создаём заказ
    _, err = tx.Exec(`
        INSERT INTO orders (user_id, product_id, quantity, status)
        VALUES ($1, $2, $3, 'pending')
    `, userID, productID, quantity)

    if err != nil {
        return err
    }

    return tx.Commit()
}

// processQueueTask — воркер очереди с SKIP LOCKED
func processQueueTask(db *sql.DB) error {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()

    var taskID int64
    var payload string

    err = tx.QueryRow(`
        WITH next_task AS (
            SELECT id, payload
            FROM tasks
            WHERE status = 'pending'
            ORDER BY id
            FOR UPDATE SKIP LOCKED
            LIMIT 1
        )
        UPDATE tasks SET status = 'processing', started_at = now()
        WHERE id = (SELECT id FROM next_task)
        RETURNING id, payload
    `).Scan(&taskID, &payload)

    if err == sql.ErrNoRows {
        return nil
    }
    if err != nil {
        return err
    }

    tx.Commit()

    // Обрабатываем задачу вне транзакции
    err = processTask(taskID, payload)

    // Помечаем результат
    if err != nil {
        db.Exec("UPDATE tasks SET status = 'failed', error = $1 WHERE id = $2", err.Error(), taskID)
    } else {
        db.Exec("UPDATE tasks SET status = 'done', completed_at = now() WHERE id = $2", taskID)
    }
    return err
}

func processTask(id int64, payload string) error {
    // ... реальная обработка
    return nil
}

func isDeadlock(err error) bool {
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) {
        return pgErr.Code == "40P01"
    }
    return false
}

func main() {
    db, err := sql.Open("pgx", "postgres://alice@localhost/mydb")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Покупка товара
    err = buyItem(db, 42, 100, 1)
    if err != nil {
        fmt.Println("Ошибка покупки:", err)
    }

    // Обработка очереди
    err = processQueueTask(db)
    if err != nil {
        fmt.Println("Ошибка очереди:", err)
    }
}
```

### Ключевые моменты Go-примера

1. **Атомарный UPDATE** — вместо `SELECT FOR UPDATE` + `UPDATE`:

```go
result, err := tx.Exec(`
    UPDATE products 
    SET quantity = quantity - $1 
    WHERE id = $2 AND quantity >= $1
`, quantity, productID)
```

2. **Проверка RowsAffected** — если 0, товар закончился:

```go
rowsAffected, _ := result.RowsAffected()
if rowsAffected == 0 {
    return errors.New("товар закончился")
}
```

3. **Обработка deadlock** — SQLSTATE `40P01`, повтор:

```go
if isDeadlock(err) {
    time.Sleep(...)
    continue
}
```

4. **Очередь с SKIP LOCKED** — атомарный выбор задачи:

```go
tx.QueryRow(`
    WITH next_task AS (
        SELECT id FROM tasks
        WHERE status = 'pending'
        ORDER BY id
        FOR UPDATE SKIP LOCKED
        LIMIT 1
    )
    UPDATE tasks SET status = 'processing'
    WHERE id = (SELECT id FROM next_task)
    RETURNING id
`)
```

5. **Короткие транзакции** — задача обрабатывается вне транзакции:

```go
tx.Commit()          // транзакция завершена
processTask(taskID)  // обработка вне блокировки
```

### Итог подглавы 14.8

- **Атомарный UPDATE** — основной инструмент конкурентности.
- **RowsAffected** — проверка результата.
- **Обработка deadlock** — SQLSTATE `40P01`, повтор.
- **SKIP LOCKED** — для очередей.
- **Короткие транзакции** — блокировки держатся миллисекунды.

---

## 14.9 Выводы и типичные ошибки

**Что мы узнали?**

PostgreSQL использует **двухуровневую систему блокировок**: строковые (FOR UPDATE, FOR SHARE, FOR NO KEY UPDATE, FOR KEY SHARE) и табличные (ACCESS SHARE, ROW EXCLUSIVE, ACCESS EXCLUSIVE). Строковые блокировки защищают данные, табличные — структуру. При конфликте транзакции встают в FIFO-очередь. Deadlock'и автоматически обнаруживаются и разрешаются прерыванием одной из транзакций. `SKIP LOCKED` и `NOWAIT` дают контроль над ожиданием. Advisory locks позволяют блокировать произвольные ключи для прикладных задач. Мониторинг через `pg_locks` и `pg_stat_activity` показывает, кто кого блокирует.

**Типичные ошибки:**

- ❌ **SELECT FOR UPDATE без последующего UPDATE** — блокировка без смысла, замедляет других.
- ❌ **Долгие транзакции с блокировками** — все ждут, очереди растут.
- ❌ **Блокировка строк в разном порядке** — deadlock'и.
- ❌ **`FOR UPDATE` для неключевых UPDATE** — блокирует FK-проверки, используй `FOR NO KEY UPDATE`.
- ❌ **Игнорировать deadlock в коде** — транзакция упала, пользователь получил ошибку без повтора.
- ❌ **Запускать ALTER TABLE без проверки длинных запросов** — блокировка на часы.
- ❌ **Advisory lock без unlock** — блокировка висит до конца сессии.

---

## 14.10 Для быстрого повторения

- **Строковые блокировки:** `FOR UPDATE` (полный), `FOR NO KEY UPDATE` (без ключа), `FOR SHARE` (чтение), `FOR KEY SHARE` (только ключ).
- **Очередь ожидания:** FIFO — первый встал, первый получил блокировку.
- **Deadlock:** цикл ожиданий, PostgreSQL прерывает одну транзакцию (SQLSTATE `40P01`).
- **SKIP LOCKED:** пропустить заблокированные строки — для очередей.
- **NOWAIT:** сразу ошибка `55P03` вместо ожидания.
- **Табличные блокировки:** ACCESS SHARE (SELECT), ROW EXCLUSIVE (UPDATE), ACCESS EXCLUSIVE (ALTER).
- **Advisory locks:** `pg_advisory_lock`, `pg_advisory_xact_lock`, `pg_try_advisory_lock`.
- **Мониторинг:** `pg_locks` (granted, mode), `pg_stat_activity` (wait_event).
- **Атомарный UPDATE** — лучшая альтернатива SELECT FOR UPDATE для простых операций.

---

## 14.11 Вопросы для самопроверки

1. Чем FOR UPDATE отличается от обычного SELECT (MVCC)?
2. Зачем нужны FOR NO KEY UPDATE и FOR KEY SHARE?
3. Что такое deadlock? Как PostgreSQL его разрешает?
4. Как избежать deadlock? Назови три правила.
5. В чём разница между NOWAIT и SKIP LOCKED?
6. Какая табличная блокировка у ALTER TABLE? Почему она опасна?
7. Что такое advisory lock? Когда его использовать?
8. Как найти блокирующего и ждущего через pg_locks?
9. Почему атомарный UPDATE лучше SELECT FOR UPDATE для простых операций?
10. Как обработать deadlock в Go-коде?

---

## 14.12 Ответы

### Ответ 1

`FOR UPDATE` **блокирует** строку до конца транзакции и видит **актуальную** версию. Обычный SELECT (MVCC) не блокирует и видит **снимок** на момент начала транзакции.

### Ответ 2

`FOR NO KEY UPDATE` блокирует строку от обычных UPDATE, но **не конфликтует с FK-проверками**. `FOR KEY SHARE` защищает только ключевые колонки — другие могут обновлять неключевые.

### Ответ 3

Deadlock — цикл: A ждёт B, B ждёт A. PostgreSQL автоматически обнаруживает цикл через deadlock detector и прерывает одну из транзакций (SQLSTATE `40P01`).

### Ответ 4

1. Блокируй строки в одинаковом порядке.
2. Держи транзакции короткими.
3. Используй атомарные UPDATE вместо SELECT FOR UPDATE.

### Ответ 5

`NOWAIT` — сразу ошибка `55P03`, если строка заблокирована. `SKIP LOCKED` — пропускает заблокированную строку, берёт следующую свободную.

### Ответ 6

`ACCESS EXCLUSIVE` — блокирует все чтения и записи. Опасна тем, что длинный SELECT может заблокировать ALTER на часы.

### Ответ 7

Advisory lock — блокировка на произвольный ключ, которую ставит приложение. Для распределённых блокировок, синхронизации между сервисами.

### Ответ 8

```sql
SELECT 
    waiting.pid AS waiting_pid,
    blocking.pid AS blocking_pid,
    blocking.query
FROM pg_stat_activity waiting
JOIN pg_locks wl ON wl.pid = waiting.pid AND NOT wl.granted
JOIN pg_locks bl ON bl.granted AND bl.relation = wl.relation
JOIN pg_stat_activity blocking ON blocking.pid = bl.pid;
```

### Ответ 9

Атомарный UPDATE блокирует строку **на миллисекунды** — на время одного UPDATE. SELECT FOR UPDATE держит блокировку **до конца транзакции**, которая может быть долгой.

### Ответ 10

```go
func isDeadlock(err error) bool {
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) {
        return pgErr.Code == "40P01"
    }
    return false
}
// При deadlock — повторять транзакцию с задержкой.
```

---

## 14.13 Куда идти дальше?

Мы разобрали блокировки и конкурентность. Теперь — **как настроить PostgreSQL** под нагрузку: память, диск, WAL, work_mem, и другие параметры конфигурации.

**Глава 15: Конфигурация — как выжать максимум из PostgreSQL на твоём железе.**