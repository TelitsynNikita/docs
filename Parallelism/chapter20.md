# 🌐 Глава 20: WebSocket, SSE, gRPC streaming

**Что вы узнаете:**
- Почему **HTTP request/response** не подходит для real-time.
- Что такое **WebSocket** и как он работает.
- Как построить **WebSocket-сервер** на Go с hub-паттерном.
- Что такое **Server-Sent Events (SSE)** и чем он отличается от WebSocket.
- Как реализовать **gRPC bidirectional streaming**.
- Что такое **backpressure** в streaming.
- Как делать **heartbeat** и **reconnect**.
- Как **graceful shutdown** для streaming-соединений.
- Как **масштабировать** streaming на несколько узлов.
- Как **тестировать** streaming.

**После прочтения вы сможете:**
- Построить WebSocket-сервер с hub-паттерном.
- Реализовать SSE для односторонней передачи.
- Написать gRPC bidirectional streaming.
- Обрабатывать backpressure в streaming.
- Делать heartbeat и graceful shutdown.
- Масштабировать streaming через pub/sub.
- Тестировать streaming-соединения.

---

## Содержание

- [20.0 Пролог: чат, который не масштабируется](#200-пролог-чат-который-не-масштабируется)
- [20.1 Почему HTTP не подходит для real-time](#201-почему-http-не-подходит-для-real-time)
- [20.2 WebSocket: полнодуплексный канал](#202-websocket-полнодуплексный-канал)
- [20.3 WebSocket-сервер: hub-паттерн](#203-websocket-сервер-hub-паттерн)
- [20.4 Server-Sent Events: односторонний поток](#204-server-sent-events-односторонний-поток)
- [20.5 gRPC bidirectional streaming](#205-grpc-bidirectional-streaming)
- [20.6 Backpressure в streaming](#206-backpressure-в-streaming)
- [20.7 Heartbeat и reconnect](#207-heartbeat-и-reconnect)
- [20.8 Graceful shutdown для streaming](#208-graceful-shutdown-для-streaming)
- [20.9 Масштабирование streaming](#209-масштабирование-streaming)
- [20.10 Практика Go: WebSocket-чат с hub](#2010-практика-go-websocket-чат-с-hub)
- [20.11 Выводы и типичные ошибки](#2011-выводы-и-типичные-ошибки)
- [20.12 Для быстрого повторения](#2012-для-быстрого-повторения)
- [20.13 Вопросы для самопроверки](#2013-вопросы-для-самопроверки)
- [20.14 Ответы](#2014-ответы)
- [20.15 Куда идти дальше?](#2015-куда-идти-дальше)
- [20.16 Чек-лист](#2016-чек-лист)

---

## 20.0 Пролог: чат, который не масштабируется

Ты пишешь чат на Go. HTTP-сервер, 1000 пользователей.

**Наивная реализация:**

```go
func handler(w http.ResponseWriter, r *http.Request) {
    for {
        msg := getNewMessage()  // polling
        json.NewEncoder(w).Encode(msg)
        time.Sleep(100 * time.Millisecond)
    }
}
```

**Проблемы:**

- **Polling каждые 100 мс** — 10 000 запросов/сек на 1000 пользователей.
- **Latency до 100 мс.**
- **Пустые ответы** — 99% запросов без данных.
- **CPU и сеть** тратятся зря.

❓ **Что произошло?** HTTP **request/response** не подходит для **real-time**. Нужен **push** от сервера.

💡 **Решение:** **WebSocket**. Одно соединение, **полнодуплексный** канал, сервер **сам** отправляет сообщения.

```go
func handler(w http.ResponseWriter, r *http.Request) {
    conn, _ := upgrader.Upgrade(w, r, nil)
    defer conn.Close()

    for msg := range messagesCh {
        conn.WriteJSON(msg)  // push
    }
}
```

**Что изменилось:**

- **Одно соединение** на пользователя.
- **Push** от сервера.
- **Latency ~1 мс.**
- **Нет polling.**

**Это WebSocket.** В этой главе — WebSocket, SSE, gRPC streaming, hub-паттерн, backpressure, heartbeat, graceful shutdown.

> **Важный мост:** streaming — это **долгоживущие** соединения. Graceful shutdown (Глава 11) сложнее. Backpressure (Глава 8) критичен. Hub-паттерн — это **fan-in/fan-out** (Глава 8). Глава 27 (Distributed tracing) — для диагностики.

---

## 20.1 Почему HTTP не подходит для real-time

Прежде чем разбирать WebSocket, поймём **ограничения** HTTP.

### HTTP request/response

**Модель:**

1. Клиент отправляет запрос.
2. Сервер обрабатывает.
3. Сервер возвращает ответ.
4. **Соединение закрывается** (или переиспользуется для нового запроса).

**Ключевое:** **инициатива у клиента**. Сервер **не может** отправить данные без запроса.

### Проблемы для real-time

**1. Polling.**

Клиент **периодически** спрашивает «есть новое?».

**Проблемы:**

- **Latency:** до интервала polling.
- **Пустые запросы:** 99% без данных.
- **Нагрузка:** 10 000 запросов/сек.

**2. Long polling.**

Клиент держит запрос **открытым**, пока сервер не ответит.

**Проблемы:**

- **Много соединений.**
- **Сложнее реализация.**
- **Timeout на прокси.**

**3. HTTP/2 Server Push.**

Сервер может **отправить** данные до запроса.

**Проблемы:**

- **Плохо поддерживается** браузерами.
- **Не работает** для push после response.

### Что нужно для real-time

**1. Push от сервера.**

Сервер **сам** отправляет данные, когда есть.

**2. Полнодуплексность.**

Клиент и сервер могут **одновременно** отправлять.

**3. Одно соединение.**

Не открывать новое на каждый запрос.

**4. Низкая latency.**

~1 мс вместо 100 мс.

### Технологии

| Технология | Направление | Протокол | Браузеры |
|:---|:---|:---|:---|
| **Polling** | Client → Server | HTTP | ✅ |
| **Long polling** | Client → Server | HTTP | ✅ |
| **SSE** | Server → Client | HTTP | ✅ |
| **WebSocket** | Bidirectional | WS | ✅ |
| **gRPC streaming** | Bidirectional | HTTP/2 | ❌ (не в браузере) |

### Выбор

**1. Только сервер → клиент (уведомления, логи).**

→ **SSE**.

**2. Bidirectional (чат, игры).**

→ **WebSocket**.

**3. Микросервисы (gRPC).**

→ **gRPC streaming**.

### Аннотация сложности

| Технология | Latency | Connections | Сложность |
|:---|:---|:---|:---|
| Polling | ~100 мс | N × частота | Низкая |
| Long polling | ~1 мс | N | Средняя |
| SSE | ~1 мс | N | Низкая |
| WebSocket | ~1 мс | N | Средняя |
| gRPC | ~1 мс | N | Средняя |

### 💡 Практика: как выбрать технологию

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **SSE** — для сервер → клиент.
2. **WebSocket** — для bidirectional.
3. **gRPC streaming** — для микросервисов.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Fallback на polling** — если WebSocket не работает.

**❌ НЕ ДЕЛАЙ:**

5. **Не используй polling** для real-time.
6. **Не используй WebSocket** для однонаправленной передачи.

### Ключевые выводы подглавы 20.1

- **HTTP request/response** — инициатива у клиента.
- **Polling** — высокий latency, пустые запросы.
- **Real-time** требует **push** от сервера.
- **SSE** — сервер → клиент. **WebSocket** — bidirectional.
- **gRPC streaming** — для микросервисов.

---

## 20.2 WebSocket: полнодуплексный канал

**WebSocket** — протокол для **bidirectional** связи.

### Как работает

**Handshake:**

1. Клиент отправляет **HTTP-запрос** с заголовками:
   ```
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key: <random>
   Sec-WebSocket-Version: 13
   ```
2. Сервер отвечает:
   ```
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: <hash>
   ```
3. **Соединение** становится WebSocket.

**После handshake:**

- **Frame-based** протокол.
- **Bidirectional.**
- **Низкая latency.**

### Frame

**WebSocket frame:**

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
```

**Что важно:**

- **Opcode:** text (1), binary (2), close (8), ping (9), pong (10).
- **Mask:** клиент → сервер — всегда маскируется.
- **Payload len:** 7, 16, или 64 бита.

### Ping/Pong

**Ping** — сервер отправляет ping, клиент отвечает pong.

**Зачем:**

- **Проверка живости.**
- **Обнаружение мёртвых соединений.**
- **Heartbeat.**

### Close

**Close frame** — корректное закрытие.

**Что делает:**

- Отправляет close code.
- Ждёт ответа.
- Закрывает соединение.

**Close codes:**

- **1000** — normal.
- **1001** — going away.
- **1002** — protocol error.
- **1008** — policy violation.
- **1009** — message too big.
- **1011** — internal error.

### Go-библиотеки

**1. `github.com/gorilla/websocket`** — старая, но проверенная.

**2. `nhooyr.io/websocket`** (теперь `github.com/coder/websocket`) — современная, context-aware.

**3. `golang.org/x/net/websocket`** — устаревшая, не рекомендуется.

### Пример на `gorilla/websocket`

```go
var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool { return true },
}

func handler(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Println(err)
        return
    }
    defer conn.Close()

    for {
        mt, msg, err := conn.ReadMessage()
        if err != nil {
            break
        }
        if err := conn.WriteMessage(mt, msg); err != nil {
            break
        }
    }
}
```

### Пример на `nhooyr/websocket`

```go
func handler(w http.ResponseWriter, r *http.Request) {
    conn, err := websocket.Accept(w, r, nil)
    if err != nil {
        log.Println(err)
        return
    }
    defer conn.Close(websocket.StatusInternalError, "closing")

    ctx := r.Context()
    for {
        mt, msg, err := conn.Read(ctx)
        if err != nil {
            break
        }
        if err := conn.Write(ctx, mt, msg); err != nil {
            break
        }
    }
}
```

**Ключевое:** `nhooyr` использует `context.Context`.

### Сравнение

| Аспект | gorilla | nhooyr/coder |
|:---|:---|:---|
| Context | ❌ | ✅ |
| Зрелость | ✅ | ✅ |
| Поддержка | Ограниченная | Активная |
| API | Классический | Современный |

**Рекомендация:** `nhooyr/coder/websocket` для новых проектов.

### Аннотация сложности

| Операция | Time |
|:---|:---|
| Handshake | ~1-10 мс |
| `Read` frame | ~10-50 мкс |
| `Write` frame | ~10-50 мкс |
| Ping | ~1-5 мс |

### 💡 Практика: как использовать WebSocket

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`nhooyr/coder/websocket`** — для новых проектов.
2. **Ping/pong** для heartbeat.
3. **Close codes** для корректного закрытия.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`context.Context`** для отмены.
5. **Ограничение размера** сообщений.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `golang.org/x/net/websocket`.**
7. **Не игнорируй ping/pong.**

### Ключевые выводы подглавы 20.2

- **WebSocket** — bidirectional.
- **Handshake** через HTTP Upgrade.
- **Frame-based** протокол.
- **Ping/pong** для heartbeat.
- **`nhooyr/coder`** — рекомендуется.

---

## 20.3 WebSocket-сервер: hub-паттерн

Разберём **hub-паттерн** для WebSocket.

### Проблема

**1000 клиентов** подключены. Сообщение от одного — всем.

**Наивно:**

```go
var conns []*websocket.Conn
var mu sync.Mutex

func broadcast(msg []byte) {
    mu.Lock()
    defer mu.Unlock()
    for _, conn := range conns {
        conn.WriteMessage(websocket.TextMessage, msg)
    }
}
```

**Проблемы:**

- **Mutex на все операции.**
- **Медленный клиент** блокирует всех.
- **Нет backpressure.**

### Решение: hub

**Hub** — центральная горутина, которая управляет клиентами.

**Идея:**

- **Регистрация** через канал.
- **Отправка** через канал.
- **Broadcast** через каналы.

### Структура

```go
type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
}

type Client struct {
    hub  *Hub
    conn *websocket.Conn
    send chan []byte
}
```

### Run

```go
func (h *Hub) Run(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        case client := <-h.register:
            h.clients[client] = true
        case client := <-h.unregister:
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }
        case message := <-h.broadcast:
            for client := range h.clients {
                select {
                case client.send <- message:
                default:
                    // Клиент не успевает — отключаем
                    close(client.send)
                    delete(h.clients, client)
                }
            }
        }
    }
}
```

**Что делает:**

1. **Register** — добавляет клиента.
2. **Unregister** — удаляет.
3. **Broadcast** — рассылает всем.
4. **Non-blocking send** — если клиент не успевает, отключаем.

### Client

```go
func (c *Client) readPump(ctx context.Context) {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()

    c.conn.SetReadLimit(512)
    c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })

    for {
        _, message, err := c.conn.ReadMessage()
        if err != nil {
            break
        }
        c.hub.broadcast <- message
    }
}

func (c *Client) writePump(ctx context.Context) {
    ticker := time.NewTicker(54 * time.Second)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case <-ctx.Done():
            return
        case message, ok := <-c.send:
            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }
            c.conn.WriteMessage(websocket.TextMessage, message)
        case <-ticker.C:
            c.conn.WriteMessage(websocket.PingMessage, nil)
        }
    }
}
```

**Что делает:**

- **readPump** — читает от клиента, отправляет в broadcast.
- **writePump** — пишет клиенту, отправляет ping.

### Схема

```
                  ┌──────────────┐
                  │     Hub      │
                  │              │
                  │  broadcast   │
                  │  register    │
                  │  unregister  │
                  └──────┬───────┘
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
  ┌─────────┐       ┌─────────┐       ┌─────────┐
  │ Client1 │       │ Client2 │       │ Client3 │
  │  send   │       │  send   │       │  send   │
  │  conn   │       │  conn   │       │  conn   │
  └─────────┘       └─────────┘       └─────────┘
```

### Ключевое

**1. Hub — одна горутина.**

Все операции с map — **без блокировок**.

**2. Client — две горутины.**

- **readPump** — читает.
- **writePump** — пишет.

**3. Backpressure через `send`.**

Если `send` полон — клиент отключается.

### Аннотация сложности

| Операция | Time |
|:---|:---|
| Register | ~100-200 нс |
| Broadcast | ~100-200 нс × N |
| Send | ~100-200 нс |

### 💡 Практика: как строить hub

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Hub — одна горутина.**
2. **Client — readPump + writePump.**
3. **Non-blocking send** для broadcast.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Ping/pong** для heartbeat.
5. **Read limit** для защиты.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй Mutex** для клиентов.
7. **Не блокируйся** на медленном клиенте.

### Ключевые выводы подглавы 20.3

- **Hub** — центральная горутина.
- **Client** — readPump + writePump.
- **Non-blocking send** — backpressure.
- **Без Mutex.**

---

## 20.4 Server-Sent Events: односторонний поток

**SSE (Server-Sent Events)** — push от сервера **только**.

### Что это

**SSE** — HTTP-ответ, который **не закрывается**.

**Формат:**

```
event: message
data: {"text": "hello"}

event: message
data: {"text": "world"}

```

**Что важно:**

- **`Content-Type: text/event-stream`**
- **`Cache-Control: no-cache`**
- **`Connection: keep-alive`**

### Когда использовать

**SSE подходит:**

- **Уведомления.**
- **Логи в реальном времени.**
- **Метрики.**
- **Обновления цен.**

**SSE не подходит:**

- **Bidirectional** (чат, игры) — используй WebSocket.

### Реализация

```go
func sseHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")

    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "streaming unsupported", http.StatusInternalServerError)
        return
    }

    ch := subscribe()
    defer unsubscribe(ch)

    for {
        select {
        case <-r.Context().Done():
            return
        case msg := <-ch:
            fmt.Fprintf(w, "data: %s\n\n", msg)
            flusher.Flush()
        }
    }
}
```

**Что делает:**

1. Устанавливает заголовки.
2. **Flush** после каждого сообщения.
3. Читает из канала.
4. Отправляет в SSE-формате.

### Формат SSE

```
data: простая строка

event: custom
data: строка с типом

id: 42
data: строка с id

: комментарий (игнорируется)

data: многострочное
data: сообщение

```

**Поля:**

- **`data`** — данные.
- **`event`** — тип события.
- **`id`** — ID (для reconnect).
- **`retry`** — время reconnect.

### Reconnect

**Браузер** автоматически переподключается.

**`Last-Event-ID`** — браузер отправляет последний ID.

**Сервер** может **продолжить** с этого ID.

### WebSocket vs SSE

| Аспект | WebSocket | SSE |
|:---|:---|:---|
| Направление | Bidirectional | Server → Client |
| Протокол | WS | HTTP |
| Binary | ✅ | ❌ |
| Reconnect | Ручной | Авто |
| Proxy | Сложнее | Проще |
| Браузеры | ✅ | ✅ |

### Аннотация сложности

| Операция | Time |
|:---|:---|
| Открытие | ~1-10 мс |
| Отправка | ~10-50 мкс |
| Reconnect | Авто |

### 💡 Практика: как использовать SSE

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`Content-Type: text/event-stream`.**
2. **`Flush`** после каждого сообщения.
3. **`context.Context`** для отмены.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`id`** для reconnect.
5. **Heartbeat** через комментарии.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй SSE** для bidirectional.
7. **Не забывай flush.**

### Ключевые выводы подглавы 20.4

- **SSE** — push от сервера.
- **`text/event-stream`.**
- **`Flush`** обязателен.
- **Auto-reconnect** в браузере.
- **Не для bidirectional.**

---

## 20.5 gRPC bidirectional streaming

**gRPC streaming** — для микросервисов.

### Три типа streaming

**1. Server streaming.**

Клиент отправляет запрос, сервер отвечает **потоком**.

```proto
rpc ServerStream(Request) returns (stream Response);
```

**2. Client streaming.**

Клиент отправляет **поток**, сервер отвечает **один раз**.

```proto
rpc ClientStream(stream Request) returns (Response);
```

**3. Bidirectional streaming.**

Оба отправляют **потоки**.

```proto
rpc Bidirectional(stream Request) returns (stream Response);
```

### Пример bidirectional

**Proto:**

```proto
syntax = "proto3";

service Chat {
    rpc Chat(stream Message) returns (stream Message);
}

message Message {
    string user = 1;
    string text = 2;
}
```

**Сервер:**

```go
func (s *Server) Chat(stream pb.Chat_ChatServer) error {
    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }

        // Обработка
        response := &pb.Message{
            User: "server",
            Text: "echo: " + msg.Text,
        }

        if err := stream.Send(response); err != nil {
            return err
        }
    }
}
```

**Клиент:**

```go
stream, err := client.Chat(ctx)
if err != nil {
    return err
}

// Отправка
go func() {
    for _, msg := range messages {
        if err := stream.Send(msg); err != nil {
            return
        }
    }
    stream.CloseSend()
}()

// Получение
for {
    resp, err := stream.Recv()
    if err == io.EOF {
        break
    }
    if err != nil {
        return err
    }
    fmt.Println(resp.Text)
}
```

### Ключевое

**1. `Recv` и `Send` — блокирующие.**

**2. `io.EOF` — поток закрыт.**

**3. `context.Context` — отмена.**

**4. Потоки мультиплексируются** на одном HTTP/2.

### HTTP/2

**gRPC работает поверх HTTP/2.**

**Что даёт:**

- **Multiplexing** — несколько потоков на одном соединении.
- **Bidirectional** — в одном соединении.
- **Compression** — заголовки.

### Backpressure

**gRPC** имеет **встроенный** flow control через HTTP/2.

**Что делает:**

- **Окно** на приём.
- **Приостановка** отправки, если получатель не успевает.

**Но:** нужно обрабатывать `Recv`/`Send` асинхронно.

### Пример с backpressure

```go
func (s *Server) Chat(stream pb.Chat_ChatServer) error {
    for {
        msg, err := stream.Recv()
        if err != nil {
            return err
        }

        // Обработка может быть медленной
        // Backpressure через flow control HTTP/2
        select {
        case <-stream.Context().Done():
            return stream.Context().Err()
        default:
        }

        if err := stream.Send(msg); err != nil {
            return err
        }
    }
}
```

### Ошибки

**`codes.Unavailable`** — сервис недоступен.

**`codes.DeadlineExceeded`** — таймаут.

**`codes.Canceled`** — отмена.

**`codes.ResourceExhausted`** — перегрузка.

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `Recv` | ~10-50 мкс |
| `Send` | ~10-50 мкс |
| Установка потока | ~1-10 мс |

### 💡 Практика: как использовать gRPC streaming

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`context.Context`** для отмены.
2. **Обрабатывай `io.EOF`** — поток закрыт.
3. **Flow control** HTTP/2.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Асинхронный `Recv`/`Send`** — если нужно.
5. **Метрики** — latency, throughput.

**❌ НЕ ДЕЛАЙ:**

6. **Не блокируйся** на `Recv`/`Send` без отмены.
7. **Не игнорируй коды ошибок.**

### Ключевые выводы подглавы 20.5

- **gRPC streaming** — 3 типа.
- **Bidirectional** — оба потока.
- **HTTP/2** — multiplexing.
- **Flow control** — встроенный backpressure.
- **`context.Context`** — отмена.

---

## 20.6 Backpressure в streaming

Разберём **backpressure** в streaming.

### Проблема

**Быстрый producer**, медленный consumer.

```
Producer ────▶ [канал] ────▶ Consumer
             рост          медленно
```

**Что происходит:** буфер растёт, память растёт, OOM.

### В WebSocket

**Проблема:** `conn.WriteMessage` **блокирует**, если клиент медленный.

**Решение:** **буферизованный `send` + non-blocking**.

```go
type Client struct {
    send chan []byte  // буфер
}

func (h *Hub) broadcast(message []byte) {
    for client := range h.clients {
        select {
        case client.send <- message:
            // OK
        default:
            // Буфер полон — клиент не успевает
            close(client.send)
            delete(h.clients, client)
        }
    }
}
```

**Что делает:**

- **Non-blocking send** в `send`.
- Если буфер полон — клиент **отключается**.

**Размер буфера:**

- **256-1024** сообщений.
- Больше — память.
- Меньше — отключаем часто.

### В SSE

**Проблема:** `fmt.Fprintf` + `Flush` **блокирует**.

**Решение:** **буферизованный канал** + **drop**.

```go
func sseHandler(w http.ResponseWriter, r *http.Request) {
    ch := make(chan string, 256)

    go func() {
        for msg := range ch {
            fmt.Fprintf(w, "data: %s\n\n", msg)
            flusher.Flush()
        }
    }()

    for {
        select {
        case msg := <-ch:
            // ...
        default:
            // Буфер полон — drop
        }
    }
}
```

### В gRPC

**gRPC** имеет **встроенный** flow control HTTP/2.

**Что делает:**

- **Окно** на приём.
- **Приостановка** отправки.
- **Автоматически.**

**Что нужно:** обрабатывать `Recv`/`Send` **асинхронно**.

### Паттерн: bounded queue

**Идея:** ограниченная очередь + стратегия.

**Стратегии:**

**1. Drop oldest.**

```go
select {
case ch <- msg:
default:
    <-ch  // удалить старый
    ch <- msg
}
```

**2. Drop newest.**

```go
select {
case ch <- msg:
default:
    // drop
}
```

**3. Block.**

```go
select {
case ch <- msg:
case <-ctx.Done():
    return
}
```

**4. Disconnect.**

```go
select {
case ch <- msg:
default:
    close(ch)  // отключить клиента
}
```

### Выбор стратегии

| Стратегия | Когда |
|:---|:---|
| Drop oldest | Логи, метрики |
| Drop newest | Уведомления |
| Block | Критичные данные |
| Disconnect | Медленный клиент |

### Аннотация сложности

| Операция | Time |
|:---|:---|
| `select` + `default` | ~1-5 нс |
| Blocking send | ~50-100 нс |
| Drop | ~1-5 нс |

### 💡 Практика: как обрабатывать backpressure

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Буферизованный `send`.**
2. **Non-blocking send.**
3. **Стратегия** — drop или disconnect.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики** — сколько drop.
5. **Размер буфера** — 256-1024.

**❌ НЕ ДЕЛАЙ:**

6. **Не блокируйся** на медленном клиенте.
7. **Не делай буфер безграничным.**

### Ключевые выводы подглавы 20.6

- **Backpressure** — критичен для streaming.
- **Буферизованный `send`** + **non-blocking**.
- **Стратегии:** drop oldest, drop newest, block, disconnect.
- **gRPC** — встроенный flow control.
- **Метрики** — сколько drop.

---

## 20.7 Heartbeat и reconnect

Разберём **heartbeat** и **reconnect**.

### Heartbeat

**Heartbeat** — периодические сообщения для проверки живости.

**Зачем:**

- **Обнаружить мёртвые соединения.**
- **Держать NAT/proxy открытыми.**
- **Обновлять таймауты.**

### Ping/pong в WebSocket

```go
func (c *Client) writePump() {
    ticker := time.NewTicker(54 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            c.conn.WriteMessage(websocket.PingMessage, nil)
        case message := <-c.send:
            c.conn.WriteMessage(websocket.TextMessage, message)
        }
    }
}

