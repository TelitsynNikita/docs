# 📝 Глава 13: Продвинутый SQL — оконные функции, рекурсия и аналитические инструменты

**Что вы узнаете:**
- Что такое оконные функции и чем они отличаются от GROUP BY.
- Как работает PARTITION BY пошагово: деление на группы, сортировка внутри окна.
- Полный список оконных функций: ROW_NUMBER, RANK, DENSE_RANK, NTILE, FIRST_VALUE, LAST_VALUE, NTH_VALUE, LAG, LEAD.
- Как работают рамки окна (FRAME): ROWS, RANGE, GROUPS, UNBOUNDED PRECEDING, CURRENT ROW.
- Как использовать агрегаты как оконные: SUM OVER, COUNT OVER, AVG OVER, MIN/MAX OVER.
- Что такое именованные окна (WINDOW) и зачем они нужны.
- Как строить накопительные суммы, скользящие средние, сравнение с предыдущим периодом.
- Как работают GROUPING SETS, ROLLUP и CUBE для продвинутой агрегации.
- Как обходить деревья и графы рекурсивными CTE.
- Что такое LATERAL JOIN и когда он полезен.
- Как использовать FILTER, DISTINCT ON, WITH ORDINALITY, UNNEST.
- Как оконные функции влияют на производительность: work_mem, сортировка, сравнение с GROUP BY.

**После прочтения вы сможете:**
- Заменить подзапросы и GROUP BY на оконные функции там, где это эффективнее.
- Написать запрос с накопительной суммой, скользящим средним, топ-N внутри группы.
- Обойти дерево любой глубины рекурсивным CTE.
- Использовать LATERAL для коррелированных подзапросов.
- Строить аналитические отчёты: сравнение с предыдущим периодом, распределение по группам.
- Понимать, когда оконные функции быстрее, а когда медленнее GROUP BY.
- Осознанно выбирать между ROWS, RANGE и GROUPS для рамки окна.
- Связать продвинутый SQL с индексами, сортировкой и work_mem из глав 1-10.

---

## Содержание

