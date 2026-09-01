# 📐 Глава 11: Нормализация и денормализация — искусство проектирования схем

**Что вы узнаете:**
- Что такое нормальные формы (1НФ, 2НФ, 3НФ, НФБК) и зачем они нужны.
- Как находить аномалии вставки, обновления и удаления.
- Когда денормализация оправдана, а когда — преждевременная оптимизация.
- Как «бизнес через год» меняет требования и что делать со схемой.
- Как нормализация влияет на JOIN'ы, индексы и производительность.
- Практические паттерны: связь 1:1, 1:N, N:M, иерархии.
- Как выбирать первичные ключи и именовать таблицы.
- Как проектировать схему с нуля и проводить миграции без блокировки.

**После прочтения вы сможете:**
- Спроектировать схему в 3НФ с нуля.
- Объяснить, почему твоя схема нарушает нормальную форму и к чему это приведёт.
- Принять осознанное решение о денормализации под конкретный запрос.
- Провести миграцию схемы без блокировки таблицы.
- Выбрать правильный первичный ключ и именовать таблицы по конвенциям.
- Связать нормализацию с индексами, JOIN'ами и VACUUM'ом из глав 1-10.

---

## Содержание

- [11.0 Пролог: таблица, которая свела с ума](#110-пролог-таблица-которая-свела-с-ума)
- [11.1 Зачем вообще нормализация: аномалии вставки, обновления, удаления](#111-зачем-вообще-нормализация-аномалии-вставки-обновления-удаления)
- [11.2 Первая нормальная форма (1НФ): атомарность](#112-первая-нормальная-форма-1нф-атомарность)
- [11.3 Вторая нормальная форма (2НФ): полная зависимость от ключа](#113-вторая-нормальная-форма-2нф-полная-зависимость-от-ключа)
- [11.4 Третья нормальная форма (3НФ): никаких транзитивных зависимостей](#114-третья-нормальная-форма-3нф-никаких-транзитивных-зависимостей)
- [11.5 Нормализация в цифрах: объём, JOIN'ы, UPDATE](#115-нормализация-в-цифрах-объём-joinы-update)
- [11.6 Нормальная форма Бойса-Кодда (НФБК): более строгая 3НФ](#116-нормальная-форма-бойса-кодда-нфбк-более-строгая-3нф)
- [11.7 Денормализация: когда скорость важнее чистоты](#117-денормализация-когда-скорость-важнее-чистоты)
- [11.8 Связи: 1:1, 1:N, N:M, иерархии](#118-связи-11-1n-nm-иерархии)
- [11.9 Выбор первичного ключа: суррогатный vs естественный](#119-выбор-первичного-ключа-суррогатный-vs-естественный)
- [11.10 Именование таблиц и колонок](#1110-именование-таблиц-и-колонок)
- [11.11 Антипаттерны проектирования](#1111-антипаттерны-проектирования)
- [11.12 История изменений (audit log)](#1112-история-изменений-audit-log)
- [11.13 Проектирование с нуля: пошаговый алгоритм](#1113-проектирование-с-нуля-пошаговый-алгоритм)
- [11.14 Миграция схемы без блокировки](#1114-миграция-схемы-без-блокировки)
- [11.15 Выводы и типичные ошибки](#1115-выводы-и-типичные-ошибки)
- [11.16 Для быстрого повторения](#1116-для-быстрого-повторения)
- [11.17 Вопросы для самопроверки](#1117-вопросы-для-самопроверки)
- [11.18 Ответы](#1118-ответы)
- [11.19 Куда идти дальше?](#1119-куда-идти-дальше)

---

## 11.0 Пролог: таблица, которая свела с ума

Ты устроился в компанию, где база проектировалась «на коленке». Первое, что бросается в глаза — таблица `orders`:

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_name TEXT,          -- имя клиента
    customer_email TEXT,         -- email клиента
    customer_phone TEXT,         -- телефон клиента
    product_name TEXT,           -- название товара
    product_price NUMERIC,       -- цена товара
    category_name TEXT,          -- категория товара
    quantity INT,                -- количество
    total NUMERIC,               -- сумма
    tags TEXT,                   -- 'electronics,gadget,sale'
    created_at TIMESTAMPTZ
);
```

Всё в одной куче. Работает? Да. Но:

- **Один и тот же клиент** — в каждой строке заказа. Поменял email → нужно обновить 10 000 строк.
- **Один и тот же товар** — в каждой строке. Изменилась цена → нужно обновить все заказы с этим товаром.
- **tags** — строка с запятыми. Нельзя нормально искать «все товары из категории gadget».
- **Категория** — просто текст. Нельзя переименовать категорию в одном месте.

❓ **Что это за болезнь и как её лечить?**

💡 Ответ: **отсутствие нормализации**. В этой главе мы разберём, как привести эту таблицу в порядок, и **почему** это нужно делать до того, как бизнес вырастет.

---

## 11.1 Зачем вообще нормализация: аномалии вставки, обновления, удаления

Нормализация — это не «красиво». Это **защита от аномалий**.

**Аномалия** — ситуация, когда данные становятся противоречивыми из-за неправильной структуры таблицы.

### Аномалия вставки

Ты хочешь добавить нового клиента, который ещё не сделал ни одного заказа. Но в таблице `orders` — только заказы. **Куда вставить клиента?**

```sql
INSERT INTO orders (customer_name, customer_email)
VALUES ('Новый Клиент', 'new@mail.com');
-- Остальные поля: NULL, NULL, NULL...
-- Клиент «без заказа» — мусорная строка.
```

### Аномалия обновления

Клиент сменил email. Нужно обновить **все** его заказы:

```sql
UPDATE orders SET customer_email = 'new@mail.com'
WHERE customer_email = 'old@mail.com';
-- 10 000 строк. Если хотя бы одну пропустить — противоречие.
```

### Аномалия удаления

Ты удалил последний заказ клиента. **Вместе с заказом исчезли данные клиента:**

```sql
DELETE FROM orders WHERE customer_email = 'alice@mail.com';
-- Заказ удалён. Теперь мы не знаем, что Alice существует.
```

### Как нормализация решает эти аномалии

**Идея:** вынести клиентов в отдельную таблицу, товары — в отдельную, категории — в отдельную. В заказах — только ссылки (ID).

```
ДО:  orders (customer_name, customer_email, product_name, product_price, ...)

ПОСЛЕ:
  customers (id, name, email, phone)         ← клиенты отдельно
  products  (id, name, price, category_id)   ← товары отдельно
  categories (id, name)                      ← категории отдельно
  orders (id, customer_id, product_id, quantity, total, created_at)
              ↑ FK          ↑ FK
```

**Теперь:**

- **Вставка клиента** → `INSERT INTO customers` — без заказов.
- **Обновление email** → `UPDATE customers WHERE id = 42` — одна строка.
- **Удаление заказа** → клиент остаётся в `customers`.

**Вот что даёт нормализация.** В следующих подглавах разберём **как** к этому прийти — через нормальные формы.

---

## 11.2 Первая нормальная форма (1НФ): атомарность

### Определение

> **Первая нормальная форма (1НФ)** — таблица находится в 1НФ, если каждая колонка содержит **атомарное** значение: одно значение на строку, без массивов, списков или вложенных структур.

**Простыми словами:** в одной ячейке — одно значение.

### Проблема: колонка `tags`

Вернёмся к таблице из пролога:

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_name TEXT,
    tags TEXT,  -- 'electronics,gadget,sale'
    ...
);
```

Колонка `tags` хранит **несколько значений** через запятую: `'electronics,gadget,sale'`. Это нарушение 1НФ.

**Почему это проблема:**

```sql
-- ❌ Поиск по тегу:
SELECT * FROM orders WHERE tags LIKE '%gadget%';
-- Seq Scan (Глава 9) — не может использовать индекс.
-- 'gadget' найдётся и в 'notgadget'.

-- ❌ Добавление тега:
UPDATE orders SET tags = tags || ',newtag' WHERE id = 42;
-- Строковая манипуляция, ломается при NULL.

-- ❌ Удаление тега:
UPDATE orders SET tags = REPLACE(tags, ',sale', '') WHERE id = 42;
-- REPLACE может задеть 'sale' в середине другого слова.

-- ❌ Подсчёт заказов по тегу:
SELECT tags, COUNT(*) FROM orders GROUP BY tags;
-- Группировка по всей строке 'electronics,gadget,sale',
-- а не по отдельным тегам.
```

### Решение: вынести в отдельную таблицу

```sql
-- Таблица тегов
CREATE TABLE tags (
    id BIGSERIAL PRIMARY KEY,
    name TEXT UNIQUE NOT NULL
);

-- Связь N:M между заказами и тегами
CREATE TABLE order_tags (
    order_id BIGINT REFERENCES orders(id) ON DELETE CASCADE,
    tag_id BIGINT REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (order_id, tag_id)
);

-- Теперь поиск по тегу:
SELECT o.*
FROM orders o
JOIN order_tags ot ON ot.order_id = o.id
JOIN tags t ON t.id = ot.tag_id
WHERE t.name = 'gadget';
-- Index Scan по tags.name (Глава 3) + Index Scan по order_tags.order_id
```

### Что изменилось

| Операция | До (CSV в колонке) | После (отдельная таблица) |
|:---|:---|:---|
| **Поиск по тегу** | `LIKE '%gadget%'` — Seq Scan | `t.name = 'gadget'` — Index Scan |
| **Добавление тега** | `UPDATE ... REPLACE` — строковая манипуляция | `INSERT INTO order_tags` |
| **Удаление тега** | `UPDATE ... REPLACE` — опасно | `DELETE FROM order_tags WHERE ...` |
| **Подсчёт по тегу** | Группировка по CSV-строке | `GROUP BY t.name` — честно |
| **Уникальность тега** | Невозможна (повторы в CSV) | `UNIQUE` constraint |

### Другие примеры нарушения 1НФ

```sql
-- ❌ 1: Колонка с JSONB (если значения используются по отдельности)
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    phones JSONB  -- '["+79160000001", "+79160000002"]'
);

-- ❌ 2: Колонки с номерами
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    image_url_1 TEXT,
    image_url_2 TEXT,
    image_url_3 TEXT  -- а если будет 4 фото?
);

-- ❌ 3: Колонка с диапазоном
CREATE TABLE discounts (
    id BIGSERIAL PRIMARY KEY,
    price_range TEXT  -- '1000-5000'
);
```

### Когда JSONB в колонке — это НЕ нарушение 1НФ

Важное уточнение. В главе 10 мы говорили, что JSONB — легитимный тип. Где граница?

**ОК — JSONB как единое целое:**

```sql
-- settings хранится как единый объект, по частям не ищут:
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    settings JSONB  -- {'theme': 'dark', 'notifications': {'email': true}}
);
```

Запросы идут **по всей колонке**:

```sql
SELECT * FROM users WHERE settings @> '{"theme": "dark"}';
-- GIN-индекс (Глава 4). Целое значение.
```

**НЕ ОК — JSONB, из которого регулярно извлекают отдельные поля:**

Телефон — это **идентификатор** пользователя, как email или паспорт. Мы ищем по нему **постоянно**:

- **Вход в систему:** «найди пользователя по телефону».
- **Регистрация:** «проверь, не занят ли телефон».
- **Восстановление доступа:** «найди по телефону».

Телефон из массива `phones` — это **самостоятельная сущность**, у которой есть своя жизнь: уникальность, подтверждение, привязка к пользователю.

```sql
-- Если постоянно ищем пользователей по phone:
SELECT * FROM users WHERE phones @> '["+79160000001"]';
-- GIN-индекс работает, но структура вложенная.

-- Лучше:
CREATE TABLE user_phones (
    user_id BIGINT REFERENCES users(id),
    phone TEXT UNIQUE,
    PRIMARY KEY (user_id, phone)
);
-- Проще: UNIQUE constraint, JOIN'ы, индекс.
```

**Критерий:**

| Аспект | settings.theme | phone |
|:---|:---|:---|
| **Частота поиска** | Редко (аналитика) | Постоянно (каждый вход) |
| **Роль** | Свойство внутри объекта | Идентификатор пользователя |
| **Уникальность** | Не нужна | Обязательна |
| **Изменение** | Редко | Может быть |
| **JOIN по этому полю** | Нет | Да |

**Правило:** если поле из JSONB ты ловишь в `WHERE`, `JOIN`, `UNIQUE` — это уже не «настройка», это колонка. Вынеси.

### «Бизнес через год» и 1НФ

Бизнес говорит: «Хочу видеть, сколько заказов по каждому тегу за месяц».

```sql
-- Если tags в CSV: нужно парсить каждую строку.
-- Если tags в отдельной таблице: простой GROUP BY.

SELECT t.name, COUNT(o.id)
FROM orders o
JOIN order_tags ot ON ot.order_id = o.id
JOIN tags t ON t.id = ot.tag_id
WHERE o.created_at > now() - interval '1 month'
GROUP BY t.name;
-- Работает быстро, использует индексы, ничего не парсит.
```

**Вывод:** 1НФ — это не «академическое правило». Это **основа для индексов, JOIN'ов и агрегаций**. CSV в колонке — это таблица, которую нельзя ни индексировать, ни фильтровать, ни группировать.

---

## 11.3 Вторая нормальная форма (2НФ): полная зависимость от ключа

### Определение

> **Вторая нормальная форма (2НФ)** — таблица находится в 2НФ, если она находится в 1НФ и каждый неключевой атрибут **полностью зависит** от первичного ключа, а не от его части.

**Простыми словами:** если первичный ключ составной — каждая колонка должна зависеть от **всего** ключа, а не от одной его части.

### Проблема: составной ключ и частичная зависимость

Вернёмся к прологу. После приведения к 1НФ у нас есть таблица `order_tags`:

```sql
CREATE TABLE order_tags (
    order_id BIGINT REFERENCES orders(id),
    tag_id BIGINT REFERENCES tags(id),
    PRIMARY KEY (order_id, tag_id)
);
```

Составной ключ: `(order_id, tag_id)`. Но есть проблема — представь, что мы решили добавить в эту таблицу **название тега** и **дату создания тега**:

```sql
CREATE TABLE order_tags (
    order_id BIGINT,
    tag_id BIGINT,
    tag_name TEXT,          -- ❌ зависит только от tag_id!
    tag_created_at TIMESTAMPTZ,  -- ❌ зависит только от tag_id!
    PRIMARY KEY (order_id, tag_id)
);
```

**Что не так:**

- `tag_name` — название тега. Оно зависит **только от `tag_id`**, а не от пары `(order_id, tag_id)`.
- `tag_created_at` — когда создан тег. Тоже зависит только от `tag_id`.

Это **частичная зависимость** — колонка зависит от части составного ключа.

### К чему приводит частичная зависимость

**Аномалия обновления:**

```sql
-- Переименовали тег 'gadget' → 'gadgets':
UPDATE order_tags SET tag_name = 'gadgets' WHERE tag_id = 42;
-- Обновили 100 000 строк, во всех заказах с этим тегом.
-- Если одну пропустить — противоречие: в одном заказе 'gadget', в другом 'gadgets'.
```

**Аномалия вставки:**

```sql
-- Хочешь добавить новый тег, который ещё не привязан к заказам:
INSERT INTO order_tags (tag_id, tag_name, tag_created_at)
VALUES (99, 'newtag', now());
-- Нельзя: order_id — часть PRIMARY KEY, NULL недопустим.
```

**Аномалия удаления:**

```sql
-- Удалил последний заказ с тегом 'gadget':
DELETE FROM order_tags WHERE tag_id = 42;
-- Вместе с заказом исчезли название и дата создания тега!
```

### Решение: вынести частичную зависимость в отдельную таблицу

```sql
-- Таблица тегов: tag_name и tag_created_at зависят от tag_id
CREATE TABLE tags (
    id BIGSERIAL PRIMARY KEY,
    name TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Таблица связей: только ссылки
CREATE TABLE order_tags (
    order_id BIGINT REFERENCES orders(id) ON DELETE CASCADE,
    tag_id BIGINT REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (order_id, tag_id)
);
```

Теперь:

```sql
-- Переименование тега — одна строка:
UPDATE tags SET name = 'gadgets' WHERE id = 42;

-- Новый тег без заказов — можно:
INSERT INTO tags (name) VALUES ('newtag');

-- Удаление заказа — тег остаётся:
DELETE FROM order_tags WHERE order_id = 1;
-- tags не тронуты.
```

### Другой пример: таблица заказов с составным ключом

```sql
CREATE TABLE order_items (
    order_id BIGINT,
    product_id BIGINT,
    product_name TEXT,        -- ❌ зависит только от product_id
    product_price NUMERIC,    -- ❌ зависит только от product_id
    quantity INT,             -- ✅ зависит от (order_id, product_id)
    PRIMARY KEY (order_id, product_id)
);
```

**Проблема:** `product_name` и `product_price` зависят только от `product_id`, а не от всего составного ключа.

**Решение:**

```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC NOT NULL
);

CREATE TABLE order_items (
    order_id BIGINT REFERENCES orders(id),
    product_id BIGINT REFERENCES products(id),
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

### Как определить, что таблица НЕ в 2НФ

**Тест:** спроси себя — «если я знаю **часть** ключа, могу ли я определить значение этой колонки?»

```
order_tags (order_id, tag_id, tag_name, tag_created_at)

Вопрос: если я знаю tag_id = 42, могу ли я определить tag_name?
Ответ: ДА → tag_name зависит только от tag_id → нарушение 2НФ.

Вопрос: если я знаю order_id = 1, могу ли я определить tag_name?
Ответ: НЕТ → tag_name не зависит от order_id.
```

**Если колонка определяется по части ключа — выноси её вместе с этой частью в отдельную таблицу.**

### Важный нюанс: таблица с простым ключом всегда в 2НФ

Если первичный ключ — **одна колонка** (суррогатный `id`), то частичная зависимость **невозможна** — нет «частей ключа».

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,  -- простой ключ
    name TEXT,
    email TEXT
);
-- Все колонки зависят от id. 2НФ автоматически.
```

**Вот почему суррогатные ключи (`id BIGSERIAL`) так популярны** — они упрощают нормализацию. Но это не значит, что составные ключи не нужны (вернёмся к этому в 11.9).

### «Бизнес через год» и 2НФ

Бизнес говорит: «Хочу видеть дату создания каждого тега».

**Если tag_created_at в order_tags:**

```sql
SELECT DISTINCT tag_id, tag_created_at FROM order_tags;
-- DISTINCT по таблице на 100 млн строк. Больно.
```

**Если tags отдельно:**

```sql
SELECT * FROM tags;
-- Маленькая таблица. Мгновенно.
```

**Вывод:** 2НФ — это не «красиво». Это **устранение дублирования данных, которые зависят от части ключа**. Дублирование → аномалии обновления → противоречивые данные.

---

## 11.4 Третья нормальная форма (3НФ): никаких транзитивных зависимостей

### Определение

> **Третья нормальная форма (3НФ)** — таблица находится в 3НФ, если она находится в 2НФ и каждый неключевой атрибут **не транзитивно зависит** от первичного ключа.

**Простыми словами:** неключевые колонки не должны зависеть от других неключевых колонок.

### Проблема: транзитивная зависимость

Вернёмся к таблице `orders` из пролога. После приведения к 2НФ у нас может остаться такая структура:

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,       -- ключ
    customer_id BIGINT,             -- неключевая
    customer_name TEXT,             -- ❌ зависит от customer_id!
    customer_email TEXT,            -- ❌ зависит от customer_id!
    product_id BIGINT,              -- неключевая
    product_name TEXT,              -- ❌ зависит от product_id!
    product_price NUMERIC,          -- ❌ зависит от product_id!
    category_id BIGINT,             -- неключевая
    category_name TEXT,             -- ❌ зависит от category_id!
    quantity INT,                   -- ✅ зависит от ключа (id заказа)
    total NUMERIC,                  -- ✅ зависит от ключа
    created_at TIMESTAMPTZ          -- ✅ зависит от ключа
);
```

**Что не так:**

- `customer_name` и `customer_email` зависят от `customer_id`, а не от `id` заказа.
- `product_name` и `product_price` зависят от `product_id`, а не от `id` заказа.
- `category_name` зависит от `category_id`.

Это **транзитивная зависимость**: `id → customer_id → customer_email`.

```
id (заказ) → customer_id → customer_email
     ↑            ↑              ↑
   ключ      неключевая      неключевая
             (FK)            зависит от customer_id
```

### К чему приводит транзитивная зависимость

**Аномалия обновления:**

```sql
-- Клиент сменил email. Обновляем ВСЕ его заказы:
UPDATE orders SET customer_email = 'new@mail.com'
WHERE customer_id = 42;
-- 50 000 строк. Если пропустить одну — противоречие.
```

**Аномалия вставки:**

```sql
-- Новый клиент без заказов:
INSERT INTO orders (customer_id, customer_name, customer_email)
VALUES (99, 'Новый', 'new@mail.com');
-- Нельзя: id заказа — PRIMARY KEY, а заказа нет.
```

**Аномалия удаления:**

```sql
-- Удалили последний заказ клиента:
DELETE FROM orders WHERE customer_id = 42;
-- Данные клиента исчезли вместе с заказом.
```

### Решение: вынести транзитивные зависимости в отдельные таблицы

```sql
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL
);

CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name TEXT UNIQUE NOT NULL
);

CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC NOT NULL,
    category_id BIGINT REFERENCES categories(id)
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT REFERENCES customers(id),
    product_id BIGINT REFERENCES products(id),
    quantity INT NOT NULL,
    total NUMERIC NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

Теперь:

```sql
-- Смена email — одна строка:
UPDATE customers SET email = 'new@mail.com' WHERE id = 42;

-- Новый клиент без заказов:
INSERT INTO customers (name, email) VALUES ('Новый', 'new@mail.com');

-- Удаление заказа — клиент остаётся:
DELETE FROM orders WHERE id = 1;
```

### Как определить, что таблица НЕ в 3НФ

**Тест:** спроси себя — «если изменится эта колонка, нужно ли обновить другие строки с тем же значением?»

```
orders (id, customer_id, customer_name, customer_email, ...)

Вопрос: если customer_email изменится для customer_id = 42,
        нужно ли обновить ВСЕ заказы этого клиента?
Ответ: ДА → customer_email транзитивно зависит от id через customer_id
      → нарушение 3НФ.
```

**Правило:** каждая неключевая колонка должна описывать **факт о ключе**, а не факт о другой неключевой колонке.

```
Хорошо:
  orders.total — факт о заказе (id)
  orders.quantity — факт о заказе (id)
  orders.created_at — факт о заказе (id)

Плохо:
  orders.customer_email — факт о клиенте (customer_id), а не о заказе
  orders.product_price — факт о товаре (product_id), а не о заказе
```

### Практический пример: «но это же удобно держать всё в одной таблице!»

Частый аргумент: «Зачем мне JOIN'ы, если я могу хранить `customer_name` прямо в `orders`?»

**Ответ:** JOIN'ы — это **нормальная работа** PostgreSQL. Индексы (Глава 3) и Hash Join / Nested Loop (Глава 9) созданы для этого.

```sql
-- Один JOIN — Index Scan по PK:
SELECT o.*, c.name, c.email
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.id = 42;
-- Nested Loop (Глава 9): orders → Index Scan по PK → customers → Index Scan по PK
-- 2 чтения индекса. Микросекунды.
```

**Плата за JOIN — ничто** по сравнению с платой за аномалии обновления.

### «Бизнес через год» и 3НФ

Бизнес говорит: «Хочу переименовать категорию "electronics" в "Электроника"».

**Если category_name в orders:**

```sql
UPDATE orders SET category_name = 'Электроника'
WHERE category_name = 'electronics';
-- 5 млн строк. Долго. Блокировки. WAL растёт (Глава 1). VACUUM будет чистить (Глава 6).
```

**Если categories отдельно:**

```sql
UPDATE categories SET name = 'Электроника' WHERE name = 'electronics';
-- 1 строка. Мгновенно.
```

**Вывод:** 3НФ — это **факты о правильных сущностях в правильных таблицах**. Колонка о клиенте — в таблице клиентов. Колонка о товаре — в таблице товаров. Колонка о заказе — в таблице заказов.

---

## 11.5 Нормализация в цифрах: объём, JOIN'ы, UPDATE

До сих пор мы говорили о нормализации качественно: «меньше дублирования», «быстрее обновление». Теперь — **численно**, на конкретном примере.

### Исходные данные

Возьмём интернет-магазин:

```
10 000 000 заказов
 1 000 000 клиентов
   100 000 товаров
     1 000 категорий
        50 тегов
Каждый заказ: 1 клиент, 1 товар, 3 тега
```

### Сравниваемые схемы

**Схема A: Денормализованная (всё в одной таблице)**

```sql
CREATE TABLE orders_denorm (
    id BIGSERIAL PRIMARY KEY,
    customer_name TEXT,
    customer_email TEXT,
    customer_phone TEXT,
    product_name TEXT,
    product_price NUMERIC,
    category_name TEXT,
    tags TEXT,  -- 'electronics,gadget,sale'
    quantity INT,
    total NUMERIC,
    created_at TIMESTAMPTZ
);
```

**Схема B: Нормализованная (3НФ)**

```sql
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    phone TEXT
);

CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name TEXT UNIQUE NOT NULL
);

CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC NOT NULL,
    category_id BIGINT REFERENCES categories(id)
);

CREATE TABLE tags (
    id BIGSERIAL PRIMARY KEY,
    name TEXT UNIQUE NOT NULL
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT REFERENCES customers(id),
    product_id BIGINT REFERENCES products(id),
    quantity INT NOT NULL,
    total NUMERIC NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE order_tags (
    order_id BIGINT REFERENCES orders(id) ON DELETE CASCADE,
    tag_id BIGINT REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (order_id, tag_id)
);
```

### Объём данных

**Схема A (денормализованная):**

```
Каждая строка заказа содержит:
  customer_name   ~20 байт
  customer_email  ~30 байт
  customer_phone  ~15 байт
  product_name    ~30 байт
  product_price   ~12 байт
  category_name   ~15 байт
  tags            ~30 байт
  quantity        ~4 байта
  total           ~12 байт
  created_at      ~8 байт
  + заголовок кортежа ~23 байта

Итого на строку: ~200 байт

10 000 000 строк × 200 байт = 2 000 000 000 байт ≈ 1.86 ГБ

Плюс индексы:
  PRIMARY KEY (id): ~280 МБ (B-tree, Глава 3)

Итого: ~2.4 ГБ
```

**Схема B (нормализованная):**

```
Таблица orders:
  10 000 000 × ~71 байт = 710 000 000 ≈ 660 МБ

Таблица customers:
  1 000 000 × ~90 байт = 90 МБ

Таблица products:
  100 000 × ~75 байт = 7.5 МБ

Таблица categories:
  1 000 × ~40 байт = 40 КБ

Таблица tags:
  50 × ~30 байт = 1.5 КБ

Таблица order_tags:
  30 000 000 строк × ~28 байт = 840 МБ

Индексы:
  orders_pkey: ~280 МБ
  customers_email_idx: ~30 МБ
  order_tags_pkey: ~700 МБ
  + FK индексы: ~300 МБ

Итого: ~2.9 ГБ
```

### Сравнение объёма

| Схема | Объём |
|:---|:---|
| A: Денормализованная | **~2.4 ГБ** |
| B: Нормализованная (3НФ) | **~2.9 ГБ** |

**Нормализованная схема занимает на ~20% БОЛЬШЕ.** Это плата за:
- Отдельные таблицы (каждая со своим заголовком страниц)
- Дополнительные индексы (FK)
- Таблица связей `order_tags` (30 млн строк)

**Вывод:** нормализация **не экономит дисковое пространство** в этом сценарии — наоборот, занимает на ~500 МБ больше. Но для базы в 2–3 ГБ это **невесомая переплата**: стоимость одного часа простоя из-за блокировок при массовом UPDATE (Схема A) превышает стоимость 500 МБ дискового пространства. Нормализация экономит **другое** — целостность, скорость обновлений, предсказуемость.

---

### Скорость SELECT: JOIN vs без JOIN

**Запрос: «Получить заказ с клиентом, товаром, категорией и тегами»**

**Схема A (без JOIN):**

```sql
SELECT * FROM orders_denorm WHERE id = 42;
-- Index Scan по PRIMARY KEY (Глава 3): спуск по B-tree + чтение 1 страницы таблицы.
-- 3 чтения индекса + 1 чтение таблицы = 4 страницы.
-- Время: ~0.05 мс (все страницы в shared_buffers, Глава 1)
```

**Схема B (с JOIN):**

```sql
SELECT o.*, c.name, c.email, p.name, p.price, cat.name, t.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
JOIN products p ON p.id = o.product_id
JOIN categories cat ON cat.id = p.category_id
JOIN order_tags ot ON ot.order_id = o.id
JOIN tags t ON t.id = ot.tag_id
WHERE o.id = 42;
```

**Что под капотом (Глава 9):**

```
Nested Loop (для каждого заказа идём по индексам):
  orders → Index Scan по PK (1 чтение)
  customers → Index Scan по PK (1 чтение)
  products → Index Scan по PK (1 чтение)
  categories → Index Scan по PK (1 чтение)
  order_tags → Index Scan по (order_id, tag_id) (1-3 чтения)
  tags → Index Scan по PK (1-3 чтения)

Итого: ~7-10 чтений индексов.
Время: ~0.5-1 мс
```

**Сравнение:**

| Метрика | Схема A | Схема B |
|:---|:---|:---|
| Чтений страниц | 4 | 7-10 |
| Время | ~0.05 мс | ~0.5-1 мс |
| Разница | — | **в 10-20 раз медленнее** |

**JOIN'ы действительно медленнее.** Но:
- 0.5 мс — это всё ещё **микросекунды**. Пользователь не заметит.
- Все чтения идут по индексам в `shared_buffers`.
- Для массовых выборок разница сглаживается Hash Join (Глава 9).

---

### Скорость UPDATE: где нормализация выигрывает

**Сценарий: клиент сменил email**

**Схема A:**

```sql
UPDATE orders_denorm
SET customer_email = 'new@mail.com'
WHERE customer_email = 'old@mail.com';
-- Обновляем ВСЕ заказы клиента: ~10 строк (если у клиента 10 заказов)
-- Но если у клиента 10 000 заказов — обновляем 10 000 строк.

-- Под капотом (Главы 5, 6):
--   Каждый UPDATE создаёт НОВУЮ версию строки (MVCC).
--   Старые версии — мёртвые кортежи. Ждут VACUUM.
--   Каждый UPDATE пишет в WAL (Глава 1).
--   Каждый UPDATE обновляет индексы (Глава 3).
```

**Схема B:**

```sql
UPDATE customers SET email = 'new@mail.com' WHERE id = 42;
-- Обновляем РОВНО 1 строку. Всегда.
-- Заказы клиента не трогаем — они ссылаются по customer_id.
```

**Сравнение для клиента с 1 000 заказов:**

| Метрика | Схема A | Схема B |
|:---|:---|:---|
| Обновлено строк | 1 000 | **1** |
| Новых версий (MVCC, Глава 5) | 1 000 | 1 |
| Мёртвых кортежей (Глава 6) | 1 000 | 1 |
| WAL-записей (Глава 1) | ~1 000 | ~1 |
| Время | ~50 мс | **~0.1 мс** |
| Разница | — | **в 500 раз быстрее** |

**Сценарий: товар подорожал**

| Метрика | Схема A (10 млн заказов с этим товаром) | Схема B |
|:---|:---|:---|
| Обновлено строк | до 10 000 000 | **1** |
| Время | часы | **мгновенно** |
| Блокировки | вся таблица | 1 строка |
| VACUUM после | сотни ГБ мёртвых кортежей | не нужен |

---

### Итоговая таблица: что мы покупаем, что теряем

| Аспект | Денормализованная (A) | Нормализованная 3НФ (B) |
|:---|:---|:---|
| **Объём данных** | 2.4 ГБ ✅ | 2.9 ГБ ❌ (+20%) |
| **SELECT по PK без JOIN** | 0.05 мс ✅ | 0.5-1 мс ❌ (10-20× медленнее) |
| **SELECT с JOIN** | — | 0.5-1 мс ✅ |
| **UPDATE одного факта** | Тысячи/миллионы строк ❌ | **1 строка** ✅ |
| **Аномалии вставки/удаления** | Есть ❌ | Нет ✅ |
| **Противоречивость данных** | Риск ❌ | Исключена ✅ |
| **Сложность SQL** | Просто ✅ | JOIN'ы ❌ |
| **Место под индексы** | Меньше ✅ | Больше ❌ |

### Вывод

**Нормализация — это trade-off:**

- **Платим:** +20% объёма, JOIN'ы, сложнее SQL.
- **Получаем:** целостность, скорость UPDATE, отсутствие аномалий.

**Для OLTP** (много UPDATE/DELETE) — 3НФ обязательна.
**Для OLAP / отчётов** — денормализация оправдана (витрины данных, Materialized Views).

**Правило:** нормализуй **всегда**, денормализуй **осознанно** под конкретный запрос, когда замеры показывают, что JOIN'ы — узкое место.

---

## 11.6 Нормальная форма Бойса-Кодда (НФБК): более строгая 3НФ

### Определение

> **Нормальная форма Бойса-Кодда (НФБК)** — таблица находится в НФБК, если для **каждой** функциональной зависимости `X → Y` левая часть `X` является **ключом** (потенциальным ключом).

**Простыми словами:** в каждой зависимости «одно значение определяет другое» определяющая сторона должна быть ключом.

### Чем НФБК отличается от 3НФ

3НФ запрещает транзитивные зависимости неключевых атрибутов от ключа. Но **3НФ не покрывает** случай, когда:

- Таблица имеет **несколько потенциальных ключей**
- Неключевая колонка зависит от **неключевого атрибута**, который сам является потенциальным ключом

Это редкая ситуация, но она есть. Разберём классический пример.

### Проблема: несколько потенциальных ключей

Представь таблицу бронирований:

```sql
CREATE TABLE bookings (
    booking_id BIGSERIAL PRIMARY KEY,  -- суррогатный ключ
    room_number TEXT NOT NULL,         -- номер комнаты
    guest_email TEXT NOT NULL,         -- email гостя
    start_time TIMESTAMPTZ NOT NULL,
    UNIQUE (room_number, start_time),  -- комната не может быть забронирована дважды в одно время
    UNIQUE (guest_email, start_time)   -- гость не может быть в двух комнатах одновременно
);
```

**Что мы имеем:**

- `booking_id` — суррогатный первичный ключ.
- `(room_number, start_time)` — **потенциальный ключ** (уникален, может быть PK).
- `(guest_email, start_time)` — **потенциальный ключ**.

Все колонки зависят от `booking_id`. Таблица **в 3НФ**. Но:

```
booking_id → room_number
booking_id → guest_email
booking_id → start_time

А также:
(room_number, start_time) → booking_id
(guest_email, start_time) → booking_id
```

**В 3НФ ничего не нарушено** — транзитивных зависимостей нет.

### Где НФБК находит проблему

Добавим колонку, которая должна быть уникальной, но не является ключом:

```sql
CREATE TABLE bookings (
    booking_id BIGSERIAL PRIMARY KEY,
    room_number TEXT NOT NULL,
    guest_email TEXT NOT NULL,
    guest_phone TEXT NOT NULL,
    start_time TIMESTAMPTZ NOT NULL,
    UNIQUE (room_number, start_time),
    UNIQUE (guest_email, start_time),
    UNIQUE (guest_phone, start_time)  -- телефон тоже уникален на время
);
```

**Теперь:**

```
booking_id → guest_phone
guest_phone → booking_id   (если guest_phone уникален)

То есть guest_phone — тоже потенциальный ключ.
```

**Проблема:** если `guest_phone` — потенциальный ключ, но он **не объявлен** как `UNIQUE`, то таблица всё ещё в 3НФ, но **не в НФБК**. Потому что есть зависимость `guest_phone → booking_id`, где `guest_phone` не является объявленным ключом.

**На практике НФБК говорит:** если колонка уникальна — **объяви её ключом** (UNIQUE constraint), иначе получишь противоречия.

### Пример, где 3НФ пропускает, а НФБК ловит

Таблица заказов с **естественным ключом**:

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,         -- суррогатный ключ
    order_number TEXT UNIQUE NOT NULL, -- естественный ключ: 'ORD-2024-001'
    customer_id BIGINT REFERENCES customers(id),
    total NUMERIC
);
```

Здесь:

```
id → order_number
order_number → id   (UNIQUE)
```

Таблица в 3НФ. Но **в НФБК она тоже**, потому что `order_number` объявлен `UNIQUE` — он **является ключом**.

**Если бы `order_number` не был UNIQUE:**

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number TEXT NOT NULL,  -- ❌ не UNIQUE, но бизнес считает его уникальным
    customer_id BIGINT,
    total NUMERIC
);
```

Таблица формально в 3НФ, но **не в НФБК** — потому что по бизнес-логике `order_number → id`, но это не закреплено constraint'ом.

### Как НФБК связана с нашей исходной таблицей

Вернёмся к прологу. После приведения к 3НФ у нас:

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT REFERENCES customers(id),
    product_id BIGINT REFERENCES products(id),
    quantity INT,
    total NUMERIC,
    created_at TIMESTAMPTZ
);
```

**Вопрос:** есть ли здесь несколько потенциальных ключей?

- `id` — суррогатный PK.
- Если бизнес требует уникальность `(customer_id, product_id, created_at)` — это потенциальный ключ, который нужно объявить `UNIQUE`.

**НФБК говорит:** не оставляй уникальные комбинации без `UNIQUE constraint`.

### Практическое правило

**Для большинства таблиц 3НФ = НФБК.** Разница проявляется только при:

- Нескольких потенциальных ключах
- Колонках, которые бизнес считает уникальными, но без `UNIQUE constraint`

**Правило:** если бизнес говорит «это уникально» — **закрепи constraint'ом**:

```sql
-- ✅ Бизнес говорит: «номер заказа уникален»
ALTER TABLE orders ADD CONSTRAINT unique_order_number UNIQUE (order_number);

-- ✅ Бизнес говорит: «клиент не может иметь два одинаковых email»
ALTER TABLE customers ADD CONSTRAINT unique_email UNIQUE (email);
```

### Аномалия, которую предотвращает НФБК

**Сценарий:** `order_number` не объявлен UNIQUE, но бизнес считает его уникальным.

```sql
-- Два заказа с одинаковым номером:
INSERT INTO orders (order_number, customer_id, total)
VALUES ('ORD-2024-001', 42, 1000);

INSERT INTO orders (order_number, customer_id, total)
VALUES ('ORD-2024-001', 43, 2000);

-- Противоречие! Бизнес думает, что order_number уникален.
-- Но БД это не проверяет.
```

**С НФБК:**

```sql
UNIQUE (order_number);
-- Вторая вставка упадёт: ERROR: duplicate key value violates unique constraint
```

### «Бизнес через год» и НФБК

Бизнес говорит: «Нам нужно искать заказы по номеру, и номер должен быть уникальным».

**Если UNIQUE constraint есть:**

```sql
SELECT * FROM orders WHERE order_number = 'ORD-2024-001';
-- Index Scan по unique_order_number (Глава 3). Мгновенно.
```

**Если constraint'а нет:**

```sql
-- Нужно добавить:
ALTER TABLE orders ADD CONSTRAINT unique_order_number UNIQUE (order_number);
-- Но если дубликаты уже есть — ОШИБКА!
-- Придётся чистить данные вручную.
```

### Итог подглавы 11.6

**НФБК** — это 3НФ + требование: **все зависимости должны исходить из объявленных ключей**. На практике:

- Большинство таблиц в 3НФ уже в НФБК.
- НФБК важна, когда есть несколько потенциальных ключей.
- **Правило:** бизнес-уникальность всегда закрепляй `UNIQUE constraint`.
- **Бонус:** UNIQUE автоматически создаёт индекс (Глава 3), ускоряя поиск.

---

## 11.7 Денормализация: когда скорость важнее чистоты

### Определение

> **Денормализация** — намеренное нарушение нормальных форм для ускорения чтения за счёт увеличения объёма данных и усложнения записи.

**Простыми словами:** ты **осознанно** дублируешь данные, чтобы не делать JOIN'ы.

### Когда денормализация оправдана

Денормализация — это **не «откат» к плохой схеме**. Это **инструмент**, который применяют точечно, когда замеры показывают, что JOIN'ы — узкое место.

**Признаки, что денормализация нужна:**

1. **Запрос выполняется часто** (тысячи раз в секунду) и JOIN'ы занимают > 50% времени.
2. **JOIN'ы больших таблиц** (миллионы строк × миллионы строк) — Hash Join сбрасывается на диск (Глава 9, `Batches > 1`).
3. **Отчёты / аналитика** — данные читаются гораздо чаще, чем обновляются.
4. **Число JOIN'ов > 5** — каждый JOIN добавляет задержку.

### Классический пример: денормализация заказов

**Нормализованная схема (3НФ):**

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT REFERENCES customers(id),
    product_id BIGINT REFERENCES products(id),
    quantity INT,
    total NUMERIC,
    created_at TIMESTAMPTZ
);
```

**Частый запрос:**

```sql
SELECT o.id, o.total, o.created_at,
       c.name AS customer_name,
       p.name AS product_name
FROM orders o
JOIN customers c ON c.id = o.customer_id
JOIN products p ON p.id = o.product_id
WHERE o.created_at > now() - interval '1 day';
-- 3 таблицы, 2 JOIN'а. На 10 млн заказов — Hash Join (Глава 9).
```

**Денормализованный вариант:**

```sql
CREATE TABLE orders_denorm (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT,           -- остаётся для FK и целостности
    customer_name TEXT,           -- 🔴 дублируется!
    product_id BIGINT,            -- остаётся
    product_name TEXT,            -- 🔴 дублируется!
    quantity INT,
    total NUMERIC,
    created_at TIMESTAMPTZ
);

-- Теперь запрос БЕЗ JOIN:
SELECT id, total, created_at, customer_name, product_name
FROM orders_denorm
WHERE created_at > now() - interval '1 day';
-- Одна таблица. Seq Scan или Index Scan. Без JOIN'ов.
```

### Численное сравнение: JOIN vs денормализация

**Тестовая нагрузка:** 10 млн заказов, 1 млн клиентов, 100 тыс. товаров.

**Запрос с JOIN (нормализованная):**

```
Hash Join (Глава 9):
  Hash Join
    Hash Cond: (o.customer_id = c.id)
    → Hash Join
        Hash Cond: (o.product_id = p.id)
        → Seq Scan on orders o
        → Hash
            → Seq Scan on products p
    → Hash
        → Seq Scan on customers c

Execution Time: ~1 500 мс
Batches: 2  (хэш сброшен на диск! work_mem = 4 МБ)
temp written: 250 000 страниц
```

**Запрос без JOIN (денормализованная):**

```
Seq Scan on orders_denorm
  Filter: (created_at > now() - interval '1 day')

Execution Time: ~200 мс
Batches: 0
temp written: 0
```

**Разница: в 7.5 раз быстрее.**

### Как денормализация влияет на запись

**UPDATE имени клиента:**

```sql
-- Нормализованная:
UPDATE customers SET name = 'Новое имя' WHERE id = 42;
-- 1 строка. Мгновенно.

-- Денормализованная:
UPDATE orders_denorm SET customer_name = 'Новое имя' WHERE customer_id = 42;
-- 10 000 строк (все заказы клиента). Плюс:
--   MVCC: 10 000 новых версий (Глава 5)
--   VACUUM: 10 000 мёртвых кортежей (Глава 6)
--   WAL: 10 000 записей (Глава 1)
-- Время: секунды или минуты. Блокировки.
```

**Вот цена денормализации:** чтение ускоряется, запись замедляется.

### Способы денормализации

**1. Дублирование колонок:**

```sql
-- Добавили customer_name прямо в orders
ALTER TABLE orders ADD COLUMN customer_name TEXT;
-- Поддерживаем вручную или через триггер
```

**2. Предвычисленные агрегаты:**

```sql
-- Вместо COUNT(*) по order_items каждый раз:
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name TEXT,
    orders_count INT DEFAULT 0,  -- 🔴 предвычисленный счётчик
    total_revenue NUMERIC DEFAULT 0
);

-- Обновляем при каждом заказе:
UPDATE products SET orders_count = orders_count + 1 WHERE id = 42;
-- Теперь «сколько заказов у товара» — без COUNT(*), просто читаем колонку.
```

**3. Материализованные представления (Глава 8):**

```sql
CREATE MATERIALIZED VIEW product_stats AS
SELECT product_id, COUNT(*) AS orders_count, SUM(total) AS total_revenue
FROM orders
GROUP BY product_id;

REFRESH MATERIALIZED VIEW product_stats;  -- периодически
```

**4. JSONB-агрегаты:**

```sql
-- Храним заказы клиента в JSONB:
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    name TEXT,
    recent_orders JSONB  -- [{"id": 1, "total": 100}, ...]
);
-- Читаем «последние заказы» без JOIN.
```

### Когда денормализацию НЕ делать

**❌ Преждевременная денормализация:**

```
Таблица на 10 000 строк. JOIN выполняется за 0.5 мс.
Разработчик: «JOIN'ы медленные, сделаю денормализацию!»
Результат: усложнил схему, замедлил запись, не получил выигрыша.
```

**Правило:** сначала **нормализуй**. Потом **замерь**. Если JOIN'ы реально узкое место — денормализуй **точечно**.

### «Бизнес через год» и денормализация

Бизнес говорит: «Хочу видеть на главной странице топ-100 товаров по продажам за месяц».

**Нормализованная схема:**

```sql
SELECT p.name, COUNT(o.id) AS orders_count
FROM products p
JOIN orders o ON o.product_id = p.id
WHERE o.created_at > now() - interval '1 month'
GROUP BY p.name
ORDER BY orders_count DESC
LIMIT 100;
-- Каждый раз: Hash Join + агрегация по 10 млн заказов. 2-3 секунды.
```

**Денормализация через Materialized View:**

```sql
CREATE MATERIALIZED VIEW top_products AS
SELECT p.name, COUNT(o.id) AS orders_count
FROM products p
JOIN orders o ON o.product_id = p.id
WHERE o.created_at > now() - interval '1 month'
GROUP BY p.name
ORDER BY orders_count DESC
LIMIT 100;

-- Обновляем раз в 5 минут:
REFRESH MATERIALIZED VIEW top_products;

-- Главная страница читает:
SELECT * FROM top_products;
-- Готовый результат. 0.1 мс.
```

### Итог подглавы 11.7

| Аспект | Нормализация | Денормализация |
|:---|:---|:---|
| **Чтение (SELECT)** | JOIN'ы, медленнее | Быстрее |
| **Запись (UPDATE/DELETE)** | 1 строка, быстрее | Множество строк, медленнее |
| **Объём данных** | Меньше | Больше (дублирование) |
| **Аномалии** | Нет | Возможны |
| **Когда** | По умолчанию | Точечно под частые запросы |

**Правило:** нормализуй всегда, денормализуй осознанно, когда замеры (`EXPLAIN ANALYZE`, Глава 7) показывают, что JOIN'ы — реальное узкое место.

---

## 11.8 Связи: 1:1, 1:N, N:M, иерархии

После нормализации данные разнесены по таблицам. Теперь нужно **связать** их правильно. Тип связи определяет, куда вставить внешний ключ (FK), нужна ли промежуточная таблица, и как JOIN'ить.

### Связь 1:1 — один к одному

**Определение:** одной строке в таблице A соответствует **не более одной** строки в таблице B.

**Пример:** пользователь и его профиль с расширенными данными.

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email TEXT UNIQUE NOT NULL
);

CREATE TABLE user_profiles (
    user_id BIGINT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    bio TEXT,
    avatar_path TEXT,
    settings JSONB
);
```

**Где разместить FK:** в **любой** из таблиц, но обычно в той, которая **реже используется** — чтобы не тянуть лишние колонки в частые запросы.

**Зачем нужна 1:1:**

1. **Разделение горячих и холодных данных:**

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,       -- горячие: вход, поиск
    created_at TIMESTAMPTZ
);

CREATE TABLE user_profiles (
    user_id BIGINT PRIMARY KEY REFERENCES users(id),
    bio TEXT,                         -- холодные: редко читают
    avatar_path TEXT,
    settings JSONB
);
-- SELECT email FROM users — не тянет bio/avatar (Глава 2: страницы)
```

2. **Изоляция данных под разные права доступа:**

```sql
-- Основная таблица — доступна всем:
CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    name TEXT
);

-- Отдельная таблица с зарплатой — доступ только HR:
CREATE TABLE employee_salaries (
    employee_id BIGINT PRIMARY KEY REFERENCES employees(id),
    salary NUMERIC
);
```

3. **Эволюция схемы без блокировки основной таблицы:**

```sql
-- Добавляем новые поля — не трогаем users, создаём новую таблицу:
CREATE TABLE user_preferences (
    user_id BIGINT PRIMARY KEY REFERENCES users(id),
    theme TEXT,
    notifications JSONB
);
```

### Связь 1:N — один ко многим

**Определение:** одной строке в таблице A соответствует **много** строк в таблице B.

**Пример:** клиент и его заказы.

```sql
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    name TEXT
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT REFERENCES customers(id),  -- FK на стороне «много»
    total NUMERIC,
    created_at TIMESTAMPTZ
);

-- ВАЖНО: индекс на FK-колонку! (Глава 3: REFERENCES не создаёт индекс)
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
```

**Ключевое правило:** FK всегда на стороне **«много»** (N). В таблице `orders` — `customer_id`, а не наоборот.

**Проверь себя:** если бы FK был на стороне «один» (в `customers`), то как хранить несколько заказов одного клиента? В массиве — это нарушение 1НФ. Поэтому FK на стороне «много».

### Связь N:M — многие ко многим

**Определение:** строке в A соответствует много строк в B, **и наоборот**.

**Пример:** заказы и теги (из подглавы 11.2).

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY
);

CREATE TABLE tags (
    id BIGSERIAL PRIMARY KEY,
    name TEXT UNIQUE NOT NULL
);

-- Промежуточная таблица:
CREATE TABLE order_tags (
    order_id BIGINT REFERENCES orders(id) ON DELETE CASCADE,
    tag_id BIGINT REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (order_id, tag_id)  -- составной PK
);
```

**Правила для N:M:**

1. **Всегда** нужна промежуточная таблица.
2. Составной PK: `(order_id, tag_id)` — гарантирует уникальность пары.
3. Индексы: составной PK покрывает `WHERE order_id = ?`, но для `WHERE tag_id = ?` нужен отдельный индекс:

```sql
-- Для запроса «все заказы с тегом X»:
CREATE INDEX idx_order_tags_tag_id ON order_tags(tag_id);
```

4. FK с `ON DELETE CASCADE` — чтобы при удалении заказа или тега связи удалялись автоматически.

### Связь с атрибутами в N:M

Иногда промежуточная таблица хранит **дополнительные данные** о связи:

```sql
CREATE TABLE order_items (
    order_id BIGINT REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT REFERENCES products(id),
    quantity INT NOT NULL,           -- атрибут связи
    price_at_time NUMERIC NOT NULL,  -- цена на момент заказа
    PRIMARY KEY (order_id, product_id)
);
```

`order_items` — это N:M между заказами и товарами, но с **атрибутами связи**: `quantity`, `price_at_time`.

### Иерархии: дерево внутри таблицы

**Определение:** строки таблицы ссылаются на другие строки **той же таблицы**.

**Пример:** сотрудники и их руководители.

```sql
CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    manager_id BIGINT REFERENCES employees(id)  -- ссылка на себя
);

CREATE INDEX idx_employees_manager_id ON employees(manager_id);
```

**Типичные запросы:**

```sql
-- Найти начальника:
SELECT m.*
FROM employees e
JOIN employees m ON m.id = e.manager_id
WHERE e.id = 42;

-- Найти всех подчинённых (один уровень):
SELECT * FROM employees WHERE manager_id = 42;

-- Найти всех подчинённых (все уровни) — рекурсивный CTE (Глава 13):
WITH RECURSIVE subordinates AS (
    SELECT id, name, manager_id FROM employees WHERE id = 42
    UNION ALL
    SELECT e.id, e.name, e.manager_id
    FROM employees e
    JOIN subordinates s ON s.id = e.manager_id
)
SELECT * FROM subordinates;
```

**Проблема:** рекурсия на больших деревьях медленная. Для глубоких иерархий используют паттерны:
- **Materialized Path** — хранить путь: `'1/42/99/'`
- **Nested Sets** — хранить `left_id`, `right_id`
- **Adjacency List + рекурсивные CTE** — самый простой, разберём в главе 12 (Паттерны)

### Сводная таблица связей

| Связь | Где FK | Промежуточная таблица | Пример |
|:---|:---|:---|:---|
| **1:1** | В любой, обычно в реже используемой | Нет | `users` ↔ `user_profiles` |
| **1:N** | На стороне «много» | Нет | `customers` → `orders` |
| **N:M** | В промежуточной | Да | `orders` ↔ `tags` через `order_tags` |
| **Иерархия** | В той же таблице | Нет | `employees.manager_id` → `employees.id` |

### «Бизнес через год» и связи

Бизнес говорит: «Заказ может быть привязан к нескольким складам, и у каждого склада свой статус».

**Было 1:N:**

```sql
orders (id, warehouse_id)  -- один склад на заказ
```

**Стало N:M с атрибутами:**

```sql
CREATE TABLE order_warehouses (
    order_id BIGINT REFERENCES orders(id),
    warehouse_id BIGINT REFERENCES warehouses(id),
    status TEXT,                -- атрибут связи: 'pending', 'shipped'
    PRIMARY KEY (order_id, warehouse_id)
);
-- Бизнес-требование выполнено: заказ на нескольких складах, у каждого свой статус.
```

### Итог подглавы 11.8

- **1:1** — FK в любой таблице, для разделения горячих/холодных данных.
- **1:N** — FK на стороне «много», индекс обязателен.
- **N:M** — промежуточная таблица с составным PK, индексы на обе FK-колонки.
- **Иерархия** — FK на саму себя, запросы через рекурсивный CTE.
- **Атрибуты связи** — в промежуточной таблице N:M.

---

## 11.9 Выбор первичного ключа: суррогатный vs естественный

После того как таблицы разнесены по нормальным формам, встаёт вопрос: **что взять первичным ключом?**

### Два типа ключей

**Естественный ключ** — значение, которое существует в предметной области:

```sql
-- email пользователя, номер паспорта, ISBN книги, код страны:
CREATE TABLE users (
    email TEXT PRIMARY KEY,  -- email — естественный ключ
    name TEXT
);

CREATE TABLE countries (
    iso_code CHAR(2) PRIMARY KEY,  -- 'RU', 'US' — естественный
    name TEXT
);
```

**Суррогатный ключ** — искусственное значение, созданное только для идентификации:

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,  -- суррогатный: не имеет смысла в бизнесе
    email TEXT UNIQUE NOT NULL
);

CREATE TABLE countries (
    id BIGSERIAL PRIMARY KEY,  -- суррогатный
    iso_code CHAR(2) UNIQUE NOT NULL
);
```

### Сравнение

| Аспект | Суррогатный (`id BIGSERIAL`) | Естественный (`email TEXT PK`) |
|:---|:---|:---|
| **Размер** | 8 байт (bigint) | 30+ байт (email) |
| **Индекс PK** | Компактный, быстрый | Больше (Глава 3: B-tree по тексту) |
| **JOIN'ы** | По числу — быстрее | По тексту — медленнее |
| **Изменение ключа** | Никогда не меняется | Может (email сменили → обновить все FK) |
| **Вставка** | В конец индекса (Глава 3) | В середину (email не монотонный) |
| **Бизнес-смысл** | Нет | Есть |

### Влияние размера ключа на индексы и JOIN'ы

Вспомни главу 3: B-tree хранит ключ + TID. Чем больше ключ — тем меньше ключей на страницу, тем выше дерево.

```
PK по BIGINT:
  Ключ: 8 байт + TID 6 байт = 14 байт
  Ключей на страницу: ~580

PK по TEXT (email ~30 байт):
  Ключ: 30 байт + TID 6 байт = 36 байт
  Ключей на страницу: ~220

Высота B-tree для 1 млн строк:
  BIGINT: 2 уровня
  TEXT:   3 уровня

Каждый JOIN по TEXT-PK читает на 1 страницу больше.
При 1000 JOIN'ов в секунду — лишние 1000 чтений.
```

**Суррогатный ключ — меньше и быстрее.**

### Когда естественный ключ оправдан

**1. Справочники с неизменяемым кодом:**

```sql
CREATE TABLE countries (
    iso_code CHAR(2) PRIMARY KEY,  -- 'RU' никогда не изменится
    name TEXT NOT NULL
);

CREATE TABLE currencies (
    iso_code CHAR(3) PRIMARY KEY,  -- 'USD', 'EUR'
    name TEXT
);
```

- Ключ стабилен (ISO-стандарт).
- Ключ компактнее суррогатного (2-3 байта против 8).
- JOIN'ы читаемы: `WHERE country_code = 'RU'` вместо `WHERE country_id = 42`.

**2. Таблицы-словари, где ключ — само значение:**

```sql
CREATE TABLE tags (
    name TEXT PRIMARY KEY,  -- сам тег — ключ
    created_at TIMESTAMPTZ
);

-- Использование:
INSERT INTO tags (name) VALUES ('golang') ON CONFLICT (name) DO NOTHING;
-- Не нужен суррогатный id — тег и есть идентификатор.
```

**3. Таблицы, где нет смысла в суррогате:**

```sql
CREATE TABLE user_roles (
    user_id BIGINT REFERENCES users(id),
    role TEXT,  -- 'admin', 'moderator', 'user'
    PRIMARY KEY (user_id, role)
);
-- role — естественная часть составного ключа.
```

### Когда суррогатный ключ обязателен

**1. Ключ может измениться:**

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,   -- суррогатный
    email TEXT UNIQUE NOT NULL  -- email может смениться, но id — никогда
);
-- Заказы ссылаются на users.id, а не на users.email.
-- Смена email = UPDATE в одной таблице. FK не тронуты.
```

Если бы FK ссылались на `email` — смена email требовала бы обновить **все** дочерние таблицы:

```sql
-- ❌ PK по email:
UPDATE orders SET user_email = 'new@mail.com' WHERE user_email = 'old@mail.com';
-- 10 000 строк. Блокировки. WAL. VACUUM.
```

**2. Ключ большой:**

```sql
CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,      -- суррогатный
    content_hash TEXT UNIQUE NOT NULL  -- 64 символа SHA-256
);
-- PK по content_hash сделал бы все FK по TEXT(64) — ужас.
```

**3. Нет очевидного уникального атрибута:**

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,  -- у заказа нет «естественного» ключа
    customer_id BIGINT,
    ...
);
-- Номер заказа может быть, но он не всегда с самого начала.
```

### Компромисс: суррогатный PK + UNIQUE на естественный

**Рекомендуемый паттерн для большинства таблиц:**

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,      -- суррогатный: для FK и JOIN'ов
    email TEXT UNIQUE NOT NULL,    -- естественный: для бизнес-поиска и целостности
    phone TEXT UNIQUE
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,       -- суррогатный
    order_number TEXT UNIQUE NOT NULL,  -- естественный: для пользователей
    customer_id BIGINT REFERENCES users(id),
    ...
);
```

**Что даёт этот паттерн:**

- FK внутри базы — по компактному `id` (быстрые JOIN'ы, маленькие индексы).
- Поиск из приложения — по естественному (`WHERE email = ?`, `WHERE order_number = ?`).
- `UNIQUE` на естественный защищает целостность (НФБК, подглава 11.6).
- `UNIQUE` автоматически создаёт индекс (Глава 3) — поиск быстрый.

### Влияние на вставку: монотонность ключа

Вспомни главу 3 (fillfactor) и главу 4 (BRIN): **монотонный** ключ вставляется в конец B-tree, случайный — в середину.

```
BIGSERIAL: 1, 2, 3, 4, ... — монотонный, вставка в конец. Быстро.
UUID v4: случайный — вставка в середину B-tree, расщепления страниц. Медленнее.

Email TEXT: не монотонный — вставка в середину.
```

**Это ещё один аргумент за суррогатный BIGSERIAL** — вставка быстрее, индекс плотнее.

### «Бизнес через год» и выбор ключа

Бизнес говорит: «Пользователи хотят менять email».

**Если PK = email:**

```sql
UPDATE users SET email = 'new@mail.com' WHERE email = 'old@mail.com';
-- Если на email ссылаются FK в 10 таблицах — кошмар:
UPDATE orders SET user_email = 'new@mail.com' WHERE user_email = 'old@mail.com';
UPDATE sessions SET user_email = 'new@mail.com' WHERE user_email = 'old@mail.com';
-- ...
```

**Если PK = id, UNIQUE = email:**

```sql
UPDATE users SET email = 'new@mail.com' WHERE id = 42;
-- Одна строка. FK не тронуты.
```

### Итог подглавы 11.9

**Правило по умолчанию:**

```sql
CREATE TABLE <name> (
    id BIGSERIAL PRIMARY KEY,          -- суррогатный, всегда
    <естественный_ключ> TEXT UNIQUE NOT NULL,  -- если бизнес-уникальность есть
    ...
);
```

**Исключения для естественного PK:**

- Справочники с ISO-кодами (`countries.iso_code`)
- Словари, где значение = ключ (`tags.name`)
- Составные ключи в N:M (`PRIMARY KEY (order_id, tag_id)`)

**Никогда не используй как PK:**

- Email (может смениться)
- Телефон (может смениться)
- Номер паспорта (может смениться)
- Любой атрибут, который бизнес может изменить

---

## 11.10 Именование таблиц и колонок: конвенции, которые экономят годы

### Зачем вообще думать об именах

Когда таблиц 5 — можно назвать как угодно. Когда таблиц 200 и 20 разработчиков — плохие имена превращаются в **ежедневную боль**:

- `users`, `Users`, `tbl_users`, `user_table` — одна и та же сущность, четыре имени.
- `created_at`, `creation_date`, `dt_created`, `created` — одна и та же логика, четыре имени.
- `id`, `user_id`, `userId`, `userID` — один и тот же FK, четыре стиля.

**Хорошие конвенции:**
- Ускоряют чтение кода (глаза ищут знакомый паттерн).
- Упрощают JOIN'ы (всегда понятно, что `orders.user_id` ссылается на `users.id`).
- Уменьшают ошибки (не нужно гадать, как называется колонка).
- Облегчают генерацию кода, миграции, ORM.

### Конвенция 1: множественное число для таблиц

Таблица хранит **много** сущностей → имя во множественном числе:

```sql
CREATE TABLE users (...);       -- ✅
CREATE TABLE orders (...);      -- ✅
CREATE TABLE order_items (...); -- ✅

CREATE TABLE user (...);        -- ❌ непонятно, одна или много
CREATE TABLE order (...);       -- ❌ ORDER — зарезервированное слово SQL!
```

**Почему множественное:** таблица — это коллекция строк. `SELECT * FROM users` читается естественно.

**Исключение:** некоторые команды используют единственное число. Главное — **единообразие внутри проекта**.

### Конвенция 2: snake_case для колонок

PostgreSQL **складывает некавыченные имена в нижний регистр**:

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ,     -- ✅ snake_case
    first_name TEXT              -- ✅ snake_case
);

-- ❌ CamelCase требует кавычек:
CREATE TABLE users (
    "id" BIGSERIAL PRIMARY KEY,
    "createdAt" TIMESTAMPTZ,    -- придётся ВЕЗДЕ писать в кавычках
    "firstName" TEXT
);

SELECT "firstName" FROM users WHERE "createdAt" > ...;  -- боль
```

**Правило:** только `snake_case`, только нижний регистр, без кавычек.

### Конвенция 3: PK всегда `id`

Каждая таблица имеет суррогатный `id` (подглава 11.9):

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY  -- всегда id, не user_id в таблице users
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY
);
```

**Почему `id`, а не `user_id` в таблице users:** `user_id` внутри `users` — это тавтология. `users.id` и так понятно.

**Исключение:** составные PK в N:M:

```sql
CREATE TABLE order_tags (
    order_id BIGINT NOT NULL,
    tag_id BIGINT NOT NULL,
    PRIMARY KEY (order_id, tag_id)  -- составной, не id
);
```

### Конвенция 4: FK — `<таблица_в_единственном>_id`

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT REFERENCES customers(id),  -- ✅
    product_id BIGINT REFERENCES products(id)      -- ✅
);

-- ❌ Плохо:
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    cid BIGINT REFERENCES customers(id),  -- что за cid?
    p BIGINT REFERENCES products(id)       -- что за p?
);
```

**Правило:** имя FK = имя таблицы в единственном числе + `_id`.

**Нюанс для само-ссылок:**

```sql
CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    manager_id BIGINT REFERENCES employees(id)  -- ✅ manager_id, а не employee_id
);
-- «manager» — роль, а не таблица.
```

### Конвенция 5: временные метки — `_at`

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT now(),  -- ✅
    updated_at TIMESTAMPTZ,                -- ✅
    deleted_at TIMESTAMPTZ                 -- ✅ (для soft delete)
);
```

**Стандартные имена:**

| Колонка | Что означает |
|:---|:---|
| `created_at` | Когда создана запись |
| `updated_at` | Когда обновлена |
| `deleted_at` | Когда удалена (soft delete, NULL = не удалена) |
| `expired_at` | Когда истекает (кэш, сессия) |

### Конвенция 6: булевы колонки — `is_`, `has_`, `can_`

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    is_active BOOLEAN DEFAULT true,   -- ✅
    has_verified_email BOOLEAN,       -- ✅
    can_moderate BOOLEAN              -- ✅
);
```

**Читается как вопрос:** `is_active` → «активен?» → true/false.

**❌ Плохо:** `active`, `verified` — не ясно, что это булево.

### Конвенция 7: таблицы связей — `<табл1>_<табл2>`

Для N:M таблиц используй **единственное + множественное** (`order_tags`, `user_roles`):

```sql
-- N:M между orders и tags:
CREATE TABLE order_tags (...);   -- «теги заказа»

-- N:M между users и roles:
CREATE TABLE user_roles (...);   -- «роли пользователя»
```

**Почему:**
- Читается естественно: «теги заказа», «роли пользователя».
- Согласуется с ORM-конвенциями (Rails, Django, Prisma).
- Отличает таблицу связей от обычной таблицы: `tags` (сущности) vs `order_tags` (связи).

**Альтернатива:** строго `order_tag`, `user_role` (ед. + ед.). Главное — **единообразие**.

### Конвенция 8: не зарезервированные слова

PostgreSQL имеет зарезервированные слова:

```sql
-- ❌ order, user, group — зарезервированы:
CREATE TABLE "order" (...);  -- придётся в кавычках везде

-- ✅ orders, users, groups — множественное число спасает
CREATE TABLE orders (...);
CREATE TABLE users (...);
```

### Схемы: именование и паттерны

**Что такое схема** — мы разбирали в главе 2 (подглава 2.4): логическое пространство имён, `pg_namespace`, `search_path`.

**Паттерны использования схем:**

**Паттерн 1: `public` для всего — антипаттерн для больших проектов**

```sql
-- Всё в public:
CREATE TABLE users (...);
CREATE TABLE orders (...);
CREATE TABLE audit_log (...);
CREATE TABLE migrations (...);
-- 200+ таблиц в одной куче.
```

**Паттерн 2: Разделение по модулям / доменам**

```sql
CREATE SCHEMA shop;       -- таблицы магазина
CREATE SCHEMA billing;    -- платежи
CREATE SCHEMA audit;      -- история изменений
CREATE SCHEMA analytics;  -- витрины

-- Таблицы:
shop.users, shop.orders
billing.invoices, billing.transactions
audit.change_log
analytics.daily_sales
```

**Паттерн 3: Разделение по правам доступа**

```sql
CREATE SCHEMA public;   -- доступно всем
CREATE SCHEMA hr;       -- только HR-роль
CREATE SCHEMA finance;  -- только финансистам

-- Разные роли → разный доступ к схемам.
```

**Паттерн 4: Отдельная схема для миграций**

```sql
CREATE SCHEMA migrations;
CREATE TABLE migrations.schema_version (...);
```

**Именование схем:**

```sql
-- ✅ snake_case, нижний регистр, единственное число:
CREATE SCHEMA shop;
CREATE SCHEMA billing;
CREATE SCHEMA audit;

-- ❌ Плохо:
CREATE SCHEMA MySchema;     -- заглавные → кавычки
CREATE SCHEMA myschema;     -- бессмысленно
CREATE SCHEMA test;         -- бессмысленно
```

**Правило:** схема — это пространство, а не коллекция. Единственное число: `billing`, а не `billings`.

### Полный пример

```sql
CREATE SCHEMA shop;

CREATE TABLE shop.customers (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    phone TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ
);

CREATE TABLE shop.orders (
    id BIGSERIAL PRIMARY KEY,
    order_number TEXT UNIQUE NOT NULL,
    customer_id BIGINT NOT NULL REFERENCES shop.customers(id),
    total NUMERIC(12,2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE shop.order_items (
    order_id BIGINT NOT NULL REFERENCES shop.orders(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES shop.products(id),
    quantity INT NOT NULL,
    price_at_time NUMERIC(12,2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

**Что видно сразу:**

- `orders.customer_id` → `customers.id` — связь очевидна.
- `order_items.order_id` → `orders.id` — очевидно.
- `created_at` везде одно и то же.
- Все таблицы во множественном числе, snake_case, PK — `id`.
- Схема `shop` отделяет таблицы магазина от `billing`, `audit`.

### «Бизнес через год» и именование

Бизнес говорит: «У нас теперь не `users`, а `customers`» (решили, что пользователи — это клиенты).

**Плохой сценарий — таблица в единственном числе:**

```sql
CREATE TABLE user (...);  -- было, единственное число
-- Через год бизнес: «переименуйте user в customer»
ALTER TABLE user RENAME TO customer;
-- Проблемы:
--   FK: user_id в других таблицах → нужно переименовывать колонки
--   Индексы: имена не обновляются автоматически
--   Код, ORM, отчёты — всё ссылается на старое имя
```

**Хороший сценарий — таблица сразу с правильным именем:**

```sql
CREATE TABLE customers (...);  -- сразу так, как бизнес называет
-- Никаких переименований не нужно.
```

**Или, если всё же пришлось переименовать:**

```sql
-- Само переименование таблицы — быстро:
ALTER TABLE users RENAME TO customers;

-- Но FK-колонки остаются с старыми именами:
-- orders.user_id → нужно переименовать в customer_id:
ALTER TABLE orders RENAME COLUMN user_id TO customer_id;

-- Плюс обновить constraint'ы, индексы, код, ORM.
-- Это часы работы и риск ошибок.
```

**Вывод:** с самого начала называй таблицу так, как её называет бизнес. Если бизнес говорит «клиенты» — таблица `customers`, а не `users` и не `user`.

### Итог подглавы 11.10

| Правило | Пример |
|:---|:---|
| Таблицы во множественном числе, snake_case | `users`, `order_items` |
| PK всегда `id` | `users.id`, `orders.id` |
| FK — `<таблица>_id` | `orders.customer_id` |
| Временные метки — `_at` | `created_at`, `updated_at` |
| Булевы — `is_`, `has_`, `can_` | `is_active`, `has_verified_email` |
| N:M — `<табл1>_<табл2>` (ед. + мн.) | `order_tags`, `user_roles` |
| Схемы — по модулям, snake_case, ед.ч. | `shop`, `billing`, `audit` |
| Без кавычек, без CamelCase, без зарезервированных слов | — |

**Главное:** единообразие внутри проекта важнее, чем конкретные правила. Выбери конвенцию и следуй ей везде.

---

## 11.11 Антипаттерны проектирования: как не надо делать

Нормализация — это теория. Антипаттерны — это **практика**, где теорию игнорируют, и получают проблемы. Разберём четыре классических.

### Антипаттерн 1: EAV (Entity-Attribute-Value)

**Идея:** вместо обычных колонок хранить **все атрибуты в одной таблице** как пары «ключ-значение».

```sql
CREATE TABLE eav_attributes (
    entity_id BIGINT,      -- к какой сущности относится
    attr_name TEXT,        -- имя атрибута
    attr_value TEXT,       -- значение
    PRIMARY KEY (entity_id, attr_name)
);

-- Вместо users(id, name, email, age):
INSERT INTO eav_attributes VALUES
    (1, 'name', 'Alice'),
    (1, 'email', 'alice@mail.com'),
    (1, 'age', '30'),
    (2, 'name', 'Bob'),
    (2, 'email', 'bob@mail.com');
```

**Почему это плохо:**

1. **Нет типобезопасности:**

```sql
-- age хранится как текст. Можно вставить 'тридцать':
INSERT INTO eav_attributes VALUES (3, 'age', 'тридцать');
-- БД не проверит, что age — число.
```

2. **Запросы превращаются в кошмар:**

```sql
-- Найти пользователей старше 25:
SELECT entity_id
FROM eav_attributes
WHERE attr_name = 'age'
  AND attr_value::INT > 25;  -- CAST из текста в число!
-- Без индекса по attr_value. Seq Scan.
```

3. **Нет целостности:**

```sql
-- Никто не гарантирует, что у пользователя есть name и email.
-- Можно вставить только age без name.
```

4. **JOIN'ы для сбора сущности:**

```sql
-- Получить пользователя:
SELECT 
    n.attr_value AS name,
    e.attr_value AS email,
    a.attr_value AS age
FROM eav_attributes n
FULL JOIN eav_attributes e ON e.entity_id = n.entity_id AND e.attr_name = 'email'
FULL JOIN eav_attributes a ON a.entity_id = n.entity_id AND a.attr_name = 'age'
WHERE n.entity_id = 1 AND n.attr_name = 'name';
-- Ужас вместо SELECT * FROM users WHERE id = 1.
```

**Когда EAV оправдан:** почти никогда. Если нужна гибкость — используй JSONB (Глава 10), который хотя бы поддерживает индексы GIN.

**Бизнес через год:** «Добавьте колонку phone». С EAV — просто вставляем новые пары. С обычной таблицей — `ALTER TABLE users ADD COLUMN phone TEXT`. Но EAV-«гибкость» оборачивается отсутствием типов, индексов и целостности.

---

### Антипаттерн 2: «Таблица-свалка»

**Идея:** одна таблица для всего: пользователи, заказы, товары, теги — всё вперемешку.

```sql
CREATE TABLE everything (
    id BIGSERIAL PRIMARY KEY,
    type TEXT NOT NULL,      -- 'user', 'order', 'product'
    name TEXT,               -- для user, product
    email TEXT,              -- для user
    total NUMERIC,           -- для order
    price NUMERIC,           -- для product
    tags TEXT                -- для всего
);

INSERT INTO everything (type, name, email) VALUES ('user', 'Alice', 'alice@mail.com');
INSERT INTO everything (type, total) VALUES ('order', 1000);
INSERT INTO everything (type, name, price) VALUES ('product', 'iPhone', 500);
```

**Почему это плохо:**

1. **Колонки пустуют:** для строки `type = 'user'` колонки `total`, `price` — NULL. 80% ячеек пустые.

2. **Нет FK:** невозможно сделать `order.user_id REFERENCES users(id)` — пользователи и заказы в одной таблице.

3. **Индексы не работают:**

```sql
CREATE INDEX idx_email ON everything(email) WHERE type = 'user';
-- Частичный индекс (Глава 3) — но это костыль.
```

4. **SELECT страшный:**

```sql
SELECT * FROM everything WHERE type = 'user' AND email = 'alice@mail.com';
-- Постоянно фильтровать по type.
```

**Бизнес через год:** «Добавьте поле `sku` только для товаров». С таблицей-свалкой — ещё одна колонка, которая будет NULL для users и orders. С нормальными таблицами — `ALTER TABLE products ADD COLUMN sku TEXT`.

---

### Антипаттерн 3: CSV в колонке (нарушение 1НФ)

Мы разбирали это в подглаве 11.2. Повторю кратко:

```sql
-- ❌ Теги через запятую:
tags TEXT  -- 'electronics,gadget,sale'

-- ❌ Список ID через запятую:
product_ids TEXT  -- '42,100,500'
```

**Проблемы:** нет индексов, нет FK, нет группировки, `LIKE '%...%'` вместо точного поиска.

**Решение:** отдельная таблица связей N:M (подглава 11.8).

---

### Антипаттерн 4: Множественные колонки с номерами

```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    image_url_1 TEXT,
    image_url_2 TEXT,
    image_url_3 TEXT,
    -- А если будет 4 фото? 5? 10?
    price_1 NUMERIC,
    price_2 NUMERIC  -- оптовая? розничная?
);
```

**Почему это плохо:**

1. **Ограниченность:** что если понадобится 4 фото? Нужен `ALTER TABLE ADD COLUMN image_url_4`.

2. **Запросы страшные:**

```sql
-- Найти продукты с фото:
SELECT * FROM products
WHERE image_url_1 IS NOT NULL
   OR image_url_2 IS NOT NULL
   OR image_url_3 IS NOT NULL;
-- Вместо JOIN по product_images.
```

3. **Это нарушение 1НФ:** массив, замаскированный под колонки.

**Решение:**

```sql
CREATE TABLE product_images (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT REFERENCES products(id) ON DELETE CASCADE,
    url TEXT NOT NULL,
    position INT NOT NULL  -- порядок отображения
);

-- Сколько угодно фото. Один запрос:
SELECT url FROM product_images WHERE product_id = 42 ORDER BY position;
```

---

### Сводная таблица антипаттернов

| Антипаттерн | Суть проблемы | Правильное решение |
|:---|:---|:---|
| **EAV** | Всё в «ключ-значение» | Обычные колонки или JSONB |
| **Таблица-свалка** | Все сущности в одной таблице с `type` | Отдельные таблицы для каждой сущности |
| **CSV в колонке** | Множество значений в одной ячейке | Отдельная таблица связей N:M |
| **Колонки с номерами** | `image_1`, `image_2`, `image_3` | Отдельная таблица `product_images` |

---

### «Бизнес через год» и антипаттерны

Бизнес говорит: «Хочу для товара хранить от 1 до 20 фотографий».

**С колонками `image_url_1` ... `image_url_3`:**

```sql
-- Нужно 17 новых колонок:
ALTER TABLE products ADD COLUMN image_url_4 TEXT;
ALTER TABLE products ADD COLUMN image_url_5 TEXT;
-- ... 17 раз
-- И переписывать все запросы.
```

**С таблицей `product_images`:**

```sql
-- Ничего не меняем. INSERT новых строк:
INSERT INTO product_images (product_id, url, position) VALUES (42, 'photo_20.jpg', 20);
-- Всё работает.
```

---

## 11.12 История изменений (audit log)

### Зачем хранить историю

Бизнес рано или поздно спрашивает:

- «Кто изменил цену товара?»
- «Когда клиент сменил email?»
- «Какая была цена до скидки?»
- «Кто удалил заказ?»

Если история не хранится — ответа нет. **Audit log** — это таблица (или набор таблиц), куда записывается **каждое изменение** данных.

### Требования к хорошему audit log

1. **Кто** — какой пользователь сделал изменение.
2. **Когда** — точное время (`timestamptz`).
3. **Что** — какая таблица и какая строка.
4. **Какое изменение** — старое значение → новое значение.
5. **Не влияет на основную таблицу** — запись истории не должна блокировать `UPDATE`/`DELETE`.

### Подход 1: Триггеры (встроенный механизм)

PostgreSQL поддерживает триггеры (подробно разберём в главе 24). Для audit log создаётся:

```sql
-- Таблица истории:
CREATE SCHEMA audit;

CREATE TABLE audit.change_log (
    id BIGSERIAL PRIMARY KEY,
    table_name TEXT NOT NULL,           -- 'users', 'orders'
    row_id BIGINT NOT NULL,             -- id изменённой строки
    action TEXT NOT NULL,               -- 'INSERT', 'UPDATE', 'DELETE'
    old_data JSONB,                     -- старая версия (для UPDATE/DELETE)
    new_data JSONB,                     -- новая версия (для INSERT/UPDATE)
    changed_by TEXT,                    -- кто (current_user)
    changed_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_change_log_table_row ON audit.change_log(table_name, row_id, changed_at);
```

**Триггерная функция:**

```sql
CREATE OR REPLACE FUNCTION audit.log_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit.change_log (table_name, row_id, action, new_data, changed_by)
        VALUES (TG_TABLE_NAME, NEW.id, 'INSERT', to_jsonb(NEW), current_user);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit.change_log (table_name, row_id, action, old_data, new_data, changed_by)
        VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW), current_user);
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit.change_log (table_name, row_id, action, old_data, changed_by)
        VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', to_jsonb(OLD), current_user);
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

**Привязка триггера к таблицам:**

```sql
CREATE TRIGGER users_audit
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit.log_changes();

CREATE TRIGGER orders_audit
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION audit.log_changes();
```

**Плюсы:**
- Работает на уровне БД — приложение не может забыть записать историю.
- Не нужно менять код приложения.
- Ловит изменения из любых источников (API, ручные UPDATE, миграции).

**Минусы:**
- Каждый `INSERT`/`UPDATE`/`DELETE` пишет **дополнительную строку** в `audit.change_log` → замедление записи.
- WAL растёт быстрее (Глава 1): основной запрос + запись в аудит — оба логируются.
- VACUUM работает больше (Глава 6): аудит-таблица тоже растёт.

**Численный пример:**

```
Основная таблица users: 1 000 000 строк.
За день: 10 000 UPDATE.

Без аудита: 10 000 новых версий, 10 000 WAL-записей.
С аудитом: +10 000 строк в change_log, +10 000 WAL-записей.

Итого: двойной объём WAL.
```

**Когда использовать:** когда важна целостность истории (финансы, медицина, юридические данные).

---

### Подход 2: Soft delete (мягкое удаление)

Вместо физического `DELETE` — ставим `deleted_at`:

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    deleted_at TIMESTAMPTZ,  -- NULL = не удалён
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ
);

-- «Удаление»:
UPDATE users SET deleted_at = now() WHERE id = 42;

-- Все запросы учитывают deleted_at:
SELECT * FROM users WHERE id = 42 AND deleted_at IS NULL;

-- Частичный индекс (Глава 3) для быстрого поиска:
CREATE INDEX idx_users_active ON users(id) WHERE deleted_at IS NULL;
```

**Плюсы:**
- Данные не теряются — «удалённую» строку можно восстановить.
- Запросы на чтение не блокируются.
- Нет мёртвых кортежей (Глава 6) — `UPDATE` вместо `DELETE`.

**Минусы:**
- Таблица растёт (неудалённые строки занимают место).
- Все запросы должны помнить про `deleted_at IS NULL`.
- Нет «кто удалил» — только «когда удалил».

**Когда использовать:** когда удаление редкое, а восстановление важно.

---

### Подход 3: Версионирование строк (история значений)

Хранить **все версии** строки в отдельной таблице:

```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ
);

CREATE TABLE product_price_history (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT REFERENCES products(id),
    price NUMERIC NOT NULL,
    changed_at TIMESTAMPTZ DEFAULT now(),
    changed_by TEXT
);

-- При изменении цены:
UPDATE products SET price = 900, updated_at = now() WHERE id = 42;
INSERT INTO product_price_history (product_id, price, changed_by)
VALUES (42, 900, current_user);
```

**Плюсы:**
- Полная история изменений конкретного атрибута.
- Удобные запросы: «какая была цена 1 января?»

**Минусы:**
- Нужно вручную поддерживать согласованность (UPDATE + INSERT в одной транзакции).
- Таблица истории растёт.

**Когда использовать:** когда важна история конкретных полей (цена, статус, баланс).

---

### Подход 4: CDC (Change Data Capture) через логическую репликацию

Для **больших объёмов** изменений используют **логическую репликацию** (глава 19) — PostgreSQL сам отдаёт поток изменений в Kafka/ClickHouse:

```
PostgreSQL → logical replication slot → Debezium → Kafka → ClickHouse
```

**Плюсы:**
- Не замедляет основную базу (изменения читаются из WAL).
- Полная история всех изменений.

**Минусы:**
- Сложная инфраструктура (Kafka, Debezium, консьюмеры).
- Не для маленьких проектов.

**Когда использовать:** высоконагруженные системы, аналитика в реальном времени.

---

### Сравнение подходов

| Подход | Замедляет запись | Хранит «кто» | Хранит «старое» | Сложность |
|:---|:---|:---|:---|:---|
| **Триггеры** | Да (в 2 раза) | ✅ | ✅ | Средняя |
| **Soft delete** | Нет | ❌ | ❌ (только deleted_at) | Низкая |
| **Версионирование** | Да (вручную) | ✅ | ✅ | Средняя |
| **CDC / Debezium** | Нет | ✅ | ✅ | Высокая |

---

### «Бизнес через год» и история

Бизнес говорит: «Нужно знать, кто вчера поменял цену товара».

**Если истории нет:**

```sql
-- Данных нет. Ответить невозможно.
-- Придётся срочно внедрять триггеры или CDC.
```

**Если триггеры с самого начала:**

```sql
SELECT changed_by, old_data->>'price' AS old_price,
       new_data->>'price' AS new_price, changed_at
FROM audit.change_log
WHERE table_name = 'products'
  AND row_id = 42
  AND changed_at > now() - interval '1 day'
ORDER BY changed_at;
-- Ответ готов.
```

### Практическое правило

**Для стартапа:** soft delete + версионирование критичных полей (цена, баланс).

**Для продакшена с финансами:** триггеры на ключевые таблицы.

**Для highload:** CDC через Debezium + Kafka.

**Начинай с простого.** История нужна там, где бизнес может спросить «кто и когда». Не пиши аудит на каждую таблицу — это замедлит всё.

---

## 11.13 Проектирование с нуля: пошаговый алгоритм

Все предыдущие подглавы дали инструменты: нормальные формы, связи, ключи, именование, антипаттерны. Теперь — **как собрать всё вместе**, когда перед тобой пустой лист и требование «сделай базу для интернет-магазина».

### Шаг 1: Собери сущности из бизнес-требований

Не пиши SQL. Сначала — **предметная область**.

**Бизнес говорит:**

> «Пользователи регистрируются, делают заказы. У заказа есть товары. Товары принадлежат категориям. У товаров есть теги для поиска. Админ может менять цены.»

**Выписываем сущности (существительные):**

```
Пользователь (User)
Заказ (Order)
Товар (Product)
Категория (Category)
Тег (Tag)
```

**Для каждой сущности — атрибуты (что о ней знаем):**

```
User:     email, phone, имя
Order:    дата, сумма, статус, пользователь
Product:  название, цена, категория, теги
Category: название
Tag:      название
```

### Шаг 2: Определи связи между сущностями

**Задай вопрос для каждой пары:** «Один X может иметь сколько Y?»

```
User → Order:     1:N (один пользователь — много заказов)
Order → Product:  N:M (заказ — много товаров, товар — в многих заказах)
Product → Category: N:1 (товар — одна категория, категория — много товаров)
Product → Tag:    N:M (товар — много тегов, тег — много товаров)
```

**Сводка связей:**

| Связь | Тип |
|:---|:---|
| users → orders | 1:N |
| orders ↔ products | N:M |
| products → categories | N:1 |
| products ↔ tags | N:M |

### Шаг 3: Нарисуй ER-диаграмму

```
users ──< orders >── order_items ──< products >── product_tags ──< tags
                                    │
                                    └──< categories
```

**Обозначения:**
- `──<` — «один ко многим» (FK на стороне «много»)
- `>──<` — N:M через промежуточную таблицу

### Шаг 4: Создай таблицы по нормальным формам

**Таблицы-справочники (родительские, независимые):**

```sql
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE tags (
    id BIGSERIAL PRIMARY KEY,
    name TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

**Таблицы сущностей:**

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    phone TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ
);

CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC(12,2) NOT NULL CHECK (price >= 0),
    category_id BIGINT REFERENCES categories(id),
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_products_category_id ON products(category_id);
```

**Таблица на стороне «много» (1:N):**

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number TEXT UNIQUE NOT NULL,
    user_id BIGINT NOT NULL REFERENCES users(id),
    total NUMERIC(12,2) NOT NULL CHECK (total >= 0),
    status TEXT NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status) WHERE status = 'pending';
```

**Промежуточные таблицы (N:M):**

```sql
CREATE TABLE order_items (
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity INT NOT NULL CHECK (quantity > 0),
    price_at_time NUMERIC(12,2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);

CREATE INDEX idx_order_items_product_id ON order_items(product_id);

CREATE TABLE product_tags (
    product_id BIGINT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    tag_id BIGINT NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, tag_id)
);

CREATE INDEX idx_product_tags_tag_id ON product_tags(tag_id);
```

### Шаг 5: Проверь нормализацию

**Прогони тест из подглав 11.2–11.4:**

```
✅ 1НФ: нет CSV, нет JSONB-массивов, каждая колонка атомарна.
✅ 2НФ: составные ключи только в N:M (order_items, product_tags), частичных зависимостей нет.
✅ 3НФ: неключевые колонки зависят только от PK, транзитивных зависимостей нет.
✅ НФБК: естественные ключи (email, order_number, name) объявлены UNIQUE.
```

### Шаг 6: Проверь связи и индексы

**Для каждой FK — индекс:**

```sql
-- ✅ orders.user_id → idx_orders_user_id
-- ✅ products.category_id → idx_products_category_id
-- ✅ order_items.product_id → idx_order_items_product_id
-- ✅ product_tags.tag_id → idx_product_tags_tag_id
```

**Правило:** каждая FK-колонка должна иметь индекс, если по ней фильтруют или JOIN'ят (Глава 3).

### Шаг 7: Задай constraints

```sql
-- Числовые ограничения:
CHECK (price >= 0)
CHECK (total >= 0)
CHECK (quantity > 0)

-- Уникальность:
UNIQUE (email)
UNIQUE (order_number)
UNIQUE (name) — для categories, tags

-- DEFAULT:
created_at DEFAULT now()
status DEFAULT 'pending'
is_active DEFAULT true
```

### Шаг 8: Подумай о будущем

**Что спросить у бизнеса:**

1. **Будет ли товар в нескольких категориях?** Если да — N:M вместо N:1.
2. **Будет ли у пользователя несколько email?** Если да — отдельная таблица.
3. **Нужна ли история изменения цен?** Если да — таблица истории (подглава 11.12).
4. **Ожидаемый объём?** 10 млн заказов → думай о секционировании (глава 17).

### Шаг 9: Создай схему и таблицы

```sql
CREATE SCHEMA shop;

CREATE TABLE shop.categories (...);
CREATE TABLE shop.users (...);
-- ...
```

### Шаг 10: Прототипируй запросы

**Прежде чем писать код — проверь, что схема отвечает на ключевые вопросы:**

```sql
-- Заказы пользователя:
SELECT * FROM shop.orders WHERE user_id = 42;
-- ✅ Index Scan по idx_orders_user_id

-- Товары в категории:
SELECT * FROM shop.products WHERE category_id = 7;
-- ✅ Index Scan по idx_products_category_id

-- Заказы с тегом:
SELECT o.*
FROM shop.orders o
JOIN shop.order_items oi ON oi.order_id = o.id
JOIN shop.product_tags pt ON pt.product_id = oi.product_id
JOIN shop.tags t ON t.id = pt.tag_id
WHERE t.name = 'electronics';
-- ✅ Index Scan по tags.name + Index Scan по product_tags.tag_id
```

### Чек-лист проектирования

- [ ] Сущности выделены из бизнес-требований
- [ ] Связи определены (1:1, 1:N, N:M)
- [ ] Таблицы в 3НФ (НФБК)
- [ ] PK — суррогатный `id BIGSERIAL`
- [ ] Естественные ключи объявлены `UNIQUE`
- [ ] FK на стороне «много», с индексами
- [ ] N:M через промежуточные таблицы
- [ ] Constraints: CHECK, UNIQUE, DEFAULT
- [ ] Индексы на все FK-колонки
- [ ] Именование по конвенциям (snake_case, множественное число)
- [ ] Схема разделена по модулям
- [ ] Ключевые запросы прототипированы через EXPLAIN (Глава 7)

---

### «Бизнес через год» и пошаговый алгоритм

Бизнес говорит: «Хочу, чтобы товар мог быть в нескольких категориях».

**Если спроектировали N:1 (product.category_id):**

```sql
-- Нужно переделывать:
ALTER TABLE products DROP COLUMN category_id;
CREATE TABLE product_categories (
    product_id BIGINT REFERENCES products(id),
    category_id BIGINT REFERENCES categories(id),
    PRIMARY KEY (product_id, category_id)
);
-- + миграция данных + переписывание запросов.
```

**Если на шаге 8 спросили «а будет ли несколько категорий?» и сделали N:M сразу:**

```sql
-- Уже готово. Бизнес доволен.
```

**Вывод:** главное в пошаговом алгоритме — **шаг 8** (подумай о будущем). Хороший проектировщик отличается не тем, что сделал схему по учебнику, а тем, что **задал правильные вопросы бизнесу до создания таблиц**.

---

## 11.14 Миграция схемы без блокировки

### Что такое миграция

> **Миграция схемы** — изменение структуры базы данных: добавление/удаление колонок, таблиц, индексов, constraints.

**Примеры миграций:**

```sql
ALTER TABLE users ADD COLUMN phone TEXT;              -- добавить колонку
ALTER TABLE users DROP COLUMN phone;                  -- удалить колонку
ALTER TABLE users ALTER COLUMN email TYPE VARCHAR(320); -- изменить тип
CREATE INDEX idx_users_email ON users(email);         -- добавить индекс
DROP INDEX idx_users_email;                           -- удалить индекс
```

### Почему миграции опасны

Вспомни главу 1: PostgreSQL работает со страницами в `shared_buffers`. Вспомни главу 2: таблица — это файл на диске, разбитый на страницы по 8 КБ.

**Некоторые миграции требуют переписывания ВСЕХ страниц таблицы.**

```sql
-- Меняем тип колонки с VARCHAR(255) на VARCHAR(320):
ALTER TABLE users ALTER COLUMN email TYPE VARCHAR(320);
-- PostgreSQL:
--   1. Блокирует таблицу (ACCESS EXCLUSIVE LOCK)
--   2. Создаёт НОВЫЙ файл таблицы
--   3. Копирует каждую строку, конвертируя email в новый тип
--   4. Пересоздаёт все индексы
--   5. Удаляет старый файл
--   6. Снимает блокировку

-- Для таблицы 10 млн строк: минуты или часы.
-- Всё это время: INSERT/UPDATE/DELETE ждут.
```

**Проблема:** бизнес не может остановиться на час ради миграции.

### Как PostgreSQL обрабатывает ALTER TABLE

Разные операции — разная блокировка.

| Операция | Блокирует запись | Переписывает таблицу | Время |
|:---|:---|:---|:---|
| `ADD COLUMN` (без DEFAULT) | Нет | Нет | Мгновенно |
| `ADD COLUMN ... DEFAULT` (PG 11+) | Нет | Нет | Мгновенно |
| `DROP COLUMN` | Нет | Нет | Мгновенно |
| `ALTER COLUMN TYPE` | ✅ Да | ✅ Да | Долго |
| `ADD CONSTRAINT CHECK` | ✅ Да (без NOT VALID) | Нет | Зависит от размера |
| `ADD CONSTRAINT CHECK ... NOT VALID` | Нет | Нет | Мгновенно |
| `CREATE INDEX` | ✅ Да | Нет | Зависит от размера |
| `CREATE INDEX CONCURRENTLY` | Нет | Нет | Дольше, но без блокировки |

### Паттерн безопасной миграции

**Задача:** добавить колонку `phone` в таблицу `users` и заполнить её.

**Шаг 1: Добавляем колонку без блокировки**

```sql
-- Без DEFAULT: мгновенно
ALTER TABLE users ADD COLUMN phone TEXT;

-- С DEFAULT: тоже мгновенно (PostgreSQL 11+)
ALTER TABLE users ADD COLUMN phone TEXT DEFAULT '+7';
```

**Шаг 2: Заполняем данные частями**

```sql
-- Не одним UPDATE на 10 млн строк:
-- ❌ UPDATE users SET phone = '+79160000000' WHERE phone IS NULL;
--    Блокирует таблицу, создаёт миллионы мёртвых кортежей (Глава 6).

-- ✅ Частями (батчами):
UPDATE users SET phone = '+79160000000'
WHERE id IN (SELECT id FROM users WHERE phone IS NULL LIMIT 10000);
-- Повторить много раз, пока не заполним.
```

**Шаг 3: Переключаем приложение на новую колонку**

Приложение начинает читать и писать `phone`.

**Шаг 4: Удаляем старую колонку (если нужно)**

```sql
ALTER TABLE users DROP COLUMN phone_old;
-- Мгновенно (не переписывает таблицу).
```

### Сложный случай: изменение типа колонки

**Задача:** `email VARCHAR(255)` → `VARCHAR(320)`.

**Проблема:** `ALTER COLUMN TYPE` блокирует таблицу.

**Безопасный паттерн:**

**Шаг 1: Добавить новую колонку**

```sql
ALTER TABLE users ADD COLUMN email_new VARCHAR(320);
```

**Шаг 2: Копировать данные частями**

```sql
UPDATE users SET email_new = email WHERE email_new IS NULL AND id IN (
    SELECT id FROM users WHERE email_new IS NULL LIMIT 10000
);
-- Повторить.
```

**Шаг 3: Синхронизировать новые записи**

Пока копируем, приложение продолжает писать в `email`. Нужен триггер, чтобы новые/изменённые значения попадали в `email_new`:

```sql
CREATE OR REPLACE FUNCTION sync_email_new()
RETURNS TRIGGER AS $$
BEGIN
    NEW.email_new := NEW.email;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_sync_email
BEFORE INSERT OR UPDATE OF email ON users
FOR EACH ROW EXECUTE FUNCTION sync_email_new();
```

**Шаг 4: Переключить приложение**

Приложение начинает использовать `email_new`.

**Шаг 5: Удалить старую колонку**

```sql
ALTER TABLE users DROP COLUMN email;
ALTER TABLE users RENAME COLUMN email_new TO email;
```

### CREATE INDEX CONCURRENTLY — без блокировки

Мы упоминали это в главе 3. Для миграций это критично:

```sql
-- ❌ Блокирует запись на время создания индекса:
CREATE INDEX idx_users_phone ON users(phone);

-- ✅ Без блокировки:
CREATE INDEX CONCURRENTLY idx_users_phone ON users(phone);
```

**Ограничения CONCURRENTLY:**

- Нельзя внутри транзакции (`BEGIN; ... COMMIT;`)
- Медленнее обычного (2-3 раза)
- Если упадёт — останется «невалидный» индекс, нужно `DROP INDEX CONCURRENTLY`

```sql
-- Проверить невалидные индексы:
SELECT indexrelid::regclass, indrelid::regclass
FROM pg_index
WHERE NOT indisvalid;

-- Удалить невалидный:
DROP INDEX CONCURRENTLY idx_users_phone;
```

### ADD CONSTRAINT CHECK ... NOT VALID

Мы разбирали это в главе 10.2. Повторю кратко:

```sql
-- ❌ Блокирует — проверяет ВСЕ строки:
ALTER TABLE users ADD CONSTRAINT users_age_check CHECK (age >= 0);

-- ✅ Без блокировки:
ALTER TABLE users ADD CONSTRAINT users_age_check CHECK (age >= 0) NOT VALID;
-- Мгновенно. Новые строки проверяются, старые — нет.

-- Потом валидируем в фоне:
ALTER TABLE users VALIDATE CONSTRAINT users_age_check;
-- Не блокирует запись, но может занять время на большой таблице.
```

### Что под капотом при блокирующих миграциях

Вспомни главы 1, 2, 5, 6:

```
ALTER TABLE users ALTER COLUMN email TYPE VARCHAR(320):

1. Захватывает ACCESS EXCLUSIVE LOCK — никто не может читать/писать.
2. Создаёт новый файл таблицы (relfilenode меняется, Глава 2).
3. Читает каждую страницу через Buffer Manager (Глава 1).
4. Для каждой строки: конвертирует email, создаёт новую версию (MVCC, Глава 5).
5. Пишет WAL-записи на каждую операцию (Глава 1).
6. Создаёт новые страницы, пишет на диск.
7. Пересоздаёт все индексы (Глава 3).
8. Удаляет старый файл, снимает блокировку.

Для 10 млн строк:
  - 10 млн конверсий
  - ~1.25 млн страниц прочитано и записано
  - Гигабайты WAL
  - Минуты или часы блокировки
```

### «Бизнес через год» и миграции

Бизнес говорит: «Email теперь может быть длиннее 255 символов».

**Если это предусмотрено (text + CHECK из главы 10.2):**

```sql
ALTER TABLE users DROP CONSTRAINT users_email_length_check;
ALTER TABLE users ADD CONSTRAINT users_email_length_check
    CHECK (length(email) <= 320) NOT VALID;
ALTER TABLE users VALIDATE CONSTRAINT users_email_length_check;
-- Без блокировки. Мгновенно.
```

**Если email был VARCHAR(255):**

```sql
ALTER TABLE users ALTER COLUMN email TYPE VARCHAR(320);
-- Блокировка на часы. Бизнес недоволен.
```

**Вывод:** выбор типа в главе 10 напрямую влияет на сложность миграций.

### Итог подглавы 11.14

- **Миграция** — изменение структуры БД.
- **Блокирующие операции:** `ALTER COLUMN TYPE`, `CREATE INDEX` (обычный), `ADD CONSTRAINT CHECK` (без NOT VALID).
- **Неблокирующие:** `ADD COLUMN`, `DROP COLUMN`, `CREATE INDEX CONCURRENTLY`, `ADD CONSTRAINT ... NOT VALID`.
- **Безопасный паттерн:** добавь новое → заполни частями → переключи приложение → удали старое.
- **Инструменты** (Goose, Flyway) — в главе 25.

---

## 11.15 Выводы и типичные ошибки

**Что мы узнали?**

Нормализация — это метод проектирования схемы, при котором данные разносятся по таблицам так, чтобы исключить дублирование и аномалии вставки/обновления/удаления. Нормальные формы (1НФ, 2НФ, 3НФ, НФБК) — последовательные шаги: атомарность, полная зависимость от ключа, отсутствие транзитивных зависимостей, все зависимости от объявленных ключей. Нормализация платит объёмом и JOIN'ами, но выигрывает целостность и скорость UPDATE. Денормализация — осознанное нарушение форм под конкретные частые запросы. Выбор ключей, именование, связи, антипаттерны и миграции — практические инструменты, которые превращают теорию в рабочую схему.

**Типичные ошибки:**

- ❌ **Хранить CSV в колонке** (`tags TEXT: 'a,b,c'`) — нарушение 1НФ, невозможность индексов и JOIN'ов.
- ❌ **Держать транзитивные зависимости** (`orders.customer_email`) — аномалии обновления, массовые UPDATE.
- ❌ **Использовать естественный ключ, который может измениться** (email как PK) — смена ключа ломает все FK.
- ❌ **Не объявлять бизнес-уникальность** (`UNIQUE` на order_number) — противоречивые данные.
- ❌ **Денормализовать преждевременно** — JOIN'ы на 10 000 строк не медленные, не усложняй схему.
- ❌ **EAV или таблица-свалка** — отсутствие типов, индексов, целостности.
- ❌ **Колонки с номерами** (`image_1`, `image_2`) — не масштабируется, нарушение 1НФ.
- ❌ **Блокирующие миграции** (`ALTER COLUMN TYPE`) без паттерна «добавь → заполни → переключи → удали».
- ❌ **Непоследовательное именование** — `users` и `tbl_users` в одном проекте.
- ❌ **Всё в схеме `public`** — 200 таблиц в одной куче, нет разделения по модулям.

---

## 11.16 Для быстрого повторения

- **1НФ** — атомарность: одна ячейка = одно значение. CSV, массивы, JSON с извлекаемыми полями — выноси.
- **2НФ** — полная зависимость от ключа: неключевые колонки не должны зависеть от части составного ключа.
- **3НФ** — нет транзитивных зависимостей: колонка описывает факт о ключе, а не о другой неключевой колонке.
- **НФБК** — все зависимости исходят из объявленных ключей: бизнес-уникальность = `UNIQUE constraint`.
- **Нормализация платит:** +20% объёма, JOIN'ы в 10-20 раз медленнее точечного чтения. **Выигрывает:** UPDATE в 500 раз быстрее, нет аномалий.
- **Связи:** 1:1 — FK в любой; 1:N — FK на стороне «много»; N:M — промежуточная таблица; иерархия — FK на себя.
- **PK по умолчанию:** `id BIGSERIAL PRIMARY KEY` + `UNIQUE` на естественный ключ.
- **Именование:** таблицы во множественном (users), колонки snake_case, FK — `customer_id`, булевы — `is_active`, схемы по модулям.
- **Антипаттерны:** EAV, таблица-свалка, CSV в колонке, колонки с номерами.
- **Миграции:** `ADD COLUMN` и `DROP COLUMN` — без блокировки; `ALTER COLUMN TYPE` — блокирует; `CREATE INDEX CONCURRENTLY` — без блокировки.

---

## 11.17 Вопросы для самопроверки

1. Какие три аномалии решает нормализация? Приведи пример каждой.
2. Что такое 1НФ? Почему CSV в колонке — нарушение?
3. В чём разница между 2НФ и 3НФ? Приведи пример транзитивной зависимости.
4. Что такое НФБК? Чем она строже 3НФ?
5. Нормализация в цифрах: почему нормализованная схема занимает БОЛЬШЕ места? Насколько быстрее UPDATE?
6. Когда денормализация оправдана? Приведи конкретный сценарий.
7. Как связать две таблицы N:M? Что должно быть в промежуточной таблице?
8. Суррогатный vs естественный ключ: что выбрать для `users` и почему?
9. Почему email — плохой первичный ключ?
10. Назови четыре антипаттерна проектирования и объясни, чем каждый плох.
11. Как добавить колонку с DEFAULT без блокировки таблицы?
12. Как изменить тип колонки без блокировки? Опиши безопасный паттерн.

---

## 11.18 Ответы

### Ответ 1

**Аномалия вставки:** нельзя добавить клиента без заказа, если данные клиента хранятся в таблице заказов.

**Аномалия обновления:** смена email клиента требует обновить все его заказы (10 000 строк).

**Аномалия удаления:** удаление последнего заказа клиента удаляет и данные клиента.

### Ответ 2

**1НФ** — атомарность: каждая колонка содержит одно значение. CSV (`'a,b,c'`) — это несколько значений в одной ячейке. Нельзя индексировать, JOIN'ить, группировать по отдельным значениям.

### Ответ 3

**2НФ** — неключевые колонки полностью зависят от всего составного ключа (не от части). **3НФ** — неключевые колонки не зависят от других неключевых колонок (нет транзитивных зависимостей). Пример транзитивной зависимости: `orders.customer_email` зависит от `customer_id`, который зависит от `orders.id`.

### Ответ 4

**НФБК** — каждая функциональная зависимость `X → Y` имеет в качестве `X` объявленный ключ. Строже 3НФ тем, что ловит случаи с несколькими потенциальными ключами, когда колонка уникальна, но не объявлена `UNIQUE`.

### Ответ 5

Нормализованная схема занимает больше из-за: отдельных таблиц (заголовки страниц), дополнительных FK-индексов, промежуточных таблиц N:M. UPDATE в 500 раз быстрее (1 строка вместо 1000).

### Ответ 6

Когда JOIN'ы реально узкое место: частый запрос, Hash Join сбрасывается на диск, >5 JOIN'ов, отчёты читаются чаще, чем обновляются. Пример: топ-100 товаров через Materialized View.

### Ответ 7

Промежуточная таблица с составным PK `(order_id, tag_id)`, FK с `ON DELETE CASCADE`, индексы на обе FK-колонки.

### Ответ 8

Суррогатный `id BIGSERIAL PK` + `UNIQUE` на email. Суррогатный — для FK и JOIN'ов (компактный, неизменный), UNIQUE — для бизнес-поиска и целостности.

### Ответ 9

Email может измениться. Если email — PK, то смена email требует обновить все FK в дочерних таблицах. Плюс TEXT-ключ больше BIGINT — индексы больше, JOIN'ы медленнее.

### Ответ 10

**EAV** — всё в «ключ-значение»: нет типов, индексов, целостности. **Таблица-свалка** — все сущности в одной таблице с `type`: NULL-колонки, нет FK. **CSV в колонке** — нарушение 1НФ. **Колонки с номерами** — не масштабируется.

### Ответ 11

```sql
-- PostgreSQL 11+: мгновенно, без переписывания таблицы
ALTER TABLE users ADD COLUMN phone TEXT DEFAULT '+7';
```

### Ответ 12

Безопасный паттерн: добавить новую колонку → копировать данные батчами → триггер для синхронизации новых записей → переключить приложение → удалить старую колонку → переименовать новую.

---

## 11.19 Куда идти дальше?

Мы разобрали, как спроектировать схему: нормальные формы, связи, ключи, именование, антипаттерны, миграции. Но проектирование — это не только «как разложить данные по таблицам». Есть **типовые паттерны**, которые встречаются в каждом проекте:

- Как хранить деревья (категории, комментарии) без рекурсивных кошмаров?
- Как сделать очередь задач на PostgreSQL (с `SKIP LOCKED`)?
- Как хранить счётчики и балансы без гонок?
- Как проектировать подписки, лайки, голосования?

**Глава 12: Паттерны проектирования в PostgreSQL — очереди, деревья, счётчики и другие типовые задачи.**