func (c *Client) readPump() {
    c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })

    for {
        _, message, err := c.conn.ReadMessage()
        if err != nil {
            break
        }
        // ...
    }
}
```

**Что делает:**

- **Ping каждые 54 сек.**
- **Read deadline 60 сек.**
- **Pong обновляет deadline.**

**Таймауты:**

- **Ping interval < Pong timeout.**
- **Read deadline > Ping interval.**

**Рекомендация:**

- **Ping: 30-60 сек.**
- **Read deadline: 60-120 сек.**

### Heartbeat в SSE

**SSE** не имеет ping/pong.

**Решение:** **комментарии**.

```go
ticker := time.NewTicker(30 * time.Second)
defer ticker.Stop()

for {
    select {
    case <-ticker.C:
        fmt.Fprintf(w, ": heartbeat\n\n")
        flusher.Flush()
    case msg := <-ch:
        fmt.Fprintf(w, "data: %s\n\n", msg)
        flusher.Flush()
    }
}
```

**Что делает:** отправляет комментарий каждые 30 сек.

### Reconnect

**WebSocket:** reconnect **ручной**.

**Клиент:**

```go
func connect(url string) {
    for {
        conn, _, err := websocket.DefaultDialer.Dial(url, nil)
        if err != nil {
            time.Sleep(1 * time.Second)
            continue
        }

        err = handle(conn)
        conn.Close()

        time.Sleep(1 * time.Second)
    }
}
```

**Что делает:**

- **Пытается подключиться.**
- **Обрабатывает.**
- **При ошибке — retry.**

**Backoff:**

- **Exponential** для reconnect.
- **Jitter.**

### SSE reconnect

**Браузер** автоматически переподключается.

**Что делает:**

1. Соединение закрылось.
2. Ждёт `retry` (по умолчанию 3 сек).
3. Переподключается с `Last-Event-ID`.

**Сервер:**

```go
lastID := r.Header.Get("Last-Event-ID")
// Продолжить с lastID
```

### Аннотация сложности

| Операция | Time |
|:---|:---|
| Ping | ~1-5 мс |
| Read deadline | ~1-5 мс |
| Reconnect | ~1-10 сек |

### 💡 Практика: как использовать heartbeat

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Ping/pong** для WebSocket.
2. **Комментарии** для SSE.
3. **Read deadline** для обнаружения мёртвых.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Reconnect с backoff** на клиенте.
5. **`Last-Event-ID`** для SSE.

**❌ НЕ ДЕЛАЙ:**

6. **Не игнорируй heartbeat.**
7. **Не делай reconnect без backoff.**

### Ключевые выводы подглавы 20.7

- **Heartbeat** — для проверки живости.
- **Ping/pong** в WebSocket.
- **Комментарии** в SSE.
- **Reconnect** — ручной (WS) или авто (SSE).
- **Backoff** для reconnect.

---

## 20.8 Graceful shutdown для streaming

Разберём **graceful shutdown** для streaming.

### Проблема

**Streaming-соединения долгоживущие.** `srv.Shutdown` **ждёт** их завершения. Но они **не завершаются** сами.

### Что нужно

1. **Прекратить принимать** новые соединения.
2. **Отправить close** активным.
3. **Дождаться** завершения.
4. **Таймаут** для защиты.

### WebSocket

```go
func (c *Client) close() {
    c.conn.WriteControl(
        websocket.CloseMessage,
        websocket.FormatCloseMessage(websocket.CloseGoingAway, "server shutting down"),
        time.Now().Add(5*time.Second),
    )
    c.conn.Close()
}
```

**Что делает:**

- Отправляет close frame.
- Ждёт 5 сек.
- Закрывает.

### Hub shutdown

```go
func (h *Hub) Shutdown(ctx context.Context) {
    // Отправить close всем клиентам
    for client := range h.clients {
        client.close()
    }

    // Ждать завершения
    select {
    case <-ctx.Done():
        log.Println("shutdown timeout")
    default:
        // Ждём немного
        time.Sleep(100 * time.Millisecond)
    }
}
```

### srv.Shutdown

**С `srv.Shutdown`:**

```go
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
defer stop()

