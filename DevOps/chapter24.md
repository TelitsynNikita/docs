# 🛡️ Глава 24: Отказоустойчивость и автопилот

**Что вы узнаете:**
- Что такое отказоустойчивость и как её измерять (MTBF, MTTR, RTO, RPO).
- Что такое Health Probes: liveness, readiness, startup — и когда что использовать.
- Что такое Pod Disruption Budgets и зачем они нужны.
- Как работает Horizontal Pod Autoscaler (HPA) и по каким метрикам масштабировать.
- Что такое Vertical Pod Autoscaler (VPA) и когда он полезен.
- Как работает Cluster Autoscaler и Karpenter.
- Что такое anti-affinity и topology spread для отказоустойчивости.
- Что такое chaos engineering и как его проводить.
- Что такое graceful shutdown и как его реализовать.
- Как проектировать системы, которые переживают отказы.

**После прочтения вы сможете:**
- Настроить Health Probes для приложения.
- Настроить HPA и VPA.
- Настроить Pod Disruption Budgets.
- Настроить Cluster Autoscaler или Karpenter.
- Использовать anti-affinity и topology spread.
- Проводить chaos engineering эксперименты.
- Реализовать graceful shutdown.
- Диагностировать проблемы с автопилотом.

---

## Содержание

- [24.0 Пролог: пятница, 18:00 — упала нода](#240-пролог-пятница-1800--упала-нода)
- [24.1 Что такое отказоустойчивость](#241-что-такое-отказоустойчивость)
- [24.2 Health Probes: liveness, readiness, startup](#242-health-probes-liveness-readiness-startup)
- [24.3 Graceful shutdown](#243-graceful-shutdown)
- [24.4 Pod Disruption Budgets](#244-pod-disruption-budgets)
- [24.5 Horizontal Pod Autoscaler (HPA)](#245-horizontal-pod-autoscaler-hpa)
- [24.6 Vertical Pod Autoscaler (VPA)](#246-vertical-pod-autoscaler-vpa)
- [24.7 Cluster Autoscaler и Karpenter](#247-cluster-autoscaler-и-karpenter)
- [24.8 Anti-affinity и topology spread](#248-anti-affinity-и-topology-spread)
- [24.9 Chaos engineering](#249-chaos-engineering)
- [24.10 Проектирование отказоустойчивых систем](#2410-проектирование-отказоустойчивых-систем)
- [24.11 Диагностика проблем](#2411-диагностика-проблем)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 24.0 Пролог: пятница, 18:00 — упала нода

Пятница, 18:00. Ты собираешься уходить. Внезапно — алерт:

```
[CRITICAL] Node node-3 is NotReady
```

Ты открываешь дашборд. На `node-3` работают:

- **3 Pod'а** `api`.
- **2 Pod'а** `worker`.
- **1 Pod** `cache`.

Итого — **6 Pod'ов**.

**Что происходит:**

1. Kubernetes видит, что `node-3` NotReady.
2. Через `node-monitor-grace-period` (40 секунд) — помечает ноду как NotReady.
3. Через `pod-eviction-timeout` (5 минут) — начинает evict'ить Pod'ы.
4. Pod'ы пересоздаются на других нодах.

**Но есть проблемы:**

**1. Downtime.**

Пока Pod'ы пересоздаются — сервис недоступен. Если у тебя 3 реплики, а одна нода унесла 3 — осталось 0. **Downtime.**

**2. Cache.**

Cache Pod был **stateful**. Данные потеряны. Приложение работает медленнее.

**3. Worker.**

Worker Pod'ы обрабатывали задачи. Часть задач потеряна. **Дубликаты.**

**4. Capacity.**

На других нодах не хватает ресурсов. Pod'ы в Pending. **Overload.**

**5. Cascading failure.**

Оставшиеся ноды перегружены. Могут упасть тоже. **Cascade.**

**Это — не одна проблема, а отсутствие отказоустойчивости.**

**Правильная система:**

- **3+ реплики** каждого сервиса.
- **Anti-affinity** — Pod'ы на разных нодах.
- **PodDisruptionBudget** — минимум доступных.
- **Cluster Autoscaler** — добавить ноды.
- **Health probes** — быстрое обнаружение.
- **Graceful shutdown** — завершение без потери.
- **Chaos engineering** — тестирование отказов.

В этой главе мы разберём отказоустойчивость. Как проектировать системы, которые переживают отказы.

Это — **основа production**. Без отказоустойчивости — каждый сбой = инцидент.

---

## 24.1 Что такое отказоустойчивость

### 🔌 Проблема: системы падают

Любая система падает. Вопрос — **когда** и **что происходит**.

**Плохая система:** падение = downtime, потеря данных, cascade.
**Хорошая система:** падение = незаметно для пользователя.

**Отказоустойчивость** — способность системы продолжать работу при отказах компонентов.

### 📊 Термины

**MTBF (Mean Time Between Failures)** — среднее время между отказами.

**MTTR (Mean Time To Repair)** — среднее время восстановления.

**Availability** — `MTBF / (MTBF + MTTR)`.

**Пример:**

- MTBF = 1000 часов.
- MTTR = 1 час.
- Availability = 1000 / 1001 = 99.9%.

**«Девятки»:**

| Availability | Downtime/месяц | Downtime/год |
|:---|:---|:---|
| 99% (2 девятки) | 7.2 часа | 3.65 дня |
| 99.9% (3 девятки) | 43.2 минуты | 8.76 часа |
| 99.99% (4 девятки) | 4.32 минуты | 52.6 минуты |
| 99.999% (5 девяток) | 26 секунд | 5.26 минуты |

**Каждая девятка — в 10 раз дороже.**

### 🎯 RTO и RPO

**RTO (Recovery Time Objective)** — сколько времени допустимо восстанавливать.

**RPO (Recovery Point Objective)** — сколько данных допустимо потерять.

**Пример:**

- RTO = 1 час — восстановление ≤ 1 час.
- RPO = 15 минут — потеря ≤ 15 минут данных.

**Как достичь:**

- **Частые бэкапы** → меньше RPO.
- **Быстрое восстановление** → меньше RTO.
- **Replication + failover** → почти 0 RTO/RPO.

### 🎯 Уровни отказоустойчивости

**1. Single instance.**

- **Downtime** при отказе.
- **Потеря данных** при отказе.
- **MTTR** = ручное восстановление.

**2. Multi-instance.**

- **Минимальный downtime** (переключение).
- **Replication** данных.
- **MTTR** = автоматическое переключение.

**3. Multi-zone.**

- **Отказ зоны** — не влияет.
- **Replication** между зонами.
- **MTTR** = секунды.

**4. Multi-region.**

- **Отказ региона** — не влияет.
- **Replication** между регионами.
- **MTTR** = минуты.

**5. Active-active.**

- **Несколько активных** регионов.
- **Load balancing** между ними.
- **MTTR** = 0.

**Каждый уровень — дороже.**

### 🎯 Failure modes

**1. Pod failure.**

Pod упал. Kubernetes пересоздаёт.

**2. Node failure.**

Нода упала. Pod'ы переезжают.

**3. Zone failure.**

Зона недоступна. Pod'ы в других зонах.

**4. Region failure.**

Регион недоступен. Переключение.

**5. Application failure.**

Приложение падает. Health probes обнаруживают.

**6. Dependency failure.**

БД недоступна. Retries, circuit breaking.

**7. Network failure.**

Сеть недоступна. Retries, timeouts.

**8. Human error.**

Человек ошибся. Rollback, backups.

### 💡 Практика: что важно понять

**✅ ОБЯЗАТЕЛЬНО:**

1. **Failure неизбежен.** Планируй.
2. **Каждая девятка — дороже.**
3. **RTO и RPO** определяют дизайн.

**👍 СТОИТ:**

4. **Multi-zone** для production.
5. **Replication** данных.
6. **Chaos engineering** для тестирования.

**❌ НЕ ДЕЛАЙ:**

7. **Не полагайся на один инстанс.**
8. **Не игнорируй failure modes.**
9. **Не проектируй без RTO/RPO.**

### Где мы сейчас

Мы разобрали, что такое отказоустойчивость. Теперь — **Health Probes**.

---

## 24.2 Health Probes: liveness, readiness, startup

### 🔌 Проблема: как Kubernetes узнаёт о состоянии Pod'а

Pod запущен. Но:

- **Готов ли** он принимать трафик?
- **Живой ли** он?
- **Запустился ли** он полностью?

**Health Probes** отвечают на эти вопросы.

### 📊 Три типа probes

**1. Liveness Probe.**

**Вопрос:** Pod жив?

**Что делает:** если **нет** — перезапускает Pod.

**Когда использовать:** для обнаружения deadlock, hung state.

**Осторожно:** неправильная liveness probe → бесконечный restart loop.

**2. Readiness Probe.**

**Вопрос:** Pod готов принимать трафик?

**Что делает:** если **нет** — убирает из Service endpoints.

**Когда использовать:** для обнаружения, что приложение ещё не готово (миграции, warmup).

**Важно:** Pod продолжает работать, но не получает трафик.

**3. Startup Probe.**

**Вопрос:** приложение запустилось?

**Что делает:** блокирует liveness и readiness до успеха.

**Когда использовать:** для медленно стартующих приложений (Java, ML).

**Что даёт:** liveness не убивает Pod во время долгого старта.

### 🎯 Пример

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      ports:
        - containerPort: 8080
      
      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        failureThreshold: 30
        periodSeconds: 10
        # 30 × 10 = 5 минут на старт
      
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 0  # startup probe уже отработал
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3
      
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
        timeoutSeconds: 3
        failureThreshold: 2
```

### 🎯 Механизмы проверки

**1. HTTP GET.**

```yaml
httpGet:
  path: /healthz
  port: 8080
  httpHeaders:
    - name: Custom-Header
      value: Awesome
```

**Что проверяет:** ответ 200-399 = успех.

**2. TCP Socket.**

```yaml
tcpSocket:
  port: 8080
```

**Что проверяет:** соединение установлено.

**3. Exec.**

```yaml
exec:
  command:
    - cat
    - /tmp/healthy
```

**Что проверяет:** команда вернула 0.

**4. gRPC (K8s 1.24+).**

```yaml
grpc:
  port: 50051
```

**Что проверяет:** gRPC health check.

### 🎯 Параметры

| Параметр | Что означает | По умолчанию |
|:---|:---|:---|
| **initialDelaySeconds** | Задержка перед первой проверкой | 0 |
| **periodSeconds** | Интервал между проверками | 10 |
| **timeoutSeconds** | Таймаут проверки | 1 |
| **successThreshold** | Успехов для «здоров» | 1 |
| **failureThreshold** | Неудач для «нездоров» | 3 |

### 🎯 Endpoints для probes

**Liveness:**

```go
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    // Проверить, что приложение не в deadlock
    // Не проверять внешние зависимости!
    w.WriteHeader(http.StatusOK)
})
```

**Readiness:**

```go
http.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
    // Проверить БД, Redis, Kafka
    if err := db.Ping(); err != nil {
        w.WriteHeader(http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
})
```

**Правило:**

- **Liveness** — только внутреннее состояние. **Не проверять БД.** Если БД недоступна — Pod перезапустится, но БД не станет доступнее.
- **Readiness** — можно проверять зависимости. Если БД недоступна — Pod не готов, но работает.

### 🎯 Пример на Go

```go
package main

import (
    "context"
    "database/sql"
    "net/http"
    "sync/atomic"
    "time"
)

var (
    healthy int32
    ready   int32
)

func main() {
    // Симуляция старта
    time.Sleep(5 * time.Second)
    atomic.StoreInt32(&healthy, 1)
    
    db, _ := sql.Open("postgres", "...")
    
    http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
        if atomic.LoadInt32(&healthy) == 0 {
            w.WriteHeader(http.StatusServiceUnavailable)
            return
        }
        w.WriteHeader(http.StatusOK)
    })
    
    http.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 1*time.Second)
        defer cancel()
        
        if err := db.PingContext(ctx); err != nil {
            w.WriteHeader(http.StatusServiceUnavailable)
            return
        }
        w.WriteHeader(http.StatusOK)
    })
    
    http.ListenAndServe(":8080", nil)
}
```

### 🎯 Частые ошибки

**1. Liveness проверяет БД.**

**Проблема:** БД недоступна → liveness fails → Pod перезапускается → всё ещё не работает → restart loop.

**Решение:** liveness — только внутреннее.

**2. Liveness с малым timeout.**

**Проблема:** при нагрузке timeout — Pod перезапускается.

**Решение:** timeout ≥ 5 секунд. failureThreshold ≥ 3.

**3. Нет readiness.**

**Проблема:** трафик идёт на Pod, который не готов.

**Решение:** всегда readiness.

**4. Startup probe для быстрых приложений.**

**Проблема:** избыточно.

**Решение:** только для медленно стартующих.

**5. Проверка на localhost.**

**Проблема:** приложение слушает 0.0.0.0, но probe на 127.0.0.1.

**Решение:** проверять на том же порту.

### 🎯 Best practices

**1. Разные endpoints.**

- `/healthz` — liveness.
- `/ready` — readiness.

**2. Разные параметры.**

- **Startup:** длинный period, высокий failureThreshold.
- **Liveness:** редкий period, высокий failureThreshold.
- **Readiness:** частый period, низкий failureThreshold.

**3. Не проверять зависимости в liveness.**

**4. Graceful shutdown при SIGTERM.**

### 🔬 Практика: Health Probes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          ports:
            - containerPort: 8080
          
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            failureThreshold: 30
            periodSeconds: 10
          
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2
```

**Проверка:**

```bash
# Смотреть события
kubectl describe pod myapp-xxx | grep -A 10 Events

# Liveness fail
# Warning  Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 500

# Readiness fail
# Warning  Unhealthy  Readiness probe failed: ...

# Startup fail
# Warning  Unhealthy  Startup probe failed: ...
```

### 💡 Практика: как правильно настраивать probes

**✅ ОБЯЗАТЕЛЬНО:**

1. **Readiness** — всегда.
2. **Liveness** — только внутреннее состояние.
3. **Startup** — для медленно стартующих.
4. **Разные endpoints.**

**👍 СТОИТ:**

5. **failureThreshold ≥ 3.**
6. **timeoutSeconds ≥ 5** для liveness.
7. **Тестировать** probes.

**❌ НЕ ДЕЛАЙ:**

8. **Не проверяй БД в liveness.**
9. **Не используй маленькие timeouts.**
10. **Не забывай про readiness.**

### Где мы сейчас

Мы разобрали Health Probes. Теперь — **graceful shutdown**.

---

## 24.3 Graceful shutdown

### 🔌 Проблема: Pod убивается, запросы теряются

Pod получает SIGTERM. Kubernetes ждёт `terminationGracePeriodSeconds` (30 секунд). Потом SIGKILL.

**Проблема:** если приложение не завершает активные запросы — они теряются.

**Решение:** graceful shutdown.

### 📊 Что такое graceful shutdown

**Graceful shutdown** — корректное завершение приложения.

**Что делать:**

1. **Получить SIGTERM.**
2. **Перестать принимать** новые запросы.
3. **Дождаться** завершения активных.
4. **Закрыть** connections (БД, Redis).
5. **Завершиться.**

### 🎯 Проблема race condition

**Проблема:** Pod получает SIGTERM, начинает shutdown. Но Kubernetes **ещё не убрал** Pod из Service endpoints.

**Что происходит:**

1. SIGTERM → Pod.
2. Pod начинает shutdown.
3. Kubernetes обновляет endpoints (убирает Pod).
4. Но **за это время** трафик идёт на Pod.
5. Pod уже не принимает запросы → ошибки.

**Решение 1: `preStop` hook.**

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
```

**Что даёт:** Pod ждёт 10 секунд перед SIGTERM. За это время Kubernetes убирает его из endpoints.

**Решение 2: read-only readiness.**

При SIGTERM — readiness probe начинает возвращать ошибку. Kubernetes убирает Pod из endpoints.

### 🎯 Реализация в Go

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    srv := &http.Server{Addr: ":8080"}
    
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(100 * time.Millisecond)
        w.Write([]byte("OK"))
    })
    
    // Запуск в горутине
    go func() {
        log.Println("Starting server on :8080")
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()
    
    // Ждём SIGTERM
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
    <-quit
    log.Println("Shutdown signal received")
    
    // Graceful shutdown с timeout
    ctx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
    defer cancel()
    
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("Shutdown error: %v", err)
    }
    
    log.Println("Server stopped gracefully")
}
```

**Что происходит:**

1. SIGTERM → `quit`.
2. `srv.Shutdown(ctx)`:
   - Перестаёт принимать новые connections.
   - Ждёт завершения активных.
   - Возвращает, когда всё завершено или timeout.

### 🎯 `terminationGracePeriodSeconds`

```yaml
spec:
  terminationGracePeriodSeconds: 60
```

**Правило:** `terminationGracePeriodSeconds` > graceful shutdown timeout в приложении.

**Пример:**

- Graceful shutdown timeout в Go: 25 секунд.
- `terminationGracePeriodSeconds`: 60 секунд.
- SIGTERM → Go делает shutdown (≤ 25 секунд).
- Если не успел за 60 — SIGKILL.

### 🎯 preStop hook

```yaml
spec:
  containers:
    - name: myapp
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 15"]
```

**Что даёт:**

1. SIGTERM (from kubelet).
2. `preStop` выполняется: `sleep 15`.
3. Kubernetes убирает Pod из endpoints (за эти 15 секунд).
4. После `preStop` → SIGTERM в приложение.
5. Graceful shutdown.

**Зачем:** race condition между SIGTERM и обновлением endpoints.

**Рекомендация:** `sleep 5-15`.

### 🎯 Комбинация

**Правильно:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: myapp
          image: myapp:1.0
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            periodSeconds: 5
```

**Плюс graceful shutdown в коде.**

**Тайминги:**

- `preStop`: 10 секунд (убрать из endpoints).
- Graceful shutdown: 25 секунд.
- `terminationGracePeriodSeconds`: 60 секунд (запас).

### 🎯 Для Jobs

**Проблема:** sidecar (Istio, Linkerd) не завершается. Job никогда не завершится.

**Решение:**

```yaml
metadata:
  annotations:
    sidecar.istio.io/inject: "false"
```

**Или** использовать `native sidecar` (K8s 1.28+):

```yaml
initContainers:
  - name: istio-proxy
    restartPolicy: Always  # native sidecar
    image: istio/proxyv2:...
```

**Что даёт:** sidecar завершается вместе с основным контейнером.

### 🔬 Практика: graceful shutdown

```bash
# 1. Развернуть приложение с graceful shutdown
kubectl apply -f deployment.yaml

# 2. Проверить Pod
kubectl get pods

# 3. Тест: удалить Pod
kubectl delete pod myapp-xxx

# 4. Смотреть логи
kubectl logs myapp-xxx --previous
# "Shutdown signal received"
# "Server stopped gracefully"

# 5. Проверить, что нет ошибок
# В другой сессии — генерировать нагрузку
while true; do curl myapp/api; done

# При удалении Pod — не должно быть ошибок
```

### 💡 Практика: как правильно делать graceful shutdown

**✅ ОБЯЗАТЕЛЬНО:**

1. **Graceful shutdown** в коде (SIGTERM → shutdown).
2. **`terminationGracePeriodSeconds`** > graceful timeout.
3. **`preStop: sleep`** для избежания race condition.
4. **Readiness probe** для уборки из endpoints.

**👍 СТОИТ:**

5. **Тестировать** под нагрузкой.
6. **Мониторить** ошибки при деплое.
7. **Закрывать connections** (БД, Redis).

**❌ НЕ ДЕЛАЙ:**

8. **Не игнорируй SIGTERM.**
9. **Не используй SIGKILL** для приложения.
10. **Не забывай про sidecar.**

### Где мы сейчас

Мы разобрали graceful shutdown. Теперь — **Pod Disruption Budgets**.

---

## 24.4 Pod Disruption Budgets

### 🔌 Проблема: плановые disruption'ы

Kubernetes может **планово** вытеснять Pod'ы:

- **Node drain** (обновление нод).
- **Cluster Autoscaler** (уменьшение нод).
- **Upgrade кластера.**

**Проблема:** если все Pod'ы вытеснены одновременно — downtime.

**Решение:** Pod Disruption Budgets.

### 📊 Что такое PodDisruptionBudget

**PodDisruptionBudget (PDB)** — гарантия минимального количества доступных Pod'ов.

**Два способа:**

- **`minAvailable`** — минимум доступных.
- **`maxUnavailable`** — максимум недоступных.

**Важно:** PDB работает только для **плановых** disruption'ов. Для **аварийных** (node failure) — не работает.

### 🎯 Пример

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
```

**Что даёт:** всегда **минимум 2** Pod'а `myapp` доступны.

**При drain:**

- Kubernetes будет вытеснять Pod'ы по одному.
- Ждёт, пока новый Pod запустится.
- Только потом — следующий.

### 🎯 Проценты

```yaml
spec:
  minAvailable: 66%  # или maxUnavailable: 34%
  selector:
    matchLabels:
      app: myapp
```

**Что даёт:** минимум 66% Pod'ов доступны.

### 🎯 Правила

**1. `minAvailable` vs `maxUnavailable`.**

- **`minAvailable: 2`** — при 3 репликах → максимум 1 вытеснен.
- **`maxUnavailable: 1`** — при 3 репликах → максимум 1 вытеснен.

**Эквивалентны в этом случае.**

**2. Осторожно с `minAvailable: 1`.**

Если 1 реплика — PDB заблокирует все drain'ы. Возможно, не нужно.

**3. При 1 реплике.**

PDB с `minAvailable: 1` — не даст drain. **Плохо.**

**Решение:** использовать `maxUnavailable: 0` — тоже не даст. Лучше — не использовать PDB с 1 репликой.

**4. При 2 репликах.**

PDB с `minAvailable: 1` — позволяет drain по одному.

### 🎯 Проверка

```bash
# Посмотреть PDB
kubectl get pdb -A

# Детали
kubectl describe pdb myapp-pdb

# Показать:
# STATUS:     ACTIVE
# MIN AVAILABLE: 2
# MAX UNAVAILABLE: N/A
# ALLOWED DISRUPTIONS: 1   ← сколько можно вытеснить сейчас
```

**`ALLOWED DISRUPTIONS`** — ключевое поле. Если 0 — drain заблокирован.

### 🎯 Node drain

```bash
# Drain ноды
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# Если PDB блокирует
# Error: Cannot evict pod as it would violate the pod's disruption budget.
```

**Что происходит:**

1. Kubernetes пытается evict Pod.
2. Проверяет PDB.
3. Если нарушает — блокирует.
4. Ждёт, пока можно.

### 🎯 Upgrade кластера

**Managed K8s** (EKS, GKE, AKS) при upgrade:

- Drain ноды.
- Учитывает PDB.
- Постепенно.

**Без PDB:**

- Все Pod'ы вытеснены → downtime.

**С PDB:**

- Минимум доступных.
- Нет downtime.

### 🎯 Проблемы

**1. PDB блокирует drain.**

**Причины:**

- `minAvailable` = replicas.
- Нет свободных нод.

**Решение:** увеличить replicas или уменьшить `minAvailable`.

**2. PDB не защищает от node failure.**

**Причины:** PDB только для плановых.

**Решение:** anti-affinity + replicas ≥ 2.

**3. PDB с 1 репликой.**

**Проблема:** блокирует всё.

**Решение:** не использовать или использовать `maxUnavailable: 1`.

### 🔬 Практика: PDB

```bash
# 1. Deployment с 3 репликами
kubectl create deployment myapp --image=nginx --replicas=3

# 2. PDB
cat > pdb.yaml <<EOF
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
EOF
kubectl apply -f pdb.yaml

# 3. Проверить
kubectl get pdb myapp-pdb
# NAME        MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# myapp-pdb   2               N/A               1                     10s

# 4. Drain ноды
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# 5. Смотреть
kubectl get pods -w
# Pod'ы вытесняются по одному

# 6. Uncordon
kubectl uncordon node-1
```

### 💡 Практика: как правильно настраивать PDB

**✅ ОБЯЗАТЕЛЬНО:**

1. **PDB для всех production workloads.**
2. **Replicas ≥ 2.**
3. **`minAvailable` < replicas.**

**👍 СТОИТ:**

4. **Разные PDB** для разных сервисов.
5. **Мониторинг** `ALLOWED DISRUPTIONS`.
6. **Тестирование** drain.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй PDB с 1 репликой.**
8. **Не ставь `minAvailable` = replicas.**
9. **Не забывай про anti-affinity.**

### Где мы сейчас

Мы разобрали PDB. Теперь — **HPA**.

---

## 24.5 Horizontal Pod Autoscaler (HPA)

### 🔌 Проблема: нагрузка меняется

Утром 100 RPS. Вечером 1000 RPS. Ночью 10 RPS.

**Фиксированные реплики:**

- **Много** → дорого ночью.
- **Мало** → падает вечером.

**Решение:** HPA.

### 📊 Что такое HPA

**HorizontalPodAutoscaler** — автоматически масштабирует количество Pod'ов.

**Как работает:**

1. HPA контроллер каждые 15 секунд (по умолчанию) смотрит метрики.
2. Сравнивает с целевыми значениями.
3. Меняет `replicas` в Deployment/StatefulSet.

### 🎯 Метрики

**1. CPU.**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

**Что даёт:** масштабирует, чтобы CPU был ~70%.

**2. Memory.**

```yaml
metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**Осторожно:** memory не всегда хорошо для HPA. Приложения часто держат память.

**3. Custom metrics.**

```yaml
metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

**Что даёт:** масштабирует, чтобы RPS на Pod был ~100.

**Требует:** Prometheus Adapter или custom metrics API.

**4. External metrics.**

```yaml
metrics:
  - type: External
    external:
      metric:
        name: kafka_consumergroup_lag
        selector:
          matchLabels:
            consumergroup: myapp
      target:
        type: AverageValue
        averageValue: "1000"
```

**Что даёт:** масштабирует по lag в Kafka.

### 🎯 Требования

**1. Resource requests.**

HPA с CPU **требует** `resources.requests.cpu`. Без них — не работает.

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

**2. Metrics Server.**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**Проверка:**

```bash
kubectl top pods
kubectl top nodes
```

### 🎯 Алгоритм

```
desiredReplicas = ceil(currentReplicas × (currentMetric / targetMetric))
```

**Пример:**

- currentReplicas = 3.
- currentCPU = 90%.
- targetCPU = 70%.

```
desiredReplicas = ceil(3 × (90/70)) = ceil(3.86) = 4
```

**Ограничения:**

- Не чаще, чем раз в 15 секунд.
- Плавное изменение (stabilization window).

### 🎯 Поведение

**Scale up:**

- **Быстро.** При высокой нагрузке.
- **Stabilization window:** 0 секунд (по умолчанию).

**Scale down:**

- **Медленно.** Чтобы избежать flapping.
- **Stabilization window:** 5 минут (по умолчанию).

**Настройка:**

```yaml
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

### 🎯 Проблемы

**1. Flapping.**

Масштаб вверх-вниз. **Решение:** stabilization windows.

**2. Нет requests.**

HPA не работает. **Решение:** добавить requests.

**3. Медленный scale up.**

При резком всплеске — Pod'ы не успевают. **Решение:** overprovisioning, KEDA.

**4. CPU — не всегда правильная метрика.**

Приложение может быть I/O bound. **Решение:** custom metrics.

**5. HPA + Cluster Autoscaler.**

Если нод не хватает — Pod'ы в Pending. **Решение:** Cluster Autoscaler.

### 🎯 KEDA

**KEDA** — Kubernetes Event-Driven Autoscaling.

**Что даёт:**

- **Масштабирование по событиям** (Kafka lag, RabbitMQ queue, cron).
- **Scale to zero.**

**Пример:**

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer
spec:
  scaleTargetRef:
    name: kafka-consumer
  minReplicaCount: 0
  maxReplicaCount: 30
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: myapp
        topic: orders
        lagThreshold: "100"
```

**Что даёт:** масштабирует по lag в Kafka. 0 при отсутствии сообщений.

### 🔬 Практика: HPA

```bash
# 1. Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 2. Deployment с requests
kubectl create deployment myapp --image=nginx --replicas=2
kubectl set resources deployment myapp --requests=cpu=100m,memory=128Mi

# 3. HPA
kubectl autoscale deployment myapp --cpu-percent=70 --min=2 --max=10

# 4. Проверить
kubectl get hpa
# NAME    REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS
# myapp   Deployment/myapp   5%/70%    2         10        2

# 5. Нагрузка
kubectl run -it --rm load --image=busybox --restart=Never -- \
  sh -c "while true; do wget -q -O- http://myapp; done"

# 6. Смотреть
kubectl get hpa -w
# TARGETS растут, REPLICAS растут

# 7. Остановить нагрузку
# Через 5 минут — scale down
```

### 💡 Практика: как правильно настраивать HPA

**✅ ОБЯЗАТЕЛЬНО:**

1. **Resource requests.**
2. **Metrics Server.**
3. **minReplicas ≥ 2.**
4. **Stabilization windows.**

**👍 СТОИТ:**

5. **Custom metrics** для I/O bound.
6. **KEDA** для event-driven.
7. **Тестирование** под нагрузкой.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй HPA без requests.**
9. **Не забывай про Cluster Autoscaler.**
10. **Не используй только CPU** для всего.

### Где мы сейчас

Мы разобрали HPA. Теперь — **VPA**.

---

## 24.6 Vertical Pod Autoscaler (VPA)

### 🔌 Проблема: неправильные requests

Приложение использует 200 MB. Requests — 1 GB. **80% простаивает.**

Или наоборот: requests 100 MB, использует 500 MB. **OOM.**

**Решение:** VPA.

### 📊 Что такое VPA

**VerticalPodAutoscaler** — автоматически подбирает requests и limits.

**Как работает:**

1. VPA контроллер смотрит на использование.
2. Рекомендует requests/limits.
3. Опционально — обновляет Pod'ы.

### 🎯 Режимы

**1. Off.**

Только рекомендации, без изменений.

```yaml
spec:
  updatePolicy:
    updateMode: "Off"
```

**2. Initial.**

Устанавливает при создании Pod'а.

```yaml
spec:
  updatePolicy:
    updateMode: "Initial"
```

**3. Recreate.**

Пересоздаёт Pod'ы при изменении.

```yaml
spec:
  updatePolicy:
    updateMode: "Recreate"
```

**4. Auto.**

Как Recreate.

```yaml
spec:
  updatePolicy:
    updateMode: "Auto"
```

**In-place (alpha):** обновление без пересоздания.

### 🎯 Пример

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
      - containerName: myapp
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 2
          memory: 2Gi
```

**Что даёт:** рекомендации без изменений.

### 🎯 Проверка рекомендаций

```bash
kubectl get vpa myapp-vpa -o yaml
```

**Что увидишь:**

```yaml
status:
  recommendation:
    containerRecommendations:
      - containerName: myapp
        lowerBound:
          cpu: 100m
          memory: 200Mi
        target:
          cpu: 200m
          memory: 256Mi
        upperBound:
          cpu: 500m
          memory: 512Mi
```

**Что означают:**

- **lowerBound** — минимум.
- **target** — рекомендуемое.
- **upperBound** — максимум.

### 🎯 Проблемы

**1. VPA + HPA конфликт.**

Оба меняют resources. **Решение:** не использовать одновременно для CPU/memory.

**2. Recreate = downtime.**

VPA пересоздаёт Pod'ы. **Решение:** Off mode + ручное применение.

**3. Не для всех workloads.**

Для batch — хорошо. Для latency-sensitive — осторожно.

**4. Требует времени.**

Нужно собрать данные. **Решение:** запустить в Off mode, посмотреть через неделю.

### 🎯 Практическое использование

**Рекомендуемый подход:**

1. **Запустить VPA в Off mode.**
2. **Через неделю** — посмотреть рекомендации.
3. **Вручную** применить в Deployment.
4. **Периодически** проверять.

**Что даёт:** правильные requests без downtime.

### 🔬 Практика: VPA

```bash
# 1. Установить VPA
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh

# 2. Deployment
kubectl create deployment myapp --image=nginx --replicas=2
kubectl set resources deployment myapp --requests=cpu=100m,memory=128Mi

# 3. VPA в Off mode
cat > vpa.yaml <<EOF
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Off"
EOF
kubectl apply -f vpa.yaml

# 4. Через несколько часов
kubectl get vpa myapp-vpa -o yaml | grep -A 20 recommendation

# 5. Применить рекомендации
kubectl set resources deployment myapp \
  --requests=cpu=200m,memory=256Mi \
  --limits=cpu=500m,memory=512Mi
```

### 💡 Практика: как правильно использовать VPA

**✅ ОБЯЗАТЕЛЬНО:**

1. **Off mode** сначала.
2. **Мониторинг** рекомендаций.
3. **Ручное применение.**
4. **Периодически** проверять.

**👍 СТОИТ:**

5. **minAllowed, maxAllowed.**
6. **Не использовать с HPA.**
7. **Отдельно для CPU и memory.**

**❌ НЕ ДЕЛАЙ:**

8. **Не используй Recreate в production** без понимания.
9. **Не используй с HPA.**
10. **Не применяй сразу.**

### Где мы сейчас

Мы разобрали VPA. Теперь — **Cluster Autoscaler и Karpenter**.

---

## 24.7 Cluster Autoscaler и Karpenter

### 🔌 Проблема: нод не хватает

HPA создал 100 Pod'ов. Но нод только 10. Pod'ы в Pending.

**Решение:** Cluster Autoscaler или Karpenter.

### 📊 Cluster Autoscaler (CA)

**Cluster Autoscaler** — добавляет/удаляет ноды в node group.

**Как работает:**

1. Смотрит на Pending Pod'ы.
2. Определяет, какая node group подходит.
3. Увеличивает размер node group.
4. Новая нода → Pod'ы запланированы.

**И удаление:**

1. Смотрит на неиспользуемые ноды.
2. Если нода не нужна → удаляет.

### 🎯 Настройка

**В облаке:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
        - name: cluster-autoscaler
          image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.0
          command:
            - ./cluster-autoscaler
            - --v=4
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
```

**Node group с min/max:**

```yaml
# Настройка ASG (AWS)
MinSize: 1
MaxSize: 20
```

### 🎯 Expanders

**Какую node group выбрать:**

- **random** — случайно.
- **most-pods** — максимум Pod'ов.
- **least-waste** — минимум простоя.
- **price** — самая дешёвая.
- **priority** — по приоритетам.

**Рекомендация:** `least-waste` или `price`.

### 🎯 Karpenter

**Karpenter** — альтернатива CA от AWS (теперь CNCF).

**Отличия:**

| Аспект | CA | Karpenter |
|:---|:---|:---|
| **Node groups** | Да | Нет |
| **Instance types** | Ограничены | Любые |
| **Speed** | Минуты | Секунды |
| **Bin packing** | Базовый | Умный |
| **Spot** | Да | Да, автоматически |
| **Cloud** | Multi | AWS, Azure |

**Karpenter** выбирает instance type **на лету** — оптимальный для Pod'ов.

### 🎯 Karpenter пример

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["t3.medium", "t3.large", "t3a.medium", "t3a.large"]
      nodeClassRef:
        name: default
  limits:
    cpu: 1000
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
```

**Что даёт:**

- **Любые instance types.**
- **Spot + on-demand.**
- **Consolidation** — убирает неиспользуемые.
- **Expire** — пересоздаёт ноды.

### 🎯 Provisioner / NodeClass

```yaml
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2
  role: KarpenterNodeRole
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
```

**Что даёт:** Karpenter создаёт ноды в нужных subnets, с нужными security groups.

### 🎯 Когда что

**Cluster Autoscaler:**

- **Multi-cloud.**
- **On-premise.**
- **Уже настроен.**

**Karpenter:**

- **AWS (или Azure).**
- **Хочется скорость и оптимизация.**
- **Много разных workloads.**

### 🎯 Проблемы

**1. CA не масштабирует из-за taints.**

Pod не толерантен к taint ноды. CA не создаст.

**2. CA не масштабирует из-за PDB.**

PDB блокирует удаление Pod'а.

**3. CA медленный.**

Минуты. **Решение:** Karpenter.

**4. Spot interruptions.**

Spot instance может быть отозван. **Решение:** обработка SIGTERM.

**5. Не хватает instance types.**

CA ограничен node group. **Решение:** Karpenter.

### 🔬 Практика: Cluster Autoscaler

```bash
# 1. Установить
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=us-west-2

# 2. Создать нагрузку
kubectl create deployment load --image=nginx --replicas=100

# 3. Смотреть
kubectl get nodes -w
# Новые ноды добавляются

kubectl get pods -w
# Pending → Running

# 4. Уменьшить нагрузку
kubectl scale deployment load --replicas=1

# 5. Через 10 минут — ноды удалены
```

### 💡 Практика: как правильно настраивать autoscaler

**✅ ОБЯЗАТЕЛЬНО:**

1. **CA или Karpenter** для production.
2. **min/max nodes.**
3. **Taints/tolerations** для разных workloads.
4. **Мониторинг.**

**👍 СТОИТ:**

5. **Karpenter** для AWS.
6. **Spot** для экономии.
7. **Consolidation** для оптимизации.

**❌ НЕ ДЕЛАЙ:**

8. **Не забывай про PDB.**
9. **Не игнорируй Spot interruptions.**
10. **Не ставь max = unlimited.**

### Где мы сейчас

Мы разобрали autoscaler. Теперь — **anti-affinity и topology spread**.

---

## 24.8 Anti-affinity и topology spread

### 🔌 Проблема: Pod'ы на одной ноде

3 реплики на одной ноде. Нода упала. **Все 3 упали.**

**Решение:** anti-affinity.

### 📊 Anti-affinity

**Pod anti-affinity** — не размещать Pod'ы рядом.

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: myapp
          topologyKey: kubernetes.io/hostname
```

**Что даёт:** каждый Pod `myapp` — на **разной** ноде.

**Проблема:** если нод меньше, чем Pod'ов — часть Pod'ов в Pending.

**Решение:** preferred.

```yaml
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: myapp
        topologyKey: kubernetes.io/hostname
```

**Что даёт:** предпочитает разные ноды, но не блокирует.

### 📊 Multi-zone anti-affinity

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: myapp
      topologyKey: topology.kubernetes.io/zone
```

**Что даёт:** Pod'ы в **разных зонах**.

**Когда:** критичные сервисы. Отказ зоны → сервис работает.

### 📊 Topology Spread

**Topology Spread Constraints** — равномерное распределение.

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: myapp
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app: myapp
```

**Что даёт:**

- Сначала равномерно по зонам.
- Потом — по нодам (мягко).

**Разница от anti-affinity:**

- **Anti-affinity:** «не рядом» — жёстко.
- **Topology Spread:** «равномерно» — гибко.

### 🎯 Комбинация

**Правильный дизайн:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 6
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: myapp
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: myapp
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: myapp
              topologyKey: kubernetes.io/hostname
```

**Что даёт:**

- **Required:** разные ноды.
- **Preferred:** равномерно по зонам.
- **6 реплик:** 2-2-2 по зонам.

### 🎯 Stateful workloads

**Для баз данных:**

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: postgres
      topologyKey: topology.kubernetes.io/zone
```

**Что даёт:** каждая реплика Postgres — в разной зоне.

**Плюс:** PDB для гарантии.

### 🎯 Проблемы

**1. Недостаточно нод.**

Anti-affinity → Pod'ы в Pending. **Решение:** preferred или Cluster Autoscaler.

**2. Недостаточно зон.**

3 зоны, 5 реплик. **Решение:** topology spread с maxSkew.

**3. Pending Pod'ы.**

**Диагностика:** `kubectl describe pod`.

### 🔬 Практика: anti-affinity

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: myapp
              topologyKey: kubernetes.io/hostname
      containers:
        - name: myapp
          image: nginx
```

**Проверка:**

```bash
kubectl get pods -o wide
# NAME    NODE
# myapp   node-1
# myapp   node-2
# myapp   node-3
```

### 💡 Практика: как правильно использовать anti-affinity

**✅ ОБЯЗАТЕЛЬНО:**

1. **Required для критичных** (БД).
2. **Preferred для остальных.**
3. **Multi-zone** для production.

**👍 СТОИТ:**

4. **Topology spread** для равномерности.
5. **PDB** для гарантии.
6. **Cluster Autoscaler.**

**❌ НЕ ДЕЛАЙ:**

7. **Не используй required для больших replicas.**
8. **Не забывай про количество нод.**
9. **Не игнорируй Pending.**

### Где мы сейчас

Мы разобрали anti-affinity. Теперь — **chaos engineering**.

---

## 24.9 Chaos engineering

### 🔌 Проблема: как проверить отказоустойчивость

Настроил HA. Но **работает ли**? **Как проверить?**

**Решение:** chaos engineering.

### 📊 Что такое chaos engineering

**Chaos engineering** — намеренное внесение отказов для проверки системы.

**Принципы:**

1. **Начни с гипотезы.** «Если нода упадёт, сервис продолжит работать».
2. **Ограничь blast radius.** Начни с dev.
3. **Автоматизируй.**
4. **Мониторь.**
5. **Улучшай.**

### 🎯 Типы экспериментов

**1. Pod failure.**

Удалить Pod. **Проверить:** Kubernetes пересоздаёт.

**2. Pod kill.**

Убить процесс. **Проверить:** liveness probe.

**3. Network delay.**

Задержка между сервисами. **Проверить:** timeouts, retries.

**4. Network partition.**

Разделение сети. **Проверить:** circuit breaking.

**5. Node failure.**

Остановить ноду. **Проверить:** Pod'ы переезжают.

**6. Resource exhaustion.**

Заполнить диск, CPU, память. **Проверить:** limits, OOM.

**7. DNS failure.**

DNS недоступен. **Проверить:** resilience.

**8. Time skew.**

Изменение времени. **Проверить:** TTL, сертификаты.

### 🎯 Инструменты

**1. Chaos Mesh.**

Kubernetes-native. Поддерживает много типов.

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill
spec:
  action: pod-kill
  mode: one
  selector:
    labelSelectors:
      app: myapp
  scheduler:
    cron: "@every 1m"
```

**2. Litmus.**

Chaos engineering для K8s.

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: myapp-chaos
spec:
  appinfo:
    appns: production
    applabel: app=myapp
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "30"
            - name: CHAOS_INTERVAL
              value: "10"
```

**3. Gremlin.**

Commercial. Enterprise-grade.

**4. AWS Fault Injection Simulator.**

Для AWS-инфраструктуры.

**5. Istio fault injection.**

Network-level. Delay, abort.

### 🎯 Game Days

**Game Day** — запланированное тестирование отказов.

**Структура:**

1. **Подготовка.**
   - Выбрать сценарий.
   - Уведомить команду.
   - Установить метрики.
   - План rollback.

2. **Проведение.**
   - Внести fault.
   - Наблюдать.
   - Записывать.

3. **Анализ.**
   - Что сломалось?
   - Что сработало?
   - Что улучшить?

4. **Действия.**
   - Улучшения.
   - Документация.
   - Следующий Game Day.

### 🎯 Пример эксперимента

**Гипотеза:** «Если одна нода упадёт, API продолжит работать с latency < 500ms».

**Эксперимент:**

1. **Blast radius:** 1 нода из 5.
2. **Fault:** `kubectl drain node-1 --force`.
3. **Метрики:**
   - Availability.
   - Latency p99.
   - Error rate.
4. **Длительность:** 5 минут.
5. **Rollback:** `kubectl uncordon node-1`.

**Результат:**

- **Availability:** 99.9% (норма).
- **Latency p99:** 800ms (превышение!).
- **Error rate:** 0.5% (норма).

**Действия:**

- Увеличить capacity на других нодах.
- Оптимизировать запросы.
- Повторить эксперимент.

### 🎯 Best practices

**1. Начни с dev.**

**2. Ограничь blast radius.**

Не более 10% системы.

**3. Имей rollback.**

**4. Уведоми команду.**

**5. Начни с простого.**

Pod kill → Network delay → Node failure.

**6. Автоматизируй.**

Chaos Monkey в production.

**7. Мониторь.**

Все метрики.

**8. Документируй.**

Каждый эксперимент.

### 🎯 Опасности

**1. Слишком большой blast radius.**

Вся система упадёт. **Решение:** начать с малого.

**2. Без rollback.**

Нельзя остановить. **Решение:** всегда rollback.

**3. Без уведомления.**

Команда не понимает. **Решение:** уведомить.

**4. В production без подготовки.**

**Решение:** сначала dev.

**5. Забыть про клиентов.**

**Решение:** учитывать impact.

### 🔬 Практика: Chaos Mesh

```bash
# 1. Установить
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-mesh \
  --create-namespace \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock

# 2. Pod kill
kubectl apply -f - <<EOF
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill
  namespace: production
spec:
  action: pod-kill
  mode: one
  selector:
    labelSelectors:
      app: myapp
  duration: "30s"
EOF

# 3. Смотреть
kubectl get pods -w
# Pod убит, Kubernetes пересоздаёт

# 4. Network delay
kubectl apply -f - <<EOF
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay
  namespace: production
spec:
  action: delay
  mode: all
  selector:
    labelSelectors:
      app: myapp
  delay:
    latency: "500ms"
  duration: "1m"
EOF

# 5. Смотреть метрики
# Latency растёт, но сервис работает
```

### 💡 Практика: как правильно делать chaos

**✅ ОБЯЗАТЕЛЬНО:**

1. **Начать с dev.**
2. **Ограничить blast radius.**
3. **Rollback.**
4. **Уведомить.**
5. **Мониторить.**

**👍 СТОИТ:**

6. **Chaos Mesh** для K8s.
7. **Game Days** регулярно.
8. **Автоматизация.**

**❌ НЕ ДЕЛАЙ:**

9. **Не в production без подготовки.**
10. **Не без rollback.**
11. **Не без уведомления.**

### Где мы сейчас

Мы разобрали chaos engineering. Теперь — **проектирование отказоустойчивых систем**.

---

## 24.10 Проектирование отказоустойчивых систем

### 🔌 Проблема: как спроектировать систему

Всё разобрали по отдельности. Как собрать вместе?

### 📊 Принципы

**1. Assume failure.**

Любой компонент может отказать. Планируй.

**2. Redundancy.**

Не один инстанс. Минимум 2, лучше 3.

**3. Isolation.**

Failure одного не влияет на другого. Bulkhead.

**4. Graceful degradation.**

При отказе — работает хуже, не падает.

**5. Fail fast.**

Быстро обнаружить и отреагировать.

**6. Idempotency.**

Retry безопасен.

**7. Circuit breaking.**

Не молотить по мёртвому сервису.

**8. Timeouts.**

Не ждать вечно.

**9. Backpressure.**

Не перегружать downstream.

**10. Observability.**

Видеть, что происходит.

### 🎯 Пример архитектуры

```
┌─────────────────────────────────────────────────┐
│                    CDN                          │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│              Load Balancer (multi-AZ)            │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│              API Gateway                         │
│              (3 replicas, anti-affinity)        │
└──────────────────┬──────────────────────────────┘
                   │
       ┌───────────┼───────────┐
       │           │           │
┌──────▼─────┐ ┌───▼────┐ ┌───▼──────┐
│ API (3)    │ │Worker  │ │Scheduler │
│ multi-AZ   │ │(3)     │ │(2)       │
└──────┬─────┘ └───┬────┘ └───┬──────┘
       │           │          │
       └───────────┼──────────┘
                   │
       ┌───────────┼───────────┐
       │           │           │
┌──────▼─────┐ ┌───▼────┐ ┌───▼──────┐
│ PostgreSQL │ │ Redis  │ │  Kafka   │
│ Primary+2  │ │Sentinel│ │  3 brok. │
│ multi-AZ   │ │        │ │          │
└────────────┘ └────────┘ └──────────┘
```

**Что даёт:**

- **Multi-AZ** — отказ зоны.
- **3+ replicas** — отказ Pod'а.
- **Anti-affinity** — Pod'ы на разных нодах.
- **Replication** данных.
- **Circuit breaking.**
- **Timeouts.**
- **Monitoring.**

### 🎯 Чек-лист

**Compute:**

- [ ] 3+ replicas.
- [ ] Anti-affinity.
- [ ] Topology spread.
- [ ] PDB.
- [ ] HPA.
- [ ] Cluster Autoscaler.

**Network:**

- [ ] Load balancer (multi-AZ).
- [ ] Ingress с TLS.
- [ ] Network Policies.
- [ ] Service Mesh (опционально).

**Data:**

- [ ] Replication.
- [ ] Backups.
- [ ] PITR.
- [ ] Multi-AZ.

**Application:**

- [ ] Health probes.
- [ ] Graceful shutdown.
- [ ] Timeouts.
- [ ] Retries.
- [ ] Circuit breaking.
- [ ] Idempotency.

**Observability:**

- [ ] Metrics.
- [ ] Logs.
- [ ] Traces.
- [ ] Alerts.
- [ ] SLO.

**Security:**

- [ ] RBAC.
- [ ] Pod Security.
- [ ] Network Policies.
- [ ] Secrets.

**Operations:**

- [ ] Runbooks.
- [ ] Game Days.
- [ ] Chaos engineering.
- [ ] Incident response.

### 🎯 SLO

**Что определить:**

- **Availability:** 99.9%.
- **Latency p99:** < 500ms.
- **Error rate:** < 0.1%.

**Как измерять:**

- **SLI:** `success_rate = successful / total`.
- **Error budget:** `1 - SLO`.
- **Burn rate:** скорость исчерпания.

### 🎯 Incident response

**Что нужно:**

1. **Detection.** Алерты.
2. **Triage.** Что сломано?
3. **Mitigation.** Быстрое исправление.
4. **Root cause.** Почему?
5. **Prevention.** Как избежать?

**Runbooks:**

- **Что делать** при инциденте.
- **Кто отвечает.**
- **Как откатить.**

**Post-mortem:**

- **Blameless.**
- **Что произошло.**
- **Что улучшить.**

### 🎯 Resilience patterns

**1. Circuit Breaker.**

Не молотить по мёртвому сервису.

**2. Bulkhead.**

Изоляция ресурсов.

**3. Retry с backoff.**

Не сразу, а с задержкой.

**4. Timeout.**

Не ждать вечно.

**5. Fallback.**

При отказе — альтернатива.

**6. Rate Limiting.**

Защита от перегрузки.

**7. Load Shedding.**

Отбрасывать часть запросов.

**8. Queue-based Load Leveling.**

Очередь сглаживает нагрузку.

### 🎯 Как применять

**1. Определить SLO.**

Из бизнес-требований.

**2. Определить failure modes.**

Что может сломаться.

**3. Оценить impact.**

Что произойдёт.

**4. Применить паттерны.**

Для каждого failure mode.

**5. Тестировать.**

Chaos engineering.

**6. Мониторить.**

SLO, alerts.

**7. Улучшать.**

На основе инцидентов.

### 🔬 Практика: проектирование

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 6  # для multi-zone
  strategy:
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      terminationGracePeriodSeconds: 60
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: myapp
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: myapp
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: myapp
                topologyKey: kubernetes.io/hostname
      containers:
        - name: myapp
          image: myapp:1.0
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            failureThreshold: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 4
  selector:
    matchLabels:
      app: myapp
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 6
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

**Что даёт:**

- **6 replicas** — отказ ноды.
- **Multi-zone** — отказ зоны.
- **Anti-affinity** — разные ноды.
- **PDB** — минимум 4.
- **HPA** — автоскейл.
- **Graceful shutdown.**
- **Health probes.**

### 💡 Практика: как проектировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **SLO** определить.
2. **Failure modes** перечислить.
3. **Паттерны применить.**
4. **Тестировать chaos.**
5. **Мониторить.**

**👍 СТОИТ:**

6. **Multi-zone** для production.
7. **Runbooks** для инцидентов.
8. **Post-mortems** без blame.

**❌ НЕ ДЕЛАЙ:**

9. **Не проектируй без SLO.**
10. **Не игнорируй failure modes.**
11. **Не забывай про chaos.**

### Где мы сейчас

Мы разобрали проектирование. Теперь — **диагностика**.

---

## 24.11 Диагностика проблем

### 🔌 Проблема: что-то не так

Автопилот не работает. Настройки не применяются.

### 🔍 Типичные проблемы

**1. HPA не масштабирует.**

**Причины:**

- Нет Metrics Server.
- Нет requests.
- Метрики недоступны.

**Диагностика:**

```bash
kubectl get hpa
# TARGETS: <unknown>/70%

kubectl describe hpa myapp
# Events покажут ошибку

kubectl top pods
# Error: metrics not available

# Проверить Metrics Server
kubectl get pods -n kube-system | grep metrics-server
kubectl logs -n kube-system deployment/metrics-server
```

**2. VPA не даёт рекомендаций.**

**Причины:**

- Не установлен.
- Мало данных.
- Не тот режим.

**Диагностика:**

```bash
kubectl get vpa
kubectl describe vpa myapp-vpa
kubectl logs -n kube-system deployment/vpa-recommender
```

**3. Cluster Autoscaler не добавляет ноды.**

**Причины:**

- Достигнут max.
- Pod не подходит по taints.
- Cloud provider не настроен.

**Диагностика:**

```bash
kubectl logs -n kube-system deployment/cluster-autoscaler | tail -100

# Что видно:
# "No expansion options"
# "max size reached"
# "node group not found"
```

**4. Pod в Pending из-за affinity.**

**Причины:**

- Недостаточно нод.
- Anti-affinity required.

**Диагностика:**

```bash
kubectl describe pod myapp-xxx
# Events:
#   0/3 nodes are available: 3 node(s) didn't match pod anti-affinity rules
```

**5. PDB блокирует drain.**

**Причины:**

- `ALLOWED DISRUPTIONS: 0`.

**Диагностика:**

```bash
kubectl get pdb myapp-pdb
# ALLOWED DISRUPTIONS: 0

kubectl describe pdb myapp-pdb
# Status:
#   Current Healthy: 2
#   Desired Healthy: 3  ← не хватает
```

**Решение:** увеличить replicas или уменьшить minAvailable.

**6. Health probes fail.**

**Причины:**

- Неправильный endpoint.
- Слишком строгие параметры.
- Приложение не отвечает.

**Диагностика:**

```bash
kubectl describe pod myapp-xxx
# Events:
#   Warning  Unhealthy  Liveness probe failed: ...
#   Warning  Unhealthy  Readiness probe failed: ...

# Проверить вручную
kubectl exec myapp-xxx -- curl localhost:8080/healthz
```

**7. Graceful shutdown не работает.**

**Причины:**

- Нет обработки SIGTERM.
- `terminationGracePeriodSeconds` слишком мал.

**Диагностика:**

```bash
# Логи при удалении Pod'а
kubectl logs myapp-xxx --previous

# Проверить время
time kubectl delete pod myapp-xxx
```

**8. Chaos experiment не работает.**

**Причины:**

- RBAC.
- Неправильный selector.
- Не поддерживается runtime.

**Диагностика:**

```bash
kubectl describe podchaos pod-kill -n chaos-mesh
kubectl logs -n chaos-mesh daemonset/chaos-daemon
```

### 🎯 Общие команды

```bash
# События
kubectl get events -A --sort-by=.lastTimestamp

# Describe
kubectl describe pod myapp-xxx
kubectl describe hpa myapp
kubectl describe vpa myapp-vpa
kubectl describe pdb myapp-pdb

# Метрики
kubectl top pods
kubectl top nodes

# Логи
kubectl logs -n kube-system deployment/metrics-server
kubectl logs -n kube-system deployment/cluster-autoscaler
kubectl logs -n kube-system deployment/vpa-recommender
kubectl logs -n chaos-mesh daemonset/chaos-daemon
```

### 🎯 Тестирование

**1. HPA.**

```bash
# Нагрузка
kubectl run load --image=busybox --restart=Never -- \
  sh -c "while true; do wget -q -O- http://myapp; done"

# Смотреть
kubectl get hpa -w
kubectl get pods -w
```

**2. Cluster Autoscaler.**

```bash
# Много Pod'ов
kubectl create deployment load --image=nginx --replicas=100

# Смотреть
kubectl get nodes -w
kubectl get pods -w
```

**3. PDB.**

```bash
# Drain
kubectl drain node-1 --ignore-daemonsets

# Смотреть
kubectl get pods -w
```

### 🔬 Практика: диагностика

```bash
# 1. HPA не работает
kubectl get hpa
# TARGETS: <unknown>/70%

# 2. Проверить Metrics Server
kubectl top pods
# Error from server (ServiceUnavailable): the server is currently unable to handle the request

# 3. Установить Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 4. Проверить
kubectl top pods
# NAME    CPU(cores)   MEMORY(bytes)
# myapp   1m           12Mi

# 5. HPA
kubectl get hpa
# TARGETS: 5%/70%
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Events** — первый источник.
2. **Describe** ресурсов.
3. **Логи** контроллеров.
4. **Метрики** через `kubectl top`.

**👍 СТОИТ:**

5. **Тестировать** под нагрузкой.
6. **Chaos experiments** для проверки.
7. **Мониторинг** состояния.

**❌ НЕ ДЕЛАЙ:**

8. **Не игнорируй Pending.**
9. **Не забывай про метрики.**
10. **Не диагностируй без тестов.**

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **MTBF** | Mean Time Between Failures. |
| **MTTR** | Mean Time To Repair. |
| **RTO** | Recovery Time Objective. |
| **RPO** | Recovery Point Objective. |
| **SLO** | Service Level Objective. |
| **SLI** | Service Level Indicator. |
| **Liveness Probe** | Проверка живости. |
| **Readiness Probe** | Проверка готовности. |
| **Startup Probe** | Проверка старта. |
| **Graceful shutdown** | Корректное завершение. |
| **preStop hook** | Хук перед SIGTERM. |
| **TerminationGracePeriod** | Время на graceful shutdown. |
| **PDB** | Pod Disruption Budget. |
| **HPA** | Horizontal Pod Autoscaler. |
| **VPA** | Vertical Pod Autoscaler. |
| **Cluster Autoscaler** | Автоскейл нод. |
| **Karpenter** | Альтернатива CA. |
| **Anti-affinity** | Не размещать рядом. |
| **Topology spread** | Равномерное распределение. |
| **Chaos engineering** | Намеренные отказы. |
| **Chaos Mesh** | Инструмент chaos для K8s. |
| **Game Day** | Запланированное тестирование. |
| **Circuit breaker** | Разрыв цепи. |
| **Bulkhead** | Изоляция ресурсов. |
| **Load shedding** | Отбрасывание запросов. |
| **KEDA** | Event-driven autoscaling. |

---

## Что мы узнали?

- **Отказоустойчивость** — способность работать при отказах. MTBF, MTTR, RTO, RPO.
- **Health Probes** — liveness, readiness, startup. Разные endpoints и параметры.
- **Graceful shutdown** — обработка SIGTERM. `terminationGracePeriodSeconds`, `preStop`.
- **PDB** — минимум доступных Pod'ов при плановых disruption'ах.
- **HPA** — автоскейл по CPU, memory, custom metrics. Metrics Server, requests.
- **VPA** — подбор requests/limits. Off mode сначала.
- **Cluster Autoscaler и Karpenter** — автоскейл нод.
- **Anti-affinity и topology spread** — Pod'ы на разных нодах/зонах.
- **Chaos engineering** — тестирование отказов. Chaos Mesh, Game Days.
- **Проектирование** — SLO, failure modes, паттерны, тестирование.
- **Диагностика** — events, describe, logs, метрики.

---

## Типичные ошибки

- ❌ **Нет Health Probes.**
- ❌ **Liveness проверяет БД.**
- ❌ **Нет graceful shutdown.**
- ❌ **Нет PDB.**
- ❌ **HPA без requests.**
- ❌ **VPA + HPA вместе.**
- ❌ **Нет Cluster Autoscaler.**
- ❌ **Нет anti-affinity.**
- ❌ **Все Pod'ы на одной ноде.**
- ❌ **Нет chaos engineering.**
- ❌ **Chaos в production без подготовки.**
- ❌ **Нет SLO.**
- ❌ **Нет runbooks.**
- ❌ **Игнорирование инцидентов.**

---

## Для быстрого повторения

- **Health Probes:** liveness (внутреннее), readiness (трафик), startup (медленный старт).
- **Graceful shutdown:** SIGTERM → shutdown → close. `terminationGracePeriodSeconds`, `preStop`.
- **PDB:** `minAvailable`, `maxUnavailable`. Только плановые disruption'ы.
- **HPA:** CPU, memory, custom. Metrics Server, requests. Stabilization windows.
- **VPA:** Off mode → рекомендации → ручное применение.
- **Cluster Autoscaler:** node groups, min/max. Karpenter: любые instance types.
- **Anti-affinity:** required (критично), preferred (мягко).
- **Topology spread:** maxSkew, whenUnsatisfiable.
- **Chaos engineering:** Chaos Mesh, Game Days, blast radius.
- **Проектирование:** SLO, failure modes, паттерны, тестирование, мониторинг.
- **Диагностика:** events, describe, logs, `kubectl top`.

---

## Вопросы для самопроверки

1. Что такое отказоустойчивость? MTBF, MTTR, RTO, RPO?
2. Три типа Health Probes — что проверяют?
3. Что такое graceful shutdown? Как реализовать?
4. Что такое PDB? Зачем нужен?
5. Что такое HPA? Как работает?
6. Что такое VPA? Когда использовать?
7. Что такое Cluster Autoscaler и Karpenter?
8. Что такое anti-affinity? Required vs preferred?
9. Что такое topology spread?
10. Что такое chaos engineering? Как проводить?
11. Как проектировать отказоустойчивые системы?
12. HPA не масштабирует — как диагностировать?
13. Pod в Pending — как диагностировать?
14. Что такое preStop hook? Зачем нужен?
15. Что такое circuit breaker и bulkhead?

---

## Ответы

**1. Отказоустойчивость**

Способность работать при отказах. MTBF — время между отказами. MTTR — время восстановления. RTO — допустимое время восстановления. RPO — допустимая потеря данных.

**2. Health Probes**

Liveness — Pod жив? Если нет — перезапуск. Readiness — готов принимать трафик? Если нет — убрать из endpoints. Startup — запустился? Блокирует liveness/readiness.

**3. Graceful shutdown**

Корректное завершение. SIGTERM → перестать принимать → дождаться активных → закрыть connections → завершиться. `terminationGracePeriodSeconds` > timeout. `preStop: sleep 10`.

**4. PDB**

Pod Disruption Budget. Гарантия минимум доступных при плановых disruption'ах (drain, upgrade). `minAvailable` или `maxUnavailable`. Не защищает от node failure.

**5. HPA**

Horizontal Pod Autoscaler. Масштабирует replicas. По CPU, memory, custom metrics. Требует Metrics Server и resource requests. Каждые 15 секунд.

**6. VPA**

Vertical Pod Autoscaler. Подбирает requests/limits. Режимы: Off, Initial, Recreate, Auto. Не использовать с HPA. Off mode + ручное применение.

**7. Cluster Autoscaler и Karpenter**

CA — добавляет/удаляет ноды в node groups. Karpenter — выбирает instance types на лету. CA для multi-cloud, Karpenter для AWS.

**8. Anti-affinity**

Pod anti-affinity — не размещать Pod'ы рядом. Required — жёстко. Preferred — мягко. Для HA.

**9. Topology spread**

Равномерное распределение Pod'ов. `maxSkew`, `topologyKey`, `whenUnsatisfiable`. Для multi-zone.

**10. Chaos engineering**

Намеренные отказы. Гипотеза → blast radius → эксперимент → анализ. Chaos Mesh, Game Days. Начать с dev.

**11. Проектирование**

SLO → failure modes → паттерны (redundancy, isolation, timeouts) → тестирование chaos → мониторинг → улучшение.

**12. HPA не масштабирует**

1. Metrics Server работает?
2. Requests есть?
3. Метрики доступны?
4. `kubectl describe hpa` — Events.
5. `kubectl top pods`.

**13. Pod в Pending**

1. `kubectl describe pod` — Events.
2. Ресурсы (Insufficient cpu).
3. Affinity (didn't match).
4. Taints (untolerated).
5. PVC (unbound).

**14. preStop hook**

Выполняется перед SIGTERM. `sleep 10` — дать Kubernetes убрать Pod из endpoints. Избежать race condition.

**15. Circuit breaker и bulkhead**

Circuit breaker — не молотить по мёртвому сервису. Bulkhead — изоляция ресурсов (пул для БД, пул для API).

---

## Куда идти дальше?

Мы разобрали отказоустойчивость и автопилот. Теперь ты знаешь:

- Отказоустойчивость, MTBF, MTTR, RTO, RPO.
- Health Probes.
- Graceful shutdown.
- PDB.
- HPA, VPA.
- Cluster Autoscaler, Karpenter.
- Anti-affinity, topology spread.
- Chaos engineering.
- Проектирование.
- Диагностика.

Но мы пока не разобрали:

- **Платформенная инженерия** (Глава 25).
- **Облака и multi-cloud** (Глава 26).

**Глава 25: Платформенная инженерия.** Погнали. 🚀