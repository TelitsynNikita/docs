Принято. Начинаем с нуля. Глава 22 — **Observability — мониторинг и алертинг**.

Это **правильный номер** по нашему оглавлению: после логирования (Глава 21), перед трейсингом (Глава 23).

---

# 📊 Глава 22: Observability — мониторинг и алертинг

**Что вы узнаете:**
- Что такое метрики и какие типы существуют.
- Как работает Prometheus: pull-модель, exporters, service discovery.
- Что такое PromQL и как писать запросы.
- Как инструментировать приложение на Go для Prometheus.
- Как строить дашборды в Grafana.
- Что такое Alertmanager и как маршрутизировать алерты.
- Что такое SLO, SLI, SLA и error budget.
- Как настраивать алерты, которые будят только по делу.
- Что такое recording rules и зачем они нужны.
- Как масштабировать Prometheus (Thanos, VictoriaMetrics, Mimir).
- Как избежать типичных проблем: cardinality, false positives, alert fatigue.

**После прочтения вы сможете:**
- Развернуть Prometheus и Grafana в Kubernetes.
- Инструментировать Go-приложение метриками.
- Писать PromQL-запросы.
- Строить дашборды в Grafana.
- Настраивать Alertmanager с маршрутизацией.
- Определять SLO и error budget.
- Настраивать алерты, которые не шумят.
- Масштабировать Prometheus для больших нагрузок.

---

## Содержание

