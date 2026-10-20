## Глава 23: Stock Exchange: ордера, matching, стакан

### Оглавление главы

- **23.1** Уточнение требований: что такое биржа и кому она нужна
- **23.2** Оценка нагрузки: сообщения, ордера, латентность
- **23.3** API: ввод ордера, отмена, market data
- **23.4** Проектирование данных: order book, price levels, order map
- **23.5** Matching engine: price-time priority, алгоритмы, сложность
- **23.6** Критический путь: почему микросервисы не работают
- **23.7** Market data: ITCH, WebSocket, fan-out
- **23.8** Отказоустойчивость, мониторинг и безопасность

---

### Введение

В предыдущей главе мы спроектировали digital wallet — от требований до продакшена. Ты узнал, что такое ledger-based подход, балансы, переводы, конвертация, reconciliation. Ты увидел, как те же строительные блоки (PostgreSQL, Redis, Kafka) применяются к wallet задаче. Но кошелёк — это про хранение и переводы денег между пользователями. А что если задача — это торговля на бирже?

Представь, что тебя просят спроектировать stock exchange — как NASDAQ или Binance. Тысячи трейдеров отправляют ордера на покупку и продажу. Система должна匹配ить покупателей и продавцов, обеспечивать справедливую цену, обрабатывать отмены и частичные исполнения. И всё это — с задержкой в микросекунды. Каждая микросекунда задержки стоит денег, потому что трейдеры соревнуются за скорость.

На первый взгляд, всё просто: у тебя есть список ордеров на покупку и список ордеров на продажу, ты ищешь совпадения по цене. Но что если ордеров миллионы? Что если 95% сообщений — это отмены? Что если нужно обрабатывать 8 миллионов сообщений в секунду на пике? Что если задержка в 100 микросекунд означает, что твой ордер не исполнится, потому что кто-то другой успел раньше? Наивный подход ломается на масштабе и латентности.

В этой главе ты узнаешь, как строить биржу. Ты разберёшь order book, matching engine, price-time priority. Ты узнаешь, почему критический путь не может быть распределённым, и почему все компоненты ставятся на один сервер. Ты узнаешь, что такое ITCH и как доставлять market data тысячам клиентов. И ты поймёшь, что stock exchange — это задача о real-time matching и ultra-low latency, где каждая микросекунда имеет значение.

---

### 23.1 Уточнение требований: что такое биржа и кому она нужна

Прежде чем рисовать диаграмму, нужно понять, что мы строим. И первый шаг — уточнить требования.

**Функциональные требования.**
- Трейдер может отправить ордер на покупку (bid) или продажу (ask).
- Поддерживаются типы ордеров: market, limit, stop-loss, fill-or-kill, IOC (immediate-or-cancel).
- Система匹配ит ордера по price-time priority.
- Трейдер может отменить ордер.
- Трейдер может запросить статус ордера и историю исполнений.
- Клиенты получают market data (order book, trades) в реальном времени.
- Система поддерживает partial fills (частичное исполнение).

**Нефункциональные требования.**
- Ultra-low latency: matching latency в микросекундах (10–100 µs).
- Высокая пропускная способность: 1M+ сообщений в секунду на пике .
- Строгий порядок: все ордера обрабатываются в детерминированном порядке (sequencer).
- Высокая доступность: 99.99% для торгов.
- Fairness: все участники видят одинаковый порядок событий.
- Долговечность: ордера и исполнения не теряются после подтверждения.

**Вопросы, которые нужно задать.**
- Сколько символов (акций) торгуется? 100? 10 000?
- Сколько сообщений в секунду на пике? 1M? 8M?
- Какая задержка допустима? 10 µs? 100 µs? 1 ms?
- Какие типы ордеров поддерживать?
- Нужна ли поддержка аукционов (opening/closing)?
- Нужна ли поддержка деривативов или только equities?
- Какие протоколы использовать? FIX? OUCH? Binary?

Ответы на эти вопросы определяют архитектуру. Например, если latency должен быть 10 µs, ты не можешь использовать микросервисы с network hops. Ты ставишь всё на один сервер с shared memory. Если 8M сообщений в секунду на пике, ты шардируешь по символу и используешь single-threaded matching engine на каждый символ.

Клеппман в «Высоконагруженных приложениях» объясняет, что требования определяют архитектуру. Без требований ты проектируешь в вакууме. И выбор зависит от того, какие требования у тебя в системе.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» (Volume 2, Chapter 13: Stock Exchange) показывает, что биржа — это классическая задача на low-latency matching .

