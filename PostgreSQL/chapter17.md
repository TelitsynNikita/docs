# 📦 Глава 17: Секционирование — как управлять таблицами на миллиарды строк

**Что вы узнаете:**
- Что такое секционирование и зачем оно нужно.
- Какие типы секционирования поддерживает PostgreSQL: RANGE, LIST, HASH.
- Как создать секционированную таблицу и партиции.
- Как работает `partition pruning` — отсечение ненужных партиций.
- Как удалять старые данные через `DROP TABLE` вместо `DELETE`.
- Как секционирование влияет на индексы, VACUUM и запросы.
- Как перейти от обычной таблицы к секционированной без даунтайма.
- Связь с главами 2 (страницы), 6 (VACUUM), 12 (временные ряды).

**После прочтения вы сможете:**
- Выбрать тип секционирования под задачу.
- Создать секционированную таблицу с партициями по дате.
- Объяснить, почему `DROP TABLE` партиции быстрее `DELETE`.
- Понимать, как планировщик отсекает партиции.
- Провести миграцию на секционирование без блокировки.
- Проектировать хранение событий, логов, заказов на миллиарды строк.

---

## Содержание

- [17.0 Пролог: таблица, которую невозможно удалить](#170-пролог-таблица-которую-невозможно-удалить)
- [17.1 Что такое секционирование и зачем оно нужно](#171-что-такое-секционирование-и-зачем-оно-нужно)
- [17.2 Типы секционирования: RANGE, LIST, HASH](#172-типы-секционирования-range-list-hash)
- [17.3 Создание секционированной таблицы](#173-создание-секционированной-таблицы)
- [17.4 Partition pruning: как планировщик отсекает партиции](#174-partition-pruning-как-планировщик-отсекает-партиции)
- [17.5 Удаление старых данных: DROP TABLE вместо DELETE](#175-удаление-старых-данных-drop-table-вместо-delete)
- [17.6 Индексы и VACUUM на секционированных таблицах](#176-индексы-и-vacuum-на-секционированных-таблицах)
- [17.7 Миграция на секционирование без даунтайма](#177-миграция-на-секционирование-без-даунтайма)
- [17.8 Практика Go: работа с секционированной таблицей](#178-практика-go-работа-с-секционированной-таблицей)
- [17.9 Выводы и типичные ошибки](#179-выводы-и-типичные-ошибки)
- [17.10 Для быстрого повторения](#1710-для-быстрого-повторения)
- [17.11 Вопросы для самопроверки](#1711-вопросы-для-самопроверки)
- [17.12 Ответы](#1712-ответы)
- [17.13 Куда идти дальше?](#1713-куда-идти-дальше)

---

## 17.0 Пролог: таблица, которую невозможно удалить

В главе 12 мы проектировали таблицу событий:

```sql
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    event_type TEXT NOT NULL,
    payload JSONB,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

Прошло два года. Таблица выросла до **5 миллиардов строк** — 800 ГБ. Бизнес говорит:

> «Удали события старше 90 дней — они не нужны».

Ты выполняешь:

```sql
DELETE FROM events WHERE created_at < now() - interval '90 days';
```

И... запрос висит **часами**. Почему?

- **DELETE** физически не удаляет строки — помечает их через `t_xmax` (глава 5).
- Создаются **миллиарды мёртвых кортежей** (глава 6).
- VACUUM потом будет чистить их **сутками**.
- WAL пишет гигабайты (глава 1).
- Индексы переполняются мёртвыми ссылками (глава 3).

❓ **Как удалять старые данные из огромной таблицы без боли?**

💡 Ответ: **секционирование**. Разделить таблицу на **партиции по дате**. Тогда удаление старых данных = `DROP TABLE events_2023_01` — мгновенно, без мёртвых кортежей, без VACUUM.

В этой главе разберём, **как проектировать секционированные таблицы** и почему они спасают на миллиардах строк.

---

## 17.1 Что такое секционирование и зачем оно нужно

### Определение

> **Секционирование (partitioning)** — разделение одной логической таблицы на **несколько физических** таблиц (партиций) по какому-то правилу.

**Простыми словами:** таблица `events` логически одна, но физически данные разложены по отдельным таблицам: `events_2024_01`, `events_2024_02`, `events_2024_03`...

### Как это выглядит

```sql
-- Логическая таблица (родительская):
CREATE TABLE events (
    id BIGSERIAL,
    event_type TEXT NOT NULL,
    payload JSONB,
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Физические партиции:
CREATE TABLE events_2024_01 PARTITION OF events
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

**Что происходит при INSERT:**

```sql
INSERT INTO events (event_type, payload, created_at)
VALUES ('click', '{}', '2024-01-15');
-- PostgreSQL видит created_at = '2024-01-15'
-- → ищет партицию events_2024_01
-- → вставляет туда.
```

**Что происходит при SELECT:**

```sql
SELECT * FROM events WHERE created_at = '2024-01-15';
-- PostgreSQL видит: нужна только партиция events_2024_01
-- → читает только её, остальные игнорирует.
```

### Зачем нужно секционирование

**1. Быстрое удаление старых данных:**

```sql
-- Без секционирования: DELETE + VACUUM, часы.
-- С секционированием: DROP TABLE events_2023_01, мгновенно.
```

**2. Ускорение запросов:**

```sql
-- Без секционирования: Seq Scan по всей таблице.
-- С секционированием: чтение только нужных партиций.
```

**3. Ускорение VACUUM:**

```sql
-- VACUUM по одной партиции быстрее, чем по всей таблице.
-- Маленькие партиции быстрее чистить.
```

**4. Управление индексами:**

```sql
-- Индекс на партиции меньше, быстрее создаётся.
-- Можно создавать/удалять индексы на конкретных партициях.
```

### Когда секционировать

| Объём таблицы | Нужно секционирование? |
|:---|:---|
| < 10 млн строк | ❌ Не нужно |
| 10-100 млн | 🤔 Думай |
| 100 млн - 1 млрд | ✅ Да |
| > 1 млрд | ✅ Обязательно |

**Правило:** секционируй, когда таблица становится **настолько большой, что DELETE/VACUUM/Seq Scan** превращаются в проблему.

### Жизненный цикл секционированной таблицы

**Ситуация:** у тебя миллиард строк за 5 лет. Решаешь секционировать по месяцам.

**Шаг 1: Создаёшь секционированную таблицу и партиции за весь период:**

```sql
CREATE TABLE events_partitioned (
    id BIGSERIAL,
    event_type TEXT NOT NULL,
    payload JSONB,
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Генерация 60 партиций (5 лет × 12 месяцев):
DO $$
DECLARE
    start_date DATE;
    end_date DATE;
BEGIN
    FOR i IN 0..59 LOOP
        start_date := DATE '2019-01-01' + (i || ' months')::interval;
        end_date := DATE '2019-01-01' + ((i+1) || ' months')::interval;
        
        EXECUTE format(
            'CREATE TABLE events_%s PARTITION OF events_partitioned FOR VALUES FROM (%L) TO (%L)',
            to_char(start_date, 'YYYY_MM'),
            start_date,
            end_date
        );
    END LOOP;
END $$;
```

**Шаг 2: Переносишь данные батчами, переключаешь таблицы (подробно в 17.7).**

**Шаг 3: Настраиваешь автоматическое создание будущих партиций:**

```sql
-- Через pg_cron: 1 числа каждого месяца создавать партицию на следующий месяц.
SELECT cron.schedule(
    'create-next-partition',
    '0 0 1 * *',
    $$
    DO $$
    DECLARE
        next_month DATE := date_trunc('month', now()) + interval '1 month';
        next_month_end DATE := next_month + interval '1 month';
        table_name TEXT := 'events_' || to_char(next_month, 'YYYY_MM');
    BEGIN
        EXECUTE format(
            'CREATE TABLE IF NOT EXISTS %I PARTITION OF events FOR VALUES FROM (%L) TO (%L)',
            table_name, next_month, next_month_end
        );
    END $$;
    $$
);
```

**Шаг 4: Настраиваешь автоматическое удаление старых партиций:**

```sql
SELECT cron.schedule(
    'drop-old-partitions',
    '0 2 1 * *',
    $$
    DO $$
    DECLARE
        cutoff DATE := CURRENT_DATE - interval '90 days';
        r RECORD;
    BEGIN
        FOR r IN
            SELECT tablename
            FROM pg_tables
            WHERE schemaname = 'public'
              AND tablename LIKE 'events_%'
              AND tablename < 'events_' || to_char(cutoff, 'YYYY_MM')
        LOOP
            EXECUTE format('DROP TABLE IF EXISTS %I', r.tablename);
        END LOOP;
    END $$;
    $$
);
```

**Что если забыл создать партицию на новый месяц?**

```sql
INSERT INTO events (event_type, created_at)
VALUES ('click', '2024-06-15');
-- ERROR: no partition of relation "events" found for row
-- Данные НЕ попадут в таблицу!
```

**Решения:**
- Создавать партиции заранее (pg_cron).
- Создать DEFAULT-партицию:

```sql
CREATE TABLE events_default PARTITION OF events DEFAULT;
```

**Что если секционирование больше не нужно?** Объединяешь обратно:

```sql
CREATE TABLE events_plain (LIKE events INCLUDING ALL);
INSERT INTO events_plain SELECT * FROM events;
ALTER TABLE events RENAME TO events_partitioned_old;
ALTER TABLE events_plain RENAME TO events;
DROP TABLE events_partitioned_old;
```

### Когда секционирование реально нужно — сценарии и таблицы

| Сценарий | Таблицы | Ключ партиционирования | Срок хранения | Удаление |
|:---|:---|:---|:---|:---|
| Логи/события | events, logs | дата (день/месяц) | 30-90 дней | DROP партиций |
| Заказы/транзакции | orders, payments | дата (месяц) | годы | DROP или архив |
| Финансы | ledger_entries | дата (квартал/год) | 5-10 лет | Архив |
| IoT/телеметрия | sensor_readings | дата (день) | 7-30 дней | DROP |
| Сессии | sessions, tokens | дата (час/день) | часы-дни | DROP |
| История | history | дата (год) | Вечно | Перенос на HDD |

**Важно:** секционирование по регионам (LIST) — **не** для обычного веб-приложения. Для региональных товаров используй обычную таблицу + индекс:

```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    region_id BIGINT REFERENCES regions(id),
    name TEXT,
    price NUMERIC
);
CREATE INDEX idx_products_region_id ON products(region_id);
-- Запрос: WHERE region_id = 42 → Index Scan (глава 3).
```

LIST-секционирование по регионам оправдано только при **миллиардах строк** и стабильных 5-10 регионах.

### Что секционирование НЕ решает

- **JOIN'ы** — секционирование не ускоряет JOIN двух больших таблиц.
- **Точечные запросы** — если `WHERE id = 42`, индекс работает одинаково.
- **Горизонтальное масштабирование** — секционирование в PostgreSQL — это **на одном сервере** (шардирование — в главах 19-20).

---

## 17.2 Типы секционирования: RANGE, LIST, HASH

PostgreSQL поддерживает **три типа** секционирования.

### RANGE: по диапазонам значений

```sql
CREATE TABLE events (
    id BIGSERIAL,
    event_type TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_01 PARTITION OF events
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

**Когда:** временные данные, запросы по периодам, удаление по дате.

### LIST: по спискам значений

```sql
CREATE TABLE orders (
    id BIGSERIAL,
    region TEXT NOT NULL,
    total NUMERIC,
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (id, region)
) PARTITION BY LIST (region);

CREATE TABLE orders_moscow PARTITION OF orders
FOR VALUES IN ('Moscow');

CREATE TABLE orders_spb PARTITION OF orders
FOR VALUES IN ('SPb');
```

**Когда:** небольшой набор значений (регионы, статусы), запросы по конкретному значению.

### HASH: равномерное распределение

```sql
CREATE TABLE users (
    id BIGSERIAL,
    email TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (id, email)
) PARTITION BY HASH (email);

CREATE TABLE users_p0 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE users_p1 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE users_p2 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE users_p3 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

**Когда:** нет очевидного ключа для RANGE/LIST, нужно равномерное распределение.

### Сравнение

| Тип | Как делит | Когда | Удаление старых |
|:---|:---|:---|:---|
| **RANGE** | По диапазонам | Временные данные | ✅ DROP TABLE |
| **LIST** | По значениям | Категории, регионы | ✅ DROP TABLE |
| **HASH** | По хэшу | Равномерное | ❌ Нельзя по смыслу |

---

## 17.3 Создание секционированной таблицы

### Шаг 1: Родительская таблица

```sql
CREATE TABLE events (
    id BIGSERIAL,
    event_type TEXT NOT NULL,
    payload JSONB,
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (id, created_at)  -- партиционный ключ ВКЛЮЧЁН в PK
) PARTITION BY RANGE (created_at);
```

**Правила:**
- Партиционный ключ обязательно входит в `PRIMARY KEY`.
- Родительская таблица не хранит данные.

### Шаг 2: Партиции

```sql
CREATE TABLE events_2024_01 PARTITION OF events
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

- `FROM` — включительно, `TO` — исключительно.
- Границы не пересекаются.

### Шаг 3: DEFAULT-партиция

```sql
CREATE TABLE events_default PARTITION OF events DEFAULT;
-- Сюда попадут строки без партиции (забытые месяцы).
```

### Шаг 4: Индексы

```sql
-- На родительской таблице → автоматически на всех партициях:
CREATE INDEX idx_events_created_at ON events(created_at);
```

### Шаг 5: Проверка

```sql
SELECT tableoid::regclass AS partition_name, *
FROM events
WHERE created_at = '2024-01-15';
```

---

## 17.4 Partition pruning: как планировщик отсекает партиции

**Partition pruning** — механизм, при котором планировщик **исключает** партиции, не подходящие под `WHERE`.

```sql
EXPLAIN SELECT * FROM events WHERE created_at = '2024-01-15';
```

```
Append
  Subplans Removed: 2
  -> Index Scan on events_2024_01
```

- `Subplans Removed: 2` — две партиции не читались.
- Читается только `events_2024_01`.

**Pruning работает только по партиционному ключу:**

```sql
-- ✅ Сработает:
WHERE created_at = '2024-01-15'

-- ❌ Не сработает (читает все партиции):
WHERE event_type = 'click'

-- ❌ Не сработает (функция над ключом):
WHERE date_trunc('month', created_at) = '2024-01-01'
```

---

## 17.5 Удаление старых данных: DROP TABLE вместо DELETE

### Проблема DELETE

```
DELETE FROM events WHERE created_at < '2024-01-01';
-- 500 млн строк → мёртвые кортежи → VACUUM → часы.
```

### Решение: DROP TABLE партиции

```sql
DROP TABLE events_2023_01;
-- Мгновенно. Без мёртвых кортежей. Без VACUUM.
```

### DETACH + DROP (безопаснее)

```sql
ALTER TABLE events DETACH PARTITION events_2023_01;
-- Партиция стала обычной таблицей.
-- Можно архивировать, проверить, потом:
DROP TABLE events_2023_01;
```

### Автоматизация удаления (pg_cron)

```sql
SELECT cron.schedule(
    'drop-old-partitions',
    '0 2 1 * *',
    $$
    DO $$
    DECLARE
        cutoff DATE := CURRENT_DATE - interval '90 days';
        r RECORD;
    BEGIN
        FOR r IN
            SELECT tablename
            FROM pg_tables
            WHERE schemaname = 'public'
              AND tablename LIKE 'events_%'
              AND tablename < 'events_' || to_char(cutoff, 'YYYY_MM')
        LOOP
            EXECUTE format('DROP TABLE IF EXISTS %I', r.tablename);
        END LOOP;
    END $$;
    $$
);
```

---

## 17.6 Индексы и VACUUM на секционированных таблицах

### Индексы

- **Родительский индекс** → автоматически на всех партициях.
- **Каждая партиция** — физически отдельный индекс.
- **Локальные индексы** — на конкретной партиции.

### VACUUM

- **VACUUM на партиции** — быстрее, чем на всей таблице.
- **Autovacuum работает по партициям** — пороги считаются на партицию.
- **Мёртвые кортежи** локализуются в конкретных партициях.

---

## 17.7 Миграция на секционирование без даунтайма

### Шаги

1. Создай секционированную таблицу (пустую).
2. Создай партиции.
3. Переноси данные **батчами** по партициям.
4. Включи **триггер** для синхронизации новых INSERT'ов.
5. Переключи через `RENAME` (миллисекунды).
6. Проверь COUNT'ы.
7. Удали старую таблицу через несколько дней.

---

## 17.8 Практика Go: работа с секционированной таблицей

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "os"
    "time"

    _ "github.com/jackc/pgx/v5/stdlib"
)

// InsertEvent — вставка события в секционированную таблицу
func InsertEvent(db *sql.DB, eventType string, payload string, createdAt time.Time) error {
    _, err := db.Exec(`
        INSERT INTO events (event_type, payload, created_at)
        VALUES ($1, $2, $3)
    `, eventType, payload, createdAt)
    return err
}

// CreateNextPartition — создание партиции на следующий месяц
func CreateNextPartition(db *sql.DB) error {
    _, err := db.Exec(`
        DO $$
        DECLARE
            next_month DATE := date_trunc('month', now()) + interval '1 month';
            next_month_end DATE := next_month + interval '1 month';
            table_name TEXT := 'events_' || to_char(next_month, 'YYYY_MM');
        BEGIN
            EXECUTE format(
                'CREATE TABLE IF NOT EXISTS %I PARTITION OF events FOR VALUES FROM (%L) TO (%L)',
                table_name, next_month, next_month_end
            );
        END $$;
    `)
    return err
}

// DropOldPartitions — удаление партиций старше N дней
func DropOldPartitions(db *sql.DB, days int) error {
    _, err := db.Exec(`
        DO $$
        DECLARE
            cutoff DATE := CURRENT_DATE - ($1 || ' days')::interval;
            r RECORD;
        BEGIN
            FOR r IN
                SELECT tablename
                FROM pg_tables
                WHERE schemaname = 'public'
                  AND tablename LIKE 'events_%'
                  AND tablename < 'events_' || to_char(cutoff, 'YYYY_MM')
            LOOP
                EXECUTE format('DROP TABLE IF EXISTS %I', r.tablename);
            END LOOP;
        END $$;
    `, days)
    return err
}

func main() {
    db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Вставка события
    err = InsertEvent(db, "click", `{"page": "home"}`, time.Now())
    if err != nil {
        log.Fatal(err)
    }

    // Создание партиции на следующий месяц
    err = CreateNextPartition(db)
    if err != nil {
        log.Fatal(err)
    }

    // Удаление партиций старше 90 дней
    err = DropOldPartitions(db, 90)
    if err != nil {
        log.Fatal(err)
    }

    fmt.Println("Готово")
}
```

---

## 17.9 Выводы и типичные ошибки

**Что мы узнали?**

Секционирование — это разделение логической таблицы на физические партиции по ключу (RANGE, LIST, HASH). Оно решает проблемы огромных таблиц: быстрое удаление через DROP TABLE, ускорение запросов через partition pruning, ускорение VACUUM. Для обычных веб-приложений с региональными товарами секционирование не нужно — достаточно обычной таблицы с индексом.

**Типичные ошибки:**

- ❌ Секционировать таблицу на 1 млн строк — overengineering.
- ❌ Забывать создавать партиции на будущие месяцы — INSERT упадёт.
- ❌ Использовать DELETE вместо DROP TABLE для старых данных.
- ❌ Фильтровать не по партиционному ключу — pruning не работает.
- ❌ Использовать функции над партиционным ключом (`date_trunc`) — pruning ломается.
- ❌ Секционировать по регионам для веб-приложения с товарами — индекс достаточен.

---

## 17.10 Для быстрого повторения

- **Секционирование** — деление таблицы на физические партиции.
- **Типы:** RANGE (по дате), LIST (по значениям), HASH (равномерно).
- **Партиционный ключ** входит в PRIMARY KEY.
- **Partition pruning** — отсечение партиций по WHERE.
- **DROP TABLE партиции** — удаление старых данных без мёртвых кортежей.
- **Индексы на родительской** — автоматически на всех партициях.
- **Миграция** — новая таблица, батчи, RENAME.
- **pg_cron** — автоматизация создания/удаления партиций.

---

## 17.11 Вопросы для самопроверки

1. Что такое секционирование? В чём отличие от обычной таблицы?
2. Какие три типа секционирования? Приведи пример для каждого.
3. Почему партиционный ключ должен входить в PRIMARY KEY?
4. Что такое partition pruning? Как проверить, что он работает?
5. Почему DROP TABLE партиции быстрее DELETE?
6. Как создать партицию на будущий месяц автоматически?
7. Что произойдёт, если INSERT попадает в несуществующую партицию?
8. Как работает VACUUM на секционированной таблице?
9. Как мигрировать на секционирование без даунтайма?
10. Секционировать ли таблицу товаров по регионам для веб-приложения? Почему?

---

## 17.12 Ответы

### Ответ 1

Секционирование — разделение логической таблицы на физические партиции по ключу. Логически — одна таблица, физически — несколько.

### Ответ 2

- **RANGE** — по диапазонам: `events` по `created_at` (месяцы).
- **LIST** — по значениям: `orders` по `region` (Москва, СПб).
- **HASH** — равномерно: `users` по `email` (4 партиции).

### Ответ 3

PostgreSQL должен гарантировать глобальную уникальность PK. С партиционным ключом в PK он проверяет уникальность только в нужной партиции.

### Ответ 4

Partition pruning — отсечение партиций, не подходящих под WHERE. Проверить: `EXPLAIN` → `Subplans Removed: N`.

### Ответ 5

DELETE помечает строки через t_xmax → мёртвые кортежи → VACUUM. DROP TABLE удаляет файл партиции с диска мгновенно.

### Ответ 6

Через pg_cron: задача 1 числа каждого месяца создаёт партицию на следующий месяц.

### Ответ 7

INSERT упадёт с ошибкой: `ERROR: no partition of relation "events" found for row`.

### Ответ 8

VACUUM работает на уровне партиций — быстрее, мёртвые кортежи локализованы.

### Ответ 9

Создать новую таблицу → перенести батчами → триггер синхронизации → RENAME → удалить старую.

### Ответ 10

Нет. Для товаров по регионам (тысячи/миллионы строк) достаточно обычной таблицы + индекс по region_id. Секционирование — для миллиардов строк.

---

## 17.13 Куда идти дальше?

Мы разобрали, как управлять огромными таблицами через секционирование. Теперь — **как управлять соединениями**: что делать, когда приложение открывает 10 000 коннектов к базе.

**Глава 18: Пул соединений — как не утопить PostgreSQL в коннектах.**