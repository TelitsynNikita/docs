# 🚀 Глава 7: CI/CD — деплой-стратегии и интеграция с K8s

**Что вы узнаете:**
- Как деплоить приложение в Kubernetes из GitLab CI.
- Четыре стратегии деплоя: recreate, rolling, blue-green, canary.
- Как делать канареечный деплой с постепенным увеличением трафика.
- Как откатывать деплой, когда что-то пошло не так.
- Как работает push-based CI/CD и почему GitOps его заменяет.
- Как связать CI/CD с Kubernetes: `kubectl`, `helm`, secrets, RBAC.
- Как мониторить деплой и принимать решение об откате.

**После прочтения вы сможете:**
- Написать пайплайн, который деплоит приложение в Kubernetes.
- Выбрать подходящую деплой-стратегию для конкретного сервиса.
- Настроить канареечный деплой с автоматическим откатом.
- Откатить неудачный деплой одной командой.
- Ограничить права CI/CD в кластере через ServiceAccount.
- Мониторить деплой и принимать решения по метрикам.

---

## Содержание

- [7.0 Пролог: деплой, который уронил прод](#70-пролог-деплой-который-уронил-прод)
- [7.1 Что такое деплой-стратегия и зачем их несколько](#71-что-такое-деплой-стратегия-и-зачем-их-несколько)
- [7.2 Recreate: остановить и запустить заново](#72-recreate-остановить-и-запустить-заново)
- [7.3 Rolling update: постепенная замена Pod'ов](#73-rolling-update-постепенная-замена-podов)
- [7.4 Blue-Green: два окружения рядом](#74-blue-green-два-окружения-рядом)
- [7.5 Canary: постепенное увеличение трафика](#75-canary-постепенное-увеличение-трафика)
- [7.6 Rollback: как откатить деплой](#76-rollback-как-откатить-деплой)
- [7.7 Деплой из GitLab CI в Kubernetes](#77-деплой-из-gitlab-ci-в-kubernetes)
- [7.8 Мониторинг деплоя и автоматический откат](#78-мониторинг-деплоя-и-автоматический-откат)
- [7.9 Мост к GitOps](#79-мост-к-gitops)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 7.0 Пролог: деплой, который уронил прод

Пятница, 18:30. Ты задеплоил новую версию бэкенда в Kubernetes:

```bash
kubectl set image deployment/myapp myapp=myregistry.com/myapp:v1.5.0
kubectl rollout status deployment/myapp
# deployment "myapp" successfully rolled out
```

Всё прошло гладко. Ты закрываешь ноутбук и уходишь на выходные.

В субботу в 10:00 звонит телефон. Прод лежит. Пользователи не могут зайти. Метрики показывают:

- Error rate: 47%.
- Latency p99: 15 секунд.
- 500-е ошибки на всех эндпоинтах.

Что произошло? В `v1.5.0` был баг: новый запрос к БД без индекса. На dev-нагрузке работал, на проде — блокировал таблицу.

**Ирония:** деплой прошёл успешно. **Технически** всё было хорошо. Kubernetes заменил Pod'ы, они запустились, health check прошёл. Но **функционально** — прод сломан.

**Что можно было сделать по-другому:**

1. **Канареечный деплой.** Выкатить `v1.5.0` сначала на 1% трафика. Посмотреть метрики. Если всё хорошо — увеличить до 100%.
2. **Автоматический откат.** Если error rate > 5% в течение 5 минут — откатить.
3. **Не деплоить в пятницу вечером.** Правило «No deploys on Friday» существует не просто так.

В этой главе мы разберём **деплой-стратегии** — как выкатывать новую версию безопасно. И **интеграцию CI/CD с Kubernetes** — как это автоматизировать.

Это — **Второй путь DevOps (Feedback)** в действии. Из Главы 0: быстрое обнаружение проблем. Ты не ждёшь, пока пользователь напишет в поддержку — ты видишь проблему в метриках через минуту после деплоя.

---

## 7.1 Что такое деплой-стратегия и зачем их несколько

### 🔌 Проблема: деплой — это риск

Каждый деплой — это риск сломать прод. Чем чаще деплоишь, тем больше суммарный риск (но меньше риск в каждом отдельном деплое — вспомни DORA-метрики из Главы 0).

Как снизить риск?

- **Тестировать** — но тесты не покрывают всё.
- **Деплоить маленькими частями** — но даже маленький деплой может сломать.
- **Деплоить постепенно** — сначала на часть трафика, потом на весь.
- **Иметь план отката** — если что-то пошло не так, быстро вернуться.

**Деплой-стратегия** — это способ выкатить новую версию приложения, минимизируя риск и время простоя.

### 📊 Четыре основные стратегии

| Стратегия | Простой | Риск | Сложность | Откат |
|:---|:---|:---|:---|:---|
| **Recreate** | Да | Высокий | Низкая | Быстрый |
| **Rolling Update** | Нет | Средний | Низкая | Быстрый |
| **Blue-Green** | Нет | Низкий | Средняя | Мгновенный |
| **Canary** | Нет | Минимальный | Высокая | Мгновенный |

**Что означают колонки:**

- **Простой** — есть ли недоступность сервиса во время деплоя.
- **Риск** — вероятность сломать прод.
- **Сложность** — насколько сложно настроить.
- **Откат** — как быстро вернуться на предыдущую версию.

### 🎯 Как выбрать стратегию

**Recreate** — для dev-окружений, некритичных сервисов, когда простой допустим.

**Rolling Update** — по умолчанию в Kubernetes для stateless-сервисов.

**Blue-Green** — для критичных сервисов, где нужна мгновенная возможность отката.

**Canary** — для больших изменений, где нужна постепенная проверка на реальном трафике.

### 🔍 Терминология

Перед разбором стратегий зафиксируем термины:

| Термин | Что означает |
|:---|:---|
| **Replica** | Экземпляр приложения (Pod в K8s). |
| **Traffic** | Запросы пользователей. |
| **Cutover** | Момент переключения трафика на новую версию. |
| **Rollback** | Откат на предыдущую версию. |
| **Zero-downtime** | Деплой без простоя. |
| **Drain** | Постепенное завершение соединений на Pod'е перед его остановкой. |
| **Graceful shutdown** | Корректное завершение работы приложения. |

### 💡 Практика: как выбрать стратегию

**✅ ОБЯЗАТЕЛЬНО:**

1. **Для stateless-сервисов — Rolling Update.** Это дефолт K8s, работает хорошо.
2. **Для критичных сервисов — Blue-Green или Canary.** Мгновенный откат, минимум риска.
3. **Для dev — Recreate.** Просто и быстро.

**👍 СТОИТ:**

4. **Начинать с Rolling Update**, потом переходить на Canary, если нужно.
5. **Иметь план отката** для каждой стратегии.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Canary для маленьких сервисов** — избыточно.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй Recreate для production.** Простой недопустим.
8. **Не деплой в пятницу вечером.** Правило «No deploys on Friday» — не миф.

### Где мы сейчас

Мы разобрали, что такое деплой-стратегии и как их выбирать. Теперь разберём каждую подробно — начиная с самой простой.

---

## 7.2 Recreate: остановить и запустить заново

### 🔌 Проблема: самая простая стратегия

**Recreate** — это остановить все старые Pod'ы, дождаться их завершения, потом запустить новые.

```
ДО ДЕПЛОЯ:                     ПОСЛЕ ДЕПЛОЯ:
┌──────────┐                   ┌──────────┐
│ Pod v1.0 │                   │ Pod v2.0 │
│ Pod v1.0 │                   │ Pod v2.0 │
│ Pod v1.0 │                   │ Pod v2.0 │
└──────────┘                   └──────────┘
      │                              ▲
      │  остановить все              │
      └──────────────────────────────┘
              (простой!)
```

### 📊 Как это работает

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: Recreate    # ← Recreate
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myregistry.com/myapp:v1.0.0
```

**Что происходит при деплое новой версии:**

1. Kubernetes останавливает все Pod'ы старой версии.
2. Ждёт, пока они полностью завершатся.
3. Запускает Pod'ы новой версии.
4. Ждёт, пока они станут ready.

**Всё это время сервис недоступен.**

### 📊 Плюсы и минусы

**Плюсы:**

- **Простота.** Не нужно думать о совместимости.
- **Чистота.** Не остаётся старых Pod'ов.
- **Для stateful** — нет конфликтов между старыми и новыми Pod'ами.

**Минусы:**

- **Простой.** Всё время деплоя сервис недоступен.
- **Долго.** 30 секунд — 5 минут, в зависимости от размера.

### 🎯 Когда использовать

**✅ Использовать:**

- **Dev-окружения.** Простой допустим.
- **Batch-задачи.** Один Pod запущен, простой не критичен.
- **Сервисы с миграциями БД, которые несовместимы с новой версией.** Если старые и новые Pod'ы не могут работать с одной схемой БД — Recreate гарантирует, что одновременно работает только одна версия.

**❌ Не использовать:**

- **Production stateless-сервисы.** Простой недопустим.
- **Web-сервисы с пользователями.**

### 🔬 Практика: пример

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-worker
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    spec:
      containers:
        - name: worker
          image: myregistry.com/batch-worker:v1.0.0
          env:
            - name: DB_SCHEMA_VERSION
              value: "v2"    # worker требует конкретной схемы БД
```

**Логика:** worker обрабатывает задачи, требующие конкретной схемы БД. Если запустить одновременно workers v1 и v2, они могут конфликтовать. Recreate гарантирует, что работает только одна версия.

### 💡 Практика: как правильно использовать Recreate

**✅ ОБЯЗАТЕЛЬНО:**

1. **Только для dev или некритичных сервисов.**
2. **Не для web-сервисов с пользователями.**

**👍 СТОИТ:**

3. **Использовать для stateful-задач** (batch processing, миграции).
4. **Комбинировать с `terminationGracePeriodSeconds`** для graceful shutdown:
   ```yaml
   spec:
     terminationGracePeriodSeconds: 30    # дать время на завершение
   ```

**❌ НЕ ДЕЛАЙ:**

5. **Не используй Recreate для production web-сервисов.**

### Где мы сейчас

Recreate — самая простая, но самая рискованная стратегия. Теперь — самая популярная: **Rolling Update**.

---

## 7.3 Rolling update: постепенная замена Pod'ов

### 🔌 Проблема: как деплоить без простоя

**Rolling Update** — это постепенная замена Pod'ов: сначала запускаются новые Pod'ы, потом останавливаются старые. Всё время есть работающие Pod'ы, готовые обслуживать трафик.

```
ШАГ 1: Начало             ШАГ 2: Запускается новый
┌──────────┐              ┌──────────┐ ┌──────────┐
│ Pod v1.0 │              │ Pod v1.0 │ │ Pod v2.0 │
│ Pod v1.0 │              │ Pod v1.0 │ │ (starting)│
│ Pod v1.0 │              │ Pod v1.0 │ │          │
└──────────┘              └──────────┘ └──────────┘
Все v1.0                  v1.0 обслуживают трафик

ШАГ 3: Новый ready        ШАГ 4: Старый останавливается
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Pod v1.0 │ │ Pod v2.0 │ │ Pod v1.0 │ │ Pod v2.0 │
│ Pod v1.0 │ │ Pod v2.0 │ │ Pod v1.0 │ │ Pod v2.0 │
│ (ready)  │ │ (ready)  │ │(stopping)│ │ (ready)  │
└──────────┘ └──────────┘ └──────────┘ └──────────┘

ШАГ 5: Финал
┌──────────┐
│ Pod v2.0 │
│ Pod v2.0 │
│ Pod v2.0 │
└──────────┘
Все v2.0
```

### 📊 Как это работает

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1     # максимум 1 Pod может быть недоступен
      maxSurge: 1           # максимум 1 Pod может быть дополнительно создан
  template:
    spec:
      containers:
        - name: myapp
          image: myregistry.com/myapp:v1.0.0
```

**Ключевые параметры:**

| Параметр | Что означает |
|:---|:---|
| `maxUnavailable` | Максимум Pod'ов, которые могут быть недоступны во время обновления. |
| `maxSurge` | Максимум Pod'ов, которые могут быть созданы **сверх** `replicas`. |

**Как они работают вместе:**

- Если `maxUnavailable: 1`, `maxSurge: 1`, `replicas: 3`:
  - Всего может быть от 2 до 4 Pod'ов (3 - 1 = 2, 3 + 1 = 4).
  - Kubernetes постепенно заменяет Pod'ы, поддерживая это окно.

**Пример с `replicas: 3`:**

1. Запускается Pod v2.0 → теперь 3 v1 + 1 v2 = 4 Pod (maxSurge).
2. Ждёт, пока Pod v2.0 станет ready.
3. Останавливает один Pod v1.0 → 2 v1 + 1 v2 = 3 Pod.
4. Запускается второй Pod v2.0 → 2 v1 + 2 v2 = 4 Pod.
5. Ждёт ready.
6. Останавливает второй Pod v1.0 → 1 v1 + 2 v2 = 3 Pod.
7. Запускается третий Pod v2.0 → 1 v1 + 3 v2 = 4 Pod.
8. Ждёт ready.
9. Останавливает последний Pod v1.0 → 0 v1 + 3 v2 = 3 Pod.

**Итого:** 4 шага, 3 замены.

### 🎯 Параметры `RollingUpdate`

**`maxUnavailable: 0` (zero-downtime):**

```yaml
rollingUpdate:
  maxUnavailable: 0
  maxSurge: 1
```

**Гарантия:** всегда `replicas` Pod'ов ready. Никогда не теряем ёмкость.

**Цена:** нужно дополнительное место (maxSurge = 1 Pod).

**`maxUnavailable: 100%` (агрессивный):**

```yaml
rollingUpdate:
  maxUnavailable: 100%
  maxSurge: 0
```

**Гарантия:** работает быстро. Все Pod'ы заменяются одновременно.

**Цена:** простой.

**Дефолтные значения:**

```yaml
rollingUpdate:
  maxUnavailable: 25%
  maxSurge: 25%
```

Для `replicas: 4`: `maxUnavailable: 1`, `maxSurge: 1`.

### 🎯 `minReadySeconds`: пауза перед продолжением

```yaml
spec:
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
```

**Что делает:** после того как Pod стал ready, Kubernetes ждёт `minReadySeconds` секунд, прежде чем считать его стабильным. Только после этого начинает следующий шаг.

**Зачем:** если приложение падает через 5 секунд после старта (например, из-за ошибки в конфигурации), health check не успеет это заметить. `minReadySeconds` даёт время.

**Рекомендация:** 10-30 секунд для production.

### 🎯 `progressDeadlineSeconds`: таймаут

```yaml
spec:
  progressDeadlineSeconds: 600    # 10 минут
```

**Что делает:** если за 10 минут деплой не завершился — Kubernetes помечает его как failed.

**По умолчанию:** 600 секунд (10 минут).

### 🎯 `revisionHistoryLimit`: хранение истории

```yaml
spec:
  revisionHistoryLimit: 10    # хранить 10 предыдущих ReplicaSet
```

**Что делает:** сколько ReplicaSet'ов (историй деплоя) хранить для rollback.

**По умолчанию:** 10.

**Зачем:** если `revisionHistoryLimit: 0`, нельзя откатиться на предыдущую версию.

### 🎯 `terminationGracePeriodSeconds`: graceful shutdown

```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: myapp
          # ...
```

**Что делает:** сколько ждать, прежде чем SIGKILL процессу.

**Как работает:**

1. Kubernetes отправляет SIGTERM контейнеру.
2. Приложение должно завершиться gracefully (закрыть соединения, дописать данные).
3. Если через 30 секунд процесс не завершился — Kubernetes отправляет SIGKILL.

**Что нужно на стороне приложения:**

- Обработать SIGTERM.
- Перестать принимать новые запросы.
- Дождаться завершения активных запросов.
- Завершиться.

**Пример на Go:**

```go
package main

import (
    "context"
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
            panic(err)
        }
    }()

    // Ждём SIGTERM
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
    <-quit

    // Graceful shutdown
    ctx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        panic(err)
    }
}
```

**Важно:** `terminationGracePeriodSeconds` должно быть **больше**, чем таймаут graceful shutdown в приложении. Иначе SIGKILL убьёт процесс до того, как он завершится сам.

### 🎯 `preStop` hook: ещё один способ graceful shutdown

```yaml
spec:
  template:
    spec:
      containers:
        - name: myapp
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
```

**Что делает:** выполняется **перед** отправкой SIGTERM.

**Зачем:** иногда нужно дать время, чтобы балансировщик перестал отправлять трафик на этот Pod. Пока Pod не получил SIGTERM, он считается ready, и трафик идёт. `preStop` даёт время убрать Pod из endpoints.

**Типичный паттерн:**

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]    # ждём, пока endpoints обновятся
```

### 🔬 Практика: правильный Rolling Update

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
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
      containers:
        - name: myapp
          image: myregistry.com/myapp:v1.0.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
```

**Что здесь есть:**

- `maxUnavailable: 0` + `maxSurge: 1` — zero-downtime.
- `minReadySeconds: 10` — пауза перед следующим шагом.
- `terminationGracePeriodSeconds: 60` — 60 секунд на graceful shutdown.
- `readinessProbe` — проверка готовности.
- `livenessProbe` — проверка живости.
- `preStop` — пауза перед SIGTERM.

**Сценарий деплоя:**

1. Запускается Pod v2.0.
2. Через 5 секунд readinessProbe начинает проверять `/ready`.
3. Когда `/ready` отвечает 200 — Pod ready.
4. Kubernetes ждёт 10 секунд (`minReadySeconds`).
5. Останавливает один Pod v1.0:
   - Выполняется `preStop` (`sleep 10`) — Pod ещё получает трафик.
   - Через 10 секунд — SIGTERM.
   - Приложение graceful shutdown в течение 60 секунд.
6. Повторяет, пока все Pod'ы не станут v2.0.

**Итого:** 3 Pod'а, ~2 минуты.

### 💡 Практика: как правильно делать Rolling Update

**✅ ОБЯЗАТЕЛЬНО:**

1. **`maxUnavailable: 0` для zero-downtime.** Никогда не теряем ёмкость.
2. **Readiness probe обязателен.** Без него Kubernetes не знает, готов ли Pod.
3. **`terminationGracePeriodSeconds` > таймаут graceful shutdown.**
4. **Graceful shutdown в приложении.** Обработать SIGTERM, дождаться активных запросов.

**👍 СТОИТ:**

5. **`minReadySeconds: 10-30`.** Даёт время на стабилизацию.
6. **`preStop` hook с `sleep`.** Убирает Pod из endpoints до SIGTERM.
7. **`revisionHistoryLimit: 10`** для rollback.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

8. **Liveness probe** — если приложение может зависнуть.

**❌ НЕ ДЕЛАЙ:**

9. **Не используй `maxUnavailable: 100%` в production.** Простой.
10. **Не забывай про readiness probe.** Без него трафик пойдёт на неготовый Pod.
11. **Не ставь `terminationGracePeriodSeconds` меньше, чем graceful shutdown в приложении.** SIGKILL убьёт процесс.
12. **Не игнорируй `preStop`.** Без него трафик пойдёт на Pod, который уже остановлен.

### Где мы сейчас

Rolling Update — рабочая лошадка Kubernetes. Теперь — **Blue-Green** — стратегия с двумя окружениями.

---

## 7.4 Blue-Green: два окружения рядом

### 🔌 Проблема: как сделать откат мгновенным

Rolling Update хорош, но откат занимает время: нужно вернуть старые Pod'ы. А если что-то пошло не так в **первую** минуту — хочется откатиться **немедленно**.

**Blue-Green** решает это: два полных окружения работают рядом. В любой момент можно переключить трафик между ними.

```
ДО CUTOVER:                     ПОСЛЕ CUTOVER:
                                
┌──────────────┐                ┌──────────────┐
│ BLUE (v1.0)  │◄── Трафик      │ BLUE (v1.0)  │
│ 3 Pod        │                │ 3 Pod        │
└──────────────┘                └──────────────┘

┌──────────────┐                ┌──────────────┐
│ GREEN (v2.0) │                │ GREEN (v2.0) │◄── Трафик
│ 3 Pod        │                │ 3 Pod        │
└──────────────┘                └──────────────┘
```

**Плюс:** откат — просто переключить трафик обратно на blue. **Мгновенно.**

**Минус:** нужно **вдвое больше ресурсов** (два полных окружения).

### 📊 Как это работает

**Два Deployment'а:**

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  labels:
    app: myapp
    version: blue
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
          image: myregistry.com/myapp:v1.0.0
---
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  labels:
    app: myapp
    version: green
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
          image: myregistry.com/myapp:v2.0.0
```

**Service, который переключает трафик:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue    # ← здесь указываем, на какое окружение идёт трафик
  ports:
    - port: 80
      targetPort: 8080
```

**Как деплоить:**

1. Оба Deployment'а работают: `myapp-blue` (v1.0.0) и `myapp-green` (v1.0.0).
2. Service указывает на `version: blue`.
3. Обновляем `myapp-green` на v2.0.0.
4. Ждём, пока green Pod'ы станут ready.
5. **Cutover:** меняем `Service.selector.version` на `green`.
6. Трафик мгновенно идёт на green.
7. Если что-то не так — меняем обратно на `blue`. **Мгновенный откат.**

### 🎯 Автоматизация cutover

Можно автоматизировать через `kubectl patch`:

```bash
# Переключить на green
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

# Откатить на blue
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'
```

**В GitLab CI:**

```yaml
deploy-green:
  stage: deploy
  script:
    - kubectl apply -f green-deployment.yaml
    - kubectl rollout status deployment/myapp-green
  environment:
    name: production
    url: https://example.com

cutover-to-green:
  stage: deploy
  script:
    - kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'
  when: manual
  environment:
    name: production
    url: https://example.com
```

### 🎯 Blue-Green с Helm

Helm упрощает управление Blue-Green:

```yaml
# values-blue.yaml
app:
  version: v1.0.0
  color: blue

# values-green.yaml
app:
  version: v2.0.0
  color: green
```

```bash
# Установить blue
helm install myapp-blue ./chart -f values-blue.yaml

# Установить green
helm install myapp-green ./chart -f values-green.yaml

# Переключить Service
helm upgrade myapp-service ./service-chart --set color=green
```

### 🎯 Blue-Green с Istio

**Istio** (Service Mesh, Глава 18) позволяет переключать трафик через VirtualService:

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
          weight: 100
        - destination:
            host: myapp
            subset: green
          weight: 0
```

**Cutover:** меняем `weight` для green на 100.

### 📊 Плюсы и минусы

**Плюсы:**

- **Мгновенный откат.** Переключение Service — секунды.
- **Можно протестировать green перед cutover** (если у него отдельный endpoint).
- **Ноль простоя.** Оба окружения работают.

**Минусы:**

- **Двойные ресурсы.** Нужно вдвое больше Pod'ов.
- **Сложнее.** Два Deployment'а, отдельные Service.
- **База данных.** Если приложение меняет схему БД — оба окружения должны работать с одной схемой.

### 🎯 Blue-Green и база данных

**Проблема:** blue работает с v1.0.0 схемы БД, green — с v2.0.0. Если они используют **одну** БД, схема должна быть совместима с обеими версиями.

**Решение: паттерн Expand-Contract (или Parallel Change):**

**Шаг 1: Expand.** Добавить новое поле/таблицу, не удаляя старое. Приложение v1.0.0 продолжает работать.

**Шаг 2: Migrate.** Приложение v2.0.0 использует новое поле. Оба окружения работают.

**Шаг 3: Contract.** Когда v1.0.0 больше не используется — удалить старое поле.

**Пример:**

```sql
-- Шаг 1: Expand — добавить новую колонку
ALTER TABLE users ADD COLUMN email_new TEXT;

-- Шаг 2: Migrate — обновить данные, использовать новую колонку
UPDATE users SET email_new = email;
-- Приложение v2.0.0 использует email_new

-- Шаг 3: Contract — удалить старую колонку
ALTER TABLE users DROP COLUMN email;
```

**Правило:** миграции БД должны быть **обратно совместимыми**. Новая версия приложения должна работать со старой схемой, а старая версия — с новой.

### 💡 Практика: как правильно делать Blue-Green

**✅ ОБЯЗАТЕЛЬНО:**

1. **Два Deployment'а с разными labels** (`version: blue`/`version: green`).
2. **Service с selector на active version.**
3. **Cutover через patch Service.selector.**
4. **Обратно совместимые миграции БД.**

**👍 СТОИТ:**

5. **Автоматизировать cutover через CI/CD.**
6. **Тестировать green перед cutover** (smoke tests).
7. **Мониторить метрики сразу после cutover.**

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

8. **Istio/Service Mesh для более тонкого контроля.**

**❌ НЕ ДЕЛАЙ:**

9. **Не забывай про БД.** Blue-Green с несовместимой схемой = катастрофа.
10. **Не оставляй старое окружение навсегда.** Удаляй blue после успешного деплоя.
11. **Не используй Blue-Green для маленьких сервисов.** Двойные ресурсы не оправданы.

### Где мы сейчас

Blue-Green даёт мгновенный откат, но требует двойных ресурсов. Теперь — **Canary** — самая тонкая стратегия.

---

## 7.5 Canary: постепенное увеличение трафика

### 🔌 Проблема: как проверить на реальном трафике безопасно

Blue-Green переключает **весь** трафик сразу. Если в новой версии баг, который проявляется на 10% запросов — ты узнаешь об этом после cutover, когда весь прод сломан.

**Canary** решает это: новая версия получает **маленькую долю трафика** (1-5%), потом постепенно увеличивает.

```
ШАГ 1: 5% на canary        ШАГ 2: 25%                 ШАГ 3: 100%
┌──────────┐ ┌──────────┐  ┌──────────┐ ┌──────────┐  ┌──────────┐
│ v1.0     │ │ v2.0     │  │ v1.0     │ │ v2.0     │  │ v2.0     │
│ (95%)    │ │ (5%)     │  │ (75%)    │ │ (25%)    │  │ (100%)   │
└──────────┘ └──────────┘  └──────────┘ └──────────┘  └──────────┘
Мониторим метрики          Мониторим             Полный rollout
```

### 📊 Как это работает

**Три компонента:**

1. **Основной Deployment** (`myapp-stable`) с v1.0.0.
2. **Canary Deployment** (`myapp-canary`) с v2.0.0.
3. **Service**, который направляет трафик на оба.

**Проблема:** обычный Kubernetes Service не умеет распределять трафик по весам. Он отправляет на все Pod'ы, попавшие под selector.

**Решения:**

1. **Ingress/Nginx с аннотациями** — `nginx.ingress.kubernetes.io/canary-weight`.
2. **Istio VirtualService** — более тонкий контроль.
3. **Service Mesh** (Linkerd, Cilium) — встроенная поддержка.
4. **Flagger** — оператор для автоматического canary.

### 🎯 Canary через Nginx Ingress

**Nginx Ingress Controller** поддерживает canary из коробки.

**Основной Ingress:**

```yaml
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
```

**Canary Ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "5"    # 5% трафика
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

**Deployment'ы:**

```yaml
# stable
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 10
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
    spec:
      containers:
        - name: myapp
          image: myregistry.com/myapp:v1.0.0
---
# canary
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
    spec:
      containers:
        - name: myapp
          image: myregistry.com/myapp:v2.0.0
```

**Как проходит canary:**

1. Запустить canary Deployment с v2.0.0.
2. Nginx отправляет 5% трафика на canary.
3. Мониторить метрики canary vs stable:
   - Error rate.
   - Latency.
   - CPU/memory.
4. Если всё хорошо — увеличить вес: 5% → 25% → 50% → 100%.
5. Если плохо — удалить canary Ingress, весь трафик вернётся на stable.

### 🎯 Canary через Istio

**Istio VirtualService** даёт более тонкий контроль:

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
              regex: ".*canary.*"    # можно направлять по заголовкам
      route:
        - destination:
            host: myapp
            subset: canary
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

**Дополнительные возможности:**

- Направлять canary только для определённых пользователей (по header, cookie).
- Разные веса для разных условий.
- Fault injection для тестирования.

### 🎯 Canary через Flagger (автоматический)

**Flagger** — оператор Kubernetes, который автоматизирует canary:

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
    interval: 1m        # проверять каждую минуту
    threshold: 5        # максимум 5 неудачных проверок
    maxWeight: 50       # максимум 50% трафика на canary
    stepWeight: 5       # увеличивать на 5% каждую минуту
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99       # 99% успешных запросов
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500      # p99 < 500ms
        interval: 1m
```

**Что делает Flagger:**

1. Создаёт canary Deployment.
2. Постепенно увеличивает вес: 0% → 5% → 10% → ... → 50%.
3. Проверяет метрики каждую минуту.
4. Если метрики в норме — продолжает.
5. Если метрики плохие — автоматически откатывает.

**Это — автоматический canary.** Идеально для production.

### 📊 Сравнение Canary и Blue-Green

| | Blue-Green | Canary |
|:---|:---|:---|
| Трафик на новую версию | 100% сразу | 1-5%, потом растёт |
| Ресурсы | 2x | 1.1x |
| Откат | Мгновенный | Мгновенный |
| Тестирование на реальном трафике | Нет | Да |
| Сложность | Средняя | Высокая |

**Когда использовать:**

- **Blue-Green** — если нужен мгновенный откат и ресурсы позволяют.
- **Canary** — если нужна проверка на реальном трафике и есть инфраструктура (Istio, Nginx Ingress).

### 💡 Практика: как правильно делать Canary

**✅ ОБЯЗАТЕЛЬНО:**

1. **Начинать с маленького веса** (1-5%).
2. **Мониторить метрики canary vs stable** в реальном времени.
3. **Автоматический откат** при плохих метриках.
4. **Не направлять canary на критичные эндпоинты** (платежи).

**👍 СТОИТ:**

5. **Использовать Flagger** для автоматизации.
6. **Направлять canary на определённых пользователей** (beta testers).
7. **Тестировать постепенно:** 1% → 5% → 25% → 50% → 100%.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

8. **Istio** для тонкого контроля (если есть Service Mesh).

**❌ НЕ ДЕЛАЙ:**

9. **Не начинай с 50% canary.** Начни с 1%.
10. **Не оставляй canary навсегда.** После успешного rollout — удали.
11. **Не игнорируй метрики.** Canary работает только если ты смотришь.
12. **Не используй canary для stateful** без осторожности. Миграции БД сложнее.

### Где мы сейчас

Canary — самая тонкая стратегия. Теперь — **rollback** — как откатить любой деплой.

---

## 7.6 Rollback: как откатить деплой

### 🔌 Проблема: что-то пошло не так, надо вернуться

Ты задеплоил новую версию. Через 5 минут видишь:

- Error rate вырос с 0.1% до 15%.
- Latency p99 выросла с 200ms до 5s.
- Пользователи жалуются.

**Надо немедленно откатиться.**

### 🎯 Rollback в Kubernetes

Kubernetes хранит историю ReplicaSet'ов (если `revisionHistoryLimit > 0`). Можно откатиться на предыдущую версию одной командой.

**Посмотреть историю:**

```bash
kubectl rollout history deployment/myapp
# REVISION  CHANGE-CAUSE
# 1         kubectl apply --filename=myapp.yaml --record=true
# 2         kubectl apply --filename=myapp.yaml --record=true
# 3         kubectl apply --filename=myapp.yaml --record=true
```

**Откатиться на предыдущую:**

```bash
kubectl rollout undo deployment/myapp
```

**Откатиться на конкретную ревизию:**

```bash
kubectl rollout undo deployment/myapp --to-revision=1
```

**Посмотреть статус:**

```bash
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
```

### 🎯 Как работает rollback

**Что происходит:**

1. Kubernetes берёт ReplicaSet из истории (ревизия N-1).
2. Обновляет Deployment, указывая на этот ReplicaSet.
3. ReplicaSet запускает Pod'ы с предыдущей версией.
4. Rolling Update'ом заменяются текущие Pod'ы на старые.

**Важно:** rollback — это **новый деплой**. Kubernetes не «отматывает» время назад, а применяет предыдущую конфигурацию. Это значит:

- `rollout history` получит **новую** ревизию (N+1).
- Образ **скачивается заново** (если его нет в локальном кэше нод).
- Миграции БД **не откатываются** автоматически.

### 🎯 Rollback в GitLab CI

**Автоматический rollback при неудачном деплое:**

```yaml
deploy:
  stage: deploy
  script:
    - kubectl apply -f deployment.yaml
    - kubectl rollout status deployment/myapp --timeout=5m || kubectl rollout undo deployment/myapp
  environment:
    name: production
    url: https://example.com
  rules:
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
      when: manual
```

**Что произошло:**

- `kubectl apply` применяет новую версию.
- `kubectl rollout status` ждёт завершения (или таймаута 5 минут).
- Если `rollout status` упал (Pod'ы не стали ready) — выполняется `kubectl rollout undo`.

**Ручной rollback:**

```yaml
rollback:
  stage: deploy
  script:
    - kubectl rollout undo deployment/myapp
  environment:
    name: production
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

Кнопка «Rollback» в GitLab UI.

### 🎯 Rollback в Blue-Green

**Самый быстрый rollback:**

```bash
# Переключить Service обратно на blue
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'
```

**Мгновенно** — трафик возвращается на старую версию. Никаких перезапусков.

### 🎯 Rollback в Canary

**Откат canary:**

```bash
# Удалить canary Ingress — весь трафик вернётся на stable
kubectl delete ingress myapp-canary

# Или уменьшить вес до 0
kubectl annotate ingress myapp-canary nginx.ingress.kubernetes.io/canary-weight=0
```

**Мгновенно.** Stable никогда не менялся.

### 🎯 Rollback и база данных

**Проблема:** Kubernetes откатывает Pod'ы, но **не откатывает миграции БД**. Если новая версия добавила колонку — при откате старая версия может её не знать (это OK, если колонка nullable). Если новая версия удалила колонку — старая версия не сможет работать.

**Правило: миграции БД должны быть обратно совместимыми.**

**Паттерн Expand-Contract:**

1. **Expand:** добавить новое, не удаляя старое.
2. **Migrate:** использовать новое.
3. **Contract:** удалить старое (когда старая версия точно не используется).

**Пример:**

```sql
-- Релиз v2.0.0 (Expand)
ALTER TABLE users ADD COLUMN email_v2 TEXT;    -- обе версии могут работать
UPDATE users SET email_v2 = email;

-- Релиз v2.1.0 (Migrate)
-- v2.1.0 использует email_v2, v2.0.0 использует email

-- Релиз v3.0.0 (Contract) — через несколько недель
ALTER TABLE users DROP COLUMN email;    -- v2.0.0 уже не используется
```

**Если что-то пошло не так — можно откатиться на v2.0.0, потому что `email` ещё существует.**

### 🎯 PostgreSQL и rollback

PostgreSQL поддерживает транзакционные DDL:

```sql
BEGIN;
ALTER TABLE users ADD COLUMN email_v2 TEXT;
-- Если что-то не так:
ROLLBACK;
-- Или:
COMMIT;
```

Но **не все миграции можно откатить**:

- `DROP COLUMN` — данные потеряны.
- `ALTER COLUMN TYPE` — может быть несовместимо.
- `DROP TABLE` — данные потеряны.

**Правило:** не удаляй данные при миграции. Сначала пометь как deprecated, удали через релиз-два.

### 💡 Практика: как правильно откатывать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Проверяй `rollout status` после деплоя.** Если не ready — откатывай.
2. **Автоматический rollback** при таймауте: `kubectl rollout undo`.
3. **Обратно совместимые миграции БД.**

**👍 СТОИТ:**

4. **`revisionHistoryLimit: 10`** — хранить историю для rollback.
5. **Тестировать rollback** на staging.
6. **Мониторить метрики после деплоя** — если что-то не так, откатывай.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **Runbook** для rollback — что делать, кто отвечает.

**❌ НЕ ДЕЛАЙ:**

8. **Не откатывай миграции БД автоматически.** Это опасно.
9. **Не удаляй старые колонки сразу.** Через релиз-два.
10. **Не откатывай без мониторинга.** Если не понимаешь, что не так — можешь сделать хуже.
11. **Не забывай, что rollback — это новый деплой.** Время на скачивание образа.

### Где мы сейчас

Мы разобрали все четыре стратегии и rollback. Теперь — **интеграция CI/CD с Kubernetes**.

---

## 7.7 Деплой из GitLab CI в Kubernetes

### 🔌 Проблема: как дать CI доступ к кластеру

CI-джоба работает в контейнере. Ей нужно:

- **`kubectl`** — для управления кластером.
- **Credentials** — для аутентификации.
- **RBAC** — ограниченные права.

### 🎯 ServiceAccount для CI

**ServiceAccount** — это identity для Pod'ов и внешних систем. Создадим SA для GitLab CI.

**1. Создать ServiceAccount:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-ci
  namespace: default
```

**2. Создать Role с правами:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitlab-ci
  namespace: default
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "pods"]
    verbs: ["get", "list"]
```

**3. Связать Role с ServiceAccount:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitlab-ci
  namespace: default
subjects:
  - kind: ServiceAccount
    name: gitlab-ci
    namespace: default
roleRef:
  kind: Role
  name: gitlab-ci
  apiGroup: rbac.authorization.k8s.io
```

**4. Получить токен:**

```bash
# Создать токен (K8s 1.24+)
kubectl create token gitlab-ci --duration=8760h    # год

# Или использовать старый ServiceAccount token secret
kubectl get secret gitlab-ci-token -o jsonpath='{.data.token}' | base64 -d
```

**5. Сохранить в GitLab CI/CD Variables:**

```
KUBE_URL      = https://kubernetes.example.com
KUBE_TOKEN    = <токен из шага 4>
KUBE_CA_CERT  = <CA сертификат кластера>
```

Пометь как **Masked** и **Protected**.

### 🎯 Деплой в GitLab CI

```yaml
stages:
  - build
  - package
  - deploy

variables:
  KUBE_NAMESPACE: production
  APP_NAME: myapp

# Сборка и пуш образа (из главы 6)
package:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# Деплой
deploy:
  stage: deploy
  image: bitnami/kubectl:latest
  before_script:
    - mkdir -p ~/.kube
    - echo "$KUBE_CA_CERT" | base64 -d > ~/.kube/ca.crt
    - |
      kubectl config set-cluster my-cluster \
        --server=$KUBE_URL \
        --certificate-authority=~/.kube/ca.crt
    - |
      kubectl config set-credentials gitlab-ci \
        --token=$KUBE_TOKEN
    - |
      kubectl config set-context default \
        --cluster=my-cluster \
        --user=gitlab-ci \
        --namespace=$KUBE_NAMESPACE
    - kubectl config use-context default
  script:
    - kubectl set image deployment/$APP_NAME $APP_NAME=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - kubectl rollout status deployment/$APP_NAME --timeout=5m
  environment:
    name: production
    url: https://example.com
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
```

### 🎯 Деплой через `kubectl apply`

Вместо `kubectl set image` можно использовать манифесты:

```yaml
deploy:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    # Подставить версию образа в манифест
    - sed -i "s|IMAGE_PLACEHOLDER|$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA|g" k8s/deployment.yaml
    - kubectl apply -f k8s/deployment.yaml
    - kubectl apply -f k8s/service.yaml
    - kubectl rollout status deployment/$APP_NAME --timeout=5m
```

**Плюс:** весь манифест под контролем (replicas, probes, resources).

**Минус:** нужно подставлять версию через `sed` или шаблоны.

### 🎯 Деплой через Helm

**Helm** (Глава 15) — менеджер пакетов для K8s. Позволяет параметризовать манифесты.

```yaml
deploy:
  stage: deploy
  image: alpine/helm:latest
  script:
    - helm upgrade --install $APP_NAME ./chart
        --namespace $KUBE_NAMESPACE
        --set image.repository=$CI_REGISTRY_IMAGE
        --set image.tag=$CI_COMMIT_SHA
        --wait
        --timeout 5m
```

**Плюс:** параметризация, версионирование, rollback одной командой.

### 🎯 Деплой через GitOps

**GitOps** (Глава 20) — другой подход. CI/CD не деплоит напрямую. Вместо этого:

1. CI собирает образ и **обновляет манифест в Git**.
2. ArgoCD/Flux в кластере видит изменения и применяет их.

```yaml
update-manifests:
  stage: deploy
  image: alpine/git:latest
  before_script:
    - git config --global user.email "ci@example.com"
    - git config --global user.name "GitLab CI"
    - git clone https://oauth2:$GITOPS_TOKEN@gitlab.com/org/gitops-repo.git
  script:
    - cd gitops-repo
    - sed -i "s|image: myapp:.*|image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA|" apps/myapp/deployment.yaml
    - git add .
    - git commit -m "Update myapp to $CI_COMMIT_SHA"
    - git push
```

**Что произошло:**

- CI обновил манифест в Git.
- ArgoCD в кластере видит изменения.
- ArgoCD применяет их автоматически.

**Преимущества:**

- **Единый источник правды** — Git.
- **Audit trail** — история коммитов.
- **Автоматический rollback** — откатить коммит.
- **Нет прямого доступа CI к кластеру** — безопаснее.

**Разберём подробно в Главе 20.**

### 📊 Сравнение подходов

| Подход | Плюсы | Минусы |
|:---|:---|:---|
| **`kubectl set image`** | Просто | Не управляет манифестами |
| **`kubectl apply`** | Весь манифест под контролем | Нужен `sed` |
| **Helm** | Параметризация, rollback | Сложнее |
| **GitOps** | Единый источник правды | Дополнительная инфраструктура |

### 💡 Практика: как правильно деплоить в K8s

**✅ ОБЯЗАТЕЛЬНО:**

1. **ServiceAccount с минимальными правами.** Не `cluster-admin`.
2. **Токен в CI/CD Variables** (Masked + Protected).
3. **`kubectl rollout status`** для проверки.
4. **Автоматический rollback** при таймауте.

**👍 СТОИТ:**

5. **Helm или kustomize** для параметризации.
6. **GitOps (ArgoCD/Flux)** для production.
7. **Мониторинг деплоя** — метрики, логи.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

8. **`--record`** в kubectl — для истории в `rollout history`.

**❌ НЕ ДЕЛАЙ:**

9. **Не используй `cluster-admin`.** RBAC должен быть минимальным.
10. **Не храни kubeconfig в CI/CD.** Используй токен.
11. **Не деплой без `rollout status`.** Не узнаешь, что деплой упал.
12. **Не забывай про `--timeout`.** Иначе джоба будет висеть вечно.

### Где мы сейчас

Мы разобрали интеграцию CI/CD с Kubernetes. Теперь — **мониторинг деплоя** и автоматический откат.

---

## 7.8 Мониторинг деплоя и автоматический откат

### 🔌 Проблема: деплой прошёл технически, но функционально упал

Вернёмся к проблеме из пролога. Деплой прошёл успешно:

- Pod'ы запустились.
- Health check прошёл.
- `rollout status` показал success.

Но **функционально** прод сломан: error rate 47%, latency выросла.

**Как автоматически обнаружить и откатить?**

### 📊 Что мониторить во время деплоя

**1. Метрики приложения:**

- **Error rate** — процент 5xx ответов.
- **Latency p50/p95/p99** — время ответа.
- **Throughput** — запросов в секунду.
- **Saturation** — использование CPU/memory.

**2. Инфраструктурные метрики:**

- **Pod restarts** — если Pod'ы перезапускаются.
- **OOM kills** — если приложение съедает память.
- **Node pressure** — если нода перегружена.

**3. Бизнес-метрики:**

- **Successful orders** — упали ли заказы.
- **User signups** — упали ли регистрации.
- **Revenue** — упала ли выручка.

**Ключевое:** метрики должны быть доступны **сразу** после деплоя (в течение 1-5 минут).

### 🎯 Prometheus + Grafana

**Prometheus** (Глава 21) собирает метрики. **Grafana** показывает.

**Пример запроса в Prometheus:**

```promql
# Error rate для myapp
sum(rate(http_requests_total{app="myapp", status=~"5.."}[5m]))
/
sum(rate(http_requests_total{app="myapp"}[5m]))
```

**Результат:** процент ошибок за 5 минут.

### 🎯 Автоматический откат через Flagger

**Flagger** (мы разбирали в canary) умеет автоматически откатывать:

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
    webhooks:
      - name: load-test
        url: http://flagger-loadtester.test/
        metadata:
          cmd: "hey -z 1m -q 10 -c 2 http://myapp-canary.test/"
```

**Что делает Flagger:**

1. Создаёт canary Deployment.
2. Постепенно увеличивает вес (10% каждую минуту).
3. Проверяет метрики:
   - Success rate ≥ 99%.
   - p99 latency ≤ 500ms.
4. Если метрики в норме — продолжает до 50%.
5. Если нет — **автоматически откатывает**.

### 🎯 Автоматический откат через GitLab CI

Если не хочешь Flagger — можно сделать откат в CI:

```yaml
deploy:
  stage: deploy
  script:
    # 1. Задеплоить
    - kubectl apply -f deployment.yaml
    - kubectl rollout status deployment/myapp --timeout=5m
    
    # 2. Подождать 2 минуты, пока метрики накопятся
    - sleep 120
    
    # 3. Проверить error rate через Prometheus API
    - |
      ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query?query=..." | jq -r '.data.result[0].value[1]')
      if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
        echo "Error rate too high: $ERROR_RATE"
        kubectl rollout undo deployment/myapp
        exit 1
      fi
  environment:
    name: production
    url: https://example.com
```

**Проблема:** CI-джоба должна **ждать** и проверять метрики. Это долго и не всегда возможно (метрики могут быть недоступны).

**Лучше:** использовать Flagger или Argo Rollouts.

### 🎯 Argo Rollouts

**Argo Rollouts** — оператор, который даёт продвинутые деплой-стратегии:

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
        args:
          - name: service-name
            value: myapp
```

**AnalysisTemplate:**

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
            sum(rate(http_requests_total{app="{{args.service-name}}", status!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{app="{{args.service-name}}"}[5m]))
```

**Что произойдёт:**

1. Argo создаёт canary Pod'ы.
2. Даёт им 20% трафика.
3. Ждёт 5 минут.
4. Проверяет success rate.
5. Если ≥ 99% — увеличивает до 40%.
6. Если < 99% — **автоматически откатывает**.
7. Продолжает до 100%.

**Это — production-grade canary.**

### 🎯 SLO и error budget

**SLO (Service Level Objective)** — цель по надёжности. Например: «99.9% запросов успешны».

**Error budget** — допустимое количество ошибок. 99.9% = 0.1% ошибок в месяц ≈ 43 минуты простоя.

**Как использовать для деплоя:**

- Если error budget **не исчерпан** — можно деплоить, риск оправдан.
- Если **исчерпан** — никаких деплоев, все силы на надёжность.

**Мониторинг error budget:**

```promql
# Error budget remaining
1 - (
  sum(rate(http_requests_total{status=~"5.."}[30d]))
  /
  sum(rate(http_requests_total[30d]))
) / (1 - 0.999)
```

**Когда error budget исчерпан:**

- Автоматически блокировать деплои.
- Алертить команду.
- Freeze на изменения.

**Это — SRE-подход** из Главы 0.

### 💡 Практика: как правильно мониторить деплой

**✅ ОБЯЗАТЕЛЬНО:**

1. **Мониторить error rate и latency** во время деплоя.
2. **Автоматический откат** при превышении порогов.
3. **Alerting** в Slack/PagerDuty при проблемах.

**👍 СТОИТ:**

4. **Flagger или Argo Rollouts** для автоматического canary.
5. **SLO и error budget** для принятия решений.
6. **Blackbox monitoring** — проверка сервиса снаружи.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **Load testing** во время canary — если есть возможность.

**❌ НЕ ДЕЛАЙ:**

8. **Не деплой без мониторинга.** Ты не узнаешь, что что-то не так.
9. **Не игнорируй error budget.** Если он исчерпан — не деплой.
10. **Не автоматизируй откат без тестов.** Может быть хуже.
11. **Не забывай про бизнес-метрики.** Технические могут быть в норме, а заказы не идут.

### Где мы сейчас

Мы разобрали мониторинг деплоя. Теперь — **мост к GitOps**.

---

## 7.9 Мост к GitOps

### 🔌 Проблема: push-based CI/CD имеет недостатки

В push-based CI/CD (то, что мы разбирали) CI **сам** деплоит:

1. CI собирает образ.
2. CI пушит в registry.
3. CI вызывает `kubectl apply` / `helm upgrade`.

**Проблемы:**

- **CI имеет доступ к кластеру.** Если CI взломают — взломают кластер.
- **Нет единого источника правды.** Состояние кластера — результат последнего `kubectl apply`, не Git.
- **Drift.** Если кто-то руками изменил ресурс — CI об этом не узнает.
- **Нет audit trail.** История деплоев — в логах CI, не в Git.

### 🎯 GitOps: Git как источник правды

**GitOps** — подход, при котором:

1. **Git — единственный источник правды** для состояния кластера.
2. **Агент в кластере** (ArgoCD, Flux) читает Git и применяет изменения.
3. **CI не имеет доступа к кластеру.** CI только обновляет манифесты в Git.

```
Push-based CI/CD:                    GitOps:
                                     
CI ──► Registry                      CI ──► Registry
 │                                    │
 │ kubectl apply                      │ git push (manifests)
 ▼                                    ▼
Kubernetes                           Git ──► ArgoCD ──► Kubernetes
```

### 🎯 Как работает GitOps

1. **CI собирает образ** и пушит в registry.
2. **CI обновляет манифест в Git** (новый тег образа).
3. **ArgoCD (в кластере)** видит изменения в Git.
4. **ArgoCD применяет манифесты** к кластеру.
5. **ArgoCD мониторит drift** — если кто-то изменил ресурс руками, ArgoCD вернёт его к состоянию из Git.

### 🎯 Преимущества GitOps

**1. Безопасность.** CI не имеет доступа к кластеру.

**2. Единый источник правды.** Git — состояние кластера.

**3. Audit trail.** Вся история — в коммитах Git.

**4. Rollback.** Откатить коммит — откатить деплой.

**5. Disaster recovery.** Если кластер потерян — восстановить из Git.

**6. Автоматический drift detection.** Если кто-то руками изменил ресурс — ArgoCD откатит.

### 🎯 GitOps-инструменты

**ArgoCD:**

- UI для просмотра состояния.
- Поддержка Helm, Kustomize, plain YAML.
- Multi-cluster.
- RBAC.

**Flux:**

- Более Git-native (использует Git как API).
- Легче ArgoCD.
- Multi-tenancy.

**Разберём подробно в Главе 20.**

### 🎯 Пример GitOps-пайплайна

**Репозиторий с кодом:**

```
myapp/
├── src/
│   └── main.go
├── Dockerfile
└── .gitlab-ci.yml
```

**Репозиторий с манифестами:**

```
myapp-gitops/
├── apps/
│   └── myapp/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
└── environments/
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

**CI в репозитории с кодом:**

```yaml
update-gitops:
  stage: deploy
  image: alpine/git:latest
  script:
    # 1. Клонировать gitops-репозиторий
    - git clone https://oauth2:$GITOPS_TOKEN@gitlab.com/org/myapp-gitops.git
    - cd myapp-gitops
    
    # 2. Обновить образ
    - sed -i "s|image: myapp:.*|image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA|" apps/myapp/deployment.yaml
    
    # 3. Закоммитить и запушить
    - git config user.email "ci@example.com"
    - git config user.name "GitLab CI"
    - git add .
    - git commit -m "Update myapp to $CI_COMMIT_SHA"
    - git push
```

**ArgoCD в кластере:**

- Смотрит на `myapp-gitops`.
- Видит изменения.
- Применяет к кластеру.

### 💡 Практика: переход на GitOps

**✅ ОБЯЗАТЕЛЬНО:**

1. **Отдельный репозиторий для манифестов.**
2. **ArgoCD или Flux в кластере.**
3. **CI не имеет доступа к кластеру.**

**👍 СТОИТ:**

4. **Multi-environment через Kustomize или Helm.**
5. **Мониторинг drift** — если что-то изменилось.
6. **Автоматический rollback через Git revert.**

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **Multi-cluster** — если несколько кластеров.

**❌ НЕ ДЕЛАЙ:**

8. **Не давай CI доступ к кластеру, если можно GitOps.**
9. **Не коммить секреты в GitOps-репозиторий.** Используй sealed-secrets или external-secrets (Глава 11).
10. **Не забывай про drift detection.** Если кто-то руками изменил ресурс — ArgoCD откатит.

### Где мы сейчас

Мы разобрали мост к GitOps. Подробно GitOps будет в Главе 20. Сейчас — завершим Главу 7.

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **Деплой-стратегия** | Способ выкатить новую версию приложения. |
| **Recreate** | Остановить старые Pod'ы, запустить новые. Простой. |
| **Rolling Update** | Постепенная замена Pod'ов. Без простоя. |
| **Blue-Green** | Два окружения рядом. Переключение трафика мгновенное. |
| **Canary** | Постепенное увеличение трафика на новую версию. |
| **Cutover** | Момент переключения трафика. |
| **Rollback** | Откат на предыдущую версию. |
| **`maxUnavailable`** | Максимум Pod'ов, недоступных во время rolling update. |
| **`maxSurge`** | Максимум Pod'ов сверх `replicas` во время rolling update. |
| **`minReadySeconds`** | Пауза после ready перед следующим шагом. |
| **`terminationGracePeriodSeconds`** | Время на graceful shutdown. |
| **`preStop` hook** | Команда перед SIGTERM. |
| **`revisionHistoryLimit`** | Сколько ReplicaSet'ов хранить для rollback. |
| **ServiceAccount** | Identity для Pod'ов и внешних систем. |
| **RBAC** | Role-Based Access Control. |
| **Flagger** | Оператор для автоматического canary. |
| **Argo Rollouts** | Оператор для продвинутых деплой-стратегий. |
| **SLO** | Service Level Objective. |
| **Error budget** | Допустимое количество ошибок. |
| **GitOps** | Подход, при котором Git — источник правды для кластера. |
| **Drift** | Расхождение между Git и кластером. |

---

## Что мы узнали?

- **Четыре деплой-стратегии:** Recreate (простой, для dev), Rolling Update (дефолт, без простоя), Blue-Green (мгновенный откат, двойные ресурсы), Canary (постепенно, для критичных изменений).
- **Rolling Update** — дефолт K8s. `maxUnavailable: 0` + `maxSurge: 1` = zero-downtime.
- **Graceful shutdown** обязателен: `terminationGracePeriodSeconds` + SIGTERM + `preStop` hook.
- **Blue-Green** — два Deployment'а, переключение Service.selector.
- **Canary** — Nginx Ingress, Istio, Flagger, Argo Rollouts.
- **Rollback** — `kubectl rollout undo`. Мгновенный в Blue-Green, постепенный в Rolling.
- **ServiceAccount + RBAC** — минимальные права для CI.
- **Деплой через CI** — `kubectl set image`, `kubectl apply`, Helm, или GitOps.
- **Мониторинг деплоя** — error rate, latency, Pod restarts.
- **Автоматический откат** — Flagger, Argo Rollouts, SLO/error budget.
- **GitOps** — Git как источник правды, агент в кластере применяет изменения.

---

## Типичные ошибки

- ❌ **Использовать Recreate в production.** Простой недопустим.
- ❌ **Не ставить readiness probe.** Трафик пойдёт на неготовый Pod.
- ❌ **`terminationGracePeriodSeconds` меньше graceful shutdown в приложении.** SIGKILL убьёт процесс.
- ❌ **Не использовать `preStop` hook.** Трафик пойдёт на Pod, который уже остановлен.
- ❌ **Blue-Green с несовместимой схемой БД.** Оба окружения должны работать.
- ❌ **Canary с 50% сразу.** Начинай с 1%.
- ❌ **Rollback без мониторинга.** Можешь сделать хуже.
- ❌ **`cluster-admin` для CI.** RBAC должен быть минимальным.
- ❌ **Хранить kubeconfig в CI/CD.** Используй токен ServiceAccount.
- ❌ **Деплой без `rollout status`.** Не узнаешь, что деплой упал.
- ❌ **Деплой без мониторинга.** Не увидишь функциональную регрессию.
- ❌ **Деплой в пятницу вечером.** Правило не миф.

---

## Для быстрого повторения

- **Recreate:** остановить всё, запустить новое. Простой.
- **Rolling Update:** `maxUnavailable: 0`, `maxSurge: 1`, `minReadySeconds`. Zero-downtime.
- **Blue-Green:** два Deployment'а, Service.selector переключает трафик.
- **Canary:** 1-5% трафика на новую версию, постепенно увеличивать. Nginx Ingress, Istio, Flagger.
- **Rollback:** `kubectl rollout undo deployment/myapp [--to-revision=N]`.
- **Graceful shutdown:** `terminationGracePeriodSeconds: 60` + SIGTERM + `preStop: sleep 10`.
- **ServiceAccount для CI:** минимальный RBAC + токен в GitLab Variables.
- **Деплой:** `kubectl set image`, `kubectl apply`, Helm, GitOps.
- **Мониторинг:** error rate, latency, Pod restarts, business metrics.
- **Автоматический откат:** Flagger, Argo Rollouts, SLO/error budget.
- **GitOps:** Git — источник правды, ArgoCD/Flux в кластере.

---

## Вопросы для самопроверки

1. Четыре деплой-стратегии — назови, объясни, когда использовать.
2. Как работает Rolling Update? Что делают `maxUnavailable` и `maxSurge`?
3. Что такое `minReadySeconds`? Зачем нужен?
4. Что такое graceful shutdown? Как его реализовать?
5. Что делают `terminationGracePeriodSeconds` и `preStop` hook?
6. Как работает Blue-Green? Как переключить трафик?
7. Как работает Canary? Чем отличается от Blue-Green?
8. Как откатить деплой в Kubernetes?
9. Что такое ServiceAccount и RBAC? Зачем они для CI?
10. Как задеплоить из GitLab CI в Kubernetes?
11. Что мониторить во время деплоя?
12. Что такое SLO и error budget? Как связаны с деплоем?
13. Что такое GitOps? Чем отличается от push-based CI/CD?
14. Почему нельзя делать `kubectl apply` из CI без ограничений?
15. Что такое drift? Как его обнаружить?

---

## Ответы

**1. Четыре стратегии**

- **Recreate** — остановить всё, запустить новое. Простой. Для dev, batch.
- **Rolling Update** — постепенная замена Pod'ов. Без простоя. Дефолт K8s.
- **Blue-Green** — два окружения рядом. Мгновенный откат. Для критичных сервисов.
- **Canary** — 1-5% трафика, постепенно увеличивать. Для рискованных изменений.

**2. Rolling Update**

Kubernetes запускает новые Pod'ы, ждёт ready, останавливает старые. `maxUnavailable` — сколько Pod'ов может быть недоступно. `maxSurge` — сколько Pod'ов сверх `replicas` можно создать.

**3. `minReadySeconds`**

Пауза после того, как Pod стал ready, перед продолжением. Даёт время на стабилизацию.

**4. Graceful shutdown**

Приложение обрабатывает SIGTERM: перестаёт принимать запросы, ждёт завершения активных, завершается. Реализация — через `signal.Notify` в Go, `signal` в других языках.

**5. `terminationGracePeriodSeconds` и `preStop`**

`terminationGracePeriodSeconds` — сколько ждать SIGTERM до SIGKILL. `preStop` — команда перед SIGTERM (обычно `sleep`, чтобы убрать Pod из endpoints).

**6. Blue-Green**

Два Deployment'а: blue (v1.0), green (v2.0). Service с selector на активное окружение. Переключение — `kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'`.

**7. Canary**

Новая версия получает 1-5% трафика. Мониторится. Постепенно увеличивается до 100%. Отличие от Blue-Green: постепенно, не всё сразу.

**8. Rollback**

`kubectl rollout undo deployment/myapp`. Или `--to-revision=N` для конкретной ревизии. В Blue-Green — переключить Service. В Canary — удалить canary.

**9. ServiceAccount и RBAC**

ServiceAccount — identity. RBAC — права. Для CI: минимальные права (get, list, update, patch deployment). Токен в GitLab Variables (Masked + Protected).

**10. Деплой из CI в K8s**

Создать ServiceAccount + RBAC, получить токен, сохранить в Variables. В CI: настроить kubeconfig, `kubectl set image` или `kubectl apply`, `kubectl rollout status`.

**11. Что мониторить**

Error rate, latency (p50/p95/p99), throughput, Pod restarts, OOM kills, business metrics.

**12. SLO и error budget**

SLO — цель (99.9% успешных запросов). Error budget — допустимое количество ошибок (0.1% в месяц). Если исчерпан — freeze на деплои.

**13. GitOps**

Git — единственный источник правды. Агент в кластере (ArgoCD, Flux) читает Git и применяет. CI не имеет доступа к кластеру.

**14. Почему нельзя без ограничений**

CI имеет токен с правами на кластер. Если CI взломают — взломают кластер. RBAC ограничивает: только деплой, не всё.

**15. Drift**

Расхождение между состоянием в Git и в кластере. Обнаруживается ArgoCD/Flux автоматически. Исправляется revert'ом к состоянию из Git.

---

## Куда идти дальше?

Мы разобрали CI/CD с GitLab CI и деплой в Kubernetes. Теперь ты умеешь:

- Строить пайплайны: build → test → package → deploy.
- Собирать Docker-образы в CI.
- Деплоить в Kubernetes из CI.
- Выбирать деплой-стратегию: Recreate, Rolling, Blue-Green, Canary.
- Откатывать деплой.
- Мониторить деплой и автоматически откатывать.

Но мы пока не разобрали **Kubernetes** глубоко. В Главе 7 мы использовали готовые Deployment'ы, но не разбирали:

- Как работает control plane.
- Как scheduler решает, куда поставить Pod.
- Как services и ingress работают под капотом.
- Как управлять storage.
- Как работать с ConfigMaps и Secrets.

Следующий блок — **Kubernetes**. Мы начнём с архитектуры кластера и разберём всё до деталей.

**Глава 8: Kubernetes — архитектура и основные объекты.** Погнали. 🚀