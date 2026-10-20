## Глава 21: Payment system: транзакции, идемпотентность, аудит

### Оглавление главы

- **21.1** Уточнение требований: что такое платёж и кому он нужен
- **21.2** Оценка нагрузки: транзакции, состояние, события
- **21.3** API: платёж, статус, вебхуки
- **21.4** Проектирование данных: двойная запись, идемпотентность, аудит
- **21.5** Транзакции: ACID, изоляция, блокировки
- **21.6** Идемпотентность: ключи, PENDING→COMPLETED, дедупликация
- **21.7** Обработка сбоев: timeout, crash, duplicate callback
- **21.8** Вебхуки и reconciliation
- **21.9** Отказоустойчивость, мониторинг и безопасность (PCI DSS)

---

### Введение

В предыдущей главе мы спроектировали video streaming платформу — от требований до продакшена. Ты узнал, что такое транскодинг, адаптивный битрейт, CDN, per-shot encoding. Ты увидел, как те же строительные блоки применяются к video streaming задаче. Но video streaming — это про доставку контента. А что если задача — это деньги?

Представь, что тебя просят спроектировать платёжную систему. Пользователь нажимает «оплатить». Система должна списать деньги с его карты, зачислить их продавцу, записать транзакцию, отправить чек, обновить баланс. И всё это — атомарно, надёжно, аудируемо. Если что-то пойдёт не так на любом шаге, деньги не должны потеряться или задвоиться. И если пользователь нажмёт «оплатить» дважды, с него не должно списаться дважды.

На первый взгляд, всё просто: вызвать платёжный шлюз, записать результат в базу данных. Но что если шлюз ответил «успех», а сеть оборвалась до того, как ты записал результат? Что если пользователь повторил запрос? Что если воркер упал после списания денег, но до зачисления продавцу? Что если вебхук от шлюза пришёл дважды? Наивный подход ломается на сбоях.

В этой главе ты узнаешь, как строить платёжную систему. Ты разберёшь транзакции и ACID, идемпотентность и ключи, обработку сбоев и reconciliation. Ты узнаешь, что такое двойная запись и почему она критична для аудита. Ты узнаешь, как обрабатывать duplicate callback и почему Kafka exactly-once не спасает при вызове внешнего API. И ты поймёшь, что payment system — это задача о строгой согласованности и надёжности, где ошибка стоит денег.

---

### 21.1 Уточнение требований: что такое платёж и кому он нужен

Прежде чем рисовать диаграмму, нужно понять, что мы строим. И первый шаг — уточнить требования.

**Функциональные требования.**
- Пользователь может инициировать платёж (карта, банковский перевод, кошелёк).
- Система проводит платёж через внешний провайдер (Stripe, Adyen, банк).
- Система записывает транзакцию и обновляет баланс.
- Пользователь может запросить статус платежа.
- Продавец получает вебхук о successful/failed платеже.
- Система поддерживает возвраты (refunds) и отмены (voids).
- Система ведёт аудит всех операций.

**Нефункциональные требования.**
- Строгая согласованность: деньги не могут потеряться или задвоиться.
- Идемпотентность: повторный запрос не создаёт дубликат.
- Долговечность: транзакции не теряются после подтверждения.
- Доступность: 99.99% для проведения платежей.
- Аудит: каждая операция должна быть отслеживаемой.
- Безопасность: PCI DSS, шифрование, токенизация.

**Вопросы, которые нужно задать.**
- Сколько платежей в секунду? 10 000? 100 000?
- Какие методы оплаты? Карты, банки, кошельки?
- Нужна ли поддержка нескольких валют? Как обрабатывать курсы?
- Нужна ли поддержка recurring payments (подписки)?
- Нужен ли instant payout или settlement window (T+1, T+2)?
- Какие требования к аудиту? Сколько лет хранить?
- Нужна ли поддержка 3D Secure / SCA?

Ответы на эти вопросы определяют архитектуру. Например, если 20 000 платёжных команд в секунду на пике, и каждая команда требует 4 синхронных операций с состоянием, это 86 400 state ops/s . Если один шард выдерживает 3000 ops/s при tail latency, нужно минимум 48 шардов (при бюджете 60% capacity) . Это только первая оценка — нужно добавить skew, replication, failover headroom.

Клеппман в «Высоконагруженных приложениях» объясняет, что требования определяют архитектуру. Без требований ты проектируешь в вакууме. И выбор зависит от того, какие требования у тебя в системе.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» (Volume 2, Chapter 11: Payment System) показывает, что платёжная система — это классическая задача на согласованность и идемпотентность .

**Схема 21.1 — Общая картина системы (высокоуровневая)**

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│ Пользователь │         │ Продавец     │         │ Админ        │
│ (checkout)   │         │ (webhook)    │         │ (dashboard)  │
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       │                        │                        │
       └────────────────────────┼────────────────────────┘
                                │ HTTPS
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    API Gateway / L7 LB                              │
│         TLS termination, rate limiting, auth                        │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
┌─────────────────────────┐   ┌─────────────────────────┐
│   Payment Service #1    │   │   Payment Service #2    │
│   (Go)                  │   │   (Go)                  │
│   - create payment      │   │   - create payment      │
│   - status              │   │   - status              │
│   - idempotency         │   │   - idempotency         │
└───────────┬─────────────┘   └───────────┬─────────────┘
            │                             │
            └──────────────┬──────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  PostgreSQL   │  │  Redis        │  │    Kafka      │
