# Infrastructure as Code — Terraform, Ansible, GitOps

Файл содержит вопросы по Infrastructure as Code (IaC): Terraform, Ansible, Puppet,
Chef, SaltStack, Pulumi, GitOps (ArgoCD, Flux), CI/CD и best practices.

Уровни сложности:
- ![Static Badge](https://img.shields.io/badge/Middle-yellow) — Middle разработчик
- ![Static Badge](https://img.shields.io/badge/Senior-red) — Senior разработчик

---

# Оглавление

1. [IaC — основы](#1-iac--основы)
2. [Terraform](#2-terraform)
3. [Ansible и управление конфигурацией](#3-ansible-и-управление-конфигурацией)
4. [GitOps и ArgoCD/Flux](#4-gitops-и-argocdflux)
5. [CI/CD](#5-cicd)
6. [Pulumi и альтернативы](#6-pulumi-и-альтернативы)
7. [Best practices и антипаттерны](#7-best-practices-и-антипаттерны)
8. [State management](#8-state-management)

---

## 1. IaC — основы

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Что такое Infrastructure as Code? Зачем нужен? ]

<details>
<summary>Ответ</summary>

* **IaC** — управление инфраструктурой через декларативные файлы, а не ручные действия в консоли.
* **Преимущества:**
  * **Version control** — инфраструктура в Git. История изменений, code review, rollback.
  * **Reproducibility** — один и тот же код создаёт идентичные окружения (dev = prod).
  * **Automation** — создание/обновление инфраструктуры в CI/CD.
  * **Documentation** — код описывает инфраструктуру лучше, чем диаграммы.
  * **Consistency** — нет "works on my machine" для инфраструктуры.
* **Подходы:**
  * **Declarative** (Terraform, CloudFormation) — описать желаемое состояние. Tool сам вычисляет
    diff и применяет.
  * **Imperative** (Ansible, shell scripts) — описать шаги для достижения состояния.
* **Инструменты:** Terraform, Pulumi, CloudFormation (AWS), Ansible, Chef, Puppet, SaltStack.

</details>

---

### Вопрос №2: [ Declarative vs Imperative IaC? Плюсы/минусы? ]

<details>
<summary>Ответ</summary>

| | Declarative | Imperative |
|--|-------------|------------|
| Что описывает | Желаемое состояние | Шаги для достижения |
| Idempotency | Встроена | Нужно обеспечивать |
| Diff | Tool вычисляет | Ручной или скрипт |
| Примеры | Terraform, CloudFormation, K8s manifests | Ansible, Bash, Python |
| Обучение | DSL (HCL, YAML) | Знакомый язык |
| Гибкость | Ограничена DSL | Полная |
| Replay | Любой apply безопасен | Повторный запуск может сломать |

* **Declarative** лучше для **инфраструктуры** (создание ресурсов, network, IAM).
* **Imperative** лучше для **конфигурации** (установка пакетов, деплой приложения).
* **Reality:** Ansible — declarative-ish (идемпотентный), но описывает шаги. Terraform —
  декларативный, но HCL имеет императивные элементы (for_each, count).

</details>

---

### Вопрос №3: [ Что такое idempotency? Почему важна для IaC? ]

<details>
<summary>Ответ</summary>

* **Idempotency** — повторное выполнение операции даёт тот же результат, что и первое.
* **Пример не-idempotent:**
  ```bash
  # Плохо: каждый запуск создаёт новый файл
  echo "config" >> /etc/app.conf
  ```
* **Пример idempotent (Ansible):**
  ```yaml
  - name: Ensure config line present
    ansible.builtin.lineinfile:
      path: /etc/app.conf
      line: "config"
      state: present  # ничего не сделает, если уже есть
  ```
* **Почему важна:**
  * IaC применяется много раз (обновление, восстановление, scale).
  * Если не idempotent — повторный запуск ломает состояние (дубликаты, конфликты).
  * Параллельное выполнение безопасно только при идемпотентности.
* **Terraform:** идемпотентность встроена. `terraform apply` несколько раз = один результат.
* **Ansible:** модули идемпотентны (lineinfile, file, apt). Raw shell — нет.

</details>

---

</details>

---

## 2. Terraform

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №4: [ Как устроен Terraform? Architecture? ]

<details>
<summary>Ответ</summary>

* **Terraform** — декларативный IaC tool от HashiCorp. HCL (HashiCorp Configuration Language).
* **Компоненты:**
  * **Configuration files** (`.tf`) — описание желаемой инфраструктуры.
  * **Provider** — плагин для взаимодействия с API (AWS, GCP, Azure, Kubernetes, GitHub).
  * **State** (`terraform.tfstate`) — текущее состояние инфраструктуры. JSON.
  * **Plan** — diff между config и state. Что будет создано/изменено/удалено.
  * **Apply** — выполнение plan.
* **Workflow:**
  1. `terraform init` — загрузка providers, модулей.
  2. `terraform plan` — вычисление diff. Не изменяет инфраструктуру.
  3. `terraform apply` — применение изменений.
  4. `terraform destroy` — удаление инфраструктуры.
* **State** — критичный компонент. Terraform сравнивает config со state. При `plan`/`apply` также
  делается refresh — обновление state из реального облака.

</details>

---

### Вопрос №5: [ Что такое Terraform state? Почему его нельзя терять? ]

<details>
<summary>Ответ</summary>

* **State** (`terraform.tfstate`) — JSON-файл с описанием всей созданной инфраструктуры.
* **Почему важен:**
  * Terraform сравнивает config **со state**. При `plan`/`apply` делает refresh state из реального
    облака. Без state — не знает, что уже создано.
  * Потеря state → Terraform пытается создать ресурсы заново → конфликты (resource already
    exists).
* **Remote state:**
  * **Никогда** не хранить state в Git (секреты, блокировки, мердж-конфликты).
  * Backend: S3 + DynamoDB (lock), GCS, Terraform Cloud, Consul, Azure Blob.
  * `backend "s3"` — шифрование (SSE), locking (DynamoDB), versioning.
* **State locking:** при `terraform plan` / `apply` / `refresh` — lock, чтобы два процесса не меняли state одновременно.
* **Sensitive data:** state содержит secrets (пароли, ключи). Ограничить доступ, шифровать.
* `terraform state list` — список ресурсов. `terraform state mv` — переименование. `terraform
  state rm` — удаление из state (но не из облака).

</details>

---

### Вопрос №6: [ Что такое Terraform module? Зачем нужен? ]

<details>
<summary>Ответ</summary>

* **Module** — переиспользуемый набор Terraform ресурсов. Функция/компонент инфраструктуры.
* **Структура:**
  ```
  modules/
    vpc/
      main.tf       — ресурсы
      variables.tf  — inputs
      outputs.tf    — outputs
      versions.tf   — version constraints
  ```
* **Использование:**
  ```hcl
  module "vpc" {
    source       = "./modules/vpc"
    cidr_block   = "10.0.0.0/16"
    env          = "prod"
  }
  ```
* **Зачем:**
  * **Reuse** — один модуль VPC для dev/staging/prod.
  * **Encapsulation** — сложная логика скрыта. Чистый интерфейс (variables).
  * **Consistency** — стандартная инфраструктура для всех проектов.
* **Registry:** [Terraform Registry](https://registry.terraform.io/) — community modules.
  * Но: проверять перед использованием. Не все качественные.
* **Best practice:** модуль = логическая единица (VPC, RDS, EKS, app).

</details>

---

### Вопрос №7: [ Как управлять переменными в Terraform? ]

<details>
<summary>Ответ</summary>

* **Input variables** (`variables.tf`):
  ```hcl
  variable "environment" {
    type        = string
    default     = "dev"
    description = "Environment name"
    validation {
      condition     = contains(["dev", "staging", "prod"], var.environment)
      error_message = "Must be dev, staging, or prod."
    }
  }
  ```
* **Способы передачи:**
  * `terraform.tfvars` — файл по умолчанию.
  * `*.auto.tfvars` — автоматически загружается.
  * `-var="environment=prod"` — CLI flag.
  * `TF_VAR_environment=prod` — environment variable.
  * `terraform.tfvars.json` — JSON формат.
* **Secrets:**
  * **Никогда** в `.tfvars` в Git.
  * **Terraform Cloud / Vault** — динамические секреты.
  * `sensitive = true` — значение не отображается в plan output.
  * `aws_secretsmanager_secret` — чтение из Secrets Manager.
* **Outputs** (`outputs.tf`):
  ```hcl
  output "vpc_id" {
    value = module.vpc.vpc_id
  }
  ```
  * Передача между модулями. `terraform output` — получить значения.

</details>

---

### Вопрос №8: [ Что такое `terraform plan`? Как читать output? ]

<details>
<summary>Ответ</summary>

* **`terraform plan`** — вычисление diff между config, state и реальностью. Не изменяет
  инфраструктуру.
* **Output:**
  ```
  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami           = "ami-12345"
      + instance_type = "t3.micro"
      + id            = (known after apply)
    }

  # aws_security_group.old will be destroyed
  - resource "aws_security_group" "old" {
      - id = "sg-123" -> null
    }

  # aws_db_instance.main will be updated in-place
  ~ resource "aws_db_instance" "main" {
      ~ backup_retention_period = 0 -> 7
        id                      = "db-123"
      }
  ```
* **Символы:**
  * `+` — будет создано.
  * `-` — будет удалено.
  * `~` — будет изменено (in-place).
  * `-/+` — будет пересоздано (destroy + create). **Опасно для stateful ресурсов (БД)!**
* **`plan -out=tfplan`** — сохранить plan. `apply tfplan` — применить именно этот plan (без
  пересчёта). Гарантия, что применяется то, что было review'нуто.

</details>

---

### Вопрос №9: [ Что такое `terraform import`? Когда использовать? ]

<details>
<summary>Ответ</summary>

* **`terraform import`** — добавление существующего ресурса в Terraform state (без создания).
* **Сценарий:**
  * Ресурс создан вручную (в консоли), нужно перенести под IaC.
  * Разделение state (ресурс переехал в другой модуль).
  * Восстановление после потери state.
* **Процесс:**
  ```bash
  terraform import aws_instance.web i-1234567890
  ```
  * Terraform fetch'ит атрибуты ресурса и записывает в state.
  * **Но не создаёт config!** Нужно вручную написать `resource "aws_instance" "web" {...}` и
    убедиться, что `plan` показывает no changes.
* **Проблема:** некоторые ресурсы не поддерживают import (зависит от provider'а, проверять документацию).
  С TF 1.5+ доступен декларативный блок `import {}` в конфигурации — в дополнение к команде `terraform import`.
* **Mass import:** `terraformer` (Google) — генерация HCL из существующей инфраструктуры.

</details>

---

</details>

---

## 3. Ansible и управление конфигурацией

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №10: [ Что такое Ansible? Как работает? ]

<details>
<summary>Ответ</summary>

* **Ansible** — agentless configuration management tool. Python, SSH.
* **Архитектура:**
  * **Control node** — где запускается Ansible.
  * **Managed nodes** — целевые серверы. SSH доступ, Python на ноде.
  * **Inventory** — список хостов (`inventory.ini` или `inventory.yml`).
  * **Playbook** — YAML-файл с задачами (plays).
  * **Module** — код, выполняющий действие (apt, file, template, command).
  * **Role** — набор tasks, handlers, templates, vars для переиспользования.
* **Выполнение:**
  1. Ansible читает inventory + playbook.
  2. Генерирует Python-скрипт для каждой задачи.
  3. Копирует скрипт на managed node по SSH.
  4. Выполняет скрипт, возвращает результат.
  5. Удаляет скрипт.
* **Idempotent:** модули проверяют состояние перед изменением. `apt` не переустанавливает, если
  пакет уже есть.
* **Agentless:** не нужен агент на серверах. Только SSH + Python.

</details>

---

### Вопрос №11: [ Что такое Ansible role? Зачем нужен? ]

<details>
<summary>Ответ</summary>

* **Role** — структурированный набор файлов Ansible для переиспользования.
* **Структура:**
  ```
  roles/
    nginx/
      tasks/main.yml      — задачи
      handlers/main.yml   — обработчики (restart, reload)
      templates/          — Jinja2 шаблоны
      files/              — статические файлы
      vars/main.yml       — переменные
      defaults/main.yml   — значения по умолчанию
      meta/main.yml       — зависимости, metadata
  ```
* **Использование:**
  ```yaml
  - hosts: webservers
    roles:
      - nginx
      - { role: postgres, when: db_enabled }
  ```
* **Зачем:**
  * **Reuse** — одна роль nginx для всех проектов.
  * **Organization** — большой playbook разбит на роли.
  * **Sharing** — Ansible Galaxy (community roles).
* **Ansible Galaxy:** `ansible-galaxy install geerlingguy.nginx`. Но проверять код перед
  использованием.

</details>

---

### Вопрос №12: [ Что такое Ansible Vault? Как использовать? ]

<details>
<summary>Ответ</summary>

* **Ansible Vault** — шифрование секретов в Ansible (пароли, ключи, сертификаты).
* **Использование:**
  ```bash
  # Создать encrypted file
  ansible-vault create secrets.yml
  # Зашифровать существующий
  ansible-vault encrypt group_vars/prod/secrets.yml
  # Расшифровать
  ansible-vault decrypt secrets.yml
  # Редактировать (не расшифровывая файл на диске)
  ansible-vault edit secrets.yml
  ```
* **В playbook:** переменные из зашифрованного файла используются как обычные.
* **Пароль:**
  * `--ask-vault-pass` — интерактивный ввод.
  * `--vault-password-file` — файл с паролем (вне Git).
  * Несколько vault passwords (`--vault-id`).
* **Best practice:**
  * Разные пароли для dev / staging / prod.
  * Vault password в CI/CD через secret manager (HashiCorp Vault, AWS Secrets Manager).
  * Не хранить vault password в Git.
* **Альтернатива:** не Ansible Vault, а внешнее secret management (Vault, AWS SSM). Ansible
  fetch'ит секреты в runtime.

</details>

---

### Вопрос №13: [ Ansible vs Terraform? Когда что? ]

<details>
<summary>Ответ</summary>

| | Terraform | Ansible |
|--|-----------|---------|
| Подход | Declarative | Imperative (но idempotent) |
| Фокус | Provisioning (создание ресурсов) | Configuration (настройка серверов) |
| State | Есть (state file) | Нет |
| Agent | Нет (provider API) | Нет (SSH) |
| Cloud resources | Отлично (VPC, RDS, IAM) | Ограниченно |
| OS configuration | Слабо | Отлично (packages, files, services) |
| Идемпотентность | Встроена | Через модули |

* **Terraform:** "создай 3 EC2, VPC, RDS, load balancer."
* **Ansible:** "на эти 3 EC2 установи nginx, задеплой приложение, настрой firewall."
* **Вместе:** Terraform создаёт инфраструктуру → передаёт IP в Ansible → Ansible настраивает.
* **Trend:** cloud-native (EKS/GKE) + Helm/ArgoCD заменяют Ansible для конфигурации. Ansible —
  для on-prem, bare metal, network devices.

</details>

---

### Вопрос №14: [ Что такое Puppet? Плюсы и минусы? ]

<details>
<summary>Ответ</summary>

* **Puppet** — инструмент управления конфигурациями с собственным декларативным DSL.
  Использует agent-server архитектуру: агенты на серверах периодически сверяются с
  master-сервером для обновления конфигурации.
* **Плюсы:**
  * Зрелость — один из старейших и наиболее зрелых инструментов на рынке.
  * Мощный DSL — специализированный язык для описания ресурсов и зависимостей.
  * Большое сообщество — обширная база модулей (Puppet Forge).
  * Отчётность — встроенные отчёты о состоянии инфраструктуры.
* **Минусы:**
  * Сложность — крутая кривая обучения из-за собственного DSL.
  * Требует агента — установка Puppet agent на каждом управляемом сервере.
  * Master-server — дополнительная точка отказа и обслуживания.
* **Сравнение с Ansible:**

| | Puppet | Ansible |
|--|--------|---------|
| DSL | Puppet DSL (свой язык) | YAML |
| Agent | Да (puppet agent) | Нет (SSH) |
| Архитектура | Master-agent | Agentless |
| Подходит для | Большие инфраструктуры, on-prem | Быстрая автоматизация, любые среды |

</details>

---

### Вопрос №15: [ Что такое Chef? Плюсы и минусы? ]

<details>
<summary>Ответ</summary>

* **Chef** — инструмент автоматизации управления конфигурациями, использует Ruby-подобный
  DSL для описания "рецептов" (recipes) и "поваренных книг" (cookbooks). Использует
  agent-server архитектуру (chef-client ↔ chef-server).
* **Плюсы:**
  * Гибкий DSL — на основе Ruby, позволяет писать сложную логику.
  * Тестирование — встроенная поддержка тестирования (Test Kitchen, ChefSpec).
  * Широкая экосистема — обширный набор cookbooks и интеграций.
* **Минусы:**
  * Сложность — Ruby DSL требует знания Ruby, перегрузка для новичков.
  * Требует агента — chef-client на каждом управляемом сервере.
  * Меньшая популярность — сообщество сокращается в пользу Ansible/Terraform.

</details>

---

### Вопрос №16: [ Что такое SaltStack? Плюсы и минусы? ]

<details>
<summary>Ответ</summary>

* **SaltStack (Salt)** — инструмент управления конфигурациями и оркестрации, использует
  декларативный язык на основе YAML/Jinja. Для связи между master и minion использует
  ZeroMQ, что обеспечивает высокую скорость распространения команд.
* **Плюсы:**
  * Высокая производительность — ZeroMQ обеспечивает быструю передачу на тысячи узлов.
  * Гибкость — поддерживает как agent (salt-minion), так и agentless (salt-ssh) режимы.
  * Оркестрация — встроенные возможности оркестрации и event-driven автоматизации.
  * Мониторинг — интегрированные функции мониторинга и реагирования на события.
* **Минусы:**
  * Сложность — множество функций и настроек, высокая кривая обучения.
  * Документация — может быть неполной или запутанной.
  * Меньшее сообщество — по сравнению с Ansible.

</details>

---

</details>

---

## 4. GitOps и ArgoCD/Flux

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №17: [ Что такое GitOps? Принципы? ]

<details>
<summary>Ответ</summary>

* **GitOps** — подход, где Git — единственный источник истины (source of truth) для инфраструктуры
  и приложений.
* **Принципы (OpenGitOps):**
  1. **Declarative** — система описана декларативно (YAML, HCL).
  2. **Versioned and immutable** — всё в Git. История, rollback, audit.
  3. **Pulled automatically** — агент в кластере pull'ит изменения (не push из CI).
  4. **Continuously reconciled** — агент сравнивает Git с реальностью, устраняет drift.
* **Push vs Pull:**
  * **Push** (традиционный CI/CD): CI runner → `kubectl apply`. Runner имеет доступ к кластеру.
  * **Pull** (GitOps): ArgoCD/Flux в кластере → pull Git → apply. CI не имеет доступа к кластеру.
* **Преимущества:**
  * **Security** — CI не имеет cluster credentials. Только Git.
  * **Audit** — каждое изменение в Git. Кто, когда, что.
  * **Rollback** — `git revert` → автоматический rollback.
  * **Drift detection** — если кто-то изменил ресурс вручную (`kubectl edit`) — GitOps откатит
    (при `auto-sync: true` + `selfHeal: true` в ArgoCD).

</details>

---

### Вопрос №18: [ Что такое ArgoCD? Как работает? ]

<details>
<summary>Ответ</summary>

* **ArgoCD** — GitOps continuous delivery tool для Kubernetes. Работает внутри кластера.
* **Архитектура:**
  * **API Server** — UI + API.
  * **Repository Server** — cache Git repo, render manifests (Helm, Kustomize).
  * **Application Controller** — reconciliation loop. Сравнивает Git (desired) с кластером (live).
* **Application CRD:**
  ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  spec:
    source:
      repoURL: https://github.com/org/manifests
      path: prod/payment-service
      targetRevision: main
    destination:
      server: https://kubernetes.default.svc
      namespace: payment
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
  ```
* **Sync:**
  * **Manual** — кнопка "Sync" в UI.
  * **Automated** — синхронизация при изменении Git. `prune: true` — удалять ресурсы, которых нет
    в Git. `selfHeal: true` — откатывать ручные изменения.
* **Multi-environment:** App-of-Apps pattern — одно Application создаёт другие Applications.

</details>

---

### Вопрос №19: [ ArgoCD vs Flux? Что выбрать? ]

<details>
<summary>Ответ</summary>

| | ArgoCD | Flux |
|--|--------|------|
| UI | Отличный Web UI | Минимальный (CLI + webhook) |
| Модель | Application CRD | GitRepository + Kustomization/HelmRelease |
| Multi-tenancy | RBAC, projects | Namespaced, limited |
| Sync | On-demand + auto | Auto (event-driven) |
| Helm | Нативная поддержка | Нативная (HelmRelease CRD) |
| Kustomize | Нативно | Нативно |
| Notifications | Built-in | External (notification controller) |
| Image update | Argo Image Updater | Flux Image Reflector + Automation |

* **ArgoCD:** если нужен UI, multi-tenancy, сложные sync waves, RBAC.
* **Flux:** если минимализм, CLI-driven, tight Git integration, event-driven.
* **Оба** CNCF Graduated. Оба подходят для большинства случаев.
* **Ecosystem:** ArgoCD — Argo Rollouts (canary, blue-green), Argo Workflows. Flux — Flagger
  (progressive delivery).

</details>

---

### Вопрос №20: [ Как реализовать progressive delivery с GitOps? ]

<details>
<summary>Ответ</summary>

* **Progressive delivery** — постепенный rollout (canary, blue-green) с автоматическим rollback.
* **Argo Rollouts** (с ArgoCD):
  ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: Rollout
  spec:
    strategy:
      canary:
        steps:
          - setWeight: 10        # 10% трафика на новую версию
          - pause: { duration: 5m }
          - setWeight: 30
          - analysis:            # анализ метрик
              templates:
                - templateName: success-rate
          - setWeight: 100
  ```
* **Flagger** (с Flux):
  * Аналогично. Canary CRD. Интеграция с Prometheus, Datadog.
* **Анализ:**
  * Анализирует метрики (error rate, latency) во время rollout.
  * Если метрики плохие → auto-rollback. Если хорошие → продолжить rollout.
* **Canary:** постепенный shift трафика (10% → 30% → 50% → 100%).
* **Blue-green:** две среды. Switch traffic instantly. Откат = switch back.
* **A/B testing:** split по заголовкам/cookies (Service Mesh, Istio).

</details>

---

</details>

---

## 5. CI/CD

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №21: [ Что такое CI/CD? ]

<details>
<summary>Ответ</summary>

* **CI/CD** — методология разработки, автоматизирующая процессы сборки, тестирования и
  развёртывания приложений.
* **CI (Continuous Integration):** автоматическая сборка и тестирование кода при каждом
  коммите в основную ветку. Цель — быстро интегрировать новый код и обнаруживать ошибки.
* **CD (Continuous Delivery):** каждый успешно протестированный код автоматически
  подготавливается к развёртыванию, но само развёртывание требует одобрения человека.
* **CD (Continuous Deployment):** каждый успешно протестированный код автоматически
  развёртывается в production без вмешательства человека.
* **Преимущества:**
  * Быстрые релизы — автоматизация ускоряет выпуск новых версий.
  * Качество — автоматические тесты снижают количество багов в production.
  * Предсказуемость — процесс разработки становится менее трудоёмким.
  * Лёгкость отката — возможность быстрого rollback до предыдущей версии.
  * Масштабируемость — CI/CD облегчает сотрудничество между командами.

</details>

---

### Вопрос №22: [ Какие системы CI/CD существуют? ]

<details>
<summary>Ответ</summary>

* **Jenkins** — open-source, plugin-based, self-hosted. Самый популярный on-prem.
* **GitLab CI/CD** — встроен в GitLab, YAML-based, runners.
* **GitHub Actions** — встроен в GitHub, marketplace, matrix builds.
* **CircleCI** — cloud-native, Docker-based, parallelism.
* **Travis CI** — cloud, YAML-based, популярный для open-source.
* **TeamCity** — JetBrains, powerful, self-hosted/cloud.
* **Spinnaker** — multi-cloud continuous delivery, канареечные деплои.
* **ArgoCD / Flux** — GitOps CD для Kubernetes.
* **Tekton** — Kubernetes-native, modular pipelines.

</details>

---

### Вопрос №23: [ Какие best practices для CI/CD? ]

<details>
<summary>Ответ</summary>

* **Коммитьте часто** — маленькие изменения легче тестировать и откатывать.
* **Staging ≈ Production** — тестовая среда должна быть клоном production.
* **Очистка ресурсов** — pipeline должен удалять временные ресурсы (контейнеры, env).
* **Детерминированность** — pipeline должен давать одинаковый результат локально и в CI.
* **CI/CD как приложение** — относитесь к pipeline как к коду, с тестами и ревью.
* **Среды по требованию** — ephemeral environments вместо заранее выделенных.
* **Переиспользование шагов** — общие задачи (clone, build, deploy) между микросервисами.
* **Fast feedback** — pipeline должен работать быстро, кешировать зависимости.
* **Secret management** — секреты в vault, не в коде репозитория.

</details>

---

### Вопрос №24: [ Где хранить pipeline определения CI/CD? ]

<details>
<summary>Ответ</summary>

* **Репозиторий приложения** — pipeline в том же репозитории, что и приложение. Самый
  популярный подход, проще поддерживать.
* **Центральный репозиторий** — все pipeline организации в одном репозитории. Удобно при
  множестве команд и общих задач.
* **Отдельный CI-репозиторий для каждого приложения** — код CI отделён от кода приложения.
  Худший вариант из-за сложности синхронизации и поддержки.
* **На платформе выполнения** — например, pipeline в Kubernetes (Tekton) или в самом
  CI-сервере (Jenkinsfile в Jenkins).

</details>

---

### Вопрос №25: [ Как планировать capacity для CI/CD? ]

<details>
<summary>Ответ</summary>

* **Анализируйте рабочую нагрузку** — частота коммитов, длительность pipeline, количество
  параллельных задач.
* **Мониторинг текущего использования** — CPU, memory, disk, network на CI-серверах/runners.
* **Выявление узких мест** — самые медленные шаги, queue time, нехватка runners.
* **Прогноз спроса** — рост команды, новых сервисов, частоты релизов.
* **Масштабируемость** — auto-scaling runners (Kubernetes, cloud spot instances).
* **Стоимость** — баланс между self-hosted и cloud, spot instances для экономии.
* **Постоянная корректировка** — регулярно пересматривать capacity на основе метрик.

</details>

---

### Вопрос №26: [ Как организовать CI/CD для Terraform? ]

<details>
<summary>Ответ</summary>

* **Pipeline stages:**
  1. **fmt** — `terraform fmt -check` — форматирование.
  2. **validate** — `terraform validate` — синтаксис.
  3. **lint** — `tflint` — best practices (deprecated syntax, unused vars).
  4. **security** — `tfsec` / `checkov` — security scanning (public S3 bucket, no encryption).
  5. **plan** — `terraform plan -out=tfplan` + сохранение артефакта. Comment в PR.
  6. **review** — человек смотрит plan в PR. Approve.
  7. **apply** — `terraform apply tfplan` (после merge в main).
* **Tools:**
  * **Atlantis** — bot, который запускает Terraform при комментариях в PR (`atlantis plan`,
    `atlantis apply`). PR-based workflow.
  * **Terraform Cloud / Enterprise** — managed CI/CD для Terraform.
  * **GitHub Actions** — self-hosted с OIDC → cloud (без static credentials).
* **OIDC:** GitHub Actions → AWS через OIDC (no long-lived keys):
  ```yaml
  permissions:
    id-token: write   # нужно для обмена токена на временные креды
    contents: read
  jobs:
    terraform:
      steps:
        - uses: aws-actions/configure-aws-credentials@v4
          with:
            role-to-assume: arn:aws:iam::123456789012:role/gha-terraform
            aws-region: eu-central-1
  ```
  (Роль в IAM настраивается с trust policy на федерацию `token.actions.githubusercontent.com`.)
* **Branch protection:** main — protected. Apply только из main. Plan — из любой ветки.

</details>

---

### Вопрос №27: [ Что такое tfsec / checkov? Зачем нужны? ]

<details>
<summary>Ответ</summary>

* **tfsec** (now part of Aqua Security) — static analysis security scanner для Terraform.
* **checkov** (Bridgecrew / Prisma Cloud) — broader IaC scanner (Terraform, CloudFormation,
  K8s, Docker, Helm).
* **Что находят:**
  * S3 bucket без encryption / public access.
  * Security group с 0.0.0.0/0 на port 22.
  * RDS без backup / без encryption.
  * IAM policy с `Action: *` / `Resource: *`.
  * Missing tags (cost allocation).
* **В CI/CD:**
  ```bash
  tfsec . --soft-fail=false  # fail build при critical
  checkov -d . --framework terraform
  ```
* **Suppress:** `# tfsec:ignore:aws-s3-no-public-access` — для false positives.
* **Custom policies:** checkov — Python policies. tfsec — custom rules.
* **Важно:** scanner проверяет config, не runtime. Если кто-то изменил ресурс вручную — scanner
  не увидит.

</details>

---

### Вопрос №28: [ Как управлять multiple environments в Terraform? ]

<details>
<summary>Ответ</summary>

* **Подходы:**
  1. **Directories:**
     ```
     environments/
       dev/
         main.tf
         terraform.tfvars
       staging/
         main.tf
         terraform.tfvars
       prod/
         main.tf
         terraform.tfvars
     ```
     * Плюс: раздельные root configurations. State изолирован только при отдельных backend/state keys;
       разные каталоги с одним backend key используют общий state. Минус: дублирование.
  2. **Workspaces:**
     ```bash
     terraform workspace new prod
     terraform workspace select prod
     terraform apply -var-file=prod.tfvars
     ```
     * Плюс: один код. Минус: state в одном backend, сложно audit. **Не рекомендуется для
       prod** (risk of applying prod config to dev workspace).
  3. **Modules + tfvars:** общие модули, отдельные root configurations окружений:
     ```text
     modules/
       vpc/
       rds/
     environments/
       dev/
         main.tf
         terraform.tfvars
       prod/
         main.tf
         terraform.tfvars
     ```
     Оба `main.tf` вызывают общие модули. Пример раздельных ключей в remote backend:

     | Root configuration | Backend key |
     |---|---|
     | `environments/dev` | `dev/terraform.tfstate` |
     | `environments/prod` | `prod/terraform.tfstate` |

     * До plan/apply каждый root должен быть инициализирован со своим backend key;
       проверьте выбранные backend/state и окружение. Модули и разные `-var-file` сами по себе
       state не изолируют: tfvars меняют только значения переменных.
     * Плюс: DRY. Изоляция state обеспечивается отдельной настройкой backend, а не модулями.
* **Best practice:** directories (подход 1) или modules + tfvars (подход 3). Не workspaces для
  prod. Для prod нужны отдельные credentials/роли и права на state и ресурсы:
  dev/staging credentials не должны иметь доступа к prod. Разные state keys сами по себе права не разделяют.

</details>

---

### Вопрос №29: [ Что такое Terragrunt? Зачем нужен? ]

<details>
<summary>Ответ</summary>

* **Terragrunt** — thin wrapper для Terraform от Gruntwork. Решает проблемы DRY и remote state.
* **Возможности:**
  * **DRY:** один модуль, multiple environments через `terragrunt.hcl`:
    ```hcl
    # environments/prod/vpc/terragrunt.hcl
    terraform {
      source = "../../../modules/vpc"
    }
    inputs = {
      cidr_block = "10.0.0.0/16"
      env        = "prod"
    }
    ```
  * **Remote state:** автоматическая конфигурация backend:
    ```hcl
    remote_state {
      backend = "s3"
      config = {
        bucket = "my-tf-state"
        key    = "${path_relative_to_include()}/terraform.tfstate"
        region = "us-east-1"
      }
    }
    ```
  * **before_hook / after_hook** — команды до/после Terraform.
  * **Dependencies** — `dependency "vpc"` — использовать outputs VPC в RDS.
  * **Parallel apply** — `terragrunt apply-all` для всех модулей.
* **Когда нужен:** много environments, много модулей, boilerplate в backend config.

</details>

---

</details>

---

## 6. Pulumi и альтернативы

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №30: [ Что такое Pulumi? Чем отличается от Terraform? ]

<details>
<summary>Ответ</summary>

* **Pulumi** — IaC tool, использующий реальные языки программирования (Go, TypeScript, Python,
  C#, Java).
* **Пример (TypeScript):**
  ```typescript
  import * as aws from "@pulumi/aws";
  const bucket = new aws.s3.Bucket("my-bucket", {
    bucket: "my-bucket",
  });
  export const bucketName = bucket.id;
  ```
  (Свойство `acl` убрано: с @pulumi/aws v5 оно deprecated, в v6 удалено — новые бакеты
  по умолчанию создаются с отключёнными ACL; управление доступом — через bucket policy.)
* **Отличия от Terraform:**
  | | Pulumi | Terraform |
  |--|--------|-----------|
  | Язык | Go, TS, Python, C# | HCL (DSL) |
  | Логика | Полная (loops, functions, classes) | Ограничена HCL |
  | State | Pulumi Service / self-managed | Terraform Cloud / self-managed |
  | Provider | 100+ (shared с Terraform via bridge) | 100+ |
  | Тестирование | Нативное (unit tests на языке) | Terratest (Go), kitchen-terraform |
  | Обучение | Знакомый язык | Новый DSL |

* **Когда Pulumi:** разработчики пишут IaC (знакомый язык), сложная логика, нужны unit tests.
* **Когда Terraform:** ops-команда, проще для ops-инженеров, огромный community.

</details>

---

### Вопрос №31: [ Что такое CloudFormation? Чем отличается от Terraform? ]

<details>
<summary>Ответ</summary>

* **AWS CloudFormation** — нативный IaC tool от AWS. JSON/YAML templates.
* **Отличия:**
  | | CloudFormation | Terraform |
  |--|----------------|-----------|
  | Provider | Только AWS | Multi-cloud (AWS, GCP, Azure, K8s, etc.) |
  | Язык | JSON/YAML | HCL |
  | State | AWS-managed (не нужно беспокоиться) | Self-managed (или Terraform Cloud) |
  | Drift detection | Built-in (`detect_stack_drift`) | `terraform plan` (но нет continuous) |
  | Rollback | Automatic (при failure — rollback к предыдущему) | Нет (нужно `terraform apply` с fix) |
  | Cost | Бесплатно | Бесплатно (Terraform Cloud — платный) |
  | Community | AWS-only | Огромный, multi-cloud |

* **CloudFormation Nested Stacks** — аналоги модулей Terraform.
* **CDK (Cloud Development Kit)** — code (TS, Python, Go) → генерирует CloudFormation. Похож на
  Pulumi, но только AWS.
* **Выбор:** AWS-only → CloudFormation/CDK. Multi-cloud → Terraform.

</details>

---

</details>

---

## 7. Best practices и антипаттерны

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №32: [ Какие best practices для Terraform? ]

<details>
<summary>Ответ</summary>

* **State:**
  * Remote state (S3 + DynamoDB lock). Никогда в Git.
  * Раздельные state files для разных environments (не один гигантский state).
  * Раздельные state для разных слоёв (network, data, app).
* **Code:**
  * Модули для переиспользования. Один модуль = одна логическая единица.
  * `versions.tf` — пинить версии providers и Terraform.
  * `variables.tf` с `description`, `type`, `validation`.
  * `outputs.tf` — только нужные значения. `sensitive = true` для секретов.
* **Workflow:**
  * PR-based: branch → `terraform plan` (comment в PR) → review → merge → `terraform apply`.
  * `terraform plan -out=tfplan` → `terraform apply tfplan` (не пересчитывать).
  * Никогда не `apply` без `plan` review в prod.
* **Security:**
  * `tfsec` / `checkov` в CI.
  * OIDC вместо static credentials.
  * Секреты — не в variables. Secrets Manager / Vault.
* **Naming:** consistent naming convention (`{project}-{env}-{resource}`).
* **Tags:** обязательные теги (Environment, Project, Owner, CostCenter).

</details>

---

### Вопрос №33: [ Какие антипаттерны IaC? ]

<details>
<summary>Ответ</summary>

* **State in Git** — секреты, merge conflicts, no locking.
* **One giant state file** — медленный plan/apply, blast radius, lock contention.
* **Manual changes** — `kubectl edit`, AWS console. Drift. IaC не знает об изменении. Next apply
  может сломать.
* **No plan review** — `terraform apply` без plan в prod. Катастрофа.
* **`depends_on` abuse** — Terraform автоматически вычисляет зависимости через references.
  `depends_on` — крайний случай.
* **`count = 0` for disabling** — лучше `enabled = false` variable + conditional.
* **Provider aliases без необходимости** — усложняет код. Multi-region — OK, но структурировать.
* **Ignore changes** (`lifecycle { ignore_changes = [...] }`) — если игнорируете атрибут, вы
  теряете контроль. Документировать причину.
* **No version pinning** — `source = "hashicorp/aws"` (без version). Breaking changes при
  обновлении provider.
* **Hardcoded values** — AMI IDs, regions. Использовать data sources (`data.aws_ami`).

</details>

---

### Вопрос №34: [ Что такое drift detection? Как управлять? ]

<details>
<summary>Ответ</summary>

* **Drift** — расхождение между IaC (Git/state) и реальной инфраструктурой.
* **Причины:**
  * Ручные изменения (console, CLI).
  * Внешние процессы (auto-scaling,lambda rotation).
  * Другие tools изменяют тот же ресурс (CloudFormation + Terraform).
* **Обнаружение:**
  * `terraform plan` — показывает drift (но нужно запустить).
  * **Continuous drift detection:** Terraform Cloud, driftctl, Spacelift. Периодически plan без
    apply.
  * CloudFormation `DetectStackDrift`.
* **Управление:**
  * **Prevent:** нет доступа к console/CLI. Только через IaC. IAM policies.
  * **Detect:** регулярный `terraform plan` в CI (nightly job). Alert при drift.
  * **Remediate:** `terraform apply` (вернуть к IaC) ИЛИ обновить IaC (если ручное изменение
    правильное).
  * `lifecycle { ignore_changes = [...] }` — для атрибутов, которые меняются внешне (tag added
    by AWS, etc.).
* **driftctl** — open source tool для обнаружения unmanaged resources (ресурсы в облаке, которых
  нет в Terraform).

</details>

---

</details>

---

## 8. State management

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №35: [ Как масштабировать Terraform state? Стратегии разделения? ]

<details>
<summary>Ответ</summary>

* **Проблема:** один state file → медленно (plan берёт минуты), blast radius (ошибка в одном
  ресурсе ломает всё), lock contention (команда ждёт).
* **Стратегии:**
  1. **Per-environment:**
     ```
     state/dev/terraform.tfstate
     state/staging/terraform.tfstate
     state/prod/terraform.tfstate
     ```
  2. **Per-layer (stack):**
     ```
     state/prod/network/terraform.tfstate    # VPC, subnets, NAT
     state/prod/data/terraform.tfstate       # RDS, ElastiCache
     state/prod/app/terraform.tfstate        # EKS, services
     state/prod/dns/terraform.tfstate        # Route53
     ```
  3. **Per-team / per-service:**
     ```
     state/team-a/terraform.tfstate
     state/team-b/terraform.tfstate
     ```
* **Cross-state references:** `terraform_remote_state`:
  ```hcl
  data "terraform_remote_state" "network" {
    backend = "s3"
    config = {
      bucket = "my-state"
      key    = "prod/network/terraform.tfstate"
    }
  }
  # Использовать outputs
  vpc_id = data.terraform_remote_state.network.outputs.vpc_id
  ```
* **Размер state:** жёсткого лимита нет, но большой state замедляет plan/apply и повышает
  риск коллизий при блокировках. Эмпирическое правило из сообщества — держать несколько сотен
  ресурсов на state и делить дальше (модули, разделение по окружениям/слоям).

</details>

---

### Вопрос №36: [ Как безопасно рефакторить Terraform? ]

<details>
<summary>Ответ</summary>

* **`terraform state mv`** — переименование ресурса в state без destroy/create:
  ```bash
  terraform state mv aws_instance.web aws_instance.frontend
  ```
  * Использовать при переименовании resource block.
* **`terraform state mv` (cross-state):**
  ```bash
  terraform state mv -state-out=new.tfstate aws_instance.web aws_instance.web
  ```
* **`terraform state rm`** — удалить из state (ресурс остаётся в облаке, но Terraform больше им
  не управляет). Для "forget" ресурса.
* **`terraform import`** — добавить существующий ресурс в state.
* **Процесс рефакторинга:**
  1. `terraform plan` — убедиться, что no changes.
  2. Изменить config (переименовать, переместить).
  3. `terraform state mv` — обновить state.
  4. `terraform plan` — убедиться, что no changes (config и state синхронизированы).
  5. `terraform apply` — если есть другие изменения.
* **Backup:** перед рефакторингом — backup state (`terraform state pull > backup.json`).
* **Small steps:** одно изменение за раз. Не рефакторить + добавлять ресурсы одновременно.

</details>

---

### Вопрос №37: [ Что делать при потере Terraform state? ]

<details>
<summary>Ответ</summary>

* **Scenario:** state file удалён / повреждён / недоступен.
* **Восстановление:**
  1. **Backup:** проверить backup'ы backend'а (S3 versioning — восстановить предыдущую версию).
  2. **`terraform import`:** для каждого ресурса в config — `terraform import`. Долго, но
     работает.
  3. **`terraformer`:** если ресурсов много — автоматически сгенерировать config из существующей
     инфраструктуры.
* **Prevention:**
  * S3 versioning — включён. Можно восстановить.
  * S3 MFA delete — защита от случайного удаления.
  * DynamoDB lock table — тоже с backup.
  * Регулярный `terraform state pull > backup.json` → в secure storage.
  * Terraform Cloud / Enterprise — managed state с backup'ами.
* **Если import невозможен:** удалить ресурсы вручную и создать через `terraform apply`. Риск:
  данные (БД) могут быть потеряны.

</details>

---

### Вопрос №38: [ Как управлять секретами в IaC? ]

<details>
<summary>Ответ</summary>

* **Проблема:** state file содержит secrets в plaintext. `.tfvars` с секретами не должен быть в
  Git.
* **Подходы:**
  1. **External secrets store:**
     ```hcl
     data "aws_secretsmanager_secret_version" "db_password" {
       secret_id = "prod/db/password"
     }
     resource "aws_db_instance" "main" {
       password = data.aws_secretsmanager_secret_version.db_password.secret_string
     }
     ```
     * Секрет не в Git, не в tfvars. Но **в state** (plaintext).
  2. **`sensitive = true`:**
     * Скрыть из plan output. **Но в state всё равно plaintext.**
  3. **HashiCorp Vault:**
     ```hcl
     provider "vault" {}
     data "vault_generic_secret" "db" {
       path = "secret/data/prod/db"
     }
     ```
     * Динамические секреты (срок действия). В state — может быть.
  4. **`lifecycle { ignore_changes = [password] }`:** (список ссылок на атрибуты, не строки; строковый синтаксис устарел).
     * Секрет задаётся вне Terraform (управляется приложением). Terraform не управляет паролем.
  5. **Encrypted state:** Terraform Cloud / Enterprise — шифрование state. Self-managed —
     KMS-encrypted S3 (но Terraform расшифровывает в памяти).
* **Best practice:** external secrets store (Secrets Manager / Vault) + `sensitive = true` +
  ограниченный доступ к state file.

</details>

---

</details>

---

## Полезные ресурсы

- [Terraform Best Practices](https://www.terraform-best-practices.com/)
- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [OpenGitOps Principles](https://opengitops.dev/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Ansible Documentation](https://docs.ansible.com/)
- [Pulumi Documentation](https://www.pulumi.com/docs/)

---

## Связанные файлы

- [devops-questions.md](devops-questions.md) — Docker, K8s deployment
- [security.md](security.md) — IaC security, secrets management
- [observability.md](observability.md) — monitoring инфраструктуры
- [db-operations.md](db-operations.md) — IaC для БД
- [system-design-questions.md](system-design-questions.md) — архитектура infra
