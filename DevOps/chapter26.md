# ☁️ Глава 26: Облака и multi-cloud стратегии

**Что вы узнаете:**
- Что такое облачные вычисления и какие модели существуют.
- Чем IaaS, PaaS, SaaS, CaaS, FaaS отличаются друг от друга.
- Как устроены AWS, GCP и Azure: регионы, зоны, сервисы.
- Что такое managed Kubernetes (EKS, GKE, AKS) и как его выбирать.
- Что такое multi-cloud и зачем он нужен.
- Что такое vendor lock-in и как его избегать.
- Что такое hybrid cloud и когда он оправдан.
- Что такое FinOps и как управлять стоимостью облака.
- Что такое disaster recovery в облаке (RTO, RPO, multi-region).
- Как мигрировать в облако без простоя.
- Как проектировать переносимые системы.
- Как выбрать облачного провайдера.

**После прочтения вы сможете:**
- Объяснить разницу между моделями облачных сервисов.
- Выбрать между AWS, GCP и Azure.
- Развернуть managed Kubernetes.
- Спроектировать multi-cloud стратегию.
- Избежать vendor lock-in.
- Управлять стоимостью облака через FinOps.
- Настроить disaster recovery.
- Мигрировать в облако без простоя.

---

## Содержание

- [26.0 Пролог: счет на $500 000 за месяц](#260-пролог-счет-на-500-000-за-месяц)
- [26.1 Что такое облачные вычисления](#261-что-такое-облачные-вычисления)
- [26.2 Модели облачных сервисов: IaaS, PaaS, SaaS, CaaS, FaaS](#262-модели-облачных-сервисов-iaas-paas-saas-caas-faas)
- [26.3 AWS, GCP, Azure: обзор](#263-aws-gcp-azure-обзор)
- [26.4 Регионы и зоны доступности](#264-регионы-и-зоны-доступности)
- [26.5 Managed Kubernetes: EKS, GKE, AKS](#265-managed-kubernetes-eks-gke-aks)
- [26.6 Managed сервисы: базы данных, кэши, очереди](#266-managed-сервисы-базы-данных-кэши-очереди)
- [26.7 Multi-cloud: зачем и как](#267-multi-cloud-зачем-и-как)
- [26.8 Vendor lock-in и как его избежать](#268-vendor-lock-in-и-как-его-избежать)
- [26.9 Hybrid cloud](#269-hybrid-cloud)
- [26.10 FinOps: управление стоимостью](#2610-finops-управление-стоимостью)
- [26.11 Disaster recovery в облаке](#2611-disaster-recovery-в-облаке)
- [26.12 Миграция в облако](#2612-миграция-в-облако)
- [26.13 Диагностика проблем](#2613-диагностика-проблем)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 26.0 Пролог: счет на $500 000 за месяц

Вторник, 10:00. Финансовый директор пишет в Slack:

> «Почему наш счет за AWS вырос с $50 000 до $500 000 за месяц? Что происходит?»

Ты открываешь AWS Cost Explorer. Видишь:

```
Total: $512,847.23 (+926% vs last month)

Top services:
- EC2: $280,000
- RDS: $95,000
- S3: $45,000
- Data Transfer: $60,000
- NAT Gateway: $32,000
```

**Ты в панике.** Что случилось?

Начинаешь разбираться:

**1. EC2.**

Ты смотришь на instances. Обнаруживаешь:

- **100 инстансов** `m5.4xlarge` работают **круглосуточно**.
- Из них **80 — никто не использует**.
- Они были созданы для эксперимента месяц назад и забыты.

**2. RDS.**

- **Multi-AZ** базы данных, которые не нужны.
- **Snapshots** за 3 года — терабайты.

**3. S3.**

- **10 TB данных** в Standard storage, хотя 90% не читаются.
- Можно было бы перевести в Glacier → в 10 раз дешевле.

**4. Data Transfer.**

- **Cross-AZ traffic** между сервисами — $60 000.
- Если бы сервисы были в одной зоне — было бы бесплатно.

**5. NAT Gateway.**

- **Огромный трафик** через NAT.
- Можно было бы использовать VPC Endpoints → в 10 раз дешевле.

**Итого:** $500 000 в месяц можно было бы снизить до $50 000.

**Это — проблема FinOps.**

**Облако — это не «просто работает».** Это **инструмент**, которым нужно **управлять**:

- **Стоимость** (FinOps).
- **Регионы и зоны** (RTO/RPO).
- **Vendor lock-in.**
- **Multi-cloud стратегия.**
- **Миграция.**

В этой главе мы разберём облака и multi-cloud. Как использовать их правильно.

Это — **финальная глава**. Мы прошли весь путь от Linux до платформенной инженерии. Теперь — **где всё это работает** — в облаке.

---

## 26.1 Что такое облачные вычисления

### 🔌 Проблема: почему облако

Традиционно: компания покупает серверы, ставит в дата-центр, обслуживает. **Capital Expenditure (CapEx).**

**Проблемы:**

- **Дорого.** Серверы, дата-центр, электричество, охлаждение.
- **Медленно.** Заказ сервера — недели.
- **Не масштабируется.** Нужно больше — покупай ещё.
- **Простой.** Сервер упал — сам чинишь.

**Облако** решает это. **Operational Expenditure (OpEx).**

### 📊 Что такое облачные вычисления

**Облачные вычисления** — предоставление вычислительных ресурсов (серверы, хранилище, базы данных, сети) через интернет по требованию.

**Ключевые характеристики (NIST):**

**1. On-demand self-service.**

Получить ресурсы без взаимодействия с человеком.

**2. Broad network access.**

Доступ из любой точки.

**3. Resource pooling.**

Мульти-тенантность. Ресурсы разделяются.

**4. Rapid elasticity.**

Быстрое масштабирование вверх и вниз.

**5. Measured service.**

Оплата за использование.

### 🎯 Преимущества облака

**1. Скорость.**

Создать сервер — секунды. Вместо недель.

**2. Масштабирование.**

Автоматически.

**3. Стоимость.**

Платишь за использование. Нет CapEx.

**4. Глобальность.**

Регионы по всему миру.

**5. Надёжность.**

Managed сервисы. Multi-AZ.

**6. Инновации.**

Новые сервисы (AI, ML, IoT).

### 🎯 Недостатки

**1. Стоимость может выйти из-под контроля.**

Если не управлять — счёт растёт.

**2. Vendor lock-in.**

Трудно мигрировать.

**3. Безопасность.**

Данные у провайдера.

**4. Compliance.**

Не все регуляции позволяют.

**5. Зависимость от интернета.**

Нет интернета — нет облака.

**6. Latency.**

Данные далеко.

### 🎯 Модели развёртывания

**1. Public cloud.**

AWS, GCP, Azure.

**2. Private cloud.**

OpenStack, VMware.

**3. Hybrid cloud.**

Комбинация.

**4. Multi-cloud.**

Несколько public.

### 💡 Практика: что важно понять

**✅ ОБЯЗАТЕЛЬНО:**

1. **Облако — OpEx, не CapEx.**
2. **On-demand, elastic, measured.**
3. **Управление стоимостью — критично.**

**👍 СТОИТ:**

4. **Public cloud** для большинства.
5. **Hybrid** для compliance.
6. **Multi-cloud** для resilience.

**❌ НЕ ДЕЛАЙ:**

7. **Не мигрируй в облако без плана.**
8. **Не игнорируй стоимость.**
9. **Не забывай про lock-in.**

### Где мы сейчас

Мы разобрали, что такое облако. Теперь — **модели сервисов**.

---

## 26.2 Модели облачных сервисов: IaaS, PaaS, SaaS, CaaS, FaaS

### 🔌 Проблема: что выбрать

Облако предлагает разные модели:

- **IaaS** — инфраструктура.
- **PaaS** — платформа.
- **SaaS** — софт.
- **CaaS** — контейнеры.
- **FaaS** — функции.

**Что выбрать?**

### 📊 IaaS

**Infrastructure as a Service.**

**Что:** виртуальные машины, хранилище, сети.

**Пример:** AWS EC2, GCP Compute Engine, Azure VM.

**Что ты управляешь:**

- **OS.**
- **Runtime.**
- **Middleware.**
- **Приложение.**

**Что провайдер:**

- **Виртуализация.**
- **Hardware.**
- **Сеть.**
- **Дата-центр.**

**Когда:** нужен полный контроль.

### 📊 PaaS

**Platform as a Service.**

**Что:** платформа для разработки и деплоя.

**Пример:** Heroku, Google App Engine, AWS Elastic Beanstalk.

**Что ты управляешь:**

- **Приложение.**

**Что провайдер:**

- **OS.**
- **Runtime.**
- **Middleware.**
- **Scaling.**

**Когда:** хочется быстро деплоить без управления инфраструктурой.

### 📊 SaaS

**Software as a Service.**

**Что:** готовый софт.

**Пример:** Gmail, Slack, Salesforce.

**Что ты управляешь:**

- **Ничего (только данные).**

**Что провайдер:**

- **Всё.**

**Когда:** нужен готовый инструмент.

### 📊 CaaS

**Container as a Service.**

**Что:** managed Kubernetes.

**Пример:** EKS, GKE, AKS.

**Что ты управляешь:**

- **Контейнеры.**
- **K8s-ресурсы.**

**Что провайдер:**

- **Control plane.**
- **Nodes.**
- **Networking.**

**Когда:** контейнеры, но не хочешь управлять K8s.

### 📊 FaaS

**Function as a Service.**

**Что:** serverless-функции.

**Пример:** AWS Lambda, GCP Cloud Functions, Azure Functions.

**Что ты управляешь:**

- **Код функции.**

**Что провайдер:**

- **Всё остальное.**

**Когда:** event-driven, редкие вызовы.

### 🎯 Модель ответственности

```
                  On-Prem   IaaS    PaaS    CaaS    FaaS    SaaS
Application       ✅        ✅      ✅      ✅      ✅      ❌
Data              ✅        ✅      ✅      ✅      ✅      ✅
Runtime           ✅        ✅      ❌      ✅      ❌      ❌
Middleware        ✅        ✅      ❌      ✅      ❌      ❌
OS                ✅        ✅      ❌      ✅      ❌      ❌
Virtualization    ✅        ❌      ❌      ❌      ❌      ❌
Servers           ✅        ❌      ❌      ❌      ❌      ❌
Storage           ✅        ❌      ❌      ❌      ❌      ❌
Networking        ✅        ❌      ❌      ❌      ❌      ❌
```

### 🎯 Что выбрать

**IaaS:**

- **Полный контроль.**
- **Legacy-приложения.**
- **Специфичные требования.**

**PaaS:**

- **Быстрый деплой.**
- **Нет DevOps-команды.**
- **Стандартные приложения.**

**CaaS:**

- **Контейнеры.**
- **Kubernetes-опыт.**
- **Микросервисы.**

**FaaS:**

- **Event-driven.**
- **Редкие вызовы.**
- **Serverless.**

**SaaS:**

- **Готовый софт.**
- **Нет кастомизации.**

**Рекомендация:** **CaaS** (Kubernetes) для большинства современных приложений.

### 🔬 Практика: модели

**Пример: развёртывание Go-приложения.**

**IaaS:**

```bash
# Создать EC2
aws ec2 run-instances --image-id ami-xxx --instance-type t3.medium

# SSH
ssh ec2-user@...

# Установить Go, nginx, systemd
# Настроить
# Задеплоить
```

**CaaS:**

```yaml
# Kubernetes manifest
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
```

**FaaS:**

```go
// Lambda handler
func HandleRequest(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    return events.APIGatewayProxyResponse{
        StatusCode: 200,
        Body:       "Hello",
    }, nil
}
```

### 💡 Практика: как выбрать модель

**✅ ОБЯЗАТЕЛЬНО:**

1. **CaaS для контейнеров.**
2. **IaaS для полного контроля.**
3. **FaaS для event-driven.**

**👍 СТОИТ:**

4. **PaaS для быстрого старта.**
5. **SaaS для готовых инструментов.**
6. **Комбинировать модели.**

**❌ НЕ ДЕЛАЙ:**

7. **Не используй IaaS, если можно CaaS.**
8. **Не используй FaaS для долгих задач.**
9. **Не забывай про lock-in.**

### Где мы сейчас

Мы разобрали модели. Теперь — **AWS, GCP, Azure**.

---

## 26.3 AWS, GCP, Azure: обзор

### 🔌 Проблема: какой провайдер выбрать

AWS, GCP, Azure — три главных облачных провайдера.

**Что выбрать?**

### 📊 AWS (Amazon Web Services)

**Доля рынка:** ~32% (лидер).

**Сильные стороны:**

- **Самый большой** набор сервисов (200+).
- **Глобальная инфраструктура** (30+ регионов).
- **Зрелость.**
- **Экосистема.**

**Слабые стороны:**

- **Сложный.** Много сервисов, легко запутаться.
- **Стоимость.** Может выйти из-под контроля.
- **UX.** Консоль перегружена.

**Ключевые сервисы:**

- **EC2** — VM.
- **EKS** — managed Kubernetes.
- **S3** — object storage.
- **RDS** — managed базы данных.
- **Lambda** — serverless.
- **DynamoDB** — NoSQL.
- **CloudFront** — CDN.

**Когда выбирать:**

- **Большинство случаев.**
- **Много сервисов.**
- **Зрелость.**

### 📊 GCP (Google Cloud Platform)

**Доля рынка:** ~11%.

**Сильные стороны:**

- **Kubernetes** (GKE — лучший managed K8s).
- **Data analytics** (BigQuery).
- **AI/ML** (Vertex AI).
- **Простота.** Меньше сервисов, но качественных.
- **Цена.** Часто дешевле.

**Слабые стороны:**

- **Меньше сервисов,** чем AWS.
- **Меньше регионов.**
- **Меньше enterprise-функций.**

**Ключевые сервисы:**

- **GCE** — VM.
- **GKE** — managed Kubernetes.
- **Cloud Storage** — object storage.
- **Cloud SQL** — managed базы данных.
- **Cloud Functions** — serverless.
- **BigQuery** — data warehouse.
- **Cloud CDN** — CDN.

**Когда выбирать:**

- **Kubernetes-first.**
- **Data/ML.**
- **Простота.**

### 📊 Azure

**Доля рынка:** ~23%.

**Сильные стороны:**

- **Интеграция с Microsoft** (Active Directory, Office).
- **Enterprise.** Много корпоративных клиентов.
- **Hybrid cloud** (Azure Arc).
- **Глобальная инфраструктура.**

**Слабые стороны:**

- **Сложность.** Много legacy.
- **UX.** Консоль.
- **Kubernetes.** AKS хуже GKE.

**Ключевые сервисы:**

- **Azure VM** — VM.
- **AKS** — managed Kubernetes.
- **Blob Storage** — object storage.
- **Azure SQL** — managed базы данных.
- **Azure Functions** — serverless.
- **Cosmos DB** — NoSQL.
- **Azure CDN** — CDN.

**Когда выбирать:**

- **Microsoft-стек.**
- **Enterprise.**
- **Hybrid cloud.**

### 🎯 Сравнение

| Аспект | AWS | GCP | Azure |
|:---|:---|:---|:---|
| **Доля рынка** | 32% | 11% | 23% |
| **Сервисы** | 200+ | 100+ | 200+ |
| **Регионы** | 30+ | 35+ | 60+ |
| **Kubernetes** | EKS | GKE | AKS |
| **Serverless** | Lambda | Functions | Functions |
| **Data** | Redshift | BigQuery | Synapse |
| **AI/ML** | SageMaker | Vertex AI | ML |
| **Зрелость** | Высокая | Средняя | Высокая |
| **Простота** | Низкая | Высокая | Низкая |
| **Цена** | Высокая | Средняя | Средняя |

### 🎯 Как выбирать

**AWS:**

- **Большинство.**
- **Много сервисов.**
- **Зрелость.**

**GCP:**

- **Kubernetes.**
- **Data/ML.**
- **Простота.**

**Azure:**

- **Microsoft.**
- **Enterprise.**
- **Hybrid.**

**Multi-cloud:** комбинация.

**Рекомендация:** начать с **AWS** или **GCP**.

### 🎯 Managed Kubernetes

**EKS (AWS):**

- **Зрелый.**
- **Много интеграций.**
- **Цена:** $0.10/час за control plane.

**GKE (GCP):**

- **Лучший managed K8s.**
- **Autopilot** (полностью managed).
- **Цена:** бесплатно для одного кластера.

**AKS (Azure):**

- **Интеграция с Azure.**
- **Цена:** бесплатно для control plane.

**Рекомендация:** **GKE** для Kubernetes-first.

### 🔬 Практика: создание кластера

**EKS:**

```bash
eksctl create cluster \
  --name my-cluster \
  --region us-west-2 \
  --nodegroup-name standard \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 10 \
  --managed
```

**GKE:**

```bash
gcloud container clusters create my-cluster \
  --region us-central1 \
  --num-nodes 3 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 10
```

**AKS:**

```bash
az aks create \
  --resource-group mygroup \
  --name my-cluster \
  --node-count 3 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10
```

### 💡 Практика: как выбирать провайдера

**✅ ОБЯЗАТЕЛЬНО:**

1. **AWS для большинства.**
2. **GCP для Kubernetes/Data.**
3. **Azure для Microsoft.**

**👍 СТОИТ:**

4. **Managed Kubernetes.**
5. **Multi-cloud** для resilience.
6. **Оценить стоимость.**

**❌ НЕ ДЕЛАЙ:**

7. **Не выбирай без оценки.**
8. **Не игнорируй lock-in.**
9. **Не забывай про регионы.**

### Где мы сейчас

Мы разобрали провайдеров. Теперь — **регионы и зоны**.

---

## 26.4 Регионы и зоны доступности

### 🔌 Проблема: где размещать

Облако имеет глобальную инфраструктуру. **Где размещать приложение?**

**Ответ:** регионы и зоны.

### 📊 Регион

**Регион** — географическая область (например, `us-west-2` — Орегон).

**Что содержит:**

- **Несколько зон** доступности.
- **Изолированную** инфраструктуру.
- **Свой** набор сервисов.

**Примеры:**

- **AWS:** `us-east-1` (Вирджиния), `eu-west-1` (Ирландия), `ap-southeast-1` (Сингапур).
- **GCP:** `us-central1` (Айова), `europe-west1` (Бельгия), `asia-east1` (Тайвань).
- **Azure:** `eastus`, `westeurope`, `southeastasia`.

### 📊 Зона доступности (AZ)

**AZ** — изолированный дата-центр внутри региона.

**Что содержит:**

- **Своё** питание.
- **Своё** охлаждение.
- **Свою** сеть.

**Изоляция:** отказ одной AZ не влияет на другие.

**Примеры:**

- `us-west-2a`, `us-west-2b`, `us-west-2c`.

**Рекомендация:** **3 AZ** для production.

### 🎯 Как выбирать регион

**Факторы:**

**1. Latency.**

Ближе к пользователям — меньше latency.

**2. Compliance.**

Данные должны быть в определённой юрисдикции (GDPR).

**3. Стоимость.**

Разные регионы — разные цены.

**4. Сервисы.**

Не все сервисы доступны во всех регионах.

**5. Disaster recovery.**

Второй регион для DR.

### 🎯 Multi-AZ

**Что:** приложение в нескольких AZ.

**Как:**

- **Kubernetes:** `topologySpreadConstraints` по zone.
- **Базы данных:** Multi-AZ replication.
- **Load balancer:** распределяет между AZ.

**Что даёт:**

- **Отказ AZ** → приложение работает.
- **RTO:** секунды.
- **RPO:** почти 0.

**Рекомендация:** **обязательно** для production.

### 🎯 Multi-Region

**Что:** приложение в нескольких регионах.

**Как:**

- **Active-passive:** один активный, второй standby.
- **Active-active:** оба активные.

**Что даёт:**

- **Отказ региона** → приложение работает.
- **RTO:** минуты (active-passive) или 0 (active-active).
- **RPO:** минуты или секунды.

**Проблемы:**

- **Data replication** между регионами — сложно.
- **Latency** между регионами.
- **Стоимость** — 2× ресурсы.

**Когда:** критичные сервисы.

### 🎯 Пример архитектуры

```
Region 1 (us-west-2)
├── AZ a
│   ├── Node 1
│   ├── Node 2
│   └── RDS Primary
├── AZ b
│   ├── Node 3
│   ├── Node 4
│   └── RDS Standby
└── AZ c
    ├── Node 5
    └── Node 6

Region 2 (us-east-1) — для DR
├── AZ a
│   ├── Node 1
│   └── RDS Read Replica
└── AZ b
    └── Node 2
```

### 🎯 Data replication

**Синхронная:**

- **Запись** подтверждается после репликации во все AZ.
- **RPO:** 0.
- **Latency:** выше.

**Асинхронная:**

- **Запись** подтверждается сразу.
- **Репликация** в фоне.
- **RPO:** > 0.
- **Latency:** ниже.

**Multi-AZ:** синхронная (обычно).
**Multi-Region:** асинхронная (обычно).

### 🔬 Практика: multi-AZ

**Kubernetes:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: myapp
```

**RDS:**

```bash
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --multi-az \
  --engine postgres
```

**Что даёт:**

- 6 Pod'ов по 2 в каждой из 3 AZ.
- RDS с standby в другой AZ.

### 💡 Практика: как правильно размещать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Multi-AZ** для production.
2. **3 AZ** минимум.
3. **Data replication.**

**👍 СТОИТ:**

4. **Multi-Region** для критичных.
5. **Близко к пользователям.**
6. **Compliance.**

**❌ НЕ ДЕЛАЙ:**

7. **Не размещай всё в одной AZ.**
8. **Не забывай про data transfer costs.**
9. **Не игнорируй latency.**

### Где мы сейчас

Мы разобрали регионы. Теперь — **managed Kubernetes**.

---

## 26.5 Managed Kubernetes: EKS, GKE, AKS

### 🔌 Проблема: Kubernetes сложный

Kubernetes мощный, но сложный:

- **Control plane:** apiserver, etcd, scheduler, controller-manager.
- **Nodes:** kubelet, kube-proxy, container runtime.
- **Networking:** CNI, kube-proxy, DNS.
- **Upgrades:** версии, deprecated API.
- **Security:** RBAC, secrets, patching.

**Управлять всем этим — боль.**

**Решение:** managed Kubernetes.

### 📊 Что такое managed Kubernetes

**Managed Kubernetes** — провайдер управляет control plane и частью nodes.

**Что провайдер:**

- **Control plane** (apiserver, etcd, scheduler).
- **Upgrades** control plane.
- **Patching.**
- **HA** control plane.
- **Интеграция** с облачными сервисами.

**Что ты:**

- **Nodes** (или managed node groups).
- **Workloads.**
- **Networking** (частично).

### 🎯 EKS (AWS)

**Elastic Kubernetes Service.**

**Что даёт:**

- **Managed control plane.**
- **Managed node groups** (EC2).
- **Fargate** (serverless nodes).
- **Интеграция** с IAM, VPC, EBS, ELB.

**Стоимость:** $0.10/час за control plane + nodes.

**Плюсы:**

- **Зрелый.**
- **Много интеграций.**
- **Большое сообщество.**

**Минусы:**

- **Сложнее** GKE.
- **Дороже.**
- **Networking** (CNI) — нужно настраивать.

**Когда:** AWS-first.

### 🎯 GKE (GCP)

**Google Kubernetes Engine.**

**Что даёт:**

- **Managed control plane.**
- **Node pools.**
- **Autopilot** (полностью managed).
- **Интеграция** с GCP.

**Стоимость:** бесплатно для одного кластера + nodes.

**Плюсы:**

- **Лучший managed K8s.**
- **Autopilot.**
- **Простой.**
- **Быстрый.**

**Минусы:**

- **GCP-специфичный.**
- **Меньше регионов.**

**Когда:** Kubernetes-first.

### 🎯 AKS (Azure)

**Azure Kubernetes Service.**

**Что даёт:**

- **Managed control plane.**
- **Node pools.**
- **Интеграция** с Azure AD, VNet, ACR.

**Стоимость:** бесплатно для control plane + nodes.

**Плюсы:**

- **Интеграция** с Microsoft.
- **Дешёвый.**
- **Enterprise.**

**Минусы:**

- **Сложнее** GKE.
- **Меньше функций.**

**Когда:** Azure-first, Microsoft-стек.

### 🎯 Сравнение

| Аспект | EKS | GKE | AKS |
|:---|:---|:---|:---|
| **Control plane** | Managed | Managed | Managed |
| **Autopilot** | Нет | Да | Нет |
| **Стоимость** | $0.10/час | Бесплатно (1 кластер) | Бесплатно |
| **Зрелость** | Высокая | Высокая | Средняя |
| **Простота** | Средняя | Высокая | Средняя |
| **Интеграция** | AWS | GCP | Azure |
| **Networking** | CNI | Native | CNI |
| **Serverless** | Fargate | Autopilot | Virtual Nodes |

### 🎯 Что выбрать

**EKS:**

- **AWS-first.**
- **Много AWS-сервисов.**
- **Зрелость.**

**GKE:**

- **Kubernetes-first.**
- **Простота.**
- **Autopilot.**

**AKS:**

- **Azure-first.**
- **Microsoft-стек.**
- **Enterprise.**

**Рекомендация:** **GKE** для начала.

### 🔬 Практика: EKS

```bash
# 1. Установить eksctl
brew install eksctl

# 2. Создать кластер
eksctl create cluster \
  --name my-cluster \
  --region us-west-2 \
  --version 1.29 \
  --nodegroup-name standard \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 10 \
  --managed

# 3. Проверить
kubectl get nodes
kubectl get pods -A

# 4. Установить addons
eksctl create addon --name aws-ebs-csi-driver --cluster my-cluster
eksctl create addon --name vpc-cni --cluster my-cluster
eksctl create addon --name coredns --cluster my-cluster

# 5. Удалить
eksctl delete cluster --name my-cluster
```

### 💡 Практика: как выбрать managed K8s

**✅ ОБЯЗАТЕЛЬНО:**

1. **GKE** для простоты.
2. **EKS** для AWS.
3. **AKS** для Azure.

**👍 СТОИТ:**

4. **Autopilot** (GKE).
5. **Managed node groups.**
6. **Интеграция** с облачными сервисами.

**❌ НЕ ДЕЛАЙ:**

7. **Не управляй control plane сам.**
8. **Не забывай про upgrades.**
9. **Не игнорируй networking.**

### Где мы сейчас

Мы разобрали managed K8s. Теперь — **managed сервисы**.

---

## 26.6 Managed сервисы: базы данных, кэши, очереди

### 🔌 Проблема: управлять базами сложно

PostgreSQL в production требует:

- **Replication.**
- **Backups.**
- **Patching.**
- **Monitoring.**
- **HA.**
- **Scaling.**

**Управлять этим — боль.**

**Решение:** managed сервисы.

### 📊 Managed базы данных

**AWS:**

- **RDS** — PostgreSQL, MySQL, MariaDB, Oracle, SQL Server.
- **Aurora** — AWS-специфичный, быстрее.
- **DynamoDB** — NoSQL.

**GCP:**

- **Cloud SQL** — PostgreSQL, MySQL, SQL Server.
- **Cloud Spanner** — глобально распределённый.
- **Firestore** — NoSQL.

**Azure:**

- **Azure SQL** — SQL Server.
- **Database for PostgreSQL** — PostgreSQL.
- **Cosmos DB** — NoSQL.

**Что даёт:**

- **Managed backups.**
- **Replication.**
- **HA.**
- **Patching.**
- **Monitoring.**
- **Scaling.**

### 🎯 Managed кэши

**AWS:** ElastiCache (Redis, Memcached).
**GCP:** Memorystore (Redis, Memcached).
**Azure:** Cache for Redis.

**Что даёт:**

- **Managed Redis.**
- **Replication.**
- **Failover.**
- **Backups.**

### 🎯 Managed очереди

**AWS:** SQS, SNS, MSK (Kafka).
**GCP:** Pub/Sub.
**Azure:** Service Bus, Event Hubs.

**Что даёт:**

- **Managed очереди.**
- **Масштабирование.**
- **Durability.**

### 🎯 Managed vs self-hosted

| Аспект | Managed | Self-hosted |
|:---|:---|:---|
| **Управление** | Провайдер | Ты |
| **Backups** | Авто | Сам |
| **HA** | Авто | Сам |
| **Patching** | Авто | Сам |
| **Стоимость** | Дороже | Дешевле |
| **Контроль** | Меньше | Больше |
| **Lock-in** | Есть | Нет |

**Правило:**

- **Managed для большинства.**
- **Self-hosted для специфичных требований.**

### 🎯 Когда managed

**✅ Managed:**

- **Нет DBA.**
- **Стандартные требования.**
- **Хочется скорость.**

**❌ Self-hosted:**

- **Специфичные требования.**
- **Cost optimization** (большие объёмы).
- **Compliance.**

### 🎯 Пример: RDS

```bash
# Создать PostgreSQL
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 16.1 \
  --master-username postgres \
  --master-user-password secret \
  --allocated-storage 100 \
  --multi-az \
  --backup-retention-period 30 \
  --storage-encrypted

# Что даёт:
# - Multi-AZ
# - Автоматические бэкапы (30 дней)
# - Encryption
# - Monitoring
# - Patching
```

### 🎯 Пример: ElastiCache

```bash
# Создать Redis
aws elasticache create-cache-cluster \
  --cache-cluster-id my-redis \
  --cache-node-type cache.t3.medium \
  --engine redis \
  --num-cache-nodes 1

# Что даёт:
# - Managed Redis
# - Backups
# - Monitoring
```

### 🎯 Пример: MSK (Kafka)

```bash
# Создать Kafka
aws kafka create-cluster \
  --cluster-name my-kafka \
  --kafka-version 3.5.1 \
  --number-of-broker-nodes 3 \
  --broker-node-group-info file://broker.json

# Что даёт:
# - Managed Kafka
# - Replication
# - Monitoring
```

### 🔬 Практика: managed сервисы

```bash
# 1. RDS
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --master-username postgres \
  --master-user-password secret \
  --allocated-storage 20 \
  --multi-az

# 2. Подключиться
psql -h mydb.xxx.us-west-2.rds.amazonaws.com -U postgres

# 3. ElastiCache
aws elasticache create-cache-cluster \
  --cache-cluster-id my-redis \
  --cache-node-type cache.t3.medium \
  --engine redis

# 4. SQS
aws sqs create-queue --queue-name my-queue
```

### 💡 Практика: как использовать managed

**✅ ОБЯЗАТЕЛЬНО:**

1. **Managed для большинства.**
2. **Multi-AZ** для production.
3. **Backups** настроить.

**👍 СТОИТ:**

4. **Encryption** at rest и in transit.
5. **Monitoring.**
6. **Cost optimization.**

**❌ НЕ ДЕЛАЙ:**

7. **Не self-hosted без DBA.**
8. **Не забывай про lock-in.**
9. **Не игнорируй стоимость.**

### Где мы сейчас

Мы разобрали managed сервисы. Теперь — **multi-cloud**.

---

## 26.7 Multi-cloud: зачем и как

### 🔌 Проблема: один провайдер — риск

Ты используешь только AWS. Что если:

- **AWS упадёт** (бывает).
- **AWS поднимет цены.**
- **AWS закроет сервис.**
- **Регуляция** требует другого провайдера.

**Решение:** multi-cloud.

### 📊 Что такое multi-cloud

**Multi-cloud** — использование нескольких облачных провайдеров.

**Примеры:**

- **AWS + GCP.**
- **AWS + Azure.**
- **AWS + GCP + Azure.**

**Что даёт:**

- **Resilience.** Отказ провайдера → переключение.
- **Avoid lock-in.** Не зависишь от одного.
- **Best-of-breed.** Лучшие сервисы от каждого.
- **Compliance.** Данные в разных юрисдикциях.

**Проблемы:**

- **Сложность.** Разные API, инструменты.
- **Стоимость.** Дублирование ресурсов.
- **Latency.** Между провайдерами.
- **Data transfer.** Дорого.

### 🎯 Зачем multi-cloud

**1. Resilience.**

**Плохо:** всё в AWS. AWS упал → всё упало.
**Хорошо:** AWS + GCP. Один упал → второй работает.

**2. Avoid lock-in.**

**Плохо:** используешь AWS-специфичные сервисы. Трудно мигрировать.
**Хорошо:** используешь portable-сервисы. Легко мигрировать.

**3. Best-of-breed.**

**Плохо:** всё в AWS, даже если GCP лучше для ML.
**Хорошо:** ML в GCP, остальное в AWS.

**4. Compliance.**

**Плохо:** данные только в US.
**Хорошо:** данные в US и EU.

### 🎯 Как делать multi-cloud

**1. Portable workloads.**

Используй Kubernetes, а не provider-specific сервисы.

**2. Infrastructure as Code.**

Terraform для управления ресурсами в разных облаках.

**3. Abstraction layer.**

Crossplane, Kubernetes для абстракции.

**4. Data replication.**

Репликация данных между облаками.

**5. DNS-based failover.**

Route53/Cloud DNS для переключения.

### 🎯 Что portable, что нет

**Portable:**

- **Kubernetes.**
- **Docker.**
- **PostgreSQL.**
- **Redis.**
- **Kafka.**
- **Terraform.**

**Not portable:**

- **AWS Lambda.**
- **AWS DynamoDB.**
- **AWS S3** (API, но детали).
- **GCP BigQuery.**
- **Azure Cosmos DB.**

**Правило:** используй portable, где возможно.

### 🎯 Пример multi-cloud

```
┌─────────────────────────────────────────┐
│              Global DNS                  │
│         (Route53 / Cloud DNS)            │
└──────────────┬──────────────────────────┘
               │
       ┌───────┴───────┐
       │               │
┌──────▼─────┐  ┌──────▼─────┐
│   AWS      │  │   GCP      │
│            │  │            │
│ EKS        │  │ GKE        │
│ PostgreSQL │  │ PostgreSQL │
│ Redis      │  │ Redis      │
└────────────┘  └────────────┘
       │               │
       └───────┬───────┘
               │
       Data Replication
```

**Что даёт:**

- **Отказ AWS** → GCP работает.
- **Отказ GCP** → AWS работает.
- **RTO:** минуты.

### 🎯 Anti-patterns

**1. Multi-cloud без причины.**

Просто «модно». **Плохо.**

**2. Multi-cloud с provider-specific.**

Lambda + Cloud Functions. **Сложно.**

**3. Multi-cloud без автоматизации.**

Ручное управление. **Плохо.**

**4. Multi-cloud без data replication.**

Данные в одном облаке. **Плохо.**

### 🔬 Практика: multi-cloud с Terraform

```hcl
# providers.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-west-2"
}

provider "google" {
  project = "my-project"
  region  = "us-central1"
}

# AWS EKS
module "eks" {
  source = "terraform-aws-modules/eks/aws"
  # ...
}

# GCP GKE
module "gke" {
  source = "terraform-google-modules/kubernetes-engine/google"
  # ...
}
```

### 💡 Практика: как делать multi-cloud

**✅ ОБЯЗАТЕЛЬНО:**

1. **Portable workloads.**
2. **Terraform.**
3. **Data replication.**

**👍 СТОИТ:**

4. **DNS failover.**
5. **Abstraction layer.**
6. **Автоматизация.**

**❌ НЕ ДЕЛАЙ:**

7. **Не без причины.**
8. **Не с provider-specific.**
9. **Не без replication.**

### Где мы сейчас

Мы разобрали multi-cloud. Теперь — **vendor lock-in**.

---

## 26.8 Vendor lock-in и как его избежать

### 🔌 Проблема: трудно мигрировать

Ты используешь AWS. Через год хочешь перейти на GCP. **Не можешь:**

- **Lambda** — нет аналога 1:1.
- **DynamoDB** — нет аналога.
- **S3** — API, но детали.
- **IAM** — AWS-специфичный.

**Vendor lock-in.**

### 📊 Что такое vendor lock-in

**Vendor lock-in** — зависимость от провайдера, из-за которой трудно мигрировать.

**Что вызывает:**

- **Provider-specific сервисы.**
- **Proprietary API.**
- **Data formats.**
- **Интеграции.**

### 🎯 Как избежать

**1. Используй portable сервисы.**

**Вместо Lambda:** Kubernetes + Knative.
**Вместо DynamoDB:** PostgreSQL, Cassandra.
**Вместо S3:** S3-совместимые (MinIO).
**Вместо RDS:** PostgreSQL на Kubernetes.

**2. Абстракция.**

**Crossplane:** Kubernetes-native IaC.
**Terraform:** multi-cloud.
**Kubernetes:** portable workloads.

**3. Open standards.**

**OCI** для контейнеров.
**OpenTelemetry** для observability.
**SPIFFE** для identity.

**4. Data portability.**

**PostgreSQL** вместо proprietary.
**Kafka** вместо proprietary.
**S3-совместимые** для storage.

**5. Multi-cloud стратегия.**

Не привязывайся к одному.

### 🎯 Что делать с managed

**Проблема:** managed сервисы — provider-specific.

**Решение:**

- **Используй managed** для скорости.
- **Планируй миграцию** заранее.
- **Абстрагируй** где возможно.

**Правило:** если managed даёт 10× выгоду — используй. Если 2× — подумай.

### 🎯 Cost of lock-in

**Что учитывать:**

- **Стоимость миграции.**
- **Время миграции.**
- **Риски.**
- **Выгода от managed.**

**Пример:**

- **Lambda** — выгода большая, но lock-in.
- **RDS** — выгода большая, lock-in меньше (PostgreSQL portable).

### 🎯 Пример: portable stack

**Вместо:**

- AWS Lambda → Kubernetes + Knative.
- AWS DynamoDB → PostgreSQL.
- AWS SQS → Kafka.
- AWS ElastiCache → Redis на Kubernetes.
- AWS S3 → MinIO.

**Что даёт:**

- **Portable** между облаками.
- **On-premise** возможно.
- **Нет lock-in.**

**Минусы:**

- **Больше управления.**
- **Меньше managed.**
- **Сложнее.**

**Компромисс:** managed для критичных, portable для остального.

### 🔬 Практика: portable stack

```yaml
# Kubernetes deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          env:
            - name: DATABASE_URL
              value: postgres://postgres:5432/mydb
            - name: KAFKA_URL
              value: kafka:9092
            - name: REDIS_URL
              value: redis:6379
            - name: S3_ENDPOINT
              value: http://minio:9000
```

**Что даёт:** работает в AWS, GCP, Azure, on-premise.

### 💡 Практика: как избежать lock-in

**✅ ОБЯЗАТЕЛЬНО:**

1. **Portable сервисы** где возможно.
2. **Open standards.**
3. **Абстракция.**

**👍 СТОИТ:**

4. **Managed** для критичных.
5. **Планировать миграцию.**
6. **Multi-cloud.**

**❌ НЕ ДЕЛАЙ:**

7. **Не используй provider-specific без причины.**
8. **Не игнорируй lock-in.**
9. **Не забывай про cost.**

### Где мы сейчас

Мы разобрали lock-in. Теперь — **hybrid cloud**.

---

## 26.9 Hybrid cloud

### 🔌 Проблема: не всё можно в облако

Иногда нельзя всё в облако:

- **Compliance** требует on-premise.
- **Legacy** системы.
- **Latency** критична.
- **Данные** чувствительны.

**Решение:** hybrid cloud.

### 📊 Что такое hybrid cloud

**Hybrid cloud** — комбинация on-premise и public cloud.

**Что даёт:**

- **Compliance** — sensitive данные on-premise.
- **Elasticity** — burst в облако.
- **Latency** — критичное on-premise.
- **Cost** — базовое on-premise, пики в облако.

### 🎯 Модели

**1. Cloud bursting.**

Базово on-premise, пики в облако.

**2. Data gravity.**

Данные on-premise, compute в облаке.

**3. Disaster recovery.**

On-premise + облако для DR.

**4. Edge computing.**

On-premise для edge, облако для централизации.

### 🎯 Технологии

**1. Azure Arc.**

Управление on-premise через Azure.

**2. AWS Outposts.**

AWS-оборудование on-premise.

**3. Google Anthos.**

Управление K8s везде.

**4. VMware Cloud.**

VMware on AWS.

### 🎯 Kubernetes в hybrid

**Kubernetes** — идеален для hybrid:

- **Portable.**
- **Одинаковый API.**
- **Multi-cluster.**

**Инструменты:**

- **Rancher** — управление K8s.
- **Anthos** — Google.
- **Azure Arc.**
- **OpenShift.**

### 🎯 Сеть в hybrid

**Проблема:** on-premise и cloud в разных сетях.

**Решение:**

- **VPN** — site-to-site.
- **Direct Connect** (AWS).
- **Cloud Interconnect** (GCP).
- **ExpressRoute** (Azure).

**Что даёт:** приватное соединение.

### 🎯 Когда hybrid

**✅ Hybrid:**

- **Compliance.**
- **Legacy.**
- **Latency.**
- **Cost.**

**❌ Не hybrid:**

- **Стартап.**
- **Всё greenfield.**
- **Мало данных.**

### 🔬 Практика: hybrid

**1. Direct Connect:**

```bash
# AWS Direct Connect
aws directconnect create-connection \
  --location "EqLD5" \
  --bandwidth "1Gbps" \
  --connection-name "my-dc"
```

**2. VPN:**

```bash
# Site-to-site VPN
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-xxx \
  --vpn-gateway-id vgw-xxx
```

**3. Kubernetes:**

```yaml
# Одинаковые манифесты on-premise и в облаке
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
```

### 💡 Практика: как делать hybrid

**✅ ОБЯЗАТЕЛЬНО:**

1. **Kubernetes** для portable.
2. **Приватное соединение.**
3. **Единое управление.**

**👍 СТОИТ:**

4. **Anthos, Arc, Rancher.**
5. **Data replication.**
6. **Мониторинг.**

**❌ НЕ ДЕЛАЙ:**

7. **Не hybrid без причины.**
8. **Не забывай про latency.**
9. **Не игнорируй стоимость.**

### Где мы сейчас

Мы разобрали hybrid. Теперь — **FinOps**.

---

## 26.10 FinOps: управление стоимостью

### 🔌 Проблема: счёт растёт

Облако — OpEx. Платишь за использование. **Счёт растёт.**

**Решение:** FinOps.

### 📊 Что такое FinOps

**FinOps** — практика управления стоимостью облака.

**Что включает:**

- **Visibility** — видеть расходы.
- **Optimization** — оптимизировать.
- **Governance** — управлять.

### 🎯 Принципы

**1. Collaboration.**

Финансы, инженерия, бизнес.

**2. Ownership.**

Каждая команда отвечает за свои расходы.

**3. Centralized visibility.**

Видеть все расходы.

**4. Variable cost model.**

Pay-as-you-go.

**5. Real-time decisions.**

Быстрые решения.

**6. Chargeback.**

Каждая команда платит за своё.

### 🎯 Что оптимизировать

**1. Right-sizing.**

**Проблема:** инстансы больше, чем нужно.

**Решение:** подобрать правильный размер.

```bash
# AWS Cost Explorer
# Right-sizing recommendations
```

**2. Reserved Instances / Savings Plans.**

**Проблема:** on-demand дорого.

**Решение:** reserved на 1-3 года.

**Экономия:** 30-70%.

**3. Spot Instances.**

**Проблема:** on-demand дорого.

**Решение:** spot для batch.

**Экономия:** 50-90%.

**4. Auto-scaling.**

**Проблема:** ресурсы простаивают.

**Решение:** auto-scaling.

**5. Storage tiering.**

**Проблема:** всё в Standard.

**Решение:** Glacier для архивов.

**Экономия:** 10-100×.

**6. Data transfer.**

**Проблема:** cross-AZ трафик дорого.

**Решение:** размещать сервисы в одной AZ.

**Экономия:** $0.01/GB × объём.

**7. NAT Gateway.**

**Проблема:** NAT дорого.

**Решение:** VPC Endpoints.

**Экономия:** 10×.

**8. Idle resources.**

**Проблема:** забытые инстансы.

**Решение:** мониторинг и удаление.

### 🎯 Инструменты

**AWS:**

- **Cost Explorer.**
- **Budgets.**
- **Cost Anomaly Detection.**
- **Trusted Advisor.**
- **Compute Optimizer.**

**GCP:**

- **Cloud Billing.**
- **Cost Management.**
- **Recommender.**

**Azure:**

- **Cost Management.**
- **Advisor.**

**Third-party:**

- **CloudHealth.**
- **Cloudability.**
- **Kubecost** (K8s).

### 🎯 Kubecost

**Kubecost** — стоимость Kubernetes.

**Что даёт:**

- **Cost per namespace.**
- **Cost per deployment.**
- **Cost per Pod.**
- **Recommendations.**

**Пример:**

```bash
kubecost namespace cost --namespace=production
# Total: $5,000/month
# - api: $2,000
# - worker: $1,500
# - cache: $500
```

### 🎯 Chargeback

**Chargeback** — каждая команда платит за свои ресурсы.

**Как:**

- **Tags** на ресурсах.
- **Namespaces** в K8s.
- **Cost allocation.**

**Что даёт:**

- **Ownership.**
- **Мотивация** оптимизировать.

### 🎯 Budgets и алерты

```bash
# AWS Budget
aws budgets create-budget \
  --account-id 123456789 \
  --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json
```

**Что даёт:** алерт при превышении.

### 🔬 Практика: FinOps

```bash
# 1. Cost Explorer
aws ce get-cost-and-usage \
  --time-period Start=2026-01-01,End=2026-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost

# 2. Right-sizing
aws compute-optimizer get-ec2-instance-recommendations

# 3. Reserved Instances
aws ec2 describe-reserved-instances

# 4. Spot
aws ec2 describe-spot-price-history \
  --instance-types m5.large \
  --product-descriptions "Linux/UNIX"

# 5. Kubecost
kubecost namespace cost

# 6. Budgets
aws budgets describe-budgets --account-id 123456789
```

### 💡 Практика: как управлять стоимостью

**✅ ОБЯЗАТЕЛЬНО:**

1. **Visibility** — Cost Explorer.
2. **Right-sizing.**
3. **Reserved/Spot.**
4. **Budgets** и алерты.

**👍 СТОИТ:**

5. **Kubecost** для K8s.
6. **Chargeback.**
7. **Регулярный review.**

**❌ НЕ ДЕЛАЙ:**

8. **Не игнорируй счёт.**
9. **Не оставляй idle resources.**
10. **Не забывай про data transfer.**

### Где мы сейчас

Мы разобрали FinOps. Теперь — **disaster recovery**.

---

## 26.11 Disaster recovery в облаке

### 🔌 Проблема: что если всё упадёт

Регион упал. Дата-центр сгорел. Данные потеряны. **Что делать?**

**Решение:** disaster recovery.

### 📊 Что такое DR

**Disaster recovery** — план восстановления после катастрофы.

**Что включает:**

- **Backups.**
- **Replication.**
- **Failover.**
- **Runbooks.**
- **Тестирование.**

### 🎯 RTO и RPO

**RTO (Recovery Time Objective)** — сколько времени на восстановление.

**RPO (Recovery Point Objective)** — сколько данных можно потерять.

**Пример:**

- **RTO = 1 час** — восстановление ≤ 1 час.
- **RPO = 15 минут** — потеря ≤ 15 минут данных.

### 🎯 Стратегии DR

**1. Backup and Restore.**

**Что:** бэкапы, восстановление.

**RTO:** часы-дни.
**RPO:** часы-дни.
**Стоимость:** низкая.

**2. Pilot Light.**

**Что:** минимальная инфраструктура в DR-регионе.

**RTO:** часы.
**RPO:** минуты.
**Стоимость:** средняя.

**3. Warm Standby.**

**Что:** уменьшенная копия в DR-регионе.

**RTO:** минуты.
**RPO:** секунды.
**Стоимость:** высокая.

**4. Multi-Site Active-Active.**

**Что:** полная копия в обоих регионах.

**RTO:** 0.
**RPO:** 0.
**Стоимость:** очень высокая.

### 🎯 Сравнение

| Стратегия | RTO | RPO | Стоимость |
|:---|:---|:---|:---|
| **Backup/Restore** | Часы-дни | Часы-дни | $ |
| **Pilot Light** | Часы | Минуты | $$ |
| **Warm Standby** | Минуты | Секунды | $$$ |
| **Active-Active** | 0 | 0 | $$$$ |

### 🎯 Backup

**Что бэкапить:**

- **Данные** (БД).
- **Конфигурация** (IaC, manifests).
- **Secrets.**
- **Images.**

**Куда:**

- **S3** (cross-region).
- **Glacier** (долгосрочно).
- **Другой провайдер.**

**Как часто:**

- **Критичные:** ежедневно.
- **Важные:** еженедельно.
- **Архивные:** ежемесячно.

### 🎯 Replication

**Data replication:**

- **RDS:** Multi-AZ, cross-region read replicas.
- **S3:** Cross-Region Replication.
- **DynamoDB:** Global Tables.

**Что даёт:**

- **RPO:** минуты или секунды.

### 🎯 Failover

**DNS-based:**

- **Route53** health checks.
- **Cloud DNS** failover.

**Как:**

1. DNS указывает на основной регион.
2. Health check проверяет.
3. Если недоступен — DNS переключается.

### 🎯 Runbook

**Что включает:**

1. **Detection** — как узнаём.
2. **Decision** — кто решает.
3. **Actions** — что делать.
4. **Communication** — кого уведомить.
5. **Rollback** — как откатить.

### 🎯 Тестирование

**Регулярно:**

- **Раз в квартал** — DR drill.
- **Раз в месяц** — partial.
- **После изменений** — проверка.

**Что тестировать:**

- **Failover** работает.
- **RTO/RPO** достигнуты.
- **Runbook** актуален.

### 🔬 Практика: DR

```bash
# 1. Backup RDS
aws rds create-db-snapshot \
  --db-instance-identifier mydb \
  --db-snapshot-identifier mydb-snapshot

# 2. Cross-region replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier mydb-replica \
  --source-db-instance-identifier arn:aws:rds:us-west-2:xxx:db:mydb \
  --region us-east-1

# 3. S3 replication
aws s3api put-bucket-replication \
  --bucket my-bucket \
  --replication-configuration file://replication.json

# 4. Route53 failover
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123 \
  --change-batch file://failover.json

# 5. DR drill
# Симулировать отказ основного региона
# Проверить failover
# Замерить RTO
```

### 💡 Практика: как делать DR

**✅ ОБЯЗАТЕЛЬНО:**

1. **Backups** регулярно.
2. **Replication** критичных данных.
3. **Runbook.**
4. **Тестирование.**

**👍 СТОИТ:**

5. **Multi-region** для критичных.
6. **DNS failover.**
7. **Автоматизация.**

**❌ НЕ ДЕЛАЙ:**

8. **Не без бэкапов.**
9. **Не без тестирования.**
10. **Не забывай про RTO/RPO.**

### Где мы сейчас

Мы разобрали DR. Теперь — **миграция**.

---

## 26.12 Миграция в облако

### 🔌 Проблема: как переехать

Компания on-premise. Хочет в облако. **Как мигрировать?**

### 📊 Стратегии миграции (6 R)

**1. Rehost (Lift and Shift).**

**Что:** перенести как есть.

**Плюсы:** быстро.
**Минусы:** не использует облако.

**2. Replatform.**

**Что:** небольшие изменения (managed БД).

**Плюсы:** использует облако.
**Минусы:** средняя сложность.

**3. Repurchase.**

**Что:** заменить на SaaS.

**Плюсы:** нет управления.
**Минусы:** lock-in.

**4. Refactor / Re-architect.**

**Что:** переписать под облако.

**Плюсы:** максимум выгоды.
**Минусы:** долго, дорого.

**5. Retire.**

**Что:** удалить ненужное.

**Плюсы:** экономия.
**Минусы:** нужно понять, что не нужно.

**6. Retain.**

**Что:** оставить on-premise.

**Плюсы:** для compliance.
**Минусы:** hybrid.

### 🎯 Фазы миграции

**1. Assessment (1-3 месяца).**

- **Inventory** приложений.
- **Dependencies.**
- **Стоимость.**
- **План.**

**2. Foundation (1-3 месяца).**

- **Landing Zone.**
- **Networking.**
- **IAM.**
- **Security.**

**3. Migration (6-18 месяцев).**

- **Pilot** (1-2 приложения).
- **Wave 1** (некритичные).
- **Wave 2** (важные).
- **Wave 3** (критичные).

**4. Optimization (ongoing).**

- **Right-sizing.**
- **Reserved.**
- **Refactoring.**

### 🎯 Landing Zone

**Что:** базовая инфраструктура в облаке.

**Что включает:**

- **Accounts/Projects.**
- **Networking** (VPC, subnets).
- **IAM.**
- **Security.**
- **Logging.**
- **Backup.**

**Инструменты:**

- **AWS Control Tower.**
- **GCP Landing Zone.**
- **Azure Landing Zone.**

### 🎯 Порядок миграции

**1. Начать с некритичного.**

**2. Pilot** для проверки.

**3. Постепенно** критичное.

**4. Не мигрировать всё сразу.**

### 🎯 Anti-patterns

**1. Big Bang.**

Мигрировать всё сразу. **Плохо.**

**2. Без плана.**

Просто «переехали». **Плохо.**

**3. Без Landing Zone.**

Инфраструктура потом. **Плохо.**

**4. Без тестирования.**

«Работает — и ладно». **Плохо.**

**5. Без rollback.**

Нельзя вернуться. **Плохо.**

### 🔬 Практика: миграция

```bash
# 1. Assessment
aws migrationhub list-discovered-resources

# 2. Landing Zone
aws controltower enable-control

# 3. Pilot
# Мигрировать 1-2 приложения
# Проверить

# 4. Wave 1
# Некритичные приложения

# 5. Wave 2
# Важные приложения

# 6. Wave 3
# Критичные приложения

# 7. Optimization
aws compute-optimizer get-ec2-instance-recommendations
```

### 💡 Практика: как мигрировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Assessment** сначала.
2. **Landing Zone.**
3. **Pilot.**
4. **Постепенно.**

**👍 СТОИТ:**

5. **6R стратегия.**
6. **Тестирование.**
7. **Rollback план.**

**❌ НЕ ДЕЛАЙ:**

8. **Не Big Bang.**
9. **Не без плана.**
10. **Не без тестирования.**

### Где мы сейчас

Мы разобрали миграцию. Теперь — **диагностика**.

---

## 26.13 Диагностика проблем

### 🔌 Проблема: что-то не так в облаке

Приложение работает медленно. Счёт растёт. Ресурсы не создаются.

### 🔍 Типичные проблемы

**1. Счёт растёт.**

**Диагностика:**

```bash
# Cost Explorer
aws ce get-cost-and-usage \
  --time-period Start=2026-01-01,End=2026-01-31 \
  --granularity DAILY \
  --metrics BlendedCost \
  --group-by Type=SERVICE

# Top resources
aws ce get-cost-and-usage \
  --group-by Type=DIMENSION,Key=USAGE_TYPE
```

**Решение:** right-sizing, reserved, spot, idle cleanup.

**2. Latency высокая.**

**Диагностика:**

```bash
# CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=my-lb

# X-Ray traces
aws xray get-trace-summaries
```

**Решение:** ближе к пользователям, кэш, CDN.

**3. Ресурсы не создаются.**

**Диагностика:**

```bash
# CloudTrail
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances

# Service Quotas
aws service-quotas list-service-quotas --service-code ec2
```

**Решение:** увеличить квоты, проверить IAM.

**4. S3 медленный.**

**Диагностика:**

```bash
# S3 metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name FirstByteLatency
```

**Решение:** CloudFront, Transfer Acceleration.

**5. RDS медленный.**

**Диагностика:**

```bash
# Performance Insights
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier db-xxx \
  --metric-queries file://queries.json
```

**Решение:** индексы, read replicas, scaling.

**6. Ноды в K8s не создаются.**

**Диагностика:**

```bash
# EKS
kubectl describe nodes
kubectl get events -A

# Cluster Autoscaler
kubectl logs -n kube-system deployment/cluster-autoscaler
```

**Решение:** квоты, IAM, subnet capacity.

**7. Data transfer дорого.**

**Диагностика:**

```bash
aws ce get-cost-and-usage \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=USAGE_TYPE \
  | grep DataTransfer
```

**Решение:** VPC Endpoints, same-AZ, CloudFront.

### 🎯 Общие команды

```bash
# Cost
aws ce get-cost-and-usage
aws budgets describe-budgets

# Metrics
aws cloudwatch get-metric-statistics

# Logs
aws logs filter-log-events

# Traces
aws xray get-trace-summaries

# Events
aws cloudtrail lookup-events

# Quotas
aws service-quotas list-service-quotas
```

### 🔬 Практика: диагностика

```bash
# 1. Cost
aws ce get-cost-and-usage \
  --time-period Start=2026-01-01,End=2026-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost

# 2. Top services
aws ce get-cost-and-usage \
  --group-by Type=DIMENSION,Key=SERVICE \
  --metrics BlendedCost

# 3. Metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-xxx

# 4. Logs
aws logs filter-log-events \
  --log-group-name /aws/lambda/myapp \
  --filter-pattern "ERROR"

# 5. Quotas
aws service-quotas list-service-quotas --service-code ec2
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Cost Explorer** для стоимости.
2. **CloudWatch** для метрик.
3. **CloudTrail** для событий.
4. **Service Quotas** для квот.

**👍 СТОИТ:**

5. **X-Ray** для traces.
6. **Performance Insights** для RDS.
7. **Kubecost** для K8s.

**❌ НЕ ДЕЛАЙ:**

8. **Не игнорируй счёт.**
9. **Не забывай про квоты.**
10. **Не диагностируй без метрик.**

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **Cloud computing** | Вычислительные ресурсы через интернет. |
| **IaaS** | Infrastructure as a Service. |
| **PaaS** | Platform as a Service. |
| **SaaS** | Software as a Service. |
| **CaaS** | Container as a Service. |
| **FaaS** | Function as a Service. |
| **Region** | Географическая область. |
| **AZ** | Availability Zone. |
| **EKS** | Elastic Kubernetes Service (AWS). |
| **GKE** | Google Kubernetes Engine. |
| **AKS** | Azure Kubernetes Service. |
| **RDS** | Relational Database Service (AWS). |
| **Multi-cloud** | Несколько облачных провайдеров. |
| **Hybrid cloud** | On-premise + cloud. |
| **Vendor lock-in** | Зависимость от провайдера. |
| **FinOps** | Управление стоимостью облака. |
| **Right-sizing** | Подбор правильного размера. |
| **Reserved Instances** | Резервирование на 1-3 года. |
| **Spot Instances** | Дешёвые, но могут быть отозваны. |
| **DR** | Disaster Recovery. |
| **RTO** | Recovery Time Objective. |
| **RPO** | Recovery Point Objective. |
| **Landing Zone** | Базовая инфраструктура в облаке. |
| **Chargeback** | Каждая команда платит за своё. |
| **Kubecost** | Стоимость Kubernetes. |

---

## Что мы узнали?

- **Облачные вычисления** — OpEx вместо CapEx. On-demand, elastic, measured.
- **Модели:** IaaS, PaaS, SaaS, CaaS, FaaS. CaaS для большинства.
- **AWS, GCP, Azure:** AWS лидер, GCP лучший K8s, Azure для Microsoft.
- **Регионы и AZ:** multi-AZ для production, 3 AZ минимум.
- **Managed K8s:** EKS, GKE, AKS. GKE лучший.
- **Managed сервисы:** RDS, ElastiCache, MSK. Managed для большинства.
- **Multi-cloud:** resilience, avoid lock-in, best-of-breed. Portable workloads.
- **Vendor lock-in:** избегать через portable сервисы, open standards, abstraction.
- **Hybrid cloud:** on-premise + cloud. Для compliance, legacy, latency.
- **FinOps:** visibility, right-sizing, reserved, spot, budgets. Kubecost.
- **DR:** backup, pilot light, warm standby, active-active. RTO/RPO.
- **Миграция:** 6R, landing zone, pilot, постепенно.
- **Диагностика:** Cost Explorer, CloudWatch, CloudTrail, X-Ray.

---

## Типичные ошибки

- ❌ **Не управлять стоимостью.** Счёт растёт.
- ❌ **Idle resources.** Забытые инстансы.
- ❌ **Всё в одной AZ.** Отказ AZ → downtime.
- ❌ **Vendor lock-in.** Provider-specific сервисы.
- ❌ **Big Bang миграция.** Всё сразу.
- ❌ **Без Landing Zone.** Инфраструктура потом.
- ❌ **Без бэкапов.** Данные потеряны.
- ❌ **Без DR drill.** Не знаем, работает ли.
- ❌ **Не использовать reserved.** Переплата.
- ❌ **Cross-AZ traffic.** Дорого.
- ❌ **NAT Gateway для всего.** Дорого.
- ❌ **Всё в Standard storage.** Дорого.
- ❌ **Без chargeback.** Нет ownership.
- ❌ **Multi-cloud без причины.** Сложно.

---

## Для быстрого повторения

- **Облако:** OpEx, on-demand, elastic.
- **Модели:** IaaS, PaaS, SaaS, CaaS, FaaS.
- **AWS, GCP, Azure:** AWS лидер, GCP K8s, Azure Microsoft.
- **Region + AZ:** multi-AZ для production.
- **Managed K8s:** EKS, GKE, AKS.
- **Managed сервисы:** RDS, ElastiCache, MSK.
- **Multi-cloud:** resilience, avoid lock-in.
- **Vendor lock-in:** portable сервисы, open standards.
- **Hybrid:** on-premise + cloud.
- **FinOps:** visibility, right-sizing, reserved, spot.
- **DR:** backup, pilot light, warm standby, active-active.
- **Миграция:** 6R, landing zone, pilot.
- **Диагностика:** Cost Explorer, CloudWatch, CloudTrail.

---

## Вопросы для самопроверки

1. Что такое облачные вычисления? Преимущества?
2. Модели облачных сервисов: IaaS, PaaS, SaaS, CaaS, FaaS?
3. Чем AWS отличается от GCP и Azure?
4. Что такое регион и AZ? Multi-AZ?
5. Что такое managed Kubernetes? EKS, GKE, AKS?
6. Что такое managed сервисы? Когда использовать?
7. Что такое multi-cloud? Зачем нужен?
8. Что такое vendor lock-in? Как избежать?
9. Что такое hybrid cloud? Когда использовать?
10. Что такое FinOps? Принципы?
11. Что такое right-sizing, reserved, spot?
12. Что такое disaster recovery? RTO/RPO?
13. Стратегии DR: backup, pilot light, warm standby, active-active?
14. Что такое landing zone? Зачем нужна?
15. Стратегии миграции: 6R?

---

## Ответы

**1. Облачные вычисления**

Предоставление вычислительных ресурсов через интернет по требованию. Преимущества: скорость, масштабирование, OpEx, глобальность, надёжность.

**2. Модели**

IaaS (VM), PaaS (платформа), SaaS (софт), CaaS (Kubernetes), FaaS (serverless). CaaS для большинства современных приложений.

**3. AWS vs GCP vs Azure**

AWS: лидер, много сервисов, зрелость. GCP: Kubernetes, Data/ML, простота. Azure: Microsoft, Enterprise, Hybrid.

**4. Регион и AZ**

Регион — географическая область (us-west-2). AZ — изолированный дата-центр внутри региона (us-west-2a). Multi-AZ: приложение в нескольких AZ для отказоустойчивости.

**5. Managed K8s**

Провайдер управляет control plane. EKS (AWS), GKE (GCP, лучший), AKS (Azure).

**6. Managed сервисы**

RDS, ElastiCache, MSK. Когда: нет DBA, стандартные требования, хочется скорость. Не когда: специфичные требования, cost optimization.

**7. Multi-cloud**

Несколько облачных провайдеров. Зачем: resilience, avoid lock-in, best-of-breed, compliance.

**8. Vendor lock-in**

Зависимость от провайдера. Избежать: portable сервисы, open standards, abstraction, multi-cloud.

**9. Hybrid cloud**

On-premise + cloud. Когда: compliance, legacy, latency, cost.

**10. FinOps**

Управление стоимостью облака. Принципы: collaboration, ownership, visibility, variable cost, real-time, chargeback.

**11. Right-sizing, reserved, spot**

Right-sizing: подобрать правильный размер. Reserved: 1-3 года, экономия 30-70%. Spot: дешёвые, могут быть отозваны, экономия 50-90%.

**12. DR**

Disaster recovery. RTO: время восстановления. RPO: потеря данных. Backup, pilot light, warm standby, active-active.

**13. Стратегии DR**

Backup/Restore: RTO часы, RPO часы, $. Pilot Light: RTO часы, RPO минуты, $$. Warm Standby: RTO минуты, RPO секунды, $$$. Active-Active: RTO 0, RPO 0, $$$$.

**14. Landing Zone**

Базовая инфраструктура в облаке. Accounts, networking, IAM, security, logging, backup. AWS Control Tower, GCP Landing Zone, Azure Landing Zone.

**15. 6R**

Rehost (lift and shift), Replatform (небольшие изменения), Repurchase (SaaS), Refactor (переписать), Retire (удалить), Retain (оставить).

---

## Куда идти дальше?

Мы завершили книгу. **26 глав + Глава 0.**

От Linux до облаков. От процессов до платформенной инженерии. Ты прошёл весь путь DevOps-инженера.

**Что дальше:**

1. **Практика.** Каждая глава — это инструмент. Используй.
2. **Углубление.** Выбери области, которые интересны. Углубляйся.
3. **Проекты.** Построй что-то. Инфраструктуру, CI/CD, мониторинг.
4. **Сообщество.** Делись опытом. Учи других.
5. **Сертификаты.** CKA, CKS, AWS, GCP, Terraform.

**Книга завершена. Но путь только начинается.**

**Удачи. 🚀**

---

Это полная **Глава 26: Облака и multi-cloud стратегии**.

Мы завершили книгу **«DevOps: от нуля до профессионала»** — 27 глав от философии DevOps до облаков.

Если нужно — могу:
- Собрать **финальное оглавление** всей книги.
- Написать **заключение** или **приложение** (чек-листы, roadmap).
- Исправить нумерацию в главах, которые были написаны с отклонениями.