**Схема 23.1 — Общая картина системы (высокоуровневая)**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Exchange Members                            │
│  (Brokers, HFT firms, Retail brokers)                               │
└──────┬──────────────────────────────────────────────────────────────┘
       │ FIX / OUCH / Binary (co-located)
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Gateway / Order Manager                          │
│  - protocol parsing (FIX, OUCH, binary)                             │
│  - risk checks (mandatory, critical path)                           │
│  - assign sequence number                                           │
│  - forward to matching engine via shared memory                     │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ shared memory (no network)
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Matching Engine                                  │
│  - single-threaded, pinned to CPU core                              │
│  - per-symbol order book (in-memory)                                │
│  - price-time priority matching                                     │
│  - emit executions, market data deltas                              │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ shared memory
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Market Data Publisher                            │
│  - ITCH / MoldUDP64 (multicast UDP)                                 │
│  - order book snapshots + deltas                                    │
│  - fan-out to subscribers                                           │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Market Data Subscribers                          │
│  (HFT firms, brokers, data vendors)                                 │
└─────────────────────────────────────────────────────────────────────┘

Background (non-critical path):
┌─────────────────────────────────────────────────────────────────────┐
│  Order/Execution Persistence (async, sequencer log)                 │
│  Audit & Reporting                                                  │
│  Clearing House integration                                         │
│  Risk Management (post-trade)                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 23.2 Оценка нагрузки: сообщения, ордера, латентность

Теперь у нас есть требования. Давай оценим нагрузку.

**Сообщения.**
- NASDAQ INET peak: 1,134,640 messages/sec .
- Deutsche Börse T7: inbound peak ~8M messages/sec, matching ~300K messages/sec .
- Cancel:execute ratio: ~20:1 (US equities typical) .
- Это означает, что 95% сообщений — это отмены. Matching engine должен быть оптимизирован для random-delete priority queue .

**Ордера.**
- Active orders in memory: ~2M (8,000 symbols * 250 avg resting orders) .
- Memory per order: ~128 bytes .
- Total hot memory: ~256 MB — fits in L3 cache of a modern server .
- Это ключевой инсайт: entire active state fits in CPU cache. Cache misses, not CPU cycles, are the bottleneck at microsecond latencies .

**Латентность.**
- Baseline latency: few microseconds one-way in colocated settings .
- Matching latency: 10–100 µs.
- Один network round trip: hundreds of microseconds. Несколько microservice hops: milliseconds. Синхронный disk persist: tens of milliseconds .
- Поэтому критический путь должен быть на одном сервере, без network hops, без disk I/O .

**Market data.**
- Subscribers: 1,000+ (co-located HFT firms + retail brokers) .
- Order book depth: ~10,000 levels per symbol .
- Market data protocol: ITCH over MoldUDP64 (sequenced UDP multicast) .

Вывод: система ultra-low-latency, message-heavy (1M+ msg/s), cancel-heavy (95% отмен). Критический путь должен быть single-threaded, in-memory, без network hops. Matching engine шардируется по символу (один символ — один partition). Market data доставляется через multicast UDP.

Клеппман в «Высоконагруженных приложениях» объясняет, что оценки определяют, где bottleneck. Без оценок ты не знаешь, нужен ли шардинг, кеш, очередь. И выбор зависит от того, какие числа у тебя в системе.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как оценки влияют на архитектуру. Если у тебя ultra-low-latency, критический путь не может быть распределённым .

**Схема 23.2 — Потоки данных: order entry, matching, market data**

```
ORDER ENTRY PATH (critical path):

Member ──FIX/OUCH──▶ Gateway
                        │
                        ├──1. Parse message
                        ├──2. Mandatory risk checks (critical path)
                        ├──3. Assign sequence number (sequencer)
                        └──4. Write to shared memory ring buffer

MATCHING PATH (critical path):

Matching Engine (single-threaded, pinned CPU):
                        │
                        ├──1. Read from shared memory ring buffer
                        ├──2. Route to per-symbol order book
                        ├──3. Match order (price-time priority)
                        ├──4. Emit executions + market data deltas
                        └──5. Write to shared memory output ring buffer

MARKET DATA PATH (non-critical, async):

Market Data Publisher:
                        │
                        ├──1. Read market data deltas from shared memory
                        ├──2. Format as ITCH messages
                        ├──3. Send via MoldUDP64 multicast
                        └──4. Subscribers receive (HFT firms, brokers)
```

---

### 23.3 API: ввод ордера, отмена, market data

API — это контракт между клиентом и сервером. Давай определим endpoints.

**Ввод ордера (OUCH binary protocol).**
```
Enter Order (49 bytes):
- Client Token: 14 bytes (client-assigned)
- Side: 1 byte (B=buy, S=sell)
- Quantity: 4 bytes
- Symbol: 8 bytes (padded)
- Price: 4 bytes (fixed-point, 4 decimal places)
- Time-in-force: 4 bytes (DAY, IOC, GTC)
- Display: 1 byte (visible, hidden, attributable)

Accepted Response (66 bytes):
- Order ID: 8 bytes (exchange-assigned sequence)
- Side, Shares, Symbol, Price, TimeInForce, Timestamp
```