- [22.0 Пролог: алерт, который никто не читает](#220-пролог-алерт-который-никто-не-читает)
- [22.1 Метрики: что это и какие бывают](#221-метрики-что-это-и-какие-бывают)
- [22.2 Prometheus: pull-based мониторинг](#222-prometheus-pull-based-мониторинг)
- [22.3 Экспортеры и instrumentation](#223-экспортеры-и-instrumentation)
- [22.4 Инструментирование Go-приложения](#224-инструментирование-go-приложения)
- [22.5 Service discovery: как Prometheus находит цели](#225-service-discovery-как-prometheus-находит-цели)
- [22.6 PromQL: язык запросов](#226-promql-язык-запросов)
- [22.7 Recording rules: предвычисления](#227-recording-rules-предвычисления)
- [22.8 Grafana: дашборды](#228-grafana-дашборды)
- [22.9 Alertmanager: маршрутизация алертов](#229-alertmanager-маршрутизация-алертов)
- [22.10 SLO, SLI, SLA и error budget](#2210-slo-sli-sla-и-error-budget)
- [22.11 Алерты, которые не шумят](#2211-алерты-которые-не-шумят)
- [22.12 Масштабирование Prometheus](#2212-масштабирование-prometheus)
- [22.13 Cardinality: главный враг](#2213-cardinality-главный-враг)
- [22.14 Диагностика проблем](#2214-диагностика-проблем)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 22.0 Пролог: алерт, который никто не читает

Ты — DevOps-инженер. У тебя настроен мониторинг. 200 алертов в Slack.

Утро понедельника. В Slack:

```
[ALERT] HighCPUUsage - node-1 - CPU > 80% for 5m
[ALERT] HighCPUUsage - node-2 - CPU > 80% for 5m
[ALERT] HighMemoryUsage - api-xxx - Memory > 90%
[ALERT] SlowQuery - postgres - Query > 1s
[ALERT] DiskSpaceLow - node-3 - Disk > 85%
[ALERT] HighLatency - api - p99 > 500ms
[ALERT] CertificateExpiring - 30 days
[ALERT] PodRestarting - worker-xxx
...
```

**50 сообщений за ночь.** Ты просматриваешь их. Все привычные. CPU 80% — это нормально. Disk 85% — уже неделю. Certificate — есть 30 дней.

Ты закрываешь Slack. **Игнорируешь.**

В 14:00 приходит **реальная** проблема: прод лежит. Оказалось, что `[ALERT] PodRestarting - worker-xxx` — это была серьёзная проблема. Но он утонул в шуме.

**Alert fatigue.** Люди перестают реагировать на алерты, потому что их слишком много и большинство — неважные.

**Это — проблема observability.**

Метрики есть. Алерты есть. Но **сигнал теряется в шуме**.

**Решение:** SLO-based алерты, правильная маршрутизация, actionability.

В этой главе мы разберём мониторинг и алертинг. Начнём с метрик.

Это — Второй путь DevOps (Feedback) в действии. Из Главы 0: правильная обратная связь, а не шум.

---

## 22.1 Метрики: что это и какие бывают

### 🔌 Проблема: как измерить систему

Ты хочешь знать:

- Сколько запросов в секунду?
- Какой latency?
- Сколько ошибок?
- Сколько памяти используется?
- Сколько Pod'ов запущено?

**Всё это — метрики.**

### 📊 Что такое метрика

**Метрика** — числовое значение, измеряемое во времени.

**Структура:**

```
<metric_name>{<labels>} <value> <timestamp>

http_requests_total{method="POST", path="/api/orders", status="200"} 12345 1705312812
```

**Компоненты:**

- **Name** — имя метрики (`http_requests_total`).
- **Labels** — key-value пары для различения (`method`, `path`, `status`).
- **Value** — число (`12345`).
- **Timestamp** — время (`1705312812`).

### 🎯 Четыре типа метрик

**1. Counter (Счётчик).**

**Что:** значение, которое **только растёт**.

**Примеры:**

- `http_requests_total` — всего запросов.
- `errors_total` — всего ошибок.
- `bytes_sent_total` — всего байт отправлено.

**Свойства:**

- Начинается с 0.
- Только растёт (или сбрасывается при рестарте).
- Используется с `rate()` или `increase()` для получения скорости.

**Пример:**

```go
var httpRequests = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total HTTP requests",
    },
    []string{"method", "path", "status"},
)

httpRequests.WithLabelValues("GET", "/api/orders", "200").Inc()
```

**2. Gauge (Измеритель).**

**Что:** значение, которое **может расти и падать**.

**Примеры:**

- `temperature_celsius` — температура.
- `memory_usage_bytes` — использование памяти.
- `goroutines` — число горутин.
- `connections_active` — активные соединения.

**Свойства:**

- Может быть любым.
- Показывает текущее состояние.

**Пример:**

```go
var activeConnections = prometheus.NewGauge(
    prometheus.GaugeOpts{
        Name: "connections_active",
        Help: "Active connections",
    },
)

activeConnections.Inc()  // +1
activeConnections.Dec()  // -1
activeConnections.Set(42) // = 42
```

**3. Histogram (Гистограмма).**

**Что:** распределение значений по бакетам.

**Примеры:**

- `http_request_duration_seconds` — длительность запросов.
- `request_size_bytes` — размер запросов.

**Свойства:**

- Бакеты (buckets) — предопределённые диапазоны.
- Считает количество значений в каждом бакете.
- Сумму всех значений.
- Общее количество.

**Пример:**

```go
var requestDuration = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Help:    "HTTP request duration",
        Buckets: []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10},
    },
    []string{"method", "path"},
)

// Наблюдение
timer := prometheus.NewTimer(requestDuration.WithLabelValues("GET", "/api"))
defer timer.ObserveDuration()
```

**Что даёт:**

- **Percentiles** (p50, p95, p99) через `histogram_quantile()`.
- **Среднее** через `rate(sum) / rate(count)`.

**4. Summary (Сводка).**

**Что:** как histogram, но считает quantiles **на клиенте**.

**Примеры:**

- `http_request_duration_seconds{quantile="0.99"}`.

**Свойства:**

- Quantiles вычисляются приложением.
- Не агрегируются между инстансами.
- Дороже для приложения.

**Рекомендация:** **используй histogram, не summary.** Histogram агрегируется, summary — нет.

### 🎯 Labels

**Labels** — key-value пары для различения метрик.

**Пример:**

```
http_requests_total{method="GET", path="/api/orders", status="200"} 12345
http_requests_total{method="POST", path="/api/orders", status="201"} 67
http_requests_total{method="GET", path="/api/users", status="200"} 8901
```

**Правила:**

- **Низкая cardinality.** Не больше 10-100 уникальных значений на label.
- **Осмысленные имена.** `method`, `status`, `path`.
- **Не использовать high-cardinality** поля: `user_id`, `request_id`, `email`.

**Хорошие labels:**

- `method`, `path` (ограниченный набор).
- `status` (200, 400, 500, ...).
- `service`, `version`, `environment`.
- `instance`, `job`.

**Плохие labels:**

- `user_id` (миллионы значений).
- `request_id` (уникально для каждого запроса).
- `email`, `timestamp`.

### 🎯 Naming conventions

**Правила:**

- **`_total`** — для counter.
- **`_seconds`** — для времени.
- **`_bytes`** — для размера.
- **`_count`** — для count.
- **`_sum`** — для sum.
- **`_bucket`** — для histogram buckets.

**Примеры:**

```
http_requests_total              # counter
http_request_duration_seconds    # histogram (time)
http_request_size_bytes          # histogram (size)
go_goroutines                    # gauge
process_cpu_seconds_total        # counter (CPU)
```

### 🔬 Практика: метрики

```bash
# 1. Посмотреть метрики Go-приложения
curl localhost:8080/metrics

# Пример вывода:
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
# go_goroutines 8
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
# http_requests_total{method="GET",path="/",status="200"} 42
```

### 💡 Практика: как правильно выбирать метрики

**✅ ОБЯЗАТЕЛЬНО:**

1. **Counter для «всего»** (requests, errors, bytes).
2. **Gauge для «текущего»** (connections, memory, goroutines).
3. **Histogram для распределений** (latency, size).
4. **Низкая cardinality** labels.

**👍 СТОИТ:**

5. **Naming conventions** (`_total`, `_seconds`, `_bytes`).
6. **Ограниченный набор labels** (10-100 значений).
7. **Осмысленные имена.**

**❌ НЕ ДЕЛАЙ:**

8. **Не используй user_id, request_id как labels.** Cardinality.
9. **Не используй summary.** Histogram лучше.
10. **Не логируй всё как метрики.** Для этого логи.

### Где мы сейчас

Мы разобрали метрики. Теперь — **Prometheus**.

---

## 22.2 Prometheus: pull-based мониторинг

### 🔌 Проблема: как собирать метрики

Приложение экспортирует метрики на `/metrics`. Как их собирать?

**Варианты:**

- **Push** — приложение отправляет метрики в backend.
- **Pull** — backend сам опрашивает приложения.

**Prometheus** использует **pull-модель**.

### 📊 Что такое Prometheus

**Prometheus** — open-source система мониторинга от CNCF.

**Ключевые характеристики:**

- **Pull-based.** Prometheus сам опрашивает targets.
- **Time series database.** Хранит метрики во времени.
- **PromQL.** Язык запросов.
- **Service discovery.** Автоматически находит targets.
- **Alerting.** Встроенный Alertmanager.
- **Не распределённый.** Один Prometheus — одна БД.

### 🎯 Pull vs Push

**Pull (Prometheus):**

```
┌──────────────┐
│  Prometheus  │
│              │
│  1. Опрашивает /metrics каждые 15 сек
│  2. Парсит метрики
│  3. Сохраняет в TSDB
└──────┬───────┘
       │
       │ GET /metrics
       │
       ▼
┌──────────────┐
│  Application │
│  :8080/metrics│
└──────────────┘
```

**Плюсы pull:**

- **Централизованный контроль.** Prometheus решает, что и когда опрашивать.
- **Простая отладка.** `curl /metrics` — видишь, что экспортируется.
- **Health check.** Если target не отвечает — знаешь.

**Минусы pull:**

- **Не работает для short-lived jobs.** Batch-задачи, которые быстро завершаются.
- **Требует сетевого доступа** от Prometheus к target.
- **Не масштабируется** на тысячи targets (нужны federation, Thanos).

**Push для short-lived jobs:**

Prometheus имеет **Pushgateway** для случаев, когда pull невозможен:

```
Job → Pushgateway → Prometheus (pull)
```

**Использовать осторожно.** Pushgateway не удаляет метрики — нужно чистить.

### 🎯 Архитектура Prometheus

```
┌─────────────────────────────────────────────────────────────┐
│                    PROMETHEUS                                │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Retrieval   │  │  TSDB        │  │  HTTP Server │      │
│  │              │  │              │  │              │      │
│  │  Опрашивает  │  │  Хранит      │  │  API + UI    │      │
│  │  targets     │  │  метрики     │  │              │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐                        │
│  │  Service     │  │  Alerting    │                        │
│  │  Discovery   │  │  Rules       │                        │
│  └──────────────┘  └──────────────┘                        │
│                                                              │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           │ Push alerts
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    ALERTMANAGER                              │
│                                                              │
│  - Маршрутизация                                             │
│  - Группировка                                               │
│  - Подавление                                                │
│  - Отправка (Slack, PagerDuty, Email)                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 🎯 Установка Prometheus

**Через Helm (kube-prometheus-stack):**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

**Что установится:**

- **Prometheus** — сбор метрик.
- **Alertmanager** — алерты.
- **Grafana** — UI.
- **kube-state-metrics** — метрики K8s.
- **node-exporter** — метрики нод.
- **Prometheus Operator** — управление Prometheus через CRD.

### 🎯 Prometheus Operator

**Prometheus Operator** — управление Prometheus через Kubernetes CRD.

**CRD:**

- **Prometheus** — инстанс Prometheus.
- **ServiceMonitor** — как опрашивать сервис.
- **PodMonitor** — как опрашивать Pod'ы.
- **PrometheusRule** — правила алертов и recording.
- **Alertmanager** — инстанс Alertmanager.

**Пример ServiceMonitor:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: production
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
```

**Что произойдёт:** Prometheus Operator автоматически добавит target в конфигурацию Prometheus.

### 🎯 Конфигурация Prometheus

**prometheus.yml (без Operator):**

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: production
    region: us-west-2

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  
  - job_name: 'myapp'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - production
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

**С Operator** — ServiceMonitor вместо этого.

### 🎯 Retention и storage

**Retention:**

```yaml
prometheus:
  prometheusSpec:
    retention: 30d
    retentionSize: 50GB
```

**Storage:**

- **Local SSD** — быстро, но не персистентно.
- **PersistentVolume** — персистентно, но медленнее.
- **Remote write** — отправка в долгосрочное хранилище (Thanos, Mimir, VictoriaMetrics).

### 🎯 Remote write

**Remote write** — отправка метрик в удалённое хранилище.

```yaml
remoteWrite:
  - url: http://victoriametrics:8428/api/v1/write
  - url: http://thanos-receive:19291/api/v1/receive
```

**Зачем:**

- **Долгосрочное хранение.**
- **Масштабирование** (несколько Prometheus).
- **Multi-cluster** (один backend).

### 🔬 Практика: Prometheus

```bash
# 1. Установить kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123

# 2. Проверить
kubectl get pods -n monitoring
# prometheus-prometheus-kube-prometheus-prometheus-0
# alertmanager-prometheus-kube-prometheus-alertmanager-0
# prometheus-grafana-xxx
# prometheus-kube-state-metrics-xxx
# prometheus-prometheus-node-exporter-xxx

# 3. Port-forward Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# 4. Открыть http://localhost:9090
# Status → Targets — список targets

# 5. Простой запрос
# up
# up{job="prometheus"}

# 6. Port-forward Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
# Login: admin / admin123
```

### 💡 Практика: как правильно использовать Prometheus

**✅ ОБЯЗАТЕЛЬНО:**

1. **Prometheus Operator** для управления через CRD.
2. **ServiceMonitor** для опроса приложений.
3. **Retention** настроить.
4. **Persistent storage** для production.

**👍 СТОИТ:**

5. **Remote write** для долгосрочного хранения.
6. **Alertmanager** для алертов.
7. **Recording rules** для предвычислений.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй Pushgateway** для всего. Только для short-lived jobs.
9. **Не забывай про retention.** Диск кончится.
10. **Не храни метрики вечно** локально. Remote write.

### Где мы сейчас

Мы разобрали Prometheus. Теперь — **экспортеры и instrumentation**.

---

## 22.3 Экспортеры и instrumentation

### 🔌 Проблема: как получить метрики из приложений

Приложение не экспортирует метрики. Как их получить?

**Варианты:**

- **Instrumentation** — приложение само экспортирует.
- **Exporter** — отдельный процесс, который переводит метрики в формат Prometheus.

### 📊 Экспортеры

**Exporter** — программа, которая:

1. Собирает метрики из источника (БД, API, файл).
2. Экспортирует их на `/metrics` в формате Prometheus.

**Популярные exporters:**

| Exporter | Что экспортирует |
|:---|:---|
| **node_exporter** | Метрики Linux-нод (CPU, memory, disk, network) |
| **cadvisor** | Метрики контейнеров (встроен в kubelet) |
| **kube-state-metrics** | Метрики K8s объектов (Deployments, Pods, ...) |
| **postgres_exporter** | Метрики PostgreSQL |
| **redis_exporter** | Метрики Redis |
| **mysql_exporter** | Метрики MySQL |
| **nginx_exporter** | Метрики Nginx |
| **blackbox_exporter** | Проверка endpoints (HTTP, TCP, ICMP) |
| **snmp_exporter** | SNMP |
| **jmx_exporter** | Java JMX |

### 🎯 node_exporter

**Что экспортирует:**

- CPU usage.
- Memory usage.
- Disk usage.
- Network traffic.
- Load average.
- Filesystem.

**Установка:** DaemonSet на каждой ноде (в kube-prometheus-stack — автоматически).

**Пример метрик:**

```
node_cpu_seconds_total{cpu="0", mode="user"} 12345.67
node_memory_MemTotal_bytes 16777216000
node_memory_MemAvailable_bytes 8000000000
node_filesystem_size_bytes{device="/dev/sda1", mountpoint="/"} 100000000000
node_filesystem_free_bytes{device="/dev/sda1", mountpoint="/"} 50000000000
node_load1 2.5
node_network_receive_bytes_total{device="eth0"} 1234567890
```

### 🎯 kube-state-metrics

**Что экспортирует:**

- Deployment replicas (desired, ready, available).
- Pod status (Running, Pending, Failed).
- Node status.
- Service endpoints.
- PersistentVolume claims.

**Пример метрик:**

```
kube_deployment_spec_replicas{namespace="production", deployment="api"} 3
kube_deployment_status_replicas_available{namespace="production", deployment="api"} 3
kube_pod_status_phase{namespace="production", pod="api-xxx", phase="Running"} 1
kube_node_status_condition{node="node-1", condition="Ready", status="true"} 1
kube_persistentvolumeclaim_status_phase{namespace="production", persistentvolumeclaim="data", phase="Bound"} 1
```

### 🎯 blackbox_exporter

**Что делает:** проверяет endpoints снаружи.

**Проверки:**

- **HTTP/HTTPS** — код ответа, время.
- **TCP** — соединение.
- **ICMP** — ping.
- **DNS** — резолвинг.

**Пример конфигурации:**

```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200]
      method: GET
      preferred_ip_protocol: ip4
```

**Пример ServiceMonitor:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Probe
metadata:
  name: example-http
spec:
  interval: 60s
  module: http_2xx
  prober:
    url: blackbox-exporter:9115
  targets:
    staticConfig:
      static:
        - https://example.com
        - https://api.example.com/health
```

**Метрики:**

```
probe_success{instance="https://example.com", job="blackbox"} 1
probe_duration_seconds{instance="https://example.com"} 0.123
probe_http_status_code{instance="https://example.com"} 200
```

**Что даёт:** end-to-end проверка сервисов.

### 🎯 Instrumentation vs Exporter

| Аспект | Instrumentation | Exporter |
|:---|:---|:---|
| **Где** | В приложении | Отдельный процесс |
| **Что** | Бизнес-метрики + runtime | Метрики внешней системы |
| **Язык** | SDK (Go, Python, Java) | Любой |
| **Пример** | `http_requests_total` | `node_cpu_seconds_total` |

**Правило:**

- **Своё приложение** — instrumentation.
- **Внешние системы** (БД, кэш) — exporter.
- **Оба** — для полной картины.

### 🔬 Практика: exporters

```bash
# 1. Установить postgres_exporter (пример)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install postgres-exporter prometheus-community/prometheus-postgres-exporter \
  --namespace monitoring \
  --set config.datasource.host=postgres \
  --set config.datasource.user=postgres \
  --set config.datasource.password=secret

# 2. Проверить метрики
kubectl port-forward -n monitoring svc/postgres-exporter 9187:80
curl localhost:9187/metrics | grep pg_

# Примеры:
# pg_up 1
# pg_stat_database_numbackends{datname="mydb"} 5
# pg_stat_database_xact_commit{datname="mydb"} 12345
# pg_database_size_bytes{datname="mydb"} 1073741824

# 3. blackbox_exporter
kubectl port-forward -n monitoring svc/prometheus-blackbox-exporter 9115:9115
curl "localhost:9115/probe?target=https://example.com&module=http_2xx"
```

### 💡 Практика: как правильно использовать exporters

**✅ ОБЯЗАТЕЛЬНО:**

1. **node_exporter** для нод.
2. **kube-state-metrics** для K8s.
3. **Экспортеры для БД, кэшей.**
4. **blackbox_exporter** для внешних проверок.

**👍 СТОИТ:**

5. **ServiceMonitor** для управления.
6. **Гранулярные метрики** (не всё сразу).
7. **Alerting** на основе exporters.

**❌ НЕ ДЕЛАЙ:**

8. **Не собирай метрики без цели.**
9. **Не забывай про RBAC** для exporters.
10. **Не используй один exporter** для всего.

### Где мы сейчас

Мы разобрали exporters. Теперь — **instrumentation Go-приложения**.

---

## 22.4 Инструментирование Go-приложения

### 🔌 Проблема: как добавить метрики в Go

Приложение на Go. Нужно экспортировать метрики: HTTP-запросы, latency, ошибки.

**Решение:** `prometheus/client_golang`.

### 📊 Библиотека

```bash
go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promhttp
```

### 🎯 Базовый пример

```go
package main

import (
    "net/http"
    
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "path", "status"},
    )
    
    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path"},
    )
)

func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}

func handler(w http.ResponseWriter, r *http.Request) {
    timer := prometheus.NewTimer(httpRequestDuration.WithLabelValues(r.Method, r.URL.Path))
    defer timer.ObserveDuration()
    
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("OK"))
    
    httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path, "200").Inc()
}
```

### 🎯 Middleware для HTTP

```go
package middleware

import (
    "net/http"
    "strconv"
    "time"
    
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    requestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )
    
    requestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5},
        },
        []string{"method", "path"},
    )
    
    requestsInFlight = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "http_requests_in_flight",
            Help: "Current number of HTTP requests",
        },
    )
)

func Metrics(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        requestsInFlight.Inc()
        defer requestsInFlight.Dec()
        
        // Обёртка для перехвата status
        rw := &responseWriter{ResponseWriter: w, status: 200}
        
        next.ServeHTTP(rw, r)
        
        duration := time.Since(start).Seconds()
        status := strconv.Itoa(rw.status)
        
        requestsTotal.WithLabelValues(r.Method, r.URL.Path, status).Inc()
        requestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(duration)
    })
}

type responseWriter struct {
    http.ResponseWriter
    status int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.status = code
    rw.ResponseWriter.WriteHeader(code)
}
```

**Использование:**

```go
mux := http.NewServeMux()
mux.HandleFunc("/", handler)
mux.Handle("/metrics", promhttp.Handler())

http.ListenAndServe(":8080", Metrics(mux))
```

### 🎯 Бизнес-метрики

**Не только HTTP. Бизнес-метрики:**

```go
var (
    ordersCreated = promauto.NewCounter(
        prometheus.CounterOpts{
            Name: "orders_created_total",
            Help: "Total number of orders created",
        },
    )
    
    ordersAmount = promauto.NewHistogram(
        prometheus.HistogramOpts{
            Name:    "order_amount_dollars",
            Help:    "Order amount in dollars",
            Buckets: []float64{10, 50, 100, 500, 1000, 5000},
        },
    )
    
    activeUsers = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_users",
            Help: "Number of active users",
        },
    )
)