srv := &http.Server{Addr: ":8080", Handler: mux}
go srv.ListenAndServe()

<-ctx.Done()

// 1. Закрыть все WebSocket
hub.Shutdown(context.Background())

// 2. Shutdown HTTP
shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
srv.Shutdown(shutdownCtx)
```

**Важно:** сначала закрыть streaming, потом HTTP.

### gRPC

**gRPC** имеет `GracefulStop`:

```go
grpcServer.GracefulStop()  // ждёт завершения
```

**С таймаутом:**

```go
done := make(chan struct{})
go func() {
    grpcServer.GracefulStop()
    close(done)
}()

select {
case <-done:
    log.Println("graceful shutdown complete")
case <-time.After(5 * time.Second):
    grpcServer.Stop()  // force
}
```

### SSE

**SSE:** `r.Context()` отменяется при `srv.Shutdown`. Обработчик **завершается**.

```go
for {
    select {
    case <-r.Context().Done():
        return  // shutdown
    case msg := <-ch:
        // ...
    }
}
```

### Аннотация сложности

| Операция | Time |
|:---|:---|
| Close frame | ~1-5 мс |
| Shutdown | 0-30 сек |
| Force | ~1-10 мс |

### 💡 Практика: как делать graceful shutdown для streaming

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Close frame** для WebSocket.
2. **`r.Context()`** для SSE.
3. **`GracefulStop`** для gRPC.
4. **Таймаут** для защиты.

**👍 СТОИТ СДЕЛАТЬ:**

5. **Сначала streaming, потом HTTP.**
6. **Логирование** shutdown.

**❌ НЕ ДЕЛАЙ:**

7. **Не жди вечно.**
8. **Не забывай close frame.**

### Ключевые выводы подглавы 20.8

- **Streaming-соединения** долгоживущие.
- **Close frame** для WebSocket.
- **`r.Context()`** для SSE.
- **`GracefulStop`** для gRPC.
- **Таймаут** обязателен.

---

## 20.9 Масштабирование streaming

Разберём **масштабирование** streaming.

### Проблема

**Один сервер** не выдержит 100 000 соединений.

**Решение:** **несколько серверов**.

**Но:** как отправить сообщение клиенту на **другом** сервере?

### Решение: pub/sub

**Идея:** использовать **Redis Pub/Sub** или **NATS** для broadcast между серверами.

**Схема:**

```
  Server 1                    Server 2
  ┌────────┐                  ┌────────┐
  │ Hub 1  │                  │ Hub 2  │
  └───┬────┘                  └───┬────┘
      │                           │
      └───────────┬───────────────┘
                  │
          ┌───────▼────────┐
          │  Redis Pub/Sub │
          └────────────────┘
