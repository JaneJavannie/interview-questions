# Очереди сообщений (Message Brokers) — Вопросы и ответы

Файл содержит вопросы по RabbitMQ, Kafka, NATS, паттернам доставки, гарантиям и сценариям использования.

Уровни сложности:
- ![Static Badge](https://img.shields.io/badge/Middle-yellow) — Middle разработчик
- ![Static Badge](https://img.shields.io/badge/Senior-red) — Senior разработчик

---

# Оглавление

1. [Основы и сравнение брокеров](#1-основы-и-сравнение-брокеров)
2. [RabbitMQ — архитектура и паттерны](#2-rabbitmq--архитектура-и-паттерны)
3. [Apache Kafka](#3-apache-kafka)
4. [NATS и другие брокеры](#4-nats-и-другие-брокеры)
5. [Гарантии доставки и надёжность](#5-гарантии-доставки-и-надёжность)
6. [Паттерны и антипаттерны](#6-паттерны-и-антипаттерны)
7. [Сравнение брокеров: когда что выбрать](#7-сравнение-брокеров-когда-что-выбрать)

---

## 1. Основы и сравнение брокеров

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Что такое message broker и зачем он нужен? ]

<details>
<summary>Ответ</summary>

* Message broker — промежуточное ПО, которое принимает сообщения от producer'ов и доставляет их consumer'ам.
* Основные функции:
  - **Асинхронная коммуникация** — producer не ждёт обработки.
  - **Развязка (decoupling)** — producer и consumer не знают друг о друге.
  - **Буферизация** — сглаживает пики нагрузки.
  - **Гарантии доставки** — at-least-once, at-most-once, exactly-once.
  - **Маршрутизация** — сообщения могут идти разным consumer'ам по правилам.

</details>

---

### Вопрос №2: [ В чём разница между очередью (queue) и логом (log)? ]

<details>
<summary>Ответ</summary>

* **Queue** (RabbitMQ classic queue): сообщение удаляется после подтверждения обработки consumer'ом.
  Каждый consumer получает свою часть сообщений (competing consumers).
* **Log** (Kafka topic partition): сообщения не удаляются (с retention по времени/размеру).
  Multiple consumer groups могут читать независимо с любой позиции (offset).
  Порядок гарантируется в пределах partition.

| Свойство | Queue (RabbitMQ) | Log (Kafka) |
|----------|------------------|-------------|
| Удаление | После ack | По retention |
| Повторное чтение | Нет | Да (seek) |
| Порядок | В пределах queue | В пределах partition |
| Consumer groups | Competing consumers | Независимые groups |

</details>

  -----

### Вопрос №3: [ Сравните RabbitMQ, Kafka, NATS ]

<details>
<summary>Ответ</summary>

| Характеристика | RabbitMQ | Kafka | NATS |
|----------------|----------|-------|------|
| Модель | AMQP queues | Append-only log | Pub/Sub + Queue |
| Persistence | Опционально | По умолчанию | Опционально (JetStream) |
| Throughput | ~50K msg/s | ~1M+ msg/s | ~1M+ msg/s |
| Latency | ~ms | ~ms | ~µs (core) |
| Retention | До ack | По времени/размеру | По времени (JetStream) |
| Routing | Rich (exchanges) | Partition key | Subjects (wildcards) |
| Ordering | Per queue | Per partition | Per subject |
| Replay | Нет | Да (seek) | Да (JetStream) |
| Язык реализации | Erlang | Scala/Java | Go |
| Лучше всего для | Транзакционные сообщения, RPC | Event streaming, analytics | Лёгкий pub/sub, микросервисы |

</details>

---

### Вопрос №4: [ Когда использовать брокер, а когда REST/gRPC? ]

<details>
<summary>Ответ</summary>

* **Брокер** — когда:
  - Асинхронная обработка (email, notifications, batch processing).
  - Сглаживание пиков нагрузки.
  - Fan-out: одно событие → много потребителей.
  - Надёжность доставки важнее скорости ответа.
* **REST/gRPC** — когда:
  - Нужен синхронный ответ.
  - Request-response с малой задержкой.
  - Простая интеграция (без брокера).
* **Гибрид**: REST для клиентского API, брокер для внутренней асинхронной коммуникации между сервисами.

</details>

---

</details>

---

## 2. RabbitMQ — архитектура и паттерны

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Опишите архитектуру RabbitMQ ]

<details>
<summary>Ответ</summary>

* **Producer** → **Exchange** → **Queue** → **Consumer**
* **Exchange** — принимает сообщения от producer и маршрутизирует в очереди по правилам binding.
* **Queue** — FIFO буфер сообщений, привязан к exchange через binding.
* **Binding** — правило связи exchange → queue (с routing key).
* **Connection** — TCP-соединение.
* **Channel** — виртуальное соединение внутри connection (мультиплексирование).
* **Virtual Host (vhost)** — логическая изоляция exchanges/queues.

</details>

---

### Вопрос №2: [ Какие типы exchange есть в RabbitMQ? ]

<details>
<summary>Ответ</summary>

* **Direct** — сообщение идёт в очереди, где binding key **точно равен** routing key.
  ```
  routing_key = "order.created" → binding_key = "order.created" ✓
  ```
* **Fanout** — сообщение идёт во **все** очереди, привязанные к exchange (routing key игнорируется).
* **Topic** — matching по **wildcard-паттерну**:
  - `*` — одно слово, `#` — ноль и более слов.
  - `order.*` matches `order.created`, `order.cancelled`.
  - `order.#` matches `order.created.paid`, `order.cancelled`.
* **Headers** — маршрутизация по header'ам сообщения (x-match: all/any), routing key игнорируется.
* **Default exchange** (`""`) — special direct exchange, где queue binding key = queue name.

</details>

---

### Вопрос №3: [ Что такое Dead Letter Exchange (DLX)? ]

<details>
<summary>Ответ</summary>

* DLX — exchange, куда отправляются сообщения, которые были:
  - Rejected (basic.reject / basic.nack с requeue=false).
  - Истёк TTL (time-to-live).
  - Превысили лимит длины очереди (overflow).
* Настраивается на уровне очереди:

```
args = {"x-dead-letter-exchange": "dlx", "x-dead-letter-routing-key": "failed"}
```

* Типичный паттерн: основная очередь → DLX → retry queue с TTL → back to main queue (delayed retry).

</details>

---

### Вопрос №4: [ Как работают ack, nack и reject в RabbitMQ? ]

<details>
<summary>Ответ</summary>

* **ack** (basic.ack) — consumer подтверждает успешную обработку, сообщение удаляется.
* **nack** (basic.nack) — consumer сообщает об ошибке. С `requeue=true` сообщение возвращается в очередь,
  с `requeue=false` — идёт в DLX (если настроен) или удаляется.
* **reject** (basic.reject) — то же, что nack, но для одного сообщения (nack может для multiple).
* **auto-ack** — подтверждение происходит автоматически при доставке (fire and forget, риск потери).
* Если consumer отключается без ack, сообщения возвращаются в очередь (redelivery).

</details>

---

### Вопрос №5: [ Что такое prefetch_count и зачем он нужен? ]

<details>
<summary>Ответ</summary>

* `prefetch_count` (QoS) — сколько неподтверждённых сообщений брокер отправит consumer'у за раз.
* `prefetch=1` — строгая fair dispatch: следующее сообщение отправится только после ack предыдущего.
  Минус: низкий throughput.
* `prefetch=unlimited` (по умолчанию) — брокер забивает consumer'а всеми сообщениями.
  Минус: один consumer может получить все сообщения, другие простаивают.
* Рекомендация: подбирать экспериментально (часто 10–100 для баланса latency и throughput).

</details>

---

### Вопрос №6: [ Что такое подтверждение издателя (publisher confirms)? ]

<details>
<summary>Ответ</summary>

* Publisher confirms — подтверждение producer'у обработки публикации брокером, не обработки consumer'ом.
* Включается на канале: `channel.Confirm(false)`; подписки на confirms (`channel.NotifyPublish`)
  и returns (`channel.NotifyReturn`) регистрируются до публикации.
* **Положительный confirm (`basic.ack`) возможен и для unroutable сообщения** — если ни одна
  очередь не подошла. Публиковать с `mandatory=true`, обрабатывать `basic.return` вместе с confirms:
  для unroutable сообщения return предшествует confirm; один ack не доказывает доставку в очередь.
* Для сохранности нужны durable queue и persistent message; для persistent сообщения в durable queue
  ack означает запись на диск, для quorum queue — подтверждение кворумом реплик.
* При nack/timeout/обрыве соединения публикацию повторяют; потерянный confirm может привести к дублю.
  Без confirms producer не знает, принял ли брокер сообщение.

</details>

---

### Вопрос №7: [ Как реализовать delayed messages (отложенная доставка)? ]

<details>
<summary>Ответ</summary>

Способ 1 — **DLX + TTL** (классический):
* Сообщение публикуется в очередь с TTL.
* По истечении TTL сообщение идёт в DLX, который маршрутизирует в целевую очередь.
* Минус: TTL проверяется для head of queue — длинные задержки для сообщений позади.

Способ 2 — **rabbitmq_delayed_message_exchange** (plugin):
* Exchange с типом `x-delayed-message`.
* Заголовок `x-delay` задаёт задержку в миллисекундах.
* Брокер хранит сообщение и доставляет через заданное время.

</details>

---

### Вопрос №8: [ Чем classic queue отличается от quorum queue? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

* **Classic queue** — оригинальная реализация. Может быть durable, mirrored (HA).
  Persistence — на диск (опционально). Mirroring устарело (deprecated с 2021, эпоха RabbitMQ 3.9; удалено в 4.0).
* **Quorum queue** — основана на Raft-консенсусе. Реплицируется на несколько узлов автоматически.
  Data safety выше, чем у mirrored classic queues. Перезапись при падении узла автоматическая.
  Не поддерживает some features (priority, TTL per message — ограниченно).
* **Stream** (с RabbitMQ 3.9) — append-only log (как Kafka). Поддерживает replay, multiple consumers.
* Рекомендация: новые системы — quorum queues или streams, classic queues — для обратной совместимости.

</details>

---

</details>

---

## 3. Apache Kafka

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Опишите архитектуру Kafka ]

<details>
<summary>Ответ</summary>

* **Producer** → **Topic** (разбит на **partitions**) → **Broker** → **Consumer Group**
* **Topic** — логический канал сообщений. Каждое сообщение — key + value + timestamp + headers.
* **Partition** — упорядоченный, append-only лог. Каждое сообщение имеет **offset** (позиция в partition).
* **Broker** — узел кластера Kafka. Партиции реплицируются между брокерами.
* **Replica**: один partition имеет **leader** (обрабатывает чтение/запись) и **followers** (реплики).
* **Consumer Group** — группа consumer'ов, каждый читает свой набор partition'ов.
  Если consumer падает, partition'ы перераспределяются (**rebalance**).
* **ZooKeeper / KRaft** — координация кластера: KRaft стал production-ready в Kafka 3.3,
  а начиная с Kafka 4.0 ZooKeeper полностью удалён (остался только KRaft-режим).

</details>

---

### Вопрос №2: [ Как Kafka гарантирует порядок сообщений? ]

<details>
<summary>Ответ</summary>

* Порядок гарантирован **только в пределах одного partition**.
* Сообщения с одинаковым key попадают в один partition (через murmur2(key) % num_partitions).
* Если порядок важен для всех сообщений топика — использовать 1 partition (но нет параллелизма).
* Если порядок важен для конкретной сущности (например, order_id) — использовать key = order_id,
  тогда все события этого заказа идут в один partition по порядку.

</details>

---

### Вопрос №3: [ Что такое consumer offset и offset commit? ]

<details>
<summary>Ответ</summary>

* **Offset** — позиция consumer'а в partition. Хранится в специальном topic `__consumer_offsets`.
* **Commit** — сохранение текущего offset. После commit'а broker "знает", что сообщения до этого offset обработаны.
* **Auto-commit** (`enable.auto.commit=true`) — периодически (по умолчанию 5 сек) commit'ит последний offset.
  Риск: при падении между обработкой и auto-commit'ом — дублирование (at-least-once).
* **Manual commit** — consumer сам вызывает `CommitSync()` или `CommitAsync()` после обработки.
  Позволяет контролировать exactly-once (с транзакциями) или хотя бы минимизировать дубли.
* **`auto.offset.reset`** — что делать, если offset не найден: `earliest` (с начала) или `latest` (только новые).

</details>

---

### Вопрос №4: [ Что такое retention и compaction в Kafka? ]

<details>
<summary>Ответ</summary>

* **Retention** (по умолчанию 7 дней) — Kafka хранит сообщения определённое время или до лимита размера.
  `retention.ms`, `retention.bytes`.
* **Compaction** — для каждого key сохраняется только последнее значение.
  Старые версии удаляются фоновым процессом (log compactor).
  Полезно для state (current balance, latest config).
* Можно комбинировать: retention + compaction (`delete,compact`).
* Segment — файл partition'а. Compaction работает по сегментам, не затрагивая active segment.

</details>

---

### Вопрос №5: [ Что такое rebalance и как минимизировать простои? ]

<details>
<summary>Ответ</summary>

* **Rebalance** — перераспределение partition'ов между consumer'ами в группе.
  Происходит при: добавлении/удалении consumer'а, изменении количества partition'ов.
* Во время rebalance — **stop the world** для consumer group: никто не читает.
* **Cooperative rebalance** (с Kafka 2.4) — incremental: только затронутые partition'ы приостанавливаются.
  В Kafka 4.0 GA — новый протокол ребаланса (KIP-848): назначение партиций на стороне сервера,
  устраняет stop-the-world ребалансы. Но он **выключен по умолчанию** (`group.protocol=classic` остаётся
  дефолтом), а клиентские ассигноры пока не реализованы.
* **Static group membership** (`group.instance.id`) — consumer сохраняет свою идентичность при кратковременных
  отключениях, rebalance не происходит (до истечения `session.timeout.ms`).

</details>

---

### Вопрос №6: [ Как достичь exactly-once в Kafka? ]

<details>
<summary>Ответ</summary>

* **Idempotent producer** (`enable.idempotence=true`, по умолчанию с Kafka 3.0):
  producer получает PID (Producer ID), и каждая запись получает sequence number.
  Broker дедуплицирует ретраи в пределах одной сессии.
* **Transactional API** — для exactly-once между consume → process → produce:
  ```java
  producer.initTransactions();
  producer.beginTransaction();
  // consume from input topic, process, produce to output topic
  producer.sendOffsetsToTransaction(offsets, consumerGroupMetadata);
  producer.commitTransaction();
  ```
* `isolation.level=read_committed` — consumer видит только закоммиченные транзакционные сообщения.
* Ограничения: exactly-once только в пределах Kafka (consume → produce + commit offset).
  Для внешних систем (запись в БД) нужен idempotent consumer или outbox pattern.

</details>

---

### Вопрос №7: [ Что такое ISR (In-Sync Replicas)? ]

<details>
<summary>Ответ</summary>

* **ISR** — набор реплик partition'а, которые "догнали" leader'а (записали все данные до последнего offset).
* `min.insync.replicas` (обычно 2) — минимальное количество ISR для записи.
  Если ISR < min.insync.replicas, producer с `acks=all` получит ошибку (NotEnoughReplicas).
* Если follower отстаёт (`replica.lag.time.max.ms`, по умолчанию 30 сек), он исключается из ISR.
* **acks**:
  - `acks=0` — не ждём подтверждения (fire and forget).
  - `acks=1` — ждём подтверждения от leader.
  - `acks=all` (или `-1`) — ждём подтверждения от всех ISR (максимальная надёжность).

</details>

---

### Вопрос №8: [ Как работает partition assignment (стратегии)? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

* **RangeAssignor** (по умолчанию) — partition'ы распределяются по диапазонам.
  Может быть неравномерным при нескольких топиках.
* **RoundRobinAssignor** — круговое распределение всех partition'ов по consumer'ам. Более равномерно.
* **StickyAssignor** — старается сохранить текущее распределение при rebalance (минимум изменений).
* **CooperativeStickyAssignor** — sticky + cooperative rebalance (без stop-the-world).
* Настройка: `partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor`.

</details>

---

### Вопрос №9: [ Что будет если партиций больше, чем консьюмеров (в группе)? А если наоборот — консьюмеров больше, чем партиций? ]

<details>
<summary>Ответ</summary>

* If there're more partitions than consumers in a group, some consumers will consume data from more than one partition.
* If there're more consumers in a group than partitions, some consumers will get no data.
* If you add new consumer instances to the group, they will take over some partitions from old members.
* If you remove a consumer from the group (or the consumer dies), its partition will be reassigned to other member.
* You CANNOT have multiple consumers (in a consumer group) to consume data from a single partition.
    However, if there're more than one consumer group, the same partition can be consumed by one (and only one) consumer
    in each consumer group.

</details>

---

### Вопрос №10: [ Как выбрать количество партиций? Проблемы ]

<details>
<summary>Ответ</summary>

* Partitions allow users to parallelize topics, meaning data for any topic can be divided over multiple brokers.
    In other words, the greater the parallelization the greater the throughput.
    On the other hand a partition will always be consumed completely by a single consumer.
    Therefore, in general, the more partitions there are in a Kafka cluster, the higher the throughput one can achieve.
* increasing partition count simultaneously increases the number of open server files and leads to increased replication latency.
* Partitions = Desired Throughput / Partition Speed (in general single partition for a single Kafka topic runs at 10 MB/s)

</details>

---

### Вопрос №11: [ Что такое replication factor? ]

<details>
<summary>Ответ</summary>

* Replication factor — количество реплик каждого partition на разных брокерах.
  По умолчанию `default.replication.factor=1` (рекомендуется установить 3 для продакшена).
  Например, при RF=3 каждая партиция имеет 3 реплики (лидер + 2 фолловера);
  для 6 партиций это 18 реплика-партиций, распределённых по брокерам.

</details>

---

### Вопрос №12: [ Изменение количества партиций в рантайме ]

<details>
<summary>Ответ</summary>

* 1. Is it possible to change the number of partitions without tearing down the cluster?
  Yes kafka supports increasing the number of partitions at runtime but doesn't support decreasing number of partitions due to its design.
* 2. And is it possible to do that without bringing down the topic?
  Yes provided you are increasing partitions.
* 3. Will adding/removing partitions automatically take care of redistributing messages across the new partitions?
  As mentioned earlier removing partitions is not supported.
  When you increase the number of partitions, the existing messages will remain in the same partitions as before
  only the new messages will be considered for new partitions (also depending on you partitioner logic).
  Increasing the partitions for a topic will trigger a cluster rebalance, where in the consumers and producers will get notified
  with the updated metadata of the topics. Producers will start sending messages to new partitions after receiving updated metadata
  and consumer rebalancer will redistribute the partitions among the consumers groups and resume consumption from the last committed offset.
  All this will happen under the hood, so you wont have to do any changes at client side.

</details>

---

### Вопрос №13: [ Клиенты Kafka в Go: sarama, segmentio/kafka-go, franz-go, confluent-kafka-go — что выбрать? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **franz-go** — самая полная реализация на чистом Go: поддержка свежих KIP (включая новые
  протоколы транзакций и ребалансов), продуманный API, высокая производительность.
  Де-факто выбор по умолчанию для новых проектов.
* **segmentio/kafka-go** — минималистичный клиент с `Reader`/`Writer`, близкий по духу к
  стандартной библиотеке; удобен для простых сценариев, но функциональность отстаёт.
* **IBM/sarama** — исторически самый популярный; активно развивался, но имеет репутацию
  сложного в отладке (тонкие места с сессиями/ребалансами). Часто встречается в легаси.
* **confluent-kafka-go** — обёртка над `librdkafka` на C через cgo: максимум фич и боевой
  опыт, но зависимость от C-библиотеки, сложнее сборки и кросс-компиляции
  (см. вопрос про cgo в go-questions).
* **Критерии выбора**: нужные протокольные фичи (транзакции, идемпотентный продюсер,
  поддержка версии брокера), удобство работы с consumer group, операционный опыт команды,
  чистый Go против cgo в инфраструктуре сборки.
* **Независимо от клиента**: идемпотентность потребителя (см. вопрос про идемпотентный
  consumer), явные коммиты оффсетов, мониторинг consumer lag (см. вопрос про Consumer Lag).

</details>

---

</details>

---

## 4. NATS и другие брокеры

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Что такое NATS и в чём его особенности? ]

<details>
<summary>Ответ</summary>

* NATS — брокер сообщений на Go от Synadia. Очень лёгкий, быстрый (µs latency), бинарный протокол.
* **Core NATS** — fire-and-forget pub/sub, без persistence (если нет подписчика — сообщение теряется).
* **JetStream** — persistence с потоками (streams), consumer'ами, гарантиями доставки (at-least-once, exactly-once).
* **Subjects** — вместо exchanges/queues. Подписка с wildcard: `orders.*.created`, `orders.>`.
* **Queue groups** — competing consumers (как queue в RabbitMQ).
* Плюсы: простота, скорость, малый footprint (один бинарник), встроенная observability.
* Минусы: меньше экосистемы чем у Kafka/RabbitMQ, меньше коннекторов.

</details>

---

### Вопрос №2: [ Что такое NATS request-reply? ]

<details>
<summary>Ответ</summary>

* Встроенный паттерн request-response без необходимости отдельной reply-topic.
* `nats.Request("user.get", payload, timeout)` — публикует сообщение с reply-to subject,
  который генерируется автоматически, и ждёт ответа.
* Responder подписывается на subject и отвечает через `msg.Respond(data)`.
* Поддерживает scatter-gather: `nats.RequestMsg` с `nats.MaxReplies(N)`.

</details>

---

### Вопрос №3: [ Когда выбрать Redis Streams вместо отдельного брокера? ]

<details>
<summary>Ответ</summary>

* **Redis Streams** (с Redis 5.0) — log-структура в Redis. Consumer groups, pending entries list (PEL), ack.
* Подходит, когда:
  - Уже есть Redis в стеке — не нужен отдельный брокер.
  - Небольшой объём сообщений (Redis in-memory, persistence опциональна).
  - Нужна низкая задержка.
* Не подходит для: больших объёмов (память Redis), долгого хранения, сложной маршрутизации.
* `XADD`, `XREAD`, `XREADGROUP`, `XACK`, `XPENDING`, `XAUTOCLAIM` — основные команды.

</details>

---

### Вопрос №4: [ Что такое SQS и его особенности? ]

<details>
<summary>Ответ</summary>

* **Amazon SQS** — managed очередь от AWS. Нет отдельного exchange/concept, просто queue.
* **Standard queue** — no ordering, at-least-once (возможны дубли), практически unlimited throughput.
* **FIFO queue** — ordering, exactly-once обработка (дедупликация по `DeduplicationId` или
  content-based, в пределах 5-минутного окна), до 3000 msg/s (TPS, с batching — больше).
  Сама доставка остаётся at-least-once: при падении consumer'а сообщение может быть доставлено повторно.
* **Visibility timeout** — после чтения сообщение становится невидимым для других consumer'ов на N секунд.
  Если не удалено (DeleteMessage) — снова видимым.
* **Dead Letter Queue** — настраивается через RedrivePolicy (после N неудачных приёмов).
* **Long polling** — `WaitTimeSeconds` до 20 сек, снижает пустые ответы.

</details>

---

</details>

---

## 5. Гарантии доставки и надёжность

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Что такое at-most-once, at-least-once, exactly-once? ]

<details>
<summary>Ответ</summary>

* **At-most-once** — сообщение может быть доставлено 0 или 1 раз. Потери возможны, дубли нет.
  Реализация: auto-ack, fire-and-forget.
* **At-least-once** — сообщение доставляется 1 или более раз. Дубли возможны, потерь нет.
  Реализация: manual ack после обработки, retries.
  Самый распространённый вариант в распределённых системах.
* **Exactly-once** — эффект обработки фиксируется ровно один раз в оговорённых границах,
  например Kafka transactions в пределах Kafka; это не гарантия единственной доставки.
  **На практике** — outbox с at-least-once публикацией + транзакционно идемпотентный consumer
  дают effectively-once эффект в его БД, но не автоматически во внешних системах.

</details>

---

### Вопрос №2: [ Как сделать consumer идемпотентным? ]

<details>
<summary>Ответ</summary>

* **Уникальный ID сообщения** (`message_id`, `event_id`) должен сохраняться при повторной доставке.
* **В БД**: inbox/`processed_events` с unique constraint на ID. В **одной транзакции** вставить ID
  (`INSERT ... ON CONFLICT DO NOTHING`), проверить, вставлена ли строка, и только для нового ID
  выполнить бизнес-изменение. При конфликте — пропустить уже зафиксированную обработку;
  при ошибке — rollback обоих изменений. Ack брокеру — **только после commit**.
* Отдельно сохранённый маркер **до** эффекта может потерять обработку при падении; **после** —
  допускает дубли. Состояния `INPROGRESS`/`completed` сами по себе не закрывают это окно.
* **Внешние эффекты** (платёж, email) не входят в транзакцию БД: нужен стабильный idempotency key
  провайдера и reconciliation при неизвестном исходе. Без поддержки провайдера обещать exactly-once нельзя.
* **Redis SETNX** — лишь объединение конкурирующих попыток: `SET ... NX EX`, уникальный токен владельца,
  атомарные compare-and-delete/renew и продуманный срок жизни. Потеря/истечение lock не должны обходить
  дедупликацию в БД или у провайдера; Redis сам по себе не обеспечивает корректность обработки.
* **Версионирование** — проверка версии и изменение сущности должны быть одним условным UPDATE.
  Возврат сохранённого результата при повторе — идемпотентность, а не компенсация.

</details>

---

### Вопрос №3: [ Что такое Outbox Pattern? ]

<details>
<summary>Ответ</summary>

* Проблема: обновить БД и отправить сообщение в брокер атомарно нельзя (две разные системы).
* Решение: в одной транзакции БД — обновить данные **и** записать сообщение в **outbox** таблицу.
* **Кто создаёт `event_id`:** сервис, в котором произошло бизнес-событие, например генерирует UUID.
  ID сохраняется вместе с событием в outbox под `PRIMARY KEY`/`UNIQUE`, попадает в сообщение
  и без изменений используется relay при всех повторных публикациях и consumer при дедупликации.
  Генерировать новый ID на каждой отправке или использовать broker delivery tag нельзя.
* У нового события — новый ID. Повтор команды клиента не должен создавать то же событие с новым ID:
  дедупликация исходной команды по idempotency key должна быть частью бизнес-транзакции.
  ID команды и ID события — не обязательно одно и то же: одна команда может породить несколько событий.
* Отдельный relay (Outbox Publisher / CDC) публикует запись в брокер. Для RabbitMQ отметку
  «отправлено» ставить после положительного confirm с учётом `mandatory`/returns и нужной durability.
  Если relay упал после публикации, но до отметки, или потерял confirm — публикация повторится:
  сохранять `event_id`, а consumer делать идемпотентным.
* **CDC** (Change Data Capture, например Debezium) — альтернатива поллингу: читает WAL БД и публикует изменения.

В упрощённой схеме ниже `$1` — созданный сервисом ID события, передаваемый как параметр запроса.
Бизнес-проверки операции и дедупликация команды выполняются в этой же транзакции.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
INSERT INTO outbox (event_id, aggregate_type, aggregate_id, payload, created_at)
    VALUES ($1, 'account', '1', '{"amount": 100}', now());
COMMIT;
```

* **Polling relay с несколькими воркерами:**
  1. В короткой транзакции выбрать ограниченную пачку готовых записей через `FOR UPDATE SKIP LOCKED`,
     назначить каждой состояние `processing`, новый `owner_token` захвата и `lease_until`, затем commit.
     Истёкшие `processing` тоже должны возвращаться в обработку — отдельным восстановлением
     либо непосредственно запросом захвата; одного условия `status = 'pending'` недостаточно.
  2. Публиковать **после commit**, не удерживая блокировки БД во время сетевого ожидания.
     Ограничить параллелизм и timeout; размер пачки должен позволять уложиться в lease.
     Для долгой работы продлевать ещё действующий lease только с проверкой текущего токена владельца.
  3. После подтверждённой публикации отметить `sent` условным UPDATE по `event_id`,
     `status = 'processing'`, своему `owner_token` и ещё действующему lease; проверить число изменённых строк.
     При потере владения не менять состояние чужой попытки. Сам токен защищает запись в БД,
     но не запрещает старому воркеру отправить сообщение в брокер: дубликаты всё ещё возможны.
  4. При ошибке сохранять число попыток и время следующей попытки с exponential backoff и jitter;
     менять состояние только при сохранённом владении. После истечения lease запись подберёт
     восстановление. Неисправимые события помещать в состояние ошибки с оповещением и возможностью
     ручного повтора, а не молча удалять. Повтор сохраняет исходный `event_id`.
  5. При остановке перестать брать новые записи и ограниченно дождаться текущих публикаций.
     Неизвестный исход (timeout, потерянный confirm, падение) означает возможную повторную отправку,
     а не доказательство, что сообщение не было доставлено.
* **Порядок:** `ORDER BY created_at` вместе с `SKIP LOCKED` не гарантирует порядок публикации:
  несколько воркеров и retry могут отправить более позднее событие раньше. Для порядка внутри
  одной сущности нужны версия/sequence, сериализация её публикаций и соответствующий ключ
  партиции брокера. Следующее событие сущности нельзя отправлять раньше подтверждения предыдущего,
  но поздние дубли всё равно требуют дедупликации и проверки версии у consumer.
* Для самого механизма захвата см. «Задача №2: Построить job queue на PostgreSQL»
  в [разделе PostgreSQL](sql-postgres-questions.md); для outbox дополнительно нужны правила
  владения, подтверждения публикации и повторов, описанные выше.
* **Мониторинг:** возраст самого старого неотправленного события, размер очереди, число retry,
  истёкших lease и событий в состоянии ошибки.
* Плюсы: атомарность **только бизнес-данных и outbox внутри БД**, нет dual-write проблемы;
  общей транзакции с брокером нет, publisher confirms её не создают.
* Минусы: задержка (polling), сложность инфраструктуры (CDC), возможные дубли от relay.

</details>

---

### Вопрос №4: [ Что такое Saga pattern и чем отличается от распределённых транзакций? ]

<details>
<summary>Ответ</summary>

* **Saga** — последовательность локальных транзакций, где каждая следующая компенсируется при неудаче.
* В отличие от 2PC, нет глобальной блокировки — eventual consistency.
* **Choreography** — каждый сервис публикует событие, следующий сервис реагирует. Децентрализовано.
* **Orchestration** — центральный оркестратор координирует шаги. Более управляемо, но single point.
* Пример (заказ): create_order → reserve_payment → ship → confirm. При ошибке: cancel_ship → refund → cancel_order.
* Компенсация — не откат, а обратное действие (нельзя "undo" отправленный email, можно отправить ещё один).

</details>

---

### Вопрос №5: [ Что делать при образовании poison message (poison pill)? ]

<details>
<summary>Ответ</summary>

* **Poison message** — сообщение, которое вызывает ошибку при каждой попытке обработки.
  Consumer постоянно его перечитывает (requeue → fail → requeue).
* Решения:
  - **DLX / DLQ** — после N неудач переместить в dead-letter очередь для ручного разбора.
  - **Max delivery count** — ограничение попыток (RabbitMQ: x-delivery-limit в quorum queues, Kafka: нет встроенного лимта — приложение управляет через commit offset + retry topics).
  - **Логирование и пропуск** — залогировать ошибку, ack'нуть сообщение (потеря данных, но разблокировка очереди).
  - **Retry с backoff** — exponential backoff через delayed queues (RabbitMQ: TTL + DLX, Kafka: retry topics).
* Важно: метрики на redelivery count, алерты при росте DLQ.

</details>

---

### Вопрос №6: [ Как обеспечить порядок доставки при ретраях? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Проблема**: сообщение A не обработалось (в ретрае), B обработалось успешно → нарушение порядка.
* **Решение 1: Sequential processing** — 1 partition = 1 consumer, ack обязателен перед следующим.
  Минус: poison pill блокирует весь partition.
* **Решение 2: Retry topics / queues** — отдельная очередь для ретраев, после успешного ретрая возврат в основную.
  Порядок может нарушаться, но компенсируется idempotency.
* **Решение 3: Key-based ordering + sharded retry** — retry queue с тем же partition key,
  сообщения того же key идут в тот же shard.
* На практике: чаще выбирают idempotent consumer + at-least-once, порядок не гарантируют глобально,
  только per-key/per-partition.

</details>

---

</details>

---

## 6. Паттерны и антипаттерны

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Назовите основные паттерны использования брокеров ]

<details>
<summary>Ответ</summary>

* **Work Queue (Competing Consumers)** — несколько consumer'ов делят одну очередь, балансировка нагрузки.
* **Pub/Sub (Fan-out)** — каждое сообщение доставляется всем подписчикам.
* **Request/Reply** — producer отправляет сообщение с reply-to, consumer отвечает на указанный топик/очередь.
* **Routing** — сообщения маршрутизируются по ключу (RabbitMQ topic exchange, Kafka partition key).
* **Priority Queue** — сообщения с высоким приоритетом обрабатываются первыми (RabbitMQ x-max-priority).
* **Dead Letter Queue** — сообщения, которые не удалось обработать, идут в DLQ.
* **Delayed Delivery** — отложенная доставка (TTL + DLX, или delayed_message_exchange plugin).
* **Outbox Pattern** — атомарная запись в БД + сообщение в брокер.
* **Saga** — распределённая транзакция через последовательность событий + компенсаций.

</details>

---

### Вопрос №2: [ Назовите антипаттерны работы с брокерами ]

<details>
<summary>Ответ</summary>

* **Long-running consumers** — обработка сообщения занимает минуты → blocking, prefetch заполнен, throughput падает.
  Решение: разбить на этапы, использовать scheduled tasks.
* **Синхронный RPC через broker для всего** — broker не замена REST для request-response с малой задержкой.
* **Большие сообщения** — мегабайты в сообщении тормозят брокер и consumer'ов. Решение: Claim Check
  (сообщение содержит ссылку на S3/БД, данные — отдельно).
* **Игнорирование idempotency** — дубли при at-least-once приводят к двойным списаниям, отправкам email и т.д.
* **Один топик для всего** — всё в один topic/partition → нет параллелизма, сложный routing.
* **Auto-commit без понимания** — потеря сообщений или дубли.
* **Нет DLQ** — poison message блокирует очередь бесконечно.
* **Нет мониторинга lag** — consumer отстаёт, данные устаревают (Kafka consumer lag).

</details>

---

### Вопрос №3: [ Что такое Consumer Lag и как с ним бороться? ]

<details>
<summary>Ответ</summary>

* **Consumer Lag** = `log_end_offset - consumer_committed_offset` — сколько сообщений consumer не обработал.
* Причины:
  - Consumer медленный (дорогая обработка).
  - Producer быстрее, чем consumer.
  - Мало partition'ов (мало параллелизма в consumer group).
  - Rebalance storm (постоянные rebalance).
* Решения:
  - Увеличить количество partition'ов и consumer'ов.
  - Оптимизировать обработку (batching, async I/O).
  - Auto-scaling consumer'ов по lag (KEDA, Kubernetes HPA на метрику lag).
  - Объединить мелкие сообщения в батчи (batch producer).
* Мониторинг: `kafka-consumer-groups --describe`, Burrow, Kafka UI, Prometheus exporter.

</details>

---

### Вопрос №4: [ Как организовать retry с exponential backoff через брокер? ]

<details>
<summary>Ответ</summary>

* Паттерн **Retry Topology** (RabbitMQ):
  - Main queue → при ошибке → retry queue с TTL (например, 5 сек) → DLX → back to main queue.
  - При повторной ошибке → retry queue с бóльшим TTL (30 сек) → и т.д.
  - После N попыток → DLQ.
* В Kafka: отдельные retry topics (`main`, `retry-5s`, `retry-30s`, `dlq`).
  Consumer, не обработавший сообщение, пишет в retry topic с нужной задержкой.
* **Backoff**: `delay = base_delay * 2^attempt` (с jitter для избежания thundering herd).

</details>

---

### Вопрос №5: [ Что такое Claim Check pattern? ]

<details>
<summary>Ответ</summary>

* Большие сообщения (фото, видео, документы) не кладутся в брокер целиком.
* **Claim Check**: данные сохраняются во внешнем хранилище (S3, БД), а в брокер отправляется только
  ссылка (claim check) — ID или URL.
* Consumer читает ссылку, скачивает данные из хранилища, обрабатывает.
* Плюсы: брокер не перегружен, быстрая доставка метаданных, данные можно lifecycle'ить отдельно.

</details>

---

</details>

---

## 7. Сравнение брокеров: когда что выбрать

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Kafka vs RabbitMQ — принципиальная разница в архитектуре ]

<details>
<summary>Ответ</summary>

| Критерий | Kafka | RabbitMQ |
|----------|-------|----------|
| **Модель** | Append-only log (partitioned) | Queue + exchanges (AMQP) |
| **Consumers** | Pull-based, offset-based | Push-based (или pull с basic.get) |
| **Порядок** | Внутри partition (по offset) | Внутри очереди (FIFO) |
| **Replay** | Да (перемотать offset) | Нет (сообщение удаляется после ack) |
| **Retention** | По времени/размеру (дни, недели) | До ack (или TTL для lazy queues) |
| **Throughput** | Миллионы msg/сек | Десятки-сотни тысяч msg/сек |
| **Latency** | ~5–10 мс (batching) | ~1–5 мс |
| **Размер сообщения** | До 1 МБ (рекомендуется), max ~1 ГБ | Неограничен (но влияет на память) |
| **Маршрутизация** | По topic + partition key | Rich routing (direct, fanout, topic, headers) |
| **Репликация** | ISR (in-sync replicas) | Classic / Quorum queues |
| **Protocol** | Kafka protocol (TCP) | AMQP 0-9-1, AMQP 1.0, MQTT, STOMP |

* **Kafka** — log-based: сообщения хранятся на диске, consumer'ы читают по offset. Несколько
  consumer'ов могут перечитывать одни и те же данные (replay).
* **RabbitMQ** — queue-based: сообщение удаляется из очереди после ack. Routing через
  exchanges — гибкая доставка конкретным consumer'ам.

</details>

---

### Вопрос №2: [ Когда выбрать Kafka, а когда RabbitMQ? ]

<details>
<summary>Ответ</summary>

* **Выбирайте Kafka, если:**
  - **Event streaming** — потоковая обработка событий (CDC, analytics, ML pipelines).
  - **Replay** — нужно пере-прочитать историю событий (new consumer, bug fix, backfill).
  - **High throughput** — сотни тысяч или миллионы событий в секунду.
  - **Multiple consumers** — несколько независимых consumer groups читают один поток.
  - **Event sourcing** — события как source of truth.
  - **Log aggregation** — сбор логов/метрик с множества источников.
  - **Long retention** — данные хранятся днями/неделями.

* **Выбирайте RabbitMQ, если:**
  - **Task queue** — фоновые задачи (email, обработка файлов, генерация отчётов).
  - **Complex routing** — сообщения идут разным consumer'ам по правилам (topic exchange,
    headers exchange).
  - **Low latency** — каждый запрос должен быть обработан быстро.
  - **Request/reply** — RPC поверх очереди (reply-to queue).
  - **TTL + DLQ** — автоматическое истечение и dead-lettering.
  - **Малый-средний объём** — тысячи до сотен тысяч сообщений/сек.
  - **Transactional outbox** — relay публикует с confirms, `mandatory`/returns и durable queue +
    persistent messages; атомарны только данные и outbox в БД, не публикация в RabbitMQ.
    Дубли relay требуют идемпотентного consumer; этот паттерн применим и к Kafka.

* **Оба подходят, но с нюансами:**
  - **Microservices events** — Kafka для event streaming между сервисами, RabbitMQ для
    command/task dispatching.
  - **Order processing** — RabbitMQ для команд (create, pay, ship), Kafka для event log
    (order_created, order_paid — для analytics).

</details>

---

### Вопрос №3: [ Kafka vs RabbitMQ: гарантии доставки и exactly-once ]

<details>
<summary>Ответ</summary>

| | Kafka | RabbitMQ |
|--|-------|----------|
| **At-most-once** | Да (auto-commit, no ack) | Да (auto-ack) |
| **At-least-once** | Да (manual commit после обработки) | Да (manual ack) |
| **Exactly-once** | Да (idempotent producer + transactions) | Нет (только через app-level idempotency) |
| **Producer confirm** | acks=all + enable.idempotence=true | publisher confirms |
| **Consumer ack** | commit offset (group coordinator) | basic.ack (broker удаляет сообщение) |
| **Ordering** | Внутри partition (по offset) | Внутри очереди (при 1 consumer) |
| **Poison messages** | Без DLQ — consumer stuck (или skip) | DLX (dead-letter exchange) — автоматический |

* **Kafka exactly-once:** idempotent producer (дедупликация по PID + sequence number) +
  transactional producer (atomic write across partitions + commit offset).
* **RabbitMQ:** нет exactly-once — consumer может упасть после обработки, но до ack →
  повторная доставка. Нужна application-level idempotency (idempotency key в БД).

</details>

---

### Вопрос №4: [ Kafka vs RabbitMQ: масштабирование и производительность ]

<details>
<summary>Ответ</summary>

* **Kafka:**
  - Горизонтальное масштабирование через **partitions** — больше partitions → больше параллелизма.
  - Consumer scale: 1 consumer = 1 partition (больше consumers, чем partitions — idle).
  - Batch writes (линейная запись на диск — очень быстро, sequential I/O).
  - Zero-copy (sendfile) — данные идут с диска в network, минуя user space.
  - Compression (snappy, lz4, zstd) — меньше места и сети.
  - **Throughput:** 100K–1M+ msg/сек на один broker (зависит от размера и batching).

* **RabbitMQ:**
  - Масштабирование через **sharded queues / Quorum queues** + cluster nodes.
  - Consumer scale: сколько угодно (по round-robin).
  - Single-threaded queue (по умолчанию) — один queue = один core.
  - **Lazy queues** — сообщения на диске, а не в RAM (для больших очередей).
  - **Multiple queues** для параллелизма (partitioning по routing key).
  - **Throughput:** 20K–100K msg/сек на node (зависит от размера и routing).

* **Вертикальное vs горизонтальное:**
  - Kafka: добавь broker → rebalance partitions (online).
  - RabbitMQ: добавь node → cluster, но очереди не shard'ятся автоматически (нужно
    создавать новые очереди или использовать consistent hash exchange).

</details>

---

### Вопрос №5: [ NATS vs Kafka vs RabbitMQ — где место NATS? ]

<details>
<summary>Ответ</summary>

* **NATS** — lightweight, высокопроизводительный message bus (Go).
  - **Core NATS:** at-most-once, fire-and-forget, ~1M+ msg/сек на один сервер.
  - **JetStream:** persistence, at-least-once, streams (аналог Kafka topics), consumers,
    exactly-once на стороне публикации (дедупликация по `Nats-Msg-Id` в пределах окна);
    доставка потребителю — at-least-once, поэтому end-to-end нужна идемпотентность consumer'а.
  - **Subjects** вместо queues/topics — wildcard подписки (`orders.*.created`).
  - **Leaf nodes** — bridge между кластерами NATS (edge computing, multi-region).

| Критерий | NATS (Core) | NATS JetStream | Kafka | RabbitMQ |
|----------|-------------|----------------|-------|----------|
| **Throughput** | 1M+ msg/сек | 100K+ msg/сек | 100K–1M | 20K–100K |
| **Latency** | <1 мс | ~1–5 мс | ~5–10 мс | ~1–5 мс |
| **Persistence** | Нет | Да | Да | Да |
| **Replay** | Нет | Да | Да | Нет |
| **Footprint** | ~10 МБ бинарник | То же | Тяжёлый (JVM) | Средний (Erlang) |
| **Complexity** | Очень простая | Простая | Высокая | Средняя |
| **Protocols** | NATS | NATS | Kafka | AMQP, MQTT, STOMP |

* **NATS Core:** для real-time messaging, IoT, service discovery, request/reply (RPC).
* **NATS JetStream:** замена Kafka для средних нагрузок, когда Kafka слишком сложна.
* **Kafka:** для heavy event streaming, big data pipelines, когда нужна экосистема
  (Kafka Connect, Streams, Schema Registry).
* **RabbitMQ:** для task queues, complex routing, AMQP-совместимость.

</details>

---

### Вопрос №6: [ AWS SQS/SNS/EventBridge vs self-hosted Kafka/RabbitMQ — когда что? ]

<details>
<summary>Ответ</summary>

* **Managed (SQS/SNS/EventBridge):**
  - **SQS** — queue (аналог RabbitMQ queue). Standard (at-least-once, no ordering) и FIFO
    (exactly-once обработка с дедупликацией, ordered, 3000 msg/сек).
  - **SNS** — pub/sub (аналог RabbitMQ fanout). Push to SQS, Lambda, HTTP, email.
  - **EventBridge** — event bus + rule-based routing (аналог RabbitMQ topic exchange, но
    с JSON content filtering).
  - Плюсы: no-ops, auto-scaling, pay-per-use.
  - Минусы: vendor lock-in, ограниченный throughput (SQS FIFO — 3000 msg/сек), latency
    выше (~10–50 мс).

* **Self-hosted (Kafka/RabbitMQ):**
  - Полный контроль: конфигурация, версии, плагины.
  - Выше throughput (Kafka — миллионы), ниже latency.
  - Multi-cloud / on-premise.
  - Минусы: нужно управлять (patching, monitoring, scaling, backup).

* **Когда managed:** стартапы, event-driven serverless, низкий-средний объём, нет команды
  для эксплуатации брокера.
* **Когда self-hosted:** high throughput, low latency, сложные routing patterns, on-premise,
  multi-cloud, необходимость replay (Kafka).

</details>

---

### Вопрос №7: [ Практический сценарий: выбор брокера для e-commerce платформы ]

<details>
<summary>Ответ</summary>

| Компонент | Брокер | Почему |
|-----------|--------|--------|
| Order events (analytics, ML) | Kafka | Replay, multiple consumers, high volume, long retention |
| Payment processing (task queue) | RabbitMQ / SQS | Command dispatch, low latency, DLQ для failed payments |
| Notification service (email, push) | RabbitMQ / SQS | Task queue, TTL, retry |
| Inventory updates (real-time) | NATS Core / Redis Pub/Sub | Low latency, fire-and-forget |
| CDC (DB → search index, cache) | Kafka (Debezium) | Ordered, persistent, replay for backfill |
| User activity tracking | Kafka | High volume, analytics pipeline (→ S3 → Spark/Athena) |
| Webhook delivery | RabbitMQ / SQS | Per-tenant queue, retry, DLQ, rate limiting |

* **Архитектура:**
  1. API → Kafka (order_created event) — для analytics, CDC, audit log.
  2. API → RabbitMQ/SQS (process_payment command) — для команд с retry/DLQ.
  3. Kafka → consumer → Elasticsearch (search index update).
  4. Kafka → consumer → Redis (cache invalidation/update).
  5. RabbitMQ/SQS → consumer → email/push notification.

* **Ключевой принцип:** Kafka — для **events** (факты, произошедшие в прошлом, для
  analytics/replay). RabbitMQ/SQS — для **commands** (запросы на выполнение действия, с
  retry и DLQ).

</details>

---

</details>

---
