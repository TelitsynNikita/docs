# 📝 Глава 8: SQL-мышление — от задачи к запросу

**Что вы узнаете:**
- Как строить SQL-запрос от задачи, а не от синтаксиса.
- Все виды JOIN'ов: INNER, LEFT, RIGHT, FULL, CROSS, LATERAL, SELF — когда какой нужен.
- Как использовать WITH (CTE), UNION, INTERSECT, EXCEPT.
- Как писать подзапросы: скалярные, табличные, коррелированные.
- Как работать с INSERT, UPDATE, DELETE, RETURNING и UPSERT.
- Что такое DDL, constraints и как модифицировать таблицы.
- Как работать с Views и Materialized Views.
- Как писать пагинацию: LIMIT/OFFSET, Keyset, Cursor и гибридные подходы.
- Как защититься от SQL-инъекций через параметризацию.
- Как выполнять массовые вставки эффективно.

**После прочтения вы сможете:**
- Выбрать правильный JOIN под задачу.
- Написать запрос, который вернёт именно то, что нужно.
- Понять, когда использовать CTE, а когда подзапрос.
- Модифицировать данные с RETURNING и UPSERT.
- Спроектировать базовые таблицы с правильными constraints.
- Написать пагинацию, которая не деградирует на больших объёмах.
- Понимать, что происходит под капотом для каждой SQL-команды.
- Писать безопасные параметризованные запросы в Go.
- Выполнять batch-вставки эффективно через pgx.

---

## Содержание

