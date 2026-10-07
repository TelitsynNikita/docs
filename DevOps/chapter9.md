# 🎯 Глава 9: Kubernetes — планирование и ресурсы

**Что вы узнаете:**
- Как kube-scheduler решает, на какую ноду поставить Pod.
- Что такое requests и limits, чем они отличаются.
- Что такое QoS-классы: Guaranteed, Burstable, BestEffort.
- Как работает переподписка (overcommit) и почему это риск.
- Как ограничить Pod'ы по нодам через nodeSelector, affinity и anti-affinity.
- Что такое taints и tolerations и зачем они нужны.
- Как работает priority и preemption.
- Как диагностировать Pod в статусе Pending из-за нехватки ресурсов.

**После прочтения вы сможете:**
- Правильно выбирать requests и limits для приложений.
- Управлять размещением Pod'ов через affinity/anti-affinity.
- Использовать taints и tolerations для выделенных нод.
- Понимать, почему Pod в Pending и как это исправить.
- Настраивать QoS для критичных сервисов.
- Использовать topology spread для равномерного распределения.

---

## Содержание

- [9.0 Пролог: Pod в Pending](#90-пролог-pod-в-pending)
- [9.1 Как работает kube-scheduler](#91-как-работает-kube-scheduler)
- [9.2 Requests и limits: сколько нужно и сколько можно](#92-requests-и-limits-сколько-нужно-и-сколько-можно)
- [9.3 QoS-классы: Guaranteed, Burstable, BestEffort](#93-qos-классы-guaranteed-burstable-besteffort)
- [9.4 NodeSelector: простой способ выбрать ноду](#94-nodeselector-простой-способ-выбрать-ноду)
- [9.5 Node Affinity и Anti-Affinity: гибкое размещение](#95-node-affinity-и-anti-affinity-гибкое-размещение)
- [9.6 Pod Affinity и Anti-Affinity: размещение относительно других Pod'ов](#96-pod-affinity-и-anti-affinity-размещение-относительно-других-podов)
- [9.7 Taints и Tolerations: выделенные ноды](#97-taints-и-tolerations-выделенные-ноды)
- [9.8 Priority и Preemption: кто важнее](#98-priority-и-preemption-кто-важнее)
- [9.9 Topology Spread: равномерное распределение](#99-topology-spread-равномерное-распределение)
- [9.10 Диагностика: Pod в Pending из-за ресурсов](#910-диагностика-pod-в-pending-из-за-ресурсов)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 9.0 Пролог: Pod в Pending

Ты задеплоил новую версию приложения. Kubernetes создал Pod'ы. Но они висят в статусе `Pending`:

```bash
kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# myapp-7d9f8c6b4d-abc12   0/1     Pending   0          5m
# myapp-7d9f8c6b4d-def34   0/1     Pending   0          5m
# myapp-7d9f8c6b4d-ghi56   0/1     Pending   0          5m
```

Пять минут. Десять. Pod'ы не запускаются.

Ты смотришь `kubectl describe pod myapp-7d9f8c6b4d-abc12`:

```
Events:
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  5m     default-scheduler  0/5 nodes are available:
    3 Insufficient cpu,
    2 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }
```

Что это значит?

- **3 ноды не подходят** — недостаточно CPU для Pod'а.
- **2 ноды не подходят** — на них taint, который Pod не переносит.

**Scheduler не может найти подходящую ноду.** Pod остаётся в Pending.

Это — типичная проблема. И чтобы её решить, нужно понимать **как работает scheduler** и **как он выбирает ноду**.

В этой главе мы разберём:

- Алгоритм работы scheduler.
- Что такое requests и limits.
- Как управлять размещением Pod'ов.
- Как диагностировать проблемы с планированием.

Это — основа для production. Без понимания этих механизмов ты будешь гадать, почему Pod'ы не запускаются.

---

## 9.1 Как работает kube-scheduler

### 🔌 Проблема: кто решает, где запускать Pod

Ты создал Pod. Kubernetes должен решить, на какую ноду его поставить. В кластере 10 нод. Как выбрать?

**kube-scheduler** — это компонент control plane, который **назначает Pod'ы на ноды**.

Он работает так:

1. **Watch API** — подписан на новые Pod'ы без назначенной ноды.
2. Для каждого Pod'а запускает **двухфазный алгоритм**:
   - **Filtering (Predicates)** — какие ноды **подходят**?
   - **Scoring (Priorities)** — какая нода **лучшая**?
3. Выбирает лучшую ноду и сообщает apiserver.
4. apiserver обновляет Pod (`nodeName: "node-3"`).

### 📊 Двухфазный алгоритм

```
┌────────────────────────────────────────────────────────────┐
│                   ВСЕ НОДЫ В КЛАСТЕРЕ                       │
│                                                            │
│  node-1  node-2  node-3  node-4  node-5  node-6  ...      │
└────────────────────────────┬───────────────────────────────┘
                             │
                             │ ФАЗА 1: FILTERING
                             │ (какие ноды подходят?)
                             ▼
┌────────────────────────────────────────────────────────────┐
│              ПОДХОДЯЩИЕ НОДЫ                                │
│                                                            │
│  node-2  node-4  node-6                                    │
└────────────────────────────┬───────────────────────────────┘
                             │
                             │ ФАЗА 2: SCORING
                             │ (какая лучшая?)
                             ▼
┌────────────────────────────────────────────────────────────┐
│              РАНЖИРОВАНИЕ                                   │
│                                                            │
│  node-4: 85  node-6: 72  node-2: 60                        │
│                                                            │
│  ЛУЧШАЯ: node-4                                            │
└────────────────────────────┬───────────────────────────────┘
                             │
                             ▼
                       Pod → node-4
```

### 🎯 Фаза 1: Filtering (Predicates)

**Цель:** отсеять ноды, которые **не подходят** для Pod'а.

**Основные фильтры:**

| Фильтр | Что проверяет |
|:---|:---|
| **NodeResourcesFit** | Достаточно ли CPU/memory на ноде |
| **NodeName** | Соответствует ли нода `spec.nodeName` |
| **NodeSelector** | Есть ли labels, указанные в `nodeSelector` |
| **NodeAffinity** | Соответствует ли нода `nodeAffinity` |
| **TaintToleration** | Толерантен ли Pod к taints ноды |
| **PodAffinity** | Есть ли на ноде Pod'ы, соответствующие `podAffinity` |
| **PodAntiAffinity** | Нет ли на ноде Pod'ов, соответствующих `podAntiAffinity` |
| **VolumeBinding** | Можно ли смонтировать volumes Pod'а |
| **NodeUnschedulable** | Не помечена ли нода как `Unschedulable` |

**Если нода не прошла хоть один фильтр — она отсеивается.**

**Пример:** Pod требует 4 CPU. На node-1 только 2 свободных CPU → NodeResourcesFit отсеивает node-1.

### 🎯 Фаза 2: Scoring (Priorities)

**Цель:** ранжировать подходящие ноды и выбрать лучшую.

**Основные scorer'ы:**

| Scorer | Что оценивает | Вес по умолчанию |
|:---|:---|:---|
| **LeastAllocated** | Меньше загружена — выше score | 1 |
| **BalancedAllocation** | Балансирует CPU и memory | 1 |
| **ImageLocality** | Если образ уже на ноде — выше | 1 |
| **NodeAffinity** | Выше, если нода соответствует preferred affinity | 2 |
| **PodTopologySpread** | Равномерность распределения | 2 |
| **InterPodAffinity** | Соответствует preferred pod affinity | 2 |
| **TaintToleration** | Меньше taints — выше | 3 |

**Как считается:**

```
Итоговый score = Σ (scorer_score × вес)
```

**Пример:**

```
node-2: LeastAllocated=60, BalancedAllocation=80, ImageLocality=0 → 140
node-4: LeastAllocated=85, BalancedAllocation=70, ImageLocality=100 → 255
node-6: LeastAllocated=90, BalancedAllocation=85, ImageLocality=0 → 175

Лучшая: node-4 (255)
```

**Ключевые scorer'ы:**

**LeastAllocated:** предпочитает ноды с большим количеством свободных ресурсов. Распределяет нагрузку.

**BalancedAllocation:** предпочитает ноды, где CPU и memory используются пропорционально. Избегает ситуации, когда одна нода загружена по CPU, но свободна по памяти.

**ImageLocality:** если образ уже скачан на ноду — плюс. Быстрее старт Pod'а.

### 🎯 Что происходит после выбора

1. Scheduler выбирает ноду.
2. Обновляет Pod в etcd: `spec.nodeName = "node-4"`.
3. kubelet на node-4 видит событие через Watch API.
4. kubelet проверяет Pod, скачивает образ, запускает контейнер.
5. Обновляет статус Pod: `Running`.

### 🎯 Если ни одна нода не подходит

**Pod остаётся в Pending.** Scheduler периодически пытается снова — при изменениях в кластере:

- Добавили ноду.
- Удалили другой Pod (освободились ресурсы).
- Изменили Pod (например, уменьшили requests).

**Events в `kubectl describe pod` покажут, почему:**

```
Events:
  Type     Reason            Message
  ----     ------            -------
  Warning  FailedScheduling  0/5 nodes are available: 3 Insufficient cpu, 2 node(s) had untolerated taint.
```

**Это — главный источник диагностики.** Читай сообщение, понимай причину.

### 🔬 Практика: наблюдаем работу scheduler

```bash
# 1. Создать Pod с большими requests
kubectl run test --image=nginx --requests=cpu=100

# 2. Проверить статус
kubectl get pods
# NAME   READY   STATUS    RESTARTS   AGE
# test   0/1     Pending   0          10s

# 3. Описать Pod
kubectl describe pod test
# Events:
#   Warning  FailedScheduling  0/3 nodes are available:
#     3 Insufficient cpu.

# 4. Посмотреть scheduler в логах
kubectl logs -n kube-system -l component=kube-scheduler
# Покажет попытки назначить Pod
```

### 💡 Практика: как понять scheduler

**✅ ОБЯЗАТЕЛЬНО:**

1. **Всегда смотри `kubectl describe pod` при Pending.** Events объяснят причину.
2. **Понимай, что filtering — жёсткое условие.** Если нода не прошла фильтр — она отсеивается.
3. **Понимай, что scoring — мягкое условие.** Нода подходит, но может быть не оптимальной.

**👍 СТОИТ:**

4. **Знать основные фильтры:** NodeResourcesFit, NodeAffinity, TaintToleration.
5. **Знать основные scorer'ы:** LeastAllocated, BalancedAllocation, ImageLocality.

**❌ НЕ ДЕЛАЙ:**

6. **Не думай, что scheduler магический.** Он работает по алгоритму, и его можно понять.
7. **Не игнорируй `FailedScheduling`.** Это не «подожди ещё», это «исправь причину».

### Где мы сейчас

Мы разобрали, как работает scheduler. Теперь — **requests и limits** — самый важный параметр для планирования.

---

## 9.2 Requests и limits: сколько нужно и сколько можно

### 🔌 Проблема: сколько ресурсов выделять Pod'у

Ты пишешь манифест. Сколько CPU и памяти указать? Слишком мало — Pod не запустится. Слишком много — ресурсы простаивают.

**Решение:** requests и limits.

### 📊 Requests и limits

**Requests** — сколько ресурсов **гарантировано** Pod'у. Scheduler использует это значение при планировании.

**Limits** — максимальное количество ресурсов, которое Pod может использовать.

```yaml
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      resources:
        requests:
          cpu: "500m"          # 0.5 ядра
          memory: "256Mi"      # 256 мегабайт
        limits:
          cpu: "1"             # 1 ядро (максимум)
          memory: "512Mi"      # 512 мегабайт (максимум)
```

**Что это значит:**

- Pod **гарантированно** получит 0.5 CPU и 256 MB RAM.
- Pod **может** использовать до 1 CPU и 512 MB RAM.
- Scheduler учитывает **requests** при планировании: на ноде должно быть **500m CPU и 256Mi RAM** свободно.

### 🎯 CPU: requests vs limits

**Единицы измерения CPU:**

| Значение | Что означает |
|:---|:---|
| `1` | 1 ядро (1000 millicores) |
| `500m` | 0.5 ядра (500 millicores) |
| `100m` | 0.1 ядра (100 millicores) |

**Как работает:**

- **Requests** — scheduler гарантирует. Если нода имеет 4 CPU, на ней могут разместиться 8 Pod'ов с `requests.cpu: 500m`.
- **Limits** — жёсткое ограничение. Если Pod превысит — **throttling**.

**Throttling:** если Pod использует больше `limits.cpu`, ядро CFS ограничивает его. Процесс не убивается, но замедляется. Латентность растёт.

**Важно:** CPU — **сжимаемый ресурс**. Если Pod не использует свой requests, другие Pod'ы могут использовать эти ресурсы.

**Пример:**

```
Нода: 4 CPU

Pod A: requests=500m, limits=1
Pod B: requests=500m, limits=1
Pod C: requests=500m, limits=1

Scheduler видит: 3 × 500m = 1500m запрошено из 4000m → может поставить ещё

Если Pod A захочет 1 CPU — scheduler уже выделил 500m. Остальные 500m берутся из свободного пула.
Если Pod A и Pod B оба захотят по 1 CPU → они поделят доступные 4 CPU (CFS).
```

### 🎯 Memory: requests vs limits

**Единицы измерения memory:**

| Значение | Что означает |
|:---|:---|
| `128Mi` | 128 mebibytes (двоичные) |
| `128M` | 128 megabytes (десятичные) |
| `1Gi` | 1 gibibyte |
| `1G` | 1 gigabyte |

**Разница:** `Mi` = 1024×1024 байт, `M` = 1000×1000 байт. Для memory в K8s обычно используют `Mi`/`Gi`.

**Как работает:**

- **Requests** — scheduler гарантирует.
- **Limits** — если Pod превысит, **OOM Killer** убьёт процесс.

**Memory — несжимаемый ресурс.** В отличие от CPU, память нельзя «сжать». Если Pod превысил лимит — его убьют.

**Пример:**

```
Pod: requests.memory=256Mi, limits.memory=512Mi

Pod использует 400Mi → OK
Pod использует 500Mi → OK (близко к лимиту)
Pod использует 512Mi → OOM Killer убивает процесс
```

**Что произойдёт при OOM:**

1. Контейнер убивается.
2. Kubernetes перезапускает его (если `restartPolicy: Always`).
3. Через 3 перезапуска подряд — `CrashLoopBackOff`.

### 🎯 Requests без limits

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  # без limits
```

**Что это значит:**

- Гарантировано 500m CPU и 256Mi RAM.
- **Нет上限** для использования. Pod может использовать сколько угодно.
- CPU: если свободно — Pod использует больше.
- Memory: если свободно — Pod использует больше. Но если нода начнёт сжиматься — могут убить.

**Когда использовать:**

- **Batch-задачи** — могут ждать, могут использовать больше при доступности.
- **Dev/staging** — где важна гибкость.

**Не для production critical.**

### 🎯 Limits без requests

```yaml
resources:
  limits:
    cpu: "1"
    memory: "512Mi"
  # без requests
```

**Что это значит:**

- Requests **автоматически = limits** (Kubernetes подставит).
- Pod гарантированно получит 1 CPU и 512Mi RAM.

**Когда использовать:** почти никогда. Если нужны requests = limits — указывай явно, это понятнее.

### 🎯 Как выбрать requests и limits

**Правило:**

- **Requests = типичное потребление.** Что Pod использует в обычной работе.
- **Limits = 1.5-2x от типичного.** Запас на пики.

**Пример для Go-сервиса:**

- Обычно использует 200 MiB.
- Пики до 400 MiB.

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

**Как узнать реальное потребление:**

1. Запустить приложение с **временными большими limits**.
2. Мониторить метрики (`kubectl top pod`, Prometheus).
3. Через неделю посмотреть p50/p95/p99.
4. Установить requests = p50, limits = p95 или p99.

### 🎯 Overcommit: переподписка

**Overcommit** — ситуация, когда сумма `requests` всех Pod'ов **превышает** ёмкость ноды. Scheduler позволяет это для CPU (сжимаемый ресурс), но это риск.

**Пример:**

```
Нода: 4 CPU

Pod A: requests=1 CPU
Pod B: requests=1 CPU
Pod C: requests=1 CPU
Pod D: requests=1 CPU
Pod E: requests=1 CPU

Сумма requests: 5 CPU > 4 CPU

Scheduler может поставить все 5 Pod'ов, если у них limits больше requests.
```

**Что произойдёт при пиковой нагрузке:**

- Все Pod'ы захотят больше requests.
- CPU не хватит.
- Все замедлятся (throttling).

**Для memory overcommit ещё опаснее:** OOM Killer начнёт убивать Pod'ы.

**Правило:** на production не делай overcommit. Сумма requests ≤ ёмкость ноды.

### 🎯 Качество CPU: `cpu.weight` и nice

В Linux CPU — это **относительный** ресурс. Если два Pod'а хотят 1 CPU, а доступно только 1 — они получат по 0.5.

**Веса:**

- Если у Pod requests=1 CPU, а у другого requests=2 CPU — второй получит **в два раза больше** времени при конкуренции.
- Это делается через cgroup v2 `cpu.weight`.

**Пример:**

```
Нода: 2 CPU

Pod A: requests=500m (вес 500)
Pod B: requests=1500m (вес 1500)

Оба хотят максимум CPU. Доступно 2 CPU.

Pod A получит: 2 × (500 / 2000) = 0.5 CPU
Pod B получит: 2 × (1500 / 2000) = 1.5 CPU
```

### 🔬 Практика: requests и limits в действии

```bash
# 1. Создать Pod с requests и limits
kubectl run test --image=nginx \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=200m,memory=256Mi

# 2. Посмотреть ресурсы
kubectl describe pod test | grep -A 5 Limits
# Limits:
#   cpu:     200m
#   memory:  256Mi
# Requests:
#   cpu:     100m
#   memory:  128Mi

# 3. Посмотреть потребление
kubectl top pod test
# NAME   CPU(cores)   MEMORY(bytes)
# test   1m           12Mi

# 4. Попробовать съесть больше
kubectl exec test -- sh -c "stress --cpu 2 --timeout 30s"
# Pod будет throttled — не получит больше 200m CPU
```

### 💡 Практика: как правильно выбирать requests и limits

**✅ ОБЯЗАТЕЛЬНО:**

1. **Всегда указывай requests и limits.** Без них Pod попадает в BestEffort QoS и может быть убит первым.
2. **Requests = типичное потребление.**
3. **Limits = 1.5-2x от типичного.**
4. **Не делай overcommit в production.**

**👍 СТОИТ:**

5. **Мониторь реальное потребление.** `kubectl top pod`, Prometheus.
6. **Разные значения для CPU и memory.** CPU — сжимаемый (throttling), memory — несжимаемый (OOM).

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **`limits` для CPU = `requests` для CPU** — если приложение не должно throttled.

**❌ НЕ ДЕЛАЙ:**

8. **Не ставь limits = requests без понимания.** Это даёт Guaranteed QoS, но может ограничить.
9. **Не ставь requests слишком большими.** Pod'ы не будут размещаться.
10. **Не ставь requests = 0.** Scheduler не будет учитывать.
11. **Не забывай про init-контейнеры.** Они тоже потребляют ресурсы.

### Где мы сейчас

Мы разобрали requests и limits. Теперь — **QoS-классы** — как K8s классифицирует Pod'ы для OOM.

---

## 9.3 QoS-классы: Guaranteed, Burstable, BestEffort

### 🔌 Проблема: кого убивать первым при нехватке памяти

Память на ноде закончилась. OOM Killer должен выбрать жертву. Кого убить?

**Правильный ответ:** сначала тех, кто **менее важен** и **не имеет гарантий**.

Kubernetes классифицирует Pod'ы в **три QoS-класса**:

1. **Guaranteed** — высший приоритет, последние в очереди на убийство.
2. **Burstable** — средний.
3. **BestEffort** — низший, первыми убиваются.

### 📊 QoS-классы

**1. Guaranteed**

**Условия:**

- **Все** контейнеры в Pod'е имеют `requests` = `limits` для CPU и memory.
- Никаких других ресурсов с разными requests/limits.

**Пример:**

```yaml
spec:
  containers:
    - name: myapp
      resources:
        requests:
          cpu: "500m"
          memory: "256Mi"
        limits:
          cpu: "500m"           # = requests
          memory: "256Mi"       # = requests
```

**Что даёт:**

- **Гарантированные ресурсы.** Pod всегда получит ровно 500m CPU и 256Mi RAM.
- **Последний в очереди на OOM.** Убивается только если других вариантов нет.

**Когда использовать:** критичные production-сервисы (БД, API).

**2. Burstable**

**Условия:**

- Хотя бы один контейнер имеет `requests` ≠ `limits` (или limits не указаны).
- Не подходит под Guaranteed.

**Пример 1:**

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

**Пример 2:**

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  # limits не указаны
```

**Что даёт:**

- **Гарантированные requests, но может использовать больше.**
- **Средний приоритет** в очереди на OOM.
- **OOM score зависит от потребления относительно requests.**

**Когда использовать:** большинство приложений.

**3. BestEffort**

**Условия:**

- **Ни один** контейнер не имеет requests или limits.

**Пример:**

```yaml
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      # нет resources
```

**Что даёт:**

- **Никаких гарантий.** Pod может получить сколько угодно, но может быть убит первым.
- **Первый в очереди на OOM.**

**Когда использовать:** почти никогда в production. Только для dev/experiments.

### 🎯 Как QoS влияет на OOM

**Порядок убийства (при нехватке памяти на ноде):**

1. **BestEffort** — первыми.
2. **Burstable** — те, у кого потребление > requests (в порядке убывания oom_score).
3. **Guaranteed** — последние.

**Внутри Burstable:** Pod с меньшими requests убивается первым. Pod, использующий память близко к limits, убивается раньше.

### 🎯 QoS и OOM score

Kubernetes задаёт `oom_score_adj` каждому контейнеру:

| QoS | oom_score_adj |
|:---|:---|
| **Guaranteed** | -997 |
| **Burstable** | 2-999 (зависит от потребления) |
| **BestEffort** | 1000 |

**Больше oom_score_adj = вероятнее убийство.**

Для Burstable Kubernetes считает:

```
oom_score_adj = 1000 - (1000 × requests.memory / limits.memory)
```

**Пример:**

- Requests=256Mi, limits=512Mi → `oom_score_adj = 500`.
- Requests=256Mi, limits=256Mi → Guaranteed → `-997`.

Чем **ближе requests к limits**, тем **ниже oom_score_adj**, тем безопаснее.

### 🔬 Практика: QoS в действии

```bash
# 1. Guaranteed Pod
kubectl run guaranteed --image=nginx \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=100m,memory=128Mi

# 2. Burstable Pod
kubectl run burstable --image=nginx \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=500m,memory=512Mi

# 3. BestEffort Pod
kubectl run besteffort --image=nginx

# 4. Посмотреть QoS
kubectl get pod guaranteed -o jsonpath='{.status.qosClass}'
# Guaranteed

kubectl get pod burstable -o jsonpath='{.status.qosClass}'
# Burstable

kubectl get pod besteffort -o jsonpath='{.status.qosClass}'
# BestEffort

# 5. Посмотреть oom_score_adj
kubectl exec guaranteed -- cat /proc/1/oom_score_adj
# -997

kubectl exec besteffort -- cat /proc/1/oom_score_adj
# 1000
```

### 💡 Практика: как правильно выбирать QoS

**✅ ОБЯЗАТЕЛЬНО:**

1. **Guaranteed для критичных сервисов.** БД, API, что-то важное.
2. **Burstable для обычных приложений.** Большинство случаев.
3. **BestEffort — только для dev/экспериментов.**

**👍 СТОИТ:**

4. **Для Burstable устанавливать requests близко к типичному потреблению.** Чем ближе requests к limits — тем ниже oom_score_adj.

**❌ НЕ ДЕЛАЙ:**

5. **Не оставляй Pod'ы без resources в production.** Они BestEffort и будут убиты первыми.
6. **Не делай все Pod'ы Guaranteed без понимания.** Может привести к неоптимальному использованию.

### Где мы сейчас

Мы разобрали QoS-классы. Теперь — **nodeSelector** — простой способ выбрать ноду.

---

## 9.4 NodeSelector: простой способ выбрать ноду

### 🔌 Проблема: запустить Pod на конкретной ноде

У тебя есть ноды с разными характеристиками:

- **GPU-ноды** — для ML.
- **SSD-ноды** — для БД.
- **Обычные ноды** — для всего остального.

Как запустить Pod именно на GPU-ноде?

**Решение:** `nodeSelector`.

### 📊 Что такое nodeSelector

**nodeSelector** — простейший способ указать, на каких нодах может запуститься Pod.

**Как работает:**

1. Нодам присваиваются labels (`kubectl label node`).
2. Pod в `spec.nodeSelector` указывает labels.
3. Scheduler ставит Pod **только** на ноды с этими labels.

### 🎯 Пример

**Добавить label ноде:**

```bash
kubectl label node node-1 disktype=ssd
kubectl label node node-2 disktype=hdd
kubectl label node node-3 gpu=true
```

**Pod с nodeSelector:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  nodeSelector:
    disktype: ssd       # только на ноды с disktype=ssd
  containers:
    - name: myapp
      image: myapp:1.0
```

**Что произойдёт:**

- Scheduler посмотрит все ноды.
- Отфильтрует те, у которых нет `disktype=ssd`.
- Поставит Pod на одну из подходящих.

**Если нет подходящих нод:** Pod останется в Pending.

```
Events:
  Warning  FailedScheduling  0/3 nodes are available: 3 node(s) didn't match node selector.
```

### 🎯 Встроенные labels нод

Kubernetes автоматически добавляет labels нодам:

| Label | Пример значения |
|:---|:---|
| `kubernetes.io/hostname` | `node-1` |
| `kubernetes.io/os` | `linux` |
| `kubernetes.io/arch` | `amd64` |
| `node.kubernetes.io/instance-type` | `m5.large` (в облаке) |
| `topology.kubernetes.io/zone` | `us-west-2a` |
| `topology.kubernetes.io/region` | `us-west-2` |

**Пример использования:**

```yaml
spec:
  nodeSelector:
    kubernetes.io/os: linux
    topology.kubernetes.io/zone: us-west-2a
```

### 🎯 Ограничения nodeSelector

- **Только equality.** `key=value`, без операторов.
- **Нельзя указать "not in"** или "exists".
- **Нельзя дать приоритет** — просто фильтр.

**Для более гибких условий — `nodeAffinity` (подглава 9.5).**

### 🔬 Практика: nodeSelector

```bash
# 1. Посмотреть labels нод
kubectl get nodes --show-labels

# 2. Добавить label
kubectl label node node-1 environment=production

# 3. Создать Pod с nodeSelector
kubectl run test --image=nginx --dry-run=client -o yaml > pod.yaml
# Отредактировать pod.yaml, добавив nodeSelector

kubectl apply -f pod.yaml

# 4. Проверить, на какой ноде Pod
kubectl get pod test -o wide
# NAME   READY   STATUS   ...   NODE
# test   1/1     Running  ...   node-1

# 5. Если ошиблись с label — Pod в Pending
kubectl describe pod test
# Events:
#   Warning  FailedScheduling  0/3 nodes are available: 3 node(s) didn't match node selector.
```

### 💡 Практика: как правильно использовать nodeSelector

**✅ ОБЯЗАТЕЛЬНО:**

1. **Использовать для простых случаев** — GPU, SSD, регион.
2. **Добавлять labels нодам заранее**, не полагаться на случайные.

**👍 СТОИТ:**

3. **Для сложных случаев — `nodeAffinity`** (более гибкий).

**❌ НЕ ДЕЛАЙ:**

4. **Не используй nodeSelector для anti-affinity.** Для этого есть `podAntiAffinity`.

### Где мы сейчас

Мы разобрали nodeSelector. Теперь — **nodeAffinity** — более гибкий механизм.

---

## 9.5 Node Affinity и Anti-Affinity: гибкое размещение

### 🔌 Проблема: nodeSelector слишком прост

`nodeSelector` умеет только `key=value`. А если нужно:

- «Ноды с zone=us-west-2a ИЛИ zone=us-west-2b».
- «Ноды БЕЗ GPU».
- «Предпочитать ноды с SSD, но если нет — любые».

**Решение:** `nodeAffinity`.

### 📊 Что такое nodeAffinity

**nodeAffinity** — более гибкий способ указать, на каких нодах может (или хочет) запуститься Pod.

Два типа:

| Тип | Что означает |
|:---|:---|
| **requiredDuringSchedulingIgnoredDuringExecution** | **Жёсткое условие.** Pod **не запустится** без него. |
| **preferredDuringSchedulingIgnoredDuringExecution** | **Мягкое условие.** Scheduler предпочтёт, но не обязательно. |

**Ключевое слово `IgnoredDuringExecution`:** если labels ноды изменились **после** запуска Pod'а — Pod **не переедет**. Это правило применяется только при планировании.

### 🎯 Required nodeAffinity

**Синтаксис:**

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values: [ssd, nvme]
```

**Операторы:**

| Оператор | Что означает |
|:---|:---|
| `In` | Значение в списке |
| `NotIn` | Значение не в списке |
| `Exists` | Label существует |
| `DoesNotExist` | Label отсутствует |
| `Gt` / `Lt` | Больше/меньше (для числовых values) |

**Множественные matchExpressions:**

```yaml
nodeSelectorTerms:
  - matchExpressions:
      - key: disktype
        operator: In
        values: [ssd]
      - key: environment
        operator: In
        values: [production]
```

**Все условия** в одном `matchExpressions` должны выполняться (**AND**).

**Множественные nodeSelectorTerms:**

```yaml
nodeSelectorTerms:
  - matchExpressions:
      - key: zone
        operator: In
        values: [us-west-2a]
  - matchExpressions:
      - key: zone
        operator: In
        values: [us-west-2b]
```

**Любой** из terms может подойти (**OR**).

### 🎯 Preferred nodeAffinity

**Синтаксис:**

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100              # 1-100
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values: [ssd]
```

**Weight** — приоритет. Чем больше, тем важнее для scheduler.

**Пример с двумя preferences:**

```yaml
preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    preference:
      matchExpressions:
        - key: disktype
          operator: In
          values: [nvme]         # самый важный
  - weight: 50
    preference:
      matchExpressions:
        - key: disktype
          operator: In
          values: [ssd]          # менее важный
```

Если есть ноды с nvme — предпочтёт их. Если нет — с ssd. Если нет — любые.

### 🎯 Комбинирование required и preferred

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: [us-west-2a, us-west-2b]     # только эти зоны
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values: [nvme]                        # предпочитать nvme
```

**Что произойдёт:**

1. Scheduler отфильтрует ноды в `us-west-2a` или `us-west-2b`.
2. Среди них предпочтёт ноды с `disktype=nvme`.
3. Если nvme нет — запустит на любой подходящей.

### 🎯 Практический пример: GPU

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: nvidia.com/gpu
                operator: Exists
              - key: gpu-type
                operator: In
                values: [a100, h100]
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: gpu-type
                operator: In
                values: [h100]           # предпочитать h100
  containers:
    - name: trainer
      image: my-training:1.0
      resources:
        limits:
          nvidia.com/gpu: 1              # запросить 1 GPU
```

**Что произойдёт:**

- Pod запустится **только** на нодах с GPU A100 или H100.
- Из них предпочтёт H100.
- Запросит 1 GPU (Kubernetes выделит).

### 🔬 Практика: nodeAffinity

```bash
# 1. Создать Pod с required nodeAffinity
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values: [linux]
  containers:
    - name: test
      image: nginx
EOF

# 2. Проверить, что Pod запущен
kubectl get pod test-affinity

# 3. Попробовать "невозможный" required
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-impossible
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values: [quantum-disk]
  containers:
    - name: test
      image: nginx
EOF

# 4. Pod будет в Pending
kubectl get pod test-impossible
# Pending

kubectl describe pod test-impossible
# Events:
#   Warning  FailedScheduling  0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.
```

### 💡 Практика: как правильно использовать nodeAffinity

**✅ ОБЯЗАТЕЛЬНО:**

1. **Использовать `required` для жёстких требований.** «Только GPU», «только production зона».
2. **Использовать `preferred` для оптимизации.** «Лучше SSD, но не критично».

**👍 СТОИТ:**

3. **Использовать встроенные labels** (`kubernetes.io/os`, `topology.kubernetes.io/zone`).
4. **Комбинировать required + preferred** для гибкости.

**❌ НЕ ДЕЛАЙ:**

5. **Не используй required без понимания.** Если нет подходящих нод — Pod в Pending.
6. **Не пиши сложные required без тестирования.** Легко ошибиться.

### Где мы сейчас

Мы разобрали nodeAffinity. Теперь — **Pod affinity и anti-affinity** — размещение относительно других Pod'ов.

---

## 9.6 Pod Affinity и Anti-Affinity: размещение относительно других Pod'ов

### 🔌 Проблема: разместить Pod'ы рядом или далеко

Иногда нужно:

- **Рядом.** «Кэш и приложение — на одной ноде, чтобы не гонять трафик по сети».
- **Далеко.** «Реплики приложения — на разных нодах, чтобы одна нода не убила всё».

**Решение:** `podAffinity` и `podAntiAffinity`.

### 📊 Что такое Pod affinity

**PodAffinity** — разместить Pod **рядом** с другими Pod'ами (на той же ноде или в той же зоне).

**PodAntiAffinity** — разместить Pod **далеко** от других Pod'ов.

**Topology Key** — уровень топологии:

- `kubernetes.io/hostname` — нода.
- `topology.kubernetes.io/zone` — зона доступности.
- `topology.kubernetes.io/region` — регион.

### 🎯 Pod Affinity

**Пример:** разместить кэш рядом с приложением.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis-cache
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: myapp           # рядом с Pod'ами app=myapp
          topologyKey: kubernetes.io/hostname    # на той же ноде
  containers:
    - name: redis
      image: redis:7
```

**Что произойдёт:**

- Scheduler найдёт ноды, где есть Pod'ы с `app=myapp`.
- Поставит `redis-cache` **только** на одну из этих нод.
- Если нет нод с `app=myapp` — Pod в Pending.

### 🎯 Pod Anti-Affinity

**Пример:** распределить реплики по разным нодам.

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
                  app: myapp       # не размещать рядом с другими myapp
              topologyKey: kubernetes.io/hostname
      containers:
        - name: myapp
          image: myapp:1.0
```

**Что произойдёт:**

- Scheduler найдёт ноды **без** Pod'ов `app=myapp`.
- Поставит каждый Pod на **разную** ноду.
- Если нод меньше, чем реплик — часть Pod'ов будет в Pending.

**Три реплики на трёх нодах:**

```
node-1: myapp-abc (pod 1)
node-2: myapp-def (pod 2)
node-3: myapp-ghi (pod 3)
```

**Если нод две, а реплик три:**

```
node-1: myapp-abc
node-2: myapp-def
(третий Pod в Pending)
```

### 🎯 Preferred Anti-Affinity

`required` слишком жёсткое — если нод мало, Pod'ы не запустятся. **`preferred`** — мягкое:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: myapp
          topologyKey: kubernetes.io/hostname
```

**Что произойдёт:**

- Scheduler **предпочтёт** ноды без других Pod'ов `app=myapp`.
- Но если таких нет — запустит на любой.

### 🎯 Комбинация Pod + Node Affinity

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: disktype
              operator: In
              values: [ssd]
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: myapp
        topologyKey: kubernetes.io/hostname
```

**Что произойдёт:**

- Scheduler сначала отфильтрует ноды с SSD.
- Среди них выберет без Pod'ов `app=myapp`.

### 🎯 Проблемы производительности

**PodAffinity и PodAntiAffinity — дорогие для scheduler.**

- Scheduler должен смотреть **все** Pod'ы на каждой ноде.
- Для больших кластеров (1000+ нод) это медленно.

**Правило:** используй **`preferred`** где возможно. `required` — только для критичных случаев.

**Для больших кластеров** — используй `topologySpreadConstraints` (подглава 9.9).

### 🔬 Практика: Pod anti-affinity

```bash
# 1. Deployment с anti-affinity
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spread
  template:
    metadata:
      labels:
        app: spread
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: spread
                topologyKey: kubernetes.io/hostname
      containers:
        - name: app
          image: nginx
EOF

# 2. Проверить распределение
kubectl get pods -l app=spread -o wide
# NAME                     NODE
# spread-7d9f8c6b4d-abc12  node-1
# spread-7d9f8c6b4d-def34  node-2
# spread-7d9f8c6b4d-ghi56  node-3
```

### 💡 Практика: как правильно использовать Pod affinity

**✅ ОБЯЗАТЕЛЬНО:**

1. **`preferred` для распределения реплик.** Не блокирует запуск.
2. **`required` для жёстких случаев** (кэш рядом с приложением).

**👍 СТОИТ:**

3. **`topologyKey: kubernetes.io/hostname`** для распределения по нодам.
4. **`topologyKey: topology.kubernetes.io/zone`** для распределения по зонам.

**❌ НЕ ДЕЛАЙ:**

5. **Не используй `required` anti-affinity для реплик > нод.** Часть Pod'ов будет в Pending.
6. **Не используй podAffinity в больших кластерах без необходимости.** Медленно.

### Где мы сейчас

Мы разобрали Pod affinity. Теперь — **taints и tolerations** — выделенные ноды.

---

## 9.7 Taints и Tolerations: выделенные ноды

### 🔌 Проблема: как выделить ноду для конкретных Pod'ов

У тебя 10 нод. Одна — для критичной БД. Ты не хочешь, чтобы на ней запускались другие Pod'ы.

**Решение:** taints и tolerations.

### 📊 Что такое taints

**Taint** — это «пятно» на ноде: «не запускай здесь Pod'ы, если они не толерантны».

**Taint состоит из:**

- **key** — имя.
- **value** — значение (опционально).
- **effect** — что делать:
  - `NoSchedule` — не размещать новые Pod'ы без toleration.
  - `PreferNoSchedule` — предпочитать не размещать.
  - `NoExecute` — **выгнать** существующие Pod'ы без toleration.

**Toleration** — «прощение»: «этот Pod может запускаться на ноде с этим taint».

### 🎯 Добавить taint ноде

```bash
kubectl taint nodes node-1 dedicated=database:NoSchedule
```

**Что произошло:**

- Node-1 теперь имеет taint `dedicated=database:NoSchedule`.
- Scheduler **не будет** размещать Pod'ы без toleration на этой ноде.

**Удалить taint:**

```bash
kubectl taint nodes node-1 dedicated=database:NoSchedule-
```

(Минус в конце ключа удаляет taint.)

### 🎯 Pod с toleration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: database
      effect: NoSchedule
  containers:
    - name: postgres
      image: postgres:16
```

**Что произойдёт:**

- Scheduler видит taint `dedicated=database:NoSchedule`.
- Pod имеет toleration → может быть размещён на node-1.
- Другие Pod'ы без toleration — не могут.

### 🎯 Операторы tolerations

| Оператор | Что означает |
|:---|:---|
| `Equal` | Значение точно равно |
| `Exists` | Ключ существует (значение любое) |

**Пример с `Exists`:**

```yaml
tolerations:
  - key: dedicated
    operator: Exists
    effect: NoSchedule
```

Pod толерантен к **любому** taint с ключом `dedicated`.

**Toleration на все taints:**

```yaml
tolerations:
  - operator: Exists
```

Pod толерантен ко **всем** taints. Используется редко (например, для системных Pod'ов).

### 🎯 Эффекты taints

**NoSchedule:**

- Scheduler не размещает новые Pod'ы без toleration.
- Существующие Pod'ы остаются.

**PreferNoSchedule:**

- Scheduler предпочитает не размещать.
- Но если нет других нод — разместит.

**NoExecute:**

- Scheduler не размещает новые Pod'ы без toleration.
- **Существующие** Pod'ы без toleration **выгоняются**.

**NoExecute и tolerationSeconds:**

```yaml
tolerations:
  - key: node.kubernetes.io/unreachable
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 300     # 5 минут
```

**Что означает:** Pod может оставаться на ноде 5 минут после того, как нода стала недоступной. Потом выгоняется.

**Используется для:**

- `node.kubernetes.io/unreachable` — нода недоступна.
- `node.kubernetes.io/not-ready` — нода не готова.

По умолчанию Pod'ы имеют toleration на эти taints с `tolerationSeconds: 300`. Поэтому при падении ноды Pod'ы ждут 5 минут, потом переезжают.

### 🎯 Встроенные taints

Kubernetes автоматически добавляет taints:

| Taint | Когда |
|:---|:---|
| `node.kubernetes.io/not-ready` | Нода не готова |
| `node.kubernetes.io/unreachable` | Нода недоступна |
| `node.kubernetes.io/memory-pressure` | Мало памяти |
| `node.kubernetes.io/disk-pressure` | Мало места |
| `node.kubernetes.io/pid-pressure` | Много процессов |
| `node.kubernetes.io/network-unavailable` | Сеть недоступна |
| `node.kubernetes.io/unschedulable` | Нода помечена unschedulable |
| `node-role.kubernetes.io/control-plane` | Control plane нода |

**Control plane ноды** имеют taint `node-role.kubernetes.io/control-plane:NoSchedule`, чтобы обычные Pod'ы на них не запускались. Только системные Pod'ы с toleration.

### 🎯 Практический паттерн: выделенная нода для БД

**1. Пометить ноду label:**

```bash
kubectl label node node-1 workload=database
```

**2. Добавить taint:**

```bash
kubectl taint nodes node-1 workload=database:NoSchedule
```

**3. Pod с nodeAffinity + toleration:**

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: workload
                operator: In
                values: [database]
  tolerations:
    - key: workload
      operator: Equal
      value: database
      effect: NoSchedule
```

**Что произойдёт:**

- Pod может запуститься **только** на ноде с label `workload=database`.
- Только этот Pod может там запуститься (благодаря taint).
- Другие Pod'ы не могут.

**Это — полное выделение ноды.**

### 🔬 Практика: taints и tolerations

```bash
# 1. Добавить taint
kubectl taint nodes node-1 test=test-taint:NoSchedule

# 2. Создать Pod без toleration
kubectl run test --image=nginx

# 3. Pod не запустится на node-1
kubectl get pod test -o wide
# NODE: node-2 или node-3 (не node-1)

# 4. Создать Pod с toleration
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-toleration
spec:
  tolerations:
    - key: test
      operator: Equal
      value: test-taint
      effect: NoSchedule
  containers:
    - name: test
      image: nginx
EOF

# 5. Pod может запуститься на node-1
kubectl get pod test-toleration -o wide

# 6. Удалить taint
kubectl taint nodes node-1 test=test-taint:NoSchedule-
```

### 💡 Практика: как правильно использовать taints

**✅ ОБЯЗАТЕЛЬНО:**

1. **Использовать taints для выделенных нод.** БД, GPU, специализированные задачи.
2. **Комбинировать с nodeAffinity** для полного контроля.
3. **`NoSchedule` для новых Pod'ов.**

**👍 СТОИТ:**

4. **`NoExecute` для выгона существующих Pod'ов** при необходимости.
5. **`PreferNoSchedule`** для мягкого предпочтения.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй `NoExecute` без понимания.** Выгонит работающие Pod'ы.
7. **Не забывай про tolerationSeconds.** Иначе Pod'ы будут выгнаны сразу.

### Где мы сейчас

Мы разобрали taints и tolerations. Теперь — **priority и preemption** — кто важнее.

---

## 9.8 Priority и Preemption: кто важнее

### 🔌 Проблема: как запустить критичный Pod, когда места нет

Все ноды забиты. Тебе нужно запустить критичный Pod. Scheduler не может найти место.

**Решение:** priority и preemption.

### 📊 Что такое PriorityClass

**PriorityClass** — это объект Kubernetes, который задаёт **числовой приоритет** для Pod'ов.

**Чем выше число — тем важнее Pod.**

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Высокий приоритет для критичных сервисов"
```

**Использование:**

```yaml
spec:
  priorityClassName: high-priority
  containers:
    - name: myapp
      image: myapp:1.0
```

### 🎯 Как работает preemption

**Preemption** — это когда высокоприоритетный Pod **выгоняет** низкоприоритетные, чтобы освободить место.

**Алгоритм:**

1. Scheduler не может найти ноду для Pod'а.
2. Если у Pod'а высокий приоритет — scheduler пытается **выгнать** другие Pod'ы.
3. Находит ноду с Pod'ами **ниже** приоритетом.
4. Убирает их (переводит в Pending).
5. Ставит высокоприоритетный Pod.

**Пример:**

```
Нода: 4 CPU

Pod A (priority=100): requests=2 CPU
Pod B (priority=100): requests=2 CPU
Pod C (priority=1000): requests=2 CPU    ← не помещается

Preemption:
Pod C выгоняет Pod B (низший приоритет).
Pod B → Pending.
Pod C → node-1.
```

### 🎯 Встроенные PriorityClass'ы

Kubernetes имеет два встроенных:

| Class | Value | Для чего |
|:---|:---|:---|
| `system-cluster-critical` | 2000000000 | Критичные системы (kube-dns, metrics-server) |
| `system-node-critical` | 2000001000 | Самое важное (kube-proxy, CNI) |

**Обычные Pod'ы** имеют priority = 0 (default).

### 🎯 Практические PriorityClass'ы

**Примеры:**

```yaml
# Критичные (БД, API)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical
value: 100000
globalDefault: false

---
# Обычные
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: standard
value: 1000
globalDefault: true      # default для всех Pod'ов
```

**Использование:**

```yaml
spec:
  priorityClassName: critical
```

### 🎯 Preemption в действии

```bash
# 1. Создать PriorityClass
kubectl apply -f - <<EOF
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low
value: 100
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high
value: 1000
EOF

# 2. Создать Pod с низким приоритетом
kubectl run low-pod --image=nginx --priorityClassName=low

# 3. Создать Pod с высоким приоритетом
kubectl run high-pod --image=nginx --priorityClassName=high

# 4. Если места не хватает — high выгонит low
kubectl get pods
# low-pod  Pending  (выгнан)
# high-pod Running
```

### 🎯 Проблемы preemption

**1. Thundering herd.** Если много низкоприоритетных Pod'ов, preemption может выгнать многих. Приложение может не справиться с одновременным перезапуском.

**2. Непредсказуемость.** Pod'ы, которые работали, внезапно выгоняются.

**3. Cascading failures.** Если важный Pod выгнан, могут быть побочные эффекты.

**Правило:** используй preemption **осторожно**. Только для действительно критичных Pod'ов.

### 🔬 Практика: priority

```bash
# 1. Посмотреть PriorityClass'ы
kubectl get priorityclass
# NAME                      VALUE
# system-cluster-critical   2000000000
# system-node-critical      2000001000

# 2. Посмотреть priority Pod'а
kubectl get pod myapp -o jsonpath='{.spec.priority}'
# 0 (default)
```

### 💡 Практика: как правильно использовать priority

**✅ ОБЯЗАТЕЛЬНО:**

1. **Использовать `system-*-critical` не трогать.** Они для системных Pod'ов.
2. **Создавать свои PriorityClass'ы** для важных сервисов.
3. **`globalDefault: true`** для дефолтного класса.

**👍 СТОИТ:**

4. **Разные классы для разных уровней:** critical, standard, low.
5. **Осторожно с preemption.** Только для критичных.

**❌ НЕ ДЕЛАЙ:**

6. **Не ставь всем Pod'ам high priority.** Иначе preemption не работает.
7. **Не используй preemption для batch-задач.** Может выгнать всё.

### Где мы сейчас

Мы разобрали priority и preemption. Теперь — **topology spread** — равномерное распределение.

---

## 9.9 Topology Spread: равномерное распределение

### 🔌 Проблема: podAntiAffinity не масштабируется

`podAntiAffinity` с `topologyKey: hostname` требует, чтобы Pod'ы были на **разных** нодах. Если нод 3, а реплик 5 — 2 Pod'а не запустятся.

**Лучше:** «распределяй равномерно, но не блокируй».

**Решение:** `topologySpreadConstraints`.

### 📊 Что такое topology spread

**topologySpreadConstraints** — распределение Pod'ов равномерно по топологии (зонам, нодам) с контролем максимальной неравномерности.

**Поля:**

| Поле | Что означает |
|:---|:---|
| `maxSkew` | Максимальная разница между самыми заполненными и самыми пустыми доменами |
| `topologyKey` | Что считать доменом (hostname, zone) |
| `whenUnsatisfiable` | `DoNotSchedule` (жёстко) или `ScheduleAnyway` (мягко) |
| `labelSelector` | Какие Pod'ы считать |

### 🎯 Пример

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
      containers:
        - name: myapp
          image: myapp:1.0
```

**Что произойдёт:**

- У тебя 3 зоны: `us-west-2a`, `us-west-2b`, `us-west-2c`.
- 6 реплик.
- `maxSkew: 1` — разница между зонами не более 1.
- Распределение: 2-2-2.

**Пример:**

```
us-west-2a: 2 Pod'а
us-west-2b: 2 Pod'а
us-west-2c: 2 Pod'а

Skew = 0 (все равны) ✅
```

**Если зон 2, а реплик 5:**

```
us-west-2a: 3 Pod'а
us-west-2b: 2 Pod'а

Skew = 1 ✅ (maxSkew=1)
```

**Если зон 2, а реплик 6:**

```
us-west-2a: 3 Pod'а
us-west-2b: 3 Pod'а

Skew = 0 ✅
```

### 🎯 Распределение по нодам

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname     # домены — ноды
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: myapp
```

**Что произойдёт:**

- 6 реплик, 3 ноды.
- `maxSkew: 1` — разница между нодами не более 1.
- Распределение: 2-2-2.

### 🎯 Комбинирование зон и нод

```yaml
topologySpreadConstraints:
  # Сначала по зонам
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: myapp
  # Потом по нодам внутри зон
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: myapp
```

**Что произойдёт:**

- Сначала распределит по зонам равномерно.
- Внутри зон — по нодам (мягко).

### 🎯 whenUnsatisfiable

**DoNotSchedule** — жёстко. Если нельзя выполнить — Pod в Pending.

**ScheduleAnyway** — мягко. Scheduler постарается, но если не получится — разместит как угодно.

**Рекомендация:** `DoNotSchedule` для критичных (зоны), `ScheduleAnyway` для мягкого (ноды).

### 🔬 Практика: topology spread

```bash
# 1. Deployment с topology spread
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-test
spec:
  replicas: 6
  selector:
    matchLabels:
      app: spread-test
  template:
    metadata:
      labels:
        app: spread-test
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: spread-test
      containers:
        - name: app
          image: nginx
EOF

# 2. Проверить распределение
kubectl get pods -l app=spread-test -o wide
# Каждая нода получит примерно равное количество Pod'ов
```

### 💡 Практика: как правильно использовать topology spread

**✅ ОБЯЗАТЕЛЬНО:**

1. **`maxSkew: 1` для равномерного распределения.**
2. **`topologyKey: topology.kubernetes.io/zone`** для отказоустойчивости.
3. **`whenUnsatisfiable: DoNotSchedule`** для критичных сервисов.

**👍 СТОИТ:**

4. **Комбинировать зоны и ноды.** Сначала зоны (жёстко), потом ноды (мягко).

**❌ НЕ ДЕЛАЙ:**

5. **Не путай с podAntiAffinity.** TopologySpread — равномерность, AntiAffinity — «не рядом».

### Где мы сейчас

Мы разобрали topology spread. Теперь — **диагностика** проблем с планированием.

---

## 9.10 Диагностика: Pod в Pending из-за ресурсов

### 🔌 Проблема: Pod не запускается

Pod в Pending. Причина — ресурсы. Как диагностировать?

### 🔍 Алгоритм диагностики

**Шаг 1: Посмотреть статус**

```bash
kubectl get pods
# NAME    READY   STATUS    RESTARTS   AGE
# myapp   0/1     Pending   0          5m
```

**Шаг 2: Описать Pod**

```bash
kubectl describe pod myapp
```

Смотри **Events** внизу.

**Шаг 3: Прочитать сообщение**

```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  5m    default-scheduler  0/3 nodes are available:
    3 Insufficient cpu.
```

**Что означает:**

- 0 из 3 нод доступны.
- 3 ноды не подходят из-за **нехватки CPU**.
- Pod требует больше CPU, чем свободно на любой ноде.

**Шаг 4: Проверить ресурсы нод**

```bash
kubectl describe nodes | grep -A 5 "Allocated resources"
```

Вывод:

```
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                3500m (87%)   5000m (125%)
  memory             7Gi (45%)     10Gi (65%)
```

**Что означает:**

- Нода имеет 4 CPU.
- 3500m (87%) уже запрошено.
- Свободно: 500m.
- Pod требует больше 500m → не помещается.

**Шаг 5: Проверить requests Pod'а**

```bash
kubectl get pod myapp -o jsonpath='{.spec.containers[*].resources.requests}'
```

### 🎯 Типичные сообщения и решения

**1. `Insufficient cpu` / `Insufficient memory`**

**Причина:** на нодах недостаточно свободных ресурсов.

**Решения:**

- Уменьшить `requests` Pod'а.
- Добавить ноды.
- Удалить другие Pod'ы.
- Проверить, что нет "зомби" Pod'ов, которые занимают ресурсы.

**2. `node(s) didn't match node selector`**

**Причина:** нет нод с нужными labels.

**Решения:**

- Проверить labels нод: `kubectl get nodes --show-labels`.
- Исправить `nodeSelector` в Pod'е.
- Добавить labels нодам.

**3. `node(s) had untolerated taint`**

**Причина:** Pod не толерантен к taint ноды.

**Решения:**

- Добавить toleration в Pod.
- Удалить taint с ноды.

**4. `node(s) didn't match Pod's node affinity/selector`**

**Причина:** нет нод, соответствующих `nodeAffinity`.

**Решения:**

- Проверить `nodeAffinity` в Pod'е.
- Проверить labels нод.
- Исправить условие.

**5. `pod has unbound immediate PersistentVolumeClaims`**

**Причина:** PVC не привязан к PV.

**Решения:**

- Проверить PVC: `kubectl get pvc`.
- Проверить StorageClass.
- Проверить, что есть доступные PV.

**6. `0/3 nodes are available: 3 node(s) were unschedulable`**

**Причина:** все ноды помечены как unschedulable.

**Решения:**

- Проверить: `kubectl get nodes` — статус `SchedulingDisabled`.
- Снять: `kubectl uncordon node-1`.

### 🎯 Инструменты диагностики

**1. `kubectl describe pod`** — главный источник.

**2. `kubectl describe nodes`** — ресурсы нод.

**3. `kubectl top nodes`** — реальное потребление (metrics-server).

**4. `kubectl get events`** — события кластера.

**5. Scheduler logs:**

```bash
kubectl logs -n kube-system -l component=kube-scheduler
```

### 🎯 Практический пример

```bash
# 1. Создать Pod с невозможными requests
kubectl run test --image=nginx --requests=cpu=100

# 2. Проверить
kubectl get pod test
# Pending

# 3. Описать
kubectl describe pod test
# Events:
#   Warning  FailedScheduling  0/3 nodes are available:
#     3 Insufficient cpu.

# 4. Проверить ноды
kubectl describe nodes | grep -A 5 "Allocated resources"
# cpu: 3500m (87%)
# Свободно: 500m
# Pod требует: 100 CPU

# 5. Решение: уменьшить requests
kubectl delete pod test
kubectl run test --image=nginx --requests=cpu=100m
# Pod запустится
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Всегда `kubectl describe pod`.** Events внизу.
2. **Читать сообщение.** Оно точно говорит причину.
3. **Проверять ресурсы нод.** `kubectl describe nodes`.

**👍 СТОИТ:**

4. **`kubectl get events`** для общей картины.
5. **Мониторинг нод** (Prometheus, Grafana).

**❌ НЕ ДЕЛАЙ:**

6. **Не перезапускай Pod'ы без понимания.** Не поможет.
7. **Не игнорируй Pending.** Это не «подожди», это «исправь».

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **kube-scheduler** | Компонент, назначающий Pod'ы на ноды. |
| **Filtering** | Фаза scheduler: отсеять неподходящие ноды. |
| **Scoring** | Фаза scheduler: ранжировать подходящие ноды. |
| **Requests** | Гарантированные ресурсы Pod'а. Scheduler учитывает при планировании. |
| **Limits** | Максимальные ресурсы Pod'а. |
| **Throttling** | Ограничение CPU при превышении limits. |
| **OOM Killer** | Убийца процессов при превышении memory limits. |
| **Overcommit** | Ситуация, когда сумма requests > ёмкость ноды. |
| **QoS** | Quality of Service: Guaranteed, Burstable, BestEffort. |
| **oom_score_adj** | Приоритет убийства при OOM. |
| **nodeSelector** | Простой способ выбрать ноду по labels. |
| **nodeAffinity** | Гибкий способ выбора нод (required/preferred). |
| **podAffinity** | Разместить Pod рядом с другими Pod'ами. |
| **podAntiAffinity** | Разместить Pod далеко от других Pod'ов. |
| **topologyKey** | Уровень топологии (hostname, zone). |
| **Taint** | «Пятно» на ноде: не размещай Pod'ы без toleration. |
| **Toleration** | «Прощение»: Pod может запускаться на ноде с taint. |
| **NoSchedule** | Effect: не размещать новые Pod'ы. |
| **NoExecute** | Effect: выгнать существующие Pod'ы. |
| **PreferNoSchedule** | Effect: предпочитать не размещать. |
| **PriorityClass** | Класс приоритета Pod'а. |
| **Preemption** | Выгон низкоприоритетных Pod'ов для высокоприоритетных. |
| **topologySpreadConstraints** | Равномерное распределение по топологии. |
| **maxSkew** | Максимальная неравномерность. |
| **whenUnsatisfiable** | DoNotSchedule или ScheduleAnyway. |

---

## Что мы узнали?

- **kube-scheduler** работает в две фазы: filtering (подходящие ноды) и scoring (лучшая нода).
- **Requests** — гарантированные ресурсы, scheduler учитывает. **Limits** — максимальные.
- **QoS-классы:** Guaranteed (requests=limits), Burstable (requests≠limits), BestEffort (без resources). Влияют на порядок убийства при OOM.
- **nodeSelector** — простой выбор нод по labels. **nodeAffinity** — гибкий (required/preferred, операторы).
- **podAffinity** — разместить рядом. **podAntiAffinity** — разместить далеко.
- **Taints и tolerations** — выделение нод под конкретные Pod'ы.
- **PriorityClass и preemption** — приоритеты и выгон низкоприоритетных.
- **topologySpreadConstraints** — равномерное распределение с maxSkew.
- **Диагностика Pending:** `kubectl describe pod`, читать Events.

---

## Типичные ошибки

- ❌ **Не указывать requests и limits.** Pod в BestEffort QoS, убивается первым.
- ❌ **Overcommit в production.** OOM Killer будет убивать Pod'ы.
- ❌ **Ставить limits = requests для всего.** Guaranteed QoS, но неоптимально.
- ❌ **Использовать nodeSelector для anti-affinity.** Для этого есть podAntiAffinity.
- ❌ **Использовать `required` podAntiAffinity для реплик > нод.** Часть Pod'ов в Pending.
- ❌ **Не использовать taints для выделенных нод.** Другие Pod'ы будут там запускаться.
- ❌ **Использовать `NoExecute` без tolerationSeconds.** Pod'ы выгоняются сразу.
- ❌ **Не использовать PriorityClass для критичных сервисов.**
- ❌ **Preemption для batch-задач.** Выгонит всё.
- ❌ **Игнорировать Pending.** Не «подожди», а «исправь».
- ❌ **Не читать Events при Pending.** Там вся информация.

---

## Для быстрого повторения

- **Scheduler:** filtering (NodeResourcesFit, NodeAffinity, TaintToleration) + scoring (LeastAllocated, BalancedAllocation, ImageLocality).
- **Requests:** гарантированные. **Limits:** максимальные.
- **CPU:** сжимаемый (throttling). **Memory:** несжимаемый (OOM).
- **QoS:** Guaranteed (requests=limits, oom_score_adj=-997), Burstable (средний), BestEffort (oom_score_adj=1000).
- **nodeSelector:** `key=value`. **nodeAffinity:** `matchExpressions` с операторами.
- **podAntiAffinity:** `topologyKey: kubernetes.io/hostname` для разных нод.
- **Taints:** `kubectl taint nodes node-1 dedicated=database:NoSchedule`.
- **Tolerations:** в Pod'е, `key`, `operator`, `value`, `effect`.
- **PriorityClass:** `value: 100000`. **Preemption:** высокий выгоняет низких.
- **TopologySpread:** `maxSkew`, `topologyKey`, `whenUnsatisfiable`.

---

## Вопросы для самопроверки

1. Как работает scheduler? Какие две фазы?
2. Чем requests отличаются от limits?
3. Что такое CPU throttling? Когда происходит?
4. Что такое OOM Killer? Когда убивает?
5. Три QoS-класса — какие условия, что дают?
6. Что такое oom_score_adj? Как связан с QoS?
7. Чем nodeSelector отличается от nodeAffinity?
8. Что такое required vs preferred в nodeAffinity?
9. Чем podAffinity отличается от podAntiAffinity?
10. Что такое taints и tolerations? Когда использовать?
11. Что такое PriorityClass? Как работает preemption?
12. Что такое topologySpreadConstraints? Чем отличается от podAntiAffinity?
13. Pod в Pending. Как диагностировать?
14. Что означает `Insufficient cpu`? Как исправить?
15. Что означает `node(s) had untolerated taint`? Как исправить?

---

## Ответы

**1. Scheduler**

Две фазы: **Filtering** (какие ноды подходят) — NodeResourcesFit, NodeAffinity, TaintToleration. **Scoring** (какая лучшая) — LeastAllocated, BalancedAllocation, ImageLocality.

**2. Requests vs limits**

Requests — гарантированные ресурсы, scheduler учитывает при планировании. Limits — максимальные, при превышении CPU throttling, memory OOM.

**3. CPU throttling**

Когда Pod использует больше `limits.cpu`, CFS ограничивает его. Процесс не убивается, но замедляется. Латентность растёт.

**4. OOM Killer**

Когда Pod использует больше `limits.memory`, ядро убивает процесс. Kubernetes перезапускает. Через 3 перезапуска — CrashLoopBackOff.

**5. QoS-классы**

- **Guaranteed** — все контейнеры имеют requests=limits. oom_score_adj=-997. Убивается последним.
- **Burstable** — requests≠limits. Средний приоритет.
- **BestEffort** — без resources. oom_score_adj=1000. Убивается первым.

**6. oom_score_adj**

Число от -1000 до 1000. Больше = вероятнее убийство. Guaranteed = -997, BestEffort = 1000, Burstable — зависит от requests/limits.

**7. nodeSelector vs nodeAffinity**

nodeSelector — только `key=value`. nodeAffinity — операторы In, NotIn, Exists, DoesNotExist, Gt, Lt. Плюс required/preferred.

**8. Required vs preferred**

Required — жёсткое условие. Pod не запустится без него. Preferred — мягкое, scheduler предпочтёт, но не обязательно.

**9. podAffinity vs podAntiAffinity**

podAffinity — разместить Pod **рядом** с другими (на той же ноде/зоне). podAntiAffinity — разместить **далеко** (на разных нодах/зонах).

**10. Taints и tolerations**

Taint — «пятно» на ноде: не размещай Pod'ы без toleration. Toleration — «прощение»: Pod может запускаться на ноде с taint. Для выделенных нод.

**11. PriorityClass и preemption**

PriorityClass — числовой приоритет Pod'а. Preemption — если высокоприоритетный Pod не помещается, scheduler выгоняет низкоприоритетные.

**12. topologySpreadConstraints**

Равномерное распределение по топологии с maxSkew. Не блокирует запуск, как podAntiAffinity required. Позволяет распределить 6 реплик на 3 ноды как 2-2-2.

**13. Pod в Pending**

1. `kubectl get pods` — статус.
2. `kubectl describe pod` — Events.
3. Читать сообщение (`Insufficient cpu`, `didn't match node selector`, `untolerated taint`).
4. Проверить ресурсы нод (`kubectl describe nodes`).
5. Исправить причину.

**14. Insufficient cpu**

На нодах недостаточно свободных CPU. Решения: уменьшить `requests.cpu`, добавить ноды, удалить другие Pod'ы.

**15. Untolerated taint**

Pod не имеет toleration для taint ноды. Решения: добавить toleration в Pod, удалить taint с ноды.

---

## Куда идти дальше?

Мы разобрали планирование и ресурсы. Теперь ты знаешь:

- Как scheduler выбирает ноду.
- Как выбирать requests и limits.
- Как управлять размещением через affinity, taints, priority.
- Как диагностировать Pending.

Но мы пока не разобрали:

- **Как работает сеть в K8s.** CNI, Network Policies, Ingress.
- **Как хранить данные.** PV, PVC, StatefulSets.
- **Как передавать конфигурацию.** ConfigMaps, Secrets.

Следующие главы — про это.

**Глава 10: Kubernetes — сетевое взаимодействие.** Погнали. 🚀