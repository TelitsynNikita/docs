# 📝 Глава 21: Observability — логирование

**Что вы узнаете:**
- Что такое observability и чем отличается от мониторинга.
- Три столпа observability: logs, metrics, traces.
- Как собирать логи в Kubernetes: Fluent Bit, Fluentd, Vector.
- Что такое структурированные логи (JSON) и почему это важно.
- Как хранить и искать логи: Loki, Elasticsearch, OpenSearch.
- Как работает Grafana и как строить дашборды по логам.
- Что такое log sampling, retention, cardinality.
- Как коррелировать логи между сервисами через trace ID.
- Как настраивать алерты на логи.
- Как избежать типичных проблем: перегрузки, потери, утечки секретов.

**После прочтения вы сможете:**
- Объяснить разницу между monitoring и observability.
- Настроить сбор логов в Kubernetes.
- Писать структурированные логи в Go.
- Развернуть Loki и Grafana.
- Настроить алерты на логи.
- Диагностировать проблемы с логами.
- Связать логи с трейсами через trace ID.

---

## Содержание

- [21.0 Пролог: где логи?](#210-пролог-где-логи)
- [21.1 Monitoring vs Observability](#211-monitoring-vs-observability)
- [21.2 Три столпа observability](#212-три-столпа-observability)
- [21.3 Логи: структурированные vs неструктурированные](#213-логи-структурированные-vs-неструктурированные)
- [21.4 Как собирать логи в Kubernetes](#214-как-собирать-логи-в-kubernetes)
- [21.5 Fluent Bit: лёгкий агент](#215-fluent-bit-лёгкий-агент)
- [21.6 Loki: логи в стиле Prometheus](#216-loki-логи-в-стиле-prometheus)
- [21.7 Elasticsearch и OpenSearch: полнотекстовый поиск](#217-elasticsearch-и-opensearch-полнотекстовый-поиск)
- [21.8 Grafana: единая точка входа](#218-grafana-единая-точка-входа)
- [21.9 Log sampling и retention](#219-log-sampling-и-retention)
- [21.10 Корреляция логов через trace ID](#2110-корреляция-логов-через-trace-id)
- [21.11 Алерты на логи](#2111-алерты-на-логи)
- [21.12 Безопасность логов](#2112-безопасность-логов)
- [21.13 Диагностика проблем](#2113-диагностика-проблем)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 21.0 Пролог: где логи?

Понедельник, 9:00. Приходит алерт: пользователи жалуются, что `/api/orders` возвращает 500.

Ты открываешь Grafana. Метрики показывают:

- **Error rate**: 15% (обычно 0.1%).
- **Latency p99**: 8 секунд (обычно 200мс).
- **RPS**: без изменений.

**Проблема есть.** Но **что именно** сломано? Какая ошибка? В каком сервисе? В какой строке кода?

Ты идёшь в логи:

```bash
kubectl logs -n production deployment/api --tail=100
```

Вывод:

```
2026-01-15T09:00:12.345Z INFO  request received method=POST path=/api/orders
2026-01-15T09:00:12.350Z INFO  database query executing
2026-01-15T09:00:12.351Z ERROR failed to process order
2026-01-15T09:00:12.352Z INFO  request completed status=500
```

**`failed to process order`.** Но почему? Какая ошибка? Что в базе?

Ты идёшь в Pod'ы:

```bash
kubectl get pods -n production
# api-xxx    Running
# api-yyy    Running
# api-zzz    Running
```

**3 Pod'а.** Логи распределены между ними. Ты смотришь только один.

Ты пробуешь:

```bash
kubectl logs -n production deployment/api --tail=10000 | grep ERROR
```

**Тысячи строк.** Но correlation между запросами — нет. Непонятно, какой request_id к какой ошибке.

**Это — проблема observability.**

В традиционном подходе: логи на серверах, `grep` по файлам. В Kubernetes: логи в Pod'ах, эфемерные. Pod удалится — логи исчезнут.

**Решение:** централизованное логирование. Сбор, агрегация, поиск, корреляция.

В этой главе мы разберём observability. Начнём с логирования — первого столпа.

Это — Второй путь DevOps (Feedback) в действии. Из Главы 0: быстрое обнаружение и исправление проблем.

---

## 21.1 Monitoring vs Observability

### 🔌 Проблема: monitoring недостаточно

Ты настроил мониторинг: Prometheus + Grafana. 50 дашбордов. 200 алертов. Метрики собираются.

Но когда что-то идёт не так, ты **всё равно не знаешь, что происходит**.

- **Метрики** показывают **что** (error rate вырос).
- Но не показывают **почему** (какая ошибка, в каком запросе).
- И не показывают **где** (в каком сервисе, в какой строке).

**Monitoring говорит «что сломано». Observability говорит «почему».**

### 📊 Что такое monitoring

**Monitoring** — наблюдение за **известными** метриками.

**Ты знаешь, что может сломаться:**

- CPU > 80%.
- Memory > 90%.
- Error rate > 1%.
- Latency p99 > 1s.

**Ты настраиваешь алерты на эти метрики.**

**Проблема:** если сломается что-то **неизвестное** — ты не узнаешь.

**Пример:** новый код вводит deadlock при определённых условиях. Метрики не показывают. Пользователи жалуются. Ты не знаешь, где искать.

### 📊 Что такое observability

**Observability** — способность **задавать вопросы** о системе, которые не были предвидены.

**Из control theory:** система observability, если можно определить её внутреннее состояние по внешним выходам.

**На практике:**

- **Monitoring** — отвечает на вопросы, которые ты **заранее** задал («какой CPU?»).
- **Observability** — позволяет **задавать новые** вопросы («почему этот запрос медленный?», «какие запросы от этого пользователя упали?»).

### 🎯 Monitoring vs Observability

| Аспект | Monitoring | Observability |
|:---|:---|:---|
| **Вопросы** | Известные заранее | Любые |
| **Данные** | Метрики | Logs, metrics, traces |
| **Что даёт** | «Что сломано» | «Почему сломано» |
| **Подход** | Reactive | Investigative |
| **Инструменты** | Prometheus, Nagios | Loki, Jaeger, OpenTelemetry |

### 🎯 Пример: медленный запрос

**Monitoring:**

- Алерт: latency p99 > 1s.
- Ты знаешь, что **что-то** медленное.
- Но не знаешь **что**.

**Observability:**

- Логи: `SELECT * FROM orders WHERE ...` занял 5 секунд.
- Трейсы: запрос прошёл через 3 сервиса. 4.5 секунды — в БД, 0.5 — в сети.
- Метрики: таблица `orders` выросла в 10 раз.
- Ты знаешь: **что**, **почему**, **где**.

### 🎯 Когда что использовать

**Monitoring:**

- **Базовая гигиена.** CPU, memory, disk.
- **SLO** — error budget.
- **Известные проблемы.**

**Observability:**

- **Диагностика сложных проблем.**
- **Debugging в production.**
- **Понимание поведения системы.**

**Monitoring — часть observability.** Но не вся.

### 💡 Практика: что важно понять

**✅ ОБЯЗАТЕЛЬНО:**

1. **Monitoring — для известных проблем.**
2. **Observability — для неизвестных.**
3. **Observability требует logs + metrics + traces.**

**👍 СТОИТ:**

4. **Начинать с monitoring.**
5. **Добавлять observability** по мере роста.

**❌ НЕ ДЕЛАЙ:**

6. **Не думай, что monitoring достаточно.**
7. **Не игнорируй логи.** Они — основа диагностики.

### Где мы сейчас

Мы разобрали, что такое observability. Теперь — **три столпа**.

---

## 21.2 Три столпа observability

### 🔌 Проблема: как понять, что происходит

Система — чёрный ящик. Мы видим входы (запросы) и выходы (ответы). Но что внутри?

**Три столпа observability:**

1. **Logs** — что произошло.
2. **Metrics** — сколько.
3. **Traces** — где.

### 📊 Logs (Логи)

**Что:** текстовые записи о событиях.

**Пример:**

```
2026-01-15T09:00:12.345Z INFO  request received method=POST path=/api/orders user_id=123
2026-01-15T09:00:12.350Z INFO  database query executing query="SELECT * FROM orders WHERE user_id=123"
2026-01-15T09:00:12.351Z ERROR failed to process order error="connection refused" retry=3
```

**Что дают:**

- **Детали.** Что именно произошло.
- **Контекст.** Какие параметры.
- **Ошибки.** Что сломалось.

**Проблема:**

- **Объём.** Много данных.
- **Поиск.** Нужен полнотекстовый поиск.
- **Корреляция.** Как связать логи из разных сервисов.

### 📊 Metrics (Метрики)

**Что:** числовые значения, измеряемые во времени.

**Пример:**

```
http_requests_total{method="POST", path="/api/orders", status="200"} 12345
http_request_duration_seconds{method="POST", path="/api/orders", quantile="0.99"} 0.2
go_goroutines 45
```

**Что дают:**

- **Агрегация.** Легко считать средние, проценты.
- **Алерты.** Простые пороги.
- **Дашборды.** Графики.

**Проблема:**

- **Нет деталей.** Только числа.
- **Cardinality.** Много labels — много данных.

### 📊 Traces (Трейсы)

**Что:** путь запроса через сервисы.

**Пример:**

```
Trace ID: abc123
├── api-service (10ms)
│   ├── db-query (8ms)
│   └── cache-get (1ms)
└── notification-service (5ms)
    └── email-send (4ms)
```

**Что дают:**

- **Latency breakdown.** Где время тратится.
- **Dependencies.** Кто кого вызывает.
- **Bottlenecks.** Где узкое место.

**Проблема:**

- **Sampling.** Не все трейсы собираются.
- **Инструментация.** Нужно добавлять в код.

### 🎯 Как они работают вместе

**Сценарий:** пользователь жалуется на медленный запрос.

**Metrics:**

```
http_request_duration_seconds{path="/api/orders", quantile="0.99"} = 8.5s
```

**Вывод:** запрос медленный (8.5 секунд p99).

**Traces:**

```
Trace abc123 (8.5s total)
├── api-service (8.5s)
│   ├── db-query (8.0s)     ← узкое место
│   └── cache-get (0.1s)
```

**Вывод:** медленная БД (8 секунд).

**Logs:**

```
INFO  query="SELECT * FROM orders WHERE user_id=123 AND status='pending'"
INFO  query execution time: 8000ms
WARN  query is slow, index missing on status
```

**Вывод:** проблема в запросе, нет индекса.

**Три столпа вместе дают полную картину.**

### 🎯 Пример: 500 ошибка

**Metrics:**

```
http_requests_total{status="500"} = 15% (обычно 0.1%)
```

**Traces:**

```
Trace xyz789 (500 error)
├── api-service (FAILED)
│   └── db-query (FAILED: connection refused)
```

**Logs:**

```
ERROR failed to connect to database host=db-primary:5432 error="connection refused" retry=3
```

**Вывод:** проблема с БД.

### 🎯 OpenTelemetry: стандарт

**OpenTelemetry** — стандарт для observability.

**Что даёт:**

- **Единый API** для logs, metrics, traces.
- **SDK** для Go, Python, Java, JS, ...
- **Collector** для сбора и экспорта.
- **Независимость от vendor** — можно экспортировать в любой backend.

**Разберём подробно в Главе 23 (трейсинг).**

### 💡 Практика: как использовать три столпа

**✅ ОБЯЗАТЕЛЬНО:**

1. **Metrics для алертов** и дашбордов.
2. **Logs для деталей** и диагностики.
3. **Traces для latency** и dependencies.

**👍 СТОИТ:**

4. **OpenTelemetry** для instrumentation.
5. **Correlation** между столпами (trace ID в логах).
6. **Единый UI** (Grafana).

**❌ НЕ ДЕЛАЙ:**

7. **Не полагайся только на метрики.**
8. **Не игнорируй трейсы.**
9. **Не забывай про корреляцию.**

### Где мы сейчас

Мы разобрали три столпа. Теперь — **структурированные логи**.

---

## 21.3 Логи: структурированные vs неструктурированные

### 🔌 Проблема: grep по тексту не масштабируется

Классический подход к логам:

```go
log.Printf("User %s logged in from %s", userID, ip)
```

**Вывод:**

```
User 123 logged in from 10.0.1.5
```

**Проблемы:**

- **Парсинг.** Нужен regex для извлечения userID.
- **Хрупкость.** Изменишь формат — сломаешь парсеры.
- **Поиск.** «Найди все логи user_id=123» — regex.
- **Агрегация.** «Сколько логинов было?» — сложно.

**Решение:** структурированные логи.

### 📊 Что такое структурированные логи

**Структурированные логи** — логи в формате key-value (обычно JSON).

**Пример:**

```json
{
  "timestamp": "2026-01-15T09:00:12.345Z",
  "level": "INFO",
  "message": "User logged in",
  "user_id": "123",
  "ip": "10.0.1.5",
  "service": "auth",
  "trace_id": "abc123"
}
```

**Что даёт:**

- **Парсинг.** Автоматически.
- **Поиск.** `user_id=123` — точный поиск.
- **Агрегация.** `count by user_id` — легко.
- **Корреляция.** `trace_id` — связь с трейсами.

### 🎯 Структурированные логи в Go

**Стандартный `log`:**

```go
log.Printf("User %s logged in from %s", userID, ip)
// User 123 logged in from 10.0.1.5
```

**`log/slog` (Go 1.21+):**

```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("User logged in",
    "user_id", userID,
    "ip", ip,
)
```

**Вывод:**

```json
{"time":"2026-01-15T09:00:12.345Z","level":"INFO","msg":"User logged in","user_id":"123","ip":"10.0.1.5"}
```

**`zerolog`:**

```go
import "github.com/rs/zerolog/log"

log.Info().
    Str("user_id", userID).
    Str("ip", ip).
    Msg("User logged in")
```

**`zap`:**

```go
import "go.uber.org/zap"

logger, _ := zap.NewProduction()
logger.Info("User logged in",
    zap.String("user_id", userID),
    zap.String("ip", ip),
)
```

### 🎯 Уровни логирования

| Уровень | Когда использовать |
|:---|:---|
| **DEBUG** | Детали для отладки (обычно выключен в prod) |
| **INFO** | Обычные события |
| **WARN** | Потенциальные проблемы |
| **ERROR** | Ошибки, требующие внимания |
| **FATAL** | Критические ошибки, процесс завершается |

**Правило:** в production — `INFO` и выше. `DEBUG` — только при отладке.

### 🎯 Что логировать

**✅ Логировать:**

- **Запросы.** Method, path, status, duration.
- **Ошибки.** С контекстом (что, где, почему).
- **Бизнес-события.** Заказ создан, платёж прошёл.
- **Внешние вызовы.** К каким сервисам, сколько заняло.
- **Аутентификация.** Login, logout, failed attempts.

**❌ Не логировать:**

- **Пароли, токены, ключи.**
- **Персональные данные** (PII) без необходимости.
- **Полные тела запросов** (могут быть большие).
- **Слишком много.** Логи — не замена отладчику.

### 🎯 Формат логов

**JSON — стандарт для структурированных логов.**

**Пример из Go:**

```go
logger.Info("HTTP request",
    "method", r.Method,
    "path", r.URL.Path,
    "status", status,
    "duration_ms", duration.Milliseconds(),
    "user_id", userID,
    "trace_id", traceID,
    "remote_addr", r.RemoteAddr,
)
```

**Вывод:**

```json
{
  "time": "2026-01-15T09:00:12.345Z",
  "level": "INFO",
  "msg": "HTTP request",
  "method": "POST",
  "path": "/api/orders",
  "status": 200,
  "duration_ms": 45,
  "user_id": "123",
  "trace_id": "abc123",
  "remote_addr": "10.0.1.5:54321"
}
```

### 🎯 Context propagation

**В Go — через `context.Context`:**

```go
func handleRequest(ctx context.Context, w http.ResponseWriter, r *http.Request) {
    // Извлечь request ID из заголовков или сгенерировать
    requestID := r.Header.Get("X-Request-ID")
    if requestID == "" {
        requestID = uuid.New().String()
    }
    
    // Добавить в контекст
    ctx = context.WithValue(ctx, "request_id", requestID)
    
    // Логгер с request_id
    logger := slog.With("request_id", requestID)
    
    logger.Info("Processing request", "path", r.URL.Path)
    
    // Передать контекст дальше
    processOrder(ctx, logger)
}

func processOrder(ctx context.Context, logger *slog.Logger) {
    logger.Info("Processing order")
    // Логгер уже содержит request_id
}
```

### 🔬 Практика: структурированные логи в Go

```go
package main

import (
    "log/slog"
    "net/http"
    "os"
    "time"
)

func main() {
    // JSON-логгер
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))
    slog.SetDefault(logger)
    
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Request ID
        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = "req-" + time.Now().Format("20060102150405")
        }
        
        // Логгер с контекстом
        reqLogger := logger.With(
            "request_id", requestID,
            "method", r.Method,
            "path", r.URL.Path,
        )
        
        reqLogger.Info("Request received")
        
        // Обработка
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("OK"))
        
        // Лог завершения
        reqLogger.Info("Request completed",
            "status", 200,
            "duration_ms", time.Since(start).Milliseconds(),
        )
    })
    
    http.ListenAndServe(":8080", nil)
}
```

**Запуск:**

```bash
go run main.go
curl localhost:8080
```

**Вывод:**

```json
{"time":"2026-01-15T09:00:12.345Z","level":"INFO","msg":"Request received","request_id":"req-20260115090012","method":"GET","path":"/"}
{"time":"2026-01-15T09:00:12.346Z","level":"INFO","msg":"Request completed","request_id":"req-20260115090012","method":"GET","path":"/","status":200,"duration_ms":1}
```

### 💡 Практика: как правильно логировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **JSON-формат** в production.
2. **Структурированные поля** (key-value).
3. **Correlation ID** (request_id, trace_id).
4. **Уровни** (INFO, WARN, ERROR).
5. **stdout/stderr**, не файлы.

**👍 СТОИТ:**

4. **`log/slog`** (Go 1.21+) или `zerolog`/`zap`.
5. **Context propagation.**
6. **Structured logging middleware** для HTTP.

**❌ НЕ ДЕЛАЙ:**

7. **Не логируй секреты.** Пароли, токены, PII.
8. **Не используй `fmt.Printf`** для логов.
9. **Не пиши логи в файлы** внутри контейнера.
10. **Не логируй слишком много.** DEBUG в production.

### Где мы сейчас

Мы разобрали структурированные логи. Теперь — **как собирать логи в Kubernetes**.

---

## 21.4 Как собирать логи в Kubernetes

### 🔌 Проблема: логи в Pod'ах эфемерны

В Kubernetes логи пишутся в **stdout/stderr** контейнера. Docker/containerd сохраняет их в файл на ноде.

**Проблемы:**

- **Pod удалится** — логи исчезнут.
- **Много Pod'ов** — логи распределены.
- **Нет поиска** — только `kubectl logs`.
- **Нет агрегации** — нельзя искать по всем сервисам.

**Решение:** централизованный сбор логов.

### 📊 Архитектура сбора логов

```
┌─────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                        │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Pod A       │  │  Pod B       │  │  Pod C       │      │
│  │              │  │              │  │              │      │
│  │  stdout ────┐│  │  stdout ────┐│  │  stdout ────┐│      │
│  └─────────────┘│  └─────────────┘│  └─────────────┘│      │
│                 │                  │                  │      │
│                 ▼                  ▼                  ▼      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Логи на ноде (/var/log/containers/)          │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                                │
│                            │ DaemonSet читает               │
│                            ▼                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Логирующий агент (Fluent Bit)                │   │
│  │         На каждой ноде                               │   │
│  └─────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────┘
                             │
                             │ Отправка
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                  BACKEND ХРАНЕНИЯ                            │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Loki        │  │  Elasticsearch│  │  CloudWatch  │      │
│  │  (лёгкий)    │  │  (мощный)    │  │  (managed)   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
└────────────────────────────┬────────────────────────────────┘
                             │
                             │ UI
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                      Grafana                                 │
│                      Kibana                                  │
└─────────────────────────────────────────────────────────────┘
```

### 🎯 Три подхода к сбору

**1. Node-level agent (DaemonSet).**

**Как работает:** агент запускается на каждой ноде, читает логи всех Pod'ов.

**Пример:** Fluent Bit, Fluentd, Vector, Filebeat.

**Плюсы:**

- **Простой.** Один агент на ноду.
- **Не требует изменений** в Pod'ах.
- **Ловит все логи.**

**Минусы:**

- **Не знает про логи приложения** (только stdout).
- **Зависит от ноды** (если нода упала — логи потеряны).

**2. Sidecar.**

**Как работает:** sidecar-контейнер в Pod'е читает логи основного контейнера.

**Пример:** Fluent Bit sidecar.

**Плюсы:**

- **Изоляция.** Каждый Pod — свой агент.
- **Может читать логи из файлов.**

**Минусы:**

- **Больше ресурсов** (sidecar на каждый Pod).
- **Сложнее конфигурация.**

**3. Push из приложения.**

**Как работает:** приложение само отправляет логи в backend.

**Пример:** Loki client, Elasticsearch client.

**Плюсы:**

- **Прямой контроль.**
- **Структурированные логи.**

**Минусы:**

- **Изменения в коде.**
- **Блокировка при недоступности backend.**

**Рекомендация:** **Node-level agent** для большинства случаев. Sidecar — для специфичных.

### 🎯 Kubernetes metadata

**Важно:** к логам нужно добавлять **Kubernetes metadata**:

- **Pod name** (`api-7d9f8c6b4d-abc12`).
- **Namespace** (`production`).
- **Deployment** (`api`).
- **Node** (`node-1`).
- **Container** (`api`).
- **Labels** (`app=api`, `version=v1.2.3`).

**Зачем:** чтобы можно было фильтровать по сервису, окружению, версии.

**Пример лога с metadata:**

```json
{
  "timestamp": "2026-01-15T09:00:12.345Z",
  "level": "INFO",
  "message": "Request received",
  "request_id": "abc123",
  "kubernetes": {
    "namespace_name": "production",
    "pod_name": "api-7d9f8c6b4d-abc12",
    "container_name": "api",
    "labels": {
      "app": "api",
      "version": "v1.2.3"
    }
  }
}
```

### 🎯 Выбор backend

| Backend | Плюсы | Минусы | Когда |
|:---|:---|:---|:---|
| **Loki** | Лёгкий, дешёвый, интеграция с Grafana | Нет полнотекстового поиска | Большинство случаев |
| **Elasticsearch** | Мощный поиск, агрегации | Тяжёлый, дорогой | Большие объёмы, сложные запросы |
| **OpenSearch** | Open-source форк ES | Тяжёлый | Аналогично ES |
| **CloudWatch Logs** | Managed AWS | Vendor lock-in, дорого | AWS-only |
| **Datadog/New Relic** | Managed, всё в одном | Дорого | Enterprise |

**Рекомендация:** **Loki** для большинства. **Elasticsearch** — если нужен полнотекстовый поиск.

### 🔬 Практика: логи в Kubernetes

```bash
# 1. Логи Pod'а
kubectl logs api-7d9f8c6b4d-abc12 -n production

# 2. Логи всех Pod'ов Deployment
kubectl logs -n production deployment/api --all-containers=true

# 3. Follow
kubectl logs -f -n production deployment/api

# 4. Последние 100 строк
kubectl logs --tail=100 -n production deployment/api

# 5. С таймстампами
kubectl logs --timestamps -n production deployment/api

# 6. Предыдущий экземпляр (если Pod упал)
kubectl logs --previous -n production api-xxx

# 7. Все Pod'ы с меткой
kubectl logs -n production -l app=api --tail=50

# 8. Логи за период
kubectl logs --since=1h -n production deployment/api
kubectl logs --since-time="2026-01-15T09:00:00Z" -n production deployment/api
```

### 💡 Практика: как правильно собирать логи

**✅ ОБЯЗАТЕЛЬНО:**

1. **stdout/stderr** в приложении. Не файлы.
2. **Node-level agent** (Fluent Bit) на каждой ноде.
3. **Kubernetes metadata** в логах.
4. **Централизованный backend** (Loki, ES).

**👍 СТОИТ:**

4. **Структурированные логи** (JSON).
5. **Correlation ID** (request_id, trace_id).
6. **Retention policies.**

**❌ НЕ ДЕЛАЙ:**

7. **Не пиши логи в файлы** внутри контейнера.
8. **Не полагайся только на `kubectl logs`.**
9. **Не игнорируй metadata.**

### Где мы сейчас

Мы разобрали, как собирать логи. Теперь — **Fluent Bit** — конкретный агент.

---

## 21.5 Fluent Bit: лёгкий агент

### 🔌 Проблема: какой агент выбрать

Fluentd, Fluent Bit, Vector, Filebeat, Promtail. Как выбрать?

**Fluent Bit** — самый популярный для Kubernetes. Лёгкий, быстрый, написан на C.

### 📊 Что такое Fluent Bit

**Fluent Bit** — лёгкий процессор и форвардер логов.

**Характеристики:**

- **Написан на C.** Быстрый, минимальное потребление памяти.
- **~450 KB** в размере.
- **Плагины.** Input, Filter, Output.
- **Kubernetes-native.** DaemonSet, читает `/var/log/containers/`.
- **Backend-agnostic.** Loki, ES, Kafka, S3, CloudWatch.

### 🎯 Архитектура Fluent Bit

```
Input → Parser → Filter → Buffer → Output
```

**Input:**

- **tail** — читает файлы (логи контейнеров).
- **systemd** — journald.
- **forward** — принимает от других Fluent Bit.
- **http** — HTTP-запросы.

**Parser:**

- **json** — JSON.
- **regex** — regex.
- **logfmt** — key=value.

**Filter:**

- **kubernetes** — добавляет K8s metadata.
- **grep** — фильтрует по regex.
- **modify** — изменяет поля.
- **nest** — вложенность.
- **lua** — скрипты.

**Output:**

- **loki** — Grafana Loki.
- **es** — Elasticsearch.
- **kafka** — Kafka.
- **s3** — AWS S3.
- **cloudwatch** — CloudWatch.
- **stdout** — консоль.

### 🎯 Установка Fluent Bit

**Через Helm:**

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

helm install fluent-bit fluent/fluent-bit \
  --namespace logging \
  --create-namespace
```

**Что установится:**

- **DaemonSet** на каждой ноде.
- **ConfigMap** с конфигурацией.
- **ServiceAccount** с правами читать Pod'ы.

### 🎯 Конфигурация Fluent Bit

**fluent-bit.conf:**

```ini
[SERVICE]
    Flush         1
    Daemon        Off
    Log_Level     info
    Parsers_File  parsers.conf
    HTTP_Server   On
    HTTP_Listen   0.0.0.0
    HTTP_Port     2020

[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Parser            cri
    Tag               kube.*
    Refresh_Interval  5
    Mem_Buf_Limit     50MB
    Skip_Long_Lines   On

[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
    Kube_Tag_Prefix     kube.var.log.containers.
    Merge_Log           On
    Merge_Log_Key       log_processed
    Keep_Log            Off
    Annotations         Off
    Labels              On

[FILTER]
    Name    grep
    Match   kube.*
    Exclude level debug

[OUTPUT]
    Name            loki
    Match           kube.*
    Host            loki.monitoring.svc.cluster.local
    Port            3100
    Labels          job=fluent-bit
    Label_Keys      $kubernetes['namespace_name'],$kubernetes['pod_name'],$kubernetes['container_name']
    Line_Format     json
```

**Что произойдёт:**

1. Fluent Bit читает все логи контейнеров из `/var/log/containers/`.
2. Парсит CRI-формат (containerd).
3. Обогащает Kubernetes metadata (namespace, pod, labels).
4. Исключает `level=debug`.
5. Отправляет в Loki.

### 🎯 CRI-парсер

**containerd** пишет логи в CRI-формате:

```
2026-01-15T09:00:12.345Z stdout F {"level":"info","msg":"Request received"}
```

**Формат:**

- **Timestamp** — RFC3339.
- **Stream** — stdout или stderr.
- **Tag** — F (full) или P (partial).
- **Log** — сам лог.

**parsers.conf:**

```ini
[PARSER]
    Name        cri
    Format      regex
    Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<log>.*)$
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L%z
```

### 🎯 Фильтр kubernetes

**Что делает:**

- Обогащает логи metadata из Kubernetes API.
- Извлекает `namespace_name`, `pod_name`, `container_name`.
- Извлекает labels и annotations.

**Результат:**

```json
{
  "timestamp": "2026-01-15T09:00:12.345Z",
  "stream": "stdout",
  "log": "{\"level\":\"info\",\"msg\":\"Request received\"}",
  "kubernetes": {
    "namespace_name": "production",
    "pod_name": "api-7d9f8c6b4d-abc12",
    "container_name": "api",
    "labels": {
      "app": "api",
      "version": "v1.2.3"
    }
  }
}
```

### 🎯 Multi-line логи

**Проблема:** stack trace в логах — многострочный.

```
ERROR: Something failed
  at main.go:42
  at handler.go:100
```

**Каждая строка** — отдельный лог. Не связаны.

**Решение:** multi-line parser.

```ini
[PARSER]
    Name        multiline_go
    Format      regex
    Regex       ^\s
    Rules       go
```

**Или через filter:**

```ini
[FILTER]
    Name          multiline
    Match         kube.*
    Multiline.Key_Content log
    Multiline.Parser  go
```

**go-parser:**

```ini
[MULTILINE_PARSER]
    name          go
    type          regex
    flush_timeout 1000
    rule          "start_state"  "/(^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})/"  "cont"
    rule          "cont"          "/^\s/"                                  "cont"
```

**Что произойдёт:** multi-line логи объединятся в одну запись.

### 🔬 Практика: Fluent Bit

```bash
# 1. Установить Fluent Bit
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

cat > values.yaml <<'EOF'
config:
  service: |
    [SERVICE]
        Flush         1
        Daemon        Off
        Log_Level     info
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

  inputs: |
    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            cri
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On

  filters: |
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Merge_Log_Key       log_processed
        Keep_Log            Off

  outputs: |
    [OUTPUT]
        Name   stdout
        Match  *
EOF

helm install fluent-bit fluent/fluent-bit \
  --namespace logging \
  --create-namespace \
  -f values.yaml

# 2. Проверить
kubectl get pods -n logging
# fluent-bit-xxx   1/1   Running   (DaemonSet — на каждой ноде)

kubectl logs -n logging daemonset/fluent-bit --tail=20
```

### 💡 Практика: как правильно настроить Fluent Bit

**✅ ОБЯЗАТЕЛЬНО:**

1. **DaemonSet** на каждой ноде.
2. **CRI-парсер** для containerd.
3. **Kubernetes filter** для metadata.
4. **Mem_Buf_Limit** для ограничения памяти.
5. **Skip_Long_Lines** для избежания зависаний.

**👍 СТОИТ:**

4. **Multi-line parser** для stack trace.
5. **Grep filter** для исключения debug.
6. **Retry** при недоступности backend.

**❌ НЕ ДЕЛАЙ:**

7. **Не логируй всё без фильтров.** Перегрузка.
8. **Не забывай про Mem_Buf_Limit.** Утечка памяти.
9. **Не отправляй в stdout** в production.

### Где мы сейчас

Мы разобрали Fluent Bit. Теперь — **Loki** — backend для логов.

---

## 21.6 Loki: логи в стиле Prometheus

### 🔌 Проблема: Elasticsearch тяжёлый

Elasticsearch — мощный, но тяжёлый. Нужно много RAM, дисков, настройки.

**Loki** — альтернатива от Grafana Labs. Лёгкий, дешёвый.

### 📊 Что такое Loki

**Loki** — система агрегации логов от Grafana Labs.

**Философия:** «Prometheus для логов».

**Отличия от Elasticsearch:**

| Аспект | Loki | Elasticsearch |
|:---|:---|:---|
| **Индексация** | Только labels | Полнотекстовая |
| **RAM** | Мало | Много |
| **Диск** | Сжатые chunks | Индексы + документы |
| **Стоимость** | Низкая | Высокая |
| **Поиск** | По labels + regex | Полнотекстовый |
| **Сложность** | Простой | Сложный |

**Ключевая идея:** индексируются только **labels** (namespace, pod, app), а не содержимое логов.

**Пример:**

```
Log line: {"level":"error","msg":"Connection failed","user_id":"123"}
Labels: namespace=production, pod=api-xxx, app=api

Поиск по labels: быстро.
Поиск по msg: полное сканирование (медленнее).
```

### 🎯 Архитектура Loki

```
┌─────────────────────────────────────────────────────────────┐
│                    LOKI                                     │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Distributor │  │  Ingester    │  │  Querier     │      │
│  │              │  │              │  │              │      │
│  │  Принимает   │  │  Буферизует  │  │  Ищет логи   │      │
│  │  логи        │  │  и пишет     │  │              │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Storage (S3, GCS, filesystem)            │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                             │
                             │ LogQL
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    GRAFANA                                   │
└─────────────────────────────────────────────────────────────┘
```

### 🎯 LogQL: язык запросов

**LogQL** — похож на PromQL, но для логов.

**Базовый запрос:**

```logql
{namespace="production", app="api"}
```

**С фильтром:**

```logql
{namespace="production", app="api"} |= "error"
{namespace="production", app="api"} != "debug"
{namespace="production", app="api"} |~ "error|fail"
```

**С парсингом JSON:**

```logql
{namespace="production", app="api"} | json | level="error"
{namespace="production", app="api"} | json | user_id="123"
```

**Метрики из логов:**

```logql
# Rate ошибок в секунду
sum(rate({namespace="production", app="api"} |= "error" [5m]))

# Топ-10 user_id по количеству запросов
topk(10, sum by (user_id) (rate({namespace="production", app="api"} | json [5m])))
```

### 🎯 Установка Loki

**Через Helm:**

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki \
  --namespace monitoring \
  --create-namespace
```

**Loki + Promtail (для сбора):**

```bash
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --create-namespace \
  --set promtail.enabled=true \
  --set grafana.enabled=true
```

**Loki + Grafana:**

```bash
helm install loki grafana/loki \
  --namespace monitoring \
  --create-namespace

helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=true
```

### 🎯 Loki в single binary vs microservices

**Single binary:**

- **Всё в одном процессе.**
- **Просто.**
- **Для dev и небольших production.**

**Microservices:**

- **Distributor, Ingester, Querier** — отдельно.
- **Масштабируется.**
- **Для больших production.**

**Для большинства** — single binary достаточно.

### 🎯 Retention

**Retention** — сколько хранить логи.

```yaml
# loki-config.yaml
limits_config:
  retention_period: 30d

compactor:
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 150
```

**Что произойдёт:** логи старше 30 дней удаляются.

**Правило:** разные retention для разных нужд:

- **Debug** — 3-7 дней.
- **Application** — 30 дней.
- **Audit** — 1 год.

### 🎯 Loki vs другие

| Backend | RAM | Диск | Поиск | Стоимость |
|:---|:---|:---|:---|:---|
| **Loki** | 1-2 GB | Низкий | Labels + regex | Низкая |
| **Elasticsearch** | 8-16 GB | Высокий | Полнотекстовый | Высокая |
| **OpenSearch** | 8-16 GB | Высокий | Полнотекстовый | Высокая |
| **CloudWatch** | — | — | Полнотекстовый | Высокая |

### 🔬 Практика: Loki

```bash
# 1. Установить Loki
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
  --namespace monitoring \
  --create-namespace \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=10Gi \
  --set grafana.enabled=true \
  --set promtail.enabled=true

# 2. Проверить
kubectl get pods -n monitoring

# 3. Получить пароль Grafana
kubectl get secret -n monitoring grafana -o jsonpath="{.data.admin-password}" | base64 -d

# 4. Port-forward Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80

# 5. Открыть http://localhost:3000
# Login: admin / <пароль>

# 6. Добавить Loki datasource
# Configuration → Data Sources → Add → Loki
# URL: http://loki:3100

# 7. Explore → Loki → Запрос
# {namespace="default"}
```

### 💡 Практика: как правильно использовать Loki

**✅ ОБЯЗАТЕЛЬНО:**

1. **Loki для большинства случаев.** Простой, дешёвый.
2. **Labels правильно.** Не слишком много (cardinality).
3. **Retention настроить.**
4. **Grafana для UI.**

**👍 СТОИТ:**

4. **S3 для storage** в production.
5. **Single binary** для небольших нагрузок.
6. **LogQL для метрик** из логов.

**❌ НЕ ДЕЛАЙ:**

7. **Не индексируй всё.** Loki — не ES.
8. **Не используй high-cardinality labels.** user_id, request_id.
9. **Не храни логи вечно.** Дорого.

### Где мы сейчас

Мы разобрали Loki. Теперь — **Elasticsearch и OpenSearch**.

---

## 21.7 Elasticsearch и OpenSearch: полнотекстовый поиск

### 🔌 Проблема: нужен полнотекстовый поиск

Loki хорош для поиска по labels. Но если нужно:

- **Искать по содержимому** (`error` в любом месте).
- **Агрегировать по полям** (топ-10 user_id).
- **Сложные запросы** (regex, fuzzy).

**Elasticsearch** или **OpenSearch**.

### 📊 Elasticsearch

**Elasticsearch** — распределённый поисковый движок на Apache Lucene.

**Что даёт:**

- **Полнотекстовый поиск.** Быстрый поиск по любому тексту.
- **Индексация.** Все поля индексируются.
- **Агрегации.** Топ-N, гистограммы, stats.
- **Scalability.** Шардирование, репликация.

**Проблемы:**

- **Тяжёлый.** 8-16 GB RAM минимум.
- **Дорогой.** Много дисков.
- **Сложный.** Настройка, управление.
- **Лицензия.** Elastic License (не OSS).

### 📊 OpenSearch

**OpenSearch** — форк Elasticsearch от AWS. Apache 2.0 лицензия.

**Отличия:**

- **Лицензия.** Apache 2.0.
- **Управляется AWS.**
- **Совместим с ES API.**

**Использование:** аналогично ES.

### 🎯 Архитектура ELK

```
┌─────────────────────────────────────────────────────────────┐
│                          ELK                                │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Logstash    │  │Elasticsearch │  │   Kibana     │      │
│  │              │  │              │  │              │      │
│  │  Обработка   │  │  Хранение    │  │  UI          │      │
│  │  и парсинг   │  │  и поиск     │  │              │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                             ▲
                             │
┌─────────────────────────────────────────────────────────────┐
│                     Fluent Bit / Beats                       │
└─────────────────────────────────────────────────────────────┘
```

**Компоненты:**

- **Beats / Fluent Bit** — сбор логов.
- **Logstash** — обработка (парсинг, фильтрация, обогащение).
- **Elasticsearch** — хранение и поиск.
- **Kibana** — UI.

### 🎯 Установка Elasticsearch

**Через Helm:**

```bash
helm repo add elastic https://helm.elastic.co
helm repo update

helm install elasticsearch elastic/elasticsearch \
  --namespace logging \
  --create-namespace \
  --set replicas=3 \
  --set minimumMasterNodes=2 \
  --set resources.requests.memory=2Gi \
  --set resources.limits.memory=2Gi

helm install kibana elastic/kibana \
  --namespace logging
```

**OpenSearch:**

```bash
helm repo add opensearch https://opensearch-project.github.io/helm-charts
helm repo update

helm install opensearch opensearch/opensearch \
  --namespace logging \
  --create-namespace

helm install opensearch-dashboards opensearch/opensearch-dashboards \
  --namespace logging
```

### 🎯 Индексы и mappings

**Index** — как «таблица» в БД.

**Mapping** — схема полей.

```json
PUT /logs-2026.01.15
{
  "mappings": {
    "properties": {
      "timestamp": { "type": "date" },
      "level": { "type": "keyword" },
      "message": { "type": "text" },
      "user_id": { "type": "keyword" },
      "duration_ms": { "type": "integer" },
      "kubernetes": {
        "properties": {
          "namespace": { "type": "keyword" },
          "pod": { "type": "keyword" },
          "labels": { "type": "object" }
        }
      }
    }
  }
}
```

**Типы:**

- **`text`** — полнотекстовый поиск (analyzed).
- **`keyword`** — точное совпадение (не analyzed).
- **`date`** — даты.
- **`integer`, `long`, `float`** — числа.
- **`object`** — вложенные объекты.

### 🎯 Index Lifecycle Management (ILM)

**ILM** — автоматическое управление жизненным циклом индексов.

**Фазы:**

1. **Hot** — активная запись.
2. **Warm** — только чтение.
3. **Cold** — редко читается.
4. **Delete** — удаление.

**Пример:**

```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

**Что произойдёт:**

- Каждый день новый индекс.
- Через 7 дней — оптимизация.
- Через 30 дней — удаление.

### 🎯 Kibana / OpenSearch Dashboards

**UI для поиска и визуализации.**

**Возможности:**

- **Discover** — поиск логов.
- **Visualize** — графики.
- **Dashboard** — дашборды.
- **Alerting** — алерты.

**Запросы:**

```
level: "error" AND user_id: "123"
level: "error" AND duration_ms: >1000
kubernetes.namespace: "production" AND level: "error"
```

### 🎯 Loki vs Elasticsearch

| Аспект | Loki | Elasticsearch |
|:---|:---|:---|
| **Индексация** | Labels | Полнотекстовая |
| **RAM** | 1-2 GB | 8-16 GB |
| **Диск** | Низкий | Высокий |
| **Поиск по контенту** | Медленный | Быстрый |
| **Агрегации** | Ограниченные | Мощные |
| **UI** | Grafana | Kibana |
| **Стоимость** | Низкая | Высокая |
| **Сложность** | Простой | Сложный |

**Когда что:**

- **Loki** — большинство случаев. Логи по labels.
- **Elasticsearch** — если нужен полнотекстовый поиск, агрегации.

### 🔬 Практика: Elasticsearch

```bash
# 1. Установить Elasticsearch
helm repo add elastic https://helm.elastic.co
helm repo update

helm install elasticsearch elastic/elasticsearch \
  --namespace logging \
  --create-namespace \
  --set replicas=1 \
  --set minimumMasterNodes=1 \
  --set resources.requests.memory=1Gi \
  --set resources.limits.memory=1Gi \
  --set volumeClaimTemplate.resources.requests.storage=10Gi

# 2. Проверить
kubectl get pods -n logging

# 3. Port-forward
kubectl port-forward -n logging svc/elasticsearch-master 9200:9200

# 4. Проверить API
curl -X GET "localhost:9200/_cluster/health?pretty"

# 5. Создать индекс
curl -X PUT "localhost:9200/logs-test?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "timestamp": { "type": "date" },
      "level": { "type": "keyword" },
      "message": { "type": "text" },
      "user_id": { "type": "keyword" }
    }
  }
}'

# 6. Добавить документ
curl -X POST "localhost:9200/logs-test/_doc?pretty" -H 'Content-Type: application/json' -d'
{
  "timestamp": "2026-01-15T09:00:12Z",
  "level": "error",
  "message": "Connection failed",
  "user_id": "123"
}'

# 7. Поиск
curl -X GET "localhost:9200/logs-test/_search?q=level:error&pretty"
```

### 💡 Практика: как правильно использовать ES

**✅ ОБЯЗАТЕЛЬНО:**

1. **ILM** для управления retention.
2. **Mapping** явно.
3. **Replicas** для отказоустойчивости.
4. **Monitoring** кластера ES.

**👍 СТОИТ:**

4. **OpenSearch** для open-source.
5. **Managed** (AWS OpenSearch, Elastic Cloud) для простоты.
6. **Sharding** для больших объёмов.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй для маленьких объёмов.** Loki дешевле.
8. **Не храни логи без retention.**
9. **Не используй `text` для всех полей.** `keyword` быстрее.

### Где мы сейчас

Мы разобрали Elasticsearch. Теперь — **Grafana** — единый UI.

---

## 21.8 Grafana: единая точка входа

### 🔌 Проблема: много UI для разных данных

- **Prometheus** — свой UI.
- **Loki** — Grafana.
- **Jaeger** — свой UI.
- **Elasticsearch** — Kibana.

**Grafana** объединяет всё в одном UI.

### 📊 Что такое Grafana

**Grafana** — open-source платформа для визуализации и мониторинга.

**Что даёт:**

- **Единый UI** для всех datasource.
- **Dashboards** — графики, таблицы.
- **Explore** — ad-hoc запросы.
- **Alerting** — алерты.
- **Correlation** — связь между datasource.

**Поддерживаемые datasource:**

- **Prometheus** (metrics).
- **Loki** (logs).
- **Jaeger / Tempo** (traces).
- **Elasticsearch** (logs).
- **PostgreSQL, MySQL** (SQL).
- **CloudWatch, Datadog** (cloud).
- И десятки других.

### 🎯 Установка Grafana

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install grafana grafana/grafana \
  --namespace monitoring \
  --create-namespace \
  --set persistence.enabled=true \
  --set persistence.size=10Gi \
  --set adminPassword=admin123
```

**Проверка:**

```bash
kubectl get pods -n monitoring
# grafana-xxx   1/1   Running

# Пароль
kubectl get secret -n monitoring grafana -o jsonpath="{.data.admin-password}" | base64 -d

# Port-forward
kubectl port-forward -n monitoring svc/grafana 3000:80

# Открыть http://localhost:3000
# Login: admin / admin123
```

### 🎯 Data Sources

**Добавление datasource:**

1. **Configuration → Data Sources → Add data source.**
2. Выбрать тип (Prometheus, Loki, Jaeger, ...).
3. Указать URL.
4. Save & Test.

**Пример: Loki:**

```
Name: Loki
URL: http://loki.monitoring.svc.cluster.local:3100
```

**Пример: Prometheus:**

```
Name: Prometheus
URL: http://prometheus.monitoring.svc.cluster.local:9090
```

**Пример: Jaeger:**

```
Name: Jaeger
URL: http://jaeger-query.tracing.svc.cluster.local:16686
```

### 🎯 Explore

**Explore** — ad-hoc запросы.

**Для Loki:**

```logql
{namespace="production", app="api"} |= "error"
```

**Для Prometheus:**

```promql
rate(http_requests_total{status="500"}[5m])
```

**Для Jaeger:**

- Выбрать сервис.
- Выбрать операцию.
- Найти трейсы.

### 🎯 Dashboards

**Dashboard** — набор панелей.

**Панели:**

- **Time series** — график по времени.
- **Stat** — одно значение.
- **Table** — таблица.
- **Logs** — логи.
- **Heatmap** — heatmap.

**Пример дашборда для API:**

1. **RPS** (time series).
2. **Error rate** (time series).
3. **Latency p50/p95/p99** (time series).
4. **Top errors** (table).
5. **Recent errors** (logs).

### 🎯 Correlations

**Correlation** — связь между datasource.

**Пример:** клик по точке на графике метрик → переход к логам за этот период.

**Настройка:**

1. **Dashboard → Settings → Correlations.**
2. Source: Prometheus.
3. Target: Loki.
4. Result: фильтр по времени.

### 🎯 Provisioning

**Provisioning** — автоматическое создание datasources и dashboards из файлов.

**datasources.yaml:**

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
  
  - name: Loki
    type: loki
    url: http://loki:3100
    jsonData:
      derivedFields:
        - datasourceUid: tempo
          matcherRegex: "trace_id=(\\w+)"
          name: TraceID
          url: "$${__value.raw}"
```

**Что даёт:** datasource создаётся автоматически при старте.

**Для dashboards:**

```yaml
apiVersion: 1
providers:
  - name: 'default'
    folder: 'General'
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

### 🔬 Практика: Grafana

```bash
# 1. Установить Grafana
helm install grafana grafana/grafana \
  --namespace monitoring \
  --create-namespace \
  --set adminPassword=admin123

# 2. Port-forward
kubectl port-forward -n monitoring svc/grafana 3000:80

# 3. Открыть http://localhost:3000
# Login: admin / admin123

# 4. Добавить Loki datasource
# Configuration → Data Sources → Add → Loki
# URL: http://loki.monitoring.svc.cluster.local:3100
# Save & Test

# 5. Explore → Loki
# {namespace="default"} |= "error"

# 6. Импортировать дашборд
# Dashboards → Import
# ID: 13639 (Loki Dashboard)
```

### 💡 Практика: как правильно использовать Grafana

**✅ ОБЯЗАТЕЛЬНО:**

1. **Grafana как единый UI.**
2. **Provisioning datasource** из файлов.
3. **Provisioning dashboards** из файлов.
4. **Correlations** между datasource.

**👍 СТОИТ:**

4. **Alerting в Grafana** для алертов.
5. **Folders** для организации dashboards.
6. **RBAC** для команд.

**❌ НЕ ДЕЛАЙ:**

7. **Не создавай dashboards вручную** в production. Provisioning.
8. **Не используй admin для всех.** RBAC.
9. **Не забывай про backup dashboards.**

### Где мы сейчас

Мы разобрали Grafana. Теперь — **log sampling и retention**.

---

## 21.9 Log sampling и retention

### 🔌 Проблема: логи растут бесконечно

Приложение пишет логи. 1000 RPS × 5 логов на запрос × 10 сервисов = 50 000 логов в секунду. **4 миллиарда в день.**

**Диск кончится через неделю.** А стоит это тысячи долларов в месяц.

**Решение:** sampling и retention.

### 📊 Log sampling

**Sampling** — сохранять не все логи, а выборку.

**Подходы:**

**1. Random sampling.**

Сохранять 1 из N логов случайно.

```ini
[FILTER]
    Name    sampling
    Match   kube.*
    Rate    100    # 1 из 100
```

**Плюсы:** просто, равномерно.
**Минусы:** теряются редкие события.

**2. Level-based sampling.**

Сохранять все ERROR, 1% INFO.

```ini
[FILTER]
    Name    grep
    Match   kube.*
    Exclude level info
```

**Плюсы:** важные логи сохраняются.
**Минусы:** теряется контекст.

**3. Adaptive sampling.**

Больше сэмплов при проблемах.

**4. Tail-based sampling.**

Решение о сэмплировании **после** запроса (по результату).

**Правило:** **всегда сохранять ERROR, WARN.** Sampling для INFO, DEBUG.

### 📊 Retention

**Retention** — сколько хранить логи.

**Политики:**

| Тип логов | Retention |
|:---|:---|
| **Debug** | 3-7 дней |
| **Application INFO** | 30 дней |
| **WARN/ERROR** | 90 дней |
| **Audit** | 1 год |
| **Compliance** | 7 лет |

**Разные retention** для разных типов:

```yaml
# Loki
limits_config:
  retention_period: 30d

# Для audit — отдельный tenant с retention 1y
```

### 🎯 Compression

**Сжатие** — обязательно.

**Форматы:**

- **gzip** — универсальный.
- **zstd** — быстрее, лучше сжатие.
- **snappy** — быстрый.
- **lz4** — очень быстрый.

**Экономия:** 10-20x от исходного размера.

### 🎯 Cost optimization

**Что влияет на стоимость:**

1. **Объём логов** (сколько пишется).
2. **Retention** (сколько хранится).
3. **Storage backend** (S3 дешевле SSD).
4. **Индексация** (Loki дешевле ES).

**Стратегии:**

- **Sampling** — меньше писать.
- **Retention** — меньше хранить.
- **Tiering** — старые в S3.
- **Compression** — сжатие.
- **Filter** — не писать debug в prod.

### 🔬 Практика: sampling и retention

**Fluent Bit sampling:**

```ini
[FILTER]
    Name    grep
    Match   kube.*
    Exclude level debug
```

**Fluent Bit с throttling:**

```ini
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Mem_Buf_Limit     50MB
    Skip_Long_Lines   On
    Rotate_Wait       30
    DB                /var/log/flb_kube.db
```

**Loki retention:**

```yaml
# values.yaml
loki:
  limits_config:
    retention_period: 720h    # 30 дней
  
  compactor:
    retention_enabled: true
    retention_delete_delay: 2h
```

### 💡 Практика: как правильно управлять объёмом

**✅ ОБЯЗАТЕЛЬНО:**

1. **Не логировать DEBUG в production.**
2. **Sampling для INFO.**
3. **Retention policies.**
4. **Compression.**

**👍 СТОИТ:**

4. **Разные retention** для разных типов.
5. **Tiering** в S3.
6. **Мониторинг объёма.**

**❌ НЕ ДЕЛАЙ:**

7. **Не храни логи вечно.**
8. **Не логируй всё без фильтров.**
9. **Не забывай про compression.**

### Где мы сейчас

Мы разобрали sampling и retention. Теперь — **корреляция через trace ID**.

---

## 21.10 Корреляция логов через trace ID### 🔌 Проблема: логи разных сервисов не связаны

Запрос проходит через 5 сервисов. В каждом — свои логи. Как связать?

**Решение:** trace ID / request ID.

### 📊 Что такое trace ID

**Trace ID** — уникальный идентификатор запроса, проходящего через все сервисы.

**Как работает:**

1. **Первый сервис** генерирует trace ID.
2. **Передаёт** его в заголовке (`X-Trace-ID`, `traceparent`).
3. **Следующий сервис** читает trace ID и добавляет в свои логи.
4. **Все логи** одного запроса имеют один trace ID.

**Пример:**

```
# API gateway
{"trace_id": "abc123", "service": "gateway", "msg": "Request received"}

# API service
{"trace_id": "abc123", "service": "api", "msg": "Processing order"}
{"trace_id": "abc123", "service": "api", "msg": "Database query"}

# Notification service
{"trace_id": "abc123", "service": "notification", "msg": "Sending email"}

# Database
{"trace_id": "abc123", "service": "postgres", "msg": "Query executed"}
```

**Поиск:** `trace_id=abc123` → все логи одного запроса.

### 🎯 W3C Trace Context

**W3C Trace Context** — стандарт для передачи trace ID.

**Заголовок `traceparent`:**

```
traceparent: 00-abc123def456-789ghi012-01
             │  │              │        │
             │  │              │        └── flags (01 = sampled)
             │  │              └── parent-id (span ID)
             │  └── trace-id (32 hex)
             └── version
```

**Что даёт:**

- **Стандарт.** Совместимость между инструментами.
- **Context propagation.** Автоматически через HTTP, gRPC.

### 🎯 Trace ID в Go

**Middleware для HTTP:**

```go
package main

import (
    "context"
    "log/slog"
    "net/http"
    
    "github.com/google/uuid"
)

type contextKey string

const traceIDKey contextKey = "trace_id"

func TraceMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Извлечь или сгенерировать trace ID
        traceID := r.Header.Get("X-Trace-ID")
        if traceID == "" {
            traceID = r.Header.Get("traceparent")
        }
        if traceID == "" {
            traceID = uuid.New().String()
        }
        
        // Добавить в контекст
        ctx := context.WithValue(r.Context(), traceIDKey, traceID)
        
        // Добавить в response
        w.Header().Set("X-Trace-ID", traceID)
        
        // Логгер с trace_id
        logger := slog.With("trace_id", traceID)
        
        logger.Info("Request received",
            "method", r.Method,
            "path", r.URL.Path,
        )
        
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func getTraceID(ctx context.Context) string {
    if id, ok := ctx.Value(traceIDKey).(string); ok {
        return id
    }
    return ""
}
```

**Передача в другой сервис:**

```go
func callService(ctx context.Context) error {
    req, _ := http.NewRequestWithContext(ctx, "GET", "http://other-service/api", nil)
    
    // Передать trace ID
    req.Header.Set("X-Trace-ID", getTraceID(ctx))
    
    resp, err := http.DefaultClient.Do(req)
    // ...
}
```

### 🎯 Корреляция в Loki

**Derived fields в Grafana:**

```yaml
# datasource Loki
jsonData:
  derivedFields:
    - datasourceUid: tempo
      matcherRegex: "trace_id=(\\w+)"
      name: TraceID
      url: "$${__value.raw}"
```

**Что произойдёт:** в логах появится ссылка на трейс в Tempo/Jaeger.

### 🎯 Корреляция между datasource

**Grafana Correlations:**

1. Source: Loki.
2. Target: Jaeger/Tempo.
3. Поле: `trace_id`.
4. Result: переход к трейсу.

**Что даёт:** клик по trace_id в логе → открыть трейс.

### 🔬 Практика: trace ID

```bash
# 1. Приложение с trace ID
cat > main.go <<'EOF'
package main

import (
    "log/slog"
    "net/http"
    "os"
    "github.com/google/uuid"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    slog.SetDefault(logger)
    
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        traceID := r.Header.Get("X-Trace-ID")
        if traceID == "" {
            traceID = uuid.New().String()
        }
        
        logger := slog.With("trace_id", traceID)
        logger.Info("Request received", "path", r.URL.Path)
        
        w.Header().Set("X-Trace-ID", traceID)
        w.Write([]byte("OK"))
    })
    
    http.ListenAndServe(":8080", nil)
}
EOF

go mod init demo
go mod tidy
go run main.go &

# 2. Запросы
curl -H "X-Trace-ID: test-trace-123" localhost:8080
curl localhost:8080    # сгенерирует новый trace_id

# 3. Логи
# {"time":"...","level":"INFO","msg":"Request received","trace_id":"test-trace-123","path":"/"}
# {"time":"...","level":"INFO","msg":"Request received","trace_id":"<uuid>","path":"/"}
```

### 💡 Практика: как правильно коррелировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Trace ID** в каждом запросе.
2. **Передача** через headers (`X-Trace-ID`, `traceparent`).
3. **Логгер с trace_id** во всех сервисах.
4. **Derived fields** в Grafana.

**👍 СТОИТ:**

4. **W3C Trace Context** для стандарта.
5. **OpenTelemetry** для автоматизации.
6. **Correlation** между Loki и Jaeger.

**❌ НЕ ДЕЛАЙ:**

7. **Не генерируй новый trace ID** в каждом сервисе. Передавай.
8. **Не забывай про propagation** через gRPC, Kafka.
9. **Не логируй без trace_id.**

### Где мы сейчас

Мы разобрали корреляцию. Теперь — **алерты на логи**.

---

## 21.11 Алерты на логи

### 🔌 Проблема: как узнать о проблеме из логов

Метрики показывают «что-то не так». Но некоторые проблемы видны только в логах:

- **`OutOfMemoryError`** — но метрики нормальные.
- **`Authentication failed`** — но error rate низкий.
- **`Deadlock detected`** — но latency в норме.

**Решение:** алерты на логи.

### 📊 Алерты в Loki

**LogQL для алертов:**

```logql
# Много ошибок
sum(rate({namespace="production"} |= "level=error" [5m])) > 10

# Конкретная ошибка
count_over_time({namespace="production"} |= "OutOfMemoryError" [5m]) > 0

# Медленные запросы
count_over_time({namespace="production"} | json | duration_ms > 5000 [5m]) > 10
```

### 🎯 Alerting Rules в Loki

**Loki Ruler:**

```yaml
groups:
  - name: application
    interval: 1m
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate({namespace="production"} |= "level=error" [5m])) > 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate in production"
          description: "Error rate is {{ $value }} errors/sec"
      
      - alert: OutOfMemory
        expr: |
          count_over_time({namespace="production"} |= "OutOfMemoryError" [5m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Out of memory detected"
      
      - alert: AuthenticationFailures
        expr: |
          sum(rate({namespace="production"} |= "authentication failed" [5m])) > 100
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High authentication failure rate"
```

### 🎯 Alerting в Grafana

**Grafana Alerting** — универсальный для всех datasource.

**Alert Rule:**

```yaml
apiVersion: 1
groups:
  - orgId: 1
    name: application-alerts
    folder: Alerts
    interval: 1m
    rules:
      - uid: high-error-rate
        title: High Error Rate
        condition: A
        data:
          - refId: A
            datasourceUid: loki
            model:
              expr: |
                sum(rate({namespace="production"} |= "level=error" [5m]))
              queryType: range
        noDataState: NoData
        execErrState: Error
        for: 5m
        annotations:
          summary: "Error rate > 10/sec"
        labels:
          severity: critical
```

### 🎯 Алгоритмы алертов

**1. Threshold.**

Простой порог.

```logql
count_over_time({app="api"} |= "error" [5m]) > 10
```

**2. Rate.**

Скорость.

```logql
sum(rate({app="api"} |= "error" [5m])) > 1
```

**3. Absence.**

Отсутствие ожидаемого лога.

```logql
# Нет "success" логов за 10 минут
absent_over_time({app="api"} |= "success" [10m])
```

**4. Anomaly.**

Аномалии.

```logql
# Error rate выше среднего за 7 дней
sum(rate({app="api"} |= "error" [5m])) > 
  avg_over_time(sum(rate({app="api"} |= "error" [5m]))[7d:1h]) * 3
```

### 🎯 Notification channels

**Grafana Notification Policies:**

```yaml
apiVersion: 1
policies:
  - orgId: 1
    receiver: slack-default
    routes:
      - receiver: slack-critical
        matchers:
          - severity = critical
        continue: false
      - receiver: slack-warning
        matchers:
          - severity = warning

receivers:
  - name: slack-critical
    slack_configs:
      - api_url: https://hooks.slack.com/...
        channel: '#alerts-critical'
        title: '{{ .CommonLabels.alertname }}'
        text: '{{ .CommonAnnotations.summary }}'
  
  - name: slack-warning
    slack_configs:
      - api_url: https://hooks.slack.com/...
        channel: '#alerts-warning'
```

### 🎯 Best practices для алертов

**1. Только actionable.**

Алерт должен требовать действия. Если можно игнорировать — не алерт.

**2. Severity.**

Разные уровни: critical, warning, info.

**3. Runbook.**

К каждому алерту — что делать.

**4. Context.**

Добавлять ссылки на дашборды, логи.

**5. Suppression.**

Подавлять дубликаты.

**6. Grouping.**

Группировать связанные алерты.

### 🔬 Практика: алерты

```yaml
# Loki alerting rule
groups:
  - name: api-alerts
    interval: 1m
    rules:
      - alert: ApiErrorRateHigh
        expr: |
          sum(rate({namespace="production", app="api"} |= "level=error" [5m])) > 5
        for: 5m
        labels:
          severity: critical
          service: api
        annotations:
          summary: "API error rate > 5/sec"
          runbook: "https://wiki.example.com/runbooks/api-errors"
      
      - alert: DatabaseConnectionFailed
        expr: |
          count_over_time({namespace="production"} |= "connection refused" [5m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Database connection failed"
```

**Grafana Alerting:**

1. Alerting → Alert rules → New alert rule.
2. Query: Loki, `{app="api"} |= "error"`.
3. Condition: threshold > 10.
4. Evaluation: every 1m, for 5m.
5. Notifications: Slack.

### 💡 Практика: как правильно настраивать алерты

**✅ ОБЯЗАТЕЛЬНО:**

1. **Alerting Rules** для критичных ошибок.
2. **Severity levels.**
3. **Notification channels** (Slack, PagerDuty).
4. **Runbook** к каждому алерту.

**👍 СТОИТ:**

4. **Anomaly detection** для неожиданных изменений.
5. **Suppression** для дубликатов.
6. **Testing alerts** регулярно.

**❌ НЕ ДЕЛАЙ:**

7. **Не алерти на всё.** Только actionable.
8. **Не игнорируй алерты.** Если игнорируешь — удаляй.
9. **Не забывай про severity.**

### Где мы сейчас

Мы разобрали алерты. Теперь — **безопасность логов**.

---

## 21.12 Безопасность логов

### 🔌 Проблема: логи содержат секреты

Приложение логирует:

```json
{"level": "info", "msg": "Login attempt", "user": "alice", "password": "SuperSecret123"}
```

**Пароль в логах.** Логи хранятся годами. Доступ у многих.

**Компрометация.**

### 📊 Что нельзя логировать

**❌ Никогда:**

- Пароли.
- API-ключи.
- Токены (JWT, OAuth).
- Приватные ключи.
- Номера кредитных карт.
- CVV.
- PII (персональные данные) — в зависимости от compliance.
- Номера паспортов.
- Медицинские данные (HIPAA).

**⚠️ Осторожно:**

- Email, телефоны.
- IP-адреса (GDPR).
- User ID.

### 🎯 Как защитить

**1. Не логируй секреты.**

Правило: **если не уверен — не логируй**.

**2. Маскирование.**

```go
func maskPassword(password string) string {
    if len(password) < 4 {
        return "***"
    }
    return password[:2] + strings.Repeat("*", len(password)-4) + password[len(password)-2:]
}

logger.Info("Login attempt", "user", user, "password", maskPassword(password))
// password=Su********23
```

**3. Структурированные логи.**

С JSON проще применять фильтры.

**4. Redaction в pipeline.**

Fluent Bit filter:

```ini
[FILTER]
    Name    modify
    Match   kube.*
    Rename  password password_redacted
    
[FILTER]
    Name    lua
    Match   kube.*
    script  redact.lua
    call    redact
```

**redact.lua:**

```lua
function redact(tag, timestamp, record)
    local sensitive = {"password", "token", "secret", "api_key", "authorization"}
    for _, field in ipairs(sensitive) do
        if record[field] then
            record[field] = "[REDACTED]"
        end
    end
    return 1, timestamp, record
end
```

**5. Доступ к логам.**

- **RBAC** — кто может читать логи.
- **Шифрование** — at rest и in transit.
- **Audit** — кто читал логи.

**6. Retention для чувствительных.**

- **Короткий retention** для логов с PII.
- **Удаление** по требованию (GDPR right to be forgotten).

### 🎯 GDPR и логи

**GDPR требует:**

- **Право на забвение** — удаление PII по запросу.
- **Минимизация** — не собирать больше, чем нужно.
- **Согласие** — на обработку PII.

**Что делать:**

- **Не логируй PII** без необходимости.
- **Анонимизируй** где возможно.
- **Retention policies** для PII.
- **Возможность удаления** по запросу.

### 🎯 Аудит логов

**Audit logs** — кто, что, когда делал.

**Что включать:**

- **Kubernetes audit logs** — API-запросы.
- **Application audit logs** — бизнес-операции.
- **Access logs** — кто читал логи.

**Kubernetes audit:**

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    resources:
      - group: ''
        resources: ['secrets', 'configmaps']
  
  - level: Metadata
    resources:
      - group: ''
        resources: ['pods', 'services']
```

### 🔬 Практика: маскирование

```go
package main

import (
    "log/slog"
    "os"
    "regexp"
    "strings"
)

var sensitivePatterns = []*regexp.Regexp{
    regexp.MustCompile(`(?i)password[=:]\s*\S+`),
    regexp.MustCompile(`(?i)token[=:]\s*\S+`),
    regexp.MustCompile(`(?i)api[_-]?key[=:]\s*\S+`),
    regexp.MustCompile(`(?i)authorization:\s*\S+`),
}

func redact(s string) string {
    for _, p := range sensitivePatterns {
        s = p.ReplaceAllStringFunc(s, func(match string) string {
            parts := strings.SplitN(match, "=", 2)
            if len(parts) == 2 {
                return parts[0] + "=[REDACTED]"
            }
            return "[REDACTED]"
        })
    }
    return s
}

type redactingHandler struct {
    inner slog.Handler
}

func (h *redactingHandler) Enabled(ctx context.Context, level slog.Level) bool {
    return h.inner.Enabled(ctx, level)
}

func (h *redactingHandler) Handle(ctx context.Context, r slog.Record) error {
    r.Message = redact(r.Message)
    return h.inner.Handle(ctx, r)
}

func (h *redactingHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
    return &redactingHandler{inner: h.inner.WithAttrs(attrs)}
}

func (h *redactingHandler) WithGroup(name string) slog.Handler {
    return &redactingHandler{inner: h.inner.WithGroup(name)}
}

func main() {
    handler := &redactingHandler{
        inner: slog.NewJSONHandler(os.Stdout, nil),
    }
    logger := slog.New(handler)
    slog.SetDefault(logger)
    
    logger.Info("Login", "user", "alice", "password", "SuperSecret123")
    // {"msg":"Login","user":"alice","password":"[REDACTED]"}
}
```

### 💡 Практика: как правильно защищать логи

**✅ ОБЯЗАТЕЛЬНО:**

1. **Не логируй секреты.** Никогда.
2. **Маскирование** в коде и pipeline.
3. **RBAC** для доступа к логам.
4. **Retention policies.**

**👍 СТОИТ:**

4. **Redaction в Fluent Bit / Loki.**
5. **Audit logs.**
6. **Encryption at rest.**

**❌ НЕ ДЕЛАЙ:**

7. **Не логируй PII без необходимости.**
8. **Не давай всем доступ к логам.**
9. **Не храни audit логи вместе с application.**

### Где мы сейчас

Мы разобрали безопасность. Теперь — **диагностика**.

---

## 21.13 Диагностика проблем

### 🔌 Проблема: логи не там, где нужно

Настроил сбор логов. Но логи не появляются в Loki. Или появляются, но неправильные.

### 🔍 Типичные проблемы

**1. Логи не появляются в Loki.**

**Причины:**

- Fluent Bit не работает.
- Неправильный output.
- Loki недоступен.
- Сетевые проблемы.

**Диагностика:**

```bash
# Fluent Bit работает?
kubectl get pods -n logging -l app.kubernetes.io/name=fluent-bit

# Логи Fluent Bit
kubectl logs -n logging daemonset/fluent-bit --tail=100

# Loki работает?
kubectl get pods -n monitoring -l app=loki

# Loki принимает?
kubectl logs -n monitoring loki-0 --tail=100 | grep -i "error\|received"
```

**2. Логи не структурированы.**

**Причины:**

- Приложение пишет plain text.
- Неправильный parser.
- JSON не парсится.

**Диагностика:**

```bash
# Посмотреть сырые логи
kubectl logs api-xxx -n production --tail=10

# Проверить parser в Fluent Bit
kubectl logs -n logging daemonset/fluent-bit | grep -i "parser\|error"
```

**3. Metadata отсутствует.**

**Причины:**

- Kubernetes filter не работает.
- Нет прав у ServiceAccount.
- Неправильный Kube_URL.

**Диагностика:**

```bash
# Проверить ServiceAccount
kubectl get sa -n logging fluent-bit

# Проверить RBAC
kubectl auth can-i list pods --as=system:serviceaccount:logging:fluent-bit
kubectl auth can-i get pods --as=system:serviceaccount:logging:fluent-bit
```

**4. Логи теряются.**

**Причины:**

- Перегрузка Fluent Bit (Mem_Buf_Limit).
- Loki недоступен.
- Retention удалил.

**Диагностика:**

```bash
# Метрики Fluent Bit
kubectl port-forward -n logging daemonset/fluent-bit 2020:2020
curl localhost:2020/api/v1/metrics

# Потери
curl localhost:2020/api/v1/metrics | grep -i "lost\|retried"
```

**5. Медленные запросы в Loki.**

**Причины:**

- Слишком широкий диапазон.
- Много labels.
- Нет фильтров.

**Решение:**

```logql
# ❌ Плохо: без фильтра
{namespace="production"}

# ✅ Хорошо: с фильтром
{namespace="production", app="api"} |= "error"
```

**6. Cardnality problem.**

**Причины:**

- Label с высокой cardinality (user_id, request_id).
- Слишком много labels.

**Решение:** не использовать high-cardinality поля как labels.

**7. Fluent Bit падает с OOM.**

**Причины:**

- Большие логи.
- Нет Mem_Buf_Limit.
- Skip_Long_Lines не включён.

**Решение:**

```ini
[INPUT]
    Name              tail
    Mem_Buf_Limit     50MB
    Skip_Long_Lines   On
```

### 🎯 Полезные запросы

**Loki:**

```logql
# Все логи из namespace
{namespace="production"}

# Ошибки
{namespace="production"} |= "error"

# JSON парсинг
{namespace="production"} | json | level="error"

# Метрики из логов
sum(rate({namespace="production"} |= "error" [5m]))
```

**Fluent Bit metrics:**

```bash
kubectl port-forward -n logging daemonset/fluent-bit 2020:2020
curl localhost:2020/api/v1/metrics
```

### 🎯 Debug logging

**Включить debug в Fluent Bit:**

```ini
[SERVICE]
    Log_Level debug
```

**Проверить, что Fluent Bit читает:**

```bash
kubectl exec -n logging fluent-bit-xxx -- ls /var/log/containers/
kubectl exec -n logging fluent-bit-xxx -- cat /var/log/containers/api-xxx.log | head
```

### 🔬 Практика: диагностика

```bash
# 1. Проверить всю цепочку
kubectl get pods -n logging
kubectl get pods -n monitoring

# 2. Fluent Bit логи
kubectl logs -n logging daemonset/fluent-bit --tail=50

# 3. Loki логи
kubectl logs -n monitoring loki-0 --tail=50

# 4. Запрос в Loki
kubectl port-forward -n monitoring svc/loki 3100:3100
curl -G -s "http://localhost:3100/loki/api/v1/query" \
  --data-urlencode 'query={namespace="default"}' | jq

# 5. Проверить metadata
curl -G -s "http://localhost:3100/loki/api/v1/labels" | jq

# 6. Проверить parsing
kubectl logs -n logging daemonset/fluent-bit | grep -i "parser"
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Проверить всю цепочку** (Pod → Fluent Bit → Loki → Grafana).
2. **Логи Fluent Bit** — главный источник.
3. **Метрики Fluent Bit** — потери, retries.

**👍 СТОИТ:**

4. **Debug logging** при проблемах.
5. **Проверять raw logs** в `/var/log/containers/`.
6. **Проверять RBAC.**

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй потери.** Lost logs — критично.
8. **Не используй high-cardinality labels.**
9. **Не забывай про Mem_Buf_Limit.**

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **Observability** | Способность задавать вопросы о системе. |
| **Monitoring** | Наблюдение за известными метриками. |
| **Logs** | Текстовые записи о событиях. |
| **Metrics** | Числовые значения во времени. |
| **Traces** | Путь запроса через сервисы. |
| **Структурированные логи** | Логи в формате key-value (JSON). |
| **stdout/stderr** | Стандартные потоки вывода. |
| **Fluent Bit** | Лёгкий агент для сбора логов. |
| **Fluentd** | Агент для сбора логов (Ruby). |
| **Vector** | Агент для сбора (Rust). |
| **Loki** | Backend для логов от Grafana. |
| **LogQL** | Язык запросов Loki. |
| **Elasticsearch** | Поисковый движок. |
| **OpenSearch** | Форк Elasticsearch. |
| **Kibana** | UI для Elasticsearch. |
| **Grafana** | UI для визуализации. |
| **ILM** | Index Lifecycle Management. |
| **Trace ID** | Уникальный ID запроса. |
| **W3C Trace Context** | Стандарт для trace ID. |
| **Retention** | Сколько хранить логи. |
| **Sampling** | Сохранять не все логи. |
| **Cardinality** | Количество уникальных значений. |
| **Redaction** | Маскирование секретов в логах. |
| **Audit logs** | Логи доступа и операций. |

---

## Что мы узнали?

- **Observability** — способность задавать вопросы. **Monitoring** — наблюдение за известными метриками.
- **Три столпа:** logs, metrics, traces.
- **Структурированные логи** (JSON) — стандарт. `log/slog`, `zerolog`, `zap`.
- **Сбор логов в K8s:** node-level agent (Fluent Bit) + централизованный backend.
- **Fluent Bit:** лёгкий, DaemonSet, CRI-парсер, kubernetes filter.
- **Loki:** логи в стиле Prometheus. Labels, LogQL, дешёвый.
- **Elasticsearch:** полнотекстовый поиск, мощный, дорогой.
- **Grafana:** единый UI для всех datasource.
- **Sampling и retention:** для управления объёмом.
- **Trace ID:** корреляция между сервисами.
- **Алерты на логи:** Loki Ruler, Grafana Alerting.
- **Безопасность:** не логировать секреты, redaction, RBAC.
- **Диагностика:** проверять всю цепочку, метрики Fluent Bit.

---

## Типичные ошибки

- ❌ **Логировать в файлы** внутри контейнера. Только stdout/stderr.
- ❌ **Неструктурированные логи** в production.
- ❌ **Логировать секреты.** Пароли, токены, PII.
- ❌ **Использовать high-cardinality labels** (user_id, request_id) в Loki.
- ❌ **Не настраивать retention.** Диск кончится.
- ❌ **Хранить debug логи** в production.
- ❌ **Не использовать trace ID.**
- ❌ **Забывать про Mem_Buf_Limit** в Fluent Bit.
- ❌ **Не фильтровать debug** при сборе.
- ❌ **Алертить на всё.** Только actionable.
- ❌ **Не тестировать DR** для логов.
- ❌ **Не проверять RBAC** для Fluent Bit.
- ❌ **Не использовать compression.**
- ❌ **Игнорировать потери логов.**

---

## Для быстрого повторения

- **Observability:** logs + metrics + traces.
- **Структурированные логи:** JSON, `log/slog`, `zerolog`, `zap`.
- **Сбор:** Fluent Bit DaemonSet, CRI parser, kubernetes filter.
- **Backend:** Loki (лёгкий), Elasticsearch (мощный).
- **Loki:** LogQL, labels, retention.
- **Grafana:** единый UI, datasources, dashboards.
- **Retention:** разные для разных типов.
- **Sampling:** не всё логировать.
- **Trace ID:** `X-Trace-ID`, `traceparent`, W3C.
- **Алерты:** Loki Ruler, Grafana Alerting.
- **Безопасность:** не логировать секреты, redaction, RBAC.
- **Диагностика:** логи Fluent Bit, метрики, RBAC.

---

## Вопросы для самопроверки

1. Что такое observability? Чем отличается от monitoring?
2. Три столпа observability — назови и объясни.
3. Что такое структурированные логи? Зачем нужны?
4. Как собирать логи в Kubernetes? Три подхода.
5. Что такое Fluent Bit? Как работает?
6. Что такое Loki? Чем отличается от Elasticsearch?
7. Что такое LogQL? Приведи пример.
8. Как связать логи разных сервисов через trace ID?
9. Как настраивать алерты на логи?
10. Что нельзя логировать? Как защитить?
11. Что такое retention? Как настраивать?
12. Что такое cardinality? Почему важна?
13. Логи не появляются в Loki. Как диагностировать?
14. Что такое CRI-формат логов?
15. Как коррелировать логи с трейсами в Grafana?

---

## Ответы

**1. Observability vs Monitoring**

Monitoring — наблюдение за известными метриками. Observability — способность задавать любые вопросы о системе. Monitoring отвечает «что сломано», observability — «почему».

**2. Три столпа**

- **Logs** — что произошло.
- **Metrics** — сколько.
- **Traces** — где.

**3. Структурированные логи**

JSON с key-value. Легко парсить, искать, агрегировать. Стандарт для production.

**4. Сбор логов в K8s**

- **Node-level agent** (Fluent Bit DaemonSet) — рекомендуется.
- **Sidecar** — для специфичных случаев.
- **Push из приложения** — прямой контроль, но изменения в коде.

**5. Fluent Bit**

Лёгкий агент на C. DaemonSet на каждой ноде. Input → Parser → Filter → Output. CRI parser, kubernetes filter.

**6. Loki vs Elasticsearch**

Loki: индексирует только labels, лёгкий, дешёвый. ES: полнотекстовая индексация, мощный, дорогой. Loki для большинства, ES — если нужен полнотекстовый поиск.

**7. LogQL**

```logql
{namespace="production", app="api"} |= "error" | json | level="error"
```

**8. Trace ID**

Первый сервис генерирует trace ID. Передаёт через `X-Trace-ID` или `traceparent`. Все сервисы добавляют в логи. Поиск по trace_id → все логи запроса.

**9. Алерты на логи**

Loki Ruler или Grafana Alerting. LogQL-запрос с порогом. `sum(rate({namespace="production"} |= "level=error" [5m])) > 10`.

**10. Что нельзя логировать**

Пароли, токены, ключи, PII, номера карт. Защита: не логировать, маскирование, redaction в pipeline, RBAC.

**11. Retention**

Сколько хранить логи. Debug — 3-7 дней, application — 30 дней, audit — 1 год. Настраивается в Loki (retention_period), ES (ILM).

**12. Cardinality**

Количество уникальных значений label. Высокая cardinality (user_id, request_id) → много данных → проблемы. Не использовать как labels.

**13. Диагностика Loki**

1. Fluent Bit работает? `kubectl get pods -n logging`.
2. Логи Fluent Bit: `kubectl logs -n logging daemonset/fluent-bit`.
3. Метрики Fluent Bit: `curl localhost:2020/api/v1/metrics`.
4. Проверить RBAC.
5. Проверить output config.

**14. CRI-формат**

containerd пишет логи в формате:
```
2026-01-15T09:00:12.345Z stdout F {"level":"info","msg":"..."}
```
Timestamp, stream, tag, log. Парсится CRI parser.

**15. Корреляция с трейсами**

Grafana derived fields в Loki datasource. `matcherRegex: "trace_id=(\\w+)"`. Поле становится ссылкой на трейс в Jaeger/Tempo.

---

## Куда идти дальше?

Мы разобрали логирование — первый столп observability. Теперь ты знаешь:

- Что такое observability.
- Структурированные логи.
- Сбор логов в Kubernetes.
- Fluent Bit.
- Loki и Elasticsearch.
- Grafana.
- Sampling и retention.
- Корреляция через trace ID.
- Алерты на логи.
- Безопасность.
- Диагностика.

Следующая глава по оглавлению — **Глава 22: Observability — мониторинг и алертинг**.

Скажи «дальше» — и я отправлю Главу 22.