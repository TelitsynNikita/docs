# 🏗️ Глава 18: Инфраструктура как код — Terraform

**Что вы узнаете:**
- Что такое Infrastructure as Code (IaC) и зачем это нужно.
- Чем Terraform отличается от Ansible, Pulumi, CloudFormation.
- Как работает Terraform: providers, resources, data sources.
- Что такое state и почему это одновременно сила и ответственность.
- Как использовать remote backend (S3, GCS) для командной работы.
- Что такое modules и как переиспользовать код.
- Как работают variables, outputs, locals.
- Как безопасно делать `plan` и `apply`.
- Как импортировать существующие ресурсы.
- Как интегрировать Terraform с Kubernetes и Helm.
- Как использовать Terraform в CI/CD.

**После прочтения вы сможете:**
- Написать Terraform-конфигурацию для облачной инфраструктуры.
- Использовать remote state с блокировками.
- Создавать модули для переиспользования.
- Управлять несколькими окружениями.
- Интегрировать Terraform с CI/CD.
- Диагностировать типичные проблемы с state.
- Использовать Terraform для Kubernetes (namespaces, RBAC, Helm releases).

---

## Содержание

- [18.0 Пролог: инфраструктура руками vs код](#180-пролог-инфраструктура-руками-vs-код)
- [18.1 Что такое Infrastructure as Code](#181-что-такое-infrastructure-as-code)
- [18.2 Terraform vs альтернативы](#182-terraform-vs-альтернативы)
- [18.3 Providers, resources, data sources](#183-providers-resources-data-sources)
- [18.4 State: сила и ответственность](#184-state-сила-и-ответственность)
- [18.5 Remote backend: командная работа](#185-remote-backend-командная-работа)
- [18.6 Modules: переиспользование кода](#186-modules-переиспользование-кода)
- [18.7 Variables, outputs, locals](#187-variables-outputs-locals)
- [18.8 Workspaces и environments](#188-workspaces-и-environments)
- [18.9 plan, apply, destroy: безопасная работа](#189-plan-apply-destroy-безопасная-работа)
- [18.10 Import: как подружиться с существующей инфраструктурой](#1810-import-как-подружиться-с-существующей-инфраструктурой)
- [18.11 Terraform + Kubernetes](#1811-terraform--kubernetes)
- [18.12 Terraform в CI/CD](#1812-terraform-в-cicd)
- [18.13 Диагностика проблем](#1813-диагностика-проблем)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 18.0 Пролог: инфраструктура руками vs код

Ты — DevOps-инженер. Тебе нужно создать окружение для нового проекта:

- **VPC** с подсетями.
- **EKS-кластер** (или GKE, AKS).
- **RDS PostgreSQL** для данных.
- **ElastiCache Redis** для кэша.
- **S3 bucket** для файлов.
- **IAM-роли** для сервисов.
- **Route53** для DNS.
- **CloudWatch** для логов.

Как ты это делаешь? Открываешь **консоль AWS**. Кликаешь по кнопкам. Создаёшь ресурсы. Копируешь ARN и endpoint'ы. Записываешь в документацию. Через месяц повторяешь для нового окружения. Через год — забыл, как создавал. Или приходит аудитор и спрашивает: «Почему у вас RDS с такими параметрами?»

**Это не масштабируется.**

**Infrastructure as Code (IaC)** решает эту проблему. Ты описываешь инфраструктуру **в коде**:

```hcl
resource "aws_db_instance" "postgres" {
  engine         = "postgres"
  engine_version = "16.1"
  instance_class = "db.t3.medium"
  allocated_storage = 100
  
  db_name  = "myapp"
  username = "postgres"
  password = var.db_password
  
  backup_retention_period = 30
  multi_az                = true
}
```

**Этот код:**

- **Версионируется** в Git.
- **Ревьюится** через Pull Request.
- **Применяется** одной командой.
- **Воспроизводится** для любого окружения.
- **Документирует** инфраструктуру.

**Terraform** — самый популярный инструмент IaC. В этой главе мы разберём его от основ до продвинутых техник.

Это — Первый путь DevOps (Flow) в действии. Из Главы 0: инфраструктура создаётся быстро, воспроизводимо, автоматизированно.

---

## 18.1 Что такое Infrastructure as Code

### 🔌 Проблема: инфраструктура создаётся вручную

Традиционный подход к инфраструктуре:

1. **Клики в консоли.** Открыл AWS Console, создал ресурсы.
2. **Документация в wiki.** «Как создать VPC: зайти туда, нажать кнопку...»
3. **Передача знаний.** Новый сотрудник читает документацию, спрашивает коллег.
4. **Ручные операции.** Изменения делаются вручную, часто неправильно.
5. **Нет истории.** Кто, когда, зачем изменил — неизвестно.

**Проблемы:**

- **Невоспроизводимо.** Каждое окружение создаётся по-своему.
- **Медленно.** Создание VPC — 30 минут кликов.
- **Ошибки.** Человек забывает параметры, делает опечатки.
- **Нет версионирования.** Нет истории изменений.
- **Нет ревью.** Никто не проверяет изменения.
- **Не масштабируется.** Создать 10 окружений — 10 раз клики.

### 📦 Что такое Infrastructure as Code

**Infrastructure as Code (IaC)** — подход, при котором инфраструктура описывается **декларативно** в текстовых файлах, которые:

- **Хранятся в Git.**
- **Версионируются.**
- **Ревьюятся.**
- **Применяются автоматически.**

**Ключевые принципы:**

1. **Декларативность.** Описываешь **что** должно быть, не **как**.
2. **Версионирование.** Всё в Git.
3. **Идемпотентность.** `apply` можно запускать много раз — результат один.
4. **Воспроизводимость.** Один код — одинаковый результат.
5. **Автоматизация.** Применяется в CI/CD.

### 📊 Декларативный vs императивный

| Подход | Как | Пример |
|:---|:---|:---|
| **Императивный** | Пошаговые команды | `aws ec2 create-instance` |
| **Декларативный** | Описание состояния | `resource "aws_instance" {...}` |

**Императивный:** ты говоришь «сделай шаг 1, шаг 2, шаг 3». Если один шаг упал — надо разбираться.

**Декларативный:** ты говоришь «я хочу 3 инстанса». Terraform сам решает, как это сделать.

### 🎯 Преимущества IaC

**1. Воспроизводимость.**

Один код → одинаковый результат. Dev, staging, prod — одинаковые.

**2. Скорость.**

Создание инфраструктуры — секунды (если применить) вместо часов (клики).

**3. Версионирование.**

Git — история изменений. `git blame` — кто, когда, зачем.

**4. Ревью.**

Pull Request → коллеги проверяют → merge → apply.

**5. Документация.**

Код — это документация. Всегда актуальная.

**6. Автоматизация.**

CI/CD: изменения в Git → автоматический plan → ревью → apply.

**7. Откат.**

Если что-то пошло не так — `git revert` → `apply`.

### 🎯 Инструменты IaC

**Провижининг инфраструктуры:**

- **Terraform** — самый популярный.
- **Pulumi** — код на Go, Python, TypeScript.
- **CloudFormation** — AWS-specific.
- **CDK** — Cloud Development Kit (AWS, Terraform).
- **Crossplane** — Kubernetes-native.

**Конфигурация серверов:**

- **Ansible** — push-based.
- **Chef** — pull-based.
- **Puppet** — pull-based.
- **SaltStack** — pull-based.

**Разница:**

- **Terraform** — создаёт инфраструктуру (VPC, VM, БД).
- **Ansible** — настраивает существующие серверы (установить пакеты, конфиги).

Мы разберём оба: Terraform в этой главе, Ansible в следующей.

### 💡 Практика: что важно понять про IaC

**✅ ОБЯЗАТЕЛЬНО:**

1. **IaC — это декларативность.** Описываешь состояние.
2. **IaC — это Git.** Всё версионируется.
3. **IaC — это воспроизводимость.** Один код — одинаковый результат.

**👍 СТОИТ:**

4. **Ревью изменений** через Pull Request.
5. **CI/CD для применения** — автоматизация.
6. **Разделение окружений** — dev, staging, prod в разных папках/workspaces.

**❌ НЕ ДЕЛАЙ:**

7. **Не делай изменения вручную** после применения IaC. Drift.
8. **Не храни secrets в коде.** Используй Vault, AWS Secrets Manager.
9. **Не применяй без plan.** Всегда смотри, что изменится.

### Где мы сейчас

Мы разобрали, что такое IaC. Теперь — **Terraform vs альтернативы**.

---

## 18.2 Terraform vs альтернативы

### 🔌 Проблема: какой инструмент выбрать

Terraform — не единственный инструмент IaC. Есть альтернативы. Как выбрать?

### 📊 Сравнение

| Инструмент | Язык | Подход | Multi-cloud | Состояние |
|:---|:---|:---|:---|:---|
| **Terraform** | HCL | Декларативный | ✅ Да | State file |
| **Pulumi** | Go, Python, TS | Императивный | ✅ Да | State file |
| **CloudFormation** | JSON/YAML | Декларативный | ❌ AWS only | Управляется AWS |
| **CDK** | Python, TS | Императивный | ✅ (через CDKTF) | Управляется AWS |
| **Crossplane** | YAML | Kubernetes-native | ✅ Да | K8s CRD |
| **Ansible** | YAML | Императивный | ✅ Да | Не нужен |

### 🎯 Terraform

**HashiCorp Configuration Language (HCL).**

**Плюсы:**

- **Самый популярный.** Много провайдеров, документации, примеров.
- **Multi-cloud.** AWS, GCP, Azure, Kubernetes, Cloudflare, GitHub — сотни провайдеров.
- **Модули.** Переиспользование кода.
- **Большое сообщество.**
- **Terraform Cloud / Enterprise** — managed state, CI/CD.

**Минусы:**

- **HCL — отдельный язык.** Нужно учить.
- **State file — ответственность.** Можно испортить.
- **Лицензия** — сменилась на BSL. Есть форк OpenTofu.

### 🎯 OpenTofu

**Форк Terraform** от Linux Foundation. Появился после смены лицензии HashiCorp (с MPL на BSL).

**Совместим с Terraform** 1.5.x.

**Плюсы:**

- **Open source (MPL).**
- **Совместим с Terraform.**
- **Управляется сообществом.**

**Минусы:**

- **Моложе.** Меньше документации.
- **Меньше провайдеров** (хотя совместим с большинством).

**Когда использовать:** если важна лицензия или работа с open-source.

### 🎯 Pulumi

**Код на Go, Python, TypeScript.**

**Плюсы:**

- **Настоящий язык программирования.** Циклы, функции, условия.
- **Типизация** (в TS).
- **Тесты** как обычный код.
- **Много облаков.**

**Минусы:**

- **Сложнее для простых случаев.** HCL проще.
- **Меньше примеров.**
- **Меньше сообщество.**

**Пример Pulumi (Go):**

```go
package main

import (
    "github.com/pulumi/pulumi-aws/sdk/v6/go/aws/s3"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        bucket, err := s3.NewBucket(ctx, "my-bucket", &s3.BucketArgs{
            Acl: pulumi.String("private"),
        })
        if err != nil {
            return err
        }
        ctx.Export("bucketName", bucket.ID())
        return nil
    })
}
```

**Когда использовать:** если команда любит Go/Python/TS.

### 🎯 CloudFormation

**AWS-native IaC.**

**Плюсы:**

- **Полная поддержка AWS.**
- **Бесплатно.**
- **Интеграция с AWS-сервисами.**

**Минусы:**

- **Только AWS.** Vendor lock-in.
- **JSON/YAML verbose.**
- **Медленнее развивается.**

**Когда использовать:** если только AWS и не нужен multi-cloud.

### 🎯 Crossplane

**Kubernetes-native IaC.**

**Как работает:** ресурсы облака описываются как Custom Resources в Kubernetes.

```yaml
apiVersion: rds.aws.upbound.io/v1beta1
kind: Instance
metadata:
  name: my-postgres
spec:
  forProvider:
    region: us-west-2
    instanceClass: db.t3.medium
    engine: postgres
    engineVersion: "16.1"
    allocatedStorage: 100
```

**Плюсы:**

- **Kubernetes-native.** Единый workflow.
- **GitOps-friendly.**
- **Continuous reconciliation.**

**Минусы:**

- **Требует Kubernetes.**
- **Моложе.**
- **Меньше провайдеров.**

**Когда использовать:** если всё в Kubernetes и хочешь единый workflow.

### 📊 Когда что использовать

| Сценарий | Инструмент |
|:---|:---|
| **Multi-cloud, стандарт** | Terraform |
| **Open-source, лицензия важна** | OpenTofu |
| **Команда любит Go/Python/TS** | Pulumi |
| **Только AWS** | CloudFormation или CDK |
| **Всё в Kubernetes** | Crossplane |
| **Конфигурация серверов** | Ansible |

**Рекомендация:** **Terraform** для большинства случаев. Много примеров, документации, сообщества.

### 🔬 Практика: установка

```bash
# macOS
brew install terraform

# Linux (через tfenv — менеджер версий)
brew install tfenv
tfenv install 1.7.0
tfenv use 1.7.0

# Проверка
terraform version
# Terraform v1.7.0
```

**OpenTofu:**

```bash
brew install opentofu
tofu version
```

**Использование:** почти идентично Terraform. Просто `tofu` вместо `terraform`.

### 💡 Практика: как выбирать инструмент

**✅ ОБЯЗАТЕЛЬНО:**

1. **Terraform для multi-cloud.**
2. **CloudFormation для AWS-only.**
3. **Ansible для конфигурации серверов.**

**👍 СТОИТ:**

4. **OpenTofu** если важна лицензия.
5. **Pulumi** если команда предпочитает Go/Python/TS.

**❌ НЕ ДЕЛАЙ:**

6. **Не смешивай инструменты** без необходимости.
7. **Не используй CloudFormation, если планируешь multi-cloud.**

### Где мы сейчас

Мы разобрали альтернативы. Теперь — **основы Terraform**.

---

## 18.3 Providers, resources, data sources

### 🔌 Проблема: как описать инфраструктуру

Terraform-код состоит из нескольких типов блоков:

- **Providers** — плагины для облаков.
- **Resources** — создаваемые объекты.
- **Data sources** — чтение существующих объектов.
- **Variables** — входные параметры.
- **Outputs** — выходные значения.

### 📊 Providers

**Provider** — плагин, который позволяет Terraform работать с API облака.

```hcl
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
  }
}

provider "aws" {
  region = "us-west-2"
  
  default_tags {
    tags = {
      Environment = "production"
      ManagedBy   = "terraform"
    }
  }
}
```

**Ключевое:**

- `required_version` — минимальная версия Terraform.
- `required_providers` — какие провайдеры нужны.
- `provider` — конфигурация провайдера.

**Популярные providers:**

| Provider | Для чего |
|:---|:---|
| `hashicorp/aws` | AWS |
| `hashicorp/google` | GCP |
| `hashicorp/azurerm` | Azure |
| `hashicorp/kubernetes` | Kubernetes |
| `hashicorp/helm` | Helm |
| `cloudflare/cloudflare` | Cloudflare |
| `integrations/github` | GitHub |
| `hashicorp/vault` | Vault |

### 🎯 Resources

**Resource** — объект, который Terraform создаёт и управляет.

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  
  tags = {
    Name = "main-vpc"
  }
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id      # ссылка на другой ресурс
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-west-2a"
  
  tags = {
    Name = "public-subnet"
  }
}
```

**Синтаксис:**

```hcl
resource "<TYPE>" "<NAME>" {
  <ARGUMENTS>
}
```

- **TYPE** — тип ресурса (например, `aws_vpc`, `aws_instance`).
- **NAME** — локальное имя (используется в коде).
- **ARGUMENTS** — параметры.

**Ссылки на ресурсы:**

```hcl
aws_vpc.main.id          # ID созданного VPC
aws_subnet.public.id     # ID подсети
aws_instance.web.public_ip  # публичный IP
```

**Зависимости:**

Terraform автоматически строит граф зависимостей. Если `aws_subnet.public` ссылается на `aws_vpc.main.id`, Terraform сначала создаст VPC, потом подсеть.

**Явные зависимости:**

```hcl
resource "aws_instance" "web" {
  # ...
  depends_on = [aws_iam_role_policy.example]
}
```

### 🎯 Data Sources

**Data source** — **чтение** существующего объекта (не создание).

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id      # используем из data source
  instance_type = "t3.micro"
}
```

**Синтаксис:**

```hcl
data "<TYPE>" "<NAME>" {
  <FILTERS>
}
```

**Использование:**

```hcl
data.aws_ami.ubuntu.id
data.aws_vpc.existing.id
data.aws_availability_zones.available.names
```

**Когда использовать:**

- **Получить ID существующего VPC.**
- **Найти последний AMI.**
- **Получить список зон доступности.**
- **Прочитать secrets из Vault.**

### 🎯 Полный пример

```hcl
# providers.tf
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# variables.tf
variable "aws_region" {
  type    = string
  default = "us-west-2"
}

variable "environment" {
  type    = string
  default = "dev"
}

# main.tf
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  
  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${var.environment}-public-${count.index + 1}"
  }
}

# outputs.tf
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "ID созданного VPC"
}

output "subnet_ids" {
  value       = aws_subnet.public[*].id
  description = "IDs публичных подсетей"
}
```

### 🔬 Практика: первая конфигурация

```bash
# 1. Создать директорию
mkdir terraform-demo && cd terraform-demo

# 2. Создать main.tf
cat > main.tf <<'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-west-2"
}

resource "aws_s3_bucket" "demo" {
  bucket = "my-terraform-demo-bucket-unique-12345"
  
  tags = {
    Name        = "Demo bucket"
    Environment = "dev"
  }
}
EOF

# 3. Инициализировать
terraform init
# Initializing the backend...
# Initializing provider plugins...
# - Finding hashicorp/aws versions matching "~> 5.0"...
# - Installing hashicorp/aws v5.30.0...
# Terraform has been successfully initialized!

# 4. Plan (проверить, что будет создано)
terraform plan

# 5. Apply
terraform apply
# Введи yes

# 6. Посмотреть state
terraform state list
# aws_s3_bucket.demo

# 7. Удалить
terraform destroy
# Введи yes
```

### 💡 Практика: как правильно писать конфигурацию

**✅ ОБЯЗАТЕЛЬНО:**

1. **Фиксировать версии провайдеров.**
2. **Использовать variables для параметров.**
3. **Использовать outputs для важных значений.**
4. **Разделять файлы:** `main.tf`, `variables.tf`, `outputs.tf`.

**👍 СТОИТ:**

5. **default_tags в provider** для автоматических тегов.
6. **data sources** для чтения существующих ресурсов.
7. **Комментарии** для неочевидных решений.

**❌ НЕ ДЕЛАЙ:**

8. **Не хардкодь значения.** Используй variables.
9. **Не используй `latest` для версий провайдеров.**
10. **Не пиши всё в один файл** — разделяй.

### Где мы сейчас

Мы разобрали основы Terraform. Теперь — **state** — самая важная и опасная часть.

---

## 18.4 State: сила и ответственность

### 🔌 Проблема: как Terraform помнит, что создал

Ты создал S3-бакет. Terraform создал его через API AWS. Теперь ты хочешь изменить бакет. **Как Terraform знает, какой именно бакет его?**

**Ответ: state file.**

### 📊 Что такое state

**State** — файл `terraform.tfstate`, в котором Terraform хранит:

- **Какие ресурсы созданы** (mapping из кода в реальные ресурсы).
- **Их атрибуты** (ID, ARN, endpoint).
- **Метаданные** (зависимости, версии).

**Пример state:**

```json
{
  "version": 4,
  "terraform_version": "1.7.0",
  "resources": [
    {
      "mode": "managed",
      "type": "aws_s3_bucket",
      "name": "demo",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "id": "my-terraform-demo-bucket-unique-12345",
            "arn": "arn:aws:s3:::my-terraform-demo-bucket-unique-12345",
            "bucket": "my-terraform-demo-bucket-unique-12345",
            "region": "us-west-2",
            "tags": {
              "Name": "Demo bucket",
              "Environment": "dev"
            }
          }
        }
      ]
    }
  ]
}
```

**Что содержит:**

- Тип ресурса (`aws_s3_bucket`).
- Имя в коде (`demo`).
- ID в облаке (`my-terraform-demo-bucket-unique-12345`).
- Все атрибуты.

### 🎯 Зачем нужен state

**1. Mapping.**

Соответствие между кодом и реальными ресурсами.

**2. Метаданные.**

Terraform знает, какие ресурсы зависят от каких.

**3. Производительность.**

Terraform не запрашивает все ресурсы из облака при каждом `plan` — использует state.

**4. Diff.**

Сравнивает желаемое (код) с текущим (state) → показывает изменения.

### 🎯 Формат state

**State — JSON-файл.** По умолчанию — `terraform.tfstate` в текущей директории.

**Проблемы локального state:**

- **Не командный.** Каждый разработчик имеет свой state. Конфликты.
- **Не безопасный.** Может содержать secrets (пароли БД, ключи).
- **Легко потерять.** Удалил файл — потерял управление инфраструктурой.
- **Нет блокировок.** Два `apply` одновременно → конфликт.

**Решение:** remote backend.

### 🎯 Что опасно в state

**1. State содержит секреты.**

Пароли БД, API-ключи, токены — всё это в state **в открытом виде**.

**Правило:** **не коммить state в Git.** Никогда.

**2. State нельзя редактировать вручную.**

Можно, но **очень опасно**. Легко испортить.

**3. Потеря state — потеря управления.**

Если state потерян, Terraform не знает о существующих ресурсах. Он попытается создать их заново → ошибки.

**Решение:** remote backend + backups.

**4. Дрейф (drift).**

Если кто-то изменил ресурс вручную (в консоли AWS), state не знает об этом. `terraform plan` покажет diff, но `apply` может перезаписать.

**Решение:** регулярные `plan`, запрет ручных изменений.

### 🎯 Drift detection

**Drift** — расхождение между state и реальным состоянием.

**Как обнаружить:**

```bash
terraform plan -refresh-only
```

**Что произойдёт:** Terraform прочитает реальное состояние и покажет diff.

**Как исправить:**

```bash
# Обновить state (принять изменения)
terraform apply -refresh-only

# Или откатить изменения в облаке
terraform apply
```

### 🎯 Backup state

**Даже с remote backend — делай бэкапы.**

**Для S3:**

- **Versioning** на bucket с state.
- **Replication** в другой регион.

**Для Terraform Cloud:**

- Автоматические backups.

### 🎯 Чувствительные данные в state

**Проблема:** state содержит secrets в открытом виде.

**Решения:**

1. **Шифрование backend.** S3 + KMS, GCS + CMEK.
2. **Не хранить secrets в state.** Использовать `data` sources (Vault) вместо `resource`.
3. **Ограничить доступ к state.**

**Пример:**

```hcl
# ❌ Плохо: пароль в state
resource "aws_db_instance" "postgres" {
  password = "SuperSecret123"
}

# ✅ Хорошо: пароль из Vault
data "vault_generic_secret" "db_password" {
  path = "secret/db/password"
}

resource "aws_db_instance" "postgres" {
  password = data.vault_generic_secret.db_password.data["password"]
}
```

**Но:** даже с data source, значение может попасть в state (в computed полях). Использовать `sensitive = true` в variables.

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

**Что даёт:** Terraform не покажет значение в `plan`/`apply` output.

### 🎯 Команды для работы со state

```bash
# Список ресурсов в state
terraform state list

# Показать ресурс
terraform state show aws_s3_bucket.demo

# Удалить из state (не из облака!)
terraform state rm aws_s3_bucket.demo

# Переместить (переименование)
terraform state mv aws_s3_bucket.old aws_s3_bucket.new

# Импортировать существующий
terraform import aws_s3_bucket.demo my-bucket-name

# Pull/push (для remote)
terraform state pull > state.json
terraform state push state.json
```

### 🔬 Практика: state

```bash
# 1. Создать ресурс
terraform apply

# 2. Посмотреть state
terraform state list
# aws_s3_bucket.demo

terraform state show aws_s3_bucket.demo
# Покажет все атрибуты

# 3. Посмотреть локальный state file
cat terraform.tfstate

# 4. Удалить из state (без удаления в облаке)
terraform state rm aws_s3_bucket.demo

# 5. Проверить — terraform не знает о бакете
terraform plan
# Plan: 1 to add, 0 to change, 0 to destroy.  ← попытается создать заново!

# 6. Импортировать обратно
terraform import aws_s3_bucket.demo my-terraform-demo-bucket-unique-12345

# 7. Проверить
terraform plan
# No changes. Your infrastructure matches the configuration.
```

### 💡 Практика: как правильно работать со state

**✅ ОБЯЗАТЕЛЬНО:**

1. **Remote backend для командной работы.**
2. **Шифрование state.**
3. **Versioning + backups.**
4. **Блокировки (locking).**

**👍 СТОИТ:**

5. **Не коммить state в Git.**
6. **Ограничить доступ к state.**
7. **Регулярные `plan` для drift detection.**
8. **`sensitive = true` для секретов.**

**❌ НЕ ДЕЛАЙ:**

9. **Не редактируй state вручную.**
10. **Не коммить state в Git.**
11. **Не храни secrets в коде** без sensitive.
12. **Не удаляй state без понимания.**

### Где мы сейчас

Мы разобрали state. Теперь — **remote backend**.

---

## 18.5 Remote backend: командная работа

### 🔌 Проблема: локальный state не масштабируется

Локальный `terraform.tfstate` работает для одного разработчика. Для команды — нет:

- **Конфликты.** Каждый имеет свой state.
- **Потеря.** Если у разработчика удалили ноутбук — state потерян.
- **Нет блокировок.** Два `apply` одновременно → конфликт.
- **Нет бэкапов.**
- **Секреты в открытом виде.**

**Решение:** remote backend.

### 📊 Что такое backend

**Backend** — где хранится state. По умолчанию — локально. Можно настроить **remote**.

**Популярные backend'ы:**

| Backend | Где | Блокировки |
|:---|:---|:---|
| **local** | Локально | Нет |
| **s3** | AWS S3 | DynamoDB |
| **gcs** | GCS | Встроенные |
| **azurerm** | Azure Blob | Встроенные |
| **terraform cloud** | Terraform Cloud | Встроенные |
| **consul** | Consul | Встроенные |
| **postgres** | PostgreSQL | Advisory locks |

### 🎯 S3 Backend

**Самый популярный.**

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-west-2:123456789:key/abc123"
    dynamodb_table = "terraform-locks"
  }
}
```

**Что нужно:**

1. **S3 bucket** для state.
2. **DynamoDB table** для блокировок.
3. **KMS key** для шифрования.

**Создать S3 bucket:**

```bash
aws s3 mb s3://my-terraform-state --region us-west-2
aws s3api put-bucket-versioning \
  --bucket my-terraform-state \
  --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption \
  --bucket my-terraform-state \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'
```

**Создать DynamoDB table:**

```bash
aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-west-2
```

**Что произойдёт:**

- State хранится в S3.
- Блокировки — в DynamoDB.
- Шифрование — через KMS или SSE.

### 🎯 GCS Backend

```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "prod"
  }
}
```

**Что нужно:** GCS bucket с версионированием.

### 🎯 Azure Backend

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state"
    storage_account_name = "tfstate"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

### 🎯 Terraform Cloud

**Managed backend** от HashiCorp.

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "my-app-prod"
    }
  }
}
```

**Что даёт:**

- **Managed state.**
- **Блокировки.**
- **UI.**
- **CI/CD встроенный.**
- **VCS integration.**
- **Бесплатно** для малых команд (до 5 пользователей).

### 🎯 Инициализация с backend

```bash
# Первый раз
terraform init

# Если backend менялся
terraform init -migrate-state

# Переконфигурация
terraform init -reconfigure
```

**Что произойдёт:** Terraform скачает state из backend в память (не на диск).

### 🎯 Блокировки

**Locking** — механизм, предотвращающий одновременное выполнение `apply`.

**Как работает:**

1. Terraform запрашивает блокировку в backend.
2. Если блокировка свободна — устанавливает.
3. Другой Terraform не может получить блокировку → ждёт или падает.
4. После `apply` — блокировка снимается.

**Если Terraform упал:**

```bash
# Force unlock (осторожно!)
terraform force-unlock <lock-id>
```

**Когда использовать:** только если уверен, что предыдущий Terraform не работает.

### 🎯 Работа в команде

**Типичный workflow:**

1. **Разработчик 1** клонирует репозиторий.
2. **Разработчик 1** делает `terraform init` — state скачивается из S3.
3. **Разработчик 1** делает `terraform plan` — видит изменения.
4. **Разработчик 1** делает `terraform apply` — устанавливает блокировку, применяет, снимает.
5. **Разработчик 2** в это же время пытается `apply` — получает ошибку «state locked».
6. **Разработчик 2** ждёт или работает с другим workspace.

### 🔬 Практика: remote backend

```bash
# 1. Создать S3 bucket (уникальное имя)
aws s3 mb s3://my-terraform-state-$(date +%s)

# 2. Создать DynamoDB table
aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

# 3. В Terraform-коде настроить backend
cat > backend.tf <<'EOF'
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-1234567890"
    key            = "demo/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
EOF

# 4. Инициализация с миграцией
terraform init -migrate-state

# 5. Проверить, что state в S3
aws s3 ls s3://my-terraform-state-1234567890/demo/

# 6. Проверить блокировки
terraform apply  # в одном терминале
# в другом:
terraform apply  # получит "Error: Error acquiring the state lock"
```

### 💡 Практика: как правильно настроить backend

**✅ ОБЯЗАТЕЛЬНО:**

1. **Remote backend для команды.**
2. **Версионирование S3 bucket.**
3. **Шифрование (KMS, SSE).**
4. **DynamoDB для блокировок.**

**👍 СТОИТ:**

5. **Отдельный state на окружение** (`dev/`, `prod/`).
6. **Terraform Cloud** для простоты.
7. **Ограничить доступ к S3 bucket** через IAM.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй один state для всех окружений.**
9. **Не давай всем полный доступ к state.**
10. **Не коммить state в Git.**

### Где мы сейчас

Мы разобрали remote backend. Теперь — **modules**.

---

## 18.6 Modules: переиспользование кода

### 🔌 Проблема: дублирование кода

Ты создал VPC. Теперь нужно создать такое же VPC для другого проекта. Копируешь код. Меняешь имена. Через полгода — 10 копий, все разные.

**Решение:** modules.

### 📊 Что такое module

**Module** — переиспользуемый набор Terraform-конфигураций. Как функция в программировании: принимает variables, возвращает outputs.

**Типы modules:**

- **Root module** — директория, где ты запускаешь `terraform apply`.
- **Child module** — импортируется в root.

### 🎯 Создание модуля

**Структура:**

```
modules/
└── vpc/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── README.md
```

**modules/vpc/main.tf:**

```hcl
variable "name" {
  type = string
}

variable "cidr_block" {
  type    = string
  default = "10.0.0.0/16"
}

variable "azs" {
  type = list(string)
}

data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_support   = true
  enable_dns_hostnames = true
  
  tags = {
    Name = var.name
  }
}

resource "aws_subnet" "public" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.cidr_block, 8, count.index)
  availability_zone = var.azs[count.index]
  
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${var.name}-public-${count.index + 1}"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "${var.name}-igw"
  }
}
```

**modules/vpc/variables.tf:**

```hcl
variable "name" {
  description = "Имя VPC"
  type        = string
}

variable "cidr_block" {
  description = "CIDR блок VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "azs" {
  description = "Список зон доступности"
  type        = list(string)
}
```

**modules/vpc/outputs.tf:**

```hcl
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "ID VPC"
}

output "subnet_ids" {
  value       = aws_subnet.public[*].id
  description = "IDs подсетей"
}

output "igw_id" {
  value       = aws_internet_gateway.main.id
  description = "ID Internet Gateway"
}
```

### 🎯 Использование модуля

**В root module:**

```hcl
module "vpc_prod" {
  source = "./modules/vpc"
  
  name       = "prod"
  cidr_block = "10.0.0.0/16"
  azs        = ["us-west-2a", "us-west-2b", "us-west-2c"]
}

module "vpc_dev" {
  source = "./modules/vpc"
  
  name       = "dev"
  cidr_block = "10.1.0.0/16"
  azs        = ["us-west-2a", "us-west-2b"]
}
```

**Использование outputs:**

```hcl
resource "aws_instance" "web" {
  subnet_id = module.vpc_prod.subnet_ids[0]
  # ...
}
```

### 🎯 Источники модулей

**Local:**

```hcl
module "vpc" {
  source = "./modules/vpc"
}
```

**Git:**

```hcl
module "vpc" {
  source = "git::https://github.com/myorg/terraform-modules.git//vpc?ref=v1.0.0"
}
```

**Terraform Registry:**

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-west-2a", "us-west-2b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = false
}
```

**OCI:**

```hcl
module "vpc" {
  source  = "oci://myregistry.com/terraform/vpc"
  version = "1.0.0"
}
```

### 🎯 Публичные модули

**Terraform Registry** — официальный реестр модулей.

**Популярные:**

| Модуль | Для чего |
|:---|:---|
| `terraform-aws-modules/vpc/aws` | VPC |
| `terraform-aws-modules/eks/aws` | EKS |
| `terraform-aws-modules/rds/aws` | RDS |
| `terraform-aws-modules/s3-bucket/aws` | S3 |
| `terraform-google-modules/kubernetes-engine/google` | GKE |

**Использование:**

```bash
# Поиск
terraform search module eks
```

Или на [registry.terraform.io](https://registry.terraform.io).

### 🎯 Версионирование модулей

**Всегда фиксируй версию:**

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.0"        # ✅ конкретная версия
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"       # ✅ мажорная версия
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "latest"       # ❌ непредсказуемо
}
```

### 🎯 Лучшие практики для модулей

**1. Один модуль — одна задача.**

Плохо: модуль «вся инфраструктура».
Хорошо: модуль «VPC», модуль «RDS», модуль «EKS».

**2. Variables с defaults.**

```hcl
variable "instance_type" {
  type    = string
  default = "t3.micro"
}
```

**3. Outputs для важных значений.**

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}
```

**4. README с примерами.**

**5. Версионирование.**

**6. Tests.**

```hcl
# tests/vpc_test.go
func TestVpcModule(t *testing.T) {
    // terratest
}
```

### 🔬 Практика: модуль

```bash
# 1. Создать модуль
mkdir -p modules/vpc
cd modules/vpc

cat > main.tf <<'EOF'
variable "name" {
  type = string
}

variable "cidr_block" {
  type    = string
  default = "10.0.0.0/16"
}

resource "aws_vpc" "main" {
  cidr_block = var.cidr_block
  
  tags = {
    Name = var.name
  }
}

output "vpc_id" {
  value = aws_vpc.main.id
}
EOF

# 2. Использовать в root
cd ../..
cat > main.tf <<'EOF'
provider "aws" {
  region = "us-west-2"
}

module "vpc_prod" {
  source = "./modules/vpc"
  
  name       = "prod"
  cidr_block = "10.0.0.0/16"
}

module "vpc_dev" {
  source = "./modules/vpc"
  
  name       = "dev"
  cidr_block = "10.1.0.0/16"
}

output "prod_vpc_id" {
  value = module.vpc_prod.vpc_id
}

output "dev_vpc_id" {
  value = module.vpc_dev.vpc_id
}
EOF

# 3. Init (скачает модули)
terraform init

# 4. Plan
terraform plan

# 5. Apply
terraform apply
```

### 💡 Практика: как правильно писать модули

**✅ ОБЯЗАТЕЛЬНО:**

1. **Фиксировать версии модулей.**
2. **Variables с defaults.**
3. **Outputs для важных значений.**
4. **README с примерами.**

**👍 СТОИТ:**

5. **Отдельный репозиторий для модулей.**
6. **CI для тестирования модулей.**
7. **Terratest** для тестов.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй `latest`.**
9. **Не делай модуль «на всё».**
10. **Не забывай про version в source.**

### Где мы сейчас

Мы разобрали modules. Теперь — **variables, outputs, locals**.

---

## 18.7 Variables, outputs, locals

### 🔌 Проблема: как параметризовать конфигурацию

Ты хочешь использовать одну конфигурацию для dev и prod. Отличается:

- Регион.
- Размер инстансов.
- CIDR блоки.
- Количество реплик.

**Решение:** variables.

### 📊 Variables

**Input variables** — параметры конфигурации.

```hcl
variable "aws_region" {
  description = "AWS регион"
  type        = string
  default     = "us-west-2"
}

variable "instance_type" {
  description = "Тип инстанса"
  type        = string
  default     = "t3.micro"
}

variable "replicas" {
  description = "Количество реплик"
  type        = number
  default     = 2
}

variable "tags" {
  description = "Теги"
  type        = map(string)
  default     = {}
}

variable "availability_zones" {
  description = "Зоны доступности"
  type        = list(string)
  default     = ["us-west-2a", "us-west-2b"]
}
```

**Типы:**

| Тип | Пример |
|:---|:---|
| `string` | `"us-west-2"` |
| `number` | `3` |
| `bool` | `true` |
| `list(string)` | `["a", "b"]` |
| `map(string)` | `{key = "value"}` |
| `object({...})` | `{name = "x", age = 30}` |

**Validation:**

```hcl
variable "environment" {
  type = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

**Sensitive:**

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

### 🎯 Передача values

**1. Через `-var`:**

```bash
terraform apply -var="instance_type=t3.large"
terraform apply -var="environment=prod"
```

**2. Через `-var-file`:**

```bash
terraform apply -var-file="prod.tfvars"
```

**3. Через `terraform.tfvars` (автоматически):**

```hcl
# terraform.tfvars
instance_type = "t3.large"
environment   = "prod"
```

**4. Через `*.auto.tfvars` (автоматически):**

```hcl
# prod.auto.tfvars
instance_type = "t3.large"
```

**Порядок приоритета:**

1. `-var` / `-var-file` (высший).
2. `*.auto.tfvars` (алфавитный порядок).
3. `terraform.tfvars`.
4. Environment variables (`TF_VAR_*`).
5. `default` в variable.

**Environment variables:**

```bash
export TF_VAR_instance_type="t3.large"
terraform apply
```

### 🎯 Outputs

**Output variables** — значения, доступные после apply.

```hcl
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "ID VPC"
}

output "db_endpoint" {
  value       = aws_db_instance.postgres.endpoint
  sensitive   = true
}

output "instance_ips" {
  value = aws_instance.web[*].public_ip
}
```

**Просмотр:**

```bash
terraform output
# vpc_id = "vpc-abc123"
# db_endpoint = <sensitive>

terraform output vpc_id
# "vpc-abc123"

terraform output -json
# {"vpc_id": {"value": "vpc-abc123", "type": "string"}}

terraform output -raw vpc_id
# vpc-abc123 (без кавычек)
```

**Использование в других конфигурациях:**

```hcl
# В другом root module
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "vpc/terraform.tfstate"
    region = "us-west-2"
  }
}

resource "aws_instance" "web" {
  subnet_id = data.terraform_remote_state.vpc.outputs.subnet_ids[0]
}
```

### 🎯 Locals

**Locals** — локальные переменные для вычислений.

```hcl
locals {
  environment = var.environment
  name_prefix = "${var.project}-${var.environment}"
  
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
  }
  
  # Сложные вычисления
  is_production = var.environment == "prod"
  
  instance_count = local.is_production ? 3 : 1
}

resource "aws_instance" "web" {
  count = local.instance_count
  
  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-web-${count.index}"
    }
  )
}
```

**Когда использовать:**

- **Промежуточные вычисления.**
- **Общие теги.**
- **Условия.**

### 🎯 Условия

**Тернарный оператор:**

```hcl
resource "aws_instance" "web" {
  instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"
}
```

**Условное создание:**

```hcl
resource "aws_cloudwatch_log_group" "app" {
  count = var.enable_logging ? 1 : 0
  
  name = "/aws/myapp"
}
```

**Использование:**

```hcl
# Если count = 0, то log_group_name = null
log_group_name = var.enable_logging ? aws_cloudwatch_log_group.app[0].name : null
```

### 🎯 Функции

Terraform имеет много встроенных функций:

**Строки:**

```hcl
upper("hello")           # "HELLO"
lower("HELLO")           # "hello"
format("Hello, %s", var.name)  # "Hello, John"
join(",", var.list)      # "a,b,c"
split(",", "a,b,c")      # ["a", "b", "c"]
replace("hello", "l", "L")  # "heLLo"
trimspace(" hi ")        # "hi"
substr("hello", 0, 3)    # "hel"
```

**Числа:**

```hcl
max(1, 2, 3)            # 3
min(1, 2, 3)            # 1
abs(-5)                 # 5
ceil(1.5)               # 2
floor(1.5)              # 1
```

**Списки:**

```hcl
length(var.list)        # длина
concat(list1, list2)    # объединить
element(var.list, 0)    # элемент
slice(var.list, 0, 2)   # срез
contains(var.list, "x") # содержит
distinct(var.list)      # уникальные
flatten(list_of_lists)  # развернуть
```

**Maps:**

```hcl
keys(var.map)           # ключи
values(var.map)         # значения
merge(map1, map2)       # объединить
lookup(var.map, "key", "default")  # безопасный доступ
```

**Файлы:**

```hcl
file("path/to/file")    # содержимое файла
filebase64("path")      # base64
templatefile("path", {var = "value"})  # шаблон
```

**Дата:**

```hcl
timestamp()             # RFC3339
formatdate("YYYY-MM-DD", timestamp())
```

**CIDR:**

```hcl
cidrsubnet("10.0.0.0/16", 8, 1)    # "10.0.1.0/24"
cidrhost("10.0.0.0/24", 5)         # "10.0.0.5"
```

### 🎯 Пример: полная конфигурация

```hcl
# variables.tf
variable "project" {
  type        = string
  description = "Имя проекта"
}

variable "environment" {
  type        = string
  description = "Окружение"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "aws_region" {
  type    = string
  default = "us-west-2"
}

variable "db_password" {
  type      = string
  sensitive = true
}

# locals.tf
locals {
  name_prefix = "${var.project}-${var.environment}"
  
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
  }
  
  is_production = var.environment == "prod"
  
  instance_type = local.is_production ? "t3.large" : "t3.micro"
  
  replicas = local.is_production ? 3 : 1
}

# main.tf
resource "aws_db_instance" "postgres" {
  identifier = "${local.name_prefix}-postgres"
  
  engine         = "postgres"
  engine_version = "16.1"
  instance_class = local.is_production ? "db.t3.medium" : "db.t3.micro"
  
  allocated_storage = local.is_production ? 100 : 20
  storage_encrypted = true
  
  db_name  = var.project
  username = "postgres"
  password = var.db_password
  
  multi_az                = local.is_production
  backup_retention_period = local.is_production ? 30 : 7
  
  tags = local.common_tags
}

# outputs.tf
output "db_endpoint" {
  value       = aws_db_instance.postgres.endpoint
  description = "Endpoint PostgreSQL"
}

output "db_name" {
  value       = aws_db_instance.postgres.db_name
  description = "Имя БД"
}
```

### 🔬 Практика: variables и outputs

```bash
# 1. Создать variables.tf
cat > variables.tf <<'EOF'
variable "environment" {
  type        = string
  description = "Окружение"
  default     = "dev"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}

variable "instance_count" {
  type    = number
  default = 1
}

variable "tags" {
  type    = map(string)
  default = {}
}
EOF

# 2. Создать locals.tf
cat > locals.tf <<'EOF'
locals {
  name_prefix = "myapp-${var.environment}"
  
  common_tags = merge(
    var.tags,
    {
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  )
  
  is_prod = var.environment == "prod"
}
EOF

# 3. Использовать в main.tf
cat >> main.tf <<'EOF'

resource "aws_s3_bucket" "app" {
  bucket = "${local.name_prefix}-app-bucket"
  tags   = local.common_tags
}
EOF

# 4. Outputs
cat > outputs.tf <<'EOF'
output "bucket_name" {
  value = aws_s3_bucket.app.id
}

output "environment" {
  value = var.environment
}

output "is_production" {
  value = local.is_prod
}
EOF

# 5. Разные окружения
terraform apply -var="environment=dev"
terraform output

terraform apply -var="environment=prod" -var="instance_count=3"
terraform output
```

### 💡 Практика: как правильно использовать variables

**✅ ОБЯЗАТЕЛЬНО:**

1. **Variables для параметров окружения.**
2. **Locals для вычислений.**
3. **Outputs для важных значений.**
4. **Валидация для критичных values.**

**👍 СТОИТ:**

5. **`sensitive = true` для секретов.**
6. **`terraform.tfvars` для defaults окружения.**
7. **Функции для сложных вычислений.**

**❌ НЕ ДЕЛАЙ:**

8. **Не хардкодь значения.** Используй variables.
9. **Не используй secrets без sensitive.**
10. **Не забывай про validation.**

### Где мы сейчас

Мы разобрали variables. Теперь — **workspaces**.

---

## 18.8 Workspaces и environments

### 🔌 Проблема: как управлять окружениями

У тебя есть dev, staging, prod. Они похожи, но отличаются:

- Размер инстансов.
- Количество реплик.
- Регионы.
- Отдельные state.

**Варианты:**

1. **Разные директории** для каждого окружения.
2. **Workspaces.**
3. **Разные репозитории.**

### 📊 Workspaces

**Workspace** — изолированный state внутри одного backend.

```bash
# Список workspaces
terraform workspace list
# * default

# Создать
terraform workspace new dev
terraform workspace new prod

# Переключить
terraform workspace select prod

# Текущий
terraform workspace show
# prod

# Удалить
terraform workspace delete dev
```

**Использование в коде:**

```hcl
locals {
  environment = terraform.workspace
  
  instance_type = local.environment == "prod" ? "t3.large" : "t3.micro"
}

resource "aws_instance" "web" {
  instance_type = local.instance_type
  
  tags = {
    Environment = terraform.workspace
  }
}
```

**State:**

Каждый workspace имеет свой state:

```
s3://my-terraform-state/env:/dev/terraform.tfstate
s3://my-terraform-state/env:/staging/terraform.tfstate
s3://my-terraform-state/env:/prod/terraform.tfstate
```

### 🎯 Когда использовать workspaces

**✅ Использовать:**

- **Одинаковая инфраструктура** для разных окружений.
- **Разные окружения** на одном backend.
- **Быстрое переключение.**

**❌ Не использовать:**

- **Сильно разная инфраструктура** (dev — маленькая, prod — большая).
- **Разные команды** для разных окружений.
- **Разные регионы** (state в одном регионе).

### 🎯 Паттерн: директории вместо workspaces

**Более гибкий подход** — отдельная директория для каждого окружения.

```
terraform/
├── modules/
│   ├── vpc/
│   ├── rds/
│   └── eks/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── backend.tf
│   │   ├── terraform.tfvars
│   │   └── outputs.tf
│   ├── staging/
│   │   ├── main.tf
│   │   ├── backend.tf
│   │   ├── terraform.tfvars
│   │   └── outputs.tf
│   └── prod/
│       ├── main.tf
│       ├── backend.tf
│       ├── terraform.tfvars
│       └── outputs.tf
```

**Преимущества:**

- **Изоляция.** Каждое окружение — свой root module.
- **Разные версии Terraform** (если нужно).
- **Разные провайдеры.**
- **Разные backend'ы** (можно разные регионы).

**environment/dev/main.tf:**

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "dev/terraform.tfstate"
    region = "us-west-2"
  }
}

module "vpc" {
  source = "../../modules/vpc"
  
  name       = "dev"
  cidr_block = "10.0.0.0/16"
}

module "rds" {
  source = "../../modules/rds"
  
  name           = "dev"
  instance_class = "db.t3.micro"
  multi_az       = false
}
```

**environment/prod/main.tf:**

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-west-2"
  }
}

module "vpc" {
  source = "../../modules/vpc"
  
  name       = "prod"
  cidr_block = "10.1.0.0/16"
}

module "rds" {
  source = "../../modules/rds"
  
  name           = "prod"
  instance_class = "db.t3.medium"
  multi_az       = true
}
```

### 🎯 Terragrunt

**Terragrunt** — обёртка над Terraform для уменьшения дублирования.

**Проблема:** в каждой директории окружения — одинаковый backend-блок.

**Решение:** Terragrunt.

**terragrunt.hcl (root):**

```hcl
remote_state {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "${path_relative_to_include()}/terraform.tfstate"
    region = "us-west-2"
  }
}
```

**environments/dev/terragrunt.hcl:**

```hcl
include {
  path = find_in_parent_folders()
}

terraform {
  source = "../../modules/vpc"
}

inputs = {
  name       = "dev"
  cidr_block = "10.0.0.0/16"
}
```

**environments/prod/terragrunt.hcl:**

```hcl
include {
  path = find_in_parent_folders()
}

terraform {
  source = "../../modules/vpc"
}

inputs = {
  name       = "prod"
  cidr_block = "10.1.0.0/16"
}
```

**Запуск:**

```bash
cd environments/dev
terragrunt apply
```

### 🎯 Промоушен между окружениями

**Паттерн:**

1. Изменение в dev.
2. Тестирование.
3. Промоушен в staging.
4. Тестирование.
5. Промоушен в prod.

**Через Git:**

- Изменить `environments/dev/terraform.tfvars`.
- Merge в main.
- CI применяет в dev.
- После тестирования — merge в `staging` branch.
- CI применяет в staging.
- И т.д.

### 🔬 Практика: workspaces

```bash
# 1. Создать workspaces
terraform workspace new dev
terraform workspace new prod

# 2. Использовать в коде
cat > main.tf <<'EOF'
locals {
  environment = terraform.workspace
  instance_type = local.environment == "prod" ? "t3.large" : "t3.micro"
}

resource "aws_s3_bucket" "app" {
  bucket = "myapp-${local.environment}-bucket-12345"
  
  tags = {
    Environment = local.environment
  }
}

output "bucket_name" {
  value = aws_s3_bucket.app.id
}

output "instance_type" {
  value = local.instance_type
}
EOF

# 3. Применить в dev
terraform workspace select dev
terraform apply
terraform output
# bucket_name = "myapp-dev-bucket-12345"
# instance_type = "t3.micro"

# 4. Применить в prod
terraform workspace select prod
terraform apply
terraform output
# bucket_name = "myapp-prod-bucket-12345"
# instance_type = "t3.large"

# 5. Разные states
aws s3 ls s3://my-terraform-state/env:/
```

### 💡 Практика: как правильно управлять окружениями

**✅ ОБЯЗАТЕЛЬНО:**

1. **Отдельный state на окружение.**
2. **Разные переменные для окружений.**
3. **Изоляция — dev не должен влиять на prod.**

**👍 СТОИТ:**

4. **Директории вместо workspaces** для гибкости.
5. **Terragrunt** для уменьшения дублирования.
6. **CI/CD для промоушена.**

**❌ НЕ ДЕЛАЙ:**

7. **Не используй один state для всех окружений.**
8. **Не давай dev-команде доступ к prod-state.**
9. **Не применяй изменения в prod без ревью.**

### Где мы сейчас

Мы разобрали workspaces. Теперь — **безопасная работа: plan, apply, destroy**.

---

## 18.9 plan, apply, destroy: безопасная работа

### 🔌 Проблема: apply может сломать инфраструктуру

`terraform apply` может:

- Создать ресурсы.
- Изменить ресурсы.
- **Удалить ресурсы.**

Удаление — самое опасное. Если случайно удалишь production БД — катастрофа.

**Решение:** безопасный workflow.

### 📊 Правильный workflow

```
1. terraform init       — скачать провайдеры и модули
2. terraform fmt        — форматирование
3. terraform validate   — валидация синтаксиса
4. terraform plan       — посмотреть, что изменится
5. Ревью плана          — глазами или в PR
6. terraform apply      — применить
```

### 🎯 `terraform plan`

**Что делает:** показывает, что изменится.

```bash
terraform plan
```

**Вывод:**

```
Terraform will perform the following actions:

  # aws_s3_bucket.demo will be created
  + resource "aws_s3_bucket" "demo" {
      + bucket = "my-bucket"
      + id     = (known after apply)
      ...
    }

  # aws_instance.web will be updated in-place
  ~ resource "aws_instance" "web" {
      ~ instance_type = "t3.micro" -> "t3.large"
      ...
    }

  # aws_db_instance.old will be destroyed
  - resource "aws_db_instance" "old" {
      - identifier = "old-db" -> null
      ...
    }

Plan: 1 to add, 1 to change, 1 to destroy.
```

**Символы:**

| Символ | Что означает |
|:---|:---|
| `+` | Создание |
| `~` | Изменение (in-place) |
| `-` | Удаление |
| `-/+` | Удаление и создание (replacement) |

**Сохранение плана:**

```bash
terraform plan -out=tfplan
terraform apply tfplan
```

**Что даёт:** применяется **точно тот** план, который ты проверил.

### 🎯 `terraform apply`

**Что делает:** применяет изменения.

```bash
# С интерактивным подтверждением
terraform apply

# С автоматическим подтверждением (для CI/CD)
terraform apply -auto-approve

# С сохранённым планом
terraform apply tfplan
```

**Важно:** в CI/CD используй сохранённый план.

```bash
terraform plan -out=tfplan
# Ревью плана
terraform apply tfplan
```

### 🎯 `terraform destroy`

**Что делает:** удаляет **все** ресурсы.

```bash
terraform destroy
```

**Опасно!** Удаляет всё, что в state.

**Защита:**

```hcl
resource "aws_db_instance" "postgres" {
  # ...
  lifecycle {
    prevent_destroy = true      # запретить удаление
  }
}
```

**Что произойдёт:** `terraform destroy` упадёт с ошибкой, если попытается удалить этот ресурс.

**Также:**

```hcl
resource "aws_instance" "web" {
  # ...
  lifecycle {
    create_before_destroy = true    # создать новый до удаления старого
    ignore_changes        = [ami]   # игнорировать изменения в ami
  }
}
```

### 🎯 Lifecycle

**`create_before_destroy`:** создать новый ресурс до удаления старого.

**Когда использовать:** для ресурсов с downtime (например, ASG).

**`prevent_destroy`:** запретить удаление.

**Когда использовать:** для критичных ресурсов (БД, S3).

**`ignore_changes`:** игнорировать изменения в определённых полях.

**Когда использовать:** когда поле меняется вне Terraform.

**`replace_triggered_by`:** заменить ресурс при изменении другого.

```hcl
lifecycle {
  replace_triggered_by = [aws_security_group.web.id]
}
```

### 🎯 `terraform fmt` и `validate`

**Форматирование:**

```bash
terraform fmt -recursive
```

**Что делает:** приводит код к стандартному стилю.

**Валидация:**

```bash
terraform validate
```

**Что делает:** проверяет синтаксис и ссылки.

### 🎯 `terraform taint` и `untaint`

**Taint** — пометить ресурс как «испорченный». При следующем `apply` он будет пересоздан.

```bash
terraform taint aws_instance.web
terraform apply
```

**Untaint** — снять пометку.

```bash
terraform untaint aws_instance.web
```

**Когда использовать:** если ресурс в некорректном состоянии и нужен пересоздать.

### 🎯 `terraform state` команды

```bash
# Список ресурсов
terraform state list

# Показать ресурс
terraform state show aws_instance.web

# Удалить из state (не из облака)
terraform state rm aws_instance.web

# Переместить (переименование)
terraform state mv aws_instance.old aws_instance.new

# Pull/push
terraform state pull > state.json
terraform state push state.json
```

### 🎯 Безопасный workflow для production

**1. Отдельный аккаунт/регион для prod.**

**2. Ограниченный доступ.**

Только CI/CD может применять в prod.

**3. Обязательное ревью плана.**

Pull Request с выводом `terraform plan`.

**4. `prevent_destroy` для критичных ресурсов.**

**5. Backup state перед apply.**

**6. Применение через CI/CD.**

- `plan` — на PR.
- `apply` — после merge в main.

**7. Уведомления.**

Slack-нотификация о применённых изменениях.

### 🔬 Практика: безопасный workflow

```bash
# 1. Форматирование
terraform fmt

# 2. Валидация
terraform validate

# 3. Plan с сохранением
terraform plan -out=tfplan

# 4. Просмотр плана
terraform show tfplan

# 5. Apply сохранённого плана
terraform apply tfplan

# 6. Защита от destroy
cat >> main.tf <<'EOF'

resource "aws_s3_bucket" "critical" {
  bucket = "my-critical-bucket-12345"
  
  lifecycle {
    prevent_destroy = true
  }
}
EOF

terraform apply

# 7. Попытка destroy — упадёт
terraform destroy
# Error: Instance cannot be destroyed
```

### 💡 Практика: как безопасно работать с Terraform

**✅ ОБЯЗАТЕЛЬНО:**

1. **`terraform plan` перед каждым `apply`.**
2. **Сохранённый план** (`-out=tfplan`).
3. **`prevent_destroy` для БД, S3, критичных ресурсов.**
4. **Ревью плана** перед применением в prod.

**👍 СТОИТ:**

5. **`fmt` и `validate` в CI.**
6. **Уведомления о применённых изменениях.**
7. **Backup state перед apply.**
8. **Отдельный аккаунт для prod.**

**❌ НЕ ДЕЛАЙ:**

9. **Не делай `apply` без `plan`.**
10. **Не делай `destroy` в production без понимания.**
11. **Не давай всем доступ к prod.**
12. **Не применяй в prod без ревью.**

### Где мы сейчас

Мы разобрали безопасный workflow. Теперь — **import**.

---

## 18.10 Import: как подружиться с существующей инфраструктурой

### 🔌 Проблема: ресурсы уже созданы

Ты приходишь в компанию. Инфраструктура уже есть — VPC, RDS, EKS созданы вручную или через CloudFormation. Ты хочешь перейти на Terraform.

**Как импортировать существующие ресурсы?**

**Решение:** `terraform import`.

### 📊 Что такое import

**Import** — добавление существующего ресурса в state Terraform.

**Что происходит:**

1. Ты пишешь Terraform-код для ресурса.
2. Запускаешь `terraform import`.
3. Terraform читает реальный ресурс из облака.
4. Записывает его в state.
5. Дальше Terraform управляет этим ресурсом.

### 🎯 Базовый импорт

**1. Написать код:**

```hcl
resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket"
  
  tags = {
    Name = "Existing bucket"
  }
}
```

**2. Импортировать:**

```bash
terraform import aws_s3_bucket.existing my-existing-bucket
```

**Что произойдёт:**

- Terraform прочитает бакет из AWS.
- Запишет его в state.
- Дальше можно делать `plan` и `apply`.

**3. Проверить:**

```bash
terraform plan
# No changes. Your infrastructure matches the configuration.
```

Если plan показывает изменения — код не полностью соответствует реальному ресурсу. Нужно донастроить код.

### 🎯 Импорт разных ресурсов

**EC2 Instance:**

```bash
terraform import aws_instance.web i-1234567890abcdef0
```

**RDS:**

```bash
terraform import aws_db_instance.postgres my-postgres-db
```

**EKS:**

```bash
terraform import aws_eks_cluster.main my-cluster
```

**IAM Role:**

```bash
terraform import aws_iam_role.app my-role
```

**Security Group:**

```bash
terraform import aws_security_group.web sg-1234567890abcdef0
```

### 🎯 Import Blocks (Terraform 1.5+)

**Новый способ** — декларативный импорт.

**1. Написать import block:**

```hcl
import {
  to = aws_s3_bucket.existing
  id = "my-existing-bucket"
}

resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket"
}
```

**2. План:**

```bash
terraform plan
# Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.
```

**3. Apply:**

```bash
terraform apply
# aws_s3_bucket.existing: Importing... [id=my-existing-bucket]
# aws_s3_bucket.existing: Imported... [id=my-existing-bucket]
```

**Преимущества:**

- **Декларативно.** Не нужно помнить команды.
- **Версионируется в Git.**
- **Планируется как обычные изменения.**

### 🎯 Генерация кода

**`terraform plan -generate-config-out`:**

**1. Import block без resource:**

```hcl
import {
  to = aws_s3_bucket.existing
  id = "my-existing-bucket"
}
```

**2. План с генерацией:**

```bash
terraform plan -generate-config-out=generated.tf
```

**3. Что произойдёт:** Terraform создаст `generated.tf` с кодом ресурса.

```hcl
# generated.tf
resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket"
  
  tags = {
    Name = "Existing bucket"
  }
  # ... все атрибуты
}
```

**4. Просмотри и подредактируй:**

Убери лишнее (computed-поля, defaults).

**5. Apply:**

```bash
terraform apply
```

**Это — самый быстрый способ импорта.** Используй его.

### 🎯 Импорт модулей

**Проблема:** импортировать ресурс в модуль сложнее.

**Решение:** import block с `to = module.<name>.<resource>`.

```hcl
import {
  to = module.vpc.aws_vpc.main
  id = "vpc-1234567890abcdef0"
}
```

### 🎯 Best practices для импорта

**1. Импортируй по одному ресурсу.**

Не всё сразу. Легче отлаживать.

**2. После импорта — `plan`.**

Проверь, что код полностью соответствует реальному ресурсу.

**3. Донастрой код.**

Убери computed-поля, добавь defaults.

**4. Backup state перед импортом.**

**5. Не редактируй state вручную.**

### 🔬 Практика: импорт

```bash
# 1. Создать ресурс вручную через CLI
aws s3 mb s3://my-import-test-bucket-12345

# 2. Написать Terraform-код
cat > main.tf <<'EOF'
provider "aws" {
  region = "us-west-2"
}

import {
  to = aws_s3_bucket.imported
  id = "my-import-test-bucket-12345"
}
EOF

# 3. Генерировать код
terraform init
terraform plan -generate-config-out=generated.tf

# 4. Проверить generated.tf
cat generated.tf

# 5. Apply
terraform apply
# Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.

# 6. Проверить
terraform state list
# aws_s3_bucket.imported

# 7. Plan — не должно быть изменений
terraform plan
# No changes.
```

### 💡 Практика: как правильно импортировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Backup state перед импортом.**
2. **Plan после каждого импорта.**
3. **Импортировать по одному ресурсу.**

**👍 СТОИТ:**

4. **Import blocks (Terraform 1.5+).**
5. **`-generate-config-out` для генерации кода.**
6. **Постепенный переход на Terraform.**

**❌ НЕ ДЕЛАЙ:**

7. **Не редактируй state вручную.**
8. **Не импортируй всё сразу.**
9. **Не забывай про computed-поля** — они не должны быть в коде.

### Где мы сейчас

Мы разобрали импорт. Теперь — **Terraform + Kubernetes**.

---

## 18.11 Terraform + Kubernetes

### 🔌 Проблема: Kubernetes-ресурсы тоже нужно управлять

Ты используешь Terraform для облачной инфраструктуры. Но Kubernetes-ресурсы (namespaces, RBAC, Helm releases) управляются через `kubectl` или Helm.

**Можно ли управлять ими через Terraform?**

**Да.**

### 📊 Kubernetes Provider

**`hashicorp/kubernetes`** — провайдер для K8s-ресурсов.

```hcl
provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = "my-cluster"
}

resource "kubernetes_namespace" "app" {
  metadata {
    name = "myapp"
    labels = {
      environment = "production"
    }
  }
}

resource "kubernetes_config_map" "app" {
  metadata {
    name      = "app-config"
    namespace = kubernetes_namespace.app.metadata[0].name
  }
  
  data = {
    "log_level" = "info"
    "database_host" = "postgres"
  }
}
```

### 🎯 Helm Provider

**`hashicorp/helm`** — провайдер для Helm-релизов.

```hcl
provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}

resource "helm_release" "nginx_ingress" {
  name       = "nginx-ingress"
  repository = "https://kubernetes.github.io/ingress-nginx"
  chart      = "ingress-nginx"
  version    = "4.10.0"
  namespace  = "ingress-nginx"
  
  create_namespace = true
  
  values = [
    file("${path.module}/values/nginx-ingress.yaml")
  ]
  
  set {
    name  = "controller.replicaCount"
    value = "3"
  }
}

resource "helm_release" "postgresql" {
  name       = "postgresql"
  repository = "https://charts.bitnami.com/bitnami"
  chart      = "postgresql"
  version    = "12.5.6"
  namespace  = "database"
  
  create_namespace = true
  
  values = [
    templatefile("${path.module}/values/postgresql.yaml.tpl", {
      storage_size = "100Gi"
      replicas     = 3
    })
  ]
}
```

### 🎯 Kubernetes + EKS

**EKS-кластер создаётся через `aws` provider, потом K8s-ресурсы — через `kubernetes`.**

```hcl
# 1. EKS кластер
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "20.8.5"
  
  cluster_name    = "my-cluster"
  cluster_version = "1.29"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  eks_managed_node_groups = {
    main = {
      min_size     = 2
      max_size     = 10
      desired_size = 3
      
      instance_types = ["t3.medium"]
    }
  }
}

# 2. Настроить kubernetes provider на EKS
data "aws_eks_cluster_auth" "main" {
  name = module.eks.cluster_name
}

provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
  token                  = data.aws_eks_cluster_auth.main.token
}

provider "helm" {
  kubernetes {
    host                   = module.eks.cluster_endpoint
    cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
    token                  = data.aws_eks_cluster_auth.main.token
  }
}

# 3. Установить Helm-чарты
resource "helm_release" "argocd" {
  name       = "argocd"
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"
  version    = "6.7.0"
  namespace  = "argocd"
  
  create_namespace = true
  
  depends_on = [module.eks]
}
```

**Что произошло:**

1. EKS-кластер создан.
2. Kubernetes provider настроен на кластер.
3. Helm-чарты установлены в кластер.

**Всё — в одной Terraform-конфигурации.**

### 🎯 Terraform vs Helm vs kubectl

**Когда что использовать:**

| Ресурс | Инструмент |
|:---|:---|
| **Кластер EKS** | Terraform |
| **Namespaces** | Terraform или Helm |
| **RBAC** | Terraform или Helm |
| **Application** | Helm |
| **ConfigMaps** | Helm |
| **Secrets** | External Secrets или Helm |

**Рекомендация:**

- **Terraform** для инфраструктуры (кластеры, БД, сети).
- **Helm** для приложений (deployments, services, configmaps).
- **ArgoCD/Flux** для GitOps-деплоя Helm-чартов.

### 🎯 Проблема: state для K8s

Terraform хранит state для K8s-ресурсов. Если кластер удалён — state содержит ресурсы, которых нет.

**Решение:** отдельный state для K8s-ресурсов, привязанный к жизненному циклу кластера.

### 🔬 Практика: Terraform + Kubernetes

```hcl
# main.tf
provider "kubernetes" {
  config_path = "~/.kube/config"
}

resource "kubernetes_namespace" "app" {
  metadata {
    name = "myapp"
    
    labels = {
      environment = "dev"
      managed-by  = "terraform"
    }
  }
}

resource "kubernetes_config_map" "app" {
  metadata {
    name      = "app-config"
    namespace = kubernetes_namespace.app.metadata[0].name
  }
  
  data = {
    log_level = "info"
    port      = "8080"
  }
}

resource "kubernetes_secret" "app" {
  metadata {
    name      = "app-secret"
    namespace = kubernetes_namespace.app.metadata[0].name
  }
  
  data = {
    database_password = var.db_password
  }
  
  type = "Opaque"
}

output "namespace" {
  value = kubernetes_namespace.app.metadata[0].name
}
```

### 💡 Практика: как правильно использовать Terraform с K8s

**✅ ОБЯЗАТЕЛЬНО:**

1. **Terraform для инфраструктуры (кластер, БД).**
2. **Helm provider для чартов.**
3. **Kubernetes provider для namespaces, RBAC.**

**👍 СТОИТ:**

4. **Отдельный state для K8s-ресурсов.**
5. **ArgoCD/Flux для GitOps-деплоя приложений.**
6. **Не смешивать всё в одном state.**

**❌ НЕ ДЕЛАЙ:**

7. **Не используй Terraform для ConfigMaps приложения.** Helm удобнее.
8. **Не используй Terraform для частых изменений.** Только инфраструктура.

### Где мы сейчас

Мы разобрали Terraform + Kubernetes. Теперь — **Terraform в CI/CD**.

---

## 18.12 Terraform в CI/CD

### 🔌 Проблема: как применять Terraform автоматически

Ручной `terraform apply` — не масштабируется. Нужна автоматизация через CI/CD.

### 📊 Workflow

```
1. Разработчик создаёт PR с изменениями.
2. CI запускает terraform fmt, validate, plan.
3. Вывод plan публикуется в PR как комментарий.
4. Ревьюеры проверяют plan.
5. Merge в main.
6. CI запускает terraform apply.
7. Уведомление в Slack.
```

### 🎯 GitLab CI

**.gitlab-ci.yml:**

```yaml
stages:
  - validate
  - plan
  - apply

variables:
  TF_VERSION: "1.7.0"
  TF_ROOT: ${CI_PROJECT_DIR}/terraform

.terraform-base:
  image:
    name: hashicorp/terraform:${TF_VERSION}
    entrypoint: [""]
  before_script:
    - cd ${TF_ROOT}
    - terraform init
        -backend-config="bucket=${TF_STATE_BUCKET}"
        -backend-config="key=${CI_ENVIRONMENT_NAME}/terraform.tfstate"
        -backend-config="region=us-west-2"
        -backend-config="dynamodb_table=terraform-locks"

validate:
  extends: .terraform-base
  stage: validate
  script:
    - terraform fmt -check
    - terraform validate

plan:
  extends: .terraform-base
  stage: plan
  script:
    - terraform plan -out=tfplan
    - terraform show -no-color tfplan > plan.txt
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
      - ${TF_ROOT}/plan.txt
    expire_in: 1 week
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

apply:
  extends: .terraform-base
  stage: apply
  script:
    - terraform apply -auto-approve tfplan
  dependencies:
    - plan
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
  environment:
    name: production
```

**Что происходит:**

1. **Validate** — на каждый push.
2. **Plan** — на Pull Request. Сохраняется в artifacts.
3. **Apply** — после merge в main. Manual (кнопка).

### 🎯 GitHub Actions

**.github/workflows/terraform.yml:**

```yaml
name: Terraform

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

env:
  TF_VERSION: 1.7.0

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./terraform
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions
          aws-region: us-west-2
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Format
        run: terraform fmt -check
      
      - name: Terraform Validate
        run: terraform validate
      
      - name: Terraform Plan
        id: plan
        run: |
          terraform plan -no-color -out=tfplan
          terraform show -no-color tfplan > plan.txt
        continue-on-error: true
      
      - name: Comment PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('terraform/plan.txt', 'utf8');
            const output = `#### Terraform Plan 📖
            \`\`\`
            ${plan.substring(0, 60000)}
            \`\`\``;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });
      
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
```

**Что происходит:**

1. **Plan** на PR — комментарий с планом.
2. **Apply** на push в main.

### 🎯 Аутентификация в облаке

**Безопасный способ — OIDC, без статических токенов.**

**AWS:**

```yaml
# GitLab CI
variables:
  AWS_ROLE_ARN: arn:aws:iam::123456789:role/gitlab-actions

before_script:
  - |
    export $(printf "AWS_ACCESS_KEY_ID=%s AWS_SECRET_ACCESS_KEY=%s AWS_SESSION_TOKEN=%s" \
      $(aws sts assume-role-with-web-identity \
        --role-arn ${AWS_ROLE_ARN} \
        --role-session-name gitlab-ci \
        --web-identity-token ${CI_JOB_JWT_V2} \
        --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
        --output text))
```

**GitHub Actions:**

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions
    aws-region: us-west-2
```

**Что даёт:** нет статических credentials. Безопаснее.

### 🎯 Atlantis

**Atlantis** — специализированный инструмент для Terraform в PR.

**Что делает:**

- Автоматически запускает `plan` на каждый PR.
- Публикует план как комментарий.
- `apply` по команде в комментарии (`atlantis apply`).
- Блокирует merge до apply.
- Поддерживает locking.

**Установка:** в Kubernetes через Helm.

**Workflow:**

1. Разработчик создаёт PR.
2. Atlantis запускает `plan`.
3. Публикует план в PR.
4. Ревьюер проверяет.
5. Ревьюер пишет `atlantis apply`.
6. Atlantis применяет.
7. Merge разрешён.

**Преимущества:**

- **Специализирован.** Всё для Terraform.
- **Безопасен.** Locking, approval.
- **Удобен.** Работает через комментарии.

### 🎯 Секреты в CI/CD

**Не храни секреты в коде.**

**GitLab CI:**

- **CI/CD Variables** (Masked + Protected).
- **Vault integration**.

**GitHub Actions:**

- **Secrets.**
- **OIDC** для облаков.

**Для Terraform:**

- **`TF_VAR_*`** environment variables.
- **Vault provider** для чтения секретов.

```hcl
data "vault_generic_secret" "db" {
  path = "secret/db"
}

resource "aws_db_instance" "postgres" {
  password = data.vault_generic_secret.db.data["password"]
}
```

### 🔬 Практика: CI/CD для Terraform

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - plan
  - apply

image:
  name: hashicorp/terraform:1.7.0
  entrypoint: [""]

variables:
  TF_ROOT: ${CI_PROJECT_DIR}/terraform

cache:
  key: "${CI_COMMIT_REF_SLUG}"
  paths:
    - ${TF_ROOT}/.terraform

before_script:
  - cd ${TF_ROOT}
  - terraform init

validate:
  stage: validate
  script:
    - terraform fmt -check -recursive
    - terraform validate

plan:
  stage: plan
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 1 day
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

apply:
  stage: apply
  script:
    - terraform apply -auto-approve tfplan
  dependencies:
    - plan
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
  environment:
    name: production
```

### 💡 Практика: как правильно настроить CI/CD

**✅ ОБЯЗАТЕЛЬНО:**

1. **Отдельный stage для validate.**
2. **Plan на PR с публикацией в комментарий.**
3. **Apply только после merge.**
4. **Manual approval для production.**

**👍 СТОИТ:**

5. **Atlantis** для удобства.
6. **OIDC** вместо статических токенов.
7. **Vault** для секретов.
8. **Notifications** в Slack.

**❌ НЕ ДЕЛАЙ:**

9. **Не делай apply на каждый push.**
10. **Не храни state в CI.**
11. **Не используй статические credentials.**
12. **Не применяй в prod без ревью.**

### Где мы сейчас

Мы разобрали CI/CD. Теперь — **диагностика проблем**.

---

## 18.13 Диагностика проблем

### 🔌 Проблема: что-то не работает

Terraform упал с ошибкой. Или plan показывает странные изменения. Как диагностировать?

### 🔍 Типичные проблемы

**1. `Error: Failed to query available provider packages`**

**Причина:** нет доступа к registry, или неправильная версия.

**Решение:**

```bash
terraform init -upgrade
# или
terraform init -plugin-dir=/path/to/plugins
```

**2. `Error: Error acquiring the state lock`**

**Причина:** другой процесс Terraform работает, или lock не снят.

**Решение:**

```bash
# Проверить, кто держит lock
aws dynamodb get-item \
  --table-name terraform-locks \
  --key '{"LockID":{"S":"my-state-key"}}'

# Force unlock (осторожно!)
terraform force-unlock <lock-id>
```

**3. `Error: Resource already exists`**

**Причина:** ресурс уже существует в облаке, но не в state.

**Решение:** `terraform import`.

**4. `Error: Cycle: ...`**

**Причина:** циклическая зависимость между ресурсами.

**Решение:** разорвать цикл через `depends_on` или реструктуризацию.

**5. `Error: Invalid provider configuration`**

**Причина:** неправильные credentials или регион.

**Решение:** проверить `provider` блок, environment variables.

**6. `Error: error creating ... : ...`**

**Причина:** API вернул ошибку. Может быть много причин.

**Решение:** читать сообщение. Проверить IAM, квоты, ограничения.

**7. Drift**

**Причина:** кто-то изменил ресурс вручную.

**Решение:**

```bash
terraform plan -refresh-only
# Принять изменения
terraform apply -refresh-only
# Или откатить
terraform apply
```

**8. State mismatch**

**Причина:** state не соответствует реальности.

**Решение:**

```bash
terraform refresh
terraform plan
```

### 🎯 Отладка

**1. `TF_LOG`:**

```bash
export TF_LOG=DEBUG
terraform plan
```

**Уровни:** `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`.

**Сохранить в файл:**

```bash
export TF_LOG=DEBUG
export TF_LOG_PATH=tf.log
terraform plan
```

**2. `terraform console`:**

Интерактивная консоль для проверки выражений:

```bash
terraform console
> var.environment
"production"
> local.name_prefix
"myapp-production"
> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"
```

**3. `terraform graph`:**

Граф зависимостей:

```bash
terraform graph | dot -Tpng > graph.png
```

**4. `terraform state` команды:**

```bash
terraform state list
terraform state show aws_instance.web
terraform state pull
```

**5. `terraform providers`:**

```bash
terraform providers
terraform providers schema -json
```

### 🎯 Восстановление state

**Если state повреждён:**

**1. Backup:**

```bash
# S3 версионирование — есть предыдущие версии
aws s3api list-object-versions --bucket my-terraform-state --prefix env:/prod/terraform.tfstate
```

**2. Восстановить:**

```bash
aws s3api get-object --bucket my-terraform-state \
  --key env:/prod/terraform.tfstate \
  --version-id <version-id> \
  terraform.tfstate
```

**3. Или заново:**

```bash
terraform init
terraform import <resource> <id>
```

### 🎯 Проверка конфигурации

**1. `terraform validate`:**

```bash
terraform validate
```

**2. `terraform fmt -check`:**

```bash
terraform fmt -check -recursive
```

**3. `tflint`:**

```bash
brew install tflint
tflint
```

**4. `checkov`:**

```bash
pip install checkov
checkov -d .
```

**5. `terrascan`:**

```bash
brew install terrascan
terrascan scan
```

### 🎯 Линтеры и security scanners

**tflint:**

- Проверяет правила провайдера.
- Ловит deprecated ресурсы.
- Проверяет naming conventions.

**checkov:**

- Проверяет security best practices.
- Ловит: открытые S3 buckets, отсутствие encryption, etc.

**terrascan:**

- Security and compliance.
- Много политик.

**tfsec:**

```bash
brew install tfsec
tfsec .
```

**Используй в CI:**

```yaml
validate:
  stage: validate
  script:
    - terraform fmt -check
    - terraform validate
    - tflint
    - checkov -d .
    - tfsec .
```

### 🔬 Практика: отладка

```bash
# 1. Включить debug
export TF_LOG=DEBUG
terraform plan 2>&1 | head -100

# 2. Проверить консоль
terraform console
> var.aws_region
> local.name_prefix

# 3. Граф
terraform graph | dot -Tpng > graph.png

# 4. Проверить state
terraform state list
terraform state show aws_s3_bucket.demo

# 5. Проверить провайдеры
terraform providers

# 6. Линтеры
terraform fmt -check
terraform validate
tflint
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Читать сообщение об ошибке.** Оно точно говорит причину.
2. **`terraform plan` для diff.**
3. **`terraform state` для проверки.**

**👍 СТОИТ:**

4. **`TF_LOG=DEBUG`** для сложных проблем.
5. **`terraform console`** для проверки выражений.
6. **Линтеры в CI.**

**❌ НЕ ДЕЛАЙ:**

7. **Не редактируй state вручную.**
8. **Не делай `force-unlock` без понимания.**
9. **Не игнорируй drift.**

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **IaC** | Infrastructure as Code. |
| **Terraform** | Инструмент IaC от HashiCorp. |
| **HCL** | HashiCorp Configuration Language. |
| **Provider** | Плагин для облака/сервиса. |
| **Resource** | Создаваемый объект. |
| **Data source** | Чтение существующего объекта. |
| **State** | Файл с mapping между кодом и реальными ресурсами. |
| **Backend** | Где хранится state. |
| **Remote backend** | S3, GCS, Azure — для командной работы. |
| **Locking** | Блокировка state для предотвращения конфликтов. |
| **Drift** | Расхождение между state и реальностью. |
| **Module** | Переиспользуемый набор конфигураций. |
| **Root module** | Директория с `terraform apply`. |
| **Child module** | Импортируемый модуль. |
| **Variables** | Входные параметры. |
| **Outputs** | Выходные значения. |
| **Locals** | Локальные переменные. |
| **Workspace** | Изолированный state внутри backend. |
| **Plan** | Показать, что изменится. |
| **Apply** | Применить изменения. |
| **Destroy** | Удалить все ресурсы. |
| **Import** | Добавить существующий ресурс в state. |
| **Taint** | Пометить ресурс для пересоздания. |
| **Lifecycle** | Управление жизненным циклом ресурса. |
| **prevent_destroy** | Запретить удаление ресурса. |
| **Atlantis** | Инструмент для Terraform в PR. |
| **tflint** | Линтер для Terraform. |
| **checkov** | Security scanner для IaC. |

---

## Что мы узнали?

- **IaC** — декларативное описание инфраструктуры в Git.
- **Terraform** — самый популярный инструмент IaC.
- **Providers, resources, data sources** — основа конфигурации.
- **State** — mapping между кодом и реальностью. Не коммить в Git.
- **Remote backend** — S3, GCS, Azure. Для командной работы. Locking.
- **Modules** — переиспользование кода. Версионирование.
- **Variables, outputs, locals** — параметризация.
- **Workspaces** — изолированные state. Или директории для окружений.
- **plan, apply, destroy** — безопасный workflow. `prevent_destroy`, `-out`.
- **Import** — добавление существующих ресурсов.
- **Terraform + Kubernetes** — kubernetes и helm providers.
- **CI/CD** — автоматизация через GitLab CI, GitHub Actions, Atlantis.
- **Диагностика** — `TF_LOG`, `terraform console`, `terraform graph`, линтеры.

---

## Типичные ошибки

- ❌ **Коммитить state в Git.** Содержит secrets.
- ❌ **Локальный state для команды.** Конфликты, потеря.
- ❌ **`apply` без `plan`.**
- ❌ **`destroy` в production без понимания.**
- ❌ **Не использовать `prevent_destroy` для критичных.**
- ❌ **Хардкодить значения.** Используй variables.
- ❌ **Хранить secrets в коде.**
- ❌ **Не фиксировать версии провайдеров/модулей.**
- ❌ **Один state для всех окружений.**
- ❌ **Ручные изменения после apply.** Drift.
- ❌ **Не использовать remote backend.**
- ❌ **Забывать про `terraform fmt` и `validate`.**
- ❌ **Не использовать `-out=tfplan`.**
- ❌ **`force-unlock` без понимания.**

---

## Для быстрого повторения

- **Terraform workflow:** `init` → `fmt` → `validate` → `plan` → `apply`.
- **Providers:** `hashicorp/aws`, `hashicorp/kubernetes`, `hashicorp/helm`.
- **Resource:** `resource "aws_vpc" "main" { ... }`.
- **Data source:** `data "aws_ami" "ubuntu" { ... }`.
- **State:** локально или remote (S3, GCS, Terraform Cloud).
- **Backend S3:** bucket + DynamoDB для locking + KMS.
- **Modules:** `module "vpc" { source = "..." }`.
- **Variables:** `variable "x" { type = string }`.
- **Outputs:** `output "x" { value = ... }`.
- **Locals:** `locals { x = ... }`.
- **Workspaces:** `terraform workspace new prod`.
- **Import:** `terraform import aws_s3_bucket.x bucket-name`.
- **Lifecycle:** `prevent_destroy`, `create_before_destroy`, `ignore_changes`.
- **CI/CD:** GitLab CI, GitHub Actions, Atlantis.
- **Security:** OIDC, Vault, checkov, tfsec.

---

## Вопросы для самопроверки

1. Что такое Infrastructure as Code? Зачем нужен?
2. Чем Terraform отличается от Ansible, Pulumi, CloudFormation?
3. Что такое provider, resource, data source?
4. Что такое state? Зачем нужен? Почему не коммитить в Git?
5. Что такое remote backend? Какие популярные?
6. Что такое locking? Зачем нужен?
7. Что такое module? Как создать и использовать?
8. Что такое variables, outputs, locals?
9. Что такое workspaces? Когда использовать?
10. Что делают `terraform plan`, `apply`, `destroy`?
11. Что такое `prevent_destroy`? Зачем нужен?
12. Что такое `terraform import`? Как использовать?
13. Как интегрировать Terraform с Kubernetes?
14. Как настроить Terraform в CI/CD?
15. Что делать, если state повреждён?

---

## Ответы

**1. IaC**

Infrastructure as Code — декларативное описание инфраструктуры в текстовых файлах в Git. Воспроизводимость, версионирование, автоматизация, ревью.

**2. Terraform vs альтернативы**

Terraform: HCL, multi-cloud, много провайдеров. Ansible: конфигурация серверов, push-based. Pulumi: код на Go/Python/TS. CloudFormation: AWS-only.

**3. Provider, resource, data source**

Provider — плагин для облака. Resource — создаваемый объект. Data source — чтение существующего объекта.

**4. State**

Файл с mapping между кодом и реальными ресурсами. Содержит secrets — не коммитить в Git. Локальный state не масштабируется — нужен remote backend.

**5. Remote backend**

S3, GCS, Azure Blob, Terraform Cloud. Хранение state централизованно. Версионирование, блокировки, шифрование.

**6. Locking**

Блокировка state для предотвращения одновременных `apply`. DynamoDB для S3 backend. Если Terraform упал — `force-unlock`.

**7. Module**

Переиспользуемый набор конфигураций. `module "vpc" { source = "./modules/vpc" }`. Версионирование — `version = "1.0.0"`.

**8. Variables, outputs, locals**

Variables — входные параметры. Outputs — выходные значения. Locals — локальные вычисления.

**9. Workspaces**

Изолированные state внутри одного backend. `terraform workspace new prod`. Для одинаковой инфраструктуры с разными параметрами.

**10. plan, apply, destroy**

Plan — показать, что изменится. Apply — применить. Destroy — удалить все ресурсы.

**11. prevent_destroy**

В `lifecycle` блоке. Запрещает удаление ресурса. `terraform destroy` упадёт. Для БД, S3, критичных.

**12. Import**

`terraform import aws_s3_bucket.x bucket-name`. Или import block (Terraform 1.5+). Добавляет существующий ресурс в state.

**13. Terraform + Kubernetes**

`kubernetes` provider для K8s-ресурсов. `helm` provider для Helm-чартов. EKS через `aws` provider, потом K8s-ресурсы.

**14. CI/CD**

GitLab CI / GitHub Actions. `validate` → `plan` → `apply`. Atlantis для удобства. OIDC для аутентификации.

**15. State повреждён**

Backup из S3 версионирования. Или заново `terraform import`. Не редактировать state вручную.

---

## Куда идти дальше?

Мы разобрали Terraform — инфраструктуру как код. Теперь ты знаешь:

- Providers, resources, data sources.
- State и remote backend.
- Modules.
- Variables, outputs, locals.
- Workspaces.
- plan, apply, destroy.
- Import.
- Terraform + Kubernetes.
- CI/CD.
- Диагностика.

Следующая глава по оглавлению — **Глава 19: Конфигурационное управление — Ansible**.

Скажи «дальше» — и я отправлю Главу 19.