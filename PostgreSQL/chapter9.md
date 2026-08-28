# ⚙️ Глава 9: Сканирование и соединения — алгоритмы, которые решают судьбу продакшена

**Что вы узнаете:**
- Какие алгоритмы PostgreSQL использует для чтения данных: Seq Scan, Index Scan, Bitmap Scan.
- Как работают алгоритмы соединения: Nested Loop, Hash Join, Merge Join.
- Когда каждый алгоритм эффективен, а когда — катастрофа.
- Как читать план запроса с JOIN и понимать, почему выбран тот или иной алгоритм.
- Как влиять на выбор алгоритма через индексы и `work_mem`.
- Как управлять планировщиком через `SET enable_*` для диагностики.

**После прочтения вы сможете:**
- Объяснить, почему Seq Scan быстрее Index Scan для больших выборок.
- Понимать, когда Nested Loop хорош, а когда превращается в проблему.
- Настроить `work_mem` так, чтобы Hash Join не сбрасывался на диск.
- Читать EXPLAIN ANALYZE и видеть, какой алгоритм используется.
- Выбирать правильный индекс под конкретный JOIN.
- Использовать `SET enable_*` для диагностики проблем планировщика.

---

## Содержание

- [9.0 Пролог: запрос, который упал из-за JOIN](#90-пролог-запрос-который-упал-из-за-join)
- [9.1 Seq Scan: последовательное чтение](#91-seq-scan-последовательное-чтение)
- [9.2 Index Scan и Bitmap Scan: чтение через индекс](#92-index-scan-и-bitmap-scan-чтение-через-индекс)
- [9.3 Nested Loop: вложенный цикл](#93-nested-loop-вложенный-цикл)
- [9.4 Hash Join: хэш-соединение](#94-hash-join-хэш-соединение)
- [9.5 Merge Join: соединение слиянием](#95-merge-join-соединение-слиянием)
- [9.6 Как планировщик выбирает алгоритм](#96-как-планировщик-выбирает-алгоритм)
- [9.7 Практика Go: анализ JOIN-запросов](#97-практика-go-анализ-join-запросов)
- [9.8 Выводы и типичные ошибки](#98-выводы-и-типичные-ошибки)
- [9.9 Для быстрого повторения](#99-для-быстрого-повторения)
- [9.10 Вопросы для самопроверки](#910-вопросы-для-самопроверки)
- [9.11 Ответы](#911-ответы)
- [9.12 Куда идти дальше?](#912-куда-идти-дальше)

---

## 9.0 Пролог: запрос, который упал из-за JOIN

Ты написал запрос:

```sql
SELECT 
    u.name,
    o.total
FROM users u
INNER JOIN orders o ON o.user_id = u.id
WHERE u.status = 'active';
```

Утром он работал за 50 мс. Вечером — **5 секунд**. Сервер нагружен, пользователи жалуются.

Ты смотришь `EXPLAIN ANALYZE`:

```
Hash Join  (cost=5000.00..15000.00 rows=100000 width=64)
  Hash Cond: (o.user_id = u.id)
  -> Seq Scan on orders o  (cost=0.00..3000.00 rows=200000 width=16)
  -> Hash  (cost=1000.00..1000.00 rows=50000 width=32)
        -> Seq Scan on users u  (cost=0.00..1000.00 rows=50000 width=32)
              Filter: (status = 'active')
```

### Что такое Hash Join и хэш-таблица

**Хэш-таблица** — структура данных для **мгновенного поиска** по ключу.

```
Хэш-функция: hash(id) = id % 10

  Ячейка 1 → пользователь 1 (Alice)
  Ячейка 2 → пользователь 2 (Bob)

Поиск id=2: hash(2) = 2 → Ячейка 2 → Bob ✓
```

**Hash Join** — алгоритм соединения:

1. Строит хэш-таблицу из меньшей таблицы (`users` по `id`).
2. Для каждой строки из большей (`orders`) мгновенно находит соответствие.
3. Возвращает соединённую строку.

**Проблема:** если хэш-таблица не помещается в `work_mem` — сброс на диск, запрос падает в 100 раз.

### Что произошло вечером

| Время | Пользователей | Хэш-таблица | work_mem (4 МБ) |
|:---|:---|:---|:---|
| Утром | 5 000 | ~0.5 МБ | ✅ Помещается |
| Вечером | 50 000 | ~5 МБ | ❌ Не помещается |

**Решение:** увеличить `work_mem` или переписать запрос.

---

## 9.1 Seq Scan: последовательное чтение

### Определение

**Seq Scan (Sequential Scan)** — алгоритм, который читает **все страницы таблицы подряд**.

**Как работает:**

1. Buffer Manager (Глава 1) читает страницу 0, 1, 2, ... до конца.
2. Для каждой страницы проверяет каждый кортеж: видимость (MVCC, Глава 5), условие WHERE.

---

### Когда Seq Scan эффективен

| Читаем строк | Seq Scan | Index Scan |
|:---|:---|:---|
| > 50% | ✅ Эффективен | ❌ Дорог |
| 10-50% | 🤔 Дешевле, если данные **не в кэше** | 🤔 Дешевле, если данные **в кэше** |
| < 10% | ❌ Зря читает 90% | ✅ Эффективен |

**Пояснение для 10-50%:**

- **Seq Scan дешевле**, если данные не в `shared_buffers` — последовательное чтение (1.0) выгоднее случайного (4.0).
- **Index Scan дешевле**, если данные в кэше — `random_page_cost` уже не важен.
- На SSD `random_page_cost` часто = 1.1 — планировщик чаще выбирает Index Scan.

---

### Seq Scan и shared_buffers

Большой Seq Scan **вытесняет** другие таблицы из кэша (Clock-Sweep, Глава 1).

**Оптимизация `synchronize_seqscans`:** несколько Seq Scan'ов одной таблицы синхронизируются.

### Seq Scan и WAL

Seq Scan не пишет WAL, не создаёт мёртвых кортежей. Но **устанавливает hint bits** (Глава 5).

---

## 9.2 Index Scan и Bitmap Scan: чтение через индекс

### Определение

**Index Scan** — использует B-tree (Глава 3) для поиска, читает нужные страницы.

**Bitmap Scan** — строит битовую карту TID'ов, читает страницы последовательно.

---

### Почему Index Scan = 500 000 чтений, а Seq Scan = 62 500

```
Таблица: 5 млн строк, 80 строк на страницу → 62 500 страниц.
Выборка: 10% = 500 000 строк.
```

**Seq Scan:** 62 500 чтений — по одному на страницу (читает всю страницу целиком).

**Index Scan:** до 500 000 чтений — по одному на строку. Страницы повторяются!

```
Index Scan:
  TID (500, 15) → страница 500 (чтение 1)
  TID (1234, 3) → страница 1234 (чтение 2)
  TID (500, 16) → страница 500 (чтение 3 — ПОВТОР!)
```

**Bitmap Scan решает проблему:** группирует TID'ы по страницам, читает каждую страницу один раз.

---

### Слот и битовая карта

**Слот** — позиция строки на странице (номер в ItemId массиве, Глава 2).

**Битовая карта (exact):** один бит на слот.

```
Страница 500, слот 5, бит = 1 → «строка в слоте 5 попала в выборку»
```

**Lossy bitmap:** один бит на страницу (если карта не помещается в `work_mem`).

```
Страница 100: 1 → «на странице есть совпадения» (неизвестно где)
```

---

### Как Bitmap Scan работает

**Шаг 1: Bitmap Index Scan** — идёт по индексу, собирает TID'ы в карту.

**Шаг 2: Bitmap Heap Scan** — сортирует страницы, читает последовательно.

**Recheck:** проверка условий при чтении (MVCC, Глава 5).

---

### Сравнение сканирований

| Алгоритм | Чтений (для 10%) | Когда |
|:---|:---|:---|
| **Seq Scan** | 62 500 | > 50% |
| **Index Scan** | до 500 000 | < 5% |
| **Bitmap Scan** | ~40 000-50 000 | 5-30% |

---

## 9.3 Nested Loop: вложенный цикл

### Определение

**Nested Loop** — для каждой строки из таблицы A перебирает строки из таблицы B.

```go
for _, user := range users {
    for _, order := range orders {
        if order.UserID == user.ID {
            // соединение
        }
    }
}
```

---

### Сложность

**Без индекса:** O(N × M) — катастрофа.

```
5 000 × 200 000 = 1 МИЛЛИАРД проверок
```

**С индексом на внутренней таблице:** O(N × log M) — эффективен.

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

```sql
EXPLAIN ANALYZE
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON o.user_id = u.id;
```

```
Nested Loop
  -> Seq Scan on users u
  -> Index Scan using idx_orders_user_id on orders o
        Index Cond: (user_id = u.id)
```

---

### Цена индекса

- Занимает место на диске (Глава 2).
- Замедляет INSERT/UPDATE/DELETE (Глава 3).
- VACUUM работает больше (Глава 6).
- WAL растёт быстрее (Глава 1).

**Создавай индекс только под конкретный частый запрос.**

---

## 9.4 Hash Join: хэш-соединение

### Определение

**Hash Join** — строит хэш-таблицу из меньшей таблицы, ищет для большей.

**Сложность:** O(N + M) — линейная.

---

### Как работает

```
1. Читает меньшую таблицу (users) → строит хэш по id.
2. Читает большую (orders) → для каждого ищет в хэше.
3. Нашёл → соединённая строка.
```

---

### Проблема: work_mem

Если хэш не помещается в `work_mem` — сброс на диск.

**Диагностика:**

```
Hash Join
  Batches: 4  ← хэш сброшен на диск!
  temp written: 50000
```

**Решение:**

```sql
SET work_mem = '64MB';
```

---

## 9.5 Merge Join: соединение слиянием

### Определение

**Merge Join** — сортирует обе таблицы по ключу JOIN, сливает.

**Сложность:** O(N + M).

---

### Когда эффективен

**✅ Обе таблицы отсортированы** (есть индексы) — бесплатно.

**✅ Запрос с ORDER BY по ключу JOIN** — результат уже отсортирован.

**❌ Нет индексов** — дорогая сортировка.

---

### EXPLAIN

```
Merge Join
  Merge Cond: (u.id = o.user_id)
  -> Index Scan using users_pkey on users u
  -> Index Scan using idx_orders_user_id on orders o
```

---

## 9.6 Как планировщик выбирает алгоритм

| Условие | Сканирование | JOIN |
|:---|:---|:---|
| Выборка < 5% + индекс | Index Scan | Nested Loop |
| Выборка 5-30% + индекс | Bitmap Scan | Hash Join |
| Выборка > 50% | Seq Scan | Hash Join |
| Большая + индексы + ORDER BY | Index Scan | Merge Join |
| Большая без индексов | Seq Scan | Hash Join |

---

### Команды управления планировщиком (для диагностики)

PostgreSQL позволяет **временно отключать** алгоритмы, чтобы понять, почему планировщик выбрал неоптимальный план.

**Все параметры:**

| Параметр | Что отключает |
|:---|:---|
| `enable_seqscan` | Seq Scan |
| `enable_indexscan` | Index Scan |
| `enable_indexonlyscan` | Index Only Scan |
| `enable_bitmapscan` | Bitmap Scan |
| `enable_nestloop` | Nested Loop JOIN |
| `enable_hashjoin` | Hash Join |
| `enable_mergejoin` | Merge Join |
| `enable_material` | Материализация в Nested Loop |
| `enable_tidscan` | TID Scan |

**Пример использования:**

```sql
-- Проверить, почему планировщик выбрал Hash Join
EXPLAIN SELECT * FROM users u JOIN orders o ON o.user_id = u.id;
-- Hash Join

-- Отключить Hash Join и посмотреть альтернативу
SET enable_hashjoin = off;
EXPLAIN SELECT * FROM users u JOIN orders o ON o.user_id = u.id;
-- Merge Join или Nested Loop

-- Вернуть обратно
SET enable_hashjoin = on;
```

**Область применения:**

```sql
-- На уровне сессии (действует до конца сессии)
SET enable_hashjoin = off;

-- На уровне транзакции
BEGIN;
SET LOCAL enable_hashjoin = off;
COMMIT;
```

**Когда использовать:**

**✅ Диагностика:**

```sql
-- Если Seq Scan медленный, отключи его и посмотри, быстрее ли с индексом:
SET enable_seqscan = off;
EXPLAIN ANALYZE SELECT ...;
-- Если стало быстрее — статистика устарела или нет индекса.
```

**✅ Временный hotfix:**

```sql
-- Если Hash Join сбрасывается на диск, временно отключи его:
SET enable_hashjoin = off;
-- Не забудь потом увеличить work_mem и вернуть обратно!
```

**❌ НЕ ДЕЛАЙ:**

- **Не отключай алгоритмы насовсем** — это маскирует проблему.
- **Не задавай глобально** в `postgresql.conf` без крайней необходимости.

**Почему это не решение:**

```
Проблема: Hash Join сбрасывается на диск (Batches > 1).

Плохое решение:
  SET enable_hashjoin = off;
  → Nested Loop без индекса = 1 млрд проверок = ещё хуже!

Правильное решение:
  ALTER SYSTEM SET work_mem = '64MB';
  SELECT pg_reload_conf();
```

---

## 9.7 Практика Go: анализ JOIN-запросов

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "os"
    "regexp"
    "strconv"
    "strings"

    _ "github.com/jackc/pgx/v5/stdlib"
)

func main() {
    db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    queries := []string{
        "SELECT u.name, o.total FROM users u INNER JOIN orders o ON o.user_id = u.id WHERE u.id = 42",
        "SELECT u.name, o.total FROM users u INNER JOIN orders o ON o.user_id = u.id",
        "SELECT u.name, o.total FROM users u INNER JOIN orders o ON o.user_id = u.id ORDER BY u.id",
    }

    for _, query := range queries {
        analyzeJoin(db, query)
        fmt.Println()
    }
}

func analyzeJoin(db *sql.DB, query string) {
    rows, err := db.Query("EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) " + query)
    if err != nil {
        log.Fatal(err)
    }
    defer rows.Close()

    var planLines []string
    for rows.Next() {
        var line string
        rows.Scan(&line)
        planLines = append(planLines, line)
    }

    planText := strings.Join(planLines, "\n")

    joinType := "Не определён"
    switch {
    case strings.Contains(planText, "Nested Loop"):
        joinType = "Nested Loop"
    case strings.Contains(planText, "Hash Join"):
        joinType = "Hash Join"
    case strings.Contains(planText, "Merge Join"):
        joinType = "Merge Join"
    }

    batches := 0
    if m := regexp.MustCompile(`Batches: (\d+)`).FindStringSubmatch(planText); len(m) > 1 {
        batches, _ = strconv.Atoi(m[1])
    }

    tempWritten := int64(0)
    if m := regexp.MustCompile(`temp written=(\d+)`).FindStringSubmatch(planText); len(m) > 1 {
        tempWritten, _ = strconv.ParseInt(m[1], 10, 64)
    }

    execTime := ""
    if m := regexp.MustCompile(`Execution Time: ([\d.]+) ms`).FindStringSubmatch(planText); len(m) > 1 {
        execTime = m[1]
    }

    fmt.Printf("Запрос: %s\n", query)
    fmt.Printf("  Алгоритм JOIN: %s\n", joinType)
    if batches > 1 {
        fmt.Printf("  ⚠️ Batches: %d — хэш сброшен на диск!\n", batches)
    }
    if tempWritten > 0 {
        fmt.Printf("  ⚠️ temp written: %d — не хватает work_mem!\n", tempWritten)
    }
    fmt.Printf("  Время: %s мс\n", execTime)
}
```

---

## 9.8 Выводы и типичные ошибки

**Что мы узнали?**

PostgreSQL использует три алгоритма сканирования (Seq Scan, Index Scan, Bitmap Scan) и три алгоритма соединения (Nested Loop, Hash Join, Merge Join). Планировщик выбирает самый дешёвый на основе статистики, индексов и work_mem. `SET enable_*` позволяет диагностировать выбор планировщика.

**Типичные ошибки:**

- ❌ Nested Loop без индекса — миллиард проверок.
- ❌ Hash Join с Batches > 1 — хэш сброшен на диск.
- ❌ Index Scan для 50% выборки — случайное чтение.
- ❌ Seq Scan для < 5% — зря читает.
- ❌ Отключать алгоритмы насовсем — маскирует проблему.
- ❌ Не проверять EXPLAIN ANALYZE.

---

## 9.9 Для быстрого повторения

- **Seq Scan** — читает всё, для больших выборок.
- **Index Scan** — точечное, для < 5%.
- **Bitmap Scan** — битовая карта, для 5-30%.
- **Nested Loop** — O(N × log M) с индексом, для малых выборок.
- **Hash Join** — O(N + M), для больших, зависит от work_mem.
- **Merge Join** — O(N + M), для отсортированных + ORDER BY.
- **SET enable_*** — временное отключение алгоритмов для диагностики.

---

## 9.10 Вопросы для самопроверки

1. Почему Seq Scan = 62 500 чтений, а Index Scan = до 500 000 для 10% выборки?
2. Что такое битовая карта? Чем exact отличается от lossy?
3. Что такое Recheck в Bitmap Scan? Почему он нужен?
4. Почему Nested Loop без индекса — катастрофа?
5. Какой индекс нужен для Nested Loop?
6. Что происходит, если хэш в Hash Join не помещается в work_mem?
7. Когда Merge Join бесплатен?
8. Как планировщик выбирает алгоритм?
9. Для чего нужны команды `SET enable_*`?
10. Почему отключение алгоритмов — это не решение проблемы?

---

## 9.11 Ответы

### Ответ 1

Seq Scan читает страницу целиком (80 строк за чтение) — 62 500 чтений на всю таблицу. Index Scan читает каждую строку отдельно (страницы повторяются) — до 500 000 чтений.

### Ответ 2

Битовая карта — компактное представление TID'ов. Exact — бит на слот. Lossy — бит на страницу (когда не помещается в work_mem).

### Ответ 3

Recheck — повторная проверка условий при чтении страницы. Нужен из-за MVCC: строка могла измениться между построением карты и чтением.

### Ответ 4

Сложность O(N × M): 5 000 × 200 000 = 1 млрд проверок.

### Ответ 5

Индекс на внутренней таблице по колонке JOIN:

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

### Ответ 6

Хэш сбрасывается на диск (Batches > 1, temp written). Чтение с диска в 100 раз медленнее.

### Ответ 7

Когда обе таблицы уже отсортированы по ключу JOIN (есть индексы) — слияние без дополнительной сортировки.

### Ответ 8

По стоимости: статистика (cardinality), индексы, work_mem, размер таблиц, ORDER BY.

### Ответ 9

Для диагностики: временно отключить алгоритм и посмотреть, быстрее ли станет. Для временного hotfix.

### Ответ 10

Потому что это маскирует проблему (например, Hash Join сбрасывается на диск), а не решает её (нужно увеличить work_mem).

---

## 9.12 Куда идти дальше?

Мы разобрали **как PostgreSQL выполняет** запросы. Теперь — **как проектировать** таблицы и выбирать типы данных, чтобы запросы были эффективными.

**Глава 10: Типы данных — выбор, который влияет на скорость.**

---

Глава 9 готова. Начинаем главу 10?