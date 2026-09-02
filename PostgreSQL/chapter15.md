# ⚙️ Глава 15: Конфигурация — как выжать максимум из PostgreSQL на твоём железе

**Что вы узнаете:**
- Как PostgreSQL хранит и читает конфигурацию: `postgresql.conf`, `ALTER SYSTEM`, `SET`.
- Ключевые параметры памяти: `shared_buffers`, `work_mem`, `maintenance_work_mem`, `effective_cache_size`.
- Как настроить WAL: `synchronous_commit`, `checkpoint_timeout`, `max_wal_size`, `wal_compression`.
- Как настроить планировщик: `random_page_cost`, `effective_cache_size`, `default_statistics_target`.
- Как настроить autovacuum: пороги, cost delay, заморозку.
- Как узнать текущие значения и откуда они взялись: `pg_settings`, `SHOW`, `current_setting()`.
- Как применять изменения без рестарта: `ALTER SYSTEM`, `SELECT pg_reload_conf()`.
- Как измерять эффект от изменений конфигурации.

**После прочтения вы сможете:**
- Выставить базовые параметры под конкретное железо.
- Объяснить, как `work_mem` влияет на сортировку и Hash Join.
- Настроить WAL под требования к сохранности и производительности.
- Управлять autovacuum'ом так, чтобы таблицы не раздувались.
- Читать `pg_settings` и понимать, откуда взялось каждое значение.
- Применять изменения на лету без рестарта.
- Понимать связь конфигурации с главами 1-14.

---

## Содержание

