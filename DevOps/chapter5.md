# 🎼 Глава 5: Docker Compose — локальная оркестрация

**Что вы узнаете:**
- Зачем нужен Docker Compose, если есть `docker run`.
- Как описать всё приложение в одном YAML-файле.
- Как работают зависимости между сервисами и `healthcheck`.
- Как передавать конфигурацию через `.env`-файлы и переменные окружения.
- Как использовать profiles для разных окружений (dev, prod).
- Как отлаживать многоконтейнерное приложение.
- Как ограничивать ресурсы и масштабировать сервисы.

**После прочтения вы сможете:**
- Написать `docker-compose.yml` для приложения из 5 сервисов.
- Поднять всё окружение одной командой `docker compose up`.
- Настроить healthchecks и зависимости между сервисами.
- Передавать секреты через `.env`-файлы.
- Использовать `profiles` для dev/prod конфигураций.
- Отлаживать проблемы через `docker compose logs` и `docker compose exec`.

---

## Содержание

- [5.0 Пролог: пять команд вместо одной](#50-пролог-пять-команд-вместо-одной)
- [5.1 Что такое Docker Compose и зачем он нужен](#51-что-такое-docker-compose-и-зачем-он-нужен)
- [5.2 Структура `docker-compose.yml`](#52-структура-docker-composeyml)
- [5.3 Зависимости и healthchecks: когда один сервис ждёт другой](#53-зависимости-и-healthchecks-когда-один-сервис-ждёт-другой)
- [5.4 Environment: переменные окружения и `.env`-файлы](#54-environment-переменные-окружения-и-env-файлы)
- [5.5 Networks и volumes в Compose](#55-networks-и-volumes-в-compose)
- [5.6 Profiles: dev, prod и другие окружения](#56-profiles-dev-prod-и-другие-окружения)
- [5.7 Отладка и масштабирование](#57-отладка-и-масштабирование)
- [5.8 Лимиты ресурсов и deploy-секция](#58-лимиты-ресурсов-и-deploy-секция)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 5.0 Пролог: пять команд вместо одной

Ты — DevOps-инженер. Твой проект — Go-бэкенд с PostgreSQL, Redis, Kafka и Nginx в качестве reverse proxy. Пять сервисов.

Каждое утро ты запускаешь dev-окружение вручную:

```bash
# Создать сеть
docker network create myapp-net

# Запустить PostgreSQL
docker run -d --name postgres \
  --network myapp-net \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  postgres:16

# Запустить Redis
docker run -d --name redis \
  --network myapp-net \
  -p 6379:6379 \
  redis:7-alpine

# Запустить Kafka
docker run -d --name kafka \
  --network myapp-net \
  -p 9092:9092 \
  -e KAFKA_BROKER_ID=1 \
  -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
  confluentinc/cp-kafka:latest

# Подождать, пока БД поднимется (20 секунд)
sleep 20

# Запустить бэкенд
docker run -d --name app \
  --network myapp-net \
  -e DATABASE_URL=postgres://postgres:secret@postgres:5432/mydb \
  -e REDIS_URL=redis://redis:6379 \
  -e KAFKA_URL=kafka:9092 \
  -p 8080:8080 \
  myapp:latest

# Запустить nginx
docker run -d --name nginx \
  --network myapp-net \
  -p 80:80 \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx:alpine
```

Шесть команд. Каждый день. Плюс нужно помнить: какая сеть, какие порты, какие переменные. А если что-то пошло не так — надо убить контейнеры и начинать заново.

Хуже того, у коллеги другое окружение: у него другие версии образов, другие порты, другая конфигурация. «У меня работает» — а у тебя нет.

**Docker Compose решает эту проблему.** Все пять сервисов описываются в одном YAML-файле. Одна команда `docker compose up` поднимает всё. Одна команда `docker compose down` останавливает и удаляет. Файл хранится в Git — у всех одинаковое окружение.

В этой главе мы разберём Compose от основ до продвинутых техник. Ты научишься описывать целое приложение одним файлом.

---

## 5.1 Что такое Docker Compose и зачем он нужен

### 🔌 Проблема: `docker run` не масштабируется на несколько сервисов

`docker run` — отличная команда для запуска **одного** контейнера. Но когда у тебя 5-10 контейнеров, которые связаны между собой, ручное управление превращается в хаос:

- **Много команд.** Каждый контейнер — отдельная `docker run` с десятком флагов.
- **Нет декларативности.** Ты описываешь **что делать** (запустить контейнер), а не **что должно быть** (сервис с такими-то параметрами).
- **Нет воспроизводимости.** Коллега не может запустить то же окружение одной командой.
- **Нет управления зависимостями.** Бэкенд должен запуститься **после** базы данных. `docker run` этого не умеет.
- **Нет единого жизненного цикла.** Остановить, запустить, посмотреть логи — для каждого контейнера отдельно.

### 📦 Что такое Docker Compose

**Docker Compose** — это инструмент для определения и запуска многоконтейнерных приложений. Ты описываешь **все** сервисы в одном YAML-файле — `docker-compose.yml` — и управляешь ими одной командой.

**Основные понятия:**

| Понятие | Что означает |
|:---|:---|
| **Service** | Один контейнер (или группа одинаковых контейнеров). |
| **Network** | Сеть, в которой общаются сервисы. |
| **Volume** | Хранилище для персистентных данных. |
| **Project** | Группа сервисов, сетей, volumes. Имя проекта = имя директории. |
| **Compose file** | `docker-compose.yml` — описание всего проекта. |

**Основные команды:**

```bash
docker compose up         # Запустить всё
docker compose up -d      # Запустить в фоне
docker compose down       # Остановить и удалить
docker compose ps         # Статус сервисов
docker compose logs       # Логи всех сервисов
docker compose logs app   # Логи конкретного сервиса
docker compose exec app sh  # Зайти в контейнер
docker compose build      # Пересобрать образы
docker compose restart app  # Перезапустить сервис
```

### 📊 Сравнение: `docker run` vs `docker compose`

**С `docker run`:**

```bash
docker network create myapp-net
docker run -d --name postgres --network myapp-net -v postgres-data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=secret postgres:16
docker run -d --name redis --network myapp-net redis:7-alpine
docker run -d --name app --network myapp-net -e DATABASE_URL=postgres://postgres:secret@postgres:5432/mydb -p 8080:8080 myapp:latest
```

**С Docker Compose:**

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

  app:
    image: myapp:latest
    environment:
      DATABASE_URL: postgres://postgres:secret@postgres:5432/mydb
    ports:
      - "8080:8080"

volumes:
  postgres-data:
```

```bash
docker compose up -d
```

Одна команда вместо трёх. И YAML-файл хранится в Git — все разработчики получают одинаковое окружение.

### 🎯 Ключевые преимущества Compose

**1. Декларативность.**

Ты описываешь **желаемое состояние**: «есть сервис app, есть сервис postgres, они в одной сети». Compose сам создаёт контейнеры, сети, volumes и приводит систему к этому состоянию.

**2. Воспроизводимость.**

YAML-файл хранится в Git. Любой разработчик делает `git clone`, `docker compose up` — и получает то же окружение.

**3. Единый жизненный цикл.**

`docker compose up` — запускает всё. `docker compose down` — останавливает и удаляет всё. `docker compose logs` — логи всех сервисов в одном потоке.

**4. Управление зависимостями.**

Compose умеет ждать, пока один сервис будет готов, перед запуском другого (через `depends_on` + `healthcheck`).

**5. Изоляция по проектам.**

Каждый проект имеет своё имя (по имени директории). Сети, volumes, контейнеры — изолированы. Можно запустить два проекта с одинаковыми сервисами без конфликтов.

### 🔬 Установка Docker Compose

Docker Compose v2 встроен в Docker Desktop и в современный Docker CLI как плагин. Проверить:

```bash
docker compose version
# Docker Compose version v2.24.0
```

**Обрати внимание на синтаксис:** современный Compose вызывается как `docker compose` (с пробелом). Старый (v1) — `docker-compose` (с дефисом). Мы используем v2.

### Где мы сейчас

Мы разобрали, что такое Docker Compose и зачем он нужен. Теперь посмотрим на **структуру YAML-файла** — из чего он состоит.

---

## 5.2 Структура `docker-compose.yml`

### 🔌 Проблема: как описать целое приложение в одном файле

`docker-compose.yml` — это YAML-файл со строгой структурой. Верхний уровень содержит несколько секций:

```yaml
# Версия формата (устарела в v2, но всё ещё указывается)
# В Docker Compose v2 не обязательна
version: "3.9"

# Сервисы (обязательно)
services:
  # ...

# Сети (опционально)
networks:
  # ...

# Volumes (опционально)
volumes:
  # ...

# Секреты (опционально)
secrets:
  # ...

# Configs (опционально)
configs:
  # ...
```

Разберём каждую секцию.

### 📦 Секция `services`

**Обязательная секция.** Здесь описываются все контейнеры.

```yaml
services:
  app:                              # имя сервиса
    image: myapp:latest             # из какого образа
    container_name: myapp           # имя контейнера (опционально)
    ports:
      - "8080:8080"                 # проброс портов
    environment:                    # переменные окружения
      DATABASE_URL: postgres://postgres:secret@postgres:5432/mydb
    volumes:
      - ./config:/app/config:ro     # монтирование
    depends_on:
      - postgres                    # зависимости
    networks:
      - myapp-net
    restart: unless-stopped
```

**Ключевые директивы сервиса:**

| Директива | Что делает |
|:---|:---|
| `image` | Из какого образа запускать |
| `build` | Собрать образ из Dockerfile |
| `container_name` | Имя контейнера (по умолчанию — `<project>_<service>_1`) |
| `ports` | Проброс портов (как `-p` в `docker run`) |
| `environment` | Переменные окружения |
| `env_file` | Файл с переменными |
| `volumes` | Монтирования |
| `networks` | К каким сетям подключён |
| `depends_on` | Зависимости |
| `restart` | Политика перезапуска |
| `command` | Команда запуска (переопределяет CMD) |
| `entrypoint` | ENTRYPOINT (переопределяет Dockerfile) |
| `healthcheck` | Проверка здоровья |
| `deploy` | Лимиты ресурсов, реплики |
| `profiles` | К каким профилям относится сервис |

**Пример с `build`:**

```yaml
services:
  app:
    build:
      context: .                    # директория сборки
      dockerfile: Dockerfile        # имя Dockerfile
      args:                         # ARG-переменные
        GO_VERSION: "1.22"
    # или коротко:
    # build: .
```

### 🌐 Секция `networks`

**Опциональная.** Если не указана, Compose создаёт дефолтную сеть для проекта.

```yaml
networks:
  myapp-net:
    driver: bridge
  myapp-external:
    external: true                  # использовать существующую сеть
```

**По умолчанию** Compose создаёт одну сеть `<project>_default`. Все сервисы подключаются к ней. DNS работает: сервисы видят друг друга по именам.

**Явное описание сетей нужно:**

- Когда несколько сетей (например, `frontend` и `backend`).
- Когда нужны кастомные драйверы (overlay).
- Когда нужна внешняя сеть.

### 💾 Секция `volumes`

**Опциональная.** Volumes, используемые сервисами.

```yaml
volumes:
  postgres-data:
  redis-data:
    driver: local
```

**Обрати внимание:** если volume указан в `volumes:` сервиса как именованный (без `/` в начале), его нужно объявить здесь.

```yaml
services:
  postgres:
    volumes:
      - postgres-data:/var/lib/postgresql/data    # именованный
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql   # bind mount
      - /tmp/cache:/cache                          # bind mount (абсолютный путь)

volumes:
  postgres-data:    # объявление именованного volume
```

### 🔒 Секции `secrets` и `configs`

**Для production.** Позволяют передавать секреты и конфиги без монтирования файлов.

```yaml
services:
  app:
    secrets:
      - db_password
    configs:
      - source: app_config
        target: /etc/app/config.yaml

secrets:
  db_password:
    file: ./secrets/db_password.txt

configs:
  app_config:
    file: ./config/app.yaml
```

**Разница:**

- **secrets** — монтируются в `/run/secrets/<name>` в памяти (tmpfs), не попадают на диск.
- **configs** — монтируются как обычные файлы, но управляются Compose.

В Docker Swarm это работает «из коробки». В обычном Compose секреты монтируются как bind mount — с шифрованием только в Swarm.

### 📝 Полный пример

Соберём всё вместе — Compose для Go-приложения с PostgreSQL, Redis и Nginx:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  app:
    build: .
    environment:
      DATABASE_URL: postgres://postgres:secret@postgres:5432/mydb
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend
      - frontend
    restart: unless-stopped
    ports:
      - "8080:8080"

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app
    networks:
      - frontend
    restart: unless-stopped

networks:
  backend:
  frontend:

volumes:
  postgres-data:
```

**Что здесь есть:**

- Четыре сервиса: postgres, redis, app, nginx.
- Две сети: backend (postgres, redis, app) и frontend (app, nginx). Nginx не видит базу — это изоляция.
- Один volume: postgres-data.
- Healthchecks для postgres и redis.
- Зависимости: app ждёт, пока postgres и redis станут healthy.

### 💡 Практика: как правильно писать Compose-файл

**✅ ОБЯЗАТЕЛЬНО:**

1. **Используй версионирование.** Храни `docker-compose.yml` в Git вместе с кодом.
2. **Указывай конкретные теги образов** (`postgres:16-alpine`, не `postgres:latest`).
3. **Описывай healthchecks** для сервисов с состоянием (БД, кэш, брокер).

**👍 СТОИТ:**

4. **Разделяй сети** — frontend и backend — для изоляции.
5. **Используй `depends_on` с `condition: service_healthy`** вместо `sleep`.
6. **Используй `restart: unless-stopped`** для долгоживущих сервисов.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **`container_name`** — если не нужен фиксированный имя, оставь Compose генерировать (лучше для избежания конфликтов).

**❌ НЕ ДЕЛАЙ:**

8. **Не используй `version: "3"` в Compose v2.** Секция `version` устарела, можно не указывать.
9. **Не пиши секреты в Compose-файл.** Используй `.env` (подглава 5.4) или secrets.
10. **Не используй `latest`** — непредсказуемо.

### Где мы сейчас

Мы разобрали структуру `docker-compose.yml`: services, networks, volumes. Теперь посмотрим на **зависимости и healthchecks** — как заставить один сервис ждать другой.

---

## 5.3 Зависимости и healthchecks: когда один сервис ждёт другой

### 🔌 Проблема: app стартует быстрее базы

Ты запускаешь Compose:

```bash
docker compose up
```

Через 3 секунды app падает с ошибкой:

```
dial tcp postgres:5432: connect: connection refused
```

**Что произошло:** app запустился **раньше**, чем PostgreSQL был готов принимать соединения. PostgreSQL стартует 5-10 секунд, а app — за 1 секунду. За это время app успел попытаться подключиться и упал.

Compose по умолчанию запускает сервисы **параллельно**. Порядок в YAML не влияет.

### 📊 `depends_on`: управление порядком запуска

`depends_on` говорит Compose: «запускай этот сервис **после** другого».

```yaml
services:
  app:
    depends_on:
      - postgres
      - redis
  
  postgres:
    image: postgres:16
  
  redis:
    image: redis:7
```

**Что делает `depends_on`:**

- Compose запустит postgres и redis **до** app.
- Но **только запустит** (создаст контейнеры и запустит процесс), не дождётся готовности.

**Что `depends_on` НЕ делает:**

- Не ждёт, пока postgres будет готов принимать соединения.
- Не проверяет, что процесс успешно стартовал.

Это значит, что **простой `depends_on` не решает проблему**.

### 🏥 `healthcheck`: проверка готовности

**Healthcheck** — это команда, которую Docker периодически запускает внутри контейнера, чтобы проверить, «здоров» ли сервис.

```yaml
services:
  postgres:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s        # проверять каждые 5 секунд
      timeout: 3s         # ждать ответа не дольше 3 секунд
      retries: 5          # сколько неудач подряд считать «нездоровым»
      start_period: 10s   # дать сервису 10 секунд на старт
```

**Как работает:**

1. Docker запускает контейнер.
2. Каждые 5 секунд выполняет `pg_isready -U postgres` внутри контейнера.
3. Если команда вернула 0 (успех) — контейнер `healthy`.
4. Если 5 раз подряд ошибка — контейнер `unhealthy`.
5. `start_period` даёт сервису время на старт, в течение которого неудачи не считаются.

**Статусы контейнера:**

| Статус | Что означает |
|:---|:---|
| `starting` | Только запустился, идёт start_period |
| `healthy` | Проверка проходит |
| `unhealthy` | Проверка не проходит больше `retries` раз |

Посмотреть статус:

```bash
docker ps
# STATUS: Up 30 seconds (healthy)

docker inspect postgres | jq '.[0].State.Health'
```

### 🔗 `depends_on` + `healthcheck`: правильная комбинация

Чтобы app ждал **готовности** postgres, а не просто его запуска, нужно объединить `depends_on` с `condition`:

```yaml
services:
  app:
    depends_on:
      postgres:
        condition: service_healthy     # ждать healthy
      redis:
        condition: service_healthy
  
  postgres:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
  
  redis:
    image: redis:7
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

**Возможные условия:**

| Условие | Что означает |
|:---|:---|
| `service_started` | Запустить после старта сервиса (по умолчанию) |
| `service_healthy` | Запустить после того, как сервис станет healthy |
| `service_completed_successfully` | Запустить после успешного завершения сервиса (для миграций, init-скриптов) |

### 🧪 Практика: правильные healthchecks

**Для PostgreSQL:**

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
  interval: 5s
  timeout: 3s
  retries: 5
  start_period: 10s
```

**Для MySQL:**

```yaml
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  interval: 5s
  timeout: 3s
  retries: 5
  start_period: 30s
```

**Для Redis:**

```yaml
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 5s
  timeout: 3s
  retries: 5
```

**Для HTTP-сервисов:**

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
  interval: 10s
  timeout: 3s
  retries: 3
  start_period: 20s
```

**Важно:** `curl` должен быть в образе. Если его нет (например, distroless) — используй wget или напиши простую проверку на Go.

**Для Kafka:**

```yaml
healthcheck:
  test: ["CMD-SHELL", "kafka-broker-api-versions --bootstrap-server localhost:9092"]
  interval: 10s
  timeout: 10s
  retries: 10
  start_period: 30s
```

### 📊 Пример: init-скрипт с `service_completed_successfully`

Иногда нужен сервис, который **выполнит задачу и завершится** — например, миграция базы данных. App должен ждать его успешного завершения.

```yaml
services:
  migrate:
    image: myapp:latest
    command: ["./myapp", "migrate", "up"]
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - backend
    # Этот сервис выполнится и завершится

  app:
    image: myapp:latest
    depends_on:
      migrate:
        condition: service_completed_successfully    # ждать успешного завершения
      postgres:
        condition: service_healthy
    networks:
      - backend
    ports:
      - "8080:8080"
```

**Что произойдёт:**

1. Compose запустит postgres.
2. Дождётся, пока postgres станет healthy.
3. Запустит migrate — сервис выполнит миграции и завершится.
4. Дождётся успешного завершения migrate.
5. Запустит app.

### 💡 Практика: как правильно работать с зависимостями

**✅ ОБЯЗАТЕЛЬНО:**

1. **Всегда используй `healthcheck` + `condition: service_healthy`** для БД и брокеров:
   ```yaml
   depends_on:
     postgres:
       condition: service_healthy
   ```

2. **Для миграций используй `service_completed_successfully`:**
   ```yaml
   depends_on:
     migrate:
       condition: service_completed_successfully
   ```

3. **Ставь `start_period` для медленных сервисов** (БД с инициализацией):
   ```yaml
   start_period: 30s
   ```

**👍 СТОИТ:**

4. **Пиши healthcheck для всех сервисов с состоянием.** Это помогает Compose понимать, что происходит.

5. **Используй `/health` эндпоинт в своих приложениях:**
   ```go
   http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
       // Проверить БД, Redis и т. д.
       w.WriteHeader(http.StatusOK)
   })
   ```

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Внешние инструменты типа `wait-for-it`** — если не хочешь писать healthcheck.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `sleep` в command:** `command: sh -c "sleep 20 && ./myapp"`. Это костыль. Используй healthcheck.

8. **Не используй только `depends_on` без condition.** App запустится, но база может быть ещё не готова.

9. **Не пиши healthcheck, который всегда возвращает 0.** Это бессмысленно.

### Где мы сейчас

Мы разобрали зависимости и healthchecks. Теперь посмотрим, как передавать **переменные окружения** через `.env`-файлы.

---

## 5.4 Environment: переменные окружения и `.env`-файлы

### 🔌 Проблема: секреты в YAML-файле

Ты написал Compose-файл и закоммитил его в Git. Но в нём есть пароль от базы:

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: super-secret-password-123
```

Теперь **пароль в Git**. Его видят все, кто имеет доступ к репозиторию. Даже если удалить позже — пароль остаётся в истории коммитов.

**Решение:** вынести переменные в отдельный файл, который **не коммитится** в Git.

### 📝 `.env`-файл

**`.env`** — это файл с переменными в формате `KEY=value`. Compose автоматически читает его из корня проекта.

```bash
# .env
POSTGRES_PASSWORD=super-secret-password-123
POSTGRES_DB=mydb
POSTGRES_USER=postgres
APP_PORT=8080
DATABASE_URL=postgres://postgres:super-secret-password-123@postgres:5432/mydb
```

**Как использовать в Compose:**

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
  
  app:
    image: myapp:latest
    environment:
      DATABASE_URL: ${DATABASE_URL}
    ports:
      - "${APP_PORT}:8080"
```

**Синтаксис подстановки:**

| Синтаксис | Что означает |
|:---|:---|
| `${VAR}` | Подставить значение переменной. Ошибка, если не задана |
| `${VAR:-default}` | Подставить значение или default, если не задана |
| `${VAR-default}` | То же, но если задана пустая строка — использовать её |
| `${VAR:?error}` | Ошибка с сообщением, если не задана |
| `$$` | Экранирование `$` (литеральный символ) |

**Пример:**

```yaml
services:
  app:
    environment:
      APP_PORT: ${APP_PORT:-8080}       # по умолчанию 8080
      LOG_LEVEL: ${LOG_LEVEL:-info}     # по умолчанию info
      API_KEY: ${API_KEY:?API key required}  # ошибка, если не задана
```

### 📄 Несколько `.env`-файлов

Можно использовать разные `.env`-файлы для разных окружений:

```bash
# .env.dev
DATABASE_URL=postgres://localhost:5432/mydb_dev
LOG_LEVEL=debug

# .env.prod
DATABASE_URL=postgres://prod-db:5432/mydb
LOG_LEVEL=info
```

**Явно указать файл:**

```bash
docker compose --env-file .env.prod up
```

**Или указать в Compose-файле:**

```yaml
services:
  app:
    env_file:
      - .env.common
      - .env.${ENVIRONMENT:-dev}
```

### 🔒 `.gitignore`: не коммить `.env`

```gitignore
# .gitignore
.env
.env.local
.env.*.local
```

**Вместо `.env` в Git коммить `.env.example`** — шаблон без реальных значений:

```bash
# .env.example
POSTGRES_PASSWORD=changeme
POSTGRES_DB=mydb
APP_PORT=8080
```

Новый разработчик делает:

```bash
cp .env.example .env
# Редактирует .env с реальными значениями
```

### 🎯 Приоритет источников переменных

Переменные могут поступать из разных мест. Порядок приоритета (от высшего к низшему):

1. **Переменные окружения хоста** (`export VAR=value`).
2. **Файл, указанный в `--env-file`.**
3. **Файл `.env` в директории проекта.**
4. **Значение по умолчанию в `${VAR:-default}`.**

```bash
# На хосте
export POSTGRES_PASSWORD=from_host
docker compose up
# Используется from_host, а не из .env
```

### 🧪 Практика: полный пример

**`.env.example`:**

```bash
# Database
POSTGRES_USER=postgres
POSTGRES_PASSWORD=changeme
POSTGRES_DB=mydb
POSTGRES_PORT=5432

# Redis
REDIS_PORT=6379

# Application
APP_PORT=8080
LOG_LEVEL=info
```

**`.env` (в .gitignore):**

```bash
POSTGRES_USER=postgres
POSTGRES_PASSWORD=super-secret-123
POSTGRES_DB=mydb
POSTGRES_PORT=5432

REDIS_PORT=6379

APP_PORT=8080
LOG_LEVEL=debug
```

**`docker-compose.yml`:**

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "${POSTGRES_PORT}:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "${REDIS_PORT}:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  app:
    build: .
    environment:
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      REDIS_URL: redis://redis:6379
      LOG_LEVEL: ${LOG_LEVEL}
    ports:
      - "${APP_PORT}:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

volumes:
  postgres-data:
```

**Запуск:**

```bash
cp .env.example .env
# Отредактировать .env
docker compose up -d
```

### 💡 Практика: как правильно работать с переменными

**✅ ОБЯЗАТЕЛЬНО:**

1. **Всегда используй `.env` для секретов.** Никаких паролей в `docker-compose.yml`.
2. **Добавь `.env` в `.gitignore`.**
3. **Коммить `.env.example`** — шаблон для других разработчиков.
4. **Используй `${VAR:?error}` для обязательных переменных:**
   ```yaml
   environment:
     API_KEY: ${API_KEY:?API key is required}
   ```

**👍 СТОИТ:**

5. **Используй разные `.env` для dev/staging/prod:**
   ```bash
   docker compose --env-file .env.prod up
   ```

6. **Используй значения по умолчанию для необязательных:**
   ```yaml
   LOG_LEVEL: ${LOG_LEVEL:-info}
   ```

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **Secrets в Compose.** В Swarm они работают хорошо, в обычном Compose — просто bind mounts с маскировкой.

**❌ НЕ ДЕЛАЙ:**

8. **Не коммить `.env` с реальными секретами.** Даже в приватный репозиторий.
9. **Не используй переменные хоста как источник секретов в CI/CD.** Используй secrets-менеджер CI.
10. **Не хардкодь пути к `.env` в Compose-файле.** Пусть Compose сам находит `.env`.

### Где мы сейчас

Мы разобрали environment и `.env`-файлы. Теперь посмотрим, как **сети и volumes** работают в Compose.

---

## 5.5 Networks и volumes в Compose

### 🔌 Проблема: как организовать сеть между сервисами

В подглаве 4.1 мы разобрали сети в Docker. В Compose сети работают так же, но с автоматикой:

- По умолчанию Compose создаёт **одну сеть** для проекта — `<project>_default`.
- Все сервисы подключаются к ней.
- DNS работает: сервисы видят друг друга по именам.

Но иногда нужно больше контроля: разделить frontend и backend, подключить внешние сервисы, изолировать чувствительные сервисы.

### 🌐 Networks в Compose

**Простой случай — все в одной сети:**

```yaml
services:
  app:
    image: myapp
  postgres:
    image: postgres:16
# Compose сам создаст сеть <project>_default
```

**Разделение на frontend и backend:**

```yaml
services:
  nginx:
    image: nginx:alpine
    networks:
      - frontend
  
  app:
    image: myapp
    networks:
      - frontend
      - backend
  
  postgres:
    image: postgres:16
    networks:
      - backend
  
  redis:
    image: redis:7
    networks:
      - backend

networks:
  frontend:
  backend:
```

**Что получилось:**

- `nginx` видит только `app` (в frontend).
- `app` видит `nginx`, `postgres`, `redis` (в обеих сетях).
- `postgres` и `redis` видят только `app` (в backend).
- `postgres` **не видит** `nginx` напрямую.

**Зачем это:**

- **Изоляция.** Если злоумышленник взломает nginx, он не сможет подключиться к базе напрямую.
- **Ясность.** Ты видишь, кто с кем общается.
- **Безопасность.** Меньше поверхности для атаки.

**Подключение к внешней сети:**

```yaml
services:
  app:
    networks:
      - shared-network

networks:
  shared-network:
    external: true    # сеть уже существует, создана вне Compose
```

Создать внешнюю сеть:

```bash
docker network create shared-network
```

**Использование для связи с другими Compose-проектами:**

```yaml
# Проект A
networks:
  shared:
    name: shared-network

# Проект B
networks:
  shared:
    external: true
    name: shared-network
```

**Явное задание имени сети:**

```yaml
networks:
  myapp-net:
    name: myapp-production    # вместо <project>_myapp-net
```

**Кастомные драйверы:**

```yaml
networks:
  overlay-net:
    driver: overlay
    attachable: true
```

### 💾 Volumes в Compose

**Именованные volumes:**

```yaml
services:
  postgres:
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

**Короткий синтаксис vs длинный:**

```yaml
# Короткий
services:
  app:
    volumes:
      - ./config:/app/config:ro         # bind mount, read-only
      - postgres-data:/data              # именованный volume
      - /tmp:/tmp                        # анонимный volume

# Длинный (больше контроля)
services:
  app:
    volumes:
      - type: bind
        source: ./config
        target: /app/config
        read_only: true
      - type: volume
        source: postgres-data
        target: /data
        volume:
          nocopy: true                   # не копировать содержимое образа
```

**Типы mount:**

| Тип | Что означает |
|:---|:---|
| `volume` | Именованный volume (управляется Docker) |
| `bind` | Директория хоста |
| `tmpfs` | В памяти |

**Пример tmpfs:**

```yaml
services:
  app:
    tmpfs:
      - /app/cache
      - /tmp:size=100m
```

**Опции volumes:**

```yaml
volumes:
  postgres-data:
    driver: local
    driver_opts:
      type: none
      device: /mnt/fast-ssd/postgres
      o: bind
```

Это позволяет хранить volume в конкретной директории на хосте (полезно для высокопроизводительных SSD).

### 🔒 Изоляция по проектам

Каждый Compose-проект имеет своё имя (по умолчанию — имя директории). Все ресурсы получают префикс:

```
<project>_<service>_1       # контейнер
<project>_<network>          # сеть
<project>_<volume>           # volume
```

**Плюсы:**

- Можно запустить два проекта с одинаковыми сервисами без конфликтов.
- Легко удалить всё: `docker compose down -v` (удалит и volumes).

**Задать имя проекта:**

```bash
docker compose -p myproject up
```

Или через `.env`:

```bash
# .env
COMPOSE_PROJECT_NAME=myproject
```

### 🧪 Практика: типичный Compose для микросервисов

```yaml
services:
  # === Frontend ===
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    networks:
      - frontend
    depends_on:
      - api
  
  # === API ===
  api:
    build:
      context: ./api
    environment:
      DATABASE_URL: postgres://${DB_USER}:${DB_PASSWORD}@postgres:5432/${DB_NAME}
      REDIS_URL: redis://redis:6379
      KAFKA_URL: kafka:9092
    networks:
      - frontend
      - backend
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
  
  # === Data layer ===
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped
  
  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped
  
  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    networks:
      - backend
    depends_on:
      - zookeeper
    restart: unless-stopped
  
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - backend
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
    restart: unless-stopped

networks:
  frontend:
  backend:

volumes:
  postgres-data:
  redis-data:
  zookeeper-data:
```

### 💡 Практика: как правильно организовать сети и volumes

**✅ ОБЯЗАТЕЛЬНО:**

1. **Разделяй frontend и backend сети.** Изоляция + ясность.
2. **Используй именованные volumes для БД.** Не bind mounts.
3. **Задавай `name:` для сетей и volumes**, если они должны быть предсказуемыми:
   ```yaml
   networks:
     shared:
       name: shared-network
   ```

**👍 СТОИТ:**

4. **Используй `external: true` для сетей, созданных вне Compose.**
5. **Описывай volumes с опциями** для нестандартных хранилищ.
6. **Используй `tmpfs` для кэша и временных файлов.**

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **Разные сети для каждого сервиса** — избыточно. Двух-трёх сетей обычно достаточно.

**❌ НЕ ДЕЛАЙ:**

8. **Не подключай базу к frontend-сети.** База не должна быть доступна из интернета.
9. **Не используй `network_mode: host`** в Compose без причины. Теряешь изоляцию.
10. **Не используй bind mounts для продакшн-БД.** Volumes надёжнее.

### Где мы сейчас

Мы разобрали networks и volumes в Compose. Теперь посмотрим на **profiles** — как делать разные конфигурации для dev и prod.

---

## 5.6 Profiles: dev, prod и другие окружения

### 🔌 Проблема: разные сервисы для dev и prod

В dev-окружении тебе нужны:

- **Adminer** — веб-интерфейс для БД.
- **Mailhog** — для тестирования email.
- **Hot reload** — перезагрузка при изменении кода.

В production эти сервисы не нужны. Но и удалять их из Compose-файла не хочется — удобно иметь один файл.

**Решение:** **profiles**.

### 📊 Что такое profiles

**Profiles** — это способ группировать сервисы и запускать только нужные.

Сервис может принадлежать к одному или нескольким профилям. При запуске Compose запускает:

- **Все сервисы без профиля** — всегда.
- **Сервисы с профилем** — только если профиль активирован.

### 🎯 Пример: dev и prod

```yaml
services:
  # === Всегда ===
  app:
    build: .
    environment:
      DATABASE_URL: postgres://postgres:secret@postgres:5432/mydb
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
  
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
  
  # === Только в dev ===
  adminer:
    image: adminer:latest
    ports:
      - "8081:8080"
    profiles:
      - dev
  
  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "8025:8025"
    profiles:
      - dev

volumes:
  postgres-data:
```

**Запуск:**

```bash
# Только app и postgres
docker compose up

# app, postgres, adminer, mailhog
docker compose --profile dev up

# Все профили
docker compose --profile "*" up
```

### 📋 Возможные сценарии

**1. Сервис в нескольких профилях:**

```yaml
services:
  monitoring:
    image: grafana/grafana:latest
    profiles:
      - dev
      - monitoring
```

Запустится, если активирован `dev` **или** `monitoring`.

**2. Явно запустить сервис без профиля:**

```bash
# Запустить все сервисы, включая adminer, даже если у него профиль dev
docker compose up adminer
```

**3. `COMPOSE_PROFILES` в `.env`:**

```bash
# .env
COMPOSE_PROFILES=dev,monitoring
```

Теперь `docker compose up` автоматически активирует эти профили.

### 🧪 Полный пример: dev/prod/staging

```yaml
services:
  app:
    build: .
    environment:
      DATABASE_URL: ${DATABASE_URL}
    restart: unless-stopped
  
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s
  
  # === Dev ===
  adminer:
    image: adminer:latest
    ports:
      - "8081:8080"
    profiles:
      - dev
  
  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "1025:1025"     # SMTP
      - "8025:8025"     # Web UI
    profiles:
      - dev
  
  # === Monitoring ===
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    profiles:
      - monitoring
  
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    profiles:
      - monitoring

volumes:
  postgres-data:
  prometheus-data:
  grafana-data:
```

**Запуск:**

```bash
# Production — только app и postgres
docker compose up -d

# Development — добавить adminer и mailhog
docker compose --profile dev up -d

# Production + monitoring
docker compose --profile monitoring up -d

# Всё
docker compose --profile "*" up -d
```

### 💡 Практика: как правильно использовать profiles

**✅ ОБЯЗАТЕЛЬНО:**

1. **Используй профили для отладочных сервисов** (adminer, mailhog, debug tools).
2. **Не оставляй отладочные сервисы в production.** Profiles решают это.

**👍 СТОИТ:**

3. **Используй осмысленные имена профилей:** `dev`, `monitoring`, `backup`, `test`.
4. **Комбинируй с `.env`** для автоматической активации:
   ```bash
   # .env.dev
   COMPOSE_PROFILES=dev,monitoring
   ```

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **Много профилей** — усложняет понимание. Двух-трёх обычно хватает.

**❌ НЕ ДЕЛАЙ:**

6. **Не давай профилям имена, совпадающие с именами сервисов.** Это запутает.
7. **Не используй профили для базовых сервисов** (app, db). Они должны запускаться всегда.

### Где мы сейчас

Мы разобрали profiles. Теперь посмотрим на **отладку и масштабирование** — как найти проблемы в многоконтейнерном приложении.

---

## 5.7 Отладка и масштабирование

### 🔌 Проблема: что-то не работает, но что именно

В Compose-приложении 5-10 сервисов. Один из них упал. Или работает не так, как ожидалось. Как найти проблему?

### 🔍 Просмотр логов

```bash
# Логи всех сервисов
docker compose logs

# Логи в реальном времени
docker compose logs -f

# Логи конкретного сервиса
docker compose logs app

# Последние 100 строк
docker compose logs --tail 100 app

# С таймстампами
docker compose logs -t app

# С начала (не только текущие)
docker compose logs --since 1h app
```

**Важно:** по умолчанию Compose показывает логи **всех** сервисов в одном потоке, с префиксами:

```
postgres_1  | 2026-01-15 10:00:00.123 UTC [1] LOG: database system is ready to accept connections
app_1       | 2026-01-15 10:00:05.456 INFO Connected to database
redis_1     | 1:M 15 Jan 2026 10:00:00.789 * Ready to accept connections
```

Префиксы — имена сервисов. Легко различить, кто что пишет.

### 🔍 Статус сервисов

```bash
# Список всех сервисов проекта
docker compose ps
# NAME              IMAGE            COMMAND   SERVICE   CREATED          STATUS                    PORTS
# myapp-app-1       myapp:latest     ...       app       5 minutes ago    Up 5 minutes (healthy)   0.0.0.0:8080->8080/tcp
# myapp-postgres-1  postgres:16      ...       postgres  5 minutes ago    Up 5 minutes (healthy)   0.0.0.0:5432->5432/tcp

# Все сервисы, включая остановленные
docker compose ps -a

# Только ID
docker compose ps -q
```

### 🔍 Зайти в контейнер

```bash
# Зайти в контейнер сервиса app
docker compose exec app sh

# Выполнить одну команду
docker compose exec app ./myapp migrate status

# Внутри контейнера — стандартные команды
docker compose exec app ls -la /app
```

**Разница между `exec` и `run`:**

- **`exec`** — выполняет команду в **работающем** контейнере.
- **`run`** — создаёт **новый** контейнер из сервиса и выполняет команду в нём, потом удаляет.

```bash
# exec — в работающем
docker compose exec app sh

# run — новый контейнер (полезно для одноразовых задач)
docker compose run --rm app ./myapp migrate up
docker compose run --rm app sh
```

### 🔍 Перезапуск и пересборка

```bash
# Перезапустить один сервис
docker compose restart app

# Перезапустить все
docker compose restart

# Пересобрать образ сервиса и перезапустить
docker compose up -d --build app

# Пересобрать все и перезапустить
docker compose up -d --build

# Остановить всё (контейнеры удаляются, volumes остаются)
docker compose down

# Остановить всё и удалить volumes (ОСТОРОЖНО: данные удалятся!)
docker compose down -v

# Остановить всё и удалить образы
docker compose down --rmi all
```

### 🔍 Просмотр конфигурации

```bash
# Показать итоговую конфигурацию после подстановки переменных
docker compose config

# Показать только сервисы
docker compose config --services

# Показать только volumes
docker compose config --volumes
```

Очень полезно для отладки: видишь, какие переменные подставились.

### 📈 Масштабирование

**Масштабирование** — запуск нескольких экземпляров одного сервиса.

```bash
# Запустить 3 экземпляра app
docker compose up -d --scale app=3
```

**Что произойдёт:**

- Compose запустит 3 контейнера `app`.
- Они будут в одной сети.
- DNS `app` будет резолвиться в один из них (round-robin).

**Ограничения:**

- **`container_name`** нельзя использовать — имена должны быть уникальны.
- **`ports`** нельзя использовать с фиксированным портом — будет конфликт. Используй диапазон:
  ```yaml
  ports:
    - "8080-8082:8080"
  ```

**Что НЕ работает в Compose:**

- Load balancing. Compose не балансирует трафик между экземплярами автоматически (кроме простого DNS round-robin).
- Scaling с зависимостями. `depends_on` не масштабируется вместе с сервисом.

**Когда использовать:**

- **Разработка** — протестировать, как приложение работает в нескольких экземплярах.
- **Локальные тесты** — проверить, что нет состояния на инстанс.

**Когда НЕ использовать:**

- **Production** — для этого нужен Kubernetes (Глава 7+).

### 🧪 Практика: типичные проблемы

**Проблема 1: сервис не стартует**

```bash
docker compose ps
# app  Exit 1

docker compose logs app
# Error: cannot connect to database
```

Проверь:

- Правильные ли переменные (`docker compose config`).
- Ждёт ли app готовности базы (`depends_on` + `condition`).
- Правильный ли DATABASE_URL.

**Проблема 2: порт занят**

```bash
docker compose up
# Error: bind: address already in use
```

Проверь:

```bash
lsof -i :8080
# или
ss -tlnp | grep 8080
```

Или используй другой порт: `APP_PORT=8081 docker compose up`.

**Проблема 3: сервис unhealthy**

```bash
docker compose ps
# postgres  Up (unhealthy)
```

Проверь healthcheck:

```bash
docker compose exec postgres pg_isready -U postgres
# или
docker inspect myapp-postgres-1 | jq '.[0].State.Health'
```

**Проблема 4: изменения в коде не применяются**

Изменения в коде → нужно пересобрать:

```bash
docker compose up -d --build app
```

Или для разработки — используй bind mount:

```yaml
services:
  app:
    volumes:
      - ./:/app
```

**Проблема 5: не вижу логи**

Проверь, что приложение пишет в stdout/stderr, а не в файл. Docker видит только stdout/stderr.

### 💡 Практика: как правильно отлаживать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Всегда начинай с `docker compose logs`.** 80% проблем видно в логах.
2. **Проверяй `docker compose ps`** — статусы сервисов.
3. **Заходи в контейнер через `exec`** для диагностики.

**👍 СТОИТ:**

4. **Используй `docker compose config`** для проверки подстановки переменных.
5. **Используй `docker compose run --rm`** для одноразовых задач (миграции, тесты).
6. **Проверяй healthcheck** через `docker inspect`.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **`--scale`** для локальных тестов.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй `docker compose down -v` без понимания.** Удалит все volumes с данными.
9. **Не игнорируй статус `unhealthy`.** Даже если сервис работает, unhealthy может сломать зависимые сервисы.
10. **Не перезапускай всё при изменениях в одном сервисе.** `docker compose up -d --build app` быстрее.

### Где мы сейчас

Мы разобрали отладку и масштабирование. Теперь последняя подглава — **лимиты ресурсов** в Compose.

---

## 5.8 Лимиты ресурсов и deploy-секция

### 🔌 Проблема: один сервис съедает весь хост

В Compose-приложении один сервис (например, app с утечкой памяти) может съесть всю RAM хоста. Остальные сервисы начнут тормозить или умирать от OOM Killer (вспомни подглаву 1.5).

**Решение:** ограничить ресурсы для каждого сервиса.

### 📊 Секция `deploy`

В Compose v2 лимиты задаются через секцию `deploy`:

```yaml
services:
  app:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: '1.0'         # максимум 1 CPU
          memory: 512M        # максимум 512 МБ RAM
        reservations:
          cpus: '0.5'         # гарантировано 0.5 CPU
          memory: 256M        # гарантировано 256 МБ RAM
```

**Что означают:**

| Параметр | Что делает |
|:---|:---|
| `limits.cpus` | Максимальное использование CPU |
| `limits.memory` | Максимальное использование RAM |
| `reservations.cpus` | Гарантированное CPU (scheduler учитывает) |
| `reservations.memory` | Гарантированная RAM |

**Как это работает под капотом:**

Compose создаёт cgroups (вспомни подглаву 1.5) с параметрами:

```
limits.cpus       → cpu.max
limits.memory     → memory.max
reservations.cpus → cpu.weight
reservations.memory → memory.min
```

**Что произойдёт при превышении:**

- **CPU:** процесс throttled (ограничен).
- **Память:** OOM Killer убьёт процесс **внутри cgroup**, не тронув другие.

### ⚙️ Форматы значений

**CPU:**

```yaml
cpus: '0.5'    # половина ядра
cpus: '1.0'    # одно ядро
cpus: '2.5'    # два с половиной ядра
```

**Memory:**

```yaml
memory: 512M    # 512 мегабайт
memory: 1G      # 1 гигабайт
memory: 512MB   # то же, что 512M
```

**Поддерживаются суффиксы:** `B`, `K`, `M`, `G` (десятичные) и `KiB`, `MiB`, `GiB` (двоичные).

### 📈 Реплики

Секция `deploy.replicas` задаёт количество экземпляров сервиса:

```yaml
services:
  app:
    image: myapp:latest
    deploy:
      replicas: 3
```

**Важно:** `replicas` в Compose работает только с **Docker Swarm**. В обычном Compose используй `--scale`:

```bash
docker compose up -d --scale app=3
```

### 🔄 Политики перезапуска

В `deploy` есть политика restart:

```yaml
services:
  app:
    deploy:
      restart_policy:
        condition: on-failure     # перезапускать только при ошибке
        delay: 5s                 # ждать 5 секунд
        max_attempts: 3           # максимум 3 попытки
        window: 60s               # за 60 секунд
```

**Альтернатива — верхнеуровневый `restart`:**

```yaml
services:
  app:
    restart: unless-stopped       # проще, работает везде
```

`restart: unless-stopped` — обычный режим. `deploy.restart_policy` — для Swarm с более тонкой настройкой.

### 🧪 Практика: лимиты для типичного стека

```yaml
services:
  app:
    build: .
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M
    restart: unless-stopped
  
  postgres:
    image: postgres:16-alpine
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M
    restart: unless-stopped
    volumes:
      - postgres-data:/var/lib/postgresql/data
  
  redis:
    image: redis:7-alpine
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.1'
          memory: 64M
    restart: unless-stopped

volumes:
  postgres-data:
```

**Логика:**

- **app:** скромные лимиты — 1 CPU, 512 МБ.
- **postgres:** больше — 2 CPU, 2 ГБ (БД требовательна).
- **redis:** маленькие — 0.5 CPU, 256 МБ (в памяти только кэш).

### 🔍 Как проверить лимиты

```bash
# Посмотреть cgroup сервиса
docker compose exec app cat /sys/fs/cgroup/memory.max
# 536870912  (512 МБ в байтах)

docker compose exec app cat /sys/fs/cgroup/cpu.max
# 100000 100000  (1.0 CPU)

# Статистика по ресурсам
docker stats
# CONTAINER       CPU %    MEM USAGE / LIMIT
# myapp-app-1     15.2%    120 MiB / 512 MiB
# myapp-postgres  8.5%     450 MiB / 2 GiB
# myapp-redis     1.2%     45 MiB / 256 MiB
```

### 💡 Практика: как правильно ставить лимиты

**✅ ОБЯЗАТЕЛЬНО:**

1. **Всегда ставь `limits.memory`.** Без лимита один сервис может съесть всю RAM хоста.
2. **Ставь `limits.cpus`** для CPU-интенсивных сервисов.
3. **Ставь `reservations`** для критичных сервисов (гарантированные ресурсы).

**👍 СТОИТ:**

4. **Настраивай лимиты по фактическому потреблению:**
   ```bash
   docker stats
   # Наблюдай за потреблением несколько дней
   # Ставь лимит = 1.5-2x от типичного
   ```

5. **Документируй лимиты** в комментариях Compose-файла:
   ```yaml
   deploy:
     resources:
       limits:
         memory: 512M    # 2x от типичного потребления (250 МБ)
   ```

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **`deploy.replicas`** — только для Swarm.

**❌ НЕ ДЕЛАЙ:**

7. **Не ставь лимиты «на глаз»** без наблюдения за реальным потреблением.
8. **Не давай слишком мало памяти.** OOM Killer убьёт контейнер при первом же пике.
9. **Не забывай про `reservations`.** Без них scheduler может запустить сервисы на перегруженном хосте.

### Где мы сейчас

Мы разобрали лимиты ресурсов. Это была последняя подглава Главы 5. Теперь подведём итоги.

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **Docker Compose** | Инструмент для определения и запуска многоконтейнерных приложений. |
| **Service** | Один контейнер (или группа одинаковых) в Compose. |
| **Project** | Группа сервисов, сетей, volumes. Имя = имя директории. |
| **Compose file** | `docker-compose.yml` — описание проекта. |
| **`depends_on`** | Зависимость между сервисами. |
| **`condition`** | Условие зависимости: `service_started`, `service_healthy`, `service_completed_successfully`. |
| **Healthcheck** | Периодическая проверка готовности сервиса. |
| **`.env`** | Файл с переменными окружения. Читается Compose автоматически. |
| **Profiles** | Группы сервисов для разных окружений (dev, prod). |
| **Scale** | Запуск нескольких экземпляров сервиса. |
| **`deploy.resources`** | Лимиты CPU и памяти для сервиса. |
| **`deploy.replicas`** | Количество экземпляров (только для Swarm). |
| **`restart`** | Политика перезапуска: `no`, `always`, `unless-stopped`, `on-failure`. |
| **`exec`** | Выполнение команды в работающем контейнере. |
| **`run`** | Создание нового контейнера для одноразовой команды. |
| **`docker compose config`** | Показать итоговую конфигурацию после подстановки переменных. |
| **`COMPOSE_PROJECT_NAME`** | Переменная для задания имени проекта. |
| **`COMPOSE_PROFILES`** | Переменная для активации профилей. |

---

## Что мы узнали?

- **Docker Compose** решает проблему управления многоконтейнерными приложениями. Один YAML-файл, одна команда `docker compose up`.
- **`docker-compose.yml`** состоит из секций: services (обязательно), networks, volumes, secrets, configs.
- **`depends_on` + `healthcheck` + `condition: service_healthy`** — правильный способ ждать готовности сервисов.
- **`.env`-файлы** хранят переменные и секреты вне Git. `.env.example` коммитится как шаблон.
- **Networks в Compose** — по умолчанию одна сеть на проект. Для изоляции — разделяй frontend и backend.
- **Volumes в Compose** — именованные (управляются Docker) и bind mounts (директории хоста).
- **Profiles** позволяют держать в одном файле dev, prod и monitoring конфигурации.
- **Отладка** через `logs`, `ps`, `exec`, `config`.
- **Масштабирование** через `--scale`, но только для dev.
- **Лимиты ресурсов** через `deploy.resources.limits`. Всегда ставь `limits.memory`.

---

## Типичные ошибки

- ❌ **Использовать `depends_on` без `condition`.** Сервис запустится, но зависимость может быть ещё не готова.
- ❌ **Использовать `sleep` в command вместо healthcheck.** Костыль, который не работает на медленных машинах.
- ❌ **Хранить секреты в `docker-compose.yml`.** Используй `.env` или secrets.
- ❌ **Коммитить `.env` в Git.** Добавь в `.gitignore`.
- ❌ **Подключать БД к frontend-сети.** База должна быть изолирована в backend.
- ❌ **Использовать bind mounts для production БД.** Volumes надёжнее.
- ❌ **Не ставить `limits.memory`.** Один сервис съест всю RAM.
- ❌ **Использовать `container_name` с `--scale`.** Конфликт имён.
- ❌ **Использовать `docker compose down -v` без понимания.** Удалит все volumes с данными.
- ❌ **Использовать Compose для production-оркестрации.** Для этого есть Kubernetes.
- ❌ **Игнорировать `docker compose config`.** Не увидишь, какие переменные подставились.
- ❌ **Писать логи в файлы внутри контейнера.** Compose их не увидит.

---

## Для быстрого повторения

- **Установка:** Compose v2 встроен в Docker CLI. Вызов: `docker compose` (с пробелом).
- **Основные команды:** `up`, `down`, `ps`, `logs`, `exec`, `build`, `restart`, `config`.
- **Структура:** `services`, `networks`, `volumes`, `secrets`, `configs`.
- **Зависимости:** `depends_on: {service: {condition: service_healthy}}`.
- **Healthcheck:** `test`, `interval`, `timeout`, `retries`, `start_period`.
- **`.env`:** автоматически читается. Подстановка: `${VAR}`, `${VAR:-default}`, `${VAR:?error}`.
- **Networks:** по умолчанию одна на проект. Явно — для разделения frontend/backend.
- **Volumes:** именованные (Docker управляет) и bind mounts.
- **Profiles:** `--profile dev` активирует сервисы профиля `dev`.
- **Отладка:** `logs`, `ps`, `config`, `inspect`, `exec`.
- **Масштабирование:** `--scale app=3` (не для production).
- **Лимиты:** `deploy.resources.limits.cpus` и `.memory`.

---

## Вопросы для самопроверки

1. Зачем нужен Docker Compose, если есть `docker run`?
2. Из каких секций состоит `docker-compose.yml`? Какие обязательные?
3. Что делает `depends_on`? Чем отличается `service_started` от `service_healthy`?
4. Как заставить app ждать, пока PostgreSQL станет готов?
5. Что произойдёт без `healthcheck`, если app стартует быстрее базы?
6. Что такое `.env`-файл? Как Compose его использует?
7. Как передать секреты в Compose? Почему нельзя писать их в YAML?
8. Зачем разделять frontend и backend сети? Что это даёт?
9. Чем отличается `docker compose exec` от `docker compose run`?
10. Что такое profiles? Приведи пример использования.
11. Как посмотреть итоговую конфигурацию Compose с подставленными переменными?
12. Как задать лимит памяти 512 МБ для сервиса app?
13. Ты запустил Compose, но один сервис unhealthy. Как это диагностировать?
14. Что произойдёт, если выполнить `docker compose down -v`?

---

## Ответы

**1. Зачем Compose**

`docker run` — для одного контейнера. Compose — для многоконтейнерных приложений: декларативность, воспроизводимость, управление зависимостями, единый жизненный цикл.

**2. Секции Compose**

`services` (обязательно), `networks`, `volumes`, `secrets`, `configs` (опционально). `version` устарела в v2.

**3. `depends_on`**

Говорит Compose запустить сервис после другого. `service_started` — только после старта, не ждёт готовности. `service_healthy` — ждёт, пока сервис станет healthy. `service_completed_successfully` — ждёт успешного завершения (для миграций).

**4. App ждёт PostgreSQL**

```yaml
app:
  depends_on:
    postgres:
      condition: service_healthy

postgres:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 5s
    timeout: 3s
    retries: 5
```

**5. Без healthcheck**

App запустится, но PostgreSQL может быть ещё не готов. App попытается подключиться и упадёт с `connection refused`. С `restart: unless-stopped` app перезапустится, но будет цикл падений до готовности базы. Healthcheck решает это правильно.

**6. `.env`-файл**

Файл с переменными `KEY=value`. Compose читает его автоматически из корня проекта. Подстановка через `${VAR}`, `${VAR:-default}`, `${VAR:?error}`. Не коммитится в Git, коммитится `.env.example`.

**7. Секреты**

В `.env` или через `secrets` (в Swarm). В YAML **нельзя** — файл коммитится в Git, пароли видны в истории.

**8. Разделение сетей**

Изоляция: nginx не видит БД, app видит всех. Если злоумышленник взломает nginx — он не сможет подключиться к postgres напрямую.

**9. exec vs run**

`exec` — в работающем контейнере. `run` — новый контейнер для одной команды, удаляется после. `run --rm` полезен для миграций.

**10. Profiles**

Группы сервисов для разных окружений. Пример: `adminer` и `mailhog` с профилем `dev`, `prometheus` и `grafana` с профилем `monitoring`. Запуск: `docker compose --profile dev up`.

**11. Итоговая конфигурация**

`docker compose config` показывает YAML с подставленными переменными.

**12. Лимит памяти**

```yaml
deploy:
  resources:
    limits:
      memory: 512M
```

**13. Unhealthy сервис**

1. `docker compose logs <service>` — что в логах.
2. `docker compose ps` — статус.
3. `docker inspect <container> | jq '.[0].State.Health'` — детали healthcheck.
4. `docker compose exec <service> <healthcheck-команда>` — проверить вручную.
5. Проверить healthcheck-команду — возможно, она неверна.

**14. `docker compose down -v`**

Остановит все контейнеры, удалит их, удалит сети и **volumes** проекта. Все данные в volumes (БД, кэш) будут **удалены безвозвратно**. Использовать с осторожностью.

---

## Куда идти дальше?

Мы разобрали Docker Compose — как описывать многоконтейнерные приложения одним YAML-файлом. Теперь ты умеешь:

- Писать `docker-compose.yml` для приложения из 5-10 сервисов.
- Управлять зависимостями через `depends_on` и `healthcheck`.
- Передавать секреты через `.env`.
- Изолировать сети frontend и backend.
- Использовать profiles для dev/prod.
- Отлаживать проблемы через `logs`, `ps`, `config`.
- Ставить лимиты ресурсов.

Но Compose — это **локальная оркестрация**. Он работает на одной машине. В production тебе нужно:

- **Отказоустойчивость** — если сервер падает, контейнеры должны перезапуститься на другом.
- **Масштабирование** — если нагрузка растёт, нужно больше экземпляров.
- **Управление секретами** — не через `.env`, а через secrets-менеджер.
- **Rolling updates** — обновления без простоя.
- **Самовосстановление** — если контейнер упал, чтобы его заменили.

Это — **оркестрация**. В следующем блоке мы разберём **CI/CD** — как автоматизировать сборку и деплой. А затем — **Kubernetes**, стандарт production-оркестрации.

**Глава 6: CI/CD — конвейер доставки кода (GitLab CI).** Погнали. 🚀