```

**Что происходит:**

- Server 1 отправляет сообщение в **Redis Pub/Sub**.
- Server 2 **подписан** — получает сообщение.
- Server 2 отправляет клиенту.

### Реализация

```go
type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
    pubsub     *redis.PubSub
}

func (h *Hub) Run(ctx context.Context) {
    // Подписка
    ch := h.pubsub.Channel()

    for {
        select {
        case <-ctx.Done():
            return
        case client := <-h.register:
            h.clients[client] = true
        case client := <-h.unregister:
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }
        case message := <-h.broadcast:
            // Publish в Redis
            h.redis.Publish(ctx, "chat", message)
        case msg := <-ch:
            // Сообщение из Redis — отправить локальным клиентам
            for client := range h.clients {
                select {
                case client.send <- []byte(msg.Payload):
                default:
                    close(client.send)
                    delete(h.clients, client)
                }
            }
        }
    }
}
```

### Sticky sessions

**Проблема:** клиент должен попасть на **тот же** сервер.

**Решение:** **sticky sessions** в load balancer.

**Или:** **stateless** через pub/sub.

### NATS

**NATS** — альтернатива Redis.

**Плюсы:**

- **Быстрее.**
- **Проще.**
- **Встроенный clustering.**

**Использование:**

```go
nc, _ := nats.Connect(nats.DefaultURL)
defer nc.Close()

