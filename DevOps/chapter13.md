# 🔄 Глава 13: Kubernetes — обновления и деплой-стратегии

**Что вы узнаете:**
- Как работает Rolling Update в Deployment изнутри.
- Что такое Recreate, RollingUpdate, Blue-Green и Canary — плюсы и минусы каждой.
- Что такое `maxSurge` и `maxUnavailable` и как они влияют на деплой.
- Какую роль играют readiness и liveness probes в деплое.
- Что такое lifecycle hooks: preStop и postStart.
- Как работает `kubectl rollout` для управления деплоем.
- Что такое Argo Rollouts и Flagger для продвинутых стратегий.
- Как делать автоматический rollback по метрикам.
- Как использовать feature flags вместо деплоя.
- Как откатывать деплой при инциденте.

**После прочтения вы сможете:**
- Выбрать правильную стратегию деплоя для конкретного сервиса.
- Настроить rolling update с zero-downtime.
- Реализовать canary-деплой через Argo Rollouts.
- Настроить автоматический откат по метрикам.
- Использовать `kubectl rollout undo` для откатов.
- Диагностировать проблемы с деплоем.

---

## Содержание

- [13.0 Пролог: деплой, который уронил прод](#130-пролог-деплой-который-уронил-прод)
- [13.1 Rollout и Rollback в Deployment](#131-rollout-и-rollback-в-deployment)
- [13.2 Rolling Update изнутри](#132-rolling-update-изнутри)
- [13.3 Recreate: остановить и запустить](#133-recreate-остановить-и-запустить)
- [13.4 Blue-Green Deployment](#134-blue-green-deployment)
- [13.5 Canary Deployment](#135-canary-deployment)
- [13.6 Lifecycle Hooks: preStop и postStart](#136-lifecycle-hooks-prestop-и-poststart)
- [13.7 kubectl rollout: команды управления](#137-kubectl-rollout-команды-управления)
- [13.8 Argo Rollouts](#138-argo-rollouts)
- [13.9 Flagger: автоматический canary](#139-flagger-автоматический-canary)
- [13.10 Feature Flags: деплой без деплоя](#1310-feature-flags-деплой-без-деплоя)
- [13.11 Диагностика деплоя](#1311-диагностика-деплоя)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 13.0 Пролог: деплой, который уронил прод

Пятница, 18:00. Ты деплоишь новую версию `myapp:v1.5.0`. CI/CD отработал:

```bash
kubectl apply -f deployment.yaml
# deployment.apps/myapp configured

kubectl rollout status deployment/myapp
# Waiting for deployment "myapp" rollout to finish:
# 1 out of 3 new replicas have been updated...
# 2 out of 3 new replicas have been updated...
# 3 out of 3 new replicas have been updated...
# deployment "myapp" successfully rolled out
```

**Rollout успешен.** Все Pod'ы обновлены. Ты закрываешь ноутбук.

Через 10 минут — алерт:

```
[CRITICAL] High error rate on myapp
Error rate: 45% (SLO: < 0.1%)
```

**Прод сломан.** Но почему? Rollout успешен.

Ты смотришь логи:

```
Error: database migration failed - column "new_field" does not exist
```

**v1.5.0 требует колонку `new_field`, которой нет.** Миграция БД не выполнилась. Или выполнилась после деплоя. Или не применилась к базе.

**Проблема:** деплой кода прошёл, а миграция БД — нет. Или наоборот.

**Что можно было сделать:**

1. **Canary-деплой.** Выкатить 10% трафика, посмотреть на error rate. Не выкатывать всё сразу.
2. **Automated rollback.** Если error rate > 5% в течение 2 минут — откатить.
3. **Migration перед деплоем.** Отдельный Job, который выполняется до обновления Deployment.
4. **Backward-compatible migrations.** Схема БД должна поддерживать обе версии.

**В этой главе** мы разберём деплой-стратегии Kubernetes. Как безопасно выкатывать новые версии.

Это — **основа production**. Без правильной стратегии каждый деплой = риск инцидента.

---

## 13.1 Rollout и Rollback в Deployment

### 🔌 Проблема: как обновлять Pod'ы

Ты хочешь обновить версию образа в Deployment. Как Kubernetes это делает? И что произойдёт, если новая версия сломана?

**Ответ:** контроллер Deployment управляет двумя ReplicaSet'ами одновременно и хранит историю для откатов.

### 📊 Что происходит при обновлении

Когда ты меняешь `spec.template` в Deployment:

1. **Kubernetes создаёт новый ReplicaSet** с новым Pod template.
2. **Начинает масштабировать** новый ReplicaSet.
3. **Одновременно уменьшает** старый ReplicaSet.
4. **Старый ReplicaSet остаётся** в истории (для rollback).

**Ключевой момент:** Deployment **не пересоздаёт** Pod'ы напрямую. Он управляет двумя ReplicaSet'ами одновременно.

```
БЫЛО:
┌────────────────────────┐
│ Deployment: myapp       │
│ replicas: 3             │
└───────────┬────────────┘
            │ управляет
            ▼
┌────────────────────────┐
│ ReplicaSet: myapp-OLD   │
│ replicas: 3             │
│ image: myapp:1.4.0      │
└───────────┬────────────┘
            │ создаёт
            ▼
       Pod, Pod, Pod

СТАЛО (во время rollout):
┌────────────────────────┐
│ Deployment: myapp       │
│ replicas: 3             │
└─────┬──────────────┬───┘
      │              │
      ▼              ▼
┌───────────┐  ┌───────────┐
│ RS: OLD   │  │ RS: NEW   │
│ replicas:2│  │ replicas:1│
│ image:1.4 │  │ image:1.5 │
└───────────┘  └───────────┘
```

### 🎯 Пример

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
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
          image: myapp:1.4.0
```

**После `kubectl set image deployment/myapp myapp=myapp:1.5.0`:**

```bash
kubectl get replicaset
# NAME              DESIRED   CURRENT   READY   AGE
# myapp-7d9f8c6b4d  0         0         0       5d
# myapp-8a1b2c3d4e  3         3         3       10s
```

**Что произошло:**

1. **Старый ReplicaSet** `7d9f8c6b4d` — масштаб 0.
2. **Новый ReplicaSet** `8a1b2c3d4e` — масштаб 3.
3. **Deployment** следит за обоими.

### 🎯 История ревизий

```bash
kubectl rollout history deployment/myapp
# REVISION  CHANGE-CAUSE
# 1         kubectl apply --filename=deployment.yaml
# 2         kubectl set image deployment/myapp myapp=myapp:1.4.0
# 3         kubectl set image deployment/myapp myapp=myapp:1.5.0
```

**Что хранится:**

- **ReplicaSet** для каждой ревизии.
- **CHANGE-CAUSE** — аннотация `kubernetes.io/change-cause`.

**Добавить причину:**

```bash
kubectl annotate deployment/myapp kubernetes.io/change-cause="Upgrade to v1.5.0"
```

**Зачем:** через месяц ты не вспомнишь, зачем деплоил revision 7. Аннотация сохранит контекст.

### 🎯 Rollback

```bash
# Откат на предыдущую версию
kubectl rollout undo deployment/myapp

# Откат на конкретную ревизию
kubectl rollout undo deployment/myapp --to-revision=2

# Проверить
kubectl rollout history deployment/myapp
```

**Что происходит:**

1. Kubernetes **меняет местами** ReplicaSet'ы.
2. Старый ReplicaSet масштабируется до 3.
3. Новый ReplicaSet масштабируется до 0.

**Важно:** rollback **не пересоздаёт** Pod'ы с нуля. Kubernetes использует **существующий** ReplicaSet, в котором уже есть Pod template старой версии. Pod'ы старой версии создаются быстро — образ уже скачан на нодах.

### 🎯 revisionHistoryLimit

```yaml
spec:
  revisionHistoryLimit: 10
```

**Что даёт:**

- **10 ReplicaSet'ов** хранятся.
- Старые удаляются.

**По умолчанию:** 10.

**Если 0:** нельзя откатиться (история удаляется).

**Рекомендация:** 5-10. Больше — лишние ReplicaSet'ы занимают место.

### 🔬 Практика: rollout

```bash
# 1. Deployment
kubectl create deployment myapp --image=nginx:1.24 --replicas=3

# 2. Проверить
kubectl get rs
# NAME              DESIRED   CURRENT   READY
# myapp-xxx         3         3         3

# 3. Обновить
kubectl set image deployment/myapp nginx=nginx:1.25

# 4. Смотреть
kubectl rollout status deployment/myapp

# 5. История
kubectl rollout history deployment/myapp

# 6. ReplicaSet
kubectl get rs
# NAME              DESIRED   CURRENT   READY
# myapp-aaa         0         0         0     ← старый
# myapp-bbb         3         3         3     ← новый

# 7. Откатить
kubectl rollout undo deployment/myapp

# 8. Проверить
kubectl get rs
# myapp-aaa         3         3         3
# myapp-bbb         0         0         0
```

### 💡 Практика: как правильно использовать rollout

**✅ ОБЯЗАТЕЛЬНО:**

1. **`revisionHistoryLimit: 5-10`** для откатов.
2. **`kubernetes.io/change-cause`** для истории.
3. **Проверять `rollout history`** перед откатом.

**👍 СТОИТ:**

4. **`kubectl rollout undo --to-revision`** для конкретной версии.
5. **Мониторинг деплоя.**
6. **Автоматический откат.**

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `revisionHistoryLimit: 0`** без понимания.
8. **Не забывай про Pod'ы старой версии.**
9. **Не откатывай без диагностики.**

### Где мы сейчас

Мы разобрали rollout и rollback. Теперь — **как работает rolling update изнутри** — пошагово, с точными параметрами.

---

## 13.2 Rolling Update изнутри

### 🔌 Проблема: как работает постепенная замена

Rolling Update заменяет Pod'ы **постепенно**. Но что значит «постепенно»? Сколько Pod'ов можно создать? Сколько удалить? В каком порядке?

**Ответ:** через `maxSurge` и `maxUnavailable`.

### 📊 Параметры

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

| Параметр | Что означает | Default |
|:---|:---|:---|
| **maxSurge** | Максимум Pod'ов **сверх** `replicas` | 25% |
| **maxUnavailable** | Максимум **недоступных** Pod'ов | 25% |

**Пример с `replicas: 4`:**

- `maxSurge: 1` — максимум 5 Pod'ов (4 + 1).
- `maxUnavailable: 0` — минимум 4 доступных.

**Всего может быть от 4 до 5 Pod'ов.**

### 🎯 Алгоритм

**Пример: replicas: 4, maxSurge: 1, maxUnavailable: 0.**

**Шаг 1:** Все 4 Pod'а старой версии.

```
Старые: ████ (4)
Новые:  (0)
Всего:  4
```

**Шаг 2:** Создать 1 новый Pod.

```
Старые: ████ (4)
Новые:  ░ (1, создаётся)
Всего:  5 (maxSurge=1)
```

**Шаг 3:** Дождаться, пока новый Pod станет Ready.

```
Старые: ████ (4)
Новые:  ▓ (1, Ready)
Всего:  5
```

**Шаг 4:** Удалить 1 старый Pod.

```
Старые: ███ (3)
Новые:  ▓ (1)
Всего:  4
```

**Шаг 5:** Создать 1 новый Pod.

```
Старые: ███ (3)
Новые:  ▓░ (2)
Всего:  5
```

**Шаг 6:** Дождаться Ready.

**Шаг 7:** Удалить 1 старый Pod.

```
Старые: ██ (2)
Новые:  ▓▓ (2)
Всего:  4
```

**... и так далее, пока все Pod'ы не станут новыми.**

**Итого:** 4 шага, 8 операций (4 создания, 4 удаления).

### 🎯 Zero-downtime

**Для zero-downtime:**

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # создавать новые
    maxUnavailable: 0  # не удалять пока новые не готовы
```

**Что даёт:** всегда `replicas` Pod'ов Ready. Никогда не теряем ёмкость.

**Цена:** нужно место для `maxSurge` Pod'ов. Если нода заполнена — новый Pod не поместится и rollout застрянет.

### 🎯 Readiness probe

**Критически важно:**

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

**Что даёт:**

1. Новый Pod создан.
2. `readinessProbe` проверяет `/ready`.
3. **Пока не Ready** — старый Pod **не удаляется**.
4. **Когда Ready** — старый Pod удаляется.

**Без readiness:**

- Pod создан → сразу Ready → старый удалён.
- **Даже если приложение не готово**.
- **Errors** в трафике.

**Правило:** если ты используешь rolling update без readiness probe — ты гарантированно теряешь запросы при каждом деплое.

### 🎯 minReadySeconds

```yaml
spec:
  minReadySeconds: 10
```

**Что даёт:**

- Pod становится Ready.
- Ждём **10 секунд**.
- Только потом считаем «стабильным».
- Продолжаем rollout.

**Зачем:** если приложение падает через 5 секунд после старта (например, из-за ошибки в конфиге) — Kubernetes заметит и не будет продолжать rollout.

**Рекомендация:** 10-30 секунд для production.

### 🎯 progressDeadlineSeconds

```yaml
spec:
  progressDeadlineSeconds: 600
```

**Что даёт:**

- Если rollout не завершился за 10 минут — Kubernetes помечает как **Failed**.
- Deployment **не откатывается** автоматически.
- Нужно вручную.

**По умолчанию:** 600 секунд (10 минут).

### 🎯 Пример полного Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    kubernetes.io/change-cause: "Upgrade to v1.5.0"
spec:
  replicas: 4
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: myapp
          image: myapp:1.5.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 2
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
```

### 🎯 Что происходит при rollout

1. **Новый ReplicaSet** создаётся с новым template.
2. **Создаётся 1 новый Pod** (maxSurge).
3. **Readiness probe** проверяет `/ready`.
4. **Через 5 секунд** — Ready.
5. **minReadySeconds: 10** — ждём 10 секунд.
6. **Удаляется 1 старый Pod**:
   - **preStop** — sleep 10.
   - SIGTERM.
   - Graceful shutdown (60 секунд max).
7. **Повторяется** для остальных Pod'ов.

**Итого:** 4 Pod'а × (5 + 10 + 10 + shutdown) ≈ 2-3 минуты.

### 🎯 Что если что-то идёт не так

**Сценарий 1: новый Pod не становится Ready.**

- `kubectl rollout status` показывает «Waiting...»
- Через `progressDeadlineSeconds` (10 мин) — Failed.
- **Старые Pod'ы продолжают работать.** Rollout застрял.

**Сценарий 2: новый Pod крашится.**

- CrashLoopBackOff.
- Rollout застревает.
- Старые Pod'ы работают.
- **Действие:** `kubectl rollout undo`.

**Сценарий 3: приложение работает, но медленное.**

- Pod Ready.
- Rollout завершён.
- Но latency выросла в 5 раз.
- **Действие:** откат по метрикам (Argo Rollouts, Flagger).

### 🔬 Практика: rolling update

```bash
# 1. Deployment с 4 репликами
cat > deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
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
          image: nginx:1.24
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 2
EOF
kubectl apply -f deployment.yaml

# 2. Смотреть Pod'ы
kubectl get pods -w

# 3. В другом терминале — обновление
kubectl set image deployment/myapp nginx=nginx:1.25
kubectl rollout status deployment/myapp

# 4. Смотреть ReplicaSet
kubectl get rs -w

# 5. Проверить
kubectl get pods
# 4 Pod'а nginx:1.25
```

### 💡 Практика: как правильно настраивать rolling update

**✅ ОБЯЗАТЕЛЬНО:**

1. **`maxUnavailable: 0`** для zero-downtime.
2. **`maxSurge: 1`** для постепенности.
3. **`minReadySeconds: 10-30`** для стабилизации.
4. **Readiness probe** обязательно.

**👍 СТОИТ:**

4. **`progressDeadlineSeconds`** разумный.
5. **`revisionHistoryLimit: 10`.**
6. **`preStop: sleep`** для race condition.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `maxUnavailable: 100%`** в production.
8. **Не забывай про readiness.**
9. **Не ставь `minReadySeconds: 0`** без понимания.

### Где мы сейчас

Мы разобрали rolling update. Теперь — **Recreate** — простейшая стратегия, которую иногда нельзя избежать.

---

## 13.3 Recreate: остановить и запустить

### 🔌 Проблема: Rolling Update не всегда подходит

Rolling update требует, чтобы **обе версии работали одновременно**. Но что если:

- **v2 несовместима** с v1 (например, схема БД).
- **Две версии не могут** работать одновременно (конфликт ресурсов).
- **Ресурсы ограничены** (maxSurge не помещается).
- **Приложение использует singleton-ресурс** (например, лидер-лок).

**Решение:** Recreate.

### 📊 Что такое Recreate

**Recreate** — остановить **все** Pod'ы старой версии, потом запустить все новые.

```yaml
spec:
  strategy:
    type: Recreate
```

**Что происходит:**

1. Все старые Pod'ы удаляются.
2. Ждём, пока все завершатся.
3. Создаются новые Pod'ы.

**Downtime** — есть. Длительность = время остановки старых + время старта новых.

```
БЫЛО:       СТАЛО:
Pod v1.0    Pod v2.0
Pod v1.0    Pod v2.0
Pod v1.0    Pod v2.0

Между ними: 0 Pod'ов (downtime)
```

### 🎯 Плюсы и минусы

**Плюсы:**

- **Просто.** Не нужны readiness probes.
- **Чисто.** Нет смеси версий.
- **Для несовместимых версий.** v1 и v2 не работают одновременно.

**Минусы:**

- **Downtime.** 30 секунд — 5 минут.
- **Не для production** с SLO.
- **Не для web-сервисов** с пользователями.

### 🎯 Когда использовать

**✅ Использовать:**

- **Dev-окружения.**
- **Batch-задачи** (один Pod, который завершается).
- **Миграции схемы БД**, где v1 и v2 несовместимы.
- **Singleton-сервисы** (1 реплика, leader election).

**❌ Не использовать:**

- **Production web-сервисы.**
- **Сервисы с SLO.**
- **Multi-replica deployments** с трафиком.

### 🎯 Пример

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: migration-worker
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: migration-worker
  template:
    metadata:
      labels:
        app: migration-worker
    spec:
      containers:
        - name: worker
          image: migration-worker:2.0.0
          env:
            - name: DB_SCHEMA_VERSION
              value: "2.0"
```

**Что даёт:**

- Worker работает с конкретной схемой БД.
- Две версии несовместимы (v1 читает `field_a`, v2 читает `field_b`).
- Recreate гарантирует, что работает только одна.

### 🎯 Комбинация с миграцией БД

**Сценарий:** приложение требует новую схему БД, но schema migration несовместима со старой версией приложения.

**Паттерн:**

1. **Job для миграции** — выполняется один раз, обновляет схему.
2. **Deployment с Recreate** — старая версия останавливается, новая запускается.

**Важно:** миграция **должна** завершиться до старта нового Deployment. Иначе — ошибки.

```yaml
# Шаг 1: Job миграции
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: myapp:2.0.0
          command: ["./myapp", "migrate", "up"]
---
# Шаг 2: Deployment с Recreate
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: Recreate
  # ...
```

**Правило:** используй Helm hooks (`pre-upgrade`) или отдельный CI-этап для управления порядком.

### 🎯 Recreate и PDB

**Проблема:** Recreate удаляет все Pod'ы одновременно. **PDB** не работает, потому что PDB защищает от **частичного** удаления, а Recreate удаляет **все**.

**Вывод:** Recreate **несовместим** с PDB по определению. Если тебе нужен PDB — используй RollingUpdate.

### 🎯 Recreate и terminationGracePeriodSeconds

**Важно:** Recreate ждёт полного завершения Pod'ов. Если `terminationGracePeriodSeconds` = 60, downtime ≥ 60 секунд (плюс время старта).

```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30  # меньше = меньше downtime
```

### 🔬 Практика: Recreate

```bash
# 1. Deployment с Recreate
cat > deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: Recreate
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
          image: nginx:1.24
EOF
kubectl apply -f deployment.yaml

# 2. Обновить
kubectl set image deployment/myapp nginx=nginx:1.25

# 3. Смотреть — все Pod'ы удаляются сразу
kubectl get pods -w
# myapp-xxx  Terminating
# myapp-yyy  Terminating
# myapp-zzz  Terminating
# (все удалены)
# myapp-aaa  Pending
# myapp-bbb  Pending
# myapp-ccc  Pending
# myapp-aaa  Running
# ...

# 4. Downtime — есть
```

### 💡 Практика: как правильно использовать Recreate

**✅ ОБЯЗАТЕЛЬНО:**

1. **Только для dev** или несовместимых версий.
2. **Не для web-сервисов** в production.
3. **Для singleton** сервисов.

**👍 СТОИТ:**

4. **Отдельный Job** для миграций.
5. **Backward-compatible migrations** где возможно.
6. **Тестировать downtime.**

**❌ НЕ ДЕЛАЙ:**

7. **Не используй Recreate в production** web.
8. **Не забывай про downtime.**
9. **Не используй с PDB.**

### Где мы сейчас

Мы разобрали Recreate. Теперь — **Blue-Green** — стратегия с двумя окружениями и мгновенным откатом.

---

## 13.4 Blue-Green Deployment

### 🔌 Проблема: Rolling Update не даёт мгновенного отката

Rolling Update заменяет Pod'ы постепенно. Откат тоже постепенный. **Не мгновенный.**

**Если новая версия сломана** — откат занимает минуты. За это время — ошибки у пользователей.

**Решение:** Blue-Green.

### 📊 Что такое Blue-Green

**Blue-Green** — два окружения рядом:

- **Blue** — текущая версия (production).
- **Green** — новая версия (staging).

**Cutover** — переключение трафика с Blue на Green. Мгновенное.

### 🎯 Архитектура

```
ДО CUTOVER:

Service (selector: version=blue)
    │
    ├── Blue: myapp:v1.4.0 (3 Pod'а) ← трафик
    │
    └── Green: myapp:v1.5.0 (3 Pod'а) ← не получает

ПОСЛЕ CUTOVER:

Service (selector: version=green)
    │
    ├── Blue: myapp:v1.4.0 (3 Pod'а) ← не получает
    │
    └── Green: myapp:v1.5.0 (3 Pod'а) ← трафик
```

### 🎯 Реализация

**1. Два Deployment'а:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
        - name: myapp
          image: myapp:1.4.0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
        - name: myapp
          image: myapp:1.5.0
```

**2. Service с selector:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue    # ← переключение
  ports:
    - port: 80
      targetPort: 8080
```

**3. Cutover:**

```bash
# Переключить на green
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

# Откат
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'
```

### 🎯 Плюсы и минусы

**Плюсы:**

- **Мгновенный откат.** Переключение Service — секунды.
- **Тестирование** green перед cutover.
- **Zero-downtime.**

**Минусы:**

- **Двойные ресурсы.** 2× Pod'ы в момент cutover.
- **Сложнее.** Два Deployment'а, два CI-пайплайна.
- **БД.** Если схема несовместима — проблема.

### 🎯 Cutover стратегии

**1. Полный cutover.**

Все трафики сразу.

```
Service selector: blue → green
```

**Мгновенно.** Просто. Хорошо для небольших сервисов.

**2. Постепенный cutover.**

Через Istio/Service Mesh:

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
            subset: blue
          weight: 90
        - destination:
            host: myapp
            subset: green
          weight: 10
```

**Что даёт:** 10% на green. Постепенно.

**Плюс:** можно мониторить green на реальном трафике.

**Минус:** это уже не чистый Blue-Green, а Canary.

### 🎯 БД — главная проблема

**Проблема:** Blue и Green используют **одну** БД. Схема должна быть **совместима** с обеими версиями.

**Что может пойти не так:**

- v1.5.0 удалила колонку `old_field`, которую использует v1.4.0.
- При cutover → green работает, blue падает.
- При откате → blue работает, green падает.

**Решение: Expand-Contract.**

1. **Expand:** добавить новое поле/таблицу. Обе версии работают.
2. **Migrate:** использовать новое. Обе версии работают.
3. **Contract:** удалить старое (через релиз-два, когда blue точно не используется).

**Правило:** миграция БД должна быть **backward-compatible** на 1 релиз.

### 🎯 Автоматизация через Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    blueGreen:
      activeService: myapp-active
      previewService: myapp-preview
      autoPromotionEnabled: false
```

**Что даёт:**

- **`myapp-active`** — production (blue).
- **`myapp-preview`** — preview (green).
- **Ручной cutover** через Argo Rollouts UI.

**Разберём в подглаве 13.8.**

### 🔬 Практика: Blue-Green

```bash
# 1. Blue Deployment
kubectl create deployment myapp-blue --image=nginx:1.24 --replicas=3
kubectl label deployment myapp-blue version=blue --overwrite

# 2. Green Deployment
kubectl create deployment myapp-green --image=nginx:1.25 --replicas=3
kubectl label deployment myapp-green version=green --overwrite

# 3. Service
cat > service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    version: blue
  ports:
    - port: 80
      targetPort: 80
EOF
kubectl apply -f service.yaml

# 4. Проверить
kubectl get pods -l version=blue
kubectl get pods -l version=green

# 5. Cutover
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

# 6. Проверить
kubectl get endpoints myapp

# 7. Откат
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'
```

### 💡 Практика: как правильно делать Blue-Green

**✅ ОБЯЗАТЕЛЬНО:**

1. **Два Deployment'а** с разными labels.
2. **Service с selector** на active.
3. **Backward-compatible migrations** БД.
4. **Тестирование green** перед cutover.

**👍 СТОИТ:**

4. **Argo Rollouts** для автоматизации.
5. **Preview Service** для тестирования.
6. **Постепенный cutover** через Istio.

**❌ НЕ ДЕЛАЙ:**

7. **Не забывай про БД.**
8. **Не оставляй old надолго.** Удаляй blue после успешного деплоя.
9. **Не используй для маленьких сервисов.** Двойные ресурсы не оправданы.

### Где мы сейчас

Мы разобрали Blue-Green. Теперь — **Canary** — самая гибкая стратегия.

---

## 13.5 Canary Deployment

### 🔌 Проблема: Blue-Green переключает весь трафик

Blue-Green cutover — **весь трафик** сразу. Если баг в 10% запросов — узнаешь после cutover, когда прод сломан.

**Решение:** Canary.

### 📊 Что такое Canary

**Canary** — постепенное увеличение трафика на новую версию.

```
Шаг 1: 5% на canary
Шаг 2: 25%
Шаг 3: 50%
Шаг 4: 100%
```

**Мониторинг** на каждом шаге. Откат при проблемах.

### 🎯 Реализация через Deployment replicas

**Простейший способ** — два Deployment'а и Service без selector на version.

```
Stable Deployment: 9 реплик (90%)
Canary Deployment: 1 реплика (10%)

Service selector: app=myapp (выбирает оба)
```

**Что произойдёт:** kube-proxy распределит трафик пропорционально количеству Pod'ов: ~10% на canary.

**Проблемы:**

- **Грубая настройка.** 1 из 10 = 10%. 1 из 100 = 1%.
- **Не точный процент.** kube-proxy использует random.
- **Нет автоматизации.**

**Когда использовать:** для простых случаев, когда не важен точный процент.

### 🎯 Реализация через Istio VirtualService

**Более точный способ:**

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
            subset: stable
          weight: 95
        - destination:
            host: myapp
            subset: canary
          weight: 5
```

**Плюс DestinationRule с subsets:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  subsets:
    - name: stable
      labels:
        version: v1.4.0
    - name: canary
      labels:
        version: v1.5.0
```

**Что даёт:** точный контроль процента.

### 🎯 Реализация через Nginx Ingress

**Nginx Ingress Controller** поддерживает canary из коробки через аннотации:

```yaml
# Основной Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-stable
                port:
                  number: 80
---
# Canary Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-canary
                port:
                  number: 80
```

**Что даёт:** 10% трафика на canary.

**Менять вес:**

```bash
kubectl annotate ingress myapp-canary nginx.ingress.kubernetes.io/canary-weight=50 --overwrite
```

### 🎯 Прогрессивный canary

**Идеальный сценарий:**

1. **1%** — 10 минут.
2. **5%** — 10 минут.
3. **25%** — 10 минут.
4. **50%** — 10 минут.
5. **100%** — полный rollout.

**Мониторинг между шагами:**

- **Error rate** canary vs stable.
- **Latency** p99 canary vs stable.
- **CPU/memory.**
- **Business metrics.**

**Автоматический откат** при проблемах.

### 🎯 Автоматизация через Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 5
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 5m}
        - setWeight: 25
        - pause: {duration: 5m}
        - setWeight: 50
        - pause: {duration: 5m}
        - setWeight: 100
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 1
```

**Что даёт:** автоматический canary с паузами и анализом.

**Разберём в подглаве 13.8.**

### 🎯 Автоматизация через Flagger

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500
        interval: 1m
```

**Что даёт:** автоматический canary, увеличивает вес на 10% каждую минуту, откатывает при плохих метриках.

**Разберём в подглаве 13.9.**

### 🎯 Сравнение Canary и Blue-Green

| Аспект | Blue-Green | Canary |
|:---|:---|:---|
| **Трафик на новую версию** | 100% сразу | 1-5%, потом растёт |
| **Ресурсы** | 2x | 1.1x |
| **Откат** | Мгновенный | Мгновенный |
| **Тестирование на реальном трафике** | Нет | Да |
| **Сложность** | Средняя | Высокая |
| **Риск** | Средний | Минимальный |

**Когда что:**

- **Blue-Green** — если нужен мгновенный откат и есть ресурсы.
- **Canary** — если нужна проверка на реальном трафике.

### 🔬 Практика: Canary через replicas

```bash
# 1. Stable Deployment
kubectl create deployment myapp-stable --image=nginx:1.24 --replicas=9
kubectl label deployment myapp-stable version=stable --overwrite

# 2. Canary Deployment
kubectl create deployment myapp-canary --image=nginx:1.25 --replicas=1
kubectl label deployment myapp-canary version=canary --overwrite

# 3. Service
cat > service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 80
EOF
kubectl apply -f service.yaml

# 4. Проверить, что Service выбирает оба
kubectl get endpoints myapp
# 10 IP'ов (9 stable + 1 canary)

# 5. Увеличить canary до 3 (25%)
kubectl scale deployment myapp-canary --replicas=3

# 6. Полный rollout
kubectl scale deployment myapp-canary --replicas=9
kubectl scale deployment myapp-stable --replicas=0
```

### 💡 Практика: как правильно делать Canary

**✅ ОБЯЗАТЕЛЬНО:**

1. **Мониторинг** canary vs stable.
2. **Автоматический откат** при плохих метриках.
3. **Постепенно** увеличивать вес.
4. **Backward-compatible migrations.**

**👍 СТОИТ:**

4. **Argo Rollouts** или **Flagger** для автоматизации.
5. **Istio/Nginx** для точного контроля процента.
6. **Business metrics** для анализа.

**❌ НЕ ДЕЛАЙ:**

7. **Не начинай с 50%.** С 1-5%.
8. **Не оставляй canary навсегда.**
9. **Не игнорируй метрики.**

### Где мы сейчас

Мы разобрали Canary. Теперь — **Lifecycle Hooks** — preStop и postStart.

---

## 13.6 Lifecycle Hooks: preStop и postStart

### 🔌 Проблема: race condition при graceful shutdown

Pod получает SIGTERM. Начинает graceful shutdown. Но Kubernetes **ещё не убрал** Pod из Service endpoints.

**Что происходит:**

1. SIGTERM → Pod.
2. Pod начинает shutdown (перестаёт принимать запросы).
3. Kubernetes обновляет endpoints (убирает Pod).
4. **За это время** трафик ещё идёт на Pod.
5. Pod уже не принимает запросы → **ошибки**.

**Это классический race condition** в деплое Kubernetes.

**Решение:** preStop hook.

### 📊 Что такое lifecycle hooks

**Lifecycle hooks** — команды, выполняемые в определённые моменты жизненного цикла контейнера.

**Два типа:**

| Hook | Когда | Что делает |
|:---|:---|:---|
| **postStart** | После создания контейнера | Инициализация |
| **preStop** | Перед SIGTERM | Graceful shutdown |

### 🎯 postStart

```yaml
lifecycle:
  postStart:
    exec:
      command: ["/bin/sh", "-c", "echo 'Container started' >> /var/log/lifecycle.log"]
```

**Когда выполняется:** сразу после `START` контейнера, параллельно с `ENTRYPOINT`.

**Что использовать:**

- **Инициализация** (создать директории, скачать конфиги).
- **Регистрация** в service discovery.
- **Уведомление** (Slack, метрики).

**Осторожно:**

- **postStart блокирует** переход контейнера в статус Running.
- **Если postStart упал** — контейнер убивается и перезапускается.
- **Нет гарантии** порядка с ENTRYPOINT.

### 🎯 preStop

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
```

**Когда выполняется:** перед SIGTERM. **Синхронно** — kubelet ждёт завершения preStop, потом отправляет SIGTERM.

**Что использовать:**

- **`sleep`** для race condition — дать Kubernetes убрать Pod из endpoints.
- **Закрыть connections** (drain connections из load balancer).
- **Уведомить** downstream.

**Максимальное время:** ограничено `terminationGracePeriodSeconds`.

### 🎯 Почему sleep помогает

**Сценарий без preStop:**

```
t=0:   SIGTERM → Pod
t=0:   Pod начинает shutdown
t=0:   Kubernetes начинает обновлять endpoints
t=1:   Pod уже не принимает запросы
t=5:   Endpoints обновлены (Pod убран из Service)
       ← 5 секунд трафик шёл на мёртвый Pod → ошибки
```

**Сценарий с preStop sleep 10:**

```
t=0:   Pod получает preStop → sleep 10
t=0:   Kubernetes видит, что Pod не Ready (или terminating)
t=0:   Kubernetes начинает обновлять endpoints
t=5:   Endpoints обновлены (Pod убран из Service)
t=10:  preStop завершён → SIGTERM → Pod
t=10:  Pod начинает shutdown
       ← трафик уже не идёт на Pod, ошибок нет
```

**Задержка 10 секунд** даёт Kubernetes время обновить endpoints.

### 🎯 Размер sleep

**Рекомендация:** 5-15 секунд.

**Зависит от:**

- **Ingress/Service Mesh** — сколько времени на обновление endpoints.
- **Downstream** — сколько времени нужно клиентам, чтобы заметить.

**Слишком мало** (1-2 секунды) — race condition остаётся.

**Слишком много** (60 секунд) — долгий деплой.

### 🎯 Правильная комбинация

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: myapp
      image: myapp:1.5.0
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 10"]
```

**Плюс graceful shutdown в коде:**

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
    
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()
    
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
    <-quit
    
    ctx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
    defer cancel()
    
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal(err)
    }
}
```

**Тайминги:**

- **preStop:** 10 секунд (дать Kubernetes убрать endpoints).
- **Graceful shutdown:** 25 секунд.
- **terminationGracePeriodSeconds:** 60 секунд (запас 25 секунд).

### 🎯 postStart: пример

```yaml
lifecycle:
  postStart:
    exec:
      command:
        - /bin/sh
        - -c
        - |
          # Подождать, пока приложение готово
          until curl -sf localhost:8080/healthz; do
            sleep 1
          done
          # Зарегистрироваться в service discovery
          curl -X POST http://consul:8500/v1/agent/service/register \
            -d '{"name":"myapp","port":8080}'
```

**Что даёт:** регистрация в Consul после старта.

### 🎯 Для Jobs

**Проблема:** sidecar (Istio, Linkerd) не завершается. Job никогда не завершится.

**Решение:**

**Вариант 1:** отключить sidecar

```yaml
metadata:
  annotations:
    sidecar.istio.io/inject: "false"
```

**Вариант 2:** native sidecar (K8s 1.28+)

```yaml
initContainers:
  - name: istio-proxy
    restartPolicy: Always  # native sidecar
    image: istio/proxyv2:...
```

**Что даёт:** sidecar завершается вместе с основным контейнером.

### 🔬 Практика: lifecycle hooks

```bash
# 1. Deployment с preStop
cat > deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: myapp
          image: nginx:1.25
          readinessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 2
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
EOF
kubectl apply -f deployment.yaml

# 2. Тест: удалить Pod
kubectl delete pod myapp-xxx

# 3. Смотреть события
kubectl describe pod myapp-xxx
# Events:
#   Killing  Stopping container myapp
#   (через 10 секунд)

# 4. Проверить логи
kubectl logs myapp-xxx --previous
```

### 💡 Практика: как правильно использовать lifecycle hooks

**✅ ОБЯЗАТЕЛЬНО:**

1. **`preStop: sleep 10`** для race condition.
2. **Graceful shutdown** в коде.
3. **`terminationGracePeriodSeconds`** > preStop + shutdown.

**👍 СТОИТ:**

4. **postStart** для инициализации.
5. **Native sidecar** для Jobs.
6. **Тестирование** под нагрузкой.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `preStop` без graceful shutdown.**
8. **Не ставь sleep слишком маленький.**
9. **Не забывай про terminationGracePeriodSeconds.**

### Где мы сейчас

Мы разобрали lifecycle hooks. Теперь — **kubectl rollout** — практические команды.

---

## 13.7 kubectl rollout: команды управления

### 🔌 Проблема: как управлять деплоем

Деплой запущен. Как следить? Как откатить? Как поставить на паузу?

**Ответ:** `kubectl rollout`.

### 📊 Команды

| Команда | Что делает |
|:---|:---|
| `kubectl rollout status` | Статус rollout'а |
| `kubectl rollout history` | История ревизий |
| `kubectl rollout undo` | Откат |
| `kubectl rollout pause` | Пауза |
| `kubectl rollout resume` | Возобновление |
| `kubectl rollout restart` | Перезапуск |

### 🎯 rollout status

```bash
kubectl rollout status deployment/myapp
# Waiting for deployment "myapp" rollout to finish:
# 1 out of 3 new replicas have been updated...
# 2 out of 3 new replicas have been updated...
# 3 out of 3 new replicas have been updated...
# deployment "myapp" successfully rolled out
```

**Что показывает:**

- **Прогресс** rollout'а.
- **Ошибки** (если есть).
- **Финальный статус.**

**С timeout:**

```bash
kubectl rollout status deployment/myapp --timeout=5m
# Если rollout не завершится за 5 минут — ошибка
```

**Что даёт:** CI/CD может ждать завершения с таймаутом.

### 🎯 rollout history

```bash
kubectl rollout history deployment/myapp
# REVISION  CHANGE-CAUSE
# 1         kubectl apply --filename=deployment.yaml
# 2         kubectl set image deployment/myapp myapp=myapp:1.4.0
# 3         kubectl set image deployment/myapp myapp=myapp:1.5.0
```

**Детали конкретной ревизии:**

```bash
kubectl rollout history deployment/myapp --revision=2
# deployment.apps/myapp with revision #2
# Pod Template:
#   Labels:      app=myapp
#                pod-template-hash=7d9f8c6b4d
#   Containers:
#    myapp:
#     Image:      myapp:1.4.0
#     Port:       8080
```

### 🎯 rollout undo

```bash
# На предыдущую ревизию
kubectl rollout undo deployment/myapp

# На конкретную ревизию
kubectl rollout undo deployment/myapp --to-revision=2

# Dry-run
kubectl rollout undo deployment/myapp --dry-run=client
```

**Важно:** при rollback **создаётся новая ревизия** с тем же Pod template. История не перезаписывается.

**Пример:**

```
Ревизия 1: v1.0.0
Ревизия 2: v1.1.0
Ревизия 3: v1.2.0 (сломанная)

kubectl rollout undo deployment/myapp --to-revision=2
# Создаётся ревизия 4 = копия ревизии 2 (v1.1.0)
```

### 🎯 rollout pause и resume

```bash
# Поставить на паузу
kubectl rollout pause deployment/myapp

# Что-то изменить
kubectl set image deployment/myapp myapp=myapp:1.6.0
kubectl set resources deployment/myapp --limits=cpu=500m

# Возобновить
kubectl rollout resume deployment/myapp
```

**Что даёт:** позволяет внести несколько изменений и применить их **одновременно** одним rollout'ом.

**Когда использовать:**

- **Несколько изменений** в одном деплое.
- **Canary вручную** — пауза после первого Pod'а, проверка, resume.

### 🎯 rollout restart

```bash
kubectl rollout restart deployment/myapp
```

**Что даёт:** перезапускает все Pod'ы, даже если template не изменился.

**Когда использовать:**

- **Секреты обновились** — нужен restart, чтобы Pod'ы подхватили.
- **ConfigMap обновился** — то же.
- **Утечка памяти** — restart для очистки.

**Что происходит:** добавляется аннотация `kubectl.kubernetes.io/restartedAt`, template меняется, запускается новый rollout.

### 🎯 Практический пример: rollback при инциденте

**Сценарий:** деплой v1.5.0 сломал прод. Нужно срочно откатить.

```bash
# 1. Проверить статус
kubectl rollout status deployment/myapp

# 2. Посмотреть историю
kubectl rollout history deployment/myapp
# REVISION  CHANGE-CAUSE
# 3         Upgrade to v1.5.0  ← сломанная

# 3. Откатить на предыдущую
kubectl rollout undo deployment/myapp

# 4. Смотреть статус
kubectl rollout status deployment/myapp

# 5. Проверить, что старая версия работает
kubectl get pods
kubectl logs -l app=myapp --tail=20
```

**Время отката:** 30-60 секунд (в зависимости от graceful shutdown).

### 🎯 Для StatefulSet

```bash
kubectl rollout status statefulset/myapp
kubectl rollout history statefulset/myapp
kubectl rollout undo statefulset/myapp
```

**Что отличается:** StatefulSet обновляет Pod'ы по порядку (N-1, N-2, ..., 0), а не параллельно.

### 🎯 Для DaemonSet

```bash
kubectl rollout status daemonset/myapp
kubectl rollout undo daemonset/myapp
```

**Что отличается:** DaemonSet обновляет Pod'ы на каждой ноде.

### 🔬 Практика: rollout

```bash
# 1. Deployment
kubectl create deployment myapp --image=nginx:1.24 --replicas=3

# 2. History
kubectl rollout history deployment/myapp
# REVISION  CHANGE-CAUSE
# 1         <none>

# 3. Обновление с change-cause
kubectl annotate deployment/myapp kubernetes.io/change-cause="v1.25 for security fix"
kubectl set image deployment/myapp nginx=nginx:1.25

# 4. History
kubectl rollout history deployment/myapp
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         v1.25 for security fix

# 5. Pause/resume
kubectl rollout pause deployment/myapp
kubectl set image deployment/myapp nginx=nginx:1.26
kubectl set resources deployment/myapp --limits=cpu=500m
kubectl rollout resume deployment/myapp

# 6. Undo
kubectl rollout undo deployment/myapp --to-revision=1

# 7. Restart
kubectl rollout restart deployment/myapp
```

### 💡 Практика: как правильно использовать rollout

**✅ ОБЯЗАТЕЛЬНО:**

1. **`rollout status`** в CI/CD с timeout.
2. **`change-cause`** для каждой ревизии.
3. **`rollout undo`** при инциденте.

**👍 СТОИТ:**

4. **`pause`/`resume`** для нескольких изменений.
5. **`restart`** для обновления секретов.
6. **Мониторинг** после `restart`.

**❌ НЕ ДЕЛАЙ:**

7. **Не откатывай без диагностики.**
8. **Не забывай про `change-cause`.**
9. **Не игнорируй `rollout status`.**

### Где мы сейчас

Мы разобрали `kubectl rollout`. Теперь — **Argo Rollouts** для продвинутых стратегий.

---

## 13.8 Argo Rollouts

### 🔌 Проблема: Deployment не хватает

`Deployment` поддерживает **только** RollingUpdate и Recreate. Для Canary и Blue-Green нужны **внешние** инструменты (Istio, Nginx).

**Решение:** Argo Rollouts.

### 📊 Что такое Argo Rollouts

**Argo Rollouts** — CRD от Argo Project для продвинутых деплой-стратегий.

**Что даёт:**

- **Canary** с точным контролем процента.
- **Blue-Green** с preview service.
- **Автоматический анализ** метрик.
- **Автоматический откат.**
- **UI** для управления.

### 🎯 Установка

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# CLI
brew install argoproj/tap/kubectl-argo-rollouts

# UI
kubectl argo rollouts dashboard
```

### 🎯 Canary Rollout

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 5
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
          image: myapp:1.5.0
          ports:
            - containerPort: 8080
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: {duration: 5m}
        - setWeight: 40
        - pause: {duration: 5m}
        - setWeight: 60
        - pause: {duration: 5m}
        - setWeight: 80
        - pause: {duration: 5m}
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 1
```

**Что происходит:**

1. **20%** трафика на canary — 5 минут.
2. **Анализ** метрик.
3. **40%** — 5 минут.
4. ... до 100%.

### 🎯 AnalysisTemplate

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.99
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{job="{{args.service-name}}", status!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{job="{{args.service-name}}"}[5m]))
```

**Что даёт:**

- **Каждую минуту** запрашивает success rate.
- **Если < 99%** — счётчик неудач.
- **3 неудачи** — автоматический rollback.

### 🎯 Blue-Green Rollout

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 5
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
          image: myapp:1.5.0
  strategy:
    blueGreen:
      activeService: myapp-active
      previewService: myapp-preview
      autoPromotionEnabled: false
      scaleDownDelaySeconds: 30
```

**Что даёт:**

- **`myapp-active`** — production.
- **`myapp-preview`** — preview (можно тестировать).
- **Ручной cutover** через UI или CLI.

**Cutover:**

```bash
kubectl argo rollouts promote myapp
```

### 🎯 Команды

```bash
# Список Rollout'ов
kubectl argo rollouts list rollouts

# Статус
kubectl argo rollouts get rollout myapp --watch

# Promote (следующий шаг)
kubectl argo rollouts promote myapp

# Abort (отменить)
kubectl argo rollouts abort myapp

# Retry
kubectl argo rollouts retry rollout myapp

# Undo
kubectl argo rollouts undo myapp

# UI
kubectl argo rollouts dashboard
```

### 🎯 Пример: Canary с автоматическим анализом

**Rollout:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 5
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
          image: myapp:1.5.0
          ports:
            - containerPort: 8080
  strategy:
    canary:
      canaryService: myapp-canary
      stableService: myapp-stable
      trafficRouting:
        nginx:
          stableIngress: myapp
      steps:
        - setWeight: 5
        - pause: {duration: 2m}
        - setWeight: 25
        - pause: {duration: 5m}
        - setWeight: 50
        - pause: {duration: 5m}
      analysis:
        templates:
          - templateName: success-rate
          - templateName: latency
        startingStep: 1
        args:
          - name: service-name
            value: myapp
```

**AnalysisTemplate для latency:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency
spec:
  args:
    - name: service-name
  metrics:
    - name: latency-p99
      interval: 1m
      successCondition: result[0] <= 0.5
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{job="{{args.service-name}}"}[5m])) by (le)
            )
```

**Что происходит:**

1. **5%** трафика — 2 минуты.
2. **Анализ:** success rate ≥ 99%, latency p99 ≤ 500ms.
3. **25%** — 5 минут.
4. **50%** — 5 минут.
5. **100%.**

**Если метрики плохие** — **автоматический откат.**

### 🎯 Что даёт Argo Rollouts

**1. Точный контроль.**

Проценты трафика, а не количество реплик.

**2. Автоматизация.**

Analysis → rollback.

**3. UI.**

Визуализация процесса.

**4. Интеграция.**

Nginx, Istio, ALB, SMI.

**5. Расширяемость.**

Custom metrics, webhooks.

### 🎯 Ограничения

**1. Не для всех.**

Если нет Prometheus — analysis не работает.

**2. Сложнее Deployment.**

Дополнительный CRD.

**3. Требует настройки.**

AnalysisTemplate, traffic routing.

**4. Не для StatefulSet.**

Только для stateless.

### 🔬 Практика: Argo Rollouts

```bash
# 1. Установить
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# 2. CLI
brew install argoproj/tap/kubectl-argo-rollouts

# 3. Rollout
cat > rollout.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 5
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
          image: nginx:1.24
          ports:
            - containerPort: 80
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: {duration: 2m}
        - setWeight: 50
        - pause: {duration: 2m}
EOF
kubectl apply -f rollout.yaml

# 4. Обновление
kubectl argo rollouts set image myapp myapp=nginx:1.25

# 5. Смотреть
kubectl argo rollouts get rollout myapp --watch

# 6. UI
kubectl argo rollouts dashboard
# http://localhost:3100

# 7. Promote
kubectl argo rollouts promote myapp

# 8. Abort
kubectl argo rollouts abort myapp
```

### 💡 Практика: как правильно использовать Argo Rollouts

**✅ ОБЯЗАТЕЛЬНО:**

1. **Analysis** для автоматического отката.
2. **StartingStep** для пропуска первых шагов.
3. **FailureLimit** разумный.
4. **Начинать с 5-10%** на canary.

**👍 СТОИТ:**

4. **Multiple metrics** для анализа (success, latency, business).
5. **UI** для наблюдения.
6. **Notifications** (Slack, PagerDuty).

**❌ НЕ ДЕЛАЙ:**

7. **Не оставляй canary без analysis.**
8. **Не используй для StatefulSet.**
9. **Не забывай про Prometheus.**

### Где мы сейчас

Мы разобрали Argo Rollouts. Теперь — **Flagger** — альтернатива.

---

## 13.9 Flagger: автоматический canary

### 🔌 Проблема: нужна автоматизация canary

Argo Rollouts — мощный, но требует настройки. Что если хочется **проще**?

**Решение:** Flagger.

### 📊 Что такое Flagger

**Flagger** — оператор Kubernetes для автоматического canary.

**Что даёт:**

- **Автоматический canary** при обновлении Deployment.
- **Автоматический rollback** по метрикам.
- **Поддержка Istio, Linkerd, Nginx, App Mesh, Contour.**
- **Метрики из Prometheus, Datadog, CloudWatch.**

**Философия:** «Просто обновляй Deployment — Flagger сделает canary».

### 🎯 Как работает

1. Ты обновляешь **обычный** Deployment.
2. Flagger **замечает** изменение.
3. Flagger создаёт **canary Deployment** (копию с новым image).
4. Flagger **постепенно** увеличивает трафик на canary.
5. Flagger **анализирует** метрики.
6. Если метрики плохие — **откат**.
7. Если хорошие — **promote** до 100%.

**Ты работаешь с обычным Deployment.** Flagger делает магию.

### 🎯 Установка

```bash
helm repo add flagger https://flagger.app
helm repo update

helm upgrade -i flagger flagger/flagger \
  --namespace flagger-system \
  --create-namespace \
  --set meshProvider=istio \
  --set metricsServer=http://prometheus:9090
```

### 🎯 Пример: Canary для Deployment

**Deployment (обычный):**

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
          image: myapp:1.4.0
```

**Canary CRD:**

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  service:
    port: 80
    targetPort: 8080
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500
        interval: 1m
```

**Что происходит при обновлении image:**

1. Flagger замечает новый image.
2. Создаёт canary Deployment с 1 репликой.
3. **10%** трафика → canary.
4. Ждёт **1 минуту**.
5. Проверяет **success rate** и **latency**.
6. Если OK → **20%** трафика.
7. ... до **50%** (maxWeight).
8. Если OK → **promote** до 100%.
9. Если плохо → **rollback**.

### 🎯 Webhooks

**Кастомные проверки:**

```yaml
spec:
  analysis:
    webhooks:
      - name: acceptance-test
        type: pre-rollout
        url: http://flagger-loadtester.test/
        timeout: 30s
        metadata:
          type: bash
          cmd: "curl -sd 'test' http://myapp-canary:80/api | grep success"
      - name: load-test
        type: rollout
        url: http://flagger-loadtester.test/
        metadata:
          cmd: "hey -z 1m -q 10 -c 2 http://myapp-canary:80/"
```

**Что даёт:**

- **Acceptance test** перед началом canary.
- **Load test** для генерации трафика.

### 🎯 Команды

```bash
# Список canary
kubectl get canary -A

# Статус
kubectl describe canary myapp

# События
kubectl get events --field-selector involvedObject.name=myapp

# Логи Flagger
kubectl logs -n flagger-system deployment/flagger
```

### 🎯 Flagger vs Argo Rollouts

| Аспект | Flagger | Argo Rollouts |
|:---|:---|:---|
| **Подход** | Оператор над Deployment | CRD Rollout |
| **Настройка** | Проще | Сложнее |
| **Гибкость** | Ограниченная | Высокая |
| **UI** | Нет | Да |
| **Traffic routing** | Istio, Linkerd, Nginx, ... | Nginx, Istio, ALB, SMI |
| **Analysis** | Prometheus, Datadog, ... | Prometheus, ... |

**Когда что:**

- **Flagger** — если хочешь автоматический canary с минимальной настройкой.
- **Argo Rollouts** — если нужна гибкость и UI.

### 🔬 Практика: Flagger

```bash
# 1. Установить Flagger
helm repo add flagger https://flagger.app
helm install flagger flagger/flagger \
  --namespace flagger-system \
  --create-namespace \
  --set meshProvider=nginx

# 2. Deployment
kubectl create deployment myapp --image=nginx:1.24 --replicas=3

# 3. Service
kubectl expose deployment myapp --port=80

# 4. Canary
cat > canary.yaml <<EOF
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  service:
    port: 80
    targetPort: 80
  analysis:
    interval: 30s
    threshold: 5
    maxWeight: 50
    stepWeight: 10
EOF
kubectl apply -f canary.yaml

# 5. Обновление
kubectl set image deployment/myapp nginx=nginx:1.25

# 6. Смотреть
kubectl get canary myapp -w
kubectl describe canary myapp
```

### 💡 Практика: как правильно использовать Flagger

**✅ ОБЯЗАТЕЛЬНО:**

1. **Prometheus** для metrics.
2. **Traffic routing** (Nginx, Istio) настроен.
3. **Threshold** разумный (3-5 неудач).
4. **MaxWeight** не 100% сначала.

**👍 СТОИТ:**

4. **Webhooks** для acceptance и load test.
5. **Alerting** при rollback.
6. **Notifications** в Slack.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй без metrics server.**
8. **Не ставь maxWeight: 100%** (нет запаса).
9. **Не игнорируй rollbacks.**

### Где мы сейчас

Мы разобрали Flagger. Теперь — **Feature Flags** — деплой без деплоя.

---

## 13.10 Feature Flags: деплой без деплоя

### 🔌 Проблема: код в проде, но не готов к использованию

Ты деплоишь фичу. Но она **не готова** к production. Или нужно:

- **Тестировать на 5%** пользователей.
- **Включить для одной команды.**
- **Сразу выключить** при проблемах.

**Решение:** feature flags.

### 📊 Что такое feature flags

**Feature Flags (Feature Toggles)** — механизм включения/выключения функциональности **без деплоя**.

**Как работает:**

1. Код содержит **обёртки** вокруг фичи.
2. Флаг определяет, включена ли фича.
3. Флаг меняется **во время работы** (через UI, API).

**Что даёт:**

- **Деплой кода без включения фичи.**
- **Постепенное включение** для пользователей.
- **Мгновенное отключение** без деплоя.
- **A/B тестирование.**
- **Canary** на уровне фичи, а не Pod'а.

### 🎯 Пример

**Код без feature flag:**

```go
func handleRequest(w http.ResponseWriter, r *http.Request) {
    // Новая фича
    result := newFeature(r)
    w.Write([]byte(result))
}
```

**Проблема:** деплой = включение для всех.

**Код с feature flag:**

```go
func handleRequest(w http.ResponseWriter, r *http.Request) {
    if flags.IsEnabled("new_feature", r) {
        result := newFeature(r)
        w.Write([]byte(result))
    } else {
        result := oldFeature(r)
        w.Write([]byte(result))
    }
}
```

**Что даёт:** деплой безопасен. Фича включается флагом.

### 🎯 Типы флагов

**1. Release flags.**

Новая фича. Включается постепенно.

**2. Ops flags.**

Kill switch. Отключить фичу при инциденте.

**3. Experiment flags.**

A/B тестирование.

**4. Permission flags.**

Для конкретных пользователей (beta-тестеры).

### 🎯 Инструменты

**1. LaunchDarkly.**

Commercial. Самый популярный.

**2. Unleash.**

Open-source.

**3. Flagsmith.**

Open-source.

**4. Flipt.**

Open-source. Простой.

**5. OpenFeature.**

Стандарт.

**6. Custom.**

Свой флаг-сервис.

### 🎯 Unleash: пример

**Установка:**

```bash
helm repo add unleash https://docs.getunleash.io/helm-charts
helm install unleash unleash/unleash -n unleash --create-namespace
```

**Код на Go:**

```go
import (
    "github.com/Unleash/unleash-client-go/v3"
)

func init() {
    unleash.Initialize(
        unleash.WithUrl("http://unleash.unleash.svc.cluster.local:4242/api/"),
        unleash.WithAppName("myapp"),
        unleash.WithInstanceId("myapp-1"),
    )
}

func handleRequest(w http.ResponseWriter, r *http.Request) {
    userID := getUserID(r)
    
    if unleash.IsEnabled("new_feature", unleash.WithContext(unleash.Context{
        UserId: userID,
    })) {
        newFeatureHandler(w, r)
    } else {
        oldFeatureHandler(w, r)
    }
}
```

**Что даёт:** фича включается/выключается из UI Unleash.

### 🎯 Паттерны

**1. Canary через флаг.**

Включай фичу для 5% пользователей. Если OK — для 25%. И так до 100%.

**2. Trunk-based development.**

Все коммитят в main. Недоделанные фичи — за флагом.

**3. Kill switch.**

Фича ломается. Отключаешь флагом. Инцидент решён. Разбираешься.

**4. A/B testing.**

50% пользователей видят вариант A. 50% — вариант B. Смотришь метрики.

### 🎯 Проблемы

**1. Технический долг.**

Флаги накапливаются. Нужно удалять старые.

**2. Сложность кода.**

Много `if flag { }`. Читать сложно.

**3. Тестирование.**

Каждый флаг — новая комбинация. Экспоненциальный рост.

**4. Согласованность.**

Флаг в одном сервисе включён, в другом — выключен. Проблемы.

**Решение:**

- **Удалять флаги** после полного rollout.
- **Ограничивать количество** флагов.
- **Документировать** флаги.
- **Мониторить** использование.

### 🎯 Когда использовать

**✅ Использовать:**

- **Постепенное включение** фич.
- **A/B тестирование.**
- **Kill switches** для критичных фич.
- **Trunk-based development.**

**❌ Не использовать:**

- **Постоянные флаги.** Удаляй после rollout.
- **Флаги для конфигурации.** Для этого ConfigMap.
- **Слишком много флагов.** 10-20 максимум.

### 🔬 Практика: Unleash

```bash
# 1. Установить Unleash
helm repo add unleash https://docs.getunleash.io/helm-charts
helm install unleash unleash/unleash -n unleash --create-namespace

# 2. Порт-форвард
kubectl port-forward -n unleash svc/unleash 4242:4242

# 3. Открыть UI
# http://localhost:4242
# Login: admin / unleash4all

# 4. Создать feature flag
# UI → New feature toggle → "new_feature"

# 5. Код на Go
cat > main.go <<EOF
package main

import (
    "fmt"
    "log"
    "net/http"
    
    "github.com/Unleash/unleash-client-go/v3"
)

func init() {
    unleash.Initialize(
        unleash.WithUrl("http://localhost:4242/api/"),
        unleash.WithAppName("myapp"),
        unleash.WithInstanceId("myapp-1"),
    )
}

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        if unleash.IsEnabled("new_feature") {
            fmt.Fprintln(w, "New feature is enabled!")
        } else {
            fmt.Fprintln(w, "Old feature")
        }
    })
    
    log.Fatal(http.ListenAndServe(":8080", nil))
}
EOF

go mod init feature-demo
go mod tidy
go run main.go &

# 6. Проверить
curl localhost:8080
# Old feature

# 7. Включить флаг в UI Unleash
curl localhost:8080
# New feature is enabled!

# 8. Выключить — мгновенно
curl localhost:8080
# Old feature
```

### 💡 Практика: как правильно использовать feature flags

**✅ ОБЯЗАТЕЛЬНО:**

1. **Удалять флаги** после полного rollout.
2. **Ограничить количество** (10-20 активных).
3. **Документировать** флаги.
4. **Мониторить** использование.

**👍 СТОИТ:**

4. **Unleash** для open-source.
5. **LaunchDarkly** для enterprise.
6. **OpenFeature** для стандарта.

**❌ НЕ ДЕЛАЙ:**

7. **Не оставляй флаги навсегда.**
8. **Не используй для конфигурации.**
9. **Не забывай про тестирование.**

### Где мы сейчас

Мы разобрали feature flags. Теперь — **диагностика деплоя**.

---

## 13.11 Диагностика деплоя

### 🔌 Проблема: деплой застрял

`kubectl rollout status` показывает «Waiting...» уже 10 минут. Что не так?

### 🔍 Типичные проблемы

**1. Rollout застрял в «Waiting».**

**Причины:**

- **Новый Pod не становится Ready.**
- **Readiness probe fails.**
- **Не хватает ресурсов** (Pending).
- **Image pull error.**

**Диагностика:**

```bash
# Статус
kubectl rollout status deployment/myapp

# Pod'ы
kubectl get pods -l app=myapp
# NAME         READY   STATUS
# myapp-old    1/1     Running
# myapp-new    0/1     Pending  ← застрял
# myapp-old    1/1     Running
# myapp-old    1/1     Running

# Describe нового Pod'а
kubectl describe pod myapp-new-xxx
# Events:
#   Warning  FailedScheduling  Insufficient cpu
#   Warning  Unhealthy         Readiness probe failed
#   Warning  Failed            Failed to pull image

# Логи
kubectl logs myapp-new-xxx
```

**2. Rollout Failed.**

**Причины:**

- **`progressDeadlineSeconds`** истёк.
- **Pod crashes** (CrashLoopBackOff).

**Диагностика:**

```bash
kubectl describe deployment/myapp
# Conditions:
#   Type           Status  Reason
#   Available      True    MinimumReplicasAvailable
#   Progressing    False   ProgressDeadlineExceeded   ← Failed

# ReplicaSets
kubectl get rs -l app=myapp
# NAME         DESIRED   CURRENT   READY
# myapp-old    3         3         3
# myapp-new    1         1         0     ← не Ready
```

**3. Pod в CrashLoopBackOff.**

**Причины:**

- **Ошибка в коде.**
- **Неправильный конфиг.**
- **Нет секрета/ConfigMap.**
- **Нет доступа к БД.**

**Диагностика:**

```bash
# Логи
kubectl logs myapp-new-xxx --previous

# Events
kubectl describe pod myapp-new-xxx

# Exec
kubectl exec -it myapp-new-xxx -- sh
# (если Pod ещё жив)
```

**4. Readiness probe fails.**

**Причины:**

- **Приложение не готово.**
- **Неправильный endpoint.**
- **Слишком строгие параметры.**

**Диагностика:**

```bash
kubectl describe pod myapp-new-xxx
# Events:
#   Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 503

# Проверить вручную
kubectl exec myapp-new-xxx -- curl localhost:8080/ready
```

**5. Image pull error.**

**Причины:**

- **Неправильный image tag.**
- **Registry недоступен.**
- **Нет imagePullSecrets.**

**Диагностика:**

```bash
kubectl describe pod myapp-new-xxx
# Events:
#   Failed  Failed to pull image "myregistry.com/myapp:v1.5.0": not found
#   Failed  Failed to pull image: unauthorized
```

**6. Pod в Pending.**

**Причины:**

- **Не хватает ресурсов.**
- **Affinity/anti-affinity.**
- **Taints.**
- **PVC not bound.**

**Диагностика:**

```bash
kubectl describe pod myapp-new-xxx
# Events:
#   FailedScheduling  0/3 nodes are available: 3 Insufficient cpu
#   FailedScheduling  0/3 nodes: didn't match pod anti-affinity
```

**7. Rollback не работает.**

**Причины:**

- **`revisionHistoryLimit: 0`.**
- **Старый ReplicaSet удалён.**

**Диагностика:**

```bash
kubectl rollout history deployment/myapp
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>

kubectl get rs -l app=myapp
# Только один RS (старые удалены)
```

**Решение:** увеличить `revisionHistoryLimit`.

### 🎯 Алгоритм диагностики

**1. Статус.**

```bash
kubectl rollout status deployment/myapp
kubectl get deployment myapp -o yaml | grep -A 10 conditions
```

**2. Pod'ы.**

```bash
kubectl get pods -l app=myapp -o wide
kubectl describe pod <problematic-pod>
```

**3. События.**

```bash
kubectl get events --sort-by=.lastTimestamp
```

**4. Логи.**

```bash
kubectl logs <pod>
kubectl logs <pod> --previous
```

**5. ReplicaSets.**

```bash
kubectl get rs -l app=myapp
kubectl describe rs myapp-new-xxx
```

**6. Endpoints.**

```bash
kubectl get endpoints myapp
```

### 🎯 Rollback

```bash
# Откат
kubectl rollout undo deployment/myapp

# Или на конкретную ревизию
kubectl rollout undo deployment/myapp --to-revision=2
```

**После отката:**

```bash
kubectl rollout status deployment/myapp
kubectl get pods -l app=myapp
```

### 🎯 Argo Rollouts диагностика

```bash
kubectl argo rollouts get rollout myapp
kubectl argo rollouts get rollout myapp --watch
kubectl argo rollouts logs myapp
kubectl describe analysisrun -l rollout=myapp
```

### 🎯 Flagger диагностика

```bash
kubectl get canary myapp -o yaml
kubectl describe canary myapp
kubectl logs -n flagger-system deployment/flagger
kubectl get events --field-selector involvedObject.name=myapp
```

### 🔬 Практика: диагностика

```bash
# 1. Deployment с ошибкой
cat > deployment.yaml <<EOF
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
          image: nginx:nonexistent-tag
EOF
kubectl apply -f deployment.yaml

# 2. Статус
kubectl rollout status deployment/myapp
# Waiting for deployment spec update to be observed...

# 3. Pod'ы
kubectl get pods -l app=myapp
# NAME    READY   STATUS             RESTARTS
# myapp   0/1     ImagePullBackOff   0

# 4. Describe
kubectl describe pod myapp-xxx
# Events:
#   Failed  Failed to pull image "nginx:nonexistent-tag": not found

# 5. Решение — rollback
kubectl rollout undo deployment/myapp
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **`rollout status`** — что с rollout.
2. **`kubectl describe pod`** — Events.
3. **`kubectl logs`** — приложение.
4. **`kubectl get events`** — общие события.

**👍 СТОИТ:**

4. **`kubectl get rs`** — ReplicaSet'ы.
5. **`kubectl get endpoints`** — endpoints.
6. **Argo Rollouts/Flagger logs** для автоматизации.

**❌ НЕ ДЕЛАЙ:**

7. **Не откатывай без диагностики.**
8. **Не удаляй Deployment.**
9. **Не забывай про `--previous`.**

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **Rollout** | Процесс обновления Pod'ов. |
| **Rollback** | Откат на предыдущую версию. |
| **ReplicaSet** | Управляет Pod'ами. Deployment управляет ReplicaSet'ами. |
| **Rolling Update** | Постепенная замена Pod'ов. |
| **maxSurge** | Максимум Pod'ов сверх replicas. |
| **maxUnavailable** | Максимум недоступных Pod'ов. |
| **minReadySeconds** | Пауза после Ready. |
| **progressDeadlineSeconds** | Таймаут rollout. |
| **revisionHistoryLimit** | Сколько ReplicaSet'ов хранить. |
| **Recreate** | Остановить всё, запустить новое. |
| **Blue-Green** | Два окружения рядом. |
| **Canary** | Постепенное увеличение трафика. |
| **Lifecycle hook** | Команда в жизненном цикле контейнера. |
| **postStart** | Hook после старта. |
| **preStop** | Hook перед SIGTERM. |
| **terminationGracePeriodSeconds** | Время на graceful shutdown. |
| **Readiness probe** | Проверка готовности. |
| **Liveness probe** | Проверка живости. |
| **Argo Rollouts** | CRD для продвинутых деплоев. |
| **AnalysisTemplate** | Шаблон анализа метрик. |
| **Flagger** | Оператор для автоматического canary. |
| **Feature Flag** | Включение/выключение фичи без деплоя. |
| **Unleash** | Open-source feature flags. |
| **LaunchDarkly** | Commercial feature flags. |
| **OpenFeature** | Стандарт feature flags. |

---

## Что мы узнали?

- **Rollout и Rollback** в Deployment — через ReplicaSet'ы. История хранится в `revisionHistoryLimit`.
- **Rolling Update** — постепенная замена. `maxSurge`, `maxUnavailable`, `minReadySeconds`, readiness probe.
- **Recreate** — остановить всё, запустить новое. Downtime, для dev и несовместимых версий.
- **Blue-Green** — два окружения, переключение Service. Мгновенный откат, двойные ресурсы.
- **Canary** — постепенное увеличение трафика. Минимальный риск, сложнее.
- **Lifecycle hooks** — preStop (race condition), postStart (инициализация).
- **kubectl rollout** — status, history, undo, pause, resume, restart.
- **Argo Rollouts** — Canary и Blue-Green с автоматическим анализом.
- **Flagger** — автоматический canary с минимальной настройкой.
- **Feature flags** — деплой без деплоя. Unleash, LaunchDarkly, OpenFeature.
- **Диагностика** — rollout status, describe, logs, events.

---

## Типичные ошибки

- ❌ **Нет readiness probe.** Трафик идёт на неготовые Pod'ы.
- ❌ **`maxUnavailable: 100%`** в production. Downtime.
- ❌ **Нет `preStop: sleep`.** Race condition.
- ❌ **`revisionHistoryLimit: 0`.** Нельзя откатиться.
- ❌ **Canary с 50% сразу.** Риск.
- ❌ **Blue-Green с несовместимой схемой БД.**
- ❌ **Rollback без диагностики.**
- ❌ **Забыть про feature flags.** Постоянный деплой без возможности отключить.
- ❌ **Не удалять feature flags.** Технический долг.
- ❌ **Нет автоматического отката.** Узнаешь об инциденте от пользователей.
- ❌ **Нет мониторинга после деплоя.**
- ❌ **Нет `change-cause`.** Через месяц не вспомнишь.

---

## Для быстрого повторения

- **Rolling Update:** `maxSurge: 1`, `maxUnavailable: 0`, `minReadySeconds: 10`, readiness probe.
- **Recreate:** для dev, несовместимых версий, singleton.
- **Blue-Green:** два Deployment, Service selector, мгновенный откат.
- **Canary:** 5% → 25% → 50% → 100%, с анализом.
- **Lifecycle hooks:** `preStop: sleep 10`, `terminationGracePeriodSeconds: 60`.
- **kubectl rollout:** status, history, undo, pause, resume, restart.
- **Argo Rollouts:** AnalysisTemplate, canary, blue-green.
- **Flagger:** Canary CRD, автоматический canary.
- **Feature flags:** Unleash, LaunchDarkly, OpenFeature.
- **Диагностика:** rollout status, describe pod, logs, events.
- **Rollback:** `kubectl rollout undo`, `--to-revision`.

---

## Вопросы для самопроверки

1. Как работает Rolling Update? Что такое maxSurge и maxUnavailable?
2. Что такое readiness probe и зачем она в деплое?
3. Что такое minReadySeconds? Зачем нужен?
4. Когда использовать Recreate? Плюсы и минусы?
5. Что такое Blue-Green? Как реализовать?
6. Что такое Canary? Как реализовать?
7. Что такое preStop hook? Зачем нужен sleep?
8. Как работает `kubectl rollout undo`?
9. Что такое Argo Rollouts? Что даёт?
10. Что такое Flagger? Чем отличается от Argo Rollouts?
11. Что такое feature flags? Зачем нужны?
12. Rollout застрял — как диагностировать?
13. Pod в CrashLoopBackOff при деплое — что делать?
14. Rollback не работает — почему?
15. Ты хочешь деплоить без риска для production. Какую стратегию выбрать?

---

## Ответы

**1. Rolling Update**

Deployment создаёт новый ReplicaSet, постепенно масштабирует его, уменьшает старый. `maxSurge` — сколько Pod'ов сверх replicas, `maxUnavailable` — сколько недоступных. `maxSurge: 1`, `maxUnavailable: 0` — zero-downtime.

**2. Readiness probe**

Проверка готовности Pod'а принимать трафик. Пока не Ready — старый Pod не удаляется. Без readiness — трафик идёт на неготовые Pod'ы, ошибки.

**3. minReadySeconds**

Пауза после Ready перед продолжением rollout. Даёт время на стабилизацию. 10-30 секунд для production.

**4. Recreate**

Остановить все Pod'ы, запустить новые. Downtime. Для dev, несовместимых версий, singleton. Не для production web.

**5. Blue-Green**

Два Deployment (blue, green). Service с selector на active. Cutover — patch Service. Мгновенный откат. Двойные ресурсы.

**6. Canary**

Постепенное увеличение трафика на новую версию: 5% → 25% → 50% → 100%. Мониторинг между шагами. Автоматический откат. Через Istio, Nginx, Argo Rollouts, Flagger.

**7. preStop hook**

Выполняется перед SIGTERM. `sleep 10` — дать Kubernetes убрать Pod из endpoints. Избежать race condition. `terminationGracePeriodSeconds` > preStop + shutdown.

**8. kubectl rollout undo**

Откатывает Deployment на предыдущую ревизию. Kubernetes меняет местами ReplicaSet'ы. Создаётся новая ревизия с тем же template. `--to-revision=N` для конкретной.

**9. Argo Rollouts**

CRD для Canary и Blue-Green. AnalysisTemplate для автоматического анализа метрик. UI. Автоматический откат. Точный контроль процента трафика.

**10. Flagger**

Оператор для автоматического canary. Работает над обычным Deployment. Замечает обновление, создаёт canary, постепенно увеличивает трафик, анализирует метрики, откатывает при проблемах. Проще Argo Rollouts.

**11. Feature flags**

Включение/выключение фичи без деплоя. Release flags, kill switches, A/B тесты. Unleash, LaunchDarkly, OpenFeature. Удалять после полного rollout.

**12. Rollout застрял**

1. `kubectl rollout status` — что с rollout.
2. `kubectl get pods` — статус Pod'ов.
3. `kubectl describe pod` — Events.
4. `kubectl logs` — приложение.
5. Причины: Pending, ImagePullBackOff, CrashLoopBackOff, readiness fail.

**13. CrashLoopBackOff**

1. `kubectl logs --previous` — логи упавшего.
2. `kubectl describe pod` — Events.
3. Причины: ошибка в коде, конфиг, секреты, БД.
4. Решение: исправить или rollback.

**14. Rollback не работает**

1. `revisionHistoryLimit: 0` — нет истории.
2. Старые ReplicaSet'ы удалены.
3. Решение: увеличить `revisionHistoryLimit`.

**15. Стратегия для production**

- **Начать с Rolling Update** + readiness + preStop.
- **Добавить Canary** (Argo Rollouts/Flagger) для критичных сервисов.
- **Feature flags** для постепенного включения фич.
- **Автоматический откат** по метрикам.

---

## Куда идти дальше?

Мы разобрали обновления и деплой-стратегии в Kubernetes. Теперь ты знаешь:

- Rollout и Rollback.
- Rolling Update, Recreate, Blue-Green, Canary.
- Lifecycle hooks.
- kubectl rollout.
- Argo Rollouts и Flagger.
- Feature flags.
- Диагностика.

Но мы пока не разобрали:

- **Глава 14: Kubernetes — контроллеры и операторы** — как K8s внутри реализует «желаемое состояние».
- **Глава 15: Helm** — уже написан, но требует переименования.
- **Глава 16: Service Mesh** — уже написан.
- И так далее.

**Следующая — Глава 14: Kubernetes — контроллеры и операторы.** Погнали. 🚀