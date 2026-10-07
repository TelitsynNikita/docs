# 🔍 Глава 23: Observability — трейсинг

**Что вы узнаете:**
- Что такое distributed tracing и зачем он нужен.
- Три столпа observability: как traces дополняют logs и metrics.
- Что такое span, trace, context propagation.
- Как работает OpenTelemetry: API, SDK, Collector.
- Как инструментировать Go-приложение для трейсинга.
- Что такое sampling: head-based, tail-based, adaptive.
- Как развернуть Jaeger и Tempo.
- Как связать traces с logs и metrics через exemplars.
- Как диагностировать latency и bottleneck'и через трейсы.
- Как избежать типичных проблем: overhead, cardinality, sampling.

**После прочтения вы сможете:**
- Объяснить, зачем нужен distributed tracing.
- Инструментировать Go-приложение через OpenTelemetry.
- Развернуть Jaeger или Tempo в Kubernetes.
- Настроить sampling.
- Коррелировать traces с logs и metrics.
- Диагностировать latency через трейсы.
- Связывать трейсы с алертами.

---

## Содержание

- [23.0 Пролог: где застрял запрос?](#230-пролог-где-застрял-запрос)
- [23.1 Что такое distributed tracing](#231-что-такое-distributed-tracing)
- [23.2 Span, trace, context propagation](#232-span-trace-context-propagation)
- [23.3 OpenTelemetry: стандарт](#233-opentelemetry-стандарт)
- [23.4 Инструментирование Go-приложения](#234-инструментирование-go-приложения)
- [23.5 Автоматическая инструментация](#235-автоматическая-инструментация)
- [23.6 Context propagation через сервисы](#236-context-propagation-через-сервисы)
- [23.7 Sampling: head, tail, adaptive](#237-sampling-head-tail-adaptive)
- [23.8 Jaeger: UI для трейсов](#238-jaeger-ui-для-трейсов)
- [23.9 Grafana Tempo: масштабируемый трейсинг](#239-grafana-tempo-масштабируемый-трейсинг)
- [23.10 Корреляция traces + logs + metrics](#2310-корреляция-traces--logs--metrics)
- [23.11 Диагностика latency через трейсы](#2311-диагностика-latency-через-трейсы)
- [23.12 OpenTelemetry Collector](#2312-opentelemetry-collector)
- [23.13 Стоимость и overhead трейсинга](#2313-стоимость-и-overhead-трейсинга)
- [23.14 Диагностика проблем](#2314-диагностика-проблем)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 23.0 Пролог: где застрял запрос?

Вторник, 14:00. Пользователи жалуются, что `/api/orders` работает медленно. Иногда 5 секунд вместо 200ms.

Ты открываешь Grafana. Метрики показывают:

- **Latency p50**: 150ms (нормально).
- **Latency p95**: 3.5 секунды (плохо!).
- **Latency p99**: 8 секунд (очень плохо!).
- **RPS**: без изменений.
- **Error rate**: 0.1% (нормально).

**p95 плохой, p50 хороший.** Значит, **некоторые** запросы медленные, а большинство — нормальные. Проблема в **определённых** запросах или **определённых** условиях.

Ты идёшь в логи:

```bash
kubectl logs -n production deployment/api --tail=1000 | grep "duration" | sort
```

Находишь медленные запросы:

```
INFO request completed path=/api/orders duration_ms=5200
INFO request completed path=/api/orders duration_ms=4800
INFO request completed path=/api/orders duration_ms=3500
```

**5 секунд на запрос.** Но почему? Что внутри?

Запрос проходит через 5 сервисов:

- **API gateway** → **api-service** → **db-service** → **cache-service** → **notification-service**.

Каждый из них имеет свои логи. Но как связать?

Ты ищешь в логах каждого сервиса по `request_id`. Но `request_id` не передавался между сервисами.

**Ты не знаешь, где застрял запрос.**

**Это — проблема distributed tracing.**

В монолите: stack trace показывает, где медленно.
В микросервисах: нужно связать запросы между сервисами.

**Distributed tracing решает эту проблему.** Каждый запрос получает **trace ID**, передаётся между сервисами. Каждый сервис записывает **span** — отрезок работы. В UI видно **полный путь** запроса и **где** он застрял.

В этой главе мы разберём трейсинг. Начнём с основ.

Это — **третий столп observability**. Дополняет logs и metrics.

---

## 23.1 Что такое distributed tracing

### 🔌 Проблема: latency в микросервисах

В монолите:

- Один процесс.
- Один stack trace.
- Видно, где медленно.

В микросервисах:

- 5-10 сервисов на запрос.
- Разные языки, разные логи.
- **Не видно**, где медленно.

**Distributed tracing** решает это.

### 📊 Что такое distributed tracing

**Distributed tracing** — отслеживание запроса через все сервисы.

**Ключевые понятия:**

- **Trace** — весь путь запроса.
- **Span** — один отрезок работы (один сервис, одна операция).
- **Trace ID** — уникальный ID запроса.
- **Span ID** — уникальный ID отрезка.
- **Parent Span ID** — родительский span.

**Пример:**

```
Trace ID: abc123
├── Span: api-gateway (10ms)
│   ├── Span: auth-service (2ms)
│   └── Span: api-service (8ms)
│       ├── Span: db-query (5ms)
│       └── Span: cache-get (1ms)
└── Span: notification-service (100ms)
    └── Span: email-send (95ms)     ← bottleneck!
```

**Что видно:**

- **Полный путь** запроса.
- **Длительность** каждого span.
- **Bottleneck** — где медленно.
- **Dependencies** — кто кого вызывает.

### 🎯 Чем дополняет logs и metrics

| Аспект | Logs | Metrics | Traces |
|:---|:---|:---|:---|
| **Что** | События | Числа | Путь запроса |
| **Детали** | Много | Мало | Средне |
| **Агрегация** | Сложно | Легко | По запросу |
| **Латентность** | Нет | p50/p95/p99 | Breakdown |
| **Dependencies** | Нет | Нет | Да |
| **Стоимость** | Средняя | Низкая | Высокая |

**Пример:**

**Metrics:**

```
http_request_duration_seconds{quantile="0.99"} = 8.5s
```

**Вывод:** запрос медленный.

**Logs:**

```
INFO query="SELECT * FROM orders" duration_ms=5000
```

**Вывод:** медленный SQL.

**Traces:**

```
Trace abc123 (8.5s)
├── api-gateway (10ms)
└── api-service (8.5s)
    ├── db-query (8.0s)     ← bottleneck
    └── cache-get (0.5s)
```

**Вывод:** 8 секунд в БД. Конкретный запрос.

**Traces дают breakdown latency по сервисам.**

### 🎯 Когда использовать traces

**✅ Использовать:**

- **Микросервисы** (3+).
- **Сложные запросы** через много сервисов.
- **Latency debugging.**
- **Dependency analysis.**
- **Root cause analysis** в production.

**❌ Не использовать:**

- **Монолит** — stack trace достаточно.
- **Простые системы** (1-2 сервиса).
- **Без инструментации** — overhead.

### 🎯 Стоимость трейсинга

**Проблема:** traces — дорогие.

**Почему:**

- **Объём.** Каждый запрос = trace.
- **Хранение.** Терабайты данных.
- **Processing.** Агрегация, indexing.
- **Overhead.** В приложении.

**Решение:** **sampling.**

**Sampling** — сохранять не все traces, а выборку (1-10%).

**Разберём в подглаве 23.7.**

### 💡 Практика: что важно понять

**✅ ОБЯЗАТЕЛЬНО:**

1. **Traces дополняют logs и metrics.**
2. **Trace = путь запроса. Span = отрезок.**
3. **Sampling обязателен.**

**👍 СТОИТ:**

4. **OpenTelemetry** для instrumentation.
5. **Jaeger или Tempo** для backend.
6. **Корреляция** с logs и metrics.

**❌ НЕ ДЕЛАЙ:**

7. **Не собирай все traces** без sampling.
8. **Не используй traces для монолита.**
9. **Не забывай про context propagation.**

### Где мы сейчас

Мы разобрали, зачем нужен трейсинг. Теперь — **span, trace, propagation**.

---

## 23.2 Span, trace, context propagation

### 🔌 Проблема: как описать путь запроса

Запрос проходит через 5 сервисов. Как описать его путь?

**Ответ:** trace из spans.

### 📊 Span

**Span** — один отрезок работы.

**Что содержит:**

- **Trace ID** — ID всего запроса.
- **Span ID** — ID этого отрезка.
- **Parent Span ID** — ID родителя.
- **Name** — имя операции (`GET /api/orders`).
- **Start time** — начало.
- **End time** — конец.
- **Duration** — длительность.
- **Attributes** — key-value (метод, URL, status).
- **Events** — события внутри span.
- **Status** — OK, Error.

**Пример span:**

```json
{
  "trace_id": "abc123",
  "span_id": "span-1",
  "parent_span_id": "",
  "name": "GET /api/orders",
  "start_time": "2026-01-15T09:00:12.345Z",
  "end_time": "2026-01-15T09:00:12.350Z",
  "duration_ms": 5,
  "attributes": {
    "http.method": "GET",
    "http.url": "/api/orders",
    "http.status_code": 200
  },
  "status": "OK"
}
```

### 📊 Trace

**Trace** — набор spans, связанных по trace ID.

**Структура — дерево:**

```
Trace abc123
├── Span 1: API Gateway (root)
│   ├── Span 2: Auth Service
│   └── Span 3: API Service
│       ├── Span 4: DB Query
│       └── Span 5: Cache Get
└── Span 6: Notification Service
    └── Span 7: Email Send
```

**Root span** — первый span, без parent.

**Child spans** — вложенные.

### 📊 Context propagation

**Context propagation** — передача trace ID между сервисами.

**Как работает:**

1. **Root span** генерирует trace ID.
2. **Передаёт** его в заголовках к следующему сервису.
3. **Следующий сервис** читает trace ID и создаёт child span с тем же trace ID.

**Заголовки:**

**W3C Trace Context (стандарт):**

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  │                                │                │
             │  │                                │                └── flags (sampled)
             │  │                                └── parent-id (span ID)
             │  └── trace-id (32 hex)
             └── version
```

**B3 (Zipkin):**

```
X-B3-TraceId: 4bf92f3577b34da6a3ce929d0e0e4736
X-B3-SpanId: 00f067aa0ba902b7
X-B3-ParentSpanId: 0020000000000001
X-B3-Sampled: 1
```

**Jaeger:**

```
uber-trace-id: 4bf92f3577b34da6a3ce929d0e0e4736:00f067aa0ba902b7:0:1
```

**Рекомендация:** **W3C Trace Context** — стандарт. OpenTelemetry использует его по умолчанию.

### 🎯 Propagation через HTTP

**Пример:**

```
Service A → HTTP → Service B

# Service A создаёт span и отправляет запрос
GET /api/data HTTP/1.1
Host: service-b
traceparent: 00-abc123-span-1-01
...

# Service B читает traceparent и создаёт child span
# trace_id = abc123
# parent_span_id = span-1
# span_id = span-2
```

### 🎯 Propagation через gRPC

**gRPC metadata:**

```
traceparent: 00-abc123-span-1-01
```

### 🎯 Propagation через Kafka

**Kafka headers:**

```
traceparent: 00-abc123-span-1-01
```

**Проблема:** Kafka асинхронная. Trace ID передаётся, но время между producer и consumer может быть большим.

**Решение:** продолжать trace через Kafka.

### 🎯 Baggage

**Baggage** — дополнительный контекст, передаётся между сервисами.

**Пример:**

```
baggage: user_id=123,tenant=acme,request_id=xyz
```

**Что даёт:**

- **Контекст** для всех spans.
- **Фильтрация** traces по baggage.

**Осторожно:** baggage передаётся во всех запросах. Не клади секреты.

### 🎯 Attributes

**Attributes** — key-value на span.

**Примеры:**

- **HTTP:** `http.method`, `http.url`, `http.status_code`.
- **DB:** `db.system`, `db.statement`, `db.operation`.
- **RPC:** `rpc.system`, `rpc.service`, `rpc.method`.
- **Custom:** `user.id`, `order.id`, `tenant.id`.

**Стандартные семантические конвенции:**

OpenTelemetry определяет стандартные атрибуты. Используй их для совместимости.

### 🎯 Events

**Events** — точки во времени внутри span.

**Пример:**

```json
{
  "span_id": "span-1",
  "events": [
    {"name": "cache_miss", "time": "2026-01-15T09:00:12.346Z"},
    {"name": "db_query_start", "time": "2026-01-15T09:00:12.347Z"},
    {"name": "db_query_end", "time": "2026-01-15T09:00:12.350Z"}
  ]
}
```

**Когда использовать:** важные события внутри span (cache hit/miss, retry, ...).

### 🎯 Span status

**Status** — результат операции.

- **OK** — успешно.
- **Error** — ошибка.
- **Unset** — не установлен.

**Пример:**

```go
span.SetStatus(codes.Error, "database connection failed")
```

### 🔬 Практика: структура trace

```
Trace: 4bf92f3577b34da6a3ce929d0e0e4736

span-1 (root)     GET /api/orders        5ms
├── span-2        auth.verify            1ms
├── span-3        db.query               3ms
│   └── span-4    postgres.execute       2.5ms
└── span-5        cache.get              0.5ms

Total: 5ms
Bottleneck: db.query (60%)
```

### 💡 Практика: как правильно описывать traces

**✅ ОБЯЗАТЕЛЬНО:**

1. **Trace ID** передаётся между сервисами.
2. **Span ID** уникален.
3. **Parent Span ID** для иерархии.
4. **Attributes** с семантическими конвенциями.
5. **Status** для ошибок.

**👍 СТОИТ:**

6. **Events** для важных моментов.
7. **Baggage** для контекста.
8. **W3C Trace Context** для propagation.

**❌ НЕ ДЕЛАЙ:**

9. **Не генерируй новый trace ID** в каждом сервисе.
10. **Не забывай про propagation** через gRPC, Kafka.
11. **Не клади секреты в attributes или baggage.**

### Где мы сейчас

Мы разобрали span, trace, propagation. Теперь — **OpenTelemetry**.

---

## 23.3 OpenTelemetry: стандарт

### 🔌 Проблема: разные SDK для разных систем

Каждый tracing backend имеет свой SDK: Jaeger, Zipkin, Datadog. Меняешь backend — переписываешь код.

**OpenTelemetry решает это.**

### 📊 Что такое OpenTelemetry

**OpenTelemetry (OTel)** — стандарт для observability от CNCF.

**Что даёт:**

- **Единый API** для logs, metrics, traces.
- **SDK** для Go, Python, Java, JS, ...
- **Collector** для сбора и экспорта.
- **Независимость от vendor** — экспорт в любой backend.
- **Автоматическая инструментация.**

**Компоненты:**

- **API** — интерфейсы.
- **SDK** — реализация.
- **Collector** — сбор и экспорт.
- **Instrumentation** — библиотеки для автоматической инструментации.

### 🎯 Провайдеры и экспортёры

**OpenTelemetry** — vendor-neutral. Экспорт через **exporters**:

- **OTLP** (OpenTelemetry Protocol) — стандартный.
- **Jaeger.**
- **Zipkin.**
- **Datadog.**
- **New Relic.**
- **AWS X-Ray.**

**Рекомендация:** **OTLP** — стандарт. Backend принимает OTLP.

### 🎯 OTLP

**OTLP** — протокол OpenTelemetry.

**Транспорт:**

- **gRPC** — быстрее, по умолчанию.
- **HTTP/JSON** — проще.

**Порты:**

- **4317** — gRPC.
- **4318** — HTTP.

### 🎯 Архитектура

```
┌──────────────┐
│  Application │
│              │
│  OTel SDK    │
│  (instrument)│
└──────┬───────┘
       │
       │ OTLP
       ▼
┌──────────────┐
│  OTel        │
│  Collector   │
│              │
│  - Receive   │
│  - Process   │
│  - Export    │
└──────┬───────┘
       │
       │ OTLP / Jaeger / ...
       ▼
┌──────────────┐
│  Backend     │
│  (Jaeger,    │
│  Tempo, ...) │
└──────────────┘
```

**Collector** — промежуточный слой. Принимает, обрабатывает, экспортирует.

**Зачем:**

- **Демультиплексирование.** Разные backends.
- **Processing.** Sampling, filtering, enrichment.
- **Buffering.** Если backend недоступен.
- **Единая точка** для всех сервисов.

### 🎯 SDK в Go

```bash
go get go.opentelemetry.io/otel
go get go.opentelemetry.io/otel/sdk
go get go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc
```

**Инициализация:**

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
)

func initTracer(ctx context.Context) (*sdktrace.TracerProvider, error) {
    // Exporter
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("otel-collector:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }
    
    // Resource — метаданные сервиса
    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName("myapp"),
            semconv.ServiceVersion("1.0.0"),
            semconv.DeploymentEnvironment("production"),
        ),
    )
    if err != nil {
        return nil, err
    }
    
    // TracerProvider
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1))),
    )
    
    otel.SetTracerProvider(tp)
    return tp, nil
}
```

### 🔬 Практика: OTel

```bash
# 1. Создать приложение
mkdir otel-demo && cd otel-demo
go mod init otel-demo

# 2. Установить зависимости
go get go.opentelemetry.io/otel
go get go.opentelemetry.io/otel/sdk
go get go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc
go get go.opentelemetry.io/otel/sdk/trace
go get go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp

# 3. Пример (см. подглаву 23.4)

# 4. Запустить OTel Collector
docker run -d --name otel-collector \
  -p 4317:4317 -p 4318:4318 \
  -v $(pwd)/otel-config.yaml:/etc/otelcol/config.yaml \
  otel/opentelemetry-collector:latest

# 5. Запустить приложение
go run main.go

# 6. Сделать запрос
curl localhost:8080/api/orders

# 7. Проверить traces в Jaeger UI
```

### 💡 Практика: как правильно использовать OTel

**✅ ОБЯЗАТЕЛЬНО:**

1. **OpenTelemetry** для instrumentation.
2. **OTLP** для экспорта.
3. **Collector** как промежуточный слой.
4. **Resource** с service.name.

**👍 СТОИТ:**

5. **Автоматическая инструментация** для HTTP, gRPC, DB.
6. **Sampling** для снижения объёма.
7. **Semantic conventions** для attributes.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй vendor-specific SDK** без необходимости.
9. **Не забывай про propagation.**
10. **Не экспортируй напрямую в backend** без Collector (в production).

### Где мы сейчас

Мы разобрали OpenTelemetry. Теперь — **инструментирование Go**.

---

## 23.4 Инструментирование Go-приложения

### 🔌 Проблема: как добавить трейсинг в Go

Приложение на Go. Нужно добавить spans.

**Решение:** OpenTelemetry SDK для Go.

### 📊 Ручное инструментирование

**Базовый пример:**

```go
package main

import (
    "context"
    "log"
    "net/http"
    "time"
    
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
)

func initTracer(ctx context.Context) (*sdktrace.TracerProvider, error) {
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("localhost:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }
    
    res, _ := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName("myapp"),
            semconv.ServiceVersion("1.0.0"),
        ),
    )
    
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
    )
    
    otel.SetTracerProvider(tp)
    return tp, nil
}

var tracer = otel.Tracer("myapp")

func handleRequest(w http.ResponseWriter, r *http.Request) {
    ctx, span := tracer.Start(r.Context(), "handleRequest")
    defer span.End()
    
    span.SetAttributes(
        attribute.String("http.method", r.Method),
        attribute.String("http.url", r.URL.Path),
        attribute.String("http.user_agent", r.UserAgent()),
    )
    
    // Бизнес-логика
    userID, err := getUser(ctx, "123")
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        http.Error(w, "Error", http.StatusInternalServerError)
        return
    }
    
    span.SetAttributes(attribute.String("user.id", userID))
    w.Write([]byte("OK"))
}

func getUser(ctx context.Context, id string) (string, error) {
    ctx, span := tracer.Start(ctx, "getUser")
    defer span.End()
    
    span.SetAttributes(attribute.String("user.id", id))
    
    // Симуляция запроса к БД
    time.Sleep(50 * time.Millisecond)
    
    return "alice", nil
}

func main() {
    ctx := context.Background()
    tp, err := initTracer(ctx)
    if err != nil {
        log.Fatal(err)
    }
    defer tp.Shutdown(ctx)
    
    http.HandleFunc("/", handleRequest)
    log.Println("Starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### 🎯 Ключевые операции

**Создание span:**

```go
ctx, span := tracer.Start(ctx, "operation_name")
defer span.End()
```

**Attributes:**

```go
span.SetAttributes(
    attribute.String("key", "value"),
    attribute.Int("count", 42),
    attribute.Bool("enabled", true),
)
```

**Events:**

```go
span.AddEvent("cache_miss",
    trace.WithAttributes(attribute.String("key", "user:123")),
)
```

**Errors:**

```go
if err != nil {
    span.RecordError(err)
    span.SetStatus(codes.Error, err.Error())
}
```

**Nested spans:**

```go
func parent(ctx context.Context) {
    ctx, span := tracer.Start(ctx, "parent")
    defer span.End()
    
    child(ctx)  // ctx содержит parent span
}

func child(ctx context.Context) {
    ctx, span := tracer.Start(ctx, "child")
    defer span.End()
    // Этот span будет child of "parent"
}
```

### 🎯 Контекст

**context.Context** — передаёт trace через функции.

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()  // контекст запроса
    
    ctx, span := tracer.Start(ctx, "handler")
    defer span.End()
    
    // Передаём ctx дальше
    result, err := processOrder(ctx, orderID)
}

func processOrder(ctx context.Context, id string) (Result, error) {
    ctx, span := tracer.Start(ctx, "processOrder")
    defer span.End()
    
    // ...
}
```

**Правило:** **всегда** передавай `ctx` в функции.

### 🎯 SpanKind

**SpanKind** — тип span.

| Kind | Что означает |
|:---|:---|
| **Internal** | Внутренняя операция (по умолчанию) |
| **Server** | Обработка входящего запроса |
| **Client** | Исходящий запрос |
| **Producer** | Отправка в очередь |
| **Consumer** | Чтение из очереди |

**Пример:**

```go
// Server
ctx, span := tracer.Start(ctx, "handleRequest", trace.WithSpanKind(trace.SpanKindServer))

// Client
ctx, span := tracer.Start(ctx, "callService", trace.WithSpanKind(trace.SpanKindClient))
```

**Что даёт:** UI показывает spans правильно (server/client).

### 🔬 Практика: инструментирование

```go
// main.go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "time"
    
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
    "go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("myapp")

func initTracer(ctx context.Context) (*sdktrace.TracerProvider, error) {
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("localhost:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }
    
    res, _ := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName("myapp"),
            semconv.ServiceVersion("1.0.0"),
        ),
    )
    
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.AlwaysSample()),  // для теста
    )
    
    otel.SetTracerProvider(tp)
    return tp, nil
}

func handleOrders(w http.ResponseWriter, r *http.Request) {
    ctx, span := tracer.Start(r.Context(), "handleOrders",
        trace.WithSpanKind(trace.SpanKindServer))
    defer span.End()
    
    span.SetAttributes(
        attribute.String("http.method", r.Method),
        attribute.String("http.path", r.URL.Path),
    )
    
    // Валидация
    if err := validate(ctx); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    // Запрос к БД
    orders, err := queryDB(ctx)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // Кэш
    cache(ctx)
    
    span.SetAttributes(attribute.Int("orders.count", len(orders)))
    
    w.WriteHeader(http.StatusOK)
    fmt.Fprintf(w, "Orders: %v\n", orders)
}

func validate(ctx context.Context) error {
    _, span := tracer.Start(ctx, "validate")
    defer span.End()
    
    time.Sleep(5 * time.Millisecond)
    return nil
}

func queryDB(ctx context.Context) ([]string, error) {
    ctx, span := tracer.Start(ctx, "queryDB",
        trace.WithSpanKind(trace.SpanKindClient))
    defer span.End()
    
    span.SetAttributes(
        attribute.String("db.system", "postgresql"),
        attribute.String("db.statement", "SELECT * FROM orders"),
    )
    
    time.Sleep(100 * time.Millisecond)  // симуляция медленного запроса
    
    return []string{"order-1", "order-2"}, nil
}

func cache(ctx context.Context) {
    ctx, span := tracer.Start(ctx, "cache.get")
    defer span.End()
    
    span.SetAttributes(attribute.String("cache.key", "orders:123"))
    span.AddEvent("cache_miss")
    
    time.Sleep(10 * time.Millisecond)
}

func main() {
    ctx := context.Background()
    tp, err := initTracer(ctx)
    if err != nil {
        log.Fatal(err)
    }
    defer tp.Shutdown(ctx)
    
    http.HandleFunc("/api/orders", handleOrders)
    log.Println("Starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Запуск:**

```bash
# 1. OTel Collector + Jaeger
docker run -d --name jaeger \
  -p 16686:16686 -p 4317:4317 -p 4318:4318 \
  jaegertracing/all-in-one:latest

# 2. Приложение
go mod tidy
go run main.go &

# 3. Запросы
curl localhost:8080/api/orders

# 4. Jaeger UI
# http://localhost:16686
# Search → service: myapp
```

### 💡 Практика: как правильно инструментировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Span для каждой значимой операции.**
2. **Attributes с семантическими конвенциями.**
3. **RecordError и SetStatus при ошибках.**
4. **Передавать `ctx` везде.**

**👍 СТОИТ:**

5. **SpanKind** для server/client.
6. **Events** для важных моментов.
7. **Sampling** для production.

**❌ НЕ ДЕЛАЙ:**

8. **Не создавай spans для мелочей.** Overhead.
9. **Не забывай про `defer span.End()`.**
10. **Не клади секреты в attributes.**

### Где мы сейчас

Мы разобрали ручное инструментирование. Теперь — **автоматическая инструментация**.

---

## 23.5 Автоматическая инструментация

### 🔌 Проблема: ручная инструментация трудоёмкая

Каждый HTTP-запрос, каждый DB-запрос, каждый gRPC-вызов — нужно инструментировать вручную.

**Решение:** автоматическая инструментация.

### 📊 Что такое auto-instrumentation

**Auto-instrumentation** — библиотеки, которые автоматически создают spans для стандартных операций.

**Покрытие:**

- **HTTP servers** (net/http, gin, echo, fiber).
- **HTTP clients** (net/http, resty).
- **gRPC** (grpc-go).
- **DB** (database/sql, pgx, gorm, mongo).
- **Redis.**
- **Kafka.**
- **AWS SDK.**
- И многое другое.

### 🎯 HTTP server

**`otelhttp`:**

```go
import (
    "net/http"
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

func main() {
    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Твой handler
        // Span уже создан otelhttp
        w.Write([]byte("OK"))
    })
    
    wrapped := otelhttp.NewHandler(handler, "my-handler")
    http.Handle("/", wrapped)
    http.ListenAndServe(":8080", nil)
}
```

**Что создаётся автоматически:**

- **Span** для каждого запроса.
- **Attributes:** `http.method`, `http.url`, `http.status_code`, `http.user_agent`.
- **Context propagation** из заголовков (`traceparent`).
- **SpanKind: Server.**

### 🎯 HTTP client

```go
import (
    "net/http"
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

func main() {
    client := http.Client{
        Transport: otelhttp.NewTransport(http.DefaultTransport),
    }
    
    req, _ := http.NewRequestWithContext(ctx, "GET", "http://other-service/api", nil)
    resp, err := client.Do(req)
    // Span создаётся автоматически
}
```

**Что создаётся:**

- **Span** для каждого запроса.
- **Context propagation** через заголовки.
- **SpanKind: Client.**

### 🎯 Gin

```go
import (
    "github.com/gin-gonic/gin"
    "go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin"
)

func main() {
    r := gin.Default()
    r.Use(otelgin.Middleware("my-service"))
    
    r.GET("/api/orders", func(c *gin.Context) {
        // Span уже создан
        c.JSON(200, gin.H{"status": "ok"})
    })
    
    r.Run(":8080")
}
```

### 🎯 gRPC server

```go
import (
    "google.golang.org/grpc"
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

func main() {
    server := grpc.NewServer(
        grpc.UnaryInterceptor(otelgrpc.UnaryServerInterceptor()),
        grpc.StreamInterceptor(otelgrpc.StreamServerInterceptor()),
    )
    // ...
}
```

### 🎯 gRPC client

```go
conn, err := grpc.Dial(
    "other-service:50051",
    grpc.WithUnaryInterceptor(otelgrpc.UnaryClientInterceptor()),
    grpc.WithStreamInterceptor(otelgrpc.StreamClientInterceptor()),
)
```

### 🎯 database/sql

```go
import (
    "database/sql"
    "go.opentelemetry.io/contrib/instrumentation/database/sql/otelsql"
    semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
)

func main() {
    db, err := otelsql.Open("postgres", dsn,
        otelsql.WithAttributes(semconv.DBSystemPostgreSQL),
    )
    
    // Использование как обычно
    rows, err := db.QueryContext(ctx, "SELECT * FROM orders")
    // Span создаётся автоматически
}
```

**Что создаётся:**

- **Span** для каждого запроса.
- **Attributes:** `db.system`, `db.statement`, `db.operation`.

### 🎯 pgx

```go
import (
    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/exaring/otelpgx"
)

func main() {
    cfg, _ := pgxpool.ParseConfig(dsn)
    cfg.ConnConfig.Tracer = otelpgx.NewTracer()
    
    pool, _ := pgxpool.NewWithConfig(ctx, cfg)
    // Автоматический трейсинг
}
```

### 🎯 Redis

```go
import (
    "github.com/redis/go-redis/v9"
    "github.com/redis/go-redis/extra/redisotel/v9"
)

func main() {
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    redisotel.InstrumentTracing(rdb)
    
    rdb.Get(ctx, "key")
    // Span создаётся автоматически
}
```

### 🎯 Kafka

```go
import (
    "github.com/IBM/sarama"
    "go.opentelemetry.io/contrib/instrumentation/github.com/IBM/sarama/otelsarama"
)

func main() {
    // Producer
    producer, _ := sarama.NewSyncProducer(brokers, config)
    producer = otelsarama.WrapSyncProducer(config, producer)
    
    // Consumer
    handler := otelsarama.WrapConsumerGroupHandler(&myHandler{})
    // ...
}
```

### 🎯 Комбинирование

**Real-world приложение:**

```go
func main() {
    // Инициализация OTel
    tp, _ := initTracer(ctx)
    defer tp.Shutdown(ctx)
    
    // DB с трейсингом
    db, _ := otelsql.Open("postgres", dsn)
    
    // Redis с трейсингом
    rdb := redis.NewClient(...)
    redisotel.InstrumentTracing(rdb)
    
    // HTTP client с трейсингом
    httpClient := &http.Client{
        Transport: otelhttp.NewTransport(http.DefaultTransport),
    }
    
    // HTTP server с трейсингом
    handler := otelhttp.NewHandler(http.HandlerFunc(handle), "api")
    http.ListenAndServe(":8080", handler)
}
```

**Что даёт:** traces для всех операций без ручной инструментации.

### 🔬 Практика: auto-instrumentation

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "net/http"
    "time"
    
    "github.com/gin-gonic/gin"
    "go.opentelemetry.io/contrib/instrumentation/database/sql/otelsql"
    "go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin"
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
    _ "github.com/lib/pq"
)

func initTracer(ctx context.Context) (*sdktrace.TracerProvider, error) {
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("localhost:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }
    
    res, _ := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName("myapp"),
        ),
    )
    
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
    )
    
    otel.SetTracerProvider(tp)
    return tp, nil
}

func main() {
    ctx := context.Background()
    tp, _ := initTracer(ctx)
    defer tp.Shutdown(ctx)
    
    // DB с автоматическим трейсингом
    db, _ := otelsql.Open("postgres", "postgres://localhost/mydb?sslmode=disable",
        otelsql.WithAttributes(semconv.DBSystemPostgreSQL),
    )
    defer db.Close()
    
    // Gin с автоматическим трейсингом
    r := gin.Default()
    r.Use(otelgin.Middleware("my-service"))
    
    // HTTP client с трейсингом
    httpClient := &http.Client{
        Transport: otelhttp.NewTransport(http.DefaultTransport),
    }
    
    r.GET("/api/orders", func(c *gin.Context) {
        ctx := c.Request.Context()
        
        // DB query — span создаётся автоматически
        rows, err := db.QueryContext(ctx, "SELECT id FROM orders LIMIT 10")
        if err != nil {
            c.JSON(500, gin.H{"error": err.Error()})
            return
        }
        defer rows.Close()
        
        var ids []string
        for rows.Next() {
            var id string
            rows.Scan(&id)
            ids = append(ids, id)
        }
        
        // HTTP вызов — span создаётся автоматически
        req, _ := http.NewRequestWithContext(ctx, "GET", "http://other-service/api", nil)
        resp, _ := httpClient.Do(req)
        defer resp.Body.Close()
        
        c.JSON(200, gin.H{"orders": ids})
    })
    
    r.Run(":8080")
}
```

### 💡 Практика: как правильно использовать auto-instrumentation

**✅ ОБЯЗАТЕЛЬНО:**

1. **Автоматическая инструментация** для HTTP, DB, gRPC, Redis.
2. **Ручные spans** для бизнес-логики.
3. **Комбинировать** auto + manual.

**👍 СТОИТ:**

4. **Semantic conventions** для attributes.
5. **Sampling** для production.
6. **Тестировать** перед деплоем.

**❌ НЕ ДЕЛАЙ:**

7. **Не инструментируй всё вручную.** Auto.
8. **Не забывай про context propagation.**
9. **Не игнорируй overhead.**

### Где мы сейчас

Мы разобрали auto-instrumentation. Теперь — **context propagation через сервисы**.

---

## 23.6 Context propagation через сервисы

### 🔌 Проблема: как передать trace между сервисами

Service A вызывает Service B. Как передать trace ID?

**Решение:** context propagation через заголовки.

### 📊 HTTP propagation

**Автоматически через `otelhttp`:**

```go
// Service A
client := http.Client{
    Transport: otelhttp.NewTransport(http.DefaultTransport),
}

req, _ := http.NewRequestWithContext(ctx, "GET", "http://service-b/api", nil)
resp, _ := client.Do(req)
// traceparent автоматически добавляется

// Service B
handler := otelhttp.NewHandler(http.HandlerFunc(...), "api")
// traceparent автоматически читается
```

**Что происходит:**

1. Service A создаёт span (client).
2. `otelhttp` добавляет `traceparent` в заголовки.
3. Service B читает `traceparent`, создаёт child span.
4. Trace продолжается.

### 🎯 gRPC propagation

**Автоматически через `otelgrpc`:**

```go
// Client
conn, _ := grpc.Dial("service-b:50051",
    grpc.WithUnaryInterceptor(otelgrpc.UnaryClientInterceptor()),
)

// Server
server := grpc.NewServer(
    grpc.UnaryInterceptor(otelgrpc.UnaryServerInterceptor()),
)
```

### 🎯 Kafka propagation

**Producer:**

```go
producer = otelsarama.WrapSyncProducer(config, producer)

msg := &sarama.ProducerMessage{
    Topic: "orders",
    Value: sarama.StringEncoder("data"),
}
producer.SendMessage(msg)
// traceparent добавляется в headers
```

**Consumer:**

```go
handler := otelsarama.WrapConsumerGroupHandler(&myHandler{})

// В handler
func (h *myHandler) ConsumeClaim(session sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
    for msg := range claim.Messages() {
        ctx := otelsarama.ExtractTraceContext(msg)  // извлечь trace
        ctx, span := tracer.Start(ctx, "processMessage")
        // ...
        span.End()
    }
}
```

### 🎯 Worker pools

**Проблема:** задача отправляется в worker pool. Context теряется.

**Решение:** передавать context в задачу.

```go
type Task struct {
    ctx context.Context
    data string
}

func worker(tasks <-chan Task) {
    for task := range tasks {
        ctx, span := tracer.Start(task.ctx, "processTask")
        // ...
        span.End()
    }
}

func submit(ctx context.Context, data string) {
    tasks <- Task{ctx: ctx, data: data}
}
```

### 🎯 Async operations

**Проблема:** goroutine теряет context.

**Решение:** передавать context явно.

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    ctx, span := tracer.Start(ctx, "handler")
    defer span.End()
    
    go asyncOperation(ctx)  // передаём ctx
    // ...
}

func asyncOperation(ctx context.Context) {
    ctx, span := tracer.Start(ctx, "asyncOperation")
    defer span.End()
    // ...
}
```

**Важно:** не используй `context.Background()` в handler'ах. Только `r.Context()`.

### 🎯 Messaging (NATS, RabbitMQ)

**NATS:**

```go
import "github.com/nats-io/nats.go"
import "go.opentelemetry.io/contrib/instrumentation/github.com/nats-io/nats.go/otelnats"

// Publish
otelnats.Publish(ctx, nc, "subject", data)

// Subscribe
otelnats.Subscribe(nc, "subject", func(msg *nats.Msg) {
    ctx := otelnats.ExtractTraceContext(msg)
    ctx, span := tracer.Start(ctx, "processMessage")
    defer span.End()
    // ...
})
```

### 🎯 Проверка propagation

**Что проверить:**

1. **Service A** — span с `trace_id`.
2. **Service B** — span с **тем же** `trace_id`.
3. **Jaeger UI** — полный trace со всеми сервисами.

**Если trace разрывается:**

- Проверь, что `otelhttp` используется в клиенте.
- Проверь, что заголовки не удаляются (nginx, proxy).
- Проверь, что `ctx` передаётся.

### 🔬 Практика: propagation

**Service A:**

```go
package main

import (
    "context"
    "io"
    "log"
    "net/http"
    
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
)

var tracer = otel.Tracer("service-a")

func main() {
    ctx := context.Background()
    exporter, _ := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("localhost:4317"),
        otlptracegrpc.WithInsecure(),
    )
    res, _ := resource.New(ctx,
        resource.WithAttributes(semconv.ServiceName("service-a")),
    )
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
    )
    otel.SetTracerProvider(tp)
    
    client := &http.Client{
        Transport: otelhttp.NewTransport(http.DefaultTransport),
    }
    
    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        ctx, span := tracer.Start(ctx, "handleInA")
        defer span.End()
        
        // Вызов Service B
        req, _ := http.NewRequestWithContext(ctx, "GET", "http://localhost:8081/api", nil)
        resp, err := client.Do(req)
        if err != nil {
            http.Error(w, err.Error(), 500)
            return
        }
        defer resp.Body.Close()
        
        body, _ := io.ReadAll(resp.Body)
        w.Write(body)
    })
    
    wrapped := otelhttp.NewHandler(handler, "api-a")
    http.Handle("/api", wrapped)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Service B:**

```go
// Аналогично, но ServiceName("service-b") и порт 8081
handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    ctx, span := tracer.Start(ctx, "handleInB")
    defer span.End()
    
    time.Sleep(50 * time.Millisecond)
    w.Write([]byte("response from B"))
})
wrapped := otelhttp.NewHandler(handler, "api-b")
```

**Проверка:**

```bash
# Jaeger UI
# Search → service: service-a
# Trace должна содержать:
# - service-a: handleInA
# - service-a: HTTP GET (client)
# - service-b: api-b
# - service-b: handleInB
```

### 💡 Практика: как правильно propagat'ить context

**✅ ОБЯЗАТЕЛЬНО:**

1. **Использовать auto-instrumentation** для HTTP, gRPC, Kafka.
2. **Передавать `ctx`** во все функции.
3. **Не использовать `context.Background()`** в handlers.

**👍 СТОИТ:**

4. **Проверять propagation** в Jaeger.
5. **Baggage** для дополнительного контекста.
6. **W3C Trace Context** (стандарт).

**❌ НЕ ДЕЛАЙ:**

7. **Не теряй context** в goroutines.
8. **Не удаляй заголовки** в proxy.
9. **Не используй vendor-specific headers** без необходимости.

### Где мы сейчас

Мы разобрали propagation. Теперь — **sampling**.

---

## 23.7 Sampling: head, tail, adaptive

### 🔌 Проблема: слишком много traces

1000 RPS × 10 spans = 10 000 spans/sec. Терабайты в день.

**Решение:** sampling.

### 📊 Что такое sampling

**Sampling** — сохранять не все traces, а выборку.

**Цели:**

- **Снизить объём** данных.
- **Снизить стоимость** хранения.
- **Снизить overhead** в приложении.

**Осторожно:** сэмплирование может **потерять** важные traces (ошибки).

### 🎯 Head-based sampling

**Head-based** — решение о сэмплировании **в начале** trace.

**Как работает:**

1. При создании root span — решение: sample или нет.
2. Решение **распространяется** на все child spans.
3. Если sample — весь trace сохраняется.

**Пример:**

```go
tp := sdktrace.NewTracerProvider(
    sdktrace.WithSampler(sdktrace.TraceIDRatioBased(0.1)),  // 10%
)
```

**Плюсы:**

- **Простой.**
- **Низкий overhead.**
- **Consistent** — весь trace или ничего.

**Минусы:**

- **Теряет редкие события.** Ошибки могут не попасть в 10%.
- **Нет контекста.** Решение до выполнения.

**Правило:** head-based с вероятностью. Не сэмплировать ошибки — использовать tail-based.

### 🎯 Tail-based sampling

**Tail-based** — решение **после** завершения trace.

**Как работает:**

1. Все spans собираются.
2. После завершения trace — решение: сохранить или нет.
3. Критерии: ошибки, длительность, атрибуты.

**Пример (OTel Collector):**

```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors-policy
        type: status_code
        status_code:
          status_codes: [ERROR]
      - name: slow-traces-policy
        type: latency
        latency:
          threshold_ms: 1000
      - name: random-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 10
```

**Что даёт:**

- **Все ошибки** сохраняются.
- **Все медленные** traces.
- **10%** остальных.

**Плюсы:**

- **Лучше качество.** Ошибки сохраняются.
- **Гибкость.** Разные критерии.

**Минусы:**

- **Дороже.** Все spans обрабатываются.
- **Задержка.** Нужно ждать завершения trace.
- **Память.** Буферизация.

**Рекомендация:** **tail-based** для production.

### 🎯 Adaptive sampling

**Adaptive** — динамически менять rate.

**Как работает:**

- **Больше сэмплов** при проблемах (высокий error rate).
- **Меньше сэмплов** при нормальной работе.

**Пример:** Jaeger adaptive sampling.

**Плюсы:**

- **Автоматически.**
- **Баланс** между объёмом и качеством.

**Минусы:**

- **Сложнее.**
- **Менее предсказуемо.**

### 🎯 Priority sampling

**Priority sampling** — приоритеты для разных traces.

**Приоритеты:**

- **Critical** — платежи, авторизация. Always sample.
- **High** — основные API. 50%.
- **Normal** — 10%.
- **Low** — health checks. 1%.

**Реализация:**

```go
func getSampler(path string) sdktrace.Sampler {
    switch {
    case strings.HasPrefix(path, "/api/payments"):
        return sdktrace.AlwaysSample()
    case strings.HasPrefix(path, "/api/orders"):
        return sdktrace.TraceIDRatioBased(0.5)
    case strings.HasPrefix(path, "/health"):
        return sdktrace.TraceIDRatioBased(0.01)
    default:
        return sdktrace.TraceIDRatioBased(0.1)
    }
}
```

### 🎯 Sampling в OTel Collector

**Head-based в SDK:**

```go
sdktrace.WithSampler(sdktrace.TraceIDRatioBased(0.1))
```

**Tail-based в Collector:**

```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    expected_new_traces_per_sec: 1000
    policies:
      - name: errors
        type: status_code
        status_code:
          status_codes: [ERROR]
      - name: slow
        type: latency
        latency:
          threshold_ms: 500
      - name: random
        type: probabilistic
        probabilistic:
          sampling_percentage: 5
```

### 🎯 Best practices

**1. Не сэмплируй всё.**

10% — разумный старт.

**2. Сэмплируй ошибки всегда.**

Tail-based или priority.

**3. Сэмплируй критичные endpoints.**

Платежи — 100%.

**4. Сэмплируй health checks.**

1% или 0%.

**5. Consistency.**

Head-based — consistent. Tail-based — не всегда.

### 🔬 Практика: sampling

**Head-based:**

```go
tp := sdktrace.NewTracerProvider(
    sdktrace.WithSampler(sdktrace.ParentBased(
        sdktrace.TraceIDRatioBased(0.1),  // root — 10%
    )),
)
```

**Tail-based в Collector:**

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    expected_new_traces_per_sec: 1000
    policies:
      - name: errors-policy
        type: status_code
        status_code:
          status_codes: [ERROR]
      - name: slow-traces-policy
        type: latency
        latency:
          threshold_ms: 1000
      - name: random-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [tail_sampling]
      exporters: [otlp/jaeger]
```

**Что даёт:**

- **Все ошибки** сохраняются.
- **Все медленные** (> 1s).
- **10%** остальных.

### 💡 Практика: как правильно сэмплировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Head-based** для базовой экономии.
2. **Tail-based** для production.
3. **Ошибки всегда** сэмплировать.
4. **Критичные endpoints** — 100%.

**👍 СТОИТ:**

5. **Priority sampling** для разных endpoints.
6. **Adaptive** для автоматизации.
7. **Мониторить** объём traces.

**❌ НЕ ДЕЛАЙ:**

8. **Не сэмплируй всё подряд.** Overhead.
9. **Не сэмплируй ошибки.** Tail-based.
10. **Не забывай про consistency.**

### Где мы сейчас

Мы разобрали sampling. Теперь — **Jaeger**.

---

## 23.8 Jaeger: UI для трейсов

### 🔌 Проблема: как визуализировать трейсы

Traces собираются. Нужен UI для анализа.

**Jaeger** — самый популярный.

### 📊 Что такое Jaeger

**Jaeger** — distributed tracing system от Uber (теперь CNCF).

**Что даёт:**

- **UI** для поиска и анализа traces.
- **Хранение** traces (Elasticsearch, Cassandra, Badger).
- **Jaeger Query** — API.
- **Jaeger Collector** — приём traces.
- **Jaeger Agent** — локальный агент (deprecated).

### 🎯 Архитектура Jaeger

```
┌──────────────┐
│  Application │
│  (OTel SDK)  │
└──────┬───────┘
       │
       │ OTLP
       ▼
┌──────────────┐
│  Jaeger      │
│  Collector   │
└──────┬───────┘
       │
       │ Storage
       ▼
┌──────────────┐
│  Storage     │
│  (ES, Cass.) │
└──────┬───────┘
       │
       │ Query
       ▼
┌──────────────┐
│  Jaeger UI   │
└──────────────┘
```

### 🎯 Установка Jaeger

**All-in-one (для dev):**

```bash
docker run -d --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \
  -p 4317:4317 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest
```

**Что даёт:**

- **Collector** на 4317 (gRPC), 4318 (HTTP).
- **UI** на 16686.
- **Storage:** Badger (локально).

**Kubernetes (Helm):**

```bash
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

helm install jaeger jaegertracing/jaeger \
  --namespace tracing \
  --create-namespace \
  --set provisionDataStore.cassandra=false \
  --set allInOne.enabled=true \
  --set storage.type=memory
```

**Production (с Elasticsearch):**

```bash
helm install jaeger jaegertracing/jaeger \
  --namespace tracing \
  --create-namespace \
  --set provisionDataStore.cassandra=false \
  --set storage.type=elasticsearch \
  --set storage.elasticsearch.host=elasticsearch \
  --set collector.replicaCount=3 \
  --set query.replicaCount=2
```

### 🎯 Jaeger UI

**Что можно делать:**

**1. Search by service:**

```
Service: myapp
Operation: GET /api/orders
Tags: http.status_code=500
Lookback: Last 1 hour
```

**2. Search by trace ID:**

```
Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736
```

**3. Search by duration:**

```
Min Duration: 1s
Max Duration: 10s
```

**4. Search by tags:**

```
http.status_code=500
user.id=123
```

### 🎯 Trace view

**Что показывается:**

- **Timeline** — визуализация spans.
- **Span details** — attributes, events, logs.
- **Service breakdown** — время по сервисам.
- **Critical path** — самый длинный путь.

**Пример:**

```
Trace: 4bf92f3577b34da6a3ce929d0e0e4736 (8.5s)

api-gateway        |███░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░| 10ms
  auth-service     |█░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░| 2ms
  api-service      |  ████████████████████████████████████ | 8.5s
    db-query       |  ████████████████████████████████░░░░ | 8.0s  ← bottleneck
    cache-get      |                                  █░░░ | 0.5s
notification-svc   |                                      █| 100ms
  email-send       |                                      █| 95ms
```

### 🎯 Jaeger Query API

```bash
# Поиск traces
curl "http://localhost:16686/api/traces?service=myapp&limit=20"

# Trace по ID
curl "http://localhost:16686/api/traces/4bf92f3577b34da6a3ce929d0e0e4736"

# Сервисы
curl "http://localhost:16686/api/services"

# Операции
curl "http://localhost:16686/api/services/myapp/operations"
```

### 🎯 Storage

**Опции:**

- **Badger** — встроенное, для dev.
- **Elasticsearch** — для production.
- **Cassandra** — для больших объёмов.
- **Kafka** — промежуточный буфер.

**Рекомендация:** **Elasticsearch** для production.

### 🎯 Retention

**Настройка retention:**

- **Elasticsearch ILM** — управление жизненным циклом.
- **Jaeger** не удаляет автоматически.

**Пример ILM:**

```json
PUT _ilm/policy/jaeger-traces
{
  "policy": {
    "phases": {
      "hot": {"min_age": "0ms", "actions": {"rollover": {"max_age": "1d"}}},
      "delete": {"min_age": "7d", "actions": {"delete": {}}}
    }
  }
}
```

### 🔬 Практика: Jaeger

```bash
# 1. Jaeger all-in-one
docker run -d --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \
  -p 4317:4317 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest

# 2. Приложение с OTel
go run main.go

# 3. Запросы
for i in {1..100}; do curl -s localhost:8080/api/orders > /dev/null; done

# 4. Jaeger UI
# http://localhost:16686
# Service: myapp
# Find Traces

# 5. Открыть trace
# Видно spans, durations, attributes

# 6. Найти медленный span
# Sort by duration, посмотреть bottleneck

# 7. Поиск по тегам
# Tags: http.status_code=500
```

### 💡 Практика: как правильно использовать Jaeger

**✅ ОБЯЗАТЕЛЬНО:**

1. **Elasticsearch** для production.
2. **Retention** настроить.
3. **Sampling** для объёма.
4. **Jaeger UI** для диагностики.

**👍 СТОИТ:**

5. **Kafka** для буферизации.
6. **Jaeger Query API** для автоматизации.
7. **Integration** с Grafana.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй Badger в production.**
9. **Не храни traces вечно.**
10. **Не игнорируй cardinality.**

### Где мы сейчас

Мы разобрали Jaeger. Теперь — **Grafana Tempo**.

---

## 23.9 Grafana Tempo: масштабируемый трейсинг

### 🔌 Проблема: Jaeger требует storage

Jaeger с Elasticsearch — дорого и сложно. Нужен простой и дешёвый backend.

**Tempo** — альтернатива от Grafana Labs.

### 📊 Что такое Tempo

**Grafana Tempo** — масштабируемая система трейсинга.

**Философия:** «Loki для трейсов».

**Отличия от Jaeger:**

| Аспект | Jaeger | Tempo |
|:---|:---|:---|
| **Storage** | ES, Cassandra | S3, GCS, filesystem |
| **Indexing** | Полное | Только trace ID |
| **Стоимость** | Высокая | Низкая |
| **Масштаб** | Средний | Очень большой |
| **UI** | Свой | Grafana |
| **Сложность** | Средняя | Низкая |

**Ключевая идея:** индексируется только **trace ID**. Поиск по атрибутам — через exemplars или Loki.

### 🎯 Архитектура Tempo

```
┌──────────────┐
│  Application │
│  (OTel SDK)  │
└──────┬───────┘
       │
       │ OTLP
       ▼
┌──────────────┐
│  Tempo       │
│  Distributor │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Tempo       │
│  Ingester    │
└──────┬───────┘
       │
       │ S3/GCS
       ▼
┌──────────────┐
│  Object      │
│  Storage     │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Grafana     │
└──────────────┘```

### 🎯 Установка Tempo

**Через Helm:**

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install tempo grafana/tempo \
  --namespace tracing \
  --create-namespace
```

**С S3:**

```bash
helm install tempo grafana/tempo \
  --namespace tracing \
  --create-namespace \
  --set tempo.storage.trace.backend=s3 \
  --set tempo.storage.trace.s3.bucket=my-traces \
  --set tempo.storage.trace.s3.region=us-west-2
```

### 🎯 Tempo vs Jaeger

**Когда Tempo:**

- **Много traces.**
- **Дешёвое хранение.**
- **Grafana как UI.**
- **Интеграция с Loki и Prometheus.**

**Когда Jaeger:**

- **Нужен собственный UI.**
- **Сложный поиск по атрибутам.**
- **Существующая инфраструктура Jaeger.**

### 🎯 Tempo в Grafana

**Data source:**

```
Name: Tempo
URL: http://tempo.tracing.svc.cluster.local:3200
```

**Что можно делать:**

- **Search by trace ID.**
- **Search by service.**
- **Correlation с Loki** (derived fields).
- **Correlation с Prometheus** (exemplars).

### 🎯 TraceQL

**TraceQL** — язык запросов Tempo.

**Примеры:**

```
# По имени сервиса
{ .service.name = "myapp" }

# По атрибуту
{ .http.status_code = 500 }

# По длительности
{ duration > 1s }

# Комбинация
{ .service.name = "myapp" && duration > 1s && .http.status_code = 500 }
```

**Что даёт:** поиск traces по атрибутам (без полного индекса).

### 🎯 Derived fields

**В Loki datasource:**

```yaml
jsonData:
  derivedFields:
    - datasourceUid: tempo
      matcherRegex: "trace_id=(\\w+)"
      name: TraceID
      url: "$${__value.raw}"
```

**Что даёт:** в логах — ссылка на trace в Tempo.

### 🔬 Практика: Tempo

```bash
# 1. Установить Tempo
helm repo add grafana https://grafana.github.io/helm-charts
helm install tempo grafana/tempo \
  --namespace tracing \
  --create-namespace

# 2. Проверить
kubectl get pods -n tracing

# 3. Port-forward
kubectl port-forward -n tracing svc/tempo 3200:3200

# 4. Настроить OTel Collector на Tempo
# exporters:
#   otlp:
#     endpoint: tempo.tracing.svc.cluster.local:4317

# 5. Grafana datasource
# Configuration → Data Sources → Add → Tempo
# URL: http://tempo.tracing.svc.cluster.local:3200

# 6. Explore → Tempo
# { .service.name = "myapp" }
```

### 💡 Практика: как правильно использовать Tempo

**✅ ОБЯЗАТЕЛЬНО:**

1. **S3** для storage.
2. **Grafana** как UI.
3. **Correlation** с Loki и Prometheus.
4. **Sampling** (tail-based в Collector).

**👍 СТОИТ:**

5. **TraceQL** для поиска.
6. **Retention** настроить.
7. **OTel Collector** перед Tempo.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй Tempo для сложных запросов.** TraceQL ограничен.
9. **Не забывай про cardinality.**
10. **Не храни traces вечно.**

### Где мы сейчас

Мы разобрали Tempo. Теперь — **корреляция traces + logs + metrics**.

---

## 23.10 Корреляция traces + logs + metrics

### 🔌 Проблема: три столпа раздельно

Logs в Loki, metrics в Prometheus, traces в Tempo. Как связать?

**Решение:** корреляция.

### 📊 Три уровня корреляции

**1. Logs → Traces.**

Из логов — переход к трейсу.

**Как:**

- **Trace ID** в логах.
- **Derived fields** в Grafana.

**Пример лога:**

```json
{
  "level": "info",
  "msg": "Request completed",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "duration_ms": 45
}
```

**В Grafana:** `trace_id` становится ссылкой на трейс.

**2. Metrics → Traces.**

Из метрик — переход к трейсам.

**Как:**

- **Exemplars** — примеры traces для метрик.

**Пример:**

```
http_request_duration_seconds_bucket{le="1"} 1234 # {trace_id="abc123"} 0.95
```

**Exemplar** — sample с trace ID.

**3. Traces → Logs.**

Из трейса — переход к логам.

**Как:**

- **Span attributes** с логами.
- **Correlation** в Grafana.

### 🎯 Trace ID в логах

**Как добавить:**

```go
import (
    "log/slog"
    "go.opentelemetry.io/otel/trace"
)

func logWithTrace(ctx context.Context, logger *slog.Logger, msg string, args ...any) {
    span := trace.SpanFromContext(ctx)
    if span.SpanContext().IsValid() {
        args = append(args,
            "trace_id", span.SpanContext().TraceID().String(),
            "span_id", span.SpanContext().SpanID().String(),
        )
    }
    logger.Info(msg, args...)
}
```

**Или через middleware:**

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        span := trace.SpanFromContext(ctx)
        
        logger := slog.With(
            "trace_id", span.SpanContext().TraceID().String(),
            "span_id", span.SpanContext().SpanID().String(),
        )
        
        ctx = context.WithValue(ctx, "logger", logger)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### 🎯 Exemplars

**Exemplars** — примеры traces для метрик.

**Prometheus с exemplars:**

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "go.opentelemetry.io/otel/trace"
)

var httpDuration = promauto.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Help:    "HTTP request duration",
        Buckets: prometheus.DefBuckets,
    },
    []string{"method", "path"},
)

func observeWithExemplar(ctx context.Context, duration float64, method, path string) {
    span := trace.SpanFromContext(ctx)
    
    observer := httpDuration.WithLabelValues(method, path)
    
    // Exemplar с trace_id
    if eo, ok := observer.(prometheus.ExemplarObserver); ok {
        eo.ObserveWithExemplar(duration, prometheus.Labels{
            "trace_id": span.SpanContext().TraceID().String(),
        })
    }
}
```

**В Grafana:**

- Панель с exemplars.
- Клик на точку → открывает trace в Tempo.

### 🎯 Logs в Jaeger/Tempo

**Span events:**

```go
span.AddEvent("user_login",
    trace.WithAttributes(
        attribute.String("user.id", "123"),
        attribute.String("ip", "10.0.1.5"),
    ),
)
```

**Span logs:**

```go
span.AddEvent("error",
    trace.WithAttributes(
        attribute.String("error.message", err.Error()),
    ),
)
```

### 🎯 Correlation в Grafana

**Derived fields в Loki:**

```yaml
jsonData:
  derivedFields:
    - datasourceUid: tempo
      matcherRegex: "trace_id=(\\w+)"
      name: TraceID
      url: "$${__value.raw}"
```

**Exemplars в Prometheus:**

```yaml
jsonData:
  exemplarTraceIdDestinations:
    - name: trace_id
      datasourceUid: tempo
```

**Что даёт:**

- **Из логов** — переход к trace.
- **Из метрик** — переход к trace.
- **Из trace** — переход к логам.

### 🎯 Полный workflow

**Сценарий: алерт по error rate.**

**1. Metrics:**

```
Error rate > 5% (Prometheus)
```

**2. Exemplar:**

```
http_requests_total{status="500"} # {trace_id="abc123"}
```

**3. Trace:**

```
Trace abc123 (8.5s)
├── api-service (8.5s)
│   └── db-query (8.0s)   ← bottleneck
```

**4. Logs:**

```
{"trace_id": "abc123", "level": "error", "msg": "db connection timeout"}
```

**Полная картина за секунды.**

### 🔬 Практика: корреляция

**1. Logs с trace_id:**

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
ctx, span := tracer.Start(context.Background(), "operation")
defer span.End()

logger.Info("Operation completed",
    "trace_id", span.SpanContext().TraceID().String(),
    "span_id", span.SpanContext().SpanID().String(),
)
```

**2. Exemplars:**

```go
eo.ObserveWithExemplar(duration, prometheus.Labels{
    "trace_id": span.SpanContext().TraceID().String(),
})
```

**3. Grafana datasources:**

```yaml
# Loki
jsonData:
  derivedFields:
    - datasourceUid: tempo
      matcherRegex: "trace_id=(\\w+)"
      name: TraceID
      url: "$${__value.raw}"

# Prometheus
jsonData:
  exemplarTraceIdDestinations:
    - name: trace_id
      datasourceUid: tempo
```

### 💡 Практика: как правильно коррелировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Trace ID в логах.**
2. **Exemplars в метриках.**
3. **Grafana** как единый UI.

**👍 СТОИТ:**

4. **Span events** для важных моментов.
5. **Baggage** для контекста.
6. **Correlation** между datasource.

**❌ НЕ ДЕЛАЙ:**

7. **Не теряй trace_id** в логах.
8. **Не забывай про exemplars** в метриках.
9. **Не игнорируй correlation.**

### Где мы сейчас

Мы разобрали корреляцию. Теперь — **диагностика latency**.

---

## 23.11 Диагностика latency через трейсы

### 🔌 Проблема: где застрял запрос

Запрос медленный. Где именно?

**Решение:** trace с breakdown.

### 📊 Что даёт trace

**1. Latency breakdown.**

```
Trace abc123 (8.5s)
├── api-gateway (10ms)    1.2%
├── auth-service (50ms)   5.9%
└── api-service (8.4s)    98.8%
    ├── db-query (8.0s)   94.1%   ← bottleneck
    └── cache-get (0.4s)  4.7%
```

**2. Critical path.**

Самый длинный путь.

**3. Dependencies.**

Кто кого вызывает.

**4. N+1 queries.**

Много маленьких spans.

### 🎯 Типичные проблемы

**1. Медленный DB query.**

```
db-query (8.0s)
  db.statement: SELECT * FROM orders WHERE ...
```

**Решение:** индекс, оптимизация запроса.

**2. N+1 queries.**

```
api-service (2s)
├── db-query (10ms)
├── db-query (10ms)
├── db-query (10ms)
... 200 раз
```

**Решение:** batch queries, preloading.

**3. Медленный внешний API.**

```
external-api-call (5s)
  http.url: https://slow-api.example.com
```

**Решение:** таймауты, retry, circuit breaker.

**4. Медленный cache.**

```
cache-get (1s)
  cache.key: user:123
```

**Решение:** проверить Redis, оптимизировать.

**5. Сеть.**

```
api-service (5s)
  net.peer.name: other-service
```

**Решение:** проверить сеть, DNS, firewall.

### 🎯 Анализ в Jaeger

**1. Найти медленные traces:**

```
Service: myapp
Min Duration: 1s
Lookback: 1h
```

**2. Открыть trace.**

**3. Посмотреть spans:**

- Sort by duration.
- Найти самый длинный.
- Посмотреть attributes.

**4. Посмотреть critical path.**

Jaeger показывает longest path.

**5. Сравнить с успешными:**

- Найти успешный trace.
- Сравнить с медленным.

### 🎯 Анализ в Tempo

**TraceQL:**

```
{ .service.name = "myapp" && duration > 1s }
```

**Что даёт:** медленные traces.

**Correlation с Loki:**

```
{ .service.name = "myapp" && duration > 1s } | trace_id
```

**Что даёт:** логи для медленных traces.

### 🎯 Метрики из traces

**Span metrics:**

- **Span duration** — latency.
- **Span count** — RPS.
- **Span errors** — error rate.

**Генерация в OTel Collector:**

```yaml
connectors:
  spanmetrics:
    dimensions:
      - name: http.method
      - name: http.status_code
      - name: service.name

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [spanmetrics, otlp/jaeger]
    metrics/spanmetrics:
      receivers: [spanmetrics]
      exporters: [prometheus]
```

**Что даёт:** метрики из traces. `traces_spanmetrics_latency_bucket{service_name="myapp", ...}`.

### 🎯 Root cause analysis

**Сценарий: p99 latency 8s.**

**1. Metrics:**

```
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) = 8.5s
```

**2. Traces:**

Найти медленные traces. Посмотреть breakdown.

**3. Bottleneck:**

```
db-query (8.0s)
  db.statement: SELECT * FROM orders WHERE user_id = $1 AND status = 'pending'
```

**4. Логи:**

```
{"trace_id": "...", "msg": "query execution", "duration_ms": 8000, "query": "..."}
```

**5. Root cause:**

Нет индекса на `(user_id, status)`.

**6. Fix:**

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

### 🔬 Практика: диагностика

```bash
# 1. Jaeger UI
# Search:
# Service: myapp
# Min Duration: 1s
# Lookback: 1h

# 2. Открыть trace
# Посмотреть spans
# Найти bottleneck

# 3. Span details
# Attributes, events, logs

# 4. Critical path
# Longest path

# 5. Compare
# Сравнить с успешным trace

# 6. Span metrics в Grafana
# traces_spanmetrics_latency_bucket
# topk(10, traces_spanmetrics_calls_total)
```

### 💡 Практика: как правильно диагностировать latency

**✅ ОБЯЗАТЕЛЬНО:**

1. **Trace** для медленного запроса.
2. **Breakdown** по сервисам.
3. **Critical path.**
4. **Attributes** для деталей.

**👍 СТОИТ:**

5. **Span metrics** для агрегации.
6. **Correlation** с logs.
7. **Сравнение** успешных и медленных.

**❌ НЕ ДЕЛАЙ:**

8. **Не игнорируй N+1 queries.**
9. **Не забывай про внешние API.**
10. **Не диагностируй без traces.**

### Где мы сейчас

Мы разобрали диагностику. Теперь — **OpenTelemetry Collector**.

---

## 23.12 OpenTelemetry Collector

### 🔌 Проблема: приложение не должно знать о backend

Приложение отправляет traces. Но куда? Jaeger? Tempo? Оба?

**Решение:** OTel Collector.

### 📊 Что такое Collector

**OTel Collector** — промежуточный слой между приложением и backend.

**Что делает:**

- **Receive** — принимает от приложений.
- **Process** — обрабатывает (sampling, filtering, enrichment).
- **Export** — отправляет в backend.

**Зачем:**

- **Демультиплексирование.** Несколько backends.
- **Processing.** Sampling, filtering.
- **Buffering.** Если backend недоступен.
- **Единая точка.**

### 🎯 Архитектура

```
┌──────────────┐
│  Application │
└──────┬───────┘
       │ OTLP
       ▼
┌──────────────────────────────┐
│  OTel Collector              │
│                              │
│  Receivers → Processors → Exporters
│                              │
└──────────────┬───────────────┘
               │
               │ OTLP
               ▼
┌──────────────┐
│  Jaeger      │
│  Tempo       │
│  Prometheus  │
└──────────────┘
```

### 🎯 Компоненты

**Receivers:**

- **otlp** — OTLP (gRPC/HTTP).
- **jaeger** — Jaeger.
- **zipkin** — Zipkin.
- **prometheus** — Prometheus (pull).

**Processors:**

- **batch** — батчинг.
- **memory_limiter** — лимит памяти.
- **tail_sampling** — tail-based sampling.
- **attributes** — модификация attributes.
- **filter** — фильтрация.

**Exporters:**

- **otlp** — OTLP.
- **jaeger** — Jaeger.
- **prometheus** — Prometheus.
- **loki** — Loki.

### 🎯 Конфигурация

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
  
  batch:
    timeout: 1s
    send_batch_size: 1024
  
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors
        type: status_code
        status_code:
          status_codes: [ERROR]
      - name: random
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true
  
  prometheus:
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, tail_sampling, batch]
      exporters: [otlp/jaeger]
```

### 🎯 Deployment modes

**1. Agent (DaemonSet).**

На каждой ноде. Принимает от приложений локально.

**2. Gateway (Deployment).**

Централизованно. Обрабатывает и экспортирует.

**3. Комбинация.**

Agent → Gateway → Backend.

**Рекомендация:** **Gateway** для большинства.

### 🎯 Установка

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

helm install otel-collector open-telemetry/opentelemetry-collector \
  --namespace tracing \
  --create-namespace \
  --set mode=deployment \
  --set config.receivers.otlp.protocols.grpc.endpoint=0.0.0.0:4317 \
  --set config.exporters.otlp.endpoint=jaeger:4317
```

### 🔬 Практика: Collector

```bash
# 1. Конфигурация
cat > otel-config.yaml <<'EOF'
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  
  memory_limiter:
    check_interval: 1s
    limit_mib: 512

exporters:
  debug:
    verbosity: detailed
  
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug, otlp/jaeger]
EOF

# 2. Запуск
docker run -d --name otel-collector \
  -p 4317:4317 -p 4318:4318 \
  -v $(pwd)/otel-config.yaml:/etc/otelcol/config.yaml \
  otel/opentelemetry-collector:latest

# 3. Приложение → Collector → Jaeger
# 4. Проверить в Jaeger UI
```

### 💡 Практика: как правильно использовать Collector

**✅ ОБЯЗАТЕЛЬНО:**

1. **Collector** между приложением и backend.
2. **memory_limiter** для защиты.
3. **batch** для эффективности.
4. **tail_sampling** для production.

**👍 СТОИТ:**

5. **Gateway mode** для централизации.
6. **Multiple exporters** для демультиплексирования.
7. **Span metrics** для метрик из traces.

**❌ НЕ ДЕЛАЙ:**

8. **Не экспортируй напрямую в backend** без Collector.
9. **Не забывай про memory_limiter.**
10. **Не игнорируй tail_sampling.**

### Где мы сейчас

Мы разобрали Collector. Теперь — **стоимость и overhead**.

---

## 23.13 Стоимость и overhead трейсинга

### 🔌 Проблема: трейсинг дорогой

Traces — много данных. Storage — дорого. Overhead в приложении.

### 📊 Стоимость

**Что влияет:**

1. **Объём traces** (RPS × spans × attributes).
2. **Retention** (сколько хранить).
3. **Storage** (S3, ES, ...).
4. **Processing** (Collector, Jaeger, Tempo).
5. **Network** (передача данных).

**Пример:**

- 1000 RPS × 5 spans = 5000 spans/sec.
- 30 дней retention.
- 5000 × 86400 × 30 = 13 миллиардов spans.
- ~10 KB на span = 130 TB.

**Это дорого.**

### 🎯 Overhead в приложении

**Что влияет:**

1. **Создание spans** — CPU.
2. **Attributes** — память.
3. **Context propagation** — network.
4. **Export** — network, CPU.
5. **Sampling** — CPU.

**Замеры:**

- **Без трейсинга:** baseline.
- **С трейсингом (100%):** +5-20% CPU.
- **С трейсингом (10%):** +1-3% CPU.

**Правило:** sampling обязателен.

### 🎯 Снижение стоимости

**1. Sampling.**

Head-based (10%) или tail-based (errors + slow + 10%).

**2. Не логируй всё.**

Только важные attributes.

**3. Короткий retention.**

3-7 дней для traces. Логи и метрики — дольше.

**4. Дешёвый storage.**

S3 вместо ES.

**5. Compression.**

gzip, zstd.

**6. Filtering.**

Исключить health checks, статику.

**7. Aggregation.**

Span metrics вместо raw traces.

### 🎯 Budget

**Установи бюджет:**

- **Traces:** 1-10% от RPS.
- **Retention:** 3-7 дней.
- **Storage:** S3, не ES.
- **Overhead:** < 5% CPU.

**Алерт при превышении.**

### 🎯 Span metrics vs raw traces

**Raw traces:**

- **Детали.** Полный trace.
- **Дорого.** Много данных.

**Span metrics:**

- **Агрегация.** Duration, count, errors.
- **Дешевле.** Метрики.
- **Меньше деталей.**

**Компромисс:**

- **Span metrics** для всех traces.
- **Raw traces** для sampled (1-10%).

### 🔬 Практика: снижение стоимости

```yaml
# OTel Collector
processors:
  # Sampling
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors
        type: status_code
        status_code:
          status_codes: [ERROR]
      - name: slow
        type: latency
        latency:
          threshold_ms: 1000
      - name: random
        type: probabilistic
        probabilistic:
          sampling_percentage: 5
  
  # Batch
  batch:
    timeout: 1s
    send_batch_size: 1024
  
  # Filter
  filter:
    traces:
      span:
        - 'attributes["http.target"] == "/health"'
  
  # Attributes
  attributes:
    actions:
      - key: http.request.header.authorization
        action: delete

exporters:
  # Span metrics
  spanmetrics:
    dimensions:
      - name: http.method
      - name: http.status_code
  
  # Tempo (S3)
  otlp/tempo:
    endpoint: tempo:4317

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [tail_sampling, filter, attributes, batch]
      exporters: [spanmetrics, otlp/tempo]
```

**Что даёт:**

- **Sampling:** 5% + errors + slow.
- **Filter:** health checks исключены.
- **Attributes:** authorization удалён.
- **Span metrics:** метрики из всех traces.

### 💡 Практика: как правильно управлять стоимостью

**✅ ОБЯЗАТЕЛЬНО:**

1. **Sampling** (head или tail).
2. **Retention** 3-7 дней.
3. **S3** для storage.
4. **Filter** health checks.

**👍 СТОИТ:**

5. **Span metrics** для агрегации.
6. **Compression** (zstd).
7. **Budget** и алерты.

**❌ НЕ ДЕЛАЙ:**

8. **Не храни 100% traces вечно.**
9. **Не логируй секреты в attributes.**
10. **Не забывай про overhead.**

### Где мы сейчас

Мы разобрали стоимость. Теперь — **диагностика**.

---

## 23.14 Диагностика проблем

### 🔌 Проблема: traces не появляются

Приложение инструментировано. Но traces не появляются в Jaeger/Tempo.

### 🔍 Типичные проблемы

**1. Traces не появляются.**

**Причины:**

- SDK не инициализирован.
- Exporter не работает.
- Sampling = 0.
- Collector недоступен.
- Backend недоступен.

**Диагностика:**

```bash
# 1. Проверить приложение
# В логах ищи ошибки от OTel
kubectl logs myapp-xxx | grep -i otel

# 2. Проверить Collector
kubectl logs -n tracing otel-collector-xxx

# 3. Проверить backend
kubectl logs -n tracing jaeger-xxx

# 4. Проверить endpoint
kubectl exec -n production myapp-xxx -- nc -zv otel-collector.tracing 4317
```

**2. Trace разрывается.**

**Причины:**

- Нет propagation.
- Заголовки удаляются.
- Context теряется.

**Диагностика:**

```bash
# 1. Проверить заголовки
kubectl exec -n production myapp-xxx -- tcpdump -i eth0 -A port 8080 | grep traceparent

# 2. Проверить в Jaeger
# Trace с одним сервисом вместо нескольких
```

**3. Sampling = 0.**

**Причины:**

- `TraceIDRatioBased(0)`.
- Parent-based с sampled=false.

**Диагностика:**

```go
// Проверить sampler
// В коде
sdktrace.TraceIDRatioBased(0.1)  // должно быть > 0
```

**4. Overhead слишком высокий.**

**Причины:**

- 100% sampling.
- Много attributes.
- Sync exporter.

**Диагностика:**

```bash
# CPU приложения
kubectl top pod myapp-xxx

# Замеры с/без трейсинга
```

**Решение:**

- Sampling.
- Batch exporter.
- Меньше attributes.

**5. Backend перегружен.**

**Причины:**

- Много traces.
- Медленный storage.
- Мало ресурсов.

**Диагностика:**

```bash
# Jaeger
kubectl top pod -n tracing jaeger-xxx
kubectl logs -n tracing jaeger-xxx | grep -i error

# Tempo
kubectl top pod -n tracing tempo-xxx
```

**Решение:**

- Sampling.
- Больше ресурсов.
- S3 storage.

**6. Trace ID не совпадает.**

**Причины:**

- Разные trace ID для одного запроса.
- Новый trace создаётся в child.

**Диагностика:**

```go
// Проверить, что ctx передаётся
ctx, span := tracer.Start(ctx, "child")  // не context.Background()
```

**7. Spans не связаны.**

**Причины:**

- Нет parent-child связи.
- Не передаётся ctx.

**Диагностика:**

```go
// Правильно
ctx, span := tracer.Start(ctx, "parent")
defer span.End()

child(ctx)  // ctx содержит parent span

// Неправильно
child(context.Background())  // теряет связь
```

### 🎯 Логи

```bash
# OTel Collector
kubectl logs -n tracing otel-collector-xxx

# Jaeger
kubectl logs -n tracing jaeger-collector-xxx
kubectl logs -n tracing jaeger-query-xxx

# Tempo
kubectl logs -n tracing tempo-xxx

# Приложение
kubectl logs myapp-xxx | grep -i "otel\|trace"
```

### 🎯 Debug exporter

```yaml
exporters:
  debug:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]  # вывод в логи Collector
```

**Что даёт:** видишь все spans в логах Collector.

### 🎯 Метаданные

**Проверить, что resource правильный:**

```go
res, _ := resource.New(ctx,
    resource.WithAttributes(
        semconv.ServiceName("myapp"),
        semconv.ServiceVersion("1.0.0"),
    ),
)
```

**Что даёт:** service name в Jaeger.

### 🔬 Практика: диагностика

```bash
# 1. Проверить приложение
kubectl logs myapp-xxx | grep -i "trace\|otel"

# 2. Проверить Collector
kubectl port-forward -n tracing svc/otel-collector 8888:8888
curl localhost:8888/metrics | grep otelcol

# 3. Проверить Jaeger
kubectl port-forward -n tracing svc/jaeger-query 16686:16686
curl localhost:16686/api/services

# 4. Проверить trace
curl "localhost:16686/api/traces?service=myapp&limit=10"

# 5. Debug exporter
# Включить в Collector, посмотреть логи
kubectl logs -n tracing otel-collector-xxx

# 6. Endpoint connectivity
kubectl exec -n production myapp-xxx -- nc -zv otel-collector.tracing 4317
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Логи Collector** — главный источник.
2. **Debug exporter** при проблемах.
3. **Connectivity** от приложения до Collector.
4. **Resource attributes.**

**👍 СТОИТ:**

5. **Metrics Collector** для мониторинга.
6. **Jaeger API** для проверки.
7. **Тестировать** в dev.

**❌ НЕ ДЕЛАЙ:**

8. **Не игнорируй ошибки в логах.**
9. **Не забывай про sampling.**
10. **Не диагностируй без traces.**

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **Distributed tracing** | Отслеживание запроса через сервисы. |
| **Trace** | Путь запроса. |
| **Span** | Один отрезок работы. |
| **Trace ID** | Уникальный ID запроса. |
| **Span ID** | Уникальный ID отрезка. |
| **Parent Span ID** | ID родительского span. |
| **Root span** | Первый span без parent. |
| **Child span** | Вложенный span. |
| **Context propagation** | Передача trace между сервисами. |
| **W3C Trace Context** | Стандарт заголовков. |
| **Baggage** | Дополнительный контекст. |
| **Attributes** | Key-value на span. |
| **Events** | Точки во времени внутри span. |
| **SpanKind** | Server, Client, Internal, Producer, Consumer. |
| **OpenTelemetry** | Стандарт observability. |
| **OTLP** | Протокол OpenTelemetry. |
| **Collector** | Сбор и экспорт. |
| **SDK** | Реализация API. |
| **Instrumentation** | Библиотеки для автоматической инструментации. |
| **Head-based sampling** | Решение в начале. |
| **Tail-based sampling** | Решение после завершения. |
| **Adaptive sampling** | Динамический rate. |
| **Priority sampling** | Приоритеты. |
| **Jaeger** | Distributed tracing system. |
| **Tempo** | Масштабируемый трейсинг от Grafana. |
| **TraceQL** | Язык запросов Tempo. |
| **Exemplars** | Примеры traces для метрик. |
| **Span metrics** | Метрики из traces. |

---

## Что мы узнали?

- **Distributed tracing** — отслеживание запроса через сервисы. Дополняет logs и metrics.
- **Span** — отрезок работы. **Trace** — путь запроса.
- **Context propagation** — передача trace ID через заголовки (`traceparent`).
- **OpenTelemetry** — стандарт. API, SDK, Collector, Instrumentation.
- **Auto-instrumentation** для HTTP, gRPC, DB, Redis, Kafka.
- **Sampling:** head-based, tail-based, adaptive, priority.
- **Jaeger** — UI для traces. Elasticsearch для production.
- **Tempo** — масштабируемый, S3 storage, Grafana UI.
- **Корреляция** traces + logs + metrics через trace ID и exemplars.
- **Диагностика latency** через breakdown по spans.
- **OTel Collector** — промежуточный слой.
- **Стоимость** — sampling обязателен.
- **Диагностика** — логи Collector, debug exporter, connectivity.

---

## Типичные ошибки

- ❌ **Не использовать sampling.** Overhead, стоимость.
- ❌ **Не пропагировать trace ID.** Разрыв traces.
- ❌ **Использовать `context.Background()`** в handlers.
- ❌ **Хранить 100% traces вечно.** Дорого.
- ❌ **Не использовать auto-instrumentation.** Трудоёмко.
- ❌ **Не коррелировать с logs и metrics.**
- ❌ **Забывать про `defer span.End()`.**
- ❌ **Класть секреты в attributes.**
- ❌ **Использовать vendor-specific SDK.**
- ❌ **Не мониторить Collector.**
- ❌ **Не настраивать retention.**
- ❌ **Игнорировать overhead.**
- ❌ **Не использовать tail-based sampling** в production.
- ❌ **Не фильтровать health checks.**

---

## Для быстрого повторения

- **Trace** = путь запроса. **Span** = отрезок.
- **Propagation** через `traceparent` (W3C).
- **OpenTelemetry** — стандарт. API, SDK, Collector.
- **Auto-instrumentation:** `otelhttp`, `otelgin`, `otelgrpc`, `otelsql`, `redisotel`, `otelsarama`.
- **Sampling:** head (SDK), tail (Collector).
- **Jaeger** — UI, ES storage.
- **Tempo** — S3, Grafana, TraceQL.
- **Корреляция:** trace_id в логах, exemplars в метриках.
- **Диагностика:** breakdown, critical path, bottleneck.
- **Collector:** receive, process, export.
- **Стоимость:** sampling, retention, S3, filter.
- **Диагностика:** логи Collector, debug exporter, connectivity.

---

## Вопросы для самопроверки

1. Что такое distributed tracing? Чем дополняет logs и metrics?
2. Что такое span и trace? Как связаны?
3. Что такое context propagation? Как работает?
4. Что такое OpenTelemetry? Какие компоненты?
5. Как инструментировать Go-приложение?
6. Что такое auto-instrumentation? Приведи примеры.
7. Что такое sampling? Head vs tail.
8. Что такое Jaeger? Как использовать?
9. Что такое Grafana Tempo? Чем отличается от Jaeger?
10. Как коррелировать traces с logs?
11. Что такое exemplars? Зачем нужны?
12. Как диагностировать latency через traces?
13. Что такое OTel Collector? Зачем нужен?
14. Что влияет на стоимость трейсинга? Как снизить?
15. Traces не появляются. Как диагностировать?

---

## Ответы

**1. Distributed tracing**

Отслеживание запроса через сервисы. Дополняет logs (детали) и metrics (агрегация): даёт breakdown latency по сервисам и dependencies.

**2. Span и trace**

Span — отрезок работы (один сервис, одна операция). Trace — набор spans, связанных по trace ID. Root span без parent, child spans вложены.

**3. Context propagation**

Передача trace ID между сервисами через заголовки (`traceparent`). Root span генерирует, передаёт, следующий сервис читает и создаёт child span.

**4. OpenTelemetry**

Стандарт observability от CNCF. API, SDK, Collector, Instrumentation. Vendor-neutral. OTLP для экспорта.

**5. Инструментирование Go**

SDK, `otel.Tracer`, `tracer.Start(ctx, "name")`, `span.End()`. Attributes, events, errors. `ctx` передаётся везде.

**6. Auto-instrumentation**

Библиотеки для автоматического создания spans. `otelhttp`, `otelgin`, `otelgrpc`, `otelsql`, `redisotel`, `otelsarama`.

**7. Sampling**

Head-based — решение в начале trace (SDK). Tail-based — после завершения (Collector). Head проще, tail качественнее.

**8. Jaeger**

Distributed tracing system. UI, Collector, Query. Storage: ES, Cassandra, Badger. Search by service, tags, duration, trace ID.

**9. Tempo**

От Grafana Labs. S3 storage, Grafana UI, TraceQL. Масштабируемый, дешёвый. Индексируется только trace ID.

**10. Корреляция с logs**

Trace ID в логах. Derived fields в Grafana Loki datasource. Клик по trace_id в логе → открыть trace в Tempo.

**11. Exemplars**

Примеры traces для метрик. Prometheus histogram с `trace_id` в exemplar. Клик на точку в Grafana → открыть trace.

**12. Диагностика latency**

Trace для медленного запроса. Breakdown по spans. Critical path. Attributes. Сравнение с успешными traces.

**13. OTel Collector**

Промежуточный слой. Receive (OTLP), Process (sampling, batch), Export (Jaeger, Tempo). Демультиплексирование, processing, buffering.

**14. Стоимость**

Объём traces, retention, storage, processing, network, overhead. Снизить: sampling, retention 3-7 дней, S3, filter, span metrics.

**15. Traces не появляются**

1. Логи приложения (ошибки OTel).
2. Логи Collector.
3. Connectivity: `nc -zv collector 4317`.
4. Sampling: не 0.
5. Debug exporter в Collector.

---

## Куда идти дальше?

Мы разобрали трейсинг — третий столп observability. Теперь ты знаешь:

- Distributed tracing.
- Span, trace, propagation.
- OpenTelemetry.
- Инструментирование.
- Sampling.
- Jaeger и Tempo.
- Корреляция.
- Диагностика latency.
- Collector.
- Стоимость.

Но мы пока не разобрали:

- **Отказоустойчивость и автопилот** (Глава 24).
- **Платформенная инженерия** (Глава 25).
- **Облака и multi-cloud** (Глава 26).

**Глава 24: Отказоустойчивость и автопилот.** Погнали. 🚀