// Публикация
nc.Publish("chat", []byte("hello"))

// Подписка
nc.Subscribe("chat", func(m *nats.Msg) {
    // ...
})
```

### Kubernetes

**Проблема:** Pods могут меняться.

**Решение:**

- **Service** для стабильного адреса.
- **Sticky sessions** на Ingress.
- **Pub/sub** для координации.

### Аннотация сложности

| Операция | Redis | NATS |
|:---|:---|:---|
| Publish | ~1-5 мс | ~100-500 мкс |
| Subscribe | ~1-5 мс | ~100-500 мкс |

### 💡 Практика: как масштабировать streaming

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Pub/sub** (Redis, NATS) для broadcast.
2. **Sticky sessions** или stateless.
3. **Kubernetes** для оркестрации.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Метрики** — количество соединений.
5. **Алерты** на аномалии.

**❌ НЕ ДЕЛАЙ:**

6. **Не делай stateful** без необходимости.
7. **Не игнорируй pub/sub задержку.**

### Ключевые выводы подглавы 20.9

- **Масштабирование** через pub/sub.
- **Redis Pub/Sub** или **NATS**.
- **Sticky sessions** или stateless.
- **Kubernetes** для оркестрации.

---

## 20.10 Практика Go: WebSocket-чат с hub

Напишем **полный WebSocket-чат** с hub.

### Код

```go
package main

