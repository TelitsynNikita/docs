# 🌐 Глава 10: Kubernetes — сетевое взаимодействие

**Что вы узнаете:**
- Как Pod'ы общаются между собой и с внешним миром.
- Что такое CNI и какие плагины существуют (Calico, Cilium, Flannel).
- Как работает Service изнутри: kube-proxy, iptables, IPVS.
- Чем отличаются ClusterIP, NodePort, LoadBalancer, ExternalName.
- Что такое Ingress и Ingress Controller.
- Как работают Network Policies и как изолировать трафик.
- Как работает DNS в кластере (CoreDNS).
- Как диагностировать сетевые проблемы в K8s.

**После прочтения вы сможете:**
- Объяснить, как пакет идёт от клиента до Pod'а.
- Настроить Service для доступа к приложению.
- Настроить Ingress для HTTP-трафика с TLS.
- Изолировать трафик через Network Policies.
- Диагностировать проблемы: Pod не доступен, DNS не резолвится, Ingress не работает.

---

## Содержание

- [10.0 Пролог: сервис недоступен](#100-пролог-сервис-недоступен)
- [10.1 Сетевая модель Kubernetes](#101-сетевая-модель-kubernetes)
- [10.2 CNI: как Pod'ы получают IP](#102-cni-как-podы-получают-ip)
- [10.3 Service: стабильный доступ к Pod'ам](#103-service-стабильный-доступ-к-podам)
- [10.4 kube-proxy: как работает Service изнутри](#104-kube-proxy-как-работает-service-изнутри)
- [10.5 ClusterIP, NodePort, LoadBalancer, ExternalName](#105-clusterip-nodeport-loadbalancer-externalname)
- [10.6 Ingress: HTTP-маршрутизация](#106-ingress-http-маршрутизация)
- [10.7 CoreDNS: DNS внутри кластера](#107-coredns-dns-внутри-кластера)
- [10.8 Network Policies: изоляция трафика](#108-network-policies-изоляция-трафика)
- [10.9 Диагностика сетевых проблем](#109-диагностика-сетевых-проблем)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 10.0 Пролог: сервис недоступен

Ты задеплоил приложение. Pod'ы работают:

```bash
kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# myapp-7d9f8c6b4d-abc12   1/1     Running   0          5m
# myapp-7d9f8c6b4d-def34   1/1     Running   0          5m
# myapp-7d9f8c6b4d-ghi56   1/1     Running   0          5m
```

Service тоже есть:

```bash
kubectl get service myapp
# NAME    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# myapp   ClusterIP   10.96.123.45    <none>        80/TCP    5m
```

Но при попытке обратиться из другого Pod'а:

```bash
kubectl exec -it debug -- curl http://myapp
# curl: (7) Failed to connect to myapp port 80: Connection refused
```

**Connection refused.** Pod'ы работают, Service существует, но трафик не идёт.

Что не так? Где застревает пакет?

Возможные причины:

- **Selector не совпадает.** Service ищет Pod'ы с `app=myapp`, а у Pod'ов `app=my-app`. Endpoints пусты.
- **targetPort неправильный.** Service отправляет на порт 80, а Pod слушает 8080.
- **Приложение не слушает 0.0.0.0.** Слушает только localhost.
- **Network Policy блокирует.** Трафик между namespace запрещён.
- **kube-proxy не работает на ноде.**

Сетевые проблемы — самые сложные для диагностики. Пакет проходит **через много уровней**: Service → kube-proxy → iptables → Pod → приложение. На каждом может застрять.

В этой главе мы разберём **сетевую модель Kubernetes** от Pod'ов до Ingress. Ты поймёшь, как трафик ходит внутри кластера, и научишься диагностировать проблемы.

Это — Третий путь DevOps (Continuous Learning): понимание внутренностей системы, чтобы быстро находить проблемы.

---

## 10.1 Сетевая модель Kubernetes

### 🔌 Проблема: как Pod'ы общаются

В Главе 8 мы узнали: каждый Pod имеет свой IP. Но как эти IP работают? Как Pod на node-1 может обратиться к Pod на node-2?

**Ответ:** сетевая модель Kubernetes.

### 📊 Требования сетевой модели

Kubernetes определяет **четыре требования** к сети:

**1. Каждый Pod имеет уникальный IP во всём кластере.**

Не может быть двух Pod'ов с одинаковым IP — даже на разных нодах.

**2. Pod'ы могут общаться друг с другом напрямую, без NAT.**

Pod на node-1 может обратиться к Pod на node-2 по его IP. Без NAT, без прокси.

**3. Агенты на ноде (kubelet, kube-proxy) могут общаться с Pod'ами.**

kubelet должен уметь обращаться к Pod'ам для health checks.

**4. Pod'ы в host network видят сеть ноды.**

Если Pod запущен с `hostNetwork: true`, он использует сеть ноды.

### 🎯 Как это реализуется

Kubernetes **не реализует** сеть сам. Он определяет **требования**, а реализация — задача **CNI-плагина**.

**Что делает CNI:**

- Выделяет IP каждому Pod'у.
- Настраивает маршруты между нодами.
- Обеспечивает связность Pod'ов на разных нодах.

**Без CNI:** Pod'ы на одной ноде видят друг друга, но на разных — нет.

### 📊 Схема сети

```
┌─────────────────────────────────────────────────────────────────┐
│                     КЛАСТЕР KUBERNETES                          │
│                                                                  │
│  ┌────────────────────┐          ┌────────────────────┐        │
│  │  WORKER NODE 1     │          │  WORKER NODE 2     │        │
│  │                    │          │                    │        │
│  │  Pod A: 10.244.1.5 │          │  Pod C: 10.244.2.5 │        │
│  │  Pod B: 10.244.1.6 │          │  Pod D: 10.244.2.6 │        │
│  │                    │          │                    │        │
│  │  ┌──────────────┐  │          │  ┌──────────────┐  │        │
│  │  │  CNI-плагин  │  │◄────────►│  │  CNI-плагин  │  │        │
│  │  └──────────────┘  │          │  └──────────────┘  │        │
│  │                    │          │                    │        │
│  └─────────┬──────────┘          └──────────┬─────────┘        │
│            │                                 │                  │
│            │      Overlay / Routing          │                  │
│            └─────────────┬───────────────────┘                  │
│                          │                                       │
└──────────────────────────┼──────────────────────────────────────┘
                           │
                           ▼
                    Физическая сеть
```

**Ключевое:** CNI-плагин на каждой ноде создаёт **виртуальную сеть**, в которой все Pod'ы видят друг друга, как будто они в одной локальной сети.

### 🎯 Что происходит при создании Pod'а

1. kubelet на ноде получает Pod.
2. kubelet вызывает CNI-плагин: «настрой сеть для этого Pod'а».
3. CNI-плагин:
   - Создаёт **network namespace** для Pod'а.
   - Создаёт **veth pair** (виртуальный кабель).
   - Подключает один конец к bridge/routing на ноде.
   - Выделяет **IP** из подсети ноды.
   - Настраивает **маршруты** для связи с другими нодами.
4. kubelet запускает контейнеры в этом namespace.

**Результат:** Pod имеет свой IP, видит другие Pod'ы, имеет доступ к Service'ам.

### 🎯 Что такое veth pair

**veth pair** — пара виртуальных ethernet-интерфейсов, соединённых «кабелем». Что отправлено в один конец — приходит в другой.

**Как используется:**

```
┌─────────────────────────────────────┐
│  Pod (network namespace)             │
│                                      │
│  eth0 (10.244.1.5)                   │
└───────────────┬─────────────────────┘
                │
                │ veth pair
                │
┌───────────────▼─────────────────────┐
│  Нода                                │
│                                      │
│  veth-abc123 (виртуальный интерфейс) │
│  Мост cni0 (10.244.1.1)              │
└──────────────────────────────────────┘
```

**Что происходит:**

1. Один конец veth — в network namespace Pod'а, называется `eth0`.
2. Другой конец — на ноде, называется `veth-xxx`.
3. Пакет из Pod'а идёт в veth, выходит на ноде, попадает в bridge `cni0`.
4. Из bridge пакет идёт либо к другому Pod'у на ноде, либо на маршрутизацию к другой ноде.

Это похоже на Docker bridge (Глава 4), но масштабируется на кластер.

### 🎯 Overlay vs Routing

CNI-плагины используют два подхода:

**1. Overlay (VXLAN):**

- Пакеты Pod'ов инкапсулируются в VXLAN.
- VXLAN-пакеты ходят по физической сети между нодами.
- На другой ноде распаковываются и доставляются Pod'у.

**Плюсы:**

- Работает на любой физической сети.
- Не требует настройки маршрутизации.

**Минусы:**

- Инкапсуляция добавляет накладные расходы (MTU уменьшается).
- Чуть медленнее.

**Примеры:** Flannel (VXLAN), Calico (VXLAN), Cilium (VXLAN).

**2. Routing (BGP):**

- Ноды обмениваются маршрутами через BGP.
- Пакеты Pod'ов идут напрямую по физической сети.
- Нет инкапсуляции.

**Плюсы:**

- Быстрее (нет инкапсуляции).
- Меньше накладные расходы.

**Минусы:**

- Требует поддержки BGP в физической сети.
- Сложнее настраивать.

**Примеры:** Calico (BGP), Cilium (BGP).

### 💡 Практика: что важно понять про сетевую модель

**✅ ОБЯЗАТЕЛЬНО:**

1. **Kubernetes не реализует сеть сам.** Реализация — задача CNI-плагина.
2. **Каждый Pod имеет уникальный IP.**
3. **Pod'ы общаются напрямую, без NAT.**

**👍 СТОИТ:**

4. **Понимать veth pair.** Это фундамент контейнерной сети.
5. **Знать разницу Overlay vs Routing.**

**❌ НЕ ДЕЛАЙ:**

6. **Не хардкодь IP Pod'ов.** Они меняются.
7. **Не полагайся на то, что Pod'ы на одной ноде.** Scheduler может разместить где угодно.

### Где мы сейчас

Мы разобрали сетевую модель. Теперь — **CNI-плагины** — конкретные реализации.

---

## 10.2 CNI: как Pod'ы получают IP

### 🔌 Проблема: какой CNI выбрать

Kubernetes не поставляется с CNI. Ты должен **выбрать** и **установить** плагин. Их несколько:

- **Flannel** — простой, overlay.
- **Calico** — мощный, routing + network policies.
- **Cilium** — eBPF-based, самый быстрый.
- **Weave** — простой, mesh.
- **kube-router** — routing.

**Как выбрать?**

### 📊 Что такое CNI

**CNI (Container Network Interface)** — это **стандарт** для настройки сети контейнеров. Определяет:

- Как kubelet вызывает плагин.
- Какой формат JSON-конфига.
- Какие команды должны поддерживать плагины (`ADD`, `DEL`, `CHECK`).

**Любой CNI-плагин** реализует этот стандарт. kubelet работает с любым из них.

### 📊 Сравнение CNI-плагинов

| Плагин | Тип | Network Policies | Производительность | Сложность |
|:---|:---|:---|:---|:---|
| **Flannel** | Overlay (VXLAN) | ❌ (только с Calico) | Средняя | Низкая |
| **Calico** | Routing (BGP) + Overlay | ✅ Полная | Высокая | Средняя |
| **Cilium** | eBPF | ✅ Полная + L7 | Очень высокая | Высокая |
| **Weave** | Overlay (mesh) | ✅ | Средняя | Низкая |
| **kube-router** | Routing | ✅ | Высокая | Низкая |

### 🎯 Flannel

**Самый простой CNI.**

**Как работает:**

- Создаёт VXLAN-туннели между нодами.
- Каждая нода получает подсеть (например, `10.244.1.0/24`).
- Pod'ы на ноде получают IP из этой подсети.
- Пакеты между нодами идут через VXLAN.

**Плюсы:**

- Простой.
- Работает на любой сети.

**Минусы:**

- **Нет Network Policies** (по умолчанию).
- Overlay добавляет накладные расходы.

**Когда использовать:** для простых кластеров, где не нужны Network Policies.

### 🎯 Calico

**Самый популярный CNI для production.**

**Как работает:**

- **Routing (BGP)** — по умолчанию. Ноды обмениваются маршрутами через BGP.
- **Overlay (VXLAN/IPIP)** — если BGP не работает.
- Поддерживает **Network Policies** (L3/L4).
- Поддерживает **eBPF** (dataplane) в новых версиях.

**Плюсы:**

- Высокая производительность (routing, no encapsulation).
- Полные Network Policies.
- Гибкая конфигурация.

**Минусы:**

- Сложнее Flannel.
- Требует BGP (или включения VXLAN).

**Когда использовать:** для production, где нужны Network Policies и производительность.

### 🎯 Cilium

**Самый современный CNI на eBPF.**

**Как работает:**

- Использует **eBPF** — программы в ядре Linux.
- Обрабатывает пакеты **без iptables**.
- Поддерживает Network Policies на **L7** (HTTP, gRPC).
- Встроенный **Service Mesh** (замена Istio в простых случаях).
- Встроенная **observability** (Hubble).

**Плюсы:**

- **Самая высокая производительность** (eBPF быстрее iptables).
- Network Policies на L7.
- Service Mesh встроен.
- Отличная observability.

**Минусы:**

- Требует современное ядро (5.x+).
- Сложнее настраивать.
- Меньше документации.

**Когда использовать:** для современных кластеров, где важна производительность и L7 policies.

### 🎯 Как установить CNI

**Flannel:**

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

**Calico:**

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

**Cilium:**

```bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --namespace kube-system
```

**Проверка:**

```bash
# Pod'ы CNI
kubectl get pods -n kube-system | grep -E "flannel|calico|cilium"

# IP Pod'ов
kubectl get pods -o wide
# NAME    IP           NODE
# myapp   10.244.1.5   node-1
# myapp   10.244.2.5   node-2
```

### 🎯 Как узнать, какой CNI используется

```bash
# Посмотреть Pod'ы в kube-system
kubectl get pods -n kube-system

# Если есть calico-node — Calico
# Если есть cilium — Cilium
# Если есть kube-flannel — Flannel

# Посмотреть CNI-конфиг на ноде
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conflist
```

### 🎯 IP-адресация

CNI-плагин выделяет IP из **podCIDR** — подсети, заданной в кластере.

**Узнать podCIDR:**

```bash
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'
# 10.244.1.0/24 10.244.2.0/24 10.244.3.0/24
```

**Каждая нода получает свою подсеть.** Pod'ы на node-1 — `10.244.1.x`, на node-2 — `10.244.2.x`.

### 💡 Практика: как правильно выбрать CNI

**✅ ОБЯЗАТЕЛЬНО:**

1. **Calico для production.** Стандарт де-факто, network policies, высокая производительность.
2. **Cilium для современных кластеров.** Если ядро 5.x+, важна производительность, L7 policies.
3. **Flannel для простых кластеров.** Если не нужны Network Policies.

**👍 СТОИТ:**

4. **Проверять совместимость с managed K8s.** В EKS/GKE/AKS свой CNI (но можно заменить).
5. **Планировать podCIDR заранее.** Не пересекать с сетью нод.

**❌ НЕ ДЕЛАЙ:**

6. **Не устанавливай два CNI одновременно.** Конфликт.
7. **Не используй Flannel, если нужны Network Policies.**

### Где мы сейчас

Мы разобрали CNI. Теперь — **Service** — стабильный доступ к Pod'ам.

---

## 10.3 Service: стабильный доступ к Pod'ам

### 🔌 Проблема: IP Pod'ов меняется

В Главе 8 мы узнали: Pod'ы эфемерны, их IP меняется при пересоздании. Как приложение A может обратиться к приложению B, если IP B меняется?

**Решение:** Service.

### 📦 Что такое Service

**Service** — это абстракция, которая предоставляет **стабильный IP** и **DNS-имя** для группы Pod'ов.

```
Без Service:                    С Service:

App ──► 10.244.1.5:8080        App ──► myapp:80 (ClusterIP)
        (IP Pod'а)                      │
        (меняется!)                     │ kube-proxy
                                        ▼
                                 10.244.1.5:8080  (Pod 1)
                                 10.244.2.5:8080  (Pod 2)
                                 10.244.3.5:8080  (Pod 3)
```

**Service:**

- Имеет **стабильный ClusterIP** — виртуальный IP, не привязанный к Pod'ам.
- Имеет **DNS-имя** — `<service>.<namespace>.svc.cluster.local`.
- Автоматически **находит Pod'ы** по selector.
- **Балансирует** трафик между Pod'ами.

### 📊 Как это работает

**1. Создание Service.**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - port: 80          # порт Service
      targetPort: 8080  # порт Pod'а
```

**2. Kubernetes выделяет ClusterIP.**

```
ClusterIP: 10.96.123.45
```

**3. Endpoints Controller находит Pod'ы.**

```bash
kubectl get endpoints myapp
# NAME    ENDPOINTS
# myapp   10.244.1.5:8080,10.244.2.5:8080,10.244.3.5:8080
```

**4. kube-proxy настраивает правила.**

```
Пакет на 10.96.123.45:80 → перенаправить на один из Pod'ов
```

**5. Клиент обращается к Service.**

```bash
curl http://myapp:80
# Трафик идёт на один из Pod'ов (round-robin)
```

### 🎯 Ключевые понятия

**ClusterIP:**

- Виртуальный IP. **Не существует на сетевых интерфейсах.**
- Существует только как правило в iptables/IPVS.
- Работает только внутри кластера.

**port:** порт Service. Клиент обращается к нему.

**targetPort:** порт Pod'а. Куда kube-proxy перенаправляет.

**selector:** labels, по которым Service находит Pod'ы.

**Endpoints:** список IP:port Pod'ов, соответствующих selector.

### 🎯 Что если selector не совпадает

**Проблема:** Service ищет Pod'ы с `app=myapp`, а у Pod'ов `app=my-app`.

**Что произойдёт:**

```bash
kubectl get endpoints myapp
# NAME    ENDPOINTS
# myapp   <none>       ← пусто!
```

**Service существует**, но трафик некуда идти. `curl` вернёт `Connection refused`.

**Как проверить:**

```bash
# 1. Посмотреть selector Service
kubectl get service myapp -o jsonpath='{.spec.selector}'
# {"app":"myapp"}

# 2. Посмотреть labels Pod'ов
kubectl get pods --show-labels
# NAME    LABELS
# myapp   app=my-app  ← не совпадает!

# 3. Проверить endpoints
kubectl get endpoints myapp
# <none>
```

**Решение:** исправить selector или labels.

### 🎯 Что если targetPort неправильный

**Проблема:** Service отправляет на порт 80, а Pod слушает 8080.

**Что произойдёт:**

- Endpoints будут **правильные** (Pod'ы найдены).
- Но трафик на порт 80 Pod'а **не доходит** — там никто не слушает.
- `curl` вернёт `Connection refused` или `timeout`.

**Как проверить:**

```bash
# 1. Посмотреть targetPort
kubectl get service myapp -o jsonpath='{.spec.ports[*].targetPort}'
# 80

# 2. Проверить, что слушает Pod
kubectl exec -it myapp-xxx -- ss -tlnp
# LISTEN 0.0.0.0:8080  ← слушает 8080, не 80!

# 3. Исправить
kubectl edit service myapp
# Изменить targetPort: 8080
```

### 🎯 Что если приложение слушает localhost

**Проблема:** приложение в Pod'е слушает `127.0.0.1:8080`, а не `0.0.0.0:8080`.

**Что произойдёт:** трафик от kube-proxy на IP Pod'а не дойдёт до приложения.

**Как проверить:**

```bash
kubectl exec -it myapp-xxx -- ss -tlnp
# LISTEN 127.0.0.1:8080  ← только localhost!
```

**Решение:** настроить приложение слушать `0.0.0.0`. В Go:

```go
// ❌ Только localhost
http.ListenAndServe("127.0.0.1:8080", nil)

// ✅ Все интерфейсы
http.ListenAndServe(":8080", nil)         // по умолчанию 0.0.0.0
http.ListenAndServe("0.0.0.0:8080", nil)  // явно
```

### 🔬 Практика: Service в действии

```bash
# 1. Deployment
kubectl create deployment nginx --image=nginx --replicas=3

# 2. Service
kubectl expose deployment nginx --port=80 --target-port=80

# 3. Посмотреть Service
kubectl get service nginx
# NAME    TYPE        CLUSTER-IP      PORT(S)
# nginx   ClusterIP   10.96.123.45    80/TCP

# 4. Endpoints
kubectl get endpoints nginx
# NAME    ENDPOINTS
# nginx   10.244.1.5:80,10.244.2.5:80,10.244.3.5:80

# 5. Проверить доступ
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
# Внутри:
wget -qO- http://nginx
# <!DOCTYPE html>...

# 6. Проверить DNS
nslookup nginx
# Name:      nginx
# Address 1: 10.96.123.45 nginx.default.svc.cluster.local
```

### 💡 Практика: как правильно работать с Service

**✅ ОБЯЗАТЕЛЬНО:**

1. **Проверять Endpoints** при проблемах с доступом: `kubectl get endpoints`.
2. **Правильно настраивать selector** — совпадать с labels Pod'ов.
3. **Правильно настраивать targetPort** — совпадать с портом приложения.

**👍 СТОИТ:**

4. **Использовать `port` = `targetPort`** для простоты.
5. **Использовать DNS-имена**, не ClusterIP.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй ClusterIP снаружи кластера.** Не работает.
7. **Не игнорируй пустые Endpoints.** Это признак проблемы.

### Где мы сейчас

Мы разобрали Service на уровне концепции. Теперь — **kube-proxy** — как это работает изнутри.

---

## 10.4 kube-proxy: как работает Service изнутри

### 🔌 Проблема: как виртуальный IP превращается в реальный

ClusterIP — **виртуальный**. Его нет ни на одном интерфейсе. Как пакет, отправленный на ClusterIP, доходит до реального Pod'а?

**Ответ:** kube-proxy.

### 📊 Что такое kube-proxy

**kube-proxy** — это сетевой агент, работающий на **каждой ноде**. Он реализует Service'ы.

**Что делает:**

- Watch API — следит за Service'ами и Endpoints.
- Настраивает правила в iptables (или IPVS).
- Правила: «трафик на ClusterIP:port → один из Pod'ов».

**Что НЕ делает:**

- Не проксирует трафик сам.
- Не балансирует (это делает ядро).

kube-proxy только **настраивает правила**. Ядро делает всю работу.

### 🎯 Режимы kube-proxy

**1. iptables (по умолчанию долгое время):**

- kube-proxy настраивает правила в iptables.
- Пакет на ClusterIP → DNAT в случайный Pod.

**2. IPVS (с K8s 1.11):**

- Использует IPVS (IP Virtual Server).
- Быстрее и лучше масштабируется.
- Лучше для больших кластеров (1000+ Service'ов).

**3. eBPF (Cilium):**

- Полная замена iptables на eBPF.
- Самая высокая производительность.

### 🎯 iptables: как работает

**Пример Service:**

```
Service myapp:
  ClusterIP: 10.96.123.45
  port: 80
  Endpoints: [10.244.1.5:8080, 10.244.2.5:8080, 10.244.3.5:8080]
```

**kube-proxy создаёт цепочку правил:**

```
KUBE-SERVICES
  ├─ Пакет на 10.96.123.45:80? → KUBE-SVC-MYAPP
  └─ ...

KUBE-SVC-MYAPP
  ├─ 33% → KUBE-SEP-1 (10.244.1.5:8080)
  ├─ 33% → KUBE-SEP-2 (10.244.2.5:8080)
  └─ 33% → KUBE-SEP-3 (10.244.3.5:8080)

KUBE-SEP-1
  └─ DNAT: 10.96.123.45:80 → 10.244.1.5:8080
```

**Что происходит с пакетом:**

1. Клиент отправляет пакет на `10.96.123.45:80`.
2. Ядро видит пакет в цепочке KUBE-SERVICES.
3. Перенаправляет в KUBE-SVC-MYAPP.
4. KUBE-SVC-MYAPP выбирает один из SEP (random).
5. KUBE-SEP делает DNAT: заменяет destination на IP Pod'а.
6. Пакет идёт на реальный Pod.

**Всё происходит в ядре**, без участия процессов.

### 🎯 IPVS: как работает

**IPVS** — это встроенный в ядро Linux балансировщик.

**Как работает:**

1. kube-proxy создаёт **virtual server** для каждого Service.
2. Virtual server имеет ClusterIP.
3. **Real servers** — Pod'ы.
4. Алгоритм балансировки — round-robin, least connections и т.д.

**Преимущества IPVS:**

- **O(1) lookup** вместо O(n) в iptables.
- **Лучше масштабируется.** 10 000 Service'ов — работает.
- **Больше алгоритмов балансировки.**

**Недостатки:**

- Сложнее настраивать.
- Требует модуль ядра IPVS.

**Как включить:**

```bash
# В конфиге kube-proxy
--proxy-mode=ipvs
```

**Проверить:**

```bash
# На ноде
ipvsadm -L -n
# TCP  10.96.123.45:80 rr
#   -> 10.244.1.5:8080             Masq    1      0          0
#   -> 10.244.2.5:8080             Masq    1      0          0
#   -> 10.244.3.5:8080             Masq    1      0          0
```

### 🎯 Проблема: как клиент видит source IP

**Важный нюанс:** при DNAT в iptables source IP **сохраняется**. Pod видит реальный IP клиента.

**Но:** для `externalTrafficPolicy: Cluster` (для NodePort/LoadBalancer) source IP **теряется** — заменяется на IP ноды.

**Как сохранить source IP:**

```yaml
spec:
  externalTrafficPolicy: Local
```

**Что означает:** трафик идёт только на Pod'ы **на той же ноде**. Source IP сохраняется.

**Минус:** если на ноде нет Pod'ов — трафик теряется.

Разберём подробно в подглаве 10.5.

### 🎯 Почему round-robin не всегда

**kube-proxy использует random selection**, не строгий round-robin. Это значит:

- Не гарантируется равномерное распределение.
- Соединения распределяются **случайно**.
- При большом количестве соединений — распределение близко к равномерному.

**Для долгоживущих соединений** (например, gRPC streams) — клиент подключается один раз и держит соединение. Все запросы идут на один Pod.

**Решение:** использовать клиентский load balancing (например, gRPC с resolver) или Service Mesh.

### 🎯 Session Affinity

**Session Affinity** — привязка клиента к одному Pod'у.

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800    # 3 часа
```

**Что произойдёт:**

- Первый запрос клиента → Pod 1.
- Все последующие запросы с того же IP → Pod 1.
- Через 3 часа — снова random.

**Когда использовать:** для stateful-приложений, где сессия хранится в Pod'е.

**Когда НЕ использовать:** для stateless-приложений. Мешает балансировке.

### 🎯 Как посмотреть правила

**iptables:**

```bash
# На ноде (нужен root)
iptables -t nat -L KUBE-SERVICES -n | grep myapp
# KUBE-SVC-XXXX  tcp -- 0.0.0.0/0  10.96.123.45  tcp dpt:80

iptables -t nat -L KUBE-SVC-XXXX -n
# KUBE-SEP-AAAA  tcp -- 0.0.0.0/0  0.0.0.0/0  statistic mode random probability 0.33
# KUBE-SEP-BBBB  tcp -- 0.0.0.0/0  0.0.0.0/0  statistic mode random probability 0.5
# KUBE-SEP-CCCC  tcp -- 0.0.0.0/0  0.0.0.0/0
```

**IPVS:**

```bash
ipvsadm -L -n
```

**В Pod'е (для отладки):**

```bash
# Посмотреть, какой ClusterIP
kubectl get service myapp
# NAME    TYPE        CLUSTER-IP      PORT(S)
# myapp   ClusterIP   10.96.123.45    80/TCP

# Проверить доступ
wget -qO- http://10.96.123.45
```

### 🔬 Практика: смотрим kube-proxy

```bash
# 1. Pod'ы kube-proxy
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# 2. Логи
kubectl logs -n kube-system <kube-proxy-pod>

# 3. Конфиг
kubectl get configmap -n kube-system kube-proxy -o yaml
# Там proxy-mode: iptables или ipvs

# 4. На ноде — правила
# (нужен доступ к ноде)
iptables -t nat -L | grep KUBE | head
```

### 💡 Практика: как правильно работать с kube-proxy

**✅ ОБЯЗАТЕЛЬНО:**

1. **Знать режим** (iptables или IPVS).
2. **Понимать, что ClusterIP — виртуальный.**
3. **Проверять iptables при проблемах.**

**👍 СТОИТ:**

4. **IPVS для больших кластеров** (1000+ Service'ов).
5. **Cilium с eBPF** для производительности.

**❌ НЕ ДЕЛАЙ:**

6. **Не меняй iptables вручную.** kube-proxy перезапишет.
7. **Не полагайся на точный round-robin.** Random selection.

### Где мы сейчас

Мы разобрали kube-proxy. Теперь — **типы Service'ов** — подробнее.

---

## 10.5 ClusterIP, NodePort, LoadBalancer, ExternalName

### 🔌 Проблема: как открыть сервис наружу

ClusterIP работает **только внутри кластера**. Как сделать сервис доступным снаружи?

Четыре типа Service:

- **ClusterIP** — только внутри.
- **NodePort** — порт на каждой ноде.
- **LoadBalancer** — внешний LB.
- **ExternalName** — CNAME на внешний сервис.

### 📊 ClusterIP

**По умолчанию.** Только внутри кластера.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
```

**Доступ:** только из Pod'ов в кластере.

**Когда использовать:** для **внутренних** сервисов — базы данных, кэши, микросервисы, не доступные снаружи.

### 📊 NodePort

**Порт на каждой ноде.** Доступ через `<node-ip>:<nodePort>`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080      # 30000-32767
```

**Доступ:**

- `node-1:30080`
- `node-2:30080`
- `node-3:30080`

Все ведут на Pod'ы.

**Особенности:**

- **Порт открыт на каждой ноде.** Даже если Pod'ов на ней нет — трафик проксируется на другие ноды.
- **Порт должен быть уникален** в кластере.
- **Диапазон 30000-32767** (по умолчанию).

**Когда использовать:**

- **Dev/staging** — простой доступ без LB.
- **On-premise**, где нет облачного LB.
- **Когда нужен доступ по IP ноды.**

**Проблемы:**

- **Source IP теряется** (заменяется на IP ноды).
- **Порт нестандартный** — неудобно для пользователей.
- **Нет TLS-терминации.**

### 📊 LoadBalancer

**Внешний LB от облачного провайдера.**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
```

**Что произойдёт:**

1. Kubernetes просит облачного провайдера создать LB.
2. LB получает **внешний IP**.
3. LB направляет трафик на ноды (NodePort).
4. Ноды направляют трафик на Pod'ы.

**Доступ:** через внешний IP LB.

**Когда использовать:**

- **Production в облаке** (AWS, GCP, Azure).
- **Когда нужен стабильный внешний IP.**
- **TCP/UDP трафик.**

**Проблемы:**

- **Дорого.** Каждый LB — деньги.
- **Не поддерживается on-premise** без MetalLB.

### 🎯 `externalTrafficPolicy`

**Важный параметр** для NodePort и LoadBalancer.

**Cluster (по умолчанию):**

- Трафик может идти на **любую** ноду, потом на Pod.
- Source IP **теряется** (заменяется на IP ноды).
- Равномерная балансировка.

```
Client → Node-1:30080 → Pod на Node-2
                        (source IP = Node-1)
```

**Local:**

- Трафик идёт только на Pod'ы **на той же ноде**.
- Source IP **сохраняется**.
- Меньше hops.

```
Client → Node-1:30080 → Pod на Node-1
                        (source IP = Client)
```

**Если на ноде нет Pod'ов — трафик теряется.**

**Когда использовать:**

- **Local** — если важно сохранить source IP (rate limiting, аудит).
- **Cluster** — для равномерной балансировки.

### 📊 ExternalName

**CNAME на внешний сервис.** Не имеет ClusterIP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com
```

**Что произойдёт:**

- DNS-запрос к `external-db` → CNAME `db.example.com`.
- Pod обращается к `external-db`, получает ответ от `db.example.com`.

**Когда использовать:**

- **Интеграция с внешним сервисом.**
- **Миграция:** пока сервис снаружи, потом переехал в кластер.

### 🎯 MetalLB для on-premise

**MetalLB** — реализация LoadBalancer для on-premise кластеров.

**Как работает:**

- Выделяет IP из пула.
- Анонсирует через ARP (L2) или BGP (L3).
- Трафик идёт на ноду, потом на Pod.

**Установка:**

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml
```

**Конфиг:**

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.100-192.168.1.200
```

Теперь LoadBalancer Service'ы получают IP из этого пула.

### 📊 Сравнение

| Тип | Доступ | Source IP | TLS | Стоимость | Когда |
|:---|:---|:---|:---|:---|:---|
| **ClusterIP** | Только внутри | ✅ | ❌ | 0 | Внутренние |
| **NodePort** | `<node>:<port>` | ❌ (Cluster) / ✅ (Local) | ❌ | 0 | Dev, on-prem |
| **LoadBalancer** | Внешний IP | ❌/✅ | ❌ | $ | Production в облаке |
| **ExternalName** | CNAME | N/A | N/A | 0 | Внешний сервис |

**Важно:** ни один из них **не делает TLS-терминацию**. Для этого нужен **Ingress** (подглава 10.6).

### 🔬 Практика: NodePort

```bash
# 1. Service типа NodePort
kubectl create deployment nginx --image=nginx --replicas=3
kubectl expose deployment nginx --type=NodePort --port=80

# 2. Посмотреть
kubectl get service nginx
# NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# nginx   NodePort   10.96.123.45    <none>        80:31234/TCP   10s
#                                                   ^^^^^
#                                                   NodePort

# 3. Проверить доступ
curl http://<node-ip>:31234

# 4. Source IP
kubectl logs <nginx-pod>
# Если externalTrafficPolicy: Cluster — IP ноды
# Если Local — реальный IP клиента
```

### 💡 Практика: как правильно выбирать тип

**✅ ОБЯЗАТЕЛЬНО:**

1. **ClusterIP для внутренних** — 90% случаев.
2. **LoadBalancer для TCP/UDP** (в облаке).
3. **Ingress для HTTP/HTTPS** (подглава 10.6).

**👍 СТОИТ:**

4. **MetalLB для on-premise LoadBalancer.**
5. **`externalTrafficPolicy: Local`** для сохранения source IP.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй NodePort в production.** Ingress лучше.
7. **Не используй LoadBalancer для каждого сервиса.** Дорого. Используй Ingress.

### Где мы сейчас

Мы разобрали типы Service'ов. Теперь — **Ingress** — HTTP-маршрутизация.

---

## 10.6 Ingress: HTTP-маршрутизация

### 🔌 Проблема: один IP для многих сервисов

У тебя 10 микросервисов. Каждый хочет быть доступным снаружи. Если использовать LoadBalancer для каждого — 10 IP, 10 LB, 10× деньги.

**Решение:** Ingress.

### 📦 Что такое Ingress

**Ingress** — это **HTTP/HTTPS-роутер** для Kubernetes. Он принимает трафик на **один IP** и маршрутизирует по правилам:

- По хосту: `api.example.com` → api-service, `www.example.com` → web-service.
- По пути: `/api/*` → api-service, `/static/*` → static-service.

```
                    ┌──────────────────┐
                    │   Ingress        │
                    │   Controller     │
                    │  (Nginx, Traefik)│
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
         api.example.com  /api/*       www.example.com
              │              │              │
              ▼              ▼              ▼
         api-service    api-service    web-service
```

### 🎯 Ingress vs Ingress Controller

**Важное различие:**

**Ingress** — это **ресурс** (манифест). Определяет правила маршрутизации.

**Ingress Controller** — это **приложение** (Pod), которое реализует эти правила.

**Kubernetes не имеет встроенного Ingress Controller.** Ты должен установить его сам.

**Популярные Ingress Controllers:**

| Controller | Особенности |
|:---|:---|
| **NGINX Ingress** | Самый популярный, стандарт де-факто |
| **Traefik** | Динамическая конфигурация, auto-discovery |
| **HAProxy Ingress** | Высокая производительность |
| **Istio Gateway** | Часть Service Mesh |
| **Cilium Ingress** | На eBPF |

**Установка NGINX Ingress:**

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
```

**Проверка:**

```bash
kubectl get pods -n ingress-nginx
# NAME                                       READY   STATUS
# ingress-nginx-controller-xxxxxxxxx-xxxxx   1/1     Running

kubectl get service -n ingress-nginx
# NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP
# ingress-nginx-controller   LoadBalancer   10.96.x.x       <external-ip>
```

### 🎯 Ingress ресурс

**Пример: маршрутизация по хостам.**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

**Что произойдёт:**

- `api.example.com` → `api-service:80`.
- `www.example.com` → `web-service:80`.

**Пример: маршрутизация по путям.**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /static
            pathType: Prefix
            backend:
              service:
                name: static-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

**Что произойдёт:**

- `example.com/api/*` → `api-service`.
- `example.com/static/*` → `static-service`.
- `example.com/*` → `web-service`.

### 🎯 pathType

| Тип | Что означает |
|:---|:---|
| `Exact` | Точное совпадение пути |
| `Prefix` | Совпадение по префиксу (по сегментам) |
| `ImplementationSpecific` | Зависит от контроллера |

**Пример:**

- `Prefix /api` — совпадёт с `/api`, `/api/`, `/api/users`, `/api/users/1`. Но **не** с `/apixyz`.
- `Exact /api` — совпадает только с `/api`.

**По умолчанию:** `ImplementationSpecific`, но лучше указывать явно.

### 🎯 TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - example.com
      secretName: example-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

**Что нужно:**

- Secret `example-tls` с `tls.crt` и `tls.key`.

**Создать Secret:**

```bash
kubectl create secret tls example-tls \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

### 🎯 cert-manager: автоматические TLS-сертификаты

**cert-manager** автоматически получает и обновляет сертификаты (например, через Let's Encrypt).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - example.com
      secretName: example-tls          # cert-manager создаст автоматически
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

**cert-manager** создаст `example-tls` из Let's Encrypt и будет обновлять.

Подробно разберём в Главе 18 (Безопасность).

### 🎯 Ingress vs Service LoadBalancer

| Аспект | Ingress | Service LoadBalancer |
|:---|:---|:---|
| Протокол | HTTP/HTTPS | TCP/UDP |
| Один IP для многих | ✅ | ❌ (каждый — свой IP) |
| TLS-терминация | ✅ | ❌ |
| Path-based routing | ✅ | ❌ |
| Host-based routing | ✅ | ❌ |
| Стоимость | 1 LB | N LB |

**Правило:**

- **HTTP/HTTPS → Ingress.**
- **TCP/UDP → LoadBalancer.**
- **Внутренние → ClusterIP.**

### 🔬 Практика: Ingress

**1. Установить NGINX Ingress Controller:**

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
```

**2. Развернуть приложение:**

```bash
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80
```

**3. Создать Ingress:**

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: nginx
  rules:
    - host: example.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
EOF
```

**4. Проверить:**

```bash
# Получить IP Ingress Controller
kubectl get service -n ingress-nginx ingress-nginx-controller

# Если Minikube/Kind — используй port-forward
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80

# Проверить
curl -H "Host: example.local" http://localhost:8080
```

### 💡 Практика: как правильно работать с Ingress

**✅ ОБЯЗАТЕЛЬНО:**

1. **Установить Ingress Controller.** Без него Ingress — просто манифест.
2. **Указывать `ingressClassName`.**
3. **Использовать `pathType: Prefix`** явно.
4. **cert-manager для TLS.**

**👍 СТОИТ:**

5. **Один Ingress на приложение или на домен.**
6. **Annotations для настройки Ingress Controller** (timeouts, rate limiting).

**❌ НЕ ДЕЛАЙ:**

7. **Не используй Ingress для TCP/UDP.** Только HTTP/HTTPS.
8. **Не забывай про `pathType`.** По умолчанию — ImplementationSpecific, может отличаться.

### Где мы сейчас

Мы разобрали Ingress. Теперь — **CoreDNS** — DNS в кластере.

---

## 10.7 CoreDNS: DNS внутри кластера

### 🔌 Проблема: как Pod'ы находят друг друга

Pod'ы обращаются к Service'ам по именам. Как работает DNS?

**Ответ:** CoreDNS.

### 📦 Что такое CoreDNS

**CoreDNS** — это DNS-сервер, работающий **внутри кластера**. Он:

- Резолвит имена Service'ов в ClusterIP.
- Резолвит имена Pod'ов в их IP (для headless Service).
- Пересылает внешние DNS-запросы (например, `google.com`) на upstream DNS.

**Работает как Deployment в namespace `kube-system`:**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
# NAME                       READY   STATUS
# coredns-xxxxxxxxxx-xxxxx   1/1     Running
# coredns-xxxxxxxxxx-yyyyy   1/1     Running
```

**Service:**

```bash
kubectl get service -n kube-system kube-dns
# NAME       TYPE        CLUSTER-IP   PORT(S)
# kube-dns   ClusterIP   10.96.0.10   53/UDP,53/TCP,9153/TCP
```

**ClusterIP `10.96.0.10`** — адрес DNS-сервера. Все Pod'ы его знают.

### 🎯 Как Pod'ы узнают про CoreDNS

**kubelet** настраивает `/etc/resolv.conf` в каждом Pod'е:

```bash
kubectl exec -it myapp -- cat /etc/resolv.conf
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

**Что означает:**

- **nameserver** — адрес CoreDNS.
- **search** — какие домены добавлять при поиске.
- **ndots** — при каком количестве точек считать имя полным.

### 🎯 DNS-имена в Kubernetes

**Формат:**

```
<service>.<namespace>.svc.cluster.local
```

**Примеры:**

| Запрос | Резолвится в |
|:---|:---|
| `myapp` (в том же namespace) | ClusterIP Service `myapp` |
| `myapp.default` | ClusterIP Service `myapp` в `default` |
| `myapp.default.svc` | То же |
| `myapp.default.svc.cluster.local` | То же |
| `google.com` | Внешний DNS |

**Search domains и `ndots:5`:**

Когда Pod делает запрос `myapp`:
1. `myapp.default.svc.cluster.local` — если Pod в `default`.
2. `myapp.svc.cluster.local`
3. `myapp.cluster.local`
4. `myapp` (как есть, через upstream DNS)

`ndots:5` означает: если в имени **меньше 5 точек**, сначала пробовать search domains, потом как есть.

**Пример:** `myapp` (0 точек) → сначала как `myapp.<search-domains>`.

**Пример:** `google.com` (1 точка) → тоже сначала как search domains. Может быть неожиданно.

### 🎯 Headless Service и DNS

Для **headless Service** (`clusterIP: None`) DNS возвращает **IP всех Pod'ов**, не балансирует.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
```

**Что произойдёт:**

- `postgres.default.svc.cluster.local` → IP **всех** Pod'ов `postgres`.
- `postgres-0.postgres.default.svc.cluster.local` → IP Pod'а 0 (для StatefulSet).
- `postgres-1.postgres.default.svc.cluster.local` → IP Pod'а 1.

**Клиент сам выбирает**, к какому Pod'у идти.

**Когда использовать:**

- **StatefulSet** — для прямого доступа к Pod'ам.
- **Peer discovery** — например, Kafka broker discovery.
- **Кастомные балансировщики.**

### 🎯 DNS для Pod'ов

По умолчанию Pod'ы **не имеют DNS-имён** (только IP). Если нужны DNS-имена — используй headless Service.

**Для StatefulSet** Pod'ы получают DNS:

```
<statefulset-name>-<ordinal>.<headless-service>.<namespace>.svc.cluster.local
```

**Пример:** StatefulSet `postgres` с headless Service `postgres`:

- `postgres-0.postgres.default.svc.cluster.local`
- `postgres-1.postgres.default.svc.cluster.local`
- `postgres-2.postgres.default.svc.cluster.local`

### 🎯 CoreDNS ConfigMap

**Конфигурация CoreDNS** — в ConfigMap `coredns` в namespace `kube-system`.

```bash
kubectl get configmap -n kube-system coredns -o yaml
```

**Пример:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

**Что означают плагины:**

| Плагин | Что делает |
|:---|:---|
| `kubernetes` | Резолвит `<service>.<namespace>.svc.cluster.local` |
| `forward` | Пересылает внешние запросы (на `/etc/resolv.conf` хоста) |
| `cache` | Кэширует ответы (30 секунд) |
| `loadbalance` | Round-robin для headless (несколько A-записей) |
| `prometheus` | Метрики для Prometheus |
| `health` | Health check |

### 🎯 Кастомные DNS

**Добавить кастомный DNS-сервер:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        # ... стандартные плагины
        forward example.com 10.0.0.53    # кастомный DNS для example.com
        forward . /etc/resolv.conf        # всё остальное — стандартный
    }
```

**Когда использовать:** для внутренних доменов компании.

### 🔬 Практика: DNS

```bash
# 1. Проверить DNS из Pod'а
kubectl run -it --rm debug --image=nicolaka/netshoot --restart=Never -- bash
# Внутри:
nslookup kubernetes.default
# Server:    10.96.0.10
# Address:   10.96.0.10:53

nslookup myapp
nslookup myapp.default.svc.cluster.local

# 2. Посмотреть CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# 3. ConfigMap
kubectl get configmap -n kube-system coredns -o yaml

# 4. Проверить Service
kubectl get service -n kube-system kube-dns
```

### 💡 Практика: как правильно работать с DNS

**✅ ОБЯЗАТЕЛЬНО:**

1. **Использовать DNS-имена** в конфигурации приложений:
   ```
   DATABASE_URL=postgres://postgres:5432/mydb
   ```
2. **Проверять DNS** через `nslookup` при проблемах.

**👍 СТОИТ:**

4. **Headless Service для StatefulSet.**
5. **Кастомный DNS для внутренних доменов.**

**❌ НЕ ДЕЛАЙ:**

6. **Не используй IP'ы Service'ов.** Используй имена.
7. **Не хардкодь `.svc.cluster.local`** — обычно достаточно короткого имени.

### Где мы сейчас

Мы разобрали CoreDNS. Теперь — **Network Policies** — изоляция трафика.

---

## 10.8 Network Policies: изоляция трафика

### 🔌 Проблема: все Pod'ы видят всех

По умолчанию в Kubernetes **все Pod'ы видят всех**. Pod в namespace `dev` может обратиться к Pod в namespace `prod`. Pod в frontend может обратиться к базе данных напрямую.

**Это небезопасно.** Если злоумышленник взломает frontend, он получит доступ ко всей внутренней сети.

**Решение:** Network Policies.

### 📦 Что такое Network Policy

**Network Policy** — это **файрвол для Pod'ов**. Определяет, какой трафик разрешён к Pod'ам и от Pod'ов.

**Ключевое:**

- **Opt-in.** По умолчанию — всё разрешено. Network Policy **ограничивает**.
- **Whitelist.** Ты указываешь, что **разрешено**. Всё остальное — запрещено.
- **Требует CNI-поддержки.** Flannel не поддерживает. Calico, Cilium — да.

### 🎯 Структура Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: myapp-policy
  namespace: default
spec:
  podSelector:              # к каким Pod'ам применяется
    matchLabels:
      app: myapp
  policyTypes:              # какие типы трафика ограничиваем
    - Ingress
    - Egress
  ingress:                  # входящий трафик
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:                   # исходящий трафик
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

**Что означает:**

- Применяется к Pod'ам с `app=myapp`.
- **Ingress:** разрешён трафик **только** от Pod'ов с `app=frontend` на порт 8080.
- **Egress:** разрешён трафик **только** к Pod'ам с `app=database` на порт 5432.

### 🎯 Типы selector'ов

**podSelector:** Pod'ы в **том же namespace**.

```yaml
from:
  - podSelector:
      matchLabels:
        app: frontend
```

**namespaceSelector:** все Pod'ы в namespace.

```yaml
from:
  - namespaceSelector:
      matchLabels:
        name: frontend
```

**Комбинация:**

```yaml
from:
  - namespaceSelector:
      matchLabels:
        name: frontend
    podSelector:
      matchLabels:
        app: web
```

**Что означает:** Pod'ы с `app=web` в namespace с `name=frontend`.

**ipBlock:** IP-диапазон (для внешних источников).

```yaml
from:
  - ipBlock:
      cidr: 10.0.0.0/8
      except:
        - 10.0.1.0/24
```

### 🎯 Default deny

**Самый важный паттерн:** запретить весь трафик по умолчанию.

**Deny all ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}           # ко всем Pod'ам в namespace
  policyTypes:
    - Ingress
```

**Что произойдёт:** весь входящий трафик к Pod'ам в `default` **запрещён**. Пока не добавишь разрешающие policies.

**Deny all egress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

**Deny all (ingress + egress):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

**Важно:** после `default-deny` DNS **тоже** заблокируется. Нужно явно разрешить DNS:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### 🎯 Практический паттерн: микросервисы

**Сценарий:**

- `frontend` → `api`
- `api` → `database`, `cache`
- `frontend` не должен видеть `database`.

**Policies:**

```yaml
# 1. Default deny для namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# 2. Разрешить DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53

---
# 3. Frontend может обращаться к API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - protocol: TCP
          port: 8080

---
# 4. API принимает трафик от frontend и обращается к DB, cache
apiVersion: networking.k8s.io/v1kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: cache
      ports:
        - protocol: TCP
          port: 6379

---
# 5. Database принимает трафик только от API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - protocol: TCP
          port: 5432
```

**Результат:**

- `frontend` **не может** обратиться к `database` напрямую.
- `api` **может** обращаться к `database` и `cache`.
- `database` **принимает** трафик только от `api`.
- Всё остальное — запрещено.

### 🎯 CNI с поддержкой Network Policies

| CNI | Поддержка |
|:---|:---|
| **Calico** | ✅ Полная |
| **Cilium** | ✅ Полная + L7 |
| **Weave** | ✅ Полная |
| **kube-router** | ✅ Полная |
| **Flannel** | ❌ (нужен Calico для policies) |

**Проверить:**

```bash
# Если NetworkPolicy не работает — проверь CNI
kubectl get pods -n kube-system | grep -E "calico|cilium|weave|kube-router"
```

### 🎯 Network Policy не удаляются

**Важно:** Network Policy **не удаляются** при удалении Pod'ов. Они привязаны к namespace и labels.

**Проверить:**

```bash
kubectl get networkpolicies -A
# NAMESPACE    NAME              POD-SELECTOR
# production   default-deny      <none>
# production   api-policy        app=api
```

**Удалить:**

```bash
kubectl delete networkpolicy <name> -n <namespace>
```

### 🔬 Практика: Network Policy

```bash
# 1. Проверить, что CNI поддерживает NetworkPolicy
kubectl get pods -n kube-system | grep -E "calico|cilium"

# 2. Создать default-deny
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Ingress
EOF

# 3. Попробовать обратиться к Service
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
# Внутри:
wget -qO- http://kubernetes.default.svc
# wget: download timed out  ← заблокировано!

# 4. Разрешить конкретный трафик
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 80
EOF

# 5. Теперь Pod'ы в default могут обращаться к nginx
```

### 💡 Практика: как правильно работать с Network Policies

**✅ ОБЯЗАТЕЛЬНО:**

1. **Default deny для каждого namespace.**
2. **Allow DNS явно** после default deny.
3. **Разрешать только нужный трафик** (whitelist).

**👍 СТОИТ:**

4. **Разные policies для разных окружений.**
5. **Проверять policies через Network Policy Editor** (editor.cilium.io).

**❌ НЕ ДЕЛАЙ:**

6. **Не используй Network Policies без CNI-поддержки.** Не будет работать.
7. **Не забывай про DNS.** После default deny DNS блокируется.
8. **Не разрешай всё сразу.** Постепенно, по мере необходимости.

### Где мы сейчас

Мы разобрали Network Policies. Теперь — **диагностика сетевых проблем**.

---

## 10.9 Диагностика сетевых проблем

### 🔌 Проблема: не работает, но что именно

Сетевые проблемы — самые сложные. Пакет проходит через много уровней. Где застрял?

Разберём алгоритм диагностики.

### 🔍 Алгоритм диагностики

**Шаг 1: Проверить Pod'ы**

```bash
kubectl get pods -o wide
# Убедиться, что Pod'ы Running, не Pending
```

**Шаг 2: Проверить Service**

```bash
kubectl get service <name>
# Есть ClusterIP?
# Правильный порт?

kubectl describe service <name>
# Selector?
# TargetPort?
# Endpoints?
```

**Шаг 3: Проверить Endpoints**

```bash
kubectl get endpoints <name>
# Есть IP'ы Pod'ов?
# Если <none> — проблема с selector
```

**Шаг 4: Проверить, что Pod слушает порт**

```bash
kubectl exec -it <pod> -- ss -tlnp
# Приложение слушает 0.0.0.0:8080?
# Или только 127.0.0.1:8080?
```

**Шаг 5: Проверить DNS**

```bash
kubectl run -it --rm debug --image=nicolaka/netshoot --restart=Never -- bash
# Внутри:
nslookup <service>
# Резолвится?
```

**Шаг 6: Проверить доступ**

```bash
# Из другого Pod'а
kubectl exec -it <other-pod> -- curl http://<service>:80
# Или
kubectl exec -it <other-pod> -- wget -qO- http://<service>
```

**Шаг 7: Проверить Network Policies**

```bash
kubectl get networkpolicies -A
# Есть ли policies, блокирующие трафик?
```

**Шаг 8: tcpdump**

```bash
# В Pod'е (если есть tcpdump)
kubectl exec -it <pod> -- tcpdump -i eth0 -n port 8080

# На ноде (нужен доступ)
tcpdump -i cni0 -n port 8080
```

### 🎯 Типичные проблемы

**1. Pod не доступен по Service.**

- **Endpoints пусты:** selector не совпадает.
- **targetPort неправильный:** Service → порт, Pod не слушает.
- **Pod не ready:** readiness probe не проходит → Pod не в Endpoints.

**Проверить:**

```bash
kubectl get endpoints <service>
kubectl describe service <service>
kubectl get pods --show-labels
```

**2. DNS не резолвится.**

- **CoreDNS не работает:** `kubectl get pods -n kube-system -l k8s-app=kube-dns`.
- **Неправильный namespace:** Service в другом namespace.
- **ndots проблема:** `myapp.default` вместо `myapp`.

**Проверить:**

```bash
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- nslookup <service>
kubectl logs -n kube-system -l k8s-app=kube-dns
```

**3. Ingress не работает.**

- **Ingress Controller не установлен.**
- **ingressClassName неправильный.**
- **Service не существует.**
- **DNS не указывает на Ingress IP.**
- **TLS secret не существует.**

**Проверить:**

```bash
kubectl get ingress
kubectl describe ingress <name>
kubectl logs -n ingress-nginx <controller-pod>
```

**4. Таймауты.**

- **Network Policy блокирует.**
- **Файрвол на ноде.**
- **MTU проблема** (особенно с overlay).

**Проверить:**

```bash
kubectl get networkpolicies -A
kubectl exec -it <pod> -- ping <other-pod-ip>
kubectl exec -it <pod> -- traceroute <other-pod-ip>
```

**5. Connection refused.**

- **Приложение не слушает порт.**
- **Слушает localhost.**
- **targetPort неправильный.**

**Проверить:**

```bash
kubectl exec -it <pod> -- ss -tlnp
kubectl exec -it <pod> -- netstat -tlnp
```

### 🎯 Debug-Pod

**netshoot** — образ с сетевыми утилитами:

```bash
kubectl run -it --rm debug \
  --image=nicolaka/netshoot \
  --restart=Never \
  -- bash
```

**Что внутри:**

- `nslookup`, `dig`, `host` — DNS.
- `ping`, `traceroute`, `mtr` — сеть.
- `tcpdump`, `tshark` — захват пакетов.
- `curl`, `wget` — HTTP.
- `ss`, `netstat` — сокеты.
- `iperf3` — throughput.
- `nmap` — сканирование.

**Debug-Pod с нужным namespace:**

```bash
kubectl run -it --rm debug \
  --image=nicolaka/netshoot \
  --restart=Never \
  --namespace=production \
  -- bash
```

**Debug в Pod'е приложения (ephemeral container):**

```bash
kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>
```

### 🎯 kube-proxy диагностика

**Проверить, что kube-proxy работает:**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy
# Все Running?

kubectl logs -n kube-system <kube-proxy-pod>
# Ошибки?
```

**Проверить iptables:**

```bash
# На ноде
iptables -t nat -L KUBE-SERVICES -n | grep <service-ip>
# Есть правило?
```

### 🔬 Практика: полный сценарий

**Проблема:** Service не доступен.

```bash
# 1. Pod'ы работают
kubectl get pods -o wide
# NAME    READY   STATUS    IP           NODE
# myapp   1/1     Running   10.244.1.5   node-1

# 2. Service существует
kubectl get service myapp
# NAME    TYPE        CLUSTER-IP      PORT(S)
# myapp   ClusterIP   10.96.123.45    80/TCP

# 3. Endpoints пусты!
kubectl get endpoints myapp
# NAME    ENDPOINTS
# myapp   <none>       ← проблема!

# 4. Проверить selector
kubectl get service myapp -o jsonpath='{.spec.selector}'
# {"app":"myapp"}

# 5. Проверить labels Pod'ов
kubectl get pods --show-labels
# NAME    LABELS
# myapp   app=my-app    ← не совпадает!

# 6. Исправить
kubectl label pod myapp app=myapp --overwrite
kubectl label deployment myapp app=myapp --overwrite

# 7. Проверить Endpoints
kubectl get endpoints myapp
# NAME    ENDPOINTS
# myapp   10.244.1.5:8080   ← появились!
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Всегда начинай с `kubectl get endpoints`.** Пустые = selector проблема.
2. **Проверяй `kubectl describe service`.**
3. **Используй debug-Pod с netshoot.**
4. **tcpdump для глубокой диагностики.**

**👍 СТОИТ:**

5. **Проверять Network Policies.**
6. **Смотреть логи CoreDNS.**
7. **Смотреть логи Ingress Controller.**

**❌ НЕ ДЕЛАЙ:**

8. **Не используй IP'ы в диагностике.** Используй DNS.
9. **Не игнорируй `Pending` Pod'ы.** Они могут быть причиной.
10. **Не забывай про readiness probe.** Pod не ready → не в Endpoints.

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **CNI** | Container Network Interface — стандарт настройки сети контейнеров. |
| **Pod CIDR** | Подсеть Pod'ов на ноде. |
| **Veth pair** | Пара виртуальных ethernet-интерфейсов. |
| **Overlay** | Инкапсуляция (VXLAN, IPIP) для связи между нодами. |
| **BGP** | Протокол маршрутизации. Используется Calico. |
| **eBPF** | Расширенный BPF. Используется Cilium. |
| **Service** | Абстракция для стабильного доступа к Pod'ам. |
| **ClusterIP** | Виртуальный IP Service'а. |
| **Endpoints** | Список IP:port Pod'ов для Service'а. |
| **kube-proxy** | Сетевой агент, реализующий Service'ы. |
| **iptables** | Режим kube-proxy через правила iptables. |
| **IPVS** | IP Virtual Server — режим kube-proxy. |
| **NodePort** | Тип Service: порт на каждой ноде. |
| **LoadBalancer** | Тип Service: внешний LB. |
| **ExternalName** | Тип Service: CNAME на внешний сервис. |
| **Headless Service** | Service без ClusterIP. |
| **externalTrafficPolicy** | Cluster или Local — как обрабатывать source IP. |
| **Ingress** | HTTP/HTTPS-роутер для K8s. |
| **Ingress Controller** | Приложение, реализующее Ingress. |
| **pathType** | Exact, Prefix, ImplementationSpecific. |
| **CoreDNS** | DNS-сервер в кластере. |
| **ndots** | Параметр resolv.conf. |
| **Network Policy** | Файрвол для Pod'ов. |
| **Default deny** | Запретить весь трафик по умолчанию. |
| **CNI с NetworkPolicy** | Calico, Cilium, Weave, kube-router. |

---

## Что мы узнали?

- **Сетевая модель K8s:** каждый Pod имеет IP, Pod'ы общаются напрямую. Реализуется через **CNI**.
- **CNI:** Flannel (простой, overlay), Calico (routing, network policies), Cilium (eBPF).
- **Service** — стабильный IP и DNS для Pod'ов. Находит Pod'ы по selector.
- **kube-proxy** настраивает iptables/IPVS правила для маршрутизации на Pod'ы.
- **Типы Service:** ClusterIP (внутренний), NodePort (порт на ноде), LoadBalancer (внешний LB), ExternalName (CNAME).
- **Ingress** — HTTP/HTTPS-роутер. Один IP для многих сервисов. Требует Ingress Controller.
- **CoreDNS** — DNS-сервер. Резолвит `<service>.<namespace>.svc.cluster.local`.
- **Network Policies** — файрвол для Pod'ов. Opt-in, whitelist. Требует CNI-поддержки.
- **Диагностика:** `kubectl get endpoints`, `kubectl describe`, debug-Pod с netshoot, tcpdump.

---

## Типичные ошибки

- ❌ **Не проверять Endpoints.** Пустые = selector проблема.
- ❌ **Использовать IP'ы Pod'ов в конфигурации.** IP меняется.
- ❌ **Забывать про targetPort.** Service → 80, Pod → 8080.
- ❌ **Приложение слушает localhost.** Не доступно извне.
- ❌ **Использовать Flannel с Network Policies.** Не поддерживает.
- ❌ **Забывать про DNS после default-deny.** CoreDNS блокируется.
- ❌ **Использовать NodePort в production.** Ingress лучше.
- ❌ **Использовать LoadBalancer для каждого сервиса.** Дорого. Ingress.
- ❌ **Не использовать `pathType` явно.**
- ❌ **Не проверять CNI при Network Policy problems.**
- ❌ **Игнорировать kube-proxy logs при проблемах.**
- ❌ **Не использовать debug-Pod с netshoot.** Лучший инструмент.

---

## Для быстрого повторения

- **CNI:** Flannel (overlay), Calico (routing + policies), Cilium (eBPF).
- **Service:** стабильный IP и DNS. Находит Pod'ы по selector.
- **Endpoints:** список IP:port. Пусто = selector проблема.
- **kube-proxy:** iptables или IPVS. Настраивает маршрутизацию.
- **Типы Service:** ClusterIP, NodePort, LoadBalancer, ExternalName.
- **externalTrafficPolicy:** Cluster (source IP = нода), Local (source IP = клиент).
- **Ingress:** HTTP/HTTPS-роутер. Требует Ingress Controller.
- **pathType:** Exact, Prefix, ImplementationSpecific.
- **CoreDNS:** `10.96.0.10`. `<service>.<namespace>.svc.cluster.local`.
- **Network Policy:** default-deny + allow. Требует CNI-поддержки.
- **Диагностика:** `kubectl get endpoints`, debug-Pod (netshoot), tcpdump.

---

## Вопросы для самопроверки

1. Что такое CNI? Какие плагины знаешь? В чём разница?
2. Как Pod получает IP? Что такое veth pair?
3. Что такое Service? Как он находит Pod'ы?
4. Что такое Endpoints? Что означает пустой Endpoints?
5. Как работает kube-proxy? Чем iptables отличается от IPVS?
6. Четыре типа Service — когда использовать каждый?
7. Что такое `externalTrafficPolicy`? Чем Cluster отличается от Local?
8. Что такое Ingress? Зачем нужен Ingress Controller?
9. Чем Ingress отличается от Service LoadBalancer?
10. Что такое CoreDNS? Как Pod'ы находят друг друга по именам?
11. Что такое Network Policy? Зачем default-deny?
12. Как диагностировать проблему «Service не доступен»?
13. Что такое debug-Pod? Какие инструменты в netshoot?
14. Ты добавил Network Policy, и приложение перестало резолвить DNS. Что не так?
15. Клиент обращается к Service, но получает `Connection refused`. Endpoints есть. В чём может быть проблема?

---

## Ответы

**1. CNI**

Container Network Interface — стандарт настройки сети. Плагины: Flannel (простой, overlay), Calico (routing + Network Policies), Cilium (eBPF, L7 policies, service mesh). Разница в производительности, поддержке policies, сложности.

**2. Pod и IP**

kubelet вызывает CNI-плагин → создаётся network namespace → veth pair → один конец в Pod'е (`eth0`), другой на ноде. CNI выделяет IP из podCIDR.

**3. Service**

Абстракция для стабильного доступа. Находит Pod'ы по **selector** (labels). Создаёт Endpoints — список IP:port подходящих Pod'ов.

**4. Endpoints**

Список IP:port Pod'ов для Service'а. **Пустой Endpoints** = selector не совпадает с labels Pod'ов → трафик некуда идти.

**5. kube-proxy**

Сетевой агент на каждой ноде. Настраивает iptables/IPVS правила: ClusterIP → один из Pod'ов. **iptables:** O(n), медленнее на больших кластерах. **IPVS:** O(1), лучше масштабируется.

**6. Типы Service**

- **ClusterIP** — внутренние.
- **NodePort** — dev, on-prem.
- **LoadBalancer** — production в облаке (TCP/UDP).
- **ExternalName** — CNAME на внешний сервис.

**7. externalTrafficPolicy**

- **Cluster** (default) — трафик может идти на любую ноду, source IP теряется.
- **Local** — только на Pod'ы той же ноды, source IP сохраняется.

**8. Ingress**

HTTP/HTTPS-роутер. Ingress — ресурс (манифест). Ingress Controller — приложение (Nginx, Traefik). Без Controller Ingress — просто манифест.

**9. Ingress vs LoadBalancer**

Ingress: HTTP/HTTPS, один IP, path/host routing, TLS. LoadBalancer: TCP/UDP, отдельный IP для каждого, дорого.

**10. CoreDNS**

DNS-сервер в кластере (`10.96.0.10`). Резолвит `<service>.<namespace>.svc.cluster.local`. Каждый Pod имеет `/etc/resolv.conf` с `nameserver 10.96.0.10`.

**11. Network Policy**

Файрвол для Pod'ов. Default-deny — запретить весь трафик, потом разрешать нужный. Whitelist. Требует CNI-поддержки (Calico, Cilium).

**12. Диагностика**

1. `kubectl get pods -o wide` — Pod'ы Running?
2. `kubectl get service` — ClusterIP, порт?
3. `kubectl get endpoints` — есть IP'ы?
4. `kubectl exec -- ss -tlnp` — приложение слушает?
5. `nslookup` — DNS работает?
6. `curl` из другого Pod'а — доступ есть?
7. `kubectl get networkpolicies` — не блокирует?
8. tcpdump для глубокого анализа.

**13. Debug-Pod**

Pod с инструментами для отладки. `netshoot`: nslookup, dig, ping, traceroute, tcpdump, curl, ss, netstat, iperf3, nmap.

**14. Network Policy и DNS**

После default-deny **DNS тоже блокируется**. Нужно явно разрешить трафик к CoreDNS (namespace `kube-system`, `k8s-app: kube-dns`, порт 53 UDP/TCP).

**15. Connection refused при наличии Endpoints**

- **targetPort неправильный** — Service → 80, Pod слушает 8080.
- **Приложение слушает localhost** — не 0.0.0.0.
- **Приложение не запущено** — но Pod Running.
- **Приложение падает** — но readiness probe проходит.

---

## Куда идти дальше?

Мы разобрали сетевое взаимодействие в Kubernetes. Теперь ты знаешь:

- Как Pod'ы получают IP и общаются.
- Как работают Service'ы и kube-proxy.
- Как настроить Ingress для HTTP-трафика.
- Как работает DNS в кластере.
- Как изолировать трафик через Network Policies.
- Как диагностировать сетевые проблемы.

Но мы пока не разобрали:

- **Как хранить данные.** Volumes, PersistentVolumes, StatefulSets.
- **Как передавать конфигурацию.** ConfigMaps, Secrets.

Следующие главы — про это.

**Глава 11: Kubernetes — хранилища и Stateful-приложения.** Погнали. 🚀