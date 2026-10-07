# 🔧 Глава 3: Docker под капотом — namespaces, cgroups, containerd

**Что вы узнаете:**
- Как overlayfs склеивает слои образа в единую файловую систему.
- Что такое copy-up и whiteout на уровне файловой системы.
- Кто на самом деле запускает контейнер: цепочка Docker → containerd → runc.
- Что такое OCI-стандарт и почему он важен.
- Как собрать образ размером 10 МБ вместо 700 МБ через multi-stage builds.
- Что такое distroless-образы и когда их использовать.
- Как анализировать образ и находить в нём лишнее.

**После прочтения вы сможете:**
- Объяснить, как overlayfs работает с upperdir, lowerdir и merged.
- Проследить путь `docker run` от команды до запущенного процесса.
- Написать multi-stage Dockerfile для Go-приложения и уменьшить образ в 50 раз.
- Выбрать между alpine, distroless и scratch для финального образа.
- Использовать `docker inspect` и `dive` для анализа слоёв.

---

## Содержание

- [3.0 Пролог: образ весит 700 МБ — это нормально?](#30-пролог-образ-весит-700-мб--это-нормально)
- [3.1 Overlayfs: как слои становятся файловой системой](#31-overlayfs-как-слои-становятся-файловой-системой)
- [3.2 Containerd и runc: кто на самом деле запускает контейнер](#32-containerd-и-runc-кто-на-самом-деле-запускает-контейнер)
- [3.3 OCI-стандарт: образы и runtime](#33-oci-стандарт-образы-и-runtime)
- [3.4 Multi-stage builds: собираем маленькие образы](#34-multi-stage-builds-собираем-маленькие-образы)
- [3.5 Distroless и scratch: минимальные финальные образы](#35-distroless-и-scratch-минимальные-финальные-образы)
- [3.6 Анализ образа: dive и docker history](#36-анализ-образа-dive-и-docker-history)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 3.0 Пролог: образ весит 700 МБ — это нормально?

Ты написал простой Go-сервер. 50 строк кода. Собрал Docker-образ:

```bash
docker images myapp
# REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
# myapp        latest    abc123def456   1 minute ago    712 MB
```

**712 мегабайт.** Для приложения, которое в скомпилированном виде весит 8 МБ.

Почему так? Потому что ты использовал `FROM golang:1.22` — полный образ для разработки с компилятором, стандартной библиотекой, Git, curl, gcc. Всё это осталось в финальном образе, хотя для **запуска** нужен только скомпилированный бинарник.

Теперь представь: у тебя 20 микросервисов. Каждый образ — 700 МБ. Итого 14 ГБ на каждый деплой. Registry забит. CI/CD скачивает эти гигабайты при каждой сборке. Контейнеры медленно стартуют, потому что Docker выгружает слои с диска.

Это — реальная проблема. И она решается пониманием **как Docker работает под капотом**: слои, файловая система, цепочка запуска, multi-stage builds.

В Главе 2 мы разобрали Docker как инструмент и его сущности. Теперь нырнём внутрь и посмотрим, что происходит, когда ты:

1. Собираешь образ (`docker build`).
2. Запускаешь контейнер (`docker run`).
3. Оптимизируешь размер (`multi-stage`).

Начнём с самого важного: **как Docker склеивает слои в файловую систему**.

---

## 3.1 Overlayfs: как слои становятся файловой системой

### 🔌 Проблема: слои есть, а файловая система одна

В подглаве 2.2 мы разобрали, что образ состоит из **слоёв**. Каждый слой — это tar-архив с diff'ом файловой системы. Но когда ты запускаешь контейнер, внутри него — **единая файловая система**:

```bash
docker run -it alpine sh
ls /
# bin  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

Как из нескольких tar-архивов получается одна `/`? Ответ: **overlayfs**.

### 📦 Что такое overlayfs

**Overlayfs** — это файловая система Linux, которая **склеивает** несколько директорий в одну. Она входит в ядро Linux (с версии 3.18) и используется Docker'ом для монтирования слоёв образа.

**Принцип работы:**

Overlayfs объединяет три директории:

| Директория | Роль | Что хранит |
|:---|:---|:---|
| `lowerdir` | Нижние слои (только чтение) | Слои образа (read-only) |
| `upperdir` | Верхний слой (чтение и запись) | Изменения, которые делает контейнер |
| `merged` | Объединённое представление | То, что видит контейнер |

```
┌─────────────────────────────────────────┐
│            MERGED (то, что видит         │
│            контейнер)                    │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │  UPPERDIR (read-write)             │  │
│  │  Изменения, сделанные в контейнере │  │
│  └────────────────────────────────────┘  │
│  ┌────────────────────────────────────┐  │
│  │  LOWERDIR 3 (read-only)            │  │
│  │  Слой: RUN go build                │  │
│  └────────────────────────────────────┘  │
│  ┌────────────────────────────────────┐  │
│  │  LOWERDIR 2 (read-only)            │  │
│  │  Слой: COPY . .                    │  │
│  └────────────────────────────────────┘  │
│  ┌────────────────────────────────────┐  │
│  │  LOWERDIR 1 (read-only)            │  │
│  │  Базовый образ (golang:1.22)       │  │
│  └────────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### 🔍 Как Docker использует overlayfs

Когда ты запускаешь контейнер:

1. Docker берёт **слои образа** (read-only) и монтирует их как `lowerdir`.
2. Создаёт **пустую директорию** для изменений (read-write) — `upperdir`.
3. Монтирует overlayfs с этими директориями.
4. Контейнер видит `merged` — единую файловую систему.

Когда контейнер **читает файл**:

- Overlayfs ищет файл в `upperdir` (если контейнер его менял).
- Если нет — ищет в `lowerdir` (слои образа, сверху вниз).
- Первый найденный файл возвращается.

Когда контейнер **записывает файл**:

- Overlayfs **копирует** файл из `lowerdir` в `upperdir` (copy-up).
- Изменения применяются к копии в `upperdir`.
- Оригинал в `lowerdir` остаётся нетронутым.

### 🔄 Copy-up: ключевой механизм

**Copy-up** — это то, что делает overlayfs особенным. Когда контейнер **изменяет** файл, который лежит в `lowerdir` (слой образа, read-only):

1. Overlayfs копирует файл целиком из `lowerdir` в `upperdir`.
2. Изменения применяются к копии в `upperdir`.
3. Оригинал в `lowerdir` остаётся **нетронутым**.

**Пример:**

```
До изменения:
  lowerdir/app/config.yaml → "version: 1"
  upperdir/                → (пусто)
  merged/app/config.yaml   → "version: 1" (из lowerdir)

Контейнер выполняет: echo "version: 2" > /app/config.yaml

После изменения:
  lowerdir/app/config.yaml → "version: 1" (НЕ изменился!)
  upperdir/app/config.yaml → "version: 2" (копия с изменением)
  merged/app/config.yaml   → "version: 2" (из upperdir — он выше)
```

**Почему это важно:**

- **Слои образа неизменяемы.** Даже если контейнер меняет файл, оригинал в слое остаётся тем же. Поэтому **один образ может использоваться множеством контейнеров** — каждый получит свою копию при изменении.
- **Изоляция контейнеров.** Изменения контейнера A не видны контейнеру B — у каждого свой `upperdir`.
- **Размер контейнера растёт.** При copy-up файл **целиком** копируется в `upperdir`. Если файл 1 ГБ, а ты изменил один байт — в `upperdir` появится копия на 1 ГБ.

### 🌑 Whiteout: как удаляются файлы

Что происходит, когда контейнер **удаляет** файл из `lowerdir`? Нельзя же удалить файл из read-only слоя.

Overlayfs создаёт **whiteout-запись** в `upperdir` — специальный маркер, говорящий «этот файл удалён».

```
До удаления:
  lowerdir/app/old-file.txt → "content"
  merged/app/old-file.txt   → "content" (из lowerdir)

Контейнер выполняет: rm /app/old-file.txt

После удаления:
  lowerdir/app/old-file.txt → "content" (НЕ удалён!)
  upperdir/app/old-file.txt → whiteout (маркер «удалено»)
  merged/app/old-file.txt   → не существует (whiteout скрывает)
```

**Что это значит для DevOps:**

- **Удаление файла не уменьшает размер образа.** Файл физически остаётся в слое `lowerdir`. Это та самая проблема из подглавы 2.6: `RUN rm bigfile` в Dockerfile создаёт whiteout, но bigfile остаётся в предыдущем слое.
- **Whiteout-записи накапливаются.** Если приложение часто удаляет и создаёт файлы, `upperdir` будет накапливать whiteout-записи и копии.

### 🔬 Как это посмотреть вручную

Внутри контейнера `devops-lab` ты можешь создать свой overlayfs и посмотреть на его поведение:

```bash
# Внутри devops-lab:

# 1. Создай структуру директорий
mkdir -p /tmp/overlay/{lower,upper,work,merged}

# 2. Заполни lowerdir (симулируем слой образа)
echo "original content" > /tmp/overlay/lower/file.txt
echo "only in lower" > /tmp/overlay/lower/only-lower.txt
mkdir -p /tmp/overlay/lower/app
echo "config v1" > /tmp/overlay/lower/app/config.yaml

# 3. Смонтируй overlayfs
mount -t overlay overlay \
  -o lowerdir=/tmp/overlay/lower,upperdir=/tmp/overlay/upper,workdir=/tmp/overlay/work \
  /tmp/overlay/merged

# 4. Смотрим, что видно в merged
ls /tmp/overlay/merged/
# app  file.txt  only-lower.txt

cat /tmp/overlay/merged/file.txt
# original content  ← из lowerdir

# 5. Изменяем файл (сработает copy-up)
echo "modified content" > /tmp/overlay/merged/file.txt

# 6. Проверяем merged
cat /tmp/overlay/merged/file.txt
# modified content  ← из upperdir

# 7. Проверяем lowerdir — оригинал не изменился!
cat /tmp/overlay/lower/file.txt
# original content

# 8. Проверяем upperdir — там копия
cat /tmp/overlay/upper/file.txt
# modified content

# 9. Удаляем файл
rm /tmp/overlay/merged/only-lower.txt

# 10. Проверяем merged
ls /tmp/overlay/merged/
# app  file.txt  ← only-lower.txt исчез

# 11. Проверяем lowerdir — оригинал на месте!
ls /tmp/overlay/lower/
# app  file.txt  only-lower.txt

# 12. Проверяем upperdir — там whiteout
ls -la /tmp/overlay/upper/
# c--------- 1 root root 0, 0 ... only-lower.txt
# (символ 'c' — character device 0,0 — это whiteout-запись)
```

**Что ты увидел:**

- `merged` — единое представление.
- Чтение берётся из `lowerdir`.
- Запись создаёт копию в `upperdir` (copy-up).
- Оригинал в `lowerdir` не меняется.
- Удаление создаёт whiteout-запись в `upperdir` (в виде character device 0,0).
- Файл физически остаётся в `lowerdir`.

### 📊 Как это относится к Docker

Docker использует overlayfs для каждого контейнера:

```
/var/lib/docker/overlay2/<container-id>/
├── lowerdir → ссылка на слои образа (read-only)
├── upperdir/ → изменения контейнера (read-write)
└── merged/   → результат overlayfs
```

**Когда ты удаляешь контейнер (`docker rm`):**

- Docker удаляет `upperdir` и `merged`.
- Слои образа в `lowerdir` остаются (они разделяются между контейнерами).
- Все изменения контейнера теряются.

**Когда ты удаляешь образ (`docker rmi`):**

- Docker удаляет слои образа.
- Но только если ни один контейнер их не использует.

### 💡 Практика: что это даёт DevOps-инженеру

**✅ ОБЯЗАТЕЛЬНО:**

1. **Понимай, что образ — это read-only.** Контейнер не может изменить слои образа. Все изменения — в `upperdir`.

2. **Не храни данные в контейнере.** `upperdir` удаляется при `docker rm`. Данные — в volumes (Глава 4).

3. **Знай про copy-up.** При изменении файла он **целиком** копируется в `upperdir`. Если приложение меняет большой файл — это займёт много места.

**👍 СТОИТ:**

4. **Помни про whiteout-записи.** Удаление файла не освобождает место. Файл остаётся в слое `lowerdir`.

**❌ НЕ ДЕЛАЙ:**

5. **Не пиши большие временные файлы в контейнер.** Они раздуют `upperdir` и займут место на диске. Используй volumes или tmpfs.

6. **Не создавай и не удаляй файлы в разных слоях Dockerfile.** Удаление в новом слое создаст whiteout, но файл останется в старом слое. Удаляй в том же слое, где создал:
   ```dockerfile
   # ❌ Плохо: файл останется в слое 2 навсегда
   RUN wget https://example.com/big.tar.gz
   RUN rm big.tar.gz
   
   # ✅ Хорошо: одна инструкция — один слой — файл не остаётся
   RUN wget https://example.com/big.tar.gz && \
       tar -xzf big.tar.gz && \
       rm big.tar.gz
   ```

### Где мы сейчас

Мы разобрали overlayfs — файловую систему, которая склеивает слои образа в единое представление. Теперь ты знаешь:

- Overlayfs объединяет `lowerdir` (слои образа, read-only) и `upperdir` (изменения, read-write) в `merged`.
- Copy-up — механизм копирования файлов при записи.
- Whiteout — механизм удаления (файл остаётся в `lowerdir`).
- Слои неизменяемы, изменения изолированы в `upperdir`.

Но мы пока не ответили на вопрос: **кто на самом деле запускает контейнер?** Docker — это не одна программа. Это цепочка: Docker CLI → Docker daemon → containerd → runc.

---

## 3.2 Containerd и runc: кто на самом деле запускает контейнер

### 🔌 Проблема: что происходит между `docker run` и запущенным процессом

Ты выполняешь:

```bash
docker run -d nginx
```

Контейнер работает. Но что произошло **между** твоей командой и моментом, когда процесс nginx начал слушать порт 80?

Docker — это **не монолит**. Это набор компонентов, каждый из которых выполняет свою роль. Разберём всю цепочку.

### 📊 Цепочка запуска контейнера

```
┌───────────────────────────────────────────────────────────────────┐
│                    КОМАНДНАЯ СТРОКА                                │
│                    docker run -d nginx                             │
└───────────────────────────────┬───────────────────────────────────┘
                                │
                                │ Unix socket (/var/run/docker.sock)
                                ▼
┌───────────────────────────────────────────────────────────────────┐
│                       DOCKER DAEMON (dockerd)                      │
│                                                                    │
│  • Принимает команды от CLI через REST API                         │
│  • Управляет образами (pull, build, push)                          │
│  • Управляет сетями и volumes                                      │
│  • Управляет containerd через gRPC                                 │
└───────────────────────────────┬───────────────────────────────────┘
                                │
                                │ gRPC
                                ▼
┌───────────────────────────────────────────────────────────────────┐
│                        CONTAINERD                                  │
│                                                                    │
│  • Управляет жизненным циклом контейнеров                          │
│  • Скачивает образы из registry                                    │
│  • Управляет snapshotter'ами (overlayfs, btrfs, zfs)               │
│  • Запускает shim-процессы                                         │
│  • Один containerd на весь хост                                    │
└───────────────────────────────┬───────────────────────────────────┘
                                │
                                │ gRPC
                                ▼
┌───────────────────────────────────────────────────────────────────┐
│                    CONTAINERD-SHIM                                 │
│                                                                    │
│  • Процесс-посредник между containerd и runc                       │
│  • Держит stdin/stdout/stderr контейнера                           │
│  • Передаёт exit code контейнера containerd'у                      │
│  • Один shim на каждый контейнер                                   │
└───────────────────────────────┬───────────────────────────────────┘
                                │
                                │ fork + exec
                                ▼
┌───────────────────────────────────────────────────────────────────┐
│                          RUNC                                      │
│                                                                    │
│  • Низкоуровневый инструмент запуска контейнера                    │
│  • Создаёт namespaces (PID, Mount, Network, UTS, IPC)              │
│  • Настраивает cgroups (memory, cpu, pids)                         │
│  • Настраивает overlayfs                                           │
│  • Запускает процесс внутри namespaces                             │
│  • Выходит после запуска (не остаётся)                             │
└───────────────────────────────┬───────────────────────────────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────────┐
│                    ПРОЦЕСС nginx (PID 1 в контейнере)              │
│                                                                    │
│  Работает в изолированных namespaces с ограничениями cgroups       │
└───────────────────────────────────────────────────────────────────┘
```

### 🧩 Компоненты цепочки

**1. Docker CLI (`docker`)**

Это **клиент**. Он:

- Парсит твои команды (`docker run`, `docker build`, `docker ps`).
- Отправляет их по Unix socket (`/var/run/docker.sock`) демону.
- Показывает вывод демона в терминале.

Docker CLI **не запускает контейнеры**. Он только общается с демоном.

```bash
# Проверить, что CLI общается с демоном
docker version
# Client: Docker Engine - Community
#  Version:           25.0.0
# Server: Docker Engine - Community
#  Version:           25.0.0
```

**2. Docker daemon (`dockerd`)**

Это **демон**, который работает в фоне на хосте. Он:

- Принимает команды от CLI через REST API.
- Управляет образами (pull, build, push).
- Управляет сетями (bridge, overlay) и volumes.
- Общается с containerd через gRPC.

Раньше (до версии 1.11) Docker daemon сам запускал контейнеры. Сейчас он **делегирует** это containerd.

```bash
# Демон работает как процесс
ps aux | grep dockerd
# root  1234 ... /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

# Логи демона
journalctl -u docker
```

**3. containerd**

Это **менеджер контейнеров**, независимый от Docker. Его можно использовать напрямую (например, в Kubernetes через CRI).

Containerd:

- Управляет жизненным циклом контейнеров (create, start, stop, delete).
- Скачивает образы из registry (pull).
- Управляет snapshotter'ами (overlayfs, btrfs, zfs) для файловой системы.
- Запускает shim-процессы для каждого контейнера.
- Один containerd на весь хост.

```bash
# containerd работает как процесс
ps aux | grep containerd
# root  5678 ... /usr/bin/containerd

# Клиент containerd (ctr) для управления напрямую
ctr containers list
ctr images list
```

**4. containerd-shim**

Это **процесс-посредник**. Для каждого контейнера containerd запускает отдельный shim. Зачем?

- **Независимость от containerd.** Если containerd перезапустится, контейнеры продолжат работать — shim держит их.
- **Держит I/O.** Shib держит stdin/stdout/stderr контейнера.
- **Передаёт exit code.** Когда контейнер завершается, shim передаёт код возврата containerd'у.
- **Держит TTY.** Если контейнер запущен с `-it`, shim обеспечивает терминал.

```bash
# Процессы shim
ps aux | grep containerd-shim
# root  9012 ... /usr/bin/containerd-shim-runc-v2 -namespace moby -id abc123...
```

**5. runc**

Это **низкоуровневый инструмент запуска контейнера**. Он:

- Читает конфигурацию контейнера (JSON).
- Создаёт namespaces (PID, Mount, Network, UTS, IPC).
- Настраивает cgroups (memory, cpu, pids).
- Монтирует overlayfs.
- Запускает процесс внутри namespaces.
- **Выходит после запуска.** RunC не остаётся в памяти.

```bash
# runc — это бинарник
which runc
# /usr/bin/runc

# Версия
runc --version
# runc version 1.1.12
```

**6. Процесс nginx (PID 1 в контейнере)**

Это **то, что ты запустил**. Процесс работает в изолированных namespaces с ограничениями cgroups. Он думает, что он PID 1 в системе.

### 🔬 Почему такая сложная цепочка

Может показаться, что это overengineering: зачем 5 компонентов, если можно было бы всё делать одним?

**Исторически:**

- Изначально Docker был монолитом: `dockerd` сам запускал контейнеры через `libcontainer`.
- В 2015 году Docker выделил containerd как отдельный проект.
- В 2016 году появился OCI (Open Container Initiative) и стандарт runtime — runc.
- С версии Docker 1.11 `dockerd` делегирует containerd.

**Зачем разделение:**

1. **Стандартизация.** OCI-стандарт позволяет разным инструментам (Docker, Kubernetes, Podman) использовать один runtime (runc).
2. **Независимость.** containerd можно использовать без Docker. Kubernetes использует containerd напрямую (через CRI).
3. **Модульность.** Можно заменить runc на другой runtime (crun, gVisor, Kata) без изменения остальной цепочки.
4. **Стабильность.** Если один компонент упадёт, остальные продолжат работать (containerd-shim держит контейнеры).

### 🧪 Как это посмотреть

```bash
# 1. Запусти контейнер
docker run -d --name test-nginx nginx

# 2. Посмотри цепочку процессов
ps auxf | grep -A 3 -B 3 nginx
```

Вывод (упрощённо):

```
root  1234  dockerd
root  5678   └─ containerd
root  9012        └─ containerd-shim-runc-v2 -namespace moby -id abc123
root  9013             └─ nginx: master process nginx -g daemon off;
root  9020                  └─ nginx: worker process
```

**Что ты видишь:**

- `dockerd` — демон Docker.
- `containerd` — менеджер контейнеров.
- `containerd-shim-runc-v2` — shim-процесс.
- `nginx: master process` — PID 1 в контейнере.
- `nginx: worker process` — worker-процесс nginx.

**Обрати внимание:** `runc` в списке **нет**. Он уже завершился. RunC только создаёт контейнер и выходит.

```bash
# 3. Посмотри через containerd CLI (если установлен)
ctr containers list
ctr tasks list
```

### 📊 Что происходит при `docker run`

Пошагово:

1. **CLI парсит команду** `docker run -d nginx`.
2. **CLI отправляет запрос** по Unix socket демону `dockerd`.
3. **dockerd принимает запрос** и проверяет:
   - Есть ли образ `nginx` локально?
   - Если нет — скачивает через containerd.
4. **dockerd создаёт контейнер** через containerd (gRPC).
5. **containerd создаёт snapshot** (overlayfs-монтирование слоёв образа).
6. **containerd запускает containerd-shim** для контейнера.
7. **shim запускает runc** с конфигурацией контейнера.
8. **runc создаёт namespaces** и настраивает cgroups.
9. **runc запускает nginx** внутри namespaces.
10. **runc выходит**, оставляя shim и nginx работать.
11. **shim держит I/O и передаёт exit code** containerd'у, когда nginx завершится.
12. **containerd сообщает dockerd** о завершении контейнера.

### 💡 Практика: что это даёт DevOps-инженеру

**✅ ОБЯЗАТЕЛЬНО:**

1. **Знай, что Docker — это не монолит.** Если что-то не работает — проблема может быть в любом компоненте: CLI, daemon, containerd, shim, runc.

2. **Понимай, что runc — это OCI-runtime.** Kubernetes использует containerd напрямую, но runc под капотом тот же.

**👍 СТОИТ:**

3. **Знай про `ctr` и `crictl`.** Это CLI-инструменты для containerd. Полезны, когда Docker не работает, а контейнеры нужно посмотреть.

4. **Помни про containerd-shim.** Если контейнер работает, а Docker daemon упал — контейнер продолжит работать. Shim держит его.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **Разбираться в деталях OCI-конфига** (JSON-файл, описывающий контейнер). Это полезно для глубокой отладки, но не для повседневной работы.

**❌ НЕ ДЕЛАЙ:**

6. **Не думай, что `docker stop` «убивает» контейнер.** Это отправка SIGTERM процессу PID 1 внутри namespaces. Процесс сам завершается (или получает SIGKILL через 10 секунд).

7. **Не удаляй containerd-shim вручную.** Это часть работающего контейнера. Если удалишь — контейнер потеряет I/O.

### Где мы сейчас

Мы разобрали всю цепочку запуска контейнера:

- **Docker CLI** → парсит команду и отправляет демону.
- **Docker daemon** → принимает команды, управляет образами и сетями.
- **containerd** → управляет жизненным циклом контейнеров.
- **containerd-shim** → держит I/O контейнера.
- **runc** → создаёт namespaces и cgroups, запускает процесс, выходит.
- **Процесс** → работает в изолированном окружении.

Теперь посмотрим на **OCI-стандарт** — что это такое, зачем появился, и почему он важен для всего контейнерного мира.

---

## 3.3 OCI-стандарт: образы и runtime

### 🔌 Проблема: Docker был один, а теперь их много

До 2015 года существовал только Docker. Один формат образов, один runtime, один registry. Но потом появились:

- **Podman** — альтернатива Docker без демона.
- **CRI-O** — runtime для Kubernetes.
- **containerd** — выделенный из Docker.
- **Buildah** — для сборки образов.
- **Kaniko** — сборка в Kubernetes.

Каждый из них использовал **свой формат образов** и **свой runtime**. Это создавало хаос: образ, собранный в Docker, не запускался в Podman. Runtime, работающий с Podman, не работал с Kubernetes.

Решение — **стандартизация**. В 2015 году был создан **OCI (Open Container Initiative)** — организация под эгидой Linux Foundation, которая разрабатывает открытые стандарты для контейнеров.

### 📜 Что стандартизирует OCI

OCI определил **три спецификации**:

**1. Runtime Specification (runtime-spec)**

Определяет, **как запускать контейнер**. Конкретно — описывает:

- Формат конфигурации контейнера (JSON-файл `config.json`).
- Какие namespaces использовать.
- Какие cgroups настраивать.
- Как монтировать файловую систему.
- Какой процесс запускать.
- Как обрабатывать сигналы.

**Любой runtime, совместимый с OCI**, умеет читать `config.json` и запускать контейнер по этой конфигурации. Примеры OCI-runtime:

| Runtime | Особенности |
|:---|:---|
| **runc** | Стандартный, используется в Docker и containerd |
| **crun** | Написан на C, быстрее и легче runc |
| **gVisor** | Изолирует контейнер через отдельное ядро (для безопасности) |
| **Kata Containers** | Запускает контейнер в лёгкой VM (для изоляции) |

**2. Image Specification (image-spec)**

Определяет, **как хранить образы**. Конкретно:

- Формат образа: слои + манифест + конфиг.
- Формат манифеста (JSON).
- Формат конфига (JSON): CMD, ENV, EXPOSE.
- Как хранить слои (tar + gzip).
- Как адресовать слои (SHA-хеши).

**Любой образ, совместимый с OCI**, можно запустить в любом OCI-runtime. Примеры:

- Образы Docker Hub — OCI-совместимые.
- Образы в registry — OCI-совместимые.
- Образы, собранные Buildah — OCI-совместимые.

**3. Distribution Specification (distribution-spec)**

Определяет, **как registry хранит и раздаёт образы**. Конкретно:

- REST API для registry (`/v2/...`).
- Формат манифестов и blobs.
- Аутентификация и авторизация.

**Любой OCI-совместимый registry** умеет работать с любым OCI-совместимым образом. Примеры:

- Docker Hub — OCI-совместимый.
- GitLab Container Registry — OCI-совместимый.
- Harbor — OCI-совместимый.
- `registry:2` — OCI-совместимый.

### 📊 Что это значит на практике

**До OCI:**

- Образ, собранный в Docker, не работал в Podman.
- Runtime, работающий в Podman, не работал в Kubernetes.
- Registry, работающий с Docker, не работал с другими инструментами.

**После OCI:**

- Образ, собранный в Docker, работает в Podman, containerd, CRI-O.
- Образ, собранный Buildah, работает в Docker.
- Один registry для всех инструментов.

**Пример:**

```bash
# Собираем образ в Docker
docker build -t myapp:1.0 .
docker push myregistry.com/myapp:1.0

# Запускаем его в Podman
podman pull myregistry.com/myapp:1.0
podman run -d myregistry.com/myapp:1.0

# Запускаем его в Kubernetes через containerd
kubectl run myapp --image=myregistry.com/myapp:1.0

# Всё работает, потому что образ — OCI-совместимый
```

### 🔬 Как посмотреть на OCI-конфиг

Конфигурация контейнера в OCI — это JSON-файл `config.json`. Docker и runc генерируют его автоматически, но можно посмотреть, что внутри.

**Способ 1: через `docker inspect`**

```bash
docker inspect test-nginx | jq '.[0] | {Config, HostConfig}'
```

Вывод (упрощённо):

```json
{
  "Config": {
    "Hostname": "abc123def456",
    "Env": ["PATH=/usr/local/sbin:...", "NGINX_VERSION=1.25.3"],
    "Cmd": ["nginx", "-g", "daemon off;"],
    "Image": "nginx",
    "WorkingDir": "",
    "ExposedPorts": {"80/tcp": {}}
  },
  "HostConfig": {
    "Memory": 0,
    "CpuShares": 0,
    "PidsLimit": -1,
    "NetworkMode": "default"
  }
}
```

Это **почти OCI-конфиг**, но в формате Docker. RunC преобразует его в чистый OCI-формат.

**Способ 2: посмотреть OCI-конфиг внутри контейнера**

```bash
# Найти shim-процесс контейнера
ps aux | grep "containerd-shim.*test-nginx"

# Найти PID процесса nginx
docker inspect test-nginx | grep -i pid
# "Pid": 9013

# Посмотреть OCI-конфиг (на хосте Linux)
ls /run/containerd/io.containerd.runtime.v2.task/moby/<container-id>/
# config.json  init.pid  rootfs  ...

cat /run/containerd/io.containerd.runtime.v2.task/moby/<container-id>/config.json | jq
```

Вывод — полный OCI-конфиг:

```json
{
  "ociVersion": "1.0.2",
  "process": {
    "terminal": false,
    "user": {"uid": 0, "gid": 0},
    "args": ["nginx", "-g", "daemon off;"],
    "env": ["PATH=/usr/local/sbin:...", "NGINX_VERSION=1.25.3"],
    "cwd": "/"
  },
  "root": {
    "path": "rootfs",
    "readonly": false
  },
  "hostname": "abc123def456",
  "mounts": [...],
  "linux": {
    "namespaces": [
      {"type": "pid"},
      {"type": "network"},
      {"type": "ipc"},
      {"type": "uts"},
      {"type": "mount"}
    ],
    "resources": {
      "memory": {"limit": 0},
      "cpu": {"shares": 0}
    }
  }
}
```

**Что ты видишь:**

- `ociVersion` — версия OCI-стандарта.
- `process.args` — команда запуска (CMD из Dockerfile).
- `process.env` — переменные окружения.
- `root.path` — путь к overlayfs (rootfs).
- `hostname` — hostname контейнера.
- `linux.namespaces` — какие namespaces создавать.
- `linux.resources` — лимиты cgroups.

Это **полное описание контейнера**. RunC читает этот файл и создаёт контейнер по этой конфигурации.

### 🌍 Зачем это DevOps-инженеру

**1. Переносимость.** Ты можешь собирать образы в Docker, а запускать в Kubernetes через containerd. Или в Podman. Или в CRI-O. Образ один — работает везде.

**2. Выбор runtime.** Если тебе нужна **большая безопасность** — используй gVisor или Kata Containers. Если **скорость** — crun. Всё это работает через OCI, без изменения образа.

**3. Понимание Kubernetes.** Kubernetes использует CRI (Container Runtime Interface), который под капотом вызывает containerd, который вызывает OCI-runtime. Понимание OCI помогает понять, как K8s управляет контейнерами.

**4. Понимание registry.** Любой OCI-совместимый registry (Harbor, GitLab Registry, ECR) работает с любым OCI-образом. Ты можешь переключаться между registry без пересборки образов.

### 💡 Практика: что это даёт DevOps-инженеру

**✅ ОБЯЗАТЕЛЬНО:**

1. **Знай, что OCI — это три спецификации:** runtime, image, distribution.
2. **Понимай, что образы Docker — OCI-совместимые.** Они работают в Podman, containerd, CRI-O.
3. **Знай, что runc — это OCI-runtime.** Kubernetes использует его через containerd.

**👍 СТОИТ:**

4. **Попробуй запустить тот же образ в Podman,** чтобы убедиться в совместимости.

5. **Знай про альтернативные runtime** (gVisor, Kata) — они нужны для специфических сценариев безопасности.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Читать спецификации OCI целиком.** Они большие. Достаточно понимать общие принципы.

**❌ НЕ ДЕЛАЙ:**

7. **Не думай, что Docker — единственный инструмент.** OCI-стандарт открыл экосистему: Podman, Buildah, Kaniko, containerd, CRI-O — всё это работает с OCI-образами.

### Где мы сейчас

Мы разобрали OCI-стандарт. Теперь ты знаешь:

- OCI — это три спецификации: runtime, image, distribution.
- Runtime-spec определяет, как запускать контейнер (config.json).
- Image-spec определяет формат образов.
- Distribution-spec определяет API registry.
- Любой OCI-совместимый образ работает в любом OCI-runtime и registry.

Теперь перейдём к **практике оптимизации**: multi-stage builds. Как собрать образ размером 10 МБ вместо 700 МБ.

---

## 3.4 Multi-stage builds: собираем маленькие образы

### 🔌 Проблема: 700 МБ для 8 МБ приложения

Вернёмся к проблеме из пролога. Твой Dockerfile для Go-приложения:

```dockerfile
FROM golang:1.22-alpine

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o myapp .

EXPOSE 8080
CMD ["./myapp"]
```

**Размер образа:** ~700 МБ.

**Что внутри:**

- Go-компилятор: ~200 МБ.
- Стандартная библиотека Go: ~100 МБ.
- Зависимости: ~50 МБ.
- Git, curl, bash: ~30 МБ.
- Скомпилированный бинарник: **8 МБ**.

**Проблема:** для **запуска** нужен только бинарник. Всё остальное — нужно только для **сборки**. Но оно остаётся в образе навсегда.

### 📦 Решение: multi-stage builds

**Multi-stage build** — это Dockerfile с несколькими стадиями (stages). Каждая стадия — это отдельный `FROM`. Образ собирается **только из последней стадии**, но предыдущие стадии могут передавать артефакты в последнюю.

```dockerfile
# Стадия 1: сборка (builder)
FROM golang:1.22-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o myapp .

# Стадия 2: финальный образ
FROM alpine:3.19

WORKDIR /app

# Копируем ТОЛЬКО бинарник из первой стадии
COPY --from=builder /app/myapp .

EXPOSE 8080
CMD ["./myapp"]
```

**Что происходит:**

1. Docker собирает стадию `builder` (700 МБ — но это **промежуточный** образ).
2. Docker собирает стадию 2 (`alpine:3.19` — 7 МБ).
3. Из стадии `builder` копируется **только** `/app/myapp` (8 МБ) в стадию 2.
4. Финальный образ = Alpine (7 МБ) + бинарник (8 МБ) = **15 МБ**.

**Разница:** 700 МБ → 15 МБ. **В 46 раз меньше.**

### 🔬 Как это работает

**Ключевые моменты:**

1. **`AS builder`** — даёт стадии имя. По имени можно ссылаться в `COPY --from=builder`.
2. **`COPY --from=builder`** — копирует файлы **из предыдущей стадии**. Не из контекста сборки, а из промежуточного образа.
3. **Финальный образ содержит только последнюю стадию.** Все промежуточные стадии отбрасываются (если не указать `--target`).

**Посмотреть, какие стадии есть:**

```bash
docker build -t myapp .
docker history myapp
# Показывает только слои финальной стадии — Alpine + бинарник
```

### 📊 Пример: Node.js приложение

**Без multi-stage:**

```dockerfile
FROM node:20

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

CMD ["node", "dist/index.js"]
```

**Размер:** ~1.2 ГБ (node_modules + dev-зависимости + исходники).

**С multi-stage:**

```dockerfile
# Стадия 1: сборка
FROM node:20 AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Стадия 2: финальный образ
FROM node:20-alpine

WORKDIR /app

# Только production-зависимости
COPY package*.json ./
RUN npm ci --only=production

# Только собранный код
COPY --from=builder /app/dist ./dist

CMD ["node", "dist/index.js"]
```

**Размер:** ~150 МБ (Alpine + production node_modules + dist).

### 🎯 Три плюса multi-stage

**1. Маленький образ.**

- Быстрее pull в CI/CD.
- Быстрее deploy (Docker выгружает меньше слоёв с диска).
- Меньше занимает место в registry.

**2. Безопасность.**

- В финальном образе нет компилятора, Git, curl, bash.
- Меньше поверхности для атаки.
- Нет исходного кода и dev-инструментов.

**3. Чистота.**

- В финальном образе только runtime.
- Легче анализировать, что внутри.

### 🧪 Практика: собери multi-stage образ

Создай простой Go-проект и два Dockerfile:

```bash
mkdir multi-stage-test && cd multi-stage-test
```

**main.go:**

```go
package main

import (
    "fmt"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello from multi-stage!\n")
}

func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
```

**go.mod:**

```bash
go mod init multi-stage-test
```

**Dockerfile.single (один этап):**

```dockerfile
FROM golang:1.22-alpine
WORKDIR /app
COPY . .
RUN go build -o myapp .
EXPOSE 8080
CMD ["./myapp"]
```

**Dockerfile.multi (multi-stage):**

```dockerfile
# Стадия сборки
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp .

# Финальная стадия
FROM alpine:3.19
WORKDIR /app
COPY --from=builder /app/myapp .
EXPOSE 8080
CMD ["./myapp"]
```

**Собери оба и сравни:**

```bash
# Собери single-stage
docker build -f Dockerfile.single -t myapp:single .

# Собери multi-stage
docker build -f Dockerfile.multi -t myapp:multi .

# Сравни размеры
docker images | grep myapp
# REPOSITORY   TAG      SIZE
# myapp        single   712 MB
# myapp        multi    15 MB
```

**Разница:** 712 МБ → 15 МБ.

### 💡 Практика: как правильно писать multi-stage Dockerfile

**✅ ОБЯЗАТЕЛЬНО:**

1. **Давай стадиям имена** (`AS builder`):
   ```dockerfile
   FROM golang:1.22-alpine AS builder
   ```
   Без имени ты не сможешь ссылаться на стадию в `COPY --from`.

2. **Копируй только артефакты, не весь проект:**
   ```dockerfile
   COPY --from=builder /app/myapp .
   ```
   Не копируй `/app` целиком — там исходники, кэши, всё лишнее.

3. **Используй минимальный финальный образ** (alpine, distroless, scratch — см. подглаву 3.5).

**👍 СТОИТ:**

4. **Разделяй dev и prod зависимости:**
   ```dockerfile
   # В builder — все зависимости (включая dev)
   RUN npm ci
   # В финальном образе — только production
   RUN npm ci --only=production
   ```

5. **Используй кэширование слоёв** (подглава 2.6):
   ```dockerfile
   COPY go.mod go.sum ./
   RUN go mod download
   COPY . .
   ```
   Это ускоряет сборку.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

6. **Использовать `--target`** для сборки промежуточных стадий:
   ```bash
   docker build --target builder -t myapp:builder .
   ```
   Полезно для отладки, если сборка ломается.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `FROM` без стадии** в multi-stage:
   ```dockerfile
   FROM golang:1.22 AS builder  # ✅
   FROM golang:1.22              # ❌ — это тоже стадия, но без имени
   ```

8. **Не копируй секреты в промежуточную стадию.** Даже если они не попадут в финальный образ, они могут остаться в кэше слоёв.

9. **Не забывай про архитектуру.** Если собираешь на маке с Apple Silicon, а сервер x86_64 — используй `--platform`:
   ```bash
   docker build --platform linux/amd64 -t myapp .
   ```

### Где мы сейчас

Мы разобрали multi-stage builds. Теперь ты знаешь:

- Multi-stage — это Dockerfile с несколькими `FROM`.
- Финальный образ содержит только последнюю стадию.
- `COPY --from=builder` передаёт артефакты между стадиями.
- Размер образа уменьшается в 10-50 раз.

Но даже с multi-stage финальный образ содержит `alpine` — 7 МБ с shell, apk, минимальными утилитами. Можно ли сделать ещё меньше? Да — **distroless** и **scratch**.

---

## 3.5 Distroless и scratch: минимальные финальные образы

### 🔌 Проблема: даже alpine содержит лишнее

В multi-stage build мы использовали `alpine:3.19` как финальный образ. Размер — 7 МБ. Но что внутри?

- `sh` — shell.
- `apk` — пакетный менеджер.
- `busybox` — утилиты (ls, cat, grep).
- `musl libc` — стандартная библиотека C.

Для **запуска** Go-бинарника ничего из этого не нужно. Go компилируется в статический бинарник — он не зависит от libc, shell, утилит. Ему нужен только **сам бинарник**.

Можно ли убрать всё лишнее?

### 📦 Вариант 1: scratch

**`scratch`** — это **пустой образ**. 0 байт. Не содержит ничего: ни файловой системы, ни shell, ни библиотек.

```dockerfile
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o myapp .

# Финальная стадия — пустой образ
FROM scratch

COPY --from=builder /app/myapp /myapp

EXPOSE 8080
ENTRYPOINT ["/myapp"]
```

**Размер:** 8 МБ (только бинарник).

**Ключевое требование:** бинарник должен быть **статически слинкованным** (`CGO_ENABLED=0`), потому что в `scratch` нет libc.

**Что работает:**

- Запуск бинарника.
- TCP-соединения.

**Что НЕ работает:**

- Shell (`docker exec -it myapp sh` не сработает — нет shell).
- DNS (нет `/etc/resolv.conf` — нужно копировать вручную).
- HTTPS (нет CA-сертификатов — нужно копировать вручную).
- Time zones (нет `/usr/share/zoneinfo`).

**Как добавить нужное:**

```dockerfile
FROM scratch

# Копируем бинарник
COPY --from=builder /app/myapp /myapp

# Копируем CA-сертификаты (для HTTPS)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Копируем timezone
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# Копируем пользователей (для non-root)
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /etc/group /etc/group

EXPOSE 8080
ENTRYPOINT ["/myapp"]
```

### 📦 Вариант 2: distroless

**Distroless** — это образы от Google, которые содержат **минимум**: libc, CA-сертификаты, timezone, пользователей. Но **не содержат** shell, пакетного менеджера, утилит.

Цель: **безопасность**. Меньше инструментов внутри — меньше возможностей для атакующего.

**Основные distroless-образы:**

| Образ | Что содержит |
|:---|:---|
| `gcr.io/distroless/static` | Только минимальный runtime для статических бинарников |
| `gcr.io/distroless/base` | + glibc, libssl |
| `gcr.io/distroless/java` | + JRE (для Java-приложений) |
| `gcr.io/distroless/nodejs` | + Node.js runtime |
| `gcr.io/distroless/python3` | + Python 3 runtime |

**Пример для Go:**

```dockerfile
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o myapp .

FROM gcr.io/distroless/static-debian12

COPY --from=builder /app/myapp /myapp

EXPOSE 8080
ENTRYPOINT ["/myapp"]
```

**Размер:** ~12 МБ (бинарник + минимальный runtime).

**Что внутри distroless/static:**

- CA-сертификаты (для HTTPS).
- `/etc/passwd` с пользователем `nonroot`.
- Timezone.
- **Нет shell, нет пакетного менеджера, нет утилит.**

**Что не работает:**

- `docker exec -it myapp sh` — не сработает.

**Как отлаживать:**

Distroless-образы имеют `debug`-версии с busybox:

```dockerfile
FROM gcr.io/distroless/static-debian12:debug
```

Или используй ephemeral containers в Kubernetes для отладки (это отдельная тема).

### 📊 Сравнение вариантов

| Финальный образ | Размер | Shell | Пакеты | Безопасность | Когда использовать |
|:---|:---|:---|:---|:---|:---|
| **ubuntu:24.04** | 78 МБ | ✅ | ✅ | Низкая | Dev-окружения, отладка |
| **debian-slim** | 30 МБ | ✅ | ✅ | Средняя | Приложения с glibc |
| **alpine** | 7 МБ | ✅ | ✅ | Средняя | Большинство приложений |
| **distroless** | 12 МБ | ❌ | ❌ | Высокая | Продакшен, Go/Java/Node |
| **scratch** | 8 МБ | ❌ | ❌ | Максимальная | Продакшен, статические бинарники |

### 🧪 Практика: собери три варианта

Продолжим проект `multi-stage-test`:

**Dockerfile.scratch:**

```dockerfile
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o myapp .

FROM scratch
COPY --from=builder /app/myapp /myapp
EXPOSE 8080
ENTRYPOINT ["/myapp"]
```

**Dockerfile.distroless:**

```dockerfile
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o myapp .

FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/myapp /myapp
EXPOSE 8080
ENTRYPOINT ["/myapp"]
```

**Собери и сравни:**

```bash
docker build -f Dockerfile.single -t myapp:single .
docker build -f Dockerfile.multi -t myapp:multi .
docker build -f Dockerfile.scratch -t myapp:scratch .
docker build -f Dockerfile.distroless -t myapp:distroless .

docker images | grep myapp
# REPOSITORY   TAG           SIZE
# myapp        single        712 MB
# myapp        multi         15 MB
# myapp        scratch       8 MB
# myapp        distroless    12 MB
```

**Проверь, что работает:**

```bash
docker run -d --name test-scratch -p 8081:8080 myapp:scratch
curl localhost:8081
# Hello from multi-stage!

docker stop test-scratch
```

**Попробуй зайти внутрь:**

```bash
docker exec -it test-scratch sh
# OCI runtime exec failed: exec failed: unable to start container process: exec: "sh": executable file not found in $PATH
# ← В scratch нет shell!
```

### 💡 Практика: как выбирать финальный образ

**✅ ОБЯЗАТЕЛЬНО:**

1. **Для Go — используй scratch или distroless/static.** Go компилируется в статический бинарник, не нуждается в libc.

2. **Для Java — используй distroless/java.** Не содержит shell, но содержит JRE.

3. **Для Node.js — используй distroless/nodejs или alpine.** Distroless безопаснее, но не содержит npm-скриптов.

**👍 СТОИТ:**

4. **Копируй CA-сертификаты, если приложение работает с HTTPS:**
   ```dockerfile
   COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
   ```

5. **Копируй timezone, если приложение работает со временем:**
   ```dockerfile
   COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
   ```

6. **Используй non-root пользователя:**
   ```dockerfile
   FROM gcr.io/distroless/static-debian12:nonroot
   USER nonroot
   ```

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

7. **Distroless `debug`-версии** для отладки. Используй только в dev, не в prod.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй scratch для приложений с glibc-зависимостями.** Python, Ruby, Node.js требуют glibc — используй distroless/base.

9. **Не используй distroless без понимания ограничений.** Не сможешь зайти в контейнер для отладки. Нужны внешние инструменты (логи, метрики, ephemeral containers).

10. **Не забывай про CGO_ENABLED=0.** Без этого Go-бинарник будет динамически слинкован и не запустится в scratch.

### Где мы сейчас

Мы разобрали минимальные финальные образы:

- **scratch** — пустой образ, для статических бинарников.
- **distroless** — минимум для runtime, безопаснее alpine.
- **alpine** — компромисс: маленький, но с shell.
- **ubuntu** — для dev и отладки.

Теперь посмотрим, как **анализировать образ** — что внутри, какие слои большие, где лишнее.

---

## 3.6 Анализ образа: dive и docker history

### 🔌 Проблема: образ 500 МБ, но я не понимаю, что там

Ты скачал образ из Docker Hub. Размер — 500 МБ. Но что внутри? Какие слои? Что занимает больше всего места? Есть ли в образе секреты?

Без инструментов анализа — это чёрный ящик.

### 📊 Способ 1: docker history

Самый простой способ — `docker history`:

```bash
docker history nginx:latest
```

Вывод:

```
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
abc123def456   2 weeks ago   CMD ["nginx" "-g" "daemon off;"]                0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   STOPSIGNAL SIGQUIT                              0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   EXPOSE map[80/tcp:{}]                           0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   ENTRYPOINT ["/docker-entrypoint.sh"]            0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   COPY file:abc... /docker-entrypoint.d/ # buildkit 3.5kB   buildkit.dockerfile.v0
<missing>      2 weeks ago   RUN /bin/sh -c set -x && ...                   60.5MB   buildkit.dockerfile.v0
<missing>      2 weeks ago   RUN /bin/sh -c apt-get update && ...            10.2MB   buildkit.dockerfile.v0
<missing>      2 weeks ago   ADD file:def... in /                           62.5MB   buildkit.dockerfile.v0
```

**Что ты видишь:**

- Каждая строка — слой.
- `SIZE` — размер слоя.
- `CREATED BY` — инструкция, которая создала слой.
- `<missing>` — старые слои (не всегда доступны для просмотра).

**Ограничения `docker history`:**

- Не показывает содержимое слоёв.
- Не показывает, какие файлы занимают место.
- Не показывает, что можно удалить.

### 🔍 Способ 2: dive

**dive** — это инструмент для анализа образов. Он показывает:

- Каждый слой отдельно.
- Какие файлы добавлены, изменены, удалены в каждом слое.
- Эффективность образа (сколько места тратится впустую).
- Возможные оптимизации.

**Установка:**

```bash
# На маке:
brew install dive

# На Linux:
wget https://github.com/wagoodman/dive/releases/download/v0.11.0/dive_0.11.0_linux_amd64.deb
sudo dpkg -i dive_0.11.0_linux_amd64.deb
```

**Запуск:**

```bash
dive nginx:latest
```

**Что ты увидишь:**

```
┌──────────────────────────────────────────────────────────────────────┐
│ Layers                                    │ Image Details            │
│                                           │                          │
│ ● abc123def456 (62 MB)                    │ Total Size: 187 MB       │
│ ● def456abc123 (10 MB)                    Potential Savings: 5 MB   │
│ ● ghi789def456 (60 MB)                    Efficiency: 97%           │
│ ● jkl012ghi345 (55 MB)                    │                          │
│                                           │                          │
├──────────────────────────────────────────────────────────────────────┤
│ Layer Details                                                         │
│                                                                       │
│ Command: RUN apt-get update && apt-get install -y ...                │
│                                                                       │
│ Files added:                                                          │
│ /usr/bin/nginx              2.3 MB                                    │
│ /usr/share/nginx/html       1.2 MB                                    │
│ /etc/nginx/nginx.conf       5 KB                                      │
│                                                                       │
│ Files modified:                                                       │
│ /var/lib/dpkg/status        20 KB                                     │
│                                                                       │
│ Files removed:                                                        │
│ /var/lib/apt/lists/*.list   (but whiteout)                            │
└──────────────────────────────────────────────────────────────────────┘
```

**Ключевые метрики:**

- **Total Size** — общий размер образа.
- **Potential Savings** — сколько можно сэкономить, удалив ненужное.
- **Efficiency** — процент эффективности. 100% = нет лишнего.
- **Files added/modified/removed** — что изменилось в слое.

**Как использовать:**

1. **Проверяй свои образы** — где тратится место.
2. **Анализируй чужие образы** — что внутри, есть ли подозрительное.
3. **Находи оптимизации** — какие слои можно уменьшить.

### 🔬 Практика: анализируем свой образ

```bash
# 1. Собери свой образ (из подглавы 3.4)
docker build -f Dockerfile.single -t myapp:single .

# 2. Запусти dive
dive myapp:single
```

**Что ты увидишь:**

- Слой базового образа `golang:1.22-alpine` — 250 МБ.
- Слой `go mod download` — 50 МБ (зависимости).
- Слой `go build` — 8 МБ (бинарник).
- Слой `COPY . .` — 100 КБ (исходники).

**Efficiency:** низкая, потому что 300 МБ — это компилятор и зависимости, которые не нужны для запуска.

**После multi-stage:**

```bash
docker build -f Dockerfile.multi -t myapp:multi .
dive myapp:multi
```

**Что ты увидишь:**

- Слой Alpine — 7 МБ.
- Слой с бинарником — 8 МБ.

**Efficiency:** 100% — всё нужное, ничего лишнего.

### 🛠️ Другие инструменты анализа

**1. docker scout**

Встроенный в Docker инструмент для сканирования уязвимостей:

```bash
docker scout cves myapp:latest
```

Показывает уязвимые пакеты в образе.

**2. trivy**

Сканер уязвимостей и секретов:

```bash
trivy image myapp:latest
```

Показывает:

- Уязвимости (CVE) в пакетах.
- Секреты в слоях (пароли, ключи, токены).
- Misconfigurations.

**3. syft**

SBOM (Software Bill of Materials) — список всех пакетов в образе:

```bash
syft myapp:latest
```

Вывод — список пакетов с версиями.

**4. docker sbom** (встроенный)

```bash
docker sbom myapp:latest
```

### 💡 Практика: как анализировать образ

**✅ ОБЯЗАТЕЛЬНО:**

1. **Регулярно проверяй свои образы через `dive`.** Ищи слои с лишним.
2. **Сканируй образы на уязвимости** через `trivy` или `docker scout`. Особенно перед деплоем в prod.

**👍 СТОИТ:**

3. **Используй `docker history`** для быстрой проверки слоёв:
   ```bash
   docker history myapp:latest --no-trunc
   ```
   Показывает полные команды, которые создали слои.

4. **Проверяй базовые образы** перед использованием:
   ```bash
   dive nginx:latest
   ```
   Убеждаешься, что там нет ничего лишнего.

**🤔 НЕ ОБЯЗАТЕЛЬНО:**

5. **SBOM** для compliance. Нужно, если работаешь в регулируемой индустрии.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй образы без анализа.** Образ из интернета может содержать майнер, бэкдор, или уязвимые пакеты.

7. **Не игнорируй `Potential Savings`.** Если dive показывает 50% savings — образ можно уменьшить вдвое.

### Где мы сейчас

Мы разобрали анализ образов:

- `docker history` — быстрый просмотр слоёв.
- `dive` — детальный анализ с эффективностью и потенциальной экономией.
- `trivy`, `docker scout` — сканирование на уязвимости.
- `syft` — SBOM.

Теперь мы завершили техническую часть Главы 3. Ты знаешь:

- Как overlayfs склеивает слои.
- Кто запускает контейнер (Docker → containerd → runc).
- Что такое OCI-стандарт.
- Как собрать маленький образ через multi-stage.
- Что такое scratch и distroless.
- Как анализировать образ.

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **Overlayfs** | Файловая система Linux, склеивающая несколько директорий в одну. |
| **lowerdir** | Директория со слоями образа (read-only). |
| **upperdir** | Директория с изменениями контейнера (read-write). |
| **merged** | Объединённое представление lowerdir и upperdir. |
| **Copy-up** | Механизм overlayfs: при записи файл копируется из lowerdir в upperdir. |
| **Whiteout** | Маркер в upperdir, говорящий «этот файл удалён». |
| **Docker daemon** | Демон Docker (`dockerd`), принимает команды от CLI. |
| **containerd** | Менеджер контейнеров, управляет жизненным циклом. |
| **containerd-shim** | Процесс-посредник между containerd и runtime. Держит I/O контейнера. |
| **runc** | OCI-runtime, создаёт namespaces и cgroups, запускает процесс, выходит. |
| **OCI** | Open Container Initiative — организация, разрабатывающая стандарты для контейнеров. |
| **Runtime-spec** | Спецификация OCI: как запускать контейнер (config.json). |
| **Image-spec** | Спецификация OCI: формат образов (слои + манифест + конфиг). |
| **Distribution-spec** | Спецификация OCI: API registry. |
| **Multi-stage build** | Dockerfile с несколькими стадиями. Финальный образ содержит только последнюю стадию. |
| **Builder** | Промежуточная стадия сборки. Содержит компилятор и зависимости. |
| **scratch** | Пустой образ (0 байт). Для статических бинарников. |
| **Distroless** | Минимальный образ без shell и пакетного менеджера. |
| **Static linking** | Связывание бинарника со всеми библиотеками на этапе компиляции. |
| **CGO_ENABLED=0** | Отключает CGO, делает Go-бинарник статическим. |
| **dive** | Инструмент для анализа образов. |
| **trivy** | Сканер уязвимостей и секретов в образах. |
| **SBOM** | Software Bill of Materials — список всех пакетов в образе. |

---

## Что мы узнали?

- **Overlayfs** склеивает слои образа (lowerdir) с изменениями контейнера (upperdir) в единое представление (merged).
- **Copy-up** копирует файл из lowerdir в upperdir при записи. **Whiteout** скрывает удалённые файлы, но не удаляет их физически.
- **Цепочка запуска контейнера:** Docker CLI → Docker daemon → containerd → containerd-shim → runc → процесс.
- **OCI-стандарт** определяет три спецификации: runtime, image, distribution. Любой OCI-образ работает в любом OCI-runtime и registry.
- **Multi-stage builds** уменьшают размер образа в 10-50 раз. Финальный образ содержит только последнюю стадию.
- **scratch** — пустой образ для статических бинарников. **distroless** — минимум для runtime.
- **dive** показывает слои, файлы и эффективность образа. **trivy** сканирует уязвимости.

---

## Типичные ошибки

- ❌ **Думать, что удаление файла в контейнере освобождает место.** Whiteout-запись скрывает файл, но он остаётся в слое.
- ❌ **Использовать один этап сборки для Go-приложений.** Образ весит 700 МБ вместо 15 МБ.
- ❌ **Копировать весь проект в финальную стадию.** Копируй только артефакты.
- ❌ **Забыть `CGO_ENABLED=0`** при сборке для scratch. Бинарник будет динамически слинкован и не запустится.
- ❌ **Использовать scratch для приложений с glibc-зависимостями.** Python, Ruby, Node.js не запустятся.
- ❌ **Не копировать CA-сертификаты в scratch/distroless.** HTTPS-запросы будут падать.
- ❌ **Не копировать timezone в scratch/distroless.** Время будет UTC.
- ❌ **Использовать distroless без понимания ограничений.** Не сможешь зайти в контейнер для отладки.
- ❌ **Не анализировать образы перед деплоем.** Может быть майнер, бэкдор, уязвимые пакеты.
- ❌ **Игнорировать `docker history` и `dive`.** Не увидишь, что образ раздут.

---

## Для быстрого повторения

- **Overlayfs:** `lowerdir` (read-only) + `upperdir` (read-write) = `merged`.
- **Copy-up:** при записи файл копируется в `upperdir` целиком.
- **Whiteout:** удаление файла создаёт маркер в `upperdir`, но файл остаётся в `lowerdir`.
- **Цепочка запуска:** Docker CLI → dockerd → containerd → shim → runc → процесс.
- **OCI:** три спецификации (runtime, image, distribution). Стандарт для всего контейнерного мира.
- **Multi-stage:** несколько `FROM`, `COPY --from=builder`, финальный образ = последняя стадия.
- **scratch:** пустой образ, `CGO_ENABLED=0`, копируй CA-сертификаты и timezone вручную.
- **distroless:** минимум для runtime, безопаснее alpine, нет shell.
- **Анализ:** `docker history` (слои), `dive` (эффективность), `trivy` (уязвимости), `syft` (SBOM).
- **Размеры:** ubuntu 78 МБ → alpine 7 МБ → distroless 12 МБ → scratch 8 МБ (для Go + бинарник).

---

## Вопросы для самопроверки

1. Что такое overlayfs? Что такое lowerdir, upperdir, merged?
2. Что такое copy-up? Что происходит с файлом при изменении?
3. Что такое whiteout? Почему удаление файла не уменьшает образ?
4. Опиши цепочку запуска контейнера от `docker run` до процесса. Назови все компоненты.
5. Зачем нужен containerd-shim? Что произойдёт, если его удалить?
6. Что такое OCI? Назови три спецификации.
7. Почему образы Docker работают в Podman и Kubernetes?
8. Что такое multi-stage build? Как он уменьшает размер образа?
9. Чем scratch отличается от distroless? Когда использовать каждый?
10. Что такое CGO_ENABLED=0 и зачем это нужно?
11. Что нужно скопировать в scratch-образ, кроме бинарника? Почему?
12. Как проанализировать образ? Назови три инструмента и что каждый делает.
13. Ты собрал образ на 700 МБ для Go-приложения. Опиши шаги, как уменьшить его до 15 МБ.

---

## Ответы

**1. Overlayfs**

Overlayfs — файловая система Linux, которая склеивает несколько директорий в одну. `lowerdir` — слои образа (read-only). `upperdir` — изменения контейнера (read-write). `merged` — результат: то, что видит контейнер.

**2. Copy-up**

При изменении файла из `lowerdir` overlayfs копирует его целиком в `upperdir`, изменения применяются к копии. Оригинал в `lowerdir` остаётся нетронутым. Поэтому слои образа неизменяемы и разделяются между контейнерами.

**3. Whiteout**

При удалении файла overlayfs создаёт маркер (whiteout) в `upperdir`. Файл в `lowerdir` физически не удаляется — он просто скрыт. Поэтому удаление файла не уменьшает размер образа.

**4. Цепочка запуска**

Docker CLI → Docker daemon (dockerd) → containerd → containerd-shim → runc → процесс. CLI парсит команду и отправляет демону. Демон управляет образами и делегирует containerd. Containerd создаёт snapshot (overlayfs), запускает shim. Shim запускает runc. RunC создаёт namespaces и cgroups, запускает процесс, выходит. Shim держит I/O.

**5. Containerd-shim**

Процесс-посредник между containerd и runtime. Держит stdin/stdout/stderr контейнера, передаёт exit code. Если containerd перезапустится — контейнеры продолжат работать благодаря shim. Если удалить shim — контейнер потеряет I/O.

**6. OCI**

Open Container Initiative — организация под эгидой Linux Foundation. Разрабатывает три спецификации: runtime-spec (как запускать контейнер), image-spec (формат образов), distribution-spec (API registry).

**7. Совместимость**

OCI-стандарт определяет общий формат образов и runtime. Любой OCI-совместимый образ работает в любом OCI-runtime (runc, crun, gVisor) и registry. Docker, Podman, containerd, CRI-O — все совместимы с OCI.

**8. Multi-stage build**

Dockerfile с несколькими `FROM`. Каждая стадия — отдельный образ. Финальный образ содержит только последнюю стадию. `COPY --from=builder` передаёт артефакты из промежуточной стадии в финальную. Размер уменьшается в 10-50 раз.

**9. Scratch vs distroless**

scratch — пустой образ (0 байт). Только бинарник. Для статических бинарников (Go, Rust). distroless — минимальный runtime (libc, CA, timezone). Для приложений, которым нужны эти компоненты. Оба без shell. Distroless безопаснее, но больше (12 МБ vs 8 МБ).

**10. CGO_ENABLED=0**

Отключает CGO — механизм вызова C-кода из Go. Без CGO Go компилирует бинарник статически, без зависимости от libc. Это обязательно для scratch-образа, где нет libc.

**11. Что копировать в scratch**

- Бинарник.
- CA-сертификаты (`/etc/ssl/certs/ca-certificates.crt`) — для HTTPS.
- Timezone (`/usr/share/zoneinfo`) — для правильного времени.
- `/etc/passwd`, `/etc/group` — для non-root пользователей.
- `/etc/resolv.conf` — если нужен кастомный DNS.

**12. Инструменты анализа**

- `docker history` — список слоёв и их размеры.
- `dive` — детальный анализ: файлы в слоях, эффективность, savings.
- `trivy` — сканирование уязвимостей и секретов.
- `docker scout` — встроенный сканер.
- `syft` — SBOM.

**13. Уменьшить образ**

1. Использовать multi-stage build: первая стадия — `golang:1.22-alpine` с компилятором, вторая — `alpine` или `scratch`.
2. В первой стадии: `COPY go.mod go.sum`, `RUN go mod download`, `COPY . .`, `RUN CGO_ENABLED=0 go build -o myapp .`.
3. Во второй стадии: `COPY --from=builder /app/myapp .`.
4. Если нужно HTTPS — скопировать CA-сертификаты.
5. Финальный образ: 8 МБ (scratch) или 15 МБ (alpine).

---

## Куда идти дальше?

Мы разобрали Docker под капотом: overlayfs, containerd, runc, OCI, multi-stage builds, distroless, анализ образов. Теперь ты понимаешь, **как** Docker работает, а не только **что** он делает.

Но мы пока не разобрали **сеть в Docker** и **volumes**. Как контейнеры общаются друг с другом? Как работает bridge network? Как пробросить порт? Что такое volumes и зачем они нужны? И как обеспечить безопасность контейнеров — capabilities, seccomp, AppArmor?

В следующей главе:

- **Сеть в Docker:** bridge, host, none, overlay. Port mapping. DNS между контейнерами.
- **Volumes и bind mounts:** персистентность данных.
- **Безопасность:** capabilities, seccomp, AppArmor, user namespace, rootless.

**Глава 4: Docker — сети, volumes, безопасность.** Погнали. 🚀