func createOrder(amount float64) {
    ordersCreated.Inc()
    ordersAmount.Observe(amount)
}
```

### 🎯 Runtime-метрики

**Go runtime метрики** — автоматически:

```go
import (
    "github.com/prometheus/client_golang/prometheus/collectors"
    "github.com/prometheus/client_golang/prometheus"
)

func init() {
    prometheus.MustRegister(collectors.NewGoCollector())
    prometheus.MustRegister(collectors.NewProcessCollector(collectors.ProcessCollectorOpts{}))
}
```

**Что даёт:**

```
go_goroutines 45
go_threads 12
go_memstats_alloc_bytes 12345678
go_memstats_heap_alloc_bytes 8765432
go_gc_duration_seconds{quantile="0.5"} 0.0001
process_cpu_seconds_total 12.34
process_resident_memory_bytes 123456789
process_open_fds 42
```

**Всегда включай runtime-метрики.** Они бесплатно дают много информации.

### 🎯 Кастомные коллекторы

**Для сложных метрик — свой Collector:**

```go
type CustomCollector struct {
    metric *prometheus.Desc
}

func NewCustomCollector() *CustomCollector {
    return &CustomCollector{
        metric: prometheus.NewDesc(
            "custom_metric",
            "Description",
            []string{"label1", "label2"},
            nil,
        ),
    }
}

func (c *CustomCollector) Describe(ch chan<- *prometheus.Desc) {
    ch <- c.metric
}

func (c *CustomCollector) Collect(ch chan<- prometheus.Metric) {
    // Собрать метрики
    value := getValue()
    ch <- prometheus.MustNewConstMetric(
        c.metric,
        prometheus.GaugeValue,
        value,
        "label_value1", "label_value2",
    )
}

func init() {
    prometheus.MustRegister(NewCustomCollector())
}
```

### 🔬 Практика: instrumentation

```go
// main.go
package main

import (
    "fmt"
    "log"
    "math/rand"
    "net/http"
    "time"
    
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/collectors"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequests = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )
    
    httpDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: []float64{0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5},
        },
        []string{"method", "path"},
    )
    
    activeRequests = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "http_active_requests",
            Help: "Active HTTP requests",
        },
    )
    
    errorsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "errors_total",
            Help: "Total errors",
        },
        []string{"type"},
    )
)

func init() {
    prometheus.MustRegister(collectors.NewGoCollector())
    prometheus.MustRegister(collectors.NewProcessCollector(collectors.ProcessCollectorOpts{}))
}

func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.HandleFunc("/api/orders", apiOrders)
    http.HandleFunc("/api/slow", apiSlow)
    
    log.Println("Starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}

func apiOrders(w http.ResponseWriter, r *http.Request) {
    start := time.Now()
    activeRequests.Inc()
    defer activeRequests.Dec()
    
    time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
    
    if rand.Float64() < 0.05 {
        errorsTotal.WithLabelValues("api").Inc()
        w.WriteHeader(http.StatusInternalServerError)
        fmt.Fprintln(w, "Error")
        httpRequests.WithLabelValues(r.Method, "/api/orders", "500").Inc()
        httpDuration.WithLabelValues(r.Method, "/api/orders").Observe(time.Since(start).Seconds())
        return
    }
    
    w.WriteHeader(http.StatusOK)
    fmt.Fprintln(w, "Orders")
    httpRequests.WithLabelValues(r.Method, "/api/orders", "200").Inc()
    httpDuration.WithLabelValues(r.Method, "/api/orders").Observe(time.Since(start).Seconds())
}

func apiSlow(w http.ResponseWriter, r *http.Request) {
    start := time.Now()
    time.Sleep(2 * time.Second)
    w.WriteHeader(http.StatusOK)
    fmt.Fprintln(w, "Slow")
    httpRequests.WithLabelValues(r.Method, "/api/slow", "200").Inc()
    httpDuration.WithLabelValues(r.Method, "/api/slow").Observe(time.Since(start).Seconds())
}
```

**Запуск:**

```bash
go mod init metrics-demo
go mod tidy
go run main.go &

# Метрики
curl localhost:8080/metrics | grep -E "http_|go_goroutines"

# Запросы
for i in {1..100}; do curl -s localhost:8080/api/orders > /dev/null; done
for i in {1..5}; do curl -s localhost:8080/api/slow > /dev/null; done

# Проверить метрики
curl localhost:8080/metrics | grep http_requests_total
# http_requests_total{method="GET",path="/api/orders",status="200"} 95
# http_requests_total{method="GET",path="/api/orders",status="500"} 5
# http_requests_total{method="GET",path="/api/slow",status="200"} 5
```

### 💡 Практика: как правильно инструментировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Counter для requests/errors.**
2. **Histogram для latency.**
3. **Gauge для in-flight.**
4. **Runtime-метрики** (`collectors.NewGoCollector()`).
5. **Низкая cardinality** labels.

**👍 СТОИТ:**

6. **Middleware** для HTTP.
7. **Бизнес-метрики** (orders, users).
8. **Naming conventions.**

**❌ НЕ ДЕЛАЙ:**

9. **Не используй `path` с параметрами** (`/api/orders/123`). Нормализуй (`/api/orders/:id`).
10. **Не используй user_id, request_id как labels.**
11. **Не забывай про `/metrics` endpoint.**

### Где мы сейчас

Мы разобрали instrumentation. Теперь — **service discovery**.

---

## 22.5 Service discovery: как Prometheus находит цели

### 🔌 Проблема: targets меняются

В Kubernetes Pod'ы постоянно создаются и удаляются. IP меняются. Как Prometheus узнаёт, что опрашивать?

**Решение:** service discovery.

### 📊 Что такое service discovery

**Service discovery** — автоматическое обнаружение targets.

**Источники:**

- **Kubernetes** — Pod'ы, Service'ы, Endpoints.
- **Consul** — сервисы.
- **DNS** — SRV-записи.
- **EC2, GCE** — инстансы.
- **File** — статический список.
- **Static** — жёстко заданный.

### 🎯 Kubernetes SD

**Prometheus автоматически находит:**

- **Pods** — по аннотациям.
- **Services** — по labels.
- **Endpoints** — endpoints сервисов.
- **Nodes** — ноды.
- **Ingresses** — ingress.

**Пример: Pod SD с аннотациями:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
spec:
  containers:
    - name: myapp
      image: myapp:1.0
```

**Конфигурация Prometheus:**

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
```

**Что произойдёт:** Prometheus найдёт все Pod'ы с аннотацией `prometheus.io/scrape: "true"` и будет их опрашивать.

### 🎯 ServiceMonitor

**ServiceMonitor** — CRD Prometheus Operator.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: production
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_name]
          targetLabel: pod
```

**Что произойдёт:** Operator автоматически добавит конфигурацию в Prometheus.

**Плюсы:**

- **Декларативно.** В Git.
- **Оператор управляет.** Не нужно редактировать `prometheus.yml`.
- **Автоматически** подхватывает изменения.

### 🎯 PodMonitor

**Как ServiceMonitor, но для Pod'ов** (без Service).

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: myapp
  namespace: production
spec:
  selector:
    matchLabels:
      app: myapp
  podMetricsEndpoints:
    - port: metrics
      interval: 15s
```

**Когда использовать:** если у Pod'ов нет Service (например, DaemonSet).

### 🎯 Probe (blackbox)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Probe
metadata:
  name: example-http
spec:
  interval: 60s
  module: http_2xx
  prober:
    url: blackbox-exporter:9115
  targets:
    staticConfig:
      static:
        - https://example.com
```

### 🎯 Relabeling

**Relabeling** — переименование и фильтрация labels.

**Что можно делать:**

- **`keep`** — оставить только подходящие.
- **`drop`** — удалить.
- **`replace`** — заменить.
- **`labelmap`** — скопировать метки.
- **`labeldrop`** — удалить метку.

**Пример: keep только Pod'ы с аннотацией:**

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true
```

**Пример: переименовать метки:**

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_label_app]
    target_label: app
  - source_labels: [__meta_kubernetes_namespace]
    target_label: namespace
  - source_labels: [__meta_kubernetes_pod_name]
    target_label: pod
```

### 🔬 Практика: ServiceMonitor

```bash
# 1. Приложение с /metrics
# (из предыдущей подглавы)

# 2. Service с портом metrics
cat > service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: default
  labels:
    app: myapp
spec:
  selector:
    app: myapp
  ports:
    - name: metrics
      port: 8080
      targetPort: 8080
EOF

# 3. Deployment
cat > deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 2
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
          image: myapp:latest
          ports:
            - name: metrics
              containerPort: 8080
EOF

# 4. ServiceMonitor
cat > servicemonitor.yaml <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: default
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
EOF

kubectl apply -f service.yaml deployment.yaml servicemonitor.yaml

# 5. Проверить в Prometheus UI
# Status → Targets → myapp
```

### 💡 Практика: как правильно настраивать service discovery

**✅ ОБЯЗАТЕЛЬНО:**

1. **ServiceMonitor** для приложений.
2. **PodMonitor** для Pod'ов без Service.
3. **Probe** для blackbox.
4. **Relabeling** для чистых метрик.

**👍 СТОИТ:**

5. **Labels на Service** для селекции.
6. **Низкая cardinality** в relabeling.
7. **Тестировать** через Prometheus UI.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй static configs** в K8s.
9. **Не забывай про RBAC** для Prometheus.
10. **Не используй высокую cardinality** в labels.

### Где мы сейчас

Мы разобрали service discovery. Теперь — **PromQL**.

---

## 22.6 PromQL: язык запросов

### 🔌 Проблема: как запросить метрики

Метрики собраны. Как их анализировать?

**Решение:** PromQL.

### 📊 Что такое PromQL

**PromQL** — язык запросов Prometheus.

**Базовый синтаксис:**

```promql
metric_name{label="value"}
```

**Пример:**

```promql
http_requests_total{method="GET", status="200"}
```

### 🎯 Основы

**Vector (вектор):**

```promql
# Instant vector — текущее значение
http_requests_total

# Range vector — значения за период
http_requests_total[5m]
```

**Scalar (скаляр):**

```promql
42
```

**String (строка):**

```promql
"hello"
```

### 🎯 Селекторы

**По имени:**

```promql
http_requests_total
```

**По labels:**

```promql
http_requests_total{method="GET"}
http_requests_total{method="GET", status="200"}
http_requests_total{status=~"5.."}    # regex
http_requests_total{status!="200"}    # not equal
http_requests_total{status!~"5.."}    # not regex
```

### 🎯 Rate и increase

**Counter** — только растёт. Нужна **скорость**.

```promql
# Запросов в секунду за 5 минут
rate(http_requests_total[5m])

# Увеличение за 5 минут
increase(http_requests_total[5m])

# По labels
sum(rate(http_requests_total[5m])) by (method)
sum(rate(http_requests_total[5m])) by (method, status)
```

**Правило:** `rate()` для counter, `irate()` для быстрых изменений.

### 🎯 Агрегации

```promql
# Сумма
sum(http_requests_total)

# По labels
sum(http_requests_total) by (method)

# Без labels
sum(http_requests_total) without (pod)

# Среднее
avg(node_cpu_seconds_total)

# Максимум
max(node_memory_usage_bytes) by (instance)

# Минимум
min(...)

# Количество
count(up)

# Стандартное отклонение
stddev(...)
```

### 🎯 Операции

**Арифметические:**

```promql
http_requests_total * 2
http_requests_total / 100
```

**Сравнения:**

```promql
http_requests_total > 1000
node_memory_usage_bytes / node_memory_total_bytes > 0.9
```

**Логические:**

```promql
(http_requests_total > 1000) and (http_errors_total > 10)
... or ...
... unless ...
```

### 🎯 Histogram и quantiles

**Histogram метрика:**

```
http_request_duration_seconds_bucket{le="0.1"} 1234
http_request_duration_seconds_bucket{le="0.5"} 5678
http_request_duration_seconds_bucket{le="1"} 7890
http_request_duration_seconds_bucket{le="+Inf"} 8901
http_request_duration_seconds_sum 1234.56
http_request_duration_seconds_count 8901
```

**Percentiles:**

```promql
# p50
histogram_quantile(0.5, rate(http_request_duration_seconds_bucket[5m]))

# p95
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# p99
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

**По labels:**

```promql
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, path)
)
```

### 🎯 Практические запросы

**Error rate:**

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

**RPS по endpoint:**

```promql
sum(rate(http_requests_total[1m])) by (path)
```

**Latency p99 по endpoint:**

```promql
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, path)
)
```

**CPU usage:**

```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Memory usage:**