**Отмена (OUCH Cancel).**
```
Cancel Order:
- Order ID: 8 bytes
- Quantity: 4 bytes (partial cancel quantity)
```

**Market data (ITCH over MoldUDP64).**
```
Add Order (28 bytes):
- Message Type: 1 byte ('A')
- Sequence Number: 8 bytes (monotonic, gap-detectable)
- Order ID: 8 bytes
- Side: 1 byte
- Shares: 4 bytes
- Symbol: 4 bytes
- Price: 4 bytes
```

**REST API (для retail/исторических запросов).**
```
GET /v1/execution?symbol=AAPL&orderId=...&startTime=...&endTime=...
GET /v1/marketdata/orderBook/L2?symbol=AAPL&depth=10
GET /v1/marketdata/candles?symbol=AAPL&resolution=60&startTime=...
```

Важно: institutional order entry использует FIX/Binary persistent session, не HTTP. REST — для retail и исторических запросов . Price не должен использовать float/double. Используй fixed-point integers (price_ticks) .

Клеппман в «Высоконагруженных приложениях» объясняет, что API — это контракт. Он определяет, как клиент и сервер взаимодействуют. И выбор протокола (FIX, OUCH, binary) зависит от того, кто клиенты и какие требования к латентности.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как API влияет на архитектуру. Если у тебя ultra-low-latency, binary protocol обязателен.

**Схема 23.3 — API endpoints и их взаимодействие с компонентами**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Gateway                                     │
│                                                                     │
│  OUCH Enter Order ──────────▶ Order Manager ──▶ Sequencer          │
│  (binary, 49 bytes)               │              (shared memory)   │
│                                   ├──▶ Risk Checks                 │
│                                   └──▶ Matching Engine             │
│                                                                     │
│  OUCH Cancel ──────────────▶ Order Manager ──▶ Sequencer          │
│  (binary)                         │              (shared memory)   │
│                                   └──▶ Matching Engine             │
│                                                                     │
│  FIX Session (institutional) ▶ Gateway ──▶ Order Manager           │
│                                                                     │
│  REST API (retail/historical):                                      │
│  GET /execution ──────────────▶ Query Service ──▶ Cassandra        │
│  GET /orderBook/L2 ───────────▶ Query Service ──▶ Redis / DB       │
│  GET /candles ────────────────▶ Query Service ──▶ ClickHouse       │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 23.4 Проектирование данных: order book, price levels, order map

Теперь давай спроектируем данные. Order book — это центральная структура данных биржи.

**Order Book структура.**
```go
type Order struct {
    OrderID      uint64
    ClientToken  [14]byte
    SymbolID     uint16
    Side         Side      // Bid or Ask
    Price        int64     // fixed-point (price * 10000)
    RemainingQty uint32
    TimestampNs  uint64
    Prev, Next   *Order    // intrusive linked list
}

type PriceLevel struct {
    Price     int64
    Head      *Order       // FIFO queue head (oldest)
    Tail      *Order       // FIFO queue tail (newest)
    TotalQty  uint64       // aggregate visible quantity
}

type OrderBook struct {
    Bids      SortedMap[int64, *PriceLevel]  // descending by price
    Asks      SortedMap[int64, *PriceLevel]  // ascending by price
    OrderMap  map[uint64]*Order              // O(1) cancel lookup
}
```

**Почему такие структуры.**
- **Ordered map (Tree, SkipList, или Array если price range有限):** быстрый поиск price level. O(log P) для tree.
- **Doubly linked list внутри PriceLevel:** FIFO queue. Add в tail O(1), Match из head O(1).
- **OrderMap (HashMap):** O(1) cancel lookup. Критично, потому что cancels outnumber executions 20:1 .

**Сложность операций.**
| Операция | Сложность |
|----------|-----------|
| Add в tail (same price) | O(1) |
| Match из head | O(1) |
| Cancel (known order) | O(1) после lookup |
| Price level lookup/create | O(log P) для tree |
| Best price | O(1) с указателем |

**Шардирование.**
- Shard by symbol (stock code). Один символ — один partition .
- CP over AP: matching engine — single-threaded, stateful, in-memory.
- Каждый symbol имеет свой OrderBook. Matching engine обрабатывает символы последовательно.

**Persistence.**
- Critical path data не пишется в обычную БД синхронно (это добавило бы disk latency).
- Order/Execution обрабатываются в memory, затем через Sequencer/Event Store персистятся асинхронно .
- Snapshot in-memory tree-maps в БД периодически. В случае ошибки — recover из snapshot и дедупликация через cache .