- [15.0 Пролог: база тормозит, а конфиг дефолтный](#150-пролог-база-тормозит-а-конфиг-дефолтный)
- [15.1 Как PostgreSQL хранит конфигурацию](#151-как-postgresql-хранит-конфигурацию)
- [15.2 Память: shared_buffers, work_mem, maintenance_work_mem](#152-память-shared_buffers-work_mem-maintenance_work_mem)
- [15.3 WAL и контрольные точки: synchronous_commit, checkpoint_timeout, max_wal_size](#153-wal-и-контрольные-точки-synchronous_commit-checkpoint_timeout-max_wal_size)
- [15.4 Планировщик: random_page_cost, effective_cache_size, default_statistics_target](#154-планировщик-random_page_cost-effective_cache_size-default_statistics_target)
- [15.5 Autovacuum: пороги, cost delay, заморозка](#155-autovacuum-пороги-cost-delay-заморозка)
- [15.6 Как читать pg_settings и менять параметры](#156-как-читать-pg_settings-и-менять-параметры)
- [15.7 Практика Go: проверка конфигурации](#157-практика-go-проверка-конфигурации)
- [15.8 Выводы и типичные ошибки](#158-выводы-и-типичные-ошибки)
- [15.9 Для быстрого повторения](#159-для-быстрого-повторения)
- [15.10 Вопросы для самопроверки](#1510-вопросы-для-самопроверки)
- [15.11 Ответы](#1511-ответы)
- [15.12 Куда идти дальше?](#1512-куда-идти-дальше)

---

## 15.0 Пролог: база тормозит, а конфиг дефолтный

Ты арендовал сервер: 64 ГБ RAM, 16 ядер, NVMe-диск. Установил PostgreSQL, залил базу на 500 ГБ. Запросы тормозят: сортировки сбрасываются на диск, Hash Join'ы еле шевелятся, VACUUM не успевает.

Ты смотришь настройки:

```sql
SHOW shared_buffers;
-- 128MB  ← PostgreSQL установлен с дефолтами!
```

**Дефолтный `shared_buffers` — 128 МБ.** На сервере с 64 ГБ RAM. PostgreSQL использует **0.2%** доступной памяти под кэш страниц. Всё остальное — читает с диска.

**Дефолтный `work_mem` — 4 МБ.** Любая сортировка больше 4 МБ сбрасывается на диск. Hash Join с 10 млн строк сбрасывается на диск (Глава 9).

**Дефолтный `random_page_cost` — 4.0.** На NVMe-диске случайное чтение стоит не в 4 раза дороже последовательного, а в 1.1. Планировщик избегает индексов, выбирая Seq Scan (Глава 7).

❓ **В чём проблема?**

💡 Ответ: **конфигурация не настроена под железо**. PostgreSQL из коробки — это «работает на любом сервере», а не «работает быстро на твоём сервере».

В этой главе разберём, **как настроить PostgreSQL** под конкретное железо и нагрузку.

---

## 15.1 Как PostgreSQL хранит конфигурацию

### Три способа задать параметр

**Способ 1: `postgresql.conf` (файл)**

```bash
# /etc/postgresql/16/main/postgresql.conf
shared_buffers = 16GB
work_mem = 64MB
```

- Применяется при **рестарте** (для большинства параметров).
- Некоторые параметры можно применить через `pg_reload_conf()`.

**Способ 2: `ALTER SYSTEM` (SQL)**

```sql
ALTER SYSTEM SET shared_buffers = '16GB';
-- Записывает в postgresql.auto.conf (отдельный файл)
-- Применяется после рестарта (или reload, если параметр это поддерживает)

SELECT pg_reload_conf();  -- перечитать конфигурацию без рестарта
```

**Способ 3: `SET` (для сессии/транзакции)**

```sql
-- На уровне сессии:
SET work_mem = '256MB';

-- На уровне транзакции:
BEGIN;
SET LOCAL work_mem = '256MB';
COMMIT;
```

- Применяется **мгновенно** для текущей сессии.
- Не влияет на другие сессии.

### Как PostgreSQL решает, какое значение использовать

```
Значение параметра определяется по приоритету:

1. SET (сессия/транзакция)          — самый высокий
2. ALTER ROLE ... SET (для пользователя)
3. ALTER DATABASE ... SET (для базы)
4. postgresql.auto.conf (ALTER SYSTEM)
5. postgresql.conf (файл)
6. Дефолт (встроенный)               — самый низкий
```

**Пример:** если в `postgresql.conf` стоит `work_mem = 4MB`, а ты выполнил `SET work_mem = '256MB'` — для твоей сессии будет `256MB`, для остальных — `4MB`.

### Как узнать текущее значение и откуда оно

```sql
SELECT name, setting, unit, source, sourcefile, sourceline
FROM pg_settings
WHERE name = 'work_mem';
```

**Вывод:**

```
  name   | setting | unit |      source       |        sourcefile        | sourceline
---------+---------+------+-------------------+--------------------------+------------
 work_mem | 4096    | kB   | default           | NULL                     | NULL
```

**Источники (`source`):**

| source | Что означает |
|:---|:---|
| `default` | Встроенный дефолт (не задан явно) |
| `configuration file` | Из `postgresql.conf` |
| `command line` | При старте PostgreSQL |
| `session` | Из `SET` в текущей сессии |
| `override` | Из `ALTER SYSTEM` (auto.conf) |

### Как применить изменения

```sql
-- Перечитать конфигурацию (для параметров, которые это поддерживают):
SELECT pg_reload_conf();

-- Для параметров, требующих рестарт (например, shared_buffers):
-- Нужен полный рестарт PostgreSQL:
-- sudo systemctl restart postgresql
```

**Какие параметры требуют рестарт:**

| Параметр | Требует рестарт? |
|:---|:---|
| `shared_buffers` | ✅ Да |
| `work_mem` | ❌ Нет (reload) |
| `synchronous_commit` | ❌ Нет (reload) |
| `checkpoint_timeout` | ❌ Нет (reload) |
| `max_connections` | ✅ Да |
| `autovacuum` | ❌ Нет (reload) |

---

## 15.2 Память: shared_buffers, work_mem, maintenance_work_mem

### shared_buffers: кэш страниц данных

Вспомни главу 1: `shared_buffers` — общий кэш страниц, которые читает Buffer Manager.

**Дефолт:** 128 МБ — слишком мало.

**Рекомендация:**

```
Сервер 16 ГБ RAM → shared_buffers = 4 ГБ (25%)
Сервер 32 ГБ RAM → shared_buffers = 8 ГБ (25%)
Сервер 64 ГБ RAM → shared_buffers = 16 ГБ (25%)
Сервер 128 ГБ RAM → shared_buffers = 32 ГБ (25%)
```

**Почему 25%, а не 50%:**

- Операционная система тоже кэширует файлы (page cache).
- Если отдать PostgreSQL слишком много — ОС начнёт свопить.
- Двойной кэш (ОС + PostgreSQL) неэффективен.

```sql
-- Проверить текущее значение:
SHOW shared_buffers;

-- Изменить (требует рестарт):
ALTER SYSTEM SET shared_buffers = '16GB';
-- затем рестарт
```

### work_mem: память на сортировки и хэши

Вспомни главу 9: `work_mem` используется для:
- Сортировки (`Sort` в плане)
- Hash Join (хэш-таблица)
- HashAggregate

**Дефолт:** 4 МБ.

**Если не хватает:**

```
Sort Method: external merge  ← сброс на диск
temp written: 50000 страниц  ← медленно!
```

**Рекомендация:**

```
work_mem = (доступная RAM - shared_buffers) / (max_connections * 2)
```

**Пример:**

```
Сервер 64 ГБ RAM:
  shared_buffers = 16 ГБ
  доступно = 48 ГБ
  max_connections = 100

work_mem = 48 ГБ / (100 * 2) = 240 МБ
```

**Почему делить на 2 × max_connections:** каждый запрос может использовать несколько `work_mem` (сортировка + хэш). Если все 100 соединений одновременно сортируют по 240 МБ — это 24 ГБ, что приемлемо.

**Проверить, нужно ли увеличить:**

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ... ORDER BY ...;
-- Ищи: Sort Method: external merge, temp written > 0
```

**Изменить:**

```sql
ALTER SYSTEM SET work_mem = '256MB';
SELECT pg_reload_conf();
```

### maintenance_work_mem: память на обслуживание

Используется для:
- `CREATE INDEX` — сортировка ключей (Глава 3)
- `VACUUM` — обработка страниц (Глава 6)
- `REINDEX`

**Дефолт:** 64 МБ.

**Рекомендация:**

```
maintenance_work_mem = 10% от RAM (но не больше 1-2 ГБ)
```

**Пример:**

```
Сервер 64 ГБ RAM → maintenance_work_mem = 2GB
```

```sql
ALTER SYSTEM SET maintenance_work_mem = '2GB';
SELECT pg_reload_conf();
```

### effective_cache_size: сколько памяти под кэш всего

Вспомни главу 7: планировщик использует `effective_cache_size`, чтобы оценить, сколько страниц поместится в кэш (ОС + PostgreSQL).

**Дефолт:** 4 ГБ.

**Рекомендация:**

```
effective_cache_size = shared_buffers + RAM, доступная ОС под page cache
```

**Пример:**

```
Сервер 64 ГБ RAM, shared_buffers = 16 ГБ:
  effective_cache_size = 48 ГБ (остальная память под кэш ОС)
```

```sql
ALTER SYSTEM SET effective_cache_size = '48GB';
SELECT pg_reload_conf();
```

### Сводная таблица памяти

| Параметр | Что контролирует | Дефолт | Рекомендация |
|:---|:---|:---|:---|
| `shared_buffers` | Кэш страниц | 128 МБ | 25% RAM |
| `work_mem` | Сортировки, Hash Join | 4 МБ | (RAM - shared) / (2×conn) |
| `maintenance_work_mem` | CREATE INDEX, VACUUM | 64 МБ | 10% RAM (до 2 ГБ) |
| `effective_cache_size` | Оценка планировщика | 4 ГБ | ~75% RAM |

---

## 15.3 WAL и контрольные точки: synchronous_commit, checkpoint_timeout, max_wal_size

### synchronous_commit: когда сбрасывать WAL

Вспомни главу 1: при COMMIT WAL сбрасывается на диск через `fsync()`.

**Значения:**

| Значение | Когда fsync | Надёжность |
|:---|:---|:---|
| `on` (дефолт) | При каждом COMMIT | Полная |
| `off` | Фоном (до 600 мс потерь) | Меньше |

**Рекомендация:**

```sql
-- Критичные данные (платежи) — on:
ALTER DATABASE mydb SET synchronous_commit = on;

-- Некритичные (сессии, логи) — off на уровне транзакции:
BEGIN;
SET LOCAL synchronous_commit = off;
INSERT INTO sessions ...;
COMMIT;
```

### checkpoint_timeout: как часто контрольные точки

Вспомни главу 1: Checkpointer раз в N минут сбрасывает все грязные страницы.

**Дефолт:** 5 минут.

**Компромисс:**

| Значение | Восстановление | Нагрузка на диск |
|:---|:---|:---|
| Короткий (2 мин) | Быстрое | Высокая (частые сбросы) |
| Длинный (15 мин) | Медленное | Низкая |

**Рекомендация:**

```sql
ALTER SYSTEM SET checkpoint_timeout = '10min';
SELECT pg_reload_conf();
```

### max_wal_size: сколько WAL хранить до контрольной точки

**Дефолт:** 1 ГБ.

**Если WAL-файлы растут слишком быстро** — контрольная точка наступает раньше `checkpoint_timeout`.

**Рекомендация:**

```sql
-- Для сервера с NVMe: больше WAL, реже контрольные точки
ALTER SYSTEM SET max_wal_size = '8GB';
SELECT pg_reload_conf();
```

### wal_compression: сжимать WAL

**Дефолт:** off.

**Включает сжатие WAL-записей** при полном page image (полная копия страницы в WAL).

```sql
ALTER SYSTEM SET wal_compression = on;
SELECT pg_reload_conf();
```

**Эффект:** WAL меньше, но CPU на сжатие. Для NVMe не нужен, для сети — полезен.

### Сводная таблица WAL

| Параметр | Дефолт | Рекомендация |
|:---|:---|:---|
| `synchronous_commit` | on | on для критичного, off для некритичного |
| `checkpoint_timeout` | 5 мин | 10-15 мин |
| `max_wal_size` | 1 ГБ | 4-8 ГБ |
| `wal_compression` | off | on для сетевых дисков |

---

## 15.4 Планировщик: random_page_cost, effective_cache_size, default_statistics_target

### random_page_cost: стоимость случайного чтения

Вспомни главу 7: планировщик сравнивает Seq Scan (последовательное чтение) и Index Scan (случайное).

**Дефолт:** 4.0 — для HDD.

**Для SSD/NVMe:**

```sql
ALTER SYSTEM SET random_page_cost = 1.1;
SELECT pg_reload_conf();
```

**Почему:** на NVMe случайное чтение почти так же быстро, как последовательное. Дефолт 4.0 заставляет планировщика избегать индексов, выбирая Seq Scan.

### effective_cache_size

Уже разобрали в 15.2 — планировщик использует для оценки.

### default_statistics_target: точность статистики

Вспомни главу 7: `ANALYZE` собирает статистику по выборке.

**Дефолт:** 100.

**Увеличь для колонок с перекосом:**

```sql
ALTER TABLE orders ALTER COLUMN user_id SET STATISTICS 1000;
ANALYZE orders;
```

**Глобально:**

```sql
ALTER SYSTEM SET default_statistics_target = 500;
SELECT pg_reload_conf();
```

**Эффект:** точнее cardinality (Глава 7), но ANALYZE дольше.

### Сводная таблица планировщика

| Параметр | Дефолт | Рекомендация |
|:---|:---|:---|
| `random_page_cost` | 4.0 | 1.1 для SSD/NVMe |
| `effective_cache_size` | 4 ГБ | ~75% RAM |
| `default_statistics_target` | 100 | 300-500 |

---

## 15.5 Autovacuum: пороги, cost delay, заморозка

Вспомни главу 6: Autovacuum автоматически запускает VACUUM и ANALYZE.

### Пороги запуска

```sql
-- Когда запускать VACUUM:
ALTER SYSTEM SET autovacuum_vacuum_scale_factor = 0.1;  -- 10% (дефолт 20%)
ALTER SYSTEM SET autovacuum_vacuum_threshold = 50;

-- Когда запускать ANALYZE:
ALTER SYSTEM SET autovacuum_analyze_scale_factor = 0.05;
```

**Для больших таблиц** (1 млн+ строк) — снизь scale_factor, чтобы VACUUM запускался чаще.

### Cost delay: как не перегрузить диск

```sql
-- Пауза между страницами (мс):
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = 2;

-- Сколько «стоимости» до паузы:
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 200;
```

**Если Autovacuum не успевает** (таблицы растут) — увеличь лимит:

```sql
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = 0;
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 2000;
```

### Заморозка (wraparound)

Вспомни главу 6: заморозка защищает от переполнения XID.

```sql
ALTER SYSTEM SET autovacuum_freeze_max_age = 150000000;  -- 150 млн (дефолт 200 млн)
```

**Не трогай без понимания** — дефолт работает.

### Сводная таблица Autovacuum

| Параметр | Дефолт | Рекомендация |
|:---|:---|:---|
| `autovacuum_vacuum_scale_factor` | 0.2 | 0.05-0.1 |
| `autovacuum_vacuum_cost_delay` | 2 | 2 (или 0 для агрессивного) |
| `autovacuum_vacuum_cost_limit` | 200 | 200-2000 |
| `autovacuum_freeze_max_age` | 200 млн | не трогай без нужды |

---

## 15.6 Как читать pg_settings и менять параметры

### Полный список параметров

```sql
SELECT name, setting, unit, source
FROM pg_settings
ORDER BY name;
```

**Вывод (первые строки):**

```
           name           | setting | unit |      source
--------------------------+---------+------+-------------------
 allow_system_table_mods  | off     |      | default
 application_name         | psql    |      | default
 archive_command          |         |      | default
 shared_buffers           | 16384   | 8kB  | configuration file
 work_mem                 | 4096    | kB   | default
```

### Найти параметры, изменённые от дефолта

```sql
SELECT name, setting, unit, source
FROM pg_settings
WHERE source <> 'default';
```

### SHOW и current_setting()

```sql
SHOW work_mem;

SELECT current_setting('work_mem');
```

### Применение изменений

```sql
-- Изменить глобально:
ALTER SYSTEM SET work_mem = '256MB';
SELECT pg_reload_conf();

-- Изменить для базы:
ALTER DATABASE mydb SET work_mem = '512MB';

-- Изменить для пользователя:
ALTER ROLE alice SET work_mem = '128MB';

-- Изменить для сессии:
SET work_mem = '1GB';
```

### Проверить эффект

```sql
-- До:
EXPLAIN (ANALYZE, BUFFERS) SELECT ... ORDER BY ...;

-- Изменили work_mem.

-- После:
EXPLAIN (ANALYZE, BUFFERS) SELECT ... ORDER BY ...;
-- Сравни: Sort Method, temp written, Execution Time.
```

---

## 15.7 Практика Go: проверка конфигурации

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "os"

    _ "github.com/jackc/pgx/v5/stdlib"
)

type Setting struct {
    Name    string
    Setting string
    Unit    string
    Source  string
}

func main() {
    db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Ключевые параметры
    importantSettings := []string{
        "shared_buffers",
        "work_mem",
        "maintenance_work_mem",
        "effective_cache_size",
        "random_page_cost",
        "synchronous_commit",
        "checkpoint_timeout",
        "max_wal_size",
        "autovacuum_vacuum_scale_factor",
    }

    fmt.Println("=== ВАЖНЫЕ ПАРАМЕТРЫ ===\n")
    fmt.Printf("%-40s %-20s %-10s %s\n", "Параметр", "Значение", "Unit", "Source")
    fmt.Println("----------------------------------------")

    for _, name := range importantSettings {
        var s Setting
        err := db.QueryRow(`
            SELECT name, setting, COALESCE(unit, ''), source
            FROM pg_settings
            WHERE name = $1
        `, name).Scan(&s.Name, &s.Setting, &s.Unit, &s.Source)
        if err != nil {
            log.Printf("Ошибка для %s: %v\n", name, err)
            continue
        }
        fmt.Printf("%-40s %-20s %-10s %s\n", s.Name, s.Setting, s.Unit, s.Source)
    }
}
```

**Запуск:**

```bash
export DATABASE_URL="postgres://alice@localhost/mydb"
go run config-check.go
```

**Вывод:**

```
=== ВАЖНЫЕ ПАРАМЕТРЫ ===

Параметр                                Значение             Unit       Source
----------------------------------------
shared_buffers                          16384                8kB        configuration file
work_mem                                4096                 kB         default
maintenance_work_mem                    65536                kB         default
effective_cache_size                    4GB                  kB         default
random_page_cost                        4                    NULL       default
synchronous_commit                      on                   NULL       default
checkpoint_timeout                      300                  s          default
max_wal_size                            1GB                  kB         default
autovacuum_vacuum_scale_factor          0.2                  NULL       default
```

**Что видно:** почти всё — дефолт. На сервере с 64 ГБ RAM это **катастрофа**.

---

## 15.8 Выводы и типичные ошибки

**Что мы узнали?**

Конфигурация PostgreSQL — это набор параметров, которые управляют памятью, WAL, планировщиком и autovacuum. Дефолтные значения рассчитаны на минимальное железо, поэтому на продакшене их нужно настраивать. Ключевые параметры: `shared_buffers` (25% RAM), `work_mem` (сортировки), `random_page_cost` (для SSD), `checkpoint_timeout`, `max_wal_size`. Изменения применяются через `ALTER SYSTEM` + `pg_reload_conf()` или рестарт.

**Типичные ошибки:**

- ❌ **Оставлять дефолты** — `shared_buffers = 128 МБ` на сервере 64 ГБ.
- ❌ **Ставить `shared_buffers = 50%+ RAM`** — ОС начнёт свопить.
- ❌ **Ставить `work_mem` слишком большим** — все сессии одновременно съедят RAM.
- ❌ **Забывать про `random_page_cost` для SSD** — планировщик избегает индексов.
- ❌ **Менять параметры без измерения** — сначала `EXPLAIN ANALYZE`, потом тюнинг.
- ❌ **Не проверять `pg_settings.source`** — неясно, откуда взялось значение.

---

## 15.9 Для быстрого повторения

- **shared_buffers** — 25% RAM, требует рестарт.
- **work_mem** — сортировки/Hash Join, `(RAM - shared) / (2×conn)`.
- **maintenance_work_mem** — CREATE INDEX, VACUUM, ~10% RAM.
- **effective_cache_size** — для планировщика, ~75% RAM.
- **random_page_cost** — 1.1 для SSD.
- **synchronous_commit** — on для критичного, off для некритичного.
- **checkpoint_timeout** — 10-15 минут.
- **max_wal_size** — 4-8 ГБ.
- **autovacuum_vacuum_scale_factor** — 0.05-0.1 для больших таблиц.
- **pg_settings** — `source` показывает, откуда взялось значение.
- **ALTER SYSTEM SET** + **pg_reload_conf()** — применить без рестарта (не все параметры).

---

## 15.10 Вопросы для самопроверки

1. Какие три способа задать параметр конфигурации?
2. Почему `shared_buffers` должен быть ~25% RAM, а не 50%?
3. Как рассчитать `work_mem` для сервера 64 ГБ, 100 соединений?
4. Что произойдёт, если `work_mem` слишком мал?
5. Почему на SSD нужно ставить `random_page_cost = 1.1`?
6. Как узнать, откуда взялось значение параметра?
7. Какие параметры требуют рестарт, а какие применяются через reload?
8. Как проверить, что `work_mem` достаточно?
9. Что делает `effective_cache_size`?
10. Как настроить autovacuum для больших таблиц?

---

## 15.11 Ответы

### Ответ 1

1. `postgresql.conf` (файл).
2. `ALTER SYSTEM SET` (SQL, auto.conf).
3. `SET` (сессия/транзакция).

### Ответ 2

ОС тоже кэширует файлы (page cache). Если отдать PostgreSQL 50% — ОС начнёт свопить. 25% — баланс между кэшем PostgreSQL и кэшем ОС.

### Ответ 3

```
work_mem = (64 ГБ - 16 ГБ shared_buffers) / (100 × 2) = 240 МБ
```

### Ответ 4

Сортировки и Hash Join сбрасываются на диск: `Sort Method: external merge`, `temp written > 0`. Запросы замедляются в десятки раз.

### Ответ 5

На NVMe случайное чтение почти так же быстро, как последовательное. Дефолт 4.0 заставляет планировщика избегать индексов и выбирать Seq Scan.

### Ответ 6

```sql
SELECT name, setting, source FROM pg_settings WHERE name = 'work_mem';
```

### Ответ 7

`shared_buffers`, `max_connections` — требуют рестарт. `work_mem`, `synchronous_commit`, `checkpoint_timeout`, `random_page_cost` — через `pg_reload_conf()`.

### Ответ 8

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ... ORDER BY ...;
-- Ищи: Sort Method: external merge, temp written
```

### Ответ 9

Планировщик использует `effective_cache_size` для оценки, сколько страниц поместится в кэш (ОС + PostgreSQL). Влияет на выбор между Seq Scan и Index Scan.

### Ответ 10

```sql
ALTER TABLE big_table SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_vacuum_cost_delay = 10
);
```

---

## 15.12 Куда идти дальше?

Мы настроили конфигурацию. Теперь нужно **найти медленные запросы** и понять, что именно их тормозит.

**Глава 16: Профилирование запросов — как находить узкие места и чинить их.**