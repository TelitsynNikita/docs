# 🔒 Глава 17: Безопасность Kubernetes

**Что вы узнаете:**
- Что такое модель безопасности Kubernetes и из чего она состоит.
- Как работает RBAC: Role, ClusterRole, RoleBinding, ClusterRoleBinding.
- Что такое ServiceAccount и как его использовать.
- Что такое Pod Security Standards и Pod Security Admission.
- Как работают Security Contexts: runAsUser, runAsNonRoot, capabilities.
- Что такое Network Policies для изоляции трафика.
- Как защитить секреты: шифрование etcd, external-secrets, sealed-secrets.
- Как сканировать образы на уязвимости: Trivy, Grype.
- Что такое supply chain security: подписи образов, SBOM, SLSA.
- Как работает audit logging и зачем он нужен.
- Как построить zero-trust в Kubernetes.

**После прочтения вы сможете:**
- Настроить RBAC с минимальными правами.
- Создать ServiceAccount для приложения.
- Применить Pod Security Standards.
- Настроить Security Context для Pod'а.
- Изолировать namespace через Network Policies.
- Зашифровать Secrets в etcd.
- Настроить сканирование образов в CI/CD.
- Настроить audit logging.
- Провести security audit кластера.

---

## Содержание

- [17.0 Пролог: утечка секретов через 3 года](#170-пролог-утечка-секретов-через-3-года)
- [17.1 Модель безопасности Kubernetes](#171-модель-безопасности-kubernetes)
- [17.2 RBAC: доступ к API](#172-rbac-доступ-к-api)
- [17.3 ServiceAccount: identity для Pod'ов](#173-serviceaccount-identity-для-podов)
- [17.4 Pod Security Standards](#174-pod-security-standards)
- [17.5 Security Context: изоляция контейнеров](#175-security-context-изоляция-контейнеров)
- [17.6 Network Policies: изоляция трафика](#176-network-policies-изоляция-трафика)
- [17.7 Секреты: шифрование и управление](#177-секреты-шифрование-и-управление)
- [17.8 Сканирование образов: Trivy, Grype](#178-сканирование-образов-trivy-grype)
- [17.9 Supply chain security: подписи, SBOM, SLSA](#179-supply-chain-security-подписи-sbom-slsa)
- [17.10 Audit logging](#1710-audit-logging)
- [17.11 Admission Controllers и OPA](#1711-admission-controllers-и-opa)
- [17.12 Zero-trust в Kubernetes](#1712-zero-trust-в-kubernetes)
- [17.13 Security audit кластера](#1713-security-audit-кластера)
- [17.14 Диагностика проблем](#1714-диагностика-проблем)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 17.0 Пролог: утечка секретов через 3 года

Ты — DevOps-инженер в компании. Пятница, 17:00. Приходит алерт от security-сканера:

```
CRITICAL: AWS access key detected in public GitHub repository
Repository: github.com/mycompany/legacy-config
File: k8s/secrets.yaml
Commit: 3 years ago
```

Ты открываешь файл:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
type: Opaque
data:
  AWS_ACCESS_KEY_ID: AKIAIOSFODNN7EXAMPLE
  AWS_SECRET_ACCESS_KEY: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

**Три года назад** кто-то закоммитил секреты в публичный репозиторий. Прошло **3 года**. Секреты **не менялись**.

За это время:

- **Неизвестно, кто** мог их найти.
- **Неизвестно, что** могли сделать.
- **Нужно ротировать** все credentials.
- **Аудит** всего, к чему был доступ.

**Это — катастрофа.**

**Проблемы:**

1. **Секреты в Git.** Даже в приватном.
2. **Нет сканирования** на секреты.
3. **Нет ротации** credentials.
4. **Нет audit log** кто что делал.
5. **Нет RBAC** — все имеют полный доступ.
6. **Нет Pod Security** — контейнеры работают от root.
7. **Нет Network Policies** — все видят всех.

**Это — не одна ошибка, а отсутствие системной безопасности.**

**Безопасность Kubernetes** — не одна функция, а **система**.

В этой главе мы разберём безопасность Kubernetes. От RBAC до zero-trust.

Это — **основа production**. Без безопасности всё остальное — риск.

---

## 17.1 Модель безопасности Kubernetes

### 🔌 Проблема: с чего начать

Kubernetes — сложная система. У неё много компонентов: API server, etcd, kubelet, Pod'ы. Каждый — точка атаки.

**Как защитить всё?**

**Ответ:** многослойная модель.

### 📊 Четыре слоя

**1. Cluster infrastructure.**

- Nodes.
- Control plane.
- etcd.
- Network.

**2. Cluster components.**

- API server.
- kubelet.
- kube-proxy.
- Controller-manager.

**3. Application code.**

- Приложение.
- Контейнеры.
- Pod'ы.

**4. Access management.**

- RBAC.
- ServiceAccount.
- Admission Controllers.

### 🎯 4C Model

**Cloud, Cluster, Container, Code.**

```
┌─────────────────────────────────────────┐
│              CLOUD                       │
│  (AWS, GCP, Azure — инфраструктура)     │
│  ┌────────────────────────────────────┐ │
│  │           CLUSTER                   │ │
│  │  (Kubernetes — control plane, nodes)│ │
│  │  ┌────────────────────────────────┐ │ │
│  │  │         CONTAINER              │ │ │
│  │  │  (образ, runtime, capabilities)│ │ │
│  │  │  ┌──────────────────────────┐  │ │ │
│  │  │  │         CODE             │  │ │ │
│  │  │  │  (приложение, зависимости)│  │ │ │
│  │  │  └──────────────────────────┘  │ │ │
│  │  └────────────────────────────────┘ │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

**Правило:** **каждый слой защищает следующий.** Если Cloud не защищён — Cluster скомпрометирован. Если Container не защищён — Code скомпрометирован.

### 🎯 Угрозы

**1. Компрометация контейнера.**

Злоумышленник взломал приложение. Что может?

- **Escape** из контейнера.
- **Доступ** к другим Pod'ам.
- **Доступ** к API server.

**2. Компрометация API.**

Злоумышленник получил доступ к API. Что может?

- **Создавать** Pod'ы.
- **Читать** Secrets.
- **Модифицировать** ресурсы.

**3. Компрометация ноды.**

Злоумышленник получил root на ноде. Что может?

- **Читать** все Secrets на ноде.
- **Читать** все логи.
- **Атаковать** другие ноды.

### 🎯 Защита

**1. Аутентификация.**

- **X.509 certificates** для компонентов.
- **ServiceAccount tokens** для Pod'ов.
- **OIDC** для пользователей.

**2. Авторизация.**

- **RBAC** — права на API.
- **Network Policies** — права на сеть.
- **Admission Controllers** — права на создание ресурсов.

**3. Изоляция.**

- **Namespaces** — логическая изоляция.
- **Network Policies** — сетевая изоляция.
- **Pod Security Standards** — изоляция Pod'ов.

**4. Шифрование.**

- **etcd encryption** — Secrets at rest.
- **mTLS** — трафик между компонентами.
- **TLS** — трафик API server.

**5. Аудит.**

- **Audit logs** — кто что делал.
- **Runtime security** — Falco, Tetragon.
- **Image scanning** — уязвимости.

### 💡 Практика: что важно понять

**✅ ОБЯЗАТЕЛЬНО:**

1. **Безопасность — многослойная.**
2. **Каждый слой защищает следующий.**
3. **Defense in depth.**

**👍 СТОИТ:**

4. **Начинать с RBAC.**
5. **Pod Security Standards.**
6. **Network Policies.**

**❌ НЕ ДЕЛАЙ:**

7. **Не полагайся на один слой.**
8. **Не забывай про audit.**
9. **Не игнорируй supply chain.**

### Где мы сейчас

Мы разобрали модель безопасности. Теперь — **RBAC**.

---

## 17.2 RBAC: доступ к API

### 🔌 Проблема: кто что может

API server — точка входа. Каждый запрос: кто, что, где.

**RBAC** отвечает на эти вопросы.

### 📊 Что такое RBAC

**RBAC (Role-Based Access Control)** — управление доступом на основе ролей.

**Компоненты:**

- **Role** — набор прав в namespace.
- **ClusterRole** — набор прав в кластере.
- **RoleBinding** — связь Role с субъектом (namespace).
- **ClusterRoleBinding** — связь ClusterRole с субъектом (кластер).
- **Subject** — кто (User, Group, ServiceAccount).

### 🎯 Role и ClusterRole

**Role (namespace):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
```

**Что даёт:** чтение Pod'ов и их логов в namespace `production`.

**ClusterRole (кластер):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list"]
```

**Что даёт:** чтение Nodes и PV во всём кластере.

### 🎯 RoleBinding и ClusterRoleBinding

**RoleBinding:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: myapp
    namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Что даёт:** alice и ServiceAccount myapp могут читать Pod'ы в `production`.

**ClusterRoleBinding:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

**Что даёт:** alice может читать Nodes и PV во всём кластере.

### 🎯 Verbs

| Verb | Что означает |
|:---|:---|
| **get** | Читать один объект |
| **list** | Список объектов |
| **watch** | Подписка на изменения |
| **create** | Создать |
| **update** | Обновить |
| **patch** | Частично обновить |
| **delete** | Удалить |
| **deletecollection** | Удалить все |

**Важно:** `list` может быть опасен. Например, `list secrets` = прочитать все Secrets.

### 🎯 Принцип наименьших привилегий

**Правило:** давать **минимум** прав, необходимый для работы.

**Плохо:**

```yaml
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

**Это `cluster-admin`.** Полный доступ. **Никогда в production.**

**Хорошо:**

```yaml
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
    resourceNames: ["myapp-config"]  # только конкретный
```

### 🎯 Проверка прав

```bash
# Может ли ServiceAccount?
kubectl auth can-i get pods \
  --as=system:serviceaccount:production:myapp

# Может ли user?
kubectl auth can-i create deployments --as=alice

# В каком namespace?
kubectl auth can-i list secrets \
  --as=system:serviceaccount:production:myapp \
  -n production

# Все права
kubectl auth can-i --list \
  --as=system:serviceaccount:production:myapp
```

### 🎯 Аудит RBAC

**Найти опасные роли:**

```bash
# Все ClusterRoleBinding с cluster-admin
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'

# Все, кто может create pods
kubectl get clusterrolebindings,rolebindings -A -o json | \
  jq '.items[] | select(.roleRef.name | test("admin|edit")) | .subjects'

# ServiceAccounts с правами на secrets
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq '.items[] | select(.roleRef.name | test("secret"))'
```

**Инструменты:**

- **rbac-tool** — визуализация.
- **kubectl-who-can** — кто может.
- **audit2rbac** — аудит.

### 🔬 Практика: RBAC

```bash
# 1. Создать ServiceAccount
kubectl create serviceaccount myapp -n production

# 2. Создать Role
cat > role.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list"]
    resourceNames: ["myapp-config", "myapp-secret"]
EOF
kubectl apply -f role.yaml

# 3. Создать RoleBinding
cat > rolebinding.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: myapp
    namespace: production
roleRef:
  kind: Role
  name: myapp-role
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f rolebinding.yaml

# 4. Проверить
kubectl auth can-i get configmaps/myapp-config \
  --as=system:serviceaccount:production:myapp \
  -n production
# yes

kubectl auth can-i get configmaps/other-config \
  --as=system:serviceaccount:production:myapp \
  -n production
# no

kubectl auth can-i delete pods \
  --as=system:serviceaccount:production:myapp \
  -n production
# no
```

### 💡 Практика: как правильно настраивать RBAC

**✅ ОБЯЗАТЕЛЬНО:**

1. **Принцип наименьших привилегий.**
2. **`resourceNames`** для конкретных ресурсов.
3. **Не использовать `cluster-admin`** для приложений.
4. **Проверять** через `kubectl auth can-i`.

**👍 СТОИТ:**

4. **Отдельный ServiceAccount** на приложение.
5. **Аудит RBAC** регулярно.
6. **rbac-tool** для визуализации.

**❌ НЕ ДЕЛАЙ:**

7. **Не давай `*` в verbs или resources.**
8. **Не используй `default` ServiceAccount.**
9. **Не забывай про `list secrets`.** Опасно.

### Где мы сейчас

Мы разобрали RBAC. Теперь — **ServiceAccount**.

---

## 17.3 ServiceAccount: identity для Pod'ов

### 🔌 Проблема: как Pod обращается к API

Pod'у нужно обратиться к API server. Как он аутентифицируется?

**Ответ:** ServiceAccount.

### 📊 Что такое ServiceAccount

**ServiceAccount** — identity для Pod'ов.

**Каждый Pod имеет ServiceAccount:**

- **По умолчанию:** `default` в namespace.
- **Явно:** `spec.serviceAccountName`.

**ServiceAccount имеет токен** — JWT, который Pod использует для аутентификации в API.

### 🎯 Как работает

**1. ServiceAccount создаётся:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  namespace: production
```

**2. Pod использует:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp
  containers:
    - name: myapp
      image: myapp:1.0
```

**3. Токен монтируется:**

```
/var/run/secrets/kubernetes.io/serviceaccount/
├── token       # JWT для API
├── ca.crt      # CA для проверки API
└── namespace   # namespace Pod'а
```

**4. Приложение использует:**

```go
import (
    "os"
    "k8s.io/client-go/rest"
)

func main() {
    config, err := rest.InClusterConfig()
    if err != nil {
        panic(err)
    }
    
    clientset, err := kubernetes.NewForConfig(config)
    // Использовать API
}
```

### 🎯 Projected ServiceAccount Token

**Современный подход:** projected token с TTL.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp
  containers:
    - name: myapp
      image: myapp:1.0
      volumeMounts:
        - name: token
          mountPath: /var/run/secrets/tokens
          readOnly: true
  volumes:
    - name: token
      projected:
        sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600  # 1 час
              audience: api
```

**Что даёт:**

- **TTL** (1 час вместо бесконечного).
- **Audience** (для кого).
- **Ротация** автоматическая.

**Рекомендуется.** Старые токены (без TTL) — deprecated.

### 🎯 Автомонтирование

**По умолчанию** токен монтируется во все Pod'ы.

**Отключить:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  automountServiceAccountToken: false
  containers:
    - name: myapp
      image: myapp:1.0
```

**Когда:** если Pod не обращается к API.

**Что даёт:** меньше поверхности атаки. Если Pod скомпрометирован — нет токена.

### 🎯 ServiceAccount для CI/CD

**Пример для GitLab CI:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-ci
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitlab-ci-deployer
  namespace: production
subjects:
  - kind: ServiceAccount
    name: gitlab-ci
    namespace: production
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

**Что даёт:** CI может деплоить, но не может читать Secrets, удалять ресурсы.

### 🎯 Опасности

**1. `default` ServiceAccount.**

Если Pod использует `default` — может иметь права, которые ему не нужны.

**Решение:** создавать отдельные ServiceAccount.

**2. Автомонтирование токена.**

Если Pod не нужен API — не монтировать.

**3. Избыточные права.**

`cluster-admin` для приложения — катастрофа.

**4. Долгий TTL.**

Токен без TTL — если скомпрометирован, работает вечно.

**Решение:** projected token с TTL.

### 🔬 Практика: ServiceAccount

```bash
# 1. Создать ServiceAccount
kubectl create serviceaccount myapp -n production

# 2. Проверить токен в Pod'е
kubectl exec myapp-xxx -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# ca.crt  namespace  token

# 3. Использовать токен
TOKEN=$(kubectl exec myapp-xxx -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)
kubectl exec myapp-xxx -- curl -k -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/namespaces/production/pods

# 4. Projected token
cat > pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: myapp-projected
spec:
  serviceAccountName: myapp
  containers:
    - name: myapp
      image: myapp:1.0
      volumeMounts:
        - name: token
          mountPath: /var/run/secrets/tokens
  volumes:
    - name: token
      projected:
        sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
EOF
kubectl apply -f pod.yaml

# 5. Проверить
kubectl exec myapp-projected -- ls /var/run/secrets/tokens/
# token

# 6. Отключить автомонтирование
kubectl patch serviceaccount myapp -n production \
  -p '{"automountServiceAccountToken": false}'
```

### 💡 Практика: как правильно работать с ServiceAccount

**✅ ОБЯЗАТЕЛЬНО:**

1. **Отдельный ServiceAccount** на приложение.
2. **Не использовать `default`.**
3. **Минимальные права** через RBAC.
4. **Projected token** с TTL.

**👍 СТОИТ:**

4. **Отключать автомонтирование** если не нужно.
5. **Разные ServiceAccount** для разных задач.
6. **Аудит** использования.

**❌ НЕ ДЕЛАЙ:**

7. **Не давай `cluster-admin`.**
8. **Не используй `default`.**
9. **Не забывай про TTL.**

### Где мы сейчас

Мы разобрали ServiceAccount. Теперь — **Pod Security Standards**.

---

## 17.4 Pod Security Standards

### 🔌 Проблема: опасные Pod'ы

Pod может:

- **Запускаться от root.**
- **Иметь все capabilities.**
- **Использовать privileged mode.**
- **Монтировать hostPath.**
- **Использовать hostNetwork.**

**Это опасно.** Если Pod скомпрометирован — злоумышленник получает много.

**Решение:** Pod Security Standards.

### 📊 Что такое Pod Security Standards

**Pod Security Standards (PSS)** — три уровня безопасности Pod'ов.

**Уровни:**

- **Privileged** — без ограничений.
- **Baseline** — минимальные ограничения.
- **Restricted** — строгие ограничения.

### 🎯 Privileged

**Без ограничений.**

**Когда:** системные Pod'ы (CNI, storage, monitoring).

**Не для приложений.**

### 🎯 Baseline

**Минимальные ограничения.**

**Что запрещено:**

- **Privileged mode.**
- **hostNetwork, hostPID, hostIPC.**
- **hostPath volumes.**
- **Опасные capabilities** (SYS_ADMIN, NET_ADMIN, ...).
- **Root user** не запрещён (можно).

**Когда:** большинство приложений.

### 🎯 Restricted

**Строгие ограничения.**

**Что запрещено:**

- **Всё из Baseline.**
- **Root user** (runAsNonRoot: true).
- **Capabilities** (кроме NET_BIND_SERVICE).
- **Seccomp profile** обязателен.
- **Проверка imagePullPolicy.**

**Когда:** критичные приложения.

### 🎯 Pod Security Admission

**Pod Security Admission (PSA)** — реализация PSS в Kubernetes.

**Режимы:**

- **enforce** — блокировать Pod'ы.
- **audit** — логировать.
- **warn** — предупреждать.

**Настройка через labels namespace:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Что произойдёт:** Pod'ы, не соответствующие Restricted, будут **заблокированы**.

### 🎯 Проверка

```bash
# Посмотреть labels namespace
kubectl get namespace production --show-labels

# Проверить, пройдёт ли Pod
kubectl label namespace production pod-security.kubernetes.io/enforce=baseline --dry-run=server

# Audit mode — какие Pod'ы не соответствуют
kubectl get events -A | grep "pod-security"
```

### 🎯 Примеры Pod'ов

**Baseline:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: baseline-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      securityContext:
        privileged: false
        capabilities:
          drop: ["ALL"]
```

**Restricted:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myapp:1.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

### 🎯 Миграция

**Проблема:** production работает с Privileged. Как перейти на Restricted?

**Стратегия:**

1. **Включить audit mode** для Restricted.
2. **Смотреть events** — какие Pod'ы не соответствуют.
3. **Исправить** приложения.
4. **Включить warn mode.**
5. **Через неделю — enforce mode.**

**Постепенно.**

### 🎯 Исключения

**Для системных Pod'ов:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kube-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
```

**Или через `exemptions` в PSA config.**

### 🔬 Практика: Pod Security

```bash
# 1. Пометить namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# 2. Попробовать создать privileged Pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: privileged-test
  namespace: production
spec:
  containers:
    - name: app
      image: alpine
      securityContext:
        privileged: true
      command: ["sleep", "3600"]
EOF
# Error: pods "privileged-test" is forbidden:
# violates PodSecurity "baseline:latest"

# 3. Проверить audit
kubectl get events -n production | grep "pod-security"

# 4. Restricted Pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: restricted-test
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginxinc/nginx-unprivileged:latest
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      ports:
        - containerPort: 8080
EOF
kubectl get pod restricted-test -n production
# Running
```

### 💡 Практика: как правильно настраивать Pod Security

**✅ ОБЯЗАТЕЛЬНО:**

1. **Baseline** для всех namespace.
2. **Restricted** для production.
3. **Audit mode** перед enforce.
4. **Постепенная миграция.**

**👍 СТОИТ:**

4. **Restricted** для всех.
5. **Исключения** для системных Pod'ов.
6. **Мониторинг** violations.

**❌ НЕ ДЕЛАЙ:**

7. **Не включай enforce сразу.** Сломаешь production.
8. **Не давай Privileged** приложениям.
9. **Не игнорируй warnings.**

### Где мы сейчас

Мы разобрали Pod Security. Теперь — **Security Context**.

---

## 17.5 Security Context: изоляция контейнеров

### 🔌 Проблема: как ограничить контейнер

Pod может быть restricted. Но контейнер внутри — что может?

**Security Context** определяет:

- **User** — от какого пользователя.
- **Group** — группы.
- **Capabilities** — какие capabilities.
- **Filesystem** — read-only?
- **Seccomp** — какие syscalls.
- **AppArmor/SELinux.**

### 📊 Security Context

**Уровни:**

- **Pod-level** (`spec.securityContext`).
- **Container-level** (`spec.containers[].securityContext`).

**Container-level** переопределяет Pod-level.

### 🎯 User и Group

```yaml
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    runAsNonRoot: true
    fsGroup: 1000
  containers:
    - name: app
      image: myapp:1.0
      securityContext:
        runAsUser: 2000  # переопределить
```

**Что даёт:**

- **runAsUser** — UID.
- **runAsGroup** — GID.
- **runAsNonRoot** — запретить root.
- **fsGroup** — GID для volumes.

### 🎯 Capabilities

**Linux capabilities** — разбиение прав root.

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE  # только нужные
```

**Что даёт:** контейнер не может использовать ненужные capabilities.

**Рекомендация:** `drop: [ALL]`, добавить только нужные.

### 🎯 Read-only filesystem

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

**Что даёт:** файловая система контейнера — read-only. Нельзя записать malware.

**Проблема:** приложению нужно писать в `/tmp`, `/var/cache`.

**Решение:** mount emptyDir:

```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```

### 🎯 allowPrivilegeEscalation

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

**Что даёт:** процесс не может получить больше прав (через setuid).

### 🎯 Seccomp

**Seccomp** — фильтрация syscalls.

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

**Типы:**

- **Unconfined** — без ограничений.
- **RuntimeDefault** — default от runtime.
- **Localhost** — свой profile.

**Рекомендация:** `RuntimeDefault`.

### 🎯 AppArmor

```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: runtime/default
```

**Что даёт:** мандатные политики доступа.

### 🎯 Практический пример

**Полный Security Context:**

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
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: myapp
          image: myapp:1.0
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: cache
              mountPath: /var/cache
      volumes:
        - name: tmp
          emptyDir: {}
        - name: cache
          emptyDir: {}
```

**Что даёт:**

- Non-root user.
- No privilege escalation.
- Read-only filesystem (кроме /tmp, /var/cache).
- Только нужные capabilities (никаких).
- Seccomp default.

### 🔬 Практика: Security Context

```bash
# 1. Обычный Pod (root)
kubectl run test --image=alpine --restart=Never -- sleep 3600
kubectl exec test -- id
# uid=0(root) gid=0(root)

# 2. Pod с security context
cat > pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secure
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: alpine
      command: ["sleep", "3600"]
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
EOF
kubectl apply -f pod.yaml

# 3. Проверить
kubectl exec secure -- id
# uid=1000 gid=1000

kubectl exec secure -- ls -la /
# read-only

kubectl exec secure -- touch /test
# touch: /test: Read-only file system

kubectl exec secure -- touch /tmp/test
# OK

# 4. Проверить capabilities
kubectl exec secure -- cat /proc/1/status | grep Cap
# CapEff: 0000000000000000
```

### 💡 Практика: как правильно настраивать Security Context

**✅ ОБЯЗАТЕЛЬНО:**

1. **runAsNonRoot: true.**
2. **runAsUser: 1000+** (не root).
3. **capabilities: drop: [ALL].**
4. **allowPrivilegeEscalation: false.**
5. **seccompProfile: RuntimeDefault.**

**👍 СТОИТ:**

4. **readOnlyRootFilesystem: true** (с emptyDir для /tmp).
5. **fsGroup** для volumes.
6. **AppArmor** профили.

**❌ НЕ ДЕЛАЙ:**

7. **Не запускай от root.**
8. **Не давай capabilities без нужды.**
9. **Не используй privileged.**

### Где мы сейчас

Мы разобрали Security Context. Теперь — **Network Policies**.

---

## 17.6 Network Policies: изоляция трафика

### 🔌 Проблема: все Pod'ы видят всех

По умолчанию в Kubernetes **все Pod'ы могут общаться со всеми**. Даже из разных namespace.

**Это небезопасно.** Если frontend скомпрометирован — он может обратиться к database.

**Решение:** Network Policies.

### 📊 Что такое Network Policy

**Network Policy** — файрвол для Pod'ов.

**Что определяет:**

- **Ingress** — входящий трафик.
- **Egress** — исходящий трафик.
- **Selector** — к каким Pod'ам применяется.

**Opt-in:** по умолчанию всё разрешено. Network Policy **ограничивает**.

### 🎯 Базовая структура

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: myapp-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

**Что даёт:**

- **Ingress:** только от Pod'ов `app=frontend` на порт 8080.
- **Egress:** только к Pod'ам `app=database` на порт 5432.

### 🎯 Default deny

**Самое важное:** запретить всё по умолчанию.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

**Что даёт:** весь трафик запрещён. Нужно явно разрешить.

**Правило:** **default deny** в каждом namespace.

### 🎯 Разрешить DNS

**После default deny DNS тоже блокируется.**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

**Что даёт:** все Pod'ы могут резолвить DNS.

### 🎯 Микросервисы

**Сценарий:**

- **Frontend** → **API** → **Database**.
- **Frontend** не может обратиться к **Database** напрямую.

```yaml
# 1. Default deny
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

# 2. DNS
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
      ports:
        - protocol: UDP
          port: 53

# 3. Frontend → API
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - protocol: TCP
          port: 8080

# 4. API → Database
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-to-database
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432

# 5. API принимает от frontend
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080

# 6. Database принимает от API
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - protocol: TCP
          port: 5432
```

**Что даёт:**

- Frontend → API (8080).
- API → Database (5432).
- Frontend → Database **запрещено**.

### 🎯 CNI с поддержкой

**Network Policies требуют CNI-поддержки:**

| CNI | Поддержка |
|:---|:---|
| **Calico** | ✅ |
| **Cilium** | ✅ + L7 |
| **Weave** | ✅ |
| **kube-router** | ✅ |
| **Flannel** | ❌ (нужен Calico) |

**Проверить:**

```bash
kubectl get pods -n kube-system | grep -E "calico|cilium"
```

### 🎯 L7 Network Policies (Cilium)

**Cilium поддерживает L7:**

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-http-get
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: GET
                path: /api/.*
```

**Что даёт:** только GET на `/api/*`. POST запрещён.

### 🔬 Практика: Network Policy

```bash
# 1. Проверить CNI
kubectl get pods -n kube-system | grep -E "calico|cilium"

# 2. Развернуть приложение
kubectl create deployment frontend --image=nginx -n default
kubectl create deployment api --image=nginx -n default
kubectl expose deployment api --port=80

# 3. Проверить связь
kubectl exec -it frontend-xxx -- curl api
# OK

# 4. Default deny
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
EOF

# 5. Проверить — связь пропала
kubectl exec -it frontend-xxx -- curl api
# timeout

# 6. Разрешить
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 80
EOF

# 7. Проверить
kubectl exec -it frontend-xxx -- curl api
# OK
```

### 💡 Практика: как правильно настраивать Network Policies

**✅ ОБЯЗАТЕЛЬНО:**

1. **Default deny** в каждом namespace.
2. **Разрешить DNS** явно.
3. **Минимальные правила** — только нужное.
4. **CNI с поддержкой.**

**👍 СТОИТ:**

4. **Отдельные policies** для ingress и egress.
5. **L7 policies** (Cilium).
6. **Тестирование** policies.

**❌ НЕ ДЕЛАЙ:**

7. **Не забывай про DNS.**
8. **Не разрешай всё `0.0.0.0/0`.**
9. **Не используй Flannel** без Calico для policies.

### Где мы сейчас

Мы разобрали Network Policies. Теперь — **секреты**.

---

## 17.7 Секреты: шифрование и управление

### 🔌 Проблема: секреты в etcd

Secret хранится в etcd. По умолчанию — **base64**, не шифрование.

**Любой с доступом к etcd — читает все Secrets.**

**Решение:** шифрование.

### 📊 Шифрование etcd

**EncryptionConfiguration:**

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
```

**Настройка apiserver:**

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
    - command:
        - kube-apiserver
        - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
      volumeMounts:
        - name: encryption-config
          mountPath: /etc/kubernetes/encryption-config.yaml
          readOnly: true
  volumes:
    - name: encryption-config
      hostPath:
        path: /etc/kubernetes/encryption-config.yaml
```

**Проверка:**

```bash
# Прочитать Secret из etcd
etcdctl get /registry/secrets/default/my-secret
# k8s:enc:aescbc:v1:key1:...  ← зашифровано
```

**Разберём подробно в Главе 12 (уже разбирали).**

### 🎯 External Secrets

**Секреты в Vault, AWS Secrets Manager, а не в K8s.**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-password
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault
    kind: SecretStore
  target:
    name: db-password
  data:
    - secretKey: password
      remoteRef:
        key: secret/data/db
        property: password
```

**Что даёт:**

- **Секреты в Vault.** Не в K8s.
- **Audit** доступа.
- **Ротация** через Vault.
- **Динамические секреты.**

### 🎯 Sealed Secrets

**Секреты в Git, зашифрованные ключом кластера.**

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-secret
  namespace: production
spec:
  encryptedData:
    password: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
```

**Что даёт:**

- **Секреты в Git** (зашифрованные).
- **Расшифровка** в кластере.
- **GitOps-friendly.**

### 🎯 SOPS

**Шифрование через KMS/age.**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
stringData:
  password: ENC[AES256_GCM,data:abc123...,type:str]
sops:
  kms:
    - arn: arn:aws:kms:us-west-2:...
```

**Что даёт:**

- **KMS** для управления ключами.
- **Multi-cluster.**
- **CI/CD-friendly.**

### 🎯 Что нельзя делать

**1. Секреты в Git.**

Даже в приватном. Даже в `values.yaml`.

**2. Секреты в образах.**

Слои неизменяемы. Секрет останется навсегда.

**3. Секреты в логах.**

Маскирование обязательно.

**4. Секреты в env vars.**

Могут попасть в `kubectl describe`. Лучше volumes.

**5. Один Secret для всех.**

Разные для dev/staging/prod.

### 🎯 Правильная работа

**1. External Secrets + Vault.** Для production.

**2. Sealed Secrets.** Для GitOps.

**3. SOPS.** Для CI/CD.

**4. Шифрование etcd.** Обязательно.

**5. RBAC.** Ограничить доступ к Secrets.

### 🔬 Практика: секреты

```bash
# 1. Проверить шифрование etcd
# (см. Глава 12)

# 2. External Secrets
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace

# 3. SecretStore
cat > secretstore.yaml <<EOF
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
EOF

# 4. ExternalSecret
cat > externalsecret.yaml <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-password
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault
  target:
    name: db-password
  data:
    - secretKey: password
      remoteRef:
        key: secret/data/db
        property: password
EOF

# 5. Проверить
kubectl get externalsecret -n production
kubectl get secret db-password -n production
```

### 💡 Практика: как правильно работать с секретами

**✅ ОБЯЗАТЕЛЬНО:**

1. **External Secrets + Vault** для production.
2. **Sealed Secrets** для GitOps.
3. **Шифрование etcd.**
4. **RBAC** на Secrets.

**👍 СТОИТ:**

4. **Ротация** секретов.
5. **Audit** доступа.
6. **Разные секреты** для окружений.

**❌ НЕ ДЕЛАЙ:**

7. **Не коммить секреты в Git.**
8. **Не хранить в образах.**
9. **Не логировать.**

### Где мы сейчас

Мы разобрали секреты. Теперь — **сканирование образов**.

---

## 17.8 Сканирование образов: Trivy, Grype

### 🔌 Проблема: уязвимости в образах

Образ содержит:

- **OS packages** (Ubuntu, Alpine).
- **Language packages** (Go modules, npm).
- **Binaries.**

В каждом могут быть **CVE**.

**Пример:** `log4j` в Java-приложении. **Log4Shell.** Критическая уязвимость.

**Решение:** сканирование.

### 📊 Trivy

**Trivy** — сканер уязвимостей от Aqua Security.

**Что сканирует:**

- **OS packages.**
- **Language packages.**
- **Misconfigurations** (IaC).
- **Secrets.**
- **SBOM.**

**Установка:**

```bash
# macOS
brew install trivy

# Linux
wget https://github.com/aquasecurity/trivy/releases/latest/download/trivy_0.48.0_Linux-64bit.tar.gz
tar xzf trivy_*.tar.gz
sudo mv trivy /usr/local/bin
```

**Использование:**

```bash
# Образ
trivy image nginx:latest

# Файловая система
trivy fs .

# Kubernetes
trivy k8s cluster

# SBOM
trivy image --format spdx-json nginx:latest > sbom.json
```

**Пример вывода:**

```
nginx:latest (debian 12.4)
==========================
Total: 45 (UNKNOWN: 0, LOW: 20, MEDIUM: 20, HIGH: 5, CRITICAL: 0)

┌──────────────┬────────────────┬──────────┬───────────────┬───────────────┐
│   LIBRARY    │ VULNERABILITY  │ SEVERITY │ INSTALLED     │ FIXED VERSION │
├──────────────┼────────────────┼──────────┼───────────────┼───────────────┤
│ libssl3      │ CVE-2024-0727  │ MEDIUM   │ 3.0.11-1~deb… │ 3.0.11-1~deb… │
│ libc6        │ CVE-2023-6246  │ HIGH     │ 2.36-9+deb12… │ 2.36-9+deb12… │
└──────────────┴────────────────┴──────────┴───────────────┴───────────────┘
```

### 📊 Grype

**Grype** — сканер от Anchore.

**Похож на Trivy.** Используй любой.

```bash
brew install grype
grype nginx:latest
```

### 🎯 Интеграция в CI/CD

**GitLab CI:**

```yaml
trivy-scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  allow_failure: false
```

**Что даёт:** пайплайн падает при HIGH/CRITICAL.

**GitHub Actions:**

```yaml
- name: Run Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'
```

### 🎯 Admission Controller

**Trivy Admission Controller** — блокировать Pod'ы с уязвимостями.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: trivy-webhook
webhooks:
  - name: trivy.aquasecurity.github.io
    # ...
```

**Что даёт:** Pod с CRITICAL не создастся.

### 🎯 Что делать с уязвимостями

**1. Обновить образ.**

```dockerfile
FROM nginx:1.25.4  # вместо 1.25.3 с CVE
```

**2. Обновить пакеты.**

```dockerfile
RUN apt-get update && apt-get upgrade -y && rm -rf /var/lib/apt/lists/*
```

**3. Использовать minimal images.**

`distroless`, `scratch`, `alpine`. Меньше пакетов — меньше CVE.

**4. Регулярно пересобирать.**

Базовые образы обновляются. Пересобирать.

**5. Приоритизировать.**

- **CRITICAL** — fix немедленно.
- **HIGH** — fix в течение недели.
- **MEDIUM** — fix в течение месяца.
- **LOW** — backlog.

### 🎯 Ограничения

**1. Много CVE.**

В базовом образе могут быть десятки CVE. Некоторые не исправлены.

**2. False positives.**

Не все CVE применимы.

**3. Fix not available.**

Некоторые CVE без fix.

**4. Overhead.**

Сканирование замедляет CI.

**Решение:**

- **Exit-code** только для CRITICAL.
- **Ignore** неприменимые CVE.
- **Baseline** для известных.

### 🔬 Практика: Trivy

```bash
# 1. Установить
brew install trivy

# 2. Сканировать образ
trivy image nginx:1.25

# 3. Только HIGH и CRITICAL
trivy image --severity HIGH,CRITICAL nginx:1.25

# 4. Exit code при CRITICAL
trivy image --exit-code 1 --severity CRITICAL nginx:1.25

# 5. JSON вывод
trivy image --format json nginx:1.25 > scan.json

# 6. SBOM
trivy image --format spdx-json nginx:1.25 > sbom.json

# 7. Сканировать кластер
trivy k8s cluster --report summary

# 8. Сканировать файлы
trivy fs --security-checks vuln,config .
```

### 💡 Практика: как правильно сканировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Trivy в CI/CD.**
2. **Exit-code для CRITICAL.**
3. **Регулярная пересборка.**
4. **Minimal images.**

**👍 СТОИТ:**

4. **SBOM** для compliance.
5. **Admission Controller** для блокировки.
6. **Baseline** для известных.

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй CRITICAL.**
8. **Не используй старые образы.**
9. **Не сканируй без приоритизации.**

### Где мы сейчас

Мы разобрали сканирование. Теперь — **supply chain security**.

---

## 17.9 Supply chain security: подписи, SBOM, SLSA

### 🔌 Проблема: откуда образ

Ты скачал образ `myregistry.com/myapp:1.0`. Но:

- **Кто его собрал?**
- **Из какого кода?**
- **Не изменён ли?**
- **Какие зависимости?**

**Supply chain security** отвечает на эти вопросы.

### 📊 Что такое supply chain security

**Supply chain** — путь от кода до production.

```
Developer → Code → CI → Build → Registry → Cluster → Runtime
```

**Каждый шаг — точка атаки.**

**Что может пойти не так:**

- **Код изменён** в CI.
- **Зависимости подменены.**
- **Образ модифицирован** в registry.
- **Не подписан.**
- **Неизвестного происхождения.**

### 🎯 SBOM

**SBOM (Software Bill of Materials)** — список компонентов.

**Что содержит:**

- **Все пакеты** (OS, language).
- **Версии.**
- **Лицензии.**
- **Хеши.**

**Форматы:**

- **SPDX** (Linux Foundation).
- **CycloneDX** (OWASP).

**Генерация:**

```bash
# Trivy
trivy image --format spdx-json myapp:1.0 > sbom.spdx.json
trivy image --format cyclonedx myapp:1.0 > sbom.cyclonedx.json

# Syft
syft myapp:1.0 -o spdx-json > sbom.json
```

**Зачем:**

- **Compliance** (знать, что внутри).
- **Уязвимости** (знать, что обновлять).
- **Лицензии.**
- **Audit.**

### 🎯 Подписи образов

**Cosign** — подпись образов от Sigstore.

**Подпись:**

```bash
# Генерация ключа
cosign generate-key-pair

# Подпись
cosign sign --key cosign.key myregistry.com/myapp:1.0
```

**Проверка:**

```bash
cosign verify --key cosign.pub myregistry.com/myapp:1.0
```

**Что даёт:**

- **Аутентичность.** Образ от того, кто подписал.
- **Целостность.** Не изменён.
- **Неотказуемость.**

### 🎯 Keyless signing

**Sigstore** поддерживает keyless подпись через OIDC.

```bash
cosign sign myregistry.com/myapp:1.0
# Откроет браузер для OIDC
# Подпись через Fulcio + Rekor
```

**Что даёт:**

- **Не нужно управлять ключами.**
- **Audit в Rekor** (прозрачный лог).
- **Привязка к identity** (email, CI).

### 🎯 Admission Controller

**Cosign Admission Controller** — проверять подписи при деплое.

```yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-signature
spec:
  images:
    - glob: "myregistry.com/**"
  authorities:
    - key:
        data: |
          -----BEGIN PUBLIC KEY-----
          ...
          -----END PUBLIC KEY-----
```

**Что даёт:** только подписанные образы.

### 🎯 SLSA

**SLSA (Supply-chain Levels for Software Artifacts)** — framework для supply chain security.

**Уровни:**

- **SLSA 1** — документация build process.
- **SLSA 2** — hosted build, signed provenance.
- **SLSA 3** — hardened build, isolated.
- **SLSA 4** — hermetic, reproducible.

**Что даёт:**

- **Provenance** — откуда образ.
- **Reproducibility** — воспроизводимый build.
- **Isolation** — изолированный build.

**Пример SLSA 3:**

- **GitHub Actions** с OIDC.
- **Cosign** для подписи.
- **Provenance** в Rekor.

### 🎯 Sigstore

**Sigstore** — набор инструментов для supply chain:

- **Cosign** — подписи.
- **Fulcio** — CA для keyless.
- **Rekor** — прозрачный лог.
- **Policy Controller** — admission.

**Что даёт:**

- **Простая подпись.**
- **Проверка.**
- **Audit.**

### 🔬 Практика: supply chain

```bash
# 1. Установить cosign
brew install cosign

# 2. Генерация ключей
cosign generate-key-pair

# 3. Подпись образа
cosign sign --key cosign.key myregistry.com/myapp:1.0

# 4. Проверка
cosign verify --key cosign.pub myregistry.com/myapp:1.0

# 5. Keyless (OIDC)
cosign sign myregistry.com/myapp:1.0

# 6. SBOM
trivy image --format cyclonedx myapp:1.0 > sbom.json

# 7. Attach SBOM к образу
cosign attach sbom --sbom sbom.json myregistry.com/myapp:1.0

# 8. Проверить SBOM
cosign download sbom myregistry.com/myapp:1.0
```

### 💡 Практика: как правильно защищать supply chain

**✅ ОБЯЗАТЕЛЬНО:**

1. **Подписи образов** (Cosign).
2. **SBOM** для каждого образа.
3. **Admission Controller** для проверки.
4. **SLSA 3** для критичных.

**👍 СТОИТ:**

4. **Keyless signing** (Sigstore).
5. **Rekor** для audit.
6. **Provenance** в CI.

**❌ НЕ ДЕЛАЙ:**

7. **Не деплой неподписанные образы.**
8. **Не используй образы без SBOM.**
9. **Не игнорируй provenance.**

### Где мы сейчас

Мы разобрали supply chain. Теперь — **audit logging**.

---

## 17.10 Audit logging

### 🔌 Проблема: кто что делал

Инцидент. Нужно понять: кто, когда, что делал.

**Без audit log — невозможно.**

**Решение:** Kubernetes audit logging.

### 📊 Что такое audit logging

**Audit logging** — логирование всех запросов к API server.

**Что содержит:**

- **Who** — кто (user, ServiceAccount).
- **What** — что (verb, resource).
- **When** — когда.
- **Where** — откуда (IP).
- **Result** — успех/ошибка.

### 🎯 Audit Policy

**Audit Policy** — какие события логировать.

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Не логировать read-only
  - level: None
    verbs: ["get", "list", "watch"]
    resources:
      - group: ""
        resources: ["pods", "services"]
  
  # Логировать Secrets полностью
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
  
  # Логировать всё остальное
  - level: Metadata
    omitStages:
      - RequestReceived
```

**Уровни:**

| Level | Что логируется |
|:---|:---|
| **None** | Ничего |
| **Metadata** | Кто, что, когда |
| **Request** | + тело запроса |
| **RequestResponse** | + тело ответа |

**Правило:** Metadata для большинства. RequestResponse для Secrets.

### 🎯 Настройка

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
    - command:
        - kube-apiserver
        - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
        - --audit-log-path=/var/log/kubernetes/audit.log
        - --audit-log-maxage=30
        - --audit-log-maxbackup=10
        - --audit-log-maxsize=100
      volumeMounts:
        - name: audit-policy
          mountPath: /etc/kubernetes/audit-policy.yaml
          readOnly: true
        - name: audit-log
          mountPath: /var/log/kubernetes/
  volumes:
    - name: audit-policy
      hostPath:
        path: /etc/kubernetes/audit-policy.yaml
    - name: audit-log
      hostPath:
        path: /var/log/kubernetes/
```

### 🎯 Audit log backend

**1. Log file.**

Простой, но нужно собирать.

**2. Webhook.**

Отправка в внешний backend (Falco, Elasticsearch).

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
webhooks:
  - name: audit-webhook
    clientConfig:
      url: https://audit.example.com/log
    throttle:
      qps: 100
```

**3. Cloud (EKS, GKE).**

Managed audit logs. Включить.

### 🎯 Что искать в audit logs

**1. Необычные действия.**

```
- Создание privileged Pod'а.
- Изменение RBAC.
- Чтение Secrets.
- Удаление namespace.
```

**2. Из необычных мест.**

```
- Из неизвестных IP.
- Из необычных ServiceAccount.
- В необычное время.
```

**3. Много ошибок.**

```
- 403 (Forbidden) — попытки получить доступ.
- 401 (Unauthorized) — неверные credentials.
```

### 🎯 Инструменты

**1. Falco.**

Runtime security. Что происходит в Pod'ах.

**2. Tetragon.**

eBPF-based security. От Cilium.

**3. audit2rbac.**

Аудит RBAC из audit logs.

**4. Kubernetes Audit Log Parser.**

Парсинг и анализ.

### 🎯 Пример анализа

```bash
# Кто создавал Pod'ы
jq -r 'select(.verb == "create" and .objectRef.resource == "pods") | .user.username' audit.log | sort | uniq -c

# Кто читал Secrets
jq -r 'select(.objectRef.resource == "secrets" and .verb == "get") | .user.username' audit.log | sort | uniq -c

# Ошибки 403
jq -r 'select(.responseStatus.code == 403) | "\(.user.username) \(.verb) \(.objectRef.resource)"' audit.log | sort | uniq -c | sort -rn | head
```

### 🔬 Практика: audit logging

```bash
# 1. Audit policy
cat > /etc/kubernetes/audit-policy.yaml <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: None
    verbs: ["get", "list", "watch"]
  
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
  
  - level: Metadata
    omitStages:
      - RequestReceived
EOF

# 2. Настроить apiserver (см. выше)
# 3. Перезапустить apiserver
# 4. Проверить логи
tail -f /var/log/kubernetes/audit.log | jq

# 5. Анализ
cat /var/log/kubernetes/audit.log | jq 'select(.verb == "create" and .objectRef.resource == "pods")' | head

# 6. Managed K8s (EKS)
aws eks update-cluster-config --name my-cluster \
  --logging '{"clusterLogging":[{"types":["audit"],"enabled":true}]}'
```

### 💡 Практика: как правильно настраивать audit

**✅ ОБЯЗАТЕЛЬНО:**

1. **Audit policy** для кластера.
2. **RequestResponse** для Secrets.
3. **Metadata** для остального.
4. **Хранение** вне кластера.

**👍 СТОИТ:**

4. **Webhook** для внешнего backend.
5. **Анализ** audit logs.
6. **Алерты** на необычные действия.

**❌ НЕ ДЕЛАЙ:**

7. **Не логируй всё RequestResponse.** Много данных.
8. **Не храни в кластере.**
9. **Не игнорируй audit.**

### Где мы сейчас

Мы разобрали audit logging. Теперь — **Admission Controllers и OPA**.

---

## 17.11 Admission Controllers и OPA

### 🔌 Проблема: как применить политики

Как запретить:

- Pod'ы без resource limits.
- Образы из неразрешённых registry.
- Привилегированные Pod'ы.
- Сервисы без owner.

**Решение:** Admission Controllers и OPA.

### 📊 Admission Controllers

**Admission Controller** — проверяет/модифицирует ресурсы при создании.

**Встроенные:**

- **PodSecurity** — Pod Security Standards.
- **ResourceQuota** — квоты.
- **LimitRanger** — лимиты.
- **ServiceAccount** — автомонтирование.

**Кастомные:**

- **ValidatingWebhookConfiguration** — проверка.
- **MutatingWebhookConfiguration** — модификация.

### 🎯 OPA Gatekeeper

**OPA (Open Policy Agent)** — policy engine.

**Gatekeeper** — OPA для Kubernetes.

**Как работает:**

1. **ConstraintTemplate** — определение policy.
2. **Constraint** — применение policy.
3. **Admission Webhook** — проверка.

**Пример: требовать resource limits:**

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
      validation:
        openAPIV3Schema:
          properties:
            limits:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits
          msg := sprintf("Container %v has no resource limits", [container.name])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: require-resource-limits
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["production"]
  parameters:
    limits: ["cpu", "memory"]
```

**Что даёт:** Pod без limits в `production` — отклонён.

### 🎯 Kyverno

**Kyverno** — альтернатива OPA. Проще.

**Пример: требовать labels:**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: enforce
  rules:
    - name: check-team-label
      match:
        any:
          - resources:
              kinds:
                - Deployment
      validate:
        message: "Label 'team' is required"
        pattern:
          metadata:
            labels:
              team: "?*"
```

**Что даёт:** Deployment без label `team` — отклонён.

**Kyverno vs OPA:**

| Аспект | Kyverno | OPA |
|:---|:---|:---|
| **Язык** | YAML | Rego |
| **Кривая** | Низкая | Средняя |
| **Функции** | Готовые | Гибкие |
| **Для чего** | Kubernetes-native | Универсальный |

### 🎯 Примеры политик

**1. Запретить privileged:**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: enforce
  rules:
    - name: check-privileged
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Privileged mode is not allowed"
        pattern:
          spec:
            containers:
              - securityContext:
                  privileged: false
```

**2. Разрешить только определённые registry:**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-registries
spec:
  validationFailureAction: enforce
  rules:
    - name: check-registry
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Only images from myregistry.com are allowed"
        pattern:
          spec:
            containers:
              - image: "myregistry.com/*"
```

**3. Требовать owner:**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-owner
spec:
  validationFailureAction: enforce
  rules:
    - name: check-owner
      match:
        any:
          - resources:
              kinds:
                - Deployment
                - StatefulSet
      validate:
        message: "Annotation 'owner' is required"
        pattern:
          metadata:
            annotations:
              owner: "?*"
```

### 🎯 Опасности

**1. Слишком строгие политики.**

Заблокируют production. Постепенно.

**2. Audit mode сначала.**

`validationFailureAction: audit` — только логировать.

**3. Исключения.**

Для системных Pod'ов.

**4. Мониторинг.**

Много violations — что-то не так.

### 🔬 Практика: Kyverno

```bash
# 1. Установить Kyverno
helm repo add kyverno https://kyverno.github.io/kyverno
helm install kyverno kyverno/kyverno -n kyverno --create-namespace

# 2. Политика
kubectl apply -f - <<EOF
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: audit  # сначала audit
  rules:
    - name: check-team-label
      match:
        any:
          - resources:
              kinds:
                - Deployment
      validate:
        message: "Label 'team' is required"
        pattern:
          metadata:
            labels:
              team: "?*"
EOF

# 3. Проверить
kubectl create deployment test --image=nginx
kubectl get policyreport -A
# Покажет violations

# 4. Переключить на enforce
kubectl patch clusterpolicy require-labels --type=merge \
  -p '{"spec":{"validationFailureAction":"enforce"}}'

# 5. Проверить
kubectl create deployment test2 --image=nginx
# Error: Label 'team' is required
```

### 💡 Практика: как правильно использовать admission

**✅ ОБЯЗАТЕЛЬНО:**

1. **Audit mode** сначала.
2. **Enforce** после тестирования.
3. **Исключения** для системных.
4. **Мониторинг** violations.

**👍 СТОИТ:**

4. **Kyverno** для простоты.
5. **OPA** для сложных.
6. **Библиотека политик** (Kyverno policies).

**❌ НЕ ДЕЛАЙ:**

7. **Не включай enforce сразу.**
8. **Не забывай про исключения.**
9. **Не игнорируй violations.**

### Где мы сейчас

Мы разобрали admission. Теперь — **zero-trust**.

---

## 17.12 Zero-trust в Kubernetes

### 🔌 Проблема: периметр не защищает

Традиционная модель: **trust the perimeter**. Внутри сети — доверие.

**Проблема:** если периметр пробит — всё внутри доступно.

**Zero-trust:** **никогда не доверяй, всегда проверяй**.

### 📊 Принципы zero-trust

**1. Verify explicitly.**

Всегда аутентифицируй и авторизуй. Не по IP, а по identity.

**2. Least privilege.**

Минимум прав.

**3. Assume breach.**

Предполагай, что злоумышленник уже внутри. Минимизируй blast radius.

### 🎯 Zero-trust в Kubernetes

**1. Identity.**

- **ServiceAccount** для каждого Pod'а.
- **SPIFFE ID** для identity.
- **mTLS** для аутентификации.

**2. Network.**

- **Network Policies** default deny.
- **Service Mesh** mTLS.
- **Egress filtering.**

**3. Access.**

- **RBAC** minimum privilege.
- **Admission Controllers.**
- **OPA/Kyverno.**

**4. Workload.**

- **Pod Security Standards** restricted.
- **Security Context** non-root.
- **Seccomp, AppArmor.**

**5. Data.**

- **Encryption at rest** (etcd).
- **Encryption in transit** (mTLS).
- **External Secrets** (Vault).

**6. Observability.**

- **Audit logs.**
- **Runtime security** (Falco).
- **Anomaly detection.**

### 🎯 Практика zero-trust

**Сценарий:**

- Frontend → API → Database.
- Frontend не может обратиться к Database.

**Что нужно:**

**1. Identity.**

ServiceAccount для каждого:

```yaml
# frontend
serviceAccountName: frontend

# api
serviceAccountName: api

# database
serviceAccountName: database
```

**2. mTLS.**

Через Service Mesh:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT
```

**3. Authorization.**

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-access
spec:
  selector:
    matchLabels:
      app: api
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/frontend"
```

**4. Network Policy.**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

**5. Pod Security.**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

**6. RBAC.**

Минимум прав для каждого ServiceAccount.

**7. Audit.**

Включить audit logging.

**8. Runtime security.**

Falco для detection.

### 🎯 Falco

**Falco** — runtime security.

**Что делает:**

- **Мониторит** syscalls.
- **Обнаруживает** аномалии.
- **Алертит.**

**Пример rule:**

```yaml
- rule: Terminal shell in container
  desc: A shell was used as the entrypoint/exec
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
    and container_entrypoint
  output: >
    A shell was spawned in a container with an attached terminal
  priority: NOTICE
```

**Что даёт:**

- **Shell** в контейнере — алерт.
- **Запись в /etc** — алерт.
- **Network scan** — алерт.

### 🔬 Практика: zero-trust

```bash
# 1. ServiceAccount для каждого
kubectl create sa frontend -n production
kubectl create sa api -n production
kubectl create sa database -n production

# 2. Deployment с ServiceAccount
kubectl patch deployment frontend -n production -p \
  '{"spec":{"template":{"spec":{"serviceAccountName":"frontend"}}}}'

# 3. mTLS (Istio)
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
EOF

# 4. AuthorizationPolicy
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-access
  namespace: production
spec:
  selector:
    matchLabels:
      app: api
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/frontend"
EOF

# 5. Network Policy
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
EOF

# 6. Falco
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco -n falco --create-namespace
```

### 💡 Практика: как построить zero-trust

**✅ ОБЯЗАТЕЛЬНО:**

1. **Identity** для каждого Pod'а.
2. **mTLS** между всеми.
3. **Network Policies** default deny.
4. **Authorization** по identity.
5. **Audit logs.**

**👍 СТОИТ:**

4. **Falco** для runtime.
5. **Pod Security** restricted.
6. **External Secrets.**

**❌ НЕ ДЕЛАЙ:**

7. **Не доверяй по IP.**
8. **Не используй один ServiceAccount.**
9. **Не забывай про egress.**

### Где мы сейчас

Мы разобрали zero-trust. Теперь — **security audit**.

---

## 17.13 Security audit кластера

### 🔌 Проблема: как проверить безопасность

Кластер работает. Но безопасен ли он?

**Решение:** security audit.

### 📊 Инструменты

**1. kube-bench.**

CIS Kubernetes Benchmark.

```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench
```

**Что проверяет:**

- **API server** config.
- **etcd** config.
- **kubelet** config.
- **RBAC.**
- **Network Policies.**

**2. kube-hunter.**

Pentest кластера.

```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-hunter/master/job.yaml
kubectl logs job/kube-hunter
```

**Что проверяет:**

- **Открытые порты.**
- **Слабые credentials.**
- **Misconfigurations.**

**3. Trivy.**

Сканирование образов и конфигурации.

```bash
trivy k8s cluster --report summary
trivy k8s --report all
```

**4. kubesec.**

Анализ Pod'ов.

```bash
kubesec scan pod.yaml
```

**5. Polaris.**

Best practices.

```bash
kubectl apply -f https://raw.githubusercontent.com/FairwindsOps/polaris/master/stable/polaris.yaml
```

**Что проверяет:**

- **Resource limits.**
- **Health checks.**
- **Security context.**
- **Image tags.**

**6. Kubescape.**

От ARMO. Compliance frameworks.

```bash
kubescape scan framework nsa
kubescape scan framework mitre
```

**Что проверяет:**

- **NSA.**
- **MITRE ATT&CK.**
- **CIS.**

### 🎯 Чек-лист безопасности

**1. Authentication.**

- [ ] OIDC для пользователей.
- [ ] X.509 для компонентов.
- [ ] ServiceAccount для Pod'ов.
- [ ] Аудит использования credentials.

**2. Authorization.**

- [ ] RBAC с минимальными правами.
- [ ] Не используется `cluster-admin`.
- [ ] Аудит RBAC.

**3. Network.**

- [ ] Network Policies default deny.
- [ ] mTLS между сервисами.
- [ ] Egress filtering.
- [ ] Ingress с TLS.

**4. Workload.**

- [ ] Pod Security Standards.
- [ ] Non-root containers.
- [ ] Read-only filesystem.
- [ ] Seccomp, AppArmor.
- [ ] Resource limits.

**5. Data.**

- [ ] Encryption at rest (etcd).
- [ ] External Secrets.
- [ ] Ротация.
- [ ] RBAC на Secrets.

**6. Supply chain.**

- [ ] Image scanning.
- [ ] Подписи образов.
- [ ] SBOM.
- [ ] Admission Controllers.

**7. Observability.**

- [ ] Audit logs.
- [ ] Runtime security (Falco).
- [ ] Мониторинг.
- [ ] Алерты.

**8. Incident response.**

- [ ] План реагирования.
- [ ] Runbooks.
- [ ] Backup.
- [ ] Disaster recovery.

### 🎯 Регулярность

**Регулярно:**

- **Ежедневно:** audit logs, алерты.
- **Еженедельно:** scanning новых образов.
- **Ежемесячно:** RBAC audit, политики.
- **Ежеквартально:** полный security audit, pentest.
- **Ежегодно:** compliance audit.

### 🔬 Практика: security audit

```bash
# 1. kube-bench
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench | head -100

# 2. Trivy
trivy k8s cluster --report summary

# 3. Polaris
kubectl apply -f https://raw.githubusercontent.com/FairwindsOps/polaris/master/stable/polaris.yaml
kubectl port-forward -n polaris svc/polaris-dashboard 8080:80
# http://localhost:8080

# 4. Kubescape
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash
kubescape scan framework nsa --submit=false

# 5. kube-hunter
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-hunter/master/job.yaml
kubectl logs job/kube-hunter
```

### 💡 Практика: как правильно делать audit

**✅ ОБЯЗАТЕЛЬНО:**

1. **kube-bench** для CIS.
2. **Trivy** для образов.
3. **Регулярно.**
4. **Фиксить** находки.

**👍 СТОИТ:**

4. **Kubescape** для compliance.
5. **Polaris** для best practices.
6. **Pentest** ежегодно.

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй находки.**
8. **Не делай audit один раз.**
9. **Не забывай про remediation.**

### Где мы сейчас

Мы разобрали audit. Теперь — **диагностика**.

---

## 17.14 Диагностика проблем

### 🔌 Проблема: что-то не работает

Настроил безопасность. Что-то сломалось.

### 🔍 Типичные проблемы

**1. Pod не создаётся.**

**Причины:**

- Pod Security Standards.
- Admission Controller.
- RBAC.

**Диагностика:**

```bash
kubectl describe pod myapp-xxx
# Events покажут ошибку

kubectl get events -n production --sort-by=.lastTimestamp
```

**2. RBAC forbidden.**

**Причины:**

- Нет прав.
- Неправильный RoleBinding.
- Неправильный ServiceAccount.

**Диагностика:**

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:production:myapp \
  -n production

kubectl describe rolebinding -n production
kubectl describe clusterrolebinding | grep myapp
```

**3. Network Policy блокирует.**

**Причины:**

- Default deny.
- Нет allow rule.
- DNS не разрешён.

**Диагностика:**

```bash
kubectl get networkpolicies -n production
kubectl describe networkpolicy -n production

# Тест
kubectl exec -it frontend-xxx -- curl api
# timeout

# Временно удалить policy для теста
kubectl delete networkpolicy default-deny -n production
```

**4. Secret недоступен.**

**Причины:**

- RBAC.
- Не создан.
- External Secret не синхронизировался.

**Диагностика:**

```bash
kubectl get secret myapp-secret -n production
kubectl describe externalsecret myapp-secret -n production
kubectl logs -n external-secrets deployment/external-secrets
```

**5. mTLS не работает.**

**Причины:**

- PERMISSIVE режим.
- Сертификаты не выданы.
- Sidecar не готов.

**Диагностика:**

```bash
kubectl get peerauthentication -A
istioctl authn tls-check myapp-xxx api.production.svc.cluster.local
istioctl proxy-config secret myapp-xxx
```

**6. Image Pull Error.**

**Причины:**

- RBAC на registry.
- Подпись не проверена.
- Образ не существует.

**Диагностика:**

```bash
kubectl describe pod myapp-xxx | grep -A 5 Events
# Failed to pull image
# unauthorized
# signature verification failed
```

**7. Admission Controller блокирует.**

**Причины:**

- Policy violation.
- Enforce mode.

**Диагностика:**

```bash
kubectl get events -A | grep -i policy
kubectl get constrainttemplates
kubectl get constraints -A
kubectl get clusterpolicy
kubectl get policyreport -A
```

### 🎯 Логи

```bash
# API server
kubectl logs -n kube-system kube-apiserver-xxx

# Admission webhooks
kubectl logs -n gatekeeper-system deployment/gatekeeper-controller-manager
kubectl logs -n kyverno deployment/kyverno

# External Secrets
kubectl logs -n external-secrets deployment/external-secrets

# Istio
kubectl logs -n istio-system deployment/istiod
kubectl logs myapp-xxx -c istio-proxy

# Falco
kubectl logs -n falco daemonset/falco
```

### 🎯 Общие команды

```bash
# Все ресурсы
kubectl get all -n production

# События
kubectl get events -A --sort-by=.lastTimestamp

# Describe
kubectl describe pod myapp-xxx

# Auth check
kubectl auth can-i --list --as=system:serviceaccount:production:myapp

# Audit logs (managed K8s)
aws logs tail /aws/eks/my-cluster/cluster --follow
```

### 🔬 Практика: диагностика

```bash
# 1. Pod не создаётся
kubectl apply -f pod.yaml
# Error: violates PodSecurity "restricted:latest"

# 2. Проверить namespace labels
kubectl get namespace production --show-labels

# 3. RBAC
kubectl auth can-i get secrets \
  --as=system:serviceaccount:production:myapp \
  -n production
# no

# 4. Посмотреть RoleBinding
kubectl get rolebinding -n production -o yaml | grep -A 10 myapp

# 5. Network Policy
kubectl get networkpolicy -n production

# 6. Secret
kubectl get secret myapp-secret -n production
kubectl describe externalsecret myapp-secret -n production

# 7. Admission
kubectl get policyreport -A

# 8. Logs
kubectl logs -n kyverno deployment/kyverno | grep myapp
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Events** — первый источник.
2. **Describe** ресурсов.
3. **Auth can-i** для RBAC.
4. **Логи** admission controllers.

**👍 СТОИТ:**

4. **Audit logs** для глубокого анализа.
5. **Policy reports** для violations.
6. **Тестировать** в dev.

**❌ НЕ ДЕЛАЙ:**

7. **Не отключай политики** для «быстрого фикса».
8. **Не давай `cluster-admin`** для «проверки».
9. **Не игнорируй warnings.**

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **RBAC** | Role-Based Access Control. |
| **Role** | Набор прав в namespace. |
| **ClusterRole** | Набор прав в кластере. |
| **RoleBinding** | Связь Role с субъектом. |
| **ServiceAccount** | Identity для Pod'ов. |
| **Pod Security Standards** | Три уровня безопасности Pod'ов. |
| **Pod Security Admission** | Реализация PSS. |
| **Security Context** | Настройки безопасности контейнера. |
| **Capabilities** | Разбиение прав root. |
| **Seccomp** | Фильтрация syscalls. |
| **AppArmor** | Мандатные политики доступа. |
| **Network Policy** | Файрвол для Pod'ов. |
| **Default deny** | Запретить всё по умолчанию. |
| **etcd encryption** | Шифрование Secrets at rest. |
| **External Secrets** | Секреты из Vault. |
| **Sealed Secrets** | Шифрование для Git. |
| **SOPS** | Secrets OPerationS. |
| **Trivy** | Сканер уязвимостей. |
| **Grype** | Сканер от Anchore. |
| **SBOM** | Software Bill of Materials. |
| **Cosign** | Подпись образов. |
| **Sigstore** | Supply chain security. |
| **SLSA** | Supply-chain Levels. |
| **Audit logging** | Логирование API запросов. |
| **Admission Controller** | Проверка при создании. |
| **OPA** | Open Policy Agent. |
| **Gatekeeper** | OPA для K8s. |
| **Kyverno** | Policy engine. |
| **Zero-trust** | Никогда не доверяй. |
| **Falco** | Runtime security. |
| **kube-bench** | CIS benchmark. |
| **kube-hunter** | Pentest. |
| **Kubescape** | Compliance scanning. |

---

## Что мы узнали?

- **Модель безопасности** — многослойная: Cloud, Cluster, Container, Code.
- **RBAC** — Role, ClusterRole, RoleBinding, ClusterRoleBinding. Минимум прав.
- **ServiceAccount** — identity для Pod'ов. Projected token с TTL.
- **Pod Security Standards** — Privileged, Baseline, Restricted. Admission через labels.
- **Security Context** — non-root, capabilities, read-only FS, seccomp.
- **Network Policies** — default deny, разрешить нужное. Требует CNI.
- **Секреты** — external-secrets, sealed-secrets, SOPS. Шифрование etcd.
- **Сканирование** — Trivy, Grype. Exit-code для CRITICAL.
- **Supply chain** — Cosign, SBOM, SLSA, Sigstore.
- **Audit logging** — кто, что, когда. Policy для уровней.
- **Admission** — OPA Gatekeeper, Kyverno. Audit → enforce.
- **Zero-trust** — identity, mTLS, default deny, RBAC.
- **Security audit** — kube-bench, Trivy, Kubescape, Polaris.
- **Диагностика** — events, describe, auth can-i, logs.

---

## Типичные ошибки

- ❌ **Секреты в Git.** Даже приватный.
- ❌ **`cluster-admin` для приложений.**
- ❌ **`default` ServiceAccount.**
- ❌ **Privileged Pod'ы.**
- ❌ **Root в контейнере.**
- ❌ **Все capabilities.**
- ❌ **Нет Network Policies.**
- ❌ **Нет default deny.**
- ❌ **Нет шифрования etcd.**
- ❌ **Нет сканирования образов.**
- ❌ **Нет audit logs.**
- ❌ **Admission enforce сразу.**
- ❌ **Нет runtime security.**
- ❌ **Не делаешь audit.**
- ❌ **Игнорируешь findings.**

---

## Для быстрого повторения

- **RBAC:** Role, ClusterRole, RoleBinding. `kubectl auth can-i`.
- **ServiceAccount:** отдельный на приложение. Projected token.
- **Pod Security:** Privileged, Baseline, Restricted. Labels namespace.
- **Security Context:** runAsNonRoot, drop capabilities, readOnlyRootFilesystem, seccomp.
- **Network Policy:** default deny + allow. CNI с поддержкой.
- **Секреты:** External Secrets, Sealed Secrets, SOPS. Encryption etcd.
- **Trivy:** сканирование. Exit-code CRITICAL.
- **Cosign:** подписи. SBOM.
- **Audit:** policy, webhook, анализ.
- **OPA/Kyverno:** policies. Audit → enforce.
- **Zero-trust:** identity, mTLS, default deny.
- **Audit:** kube-bench, Trivy, Kubescape.
- **Диагностика:** events, describe, auth can-i, logs.

---

## Вопросы для самопроверки

1. Что такое модель безопасности Kubernetes? 4C.
2. Что такое RBAC? Role, ClusterRole, RoleBinding?
3. Что такое ServiceAccount? Зачем нужен?
4. Что такое Pod Security Standards? Три уровня?
5. Что такое Security Context? Что настраивает?
6. Что такое Network Policy? Default deny?
7. Как защитить секреты в Kubernetes?
8. Что такое Trivy? Как использовать?
9. Что такое supply chain security? Cosign, SBOM, SLSA?
10. Что такое audit logging? Что логирует?
11. Что такое OPA Gatekeeper и Kyverno?
12. Что такое zero-trust в Kubernetes?
13. Как провести security audit кластера?
14. Pod не создаётся — как диагностировать?
15. Что такое Falco? Зачем нужен?

---

## Ответы

**1. Модель безопасности**

4C: Cloud (инфраструктура), Cluster (K8s), Container (образы, runtime), Code (приложение). Каждый слой защищает следующий.

**2. RBAC**

Role (namespace), ClusterRole (кластер), RoleBinding (связь Role с субъектом), ClusterRoleBinding (связь ClusterRole). Verbs: get, list, create, update, delete.

**3. ServiceAccount**

Identity для Pod'ов. Токен для API. Projected token с TTL. Не использовать `default`. Автомонтирование отключать если не нужно.

**4. Pod Security Standards**

Privileged (без ограничений), Baseline (минимальные), Restricted (строгие). Admission через labels namespace: enforce, audit, warn.

**5. Security Context**

runAsUser, runAsGroup, runAsNonRoot, fsGroup, capabilities (drop ALL), allowPrivilegeEscalation, readOnlyRootFilesystem, seccompProfile, AppArmor.

**6. Network Policy**

Файрвол для Pod'ов. Ingress, egress, podSelector. Default deny + allow. Требует CNI с поддержкой (Calico, Cilium). Не забыть DNS.

**7. Секреты**

External Secrets (Vault), Sealed Secrets (Git), SOPS (KMS). Шифрование etcd (EncryptionConfiguration). RBAC на Secrets. Ротация.

**8. Trivy**

Сканер уязвимостей. `trivy image nginx:latest`. Exit-code для CRITICAL. Интеграция в CI/CD. SBOM generation.

**9. Supply chain**

Cosign для подписей, SBOM (SPDX, CycloneDX), SLSA (уровни 1-4), Sigstore (Fulcio, Rekor). Admission Controller для проверки.

**10. Audit logging**

Логирование API запросов. Who, what, when, where, result. Policy для уровней (None, Metadata, Request, RequestResponse). Webhook для backend.

**11. OPA Gatekeeper и Kyverno**

Policy engines. Gatekeeper: Rego, ConstraintTemplate + Constraint. Kyverno: YAML, ClusterPolicy. Audit → enforce. Мониторинг violations.

**12. Zero-trust**

Никогда не доверяй, всегда проверяй. Identity (ServiceAccount, SPIFFE), mTLS, default deny Network Policies, RBAC, Pod Security, audit logs.

**13. Security audit**

kube-bench (CIS), Trivy (образы), Kubescape (compliance), Polaris (best practices), kube-hunter (pentest). Регулярно. Фиксить findings.

**14. Pod не создаётся**

1. `kubectl describe pod` — Events.
2. Pod Security: labels namespace.
3. RBAC: `kubectl auth can-i`.
4. Admission: policy reports.
5. Network Policy.
6. Логи admission controllers.

**15. Falco**

Runtime security. Мониторит syscalls. Обнаруживает аномалии (shell в контейнере, запись в /etc). Алертит. eBPF-based.

---

## Куда идти дальше?

Мы разобрали безопасность Kubernetes. Теперь ты знаешь:

- Модель безопасности.
- RBAC.
- ServiceAccount.
- Pod Security.
- Security Context.
- Network Policies.
- Секреты.
- Сканирование.
- Supply chain.
- Audit.
- Admission.
- Zero-trust.
- Security audit.
- Диагностика.

**Следующая — Глава 18: Инфраструктура как код — Terraform.** Она уже написана, но требует переименования (была отправлена как Глава 14). Скажи «дальше» — и я отправлю её в правильной нумерации с обновлённым заголовком.