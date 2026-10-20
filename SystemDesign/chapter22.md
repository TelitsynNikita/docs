## Глава 22: Digital Wallet: балансы, переводы, конвертация

### Оглавление главы

- **22.1** Уточнение требований: что такое кошелёк и кому он нужен
- **22.2** Оценка нагрузки: пользователи, переводы, балансы
- **22.3** API: пополнение, перевод, баланс, история
- **22.4** Проектирование данных: ledger, balances, idempotency
- **22.5** Баланс: stored vs derived vs hybrid
- **22.6** Переводы: атомарность, блокировки, порядок
- **22.7** Конвертация валют: курсы, округление, мультивалютность
- **22.8** Reconciliation: сверка ledger с балансами и внешними системами
- **22.9** Отказоустойчивость, мониторинг и безопасность

---

### Введение

В предыдущей главе мы спроектировали платёжную систему — от требований до продакшена. Ты узнал, что такое транзакции, идемпотентность, двойная запись, обработка сбоев. Ты увидел, как те же строительные блоки (PostgreSQL, Redis, Kafka) применяются к финансовой задаче. Но платёжная система — это про проведение денег между продавцом и покупателем. А что если задача — это хранение и управление деньгами?

Представь, что тебя просят спроектировать digital wallet — как Venmo, Revolut или PayPal. Пользователь регистрируется, пополняет кошелёк с карты или банковского счёта, переводит деньги другим пользователям, видит баланс и историю транзакций. И всё это — с гарантией, что деньги не потеряются, не задвоятся, и баланс всегда сходится с историей.

На первый взгляд, всё просто: у тебя есть таблица `users` с полем `balance`, ты обновляешь его при переводах. Но что если два перевода происходят одновременно? Что если пользователь пополняет кошелёк, а в это время другой пользователь переводит ему деньги? Что если баланс в кеше расходится с балансом в базе данных? Что если нужно поддерживать несколько валют и конвертировать их по курсу? Наивный подход ломается на конкурентности и согласованности.

В этой главе ты узнаешь, как строить digital wallet. Ты разберёшь ledger-based подход, балансы (stored vs derived), переводы, конвертацию валют, reconciliation. Ты узнаешь, что такое CQRS для кошелька и почему это лучше, чем просто поле `balance`. И ты поймёшь, что digital wallet — это задача о состоянии и транзакциях, которая требует понимания ledger, consistency и reconciliation.

---

### 22.1 Уточнение требований: что такое кошелёк и кому он нужен

Прежде чем рисовать диаграмму, нужно понять, что мы строим. И первый шаг — уточнить требования.

**Функциональные требования.**
- Пользователь может зарегистрироваться и создать кошелёк.
- Пользователь может пополнить кошелёк (с карты, банковского счёта).
- Пользователь может перевести деньги другому пользователю.
- Пользователь может вывести деньги на карту или счёт.
- Пользователь может видеть баланс и историю транзакций.
- Система поддерживает несколько валют (USD, EUR, RUB).
- Система конвертирует валюты по актуальному курсу.
- Система обрабатывает возвраты и отмены.

**Нефункциональные требования.**
- Строгая согласованность для ledger: баланс не может быть отрицательным (если не овердрафт), деньги не могут создаваться или исчезать.
- Идемпотентность: повторный запрос не создаёт дубликат перевода.
- Долговечность: транзакции не теряются после подтверждения.
- Доступность: 99.99% для переводов и чтения баланса.
- Низкая задержка: чтение баланса < 50 мс, перевод < 500 мс.
- Аудит: каждая операция отслеживается.
- Безопасность: PCI DSS, шифрование, fraud detection.

**Вопросы, которые нужно задать.**
- Сколько пользователей? 10 миллионов? 100 миллионов?
- Сколько переводов в секунду? 1000? 10 000?
- Какие валюты поддерживать? Только USD? Или мультивалютность?
- Нужна ли конвертация? По какому курсу (рыночный, фиксированный, с наценкой)?
- Нужна ли поддержка подписок и recurring payments?
- Какие лимиты на переводы? Дневные, месячные?
- Нужна ли поддержка бизнес-аккаунтов (merchant)?
- Какие требования к аудиту? Сколько лет хранить?

Ответы на эти вопросы определяют архитектуру. Например, если 10 000 переводов в секунду, и каждый перевод требует блокировки двух кошельков, это 20 000 блокировок в секунду. Нужно шардирование по user_id. Если поддерживается мультивалютность, нужны курсы валют и правила округления.

Клеппман в «Высоконагруженных приложениях» объясняет, что требования определяют архитектуру. Без требований ты проектируешь в вакууме. И выбор зависит от того, какие требования у тебя в системе.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» (Volume 2, Chapter 12: Digital Wallet) показывает, что digital wallet — это классическая задача на ledger и согласованность .

**Схема 22.1 — Общая картина системы (высокоуровневая)**

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│ Пользователь │         │ Пользователь │         │ Админ        │
│ (мобильное   │         │ (веб)        │         │ (dashboard)  │
│  приложение) │         │              │         │              │
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
│   Wallet Service #1     │   │   Wallet Service #2     │
│   (Go)                  │   │   (Go)                  │
│   - balance queries     │   │   - balance queries     │
│   - account management  │   │   - account management  │
└───────────┬─────────────┘   └───────────┬─────────────┘
            │                             │
            └──────────────┬──────────────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
┌─────────────────────────┐   ┌─────────────────────────┐
│   Transaction Service   │   │   Payment Service       │
│   (Go)                  │   │   (Go)                  │
│   - P2P transfers       │   │   - top-up              │
│   - orchestration       │   │   - withdrawal          │
└───────────┬─────────────┘   └───────────┬─────────────┘
            │                             │
            └──────────────┬──────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Ledger Service                              │