│  (ledger,     │  │  (idempotency │  │  (events,     │
│   payments,   │  │   cache,      │  │   outbox)     │
│   outbox)     │  │   rate limit) │  │               │
└───────────────┘  └───────────────┘  └───────┬───────┘
                                               │
                                               ▼
                                       ┌───────────────┐
                                       │  Webhook      │
                                       │  Workers      │
                                       │  (delivery)   │
                                       └───────────────┘
                                               │
                                               ▼
                                       ┌───────────────┐
                                       │  External     │
                                       │  Payment      │
                                       │  Provider     │
                                       │  (Stripe)     │
                                       └───────────────┘
```

---

### 21.2 Оценка нагрузки: транзакции, состояние, события

Теперь у нас есть требования. Давай оценим нагрузку.

**Платежи.**
- 20 000 оригинальных платёжных команд в секунду на пике (reference sizing) .
- После дедупликации: 1.08 API executions на команду = 21 600 executions/s .
- Синхронные операции с состоянием: 4 на команду = 86 400 state ops/s .
- Ledger entries: 2 на авторизацию + 2 на capture для 70% платежей = 68 000 entries/s .
- Domain events: 1.5 на команду = 30 000 events/s .

**Шардирование.**
- Если один state shard выдерживает 3000 ops/s при требуемом tail latency, и только 60% capacity бюджетируется для нормальной работы: N = ceil(86 400 / (3000 * 0.60)) = 48 шардов .
- Это только первая оценка. Нужно добавить skew, storage, indexes, replication, failover headroom, migration bandwidth, regional evacuation. 48-шардовый флот, которому нужно 48 шардов для выживания обычного пика, не имеет failure margin .

**Вебхуки.**
- Если 2% доставок требуют 3 дополнительных попыток, и среднее количество endpoint'ов на событие = f, то attempts/s = E * f * (1 + 0.02 * 3) .
- Это связывает retry policy с queue и egress cost.

**Объём данных.**
- Один платёж: payment_id (20 байт), amount (8 байт), currency (3 байта), status (10 байт), created_at (8 байт), metadata (100 байт). Итого — примерно 150 байт.
- 20 000 платежей/с * 150 байт = 3 МБ/с. За день — 260 ГБ. За год — 95 ТБ.
- Ledger entries: 68 000/s * 50 байт = 3.4 МБ/с. За день — 294 ГБ. За год — 107 ТБ.
- С репликацией и индексами — в 3-5 раз больше.

Вывод: система state-heavy (86 400 state ops/s) и ledger-heavy (68 000 entries/s). Нужно шардирование по merchant/account identifier. Не шардировать double-entry транзакцию across independent authorities, если ledger protocol не поддерживает atomic cross-shard posting. Безопаснее назначать все entries одной ledger транзакции на один accounting partition и выводить cross-partition settlement через paired, reconcilable transfer records .

Клеппман в «Высоконагруженных приложениях» объясняет, что оценки определяют, где bottleneck. Без оценок ты не знаешь, нужен ли шардинг, кеш, очередь. И выбор зависит от того, какие числа у тебя в системе.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как оценки влияют на архитектуру. Если у тебя state-heavy, нужен шардинг. Если ledger-heavy, нужна двойная запись.

**Схема 21.2 — Потоки данных: платёж, ledger, вебхук**

```
PAYMENT PATH (создание платежа):

Клиент ──POST /payments──▶ Payment Service
                              │
                              ├──1. Проверить idempotency key
                              │      ├── если есть COMPLETED: вернуть результат
                              │      ├── если есть PENDING: ждать или вернуть 409
                              │      └── если нет: INSERT PENDING (unique constraint)
                              ├──2. Вызвать external provider (Stripe)
                              │      ├── success ──▶ записать ledger entries
                              │      └── failed ──▶ обновить status = failed
                              ├──3. UPDATE idempotency key = COMPLETED
                              ├──4. INSERT в outbox (событие payment_completed)
                              └──5. Вернуть результат клиенту

LEDGER PATH (двойная запись):

Payment Service ──▶ PostgreSQL (ledger)
                        │
                        ├── BEGIN TRANSACTION
                        ├── INSERT debit (merchant account, amount)
                        ├── INSERT credit (platform fee account, fee)
                        ├── INSERT credit (provider account, amount - fee)
                        ├── UPDATE balances
                        ├── INSERT outbox event
                        └── COMMIT

WEBHOOK PATH (уведомление продавца):

Outbox ──▶ Kafka (payment_events) ──▶ Webhook Workers
                                            │
                                            ├──1. Получить event
                                            ├──2. Найти endpoint продавца
                                            ├──3. Отправить HTTP POST с signature
                                            ├──4. Ждать ACK (2xx)
                                            ├──5. Если ACK: обновить delivery status
                                            └──6. Если нет: retry с exponential backoff
