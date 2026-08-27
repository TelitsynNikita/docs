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

**После прочтения вы сможете:**
- Выбрать правильный JOIN под задачу.
- Написать запрос, который вернёт именно то, что нужно.
- Понять, когда использовать CTE, а когда подзапрос.
- Модифицировать данные с RETURNING и UPSERT.
- Спроектировать базовые таблицы с правильными constraints.
- Написать пагинацию, которая не деградирует на больших объёмах.
- Понимать, что происходит под капотом для каждой SQL-команды.

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
- [8.10 Практика Go: работа с SQL в коде](#810-практика-go-работа-с-sql-в-коде)
- [8.11 Выводы и типичные ошибки](#811-выводы-и-типичные-ошибки)
- [8.12 Для быстрого повторения](#812-для-быстрого-повторения)
- [8.13 Вопросы для самопроверки](#813-вопросы-для-самопроверки)
- [8.14 Ответы](#814-ответы)
- [8.15 Куда идти дальше?](#815-куда-идти-дальше)

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

## 8.10 Практика Go: работа с SQL в коде

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

    // INSERT с RETURNING
    var userID int64
    err = db.QueryRowContext(ctx,
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id",
        "Alice", "alice@mail.com",
    ).Scan(&userID)
    fmt.Printf("Создан пользователь id=%d\n", userID)

    // SELECT с фильтром
    var user User
    err = db.QueryRowContext(ctx,
        "SELECT id, name, COALESCE(email, '') FROM users WHERE id = $1",
        userID,
    ).Scan(&user.ID, &user.Name, &user.Email)

    // Keyset пагинация
    products, lastID, _ := getProducts(ctx, db, 0, 10)
    fmt.Printf("Загружено %d товаров, lastID=%d\n", len(products), lastID)
}

func getProducts(ctx context.Context, db *sql.DB, afterID int64, limit int) ([]string, int64, error) {
    rows, err := db.QueryContext(ctx,
        "SELECT id, name FROM products WHERE id > $1 ORDER BY id LIMIT $2",
        afterID, limit,
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

## 8.11 Выводы и типичные ошибки

**Что мы узнали?**

SQL — декларативный язык: ты говоришь «что», а PostgreSQL — «как». Но понимание «как» (индексы, MVCC, VACUUM) помогает писать эффективные запросы.

**Типичные ошибки:**

- ❌ `SELECT *` в проде — читает все колонки зря.
- ❌ `WHERE col = NULL` — не работает, только `IS NULL`.
- ❌ `LIKE '%...%'` без GIN-индекса — Seq Scan.
- ❌ OFFSET > 10000 — деградация.
- ❌ UPDATE без WHERE — массовое создание мёртвых кортежей.
- ❌ FK без индекса — Seq Scan на JOIN.
- ❌ DISTINCT без необходимости — дорогая сортировка.

---

## 8.12 Для быстрого повторения

- **Порядок выполнения:** FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT.
- **JOIN'ы:** INNER (совпадения), LEFT (все из левой), FULL (все), CROSS (декарт), LATERAL (подзапрос с доступом).
- **Подзапросы:** скалярные (одно значение), табличные (FROM), коррелированные (N раз).
- **UNION ALL** — быстрее UNION (без удаления дубликатов).
- **CTE** — читаемость, MATERIALIZED, RECURSIVE.
- **View** — сохранённый SQL. **MV** — сохранённый результат, нужен REFRESH.
- **Пагинация:** Keyset для больших таблиц, OFFSET для маленьких.
- **UPSERT:** ON CONFLICT DO UPDATE / DO NOTHING.
- **TRUNCATE** — мгновенно, не создаёт мёртвых кортежей.

---

## 8.13 Вопросы для самопроверки

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

---

## 8.14 Ответы

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

---

## 8.15 Куда идти дальше?

Мы разобрали **как писать SQL**. Теперь — **как PostgreSQL выполняет** эти запросы на уровне алгоритмов.

**Глава 9: Сканирование и соединения — алгоритмы, которые решают судьбу продакшена.**