Клеппман в «Высоконагруженных приложениях» объясняет, что проектирование данных — это компромисс между нормализацией и денормализацией, между шардированием и cross-shard запросами. И выбор зависит от того, какие запросы у тебя в системе.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как проектирование данных влияет на архитектуру. Если у тебя matching engine, order book в памяти обязателен.

**Схема 23.4 — Схема данных: order book, price levels, order map**

```
OrderBook (per symbol):
┌─────────────────────────────────────────────────────────────┐
│ Bids (descending)              │ Asks (ascending)           │
├────────────────────────────────┼────────────────────────────┤
│ Price 100.50 → Level           │ Price 100.60 → Level       │
│   Head → Order A (09:00:01)    │   Head → Order X (09:00:05)│
│   → Order B (09:00:03)         │   → Order Y (09:00:07)     │
│   → Tail                       │   → Tail                   │
│   TotalQty: 500                │   TotalQty: 300            │
├────────────────────────────────┼────────────────────────────┤
│ Price 100.40 → Level           │ Price 100.70 → Level       │
│   Head → Order C               │   Head → Order Z           │
│   TotalQty: 200                │   TotalQty: 100            │
└────────────────────────────────┴────────────────────────────┘

OrderMap (HashMap):
┌─────────────────────────────────────────────────────────────┐
│ Order A → *Order (points to node in PriceLevel 100.50)     │
│ Order B → *Order (points to node in PriceLevel 100.50)     │
│ Order X → *Order (points to node in PriceLevel 100.60)     │
│ ...                                                         │
└─────────────────────────────────────────────────────────────┘

Memory layout:
- All orders: ~128 bytes each
- Active orders: ~2M → ~256 MB (fits in L3 cache)
- Order book depth: ~10,000 levels per symbol
```

---

### 23.5 Matching engine: price-time priority, алгоритмы, сложность

Теперь давай разберём matching engine — сердце биржи.

**Price-time priority (FIFO).**
- **Buy side:** higher price first. Same price → earlier order first .
- **Sell side:** lower price first. Same price → earlier order first .

Это означает: buy orders сортируются по убыванию цены, затем по возрастанию времени. Sell orders — по возрастанию цены, затем по возрастанию времени.

**Matching algorithm.**
Когда приходит aggressive order (taker):
1. Определить противоположную сторону (buy → sell book, sell → buy book).
2. Проверить best price. Если цена не пересекается (limit order), положить в book.
3. Если пересекается: matching против best price level.
4. Match против head order в price level (FIFO). Создать execution.
5. Если taker order полностью filled: stop.
6. Если maker order полностью filled: удалить из book, перейти к следующему order в level.
7. Если price level пуст: перейти к следующему price level.
8. Повторять до полного fill или пока цена не перестанет пересекаться.

**Сложность.**
- Match: O(k), где k — количество исполнений. Каждый match O(1) (head removal).
- Add limit order (no match): O(log P) для поиска price level + O(1) для добавления в linked list.
- Cancel: O(1) после hash lookup.
- Best price: O(1) с указателем.

**Почему single-threaded.**
- Строгий порядок: все ордера должны обрабатываться в детерминированном порядке. Multi-threading требует locks, что добавляет latency и непредсказуемость .
- Cache locality: single-threaded, pinned to CPU core, no context switch .
- 95% messages — cancels. Random-delete priority queue — доминирующая нагрузка .

**Multi-symbol scaling.**
- Один matching engine thread на символ (или на группу символов).
- Каждый thread pinned to CPU core.
- Shared memory для коммуникации между gateway и matching engine.

Клеппман в «Высоконагруженных приложениях» объясняет, что matching engine — это компромисс между latency и throughput. Single-threaded даёт lowest latency, но ограничивает throughput. Multi-symbol scaling даёт throughput, но требует больше cores. И выбор зависит от того, что важнее для задачи.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как matching engine влияет на архитектуру. Если у тебя ultra-low-latency, single-threaded обязателен.

**Схема 23.5 — Matching engine: price-time priority, алгоритмы, сложность**

