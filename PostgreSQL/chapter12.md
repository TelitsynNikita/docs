# 📐 Глава 12: Паттерны проектирования в PostgreSQL — очереди, деревья, счётчики и другие типовые задачи

**Что вы узнаете:**
- Как хранить деревья и иерархии: Adjacency List, Materialized Path, Nested Sets.
- Как реализовать очередь задач на PostgreSQL с `FOR UPDATE SKIP LOCKED`.
- Как безопасно вести счётчики и балансы без гонок.
- Как проектировать лайки, голосования, подписки.
- Как хранить временные ряды и логи с учётом роста.
- Как реализовать soft delete и версионирование строк.
- Паттерны «последний заказ клиента», «топ-N», «агрегаты».

**После прочтения вы сможете:**
- Выбрать правильный паттерн для дерева под свою задачу.
- Реализовать надёжную очередь на PostgreSQL без внешних брокеров.
- Проектировать счётчики, которые не теряют обновления при конкуренции.
- Строить схемы для лайков, голосований и подписок с защитой от дублей.
- Понимать, когда паттерн на PostgreSQL лучше, чем Kafka/Redis.
- Связать паттерны с индексами, блокировками и VACUUM'ом из глав 1-10.

---

## Содержание

- [12.0 Пролог: задачи, которые повторяются из проекта в проект](#120-пролог-задачи-которые-повторяются-из-проекта-в-проект)
- [12.1 Деревья: Adjacency List, Materialized Path, Nested Sets](#121-деревья-adjacency-list-materialized-path-nested-sets)
- [12.2 Очередь задач на PostgreSQL: FOR UPDATE SKIP LOCKED](#122-очередь-задач-на-postgresql-for-update-skip-locked)
- [12.3 Счётчики и балансы: атомарные UPDATE без гонок](#123-счётчики-и-балансы-атомарные-update-без-гонок)
- [12.4 Лайки, голосования, подписки: защита от дублей](#124-лайки-голосования-подписки-защита-от-дублей)
- [12.5 Временные ряды и логи: проектирование под рост](#125-временные-ряды-и-логи-проектирование-под-рост)
- [12.6 Soft delete и версионирование строк](#126-soft-delete-и-версионирование-строк)
- [12.7 Паттерны «последний заказ», «топ-N», «агрегаты»](#127-паттерны-последний-заказ-топ-n-агрегаты)
- [12.8 Практика Go: реализация очереди и счётчика](#128-практика-go-реализация-очереди-и-счётчика)
- [12.9 Выводы и типичные ошибки](#129-выводы-и-типичные-ошибки)
- [12.10 Для быстрого повторения](#1210-для-быстрого-повторения)
- [12.11 Вопросы для самопроверки](#1211-вопросы-для-самопроверки)
- [12.12 Ответы](#1212-ответы)
- [12.13 Куда идти дальше?](#1213-куда-идти-дальше)

---

## 12.0 Пролог: задачи, которые повторяются из проекта в проект

Ты спроектировал схему по всем правилам главы 11: 3НФ, суррогатные ключи, правильные связи. Но теперь перед тобой задачи, которые **не решаются нормализацией**:

- **Дерево категорий** — как хранить вложенность? Как получить всех потомков узла?
- **Очередь задач** — 100 воркеров разбирают задачи. Как не дать двум воркерам взять одну задачу?
- **Счётчик лайков** — 1000 лайков в секунду. Как не потерять обновления?
- **Лента событий** — миллиард записей в день. Как хранить, чтобы не раздуть базу?

Это **типовые паттерны**. Они повторяются в каждом проекте, и для каждого есть **проверенное решение**.

❓ **Почему бы не использовать готовые инструменты — Kafka, Redis?**

Иногда — да, нужно. Но PostgreSQL умеет **многое** из этого сам. Очередь на `FOR UPDATE SKIP LOCKED`, счётчики на атомарных UPDATE, деревья на рекурсивных CTE. И когда бизнес только начинает — **PostgreSQL достаточно**. Внешние брокеры добавляют сложность: инфраструктура, мониторинг, отладка.

В этой главе разберём паттерны **на PostgreSQL** — когда они работают, когда ломаются, и когда пора переезжать на Kafka/Redis.

---

## 12.1 Деревья: Adjacency List, Materialized Path, Nested Sets

### Задача

У тебя есть **иерархия**: категории товаров, комментарии к постам, подразделения компании.

```
Электроника
├── Телефоны
│   ├── iPhone
│   └── Samsung
├── Ноутбуки
└── Аксессуары
    ├── Чехлы
    └── Зарядки
```

**Нужно уметь:**

1. Найти всех **детей** узла (один уровень).
2. Найти всех **потомков** узла (все уровни вниз).
3. Найти всех **предков** узла (путь вверх).
4. Найти **глубину** узла.
5. Добавить/переместить/удалить узел.

### Паттерн 1: Adjacency List (список смежности)

**Идея:** каждая строка ссылается на родителя через `parent_id`.

```sql
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    parent_id BIGINT REFERENCES categories(id)
);

CREATE INDEX idx_categories_parent_id ON categories(parent_id);

INSERT INTO categories (id, name, parent_id) VALUES
    (1, 'Электроника', NULL),
    (2, 'Телефоны', 1),
    (3, 'Ноутбуки', 1),
    (4, 'Аксессуары', 1),
    (5, 'iPhone', 2),
    (6, 'Samsung', 2),
    (7, 'Чехлы', 4),
    (8, 'Зарядки', 4);
```

**Запросы:**

```sql
-- Дети (один уровень):
SELECT * FROM categories WHERE parent_id = 2;

-- Потомки (все уровни) — рекурсивный CTE:
WITH RECURSIVE descendants AS (
    SELECT id, name, parent_id, 1 AS depth
    FROM categories
    WHERE id = 1  -- корень
    UNION ALL
    SELECT c.id, c.name, c.parent_id, d.depth + 1
    FROM categories c
    JOIN descendants d ON d.id = c.parent_id
)
SELECT * FROM descendants;

-- Предки (путь вверх):
WITH RECURSIVE ancestors AS (
    SELECT id, name, parent_id
    FROM categories
    WHERE id = 5  -- iPhone
    UNION ALL
    SELECT c.id, c.name, c.parent_id
    FROM categories c
    JOIN ancestors a ON c.id = a.parent_id
)
SELECT * FROM ancestors;
```

**Плюсы:**
- Простота: одна таблица, понятные связи.
- Вставка/перемещение: один UPDATE.

**Минусы:**
- Потомки/предки — через рекурсию. На глубоких деревьях (> 10 уровней) медленно.
- Нельзя посчитать глубину без рекурсии.

**Когда использовать:** дерево неглубокое (до 5-10 уровней), количество узлов до миллионов.

---

### Паттерн 2: Materialized Path (материализованный путь)

**Идея:** хранить **полный путь** до узла в виде строки.

```sql
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    path TEXT NOT NULL,  -- '1/2/5/'
    parent_id BIGINT REFERENCES categories(id)
);

CREATE INDEX idx_categories_path ON categories(path);

INSERT INTO categories (id, name, path, parent_id) VALUES
    (1, 'Электроника', '1/', NULL),
    (2, 'Телефоны', '1/2/', 1),
    (5, 'iPhone', '1/2/5/', 2);
```

**Запросы:**

```sql
-- Потомки узла 2 (все уровни) — без рекурсии!
SELECT * FROM categories WHERE path LIKE '1/2/%';

-- Предки узла 5:
SELECT * FROM categories WHERE id IN (1, 2);

-- Глубина узла:
SELECT array_length(string_to_array(path, '/'), 1) - 1 AS depth
FROM categories WHERE id = 5;
```

**Плюсы:**
- Потомки/предки — без рекурсии, простой `LIKE` по индексу.
- Глубина — из длины пути.

**Минусы:**
- `path` — денормализация: при перемещении узла нужно обновить пути всех потомков.
- `LIKE '1/2/%'` — индекс работает только для префиксного поиска (`text_pattern_ops`).
- Максимальная глубина ограничена длиной строки.

**Когда использовать:** дерево читается часто, изменяется редко. Глубина до 20-30.

---

### Паттерн 3: Nested Sets (вложенные множества)

**Идея:** хранить **левую и правую границы** поддерева.

```sql
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    left_id INT NOT NULL,
    right_id INT NOT NULL
);

-- Каждый узел — интервал [left_id, right_id].
-- Дети — внутри интервала родителя.
```

**Запросы:**

```sql
-- Потомки узла (все уровни) — без рекурсии:
SELECT * FROM categories
WHERE left_id > 1 AND right_id < 8;

-- Предки:
SELECT * FROM categories
WHERE left_id < 5 AND right_id > 5;

-- Глубина:
SELECT COUNT(*) - 1 FROM categories
WHERE left_id < 5 AND right_id > 5;
```

**Плюсы:**
- Потомки/предки/глубина — простые сравнения чисел.
- Быстро на больших деревьях.

**Минусы:**
- Вставка/перемещение — пересчёт всех left_id/right_id.
- Сложно поддерживать.

**Когда использовать:** дерево **почти не меняется**, но часто читается. Например, справочник подразделений.

---

### Сравнение

| Паттерн | Чтение потомков | Вставка | Перемещение | Глубина дерева |
|:---|:---|:---|:---|:---|
| **Adjacency List** | Рекурсия | O(1) | O(1) | До 10 |
| **Materialized Path** | LIKE | O(1) | O(N) потомков | До 30 |
| **Nested Sets** | Сравнение чисел | O(N) | O(N) | Любая |

**Рекомендация:** начинай с **Adjacency List**. Если дерево глубокое и рекурсия медленная — переходи на **Materialized Path**. Nested Sets — только если дерево статично.

---

## 12.2 Очередь задач на PostgreSQL: FOR UPDATE SKIP LOCKED

### Задача

У тебя 100 воркеров обрабатывают задачи: отправка email, генерация отчётов, обработка изображений.

```
Задача создаётся → попадает в очередь → воркер берёт её → обрабатывает → помечает выполненной.
```

**Требования:**

1. Задачу берёт **только один** воркер (не два одновременно).
2. Воркеры работают **параллельно**, не ждут друг друга.
3. Если воркер упал — задача **не теряется**.
4. Порядок обработки — FIFO (первый пришёл — первый вышел).

### Наивное решение: SELECT + UPDATE

```sql
-- Воркер 1:
SELECT * FROM tasks WHERE status = 'pending' ORDER BY id LIMIT 1;
-- Видит задачу id=42

-- Воркер 2 (в тот же момент):
SELECT * FROM tasks WHERE status = 'pending' ORDER BY id LIMIT 1;
-- Тоже видит задачу id=42!

-- Оба UPDATE:
UPDATE tasks SET status = 'processing' WHERE id = 42;
-- Оба думают, что взяли задачу. Гонка!
```

**Проблема:** между SELECT и UPDATE другой воркер может взять ту же задачу. Это **потерянное обновление** (Глава 5).

### Решение: FOR UPDATE SKIP LOCKED

```sql
-- Воркер берёт задачу атомарно:
WITH next_task AS (
    SELECT id
    FROM tasks
    WHERE status = 'pending'
    ORDER BY id
    FOR UPDATE SKIP LOCKED
    LIMIT 1
)
UPDATE tasks SET status = 'processing'
WHERE id IN (SELECT id FROM next_task)
RETURNING *;
```

**Что происходит под капотом (Главы 5, 13):**

```
Воркер 1:
  SELECT ... FOR UPDATE SKIP LOCKED
  → Блокирует строку id=42 (статус 'pending')
  → Возвращает её

Воркер 2 (параллельно):
  SELECT ... FOR UPDATE SKIP LOCKED
  → Пытается взять id=42, но она ЗАБЛОКИРОВАНА
  → SKIP LOCKED: пропускает заблокированную
  → Идёт к id=43, блокирует её
  → Возвращает id=43

Результат: воркер 1 → id=42, воркер 2 → id=43.
Никаких гонок.
```

**Ключевые механики:**

| Элемент | Что делает |
|:---|:---|
| `FOR UPDATE` | Блокирует выбранные строки до конца транзакции |
| `SKIP LOCKED` | Пропускает заблокированные строки вместо ожидания |
| `LIMIT 1` | Берёт одну задачу |
| `ORDER BY id` | FIFO: первая в очереди — первая обрабатывается |

### Полный цикл обработки задачи в Go

```go
func processNextTask(db *sql.DB) error {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // 1. Взять задачу атомарно
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
        return nil // нет задач
    }
    if err != nil {
        return err
    }

    // 2. Обработать задачу (вне транзакции, если долго)
    err = processTask(taskID, payload)

    // 3. Пометить выполненной или вернуть в очередь
    if err != nil {
        _, _ = tx.Exec("UPDATE tasks SET status = 'failed', error = $1 WHERE id = $2", err.Error(), taskID)
    } else {
        _, _ = tx.Exec("UPDATE tasks SET status = 'done', completed_at = now() WHERE id = $2", taskID)
    }

    return tx.Commit()
}
```

**Почему транзакция короткая:** `FOR UPDATE` держит блокировку до `COMMIT`. Если обрабатывать задачу внутри транзакции — блокировка висит минуты. Поэтому:

1. Транзакция: взять задачу + пометить `processing` + COMMIT.
2. Вне транзакции: обработать.
3. Новая транзакция: пометить `done` или `failed`.

### Восстановление зависших задач

Воркер взял задачу, пометил `processing`, но **упал** — транзакция откатилась автоматически (Глава 5), но статус в таблице остался `processing`. Задача «зависла».

**Решение:** периодический «сборщик» возвращает зависшие задачи в очередь:

```sql
-- Вернуть в pending задачи, которые обрабатываются > 5 минут:
UPDATE tasks
SET status = 'pending', started_at = NULL
WHERE status = 'processing'
  AND started_at < now() - interval '5 minutes';
```

### Индекс для очереди

```sql
-- Частичный индекс: только pending задачи (Глава 3):
CREATE INDEX idx_tasks_pending ON tasks(id)
WHERE status = 'pending';
```

**Зачем:** запрос `WHERE status = 'pending' ORDER BY id LIMIT 1` сканирует только pending, а не всю таблицу.

### Когда очередь на PostgreSQL НЕ подходит

| Симптом | Причина | Что делать |
|:---|:---|:---|
| > 10 000 задач/сек | Блокировки строк, WAL, VACUUM | Kafka, Redis |
| Задачи > 100 МБ | TOAST, чтение больших значений | S3 + ссылка |
| Нужна гарантированная доставка | PostgreSQL не брокер | Kafka |

**Правило:** до 1 000-5 000 задач/сек PostgreSQL — отлично. Выше — смотри Kafka.

### «Бизнес через год» и очередь

Бизнес говорит: «У нас 50 000 задач в секунду, PostgreSQL не справляется».

**Если очередь была на PostgreSQL:**

```
Мигрируем на Kafka:
1. Producer пишет в Kafka вместо INSERT в tasks.
2. Consumer'ы читают из Kafka.
3. tasks остаётся как таблица для истории (что обработано).
```

**Если сразу на Kafka:** не нужно было бы. Но для стартапа PostgreSQL — быстрее в разработке, дешевле в инфраструктуре.

**Итог подглавы 12.2:**

- `FOR UPDATE SKIP LOCKED` — атомарный выбор задачи без гонок.
- Транзакция короткая: взять → COMMIT → обработать → COMMIT.
- Частичный индекс на pending — быстрое сканирование.
- Периодический сборщик — восстановление зависших задач.
- PostgreSQL — до ~5 000 задач/сек. Выше — Kafka.

---

## 12.3 Счётчики и балансы: атомарные UPDATE без гонок

### Задача

У тебя есть счётчики:

- **Количество лайков** под постом
- **Баланс пользователя** (нельзя уйти в минус)
- **Количество просмотров** товара
- **Остаток на складе** (нельзя продать больше, чем есть)

**Требования:**

1. Обновления **не теряются** при конкуренции.
2. Баланс/остаток **не может быть отрицательным**.
3. Обновление **быстрое**, даже при 1000 запросов в секунду.

### Проблема: SELECT + UPDATE с фиксированным значением

```sql
-- ❌ Гонка:
SELECT balance FROM users WHERE id = 42;  -- balance = 100

-- Другой запрос: UPDATE balance = 50 WHERE id = 42; COMMIT;

UPDATE users SET balance = 90 WHERE id = 42;  -- записываем 90, а не 50!
-- Потерянное обновление (Глава 5).
```

### Решение: атомарный UPDATE

```sql
-- ✅ Атомарно: читаем и обновляем в одной операции
UPDATE users SET balance = balance - 50 WHERE id = 42;
-- PostgreSQL сам берёт блокировку строки, перечитывает balance, вычитает.
```

**Что под капотом (Глава 5):**

```
UPDATE users SET balance = balance - 50 WHERE id = 42:

1. Backend-процесс находит строку (Index Scan по PK).
2. Блокирует строку (row lock).
3. Читает актуальный balance = 100.
4. Вычисляет 100 - 50 = 50.
5. Создаёт новую версию строки (MVCC), balance = 50.
6. Старая версия — мёртвый кортеж (Глава 6).
7. COMMIT — блокировка снимается.

Другой UPDATE ждёт блокировку, потом перечитывает УЖЕ 50.
```

### С ограничением целостности

```sql
-- Баланс не может быть отрицательным:
ALTER TABLE users ADD CONSTRAINT positive_balance CHECK (balance >= 0);

-- Списание с проверкой:
UPDATE users SET balance = balance - 50 WHERE id = 42 AND balance >= 50;
-- Если balance < 50 → 0 rows affected → приложение обрабатывает ошибку.
```

### Go-пример: списание баланса

```go
func withdraw(db *sql.DB, userID int64, amount int64) error {
    result, err := db.Exec(
        "UPDATE users SET balance = balance - $1 WHERE id = $2 AND balance >= $1",
        amount, userID,
    )
    if err != nil {
        return err
    }

    rowsAffected, _ := result.RowsAffected()
    if rowsAffected == 0 {
        return errors.New("недостаточно средств")
    }
    return nil
}
```

### Счётчик лайков

**Вариант 1: лайки в отдельной таблице, счётчик — агрегат**

```sql
CREATE TABLE likes (
    user_id BIGINT REFERENCES users(id),
    post_id BIGINT REFERENCES posts(id),
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (user_id, post_id)  -- один лайк от пользователя
);

-- Количество лайков:
SELECT COUNT(*) FROM likes WHERE post_id = 42;
-- Медленно при миллионах лайков — агрегация по всей таблице.
```

**Вариант 2: счётчик в колонке**

```sql
ALTER TABLE posts ADD COLUMN likes_count BIGINT DEFAULT 0;

-- Лайк:
INSERT INTO likes (user_id, post_id) VALUES (42, 100)
    ON CONFLICT (user_id, post_id) DO NOTHING;  -- защита от дубля
UPDATE posts SET likes_count = likes_count + 1 WHERE id = 100;
-- Атомарный UPDATE, быстро.
```

**Вариант 3: гибрид (для больших нагрузок)**

```sql
-- Счётчик в Redis, синхронизация с PostgreSQL раз в N секунд.
-- Или таблица счётчиков с периодическим сбросом в основную.
```

### Счётчик просмотров (не критичная точность)

```sql
-- Просто атомарный UPDATE:
UPDATE products SET views_count = views_count + 1 WHERE id = 42;
-- 1000 UPDATE в секунду — строки блокируются, но быстро.
-- При 100 000 в секунду — узкое место: одна строка = одна блокировка.
```

**Для очень больших счётчиков** (миллионы в секунду) используют:

- **Буфер в памяти** (Redis) → сброс в PostgreSQL раз в секунду.
- **Таблицу счётчиков с шардированием** — несколько строк на один счётчик.

### Остаток на складе

```sql
-- ❌ Гонка:
SELECT quantity FROM products WHERE id = 42;  -- quantity = 1
UPDATE products SET quantity = 0 WHERE id = 42;

-- ✅ Атомарно:
UPDATE products SET quantity = quantity - 1 WHERE id = 42 AND quantity >= 1;
-- Если 0 rows affected → товар закончился.
```

**Дополнительная защита — CHECK:**

```sql
ALTER TABLE products ADD CONSTRAINT non_negative_quantity CHECK (quantity >= 0);
-- Если какой-то запрос забудет WHERE quantity >= 1 → ошибка constraint'а.
```

### Таблица счётчиков

Для множества счётчиков (лайки, просмотры, комментарии):

```sql
CREATE TABLE counters (
    entity_type TEXT NOT NULL,   -- 'post', 'user', 'product'
    entity_id BIGINT NOT NULL,
    counter_name TEXT NOT NULL,  -- 'likes', 'views', 'comments'
    value BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (entity_type, entity_id, counter_name)
);

-- Инкремент:
INSERT INTO counters (entity_type, entity_id, counter_name, value)
VALUES ('post', 42, 'likes', 1)
ON CONFLICT (entity_type, entity_id, counter_name)
DO UPDATE SET value = counters.value + 1;
```

**Почему ON CONFLICT:** если строки счётчика нет — создаём. Если есть — инкрементируем. Всё в одной атомарной операции.

### «Бизнес через год» и счётчики

Бизнес говорит: «Лайков стало 1 млн в секунду, PostgreSQL не справляется».

**Решение — Redis:**

```go
// Пишем лайк в Redis:
redis.Incr("post:42:likes")

// Фоновый процесс раз в 10 секунд:
val := redis.Get("post:42:likes")
db.Exec("UPDATE posts SET likes_count = $1 WHERE id = 42", val)
```

**PostgreSQL остаётся источником правды, Redis — быстрый буфер.**

### Итог подглавы 12.3

- **Атомарный UPDATE** (`SET x = x - N WHERE ... AND x >= N`) — основа счётчиков.
- **CHECK constraint** — защита от отрицательных значений.
- **ON CONFLICT** — идемпотентный инкремент для таблицы счётчиков.
- **Redis** — для экстремальных нагрузок (> 100 000/сек), PostgreSQL остаётся источником правды.

---

## 12.4 Лайки, голосования, подписки: защита от дублей

### Задача

Три классические задачи:

1. **Лайки** — пользователь ставит лайк посту. Один пользователь = один лайк.
2. **Голосования** — пользователь выбирает вариант. Один пользователь = один голос.
3. **Подписки** — пользователь подписывается на другого. Одна пара = одна подписка.

**Требования:**

- Нельзя поставить два лайка.
- Нельзя проголосовать дважды.
- Нельзя подписаться дважды.
- Быстрая проверка: «лайкнул ли я?», «на кого я подписан?»

### Общий принцип: составной PRIMARY KEY

```sql
CREATE TABLE likes (
    user_id BIGINT REFERENCES users(id),
    post_id BIGINT REFERENCES posts(id),
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (user_id, post_id)  -- ✅ гарантирует: один пользователь = один лайк
);
```

**Что под капотом:** составной PK — это уникальный индекс (Глава 3). Вторая вставка той же пары `(user_id, post_id)` упадёт:

```sql
INSERT INTO likes (user_id, post_id) VALUES (42, 100);
-- OK

INSERT INTO likes (user_id, post_id) VALUES (42, 100);
-- ERROR: duplicate key value violates unique constraint "likes_pkey"
```

### Идемпотентная вставка: ON CONFLICT DO NOTHING

```sql
INSERT INTO likes (user_id, post_id)
VALUES (42, 100)
ON CONFLICT (user_id, post_id) DO NOTHING;
-- Если лайк уже есть — ничего не происходит, ошибки нет.
```

**Зачем:** приложение может повторять запрос (ретраи, двойной клик) — идемпотентность защищает от дублей.

### Лайки: полный пример

```sql
CREATE TABLE likes (
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    post_id BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (user_id, post_id)
);

-- Для запроса «лайки поста» (обратный порядок):
CREATE INDEX idx_likes_post_id ON likes(post_id);

-- Поставить лайк:
INSERT INTO likes (user_id, post_id) VALUES (42, 100)
ON CONFLICT (user_id, post_id) DO NOTHING;

-- Убрать лайк:
DELETE FROM likes WHERE user_id = 42 AND post_id = 100;

-- Проверить, лайкнул ли пользователь:
SELECT EXISTS (
    SELECT 1 FROM likes WHERE user_id = 42 AND post_id = 100
) AS liked;

-- Все лайки поста:
SELECT user_id FROM likes WHERE post_id = 100;
```

**Индексы:** PK `(user_id, post_id)` покрывает «лайки пользователя». Дополнительный индекс `(post_id)` — «кто лайкнул пост».

### Голосования: с защитой от смены голоса

```sql
CREATE TABLE votes (
    user_id BIGINT NOT NULL REFERENCES users(id),
    poll_id BIGINT NOT NULL REFERENCES polls(id),
    option_id BIGINT NOT NULL REFERENCES poll_options(id),
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (user_id, poll_id)  -- один голос в одном опросе
);

-- Проголосовать:
INSERT INTO votes (user_id, poll_id, option_id)
VALUES (42, 10, 3)
ON CONFLICT (user_id, poll_id) DO UPDATE SET option_id = EXCLUDED.option_id;
-- Если голос был — обновляем выбор.
```

**Отличие от лайков:** `ON CONFLICT DO UPDATE` — разрешаем сменить голос.

### Подписки: с защитой от самоподписки

```sql
CREATE TABLE subscriptions (
    follower_id BIGINT NOT NULL REFERENCES users(id),
    following_id BIGINT NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (follower_id, following_id),
    CHECK (follower_id <> following_id)  -- нельзя подписаться на себя
);

-- Подписаться:
INSERT INTO subscriptions (follower_id, following_id)
VALUES (42, 100)
ON CONFLICT (follower_id, following_id) DO NOTHING;

-- Подписчики пользователя:
SELECT follower_id FROM subscriptions WHERE following_id = 100;

-- На кого подписан пользователь:
SELECT following_id FROM subscriptions WHERE follower_id = 42;
```

**CHECK** — дополнительная защита от логической ошибки.

### Паттерн «fan-out» для ленты

Когда пользователь с миллионом подписчиков постит — нужно доставить пост миллиону подписчиков.

**Вариант 1: Pull (тянуть)**

```sql
-- Лента пользователя: посты тех, на кого он подписан
SELECT p.*
FROM posts p
JOIN subscriptions s ON s.following_id = p.user_id
WHERE s.follower_id = 42
ORDER BY p.created_at DESC
LIMIT 20;
-- JOIN на каждый запрос. При 1000 подписок — Hash Join (Глава 9).
```

**Вариант 2: Push (толкать)**

```sql
-- При создании поста — разложить по лентам подписчиков:
CREATE TABLE feeds (
    user_id BIGINT NOT NULL,   -- кому показывать
    post_id BIGINT NOT NULL,   -- какой пост
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (user_id, post_id)
);

-- Лента:
SELECT p.*
FROM feeds f
JOIN posts p ON p.id = f.post_id
WHERE f.user_id = 42
ORDER BY f.created_at DESC
LIMIT 20;
-- Быстро: Index Scan по feeds.user_id.
```

**Минус Push:** миллион подписчиков = миллион строк в `feeds`. Для знаменитостей — кошмар.

**Решение — гибрид:**
- Обычные пользователи: Push.
- Знаменитости (> 100 000 подписчиков): Pull.

### Итог подглавы 12.4

- **Составной PK** — главная защита от дублей в лайках/голосах/подписках.
- **ON CONFLICT DO NOTHING** — идемпотентность.
- **ON CONFLICT DO UPDATE** — смена голоса.
- **CHECK** — защита от логических ошибок (самоподписка).
- **Push vs Pull** — для ленты: Push для обычных, Pull для знаменитостей.

---

## 12.5 Временные ряды и логи: проектирование под рост

### Задача

Ты пишешь события: клики, просмотры, логи приложения, метрики.

```
1 000 000 событий в день
365 000 000 событий в год
```

**Требования:**

- Вставка **очень быстрая** (тысячи в секунду).
- Чтение — по времени: «события за последний час», «за вчера».
- Таблица не должна «раздуваться» (VACUUM из главы 6).
- Старые данные **удаляются** без боли.

### Паттерн 1: Простая таблица событий

```sql
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    event_type TEXT NOT NULL,
    user_id BIGINT,
    payload JSONB,
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_events_created_at ON events(created_at);
```

**Проблемы при росте:**

```
1 млрд строк:
  - Таблица ~100 ГБ
  - INSERT: вставка в конец, OK
  - DELETE старых: массовое удаление → миллионы мёртвых кортежей (Глава 6)
  - VACUUM: часы на очистку
  - Индекс по created_at: ~30 ГБ, растёт
```

### Паттерн 2: Партиционирование по дате

```sql
CREATE TABLE events (
    id BIGSERIAL,
    event_type TEXT NOT NULL,
    user_id BIGINT,
    payload JSONB,
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (id, created_at)  -- партиционный ключ в PK
) PARTITION BY RANGE (created_at);

-- Партиции по дням:
CREATE TABLE events_2024_01_01 PARTITION OF events
FOR VALUES FROM ('2024-01-01') TO ('2024-01-02');

CREATE TABLE events_2024_01_02 PARTITION OF events
FOR VALUES FROM ('2024-01-02') TO ('2024-01-03');
```

**Преимущества:**

- Удаление старых — `DROP TABLE events_2024_01_01` (мгновенно, без мёртвых кортежей).
- Чтение по дате — только нужные партиции.
- Каждая партиция — отдельная таблица, VACUUM по ней быстрее.

**Подробно о партиционировании** — в главе 17.

### Паттерн 3: Таблица с BRIN-индексом

Вспомни главу 4: BRIN для коррелированных данных.

```sql
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    event_type TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- BRIN вместо B-tree:
CREATE INDEX idx_events_created_at_brin ON events USING BRIN (created_at);
-- Размер: ~2-5 МБ вместо 30 ГБ (для 1 млрд строк).
```

**Плюсы:** индекс крошечный, всегда в памяти.
**Минусы:** точность ниже — читает диапазоны страниц, а не точечные TID'ы.

### Паттерн 4: Гибрид — горячие в PostgreSQL, холодные в ClickHouse

```
PostgreSQL: данные за последние 7 дней (горячие, быстрые запросы).
ClickHouse: все данные (аналитика, агрегаты).

Перенос: фоновый процесс раз в час копирует старые данные в ClickHouse
и удаляет из PostgreSQL.
```

**Когда:** событий > 10 млн/день, нужна аналитика по году.

### Выбор паттерна по объёму

| Объём | Паттерн |
|:---|:---|
| < 1 млн/день | Простая таблица + B-tree |
| 1-10 млн/день | Партиционирование по дате |
| 10-100 млн/день | BRIN + партиционирование |
| > 100 млн/день | PostgreSQL (горячие) + ClickHouse (холодные) |

### «Бизнес через год» и временные ряды

Бизнес говорит: «Событий стало 50 млн в день, аналитика тормозит».

**Если была простая таблица:**

```sql
-- Мигрируем на партиционирование или ClickHouse.
-- Больно: перенос данных, изменение запросов.
```

**Если сразу партиционирование:**

```sql
-- Просто добавляем партиции. Всё работает.
```

### Итог подглавы 12.5

- **Простая таблица** — до 1 млн событий/день.
- **Партиционирование** — от 1 до 100 млн/день, удаление старых через DROP.
- **BRIN** — компактный индекс для коррелированных `created_at`.
- **ClickHouse** — для аналитики > 100 млн/день, PostgreSQL держит горячие данные.

---

## 12.6 Soft delete и версионирование строк

### Задача

Бизнес требует:

- «Не удаляй навсегда — вдруг понадобится восстановить».
- «Хочу видеть, как менялась цена за последний год».
- «Кто и когда изменил статус заказа?»

Это два связанных паттерна: **soft delete** (мягкое удаление) и **версионирование** (история изменений).

### Soft delete: удаление через `deleted_at`

**Идея:** вместо физического `DELETE` — ставим временную метку.

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    deleted_at TIMESTAMPTZ,  -- NULL = не удалён
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ
);

-- «Удаление»:
UPDATE users SET deleted_at = now() WHERE id = 42;

-- Все рабочие запросы учитывают deleted_at:
SELECT * FROM users WHERE id = 42 AND deleted_at IS NULL;
```

**Частичный индекс для активных (Глава 3):**

```sql
-- Быстрый поиск по email только среди активных:
CREATE UNIQUE INDEX idx_users_email_active ON users(email)
WHERE deleted_at IS NULL;
-- Гарантирует: активный email уникален, удалённый — можно переиспользовать.
```

**Минусы soft delete:**

1. Таблица растёт — «удалённые» строки занимают место.
2. Все запросы должны помнить `AND deleted_at IS NULL`.
3. `UNIQUE` с soft delete — нужен частичный индекс (обычный UNIQUE не даст переиспользовать email).

### Версионирование: история изменений строки

**Идея:** хранить **все версии** строки в отдельной таблице.

```sql
-- Основная таблица:
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC(12,2) NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT now()
);

-- Таблица истории:
CREATE TABLE product_history (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id),
    name TEXT,
    price NUMERIC(12,2),
    changed_at TIMESTAMPTZ DEFAULT now(),
    changed_by TEXT
);

CREATE INDEX idx_product_history_product_id ON product_history(product_id, changed_at);
```

**Запись истории при обновлении:**

```sql
-- В одной транзакции:
BEGIN;

UPDATE products SET price = 900, updated_at = now() WHERE id = 42;

INSERT INTO product_history (product_id, name, price, changed_by)
SELECT id, name, price, current_user FROM products WHERE id = 42;

COMMIT;
```

**Запрос «какая была цена 1 января»:**

```sql
SELECT price
FROM product_history
WHERE product_id = 42
  AND changed_at <= '2024-01-01'
ORDER BY changed_at DESC
LIMIT 1;
```

### Автоматическое версионирование через триггер

Вручную INSERT в историю легко забыть. Триггер делает это автоматически (разберём в главе 24):

```sql
CREATE OR REPLACE FUNCTION log_product_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO product_history (product_id, name, price, changed_by)
    VALUES (NEW.id, NEW.name, NEW.price, current_user);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_history
AFTER UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION log_product_changes();
```

### Комбинированный паттерн: soft delete + версионирование

```sql
CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    content TEXT,
    version INT NOT NULL DEFAULT 1,
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- История версий:
CREATE TABLE document_versions (
    document_id BIGINT NOT NULL REFERENCES documents(id),
    version INT NOT NULL,
    title TEXT,
    content TEXT,
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (document_id, version)
);
```

### «Бизнес через год» и soft delete

Бизнес говорит: «Верните удалённого пользователя».

**Если был физический DELETE:**

```sql
-- Данных нет. Восстановить невозможно.
```

**Если был soft delete:**

```sql
UPDATE users SET deleted_at = NULL WHERE id = 42;
-- Готово.
```

### Итог подглавы 12.6

- **Soft delete** — `deleted_at` вместо `DELETE`, частичный UNIQUE индекс.
- **Версионирование** — таблица истории, вручную или через триггер.
- **Комбинация** — soft delete + версии для важных документов.
- **Цена:** таблицы растут, запросы усложняются. Применяй осознанно.

---

## 12.7 Паттерны «последний заказ», «топ-N», «агрегаты»

Три задачи, которые встречаются в каждом проекте:

1. **Последний заказ клиента** — как получить последнюю запись в группе.
2. **Топ-N** — как получить N самых популярных.
3. **Агрегаты** — как считать суммы/количества без пересчёта.

### Паттерн 1: «Последний заказ клиента»

**Задача:** для списка клиентов показать их последний заказ.

**Наивное решение — коррелированный подзапрос:**

```sql
SELECT u.id, u.name,
    (SELECT o.id FROM orders o
     WHERE o.user_id = u.id
     ORDER BY o.created_at DESC
     LIMIT 1) AS last_order_id
FROM users u;
-- Для каждого пользователя — подзапрос. N+1 проблема.
```

**Решение — DISTINCT ON:**

```sql
SELECT DISTINCT ON (user_id)
    user_id,
    id AS order_id,
    created_at
FROM orders
ORDER BY user_id, created_at DESC;
-- Для каждого user_id — первая строка по created_at DESC.
```

**Что под капотом:** сортировка по `(user_id, created_at DESC)`, затем выбор первой строки для каждого `user_id`. Индекс `(user_id, created_at DESC)` делает это быстрым.

```sql
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);
```

**Решение — оконная функция (глава 13):**

```sql
SELECT user_id, order_id, created_at
FROM (
    SELECT user_id, id AS order_id, created_at,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
) sub
WHERE rn = 1;
```

### Паттерн 2: «Топ-N»

**Задача:** топ-10 самых продаваемых товаров.

**Решение — GROUP BY + LIMIT:**

```sql
SELECT product_id, COUNT(*) AS sales_count
FROM order_items
GROUP BY product_id
ORDER BY sales_count DESC
LIMIT 10;
-- Агрегация по всей таблице order_items. На 100 млн строк — секунды.
```

**Оптимизация — Materialized View (глава 8):**

```sql
CREATE MATERIALIZED VIEW top_products AS
SELECT product_id, COUNT(*) AS sales_count
FROM order_items
GROUP BY product_id
ORDER BY sales_count DESC;

-- Обновление раз в 5 минут:
REFRESH MATERIALIZED VIEW top_products;

-- Чтение:
SELECT * FROM top_products LIMIT 10;
-- Мгновенно.
```

**Оптимизация — счётчик в колонке:**

```sql
ALTER TABLE products ADD COLUMN sales_count BIGINT DEFAULT 0;

-- При каждом заказе:
UPDATE products SET sales_count = sales_count + 1 WHERE id = 42;

-- Топ-10:
SELECT id, sales_count FROM products ORDER BY sales_count DESC LIMIT 10;
-- Index Scan по sales_count. Быстро.
```

### Паттерн 3: «Агрегаты»

**Задача:** считать сумму заказов за день, средний чек, количество новых пользователей.

**Решение — GROUP BY:**

```sql
SELECT created_at::DATE AS day,
       COUNT(*) AS orders_count,
       SUM(total) AS revenue
FROM orders
GROUP BY created_at::DATE
ORDER BY day;
-- На 100 млн строк — HashAggregate (глава 9), секунды.
```

**Оптимизация — таблица агрегатов:**

```sql
CREATE TABLE daily_stats (
    day DATE PRIMARY KEY,
    orders_count BIGINT NOT NULL,
    revenue NUMERIC(14,2) NOT NULL
);

-- Обновление при каждом заказе:
INSERT INTO daily_stats (day, orders_count, revenue)
VALUES (CURRENT_DATE, 1, 100)
ON CONFLICT (day) DO UPDATE SET
    orders_count = daily_stats.orders_count + 1,
    revenue = daily_stats.revenue + EXCLUDED.revenue;
```

**Чтение:**

```sql
SELECT * FROM daily_stats WHERE day = CURRENT_DATE;
-- 1 строка. Мгновенно.
```

### Выбор паттерна по частоте

| Паттерн | Когда |
|:---|:---|
| **DISTINCT ON** | Последний заказ — частый запрос, индекс `(user_id, created_at DESC)` |
| **Оконные функции** | Сложная логика внутри группы (глава 13) |
| **Materialized View** | Топ-N обновляется редко, читается часто |
| **Счётчик в колонке** | Топ-N обновляется часто, нужна точность |
| **Таблица агрегатов** | Агрегаты нужны постоянно, пересчёт дорогой |

### «Бизнес через год» и агрегаты

Бизнес говорит: «Хочу видеть выручку за каждый день за последний год».

**Если GROUP BY по orders:**

```sql
SELECT created_at::DATE, SUM(total)
FROM orders
WHERE created_at > now() - interval '1 year'
GROUP BY created_at::DATE;
-- 365 млн строк, HashAggregate. Минуты.
```

**Если таблица daily_stats:**

```sql
SELECT * FROM daily_stats
WHERE day > now() - interval '1 year'
ORDER BY day;
-- 365 строк. Мгновенно.
```

### Итог подглавы 12.7

- **DISTINCT ON** — последний заказ, с индексом `(user_id, created_at DESC)`.
- **Materialized View** — топ-N, когда обновление редкое.
- **Счётчик в колонке** — топ-N, когда обновление частое.
- **Таблица агрегатов** — дневная статистика, экономит часы пересчёта.

---

## 12.8 Практика Go: реализация очереди и счётчика

Соберём два ключевых паттерна в Go-коде: очередь задач и счётчик.

```go
package main

import (
    "database/sql"
    "errors"
    "fmt"
    "log"
    "time"

    _ "github.com/jackc/pgx/v5/stdlib"
)

// Задача из очереди
type Task struct {
    ID      int64
    Payload string
}

// Взять следующую задачу атомарно (FOR UPDATE SKIP LOCKED)
func takeNextTask(db *sql.DB) (*Task, error) {
    tx, err := db.Begin()
    if err != nil {
        return nil, err
    }
    defer tx.Rollback()

    var task Task
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
    `).Scan(&task.ID, &task.Payload)

    if err == sql.ErrNoRows {
        return nil, nil // нет задач
    }
    if err != nil {
        return nil, err
    }

    if err := tx.Commit(); err != nil {
        return nil, err
    }
    return &task, nil
}

// Завершить задачу
func completeTask(db *sql.DB, taskID int64, success bool, errMsg string) error {
    if success {
        _, err := db.Exec("UPDATE tasks SET status = 'done', completed_at = now() WHERE id = $1", taskID)
        return err
    }
    _, err := db.Exec("UPDATE tasks SET status = 'failed', error = $1 WHERE id = $2", errMsg, taskID)
    return err
}

// Списание баланса (атомарный UPDATE)
func withdraw(db *sql.DB, userID int64, amount int64) error {
    result, err := db.Exec(
        "UPDATE users SET balance = balance - $1 WHERE id = $2 AND balance >= $1",
        amount, userID,
    )
    if err != nil {
        return err
    }

    rowsAffected, _ := result.RowsAffected()
    if rowsAffected == 0 {
        return errors.New("недостаточно средств")
    }
    return nil
}

// Инкремент счётчика (идемпотентный)
func incrementCounter(db *sql.DB, entityType string, entityID int64, counterName string) error {
    _, err := db.Exec(`
        INSERT INTO counters (entity_type, entity_id, counter_name, value)
        VALUES ($1, $2, $3, 1)
        ON CONFLICT (entity_type, entity_id, counter_name)
        DO UPDATE SET value = counters.value + 1
    `, entityType, entityID, counterName)
    return err
}

func main() {
    db, err := sql.Open("pgx", "postgres://alice@localhost/mydb")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Воркер: бесконечный цикл обработки задач
    for {
        task, err := takeNextTask(db)
        if err != nil {
            log.Printf("Ошибка взятия задачи: %v", err)
            time.Sleep(time.Second)
            continue
        }
        if task == nil {
            // Нет задач — ждём
            time.Sleep(time.Second)
            continue
        }

        // Обрабатываем задачу
        fmt.Printf("Обрабатываю задачу %d: %s\n", task.ID, task.Payload)
        err = completeTask(db, task.ID, true, "")
        if err != nil {
            log.Printf("Ошибка завершения задачи: %v", err)
        }
    }
}
```

**Что делает код:**

1. `takeNextTask` — атомарно берёт задачу через `FOR UPDATE SKIP LOCKED`.
2. `completeTask` — помечает задачу выполненной или упавшей.
3. `withdraw` — списание баланса с проверкой.
4. `incrementCounter` — идемпотентный инкремент счётчика.

---

## 12.9 Выводы и типичные ошибки

**Что мы узнали?**

Паттерны проектирования — это проверенные решения типовых задач: деревья (Adjacency List, Materialized Path, Nested Sets), очереди (FOR UPDATE SKIP LOCKED), счётчики (атомарные UPDATE), лайки/голосования/подписки (составной PK + ON CONFLICT), временные ряды (партиционирование, BRIN, ClickHouse), soft delete и версионирование, агрегаты (DISTINCT ON, Materialized View, таблица агрегатов). Каждый паттерн — это trade-off между скоростью чтения, скоростью записи и сложностью.

**Типичные ошибки:**

- ❌ **SELECT + UPDATE для очереди** — гонка, два воркера берут одну задачу.
- ❌ **Держать транзакцию открытой во время обработки задачи** — блокировка на минуты.
- ❌ **SELECT + UPDATE для счётчиков** — потерянные обновления.
- ❌ **COUNT(*) по лайкам для каждой страницы** — агрегация по всей таблице.
- ❌ **Простая таблица для 100 млн событий в день** — раздувание, VACUUM не справляется.
- ❌ **Физический DELETE вместо soft delete** — данные не восстановить.
- ❌ **Push fan-out для знаменитостей** — миллион строк на один пост.
- ❌ **Рекурсия по Adjacency List на глубине 20+** — медленно, нужен Materialized Path.

---

## 12.10 Для быстрого повторения

- **Деревья:** Adjacency List (простой, рекурсия) → Materialized Path (LIKE, глубина до 30) → Nested Sets (числа, статичное дерево).
- **Очередь:** `FOR UPDATE SKIP LOCKED` + короткая транзакция + частичный индекс + сборщик зависших.
- **Счётчики:** атомарный `UPDATE ... SET x = x - N WHERE x >= N` + CHECK constraint.
- **Лайки/голоса:** составной PRIMARY KEY + `ON CONFLICT DO NOTHING/DO UPDATE`.
- **Временные ряды:** простая таблица → партиционирование → BRIN → ClickHouse.
- **Soft delete:** `deleted_at` + частичный UNIQUE индекс.
- **Агрегаты:** DISTINCT ON для «последнего», Materialized View для топ-N, таблица агрегатов для статистики.

---

## 12.11 Вопросы для самопроверки

1. Чем Materialized Path лучше Adjacency List для глубоких деревьев? А в чём его минус?
2. Как `FOR UPDATE SKIP LOCKED` предотвращает гонку в очереди?
3. Почему транзакция в очереди должна быть короткой?
4. Как атомарный UPDATE решает проблему потерянных обновлений в счётчиках?
5. Как защититься от дублей в лайках? Как разрешить смену голоса?
6. Когда BRIN лучше B-tree для временных рядов?
7. Почему физический DELETE хуже soft delete для пользователей?
8. Как получить последний заказ каждого клиента без N+1?
9. Чем Push fan-out отличается от Pull? Какой для знаменитостей?
10. Когда PostgreSQL-очередь не справляется и нужен Kafka?

---

## 12.12 Ответы

### Ответ 1

Materialized Path хранит путь (`'1/2/5/'`) — потомки через `LIKE` без рекурсии. Минус: при перемещении узла нужно обновить пути всех потомков.

### Ответ 2

`FOR UPDATE` блокирует выбранную строку, `SKIP LOCKED` пропускает заблокированные. Второй воркер не ждёт, а берёт следующую свободную задачу.

### Ответ 3

`FOR UPDATE` держит блокировку до COMMIT. Длинная транзакция = долгие блокировки = другие воркеры ждут.

### Ответ 4

PostgreSQL сам блокирует строку, перечитывает текущее значение, вычисляет новое. Конкурентный UPDATE ждёт блокировку, потом перечитывает уже обновлённое значение.

### Ответ 5

Составной PRIMARY KEY (`user_id, post_id`) + `ON CONFLICT DO NOTHING`. Для смены голоса — `ON CONFLICT DO UPDATE SET option_id = EXCLUDED.option_id`.

### Ответ 6

BRIN в тысячи раз меньше B-tree для коррелированных `created_at`. Цена: меньшая точность — читает диапазоны страниц.

### Ответ 7

Физический DELETE необратим. Soft delete (`deleted_at`) позволяет восстановить: `UPDATE ... SET deleted_at = NULL`.

### Ответ 8

`DISTINCT ON (user_id) ... ORDER BY user_id, created_at DESC` с индексом `(user_id, created_at DESC)`.

### Ответ 9

Push — разложить пост по лентам подписчиков (быстрое чтение). Pull — JOIN при каждом чтении ленты. Для знаменитостей — Pull, иначе миллион строк в feeds.

### Ответ 10

> 5 000-10 000 задач/сек, задачи > 100 МБ, гарантированная доставка — тогда Kafka.

---

## 12.13 Куда идти дальше?

Мы разобрали паттерны проектирования: деревья, очереди, счётчики, лайки, временные ряды, soft delete, агрегаты. Но SQL, который мы писали, был **относительно простым** — JOIN'ы, GROUP BY, подзапросы.

Есть ещё **мощный инструмент**, который решает задачи, не решаемые обычным SQL:

- Как посчитать «топ-3 заказа каждого клиента» без N+1?
- Как посчитать «нарастающий итог» (running total)?
- Как сравнить значение со вчерашним?
- Как найти аномалии в последовательности?

**Глава 13: Продвинутый SQL — оконные функции, рекурсия и то, что невозможно сделать обычным SQL.**