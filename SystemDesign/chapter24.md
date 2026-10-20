## Глава 24: Real-time Gaming Leaderboard: ранжирование, обновления, масштабирование

### Оглавление главы

- **24.1** Уточнение требований: что такое лидерборд и кому он нужен
- **24.2** Оценка нагрузки: игроки, обновления, чтения
- **24.3** API: обновление счёта, топ-N, ранг игрока
- **24.4** Проектирование данных: Redis Sorted Sets, MySQL, tie-breaking
- **24.5** Почему не MySQL: цена сортировки на миллионах записей
- **24.6** Масштабирование: fixed partition vs hash partition
- **24.7** Отказоустойчивость и восстановление
- **24.8** Мониторинг и безопасность

---

### Введение

В предыдущей главе мы спроектировали биржу — от требований до продакшена. Ты узнал, что такое order book, matching engine, price-time priority, критический путь. Ты увидел, как те же строительные блоки (shared memory, multicast UDP, sequencer) применяются к ultra-low-latency задаче. Но биржа — это про торговлю финансовыми инструментами. А что если задача — это координация действий в реальном времени между множеством игроков?

Представь, что тебя просят спроектировать лидерборд для онлайн-игры. У тебя 5 миллионов активных игроков в день. Каждый игрок после матча получает очки. Игрок хочет видеть топ-10 мирового рейтинга, свой собственный ранг и позиции игроков рядом с ним. И всё это — в реальном времени, с задержкой в миллисекунды.

На первый взгляд, всё просто: у тебя есть таблица `players` с полем `score`, ты сортируешь по `score` и возвращаешь топ. Но что если игроков 25 миллионов? Что если 2500 обновлений счёта в секунду на пике? Что если каждый игрок хочет знать свой точный ранг, а не приблизительный? Наивный подход с `ORDER BY score DESC` на миллионах строк ломается на масштабе.

В этой главе ты узнаешь, как строить real-time gaming leaderboard. Ты разберёшь Redis Sorted Sets, почему они дают O(log N) для обновлений и ранжирования. Ты узнаешь, почему MySQL не подходит для больших лидербордов. Ты узнаешь, как масштабировать Redis через fixed partition и hash partition. И ты поймёшь, что gaming leaderboard — это задача о real-time ranking и consistency, где миллионы игроков соревнуются за место в топе.

---

### 24.1 Уточнение требований: что такое лидерборд и кому он нужен

Прежде чем рисовать диаграмму, нужно понять, что мы строим. И первый шаг — уточнить требования.

**Функциональные требования.**
- Игрок получает очки после победы в матче.
- Игрок может видеть топ-10 игроков мирового рейтинга.
- Игрок может видеть свой собственный ранг.
- Игрок может видеть игроков рядом с собой (например, ранги 45–55, если игрок на 50-м месте).
- Игрок может видеть историю прошлых сезонов (опционально).

**Нефункциональные требования.**
- Real-time обновления: изменение счёта отражается в лидерборде немедленно.
- Low latency: запросы к лидерборду возвращаются за миллисекунды.
- High concurrency: тысячи игроков одновременно отправляют счета и запрашивают ранги.
- Масштабируемость: система должна выдерживать рост до сотен миллионов игроков.
- Точность ранга: игрок должен видеть свой точный ранг, а не приблизительный (если это требование).

**Вопросы, которые нужно задать.**
- Сколько DAU (daily active users)? 5 миллионов? 500 миллионов?
- Сколько обновлений счёта в секунду на пике? 2500? 100 000?
- Нужен ли точный ранг или приблизительный?
- Нужны ли исторические лидерборды (прошлые месяцы)?
- Как обрабатывать tie-breaking (одинаковые счета)?
- Нужна ли поддержка разных лидербордов (глобальный, региональный, по друзьям)?
- Какая задержка допустима для чтения лидерборда? 10 мс? 100 мс?

Ответы на эти вопросы определяют архитектуру. Например, если 5 миллионов DAU и 2500 обновлений/с, одного Redis-сервера достаточно . Если 500 миллионов DAU, нужно шардирование . Если нужен точный ранг, fixed partition предпочтительнее hash partition .

Клеппман в «Высоконагруженных приложениях» объясняет, что требования определяют архитектуру. Без требований ты проектируешь в вакууме. И выбор зависит от того, какие требования у тебя в системе.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» (Volume 2, Chapter 10: Real-time Gaming Leaderboard) показывает, что лидерборд — это классическая задача на real-time ranking и sorted sets .

