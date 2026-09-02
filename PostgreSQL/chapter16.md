# 🔍 Глава 16: Профилирование запросов — как находить узкие места и чинить их

**Что вы узнаете:**
- Как найти медленные запросы через `pg_stat_statements` и `pg_stat_activity`.
- Как читать `EXPLAIN (ANALYZE, BUFFERS)` и видеть, где теряется время.
- Что такое `auto_explain` и как он автоматически логирует медленные запросы.
- Как анализировать статистику использования индексов и таблиц.
- Как находить запросы, которые читают лишние страницы.
- Как профилировать запросы с `EXPLAIN (FORMAT JSON)` и Go-утилитами.
- Связь профилирования с главами 7 (планировщик), 9 (алгоритмы), 15 (конфигурация).

**После прочтения вы сможете:**
- Найти топ-10 самых медленных запросов.
- Определить, почему запрос медленный: Seq Scan, сброс на диск, блокировки.
- Настроить `auto_explain` для автоматического логирования.
- Использовать `EXPLAIN (ANALYZE, BUFFERS)` для точной диагностики.
- Понимать, какие метрики снимать и на что смотреть.
- Чинить найденные проблемы: индексы, work_mem, запросы.

---

## Содержание

- [16.0 Пролог: запрос, который убивает базу](#160-пролог-запрос-который-убивает-базу)
- [16.1 pg_stat_statements: топ медленных запросов](#161-pg_stat_statements-топ-медленных-запросов)
- [16.2 pg_stat_activity: что происходит прямо сейчас](#162-pg_stat_activity-что-происходит-прямо-сейчас)
- [16.3 EXPLAIN ANALYZE BUFFERS: точная диагностика](#163-explain-analyze-buffers-точная-диагностика)
- [16.4 auto_explain: автоматическое логирование](#164-auto_explain-автоматическое-логирование)
- [16.5 Статистика использования индексов и таблиц](#165-статистика-использования-индексов-и-таблиц)
- [16.6 Типичные проблемы и их решения](#166-типичные-проблемы-и-их-решения)
- [16.7 Практика Go: профилирование запросов](#167-практика-go-профилирование-запросов)
- [16.8 Выводы и типичные ошибки](#168-выводы-и-типичные-ошибки)
- [16.9 Для быстрого повторения](#169-для-быстрого-повторения)
- [16.10 Вопросы для самопроверки](#1610-вопросы-для-самопроверки)
- [16.11 Ответы](#1611-ответы)
- [16.12 Куда идти дальше?](#1612-куда-идти-дальше)

---

## 16.0 Пролог: запрос, который убивает базу

Продолжаем историю из главы 15. Ты настроил конфигурацию: `shared_buffers = 16GB`, `work_mem = 256MB`. Но база всё равно иногда «залипает». CPU на 100%, диск шуршит, пользователи жалуются.

Ты открываешь `pg_stat_activity` (глава 1) и видишь:

```
 pid  | state | wait_event | query                                                      | duration
------+-------+------------+------------------------------------------------------------+----------
 1234 | active| DataFileRead| SELECT * FROM orders WHERE created_at > now() - '1 day' | 00:02:35
 5678 | active| DataFileRead| SELECT * FROM orders WHERE created_at > now() - '1 day' | 00:02:31
 9012 | active| DataFileRead| SELECT * FROM orders WHERE created_at > now() - '1 day' | 00:02:28
```

**Один и тот же запрос** выполняется параллельно 10 раз, каждый читает диск по 2.5 минуты.

❓ **Что происходит?**

💡 Ответ: **Seq Scan по огромной таблице**. Запрос `SELECT * FROM orders WHERE created_at > ...` читает **всю таблицу** (глава 9), и таких запросов 10 одновременно. Диск не справляется.

В этой главе разберём, **как находить** такие запросы, **как диагностировать** их, и **как чинить**.

---

## 16.1 pg_stat_statements: топ медленных запросов

### Что это

`pg_stat_statements` — расширение, которое собирает **статистику по всем выполненным запросам**: сколько раз выполнился, сколько времени занял, сколько страниц прочитал.

### Установка

```sql
-- 1. В postgresql.conf:
shared_preload_libraries = 'pg_stat_statements'

-- 2. Рестарт PostgreSQL.

-- 3. Создать расширение:
CREATE EXTENSION pg_stat_statements;

-- 4. Проверить:
SELECT * FROM pg_stat_statements LIMIT 1;
```

### Ключевые колонки

| Колонка | Что означает |
|:---|:---|
| `query` | Нормализованный текст запроса |
| `calls` | Сколько раз выполнялся |
| `total_exec_time` | Общее время выполнения (мс) |
| `mean_exec_time` | Среднее время (мс) |
| `rows` | Сколько строк вернул/изменил |
| `shared_blks_read` | Прочитано страниц с диска |
| `shared_blks_hit` | Прочитано страниц из кэша |
| `temp_blks_written` | Сброшено на диск (сортировки, хэши) |

### Топ-10 медленных запросов

```sql
SELECT 
    query,
    calls,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    ROUND(mean_exec_time::numeric, 2) AS mean_ms,
    rows,
    shared_blks_read,
    temp_blks_written
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

**Вывод:**

```
                    query                     | calls | total_ms  | mean_ms | shared_blks_read | temp_blks_written
----------------------------------------------+-------+-----------+---------+------------------+-------------------
 SELECT * FROM orders WHERE created_at > $1   | 1500  | 225000.00 | 150.00  | 75000            | 0
 SELECT * FROM users WHERE email = $1         | 50000 | 2500.00   | 0.05    | 0                | 0
 SELECT ... ORDER BY created_at               | 300   | 45000.00  | 150.00  | 0                | 12000
```

**Анализ:**

- **Первый запрос:** 1500 вызовов, среднее 150 мс, читает **75 000 страниц с диска**. Seq Scan!
- **Второй запрос:** 50 000 вызовов, среднее 0.05 мс, читает из кэша. Всё хорошо.
- **Третий запрос:** `temp_blks_written = 12000` — сортировка сбрасывается на диск. Увеличь `work_mem`.

### Сброс статистики

```sql
-- Сбросить все счётчики:
SELECT pg_stat_statements_reset();
```

### Нормализация запросов

`pg_stat_statements` **нормализует** запросы: заменяет литералы на `$1`, `$2`:

```sql
-- Эти два запроса:
SELECT * FROM users WHERE id = 42;
SELECT * FROM users WHERE id = 100;

-- Станут одним:
SELECT * FROM users WHERE id = $1;
```

**Поэтому видно частоту по шаблону, а не по конкретным значениям.**

---

## 16.2 pg_stat_activity: что происходит прямо сейчас

Вспомни главу 1: `pg_stat_activity` показывает текущие сессии и их запросы.

### Активные запросы дольше 1 секунды

```sql
SELECT 
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS query_duration,
    now() - xact_start AS xact_duration,
    query
FROM pg_stat_activity
WHERE state <> 'idle'
  AND now() - query_start > interval '1 second'
ORDER BY query_start;
```

**Вывод:**

```
 pid  | usename | state | wait_event | query_duration | query
------+---------+-------+------------+----------------+---------------------------
 1234 | alice   | active| DataFileRead| 00:02:35      | SELECT * FROM orders ...
 5678 | bob     | active| DataFileRead| 00:02:31      | SELECT * FROM orders ...
```

### Что означает wait_event

| wait_event | Что происходит |
|:---|:---|
| `DataFileRead` | Читает страницу с диска — медленный диск или Seq Scan |
| `DataFileWrite` | Пишет страницу на диск — контрольная точка, WAL |
| `WALWrite` | Ждёт записи WAL |
| `transactionid` | Ждёт блокировку (глава 14) |
| `tuple` | Ждёт конкретную версию строки |
| `CPU` | Процессор занят (сортировка, вычисления) |
| `ClientRead` | Ждёт запрос от клиента |

### Найти блокировки

```sql
SELECT 
    waiting.pid AS waiting_pid,
    waiting.query AS waiting_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity waiting
JOIN pg_locks wl ON wl.pid = waiting.pid AND NOT wl.granted
JOIN pg_locks bl ON bl.granted AND bl.relation = wl.relation AND bl.pid <> waiting.pid
JOIN pg_stat_activity blocking ON blocking.pid = bl.pid;
```

---

## 16.3 EXPLAIN ANALYZE BUFFERS: точная диагностика

Вспомни главу 7: `EXPLAIN` показывает план, `EXPLAIN ANALYZE` — реальные значения.

### Полный синтаксис

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 42;
```

**Вывод:**

```
Index Scan using idx_orders_user_id on orders  (cost=0.29..8.31 rows=1 width=64) (actual time=0.012..0.015 rows=1 loops=1)
  Index Cond: (user_id = 42)
  Buffers: shared hit=4 read=0
Planning Time: 0.050 ms
Execution Time: 0.020 ms
```

### Ключевые метрики

| Метрика | Что означает |
|:---|:---|
| `cost` | Оценка планировщика (глава 7) |
| `actual time` | Реальное время (мс) |
| `rows` | Оценка строк |
| `actual rows` | Реальные строки |
| `Buffers: shared hit` | Прочитано из кэша |
| `Buffers: shared read` | Прочитано с диска |
| `temp written` | Сброшено на диск |
| `Execution Time` | Общее время |

### На что смотреть

**1. `rows` vs `actual rows`** — если разница > 10 раз, статистика устарела (глава 7).

**2. `shared read` большой** — читает с диска. Проверь `shared_buffers`, `effective_cache_size`.

**3. `temp written > 0`** — сортировка/хэш сбрасывается на диск. Увеличь `work_mem`.

**4. `Seq Scan` на большой таблице** — нет индекса. Создай.

**5. `actual time` большой при малом `rows`** — что-то не так: блокировки, диск, планировщик.

### Пример: медленный запрос

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE created_at > now() - interval '1 day';
```

```
Seq Scan on orders  (cost=0.00..125000.00 rows=500000 width=64)
  (actual time=0.500..2500.000 rows=500000 loops=1)
  Filter: (created_at > (now() - '1 day'::interval))
  Rows Removed by Filter: 9500000
  Buffers: shared read=125000
Execution Time: 2500.000 ms
```

**Диагноз:**

- Seq Scan — читает **всю таблицу** (10 млн строк).
- `Rows Removed by Filter: 9500000` — 95% строк отброшено.
- `shared read=125000` — все страницы с диска.
- Нет индекса по `created_at`.

**Решение:**

```sql
CREATE INDEX idx_orders_created_at ON orders(created_at);
-- Теперь Index Scan, читает только 500 000 строк.
```

---

## 16.4 auto_explain: автоматическое логирование

### Что это

`auto_explain` — расширение, которое **автоматически логирует план** медленных запросов.

### Установка

```sql
-- В postgresql.conf:
shared_preload_libraries = 'pg_stat_statements,auto_explain'

-- Параметры:
auto_explain.log_min_duration = 1000  -- логировать запросы дольше 1 сек
auto_explain.log_analyze = on          -- включать ANALYZE
auto_explain.log_buffers = on          -- включать BUFFERS
auto_explain.log_format = text         -- формат

-- Рестарт.
```

### Что попадает в лог

```
2024-01-15 10:30:00 UTC [1234] LOG: duration: 2500.000 ms plan:
Seq Scan on orders (cost=0.00..125000.00 rows=500000)
  Filter: (created_at > (now() - '1 day'))
  Buffers: shared read=125000
```

**Теперь каждый медленный запрос (дольше 1 сек) автоматически логируется с планом.**

### Параметры auto_explain

| Параметр | Что делает |
|:---|:---|
| `log_min_duration` | Порог логирования (мс). `0` — все запросы |
| `log_analyze` | Добавлять `ANALYZE` (реальные значения) |
| `log_buffers` | Добавлять `BUFFERS` |
| `log_format` | `text`, `json`, `xml`, `yaml` |
| `log_nested_statements` | Логировать вложенные запросы |

### Как использовать

**В проде:**

```sql
-- Логировать только медленные (> 500 мс):
auto_explain.log_min_duration = 500
auto_explain.log_analyze = off  -- быстрее, но без actual rows
auto_explain.log_buffers = on
```

**При отладке:**

```sql
-- Логировать всё с полным ANALYZE:
SET auto_explain.log_min_duration = 0;
SET auto_explain.log_analyze = on;
SET auto_explain.log_buffers = on;
```

---

## 16.5 Статистика использования индексов и таблиц

### Использование индексов

Вспомни главу 3: `pg_stat_user_indexes` показывает, сколько раз индекс использовался.

```sql
SELECT 
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

**Неиспользуемые индексы (`idx_scan = 0`)** — кандидаты на удаление.

### Статистика таблиц

```sql
SELECT 
    relname,
    n_live_tup,
    n_dead_tup,
    last_autovacuum,
    last_autoanalyze,
    seq_scan,
    idx_scan
FROM pg_stat_user_tables
WHERE relname = 'orders';
```

**Ключевые поля:**

| Поле | Что означает |
|:---|:---|
| `seq_scan` | Сколько раз был Seq Scan |
| `idx_scan` | Сколько раз был Index Scan |
| `n_dead_tup` | Мёртвые кортежи (глава 6) |
| `last_autovacuum` | Когда был последний autovacuum |

**Если `seq_scan` растёт, а `idx_scan` = 0** — индекс не используется или его нет.

### Размер таблиц и индексов

```sql
SELECT 
    relname,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(pg_indexes_size(relid)) AS indexes_size,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 10;
```

---

## 16.6 Типичные проблемы и их решения

### Проблема 1: Seq Scan на большой таблице

**Симптом:**

```
Seq Scan on orders (actual rows=500000, shared read=125000)
```

**Причина:** нет индекса.

**Решение:**

```sql
CREATE INDEX idx_orders_created_at ON orders(created_at);
```

### Проблема 2: Сортировка сбрасывается на диск

**Симптом:**

```
Sort Method: external merge
temp written: 12000
```

**Причина:** `work_mem` слишком мал.

**Решение:**

```sql
ALTER SYSTEM SET work_mem = '256MB';
SELECT pg_reload_conf();
```

### Проблема 3: Hash Join сбрасывается на диск

**Симптом:**

```
Hash Join
  Batches: 4
  temp written: 50000
```

**Причина:** хэш-таблица не помещается в `work_mem`.

**Решение:**

```sql
ALTER SYSTEM SET work_mem = '512MB';
SELECT pg_reload_conf();
```

### Проблема 4: Устаревшая статистика

**Симптом:**

```
rows=100 actual rows=1000000
```

**Причина:** `ANALYZE` давно не запускался.

**Решение:**

```sql
ANALYZE orders;
```

### Проблема 5: Читает с диска, хотя кэш большой

**Симптом:**

```
Buffers: shared read=125000 shared hit=0
```

**Причина:** страницы вытеснены из `shared_buffers`.

**Решение:**

```sql
ALTER SYSTEM SET shared_buffers = '16GB';
-- рестарт
```

### Проблема 6: Блокировки

**Симптом:** `wait_event = 'transactionid'`.

**Решение:** глава 14 — найти блокирующего, оптимизировать запросы.

---

## 16.7 Практика Go: профилирование запросов

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

type QueryStats struct {
    Query          string
    Calls          int64
    TotalExecTime  float64
    MeanExecTime   float64
    SharedBlksRead int64
    TempBlksWriten int64
}

func main() {
    db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Топ медленных запросов
    topQueries(db)

    // Активные запросы
    activeQueries(db)

    // Неиспользуемые индексы
    unusedIndexes(db)
}

func topQueries(db *sql.DB) {
    rows, err := db.Query(`
        SELECT query, calls, total_exec_time, mean_exec_time, shared_blks_read, temp_blks_written
        FROM pg_stat_statements
        ORDER BY total_exec_time DESC
        LIMIT 5
    `)
    if err != nil {
        log.Printf("pg_stat_statements не установлен: %v\n", err)
        return
    }
    defer rows.Close()

    fmt.Println("=== ТОП МЕДЛЕННЫХ ЗАПРОСОВ ===\n")
    for rows.Next() {
        var q QueryStats
        rows.Scan(&q.Query, &q.Calls, &q.TotalExecTime, &q.MeanExecTime, &q.SharedBlksRead, &q.TempBlksWriten)
        fmt.Printf("Запрос: %.80s...\n", q.Query)
        fmt.Printf("  Вызовов: %d, Общее время: %.0f мс, Среднее: %.2f мс\n", q.Calls, q.TotalExecTime, q.MeanExecTime)
        fmt.Printf("  Прочитано с диска: %d страниц, Сброшено на диск: %d\n\n", q.SharedBlksRead, q.TempBlksWriten)
    }
}

func activeQueries(db *sql.DB) {
    rows, err := db.Query(`
        SELECT pid, wait_event, now() - query_start, query
        FROM pg_stat_activity
        WHERE state = 'active' AND now() - query_start > interval '1 second'
    `)
    if err != nil {
        log.Fatal(err)
    }
    defer rows.Close()

    fmt.Println("=== АКТИВНЫЕ ЗАПРОСЫ > 1 СЕК ===\n")
    for rows.Next() {
        var pid int
        var waitEvent sql.NullString
        var duration string
        var query string
        rows.Scan(&pid, &waitEvent, &duration, &query)
        fmt.Printf("PID %d [%s] %s: %.80s...\n", pid, waitEvent.String, duration, query)
    }
}

func unusedIndexes(db *sql.DB) {
    rows, err := db.Query(`
        SELECT relname, indexrelname, pg_size_pretty(pg_relation_size(indexrelid))
        FROM pg_stat_user_indexes
        WHERE idx_scan = 0
        ORDER BY pg_relation_size(indexrelid) DESC
        LIMIT 5
    `)
    if err != nil {
        log.Fatal(err)
    }
    defer rows.Close()

    fmt.Println("\n=== НЕИСПОЛЬЗУЕМЫЕ ИНДЕКСЫ ===\n")
    for rows.Next() {
        var table, index, size string
        rows.Scan(&table, &index, &size)
        fmt.Printf("%s.%s — %s\n", table, index, size)
    }
}
```

---

## 16.8 Выводы и типичные ошибки

**Что мы узнали?**

Профилирование — это система: `pg_stat_statements` для поиска медленных запросов, `pg_stat_activity` для текущих проблем, `EXPLAIN ANALYZE BUFFERS` для точной диагностики, `auto_explain` для автоматического логирования, `pg_stat_user_tables` и `pg_stat_user_indexes` для статистики. Каждая метрика (`shared read`, `temp written`, `wait_event`, `rows vs actual rows`) указывает на конкретную проблему.

**Типичные ошибки:**

- ❌ **Не устанавливать `pg_stat_statements`** — слепой.
- ❌ **Смотреть только на `total_exec_time`** — важны и `shared_blks_read`, и `temp_blks_written`.
- ❌ **Игнорировать `Rows Removed by Filter`** — Seq Scan читает лишнее.
- ❌ **Не настраивать `auto_explain`** — медленные запросы не логируются.
- ❌ **Забывать про `pg_stat_user_indexes`** — неиспользуемые индексы висят годами.

---

## 16.9 Для быстрого повторения

- **pg_stat_statements** — топ запросов: `calls`, `total_exec_time`, `shared_blks_read`, `temp_blks_written`.
- **pg_stat_activity** — текущие запросы: `wait_event`, `query_duration`.
- **EXPLAIN ANALYZE BUFFERS** — точная диагностика: `rows vs actual rows`, `shared hit/read`, `temp written`.
- **auto_explain** — автоматическое логирование медленных запросов.
- **pg_stat_user_indexes** — `idx_scan = 0` → неиспользуемые индексы.
- **Типичные проблемы:** Seq Scan (нет индекса), external merge (work_mem), Batches > 1 (Hash Join), rows ≠ actual (статистика).

---

## 16.10 Вопросы для самопроверки

1. Как найти топ-10 медленных запросов?
2. Что означает `temp_blks_written > 0` в `pg_stat_statements`?
3. Как узнать, что происходит прямо сейчас?
4. Что означает `wait_event = 'DataFileRead'`?
5. Как диагностировать Seq Scan?
6. Что делать, если `Sort Method: external merge`?
7. Как автоматически логировать медленные запросы?
8. Как найти неиспользуемые индексы?
9. Что означает `Rows Removed by Filter`?
10. Как понять, что статистика устарела?

---

## 16.11 Ответы

### Ответ 1

```sql
SELECT query, calls, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

### Ответ 2

Сортировки или Hash Join сбрасываются на диск — `work_mem` слишком мал.

### Ответ 3

```sql
SELECT pid, wait_event, query, now() - query_start
FROM pg_stat_activity
WHERE state <> 'idle';
```

### Ответ 4

Процесс читает страницу с диска — медленный диск, Seq Scan, или холодный кэш.

### Ответ 5

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
-- Смотри: Seq Scan, Rows Removed by Filter, shared read.
```

### Ответ 6

```sql
ALTER SYSTEM SET work_mem = '256MB';
SELECT pg_reload_conf();
```

### Ответ 7

Настроить `auto_explain.log_min_duration` и `auto_explain.log_analyze` в `postgresql.conf`.

### Ответ 8

```sql
SELECT relname, indexrelname
FROM pg_stat_user_indexes
WHERE idx_scan = 0;
```

### Ответ 9

Сколько строк отбросил фильтр. Большое число = читает лишнее, нужен индекс.

### Ответ 10

`EXPLAIN ANALYZE`: если `rows` (оценка) сильно отличается от `actual rows`.

---

## 16.12 Куда идти дальше?

Мы разобрали, как находить и чинить медленные запросы. Теперь — **секционирование**: как делить огромные таблицы на части для ускорения запросов и удаления старых данных.

**Глава 17: Секционирование — как управлять таблицами на миллиарды строк.**