```promql
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
/
node_memory_MemTotal_bytes
* 100
```

**Disk usage:**

```promql
(node_filesystem_size_bytes - node_filesystem_free_bytes)
/
node_filesystem_size_bytes
* 100
```

**Pod restarts:**

```promql
rate(kube_pod_container_status_restarts_total[1h])
```

**Deployment replicas mismatch:**

```promql
kube_deployment_spec_replicas
!=
kube_deployment_status_replicas_available
```

### 🎯 PromQL в Grafana

**Variables:**

```promql
label_values(http_requests_total, path)
label_values(up, job)
```

**Dashboard queries:**

```promql
# С фильтром по переменной
sum(rate(http_requests_total{path="$path"}[5m])) by (status)
```

### 🔬 Практика: PromQL

```bash
# 1. Открыть Prometheus UI
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# 2. Простые запросы
up
up{job="kubelet"}
node_load1
node_memory_MemAvailable_bytes

# 3. Rate
rate(node_cpu_seconds_total[5m])

# 4. Агрегация
sum(rate(node_cpu_seconds_total[5m])) by (instance)

# 5. CPU usage
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 6. Memory usage %
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100

# 7. Error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# 8. p99 latency
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```

### 💡 Практика: как правильно писать PromQL

**✅ ОБЯЗАТЕЛЬНО:**

1. **`rate()` для counter.**
2. **`histogram_quantile()` для percentiles.**
3. **`by ()` для группировки.**
4. **`{label="value"}` для фильтрации.**

**👍 СТОИТ:**

5. **`without ()`** для исключения labels.
6. **Переменные в Grafana** для динамики.
7. **Recording rules** для сложных запросов.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй `rate()` для gauge.**
9. **Не забывай про `[5m]` диапазон.**
10. **Не пиши слишком сложные запросы** без testing.

### Где мы сейчас

Мы разобрали PromQL. Теперь — **recording rules**.

---

## 22.7 Recording rules: предвычисления

### 🔌 Проблема: сложные запросы медленные

Запрос:

```promql
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{job="api"}[5m])) by (le, path, method)
)
```

**Проблема:** вычисляется при каждом запросе. Медленно для дашбордов.

**Решение:** recording rules.

### 📊 Что такое recording rules

**Recording rules** — предвычисление запросов и сохранение как новых метрик.

**Пример:**

```yaml
groups:
  - name: api.rules
    interval: 30s
    rules:
      - record: job:http_request_duration_seconds:p99
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job, path)
          )
```

**Что произойдёт:** каждые 30 секунд вычисляется новый metric `job:http_request_duration_seconds:p99`. Дашборды используют его — быстро.

### 🎯 Naming convention

**Формат:** `level:metric:operations`

- **level** — уровень агрегации (`job`, `instance`).
- **metric** — имя метрики.
- **operations** — операции (`rate`, `sum`, `p99`).

**Примеры:**

```
job:http_requests:rate5m
job:http_errors:rate5m
job:http_request_duration_seconds:p99
instance:node_cpu:usage_percent
```

### 🎯 Recording rules в Prometheus Operator

**PrometheusRule CRD:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-rules
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: api.recording
      interval: 30s
      rules:
        - record: job:http_requests:rate5m
          expr: sum(rate(http_requests_total[5m])) by (job)
        
        - record: job:http_errors:rate5m
          expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
        
        - record: job:http_error_rate:ratio5m
          expr: |
            job:http_errors:rate5m
            /
            job:http_requests:rate5m
        
        - record: job:http_request_duration_seconds:p99
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
            )
        
        - record: job:http_request_duration_seconds:p95
          expr: |
            histogram_quantile(0.95,
              sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
            )
```

### 🎯 Зачем recording rules

**1. Производительность.**

Предвычисление быстрее, чем вычисление при каждом запросе.

**2. Дашборды.**

Быстрые графики.

**3. Алерты.**

Быстрая проверка условий.

**4. Упрощение.**

Сложные запросы → простые метрики.

### 🎯 Правила

**1. Не создавай слишком много.**

Каждая rule — новая time series. Cardinality.

**2. Interval ≥ scrape_interval.**

Если scrape каждые 15 секунд, rule каждые 30 секунд.

**3. Не дублируй.**

Если можно использовать raw metric — используй.

**4. Тестируй.**

Проверяй в Prometheus UI до деплоя.

### 🔬 Практика: recording rules

```yaml
cat > recording-rules.yaml <<'EOF'
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: node-recording-rules
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: node.recording
      interval: 30s
      rules:
        - record: instance:node_cpu:usage_percent
          expr: |
            100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)
        
        - record: instance:node_memory:usage_percent
          expr: |
            (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
            /
            node_memory_MemTotal_bytes
            * 100
        
        - record: instance:node_filesystem:usage_percent
          expr: |
            (node_filesystem_size_bytes - node_filesystem_free_bytes)
            /
            node_filesystem_size_bytes
            * 100
EOF

kubectl apply -f recording-rules.yaml

# Проверить через минуту
# В Prometheus UI:
# instance:node_cpu:usage_percent
# instance:node_memory:usage_percent
```

### 💡 Практика: как правильно использовать recording rules

**✅ ОБЯЗАТЕЛЬНО:**

1. **Для сложных запросов** (histogram_quantile).
2. **Naming convention** (`level:metric:operations`).
3. **Interval ≥ scrape_interval.**
4. **Тестировать** перед деплоем.

**👍 СТОИТ:**

5. **Recording rules для алертов.**
6. **Документация** в PrometheusRule.

**❌ НЕ ДЕЛАЙ:**

7. **Не дублируй** raw metrics.
8. **Не создавай слишком много.**
9. **Не забывай про cardinality.**

### Где мы сейчас

Мы разобрали recording rules. Теперь — **Grafana**.

---

## 22.8 Grafana: дашборды

### 🔌 Проблема: как визуализировать метрики

Метрики есть. PromQL работает. Но как это **показать** команде?

**Решение:** Grafana.

### 📊 Что такое Grafana

**Grafana** — open-source платформа для визуализации.

**Что даёт:**

- **Dashboards** — графики, таблицы.
- **Multiple datasources** — Prometheus, Loki, Jaeger, ...
- **Alerting** — встроенный.
- **Templating** — переменные.
- **Sharing** — ссылки, снапшоты.

### 🎯 Dashboard структура

**Dashboard** состоит из **panels** (панелей).

**Типы panels:**

- **Time series** — график по времени.
- **Stat** — одно значение.
- **Gauge** — круглый индикатор.
- **Bar chart** — столбцы.
- **Table** — таблица.
- **Heatmap** — heatmap.
- **Logs** — логи.
- **Text** — текст.

### 🎯 Пример дашборда для API

**1. RPS:**

```promql
sum(rate(http_requests_total[5m])) by (path)
```

**2. Error rate:**

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

**3. Latency p50/p95/p99:**

```promql
histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```

**4. In-flight requests:**

```promql
sum(http_requests_in_flight)
```

**5. Goroutines:**

```promql
go_goroutines{job="myapp"}
```

**6. Memory:**

```promql
go_memstats_alloc_bytes{job="myapp"}
```

**7. Top endpoints:**

```promql
topk(10, sum(rate(http_requests_total[5m])) by (path))
```

### 🎯 Variables

**Variables** — переменные для динамических дашбордов.

**Query variable:**

```
label_values(http_requests_total, path)
```

**Использование:**

```promql
sum(rate(http_requests_total{path="$path"}[5m])) by (status)
```

### 🎯 Provisioning

**Provisioning** — автоматическое создание дашбордов из файлов.

**datasources.yaml:**

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      timeInterval: 15s
```

**dashboards.yaml:**

```yaml
apiVersion: 1
providers:
  - name: 'default'
    folder: 'General'
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

**Дашборды — JSON-файлы:**

```json
{
  "title": "API Dashboard",
  "panels": [
    {
      "title": "RPS",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total[5m])) by (path)",
          "legendFormat": "{{path}}"
        }
      ]
    }
  ]
}
```

### 🎯 Best practices

**1. USE Method дашборд.**

- **Utilization** — % использования.
- **Saturation** — очередь, нагрузка.
- **Errors** — ошибки.

**2. RED Method для сервисов.**

- **Rate** — RPS.
- **Errors** — error rate.
- **Duration** — latency.

**3. Four Golden Signals (Google SRE).**

- **Latency** — время ответа.
- **Traffic** — RPS.
- **Errors** — ошибки.
- **Saturation** — использование.

**4. Организация.**

- **Overview** — верхний уровень.
- **Service** — по сервису.
- **Infrastructure** — по инфраструктуре.
- **Business** — бизнес-метрики.

### 🎯 Grafana в kube-prometheus-stack

**Что включено:**

- **Дашборды для K8s:** Nodes, Pods, Deployments.
- **Дашборды для Prometheus:** scrape targets, rules.
- **Дашборды для Alertmanager.**
- **Дашборды для K8s resources.**

**Доступ:**

```bash
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
# Login: admin / admin123 (или пароль из secret)
```

### 🔬 Практика: Grafana

```bash
# 1. Port-forward
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# 2. Открыть http://localhost:3000
# Login: admin / <пароль>