- [13.0 Пролог: отчёт, который нельзя написать простым SQL](#130-пролог-отчёт-который-нельзя-написать-простым-sql)
- [13.1 Тестовая таблица и PARTITION BY пошагово](#131-тестовая-таблица-и-partition-by-пошагово)
- [13.2 Полный список оконных функций](#132-полный-список-оконных-функций)
- [13.3 FRAME: рамки окна](#133-frame-рамки-окна)
- [13.4 Агрегаты как оконные](#134-агрегаты-как-оконные)
- [13.5 WINDOW: именованные окна](#135-window-именованные-окна)
- [13.6 Производительность оконных функций](#136-производительность-оконных-функций)
- [13.7 GROUPING SETS, ROLLUP, CUBE](#137-grouping-sets-rollup-cube)
- [13.8 Рекурсивные CTE](#138-рекурсивные-cte)
- [13.9 LATERAL JOIN](#139-lateral-join)
- [13.10 FILTER, DISTINCT ON, WITH ORDINALITY, UNNEST](#1310-filter-distinct-on-with-ordinality-unnest)
- [13.11 Практика Go: аналитические запросы](#1311-практика-go-аналитические-запросы)
- [13.12 Выводы и типичные ошибки](#1312-выводы-и-типичные-ошибки)
- [13.13 Для быстрого повторения](#1313-для-быстрого-повторения)
- [13.14 Вопросы для самопроверки](#1314-вопросы-для-самопроверки)
- [13.15 Ответы](#1315-ответы)
- [13.16 Куда идти дальше?](#1316-куда-идти-дальше)

---

## 13.0 Пролог: отчёт, который нельзя написать простым SQL

Ты работаешь в аналитике интернет-магазина. Бизнес просит отчёт:

> «Покажи для каждого клиента: дату последнего заказа, сумму этого заказа, разницу с предыдущим заказом, и накопительную сумму всех его заказов за всё время».

**Попытка 1 — GROUP BY:**

```sql
SELECT user_id, MAX(created_at) AS last_order_date
FROM orders
GROUP BY user_id;
-- Получили только дату последнего заказа.
-- А сумму этого заказа? А разницу с предыдущим? А накопительную сумму?
```

**Попытка 2 — подзапросы:**

```sql
SELECT u.id,
    (SELECT o.created_at FROM orders o WHERE o.user_id = u.id ORDER BY created_at DESC LIMIT 1) AS last_date,
    (SELECT o.total FROM orders o WHERE o.user_id = u.id ORDER BY created_at DESC LIMIT 1) AS last_total,
    (SELECT SUM(o2.total) FROM orders o2 WHERE o2.user_id = u.id) AS lifetime_sum
FROM users u;
-- Работает, но 3 подзапроса на каждого пользователя. N+1 кошмар (Глава 9).
```

**Попытка 3 — оконные функции:**

```sql
SELECT user_id, created_at, total,
    FIRST_VALUE(created_at) OVER w AS first_order_date,
    LAST_VALUE(created_at) OVER w AS last_order_date,
    LAG(total) OVER w AS prev_total,
    SUM(total) OVER w AS lifetime_sum
FROM orders
WINDOW w AS (PARTITION BY user_id ORDER BY created_at);
-- Один проход по таблице. Всё в одном запросе.
```

❓ **Что это за магия?**

💡 Ответ: **оконные функции**. Они позволяют выполнять вычисления **внутри набора строк**, не схлопывая их в одну (как GROUP BY) и не делая подзапросы (как N+1).

---

## 13.1 Тестовая таблица и PARTITION BY пошагово

### Определение оконной функции

> **Оконная функция (Window Function)** — функция, которая выполняется над **набором строк** (окном), определённым конструкцией `OVER (...)`, и возвращает значение **для каждой строки**, не схлопывая строки в группы (в отличие от GROUP BY).

**Простыми словами:** оконная функция добавляет вычисление **к каждой строке**, опираясь на другие строки из её окна.

**Пример:**

```sql
SELECT id, region, amount,
       SUM(amount) OVER (PARTITION BY region) AS region_sum
FROM sales;
-- Для каждой строки добавляется сумма продаж её региона.
-- Строки не схлопываются — все 8 строк остаются в результате.
```

**В отличие от GROUP BY:**

```sql
-- GROUP BY: 3 региона → 3 строки
SELECT region, SUM(amount)
FROM sales
GROUP BY region;

-- OVER: 8 строк → 8 строк (с добавленной колонкой)
SELECT id, region, amount,
       SUM(amount) OVER (PARTITION BY region)
FROM sales;
```

---

### Тестовые данные

Все примеры в этой главе — на одной таблице:

```sql
CREATE TABLE sales (
    id BIGSERIAL PRIMARY KEY,
    region TEXT NOT NULL,       -- регион
    salesperson TEXT NOT NULL,  -- продавец
    amount NUMERIC(12,2) NOT NULL,  -- сумма продажи
    sale_date DATE NOT NULL          -- дата продажи
);

INSERT INTO sales (region, salesperson, amount, sale_date) VALUES
    ('Moscow', 'Alice',   100, '2024-01-05'),
    ('Moscow', 'Bob',     150, '2024-01-10'),
    ('Moscow', 'Alice',   200, '2024-01-15'),
    ('SPb',    'Charlie', 120, '2024-01-08'),
    ('SPb',    'Alice',   180, '2024-01-12'),
    ('SPb',    'Charlie',  90, '2024-01-20'),
    ('Kazan',  'Bob',     130, '2024-01-03'),
    ('Kazan',  'Charlie', 160, '2024-01-18');
```

**Исходные данные:**

```
 id | region | salesperson | amount | sale_date
----+--------+-------------+--------+------------
  1 | Moscow | Alice       | 100    | 2024-01-05
  2 | Moscow | Bob         | 150    | 2024-01-10
  3 | Moscow | Alice       | 200    | 2024-01-15
  4 | SPb    | Charlie     | 120    | 2024-01-08
  5 | SPb    | Alice       | 180    | 2024-01-12
  6 | SPb    | Charlie     |  90    | 2024-01-20
  7 | Kazan  | Bob         | 130    | 2024-01-03
  8 | Kazan  | Charlie     | 160    | 2024-01-18
```

---

### Что такое PARTITION BY

> **PARTITION BY** — часть синтаксиса оконной функции (внутри `OVER (...)`), которая делит строки на **группы (партиции)**. Оконная функция выполняется **независимо внутри каждой группы**, не смешивая строки из разных групп.

**Простыми словами:** `PARTITION BY region` говорит: «сначала раздели строки по регионам, потом для каждой группы выполни вычисление отдельно».

---

### PARTITION BY region: деление на 3 группы

```sql
SELECT id, region, salesperson, amount,
       COUNT(*) OVER (PARTITION BY region) AS rows_in_region
FROM sales
ORDER BY region, id;
```

**Что происходит пошагово:**

**Шаг 1:** PostgreSQL читает все 8 строк.

**Шаг 2:** `PARTITION BY region` делит строки на 3 группы:

```
Группа 'Kazan':
  id=7, Kazan, Bob, 130
  id=8, Kazan, Charlie, 160

Группа 'Moscow':
  id=1, Moscow, Alice, 100
  id=2, Moscow, Bob, 150
  id=3, Moscow, Alice, 200

Группа 'SPb':
  id=4, SPb, Charlie, 120
  id=5, SPb, Alice, 180
  id=6, SPb, Charlie, 90
```

**Шаг 3:** Для каждой строки `COUNT(*) OVER (PARTITION BY region)` считает количество строк **в её группе**:

```
 id | region | salesperson | amount | rows_in_region
----+--------+-------------+--------+----------------
  7 | Kazan  | Bob         | 130    | 2
  8 | Kazan  | Charlie     | 160    | 2
  1 | Moscow | Alice       | 100    | 3
  2 | Moscow | Bob         | 150    | 3
  3 | Moscow | Alice       | 200    | 3
  4 | SPb    | Charlie     | 120    | 3
  5 | SPb    | Alice       | 180    | 3
  6 | SPb    | Charlie     |  90    | 3
```

**Каждая строка «знает», сколько строк в её регионе.**

---

### PARTITION BY с двумя колонками: region + salesperson

```sql
SELECT id, region, salesperson, amount,
       COUNT(*) OVER (PARTITION BY region, salesperson) AS rows_in_group
FROM sales
ORDER BY region, salesperson, id;
```

**PARTITION BY (region, salesperson)** делит на группы по **паре** значений:

```
Группа (Kazan, Bob):        id=7
Группа (Kazan, Charlie):    id=8
Группа (Moscow, Alice):     id=1, id=3
Группа (Moscow, Bob):       id=2
Группа (SPb, Alice):        id=5
Группа (SPb, Charlie):      id=4, id=6
```

**Результат:**

```
 id | region | salesperson | amount | rows_in_group
----+--------+-------------+--------+---------------
  7 | Kazan  | Bob         | 130    | 1
  8 | Kazan  | Charlie     | 160    | 1
  1 | Moscow | Alice       | 100    | 2
  3 | Moscow | Alice       | 200    | 2
  2 | Moscow | Bob         | 150    | 1
  5 | SPb    | Alice       | 180    | 1
  4 | SPb    | Charlie     | 120    | 2
  6 | SPb    | Charlie     |  90    | 2
```

---

### PARTITION BY + ORDER BY: сортировка внутри группы

```sql
SELECT id, region, salesperson, amount, sale_date,
       ROW_NUMBER() OVER (PARTITION BY region ORDER BY sale_date) AS rn
FROM sales
ORDER BY region, sale_date;
```

**Что происходит:**

**Шаг 1:** `PARTITION BY region` делит на 3 группы.

**Шаг 2:** `ORDER BY sale_date` **внутри каждой группы** сортирует строки по дате.

**Шаг 3:** `ROW_NUMBER()` нумерует строки **внутри каждой группы** по порядку:

```
 id | region | salesperson | amount | sale_date  | rn
----+--------+-------------+--------+------------+----
  7 | Kazan  | Bob         | 130    | 2024-01-03 | 1
  8 | Kazan  | Charlie     | 160    | 2024-01-18 | 2
  1 | Moscow | Alice       | 100    | 2024-01-05 | 1
  2 | Moscow | Bob         | 150    | 2024-01-10 | 2
  3 | Moscow | Alice       | 200    | 2024-01-15 | 3
  4 | SPb    | Charlie     | 120    | 2024-01-08 | 1
  5 | SPb    | Alice       | 180    | 2024-01-12 | 2
  6 | SPb    | Charlie     |  90    | 2024-01-20 | 3
```

**Нумерация начинается с 1 в каждой группе.**

---

### Без PARTITION BY: окно = все строки

```sql
SELECT id, region, amount,
       ROW_NUMBER() OVER (ORDER BY amount DESC) AS rn
FROM sales;
```

**Без PARTITION BY** — одна группа из всех 8 строк. `ORDER BY amount DESC` сортирует все строки:

```
 id | region | amount | rn
----+--------+--------+----
  3 | Moscow | 200    | 1
  5 | SPb    | 180    | 2
  8 | Kazan  | 160    | 3
  2 | Moscow | 150    | 4
  7 | Kazan  | 130    | 5
  4 | SPb    | 120    | 6
  1 | Moscow | 100    | 7
  6 | SPb    |  90    | 8
```

---

### Визуализация: как PARTITION BY делит данные

```
Без PARTITION BY:                    С PARTITION BY region:

Все 8 строк — одна группа:           ┌─ Kazan  (2 строки)
┌─────────────────────┐              │
│ id=1 id=2 id=3      │              ├─ Moscow (3 строки)
│ id=4 id=5 id=6      │              │
│ id=7 id=8           │              └─ SPb    (3 строки)
└─────────────────────┘

Оконная функция видит           Оконная функция видит
ВСЕ строки сразу.               только свою группу.
```

---

### Ключевые правила PARTITION BY

1. **PARTITION BY не схлопывает строки** — в отличие от GROUP BY, строки остаются.
2. **Каждая группа вычисляется независимо** — функция не «видит» строки из других групп.
3. **ORDER BY внутри OVER** — сортирует только внутри группы, не влияет на общий порядок результата.
4. **Без PARTITION BY** — одна группа из всех строк.
5. **Множественные PARTITION BY** — деление по комбинации колонок.

---

## 13.2 Полный список оконных функций: ROW_NUMBER, RANK, DENSE_RANK, NTILE, FIRST_VALUE, LAST_VALUE, NTH_VALUE, LAG, LEAD

### Функции нумерации: ROW_NUMBER, RANK, DENSE_RANK

**Разница в обработке одинаковых значений.**

```sql
SELECT id, region, salesperson, amount,
       ROW_NUMBER() OVER (PARTITION BY region ORDER BY amount DESC) AS row_num,
       RANK()       OVER (PARTITION BY region ORDER BY amount DESC) AS rank,
       DENSE_RANK() OVER (PARTITION BY region ORDER BY amount DESC) AS dense_rank
FROM sales
ORDER BY region, amount DESC;
```

**Результат для Moscow (3 строки):**

```
 id | region | salesperson | amount | row_num | rank | dense_rank
----+--------+-------------+--------+---------+------+------------
  3 | Moscow | Alice       | 200    | 1       | 1    | 1
  2 | Moscow | Bob         | 150    | 2       | 2    | 2
  1 | Moscow | Alice       | 100    | 3       | 3    | 3
-- Одинаковых значений нет — все три функции дают одинаковый результат.
```

**Результат для SPb — есть одинаковые значения:**

Добавим ещё одну строку, чтобы увидеть разницу:

```sql
INSERT INTO sales (region, salesperson, amount, sale_date)
VALUES ('SPb', 'Diana', 120, '2024-01-22');
-- Теперь в SPb два продавца с amount = 120.
```

```sql
SELECT id, region, salesperson, amount,
       ROW_NUMBER() OVER (PARTITION BY region ORDER BY amount DESC) AS row_num,
       RANK()       OVER (PARTITION BY region ORDER BY amount DESC) AS rank,
       DENSE_RANK() OVER (PARTITION BY region ORDER BY amount DESC) AS dense_rank
FROM sales
WHERE region = 'SPb'
ORDER BY amount DESC;
```

```
 id | region | salesperson | amount | row_num | rank | dense_rank
----+--------+-------------+--------+---------+------+------------
  5 | SPb    | Alice       | 180    | 1       | 1    | 1
  4 | SPb    | Charlie     | 120    | 2       | 2    | 2
  9 | SPb    | Diana       | 120    | 3       | 2    | 2
  6 | SPb    | Charlie     |  90    | 4       | 4    | 3
```

**Что видно:**

- `ROW_NUMBER()` — **всегда уникальные** номера: 1, 2, 3, 4. При одинаковых значениях — произвольный порядок (но не случайный).
- `RANK()` — одинаковые значения получают **одинаковый номер**, следующий — с **пропуском**: 1, 2, 2, 4.
- `DENSE_RANK()` — одинаковые значения получают **одинаковый номер**, следующий — **без пропуска**: 1, 2, 2, 3.

**Как запомнить:**
- `ROW_NUMBER` — «просто номер строки».
- `RANK` — «как в спорте: два вторых места, следующее — четвёртое».
- `DENSE_RANK` — «плотный: два вторых, следующее — третье».

### NTILE(n): деление на N равных групп

```sql
SELECT id, region, amount,
       NTILE(3) OVER (PARTITION BY region ORDER BY amount DESC) AS tile
FROM sales
ORDER BY region, amount DESC;
```

**NTILE(3)** делит строки каждой группы на **3 равные части** (по возможности):

```
Moscow (3 строки) → NTILE(3):
  200 → tile 1
  150 → tile 2
  100 → tile 3

SPb (4 строки) → NTILE(3):
  180 → tile 1
  120 → tile 1   ← первая группа получила лишнюю строку
  120 → tile 2
   90 → tile 3
```

**Когда использовать:** деление на квартили, процентили, распределение по группам.

### FIRST_VALUE, LAST_VALUE, NTH_VALUE: значения из окна

```sql
SELECT id, region, amount, sale_date,
       FIRST_VALUE(amount) OVER (PARTITION BY region ORDER BY sale_date) AS first_amount,
       LAST_VALUE(amount) OVER (PARTITION BY region ORDER BY sale_date) AS last_amount,
       NTH_VALUE(amount, 2) OVER (PARTITION BY region ORDER BY sale_date) AS second_amount
FROM sales
ORDER BY region, sale_date;
```

**Результат для Moscow (отсортировано по дате):**

```
 id | region | amount | sale_date  | first_amount | last_amount | second_amount
----+--------+--------+------------+--------------+-------------+---------------
  1 | Moscow | 100    | 2024-01-05 | 100          | 100         | NULL
  2 | Moscow | 150    | 2024-01-10 | 100          | 150         | 150
  3 | Moscow | 200    | 2024-01-15 | 100          | 200         | 150
```

- `FIRST_VALUE` — **всегда** первое значение окна (100).
- `LAST_VALUE` — **по умолчанию** меняется: это последнее значение **до текущей строки** (не обязательно конец окна!). Подробно разберём в 13.3 (FRAME).
- `NTH_VALUE(amount, 2)` — значение **на позиции 2** в окне (150). Для первой строки — NULL (в окне только одна строка).

### LAG и LEAD: сравнение с соседними строками

```sql
SELECT id, region, sale_date, amount,
       LAG(amount) OVER (PARTITION BY region ORDER BY sale_date) AS prev_amount,
       LEAD(amount) OVER (PARTITION BY region ORDER BY sale_date) AS next_amount
FROM sales
ORDER BY region, sale_date;
```

**Результат для Moscow:**

```
 id | sale_date  | amount | prev_amount | next_amount
----+------------+--------+-------------+-------------
  1 | 2024-01-05 | 100    | NULL        | 150
  2 | 2024-01-10 | 150    | 100         | 200
  3 | 2024-01-15 | 200    | 150         | NULL
```

- `LAG(amount)` — значение `amount` из **предыдущей** строки (в порядке окна).
- `LEAD(amount)` — значение `amount` из **следующей** строки.
- Для первой строки `LAG` = NULL (предыдущей нет).
- Для последней строки `LEAD` = NULL (следующей нет).

**С аргументом смещения:**

```sql
LAG(amount, 2)  -- за две строки до текущей
LEAD(amount, 2) -- через две строки после текущей
```

**С дефолтным значением:**

```sql
LAG(amount, 1, 0)  -- если предыдущей нет → 0 вместо NULL
```

### Сводная таблица

| Функция | Что возвращает | Одинаковые значения |
|:---|:---|:---|
| `ROW_NUMBER()` | Порядковый номер строки | Всегда уникальные |
| `RANK()` | Ранг, с пропусками | Одинаковые → одинаковый ранг, следующий с пропуском |
| `DENSE_RANK()` | Ранг, без пропусков | Одинаковые → одинаковый ранг, следующий без пропуска |
| `NTILE(n)` | Номер группы (1..n) | Равные группы |
| `FIRST_VALUE(x)` | Первое значение окна | — |
| `LAST_VALUE(x)` | Последнее значение до текущей строки (по умолчанию) | — |
| `NTH_VALUE(x, n)` | Значение на позиции n | — |
| `LAG(x)` | Значение из предыдущей строки | — |
| `LEAD(x)` | Значение из следующей строки | — |

---

## 13.3 FRAME: рамки окна — ROWS, RANGE, GROUPS, UNBOUNDED PRECEDING, CURRENT ROW

### Что такое FRAME

> **FRAME (рамка окна)** — подмножество строк внутри партиции, которое определяет, какие именно строки видит оконная функция для **текущей строки**.

**Простыми словами:** `PARTITION BY` делит на группы. `ORDER BY` сортирует внутри группы. А **FRAME** говорит: «для этой строки возьми строки от X до Y внутри её группы».

### FRAME по умолчанию

Когда есть `ORDER BY` внутри `OVER`, но FRAME не указан явно, PostgreSQL использует:

```sql
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

**Это означает:** для текущей строки окно = все строки **от начала группы до текущей строки** (включительно).

**Вот почему `LAST_VALUE` для первой строки вернул 100, а не 200:**

```
Строка id=1 (первая в Moscow):
  Окно по умолчанию: строки от начала до id=1
  → только сама строка id=1
  → LAST_VALUE = amount строки id=1 = 100

Строка id=2:
  Окно: строки от начала до id=2
  → id=1, id=2
  → LAST_VALUE = amount строки id=2 = 150

Строка id=3:
  Окно: id=1, id=2, id=3
  → LAST_VALUE = amount строки id=3 = 200
```

### Ключевые элементы FRAME

```sql
ROWS BETWEEN <начало> AND <конец>
```

**Границы:**

| Элемент | Что означает |
|:---|:---|
| `UNBOUNDED PRECEDING` | От начала группы |
| `N PRECEDING` | N строк до текущей |
| `CURRENT ROW` | Текущая строка |
| `N FOLLOWING` | N строк после текущей |
| `UNBOUNDED FOLLOWING` | До конца группы |

**Типы рамок:**

| Тип | Как определяет строки |
|:---|:---|
| `ROWS` | По **физическим** позициям строк |
| `RANGE` | По **значениям** ORDER BY (строки с одинаковыми значениями — вместе) |
| `GROUPS` | По **группам** одинаковых значений |

### ROWS: физические строки

```sql
SELECT id, region, amount, sale_date,
       SUM(amount) OVER (
           PARTITION BY region
           ORDER BY sale_date
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_sum
FROM sales
ORDER BY region, sale_date;
```

**Что происходит для Moscow (отсортировано по дате):**

```
 id | amount | sale_date  | running_sum
----+--------+------------+-------------
  1 | 100    | 2024-01-05 | 100          ← строки 1..1
  2 | 150    | 2024-01-10 | 250          ← строки 1..2
  3 | 200    | 2024-01-15 | 450          ← строки 1..3
```

**Накопительная сумма:** каждая строка суммирует себя и все предыдущие в группе.

### ROWS: скользящее окно

```sql
SELECT id, region, amount, sale_date,
       AVG(amount) OVER (
           PARTITION BY region
           ORDER BY sale_date
           ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
       ) AS moving_avg_2
FROM sales
ORDER BY region, sale_date;
```

**Для Moscow:**

```
 id | amount | moving_avg_2
----+--------+-------------
  1 | 100    | 100          ← (100) / 1
  2 | 150    | 125          ← (100 + 150) / 2
  3 | 200    | 175          ← (150 + 200) / 2
```

**Скользящее среднее по 2 строкам:** текущая + предыдущая.

### RANGE: по значениям, одинаковые значения — вместе

**Отличие RANGE от ROWS** проявляется, когда есть **одинаковые значения** в `ORDER BY`.

Добавим одинаковые даты:

```sql
INSERT INTO sales (region, salesperson, amount, sale_date)
VALUES ('Moscow', 'Diana', 50, '2024-01-10');
-- Теперь в Moscow две строки с sale_date = 2024-01-10.
```

**ROWS — каждая строка отдельно:**

```sql
SELECT id, amount, sale_date,
       SUM(amount) OVER (
           PARTITION BY region
           ORDER BY sale_date
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS rows_sum
FROM sales
WHERE region = 'Moscow'
ORDER BY sale_date, id;
```

```
 id | amount | sale_date  | rows_sum
----+--------+------------+----------
  1 | 100    | 2024-01-05 | 100
  2 | 150    | 2024-01-10 | 250      ← 100 + 150
 10 |  50    | 2024-01-10 | 300      ← 100 + 150 + 50
  3 | 200    | 2024-01-15 | 500
```

**RANGE — одинаковые даты суммируются вместе:**

```sql
SELECT id, amount, sale_date,
       SUM(amount) OVER (
           PARTITION BY region
           ORDER BY sale_date
           RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS range_sum
FROM sales
WHERE region = 'Moscow'
ORDER BY sale_date, id;
```

```
 id | amount | sale_date  | range_sum
----+--------+------------+-----------
  1 | 100    | 2024-01-05 | 100
  2 | 150    | 2024-01-10 | 300      ← 100 + 150 + 50 (обе строки даты!)
 10 |  50    | 2024-01-10 | 300      ← 100 + 150 + 50 (обе строки даты!)
  3 | 200    | 2024-01-15 | 500
```

**Ключевое отличие:** `RANGE` включает **все строки с одинаковым значением** ORDER BY. Строки id=2 и id=10 (обе `2024-01-10`) видят **одинаковое окно**.

### GROUPS: по группам одинаковых значений

```sql
SELECT id, amount, sale_date,
       SUM(amount) OVER (
           PARTITION BY region
           ORDER BY sale_date
           GROUPS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS groups_sum
FROM sales
WHERE region = 'Moscow'
ORDER BY sale_date, id;
```

**Результат аналогичен RANGE:** группы одинаковых значений обрабатываются вместе.

### Как FRAME чинит LAST_VALUE

Вернёмся к проблеме из 13.2. Чтобы `LAST_VALUE` вернул **последнее значение всей группы**, нужно указать FRAME:

```sql
SELECT id, region, amount, sale_date,
       LAST_VALUE(amount) OVER (
           PARTITION BY region
           ORDER BY sale_date
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS last_in_group
FROM sales
WHERE region = 'Moscow'
ORDER BY sale_date;
```

**Для Moscow:**

```
 id | amount | sale_date  | last_in_group
----+--------+------------+---------------
  1 | 100    | 2024-01-05 | 200
  2 | 150    | 2024-01-10 | 200
  3 | 200    | 2024-01-15 | 200
```

**Теперь `LAST_VALUE` = 200 для всех строк** — окно расширено до конца группы.

### Сводная таблица FRAME

| FRAME | Что означает | Пример использования |
|:---|:---|:---|
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | От начала до текущей (по физическим строкам) | Накопительная сумма |
| `ROWS BETWEEN 1 PRECEDING AND CURRENT ROW` | Текущая + 1 предыдущая | Скользящее среднее |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` | Вся группа | `LAST_VALUE` для всей группы |
| `RANGE ...` | По значениям ORDER BY, одинаковые вместе | Суммы по одинаковым датам |
| `GROUPS ...` | По группам одинаковых значений | Аналогично RANGE |

**Дефолт (когда ORDER BY есть, а FRAME нет):**

```sql
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

---

## 13.4 Агрегаты как оконные: SUM OVER, COUNT OVER, AVG OVER, MIN/MAX OVER

### Как агрегат становится оконным

Обычный агрегат:

```sql
-- GROUP BY: схлопывает в одну строку на группу
SELECT region, SUM(amount) AS total_sum
FROM sales
GROUP BY region;
```

```
 region | total_sum
--------+-----------
 Kazan  | 290
 Moscow | 500
 SPb    | 390
```

**Тот же агрегат с OVER:**

```sql
-- OVER: значение для каждой строки
SELECT id, region, amount,
       SUM(amount) OVER (PARTITION BY region) AS region_sum
FROM sales
ORDER BY region, id;
```

```
 id | region | amount | region_sum
----+--------+--------+------------
  7 | Kazan  | 130    | 290
  8 | Kazan  | 160    | 290
  1 | Moscow | 100    | 500
  2 | Moscow | 150    | 500
  3 | Moscow | 200    | 500
 10 | Moscow |  50    | 500
  4 | SPb    | 120    | 390
  5 | SPb    | 180    | 390
  6 | SPb    |  90    | 390
  9 | SPb    | 120    | 390
```

**Каждая строка «знает» сумму своего региона, оставаясь в деталях.**

### SUM OVER: накопительная сумма и сумма по группе

**Без ORDER BY — сумма по всей партиции:**

```sql
SELECT id, region, amount,
       SUM(amount) OVER (PARTITION BY region) AS region_sum
FROM sales;
```

**С ORDER BY — накопительная сумма (дефолтный FRAME):**

```sql
SELECT id, region, amount, sale_date,
       SUM(amount) OVER (
           PARTITION BY region
           ORDER BY sale_date
       ) AS running_sum
FROM sales
ORDER BY region, sale_date;
```

```
Для Moscow:
 id | amount | sale_date  | running_sum
----+--------+------------+-------------
  1 | 100    | 2024-01-05 | 100
  2 | 150    | 2024-01-10 | 250
  3 | 200    | 2024-01-15 | 450
```

**Правило:** если есть `ORDER BY` внутри `OVER` — дефолтный FRAME = `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, и SUM становится **накопительной**. Без `ORDER BY` — FRAME = вся партиция, SUM = **сумма по группе**.

### COUNT OVER: количество в группе и накопительный счётчик

**Без ORDER BY — сколько строк в партиции:**

```sql
SELECT id, region, salesperson,
       COUNT(*) OVER (PARTITION BY region) AS rows_in_region
FROM sales;
```

**С ORDER BY — сколько строк до текущей включительно:**

```sql
SELECT id, region, sale_date,
       COUNT(*) OVER (
           PARTITION BY region
           ORDER BY sale_date
       ) AS running_count
FROM sales
ORDER BY region, sale_date;
```

```
Для Moscow:
 id | sale_date  | running_count
----+------------+---------------
  1 | 2024-01-05 | 1
  2 | 2024-01-10 | 2
  3 | 2024-01-15 | 3
```

### AVG OVER: среднее по группе и скользящее среднее

**Без ORDER BY — среднее по партиции:**

```sql
SELECT id, region, amount,
       AVG(amount) OVER (PARTITION BY region) AS region_avg
FROM sales;
```

**С ORDER BY + ROWS — скользящее среднее:**

```sql
SELECT id, region, amount, sale_date,
       AVG(amount) OVER (
           PARTITION BY region
           ORDER BY sale_date
           ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
       ) AS moving_avg_2
FROM sales
ORDER BY region, sale_date;
```

```
Для Moscow:
 id | amount | moving_avg_2
----+--------+-------------
  1 | 100    | 100          ← (100) / 1
  2 | 150    | 125          ← (100 + 150) / 2
  3 | 200    | 175          ← (150 + 200) / 2
```

### MIN/MAX OVER: экстремумы по группе и накопительные

**Без ORDER BY — минимум/максимум по партиции:**

```sql
SELECT id, region, amount,
       MIN(amount) OVER (PARTITION BY region) AS region_min,
       MAX(amount) OVER (PARTITION BY region) AS region_max
FROM sales;
```

**С ORDER BY — накопительный минимум/максимум:**

```sql
SELECT id, region, amount, sale_date,
       MAX(amount) OVER (
           PARTITION BY region
           ORDER BY sale_date
       ) AS running_max
FROM sales
ORDER BY region, sale_date;
```

```
Для Moscow:
 id | amount | sale_date  | running_max
----+--------+------------+-------------
  1 | 100    | 2024-01-05 | 100         ← максимум из {100}
  2 | 150    | 2024-01-10 | 150         ← максимум из {100, 150}
  3 | 200    | 2024-01-15 | 200         ← максимум из {100, 150, 200}
```

### Сравнение: агрегат с ORDER BY vs без ORDER BY

| Агрегат | Без ORDER BY | С ORDER BY (дефолтный FRAME) |
|:---|:---|:---|
| `SUM` | Сумма по всей партиции | Накопительная сумма |
| `COUNT` | Количество в партиции | Накопительный счётчик |
| `AVG` | Среднее по партиции | Накопительное среднее |
| `MIN` | Минимум по партиции | Накопительный минимум |
| `MAX` | Максимум по партиции | Накопительный максимум |

### Практический пример: доля продажи от региона

```sql
SELECT id, region, salesperson, amount,
       amount / SUM(amount) OVER (PARTITION BY region) * 100 AS percent_of_region
FROM sales
ORDER BY region, amount DESC;
```

```
Для Moscow:
 id | amount | percent_of_region
----+--------+-------------------
  3 | 200    | 44.44
  2 | 150    | 33.33
  1 | 100    | 22.22
```

**Каждая строка показывает, какой процент от продаж региона она составляет.**

### Итог подглавы 13.4

- **Любой агрегат** (`SUM`, `COUNT`, `AVG`, `MIN`, `MAX`) работает как оконный с `OVER`.
- **Без ORDER BY** — агрегат по всей партиции, значение повторяется для каждой строки.
- **С ORDER BY** — дефолтный FRAME = `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` → **накопительный** агрегат.
- **Явный FRAME** (`ROWS BETWEEN 1 PRECEDING AND CURRENT ROW`) — скользящие окна.
- Это позволяет считать **доли, накопительные суммы, скользящие средние** без подзапросов.

---

## 13.5 WINDOW: именованные окна

### Проблема: дублирование OVER

```sql
-- ❌ Одинаковое окно повторяется 3 раза:
SELECT id, region, amount, sale_date,
       ROW_NUMBER() OVER (PARTITION BY region ORDER BY sale_date) AS rn,
       SUM(amount) OVER (PARTITION BY region ORDER BY sale_date) AS running_sum,
       LAG(amount) OVER (PARTITION BY region ORDER BY sale_date) AS prev_amount
FROM sales;
-- Если окно изменится — нужно менять в 3 местах.
```

### Решение: WINDOW

```sql
-- ✅ Именованное окно:
SELECT id, region, amount, sale_date,
       ROW_NUMBER() OVER w AS rn,
       SUM(amount) OVER w AS running_sum,
       LAG(amount) OVER w AS prev_amount
FROM sales
WINDOW w AS (PARTITION BY region ORDER BY sale_date)
ORDER BY region, sale_date;
```

**Синтаксис:**

```sql
WINDOW <имя> AS (<определение окна>)
```

Использование: `OVER w` — ссылка на именованное окно.

### Несколько именованных окон

```sql
SELECT id, region, amount, sale_date,
       SUM(amount) OVER by_region AS region_sum,
       ROW_NUMBER() OVER by_region_date AS rn
FROM sales
WINDOW
    by_region AS (PARTITION BY region),
    by_region_date AS (PARTITION BY region ORDER BY sale_date)
ORDER BY region, sale_date;
```

### Наследование окон

Одно именованное окно может **расширять** другое:

```sql
SELECT id, region, amount, sale_date,
       SUM(amount) OVER by_region AS region_sum,
       SUM(amount) OVER by_region_date AS running_sum
FROM sales
WINDOW
    by_region AS (PARTITION BY region),
    by_region_date AS (by_region ORDER BY sale_date)
    --                    ↑ наследует PARTITION BY region
ORDER BY region, sale_date;
```

**`by_region_date` наследует `PARTITION BY region` из `by_region` и добавляет `ORDER BY sale_date`.**

### Добавление FRAME к именованному окну

```sql
SELECT id, region, amount, sale_date,
       SUM(amount) OVER by_region_date AS running_sum,
       SUM(amount) OVER (by_region_date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS total_sum
FROM sales
WINDOW by_region_date AS (PARTITION BY region ORDER BY sale_date)
ORDER BY region, sale_date;
```

**Синтаксис:** `OVER (имя_окна FRAME)` — берём именованное окно и добавляем/переопределяем FRAME.

### Когда использовать

- **3+ функций с одинаковым OVER** — выноси в WINDOW.
- **Читаемость** — `OVER w` короче, чем `OVER (PARTITION BY region ORDER BY sale_date)`.
- **Изменение окна в одном месте** — не нужно менять все OVER.

### Итог подглавы 13.5

- **WINDOW** — именованное окно, объявляется в конце запроса.
- **Использование** — `OVER w` вместо полного определения.
- **Наследование** — `WINDOW w2 AS (w1 ORDER BY ...)`.
- **FRAME поверх** — `OVER (w ROWS BETWEEN ...)`.
- Применяй, когда 3+ функций используют одинаковый OVER.

---

## 13.6 Производительность оконных функций: work_mem, сортировка, сравнение с GROUP BY

### Как PostgreSQL выполняет оконные функции

**Что под капотом:**

```
SELECT id, region, amount,
       ROW_NUMBER() OVER (PARTITION BY region ORDER BY sale_date) AS rn
FROM sales;

План (EXPLAIN, Глава 7):
  WindowAgg
    → Sort (sort key: region, sale_date)
        → Seq Scan on sales
```

**Пошагово:**

1. **Seq Scan** — читает таблицу (Глава 9).
2. **Sort** — сортирует строки по `region, sale_date` (ключ сортировки = PARTITION BY + ORDER BY).
3. **WindowAgg** — идёт по отсортированным строкам, вычисляет оконную функцию для каждой партиции.

### Сортировка — самая дорогая часть

**Если `work_mem` не хватает:**

```
Sort
  Sort Key: region, sale_date
  Sort Method: external merge  ← сброс на диск!
  temp written: 50000 страниц
```

Вспомни главу 9: сортировка использует `work_mem`. Если данные не помещаются — PostgreSQL сбрасывает на диск, и запрос **замедляется в десятки раз**.

**Решение:**

```sql
-- Для конкретной транзакции:
SET work_mem = '256MB';

-- Или глобально:
ALTER SYSTEM SET work_mem = '64MB';
SELECT pg_reload_conf();
```

### Индекс помогает избежать сортировки

Если индекс уже **отсортирован по нужному ключу**, PostgreSQL пропускает Sort:

```sql
CREATE INDEX idx_sales_region_date ON sales(region, sale_date);

-- Теперь план:
  WindowAgg
    → Index Scan using idx_sales_region_date on sales
    -- Без Sort! Индекс уже даёт нужный порядок.
```

**Правило:** индекс по `(PARTITION BY колонки, ORDER BY колонки)` ускоряет оконные функции.

### Сравнение: OVER vs GROUP BY

**Задача:** сумма по регионам.

**GROUP BY:**

```sql
SELECT region, SUM(amount)
FROM sales
GROUP BY region;
```

```
План:
  HashAggregate
    → Seq Scan on sales

Время: ~0.1 мс
Память: work_mem на хэш-таблицу
```

**OVER:**

```sql
SELECT DISTINCT region,
       SUM(amount) OVER (PARTITION BY region) AS region_sum
FROM sales;
```

```
План:
  WindowAgg
    → Sort (region)
        → Seq Scan on sales

Время: ~0.3 мс
Память: work_mem на сортировку
```

**Когда GROUP BY быстрее:**

- Нужен **только агрегат** без деталей строк.
- Групп немного — HashAggregate дешевле сортировки.

**Когда OVER быстрее:**

- Нужны **агрегаты + детали** в одном запросе (иначе — JOIN с GROUP BY).
- Нужны **несколько разных агрегатов** по разным окнам.

### Сравнение: OVER vs подзапросы

**Задача:** последний заказ каждого клиента.

**Подзапросы (N+1):**

```sql
SELECT u.id,
    (SELECT o.id FROM orders o WHERE o.user_id = u.id ORDER BY created_at DESC LIMIT 1)
FROM users u;
-- Для каждого пользователя — отдельный подзапрос. O(N) подзапросов.
```

**Оконная функция:**

```sql
SELECT user_id, order_id
FROM (
    SELECT user_id, id AS order_id,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
) sub
WHERE rn = 1;
-- Один проход по таблице. O(N log N) на сортировку.
```

**На больших таблицах оконная функция в разы быстрее подзапросов.**

### 💡 Практика

**✅ ОБЯЗАТЕЛЬНО:**

1. **Создавай индекс по `(PARTITION BY, ORDER BY)`** — ускоряет оконные функции, убирает Sort.

2. **Проверяй `EXPLAIN ANALYZE`** — ищи `external merge` и `temp written`.

3. **Фильтруй (WHERE) до оконной функции** — уменьшай окно.

**👍 СТОИТ:**

4. **Используй WINDOW** — читаемость и одинаковый план.

5. **`work_mem`** — если сортировка сбрасывается на диск, увеличь.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй OVER для простого GROUP BY** — GROUP BY быстрее.

7. **Не пиши подзапросы (N+1)** там, где работает ROW_NUMBER.

### Итог подглавы 13.6

- Оконные функции = **сортировка + вычисление**. Сортировка — главная цена.
- **Индекс по `(PARTITION BY, ORDER BY)`** убирает сортировку.
- **work_mem** — критичен при больших окнах.
- **GROUP BY быстрее** для простых агрегатов; **OVER быстрее** для агрегатов с деталями.
- **Фильтруй до окна, не после.**
- **Проверяй EXPLAIN ANALYZE** на `external merge` и `temp written`.

---

## 13.7 GROUPING SETS, ROLLUP, CUBE: продвинутая агрегация

### Проблема: несколько GROUP BY

**Бизнес хочет:**

```
Регион | Продавец | Сумма
-------+----------+-------
Moscow | NULL     | 500    ← итог по региону
SPb    | NULL     | 390
Kazan  | NULL     | 290
NULL   | Alice    | 480    ← итог по продавцу
NULL   | Bob      | 280
NULL   | Charlie  | 370
Moscow | Alice    | 300    ← детали по паре
Moscow | Bob      | 150
...
NULL   | NULL     | 1430   ← общий итог
```

**Наивно — UNION ALL:**

```sql
SELECT region, NULL AS salesperson, SUM(amount)
FROM sales GROUP BY region
UNION ALL
SELECT NULL, salesperson, SUM(amount)
FROM sales GROUP BY salesperson
UNION ALL
SELECT region, salesperson, SUM(amount)
FROM sales GROUP BY region, salesperson
UNION ALL
SELECT NULL, NULL, SUM(amount)
FROM sales;
-- 4 прохода по таблице! Каждый GROUP BY читает sales заново.
```

### Решение: GROUPING SETS

```sql
SELECT region, salesperson, SUM(amount) AS total
FROM sales
GROUP BY GROUPING SETS (
    (region),
    (salesperson),
    (region, salesperson),
    ()
)
ORDER BY region NULLS LAST, salesperson NULLS LAST;
```

**Что происходит:** один проход по таблице, PostgreSQL считает все агрегации одновременно.

**NULL в колонке** означает «эта колонка не участвует в агрегации».

### ROLLUP: иерархические итоги

```sql
SELECT region, salesperson, SUM(amount) AS total
FROM sales
GROUP BY ROLLUP (region, salesperson)
ORDER BY region NULLS LAST, salesperson NULLS LAST;
```

**Эквивалент:**

```sql
GROUP BY GROUPING SETS (
    (region, salesperson),  -- детали
    (region),               -- итог по региону
    ()                      -- общий итог
)
```

**Когда:** отчёт «детали → итоги по группам → общий итог».

### CUBE: все комбинации

```sql
SELECT region, salesperson, SUM(amount) AS total
FROM sales
GROUP BY CUBE (region, salesperson)
ORDER BY region NULLS LAST, salesperson NULLS LAST;
```

**Эквивалент:**

```sql
GROUP BY GROUPING SETS (
    (region, salesperson),  -- по паре
    (region),               -- по региону
    (salesperson),          -- по продавцу
    ()                      -- общий итог
)
```

### GROUPING(): отличить NULL-агрегацию от реального NULL

```sql
SELECT region, salesperson, SUM(amount) AS total,
       GROUPING(region) AS grp_region,
       GROUPING(salesperson) AS grp_salesperson
FROM sales
GROUP BY CUBE (region, salesperson);
```

**`GROUPING(колонка)`** возвращает:

- `1` — колонка **не участвует** в агрегации.
- `0` — колонка **участвует**.

### Сравнение

| Инструмент | Какие группы | Когда |
|:---|:---|:---|
| **GROUPING SETS** | Явный список групп | Нужны конкретные разрезы |
| **ROLLUP** | Иерархия слева направо + общий итог | Детали → итоги → общий |
| **CUBE** | Все комбинации + общий итог | Нужны все разрезы |

### Производительность

**GROUPING SETS / ROLLUP / CUBE** вычисляют все агрегации **за один проход** (в отличие от UNION ALL — за N проходов).

### Итог подглавы 13.7

- **GROUPING SETS** — несколько агрегаций в одном запросе.
- **ROLLUP** — иерархия: детали → итоги → общий.
- **CUBE** — все комбинации.
- **GROUPING()** — отличает NULL-агрегацию от реального NULL.
- **Один проход** вместо UNION ALL с N проходами.

---

## 13.8 Рекурсивные CTE: обход деревьев и графов

### Что такое рекурсивный CTE

> **Рекурсивный CTE (WITH RECURSIVE)** — конструкция SQL, которая позволяет запросу ссылаться на **самого себя**, постепенно наращивая результат.

### Синтаксис

```sql
WITH RECURSIVE <имя> AS (
    -- Базовая часть:
    SELECT ...
    FROM ...
    WHERE ...

    UNION ALL

    -- Рекурсивная часть (ссылается на <имя>):
    SELECT ...
    FROM ...
    JOIN <имя> ON ...
)
SELECT * FROM <имя>;
```

### Тестовые данные: дерево сотрудников

```sql
CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    manager_id BIGINT REFERENCES employees(id)
);

INSERT INTO employees (id, name, manager_id) VALUES
    (1, 'Alice', NULL),
    (2, 'Bob', 1),
    (3, 'Charlie', 1),
    (4, 'Diana', 2),
    (5, 'Eve', 2),
    (6, 'Frank', 4);
```

**Дерево:**

```
Alice (1)
├── Bob (2)
│   ├── Diana (4)
│   │   └── Frank (6)
│   └── Eve (5)
└── Charlie (3)
```

### Задача: все подчинённые Alice

```sql
WITH RECURSIVE subordinates AS (
    SELECT id, name, manager_id, 1 AS depth
    FROM employees
    WHERE id = 1

    UNION ALL

    SELECT e.id, e.name, e.manager_id, s.depth + 1
    FROM employees e
    JOIN subordinates s ON s.id = e.manager_id
)
SELECT * FROM subordinates;
```

**Как выполняется:**

```
Итерация 0: id=1 (Alice)
Итерация 1: id=2 (Bob), id=3 (Charlie)
Итерация 2: id=4 (Diana), id=5 (Eve)
Итерация 3: id=6 (Frank)
Итерация 4: 0 строк → конец
```

**Результат:**

```
 id | name    | manager_id | depth
----+---------+------------+-------
  1 | Alice   | NULL       | 1
  2 | Bob     | 1          | 2
  3 | Charlie | 1          | 2
  4 | Diana   | 2          | 3
  5 | Eve     | 2          | 3
  6 | Frank   | 4          | 4
```

### Задача: путь от Frank до Alice (предки)

```sql
WITH RECURSIVE ancestors AS (
    SELECT id, name, manager_id, 1 AS depth
    FROM employees
    WHERE id = 6

    UNION ALL

    SELECT e.id, e.name, e.manager_id, a.depth + 1
    FROM employees e
    JOIN ancestors a ON e.id = a.manager_id
)
SELECT * FROM ancestors;
```

**Результат:** Frank → Diana → Bob → Alice.

### Защита от бесконечного цикла

```sql
WITH RECURSIVE subordinates AS (
    SELECT id, name, manager_id, 1 AS depth
    FROM employees WHERE id = 1

    UNION ALL

    SELECT e.id, e.name, e.manager_id, s.depth + 1
    FROM employees e
    JOIN subordinates s ON s.id = e.manager_id
    WHERE s.depth < 10  -- ✅ максимум 10 уровней
)
SELECT * FROM subordinates;
```

### Производительность

- Каждая итерация — отдельный доступ к таблице.
- Глубина 10 → 10 обращений.
- **Индекс** по `manager_id` ускоряет рекурсивную часть.

```sql
CREATE INDEX idx_employees_manager_id ON employees(manager_id);
```

### Итог подглавы 13.8

- **WITH RECURSIVE** — цикл в SQL: базовая часть + рекурсивная с UNION ALL.
- **Итерации** — пока рекурсивная часть не вернёт 0 строк.
- **Деревья:** потомки, предки, глубина.
- **Защита от циклов:** `WHERE depth < N`.
- **Индекс** по FK ускоряет.
- **Цена:** O(глубина × N) — для глубоких деревьев используй Materialized Path.

---

## 13.9 LATERAL JOIN: коррелированные подзапросы в FROM

### Проблема: подзапрос не видит внешние колонки

```sql
-- ❌ Ошибка: подзапрос в FROM не видит u.id
SELECT u.name, lo.total
FROM users u
JOIN (
    SELECT user_id, total FROM orders
    WHERE user_id = u.id  -- ❌ не виден!
    ORDER BY created_at DESC
    LIMIT 3
) lo ON true;
```

### Решение: LATERAL

```sql
SELECT u.name, lo.total
FROM users u
CROSS JOIN LATERAL (
    SELECT o.total FROM orders o
    WHERE o.user_id = u.id  -- ✅ теперь виден!
    ORDER BY o.created_at DESC
    LIMIT 3
) lo;
```

**LATERAL** — для каждой строки из `users` выполняет подзапрос, который видит колонки `u`.

### LATERAL vs оконные функции

**Задача:** последние 3 заказа каждого пользователя.

**LATERAL:**

```sql
SELECT u.name, lo.total
FROM users u
CROSS JOIN LATERAL (
    SELECT o.total FROM orders o
    WHERE o.user_id = u.id
    ORDER BY o.created_at DESC
    LIMIT 3
) lo;
```

**Оконная функция:**

```sql
SELECT name, total
FROM (
    SELECT u.name, o.total,
           ROW_NUMBER() OVER (PARTITION BY u.id ORDER BY o.created_at DESC) AS rn
    FROM users u
    JOIN orders o ON o.user_id = u.id
) sub
WHERE rn <= 3;
```

**Когда LATERAL:** читаемее с LIMIT внутри подзапроса. **Когда оконная:** один проход на больших объёмах.

### Итог подглавы 13.9

- **LATERAL** — подзапрос в FROM с доступом к колонкам из предыдущих таблиц.
- **Цикл:** для каждой строки слева — подзапрос справа.
- **Когда:** LIMIT внутри, агрегаты по внешней колонке, UNNEST.
- **Индекс** на колонку JOIN в подзапросе обязателен.

---

## 13.10 FILTER, DISTINCT ON, WITH ORDINALITY, UNNEST

### FILTER: условная агрегация

```sql
SELECT region,
       SUM(amount) AS total_amount,
       SUM(amount) FILTER (WHERE amount > 150) AS big_amount
FROM sales
GROUP BY region;
```

### DISTINCT ON: первая строка в группе

```sql
SELECT DISTINCT ON (region)
    region, salesperson, amount, sale_date
FROM sales
ORDER BY region, sale_date DESC;
-- Для каждого региона — последняя продажа по дате.
```

**Правила:**

1. `ORDER BY` обязателен.
2. Колонки `DISTINCT ON` должны быть **первыми** в `ORDER BY`.

### WITH ORDINALITY: нумерация из UNNEST

```sql
SELECT *
FROM UNNEST(ARRAY['a', 'b', 'c']) WITH ORDINALITY AS t(letter, position);
```

```
 letter | position
--------+----------
 a      | 1
 b      | 2
 c      | 3
```

### UNNEST с LATERAL

```sql
SELECT u.name, t.tag, t.position
FROM users_with_tags u
CROSS JOIN LATERAL UNNEST(u.tags) WITH ORDINALITY AS t(tag, position);
```

### Итог подглавы 13.10

- **FILTER** — условная агрегация.
- **DISTINCT ON** — первая строка в группе.
- **WITH ORDINALITY** — нумерация строк из UNNEST.
- **UNNEST + LATERAL** — разворот массива с привязкой.

---

## 13.11 Практика Go: аналитические запросы

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "os"

    _ "github.com/jackc/pgx/v5/stdlib"
)

func main() {
    db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // 1. Накопительная сумма по регионам
    rows, _ := db.Query(`
        SELECT region, sale_date, amount,
               SUM(amount) OVER (PARTITION BY region ORDER BY sale_date) AS running_sum
        FROM sales
        ORDER BY region, sale_date
    `)
    defer rows.Close()

    fmt.Println("=== Накопительная сумма ===")
    for rows.Next() {
        var region string
        var saleDate string
        var amount, runningSum float64
        rows.Scan(&region, &saleDate, &amount, &runningSum)
        fmt.Printf("%s | %s | %.2f | %.2f\n", region, saleDate, amount, runningSum)
    }

    // 2. Топ-2 продавца в каждом регионе
    rows2, _ := db.Query(`
        SELECT region, salesperson, total
        FROM (
            SELECT region, salesperson, SUM(amount) AS total,
                   ROW_NUMBER() OVER (PARTITION BY region ORDER BY SUM(amount) DESC) AS rn
            FROM sales
            GROUP BY region, salesperson
        ) sub
        WHERE rn <= 2
        ORDER BY region, total DESC
    `)
    defer rows2.Close()

    fmt.Println("\n=== Топ-2 продавца по регионам ===")
    for rows2.Next() {
        var region, salesperson string
        var total float64
        rows2.Scan(&region, &salesperson, &total)
        fmt.Printf("%s | %s | %.2f\n", region, salesperson, total)
    }

    // 3. Рекурсивный CTE: дерево сотрудников
    rows3, _ := db.Query(`
        WITH RECURSIVE subordinates AS (
            SELECT id, name, manager_id, 1 AS depth
            FROM employees WHERE id = 1
            UNION ALL
            SELECT e.id, e.name, e.manager_id, s.depth + 1
            FROM employees e
            JOIN subordinates s ON s.id = e.manager_id
        )
        SELECT * FROM subordinates
    `)
    defer rows3.Close()

    fmt.Println("\n=== Дерево сотрудников ===")
    for rows3.Next() {
        var id int64
        var name string
        var managerID sql.NullInt64
        var depth int
        rows3.Scan(&id, &name, &managerID, &depth)
        fmt.Printf("id=%d, name=%s, depth=%d\n", id, name, depth)
    }
}
```

---

## 13.12 Выводы и типичные ошибки

**Что мы узнали?**

Оконные функции выполняют вычисления над набором строк (окном), не схлопывая их. PARTITION BY делит строки на группы, ORDER BY сортирует внутри группы, FRAME определяет подмножество строк для каждой строки. Полный набор функций: нумерация (ROW_NUMBER, RANK, DENSE_RANK, NTILE), значения из окна (FIRST_VALUE, LAST_VALUE, NTH_VALUE), сравнение с соседями (LAG, LEAD), агрегаты (SUM, COUNT, AVG, MIN, MAX). GROUPING SETS, ROLLUP, CUBE — множественные агрегации в одном проходе. Рекурсивные CTE обходят деревья и графы. LATERAL даёт подзапросам доступ к внешним колонкам. FILTER, DISTINCT ON, WITH ORDINALITY, UNNEST — компактные инструменты.

**Типичные ошибки:**

- ❌ **Использовать GROUP BY там, где нужны детали + агрегаты** — получишь только агрегаты, без деталей.
- ❌ **Забывать про дефолтный FRAME** — `LAST_VALUE` возвращает значение до текущей строки, а не конец группы.
- ❌ **Использовать UNION ALL для множественных агрегаций** — GROUPING SETS в разы быстрее.
- ❌ **Писать подзапросы (N+1) там, где работает ROW_NUMBER** — на больших таблицах разница в разы.
- ❌ **Не создавать индекс по `(PARTITION BY, ORDER BY)`** — сортировка сбрасывается на диск.
- ❌ **Использовать ROW_NUMBER в WHERE напрямую** — нужен подзапрос.
- ❌ **Рекурсивный CTE без защиты от циклов** — бесконечный цикл при циклических данных.
- ❌ **Использовать LATERAL без индекса на колонку JOIN в подзапросе** — N+1 без ускорения.
- ❌ **Не проверять `EXPLAIN ANALYZE`** — `external merge` и `temp written` — признаки нехватки work_mem.

---

## 13.13 Для быстрого повторения

- **Оконная функция** — `FUNC() OVER (PARTITION BY ... ORDER BY ... FRAME)`.
- **PARTITION BY** — делит на группы, не схлопывая.
- **Дефолтный FRAME** — `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.
- **ROW_NUMBER** — уникальные номера; **RANK** — с пропусками; **DENSE_RANK** — без пропусков.
- **LAG/LEAD** — предыдущее/следующее значение; **FIRST_VALUE/LAST_VALUE** — первое/последнее.
- **SUM OVER с ORDER BY** — накопительная сумма; **без ORDER BY** — сумма по группе.
- **WINDOW** — именованные окна: `WINDOW w AS (PARTITION BY ...)`.
- **GROUPING SETS** — множественные агрегации за один проход; **ROLLUP** — иерархия; **CUBE** — все комбинации.
- **Рекурсивный CTE** — `WITH RECURSIVE ... UNION ALL`, защита от циклов через `depth`.
- **LATERAL** — подзапрос с доступом к внешним колонкам.
- **Индекс по `(PARTITION BY, ORDER BY)`** убирает Sort, **work_mem** влияет на сброс на диск.

---

## 13.14 Вопросы для самопроверки

1. Чем оконная функция отличается от GROUP BY?
2. Что делает PARTITION BY? Как это работает пошагово?
3. Чем RANK отличается от DENSE_RANK? Приведи пример.
4. Почему LAST_VALUE для первой строки возвращает её же значение, а не последнее в группе?
5. Чем ROWS отличается от RANGE в FRAME?
6. Как получить накопительную сумму по группе?
7. Что такое WINDOW и зачем?
8. Когда GROUPING SETS быстрее UNION ALL?
9. Как работает рекурсивный CTE? Как защититься от бесконечного цикла?
10. Что такое LATERAL и когда он полезен?
11. Чем DISTINCT ON отличается от ROW_NUMBER?
12. Что такое FILTER и зачем?

---

## 13.15 Ответы

### Ответ 1

Оконная функция добавляет вычисление **к каждой строке**, не схлопывая. GROUP BY сжимает группу в одну строку.

### Ответ 2

PARTITION BY делит строки на группы по значению колонок. Оконная функция выполняется независимо в каждой группе, не видя строки из других групп.

### Ответ 3

RANK даёт одинаковый ранг одинаковым значениям, следующий — с пропуском (1, 2, 2, 4). DENSE_RANK — без пропуска (1, 2, 2, 3).

### Ответ 4

Потому что дефолтный FRAME = `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Для первой строки окно = только она сама. Чтобы получить последнее значение всей группы — `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

### Ответ 5

ROWS определяет строки по физическим позициям. RANGE — по значениям ORDER BY: строки с одинаковыми значениями включаются вместе.

### Ответ 6

```sql
SUM(amount) OVER (PARTITION BY region ORDER BY sale_date)
```

С ORDER BY — дефолтный FRAME даёт накопительную сумму.

### Ответ 7

WINDOW — именованное окно, объявляется в конце запроса. Позволяет переиспользовать OVER, наследовать, добавлять FRAME.

### Ответ 8

GROUPING SETS считает все агрегации за один проход по таблице. UNION ALL — за N проходов (N = число агрегаций).

### Ответ 9

Рекурсивный CTE — базовая часть + рекурсивная с UNION ALL. Итерации, пока рекурсивная часть не вернёт 0 строк. Защита от циклов — `WHERE depth < N`.

### Ответ 10

LATERAL — подзапрос в FROM с доступом к колонкам из предыдущих таблиц. Полезен для LIMIT внутри подзапроса, агрегатов по внешней колонке, UNNEST.

### Ответ 11

DISTINCT ON — PostgreSQL-специфичный, короче, требует ORDER BY. ROW_NUMBER — стандартный, через подзапрос, гибче.

### Ответ 12

FILTER — условная агрегация: `SUM(x) FILTER (WHERE cond)` вместо `SUM(CASE WHEN cond THEN x END)`.

---

## 13.16 Куда идти дальше?

Мы разобрали продвинутый SQL: оконные функции, рекурсию, LATERAL, GROUPING SETS. Но всё это — **чтение данных**. А теперь бизнес говорит: «У нас 1000 пользователей одновременно делают заказы. Как PostgreSQL справляется с конкуренцией?»

- Что происходит, когда два UPDATE пытаются обновить одну строку?
- Какие бывают блокировки: строки, таблицы, advisory?
- Что такое deadlock и как его избежать?
- Как `FOR UPDATE` и `SKIP LOCKED` решают задачи конкурентности?

**Глава 14: Блокировки и конкурентность — FOR UPDATE, SKIP LOCKED, deadlock.**