**Схема 24.1 — Общая картина системы (высокоуровневая)**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Game Clients                                │
│  (Mobile, Web, Console)                                             │
└──────┬──────────────────────────────────────────────────────────────┘
       │ HTTPS / WebSocket
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    API Gateway / Load Balancer                      │
│         TLS termination, rate limiting, auth                        │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
┌─────────────────────────┐   ┌─────────────────────────┐
│   Game Service #1       │   │   Game Service #2       │
│   (Go)                  │   │   (Go)                  │
│   - validate win        │   │   - validate win        │
│   - anti-cheat          │   │   - anti-cheat          │
│   - call Leaderboard    │   │   - call Leaderboard    │
│     Service (internal)  │   │     Service (internal)  │
└───────────┬─────────────┘   └───────────┬─────────────┘
            │                             │
            └──────────────┬──────────────┘
                           │ internal API (POST /v1/scores)
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Leaderboard Service                              │
│  - score update (ZINCRBY)                                           │
│  - top-N query (ZREVRANGE)                                          │
│  - rank query (ZREVRANK)                                            │
│  - nearby ranks                                                     │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  Redis        │  │  MySQL        │  │  Redis        │
│  (Sorted Set) │  │  (user,       │  │  (tie-break   │
│  leaderboard  │  │   score       │  │   timestamps) │
│               │  │   history)    │  │               │
└───────────────┘  └───────────────┘  └───────────────┘
```

---

### 24.2 Оценка нагрузки: игроки, обновления, чтения

Теперь у нас есть требования. Давай оценим нагрузку.

**Игроки.**
- 5 миллионов DAU (daily active users) — reference sizing из книги Алекса Сюя .
- 25 миллионов MAU (monthly active users) — worst case для хранения в Redis .

**Обновления счёта.**
- Peak QPS: 2500 обновлений счёта в секунду .
- Каждое обновление — это `ZINCRBY leaderboard_feb_2021 1 'user_id'` .

**Чтения.**
- Top-10 запросов: каждый игрок открывает лидерборд несколько раз в день.
- Rank запросов: игрок проверяет свой ранг после матча.
- Read:write ratio: примерно 5:1 (лидерборд write-heavy, но чтения тоже частые) .

**Хранение.**
- Одна запись: user_id (24 байта) + score (2 байта) = 26 байт .
- 25 миллионов записей * 26 байт = 650 МБ .
- С overhead Redis Sorted Set (skiplist + hash table): ~1.3 ГБ.
- Один современный Redis-сервер легко вмещает это .

**Latency.**
- Top-10: < 10 мс.
- Rank: < 10 мс.
- Score update: < 5 мс.

Вывод: система write-heavy (2500 обновлений/с) и read-heavy (частые top-N и rank запросы). Redis Sorted Sets дают O(log N) для обновлений и запросов, что идеально подходит. Один Redis-сервер справляется с 5M DAU. Для 500M DAU нужно шардирование.

Клеппман в «Высоконагруженных приложениях» объясняет, что оценки определяют, где bottleneck. Без оценок ты не знаешь, нужен ли шардинг, кеш, очередь. И выбор зависит от того, какие числа у тебя в системе.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как оценки влияют на архитектуру. Если у тебя 5M DAU, один Redis. Если 500M DAU, шардинг .

**Схема 24.2 — Потоки данных: score update, top-N, rank**

```
SCORE UPDATE PATH:

Game Client ──POST /game/win──▶ Game Service
                                    │
                                    ├──1. Validate win (anti-cheat)
                                    ├──2. Internal API: POST /v1/scores
                                    │      Body: {"user_id": "mary1934", "score_delta": 1}
                                    └──3. Leaderboard Service:
                                           ├── ZINCRBY leaderboard_feb_2021 1 'mary1934'
                                           └── (async) INSERT INTO score_history (user_id, score, timestamp)

TOP-N QUERY PATH:

Game Client ──GET /leaderboard/top?n=10──▶ Leaderboard Service
                                                │
                                                ├──1. ZREVRANGE leaderboard_feb_2021 0 9 WITHSCORES
                                                ├──2. Для каждого user_id:
                                                │      └── GET user:{user_id} из Redis cache
                                                │            (или MySQL, если MISS)
                                                └──3. Вернуть список с именами и аватарами

