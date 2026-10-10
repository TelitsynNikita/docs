# 🔌 Глава 38: WebSocket и streaming

**Что вы узнаете:**
- Что такое WebSocket и чем он отличается от HTTP.
- Как построить **production-ready WebSocket-сервер** на Go.
- Как организовать **hub** для управления соединениями.
- Как реализовать **broadcast** и **приватные сообщения**.
- Как обрабатывать **разрывы соединений** и **reconnect**.
- Как добавить **heartbeat** (ping/pong) для обнаружения мёртвых клиентов.
- Как реализовать **backpressure** для медленных клиентов.
- Как сделать **graceful shutdown** WebSocket-сервера.
- Как **мониторить** соединения, throughput, ошибки.

**После прочтения вы сможете:**
- Построить WebSocket-сервер с hub'ом.
- Организовать broadcast и приватные сообщения.
- Обрабатывать heartbeat и reconnect.
- Реализовать backpressure для медленных клиентов.
- Корректно завершать WebSocket-сервер.
- Мониторить соединения и throughput.

---

## Содержание

- [38.0 Пролог: чат, который терял сообщения](#380-пролог-чат-который-терял-сообщения)
- [38.1 Что такое WebSocket](#381-что-такое-websocket)
- [38.2 Базовая структура](#382-базовая-структура)
- [38.3 Hub: управление соединениями](#383-hub-управление-соединениями)
- [38.4 Broadcast и приватные сообщения](#384-broadcast-и-приватные-сообщения)
- [38.5 Heartbeat: ping/pong](#385-heartbeat-pingpong)
- [38.6 Backpressure: медленные клиенты](#386-backpressure-медленные-клиенты)
- [38.7 Graceful shutdown](#387-graceful-shutdown)
- [38.8 Мониторинг: connections, throughput, errors](#388-мониторинг-connections-throughput-errors)
- [38.9 Полный production-ready WebSocket-сервер](#389-полный-production-ready-websocket-сервер)
- [38.10 Выводы и типичные ошибки](#3810-выводы-и-типичные-ошибки)
- [38.11 Для быстрого повторения](#3811-для-быстрого-повторения)
- [38.12 Вопросы для самопроверки](#3812-вопросы-для-самопроверки)
- [38.13 Ответы](#3813-ответы)
- [38.14 Куда идти дальше?](#3814-куда-идти-дальше)
- [38.15 Чек-лист](#3815-чек-лист)

---

## 38.0 Пролог: чат, который терял сообщения

У нас есть чат на Go. 1000 пользователей. Каждый держит WebSocket-соединение. Пишем наивно:

```go
var conns []*websocket.Conn

func handleWS(w http.ResponseWriter, r *http.Request) {
    conn, _ := upgrader.Upgrade(w, r, nil)
    conns = append(conns, conn)

    for {
        msgType, msg, err := conn.ReadMessage()
        if err != nil {
            return
        }
        // Broadcast всем
        for _, c := range conns {
            c.WriteMessage(msgType, msg)  // ← блокируется на медленных
        }
    }
}
```

Работает. Пока не замечаем:

- **Один медленный клиент** блокирует broadcast.
- **Все остальные** ждут.
- **Сообщения** накапливаются.
- **Через час** — сервер падает.

**Что произошло:**

- `c.WriteMessage` **блокируется**, если клиент медленный.
- Один медленный клиент **задерживает** всех.
- **Нет backpressure** для медленных.
- **Нет ping/pong** — мёртвые соединения не обнаруживаются.

Хочется: **production-ready WebSocket-сервер**, который:

1. **Не блокируется** на медленных клиентах.
2. **Обнаруживает** мёртвые соединения через ping/pong.
3. **Обрабатывает** backpressure.
4. **Поддерживает** broadcast и приватные сообщения.
5. **Корректно завершается** по сигналу.
6. **Мониторится** через метрики.

> **Мост к следующим главам:** WebSocket — синтез паттернов: hub (Глава 8), broadcast (Глава 10), backpressure (Глава 27), heartbeat (Глава 17), graceful shutdown (Глава 26). Понимание WebSocket даёт понимание, **как строить real-time сервисы**.

---

## 38.1 Что такое WebSocket

**WebSocket** — протокол двусторонней связи между клиентом и сервером.

### Отличие от HTTP

**HTTP:**

- **Запрос-ответ.** Клиент спрашивает — сервер отвечает.
- **Короткое соединение.** Закрывается после ответа.
- **Односторонне.** Сервер не может инициировать.

**WebSocket:**

- **Постоянное соединение.** Открыто, пока не закроют.
- **Двусторонне.** Обе стороны могут отправлять.
- **Real-time.** Низкая задержка.

### Как работает

```
HTTP Upgrade:
  Клиент: GET /ws Upgrade: websocket
  Сервер: 101 Switching Protocols
  │
  ▼
WebSocket-соединение (постоянное):
  Клиент ◄──────────────► Сервер
         Сообщения в обе стороны
```

**Upgrade** — HTTP-запрос с заголовками:

- `Upgrade: websocket`
- `Connection: Upgrade`
- `Sec-WebSocket-Key`

Сервер отвечает `101 Switching Protocols` — соединение становится WebSocket.

### Когда использовать

**1. Real-time.**

- Чаты.
- Уведомления.
- Live-обновления.

**2. Двусторонняя связь.**

- Игры.
- Совместная работа.
- Стриминг.

**3. Низкая задержка.**

- Трейдинг.
- Мониторинг.

### Когда НЕ использовать

**1. Одноразовые запросы.**

Для обычных API — HTTP.

**2. Большие файлы.**

Для загрузки — HTTP.

**3. Stateless.**

Если не нужна постоянная связь — HTTP.

### 💡 Практика: как думать о WebSocket

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **WebSocket — для real-time.**
2. **Hub** — для управления соединениями.
3. **Ping/pong** — для обнаружения мёртвых.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Backpressure** для медленных.
5. **Метрики.**

**❌ НЕ ДЕЛАЙ:**

6. **Не блокируй broadcast.**
7. **Не игнорируй мёртвые соединения.**

---

## 38.2 Базовая структура

Начнём с простейшего WebSocket-сервера.

### Библиотека

**`github.com/gorilla/websocket`** — стандарт де-факто.

```bash
go get github.com/gorilla/websocket
```

### Upgrader

```go
var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin: func(r *http.Request) bool {
        return true  // ← в production проверяй origin
    },
}
```

**Что важно:**

- **`ReadBufferSize` / `WriteBufferSize`** — буферы.
- **`CheckOrigin`** — защита от CSRF.

### Обработчик

```go
func handleWS(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Println("upgrade error:", err)
        return
    }
    defer conn.Close()

    for {
        msgType, msg, err := conn.ReadMessage()
        if err != nil {
            log.Println("read error:", err)
            return
        }
        log.Printf("received: %s", msg)

        if err := conn.WriteMessage(msgType, msg); err != nil {
            log.Println("write error:", err)
            return
        }
    }
}

func main() {
    http.HandleFunc("/ws", handleWS)
    http.ListenAndServe(":8080", nil)
}
```

**Что происходит:**

- Клиент подключается к `/ws`.
- Сервер читает сообщения и эхо-отвечает.
- **Один клиент — одна горутина.**

### Схема

```
Клиент 1 ──► handleWS ──► conn1
Клиент 2 ──► handleWS ──► conn2
Клиент 3 ──► handleWS ──► conn3
```

**Проблема:** каждый клиент **изолирован**. Нет broadcast.

### Клиент

```javascript
const ws = new WebSocket("ws://localhost:8080/ws");

ws.onopen = () => {
    console.log("connected");
    ws.send("hello");
};

ws.onmessage = (event) => {
    console.log("received:", event.data);
};

ws.onclose = () => {
    console.log("disconnected");
};
```

### 💡 Практика: как начать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`gorilla/websocket`.**
2. **`upgrader.Upgrade`.**
3. **`CheckOrigin`** в production.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Буферы Read/Write.**
5. **`defer conn.Close()`.**

**❌ НЕ ДЕЛАЙ:**

6. **Не игнорируй `CheckOrigin`.**
7. **Не забывай `conn.Close()`.**

---

## 38.3 Hub: управление соединениями

**Hub** — центральная структура для управления соединениями.

### Идея

- **Hub** держит **все** соединения.
- Клиенты **регистрируются** и **удаляются**.
- Broadcast идёт **через hub**.

### Структура

```go
type Client struct {
    conn *websocket.Conn
    send chan []byte
    hub  *Hub
}

type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
    mu         sync.RWMutex
}
```

**Что важно:**

- **`clients`** — все клиенты.
- **`broadcast`** — канал для broadcast.
- **`register` / `unregister`** — каналы регистрации.
- **`mu`** — защита `clients`.

### Run hub

```go
func (h *Hub) Run(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        case client := <-h.register:
            h.mu.Lock()
            h.clients[client] = true
            h.mu.Unlock()

        case client := <-h.unregister:
            h.mu.Lock()
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }
            h.mu.Unlock()

        case msg := <-h.broadcast:
            h.mu.RLock()
            for client := range h.clients {
                select {
                case client.send <- msg:
                default:
                    // Буфер полон — удаляем клиента
                    close(client.send)
                    delete(h.clients, client)
                }
            }
            h.mu.RUnlock()
        }
    }
}
```

### Client: read и write

**Каждый клиент** имеет **две горутины:**

- **Read** — читает из WebSocket, пишет в hub.
- **Write** — читает из `send`, пишет в WebSocket.

```go
func (c *Client) ReadPump(ctx context.Context) {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()

    c.conn.SetReadLimit(512 * 1024)  // 512 KB
    c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })

    for {
        _, msg, err := c.conn.ReadMessage()
        if err != nil {
            return
        }
        c.hub.broadcast <- msg
    }
}

func (c *Client) WritePump(ctx context.Context) {
    ticker := time.NewTicker(54 * time.Second)  // ping каждые 54 сек
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case <-ctx.Done():
            return
        case msg, ok := <-c.send:
            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }
            c.conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
            if err := c.conn.WriteMessage(websocket.TextMessage, msg); err != nil {
                return
            }
        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}
```

### Обработчик

```go
func (h *Hub) HandleWS(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        return
    }

    client := &Client{
        conn: conn,
        send: make(chan []byte, 256),  // буфер
        hub:  h,
    }

    h.register <- client

    go client.WritePump(r.Context())
    client.ReadPump(r.Context())
}
```

### Схема

```
Клиент 1 ──► ReadPump ──► hub.broadcast ──┐
                                          │
Клиент 2 ──► ReadPump ──► hub.broadcast ──┼──► hub.Run
                                          │
Клиент 3 ──► ReadPump ──► hub.broadcast ──┘
                                          │
                                          ▼
                              ┌──────────────────────┐
                              │  broadcast канал     │
                              └──────────────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
                    ▼                     ▼                     ▼
              Client 1.send         Client 2.send         Client 3.send
                    │                     │                     │
                    ▼                     ▼                     ▼
              WritePump 1           WritePump 2           WritePump 3
```

### 💡 Практика: как построить hub

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Hub** — центральная структура.
2. **Каналы `register` / `unregister` / `broadcast`.**
3. **Две горутины на клиента** (read/write).

**👍 СТОИТ СДЕЛАТЬ:**

4. **Буфер `send` 256.**
5. **`SetReadLimit`** — лимит.

**❌ НЕ ДЕЛАЙ:**

6. **Не пиши в `send` из нескольких горутин.**
7. **Не блокируй `broadcast`.**

---

## 38.4 Broadcast и приватные сообщения

Broadcast — **всем**. Приватные — **одному**.

### Broadcast

**Уже реализован** через `hub.broadcast`:

```go
h.broadcast <- msg
```

**Что происходит:**

- Все клиенты получают сообщение.
- Медленные — удаляются.

### Приватные сообщения

**Идея:** каждый клиент имеет **ID**. Отправляем конкретному.

```go
type Client struct {
    id   string  // ← ID клиента
    conn *websocket.Conn
    send chan []byte
    hub  *Hub
}

type PrivateMessage struct {
    ClientID string
    Data     []byte
}

type Hub struct {
    // ...
    clients   map[*Client]bool
    byID      map[string]*Client  // ← по ID
    private   chan PrivateMessage
}
```

### Обработка приватных

```go
case pm := <-h.private:
    h.mu.RLock()
    client, ok := h.byID[pm.ClientID]
    h.mu.RUnlock()
    if !ok {
        continue
    }
    select {
    case client.send <- pm.Data:
    default:
        // Буфер полон
    }
```

### Комнаты (rooms)

**Группы клиентов:**

```go
type Room struct {
    name    string
    clients map[*Client]bool
    mu      sync.RWMutex
}

type Hub struct {
    // ...
    rooms map[string]*Room
}
```

**Broadcast в комнату:**

```go
func (h *Hub) BroadcastToRoom(room string, msg []byte) {
    h.mu.RLock()
    r, ok := h.rooms[room]
    h.mu.RUnlock()
    if !ok {
        return
    }

    r.mu.RLock()
    defer r.mu.RUnlock()
    for client := range r.clients {
        select {
        case client.send <- msg:
        default:
            // Буфер полон
        }
    }
}
```

### Схема

```
Broadcast:
  hub.broadcast ──► ВСЕ клиенты

Private:
  hub.private ──► ОДИН клиент (по ID)

Room:
  room.broadcast ──► клиенты КОМНАТЫ
```

### 💡 Практика: как отправлять сообщения

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Broadcast** — через hub.
2. **Private** — через `byID`.
3. **Rooms** — для групповых.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики** — сколько сообщений.
5. **Логирование.**

**❌ НЕ ДЕЛАЙ:**

6. **Не пиши в `conn` напрямую.**
7. **Не блокируй на медленных.**

---

## 38.5 Heartbeat: ping/pong

**Ping/pong** — обнаружение мёртвых соединений.

### Проблема

**Что если клиент упал?**

- TCP-соединение **не закрыто**.
- Сервер **не знает**.
- Клиент **висит** в hub.
- Память **растёт**.

### Решение: ping/pong

**Сервер отправляет ping** каждые N секунд.
**Клиент отвечает pong.**
**Если pong не пришёл** — соединение мёртво.

### Реализация

**В `WritePump`:**

```go
ticker := time.NewTicker(54 * time.Second)  // ping каждые 54 сек
defer ticker.Stop()

for {
    select {
    case msg, ok := <-c.send:
        // ...
    case <-ticker.C:
        c.conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
        if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
            return
        }
    }
}
```

**В `ReadPump`:**

```go
c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
c.conn.SetPongHandler(func(string) error {
    c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    return nil
})
```

**Что происходит:**

- Ping каждые 54 секунды.
- Pong обновляет `ReadDeadline` на 60 секунд.
- Если pong не пришёл за 60 секунд — `ReadMessage` вернёт ошибку.

### Схема

```
Сервер ──ping──► Клиент
Сервер ◄──pong── Клиент

Если pong не пришёл за 60 сек:
  ReadMessage возвращает ошибку
  → соединение закрывается
```

### Параметры

**Рекомендации:**

- **Ping interval:** 54 сек.
- **Pong wait:** 60 сек.
- **Ping wait:** 10 сек (write deadline).

**Формула:** `pongWait > pingPeriod`.

### 💡 Практика: как настроить ping/pong

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Ping** каждые 54 сек.
2. **Pong handler** обновляет deadline.
3. **Read deadline** 60 сек.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Write deadline** 10 сек.

**❌ НЕ ДЕЛАЙ:**

5. **Не игнорируй мёртвые соединения.**
6. **Не делай ping чаще 30 сек.**

---

## 38.6 Backpressure: медленные клиенты

Backpressure (Глава 27) — **критичен** для WebSocket.

### Проблема

**Что если клиент медленный?**

- Сервер пишет в `conn.WriteMessage`.
- **Блокируется**.
- Остальные **ждут**.
- **Все тормозят.**

### Решение: буфер `send` + `select default`

**Клиент имеет буфер `send`:**

```go
type Client struct {
    send chan []byte  // буфер 256
}
```

**Broadcast:**

```go
select {
case client.send <- msg:
    // OK
default:
    // Буфер полон — удаляем клиента
    close(client.send)
    delete(h.clients, client)
}
```

**Что происходит:**

- Если буфер полон — **не блокируемся**.
- **Удаляем** медленного клиента.
- Остальные **работают**.

### Схема

```
Broadcast:
  │
  ├── Client 1.send (буфер 256) ──► ok
  ├── Client 2.send (буфер 256) ──► ok
  ├── Client 3.send (буфер полон) ──► удаляем
  └── Client 4.send (буфер 256) ──► ok
```

### Размер буфера

**Рекомендации:**

- **Маленький (10–50)** — быстрый отсев.
- **Средний (256–1024)** — большинство случаев.
- **Большой (10 000+)** — маскирует проблему.

### Что теряется

**Медленный клиент теряет сообщения:**

- Буфер полон → удаление.
- Клиент **переподключается**.
- Может получить **последнее состояние**.

**Для критичных данных:**

- **Не удаляй** клиента.
- **Жди** его.

**Для real-time:**

- **Удаляй** клиента.
- **Пусть reconnect**.

### 💡 Практика: как обрабатывать backpressure

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Буфер `send` 256.**
2. **`select default`** при broadcast.
3. **Удаление медленных.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики** — сколько удалено.
5. **Реконнект** на клиенте.

**❌ НЕ ДЕЛАЙ:**

6. **Не блокируй broadcast.**
7. **Не делай буфер 10 000+.**

---

## 38.7 Graceful shutdown

Graceful shutdown (Глава 26) — **обязателен**.

### Что должно произойти

1. **HTTP-сервер** — `Shutdown`.
2. **Hub** — закрытие всех соединений.
3. **Клиенты** — `CloseMessage`.
4. **Горутины** — завершение.

### Реализация

```go
func (h *Hub) Shutdown(ctx context.Context) {
    h.mu.Lock()
    for client := range h.clients {
        // Отправляем close message
        select {
        case client.send <- websocket.FormatCloseMessage(websocket.CloseNormalClosure, "server shutdown"):
        case <-ctx.Done():
            h.mu.Unlock()
            return
        }
    }
    h.mu.Unlock()

    // Ждём закрытия
    for {
        h.mu.RLock()
        n := len(h.clients)
        h.mu.RUnlock()
        if n == 0 {
            return
        }
        select {
        case <-ctx.Done():
            return
        case <-time.After(100 * time.Millisecond):
        }
    }
}
```

### Main

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    hub := NewHub()
    go hub.Run(ctx)

    mux := http.NewServeMux()
    mux.HandleFunc("/ws", hub.HandleWS)

    srv := &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }

    go srv.ListenAndServe()

    <-ctx.Done()
    log.Println("shutdown signal received")

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    // Закрываем WebSocket-соединения
    hub.Shutdown(shutdownCtx)

    // Закрываем HTTP-сервер
    srv.Shutdown(shutdownCtx)

    log.Println("shutdown complete")
}
```

### Схема

```
SIGTERM:
  │
  ├── Hub.Shutdown:
  │   ├── CloseMessage каждому клиенту
  │   └── Ждём закрытия
  │
  └── srv.Shutdown:
      └── Ждём HTTP-обработчики
```

### 💡 Практика: как завершать WebSocket-сервер

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`signal.NotifyContext`.**
2. **`Hub.Shutdown`** — CloseMessage.
3. **`srv.Shutdown`** — HTTP.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Таймаут** 10 сек.
5. **Логирование** — сколько соединений.

**❌ НЕ ДЕЛАЙ:**

6. **Не убивай соединения.**
7. **Не забывай CloseMessage.**

---

## 38.8 Мониторинг: connections, throughput, errors

Мониторинг — **обязателен**.

### Ключевые метрики

| Метрика | Что измеряет |
|:---|:---|
| **Active connections** | Число активных |
| **Messages sent** | Отправлено |
| **Messages received** | Получено |
| **Bytes sent/received** | Трафик |
| **Errors** | Ошибок |
| **Slow clients removed** | Удалено медленных |
| **Ping/pong RTT** | Round-trip time |

### Метрики

```go
type Metrics struct {
    ActiveConnections atomic.Int64
    MessagesSent      atomic.Int64
    MessagesReceived  atomic.Int64
    BytesSent         atomic.Int64
    BytesReceived     atomic.Int64
    Errors            atomic.Int64
    SlowClientsRemoved atomic.Int64
}
```

**В hub:**

```go
case client := <-h.register:
    h.metrics.ActiveConnections.Add(1)
    // ...

case client := <-h.unregister:
    h.metrics.ActiveConnections.Add(-1)
    // ...

case msg := <-h.broadcast:
    h.metrics.MessagesSent.Add(int64(len(h.clients)))
    h.metrics.BytesSent.Add(int64(len(msg)) * int64(len(h.clients)))
    // ...
```

### Prometheus

```go
var (
    activeConnections = prometheus.NewGauge(
        prometheus.GaugeOpts{Name: "websocket_active_connections"},
    )
    messagesTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "websocket_messages_total"},
        []string{"direction"},
    )
    slowClientsRemoved = prometheus.NewCounter(
        prometheus.CounterOpts{Name: "websocket_slow_clients_removed_total"},
    )
)
```

### Что алертить

- **Active connections > N** — перегрузка.
- **Errors > N/сек** — проблемы.
- **Slow clients removed > N/сек** — медленные клиенты.

### 💡 Практика: как мониторить WebSocket

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Active connections** — ключевая метрика.
2. **Messages sent/received.**
3. **Errors.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Bytes sent/received.**
5. **Slow clients removed.**

**❌ НЕ ДЕЛАЙ:**

6. **Не игнорируй active connections.**
7. **Не игнорируй slow clients.**

---

## 38.9 Полный production-ready WebSocket-сервер

Соберём всё вместе.

### Полный код

```go
package main

import (
    "context"
    "errors"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "sync"
    "sync/atomic"
    "syscall"
    "time"

    "github.com/gorilla/websocket"
)

const (
    writeWait      = 10 * time.Second
    pongWait       = 60 * time.Second
    pingPeriod     = (pongWait * 9) / 10
    maxMessageSize = 512 * 1024
    sendBufferSize = 256
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin: func(r *http.Request) bool {
        return true  // ← в production проверяй
    },
}

type Metrics struct {
    ActiveConnections  atomic.Int64
    MessagesSent       atomic.Int64
    MessagesReceived   atomic.Int64
    BytesSent          atomic.Int64
    BytesReceived      atomic.Int64
    Errors             atomic.Int64
    SlowClientsRemoved atomic.Int64
}

type Client struct {
    conn    *websocket.Conn
    send    chan []byte
    hub     *Hub
    metrics *Metrics
}

type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
    mu         sync.RWMutex
    metrics    *Metrics
}

func NewHub() *Hub {
    return &Hub{
        clients:    make(map[*Client]bool),
        broadcast:  make(chan []byte, 256),
        register:   make(chan *Client),
        unregister: make(chan *Client),
        metrics:    &Metrics{},
    }
}

func (h *Hub) Run(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return

        case client := <-h.register:
            h.mu.Lock()
            h.clients[client] = true
            h.mu.Unlock()
            h.metrics.ActiveConnections.Add(1)

        case client := <-h.unregister:
            h.mu.Lock()
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }
            h.mu.Unlock()
            h.metrics.ActiveConnections.Add(-1)

        case msg := <-h.broadcast:
            h.mu.RLock()
            for client := range h.clients {
                select {
                case client.send <- msg:
                    h.metrics.MessagesSent.Add(1)
                    h.metrics.BytesSent.Add(int64(len(msg)))
                default:
                    h.metrics.SlowClientsRemoved.Add(1)
                    close(client.send)
                    delete(h.clients, client)
                }
            }
            h.mu.RUnlock()
        }
    }
}

func (h *Hub) HandleWS(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        h.metrics.Errors.Add(1)
        return
    }

    client := &Client{
        conn:    conn,
        send:    make(chan []byte, sendBufferSize),
        hub:     h,
        metrics: h.metrics,
    }

    h.register <- client

    go client.WritePump(r.Context())
    client.ReadPump(r.Context())
}

func (c *Client) ReadPump(ctx context.Context) {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()

    c.conn.SetReadLimit(maxMessageSize)
    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        select {
        case <-ctx.Done():
            return
        default:
        }

        _, msg, err := c.conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
                c.metrics.Errors.Add(1)
                slog.Error("read error", "error", err)
            }
            return
        }

        c.metrics.MessagesReceived.Add(1)
        c.metrics.BytesReceived.Add(int64(len(msg)))

        select {
        case c.hub.broadcast <- msg:
        case <-ctx.Done():
            return
        }
    }
}

func (c *Client) WritePump(ctx context.Context) {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case <-ctx.Done():
            return

        case msg, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            if err := c.conn.WriteMessage(websocket.TextMessage, msg); err != nil {
                return
            }

        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}

func (h *Hub) Shutdown(ctx context.Context) {
    h.mu.Lock()
    for client := range h.clients {
        select {
        case client.send <- websocket.FormatCloseMessage(websocket.CloseNormalClosure, "server shutdown"):
        case <-ctx.Done():
            h.mu.Unlock()
            return
        }
    }
    h.mu.Unlock()

    for {
        h.mu.RLock()
        n := len(h.clients)
        h.mu.RUnlock()
        if n == 0 {
            return
        }
        select {
        case <-ctx.Done():
            return
        case <-time.After(100 * time.Millisecond):
        }
    }
}

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    slog.SetDefault(logger)

    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    hub := NewHub()
    go hub.Run(ctx)

    mux := http.NewServeMux()
    mux.HandleFunc("/ws", hub.HandleWS)

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    go func() {
        slog.Info("server starting", "addr", srv.Addr)
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            slog.Error("server error", "error", err)
        }
    }()

    <-ctx.Done()
    slog.Info("shutdown signal received")

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    hub.Shutdown(shutdownCtx)
    srv.Shutdown(shutdownCtx)

    slog.Info("shutdown complete",
        "total_messages_sent", hub.metrics.MessagesSent.Load(),
        "total_messages_received", hub.metrics.MessagesReceived.Load(),
        "slow_clients_removed", hub.metrics.SlowClientsRemoved.Load(),
    )
}
```

### Что демонстрирует

1. **Hub** — управление соединениями.
2. **Read/Write pumps** — две горутины на клиента.
3. **Ping/pong** — обнаружение мёртвых.
4. **Backpressure** — буфер `send` + `select default`.
5. **Graceful shutdown** — CloseMessage + ожидание.
6. **Метрики** — connections, messages, slow clients.
7. **Таймауты** — Read, Write, Idle.

### 💡 Практика: как строить production WebSocket

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Hub** — центральная структура.
2. **Read/Write pumps.**
3. **Ping/pong.**
4. **Backpressure.**
5. **Graceful shutdown.**

**👍 СТОИТ СДЕЛАТЬ:**

6. **Метрики** — connections, messages.
7. **Таймауты.**

**❌ НЕ ДЕЛАЙ:**

8. **Не блокируй broadcast.**
9. **Не игнорируй slow clients.**

---

## 38.10 Выводы и типичные ошибки

**Что мы узнали?**

WebSocket — протокол двусторонней связи. **Hub** управляет соединениями. **Read/Write pumps** — две горутины на клиента. **Ping/pong** обнаруживает мёртвых. **Backpressure** через буфер `send`. **Graceful shutdown** через CloseMessage. **Метрики:** connections, messages, slow clients.

**Типичные ошибки:**

- ❌ **Блокировка на `WriteMessage`.** Медленный клиент тормозит всех.
- ❌ **Нет ping/pong.** Мёртвые соединения не обнаруживаются.
- ❌ **Нет backpressure.** Буфер растёт.
- ❌ **Огромный буфер `send`.** Маскирует проблему.
- ❌ **Нет graceful shutdown.** Обрыв соединений.
- ❌ **Нет метрик.** Не видно проблем.
- ❌ **Нет таймаутов.** Зависание.
- ❌ **Нет `CheckOrigin`.** CSRF.
- ❌ **Пишут в `conn` из нескольких горутин.** Race.
- ❌ **Не читают из `conn`.** Буфер TCP растёт.

---

## 38.11 Для быстрого повторения

- **WebSocket** — двусторонняя связь.
- **Hub** — управление соединениями.
- **Read/Write pumps** — две горутины на клиента.
- **`register` / `unregister` / `broadcast`** — каналы hub.
- **Ping/pong** — обнаружение мёртвых.
- **Ping 54 сек, Pong wait 60 сек.**
- **Backpressure** — буфер `send` 256.
- **`select default`** — не блокировать.
- **Slow clients removed** — метрика.
- **Graceful shutdown** — CloseMessage.
- **`SetReadLimit`** — 512 KB.
- **`SetReadDeadline` / `SetWriteDeadline`.**
- **`CheckOrigin`** — CSRF.

---

## 38.12 Вопросы для самопроверки

1. Чем WebSocket отличается от HTTP?
2. Что такое hub?
3. Зачем две горутины на клиента?
4. Что такое ping/pong?
5. Что такое backpressure в WebSocket?
6. Как правильно завершать WebSocket-сервер?
7. Что мониторить в WebSocket?
8. Почему один медленный клиент тормозит всех?

---

## 38.13 Ответы

### Ответ 1

**HTTP:** запрос-ответ, короткое, одностороннее.

**WebSocket:** постоянное, двустороннее, real-time.

Upgrade: HTTP → WebSocket через `101 Switching Protocols`.

### Ответ 2

**Hub** — центральная структура для управления соединениями. Держит `clients`, каналы `register`/`unregister`/`broadcast`.

### Ответ 3

**Read pump** — читает из WebSocket.
**Write pump** — пишет в WebSocket.

**Зачем:** WebSocket **не потокобезопасен**. Нельзя читать и писать из одной горутины.

### Ответ 4

**Ping/pong** — обнаружение мёртвых соединений.

Сервер шлёт **ping** каждые 54 сек. Клиент отвечает **pong**. Если pong не пришёл за 60 сек — соединение закрывается.

### Ответ 5

**Backpressure** — буфер `send` на клиента (256).

При broadcast — `select default`. Если буфер полон — удаляем клиента. **Не блокируемся.**

### Ответ 6

**Graceful shutdown:**

1. `ctx.Done()`.
2. **CloseMessage** каждому клиенту.
3. Ждём закрытия.
4. `srv.Shutdown`.

### Ответ 7

**Мониторинг:**

- **Active connections.**
- **Messages sent/received.**
- **Bytes sent/received.**
- **Errors.**
- **Slow clients removed.**

### Ответ 8

**Один медленный клиент** блокирует `WriteMessage`.

**Что происходит:** broadcast блокируется на медленном. Все остальные ждут.

**Решение:** буфер `send` + `select default` — не блокируемся, удаляем медленного.

---

## 38.14 Куда идти дальше?

Мы разобрали WebSocket — real-time связь. Теперь мы умеем строить production-ready WebSocket-сервер.

Осталась **последняя глава** — безопасность конкурентного кода.

- **Как обеспечить безопасность?** → **Глава 39: Безопасность конкурентного кода.**

---

## 38.15 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **WebSocket** | Двусторонняя связь | Постоянное соединение |
| **Upgrade** | HTTP → WS | 101 Switching Protocols |
| **Hub** | Управление | clients, register, unregister, broadcast |
| **Read/Write pumps** | Две горутины | На клиента |
| **Ping** | 54 сек | Heartbeat |
| **Pong wait** | 60 сек | Обнаружение мёртвых |
| **`send` buffer** | 256 | Backpressure |
| **`select default`** | Не блокировать | Удаление медленных |
| **Slow clients removed** | Метрика | — |
| **CloseMessage** | Graceful shutdown | — |
| **`SetReadLimit`** | 512 KB | Защита |
| **Таймауты** | Read, Write | — |
| **`CheckOrigin`** | CSRF | — |
| **Метрики** | connections, messages | Prometheus |

🔌 **Ключевая идея:** WebSocket — протокол двусторонней связи поверх HTTP (Upgrade → 101). **Hub** управляет соединениями через каналы `register`/`unregister`/`broadcast`. **Две горутины на клиента** (Read/Write pumps) — WebSocket не потокобезопасен. **Ping/pong** (54/60 сек) обнаруживает мёртвых. **Backpressure** через буфер `send` 256 + `select default` — не блокируем broadcast, удаляем медленных. **Graceful shutdown** — CloseMessage + ожидание. **Метрики:** active connections, messages sent/received, slow clients removed. **Таймауты** Read/Write. **`CheckOrigin`** для CSRF.