# 3. Проверить datasources
# Configuration → Data Sources → Prometheus

# 4. Создать dashboard
# + → Dashboard → Add visualization

# 5. Query: RPS
# sum(rate(http_requests_total[5m])) by (path)

# 6. Сохранить
# Save dashboard → "API Dashboard"

# 7. Импортировать готовые
# Dashboards → Import
# 315 (Kubernetes cluster monitoring)
# 6417 (Kubernetes Deployment metrics)
# 8588 (Kubernetes Pod metrics)
```

### 💡 Практика: как правильно строить дашборды

**✅ ОБЯЗАТЕЛЬНО:**

1. **RED/USE метод.**
2. **Variables** для динамики.
3. **Provisioning** для версионирования.
4. **Организация** по папкам.

**👍 СТОИТ:**

5. **Готовые дашборды** из Grafana.com.
6. **Annotations** для событий (деплои).
7. **Links** между дашбордами.

**❌ НЕ ДЕЛАЙ:**

8. **Не создавай слишком много дашбордов.** Один хороший лучше десяти.
9. **Не забывай про time range.** Автоматический.
10. **Не игнорируй units.** Seconds, bytes, percent.

### Где мы сейчас

Мы разобрали Grafana. Теперь — **Alertmanager**.

---

## 22.9 Alertmanager: маршрутизация алертов

### 🔌 Проблема: алерты в Slack не структурированы

200 алертов в Slack. Все в один канал. Нет приоритетов. Нет группировки.

**Решение:** Alertmanager.

### 📊 Что такое Alertmanager

**Alertmanager** — компонент Prometheus для управления алертами.

**Что делает:**

- **Дедупликация** — убирает дубликаты.
- **Группировка** — объединяет связанные алерты.
- **Маршрутизация** — куда отправить.
- **Подавление** — не отправлять, если есть более важный.
- **Отправка** — Slack, PagerDuty, Email, Webhook.

### 🎯 Архитектура

```
┌──────────────────┐
│   Prometheus     │
│                  │
│  Alerting Rules  │
│  (что алертить)  │
└────────┬─────────┘
         │
         │ Alerts
         ▼
┌──────────────────┐
│  Alertmanager    │
│                  │
│  - Deduplication │
│  - Grouping      │
│  - Routing       │
│  - Silencing     │
│  - Inhibition    │
└────────┬─────────┘
         │
         │ Notifications
         ▼
┌──────────────────┐
│  Slack / PagerDuty / Email
└──────────────────┘
```

### 🎯 Alerting Rules

**Alerting rules** — в Prometheus. Определяют, что алертить.

```yamlgroups:
  - name: api.alerts
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
          > 0.05
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate on {{ $labels.job }}"
          description: "Error rate is {{ $value | humanizePercentage }}"
          runbook_url: "https://wiki.example.com/runbooks/high-error-rate"
```

**Ключевые поля:**

- **`alert`** — имя алерта.
- **`expr`** — условие.
- **`for`** — сколько времени условие должно быть true.
- **`labels`** — labels алерта.
- **`annotations`** — человекочитаемая информация.

### 🎯 Alertmanager конфигурация

```yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/...'

route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true
    - match:
        severity: critical
      receiver: 'slack-critical'
    - match:
        severity: warning
      receiver: 'slack-warning'
    - match:
        team: backend
      receiver: 'slack-backend'

receivers:
  - name: 'default'
    slack_configs:
      - channel: '#alerts'
  
  - name: 'slack-critical'
    slack_configs:
      - channel: '#alerts-critical'
        title: '🚨 {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
  
  - name: 'slack-warning'
    slack_configs:
      - channel: '#alerts-warning'
  
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'xxx'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']
```

### 🎯 Ключевые настройки

**Grouping:**

- **`group_by`** — по каким labels группировать.
- **`group_wait`** — сколько ждать перед отправкой первой группы.
- **`group_interval`** — интервал между отправками группы.
- **`repeat_interval`** — как часто повторять.

**Что даёт:**

- 10 Pod'ов с одной проблемой → 1 алерт вместо 10.
- Группировка по сервису.

**Inhibition:**

- **Критичный алерт подавляет warning.**
- Если сервер недоступен — не нужно алертить про CPU.

**Silencing:**

- **Временно отключить алерт.**
- Во время maintenance.

### 🎯 Маршрутизация

**Пример: разные команды — разные каналы:**

```yaml
route:
  receiver: 'default'
  routes:
    - match:
        team: backend
      receiver: 'slack-backend'
    - match:
        team: frontend
      receiver: 'slack-frontend'
    - match:
        team: data
      receiver: 'slack-data'
```

**Пример: severity → разные каналы:**

```yaml
routes:
  - match:
      severity: critical
    receiver: 'pagerduty'
    continue: true
  - match:
      severity: critical
    receiver: 'slack-critical'
  - match:
      severity: warning
    receiver: 'slack-warning'
```

### 🎯 PrometheusRule в Operator

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: api.alerts
      interval: 30s
      rules:
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total[5m]))
            > 0.05
          for: 5m
          labels:
            severity: critical
            team: backend
          annotations:
            summary: "High error rate on {{ $labels.job }}"
            description: "Error rate is {{ $value | humanizePercentage }}"
        
        - alert: HighLatency
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
            ) > 1
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High p99 latency on {{ $labels.job }}"
```

### 🔬 Практика: Alertmanager

```bash
# 1. Проверить Alertmanager
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093

# 2. Открыть http://localhost:9093
# Alerts — текущие алерты
# Silences — silenced
# Status — конфигурация

# 3. Создать тестовый алерт
cat > test-alert.yaml <<'EOF'
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: test-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: test
      rules:
        - alert: TestAlert
          expr: vector(1)
          for: 1m
          labels:
            severity: warning
            team: test
          annotations:
            summary: "This is a test alert"
            description: "Testing Alertmanager"
EOF

kubectl apply -f test-alert.yaml

# 4. Через минуту — алерт в Alertmanager UI

# 5. Проверить конфигурацию
kubectl get secret -n monitoring alertmanager-prometheus-kube-prometheus-alertmanager -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d
```

### 💡 Практика: как правильно настраивать Alertmanager

**✅ ОБЯЗАТЕЛЬНО:**

1. **Grouping** по alertname + service.
2. **Маршрутизация** по severity и team.
3. **Inhibition** для подавления.
4. **Repeat interval** разумный (4-12 часов).

**👍 СТОИТ:**

5. **PagerDuty** для critical.
6. **Slack** для warning.
7. **Runbook URLs** в annotations.

**❌ НЕ ДЕЛАЙ:**

8. **Не отправляй всё в один канал.**
9. **Не забывай про repeat.** Слишком часто — шум.
10. **Не игнорируй silence.** Maintenance.

### Где мы сейчас

Мы разобрали Alertmanager. Теперь — **SLO, SLI, SLA**.

---

## 22.10 SLO, SLI, SLA и error budget

### 🔌 Проблема: как определить «нормально»

Сервис работает. Но **нормально** ли? 99% uptime — это хорошо или плохо? 200ms latency — это хорошо?

Без **цели** невозможно сказать.

**Решение:** SLO.

### 📊 Три термина

**SLI (Service Level Indicator)** — метрика.

**Примеры:**

- Availability (процент успешных запросов).
- Latency (процент запросов быстрее X ms).
- Throughput (RPS).
- Error rate.

**SLO (Service Level Objective)** — цель для SLI.

**Примеры:**

- 99.9% запросов успешны за 30 дней.
- 99% запросов быстрее 200ms.
- 99.99% uptime.

**SLA (Service Level Agreement)** — контракт с клиентом.

**Примеры:**

- 99.9% uptime, иначе штраф.
- Обычно SLA ниже SLO.

### 🎯 Пример

```
SLI: error_rate = errors / total_requests
SLO: error_rate < 0.1% за 30 дней
SLA: error_rate < 1% за 30 дней, иначе возврат денег
```

**SLO строже SLA.** Внутренняя цель выше, чем обязательство.

### 🎯 Error Budget

**Error budget** — сколько ошибок можно допустить.

**Формула:**

```
Error budget = (1 - SLO) × total_requests
```

**Пример:**

- SLO: 99.9%.
- За 30 дней: 1 000 000 запросов.
- Error budget: 0.1% × 1 000 000 = 1000 ошибок.

**Как использовать:**

- **Budget не исчерпан** → можно рисковать (деплоить).
- **Budget исчерпан** → freeze на изменения, все силы на надёжность.

### 🎯 Как определить SLO

**1. Начать с текущего состояния.**

Измерь SLI за последние 30 дней. Это baseline.

**2. Поставить цель.**

Чуть выше текущего. Не 100% — это невозможно.

**3. Учитывать бизнес.**

- **99.9%** — большинство сервисов.
- **99.99%** — критичные (платежи).
- **99%** — внутренние.

**4. Реалистично.**