import (
    "context"
    "encoding/json"
    "log"
    "net/http"
    "os/signal"
    "sync"
    "syscall"
    "time"

    "github.com/gorilla/websocket"
)

type Message struct {
    User string `json:"user"`
    Text string `json:"text"`
}

type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
}

func NewHub() *Hub {
    return &Hub{
        clients:    make(map[*Client]bool),
        broadcast:  make(chan []byte, 256),
        register:   make(chan *Client),
        unregister: make(chan *Client),
    }
}

func (h *Hub) Run(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        case client := <-h.register:
            h.clients[client] = true
            log.Printf("client connected, total: %d", len(h.clients))
        case client := <-h.unregister:
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
                log.Printf("client disconnected, total: %d", len(h.clients))
            }
        case message := <-h.broadcast:
            for client := range h.clients {
                select {
                case client.send <- message:
                default:
                    close(client.send)
                    delete(h.clients, client)
                }
            }
        }
    }
}

func (h *Hub) Shutdown() {
    for client := range h.clients {
        client.close()
    }
}

type Client struct {
    hub  *Hub
    conn *websocket.Conn
    send chan []byte
}

var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool { return true },
}

func (h *Hub) HandleWS(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Println(err)
        return
    }

    client := &Client{
        hub:  h,
        conn: conn,
        send: make(chan []byte, 256),
    }

    client.hub.register <- client

    go client.writePump(r.Context())
    go client.readPump(r.Context())
}