```

---

### 21.3 API: платёж, статус, вебхуки

API — это контракт между клиентом и сервером. Давай определим endpoints.

**Создание платежа.**
```
POST /api/v1/payments
Headers: {
  "Idempotency-Key": "key-42"
}
Body: {
  "amount": 1000,
  "currency": "USD",
  "payment_method": "card_visa_4242",
  "merchant_id": "merchant_123",
  "metadata": {"order_id": "order_456"}
}
Response: {
  "payment_id": "pay_789",
  "status": "pending",  // pending, authorized, captured, failed, refunded
  "amount": 1000,
  "currency": "USD",
  "created_at": "2024-01-15T10:30:45Z"
}
```

**Получение статуса.**
```
GET /api/v1/payments/{payment_id}
Response: {
  "payment_id": "pay_789",
  "status": "captured",
  "amount": 1000,
  "currency": "USD",
  "ledger_entries": [
    {"account": "merchant_123", "type": "debit", "amount": 1000},
    {"account": "platform_fee", "type": "credit", "amount": 30},
    {"account": "provider", "type": "credit", "amount": 970}
  ]
}
```

**Вебхук (отправляется продавцу).**
```
POST {merchant_webhook_url}
Headers: {
  "X-Signature": "sha256=...",
  "X-Event-Id": "evt_001"
}
Body: {
  "event_id": "evt_001",
  "event_type": "payment.captured",
  "payment_id": "pay_789",
  "amount": 1000,
  "currency": "USD",
  "created_at": "2024-01-15T10:30:45Z"
}
```

Важно: `Idempotency-Key` должен быть уникальным для каждой логической операции, а не для каждой попытки retry. Он должен быть derived from the event, а не generated at processing time. Например, `payment:{orderId}` или `event.getIdempotencyKey()` . Не использовать random UUID в consumer (новый UUID на каждый retry = нет дедупликации). Не использовать timestamp alone (два события в одну миллисекунду = коллизия) .

Клеппман в «Высоконагруженных приложениях» объясняет, что API — это контракт. Он определяет, как клиент и сервер взаимодействуют. И выбор стиля API (REST, gRPC) зависит от того, кто клиенты и какие требования к производительности.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как API влияет на архитектуру. Если у тебя платежи, идемпотентность обязательна.

**Схема 21.3 — API endpoints и их взаимодействие с компонентами**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         API Gateway / LB                            │
│                                                                     │
│  POST /api/v1/payments ──────▶ Payment Service ──▶ PostgreSQL      │
│  (Idempotency-Key)                  │              (payments, ledger)│
│                                     ├──▶ Redis (idempotency cache)   │
│                                     ├──▶ External Provider (Stripe)  │
│                                     └──▶ Kafka (outbox events)      │
│                                                                     │
│  GET /api/v1/payments/{id} ──▶ Payment Service ──▶ PostgreSQL      │
│                                     │              (payments)        │
│                                     └──▶ Redis (cache)              │
│                                                                     │
│  POST /webhook (merchant) ◀─── Webhook Workers ◀── Kafka           │
│                                     │                               │
│                                     └──▶ Retry / DLQ               │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 21.4 Проектирование данных: двойная запись, идемпотентность, аудит

Теперь давай спроектируем данные. Нам нужны таблицы для платежей, ledger и идемпотентности.

**Схема.**
```sql
CREATE TABLE payments (
  id BIGSERIAL PRIMARY KEY,
  payment_id VARCHAR(20) UNIQUE NOT NULL,
  merchant_id BIGINT NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  currency VARCHAR(3) NOT NULL,
  status VARCHAR(20) DEFAULT 'pending',  -- pending, authorized, captured, failed, refunded
  provider_id VARCHAR(100),  -- external provider's ID
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_merchant ON payments(merchant_id, created_at DESC);
CREATE INDEX idx_payments_status ON payments(status) WHERE status IN ('pending', 'authorized');

CREATE TABLE ledger_entries (
  id BIGSERIAL PRIMARY KEY,
  entry_id VARCHAR(20) UNIQUE NOT NULL,
  payment_id VARCHAR(20) NOT NULL,
  account_id BIGINT NOT NULL,
  entry_type VARCHAR(10) NOT NULL,  -- 'debit', 'credit'
  amount DECIMAL(10,2) NOT NULL,
  currency VARCHAR(3) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ledger_payment ON ledger_entries(payment_id);
CREATE INDEX idx_ledger_account ON ledger_entries(account_id, created_at DESC);

CREATE TABLE idempotency_keys (
  key VARCHAR(255) PRIMARY KEY,
  payment_id VARCHAR(20),
  status VARCHAR(20) NOT NULL,  -- 'PENDING', 'COMPLETED', 'FAILED'
  response_body JSONB,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_idempotency_status ON idempotency_keys(status, created_at);

CREATE TABLE outbox (
  id BIGSERIAL PRIMARY KEY,
  event_id VARCHAR(20) UNIQUE NOT NULL,
  event_type VARCHAR(50) NOT NULL,
  aggregate_id VARCHAR(20) NOT NULL,
  payload JSONB NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  published_at TIMESTAMP
);

CREATE INDEX idx_outbox_unpublished ON outbox(created_at) WHERE published_at IS NULL;
```

**Двойная запись (double-entry).**
Каждый платёж создаёт две или более ledger entries. Сумма debit'ов равна сумме credit'ов. Это гарантирует, что деньги не создаются и не исчезают. Например, платёж 1000 USD:
- Debit: merchant account 1000
- Credit: platform fee account 30
- Credit: provider account 970
Сумма debit = 1000, сумма credit = 1000. Баланс сходится.

**Идемпотентность.**
Два подхода:
- Redis SETNX с TTL: быстрый (~1ms), но volatile. Подходит для payment consumers с коротким retry window (часы-дни) .
- Database unique constraint: persistent, audit trail, slower (~5-20ms). Подходит для financial transactions, требующих full history .
- Гибрид: Redis как primary cache, database как authoritative store. Redis: fast first-check. Database: fallback if Redis miss .

Двухфазный подход: PENDING → COMPLETED. При crash before gateway call: key PENDING, expires/staleness check, retry processes correctly. При crash during gateway call: key PENDING, gateway deduplicates on retry, exactly once. При crash after gateway, before SET COMPLETED: key PENDING, retry calls gateway again with same idempotency key, gateway returns original result, order confirmed .

**Аудит.**
Ledger entries immutable (append-only). Payments имеют audit trail через outbox events. Все изменения логируются. Retention: 7 лет (financial compliance).

Клеппман в «Высоконагруженных приложениях» объясняет, что проектирование данных — это компромисс между нормализацией и денормализацией, между шардированием и cross-shard запросами. И выбор зависит от того, какие запросы у тебя в системе.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как проектирование данных влияет на архитектуру. Если у тебя платежи, двойная запись и идемпотентность обязательны.

**Схема 21.4 — Схема данных: payments, ledger, idempotency, outbox**

```
PostgreSQL (payments) — шардирован по merchant_id
┌─────────────────────────────────────────────────────────────┐
│ payments                                                    │
├─────────────────────────────────────────────────────────────┤
│ id BIGSERIAL PK                                             │
│ payment_id VARCHAR(20) UNIQUE                               │
│ merchant_id BIGINT (shard key)                              │
│ amount DECIMAL(10,2)                                        │
│ currency VARCHAR(3)                                         │
│ status VARCHAR(20)                                          │
│ provider_id VARCHAR(100)                                    │
│ created_at TIMESTAMP                                        │
│ updated_at TIMESTAMP                                        │
└─────────────────────────────────────────────────────────────┘

PostgreSQL (ledger_entries) — шардирован по account_id
┌─────────────────────────────────────────────────────────────┐
│ ledger_entries                                              │
├─────────────────────────────────────────────────────────────┤
│ id BIGSERIAL PK                                             │
│ entry_id VARCHAR(20) UNIQUE                                 │
│ payment_id VARCHAR(20)                                      │
│ account_id BIGINT (shard key)                               │
│ entry_type VARCHAR(10)  -- debit/credit                     │
│ amount DECIMAL(10,2)                                        │
│ currency VARCHAR(3)                                         │
│ created_at TIMESTAMP                                        │
└─────────────────────────────────────────────────────────────┘

PostgreSQL (idempotency_keys) — глобальная таблица (или шардирована по key)
┌─────────────────────────────────────────────────────────────┐
│ idempotency_keys                                            │
├─────────────────────────────────────────────────────────────┤
│ key VARCHAR(255) PRIMARY KEY                                │
│ payment_id VARCHAR(20)                                      │
│ status VARCHAR(20)  -- PENDING, COMPLETED, FAILED           │
│ response_body JSONB                                         │
│ created_at TIMESTAMP                                        │
│ updated_at TIMESTAMP                                        │
└─────────────────────────────────────────────────────────────┘

PostgreSQL (outbox) — шардирован по aggregate_id
┌─────────────────────────────────────────────────────────────┐
│ outbox                                                      │
├─────────────────────────────────────────────────────────────┤
│ id BIGSERIAL PK                                             │
│ event_id VARCHAR(20) UNIQUE                                 │
│ event_type VARCHAR(50)                                      │
│ aggregate_id VARCHAR(20) (shard key)                        │
│ payload JSONB                                               │
│ created_at TIMESTAMP                                        │
│ published_at TIMESTAMP                                      │
└─────────────────────────────────────────────────────────────┘
```

---

### 21.5 Транзакции: ACID, изоляция, блокировки

Теперь давай разберём транзакции. В платёжной системе ACID критичен.

**Atomicity.** Транзакция выполняется целиком или не выполняется вообще. Если одна ledger entry fails, вся транзакция откатывается. Нет частично применённых транзакций.

**Consistency.** Транзакция переводит базу данных из одного согласованного состояния в другое. Сумма debit'ов равна сумме credit'ов. Ограничения целостности соблюдаются.

**Isolation.** Транзакции выполняются независимо. Уровень изоляции: SERIALIZABLE или REPEATABLE READ для платёжных транзакций. READ COMMITTED недостаточно, потому что может привести к phantom reads или non-repeatable reads.

**Durability.** После коммита изменения сохраняются, даже если система падает. PostgreSQL с synchronous_commit = on гарантирует это.

**Блокировки.**
- Row-level locks: `SELECT ... FOR UPDATE` для блокировки строки платежа на время транзакции.
- Advisory locks: для координации между воркерами (например, чтобы только один воркер обрабатывал платёж).
- Optimistic concurrency: version column, compare-and-set. Два воркера race to transition payment. One compare-and-set from version 3 to 4 succeeds; the other observes version 4 and becomes a no-op .

Клеппман в «Высоконагруженных приложениях» объясняет, что транзакции — это компромисс между согласованностью и производительностью. SERIALIZABLE безопаснее, но медленнее. READ COMMITTED быстрее, но может привести к аномалиям. И выбор зависит от того, насколько критична согласованность.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как транзакции влияют на архитектуру. Если у тебя платежи, ACID обязателен.

**Схема 21.5 — Транзакции: ACID, изоляция, блокировки**

```
PAYMENT TRANSACTION:

BEGIN TRANSACTION;
  -- 1. Lock payment row
  SELECT * FROM payments WHERE payment_id = 'pay_789' FOR UPDATE;
  
  -- 2. Check idempotency
  SELECT * FROM idempotency_keys WHERE key = 'key-42' FOR UPDATE;
  -- если status = COMPLETED: COMMIT; return response
  
  -- 3. Insert ledger entries (double-entry)
  INSERT INTO ledger_entries (entry_id, payment_id, account_id, entry_type, amount)
  VALUES 
    ('entry_1', 'pay_789', 'merchant_123', 'debit', 1000),
    ('entry_2', 'pay_789', 'platform_fee', 'credit', 30),
    ('entry_3', 'pay_789', 'provider', 'credit', 970);
  
  -- 4. Update balances
  UPDATE accounts SET balance = balance - 1000 WHERE account_id = 'merchant_123';
  UPDATE accounts SET balance = balance + 30 WHERE account_id = 'platform_fee';
  UPDATE accounts SET balance = balance + 970 WHERE account_id = 'provider';
  
  -- 5. Insert outbox event
  INSERT INTO outbox (event_id, event_type, aggregate_id, payload)
  VALUES ('evt_001', 'payment.captured', 'pay_789', '{"amount": 1000}');
  
  -- 6. Update idempotency key
  UPDATE idempotency_keys SET status = 'COMPLETED', response_body = '...' WHERE key = 'key-42';
COMMIT;

ISOLATION LEVELS:

READ COMMITTED:
- Видит только закоммиченные данные.
- Может видеть phantom reads (новые строки).
- Недостаточно для платежей.

REPEATABLE READ:
- Видит snapshot данных на момент начала транзакции.
- Не видит phantom reads.
- Достаточно для большинства платежей.

SERIALIZABLE:
- Полная изоляция, как если бы транзакции выполнялись последовательно.
- Самый безопасный, но медленный.
- Для критичных платежей.

OPTIMISTIC CONCURRENCY:

Worker 1: SELECT version FROM payments WHERE payment_id = 'pay_789'; -- version=3
Worker 2: SELECT version FROM payments WHERE payment_id = 'pay_789'; -- version=3

Worker 1: UPDATE payments SET version=4, status='captured' WHERE payment_id='pay_789' AND version=3;
-- 1 row affected (success)

Worker 2: UPDATE payments SET version=4, status='captured' WHERE payment_id='pay_789' AND version=3;
-- 0 rows affected (version уже 4)
-- Worker 2 становится no-op.
```

---

### 21.6 Идемпотентность: ключи, PENDING→COMPLETED, дедупликация

Теперь давай разберём идемпотентность подробнее. Это самая критичная часть платёжной системы.

**Idempotency key.**
Клиент генерирует уникальный ключ для каждой логической операции. Например, `payment:order_456`. Ключ передаётся в заголовке `Idempotency-Key`. Сервер использует ключ для дедупликации.

Правила:
- Ключ должен быть unique per logical operation, not per retry attempt .
- Ключ должен быть derived from the event, not generated at processing time .
- Не использовать random UUID в consumer (новый UUID на каждый retry = нет дедупликации) .
- Не использовать timestamp alone (два события в одну миллисекунду = коллизия) .

**PENDING → COMPLETED.**
Двухфазный подход к идемпотентности:
1. При получении запроса: попытаться INSERT PENDING row с unique constraint. Если duplicate key — значит, операция уже в процессе или завершена.
2. Если INSERT успешен: мы владеем операцией. Выполнить операцию (вызвать gateway). Затем UPDATE status = COMPLETED (или FAILED).
3. Если INSERT failed (duplicate): прочитать existing row. Если status = COMPLETED: вернуть сохранённый response. Если status = PENDING: ждать (с backoff) или вернуть 409 Conflict.

Этот подход обрабатывает crash scenarios:
- Crash before gateway call: key PENDING, expires/staleness check, retry processes correctly .
- Crash during gateway call: key PENDING, gateway deduplicates on retry, exactly once .
- Crash after gateway, before SET COMPLETED: key PENDING, retry calls gateway again with same idempotency key, gateway returns original result, order confirmed .
- Crash after SET COMPLETED: key COMPLETED, retry skips, correct .

**Redis vs Database.**
- Redis SETNX с TTL: fastest (~1ms), volatile. OK if redelivery window is short. Use: payment consumers with short retry windows .
- Database unique constraint: persistent audit trail, slower (~5-20ms). Use: financial transactions requiring full history .
- Both: Redis as primary cache, Database as authoritative store. Redis: fast first-check. DB: fallback if Redis miss. Best of both worlds .

**Kafka exactly-once не спасает.**
Kafka EOS guarantees exactly-once between Kafka topics. The moment you call an external API (payment gateway), you're outside Kafka. No transaction can span Kafka + external HTTP call. For external side effects, idempotency key is the only correct solution .

Клеппман в «Высоконагруженных приложениях» объясняет, что идемпотентность — это способ достичь exactly-once processing в системах с at-least-once доставкой. Вместо того чтобы гарантировать, что сообщение доставлено ровно один раз (что сложно), ты гарантируешь, что повторная обработка не меняет результат. И выбор зависит от того, насколько критично дублирование.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как идемпотентность влияет на архитектуру. Если у тебя платежи, идемпотентность обязательна.

**Схема 21.6 — Идемпотентность: PENDING→COMPLETED, дедупликация**

```
IDEMPOTENCY FLOW:

Request (Idempotency-Key: key-42)
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  1. INSERT INTO idempotency_keys (key, status)             │
│     VALUES ('key-42', 'PENDING')                            │
│     ON CONFLICT (key) DO NOTHING;                           │
│                                                             │
│  Если INSERT успешен (1 row):                               │
│  ├── Мы владеем операцией.                                  │
│  ├── Выполнить платёж (вызвать gateway).                   │
│  ├── UPDATE idempotency_keys SET status='COMPLETED',        │
│  │   response_body='...' WHERE key='key-42';                │
│  └── Вернуть response.                                      │
│                                                             │
│  Если INSERT failed (0 rows):                               │
│  ├── Прочитать existing row.                                │
│  ├── Если status = 'COMPLETED': вернуть response_body.      │
│  ├── Если status = 'PENDING':                               │
│  │   ├── Ждать (с backoff) до COMPLETED/FAILED.            │
│  │   └── Или вернуть 409 Conflict.                          │
│  └── Если status = 'FAILED': вернуть ошибку.                │
└─────────────────────────────────────────────────────────────┘

CRASH SCENARIOS:

1. Crash before gateway call:
   - idempotency_keys: PENDING (expires/staleness check)
   - Retry: SETNX succeeds (PENDING истёк) → processes correctly.

2. Crash during gateway call:
   - idempotency_keys: PENDING
   - Retry: gateway deduplicates on retry (same idempotency key) → exactly once.

3. Crash after gateway, before SET COMPLETED:
   - idempotency_keys: PENDING
   - Retry: calls gateway again with same idempotency key → gateway returns original result → order confirmed.

4. Crash after SET COMPLETED:
   - idempotency_keys: COMPLETED
   - Retry: skips → correct.

REDIS VS DATABASE:

Redis SETNX (TTL: 24h):
├── Fast (~1ms)
├── Volatile (lost on restart)
└── Use: short retry window (hours-days)

Database unique constraint:
├── Persistent (survives restart)
├── Audit trail
├── Slower (~5-20ms)
└── Use: financial transactions (full history)

Both:
├── Redis: fast first-check
└── Database: authoritative store, fallback
```

---

### 21.7 Обработка сбоев: timeout, crash, duplicate callback

Теперь давай разберём конкретный failure trace. Это reference-design trace из документации .

**Сценарий: timeout, crash, duplicate callback.**

1. Merchant отправляет confirm с key k-42.
2. Platform записывает rail attempt a-1, затем отправляет его процессору.
3. Процессор авторизует charge, но response теряется.
4. Connector deadline истекает и записывает outcome_unknown.
5. До ответа orchestrator падает. Merchant видит connection reset.
6. Merchant повторяет с k-42; idempotency layer attaches it to the existing operation instead of creating a-2.
7. Processor callback reports a-1=authorized; recovery query reports the same result.
8. Both workers race to transition the payment. One compare-and-set from version 3 to 4 succeeds; the other observes version 4 and becomes a no-op.
9. The winning transaction posts the balanced ledger movement and outbox event atomically.
10. The webhook worker delivers the event twice because its first acknowledgement is lost. The merchant deduplicates by event ID.

The design tolerates duplicate messages at every boundary while committing one payment transition and one economic effect .

**Вот как это выглядит на схеме.**

Клеппман в «Высоконагруженных приложениях» объясняет, что обработка сбоев — это компромисс между надёжностью и сложностью. Каждый сбой требует дополнительной логики (retry, idempotency, reconciliation). И выбор зависит от того, насколько критична система.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как обработка сбоев влияет на архитектуру. Если у тебя платежи, обработка сбоев обязательна.

**Схема 21.7 — Обработка сбоев: timeout, crash, duplicate callback**

```
FAILURE TRACE:

1. Merchant ──confirm(k-42)──▶ Platform
                                    │
2. Platform ──rail attempt a-1──▶ Processor
                                    │
3. Processor: charge authorized, но response lost.
                                    │
4. Connector deadline expires: outcome_unknown.
                                    │
5. Orchestrator crashes. Merchant: connection reset.
                                    │
6. Merchant ──retry(k-42)──▶ Platform
                                    │
   Idempotency layer: attaches to existing operation (a-1),
   не создаёт a-2.
                                    │
7. Processor callback: a-1=authorized.
   Recovery query: same result.
                                    │
8. Two workers race to transition payment:
   Worker 1: UPDATE payments SET version=4, status='captured'
             WHERE payment_id='pay_789' AND version=3;
   → 1 row affected (success)
   
   Worker 2: UPDATE payments SET version=4, status='captured'
             WHERE payment_id='pay_789' AND version=3;
   → 0 rows affected (version уже 4)
   → no-op.
                                    │
9. Winning transaction: posts balanced ledger movement
   + outbox event atomically.
                                    │
10. Webhook worker delivers event twice (first ACK lost).
    Merchant deduplicates by event ID.

RESULT:
- Duplicate messages at every boundary.
- One payment transition.
- One economic effect.
```

---

### 21.8 Вебхуки и reconciliation

Теперь давай разберём вебхуки и reconciliation.

**Вебхуки.**
- Каждый event имеет immutable event ID.
- Каждый subscribed endpoint имеет свои delivery rows, attempt counter, next-attempt time, terminal state, response evidence.
- Delivery is at-least-once. Consumers deduplicate by event ID.
- Retries use exponential backoff with jitter and finite retention policy. Slow endpoints cannot hold payment locks .
- Stripe's published webhook guidance requires endpoint signature verification and warns that event ordering is not guaranteed .

**Reconciliation.**
- Периодически (например, каждую ночь) сравнивать ledger с external provider's records.
- Если есть расхождения: создать reconciliation report, investigate, исправить.
- Reconciliation должен быть automated, но с manual review для сложных случаев.

Клеппман в «Высоконагруженных приложениях» объясняет, что вебхуки и reconciliation — это компромисс между real-time и batch. Вебхуки дают near-real-time уведомления, но могут теряться. Reconciliation даёт гарантию, но с задержкой. И выбор зависит от того, что важнее для задачи.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как вебхуки и reconciliation влияют на архитектуру. Если у тебя платежи, reconciliation обязателен.

**Схема 21.8 — Вебхуки и reconciliation**

```
WEBHOOK DELIVERY:

Outbox ──▶ Kafka ──▶ Webhook Workers
                          │
                          ├──1. Получить event (event_id, payload)
                          ├──2. Найти endpoint (merchant_webhook_url)
                          ├──3. Отправить HTTP POST с signature (X-Signature)
                          ├──4. Ждать ACK (2xx)
                          ├──5. Если ACK: обновить delivery status
                          └──6. Если нет: retry с exponential backoff + jitter

Delivery rows:
┌─────────────────────────────────────────────────────────────┐
│ event_id │ endpoint │ attempt │ next_attempt │ state       │
├──────────┼──────────┼─────────┼──────────────┼─────────────┤
│ evt_001  │ ep_1     │ 1       │ now          │ pending     │
│ evt_001  │ ep_2     │ 0       │ now          │ pending     │
│ evt_002  │ ep_1     │ 3       │ now+8s       │ failed      │
└─────────────────────────────────────────────────────────────┘

RECONCILIATION:

Nightly job:
┌─────────────────────────────────────────────────────────────┐
│ 1. Fetch ledger entries from our DB (last 24h)              │
│ 2. Fetch transactions from provider (Stripe API)            │
│ 3. Compare by payment_id / provider_id                      │
│ 4. If match: OK                                             │
│ 5. If mismatch:                                             │
│    ├── create reconciliation report                         │
│    ├── flag for manual review                               │
│    └── (if automated) fix if safe                           │
└─────────────────────────────────────────────────────────────┘
```

---

### 21.9 Отказоустойчивость, мониторинг и безопасность (PCI DSS)

Теперь давай разберём, как обеспечить отказоустойчивость, мониторинг и безопасность.

**Отказоустойчивость.**
- Failover: если Payment Service упал, клиенты переподключаются к другому экземпляру.
- Circuit breaker: если external provider недоступен, быстро вернуть ошибку.
- Graceful degradation: если provider недоступен, платежи ставятся в очередь (pending) и обрабатываются позже.
- Retry: с exponential backoff для временных сбоев.
- Timeout: все запросы должны иметь timeout (5 секунд для provider call).

**Мониторинг.**
- Метрики: payment success rate, payment latency (p50, p95, p99), idempotency hit rate, ledger imbalance, provider error rate, webhook delivery rate.
- Логи: структурированные, с trace ID, уровни.
- Трейсинг: OpenTelemetry, Jaeger.
- Алерты: на симптомы (success rate < 99%, p99 latency > 1s, ledger imbalance > 0, provider error rate > 1%).

**Безопасность (PCI DSS).**
- Не хранить PAN (Primary Account Number). Использовать токенизацию (Stripe tokens).
- Шифрование: TLS для транспорта, encryption at rest для базы данных.
- Аутентификация: API keys для merchant, JWT для пользователей.
- Авторизация: merchant может видеть только свои платежи.
- Rate limiting: ограничение создания платежей (например, 1000 в минуту).
- Fraud detection: проверка аномалий (слишком много платежей, необычные суммы).
- Секреты: хранение в Vault, ротация.
- Аудит: логирование всех операций, immutable, retention 7 лет.

Клеппман в «Высоконагруженных приложениях» объясняет, что отказоустойчивость, мониторинг и безопасность — это не фичи, а свойства системы. Без них система не может эксплуатироваться. И выбор зависит от того, насколько критична система.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как отказоустойчивость, мониторинг и безопасность влияют на архитектуру. Если у тебя платежи, они обязательны.

**Схема 21.9 — Полная архитектура с отказоустойчивостью и мониторингом**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Observability Stack                         │
│  Prometheus ──▶ Grafana (dashboards)                                │
│  Loki ──▶ Grafana (logs)                                            │
│  Jaeger/Tempo ──▶ Grafana (traces)                                  │
│  Alertmanager ──▶ PagerDuty/Slack (alerts)                          │
└─────────────────────────────────────────────────────────────────────┘
                                  ▲
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                         Payment Service                             │
│  ┌──────────────────────────────┼──────────────────────────────┐   │
│  │  Circuit Breaker             │                              │   │
│  │  ├── PostgreSQL: CLOSED/OPEN/HALF-OPEN                      │   │
│  │  ├── Redis: CLOSED/OPEN/HALF-OPEN                           │   │
│  │  └── External Provider: CLOSED/OPEN/HALF-OPEN               │   │
│  │                                                             │   │
│  │  Graceful Degradation:                                      │   │
│  │  ├── Redis недоступен: idempotency через DB (медленнее)     │   │
│  │  ├── Provider недоступен: платежи в pending (retry позже)   │   │
│  │  └── Kafka недоступен: outbox не публикуется (retry позже)  │   │
│  │                                                             │   │
│  │  Timeout: 5s для provider call, 100ms для DB                │   │
│  │  Retry: exponential backoff + jitter                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                         ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│ PostgreSQL    │         │ Redis         │         │ Kafka Cluster │
│ Master        │         │ (idempotency, │         │ (3 brokers)   │
│      │        │         │  rate limit)  │         │               │
│      ▼        │         │               │         │ topic:        │
│ Replica 1     │         │               │         │ payment_events│
│ Replica 2     │         │               │         │               │
└───────────────┘         └───────────────┘         └───────┬───────┘
                                                            │
                                                            ▼
                                                    ┌───────────────┐
                                                    │ Webhook       │
                                                    │ Workers       │
                                                    │ (3 replicas)  │
                                                    │               │
                                                    │ - delivery    │
                                                    │ - retry       │
                                                    │ - DLQ         │
                                                    └───────────────┘

Security Layer:
┌─────────────────────────────────────────────────────────────────────┐
│  WAF (Cloudflare) ──▶ Rate Limiter (Redis) ──▶ API Keys (merchant) │
│  Vault (secrets) ──▶ Tokenization (Stripe) ──▶ Fraud Detection     │
│  Audit Log (append-only, 7 years)                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Выводы

В этой главе ты узнал, как строить платёжную систему. Ты разобрал транзакции и ACID, идемпотентность и ключи, обработку сбоев и reconciliation. Ты узнал, что такое двойная запись и почему она критична для аудита. Ты узнал, как обрабатывать duplicate callback и почему Kafka exactly-once не спасает при вызове внешнего API.

Главное, что ты должен вынести из этой главы: **payment system — это задача о строгой согласованности и надёжности, где ошибка стоит денег.** Хорошая архитектура — не та, которая использует самую модную очередь. Хорошая архитектура — та, которая гарантирует, что деньги не потеряются и не задвоятся, даже когда всё идёт не так.

---

## Выжимка для быстрого повторения

- **Требования:** создание платежа, статус, вебхуки, возвраты. Строгая согласованность, идемпотентность, долговечность, аудит, PCI DSS.
- **Оценки:** 20 000 платёжных команд/с = 86 400 state ops/s, 68 000 ledger entries/s, 30 000 events/s. Шардирование: минимум 48 шардов (при 3000 ops/s на шард, 60% capacity).
- **API:** POST /payments с `Idempotency-Key`, GET /payments/{id}, POST webhook с signature. Ключ unique per logical operation, derived from event.
- **Данные:** payments (шардирован по merchant_id), ledger_entries (шардирован по account_id, double-entry), idempotency_keys (PENDING/COMPLETED/FAILED), outbox.
- **Транзакции:** ACID, SERIALIZABLE/REPEATABLE READ, row locks, optimistic concurrency (version column).
- **Идемпотентность:** PENDING → COMPLETED. Redis SETNX (fast, volatile) vs Database unique constraint (persistent, audit). Both: Redis first-check, DB authoritative.
- **Сбои:** timeout, crash, duplicate callback. Idempotency key + compare-and-set + outbox + dedup by event ID. Kafka EOS не спасает при external API.
- **Вебхуки:** at-least-once, retry с exponential backoff, signature verification, dedup by event ID. Reconciliation nightly.
- **Безопасность:** tokenization (no PAN), TLS, encryption at rest, fraud detection, audit log (7 years).

---

## Вопросы для самопроверки

1. Почему идемпотентность критична для платёжной системы? Как работает PENDING → COMPLETED?
2. Почему Kafka exactly-once не спасает при вызове внешнего платёжного API?
3. Что такое двойная запись и почему она важна для аудита?
4. Как обрабатывать duplicate callback от платёжного провайдера?
5. Как обеспечить отказоустойчивость платёжной системы?

---

## Эталонные ответы

**1. Почему идемпотентность критична для платёжной системы? Как работает PENDING → COMPLETED?**

Идемпотентность критична, потому что клиент может повторить запрос (timeout, network error), и повторный запрос не должен создать дубликат платежа. PENDING → COMPLETED: при получении запроса попытаться INSERT PENDING row с unique constraint. Если INSERT успешен — мы владеем операцией, выполняем платёж, затем UPDATE status = COMPLETED. Если INSERT failed (duplicate) — читаем existing row. Если COMPLETED: возвращаем сохранённый response. Если PENDING: ждём или возвращаем 409. Это обрабатывает crash scenarios: crash before gateway (PENDING expires, retry processes), crash during gateway (gateway deduplicates on retry), crash after gateway before COMPLETED (retry calls gateway again, gateway returns original result).

**2. Почему Kafka exactly-once не спасает при вызове внешнего платёжного API?**

Kafka EOS guarantees exactly-once between Kafka topics. The moment you call an external API (payment gateway), you're outside Kafka. No transaction can span Kafka + external HTTP call. For external side effects, idempotency key is the only correct solution . Kafka EOS works for Kafka-to-Kafka pipelines (stream processing, topology transforms). But when you talk to an external system — a payment gateway, a database, an email service — you must implement idempotency yourself.

**3. Что такое двойная запись и почему она важна для аудита?**

Double-entry: каждый платёж создаёт две или более ledger entries. Сумма debit'ов равна сумме credit'ов. Это гарантирует, что деньги не создаются и не исчезают. Для аудита: ledger entries immutable (append-only), полная история всех транзакций. Если есть расхождение (ledger imbalance), это сигнал о bug или fraud. Reconciliation сравнивает ledger с external provider's records. Double-entry — это фундамент финансового аудита.

**4. Как обрабатывать duplicate callback от платёжного провайдера?**

Каждый event имеет immutable event ID. При получении callback: проверить, был ли уже обработан event с таким ID. Если да — игнорировать (dedup). Если нет — обработать и записать event ID. Delivery is at-least-once, consumers deduplicate by event ID. Retries use exponential backoff with jitter. Stripe's guidance requires endpoint signature verification and warns that event ordering is not guaranteed . Duplicate callback обрабатывается через idempotency (event ID) и compare-and-set (version).

**5. Как обеспечить отказоустойчивость платёжной системы?**

Failover: load balancer направляет трафик на здоровые экземпляры. Circuit breaker: если provider недоступен, быстро вернуть ошибку. Graceful degradation: если provider недоступен, платежи в pending, retry позже. Retry: exponential backoff. Timeout: 5s для provider call. Reconciliation: nightly сравнение ledger с provider. Мониторинг: success rate, latency, ledger imbalance, provider error rate. Безопасность: tokenization, TLS, encryption, fraud detection, audit log.

---

## Мост к следующей главе

Ты спроектировал платёжную систему — от требований до продакшена. Ты узнал, что такое транзакции, идемпотентность, двойная запись, обработка сбоев. Ты увидел, как те же строительные блоки (PostgreSQL, Redis, Kafka) применяются к финансовой задаче.

Но платёжная система — это про проведение денег. А что если задача — это хранение и управление деньгами? Что если пользователь должен иметь кошелёк, пополнять его, переводить деньги другим пользователям, видеть баланс и историю транзакций? Что если нужно поддерживать несколько валют, конвертацию, лимиты?

В следующей главе — **Digital Wallet: балансы, переводы, конвертация**. Ты узнаешь, как хранить балансы, как проводить переводы между пользователями, как обрабатывать конвертацию валют, как обеспечивать консистентность и аудит. Ты узнаешь, что такое ledger-based wallet и почему это лучше, чем просто поле balance в таблице. И ты поймёшь, что digital wallet — это задача о состоянии и транзакциях, которая требует понимания ledger, consistency и reconciliation.