│   - double-entry writes                                             │
│   - enforces invariant (debits = credits)                           │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  PostgreSQL   │  │  Redis        │  │    Kafka      │
│  (ledger,     │  │  (balance     │  │  (events,     │
│   accounts,   │  │   cache,      │  │   outbox)     │
│   outbox)     │  │   idempotency)│  │               │
└───────────────┘  └───────────────┘  └───────┬───────┘
                                               │
                                               ▼
                                       ┌───────────────┐
│  Reconciliation │
│  Engine         │
│  (nightly)      │
└───────────────┘
                                               │
                                               ▼
                                       ┌───────────────┐
                                       │  External     │
                                       │  Banks /      │
                                       │  Card Networks│
                                       └───────────────┘
```

---

### 22.2 Оценка нагрузки: пользователи, переводы, балансы

Теперь у нас есть требования. Давай оценим нагрузку.

**Пользователи.**
- 100 миллионов пользователей, 10% активны в день = 10 миллионов активных пользователей.
- Каждый пользователь делает 2-3 операции в день (пополнение, перевод, вывод).
- Средняя нагрузка: 10 миллионов * 2.5 / 100 000 секунд = 250 операций в секунду.
- Пиковая нагрузка: коэффициент пиковости 3-5 = 750-1250 операций в секунду.

**Переводы.**
- 10 000 переводов в секунду на пике (reference sizing для крупного кошелька).
- Каждый перевод требует: блокировки двух кошельков, проверки баланса, двух ledger entries, обновления балансов, публикации события.
- State ops: 10 000 * 4 = 40 000 state ops/s.

**Чтение баланса.**
- Каждый пользователь проверяет баланс несколько раз в день.
- 10 миллионов пользователей * 5 проверок = 50 миллионов чтений в день = 500 чтений в секунду.
- Пиковая нагрузка: 1500-2500 чтений в секунду.

**Объём данных.**
- Один перевод: transaction_id (20 байт), sender_id (8 байт), receiver_id (8 байт), amount (8 байт), currency (3 байта), status (10 байт), created_at (8 байт). Итого — примерно 100 байт.
- Ledger entries: 2 на перевод * 100 байт = 200 байт.
- 10 000 переводов/с * 200 байт = 2 МБ/с. За день — 173 ГБ. За год — 63 ТБ.
- С индексами и репликацией — в 3-5 раз больше.

**Latency.**
- Чтение баланса: < 50 мс (из кеша).
- Перевод: < 500 мс (с блокировками и ledger).

Вывод: система balance-heavy (500+ чтений/с) и transfer-heavy (10 000 переводов/с на пике). Нужен кеш для балансов (Redis) и шардирование ledger по user_id. Чтение баланса должно идти из кеша (CQRS), а запись — через ledger.

Клеппман в «Высоконагруженных приложениях» объясняет, что оценки определяют, где bottleneck. Без оценок ты не знаешь, нужен ли шардинг, кеш, очередь. И выбор зависит от того, какие числа у тебя в системе.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как оценки влияют на архитектуру. Если у тебя balance-heavy, кеш обязателен. Если transfer-heavy, шардинг и ledger.

**Схема 22.2 — Потоки данных: пополнение, перевод, чтение баланса**

```
TOP-UP PATH (пополнение с карты):

Пользователь ──POST /topup──▶ Payment Service
                                    │
                                    ├──1. Проверить idempotency key
                                    ├──2. Вызвать external provider (Stripe)
                                    ├──3. Если success:
                                    │      ├── Ledger Service: INSERT journal_entry (TOP_UP)
                                    │      │      ├── DEBIT: external_account
                                    │      │      └── CREDIT: user_wallet
                                    │      ├── UPDATE balances (user_wallet +amount)
                                    │      ├── PUBLISH Kafka (balance_updated)
                                    │      └── INVALIDATE Redis (balance:{user_id})
                                    └──4. Вернуть результат

P2P TRANSFER PATH (перевод между пользователями):

Пользователь A ──POST /transfer──▶ Transaction Service
                                          │
                                          ├──1. Проверить idempotency key
                                          ├──2. BEGIN TRANSACTION
                                          ├──3. SELECT wallet A FOR UPDATE (lock)
                                          ├──4. SELECT wallet B FOR UPDATE (lock)
                                          ├──5. Проверить баланс A >= amount
                                          ├──6. Ledger Service: INSERT journal_entry (P2P_TRANSFER)
                                          │      ├── DEBIT: wallet A
                                          │      └── CREDIT: wallet B
                                          ├──7. UPDATE balances (A -amount, B +amount)
                                          ├──8. INSERT outbox event
                                          ├──9. COMMIT
                                          ├──10. INVALIDATE Redis (balance:A, balance:B)
                                          └──11. PUBLISH Kafka (transfer_completed)

READ BALANCE PATH (чтение баланса):

Пользователь ──GET /balance──▶ Wallet Service
                                      │
                                      ├──1. GET Redis (balance:{user_id})
                                      │      │
                                      │      ├── HIT ──▶ вернуть баланс (fast, < 50ms)
                                      │      │
                                      │      └── MISS ──▶ SELECT balances FROM PostgreSQL
                                      │                     │
                                      │                     └── SET Redis (TTL 5 min)
                                      │
                                      └──2. Вернуть баланс