```
PRICE-TIME PRIORITY:

Buy Book (Bids, descending):
┌─────────────────────────────────────────────────────────────┐
│ Price 100.60: [Order B1 (09:00:01), Order B2 (09:00:03)]   │
│ Price 100.50: [Order B3 (09:00:05)]                        │
│ Price 100.40: [Order B4 (09:00:07)]                        │
└─────────────────────────────────────────────────────────────┘

Sell Book (Asks, ascending):
┌─────────────────────────────────────────────────────────────┐
│ Price 100.70: [Order S1 (09:00:02), Order S2 (09:00:04)]   │
│ Price 100.80: [Order S3 (09:00:06)]                        │
└─────────────────────────────────────────────────────────────┘

MATCHING:

Aggressive Buy Order: 150 shares @ 100.70 (limit)
         │
         ▼
1. Best ask = 100.70. Price crosses (100.70 <= 100.70).
2. Match against S1 (100 shares). Execution: 100 @ 100.70.
3. S1 fully filled. Remove from book.
4. Remaining: 50 shares. Match against S2 (200 shares).
   Execution: 50 @ 100.70.
5. Aggressive order fully filled. Stop.

Result:
- S1: filled 100/100 (removed)
- S2: filled 50/200 (remaining 150)
- Trade tape: 2 executions

COMPLEXITY:

Operation          │ Complexity
───────────────────┼────────────
Match (per fill)   │ O(1)
Add limit order    │ O(log P) + O(1)
Cancel             │ O(1) after lookup
Best price         │ O(1) with pointer

SINGLE-THREADED:

Gateway ──shared memory──▶ Matching Engine (pinned CPU core)
                              │
                              ├── No locks
                              ├── No context switch
                              ├── No network hops
                              └── Cache-friendly

Why: 95% messages are cancels. Random-delete priority queue.
Locks would add unpredictable latency.
```

---

### 23.6 Критический путь: почему микросервисы не работают

Теперь давай разберём критический путь. Это самая важная часть биржи.

**Critical path** — это путь от входа ордера до выхода исполнения:
1. Order comes into Order Manager.
2. Mandatory risk checks.
3. Order gets matched and execution is sent back.
4. Execution comes out of Order Manager .

**Что НЕ на критическом пути:**
- Persistence (async, sequencer log).
- Market data publishing (async).
- Audit & reporting (async).
- Clearing house integration (async).
- Risk management (post-trade).

**Почему микросервисы не работают на критическом пути.**
- Один network round trip: hundreds of microseconds.
- Несколько microservice hops: milliseconds.
- Синхронный disk persist: tens of milliseconds.
- Modern exchange target: tens of microseconds .

Поэтому: deploy all components in a single giant server (no containers), use shared memory as event bus, key components single-threaded and pinned to CPU .

**Architecture.**
```
┌─────────────────────────────────────────────────────────────┐
│                    Single Giant Server                      │
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ Gateway     │───▶│ Order       │───▶│ Matching    │     │
│  │ (network)   │    │ Manager     │    │ Engine      │     │
│  │             │    │ (pinned CPU)│    │ (pinned CPU)│     │
│  └─────────────┘    └─────────────┘    └──────┬──────┘     │
│                                                │            │
│  ┌─────────────┐    ┌─────────────┐           │            │
│  │ Market Data │◀───│ Shared      │◀──────────┘            │
│  │ Publisher   │    │ Memory      │                        │
│  │ (async)     │    │ Ring Buffer │                        │
│  └─────────────┘    └─────────────┘                        │
│                                                             │
│  ┌─────────────┐    ┌─────────────┐                        │
│  │ Persistence │    │ Audit       │                        │
│  │ (async)     │    │ (async)     │                        │
│  └─────────────┘    └─────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

Клеппман в «Высоконагруженных приложениях» объясняет, что критический путь — это компромисс между latency и гибкостью. Микросервисы гибче, но добавляют network hops. Single server быстрее, но менее гибок. И выбор зависит от того, что важнее для задачи.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как критический путь влияет на архитектуру. Если у тебя ultra-low-latency, single server обязателен .

**Схема 23.6 — Критический путь: почему микросервисы не работают**

```
CRITICAL PATH (single server, shared memory):

Member ──network──▶ Gateway ──shared mem──▶ Order Manager
                                                    │
                                                    ├── Risk checks
                                                    ├── Sequencer
                                                    └── Matching Engine
                                                              │
                                                              ▼
                                                    Execution ──shared mem──▶ Gateway ──network──▶ Member

Latency budget:
- Network in: ~5 µs (co-located)
- Gateway parse: ~1 µs
- Shared memory: ~0.1 µs
- Risk checks: ~1 µs
- Matching: ~1 µs
- Shared memory: ~0.1 µs
- Gateway format: ~1 µs
- Network out: ~5 µs
Total: ~14 µs

MICROSERVICES (NOT for critical path):

Member ──network──▶ API Gateway ──network──▶ Order Service
                                                    │
                                                    ├── network ──▶ Risk Service
                                                    ├── network ──▶ Matching Service
                                                    └── network ──▶ Persistence

Latency:
- Each network hop: ~100-500 µs
- 4 hops: ~400-2000 µs (0.4-2 ms)
- Plus disk persist: ~10 ms
Total: ~10-12 ms

