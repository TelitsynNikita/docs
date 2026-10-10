# 🌐 Глава 34: HTTP-сервер

**Что вы узнаете:**
- Как построить **production-ready HTTP-сервер** на Go.
- Как ограничить **параллелизм** запросов через семафор.
- Как защитить сервер от **DDoS** через rate limiter.
- Как обрабатывать **таймауты** (Read, Write, Idle).
- Как добавить **health checks** (`/healthz`, `/readyz`).
- Как реализовать **middleware** для cross-cutting concerns.
- Как корректно завершить сервер через **graceful shutdown** (Глава 26).
- Как **мониторить** сервер через метрики и логи.
- Как избежать типичных ошибок в production.

**После прочтения вы сможете:**
- Построить HTTP-сервер с ограничением параллелизма.
- Настроить таймауты для защиты от медленных клиентов.
- Реализовать middleware (rate limiting, семафор, логирование, метрики).
- Корректно завершать сервер по `SIGTERM`.
- Диагностировать проблемы через pprof.
- Понимать, где HTTP-сервер уместен, а где — нет.

---

## Содержание

- [34.0 Пролог: сервис, который не выжил под нагрузкой](#340-пролог-сервис-который-не-выжил-под-нагрузкой)
- [34.1 Базовая структура](#341-базовая-структура)
- [34.2 Ограничение параллелизма через семафор](#342-ограничение-параллелизма-через-семафор)
- [34.3 Rate limiter для защиты от DDoS](#343-rate-limiter-для-защиты-от-ddos)
- [34.4 Таймауты: Read, Write, Idle](#344-таймауты-read-write-idle)
- [34.5 Middleware: логирование, метрики, recovery](#345-middleware-логирование-метрики-recovery)
- [34.6 Health checks: /healthz и /readyz](#346-health-checks-healthz-и-readyz)
- [34.7 Graceful shutdown](#347-graceful-shutdown)
- [34.8 Мониторинг и pprof](#348-мониторинг-и-pprof)
- [34.9 Полный production-ready сервер](#349-полный-production-ready-сервер)
- [34.10 Выводы и типичные ошибки](#3410-выводы-и-типичные-ошибки)
- [34.11 Для быстрого повторения](#3411-для-быстрого-повторения)
- [34.12 Вопросы для самопроверки](#3412-вопросы-для-самопроверки)
- [34.13 Ответы](#3413-ответы)
- [34.14 Куда идти дальше?](#3414-куда-идти-дальше)
- [34.15 Чек-лист](#3415-чек-лист)

---

## 34.0 Пролог: сервис, который не выжил под нагрузкой

У нас есть HTTP-сервис на Go. Работает в Kubernetes. Всё хорошо, пока не приходит **рекламная кампания** — 10 000 одновременных пользователей.

За минуту сервис падает. Что произошло?

Смотрим в код:

```go
func main() {
    http.HandleFunc("/api/users", handleUsers)
    http.HandleFunc("/api/orders", handleOrders)
    http.ListenAndServe(":8080", nil)
}

func handleUsers(w http.ResponseWriter, r *http.Request) {
    users, err := db.Query("SELECT * FROM users")  // 100 мс
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    json.NewEncoder(w).Encode(users)
}
```

**Что происходит при 10 000 одновременных запросах:**

- 10 000 горутин (по одной на запрос).
- 10 000 соединений к БД. **БД захлёбывается.**
- Память **растёт**.
- Через минуту — **OOM** или БД-падение.

**Проблемы:**

1. **Нет ограничения параллелизма.** 10 000 запросов обрабатываются одновременно.
2. **Нет таймаутов.** Если клиент медленный — горутина висит.
3. **Нет rate limiter.** Один клиент может забить сервер.
4. **Нет health checks.** Kubernetes не знает, жив ли сервис.
5. **Нет graceful shutdown.** При деплое обрываются запросы.

**Что хочется:** production-ready HTTP-сервер с ограничениями, таймаутами, health checks и корректным завершением.

> **Мост к следующим главам:** HTTP-сервер — синтез всех паттернов: семафор (Глава 12), rate limiter (Глава 14), context (Глава 5), graceful shutdown (Глава 26), bulkhead (Глава 22). Понимание HTTP-сервера даёт понимание, **как применять паттерны вместе**.

---

## 34.1 Базовая структура

Начнём с простейшего сервера.

### Код

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/api/users", handleUsers)
    mux.HandleFunc("/api/orders", handleOrders)

    srv := &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }

    log.Println("starting on :8080")
    if err := srv.ListenAndServe(); err != nil {
        log.Fatal(err)
    }
}

func handleUsers(w http.ResponseWriter, r *http.Request) {
    users := []string{"alice", "bob"}
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func handleOrders(w http.ResponseWriter, r *http.Request) {
    orders := []string{"order1", "order2"}
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(orders)
}
```

**Что есть:**

- `http.Server` — сервер.
- `mux` — роутер.
- Обработчики.

**Что отсутствует:**

- Таймауты.
- Graceful shutdown.
- Ограничение параллелизма.
- Health checks.
- Логирование.
- Метрики.

### Структура для production

Для production лучше **инкапсулировать** сервер в структуру:

```go
type Server struct {
    httpSrv *http.Server
    mux     *http.ServeMux
}

func NewServer(addr string) *Server {
    s := &Server{
        mux: http.NewServeMux(),
    }

    s.mux.HandleFunc("/api/users", s.handleUsers)
    s.mux.HandleFunc("/api/orders", s.handleOrders)

    s.httpSrv = &http.Server{
        Addr:    addr,
        Handler: s.mux,
    }

    return s
}
```

**Что даёт:**

- **Инкапсуляция.** Сервер — одна структура.
- **Легко добавить middleware.**
- **Легко тестировать.**

### Запуск

```go
func (s *Server) Run(ctx context.Context) error {
    errCh := make(chan error, 1)

    go func() {
        if err := s.httpSrv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            errCh <- err
        }
        close(errCh)
    }()

    select {
    case <-ctx.Done():
        return s.shutdown()
    case err := <-errCh:
        return err
    }
}
```

### 💡 Практика: как начать

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Инкапсулируй сервер** в структуру.
2. **`http.Server` с `Handler`.**
3. **`ListenAndServe` в горутине.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **`NewServer(addr)`** — конструктор.
5. **`Run(ctx)`** — для запуска.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `log.Fatal` в Run.** Возвращай ошибку.
7. **Не игнорируй `http.ErrServerClosed`.**

---

## 34.2 Ограничение параллелизма через семафор

Ограничим число одновременных запросов.

### Проблема

Без ограничения **10 000 запросов** = 10 000 горутин = 10 000 соединений к БД. **БД захлёбывается.**

### Решение: семафор

**Семафор** (Глава 12) — ограничивает число одновременных запросов.

```go
type Server struct {
    httpSrv *http.Server
    mux     *http.ServeMux
    sem     chan struct{}  // семафор
}

func NewServer(addr string, maxConcurrent int) *Server {
    s := &Server{
        mux: http.NewServeMux(),
        sem: make(chan struct{}, maxConcurrent),
    }

    s.mux.HandleFunc("/api/users", s.handleUsers)
    s.mux.HandleFunc("/api/orders", s.handleOrders)

    s.httpSrv = &http.Server{
        Addr:    addr,
        Handler: s.semaphoreMiddleware(s.mux),
    }

    return s
}

func (s *Server) semaphoreMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        select {
        case s.sem <- struct{}{}:
            defer func() { <-s.sem }()
        case <-r.Context().Done():
            return
        case <-time.After(1 * time.Second):
            http.Error(w, "server busy", http.StatusServiceUnavailable)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

**Что происходит:**

1. Middleware захватывает слот семафора.
2. Если все слоты заняты — ждём 1 секунду.
3. Если за секунду не освободилось — **503 Service Unavailable**.
4. Обработчик выполняется.
5. Слот освобождается.

### Схема

```
Запрос 1 → sem ← захватить
Запрос 2 → sem ← захватить
...
Запрос N → sem ← захватить (лимит)
Запрос N+1 → sem ← ждёт
Запрос N+2 → sem ← ждёт
...

Если через 1 секунду слот не освободился:
  → 503 Service Unavailable
```

### Как выбрать размер семафора

**Рекомендации:**

- **CPU-bound:** `GOMAXPROCS`.
- **I/O-bound:** 10–100.
- **С БД:** размер пула БД + запас.

**Пример:**

- БД выдерживает 50 соединений.
- Семафор — 50.
- При 100 запросах — 50 обрабатываются, 50 ждут.

### 💡 Практика: как ограничить параллелизм

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Семафор** для ограничения.
2. **`select` с `ctx.Done()`** — для отмены.
3. **503 Service Unavailable** при переполнении.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Таймаут** на ожидание слота (1 сек).
5. **Метрики** — сколько запросов отвергнуто.

**❌ НЕ ДЕЛАЙ:**

6. **Не делай семафор бесконечным.**
7. **Не блокируй без таймаута.**

---

## 34.3 Rate limiter для защиты от DDoS

Один клиент может **забить** сервер. Защитимся rate limiter'ом.

### Проблема

Злоумышленник может сделать **10 000 запросов** с одного IP. Семафор не поможет — все запросы от одного клиента.

### Решение: rate limiter

**Rate limiter** (Глава 14) — ограничивает **скорость** запросов.

```go
import "golang.org/x/time/rate"

type Server struct {
    httpSrv *http.Server
    mux     *http.ServeMux
    sem     chan struct{}
    limiter *rate.Limiter  // rate limiter
}

func NewServer(addr string, maxConcurrent, rps int) *Server {
    s := &Server{
        mux:     http.NewServeMux(),
        sem:     make(chan struct{}, maxConcurrent),
        limiter: rate.NewLimiter(rate.Limit(rps), rps*2),
    }
    // ...
    return s
}

func (s *Server) rateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if !s.limiter.Allow() {
            http.Error(w, "too many requests", http.StatusTooManyRequests)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

**Что происходит:**

- Каждый запрос проверяет rate limiter.
- Если превышен лимит — **429 Too Many Requests**.
- Иначе — обработка.

### Rate limiter на клиента

**Для более точного контроля** — rate limiter **на каждого клиента**:

```go
type Server struct {
    // ...
    limiters sync.Map  // map[string]*rate.Limiter
}

func (s *Server) getLimiter(ip string) *rate.Limiter {
    if l, ok := s.limiters.Load(ip); ok {
        return l.(*rate.Limiter)
    }

    limiter := rate.NewLimiter(rate.Limit(10), 20)  // 10/сек на клиента
    s.limiters.Store(ip, limiter)
    return limiter
}

func (s *Server) rateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ip := getClientIP(r)
        limiter := s.getLimiter(ip)

        if !limiter.Allow() {
            http.Error(w, "too many requests", http.StatusTooManyRequests)
            return
        }
        next.ServeHTTP(w, r)
    })
}

func getClientIP(r *http.Request) string {
    if xff := r.Header.Get("X-Forwarded-For"); xff != "" {
        return strings.Split(xff, ",")[0]
    }
    if xri := r.Header.Get("X-Real-IP"); xri != "" {
        return xri
    }
    host, _, _ := net.SplitHostPort(r.RemoteAddr)
    return host
}
```

**Что даёт:**

- **Каждый клиент** имеет свой лимит.
- Один клиент не забивает сервер.

### Схема

```
Клиент A: 10 запросов/сек ────► ok
Клиент B: 100 запросов/сек ───► 429 (превышен)
Клиент C: 10 запросов/сек ────► ok

Сервер защищён от DDoS.
```

### Глобальный + на клиента

**Комбинация:**

```go
// Глобальный rate limiter (защита сервера)
globalLimiter := rate.NewLimiter(rate.Limit(10000), 20000)

// На клиента (защита от DDoS)
perClientLimiter := getLimiter(ip)
```

**Что даёт:**

- **Глобальный** — защищает от общей перегрузки.
- **На клиента** — защищает от одного злоумышленника.

### 💡 Практика: как защититься от DDoS

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Глобальный rate limiter.**
2. **На клиента** — по IP.
3. **429 Too Many Requests** при превышении.

**👍 СТОИТ СДЕЛАТЬ:**

4. **`X-Forwarded-For`** — если за прокси.
5. **Метрики** — сколько запросов отвергнуто.

**❌ НЕ ДЕЛАЙ:**

6. **Не блокируй запросы без rate limiter.**
7. **Не игнорируй X-Forwarded-For.**

---

## 34.4 Таймауты: Read, Write, Idle

Таймауты — **критичны** для production.

### Проблема

**Что если клиент медленный?**

- Открывает соединение и **не отправляет запрос**.
- Горутина **висит**.
- Через минуту — **10 000 висящих соединений**.
- **Сервер падает.**

### Решение: таймауты

```go
srv := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  60 * time.Second,
}
```

### Три типа таймаутов

**1. `ReadTimeout`.**

- Время на **чтение** всего запроса (заголовки + тело).
- 5 секунд — разумно.
- Защита от **медленных клиентов**.

**2. `WriteTimeout`.**

- Время на **запись** ответа.
- 10 секунд — разумно.
- Защита от **медленных клиентов**.

**3. `IdleTimeout`.**

- Время на **ожидание** следующего запроса.
- 60 секунд — разумно.
- Для **keep-alive** соединений.

### Схема

```
Клиент подключается:
  │
  ├── ReadTimeout (5s) — чтение запроса
  │
  ├── Обработка
  │
  ├── WriteTimeout (10s) — запись ответа
  │
  └── IdleTimeout (60s) — ожидание следующего запроса
```

### Что не защищено

**Таймауты сервера НЕ защищают от:**

- **Медленной БД.** Если `db.Query` висит — таймаут не сработает.
- **Медленного внешнего API.** То же.

**Решение:** таймауты на **каждую операцию** через `context`:

```go
func (s *Server) handleUsers(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    users, err := s.db.QueryContext(ctx, "SELECT * FROM users")
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "timeout", http.StatusGatewayTimeout)
            return
        }
        http.Error(w, err.Error(), 500)
        return
    }
    json.NewEncoder(w).Encode(users)
}
```

### Рекомендации

| Таймаут | Рекомендация | Почему |
|:---|:---|:---|
| `ReadTimeout` | 5 сек | Защита от медленных клиентов |
| `ReadHeaderTimeout` | 2 сек | Защита от медленных заголовков |
| `WriteTimeout` | 10 сек | Защита от медленных клиентов |
| `IdleTimeout` | 60 сек | Keep-alive |

### 💡 Практика: как настроить таймауты

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`ReadTimeout`** — 5 сек.
2. **`WriteTimeout`** — 10 сек.
3. **`IdleTimeout`** — 60 сек.
4. **Таймауты на операции** — через `context`.

**👍 СТОИТ СДЕЛАТЬ:**

5. **`ReadHeaderTimeout`** — 2 сек.
6. **`context.WithTimeout`** в каждом обработчике.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `http.Server` без таймаутов.**
8. **Не забывай про таймауты БД и API.**

---

## 34.5 Middleware: логирование, метрики, recovery

Middleware — **cross-cutting concerns**.

### Паттерн middleware

```go
type Middleware func(http.Handler) http.Handler
```

**Что делает:** оборачивает `http.Handler` в другой `http.Handler`.

### Цепочка middleware

```go
func chain(h http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}
```

**Порядок:** последний в списке — **внешний**.

### Логирование

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}
        next.ServeHTTP(wrapped, r)

        slog.Info("request",
            "method", r.Method,
            "path", r.URL.Path,
            "status", wrapped.statusCode,
            "duration", time.Since(start),
            "remote", r.RemoteAddr,
        )
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (w *responseWriter) WriteHeader(code int) {
    w.statusCode = code
    w.ResponseWriter.WriteHeader(code)
}
```

### Метрики

```go
var (
    requestsTotal   = prometheus.NewCounterVec(...)
    requestDuration = prometheus.NewHistogramVec(...)
)

func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}

        next.ServeHTTP(wrapped, r)

        requestsTotal.WithLabelValues(r.Method, r.URL.Path, strconv.Itoa(wrapped.statusCode)).Inc()
        requestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(time.Since(start).Seconds())
    })
}
```

### Recovery от паник

```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                slog.Error("panic",
                    "error", rec,
                    "stack", string(debug.Stack()),
                )
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

### Комбинация

```go
handler := chain(
    mux,
    recoveryMiddleware,
    loggingMiddleware,
    metricsMiddleware,
    s.rateLimitMiddleware,
    s.semaphoreMiddleware,
)
```

**Порядок** — от внешнего к внутреннему.

### 💡 Практика: как использовать middleware

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Recovery** — защита от паник.
2. **Logging** — структурированный лог.
3. **Metrics** — Prometheus.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Rate limiter** — защита от DDoS.
5. **Семафор** — ограничение параллелизма.

**❌ НЕ ДЕЛАЙ:**

6. **Не забывай recovery.**
7. **Не логируй всё подряд.**

---

## 34.6 Health checks: /healthz и /readyz

Health checks — **обязательны** для Kubernetes.

### Два эндпоинта

**`/healthz`** — жив ли сервис.

**`/readyz`** — готов ли принимать трафик.

### Разница

| Эндпоинт | Что проверяет | Что делать при провале |
|:---|:---|:---|
| `/healthz` | Процесс жив | **Перезапустить** pod |
| `/readyz` | Готов принимать трафик | **Убрать из load balancer** |

### Реализация

```go
func (s *Server) handleHealthz(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    fmt.Fprintln(w, "ok")
}

func (s *Server) handleReadyz(w http.ResponseWriter, r *http.Request) {
    // Проверяем зависимости
    if !s.db.Ping(r.Context()) {
        http.Error(w, "db not ready", http.StatusServiceUnavailable)
        return
    }

    if !s.cache.Ping(r.Context()) {
        http.Error(w, "cache not ready", http.StatusServiceUnavailable)
        return
    }

    w.WriteHeader(http.StatusOK)
    fmt.Fprintln(w, "ready")
}
```

### Что проверять в `/readyz`

**Обязательно:**

- **БД** — `db.Ping`.
- **Кэш** — `redis.Ping`.
- **Внешние сервисы** — если критичны.

**Опционально:**

- **Миграции** — если применились.
- **Конфигурация** — если загружена.

### Kubernetes

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /readyz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Схема

```
Kubernetes:
  │
  ├── /healthz каждые 10 сек
  │   └── если fail — перезапустить pod
  │
  └── /readyz каждые 5 сек
      └── если fail — убрать из load balancer
```

### 💡 Практика: как делать health checks

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`/healthz`** — процесс жив.
2. **`/readyz`** — зависимости готовы.
3. **`Ping` для БД и кэша.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Liveness и readiness probes** в K8s.
5. **Таймауты** на проверки.

**❌ НЕ ДЕЛАЙ:**

6. **Не делай `/healthz` тяжёлым.** Он вызывается часто.
7. **Не игнорируй зависимости в `/readyz`.**

---

## 34.7 Graceful shutdown

Graceful shutdown (Глава 26) — **обязателен** для production.

### Реализация

```go
func (s *Server) Run(ctx context.Context) error {
    errCh := make(chan error, 1)

    go func() {
        slog.Info("server starting", "addr", s.httpSrv.Addr)
        if err := s.httpSrv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            errCh <- err
        }
        close(errCh)
    }()

    select {
    case <-ctx.Done():
        slog.Info("shutdown signal received")
    case err := <-errCh:
        return err
    }

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
    defer cancel()

    if err := s.httpSrv.Shutdown(shutdownCtx); err != nil {
        slog.Error("shutdown error", "error", err)
        s.httpSrv.Close()
        return err
    }

    slog.Info("shutdown complete")
    return nil
}
```

### Main

```go
func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    slog.SetDefault(logger)

    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    srv := NewServer(":8080", 100, 1000)

    if err := srv.Run(ctx); err != nil {
        slog.Error("run failed", "error", err)
        os.Exit(1)
    }
}
```

### Что происходит при shutdown

```
t=0:    SIGTERM
t=0:    ctx.Done() закрыт
t=0:    srv.Shutdown вызывается
        - Новые соединения НЕ принимаются
        - r.Context() отменяется
        - Ждём активные обработчики
t=25s:  Таймаут (если не успели)
        - srv.Close (force)
```

### 💡 Практика: как делать graceful shutdown

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **`signal.NotifyContext`.**
2. **`srv.Shutdown`** с таймаутом.
3. **`srv.Close`** при таймауте.

**👍 СТОИТ СДЕЛАТЬ:**

4. **25 секунд** — меньше K8s.
5. **Логирование** переходов.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `os.Exit` в Run.**
7. **Не забывай `http.ErrServerClosed`.**

---

## 34.8 Мониторинг и pprof

Мониторинг — **обязателен** для production.

### pprof

**Включаем pprof сервер:**

```go
import _ "net/http/pprof"

func main() {
    // pprof на localhost:6060
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()

    // основной сервер на :8080
    srv := NewServer(":8080", 100, 1000)
    srv.Run(ctx)
}
```

**Что даёт:**

- **CPU profile:** `curl localhost:6060/debug/pprof/profile`.
- **Memory profile:** `curl localhost:6060/debug/pprof/heap`.
- **Goroutine dump:** `curl localhost:6060/debug/pprof/goroutine?debug=2`.
- **Block profile:** `curl localhost:6060/debug/pprof/block`.
- **Mutex profile:** `curl localhost:6060/debug/pprof/mutex`.

### Метрики Prometheus

```go
import "github.com/prometheus/client_golang/prometheus/promhttp"

func (s *Server) handleMetrics(w http.ResponseWriter, r *http.Request) {
    promhttp.Handler().ServeHTTP(w, r)
}

// Регистрируем
s.mux.Handle("/metrics", http.HandlerFunc(s.handleMetrics))
```

**Что даёт:** endpoint `/metrics` для Prometheus.

### Структурированное логирование

**`slog`** (Go 1.21+):

```go
slog.Info("request handled",
    "method", r.Method,
    "path", r.URL.Path,
    "status", status,
    "duration_ms", duration.Milliseconds(),
    "user_id", userID,
)
```

### Ключевые метрики

| Метрика | Что измеряет |
|:---|:---|
| `http_requests_total` | Всего запросов |
| `http_request_duration_seconds` | Latency |
| `http_requests_in_flight` | Активные запросы |
| `http_response_status_total` | По статусам |
| `goroutines` | Число горутин |
| `memory_bytes` | Память |

### 💡 Практика: как мониторить

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **pprof на localhost:6060.**
2. **Prometheus `/metrics`.**
3. **`slog` для логов.**

**👍 СТОИТ СДЕЛАТЬ:**

4. **Алерты** на latency, errors.
5. **Grafana** для визуализации.

**❌ НЕ ДЕЛАЙ:**

6. **Не открывай pprof наружу.**
7. **Не логируй всё подряд.**

---

## 34.9 Полный production-ready сервер

Соберём всё вместе.

### Полный код

```go
package main

import (
    "context"
    "encoding/json"
    "errors"
    "fmt"
    "log/slog"
    "net"
    "net/http"
    _ "net/http/pprof"
    "os"
    "os/signal"
    "strings"
    "sync"
    "syscall"
    "time"

    "golang.org/x/time/rate"
)

type Server struct {
    httpSrv *http.Server
    mux     *http.ServeMux
    sem     chan struct{}
    limiter *rate.Limiter
    limiters sync.Map
}

func NewServer(addr string, maxConcurrent, rps int) *Server {
    s := &Server{
        mux:     http.NewServeMux(),
        sem:     make(chan struct{}, maxConcurrent),
        limiter: rate.NewLimiter(rate.Limit(rps), rps*2),
    }

    s.mux.HandleFunc("/api/users", s.handleUsers)
    s.mux.HandleFunc("/api/orders", s.handleOrders)
    s.mux.HandleFunc("/healthz", s.handleHealthz)
    s.mux.HandleFunc("/readyz", s.handleReadyz)

    handler := chain(
        s.mux,
        recoveryMiddleware,
        loggingMiddleware,
        s.rateLimitMiddleware,
        s.semaphoreMiddleware,
    )

    s.httpSrv = &http.Server{
        Addr:              addr,
        Handler:           handler,
        ReadTimeout:       5 * time.Second,
        ReadHeaderTimeout: 2 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       60 * time.Second,
    }

    return s
}

func (s *Server) Run(ctx context.Context) error {
    errCh := make(chan error, 1)

    go func() {
        slog.Info("server starting", "addr", s.httpSrv.Addr)
        if err := s.httpSrv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            errCh <- err
        }
        close(errCh)
    }()

    select {
    case <-ctx.Done():
        slog.Info("shutdown signal received")
    case err := <-errCh:
        return err
    }

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
    defer cancel()

    if err := s.httpSrv.Shutdown(shutdownCtx); err != nil {
        slog.Error("shutdown error", "error", err)
        s.httpSrv.Close()
        return err
    }

    slog.Info("shutdown complete")
    return nil
}

func (s *Server) semaphoreMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        select {
        case s.sem <- struct{}{}:
            defer func() { <-s.sem }()
        case <-r.Context().Done():
            return
        case <-time.After(1 * time.Second):
            http.Error(w, "server busy", http.StatusServiceUnavailable)
            return
        }
        next.ServeHTTP(w, r)
    })
}

func (s *Server) rateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if !s.limiter.Allow() {
            http.Error(w, "too many requests", http.StatusTooManyRequests)
            return
        }
        next.ServeHTTP(w, r)
    })
}