- **99.9%** = 43 минуты простоя в месяц.
- **99.99%** = 4.3 минуты.
- **99.999%** = 26 секунд.

**Каждый «девять» — в 10 раз дороже.**

### 🎯 SLO в Prometheus

**Recording rules:**

```yaml
groups:
  - name: slo.rules
    interval: 30s
    rules:
      - record: slo:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m]))
      
      - record: slo:http_errors:rate5m
        expr: sum(rate(http_requests_total{status=~"5.."}[5m]))
      
      - record: slo:http_error_rate:ratio5m
        expr: slo:http_errors:rate5m / slo:http_requests:rate5m
      
      - record: slo:http_error_budget:ratio
        expr: 1 - 0.999    # 0.1% error budget
```

**Alerting rules:**

```yaml
- alert: SLOErrorBudgetBurnRate
  expr: |
    slo:http_error_rate:ratio5m
    /
    0.001    # error budget = 0.1%
    > 14.4   # burn rate > 14.4x
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Error budget burning too fast"
```

**Burn rate** — скорость исчерпания error budget. 14.4x означает, что budget исчерпается за 2 дня (вместо 30).

### 🎯 Multi-window burn rate

**Google SRE** рекомендует multi-window:

```yaml
- alert: ErrorBudgetBurnFast
  expr: |
    (
      slo:http_error_rate:ratio5m > (14.4 * 0.001)
      and
      slo:http_error_rate:ratio1h > (14.4 * 0.001)
    )
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Fast error budget burn"

- alert: ErrorBudgetBurnSlow
  expr: |
    (
      slo:http_error_rate:ratio30m > (6 * 0.001)
      and
      slo:http_error_rate:ratio6h > (6 * 0.001)
    )
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "Slow error budget burn"
```

**Что даёт:**

- **Fast burn** — критичный, алерт сразу.
- **Slow burn** — warning, алерт через 15 минут.

**Меньше false positives**, чем простой threshold.

### 🎯 SLO дашборд в Grafana

**Панели:**

1. **SLI** — текущий.
2. **SLO** — цель.
3. **Error budget remaining** — %.
4. **Burn rate** — скорость.
5. **Time to exhaustion** — когда исчерпается.

**Query:**

```promql
# Error budget remaining
1 - (
  sum(rate(http_requests_total{status=~"5.."}[30d]))
  /
  sum(rate(http_requests_total[30d]))
) / 0.001
```

### 🔬 Практика: SLO

```yaml
cat > slo-rules.yaml <<'EOF'
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-rules
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: slo.recording
      interval: 30s
      rules:
        - record: slo:http_requests:rate5m
          expr: sum(rate(http_requests_total[5m]))
        
        - record: slo:http_errors:rate5m
          expr: sum(rate(http_requests_total{status=~"5.."}[5m]))
        
        - record: slo:http_error_rate:ratio5m
          expr: slo:http_errors:rate5m / slo:http_requests:rate5m
    
    - name: slo.alerts
      interval: 30s
      rules:
        - alert: HighErrorBudgetBurnRate
          expr: |
            slo:http_error_rate:ratio5m > 0.0144
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Error budget burning fast"
            description: "Error rate: {{ $value | humanizePercentage }}"
EOF

kubectl apply -f slo-rules.yaml
```

### 💡 Практика: как правильно определять SLO

**✅ ОБЯЗАТЕЛЬНО:**

1. **Начать с baseline.**
2. **Реалистичная цель** (99.9%, не 100%).
3. **Error budget** для решений.
4. **Multi-window burn rate** для алертов.

**👍 СТОИТ:**

5. **SLO на уровне сервиса.**
6. **Дашборд** с SLI/SLO/budget.
7. **Регулярный review** SLO.

**❌ НЕ ДЕЛАЙ:**

8. **Не ставь 100%.** Невозможно.
9. **Не используй SLA как SLO.** SLO строже.
10. **Не забывай про error budget.** Это инструмент.

### Где мы сейчас

Мы разобрали SLO. Теперь — **алерты, которые не шумят**.

---

## 22.11 Алерты, которые не шумят

### 🔌 Проблема: alert fatigue

200 алертов. Большинство — неважные. Люди перестают реагировать.

**Решение:** правильные алерты.

### 📊 Принципы хороших алертов

**1. Actionable.**

Каждый алерт требует действия. Если можно игнорировать — удали.

**2. Symptom-based, не cause-based.**

- **Плохо:** CPU > 80%. (причина)
- **Хорошо:** Latency p99 > 1s. (симптом)

CPU может быть 90% и всё работать. Latency — прямой симптом проблемы.

**3. SLO-based.**

Алерты на error budget burn rate, не на отдельные метрики.

**4. Severity.**

- **Critical** — будит ночью.
- **Warning** — рабочее время.
- **Info** — дашборд.

**5. Runbook.**

К каждому алерту — что делать.

**6. Контекст.**

- Ссылка на дашборд.
- Ссылка на логи.
- Ссылка на runbook.

### 🎯 Multi-window, multi-burn-rate

**Google SRE** подход:

```yaml
# Fast burn — критично
- alert: ErrorBudgetBurnFast
  expr: |
    (
      slo:http_error_rate:ratio5m > (14.4 * 0.001)
      and
      slo:http_error_rate:ratio1h > (14.4 * 0.001)
    )
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Fast error budget burn - 2 days to exhaustion"
    dashboard_url: "..."
    runbook_url: "..."

# Slow burn — warning
- alert: ErrorBudgetBurnSlow
  expr: |
    (
      slo:http_error_rate:ratio30m > (6 * 0.001)
      and
      slo:http_error_rate:ratio6h > (6 * 0.001)
    )
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "Slow error budget burn - 5 days to exhaustion"
```

**Что даёт:**

- **Fast burn** — 2% budget за час → алерт.
- **Slow burn** — 5% budget за 6 часов → алерт.
- **Меньше false positives** — 2 окна.

### 🎯 Группировка

**Alertmanager группирует:**

```yaml
route:
  group_by: ['alertname', 'service', 'namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
```

**Что даёт:**

- 10 Pod'ов с одной проблемой → 1 алерт.
- Первый алерт через 30 секунд.
- Повтор через 4 часа.

### 🎯 Подавление (inhibition)

**Inhibition rules:**

```yaml
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'service']
  
  - source_match:
      alertname: 'NodeDown'
    target_match_re:
      alertname: '.*'
    equal: ['node']
```

**Что даёт:**

- Если сервер недоступен — не алертить про CPU, memory.
- Если critical — подавить warning по тому же сервису.

### 🎯 Silences

**Silence** — временное отключение.

```bash
# Через UI
# Silences → New Silence
# Matchers: alertname=TestAlert
# Duration: 2h
# Comment: Testing

# Через CLI
amtool silence add alertname=TestAlert --duration=2h --comment="Testing"
```

**Когда использовать:**

- **Maintenance.**
- **Известная проблема.**
- **Тестирование.**

### 🎯 Anti-patterns

**❌ Плохие алерты:**

1. **CPU > 80%.** Причина, не симптом.
2. **Memory > 90%.** Не actionable без контекста.
3. **Disk > 85%.** Может быть нормально.
4. **Pod restarted.** Единичный — норма.
5. **Certificate expiring 30 days.** Слишком рано.

**✅ Хорошие алерты:**

1. **Error rate > 5% for 5m.** Прямой симптом.
2. **Latency p99 > 1s for 10m.** Влияет на пользователей.
3. **Error budget burn > 14.4x.** SLO-based.
4. **Pod restarts > 3 in 1h.** Реальная проблема.
5. **Certificate expiring 7 days.** Действие нужно.

### 🎯 Review алертов

**Регулярно (раз в месяц):**

- Какие алерты срабатывали?
- Были ли actionable?
- Были ли false positives?
- Что можно удалить?

**Метрики:**

- **Alert count** — сколько алертов.
- **Actionable rate** — % actionable.
- **MTTA** — mean time to acknowledge.
- **MTTR** — mean time to resolve.

### 🔬 Практика: хорошие алерты

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: good-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: application.alerts
      interval: 30s
      rules:
        # Хорошо: symptom-based
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
            /
            sum(rate(http_requests_total[5m])) by (service)
            > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High error rate on {{ $labels.service }}"
            description: "Error rate is {{ $value | humanizePercentage }}"
            dashboard_url: "https://grafana.example.com/d/api"
            runbook_url: "https://wiki.example.com/runbooks/high-error"
        
        # Хорошо: SLO-based
        - alert: ErrorBudgetBurnFast
          expr: |
            (
              slo:http_error_rate:ratio5m > (14.4 * 0.001)
              and
              slo:http_error_rate:ratio1h > (14.4 * 0.001)
            )
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Fast error budget burn"
        
        # Хорошо: latency, влияет на пользователей
        - alert: HighLatency
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
            ) > 1
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High p99 latency on {{ $labels.service }}"
        
        # Плохо: cause-based, не actionable
        # - alert: HighCPU
        #   expr: node_cpu_usage > 80