10x-1000x slower than single server.
NOT acceptable for ultra-low-latency exchange.
```

---

### 23.7 Market data: ITCH, WebSocket, fan-out

Теперь давай разберём market data — как доставлять order book и trades клиентам.

**ITCH protocol.**
- NASDAQ's protocol for market data.
- Binary, compact messages (28 bytes for Add Order).
- Sequenced: each message has monotonic sequence number. Gap-detectable.
- Delivered over MoldUDP64 (sequenced UDP multicast) .

**Почему multicast UDP.**
- One-to-many: 1,000+ subscribers receive same data simultaneously.
- Low latency: no per-subscriber connection overhead.
- Scalable: publisher sends once, network replicates.

**Order book snapshots + deltas.**
- Initial snapshot: full order book (10,000 levels).
- Deltas: Add Order, Cancel Order, Execute Order, Modify Order.
- Subscribers apply deltas to maintain real-time order book.

**WebSocket для retail.**
- Retail clients (web, mobile) не могут использовать multicast UDP.
- WebSocket для retail: server pushes market data over persistent connection.
- Lower throughput, higher latency than ITCH, but works in browsers.

**Fan-out.**
- Market Data Publisher reads from shared memory, formats ITCH messages, sends via multicast.
- Subscribers (HFT firms, brokers, data vendors) receive and process.
- For retail: WebSocket gateway subscribes to ITCH feed, fans out to clients.

Клеппман в «Высоконагруженных приложениях» объясняет, что market data — это компромисс между latency и масштабируемостью. Multicast UDP быстрее, но сложнее. WebSocket проще, но медленнее. И выбор зависит от того, кто клиенты.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как market data влияет на архитектуру. Если у тебя HFT clients, multicast UDP обязателен.

**Схема 23.7 — Market data: ITCH, WebSocket, fan-out**

```
ITCH over MoldUDP64:

Matching Engine ──shared mem──▶ Market Data Publisher
                                      │
                                      ├──1. Read deltas from shared memory
                                      ├──2. Format as ITCH messages
                                      │      (Add Order: 28 bytes, Cancel: 20 bytes, Execute: 28 bytes)
                                      ├──3. Assign sequence number (monotonic)
                                      └──4. Send via UDP multicast

Multicast Network:
┌─────────────────────────────────────────────────────────────┐
│  Publisher ──▶ multicast group (239.x.x.x:port)            │
│                                                             │
│  Subscribers (1,000+):                                      │
│  ├── HFT Firm A (co-located)                               │
│  ├── HFT Firm B (co-located)                               │
│  ├── Broker C (remote)                                      │
│  └── Data Vendor D (remote)                                 │
└─────────────────────────────────────────────────────────────┘

ITCH Message (Add Order):
┌─────────────────────────────────────────────────────────────┐
│ Message Type: 'A' (1 byte)                                  │
│ Sequence Number: 8 bytes (monotonic, gap-detectable)        │
│ Order ID: 8 bytes                                           │
│ Side: 1 byte (B/S)                                          │
│ Shares: 4 bytes                                             │
│ Symbol: 4 bytes                                             │
│ Price: 4 bytes (fixed-point)                                │
└─────────────────────────────────────────────────────────────┘

WEBSOCKET (retail):