```

---

### 22.3 API: пополнение, перевод, баланс, история

API — это контракт между клиентом и сервером. Давай определим endpoints.

**Пополнение.**
```
POST /api/v1/wallet/topup
Headers: {
  "Idempotency-Key": "topup-42"
}
Body: {
  "amount": 1000,
  "currency": "USD",
  "payment_method_id": "pm_123"
}
Response: {
  "transaction_id": "txn_789",
  "status": "completed",
  "amount": 1000,
  "currency": "USD",
  "new_balance": 1500
}
```

**Перевод.**
```
POST /api/v1/wallet/transfer
Headers: {
  "Idempotency-Key": "transfer-43"
}
Body: {
  "receiver_id": "user_456",
  "amount": 500,
  "currency": "USD",
  "note": "Lunch"
}
Response: {
  "transaction_id": "txn_790",
  "status": "completed",
  "amount": 500,
  "currency": "USD",
  "sender_balance": 1000,
  "receiver_balance": 2000
}
```

**Чтение баланса.**
```
GET /api/v1/wallet/balance?currency=USD
Response: {
  "user_id": "user_123",
  "balances": [
    {"currency": "USD", "available": 1000, "pending": 0},
    {"currency": "EUR", "available": 500, "pending": 0}
  ]
}
```

**История транзакций.**
```
GET /api/v1/wallet/transactions?limit=20&cursor=txn_790
Response: {
  "transactions": [
    {
      "transaction_id": "txn_790",
      "type": "P2P_TRANSFER",
      "direction": "outgoing",
      "amount": 500,
      "currency": "USD",
      "counterparty": "user_456",
      "status": "completed",
      "created_at": "2024-01-15T10:30:45Z"
    },
    ...
  ],
  "next_cursor": "txn_780"
}
```

**Конвертация (опционально).**
```
POST /api/v1/wallet/convert
Headers: {
  "Idempotency-Key": "convert-44"
}
Body: {
  "from_currency": "USD",
  "to_currency": "EUR",
  "amount": 100
}
Response: {
  "transaction_id": "txn_791",
  "from": {"currency": "USD", "amount": 100},
  "to": {"currency": "EUR", "amount": 92.5},
  "rate": 0.925,
  "fee": 1.0
}
```

Важно: все операции должны быть идемпотентными. `Idempotency-Key` должен быть unique per logical operation, derived from the event. Для переводов — `transfer:{sender_id}:{receiver_id}:{timestamp}:{amount}` или client-generated UUID.

Клеппман в «Высоконагруженных приложениях» объясняет, что API — это контракт. Он определяет, как клиент и сервер взаимодействуют. И выбор стиля API (REST, gRPC) зависит от того, кто клиенты и какие требования к производительности.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как API влияет на архитектуру. Если у тебя кошелёк, идемпотентность обязательна.

**Схема 22.3 — API endpoints и их взаимодействие с компонентами**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         API Gateway / LB                            │
│                                                                     │
│  POST /wallet/topup ──────────▶ Payment Service ──▶ Ledger Service │
│  (Idempotency-Key)                  │              (journal_entry) │
│                                     ├──▶ External Provider         │
│                                     └──▶ Kafka (balance_updated)   │
│                                                                     │
│  POST /wallet/transfer ───────▶ Transaction Service ──▶ Ledger     │
│  (Idempotency-Key)                  │              (journal_entry) │
│                                     ├──▶ PostgreSQL (locks)        │
│                                     └──▶ Kafka (transfer_completed)│
│                                                                     │
│  GET /wallet/balance ─────────▶ Wallet Service ──▶ Redis (cache)   │
│                                     │              │               │
│                                     │              └── MISS ──▶ DB │
│                                     └──▶ вернуть баланс            │
│                                                                     │
│  GET /wallet/transactions ────▶ Wallet Service ──▶ PostgreSQL      │
│                                     │              (ledger_entries)│
│                                     └──▶ вернуть историю           │
│                                                                     │
│  POST /wallet/convert ────────▶ Transaction Service ──▶ Ledger     │
│  (Idempotency-Key)                  │              (journal_entry) │
│                                     ├──▶ Exchange Rate Service     │
│                                     └──▶ Kafka (convert_completed) │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 22.4 Проектирование данных: ledger, balances, idempotency

Теперь давай спроектируем данные. Нам нужны таблицы для ledger, балансов и идемпотентности.

**Схема.**
```sql
CREATE TABLE accounts (
  id BIGSERIAL PRIMARY KEY,
  account_id VARCHAR(20) UNIQUE NOT NULL,
  user_id BIGINT NOT NULL,
  account_type VARCHAR(20) NOT NULL,  -- 'USER_WALLET', 'TREASURY', 'ESCROW', 'EXTERNAL'
  currency VARCHAR(3) NOT NULL,
  status VARCHAR(20) DEFAULT 'active',  -- 'active', 'frozen', 'closed'
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_accounts_user ON accounts(user_id, currency);

CREATE TABLE journal_entries (
  id BIGSERIAL PRIMARY KEY,
  journal_id VARCHAR(20) UNIQUE NOT NULL,
  idempotency_key VARCHAR(255) UNIQUE NOT NULL,
  transaction_type VARCHAR(50) NOT NULL,  -- 'P2P_TRANSFER', 'TOP_UP', 'WITHDRAWAL', 'CONVERSION'
  status VARCHAR(20) DEFAULT 'pending',  -- 'pending', 'completed', 'failed'
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  completed_at TIMESTAMP
);

CREATE INDEX idx_journal_idempotency ON journal_entries(idempotency_key);
CREATE INDEX idx_journal_type_created ON journal_entries(transaction_type, created_at DESC);

CREATE TABLE ledger_entries (
  id BIGSERIAL PRIMARY KEY,
  entry_id VARCHAR(20) UNIQUE NOT NULL,
  journal_id VARCHAR(20) NOT NULL REFERENCES journal_entries(journal_id),
  account_id VARCHAR(20) NOT NULL REFERENCES accounts(account_id),
  direction VARCHAR(6) NOT NULL CHECK (direction IN ('DEBIT', 'CREDIT')),
  amount DECIMAL(18,4) NOT NULL,
  currency VARCHAR(3) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ledger_journal ON ledger_entries(journal_id);
CREATE INDEX idx_ledger_account ON ledger_entries(account_id, created_at DESC);

-- Invariant: for every journal_id, sum(DEBIT) = sum(CREDIT)

CREATE TABLE balances (
  account_id VARCHAR(20) PRIMARY KEY REFERENCES accounts(account_id),
  available DECIMAL(18,4) NOT NULL DEFAULT 0,
  pending DECIMAL(18,4) NOT NULL DEFAULT 0,
  version BIGINT NOT NULL DEFAULT 0,
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_balances_account ON balances(account_id);
```

**Ledger-based подход.**
Вместо того чтобы хранить баланс как поле в таблице `users`, мы храним журнал транзакций (journal_entries) и записи (ledger_entries). Баланс вычисляется как сумма credit'ов минус сумма debit'ов для каждого аккаунта.

Преимущества:
- Полная история всех транзакций.
- Аудит: можно отследить, откуда пришли деньги.
- Согласованность: debit'ы всегда равны credit'ам.
- Восстановление: если баланс потерян, его можно вычислить из ledger.

Недостатки:
- Чтение баланса медленное (нужно суммировать все entries).
- Решение: хранить материализованный баланс (balances table) и обновлять его атомарно с ledger entries .

**Stored vs derived vs hybrid.**
- Derived balance: `SELECT SUM(credits) - SUM(debits) FROM ledger_entries WHERE account_id = ?`. Всегда правильный, но медленный для больших аккаунтов .
- Stored balance: таблица `balances`, обновляется атомарно с ledger entries. O(1) чтение, но может расходиться с ledger при багах .
- Hybrid (рекомендуется): stored balance для чтения, derived balance периодически для reconciliation .

Клеппман в «Высоконагруженных приложениях» объясняет, что проектирование данных — это компромисс между нормализацией и денормализацией, между шардированием и cross-shard запросами. И выбор зависит от того, какие запросы у тебя в системе.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как проектирование данных влияет на архитектуру. Если у тебя кошелёк, ledger и балансы обязательны.

**Схема 22.4 — Схема данных: ledger, balances, idempotency**

```
PostgreSQL (accounts) — шардирован по user_id
┌─────────────────────────────────────────────────────────────┐
│ accounts                                                    │
├─────────────────────────────────────────────────────────────┤
│ id BIGSERIAL PK                                             │
│ account_id VARCHAR(20) UNIQUE                               │
│ user_id BIGINT (shard key)                                  │
│ account_type VARCHAR(20)                                    │
│ currency VARCHAR(3)                                         │
│ status VARCHAR(20)                                          │
│ created_at TIMESTAMP                                        │
└─────────────────────────────────────────────────────────────┘

PostgreSQL (journal_entries) — шардирован по journal_id
┌─────────────────────────────────────────────────────────────┐
│ journal_entries                                             │
├─────────────────────────────────────────────────────────────┤
│ id BIGSERIAL PK                                             │
│ journal_id VARCHAR(20) UNIQUE (shard key)                   │
│ idempotency_key VARCHAR(255) UNIQUE                         │
│ transaction_type VARCHAR(50)                                │
│ status VARCHAR(20)                                          │
│ created_at TIMESTAMP                                        │
│ completed_at TIMESTAMP                                      │
└─────────────────────────────────────────────────────────────┘

PostgreSQL (ledger_entries) — шардирован по account_id
┌─────────────────────────────────────────────────────────────┐
│ ledger_entries                                              │
├─────────────────────────────────────────────────────────────┤
│ id BIGSERIAL PK                                             │
│ entry_id VARCHAR(20) UNIQUE                                 │
│ journal_id VARCHAR(20)                                      │
│ account_id VARCHAR(20) (shard key)                          │
│ direction VARCHAR(6)  -- DEBIT/CREDIT                       │
│ amount DECIMAL(18,4)                                        │
│ currency VARCHAR(3)                                         │
│ created_at TIMESTAMP                                        │
└─────────────────────────────────────────────────────────────┘

PostgreSQL (balances) — шардирован по account_id
┌─────────────────────────────────────────────────────────────┐
│ balances                                                    │
├─────────────────────────────────────────────────────────────┤
│ account_id VARCHAR(20) PK (shard key)                       │
│ available DECIMAL(18,4)                                     │
│ pending DECIMAL(18,4)                                       │
│ version BIGINT                                              │
│ updated_at TIMESTAMP                                        │
└─────────────────────────────────────────────────────────────┘

Redis:
┌─────────────────────────────────────────────────────────────┐
│ balance:{user_id}:{currency} → amount (TTL: 5 min)         │
│ idempotency:{key} → status (TTL: 24h)                       │
└─────────────────────────────────────────────────────────────┘
```

---

### 22.5 Баланс: stored vs derived vs hybrid

Теперь давай разберём, как хранить баланс.

**Derived balance.**
Баланс вычисляется на лету из ledger entries:
```sql
SELECT 
  COALESCE(SUM(CASE WHEN direction = 'CREDIT' THEN amount ELSE 0 END), 0) -
  COALESCE(SUM(CASE WHEN direction = 'DEBIT' THEN amount ELSE 0 END), 0) AS balance
FROM ledger_entries
WHERE account_id = 'acc_123';
```

Преимущества: всегда правильный, нет расхождений.
Недостатки: медленный для аккаунтов с миллионами entries. Каждый раз нужно сканировать все записи .

**Stored balance.**
Баланс хранится в таблице `balances` и обновляется атомарно с ledger entries:
```sql
BEGIN TRANSACTION;
  INSERT INTO journal_entries (...) VALUES (...);
  INSERT INTO ledger_entries (account_id, direction, amount) VALUES ('acc_123', 'DEBIT', 500);
  UPDATE balances SET available = available - 500, version = version + 1 WHERE account_id = 'acc_123';
COMMIT;
```

Преимущества: O(1) чтение, быстрый.
Недостатки: может расходиться с ledger при багах. Нужен reconciliation .

**Hybrid (рекомендуется).**
Stored balance для чтения, derived balance периодически для reconciliation.
- Чтение: из `balances` (или Redis кеша).
- Reconciliation: nightly job вычисляет derived balance и сравнивает с stored. Если расхождение — алерт .

Клеппман в «Высоконагруженных приложениях» объясняет, что балансы — это компромисс между скоростью и правильностью. Derived всегда правильный, но медленный. Stored быстрый, но может расходиться. Hybrid даёт лучшее из обоих миров. И выбор зависит от того, что важнее для задачи.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как балансы влияют на архитектуру. Если у тебя кошелёк, hybrid балансы обязательны.

**Схема 22.5 — Балансы: stored vs derived vs hybrid**

```
STORED BALANCE:

Чтение:
SELECT available FROM balances WHERE account_id = 'acc_123';
→ 1000 (O(1), < 10ms)

Запись:
BEGIN TRANSACTION;
  INSERT INTO ledger_entries (...) VALUES (...);
  UPDATE balances SET available = available - 500 WHERE account_id = 'acc_123';
COMMIT;

Проблема: если баг в коде, balance может разойтись с ledger.

DERIVED BALANCE:

Чтение:
SELECT 
  SUM(CASE WHEN direction='CREDIT' THEN amount ELSE 0 END) -
  SUM(CASE WHEN direction='DEBIT' THEN amount ELSE 0 END) AS balance
FROM ledger_entries WHERE account_id = 'acc_123';
→ 1000 (O(n), может быть медленно для миллионов entries)

Преимущество: всегда правильный, нет расхождений.

HYBRID:

Чтение:
GET Redis balance:{user_id} → 1000 (fast)
Если MISS: SELECT available FROM balances → 1000 (fast)
Если MISS: SELECT SUM(...) FROM ledger_entries → 1000 (slow, fallback)

Reconciliation (nightly):
Для каждого account:
  derived = SELECT SUM(...) FROM ledger_entries
  stored = SELECT available FROM balances
  if derived != stored:
    ALERT (расхождение)
    (manual review или auto-fix если safe)
```

---

### 22.6 Переводы: атомарность, блокировки, порядок

Теперь давай разберём переводы между пользователями.

**Атомарность.**
Перевод должен быть атомарным: либо деньги списываются с A и зачисляются B, либо ничего не происходит. Это достигается через транзакцию:
```sql
BEGIN TRANSACTION;
  SELECT wallet A FOR UPDATE;  -- lock
  SELECT wallet B FOR UPDATE;  -- lock
  -- check balance
  INSERT INTO ledger_entries (A, DEBIT, amount);
  INSERT INTO ledger_entries (B, CREDIT, amount);
  UPDATE balances SET available = available - amount WHERE account_id = A;
  UPDATE balances SET available = available + amount WHERE account_id = B;
  INSERT INTO outbox (event);
COMMIT;
```

**Блокировки.**
`SELECT ... FOR UPDATE` блокирует строку на время транзакции. Это предотвращает race conditions: два перевода не могут одновременно изменить баланс одного кошелька.

**Порядок блокировок.**
Чтобы избежать deadlock, блокировки должны быть в определённом порядке (например, по account_id). Если A переводит B, и B переводит A одновременно, без порядка блокировок может возникнуть deadlock.

**Оптимистичная конкурентность.**
Вместо блокировок можно использовать version column:
```sql
UPDATE balances SET available = available - 500, version = version + 1
WHERE account_id = 'acc_123' AND version = 5;
-- если 0 rows affected: retry (кто-то другой изменил баланс)
```

Клеппман в «Высоконагруженных приложениях» объясняет, что переводы — это компромисс между согласованностью и производительностью. Блокировки безопаснее, но медленнее. Оптимистичная конкурентность быстрее, но требует retry. И выбор зависит от того, насколько высока конкурентность.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как переводы влияют на архитектуру. Если у тебя кошелёк, атомарность и блокировки обязательны.

**Схема 22.6 — Переводы: атомарность, блокировки, порядок**

```
TRANSFER (A → B):

BEGIN TRANSACTION;
  -- Lock in order (by account_id) to avoid deadlock
  SELECT * FROM balances WHERE account_id = 'A' FOR UPDATE;  -- A < B
  SELECT * FROM balances WHERE account_id = 'B' FOR UPDATE;
  
  -- Check balance
  IF A.available < amount THEN ROLLBACK; RAISE 'Insufficient funds';
  
  -- Ledger entries (double-entry)
  INSERT INTO ledger_entries (journal_id, account_id, direction, amount)
  VALUES 
    ('j_1', 'A', 'DEBIT', 500),
    ('j_1', 'B', 'CREDIT', 500);
  
  -- Update balances
  UPDATE balances SET available = available - 500 WHERE account_id = 'A';
  UPDATE balances SET available = available + 500 WHERE account_id = 'B';
  
  -- Outbox
  INSERT INTO outbox (event_type, aggregate_id, payload)
  VALUES ('transfer_completed', 'j_1', '{"from":"A","to":"B","amount":500}');
COMMIT;

DEADLOCK AVOIDANCE:

Плохо (deadlock):
T1: lock A, lock B
T2: lock B, lock A
→ deadlock

Хорошо (ordered locks):
T1: lock A (min), lock B (max)
T2: lock A (min), lock B (max)
→ no deadlock

OPTIMISTIC CONCURRENCY:

UPDATE balances SET available = available - 500, version = version + 1
WHERE account_id = 'A' AND version = 5;
-- 1 row affected: success
-- 0 rows affected: retry (version changed)
```

---

### 22.7 Конвертация валют: курсы, округление, мультивалютность

Теперь давай разберём конвертацию валют.

**Курсы.**
- Рыночный курс: обновляется часто (каждую секунду).
- Фиксированный курс: устанавливается на день.
- С наценкой: рыночный курс + комиссия (например, 1%).

**Источник курсов.**
- External API (например, OpenExchangeRates, CurrencyLayer).
- Внутренний сервис, который агрегирует курсы.

**Округление.**
При конвертации нужно округлять до определённого количества знаков (например, 2 для USD/EUR, 0 для JPY). Правила округления должны быть детерминированными и аудируемыми.

**Мультивалютность.**
Пользователь может иметь несколько кошельков в разных валютах. Перевод между валютами требует конвертации.

**Ledger для конвертации.**
Конвертация — это тоже транзакция:
```
Journal (CONVERSION):
  DEBIT:  user_wallet_USD (100 USD)
  CREDIT: user_wallet_EUR (92.5 EUR)
  CREDIT: platform_fee_USD (1 USD)  -- fee
  CREDIT: treasury_USD (99 USD)      -- residual
```
Курс и комиссия фиксируются в ledger entry для аудита.

Клеппман в «Высоконагруженных приложениях» объясняет, что конвертация — это компромисс между точностью и простотой. Рыночный курс точнее, но волатильнее. Фиксированный курс проще, но может быть невыгодным. И выбор зависит от того, что важнее для задачи.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как конвертация влияет на архитектуру. Если у тебя мультивалютность, курсы и округление обязательны.

**Схема 22.7 — Конвертация валют: курсы, округление, мультивалютность**

```
CONVERSION FLOW (USD → EUR):

Пользователь ──POST /convert──▶ Transaction Service
                                      │
                                      ├──1. Получить курс USD/EUR (Exchange Rate Service)
                                      │      └── rate = 0.925
                                      ├──2. Вычислить fee = amount * 0.01 = 1 USD
                                      ├──3. Вычислить converted = (amount - fee) * rate = 99 * 0.925 = 91.575
                                      ├──4. Округлить converted = 91.58 EUR (round half up)
                                      ├──5. Ledger Service: INSERT journal_entry (CONVERSION)
                                      │      ├── DEBIT:  user_wallet_USD (100 USD)
                                      │      ├── CREDIT: platform_fee_USD (1 USD)
                                      │      ├── CREDIT: treasury_USD (99 USD)
                                      │      ├── DEBIT:  treasury_EUR (91.58 EUR)
                                      │      └── CREDIT: user_wallet_EUR (91.58 EUR)
                                      ├──6. UPDATE balances
                                      └──7. PUBLISH Kafka (convert_completed)

LEDGER ENTRIES:

journal_entries:
┌─────────────────────────────────────────────────────────────┐
│ journal_id: j_conv_1                                        │
│ type: CONVERSION                                            │
│ status: completed                                           │
│ metadata: {"rate": 0.925, "fee": 1.0, "from": "USD", "to": "EUR"} │
└─────────────────────────────────────────────────────────────┘

ledger_entries:
┌─────────────────────────────────────────────────────────────┐
│ account_id           │ direction │ amount │ currency       │
├──────────────────────┼───────────┼────────┼────────────────┤
│ user_wallet_USD      │ DEBIT     │ 100.00 │ USD            │
│ platform_fee_USD     │ CREDIT    │ 1.00   │ USD            │
│ treasury_USD         │ CREDIT    │ 99.00  │ USD            │
│ treasury_EUR         │ DEBIT     │ 91.58  │ EUR            │
│ user_wallet_EUR      │ CREDIT    │ 91.58  │ EUR            │
└─────────────────────────────────────────────────────────────┘

Invariant: sum(DEBIT) = sum(CREDIT) per currency.
USD: 100 = 1 + 99 ✓
EUR: 91.58 = 91.58 ✓
```

---

### 22.8 Reconciliation: сверка ledger с балансами и внешними системами

Теперь давай разберём reconciliation.

**Reconciliation** — это процесс сверки ledger с балансами и внешними системами (банки, карточные сети). Он нужен, потому что:
- Stored balance может расходиться с derived balance (баги, race conditions).
- Внешние системы могут иметь расхождения (банк не получил платёж, но мы думаем, что получил).
- Аудит требует, чтобы всё сходилось.

**Типы reconciliation.**
1. **Internal:** сверка stored balance с derived balance (из ledger). Nightly job.
2. **External:** сверка ledger с выписками банков/провайдеров. Ежедневно.
3. **Cross-currency:** сверка остатков на treasury accounts с ожидаемыми.

**Процесс.**
1. Получить список транзакций за период.
2. Сравнить с внешней системой.
3. Если match: OK.
4. Если mismatch: создать reconciliation report, flag для manual review.
5. **Не auto-fix** финансовые расхождения: auto-correcting money movements is illegal . Вместо этого — alert ops для manual review.

Клеппман в «Высоконагруженных приложениях» объясняет, что reconciliation — это компромисс между автоматизацией и контролем. Автоматическая сверка быстрее, но ручная безопаснее для финансов. И выбор зависит от того, насколько критична система.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как reconciliation влияет на архитектуру. Если у тебя кошелёк, reconciliation обязателен.

**Схема 22.8 — Reconciliation: сверка ledger с балансами и внешними системами**

```
RECONCILIATION PIPELINE (nightly):

┌─────────────────────────────────────────────────────────────┐
│  1. Internal Reconciliation                                 │
│                                                             │
│  Для каждого account:                                       │
│    derived = SELECT SUM(credits) - SUM(debits)              │
│              FROM ledger_entries WHERE account_id = ?       │
│    stored = SELECT available FROM balances WHERE account_id = ? │
│                                                             │
│    if derived != stored:                                    │
│      ALERT (расхождение)                                    │
│      flag для manual review                                 │
│                                                             │
│  Метрика: internal_reconciliation_match_rate                │
│  Цель: > 99.99%                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  2. External Reconciliation                                 │
│                                                             │
│  Получить выписку от банка/провайдера (settlement file).    │
│  Для каждого payment:                                       │
│    our_ledger = SELECT * FROM ledger_entries WHERE journal_id = ? │
│    external = bank_settlement[payment_id]                   │
│                                                             │
│    if our_ledger.amount != external.amount:                 │
│      ALERT (расхождение)                                    │
│      flag для manual review                                 │
│      (НЕ auto-fix: money movements illegal)                 │
│                                                             │
│  Метрика: external_reconciliation_match_rate                │
│  Цель: > 99.5%                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  3. Cross-Currency Reconciliation                           │
│                                                             │
│  Для каждого treasury account:                              │
│    expected = SUM(credits) - SUM(debits)                    │
│    actual = bank_balance[treasury_account]                  │
│                                                             │
│    if expected != actual:                                   │
│      ALERT                                                  │
│                                                             │
│  Метрика: treasury_reconciliation_match_rate                │
│  Цель: 100%                                                 │
└─────────────────────────────────────────────────────────────┘

RECONCILIATION REPORT:

┌─────────────────────────────────────────────────────────────┐
│ date: 2024-01-15                                            │
│ internal_match_rate: 99.99%                                 │
│ external_match_rate: 99.7%                                  │
│ mismatches: 3                                               │
│   - txn_123: ledger=100, bank=0 (bank didn't receive)      │
│   - txn_456: ledger=50, bank=50 (duplicate in bank?)       │
│   - txn_789: ledger=25, bank=25 (currency mismatch?)       │
│ action: manual review required                              │
└─────────────────────────────────────────────────────────────┘
```

---

### 22.9 Отказоустойчивость, мониторинг и безопасность

Теперь давай разберём, как обеспечить отказоустойчивость, мониторинг и безопасность.

**Отказоустойчивость.**
- Failover: если Wallet Service или Transaction Service упал, клиенты переподключаются к другому экземпляру.
- Circuit breaker: если база данных или Redis недоступны, сервис быстро возвращает ошибку.
- Graceful degradation: если Redis недоступен, чтение баланса идёт из PostgreSQL (медленнее, но работает).
- Retry: с exponential backoff для временных сбоев.
- Timeout: 500 мс для перевода, 50 мс для чтения баланса.

**Мониторинг.**
- Метрики: balance read latency, transfer latency, ledger posting latency, reconciliation match rate, idempotency hit rate, balance projection drift .
- Логи: структурированные, с trace ID, уровни.
- Трейсинг: OpenTelemetry, Jaeger (trace от API до ledger).
- Алерты: на симптомы (transfer latency > 1s, reconciliation match rate < 99.5%, balance drift > 0).

**Безопасность.**
- Аутентификация: JWT для пользователей, API keys для сервисов.
- Авторизация: пользователь может видеть только свои кошельки и транзакции.
- Rate limiting: ограничение переводов (например, 100 в день), пополнений (например, 10 в день).
- Валидация: проверка суммы (положительная, в пределах лимита), валюты.
- Fraud detection: проверка аномалий (необычные суммы, частые переводы, новые получатели).
- PCI DSS: не хранить PAN, использовать токенизацию .
- Секреты: хранение в Vault, ротация.
- Аудит: логирование всех операций, immutable ledger, retention 7 лет.

Клеппман в «Высоконагруженных приложениях» объясняет, что отказоустойчивость, мониторинг и безопасность — это не фичи, а свойства системы. Без них система не может эксплуатироваться. И выбор зависит от того, насколько критична система.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как отказоустойчивость, мониторинг и безопасность влияют на архитектуру. Если у тебя кошелёк, они обязательны.

**Схема 22.9 — Полная архитектура с отказоустойчивостью и мониторингом**

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
│                    Wallet / Transaction Service                     │
│  ┌──────────────────────────────┼──────────────────────────────┐   │
│  │  Circuit Breaker             │                              │   │
│  │  ├── PostgreSQL: CLOSED/OPEN/HALF-OPEN                      │   │
│  │  ├── Redis: CLOSED/OPEN/HALF-OPEN                           │   │
│  │  └── Kafka: CLOSED/OPEN/HALF-OPEN                           │   │
│  │                                                             │   │
│  │  Graceful Degradation:                                      │   │
│  │  ├── Redis недоступен: balance из PostgreSQL (медленнее)    │   │
│  │  ├── Kafka недоступен: outbox не публикуется (retry позже)  │   │
│  │  └── Exchange Rate недоступен: конвертация отключена        │   │
│  │                                                             │   │
│  │  Timeout: 500ms для transfer, 50ms для balance              │   │
│  │  Retry: exponential backoff + jitter                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                         ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│ PostgreSQL    │         │ Redis         │         │ Kafka Cluster │
│ Master        │         │ (balance      │         │ (3 brokers)   │
│      │        │         │  cache,       │         │               │
│      ▼        │         │  idempotency) │         │ topic:        │
│ Replica 1     │         │               │         │ wallet_events │
│ Replica 2     │         │               │         │               │
└───────────────┘         └───────────────┘         └───────┬───────┘
                                                            │
                                                            ▼
                                                    ┌───────────────┐
                                                    │ Reconciliation│
                                                    │ Engine        │
                                                    │ (nightly)     │
                                                    │               │
                                                    │ - internal    │
                                                    │ - external    │
                                                    │ - cross-curr  │
                                                    └───────────────┘

Security Layer:
┌─────────────────────────────────────────────────────────────────────┐
│  WAF (Cloudflare) ──▶ Rate Limiter (Redis) ──▶ JWT Auth            │
│  Vault (secrets) ──▶ Fraud Detection ──▶ PCI DSS (tokenization)    │
│  Audit Log (append-only, 7 years)                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Выводы

В этой главе ты узнал, как строить digital wallet. Ты разобрал ledger-based подход, балансы (stored vs derived vs hybrid), переводы, конвертацию валют, reconciliation. Ты узнал, что такое CQRS для кошелька и почему это лучше, чем просто поле `balance`. Ты увидел, как те же строительные блоки (PostgreSQL, Redis, Kafka) применяются к wallet задаче.

Главное, что ты должен вынести из этой главы: **digital wallet — это задача о состоянии и транзакциях.** Хорошая архитектура — не та, которая использует самую модную базу данных. Хорошая архитектура — та, которая гарантирует, что баланс всегда сходится с ledger, переводы атомарны, а reconciliation находит расхождения до того, как они станут проблемой.

---

## Выжимка для быстрого повторения

- **Требования:** пополнение, переводы, вывод, баланс, история. Строгая согласованность для ledger, идемпотентность, аудит, PCI DSS.
- **Оценки:** 10 000 переводов/с на пике, 500+ чтений баланса/с. Balance-heavy и transfer-heavy. Нужен кеш и шардирование.
- **API:** POST /topup, POST /transfer, GET /balance, GET /transactions. Все с `Idempotency-Key`.
- **Данные:** accounts (шардирован по user_id), journal_entries (шардирован по journal_id), ledger_entries (шардирован по account_id), balances (шардирован по account_id).
- **Балансы:** derived (медленно, но правильно), stored (быстро, но может расходиться), hybrid (stored для чтения, derived для reconciliation).
- **Переводы:** атомарность через транзакцию, `SELECT FOR UPDATE` для блокировки, ordered locks для избежания deadlock, оптимистичная конкурентность как альтернатива.
- **Конвертация:** курсы (рыночный, фиксированный, с наценкой), округление, мультивалютность. Ledger entries для аудита.
- **Reconciliation:** internal (stored vs derived), external (ledger vs bank), cross-currency (treasury). Nightly job, alert на mismatch, НЕ auto-fix (manual review).
- **Безопасность:** JWT, rate limiting, fraud detection, PCI DSS (tokenization), audit log (7 years).

---

## Вопросы для самопроверки

1. Почему digital wallet требует ledger-based подхода, а не просто поля `balance`?
2. Чем stored balance отличается от derived? Почему hybrid — лучший выбор?
3. Как обеспечить атомарность перевода между двумя кошельками? Какие блокировки использовать?
4. Как работает конвертация валют? Какие ledger entries создаются?
5. Зачем нужен reconciliation и почему нельзя auto-fix финансовые расхождения?

---

## Эталонные ответы

**1. Почему digital wallet требует ledger-based подхода, а не просто поля `balance`?**

Поле `balance` в таблице `users` не даёт истории транзакций, аудита и возможности восстановления. Если баланс потерян или испорчен, его нельзя восстановить. Ledger-based подход хранит все транзакции (journal_entries + ledger_entries), и баланс можно вычислить из них. Это даёт: полную историю, аудит (кто, когда, сколько), согласованность (debit = credit), восстановление (derived balance). Stored balance — это оптимизация для чтения, но source of truth — ledger.

**2. Чем stored balance отличается от derived? Почему hybrid — лучший выбор?**

Derived balance вычисляется из ledger entries: `SUM(credits) - SUM(debits)`. Всегда правильный, но медленный для больших аккаунтов. Stored balance хранится в таблице `balances` и обновляется атомарно с ledger entries. O(1) чтение, но может расходиться с ledger при багах. Hybrid: stored для чтения (быстро), derived периодически для reconciliation (проверка правильности). Это даёт скорость и правильность. Если расхождение — алерт для manual review.

**3. Как обеспечить атомарность перевода между двумя кошельками? Какие блокировки использовать?**

Атомарность через транзакцию: `BEGIN; SELECT wallet A FOR UPDATE; SELECT wallet B FOR UPDATE; check balance; INSERT ledger entries; UPDATE balances; INSERT outbox; COMMIT;`. Блокировки: `SELECT ... FOR UPDATE` блокирует строки на время транзакции, предотвращая race conditions. Чтобы избежать deadlock, блокировки должны быть в определённом порядке (по account_id). Альтернатива: оптимистичная конкурентность (version column, retry при конфликте).

**4. Как работает конвертация валют? Какие ledger entries создаются?**

Конвертация: получить курс, вычислить fee, вычислить converted amount, округлить. Ledger entries: DEBIT user_wallet_USD (100), CREDIT platform_fee_USD (1), CREDIT treasury_USD (99), DEBIT treasury_EUR (91.58), CREDIT user_wallet_EUR (91.58). Invariant: sum(DEBIT) = sum(CREDIT) per currency. USD: 100 = 1+99. EUR: 91.58 = 91.58. Курс и fee фиксируются в metadata journal_entry для аудита.

**5. Зачем нужен reconciliation и почему нельзя auto-fix финансовые расхождения?**

Reconciliation нужен, потому что stored balance может расходиться с derived (баги), а ledger может расходиться с внешними системами (банк не получил платёж). Reconciliation находит расхождения до того, как они станут проблемой. Auto-fix нельзя, потому что auto-correcting money movements is illegal . Вместо этого — alert ops для manual review. Ручная проверка гарантирует, что исправления корректны и аудируемы.

---

## Мост к следующей главе

Ты спроектировал digital wallet — от требований до продакшена. Ты узнал, что такое ledger-based подход, балансы, переводы, конвертация, reconciliation. Ты увидел, как те же строительные блоки (PostgreSQL, Redis, Kafka) применяются к wallet задаче.

Но кошелёк — это про хранение и переводы денег между пользователями. А что если задача — это торговля на бирже? Что если нужно обрабатывать миллионы ордеров в секунду,匹配ить покупателей и продавцов, обеспечивать справедливую цену, обрабатывать отмены и частичные исполнения? Что если задержка в микросекунды стоит денег?

В следующей главе — **Stock Exchange: ордера, matching, стакан**. Ты узнаешь, как работает биржевой matching engine, что такое order book, как обеспечить fairness и low latency. Ты узнаешь, что такое market data и как доставлять его тысячам клиентов. И ты поймёшь, что stock exchange — это задача о real-time matching и ultra-low latency, где каждая микросекунда имеет значение.