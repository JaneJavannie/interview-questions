# Backend Interview Questions
Репозиторий предназначен для подготовки к собеседованиям на позицию Backend разработчика.  
Вопросы и ответы распределены по нескольким файлам для удобства навигации.

## Инструкции по применению
**На данный момент репозиторий содержит:**  

➡️ Теоретические вопросы: 760+ (распределены по файлам ниже)  
➡️ Практические задачи: 33  

## Структура репозитория

| Файл | Разделы | Кол-во вопросов |
|------|---------|-----------------|
| [README.md](README.md) | Главная страница (ссылки на перенесённые разделы) | 0 |
| [algorithms.md](algorithms.md) | Алгоритмы (сортировка, поиск, внешняя сортировка, анализ сложности) | 3 |
| [sql-postgres-questions.md](sql-postgres-questions.md) | Базы данных, SQL, PostgreSQL (внутреннее устройство, оптимизация, практика) | 79 |
| [go-questions.md](go-questions.md) | Golang (Map/Swiss Tables, GC, планировщик, горутины, sync, дженерики, reflection, практика) | 177 |
| [system-design-questions.md](system-design-questions.md) | System Design (CAP, шардирование, масштабирование, практические примеры систем, принципы и методологии, общие темы) | 37 |
| [redis-questions.md](redis-questions.md) | Redis (внутреннее устройство, репликация, кластер, паттерны), in-memory хранилища (сравнение) | 20 |
| [devops-questions.md](devops-questions.md) | Docker, Kubernetes (архитектура, операции, практические задачи), Deployment, Cloud, Виртуализация | 32 |
| [go-testing-tooling.md](go-testing-tooling.md) | Go testing (table tests, benchmarks, fuzzing, mocking, integration testing) | 42 |
| [message-brokers.md](message-brokers.md) | Kafka, RabbitMQ, NATS, SQS/SNS, EventBridge (архитектура, паттерны, сравнение, exactly-once) | 46 |
| [api-design.md](api-design.md) | API Design (REST, GraphQL, gRPC, versioning, pagination, authentication) | 38 |
| [linux-internals.md](linux-internals.md) | Linux internals (процессы, IPC, файловая система, cgroups, namespaces, eBPF) | 49 |
| [db-operations.md](db-operations.md) | DB Operations (миграции, бэкапы, репликация, шардирование, VACUUM, сравнение БД, практики) | 43 |
| [security.md](security.md) | Security (OWASP, аутентификация, авторизация, шифрование, secrets, network security) | 54 |
| [observability.md](observability.md) | Observability (метрики, логи, трейсы, OpenTelemetry, Prometheus, SLO/SLI) | 32 |
| [infrastructure-as-code.md](infrastructure-as-code.md) | IaC (Terraform, Ansible, Puppet, Chef, SaltStack, Pulumi, GitOps, CI/CD, state management, best practices) | 38 |
| [serverless.md](serverless.md) | Serverless (AWS Lambda, Step Functions, Serverless БД, patterns, cost optimization) | 42 |
| [networking.md](networking.md) | Сеть (TCP/IP, HTTP, DNS, DHCP, CIDR, VPN, Load Balancing), сетевые инструменты | 12 |
| [programming-concepts.md](programming-concepts.md) | Общие концепции программирования (парадигмы, SOLID, дизайн-паттерны) | 5 |
| [git-questions.md](git-questions.md) | GIT (основы, ветвление, коммиты, force push, clean) | 13 |

1️⃣ Настоятельно рекомендую, прежде чем открывать ответ на вопрос попробовать ответить на него самостоятельно.  
2️⃣ Если ответить не получается, попробуйте выдумать ответ основываясь на уже имеющихся у вас знаниях.  
3️⃣ И только если вы в полном ступоре открывайте ответ на вопрос.  

▶️ У практических задач рядом с вопросом будет стоять сложность: ![Static Badge](https://img.shields.io/badge/Easy_peasy-brightgreen) ![Static Badge](https://img.shields.io/badge/Wow-yellow?color=yellow) ![Static Badge](https://img.shields.io/badge/Holy_Moly-red) 

---

P.S.  

*️⃣ Вы будете благодарны себе если выполните инструкцию в точности как написано.  

P.S.S.  

⏺️ Если вы заметили неточность.  
⏺️ У вас есть свой вопрос который вы добавили бы в список.  
⏺️ Вы недавно проходили собеседование.  
🆓 То смело пишите мне в [Telegram](https://t.me/drburgerm) я обязательно исправлю/добавлю вопрос/ответ.

---

## Features
Данный репозиторий будет обновляться по мере появления новых вопросов и актуальной информации.  
Также в разделах будут добавлены другие языки программирования (с этим я прошу помощи у неравнодушного сообщества GitHub)  
Давайте поможем друг другу и новичкам пришедшим в мир IT 🙌

---
<p align="center">
  <img src="https://github.com/DrrBurger/Photos_for_git/blob/master/Untitled-1.png" alt="logo">
</p>

---
### 1️⃣ Общие темы:

#### Перенесённые разделы

> **Общие вопросы** — перенесены по файлам:
> - Микросервисы, монолит vs микросервис, message broker, CQRS, zero-downtime deployment → [system-design-questions.md](system-design-questions.md)
> - Анализ сложности алгоритмов → [algorithms.md](algorithms.md)
> - Reflection, CAS/atomic → [go-questions.md](go-questions.md)
> - Парадигмы программирования, SOLID, дизайн-паттерны → [programming-concepts.md](programming-concepts.md)
> - Blue-Green Deployment, feature flags, Cloud, Виртуализация → [devops-questions.md](devops-questions.md)
> **Сеть** — перенесены в отдельный файл: [networking.md](networking.md)
> **Облачные сервисы** — перенесены в отдельный файл: [devops-questions.md](devops-questions.md)
> **Виртуализация** — перенесены в отдельный файл: [devops-questions.md](devops-questions.md)
> **Сетевые инструменты** — перенесены в отдельный файл: [networking.md](networking.md)
> **GIT** — перенесён в отдельный файл: [git-questions.md](git-questions.md)
> **Операционная система** — перенесены в отдельный файл: [linux-internals.md](linux-internals.md)
> **Kafka / Очереди сообщений** — перенесены в отдельный файл: [message-brokers.md](message-brokers.md)
> **Базы данных** — перенесены в отдельный файл: [sql-postgres-questions.md](sql-postgres-questions.md)
> **Алгоритмы** — перенесены в отдельный файл: [algorithms.md](algorithms.md)
> **Управление конфигурацией** — перенесены в отдельный файл: [infrastructure-as-code.md](infrastructure-as-code.md)
> **Redis / in-memory хранилища** — перенесены в отдельный файл: [redis-questions.md](redis-questions.md)
> **Контейнеризация и оркестрация** — перенесены в отдельный файл: [devops-questions.md](devops-questions.md)
> **CI/CD** — перенесены в отдельный файл: [infrastructure-as-code.md](infrastructure-as-code.md)
> **Мониторинг и логирование** — перенесены в отдельный файл: [observability.md](observability.md)
> **Принципы и методологии** — перенесены в отдельный файл: [system-design-questions.md](system-design-questions.md)
> **System Design** — перенесён в отдельный файл: [system-design-questions.md](system-design-questions.md)

---

### 2️⃣ Языки программирования

#### Перенесённые разделы

> **Golang** — перенесён в отдельный файл: [go-questions.md](go-questions.md)

---
