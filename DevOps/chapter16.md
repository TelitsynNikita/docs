# 🕸️ Глава 16: Service Mesh — Istio и Linkerd

**Что вы узнаете:**
- Что такое Service Mesh и зачем он нужен.
- Проблемы, которые решает Service Mesh: mTLS, traffic management, observability.
- Что такое sidecar proxy и как он работает.
- Чем Istio отличается от Linkerd и Cilium.
- Что такое Envoy и почему он стал стандартом data plane.
- Как работает mTLS между сервисами.
- Что такое traffic splitting, retries, circuit breaking.
- Что такое fault injection и как тестировать отказоустойчивость.
- Когда Service Mesh оправдан, а когда — избыточен.
- Как мигрировать на Service Mesh без простоя.

**После прочтения вы сможете:**
- Объяснить, зачем нужен Service Mesh.
- Выбрать между Istio, Linkerd и Cilium.
- Настроить mTLS между сервисами.
- Настроить traffic splitting для canary.
- Настроить retries и circuit breaking.
- Использовать fault injection для тестирования.
- Оценить стоимость и сложность Service Mesh.
- Мигрировать на Service Mesh постепенно.

---

## Содержание

- [16.0 Пролог: 50 микросервисов и 2500 соединений](#160-пролог-50-микросервисов-и-2500-соединений)
- [16.1 Что такое Service Mesh](#161-что-такое-service-mesh)
- [16.2 Проблемы, которые решает Service Mesh](#162-проблемы-которые-решает-service-mesh)
- [16.3 Sidecar proxy: как это работает](#163-sidecar-proxy-как-это-работает)
- [16.4 Envoy: стандарт data plane](#164-envoy-стандарт-data-plane)
- [16.5 Istio: архитектура и компоненты](#165-istio-архитектура-и-компоненты)
- [16.6 Linkerd: простота и производительность](#166-linkerd-простота-и-производительность)
- [16.7 Cilium Service Mesh: eBPF-подход](#167-cilium-service-mesh-ebpf-подход)
- [16.8 mTLS: шифрование между сервисами](#168-mtls-шифрование-между-сервисами)
- [16.9 Traffic management: routing, splitting, mirroring](#169-traffic-management-routing-splitting-mirroring)
- [16.10 Отказоустойчивость: retries, timeouts, circuit breaking](#1610-отказоустойчивость-retries-timeouts-circuit-breaking)
- [16.11 Fault injection: тестирование отказов](#1611-fault-injection-тестирование-отказов)
- [16.12 Observability в Service Mesh](#1612-observability-в-service-mesh)
- [16.13 Когда Service Mesh оправдан](#1613-когда-service-mesh-оправдан)
- [16.14 Миграция на Service Mesh](#1614-миграция-на-service-mesh)
- [16.15 Диагностика проблем](#1615-диагностика-проблем)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 16.0 Пролог: 50 микросервисов и 2500 соединений

Ты — DevOps-инженер. В кластере 50 микросервисов. Каждый общается с десятками других.

**Проблемы:**

**1. mTLS.**

Требование безопасности: весь трафик между сервисами должен быть зашифрован. Как?

**2. Retries.**

Сервис A вызывает B. B временно недоступен. Нужно retry. Где это делать?

**3. Timeouts.**

Запрос завис. Нужен timeout. В каждом сервисе?

**4. Circuit breaking.**

B постоянно падает. Нужно перестать к нему обращаться. Как?

**5. Traffic splitting.**

Нужно 10% трафика на новую версию. В каждом сервисе?

**6. Observability.**

Метрики между сервисами. Latency, error rate. В каждом сервисе?

**7. Retry storms.**

A retry'ит B. B retry'ит C. Лавина retry'ев. Как остановить?

**8. Service discovery.**

Как A находит B?

**Каждый сервис должен реализовать всё это сам.**

На Go — библиотека. На Python — другая. На Java — третья. **Дублирование. Несогласованность. Ошибки.**

**Решение:** вынести всё это в **инфраструктуру**. Не в код сервисов.

**Service Mesh.**

**Идея:** между сервисами — **прокси**. Весь трафик идёт через прокси. Прокси делает mTLS, retries, timeouts, circuit breaking, observability.

**Сервис не знает** о прокси. Просто вызывает B по HTTP. Прокси перехватывает и делает всё.

**В этой главе** мы разберём Service Mesh. Начнём с основ.

---

## 16.1 Что такое Service Mesh

### 🔌 Проблема: cross-cutting concerns в микросервисах

Каждый сервис должен реализовать:

- **Service discovery.**
- **Load balancing.**
- **Retries.**
- **Timeouts.**
- **Circuit breaking.**
- **mTLS.**
- **Observability** (metrics, traces).
- **Traffic management** (canary, blue-green).
- **Rate limiting.**
- **Access control.**

**Это называется cross-cutting concerns** — общие для всех сервисов.

**Проблемы:**

- **Дублирование.** Каждый сервис.
- **Несогласованность.** Разные библиотеки.
- **Ошибки.** Легко забыть.
- **Обновления.** Обновить 50 сервисов.
- **Разные языки.** Go, Python, Java, Node.js.

**Решение:** вынести в инфраструктуру.

### 📊 Что такое Service Mesh

**Service Mesh** — инфраструктурный слой для управления service-to-service коммуникацией.

**Ключевая идея:** **sidecar proxy** рядом с каждым сервисом. Весь трафик идёт через прокси.

```
┌─────────────────────────────────────────┐
│  Pod A                                   │
│                                          │
│  ┌─────────────┐    ┌─────────────┐    │
│  │  Service A  │    │  Proxy      │    │
│  │             │◄──►│  (sidecar)  │    │
│  └─────────────┘    └──────┬──────┘    │
└────────────────────────────┼────────────┘
                             │
                             │ mTLS
                             │
┌────────────────────────────┼────────────┐
│  Pod B                     │             │
│                            ▼             │
│  ┌─────────────┐    ┌─────────────┐    │
│  │  Service B  │◄──►│  Proxy      │    │
│  │             │    │  (sidecar)  │    │
│  └─────────────┘    └─────────────┘    │
└─────────────────────────────────────────┘
```

**Что происходит:**

1. Service A вызывает Service B.
2. Запрос идёт в **proxy A** (sidecar).
3. Proxy A делает mTLS, retries, timeout, добавляет headers.
4. Запрос идёт в **proxy B** (sidecar).
5. Proxy B делает mTLS, проверяет ACL, добавляет metrics.
6. Запрос идёт в Service B.

**Service A и B не знают** о прокси. Просто вызывают друг друга.

### 🎯 Control Plane и Data Plane

**Data Plane** — прокси. Обрабатывают трафик.

**Control Plane** — управляет прокси. Конфигурация, сертификаты.

```
┌─────────────────────────────────────────┐
│           CONTROL PLANE                  │
│                                          │
│  - Config management                    │
│  - Certificate authority (mTLS)        │
│  - Service discovery                    │
│  - Policy enforcement                   │
└──────────────────┬──────────────────────┘
                   │
                   │ xDS API
                   │
┌──────────────────▼──────────────────────┐
│           DATA PLANE                     │
│                                          │
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  Proxy A │  │  Proxy B │  │ Proxy C│ │
│  │ (sidecar)│  │ (sidecar)│  │        │ │
│  └──────────┘  └──────────┘  └────────┘ │
└─────────────────────────────────────────┘
```

**Control Plane** (Istiod, Linkerd control plane):

- **Config** — что делать прокси.
- **CA** — сертификаты для mTLS.
- **Service discovery** — где сервисы.

**Data Plane** (Envoy, Linkerd2-proxy):

- **Обрабатывает трафик.**
- **Применяет политики.**
- **Собирает метрики.**

### 🎯 Что даёт Service Mesh

**1. mTLS.**

Автоматическое шифрование между сервисами.

**2. Traffic management.**

Routing, splitting, mirroring.

**3. Отказоустойчивость.**

Retries, timeouts, circuit breaking.

**4. Observability.**

Metrics, traces, logs для каждого запроса.

**5. Security.**

Access control, rate limiting.

**6. Zero-trust.**

Каждый запрос аутентифицирован.

### 💡 Практика: что важно понять

**✅ ОБЯЗАТЕЛЬНО:**

1. **Service Mesh — инфраструктурный слой.**
2. **Sidecar proxy** рядом с каждым сервисом.
3. **Control plane + data plane.**

**👍 СТОИТ:**

4. **Начинать с малого.** Не всё сразу.
5. **mTLS как первый шаг.**
6. **Observability как второй.**

**❌ НЕ ДЕЛАЙ:**

7. **Не используй Service Mesh для 5 сервисов.** Избыточно.
8. **Не игнорируй сложность.** Service Mesh — это +1 система.
9. **Не внедряй всё сразу.**

### Где мы сейчас

Мы разобрали, что такое Service Mesh. Теперь — **проблемы, которые он решает**.

---

## 16.2 Проблемы, которые решает Service Mesh

### 🔌 Проблема: 8 cross-cutting concerns

Разберём каждую проблему подробно.

### 📊 1. mTLS

**Проблема:** трафик между сервисами — plain text. Любой, кто получит доступ к сети, может прочитать.

**Решение в Service Mesh:** автоматическое шифрование.

```
Service A ──mTLS──► Service B
```

**Что даёт:**

- **Шифрование.** Никто не прочитает.
- **Аутентификация.** A и B доказывают identity.
- **Автоматическая ротация.** Сертификаты обновляются.

**Без Service Mesh:** каждая команда реализует mTLS сама. Разные библиотеки, разные подходы.

### 📊 2. Service Discovery

**Проблема:** A хочет вызвать B. Как найти B?

**Решение в Service Mesh:** прокси знает о сервисах.

**В Kubernetes:** уже есть Service DNS. Service Mesh использует его.

**Зачем в Service Mesh:** дополнительная логика — routing, health checks, load balancing.

### 📊 3. Load Balancing

**Проблема:** B имеет 10 Pod'ов. Как распределить трафик?

**Решение в K8s:** Service (kube-proxy) — random.

**Решение в Service Mesh:** более умное.

- **Round-robin.**
- **Least connections.**
- **Consistent hashing.**
- **Locality-aware** (ближайшие Pod'ы).

### 📊 4. Retries

**Проблема:** B временно недоступен. A должен retry.

**Решение в Service Mesh:**

```yaml
retries:
  attempts: 3
  perTryTimeout: 2s
  retryOn: 5xx,reset,connect-failure
```

**Что даёт:**

- **Автоматический retry.**
- **Настраиваемые условия.**
- **Backoff.**

**Важно:** retries могут вызвать **retry storm**. Ограничивать.

### 📊 5. Timeouts

**Проблема:** запрос завис. Нужен timeout.

**Решение в Service Mesh:**

```yaml
timeout: 5s
```

**Что даёт:**

- **Автоматический timeout.**
- **Настраиваемый по endpoint.**

### 📊 6. Circuit Breaking

**Проблема:** B постоянно падает. A продолжает обращаться. Нагрузка.

**Решение в Service Mesh:**

```yaml
outlierDetection:
  consecutive5xxErrors: 5
  interval: 30s
  baseEjectionTime: 30s
```

**Что даёт:**

- **Автоматическое исключение** плохих Pod'ов.
- **Восстановление** после проверки.

**Как работает:**

1. Pod B1 возвращает 5xx.
2. Через 5 ошибок — исключён.
3. Через 30 секунд — вернуть в пул.
4. Если снова 5xx — исключить снова.

### 📊 7. Traffic Management

**Проблема:** нужно 10% трафика на новую версию.

**Решение в Service Mesh:**

```yaml
route:
  - destination:
      host: myapp
      subset: v1
    weight: 90
  - destination:
      host: myapp
      subset: v2
    weight: 10
```

**Что даёт:**

- **Canary.**
- **Blue-Green.**
- **A/B testing.**
- **Traffic mirroring.**

### 📊 8. Observability

**Проблема:** метрики между сервисами.

**Решение в Service Mesh:** прокси автоматически собирает метрики.

**Что даёт:**

- **Latency** между сервисами.
- **Error rate.**
- **RPS.**
- **Traces** (автоматически).

**Без Service Mesh:** каждый сервис реализует.

### 📊 9. Access Control

**Проблема:** A не должен вызывать C.

**Решение в Service Mesh:**

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-a-to-b
spec:
  selector:
    matchLabels:
      app: b
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/default/sa/a"]
```

**Что даёт:**

- **Zero-trust.**
- **Authorization** на уровне прокси.

### 📊 10. Rate Limiting

**Проблема:** клиент отправляет 1000 RPS. Сервис не справляется.

**Решение в Service Mesh:** rate limiting на прокси.

### 🎯 Что Service Mesh НЕ решает

**1. Бизнес-логику.**

**2. Оптимизацию запросов** (N+1).

**3. Проблемы БД.**

**4. Application-level проблемы.**

**Service Mesh — инфраструктура.** Не замена application-логике.

### 💡 Практика: что важно понять

**✅ ОБЯЗАТЕЛЬНО:**

1. **8-10 cross-cutting concerns** решаются в одном месте.
2. **Автоматизация** — не код.
3. **Zero-trust** через mTLS и ACL.

**👍 СТОИТ:**

4. **Начинать с mTLS + observability.**
5. **Traffic management** для canary.
6. **Circuit breaking** для отказоустойчивости.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй Service Mesh для бизнес-логики.**
8. **Не забывай про retry storm.**
9. **Не игнорируй overhead.**

### Где мы сейчас

Мы разобрали проблемы. Теперь — **sidecar proxy**.

---

## 16.3 Sidecar proxy: как это работает

### 🔌 Проблема: как перехватить трафик

Как прокси перехватывает весь трафик сервиса, не изменяя код?

**Ответ:** iptables.

### 📊 Как работает sidecar

**1. Pod с двумя контейнерами:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  containers:
    - name: myapp              # основной контейнер
      image: myapp:1.0
      ports:
        - containerPort: 8080
    
    - name: istio-proxy        # sidecar
      image: istio/proxyv2:1.20.0
      ports:
        - containerPort: 15090  # metrics
        - containerPort: 15021  # health
```

**2. iptables rules:**

При создании Pod **init-контейнер** настраивает iptables:

```
# Исходящий трафик → redirect на Envoy
iptables -t nat -A OUTPUT -p tcp -j ISTIO_OUTPUT

# Входящий трафик → redirect на Envoy
iptables -t nat -A PREROUTING -p tcp -j ISTIO_INBOUND
```

**3. Что происходит:**

**Исходящий:**

```
Service A → :8080 → iptables → Envoy (15001) → B
                                   │
                                   │ mTLS, retries, timeout
                                   ▼
                                  B
```

**Входящий:**

```
A → :8080 → iptables → Envoy (15006) → Service B
                          │
                          │ mTLS check, ACL, metrics
                          ▼
                       Service B
```

**Service A и B не знают.** Просто вызывают :8080. iptables перенаправляет.

### 🎯 Ports Envoy

| Порт | Что |
|:---|:---|
| **15001** | Outbound (исходящий) |
| **15006** | Inbound (входящий) |
| **15000** | Admin |
| **15020** | Metrics (merged) |
| **15021** | Health check |
| **15090** | Prometheus metrics |
| **15010** | XDS (control plane) |

### 🎯 Init container

**`istio-init`** — настраивает iptables.

```yaml
initContainers:
  - name: istio-init
    image: istio/proxyv2:1.20.0
    args: ["istio-iptables", "-p", "15001", "-z", "15006", ...]
    securityContext:
      capabilities:
        add:
          - NET_ADMIN
```

**Что делает:**

- **Настраивает iptables** для перехвата.
- **Требует NET_ADMIN** capability.

### 🎯 Envoy sidecar

**`istio-proxy`** — Envoy.

**Что делает:**

- **Принимает трафик** от iptables.
- **Применяет конфигурацию** от control plane.
- **Делает mTLS, retries, timeouts.**
- **Собирает метрики.**
- **Экспортирует traces.**

### 🎯 Ресурсы

**Envoy потребляет:**

- **CPU:** ~10-50m per pod.
- **Memory:** ~50-100 MB per pod.

**Для 100 Pod'ов:**

- **CPU:** 1-5 CPU.
- **Memory:** 5-10 GB.

**Это overhead.** Учитывать.

### 🎯 Sidecar injection

**Автоматическая инъекция:**

```bash
# Включить для namespace
kubectl label namespace production istio-injection=enabled

# Все новые Pod'ы получат sidecar
```

**Ручная:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    sidecar.istio.io/inject: "true"
```

### 🎯 Проблемы sidecar

**1. Overhead.**

CPU, memory на каждый Pod.

**2. Latency.**

Дополнительный hop. +1-5ms.

**3. Startup.**

Envoy стартует дольше. Race condition с приложением.

**4. Debugging.**

Сложнее отлаживать. iptables, Envoy.

**5. Jobs.**

Sidecar не завершается. Job не может завершиться.

**Решение:** `sidecar.istio.io/inject: "false"` для Jobs.

### 🔬 Практика: sidecar

```bash
# 1. Установить Istio
istioctl install --set profile=demo -y

# 2. Включить инъекцию
kubectl label namespace default istio-injection=enabled

# 3. Развернуть приложение
kubectl apply -f myapp.yaml

# 4. Проверить Pod'ы
kubectl get pods
# NAME                    READY   STATUS
# myapp-xxx               2/2     Running
#                              ^^^
#                              2 контейнера: myapp + istio-proxy

# 5. Проверить контейнеры
kubectl describe pod myapp-xxx | grep -A 5 Containers
# myapp
# istio-proxy

# 6. Проверить iptables
kubectl exec myapp-xxx -c istio-proxy -- iptables -t nat -L -n
```

### 💡 Практика: как правильно работать со sidecar

**✅ ОБЯЗАТЕЛЬНО:**

1. **Автоматическая инъекция** через label.
2. **NET_ADMIN** для init-контейнера.
3. **Ресурсы** для Envoy.

**👍 СТОИТ:**

4. **Отключать инъекцию** для Jobs, CronJobs.
5. **Мониторить overhead.**
6. **Namespace-level injection** для простоты.

**❌ НЕ ДЕЛАЙ:**

7. **Не инжектируй везде.** Только нужные сервисы.
8. **Не забывай про Jobs.** Sidecar не завершается.
9. **Не игнорируй overhead.**

### Где мы сейчас

Мы разобрали sidecar. Теперь — **Envoy**.

---

## 16.4 Envoy: стандарт data plane

### 🔌 Проблема: какой прокси использовать

Istio, Linkerd, Cilium — разные data plane. Какой лучше?

**Envoy** — стандарт для Istio. Рассмотрим его.

### 📊 Что такое Envoy

**Envoy** — высокопроизводительный прокси от Lyft.

**Что даёт:**

- **L4/L7 proxy.**
- **HTTP/2, gRPC.**
- **Load balancing.**
- **Retries, timeouts, circuit breaking.**
- **mTLS.**
- **Observability.**
- **Dynamic configuration** через xDS.

**Написан на C++.** Высокая производительность.

### 🎯 Архитектура Envoy

```
┌─────────────────────────────────────┐
│  Envoy                              │
│                                     │
│  ┌──────────────┐  ┌─────────────┐ │
│  │  Listener    │  │  Cluster    │ │
│  │              │  │             │ │
│  │  Принимает   │  │  Куда       │ │
│  │  трафик      │  │  отправлять │ │
│  └──────────────┘  └─────────────┘ │
│                                     │
│  ┌──────────────┐  ┌─────────────┐ │
│  │  Filter      │  │  Route      │ │
│  │  chain       │  │             │ │
│  │  Обработка   │  │  Правила    │ │
│  └──────────────┘  └─────────────┘ │
└─────────────────────────────────────┘
```

**Компоненты:**

- **Listener** — принимает трафик на порту.
- **Filter chain** — цепочка фильтров (HTTP, TCP, custom).
- **Route** — правила маршрутизации.
- **Cluster** — группа upstream сервисов.

### 🎯 xDS API

**xDS** — протокол для dynamic configuration.

**Типы:**

- **LDS** (Listener Discovery Service) — listeners.
- **RDS** (Route Discovery Service) — routes.
- **CDS** (Cluster Discovery Service) — clusters.
- **EDS** (Endpoint Discovery Service) — endpoints.
- **SDS** (Secret Discovery Service) — TLS certs.

**Как работает:**

1. Envoy подключается к control plane (Istiod).
2. Получает конфигурацию через xDS.
3. Применяет.
4. Подписывается на обновления.

**Что даёт:** не нужно перезапускать Envoy при изменениях.

### 🎯 Фильтры

**Envoy имеет фильтры:**

- **HTTP connection manager** — L7.
- **TCP proxy** — L4.
- **Rate limit** — ограничение.
- **JWT auth** — аутентификация.
- **RBAC** — авторизация.
- **Fault injection** — тестирование.
- **CORS.**

**Кастомизация:** можно писать свои фильтры.

### 🎯 Load Balancing

**Алгоритмы:**

- **Round Robin.**
- **Least Request.**
- **Random.**
- **Ring Hash** (consistent hashing).
- **Maglev** (consistent hashing).

**Health checking:**

- **Active** — активные probe.
- **Passive** — outlier detection.

### 🎯 Outlier Detection

**Автоматическое исключение плохих endpoints:**

```yaml
outlierDetection:
  consecutive5xxErrors: 5
  interval: 30s
  baseEjectionTime: 30s
  maxEjectionPercent: 50
```

**Что даёт:**

- **Circuit breaking.**
- **Автоматическое восстановление.**

### 🔬 Практика: Envoy

```bash
# 1. В Pod'е с Istio
kubectl exec myapp-xxx -c istio-proxy -- curl localhost:15000/help

# 2. Посмотреть конфигурацию
kubectl exec myapp-xxx -c istio-proxy -- curl localhost:15000/config_dump

# 3. Посмотреть кластеры
kubectl exec myapp-xxx -c istio-proxy -- curl localhost:15000/clusters

# 4. Посмотреть listeners
kubectl exec myapp-xxx -c istio-proxy -- curl localhost:15000/listeners

# 5. Метрики
kubectl exec myapp-xxx -c istio-proxy -- curl localhost:15000/stats

# 6. Prometheus metrics
kubectl exec myapp-xxx -c istio-proxy -- curl localhost:15090/stats/prometheus | head -50
```

### 💡 Практика: как правильно работать с Envoy

**✅ ОБЯЗАТЕЛЬНО:**

1. **Знать admin endpoint** (15000) для debug.
2. **Понимать listeners, clusters, routes.**
3. **Использовать outlier detection.**

**👍 СТОИТ:**

4. **Custom filters** для специфичных задач.
5. **Мониторить Envoy metrics.**

**❌ НЕ ДЕЛАЙ:**

7. **Не редактируй Envoy config вручную.** Через control plane.
8. **Не забывай про xDS.**

### Где мы сейчас

Мы разобрали Envoy. Теперь — **Istio**.

---

## 16.5 Istio: архитектура и компоненты

### 🔌 Проблема: как управлять 1000 Envoy

1000 Pod'ов с Envoy. Как их конфигурировать?

**Istio** — control plane для Envoy.

### 📊 Что такое Istio

**Istio** — самый популярный Service Mesh.

**Что даёт:**

- **Control plane** для Envoy.
- **Traffic management.**
- **Security** (mTLS, ACL).
- **Observability.**
- **Policy enforcement.**

### 🎯 Архитектура Istio

```
┌─────────────────────────────────────────┐
│              ISTIOD                      │
│         (Control Plane)                  │
│                                          │
│  - Pilot (traffic management)           │
│  - Citadel (certificates)               │
│  - Galley (config)                      │
└──────────────────┬──────────────────────┘
                   │
                   │ xDS
                   │
┌──────────────────▼──────────────────────┐
│           DATA PLANE                     │
│                                          │
│  Envoy sidecar в каждом Pod             │
│                                          │
└─────────────────────────────────────────┘
```

**Раньше:** Pilot, Citadel, Galley — отдельные компоненты.

**Сейчас:** **Istiod** — объединены.

### 🎯 Установка Istio

```bash
# 1. Скачать istioctl
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.20.0
export PATH=$PWD/bin:$PATH

# 2. Установить
istioctl install --set profile=demo -y

# 3. Проверить
kubectl get pods -n istio-system
# istiod-xxx
# istio-ingressgateway-xxx
# istio-egressgateway-xxx

# 4. Включить инъекцию
kubectl label namespace default istio-injection=enabled
```

### 🎯 Профили установки

| Профиль | Что устанавливает |
|:---|:---|
| **minimal** | Istiod |
| **default** | Istiod + ingress gateway |
| **demo** | + egress gateway, Prometheus, Grafana, Jaeger, Kiali |
| **empty** | Ничего (для кастомизации) |

### 🎯 Основные CRD

**Traffic management:**

- **VirtualService** — routing rules.
- **DestinationRule** — policies for destinations.
- **Gateway** — ingress/egress.
- **ServiceEntry** — external services.
- **Sidecar** — sidecar configuration.

**Security:**

- **PeerAuthentication** — mTLS.
- **RequestAuthentication** — JWT.
- **AuthorizationPolicy** — ACL.

### 🎯 VirtualService

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  http:
    - match:
        - headers:
            user-agent:
              regex: ".*mobile.*"
      route:
        - destination:
            host: myapp
            subset: mobile
    - route:
        - destination:
            host: myapp
            subset: v1
          weight: 90
        - destination:
            host: myapp
            subset: v2
          weight: 10
      retries:
        attempts: 3
        perTryTimeout: 2s
      timeout: 5s
```

**Что даёт:**

- **Routing по headers.**
- **Weighted split** (90/10).
- **Retries, timeouts.**

### 🎯 DestinationRule

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

**Что даёт:**

- **Load balancing.**
- **Connection pool.**
- **Outlier detection.**
- **Subsets** (для routing).

### 🎯 Gateway

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: myapp-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: myapp-tls
      hosts:
        - myapp.example.com
```

**Что даёт:** ingress для внешнего трафика.

### 🔬 Практика: Istio

```bash
# 1. Установить Istio
istioctl install --set profile=demo -y

# 2. Включить инъекцию
kubectl label namespace default istio-injection=enabled

# 3. Развернуть приложение
kubectl apply -f myapp.yaml

# 4. Проверить
kubectl get pods
# 2/2 контейнера

# 5. Проверить sidecar
istioctl analyze -n default

# 6. Проверить proxy config
istioctl proxy-config cluster myapp-xxx
istioctl proxy-config listener myapp-xxx
istioctl proxy-config route myapp-xxx
istioctl proxy-config endpoint myapp-xxx

# 7. Kiali UI
istioctl dashboard kiali

# 8. Проверить mTLS
kubectl exec myapp-xxx -c istio-proxy -- curl localhost:15000/config_dump | grep -i mtls
```

### 💡 Практика: как правильно использовать Istio

**✅ ОБЯЗАТЕЛЬНО:**

1. **istioctl** для установки и debugging.
2. **Profiles** для разных окружений.
3. **VirtualService + DestinationRule** для routing.

**👍 СТОИТ:**

4. **Kiali** для визуализации.
5. **istioctl analyze** для проверки.
6. **PeerAuthentication** для mTLS.

**❌ НЕ ДЕЛАЙ:**

7. **Не устанавливай все компоненты сразу.** Начни с minimal.
8. **Не забывай про resource limits.**
9. **Не игнорируй istioctl analyze.**

### Где мы сейчас

Мы разобрали Istio. Теперь — **Linkerd**.

---

## 16.6 Linkerd: простота и производительность

### 🔌 Проблема: Istio сложный

Istio мощный, но сложный: 1000 настроек, 20 CRD. Для маленьких команд — overkill.

**Linkerd** — проще, легче, быстрее.

### 📊 Что такое Linkerd

**Linkerd** — Service Mesh от Buoyant (теперь CNCF).

**Отличия от Istio:**

| Аспект | Istio | Linkerd |
|:---|:---|:---|
| **Data plane** | Envoy (C++) | Linkerd2-proxy (Rust) |
| **Memory** | 50-100 MB/pod | 10-20 MB/pod |
| **CPU** | 10-50m | 5-20m |
| **Latency** | +2-5ms | +1-2ms |
| **Сложность** | Высокая | Низкая |
| **CRD** | Много | Мало |
| **Установка** | istioctl, Helm | CLI, Helm |
| **UI** | Kiali | Linkerd Viz |
| **mTLS** | Да (по умолчанию off) | Да (по умолчанию on) |
| **Traffic mgmt** | Powerful | Simple |

### 🎯 Linkerd2-proxy

**Написан на Rust.** Быстрее, меньше, безопаснее.

**Что даёт:**

- **Меньше памяти.**
- **Меньше latency.**
- **Безопасность** (Rust memory safety).

**Не Envoy.** Свой прокси. Меньше функций, но достаточно для большинства.

### 🎯 Установка Linkerd

```bash
# 1. Установить CLI
curl -sL https://run.linkerd.io/install | sh

# 2. Проверить prerequisites
linkerd check --pre

# 3. Установить control plane
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# 4. Проверить
linkerd check

# 5. Установить Viz (UI)
linkerd viz install | kubectl apply -f -

# 6. Проверить UI
linkerd viz dashboard
```

### 🎯 Инъекция sidecar

```bash
# Namespace
kubectl annotate namespace default linkerd.io/inject=enabled

# Pod
kubectl annotate pod myapp linkerd.io/inject=enabled
```

**Отличие от Istio:** аннотация, не label.

### 🎯 mTLS по умолчанию

**Linkerd включает mTLS по умолчанию.** Без настройки.

**Проверка:**

```bash
linkerd viz edges deployment -n default
# SRC           DST          SECURED
# myapp         api          ✓
# api           database     ✓
```

**Что даёт:** zero-trust из коробки.

### 🎯 Traffic Splitting

```yaml
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: myapp
spec:
  service: myapp
  backends:
    - service: myapp-v1
      weight: 900
    - service: myapp-v2
      weight: 100
```

**Что даёт:** 90/10 split.

**SMI (Service Mesh Interface)** — стандарт для traffic splitting.

### 🎯 Retries и Timeouts

**ServiceProfile:**

```yaml
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: myapp.default.svc.cluster.local
spec:
  routes:
    - name: GET /api/orders
      condition:
        method: GET
        pathRegex: /api/orders
      timeout: 5s
      retries:
        budget:
          minRetriesPerSec: 5
          percentCanRetry: 0.5
          ttl: 10s
```

**Что даёт:** retries и timeouts per route.

### 🎯 Observability

**Linkerd Viz:**

- **Golden metrics** (success rate, RPS, latency).
- **Service topology.**
- **Routes.**
- **Tap** (live traffic).

**Команды:**

```bash
# Golden metrics
linkerd viz stat deployment -n default
# NAME    MESHED   SUCCESS   RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99
# myapp   2/2      100.00%   10.5  2ms           5ms           8ms

# Edges (топология)
linkerd viz edges deployment -n default

# Tap (live traffic)
linkerd viz tap deployment/myapp -n default

# Routes
linkerd viz routes deployment/myapp -n default
```

### 🔬 Практика: Linkerd

```bash
# 1. Установить Linkerd
curl -sL https://run.linkerd.io/install | sh
export PATH=$HOME/.linkerd2/bin:$PATH

# 2. Проверить
linkerd check --pre

# 3. Установить
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -
linkerd check

# 4. Viz
linkerd viz install | kubectl apply -f -

# 5. Включить инъекцию
kubectl annotate namespace default linkerd.io/inject=enabled

# 6. Развернуть приложение
kubectl apply -f myapp.yaml
kubectl rollout restart deployment myapp

# 7. Проверить
linkerd viz stat deployment -n default
linkerd viz edges deployment -n default
linkerd viz tap deployment/myapp -n default

# 8. Dashboard
linkerd viz dashboard
```

### 💡 Практика: как правильно использовать Linkerd

**✅ ОБЯЗАТЕЛЬНО:**

1. **mTLS включён** по умолчанию.
2. **Linkerd Viz** для observability.
3. **ServiceProfile** для retries, timeouts.

**👍 СТОИТ:**

4. **TrafficSplit** для canary.
5. **Linkerd check** для диагностики.
6. **Annotation-based injection.**

**❌ НЕ ДЕЛАЙ:**

7. **Не используй Linkerd для сложного traffic management.** Istio лучше.
8. **Не забывай про аннотацию.**
9. **Не игнорируй Viz.**

### Где мы сейчас

Мы разобрали Linkerd. Теперь — **Cilium Service Mesh**.

---

## 16.7 Cilium Service Mesh: eBPF-подход

### 🔌 Проблема: sidecar добавляет latency

Sidecar proxy — дополнительный hop. +1-5ms latency. Overhead CPU/memory.

**Cilium** — Service Mesh без sidecar. Через eBPF в ядре.

### 📊 Что такое Cilium Service Mesh

**Cilium** — CNI-плагин с Service Mesh через eBPF.

**Ключевая идея:** обработка трафика в ядре Linux через eBPF. Без sidecar.

**Отличия:**

| Аспект | Istio/Linkerd | Cilium |
|:---|:---|:---|
| **Data plane** | Sidecar | eBPF в ядре |
| **Latency** | +1-5ms | +0.1-0.5ms |
| **CPU** | Overhead на pod | Минимальный |
| **Memory** | Overhead на pod | Минимальный |
| **Kernel** | 4.x+ | 5.x+ |
| **Complexity** | Высокая | Средняя |

### 🎯 Как работает

**eBPF-программы загружаются в ядро.**

**Что делают:**

- **L3/L4 processing** в ядре.
- **L7 parsing** (HTTP, gRPC, Kafka).
- **mTLS** через WireGuard или IPsec.
- **Network policy** на L7.
- **Observability** (Hubble).

**Без iptables, без sidecar.**

### 🎯 Установка

```bash
# 1. Установить Cilium CLI
curl -L --remote-name https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
tar xzvf cilium-linux-amd64.tar.gz
sudo mv cilium /usr/local/bin

# 2. Установить Cilium
cilium install

# 3. Проверить
cilium status

# 4. Включить Service Mesh
cilium upgrade --set ingressController.enabled=true \
  --set kubeProxyReplacement=true \
  --set l7Proxy=true

# 5. Hubble (observability)
cilium hubble enable --ui

# 6. Hubble UI
cilium hubble ui
```

### 🎯 mTLS

**Cilium mTLS** через **WireGuard** (прозрачно) или **SPIFFE** (для L7).

```bash
# WireGuard
cilium upgrade --set encryption.enabled=true \
  --set encryption.type=wireguard
```

**Что даёт:** прозрачное шифрование между нодами.

### 🎯 Network Policy

**Cilium поддерживает L7 policies:**

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-http-get
spec:
  endpointSelector:
    matchLabels:
      app: myapp
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: GET
                path: /api/.*
```

**Что даёт:** HTTP-level policies (метод, path).

### 🎯 Hubble

**Hubble** — observability для Cilium.

**Что даёт:**

- **Network flows** между сервисами.
- **L7 metrics** (HTTP, gRPC).
- **DNS queries.**
- **Service dependencies.**

**UI:** визуализация flows.

### 🎯 Ingress

**Cilium Ingress Controller** — замена Nginx Ingress.

**Что даёт:**

- **eBPF-based.**
- **Меньше latency.**
- **L7 visibility.**

### 🔬 Практика: Cilium

```bash
# 1. Установить Cilium
cilium install

# 2. Проверить
cilium status

# 3. Hubble
cilium hubble enable --ui
cilium hubble ui

# 4. Проверить flows
hubble observe --namespace default

# 5. Service Map
# Hubble UI → Service Map
# Видно все связи между сервисами

# 6. Network Policy
kubectl apply -f cilium-network-policy.yaml

# 7. Проверить policy
cilium policy get
```

### 📊 Сравнение всех трёх

| Аспект | Istio | Linkerd | Cilium |
|:---|:---|:---|:---|
| **Data plane** | Envoy | linkerd2-proxy | eBPF |
| **Latency** | +2-5ms | +1-2ms | +0.1-0.5ms |
| **CPU** | Высокий | Низкий | Минимальный |
| **Memory** | 50-100 MB | 10-20 MB | 0 (ядро) |
| **mTLS** | Опционально | По умолчанию | WireGuard/IPsec |
| **Traffic mgmt** | Powerful | Simple | Средний |
| **CRD** | Много | Мало | Средне |
| **Сложность** | Высокая | Низкая | Средняя |
| **Kernel** | 4.x+ | 4.x+ | 5.x+ |
| **Когда** | Сложные сценарии | Простота | Производительность |

### 💡 Практика: как выбирать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Istio** для сложного traffic management.
2. **Linkerd** для простоты.
3. **Cilium** для производительности.

**👍 СТОИТ:**

4. **Cilium** если уже используется как CNI.
5. **Linkerd** для маленьких команд.
6. **Istio** для больших.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй два Service Mesh.** Конфликт.
8. **Не выбирай без понимания.**
9. **Не забывай про kernel version** (Cilium).

### Где мы сейчас

Мы разобрали три Service Mesh. Теперь — **mTLS**.

---

## 16.8 mTLS: шифрование между сервисами

### 🔌 Проблема: трафик между сервисами — plain text

Service A → Service B. HTTP без шифрования. Любой в сети может прочитать.

**Решение:** mTLS.

### 📊 Что такое mTLS

**mTLS (mutual TLS)** — двусторонняя аутентификация.

**Отличия от TLS:**

- **TLS:** клиент проверяет сервер.
- **mTLS:** и клиент, и сервер проверяют друг друга.

**Как работает:**

1. **Service A** имеет сертификат.
2. **Service B** имеет сертификат.
3. **При соединении:**
   - A проверяет сертификат B.
   - B проверяет сертификат A.
4. **Соединение шифруется.**

### 🎯 SPIFFE

**SPIFFE** — стандарт identity для сервисов.

**SPIFFE ID:**

```
spiffe://cluster.local/ns/production/sa/api
       │       │         │           │
       │       │         │           └── service account
       │       │         └── namespace
       │       └── trust domain
       └── scheme
```

**Что даёт:**

- **Уникальная identity** для каждого сервиса.
- **Не зависит от сети.**
- **Стандарт.**

### 🎯 mTLS в Istio

**По умолчанию — permissive.** Работает и с mTLS, и без.

**Включить strict:**

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

**Режимы:**

| Режим | Что означает |
|:---|:---|
| **STRICT** | Только mTLS |
| **PERMISSIVE** | И mTLS, и plain |
| **DISABLE** | Только plain |

**Проверка:**

```bash
kubectl exec myapp-xxx -c istio-proxy -- curl localhost:15000/config_dump | grep -i mtls
```

### 🎯 mTLS в Linkerd

**Включён по умолчанию.** Без настройки.

**Проверка:**

```bash
linkerd viz edges deployment -n default
# SRC     DST     SECURED
# myapp   api     ✓
```

### 🎯 Certificates

**Control plane выдаёт сертификаты.**

**Как работает:**

1. Pod с sidecar стартует.
2. Sidecar запрашивает сертификат у control plane.
3. Control plane проверяет ServiceAccount Pod'а.
4. Выдаёт сертификат с SPIFFE ID.
5. Сертификат живёт 24 часа (по умолчанию).
6. Автоматически ротируется.

**Что даёт:**

- **Автоматическая ротация.**
- **Короткий TTL.**
- **Не нужно управлять.**

### 🎯 Identity

**Каждый сервис имеет identity:**

```
spiffe://cluster.local/ns/production/sa/api
```

**Что можно делать:**

- **Authorization** по identity.
- **Audit** кто вызывал кого.
- **Zero-trust.**

**Пример AuthorizationPolicy:**

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend
spec:
  selector:
    matchLabels:
      app: api
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/frontend"
```

**Что даёт:** только frontend может вызывать api.

### 🔬 Практика: mTLS

**Istio:**

```bash
# 1. Проверить mTLS
kubectl exec myapp-xxx -c istio-proxy -- \
  openssl s_client -connect api:8080 -showcerts

# 2. Включить STRICT
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
EOF

# 3. Проверить
istioctl authn tls-check myapp-xxx api.production.svc.cluster.local
# HOST:PORT                          STATUS     SERVER     CLIENT
# api.production.svc.cluster.local   OK         STRICT     ISTIO
```

**Linkerd:**

```bash
# 1. Edges
linkerd viz edges deployment -n default

# 2. Проверить identity
linkerd viz identity -n default

# 3. Tap
linkerd viz tap deployment/myapp -n default --to deployment/api
```

### 💡 Практика: как правильно работать с mTLS

**✅ ОБЯЗАТЕЛЬНО:**

1. **STRICT mode** для production.
2. **SPIFFE ID** для identity.
3. **AuthorizationPolicy** по identity.

**👍 СТОИТ:**

4. **Автоматическая ротация** (по умолчанию).
5. **Audit logs** кто вызывал кого.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй PERMISSIVE в production.** Только STRICT.
8. **Не забывай про external services.** ServiceEntry.
9. **Не игнорируй certificate expiry.**

### Где мы сейчас

Мы разобрали mTLS. Теперь — **traffic management**.

---

## 16.9 Traffic management: routing, splitting, mirroring

### 🔌 Проблема: как управлять трафиком

Нужно:

- **Canary** — 10% на новую версию.
- **Blue-Green** — переключение.
- **A/B testing** — по headers.
- **Mirroring** — копия трафика на новую версию.

**Решение:** Service Mesh.

### 📊 Routing

**VirtualService в Istio:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  http:
    # Routing по headers
    - match:
        - headers:
            user-agent:
              regex: ".*Mobile.*"
      route:
        - destination:
            host: myapp
            subset: mobile
    # Routing по path
    - match:
        - uri:
            prefix: /api/v2
      route:
        - destination:
            host: myapp
            subset: v2
    # Default
    - route:
        - destination:
            host: myapp
            subset: v1
```

**Что даёт:** routing по headers, path, method, query.

### 📊 Traffic Splitting

**Canary:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  http:
    - route:
        - destination:
            host: myapp
            subset: v1
          weight: 90
        - destination:
            host: myapp
            subset: v2
          weight: 10
```

**Что даёт:** 90% на v1, 10% на v2.

**Изменение веса:**

```bash
# 50/50
kubectl patch virtualservice myapp --type=merge -p '...'

# Или через Git (GitOps)
git commit -am "Increase v2 to 50%"
git push
```

### 📊 Blue-Green

**Переключение:**

```yaml
# 100% на v1
- route:
    - destination:
        host: myapp
        subset: v1
      weight: 100
    - destination:
        host: myapp
        subset: v2
      weight: 0
```

**Cutover:**

```bash
# Изменить на 100% v2
kubectl patch virtualservice myapp --type=merge -p '...'
```

**Мгновенный rollback:** обратно на v1.

### 📊 Traffic Mirroring

**Mirror** — копия трафика на другую версию.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  http:
    - route:
        - destination:
            host: myapp
            subset: v1
      mirror:
        host: myapp
        subset: v2
      mirrorPercentage:
        value: 100.0
```

**Что даёт:**

- **v1 получает трафик и отвечает.**
- **v2 получает копию, но не отвечает.**
- **Тестирование без влияния на пользователей.**

**Когда использовать:** для тестирования новой версии на реальном трафике.

### 📊 Fault Injection

**Fault injection** — намеренные ошибки для тестирования.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  http:
    - fault:
        delay:
          percentage:
            value: 10.0
          fixedDelay: 5s
      route:
        - destination:
            host: myapp
```

**Что даёт:** 10% запросов с задержкой 5 секунд.

**Разберём подробно в подглаве 16.11.**

### 🎯 Linkerd TrafficSplit

```yaml
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: myapp
spec:
  service: myapp
  backends:
    - service: myapp-v1
      weight: 900
    - service: myapp-v2
      weight: 100
```

**Что даёт:** 90/10 split.

### 🔬 Практика: traffic management

**Canary в Istio:**

```bash
# 1. Развернуть v1 и v2
kubectl apply -f myapp-v1.yaml
kubectl apply -f myapp-v2.yaml

# 2. DestinationRule с subsets
cat > destinationrule.yaml <<EOF
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
EOF
kubectl apply -f destinationrule.yaml

# 3. VirtualService с 90/10
cat > virtualservice.yaml <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  http:
    - route:
        - destination:
            host: myapp
            subset: v1
          weight: 90
        - destination:
            host: myapp
            subset: v2
          weight: 10
EOF
kubectl apply -f virtualservice.yaml

# 4. Проверить
kubectl exec myapp-xxx -c istio-proxy -- \
  curl localhost:15000/config_dump | grep -A 5 "weight"

# 5. Изменить на 50/50
kubectl patch virtualservice myapp --type=merge -p '...'
```

### 💡 Практика: как правильно управлять трафиком

**✅ ОБЯЗАТЕЛЬНО:**

1. **DestinationRule** для subsets.
2. **VirtualService** для routing.
3. **Weighted split** для canary.

**👍 СТОИТ:**

4. **Mirroring** для тестирования.
5. **GitOps** для управления.
6. **Автоматизация** через Argo Rollouts/Flagger.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй 100% сразу.** Начинай с 1-5%.
8. **Не забывай про monitoring.**
9. **Не игнорируй rollback.**

### Где мы сейчас

Мы разобрали traffic management. Теперь — **отказоустойчивость**.

---

## 16.10 Отказоустойчивость: retries, timeouts, circuit breaking

### 🔌 Проблема: сервисы падают

Service B временно недоступен. Service A падает. **Cascading failure.**

**Решение:** отказоустойчивость в Service Mesh.

### 📊 Retries

**Автоматический retry:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  http:
    - route:
        - destination:
            host: myapp
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,reset,connect-failure,retriable-4xx
```

**Что даёт:**

- **3 попытки** при ошибке.
- **Timeout 2s** на попытку.
- **Только на retriable** ошибки.

**Осторожно: retry storm.**

A retry'ит B. B retry'ит C. Нагрузка растёт экспоненциально.

**Решение:**

- **Ограничить attempts.**
- **Retry budget** (Linkerd).
- **Circuit breaking.**

### 📊 Timeouts

```yaml
http:
  - route:
      - destination:
          host: myapp
    timeout: 5s
```

**Что даёт:** запрос > 5s — отменяется.

**Важно:** timeout должен быть **больше** retry timeout.

### 📊 Circuit Breaking

**Outlier detection:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

**Что даёт:**

- **5 подряд 5xx** → Pod исключён.
- **30 секунд** — проверка.
- **50% максимум** — исключить.

**Connection pool:**

```yaml
trafficPolicy:
  connectionPool:
    tcp:
      maxConnections: 100
    http:
      http1MaxPendingRequests: 100
      http2MaxRequests: 1000
      maxRequestsPerConnection: 10
```

**Что даёт:**

- **Лимиты** на connections, requests.
- **Fail fast** при перегрузке.

### 📊 Retry Budget (Linkerd)

```yaml
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: myapp.default.svc.cluster.local
spec:
  routes:
    - name: GET /api/orders
      condition:
        method: GET
        pathRegex: /api/orders
      timeout: 5s
      retries:
        budget:
          minRetriesPerSec: 5
          percentCanRetry: 0.5
          ttl: 10s
```

**Что даёт:**

- **Минимум 5 retries/sec.**
- **Максимум 50% от запросов.**
- **TTL 10 секунд.**

**Защита от retry storm.**

### 📊 Bulkhead

**Bulkhead** — изоляция ресурсов.

**Пример:** A вызывает B и C. B медленный. A должен изолировать пул для B.

```yaml
trafficPolicy:
  connectionPool:
    http:
      http1MaxPendingRequests: 10  # для B
```

**Что даёт:** B не влияет на C.

### 🔬 Практика: отказоустойчивость

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  http:
    - route:
        - destination:
            host: myapp
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,reset,connect-failure
      timeout: 10s

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels:
        version: v1
```

### 💡 Практика: как правильно обеспечивать отказоустойчивость

**✅ ОБЯЗАТЕЛЬНО:**

1. **Retries с ограничением.**
2. **Timeout > retry timeout.**
3. **Circuit breaking.**

**👍 СТОИТ:**

4. **Retry budget** (Linkerd).
5. **Bulkhead** для изоляции.
6. **Мониторинг** retries.

**❌ НЕ ДЕЛАЙ:**

7. **Не делай retries без ограничений.** Retry storm.
8. **Не забывай про timeout.**
9. **Не игнорируй circuit breaking.**

### Где мы сейчас

Мы разобрали отказоустойчивость. Теперь — **fault injection**.

---

## 16.11 Fault injection: тестирование отказов

### 🔌 Проблема: как тестировать отказоустойчивость

Ты настроил retries, timeouts. Но **работают ли они**? Как проверить?

**Решение:** fault injection.

### 📊 Что такое fault injection

**Fault injection** — намеренное внесение ошибок для тестирования.

**Типы:**

- **Delay** — задержка.
- **Abort** — ошибка.

### 🎯 Delay

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  http:
    - fault:
        delay:
          percentage:
            value: 10.0
          fixedDelay: 5s
      route:
        - destination:
            host: myapp
```

**Что даёт:** 10% запросов с задержкой 5 секунд.

**Тестирует:**

- **Timeouts.**
- **Retries.**
- **UI behavior** при медленном ответе.

### 🎯 Abort

```yaml
http:
  - fault:
      abort:
        percentage:
          value: 10.0
        httpStatus: 500
    route:
      - destination:
          host: myapp
```

**Что даёт:** 10% запросов возвращают 500.

**Тестирует:**

- **Retries.**
- **Circuit breaking.**
- **Error handling.**

### 🎯 Условия

**Fault injection можно ограничить:**

```yaml
http:
  - match:
      - headers:
          x-test-fault:
            exact: "true"
    fault:
      abort:
        percentage:
          value: 100.0
        httpStatus: 500
    route:
      - destination:
          host: myapp
```

**Что даёт:** fault только для запросов с header `x-test-fault: true`.

**Безопасное тестирование.**

### 🎯 Chaos Engineering

**Fault injection — часть chaos engineering.**

**Что тестировать:**

- **Latency** между сервисами.
- **Отказы** сервисов.
- **Ошибки** БД.
- **Сетевые проблемы.**
- **Заполнение диска.**
- **OOM.**

**Инструменты:**

- **Istio fault injection** — network-level.
- **Chaos Mesh** — Kubernetes chaos engineering.
- **Litmus** — chaos engineering для K8s.
- **Gremlin** — enterprise.

### 🎯 Game Days

**Game Day** — запланированное тестирование отказов.

**Что делать:**

1. **Выбрать сценарий** (отказ БД).
2. **Уведомить команду.**
3. **Внести fault.**
4. **Наблюдать.**
5. **Записать результаты.**
6. **Улучшить систему.**

**Что даёт:**

- **Понимание** слабых мест.
- **Обучение** команды.
- **Улучшение** отказоустойчивости.

### 🔬 Практика: fault injection

**1. Delay:**

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-fault
spec:
  hosts:
    - myapp
  http:
    - fault:
        delay:
          percentage:
            value: 100.0
          fixedDelay: 3s
      route:
        - destination:
            host: myapp
EOF

# Проверить задержку
time curl http://myapp/api/orders
# real 3s
```

**2. Abort:**

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-fault
spec:
  hosts:
    - myapp
  http:
    - fault:
        abort:
          percentage:
            value: 100.0
          httpStatus: 500
      route:
        - destination:
            host: myapp
EOF

# Проверить ошибку
curl -v http://myapp/api/orders
# < HTTP/1.1 500 Internal Server Error
```

**3. Удалить fault:**

```bash
kubectl delete virtualservice myapp-fault
```

### 💡 Практика: как правильно тестировать отказы

**✅ ОБЯЗАТЕЛЬНО:**

1. **Fault injection** для тестирования.
2. **Ограничить условиями** (headers).
3. **Game Days** регулярно.

**👍 СТОИТ:**

4. **Chaos Mesh** для Kubernetes.
5. **Автоматизация** chaos.
6. **Документация** результатов.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй fault injection в production** без ограничений.
8. **Не забывай удалять faults.**
9. **Не игнорируй результаты.**

### Где мы сейчас

Мы разобрали fault injection. Теперь — **observability в Service Mesh**.

---

## 16.12 Observability в Service Mesh

### 🔌 Проблема: как видеть, что происходит между сервисами

Metrics, traces, logs между сервисами.

**Service Mesh даёт автоматически.**

### 📊 Метрики

**Envoy экспортирует:**

- **Requests** (count, status).
- **Latency** (histogram).
- **Bytes** (sent, received).
- **Connections.**

**Стандартные метрики:**

```
istio_requests_total{source_workload="myapp", destination_workload="api", response_code="200"}
istio_request_duration_milliseconds_bucket{...}
istio_request_bytes_sum{...}
```

**Что даёт:**

- **RPS** между сервисами.
- **Error rate.**
- **Latency p50/p95/p99.**

**Grafana dashboards** из коробки.

### 📊 Traces

**Envoy генерирует spans автоматически.**

**Что даёт:**

- **Trace** для каждого запроса.
- **Spans** между сервисами.
- **Propagation** через headers.

**Экспорт в Jaeger/Tempo:**

```yaml
meshConfig:
  defaultConfig:
    tracing:
      sampling: 10.0
      zipkin:
        address: jaeger-collector.tracing:9411
```

**Что даёт:** traces без изменения кода.

### 📊 Logs

**Envoy access logs:**

```yaml
meshConfig:
  accessLogFile: /dev/stdout
  accessLogEncoding: JSON
```

**Что даёт:** логи всех запросов.

**Формат:**

```json
{
  "start_time": "...",
  "method": "GET",
  "path": "/api/orders",
  "response_code": 200,
  "duration": 45,
  "upstream_host": "10.0.0.5:8080",
  "downstream_remote_address": "10.0.0.1:12345",
  "x_request_id": "abc123"
}
```

### 📊 Service Map

**Kiali** (Istio) или **Linkerd Viz** показывают **service map**.

**Что видно:**

- **Сервисы** (nodes).
- **Связи** (edges).
- **RPS, latency, errors** на каждой связи.
- **Health.**

**Что даёт:**

- **Общая картина** системы.
- **Быстрое обнаружение** проблем.
- **Понимание** dependencies.

### 📊 Golden Metrics

**RED method:**

- **Rate** — RPS.
- **Errors** — error rate.
- **Duration** — latency.

**Linkerd Viz:**

```bash
linkerd viz stat deployment -n default
# NAME    MESHED   SUCCESS   RPS    LATENCY_P50   LATENCY_P95   LATENCY_P99
# myapp   2/2      100.00%   10.5   2ms           5ms           8ms
```

**Что даёт:** быстрый обзор.

### 📊 Distributed Tracing

**Service Mesh генерирует traces автоматически.**

**Что даёт:**

- **End-to-end** traces.
- **Без изменения кода.**
- **Propagation** через headers.

**Связь с приложением:** application spans + mesh spans = полный trace.

### 🎯 Kiali

**Kiali** — UI для Istio.

**Что показывает:**

- **Service graph.**
- **Health.**
- **Traffic.**
- **Traces.**
- **Logs.**
- **Configuration.**

**Установка:**

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml
istioctl dashboard kiali
```

### 🎯 Linkerd Viz

**UI для Linkerd.**

**Что показывает:**

- **Golden metrics.**
- **Service topology.**
- **Tap** (live traffic).
- **Routes.**

**Установка:**

```bash
linkerd viz install | kubectl apply -f -
linkerd viz dashboard
```

### 🎯 Hubble

**Hubble** — observability для Cilium.

**Что показывает:**

- **Network flows.**
- **L7 metrics.**
- **DNS queries.**
- **Service dependencies.**

**Установка:**

```bash
cilium hubble enable --ui
cilium hubble ui
```

### 🔬 Практика: observability

**Istio:**

```bash
# 1. Установить addons
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/jaeger.yaml

# 2. Kiali
istioctl dashboard kiali

# 3. Grafana
istioctl dashboard grafana

# 4. Jaeger
istioctl dashboard jaeger

# 5. Метрики
kubectl exec myapp-xxx -c istio-proxy -- curl localhost:15090/stats/prometheus | grep istio_requests_total
```

**Linkerd:**

```bash
# 1. Viz
linkerd viz install | kubectl apply -f -

# 2. Stat
linkerd viz stat deployment -n default

# 3. Edges
linkerd viz edges deployment -n default

# 4. Tap
linkerd viz tap deployment/myapp -n default

# 5. Dashboard
linkerd viz dashboard
```

### 💡 Практика: как правильно использовать observability

**✅ ОБЯЗАТЕЛЬНО:**

1. **Kiali/Linkerd Viz** для UI.
2. **Grafana** для дашбордов.
3. **Jaeger/Tempo** для traces.
4. **Golden metrics** для обзора.

**👍 СТОИТ:**

4. **Service map** для понимания.
5. **Access logs** для деталей.
6. **Alerts** на основе метрик.

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй observability.**
8. **Не забывай про sampling.**
9. **Не используй 100% logs** в production.

### Где мы сейчас

Мы разобрали observability. Теперь — **когда Service Mesh оправдан**.

---

## 16.13 Когда Service Mesh оправдан

### 🔌 Проблема: Service Mesh сложный

Service Mesh даёт много, но стоит:

- **+1 система** для управления.
- **+sidecar** на каждый Pod.
- **+latency** 1-5ms.
- **+CPU/memory** overhead.
- **+кривая обучения.**

**Когда он оправдан?**

### 📊 Когда оправдан

**✅ Использовать:**

1. **50+ микросервисов.**
2. **Требования zero-trust.**
3. **Нужен mTLS** между всеми сервисами.
4. **Сложный traffic management** (canary, A/B).
5. **Нужны retries, timeouts, circuit breaking** везде.
6. **Multi-language** команды.
7. **Compliance** требования.

**❌ Не использовать:**

1. **Маленькие системы** (5-10 сервисов).
2. **Монолит.**
3. **Нет требований** к mTLS.
4. **Простой traffic.**
5. **Нет команды** для управления.
6. **Критична latency** (high-frequency trading).
7. **Ресурсы ограничены.**

### 📊 Альтернативы

**Если Service Mesh избыточен:**

1. **Библиотеки** в коде (Go gRPC interceptors, resilience4j).
2. **API Gateway** для traffic management.
3. **Nginx/HAProxy** для retries, timeouts.
4. **Kubernetes Network Policies** для ACL.
5. **Cert-manager** для mTLS.

**Service Mesh — не единственный способ.**

### 📊 Decision tree

```
Нужен mTLS между всеми сервисами?
├── Да → 50+ сервисов?
│         ├── Да → Service Mesh (Istio/Linkerd)
│         └── Нет → Библиотеки или cert-manager
└── Нет → Сложный traffic management?
          ├── Да → API Gateway или Service Mesh
          └── Нет → Обычный K8s
```

### 🎯 Критерии

**Использовать Service Mesh, если:**

- **3+ из следующих:**
  - mTLS обязательно.
  - Сложный traffic management.
  - Multi-language.
  - 50+ сервисов.
  - Zero-trust.
  - Compliance.

**Не использовать, если:**

- **Меньше 3 из следующих:**
  - Маленькая команда.
  - Простые сервисы.
  - Нет требований безопасности.
  - Не хватает ресурсов.

### 💡 Практика: как принимать решение

**✅ ОБЯЗАТЕЛЬНО:**

1. **Оценить требования.**
2. **Оценить сложность.**
3. **Пилот** перед полным внедрением.

**👍 СТОИТ:**

4. **Начать с mTLS + observability.**
5. **Постепенно** добавлять функции.
6. **Измерить** benefit.

**❌ НЕ ДЕЛАЙ:**

7. **Не внедряй для 5 сервисов.**
8. **Не внедряй без команды.**
9. **Не внедряй всё сразу.**

### Где мы сейчас

Мы разобрали критерии. Теперь — **миграция**.

---

## 16.14 Миграция на Service Mesh

### 🔌 Проблема: как перейти без простоя

Production работает. Как внедрить Service Mesh?

### 📊 Стратегия миграции

**Фаза 1: Подготовка (1-2 недели).**

1. **Изучить требования.**
2. **Выбрать Service Mesh.**
3. **Развернуть в dev.**
4. **Пилот** на 1-2 сервисах.

**Фаза 2: mTLS (2-4 недели).**

5. **Включить PERMISSIVE** для всего.
6. **Постепенно** включать STRICT.
7. **Проверять** каждый сервис.

**Фаза 3: Observability (2-4 недели).**

8. **Развернуть Kiali/Linkerd Viz.**
9. **Настроить Grafana dashboards.**
10. **Настроить traces.**

**Фаза 4: Traffic Management (4-8 недель).**

11. **Внедрить canary.**
12. **Внедрить retries, timeouts.**
13. **Внедрить circuit breaking.**

**Фаза 5: Full adoption.**

14. **Все сервисы** в mesh.
15. **Все функции** включены.

### 🎯 Постепенное внедрение

**1. Namespace-by-namespace:**

```bash
kubectl label namespace staging istio-injection=enabled
# Тестировать
kubectl label namespace production istio-injection=enabled
```

**2. Service-by-service:**

```yaml
# Только для конкретного Deployment
metadata:
  annotations:
    sidecar.istio.io/inject: "true"
```

**3. Workload-by-workload.**

### 🎯 Pilot

**Пилот:**

- **1-2 сервиса.**
- **Не критичные.**
- **Мониторинг.**
- **Метрики.**

**Что измерить:**

- **Latency** до/после.
- **Error rate.**
- **CPU/memory overhead.**
- **Стабильность.**

**Если плохо — откатить.**

### 🎯 Rollback

**Как откатить:**

1. **Удалить инъекцию** из namespace.
2. **Перезапустить Pod'ы.**
3. **Удалить Istio** (если нужно).

```bash
kubectl label namespace production istio-injection-
kubectl rollout restart deployment -n production
```

### 🎯 Проблемы миграции

**1. Jobs не завершаются.**

Sidecar не завершается. **Решение:** `sidecar.istio.io/inject: "false"`.

**2. Startup race.**

Envoy стартует, приложение — нет. **Решение:** `holdApplicationUntilProxyStarts: true`.

**3. Health checks.**

kubelet → Pod. mTLS? **Решение:** Istio обрабатывает.

**4. External services.**

ServiceEntry для внешних.

**5. Database.**

Не в mesh. ServiceEntry для БД.

**6. Monitoring.**

Prometheus scrape. **Решение:** annotations.

### 🔬 Практика: миграция

```bash
# 1. Развернуть Istio в dev
istioctl install --set profile=demo -y

# 2. Включить инъекцию в dev
kubectl label namespace dev istio-injection=enabled

# 3. Развернуть приложение
kubectl rollout restart deployment -n dev

# 4. Проверить
kubectl get pods -n dev
# 2/2 контейнера

# 5. Метрики до/после
# Latency, error rate, CPU

# 6. Включить PERMISSIVE mTLS
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: dev
spec:
  mtls:
    mode: PERMISSIVE
EOF

# 7. Через неделю — STRICT
kubectl patch peerauthentication default -n dev --type=merge -p '{"spec":{"mtls":{"mode":"STRICT"}}}'

# 8. В production — постепенно
kubectl label namespace production istio-injection=enabled
kubectl rollout restart deployment -n production
```

### 💡 Практика: как правильно мигрировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Пилот** на 1-2 сервисах.
2. **Namespace-by-namespace.**
3. **PERMISSIVE → STRICT.**
4. **Мониторинг** до/после.

**👍 СТОИТ:**

4. **Метрики** benefit.
5. **План отката.**
6. **Документация.**

**❌ НЕ ДЕЛАЙ:**

7. **Не внедряй всё сразу.**
8. **Не забывай про Jobs.**
9. **Не игнорируй overhead.**

### Где мы сейчас

Мы разобрали миграцию. Теперь — **диагностика**.

---

## 16.15 Диагностика проблем

### 🔌 Проблема: что-то не работает

Service Mesh внедрён. Но что-то не так.

### 🔍 Типичные проблемы

**1. Sidecar не инжектится.**

**Причины:**

- Namespace не помечен.
- Pod не перезапущен.
- Annotation отключена.

**Диагностика:**

```bash
# Проверить labels namespace
kubectl get namespace production --show-labels

# Проверить Pod
kubectl get pod myapp-xxx -o yaml | grep -i inject

# Проверить контейнеры
kubectl get pod myapp-xxx -o jsonpath='{.spec.containers[*].name}'
```

**2. Service недоступен.**

**Причины:**

- mTLS mismatch.
- AuthorizationPolicy блокирует.
- Sidecar не готов.

**Диагностика:**

```bash
# Проверить контейнеры
kubectl get pods

# Логи sidecar
kubectl logs myapp-xxx -c istio-proxy

# Проверить Envoy config
istioctl proxy-config cluster myapp-xxx
istioctl proxy-config endpoint myapp-xxx

# Проверить authorization
istioctl analyze
```

**3. Latency выросла.**

**Причины:**

- Sidecar overhead.
- Неправильный routing.
- Retry storm.

**Диагностика:**

```bash
# Метрики
kubectl exec myapp-xxx -c istio-proxy -- curl localhost:15000/stats | grep upstream_rq_time

# Kiali
istioctl dashboard kiali

# Traces
istioctl dashboard jaeger
```

**4. Circuit breaking не работает.**

**Причины:**

- Неправильный DestinationRule.
- Не подходит к endpoints.

**Диагностика:**

```bash
# Проверить DestinationRule
kubectl get destinationrule myapp -o yaml

# Проверить Envoy
kubectl exec myapp-xxx -c istio-proxy -- curl localhost:15000/clusters | grep outlier
```

**5. mTLS не работает.**

**Причины:**

- PERMISSIVE режим.
- Сертификаты не выданы.
- Sidecar не готов.

**Диагностика:**

```bash
# Проверить PeerAuthentication
kubectl get peerauthentication -A

# Проверить mTLS
istioctl authn tls-check myapp-xxx api.default.svc.cluster.local

# Проверить сертификаты
istioctl proxy-config secret myapp-xxx
```

**6. Sidecar crash.**

**Причины:**

- OOM.
- Config error.
- Несовместимость версий.

**Диагностика:**

```bash
# Логи
kubectl logs myapp-xxx -c istio-proxy --previous

# Describe
kubectl describe pod myapp-xxx
```

**7. Traffic split не работает.**

**Причины:**

- Subsets не совпадают с labels.
- VirtualService не применяется.

**Диагностика:**

```bash
# Проверить labels Pod'ов
kubectl get pods -l version=v2

# Проверить DestinationRule
kubectl get destinationrule myapp -o yaml

# Проверить routes
istioctl proxy-config route myapp-xxx
```

### 🎯 Инструменты диагностики

**istioctl:**

```bash
# Анализ
istioctl analyze -A

# Прокси-конфиг
istioctl proxy-config cluster myapp-xxx
istioctl proxy-config listener myapp-xxx
istioctl proxy-config route myapp-xxx
istioctl proxy-config endpoint myapp-xxx
istioctl proxy-config secret myapp-xxx

# mTLS check
istioctl authn tls-check myapp-xxx api.default.svc.cluster.local

# Dashboard
istioctl dashboard kiali
istioctl dashboard grafana
istioctl dashboard jaeger

# Proxy status
istioctl proxy-status
```

**Linkerd:**

```bash
# Check
linkerd check

# Stat
linkerd viz stat deployment -n default

# Edges
linkerd viz edges deployment -n default

# Tap
linkerd viz tap deployment/myapp -n default

# Routes
linkerd viz routes deployment/myapp -n default

# Authz
linkerd viz authz -n default
```

### 🎯 Debug Envoy

```bash
# Config dump
kubectl exec myapp-xxx -c istio-proxy -- curl localhost:15000/config_dump > config.json

# Clusters
kubectl exec myapp-xxx -c istio-proxy -- curl localhost:15000/clusters

# Listeners
kubectl exec myapp-xxx -c istio-proxy -- curl localhost:15000/listeners

# Stats
kubectl exec myapp-xxx -c istio-proxy -- curl localhost:15000/stats

# Logging
kubectl exec myapp-xxx -c istio-proxy -- curl -X POST localhost:15000/logging?level=debug
```

### 🎯 Логи

```bash
# Sidecar
kubectl logs myapp-xxx -c istio-proxy

# Access logs
kubectl logs myapp-xxx -c istio-proxy | grep "GET\|POST"

# Istiod
kubectl logs -n istio-system deployment/istiod

# Ingress gateway
kubectl logs -n istio-system deployment/istio-ingressgateway
```

### 🔬 Практика: диагностика

```bash
# 1. Общий анализ
istioctl analyze -A

# 2. Proxy status
istioctl proxy-status

# 3. Проверить pod
kubectl get pod myapp-xxx -o jsonpath='{.spec.containers[*].name}'
# myapp istio-proxy

# 4. Проверить Envoy config
istioctl proxy-config cluster myapp-xxx
istioctl proxy-config endpoint myapp-xxx

# 5. Проверить mTLS
istioctl authn tls-check myapp-xxx api.default.svc.cluster.local

# 6. Kiali
istioctl dashboard kiali

# 7. Логи
kubectl logs myapp-xxx -c istio-proxy --tail=100
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **istioctl analyze** — первый шаг.
2. **istioctl proxy-status** — состояние.
3. **Логи sidecar** — детали.
4. **Envoy admin** — глубоко.

**👍 СТОИТ:**

4. **Kiali** для визуализации.
5. **Jaeger** для traces.
6. **Grafana** для метрик.

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй warning в analyze.**
8. **Не забывай про access logs.**
9. **Не диагностируй без traces.**

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **Service Mesh** | Инфраструктурный слой для service-to-service коммуникации. |
| **Sidecar** | Proxy рядом с сервисом. |
| **Control Plane** | Управляет прокси. |
| **Data Plane** | Прокси, обрабатывают трафик. |
| **Envoy** | Прокси от Lyft. Data plane Istio. |
| **istiod** | Control plane Istio. |
| **VirtualService** | CRD Istio для routing. |
| **DestinationRule** | CRD для policies. |
| **Gateway** | Ingress/egress Istio. |
| **PeerAuthentication** | mTLS policy. |
| **AuthorizationPolicy** | ACL policy. |
| **SPIFFE** | Стандарт identity. |
| **mTLS** | Mutual TLS. |
| **Retry storm** | Лавина retries. |
| **Circuit breaking** | Исключение плохих endpoints. |
| **Outlier detection** | Автоматическое исключение. |
| **Fault injection** | Намеренные ошибки. |
| **Chaos engineering** | Тестирование отказов. |
| **Linkerd** | Service Mesh от Buoyant. |
| **Linkerd2-proxy** | Прокси Linkerd (Rust). |
| **Linkerd Viz** | UI Linkerd. |
| **TrafficSplit** | SMI CRD для canary. |
| **ServiceProfile** | CRD Linkerd для routes. |
| **Cilium** | CNI + Service Mesh. |
| **eBPF** | Extended BPF. |
| **Hubble** | Observability Cilium. |
| **Kiali** | UI Istio. |
| **Golden metrics** | Rate, Errors, Duration. |
| **xDS** | Протокол dynamic config. |

---

## Что мы узнали?

- **Service Mesh** — инфраструктурный слой для service-to-service.
- **Sidecar proxy** перехватывает трафик через iptables.
- **Control plane + data plane** — управление и обработка.
- **Envoy** — стандарт data plane. **istiod** — control plane Istio.
- **Istio** — мощный, сложный. **Linkerd** — простой, лёгкий. **Cilium** — eBPF, быстрый.
- **mTLS** — шифрование и аутентификация через SPIFFE ID.
- **Traffic management** — routing, splitting, mirroring.
- **Отказоустойчивость** — retries, timeouts, circuit breaking.
- **Fault injection** — тестирование отказов.
- **Observability** — metrics, traces, logs автоматически.
- **Когда оправдан:** 50+ сервисов, mTLS, сложный traffic.
- **Миграция:** постепенно, namespace-by-namespace.
- **Диагностика:** istioctl, Kiali, Envoy admin, logs.

---

## Типичные ошибки

- ❌ **Использовать Service Mesh для 5 сервисов.** Избыточно.
- ❌ **Не тестировать перед production.**
- ❌ **PERMISSIVE mTLS в production.** Только STRICT.
- ❌ **Retries без ограничений.** Retry storm.
- ❌ **Забывать про Jobs.** Sidecar не завершается.
- ❌ **Не мониторить overhead.**
- ❌ **Игнорировать istioctl analyze.**
- ❌ **Не использовать observability.**
- ❌ **Внедрять всё сразу.**
- ❌ **Не иметь плана отката.**
- ❌ **Забывать про external services.**
- ❌ **Не обновлять версии.**
- ❌ **Игнорировать resource limits.**
- ❌ **Не документировать.**

---

## Для быстрого повторения

- **Service Mesh:** sidecar + control plane.
- **Data plane:** Envoy (Istio), linkerd2-proxy (Linkerd), eBPF (Cilium).
- **Control plane:** istiod (Istio), linkerd-control-plane.
- **Istio CRD:** VirtualService, DestinationRule, Gateway, PeerAuthentication, AuthorizationPolicy.
- **Linkerd CRD:** TrafficSplit, ServiceProfile.
- **mTLS:** SPIFFE ID, автоматические сертификаты.
- **Traffic mgmt:** routing, weight, mirror.
- **Отказоустойчивость:** retries, timeouts, outlier detection.
- **Fault injection:** delay, abort.
- **Observability:** Kiali, Linkerd Viz, Hubble, Jaeger.
- **Когда:** 50+ сервисов, mTLS, сложный traffic.
- **Миграция:** namespace-by-namespace, PERMISSIVE → STRICT.
- **Диагностика:** istioctl, Envoy admin (15000), logs.

---

## Вопросы для самопроверки

1. Что такое Service Mesh? Какую проблему решает?
2. Что такое sidecar? Как перехватывает трафик?
3. Что такое control plane и data plane?
4. Чем Istio отличается от Linkerd и Cilium?
5. Что такое Envoy? Какие компоненты?
6. Что такое mTLS? Как работает в Service Mesh?
7. Что такое SPIFFE ID?
8. Как работает traffic splitting?
9. Что такое retries, timeouts, circuit breaking?
10. Что такое fault injection? Зачем нужен?
11. Как Service Mesh помогает observability?
12. Когда Service Mesh оправдан?
13. Как мигрировать на Service Mesh?
14. Как диагностировать проблемы?
15. Что такое retry storm? Как избежать?

---

## Ответы

**1. Service Mesh**

Инфраструктурный слой для service-to-service. Решает cross-cutting concerns: mTLS, retries, timeouts, circuit breaking, observability, traffic management.

**2. Sidecar**

Proxy рядом с сервисом. iptables перенаправляет трафик. Service не знает о proxy. Sidecar делает mTLS, retries, метрики.

**3. Control plane и data plane**

Control plane (istiod) — управляет proxy: config, сертификаты, service discovery. Data plane (Envoy) — обрабатывает трафик.

**4. Istio vs Linkerd vs Cilium**

Istio: Envoy, мощный, сложный. Linkerd: linkerd2-proxy (Rust), простой, лёгкий, mTLS по умолчанию. Cilium: eBPF, без sidecar, быстрый.

**5. Envoy**

Прокси от Lyft. Data plane Istio. Listener, filter chain, route, cluster. Dynamic config через xDS.

**6. mTLS**

Mutual TLS: и клиент, и сервер проверяют сертификаты. Автоматические сертификаты от control plane. SPIFFE ID для identity.

**7. SPIFFE ID**

Стандарт identity: `spiffe://cluster.local/ns/production/sa/api`. Уникальная для каждого сервиса. Используется для authorization.

**8. Traffic splitting**

Weighted routing: 90% v1, 10% v2. VirtualService с weight. Для canary.

**9. Retries, timeouts, circuit breaking**

Retries: 3 попытки при ошибке. Timeouts: > 5s — отмена. Circuit breaking: 5 подряд 5xx → Pod исключён (outlier detection).

**10. Fault injection**

Намеренные ошибки: delay (10% запросов +5s), abort (10% запросов 500). Тестирует timeouts, retries, UI.

**11. Observability**

Автоматические метрики (istio_requests_total), traces (через Envoy), logs (access logs). Kiali для UI, Grafana для дашбордов, Jaeger для traces.

**12. Когда оправдан**

50+ сервисов, mTLS обязательно, сложный traffic management, multi-language, zero-trust, compliance. Не для 5 сервисов.

**13. Миграция**

Постепенно: dev → staging → prod. Namespace-by-namespace. PERMISSIVE → STRICT. Пилот на 1-2 сервисах. Мониторинг benefit.

**14. Диагностика**

`istioctl analyze`, `istioctl proxy-status`, логи sidecar, Envoy admin (15000), Kiali. Проверить mTLS, authorization, routes.

**15. Retry storm**

A retry'ит B, B retry'ит C. Нагрузка растёт. Решение: ограничить attempts, retry budget, circuit breaking.

---

## Куда идти дальше?

Мы разобрали Service Mesh. Теперь ты знаешь:

- Что такое Service Mesh.
- Sidecar proxy.
- Istio, Linkerd, Cilium.
- mTLS.
- Traffic management.
- Отказоустойчивость.
- Fault injection.
- Observability.
- Когда оправдан.
- Миграция.
- Диагностика.

**Следующая — Глава 17: Безопасность Kubernetes.** Она уже написана, но требует переименования (была отправлена как Глава 21). Скажи «дальше» — и я отправлю её в правильной нумерации с обновлённым заголовком.