func (s *Server) handleUsers(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    select {
    case <-ctx.Done():
        http.Error(w, "timeout", http.StatusGatewayTimeout)
        return
    case <-time.After(50 * time.Millisecond):
    }

    users := []string{"alice", "bob"}
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func (s *Server) handleOrders(w http.ResponseWriter, r *http.Request) {
    orders := []string{"order1", "order2"}
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(orders)
}

func (s *Server) handleHealthz(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    fmt.Fprintln(w, "ok")
}

func (s *Server) handleReadyz(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    fmt.Fprintln(w, "ready")
}

func chain(h http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}
        next.ServeHTTP(wrapped, r)

        slog.Info("request",
            "method", r.Method,
            "path", r.URL.Path,
            "status", wrapped.statusCode,
            "duration_ms", time.Since(start).Milliseconds(),
        )
    })
}

func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                slog.Error("panic", "error", rec)
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (w *responseWriter) WriteHeader(code int) {
    w.statusCode = code
    w.ResponseWriter.WriteHeader(code)
}

func getClientIP(r *http.Request) string {
    if xff := r.Header.Get("X-Forwarded-For"); xff != "" {
        return strings.Split(xff, ",")[0]
    }
    host, _, _ := net.SplitHostPort(r.RemoteAddr)
    return host
}

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    slog.SetDefault(logger)

    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    // pprof на localhost
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()

    srv := NewServer(":8080", 100, 1000)

    if err := srv.Run(ctx); err != nil {
        slog.Error("run failed", "error", err)
        os.Exit(1)
    }
}
```

### Что демонстрирует

1. **Семафор** — не более 100 одновременных запросов.
2. **Rate limiter** — не более 1000 запросов/сек.
3. **Таймауты** — Read, Write, Idle.
4. **Middleware** — recovery, logging, rate limit, semaphore.
5. **Health checks** — `/healthz`, `/readyz`.
6. **Graceful shutdown** — по сигналу.
7. **pprof** — на localhost:6060.
8. **slog** — structured logging.

### 💡 Практика: как строить production HTTP

**✅ ОБЯЗАТЕЛЬНО ДЕЛАЙ:**

1. **Всё из примера выше.**
2. **Тесты** — goleak, -race, stress.
3. **Метрики** — Prometheus.

**👍 СТОИТ СДЕЛАТЬ:**

4. **Grafana** — визуализация.
5. **Алерты** на latency, errors.

**❌ НЕ ДЕЛАЙ:**

6. **Не выпускай в production без тестов.**
7. **Не игнорируй pprof.**

---

## 34.10 Выводы и типичные ошибки

**Что мы узнали?**

HTTP-сервер — синтез паттернов. **Семафор** ограничивает параллелизм. **Rate limiter** защищает от DDoS. **Таймауты** (Read, Write, Idle) защищают от медленных клиентов. **Middleware** — для cross-cutting concerns. **Health checks** — `/healthz`, `/readyz`. **Graceful shutdown** — по сигналу. **pprof** — для диагностики. **slog** — для structured logging.

**Типичные ошибки:**

- ❌ **Нет ограничения параллелизма.** БД захлёбывается.
- ❌ **Нет rate limiter.** DDoS.
- ❌ **Нет таймаутов.** Медленные клиенты.
- ❌ **Нет recovery middleware.** Паника роняет сервер.
- ❌ **Нет health checks.** Kubernetes не знает состояние.
- ❌ **Нет graceful shutdown.** Обрыв запросов при деплое.
- ❌ **pprof открыт наружу.** Безопасность.
- ❌ **Игнорировать `http.ErrServerClosed`.**
- ❌ **`srv.Shutdown` без таймаута.** Висеть вечно.
- ❌ **Не логировать запросы.**
- ❌ **Слишком маленькие таймауты.** False timeout.
- ❌ **Слишком большие таймауты.** Клиент ушёл.

---

## 34.11 Для быстрого повторения

- **Семафор** — ограничение параллелизма.
- **Rate limiter** — защита от DDoS.
- **Таймауты** — Read, Write, Idle.
- **`ReadTimeout` 5 сек**, **`WriteTimeout` 10 сек**, **`IdleTimeout` 60 сек**.
- **Таймауты на операции** — через `context`.
- **Middleware** — recovery, logging, metrics.
- **`/healthz`** — процесс жив.
- **`/readyz`** — готов.
- **Graceful shutdown** — `srv.Shutdown` с таймаутом.
- **pprof** на localhost:6060.
- **slog** — structured logging.
- **Prometheus** — `/metrics`.
- **Ключевые метрики:** requests, latency, in_flight.

---

## 34.12 Вопросы для самопроверки

1. Как ограничить параллелизм в HTTP-сервере?
2. Как защититься от DDoS?
3. Какие таймауты у `http.Server`?
4. Зачем middleware? Какие?
5. Что такое `/healthz` и `/readyz`?
6. Как сделать graceful shutdown?
7. Зачем pprof?
8. Что будет без таймаутов?

---

## 34.13 Ответы

### Ответ 1

**Семафор:**

```go
select {
case sem <- struct{}{}:
    defer func() { <-sem }()
case <-time.After(1 * time.Second):
    http.Error(w, "server busy", 503)
    return
}
```

### Ответ 2

**Rate limiter:**

```go
if !limiter.Allow() {
    http.Error(w, "too many requests", 429)
    return
}
```

### Ответ 3

**Три таймаута:**
- `ReadTimeout` — чтение запроса.
- `WriteTimeout` — запись ответа.
- `IdleTimeout` — ожидание следующего запроса.

### Ответ 4

**Middleware** — для cross-cutting concerns:
- **Recovery** — защита от паник.
- **Logging** — структурированный лог.
- **Metrics** — Prometheus.
- **Rate limiter** — DDoS.
- **Semaphore** — параллелизм.

### Ответ 5

**`/healthz`** — процесс жив. Если fail — Kubernetes перезапустит pod.

**`/readyz`** — готов принимать трафик. Если fail — убрать из load balancer.

### Ответ 6

```go
ctx, stop := signal.NotifyContext(context.Background(),
    syscall.SIGINT, syscall.SIGTERM)
