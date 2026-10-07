# 🎭 Глава 19: Конфигурационное управление — Ansible

**Что вы узнаете:**
- Что такое конфигурационное управление и зачем оно нужно.
- Чем Ansible отличается от Terraform, Chef, Puppet.
- Как работает Ansible: inventory, playbooks, tasks, modules.
- Что такое идемпотентность и почему это критично.
- Как использовать roles для переиспользования.
- Как работать с переменными, шаблонами, handlers.
- Как хранить секреты через Ansible Vault.
- Как использовать динамические inventory (облака, K8s).
- Как тестировать playbooks через Molecule.
- Как интегрировать Ansible с CI/CD.

**После прочтения вы сможете:**
- Написать playbook для настройки серверов.
- Создать role с переменными и шаблонами.
- Использовать Vault для секретов.
- Настроить динамический inventory для AWS/GCP.
- Тестировать playbooks.
- Интегрировать Ansible в CI/CD.
- Диагностировать типичные ошибки.

---

## Содержание

- [19.0 Пролог: 100 серверов и одна команда](#190-пролог-100-серверов-и-одна-команда)
- [19.1 Что такое конфигурационное управление](#191-что-такое-конфигурационное-управление)
- [19.2 Ansible vs альтернативы](#192-ansible-vs-альтернативы)
- [19.3 Inventory: список серверов](#193-inventory-список-серверов)
- [19.4 Playbooks, tasks, modules](#194-playbooks-tasks-modules)
- [19.5 Идемпотентность](#195-идемпотентность)
- [19.6 Variables, facts, templates](#196-variables-facts-templates)
- [19.7 Handlers и notify](#197-handlers-и-notify)
- [19.8 Roles: переиспользование кода](#198-roles-переиспользование-кода)
- [19.9 Ansible Vault: секреты](#199-ansible-vault-секреты)
- [19.10 Динамический inventory](#1910-динамический-inventory)
- [19.11 Ansible в CI/CD](#1911-ansible-в-cicd)
- [19.12 Тестирование playbooks](#1912-тестирование-playbooks)
- [19.13 Диагностика проблем](#1913-диагностика-проблем)
- [Глоссарий](#глоссарий)
- [Что мы узнали?](#что-мы-узнали)
- [Типичные ошибки](#типичные-ошибки)
- [Для быстрого повторения](#для-быстрого-повторения)
- [Вопросы для самопроверки](#вопросы-для-самопроверки)
- [Ответы](#ответы)
- [Куда идти дальше?](#куда-идти-дальше)

---

## 19.0 Пролог: 100 серверов и одна команда

Ты — DevOps-инженер. У тебя 100 серверов. На каждом нужно:

- **Установить nginx.**
- **Настроить конфиг** (свои параметры для каждого окружения).
- **Создать пользователя** `appuser`.
- **Настроить firewall** (открыть порты 80, 443).
- **Установить сертификаты** (TLS).
- **Настроить логирование** в CloudWatch.
- **Установить node_exporter** для Prometheus.

**Как ты это делаешь?** Заходишь на каждый сервер по SSH. Вводишь команды вручную. Или пишешь bash-скрипт, который запускаешь на каждом.

**Проблемы:**

- **Медленно.** 100 серверов × 15 минут = 25 часов.
- **Неповторяемо.** Каждый раз что-то забываешь, опечатываешься.
- **Не идемпотентно.** Bash-скрипт запустишь дважды — может сломать.
- **Нет истории.** Что, когда, зачем изменено — неизвестно.
- **Не масштабируется.** Добавил 10 серверов — снова вручную.

**Ansible решает эти проблемы.**

Ты пишешь **playbook** — описание желаемого состояния. Запускаешь одной командой на всех серверах. **Идемпотентно** — можно запускать много раз. **Декларативно** — описываешь что, не как. **Версионируется** в Git.

```bash
ansible-playbook -i inventory site.yml
```

И через 5 минут — все 100 серверов настроены одинаково.

В этой главе мы разберём Ansible от основ до продвинутых техник. Ты научишься писать playbooks, roles, использовать Vault, динамические inventory, тестировать.

Это — Первый путь DevOps (Flow) в действии. Из Главы 0: автоматизация конфигурации серверов.

---

## 19.1 Что такое конфигурационное управление

### 🔌 Проблема: ручная настройка не масштабируется

Традиционный подход к настройке серверов:

1. **Ручные SSH-команды.** Зайти на сервер, ввести команды.
2. **Bash-скрипты.** Написать скрипт, запустить на серверах.
3. **Документация в wiki.** «Как настроить nginx: ...»

**Проблемы:**

- **Неповторяемо.** Сервер A настроен одним способом, сервер B — другим.
- **Не идемпотентно.** Скрипт запустишь дважды — сломается.
- **Нет состояния.** Нельзя узнать, что настроено.
- **Нет отката.** Если что-то сломалось — откатывать вручную.
- **Медленно.** Ручная работа — часы.

### 📦 Что такое конфигурационное управление

**Конфигурационное управление (Configuration Management, CM)** — подход, при котором состояние серверов описывается **декларативно** в коде.

**Ключевые принципы:**

1. **Декларативность.** Описываешь **что** должно быть, не **как**.
2. **Идемпотентность.** Многократный запуск = один результат.
3. **Версионирование.** Всё в Git.
4. **Инвентаризация.** Список управляемых серверов.
5. **Автоматизация.** Запускается через CI/CD.

### 📊 Ansible vs Terraform

**Важное различие:**

| Аспект | Terraform | Ansible |
|:---|:---|:---|
| **Что управляет** | Инфраструктура (VPC, VM, БД) | Конфигурация серверов |
| **Подход** | Декларативный | Декларативный + императивный |
| **Агенты** | Нет | Нет (SSH) |
| **State** | Есть | Нет |
| **Порядок** | Не важен (граф зависимостей) | Важен (последовательность задач) |
| **Идемпотентность** | Да | Да |
| **Язык** | HCL | YAML |

**Правило:**

- **Terraform** — создать VPC, EC2, RDS.
- **Ansible** — настроить nginx, пользователей, конфиги на EC2.

**Они дополняют друг друга.**

### 🎯 Преимущества Ansible

**1. Агентless.**

Не нужен агент на серверах. Работает через **SSH**.

**2. Простой язык.**

YAML — легко читать и писать.

**3. Идемпотентность.**

Можно запускать много раз.

**4. Большое сообщество.**

Много модулей (3000+), ролей (Ansible Galaxy).

**5. Кроссплатформенность.**

Linux, Windows, сетевые устройства, облака.

**6. Push-based.**

Не нужен сервер управления (в отличие от Puppet/Chef).

### 🎯 Инструменты CM

| Инструмент | Подход | Агент | Язык |
|:---|:---|:---|:---|
| **Ansible** | Push | Нет | YAML |
| **Chef** | Pull | Да | Ruby |
| **Puppet** | Pull | Да | DSL |
| **SaltStack** | Pull/Push | Да | YAML + Python |
| **CFEngine** | Pull | Да | DSL |

**Ansible — самый популярный.** Простой, без агентов.

### 💡 Практика: что важно понять про CM

**✅ ОБЯЗАТЕЛЬНО:**

1. **CM — это декларативность.** Описываешь состояние.
2. **CM — это идемпотентность.** Много запусков — один результат.
3. **CM — это Git.** Версионируется.

**👍 СТОИТ:**

4. **Ansible для конфигурации серверов.**
5. **Terraform для инфраструктуры.**
6. **Вместе** — полный lifecycle.

**❌ НЕ ДЕЛАЙ:**

7. **Не используй Ansible для создания инфраструктуры.** Terraform лучше.
8. **Не пиши bash-скрипты, если можно Ansible.** Идемпотентность.
9. **Не настраивай серверы вручную.**

### Где мы сейчас

Мы разобрали, что такое CM. Теперь — **Ansible vs альтернативы**.

---

## 19.2 Ansible vs альтернативы

### 🔌 Проблема: какой инструмент CM выбрать

Ansible — не единственный. Есть Chef, Puppet, SaltStack. Как выбрать?

### 📊 Сравнение

| Инструмент | Архитектура | Язык | Кривая обучения | Популярность |
|:---|:---|:---|:---|:---|
| **Ansible** | Push (SSH) | YAML | Низкая | Высокая |
| **Chef** | Pull (агент) | Ruby | Высокая | Средняя |
| **Puppet** | Pull (агент) | DSL | Высокая | Средняя |
| **SaltStack** | Pull/Push | YAML + Python | Средняя | Средняя |

### 🎯 Ansible

**Push-based, агентless.**

**Плюсы:**

- **Простой.** YAML, легко читать.
- **Агентless.** Только SSH.
- **Большое сообщество.** Ansible Galaxy.
- **3000+ модулей.**
- **Кроссплатформенность.**
- **Не нужен сервер управления.**

**Минусы:**

- **Push-based.** Медленнее для больших кластеров (1000+ серверов).
- **YAML ограничен** для сложной логики.
- **Нет состояния** (нет «state file»).

**Когда использовать:**

- **Небольшие и средние инфраструктуры** (до 1000 серверов).
- **Одноразовые задачи** (bootstrap, deploy).
- **Когда нужна простота.**
- **Когда нельзя ставить агентов.**

### 🎯 Chef

**Pull-based, с агентом (chef-client).**

**Плюсы:**

- **Ruby DSL** — мощный язык.
- **Хорошо для больших инфраструктур.**
- **Chef Server** — центральное управление.
- **Test-driven** (ChefSpec).

**Минусы:**

- **Агент на каждом сервере.**
- **Chef Server** — дополнительная инфраструктура.
- **Крутая кривая обучения.** Ruby + Chef DSL.
- **Меньше сообщество,** чем у Ansible.

**Когда использовать:**

- **Большие инфраструктуры** (1000+ серверов).
- **Когда нужна сложная логика** (Ruby).
- **Когда нужен центральный сервер управления.**

### 🎯 Puppet

**Pull-based, с агентом (puppet-agent).**

**Плюсы:**

- **Декларативный DSL.**
- **Хорошо для compliance** (аудит, отчёты).
- **Puppet Enterprise** — managed решение.
- **Большое сообщество** (Puppet Forge).

**Минусы:**

- **Агент на каждом сервере.**
- **Puppet Server** — дополнительная инфраструктура.
- **Крутая кривая обучения.** Puppet DSL.
- **Медленнее** Ansible.

**Когда использовать:**

- **Большие инфраструктуры** с требованиями к compliance.
- **Когда нужен аудит и отчёты.**

### 🎯 SaltStack

**Pull/Push, с агентом (minion).**

**Плюсы:**

- **Быстрый.** ZeroMQ для коммуникации.
- **Мощный.** Python + YAML.
- **Поддерживает и push, и pull.**
- **Event-driven** (реакция на события).

**Минусы:**

- **Агент на серверах.**
- **Salt Master** — дополнительная инфраструктура.
- **Сложнее** Ansible.

**Когда использовать:**

- **Большие инфраструктуры,** где важна скорость.
- **Когда нужна event-driven архитектура.**

### 📊 Когда что использовать

| Сценарий | Инструмент |
|:---|:---|
| **Малая/средняя инфраструктура** | Ansible |
| **Одноразовые задачи** | Ansible |
| **Bootstrap серверов** | Ansible |
| **Большая инфраструктура (1000+)** | Chef / Puppet / Salt |
| **Compliance и аудит** | Puppet |
| **Сложная логика** | Chef (Ruby) |
| **Event-driven** | SaltStack |
| **Нельзя ставить агентов** | Ansible |

**Рекомендация:** **Ansible** для большинства случаев. Простой, агентless, большое сообщество.

### 🎯 Ansible в современном мире

**Ansible + Terraform + Kubernetes:**

- **Terraform** создаёт инфраструктуру (VPC, EC2, EKS).
- **Ansible** настраивает серверы (nginx, users, configs).
- **Kubernetes** оркестрирует контейнеры.

**Для Kubernetes** Ansible используется реже (Helm, ArgoCD). Но для:

- **Bootstrap нод** (kubelet, containerd).
- **Настройка control plane.**
- **Установка CNI.**
- **Всё ещё актуален.**

**Ansible + Packer:**

- **Packer** создаёт образы (AMI, VM).
- **Ansible** используется как provisioner внутри Packer.

### 🔬 Практика: установка

```bash
# macOS
brew install ansible

# Ubuntu/Debian
sudo apt update
sudo apt install ansible

# RHEL/CentOS
sudo dnf install ansible

# Через pip (любая ОС)
pip install ansible

# Проверка
ansible --version
# ansible [core 2.16.0]
```

### 💡 Практика: как выбирать инструмент

**✅ ОБЯЗАТЕЛЬНО:**

1. **Ansible для малых/средних инфраструктур.**
2. **Chef/Puppet для больших с compliance.**
3. **SaltStack для event-driven.**

**👍 СТОИТ:**

4. **Ansible + Terraform** вместе.
5. **Ansible в Packer** для образов.

**❌ НЕ ДЕЛАЙ:**

6. **Не используй Chef/Puppet для 5 серверов.** Overkill.
7. **Не используй Ansible для 5000 серверов.** Медленно.

### Где мы сейчас

Мы разобрали альтернативы. Теперь — **inventory** — список серверов.

---

## 19.3 Inventory: список серверов

### 🔌 Проблема: как Ansible узнаёт о серверах

Ansible должен знать, **какие серверы** настраивать. Список серверов называется **inventory**.

### 📊 Формат inventory

**INI-формат (простой):**

```ini
# inventory.ini
[webservers]
web1.example.com
web2.example.com
web3.example.com

[databases]
db1.example.com
db2.example.com

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

**YAML-формат:**

```yaml
# inventory.yaml
all:
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
  children:
    webservers:
      hosts:
        web1.example.com:
        web2.example.com:
        web3.example.com:
    databases:
      hosts:
        db1.example.com:
        db2.example.com:
```

**YAML предпочтительнее** — структурированнее.

### 🎯 Группы

**Группы** — логическая группировка серверов.

```yaml
all:
  children:
    webservers:
      hosts:
        web1:
        web2:
    databases:
      hosts:
        db1:
        db2:
    production:
      children:
        webservers:
        databases:
    staging:
      hosts:
        stg-web1:
        stg-db1:
```

**Вложенные группы** (parent/child):

```yaml
all:
  children:
    production:
      children:
        prod_webservers:
          hosts:
            prod-web1:
            prod-web2:
```

**Использование в playbook:**

```yaml
- hosts: webservers        # все web-серверы
- hosts: databases         # все БД
- hosts: production        # всё в production
- hosts: all               # все серверы
```

### 🎯 Переменные в inventory

**Host variables:**

```yaml
all:
  hosts:
    web1.example.com:
      ansible_host: 10.0.1.10          # IP, если DNS нет
      nginx_port: 8080
      environment: production
```

**Group variables:**

```yaml
all:
  children:
    webservers:
      vars:
        nginx_worker_processes: 4
        nginx_worker_connections: 1024
      hosts:
        web1:
        web2:
```

**`all` vars:**

```yaml
all:
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
    ansible_python_interpreter: /usr/bin/python3
  children:
    webservers:
      hosts:
        web1:
        web2:
```

### 🎯 Параметры подключения

| Параметр | Что означает |
|:---|:---|
| `ansible_host` | IP или DNS сервера |
| `ansible_user` | SSH-пользователь |
| `ansible_port` | SSH-порт |
| `ansible_ssh_private_key_file` | SSH-ключ |
| `ansible_ssh_pass` | SSH-пароль (не рекомендуется) |
| `ansible_python_interpreter` | Python на сервере |
| `ansible_become` | Использовать sudo |
| `ansible_become_user` | Какой пользователь для sudo |

**Пример:**

```yaml
all:
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
    ansible_python_interpreter: /usr/bin/python3
  hosts:
    web1:
      ansible_host: 10.0.1.10
      ansible_port: 2222
      ansible_become: true
      ansible_become_user: root
```

### 🎯 `ansible.cfg`

**Конфигурация Ansible:**

```ini
# ansible.cfg
[defaults]
inventory = ./inventory.yaml
remote_user = ubuntu
private_key_file = ~/.ssh/id_rsa
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml
callback_whitelist = profile_tasks

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

**Что даёт:**

- Не нужно каждый раз указывать `-i inventory.yaml`.
- Общие параметры.
- Ускорение (pipelining, ControlMaster).

### 🎯 Команды для работы с inventory

```bash
# Показать все хосты
ansible all --list-hosts

# Показать группу
ansible webservers --list-hosts

# Показать переменные для хоста
ansible-inventory --host web1

# Показать всё дерево
ansible-inventory --list

# Показать в YAML
ansible-inventory --list --yaml

# Ping всех серверов
ansible all -m ping

# Выполнить команду
ansible all -m shell -a "uptime"

# Только определённая группа
ansible webservers -m shell -a "df -h"
```

### 🎯 Patterns

**Patterns** — способы указать, на каких хостах запускать.

| Pattern | Что означает |
|:---|:---|
| `all` | Все хосты |
| `webservers` | Группа |
| `web1` | Один хост |
| `web1:web2` | Несколько |
| `webservers:databases` | Объединение групп |
| `webservers:&production` | Пересечение |
| `webservers:!production` | Исключение |
| `web*.example.com` | Wildcard |

**Пример:**

```bash
# Все webservers, кроме production
ansible 'webservers:!production' -m ping

# Все в webservers ИЛИ databases
ansible 'webservers:databases' -m ping
```

### 🔬 Практика: inventory

```bash
# 1. Создать inventory
mkdir ansible-demo && cd ansible-demo

cat > inventory.yaml <<'EOF'
all:
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
  children:
    webservers:
      hosts:
        web1:
          ansible_host: 10.0.1.10
        web2:
          ansible_host: 10.0.1.11
    databases:
      hosts:
        db1:
          ansible_host: 10.0.2.10
EOF

# 2. Показать хосты
ansible-inventory --list

# 3. Проверить соединение
ansible all -m ping -i inventory.yaml

# 4. Выполнить команду
ansible webservers -m shell -a "uname -a" -i inventory.yaml
```

**Для тестирования** можно использовать **localhost**:

```yaml
all:
  hosts:
    localhost:
      ansible_connection: local
```

**`ansible_connection: local`** — не подключаться по SSH, работать локально.

### 💡 Практика: как правильно организовать inventory

**✅ ОБЯЗАТЕЛЬНО:**

1. **YAML-формат** для inventory.
2. **Логические группы** (webservers, databases, production).
3. **Переменные в inventory** для общих параметров.

**👍 СТОИТ:**

4. **Динамический inventory** для облаков (подглава 19.10).
5. **`ansible.cfg`** для общих настроек.
6. **Разные inventory** для окружений (dev, prod).

**❌ НЕ ДЕЛАЙ:**

7. **Не хардкодь пароли** в inventory. Используй Vault.
8. **Не используй один inventory** для всех окружений.
9. **Не забывай про `ansible_python_interpreter`** на новых серверах.

### Где мы сейчас

Мы разобрали inventory. Теперь — **playbooks, tasks, modules**.

---

## 19.4 Playbooks, tasks, modules

### 🔌 Проблема: как описать, что делать на серверах

Inventory — это список серверов. Но **что** на них делать? **Playbook** отвечает на этот вопрос.

### 📊 Что такое playbook

**Playbook** — YAML-файл с описанием действий на серверах.

**Структура:**

```yaml
---
- name: Play 1                       # play
  hosts: webservers                  # на каких хостах
  become: true                       # использовать sudo
  vars:                              # переменные
    nginx_port: 80
  tasks:                             # задачи
    - name: Install nginx            # task
      apt:                           # module
        name: nginx
        state: present
      notify: Restart nginx          # notification
    
    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: true
  
  handlers:                          # handlers
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

**Ключевые понятия:**

| Понятие | Что означает |
|:---|:---|
| **Play** | Группа задач для группы хостов |
| **Task** | Одна задача (вызов модуля) |
| **Module** | Действие (apt, service, copy) |
| **Handler** | Задача, запускаемая по notify |
| **Role** | Переиспользуемый набор tasks/vars/templates |

### 🎯 Модули

**Module** — это единица работы. Ansible имеет **3000+ модулей**.

**Основные категории:**

| Категория | Модули |
|:---|:---|
| **Пакеты** | `apt`, `yum`, `dnf`, `package` |
| **Файлы** | `copy`, `template`, `file`, `lineinfile` |
| **Сервисы** | `service`, `systemd` |
| **Пользователи** | `user`, `group` |
| **Команды** | `command`, `shell`, `raw` |
| **Сеть** | `uri`, `get_url` |
| **Git** | `git` |
| **Docker** | `docker_container`, `docker_image` |
| **Kubernetes** | `k8s`, `helm` |
| **Облака** | `aws_*`, `gcp_*`, `azure_*` |

**Синтаксис:**

```yaml
- name: Task name
  module_name:
    param1: value1
    param2: value2
```

**Или короткая форма:**

```yaml
- name: Install nginx
  apt: name=nginx state=present
```

**Полная форма предпочтительнее** — читаемее.

### 🎯 Пример playbook

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: true
  vars:
    nginx_port: 80
    app_user: appuser
  
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
    
    - name: Install packages
      apt:
        name:
          - nginx
          - curl
          - git
        state: present
    
    - name: Create app user
      user:
        name: "{{ app_user }}"
        shell: /bin/bash
        create_home: yes
        state: present
    
    - name: Create app directory
      file:
        path: /opt/myapp
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0755'
    
    - name: Copy nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
      notify: Restart nginx
    
    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: yes
  
  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

**Что произошло:**

1. Обновлён кэш apt.
2. Установлены пакеты.
3. Создан пользователь.
4. Создана директория.
5. Скопирован конфиг (с notify).
6. Запущен nginx.
7. Если конфиг изменился — handler перезапустит nginx.

### 🎯 `state` в модулях

**Модули обычно имеют параметр `state`:**

| Модуль | `state` | Что означает |
|:---|:---|:---|
| `apt` | `present` | Установлено |
| `apt` | `absent` | Удалено |
| `apt` | `latest` | Последняя версия |
| `service` | `started` | Запущен |
| `service` | `stopped` | Остановлен |
| `service` | `restarted` | Перезапущен |
| `file` | `directory` | Директория |
| `file` | `absent` | Удалено |
| `file` | `touch` | Создан пустой файл |

### 🎯 Запуск playbook

```bash
# Базовый запуск
ansible-playbook -i inventory.yaml site.yml

# Check mode (dry-run)
ansible-playbook -i inventory.yaml site.yml --check

# Diff (показать изменения)
ansible-playbook -i inventory.yaml site.yml --diff

# Ограничить хосты
ansible-playbook -i inventory.yaml site.yml --limit web1

# Теги
ansible-playbook -i inventory.yaml site.yml --tags "nginx,config"

# Пропустить теги
ansible-playbook -i inventory.yaml site.yml --skip-tags "debug"

# Verbose
ansible-playbook -i inventory.yaml site.yml -v
ansible-playbook -i inventory.yaml site.yml -vvv  # очень подробно

# Только определённые tasks (по имени)
ansible-playbook -i inventory.yaml site.yml --start-at-task "Install packages"
```

### 🎯 Теги

**Tags** — группировка tasks для выборочного запуска.

```yaml
- name: Install nginx
  apt:
    name: nginx
  tags:
    - nginx
    - install

- name: Copy config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  tags:
    - nginx
    - config
```

**Запуск:**

```bash
# Только tasks с тегом nginx
ansible-playbook site.yml --tags nginx

# Все, кроме debug
ansible-playbook site.yml --skip-tags debug

# Список всех тегов
ansible-playbook site.yml --list-tags
```

### 🎯 Check mode и diff

**Check mode (`--check`):** показать, что **будет** сделано, без изменений.

```bash
ansible-playbook site.yml --check
```

**Diff (`--diff`):** показать **какие изменения** будут в файлах.

```bash
ansible-playbook site.yml --check --diff
```

**Что даёт:**

- Безопасно проверить playbook.
- Увидеть изменения до применения.
- **Критически важно для production.**

### 🔬 Практика: первый playbook

```bash
# 1. Создать inventory для localhost
cat > inventory.yaml <<'EOF'
all:
  hosts:
    localhost:
      ansible_connection: local
EOF

# 2. Создать playbook
cat > site.yml <<'EOF'
---
- name: Configure localhost
  hosts: localhost
  become: false
  
  tasks:
    - name: Show OS info
      debug:
        msg: "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
    
    - name: Create test directory
      file:
        path: /tmp/ansible-test
        state: directory
        mode: '0755'
    
    - name: Create test file
      copy:
        content: "Hello from Ansible!\n"
        dest: /tmp/ansible-test/hello.txt
        mode: '0644'
    
    - name: Show file content
      command: cat /tmp/ansible-test/hello.txt
      register: file_content
    
    - name: Display content
      debug:
        msg: "{{ file_content.stdout }}"
EOF

# 3. Запустить
ansible-playbook -i inventory.yaml site.yml

# 4. Проверить
cat /tmp/ansible-test/hello.txt

# 5. Запустить повторно — идемпотентность
ansible-playbook -i inventory.yaml site.yml
# Все tasks: "ok", не "changed" — идемпотентно!
```

### 💡 Практика: как правильно писать playbooks

**✅ ОБЯЗАТЕЛЬНО:**

1. **`name` для каждого play и task.** Читаемость.
2. **Полная форма модулей** (не `key=value`).
3. **`state` явно.**
4. **`become: true`** когда нужен root.

**👍 СТОИТ:**

5. **Tags** для группировки.
6. **Handlers** для перезапуска сервисов.
7. **`--check --diff`** перед реальным применением.

**❌ НЕ ДЕЛАЙ:**

8. **Не используй `shell`/`command`, если есть модуль.** Модули идемпотентны.
9. **Не хардкодь значения.** Используй variables.
10. **Не забывай про `become`.**

### Где мы сейчас

Мы разобрали playbooks. Теперь — **идемпотентность** — ключевая концепция.

---

## 19.5 Идемпотентность

### 🔌 Проблема: скрипт запустишь дважды — сломается

Bash-скрипт:

```bash
#!/bin/bash
echo "user ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
```

**Запустишь дважды** — добавит строку **дважды**.

**Ansible решает это** через **идемпотентность**.

### 📊 Что такое идемпотентность

**Идемпотентность** — свойство операции давать **тот же результат** при многократном выполнении.

**Пример:**

- **Идемпотентно:** `mkdir /tmp/test` (если директория есть — ничего не делает).
- **Не идемпотентно:** `echo "text" >> file.txt` (добавляет каждый раз).

**Ansible-модули** спроектированы идемпотентными.

### 🎯 Как Ansible обеспечивает идемпотентность

**1. Модули знают текущее состояние.**

Модуль `apt` проверяет, установлен ли пакет. Если да — ничего не делает.

```yaml
- name: Install nginx
  apt:
    name: nginx
    state: present
```

**Первый запуск:** `changed: true` — установил.
**Второй запуск:** `ok: true` — уже установлен.

**2. Модули приводят к состоянию.**

Модуль `service` проверяет состояние сервиса и приводит к нужному.

```yaml
- name: Start nginx
  service:
    name: nginx
    state: started
    enabled: true
```

**3. Модули `copy` и `template` сравнивают checksum.**

Если файл тот же — не копирует.

```yaml
- name: Copy config
  copy:
    src: config.conf
    dest: /etc/app/config.conf
```

### 🎯 Как определить идемпотентность

**Вывод Ansible:**

```
TASK [Install nginx] ********************************
ok: [web1]              # уже установлен — не изменилось
changed: [web2]         # установлен — изменилось
```

**Цвета:**

- **Green (ok)** — состояние уже правильное.
- **Yellow (changed)** — состояние изменено.
- **Red (failed)** — ошибка.

**Цель:** при повторном запуске **все tasks должны быть green**.

### 🎯 Не идемпотентные модули

**`shell` и `command`** — **не идемпотентны** по умолчанию. Ansible не знает, что они делают.

```yaml
# ❌ Не идемпотентно
- name: Append to file
  shell: echo "text" >> /etc/file
```

**Решение:** использовать `creates` или `changed_when`.

**`creates`:**

```yaml
- name: Extract archive
  shell: tar -xzf /tmp/archive.tar.gz -C /opt/
  args:
    creates: /opt/archive  # если существует — не запускать
```

**`changed_when`:**

```yaml
- name: Check status
  command: /usr/bin/check-status
  register: status
  changed_when: false     # никогда не changed (read-only)
```

### 🎯 `changed_when` и `failed_when`

**`changed_when`** — когда считать task изменённым:

```yaml
- name: Check if file exists
  command: test -f /etc/config
  register: result
  changed_when: false     # read-only команда
  failed_when: result.rc not in [0, 1]  # 0 или 1 — OK, другое — fail
```

**`failed_when`** — когда считать task упавшим:

```yaml
- name: Check service
  command: systemctl is-active nginx
  register: result
  failed_when: false      # никогда не fail
  changed_when: false
```

### 🎯 `check_mode`

**Некоторые модули поддерживают `check_mode`** — проверить без изменений.

```bash
ansible-playbook site.yml --check
```

**Что произойдёт:** Ansible пройдёт все tasks, но **не будет** вносить изменения.

**Если модуль не поддерживает check_mode:**

```yaml
- name: Run custom script
  shell: /opt/script.sh
  check_mode: no          # игнорировать check_mode
```

### 🎯 Пример идемпотентного playbook

```yaml
---
- name: Configure web server
  hosts: webservers
  become: true
  
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600    # не обновлять, если кэш < 1 часа
    
    - name: Install nginx
      apt:
        name: nginx
        state: present
    
    - name: Create user
      user:
        name: appuser
        state: present
        shell: /bin/bash
    
    - name: Copy config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        mode: '0644'
      notify: Restart nginx
    
    - name: Ensure nginx is running
      service:
        name: nginx
        state: started
        enabled: yes
  
  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

**Идемпотентность:**

- `apt update_cache` — обновляет только если кэш старый.
- `apt install` — устанавливает только если не установлен.
- `user` — создаёт только если не существует.
- `template` — копирует только если содержимое изменилось.
- `service` — запускает только если не запущен.

### 🔬 Практика: проверка идемпотентности

```bash
# 1. Playbook
cat > site.yml <<'EOF'
---
- name: Test idempotency
  hosts: localhost
  connection: local
  
  tasks:
    - name: Create file
      copy:
        content: "Hello\n"
        dest: /tmp/ansible-idempotent.txt
        mode: '0644'
    
    - name: Create user (idempotent)
      user:
        name: testuser_ansible
        state: present
      become: true
EOF

# 2. Первый запуск — changed
ansible-playbook -i inventory.yaml site.yml
# TASK [Create file] ****
# changed: [localhost]
# TASK [Create user (idempotent)] ****
# changed: [localhost]

# 3. Второй запуск — ok (не changed!)
ansible-playbook -i inventory.yaml site.yml
# TASK [Create file] ****
# ok: [localhost]
# TASK [Create user (idempotent)] ****
# ok: [localhost]

# 4. Если изменить content — снова changed
sed -i 's/Hello/Hello World/' site.yml
ansible-playbook -i inventory.yaml site.yml
# TASK [Create file] ****
# changed: [localhost]
```

### 💡 Практика: как правильно обеспечивать идемпотентность

**✅ ОБЯЗАТЕЛЬНО:**

1. **Использовать модули вместо `shell`/`command`.**
2. **Для `shell` использовать `creates` или `changed_when`.**
3. **Проверять идемпотентность через повторный запуск.**

**👍 СТОИТ:**

4. **`--check --diff` перед применением.**
5. **`cache_valid_time` для apt/yum.**
6. **Test idempotency в CI.**

**❌ НЕ ДЕЛАЙ:**

7. **Не используй `shell` для всего.** Модули идемпотентны.
8. **Не игнорируй `changed` status.** Должно быть `ok` при повторном запуске.

### Где мы сейчас

Мы разобрали идемпотентность. Теперь — **variables, facts, templates**.

---

## 19.6 Variables, facts, templates

### 🔌 Проблема: как параметризовать playbooks

Один playbook для dev и prod. Отличаются:

- Порт nginx.
- Размер worker_processes.
- Путь к БД.
- Пользователь.

**Решение:** variables.

### 📊 Типы variables

**1. Playbook variables:**

```yaml
- hosts: webservers
  vars:
    nginx_port: 80
    nginx_worker_processes: 4
  tasks:
    - name: Show port
      debug:
        msg: "Port: {{ nginx_port }}"
```

**2. Inventory variables:**

```yaml
# inventory.yaml
all:
  children:
    webservers:
      vars:
        nginx_port: 80
      hosts:
        web1:
          nginx_port: 8080    # override
```

**3. Group vars (`group_vars/`):**

```
ansible/
├── inventory.yaml
├── group_vars/
│   ├── all.yml            # для всех
│   ├── webservers.yml     # для группы webservers
│   └── production.yml     # для группы production
└── host_vars/
    ├── web1.yml           # для хоста web1
    └── db1.yml
```

**group_vars/webservers.yml:**

```yaml
nginx_port: 80
nginx_worker_processes: 4
nginx_worker_connections: 1024
```

**4. Facts:**

Ansible автоматически собирает **факты** о серверах:

```yaml
- name: Show facts
  debug:
    msg:
      OS: "{{ ansible_distribution }}"
      Version: "{{ ansible_distribution_version }}"
      IP: "{{ ansible_default_ipv4.address }}"
      Memory: "{{ ansible_memtotal_mb }} MB"
```

**5. Extra vars (`-e`):**

```bash
ansible-playbook site.yml -e "nginx_port=8080"
```

**Высший приоритет.**

### 🎯 Приоритет variables

**От низшего к высшему:**

1. `role defaults` (`roles/x/defaults/main.yml`)
2. Inventory `group_vars/all`
3. Inventory `group_vars/*`
4. Inventory `host_vars/*`
5. Playbook `vars`
6. Playbook `vars_files`
7. `role vars` (`roles/x/vars/main.yml`)
8. `set_fact` / `register`
9. Extra vars (`-e`)

**Правило:** более специфичные переопределяют более общие.

### 🎯 Facts

**Facts** — информация о сервере, собираемая Ansible.

**Просмотр фактов:**

```bash
ansible web1 -m setup
```

**Основные facts:**

| Fact | Что содержит |
|:---|:---|
| `ansible_hostname` | Hostname |
| `ansible_distribution` | Ubuntu, CentOS, ... |
| `ansible_distribution_version` | 22.04, 9, ... |
| `ansible_os_family` | Debian, RedHat, ... |
| `ansible_architecture` | x86_64, arm64, ... |
| `ansible_default_ipv4.address` | IP |
| `ansible_memtotal_mb` | RAM (MB) |
| `ansible_processor_vcpus` | Число CPU |
| `ansible_devices` | Диски |
| `ansible_interfaces` | Сетевые интерфейсы |

**Использование:**

```yaml
- name: Install package based on OS
  apt:
    name: nginx
  when: ansible_os_family == "Debian"

- name: Install on RHEL
  yum:
    name: nginx
  when: ansible_os_family == "RedHat"
```

**Отключить сбор фактов (для ускорения):**

```yaml
- hosts: webservers
  gather_facts: false
```

### 🎯 `when` — условия

**`when`** — выполнять task только если условие истинно.

```yaml
- name: Install on Debian
  apt:
    name: nginx
  when: ansible_os_family == "Debian"

- name: Create directory
  file:
    path: /opt/app
    state: directory
  when: inventory_hostname in groups['webservers']

- name: Multiple conditions
  apt:
    name: nginx
  when:
    - ansible_os_family == "Debian"
    - ansible_distribution_version is version('22.04', '>=')
```

**Операторы:**

| Оператор | Что означает |
|:---|:---|
| `==`, `!=` | Равно, не равно |
| `<`, `>`, `<=`, `>=` | Сравнение |
| `and`, `or`, `not` | Логические |
| `in`, `not in` | В списке |
| `is defined`, `is not defined` | Определено |
| `is version('1.0', '>=')` | Сравнение версий |

### 🎯 Templates (Jinja2)

**Templates** — файлы с переменными Jinja2.

**`templates/nginx.conf.j2`:**

```jinja
user www-data;
worker_processes {{ nginx_worker_processes }};
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections }};
}

http {
    {% for server in nginx_servers %}
    server {
        listen {{ server.port }};
        server_name {{ server.name }};
        
        location / {
            proxy_pass http://{{ server.backend }};
        }
    }
    {% endfor %}
}
```

**Использование:**

```yaml
- name: Configure nginx
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: '0644'
  vars:
    nginx_servers:
      - name: api.example.com
        port: 80
        backend: localhost:8080
      - name: www.example.com
        port: 80
        backend: localhost:8081
```

**Jinja2 синтаксис:**

| Синтаксис | Что означает |
|:---|:---|
| `{{ variable }}` | Вывести значение |
| `{% if condition %}...{% endif %}` | Условие |
| `{% for item in list %}...{% endfor %}` | Цикл |
| `{{ variable \| default('x') }}` | Default |
| `{{ variable \| upper }}` | Фильтры |
| `{# comment #}` | Комментарий |

**Фильтры:**

```jinja
{{ name | upper }}                    # UPPER
{{ name | lower }}                    # lower
{{ list | join(',') }}                # a,b,c
{{ value | default('default') }}      # default если undefined
{{ number | int }}                    # int
{{ str | bool }}                      # bool
{{ path | basename }}                 # filename
{{ path | dirname }}                  # directory
{{ data | to_json }}                  # JSON
{{ data | to_yaml }}                  # YAML
```

### 🎯 `register` — сохранение результата

```yaml
- name: Check service status
  command: systemctl is-active nginx
  register: nginx_status
  ignore_errors: yes

- name: Show status
  debug:
    msg: "Nginx is {{ nginx_status.stdout }}"
```

**Использование:**

```yaml
- name: Get file content
  slurp:
    src: /etc/config
  register: config

- name: Decode content
  set_fact:
    config_content: "{{ config.content | b64decode }}"

- name: Show
  debug:
    var: config_content
```

### 🎯 `set_fact` — создание переменной

```yaml
- name: Set fact
  set_fact:
    my_variable: "computed value"

- name: Use fact
  debug:
    msg: "{{ my_variable }}"
```

**`set_fact` создаёт переменную**, доступную в последующих tasks.

### 🔬 Практика: variables и templates

```bash
# 1. Создать структуру
mkdir -p group_vars templates

# 2. group_vars/all.yml
cat > group_vars/all.yml <<'EOF'
app_name: myapp
app_port: 8080
nginx_worker_processes: 4
nginx_worker_connections: 1024
EOF

# 3. templates/index.html.j2
cat > templates/index.html.j2 <<'EOF'
<!DOCTYPE html>
<html>
<head><title>{{ app_name }}</title></head>
<body>
  <h1>Welcome to {{ app_name }}</h1>
  <p>Server: {{ ansible_hostname }}</p>
  <p>OS: {{ ansible_distribution }} {{ ansible_distribution_version }}</p>
  <p>Port: {{ app_port }}</p>
</body>
</html>
EOF

# 4. site.yml
cat > site.yml <<'EOF'
---
- name: Deploy web app
  hosts: localhost
  connection: local
  
  tasks:
    - name: Create directory
      file:
        path: /tmp/ansible-web
        state: directory
    
    - name: Deploy index.html
      template:
        src: templates/index.html.j2
        dest: /tmp/ansible-web/index.html
        mode: '0644'
    
    - name: Show file
      command: cat /tmp/ansible-web/index.html
      register: content
      changed_when: false
    
    - debug:
        var: content.stdout_lines
EOF

# 5. Запустить
ansible-playbook -i inventory.yaml site.yml

# 6. Проверить
cat /tmp/ansible-web/index.html
```

### 💡 Практика: как правильно использовать variables

**✅ ОБЯЗАТЕЛЬНО:**

1. **`group_vars/` и `host_vars/`** для организации.
2. **Defaults в roles** для значений по умолчанию.
3. **Facts** для динамических значений.

**👍 СТОИТ:**

4. **Templates (Jinja2)** для конфигов.
5. **`when`** для условий.
6. **`register`** для сохранения результатов.

**❌ НЕ ДЕЛАЙ:**

7. **Не хардкодь значения.** Используй variables.
8. **Не забывай про `default`** для необязательных переменных.
9. **Не используй `set_fact` без необходимости.**

### Где мы сейчас

Мы разобрали variables. Теперь — **handlers и notify**.

---

## 19.7 Handlers и notify

### 🔌 Проблема: перезапускать сервис только при изменении конфига

Ты обновляешь конфиг nginx. Нужно **перезапустить** nginx. Но **только если конфиг изменился**. Если запускать всегда — простой.

**Решение:** handlers.

### 📊 Что такое handler

**Handler** — специальная task, которая выполняется **только при notify** и **в конце play**.

**Структура:**

```yaml
- hosts: webservers
  become: true
  
  tasks:
    - name: Copy nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx        # ← notify handler
  
  handlers:
    - name: Restart nginx           # ← handler
      service:
        name: nginx
        state: restarted
```

**Что произойдёт:**

1. Если конфиг **изменился** — `template` возвращает `changed: true`.
2. Ansible вызывает handler `Restart nginx`.
3. Handler выполнится **в конце play**.
4. Если конфиг **не изменился** — handler не вызывается.

### 🎯 Почему в конце play

**Handlers выполняются в конце**, чтобы:

- Избежать многократных перезапусков.
- Если несколько tasks notify один handler — перезапуск будет **один раз**.
- Если handler упадёт — play упадёт.

**Пример:**

```yaml
tasks:
  - name: Copy config 1
    template:
      src: config1.j2
      dest: /etc/app/config1
    notify: Restart app          # notify 1
  
  - name: Copy config 2
    template:
      src: config2.j2
      dest: /etc/app/config2
    notify: Restart app          # notify 2 (тот же handler)
  
  - name: Copy config 3
    template:
      src: config3.j2
      dest: /etc/app/config3
    notify: Restart app          # notify 3

handlers:
  - name: Restart app
    service:
      name: app
      state: restarted
```

**Что произойдёт:**

- Если изменились все три конфига — handler выполнится **один раз** в конце.
- Не три раза.

### 🎯 Несколько handlers

```yaml
handlers:
  - name: Restart nginx
    service:
      name: nginx
      state: restarted
  
  - name: Reload nginx
    service:
      name: nginx
      state: reloaded
  
  - name: Restart app
    systemd:
      name: app
      state: restarted
```

**Notify:**

```yaml
tasks:
  - name: Update nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Reload nginx          # reload вместо restart
  
  - name: Update app config
    template:
      src: app.conf.j2
      dest: /etc/app/app.conf
    notify: Restart app
```

### 🎯 Flush handlers

**По умолчанию handlers в конце play.** Можно вызвать **раньше**:

```yaml
tasks:
  - name: Copy config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Restart nginx
  
  - name: Flush handlers
    meta: flush_handlers       # ← выполнить handlers сейчас
  
  - name: Check nginx
    uri:
      url: http://localhost
      status_code: 200
```

**Когда использовать:** если следующая task зависит от перезапуска.

### 🎯 Handlers в roles

**В role:**

```
roles/nginx/
├── tasks/
│   └── main.yml
└── handlers/
    └── main.yml
```

**handlers/main.yml:**

```yaml
- name: Restart nginx
  service:
    name: nginx
    state: restarted
```

**tasks/main.yml:**

```yaml
- name: Configure nginx
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Restart nginx
```

**Handlers из roles** автоматически доступны.

### 🎯 Listen (Ansible 2.2+)

**`listen`** — handler, который слушает несколько тем:

```yaml
handlers:
  - name: Restart nginx
    service:
      name: nginx
      state: restarted
    listen: "restart web services"
  
  - name: Restart php
    service:
      name: php-fpm
      state: restarted
    listen: "restart web services"
```

**Notify:**

```yaml
tasks:
  - name: Update config
    template:
      src: config.j2
      dest: /etc/config
    notify: "restart web services"    # вызовет оба handler
```

**Что даёт:** один notify вызывает несколько handlers.

### 🔬 Практика: handlers

```bash
# 1. Playbook с handler
cat > site.yml <<'EOF'
---
- name: Configure web server
  hosts: localhost
  connection: local
  become: true
  
  tasks:
    - name: Create config directory
      file:
        path: /tmp/ansible-config
        state: directory
    
    - name: Copy config
      copy:
        content: |
          port: 8080
          log_level: info
        dest: /tmp/ansible-config/app.conf
      notify: Config changed
    
    - name: This runs after
      debug:
        msg: "Regular task"
  
  handlers:
    - name: Config changed
      debug:
        msg: "!!! Handler executed: config changed !!!"
EOF

# 2. Первый запуск — handler выполнится
ansible-playbook -i inventory.yaml site.yml
# TASK [Copy config] ***
# changed: [localhost]
# TASK [This runs after] ***
# ok: [localhost]
# RUNNING HANDLER [Config changed] ***
# "!!! Handler executed: config changed !!!"

# 3. Второй запуск — config не изменился → handler НЕ выполнится
ansible-playbook -i inventory.yaml site.yml
# TASK [Copy config] ***
# ok: [localhost]  ← не changed
# TASK [This runs after] ***
# ok: [localhost]
# ← handler не выполнился

# 4. Изменить config
sed -i 's/8080/9090/' site.yml
ansible-playbook -i inventory.yaml site.yml
# TASK [Copy config] ***
# changed: [localhost]
# RUNNING HANDLER [Config changed] ***
```

### 💡 Практика: как правильно использовать handlers

**✅ ОБЯЗАТЕЛЬНО:**

1. **Handlers для перезапуска сервисов.**
2. **Notify только при изменении.**
3. **Handlers в конце play** (по умолчанию).

**👍 СТОИТ:**

4. **`reload` вместо `restart`** где возможно (без простоя).
5. **`listen`** для группировки.
6. **`meta: flush_handlers`** если нужно раньше.

**❌ НЕ ДЕЛАЙ:**

7. **Не перезапускай сервисы без handlers.** Простой.
8. **Не забывай про `notify`.** Без него handler не вызовется.
9. **Не используй handlers для обычных tasks.**

### Где мы сейчас

Мы разобрали handlers. Теперь — **roles** — переиспользование кода.

---

## 19.8 Roles: переиспользование кода

### 🔌 Проблема: дублирование кода

Ты настраиваешь nginx для проекта A. Потом для проекта B. Копируешь tasks. Меняешь параметры. Через год — 10 копий.

**Решение:** roles.

### 📊 Что такое role

**Role** — переиспользуемый набор tasks, handlers, variables, templates, files.

**Структура:**

```
roles/nginx/
├── tasks/
│   └── main.yml            # основные задачи
├── handlers/
│   └── main.yml            # handlers
├── defaults/
│   └── main.yml            # значения по умолчанию (низкий приоритет)
├── vars/
│   └── main.yml            # переменные (высокий приоритет)
├── files/
│   └── index.html          # статические файлы
├── templates/
│   └── nginx.conf.j2       # шаблоны
├── meta/
│   └── main.yml            # зависимости
└── README.md               # документация
```

**Каждая директория опциональна.** Можно использовать только нужные.

### 🎯 Создание role

```bash
ansible-galaxy init nginx
```

**Что создаст:**

```
nginx/
├── defaults/
│   └── main.yml
├── files/
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── README.md
├── tasks/
│   └── main.yml
├── templates/
├── tests/
│   ├── inventory
│   └── test.yml
└── vars/
    └── main.yml
```

### 🎯 Пример role

**roles/nginx/defaults/main.yml:**

```yaml
nginx_port: 80
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_server_name: localhost
nginx_root: /var/www/html
```

**roles/nginx/tasks/main.yml:**

```yaml
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes
  when: ansible_os_family == "Debian"

- name: Install nginx on RHEL
  yum:
    name: nginx
    state: present
  when: ansible_os_family == "RedHat"

- name: Create web root
  file:
    path: "{{ nginx_root }}"
    state: directory
    mode: '0755'

- name: Configure nginx
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: '0644'
  notify: Restart nginx

- name: Copy index.html
  copy:
    src: index.html
    dest: "{{ nginx_root }}/index.html"
    mode: '0644'

- name: Start nginx
  service:
    name: nginx
    state: started
    enabled: yes
```

**roles/nginx/handlers/main.yml:**

```yaml
- name: Restart nginx
  service:
    name: nginx
    state: restarted
```

**roles/nginx/templates/nginx.conf.j2:**

```jinja
user www-data;
worker_processes {{ nginx_worker_processes }};
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections }};
}

http {
    server {
        listen {{ nginx_port }};
        server_name {{ nginx_server_name }};
        
        root {{ nginx_root }};
        index index.html;
        
        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

### 🎯 Использование role

**В playbook:**

```yaml
---
- hosts: webservers
  become: true
  
  roles:
    - nginx
```

**С параметрами:**

```yaml
- hosts: webservers
  become: true
  
  roles:
    - role: nginx
      vars:
        nginx_port: 8080
        nginx_server_name: example.com
```

**Или через `vars` в play:**

```yaml
- hosts: webservers
  become: true
  vars:
    nginx_port: 8080
    nginx_server_name: example.com
  roles:
    - nginx
```

### 🎯 `defaults` vs `vars`

**defaults** — значения по умолчанию. Легко переопределить.

**vars** — переменные role. Высокий приоритет. Переопределить сложнее.

**Правило:** используй **defaults** для параметров, которые можно переопределить. **vars** — только для внутренних констант.

### 🎯 `meta/main.yml` — зависимости

```yaml
# roles/myapp/meta/main.yml
dependencies:
  - role: nginx
    vars:
      nginx_port: 8080
  - role: postgresql
    vars:
      postgres_version: 16
  - role: common
```

**Что произойдёт:** при использовании role `myapp`, сначала выполнятся `nginx`, `postgresql`, `common`.

### 🎯 Roles в Ansible Galaxy

**Ansible Galaxy** — реестр ролей.

**Поиск:**

```bash
ansible-galaxy search nginx
```

**Установка:**

```bash
# Из Galaxy
ansible-galaxy install geerlingguy.nginx

# Конкретная версия
ansible-galaxy install geerlingguy.nginx,3.1.4

# Из Git
ansible-galaxy install git+https://github.com/user/role.git

# Список установленных
ansible-galaxy list
```

**requirements.yml:**

```yaml
# requirements.yml
roles:
  - name: geerlingguy.nginx
    version: 3.1.4
  - name: geerlingguy.postgresql
    version: 3.4.0
  - src: https://github.com/user/role.git
    name: custom-role
    version: v1.0.0
```

**Установка:**

```bash
ansible-galaxy install -r requirements.yml
```

### 🎯 Популярные роли

| Role | Что делает |
|:---|:---|
| `geerlingguy.nginx` | Nginx |
| `geerlingguy.postgresql` | PostgreSQL |
| `geerlingguy.mysql` | MySQL |
| `geerlingguy.docker` | Docker |
| `geerlingguy.kubernetes` | Kubernetes |
| `geerlingguy.nodejs` | Node.js |
| `robertdebock.common` | Common setup |
| `ANXS.postgresql` | PostgreSQL |

**Jeff Geerling** (geerlingguy) — самый популярный автор ролей.

### 🎯 Включение tasks из role

**`include_role`** — включить role в tasks:

```yaml
tasks:
  - name: Configure nginx
    include_role:
      name: nginx
    vars:
      nginx_port: 8080
```

**`import_role`** — статический импорт:

```yaml
tasks:
  - import_role:
      name: nginx
```

**Разница:**

- `include_role` — динамический (обрабатывается во время выполнения).
- `import_role` — статический (обрабатывается при парсинге).

### 🔬 Практика: создание role

```bash
# 1. Создать role
ansible-galaxy init roles/nginx

# 2. defaults
cat > roles/nginx/defaults/main.yml <<'EOF'
nginx_port: 80
nginx_server_name: localhost
EOF

# 3. tasks
cat > roles/nginx/tasks/main.yml <<'EOF'
- name: Create config
  template:
    src: nginx.conf.j2
    dest: /tmp/ansible-nginx-{{ nginx_port }}.conf
  notify: Restart nginx

- name: Show config
  debug:
    msg: "Configured nginx on port {{ nginx_port }}"
EOF

# 4. template
cat > roles/nginx/templates/nginx.conf.j2 <<'EOF'
server {
    listen {{ nginx_port }};
    server_name {{ nginx_server_name }};
}
EOF

# 5. handlers
cat > roles/nginx/handlers/main.yml <<'EOF'
- name: Restart nginx
  debug:
    msg: "Nginx restarted"
EOF

# 6. Playbook
cat > site.yml <<'EOF'
---
- hosts: localhost
  connection: local
  
  roles:
    - role: nginx
      vars:
        nginx_port: 8080
        nginx_server_name: example.com
EOF

# 7. Запустить
ansible-playbook -i inventory.yaml site.yml
```

### 💡 Практика: как правильно писать roles

**✅ ОБЯЗАТЕЛЬНО:**

1. **Defaults для параметров.**
2. **Tasks разбиты на файлы** (если их много).
3. **README с примерами.**
4. **Tests в `tests/`.**

**👍 СТОИТ:**

5. **Ansible Galaxy** для общих ролей.
6. **`meta/main.yml`** для зависимостей.
7. **Idempotency tests.**

**❌ НЕ ДЕЛАЙ:**

8. **Не дублируй код.** Role для переиспользования.
9. **Не хардкодь значения** в tasks. Только defaults.
10. **Не забывай про `handlers`.**

### Где мы сейчас

Мы разобрали roles. Теперь — **Ansible Vault** для секретов.

---

## 19.9 Ansible Vault: секреты

### 🔌 Проблема: секреты в Git

Твой playbook содержит пароль к БД:

```yaml
- name: Create user
  mysql_user:
    name: app
    password: SuperSecret123    # ← в Git!
```

**Проблема:** пароль в Git → компрометация.

**Решение:** Ansible Vault.

### 📊 Что такое Ansible Vault

**Ansible Vault** — встроенный механизм шифрования секретов.

**Как работает:**

1. Шифруешь файл с секретами.
2. Коммитишь зашифрованный файл в Git.
3. Ansible расшифровывает при запуске.

### 🎯 Создание зашифрованного файла

```bash
# Создать новый зашифрованный файл
ansible-vault create secrets.yml
# Попросит пароль
# Откроет редактор (vim)
```

**Внутри:**

```yaml
db_password: SuperSecret123
api_key: sk_live_abc123def456
```

**Сохранить и выйти.** Файл зашифрован.

### 🎯 Шифрование существующего файла

```bash
ansible-vault encrypt secrets.yml
```

### 🎯 Просмотр зашифрованного файла

```bash
ansible-vault view secrets.yml
# Попросит пароль, покажет расшифрованное
```

### 🎯 Редактирование

```bash
ansible-vault edit secrets.yml
# Откроет редактор с расшифрованным содержимым
```

### 🎯 Расшифровка

```bash
# Расшифровать (файл останется расшифрованным)
ansible-vault decrypt secrets.yml

# ⚠️ Не коммить расшифрованный файл!
```

### 🎯 Использование в playbook

**`vars_files`:**

```yaml
- hosts: databases
  vars_files:
    - secrets.yml
    - vars.yml
  tasks:
    - name: Create user
      mysql_user:
        name: app
        password: "{{ db_password }}"
```

**Запуск:**

```bash
ansible-playbook site.yml --ask-vault-pass
# Попросит пароль
```

**С файлом пароля:**

```bash
# Создать файл с паролем
echo "MyVaultPassword" > .vault_pass
chmod 600 .vault_pass

# Добавить в .gitignore
echo ".vault_pass" >> .gitignore

# Запуск
ansible-playbook site.yml --vault-password-file .vault_pass
```

**В `ansible.cfg`:**

```ini
[defaults]
vault_password_file = .vault_pass
```

### 🎯 Шифрование отдельной строки

**`ansible-vault encrypt_string`:**

```bash
ansible-vault encrypt_string 'SuperSecret123' --name 'db_password'
```

**Вывод:**

```yaml
db_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          66386439653236336...
```

**Использование в playbook:**

```yaml
vars:
  db_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          66386439653236336...
```

**Плюс:** не нужно отдельный файл.

### 🎯 Множественные vault-id

**Разные пароли для разных окружений:**

```bash
# Создать с vault-id
ansible-vault create --vault-id dev@prompt secrets-dev.yml
ansible-vault create --vault-id prod@prompt secrets-prod.yml
```

**Запуск:**

```bash
ansible-playbook site.yml \
  --vault-id dev@~/.vault-dev-pass \
  --vault-id prod@~/.vault-prod-pass
```

### 🎯 Rekey (смена пароля)

```bash
ansible-vault rekey secrets.yml
# Попросит старый и новый пароль
```

### 🎯 Работа с Vault в CI/CD

**GitLab CI:**

```yaml
deploy:
  stage: deploy
  script:
    - echo "$ANSIBLE_VAULT_PASSWORD" > .vault_pass
    - ansible-playbook site.yml --vault-password-file .vault_pass
    - rm .vault_pass
```

**Переменная `ANSIBLE_VAULT_PASSWORD`** — в CI/CD Variables (Masked, Protected).

**GitHub Actions:**

```yaml
- name: Run Ansible
  env:
    ANSIBLE_VAULT_PASSWORD: ${{ secrets.ANSIBLE_VAULT_PASSWORD }}
  run: |
    echo "$ANSIBLE_VAULT_PASSWORD" > .vault_pass
    ansible-playbook site.yml --vault-password-file .vault_pass
    rm .vault_pass
```

### 🎯 Альтернативы Vault

**1. HashiCorp Vault:**

```yaml
- hosts: databases
  tasks:
    - name: Get password from Vault
      set_fact:
        db_password: "{{ lookup('hashi_vault', 'secret=secret/data/db:password url=https://vault.example.com token=' + vault_token) }}"
```

**2. AWS Secrets Manager:**

```yaml
- name: Get secret
  set_fact:
    db_password: "{{ lookup('amazon.aws.secretsmanager_secret', 'prod/db/password') }}"
```

**3. Environment variables:**

```yaml
- name: Create user
  mysql_user:
    password: "{{ lookup('env', 'DB_PASSWORD') }}"
```

**Преимущества внешних менеджеров:**

- Централизованное управление.
- Аудит.
- Ротация.
- Динамические секреты.

### 🔬 Практика: Ansible Vault

```bash
# 1. Создать vault password file
echo "MyVaultPass123" > .vault_pass
chmod 600 .vault_pass

# 2. Добавить в .gitignore
echo ".vault_pass" > .gitignore

# 3. Создать зашифрованный файл
ansible-vault create secrets.yml
# Внутри написать:
# db_password: SuperSecret123
# api_key: sk_live_abc123

# 4. Посмотреть
ansible-vault view secrets.yml

# 5. Использовать в playbook
cat > site.yml <<'EOF'
---
- hosts: localhost
  connection: local
  vars_files:
    - secrets.yml
  
  tasks:
    - name: Show secret (маскированный)
      debug:
        msg: "Password length: {{ db_password | length }}"
      
    - name: Use secret
      debug:
        msg: "API key starts with: {{ api_key[:7] }}..."
EOF

# 6. Запустить
ansible-playbook -i inventory.yaml site.yml --vault-password-file .vault_pass

# 7. Encrypt string
ansible-vault encrypt_string 'SuperSecret123' --name 'db_password' --vault-password-file .vault_pass
```

### 💡 Практика: как правильно работать с Vault

**✅ ОБЯЗАТЕЛЬНО:**

1. **Vault для всех секретов в Git.**
2. **Пароль Vault — не в Git.**
3. **Пароль Vault — в CI/CD Variables (Masked).**

**👍 СТОИТ:**

4. **Разные Vault password** для dev/prod.
5. **Hashicorp Vault** или AWS Secrets Manager для production.
6. **Ротация паролей** регулярно.

**❌ НЕ ДЕЛАЙ:**

7. **Не коммить `.vault_pass`.**
8. **Не коммить расшифрованные secrets.**
9. **Не используй один Vault password для всех.**
10. **Не хардкодь секреты** в playbook.

### Где мы сейчас

Мы разобрали Vault. Теперь — **динамический inventory**.

---

## 19.10 Динамический inventory

### 🔌 Проблема: серверы меняются

В облаке серверы создаются и удаляются постоянно. Статический inventory устаревает. Нужен **динамический**.

**Решение:** динамический inventory.

### 📊 Что такое динамический inventory

**Динамический inventory** — скрипт или плагин, который **запрашивает** список серверов из облака.

**Ansible имеет встроенные плагины для:**

- AWS EC2
- GCP Compute
- Azure
- DigitalOcean
- Kubernetes
- Terraform state
- И многих других

### 🎯 AWS EC2 Inventory

**`aws_ec2.yml`:**

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - us-west-2
  - us-east-1

filters:
  instance-state-name: running
  tag:Environment:
    - production
    - staging

keyed_groups:
  - key: tags.Environment
    prefix: env
    separator: "_"
  - key: tags.Role
    prefix: role
  - key: instance_type
    prefix: type

hostnames:
  - tag:Name
  - private-ip-address

compose:
  ansible_host: private_ip_address
  ansible_user: "'ubuntu'"
```

**Использование:**

```bash
# Проверить
ansible-inventory -i aws_ec2.yml --list

# Использовать
ansible-playbook -i aws_ec2.yml site.yml
```

**Установка коллекции:**

```bash
ansible-galaxy collection install amazon.aws
pip install boto3 botocore
```

**Что произойдёт:**

1. Ansible запросит EC2-инстансы у AWS.
2. Создаст группы по tags (`env_production`, `role_web`).
3. Использует эти группы в playbook.

### 🎯 GCP Inventory

**`gcp_compute.yml`:**

```yaml
plugin: google.cloud.gcp_compute
projects:
  - my-project
zones:
  - us-west1-a
  - us-west1-b

filters:
  - status = RUNNING

keyed_groups:
  - key: labels.environment
    prefix: env
  - key: labels.role
    prefix: role

hostnames:
  - name
  - public_ip

compose:
  ansible_host: networkInterfaces[0].accessConfigs[0].natIP
```

**Установка:**

```bash
ansible-galaxy collection install google.cloud
pip install requests google-auth
```

### 🎯 Kubernetes Inventory

**`k8s.yml`:**

```yaml
plugin: kubernetes.core.k8s
connections:
  - namespaces:
      - default
      - production
  - kubeconfig: ~/.kube/config

keyed_groups:
  - key: metadata.labels.app
    prefix: app
```

**Установка:**

```bash
ansible-galaxy collection install kubernetes.core
pip install kubernetes
```

### 🎯 Terraform State Inventory

**Читает IP-адреса из Terraform state.**

**`terraform.yml`:**

```yaml
plugin: community.general.terraform_state
backend: s3
config:
  bucket: my-terraform-state
  key: prod/terraform.tfstate
  region: us-west-2
```

**Установка:**

```bash
ansible-galaxy collection install community.general
```

**Что даёт:** использует output'ы Terraform для inventory.

### 🎯 Кастомный динамический inventory

**Скрипт, который возвращает JSON:**

```python
#!/usr/bin/env python3
# inventory.py
import json
import subprocess

def get_hosts():
    # Пример: получить хосты из API
    hosts = [
        {"name": "web1", "ip": "10.0.1.10", "env": "prod", "role": "web"},
        {"name": "web2", "ip": "10.0.1.11", "env": "prod", "role": "web"},
        {"name": "db1", "ip": "10.0.2.10", "env": "prod", "role": "db"},
    ]
    return hosts

hosts = get_hosts()

inventory = {
    "_meta": {
        "hostvars": {}
    },
    "all": {
        "hosts": [h["name"] for h in hosts]
    },
    "webservers": {
        "hosts": [h["name"] for h in hosts if h["role"] == "web"]
    },
    "databases": {
        "hosts": [h["name"] for h in hosts if h["role"] == "db"]
    }
}

for h in hosts:
    inventory["_meta"]["hostvars"][h["name"]] = {
        "ansible_host": h["ip"],
        "environment": h["env"],
    }

print(json.dumps(inventory, indent=2))
```

**Использование:**

```bash
chmod +x inventory.py
ansible-playbook -i inventory.py site.yml
```

**Что даёт:** полный контроль над inventory.

### 🎯 Кэширование

**Динамический inventory может быть медленным** (запросы к API). Решение — кэширование.

**`ansible.cfg`:**

```ini
[inventory]
cache = True
cache_plugin = jsonfile
cache_timeout = 3600
cache_connection = /tmp/ansible_inventory_cache
```

**Что даёт:** inventory кэшируется на час.

### 🔬 Практика: динамический inventory для AWS

```bash
# 1. Установить коллекции
ansible-galaxy collection install amazon.aws
pip install boto3 botocore

# 2. Настроить AWS credentials
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_REGION=us-west-2

# 3. Создать aws_ec2.yml
cat > aws_ec2.yml <<'EOF'
plugin: amazon.aws.aws_ec2
regions:
  - us-west-2
filters:
  instance-state-name: running
keyed_groups:
  - key: tags.Environment
    prefix: env
  - key: tags.Role
    prefix: role
hostnames:
  - tag:Name
  - private-ip-address
compose:
  ansible_host: private_ip_address
EOF

# 4. Проверить
ansible-inventory -i aws_ec2.yml --list

# 5. Использовать
ansible-playbook -i aws_ec2.yml site.yml
```

### 💡 Практика: как правильно работать с динамическим inventory

**✅ ОБЯЗАТЕЛЬНО:**

1. **Динамический inventory для облаков.**
2. **Keyed_groups** для логических групп.
3. **Кэширование** для ускорения.

**👍 СТОИТ:**

4. **Terraform state inventory** для связи с Terraform.
5. **Кастомный скрипт** для специфичных API.

**❌ НЕ ДЕЛАЙ:**

6. **Не хардкодь IP-адреса.** Динамический inventory.
7. **Не забывай про credentials** для облаков.

### Где мы сейчас

Мы разобрали динамический inventory. Теперь — **Ansible в CI/CD**.

---

## 19.11 Ansible в CI/CD

### 🔌 Проблема: как применять playbooks автоматически

Ручной `ansible-playbook` — не масштабируется. Нужна автоматизация.

### 📊 Workflow

```
1. Разработчик изменяет playbook.
2. Push в Git.
3. CI запускает ansible-lint.
4. Merge в main.
5. CI применяет playbook на серверах.
6. Уведомление в Slack.
```

### 🎯 GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - deploy

variables:
  ANSIBLE_HOST_KEY_CHECKING: "False"

.ansible-base:
  image: alpine/ansible:latest
  before_script:
    - apk add --no-cache openssh-client git
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -H $ANSIBLE_HOST >> ~/.ssh/known_hosts 2>/dev/null
    - echo "$ANSIBLE_VAULT_PASSWORD" > .vault_pass
  after_script:
    - rm -f .vault_pass

ansible-lint:
  extends: .ansible-base
  stage: lint
  script:
    - ansible-lint playbooks/

ansible-check:
  extends: .ansible-base
  stage: deploy
  script:
    - ansible-playbook -i inventory/prod.yml playbooks/site.yml --check --diff
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

ansible-deploy:
  extends: .ansible-base
  stage: deploy
  script:
    - ansible-playbook -i inventory/prod.yml playbooks/site.yml
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
  environment:
    name: production
```

**Что происходит:**

1. **Lint** на каждый push.
2. **Check** (dry-run) на PR.
3. **Deploy** после merge (manual).

### 🎯 GitHub Actions

```yaml
# .github/workflows/ansible.yml
name: Ansible

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      
      - name: Install ansible
        run: pip install ansible ansible-lint
      
      - name: Lint
        run: ansible-lint playbooks/

  deploy:
    needs: lint
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      
      - name: Install ansible
        run: pip install ansible
      
      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
      
      - name: Create vault password file
        run: echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > .vault_pass
      
      - name: Run playbook
        run: |
          ansible-playbook -i inventory/prod.yml playbooks/site.yml \
            --vault-password-file .vault_pass
      
      - name: Cleanup
        if: always()
        run: rm -f .vault_pass
```

### 🎯 AWX / Ansible Tower

**AWX** — open-source версия Ansible Tower.

**Что даёт:**

- **Web UI** для запуска playbooks.
- **RBAC** — кто может что запускать.
- **Credentials management** — централизованное хранение секретов.
- **Scheduling** — запуск по расписанию.
- **Audit log** — история запусков.
- **Templates** — сохранённые конфигурации запусков.

**Установка:** в Kubernetes через Operator.

**Когда использовать:**

- **Большие команды.**
- **Когда нужен UI.**
- **Когда нужен audit log.**

### 🎯 Ansible Pull

**`ansible-pull`** — обратный подход: сервер сам забирает playbook и применяет.

```bash
# На сервере
ansible-pull -U https://github.com/myorg/ansible.git playbooks/site.yml
```

**Что произойдёт:**

1. Ansible клонирует репозиторий.
2. Применяет playbook на **самом сервере**.
3. Работает как cron.

**Когда использовать:**

- **Serverless** (автоскейл).
- **Когда нет push-доступа** к серверам.
- **Когда нужна автономность.**

### 🎯 Линтеры

**ansible-lint:**

```bash
pip install ansible-lint
ansible-lint playbooks/
```

**Что проверяет:**

- Best practices.
- Deprecated модули.
- YAML-синтаксис.
- Безопасность.

**yamllint:**

```bash
pip install yamllint
yamllint .
```

**Проверка YAML.**

**Конфиги:**

**.ansible-lint:**

```yaml
skip_list:
  - no-changed-when
  - risky-file-permissions
exclude_paths:
  - .cache/
  - tests/
```

**.yamllint:**

```yaml
extends: default
rules:
  line-length:
    max: 120
  truthy:
    allowed-values: ['true', 'false', 'yes', 'no']
```

### 🔬 Практика: CI/CD для Ansible

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - deploy

image: python:3.12

before_script:
  - pip install ansible ansible-lint yamllint
  - mkdir -p ~/.ssh
  - echo "$SSH_KEY" > ~/.ssh/id_rsa
  - chmod 600 ~/.ssh/id_rsa

yamllint:
  stage: lint
  script:
    - yamllint .

ansible-lint:
  stage: lint
  script:
    - ansible-lint playbooks/

test-playbook:
  stage: test
  script:
    - echo "$VAULT_PASSWORD" > .vault_pass
    - ansible-playbook -i inventory/staging.yml playbooks/site.yml --check --diff
  only:
    - merge_requests

deploy-prod:
  stage: deploy
  script:
    - echo "$VAULT_PASSWORD" > .vault_pass
    - ansible-playbook -i inventory/prod.yml playbooks/site.yml
  when: manual
  only:
    - main
  environment:
    name: production
  after_script:
    - rm -f .vault_pass
```

### 💡 Практика: как правильно настроить CI/CD

**✅ ОБЯЗАТЕЛЬНО:**

1. **Lint в CI.**
2. **`--check --diff` на PR.**
3. **Manual approval для production.**
4. **Vault password в CI/CD Variables.**

**👍 СТОИТ:**

5. **AWX** для больших команд.
6. **`ansible-pull`** для serverless.
7. **Notifications** в Slack.

**❌ НЕ ДЕЛАЙ:**

8. **Не храни SSH-ключи в коде.**
9. **Не применяй в prod без ревью.**
10. **Не игнорируй lint.**

### Где мы сейчас

Мы разобрали CI/CD. Теперь — **тестирование**.

---

## 19.12 Тестирование playbooks

### 🔌 Проблема: как проверить playbook перед применением

Ты написал playbook. Он применяется на production. Если что-то не так — прод сломан.

**Решение:** тестирование.

### 📊 Уровни тестирования

**1. Lint** — синтаксис, best practices.

**2. `--check --diff`** — dry-run.

**3. Molecule** — unit-тесты в контейнерах/VMs.

**4. Integration tests** — реальное применение в тестовой среде.

### 🎯 Lint

**ansible-lint:**

```bash
ansible-lint playbooks/
```

**Проверки:**

- Deprecated модули.
- Hardcoded значения.
- Отсутствие `name`.
- Отсутствие `changed_when`.
- Security issues.

### 🎯 `--check --diff`

**Dry-run:**

```bash
ansible-playbook -i inventory/staging.yml site.yml --check --diff
```

**Что произойдёт:**

- Ansible пройдёт все tasks.
- **Не будет** применять изменения.
- Покажет, что **было бы** сделано.

**Ограничения:**

- Не все модули поддерживают check_mode.
- Некоторые tasks могут упасть.

### 🎯 Molecule

**Molecule** — инструмент для тестирования roles.

**Что делает:**

- Создаёт Docker-контейнер или VM.
- Применяет role.
- Проверяет результат (idempotence, verify).
- Удаляет.

**Установка:**

```bash
pip install molecule molecule-docker
```

**Инициализация для role:**

```bash
cd roles/nginx
molecule init scenario --driver-name docker
```

**Структура:**

```
roles/nginx/
├── molecule/
│   └── default/
│       ├── molecule.yml      # конфигурация
│       ├── converge.yml      # playbook для применения role
│       ├── verify.yml        # проверки
│       └── prepare.yml       # подготовка
```

**molecule.yml:**

```yaml
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: ubuntu-22.04
    image: ubuntu:22.04
    pre_build_image: true
  - name: rocky-9
    image: rockylinux:9
    pre_build_image: true
provisioner:
  name: ansible
  inventory:
    group_vars:
      all:
        nginx_port: 8080
verifier:
  name: ansible
```

**converge.yml:**

```yaml
---
- name: Converge
  hosts: all
  become: true
  roles:
    - role: nginx
```

**verify.yml:**

```yaml
---
- name: Verify
  hosts: all
  become: true
  tasks:
    - name: Check nginx is running
      service:
        name: nginx
        state: started
      check_mode: true
      register: result
      failed_when: result.changed
    
    - name: Check config
      stat:
        path: /etc/nginx/nginx.conf
      register: config
      failed_when: not config.stat.exists
    
    - name: Check port
      wait_for:
        port: 8080
        timeout: 5
```

**Запуск:**

```bash
# Создать и запустить
molecule test

# Только создать
molecule create

# Применить role
molecule converge

# Проверить
molecule verify

# Проверить идемпотентность
molecule idempotence

# Удалить
molecule destroy
```

**Что проверяет `molecule test`:**

1. **create** — создаёт контейнеры.
2. **converge** — применяет role.
3. **idempotence** — применяет снова, проверяет что не changed.
4. **verify** — проверяет результат.
5. **destroy** — удаляет.

**Цель:** все шаги проходят.

### 🎯 Integration Tests

**Testinfra** — Python-фреймворк для тестирования инфраструктуры.

**tests/test_nginx.py:**

```python
import pytest

def test_nginx_is_installed(host):
    nginx = host.package("nginx")
    assert nginx.is_installed

def test_nginx_running_and_enabled(host):
    nginx = host.service("nginx")
    assert nginx.is_running
    assert nginx.is_enabled

def test_nginx_config(host):
    config = host.file("/etc/nginx/nginx.conf")
    assert config.exists
    assert config.user == "root"
    assert config.mode == 0o644
    assert "worker_processes auto" in config.content_string

def test_nginx_listening(host):
    socket = host.socket("tcp://0.0.0.0:80")
    assert socket.is_listening
```

**Запуск:**

```bash
pip install pytest-testinfra
pytest tests/
```

### 🎯 Ansible Test Framework

**Ansible имеет встроенные тесты:**

**assert:**

```yaml
- name: Verify nginx is running
  assert:
    that:
      - "'nginx' in ansible_facts.packages"
      - nginx_status.status.ActiveState == "active"
    fail_msg: "Nginx is not configured correctly"
```

**`check_mode` в verify:**

```yaml
- name: Verify config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  check_mode: true
  register: result
  failed_when: result.changed     # если changed — конфиг не совпадает
```

### 🔬 Практика: Molecule

```bash
# 1. Установить Molecule
pip install molecule molecule-docker ansible

# 2. Создать role
ansible-galaxy init roles/nginx
cd roles/nginx

# 3. Инициализировать Molecule
molecule init scenario --driver-name docker

# 4. Настроить molecule.yml
cat > molecule/default/molecule.yml <<'EOF'
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: ubuntu-22.04
    image: ubuntu:22.04
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible
EOF

# 5. Написать простые tasks
cat > tasks/main.yml <<'EOF'
- name: Create test file
  copy:
    content: "Hello from Molecule!\n"
    dest: /tmp/molecule-test.txt
    mode: '0644'
EOF

# 6. Converge
cat > molecule/default/converge.yml <<'EOF'
---
- name: Converge
  hosts: all
  become: true
  roles:
    - role: nginx
EOF

# 7. Verify
cat > molecule/default/verify.yml <<'EOF'
---
- name: Verify
  hosts: all
  become: true
  tasks:
    - name: Check file exists
      stat:
        path: /tmp/molecule-test.txt
      register: file
    
    - name: Assert
      assert:
        that:
          - file.stat.exists
          - file.stat.mode == '0644'
EOF

# 8. Запустить тесты
molecule test
```

### 💡 Практика: как правильно тестировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **Lint в CI.**
2. **`--check --diff` перед применением.**
3. **Molecule для roles.**

**👍 СТОИТ:**

4. **Testinfra** для интеграционных тестов.
5. **Idempotency tests** (Molecule делает автоматически).
6. **Multi-OS tests** (Ubuntu, Rocky, ...).

**❌ НЕ ДЕЛАЙ:**

7. **Не применяй в production без тестов.**
8. **Не игнорируй lint warnings.**
9. **Не тестируй только на одном OS.**

### Где мы сейчас

Мы разобрали тестирование. Теперь — **диагностика**.

---

## 19.13 Диагностика проблем

### 🔌 Проблема: playbook упал с ошибкой

Ansible упал. Или task не работает. Как диагностировать?

### 🔍 Типичные проблемы

**1. `UNREACHABLE` — не может подключиться.**

**Причины:**

- Неправильный host/IP.
- SSH-ключ не настроен.
- Firewall блокирует.
- Неправильный пользователь.

**Диагностика:**

```bash
# Проверить ping
ansible web1 -m ping -vvv

# Проверить SSH вручную
ssh -i ~/.ssh/id_rsa ubuntu@10.0.1.10
```

**2. `FAILED!` — task упал.**

**Причины:**

- Неправильные параметры модуля.
- Недостаточно прав.
- Ресурс не существует.

**Диагностика:**

```bash
# Verbose
ansible-playbook site.yml -vvv

# Посмотреть конкретную task
ansible-playbook site.yml --start-at-task "Install nginx"
```

**3. `Permission denied`.**

**Причины:**

- Нужен `become: true`.
- Неправильный `become_user`.

**Решение:**

```yaml
- hosts: webservers
  become: true
  become_user: root
```

**4. `Module not found`.**

**Причины:**

- Не установлена коллекция.
- Неправильное имя модуля.

**Решение:**

```bash
ansible-galaxy collection install community.general
```

**5. `YAML syntax error`.**

**Причины:**

- Неправильные отступы.
- Табы вместо пробелов.
- Специальные символы.

**Решение:**

```bash
yamllint .
ansible-playbook site.yml --syntax-check
```

**6. `Variable undefined`.**

**Причины:**

- Переменная не определена.
- Опечатка в имени.
- Неправильный приоритет.

**Решение:**

```yaml
- debug:
    var: my_variable
```

**7. `Handlers not running`.**

**Причины:**

- Task не вернул `changed`.
- Забыли `notify`.

**Решение:**

```bash
ansible-playbook site.yml -vvv | grep -i handler
```

**8. `Idempotency issues` — второй запуск снова changed.**

**Причины:**

- `shell`/`command` без `creates`/`changed_when`.
- Неправильная логика.

**Решение:**

```yaml
- name: Command
  shell: /opt/script.sh
  args:
    creates: /opt/marker-file
```

### 🎯 Verbose режимы

```bash
ansible-playbook site.yml -v      # базовый
ansible-playbook site.yml -vv     # более подробно
ansible-playbook site.yml -vvv    # очень подробно
ansible-playbook site.yml -vvvv   # debug
```

**Уровни:**

- `-v` — вывод результата.
- `-vv` — параметры модуля.
- `-vvv` — SSH-команды.
- `-vvvv` — максимальная детализация.

### 🎯 Отладка

**`debug` module:**

```yaml
- name: Show variable
  debug:
    var: my_variable

- name: Show message
  debug:
    msg: "Value is {{ my_variable }}"
```

**`assert`:**

```yaml
- name: Check condition
  assert:
    that:
      - my_variable is defined
      - my_variable | length > 0
    fail_msg: "my_variable is not set correctly"
```

**`fail`:**

```yaml
- name: Fail explicitly
  fail:
    msg: "This should not happen"
  when: some_condition
```

**`pause`:**

```yaml
- name: Pause for debugging
  pause:
    prompt: "Check the server and press Enter"
```

### 🎯 `--step` — пошаговое выполнение

```bash
ansible-playbook site.yml --step
```

**Что произойдёт:** Ansible спросит перед каждой task, выполнять ли её.

**Полезно для:** отладки в реальном времени.

### 🎯 `--start-at-task`

```bash
ansible-playbook site.yml --start-at-task "Install nginx"
```

**Что произойдёт:** начнёт с указанной task.

**Полезно:** если предыдущие tasks уже выполнены.

### 🎯 `--list-tasks` и `--list-hosts`

```bash
# Список tasks
ansible-playbook site.yml --list-tasks

# Список hosts
ansible-playbook site.yml --list-hosts

# Список tags
ansible-playbook site.yml --list-tags
```

### 🎯 `--syntax-check`

```bash
ansible-playbook site.yml --syntax-check
```

**Что делает:** проверяет синтаксис без запуска.

### 🎯 Ansible Debugger

**Встроенный debugger:**

```yaml
- hosts: all
  debugger: on_failed        # включить при failed
  tasks:
    - name: Task
      command: /bin/false
```

**При падении — вход в debugger:**

```
[web1] TASK [Task] ***
fatal: [web1]: FAILED! => {"changed": false, "msg": "..."}

[web1] TASK [Task] ***
[web1] debug> p task
[web1] debug> p task.args
[web1] debug> p task_vars
[web1] debug> continue
```

**Команды:**

| Команда | Что делает |
|:---|:---|
| `p <var>` | Показать переменную |
| `task` | Показать текущую task |
| `task_vars` | Показать все переменные |
| `continue` | Продолжить |
| `quit` | Выйти |
| `redo` | Повторить task |

### 🎯 Логи

**Ansible логи:**

```ini
# ansible.cfg
[defaults]
log_path = /var/log/ansible.log
```

**Что даёт:** все запуски логируются.

### 🔬 Практика: отладка

```bash
# 1. Создать playbook с ошибкой
cat > broken.yml <<'EOF'
---
- hosts: localhost
  connection: local
  tasks:
    - name: Show undefined variable
      debug:
        var: undefined_var
    
    - name: Check
      assert:
        that:
          - undefined_var is defined
        fail_msg: "undefined_var is not defined!"
EOF

# 2. Запустить с verbose
ansible-playbook -i inventory.yaml broken.yml -vvv

# 3. Запустить с debugger
cat > broken.yml <<'EOF'
---
- hosts: localhost
  connection: local
  debugger: on_failed
  tasks:
    - name: Fail
      command: /bin/false
EOF

ansible-playbook -i inventory.yaml broken.yml
# На падении войдёт в debugger

# 4. Пошаговое выполнение
ansible-playbook -i inventory.yaml site.yml --step
```

### 💡 Практика: как правильно диагностировать

**✅ ОБЯЗАТЕЛЬНО:**

1. **`-vvv` для подробной информации.**
2. **`--syntax-check` перед запуском.**
3. **`--check --diff` для dry-run.**

**👍 СТОИТ:**

4. **`debug` module** для отладки.
5. **Ansible Debugger** для сложных случаев.
6. **`log_path`** для логирования.

**❌ НЕ ДЕЛАЙ:**

7. **Не игнорируй ошибки.** Читай сообщения.
8. **Не отлаживай в production.**
9. **Не забывай про `--check`.**

---

## Глоссарий

| Термин | Определение |
|:---|:---|
| **Ansible** | Инструмент конфигурационного управления. |
| **Inventory** | Список управляемых серверов. |
| **Playbook** | YAML-файл с play'ями. |
| **Play** | Группа tasks для группы хостов. |
| **Task** | Одна задача (вызов модуля). |
| **Module** | Действие (apt, copy, service). |
| **Handler** | Task, запускаемая по notify. |
| **Notify** | Уведомление handler'а. |
| **Role** | Переиспользуемый набор tasks/vars/templates. |
| **Facts** | Информация о сервере. |
| **Variables** | Переменные. |
| **Templates** | Jinja2-шаблоны. |
| **Inventory** | Список хостов. |
| **Dynamic inventory** | Inventory из API. |
| **Vault** | Шифрование секретов. |
| **Idempotency** | Многократный запуск = один результат. |
| **Check mode** | Dry-run. |
| **Diff** | Показать изменения. |
| **Tags** | Группировка tasks. |
| **Molecule** | Тестирование roles. |
| **Testinfra** | Тестирование инфраструктуры. |
| **AWX** | Open-source Ansible Tower. |
| **Ansible Galaxy** | Реестр ролей. |
| **ansible-pull** | Обратный pull-подход. |

---

## Что мы узнали?

- **Ansible** — инструмент конфигурационного управления. Агентless, push-based, YAML.
- **Inventory** — список серверов. Группы, переменные.
- **Playbooks** — YAML с plays и tasks. Modules — единицы работы.
- **Идемпотентность** — многократный запуск = один результат. Ключевая концепция.
- **Variables, facts, templates** — параметризация. Jinja2 для конфигов.
- **Handlers** — перезапуск сервисов только при изменении.
- **Roles** — переиспользование кода. Ansible Galaxy.
- **Vault** — шифрование секретов. Коммитить зашифрованные.
- **Динамический inventory** — AWS, GCP, K8s, Terraform state.
- **CI/CD** — lint, check, deploy. AWX для больших команд.
- **Molecule** — тестирование roles в контейнерах.
- **Диагностика** — `-vvv`, `--check`, debugger, `--syntax-check`.

---

## Типичные ошибки

- ❌ **Использовать `shell` вместо модулей.** Не идемпотентно.
- ❌ **Хардкодить значения.** Используй variables.
- ❌ **Не использовать handlers для сервисов.** Простой.
- ❌ **Коммитить секреты без Vault.**
- ❌ **Использовать один inventory для всех окружений.**
- ❌ **Не проверять идемпотентность.**
- ❌ **Применять в production без `--check`.**
- ❌ **Не использовать roles.** Дублирование.
- ❌ **Забывать про `become` для root-операций.**
- ❌ **Не тестировать playbooks.**
- ❌ **Игнорировать lint.**
- ❌ **Не использовать dynamic inventory для облаков.**
- ❌ **Отлаживать в production.**
- ❌ **Забывать про `.vault_pass` в `.gitignore`.**

---

## Для быстрого повторения

- **Установка:** `pip install ansible` или `brew install ansible`.
- **Inventory:** YAML-файл с groups и hosts.
- **Playbook:** `ansible-playbook -i inventory.yaml site.yml`.
- **Modules:** `apt`, `copy`, `template`, `service`, `user`, `file`.
- **Idempotency:** модули проверяют состояние. `changed` vs `ok`.
- **Variables:** `vars`, `group_vars/`, `host_vars/`, facts.
- **Templates:** Jinja2, `{{ var }}`, `{% for %}`, `{% if %}`.
- **Handlers:** `notify` + handler для перезапуска.
- **Roles:** `roles/<name>/` с tasks, handlers, defaults, templates.
- **Vault:** `ansible-vault create secrets.yml`.
- **Dynamic inventory:** `aws_ec2.yml`, `gcp_compute.yml`.
- **CI/CD:** lint + check + deploy.
- **Molecule:** `molecule test` для roles.
- **Диагностика:** `-vvv`, `--check`, `--syntax-check`, debugger.

---

## Вопросы для самопроверки

1. Что такое конфигурационное управление? Зачем нужно?
2. Чем Ansible отличается от Terraform, Chef, Puppet?
3. Что такое inventory? Как организовать группы?
4. Что такое playbook, play, task, module?
5. Что такое идемпотентность? Как Ansible её обеспечивает?
6. Что такое facts? Как их использовать?
7. Что такое handlers? Зачем нужны?
8. Что такое roles? Как создать и использовать?
9. Что такое Ansible Vault? Как использовать?
10. Что такое динамический inventory? Когда использовать?
11. Как настроить Ansible в CI/CD?
12. Что такое Molecule? Зачем нужен?
13. Playbook упал с ошибкой. Как диагностировать?
14. Что делать, если второй запуск playbook снова `changed`?
15. Как передать секреты в playbook безопасно?

---

## Ответы

**1. Конфигурационное управление**

Декларативное описание состояния серверов в коде. Автоматизация, идемпотентность, версионирование.

**2. Ansible vs альтернативы**

Ansible: агентless, push, YAML. Terraform: инфраструктура (VPC, VM). Chef: Ruby, агент. Puppet: DSL, агент. Ansible проще, Chef/Puppet мощнее.

**3. Inventory**

Список серверов. Группы: `webservers`, `databases`. Вложенные: `production > prod_webservers`. Переменные: `group_vars/`, `host_vars/`.

**4. Playbook, play, task, module**

Playbook — YAML-файл. Play — группа tasks для группы хостов. Task — одна задача. Module — действие (apt, copy).

**5. Идемпотентность**

Многократный запуск = один результат. Модули проверяют текущее состояние. `apt` не устанавливает, если уже есть. `service` не запускает, если уже running.

**6. Facts**

Информация о сервере, собираемая Ansible. `ansible_distribution`, `ansible_default_ipv4.address`, `ansible_memtotal_mb`. Просмотр: `ansible web1 -m setup`.

**7. Handlers**

Task, запускаемая по `notify` при `changed`. Для перезапуска сервисов при изменении конфига. Выполняются в конце play.

**8. Roles**

Переиспользуемый набор tasks, handlers, defaults, templates. `ansible-galaxy init role-name`. Использование: `roles: [nginx]` в playbook.

**9. Ansible Vault**

Шифрование секретов. `ansible-vault create secrets.yml`. Использование: `vars_files: [secrets.yml]`. Пароль: `--vault-password-file .vault_pass`.

**10. Динамический inventory**

Inventory из API (AWS, GCP, K8s). Плагины: `aws_ec2`, `gcp_compute`, `kubernetes`. Не нужно обновлять вручную.

**11. CI/CD для Ansible**

Lint → check (на PR) → deploy (после merge, manual). Vault password в CI/CD Variables. SSH-ключи через secrets.

**12. Molecule**

Инструмент для тестирования roles. Создаёт контейнер, применяет role, проверяет, удаляет. `molecule test`.

**13. Диагностика**

`-vvv` для verbose. `--syntax-check` для синтаксиса. `--check --diff` для dry-run. Debugger: `debugger: on_failed`. Логи в `/var/log/ansible.log`.

**14. Второй запуск changed**

Причина: `shell`/`command` без `creates`/`changed_when`. Решение: использовать модули или добавить `creates`. Проверить: `-vvv` для деталей.

**15. Секреты в playbook**

Ansible Vault (`ansible-vault create secrets.yml`). Или внешние менеджеры: HashiCorp Vault, AWS Secrets Manager. Не хранить в plain text.

---

## Куда идти дальше?

Мы разобрали Ansible — конфигурационное управление. Теперь ты знаешь:

- Inventory и группы.
- Playbooks, tasks, modules.
- Идемпотентность.
- Variables, facts, templates.
- Handlers.
- Roles.
- Vault.
- Динамический inventory.
- CI/CD.
- Тестирование.
- Диагностика.

Следующая глава по оглавлению — **Глава 20: GitOps — Git как источник правды**.

Скажи «дальше» — и я отправлю Главу 20.