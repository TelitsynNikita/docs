# 🌳 Глава 20: GitOps — Git как источник правды

**Что вы узнаете:**
- Что такое GitOps и чем отличается от push-based CI/CD.
- Четыре принципа GitOps.
- Как работает ArgoCD: Application, Sync, Health, Diff.
- Как работает Flux: GitRepository, Kustomization, HelmRelease.
- Что такое drift detection и как его исправлять.
- Как организовать репозитории для GitOps (monorepo, polyrepo).
- Как делать promotion между окружениями.
- Как реализовать Blue-Green и Canary через GitOps.
- Как использовать Secrets в GitOps (Sealed Secrets, External Secrets).
- Как делать disaster recovery через Git.
- Как мигрировать с push-based CI/CD на GitOps.

**После прочтения вы сможете:**
- Настроить ArgoCD или Flux в кластере.
- Организовать репозиторий для GitOps.
- Деплоить приложения через Git.
- Обнаруживать и исправлять drift.
- Делать rollback через `git revert`.
- Интегрировать GitOps с Helm и Kustomize.
- Работать с секретами в GitOps.
- Мигрировать с push-based на pull-based деплой.

---

## Содержание

- [20.0 Пролог: кто на самом деле задеплоил в прод?](#200-пролог-кто-на-самом-деле-задеплоил-в-прод)
- [20.1 Что такое GitOps](#201-что-такое-gitops)
- [20.2 Четыре принципа GitOps](#202-четыре-принципа-gitops)
- [20.3 Push vs Pull: почему pull безопаснее](#203-push-vs-pull-почему-pull-безопаснее)
- [20.4 ArgoCD: архитектура и основные объекты](#204-argocd-архитектура-и-основные-объекты)
- [20.5 Flux: альтернатива ArgoCD](#205-flux-альтернатива-argocd)
- [20.6 Организация репозиториев](#206-организация-репозиториев)
- [20.7 Drift detection и self-healing](#207-drift-detection-и-self-healing)
- [20.8 Promotion между окружениями](#208-promotion-между-окружениями)
- [20.9 Blue-Green и Canary через GitOps](#209-blue-green-и-canary-через-gitops)
- [20.10 Secrets в GitOps](#2010-secrets-в-gitops)
- [20.11 Disaster recovery через Git](#2011-disaster-recovery-через-git)
- [20.12 Миграция с push-based на GitOps](#2012-миграция-с-push-based-на-gitops)
- [20.13 Диагностика проблем](#2013-диагностика-проблем)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 20.0 Пролог: кто на самом деле задеплоил в прод?

Пятница, 18:00. Ты собираешься уходить. Внезапно приходит алерт: прод лежит.

Ты открываешь Slack и спрашиваешь: «Кто задеплоил в прод?»

Тишина. Никто не отвечает.

Ты смотришь логи CI/CD:

```bash
git log --oneline -5
# a1b2c3d Fix: update user service
# e4f5g6h Feature: add payment endpoint
# i7j8k9l Refactor: database queries
# m1n2o3p Chore: update dependencies
# q4r5s6t Fix: typo in README
```

Пять коммитов за последние 2 часа. Какой из них задеплоен в прод? Когда? Кем?

Ты смотришь в кластер:

```bash
kubectl get deployments -n production
# NAME         READY   UP-TO-DATE   AVAILABLE   AGE
# myapp        3/3     3            3           45d
# api          3/3     3            3           45d
# worker       3/3     3            3           45d
```

**AGE 45d.** Деплой был 45 дней назад? Но CI/CD показывал успешный деплой вчера.

Ты смотришь в GitLab CI:

```yaml
deploy:
  script:
    - kubectl apply -f k8s/
```

**CI/CD делает `kubectl apply`.** Но кто и что применял — неясно. Логи CI/CD закончились. Разработчик мог применить вручную. Кто-то мог изменить Deployment через `kubectl edit`.

**Дрейф (drift).** Состояние в кластере **не соответствует** состоянию в Git.

**Это — фундаментальная проблема push-based CI/CD.**

**GitOps решает эту проблему.** Git — **единственный источник правды**. Всё, что в кластере, **должно** быть в Git. Если что-то изменилось в кластере вручную — GitOps-агент **откатит** изменения.

В этой главе мы разберём GitOps от основ до продвинутых техник. Ты научишься использовать ArgoCD и Flux, организовывать репозитории, делать promotion между окружениями, работать с секретами.

Это — **эволюция CI/CD**. Не замена, а следующий шаг. CI собирает образ, GitOps деплоит.

---

## 20.1 Что такое GitOps

### 🔌 Проблема: кластер — чёрный ящик

В push-based CI/CD:

1. Разработчик коммитит код.
2. CI собирает образ, пушит в registry.
3. CI делает `kubectl apply` или `helm upgrade`.
4. Кластер обновляется.

**Проблемы:**

**1. CI имеет доступ к кластеру.**

Если CI взломают — взломают кластер. CI — это «god mode» на прод.

**2. Нет источника правды.**

Состояние кластера — результат последнего `kubectl apply`. Не Git.

**3. Drift.**

Кто-то изменил ресурс вручную (`kubectl edit`, `kubectl scale`). CI об этом не знает. При следующем деплое изменения перезапишутся или останутся.

**4. Нет audit trail.**

Кто, когда, что задеплоил — в логах CI. Но если разработчик применил вручную — ничего.

**5. Нет автоматического восстановления.**

Если кластер удалили — восстанавливать вручную. Даже с бэкапами — долго.

**6. Медленно.**

CI должен дождаться, пока `kubectl apply` завершится. Плюс credentials, плюс доступ.

**7. Нет preview.**

Как посмотреть, что изменится? Только через `kubectl diff` в CI.

### 📦 Что такое GitOps

**GitOps** — подход, при котором:

1. **Git — единственный источник правды** для состояния кластера.
2. **Агент в кластере** (ArgoCD, Flux) читает Git и применяет изменения.
3. **CI не имеет доступа к кластеру.**

**Ключевое изменение:**

```
Push-based CI/CD:               GitOps:

CI ──► Registry                 CI ──► Registry
 │                               │
 │ kubectl apply                 │ git push (manifests)
 ▼                               ▼
Kubernetes                      Git ──► ArgoCD ──► Kubernetes
```

**CI только обновляет манифесты в Git.** ArgoCD в кластере видит изменения и применяет.

### 🎯 Что даёт GitOps

**1. Безопасность.**

CI не имеет credentials к кластеру. Даже если CI взломают — не смогут задеплоить что угодно.

**2. Единый источник правды.**

Git — состояние кластера. Всё остальное — drift.

**3. Audit trail.**

Вся история — в коммитах Git. `git log`, `git blame`, `git revert`.

**4. Drift detection.**

ArgoCD/Flux сравнивают состояние кластера с Git. Если что-то изменилось — показывают.

**5. Self-healing.**

Можно настроить автоматический откат drift'а. Изменил вручную → ArgoCD откатит через минуту.

**6. Rollback.**

`git revert` → автоматический rollback.

**7. Disaster recovery.**

Если кластер потерян — восстановить из Git. Одна команда.

**8. Preview.**

Pull Request → ArgoCD показывает diff → ревью.

### 🎯 История GitOps

- **2017** — Weaveworks популяризовали термин «GitOps».
- **2018** — ArgoCD выпущен (Intuit).
- **2019** — Flux v2 (Weaveworks).
- **2020+** — ArgoCD и Flux становятся стандартом.

Сегодня GitOps используют: Intuit, Tesla, Spotify, Adobe, и тысячи других.

### 💡 Практика: что важно понять про GitOps

**✅ ОБЯЗАТЕЛЬНО:**

1. **Git — единственный источник правды.**
2. **Агент в кластере** (ArgoCD, Flux) применяет изменения.
3. **CI не имеет доступа к кластеру.**

**👍 СТОИТ:**

4. **Drift detection** для контроля.
5. **Self-healing** для автоматического отката.
6. **Preview через PR** для ревью.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `kubectl apply` из CI** для production.
8. **Не давай CI credentials к кластеру.**
9. **Не игнорируй drift.**

### Где мы сейчас

Мы разобрали, что такое GitOps. Теперь — **четыре принципа**.

---

## 20.2 Четыре принципа GitOps

### 🔌 Проблема: как понять, что такое «настоящий GitOps»

Термин «GitOps» используется по-разному. Одни называют GitOps любой деплой через Git. Другие — только pull-based. Как понять, что такое «настоящий GitOps»?

**Ответ:** четыре принципа от OpenGitOps (CNCF).

### 📊 Четыре принципа

**1. Declarative (Декларативный).**

Всё состояние системы описывается декларативно. Не «выполни эти команды», а «вот желаемое состояние».

```yaml
# ✅ Декларативно
replicas: 3

# ❌ Императивно
kubectl scale deployment myapp --replicas=3
```

**2. Versioned and Immutable (Версионировано и неизменяемо).**

Желаемое состояние хранится в Git. Каждое изменение — коммит. История сохраняется.

**3. Pulled Automatically (Автоматически применяется).**

Агент в кластере **сам** читает Git и применяет изменения. Не CI «пушит», а агент «пуллит».

**4. Continuously Reconciled (Непрерывно согласуется).**

Агент **постоянно** сравнивает состояние кластера с Git. Если расхождение — исправляет.

### 🎯 Как принципы работают вместе

```
1. Declarative:
   Манифесты описывают состояние (replicas: 3)

2. Versioned:
   Манифесты в Git, каждое изменение — коммит

3. Pulled:
   ArgoCD читает Git каждые 3 минуты

4. Reconciled:
   ArgoCD сравнивает с кластером.
   Если replicas = 2 — создаёт третий Pod.
```

### 🎯 Что НЕ является GitOps

**1. CI делает `kubectl apply`.**

Это **push-based**, не GitOps. CI имеет доступ к кластеру.

**2. Git как хранилище манифестов, но деплой вручную.**

Это **Git как документация**, не GitOps.

**3. Git + скрипт, который запускается по cron.**

Это **cron-based**, не GitOps. Нет continuous reconciliation.

**4. Git + webhook, который триггерит `kubectl apply`.**

Это **push-based**, не GitOps.

### 🎯 Spectrum GitOps

**Не всё чёрно-белое.** Есть спектр:

| Уровень | Что делается |
|:---|:---|
| **0. Manual** | Манифесты в Git, деплой вручную |
| **1. CI/CD push** | CI делает `kubectl apply` |
| **2. Webhook** | Git webhook триггерит apply |
| **3. GitOps (pull)** | Агент в кластере читает Git |
| **4. GitOps + auto-sync** | Агент сам применяет изменения |
| **5. GitOps + self-heal** | Агент откатывает drift |

**Настоящий GitOps** — уровни 3-5.

### 🎯 Преимущества «настоящего» GitOps

**1. Безопасность.**

CI не имеет credentials. Агент в кластере имеет **только read** к Git.

**2. Восстановление.**

Кластер потерян? Развернуть новый → ArgoCD → всё восстановится из Git.

**3. Audit.**

Кто, что, когда — в Git.

**4. Один workflow для всего.**

Не нужно помнить команды `kubectl`, `helm`. Всё через Git.

### 🔬 Практика: проверка GitOps

**Вопросы для проверки:**

1. **Где живёт желаемое состояние?**
   - ✅ В Git.
   - ❌ В CI.

2. **Кто применяет изменения?**
   - ✅ Агент в кластере.
   - ❌ CI.

3. **Что произойдёт при изменении ресурса вручную?**
   - ✅ Агент откатит (если self-heal).
   - ❌ Ничего.

4. **Можно ли восстановить кластер из Git?**
   - ✅ Да, одной командой.
   - ❌ Нет.

5. **Есть ли audit trail?**
   - ✅ В коммитах Git.
   - ❌ В логах CI (если повезёт).

**Если все ответы ✅ — у тебя настоящий GitOps.**

### 💡 Практика: как внедрять GitOps

**✅ ОБЯЗАТЕЛЬНО:**

1. **Декларативные манифесты.**
2. **Всё в Git.**
3. **Агент в кластере** (ArgoCD, Flux).
4. **Continuous reconciliation.**

**👍 СТОИТ:**

5. **Auto-sync** для dev/staging.
6. **Manual sync** для production.
7. **Self-heal** для критичных.

**❌ НЕ ДЕЛАЙ:**

8. **Не называй GitOps `kubectl apply` из CI.**
9. **Не игнорируй drift.**
10. **Не давай CI credentials к кластеру.**

### Где мы сейчас

Мы разобрали четыре принципа. Теперь — **push vs pull**.

---

## 20.3 Push vs Pull: почему pull безопаснее

### 🔌 Проблема: почему не push

«У меня CI делает `kubectl apply`. Чем это хуже?»

**Ответ:** безопасность, audit, drift.

### 📊 Push-based

```
┌─────────────────────────────────────┐
│  CI/CD (GitLab Runner, GitHub)      │
│                                      │
│  1. Собирает образ                   │
│  2. Пушит в registry                 │
│  3. kubectl apply -f k8s/            │
│                                      │
│  Имеет:                              │
│  - kubeconfig (полный доступ)        │
│  - registry credentials              │
│  - SSH-ключи                         │
└──────────────┬──────────────────────┘
               │
               │ kubectl apply
               │ (credentials в CI)
               ▼
┌─────────────────────────────────────┐
│  Kubernetes Cluster                  │
└─────────────────────────────────────┘
```

**Проблемы:**

**1. CI — god mode.**

Credentials в CI = доступ ко всему кластеру. Если CI взломают — взломают кластер.

**2. Много credentials.**

kubeconfig, registry, cloud credentials. Каждый — риск.

**3. Нет reconciliation.**

CI применил и ушёл. Если что-то изменилось — не узнает.

**4. Нет audit.**

Кто изменил — в логах CI, но не в Git.

### 📊 Pull-based

```
┌─────────────────────────────────────┐
│  CI/CD                               │
│                                      │
│  1. Собирает образ                   │
│  2. Пушит в registry                 │
│  3. Обновляет манифест в Git         │
│                                      │
│  Не имеет:                           │
│  - Доступа к кластеру                │
│                                      │
│  Имеет:                              │
│  - Только git push к manifests-repo  │
└──────────────┬──────────────────────┘
               │
               │ git push
               ▼
┌─────────────────────────────────────┐
│  Git (manifests repo)                │
└──────────────┬──────────────────────┘
               │
               │ git pull (каждые 3 мин)
               │
┌──────────────▼──────────────────────┐
│  Kubernetes Cluster                  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  ArgoCD / Flux                 │  │
│  │  (агент в кластере)            │  │
│  │                                │  │
│  │  - читает Git                  │  │
│  │  - применяет изменения         │  │
│  │  - мониторит drift             │  │
│  └────────────────────────────────┘  │
└─────────────────────────────────────┘
```

**Преимущества:**

**1. CI без credentials к кластеру.**

CI только обновляет Git. Даже если CI взломают — максимум изменят манифест, который ревьюится.

**2. Единая точка входа.**

Только Git. Не нужно помнить про `kubectl`, `helm`.

**3. Continuous reconciliation.**

ArgoCD постоянно смотрит. Drift исправляется.

**4. Audit.**

Всё в Git.

**5. Disaster recovery.**

Кластер потерян → ArgoCD восстановит из Git.

### 📊 Сравнение

| Аспект | Push | Pull |
|:---|:---|:---|
| **Credentials в CI** | Kubeconfig (полный доступ) | Только git push |
| **Направление** | CI → Кластер | Git → Кластер |
| **Reconciliation** | Нет | Да |
| **Drift detection** | Нет | Да |
| **Audit trail** | Логи CI | Git |
| **Disaster recovery** | Ручное | Из Git |
| **Preview** | `kubectl diff` | PR + ArgoCD |
| **Безопасность** | Низкая | Высокая |
| **Сложность настройки** | Простая | Средняя |

### 🎯 Когда использовать что

**Push-based:**

- **Маленькие проекты** (1-2 сервиса).
- **Dev-окружения.**
- **Когда нет времени на GitOps.**
- **Bootstrap (первый деплой ArgoCD).**

**Pull-based (GitOps):**

- **Production.**
- **Много сервисов** (5+).
- **Много команд.**
- **Compliance requirements.**

### 🎯 Гибридный подход

**Можно комбинировать:**

- **CI push** для сбора образов в registry.
- **GitOps pull** для деплоя манифестов.

Это — стандартный подход. CI собирает, GitOps деплоит.

### 🔬 Практика: проверка безопасности

**Push-based CI:**

```yaml
# .gitlab-ci.yml
deploy:
  script:
    - kubectl config set-cluster ...    # kubeconfig в CI
    - kubectl apply -f k8s/
```

**Что в CI:**

- Kubeconfig с полным доступом.
- Риск: компрометация → полный доступ к кластеру.

**GitOps:**

```yaml
# .gitlab-ci.yml
update-manifests:
  script:
    - git clone https://gitlab.com/org/gitops.git
    - sed -i "s|image:.*|image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA|" apps/myapp/deployment.yaml
    - git commit -am "Update myapp to $CI_COMMIT_SHA"
    - git push
```

**Что в CI:**

- Только git push к manifests-repo.
- Нет доступа к кластеру.
- Риск: компрометация → изменение манифеста (ревьюится).

### 💡 Практика: как перейти на pull-based

**✅ ОБЯЗАТЕЛЬНО:**

1. **Отдельный репозиторий для манифестов.**
2. **ArgoCD/Flux в кластере.**
3. **CI без credentials к кластеру.**

**👍 СТОИТ:**

4. **Auto-sync** для dev/staging.
5. **Manual sync** для prod.
6. **Notifications** при изменении.

**❌ НЕ ДЕЛАЙ:**

7. **Не давай CI полный доступ к кластеру.**
8. **Не используй `kubectl apply` из CI.**
9. **Не храни kubeconfig в CI.**

### Где мы сейчас

Мы разобрали push vs pull. Теперь — **ArgoCD**.

---

## 20.4 ArgoCD: архитектура и основные объекты

### 🔌 Проблема: как работает GitOps-агент

GitOps-агент в кластере читает Git и применяет изменения. Как это работает?

**ArgoCD** — самый популярный GitOps-агент.

### 📊 Архитектура ArgoCD

```
┌─────────────────────────────────────────────────────────────┐
│                     ARGOCD                                   │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │  API Server      │  │  Repository      │                │
│  │  (gRPC/REST)     │  │  Server          │                │
│  │                  │  │                  │                │
│  │  - Web UI        │  │  - Клонирует     │                │
│  │  - CLI           │  │    Git-репо      │                │
│  │  - RBAC          │  │  - Рендерит      │                │
│  │                  │  │    манифесты     │                │
│  └────────┬─────────┘  └──────────────────┘                │
│           │                                                  │
│           │                                                  │
│  ┌────────▼─────────┐  ┌──────────────────┐                │
│  │  Application     │  │  Redis           │                │
│  │  Controller      │  │  (кэш)           │                │
│  │                  │  │                  │                │
│  │  - Сравнивает    │  │                  │                │
│  │    Git и кластер │  │                  │                │
│  │  - Применяет     │  │                  │                │
│  │    изменения     │  │                  │                │
│  └──────────────────┘  └──────────────────┘                │
└─────────────────────────────────────────────────────────────┘
                           │
                           │ kubectl / Helm
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                        │
└─────────────────────────────────────────────────────────────┘
```

**Компоненты:**

| Компонент | Что делает |
|:---|:---|
| **API Server** | Web UI, REST API, gRPC, RBAC |
| **Repository Server** | Клонирует Git, рендерит манифесты (Helm, Kustomize) |
| **Application Controller** | Сравнивает Git и кластер, применяет |
| **Redis** | Кэш для быстрого доступа |
| **Dex** (опционально) | SSO (OIDC, LDAP, SAML) |

### 🎯 Application — главный объект

**Application** — это CRD, который описывает:

- **Что** деплоить (source).
- **Куда** деплоить (destination).
- **Как** деплоить (syncPolicy).

**Пример:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/myorg/gitops.git
    targetRevision: main
    path: apps/myapp/overlays/production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true          # удалять ресурсы, которых нет в Git
      selfHeal: true       # откатывать drift
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Что произойдёт:**

1. ArgoCD клонирует репозиторий `https://github.com/myorg/gitops.git`.
2. Читает манифесты из `apps/myapp/overlays/production`.
3. Применяет их в namespace `production`.
4. Сравнивает каждые 3 минуты.
5. Если drift — откатывает (из-за `selfHeal: true`).
6. Удаляет ресурсы, которых нет в Git (из-за `prune: true`).

### 🎯 Sync Policy

**Automated:**

```yaml
syncPolicy:
  automated:
    prune: true          # удалять ресурсы, которых нет в Git
    selfHeal: true       # откатывать drift
    allowEmpty: false    # не применять пустой набор манифестов
```

**Что даёт:**

- **`prune: true`** — если удалил манифест из Git, ресурс удалится из кластера.
- **`selfHeal: true`** — если кто-то изменил ресурс вручную, ArgoCD откатит.

**Manual:**

```yaml
syncPolicy: {}          # пустая политика — только manual sync
```

**Что даёт:** изменения **не применяются автоматически**. Нужно нажать «Sync» в UI или `argocd app sync myapp`.

**Когда использовать:**

- **`automated`** — dev, staging.
- **`manual`** — production.

### 🎯 Application Health и Sync Status

**Sync Status:**

| Статус | Что означает |
|:---|:---|
| **Synced** | Git = кластер |
| **OutOfSync** | Git ≠ кластер |
| **Unknown** | Не удалось определить |

**Health Status:**

| Статус | Что означает |
|:---|:---|
| **Healthy** | Все ресурсы работают |
| **Progressing** | Ресурсы разворачиваются |
| **Degraded** | Есть проблемы |
| **Suspended** | Приостановлен |
| **Missing** | Ресурсы отсутствуют |
| **Unknown** | Не удалось определить |

**Проверка:**

```bash
argocd app get myapp
```

Вывод:

```
Name:               argocd/myapp
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          production
URL:                https://argocd.example.com/applications/myapp
Repo:               https://github.com/myorg/gitops.git
Target:             main
Path:               apps/myapp/overlays/production
SyncWindow:         Sync Allowed
Sync Policy:        Automated (Prune)
Sync Status:        Synced to main (a1b2c3d)
Health Status:      Healthy

GROUP  KIND        NAMESPACE   NAME     STATUS  HEALTH
apps   Deployment  production  myapp    Synced  Healthy
       Service     production  myapp    Synced  Healthy
net... Ingress     production  myapp    Synced  Healthy
```

### 🎯 ApplicationSet — много приложений

**ApplicationSet** — генератор Applications. Создаёт много Applications из шаблона.

**Пример: Application для каждого окружения:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            namespace: dev
            revision: main
          - env: staging
            namespace: staging
            revision: main
          - env: prod
            namespace: production
            revision: v1.0.0
  
  template:
    metadata:
      name: 'myapp-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/gitops.git
        targetRevision: '{{revision}}'
        path: 'apps/myapp/overlays/{{env}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

**Что произойдёт:** создастся 3 Application: `myapp-dev`, `myapp-staging`, `myapp-prod`.

**Другие generators:**

- **Git generator** — из файлов в репо.
- **Git directories** — для каждого каталога.
- **Git files** — из JSON/YAML.
- **Cluster generator** — для каждого кластера.
- **Matrix** — комбинация.

### 🎯 Projects

**AppProject** — группировка Applications с RBAC.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production applications
  
  sourceRepos:
    - https://github.com/myorg/gitops.git
  
  destinations:
    - namespace: production
      server: https://kubernetes.default.svc
  
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
  
  roles:
    - name: developer
      policies:
        - p, proj:production:developer, applications, get, production/*, allow
        - p, proj:production:developer, applications, sync, production/*, allow
      groups:
        - developers
```

**Что даёт:**

- **Ограничение source repos.**
- **Ограничение destinations.**
- **RBAC** для команд.

### 🎯 Установка ArgoCD

```bash
# 1. Создать namespace
kubectl create namespace argocd

# 2. Установить
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Проверить
kubectl get pods -n argocd
# NAME                                  READY   STATUS
# argocd-application-controller-0       1/1     Running
# argocd-applicationset-controller-xxx  1/1     Running
# argocd-dex-server-xxx                 1/1     Running
# argocd-notifications-controller-xxx   1/1     Running
# argocd-redis-xxx                      1/1     Running
# argocd-repo-server-xxx                1/1     Running
# argocd-server-xxx                     1/1     Running

# 4. Получить пароль admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# 5. Port-forward для UI
kubectl port-forward -n argocd svc/argocd-server 8080:443

# 6. Открыть https://localhost:8080
# Login: admin / <пароль>
```

**Через Helm:**

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace
```

### 🎯 ArgoCD CLI

```bash
# Установить
brew install argocd

# Login
argocd login localhost:8080

# Список applications
argocd app list

# Получить app
argocd app get myapp

# Sync
argocd app sync myapp

# Diff
argocd app diff myapp

# History
argocd app history myapp

# Rollback
argocd app rollback myapp 1

# Logs
argocd app logs myapp

# Delete
argocd app delete myapp
```

### 🔬 Практика: первое приложение

```bash
# 1. Создать Git-репозиторий с манифестами
mkdir gitops-demo && cd gitops-demo

cat > deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
EOF

git init
git add .
git commit -m "Add nginx"
git remote add origin https://github.com/myorg/gitops-demo.git
git push -u origin main

# 2. Создать Application в ArgoCD
cat > application.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-demo.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

kubectl apply -f application.yaml

# 3. Проверить
argocd app list
argocd app get nginx

# 4. Изменить replicas в Git
sed -i 's/replicas: 3/replicas: 5/' deployment.yaml
git commit -am "Scale to 5"
git push

# 5. ArgoCD подхватит через ~3 минуты
kubectl get pods -l app=nginx
# 5 Pod'ов
```

### 💡 Практика: как правильно использовать ArgoCD

**✅ ОБЯЗАТЕЛЬНО:**

1. **Application** для каждого приложения.
2. **AppProject** для группировки и RBAC.
3. **Sync Policy** — automated для dev, manual для prod.
4. **Notifications** для алертов.

**👍 СТОИТ:**

5. **ApplicationSet** для множества окружений.
6. **RBAC** для команд.
7. **SSO** (Dex) для аутентификации.

**❌ НЕ ДЕЛАЙ:**

8. **Не деплой всё в один Application.** Разделяй по приложениям.
9. **Не используй `automated` без `prune` и `selfHeal`** без понимания.
10. **Не давай всем доступ ко всем Projects.**

### Где мы сейчас

Мы разобрали ArgoCD. Теперь — **Flux** — альтернатива.

---

## 20.5 Flux: альтернатива ArgoCD

### 🔌 Проблема: ArgoCD не единственный

Flux — второй популярный GitOps-инструмент. Чем отличается?

### 📊 Что такое Flux

**Flux** — GitOps-оператор от Weaveworks (теперь CNCF).

**Отличия от ArgoCD:**

| Аспект | ArgoCD | Flux |
|:---|:---|:---|
| **UI** | Богатый | Минимальный (Weave GitOps) |
| **Multi-tenancy** | Через Projects | Через namespaces |
| **Approach** | Application CRD | GitRepository + Kustomization + HelmRelease |
| **RBAC** | Встроенный | Через Kubernetes RBAC |
| **Multi-cluster** | Встроенный | Через отдельные Flux на кластер |
| **Helm** | Поддерживает | HelmRelease — first-class |
| **Kustomize** | Поддерживает | Встроен |

### 🎯 Flux: основные объекты

**1. GitRepository** — источник манифестов.

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/gitops.git
  ref:
    branch: main
  secretRef:
    name: git-credentials    # для приватного репо
```

**2. Kustomization** — что применять.

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 10m
  targetNamespace: production
  sourceRef:
    kind: GitRepository
    name: myapp
  path: ./apps/myapp/overlays/production
  prune: true              # удалять ресурсы, которых нет в Git
  wait: true               # ждать готовности
  timeout: 5m
```

**3. HelmRelease** — Helm-чарт.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: postgresql
  namespace: production
spec:
  interval: 5m
  chart:
    spec:
      chart: postgresql
      version: '12.x.x'
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
  values:
    auth:
      database: myapp
      username: myapp
      password: secret
    primary:
      persistence:
        size: 100Gi
```

**4. HelmRepository** — источник Helm-чартов.

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.bitnami.com/bitnami
```

### 🎯 Flux: архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                    FLUX (в кластере)                         │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │  source-         │  │  kustomize-      │                │
│  │  controller      │  │  controller      │                │
│  │                  │  │                  │                │
│  │  - Клонирует Git │  │  - Применяет     │                │
│  │  - Helm repos    │  │    Kustomization │                │
│  └──────────────────┘  └──────────────────┘                │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │  helm-           │  │  notification-   │                │
│  │  controller      │  │  controller      │                │
│  │                  │  │                  │                │
│  │  - HelmRelease   │  │  - Slack, Teams  │                │
│  └──────────────────┘  └──────────────────┘                │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │  image-          │  │  image-reflector │                │
│  │  automation      │  │  controller      │                │
│  │                  │  │                  │                │
│  │  - Автообновление│  │  - Сканирует     │                │
│  │    образов       │  │    registry      │                │
│  └──────────────────┘  └──────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

**Компоненты:**

| Компонент | Что делает |
|:---|:---|
| **source-controller** | Клонирует Git, Helm repos, OCI |
| **kustomize-controller** | Применяет Kustomization |
| **helm-controller** | Применяет HelmRelease |
| **notification-controller** | Уведомления (Slack, Teams, webhook) |
| **image-automation-controller** | Автообновление образов |
| **image-reflector-controller** | Сканирует registry |

### 🎯 Установка Flux

```bash
# 1. Установить CLI
brew install fluxcd/tap/flux

# 2. Проверить prerequisites
flux check --pre

# 3. Bootstrap (устанавливает Flux и настраивает Git)
export GITHUB_TOKEN=<token>
flux bootstrap github \
  --owner=myorg \
  --repository=gitops \
  --branch=main \
  --path=clusters/production \
  --personal

# Что произойдёт:
# 1. Flux установлен в кластер
# 2. Создан репозиторий gitops (или использован существующий)
# 3. Добавлены манифесты Flux в clusters/production
# 4. Flux синхронизирует себя из Git
```

### 🎯 Flux: пример

**Структура репозитория:**

```
gitops/
├── clusters/
│   └── production/
│       ├── flux-system/       # Flux сам себя
│       ├── apps.yaml          # Kustomization для apps
│       └── infrastructure.yaml
├── apps/
│   ├── base/
│   │   └── myapp/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── kustomization.yaml
│   └── overlays/
│       ├── dev/
│       ├── staging/
│       └── production/
└── infrastructure/
    ├── ingress-nginx/
    ├── cert-manager/
    └── monitoring/
```

**clusters/production/apps.yaml:**

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/overlays/production
  prune: true
  wait: true
```

**apps/overlays/production/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/myapp
images:
  - name: myapp
    newTag: v1.2.3
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
    target:
      kind: Deployment
      name: myapp
```

### 📊 ArgoCD vs Flux

| Аспект | ArgoCD | Flux |
|:---|:---|:---|
| **UI** | Богатый | Минимальный |
| **CLI** | Мощный | Мощный |
| **Multi-tenancy** | AppProject | Namespaces |
| **Helm** | Через Application | HelmRelease (first-class) |
| **Kustomize** | Через Application | Kustomization (first-class) |
| **Multi-cluster** | Встроенный | Через отдельные Flux |
| **Image automation** | Есть | Есть (отдельный контроллер) |
| **Notifications** | Есть | Есть |
| **Сообщество** | Большое | Большое |
| **Кривая обучения** | Средняя | Средняя |

### 🎯 Когда что использовать

**ArgoCD:**

- **Нужен UI** для команды.
- **Multi-cluster** из одного места.
- **Много команд** (Projects, RBAC).
- **Простота.**

**Flux:**

- **Git-native** подход.
- **Helm-first.**
- **Много Flux-инстансов** (per-cluster).
- **Image automation** из коробки.
- **Меньше компонентов.**

**Оба хороши.** Выбор — по предпочтениям.

### 🔬 Практика: Flux

```bash
# 1. Установить Flux CLI
brew install fluxcd/tap/flux

# 2. Bootstrap с GitHub
export GITHUB_TOKEN=<your-token>

flux bootstrap github \
  --owner=myorg \
  --repository=gitops \
  --branch=main \
  --path=clusters/production \
  --personal

# 3. Проверить
flux get all
kubectl get pods -n flux-system

# 4. Создать приложение
mkdir -p apps/base/myapp
cat > apps/base/myapp/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
EOF

cat > apps/base/myapp/kustomization.yaml <<'EOF'
resources:
  - deployment.yaml
EOF

# 5. Создать Kustomization
cat > clusters/production/apps.yaml <<'EOF'
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/base/myapp
  prune: true
  targetNamespace: default
EOF

# 6. Commit и push
git add .
git commit -m "Add nginx"
git push

# 7. Проверить через минуту
flux get kustomizations
kubectl get pods -l app=nginx
```

### 💡 Практика: как выбирать

**✅ ОБЯЗАТЕЛЬНО:**

1. **ArgoCD для команд** с UI-требованиями.
2. **Flux для Git-native** подходов.
3. **Оба хороши** — выбирай по предпочтениям.

**👍 СТОИТ:**

4. **Попробовать оба** перед выбором.
5. **Учитывать экосистему** (Helm, Kustomize).

**❌ НЕ ДЕЛАЙ:**

6. **Не используй оба одновременно** в одном кластере.
7. **Не выбирай без понимания.**

### Где мы сейчас

Мы разобрали Flux. Теперь — **организация репозиториев**.

---

## 20.6 Организация репозиториев

### 🔌 Проблема: где хранить манифесты

В GitOps всё в Git. Но **где** хранить манифесты? **Как** организовать?

### 📊 Два подхода

**1. Monorepo.**

Всё в одном репозитории: код, манифесты, конфиги.

```
mycompany/
├── services/
│   ├── myapp/
│   ├── api/
│   └── worker/
├── gitops/
│   ├── apps/
│   │   ├── myapp/
│   │   ├── api/
│   │   └── worker/
│   └── infrastructure/
└── ...
```

**2. Polyrepo.**

Отдельные репозитории: код в одном, манифесты в другом.

```
myorg/myapp/          # код + Dockerfile
myorg/api/            # код + Dockerfile
myorg/gitops/         # манифесты для всех
myorg/infrastructure/ # Terraform, Ansible
```

### 🎯 Monorepo

**Плюсы:**

- **Одно место.** Легко найти.
- **Атомарные коммиты.** Изменения кода и манифестов вместе.
- **Проще CI/CD.**

**Минусы:**

- **Большой репозиторий.** Медленно клонировать.
- **Права доступа.** Все видят всё.
- **Конфликты.** Много разработчиков.

**Когда использовать:** небольшие команды, тесная связь кода и манифестов.

### 🎯 Polyrepo

**Плюсы:**

- **Изоляция.** Каждая команда — свой репо.
- **Права.** Разные права на разные репо.
- **Меньше конфликтов.**
- **Быстрее клонировать.**

**Минусы:**

- **Два репо.** Изменение кода + изменение манифеста — два PR.
- **Синхронизация.** Нужно связывать версии.
- **Сложнее CI/CD.**

**Когда использовать:** большие команды, много сервисов.

### 🎯 Рекомендуемая структура (Polyrepo)

**Репозиторий кода:**

```
myapp/
├── src/
├── Dockerfile
├── go.mod
├── go.sum
└── .gitlab-ci.yml
```

**Репозиторий манифестов (gitops):**

```
gitops/
├── apps/
│   ├── myapp/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       │   └── kustomization.yaml
│   │       ├── staging/
│   │       │   └── kustomization.yaml
│   │       └── production/
│   │           └── kustomization.yaml
│   ├── api/
│   └── worker/
├── infrastructure/
│   ├── ingress-nginx/
│   ├── cert-manager/
│   ├── monitoring/
│   └── external-secrets/
├── clusters/
│   ├── dev/
│   ├── staging/
│   └── production/
│       ├── flux-system/
│       ├── apps.yaml
│       └── infrastructure.yaml
└── README.md
```

### 🎯 Структура с Kustomize

**Base:**

```
apps/myapp/base/
├── deployment.yaml
├── service.yaml
├── configmap.yaml
└── kustomization.yaml
```

**base/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
commonLabels:
  app: myapp
```

**Overlay dev:**

```
apps/myapp/overlays/dev/
└── kustomization.yaml
```

**overlays/dev/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: dev
resources:
  - ../../base
images:
  - name: myapp
    newTag: latest
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 1
    target:
      kind: Deployment
      name: myapp
```

**Overlay production:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
  - ../../base
images:
  - name: myapp
    newTag: v1.2.3
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
    target:
      kind: Deployment
      name: myapp
```

### 🎯 Структура с Helm

```
gitops/
├── apps/
│   └── myapp/
│       ├── chart/
│       │   ├── Chart.yaml
│       │   ├── values.yaml
│       │   └── templates/
│       ├── values-dev.yaml
│       ├── values-staging.yaml
│       └── values-production.yaml
└── clusters/
    └── production/
        └── myapp.yaml        # HelmRelease
```

### 🎯 Рекомендации

**1. Разделяй apps и infrastructure.**

- `apps/` — приложения.
- `infrastructure/` — ingress, cert-manager, monitoring.
- `clusters/` — конфигурация для каждого кластера.

**2. Используй base + overlays.**

- `base/` — общее.
- `overlays/` — специфичное для окружения.

**3. Отдельные репозитории для секретов.**

- Sealed Secrets или External Secrets.
- Не в основном gitops repo.

**4. Environment branches или overlays.**

- **Branches** — `main`, `staging`, `production`.
- **Overlays** — одна ветка, разные overlays.

**Рекомендация:** overlays. Одна ветка, всё видно.

### 🔬 Практика: структура

```bash
# 1. Создать структуру
mkdir -p gitops/{apps,infrastructure,clusters}
mkdir -p gitops/apps/myapp/{base,overlays/{dev,staging,production}}

# 2. Base
cat > gitops/apps/myapp/base/deployment.yaml <<'EOF'
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
          image: myapp:latest
          ports:
            - containerPort: 8080
EOF

cat > gitops/apps/myapp/base/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
EOF

# 3. Overlay production
cat > gitops/apps/myapp/overlays/production/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
  - ../../base
images:
  - name: myapp
    newTag: v1.2.3
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
    target:
      kind: Deployment
      name: myapp
EOF

# 4. Проверить рендеринг
kubectl kustomize gitops/apps/myapp/overlays/production

# 5. Структура
tree gitops/
```

### 💡 Практика: как организовать репозитории

**✅ ОБЯЗАТЕЛЬНО:**

1. **Отдельный gitops-репозиторий** (polyrepo).
2. **`apps/` и `infrastructure/`.**
3. **Base + overlays.**
4. **Одна ветка (main).**

**👍 СТОИТ:**

5. **Kustomize** для overlays.
6. **Helm** для сложных приложений.
7. **Отдельный репо для секретов.**

**❌ НЕ ДЕЛАЙ:**

8. **Не смешивай код и манифесты.** Отдельные репо.
9. **Не дублируй манифесты.** Base + overlays.
10. **Не коммить секреты** без шифрования.

### Где мы сейчас

Мы разобрали организацию репозиториев. Теперь — **drift detection**.

---

## 20.7 Drift detection и self-healing

### 🔌 Проблема: кто-то изменил ресурс вручную

Разработчик в 2 часа ночи срочно поправил Deployment через `kubectl edit`. Увеличил replicas. Или изменил лимиты. Или поменял образ.

**Теперь состояние кластера ≠ состоянию в Git.**

**Drift.**

**Решение:** GitOps-агент обнаруживает и исправляет.

### 📊 Что такое drift

**Drift** — расхождение между:

- **Желаемым состоянием** (Git).
- **Текущим состоянием** (кластер).

**Примеры:**

- Кто-то изменил replicas через `kubectl edit`.
- Кто-то удалил Deployment.
- Кто-то изменил ConfigMap.
- Автоскейлер изменил replicas (это нормально, но drift).
- Cluster Autoscaler изменил nodes.

### 🎯 Как работает drift detection

**ArgoCD:**

1. Каждые 3 минуты (по умолчанию) ArgoCD сравнивает Git и кластер.
2. Если расхождение — статус `OutOfSync`.
3. Если `selfHeal: true` — ArgoCD применяет Git-состояние.
4. Если `selfHeal: false` — только показывает (нужно нажать Sync).

**Flux:**

1. Каждые 10 минут (по умолчанию) Flux сравнивает.
2. Если расхождение — применяет (если `prune: true`).
3. Можно настроить `suspend` для остановки.

### 🎯 Self-healing

**Self-healing** — автоматический откат drift'а.

**ArgoCD:**

```yaml
syncPolicy:
  automated:
    selfHeal: true
```

**Что произойдёт:**

- Изменил replicas вручную → ArgoCD вернёт к Git-значению.
- Удалил Deployment → ArgoCD создаст заново.
- Изменил ConfigMap → ArgoCD вернёт.

**Flux:**

```yaml
spec:
  prune: true
  # Flux всегда применяет Git-состояние
```

### 🎯 Когда использовать self-healing

**✅ Использовать:**

- **Production** — гарантия соответствия Git.
- **Критичные ресурсы** — Deployments, Services, ConfigMaps.
- **Когда команда дисциплинирована** — не делает ручных изменений.

**❌ Не использовать:**

- **Dev** — иногда нужно быстро поправить.
- **Когда автоскейлер управляет ресурсами** — HPA меняет replicas.
- **Во время отладки** — self-heal мешает.

### 🎯 Исключения из self-healing

**HPA и replicas:**

Если используешь HPA, он управляет replicas. ArgoCD будет считать это drift'ом.

**Решение 1:** убрать `replicas` из Git.

```yaml
# ❌ Плохо: replicas в Git
spec:
  replicas: 3

# ✅ Хорошо: replicas управляется HPA
spec:
  # replicas не указан
```

**Решение 2:** `ignoreDifferences` в ArgoCD.

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

**Что произойдёт:** ArgoCD игнорирует изменения в replicas.

**Другие исключения:**

- **Cluster Autoscaler** — меняет nodes.
- **MutatingWebhooks** — меняют ресурсы при создании.
- **Default values** — Kubernetes добавляет defaults.

**Игнорирование:**

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
    - group: ''
      kind: Service
      jsonPointers:
        - /spec/clusterIP
```

### 🎯 Manual sync

**Для production** часто используют **manual sync**:

```yaml
syncPolicy: {}
```

**Что произойдёт:**

- ArgoCD показывает `OutOfSync`.
- Ничего не применяет.
- Ты нажимаешь «Sync» в UI или `argocd app sync myapp`.

**Плюсы:**

- **Контроль.** Ты решаешь, когда применять.
- **Ревью.** Видишь diff перед применением.
- **Безопасность.** Никаких неожиданных изменений.

**Минусы:**

- **Ручной.** Нужно нажимать.
- **Медленнее.**

**Гибридный подход:**

- **Auto-sync для dev/staging.**
- **Manual sync для production.**

### 🎯 Notifications о drift

**ArgoCD Notifications:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  trigger.on-sync-status-unknown: |
    - when: app.status.sync.status == 'Unknown'
      send: [slack-notification]
  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [slack-notification]
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [slack-notification]
  
  service.slack: |
    token: $slack-token
  
  template.slack-notification: |
    message: |
      Application {{.app.metadata.name}} has sync/health issue.
      Sync Status: {{.app.status.sync.status}}
      Health Status: {{.app.status.health.status}}
```

**Что даёт:** уведомления в Slack при проблемах.

### 🔬 Практика: drift detection

```bash
# 1. Развернуть приложение через ArgoCD
argocd app create myapp \
  --repo https://github.com/myorg/gitops.git \
  --path apps/myapp \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# 2. Проверить статус
argocd app get myapp
# Sync Status: Synced
# Health Status: Healthy

# 3. Изменить replicas вручную
kubectl scale deployment myapp --replicas=10

# 4. Проверить статус (через минуту)
argocd app get myapp
# Sync Status: OutOfSync
# (показывает diff)

# 5. Self-heal откатит
kubectl get deployment myapp
# REPLICAS: 3 (вернулось)

# 6. Посмотреть diff
argocd app diff myapp
```

### 💡 Практика: как правильно работать с drift

**✅ ОБЯЗАТЕЛЬНО:**

1. **`selfHeal: true` для production.**
2. **`prune: true` для удаления orphaned ресурсов.**
3. **Notifications о drift.**

**👍 СТОИТ:**

4. **`ignoreDifferences` для HPA, mutating webhooks.**
5. **Manual sync для критичных ресурсов.**
6. **Регулярный аудит drift.**

**❌ НЕ ДЕЛАЙ:**

7. **Не делай ручных изменений** после внедрения GitOps.
8. **Не игнорируй `OutOfSync`.**
9. **Не используй `selfHeal` без понимания** — может конфликтовать с HPA.

### Где мы сейчас

Мы разобрали drift detection. Теперь — **promotion между окружениями**.

---

## 20.8 Promotion между окружениями

### 🔌 Проблема: как продвигать версию из dev в prod

Ты задеплоил v1.2.3 в dev. Протестировал. Теперь нужно в staging, потом в prod.

**Как это делать в GitOps?**

### 📊 Стратегии promotion

**1. Overlays (рекомендуется).**

Одна ветка `main`, разные overlays для окружений.

```
apps/myapp/
├── base/
└── overlays/
    ├── dev/          → v1.2.3
    ├── staging/      → v1.2.2
    └── production/   → v1.2.1
```

**Promotion:**

1. Обновить `overlays/dev/kustomization.yaml` → v1.2.3.
2. PR, merge.
3. ArgoCD применяет в dev.
4. Тестируешь.
5. Обновить `overlays/staging` → v1.2.3.
6. PR, merge.
7. ...
8. Обновить `overlays/production` → v1.2.3.

**Плюсы:**

- **Всё видно.** Один файл — все версии.
- **Просто.**

**Минусы:**

- **Много PR'ов.**
- **Ручной процесс.**

**2. Branches.**

Отдельные ветки: `main`, `staging`, `production`.

**Promotion:** merge `main` → `staging` → `production`.

**Плюсы:**

- **Git-native.**
- **Автоматизация через merge.**

**Минусы:**

- **Merge-конфликты.**
- **Сложнее синхронизировать.**

**3. Image tags.**

Одна ветка, разные теги в overlays.

**Promotion:** обновление тега.

**4. Git tags.**

Отдельные git tags для окружений.

**Promotion:** создать tag.

### 🎯 Стратегия с overlays (детально)

**Структура:**

```
apps/myapp/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml      # image: v1.2.3
    ├── staging/
    │   └── kustomization.yaml      # image: v1.2.2
    └── production/
        └── kustomization.yaml      # image: v1.2.1
```

**CI/CD:**

```yaml
# В репозитории кода
update-manifests:
  stage: deploy
  script:
    # Обновить только dev
    - git clone https://oauth2:$TOKEN@gitlab.com/org/gitops.git
    - cd gitops
    - |
      cd apps/myapp/overlays/dev
      kustomize edit set image myapp=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - git commit -am "Update myapp dev to $CI_COMMIT_SHA"
    - git push
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

**Что произошло:**

- CI обновил только dev.
- ArgoCD применил в dev.
- Тестирование.

**Promotion в staging:**

**Вариант 1: Вручную через PR.**

```bash
# Создать PR
git checkout -b promote-to-staging
cd apps/myapp/overlays/staging
kustomize edit set image myapp=myregistry.com/myapp:v1.2.3
git commit -am "Promote myapp to staging: v1.2.3"
git push
# Создать PR в main
```

**Вариант 2: Автоматизация через CLI.**

```bash
# Скрипт promotion
./promote.sh myapp staging v1.2.3
```

**promote.sh:**

```bash
#!/bin/bash
APP=$1
ENV=$2
VERSION=$3

cd gitops/apps/$APP/overlays/$ENV
kustomize edit set image $APP=myregistry.com/$APP:$VERSION

git checkout -b "promote-$APP-$ENV-$VERSION"
git add .
git commit -m "Promote $APP to $ENV: $VERSION"
git push origin "promote-$APP-$ENV-$VERSION"

# Создать PR через gh/glab
gh pr create --title "Promote $APP to $ENV: $VERSION" \
  --body "Promoting $APP to $ENV with version $VERSION"
```

**Вариант 3: ArgoCD Image Updater.**

**ArgoCD Image Updater** — контроллер, который автоматически обновляет image tag.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=myregistry.com/myapp
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver
    argocd-image-updater.argoproj.io/myapp.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
    argocd-image-updater.argoproj.io/write-back-method: git
```

**Что произойдёт:** Image Updater находит новый тег → обновляет Git.

**Вариант 4: Flux Image Automation.**

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImagePolicy
metadata:
  name: myapp
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: myapp
  policy:
    semver:
      range: '>=1.0.0'
```

**Что произойдёт:** Flux обновляет Git при новом образе.

### 🎯 Ручной promotion через PR

**Workflow:**

1. **CI обновляет dev** автоматически.
2. **QA тестирует** dev.
3. **QA создаёт PR** для promotion в staging.
4. **Ревьюер проверяет** diff.
5. **Merge.**
6. **ArgoCD применяет в staging.**
7. **QA тестирует** staging.
8. **QA создаёт PR** для promotion в production.
9. **Ревьюер + approval.**
10. **Merge.**
11. **ArgoCD применяет в production.**

**Плюсы:**

- **Ревью.**
- **Audit trail.**
- **Контроль.**

### 🎯 Автоматический promotion

**Для быстрых команд:**

1. **CI обновляет dev.**
2. **Автотесты** проходят.
3. **Автоматический PR** для staging.
4. **Merge автоматом.**
5. **Автотесты** проходят.
6. **Автоматический PR** для production.
7. **Manual approval.**
8. **Merge.**

**Инструменты:**

- **Keptn** — оркестрация promotion.
- **Argo Rollouts** — progressive delivery.
- **Flux image automation.**

### 🔬 Практика: promotion

```bash
# 1. Структура
mkdir -p gitops/apps/myapp/overlays/{dev,staging,production}

# 2. Overlays
cat > gitops/apps/myapp/overlays/dev/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: dev
resources:
  - ../../base
images:
  - name: myapp
    newTag: v1.2.3
EOF

cat > gitops/apps/myapp/overlays/staging/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: staging
resources:
  - ../../base
images:
  - name: myapp
    newTag: v1.2.2
EOF

cat > gitops/apps/myapp/overlays/production/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
  - ../../base
images:
  - name: myapp
    newTag: v1.2.1
EOF

# 3. Promotion скрипт
cat > promote.sh <<'EOF'
#!/bin/bash
set -e

APP=$1
ENV=$2
VERSION=$3

if [ -z "$APP" ] || [ -z "$ENV" ] || [ -z "$VERSION" ]; then
  echo "Usage: $0 <app> <env> <version>"
  exit 1
fi

cd "gitops/apps/$APP/overlays/$ENV"
kustomize edit set image "$APP=myregistry.com/$APP:$VERSION"

cd - > /dev/null

BRANCH="promote-$APP-$ENV-$VERSION"
git checkout -b "$BRANCH"
git add .
git commit -m "Promote $APP to $ENV: $VERSION"
git push origin "$BRANCH"

echo "Created branch $BRANCH. Create PR."
EOF

chmod +x promote.sh

# 4. Promotion
./promote.sh myapp staging v1.2.3
```

### 💡 Практика: как правильно делать promotion

**✅ ОБЯЗАТЕЛЬНО:**

1. **Overlays** для окружений.
2. **PR для promotion.**
3. **Manual approval для production.**

**👍 СТОИТ:**

4. **Автоматизация через ArgoCD Image Updater** или Flux Image Automation.
5. **Automatic tests** перед promotion.
6. **Keptn** для сложных workflow.

**❌ НЕ ДЕЛАЙ:**

7. **Не делай promotion через `kubectl` вручную.**
8. **Не пропускай staging.**
9. **Не делай promotion без тестов.**

### Где мы сейчас

Мы разобрали promotion. Теперь — **Blue-Green и Canary в GitOps**.

---

## 20.9 Blue-Green и Canary через GitOps

### 🔌 Проблема: как делать Blue-Green и Canary в GitOps

В Главе 13 мы разбирали Blue-Green и Canary в Kubernetes. Как это работает в GitOps?

### 📊 Blue-Green в GitOps

**Подход:** два Deployment'а, переключение Service.

**Структура:**

```
apps/myapp/
├── base/
│   ├── service.yaml           # selector: version=blue
│   └── kustomization.yaml
└── overlays/
    ├── blue/
    │   ├── deployment-blue.yaml
    │   └── kustomization.yaml
    └── green/
        ├── deployment-green.yaml
        └── kustomization.yaml
```

**Service (base):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue       # ← переключение здесь
```

**Deployment blue:**

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
          image: myapp:v1.2.3
```

**Deployment green:**

```yaml
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
          image: myapp:v1.3.0
```

**Cutover:**

```bash
# Изменить selector в base/service.yaml
sed -i 's/version: blue/version: green/' base/service.yaml
git commit -am "Cutover to green"
git push
# ArgoCD применит
```

**Rollback:**

```bash
# Изменить обратно
sed -i 's/version: green/version: blue/' base/service.yaml
git commit -am "Rollback to blue"
git push
```

### 📊 Canary в GitOps

**Подход:** два Deployment'а, оба под одним Service, но с разным количеством реплик.

**Структура:**

```
apps/myapp/
├── base/
│   ├── service.yaml          # selector: app=myapp
│   └── kustomization.yaml
└── overlays/
    └── production/
        ├── deployment-stable.yaml    # 9 реплик
        ├── deployment-canary.yaml    # 1 реплика
        └── kustomization.yaml
```

**Service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp        # выбирает и stable, и canary
```

**Stable (9 реплик):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9
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
          image: myapp:v1.2.3
```

**Canary (1 реплика = 10%):**

```yaml
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
          image: myapp:v1.3.0
```

**Что произойдёт:** Service отправляет трафик на 10 Pod'ов (9 stable + 1 canary). 10% трафика — на canary.

**Увеличение canary:**

```bash
# Изменить replicas в canary: 1 → 3 (25%)
sed -i 's/replicas: 1/replicas: 3/' deployment-canary.yaml
git commit -am "Increase canary to 25%"
git push
```

**Полный rollout:**

```bash
# Увеличить canary до 10, уменьшить stable до 0
sed -i 's/replicas: 3/replicas: 10/' deployment-canary.yaml
sed -i 's/replicas: 9/replicas: 0/' deployment-stable.yaml
git commit -am "Promote canary to 100%"
git push
```

**Rollback:**

```bash
# Удалить canary
git rm deployment-canary.yaml
git commit -am "Rollback canary"
git push
```

### 🎯 Argo Rollouts в GitOps

**Argo Rollouts** — CRD для продвинутых деплой-стратегий.

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

**Что даёт:**

- **Progressive delivery** из коробки.
- **Автоматический rollback** при плохих метриках.
- **GitOps-friendly** — всё в Git.

**В GitOps:**

1. Обновить image в Rollout.
2. Git push.
3. ArgoCD применяет.
4. Rollout постепенно увеличивает трафик.
5. Анализирует метрики.
6. Если плохо — откатывает.

### 🎯 Flagger в GitOps

**Flagger** — оператор для автоматического canary.

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
    stepWeight: 5
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99
        interval: 1m
```

**В GitOps:**

1. Обновить image в Deployment.
2. Git push.
3. ArgoCD применяет.
4. Flagger видит новый image.
5. Создаёт canary.
6. Постепенно увеличивает трафик.
7. Анализирует метрики.
8. Если плохо — откатывает.

### 🔬 Практика: Blue-Green в GitOps

```bash
# 1. Base
mkdir -p gitops/apps/myapp/base
cat > gitops/apps/myapp/base/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue
  ports:
    - port: 80
      targetPort: 8080
EOF

# 2. Blue
cat > gitops/apps/myapp/base/deployment-blue.yaml <<'EOF'
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
          image: myapp:v1.2.3
          ports:
            - containerPort: 8080
EOF

# 3. Green
cat > gitops/apps/myapp/base/deployment-green.yaml <<'EOF'
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
          image: myapp:v1.3.0
          ports:
            - containerPort: 8080
EOF

# 4. kustomization
cat > gitops/apps/myapp/base/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - service.yaml
  - deployment-blue.yaml
  - deployment-green.yaml
EOF

# 5. Cutover
sed -i 's/version: blue/version: green/' gitops/apps/myapp/base/service.yaml
git commit -am "Cutover to green"
git push

# 6. Rollback
sed -i 's/version: green/version: blue/' gitops/apps/myapp/base/service.yaml
git commit -am "Rollback to blue"
git push
```

### 💡 Практика: как делать Blue-Green и Canary

**✅ ОБЯЗАТЕЛЬНО:**

1. **Blue-Green через Service selector.**
2. **Canary через replicas ratio.**
3. **Argo Rollouts/Flagger** для автоматизации.

**👍 СТОИТ:**

4. **Analysis** для автоматического rollback.
5. **Notifications** при проблемах.
6. **Prometheus метрики** для анализа.

**❌ НЕ ДЕЛАЙ:**

7. **Не делай Blue-Green без тестов.**
8. **Не оставляй старый Deployment навсегда.**
9. **Не используй Canary без метрик.**

### Где мы сейчас

Мы разобрали Blue-Green и Canary. Теперь — **Secrets в GitOps**.

---

## 20.10 Secrets в GitOps

### 🔌 Проблема: как хранить секреты в Git

GitOps требует, чтобы **всё** было в Git. Но секреты в Git — компрометация.

**Решение:** шифрование или внешние менеджеры.

### 📊 Подходы

**1. Sealed Secrets.**

Шифрование через публичный ключ кластера. Расшифровка в контроллере.

**2. External Secrets Operator.**

Секреты в Vault/AWS SM, ExternalSecret в Git.

**3. SOPS.**

Шифрование через KMS/age. Расшифровка при деплое.

**4. Vault + CSI Driver.**

Vault Secrets Store CSI Driver для монтирования секретов.

**5. age + Kustomize.**

Шифрование через `sops` + `kustomize`.

### 🎯 Sealed Secrets в GitOps

**1. Установить Sealed Secrets Controller:**

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system
```

**2. Создать Secret:**

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: production
type: Opaque
stringData:
  password: SuperSecret123
```

**3. Зашифровать:**

```bash
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
```

**4. Закоммитить в Git:**

```yaml
# sealed-secret.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-secret
  namespace: production
spec:
  encryptedData:
    password: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
```

**5. ArgoCD применяет:**

ArgoCD применяет SealedSecret. Controller расшифровывает → Secret.

**Плюсы:**

- **Просто.**
- **Всё в Git.**
- **Привязка к namespace.**

**Минусы:**

- **Нельзя переиспользовать** в другом кластере (разный ключ).
- **Ротация сложнее.**

### 🎯 External Secrets в GitOps

**1. Установить External Secrets Operator:**

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

**2. SecretStore (в Git):**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault
  namespace: production
spec:
  provider:
    vault:
      server: https://vault.example.com
      path: secret
      auth:
        kubernetes:
          mountPath: kubernetes
          role: myapp
```

**3. ExternalSecret (в Git):**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault
    kind: SecretStore
  target:
    name: db-secret
  data:
    - secretKey: password
      remoteRef:
        key: secret/data/db
        property: password
```

**4. ArgoCD применяет:**

ESO читает секрет из Vault → создаёт Secret.

**Плюсы:**

- **Секреты в Vault**, не в Git.
- **Централизованное управление.**
- **Ротация через Vault.**
- **Audit.**

**Минусы:**

- **Нужен Vault.**
- **Сложнее настройка.**

### 🎯 SOPS + age в GitOps

**1. Установить SOPS и age:**

```bash
brew install sops age
```

**2. Создать age key:**

```bash
age-keygen -o key.txt
# Public key: age1abc123...
```

**3. Настроить `.sops.yaml`:**

```yaml
creation_rules:
  - path_regex: .*\.enc\.yaml$
    age: age1abc123...
```

**4. Зашифровать secret:**

```bash
sops -e secret.yaml > secret.enc.yaml
```

**5. Расшифровка в ArgoCD:**

**Вариант 1: Плагин.**

ArgoCD Config Management Plugin для расшифровки.

**Вариант 2: Pre-commit hook.**

Расшифровка перед применением (но это уже не GitOps).

**Вариант 3: Flux.**

Flux имеет встроенную поддержку SOPS.

**Flux с SOPS:**

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

### 🎯 Vault Secrets Store CSI Driver

**Монтирует секреты из Vault как volumes в Pod'ы.**

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-db
spec:
  provider: vault
  parameters:
    vaultAddress: https://vault.example.com
    roleName: myapp
    objects: |
      - objectName: "password"
        secretPath: "secret/data/db"
        secretKey: "password"
---
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      volumeMounts:
        - name: secrets
          mountPath: /mnt/secrets
          readOnly: true
  volumes:
    - name: secrets
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: vault-db
```

**Плюсы:**

- **Секреты не в etcd.**
- **Динамические секреты.**
- **Ротация.**

**Минусы:**

- **Сложно.**
- **Нужен Vault.**

### 📊 Сравнение

| Подход | Секреты в Git | Сложность | Ротация | Best for |
|:---|:---|:---|:---|:---|
| **Sealed Secrets** | Зашифрованы | Низкая | Ручная | Небольшие команды |
| **External Secrets** | Нет (ссылки) | Средняя | Авто | Production |
| **SOPS** | Зашифрованы | Средняя | Ручная | Flux, Kustomize |
| **Vault CSI** | Нет | Высокая | Авто | Enterprise |

### 🎯 Рекомендация

- **Небольшая команда:** Sealed Secrets.
- **Production:** External Secrets + Vault.
- **Flux:** SOPS.
- **Enterprise:** Vault CSI.

### 🔬 Практика: Sealed Secrets в GitOps

```bash
# 1. Установить Sealed Secrets
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# 2. Установить kubeseal
brew install kubeseal

# 3. Создать Secret
cat > secret.yaml <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: production
type: Opaque
stringData:
  password: SuperSecret123
EOF

# 4. Зашифровать
kubeseal --format yaml < secret.yaml > gitops/apps/myapp/overlays/production/sealed-secret.yaml

# 5. Удалить оригинал
rm secret.yaml

# 6. Коммит
git add gitops/apps/myapp/overlays/production/sealed-secret.yaml
git commit -m "Add sealed db secret"
git push

# 7. ArgoCD применит
# 8. Проверить
kubectl get secret db-secret -n production
kubectl get secret db-secret -n production -o jsonpath='{.data.password}' | base64 -d
# SuperSecret123
```

### 💡 Практика: как правильно работать с секретами

**✅ ОБЯЗАТЕЛЬНО:**

1. **Sealed Secrets для простоты.**
2. **External Secrets для production.**
3. **Никогда не коммить plain secrets.**

**👍 СТОИТ:**

4. **Vault** для централизованного управления.
5. **Ротация секретов.**
6. **Разные секреты для окружений.**

**❌ НЕ ДЕЛАЙ:**

7. **Не коммить secrets без шифрования.**
8. **Не используй один secret для всех окружений.**
9. **Не храни Vault token в Git.**

### Где мы сейчас

Мы разобрали Secrets. Теперь — **disaster recovery**.

---

## 20.11 Disaster recovery через Git

### 🔌 Проблема: кластер потерян

Кластер полностью упал. Все ресурсы потеряны. Как восстановить?

**С GitOps — из Git.**

### 📊 Disaster recovery

**Сценарии:**

1. **Кластер полностью потерян** (data center сгорел).
2. **Namespace удалён** по ошибке.
3. **etcd повреждён.**
4. **Ransomware** зашифровал кластер.

**Что нужно:**

1. **Git** с манифестами.
2. **Backup etcd** (для stateful данных).
3. **Backup volumes** (для PV).
4. **Registry** с образами.
5. **Секреты** (Vault, Sealed Secrets ключи).

### 🎯 Восстановление кластера

**1. Создать новый кластер.**

**2. Установить ArgoCD/Flux.**

**3. Bootstrap:**

```bash
# ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Или Flux
flux bootstrap github --owner=myorg --repository=gitops --path=clusters/production
```

**4. ArgoCD применяет все Applications из Git:**

- Namespaces.
- Deployments.
- Services.
- ConfigMaps.
- Secrets (если Sealed Secrets).
- Ingress.

**5. Восстановить данные:**

- **StatefulSets** — PVC создадутся заново.
- **Данные** — из бэкапов (Velero, pg_dump).

### 🎯 Velero + GitOps

**Velero** — для бэкапа ресурсов и volumes.

```bash
# Установка Velero
velero install --provider aws --bucket my-backups ...

# Бэкап
velero backup create production-backup --include-namespaces production

# Восстановление в новый кластер
velero restore create --from-backup production-backup
```

**Что бэкапит Velero:**

- Все ресурсы namespace.
- PVC через snapshots.
- Secrets.

**Комбинирование с GitOps:**

- **Velero** — для stateful данных (PVC).
- **GitOps** — для конфигурации (Deployments, Services).

### 🎯 Rehearsal (тестирование DR)

**Ключевое правило:** **DR без тестирования — не DR.**

**Что тестировать:**

1. **Восстановление в тестовый кластер.**
2. **Время восстановления (RTO).**
3. **Потери данных (RPO).**
4. **Полнота восстановления.**

**Регулярность:**

- **Раз в квартал** — полный DR drill.
- **Раз в месяц** — частичное восстановление.

### 🎯 Backup Git

**Проблема:** Git-репозиторий — единственный источник правды. Если он потерян — потеряно всё.

**Решение:** backup Git.

- **Mirror** в другой Git-сервер (GitHub → GitLab).
- **Backup** репозитория в S3.
- **Multiple remotes.**

```bash
# Mirror
git clone --mirror https://github.com/myorg/gitops.git
cd gitops.git
git remote add gitlab https://gitlab.com/myorg/gitops.git
git push --mirror gitlab
```

### 🎯 Backup Sealed Secrets ключ

**Проблема:** Sealed Secrets шифрует ключом кластера. Если ключ потерян — не расшифруешь.

**Решение:** backup ключ.

```bash
# Экспорт ключа
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-key.yaml

# Сохранить в безопасное место (Vault, S3 с шифрованием)
```

### 🔬 Практика: DR

```bash
# 1. Backup etcd (для self-hosted)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 2. Backup Velero
velero backup create full-backup --include-namespaces production,staging,dev

# 3. Backup Git
git clone --mirror https://github.com/myorg/gitops.git
aws s3 sync gitops.git s3://my-backups/gitops/

# 4. Backup Sealed Secrets keys
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-keys.yaml
# Закоммитить в Vault

# 5. Восстановление
# - Новый кластер
# - Установить ArgoCD/Flux
# - ArgoCD подхватит из Git
# - Velero restore для PVC
# - Restore sealed-secrets keys
```

### 💡 Практика: как правильно делать DR

**✅ ОБЯЗАТЕЛЬНО:**

1. **Git — источник правды.**
2. **Backup etcd** для stateful данных.
3. **Velero** для volumes.
4. **Backup Git.**
5. **Backup Sealed Secrets keys.**
6. **Регулярные DR drills.**

**👍 СТОИТ:**

4. **Multi-region Git.**
5. **Multi-region кластеры.**
6. **Автоматизация DR.**

**❌ НЕ ДЕЛАЙ:**

7. **Не полагайся только на Git.** Stateful данные в PVC.
8. **Не забывай про backup Git.**
9. **Не тестируй DR только на бумаге.**

### Где мы сейчас

Мы разобрали DR. Теперь — **миграция с push-based на GitOps**.

---

## 20.12 Миграция с push-based на GitOps

### 🔌 Проблема: как перейти на GitOps

У тебя уже есть production с push-based CI/CD. Как перейти на GitOps без простоя?

### 📊 Стратегия миграции

**Фаза 1: Подготовка (2-4 недели).**

1. **Организовать репозиторий** манифестов.
2. **Установить ArgoCD/Flux** в отдельный namespace.
3. **Начать с dev-окружения.**

**Фаза 2: Dev (2-4 недели).**

4. **Перенести dev-приложения** в GitOps.
5. **Тестировать** drift detection, self-healing.
6. **Обучить команду.**

**Фаза 3: Staging (2-4 недели).**

7. **Перенести staging.**
8. **Настроить promotion workflow.**
9. **Тестировать rollback через Git.**

**Фаза 4: Production (4-8 недель).**

10. **Перенести production** по одному приложению.
11. **Manual sync** сначала.
12. **Auto-sync** после стабилизации.

**Фаза 5: Отключение CI/CD деплоя.**

13. **Убрать `kubectl apply`** из CI.
14. **CI только обновляет Git.**

### 🎯 Пошаговая миграция

**1. Экспорт текущего состояния.**

```bash
# Экспорт всех ресурсов namespace
kubectl get all -n production -o yaml > production-export.yaml

# Или через kubectl-neat для очистки
kubectl get all -n production -o yaml | kubectl neat > production-neat.yaml
```

**2. Организация в Git.**

```
gitops/
└── apps/
    └── myapp/
        ├── base/
        │   ├── deployment.yaml
        │   ├── service.yaml
        │   └── kustomization.yaml
        └── overlays/
            └── production/
                └── kustomization.yaml
```

**3. Установка ArgoCD.**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**4. Создание Application.**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops.git
    targetRevision: main
    path: apps/myapp/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy: {}    # manual sync сначала
```

**5. Синхронизация.**

```bash
# Проверить diff
argocd app diff myapp

# Если diff пустой — Git совпадает с кластером
# Если нет — исправить манифесты

# Синхронизировать
argocd app sync myapp
```

**6. Отключить CI/CD деплой.**

```yaml
# .gitlab-ci.yml
deploy:
  stage: deploy
  script:
    # Убрать kubectl apply
    # Только обновить манифесты в GitOps-репо
    - git clone https://oauth2:$TOKEN@gitlab.com/org/gitops.git
    - cd gitops
    - sed -i "s|image: myapp:.*|image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA|" apps/myapp/overlays/production/kustomization.yaml
    - git commit -am "Update myapp to $CI_COMMIT_SHA"
    - git push
```

**7. Включить auto-sync.**

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

### 🎯 Импорт существующих ресурсов

**Если ресурсы уже в кластере:**

1. **Экспортировать YAML** (kubectl get -o yaml).
2. **Очистить** (kubectl-neat).
3. **Убрать** `status`, `metadata.uid`, `metadata.resourceVersion` и т.д.
4. **Закоммитить в Git.**
5. **Создать Application** с `syncPolicy: {}`.
6. **Проверить diff** — должен быть пустой.
7. **Включить auto-sync.**

**Важно:** Git должен **точно соответствовать** кластеру. Иначе ArgoCD покажет diff и попытается изменить.

### 🎯 Постепенный переход

**Не мигрируй всё сразу.** По одному приложению:

1. **Выбрать приложение.**
2. **Экспортировать в Git.**
3. **Создать Application (manual sync).**
4. **Проверить diff.**
5. **Sync.**
6. **Мониторить неделю.**
7. **Включить auto-sync.**
8. **Следующее приложение.**

### 🔬 Практика: миграция

```bash
# 1. Экспорт
kubectl get deployment myapp -n production -o yaml > myapp-deployment.yaml
kubectl get service myapp -n production -o yaml > myapp-service.yaml

# 2. Очистка
# Убрать: metadata.uid, metadata.resourceVersion, metadata.creationTimestamp, metadata.generation, status
# Оставить: apiVersion, kind, metadata.name, metadata.namespace, metadata.labels, spec

# 3. Структура в Git
mkdir -p gitops/apps/myapp/{base,overlays/production}

# myapp-deployment.yaml → base/
# myapp-service.yaml → base/

# 4. base/kustomization.yaml
cat > gitops/apps/myapp/base/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
EOF

# 5. overlays/production/kustomization.yaml
cat > gitops/apps/myapp/overlays/production/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
  - ../../base
EOF

# 6. Commit
git add .
git commit -m "Import myapp from production"
git push

# 7. Application (manual sync)
cat > argocd/application.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops.git
    targetRevision: main
    path: apps/myapp/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy: {}
EOF

kubectl apply -f argocd/application.yaml

# 8. Diff
argocd app diff myapp

# 9. Sync
argocd app sync myapp

# 10. Включить auto-sync после недели мониторинга
```

### 💡 Практика: как правильно мигрировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Постепенно** — по одному приложению.
2. **Начинать с dev.**
3. **Manual sync сначала.**
4. **Проверять diff** перед sync.
5. **Мониторить** после миграции.

**👍 СТОИТ:**

4. **Обучение команды.**
5. **Документация.**
6. **Rollback план.**

**❌ НЕ ДЕЛАЙ:**

7. **Не мигрируй всё сразу.**
8. **Не включай auto-sync без тестов.**
9. **Не забывай про backup.**

### Где мы сейчас

Мы разобрали миграцию. Теперь — **диагностика**.

---

## 20.13 Диагностика проблем

### 🔌 Проблема: ArgoCD не синхронизирует

Application в статусе `OutOfSync`. Или `Unknown`. Или `Failed`. Как диагностировать?

### 🔍 Типичные проблемы

**1. `OutOfSync` — Git ≠ кластер.**

**Причины:**

- Кто-то изменил ресурс вручную (drift).
- Изменения в Git ещё не применены.
- Ошибка в манифестах.

**Диагностика:**

```bash
# Diff
argocd app diff myapp

# Или через UI — вкладка App Diff
```

**Решение:**

- **Sync** — применить Git.
- **Или откатить** drift.

**2. `Unknown` — не может получить статус.**

**Причины:**

- Проблемы с кластером.
- ArgoCD не может подключиться к API.
- Неправильный namespace.

**Диагностика:**

```bash
# Логи ArgoCD
kubectl logs -n argocd deployment/argocd-application-controller

# Проверить подключение к кластеру
argocd cluster list
```

**3. `Failed` — sync не удался.**

**Причины:**

- Ошибка в манифесте.
- Не хватает прав.
- Конфликт ресурсов.

**Диагностика:**

```bash
# Sync status
argocd app get myapp

# Логи операции
argocd app logs myapp
```

**4. `Degraded` — ресурсы не работают.**

**Причины:**

- Pod'ы не запускаются.
- Проблемы с образами.
- Проблемы с PVC.

**Диагностика:**

```bash
kubectl get pods -n production
kubectl describe pod <pod>
kubectl logs <pod>
```

**5. Application не появляется.**

**Причины:**

- ArgoCD не установлен.
- Неправильный namespace.
- Ошибка в Application.

**Диагностика:**

```bash
kubectl get pods -n argocd
kubectl get applications -n argocd
kubectl describe application myapp -n argocd
```

**6. `ComparisonError`.**

**Причины:**

- Ошибка при рендеринге манифестов (Helm, Kustomize).
- Неправильный path.
- Проблемы с Git-репозиторием.

**Диагностика:**

```bash
# Логи repo-server
kubectl logs -n argocd deployment/argocd-repo-server

# Проверить вручную
git clone <repo>
cd <repo>
kubectl kustomize <path>
```

### 🎯 Логи ArgoCD

```bash
# API Server
kubectl logs -n argocd deployment/argocd-server

# Application Controller
kubectl logs -n argocd deployment/argocd-application-controller

# Repo Server
kubectl logs -n argocd deployment/argocd-repo-server

# Notifications
kubectl logs -n argocd deployment/argocd-notifications-controller
```

### 🎯 Логи Flux

```bash
# Source Controller
kubectl logs -n flux-system deployment/source-controller

# Kustomize Controller
kubectl logs -n flux-system deployment/kustomize-controller

# Helm Controller
kubectl logs -n flux-system deployment/helm-controller
```

### 🎯 Команды диагностики

**ArgoCD:**

```bash
# Список apps
argocd app list

# Детали
argocd app get myapp

# Diff
argocd app diff myapp

# History
argocd app history myapp

# Manifests
argocd app manifests myapp

# Logs
argocd app logs myapp

# Список resources
argocd app resources myapp

# Terminate sync
argocd app terminate-op myapp
```

**Flux:**

```bash
# Список всех ресурсов
flux get all

# Kustomizations
flux get kustomizations

# Helm releases
flux get helmreleases -A

# Sources
flux get sources all

# Logs
flux logs

# Reconcile
flux reconcile kustomization myapp --with-source
```

### 🎯 `OutOfSync` при отсутствии изменений

**Проблема:** Git не менялся, но ArgoCD показывает `OutOfSync`.

**Причины:**

- **Mutating webhook** изменил ресурс.
- **HPA** изменил replicas.
- **Default values** от Kubernetes.
- **Field ordering** в JSON.

**Решение:** `ignoreDifferences`.

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
    - group: ''
      kind: Service
      jsonPointers:
        - /spec/clusterIP
```

### 🎯 Sync loop

**Проблема:** Application постоянно в `OutOfSync` → `Syncing` → `OutOfSync`.

**Причины:**

- Sync меняет ресурс, но не достигает желаемого.
- Mutating webhook меняет обратно.
- Неправильный `ignoreDifferences`.

**Диагностика:**

```bash
# Смотреть diff
argocd app diff myapp

# Что меняется
argocd app get myapp --show-operation
```

### 🎯 Проблемы с Git

**1. Не может клонировать репозиторий.**

```bash
# Логи repo-server
kubectl logs -n argocd deployment/argocd-repo-server | grep -i "clone\|fetch"
```

**Причины:**

- Неправильные credentials.
- Приватный репо без токена.
- Проблемы с сетью.

**2. Неправильный branch/tag.**

```yaml
spec:
  source:
    targetRevision: main    # проверить
```

### 🔬 Практика: диагностика

```bash
# 1. Проверить статус
argocd app get myapp

# 2. Diff
argocd app diff myapp

# 3. Логи
argocd app logs myapp
kubectl logs -n argocd deployment/argocd-application-controller

# 4. Ресурсы
argocd app resources myapp

# 5. Manifests
argocd app manifests myapp

# 6. Вручную проверить рендеринг
git clone <repo> /tmp/repo
cd /tmp/repo
kubectl kustomize apps/myapp/overlays/production > /tmp/rendered.yaml
cat /tmp/rendered.yaml

# 7. Сравнить с кластером
kubectl get deployment myapp -n production -o yaml
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **`argocd app diff`** — что расходится.
2. **`argocd app logs`** — что произошло.
3. **Логи контроллеров ArgoCD.**

**👍 СТОИТ:**

4. **Проверить рендеринг вручную** (`kubectl kustomize`).
5. **`ignoreDifferences`** для известных drift'ов.
6. **Notifications** для алертов.

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй `OutOfSync`.**
8. **Не делай ручных изменений** для «исправления».
9. **Не забывай про логи.**

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **GitOps** | Git как источник правды для кластера. |
| **Push-based** | CI применяет изменения в кластер. |
| **Pull-based** | Агент в кластере читает Git. |
| **Declarative** | Декларативное описание состояния. |
| **Reconciliation** | Согласование Git и кластера. |
| **Drift** | Расхождение Git и кластера. |
| **Self-healing** | Автоматический откат drift. |
| **ArgoCD** | GitOps-агент от Intuit. |
| **Flux** | GitOps-агент от Weaveworks. |
| **Application** | CRD ArgoCD для приложения. |
| **AppProject** | Группировка Applications + RBAC. |
| **ApplicationSet** | Генератор Applications. |
| **Sync Policy** | Как синхронизировать (auto/manual). |
| **Prune** | Удалять ресурсы, которых нет в Git. |
| **GitRepository** | CRD Flux для Git-репо. |
| **Kustomization** | CRD Flux для применения. |
| **HelmRelease** | CRD Flux для Helm. |
| **Base** | Общие манифесты (Kustomize). |
| **Overlay** | Специфичные для окружения (Kustomize). |
| **Promotion** | Продвижение версии между окружениями. |
| **Sealed Secrets** | Шифрование секретов ключом кластера. |
| **External Secrets** | Секреты из Vault/AWS SM. |
| **SOPS** | Secrets OPerationS. |
| **Blue-Green** | Два окружения рядом. |
| **Canary** | Постепенное увеличение трафика. |
| **Argo Rollouts** | CRD для продвинутых деплоев. |
| **Flagger** | Оператор для canary. |

---

## Что мы узнали?

- **GitOps** — Git как источник правды. Агент в кластере применяет изменения.
- **Четыре принципа:** declarative, versioned, pulled, reconciled.
- **Push vs Pull:** pull безопаснее, потому что CI не имеет credentials к кластеру.
- **ArgoCD:** Application, AppProject, ApplicationSet. Sync Policy, drift detection, self-healing.
- **Flux:** GitRepository, Kustomization, HelmRelease. Git-native.
- **Организация:** polyrepo, base + overlays, apps/ + infrastructure/.
- **Drift detection:** self-heal, ignoreDifferences, notifications.
- **Promotion:** overlays, branches, automation (ArgoCD Image Updater, Flux Image Automation).
- **Blue-Green и Canary:** через Git (Service selector, replicas ratio).
- **Secrets:** Sealed Secrets, External Secrets, SOPS, Vault CSI.
- **Disaster recovery:** Git + Velero + backup etcd + backup Sealed Secrets keys.
- **Миграция:** постепенно, по одному приложению, manual sync сначала.
- **Диагностика:** diff, logs, manifests, ignoreDifferences.

---

## Типичные ошибки

- ❌ **Называть GitOps `kubectl apply` из CI.** Это push-based.
- ❌ **Давать CI credentials к кластеру.**
- ❌ **Коммитить секреты без шифрования.**
- ❌ **Включать auto-sync без тестов.**
- ❌ **Игнорировать drift.**
- ❌ **Делать ручные изменения после внедрения GitOps.**
- ❌ **Не использовать `prune: true`.**
- ❌ **Не использовать `selfHeal: true` для production.**
- ❌ **Не настраивать notifications.**
- ❌ **Не тестировать disaster recovery.**
- ❌ **Не backup Git.**
- ❌ **Не backup Sealed Secrets keys.**
- ❌ **Мигрировать всё сразу.**
- ❌ **Не обучать команду.**
- ❌ **Игнорировать `OutOfSync` без диагностики.**

---

## Для быстрого повторения

- **GitOps:** Git — источник правды. Агент в кластере.
- **4 принципа:** declarative, versioned, pulled, reconciled.
- **ArgoCD:** Application + Sync Policy. UI, CLI.
- **Flux:** GitRepository + Kustomization + HelmRelease.
- **Организация:** polyrepo, apps/ + infrastructure/, base + overlays.
- **Drift:** selfHeal, prune, ignoreDifferences.
- **Promotion:** overlays, PR, ArgoCD Image Updater.
- **Blue-Green:** Service selector. **Canary:** replicas ratio.
- **Secrets:** Sealed Secrets, External Secrets, SOPS.
- **DR:** Git + Velero + backup keys.
- **Миграция:** постепенно, manual → auto.
- **Диагностика:** `argocd app diff`, logs, manifests.

---

## Вопросы для самопроверки

1. Что такое GitOps? Чем отличается от push-based CI/CD?
2. Четыре принципа GitOps — назови и объясни.
3. Почему pull-based безопаснее push-based?
4. Что такое Application в ArgoCD?
5. Что такое Sync Policy? Какие бывают?
6. Что такое drift? Как обнаружить?
7. Что такое self-healing? Когда использовать?
8. Что такое ApplicationSet? Зачем нужен?
9. Чем Flux отличается от ArgoCD?
10. Как организовать gitops-репозиторий?
11. Как делать promotion между окружениями?
12. Как делать Blue-Green в GitOps?
13. Как хранить секреты в GitOps?
14. Как делать disaster recovery через Git?
15. Как мигрировать с push-based на GitOps?

---

## Ответы

**1. GitOps**

Git — источник правды. Агент в кластере (ArgoCD, Flux) читает Git и применяет. CI не имеет доступа к кластеру. Push-based: CI применяет изменения напрямую.

**2. Четыре принципа**

- **Declarative** — состояние описано декларативно.
- **Versioned** — всё в Git.
- **Pulled** — агент сам читает Git.
- **Reconciled** — постоянно согласует.

**3. Pull vs Push безопасность**

В pull CI не имеет kubeconfig. Только git push к manifests. Даже при компрометации CI — максимум изменят манифест (ревьюится).

**4. Application**

CRD ArgoCD. Описывает: source (repo, path), destination (cluster, namespace), syncPolicy. ArgoCD применяет манифесты в кластер.

**5. Sync Policy**

`automated` (auto-sync) или пустая (manual). `automated` с `prune` (удалять orphaned) и `selfHeal` (откатывать drift).

**6. Drift**

Расхождение Git и кластера. Обнаружение: ArgoCD сравнивает каждые 3 минуты. Статус `OutOfSync`. `argocd app diff myapp`.

**7. Self-healing**

Автоматический откат drift'а. `selfHeal: true`. Для production. Не использовать с HPA (конфликт).

**8. ApplicationSet**

Генератор Applications. Создаёт много Applications из шаблона. Generators: list, git, cluster, matrix.

**9. Flux vs ArgoCD**

Flux: Git-native, Helm-first, HelmRelease. ArgoCD: UI, multi-cluster, AppProject. Оба хороши.

**10. Организация**

Polyrepo. `apps/` (приложения), `infrastructure/` (ingress, monitoring), `clusters/` (per-cluster). Base + overlays. Одна ветка (main).

**11. Promotion**

Overlays (рекомендуется): обновить overlay → PR → merge. Или branches (main → staging → prod). Или ArgoCD Image Updater / Flux Image Automation.

**12. Blue-Green в GitOps**

Два Deployment (blue, green). Service с selector на active. Cutover — изменение selector в Git. Rollback — обратно.

**13. Secrets**

Sealed Secrets (шифрование ключом кластера). External Secrets (Vault, AWS SM). SOPS (KMS, age). Vault CSI. Никогда не коммить plain secrets.

**14. DR через Git**

Новый кластер → установить ArgoCD → ArgoCD применяет из Git. Stateful данные — Velero restore. Backup Sealed Secrets keys. Регулярные DR drills.

**15. Миграция**

Постепенно: dev → staging → prod. По одному приложению. Manual sync сначала, auto-sync после стабилизации. Убрать `kubectl apply` из CI.

---

## Куда идти дальше?

Мы разобрали GitOps — Git как источник правды. Теперь ты знаешь:

- Четыре принципа GitOps.
- Push vs Pull.
- ArgoCD и Flux.
- Организацию репозиториев.
- Drift detection и self-healing.
- Promotion между окружениями.
- Blue-Green и Canary.
- Secrets.
- Disaster recovery.
- Миграцию.

Следующая глава по оглавлению — **Глава 21: Observability — логирование**.

Скажи «дальше» — и я отправлю Главу 21.