- [8.0 Пролог: как думать запросами](#80-пролог-как-думать-запросами)
- [8.1 SELECT: как выбрать данные](#81-select-как-выбрать-данные)
- [8.2 Модификация данных: INSERT, UPDATE, DELETE, RETURNING](#82-модификация-данных-insert-update-delete-returning)
- [8.3 DDL и constraints: как создавать таблицы](#83-ddl-и-constraints-как-создавать-таблицы)
- [8.4 JOIN'ы: когда какой использовать](#84-joinы-когда-какой-использовать)
- [8.5 Подзапросы: скалярные, табличные, коррелированные](#85-подзапросы-скалярные-табличные-коррелированные)
- [8.6 Объединение результатов: UNION, INTERSECT, EXCEPT](#86-объединение-результатов-union-intersect-except)
- [8.7 CTE и WITH: временные таблицы](#87-cte-и-with-временные-таблицы)
- [8.8 Views и Materialized Views](#88-views-и-materialized-views)
- [8.9 Пагинация: LIMIT/OFFSET, Keyset, Cursor](#89-пагинация-limitoffset-keyset-cursor)
- [8.10 Безопасность: параметризация запросов](#810-безопасность-параметризация-запросов)
- [8.11 Batch insert: массовые вставки эффективно](#811-batch-insert-массовые-вставки-эффективно)
- [8.12 Практика Go: работа с SQL в коде](#812-практика-go-работа-с-sql-в-коде)
- [8.13 Выводы и типичные ошибки](#813-выводы-и-типичные-ошибки)
- [8.14 Для быстрого повторения](#814-для-быстрого-повторения)
- [8.15 Вопросы для самопроверки](#815-вопросы-для-самопроверки)
- [8.16 Ответы](#816-ответы)
- [8.17 Куда идти дальше?](#817-куда-идти-дальше)

---

## 8.0 Пролог: как думать запросами

Ты хочешь написать запрос. Но с чего начать?

**Плохой подход:** «Какие SQL-команды я знаю? Напишу что-нибудь из них.»

**Хороший подход:** «Какие данные мне нужны? Из каких таблиц? Какое условие? Какая связь между таблицами?»

### Постановка задачи

**Задача:** «Показать список заказов с именем пользователя за последнюю неделю».

**Шаги мышления:**

1. **Какие данные нужны?** — id заказа, дата заказа, сумма, имя пользователя.
2. **В каких таблицах эти данные?** — Заказы: `orders`, Пользователи: `users`.
3. **Как связаны таблицы?** — `orders.user_id` → `users.id`.
4. **Какое условие?** — `orders.created_at > now() - interval '7 days'`.
5. **Какой JOIN?** — INNER JOIN — только заказы с существующим пользователем.

**Собрать запрос:**

```sql
SELECT 
    o.id,
    o.created_at,
    o.total,
    u.name AS user_name
FROM orders o
INNER JOIN users u ON u.id = o.user_id
WHERE o.created_at > now() - interval '7 days'
ORDER BY o.created_at DESC;
```

### Что происходит под капотом, когда этот запрос выполняется

Вспомни Главу 7: каждый SQL-запрос проходит **Parse → Rewrite → Plan → Execute**.

**Parse:** Backend-процесс (Глава 1) разбирает SQL-текст в дерево.

**Rewrite:** применяет правила и views.

**Plan:** планировщик (Глава 7) анализирует индексы, статистику, выбирает алгоритм JOIN (Глава 9).

**Execute:** Backend-процесс выполняет план — читает страницы через Buffer Manager (Глава 1), проверяет видимость через MVCC (Глава 5).

### Что происходит при написании запроса — сводка

| SQL-операция | Что под капотом | Ссылка |
|:---|:---|:---|
| **SELECT** | Чтение страниц, проверка видимости (MVCC) | Глава 1, 5 |
| **WHERE** | Фильтрация: индекс или Seq Scan | Глава 3, 7 |
| **JOIN** | Выбор алгоритма: Nested Loop, Hash Join, Merge Join | Глава 9 |
| **ORDER BY** | Сортировка: индекс или Sort в памяти | Глава 3, 7 |
| **INSERT** | Новые кортежи, WAL, грязные страницы | Глава 1, 2 |
| **UPDATE** | Новые версии строк, мёртвые кортежи | Глава 5, 6 |
| **DELETE** | Помечает строки через t_xmax, мёртвые кортежи | Глава 5, 6 |

### Четыре вопроса перед запросом

| Вопрос | Пример |
|:---|:---|
| **Что выбрать?** | id заказа, дату, сумму, имя |
| **Из какой таблицы?** | orders, users |
| **Как связаны?** | orders.user_id = users.id |
| **Что отфильтровать?** | created_at > 7 дней |

Ответил на все — запрос готов на 80%.

---

## 8.1 SELECT: как выбрать данные

### Порядок выполнения операторов в SELECT

**Важно:** порядок **написания** операторов в SQL не совпадает с порядком **выполнения**.

**Порядок написания:**

```sql
SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT ... OFFSET ...;
```

**Порядок выполнения:**

```
1. FROM      — читает данные из таблицы (Seq Scan или Index Scan)
2. WHERE     — фильтрует строки до группировки
3. GROUP BY  — группирует строки
4. HAVING    — фильтрует группы после группировки
5. SELECT    — вычисляет выражения, применяет агрегаты
6. ORDER BY  — сортирует результат
7. LIMIT     — ограничивает число строк
8. OFFSET    — пропускает строки
```

**Практический пример:**

```sql
SELECT                              -- 5: вычисляем агрегаты
    age,
    COUNT(*) AS user_count
FROM users                          -- 1: читаем таблицу
WHERE age IS NOT NULL               -- 2: фильтруем строки
GROUP BY age                        -- 3: группируем
HAVING COUNT(*) > 1                 -- 4: фильтруем группы
ORDER BY user_count DESC            -- 6: сортируем
LIMIT 5;                            -- 7: ограничиваем
```

---

### Начальные данные

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE,
    age INT,
    salary NUMERIC(10,2),
    created_at TIMESTAMPTZ DEFAULT now()
);

INSERT INTO users (name, email, age, salary) VALUES
    ('Alice', 'alice@mail.com', 30, 100000),
    ('Bob', 'bob@mail.com', 25, 80000),
    ('Charlie', 'charlie@mail.com', NULL, 60000),
    ('Diana', NULL, 28, 90000),
    ('Eve', 'eve@mail.com', 35, 120000);
```

---

### Часть 1: Базовый SELECT

**Определение:** SELECT — оператор, который **выбирает строки** из таблицы и возвращает их. Не изменяет данные.

**Что под капотом:** читает страницы через Buffer Manager (Глава 1), проверяет видимость через MVCC (Глава 5). Не пишет в WAL, не создаёт мёртвых кортежей.

**Задача: «Показать пользователей с алиасами»**

```sql
SELECT 
    id AS user_id,
    name AS user_name,
    email AS user_email
FROM users;
```

---

### Часть 2: Фильтрация (WHERE)

**Определение:** WHERE — оператор, который **фильтрует строки** по условию.

**Что под капотом:** с индексом — Index Scan (Глава 3), без — Seq Scan.

**Задачи:**

```sql
-- Старше 25 лет
SELECT id, name, age FROM users WHERE age > 25;

-- В списке
SELECT id, name, age FROM users WHERE age IN (25, 30, 35);

-- В диапазоне
SELECT id, name, age FROM users WHERE age BETWEEN 25 AND 30;

-- По шаблону
SELECT id, name FROM users WHERE name LIKE 'A%';  -- Index Scan
SELECT id, name FROM users WHERE name LIKE '%A%'; -- Seq Scan (нужен GIN)

-- NULL
SELECT id, name FROM users WHERE email IS NULL;   -- Seq Scan
```

---

### Часть 3: Условные выражения (CASE, COALESCE, NULLIF)

**Определение:** выражения, которые возвращают **разные значения в зависимости от условий**. Не фильтруют строки, а вычисляют значение для каждой строки.

**CASE WHEN:**

```sql
SELECT 
    name,
    age,
    CASE 
        WHEN age < 25 THEN 'молодой'
        WHEN age < 35 THEN 'средний'
        ELSE 'старший'
    END AS age_category
FROM users;
```

**COALESCE:**

```sql
SELECT name, COALESCE(email, 'не указан') AS email_display FROM users;
```

**NULLIF:**

```sql
SELECT name, NULLIF(salary, 0) AS salary_no_zero FROM users;
```

---

### Часть 4: Агрегатные функции

**Определение:** функции, которые **объединяют несколько строк в одну**, вычисляя одно значение из множества.

| Функция | Что делает | Индекс? |
|:---|:---|:---|
| `COUNT(*)` | Число строк | Seq Scan |
| `SUM(col)` | Сумма | Seq Scan |
| `AVG(col)` | Среднее | Seq Scan |
| `MIN(col)` | Минимум | **Индекс Scan** |
| `MAX(col)` | Максимум | **Индекс Scan** |

```sql
SELECT 
    COUNT(*) AS total_users,
    AVG(age) AS avg_age,
    SUM(salary) AS total_salary,
    MIN(age) AS min_age,
    MAX(age) AS max_age
FROM users;
```

---

### Часть 5: GROUP BY и HAVING

**Определение:** GROUP BY **группирует строки** по колонке. HAVING **фильтрует группы**.

**Реальная задача:**

> «Покажи топ-5 пользователей по сумме заказов за последний месяц»

```sql
SELECT 
    user_id,
    SUM(total) AS total_sum,
    COUNT(*) AS order_count
FROM orders
WHERE created_at > now() - interval '1 month'
GROUP BY user_id
ORDER BY total_sum DESC
LIMIT 5;
```

**Порядок выполнения:** FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT.

---

### Часть 6: Преобразование типов (CAST)

**Определение:** меняет **тип** значения.

```sql
SELECT age::TEXT FROM users;
SELECT created_at::DATE FROM users;
SELECT '42'::INT AS answer;
```

**Внимание:** `WHERE created_at::DATE = ...` не использует индекс (Глава 3). Нужен функциональный индекс.

---

### Часть 7: ANY / ALL / EXISTS

```sql
-- Любой из списка
WHERE age = ANY(ARRAY[25, 30, 35]);

-- Больше всех из списка
WHERE age > ALL(ARRAY[20, 25, 30]);

-- Есть заказы
WHERE EXISTS (SELECT 1 FROM orders WHERE user_id = users.id);

-- Нет заказов
WHERE NOT EXISTS (SELECT 1 FROM orders WHERE user_id = users.id);
```

---

## 8.2 Модификация данных: INSERT, UPDATE, DELETE, RETURNING

### INSERT

**Что под капотом:** создаёт новые кортежи (Глава 2), пишет WAL (Глава 1), обновляет индексы (Глава 3). Не создаёт мёртвых кортежей.

```sql
-- Один
INSERT INTO users (name, email) VALUES ('Diana', 'diana@mail.com') RETURNING id;

-- Несколько
INSERT INTO users (name, email) VALUES
    ('Eve', 'eve@mail.com'),
    ('Frank', 'frank@mail.com');

-- Из другой таблицы
INSERT INTO new_users (name, email)
SELECT name, email FROM users WHERE age > 25;

-- Массовая (самый быстрый)
COPY users (name, email) FROM '/tmp/users.csv' WITH (FORMAT csv);
```

---

### UPDATE

**Что под капотом:** создаёт **новую версию** строки (Глава 5), старая становится мёртвым кортежем (Глава 6).

```sql
-- Обновить одну строку
UPDATE users SET email = 'new@mail.com' WHERE id = 1 RETURNING id, email;

-- Обновить несколько
UPDATE users SET salary = salary * 1.1 WHERE age < 30;

-- Обновить на основе другой таблицы
UPDATE users u
SET salary = su.new_salary
FROM salary_updates su
WHERE u.id = su.user_id;
```

---

### DELETE

**Что под капотом:** помечает строку через `t_xmax` (Глава 5), не удаляет физически. Становится мёртвым кортежем (Глава 6).

```sql
-- Удалить одну строку
DELETE FROM users WHERE id = 5 RETURNING id, name;

-- Массовое удаление
DELETE FROM orders WHERE created_at < '2023-01-01';
-- После массового DELETE: VACUUM orders;  (Глава 6)

-- Удалить с JOIN
DELETE FROM orders o
USING users u
WHERE o.user_id = u.id AND u.balance < 0;
```

---

### UPSERT (ON CONFLICT)

**Определение:** идемпотентная вставка — если конфликт, то UPDATE; иначе INSERT.

```sql
INSERT INTO users (name, email, age) 
VALUES ('Alice', 'alice@mail.com', 31)
ON CONFLICT (email) 
DO UPDATE SET name = EXCLUDED.name, age = EXCLUDED.age
RETURNING id;

-- Или игнорировать
INSERT INTO users (name, email, age) 
VALUES ('Alice', 'alice@mail.com', 31)
ON CONFLICT (email) DO NOTHING;
```

---

## 8.3 DDL и constraints: как создавать таблицы

### CREATE TABLE

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE,
    age INT CHECK (age >= 0),
    created_at TIMESTAMPTZ DEFAULT now()
);
```

**Что под капотом:** создаёт запись в `pg_class` (Глава 2), файлы на диске. PRIMARY KEY и UNIQUE создают индексы (Глава 3).

---

### FOREIGN KEY — полные возможности

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id)
        ON DELETE CASCADE
        ON UPDATE CASCADE,
    total NUMERIC(10,2) CHECK (total >= 0)
);

-- Индекс для FK (REFERENCES не создаёт!)
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

| Правило | Что делает |
|:---|:---|
| `ON DELETE CASCADE` | Удаляет строки из дочерней таблицы |
| `ON DELETE SET NULL` | Устанавливает NULL |
| `ON DELETE RESTRICT` | Запрещает удаление |
| `DEFERRABLE` | Проверка в конце транзакции |

---

### ALTER TABLE

```sql
ALTER TABLE users ADD COLUMN phone TEXT;
ALTER TABLE users DROP COLUMN phone;
ALTER TABLE users ALTER COLUMN age TYPE BIGINT;
ALTER TABLE users RENAME TO customers;
ALTER TABLE users ADD CONSTRAINT age_check CHECK (age >= 0);
ALTER TABLE users DROP CONSTRAINT age_check;
```

---

### TRUNCATE

```sql
TRUNCATE TABLE orders;
```

**В отличие от DELETE:** не создаёт мёртвых кортежей (Глава 6), мгновенно.

---

## 8.4 JOIN'ы: когда какой использовать

### INNER JOIN

**Определение:** только совпадающие строки.

```sql
SELECT o.id, u.name
FROM orders o
INNER JOIN users u ON u.id = o.user_id;
```

---

### LEFT JOIN

**Определение:** все строки из левой таблицы + совпадения.

```sql
SELECT o.id, u.name
FROM orders o
LEFT JOIN users u ON u.id = o.user_id;
-- Заказы без пользователя: user_name = NULL
```

---

### RIGHT JOIN

**Определение:** все строки из правой таблицы + совпадения.

```sql
SELECT u.name, o.id
FROM orders o
RIGHT JOIN users u ON u.id = o.user_id;
-- Пользователи без заказов: order_id = NULL
```

---

### FULL OUTER JOIN

**Определение:** все строки из обеих таблиц.

```sql
SELECT u.name, o.id
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;
```

---

### CROSS JOIN

**Определение:** декартово произведение. Опасно на больших таблицах!

```sql
SELECT u.name, o.id
FROM users u
CROSS JOIN orders o
LIMIT 10;
```

---

### SELF JOIN

**Определение:** таблица сама с собой.

```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id;
```

---

### LATERAL

**Определение:** подзапрос с доступом к внешним колонкам.

```sql
SELECT u.name, o.id, o.total
FROM users u
CROSS JOIN LATERAL (
    SELECT id, total FROM orders WHERE user_id = u.id ORDER BY id DESC LIMIT 3
) o;
```

---

### Бизнес через год: новый отчёт

**Сценарий:**

```
Сегодня:
  Запросы: заказы пользователя → INNER JOIN orders + users.

Через год:
  Бизнес просит: «Покажи ВСЕХ пользователей и их последний заказ,
  даже если заказа нет».

  INNER JOIN не подходит — он исключает пользователей без заказов.

Решение: LEFT JOIN + LATERAL:

  SELECT u.name, o.id AS last_order_id, o.created_at
  FROM users u
  LEFT JOIN LATERAL (
      SELECT id, created_at 
      FROM orders 
      WHERE user_id = u.id 
      ORDER BY created_at DESC 
      LIMIT 1
  ) o ON true;
```

---

## 8.5 Подзапросы

### Скалярный подзапрос

```sql
SELECT name, salary
FROM users
WHERE salary > (SELECT AVG(salary) FROM users);
```

### Табличный подзапрос (в FROM)

```sql
SELECT u.name, stats.order_count
FROM users u
INNER JOIN (
    SELECT user_id, COUNT(*) AS order_count
    FROM orders GROUP BY user_id
) stats ON stats.user_id = u.id;
```

### Коррелированный подзапрос

```sql
SELECT u.name,
    (SELECT MAX(o.id) FROM orders o WHERE o.user_id = u.id) AS last_order_id
FROM users u;
```

### EXISTS

```sql
SELECT id, name FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

---

## 8.6 UNION, INTERSECT, EXCEPT

```sql
-- UNION — объединяет, удаляет дубликаты
SELECT id, name FROM active_users
UNION
SELECT id, name FROM archived_users;

-- UNION ALL — объединяет, сохраняет дубликаты (быстрее)
SELECT id, name FROM active_users
UNION ALL
SELECT id, name FROM archived_users;

-- INTERSECT — пересечение
SELECT id, name FROM active_users
INTERSECT
SELECT id, name FROM archived_users;

-- EXCEPT — разность
SELECT id, name FROM active_users
EXCEPT
SELECT id, name FROM archived_users;
```

---

## 8.7 CTE и WITH

### Обычный CTE

```sql
WITH user_stats AS (
    SELECT user_id, SUM(total) AS total_sum
    FROM orders GROUP BY user_id
)
SELECT * FROM user_stats WHERE total_sum > 15000;
```

### Материализация

```sql
WITH user_stats AS MATERIALIZED (
    SELECT user_id, SUM(total) AS total_sum
    FROM orders GROUP BY user_id
)
SELECT * FROM user_stats;
```

### Рекурсивный CTE

```sql
WITH RECURSIVE subordinates AS (
    SELECT id, name, manager_id FROM employees WHERE name = 'Alice'
    UNION ALL
    SELECT e.id, e.name, e.manager_id
    FROM employees e
    INNER JOIN subordinates s ON s.id = e.manager_id
)
SELECT * FROM subordinates;
```

---

## 8.8 Views и Materialized Views

### View

```sql
CREATE VIEW active_users AS
SELECT id, name FROM users WHERE status = 'active';

SELECT * FROM active_users;
```

**Что под капотом:** View = сохранённый SQL-запрос. PostgreSQL подставляет его в основной запрос.

---

### Materialized View

```sql
CREATE MATERIALIZED VIEW daily_sales AS
SELECT created_at::DATE AS sale_date, SUM(total) AS total_revenue
FROM orders GROUP BY created_at::DATE;

SELECT * FROM daily_sales;  -- читает сохранённые данные

REFRESH MATERIALIZED VIEW daily_sales;  -- обновить
```

**Что под капотом:** MV = сохранённый результат. Читается быстро, но устаревает.

---

## 8.9 Пагинация

### LIMIT/OFFSET

```sql
SELECT id, name FROM products ORDER BY id LIMIT 10 OFFSET 90;
```

**Проблема:** читает OFFSET строк зря. Деградирует на больших смещениях.

---

### Keyset

```sql
SELECT id, name FROM products WHERE id > 90 ORDER BY id LIMIT 10;
```

**Преимущество:** стабильная скорость. Не читает лишнего.

**Keyset по нескольким колонкам:**

```sql
WHERE (created_at, id) > ('2024-01-15', 100)
ORDER BY created_at, id
LIMIT 10;
```

---

### Cursor

```sql
BEGIN;
DECLARE products_cursor CURSOR FOR SELECT id, name FROM products ORDER BY id;
FETCH 10 FROM products_cursor;
FETCH 10 FROM products_cursor;
CLOSE products_cursor;
COMMIT;
```

---

### Гибрид

```sql
-- Первые страницы: OFFSET
LIMIT 10 OFFSET 40;

-- Дальние страницы: один раз OFFSET для ключа, потом Keyset
SELECT id FROM products ORDER BY id LIMIT 1 OFFSET 279;  -- id = 289
SELECT id, name FROM products WHERE id > 289 ORDER BY id LIMIT 10;
```

---

## 8.10 Безопасность: параметризация запросов

До сих пор мы писали SQL-запросы с **захардкоженными** значениями. В реальном приложении значения приходят от пользователя — и это **опасно**.

### Проблема: SQL-инъекция

**SQL-инъекция** — атака, при которой злоумышленник вставляет свой SQL-код в запрос.

```go
// ❌ ОПАСНО: конкатенация строк
func getUserByName(db *sql.DB, name string) (*User, error) {
    query := fmt.Sprintf("SELECT id, name, email FROM users WHERE name = '%s'", name)
    // Если name = "' OR '1'='1" →
    // SELECT id, name, email FROM users WHERE name = '' OR '1'='1'
    // Вернёт ВСЕХ пользователей!
    return db.QueryRow(query).Scan(...)
}

// Ещё хуже:
// name = "'; DROP TABLE users; --"
// → SELECT ... WHERE name = ''; DROP TABLE users; --'
// Таблица users УДАЛЕНА!
```

**Как это работает:**

```
Пользователь вводит в поле поиска:
  '; DROP TABLE users; --

Твой код подставляет в SQL:
  SELECT * FROM users WHERE name = ''; DROP TABLE users; --';

PostgreSQL видит ДВА запроса:
  1. SELECT * FROM users WHERE name = '';
  2. DROP TABLE users;

Второй запрос УДАЛЯЕТ таблицу!
```

### Решение: параметризованные запросы

**Параметризованный запрос** — SQL, в котором значения заменены на плейсхолдеры (`$1`, `$2`), а сами значения передаются **отдельно**.

```go
// ✅ БЕЗОПАСНО: параметризация
func getUserByName(db *sql.DB, name string) (*User, error) {
    query := "SELECT id, name, email FROM users WHERE name = $1"
    // Значение name передаётся ОТДЕЛЬНО от SQL-текста.
    // PostgreSQL обрабатывает его как ДАННЫЕ, а не как SQL-код.
    return db.QueryRow(query, name).Scan(...)
}
```

**Почему это безопасно:**

```
Пользователь вводит:
  '; DROP TABLE users; --

Параметризованный запрос:
  SELECT * FROM users WHERE name = $1;

PostgreSQL получает:
  Параметр $1 = "'; DROP TABLE users; --"

PostgreSQL ищет строку с name РАВНЫМ этой строке.
Он НЕ интерпретирует её как SQL-код.
Это просто ДАННЫЕ.
```

### Как это работает под капотом

Вспомни Главу 1: PostgreSQL Wire Protocol поддерживает **подготовленные запросы** (prepared statements).

```
Параметризованный запрос в Go:

  1. Go-драйвер (pgx) отправляет SQL с плейсхолдерами:
     Parse: "SELECT * FROM users WHERE name = $1"

  2. PostgreSQL разбирает SQL и создаёт план запроса.
     На этом этапе $1 — это «параметр», а не значение.

  3. Go-драйвер отправляет значение параметра ОТДЕЛЬНО:
     Bind: $1 = "'; DROP TABLE users; --"

  4. PostgreSQL подставляет значение как ДАННЫЕ.
     Оно НЕ может быть интерпретировано как SQL-код.

  5. Execute: выполняется запрос с параметром.
```

**Ключевой момент:** SQL-текст и данные **разделены** на уровне протокола. PostgreSQL разбирает SQL **до** получения данных. Данные не могут изменить структуру запроса.

### Правила безопасности

**✅ ОБЯЗАТЕЛЬНО:**

1. **Всегда параметризируй запросы с пользовательскими данными:**
   ```go
   db.Query("SELECT * FROM users WHERE name = $1", name)
   db.Exec("UPDATE users SET email = $1 WHERE id = $2", email, id)
   ```

2. **Никогда не конкатенируй строки для SQL:**
   ```go
   // ❌ НИКОГДА ТАК НЕ ДЕЛАЙ:
   query := "SELECT * FROM users WHERE name = '" + name + "'"
   ```

**👍 СТОИТ:**

3. **Используй pgx с нативными типами:**
   ```go
   var id int64 = 42
   db.Query("SELECT * FROM users WHERE id = $1", id)  // передаётся как bigint
   ```

**❌ НЕ ДЕЛАЙ:**

4. **Не «экранируй» вручную** — параметризация надёжнее:
   ```go
   // ❌ Плохо: ручное экранирование
   escapedName := strings.ReplaceAll(name, "'", "''")
   query := "SELECT * FROM users WHERE name = '" + escapedName + "'"
   ```

5. **Не параметризируй имя таблицы или колонки** — параметризация только для **значений**:
   ```go
   // ❌ Не работает: $1 не может быть именем таблицы
   db.Query("SELECT * FROM $1 WHERE id = 42", "users")
   
   // ✅ Правильно: имена таблиц/колонок — только из доверенного источника
   table := "users"  // из белого списка, а не от пользователя
   db.Query(fmt.Sprintf("SELECT * FROM %s WHERE id = $1", table), 42)
   ```

### Итог подглавы 8.10

- **SQL-инъекция** — вставка SQL-кода через пользовательские данные.
- **Параметризация** — значения передаются отдельно от SQL-текста.
- **Протокол** — PostgreSQL разбирает SQL до получения данных (Parse → Bind → Execute).
- **Правило:** всегда параметризируй **значения**, никогда не параметризируй **имена таблиц/колонок**.

---

## 8.11 Batch insert: массовые вставки эффективно

В подглаве 8.2 мы разобрали обычный `INSERT`. Но если нужно вставить **тысячи или миллионы строк** — обычные INSERT'ы по одному будут **очень медленными**.

### Проблема: INSERT по одному

```go
// ❌ Медленно: 10 000 отдельных INSERT'ов
for _, user := range users {
    db.Exec("INSERT INTO users (name, email) VALUES ($1, $2)", user.Name, user.Email)
}
// Каждый INSERT:
//   - Отдельный сетевой round-trip (Go → PostgreSQL → Go)
//   - Отдельный Parse → Plan → Execute
//   - Отдельный WAL-запись (Глава 1)
//   - Отдельное обновление индексов (Глава 3)
// 10 000 INSERT'ов = 10 000 round-trips = очень медленно!
```

### Решение: batch insert

**Batch insert** — вставка **многих строк одним запросом**.

```sql
-- Один INSERT с многими VALUES:
INSERT INTO users (name, email) VALUES
    ('Alice', 'alice@mail.com'),
    ('Bob', 'bob@mail.com'),
    ('Charlie', 'charlie@mail.com'),
    ...  -- ещё 9997 строк
;
```

**Почему это быстро:**

- **Один** сетевой round-trip.
- **Один** Parse → Plan → Execute.
- **Одна** WAL-запись (или меньше записей).
- **Одно** обновление индексов (более эффективное).

### Как сделать batch insert в Go

**Способ 1: pgx.Batch (нативная поддержка)**

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"

    "github.com/jackc/pgx/v5"
)

type User struct {
    Name  string
    Email string
}

func main() {
    ctx := context.Background()

    conn, err := pgx.Connect(ctx, os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close(ctx)

    users := []User{
        {"Alice", "alice@mail.com"},
        {"Bob", "bob@mail.com"},
        {"Charlie", "charlie@mail.com"},
        // ... ещё тысячи
    }

    // Batch — пакетная вставка
    batch := &pgx.Batch{}
    for _, u := range users {
        batch.Queue(
            "INSERT INTO users (name, email) VALUES ($1, $2)",
            u.Name, u.Email,
        )
    }

    // Отправляем все INSERT'ы одним вызовом
    results := conn.SendBatch(ctx, batch)
    defer results.Close()

    // Проверяем результаты
    for range users {
        _, err := results.Exec()
        if err != nil {
            log.Fatal(err)
        }
    }

    fmt.Printf("Вставлено %d пользователей\n", len(users))
}
```

**Способ 2: Multi-VALUES INSERT (один SQL-запрос)**

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"
    "strings"

    "github.com/jackc/pgx/v5"
)

func main() {
    ctx := context.Background()

    conn, err := pgx.Connect(ctx, os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close(ctx)

    users := []User{
        {"Alice", "alice@mail.com"},
        {"Bob", "bob@mail.com"},
        {"Charlie", "charlie@mail.com"},
        // ... ещё тысячи
    }

    // Строим один INSERT с многими VALUES
    const batchSize = 1000  // по 1000 строк за запрос
    
    for start := 0; start < len(users); start += batchSize {
        end := start + batchSize
        if end > len(users) {
            end = len(users)
        }
        chunk := users[start:end]

        // INSERT INTO users (name, email) VALUES ($1,$2),($3,$4),($5,$6),...
        placeholders := make([]string, 0, len(chunk))
        args := make([]interface{}, 0, len(chunk)*2)
        
        for i, u := range chunk {
            placeholders = append(placeholders, fmt.Sprintf("($%d,$%d)", i*2+1, i*2+2))
            args = append(args, u.Name, u.Email)
        }

        query := "INSERT INTO users (name, email) VALUES " + strings.Join(placeholders, ",")
        
        _, err := conn.Exec(ctx, query, args...)
        if err != nil {
            log.Fatal(err)
        }
    }

    fmt.Printf("Вставлено %d пользователей\n", len(users))
}
```

**Способ 3: COPY FROM (самый быстрый)**

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"

    "github.com/jackc/pgx/v5"
)

func main() {
    ctx := context.Background()

    conn, err := pgx.Connect(ctx, os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close(ctx)

    // COPY — самый быстрый способ массовой вставки
    rows := [][]interface{}{
        {"Alice", "alice@mail.com"},
        {"Bob", "bob@mail.com"},
        {"Charlie", "charlie@mail.com"},
        // ... ещё миллионы
    }

    copied, err := conn.CopyFrom(
        ctx,
        pgx.Identifier{"users"},           // таблица
        []string{"name", "email"},          // колонки
        pgx.CopyFromRows(rows),             // данные
    )
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("Вставлено %d строк через COPY\n", copied)
}
```

### Сравнение скорости

| Способ | 100 000 строк | Примечание |
|:---|:---|:---|
| INSERT по одному | **30-60 сек** | 100 000 round-trips |
| pgx.Batch | **2-5 сек** | Один round-trip, но много отдельных INSERT |
| Multi-VALUES INSERT | **1-3 сек** | Один SQL-запрос с многими VALUES |
| COPY FROM | **< 1 сек** | Самый быстрый, минимальный оверхед |

### Что происходит под капотом при COPY

Вспомни Главу 1: `COPY` использует **потоковую передачу** через Wire Protocol — данные идут непрерывным потоком, а не отдельными сообщениями.

**COPY FROM:**

```
1. Backend-процесс получает данные непрерывным потоком.
2. Для каждой строки:
   - Создаёт кортеж (Глава 2).
   - Записывает WAL (Глава 1) — но эффективнее, чем отдельные INSERT.
   - Обновляет индексы (Глава 3) — батчами.
3. В конце: WAL-запись о завершении COPY.
```

**Почему быстрее:** нет накладных расходов на Parse/Plan для каждой строки, меньше WAL-записей, меньше переключений контекста.

### 💡 Практика: batch insert

**✅ ОБЯЗАТЕЛЬНО:**

1. **Для массовых вставок (> 1000 строк) используй batch:**
   - `pgx.Batch` — для обычных INSERT'ов.
   - `Multi-VALUES INSERT` — по 500-1000 строк за запрос.
   - `COPY FROM` — для максимальной скорости.

2. **Разбивай на батчи по 500-1000 строк:**
   ```go
   const batchSize = 1000
   ```

**👍 СТОИТ:**

3. **Используй транзакцию для batch insert:**
   ```go
   tx, _ := conn.Begin(ctx)
   defer tx.Rollback(ctx)
   // batch insert...
   tx.Commit(ctx)
   ```

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

4. **Для < 100 строк — обычные INSERT'ы достаточно быстры.**

**❌ НЕ ДЕЛАЙ:**

5. **Не вставляй миллионы строк одним INSERT** — огромный запрос может упереться в лимиты памяти и WAL.

6. **Не используй COPY для таблиц с триггерами** — COPY может пропускать триггеры (зависит от настроек).

### Итог подглавы 8.11

- **Batch insert** — вставка многих строк одним запросом.
- **pgx.Batch** — пакетная отправка INSERT'ов.
- **Multi-VALUES INSERT** — один SQL с многими VALUES.
- **COPY FROM** — самый быстрый способ.
- **Разбивай на батчи** по 500-1000 строк.
- **Используй транзакцию** для согласованности.

---

## 8.12 Практика Go: работа с SQL в коде

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "os"

    _ "github.com/jackc/pgx/v5/stdlib"
)

type User struct {
    ID    int64
    Name  string
    Email string
}

func main() {
    db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    ctx := context.Background()

    // INSERT с RETURNING (параметризованный!)
    var userID int64
    err = db.QueryRowContext(ctx,
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id",
        "Alice", "alice@mail.com",  // ← параметры, не конкатенация!
    ).Scan(&userID)
    fmt.Printf("Создан пользователь id=%d\n", userID)

    // SELECT с фильтром (параметризованный!)
    var user User
    err = db.QueryRowContext(ctx,
        "SELECT id, name, COALESCE(email, '') FROM users WHERE id = $1",
        userID,  // ← параметр
    ).Scan(&user.ID, &user.Name, &user.Email)

    // Keyset пагинация (параметризованная!)
    products, lastID, _ := getProducts(ctx, db, 0, 10)
    fmt.Printf("Загружено %d товаров, lastID=%d\n", len(products), lastID)
}

func getProducts(ctx context.Context, db *sql.DB, afterID int64, limit int) ([]string, int64, error) {
    rows, err := db.QueryContext(ctx,
        "SELECT id, name FROM products WHERE id > $1 ORDER BY id LIMIT $2",
        afterID, limit,  // ← параметры
    )
    if err != nil {
        return nil, 0, err
    }
    defer rows.Close()

    var names []string
    var lastID int64
    for rows.Next() {
        var id int64
        var name string
        rows.Scan(&id, &name)
        names = append(names, name)
        lastID = id
    }
    return names, lastID, nil
}
```

---

## 8.13 Выводы и типичные ошибки

**Что мы узнали?**

SQL — декларативный язык: ты говоришь «что», а PostgreSQL — «как». Но понимание «как» (индексы, MVCC, VACUUM) помогает писать эффективные запросы. Параметризация защищает от SQL-инъекций. Batch insert ускоряет массовые вставки в десятки раз.

**Типичные ошибки:**

- ❌ `SELECT *` в проде — читает все колонки зря.
- ❌ `WHERE col = NULL` — не работает, только `IS NULL`.
- ❌ `LIKE '%...%'` без GIN-индекса — Seq Scan.
- ❌ OFFSET > 10000 — деградация.
- ❌ UPDATE без WHERE — массовое создание мёртвых кортежей.
- ❌ FK без индекса — Seq Scan на JOIN.
- ❌ DISTINCT без необходимости — дорогая сортировка.
- ❌ Конкатенация строк для SQL — SQL-инъекция. Всегда параметризируй.
- ❌ INSERT по одному в цикле — используй batch insert.
- ❌ Параметризация имён таблиц — параметризируются только **значения**.

---

## 8.14 Для быстрого повторения

- **Порядок выполнения:** FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT.
- **JOIN'ы:** INNER (совпадения), LEFT (все из левой), FULL (все), CROSS (декарт), LATERAL (подзапрос с доступом).
- **Подзапросы:** скалярные (одно значение), табличные (FROM), коррелированные (N раз).
- **UNION ALL** — быстрее UNION (без удаления дубликатов).
- **CTE** — читаемость, MATERIALIZED, RECURSIVE.
- **View** — сохранённый SQL. **MV** — сохранённый результат, нужен REFRESH.
- **Пагинация:** Keyset для больших таблиц, OFFSET для маленьких.
- **UPSERT:** ON CONFLICT DO UPDATE / DO NOTHING.
- **TRUNCATE** — мгновенно, не создаёт мёртвых кортежей.
- **Параметризация** — значения отдельно от SQL-текста, защита от инъекций.
- **Batch insert** — pgx.Batch, Multi-VALUES, COPY FROM. Разбивай на батчи 500-1000.

---

## 8.15 Вопросы для самопроверки

1. Какой порядок выполнения операторов в SELECT?
2. Чем LEFT JOIN отличается от INNER JOIN?
3. Когда использовать EXISTS вместо IN?
4. Чем UNION отличается от UNION ALL?
5. Когда использовать CTE вместо подзапроса?
6. Чем View отличается от Materialized View?
7. Почему OFFSET деградирует на больших смещениях?
8. Как работает Keyset pagination?
9. Что делает UPDATE под капотом?
10. Почему FK не создаёт индекс автоматически?
11. Что такое SQL-инъекция? Как параметризация защищает от неё?
12. Что такое batch insert? Какие способы batch insert есть в Go?
13. Почему COPY FROM быстрее, чем INSERT по одному?

---

## 8.16 Ответы

### Ответ 1

FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT → OFFSET.

### Ответ 2

INNER JOIN возвращает только совпадения. LEFT JOIN возвращает все строки из левой таблицы, для несовпавших — NULL.

### Ответ 3

EXISTS останавливается при первом совпадении — быстрее на больших подзапросах. IN строит весь список.

### Ответ 4

UNION удаляет дубликаты (сортирует). UNION ALL сохраняет дубликаты (быстрее).

### Ответ 5

CTE — для читаемости, переиспользования, рекурсии. Подзапрос — для простых случаев.

### Ответ 6

View — сохранённый SQL-запрос, выполняется каждый раз. MV — сохранённый результат, читается быстро, но устаревает.

### Ответ 7

OFFSET читает и отбрасывает N строк зря. Чем больше OFFSET, тем больше строк читается впустую.

### Ответ 8

Вместо пропуска строк — фильтр по ключу: `WHERE id > last_id LIMIT N`. Читает только нужные строки.

### Ответ 9

UPDATE создаёт новую версию строки (t_xmin), помечает старую (t_xmax). Старая становится мёртвым кортежем — VACUUM удалит.

### Ответ 10

Потому что REFERENCES проверяет существование в **другой** таблице (по её PK). Для фильтрации по FK-колонке индекс нужно создавать вручную.

### Ответ 11

SQL-инъекция — атака, при которой злоумышленник вставляет SQL-код через пользовательские данные. Параметризация защищает: SQL-текст разбирается **до** получения данных (Parse → Bind → Execute), данные передаются отдельно и не могут изменить структуру запроса.

### Ответ 12

Batch insert — вставка многих строк одним запросом. Способы в Go: `pgx.Batch` (пакетная отправка), Multi-VALUES INSERT (один SQL с многими VALUES), `COPY FROM` (самый быстрый). Разбивать на батчи по 500-1000 строк.

### Ответ 13

COPY FROM использует потоковую передачу через Wire Protocol — данные идут непрерывным потоком. Нет накладных расходов на Parse/Plan для каждой строки, меньше WAL-записей, эффективнее обновление индексов.

---

## 8.17 Куда идти дальше?

Мы разобрали **как писать SQL**. Теперь — **как PostgreSQL выполняет** эти запросы на уровне алгоритмов.

**Глава 9: Сканирование и соединения — алгоритмы, которые решают судьбу продакшена.**