```

### 💡 Практика: как правильно писать алерты

**✅ ОБЯЗАТЕЛЬНО:**

1. **Actionable.**
2. **Symptom-based.**
3. **SLO-based для критичных.**
4. **Runbook URL.**
5. **Dashboard URL.**

**👍 СТОИТ:**

6. **Multi-window burn rate.**
7. **Inhibition rules.**
8. **Регулярный review.**

**❌ НЕ ДЕЛАЙ:**

9. **Не алерти на cause.**
10. **Не игнорируй false positives.**
11. **Не забывай про severity.**
12. **Не создавай 200 алертов.**

### Где мы сейчас

Мы разобрали алерты. Теперь — **масштабирование Prometheus**.

---

## 22.12 Масштабирование Prometheus

### 🔌 Проблема: Prometheus не масштабируется

Один Prometheus:

- **Не распределённый.** Одна БД.
- **Ограничен** CPU, RAM, диском.
- **Не масштабируется горизонтально.**
- **Единая точка отказа.**

**Для больших кластеров** — нужны решения.

### 📊 Проблемы одного Prometheus

**1. Объём метрик.**

Миллионы time series. RAM кончается.

**2. Retention.**

30 дней × миллионы series = терабайты.

**3. Multi-cluster.**

Несколько кластеров — несколько Prometheus.

**4. HA.**

Prometheus упал — потеря данных.

### 🎯 Решения

**1. Federation.**

**Что:** Prometheus верхнего уровня опрашивает Prometheus нижнего уровня.

**Плюсы:**

- **Просто.**
- **Иерархия.**

**Минусы:**

- **Потеря деталей** (только агрегированные).
- **Сложно настраивать.**

**2. Thanos.**

**Что:** расширение Prometheus для долгосрочного хранения и глобальных запросов.

**Компоненты:**

- **Sidecar** — рядом с каждым Prometheus. Отправляет данные в S3.
- **Store Gateway** — читает из S3.
- **Query** — глобальный запрос.
- **Compactor** — компактификация.
- **Receiver** — приём от remote write.

**Плюсы:**

- **Долгосрочное хранение** (S3 — дёшево).
- **Глобальные запросы** через несколько Prometheus.
- **HA** (два Prometheus + deduplication).
- **Downsampling.**

**Минусы:**

- **Сложно.**
- **Много компонентов.**

**3. VictoriaMetrics.**

**Что:** альтернатива Prometheus. Совместим с PromQL.

**Плюсы:**

- **Высокая производительность.**
- **Меньше RAM.**
- **Лучше сжатие.**
- **Single binary или cluster.**
- **Проще Thanos.**

**Минусы:**

- **Не Prometheus.**
- **Меньше экосистема.**

**4. Grafana Mimir.**

**Что:** горизонтально масштабируемый Prometheus от Grafana Labs.

**Плюсы:**

- **Масштабируется** на миллиарды series.
- **Multi-tenancy.**
- **Long-term storage.**
- **Совместим с PromQL.**

**Минусы:**

- **Сложно.**
- **Много компонентов.**

### 📊 Сравнение

| Решение | Сложность | Масштаб | Retention | Стоимость |
|:---|:---|:---|:---|:---|
| **Prometheus** | Низкая | Средний | Короткий | Низкая |
| **Thanos** | Высокая | Большой | Долгий | Средняя |
| **VictoriaMetrics** | Средняя | Большой | Долгий | Низкая |
| **Mimir** | Высокая | Очень большой | Долгий | Средняя |

### 🎯 Когда что

**Prometheus:**

- **Небольшой кластер** (< 100 нод).
- **Короткий retention** (15-30 дней).
- **Один кластер.**

**Thanos:**

- **Несколько кластеров.**
- **Долгосрочное хранение.**
- **Глобальные запросы.**
- **HA.**

**VictoriaMetrics:**

- **Большие объёмы.**
- **Хочется проще Thanos.**
- **PromQL совместимость.**

**Mimir:**

- **Очень большие объёмы.**
- **Multi-tenancy.**
- **Managed (Grafana Cloud).**

### 🎯 Remote write

**Remote write** — отправка метрик в удалённый backend.

```yaml
prometheus:
  prometheusSpec:
    remoteWrite:
      - url: http://victoriametrics:8428/api/v1/write
        queueConfig:
          maxSamplesPerSend: 10000
          maxShards: 200
```

**Что даёт:**

- **Локальный Prometheus** — короткий retention.
- **Remote** — долгосрочное хранение.

### 🎯 HA Prometheus

**Проблема:** один Prometheus — SPOF.

**Решение:** два Prometheus + deduplication.

```
┌──────────────┐  ┌──────────────┐
│ Prometheus A │  │ Prometheus B │
│              │  │              │
│ Один и тот же config
└──────┬───────┘  └───────┬──────┘
       │                  │
       │ remote write     │
       ▼                  ▼
┌──────────────────────────────────┐
│  Thanos / VictoriaMetrics        │
│  Deduplication                   │
└──────────────────────────────────┘
```

**Что даёт:** если один упал — второй работает.

### 🔬 Практика: Thanos

```bash
# 1. Установить Thanos
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install thanos bitnami/thanos \
  --namespace monitoring \
  --set objstoreConfig=... \
  --set query.replicaLabel=...

# 2. Или через kube-prometheus-stack с Thanos sidecar
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.thanos.create=true \
  --set prometheus.thanos.objectStorageConfig.existingSecret=thanos-objstore
```

### 💡 Практика: как правильно масштабировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Начать с одного Prometheus.**
2. **Remote write** для долгосрочного хранения.
3. **HA** для критичных.

**👍 СТОИТ:**

4. **VictoriaMetrics** для простоты.
5. **Thanos** для multi-cluster.
6. **Managed** (Grafana Cloud, AWS Managed Prometheus) для простоты.

**❌ НЕ ДЕЛАЙ:**

7. **Не масштабируй без необходимости.**
8. **Не забывай про cardinality.**
9. **Не используй federation для всего.**

### Где мы сейчас

Мы разобрали масштабирование. Теперь — **cardinality**.

---

## 22.13 Cardinality: главный враг

### 🔌 Проблема: слишком много метрик

Ты добавил label `user_id` к метрике. Миллион пользователей. **Миллион time series.** Prometheus упал.

**Cardinality** — главный враг Prometheus.

### 📊 Что такое cardinality

**Cardinality** — количество уникальных комбинаций labels.

**Формула:**

```
Cardinality = произведение количества уникальных значений каждого label
```

**Пример:**

```
http_requests_total{method, path, status}
methods = 5 (GET, POST, PUT, DELETE, PATCH)
paths = 100
statuses = 10 (200, 201, 400, 401, 403, 404, 500, 502, 503, 504)

Cardinality = 5 × 100 × 10 = 5000 time series
```

**5000 — нормально.** Prometheus справится.

**Плохой пример:**

```
http_requests_total{user_id}
users = 1 000 000

Cardinality = 1 000 000 time series
```

**1 миллион — катастрофа.** RAM кончится.

### 🎯 Что вызывает высокую cardinality

**Плохие labels:**

- **`user_id`** — миллионы.
- **`request_id`** — уникально для каждого запроса.
- **`session_id`** — миллионы.
- **`email`** — миллионы.
- **`timestamp`** — бесконечно.
- **`path` с параметрами** (`/api/orders/123`) — миллионы.

**Хорошие labels:**

- **`method`** — 5-10.
- **`status`** — 10-20.
- **`path`** (нормализованный, `/api/orders/:id`) — сотни.
- **`service`** — десятки.
- **`environment`** — 3-5.
- **`version`** — десятки.

### 🎯 Как обнаружить

**1. Prometheus UI:**

```
Status → TSDB Status
```

**Показывает:**

- **Top 10 series by metric name.**
- **Top 10 labels by number of values.**
- **Total series.**

**2. PromQL:**

```promql
# Количество series для метрики
count({__name__="http_requests_total"})

# Количество уникальных значений label
count(count by (path) (http_requests_total))

# Топ-10 метрик по количеству series
topk(10, count by (__name__) ({__name__=~".+"}))
```

**3. Алерт:**

```yaml
- alert: HighCardinality
  expr: |
    count({__name__=~".+"}) > 1000000
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Prometheus has > 1M series"
```

### 🎯 Как снизить cardinality

**1. Убрать плохие labels.**

```go
// ❌ Плохо
httpRequests.WithLabelValues("GET", "/api/orders/123", userID, "200").Inc()

// ✅ Хорошо
httpRequests.WithLabelValues("GET", "/api/orders/:id", "200").Inc()
```

**2. Нормализовать path.**

```go
// Нормализация path
func normalizePath(path string) string {
    // /api/orders/123 → /api/orders/:id
    re := regexp.MustCompile(`/[0-9]+`)
    return re.ReplaceAllString(path, "/:id")
}
```

**3. Агрегировать.**

```promql
# Вместо детальных labels
sum(rate(http_requests_total[5m])) by (service, method, status)

# Отбросить высокую cardinality
sum(rate(http_requests_total[5m])) without (path, instance)
```

**4. Recording rules.**

Предвычисление агрегатов с меньшей cardinality.

**5. Не использовать labels для бизнес-данных.**

- `user_id` → логи.
- `request_id` → трейсы.
- `email` → логи.

### 🎯 Cardinality budget

**Установи бюджет:**

- **Prometheus:** 1-2 миллиона series максимум.
- **На метрику:** не больше 10 000 series.
- **На label:** не больше 100 значений.

**Алерт при превышении.**

### 🎯 Prometheus limits

**`sample_limit` в scrape config:**

```yaml
scrape_configs:
  - job_name: 'myapp'
    sample_limit: 10000
```

**Что даёт:** если metric превышает лимит — scrape fails.

**`label_limit` в scrape config:**

```yaml
scrape_configs:
  - job_name: 'myapp'
    label_limit: 30
    label_name_length_limit: 100
    label_value_length_limit: 200
```

**Что даёт:** ограничение количества labels и их длины.

### 🔬 Практика: cardinality

```promql
# 1. Общее количество series
count({__name__=~".+"})

# 2. Топ-10 метрик
topk(10, count by (__name__) ({__name__=~".+"}))

# 3. Уникальные значения label
count(count by (path) (http_requests_total))

# 4. Топ-10 labels по количеству значений
topk(10, count(count by (__name__, path) ({__name__=~".+"})))

# 5. Series по job
count by (job) ({__name__=~".+"})
```

**В Prometheus UI:**

```
Status → TSDB Status
```

**Увидишь:**

- Total series.
- Top 10 by metric name.
- Top 10 by label count.

### 💡 Практика: как правильно управлять cardinality

**✅ ОБЯЗАТЕЛЬНО:**

1. **Низкая cardinality labels.**
2. **Нормализация path.**
3. **Агрегация.**
4. **Cardinality budget.**

**👍 СТОИТ:**

5. **Recording rules** для агрегатов.
6. **Лимиты** в scrape config.
7. **Алерт** на высокую cardinality.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй user_id, request_id как labels.**
9. **Не забывай про path с параметрами.**
10. **Не храни бизнес-данные в метриках.**

### Где мы сейчас

Мы разобрали cardinality. Теперь — **диагностика**.

---

## 22.14 Диагностика проблем

### 🔌 Проблема: метрики не собираются

Prometheus не опрашивает target. Или опрашивает, но метрик нет. Или алерты не приходят.

### 🔍 Типичные проблемы

**1. Target не опрашивается.**

**Диагностика:**

```bash
# Prometheus UI → Status → Targets
# Ищи target, проверь Status (UP/DOWN) и Error
```

**Причины:**

- Неправильный ServiceMonitor.
- Target недоступен.
- Неправильные labels.
- RBAC.

**Решение:**

```bash
# Проверить ServiceMonitor
kubectl get servicemonitor -A
kubectl describe servicemonitor myapp -n production

# Проверить labels Service
kubectl get svc myapp -n production --show-labels

# Проверить Prometheus config
kubectl exec -n monitoring prometheus-xxx -- cat /etc/prometheus/config_out/prometheus.env.yaml | grep -A 10 myapp
```

**2. Метрики не появляются.**

**Причины:**

- Приложение не экспортирует.
- Неправильный path.
- Неправильный порт.

**Диагностика:**

```bash
# Проверить вручную
kubectl exec -n production myapp-xxx -- curl localhost:8080/metrics

# Проверить Prometheus scrape
# Status → Targets → ошибки
```

**3. Метрики есть, но не те.**

**Причины:**

- Relabeling удаляет.
- Дубликаты.
- Aggregation.

**Диагностика:**

```promql
# Найти метрику
{__name__=~".*http.*"}

# Проверить labels
count by (__name__) ({__name__=~".*http.*"})
```

**4. Алерты не приходят.**

**Причины:**

- Prometheus не отправляет в Alertmanager.
- Alertmanager не маршрутизирует.
- Notification не настроен.

**Диагностика:**

```bash
# Prometheus UI → Alerts
# Проверить, что алерт firing

# Alertmanager UI
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093
# Alerts → проверить

