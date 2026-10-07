# 📦 Глава 15: Helm — менеджер пакетов для Kubernetes

**Что вы узнаете:**
- Зачем нужен Helm, если есть `kubectl apply`.
- Что такое chart, values, templates, release.
- Как писать Helm-чарты с нуля.
- Как работает шаблонизация и helper-функции.
- Как управлять зависимостями между чартами (subcharts).
- Как делать upgrades и rollbacks.
- Как использовать hooks для миграций.
- Как хранить чарты в OCI-registry.
- Как тестировать чарты.

**После прочтения вы сможете:**
- Написать Helm-чарт для приложения.
- Параметризовать манифесты через values.
- Управлять релизами: install, upgrade, rollback.
- Использовать subcharts для зависимостей.
- Применять hooks для миграций БД.
- Публиковать чарты в OCI-registry.
- Тестировать чарты перед деплоем.

---

## Содержание

- [15.0 Пролог: 50 манифестов для одного приложения](#150-пролог-50-манифестов-для-одного-приложения)
- [15.1 Зачем нужен Helm](#151-зачем-нужен-helm)
- [15.2 Структура чарта](#152-структура-чарта)
- [15.3 Шаблонизация: values и templates](#153-шаблонизация-values-и-templates)
- [15.4 Встроенные объекты и функции](#154-встроенные-объекты-и-функции)
- [15.5 Helper-функции и _helpers.tpl](#155-helper-функции-и-_helperstpl)
- [15.6 Управление релизами: install, upgrade, rollback](#156-управление-релизами-install-upgrade-rollback)
- [15.7 Зависимости: subcharts и Condition](#157-зависимости-subcharts-и-condition)
- [15.8 Hooks: миграции и другие задачи](#158-hooks-миграции-и-другие-задачи)
- [15.9 OCI-registry: чарты как артефакты](#159-oci-registry-чарты-как-артефакты)
- [15.10 Тестирование и отладка чартов](#1510-тестирование-и-отладка-чартов)
- [15.11 Helm vs Kustomize](#1511-helm-vs-kustomize)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 15.0 Пролог: 50 манифестов для одного приложения

Ты деплоишь приложение в Kubernetes. У тебя есть:

- **Deployment** для API.
- **Service** для API.
- **ConfigMap** для конфигурации.
- **Secret** для секретов.
- **Ingress** для HTTP-трафика.
- **Deployment** для воркера.
- **HorizontalPodAutoscaler** для API.
- **PodDisruptionBudget**.
- **ServiceAccount**.
- **NetworkPolicy**.

**10 манифестов.** А теперь представь: у тебя **три окружения** — dev, staging, production. И **пять микросервисов**. Это **150 манифестов**.

И каждый надо обновлять вручную. Меняешь версию образа — правишь в трёх местах. Меняешь replicas для prod — ищешь, где это. Хочешь откатиться — вручную применяешь старые YAML.

**Это не масштабируется.**

**Helm решает эту проблему.** Ты пишешь **один чарт** с параметрами. Для каждого окружения — свой **values.yaml** с параметрами. Одна команда `helm upgrade` — и все манифесты применяются с нужными значениями.

**Плюс:**

- **История релизов** — можно откатиться на предыдущую версию.
- **Rollback** — одной командой.
- **Templating** — параметризация через Go templates.
- **Зависимости** — PostgreSQL, Redis как subcharts.
- **Hooks** — миграции БД, smoke tests.

В этой главе мы разберём Helm от основ до продвинутых техник. Ты научишься писать чарты, управлять релизами, использовать зависимости.

Это — стандарт де-факто для упаковки приложений в Kubernetes. Без Helm ты будешь тонуть в YAML.

---

## 15.1 Зачем нужен Helm

### 🔌 Проблема: kubectl apply не масштабируется

`kubectl apply -f` — отличная команда для **одного** манифеста. Но когда манифестов **десятки** и они отличаются **между окружениями**, начинается хаос.

**Проблемы:**

1. **Дублирование.** Один и тот же Deployment для dev и prod отличается только `replicas` и `image`. Но ты копируешь весь манифест.

2. **Нет параметризации.** Хочешь поменять версию образа — ищешь во всех файлах.

3. **Нет истории.** Что задеплоено в prod? Какая версия? Когда? Никак не узнать.

4. **Нет rollback.** Откатиться на предыдущую версию — вручную применять старые YAML.

5. **Нет управления зависимостями.** Приложение требует PostgreSQL. Ты должен установить его отдельно.

6. **Нет упаковки.** Как поделиться своим приложением с другими? Копировать YAML-файлы.

### 📦 Что такое Helm

**Helm** — это **менеджер пакетов** для Kubernetes. Аналогия:

| Docker | Helm |
|:---|:---|
| Docker image | Helm chart |
| Docker container | Helm release |
| Dockerfile | Chart templates |
| Docker Hub | Helm repository |
| `docker run` | `helm install` |

**Основные понятия:**

| Понятие | Что означает |
|:---|:---|
| **Chart** | Пакет с манифестами и параметрами |
| **Release** | Установленный экземпляр чарта |
| **Values** | Параметры для чарта |
| **Templates** | Шаблоны манифестов (Go templates) |
| **Repository** | Хранилище чартов |
| **Revision** | Версия релиза (для rollback) |

### 🎯 Что даёт Helm

**1. Параметризация.**

Один чарт, разные values для разных окружений:

```yaml
# values-dev.yaml
replicas: 1
image:
  tag: latest

# values-prod.yaml
replicas: 5
image:
  tag: v1.2.3
```

**2. Упаковка.**

Чарт — это **архив** (`.tgz`) с манифестами. Легко делиться, версионировать, публиковать.

**3. История и rollback.**

Каждый `helm upgrade` создаёт **новую ревизию**. Откат — `helm rollback myapp 1`.

**4. Зависимости.**

Чарт может зависеть от других чартов:

```yaml
dependencies:
  - name: postgresql
    version: 12.x.x
    repository: https://charts.bitnami.com/bitnami
```

Одна команда — установит приложение + PostgreSQL.

**5. Hooks.**

Специальные манифесты, которые выполняются **до** или **после** деплоя:

- Миграции БД.
- Smoke tests.
- Backup.

**6. Templating.**

Go templates + Sprig functions. Мощная параметризация.

### 🎯 Версии Helm

- **Helm 2** — устарел. Два компонента: `helm` (CLI) + `tiller` (в кластере). Tiller имел полные права — небезопасно.
- **Helm 3** — текущая версия. **Без tiller.** Использует Kubernetes API напрямую, RBAC.

**Мы используем Helm 3.**

### 🎯 Установка

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Проверка
helm version
# version.BuildInfo{Version:"v3.14.0", ...}
```

### 🔬 Практика: первое знакомство

```bash
# 1. Добавить репозиторий
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 2. Поиск чартов
helm search repo postgresql

# 3. Посмотреть чарт без установки
helm show chart bitnami/postgresql
helm show values bitnami/postgresql

# 4. Установить
helm install my-postgres bitnami/postgresql

# 5. Список релизов
helm list

# 6. Удалить
helm uninstall my-postgres
```

### 💡 Практика: что важно понять про Helm

**✅ ОБЯЗАТЕЛЬНО:**

1. **Helm — это параметризация.** Один чарт, разные values для окружений.
2. **Release — это установленный чарт.** Может быть много releases из одного чарта.
3. **Helm 3 без tiller.** Использует RBAC.

**👍 СТОИТ:**

4. **Использовать Helm для приложений** с 5+ манифестами.
5. **Хранить чарты в Git** (или OCI-registry).

**❌ НЕ ДЕЛАЙ:**

6. **Не используй Helm для одного манифеста.** `kubectl apply` проще.
7. **Не используй Helm 2.** Tiller — небезопасно.

### Где мы сейчас

Мы разобрали, зачем нужен Helm. Теперь — **структура чарта**.

---

## 15.2 Структура чарта

### 🔌 Проблема: как организовать чарт

Чарт — это **директория** с определённой структурой. Разберём каждый файл.

### 📊 Стандартная структура

```
myapp/
├── Chart.yaml              # метаданные чарта
├── values.yaml             # значения по умолчанию
├── values.schema.json      # JSON Schema для валидации (опционально)
├── README.md               # документация
├── LICENSE                 # лицензия
├── .helmignore             # что не включать в пакет
├── charts/                 # зависимости (subcharts)
│   └── postgresql/
├── templates/              # шаблоны манифестов
│   ├── NOTES.txt           # сообщение после установки
│   ├── _helpers.tpl        # helper-функции
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   └── tests/
│       └── test-connection.yaml
└── crds/                   # CRD (опционально)
```

### 📄 Chart.yaml

**Метаданные чарта:**

```yaml
apiVersion: v2                    # Helm 3
name: myapp
description: My Go application
type: application                 # application или library
version: 1.2.3                    # версия чарта (semver)
appVersion: "1.0.0"               # версия приложения
kubeVersion: ">=1.25.0"           # требуемая версия K8s
keywords:
  - go
  - api
home: https://example.com
sources:
  - https://github.com/myorg/myapp
maintainers:
  - name: John Doe
    email: john@example.com
icon: https://example.com/icon.png
annotations:
  category: Application
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

**Поля:**

| Поле | Что означает |
|:---|:---|
| `apiVersion` | `v2` для Helm 3 |
| `name` | Имя чарта |
| `version` | Версия чарта (semver) |
| `appVersion` | Версия приложения |
| `type` | application (деплоится) или library (только helpers) |
| `kubeVersion` | Требуемая версия K8s |
| `dependencies` | Зависимости от других чартов |

**Важно:** `version` — это версия **чарта**, `appVersion` — версия **приложения**. Они независимы. Можно обновить чарт (изменить шаблон) без обновления приложения.

### 📄 values.yaml

**Значения по умолчанию:**

```yaml
# Количество реплик
replicaCount: 3

# Образ
image:
  repository: myregistry.com/myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

# Порт
service:
  type: ClusterIP
  port: 80
  targetPort: 8080

# Ресурсы
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

# ConfigMap
config:
  logLevel: info
  database:
    host: postgres
    port: 5432

# Ingress
ingress:
  enabled: false
  className: nginx
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix

# PostgreSQL (subchart)
postgresql:
  enabled: true
  auth:
    database: myapp
    username: myapp
    password: ""  # передать через --set или secret
```

### 📄 templates/

**Шаблоны манифестов.** Используют Go templates + Sprig functions.

**Пример deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### 📄 _helpers.tpl

**Helper-функции** для переиспользования:

```
{{/*
Имя чарта + имя релиза.
*/}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels.
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels.
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### 📄 .helmignore

**Что не включать в пакет:**

```
.DS_Store
.git/
.gitignore
*.swp
*.bak
.idea/
.vscode/
*.tgz
```

### 📄 NOTES.txt

**Сообщение после установки:**

```
Thank you for installing {{ .Chart.Name }}!

Your release is named {{ .Release.Name }}.

To access your application:
{{- if .Values.ingress.enabled }}
  https://{{ (index .Values.ingress.hosts 0).host }}
{{- else }}
  kubectl port-forward svc/{{ include "myapp.fullname" . }} 8080:{{ .Values.service.port }}
{{- end }}

To check the status:
  helm status {{ .Release.Name }}
```

### 🔬 Практика: создание чарта

```bash
# 1. Создать чарт
helm create myapp

# 2. Посмотреть структуру
tree myapp/
# myapp/
# ├── Chart.yaml
# ├── values.yaml
# ├── charts/
# └── templates/
#     ├── NOTES.txt
#     ├── _helpers.tpl
#     ├── deployment.yaml
#     ├── service.yaml
#     ├── serviceaccount.yaml
#     ├── hpa.yaml
#     ├── ingress.yaml
#     └── tests/
#         └── test-connection.yaml

# 3. Заполнить Chart.yaml
cat myapp/Chart.yaml

# 4. Заполнить values.yaml
cat myapp/values.yaml

# 5. Проверить шаблоны (без установки)
helm template my-release myapp/

# 6. Lint
helm lint myapp/
```

### 💡 Практика: как правильно организовать чарт

**✅ ОБЯЗАТЕЛЬНО:**

1. **`Chart.yaml` с version и appVersion.**
2. **`values.yaml` с разумными defaults.**
3. **`templates/` с манифестами.**
4. **`_helpers.tpl` для переиспользуемых функций.**
5. **`NOTES.txt`** — что делать после установки.

**👍 СТОИТ:**

6. **`.helmignore`** — исключить лишнее.
7. **`values.schema.json`** — валидация values.
8. **`README.md`** — документация чарта.
9. **`templates/tests/`** — smoke tests.

**❌ НЕ ДЕЛАЙ:**

10. **Не хардкодь значения в templates.** Всё через values.
11. **Не дублируй labels.** Используй helpers.
12. **Не забывай про `| nindent`.** Без него YAML сломается.

### Где мы сейчас

Мы разобрали структуру чарта. Теперь — **шаблонизация**.

---

## 15.3 Шаблонизация: values и templates

### 🔌 Проблема: как параметризовать манифесты

Templates — это Go templates. Они позволяют вставлять значения из values.yaml в манифесты.

### 📊 Синтаксис Go templates

**Основные конструкции:**

| Синтаксис | Что означает |
|:---|:---|
| `{{ .Values.key }}` | Значение из values.yaml |
| `{{ .Chart.Name }}` | Имя чарта из Chart.yaml |
| `{{ .Release.Name }}` | Имя релиза |
| `{{ .Release.Namespace }}` | Namespace |
| `{{ include "template" . }}` | Включить helper |
| `{{ if .Values.enabled }}...{{ end }}` | Условие |
| `{{ range .Values.items }}...{{ end }}` | Цикл |
| `{{- ... }}` | Убрать пробелы слева |
| `{{ ... -}}` | Убрать пробелы справа |
| `{{ ... | nindent 4 }}` | Отступ 4 пробела |

### 🎯 Базовые примеры

**Значение из values:**

```yaml
replicas: {{ .Values.replicaCount }}
```

**Строка с кавычками:**

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

**Условие:**

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}
```

**Цикл:**

```yaml
{{- range .Values.ingress.hosts }}
- host: {{ .host }}
  http:
    paths:
      {{- range .paths }}
      - path: {{ .path }}
        pathType: {{ .pathType }}
      {{- end }}
{{- end }}
```

**Вложенные значения:**

```yaml
host: {{ .Values.config.database.host }}
port: {{ .Values.config.database.port }}
```

### 🎯 Функции Sprig

**Helm включает Sprig** — библиотеку функций для Go templates.

**Строки:**

| Функция | Что делает |
|:---|:---|
| `upper` | Верхний регистр |
| `lower` | Нижний регистр |
| `title` | Title Case |
| `trim` | Убрать пробелы |
| `trunc 63` | Обрезать до 63 символов |
| `printf` | Форматирование |
| `default "x"` | Значение по умолчанию |
| `quote` | Добавить кавычки |
| `squote` | Одинарные кавычки |

**Примеры:**

```yaml
name: {{ .Values.name | lower | trunc 63 }}
env: {{ .Values.env | default "production" }}
log: {{ printf "%s-%s" .Release.Name .Chart.Name }}
```

**Числа:**

| Функция | Что делает |
|:---|:---|
| `add` | Сложение |
| `sub` | Вычитание |
| `mul` | Умножение |
| `div` | Деление |
| `mod` | Остаток |

**Списки:**

| Функция | Что делает |
|:---|:---|
| `first` | Первый элемент |
| `last` | Последний элемент |
| `join ","` | Объединить |
| `splitList ","` | Разделить |
| `has "x"` | Проверить наличие |

**Пример:**

```yaml
hosts: {{ .Values.ingress.hosts | join "," }}
```

**Словари:**

| Функция | Что делает |
|:---|:---|
| `keys` | Ключи |
| `values` | Значения |
| `hasKey` | Проверить ключ |
| `get` | Получить значение |

**YAML/JSON:**

| Функция | Что делает |
|:---|:---|
| `toYaml` | В YAML |
| `fromYaml` | Из YAML |
| `toJson` | В JSON |
| `fromJson` | Из JSON |

**Пример:**

```yaml
resources:
  {{- toYaml .Values.resources | nindent 2 }}
```

### 🎯 `nindent` и `indent`

**Проблема:** когда ты вставляешь многострочное значение, нужно соблюсти отступы.

**Решение:** `nindent` (newline + indent) и `indent`.

```yaml
labels:
  {{- include "myapp.labels" . | nindent 2 }}
```

**Что произойдёт:**

- `include "myapp.labels" .` — вернёт многострочную строку.
- `nindent 2` — добавит перевод строки и отступ 2 пробела к каждой строке.

**Результат:**

```yaml
labels:
  app: myapp
  version: v1.0.0
```

**Без `nindent`:**

```yaml
labels:
  app: myapp
version: v1.0.0    ← сломается!
```

### 🎯 `{{-` и `-}}`

**Проблема:** Go templates оставляют пустые строки и пробелы.

**Решение:** `{{-` убирает пробелы **слева**, `-}}` — **справа**.

```yaml
metadata:
  name: myapp
  {{- if .Values.annotations }}
  annotations:
    {{- toYaml .Values.annotations | nindent 4 }}
  {{- end }}
```

**Что произойдёт:** пустые строки от `{{ if }}` и `{{ end }}` не появятся в итоговом YAML.

### 🎯 Values: переопределение

**Порядок приоритета (от низшего к высшему):**

1. `values.yaml` в чарте.
2. Values из родительского чарта (для subcharts).
3. `--values` / `-f` (файлы values).
4. `--set` (аргументы командной строки).

**Пример:**

```bash
# values.yaml в чарте: replicaCount: 1

# Через файл
helm install myapp ./myapp -f values-prod.yaml
# values-prod.yaml: replicaCount: 5

# Через --set
helm install myapp ./myapp --set replicaCount=10

# --set переопределяет -f
helm install myapp ./myapp -f values-prod.yaml --set replicaCount=10
# Итог: replicaCount=10
```

### 🎯 `--set` синтаксис

```bash
# Простое значение
--set replicaCount=3

# Вложенное
--set image.tag=v1.0.0
--set config.database.host=postgres

# Массив
--set ingress.hosts[0].host=example.com
--set ingress.hosts[0].paths[0].path=/

# Список через запятую
--set "list={a,b,c}"

# Экранирование точки в ключе
--set "annotations.pod\.annotation=value"

# Из файла
--set-file config=./config.yaml
```

### 🎯 `values.schema.json`

**JSON Schema для валидации:**

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["image", "service"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1
    },
    "image": {
      "type": "object",
      "required": ["repository", "tag"],
      "properties": {
        "repository": {
          "type": "string"
        },
        "tag": {
          "type": "string"
        }
      }
    },
    "service": {
      "type": "object",
      "properties": {
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        }
      }
    }
  }
}
```

**Что даёт:** Helm валидирует values при установке. Ошибки — сразу, а не в процессе.

### 🎯 Мини-пример полного чарта

**Chart.yaml:**

```yaml
apiVersion: v2
name: myapp
version: 1.0.0
appVersion: "1.0.0"
```

**values.yaml:**

```yaml
replicaCount: 3
image:
  repository: myregistry.com/myapp
  tag: "1.0.0"
config:
  logLevel: info
  database:
    host: postgres
    port: 5432
```

**templates/configmap.yaml:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
data:
  LOG_LEVEL: {{ .Values.config.logLevel | quote }}
  DATABASE_HOST: {{ .Values.config.database.host | quote }}
  DATABASE_PORT: {{ .Values.config.database.port | quote }}
```

**templates/deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          envFrom:
            - configMapRef:
                name: {{ include "myapp.fullname" . }}
          ports:
            - containerPort: 8080
```

**templates/_helpers.tpl:**

```
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{- define "myapp.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
{{ include "myapp.selectorLabels" . }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### 🔬 Практика: шаблонизация

```bash
# 1. Создать чарт
helm create demo
cd demo

# 2. Отредактировать values.yaml
cat > values.yaml <<EOF
replicaCount: 3
image:
  repository: nginx
  tag: "1.25"
config:
  logLevel: info
EOF

# 3. Отредактировать templates/configmap.yaml
cat > templates/configmap.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "demo.fullname" . }}
data:
  LOG_LEVEL: {{ .Values.config.logLevel | quote }}
EOF

# 4. Отрендерить шаблоны
helm template my-release .

# 5. Lint
helm lint .

# 6. Установить
helm install my-release . --dry-run  # проверка
helm install my-release .            # реально

# 7. Посмотреть
helm list
kubectl get configmap
kubectl get configmap -o yaml | grep LOG_LEVEL
```

### 💡 Практика: как правильно шаблонизировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Все значения — через `.Values`.** Никаких хардкодов.
2. **Использовать `quote` для строк:**
   ```yaml
   value: {{ .Values.string | quote }}
   ```
3. **Использовать `nindent` для многострочных значений:**
   ```yaml
   {{- toYaml .Values.labels | nindent 4 }}
   ```

**👍 СТОИТ:**

4. **`values.schema.json`** для валидации.
5. **Helpers** для повторяющихся фрагментов.
6. **`default`** для значений по умолчанию:
   ```yaml
   env: {{ .Values.env | default "production" }}
   ```

**❌ НЕ ДЕЛАЙ:**

7. **Не забывай про `{{-` и `-}}`.** Пустые строки ломают YAML.
8. **Не используй `--set` для сложных значений.** Лучше файл values.
9. **Не дублируй логику.** Используй helpers.

### Где мы сейчас

Мы разобрали шаблонизацию. Теперь — **встроенные объекты и функции**.

---

## 15.4 Встроенные объекты и функции

### 🔌 Проблема: что доступно в шаблоне

В шаблоне доступны не только `.Values` и `.Chart`. Есть много других объектов.

### 📊 Встроенные объекты

| Объект | Что содержит |
|:---|:---|
| `.Values` | Значения из values.yaml |
| `.Chart` | Метаданные из Chart.yaml |
| `.Release` | Информация о релизе |
| `.Files` | Файлы чарта (не templates) |
| `.Capabilities` | Возможности кластера |
| `.Template` | Информация о текущем шаблоне |

### 🎯 `.Chart`

**Метаданные из Chart.yaml:**

| Поле | Что содержит |
|:---|:---|
| `.Chart.Name` | Имя чарта |
| `.Chart.Version` | Версия чарта |
| `.Chart.AppVersion` | Версия приложения |
| `.Chart.Description` | Описание |
| `.Chart.Annotations` | Аннотации |

**Пример:**

```yaml
labels:
  helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
  app.kubernetes.io/version: {{ .Chart.AppVersion }}
```

### 🎯 `.Release`

**Информация о релизе:**

| Поле | Что содержит |
|:---|:---|
| `.Release.Name` | Имя релиза |
| `.Release.Namespace` | Namespace |
| `.Release.IsInstall` | true, если install |
| `.Release.IsUpgrade` | true, если upgrade |
| `.Release.Revision` | Номер ревизии |
| `.Release.Service` | Всегда "Helm" |

**Пример:**

```yaml
metadata:
  name: {{ include "myapp.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    app.kubernetes.io/instance: {{ .Release.Name }}
```

**Условная логика:**

```yaml
{{- if .Release.IsInstall }}
# Только при install
{{- end }}

{{- if .Release.IsUpgrade }}
# Только при upgrade
{{- end }}
```

### 🎯 `.Files`

**Доступ к файлам чарта** (не из templates).

**Пример:**

```
myapp/
├── files/
│   ├── config.yaml
│   └── init.sql
└── templates/
    └── configmap.yaml
```

**В шаблоне:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}
data:
  config.yaml: |
    {{- .Files.Get "files/config.yaml" | nindent 4 }}
```

**Методы:**

| Метод | Что делает |
|:---|:---|
| `.Files.Get "path"` | Получить содержимое файла |
| `.Files.GetBytes "path"` | Получить как []byte |
| `.Files.Glob "pattern"` | Найти файлы по glob |
| `.Files.Lines "path"` | Построчно |
| `.Files.AsConfig` | Как ConfigMap data |
| `.Files.AsSecrets` | Как Secret data |

**Пример с glob:**

```yaml
{{- range $path, $_ := .Files.Glob "files/*.yaml" }}
{{ $path }}: |
  {{- $.Files.Get $path | nindent 4 }}
{{- end }}
```

### 🎯 `.Capabilities`

**Возможности кластера:**

| Поле | Что содержит |
|:---|:---|
| `.Capabilities.KubeVersion` | Версия K8s |
| `.Capabilities.APIVersions` | Доступные API |
| `.Capabilities.APIVersions.Has "..."` | Проверка API |

**Пример:**

```yaml
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1/Ingress" }}
apiVersion: networking.k8s.io/v1
kind: Ingress
{{- else }}
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
{{- end }}
```

**Версия K8s:**

```yaml
{{- if semverCompare ">=1.25.0" .Capabilities.KubeVersion.Version }}
# Использовать новую фичу
{{- end }}
```

### 🎯 Функции для работы с датами

| Функция | Что делает |
|:---|:---|
| `now` | Текущее время |
| `date "2006-01-02"` | Форматирование |
| `dateInZone` | В часовом поясе |
| `htmlDate` | HTML-формат |
| `ago` | Время назад |

**Пример:**

```yaml
annotations:
  timestamp: {{ now | date "2006-01-02T15:04:05Z07:00" | quote }}
```

### 🎯 Хеширование

**Для checksum ConfigMap:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

**Что произойдёт:** при изменении ConfigMap меняется checksum → Deployment перезапускает Pod'ы.

### 🎯 `required` и `fail`

**`required` — обязательное значение:**

```yaml
image:
  tag: {{ required "image.tag is required" .Values.image.tag }}
```

**Если не задано — Helm выдаст ошибку.**

**`fail` — явная ошибка:**

```yaml
{{- if not .Values.image.repository }}
{{- fail "image.repository is required" }}
{{- end }}
```

### 🎯 `lookup` — запрос к API

**`lookup` — получить существующий ресурс:**

```yaml
{{- $secret := lookup "v1" "Secret" .Release.Namespace "my-secret" }}
{{- if $secret }}
# Secret существует — использовать его значение
{{- else }}
# Secret не существует — создать новый
{{- end }}
```

**Пример: сохранение пароля между upgrades:**

```yaml
{{- $secret := lookup "v1" "Secret" .Release.Namespace (include "myapp.fullname" .) }}
{{- $password := "" }}
{{- if $secret }}
  {{- $password = index $secret.data "password" | b64dec }}
{{- else }}
  {{- $password = randAlphaNum 32 }}
{{- end }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapp.fullname" . }}
type: Opaque
data:
  password: {{ $password | b64enc }}
```

**Что произойдёт:**

- При первом install: пароль генерируется.
- При upgrade: пароль читается из существующего Secret.
- Не меняется между релизами.

**Важно:** `lookup` не работает в `helm template` (только в реальном кластере).

### 🔬 Практика: встроенные объекты

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "demo.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
data:
  release-name: {{ .Release.Name | quote }}
  release-revision: {{ .Release.Revision | quote }}
  kube-version: {{ .Capabilities.KubeVersion.Version | quote }}
  current-time: {{ now | date "2006-01-02T15:04:05Z" | quote }}
  config.yaml: |
    {{- .Files.Get "files/config.yaml" | nindent 4 }}
```

### 💡 Практика: как использовать встроенные объекты

**✅ ОБЯЗАТЕЛЬНО:**

1. **`.Release.Name`** для уникальных имён.
2. **`.Release.Namespace`** для namespace.
3. **`.Chart.Version` / `.Chart.AppVersion`** для labels.
4. **`.Capabilities.KubeVersion`** для условной логики.

**👍 СТОИТ:**

5. **`.Files`** для конфигов из файлов.
6. **`lookup`** для сохранения секретов между upgrades.
7. **`required`** для обязательных значений.

**❌ НЕ ДЕЛАЙ:**

8. **Не хардкодь namespace.** Используй `.Release.Namespace`.
9. **Не используй `lookup` в `helm template`.** Не работает.

### Где мы сейчас

Мы разобрали встроенные объекты. Теперь — **helper-функции**.

---

## 15.5 Helper-функции и _helpers.tpl

### 🔌 Проблема: дублирование в шаблонах

В каждом шаблоне нужно повторять одно и то же:

- Labels.
- Selector labels.
- Имя ресурса.
- Namespace.

**Решение:** helper-функции в `_helpers.tpl`.

### 📊 Что такое helper

**Helper** — это шаблон, определённый через `define`. Его можно **включать** в другие шаблоны через `include`.

**Синтаксис:**

```
{{- define "myapp.fullname" -}}
...содержимое...
{{- end }}
```

**Использование:**

```yaml
name: {{ include "myapp.fullname" . }}
```

### 🎯 Стандартные helpers

**`helm create`** создаёт базовые helpers. Разберём их.

**`_helpers.tpl`:**

```
{{/*
Имя чарта.
*/}}
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Полное имя (release + chart).
*/}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Имя чарта с версией.
*/}}
{{- define "myapp.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels.
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels.
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
ServiceAccount name.
*/}}
{{- define "myapp.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "myapp.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### 🎯 Как использовать

**В deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "myapp.serviceAccountName" . }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

**Ключевое:** `include "myapp.labels" .` — второй аргумент `.` передаёт контекст. Без него helper не увидит `.Values`.

### 🎯 Почему `include`, а не `template`

**`template`** — встроенная функция Go templates. **Проблема:** не возвращает значение, не работает с `nindent`.

**`include`** — функция Helm. **Возвращает строку.** Работает с `nindent`.

```yaml
# ❌ Плохо
labels:
  {{ template "myapp.labels" . }}

# ✅ Хорошо
labels:
  {{- include "myapp.labels" . | nindent 4 }}
```

### 🎯 Свои helpers

**Пример: helper для image:**

```
{{- define "myapp.image" -}}
{{- printf "%s:%s" .Values.image.repository (.Values.image.tag | default .Chart.AppVersion) }}
{{- end }}
```

**Использование:**

```yaml
image: {{ include "myapp.image" . }}
```

**Пример: helper для env:**

```
{{- define "myapp.env" -}}
{{- range $key, $value := .Values.env }}
- name: {{ $key }}
  value: {{ $value | quote }}
{{- end }}
{{- end }}
```

**Использование:**

```yaml
env:
  {{- include "myapp.env" . | nindent 2 }}
```

### 🎯 Параметризация helpers

**Helper может принимать аргументы:**

```
{{- define "myapp.env" -}}
{{- range $key, $value := . -}}
- name: {{ $key }}
  value: {{ $value | quote }}
{{- end }}
{{- end }}
```

**Использование:**

```yaml
env:
  {{- include "myapp.env" .Values.env | nindent 2 }}
```

Здесь `.` внутри helper — это `.Values.env` (то, что передали).

### 🎯 Парсинг имени для секретов

**Паттерн:** helpers для имён Secret/ConfigMap.

```
{{- define "myapp.secretName" -}}
{{- if .Values.existingSecret }}
{{- .Values.existingSecret }}
{{- else }}
{{- include "myapp.fullname" . }}
{{- end }}
{{- end }}
```

**Использование:**

```yaml
env:
  - name: DATABASE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: {{ include "myapp.secretName" . }}
        key: password
```

**Что даёт:** можно использовать существующий Secret или создать свой.

### 🔬 Практика: helpers

```bash
# 1. Создать чарт
helm create demo
cd demo

# 2. Заменить _helpers.tpl
cat > templates/_helpers.tpl <<'EOF'
{{- define "demo.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "demo.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name (include "demo.name" .) | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{- define "demo.labels" -}}
app.kubernetes.io/name: {{ include "demo.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "demo.image" -}}
{{- printf "%s:%s" .Values.image.repository (.Values.image.tag | default .Chart.AppVersion) }}
{{- end }}
EOF

# 3. Использовать в configmap
cat > templates/configmap.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "demo.fullname" . }}
  labels:
    {{- include "demo.labels" . | nindent 4 }}
data:
  image: {{ include "demo.image" . | quote }}
EOF

# 4. Отрендерить
helm template my-release .
```

### 💡 Практика: как правильно писать helpers

**✅ ОБЯЗАТЕЛЬНО:**

1. **Использовать `include`, не `template`.**
2. **Передавать `.` в `include`** для доступа к `.Values`.
3. **Использовать `nindent`** для многострочных helpers.
4. **`trunc 63`** для имён (ограничение K8s).

**👍 СТОИТ:**

5. **Свои helpers для повторяющихся фрагментов.**
6. **Helpers с параметрами** для гибкости.
7. **`existingSecret` паттерн** для секретов.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй `template`** — не работает с `nindent`.
9. **Не забывай про `{{-` и `-}}`** в helpers.
10. **Не делай слишком сложные helpers.** Читаемость важнее.

### Где мы сейчас

Мы разобрали helpers. Теперь — **управление релизами**.

---

## 15.6 Управление релизами: install, upgrade, rollback

### 🔌 Проблема: как обновлять и откатывать

Приложение задеплоено. Ты хочешь:

- **Обновить** версию.
- **Изменить** конфигурацию.
- **Откатиться** на предыдущую версию.

**Решение:** релизы Helm.

### 📊 Что такое релиз

**Release** — это **установленный экземпляр чарта** с определёнными values.

**Один чарт** может быть установлен **несколько раз** с разными релизами:

```bash
helm install myapp-dev ./myapp -f values-dev.yaml
helm install myapp-prod ./myapp -f values-prod.yaml
```

**Два релиза** одного чарта в одном кластере.

### 🎯 `helm install`

**Установка чарта:**

```bash
helm install myapp ./myapp
```

**Что произойдёт:**

1. Helm отрендерит шаблоны с values.
2. Применит манифесты к кластеру.
3. Создаст Secret `sh.helm.release.v1.myapp.v1` с метаданными релиза.

**С ожиданием готовности:**

```bash
helm install myapp ./myapp --wait --timeout 5m
```

**Что даёт:**

- `--wait` — ждёт, пока все ресурсы станут Ready.
- `--timeout` — максимальное время ожидания.

**Dry-run:**

```bash
helm install myapp ./myapp --dry-run
```

**Показывает, что будет применено, но не применяет.**

**С namespace:**

```bash
helm install myapp ./myapp --namespace production --create-namespace
```

### 🎯 `helm upgrade`

**Обновление релиза:**

```bash
helm upgrade myapp ./myapp
```

**Что произойдёт:**

1. Helm отрендерит шаблоны с новыми values.
2. Сравнит с текущим состоянием.
3. Применит изменения.
4. Создаст новую ревизию (`v2`).

**С установкой, если не существует:**

```bash
helm upgrade --install myapp ./myapp
```

**Что даёт:** если релиз не существует — установит, если существует — обновит. Удобно для CI/CD.

**С новыми values:**

```bash
helm upgrade myapp ./myapp -f values-prod.yaml --set image.tag=v1.2.3
```

**Atomic:**

```bash
helm upgrade myapp ./myapp --atomic --timeout 5m
```

**Что даёт:**

- Если upgrade упал — автоматический rollback на предыдущую версию.
- **Важно для production.**

**С ожиданием:**

```bash
helm upgrade myapp ./myapp --wait --timeout 5m
```

### 🎯 `helm rollback`

**Откат на предыдущую версию:**

```bash
# Откат на предыдущую ревизию
helm rollback myapp

# Откат на конкретную ревизию
helm rollback myapp 1
```

**Что произойдёт:**

1. Helm применит манифесты ревизии 1.
2. Создаст новую ревизию с номером текущая + 1 (не перезаписывает историю).

**Пример:**

```
Ревизия 1: v1.0.0 (оригинал)
Ревизия 2: v1.1.0 (upgrade)
Ревизия 3: v1.2.0 (upgrade, сломалось)

helm rollback myapp 1
# Создаст ревизию 4 = копия ревизии 1
```

### 🎯 `helm history`

**История релиза:**

```bash
helm history myapp
# REVISION  UPDATED                   STATUS      CHART         APP VERSION  DESCRIPTION
# 1         Mon Jan 15 10:00:00 2026  superseded  myapp-1.0.0   1.0.0        Install complete
# 2         Mon Jan 15 11:00:00 2026  superseded  myapp-1.1.0   1.1.0        Upgrade complete
# 3         Mon Jan 15 12:00:00 2026  deployed    myapp-1.2.0   1.2.0        Upgrade complete
```

**Статусы:**

| Статус | Что означает |
|:---|:---|
| `deployed` | Текущая версия |
| `superseded` | Заменена новой версией |
| `failed` | Ошибка при деплое |
| `uninstalled` | Удалена |

### 🎯 `helm list`

**Список релизов:**

```bash
helm list
# NAME    NAMESPACE  REVISION  UPDATED                   STATUS    CHART         APP VERSION
# myapp   default    3         Mon Jan 15 12:00:00 2026  deployed  myapp-1.2.0   1.2.0

# Все namespace
helm list --all-namespaces

# Включая failed
helm list --all

# Только failed
helm list --failed
```

### 🎯 `helm status`

**Статус релиза:**

```bash
helm status myapp
# NAME: myapp
# LAST DEPLOYED: Mon Jan 15 12:00:00 2026
# NAMESPACE: default
# STATUS: deployed
# REVISION: 3
# ...
```

### 🎯 `helm uninstall`

**Удаление релиза:**

```bash
helm uninstall myapp
```

**Что произойдёт:** все ресурсы релиза будут удалены.

**Сохранить историю:**

```bash
helm uninstall myapp --keep-history
```

**Что даёт:** можно посмотреть историю через `helm history --all`.

### 🎯 `helm get`

**Получить информацию о релизе:**

```bash
# Values, с которыми установлен
helm get values myapp

# Все values (включая defaults)
helm get values myapp --all

# Манифесты
helm get manifest myapp

# Notes
helm get notes myapp

# Hooks
helm get hooks myapp
```

### 🎯 `helm diff` (плагин)

**Что изменится при upgrade:**

```bash
# Установить плагин
helm plugin install https://github.com/databus23/helm-diff

# Diff
helm diff upgrade myapp ./myapp -f values-prod.yaml
```

**Что показывает:**

- Какие ресурсы будут созданы.
- Какие изменены (с diff).
- Какие удалены.

**Критически важно для production.** Всегда смотри diff перед upgrade.

### 🔬 Практика: релизы

```bash
# 1. Создать чарт
helm create demo

# 2. Установить
helm install my-release ./demo

# 3. Посмотреть
helm list
helm history my-release

# 4. Обновить (изменить replicas)
helm upgrade my-release ./demo --set replicaCount=5

# 5. История
helm history my-release
# REVISION  STATUS      DESCRIPTION
# 1         superseded  Install complete
# 2         deployed    Upgrade complete

# 6. Откатить
helm rollback my-release 1

# 7. История
helm history my-release
# REVISION  STATUS      DESCRIPTION
# 1         superseded  Install complete
# 2         superseded  Upgrade complete
# 3         deployed    Rollback to 1

# 8. Удалить
helm uninstall my-release
```

### 💡 Практика: как правильно управлять релизами

**✅ ОБЯЗАТЕЛЬНО:**

1. **`--atomic` для production.** Автоматический rollback при ошибке.
2. **`--wait --timeout` для контроля.**
3. **`helm diff` перед upgrade.**

**👍 СТОИТ:**

4. **`--install` для идемпотентности.**
5. **Version в имени релиза** для окружений (`myapp-prod`, `myapp-dev`).
6. **`helm history` и `helm get values`** для диагностики.

**❌ НЕ ДЕЛАЙ:**

7. **Не делай `helm upgrade` без diff в production.**
8. **Не удаляй релиз без `--keep-history`, если может понадобиться.**
9. **Не используй `helm install` для upgrade.**

### Где мы сейчас

Мы разобрали управление релизами. Теперь — **зависимости (subcharts)**.

---

## 15.7 Зависимости: subcharts и Condition

### 🔌 Проблема: приложению нужны зависимости

Твоему приложению нужны:

- **PostgreSQL** для данных.
- **Redis** для кэша.
- **Kafka** для событий.

Ты можешь устанавливать их отдельно, но это неудобно. Лучше — **включить их в чарт**.

**Решение:** subcharts.

### 📊 Что такое subcharts

**Subchart** — это чарт, вложенный в другой чарт. Устанавливается **вместе** с родительским.

**Структура:**

```
myapp/
├── Chart.yaml
├── values.yaml
├── templates/
└── charts/
    ├── postgresql/
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   └── templates/
    └── redis/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

### 🎯 Два способа добавить зависимость

**Способ 1: `helm dependency add`**

```bash
# Добавить зависимость
helm dependency add postgresql https://charts.bitnami.com/bitnami --version 12.x.x

# Обновить зависимости
helm dependency update
```

**Способ 2: `Chart.yaml`**

```yaml
apiVersion: v2
name: myapp
version: 1.0.0
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  - name: redis
    version: "18.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
    tags:
      - cache
```

**После редактирования:**

```bash
helm dependency update
```

**Что произойдёт:**

- Скачает `postgresql-12.x.x.tgz` в `charts/`.
- Создаст `Chart.lock` с зафиксированными версиями.

### 🎯 `Chart.lock`

**Зафиксированные версии:**

```yaml
dependencies:
  - name: postgresql
    repository: https://charts.bitnami.com/bitnami
    version: 12.5.6
  - name: redis
    repository: https://charts.bitnami.com/bitnami
    version: 18.4.2
digest: sha256:abc123...
generated: "2026-01-15T10:00:00Z"
```

**Зачем:** чтобы у всех была **одна** версия зависимостей.

**Коммитить `Chart.lock` в Git.**

### 🎯 Condition и Tags

**Condition** — включить/выключить зависимость через values:

```yaml
dependencies:
  - name: postgresql
    condition: postgresql.enabled
```

**values.yaml:**

```yaml
postgresql:
  enabled: true      # включить PostgreSQL
redis:
  enabled: false     # не устанавливать Redis
```

**Когда использовать:** для окружений, где БД — managed.

**Tags** — группировка:

```yaml
dependencies:
  - name: postgresql
    tags:
      - database
  - name: redis
    tags:
      - cache
```

**values.yaml:**

```yaml
tags:
  database: true
  cache: false
```

**Когда использовать:** для группировки связанных зависимостей.

### 🎯 Values для subcharts

**Values subchart'а указываются в родительском `values.yaml` под ключом с именем subchart'а:**

```yaml
# Родительский values.yaml
postgresql:
  enabled: true
  auth:
    database: myapp
    username: myapp
    password: secret
  primary:
    persistence:
      size: 10Gi

redis:
  enabled: false
```

**Что произойдёт:** values `postgresql` передаются в subchart `postgresql`.

### 🎯 Global values

**Global values** доступны **всем** subchart'ам:

```yaml
global:
  environment: production
  imageRegistry: myregistry.com
```

**В subchart'ах:**

```yaml
image: "{{ .Values.global.imageRegistry }}/myapp:1.0"
```

### 🎯 Переопределение values subchart

**Файл `values-postgresql.yaml`** для override:

```bash
helm install myapp ./myapp -f values-postgresql.yaml
```

**Или через `--set`:**

```bash
helm install myapp ./myapp --set postgresql.auth.password=secret
```

### 🎯 Обновление версий subchart

```bash
# Обновить все
helm dependency update

# Обновить конкретный
helm dependency update --skip-refresh

# Посмотреть зависимости
helm dependency list
```

### 🔬 Практика: subcharts

```bash
# 1. Создать родительский чарт
helm create myapp
cd myapp

# 2. Добавить postgresql как зависимость
cat >> Chart.yaml <<EOF
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
EOF

# 3. Обновить зависимости
helm dependency update
# Hang tight while we grab the latest from your chart repositories...
# ...Successfully got an update from the "bitnami" chart repository
# Saving 1 charts
# Downloading postgresql from repo https://charts.bitnami.com/bitnami
# ...
# Deleting outdated charts

# 4. Проверить
ls charts/
# postgresql-12.x.x.tgz

cat Chart.lock

# 5. Values
cat > values.yaml <<EOF
postgresql:
  enabled: true
  auth:
    database: myapp
    username: myapp
    password: secret
  primary:
    persistence:
      size: 10Gi
EOF

# 6. Установить
helm install my-release .

# 7. Проверить — PostgreSQL установлен
kubectl get pods
# NAME                          READY   STATUS
# my-release-postgresql-0       1/1     Running
# my-release-myapp-xxx          1/1     Running
```

### 💡 Практика: как правильно использовать subcharts

**✅ ОБЯЗАТЕЛЬНО:**

1. **`Chart.lock` в Git** для фиксации версий.
2. **Condition для опциональных зависимостей.**
3. **Values subchart'ов** в родительском `values.yaml`.

**👍 СТОИТ:**

4. **Tags для группировки.**
5. **Global values** для общих настроек.
6. **Отдельные чарты** для переиспользования.

**❌ НЕ ДЕЛАЙ:**

7. **Не переопределяй subchart values из CLI без необходимости.** Лучше в `values.yaml`.
8. **Не используй `latest` для версий subchart'ов.**

### Где мы сейчас

Мы разобрали subcharts. Теперь — **hooks**.

---

## 15.8 Hooks: миграции и другие задачи

### 🔌 Проблема: миграции БД должны выполниться до старта

Приложение требует миграций БД. Они должны выполниться **до** того, как Pod'ы начнут обрабатывать трафик.

Как это сделать в Helm?

**Решение:** hooks.

### 📊 Что такое hook

**Hook** — это специальный манифест, который выполняется **в определённый момент** жизненного цикла релиза.

**Этапы (hooks):**

| Hook | Когда выполняется |
|:---|:---|
| `pre-install` | До установки ресурсов |
| `post-install` | После установки |
| `pre-delete` | До удаления |
| `post-delete` | После удаления |
| `pre-upgrade` | До upgrade |
| `post-upgrade` | После upgrade |
| `pre-rollback` | До rollback |
| `post-rollback` | После rollback |
| `test` | При `helm test` |

### 🎯 Пример: миграция БД

**templates/migration-job.yaml:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-migration
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "1"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migration
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["./myapp", "migrate", "up"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "myapp.fullname" . }}-secret
                  key: database-url
```

**Разберём annotations:**

| Annotation | Что означает |
|:---|:---|
| `helm.sh/hook` | На каких этапах запускать |
| `helm.sh/hook-weight` | Порядок (меньше — раньше) |
| `helm.sh/hook-delete-policy` | Когда удалять hook |

**Что произойдёт:**

1. **При `helm install`** — выполнится `pre-install` Job.
2. **При `helm upgrade`** — выполнится `pre-upgrade` Job.
3. После успешного завершения — Helm продолжит установку/обновление.
4. Если Job упал — Helm остановит операцию.

### 🎯 Hook Weights

**Порядок выполнения хуков одного типа:**

```yaml
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "-5"    # выполнится первым
```

```yaml
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "5"     # выполнится позже
```

**Правило:** меньше weight — раньше.

### 🎯 Delete Policy

| Policy | Что означает |
|:---|:---|
| `before-hook-creation` | Удалить предыдущий hook перед созданием нового |
| `hook-succeeded` | Удалить после успешного выполнения |
| `hook-failed` | Удалить после неудачи |

**Пример:**

```yaml
annotations:
  "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

**Что произойдёт:**

- Перед новым запуском — удалит старый hook.
- После успешного выполнения — удалит.
- При неудаче — оставит (для диагностики).

### 🎯 Smoke tests через `helm test`

**templates/tests/test-connection.yaml:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "myapp.fullname" . }}-test-connection
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  restartPolicy: Never
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['-qO-', 'http://{{ include "myapp.fullname" . }}:{{ .Values.service.port }}']
```

**Запуск:**

```bash
helm test myapp
# NAME: myapp
# LAST DEPLOYED: ...
# STATUS: deployed
# ...
# Phase: Succeeded
```

**Что произойдёт:** Pod запустится, выполнит `wget`, завершится. Helm покажет результат.

### 🎯 Практический пример: полный lifecycle

**Чарт с hooks:**

```
myapp/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── migration-job.yaml     # pre-upgrade hook
    ├── backup-job.yaml         # pre-upgrade hook (weight -5)
    └── tests/
        └── test-connection.yaml # test hook
```

**При `helm install`:**

1. `pre-install` hooks (backup, migration).
2. Ресурсы (deployment, service).
3. `post-install` hooks (если есть).
4. Notes.

**При `helm upgrade`:**

1. `pre-upgrade` hooks (backup → migration).
2. Обновление ресурсов.
3. `post-upgrade` hooks.
4. Notes.

**При `helm test`:**

1. `test` hooks.

### 🎯 Отладка hooks

**Проблема:** hook упал, но непонятно почему.

**Диагностика:**

```bash
# Посмотреть hook
kubectl get jobs
kubectl describe job <name>
kubectl logs job/<name>

# Посмотреть pods
kubectl get pods
kubectl logs <pod>
```

**`helm get hooks myapp`** — список hooks релиза.

**`helm get manifest myapp`** — манифесты.

### 🔬 Практика: hooks

```bash
# 1. Создать чарт
helm create demo
cd demo

# 2. Создать migration hook
cat > templates/migration-job.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "demo.fullname" . }}-migration
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "1"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migration
          image: busybox
          command: ["sh", "-c", "echo 'Running migration...'; sleep 5; echo 'Done!'"]
EOF

# 3. Установить
helm install my-release .
# Migration Job выполнится до установки

# 4. Проверить
kubectl get jobs
kubectl logs job/my-release-demo-migration

# 5. Upgrade
helm upgrade my-release .
# Migration Job выполнится снова

# 6. History hooks
helm get hooks my-release
```

### 💡 Практика: как правильно использовать hooks

**✅ ОБЯЗАТЕЛЬНО:**

1. **`pre-upgrade,pre-install` для миграций.**
2. **`hook-delete-policy: before-hook-creation,hook-succeeded`** для чистоты.
3. **`restartPolicy: Never`** в hook Pod.

**👍 СТОИТ:**

4. **`hook-weight` для порядка.**
5. **`test` hooks для smoke tests.**
6. **Backup hook перед upgrade** (weight -5).

**❌ НЕ ДЕЛАЙ:**

7. **Не делай hook долгим.** Timeout может убить его.
8. **Не забывай про delete policy.** Иначе накопятся Job'ы.
9. **Не используй hooks для основных ресурсов.** Только для специальных операций.

### Где мы сейчас

Мы разобрали hooks. Теперь — **OCI-registry** для чартов.

---

## 15.9 OCI-registry: чарты как артефакты

### 🔌 Проблема: как распространять чарты

Ты написал чарт. Хочешь поделиться с командой. Или использовать в CI/CD.

**Варианты:**

1. **Git-репозиторий** — чарт в Git, Helm читает через HTTP.
2. **Chart repository** — специальный сервер (ChartMuseum).
3. **OCI-registry** — использовать Docker registry (Harbor, ECR, GitLab Registry).

**OCI-registry — современный стандарт.** Чарты — такие же артефакты, как Docker-образы.

### 📊 Что такое OCI для чартов

**OCI (Open Container Initiative)** — стандарт для контейнеров. Но он универсален: можно хранить **любые артефакты**, включая Helm-чарты.

**Преимущества:**

- **Использовать существующий registry** (Harbor, ECR, GCR, GitLab Registry).
- **Единый интерфейс** для образов и чартов.
- **Аутентификация** уже настроена.
- **Retention policies** уже настроены.

### 🎯 Публикация чарта

**1. Запаковать чарт:**

```bash
helm package ./myapp
# Successfully packaged chart and saved it to: myapp-1.0.0.tgz
```

**2. Залогиниться в registry:**

```bash
helm registry login myregistry.com
# Username: myuser
# Password: ********
```

**3. Запушить:**

```bash
helm push myapp-1.0.0.tgz oci://myregistry.com/charts
# Pushed: myregistry.com/charts/myapp:1.0.0
# Digest: sha256:abc123...
```

**Что произошло:** чарт загружен как OCI-артефакт.

### 🎯 Установка из OCI

```bash
# Установить
helm install myapp oci://myregistry.com/charts/myapp --version 1.0.0

# С values
helm install myapp oci://myregistry.com/charts/myapp \
  --version 1.0.0 \
  -f values-prod.yaml

# Показать values без установки
helm show values oci://myregistry.com/charts/myapp --version 1.0.0

# Скачать чарт
helm pull oci://myregistry.com/charts/myapp --version 1.0.0
```

### 🎯 Публикация в GitLab Registry

**GitLab Container Registry** поддерживает OCI-артефакты.

**CI/CD:**

```yaml
publish-chart:
  stage: publish
  image: alpine/helm:latest
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | helm registry login $CI_REGISTRY -u $CI_REGISTRY_USER --password-stdin
  script:
    - helm package ./myapp
    - helm push myapp-*.tgz oci://$CI_REGISTRY/$CI_PROJECT_PATH/charts
  rules:
    - if: $CI_COMMIT_TAG
```

**Установка:**

```bash
helm registry login registry.gitlab.com
helm install myapp oci://registry.gitlab.com/myorg/myapp/charts/myapp --version 1.0.0
```

### 🎯 Публикация в AWS ECR

**ECR** поддерживает OCI-артефакты.

```bash
# Логин
aws ecr get-login-password --region us-west-2 | helm registry login --username AWS --password-stdin 123456789.dkr.ecr.us-west-2.amazonaws.com

# Пуш
helm push myapp-1.0.0.tgz oci://123456789.dkr.ecr.us-west-2.amazonaws.com/charts
```

### 🎯 Публикация в Harbor

**Harbor** — популярный self-hosted registry.

```bash
# Логин
helm registry login harbor.example.com

# Пуш
helm push myapp-1.0.0.tgz oci://harbor.example.com/myproject/charts
```

**Harbor** имеет UI, где можно посмотреть чарты, версии, значения.

### 🎯 CI/CD с OCI-чартами

**Полный пайплайн:**

```yaml
stages:
  - lint
  - package
  - publish

lint-chart:
  stage: lint
  image: alpine/helm:latest
  script:
    - helm lint ./myapp

package-chart:
  stage: package
  image: alpine/helm:latest
  script:
    - helm package ./myapp --version $CI_COMMIT_TAG
  artifacts:
    paths:
      - myapp-*.tgz
  rules:
    - if: $CI_COMMIT_TAG

publish-chart:
  stage: publish
  image: alpine/helm:latest
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | helm registry login $CI_REGISTRY -u $CI_REGISTRY_USER --password-stdin
  script:
    - helm push myapp-*.tgz oci://$CI_REGISTRY/$CI_PROJECT_PATH/charts
  rules:
    - if: $CI_COMMIT_TAG
```

**Что произошло:**

1. **Lint** — проверка чарта.
2. **Package** — упаковка в `.tgz`.
3. **Publish** — пуш в OCI-registry.

**При создании git-тега** — автоматическая публикация.

### 🔬 Практика: OCI-registry

```bash
# 1. Запустить локальный registry (см. Главу 6)
docker run -d -p 5000:5000 --name registry registry:2

# 2. Запаковать чарт
helm package ./demo

# 3. Запушить
helm push demo-0.1.0.tgz oci://localhost:5000/charts --insecure-skip-tls-verify

# 4. Проверить
curl -X GET http://localhost:5000/v2/charts/demo/tags/list

# 5. Установить из OCI
helm install my-release oci://localhost:5000/charts/demo \
  --version 0.1.0 \
  --insecure-skip-tls-verify

# 6. Скачать
helm pull oci://localhost:5000/charts/demo --version 0.1.0 --insecure-skip-tls-verify
```

### 💡 Практика: как правильно публиковать чарты

**✅ ОБЯЗАТЕЛЬНО:**

1. **OCI-registry для production.** Не ChartMuseum.
2. **Версионирование через git tags.**
3. **Lint перед публикацией.**

**👍 СТОИТ:**

4. **GitLab Registry или Harbor** — встроенные, удобные.
5. **Retention policies** — удалять старые версии.
6. **Подписывание чартов** (Provenance).

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `latest` тег.** Только semver.
8. **Не публикуй чарты без lint.**
9. **Не храни чарты только в Git.** OCI удобнее.

### Где мы сейчас

Мы разобрали OCI-registry. Теперь — **тестирование и отладка**.

---

## 15.10 Тестирование и отладка чартов

### 🔌 Проблема: чарт сломался при деплое

Ты написал чарт. `helm install` упал с непонятной ошибкой. Или применился, но Pod'ы не работают.

Как отлаживать?

### 📊 Инструменты

**1. `helm lint`**

Проверка чарта на ошибки:

```bash
helm lint ./myapp
# ==> Linting ./myapp
# [INFO] Chart.yaml: icon is recommended
# [ERROR] templates/: template: myapp/templates/deployment.yaml:25:12: executing "..." at <.Values.image.tag>: nil pointer evaluating interface {}.tag
```

**Что проверяет:**

- Синтаксис `Chart.yaml`.
- Синтаксис `values.yaml`.
- Синтаксис шаблонов.
- Ссылки на несуществующие values.

**2. `helm template`**

Рендеринг шаблонов **без установки**:

```bash
# Все шаблоны
helm template my-release ./myapp

# Один шаблон
helm template my-release ./myapp -s templates/deployment.yaml

# С values
helm template my-release ./myapp -f values-prod.yaml

# Debug (показать все values)
helm template my-release ./myapp --debug
```

**Что даёт:** видишь **итоговый YAML** без установки.

**3. `helm install --dry-run`**

Проверка без установки:

```bash
helm install my-release ./myapp --dry-run

# С debug
helm install my-release ./myapp --dry-run --debug
```

**Что даёт:** Helm проверит, что манифесты валидны для API.

**4. `helm install --debug`**

Полный вывод с отладкой:

```bash
helm install my-release ./myapp --debug
```

**Что даёт:**

- Все values (включая defaults).
- Отрендеренные манифесты.
- Действия Helm.

**5. `helm get manifest`**

Посмотреть, что **реально** задеплоено:

```bash
helm get manifest my-release
```

**Полезно для:** сравнения с `helm template`.

**6. `helm diff`**

Сравнение с текущим состоянием:

```bash
helm diff upgrade my-release ./myapp
```

**Критически важно** перед upgrade.

**7. `kubectl`**

```bash
kubectl get all
kubectl describe pod <pod>
kubectl logs <pod>
kubectl get events --sort-by=.lastTimestamp
```

### 🎯 Типичные проблемы

**1. `nil pointer evaluating interface {}.tag`**

**Причина:** обращение к несуществующему ключу в values.

**Решение:**

```yaml
# Вместо
image: {{ .Values.image.tag }}

# Использовать default
image: {{ .Values.image.tag | default .Chart.AppVersion }}
```

**2. `Error: UPGRADE FAILED: cannot patch "..." with kind Deployment`**

**Причина:** нельзя изменить immutable поле (например, selector).

**Решение:** удалить и создать заново, или использовать `--force`.

**3. `Error: release myapp failed: ... is invalid: ...`**

**Причина:** невалидный манифест.

**Решение:** `helm template` чтобы увидеть итоговый YAML.

**4. `Error: cannot re-use a name that is still in use`**

**Причина:** релиз с таким именем уже существует.

**Решение:** `helm upgrade --install` или удалить старый.

**5. `Error: timed out waiting for the condition`**

**Причина:** `--wait` включён, но ресурсы не стали Ready за timeout.

**Решение:** увеличить `--timeout` или разобраться, почему Pod'ы не Ready.

**6. `Error: unable to build kubernetes objects from release manifest`**

**Причина:** невалидный YAML (часто из-за отступов).

**Решение:** `helm template --debug` для проверки.

### 🎯 Debugging техники

**1. Пошаговая отладка:**

```bash
# 1. Lint
helm lint ./myapp

# 2. Template
helm template my-release ./myapp > /tmp/rendered.yaml
cat /tmp/rendered.yaml

# 3. Dry-run
helm install my-release ./myapp --dry-run --debug

# 4. Install
helm install my-release ./myapp

# 5. Status
helm status my-release
kubectl get all

# 6. If failed
kubectl describe pod <pod>
kubectl logs <pod>
```

**2. Отладка values:**

```bash
# Какие values передаются
helm get values my-release --all

# Как values рендерятся
helm template my-release ./myapp --debug
```

**3. Отладка templates:**

Добавить в шаблон:

```yaml
{{/* Debug: {{ .Values | toYaml | nindent 2 }} */}}
```

**4. Проверка схемы:**

```bash
# Если есть values.schema.json — проверить
helm install my-release ./myapp --set wrong=value
# Error: values don't meet the specifications of the schema
```

### 🎯 Плагины для тестирования

**1. helm-unittest**

Unit-тесты для шаблонов:

```bash
helm plugin install https://github.com/helm-unittest/helm-unittest
```

**tests/deployment_test.yaml:**

```yaml
suite: test deployment
templates:
  - deployment.yaml
tests:
  - it: should create deployment with correct replicas
    set:
      replicaCount: 3
    asserts:
      - equal:
          path: spec.replicas
          value: 3
  
  - it: should use correct image
    set:
      image.repository: myapp
      image.tag: v1.0.0
    asserts:
      - equal:
          path: spec.template.spec.containers[0].image
          value: myapp:v1.0.0
```

**Запуск:**

```bash
helm unittest ./myapp
```

**2. helm-docs**

Автогенерация README из values:

```bash
helm plugin install https://github.com/norwoodj/helm-docs
helm-docs ./myapp
```

**3. kubeconform**

Валидация манифестов против API:

```bash
helm template my-release ./myapp | kubeconform -strict -summary
```

### 🔬 Практика: отладка

```bash
# 1. Создать чарт с ошибкой
helm create demo
cd demo

# 2. Добавить шаблон с ошибкой
cat > templates/broken.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "demo.fullname" . }}
data:
  image: {{ .Values.image.tag }}
EOF

# 3. Lint — не покажет ошибку (values есть)
helm lint .

# 4. Если убрать image из values.yaml
# Lint покажет ошибку

# 5. Template — покажет итоговый YAML
helm template my-release .

# 6. Dry-run
helm install my-release . --dry-run --debug
```

### 💡 Практика: как правильно тестировать чарты

**✅ ОБЯЗАТЕЛЬНО:**

1. **`helm lint` перед коммитом.**
2. **`helm template` для проверки рендеринга.**
3. **`helm install --dry-run` перед реальной установкой.**
4. **`helm diff` перед upgrade в production.**

**👍 СТОИТ:**

5. **helm-unittest** для unit-тестов.
6. **kubeconform** для валидации против API.
7. **helm-docs** для автогенерации документации.

**❌ НЕ ДЕЛАЙ:**

8. **Не игнорируй lint errors.**
9. **Не деплой без dry-run.**
10. **Не используй `--force` без понимания.**

### Где мы сейчас

Мы разобрали тестирование. Теперь — **Helm vs Kustomize**.

---

## 15.11 Helm vs Kustomize

### 🔌 Проблема: какой инструмент выбрать

Helm — не единственный инструмент для управления манифестами. **Kustomize** — альтернатива от Kubernetes SIG.

**Как выбрать?**

### 📊 Что такое Kustomize

**Kustomize** — инструмент для **патчинга** YAML без шаблонов. Встроен в `kubectl` (`kubectl apply -k`).

**Основные понятия:**

- **Base** — базовые манифесты.
- **Overlay** — патчи для окружений.
- **kustomization.yaml** — описание, что применять.

**Пример:**

```
myapp/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── replicas-patch.yaml
    └── prod/
        ├── kustomization.yaml
        └── replicas-patch.yaml
```

**base/kustomization.yaml:**

```yaml
resources:
  - deployment.yaml
  - service.yaml
```

**overlays/prod/kustomization.yaml:**

```yaml
resources:
  - ../../base
patches:
  - path: replicas-patch.yaml
images:
  - name: myapp
    newTag: v1.0.0
```

**overlays/prod/replicas-patch.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
```

**Применение:**

```bash
kubectl apply -k overlays/prod
```

### 📊 Сравнение

| Аспект | Helm | Kustomize |
|:---|:---|:---|
| Подход | Templating | Patching |
| Синтаксис | Go templates | YAML |
| Управление релизами | ✅ Да | ❌ Нет |
| Rollback | ✅ Да | ❌ Нет |
| Hooks | ✅ Да | ❌ Нет |
| Зависимости | ✅ Subcharts | ❌ Нет |
| Параметризация | Values | Patches |
| Сложность | Средняя | Низкая |
| Встроен в kubectl | ❌ Нет | ✅ Да |
| Обучаемость | Дольше | Быстрее |

### 🎯 Когда использовать Helm

**✅ Helm:**

- **Сложные приложения** с параметризацией.
- **Управление релизами** — install, upgrade, rollback.
- **Hooks** — миграции, smoke tests.
- **Зависимости** — subcharts.
- **Публикация** — OCI-registry, sharing с командой.
- **Third-party чарты** — Bitnami, Prometheus.

**❌ Не Helm:**

- **Простые приложения** — Kustomize проще.
- **Только патчинг** — Kustomize элегантнее.
- **Нет сложной логики** — Kustomize читаемее.

### 🎯 Когда использовать Kustomize

**✅ Kustomize:**

- **Патчинг base-манифестов** для окружений.
- **Простые приложения** без сложной параметризации.
- **Kubernetes-native** — встроен в kubectl.
- **GitOps-friendly** — ArgoCD, Flux поддерживают.
- **Нет логики** — только изменения YAML.

**❌ Не Kustomize:**

- **Нужны hooks** — Helm.
- **Нужны релизы** — Helm.
- **Нужны subcharts** — Helm.
- **Сложная параметризация** — Helm.

### 🎯 Комбинирование: лучшее из двух

**Helm + Kustomize:**

1. **Helm** для third-party чартов (Postgres, Redis, Prometheus).
2. **Kustomize** для своих приложений.

**Или:**

1. **Helm** для параметризации.
2. **Kustomize** для патчинга Helm-выхода (редко).

**Пример:**

```bash
# Helm рендерит чарт
helm template my-release ./myapp > /tmp/rendered.yaml

# Kustomize патчит
kubectl apply -k /tmp/kustomize-overlay
```

**Но это редкий случай.** Обычно выбирают один инструмент.

### 🎯 GitOps с Helm и Kustomize

**ArgoCD** поддерживает **оба**:

**Helm:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  source:
    repoURL: https://github.com/myorg/myapp
    targetRevision: HEAD
    path: charts/myapp
    helm:
      values: |
        replicaCount: 3
```

**Kustomize:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  source:
    repoURL: https://github.com/myorg/myapp
    targetRevision: HEAD
    path: overlays/prod
    kustomize:
      images:
        - myapp:v1.0.0
```

### 🎯 Recommendation

**Правило:**

1. **Если ты публикуешь чарт** (third-party) — Helm.
2. **Если ты используешь third-party чарты** — Helm.
3. **Если тебе нужны релизы/rollback/hooks** — Helm.
4. **Если у тебя простые приложения с патчами** — Kustomize.
5. **Если ты хочешь Kubernetes-native** — Kustomize.

**Для большинства команд:** Helm для third-party, Kustomize для своих приложений. Или только Helm.

### 🔬 Практика: Kustomize

```bash
# 1. Создать base
mkdir -p demo/base demo/overlays/prod
cd demo

# 2. base/deployment.yaml
cat > base/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
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

# 3. base/kustomization.yaml
cat > base/kustomization.yaml <<'EOF'
resources:
  - deployment.yaml
EOF

# 4. overlays/prod/kustomization.yaml
cat > overlays/prod/kustomization.yaml <<'EOF'
resources:
  - ../../base
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
    target:
      kind: Deployment
      name: myapp
images:
  - name: myapp
    newTag: v1.0.0
EOF

# 5. Просмотр
kubectl kustomize overlays/prod

# 6. Применение
kubectl apply -k overlays/prod
```

### 💡 Практика: как выбирать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Helm для third-party чартов.**
2. **Helm для сложных приложений** с hooks, releases.
3. **Kustomize для простых приложений** с патчами.

**👍 СТОИТ:**

4. **Использовать оба** для разных задач.
5. **Не смешивать** в одном приложении без необходимости.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй Kustomize для third-party чартов.** Helm удобнее.
7. **Не используй Helm для простых патчей.** Kustomize проще.

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **Helm** | Менеджер пакетов для Kubernetes. |
| **Chart** | Пакет с манифестами и параметрами. |
| **Release** | Установленный экземпляр чарта. |
| **Values** | Параметры для чарта. |
| **Templates** | Шаблоны манифестов (Go templates). |
| **Chart.yaml** | Метаданные чарта. |
| **values.yaml** | Значения по умолчанию. |
| **values.schema.json** | JSON Schema для валидации. |
| **_helpers.tpl** | Helper-функции. |
| **NOTES.txt** | Сообщение после установки. |
| **Sprig** | Библиотека функций для Go templates. |
| **nindent** | Newline + indent. |
| **Subchart** | Вложенный чарт. |
| **Chart.lock** | Зафиксированные версии зависимостей. |
| **Hook** | Специальный манифест на определённом этапе. |
| **pre-install, post-install** | Hooks для install. |
| **pre-upgrade, post-upgrade** | Hooks для upgrade. |
| **helm test** | Запуск test hooks. |
| **OCI-registry** | Registry для хранения чартов. |
| **helm lint** | Проверка чарта. |
| **helm template** | Рендеринг без установки. |
| **helm diff** | Сравнение с текущим состоянием. |
| **helm-unittest** | Плагин для unit-тестов. |
| **Kustomize** | Альтернатива Helm (patches). |

---

## Что мы узнали?

- **Helm** — менеджер пакетов для Kubernetes. Параметризация, релизы, rollback, hooks, зависимости.
- **Chart** — пакет с манифестами и values. **Release** — установленный экземпляр.
- **Шаблонизация:** Go templates + Sprig. `{{ .Values.key }}`, `include`, `nindent`.
- **Встроенные объекты:** `.Values`, `.Chart`, `.Release`, `.Files`, `.Capabilities`.
- **Helpers** в `_helpers.tpl` для переиспользования.
- **Релизы:** `install`, `upgrade`, `rollback`, `history`, `diff`.
- **Subcharts:** зависимости через `Chart.yaml`, `Chart.lock` для фиксации.
- **Hooks:** pre-install, post-upgrade, test. Для миграций, smoke tests.
- **OCI-registry:** чарты как артефакты. Публикация в GitLab, Harbor, ECR.
- **Тестирование:** `helm lint`, `helm template`, `--dry-run`, `helm diff`.
- **Helm vs Kustomize:** Helm для сложных, Kustomize для простых.

---

## Типичные ошибки

- ❌ **Хардкодить значения в templates.** Всё через `.Values`.
- ❌ **Забывать `nindent`.** YAML сломается.
- ❌ **Использовать `template` вместо `include`.**
- ❌ **Не использовать `--atomic` в production.**
- ❌ **Делать upgrade без `helm diff`.**
- ❌ **Игнорировать `helm lint`.**
- ❌ **Не фиксировать версии зависимостей в `Chart.lock`.**
- ❌ **Делать долгие hooks.** Timeout убьёт.
- ❌ **Забывать про `hook-delete-policy`.**
- ❌ **Публиковать чарты без версионирования.**
- ❌ **Использовать `latest` для версий.**
- ❌ **Смешивать Helm и Kustomize в одном приложении.**

---

## Для быстрого повторения

- **Helm:** менеджер пакетов. Chart, Release, Values.
- **Chart:** `Chart.yaml`, `values.yaml`, `templates/`, `_helpers.tpl`, `charts/`.
- **Templates:** Go templates + Sprig. `{{ .Values.key }}`, `include`, `nindent`, `{{-`.
- **Встроенные:** `.Values`, `.Chart`, `.Release`, `.Files`, `.Capabilities`.
- **Helpers:** `define`, `include`. Переиспользуемые фрагменты.
- **Релизы:** `install`, `upgrade --install --atomic`, `rollback`, `history`, `diff`.
- **Subcharts:** `dependencies` в Chart.yaml, `Chart.lock`, `condition`.
- **Hooks:** `pre-install`, `pre-upgrade`, `test`. `hook-weight`, `hook-delete-policy`.
- **OCI:** `helm push`, `helm pull oci://...`, `helm install oci://...`.
- **Тестирование:** `lint`, `template`, `--dry-run`, `--debug`, `helm-diff`.
- **Kustomize:** patches вместо templating. Встроен в kubectl.

---

## Вопросы для самопроверки

1. Зачем нужен Helm, если есть `kubectl apply`?
2. Что такое Chart, Release, Values?
3. Из чего состоит чарт? Назови основные файлы.
4. Что такое `_helpers.tpl`? Зачем нужен?
5. Чем `include` отличается от `template`?
6. Что делают `{{-` и `-}}`?
7. Что такое `nindent`? Когда использовать?
8. Как работает `helm rollback`?
9. Что такое `--atomic`? Зачем нужен?
10. Что такое subcharts? Как добавить зависимость?
11. Что такое hooks? Какие бывают?
12. Как опубликовать чарт в OCI-registry?
13. Чем Helm отличается от Kustomize?
14. Чарт сломался. Как отлаживать?
15. Что такое `Chart.lock`? Зачем нужен?

---

## Ответы

**1. Зачем Helm**

Параметризация (один чарт — много окружений), релизы (install, upgrade, rollback), hooks (миграции), зависимости (subcharts), упаковка (sharing).

**2. Chart, Release, Values**

Chart — пакет с манифестами. Release — установленный экземпляр чарта. Values — параметры для чарта.

**3. Основные файлы чарта**

- `Chart.yaml` — метаданные.
- `values.yaml` — значения по умолчанию.
- `templates/` — шаблоны.
- `_helpers.tpl` — helper-функции.
- `NOTES.txt` — сообщение после установки.
- `.helmignore` — исключения.
- `charts/` — зависимости.

**4. `_helpers.tpl`**

Файл с helper-функциями (define). Используется для переиспользования: labels, fullname, image и т.д.

**5. `include` vs `template`**

`include` возвращает строку, работает с `nindent`. `template` не возвращает значение. Всегда используй `include`.

**6. `{{-` и `-}}`**

Убирают пробелы/переводы строк слева (`{{-`) и справа (`-}}`). Нужны, чтобы не было пустых строк в YAML.

**7. `nindent`**

Newline + indent. Добавляет перевод строки и отступ к каждой строке. Используется для многострочных значений: `{{- include "helper" . | nindent 4 }}`.

**8. `helm rollback`**

Применяет манифесты указанной ревизии. Создаёт новую ревизию (не перезаписывает историю). `helm rollback myapp 1`.

**9. `--atomic`**

Автоматический rollback при неудачном upgrade. Важно для production.

**10. Subcharts**

Чарты, вложенные в родительский. Добавляются через `dependencies` в `Chart.yaml`. Обновляются через `helm dependency update`.

**11. Hooks**

Специальные манифесты на этапах релиза. `pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, `pre-delete`, `post-delete`, `test`. Для миграций, smoke tests.

**12. OCI-registry**

`helm package ./chart` → `helm push chart-1.0.0.tgz oci://registry/charts`. Установка: `helm install myapp oci://registry/charts/chart --version 1.0.0`.

**13. Helm vs Kustomize**

Helm: templating, releases, rollback, hooks, subcharts. Kustomize: patches, нет релизов, встроен в kubectl. Helm для сложных, Kustomize для простых.

**14. Отладка чарта**

`helm lint`, `helm template`, `helm install --dry-run --debug`, `helm diff upgrade`. Плюс `kubectl describe`, `kubectl logs`.

**15. `Chart.lock`**

Зафиксированные версии зависимостей. Гарантирует, что все используют одни и те же версии. Коммитить в Git.

---

## Куда идти дальше?

Мы разобрали Helm — менеджер пакетов для Kubernetes. Теперь ты знаешь:

- Chart, Release, Values.
- Шаблонизацию.
- Helpers.
- Управление релизами.
- Subcharts.
- Hooks.
- OCI-registry.
- Тестирование.

Но мы пока не разобрали:

- **Глава 16: Service Mesh** — уже написан, требует переименования (была как 20).
- **Глава 17: Безопасность Kubernetes** — уже написан (была как 21).
- **Глава 18: Terraform** — уже написан (была как 14).
- **Глава 19: Ansible** — уже написан (была как 15).
- **Глава 20: GitOps** — уже написан (была как 16).
- **Глава 21: Логирование** — уже написан (была как 17).
- **Глава 22: Мониторинг** — уже написан (была как 18).
- **Глава 23: Трейсинг** — уже написан (была как 19).
- **Глава 24: Отказоустойчивость** — уже написан (была как 22).
- **Глава 25: Платформенная инженерия** — уже написан (была как 23).
- **Глава 26: Облака** — не написана.

**Следующая — Глава 16: Service Mesh — Istio и Linkerd.** Она уже написана, но требует переименования. Скажи «дальше» — и я отправлю её в правильной нумерации.