WebSocket Gateway:
┌─────────────────────────────────────────────────────────────┐
│  Subscribes to ITCH feed                                    │
│  Fans out to retail clients via WebSocket                   │
│                                                             │
│  Client A ──WS──▶ Gateway ──▶ (filters by symbol)          │
│  Client B ──WS──▶ Gateway ──▶ (filters by symbol)          │
│  Client C ──WS──▶ Gateway ──▶ (filters by symbol)          │
└─────────────────────────────────────────────────────────────┘
```

---

### 23.8 Отказоустойчивость, мониторинг и безопасность

Теперь давай разберём, как обеспечить отказоустойчивость, мониторинг и безопасность.

**Отказоустойчивость.**
- **Failover:** matching engine — single point of failure для символа. Нужен hot standby. Sequencer log реплицируется на standby. Если primary падает, standby восстанавливается из snapshot + replay log .
- **Snapshot:** periodically iterate treemap and snapshot into database. В случае ошибки — recover from snapshot and de-duplicate with cache .
- **Sequencer:** all commands (orders, cancels) go through sequencer. Sequencer log is the source of truth. Deterministic replay for recovery.
- **No locks on critical path:** single-threaded, no context switch, no unpredictable latency .

**Мониторинг.**
- **Latency:** p50, p99, p999 matching latency (target: < 100 µs).
- **Throughput:** messages/sec, executions/sec.
- **Queue depth:** sequencer queue, matching engine input queue.
- **Cache hit rate:** order book in L3 cache.
- **Market data:** multicast gap detection, subscriber lag.
- **Алерты:** latency p99 > 100 µs, queue depth > threshold, multicast gaps > 0.

**Безопасность.**
- **Authentication:** FIX session login, OUCH session login. Co-location access control.
- **Authorization:** member can only trade their own account.
- **Rate limiting:** max messages/sec per member (prevent quote stuffing).
- **Risk checks:** mandatory on critical path. Max order size, max position, max notional. Pre-trade risk checks .
- **Market abuse surveillance:** spoofing, layering, wash trading detection (post-trade) .
- **Audit:** all orders, cancels, executions logged immutably. Retention: years (regulatory).

Клеппман в «Высоконагруженных приложениях» объясняет, что отказоустойчивость, мониторинг и безопасность — это не фичи, а свойства системы. Без них система не может эксплуатироваться. И выбор зависит от того, насколько критична система.

Книга Алекса Сюя «System Design. Подготовка к сложному интервью» показывает, как отказоустойчивость, мониторинг и безопасность влияют на архитектуру. Если у тебя биржа, они обязательны.

**Схема 23.8 — Полная архитектура с отказоустойчивостью и мониторингом**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Single Giant Server (Primary)               │
│                                                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │
│  │ Gateway     │───▶│ Order       │───▶│ Matching    │             │
│  │             │    │ Manager     │    │ Engine      │             │
│  │             │    │ (pinned)    │    │ (pinned)    │             │
│  └─────────────┘    └─────────────┘    └──────┬──────┘             │
│                                                │                    │
│  ┌─────────────┐    ┌─────────────┐           │                    │
│  │ Market Data │◀───│ Shared      │◀──────────┘                    │
│  │ Publisher   │    │ Memory      │                                │
│  └─────────────┘    └─────────────┘                                │
│                                                                     │
│  ┌─────────────┐    ┌─────────────┐                                │
│  │ Sequencer   │    │ Persistence │                                │
│  │ Log (async) │    │ (async)     │                                │
│  └─────────────┘    └─────────────┘                                │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ replication (sequencer log)
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Single Giant Server (Standby)               │
│                                                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │
│  │ Gateway     │    │ Order       │    │ Matching    │             │
│  │ (standby)   │    │ Manager     │    │ Engine      │             │
│  │             │    │ (standby)   │    │ (standby)   │             │
│  └─────────────┘    └─────────────┘    └─────────────┘             │
│                                                                     │
│  Recovery: snapshot + replay sequencer log                         │
└─────────────────────────────────────────────────────────────────────┘

Monitoring:
┌─────────────────────────────────────────────────────────────────────┐
│  Latency: p50, p99, p999 (< 100 µs)                                │
│  Throughput: messages/sec, executions/sec                          │
│  Queue depth: sequencer, matching engine                            │
│  Cache hit rate: L3 cache                                           │
│  Market data: multicast gap detection, subscriber lag              │
│  Alerts: p99 > 100 µs, queue depth > threshold, gaps > 0          │
└─────────────────────────────────────────────────────────────────────┘

Security:
┌─────────────────────────────────────────────────────────────────────┐
│  FIX/OUCH session auth ──▶ Rate limiting ──▶ Risk checks           │
│  (critical path) ──▶ Market abuse surveillance (post-trade)        │
│  Audit log (immutable, years)                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Выводы

В этой главе ты узнал, как строить биржу. Ты разобрал order book, matching engine, price-time priority. Ты узнал, почему критический путь не может быть распределённым, и почему все компоненты ставятся на один сервер. Ты узнал, что такое ITCH и как доставлять market data тысячам клиентов. Ты увидел, как те же строительные блоки (shared memory, multicast UDP, sequencer) применяются к ultra-low-latency задаче.

Главное, что ты должен вынести из этой главы: **stock exchange — это задача о real-time matching и ultra-low latency.** Хорошая архитектура — не та, которая использует самые модные микросервисы. Хорошая архитектура — та, которая обеспечивает минимальную задержку, строгий порядок и fairness для всех участников.

---

## Выжимка для быстрого повторения

- **Требования:** ввод/отмена ордеров, matching, market data. Ultra-low latency (10–100 µs), 1M+ msg/s, строгий порядок, fairness.
- **Оценки:** NASDAQ peak 1.1M msg/s, Deutsche Börse 8M msg/s inbound, 300K msg/s matching. 95% cancels. Active orders ~2M, ~256 MB (fits L3 cache).
- **API:** OUCH binary (49 bytes Enter Order), FIX for institutional, REST for retail. Price fixed-point, not float.
- **Данные:** OrderBook (SortedMap price → PriceLevel), PriceLevel (doubly linked FIFO), OrderMap (HashMap O(1) cancel).
- **Matching:** price-time priority. Buy: high price, early time. Sell: low price, early time. Match O(1) per fill. Single-threaded, pinned CPU.
- **Критический путь:** single server, shared memory, no network hops, no disk I/O. Risk checks mandatory. Async persistence off critical path.
- **Market data:** ITCH over MoldUDP64 (multicast UDP). Sequenced, gap-detectable. WebSocket for retail.
- **Отказоустойчивость:** hot standby, sequencer log replication, snapshot + replay. **Мониторинг:** latency p99 < 100 µs, queue depth, cache hit rate. **Безопасность:** FIX/OUCH auth, rate limiting, risk checks, market abuse surveillance, audit.

---

## Вопросы для самопроверки

1. Почему matching engine должен быть single-threaded? Почему микросервисы не работают на критическом пути?
2. Как работает price-time priority? Какой порядок для buy и sell?
3. Почему order book использует SortedMap + doubly linked list + HashMap? Какую сложность даёт каждая структура?
4. Почему 95% сообщений — это отмены? Как это влияет на дизайн?
5. Как доставлять market data тысячам клиентов? Почему multicast UDP?

---

## Эталонные ответы

**1. Почему matching engine должен быть single-threaded? Почему микросервисы не работают на критическом пути?**

Single-threaded: строгий порядок (все ордера обрабатываются в детерминированном порядке), no locks (locks добавляют unpredictable latency), cache locality (no context switch, pinned CPU core). Микросервисы: один network hop = 100–500 µs, несколько hops = milliseconds. Modern exchange target = 10–100 µs. Поэтому все компоненты на одном сервере, shared memory как event bus, no network hops на критическом пути .

**2. Как работает price-time priority? Какой порядок для buy и sell?**

Buy side: higher price first. Same price → earlier order first (FIFO). Sell side: lower price first. Same price → earlier order first (FIFO). Это означает: buy orders сортируются по убыванию цены, затем по возрастанию времени. Sell orders — по возрастанию цены, затем по возрастанию времени. Aggressive order matching против противоположной стороны, начиная с best price .

**3. Почему order book использует SortedMap + doubly linked list + HashMap? Какую сложность даёт каждая структура?**

SortedMap (Tree/SkipList): быстрый поиск price level. O(log P) для lookup/create. Doubly linked list (внутри PriceLevel): FIFO queue. Add в tail O(1), Match из head O(1). HashMap (OrderMap): O(1) cancel lookup. Критично, потому что cancels outnumber executions 20:1 . Все вместе дают O(1) для match и cancel, O(log P) для добавления нового price level.

**4. Почему 95% сообщений — это отмены? Как это влияет на дизайн?**

HFT firms и market makers постоянно обновляют quotes (cancel/replace). В US equities cancel:execute ratio ~20:1 . Это означает: matching engine должен быть оптимизирован для random-delete priority queue, не только для matching. OrderMap (HashMap) для O(1) cancel lookup критичен. Doubly linked list для O(1) removal. Без этих структур cancel был бы O(n) или O(log n), что добавило бы latency .

**5. Как доставлять market data тысячам клиентов? Почему multicast UDP?**

ITCH over MoldUDP64: binary messages (28 bytes Add Order), sequenced (monotonic sequence number, gap-detectable), delivered over multicast UDP. Multicast: publisher sends once, network replicates to 1,000+ subscribers. No per-subscriber connection overhead. Low latency. Subscribers apply deltas (Add, Cancel, Execute) to maintain real-time order book. For retail (browsers): WebSocket gateway subscribes to ITCH feed, fans out to clients .

---

## Мост к следующей главе

Ты спроектировал биржу — от требований до продакшена. Ты узнал, что такое order book, matching engine, price-time priority, критический путь. Ты увидел, как те же строительные блоки (shared memory, multicast UDP, sequencer) применяются к ultra-low-latency задаче.

Но биржа — это про торговлю финансовыми инструментами. А что если задача — это координация действий в реальном времени между множеством игроков? Что если нужно обеспечить fairness, low latency и consistency для игроков по всему миру? Что если каждый игрок видит мир по-своему, и сервер должен синхронизировать их состояние?

В следующей главе — **Real-time Gaming Leaderboard: ранжирование, обновления, масштабирование**. Ты узнаешь, как строить лидерборд для миллионов игроков, как обновлять рейтинги в реальном времени, как масштабировать на глобальные аудитории. Ты узнаешь, что такое Redis Sorted Sets и почему они подходят для лидербордов. И ты поймёшь, что gaming leaderboard — это задача о real-time ranking и consistency, где миллионы игроков соревнуются за место в топе.