# Проверить конфигурацию
kubectl get secret -n monitoring alertmanager-prometheus-kube-prometheus-alertmanager -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d
```

**5. Высокая cardinality.**

**Диагностика:**

```
Prometheus UI → Status → TSDB Status
```

**Решение:**

- Убрать плохие labels.
- Нормализовать path.
- Агрегировать.

**6. Prometheus падает с OOM.**

**Причины:**

- Много series.
- Большие запросы.
- Недостаточно RAM.

**Диагностика:**

```bash
kubectl top pod -n monitoring prometheus-xxx
kubectl logs -n monitoring prometheus-xxx | grep -i oom
```

**Решение:**

- Увеличить RAM.
- Снизить cardinality.
- Оптимизировать запросы.
- Remote write.

**7. Медленные запросы.**

**Причины:**

- Слишком широкий диапазон.
- Сложные запросы.
- Много series.

**Решение:**

- Recording rules.
- Ограничить диапазон.
- Оптимизировать.

### 🎯 Prometheus tools

**`promtool`:**

```bash
# Проверить конфигурацию
promtool check config prometheus.yml

# Проверить rules
promtool check rules rules.yml

# Тестировать rules
promtool test rules tests.yml

# Query
promtool query instant http://localhost:9090 'up'
```

### 🎯 Logs

```bash
# Prometheus
kubectl logs -n monitoring prometheus-xxx

# Alertmanager
kubectl logs -n monitoring alertmanager-xxx

# Operator
kubectl logs -n monitoring prometheus-operator-xxx
```

### 🔬 Практика: диагностика

```bash
# 1. Targets
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
# http://localhost:9090/targets

# 2. Rules
# http://localhost:9090/rules

# 3. Alerts
# http://localhost:9090/alerts

# 4. TSDB Status
# http://localhost:9090/tsdb-status

# 5. Alertmanager
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093
# http://localhost:9093

# 6. Проверить config
kubectl exec -n monitoring prometheus-xxx -- cat /etc/prometheus/config_out/prometheus.env.yaml

# 7. Логи
kubectl logs -n monitoring prometheus-xxx --tail=100
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Status → Targets** — что опрашивается.
2. **Status → TSDB** — cardinality.
3. **Alerts** — что firing.
4. **Логи** — ошибки.

**👍 СТОИТ:**

5. **`promtool`** для конфигурации.
6. **Alertmanager UI** для алертов.
7. **Grafana** для дашбордов.

**❌ НЕ ДЕЛАЙ:**

8. **Не игнорируй DOWN targets.**
9. **Не забывай про cardinality.**
10. **Не используй слишком сложные запросы.**

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **Metric** | Числовое значение во времени. |
| **Counter** | Только растёт. |
| **Gauge** | Растёт и падает. |
| **Histogram** | Распределение по бакетам. |
| **Summary** | Quantiles на клиенте. |
| **Label** | Key-value для различения. |
| **Cardinality** | Количество уникальных комбинаций labels. |
| **Prometheus** | Система мониторинга. |
| **Pull-based** | Prometheus сам опрашивает. |
| **Push-based** | Приложение отправляет. |
| **Exporter** | Программа для экспорта метрик. |
| **ServiceMonitor** | CRD Prometheus Operator. |
| **PodMonitor** | CRD для Pod'ов. |
| **PromQL** | Язык запросов Prometheus. |
| **Recording rules** | Предвычисление запросов. |
| **Alerting rules** | Правила алертов. |
| **Alertmanager** | Управление алертами. |
| **Inhibition** | Подавление алертов. |
| **Silence** | Временное отключение. |
| **SLO** | Service Level Objective. |
| **SLI** | Service Level Indicator. |
| **SLA** | Service Level Agreement. |
| **Error budget** | Допустимое количество ошибок. |
| **Burn rate** | Скорость исчерпания budget. |
| **RED method** | Rate, Errors, Duration. |
| **USE method** | Utilization, Saturation, Errors. |
| **Four Golden Signals** | Latency, Traffic, Errors, Saturation. |
| **Thanos** | Долгосрочное хранение Prometheus. |
| **VictoriaMetrics** | Альтернатива Prometheus. |
| **Mimir** | Масштабируемый Prometheus. |

---

## Что мы узнали?

- **Метрики:** counter, gauge, histogram, summary.
- **Prometheus:** pull-based, TSDB, PromQL, Alertmanager.
- **Exporters:** node_exporter, kube-state-metrics, blackbox.
- **Instrumentation:** `prometheus/client_golang`, counter/histogram/gauge.
- **Service discovery:** ServiceMonitor, PodMonitor, Probe.
- **PromQL:** `rate()`, `histogram_quantile()`, агрегации.
- **Recording rules:** предвычисления.
- **Grafana:** дашборды, переменные, provisioning.
- **Alertmanager:** маршрутизация, группировка, inhibition.
- **SLO/SLI/SLA:** цели, error budget.
- **Хорошие алерты:** actionable, symptom-based, SLO-based.
- **Масштабирование:** Thanos, VictoriaMetrics, Mimir.
- **Cardinality:** главный враг, низкие labels.
- **Диагностика:** targets, TSDB, alerts, logs.

---

## Типичные ошибки

- ❌ **Использовать `user_id` как label.** Cardinality.
- ❌ **Не нормализовать path.** `/api/orders/123` → миллионы.
- ❌ **Алертить на cause, не symptom.** CPU > 80%.
- ❌ **Не использовать `for` в алертах.** Слишком много false positives.
- ❌ **Не группировать алерты.** 200 сообщений.
- ❌ **Не использовать SLO.** Нет цели — нет понимания.
- ❌ **Слишком много алертов.** Alert fatigue.
- ❌ **Не писать runbook.** Что делать при алерте.
- ❌ **Не использовать recording rules** для сложных запросов.
- ❌ **Не настраивать retention.** Диск кончится.
- ❌ **Не мониторить Prometheus.** Prometheus — тоже система.
- ❌ **Не использовать HA.** SPOF.
- ❌ **Игнорировать cardinality.** Prometheus упадёт.
- ❌ **Не тестировать алерты.** Могут не работать.

---

## Для быстрого повторения

- **Метрики:** counter, gauge, histogram.
- **Naming:** `_total`, `_seconds`, `_bytes`.
- **Prometheus:** pull-based, TSDB, PromQL.
- **Exporters:** node_exporter, kube-state-metrics, blackbox.
- **Instrumentation:** `promauto.NewCounterVec`, `NewHistogramVec`, `NewGauge`.
- **Service discovery:** ServiceMonitor, PodMonitor, Probe.
- **PromQL:** `rate()`, `sum by ()`, `histogram_quantile()`.
- **Recording rules:** `record: name`, `expr: ...`.
- **Grafana:** dashboards, variables, provisioning.
- **Alertmanager:** routing, grouping, inhibition, silence.
- **SLO:** 99.9%, error budget, burn rate.
- **Хорошие алерты:** symptom-based, SLO-based, actionable.
- **Масштабирование:** Thanos, VictoriaMetrics, Mimir.
- **Cardinality:** низкие labels, нормализация path.

---

## Вопросы для самопроверки

1. Четыре типа метрик — назови и объясни.
2. Что такое cardinality? Почему важно?
3. Что такое Prometheus? Pull-based vs push-based.
4. Что такое exporter? Приведи примеры.
5. Как инструментировать Go-приложение для Prometheus?
6. Что такое ServiceMonitor? Зачем нужен?
7. Что такое PromQL? Приведи пример запроса error rate.
8. Что такое recording rules? Зачем нужны?
9. Что такое Alertmanager? Что делает?
10. Что такое SLO, SLI, SLA? Чем отличаются?
11. Что такое error budget? Как использовать?
12. Что такое burn rate? Зачем нужен?
13. Что такое inhibition и silence в Alertmanager?
14. Как масштабировать Prometheus?
15. Что такое cardinality? Как её снизить?

---

## Ответы

**1. Четыре типа метрик**

Counter (только растёт), Gauge (растёт и падает), Histogram (распределение по бакетам), Summary (quantiles на клиенте). Используй histogram, не summary.

**2. Cardinality**

Количество уникальных комбинаций labels. Высокая cardinality → много series → RAM кончится. Не использовать user_id, request_id как labels.

**3. Prometheus**

Система мониторинга. Pull-based: сам опрашивает targets. Push-based: приложение отправляет. Pull проще для отладки, push для short-lived jobs.

**4. Exporter**

Программа для экспорта метрик. node_exporter (ноды), kube-state-metrics (K8s), postgres_exporter (PostgreSQL), blackbox_exporter (endpoints).

**5. Instrumentation Go**

```go
promauto.NewCounterVec(prometheus.CounterOpts{Name: "http_requests_total"}, []string{"method", "status"})
promauto.NewHistogramVec(...)
promauto.NewGauge(...)
```

**6. ServiceMonitor**

CRD Prometheus Operator. Декларативно описывает, как опрашивать сервис. Operator добавляет config в Prometheus.

**7. PromQL**

```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
```

**8. Recording rules**

Предвычисление запросов. Сохраняются как новые метрики. Быстрее для дашбордов. Naming: `level:metric:operations`.

**9. Alertmanager**

Управляет алертами: дедупликация, группировка, маршрутизация, подавление, отправка. От Prometheus получает alerts, отправляет в Slack/PagerDuty.

**10. SLO, SLI, SLA**

SLI — метрика (error rate). SLO — цель (99.9%). SLA — контракт (штраф). SLO строже SLA.

**11. Error budget**

Допустимое количество ошибок. `(1 - SLO) × total`. Budget не исчерпан → можно деплоить. Исчерпан → freeze.

**12. Burn rate**

Скорость исчерпания error budget. 14.4x → budget исчерпается за 2 дня вместо 30. Multi-window: fast + slow.

**13. Inhibition и silence**

Inhibition — подавление связанных алертов (critical подавляет warning). Silence — временное отключение (maintenance).

**14. Масштабирование Prometheus**

Thanos (долгосрочное хранение, глобальные запросы), VictoriaMetrics (проще, быстрее), Mimir (очень большие объёмы). Remote write.

**15. Cardinality**

Количество уникальных комбинаций labels. Снизить: убрать плохие labels, нормализовать path, агрегировать, recording rules.

---

## Куда идти дальше?

Мы разобрали мониторинг и алертинг. Теперь ты знаешь:

- Метрики: counter, gauge, histogram.
- Prometheus и pull-based.
- Exporters и instrumentation.
- Service discovery.
- PromQL.
- Recording rules.
- Grafana.
- Alertmanager.
- SLO/SLI/SLA и error budget.
- Хорошие алерты.
- Масштабирование.
- Cardinality.

Но мы пока не разобрали:

- **Трейсинг** (Глава 23) — OpenTelemetry, Jaeger, Tempo.
- **Отказоустойчивость и автопилот** (Глава 24).
- **Платформенная инженерия** (Глава 25).
- **Облака и multi-cloud** (Глава 26).

**Глава 23: Observability — трейсинг.** Погнали. 🚀