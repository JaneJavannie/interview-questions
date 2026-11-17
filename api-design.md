# Проектирование API — Вопросы и ответы

Файл содержит вопросы по проектированию API: REST, gRPC, GraphQL, WebSocket, Webhooks,
версионирование, пагинация, обработка ошибок, аутентификация.

Уровни сложности:
- ![Static Badge](https://img.shields.io/badge/Middle-yellow) — Middle разработчик
- ![Static Badge](https://img.shields.io/badge/Senior-red) — Senior разработчик

---

# Оглавление

1. [REST API](#1-rest-api)
2. [gRPC](#2-grpc)
3. [GraphQL](#3-graphql)
4. [WebSocket и Server-Sent Events](#4-websocket-и-server-sent-events)
5. [Webhooks](#5-webhooks)
6. [Версионирование и совместимость](#6-версионирование-и-совместимость)
7. [Пагинация, фильтрация, сортировка](#7-пагинация-фильтрация-сортировка)
8. [Обработка ошибок и статус-коды](#8-обработка-ошибок-и-статус-коды)
9. [Аутентификация и авторизация в API](#9-аутентификация-и-авторизация-в-api)

---

## 1. REST API

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Что такое REST и какие его принципы? ]

<details>
<summary>Ответ</summary>

* **REST** (Representational State Transfer) — архитектурный стиль для распределённых систем.
* 6 принципов (по Fielding):
  1. **Client-Server** — разделение ответственности (UI отделён от хранения данных).
  2. **Stateless** — каждый запрос содержит всю необходимую информацию, сервер не хранит состояние сессии.
  3. **Cacheable** — ответы должны явно указывать, можно ли их кэшировать и на сколько.
  4. **Uniform Interface** — единый интерфейс (URI, HTTP методы, представления, HATEOAS).
  5. **Layered System** — клиент не знает, подключён ли напрямую к серверу или через промежуточные слои (proxy, LB).
  6. **Code on Demand** (опционально) — сервер может отправлять исполняемый код (JS), который клиент выполняет.

</details>

---

### Вопрос №2: [ Что такое HATEOAS и используется ли на практике? ]

<details>
<summary>Ответ</summary>

* **HATEOAS** (Hypermedia As The Engine Of Application State) — клиент получает не только данные, но и ссылки
  на доступные действия (гиперссылки). Клиент не должен заранее знать URL'ы — он следует по ссылкам.

```json
{
  "id": 123,
  "status": "pending",
  "_links": {
    "self": { "href": "/orders/123" },
    "cancel": { "href": "/orders/123/cancel", "method": "POST" },
    "pay": { "href": "/orders/123/pay", "method": "POST" }
  }
}
```

* На практике: редко используется в чистом виде. Больше применяется в зрелых REST API (например, Spring Boot HAL).
  Большинство API останавливаются на уровне 2 ( Richardson Maturity Model) — HTTP методы и статус-коды.

</details>

---

### Вопрос №3: [ Какие HTTP методы использовать для CRUD и их свойства? ]

<details>
<summary>Ответ</summary>

| Метод | CRUD | Safe | Idempotent | Cacheable | Body |
|-------|------|------|------------|-----------|------|
| GET | Read | ✅ | ✅ | ✅ | ❌ |
| POST | Create | ❌ | ❌ | Условно | ✅ |
| PUT | Update/Replace | ❌ | ✅ | ❌ | ✅ |
| PATCH | Partial update | ❌ | ❌ (зависит от операции) | ❌ | ✅ |
| DELETE | Delete | ❌ | ✅ | ❌ | ❌ |
| HEAD | Read (headers only) | ✅ | ✅ | ✅ | ❌ |
| OPTIONS | Metadata | ✅ | ✅ | ❌ | ❌ |

* **Safe** — не изменяет состояние сервера.
* **Idempotent** — повторный вызов даёт тот же результат.
* PUT — полная замена ресурса. PATCH — частичное обновление.
* DELETE — идемпотентный (удалить уже удалённое — тот же результат: ресурса нет).

</details>

---

### Вопрос №4: [ REST vs RPC — в чём разница? ]

<details>
<summary>Ответ</summary>

* **REST** — ориентирован на ресурсы (существительные): `GET /users/1`, `POST /orders`.
* **RPC** (Remote Procedure Call) — ориентирован на действия (глаголы): `POST /getUser`, `POST /createOrder`.
* REST лучше для CRUD-операций, стандартное кэширование, понятная семантика HTTP.
* RPC лучше для сложных операций, не укладывающихся в CRUD: `POST /users/1/transfer`, `POST /calculateTax`.
* gRPC — современный RPC на Protocol Buffers + HTTP/2.
* На практике: REST для публичного API, gRPC для внутреннего межсервисного.

</details>

---

### Вопрос №5: [ Что такое Richardson Maturity Model? ]

<details>
<summary>Ответ</summary>

* Модель зрелости REST API (по Leonard Richardson), 4 уровня:
  - **Level 0 (POX)** — один URI, один HTTP метод (обычно POST), всё через SOAP-like.
  - **Level 1 (Resources)** — разные URI для разных ресурсов, но один метод.
  - **Level 2 (HTTP Verbs)** — правильное использование HTTP методов и статус-кодов.
  - **Level 3 (HATEOAS)** — гипермедиа-ссылки в ответах.
* Большинство "REST API" в мире — Level 2.

</details>

---

</details>

---

## 2. gRPC

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Что такое gRPC и как он работает? ]

<details>
<summary>Ответ</summary>

* **gRPC** — RPC-фреймворк от Google на базе HTTP/2 и Protocol Buffers (protobuf).
* **Схема**: `.proto` файл описывает сервисы и сообщения → кодогенерация на нужный язык → клиент/сервер.
* **Protocol Buffers** — бинарный формат сериализации. Компактнее JSON, строгая схема, обратная совместимость.
* **HTTP/2** — мультиплексирование, binary framing, server push, header compression (HPACK).
* Плюсы: строгая типизация, производительность (бинарный протокол, HTTP/2 мультиплексирование),
  встроенная кодогенерация клиентов, streaming.
* Минусы: нечитаемо без инструментов (нельзя просто curl), требует кодогенерации, heavyweight для простых API.

</details>

---

### Вопрос №2: [ Какие виды streaming поддерживает gRPC? ]

<details>
<summary>Ответ</summary>

* **Unary RPC** — обычный request → response (как REST).
* **Server streaming** — клиент отправляет один запрос, сервер потоково отправляет несколько сообщений.
  Полезно для: large result sets, real-time updates, прогресс-бары.
* **Client streaming** — клиент отправляет поток сообщений, сервер отвечает одним.
  Полезно для: загрузка данных, агрегация, batching.
* **Bidirectional streaming** — обе стороны отправляют потоки сообщений независимо.
  Полезно для: чаты, real-time collaboration, bidirectional sync.

```protobuf
service Chat {
  rpc SendMessage(stream Message) returns (stream Message);  // bidirectional
  rpc Subscribe(SubscribeRequest) returns (stream Event);     // server streaming
  rpc Upload(stream Chunk) returns (UploadResponse);          // client streaming
  rpc GetUser(GetUserRequest) returns (User);                 // unary
}
```

</details>

---

### Вопрос №3: [ Что такое gRPC interceptors? ]

<details>
<summary>Ответ</summary>

* **Interceptors** — middleware для gRPC, аналог HTTP middleware в REST.
* **Unary interceptors** — для обычных unary RPC.
* **Stream interceptors** — для streaming RPC.
* Используются для: аутентификации, логирования, метрик, tracing, rate limiting, recovery.

```go
func LoggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (resp interface{}, err error) {
    start := time.Now()
    resp, err = handler(ctx, req)
    log.Printf("%s %v %v", info.FullMethod, time.Since(start), err)
    return resp, err
}

server := grpc.NewServer(grpc.UnaryInterceptor(LoggingInterceptor))
```

</details>

---

### Вопрос №4: [ Что такое gRPC-Gateway и зачем нужен? ]

<details>
<summary>Ответ</summary>

* **gRPC-Gateway** — плагин для protobuf, генерирующий REST-прокси (reverse proxy) для gRPC сервисов.
* В `.proto` добавляются аннотации `google.api.http`:

```protobuf
rpc GetUser(GetUserRequest) returns (User) {
  option (google.api.http) = {
    get: "/v1/users/{id}"
  };
}
```

* Генерируется HTTP-сервер, который принимает REST-запросы и транслирует их в gRPC-вызовы.
* Плюсы: оба мира — gRPC для внутренних сервисов, REST для браузеров/мобильных клиентов.
* Альтернатива: Envoy/Envoy gRPC-JSON transcoding.

</details>

---

### Вопрос №5: [ Как работает deadline/timeout propagation в gRPC? ]

<details>
<summary>Ответ</summary>

* gRPC использует `context.Context` с deadline для управления таймаутами.
* Deadline распространяется по цепочке вызовов: если сервис A вызывает B с deadline=5s, и B вызывает C,
  то C получит оставшееся время (например, 3s, если B уже работал 2s).
* Сервер должен уважать `ctx.Done()` — отменять долгие операции при отмене контекста.

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
resp, err := client.GetUser(ctx, req)
```

* `grpc-timeout` header передаётся в каждом вызове. Если deadline истёк — `status.Code = DeadlineExceeded`.

</details>

---

### Вопрос №6: [ Чем protobuf отличается от JSON? ]

<details>
<summary>Ответ</summary>

| Свойство | Protobuf | JSON |
|----------|----------|------|
| Формат | Бинарный | Текстовый |
| Размер | Меньше (нет имён полей) | Больше (имена полей в каждом объекте) |
| Скорость | Быстрее (десериализация) | Медленнее |
| Схема | Обязательна (.proto) | Опциональна (JSON Schema) |
| Читаемость | Нет (нужен инструмент) | Да |
| Совместимость | Строгие правила (не менять номер поля) | Гибкая |
| Типы | Строгие | Динамические |

* Protobuf — строгая схема, бинарный, для внутреннего межсервисного.
* JSON — гибкий, читаемый, для публичного API и браузеров.

</details>

---

### Вопрос №7: [ Особенности реализации gRPC в Go (grpc-go) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Соединение**: `grpc.NewClient` (появился в grpc-go v1.63, рекомендуется вместо
  устаревшего `grpc.Dial`) — создаётся лениво, не блокирует; одно соединение мультиплексирует
  множество RPC поверх HTTP/2.
* **Балансировка на клиенте**: `pick_first` (по умолчанию) и `round_robin` — когда один адрес
  резолвится в несколько эндпоинтов. Для клиентской балансировки без L7-прокси
  обычно ставят `round_robin`.
* **Keepalive**: `keepalive.ClientParameters` / `ServerParameters` — пинги для обнаружения
  мёртвых соединений и обхода таймаутов балансировщиков; согласовывать с серверной
  `EnforcementPolicy`, иначе сервер может разорвать соединение (GOAWAY).
* **Interceptors** (см. вопрос про интерсепторы): в Go — `grpc.UnaryInterceptor` /
  `StreamInterceptor` и цепочки `grpc.ChainUnaryInterceptor`; типичные применения:
  логирование, метрики, трейсинг, ретраи, авторизация.
* **Дедлайны**: всегда задавать `context.WithTimeout` на каждый вызов; дедлайн пробрасывается
  на сервер (см. вопрос про deadline propagation).
* **Graceful shutdown сервера**: `server.GracefulStop()` — дожидается активные RPC;
  `server.Stop()` — немедленный. В связке с `http.Server`-хэндлингом сигналов.
* **Генерация кода**: `protoc` + `protoc-gen-go` + `protoc-gen-go-grpc`; версии плагинов
  должны соответствовать версиям библиотек `google.golang.org/protobuf` и `grpc`.

</details>

---

</details>

---

## 3. GraphQL

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Что такое GraphQL и чем отличается от REST? ]

<details>
<summary>Ответ</summary>

* **GraphQL** — язык запросов для API, разработанный Facebook.
* Вместо множества endpoint'ов — **одна** endpoint (`/graphql`), клиент описывает нужные поля в запросе.
* Клиент получает **ровно те поля, которые запросил** — no overfetching, no underfetching.

```graphql
query {
  user(id: 1) {
    name
    email
    orders(last: 5) {
      total
      status
    }
  }
}
```

| Свойство | GraphQL | REST |
|----------|---------|------|
| Endpoint | Один | Много |
| Данные | Клиент выбирает поля | Сервер определяет |
| Overfetching | Нет | Часто |
| Underfetching | Нет (nested queries решают N+1, DataLoader — батчинг) | Часто |
| Версионирование | Эволюция схемы | Версии в URL |
| Кэширование | Сложнее (POST) | HTTP caching (GET) |
| Файлы | Сложно | Просто |

</details>

---

### Вопрос №2: [ Что такое N+1 проблема в GraphQL и как её решить? ]

<details>
<summary>Ответ</summary>

* **Проблема**: запрос `user { orders { product { name } } }` — для каждого user'а загружаются orders,
  для каждого order — product. Если 100 users × 10 orders = 1000 запросов в БД (N+1).
* **Решение — DataLoader**: батчит запросы за один "tick" (event loop) и делает один запрос в БД с `WHERE id IN (...)`.

```go
loader := &dataloader.Loader{
    Fetch: func(keys []string) ([]*Product, []error) {
        // один SELECT * FROM products WHERE id IN (...)
        return fetchProductsByIDs(keys)
    },
}
```

* DataLoader также кэширует результаты в рамках одного запроса (per-request cache).

</details>

---

### Вопрос №3: [ Что такое mutation и subscription в GraphQL? ]

<details>
<summary>Ответ</summary>

* **Query** — чтение данных (GET аналог).
* **Mutation** — изменение данных (POST/PUT/DELETE аналог). Выполняются последовательно (top-to-bottom),
  в отличие от query, где поля могут выполняться параллельно.

```graphql
mutation {
  createUser(input: { name: "Alice", email: "alice@example.com" }) {
    id
    name
  }
}
```

* **Subscription** — long-lived подключение (обычно через WebSocket), сервер отправляет события клиенту.
  Полезно для real-time updates.

```graphql
subscription {
  newMessage(roomId: 1) {
    id
    text
    author
  }
}
```

</details>

---

### Вопрос №4: [ Как работает schema stitching / federation? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

* **Schema stitching** — объединение схем нескольких GraphQL-сервисов в одну gateway-схему.
  Gateway делегирует запросы нужным под-сервисам.
* **Apollo Federation** — стандартизированный подход:
  - Каждый сервис определяет свою часть схемы.
  - `@extends` / `@external` — расширение типов из других сервисов.
  - Gateway (Apollo Router) строит единый граф и маршрутизирует запросы.
* **Пример**: Users service определяет `User`, Orders service определяет `Order` и расширяет `User` полем `orders`.
  Gateway объединяет и клиент может запросить `user { name orders { total } }` — gateway сам соберёт данные из двух сервисов.

</details>

---

### Вопрос №5: [ Как кэшировать GraphQL-запросы? ]

<details>
<summary>Ответ</summary>

* GraphQL обычно отправляется через POST — HTTP кэширование не работает напрямую.
* **Persisted Queries** — клиент отправляет hash запроса вместо полного текста. Сервер по hash находит query.
  Можно использовать GET с hash: `GET /graphql?extensions={"persistedQuery":{"sha256Hash":"abc..."}}` → кэшируется CDN.
* **Apollo Client cache** — нормализованный кэш на клиенте (по `__typename:id`).
* **Server-side cache** — Redis для дорогих resolver'ов с TTL.
* **CDN** — для persisted queries через GET, CDN кэширует ответ.

</details>

---

</details>

---

## 4. WebSocket и Server-Sent Events

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Что такое WebSocket и как он работает? ]

<details>
<summary>Ответ</summary>

* **WebSocket** — протокол поверх TCP для полнодуплексной (двунаправленной) коммуникации.
* **Handshake**: клиент отправляет HTTP-запрос с `Upgrade: websocket`, сервер отвечает `101 Switching Protocols`.
  После этого соединение "апгрейдится" до WebSocket (больше не HTTP).
* Фреймы: text (UTF-8), binary, ping/pong (keepalive), close.
* Плюсы: real-time, двунаправленный, низкая задержка (нет overhead HTTP headers на каждое сообщение).
* Минусы: сложнее инфраструктура (load balancer, proxy должны поддерживать WS), нет встроенного кэширования,
  stateful — сложно масштабировать (sticky sessions, Redis pub/sub для broadcast).

</details>

---

### Вопрос №2: [ WebSocket vs Server-Sent Events (SSE) vs Long Polling ]

<details>
<summary>Ответ</summary>

| Свойство | WebSocket | SSE | Long Polling |
|----------|-----------|-----|--------------|
| Направление | Двунаправленный | Сервер → клиент | Сервер → клиент (через запросы) |
| Протокол | WS (over TCP) | HTTP | HTTP |
| Бинарные данные | ✅ | ❌ (текст) | ❌ |
| Reconnect | Вручную | Авто (EventSource) | Вручную |
| Proxy/LB | Нужна поддержка | Обычный HTTP | Обычный HTTP |
| Complexity | Выше | Ниже | Низшая |
| Использование | Чаты, игры, real-time | Уведомления, feeds | Legacy fallback |

* **SSE** — простое server→client streaming, `Content-Type: text/event-stream`.
* **Long polling** — клиент делает запрос, сервер держит открытым, пока не появится данных, отвечает, клиент повторяет.

</details>

---

### Вопрос №3: [ Как масштабировать WebSocket-серверы? ]

<details>
<summary>Ответ</summary>

* **Проблема**: WebSocket stateful — соединение привязано к конкретному серверу.
* **Sticky sessions** — LB направляет запросы от одного клиента на один сервер (по cookie/IP).
  Минус: при падении сервера — потеря всех соединений.
* **Pub/Sub backbone** — Redis Pub/Sub, Kafka, NATS для broadcast между серверами.
  Сервер A получает WS-сообщение → публикует в Redis → все серверы (включая A) получают и рассылают своим клиентам.
* **Connection draining** — при graceful shutdown сервер отправляет close frame, клиенты реконнектят.
* **Heartbeat** — ping/pong для обнаружения "мёртвых" соединений и cleanup.
* **Metrics** — отслеживать active connections per node, latency, message rate.

</details>

---

</details>

---

## 5. Webhooks

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Что такое webhooks и как они работают? ]

<details>
<summary>Ответ</summary>

* **Webhook** — сервер вызывает HTTP endpoint клиента при наступлении события (reverse API).
* Клиент регистрирует URL (callback), сервер делает POST-запрос на этот URL при событии.
* Пример: Stripe отправляет webhook на `https://example.com/webhooks/stripe` при успешном платеже.

```http
POST /webhooks/stripe HTTP/1.1
Content-Type: application/json
Stripe-Signature: t=...,v1=...

{
  "type": "payment_intent.succeeded",
  "data": { "object": { "id": "pi_123", "amount": 1000 } }
}
```

* Сервер должен ответить `200 OK` в разумное время (обычно 5–30 сек), иначе — retry.

</details>

---

### Вопрос №2: [ Как обеспечить надёжность webhooks? ]

<details>
<summary>Ответ</summary>

* **Retry с exponential backoff** — если клиент не ответил 2xx, повторить через 1 мин, 5 мин, 30 мин, 2 часа...
* **Подпись** — HMAC-SHA256 от тела запроса с секретным ключом. Клиент проверяет подпись, чтобы убедиться,
  что запрос от реального отправителя.

```go
func verifySignature(payload []byte, sig string, secret string) bool {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(payload)
    expectedMAC := hex.EncodeToString(mac.Sum(nil))
    return hmac.Equal([]byte(sig), []byte(expectedMAC))
}
```

* **Idempotency** — webhook может прийти дважды (retry). Worker использует `event_id`:
  отметка обработки и бизнес-изменение — в одной транзакции БД; ack очереди — после commit.
  Для внешних эффектов нужны idempotency key провайдера и сверка результата при неопределённом исходе.
* **Timeout** — короткий timeout (5–10 сек): проверить подпись и ответить 2xx **только после**
  commit события в durable inbox или подтверждённой записи в надёжную очередь. Если сохранить
  не удалось — не отвечать 2xx, чтобы отправитель повторил доставку. Отдельный worker обрабатывает
  событие асинхронно с retry; fire-and-forget goroutine перед ответом не защищает от потери при падении.
* **Мониторинг** — метрики на delivery success rate, retry count, DLQ для неудачных.
* **Порядок** — webhooks могут прийти не по порядку. Использовать timestamp / version для определения актуальности.

</details>

---

### Вопрос №3: [ В чём разница между webhooks и polling? ]

<details>
<summary>Ответ</summary>

* **Polling** — клиент периодически опрашивает сервер: `GET /status/123` каждые 10 секунд.
  Плюсы: просто реализовать, клиент контролирует частоту.
  Минусы: лишние запросы (пустые ответы), задержка между событием и обнаружением, нагрузка на сервер.
* **Webhooks** — сервер сам вызывает клиента при событии.
  Плюсы: мгновенная доставка, нет лишних запросов, эффективнее.
  Минусы: клиент должен иметь публичный endpoint, сложнее безопасность, retry-логика.
* **Long polling** — компромисс: клиент делает запрос, сервер держит открытым, пока не появится событие.
* На практике: webhooks для server-to-server, polling для клиентских приложений без public endpoint.

</details>

---

</details>

---

## 6. Версионирование и совместимость

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Какие способы версионирования API существуют? ]

<details>
<summary>Ответ</summary>

* **URI versioning** — `/v1/users`, `/v2/users`. Самый популярный, простой, явный.
* **Header versioning** — `Accept: application/vnd.api+json; version=2`. Чистый URL, но менее очевидно.
* **Query parameter** — `/users?version=2`. Просто, но может конфликтовать с бизнес-параметрами.
* **Content negotiation** — `Accept: application/vnd.example.user.v2+json`.
* На практике: URI versioning (`/v1/`) — самый распространённый и понятный.

</details>

---

### Вопрос №2: [ Что такое backward и forward compatibility? ]

<details>
<summary>Ответ</summary>

* **Backward compatibility** для API — новый сервер работает со старыми клиентами.
  (Клиент v1 может работать с API v2.)
* **Forward compatibility** для API — старый сервер работает с новыми клиентами.
  (Клиент v2 учитывает отсутствие новых возможностей API v1; сервер терпим к новым данным.)
* Для схем сообщений: backward — новый читатель понимает старые данные, forward — старый читатель новые.
* Правила для REST/JSON:
  - Добавление полей ответа совместимо, если старый клиент допускает неизвестные поля.
  - Удаление ожидаемых полей ответа, переименование и изменение типа — несовместимые изменения.
  - В запросе обязательное → опциональное совместимо со старыми клиентами; обратное — нет.
  - В ответе обязательное → опциональное может сломать клиента, который рассчитывает на наличие поля.

* **Protobuf: проверять отдельно binary wire, исходный код/API и ProtoJSON.**
  - Номер поля (tag) — неизменный идентификатор в бинарном формате. Никогда не менять его
    у существующего поля и не переиспользовать, даже если поле давно удалено.
  - После удаления резервировать **и номера, и имена** через отдельные `reserved`:
    это защищает от повторного использования в схеме, но не создаёт алиасы для старого JSON.
  - Новые поля должны допускать отсутствие у старых отправителей. В proto3 нет `required`,
    но это не означает, что у всех полей одинаково отслеживается наличие (presence).

```proto
syntax = "proto3";

message User {
  reserved 2, 3;
  reserved "old_name", "legacy_age";

  string name = 1;
  optional int32 age = 4;
}
```

* **Presence в proto3:**
  - Обычный scalar без `optional` (и вне `oneof`) имеет implicit presence: после чтения нельзя
    отличить отсутствие от значения по умолчанию (`0`, `false`, `""`); default обычно не сериализуется.
  - `optional` даёт explicit presence: `age` не задан и `age = 0` различимы через API наличия поля;
    явно заданный ноль сериализуется. Это не делает поле обязательным.
  - Одиночное поле типа message имеет presence и без `optional`: отсутствует ≠ задано пустое сообщение.
  - `repeated` и `map` не имеют presence: пустая коллекция и отсутствие неразличимы.

* **Границы совместимости:**
  - Переименование при том же tag и типе безопасно для binary wire, но ломает обращения к полю
    в сгенерированном коде и по умолчанию меняет ключ ProtoJSON. `json_name` позволяет сохранить
    JSON-ключ, но не совместимость исходного кода и не все варианты входных имён.
  - Binary protobuf сохраняет неизвестные поля при чтении и повторной сериализации сообщения;
    преобразование в JSON или копирование только известных полей может их потерять.
  - ProtoJSON по умолчанию отвергает неизвестные поля. Режим их игнорирования не сохраняет их
    для обратной сериализации, поэтому даже добавление поля требует учёта старых JSON-читателей.
  - Не всякая смена типа безопасна: даже бинарно совместимое `int32` → `int64` может привести
    к обрезанию значений у старого читателя. Совместимость wire не гарантирует прежнюю бизнес-семантику.

* Источники: [proto3](https://protobuf.dev/programming-guides/proto3/),
  [ProtoJSON](https://protobuf.dev/programming-guides/json/).

</details>

---

### Вопрос №3: [ Как деплоить новую версию API без даунтайма? ]

<details>
<summary>Ответ</summary>

* **Expand → Migrate → Contract**:
  1. **Expand**: добавить новые поля/endpoint'ы, не удаляя старые. Обе версии работают.
  2. **Migrate**: перевести клиентов на новую версию (постепенно, с observability).
  3. **Contract**: удалить старые поля/endpoint'ы после полной миграции.
* **Shadow traffic** — отправлять копии запросов на новую версию, сравнивать результаты.
* **Feature flags** — включать новое поведение для части пользователей.
* **Deprecation header** — `Sunset: Sat, 31 Dec 2025 23:59:59 GMT` + `Deprecation: true`.
* **Мониторинг** — отслеживать использование старой версии, уведомлять клиентов.

</details>

---

</details>

---

## 7. Пагинация, фильтрация, сортировка

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Какие виды пагинации существуют? ]

<details>
<summary>Ответ</summary>

* **Offset/Limit** — `GET /users?offset=20&limit=10` (или `?page=3&size=10`).
  Простой, но: при добавлении новых записей смещаются, `OFFSET 1000000` — медленно (сканирует все строки).
* **Cursor-based** (keyset) — `GET /users?cursor=eyJpZCI6MTIwfQ==&limit=10`.
  Cursor = закодированный указатель (обычно last item ID + sort field).
  Стабилен при добавлении новых записей, быстрый (uses index). Не позволяет перейти на страницу N.
* **Page-based** — `GET /users?page=3&per_page=10`. По сути offset/limit, удобен для UI с номерами страниц.
* **Time-based** — `GET /events?since=2025-01-01T00:00:00Z&limit=10`. Для временных рядов.

```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTMwfQ==",
    "has_more": true
  }
}
```

</details>

---

### Вопрос №2: [ Почему offset-пагинация медленная на больших таблицах? ]

<details>
<summary>Ответ</summary>

* `OFFSET 1000000 LIMIT 10` — PostgreSQL должен сканировать и отбросить 1M строк, затем вернуть 10.
  Это O(N) где N = offset.
* **Cursor/keyset** — `WHERE id > last_id ORDER BY id LIMIT 10` — использует index seek, O(log N + limit).
* Для сортировки по другому полю: `WHERE (created_at, id) < ('2025-01-01', 12345) ORDER BY created_at, id LIMIT 10` —
  composite index делает это быстрым.
* Cursor pagination не работает с "jump to page 5", но для infinite scroll / "load more" — идеален.

</details>

---

### Вопрос №3: [ Как реализовать фильтрацию и сортировку в API? ]

<details>
<summary>Ответ</summary>

* **Query parameters**:

```
GET /products?category=electronics&min_price=100&max_price=500&sort=-price,name
```

* `sort=-price` — по убыванию, `sort=price` — по возрастанию. Запятая — несколько полей.
* **Filter operators**: `?price__gte=100&price__lte=500` (Django-style), или `?min_price=100&max_price=500`.
* **RSQL / FIQL** — стандартизированный язык фильтрации: `?q=price>100;category==electronics`.
* **JSON:API** — `?filter[category]=electronics&sort=-price`.
* Для сложных фильтров (full-text search, geo) — отдельные endpoint'ы или GraphQL.

</details>

---

</details>

---

## 8. Обработка ошибок и статус-коды

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Какие категории HTTP статус-кодов существуют? ]

<details>
<summary>Ответ</summary>

* **1xx** — Informational (редко используются).
* **2xx** — Success: 200 OK, 201 Created, 202 Accepted, 204 No Content.
* **3xx** — Redirection: 301 Moved Permanently, 302 Found, 304 Not Modified.
* **4xx** — Client Error: 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict,
  422 Unprocessable Entity, 429 Too Many Requests.
* **5xx** — Server Error: 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout.

</details>

  ---

### Вопрос №2: [ 401 vs 403 — в чём разница? ]

<details>
<summary>Ответ</summary>

* **401 Unauthorized** — клиент **не аутентифицирован** (не логинился, нет/неверный токен).
  Response должен включать `WWW-Authenticate` header.
* **403 Forbidden** — клиент аутентифицирован, но **не имеет прав** на это действие.
  (Залогинен, но не admin; пытается удалить чужой ресурс.)
* Простое правило: 401 = "я не знаю, кто ты", 403 = "я знаю, кто ты, но тебе нельзя".

</details>

---

### Вопрос №3: [ Какой формат ошибки использовать в REST API? ]

<details>
<summary>Ответ</summary>

* Стандартизированный формат с machine-readable кодом и human-readable сообщением:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid email format",
    "details": [
      { "field": "email", "message": "Must be a valid email address" }
    ],
    "request_id": "req_abc123"
  }
}
```

* **RFC 9457 (Problem Details for HTTP APIs)** — стандартный формат:

```json
{
  "type": "https://example.com/errors/validation",
  "title": "Validation failed",
  "status": 422,
  "detail": "Email format is invalid",
  "instance": "/users"
}
```

* `request_id` — для корреляции с логами и трейсами. Полезно для debugging.

</details>

---

### Вопрос №4: [ Когда использовать 200 с ошибкой в body vs 4xx/5xx? ]

<details>
<summary>Ответ</summary>

* **Правило**: используйте HTTP статус-коды для передачи результата. 4xx/5xx — для ошибок.
* **Антипаттерн**: `200 OK` с `{"success": false, "error": "..."}` — клиент не может полагаться на статус-код,
  proxy/LB могут не отреагировать на ошибку.
* **Исключение**: GraphQL возвращает всегда 200 (даже при ошибке), ошибки в body. Это спецификация GraphQL.
* **gRPC** — использует свои статус-коды (OK, NOT_FOUND, PERMISSION_DENIED, ...), а не коды HTTP:
  они передаются в заголовке/трейлере `grpc-status` поверх HTTP/2 (на транспортном уровне — всегда 200).

</details>

---

</details>

---

## 9. Аутентификация и авторизация в API

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ API Key vs Bearer Token (JWT) — что выбрать? ]

<details>
<summary>Ответ</summary>

* **API Key** — статический ключ, передаётся в header (`X-API-Key: abc123`) или query (`?api_key=abc123`).
  Просто, но: нельзя отозвать без перевыдачи, нет expiration, нет метаданных (кто, какие права).
* **JWT (Bearer Token)** — self-contained токен с claims (user_id, roles, exp).
  Stateless: сервер не делает запрос в БД для проверки (проверяет подпись).
  Минусы: нельзя отозвать до истечения (решение: short-lived + refresh token, или blacklist в Redis).
* **OAuth2 Access Token** — opaque token, сервер делает introspection request к auth-серверу.
  Или JWT Bearer token.
* На практике: API Key для server-to-server, JWT для user-facing API, OAuth2 для third-party access.

</details>

---

### Вопрос №2: [ Как передавать токен в API? ]

<details>
<summary>Ответ</summary>

* **Authorization header** (стандарт):
  ```
  Authorization: Bearer eyJhbGciOi...
  ```
* **API Key header**: `X-API-Key: abc123`
* **Cookie** — `Cookie: session=abc123`. Для браузерных приложений с `HttpOnly`, `Secure`, `SameSite`.
* **Query parameter** — `?token=abc123`. **Небезопасно**: логируется в access logs, виден в referrer.
* Best practice: Authorization header для API, HttpOnly cookie для веб-приложений.

</details>

---

### Вопрос №3: [ Что такое CORS и зачем нужен? ]

<details>
<summary>Ответ</summary>

* **CORS** (Cross-Origin Resource Sharing) — механизм браузера, запрещающий JavaScript делать запросы на другой origin
  без разрешения.
* **Origin** = scheme + host + port (например, `https://example.com:443`).
* **Simple requests** (GET, POST с определёнными content-type) — браузер отправляет напрямую, но проверяет
  `Access-Control-Allow-Origin` в ответе.
* **Preflight requests** — для "нестандартных" запросов (PUT, DELETE, custom headers) браузер сначала отправляет
  `OPTIONS` запрос. Сервер отвечает с CORS headers, и только потом браузер отправляет основной запрос.
* Headers:
  ```
  Access-Control-Allow-Origin: https://example.com
  Access-Control-Allow-Methods: GET, POST, PUT, DELETE
  Access-Control-Allow-Headers: Authorization, Content-Type
  Access-Control-Allow-Credentials: true
  Access-Control-Max-Age: 3600
  ```
* `Access-Control-Allow-Origin: *` + `Allow-Credentials: true` — **не работает** (безопасность).

</details>

---

</details>

---