RANK QUERY PATH:

Game Client ──GET /leaderboard/rank/{user_id}──▶ Leaderboard Service
                                                      │
                                                      ├──1. ZREVRANK leaderboard_feb_2021 'mary1934'
                                                      ├──2. ZSCORE leaderboard_feb_2021 'mary1934'
                                                      └──3. Вернуть ранг и счёт

NEARBY RANKS PATH:

Game Client ──GET /leaderboard/nearby/{user_id}?range=5──▶ Leaderboard Service
                                                              │
                                                              ├──1. ZREVRANK leaderboard_feb_2021 'mary1934'
                                                              │      → rank = 361
                                                              ├──2. ZREVRANGE leaderboard_feb_2021 356 366
                                                              └──3. Вернуть 5 выше + 5 ниже + сам игрок
```

---

### 24.3 API: обновление счёта, топ-N, ранг игрока

API — это контракт между клиентом и сервером. Давай определим endpoints.

**Обновление счёта (internal API).**
```
POST /v1/scores
Body: {
  "user_id": "mary1934",
  "score_delta": 1
}
Response: {
  "user_id": "mary1934",
  "new_score": 1500,
  "rank": 42
}
```

Важно: Game Service вызывает Leaderboard Service через internal API. Клиент не может напрямую обновлять счёт — это защита от манипуляций . Score устанавливается на сервере, не на клиенте.

**Топ-N игроков.**
```
GET /v1/leaderboard/top?n=10
Response: {
  "leaderboard_id": "feb_2021",
  "updated_at": "2024-01-15T10:30:45Z",
  "players": [
    {"rank": 1, "user_id": "user2", "score": 2000, "display_name": "Alice", "avatar": "..."},
    {"rank": 2, "user_id": "user1", "score": 1900, "display_name": "Bob", "avatar": "..."},
    ...
  ]
}
```

**Ранг игрока.**
```
GET /v1/leaderboard/rank/{user_id}
Response: {
  "user_id": "mary1934",
  "score": 1500,
  "rank": 42
}
```

**Игроки рядом.**
```
GET /v1/leaderboard/nearby/{user_id}?range=5
Response: {
  "user_id": "mary1934",
  "rank": 42,
  "players": [
    {"rank": 37, "user_id": "...", "score": 1520},
    ...
    {"rank": 42, "user_id": "mary1934", "score": 1500},
    ...
    {"rank": 47, "user_id": "...", "score": 1480}
  ]
}
```

Клеппман в «Высоконагруженных приложениях» объясняет, что API — это контракт. Он определяет, как клиент и сервер взаимодействуют. И выбор стиля API (REST, WebSocket) зависит от того, кто клиенты и какие требования к производительности.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как API влияет на архитектуру. Если у тебя real-time лидерборд, WebSocket для push-обновлений (опционально).

**Схема 24.3 — API endpoints и их взаимодействие с компонентами**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         API Gateway / LB                            │
│                                                                     │
│  POST /game/win ──────────────▶ Game Service ──▶ POST /v1/scores   │
│  (client)                           │              (internal)       │
│                                     └──▶ Redis (ZINCRBY)            │
│                                                                     │
│  GET /leaderboard/top?n=10 ───▶ Leaderboard Service ──▶ Redis     │
│                                     │              (ZREVRANGE)     │
│                                     └──▶ MySQL (user details)      │
│                                                                     │
│  GET /leaderboard/rank/{id} ──▶ Leaderboard Service ──▶ Redis     │
│                                     │              (ZREVRANK)      │
│                                     └──▶ вернуть ранг и счёт       │
│                                                                     │
│  GET /leaderboard/nearby/{id} ▶ Leaderboard Service ──▶ Redis     │
│                                     │              (ZREVRANK +    │
│                                     │               ZREVRANGE)    │
│                                     └──▶ вернуть список игроков    │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 24.4 Проектирование данных: Redis Sorted Sets, MySQL, tie-breaking

Теперь давай спроектируем данные. Redis Sorted Sets — это ключевая структура.

**Redis Sorted Set.**
- Key: `leaderboard_{year}_{month}` (например, `leaderboard_feb_2021`).
- Member: `user_id`.
- Score: очки игрока.

Внутри Redis Sorted Set использует hash table для O(1) lookup по member и skiplist для O(log N) вставки, удаления и ранжирования .

**Операции.**
- `ZINCRBY leaderboard_feb_2021 1 'mary1934'` — увеличить счёт на 1 .
- `ZREVRANGE leaderboard_feb_2021 0 9 WITHSCORES` — получить топ-10 .
- `ZREVRANK leaderboard_feb_2021 'mary1934'` — получить ранг .
- `ZREVRANGE leaderboard_feb_2021 357 365` — получить игроков рядом .

**MySQL (supporting tables).**
```sql
CREATE TABLE users (
  user_id VARCHAR(24) PRIMARY KEY,
  display_name VARCHAR(100) NOT NULL,
  avatar_url VARCHAR(500),
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE score_history (
  id BIGSERIAL PRIMARY KEY,
  user_id VARCHAR(24) NOT NULL,
  score INT NOT NULL,
  won_at TIMESTAMP NOT NULL DEFAULT NOW(),
  leaderboard_id VARCHAR(20) NOT NULL
);

CREATE INDEX idx_score_history_user ON score_history(user_id, won_at DESC);
CREATE INDEX idx_score_history_leaderboard ON score_history(leaderboard_id, won_at DESC);
```

**Tie-breaking.**
Если два игрока имеют одинаковый счёт, как определить, кто выше? Решение: хранить timestamp последней победы. Игрок с более ранним timestamp получает более высокий ранг . Timestamp хранится в Redis Hash (`user:timestamps`) или в MySQL.

**Почему не MySQL для лидерборда.**
`ORDER BY score DESC LIMIT 10` на миллионах строк требует сортировки, что занимает десятки секунд . Для real-time лидерборда это неприемлемо. Redis Sorted Set даёт O(log N) для всех операций.

Клеппман в «Высоконагруженных приложениях» объясняет, что проектирование данных — это компромисс между нормализацией и денормализацией, между шардированием и cross-shard запросами. И выбор зависит от того, какие запросы у тебя в системе.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как проектирование данных влияет на архитектуру. Если у тебя real-time лидерборд, Redis Sorted Sets обязательны .

**Схема 24.4 — Схема данных: Redis Sorted Set, MySQL, tie-breaking**

```
Redis Sorted Set (leaderboard_feb_2021):
┌─────────────────────────────────────────────────────────────┐
│  Rank │ User ID     │ Score │ (internal)                    │
├───────┼─────────────┼───────┼───────────────────────────────┤
│  1    │ user2       │ 2000  │                               │
│  2    │ user1       │ 1900  │                               │
│  3    │ user5       │ 1800  │                               │
│  ...  │ ...         │ ...   │                               │
│  42   │ mary1934    │ 1500  │                               │
│  ...  │ ...         │ ...   │                               │
└─────────────────────────────────────────────────────────────┘

Internal structure:
- Hash table: user_id → score (O(1) lookup)
- Skiplist: sorted by score (O(log N) insert, delete, rank)

MySQL (users):
┌─────────────────────────────────────────────────────────────┐
│ user_id     │ display_name │ avatar_url                    │
├─────────────┼──────────────┼───────────────────────────────┤
│ mary1934    │ Mary         │ https://cdn.example.com/...   │
│ user2       │ Alice        │ https://cdn.example.com/...   │
└─────────────────────────────────────────────────────────────┘

MySQL (score_history):
┌─────────────────────────────────────────────────────────────┐
│ user_id     │ score │ won_at              │ leaderboard_id  │
├─────────────┼───────┼─────────────────────┼─────────────────┤
│ mary1934    │ 1     │ 2024-02-15 10:30:45 │ feb_2021        │
│ mary1934    │ 1     │ 2024-02-15 10:35:12 │ feb_2021        │
└─────────────────────────────────────────────────────────────┘

Redis Hash (user:timestamps):
┌─────────────────────────────────────────────────────────────┐
│ mary1934 → 1707993045 (timestamp последней победы)         │
└─────────────────────────────────────────────────────────────┘

Tie-breaking: если score одинаковый, игрок с меньшим timestamp
(более ранняя победа) получает более высокий ранг.
```

---

### 24.5 Почему не MySQL: цена сортировки на миллионах записей

Теперь давай разберём подробнее, почему MySQL не подходит для больших лидербордов.

**Проблема.**
Запрос `SELECT user_id, SUM(score) FROM scores GROUP BY user_id ORDER BY total DESC LIMIT 10` на 25 миллионах строк требует:
- Полного сканирования таблицы (или индекса).
- Группировки по user_id.
- Сортировки всех результатов.
- Взятия топ-10.

Это занимает десятки секунд . Для real-time лидерборда, где игрок ожидает ответ за миллисекунды, это неприемлемо.

**Даже с индексом.**
Индекс по `score` ускоряет `ORDER BY score DESC LIMIT 10`, но:
- При обновлении счёта индекс нужно обновлять, что добавляет write latency.
- При большом количестве обновлений индекс становится bottleneck.
- Точный ранг игрока (`SELECT COUNT(*) FROM scores WHERE score > ?`) требует сканирования индекса, что O(N).

**Redis Sorted Set решает это.**
- Вставка/обновление: O(log N) через skiplist.
- Top-N: O(log N + N) через ZREVRANGE.
- Rank: O(log N) через ZREVRANK.
- Всё в памяти, без disk I/O.

Клеппман в «Высоконагруженных приложениях» объясняет, что выбор структуры данных определяет производительность. B-tree (MySQL) хорош для диапазонных запросов на диске, но skiplist (Redis) лучше для in-memory ранжирования. И выбор зависит от того, что важнее для задачи.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как выбор структуры данных влияет на архитектуру. Если у тебя real-time ranking, Redis Sorted Set .

**Схема 24.5 — Почему не MySQL: цена сортировки**

```
MYSQL QUERY (25M rows):

SELECT user_id, SUM(score) AS total
FROM scores
GROUP BY user_id
ORDER BY total DESC
LIMIT 10;

Execution plan:
1. Scan scores table (25M rows) ──▶ ~10 seconds
2. Group by user_id ──▶ ~5 seconds
3. Sort by total DESC ──▶ ~5 seconds
4. Limit 10 ──▶ ~0 seconds
Total: ~20 seconds

REDIS SORTED SET (25M members):

ZREVRANGE leaderboard_feb_2021 0 9 WITHSCORES

Execution:
1. Skiplist traversal to highest score ──▶ O(log N)
2. Collect 10 members ──▶ O(10)
Total: < 1 millisecond

RANK QUERY:

MySQL:
SELECT COUNT(*) FROM scores WHERE score > (SELECT score FROM scores WHERE user_id = 'mary1934');
──▶ O(N) scan, ~5 seconds

Redis:
ZREVRANK leaderboard_feb_2021 'mary1934'
──▶ O(log N), < 1 millisecond
```

---

### 24.6 Масштабирование: fixed partition vs hash partition

Теперь давай разберём, как масштабировать Redis для больших лидербордов.

**Fixed partition (шардирование по диапазону счёта).**
- Ты сам шардируешь данные по диапазону счёта. Например, шард 1: 0–1000, шард 2: 1001–2000, и т.д. .
- При обновлении: определить, в каком шарде находится игрок (по текущему счёту). Если счёт выходит за диапазон, переместить игрока в другой шард .
- Top-N: взять топ-N из шарда с наивысшими счетами .
- Rank: вычислить локальный ранг в шарде + количество игроков с более высокими счетами в других шардах .

Преимущества: эффективный rank query (можно вычислить точный ранг). Недостатки: нужно управлять диапазонами и перемещать игроков при изменении счёта.

**Hash partition (Redis Cluster).**
- Redis Cluster автоматически шардирует данные по 16384 hash slots. `CRC16(key) % 16384` определяет slot, slot определяет ноду .
- При обновлении: просто обновить счёт в соответствующем шарде.
- Top-N: собрать топ-N из каждого шарда и отсортировать в приложении (scatter-gather) .
- Rank: сложно вычислить точный ранг (нужно знать количество игроков с более высокими счетами во всех шардах) .

Преимущества: автоматическое шардирование, равномерное распределение. Недостатки: scatter-gather для top-N, сложно для rank query.

**Что выбрать.**
Если нужен точный ранг (как в книге Алекса Сюя), fixed partition предпочтительнее . Если rank не критичен или можно приблизительный, hash partition проще.

Клеппман в «Высоконагруженных приложениях» объясняет, что шардирование — это компромисс между масштабируемостью и сложностью. Fixed partition сложнее в управлении, но даёт точный rank. Hash partition проще, но требует scatter-gather. И выбор зависит от того, что важнее для задачи.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как шардирование влияет на архитектуру. Если у тебя 500M DAU, шардирование обязательно .

**Схема 24.6 — Масштабирование: fixed partition vs hash partition**

```
FIXED PARTITION (by score range):

Shard 1: [0, 1000]
┌─────────────────────────────────────────────────────────────┐
│ user_a: 500, user_b: 800, user_c: 1000                     │
└─────────────────────────────────────────────────────────────┘

Shard 2: [1001, 2000]
┌─────────────────────────────────────────────────────────────┐
│ user_d: 1200, user_e: 1500, user_f: 2000                   │
└─────────────────────────────────────────────────────────────┘

Shard 3: [2001, 3000]
┌─────────────────────────────────────────────────────────────┐
│ user_g: 2500, user_h: 3000                                 │
└─────────────────────────────────────────────────────────────┘

Top-N: взять из Shard 3 (наивысшие счета).
Rank of user_e (1500):
  - local_rank в Shard 2: 2 (user_f выше)
  - players with higher scores in other shards: 2 (user_g, user_h)
  - global_rank = 2 + 2 + 1 = 5

HASH PARTITION (Redis Cluster):

Node 1: slots [0, 5500]
┌─────────────────────────────────────────────────────────────┐
│ user_a (hash slot 100): 500                                │
│ user_d (hash slot 200): 1200                               │
└─────────────────────────────────────────────────────────────┘

Node 2: slots [5501, 11000]
┌─────────────────────────────────────────────────────────────┐
│ user_b (hash slot 6000): 800                               │
│ user_e (hash slot 7000): 1500                              │
└─────────────────────────────────────────────────────────────┘

Node 3: slots [11001, 16383]
┌─────────────────────────────────────────────────────────────┐
│ user_c (hash slot 12000): 1000                             │
│ user_f (hash slot 13000): 2000                             │
└─────────────────────────────────────────────────────────────┘

Top-N: scatter-gather (top-10 from each node, then merge).
Rank of user_e: сложно (нужно знать счета всех игроков).
```

---

### 24.7 Отказоустойчивость и восстановление

Теперь давай разберём, как обеспечить отказоустойчивость и восстановление.

**Redis persistence.**
- Redis поддерживает RDB (snapshot) и AOF (append-only file). Для лидерборда RDB достаточно: если Redis упадёт, можно восстановить из snapshot .
- Но восстановление большого Redis instance с диска медленное. Поэтому используется репликация: primary + replica. Если primary падает, replica повышается .

**Восстановление из MySQL.**
- Score history в MySQL — это source of truth. Если Redis потерян, можно пересоздать лидерборд: `SELECT user_id, SUM(score) FROM score_history WHERE leaderboard_id = 'feb_2021' GROUP BY user_id` и загрузить в Redis .
- Это медленно (минуты), но гарантирует восстановление.

**Tie-breaking timestamps.**
- Timestamps последних побед хранятся в Redis Hash и в MySQL. Если Redis потерян, timestamps восстанавливаются из MySQL .

Клеппман в «Высоконагруженных приложениях» объясняет, что отказоустойчивость — это компромисс между доступностью и стоимостью. Репликация быстрее, но дороже. Восстановление из MySQL медленнее, но дешевле. И выбор зависит от того, что важнее для задачи.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как отказоустойчивость влияет на архитектуру. Если у тебя real-time лидерборд, репликация обязательна .

**Схема 24.7 — Отказоустойчивость и восстановление**

```
REDIS REPLICATION:

Primary Redis (leaderboard_feb_2021)
         │
         │ async replication
         ▼
Replica Redis (leaderboard_feb_2021)
         │
         └── если primary упал: replica повышается

RECOVERY FROM MYSQL:

Redis lost ──▶ Recovery Job:
                  │
                  ├──1. SELECT user_id, SUM(score) AS total
                  │      FROM score_history
                  │      WHERE leaderboard_id = 'feb_2021'
                  │      GROUP BY user_id
                  ├──2. ZADD leaderboard_feb_2021 {total} {user_id}
                  │      (batch, 1000 members at a time)
                  └──3. Restore timestamps from MySQL
                         (user:timestamps hash)

TIE-BREAKING STORAGE:

Redis Hash: user:timestamps
┌─────────────────────────────────────────────────────────────┐
│ mary1934 → 1707993045                                       │
│ user2    → 1707993100                                       │
└─────────────────────────────────────────────────────────────┘
(также в MySQL score_history.won_at)
```

---

### 24.8 Мониторинг и безопасность

Теперь давай разберём, как обеспечить мониторинг и безопасность.

**Мониторинг.**
- Метрики: score update QPS, top-N latency (p50, p95, p99), rank latency, Redis memory usage, Redis CPU, replication lag.
- Логи: структурированные, с trace ID.
- Алерты: latency p99 > 50 мс, Redis memory > 80%, replication lag > 1s.

**Безопасность.**
- Anti-cheat: Game Service валидирует победу. Клиент не может напрямую обновлять счёт .
- Rate limiting: ограничение обновлений счёта (например, 10 в секунду на игрока).
- Валидация: score_delta должен быть положительным и в разумных пределах (например, 1–100).
- Аутентификация: JWT для игроков.
- Авторизация: игрок может видеть только публичный лидерборд, не приватные данные других игроков.

Клеппман в «Высоконагруженных приложениях» объясняет, что мониторинг и безопасность — это не фичи, а свойства системы. Без них система не может эксплуатироваться. И выбор зависит от того, насколько критична система.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как мониторинг и безопасность влияют на архитектуру. Если у тебя gaming лидерборд, anti-cheat обязателен .

**Схема 24.8 — Полная архитектура с мониторингом и безопасностью**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Observability Stack                         │
│  Prometheus ──▶ Grafana (dashboards)                                │
│  Loki ──▶ Grafana (logs)                                            │
│  Alertmanager ──▶ PagerDuty/Slack (alerts)                          │
└─────────────────────────────────────────────────────────────────────┘
                                  ▲
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                    Leaderboard Service                              │
│  ┌──────────────────────────────┼──────────────────────────────┐   │
│  │  Metrics:                    │                              │   │
│  │  ├── score_update_qps        │                              │   │
│  │  ├── top_n_latency_p99       │                              │   │
│  │  ├── rank_latency_p99        │                              │   │
│  │  └── redis_memory_usage      │                              │   │
│  └──────────────────────────────┼──────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                         ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│ Redis         │         │ MySQL         │         │ Redis         │
│ Primary       │         │ (users,       │         │ (timestamps)  │
│ (Sorted Set)  │         │  score_history)│         │               │
│      │        │         │               │         │               │
│      ▼        │         │               │         │               │
│ Redis Replica │         │               │         │               │
│ (failover)    │         │               │         │               │
└───────────────┘         └───────────────┘         └───────────────┘

Security:
┌─────────────────────────────────────────────────────────────────────┐
│  JWT Auth ──▶ Rate Limiting ──▶ Anti-cheat (Game Service)          │
│  Score validation (positive, bounded)                               │
│  Audit log (score updates, rank queries)                            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Выводы

В этой главе ты узнал, как строить real-time gaming leaderboard. Ты разобрал Redis Sorted Sets, почему они дают O(log N) для обновлений и ранжирования. Ты узнал, почему MySQL не подходит для больших лидербордов. Ты узнал, как масштабировать Redis через fixed partition и hash partition. Ты увидел, как те же строительные блоки (Redis, MySQL, API) применяются к gaming задаче.

Главное, что ты должен вынести из этой главы: **real-time leaderboard — это задача о ranking и consistency.** Хорошая архитектура — не та, которая использует самую модную базу данных. Хорошая архитектура — та, которая обеспечивает O(log N) для обновлений и ранжирования, точный ранг для игрока и масштабируемость до сотен миллионов пользователей.

---

## Выжимка для быстрого повторения

- **Требования:** top-N, rank, nearby ranks, real-time updates. Low latency, high concurrency, точный ранг.
- **Оценки:** 5M DAU, 2500 updates/s, 25M MAU worst case. 650 MB в Redis (fits in one server).
- **API:** POST /v1/scores (internal), GET /leaderboard/top, GET /leaderboard/rank/{id}, GET /leaderboard/nearby/{id}.
- **Данные:** Redis Sorted Set (leaderboard_{year}_{month}), MySQL (users, score_history), Redis Hash (timestamps для tie-breaking).
- **Почему не MySQL:** `ORDER BY score DESC LIMIT 10` на 25M строк = десятки секунд. Redis Sorted Set = O(log N) = миллисекунды.
- **Масштабирование:** fixed partition (точный rank, сложнее управление) vs hash partition (автоматическое, scatter-gather для top-N).
- **Отказоустойчивость:** Redis replication, восстановление из MySQL (score_history).
- **Безопасность:** anti-cheat (Game Service валидирует победу), rate limiting, score validation.

---

## Вопросы для самопроверки

1. Почему Redis Sorted Set подходит для лидерборда? Какую сложность дают ZINCRBY, ZREVRANGE, ZREVRANK?
2. Почему MySQL не подходит для больших лидербордов? Какую проблему решает Redis?
3. Чем fixed partition отличается от hash partition? Когда выбирают каждый?
4. Как обрабатывать tie-breaking (одинаковые счета)?
5. Как восстановить лидерборд, если Redis потерян?

---

## Эталонные ответы

**1. Почему Redis Sorted Set подходит для лидерборда? Какую сложность дают ZINCRBY, ZREVRANGE, ZREVRANK?**

Redis Sorted Set использует hash table (O(1) lookup по member) и skiplist (O(log N) для вставки, удаления, ранжирования). ZINCRBY: O(log N) — обновление счёта. ZREVRANGE: O(log N + N) — получение топ-N. ZREVRANK: O(log N) — получение ранга. Это даёт миллисекундную latency для миллионов записей, что невозможно с MySQL `ORDER BY` на диске .

**2. Почему MySQL не подходит для больших лидербордов? Какую проблему решает Redis?**

MySQL `SELECT ... ORDER BY score DESC LIMIT 10` на 25 миллионах строк требует сканирования, группировки и сортировки, что занимает десятки секунд . Даже с индексом точный rank query (`COUNT(*) WHERE score > ?`) требует O(N) сканирования. Redis Sorted Set решает это через in-memory skiplist, давая O(log N) для всех операций. Всё в памяти, без disk I/O.

**3. Чем fixed partition отличается от hash partition? Когда выбирают каждый?**

Fixed partition: шардирование по диапазону счёта. Top-N из шарда с наивысшими счетами. Rank вычисляется как локальный ранг + количество игроков с более высокими счетами в других шардах. Эффективный точный rank, но нужно управлять диапазонами и перемещать игроков при изменении счёта . Hash partition: Redis Cluster, 16384 hash slots. Top-N требует scatter-gather (top-N из каждого шарда, merge). Rank сложно вычислить. Автоматическое шардирование, равномерное распределение, но scatter-gather для top-N . Fixed partition выбирают, когда нужен точный rank. Hash partition — когда rank не критичен.

**4. Как обрабатывать tie-breaking (одинаковые счета)?**

Хранить timestamp последней победы. Игрок с более ранним timestamp получает более высокий ранг. Timestamps хранятся в Redis Hash (`user:timestamps`) и в MySQL (`score_history.won_at`). При ZREVRANGE, если счета одинаковые, сравнить timestamps .

**5. Как восстановить лидерборд, если Redis потерян?**

Score history в MySQL — это source of truth. Recovery job: `SELECT user_id, SUM(score) AS total FROM score_history WHERE leaderboard_id = 'feb_2021' GROUP BY user_id`, затем `ZADD leaderboard_feb_2021 {total} {user_id}` batch'ами (1000 members at a time). Timestamps восстанавливаются из MySQL. Это медленно (минуты), но гарантирует восстановление .

---

## Мост к следующей главе

Ты спроектировал real-time gaming leaderboard — от требований до продакшена. Ты узнал, что такое Redis Sorted Sets, почему они дают O(log N), как масштабировать через fixed partition и hash partition. Ты увидел, как те же строительные блоки (Redis, MySQL, API) применяются к gaming задаче.

Но gaming leaderboard — это про ранжирование игроков. А что если задача — это координация между людьми, которые не знают друг друга, но находятся рядом? Что если нужно показать пользователю, кто из его друзей сейчас поблизости? Что если нужно обновлять местоположение в реальном времени и искать пересечения?

В следующей главе — **Nearby Friends: геолокация, real-time обновления, fan-out**. Ты узнаешь, как работает геопоиск для друзей, как обновлять местоположение и находить пересечения, как масштабировать на миллионы пользователей. Ты узнаешь, что такое geohash и почему он подходит для поиска друзей. И ты поймёшь, что nearby friends — это задача о real-time геолокации и fan-out, где пользователи хотят знать, кто из их круга рядом.