# 🏗️ Глава 25: Платформенная инженерия

**Что вы узнаете:**
- Что такое платформенная инженерия и почему она появилась.
- Чем Platform Engineering отличается от DevOps и SRE.
- Что такое Internal Developer Platform (IDP).
- Что такое golden path и как его проектировать.
- Как устроен self-service для разработчиков.
- Что такое Backstage и как его использовать.
- Как измерять успех платформы через DORA и SPACE.
- Как построить платформенную команду.
- Что такое «Platform as a Product».
- Как мигрировать от DevOps-команды к платформенной.

**После прочтения вы сможете:**
- Объяснить, зачем нужна платформенная инженерия.
- Спроектировать golden path для разработчиков.
- Развернуть Backstage как портал.
- Определить метрики успеха платформы.
- Построить платформенную команду.
- Оценить, нужна ли платформа вашей компании.

---

## Содержание

- [25.0 Пролог: DevOps-команда стала бутылочным горлышком](#250-пролог-devops-команда-стала-бутылочным-горлышком)
- [25.1 Что такое платформенная инженерия](#251-что-такое-платформенная-инженерия)
- [25.2 DevOps vs SRE vs Platform Engineering](#252-devops-vs-sre-vs-platform-engineering)
- [25.3 Internal Developer Platform (IDP)](#253-internal-developer-platform-idp)
- [25.4 Golden Path: золотой путь разработчика](#254-golden-path-золотой-путь-разработчика)
- [25.5 Self-service: разработчик сам себе DevOps](#255-self-service-разработчик-сам-себе-devops)
- [25.6 Backstage: портал разработчика](#256-backstage-портал-разработчика)
- [25.7 Platform as a Product](#257-platform-as-a-product)
- [25.8 Метрики платформы: DORA и SPACE](#258-метрики-платформы-dora-и-space)
- [25.9 Платформенная команда](#259-платформенная-команда)
- [25.10 Миграция к платформенной инженерии](#2510-миграция-к-платформенной-инженерии)
- [25.11 Антипаттерны платформенной инженерии](#2511-антипаттерны-платформенной-инженерии)
- [25.12 Диагностика проблем](#2512-диагностика-проблем)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 25.0 Пролог: DevOps-команда стала бутылочным горлышком

Понедельник, 10:00. В твоей компании:

- **50 разработчиков** в 10 командах.
- **DevOps-команда** из 5 человек.
- **200 микросервисов.**

**Задачи DevOps-команды:**

- Настроить CI/CD для нового сервиса.
- Создать Kubernetes namespace.
- Настроить мониторинг.
- Настроить секреты.
- Помочь с деплоем.
- Разобраться с инцидентом.
- Обновить кластер.
- Настроить ingress.
- ...

**Каждый день — очередь из тикетов:**

```
[DEVOPS-1234] Настроить CI для нового сервиса payments
[DEVOPS-1235] Создать namespace для команды checkout
[DEVOPS-1236] Добавить мониторинг для user-service
[DEVOPS-1237] Помочь с деплоем (срочно!)
[DEVOPS-1238] Настроить секреты для api-gateway
[DEVOPS-1239] Почему не работает ingress?
[DEVOPS-1240] Обновить версию PostgreSQL
...
```

**Ты — DevOps-инженер. Ты завален работой.**

**Проблемы:**

1. **Бутылочное горлышко.** Один DevOps-инженер на 10 разработчиков.
2. **Очередь тикетов.** Разработчики ждут днями.
3. **Повторяющаяся работа.** Одно и то же для каждого сервиса.
4. **Контекст-свитчинг.** Срочный инцидент прерывает настройку.
5. **Нет масштабирования.** Нанять больше DevOps — дорого и не решает.
6. **Разработчики не могут сами.** Зависят от DevOps.

**Это — не проблема DevOps. Это — проблема отсутствия платформы.**

**Решение:** платформенная инженерия.

**Идея:** вместо того чтобы DevOps делал всё за разработчиков, создать **платформу**, где разработчики **сами** могут:

- Создать новый сервис.
- Настроить CI/CD.
- Задеплоить.
- Настроить мониторинг.
- Управлять секретами.

**Платформа — self-service.** Разработчики не ждут DevOps.

**Платформенная команда** строит платформу, а не тушит пожары.

В этой главе мы разберём платформенную инженерию. Как построить платформу, которая масштабируется.

Это — **следующий шаг DevOps**. Из Главы 0: не «DevOps-инженер как роль», а «платформа как продукт».

---

## 25.1 Что такое платформенная инженерия

### 🔌 Проблема: DevOps не масштабируется

В Главе 0 мы узнали: **DevOps — это не роль, а культура**. Но на практике:

- Компании нанимают **DevOps-инженеров**.
- DevOps-команда делает CI/CD, инфраструктуру, мониторинг.
- Разработчики зависят от DevOps.
- **Бутылочное горлышко.**

**Это антипаттерн.** DevOps-инженер как «универсальный солдат» становится узким местом.

**Платформенная инженерия** — ответ на это.

### 📊 Что такое платформенная инженерия

**Платформенная инженерия** — дисциплина построения **внутренних платформ** для разработчиков.

**Ключевая идея:** вместо того чтобы делать за разработчиков, **создай инструменты**, которыми они сами пользуются.

**Аналогия:**

**Плохо:** ресторан, где официант готовит еду для каждого гостя.
**Хорошо:** ресторан с шведским столом, где гость сам берёт, что хочет.

**Платформа — шведский стол.** Разработчики сами:

- Создают сервисы.
- Настраивают CI/CD.
- Деплоят.
- Мониторят.

**Платформенная команда** строит и поддерживает «шведский стол».

### 🎯 Что даёт платформа

**1. Скорость.**

Разработчик создаёт сервис за 5 минут, а не за 5 дней (ожидания DevOps).

**2. Масштабирование.**

Платформа масштабируется на 100 команд. DevOps-команда — нет.

**3. Стандартизация.**

Все сервисы создаются одинаково. Единые практики.

**4. Снижение когнитивной нагрузки.**

Разработчик не думает о K8s, CI/CD, мониторинге. Платформа делает это.

**5. Автономность.**

Команды не зависят от DevOps.

### 🎯 Принципы

**1. Platform as a Product.**

Платформа — продукт. У неё есть пользователи (разработчики). Их опыт важен.

**2. Self-service.**

Разработчики сами делают, что нужно. Без тикетов.

**3. Golden Path.**

Оптимальный путь для типичных задач. Opinionated.

**4. Abstraction.**

Скрыть сложность. Разработчик говорит «мне нужен сервис», платформа делает.

**5. Standardization.**

Единые практики. Меньше вариантов — меньше ошибок.

**6. Developer Experience.**

Опыт разработчика — главное. Удобство, документация, скорость.

### 🎯 Что входит в платформу

**1. Compute.**

Kubernetes, VM, serverless.

**2. CI/CD.**

Пайплайны, деплой, rollback.

**3. Observability.**

Метрики, логи, трейсы.

**4. Security.**

Секреты, RBAC, scan.

**5. Data.**

Базы данных, кэши, брокеры.

**6. Networking.**

Ingress, DNS, service mesh.

**7. Developer Portal.**

Документация, каталог сервисов, self-service.

**8. Environment Management.**

Dev, staging, prod.

### 💡 Практика: что важно понять

**✅ ОБЯЗАТЕЛЬНО:**

1. **Платформа — продукт.**
2. **Self-service — ключевая идея.**
3. **Golden path — стандартный путь.**

**👍 СТОИТ:**

4. **Начинать с боли разработчиков.**
5. **Измерять DX.**
6. **Итерировать.**

**❌ НЕ ДЕЛАЙ:**

7. **Не строй платформу «для себя».**
8. **Не навязывай без выгоды.**
9. **Не забывай про разработчиков.**

### Где мы сейчас

Мы разобрали, что такое платформенная инженерия. Теперь — **DevOps vs SRE vs Platform**.

---

## 25.2 DevOps vs SRE vs Platform Engineering

### 🔌 Проблема: как понять разницу

DevOps, SRE, Platform Engineering — три термина, которые часто путают.

**Разберём.**

### 📊 DevOps

**Что:** культурное движение.

**Ключевая идея:** разрушить стену между Dev и Ops.

**Кто:** все. DevOps — не роль.

**Что делает:**

- **CI/CD.**
- **Автоматизация.**
- **Культура.**
- **Метрики (DORA).**

**Проблема:** на практике превратилось в «DevOps-инженер» как роль.

### 📊 SRE

**Что:** инженерная дисциплина от Google.

**Ключевая идея:** надёжность как фича. SLO и error budget.

**Кто:** SRE-инженеры.

**Что делает:**

- **SLO/SLI.**
- **Error budget.**
- **Toil reduction.**
- **Incident response.**
- **Capacity planning.**

**Отличие от DevOps:** SRE — конкретная реализация DevOps-принципов через инженерию.

### 📊 Platform Engineering

**Что:** построение внутренних платформ.

**Ключевая идея:** self-service для разработчиков.

**Кто:** платформенные инженеры.

**Что делает:**

- **IDP.**
- **Golden paths.**
- **Developer portal.**
- **Self-service.**
- **Developer experience.**

**Отличие от DevOps:** Platform Engineering — продукт для разработчиков. DevOps — культура.

### 🎯 Сравнение

| Аспект | DevOps | SRE | Platform Engineering |
|:---|:---|:---|:---|
| **Что** | Культура | Дисциплина | Продукт |
| **Кто** | Все | SRE-инженеры | Платформенные инженеры |
| **Фокус** | Процессы | Надёжность | Developer experience |
| **Метрики** | DORA | SLO/SLI | DX, adoption |
| **Пользователи** | Команда | Сервисы | Разработчики |
| **Артефакты** | Пайплайны | SLO | Платформа |

### 🎯 Как они связаны

```
DevOps (культура)
    │
    ├── SRE (надёжность)
    │
    └── Platform Engineering (self-service)
```

**DevOps — зонтик.** SRE и Platform Engineering — конкретные реализации.

**SRE** фокусируется на надёжности production.
**Platform Engineering** фокусируется на скорости разработки.

**Оба — DevOps.** Но разные аспекты.

### 🎯 Когда что

**SRE:**

- **Критичные сервисы.**
- **SLO важны.**
- **Много инцидентов.**
- **Есть команда.**

**Platform Engineering:**

- **50+ разработчиков.**
- **DevOps — бутылочное горлышко.**
- **Много микросервисов.**
- **Нужен self-service.**

**Оба могут сосуществовать.**

### 🎯 Эволюция

**Типичный путь:**

1. **Нет DevOps.** Разработчики делают всё сами.
2. **DevOps-команда.** Централизованная.
3. **Бутылочное горлышко.** DevOps не справляется.
4. **SRE.** Для надёжности.
5. **Platform Engineering.** Для self-service.
6. **Зрелая платформа.** Разработчики автономны.

**Не все проходят все этапы.** Зависит от размера и зрелости.

### 💡 Практика: что важно понять

**✅ ОБЯЗАТЕЛЬНО:**

1. **DevOps — культура.**
2. **SRE — надёжность.**
3. **Platform Engineering — self-service.**

**👍 СТОИТ:**

4. **Начинать с DevOps-принципов.**
5. **SRE при росте надёжности.**
6. **Platform при росте команды.**

**❌ НЕ ДЕЛАЙ:**

7. **Не путай термины.**
8. **Не внедряй Platform без нужды.**
9. **Не забывай про DevOps-культуру.**

### Где мы сейчас

Мы разобрали разницу. Теперь — **IDP**.

---

## 25.3 Internal Developer Platform (IDP)

### 🔌 Проблема: что такое IDP

Платформа — абстрактное понятие. Что конкретно?

**IDP** — Internal Developer Platform.

### 📊 Что такое IDP

**IDP** — внутренняя платформа для разработчиков.

**Что включает:**

- **Self-service портал.**
- **CI/CD.**
- **Environment management.**
- **Observability.**
- **Security.**
- **Documentation.**

**Ключевая идея:** единая точка входа для разработчика.

### 🎯 Компоненты IDP

**1. Developer Portal.**

- **UI** для self-service.
- **Каталог** сервисов.
- **Документация.**
- **Метрики.**

**Пример:** Backstage.

**2. Self-service actions.**

- **Создать сервис.**
- **Настроить CI/CD.**
- **Задеплоить.**
- **Создать БД.**

**Пример:** Scaffolder в Backstage.

**3. Environment Management.**

- **Dev, staging, prod.**
- **Ephemeral environments.**
- **Preview environments.**

**4. CI/CD.**

- **Пайплайны.**
- **Артефакты.**
- **Деплой.**

**5. Observability.**

- **Метрики.**
- **Логи.**
- **Трейсы.**
- **Алерты.**

**6. Security.**

- **Секреты.**
- **RBAC.**
- **Scanning.**
- **Compliance.**

**7. Documentation.**

- **API docs.**
- **Runbooks.**
- **Onboarding.**

**8. Service Catalog.**

- **Список сервисов.**
- **Owners.**
- **Dependencies.**
- **Health.**

### 🎯 Уровни IDP

**Level 1: Ad-hoc.**

- Разные инструменты.
- Разные процессы.
- Нет стандартизации.

**Level 2: Standardized.**

- Единые практики.
- Единые инструменты.
- Документация.

**Level 3: Self-service.**

- Разработчики сами.
- Портал.
- Scaffolder.

**Level 4: Platform as a Product.**

- Метрики DX.
- Обратная связь.
- Итерации.

**Level 5: Autonomous.**

- Полная автономность.
- Платформа как продукт.
- Непрерывное улучшение.

### 🎯 Примеры IDP

**Открытые:**

- **Backstage** (Spotify).
- **Port** (commercial).
- **Cortex** (commercial).
- **OpsLevel** (commercial).

**Внутренние:**

- **Spotify Backstage** (open source).
- **Netflix** — внутренняя.
- **Uber** — внутренняя.
- **Airbnb** — внутренняя.

### 🎯 Минимальная IDP

**Что нужно в первую очередь:**

1. **Service catalog.** Список сервисов с owners.
2. **Scaffolder.** Создание нового сервиса.
3. **CI/CD templates.** Стандартные пайплайны.
4. **Documentation.** Central hub.
5. **Metrics.** Dashboards.

**Этого достаточно для старта.**

### 🔬 Практика: минимальная IDP

**Пример: Backstage + GitLab + ArgoCD.**

```yaml
# Backstage: scaffolder для нового сервиса
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: go-service
  title: Go Microservice
spec:
  parameters:
    - title: Service Info
      properties:
        name:
          type: string
        owner:
          type: string
        description:
          type: string
  steps:
    - id: fetch
      name: Fetch template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          owner: ${{ parameters.owner }}
    
    - id: publish
      name: Publish to GitLab
      action: publish:gitlab
      input:
        repoUrl: gitlab.com?owner=myorg&repo=${{ parameters.name }}
    
    - id: register
      name: Register in catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
```

**Что даёт:** разработчик создаёт сервис через UI. Получает:

- **Git-репозиторий** со скелетом.
- **CI/CD** пайплайн.
- **K8s манифесты.**
- **Мониторинг.**
- **Запись в каталоге.**

**За 5 минут.**

### 💡 Практика: как строить IDP

**✅ ОБЯЗАТЕЛЬНО:**

1. **Начать с боли** разработчиков.
2. **Service catalog** первым.
3. **Scaffolder** для типовых задач.
4. **Интеграция** с существующими инструментами.

**👍 СТОИТ:**

5. **Backstage** как портал.
6. **Метрики** DX.
7. **Обратная связь** от разработчиков.

**❌ НЕ ДЕЛАЙ:**

8. **Не строй всё сразу.**
9. **Не дублируй** существующие инструменты.
10. **Не забывай про UX.**

### Где мы сейчас

Мы разобрали IDP. Теперь — **Golden Path**.

---

## 25.4 Golden Path: золотой путь разработчика

### 🔌 Проблема: разработчик не знает, как делать

Разработчик хочет создать сервис. **Как?**

- Какой язык?
- Какая структура?
- Как настроить CI?
- Как деплоить?
- Как мониторить?
- Как секреты?

**Вариантов много. Разработчик теряется.**

**Решение:** Golden Path.

### 📊 Что такое Golden Path

**Golden Path** — рекомендованный путь для типичных задач.

**Opinionated.** Меньше вариантов — больше скорости.

**Что включает:**

- **Шаблоны** проектов.
- **Стандартные** пайплайны.
- **Инструменты** по умолчанию.
- **Документация.**
- **Поддержка.**

**Разработчик идёт по Golden Path — получает работающий сервис.**

### 🎯 Примеры Golden Path

**1. Новый сервис.**

**Путь:**

1. **Открыть** Backstage.
2. **Выбрать** шаблон «Go Microservice».
3. **Заполнить** поля (name, owner).
4. **Нажать** Create.
5. **Через 5 минут** — сервис в dev.

**Что создаётся:**

- Git-репозиторий.
- Dockerfile.
- CI/CD пайплайн.
- K8s манифесты.
- Мониторинг.
- Логи.
- Alert.
- Документация.
- Запись в каталоге.

**2. Новый endpoint.**

**Путь:**

1. **Форк** шаблон endpoint'а.
2. **Заполнить** параметры.
3. **PR** → merge.
4. **CI** деплоит.

**3. Новая БД.**

**Путь:**

1. **Открыть** Backstage.
2. **Выбрать** «PostgreSQL».
3. **Заполнить** параметры.
4. **Create.**
5. **Через 10 минут** — БД готова.

**Connection string в секретах.**

### 🎯 Принципы Golden Path

**1. Opinionated.**

Один правильный путь. Не «вот 10 вариантов».

**2. Safe.**

Идёшь по Golden Path — получаешь правильное. Безопасность, надёжность, best practices.

**3. Fast.**

5 минут от идеи до running сервиса.

**4. Supported.**

Если сломалось — платформенная команда помогает.

**5. Documented.**

Понятная документация.

**6. Not mandatory.**

Можно пойти другим путём. Но Golden Path — проще.

### 🎯 Escaping the Golden Path

**Что если Golden Path не подходит?**

**Варианты:**

1. **Расширить** Golden Path.
2. **Создать новый** Golden Path.
3. **Идти своим путём.** С поддержкой.

**Правило:** не запрещать. Но делать Golden Path **проще** других путей.

### 🎯 Anti-patterns

**1. Слишком много Golden Paths.**

10 путей — это не Golden Path. Это confusion.

**2. Golden Path без поддержки.**

«Иди этим путём, но если сломается — сам разбирайся». Плохо.

**3. Mandatory Golden Path.**

«Только этот путь». Слишком жёстко.

**4. Golden Path без эволюции.**

Не обновляется. Устаревает.

### 🎯 Пример: Golden Path для Go-сервиса

**Шаблон:**

```
go-service-template/
├── .gitlab-ci.yml              # CI/CD
├── Dockerfile                  # Multi-stage
├── go.mod
├── go.sum
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── api/
│   ├── config/
│   └── telemetry/
├── deploy/
│   ├── base/
│   └── overlays/
│       ├── dev/
│       ├── staging/
│       └── prod/
├── Makefile
└── README.md
```

**Что входит:**

- **Metrics** (Prometheus).
- **Logs** (structured, JSON).
- **Traces** (OpenTelemetry).
- **Health checks.**
- **Graceful shutdown.**
- **CI/CD.**
- **K8s манифесты.**
- **ArgoCD application.**

**Разработчик заполняет поля — получает всё.**

### 🔬 Практика: Golden Path

**Backstage Template:**

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: go-service
  title: Go Microservice
  description: Production-ready Go microservice
spec:
  owner: platform-team
  type: service
  
  parameters:
    - title: Service Info
      required: [name, owner, system]
      properties:
        name:
          type: string
          pattern: '^[a-z][a-z0-9-]*$'
          description: Service name (lowercase, hyphens)
        owner:
          type: string
          ui:field: OwnerPicker
        system:
          type: string
          ui:field: EntityPicker
          ui:options:
            catalogFilter:
              kind: System
        description:
          type: string
    
    - title: Tech Details
      properties:
        port:
          type: number
          default: 8080
        database:
          type: boolean
          default: false
          description: Need PostgreSQL?
        cache:
          type: boolean
          default: false
          description: Need Redis?
  
  steps:
    - id: fetch
      name: Fetch skeleton
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          owner: ${{ parameters.owner }}
          port: ${{ parameters.port }}
          database: ${{ parameters.database }}
          cache: ${{ parameters.cache }}
    
    - id: publish
      name: Create GitLab repo
      action: publish:gitlab
      input:
        repoUrl: gitlab.com?owner=myorg&repo=${{ parameters.name }}
        defaultBranch: main
    
    - id: register
      name: Register in catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml
    
    - id: argocd
      name: Create ArgoCD app
      action: argocd:create-app
      input:
        name: ${{ parameters.name }}
        repoUrl: ${{ steps.publish.output.remoteUrl }}
        path: deploy/overlays/dev
```

**Что произойдёт:**

1. Разработчик заполняет форму.
2. Backstage создаёт Git-репо.
3. Применяет скелет.
4. Создаёт ArgoCD application.
5. Регистрирует в каталоге.
6. Отправляет уведомление в Slack.

**Сервис работает через 5 минут.**

### 💡 Практика: как проектировать Golden Path

**✅ ОБЯЗАТЕЛЬНО:**

1. **Opinionated** — один путь.
2. **Safe** — best practices из коробки.
3. **Fast** — 5 минут.
4. **Supported** — платформенная команда помогает.
5. **Documented.**

**👍 СТОИТ:**

6. **Несколько Golden Paths** для разных случаев (Go, Python, ML).
7. **Эволюция** — обновлять.
8. **Feedback loop** — от разработчиков.

**❌ НЕ ДЕЛАЙ:**

9. **Не делай mandatory.**
10. **Не игнорируй UX.**
11. **Не забывай про поддержку.**

### Где мы сейчас

Мы разобрали Golden Path. Теперь — **self-service**.

---

## 25.5 Self-service: разработчик сам себе DevOps

### 🔌 Проблема: разработчик зависит от DevOps

Разработчик хочет:

- **Создать сервис.**
- **Задеплоить.**
- **Настроить БД.**
- **Управлять секретами.**

Но **не может**. Идёт в DevOps. Ждёт.

**Решение:** self-service.

### 📊 Что такое self-service

**Self-service** — разработчик сам делает, что нужно, через платформу.

**Без тикетов. Без ожидания. Без DevOps.**

### 🎯 Что должно быть self-service

**1. Service creation.**

Создание нового сервиса.

**2. Deployment.**

Деплой в dev/staging/prod.

**3. Environment provisioning.**

Создание окружений.

**4. Database provisioning.**

Создание БД.

**5. Secret management.**

Управление секретами.

**6. Access management.**

Доступы.

**7. Observability setup.**

Мониторинг.

**8. Incident response.**

Runbooks, alerts.

### 🎯 Как работает

**Пример: создание PostgreSQL.**

**Без self-service:**

1. Разработчик пишет тикет в DevOps.
2. Ждёт 2 дня.
3. DevOps создаёт БД вручную.
4. Отдаёт connection string.
5. Разработчик настраивает.

**С self-service:**

1. Разработчик открывает Backstage.
2. Выбирает «PostgreSQL».
3. Заполняет форму (size, env, backup).
4. Нажимает Create.
5. Через 5 минут — БД готова.
6. Connection string в секретах.
7. Метрики и алерты настроены.

**Автоматически.**

### 🎯 Реализация

**Backstage + Terraform + Crossplane.**

```yaml
# Backstage Template для PostgreSQL
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: postgresql
  title: PostgreSQL Database
spec:
  parameters:
    - title: Database Info
      properties:
        name:
          type: string
        size:
          type: string
          enum: [small, medium, large]
        environment:
          type: string
          enum: [dev, staging, prod]
        backup:
          type: boolean
          default: true
  
  steps:
    - id: terraform
      name: Create database
      action: terraform:apply
      input:
        workspace: ${{ parameters.environment }}
        variables:
          name: ${{ parameters.name }}
          size: ${{ parameters.size }}
          backup: ${{ parameters.backup }}
    
    - id: secret
      name: Store credentials
      action: vault:write
      input:
        path: secret/db/${{ parameters.name }}
        data:
          host: ${{ steps.terraform.output.host }}
          port: ${{ steps.terraform.output.port }}
          password: ${{ steps.terraform.output.password }}
    
    - id: register
      name: Register in catalog
      action: catalog:register
      input:
        kind: Resource
        name: ${{ parameters.name }}
```

**Что произойдёт:**

1. Разработчик заполняет форму.
2. Terraform создаёт RDS.
3. Vault сохраняет credentials.
4. Каталог регистрирует ресурс.
5. Уведомление в Slack.

**Разработчик получил БД за 5 минут.**

### 🎯 Guardrails

**Self-service ≠ без контроля.**

**Что нужно:**

- **Квоты** — лимиты на ресурсы.
- **Политики** — что можно/нельзя.
- **Approval** — для sensitive.
- **Audit** — кто что делал.
- **Cost tracking** — кто сколько тратит.

**Пример: approval для prod.**

```yaml
# В Backstage
- id: approval
  name: Require approval for prod
  if: ${{ parameters.environment === 'prod' }}
  action: approval:request
  input:
    approvers: [team-lead]
```

**Что даёт:** prod требует approval. Dev — сразу.

### 🎯 Что НЕ должно быть self-service

**1. Изменение политик безопасности.**

**2. Доступ к production data.**

**3. Удаление production ресурсов.**

**4. Network policies.**

**5. Cluster-wide изменения.**

**Эти вещи — через DevOps/SRE.**

### 🎯 Уровни self-service

**Level 1: Read-only.**

Разработчик видит, но не меняет.

**Level 2: Dev self-service.**

Разработчик может в dev.

**Level 3: Staging self-service.**

Разработчик может в staging.

**Level 4: Prod self-service (с guardrails).**

Разработчик может в prod через approval.

**Level 5: Full autonomy.**

Разработчик может всё.

**Не всем нужен Level 5.** Зависит от зрелости.

### 🔬 Практика: self-service

**Backstage + ArgoCD + Crossplane.**

```yaml
# CompositeResourceDefinition (Crossplane)
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xpostgresqlinstances.database.example.org
spec:
  group: database.example.org
  names:
    kind: XPostgreSQLInstance
    plural: xpostgresqlinstances
  claimNames:
    kind: PostgreSQLInstance
    plural: postgresqlinstances
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    storageGB:
                      type: integer
                    environment:
                      type: string
```

**Разработчик создаёт Claim:**

```yaml
apiVersion: database.example.org/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: my-db
  namespace: my-team
spec:
  parameters:
    storageGB: 20
    environment: dev
  writeConnectionSecretToRef:
    name: my-db-conn
```

**Что произойдёт:**

1. Crossplane видит Claim.
2. Создаёт RDS в AWS.
3. Создаёт Secret с connection string.
4. Разработчик использует.

**Self-service.**

### 💡 Практика: как строить self-service

**✅ ОБЯЗАТЕЛЬНО:**

1. **Начать с простых задач.**
2. **Guardrails** для sensitive.
3. **Audit** всего.
4. **Метрики** использования.

**👍 СТОИТ:**

5. **Backstage** как UI.
6. **Crossplane** для provisioning.
7. **Approval** для prod.

**❌ НЕ ДЕЛАЙ:**

8. **Не давай полную свободу сразу.**
9. **Не забывай про квоты.**
10. **Не игнорируй feedback.**

### Где мы сейчас

Мы разобрали self-service. Теперь — **Backstage**.

---

## 25.6 Backstage: портал разработчика

### 🔌 Проблема: где разработчик находит всё

У разработчика 20 инструментов:

- Git.
- CI/CD.
- Monitoring.
- Logging.
- Tracing.
- Documentation.
- Secrets.
- K8s.
- ...

**Где найти всё?**

**Решение:** Backstage.

### 📊 Что такое Backstage

**Backstage** — open-source портал разработчика от Spotify.

**Что даёт:**

- **Service catalog** — все сервисы.
- **Software templates** — создание сервисов.
- **TechDocs** — документация.
- **Plugins** — интеграции.
- **Search** — поиск по всему.
- **Homepage** — единая точка входа.

### 🎯 Архитектура

```
┌─────────────────────────────────────────┐
│              BACKSTAGE                   │
│                                          │
│  ┌──────────────┐  ┌──────────────────┐ │
│  │  Frontend    │  │  Backend         │ │
│  │  (React)     │  │  (Node.js)       │ │
│  └──────────────┘  └──────────────────┘ │
│                                          │
│  ┌──────────────┐  ┌──────────────────┐ │
│  │  Catalog     │  │  Scaffolder      │ │
│  │              │  │                  │ │
│  │  Сервисы     │  │  Шаблоны         │ │
│  └──────────────┘  └──────────────────┘ │
│                                          │
│  ┌──────────────┐  ┌──────────────────┐ │
│  │  TechDocs    │  │  Plugins         │ │
│  │              │  │                  │ │
│  │  Документация│  │  Интеграции      │ │
│  └──────────────┘  └──────────────────┘ │
└─────────────────────────────────────────┘
```

### 🎯 Catalog

**Catalog** — реестр всех сервисов.

**Что содержит:**

- **Components** (сервисы, библиотеки, websites).
- **APIs.**
- **Resources** (БД, очереди).
- **Systems** (группы сервисов).
- **Domains** (бизнес-домены).
- **Groups** (команды).
- **Users.**

**Каждый компонент — файл `catalog-info.yaml` в репо:**

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: myapp
  description: My Go service
  annotations:
    github.com/project-slug: myorg/myapp
    backstage.io/techdocs-ref: dir:.
    prometheus.io/alert: myapp
  tags:
    - go
    - api
  links:
    - url: https://grafana.example.com/d/myapp
      title: Dashboard
    - url: https://myapp.example.com
      title: Production
spec:
  type: service
  lifecycle: production
  owner: team-payments
  system: payments
  providesApis:
    - myapp-api
  dependsOn:
    - resource:myapp-db
    - component:auth-service
```

**Что даёт:**

- **Owners** — кто отвечает.
- **Dependencies** — что зависит.
- **Health** — статус.
- **Links** — дашборды, docs.

### 🎯 Scaffolder

**Scaffolder** — создание сервисов из шаблонов.

**Что даёт:**

- **Шаблоны** для типовых задач.
- **Self-service** создание.
- **Автоматизация** setup.

**Разобрали в Golden Path.**

### 🎯 TechDocs

**TechDocs** — документация рядом с кодом.

**Как работает:**

1. **Markdown** файлы в `docs/` в репо.
2. **Backstage** рендерит.
3. **Версионируется** с кодом.

**Что даёт:**

- **Документация** всегда актуальна.
- **Search** по всей документации.
- **Единый стиль.**

### 🎯 Plugins

**Плагины для интеграций:**

- **Kubernetes** — статус Pod'ов.
- **ArgoCD** — статус деплоев.
- **GitLab/GitHub** — PR, issues.
- **Prometheus** — метрики.
- **Grafana** — дашборды.
- **PagerDuty** — on-call.
- **Sentry** — ошибки.
- **Jira** — задачи.
- ...

**Что даёт:** единая точка входа для всей информации о сервисе.

### 🎯 Установка

**Backstage в Kubernetes:**

```bash
# Создать приложение
npx @backstage/create-app@latest

# Собрать
yarn install
yarn build

# Docker
docker build -t backstage:latest .

# Helm
helm repo add backstage https://backstage.github.io/charts
helm install backstage backstage/backstage
```

**Минимальная конфигурация:**

```yaml
# app-config.yaml
app:
  title: My Company Portal
  baseUrl: http://localhost:3000

backend:
  baseUrl: http://localhost:7007
  database:
    client: pg
    connection:
      host: postgres
      port: 5432
      user: backstage
      password: ${POSTGRES_PASSWORD}

catalog:
  locations:
    - type: file
      target: ../../examples/entities.yaml
    - type: url
      target: https://gitlab.com/myorg/*/blob/main/catalog-info.yaml
      rules:
        - allow: [Component, System, API, Resource]

integrations:
  gitlab:
    - host: gitlab.com
      token: ${GITLAB_TOKEN}
```

### 🎯 Кейсы использования

**1. Onboarding нового разработчика.**

**Без Backstage:** 2 недели разбираться.

**С Backstage:** 1 день. Всё в одном месте.

**2. Поиск сервиса.**

**Без Backstage:** спрашивать в Slack.

**С Backstage:** поиск по каталогу.

**3. Создание сервиса.**

**Без Backstage:** тикет в DevOps, 2 дня.

**С Backstage:** 5 минут.

**4. Инцидент.**

**Без Backstage:** искать дашборды, логи, owners.

**С Backstage:** всё на странице сервиса.

### 🔬 Практика: Backstage

```bash
# 1. Создать приложение
npx @backstage/create-app@latest
cd my-portal

# 2. Запустить
yarn dev
# http://localhost:3000

# 3. Добавить каталог
cat > examples/entities.yaml <<EOF
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: myapp
  description: My Go service
spec:
  type: service
  lifecycle: production
  owner: team-payments
EOF

# 4. Обновить app-config.yaml
# catalog:
#   locations:
#     - type: file
#       target: ../../examples/entities.yaml

# 5. Посмотреть в UI
# http://localhost:3000/catalog

# 6. Добавить GitLab integration
# integrations:
#   gitlab:
#     - host: gitlab.com
#       token: ${GITLAB_TOKEN}

# 7. Добавить plugin для K8s
yarn add @backstage/plugin-kubernetes
```

### 💡 Практика: как использовать Backstage

**✅ ОБЯЗАТЕЛЬНО:**

1. **Service catalog** первым.
2. **catalog-info.yaml** в каждом репо.
3. **TechDocs** для документации.
4. **Plugins** для интеграций.

**👍 СТОИТ:**

5. **Scaffolder** для Golden Path.
6. **Search** по всему.
7. **Custom plugins** для специфичных задач.

**❌ НЕ ДЕЛАЙ:**

8. **Не дублируй** существующие инструменты.
9. **Не забывай про поддержку.**
10. **Не строй всё сразу.**

### Где мы сейчас

Мы разобрали Backstage. Теперь — **Platform as a Product**.

---

## 25.7 Platform as a Product

### 🔌 Проблема: платформа без пользователей

Построили платформу. Но разработчики **не используют**. Или используют и **недовольны**.

**Почему?**

**Платформа сделана «для себя», а не для пользователей.**

**Решение:** Platform as a Product.

### 📊 Что такое Platform as a Product

**Platform as a Product** — подход, при котором платформа рассматривается как **продукт** с **пользователями** (разработчиками).

**Принципы:**

1. **Пользователи — разработчики.**
2. **Их опыт — главное.**
3. **Обратная связь.**
4. **Итерации.**
5. **Метрики.**
6. **Product manager** для платформы.

### 🎯 Разница подходов

**Project approach:**

- Платформа — проект.
- Deadline.
- Deliverable.
- Потом поддержка.

**Product approach:**

- Платформа — продукт.
- Непрерывное развитие.
- Метрики.
- Обратная связь.

**Проблема project approach:** после «запуска» никто не развивает. Устаревает. Разработчики недовольны.

### 🎯 Кто пользователи

**Разработчики:**

- **Backend.**
- **Frontend.**
- **Data.**
- **ML.**
- **Mobile.**

**Разные потребности:**

- Backend — K8s, БД.
- Frontend — CDN, SSR.
- Data — Spark, Airflow.
- ML — GPU, MLflow.

**Платформа должна учитывать.**

### 🎯 Что важно

**1. Developer Experience (DX).**

Опыт разработчика. Удобство, скорость, понятность.

**2. Feedback loop.**

Собирать обратную связь. Регулярно.

**3. Metrics.**

- **Adoption** — сколько используют.
- **Satisfaction** — довольны ли.
- **Time to first deploy** — как быстро новый разработчик.
- **Time to create service** — как быстро создать сервис.

**4. Roadmap.**

Публичный. Основан на feedback.

**5. Support.**

Помощь разработчикам. Быстрая.

### 🎯 Метрики DX

**1. DORA.**

- **Deployment Frequency.**
- **Lead Time for Changes.**
- **MTTR.**
- **Change Failure Rate.**

**2. SPACE.**

- **Satisfaction** — довольны ли.
- **Performance** — производительность.
- **Activity** — активность.
- **Communication** — коммуникация.
- **Efficiency** — эффективность.

**3. DX Metrics.**

- **Time to first PR.**
- **Time to first deploy.**
- **Time to create service.**
- **Number of clicks** для типовых задач.

### 🎯 Как собирать feedback

**1. Регулярные опросы.**

Раз в квартал. NPS, CSAT.

**2. Интервью.**

С разработчиками. Что болит?

**3. Office hours.**

Еженедельно. Открытые вопросы.

**4. Slack channel.**

#platform-support.

**5. Analytics.**

Что используется, что нет.

### 🎯 Roadmap

**Публичный:**

- **Что делаем.**
- **Когда.**
- **Почему.**

**Основан на:**

- **Feedback.**
- **Бизнес-приоритетах.**
- **Технических ограничениях.**

**Обновляется:**

- **Регулярно.**
- **Прозрачно.**

### 🎯 Anti-patterns

**1. Платформа «для себя».**

DevOps строит, что хочет. Не то, что нужно.

**2. Нет feedback.**

Не спрашивают разработчиков.

**3. Нет метрик.**

Не знают, помогает ли.

**4. Нет roadmap.**

Разработчики не знают, что будет.

**5. Нет поддержки.**

«Используйте, но если сломается — сами».

### 🔬 Практика: Platform as a Product

**1. Опрос:**

```yaml
# Ежеквартальный опрос
Вопросы:
  - Насколько вы довольны платформой? (1-5)
  - Что болит больше всего?
  - Что хотели бы видеть?
  - Что мешает использовать?
```

**2. Метрики:**

```promql
# Time to first deploy (новый разработчик)
# Метрика: от onboarding до первого деплоя

# Adoption
# % сервисов, использующих Golden Path

# Satisfaction
# NPS из опросов
```

**3. Roadmap:**

```yaml
Q1:
  - Service catalog
  - Golden path для Go
Q2:
  - Self-service DB
  - Golden path для Python
Q3:
  - Self-service Kafka
  - Preview environments
```

### 💡 Практика: как быть Product

**✅ ОБЯЗАТЕЛЬНО:**

1. **Пользователи — разработчики.**
2. **Feedback** регулярно.
3. **Метрики** DX.
4. **Roadmap** публичный.
5. **Support** быстрый.

**👍 СТОИТ:**

6. **Product manager** для платформы.
7. **Interview** разработчиков.
8. **Analytics** использования.

**❌ НЕ ДЕЛАЙ:**

9. **Не строй «для себя».**
10. **Не игнорируй feedback.**
11. **Не забывай про метрики.**

### Где мы сейчас

Мы разобрали Platform as a Product. Теперь — **метрики DORA и SPACE**.

---

## 25.8 Метрики платформы: DORA и SPACE

### 🔌 Проблема: как измерить успех

Платформа построена. **Как понять, что она работает?**

**Решение:** метрики.

### 📊 DORA

**DORA (DevOps Research and Assessment)** — четыре метрики.

**1. Deployment Frequency.**

Как часто деплоим.

**Elite:** несколько раз в день.
**Low:** раз в месяц.

**2. Lead Time for Changes.**

Время от коммита до прода.

**Elite:** < 1 часа.
**Low:** > 1 месяца.

**3. MTTR (Mean Time To Restore).**

Время восстановления после инцидента.

**Elite:** < 1 часа.
**Low:** > 1 недели.

**4. Change Failure Rate.**

% деплоев, вызвавших инцидент.

**Elite:** 0-15%.
**Low:** 46-60%.

**Как измерять:**

- **Deployment Frequency:** из CI/CD.
- **Lead Time:** из Git + CI/CD.
- **MTTR:** из incident management.
- **CFR:** из incident management.

### 📊 SPACE

**SPACE** — пять измерений продуктивности.

**1. Satisfaction.**

Довольны ли разработчики.

**2. Performance.**

Результаты работы.

**3. Activity.**

Активность (коммиты, PR).

**4. Communication.**

Коммуникация (review, комментарии).

**5. Efficiency.**

Эффективность (flow, отсутствие блокеров).

**Как измерять:**

- **Satisfaction:** опросы.
- **Performance:** метрики бизнеса.
- **Activity:** Git, Jira.
- **Communication:** Git, Slack.
- **Efficiency:** cycle time, throughput.

### 🎯 Метрики платформы

**1. Adoption.**

% команд, использующих платформу.

**2. Time to First Deploy.**

Как быстро новый разработчик деплоит.

**3. Time to Create Service.**

Как быстро создать сервис.

**4. Golden Path Usage.**

% сервисов, созданных через Golden Path.

**5. Self-Service Rate.**

% задач, решённых без DevOps.

**6. Support Tickets.**

Сколько тикетов в DevOps.

**7. Developer Satisfaction.**

NPS.

**8. Platform Reliability.**

Uptime платформы.

### 🎯 Как измерять

**1. Adoption.**

```promql
# % сервисов, использующих платформу
count(services_using_platform) / count(all_services) * 100
```

**2. Time to First Deploy.**

```
От onboarding до первого деплоя
Цель: < 1 дня
```

**3. Time to Create Service.**

```
От запроса до running сервиса
Цель: < 1 часа
```

**4. Self-Service Rate.**

```
% задач, решённых без DevOps
Цель: > 80%
```

**5. Support Tickets.**

```
Тикетов в DevOps в неделю
Цель: уменьшается
```

### 🎯 Цели

**Установи цели:**

- **Adoption:** > 80% через год.
- **Time to First Deploy:** < 1 дня.
- **Time to Create Service:** < 1 часа.
- **Self-Service Rate:** > 80%.
- **Support Tickets:** -50% через год.
- **NPS:** > 30.

**Измеряй регулярно.**

### 🎯 Дашборд

**Grafana dashboard для платформы:**

- **Adoption** (time series).
- **Time to First Deploy** (histogram).
- **Time to Create Service** (histogram).
- **Golden Path Usage** (pie chart).
- **Self-Service Rate** (gauge).
- **Support Tickets** (time series).
- **NPS** (stat).
- **DORA metrics** (time series).

### 🔬 Практика: метрики

**1. DORA из CI/CD:**

```yaml
# GitLab CI
metrics:
  deployment_frequency:
    query: count(deployments) / 30d
    
  lead_time:
    query: avg(commit_to_deploy_time)
    
  mttr:
    query: avg(incident_resolution_time)
    
  change_failure_rate:
    query: count(incidents_after_deploy) / count(deployments)
```

**2. Adoption:**

```promql
# % сервисов, использующих Golden Path
sum(services_created_via_golden_path) / sum(all_services) * 100
```

**3. Support Tickets:**

```promql
# Тикетов в DevOps в неделю
sum(increase(devops_tickets_total[7d]))
```

**4. NPS:**

```yaml
# Из опросов
NPS = % Promoters - % Detractors
```

### 💡 Практика: как правильно измерять

**✅ ОБЯЗАТЕЛЬНО:**

1. **DORA метрики.**
2. **Adoption.**
3. **Time to First Deploy.**
4. **Support Tickets.**
5. **NPS.**

**👍 СТОИТ:**

6. **SPACE метрики.**
7. **Дашборд** в Grafana.
8. **Цели** и регулярный review.

**❌ НЕ ДЕЛАЙ:**

9. **Не измеряй только adoption.**
10. **Не игнорируй качество.**
11. **Не забывай про satisfaction.**

### Где мы сейчас

Мы разобрали метрики. Теперь — **платформенная команда**.

---

## 25.9 Платформенная команда

### 🔌 Проблема: кто строит платформу

Платформу нужно строить. **Кто?**

**Платформенная команда.**

### 📊 Что такое платформенная команда

**Платформенная команда** — команда, которая строит и поддерживает IDP.

**Что делает:**

- **Разрабатывает** платформу.
- **Поддерживает** разработчиков.
- **Собирает** feedback.
- **Итерирует.**

**Не делает:**

- **Не деплоит** за разработчиков.
- **Не тушит** пожары.
- **Не администрирует** вручную.

### 🎯 Роли

**1. Platform Engineer.**

- **Разрабатывает** компоненты платформы.
- **Интегрирует** инструменты.
- **Автоматизирует.**

**2. Platform Product Manager.**

- **Roadmap.**
- **Приоритеты.**
- **Feedback** от разработчиков.
- **Метрики.**

**3. Developer Advocate.**

- **Документация.**
- **Обучение.**
- **Support.**
- **Community.**

**4. SRE (часть команды).**

- **Надёжность** платформы.
- **SLO** для платформы.
- **On-call** для платформы.

### 🎯 Размер команды

**Правило:** 1 платформенный инженер на 10-20 разработчиков.

**Пример:**

- 50 разработчиков → 3-5 платформенных инженеров.
- 200 разработчиков → 10-15 платформенных инженеров.

**Масштабируется лучше, чем DevOps.**

### 🎯 Принципы

**1. Продуктовый подход.**

Платформа — продукт. Разработчики — пользователи.

**2. Внутренний open source.**

Документация, contribution, feedback.

**3. Автоматизация.**

Автоматизировать всё, что можно.

**4. Self-service.**

Разработчики не ждут.

**5. Метрики.**

Измерять всё.

### 🎯 Что делать

**1. Разрабатывать платформу.**

- **Golden paths.**
- **Scaffolder.**
- **Self-service.**

**2. Поддерживать.**

- **Slack channel.**
- **Office hours.**
- **Документация.**

**3. Собирать feedback.**

- **Опросы.**
- **Интервью.**
- **Analytics.**

**4. Итерировать.**

- **Roadmap.**
- **Приоритеты.**

### 🎯 Организационная структура

**Вариант 1: Централизованная.**

Одна платформенная команда.

**Плюсы:**

- **Единые практики.**
- **Просто.**

**Минусы:**

- **Может стать бутылочным горлышком.**
- **Далеко от команд.**

**Вариант 2: Федеративная.**

Платформенные инженеры в каждой команде.

**Плюсы:**

- **Близко к командам.**
- **Быстрый feedback.**

**Минусы:**

- **Сложнее координация.**
- **Дублирование.**

**Вариант 3: Гибрид.**

Центральная команда + embedded.

**Плюсы:**

- **Баланс.**

**Минусы:**

- **Сложнее.**

**Рекомендация:** начать с централизованной, потом гибрид.

### 🎯 Как нанимать

**Что искать:**

- **Опыт в DevOps/SRE.**
- **Опыт в разработке.**
- **Понимание DX.**
- **Коммуникация.**
- **Продуктовое мышление.**

**Не только:**

- **Знание K8s.**
- **Знание Terraform.**
- **Сертификаты.**

**Важнее:**

- **Эмпатия к разработчикам.**
- **Умение строить продукты.**

### 🔬 Практика: платформенная команда

**Пример структуры:**

```
Platform Team (5 человек)
├── Tech Lead (1)
├── Platform Engineers (3)
│   ├── Compute & Networking
│   ├── CI/CD & GitOps
│   └── Observability & Security
└── Developer Advocate (1)
```

**Процессы:**

- **Weekly planning.**
- **Daily standup.**
- **Bi-weekly demo.**
- **Monthly feedback review.**
- **Quarterly roadmap.**

### 💡 Практика: как строить команду

**✅ ОБЯЗАТЕЛЬНО:**

1. **Продуктовый подход.**
2. **1 инженер на 10-20 разработчиков.**
3. **Роли:** инженер, PM, advocate.
4. **Метрики** и feedback.

**👍 СТОИТ:**

5. **Гибридная структура** при росте.
6. **Внутренний open source.**
7. **Регулярные демо.**

**❌ НЕ ДЕЛАЙ:**

8. **Не превращай в DevOps-команду.**
9. **Не игнорируй DX.**
10. **Не забывай про поддержку.**

### Где мы сейчас

Мы разобрали команду. Теперь — **миграция**.

---

## 25.10 Миграция к платформенной инженерии

### 🔌 Проблема: как перейти

Компания работает. DevOps-команда завалена. **Как перейти к платформе?**

### 📊 Стратегия миграции

**Фаза 1: Подготовка (1-2 месяца).**

1. **Оценить боль** разработчиков.
2. **Определить** метрики.
3. **Собрать** команду.
4. **Выбрать** инструменты.

**Фаза 2: Service Catalog (2-3 месяца).**

5. **Развернуть** Backstage.
6. **Зарегистрировать** все сервисы.
7. **Добавить** documentation.
8. **Интегрировать** с инструментами.

**Фаза 3: Golden Path (3-6 месяцев).**

9. **Создать** шаблоны для типовых сервисов.
10. **Автоматизировать** setup.
11. **Обучить** команды.

**Фаза 4: Self-service (6-12 месяцев).**

12. **Добавить** self-service для БД, Kafka.
13. **Guardrails** и approval.
14. **Metrics** и feedback.

**Фаза 5: Platform as a Product (постоянно).**

15. **Метрики** DX.
16. **Roadmap.**
17. **Итерации.**

### 🎯 Пошагово

**1. Оценить боль.**

**Опрос разработчиков:**

- Что болит?
- Сколько ждёте DevOps?
- Что хотели бы автоматизировать?

**2. Определить метрики.**

- **Time to First Deploy.**
- **Support Tickets.**
- **Adoption.**

**3. Собрать команду.**

- **Tech Lead.**
- **Platform Engineers.**
- **PM.**

**4. Выбрать инструменты.**

- **Backstage** для портала.
- **Crossplane** для provisioning.
- **ArgoCD** для GitOps.
- **Terraform** для инфраструктуры.

**5. Развернуть Backstage.**

- **Catalog.**
- **Scaffolder.**
- **TechDocs.**
- **Plugins.**

**6. Зарегистрировать сервисы.**

- **catalog-info.yaml** в каждом репо.
- **Owners.**
- **Dependencies.**

**7. Создать Golden Path.**

- **Шаблоны** для Go, Python.
- **CI/CD** из коробки.
- **Мониторинг** автоматически.

**8. Добавить self-service.**

- **БД.**
- **Kafka.**
- **Secrets.**

**9. Метрики и feedback.**

- **DORA.**
- **Adoption.**
- **NPS.**

**10. Итерировать.**

- **Roadmap.**
- **Приоритеты.**

### 🎯 Anti-patterns миграции

**1. Big Bang.**

Мигрировать всё сразу. **Плохо.**

**Решение:** постепенно.

**2. Без метрик.**

Не знаем, помогает ли. **Плохо.**

**Решение:** метрики с самого начала.

**3. Без feedback.**

Не спрашиваем разработчиков. **Плохо.**

**Решение:** регулярный feedback.

**4. Mandatory.**

Заставляем использовать. **Плохо.**

**Решение:** делать проще, чем альтернативы.

**5. Без поддержки.**

Запустили и забыли. **Плохо.**

**Решение:** ongoing support.

### 🎯 Когда НЕ нужна платформа

**Не нужна, если:**

- **< 20 разработчиков.**
- **Мало микросервисов.**
- **Нет боли.**
- **Нет ресурсов.**

**Что делать:**

- **DevOps-принципы.**
- **Автоматизация.**
- **Документация.**

**Не строй платформу ради платформы.**

### 🎯 Success criteria

**Через год:**

- **Adoption:** > 80%.
- **Time to First Deploy:** < 1 дня.
- **Support Tickets:** -50%.
- **NPS:** > 30.
- **DORA:** elite.

**Если не достигнуто — что-то не так.**

### 🔬 Практика: миграция

**1. Опрос:**

```yaml
# Ежеквартальный опрос
- Сколько времени ждёте DevOps? (часы)
- Что болит? (open)
- Что хотели бы автоматизировать? (multi)
- NPS (0-10)
```

**2. Метрики:**

```promql
# Time to First Deploy
avg(time_to_first_deploy)

# Support Tickets
sum(increase(devops_tickets_total[7d]))

# Adoption
sum(services_using_platform) / sum(all_services) * 100
```

**3. Roadmap:**

```yaml
Q1:
  - Backstage catalog
  - DORA metrics
Q2:
  - Golden path Go
  - Self-service DB
Q3:
  - Golden path Python
  - Self-service Kafka
Q4:
  - Preview environments
  - Advanced self-service
```

### 💡 Практика: как мигрировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Оценить боль.**
2. **Определить метрики.**
3. **Постепенно.**
4. **Feedback** регулярно.
5. **Support.**

**👍 СТОИТ:**

6. **Backstage** как портал.
7. **Crossplane** для provisioning.
8. **DORA** метрики.

**❌ НЕ ДЕЛАЙ:**

9. **Не Big Bang.**
10. **Не без метрик.**
11. **Не без feedback.**

### Где мы сейчас

Мы разобрали миграцию. Теперь — **антипаттерны**.

---

## 25.11 Антипаттерны платформенной инженерии

### 🔌 Проблема: что может пойти не так

Платформа построена. Но **не работает** как задумано.

**Разберём антипаттерны.**

### 📊 Антипаттерн 1: Платформа как DevOps 2.0

**Что:** платформенная команда делает всё за разработчиков.

**Проблема:** то же бутылочное горлышко.

**Решение:** self-service. Разработчики сами.

### 📊 Антипаттерн 2: Платформа без пользователей

**Что:** строим платформу, но разработчики не используют.

**Проблема:** зря потратили время.

**Решение:** feedback, метрики, продуктовый подход.

### 📊 Антипаттерн 3: Mandatory платформа

**Что:** «Только через платформу».

**Проблема:** разработчики сопротивляются.

**Решение:** делать проще, чем альтернативы.

### 📊 Антипаттерн 4: Платформа «для себя»

**Что:** DevOps строит, что хочет.

**Проблема:** не решает боль разработчиков.

**Решение:** продуктовый подход, feedback.

### 📊 Антипаттерн 5: Слишком много Golden Paths

**Что:** 10 путей.

**Проблема:** confusion.

**Решение:** 1-3 пути. Opinionated.

### 📊 Антипаттерн 6: Без метрик

**Что:** не измеряем.

**Проблема:** не знаем, помогает ли.

**Решение:** DORA, adoption, NPS.

### 📊 Антипаттерн 7: Без поддержки

**Что:** запустили и забыли.

**Проблема:** разработчики не могут использовать.

**Решение:** ongoing support.

### 📊 Антипаттерн 8: Платформа без Product Manager

**Что:** только инженеры.

**Проблема:** нет roadmap, приоритетов.

**Решение:** PM для платформы.

### 📊 Антипаттерн 9: Слишком сложная платформа

**Что:** 100 инструментов.

**Проблема:** разработчики теряются.

**Решение:** простота, abstraction.

### 📊 Антипаттерн 10: Платформа без SLO

**Что:** платформа падает.

**Проблема:** разработчики не могут работать.

**Решение:** SLO для платформы, надёжность.

### 🎯 Как избежать

**1. Продуктовый подход.**

Разработчики — пользователи.

**2. Feedback.**

Регулярно.

**3. Метрики.**

DORA, adoption, NPS.

**4. Простота.**

Меньше инструментов.

**5. Поддержка.**

Ongoing.

**6. SLO.**

Надёжность платформы.

**7. Roadmap.**

Публичный.

**8. PM.**

Product manager.

### 🔬 Практика: аудит платформы

**Вопросы:**

- [ ] Разработчики используют платформу?
- [ ] Есть метрики?
- [ ] Есть feedback?
- [ ] Есть roadmap?
- [ ] Есть поддержка?
- [ ] Есть SLO?
- [ ] Есть PM?
- [ ] Платформа простая?

**Если нет — что-то не так.**

### 💡 Практика: как избежать антипаттернов

**✅ ОБЯЗАТЕЛЬНО:**

1. **Продуктовый подход.**
2. **Feedback** регулярно.
3. **Метрики.**
4. **Поддержка.**
5. **Простота.**

**👍 СТОИТ:**

6. **PM** для платформы.
7. **SLO** для платформы.
8. **Roadmap.**

**❌ НЕ ДЕЛАЙ:**

9. **Не делай DevOps 2.0.**
10. **Не без метрик.**
11. **Не без feedback.**

### Где мы сейчас

Мы разобрали антипаттерны. Теперь — **диагностика**.

---

## 25.12 Диагностика проблем

### 🔌 Проблема: платформа не работает

Платформа построена. Но что-то не так.

### 🔍 Типичные проблемы

**1. Разработчики не используют.**

**Причины:**

- Не знают.
- Не удобно.
- Не решает боль.
- Обязательно, но плохо.

**Диагностика:**

```bash
# Adoption
# % сервисов, использующих платформу
# Если < 50% — проблема

# Опрос
# NPS < 0 — проблема

# Slack
# Мало вопросов — не используют
```

**Решение:**

- **Feedback.**
- **Улучшить UX.**
- **Обучить.**

**2. Time to Create Service большой.**

**Причины:**

- Много шагов.
- Медленные пайплайны.
- Ручные действия.

**Диагностика:**

```bash
# Замер
# От запроса до running сервиса
# Если > 1 дня — проблема
```

**Решение:**

- **Автоматизация.**
- **Golden Path.**

**3. Много support tickets.**

**Причины:**

- Платформа не работает.
- Нет документации.
- Не удобно.

**Диагностика:**

```bash
# Tickets в неделю
# Если растёт — проблема

# Категории
# Что чаще всего спрашивают?
```

**Решение:**

- **Документация.**
- **Self-service.**
- **Автоматизация.**

**4. Платформа падает.**

**Причины:**

- Нет SLO.
- Нет надёжности.
- Мало ресурсов.

**Диагностика:**

```bash
# Uptime платформы
# Если < 99.9% — проблема

# Инциденты
# Сколько было?
```

**Решение:**

- **SLO** для платформы.
- **Надёжность.**
- **Больше ресурсов.**

**5. Нет метрик.**

**Причины:**

- Не настроены.
- Не собираются.

**Диагностика:**

```bash
# Есть дашборд?
# Есть DORA?
# Есть adoption?
```

**Решение:**

- **Настроить** метрики.
- **Дашборд** в Grafana.

**6. Нет roadmap.**

**Причины:**

- Нет PM.
- Нет приоритетов.

**Диагностика:**

```bash
# Есть публичный roadmap?
# Разработчики знают, что будет?
```

**Решение:**

- **PM** для платформы.
- **Roadmap** публичный.

### 🎯 Общие команды

```bash
# Adoption
# Метрика из каталога

# DORA
# Из CI/CD

# Support Tickets
# Из Jira

# NPS
# Из опросов

# Uptime
# Из мониторинга
```

### 🎯 Аудит платформы

**Раз в квартал:**

- **Adoption.**
- **Time to Create Service.**
- **Support Tickets.**
- **NPS.**
- **DORA.**
- **Uptime.**

**Что улучшить?**

### 🔬 Практика: диагностика

```bash
# 1. Adoption
# Опрос: сколько команд использует?
# Если < 50% — проблема

# 2. NPS
# Опрос: 0-10
# Если < 0 — проблема

# 3. Support Tickets
# Jira: tickets в неделю
# Если растёт — проблема

# 4. DORA
# CI/CD метрики
# Если не elite — улучшить

# 5. Uptime
# Prometheus
# Если < 99.9% — улучшить
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Adoption.**
2. **NPS.**
3. **Support Tickets.**
4. **DORA.**
5. **Uptime.**

**👍 СТОИТ:**

6. **Аудит** раз в квартал.
7. **Feedback** от разработчиков.

**❌ НЕ ДЕЛАЙ:**

8. **Не игнорируй метрики.**
9. **Не забывай про feedback.**
10. **Не оставляй без поддержки.**

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **Platform Engineering** | Построение внутренних платформ для разработчиков. |
| **IDP** | Internal Developer Platform. |
| **Golden Path** | Рекомендованный путь для типовых задач. |
| **Self-service** | Разработчик сам делает через платформу. |
| **Backstage** | Портал разработчика от Spotify. |
| **Scaffolder** | Создание сервисов из шаблонов. |
| **TechDocs** | Документация в Backstage. |
| **Platform as a Product** | Платформа — продукт с пользователями. |
| **DX** | Developer Experience. |
| **DORA** | 4 метрики DevOps. |
| **SPACE** | 5 измерений продуктивности. |
| **Adoption** | % команд, использующих платформу. |
| **Time to First Deploy** | Время от onboarding до первого деплоя. |
| **Time to Create Service** | Время создания сервиса. |
| **Crossplane** | Kubernetes-native IaC. |
| **Portal** | UI для разработчиков. |
| **Product Manager** | Роль для платформы. |
| **Developer Advocate** | Роль для поддержки разработчиков. |
| **Roadmap** | План развития платформы. |
| **NPS** | Net Promoter Score. |

---

## Что мы узнали?

- **Платформенная инженерия** — построение IDP для разработчиков.
- **DevOps vs SRE vs Platform:** культура, надёжность, продукт.
- **IDP** — self-service портал, CI/CD, observability.
- **Golden Path** — рекомендованный путь для типовых задач.
- **Self-service** — разработчик сам делает через платформу.
- **Backstage** — портал разработчика: catalog, scaffolder, techdocs.
- **Platform as a Product** — платформа как продукт с пользователями.
- **Метрики:** DORA, SPACE, adoption, NPS.
- **Платформенная команда:** инженеры, PM, advocate.
- **Миграция:** постепенно, с метриками и feedback.
- **Антипаттерны:** DevOps 2.0, mandatory, без метрик.
- **Диагностика:** adoption, NPS, support tickets, DORA.

---

## Типичные ошибки

- ❌ **Платформа «для себя».**
- ❌ **Mandatory платформа.**
- ❌ **Слишком много Golden Paths.**
- ❌ **Без метрик.**
- ❌ **Без feedback.**
- ❌ **Без поддержки.**
- ❌ **Без PM.**
- ❌ **Слишком сложная.**
- ❌ **Без SLO.**
- ❌ **DevOps 2.0.**
- ❌ **Big Bang миграция.**
- ❌ **Не обучать команды.**
- ❌ **Не документировать.**
- ❌ **Игнорировать adoption.**

---

## Для быстрого повторения

- **Platform Engineering:** IDP для разработчиков.
- **DevOps vs SRE vs Platform:** культура, надёжность, продукт.
- **IDP:** портал, CI/CD, observability, security.
- **Golden Path:** opinionated, safe, fast, supported.
- **Self-service:** разработчик сам.
- **Backstage:** catalog, scaffolder, techdocs, plugins.
- **Platform as a Product:** пользователи, feedback, метрики.
- **DORA:** Deployment Frequency, Lead Time, MTTR, CFR.
- **SPACE:** Satisfaction, Performance, Activity, Communication, Efficiency.
- **Команда:** инженеры, PM, advocate.
- **Метрики:** adoption, time to first deploy, NPS.
- **Миграция:** постепенно, feedback.
- **Антипаттерны:** DevOps 2.0, mandatory, без метрик.

---

## Вопросы для самопроверки

1. Что такое платформенная инженерия? Чем отличается от DevOps?
2. Что такое IDP? Какие компоненты?
3. Что такое Golden Path? Принципы?
4. Что такое self-service? Что должно быть self-service?
5. Что такое Backstage? Какие компоненты?
6. Что такое Platform as a Product?
7. Что такое DORA метрики?
8. Что такое SPACE метрики?
9. Как построить платформенную команду?
10. Как мигрировать к платформенной инженерии?
11. Какие антипаттерны платформенной инженерии?
12. Когда НЕ нужна платформа?
13. Как измерить успех платформы?
14. Что такое Crossplane? Зачем нужен?
15. Разработчики не используют платформу — что делать?

---

## Ответы

**1. Платформенная инженерия**

Построение внутренних платформ для разработчиков. Отличие от DevOps: DevOps — культура, Platform Engineering — продукт для разработчиков (self-service).

**2. IDP**

Internal Developer Platform. Портал, CI/CD, observability, security, documentation. Self-service для разработчиков.

**3. Golden Path**

Рекомендованный путь для типовых задач. Opinionated, safe, fast, supported, documented, not mandatory. 1-3 пути.

**4. Self-service**

Разработчик сам делает через платформу. Создание сервисов, деплой, БД, секреты. С guardrails (квоты, approval, audit).

**5. Backstage**

Портал разработчика от Spotify. Catalog, scaffolder, techdocs, plugins. Единая точка входа.

**6. Platform as a Product**

Платформа — продукт. Пользователи — разработчики. Feedback, метрики, roadmap, PM.

**7. DORA**

Deployment Frequency, Lead Time for Changes, MTTR, Change Failure Rate. Измеряются из CI/CD и incident management.

**8. SPACE**

Satisfaction, Performance, Activity, Communication, Efficiency. Пять измерений продуктивности.

**9. Платформенная команда**

Инженеры, PM, advocate, SRE. 1 инженер на 10-20 разработчиков. Продуктовый подход.

**10. Миграция**

Постепенно: оценка боли → метрики → команда → Backstage → Golden Path → Self-service → Platform as a Product.

**11. Антипаттерны**

DevOps 2.0, без пользователей, mandatory, для себя, много Golden Paths, без метрик, без поддержки, без PM.

**12. Когда НЕ нужна**

< 20 разработчиков, мало микросервисов, нет боли, нет ресурсов. Тогда DevOps-принципы + автоматизация.

**13. Метрики успеха**

Adoption > 80%, Time to First Deploy < 1 дня, Support Tickets -50%, NPS > 30, DORA elite.

**14. Crossplane**

Kubernetes-native IaC. Позволяет создавать облачные ресурсы через Kubernetes CRD. Self-service provisioning.

**15. Не используют**

1. Feedback от разработчиков.
2. Улучшить UX.
3. Обучить.
4. Сделать проще альтернатив.

---

## Куда идти дальше?

Мы разобрали платформенную инженерию. Теперь ты знаешь:

- Что такое Platform Engineering.
- DevOps vs SRE vs Platform.
- IDP и Golden Path.
- Self-service и Backstage.
- Platform as a Product.
- DORA и SPACE.
- Платформенная команда.
- Миграция.
- Антипаттерны.
- Диагностика.

Осталась последняя глава:

- **Облака и multi-cloud стратегии** (Глава 26).

**Глава 26: Облака и multi-cloud стратегии.** Погнали. 🚀