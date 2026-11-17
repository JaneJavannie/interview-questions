# Serverless — Lambda, FaaS, Serverless Architecture

Файл содержит вопросы по Serverless-архитектуре: FaaS (AWS Lambda, Google Cloud Functions,
Azure Functions), serverless базы данных, event-driven patterns, cold starts,
тестирование и cost optimization.

Уровни сложности:
- ![Static Badge](https://img.shields.io/badge/Middle-yellow) — Middle разработчик
- ![Static Badge](https://img.shields.io/badge/Senior-red) — Senior разработчик

---

# Оглавление

1. [Serverless — основы](#1-serverless--основы)
2. [Azure Functions и Google Cloud Functions](#2-azure-functions-и-google-cloud-functions)
3. [Serverless patterns](#3-serverless-patterns)
4. [AWS Lambda](#4-aws-lambda)
5. [Serverless базы данных](#5-serverless-базы-данных)
6. [Serverless-вызовы и ограничения](#6-serverless-вызовы-и-ограничения)
7. [Serverless vs Containers](#7-serverless-vs-containers)
8. [Cost optimization и best practices](#8-cost-optimization-и-best-practices)

---

## 1. Serverless — основы

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Что такое Serverless? Это действительно «без серверов»? ]

<details>
<summary>Ответ</summary>

* **Serverless** — модель облачных вычислений, где провайдер управляет инфраструктурой
  (серверами, ОС, масштабированием), а разработчик пишет только бизнес-логику.
* Серверы существуют — просто их обслуживает провайдер. Разработчик не видит и не настраивает ВМ.
* **Ключевые характеристики:**
  * **No server management** — нет нужды патчить ОС, настраивать LB, следить за дисками.
  * **Auto-scaling** — масштабирование от 0 до тысяч запросов автоматически.
  * **Pay-per-use** — оплата за фактическое время выполнения (мс), а не за простаивающие ВМ.
  * **Event-driven** — функции запускаются событиями (HTTP-запрос, сообщение в очереди, таймер).
* **Два основных типа:**
  1. **FaaS** (Function-as-a-Service) — AWS Lambda, Google Cloud Functions, Azure Functions.
  2. **BaaS** (Backend-as-a-Service) — DynamoDB, Firebase, Cognito, AppSync (управляемые сервисы).
* **Serverless ≠ managed services.** Managed RDS — это не serverless (нужно выбирать instance size).
  Aurora Serverless — serverless (auto-scaling, pay-per-use).

</details>

---

### Вопрос №2: [ FaaS vs PaaS — в чём разница? ]

<details>
<summary>Ответ</summary>

| | FaaS (Lambda) | PaaS (Heroku, App Engine) |
|--|---------------|---------------------------|
| **Масштабирование** | Auto, 0 → N | Auto, но обычно 1 → N (минимум 1 instance) |
| **Оплата** | За мс выполнения | За час работы instance |
| **Cold start** | Да (при scale-from-zero) | Нет (instance всегда warm) |
| **State** | Stateless | Может быть stateful |
| **Timeout** | Жёсткий (15 мин для Lambda) | Нет (или настраиваемый) |
| **Deployment unit** | Функция | Приложение целиком |
| **Use case** | Event-driven, short tasks | Web app, long-running |

* FaaS лучше для event-driven, неравномерной нагрузки, short-lived задач.
* PaaS лучше для долгоживущих процессов, WebSocket, постоянной нагрузки.

</details>

---

### Вопрос №3: [ Преимущества и недостатки Serverless? ]

<details>
<summary>Ответ</summary>

**Преимущества:**
* **No ops** — нет нужды управлять серверами, ОС, патчами.
* **Auto-scaling** — автоматическое масштабирование под нагрузку.
* **Pay-per-use** — оплата только за реальное использование (нет idle-затрат).
* **Time to market** — быстрый деплой, фокус на бизнес-логике.
* **Built-in HA** — провайдер обеспечивает отказоустойчивость.

**Недостатки:**
* **Cold starts** — задержка при первом вызове (сотни мс до секунд).
* **Vendor lock-in** — каждая платформа имеет свой API, формат событий.
* **Timeouts** — максимальное время выполнения (Lambda — 15 мин).
* **Statelessness** — состояние нужно хранить во внешних сервисах (БД, Redis).
* **Debugging** — сложнее локально отлаживать и трассировать.
* **Cost at scale** — при постоянной высокой нагрузке может быть дороже ВМ.
* **Limited customization** — нельзя установить arbitrary system packages (в Lambda).

</details>

---

### Вопрос №4: [ Что такое event-driven архитектура в контексте Serverless? ]

<details>
<summary>Ответ</summary>

* **Event-driven** — функции запускаются событиями, а не непрерывно работают и ждут запросы.
* **Источники событий (triggers):**
  * **HTTP** — API Gateway → Lambda.
  * **Queue** — SQS → Lambda, Pub/Sub → Cloud Functions.
  * **Stream** — DynamoDB Streams, Kinesis, EventBridge.
  * **Storage** — S3 event (файл загружен), Cloud Storage.
  * **Timer** — EventBridge Scheduler, Cloud Scheduler (cron).
  * **DB** — DynamoDB Streams (на изменение записи).
* **Паттерны:**
  * **Event routing** — EventBridge маршрутизирует события в разные функции.
  * **Fan-out** — одно событие → несколько функций параллельно.
  * **Saga** — цепочка функций для распределённых транзакций.
  * **Queue-based** — SQS между продюсером и Lambda для буферизации.
* **Важно:** функции должны быть **idempotent**, т.к. события могут доставляться дублированно
  (at-least-once delivery).

</details>

---

### Вопрос №5: [ Что такое BaaS? Примеры? ]

<details>
<summary>Ответ</summary>

* **BaaS** (Backend-as-a-Service) — управляемые сервисы, заменяющие части бэкенда.
* **Примеры:**
  * **Auth:** AWS Cognito, Firebase Auth, Auth0 — регистрация, логин, JWT.
  * **DB:** DynamoDB, Firestore, Aurora Serverless — БД без управления серверами.
  * **Storage:** S3, Cloud Storage — файловое хранилище.
  * **API:** AppSync (GraphQL), API Gateway — REST/GraphQL endpoint.
  * **Real-time:** WebSocket API Gateway, Firebase Realtime Database.
  * **Notifications:** SNS, Firebase Cloud Messaging.
  * **Analytics:** Firebase Analytics, AWS Pinpoint.
* BaaS позволяет строить full-stack приложения почти без серверного кода.

</details>

---

</details>

---

## 2. Azure Functions и Google Cloud Functions

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Azure Functions — особенности? ]

<details>
<summary>Ответ</summary>

* **Azure Functions** — FaaS от Microsoft Azure.
* **Triggers и Bindings:**
  * **Trigger** — что запускает функцию (HTTP, Timer, Queue, Blob, Cosmos DB, Event Hub).
  * **Binding** — декларативное подключение к ресурсам (input/output) без написания кода.
    ```csharp
    [FunctionName("HttpExample")]
    public static async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")] HttpRequest req,
        [CosmosDB("mydb", "mycol", ConnectionStringSetting = "CosmosDB")] IAsyncCollector<dynamic> documents)
    ```
  * Bindings упрощают код, но уменьшают переносимость.
* **Hosting plans:**
  * **Consumption** — serverless, pay-per-use, auto-scaling, cold starts.
  * **Premium** — pre-warmed instances, no cold start, VNet integration.
  * **Dedicated (App Service)** — выделенные ВМ, для постоянной нагрузки.
* **Durable Functions** — расширение для stateful workflows (orchestrator + activity functions):
  * Function chaining, fan-out/fan-in, async HTTP APIs, monitoring.
  * Аналог Step Functions в AWS.
* **Languages:** C#, JavaScript, Python, Java, PowerShell.

</details>

---

### Вопрос №2: [ Google Cloud Functions — особенности? ]

<details>
<summary>Ответ</summary>

* **Google Cloud Functions** — FaaS от Google Cloud.
* **Два поколения:**
  * **1st gen** — основано на Google App Engine infrastructure.
  * **2nd gen** — основано на **Cloud Run** (масштабирование, concurrency до 1000).
* **Triggers:**
  * HTTP — HTTP endpoint.
  * Cloud Pub/Sub — сообщение в топике.
  * Cloud Storage — событие (upload/delete).
  * Eventarc — события из множества Google сервисов.
  * Cloud Tasks — очередь задач.
* **Особенности 2nd gen:**
  * **Concurrency** — один instance обрабатывает до 1000 одновременных запросов
    (1st gen — 1 запрос на instance).
  * Больше memory (до 8 ГБ), больше timeout (до 60 мин).
  * Background на Cloud Run — можно использовать любой контейнер.
* **Languages:** Node.js, Python, Go, Java, C#, Ruby, PHP.

</details>

---

### Вопрос №3: [ Cloud Run vs Cloud Functions — что выбрать? ]

<details>
<summary>Ответ</summary>

| | Cloud Run | Cloud Functions |
|--|-----------|-----------------|
| **Deployment unit** | Любой контейнер | Функция (код) |
| **Concurrency** | До 1000 на instance | 1 (1st gen), до 1000 (2nd gen) |
* **Use Cloud Run если:**
  * Нужен custom runtime или system packages.
  * WebSockets / streaming.
  * gRPC.
  * Долгоживущие процессы (до 60 мин).
* **Use Cloud Functions если:**
  * Простой event-driven trigger.
  * Быстрый деплой без Docker.
  * Минимум boilerplate.
* **Trend:** Google рекомендует Cloud Run для большинства новых use cases.

</details>

---

### Вопрос №4: [ Сравнение Lambda, Azure Functions, Cloud Functions ]

<details>
<summary>Ответ</summary>

| | AWS Lambda | Azure Functions | GCF (2nd gen) |
|--|------------|-----------------|---------------|
| **Max timeout** | 15 мин | 10 мин (Consumption, default 5 мин) | 60 мин |
| **Max memory** | 10 ГБ | 1.5 ГБ (Consumption) | 8 ГБ |
| **Concurrency per instance** | 1 | 1 (Consumption) | До 1000 |
| **Cold start** | Да | Да | Да |
| **Provisioned (warm)** | Provisioned Concurrency | Premium plan | Min instances |
| **Container support** | Да (container image) | Да (Premium/Dedicated) | Да (Cloud Run) |
| **Orchestration** | Step Functions | Durable Functions | Cloud Workflows |
| **Pricing** | За мс × memory | За сек × memory | За 100 мс × memory |

* Все три поддерживают container deployment, что уменьшает vendor lock-in.
* Lambda — самая зрелая, больше ecosystem и integrations.
* Cloud Run/GCF 2nd gen — лучшая concurrency (до 1000 на instance).

</details>

---

</details>

---

## 3. Serverless patterns

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Паттерн «Serverless Microservices» — как строить? ]

<details>
<summary>Ответ</summary>

* **Архитектура:**
  ```
  Client → API Gateway → Lambda (Service A) → DynamoDB
                       → Lambda (Service B) → SQS → Lambda (Worker)
                       → Lambda (Service C) → S3
  ```
* **Принципы:**
  * Каждый микросервис — отдельная Lambda + своя БД (DynamoDB table).
  * Связь между сервисами — через events (EventBridge, SNS/SQS) или HTTP (API Gateway).
  * **Single responsibility** — одна Lambda = одна задача.
* **Плюсы:**
  * Независельный деплой и масштабирование.
  * Pay-per-use.
  * No servers to manage.
* **Минусы:**
  * Сложность отладки distributed system.
  * Latency между сервисами (cold starts, network).
  * Vendor lock-in.

</details>

---

### Вопрос №2: [ Паттерн «Fan-out / Fan-in» в Serverless ]

<details>
<summary>Ответ</summary>

* **Fan-out** — одно событие запускает множество параллельных функций.
* **Fan-in** — результаты собираются и агрегируются.
* **Реализация:**
  ```
  S3 upload → Lambda (trigger) → SQS (1 message per file chunk)
                                 → N Lambda workers (parallel processing)
                                 → Lambda aggregator (fan-in)
  ```
* **AWS:**
  * **SNS → SQS → Lambda** — SNS fan-out на несколько SQS очередей.
  * **Step Functions Parallel state** — параллельное выполнение веток + агрегация.
  * **EventBridge** — одно событие → несколько targets (Lambda, SQS, SNS).
* **Use cases:**
  * Обработка множества файлов (batch processing).
  * Распределённые вычисления (map-reduce).
  * Параллельные HTTP запросы к внешним API.

</details>

  -----

### Вопрос №3: [ Паттерн «Saga» в Serverless. Как управлять распределёнными транзакциями? ]

<details>
<summary>Ответ</summary>

* **Saga** — последовательность локальных транзакций, каждая с compensating action (откат).
* **В Serverless реализуется через:**
  * **Step Functions (AWS)** — State Machine с шагами и компенсацией.
  * **Durable Functions (Azure)** — orchestrator pattern.
  * **EventBridge + Lambda** — event-driven chain.
* **Пример (заказ товара):**
  ```
  1. Create Order → 2. Reserve Inventory → 3. Charge Payment → 4. Confirm Order
                     ↗ (fail) Compensate: Release Inventory
                                     ↗ (fail) Compensate: Refund Payment
  ```
* **Step Functions:**
  ```json
  {
    "StartAt": "CreateOrder",
    "States": {
      "CreateOrder": { "Type": "Task", "Next": "ReserveInventory", "Catch": [...] },
      "ReserveInventory": { "Type": "Task", "Next": "ChargePayment", "Catch": [{ "Next": "CompensateOrder" }] },
      "ChargePayment": { "Type": "Task", "Next": "ConfirmOrder", "Catch": [{ "Next": "ReleaseInventory" }] },
      "ReleaseInventory": { "Type": "Task", "Next": "CompensateOrder" },
      "CompensateOrder": { "Type": "Task", "Next": "FailOrder" },
      "ConfirmOrder": { "Type": "Succeed" },
      "FailOrder": { "Type": "Fail" }
    }
  }
  ```
* **Важно:** каждый шаг должен быть **idempotent** (повторное выполнение не ломает).

</details>

---

### Вопрос №4: [ Паттерн «API Gateway as facade» ]

<details>
<summary>Ответ</summary>

* API Gateway выступает единой точкой входа, маршрутизируя запросы в разные Lambda.
* **Плюсы:**
  * Единый domain, TLS, auth.
  * Throttling и rate limiting на gateway уровне.
  * Request/response transformation (mapping templates).
  * Caching на gateway уровне (TTL-based).
  * Versioning (v1, v2).
* **Архитектура:**
  ```
  Client → API Gateway → /users    → Lambda (Users service)
                       → /orders   → Lambda (Orders service)
                       → /payments → Lambda (Payments service)
  ```
* **Anti-pattern:** «Fat Lambda» — одна Lambda обрабатывает все маршруты. Лучше —
  отдельная Lambda на resource.

</details>

---

### Вопрос №5: [ Паттерн «Queue-based load leveling» в Serverless ]

<details>
<summary>Ответ</summary>

* **Проблема:** резкий всплеск запросов → Lambda concurrency исчерпан → ошибки.
* **Решение:** поставить SQS между client и Lambda.
  ```
  Producer → SQS (buffer) → Lambda (consumes at controlled rate)
  ```
* **Преимущества:**
  * **Buffer** — SQS накапливает сообщения, Lambda обрабатывает в своём темпе.
  * **Retry** — неудачные сообщения возвращаются в очередь (visibility timeout).
  * **DLQ** — окончательно неудачные сообщения → Dead Letter Queue.
  * **Batching** — Lambda может читать до 10 сообщений за раз (batch processing).
* **Lambda concurrency:** можно установить `Reserved Concurrency` для consumer Lambda,
  чтобы ограничить скорость обработки и не перегрузить downstream (БД).
* **Visibility timeout** должен быть больше, чем Lambda timeout.

</details>

---

### Вопрос №6: [ Паттерн «Circuit Breaker» в Serverless. Как реализовать? ]

<details>
<summary>Ответ</summary>

* **Проблема:** Lambda вызывает внешний API, который упал. Retry-и нагружают downstream сервис.
* **Circuit Breaker** — разрывает цепь при превышении порога ошибок.
* **Реализация в Serverless:**
  * **DynamoDB / ElastiCache** — хранить счётчик ошибок и состояние (closed/open/half-open).
  * ```
    Lambda → read state and open_until (DynamoDB)
          → if OPEN and now < open_until: return fallback
          → if OPEN and now >= open_until: conditionally admit HALF_OPEN probe
          → if CLOSED: call API, on error increment counter
          → if counter > threshold: set OPEN, open_until = now + cooldown
    ```
  * `open_until` проверяется явно при чтении. Conditional update по состоянию/версии допускает
    ограниченное число half-open проб (например, одну); остальные вызовы получают fallback.
    У пробы есть токен и deadline: только её владелец завершает переход в CLOSED при успехе
    или OPEN с новым `open_until` при ошибке; зависшую пробу можно условно заменить после deadline.
  * **DynamoDB TTL** — только необязательная очистка: удаление может задержаться на несколько дней,
    это не таймер перехода в half-open. Живую запись состояния не удалять по `open_until`.
* **Альтернатива:** AWS SDK built-in retry + exponential backoff.
* **Step Functions** — Retry/Catch с ErrorEquals и backoff.

</details>

---

</details>

---

## 4. AWS Lambda

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Как устроена AWS Lambda под капотом? ]

<details>
<summary>Ответ</summary>

* **Execution environment** — изолированный контейнер (Firecracker microVM), в котором
  выполняется функция.
* **Lifecycle:**
  1. **Init** — загрузка runtime, загрузка кода, выполнение `init` кода (global variables).
  2. **Invoke** — обработка запроса handler-функцией.
  3. **Reuse** — environment переиспользуется для следующих вызовов (warm).
  4. **Shutdown** — после периода неактивности environment уничтожается (cold).
* **Cold start** — первый вызов после создания environment: скачивание кода, init runtime,
  init кода. Занимает 100 мс — несколько секунд.
* **Warm start** — повторный вызов в существующем environment. Быстро (мс).
* **Provisioned Concurrency** — предзаготовленные warm environments (нет cold start).
* **Concurrency** — количество одновременных вызовов. По умолчанию 1000 на аккаунт (soft limit).
* **Execution role** — IAM role с правами на другие AWS сервисы.

</details>

---

### Вопрос №2: [ Что такое cold start? Как его минимизировать? ]

<details>
<summary>Ответ</summary>

* **Cold start** — задержка при первом вызове функции (или после idle-периода).
* **Причины задержки:**
  1. Создание microVM (Firecracker) — ~10-20 мс.
  2. Загрузка кода и слоёв из S3.
  3. Init runtime (Node.js ~100 мс, Java ~1-3 с).
  4. Выполнение `init` кода (global variables, connections).
* **Факторы:**
  * **Runtime:** Java/C# — самые долгие. Go, Rust — быстрые. Python, Node.js — средние.
  * **Package size:** больше кода = дольше загрузка.
  * **VPC:** Lambda в VPC имеет дополнительную задержку (ENI creation — раньше, сейчас улучшено).
* **Минимизация:**
  * **Provisioned Concurrency** — держать N warm environments. Платно, но нет cold start.
  * **Уменьшить package size** — убрать неиспользуемые зависимости, tree-shaking.
  * **Lazy initialization** — инициализировать тяжёлые объекты при первом вызове, не в global scope.
  * **Lightweight runtime** — Go/Rust вместо Java.
  * **Lambda Layers** — вынести тяжёлые зависимости в layer (кэшируется).
  * **SnapStart** (Java) — Lambda создаёт snapshot после init, восстанавливает из него (ускоряет x10).

</details>

---

### Вопрос №3: [ Lambda Layers — что это и зачем? ]

<details>
<summary>Ответ</summary>

* **Lambda Layer** — отдельный ZIP-архив с зависимостями (библиотеки, custom runtime),
  который можно подключить к нескольким функциям.
* **Преимущества:**
  * **Reuse** — общий код/зависимости для нескольких функций.
  * **Уменьшение deployment package** — основная функция меньше.
  * **Разделение ответственности** — deps обновляются отдельно от кода.
  * **Кэширование** — layers кэшируются на execution environment (быстрее cold start).
* **Ограничения:**
  * До 5 layers на функцию.
  * Общий размер (code + layers) — 250 МБ (unzipped).
* **Примеры использования:**
  * Общий SDK (AWS SDK, DB driver).
  * Custom runtime (e.g., Rust runtime).
  * Шаблонный код (logging, tracing, error handling).

</details>

---

### Вопрос №4: [ Ограничения AWS Lambda? ]

<details>
<summary>Ответ</summary>

| Параметр | Лимит |
|----------|-------|
| **Memory** | 128 МБ — 10 ГБ |
| **Timeout** | 1 сек — 15 мин |
| **Deployment package (zipped)** | 50 МБ |
| **Deployment package (unzipped)** | 250 МБ |
| **/tmp storage** | 512 МБ — 10 ГБ |
| **Concurrency** | 1000 (soft limit, можно увеличить) |
| **Layers** | До 5 |
| **Environment variables** | 4 КБ |
| **Payload (sync)** | 6 МБ |
| **Payload (async)** | 1 МБ (ранее — 256 КБ) |

* Memory и CPU связаны: больше memory = больше vCPU (пропорционально).
* При 1769 МБ — 1 full vCPU. При 10 ГБ — ~6 vCPU.
* Для CPU-bound задач имеет смысл увеличить memory (даже если память не нужна).

</details>

---

### Вопрос №5: [ Как Lambda работает с VPC? ]

<details>
<summary>Ответ</summary>

* **По умолчанию** Lambda НЕ в VPC — имеет доступ к интернету и публичным AWS API.
* **Lambda in VPC** — для доступа к private ресурсам (RDS, ElastiCache, internal API).
* **Как работает:**
  * Lambda создаёт **ENI** (Elastic Network Interface) в указанных subnets.
  * Трафик идёт через ENI → VPC → ресурс.
  * **Security Groups** применяются к ENI.
* **Проблема:** Lambda в VPC **не имеет интернет-доступа** (NAT Gateway нужен для исходящего).
* **Решения:**
  * **NAT Gateway** в public subnet + route table → интернет (платно).
  * **VPC Endpoints** (PrivateLink) — для AWS сервисов без интернета (S3, DynamoDB).
  * **Split architecture** — одна Lambda в VPC (для БД), другая вне VPC (для внешних API).
* **Hyperplane ENI** (с 2019) — Lambda переиспользует ENI, создание намного быстрее
  (раньше cold start в VPC был очень долгим).

</details>

---

### Вопрос №6: [ Lambda + API Gateway — как это работает? ]

<details>
<summary>Ответ</summary>

* **API Gateway** — управляемый HTTP endpoint, который маршрутизирует запросы в Lambda.
* **Поток:**
  ```
  Client → API Gateway (auth, throttling, validation) → Lambda → Response
  ```
* **Типы API Gateway:**
  * **REST API** — полный набор функций (auth, WAF, request validation, mapping templates).
  * **HTTP API** — упрощённый, дешевле, быстрее (но меньше функций).
  * **WebSocket API** — для real-time bidirectional communication.
* **Integration types:**
  * **Lambda Proxy** — весь HTTP-запрос (headers, body, path) передаётся в Lambda как event.
    Lambda возвращает `{statusCode, headers, body}`.
  * **Lambda (non-proxy)** — mapping templates для трансформации запроса/ответа.
* **Throttling:** API Gateway может ограничивать RPS (rate limiting) перед Lambda.
* **Auth:** Lambda authorizer, Cognito, IAM, custom authorizer.

</details>

---

### Вопрос №7: [ Lambda destinations — что это и зачем? ]

<details>
<summary>Ответ</summary>

* **Lambda Destinations** — механизм маршрутизации результата async-вызова в другой сервис
  (без ручной обработки в коде).
* **Для async invocation** (EventBridge, S3, SNS):
  * **On success** → SQS / SNS / Lambda / EventBridge.
  * **On failure** → SQS / SNS / Lambda / EventBridge.
* **Зачем:**
  * Раньше async-ошибки терялись (только DLQ). Теперь — точная маршрутизация.
  * Chain-of-responsibility без оркестратора (Step Functions).
  * Dead Letter Queue для неудачных вызовов.
* **DLQ vs Destinations:**
  * DLQ — только для failed events (старый механизм).
  * Destinations — для success AND failure, более гибкий.

</details>

---

</details>

---

## 5. Serverless базы данных

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ DynamoDB — как устроена? Почему подходит для Serverless? ]

<details>
<summary>Ответ</summary>

* **DynamoDB** — managed NoSQL (key-value/document) БД от AWS.
* **Архитектура:**
  * **Partition Key (PK)** + опционально **Sort Key (SK)** — составной primary key.
  * Данные автоматически шардятся по партициям по PK (consistent hashing).
  * **RCU / WCU** — Read/Write Capacity Units (provisioned) или **On-Demand** (pay-per-request).
* **Почему Serverless:**
  * Auto-scaling (provisioned) или On-Demand (от 0 до неограниченной нагрузки).
  * Pay-per-use (On-Demand: ~$1.25 per million write requests, ~$0.25 per million read requests).
  * No server management — не нужно настраивать инстансы, репликацию, backup.
  * Single-digit ms latency при любом масштабе.
* **Особенности:**
  * **Global Tables** — multi-region репликация (active-active).
  * **DynamoDB Streams** — event stream при изменении записей → Lambda trigger.
  * **TTL** — автоматическое удаление записей по timestamp.
  * **DAX** — in-memory caching (микросекундная latency).
* **Ограничения:**
  * Item size — до 400 КБ.
  * Query — только по PK (и SK с условиями). Нет arbitrary JOIN.
  * **No flexible queries** — нужно проектировать schema под access patterns (single-table design).

</details>

---

### Вопрос №2: [ Aurora Serverless — чем отличается от обычной Aurora? ]

<details>
<summary>Ответ</summary>

* **Aurora Serverless v2:**
  * Автоматическое масштабирование capacity (ACU — Aurora Capacity Units) от 0.5 до 128 ACU.
  * Совместима с обычной Aurora (MySQL/PostgreSQL).
  * Scale-up/scale-down за секунды (fine-grained).
  * **Может не до конца scale to zero** (минимум 0.5 ACU — всё ещё платно).
* **Aurora Serverless v1 (оригинальная):**
  * Scale to zero (полная остановка при простое).
  * Но cold start при возобновлении (секунды).
  * Не поддерживает многие функции обычной Aurora (replicas, глобальная БД).
* **Когда использовать Serverless:**
  * Непредсказуемая нагрузка (dev/test, редко используемые БД).
  * Multi-tenant apps (каждый tenant = свой cluster).
* **Когда НЕ использовать:**
  * Постоянная высокая нагрузка — дешевле provisioned Aurora.
  * Low-latency требования (scale-up задержка).

</details>

---

### Вопрос №3: [ Serverless БД сравнение: DynamoDB vs Aurora Serverless vs Firestore ]

<details>
<summary>Ответ</summary>

| | DynamoDB | Aurora Serverless v2 | Firestore |
|--|----------|---------------------|-----------|
| **Тип** | Key-value / Document | Relational (SQL) | Document |
| **Latency** | Single-digit ms | ms | ms |
| **Scale to zero** | Да (On-Demand, $0 при простое) | Нет (мин 0.5 ACU) | Да |
| **Query flexibility** | Low (PK/SK) | High (SQL) | Medium (queries, indexes) |
| **Pricing** | Per request / provisioned | Per ACU-hour + storage | Per read/write + storage |
| **Best for** | High-throughput, simple access | SQL apps, mixed load | Mobile/web apps, real-time |
| **Vendor** | AWS | AWS | Google Cloud |

* **DynamoDB** — лучшая для serverless-нативных приложений (Lambda).
* **Aurora Serverless** — миграция с SQL без управления серверами.
* **Firestore** — real-time, mobile/web (Firebase ecosystem).

</details>

---

### Вопрос №4: [ Single Table Design в DynamoDB — что это? ]

<details>
<summary>Ответ</summary>

* **Single Table Design (STD)** — все сущности (entities) в одной DynamoDB таблице.
* **Зачем:** DynamoDB поддерживает query только по PK/SK. STD проектируется так, чтобы
  все access patterns были эффективны.
* **Пример:**
  ```
  PK                SK                Data
  USER#123          METADATA          {"name": "Alice", ...}
  USER#123          ORDER#456         {"amount": 100, ...}
  USER#123          ORDER#789         {"amount": 200, ...}
  ORDER#456         METADATA          {"status": "shipped", ...}
  ```
* **Query:** `PK = USER#123` → получить пользователя и все его заказы за один query.
* **Плюсы:**
  * Минимум запросов (1 query = все данные для access pattern).
  * Эффективность (нет JOIN).
* **Минусы:**
  * Сложность проектирования (нужно знать все access patterns заранее).
  * Сложность изменения schema.
  * Переход от реляционного мышления — сложный.

</details>

---

</details>

---

## 6. Serverless-вызовы и ограничения

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Как тестировать Serverless приложения? ]

<details>
<summary>Ответ</summary>

* **Уровни тестирования:**
  1. **Unit tests** — тестирование handler-функции с mock event.
     ```python
     def test_handler():
         event = {"httpMethod": "GET", "path": "/users/123"}
         context = MagicMock()
         result = handler(event, context)
         assert result["statusCode"] == 200
     ```
  2. **Integration tests** — с реальными AWS сервисами (LocalStack, DynamoDB Local).
     * **LocalStack** — эмулятор AWS сервисов локально.
     * **DynamoDB Local** — локальная DynamoDB (Docker).
  3. **End-to-end** — деплой в test environment и реальные вызовы.
* **Инструменты:**
  * **AWS SAM CLI** — `sam local invoke`, `sam local start-api` (локальный запуск Lambda).
  * **Serverless Framework** — `serverless invoke local`.
  * **LocalStack** — полноценная эмуляция AWS.
* **Сложности:**
  * Mock-и AWS SDK (boto3) — `moto` library для Python.
  * Event format зависит от trigger (API Gateway event ≠ SQS event).
  * State между вызовами (Lambda stateless → state в БД → сложнее тестировать).

</details>

---

### Вопрос №2: [ Как дебажить Serverless приложения? ]

<details>
<summary>Ответ</summary>

* **Локальная отладка:**
  * **AWS SAM CLI** — `sam local invoke --debug-port 5858` + IDE debugger (VS Code, PyCharm).
  * **Serverless Offline** — плагин для Serverless Framework.
* **Cloud отладка:**
  * **CloudWatch Logs** — Lambda автоматически пишет логи в CloudWatch.
    * `console.log` / `print` → CloudWatch Log Stream.
  * **AWS X-Ray** — distributed tracing (видно всю цепочку Lambda → DynamoDB → S3).
  * **Lambda Insights** — метрики (memory, CPU, duration) в CloudWatch.
* **Структурированные логи:**
  * JSON-формат логов для парсинга в CloudWatch Logs Insights.
  * ```json
    {"level": "error", "message": "DB connection failed", "requestId": "abc-123", "timestamp": "2024-01-01T00:00:00Z"}
    ```
* **Совет:** коррелировать логи по `requestId` (Lambda генерирует для каждого вызова).

</details>

---

### Вопрос №3: [ Vendor lock-in в Serverless — как бороться? ]

<details>
<summary>Ответ</summary>

* **Проблема:** код зависит от конкретного провайдера (event format, SDK, triggers).
* **Стратегии:**
  1. **Hexagonal Architecture (Ports & Adapters):**
     * Бизнес-логика не зависит от AWS SDK.
     * Adapters для конкретных сервисов (S3 adapter, DynamoDB adapter).
     * Замена adapter = смена провайдера.
  2. **Container deployment:**
     * Lambda поддерживает container images → можно деплоить обычный web app.
     * Cloud Run — нативно контейнеры.
     * Перенос между Lambda, Cloud Run, Azure Container Apps — минимальные изменения.
  3. **Framework abstraction:**
     * **Serverless Framework** / **AWS SAM** — абстракция над разными провайдерами
       (но не полная).
     * **OpenFaaS** / **Knative** — open-source FaaS на Kubernetes.
  4. **Event schema externalization:**
     * Event schema вне кода (JSON Schema, Protobuf).
     * Mapper layer для преобразования vendor event → internal event.
* **Реальность:** полный vendor lock-in избежать сложно. Лучше — изолировать его (adapters).

</details>

---

### Вопрос №4: [ Как управлять состоянием (state) в Serverless? ]

<details>
<summary>Ответ</summary>

* Lambda **stateless** — после вызова environment может быть уничтожен. State нужно
  хранить внешне.
* **Опции:**
  * **Session state:** DynamoDB, Redis (ElastiCache), JWT (client-side).
  * **File state:** S3 (для больших данных), EFS (mounted в Lambda).
  * **Workflow state:** Step Functions (state machine для долгоживущих процессов).
  * **Cache:** Lambda `/tmp` (512 МБ — 10 ГБ) — работает только в warm environment.
    Не гарантируется между вызовами.
* **Best practice:**
  * Connection pooling — инициализировать DB connections в global scope (переиспользуется
    в warm calls).
  * Не хранить состояние между вызовами в памяти (не гарантируется).
  * `/tmp` — для кэша, который можно пересоздать.

</details>

---

### Вопрос №5: [ Idempotency в Serverless — зачем и как? ]

<details>
<summary>Ответ</summary>

* **Зачем:** Lambda (и другие FaaS) могут вызвать функцию повторно:
  * At-least-once delivery для async events (SQS, SNS, EventBridge).
  * Lambda retry при ошибке (2 retry для async).
  * Client retry (timeout → повторный запрос).
* **Idempotency** — повторный вызов с тем же input даёт тот же результат без побочных эффектов.
* **Реализация:**
  1. **Idempotency key** — стабильный ID бизнес-операции (`requestId`, `eventId`), один для всех retry,
     не новый ID каждого запуска Lambda.
  2. **Локальная бизнес-операция:** inbox/dedup с уникальным ключом и бизнес-изменение — в одной
     транзакции БД (для DynamoDB — условная запись маркера и изменение данных через `TransactWriteItems`).
     Ack/успех обработки сообщения — только после commit. Отдельный маркер до эффекта теряет обработку
     при падении, после эффекта — допускает дубли; наивный get → work → put ещё и допускает гонку.
  3. **AWS Lambda Powertools:** минимальный пример для прямого Lambda event с полями
     `requestId`, `orderId`, `amount`, `currency` (не обёртка API Gateway/SQS). Он только формирует
     данные для платёжного адаптера, **не выполняет списание**:
     ```python
     from aws_lambda_powertools.utilities.idempotency import (
         DynamoDBPersistenceLayer,
         IdempotencyConfig,
         idempotent,
     )

     persistence = DynamoDBPersistenceLayer(table_name="idem_table")
     config = IdempotencyConfig(
         event_key_jmespath="requestId",
         payload_validation_jmespath="[orderId, amount, currency]",
         raise_on_no_idempotency_key=True,
         expires_after_seconds=86400,
     )

     @idempotent(config=config, persistence_store=persistence)
     def handler(event, context):
         return {
             "provider_idempotency_key": "payment:" + event["requestId"],
             "orderId": event["orderId"],
             "amount": event["amount"],
             "currency": event["currency"],
         }
     ```
     Нужны установленный Powertools, таблица `idem_table` с partition key `id` (String)
     и права Lambda на GetItem/PutItem/UpdateItem/DeleteItem. Входные поля валидируются на границе API;
     `payload_validation_jmespath` проверяет, что тот же ключ не используется с другим платёжным payload.
     Реальный платёжный адаптер должен передавать `provider_idempotency_key` в механизм идемпотентности
     выбранного провайдера при **каждой** попытке списания, не генерировать новый ключ при retry.
  4. Powertools условно создаёт `INPROGRESS`, блокируя конкурентный вызов с тем же ключом;
     активный дубликат получает `IdempotencyAlreadyInProgressError` и может быть повторён позже.
     После успеха сохраняет `COMPLETE` и результат. `@idempotent` использует Lambda context для
     `in_progress_expiration`: после истечения незавершённую попытку можно повторить.
* **Граница гарантии:** `INPROGRESS`/`COMPLETE` не делают запись результата и внешний эффект атомарными.
  Если провайдер списал деньги, а Lambda упала до сохранения результата, повтор снова дойдёт до провайдера.
  Нужны его idempotency key и reconciliation (сверка статуса платежа) при неизвестном исходе;
  Powertools сам по себе не гарантирует exactly-once списание и не заменяет транзакцию бизнес-данных.
* **Срок хранения:** окно дедупликации в БД и у провайдера должно покрывать допустимые retry;
  после его истечения старый ключ нельзя слепо использовать для нового списания — сначала сверка.
  Powertools проверяет expiration явно. DynamoDB TTL на `expiration` — только отложенная очистка,
  а не точный таймер и не гарантия идемпотентности.

</details>

---

### Вопрос №6: [ Как обрабатывать long-running задачи в Serverless? ]

<details>
<summary>Ответ</summary>

* **Проблема:** Lambda max timeout — 15 мин. Что если задача занимает дольше?
* **Решения:**
  1. **Step Functions:**
     * Разбить задачу на шаги (каждый < 15 мин).
     * State Machine хранит прогресс.
     * Поддерживает wait states (Wait up to 1 год).
  2. **Chunking:**
     * Разбить данные на чанки → SQS → Lambda workers (parallel).
     * Каждый чанк обрабатывается < 15 мин.
  3. **Fargate / ECS:**
     * Для действительно долгих задач — использовать контейнеры (нет timeout).
  4. **AWS Batch:**
     * Для batch processing без timeout ограничений.
  5. **EventBridge Scheduler:**
     * Периодический запуск Lambda (cron) — для прогрессирующей обработки.
  6. **DynamoDB Streams + Lambda:**
     * Обрабатывать данные по мере их появления (stream processing).

</details>

---

</details>

---

## 7. Serverless vs Containers

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Когда выбрать Serverless, а когда — Containers (K8s/ECS)? ]

<details>
<summary>Ответ</summary>

| Критерий | Serverless (Lambda) | Containers (K8s/ECS/Fargate) |
|----------|---------------------|------------------------------|
| **Load pattern** | Спорадическая, неравномерная | Постоянная, предсказуемая |
| **Cold start** | Проблема (мс — сек) | Нет (pods всегда warm) |
| **Max duration** | 15 мин | Без ограничений |
| **Scaling** | Авто (0 → N) | Ручное/HPA (1 → N) |
| **Cost (low traffic)** | Очень дешёвый | Платишь за idle nodes |
| **Cost (high traffic)** | Дорогой | Дешевле |
| **Control** | Ограниченный | Полный (ОС, packages) |
| **Stateful** | Нет (stateless only) | Да (volumes, stateful sets) |
| **WebSockets / streaming** | Сложно (API Gateway WS) | Нативно |
| **Vendor lock-in** | Высокий | Низкий (container стандарт) |

* **Serverless** — для event-driven, прототипов, неравномерной нагрузки, short-lived задач.
* **Containers** — для постоянной нагрузки, сложных приложений, full control, long-running.

</details>

---

### Вопрос №2: [ Можно ли запустить Lambda в контейнере? ]

<details>
<summary>Ответ</summary>

* **Да.** Lambda поддерживает **Container Image deployment** (с 2020).
* **Как это работает:**
  * Запаковать приложение в Docker image (до 10 ГБ).
  * Зарегистрировать image в ECR.
  * Создать Lambda function из image.
  * Lambda runtime interface (RIE) — обеспечивает совместимость с Lambda runtime API.
* **Преимущества:**
  * Custom runtime и dependencies.
  * Больший размер (до 10 ГБ vs 250 МБ для ZIP).
  * Переиспользование Docker-инфраструктуры (build, test, registry).
  * Упрощённая миграция контейнерных приложений в Serverless.
* **Ограничения:**
  * Те же timeout/memory ограничения Lambda.
  * Cold start может быть дольше (образ больше).
  * Container должен реализовывать Lambda Runtime API.

</details>

---

### Вопрос №3: [ Knative и OpenFaaS — что это? Зачем? ]

<details>
<summary>Ответ</summary>

* **Knative** — open-source platform для Serverless на Kubernetes.
  * **Serving** — auto-scaling (scale-to-zero), traffic management, routing.
  * **Eventing** — event delivery (CloudEvents), triggers, sources.
  * Совместим с Cloud Run (Google Cloud Run built on Knative).
* **OpenFaaS** — open-source FaaS framework.
  * Запуск на Kubernetes или Swarm.
  * Поддержка любого языка (через контейнеры).
  * `faas-cli` — CLI для деплоя функций.
  * Queue worker ( NATS ) для async вызовов.
* **Зачем:**
  * **Avoid vendor lock-in** — не привязан к конкретному облаку.
  * **On-premise Serverless** — запуск в своём дата-центре.
  * **Multi-cloud** — одинаковая платформа в разных облаках.
* **Минусы:**
  * Нужно управлять Kubernetes (не "no-ops").
  * Меньше integrations чем у managed FaaS.

</details>

---

### Вопрос №4: [ Fargate vs Lambda — когда что? ]

<details>
<summary>Ответ</summary>

* **AWS Fargate** — serverless containers (ECS/EKS без управления nodes).
* **Lambda** — serverless functions.

| | Lambda | Fargate |
|--|--------|---------|
| **Deployment unit** | Функция | Контейнер |
| **Scaling** | 0 → N (event-driven) | 1 → N (by CPU/memory) |
| **Duration** | Max 15 мин | Без ограничений |
| **State** | Stateless | Может быть stateful (EFS) |
| **Cold start** | Да | Нет (если min task > 0) |
| **Pricing** | Per ms + memory | Per sec + CPU/memory |
| **Use case** | Event-driven, short tasks | Web servers, workers, long tasks |

* **Lambda:** event processing, API endpoints с неравномерной нагрузкой, cron jobs.
* **Fargate:** long-running web servers, WebSocket servers, batch jobs > 15 мин, custom
  runtime.

</details>

---

</details>

---

## 8. Cost optimization и best practices

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Как оптимизировать стоимость Serverless? ]

<details>
<summary>Ответ</summary>

* **1. Right-size memory:**
  * Больше memory = больше CPU (пропорционально).
  * Для CPU-bound: больше memory → быстрее выполнение → дешевле (оплата за мс × memory).
  * Для I/O-bound: меньше memory достаточно.
  * **AWS Lambda Power Tuning** — tool для поиска оптимального memory.
* **2. Minimize cold starts:**
  * Cold start = лишние мс выполнения = дороже.
  * Provisioned Concurrency — платно, но нет cold start (для latency-sensitive).
* **3. Use On-Demand vs Provisioned (DynamoDB):**
  * On-Demand — для непредсказуемой нагрузки.
  * Provisioned — для предсказуемой (дешевле при стабильной нагрузке).
* **4. Reduce invocations:**
  * Batch processing (SQS → Lambda batch of 10) вместо 1 message = 1 invocation.
  * Step Functions — объединить мелкие шаги.
* **5. Avoid NAT Gateway:**
  * NAT Gateway — $0.045/GB + $0.045/hour.
  * VPC Endpoints для AWS сервисов (S3, DynamoDB) — дешевле при большом трафике.
* **6. CloudFront caching:**
  * Кэшировать API responses на CDN → меньше вызовов Lambda.
* **7. Duration optimization:**
  * Lazy initialization (не грузить всё в init).
  * Connection reuse (warm calls → тот же DB connection).
  * Async I/O вместо sync (не блокировать).

</details>

---

### Вопрос №2: [ AWS Step Functions — что это и зачем? ]

<details>
<summary>Ответ</summary>

* **Step Functions** — serverless оркестратор (state machine) для Lambda и других сервисов.
* **Возможности:**
  * **Sequence** — шаги выполняются последовательно.
  * **Parallel** — параллельное выполнение веток.
  * **Choice** — условное ветвление (if/else).
  * **Wait** — задержка (до 1 года).
  * **Map** — итерация по массиву (fan-out).
  * **Retry/Catch** — retry с backoff, error handling.
* **Типы:**
  * **Standard** — для long-running (до 1 года), at-least-once, $0.025 per 1000 transitions.
  * **Express** — для high-throughput short-lived (до 5 мин), at-least-once или at-most-once,
    дешевле.
* **Use cases:**
  * Order processing (сага с компенсацией).
  * ETL pipeline.
  * User onboarding (многошаговый).
  * Retry с exponential backoff.
* **Зачем вместо Lambda chain (SQS → Lambda → SQS):**
  * Визуализация workflow.
  * Built-in retry/error handling.
  * State management (не нужно хранить прогресс в БД).
  * Версионирование и audit.

</details>

---

### Вопрос №3: [ Serverless security — основные принципы? ]

<details>
<summary>Ответ</summary>

* **1. Least privilege IAM:**
  * Lambda execution role — только нужные действия на нужных ресурсах.
  * `lambda:InvokeFunction` только для конкретных Lambda.
* **2. Environment variables encryption:**
  * Secrets в env vars шифруются KMS.
  * Лучше — Secrets Manager / Parameter Store ( rotation, audit).
* **3. VPC для приватных ресурсов:**
  * Lambda в VPC для доступа к RDS/ElastiCache.
  * Security Groups — ограничить входящий/исходящий трафик.
* **4. API Gateway authorization:**
  * Lambda authorizer / Cognito / IAM — не оставлять открытый API.
  * WAF для защиты от SQL injection, XSS, DDoS.
* **5. Input validation:**
  * Валидация входных данных в Lambda (schema validation).
  * API Gateway request validation (schema).
* **6. Logging и monitoring:**
  * CloudWatch Logs — все вызовы.
  * CloudTrail — audit API calls.
  * X-Ray — tracing для обнаружения аномалий.
* **7. Code signing:**
  * AWS Signer — подписывать Lambda deployment packages.
  * Запретить деплой неподписанных пакетов.

</details>

---

### Вопрос №4: [ Serverless deployment strategies? ]

<details>
<summary>Ответ</summary>

* **1. All-at-once:**
  * Простая замена старой версии на новую. Рискованно.
* **2. Canary:**
  * Деплой на маленький % трафика (e.g., 5%), проверка, затем 100%.
  * Lambda Aliases + weighted routing:
    ```
    Alias: PROD
      v1 (old): 95%
      v2 (new): 5%
    ```
* **3. Linear:**
  * Постепенное увеличение: 10% → 25% → 50% → 100%.
* **4. Blue/Green:**
  * Две версии (blue = old, green = new). Переключение traffic.
  * Lambda: два aliases, switch API Gateway integration.
* **5. Traffic shifting with rollback:**
  * CloudWatch Alarm → automatic rollback при ошибках.
  * CodeDeploy + Lambda — automated canary deployment.
* **Инструменты:**
  * AWS SAM + CodeDeploy.
  * Serverless Framework.
  * Terraform (lambda alias + provisioned config).

</details>

---

### Вопрос №5: [ Мониторинг Serverless — что отслеживать? ]

<details>
<summary>Ответ</summary>

* **Ключевые метрики Lambda:**
  * **Invocations** — количество вызовов.
  * **Duration** — время выполнения (p50, p90, p99).
  * **Errors** — количество ошибок + error rate.
  * **Throttles** — вызовы отклонены (concurrency limit).
  * **ConcurrentExecutions** — текущее кол-во одновременных вызовов.
  * **IteratorAge** (для stream-based) — задержка обработки (DynamoDB Streams, Kinesis).
* **Алерты:**
  * Error rate > 1% → pager.
  * Duration p99 > threshold → investigate.
  * Throttles > 0 → scale or increase concurrency.
  * IteratorAge > threshold → consumer не успевает.
* **Distributed tracing:**
  * AWS X-Ray — trace через Lambda → DynamoDB → S3 → external API.
  * Видно bottleneck в цепочке.
* **Structured logging:**
  * JSON-логи → CloudWatch Logs Insights → query и dashboard.
* **Cost monitoring:**
  * AWS Cost Explorer → tag-based cost allocation.
  * Budget alarms.

</details>

---

### Вопрос №6: [ Serverless anti-patterns? ]

<details>
<summary>Ответ</summary>

* **1. Fat Lambda** — одна огромная функция делает всё. Лучше — одна функция = одна задача.
* **2. Chatty Lambda** — функция делает много вызовов к БД/API в цикле. Лучше — batch.
* **3. Blocking I/O в Lambda** — sync HTTP-запросы с длинным timeout → дорого и медленно.
  Лучше — async (Promise.all) или EventBridge.
* **4. Storing state in global variables** — не гарантируется сохранность между вызовами.
  Лучше — DynamoDB/Redis.
* **5. Heavy init code** — загрузка больших моделей/данных в global scope → долгий cold start.
  Лучше — lazy init.
* **6. Over-provisioned memory** — память не используется, но платишь за CPU тоже. Найти
  оптимальный размер (Power Tuning).
* **7. Lambda for long-polling/WebSocket** — Lambda не предназначен для долгоживущих
  connections. Лучше — API Gateway WebSocket + Lambda для обработки events.
* **8. Direct Lambda-to-Lambda calls** — sync вызов Lambda из Lambda → blocking, cascading
  failures. Лучше — event-driven (SQS/EventBridge).
* **9. Ignoring idempotency** — at-least-once delivery → дубликаты. Всегда делать idempotent.

</details>

---

</details>

---

## Полезные ресурсы

- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)
- [Serverless Framework Documentation](https://www.serverless.com/framework/docs/)
- [AWS SAM Documentation](https://docs.aws.amazon.com/serverless-application-model/)
- [Google Cloud Functions Documentation](https://cloud.google.com/functions/docs)
- [Azure Functions Documentation](https://docs.microsoft.com/en-us/azure/azure-functions/)
- [AWS Step Functions Documentation](https://docs.aws.amazon.com/step-functions/)
- [Serverless Architecture Patterns (AWS)](https://serverlessland.com/patterns)
- [DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)

---

## Связанные файлы

- [api-design.md](api-design.md) — REST/GraphQL API design
- [message-brokers.md](message-brokers.md) — SQS/SNS/EventBridge в Serverless
- [observability.md](observability.md) — мониторинг и tracing Serverless
- [security.md](security.md) — security best practices
- [infrastructure-as-code.md](infrastructure-as-code.md) — IaC для Serverless (SAM, Serverless Framework)
- [system-design-questions.md](system-design-questions.md) — архитектура distributed систем