func (c *Client) readPump(ctx context.Context) {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()

    c.conn.SetReadLimit(512)
    c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })

    for {
        _, message, err := c.conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway) {
                log.Printf("error: %v", err)
            }
            return
        }
        c.hub.broadcast <- message
    }
}

func (c *Client) writePump(ctx context.Context) {
    ticker := time.NewTicker(54 * time.Second)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case <-ctx.Done():
            return
        case message, ok := <-c.send:
            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }
            c.conn.WriteMessage(websocket.TextMessage, message)
        case <-ticker.C:
            c.conn.WriteMessage(websocket.PingMessage, nil)
        }
    }
}

func (c *Client) close() {
    c.conn.WriteControl(
        websocket.CloseMessage,
        websocket.FormatCloseMessage(websocket.CloseGoingAway, "shutdown"),
        time.Now().Add(5*time.Second),
    )
    c.conn.Close()
}

func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    hub := NewHub()
    go hub.Run(ctx)

    http.HandleFunc("/ws", hub.HandleWS)

    srv := &http.Server{Addr: ":8080"}
    go func() {
        log.Println("server starting on :8080")
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    <-ctx.Done()
    log.Println("shutting down...")

    // 1. Закрыть WebSocket
    hub.Shutdown()

    // 2. Shutdown HTTP
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    srv.Shutdown(shutdownCtx)

    log.Println("done")
}
```

### Что демонстрирует

1. **Hub** — центральная горутина.
2. **Client** — readPump + writePump.
3. **Backpressure** — non-blocking send.
4. **Ping/pong** — heartbeat.
5. **Graceful shutdown** — close + shutdown.

### Аннотация сложности

| Операция | Time |
|:---|:---|
| Connect | ~1-10 мс |
| Broadcast | ~100-200 нс × N |
| Ping | ~1-5 мс |
| Shutdown | 0-5 сек |

### 💡 Практика: как строить WebSocket-чат

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Hub** — одна горутина.
2. **readPump + writePump.**
3. **Ping/pong.**
4. **Graceful shutdown.**

**👍 СТОИТ СДЕЛАТЬ:**

5. **Метрики** — количество клиентов.
6. **Логирование** подключений.

**❌ НЕ ДЕЛАЙ:**

7. **Не блокируйся** на медленном клиенте.
8. **Не забывай close frame.**

### Ключевые выводы подглавы 20.10

- **Hub** — центральная горутина.
- **readPump + writePump.**
- **Ping/pong** каждые 54 сек.
- **Graceful shutdown** — close + shutdown.

---

## 20.11 Выводы и типичные ошибки

**Что мы узнали?**

HTTP не подходит для real-time. WebSocket — bidirectional. SSE — сервер → клиент. gRPC streaming — для микросервисов. Hub-паттерн — для WebSocket. Backpressure — критичен. Heartbeat — ping/pong. Graceful shutdown — close + shutdown. Масштабирование — pub/sub.

**Типичные ошибки:**

- ❌ **Polling для real-time.** Latency.
- ❌ **Mutex для клиентов.** Hub лучше.
- ❌ **Блокировка на медленном клиенте.** Backpressure.
- ❌ **Без heartbeat.** Мёртвые соединения.
- ❌ **Без graceful shutdown.** Клиенты отключаются.
- ❌ **Без backoff reconnect.** Thundering herd.
- ❌ **Stateful без pub/sub.** Не масштабируется.
- ❌ **`golang.org/x/net/websocket`.** Устаревшая.
- ❌ **Не обрабатывать `io.EOF`.** Ошибки.
- ❌ **Игнорировать close codes.**

---

## 20.12 Для быстрого повторения

- **HTTP request/response** — инициатива у клиента.
- **Polling** — latency, пустые запросы.
- **WebSocket** — bidirectional, frame-based.
- **Handshake** через HTTP Upgrade.
- **Ping/pong** — heartbeat.
- **SSE** — сервер → клиент, `text/event-stream`.
- **gRPC streaming** — 3 типа, HTTP/2.
- **Hub** — одна горутина, map без Mutex.
- **Client** — readPump + writePump.
- **Backpressure** — non-blocking send.
- **Drop или disconnect** — стратегии.
- **Heartbeat** — 30-60 сек.
- **Reconnect** — с backoff.
- **Graceful shutdown** — close + shutdown.
- **Масштабирование** — pub/sub.
- **Redis Pub/Sub** или **NATS**.
- **Sticky sessions** или stateless.
- **`nhooyr/coder/websocket`** — рекомендуется.

---

## 20.13 Вопросы для самопроверки

1. Почему HTTP не подходит для real-time?
2. Что такое WebSocket?
3. Как работает handshake?
4. Что такое ping/pong?
5. Что такое hub-паттерн?
6. Зачем readPump + writePump?
7. Что такое SSE?
8. Чем SSE отличается от WebSocket?
9. Что такое gRPC streaming?
10. Какие 3 типа streaming?
11. Что такое backpressure в streaming?
12. Какие стратегии backpressure?
13. Что такое heartbeat?
14. Как делать reconnect?
15. Как делать graceful shutdown для streaming?
16. Как масштабировать streaming?
17. Что такое Redis Pub/Sub?
18. Почему `nhooyr/coder/websocket` рекомендуется?

---

## 20.14 Ответы

### Ответ 1

**HTTP** — инициатива у клиента. Сервер не может отправить данные без запроса. Polling даёт latency и пустые запросы.

### Ответ 2

**WebSocket** — bidirectional, frame-based протокол поверх HTTP Upgrade.

### Ответ 3

**Handshake:** клиент отправляет HTTP-запрос с `Upgrade: websocket`. Сервер отвечает `101 Switching Protocols`. Соединение становится WebSocket.

### Ответ 4

**Ping/pong** — heartbeat. Сервер отправляет ping, клиент отвечает pong. Проверяет живость.

### Ответ 5

**Hub-паттерн** — центральная горутина, которая управляет клиентами. Register, unregister, broadcast через каналы.

### Ответ 6

**readPump** — читает от клиента. **writePump** — пишет клиенту. Разделение позволяет обрабатывать параллельно.

### Ответ 7

**SSE** — Server-Sent Events. Push от сервера через HTTP.

### Ответ 8

**SSE** — односторонний (сервер → клиент). **WebSocket** — bidirectional.

### Ответ 9

**gRPC streaming** — потоки поверх HTTP/2.

### Ответ 10

**3 типа:** server streaming, client streaming, bidirectional.

### Ответ 11

**Backpressure** — медленный consumer замедляет producer.

### Ответ 12

**Стратегии:** drop oldest, drop newest, block, disconnect.

### Ответ 13

**Heartbeat** — периодические сообщения для проверки живости.

### Ответ 14

**Reconnect** — с exponential backoff.

### Ответ 15

**Graceful shutdown:** close frame для WebSocket, `r.Context()` для SSE, `GracefulStop` для gRPC.

### Ответ 16

**Масштабирование** через pub/sub (Redis, NATS).

### Ответ 17

**Redis Pub/Sub** — механизм publish/subscribe для broadcast между серверами.

### Ответ 18

**`nhooyr/coder/websocket`** — context-aware, современный, активно поддерживается.

---

## 20.15 Куда идти дальше?

Мы разобрали WebSocket, SSE, gRPC streaming. Теперь мы умеем строить real-time сервисы.

Но остаётся **следующая тема**: как писать **безопасный** конкурентный код?

- **Как писать безопасный конкурентный код?** TOCTOU, timing attacks, DoS через горутины. → **Глава 21: Безопасность конкурентного кода.**
- **Как строить распределённые системы?** CAP, consensus. → **Глава 22: Распределённые системы — введение.**
- **Как работает Raft?** → **Глава 23: Consensus — Raft и Paxos.**

---

## 20.16 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **Polling** | HTTP-запросы | Latency, пустые |
| **WebSocket** | Bidirectional | Frame-based, WS |
| **Handshake** | Upgrade | HTTP 101 |
| **Ping/pong** | Heartbeat | 30-60 сек |
| **SSE** | Server → Client | `text/event-stream` |
| **gRPC streaming** | Потоки | HTTP/2 |
| **Hub** | Центральная горутина | Map без Mutex |
| **readPump** | Чтение | От клиента |
| **writePump** | Запись | К клиенту |
| **Backpressure** | Замедление | Non-blocking send |
| **Drop/disconnect** | Стратегии | Для медленных |
| **Reconnect** | Переподключение | Backoff |
| **Graceful shutdown** | Close + shutdown | Таймаут |
| **Pub/sub** | Масштабирование | Redis, NATS |
| **Sticky sessions** | Привязка | Или stateless |
| **`nhooyr/coder`** | Библиотека | Рекомендуется |

🌐 **Ключевая идея:** HTTP request/response не подходит для real-time — нужен push от сервера. **WebSocket** — bidirectional, frame-based. **Handshake** через HTTP Upgrade. **Ping/pong** — heartbeat. **SSE** — сервер → клиент, `text/event-stream`, auto-reconnect. **gRPC streaming** — 3 типа, HTTP/2, flow control. **Hub-паттерн** — центральная горутина, map без Mutex. **Client** — readPump + writePump. **Backpressure** — non-blocking send; drop или disconnect. **Heartbeat** — 30-60 сек. **Reconnect** — с backoff. **Graceful shutdown** — close + shutdown. **Масштабирование** через pub/sub (Redis, NATS) и sticky sessions. **`nhooyr/coder/websocket`** — рекомендуется для новых проектов.