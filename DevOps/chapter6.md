# 🔄 Глава 6: CI/CD — конвейер доставки кода (GitLab CI)

**Что вы узнаете:**
- Что такое CI/CD и зачем это нужно.
- Какие этапы должны быть в пайплайне.
- Как писать `.gitlab-ci.yml`: stages, jobs, runners.
- Как собирать Docker-образы в CI и пушить в registry.
- Как запускать тесты параллельно и кэшировать зависимости.
- Как запустить свой приватный registry и интегрировать его с CI/CD.
- Как использовать artifacts и dependencies между стадиями.
- Как настраивать rules и conditions — когда запускать пайплайн.
- Как управлять секретами в CI/CD.

**После прочтения вы сможете:**
- Написать `.gitlab-ci.yml` для Go-проекта.
- Построить пайплайн: build → test → package → push.
- Настроить кэширование для ускорения сборки.
- Запустить приватный registry и пушить в него образы.
- Использовать artifacts для передачи файлов между стадиями.
- Разделять окружения (dev/staging/prod) через environments.

---

## Содержание

- [6.0 Пролог: «у меня работает» vs «работает в проде»](#60-пролог-у-меня-работает-vs-работает-в-проде)
- [6.1 Что такое CI/CD и зачем это нужно](#61-что-такое-cicd-и-зачем-это-нужно)
- [6.2 GitLab CI: первое знакомство](#62-gitlab-ci-первое-знакомство)
- [6.3 `.gitlab-ci.yml`: stages, jobs, runners](#63-gitlab-ciyml-stages-jobs-runners)
- [6.4 Сборка Docker-образа в CI](#64-сборка-docker-образа-в-ci)
- [6.5 Приватный registry: свой Docker Hub](#65-приватный-registry-свой-docker-hub)
- [6.6 Кэширование и ускорение пайплайна](#66-кэширование-и-ускорение-пайплайна)
- [6.7 Artifacts и dependencies: передача файлов между стадиями](#67-artifacts-и-dependencies-передача-файлов-между-стадиями)
- [6.8 Rules: когда запускать пайплайн](#68-rules-когда-запускать-пайплайн)
- [6.9 Environments: dev, staging, production](#69-environments-dev-staging-production)
- [6.10 Секреты в CI/CD](#610-секреты-в-cicd)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 6.0 Пролог: «у меня работает» vs «работает в проде»

Ты написал Go-приложение. Собрал Docker-образ локально. Запустил — работает. Закоммитил. Запушил в GitLab.

Твой коллега клонирует репозиторий, собирает у себя — **не работает**. Ошибка: `go.sum` устарел. Или версия Go другая. Или переменные окружения не те.

Ты деплоишь в прод вручную:

```bash
docker build -t myapp:1.2.3 .
docker push myregistry.com/myapp:1.2.3
ssh prod-server
docker pull myregistry.com/myapp:1.2.3
docker stop myapp
docker rm myapp
docker run -d --name myapp ... myregistry.com/myapp:1.2.3
```

Пять команд, три места, каждый шаг — потенциальная ошибка:

- Забыл обновить версию → запушил `latest`, прод подтянул не то.
- Опечатался в имени образа → запушил не туда.
- Прод-сервер перезагрузился, а ты об этом не знаешь.
- Деплой в 3 часа ночи, потому что «по-другому не успеваем».

**CI/CD решает эти проблемы.**

CI/CD — это автоматизация всего пути от коммита до прода. Ты коммитишь — система сама собирает, тестирует, упаковывает и деплоит. Одинаково для всех. Воспроизводимо. Каждый раз.

В этой главе мы разберём **GitLab CI** — самый популярный self-hosted инструмент для CI/CD. Ты научишься писать пайплайны, которые собирают, тестируют и пушат Docker-образы автоматически.

Это — **Первый путь DevOps (Flow)** в действии. Из Главы 0: ускорить поток создания ценности от коммита до прода.

---

## 6.1 Что такое CI/CD и зачем это нужно

### 🔌 Проблема: ручные процессы не масштабируются

Когда проект маленький, ручные процессы работают: ты помнишь все команды, все шаги, все нюансы. Но по мере роста:

- **Больше разработчиков.** Каждый собирает по-своему. «У меня работает» — не работает у других.
- **Больше сервисов.** 10 микросервисов — 10 разных деплоев.
- **Больше окружений.** Dev, staging, prod — в каждом своя конфигурация.
- **Чаще релизы.** Вместо раза в месяц — несколько раз в день.
- **Больше ошибок.** Человек устаёт, забывает, ошибается.

**CI/CD** — это ответ на эти проблемы. Автоматизация всего пути от кода до прода.

### 📊 Что такое CI и CD

**CI (Continuous Integration)** — непрерывная интеграция. Каждый коммит автоматически:

1. **Собирается** (компилируется, если нужно).
2. **Тестируется** (unit, integration).
3. **Проверяется** (линтеры, статические анализаторы).

Цель: **быстро находить ошибки**. Если ты сломал сборку — узнаешь через минуту, а не через неделю.

**CD (Continuous Delivery)** — непрерывная доставка. Каждый успешный билд:

1. **Готов к деплою** — собран, протестирован, упакован в Docker-образ, загружен в registry.
2. **Может быть задеплоен** одной командой (или кнопкой).

Решение о деплое в прод — **человек**. CD гарантирует, что **всё готово** к деплою.

**CD (Continuous Deployment)** — непрерывный деплой. Каждый успешный билд:

1. **Автоматически деплоится** в прод.
2. Без ручного подтверждения.

**Разница:**

| | Continuous Delivery | Continuous Deployment |
|:---|:---|:---|
| Готово к деплою? | ✅ Да | ✅ Да |
| Кто деплоит? | Человек (кнопка) | Автоматика |
| Когда использовать | Регулируемые индустрии (банки, медицина) | Быстрые команды с хорошими тестами |

### 🎯 Что даёт CI/CD

**1. Быстрая обратная связь.**

Ты коммитишь. Через 2-5 минут видишь: сборка прошла, тесты прошли, образ собран. Если что-то не так — знаешь сразу.

**2. Воспроизводимость.**

Все сборки одинаковы. Нет «у меня работает, у тебя нет». Окружение сборки фиксировано.

**3. Освобождение времени.**

Разработчики не тратят часы на ручной деплой. CI/CD делает это за секунды.

**4. Меньше ошибок.**

Автоматика не забывает обновить версию, не путает окружение, не опечатывается в командах.

**5. История.**

Каждый пайплайн логируется. Видно, кто, когда, что задеплоил. Легко найти регрессию.

### 📈 DORA-метрики: как измерить CI/CD

Вспомни Главу 0 — четыре DORA-метрики:

| Метрика | Что измеряет | Как CI/CD влияет |
|:---|:---|:---|
| **Deployment Frequency** | Как часто деплоим | CI/CD позволяет деплоить чаще |
| **Lead Time for Changes** | Время от коммита до прода | CI/CD сокращает с часов до минут |
| **MTTR** | Время восстановления | CI/CD ускоряет откат и hotfix |
| **Change Failure Rate** | Доля неудачных деплоев | CI/CD снижает через тесты и канареечные деплои |

### 🛠️ Инструменты CI/CD

Популярные:

- **GitLab CI** — встроен в GitLab. `.gitlab-ci.yml`. Самое популярное для self-hosted.
- **GitHub Actions** — встроен в GitHub. `.github/workflows/*.yml`.
- **Jenkins** — старейший. Плагины, свой DSL, groovy.
- **CircleCI** — облачный, быстрый.
- **Drone** — лёгкий, для контейнеров.
- **ArgoCD / Flux** — GitOps для Kubernetes (Глава 20).

Мы разбираем **GitLab CI** — он самый популярный и концептуально похож на остальные.

### 💡 Практика: что должно быть в CI/CD

**✅ ОБЯЗАТЕЛЬНО:**

1. **CI на каждом коммите.** Сборка + тесты. Без этого — не CI.
2. **Одинаковое окружение для сборки.** Все сборки на одном runner'е / в одном образе.
3. **Автоматический пуш образа.** Сборка → push в registry — единый процесс.

**👍 СТОИТ:**

4. **Параллельный запуск тестов.** Если тестов много — разбить на несколько джоб.
5. **Кэширование зависимостей.** Гораздо быстрее.
6. **Автоматический деплой в dev/staging.** Прод — по кнопке или канареечно.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **Автоматический деплой в прод** (Continuous Deployment). Работает не для всех.

**❌ НЕ ДЕЛАЙ:**

8. **Не деплой в прод без тестов.** Это не CI/CD, это быстрый способ положить прод.
9. **Не храни секреты в `.gitlab-ci.yml`.** Только через CI/CD variables.
10. **Не игнорируй упавшие пайплайны.** Красный пайплайн — это stop-the-line (вспомни Andon из Главы 0).

### Где мы сейчас

Мы разобрали, что такое CI/CD и зачем это нужно. Теперь посмотрим на **GitLab CI** — как он устроен.

---

## 6.2 GitLab CI: первое знакомство

### 🔌 Проблема: как GitLab узнаёт, что делать

GitLab — это не просто Git-репозиторий. Это целая платформа, которая включает:

- **Git-репозитории** — хранение кода.
- **GitLab CI/CD** — встроенная система пайплайнов.
- **Container Registry** — встроенный Docker registry.
- **Issues, Merge Requests, Wiki** — управление проектами.

**GitLab CI** работает так: ты описываешь пайплайн в YAML-файле, GitLab его запускает при определённых событиях (коммит, push, merge request).

### 📦 Как GitLab CI устроен

**Ключевые компоненты:**

| Компонент | Что это |
|:---|:---|
| **GitLab Server** | Сервер, где живут репозитории и CI/CD coordinator. |
| **GitLab Runner** | Агент, который **выполняет** джобы. Может быть на отдельной машине. |
| **Executor** | Способ запуска джоб: `docker`, `shell`, `kubernetes`, `docker-machine`. |
| **`.gitlab-ci.yml`** | YAML-файл в корне репозитория с описанием пайплайна. |
| **Pipeline** | Запуск пайплайна для конкретного коммита. |
| **Stage** | Этап пайплайна (build, test, deploy). |
| **Job** | Конкретная задача внутри стадии. |
| **Artifact** | Файл, который джоба сохраняет для других джоб. |

**Как это работает:**

1. Ты коммитишь код в GitLab.
2. GitLab видит `.gitlab-ci.yml` и создаёт **pipeline** для этого коммита.
3. Pipeline состоит из **stages** (build, test, deploy).
4. Каждая stage — из **jobs**.
5. Каждая job выполняется на **runner'е** в контейнере (или shell).
6. Результаты (артефакты, логи) сохраняются в GitLab.

**Схема:**

```
┌──────────────────────────────────────────────────────────────────┐
│                          GitLab Server                            │
│                                                                   │
│  1. Push в репозиторий                                            │
│  2. GitLab читает .gitlab-ci.yml                                  │
│  3. Создаёт pipeline:                                            │
│     Stage build → job build                                       │
│     Stage test → job test-unit, job test-integration              │
│     Stage deploy → job deploy-staging                             │
│                                                                   │
│  4. Отправляет джобы на runner'ы                                  │
└───────────────────────────────┬──────────────────────────────────┘
                                │
                                │ HTTP
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                       GitLab Runner                               │
│                                                                   │
│  Получает джобу:                                                  │
│  - Какой образ использовать (image)                               │
│  - Какие команды выполнить (script)                               │
│  - Какие переменные передать (variables)                          │
│                                                                   │
│  Запускает контейнер, выполняет команды, возвращает результат     │
└──────────────────────────────────────────────────────────────────┘
```

### 🚀 Быстрый старт

**Минимальный `.gitlab-ci.yml`:**

```yaml
stages:
  - build

build-job:
  stage: build
  image: golang:1.22-alpine
  script:
    - echo "Building..."
    - go build -o myapp .
    - echo "Build complete!"
```

**Что произойдёт:**

1. Ты коммитишь этот файл.
2. GitLab видит пайплайн.
3. Запускает runner.
4. Runner качает образ `golang:1.22-alpine`.
5. Выполняет `go build -o myapp .`.
6. Показывает логи.

### 🏃 Runner: где выполняются джобы

**Runner** — это машина/контейнер, который выполняет джобы. Может быть:

- **Shared runners** — общие runner'ы GitLab (если используешь SaaS).
- **Group runners** — runner'ы для группы проектов.
- **Project runners** — для конкретного проекта.
- **Self-hosted runners** — свои машины.

**Executor** — как runner запускает джобы:

| Executor | Что делает |
|:---|:---|
| **docker** | Джоба выполняется в контейнере. Самый популярный. |
| **shell** | Джоба выполняется прямо на машине runner'а. |
| **kubernetes** | Джобы запускаются как Pod'ы в K8s. |
| **docker-machine** | Runner в облаке, создаёт VM для каждой джобы. |

**Для Docker executor** важно:

- Каждая джоба — **новый контейнер**. Состояние между джобами не сохраняется.
- Для кэша — используется **кэш** (подглава 6.6).
- Для передачи файлов — **artifacts** (подглава 6.7).

### 🧪 Как запустить GitLab локально

Для экспериментов можно поднять GitLab в Docker:

```bash
docker run -d --name gitlab \
  --hostname gitlab.local \
  -p 80:80 -p 443:443 -p 2222:22 \
  -v gitlab-config:/etc/gitlab \
  -v gitlab-logs:/var/log/gitlab \
  -v gitlab-data:/var/opt/gitlab \
  gitlab/gitlab-ce:latest
```

Через 5-10 минут GitLab доступен на `http://localhost`. Логин: `root`, пароль — в `/etc/gitlab/initial_root_password`.

Плюс нужно зарегистрировать **runner**:

```bash
docker run -d --name gitlab-runner \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v gitlab-runner-config:/etc/gitlab-runner \
  gitlab/gitlab-runner:latest
```

Регистрация runner'а:

```bash
docker exec -it gitlab-runner gitlab-runner register
# URL: http://gitlab.local/
# token: <из GitLab UI: Settings → CI/CD → Runners>
# executor: docker
# default image: alpine:latest
```

**Для обучения достаточно использовать gitlab.com** — там есть бесплатные shared runners.

### 💡 Практика: как правильно начать с GitLab CI

**✅ ОБЯЗАТЕЛЬНО:**

1. **Храни `.gitlab-ci.yml` в корне репозитория.**
2. **Используй Docker executor** — одинаковое окружение для всех.
3. **Указывай конкретные образы** (`golang:1.22-alpine`, не `golang:latest`).

**👍 СТОИТ:**

4. **Разделяй пайплайн на stages** — build, test, package, deploy.
5. **Делай пайплайн быстрым** — кэш, параллельные джобы.
6. **Настрой protected branches** — чтобы в main/prod можно было пушить только через MR.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **Self-hosted GitLab.** Можно использовать gitlab.com. Self-hosted нужен для compliance или приватности.

**❌ НЕ ДЕЛАЙ:**

8. **Не пиши весь скрипт в одну джобу.** Разделяй.
9. **Не используй `latest` в image.** Внезапно сломается.
10. **Не храни секреты в `.gitlab-ci.yml`.** Используй CI/CD variables.

### Где мы сейчас

Мы разобрали, что такое GitLab CI и как он устроен. Теперь разберём **синтаксис `.gitlab-ci.yml`** — stages, jobs, runners.

---

## 6.3 `.gitlab-ci.yml`: stages, jobs, runners

### 🔌 Проблема: как описать пайплайн

`.gitlab-ci.yml` — это YAML-файл с описанием пайплайна. Он состоит из нескольких сущностей:

- **`stages`** — список этапов пайплайна.
- **Jobs** — конкретные задачи. Каждая job ссылается на stage.
- **`image`** — Docker-образ для job.
- **`script`** — команды для выполнения.
- **`variables`** — переменные.
- **`before_script`** / **`after_script`** — команды до и после.
- **`rules`** — условия запуска.
- **`artifacts`** — файлы, которые job сохраняет.
- **`cache`** — файлы, которые кэшируются между запусками.

### 📦 Структура `.gitlab-ci.yml`

```yaml
# 1. Stages
stages:
  - build
  - test
  - package
  - deploy

# 2. Глобальные переменные
variables:
  GO_VERSION: "1.22"
  APP_NAME: "myapp"

# 3. Глобальные before_script
before_script:
  - echo "Starting job in $CI_JOB_NAME"

# 4. Jobs
build:
  stage: build
  image: golang:1.22-alpine
  script:
    - go build -o myapp .
  artifacts:
    paths:
      - myapp

test:
  stage: test
  image: golang:1.22-alpine
  script:
    - go test ./...

package:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .
    - docker push myapp:$CI_COMMIT_SHA
```

### 🏗️ Stages

**Stages** — это этапы пайплайна. Джобы одного stage выполняются **параллельно**, stages выполняются **последовательно**.

```yaml
stages:
  - build
  - test
  - deploy
```

**Поведение:**

- **Stage build** — джобы этого stage выполняются параллельно.
- Когда все джобы build завершены успешно — начинается stage test.
- Если хоть одна джоба build упала — пайплайн останавливается.

**Схема:**

```
Stage build         Stage test              Stage deploy
┌──────────────┐    ┌──────────────────┐    ┌───────────────┐
│ build-job-1  │    │ test-unit        │    │ deploy-staging│
│ build-job-2  │ →  │ test-integration │ →  │ deploy-prod   │
└──────────────┘    └──────────────────┘    └───────────────┘
    Параллельно         Параллельно             Параллельно
```

**Если stages не указаны** — по умолчанию один stage `test`, все джобы параллельны.

### 🧩 Jobs

**Job** — это конкретная задача. Основные поля:

```yaml
job-name:
  stage: build                    # какой stage
  image: golang:1.22-alpine       # Docker-образ
  before_script:                  # команды до script
    - go version
  script:                         # основные команды
    - go build -o myapp .
  after_script:                   # команды после script
    - echo "Done"
  variables:                      # переменные
    GOOS: linux
    GOARCH: amd64
  artifacts:                      # файлы для следующих джоб
    paths:
      - myapp
    expire_in: 1 week
  cache:                          # кэш между запусками
    paths:
      - .go-cache/
  rules:                          # условия запуска
    - if: $CI_COMMIT_BRANCH == "main"
  tags:                           # теги runner'а
    - docker
```

**Правила именования:**

- Имя джобы — свободное, но не должно совпадать с зарезервированными словами (`image`, `services`, `stages` и др.).
- Если имя начинается с `.` — джоба **не запускается**, используется как шаблон.

**Скрытые джобы (шаблоны):**

```yaml
.go-build:
  image: golang:1.22-alpine
  before_script:
    - go mod download
  script:
    - go build -o myapp .

build-linux:
  extends: .go-build
  variables:
    GOOS: linux

build-mac:
  extends: .go-build
  variables:
    GOOS: darwin
```

`extends` — наследование конфигурации. Полезно для уменьшения дублирования.

### 🖼️ `image` и `services`

**`image`** — Docker-образ для job:

```yaml
image: golang:1.22-alpine
```

Runner использует этот образ для запуска контейнера. Все команды выполняются внутри.

**`services`** — дополнительные сервисы (обычно базы данных, кэши):

```yaml
test-integration:
  image: golang:1.22-alpine
  services:
    - postgres:16-alpine
    - redis:7-alpine
  variables:
    DATABASE_URL: postgres://postgres:secret@postgres:5432/test
    REDIS_URL: redis://redis:6379
  script:
    - go test -tags=integration ./...
```

**Как это работает:**

- Runner запускает основной контейнер (`image`).
- Запускает контейнеры сервисов (`postgres`, `redis`) в той же сети.
- Приложение обращается к сервисам по именам (`postgres`, `redis`).

### 🔄 `before_script` и `after_script`

**`before_script`** выполняется **до** `script`. Обычно для подготовки:

```yaml
before_script:
  - apk add --no-cache git
  - go mod download
```

**`after_script`** выполняется **после** `script`, даже если `script` упал. Обычно для cleanup:

```yaml
after_script:
  - rm -rf /tmp/artifacts
  - echo "Job finished"
```

**Важно:** `after_script` выполняется в **отдельном** shell. Если `script` что-то изменил в текущей директории — `after_script` это не увидит.

### 🎯 Полный пример: Go-приложение

```yaml
stages:
  - build
  - test
  - lint

variables:
  GO_VERSION: "1.22"
  GOPATH: "$CI_PROJECT_DIR/.go"

# Кэшируем Go-модули между запусками
cache:
  paths:
    - .go/pkg/mod/

before_script:
  - mkdir -p .go/pkg/mod

build:
  stage: build
  image: golang:${GO_VERSION}-alpine
  script:
    - go build -o myapp .
  artifacts:
    paths:
      - myapp
    expire_in: 1 day

test:
  stage: test
  image: golang:${GO_VERSION}-alpine
  script:
    - go test -v -cover ./...
  coverage: '/total:\s+\(statements\)\s+(\d+\.\d+)%/'

lint:
  stage: lint
  image: golangci/golangci-lint:latest
  script:
    - golangci-lint run --timeout 5m
  allow_failure: true

test-integration:
  stage: test
  image: golang:${GO_VERSION}-alpine
  services:
    - postgres:16-alpine
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    DATABASE_URL: postgres://test:test@postgres:5432/testdb?sslmode=disable
  script:
    - go test -tags=integration ./...
```

**Что здесь есть:**

- Три stages: build, test, lint.
- Кэш Go-модулей.
- Сборка с артефактом.
- Тесты с покрытием.
- Линтер (allow_failure — не блокирует пайплайн).
- Интеграционные тесты с PostgreSQL.

### 💡 Практика: как правильно писать `.gitlab-ci.yml`

**✅ ОБЯЗАТЕЛЬНО:**

1. **Разделяй пайплайн на stages.** Build отдельно, test отдельно, deploy отдельно.
2. **Используй кэш для зависимостей.** Go modules, npm packages — не качаются заново каждый раз.
3. **Указывай `artifacts` с `expire_in`.** Иначе артефакты займут всё место.

**👍 СТОИТ:**

4. **Используй `extends` для уменьшения дублирования.**
5. **Используй `services` для интеграционных тестов** (БД, Kafka, Redis).
6. **Делай линтеры `allow_failure: true`** — они не должны блокировать релиз.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **`after_script`** — если не нужен cleanup.

**❌ НЕ ДЕЛАЙ:**

8. **Не пиши всё в одну job.** Разделяй по смыслу.
9. **Не используй `latest` в image.** Изменится — пайплайн сломается.
10. **Не храни секреты в `.gitlab-ci.yml`.** Только через CI/CD variables (подглава 6.10).
11. **Не игнорируй упавшие джобы.** Красный пайплайн — это stop-the-line.

### Где мы сейчас

Мы разобрали структуру `.gitlab-ci.yml`: stages, jobs, runners. Теперь разберём **сборку Docker-образа в CI** — самую важную часть для DevOps.

---

## 6.4 Сборка Docker-образа в CI

### 🔌 Проблема: как собрать образ в CI

Вспомни Главу 2 — `docker build` требует Docker daemon. Но CI-джоба выполняется в контейнере, где нет Docker daemon.

Как собрать образ внутри CI-джобы?

### 🎯 Решение: Docker-in-Docker

**Docker-in-Docker (DinD)** — запуск Docker daemon **внутри** контейнера. GitLab CI поддерживает это через `services`.

```yaml
build-image:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
    DOCKER_TLS_VERIFY: 1
    DOCKER_CERT_PATH: "$DOCKER_TLS_CERTDIR/client"
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .
    - docker push myapp:$CI_COMMIT_SHA
```

**Что произошло:**

1. Job запускается в контейнере `docker:24` (клиент Docker).
2. Runner запускает второй контейнер `docker:24-dind` (Docker daemon).
3. Клиент подключается к daemon через TCP.
4. `docker build` собирает образ **внутри** `dind`-контейнера.
5. `docker push` пушит его в registry.

**Схема:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Runner (docker executor)                 │
│                                                             │
│  ┌───────────────────────┐    ┌──────────────────────────┐ │
│  │ job контейнер         │    │ dind контейнер           │ │
│  │ (docker:24)           │    │ (docker:24-dind)         │ │
│  │                       │    │                          │ │
│  │ docker build          │───▶│ Docker daemon            │ │
│  │ docker push           │    │ (собирает образ)         │ │
│  │                       │    │                          │ │
│  └───────────────────────┘    └──────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### ⚠️ Проблемы с DinD

**Docker-in-Docker имеет проблемы:**

1. **Безопасность.** DinD требует `--privileged`, что опасно.
2. **Производительность.** Вложенная файловая система — медленнее.
3. **Кэш.** Кэш dind-контейнера теряется между джобами.

### 🎯 Альтернатива: Kaniko

**Kaniko** — инструмент от Google для сборки Docker-образов **без Docker daemon**. Он читает Dockerfile, выполняет каждый шаг в userspace и создаёт образ без привилегий.

```yaml
build-image:
  stage: package
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  script:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(echo -n $CI_REGISTRY_USER:$CI_REGISTRY_PASSWORD | base64)\"}}}" > /kaniko/.docker/config.json
    - /kaniko/executor
        --context $CI_PROJECT_DIR
        --dockerfile $CI_PROJECT_DIR/Dockerfile
        --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        --destination $CI_REGISTRY_IMAGE:latest
        --cache=true
```

**Преимущества Kaniko:**

- **Безопасно.** Не требует привилегий, не запускает daemon.
- **Быстро.** Может работать параллельно с несколькими джобами.
- **Кэш в registry.** Кэширует слои в registry, а не на диске.

**Недостатки:**

- **Не поддерживает все фичи Docker.** Например, `RUN --mount=type=secret` работает, но `docker buildx build --platform` — нет.
- **Меньше документации.** Меньше примеров, чем для DinD.

### 🎯 Альтернатива: Buildah

**Buildah** — инструмент от Red Hat для сборки образов без daemon. Похож на Kaniko, но на другой архитектуре.

```yaml
build-image:
  stage: package
  image: quay.io/buildah/stable
  script:
    - buildah bud -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - buildah push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

### 📊 Сравнение подходов

| Подход | Безопасность | Скорость | Совместимость | Кэш |
|:---|:---|:---|:---|:---|
| **DinD** | Низкая (privileged) | Средняя | Полная | Теряется между джобами |
| **Kaniko** | Высокая | Быстрая | Почти полная | В registry |
| **Buildah** | Высокая | Быстрая | Почти полная | В registry |

**Рекомендация:**

- **Для обучения и простых проектов — DinD.**
- **Для production — Kaniko.** Безопаснее и быстрее.

### 🎯 Полный пример: сборка и пуш

```yaml
stages:
  - build
  - test
  - package

variables:
  GO_VERSION: "1.22"
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"

# Build stage
build:
  stage: build
  image: golang:${GO_VERSION}-alpine
  cache:
    paths:
      - .go/pkg/mod/
  variables:
    GOPATH: "$CI_PROJECT_DIR/.go"
  script:
    - mkdir -p .go/pkg/mod
    - go mod download
    - go build -o myapp .
  artifacts:
    paths:
      - myapp
    expire_in: 1 day

# Test stage
test:
  stage: test
  image: golang:${GO_VERSION}-alpine
  cache:
    paths:
      - .go/pkg/mod/
  variables:
    GOPATH: "$CI_PROJECT_DIR/.go"
  script:
    - go test -v ./...

# Package stage: собрать Docker-образ и запушить в GitLab Registry
package:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_HOST: tcp://docker:2376
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

**Что произошло:**

1. **Build** — собрал Go-бинарник, сохранил как артефакт.
2. **Test** — запустил тесты.
3. **Package** — собрал Docker-образ, запушил в GitLab Container Registry.

**GitLab Container Registry** — встроенный registry в GitLab. Не нужен внешний — он уже есть.

### 🏷️ Тегирование образов

**Правила тегирования:**

```bash
# По коммиту (уникальный)
docker tag myapp:$CI_COMMIT_SHA

# По ветке
docker tag myapp:$CI_COMMIT_REF_SLUG    # например, "main" или "feature-xyz"

# По тегу git
docker tag myapp:$CI_COMMIT_TAG         # например, "v1.2.3"

# latest — только для main
docker tag myapp:latest

# Semver
docker tag myapp:1.2.3
```

**Стратегия:**

```yaml
script:
  # Всегда — по коммиту
  - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  
  # Для main — latest
  - if [ "$CI_COMMIT_BRANCH" = "main" ]; then
      docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest;
    fi
  
  # Для git-тега v1.2.3 — semver
  - if [ -n "$CI_COMMIT_TAG" ]; then
      docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG;
    fi
  
  - docker push $CI_REGISTRY_IMAGE --all-tags
```

### 💡 Практика: как правильно собирать образ в CI

**✅ ОБЯЗАТЕЛЬНО:**

1. **Используй GitLab Container Registry** (`$CI_REGISTRY_IMAGE`). Он уже встроен.
2. **Тегируй по коммиту** — уникальный тег для каждого билда. Легко откатиться.
3. **Используй DinD или Kaniko** — в зависимости от требований безопасности.

**👍 СТОИТ:**

4. **Пушить `latest` только для main.**
5. **Пушить semver для git-тегов.**
6. **Использовать `--cache-from` для ускорения сборки:**
   ```bash
   docker pull $CI_REGISTRY_IMAGE:latest || true
   docker build --cache-from $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
   ```

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **Multi-arch builds** (`buildx`) — если нужно для разных архитектур.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй privileged DinD без необходимости.** Используй Kaniko.
9. **Не пушить образы с секретами.** Проверяй `.dockerignore` (подглава 2.7).
10. **Не пушить в public registry** без причины. GitLab Registry — приватный по умолчанию.

### Где мы сейчас

Мы разобрали сборку Docker-образа в CI. Теперь — про **приватный registry** и его интеграцию.

---

## 6.5 Приватный registry: свой Docker Hub

### 🔌 Проблема: где хранить образы

В подглаве 6.4 мы пушили в GitLab Container Registry. Это удобно, если используешь GitLab. Но:

- Не все используют GitLab.
- Для некоторых задач нужен **отдельный** registry (например, для staging vs prod).
- Хочется **свой** registry для полного контроля.

**Решение:** запустить **свой приватный registry**.

### 📦 Что такое registry

**Registry** — это сервер, который хранит Docker-образы и раздаёт их по HTTP. Docker Hub — публичный registry. GitLab Container Registry — встроенный в GitLab.

**Простой registry** — образ `registry:2` от Docker. Один контейнер, ~30 МБ.

### 🚀 Запуск registry

```bash
# Создать директорию для данных
mkdir -p /opt/registry/data

# Запустить registry
docker run -d \
  --name registry \
  --restart=always \
  -p 5000:5000 \
  -v /opt/registry/data:/var/lib/registry \
  registry:2

# Проверить, что работает
curl http://localhost:5000/v2/
# {}
```

**Что произошло:**

- Registry запущен на порту 5000.
- Данные хранятся в `/opt/registry/data` (volume).
- API доступно на `/v2/`.

### 🧪 Практика: работа с registry

```bash
# 1. Скачай тестовый образ
docker pull alpine:latest

# 2. Помечай его для нашего registry
docker tag alpine:latest localhost:5000/alpine:test

# 3. Отправь
docker push localhost:5000/alpine:test

# 4. Проверь, что образ в registry
curl http://localhost:5000/v2/alpine/tags/list
# {"name":"alpine","tags":["test"]}

# 5. Удали локальный образ
docker rmi localhost:5000/alpine:test
docker rmi alpine:latest

# 6. Скачай из registry
docker pull localhost:5000/alpine:test

# 7. Запусти
docker run --rm localhost:5000/alpine:test echo "Hello from private registry!"
```

### 🔐 Базовая аутентификация

По умолчанию registry **не требует авторизации**. Это опасно: любой, кто получит доступ к порту, может пушить и пуллить.

**Добавляем htpasswd-авторизацию:**

```bash
# 1. Создать файл с пользователями
mkdir -p /opt/registry/auth

# Установить apache2-utils (или использовать docker)
docker run --rm --entrypoint htpasswd httpd:2 -Bbn myuser mypassword > /opt/registry/auth/htpasswd

# 2. Остановить и удалить старый registry
docker rm -f registry

# 3. Запустить с авторизацией
docker run -d \
  --name registry \
  --restart=always \
  -p 5000:5000 \
  -v /opt/registry/data:/var/lib/registry \
  -v /opt/registry/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  registry:2

# 4. Логин
docker login localhost:5000
# Username: myuser
# Password: mypassword

# 5. Теперь push/pull работают с авторизацией
```

### 🔒 TLS: без него Docker не пушит на удалённый хост

**Важный нюанс:** Docker по умолчанию пушит только в `localhost:5000` по HTTP. Для удалённого registry **обязателен HTTPS**.

```bash
# Docker пытается пушить на registry.example.com:5000
docker push registry.example.com:5000/myapp:latest
# Error: Get https://registry.example.com:5000/v2/: x509: certificate signed by unknown authority
```

**Два варианта:**

**Вариант 1: Let's Encrypt (для публичного домена).**

Используй Nginx/Caddy как reverse proxy перед registry. Nginx терминирует TLS, registry работает по HTTP.

```nginx
# /etc/nginx/sites-available/registry
server {
    listen 443 ssl;
    server_name registry.example.com;
    
    ssl_certificate /etc/letsencrypt/live/registry.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/registry.example.com/privkey.pem;
    
    client_max_body_size 2G;    # для больших образов
    
    location / {
        proxy_pass http://localhost:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Вариант 2: self-signed сертификат (для внутренней сети).**

```bash
# 1. Сгенерировать self-signed сертификат
mkdir -p /opt/registry/certs
openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout /opt/registry/certs/domain.key \
  -x509 -days 365 \
  -out /opt/registry/certs/domain.crt \
  -subj "/CN=registry.local"

# 2. Запустить registry с TLS
docker run -d \
  --name registry \
  --restart=always \
  -p 5000:5000 \
  -v /opt/registry/data:/var/lib/registry \
  -v /opt/registry/auth:/auth \
  -v /opt/registry/certs:/certs \
  -e "REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt" \
  -e "REGISTRY_HTTP_TLS_KEY=/certs/domain.key" \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  registry:2

# 3. Добавить сертификат в доверенные на клиентской машине (Linux)
sudo mkdir -p /etc/docker/certs.d/registry.local:5000
sudo cp /opt/registry/certs/domain.crt /etc/docker/certs.d/registry.local:5000/ca.crt
sudo systemctl restart docker

# На macOS (Docker Desktop):
# Preferences → Docker Engine → добавь insecure-registries или установи сертификат
```

### 🔗 Интеграция registry с GitLab CI

```yaml
variables:
  REGISTRY_URL: registry.example.com:5000
  REGISTRY_IMAGE: $REGISTRY_URL/myapp

build-and-push:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_HOST: tcp://docker:2376
  before_script:
    # Логин в наш registry
    - echo "$REGISTRY_PASSWORD" | docker login -u "$REGISTRY_USER" --password-stdin $REGISTRY_URL
  script:
    - docker build -t $REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $REGISTRY_IMAGE:$CI_COMMIT_SHA
```

**Переменные (`REGISTRY_URL`, `REGISTRY_USER`, `REGISTRY_PASSWORD`)** задаются в GitLab UI: **Settings → CI/CD → Variables**. Помечай `REGISTRY_PASSWORD` как `Masked` и `Protected`.

### 🧹 Очистка старых образов

Registry **не удаляет** старые образы автоматически. Через год у тебя будут тысячи образов, занимающих терабайты.

**Очистка через garbage collection:**

```bash
# 1. Остановить registry
docker stop registry

# 2. Запустить garbage collection
docker run --rm \
  -v /opt/registry/data:/var/lib/registry \
  registry:2 garbage-collect /etc/registry/config.yml

# 3. Запустить registry
docker start registry
```

**Garbage collection удаляет** слои, на которые не ссылается ни один манифест. Но **не удаляет** сами манифесты — нужно сначала удалить теги через API.

**Удаление тега:**

```bash
# Получить digest манифеста
curl -u myuser:mypassword -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  http://localhost:5000/v2/myapp/manifests/v1.0.0 -I | grep Docker-Content-Digest
# Docker-Content-Digest: sha256:abc123...

# Удалить манифест
curl -u myuser:mypassword -X DELETE \
  http://localhost:5000/v2/myapp/manifests/sha256:abc123...
```

**Автоматизация очистки:**

Для production лучше использовать:

- **Harbor** — расширенный registry с retention policies, сканированием, UI.
- **GitLab Container Registry** — встроенные retention policies.
- **AWS ECR** — lifecycle policies.

### 📊 Сравнение registry-решений

| Решение | Сложность | Фичи | Когда использовать |
|:---|:---|:---|:---|
| **`registry:2`** | Простое | Минимум | Локально, dev, обучение |
| **Harbor** | Среднее | UI, сканирование, retention, RBAC | Production, enterprise |
| **GitLab Registry** | Встроенное | Retention, интеграция | Если используешь GitLab |
| **AWS ECR / GCP GCR** | Управляемое | Всё | В облаке |

### 💡 Практика: как правильно работать с registry

**✅ ОБЯЗАТЕЛЬНО:**

1. **Для production — используй Harbor или managed registry** (ECR, GCR). `registry:2` — для обучения.
2. **Всегда включай аутентификацию.** Даже для внутреннего registry.
3. **Используй TLS.** Без него Docker откажется пушить на удалённый хост.
4. **Настрой retention policies.** Иначе диск закончится.

**👍 СТОИТ:**

5. **Используй reverse proxy (Nginx) перед registry** для TLS-терминации.
6. **Автоматизируй очистку** через скрипт или встроенные retention.
7. **Мониторь registry** — метрики, размер, количество образов.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

8. **Self-hosted registry** — если используешь managed (ECR, GitLab Registry).

**❌ НЕ ДЕЛАЙ:**

9. **Не используй `registry:2` в production без TLS и auth.** Публичный доступ к твоим образам — это компрометация.
10. **Не храни образы без ограничений.** Терабайты места, замедление pull.
11. **Не забывай про garbage collection.** Без него registry раздувается.

### Где мы сейчас

Мы разобрали приватный registry. Теперь — **кэширование** и **ускорение пайплайна**.

---

## 6.6 Кэширование и ускорение пайплайна

### 🔌 Проблема: пайплайн слишком долгий

Каждый коммит запускает пайплайн. Если он длится 30 минут — разработчики не ждут, коммитят дальше, ломают сборку. Через час — 10 сломанных сборок.

**Быстрый пайплайн — критически важен.** Он должен завершаться за **5-10 минут**.

Что замедляет пайплайн:

- **Скачивание зависимостей** (go mod download, npm install).
- **Сборка образа с нуля** (без кэша слоёв).
- **Последовательные джобы** (когда могли бы идти параллельно).
- **Большие артефакты** (передача между джобами).

### 🚀 Кэш в GitLab CI

**Кэш** сохраняет файлы между **разными запусками** пайплайна (для одного проекта/ветки). В отличие от artifacts (которые передаются **между джобами** одного пайплайна).

```yaml
cache:
  key: "$CI_COMMIT_REF_SLUG"    # кэш для конкретной ветки
  paths:
    - .go/pkg/mod/
    - node_modules/
    - ~/.cache/go-build/
```

**Ключевые параметры:**

| Параметр | Что делает |
|:---|:---|
| `key` | Идентификатор кэша. Разные ветки — разные кэши. |
| `paths` | Что кэшировать. |
| `policy` | `pull` (только скачивать), `push` (только сохранять), `pull-push` (по умолчанию). |
| `when` | `on_success`, `on_failure`, `always`. |

**Примеры `key`:**

```yaml
# По ветке (кэш отдельно для каждой ветки)
key: "$CI_COMMIT_REF_SLUG"

# По файлу (кэш инвалидируется при изменении файла)
key:
  files:
    - go.sum
    - go.mod

# Глобальный (общий для всего проекта)
key: global

# По job (кэш отдельно для каждой джобы)
key: "$CI_JOB_NAME"
```

### 📦 Кэш для Go

```yaml
variables:
  GOPATH: "$CI_PROJECT_DIR/.go"
  GOCACHE: "$CI_PROJECT_DIR/.go-build"

cache:
  key: "${CI_COMMIT_REF_SLUG}-go"
  paths:
    - .go/pkg/mod/
    - .go-build/

before_script:
  - mkdir -p .go/pkg/mod .go-build

build:
  stage: build
  image: golang:1.22-alpine
  script:
    - go mod download
    - go build -o myapp .
```

**Что кэшируется:**

- `.go/pkg/mod/` — скачанные модули.
- `.go-build/` — кэш компиляции.

**Эффект:** первый запуск — 2 минуты. Последующие — 30 секунд.

### 📦 Кэш для Docker-слоёв

Кэш Docker-слоёв **не сохраняется** между запусками DinD автоматически. Нужно явно указать `--cache-from`:

```yaml
build-image:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin $CI_REGISTRY
    # Скачать предыдущий образ для кэша
    - docker pull $CI_REGISTRY_IMAGE:latest || true
  script:
    - |
      docker build \
        --cache-from $CI_REGISTRY_IMAGE:latest \
        --build-arg BUILDKIT_INLINE_CACHE=1 \
        -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA \
        -t $CI_REGISTRY_IMAGE:latest \
        .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
```

**Что произошло:**

- Docker скачал `$CI_REGISTRY_IMAGE:latest` — предыдущий образ.
- `--cache-from` использовал слои из этого образа.
- `BUILDKIT_INLINE_CACHE=1` встроил кэш в образ при пуше.
- При следующей сборке `--cache-from` снова восстановит кэш.

**С BuildKit:**

```bash
docker buildx build \
  --cache-from type=registry,ref=$CI_REGISTRY_IMAGE:cache \
  --cache-to type=registry,ref=$CI_REGISTRY_IMAGE:cache,mode=max \
  -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA \
  --push \
  .
```

BuildKit хранит кэш в **отдельном теге** в registry. Эффективнее.

### ⚡ Параллельные джобы

Если джобы **не зависят** друг от друга — они выполняются параллельно **в одном stage**:

```yaml
test-unit:
  stage: test
  script: go test -short ./...

test-integration:
  stage: test
  script: go test -tags=integration ./...

test-lint:
  stage: test
  script: golangci-lint run
```

Все три запустятся **одновременно**. Пайплайн идёт за время самой долгой.

**DAG (Directed Acyclic Graph) с `needs`:**

Если джоба зависит **только от одной** джобы, но не от всего stage, используй `needs`:

```yaml
build:
  stage: build
  script: go build -o myapp .
  artifacts:
    paths:
      - myapp

test-unit:
  stage: test
  needs: [build]      # не ждёт весь stage build, только эту джобу
  script: go test -short ./...

test-integration:
  stage: test
  needs: [build]
  script: go test -tags=integration ./...

package:
  stage: package
  needs: [test-unit]  # запускается сразу после test-unit
  script: docker build -t myapp .
```

**DAG-граф:**

```
build ──┬──▶ test-unit ──▶ package
        └──▶ test-integration
```

`package` не ждёт `test-integration`. Быстрее.

### 🎯 Как ускорить пайплайн

**1. Кэшируй зависимости.**

```yaml
cache:
  paths:
    - .go/pkg/mod/
```

**2. Используй `--cache-from` для Docker.**

**3. Параллельные джобы в stage.**

**4. DAG через `needs`.**

**5. Маленькие образы.**

`alpine` вместо `ubuntu` — в разы быстрее pull.

**6. Не собирай всё в одном job.**

Разделяй: build, test, lint, package.

**7. Используй `interruptible: true`.**

Если новый коммит — старый пайплайн отменяется.

```yaml
build:
  interruptible: true
```

**8. Отключай лишнее.**

Логирование, verbose-режимы — только когда нужно.

### 📊 Типичные тайминги

| Джоба | Без кэша | С кэшем |
|:---|:---|:---|
| Go mod download | 1-2 мин | 5 сек |
| Go build | 1 мин | 20 сек |
| Go test | 2 мин | 1 мин |
| Docker build | 5 мин | 1 мин |
| Push в registry | 1 мин | 30 сек |
| **Всего** | **10 мин** | **3 мин** |

### 💡 Практика: как правильно ускорять пайплайн

**✅ ОБЯЗАТЕЛЬНО:**

1. **Кэшируй зависимости** (go mod, npm, pip).
2. **Кэшируй Docker-слои** через `--cache-from`.
3. **Разделяй джобы.** Не пиши всё в один job.
4. **Параллельные джобы** в одном stage.

**👍 СТОИТ:**

5. **DAG через `needs`** — быстрее чем sequential stages.
6. **`interruptible: true`** — отменяет старые пайплайны при новом коммите.
7. **Маленькие базовые образы** — alpine, distroless.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

8. **BuildKit cache в registry** — сложнее, но эффективнее.

**❌ НЕ ДЕЛАЙ:**

9. **Не кэшируй `node_modules` между разными версиями Node.** Кэш сломается.
10. **Не кэшируй всё подряд.** Кэш должен быть маленьким и актуальным.
11. **Не игнорируй длительность пайплайна.** Если больше 15 минут — что-то не так.

### Где мы сейчас

Мы разобрали кэширование. Теперь — **artifacts и dependencies** — передача файлов между джобами.

---

## 6.7 Artifacts и dependencies: передача файлов между стадиями

### 🔌 Проблема: файлы не переходят между джобами

Каждая джоба выполняется в **новом контейнере**. Файлы, созданные в одной джобе, **не видны** в другой.

Если `build` создал бинарник `myapp`, а `package` хочет его упаковать в Docker-образ — `package` не увидит `myapp` без специального механизма.

**Решение:** **artifacts**.

### 📦 Artifacts

**Artifacts** — это файлы, которые джоба сохраняет в GitLab. Другие джобы могут их скачать.

```yaml
build:
  stage: build
  script:
    - go build -o myapp .
  artifacts:
    paths:
      - myapp
    expire_in: 1 day
```

**Что произошло:**

1. Джоба `build` выполнилась, создала `myapp`.
2. GitLab сохранил `myapp` как артефакт.
3. Джоба `package` (в следующем stage) **автоматически** скачает `myapp` перед запуском.

### 📊 Параметры artifacts

```yaml
artifacts:
  paths:
    - myapp
    - build/
  exclude:
    - build/tmp/
  expire_in: 1 week
  when: on_success      # on_success, on_failure, always
  name: "build-$CI_COMMIT_SHA"
  reports:
    junit: test-results.xml
    coverage_report:
      coverage_format: cobertura
      path: coverage.xml
```

| Параметр | Что делает |
|:---|:---|
| `paths` | Что сохранять |
| `exclude` | Что исключить |
| `expire_in` | Когда удалить (по умолчанию — 30 дней) |
| `when` | Когда сохранять |
| `name` | Имя артефакта |
| `reports` | Специальные отчёты (JUnit, coverage) |

### 🔗 Dependencies

По умолчанию джоба **скачивает артефакты всех предыдущих stages**. Это может быть медленно. **`dependencies`** ограничивает, какие артефакты скачивать.

```yaml
build:
  stage: build
  script: go build -o myapp .
  artifacts:
    paths:
      - myapp

test:
  stage: test
  dependencies: []       # не скачивать артефакты
  script: go test ./...

package:
  stage: package
  dependencies:
    - build              # только артефакты build
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .
```

**Правило:**

- Если джоба **не нуждается** в артефактах — `dependencies: []`.
- Если нужны артефакты **конкретных** джоб — перечисли их.
- Иначе — скачаются все.

### 🎯 Пример: сборка Go-бинарника и упаковка в Docker

```yaml
stages:
  - build
  - test
  - package

build:
  stage: build
  image: golang:1.22-alpine
  script:
    - go build -o myapp .
  artifacts:
    paths:
      - myapp
    expire_in: 1 day

test:
  stage: test
  image: golang:1.22-alpine
  dependencies: []       # не нужны артефакты
  script:
    - go test ./...

package:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  dependencies:
    - build              # нужен myapp из build
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin $CI_REGISTRY
  script:
    # myapp доступен в текущей директории
    - ls -la myapp
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

**Что произошло:**

1. `build` собрал `myapp` и сохранил как артефакт.
2. `test` не скачивает артефакты — быстрее.
3. `package` скачал `myapp` из артефакта `build`, использовал в Dockerfile.

**Dockerfile для этого примера:**

```dockerfile
FROM alpine:3.19
COPY myapp /myapp
CMD ["/myapp"]
```

Go-бинарник собран **вне** Dockerfile (в CI). Dockerfile просто копирует готовый бинарник. Это быстрее, чем multi-stage build в самом Dockerfile.

### 🎯 JUnit-отчёты

GitLab CI умеет показывать результаты тестов, если они в JUnit формате:

```yaml
test:
  stage: test
  script:
    - go test -v ./... | go-junit-report > report.xml
  artifacts:
    reports:
      junit: report.xml
```

GitLab покажет:

- Сколько тестов прошло.
- Сколько упало.
- Названия упавших тестов.
- В UI: **CI/CD → Pipelines → Tests**.

### 🎯 Coverage-отчёты

```yaml
test:
  stage: test
  script:
    - go test -coverprofile=coverage.out ./...
    - go tool cover -func=coverage.out
  coverage: '/total:\s+\(statements\)\s+(\d+\.\d+)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
```

**`coverage:`** — regex, по которому GitLab находит процент покрытия в логах. Показывается в badge.

### 💡 Практика: как правильно работать с artifacts

**✅ ОБЯЗАТЕЛЬНО:**

1. **Всегда указывай `expire_in`.** Иначе артефакты займут всё место.
2. **Используй `dependencies` для ограничения.** Не скачивай лишнее.
3. **Сохраняй только нужное.** Не `paths: [.]` — только конкретные файлы.

**👍 СТОИТ:**

4. **Используй `reports.junit`** для отображения тестов в UI.
5. **Используй `reports.coverage_report`** для покрытия.
6. **Используй `exclude`** для исключения временных файлов.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **`name` артефакта** — если нужно кастомное имя.

**❌ НЕ ДЕЛАЙ:**

8. **Не сохраняй секреты в артефакты.** Они доступны в UI GitLab.
9. **Не кэшируй то, что должно быть в артефактах, и наоборот.** Cache — между запусками, artifacts — между джобами одного запуска.
10. **Не забывай про `dependencies: []`.** По умолчанию скачиваются **все** артефакты предыдущих stage, что замедляет пайплайн.

### Где мы сейчас

Мы разобрали artifacts и dependencies. Теперь — **rules** — условия запуска пайплайна.

---

## 6.8 Rules: когда запускать пайплайн

### 🔌 Проблема: не всё нужно запускать на каждом коммите

Пайплайн имеет разные стадии:

- **Build, test, lint** — нужно запускать на **каждом** коммите.
- **Deploy to staging** — только для ветки `main`.
- **Deploy to production** — только для git-тегов `v*`.

Как это описать?

### 🎯 `rules`

**`rules`** — механизм условий в GitLab CI. Каждое правило проверяется по порядку. Первое подходящее определяет, что делать с джобой.

```yaml
deploy-prod:
  stage: deploy
  script:
    - ./deploy.sh prod
  rules:
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
      when: manual
    - when: never
```

**Что произошло:**

- Если тег вида `v1.2.3` — запустить вручную.
- Иначе — не запускать.

### 📊 Ключевые слова `rules`

| Ключевое слово | Что делает |
|:---|:---|
| `if` | Условие (bash-подобное) |
| `when` | Что делать: `on_success`, `manual`, `always`, `never`, `delayed` |
| `changes` | Запускать, если изменились файлы |
| `exists` | Запускать, если файлы существуют |
| `allow_failure` | Разрешить падение |

### 🎯 Примеры условий

**По ветке:**

```yaml
rules:
  - if: $CI_COMMIT_BRANCH == "main"
  - if: $CI_COMMIT_BRANCH == "develop"
  - if: $CI_COMMIT_BRANCH =~ /^feature\//
```

**По тегу:**

```yaml
rules:
  - if: $CI_COMMIT_TAG
    when: manual
  - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
    when: always
```

**По Merge Request:**

```yaml
rules:
  - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

**По изменённым файлам:**

```yaml
rules:
  - changes:
      - "src/**/*.go"
    when: always
  - when: never
```

**Комбинирование:**

```yaml
rules:
  - if: $CI_COMMIT_BRANCH == "main" && $CI_COMMIT_MESSAGE =~ /\[skip ci\]/
    when: never
  - if: $CI_COMMIT_BRANCH == "main"
    when: always
```

### 🎯 Полный пример: разные окружения

```yaml
stages:
  - build
  - test
  - deploy

# Build и test на каждом коммите
build:
  stage: build
  script: go build -o myapp .
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"

test:
  stage: test
  script: go test ./...
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"

# Deploy to staging для main
deploy-staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - ./deploy.sh staging
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual

# Deploy to production для тегов
deploy-prod:
  stage: deploy
  environment:
    name: production
    url: https://example.com
  script:
    - ./deploy.sh production
  rules:
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
      when: manual
```

**Что произошло:**

| Событие | Build | Test | Deploy staging | Deploy prod |
|:---|:---|:---|:---|:---|
| Merge Request | ✅ | ✅ | ❌ | ❌ |
| Push в main | ✅ | ✅ | ✅ (manual) | ❌ |
| Тег v1.2.3 | ❌ | ❌ | ❌ | ✅ (manual) |

### 🎯 `workflow`: правила для всего пайплайна

**`workflow.rules`** — правила для **всего** пайплайна. Если ни одно правило не подходит — пайплайн не создаётся.

```yaml
workflow:
  rules:
    # Не запускать пайплайн для push в feature-ветки без MR
    - if: $CI_PIPELINE_SOURCE == "push" && $CI_COMMIT_BRANCH != "main"
      when: never
    # Запускать для MR
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    # Запускать для main
    - if: $CI_COMMIT_BRANCH == "main"
    # Запускать для тегов
    - if: $CI_COMMIT_TAG
```

### 💡 Практика: как правильно писать rules

**✅ ОБЯЗАТЕЛЬНО:**

1. **Ограничивай deploy rules.** Не деплой в prod на каждый коммит.
2. **Используй `when: manual` для prod.** Человек нажимает кнопку.
3. **Не запускай тяжёлые джобы на feature-ветках.** Только build и test.

**👍 СТОИТ:**

4. **`changes` для запуска только нужных джоб:**
   ```yaml
   rules:
     - changes:
         - "frontend/**/*"
   ```
5. **`workflow.rules` для ограничения всего пайплайна.**

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **`exists`** для проверки наличия файлов.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `only`/`except`.** Они устарели, используй `rules`.
8. **Не пиши сложные условия без необходимости.** Простые `if`-условия читаются лучше.
9. **Не деплой в prod без тестов.** В rules должно быть `needs: [test]`.

### Где мы сейчас

Мы разобрали rules. Теперь — **environments** — управление окружениями.

---

## 6.9 Environments: dev, staging, production

### 🔌 Проблема: как управлять разными окружениями

В CI/CD обычно три окружения:

- **Development (dev)** — для разработки, деплой на каждый коммит.
- **Staging** — для тестирования, близкое к prod, деплой для main.
- **Production (prod)** — реальные пользователи, деплой по кнопке.

Каждое окружение имеет:

- Свой URL.
- Свои переменные.
- Свои правила деплоя.
- Историю деплоев.

### 🎯 Environments в GitLab CI

**Environments** — это встроенный в GitLab механизм управления окружениями. Ты создаёшь окружение, описываешь его в `environment` джобы, и GitLab:

- Показывает список окружений.
- Показывает текущий деплой.
- Позволяет откатывать.
- Ведёт историю.

```yaml
deploy-staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - ./deploy.sh staging
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
```

**В UI GitLab:**

- **Deployments → Environments** — список окружений.
- У каждого: последний деплой, кто, когда, какая версия.
- Кнопка «Rollback» к предыдущему деплою.
- Кнопка «Re-deploy» для повторного деплоя.

### 🎯 Динамические environments

Можно создавать environment **на лету** — например, для каждого MR:

```yaml
review-app:
  stage: deploy
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_COMMIT_REF_SLUG.review.example.com
    on_stop: stop-review-app
  script:
    - ./deploy.sh review $CI_COMMIT_REF_SLUG
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

stop-review-app:
  stage: deploy
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  script:
    - ./cleanup.sh $CI_COMMIT_REF_SLUG
  when: manual
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: manual
```

**Что произошло:**

- Для каждого MR создаётся environment `review/<branch-name>`.
- Деплой на `https://<branch>.review.example.com`.
- При закрытии MR — удаляется.

**Это — Review Apps.** Мощная фича для preview окружений.

### 🎯 Protected environments

**Protected environments** — окружения, деплой в которые разрешён только определённым пользователям.

**Настройка:**

1. **Settings → CI/CD → Protected environments.**
2. Выбрать окружение (`production`).
3. Выбрать, кто может деплоить (maintainers, specific users).
4. Выбрать, из каких веток (main, tags).

```yaml
deploy-prod:
  stage: deploy
  environment:
    name: production
    url: https://example.com
  script:
    - ./deploy.sh production
  rules:
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
      when: manual
```

Только maintainer может нажать кнопку «Deploy» для prod.

### 🎯 Переменные окружения per environment

Можно задавать **разные переменные** для разных окружений:

**Settings → CI/CD → Variables:**

- `DATABASE_URL` (protected, для production).
- `DATABASE_URL_STAGING` (для staging).

Или через `environment`:

```yaml
deploy-staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  variables:
    APP_ENV: staging
    LOG_LEVEL: debug
    DATABASE_URL: $STAGING_DATABASE_URL
  script:
    - ./deploy.sh

deploy-prod:
  stage: deploy
  environment:
    name: production
    url: https://example.com
  variables:
    APP_ENV: production
    LOG_LEVEL: info
    DATABASE_URL: $PROD_DATABASE_URL
  script:
    - ./deploy.sh
```

### 🎯 Откат (rollback)

GitLab позволяет откатить деплой одной кнопкой:

**Deployments → Environments → production → History:**

- Список предыдущих деплоев.
- Кнопка «Rollback» у каждого.

**Что происходит:** GitLab запускает пайплайн с предыдущим коммитом. Твоя задача — обеспечить, чтобы деплой был **идемпотентным** и умел откатываться.

**Пример скрипта деплоя:**

```bash
#!/bin/bash
# deploy.sh
ENV=$1
VERSION=$2

# 1. Получить текущую версию
CURRENT=$(kubectl get deployment myapp -o jsonpath='{.spec.template.spec.containers[0].image}')

# 2. Задеплоить новую
kubectl set image deployment/myapp myapp=$VERSION

# 3. Ждать готовности
kubectl rollout status deployment/myapp

# 4. Если что-то пошло не так — откатить
if [ $? -ne 0 ]; then
  kubectl rollout undo deployment/myapp
  exit 1
fi
```

### 💡 Практика: как правильно работать с environments

**✅ ОБЯЗАТЕЛЬНО:**

1. **Создавай environment для каждого окружения** (dev, staging, prod).
2. **Указывай `url`** — GitLab покажет ссылку в UI.
3. **Используй protected environments** для prod.

**👍 СТОИТ:**

4. **Review Apps для MR** — каждому MR свой preview.
5. **Разные переменные для разных окружений.**
6. **Автоматический rollback** при неудачном деплое.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **`on_stop`** для cleanup — если нужны временные окружения.

**❌ НЕ ДЕЛАЙ:**

8. **Не деплой в prod автоматически.** Только `when: manual` + protected environment.
9. **Не используй одну переменную DATABASE_URL для всех окружений.** Разные окружения — разные переменные.
10. **Не игнорируй rollback.** Если деплой сломал прод — откатывайся, разбирайся потом.

### Где мы сейчас

Мы разобрали environments. Теперь последняя подглава — **секреты в CI/CD**.

---

## 6.10 Секреты в CI/CD

### 🔌 Проблема: как передать секреты в пайплайн

Твоему пайплайну нужны:

- **Docker registry credentials** — чтобы пушить образы.
- **SSH-ключи** — чтобы деплоить на сервер.
- **API-токены** — для внешних сервисов.
- **DATABASE_URL** — для интеграционных тестов.

Как передать их в пайплайн **безопасно**?

**НЕЛЬЗЯ:**

- Захардкодить в `.gitlab-ci.yml` (видно в Git).
- Положить в артефакты (доступно в UI).
- Положить в кэш (может утечь).

**МОЖНО:**

- **CI/CD variables** (GitLab UI).
- **Secrets manager** (Vault, AWS Secrets Manager).
- **OIDC** (federated identity).

### 🎯 CI/CD Variables

**Variables** задаются в **Settings → CI/CD → Variables** в GitLab UI.

**Типы:**

| Тип | Что означает |
|:---|:---|
| **Variable** | Обычная переменная |
| **File** | Файл (содержимое в переменной, GitLab создаёт файл) |

**Опции:**

| Опция | Что делает |
|:---|:---|
| **Protected** | Доступна только для protected branches/tags |
| **Masked** | Скрыта в логах (показывается как `[MASKED]`) |
| **Expanded** | Раскрывать `$`-переменные внутри |

**Пример:**

```
Key: DOCKER_REGISTRY_PASSWORD
Value: super-secret-123
Type: Variable
Protected: ✅
Masked: ✅
```

**Использование в `.gitlab-ci.yml`:**

```yaml
build:
  stage: build
  script:
    - echo "$DOCKER_REGISTRY_PASSWORD" | docker login -u "$DOCKER_REGISTRY_USER" --password-stdin $DOCKER_REGISTRY
```

### 🎯 Masked: почему важно

**Masked** скрывает значение переменной в логах:

```
# Без Masked:
$ echo "Password is $PASSWORD"
Password is super-secret-123

# С Masked:
$ echo "Password is $PASSWORD"
Password is [MASKED]
```

**Требования к Masked:**

- Не меньше 8 символов.
- Только base64-символы (`A-Z`, `a-z`, `0-9`, `+`, `/`, `=`, `@`, `:`, `.`, `~`).
- Без пробелов и переносов строк.

**Если пароль содержит спецсимволы** (например, `$`, `!`) — Masked может не сработать. Используй base64:

```bash
# В GitLab UI:
Key: DB_PASSWORD_BASE64
Value: <base64-encoded-password>
Masked: ✅

# В CI:
script:
  - export DB_PASSWORD=$(echo $DB_PASSWORD_BASE64 | base64 -d)
```

### 🎯 File-переменные

**File-переменные** — GitLab создаёт файл с содержимым переменной. Путь доступен через переменную.

```yaml
deploy:
  script:
    # Переменная SSH_PRIVATE_KEY имеет тип File
    # Её значение — путь к файлу с ключом
    - chmod 600 $SSH_PRIVATE_KEY
    - ssh -i $SSH_PRIVATE_KEY user@server "docker pull ..."
```

**Зачем:** для многострочных значений (SSH-ключи, сертификаты, JSON-конфиги).

### 🎯 Protected: защита секретов

**Protected** — переменная доступна только в пайплайнах для protected branches/tags.

**Настройка:**

1. **Settings → Repository → Protected branches** — выбрать ветку (например, `main`).
2. **Settings → CI/CD → Variables** — пометить `DATABASE_URL` как Protected.

**Что это даёт:** переменная **не будет** доступна в пайплайне для feature-ветки. Даже если злоумышленник создаст MR из feature-ветки и попытается вытащить секрет — не получится.

### 🎯 Predefined variables

GitLab автоматически предоставляет переменные:

```yaml
script:
  - echo "Commit: $CI_COMMIT_SHA"
  - echo "Branch: $CI_COMMIT_BRANCH"
  - echo "Author: $CI_COMMIT_AUTHOR"
  - echo "Project: $CI_PROJECT_NAME"
  - echo "Registry: $CI_REGISTRY"
  - echo "Image: $CI_REGISTRY_IMAGE"
```

**Полный список:** [GitLab docs: Predefined variables](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html).

**Полезные:**

| Переменная | Что содержит |
|:---|:---|
| `CI_COMMIT_SHA` | SHA коммита |
| `CI_COMMIT_REF_SLUG` | Имя ветки в slug-формате |
| `CI_COMMIT_TAG` | Git-тег (если есть) |
| `CI_PIPELINE_SOURCE` | Что триггернуло пайплайн (`push`, `merge_request_event`, `schedule`) |
| `CI_REGISTRY` | URL GitLab Registry |
| `CI_REGISTRY_IMAGE` | Полный путь к образу в GitLab Registry |
| `CI_REGISTRY_USER`, `CI_REGISTRY_PASSWORD` | Логин и пароль для GitLab Registry (автоматически) |
| `CI_JOB_TOKEN` | Токен для API GitLab (для взаимодействия с другими проектами) |

### 🎯 Внешние secrets managers

Для продакшена лучше использовать **специализированные** secrets managers:

**HashiCorp Vault:**

```yaml
deploy:
  image: vault:latest
  script:
    # Аутентификация через JWT
    - export VAULT_ADDR=https://vault.example.com
    - export VAULT_TOKEN=$(vault write -field=token auth/jwt/login role=myapp jwt=$CI_JOB_JWT)
    # Получить секрет
    - export DB_PASSWORD=$(vault kv get -field=password secret/db)
    - ./deploy.sh
```

**AWS Secrets Manager:**

```yaml
deploy:
  image: amazon/aws-cli:latest
  script:
    - export DB_PASSWORD=$(aws secretsmanager get-secret-value --secret-id prod/db/password --query SecretString --output text)
    - ./deploy.sh
```

**OIDC (без статических токенов):**

```yaml
deploy:
  image: amazon/aws-cli:latest
  id_tokens:
    AWS_TOKEN:
      aud: https://gitlab.example.com
  script:
    - export $(aws sts assume-role-with-web-identity
        --role-arn arn:aws:iam::123:role/gitlab-deploy
        --role-session-name gitlab-ci
        --web-identity-token $AWS_TOKEN
        --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]'
        --output text)
    - ./deploy.sh
```

OIDC — самый безопасный: **нет статических токенов**. CI доказывает своё identity через JWT, AWS выдаёт временные credentials.

### 🔒 Что нельзя делать с секретами

**❌ Нельзя:**

1. **Класть в `.gitlab-ci.yml`** — видно в Git.
2. **Логировать.** Даже с Masked, если значение слишком короткое или содержит спецсимволы.
3. **Класть в артефакты.** Доступно в UI GitLab.
4. **Коммитить в репозиторий** — даже в закрытый.
5. **Использовать в публичных форках.** Если fork публичный — секреты protected не передаются, но non-protected могут утечь.
6. **Передавать между джобами через cache.** Cache общий для ветки, может утечь.
7. **Хранить в Docker-образе.** Слои неизменяемы, секрет останется в слое навсегда (вспомни подглаву 2.5).

### 🧪 Практика: правильная работа с секретами

**Настройка GitLab Registry:**

```
# Settings → CI/CD → Variables:

CI_REGISTRY_USER     = myuser         (Variable)
CI_REGISTRY_PASSWORD = super-secret   (Variable, Masked, Protected)
```

**Настройка SSH для деплоя:**

```
# Settings → CI/CD → Variables:

SSH_PRIVATE_KEY   = <contents-of-ssh-key>  (File, Masked, Protected)
SSH_KNOWN_HOSTS   = <known_hosts-content>  (File, Protected)
DEPLOY_SERVER     = user@server.example.com (Variable, Protected)
```

**Использование в деплое:**

```yaml
deploy-prod:
  stage: deploy
  environment:
    name: production
    url: https://example.com
  before_script:
    - eval $(ssh-agent -s)
    - chmod 600 $SSH_PRIVATE_KEY
    - ssh-add $SSH_PRIVATE_KEY
    - mkdir -p ~/.ssh && cp $SSH_KNOWN_HOSTS ~/.ssh/known_hosts
  script:
    - ssh $DEPLOY_SERVER "cd /opt/myapp && docker compose pull && docker compose up -d"
  rules:
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
      when: manual
```

### 💡 Практика: как правильно работать с секретами

**✅ ОБЯЗАТЕЛЬНО:**

1. **Используй CI/CD Variables.** Не хардкодь.
2. **Masked для всех паролей и токенов.**
3. **Protected для секретов production.**
4. **File для SSH-ключей и сертификатов.**

**👍 СТОИТ:**

5. **Используй Vault или managed secrets** для production.
6. **Ротируй секреты** регулярно.
7. **Используй OIDC** для AWS/GCP/Azure — без статических токенов.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

8. **Свой Vault** — если нужен полный контроль.

**❌ НЕ ДЕЛАЙ:**

9. **Не логируй секреты.** Даже с Masked.
10. **Не храни секреты в `.gitlab-ci.yml`, в артефактах, в кэше.**
11. **Не используй один секрет для всех окружений.** Разные окружения — разные секреты.
12. **Не забывай про non-protected ветки.** Если переменная не protected, она доступна в MR из любой ветки.

### Где мы сейчас

Мы завершили Главу 6: CI/CD с GitLab CI. Разобрали:

- Что такое CI/CD.
- Структуру `.gitlab-ci.yml`.
- Сборку Docker-образов в CI (DinD, Kaniko).
- Приватный registry.
- Кэширование и ускорение пайплайна.
- Artifacts и dependencies.
- Rules и условия.
- Environments.
- Секреты.

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **CI (Continuous Integration)** | Непрерывная интеграция: каждый коммит автоматически собирается и тестируется. |
| **CD (Continuous Delivery)** | Непрерывная доставка: каждый успешный билд готов к деплою, но решение — за человеком. |
| **CD (Continuous Deployment)** | Непрерывный деплой: каждый успешный билд автоматически деплоится. |
| **Pipeline** | Запуск CI/CD для конкретного коммита. |
| **Stage** | Этап пайплайна (build, test, deploy). |
| **Job** | Конкретная задача внутри stage. |
| **Runner** | Агент, который выполняет джобы. |
| **Executor** | Способ запуска джоб (docker, shell, kubernetes). |
| **Artifact** | Файл, сохраняемый джобой для других джоб. |
| **Cache** | Файлы, кэшируемые между запусками пайплайна. |
| **DinD (Docker-in-Docker)** | Запуск Docker daemon внутри контейнера для сборки образов в CI. |
| **Kaniko** | Инструмент сборки Docker-образов без daemon. |
| **Buildah** | Альтернатива Kaniko от Red Hat. |
| **GitLab Container Registry** | Встроенный в GitLab registry для Docker-образов. |
| **Harbor** | Расширенный registry с UI, сканированием, retention. |
| **Rules** | Условия запуска джобы в GitLab CI. |
| **Environment** | Окружение (dev, staging, prod) в GitLab CI. |
| **Protected environment** | Окружение с ограниченным доступом для деплоя. |
| **Review App** | Временное окружение для Merge Request. |
| **CI/CD Variables** | Переменные, задаваемые в GitLab UI. |
| **Masked** | Скрытие значения переменной в логах. |
| **Protected** | Доступность переменной только для protected branches. |
| **OIDC** | OpenID Connect — аутентификация без статических токенов. |
| **DAG** | Directed Acyclic Graph — граф зависимостей через `needs`. |

---

## Что мы узнали?

- **CI/CD** автоматизирует путь от коммита до прода. CI — сборка и тесты на каждый коммит. CD — готовность к деплою или автоматический деплой.
- **GitLab CI** описывается в `.gitlab-ci.yml`. Состоит из stages, jobs, runners.
- **DinD** или **Kaniko** собирают Docker-образы в CI. Kaniko безопаснее (без privileged).
- **Приватный registry** — один контейнер `registry:2`. Для production — Harbor или managed.
- **Кэш** ускоряет пайплайн в 3-10 раз. Кэшируй зависимости и Docker-слои.
- **Artifacts** передают файлы между джобами. **Cache** — между запусками.
- **Rules** управляют условиями запуска джоб.
- **Environments** управляют окружениями: dev, staging, prod. Protected environments защищают prod.
- **Секреты** — только через CI/CD Variables. Masked + Protected + File.
- **OIDC** — лучший способ аутентификации в облаке без статических токенов.

---

## Типичные ошибки

- ❌ **Хранить секреты в `.gitlab-ci.yml`.** Они видны в Git.
- ❌ **Использовать `latest` в image.** Внезапно сломается.
- ❌ **Использовать `only`/`except`** вместо `rules`. Устарело.
- ❌ **Не кэшировать зависимости.** Пайплайн будет медленным.
- ❌ **Кэшировать всё подряд.** Кэш должен быть маленьким и актуальным.
- ❌ **Использовать `paths: [.]` в артефактах.** Займёт всё место.
- ❌ **Не ставить `expire_in`.** Артефакты будут храниться вечно.
- ❌ **Использовать privileged DinD без необходимости.** Используй Kaniko.
- ❌ **Деплой в prod без manual и protected.** Автоматический деплой в prod — риск.
- ❌ **Игнорировать red pipeline.** Красный пайплайн — stop-the-line.
- ❌ **Не использовать `needs` для DAG.** Stage-based pipeline медленнее.
- ❌ **Забывать про `interruptible: true`.** Старые пайплайны не отменяются.
- ❌ **Хранить state в CI-джобе.** Каждая джоба — новый контейнер.

---

## Для быстрого повторения

- **CI/CD:** CI — сборка и тесты на каждый коммит. CD — доставка/деплой.
- **`.gitlab-ci.yml`:** stages, jobs, `image`, `script`, `before_script`, `after_script`.
- **Runner:** docker executor. Каждая джоба — новый контейнер.
- **DinD:** `image: docker:24`, `services: docker:24-dind`. Для сборки образов.
- **Kaniko:** `gcr.io/kaniko-project/executor`. Безопаснее DinD.
- **Registry:** `registry:2` + htpasswd + TLS. GitLab Registry — встроенный.
- **Кэш:** `cache.paths`, `key`. Для go mod, npm, Docker-слоёв.
- **Artifacts:** `artifacts.paths`, `expire_in`. Передача файлов между джобами.
- **Dependencies:** `dependencies: []` — не скачивать артефакты.
- **Rules:** `if`, `when`, `changes`. Условия запуска.
- **Environments:** `environment.name`, `url`. Protected для prod.
- **Секреты:** CI/CD Variables. Masked + Protected + File.
- **OIDC:** аутентификация в облаке без статических токенов.
- **DAG:** `needs` — запуск джоб без ожидания всего stage.

---

## Вопросы для самопроверки

1. Что такое CI и CD? Чем Continuous Delivery отличается от Continuous Deployment?
2. Из каких этапов состоит типичный пайплайн? Почему порядок важен?
3. Что такое stage и job в GitLab CI? Как они связаны?
4. Что такое runner? Какой executor использовать для сборки Docker-образов?
5. Что такое DinD? Какие у него проблемы? Какую альтернативу использовать?
6. Как собрать Docker-образ в CI и запушить в registry?
7. Что такое приватный registry? Как запустить свой?
8. Почему Docker требует TLS для удалённого registry?
9. Что такое кэш в GitLab CI? Чем он отличается от artifacts?
10. Как ускорить пайплайн в 3-10 раз?
11. Что такое `needs` (DAG)? Чем он отличается от stages?
12. Что такое `rules`? Как ограничить деплой в prod только для тегов?
13. Что такое environment? Зачем нужны protected environments?
14. Как безопасно передать секреты в пайплайн? Почему нельзя в `.gitlab-ci.yml`?
15. Что такое Masked и Protected переменные?
16. Что такое OIDC? Зачем он нужен?

---

## Ответы

**1. CI vs CD**

CI (Continuous Integration) — каждый коммит автоматически собирается и тестируется. CD (Continuous Delivery) — каждый успешный билд готов к деплою, решение о деплое за человеком. Continuous Deployment — автоматический деплой в прод без подтверждения.

**2. Этапы пайплайна**

Build → test → package → deploy. Порядок: сначала убедиться, что код работает (build, test), потом упаковать (package), потом деплоить (deploy). Если build упал — нет смысла тестировать.

**3. Stage и job**

Stage — этап пайплайна (build, test, deploy). Job — задача внутри stage. Jobs одного stage выполняются параллельно. Stages выполняются последовательно.

**4. Runner и executor**

Runner — агент, выполняющий джобы. Для сборки Docker-образов — docker executor с DinD или Kaniko.

**5. DinD**

Docker-in-Docker — запуск Docker daemon внутри контейнера. Требует `--privileged` (небезопасно), теряет кэш между джобами. Альтернатива — Kaniko (без daemon, безопаснее, быстрее).

**6. Сборка образа в CI**

```yaml
build-image:
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

**7. Приватный registry**

`registry:2` — один контейнер. Запуск: `docker run -d -p 5000:5000 -v data:/var/lib/registry registry:2`. Для production — Harbor или managed (ECR, GCR).

**8. TLS для registry**

Docker по умолчанию пушит по HTTP только в `localhost:5000`. Для удалённого — обязателен HTTPS. Иначе: `x509: certificate signed by unknown authority`. Решение: Nginx с Let's Encrypt или self-signed сертификат в доверенных.

**9. Кэш vs artifacts**

Cache — между **разными запусками** пайплайна (для одной ветки). Artifacts — между **разными джобами** одного пайплайна. Cache для зависимостей (go mod, npm), artifacts для собранных артефактов.

**10. Ускорение пайплайна**

Кэш зависимостей, `--cache-from` для Docker, параллельные джобы в stage, DAG через `needs`, маленькие базовые образы, `interruptible: true`.

**11. `needs` (DAG)**

Позволяет джобе начать, не дожидаясь завершения всего stage. Если джоба зависит только от одной джобы, `needs: [job-name]` запустит её сразу после этой джобы. Быстрее, чем ждать весь stage.

**12. `rules`**

Условия запуска джобы. Для деплоя в prod только по тегам:
```yaml
rules:
  - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
    when: manual
```

**13. Environment**

Окружение (dev, staging, prod). GitLab показывает список, историю деплоев, позволяет откатывать. Protected environment — деплой только для определённых пользователей/веток.

**14. Секреты**

Только через CI/CD Variables (GitLab UI: Settings → CI/CD → Variables). Не в `.gitlab-ci.yml` — он в Git, все видят. Не в артефактах — видны в UI. Не в кэше — может утечь.

**15. Masked и Protected**

Masked — значение скрыто в логах как `[MASKED]`. Требования: 8+ символов, base64-совместимые. Protected — переменная доступна только в пайплайнах для protected branches/tags.

**16. OIDC**

OpenID Connect. CI доказывает свою identity через JWT-токен, облако (AWS, GCP, Azure) выдаёт временные credentials. Нет статических токенов — максимальная безопасность.

---

## Куда идти дальше?

Мы разобрали CI/CD с GitLab CI. Теперь ты умеешь:

- Писать `.gitlab-ci.yml` для Go-проекта.
- Собирать Docker-образы в CI (DinD и Kaniko).
- Запускать свой приватный registry.
- Кэшировать зависимости и Docker-слои.
- Передавать файлы через artifacts.
- Управлять окружениями через environments.
- Безопасно работать с секретами.

Но мы пока не разобрали **деплой в Kubernetes** из CI/CD. Как обновлять Pod'ы без простоя? Как делать канареечные деплои? Как откатывать?

Об этом — в следующей главе.

**Глава 7: CI/CD — деплой-стратегии и интеграция с K8s.** Погнали. 🚀