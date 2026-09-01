# 🧠 Глава 7: Планировщик запросов — почему один и тот же запрос то летает, то ползает

**Что вы узнаете:**
- Что происходит между SQL-запросом и его выполнением: Parse → Rewrite → Plan → Execute.
- Как PostgreSQL оценивает стоимость разных способов выполнения запроса.
- Что такое статистика, cardinality и selectivity — и как они влияют на выбор плана.
- Почему планировщик иногда выбирает Seq Scan вместо Index Scan.
- Как читать `EXPLAIN` и понимать, что означает каждая цифра.
- Как `ANALYZE` обновляет статистику и почему это критично.
- Почему планировщик ошибается и как это диагностировать.
- Как находить медленные запросы через `pg_stat_statements`.

**После прочтения вы сможете:**
- Объяснить, почему один и тот же запрос может выполняться по-разному.
- Читать `EXPLAIN (ANALYZE, BUFFERS)` и видеть, где теряется время.
- Понимать, когда планировщик ошибается и почему.
- Управлять статистикой через `ANALYZE` и `pg_stats`.
- Предсказывать, какой план выберет PostgreSQL для простых запросов.
- Настраивать cost-параметры под своё железо.
- Находить самые медленные запросы через `pg_stat_statements`.

---

## Содержание

- [7.0 Пролог: тот же запрос, но в 100 раз медленнее](#70-пролог-тот-же-запрос-но-в-100-раз-медленнее)
- [7.1 Что происходит между SQL и выполнением](#71-что-происходит-между-sql-и-выполнением)
- [7.2 Статистика: откуда планировщик знает про данные](#72-статистика-откуда-планировщик-знает-про-данные)
- [7.3 Cardinality и selectivity: сколько строк вернёт запрос](#73-cardinality-и-selectivity-сколько-строк-вернёт-запрос)
- [7.4 Стоимость операций: как планировщик сравнивает планы](#74-стоимость-операций-как-планировщик-сравнивает-планы)
- [7.5 EXPLAIN: читаем план запроса](#75-explain-читаем-план-запроса)
- [7.6 Почему планировщик ошибается](#76-почему-планировщик-ошибается)
- [7.7 pg_stat_statements: находим медленные запросы](#77-pg_stat_statements-находим-медленные-запросы)
- [7.8 Практика Go: анализ планов запросов](#78-практика-go-анализ-планов-запросов)
- [7.9 Выводы и типичные ошибки](#79-выводы-и-типичные-ошибки)
- [7.10 Для быстрого повторения](#710-для-быстрого-повторения)
- [7.11 Вопросы для самопроверки](#711-вопросы-для-самопроверки)
- [7.12 Ответы](#712-ответы)
- [7.13 Куда идти дальше?](#713-куда-идти-дальше)

---

## 7.0 Пролог: тот же запрос, но в 100 раз медленнее

Ты написал запрос:

```sql
SELECT * FROM orders WHERE user_id = 42;
```

Утром он выполнялся за **1 мс**. Вечером — тот же запрос, та же база, тот же `user_id` — **100 мс**. В 100 раз медленнее!

Ты смотришь `EXPLAIN`:

```
Утром:
  Index Scan using idx_orders_user on orders
    Index Cond: (user_id = 42)

Вечером:
  Seq Scan on orders
    Filter: (user_id = 42)
```

**Планировщик изменил решение.** Утром он считал, что Index Scan дешевле. Вечером — что Seq Scan дешевле.

❓ **Почему планировщик передумал?**

Ответ: **изменилась статистика** — данные о том, сколько строк в таблице, как распределены значения, сколько строк вернёт условие `user_id = 42`.

Планировщик — это **не магия**. Это алгоритм, который:

1. Смотрит на статистику (`pg_stats`).
2. Оценивает, сколько строк вернёт каждый вариант плана.
3. Оценивает **стоимость** каждого варианта.
4. Выбирает **самый дешёвый**.

Если статистика устарела или неточна — планировщик **ошибается**. В этой главе мы разберём, как он принимает решения, и как читать его планы.

---

## 7.1 Что происходит между SQL и выполнением

Вспомни Главу 1: Backend-процесс обрабатывает запрос в четыре этапа:

```
Parse → Rewrite → Plan → Execute
```

Разберём **каждый этап** на примере:

```sql
SELECT * FROM orders WHERE user_id = 42;
```

### Этап 1: Parse (разбор)

**Что делает:** Backend-процесс разбирает SQL-текст и превращает его в **дерево разбора** (parse tree).

```
SELECT * FROM orders WHERE user_id = 42;

Parse:

  Дерево разбора:
    - Тип: SELECT
    - Таблица: orders
    - Колонки: * (все)
    - Условие: user_id = 42
```

**Что проверяет:**
- Синтаксис: правильный ли SQL?
- Существование таблицы `orders`?
- Существование колонки `user_id`?

**Кто делает:** Backend-процесс вызывает функцию `raw_parser()`.

### Этап 2: Rewrite (переписывание)

**Что делает:** применяет правила переписывания.

- Раскрывает представления (views).
- Применяет правила (rules), если есть.

```sql
-- Если есть view:
CREATE VIEW active_orders AS 
SELECT * FROM orders WHERE status = 'active';

-- Запрос:
SELECT * FROM active_orders WHERE user_id = 42;

-- Rewrite превращает в:
SELECT * FROM orders WHERE status = 'active' AND user_id = 42;
```

**Кто делает:** Backend-процесс вызывает `pg_rewrite_query()`.

### Этап 3: Plan (планирование)

**Что делает:** генерирует **все возможные** способы выполнения запроса и выбирает **самый дешёвый**.

```
Запрос: SELECT * FROM orders WHERE user_id = 42;

Возможные планы:

  План A: Seq Scan
    - Прочитать всю таблицу (62 500 страниц).
    - Отфильтровать по user_id = 42.
    - Стоимость: 125 000

  План B: Index Scan по idx_orders_user
    - Спуститься по B-tree (3 страницы).
    - Прочитать 1 страницу таблицы.
    - Стоимость: 4

  План C: Bitmap Index Scan
    - Построить битовую карту TID'ов.
    - Прочитать страницы по карте.
    - Стоимость: 50

Планировщик выбирает План B — самый дешёвый.
```

**Кто делает:** Backend-процесс вызывает `planner()`.

**Что учитывает:**
- Статистику из `pg_stats`.
- Стоимость операций (seq_page_cost, random_page_cost).
- Наличие индексов.

### Этап 4: Execute (выполнение)

**Что делает:** выполняет выбранный план, читает страницы через Buffer Manager, возвращает результат.

**Кто делает:** Backend-процесс вызывает `ExecutorRun()`.

### Визуализация: путь запроса

```
SQL-запрос: SELECT * FROM orders WHERE user_id = 42;

  │
  ▼
Parse (raw_parser):
  «Разбираю текст в дерево»
  Проверяю: синтаксис, таблица, колонки
  │
  ▼
Rewrite (pg_rewrite_query):
  «Применяю правила и views»
  │
  ▼
Plan (planner):
  «Генерирую все возможные планы»
  «Оцениваю стоимость каждого»
  «Выбираю самый дешёвый: Index Scan (cost=4)»
  │
  ▼
Execute (ExecutorRun):
  «Выполняю выбранный план»
  «Читаю страницы через Buffer Manager»
  «Возвращаю результат клиенту»
```

### Связь с сущностями из предыдущих глав

| Этап | Что делает | Сущность |
|:---|:---|:---|
| **Parse** | Разбирает SQL | Backend-процесс |
| **Rewrite** | Применяет правила | Backend-процесс |
| **Plan** | Выбирает план | Backend-процесс + статистика из `pg_stats` |
| **Execute** | Выполняет план | Backend-процесс + Buffer Manager + shared_buffers |

### Где хранится статистика

Статистика, которую использует планировщик, хранится в системной таблице **`pg_stats`**:

```sql
SELECT 
    tablename,
    attname,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    histogram_bounds
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'user_id';
```

Разберём `pg_stats` в подглаве 7.2.

### Итог подглавы 7.1

- **Четыре этапа:** Parse → Rewrite → Plan → Execute.
- **Parse** — разбор SQL в дерево.
- **Rewrite** — применение views и правил.
- **Plan** — генерация всех планов, выбор самого дешёвого.
- **Execute** — выполнение через Buffer Manager.
- **Планировщик** работает на этапе Plan, используя статистику из `pg_stats`.

---

## 7.2 Статистика: откуда планировщик знает про данные

В подглаве 7.1 мы узнали, что планировщик использует статистику из `pg_stats`. Теперь разберём **что именно** там хранится и **как** планировщик это использует.

### Что такое pg_stats

`pg_stats` — это **представление** (view) над системной статистикой. Оно показывает, что PostgreSQL знает о **каждой колонке** каждой таблицы.

```sql
SELECT 
    tablename,
    attname,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    histogram_bounds,
    null_frac
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'user_id';
```

Вывод:

```
 tablename | attname  | n_distinct | most_common_vals | most_common_freqs | null_frac
-----------+----------+------------+------------------+-------------------+-----------
 orders    | user_id  | -0.8       | {42, 7, 100}     | {0.1, 0.05, 0.03} | 0.0
```

### Что означает каждое поле

| Поле | Что означает | Пример |
|:---|:---|:---|
| `n_distinct` | Число уникальных значений | -0.8 (доля уникальных от общего) |
| `most_common_vals` | Самые частые значения | {42, 7, 100} |
| `most_common_freqs` | Частота этих значений | {0.1, 0.05, 0.03} (10%, 5%, 3%) |
| `histogram_bounds` | Границы гистограммы | {1, 10, 50, 100, 500, 1000} |
| `null_frac` | Доля NULL | 0.0 (0% NULL) |

### Что такое n_distinct

`n_distinct` показывает, сколько **уникальных значений** в колонке.

**Положительное число** — абсолютное число уникальных:

```
n_distinct = 500
→ В колонке 500 уникальных значений.
```

**Отрицательное число** — доля уникальных от общего числа строк:

```
n_distinct = -0.8
→ 80% строк имеют уникальные значения.
→ В таблице 1 млн строк → 800 000 уникальных значений.
```

### Что такое most_common_vals и most_common_freqs

Для колонок с **перекосом** (некоторые значения встречаются чаще) PostgreSQL хранит **самые частые значения** и их **частоты**:

```
most_common_vals = {42, 7, 100}
most_common_freqs = {0.1, 0.05, 0.03}

Это означает:
  user_id = 42  → 10% строк (100 000 из 1 млн)
  user_id = 7   → 5% строк (50 000 из 1 млн)
  user_id = 100 → 3% строк (30 000 из 1 млн)
```

**Зачем это планировщику:**

Запрос `WHERE user_id = 42` вернёт ~100 000 строк — это **много**. Индекс может быть неэффективен, если нужно прочитать 10% таблицы.

Запрос `WHERE user_id = 999` (редкий) вернёт ~1 строку — индекс идеален.

### Что такое histogram_bounds

Для колонок **без перекоса** (значения распределены равномерно) PostgreSQL хранит **гистограмму** — границы интервалов:

```
histogram_bounds = {1, 10, 50, 100, 500, 1000}

Это означает:
  Значения от 1 до 10 → ~20% строк
  Значения от 10 до 50 → ~20% строк
  Значения от 50 до 100 → ~20% строк
  ...
```

**Как планировщик использует гистограмму:**

Запрос `WHERE user_id BETWEEN 10 AND 50` → попадает в интервал 10-50 → ~20% строк → ~200 000 строк из 1 млн.

### Что такое null_frac

Доля NULL-значений:

```
null_frac = 0.3
→ 30% строк имеют NULL в этой колонке.
```

Запрос `WHERE col IS NULL` → вернёт 30% строк.

### Как собирается статистика

Статистика собирается командой **ANALYZE**:

```sql
ANALYZE orders;
```

**Что делает ANALYZE:**

1. Читает **не всю таблицу**, а **выборку** (обычно 30 000 случайных строк).
2. Считает уникальные значения, частоты, гистограмму.
3. Записывает результат в `pg_statistic` (основа для `pg_stats`).

**Кто выполняет ANALYZE:**

- Вручную: `ANALYZE orders;` — Backend-процесс.
- Автоматически: Autovacuum (Глава 6) — Autovacuum worker.

### Почему статистика устаревает

```sql
INSERT 1 млн строк в таблицу orders;

-- Планировщик всё ещё думает, что в таблице 100 000 строк.
-- Его оценки cardinality неверны.
-- Он может выбрать неправильный план.
```

**Решение:** Autovacuum автоматически запускает ANALYZE, когда таблица изменилась достаточно сильно (порог из Главы 6). Или `VACUUM ANALYZE` (Глава 6, подглава 6.7) — вручную после массовых изменений.

### Как проверить, когда была собрана статистика

```sql
SELECT 
    relname,
    last_analyze,
    last_autoanalyze,
    autoanalyze_count
FROM pg_stat_user_tables
WHERE relname = 'orders';
```

Вывод:

```
 relname | last_analyze | last_autoanalyze     | autoanalyze_count
---------+--------------+----------------------+-------------------
 orders  | (null)       | 2024-01-15 10:30:00 | 42
```

**Если `last_analyze` давно, а таблица изменилась — планировщик работает со устаревшими данными.**

### Практический пример: как статистика влияет на план

**Сценарий 1: Статистика актуальна**

```
Таблица orders: 1 млн строк.
user_id = 42 → 100 000 строк (10%).

Запрос: SELECT * FROM orders WHERE user_id = 42;

Планировщик оценивает:
  Seq Scan: прочитать 62 500 страниц → стоимость 125 000.
  Index Scan: спуск по B-tree + прочитать 100 000 строк из таблицы → стоимость 150 000.

Вывод: Seq Scan дешевле.
```

**Сценарий 2: Статистика устарела**

```
Реально: 10 млн строк.
Статистика: 1 млн строк.

Планировщик думает: 100 000 строк (10% от 1 млн).
Реально: 1 000 000 строк (10% от 10 млн).

Планировщик выбирает Seq Scan (думает, что 100 000 строк).
Но реально Seq Scan читает 10 млн строк → 10 раз медленнее.
```

### 💡 Практика: как поддерживать статистику актуальной

**✅ ОБЯЗАТЕЛЬНО:**

1. **Убедись, что Autovacuum включён** — он автоматически запускает ANALYZE (Глава 6).

2. **После массовых изменений запусти ANALYZE вручную:**
   ```sql
   INSERT 1 млн строк;
   ANALYZE orders;  -- обновить статистику сразу
   ```

   Или очистку и статистику за раз:
   ```sql
   DELETE FROM orders WHERE created_at < '2023-01-01';  -- удалили 50%
   VACUUM ANALYZE orders;  -- очистили мёртвые кортежи И обновили статистику
   ```

**👍 СТОИТ:**

3. **Для колонок с перекосом настрой повышенную точность:**
   ```sql
   ALTER TABLE orders ALTER COLUMN user_id SET STATISTICS 1000;
   ANALYZE orders;
   ```

**❌ НЕ ДЕЛАЙ:**

4. **Не запускай ANALYZE слишком часто** — он читает выборку (30 000 строк), это нагрузка.

### Итог подглавы 7.2

- **pg_stats** — статистика по каждой колонке.
- **n_distinct** — число уникальных значений.
- **most_common_vals/freqs** — частые значения и их частоты.
- **histogram_bounds** — распределение значений.
- **null_frac** — доля NULL.
- **ANALYZE** — собирает статистику (выборка 30 000 строк).
- **Устаревшая статистика** → неправильные оценки → неправильный план.

---

## 7.3 Cardinality и selectivity: сколько строк вернёт запрос

В подглаве 7.2 мы разобрали, **какую** статистику хранит PostgreSQL. Теперь — **как планировщик её использует** для оценки числа строк.

### Что такое cardinality и selectivity

**Selectivity (селективность)** — доля строк, которые удовлетворяют условию. Значение от 0 до 1.

**Cardinality (кардинальность)** — **абсолютное** число строк, которые вернёт условие.

```
Таблица orders: 1 000 000 строк.

Запрос: WHERE user_id = 42.

Selectivity = 0.1 (10% строк подходят)
Cardinality = 1 000 000 × 0.1 = 100 000 строк
```

**Формула:**

```
Cardinality = Selectivity × Общее число строк в таблице
```

### Как планировщик считает selectivity

Для разных условий — **разные формулы**.

#### Условие: `column = value` (равенство)

**Если value в most_common_vals:**

```
most_common_vals = {42, 7, 100}
most_common_freqs = {0.1, 0.05, 0.03}

Запрос: WHERE user_id = 42
→ 42 есть в most_common_vals
→ Selectivity = 0.1 (из most_common_freqs)
```

**Если value НЕ в most_common_vals:**

```
Запрос: WHERE user_id = 999
→ 999 нет в most_common_vals
→ Selectivity = (1 - сумма_freqs_most_common) / (n_distinct - число_most_common)

Пример:
  n_distinct = 500
  most_common_freqs = {0.1, 0.05, 0.03} → сумма = 0.18
  число most_common = 3

  Selectivity = (1 - 0.18) / (500 - 3) = 0.82 / 497 ≈ 0.00165
  Cardinality = 1 000 000 × 0.00165 ≈ 1 650 строк
```

#### Условие: `column BETWEEN a AND b` (диапазон)

**Использует гистограмму:**

```
histogram_bounds = {1, 10, 50, 100, 500, 1000}

Запрос: WHERE user_id BETWEEN 10 AND 50

Попадает в интервал [10, 50]:
  Selectivity ≈ 0.2 (один интервал из пяти)
  Cardinality ≈ 1 000 000 × 0.2 = 200 000 строк
```

#### Условие: `column > value` (больше)

```
histogram_bounds = {1, 10, 50, 100, 500, 1000}

Запрос: WHERE user_id > 500

Интервал [500, 1000] → 20% строк
Selectivity ≈ 0.2
Cardinality ≈ 200 000 строк
```

#### Условие: `column IS NULL`

```
null_frac = 0.3

Запрос: WHERE user_id IS NULL
→ Selectivity = 0.3
→ Cardinality = 300 000 строк
```

### Сводная таблица: как считается selectivity

| Условие | Как считается selectivity |
|:---|:---|
| `col = value` (value в most_common_vals) | `most_common_freqs[i]` |
| `col = value` (value не в most_common_vals) | `(1 - сумма_freqs) / (n_distinct - число_mcv)` |
| `col BETWEEN a AND b` | Доля интервалов гистограммы в диапазоне |
| `col > value` | Доля интервалов гистограммы после value |
| `col IS NULL` | `null_frac` |
| `col LIKE 'abc%'` | Доля строк, начинающихся на 'abc' |
| `col LIKE '%abc%'` | Очень маленькая selectivity (обычно 0.001) |

### Практический пример: как selectivity определяет план

```sql
-- Таблица users: 5 млн строк.
-- Индекс по email (уникальный).
-- Индекс по status (всего 3 значения).

-- Запрос 1: WHERE email = 'alice@example.com'
-- Selectivity = 1 / 5 000 000 ≈ 0.0000002
-- Cardinality = 1 строка
-- План: Index Scan

-- Запрос 2: WHERE status = 'active'
-- Selectivity = 0.33
-- Cardinality = 1 666 667 строк
-- План: Seq Scan
```

### Что происходит, когда cardinality неверна

**Недооценка:**

```
Реально: 1 000 000 строк.
Оценка: 100 строк.
Планировщик выбрал Index Scan → читает 1 млн TID'ов → медленно.
```

**Переоценка:**

```
Реально: 10 строк.
Оценка: 100 000 строк.
Планировщик выбрал Seq Scan → читает всю таблицу → медленно.
```

### Как проверить реальную cardinality

```sql
EXPLAIN (ANALYZE) SELECT * FROM users WHERE status = 'active';

-- Rows: 1666667 (оценка)
-- Actual Rows: 1666667 (реальность)
-- Если разница большая — статистика устарела.
```

### 💡 Практика: что делать, если cardinality неверна

**✅ ОБЯЗАТЕЛЬНО:**

1. **Запусти ANALYZE:**
   ```sql
   ANALYZE users;
   ```

2. **Проверь `Rows` vs `Actual Rows`** — если разница > 10 раз, статистика устарела.

**👍 СТОИТ:**

3. **Для колонок с перекосом увеличь точность:**
   ```sql
   ALTER TABLE users ALTER COLUMN status SET STATISTICS 1000;
   ANALYZE users;
   ```

### Итог подглавы 7.3

- **Selectivity** — доля строк, удовлетворяющих условию.
- **Cardinality** — абсолютное число строк = selectivity × общее число строк.
- **Для равенства** — most_common_vals/freqs.
- **Для диапазонов** — histogram_bounds.
- **Для NULL** — null_frac.
- **Неправильная cardinality** → неправильный план.

---

## 7.4 Стоимость операций: как планировщик сравнивает планы

В подглаве 7.3 мы разобрали, как планировщик оценивает **число строк**. Теперь — как он оценивает **стоимость** каждого варианта.

### Что такое стоимость

**Стоимость (cost)** — **абстрактная единица**. Чем меньше — тем быстрее (по мнению планировщика).

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 42;

-- Index Scan: cost=0.29..8.31
-- Первое число — startup cost (до первой строки)
-- Второе число — total cost (до всех строк)
```

### Как PostgreSQL считает стоимость

| Компонент | Что означает | Параметр |
|:---|:---|:---|
| **CPU** | Обработка строк | `cpu_tuple_cost`, `cpu_operator_cost` |
| **Последовательное чтение** | Чтение страниц подряд | `seq_page_cost` (1.0) |
| **Случайное чтение** | Чтение страниц вразнобой | `random_page_cost` (4.0) |

### Как считается стоимость Seq Scan

```
Стоимость = Число страниц × seq_page_cost + Число строк × cpu_tuple_cost

Пример:
  62 500 страниц × 1.0 + 5 000 000 строк × 0.01 = 112 500
```

### Как считается стоимость Index Scan

```
Стоимость = Спуск по B-tree × random_page_cost + Число строк × random_page_cost

Пример (1 строка):
  3 × 4.0 + 1 × 4.0 = 16
```

**Ключевая разница:** Seq Scan — последовательно (1.0), Index Scan — случайно (4.0).

### Практический пример: когда Seq Scan дешевле

```
Таблица: 62 500 страниц, 5 млн строк.

user_id = 42 → 1 строка:
  Seq Scan:  112 500
  Index Scan: 16 ← ВЫБРАН

user_id = 42 → 500 000 строк (10%):
  Seq Scan:  112 500 ← ВЫБРАН
  Index Scan: 2 000 012
```

**Вот почему планировщик выбирает Seq Scan для больших выборок.**

### 💡 Практика: как влиять на стоимость

**✅ ОБЯЗАТЕЛЬНО:**

1. **Для SSD настрой `random_page_cost = 1.1`:**
   ```sql
   ALTER SYSTEM SET random_page_cost = 1.1;
   SELECT pg_reload_conf();
   ```

**👍 СТОИТ:**

2. **Обнови `effective_cache_size`:**
   ```sql
   ALTER SYSTEM SET effective_cache_size = '8GB';
   SELECT pg_reload_conf();
   ```

**❌ НЕ ДЕЛАЙ:**

3. **Не меняй `seq_page_cost` без понимания.**

### Итог подглавы 7.4

- **Стоимость** — абстрактная единица.
- **Компоненты:** CPU, seq_page_cost, random_page_cost.
- **Seq Scan** — последовательно, дешевле для больших выборок.
- **Index Scan** — случайно, дешевле для малых.
- **Настройка** — random_page_cost, effective_cache_size.

---

## 7.5 EXPLAIN: читаем план запроса

### EXPLAIN vs EXPLAIN ANALYZE

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 42;
-- Показывает оценки

EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;
-- Выполняет запрос и показывает реальные значения
```

### EXPLAIN (ANALYZE, BUFFERS)

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = 42;
```

Вывод:

```
Index Scan using idx_orders_user on orders
  Index Cond: (user_id = 42)
  Buffers: shared hit=4 read=0
  Rows: 1 (actual=1)
  Actual Time: 0.012..0.015 ms
  Planning Time: 0.050 ms
  Execution Time: 0.020 ms
```

**Ключевые метрики:**

| Метрика | Что означает |
|:---|:---|
| `Rows` | Оценка числа строк |
| `actual` | Реальное число строк |
| `Buffers: shared hit` | Прочитано из shared_buffers |
| `Buffers: shared read` | Прочитано с диска |
| `temp read/written` | Временные файлы (сортировка, hash join) |
| `Execution Time` | Время выполнения |

### Как читать дерево плана

```sql
EXPLAIN SELECT u.name, COUNT(o.id)
FROM users u
JOIN orders o ON o.user_id = u.id
GROUP BY u.name;
```

```
HashAggregate
  Group Key: u.name
  → Hash Join
      Hash Cond: (o.user_id = u.id)
      → Seq Scan on users u
      → Seq Scan on orders o
```

**Читай снизу вверх:** источники → операции → финальный результат.

### 💡 Практика: как анализировать план

**✅ ОБЯЗАТЕЛЬНО:**

1. **Используй `EXPLAIN (ANALYZE, BUFFERS)`.**

2. **Проверяй `Rows` vs `actual`** — разница > 10 раз = проблема.

**👍 СТОИТ:**

3. **Ищи `Seq Scan` на больших таблицах** — нужен индекс.

4. **Ищи `temp read/written`** — не хватает `work_mem`.

### Итог подглавы 7.5

- **EXPLAIN** — план без выполнения.
- **EXPLAIN ANALYZE** — с реальными значениями.
- **BUFFERS** — сколько страниц прочитано.
- **Читать снизу вверх.**
- **Диагностика** — Rows vs actual, temp, Seq Scan.

---

## 7.6 Почему планировщик ошибается

### Причина 1: Устаревшая статистика

```sql
-- Симптом: Rows сильно отличается от actual
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'active';
-- Rows: 100000, Actual: 1000000

-- Решение:
ANALYZE orders;
```

### Причина 2: Перекос данных

```sql
-- Для редких значений планировщик может ошибаться
-- Решение: увеличить точность статистики
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;
ANALYZE orders;
```

### Причина 3: Коррелированные колонки

```sql
-- PostgreSQL перемножает selectivity независимо
-- Решение: CREATE STATISTICS
CREATE STATISTICS stats_orders_user_date (dependencies)
ON user_id, created_at
FROM orders;
ANALYZE orders;
```

### Причина 4: Устаревшая VM

```sql
-- Симптом: много Heap Fetches
EXPLAIN (ANALYZE, BUFFERS) SELECT user_id FROM orders WHERE user_id = 42;
-- Heap Fetches: 150

-- Решение:
VACUUM orders;
```

### Причина 5: Неточные cost-параметры

```sql
-- Для SSD:
ALTER SYSTEM SET random_page_cost = 1.1;
SELECT pg_reload_conf();
```

### Причина 6: Неточный effective_cache_size

```sql
ALTER SYSTEM SET effective_cache_size = '48GB';
SELECT pg_reload_conf();
```

### Сводная таблица

| Причина | Симптом | Решение |
|:---|:---|:---|
| Устаревшая статистика | Rows ≠ actual | ANALYZE |
| Перекос данных | Неточная selectivity для редких | SET STATISTICS |
| Корреляция колонок | Недооценка cardinality | CREATE STATISTICS |
| Устаревшая VM | Heap Fetches | VACUUM |
| Неточные cost | Избегает Index Scan | random_page_cost |
| Неточный cache | Думает, что мало кэша | effective_cache_size |

### Итог подглавы 7.6

- **Планировщик ошибается** из-за: статистики, перекоса, корреляции, VM, параметров.
- **Симптом** — Rows ≠ actual.
- **Решения** — ANALYZE, SET STATISTICS, CREATE STATISTICS, VACUUM, cost-параметры.

---

## 7.7 pg_stat_statements: находим медленные запросы

До сих пор мы разбирали, как планировщик выбирает план для **одного конкретного** запроса. Но как найти **какие запросы вообще** выполняются в базе и какие из них самые медленные?

Ответ: **pg_stat_statements** — расширение, которое собирает статистику по **всем** выполненным запросам.

### Что такое pg_stat_statements

**pg_stat_statements** — расширение PostgreSQL, которое:

- Перехватывает **каждый** SQL-запрос.
- Нормализует его (заменяет константы на `$1`, `$2`).
- Считает: сколько раз выполнен, среднее время, общее время, сколько строк вернул.
- Хранит статистику в представлении `pg_stat_statements`.

### Установка и настройка

**Шаг 1: Включи в `postgresql.conf`:**

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all  -- all | top | none
```

**Шаг 2: Перезапусти PostgreSQL:**

```bash
sudo systemctl restart postgresql
```

**Шаг 3: Создай расширение в базе:**

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

**Шаг 4: Проверь, что работает:**

```sql
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
LIMIT 5;
```

### Как это работает под капотом

```
Запросы от клиентов:
  SELECT * FROM orders WHERE user_id = 42;
  SELECT * FROM orders WHERE user_id = 7;
  SELECT * FROM orders WHERE user_id = 100;

pg_stat_statements нормализует:
  SELECT * FROM orders WHERE user_id = $1;

Статистика:
  query = "SELECT * FROM orders WHERE user_id = $1"
  calls = 3                    — сколько раз выполнен
  mean_exec_time = 0.5 мс      — среднее время
  total_exec_time = 1.5 мс     — общее время
```

**Нормализация** — замена констант (`42`, `'active'`, `'2024-01-01'`) на параметры (`$1`, `$2`). Это позволяет группировать **одинаковые по структуре** запросы и видеть, сколько раз они выполнялись.

### Ключевые поля pg_stat_statements

| Поле | Что означает |
|:---|:---|
| `query` | Нормализованный текст запроса |
| `calls` | Сколько раз выполнен |
| `mean_exec_time` | Среднее время выполнения (мс) |
| `total_exec_time` | Общее время выполнения (мс) |
| `rows` | Общее число возвращённых строк |
| `shared_blks_hit` | Страниц прочитано из кэша |
| `shared_blks_read` | Страниц прочитано с диска |
| `temp_blks_written` | Временных страниц записано (сортировка/хэш) |

### Топ медленных запросов

**Топ-10 по общему времени (главный показатель!):**

```sql
SELECT 
    query,
    calls,
    mean_exec_time,
    total_exec_time,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

Вывод:

```
 query                              | calls  | mean_exec_time | total_exec_time | rows
------------------------------------+--------+----------------+-----------------+--------
 SELECT * FROM orders WHERE user_   | 50000  | 150.0          | 7500000.0       | 5000000
 SELECT * FROM users WHERE email =  | 100000 | 0.5            | 50000.0         | 100000
```

**Интерпретация:**

- `total_exec_time` — самый важный показатель. Это **суммарное** время, которое запрос занял у базы.
- Запрос с `total_exec_time = 7 500 000 мс (2 часа)` — главный кандидат на оптимизацию.

### Поиск запросов с проблемами

**1. Запросы, читающие много с диска (не хватает кэша):**

```sql
SELECT 
    query,
    calls,
    shared_blks_read,
    shared_blks_hit,
    ROUND(shared_blks_read::numeric / NULLIF(shared_blks_hit, 0) * 100, 1) AS read_ratio
FROM pg_stat_statements
WHERE shared_blks_read > 0
ORDER BY shared_blks_read DESC
LIMIT 10;
```

**2. Запросы со сбросом на диск (не хватает work_mem):**

```sql
SELECT 
    query,
    calls,
    temp_blks_written
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC
LIMIT 10;
```

**3. Запросы с большим разбросом времени (иногда быстрые, иногда медленные):**

```sql
SELECT 
    query,
    calls,
    mean_exec_time,
    max_exec_time,
    min_exec_time
FROM pg_stat_statements
WHERE calls > 100
  AND max_exec_time > mean_exec_time * 10
ORDER BY max_exec_time DESC
LIMIT 10;
```

### Сброс статистики

```sql
-- Сбросить всю статистику:
SELECT pg_stat_statements_reset();

-- Статистика накапливается с момента включения.
-- Сбрасывай перед тестированием или после оптимизации.
```

### Связь с планировщиком

`pg_stat_statements` показывает **реальную** статистику выполнения — сколько запрос выполнялся, сколько строк вернул. Это **дополняет** `EXPLAIN (ANALYZE, BUFFERS)`, который показывает план для **одного** выполнения.

**Как использовать вместе:**

1. `pg_stat_statements` → находит **какие запросы** медленные (по `total_exec_time`).
2. `EXPLAIN (ANALYZE, BUFFERS)` → показывает **почему** конкретный запрос медленный (Seq Scan, temp, Hash Join с Batches).

### Бизнес через год: запросы замедлились

**Сценарий:**

```
Сегодня:
  Таблица orders: 1 млн строк.
  Запросы летают: mean_exec_time = 1 мс.
  pg_stat_statements показывает: 1000 запросов/сек, всё быстро.

Через год:
  Бизнес вырос. Таблица: 50 млн строк.
  Пользователи жалуются: «всё медленно».

Ты открываешь pg_stat_statements:

  query                              | calls  | mean_exec_time | total_exec_time
  -----------------------------------+--------+----------------+----------------
  SELECT * FROM orders WHERE status  | 800000 | 250.0          | 200000000.0
  SELECT * FROM orders WHERE user_   | 500000 | 5.0            | 2500000.0

Видишь: запрос по status стал выполняться 250 мс вместо 1 мс.
Открываешь EXPLAIN ANALYZE:

  Seq Scan on orders
    Filter: (status = 'active')
    Rows: 100000 (actual=5000000)

Диагноз: статистика устарела (Rows ≠ actual) → планировщик выбрал Seq Scan.
Решение: VACUUM ANALYZE orders;
```

### 💡 Практика: pg_stat_statements

**✅ ОБЯЗАТЕЛЬНО:**

1. **Включи на проде** — `shared_preload_libraries = 'pg_stat_statements'`.

2. **Мониторь `total_exec_time`** — это главный показатель.

3. **Раз в неделю проверяй топ-10 медленных запросов.**

**👍 СТОИТ:**

4. **Настрой алерты в Grafana** (Приложение E):
   - `mean_exec_time > 100 мс` → алерт.
   - `temp_blks_written > 0` → алерт (не хватает work_mem).

**❌ НЕ ДЕЛАЙ:**

5. **Не забывай, что статистика накапливается.** Сбрасывай перед тестированием.

6. **Не игнорируй запросы с большим `total_exec_time`** — даже если `mean_exec_time` небольшой, но вызовов много, суммарное время огромное.

### Итог подглавы 7.7

- **pg_stat_statements** — расширение для сбора статистики по всем запросам.
- **Нормализация** — замена констант на параметры.
- **`total_exec_time`** — главный показатель: суммарное время запроса.
- **Установка** — `shared_preload_libraries` + `CREATE EXTENSION`.
- **Диагностика** — топ медленных, чтение с диска, temp_blks_written.
- **Связь с EXPLAIN** — pg_stat_statements находит «какие», EXPLAIN — «почему».

---

## 7.8 Практика Go: анализ планов запросов

Напишем утилиту, которая выполняет `EXPLAIN (ANALYZE, BUFFERS)` для списка запросов и показывает расхождения между оценками и реальностью.

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

type PlanInfo struct {
	Query         string
	PlanLine      string
	EstimatedRows int64
	ActualRows    int64
	ExecutionTime float64
	SharedRead    int64
	SharedHit     int64
	TempWritten   int64
}

func main() {
	db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	queries := []string{
		"SELECT * FROM orders WHERE user_id = 42",
		"SELECT * FROM orders WHERE status = 'active'",
		"SELECT * FROM orders WHERE created_at >= '2024-01-01'",
	}

	fmt.Println("=== АНАЛИЗ ПЛАНОВ ЗАПРОСОВ ===\n")

	for _, query := range queries {
		plan := analyzeQuery(db, query)
		printPlan(plan)
	}
}

func analyzeQuery(db *sql.DB, query string) PlanInfo {
	plan := PlanInfo{Query: query}

	rows, err := db.Query("EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) " + query)
	if err != nil {
		log.Printf("Ошибка для запроса %q: %v\n", query, err)
		return plan
	}
	defer rows.Close()

	var planLines []string
	for rows.Next() {
		var line string
		if err := rows.Scan(&line); err != nil {
			log.Fatal(err)
		}
		planLines = append(planLines, line)
	}

	planText := strings.Join(planLines, "\n")
	plan.PlanLine = planText

	// Извлекаем Rows
	if m := regexp.MustCompile(`rows=(\d+)`).FindStringSubmatch(planText); len(m) > 1 {
		plan.EstimatedRows, _ = strconv.ParseInt(m[1], 10, 64)
	}

	// Извлекаем actual rows
	if m := regexp.MustCompile(`actual=(\d+)`).FindStringSubmatch(planText); len(m) > 1 {
		plan.ActualRows, _ = strconv.ParseInt(m[1], 10, 64)
	}

	// Извлекаем Execution Time
	if m := regexp.MustCompile(`Execution Time: (\d+\.\d+)`).FindStringSubmatch(planText); len(m) > 1 {
		plan.ExecutionTime, _ = strconv.ParseFloat(m[1], 64)
	}

	// Извлекаем Buffers
	if m := regexp.MustCompile(`shared read=(\d+)`).FindStringSubmatch(planText); len(m) > 1 {
		plan.SharedRead, _ = strconv.ParseInt(m[1], 10, 64)
	}
	if m := regexp.MustCompile(`shared hit=(\d+)`).FindStringSubmatch(planText); len(m) > 1 {
		plan.SharedHit, _ = strconv.ParseInt(m[1], 10, 64)
	}
	if m := regexp.MustCompile(`temp written=(\d+)`).FindStringSubmatch(planText); len(m) > 1 {
		plan.TempWritten, _ = strconv.ParseInt(m[1], 10, 64)
	}

	return plan
}

func printPlan(p PlanInfo) {
	fmt.Printf("Запрос: %s\n", p.Query)
	fmt.Printf("  Оценка строк: %d\n", p.EstimatedRows)
	fmt.Printf("  Реально строк: %d\n", p.ActualRows)

	if p.EstimatedRows > 0 && p.ActualRows > 0 {
		ratio := float64(p.ActualRows) / float64(p.EstimatedRows)
		status := "OK"
		switch {
		case ratio > 10:
			status = "🔴 НЕДООЦЕНКА (реально в 10+ раз больше)"
		case ratio < 0.1:
			status = "🔴 ПЕРЕОЦЕНКА (реально в 10+ раз меньше)"
		case ratio > 3:
			status = "🟡 Заметное расхождение"
		}
		fmt.Printf("  Расхождение: %.1fx — %s\n", ratio, status)
	}

	fmt.Printf("  Время выполнения: %.3f мс\n", p.ExecutionTime)
	fmt.Printf("  Прочитано из кэша: %d страниц\n", p.SharedHit)
	fmt.Printf("  Прочитано с диска: %d страниц\n", p.SharedRead)

	if p.TempWritten > 0 {
		fmt.Printf("  🔴 Временные файлы: %d страниц (не хватает work_mem!)\n", p.TempWritten)
	}

	fmt.Println()
}
```

### Что делает утилита

1. Выполняет `EXPLAIN (ANALYZE, BUFFERS)` для каждого запроса.
2. Извлекает: оценку строк, реальные строки, время, число прочитанных страниц.
3. Сравнивает оценку с реальностью и выдаёт предупреждения.

### Запуск

```bash
export DATABASE_URL="postgres://alice@localhost/mydb"
go run plan-analyzer.go
```

---

## 7.9 Выводы и типичные ошибки

**Что мы узнали?**

Планировщик — это алгоритм, который выбирает самый дешёвый план выполнения запроса. Он опирается на статистику (`pg_stats`), оценивает cardinality (сколько строк) и cost (насколько дорого). `EXPLAIN` показывает эти оценки, `EXPLAIN ANALYZE` — реальность. Планировщик ошибается, когда статистика устарела или неточна. `pg_stat_statements` помогает находить медленные запросы в масштабе всей базы.

**Типичные ошибки:**

- ❌ **Полагаться на EXPLAIN без ANALYZE** — оценки могут быть неточными.
- ❌ **Игнорировать расхождение Rows vs actual** — это главный симптом проблемы.
- ❌ **Не обновлять статистику после массовых изменений.**
- ❌ **Не настраивать random_page_cost для SSD.**
- ❌ **Отключать планировщик** (`enable_indexscan = off`) вместо исправления причины.
- ❌ **Не использовать pg_stat_statements** — без него ты не знаешь, какие запросы реально тормозят.
- ❌ **Оптимизировать запросы с большим `mean_exec_time`, но малым `calls`** — важнее `total_exec_time` (суммарное влияние).

---

## 7.10 Для быстрого повторения

- **Этапы:** Parse → Rewrite → Plan → Execute.
- **Статистика:** `pg_stats` — n_distinct, most_common_vals, histogram_bounds, null_frac.
- **ANALYZE** — собирает статистику (выборка 30 000 строк).
- **Selectivity** — доля строк. **Cardinality** — абсолютное число = selectivity × общее число.
- **Стоимость:** seq_page_cost (1.0), random_page_cost (4.0).
- **Seq Scan** — дешевле для больших выборок. **Index Scan** — для малых.
- **EXPLAIN ANALYZE BUFFERS** — план + реальность + страницы.
- **Ошибки планировщика** — устаревшая статистика, перекос, корреляция, VM, cost-параметры.
- **pg_stat_statements** — статистика по всем запросам: calls, total_exec_time, temp_blks_written.

---

## 7.11 Вопросы для самопроверки

1. Какие четыре этапа проходит запрос? Что делает каждый?
2. Что такое pg_stats? Какие ключевые поля?
3. Что такое n_distinct? Что означает -0.8?
4. Что такое selectivity и cardinality? Как они связаны?
5. Как планировщик считает selectivity для `WHERE col = value`?
6. Что такое cost? Из каких компонентов складывается?
7. Почему Index Scan дороже Seq Scan для больших выборок?
8. Чем EXPLAIN отличается от EXPLAIN ANALYZE?
9. Что означает `Rows` vs `actual` в EXPLAIN ANALYZE?
10. Почему планировщик ошибается? Перечисли 5 причин.
11. Что такое pg_stat_statements? Как он нормализует запросы?
12. Какое поле в pg_stat_statements важнее всего для оптимизации? Почему?

---

## 7.12 Ответы

### Ответ 1

Parse (разбор SQL в дерево) → Rewrite (применение views/правил) → Plan (выбор самого дешёвого плана) → Execute (выполнение).

### Ответ 2

`pg_stats` — статистика по колонкам. Ключевые поля: `n_distinct`, `most_common_vals`, `most_common_freqs`, `histogram_bounds`, `null_frac`.

### Ответ 3

`n_distinct` — число уникальных значений. `-0.8` означает, что 80% строк имеют уникальные значения.

### Ответ 4

Selectivity — доля строк (0-1). Cardinality — абсолютное число = selectivity × общее число строк.

### Ответ 5

Если value в `most_common_vals` → selectivity = частота из `most_common_freqs`. Если нет → `(1 - сумма_freqs) / (n_distinct - число_mcv)`.

### Ответ 6

Cost — абстрактная мера. Компоненты: CPU (cpu_tuple_cost), последовательное чтение (seq_page_cost), случайное чтение (random_page_cost).

### Ответ 7

Index Scan читает страницы **случайно** (random_page_cost = 4.0), Seq Scan — **последовательно** (seq_page_cost = 1.0). Для многих строк случайное чтение дороже.

### Ответ 8

EXPLAIN — только оценки. EXPLAIN ANALYZE — выполняет запрос и показывает реальные значения (actual rows, actual time).

### Ответ 9

`Rows` — оценка планировщика. `actual` — реальное число строк. Большое расхождение = устаревшая статистика.

### Ответ 10

Устаревшая статистика, перекос данных, корреляция колонок, устаревшая VM, неточные cost-параметры.

### Ответ 11

pg_stat_statements — расширение, собирающее статистику по всем выполненным запросам. Нормализует запросы, заменяя константы на параметры (`$1`, `$2`), чтобы группировать одинаковые по структуре.

### Ответ 12

`total_exec_time` — суммарное время выполнения запроса. Это `mean_exec_time × calls`. Оптимизировать нужно запросы с наибольшим `total_exec_time`, потому что они создают наибольшую нагрузку на базу.

---

## 7.13 Куда идти дальше?

Мы разобрали, **как** планировщик выбирает план. Но мы не разобрали **сами операции** — что делает каждая: Seq Scan, Index Scan, Bitmap Scan, Nested Loop, Hash Join, Merge Join.
Глава 8: SQL-мышление — от задачи к запросу