defer stop()

<-ctx.Done()

shutdownCtx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
defer cancel()

srv.Shutdown(shutdownCtx)
```

### Ответ 7

**pprof** — для диагностики:
- CPU profile.
- Memory profile.
- Goroutine dump.
- Block profile.
- Mutex profile.

### Ответ 8

**Без таймаутов:** медленные клиенты занимают горутины. 10 000 висящих соединений — сервер падает.

---

## 34.14 Куда идти дальше?

Мы разобрали HTTP-сервер — синтез паттернов. Теперь мы умеем строить production-ready сервисы.

Но есть другие **реальные сценарии**: очереди и consumer, ETL, scraper, WebSocket.

- **Как обрабатывать очереди?** → **Глава 35: Очереди и consumer.**
- **Как построить ETL?** → **Глава 36: ETL.**
- **Как написать scraper?** → **Глава 37: Scraper.**
- **Как работать с WebSocket?** → **Глава 38: WebSocket и streaming.**

---

## 34.15 Чек-лист

| Компонент | Что это | Ключевые факты |
|:---|:---|:---|
| **HTTP-сервер** | Синтез паттернов | Все вместе |
| **Семафор** | Ограничение параллелизма | `make(chan struct{}, N)` |
| **Rate limiter** | Защита от DDoS | `rate.NewLimiter` |
| **`ReadTimeout`** | 5 сек | Чтение запроса |
| **`WriteTimeout`** | 10 сек | Запись ответа |
| **`IdleTimeout`** | 60 сек | Keep-alive |
| **Middleware** | Recovery, logging, metrics | Cross-cutting |
| **`/healthz`** | Процесс жив | Liveness |
| **`/readyz`** | Готов | Readiness |
| **Graceful shutdown** | `srv.Shutdown` | 25 сек |
| **pprof** | localhost:6060 | Диагностика |
| **slog** | Structured logging | JSON |
| **Prometheus** | `/metrics` | Метрики |
| **Таймауты на операции** | `context.WithTimeout` | Для БД, API |

🌐 **Ключевая идея:** HTTP-сервер — синтез паттернов: семафор (параллелизм), rate limiter (DDoS), таймауты (медленные клиенты), middleware (cross-cutting), health checks (K8s), graceful shutdown (деплой), pprof (диагностика), slog (логи). **Семафор** ограничивает число одновременных запросов. **Rate limiter** защищает от DDoS. **Таймауты** Read 5s, Write 10s, Idle 60s. **Middleware** — recovery, logging, metrics. **`/healthz`** — процесс жив. **`/readyz`** — зависимости готовы. **Graceful shutdown** — `srv.Shutdown` с таймаутом. **pprof** на localhost:6060. **Ключевые метрики:** requests_total, duration, in_flight, goroutines. Не выпускай в production без тестов и мониторинга.