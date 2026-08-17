# System Design — Вопросы и ответы для Senior

Раздел перенесён из основного README. Содержит вопросы по проектированию распределённых систем,
практические примеры с описанием подводных камней и способов их преодоления.

---

<details>
<summary><b><i>System Design (Проектирование систем)</i></b></summary>

---

### Вопрос №1: [ Что такое System Design? Из чего состоит типичное системное интервью? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* System Design — это интервью на умение проектировать распределённые системы заданного масштаба.
  Кандидату даётся открытая задача (например, «спроектируй Twitter»), и за 45–60 минут нужно:
  1. **Уточнить требования** (functional + non-functional: RPS, latency, availability, consistency);
  2. **Оценить масштаб** (back-of-the-envelope: RPS, объём данных, bandwidth);
  3. **Нарисовать high-level архитектуру** (клиенты → LB → сервисы → БД → кэш → очередь);
  4. **Углубить ключевые компоненты** (схема БД, API, кэширование, шардинг);
  5. **Обсудить trade-offs** (CAP, consistency vs availability, latency vs cost);
  6. **Identify bottlenecks** (SPoF, hotspots, race conditions).

* Ключевые концепции, которые проверяются:
  - Масштабирование (horizontal vs vertical, stateless vs stateful);
  - Балансировка нагрузки (L4/L7, алгоритмы);
  - Кэширование (cache-aside, write-through, eviction policies);
  - Шардинг и репликация БД;
  - Очереди сообщений и асинхронная обработка;
  - CAP/PACELC теорема, consistency models;
  - Rate limiting, circuit breaker, backpressure;
  - CDN, edge computing;
  - Мониторинг, observability (metrics, logs, traces).

</details>

---

### Вопрос №2: [ Сформулируй теорему CAP. Что такое PACELC? ]

<details>
<summary>Ответ</summary>

* **CAP теорема** (Brewer, 2000): распределённая система может одновременно обеспечить не более
  двух из трёх свойств:
  - **C (Consistency)** — все узлы видят одни и те же данные одновременно (linearizability);
  - **A (Availability)** — каждый запрос получает ответ (без гарантии актуальности);
  - **P (Partition tolerance)** — система работает при разрыве сети между узлами.

  Поскольку сетевые разделы (P) в реальности неизбежны, выбор сводится к **CP** или **AP**.
  - CP (например, HBase, MongoDB при определённых настройках) — при разделении отказывает в
    обслуживании для обеспечения консистентности;
  - AP (например, Cassandra, DynamoDB) — при разделении продолжает обслуживать, но данные могут
    расходиться (eventual consistency).

* **PACELC теорема** (Abadi, 2010): расширение CAP. Если есть разделение (P) → выбор между A и C;
  **иначе (E — else)**, при нормальной работе → выбор между L (latency) и C (consistency).
  То есть: `if P then (A or C) else (L or C)`.
  - Cassandra: PA/EL — при разделении предпочитает availability, при нормальной работе — low latency;
  - Spanner: PC/EC — при разделении отказывает, при нормальной работе — strong consistency;
  - MongoDB: PC/EC (с majority read concern).

</details>

---

### Вопрос №3: [ Горизонтальное vs вертикальное масштабирование. Когда какое? ]

<details>
<summary>Ответ</summary>

* **Вертикальное (scale up):** Увеличение ресурсов одного сервера (больше CPU, RAM, дисков).
  - Плюсы: просто, не требует изменения кода, нет проблем с консистентностью;
  - Минусы: есть предел (железо), downtime при апгрейде, SPOF;
  - Когда: малые нагрузки, stateful системы (БД), быстрые временные решения.

* **Горизонтальное (scale out):** Добавление новых серверов.
  - Плюсы: нет предела (теоретически), no downtime, отказоустойчивость;
  - Минусы: сложность (балансировка, консистентность, деплой), требует stateless-сервисов;
  - Когда: high-load, stateless-сервисы, микросервисы.

* **Практический подход:** Сначала vertical (пока просто и хватает), затем horizontal
  (когда упираемся в лимиты железа). БД обычно сначала scale up, потом sharding (что сложно).
  Stateless-сервисы — сразу scale out.

</details>

---

### Вопрос №4: [ Спроектируй URL Shortener (типа bit.ly) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Требования:**
  - Functional: короткий URL → редирект на оригинальный; создание короткого URL из длинного;
  - Non-functional: 100M URL, 10K RPS на чтение, 100 RPS на запись; latency < 50ms;
    availability 99.9%.

* **High-level архитектура:**
  ```
  Client → API Gateway → [Shorten Service] → DB (URL mappings)
                      → [Redirect Service] → Cache (Redis) → DB
  ```

* **Генерация короткого ключа:**
  - **Вариант 1 — base62 encoding:** `id` из auto-increment БД → base62 (a-zA-Z0-9).
    7 символов = 62^7 ≈ 3.5 триллиона комбинаций. Плюс: короткие, детерминированные.
    Минус: предсказуемые (можно скрейпить). Решение: добавить случайный offset или использовать
    reverse-proxy с уникальным ID.
  - **Вариант 2 — hash:** MD5/SHA длинного URL → взять первые 7 символов base62.
    Минус: коллизии. Решение: проверять коллизию в БД, добавлять salt.
  - **Вариант 3 — random:** генерировать случайные 7 символов, проверять уникальность.
    Минус: коллизии при росте (парадокс дней рождения).

* **Хранилище:** Ключ-значение (Redis для hot URLs, Cassandra/DynamoDB для постоянного хранения).
  Или PostgreSQL: `(short_url, original_url, created_at, user_id, expires_at)`.

* **Кэширование:** Redis LRU cache для популярных ссылок. 80/20 rule: 20% URL дают 80% трафика.
  Cache-aside: при редиректе сначала проверяем кэш, потом БД.

* **Подводные камни и решения:**

  | Проблема | Решение |
  |----------|---------|
  | **Hot URL** (вирусный линк, 10K RPS на один URL) | Кэш + CDN edge caching; multi-level cache (L1 in-memory, L2 Redis) |
  | **Predictable URLs** (скрейпинг) | Rate limiting per IP; случайные ключи; варьируемая длина |
  | **Срок жизни ссылок** | TTL в БД + background job для очистки истёкших |
  | **Аналитика кликов** | Асинхронная запись кликов в Kafka → batch processing → ClickHouse |
  | **Гео-распределение** | GeoDNS + CDN для редиректов; данные в regional DB с репликацией |
  | **SPOF кэша** | Redis Cluster (шардинг + реплики); fallback на БД при недоступности кэша |
  | **Консистентность при записи** | Idempotent API (одинаковый длинный URL → тот же короткий); распределённый lock или unique constraint |

</details>

---

### Вопрос №5: [ Спроектируй ленту новостей / News Feed (типа Twitter) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Требования:**
  - 300M DAU, 5000 RPS на постинг, 100K RPS на чтение ленты;
  - Latency ленты < 200ms; eventual consistency допустима.

* **Ключевая архитектурная дилемма: Pull vs Push (Fan-out)**

* **Pull (Fan-out on read):** При запросе ленты — собираем посты от всех followings в реальном времени.
  - Плюсы: нет лишней работы при постинге; актуальные данные; просто для sparse users;
  - Минусы: дорогой read (N запросов к followings); медленно для пользователей с 10K followings.
  - Подходит: для знаменитостей (много подписчиков, мало постов).

* **Push (Fan-out on write):** При постинге — копируем пост в pre-computed ленты всех followers.
  - Плюсы: O(1) чтение ленты; отлично для пользователей с < 1000 followings;
  - Минусы: fan-out на N followers = много записей; celebrity problem (1 пост → 50M записей).
  - Подходит: для обычных пользователей.

* **Hybrid (как Twitter/Instagram):**
  - Обычные пользователи → **Push** (fan-out на write в их ленты followers);
  - Знаменитости → **Pull** (при запросе ленты подмешиваются их последние посты);
  - Лента хранится в Redis (sorted set по timestamp) или Cassandra.

* **Архитектура:**
  ```
  Post API → Kafka → Fan-out Worker → [Redis: user_id → sorted set of post_ids]
  Feed API → Redis (pre-computed) + Pull(celebrity posts) → merge → return
  ```

* **Подводные камни и решения:**

  | Проблема | Решение |
  |----------|---------|
  | **Celebrity problem** (Justin Bieber → 100M followers, fan-out убивает БД) | Hybrid: celebrity = pull при чтении; порог followers > 10K |
  | **Stale feed** (eventual consistency) | Допустимо для соцсетей; timestamp-based merge при pull |
  | **Hot partition** (один пользователь с 50M followers) | Параллельный fan-out через Kafka partitioning |
  | **Удаление/редактирование постов** | Soft delete + фильтрация при рендере ленты; или tombstones в Redis |
  | **Ranking / алгоритмическая лента** | Pre-compute score (ML model) в фоне; store ranked feed отдельно |
  | **Pagination** | Cursor-based (post_id), не offset (offset медленный при вставках) |
  | **Гео-репликация** | Regional Redis clusters; cross-region replication для followings |

</details>

---

### Вопрос №6: [ Спроектируй систему мгновенных сообщений / Chat (типа WhatsApp/Telegram) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Требования:**
  - 100M DAU, 10B сообщений/день; real-time delivery; offline messages;
  - Group chats до 1000 участников; read receipts; typing indicators.

* **High-level архитектура:**
  ```
  Client ←→ WebSocket Gateway ←→ Chat Service ←→ Message Store (Cassandra)
                                 ↕
                              Redis (presence, session routing)
                                 ↕
                              Kafka (events, notifications)
  ```

* **Соединение:** WebSocket (persistent bidirectional) для real-time. Один gateway-сервер держит
  до 100K–1M WebSocket-соединений (Go/Netty/epoll). Сессионная информация (какой сервер держит
  соединение пользователя) — в Redis.

* **Маршрутизация сообщения:**
  1. Отправитель → WebSocket → Chat Service;
  2. Chat Service записывает сообщение в Message Store (Cassandra, partitioned by conversation_id);
  3. Для каждого получателя: проверка presence (онлайн?) через Redis;
  4. Если онлайн — отправка через WebSocket-сервер получателя (по session routing из Redis);
  5. Если офлайн — push notification (APNs/FCM) + сообщение ждёт в Message Store.

* **Хранилище сообщений:**
  - Cassandra: `conversation_id` как partition key, `message_id` (timeuuid) как clustering key;
  - Сообщения immutable (нельзя редактировать — можно только добавить edit-запись);
  - TTL для авто-удаления (например, исчезающие сообщения).

* **Подводные камни и решения:**

  | Проблема | Решение |
  |----------|---------|
  | **Message ordering** (гарантия порядка в групповом чате) | Vector clocks / Lamport timestamps; server-assigned monotonic sequence per conversation |
  | **At-least-once delivery** (дубликаты) | Idempotency key (client_message_id); dedup на сервере |
  | **Presence spam** (миллионы presence-обновлений) | Debounce (обновлять presence раз в 30с); lazy presence (только когда кто-то смотрит) |
  | **Group fan-out** (1 сообщение → 1000 получателей) | Async fan-out через Kafka; batch writes в Cassandra |
  | **Connection stickiness** (user reconnects to different WS server) | Session registry в Redis: user_id → server_id; re-subscribe при реконнекте |
  | **Scaling WebSocket servers** | Stateless WS-gateways (state в Redis); graceful drain при деплое (close со 1006 → client reconnect) |
  | **Media (фото/видео)** | Upload to S3 → ссылка в сообщении; CDN для скачивания; thumbnail generation |
  | **End-to-end encryption** | Diffie-Hellman key exchange; сервер не видит content; ключи хранятся на клиентах |
  | **Global latency** | Regional clusters; cross-region message replication (Cassandra multi-DC) |

</details>

---

### Вопрос №7: [ Спроектируй распределённый Rate Limiter ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Требования:** 1M RPS, ограничение по user_id / IP / API key; несколько политик
  (fixed window, sliding window, token bucket); < 10ms latency.

* **Алгоритмы:**
  - **Fixed Window Counter:** счётчик за период (минута). Плюс: просто. Минус: burst на границе окна
    (99 запросов в 59с + 99 в 1с следующего окна = 198 за 2с);
  - **Sliding Window Log:** timestamp каждого запроса. Плюс: точно. Минус: O(n) памяти;
  - **Sliding Window Counter** (hybrid): счётчик текущего окна + доля предыдущего окна.
    Плюс: O(1) памяти, приблизительно точно. Минус: аппроксимация;
  - **Token Bucket:** бакет с N токенами, пополняется R/сек. Каждый запрос = 1 токен.
    Плюс: позволяет burst (до N). Минус: сложнее. Используется в большинстве production-систем;
  - **Leaky Bucket:** запросы как вода, вытекают с постоянной скоростью. Плюс: сглаживает burst.

* **Распределённая архитектура:**
  ```
  Client → API Gateway → Rate Limiter Service → Redis (atomic counter)
  ```
  - Redis с Lua-скриптом: атомарная операция `incr + expire` (или token bucket logic);
  - Lua-скрипт выполняется на стороне Redis → atomicity без race conditions;
  - **Альтернатива:** local in-memory limiter per instance + periodic sync с Redis
    (allows some over-limit, но без сетевого round-trip на каждый запрос).

* **Подводные камни и решения:**

  | Проблема | Решение |
  |----------|---------|
  | **Redis SPOF** | Redis Cluster (replication + failover); fallback: allow request (fail-open) или deny (fail-close) — зависит от бизнес-требований |
  | **Clock skew** между узлами | Использовать Redis-время (серверное), не клиентское; или NTP sync |
  | **Hot keys** (один user_id → один Redis-шард) | Consistent hashing; для супер-активных ключей — local + global (два слоя) |
  | **Race condition** (между read и update счётчика) | Lua-скрипт (атомарен) или Redis MULTI/EXEC |
  | **Thundering herd при восстановлении Redis** | Circuit breaker + local fallback cache; exponential backoff |
  | **Different limits for different tiers** | Конфигурация в БД + in-memory cache конфигов (обновление по pub/sub) |
  | **Global vs per-instance limits** | Per-instance: local counter (быстро, но неточно); Global: Redis (точно, но медленнее) |

</details>

---

### Вопрос №8: [ Спроектируй систему онлайн-бронирования билетов (кино, поезда, концерты) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Требования:**
  - 10K сеансов/день, до 500 мест на сеанс; пиковая нагрузка при старте продаж (flash sale);
  - Гарантия: одно место — один пользователь (no double-booking); eventual consistency недопустима
    для бронирования (но допустима для аналитики).

* **Ключевая проблема: конкуренция за одно и то же место (concurrent booking).**

* **High-level архитектура:**
  ```
  Client → API Gateway → Booking Service → DB (seats, reservations)
                                 ↕
                              Cache (Redis: seat locks)
                                 ↕
                              Payment Service → Payment Gateway
  ```

* **Подходы к резервированию:**

* **Вариант 1 — Pessimistic locking (БД):**
  `SELECT * FROM seats WHERE id = ? FOR UPDATE` — блокировка строки на уровне БД.
  - Плюс: надёжно, просто;
  - Минус: низкая пропускная способность под нагрузкой; long transactions блокируют другие.
  - Подходит: низкая конкуренция, мало одновременных бронирований.

* **Вариант 2 — Optimistic locking (version field):**
  ```sql
  UPDATE seats SET status='booked', version=version+1
  WHERE id=? AND version=?
  ```
  - Плюс: нет долгих блокировок; хорошо при низкой конкуренции;
  - Минус: retry при конфликте; при flash sale — 99% запросов получат конфликт → retry storm.

* **Вариант 3 — Redis distributed lock (рекомендуется):**
  ```redis
  SET lock:seat:{show_id}:{seat_id} {user_id} NX EX 300
  ```
  - Бронирование места на 5 минут (TTL) в Redis (атомарно, O(1));
  - Если получилось — место зарезервировано, пользователь идёт на оплату;
  - Если не получилось — место занято, предложить другое;
  - После оплаты — статус меняется в БД на "sold"; lock снимается;
  - Если не оплатил за 5 минут — TTL истекает, место освобождается (background sweeper).

* **Вариант 4 — Event sourcing / saga:**
  - Бронирование = событие; payment = событие; confirm = событие;
  - При cancel/timeout — compensating transaction;
  - Плюс: audit log, легко откатывать; Минус: сложность, eventual consistency.

* **Подводные камни и решения:**

  | Проблема | Решение |
  |----------|---------|
  | **Flash sale** (10K пользователей на 1 сеанс одновременно) | Queue-based: запросы в Kafka → обработка по очереди; "virtual queue" (как Ticketmaster) |
  | **Seat holding abuse** (боты бронируют всё и не платят) | Captcha; pre-auth hold на карте; limit per user; bot detection |
  | **Redis failover** (брони потеряны) | Persistence (AOF); fallback на БД pessimistic lock; idempotency (повторная запись не вредит) |
  | **Payment timeout vs seat lock** | Seat lock TTL > payment timeout; webhook от платёжной системы для подтверждения; background job для очистки |
  | **Partial booking** (хотим 3 места рядом) | Multi-lock: SETNX на все места в одной Lua-транзакции (all-or-nothing) |
  | **Double-charging** (retry payment webhook) | Idempotency key в payment API; проверка статуса перед списанием |
  | **Read-heavy (просмотр схемы зала)** | Кэш схемы зала в Redis/CDN; read replica БД для чтения; WebSocket для real-time обновления занятых мест |
  | **Гео-распределение** | Regional booking (seat inventory per region); global inventory sync через CDC |

</details>

---

### Вопрос №9: [ Спроектируй систему потокового видео (типа Netflix/YouTube) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Требования:**
  - 100M пользователей, 10K видео загружается/день; 1M часов просмотров/день;
  - Adaptive bitrate (240p–4K); latency старта видео < 2с; минимальные буферизации.

* **High-level архитектура:**
  ```
  Upload:    Creator → Upload Service → Transcoding Pipeline → Storage (S3)
                                                          → CDN

  Playback:  Viewer → CDN (edge) → Video segments → Player
                     ↑ (cache miss)
                     → Origin Server → Storage (S3)
  ```

* **Загрузка и транскодинг:**
  1. Creator загружает видео (chunked upload, resumable) → S3;
  2. Триггер → сообщение в Kafka → Transcoding Worker;
  3. Транскодинг в несколько разрешений (240p, 480p, 720p, 1080p, 4K) через FFmpeg;
  4. **HLS (HTTP Live Streaming):** видео нарезается на сегменты по 2–10 секунд + playlist (.m3u8);
  5. Сегменты загружаются в S3 → инвалидация CDN.

* **Воспроизведение:**
  - Клиент запрашивает .m3u8 playlist → CDN → (cache miss) → Origin → S3;
  - Клиент запрашивает сегменты → CDN;
  - **Adaptive bitrate (ABR):** клиент измеряет bandwidth и выбирает качество сегмента;
  - CDN кэширует сегменты на edge-серверах (TTL по популярности).

* **Подводные камни и решения:**

  | Проблема | Решение |
  |----------|---------|
  | **Большой объём хранения** (4K видео = 10GB/час) | Erasure coding в S3; cold storage для непопулярных; lifecycle policies |
  | **Транскодинг — CPU-intensive** | Worker pool (auto-scaled); приоритезация (популярные каналы → быстрее); pre-set codecs (H.264, H.265, AV1) |
  | **High latency первого сегмента** | CDN edge prefetch; TTL=inf для популярных; chunked transfer encoding |
  | **Buffering при деградации сети** | ABR aggressively downgrade; prefetch following segments; QUIC для уменьшения head-of-line blocking |
  | **Viral video hot spot** (1 видео → 10M просмотров) | Multi-tier CDN; proactive cache warming; origin shield |
  | **DRM (защита контента)** | Widevine/FairPlay/PlayReady; encrypted segments; license server |
  | **Live streaming** (низкая задержка) | LL-HLS / WebRTC; segment duration 1с; edge transcoding |
  | **Global latency** | Multi-CDN strategy (Cloudflare + Akamai); geo-routing (GeoDNS) |
  | **Cost optimization** | Compute-at-edge (транскодинг на edge); cache ratio monitoring; cold storage для старых видео |

</details>

---

### Вопрос №10: [ Спроектируй распределённый кэш (типа Redis/Memcached) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Требования:**
  - 1M RPS, < 1ms latency; 100GB данных; отказоустойчивость; linear scalability.

* **Архитектура:**
  ```
  Client → Consistent Hash Ring → [Shard 1] [Shard 2] ... [Shard N]
                                   ↓ (replication)
                                 [Replica 1] [Replica 2] ...
  ```

* **Шардинг — Consistent Hashing:**
  - Каждый ключ хэшируется → попадает на узел в кольце;
  - При добавлении/удалении узла перемещается только K/N ключей (vs all keys с mod-N);
  - **Virtual nodes** (150–200 per физический узел) — для равномерного распределения нагрузки.

* **Replication:**
  - **Master-replica:** запись → master, чтение → replica; async replication;
  - **Quorum-based (Raft/Paxos):** запись → majority согласие; сильная консистентность;
    дороже, но безопаснее (Redis Raft, CockroachDB-style).

* **Eviction policies:**
  - **LRU** (Least Recently Used) — стандарт для Memcached/Redis;
  - **LFU** (Least Frequently Used) — лучше для skew-распределения (Redis 4.0+);
  - **TTL-based** — автоматическое удаление по времени;
  - **Random** — дешёвый, но плохой hit rate.

* **Подводные камни и решения:**

  | Проблема | Решение |
  |----------|---------|
  | **Cache stampede** (thundering herd: истёк TTL → 1000 запросов к БД) | **Request coalescing** (singleflight): один запрос к БД, остальные ждут; **early expiration** (обновить кэш до истечения TTL); **probabilistic early expiration** (XFetch algorithm) |
  | **Hot keys** (один ключ → 1M RPS → перегрузка одного шарда) | **Client-side L1 cache** (in-process); **key splitting** (key → key#1, key#2, ... — распределить по шардам) |
  | **Big values** (10MB value блокирует event loop) | Chunk values; separate shard для big values; chunked read |
  | **Cache penetration** (запросы за несуществующими ключами → мимо кэша → БД) | **Bloom filter** перед кэшем (быстро отсеивает несуществующие); **negative caching** (кэшировать "not found" с коротким TTL) |
  | **Cache avalanche** (массовый expiry ключей одновременно → БД ложится) | **Jittered TTL** (TTL + random offset); **staggered refresh** |
  | **Consistency** (данные в кэше устарели) | **Write-through** (запись → кэш → БД); **write-behind** (запись → кэш → async БД); **cache-aside** (lazy load + TTL) — самый частый |
  | **Replication lag** (replica отстаёт от master) | Read-your-writes: клиент пишет в master, читает из master в течение short window; или version-based consistency |
  | **Split-brain** (сеть упала, два master) | Quorum (Raft); fencing tokens; automatic failover с witness |
  | **Memory fragmentation** | jemalloc/tcmalloc; active defrag (Redis); slab allocator (Memcached) |

</details>

---

### Вопрос №11: [ Спроектируй поисковую систему (типа Google/Elasticsearch) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Требования:**
  - 10B документов, 100K RPS; latency < 100ms; релевантность (ranking); поддержка
    опечаток, синонимов, многоязычности.

* **Архитектура:**
  ```
  Crawler → Indexer → Inverted Index (sharded) → Search Nodes
                                                     ↓
  Query → Query Parser → [Shard 1] [Shard 2] ... → Merge & Rank → Results
  ```

* **Inverted Index (основная структура):**
  - Для каждого слова (term) → список документов (postings list), где оно встречается;
  - `term → [(doc_id, term_frequency, positions)]`;
  - Построение: tokenize → normalize (stemming, lowercase) → build postings;
  - Хранение: compressed postings (delta encoding, varint, Roaring bitmaps).

* **Поиск (query pipeline):**
  1. **Query parsing:** tokenize, spell correction, synonym expansion, query understanding (intent);
  2. **Retrieval:** для каждого term → postings list → intersection/union (AND/OR);
  3. **Ranking:** TF-IDF, BM25, или ML-модель (Learning to Rank) с сотнями фич
     (PageRank, freshness, click-through rate, personalization);
  4. **Merge:** top-K результатов с каждого шарда → global top-K.

* **Шардинг:**
  - По `doc_id` hash → N шардов;
  - Каждый шард — независимый inverted index;
  - **Scatter-gather:** запрос → все шарды параллельно → merge top-K.

* **Подводные камни и решения:**

  | Проблема | Решение |
  |----------|---------|
  | **Index size** (10B docs → TBs of index) | Sharding + replicas; compressed postings; columnar storage |
  | **Real-time updates** (новые документы) | **Near-real-time:** in-memory buffer → flush в segments (Lucene); **merge policy** (TieredMergePolicy) |
  | **High query fan-out** (запрос ко всем шардам) | **Adaptive replica selection** (нагруженные шарды реже опрашивать); **cache query results** |
  | **Spell correction** | Levenshtein automaton / BK-tree; n-gram index для fuzzy search; language models |
  | **Multilingual search** | Language detection → language-specific analyzer (stemmer, stopwords); Unicode normalization |
  | **Personalization** | User profile (history, location) → re-ranking фича; but privacy concerns |
  | **Cold start / long-tail queries** | Pre-computed results для топ-100K запросов; cache warming |
  | **Index corruption** | Replication; checksums; atomic segment commits; WAL (write-ahead log) |

</details>

---

### Вопрос №12: [ Спроектируй систему уведомлений (Email, Push, SMS) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Требования:**
  - 10M уведомлений/день; мультиканальность (email, push, SMS, in-app);
  - шаблоны, локализация, user preferences (opt-out), retry, priority.

* **Архитектура:**
  ```
  Event Source → Notification API → Kafka → [Worker Pool]
                                            ↓
                                Channel Router → [Email (SES)] [Push (FCM/APNs)] [SMS (Twilio)]
                                            ↓
                                Delivery Tracker → DB (status, attempts)
  ```

* **Поток обработки:**
  1. Источник (order service, social service) → POST /notifications;
  2. Notification Service валидирует, рендерит шаблон, кладёт в Kafka;
  3. Worker забирает → определяет каналы (по user preferences) → отправляет через провайдера;
  4. Webhook от провайдера → обновление статуса (delivered, bounced, failed);
  5. При failure → exponential backoff retry (3 попытки).

* **Подводные камни и решения:**

  | Проблема | Решение |
  |----------|---------|
  | **Email bounces/spam** | SES bounce handling; suppress list; DKIM/SPF/DMARC; warmup IP; feedback loop с Gmail/Outlook |
  | **Push token expiry** | Background job: clean expired tokens; APNs/FCM feedback service; re-register on app launch |
  | **SMS cost** (дорого) | Priority routing: SMS только для critical; fallback email для non-critical; regional providers |
  | **User preferences / opt-out** | Centralized preference service; проверка перед отправкой; compliance (GDPR, CAN-SPAM) |
  | **Rate limiting per user** (spam protection) | Redis token bucket per user_id + per channel; max N/hour |
  | **Idempotency** (дубликаты от retry event source) | Dedup key (event_id + notification_type); Redis SETNX |
  | **Batching** (отправка 10K email одним вызовом SES) | Kafka consumer batch; SES SendBulkEmail; reduce API calls |
  | **Priority** (2FA SMS vs marketing email) | Kafka topic partitioning by priority; separate worker pools; preemption |
  | **Template rendering performance** | Pre-compile templates (Handlebars/Go templates); cache rendered; i18n bundles |
  | **At-least-once → duplicate notifications** | Idempotency at consumer level; dedup window in Redis; user-side tolerance |

</details>

---

### Вопрос №13: [ Что такое distributed tracing и зачем он нужен в распределённых системах? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Distributed tracing** — механизм отслеживания запроса через все микросервисы.
  Без него в микросервисной архитектуре невозможно понять, где latency bottleneck или где произошла ошибка.

* **Ключевые понятия (OpenTelemetry / Jaeger / Zipkin):**
  - **Trace** — полное путешествие запроса через все сервисы;
  - **Span** — единичная операция (вызов сервиса, БД-запрос, кэш-lookup) с duration и tags;
  - **Span context** — `trace_id` + `span_id` → передаётся через headers (W3C Trace Context);
  - **Propagation** — trace context пробрасывается через HTTP headers / gRPC metadata / Kafka headers.

* **Как работает:**
  1. API Gateway создаёт root span (`trace_id`) при входящем запросе;
  2. При вызове другого сервиса — `trace_id` передаётся в header `traceparent`;
  3. Каждый сервис создаёт child span (`parent_span_id` → `span_id`);
  4. Spans отправляются (async) в collector → storage (Elasticsearch/ClickHouse) → UI (Jaeger).

* **Зачем:**
  - **Latency analysis:** увидеть, какой span занимает 80% времени (DB query? external API?);
  - **Error tracing:** найти сервис, который вернул 500;
  - **Service dependency map:** визуализация вызовов между сервисами;
  - **Performance optimization:** выявить N+1 queries, unnecessary serial calls.

* **Подводные камни:**
  - **Overhead:** sampling нужен (не трейсить 100% запросов при high RPS; head-based или tail-based sampling);
  - **Context propagation:** если сервис не пробрасывает headers → trace рвётся;
  - **Async boundaries:** Kafka/queue → trace context в message headers (не всегда поддерживается);
  - **Storage cost:** traces объёмные → retention policies, sampling rate.

</details>

---

### Вопрос №14: [ Что такое Eventual Consistency? Какие бывают модели консистентности? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Модели консистентности (от слабой к сильной):**

  1. **Eventual Consistency:** если нет новых записей, все реплики рано или поздно сходятся.
     Нет гарантии, когда. Чтение может вернуть stale data. (Cassandra, DynamoDB, DNS).
  2. **Read-Your-Writes:** клиент видит свои собственные записи (но не чужие). Достигается через
     session consistency (sticky session) или read-from-master-after-write.
  3. **Monotonic Read:** если клиент прочитал значение версии N, он не прочитает версию < N.
     Достигается через sticky replica.
  4. **Causal Consistency:** сохраняется причинно-следственная связь (если A → B, то все узлы
     видят A до B). Векторные часы.
  5. **Sequential Consistency:** все операции видны в одном и том же порядке всем клиентам
     (но порядок может не совпадать с real-time).
  6. **Linearizability (Strong Consistency):** каждая операция атомарна и видна мгновенно всем.
     = Sequential + real-time ordering. (Spanner, etcd, MongoDB с majority).

* **Когда что использовать:**
  - Strong: финансовые транзакции, inventory, координация (etcd, ZooKeeper);
  - Eventual: social feeds, likes, counters, кэш (Cassandra, Redis);
  - Read-your-writes: user profile updates (сессия направляется на master после записи).

* **Sagas для распределённых транзакций:**
  - Вместо 2PC (XA), который блокирует и медленный → **Saga pattern**: последовательность
    локальных транзакций, каждая с **compensating action** (откат);
  - Пример: order → reserve_inventory → charge_payment → confirm_order;
    если charge fails → compensate: release_inventory, cancel_order;
  - Координация: **orchestration** (central coordinator) или **choreography** (event-driven, без coordinator).

</details>

---

### Вопрос №15: [ Что такое CQRS и Event Sourcing? Когда их применять? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **CQRS (Command Query Responsibility Segregation):**
  - Разделение моделей на **Command** (запись, изменение состояния) и **Query** (чтение);
  - Write side: нормализованная БД, оптимизированная под запись (OLTP);
  - Read side: денормализованная, оптимизированная под чтение (Elasticsearch, Redis, materialized views);
  - Синхронизация: через события (domain events) или CDC (Change Data Capture — Debezium).

  Схема CQRS:

  ```
  +------------------+         +----------------------+
  | Command Handler  |         | Query Handler        |
  |                  |         |                      |
  |   +----------+   |         |   +--------------+   |
  |   | Command  |   |         |   |   Query      |   |
  |   | Bus      |---|-------->|---|   Bus        |   |
  |   +----------+   |         |   +--------------+   |
  +------------------+         +----------------------+
           |                            |
           v                            v
  +------------------+         +----------------------+
  | Aggregate Root   |         | Read Model          |
  |                  |         |                      |
  |   +----------+   |         |   +--------------+   |
  |   |  Event   |   |         |   |   Query      |   |
  |   |  Store   |---|-------->|---|   Database   |   |
  |   +----------+   |         |   +--------------+   |
  +------------------+         +----------------------+
  ```

  Элементы схемы:
  - **Command Handler** — обрабатывает команды, изменяющие состояние системы; отправляет команды агрегатам (Aggregate Roots);
  - **Query Handler** — обрабатывает запросы на чтение; извлекает данные из Read Model;
  - **Command Bus** — направляет команды от Command Handler к агрегатам;
  - **Query Bus** — направляет запросы от Query Handler к Read Model;
  - **Aggregate Root** — объект, ответственный за обработку команд и содержащий логику бизнес-правил;
  - **Event Store** — хранит события, представляющие изменения состояния системы; используются для обновления Read Model;
  - **Read Model** — данные, оптимизированные для конкретных запросов чтения;
  - **Query Database** — хранилище для запросов, оптимизированное под чтение.

* **Event Sourcing:**
  - Вместо хранения текущего состояния — хранение **серии событий** (event log);
  - Текущее состояние = replay всех событий;
  - Event Store (Kafka, EventStoreDB) — append-only, immutable;
  - Пример: вместо `balance = 500` → `[Deposited(1000), Withdrew(300), Withdrew(200)]`.

* **Когда использовать:**
  - CQRS: когда read/write паттерны сильно разные (например, heavy read с фильтрацией vs simple write);
  - Event Sourcing: когда нужен audit log, temporal queries («каким был баланс на 1 января?»),
    undo/redo, event-driven архитектура;
  - **Вместе:** CQRS + Event Sourcing — write side = event store, read side = projections
    (materialized views, построенные из событий).

* **Подводные камни:**

  | Проблема | Решение |
  |----------|---------|
  | **Complexity** (значительно сложнее CRUD) | Использовать только когда benefit > cost; не для простых доменов |
  | **Eventual consistency** (read model отстаёт) | Acceptable для большинства; read-your-writes для критичных; version markers |
  | **Event schema evolution** (старые события в новом формате) | Versioning (event version field); upcasters (трансформация старых событий); forward/backward compatibility |
  | **Replay performance** (миллионы событий → долго) | Snapshots (периодическое сохранение состояния); incremental projections |
  | **Debugging difficulty** | Event log = source of truth → tools для replay и inspection; correlation IDs |

</details>

---

### Вопрос №16: [ Как обеспечить идемпотентность API в распределённой системе? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Идемпотентность** — повторный вызов с теми же параметрами даёт тот же результат, что и первый.
  Критична для: retry logic, at-least-once delivery, payment APIs.

* **Подходы:**

* **1. Idempotency Key (рекомендуется):**
  - Клиент генерирует уникальный `Idempotency-Key` (UUID) при первом запросе;
  - Сервер хранит ключ → результат в Redis/БД с TTL;
  - При повторе с тем же ключом — вернуть сохранённый результат (без повторного выполнения);
  - Пример: Stripe API требует `Idempotency-Key` для всех POST-запросов.

* **2. Natural idempotency (через бизнес-логику):**
  - `PUT /users/123 {name: "Alice"}` — идемпотентен по определению HTTP PUT;
  - `DELETE /users/123` — идемпотентен (удаление уже удалённого = OK);
  - `POST /transfer {from: A, to: B, amount: 100, reference: "TX123"}` — reference = unique constraint.

* **3. Optimistic locking (version/etag):**
  - `If-Match: "v3"` → обновление только если версия не изменилась;
  - 409 Conflict при рассинхронизации → клиент retry с обновлёнными данными.

* **Подводные камни:**

  | Проблема | Решение |
  |----------|---------|
  | **Race condition** (два запроса с одним ключом одновременно) | Distributed lock (Redis SETNX) или unique constraint в БД |
  | **Key storage TTL** | TTL > max retry window (обычно 24–48 часов); или permanent storage для критичных |
  | **Partial execution + crash** | Состояние ключа: `in_progress` → `completed`/`failed`; при retry — проверить статус |
  | **Network timeout** (запрос дошёл, ответ нет) | Клиент retry → сервер возвращает сохранённый результат |
  | **Different request body, same key** | Сохранять hash тела запроса → 422 Unprocessable Entity при mismatch |

</details>

---

### Вопрос №17: [ Что такое backpressure и как с ним работать? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Backpressure** — ситуация, когда производитель (producer) генерирует данные быстрее,
  чем потребитель (consumer) может их обработать. Без управления → растущая очередь → OOM или
  деградация всей системы.

* **Стратегии:**

  1. **Drop (lossy):** отбрасывать новые данные при перегрузке (например, rate limit).
     Подходит: телеметрия, логи, real-time видео (лучше потерять кадр, чем зависнуть).
  2. **Buffer:** буферизовать в очереди (Kafka, RabbitMQ). Подходит: burst-нагрузки.
     Риск: безграничный буфер → OOM. Решение: bounded buffer + backpressure signal.
  3. **Throttle producer:** сигнал producer-у замедлиться (reactive streams: `request(n)`).
     Подходит: streaming pipelines (gRPC streaming, ReactiveX, Project Reactor).
  4. **Scale out consumer:** auto-scaling consumer-подов (KEDA, HPA на queue depth).
     Подходит: stateless consumers, облачная инфраструктура.
  5. **Shed load:** намеренно отклонять часть запросов (circuit breaker, 429 Too Many Requests).
     Подходит: защита критических сервисов от каскадного отказа.

* **В Go:**
  - Bounded channel: `ch := make(chan Job, maxQueue)` → `ch <- job` блокируется при заполнении
    → естественный backpressure на producer;
  - `context.WithTimeout` для таймаута → не копить запросы бесконечно;
  - Worker pool pattern: фиксированное число воркеров → control concurrency.

* **Circuit Breaker** (дополняет backpressure):
  - **Closed** → нормальная работа; при N failures → **Open** (быстрый отказ, не нагружать);
  - После timeout → **Half-Open** (пропустить 1 запрос для проверки);
  - Если успех → **Closed**; если fail → **Open** снова;
  - Реализации: gobreaker, resilience4j, Hystrix (deprecated).

</details>

---

### Вопрос №18: [ Что такое CDC (Change Data Capture) и где применяется? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **CDC** — паттерн захвата изменений данных из БД в реальном времени и передачи их
  downstream-потребителям (через Kafka и т.п.).

* **Как работает (на примере Debezium + PostgreSQL):**
  1. Debezium читает WAL (Write-Ahead Log) PostgreSQL → генерирует change events
     (INSERT/UPDATE/DELETE) с before/after значениями;
  2. События публикуются в Kafka topic (one topic per table);
  3. Downstream сервисы потребляют события: обновление поискового индекса (Elasticsearch),
     кэша (Redis), аналитического хранилища (ClickHouse), CQRS read model.

* **Преимущества над polling:**
  - Near real-time (ms latency vs seconds/minutes для polling);
  - Не нагружает БД (чтение WAL vs `SELECT * WHERE updated_at > ?`);
  - Гарантия порядка ( WAL → ordered events).

* **Применения:**
  - Синхронизация БД → Elasticsearch/Redis (без dual-write);
  - Event-driven микросервисы (без изменения исходного сервиса);
  - Аналитика: OLTP → OLAP репликация (Postgres → ClickHouse);
  - Audit log (кто, что, когда изменил);
  - Cache invalidation.

* **Подводные камни:**

  | Проблема | Решение |
  |----------|---------|
  | **Schema changes** (ALTER TABLE → событие ломает consumer) | Schema registry (Avro/Protobuf); backward-compatible schema evolution |
  | **Exactly-once** (CDC = at-least-once → дубликаты) | Idempotent consumers; dedup по primary key + event timestamp |
  | **Large transactions** (1 transaction → 10K events) | Consumer должен быть готов к batch; transaction metadata в событиях |
  | **Backpressure** (consumer отстаёт → WAL растёт) | Monitoring lag; auto-scaling consumers; alerting на consumer lag |
  | **Initial snapshot** (при первом запуске — весь dataset) | Staggered snapshot; offline initial load; replicated from backup |
  | **DDL changes** | Debezium может не поддерживать все DDL → ручная миграция consumer schema |

</details>

---

### Вопрос №19: [ Спроектируй систему для обработки данных в реальном времени (Streaming Pipeline) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Требования:**
  - 1M events/sec (clickstream, IoT, transactions); processing latency < 1с;
  - Агрегации, windowing, joins; exactly-once processing; fault tolerance.

* **Архитектура:**
  ```
  Producers → Kafka (partitioned by key) → Stream Processors → Sink (DB, DW, another Kafka)
                                                ↓
                                           State Store (RocksDB)
  ```

* **Технологии:**
  - **Kafka:** distributed log, partitioning, retention, exactly-once (idempotent producer + transactions);
  - **Kafka Streams / Apache Flink / Spark Streaming:** stream processing engines;
  - **RocksDB (embedded):** local state store для windowed aggregations;
  - **Schema Registry:** Avro/Protobuf для контрактной совместимости.

* **Ключевые концепции:**
  - **Partitioning:** key-based partitioning → события с одним key идут в одну партицию →
    локальный state для этого key;
  - **Windowing:** tumbling (fixed), hopping (overlapping), sliding (event-time based);
  - **Watermarks:** механизм обработки late events (события пришли позже watermark → drop или side-output);
  - **Checkpointing:** periodic snapshot state → restore после сбоя (Flink: Chandy-Lamport algorithm).

* **Exactly-once semantics (EOS):**
  - Kafka idempotent producer (dedup на broker);
  - Kafka transactions: consume-process-produce атомарно (commit offset + produce в одной транзакции);
  - Flink: checkpoint + two-phase commit sink.

* **Подводные камни и решения:**

  | Проблема | Решение |
  |----------|---------|
  | **Out-of-order events** | Event-time processing + watermarks; allowed lateness; side output для late events |
  | **State growth** (session windows не закрываются) | Session timeout; RocksDB compaction; incremental checkpoints |
  | **Repartitioning** (join двух streams по разным key) | Shuffle (re-partition): consume → re-key → produce → consume. Дорого. |
  | **Hot partition** (один key → 90% трафика) | Salting (split key: `user_id_0..9`); local aggregation + global merge |
  | **Backpressure** (processor не успевает) | Auto-scaling (Kafka consumer groups); bounded buffers; load shedding |
  | **Schema evolution** | Schema Registry + backward-compatible Avro/Protobuf; consumer-driven contracts |
  | **Exactly-once overhead** | EOS дороже (transactions, checkpoints) → использовать только когда нужно |
  | **Replay** (переобработка данных) | Kafka retention; rewind consumer offset; separate replay cluster |

</details>

---

### Вопрос №20: [ Как проектировать отказоустойчивость (resilience) в распределённых системах? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Принципы отказоустойчивости:**

* **1. Eliminate SPOF (Single Point of Failure):**
  - Каждый компонент должен иметь N+1 реплику (минимум);
  - Load balancer, БД, кэш, message broker — всё в HA-конфигурации;
  - Multi-AZ deployment (минимум) / multi-region (для mission-critical).

* **2. Graceful degradation:**
  - При отказе зависимого сервиса — система не падает, а деградирует функциональность;
  - Пример: кэш недоступен → чтение из БД (медленнее, но работает); recommendation service down →
    показать "popular" вместо "personalized".

* **3. Circuit Breaker:**
  - Предотвращает cascading failure: при отказе downstream — быстрый fail (не ждать timeout);
  - Три состояния: Closed (норма) → Open (fail fast) → Half-Open (probe).

* **4. Bulkhead (изоляция ресурсов):**
  - Разделение пулов ресурсов по типам запросов → один медленный запрос не исчерпывает все ресурсы;
  - Пример: separate connection pools для different downstream services; separate thread pools
    для read vs write.

* **5. Timeout everywhere:**
  - Каждый сетевой вызов должен иметь timeout (HTTP, DB, cache);
  - Без timeout → thread/goroutine leak → cascading failure;
  - Timeout cascading: client timeout < server timeout (чтобы клиент не сдался раньше сервера).

* **6. Retry with backoff + jitter:**
  - При transient failure → retry с exponential backoff (1с, 2с, 4с, 8с);
  - **Jitter** (random offset) — чтобы все клиенты не ретраили одновременно (thundering herd);
  - Max retries (3–5) → затем circuit breaker или dead letter queue.

* **7. Idempotency:**
  - Все retry-safe операции должны быть идемпотентны (см. вопрос №16).

* **8. Health checks + graceful shutdown:**
  - Liveness probe (жив ли процесс?) + Readiness probe (готов ли обслуживать?);
  - Graceful shutdown: stop accepting new → finish in-flight → close connections;
  - LB убирает инстанс из пула до начала shutdown.

* **Метрики для мониторинга:**
  - Error rate (5xx, exceptions);
  - Latency percentiles (p50, p95, p99);
  - Saturation (CPU, memory, connection pool, queue depth);
  - Throughput (RPS);
  - Recovery time (MTTR — Mean Time To Recovery).

</details>

---

### Вопрос №21: [ Что такое шардирование БД? Какие стратегии существуют? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Шардирование** — разделение данных БД на несколько серверов (шардов) для горизонтального
  масштабирования. Каждый шард хранит подмножество данных.

* **Стратегии:**

* **1. Range-based (по диапазону):**
  - Данные делятся по диапазону ключа (A–M → shard 1, N–Z → shard 2);
  - Плюс: range queries эффективны;
  - Минус: **hotspots** (если данные сконцентрированы в одном диапазоне — все идут на один шард);
  - Пример: MongoDB default sharding, HBase regions.

* **2. Hash-based:**
  - `shard = hash(key) % N` — равномерное распределение;
  - Плюс: нет hotspots;
  - Минус: **rebalancing при добавлении шарда** — все ключи перемешиваются (mod-N не масштабируется);
  - Решение: **consistent hashing** (перемещается только K/N ключей при добавлении узла).

* **3. Directory-based:**
  - Отдельная lookup-таблица (router): `key → shard_id`;
  - Плюс: гибкость (можно перемещать данные, менять маппинг);
  - Минус: router — SPOF (решается репликацией); extra lookup на каждый запрос (кэшировать).

* **4. Geo-based:**
  - Данные по геолокации пользователя (EU users → EU shard, US users → US shard);
  - Плюс: data sovereignty (GDPR), низкая latency;
  - Минус: cross-region queries сложны; uneven distribution.

* **Подводные камни:**

  | Проблема | Решение |
  |----------|---------|
  | **Cross-shard queries** (JOIN между шардами) | Денормализация; application-level joins; или не шардировать, если JOIN критичен |
  | **Distributed transactions** | избегать (2PC медленный); saga pattern; или спроектировать так, чтобы транзакция была в одном шарде |
  | **Rebalancing** | Consistent hashing; online resharding (Vitess, CockroachDB); двойная запись во время миграции |
  | **Hot shard** | Shard splitting; salting; application-level routing для известных hot keys |
  | **Counting / aggregations** | Pre-aggregated counters (per-shard → sum); approximate (HyperLogLog) |
  | **Global unique IDs** | Snowflake IDs (timestamp + shard_id + sequence); UUID; DB sequences per shard |
  | **Schema changes** | Rolling migration across shards; backward-compatible DDL; deploy schema before code |

</details>

---

### Вопрос №22: [ Как обеспечить безопасность API в распределённой системе? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Аутентификация (кто ты?):**
  - **JWT (JSON Web Token):** stateless, self-contained (header.payload.signature);
    Access token (короткий, 15 мин) + Refresh token (длинный, 7 дней);
    Минус: нельзя отозвать (решение: blacklist с коротким TTL; или short access token + refresh rotation);
  - **OAuth 2.0 / OIDC:** делегированная авторизация (вход через Google/GitHub);
    Authorization Code flow + PKCE для SPA/mobile;
  - **mTLS (mutual TLS):** клиент и сервер проверяют сертификаты друг друга; для service-to-service
    (zero-trust, service mesh).

* **Авторизация (что тебе можно?):**
  - **RBAC (Role-Based):** роли → permissions; просто, но негибко (role explosion);
  - **ABAC (Attribute-Based):** правила на атрибутах (user.department == resource.department);
    гибко, но сложно;
  - **ReBAC (Relationship-Based):** Google Zanzibar — проверки через граф отношений
    (user:alice → editor → document:123). Open-source: OpenFGA, SpiceDB.

* **Защита API:**
  - **Rate limiting** (см. вопрос №7);
  - **API Gateway:** единая точка для auth, rate limit, logging, routing;
  - **Input validation:** schema validation (OpenAPI), sanitization (SQL/XSS injection);
  - **CORS:** restrict origins;
  - **CSRF protection:** для cookie-based auth — CSRF tokens / SameSite cookies;
  - **Secrets management:** Vault, AWS Secrets Manager; не в коде/env vars;
  - **Encryption in transit:** TLS 1.3; in rest: DB encryption, S3 SSE.

* **Service-to-service security (zero-trust):**
  - **mTLS everywhere** (через service mesh: Istio, Linkerd);
  - **Service identities** (SPIFFE/SPIRE);
  - **Token propagation** (JWT forwarded с original user context);
  - **Least privilege:** каждый сервис имеет доступ только к нужным ему эндпоинтам.

</details>

---

### Вопрос №23: [ Что такое паттерн BFF (Backend for Frontend)? Когда его применять? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **BFF (Backend for Frontend)** — паттерн, при котором создаётся отдельный бэкенд-сервис для каждого
  типа клиента (web, mobile, desktop, IoT). Каждый BFF оптимизирован под конкретные потребности клиента.

* **Проблема, которую решает:**
  - Generic API пытается удовлетворить все клиенты → компромиссы в дизайне;
  - Mobile нужен компактный ответ (экономия трафика), Web — полный;
  - Разная логика агрегации данных для разных UI;
  - Разные требования к безопасности, кэшированию, rate limiting.

* **Архитектура:**
  ```
  Mobile App → Mobile BFF → [Core Services]
  Web App    → Web BFF    → [Core Services]
  Admin UI   → Admin BFF  → [Core Services]
  ```

* **Что делает BFF:**
  - **Aggregation:** собирает данные из нескольких сервисов в один ответ (eliminates chatty calls);
  - **Transformation:** адаптирует формат данных под конкретный UI;
  - **Protocol adaptation:** gRPC internally → GraphQL/REST externally;
  - **Optimization:** кэширование, compression, field filtering для mobile;
  - **Security:** different auth policies per client type.

* **Когда использовать:**
  - Несколько клиентов с разными потребностями (mobile vs web vs desktop);
  - Нужна агрегация данных из нескольких сервисов;
  - Разные требования к безопасности/авторизации;
  - Оптимизация под конкретный UI (mobile = minimal payload).

* **Когда НЕ использовать:**
  - Один клиент (over-engineering);
  - Простые CRUD-приложения;
  - Команда слишком маленькая (дополнительная инфраструктура).

* **Подводные камни:**

  | Проблема | Решение |
  |----------|---------|
  | **Code duplication** (логика дублируется в разных BFF) | Shared libraries для business logic; BFF = thin aggregation layer |
  | **Maintenance overhead** (N клиентов = N BFF) | Автоматическая генерация API (GraphQL codegen); shared proto/IDL |
  | **Latency** (extra hop через BFF) | BFF должен быть lightweight; colocate с API Gateway |
  | **Consistency** (разные BFF возвращают разные данные) | Clear ownership: каждый BFF отвечает за свой use case; contracts testing |
  | **Deployment complexity** (N сервисов вместо 1) | GitOps; shared infrastructure; auto-scaling per BFF |

* **Пример (Go):**
  ```go
  // Mobile BFF: агрегирует данные из UserService и OrderService
  func (bff *MobileBFF) GetUserProfile(ctx context.Context, userID string) (*MobileProfile, error) {
      // Parallel calls to core services
      g, ctx := errgroup.WithContext(ctx)

      var user *User
      var orders []Order

      g.Go(func() error {
          var err error
          user, err = bff.userClient.GetUser(ctx, userID)
          return err
      })

      g.Go(func() error {
          var err error
          orders, err = bff.orderClient.GetRecentOrders(ctx, userID, 5) // limit for mobile
          return err
      })

      if err := g.Wait(); err != nil {
          return nil, err
      }

      // Transform to mobile-specific response
      return &MobileProfile{
          Name:        user.Name,
          Avatar:      user.AvatarURL,
          RecentOrders: transformOrders(orders), // compact format
      }, nil
  }
  ```

</details>

---

### Вопрос №24: [ Что такое Sidecar паттерн? Какие проблемы он решает? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Sidecar** — паттерн, при котором дополнительный компонент (sidecar container) развёртывается
  вместе с основным приложением в одном Pod (Kubernetes) или на одном хосте. Sidecar расширяет
  функциональность приложения без изменения его кода.

* **Примеры использования:**

* **1. Service Mesh Proxy (Envoy/Linkerd):**
  - Sidecar перехватывает весь сетевой трафик приложения;
  - Реализует: mTLS, retries, circuit breaking, tracing, metrics;
  - Приложение не знает о mesh (transparent proxy);
  - Пример: Istio injects Envoy sidecar автоматически.

* **2. Logging/Fluentd sidecar:**
  - Приложение пишет логи в file/stdout;
  - Sidecar читает логи → отправляет в centralized system (Elasticsearch, CloudWatch);
  - Разделение ответственности: app = business logic, sidecar = log shipping.

* **3. Config reloader:**
  - Sidecar watches ConfigMap/Secret changes;
  - При изменении → reloads app config (SIGHUP, API call);
  - App не знает о Kubernetes ConfigMap.

* **4. SSL/TLS termination:**
  - Sidecar (nginx/envoy) terminates TLS;
  - App receives unencrypted traffic (localhost only);
  - Centralized certificate management.

* **5. Monitoring/Exporter:**
  - Sidecar collects metrics from app (JMX, statsd);
  - Exposes in Prometheus format;
  - App doesn't need Prometheus client library.

* **Архитектура (Kubernetes Pod):**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: app-with-sidecar
  spec:
    containers:
    - name: main-app
      image: myapp:1.0
      ports:
      - containerPort: 8080

    - name: envoy-sidecar
      image: envoyproxy/envoy:v1.28
      ports:
      - containerPort: 9901  # admin
      - containerPort: 15001 # proxy
      volumeMounts:
      - name: envoy-config
        mountPath: /etc/envoy

    volumes:
    - name: envoy-config
      configMap:
        name: envoy-config
  ```

* **Преимущества:**
  - **Separation of concerns:** infrastructure logic вынесена из приложения;
  - **Reusability:** один sidecar используется многими приложениями;
  - **Independent updates:** sidecar можно обновлять без изменения приложения;
  - **Language-agnostic:** sidecar работает с любым языком (polyglot);
  - **No code changes:** приложение не модифицируется.

* **Подводные камни:**

  | Проблема | Решение |
  |----------|---------|
  | **Resource overhead** (каждый Pod = N контейнеров) | Resource limits (CPU/memory); shared process namespace (optional) |
  | **Networking complexity** (localhost communication) | Use `127.0.0.1` between containers; document ports clearly |
  | **Lifecycle management** (sidecar crash vs app crash) | `restartPolicy: Always`; health checks for both containers |
  | **Log aggregation** (logs from multiple containers) | Separate log paths; sidecar collects all (fluentd tail plugin) |
  | **Debugging difficulty** (multi-container Pod) | `kubectl logs -c <container>`; ephemeral containers for debugging |
  | **Versioning** (sidecar version != app version) | Automate sidecar injection (Istio, Linkerd); GitOps |
  | **Cold start latency** (sidecar init before app) | Init containers for setup; readiness probes |

* **Ambassador pattern (вариация):**
  - Sidecar помогает основному контейнеру взаимодействовать с внешним миром;
  - Пример: proxy для доступа к external service (authentication, retry, routing);
  - Отличие: sidecar = расширяет, ambassador = помогает общаться с external world.

</details>

---

### Вопрос №25: [ Что такое Service Discovery? Какие бывают подходы? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Service Discovery** — механизм автоматического обнаружения сервисов в распределённой системе.
  В динамической среде (Kubernetes, cloud) IP-адреса меняются (scaling, failures, deployments),
  поэтому hardcoded endpoints не работают.

* **Компоненты Service Discovery:**
  1. **Service Registry** — база данных сервисов (IP, port, metadata, health status);
  2. **Registration** — сервис регистрируется при старте (self-registration или sidecar);
  3. **Discovery** — сервис находит другие сервисы через registry;
  4. **Health checking** — periodic checks → удаление unhealthy instances.

* **Подходы:**

* **1. Client-side discovery:**
  - Клиент сам запрашивает service registry → получает список instances → выбирает (load balancing);
  - Пример: Netflix Eureka + Ribbon;
  - Плюс: клиент контролирует load balancing logic;
  - Минус: клиент должен знать о discovery; extra network hop.
  ```
  Client → Service Registry → [List of instances]
  Client → Load balancer → Instance 1/2/3
  ```

* **2. Server-side discovery:**
  - Клиент делает запрос к load balancer → LB сам запрашивает registry → routing;
  - Пример: AWS ELB + Route 53, Kubernetes Service;
  - Плюс: клиент не знает о discovery (transparent);
  - Минус: LB = SPOF (решается clustering); extra hop.
  ```
  Client → Load Balancer → Service Registry → Instance 1/2/3
  ```

* **3. DNS-based discovery:**
  - Сервисы регистрируются через DNS (A/SRV records);
  - Клиент использует DNS resolution;
  - Пример: Kubernetes DNS (`service.namespace.svc.cluster.local`), Consul DNS interface;
  - Плюс: language-agnostic (все знают DNS);
  - Минус: DNS caching → slow failover; TTL trade-offs.

* **Service Registry инструменты:**

  | Инструмент | Тип | Особенности |
  |------------|-----|-------------|
  | **Consul** | External | Health checking, KV store, DNS interface, multi-DC |
  | **etcd** | External | Strong consistency (Raft), used by Kubernetes |
  | **ZooKeeper** | External | Strong consistency, widely used (Hadoop, Kafka) |
  | **Eureka** | External | AWS-style, eventual consistency (Netflix OSS) |
  | **Kubernetes DNS** | Built-in | Automatic (Pod/Service → DNS); no manual registration |
  | **Nacos** | External | Dynamic DNS, config management (Alibaba) |

* **Kubernetes Service Discovery:**
  ```yaml
  # Kubernetes автоматически создаёт DNS-запись
  apiVersion: v1
  kind: Service
  metadata:
    name: user-service
  spec:
    selector:
      app: user-service
    ports:
    - port: 80
      targetPort: 8080

  # Другие Pod'ы могут обращаться:
  # http://user-service.default.svc.cluster.local
  # или просто: http://user-service (в том же namespace)
  ```

* **Health checking:**
  - **Passive:** LB отслеживает errors/timeouts → removes instance;
  - **Active:** registry periodically pings `/health` → marks unhealthy;
  - **TTL-based:** service renewes lease (etcd/Consul); no renew → expired.

* **Подводные камни:**

  | Проблема | Решение |
  |----------|---------|
  | **Stale registrations** (сервис упал, но запись осталась) | Active health checks; TTL-based leases; automatic deregistration |
  | **DNS caching** (долгое переключение при failover) | Low TTL; client-side caching with refresh; service mesh |
  | **Registry SPOF** | Cluster registry (etcd: Raft quorum; Consul: gossip + Raft); multi-region replication |
  | **Network partition** (registry недоступен) | Client-side cache (last known good instances); circuit breaker |
  | **Registration storm** (1000 instances регистрируются одновременно при деплое) | Batch registration; rate limiting; gradual rollout |
  | **Consistency vs availability** | Strong consistency (etcd/Raft) для критичных сервисов; eventual (Eureka) для scalability |
  | **Multi-region discovery** | Consul multi-DC; Kubernetes multi-cluster (Federation); global load balancer |
  | **Service mesh integration** | Istio/Linkerd automatic service discovery; no manual registration |

* **Go пример (Consul client):**
  ```go
  import "github.com/hashicorp/consul/api"

  // Регистрация сервиса
  func RegisterService(consulClient *api.Client, serviceID, serviceName, address string, port int) error {
      registration := &api.AgentServiceRegistration{
          ID:      serviceID,
          Name:    serviceName,
          Address: address,
          Port:    port,
          Check: &api.AgentServiceCheck{
              HTTP:                           fmt.Sprintf("http://%s:%d/health", address, port),
              Interval:                       "10s",
              Timeout:                        "2s",
              DeregisterCriticalServiceAfter: "1m",
          },
      }
      return consulClient.Agent().ServiceRegister(registration)
  }

  // Discovery: найти все instances сервиса
  func DiscoverService(consulClient *api.Client, serviceName string) ([]*api.ServiceEntry, error) {
      entries, _, err := consulClient.Health().Service(serviceName, "", true, nil)
      return entries, err
  }
  ```

</details>

---

### Вопрос №26: [ Что такое SOLID? ]

<details>
<summary>Ответ</summary>

* SOLID — это аббревиатура, которая представляет пять основных принципов объектно-ориентированного программирования и проектирования:
- S - Single Responsibility Principle (Принцип единственной ответственности): Класс должен иметь только одну причину для изменения. В других словах, у него должна быть только одна задача или ответственность.
- O - Open/Closed Principle (Принцип открытости/закрытости): Программные сущности (классы, модули, функции и т.д.) должны быть открыты для расширения, но закрыты для модификации.
- L - Liskov Substitution Principle (Принцип подстановки Барбары Лисков): Объекты в программе должны быть заменяемыми на экземпляры их подтипов, без изменения желательности программы.
- I - Interface Segregation Principle (Принцип разделения интерфейса): Клиенты не должны быть вынуждены зависеть от интерфейсов, которые они не используют.
- D - Dependency Inversion Principle (Принцип инверсии зависимостей): Зависимость на Абстракциях. Нет зависимости на что-то конкретное.

</details>

---

### Вопрос №27: [ Что такое KISS? ]

<details>
<summary>Ответ</summary>

* KISS (Keep It Simple, Stupid) — это принцип проектирования, который утверждает, что большинство систем работают лучше
  всего, если они остаются простыми, а не усложненными. Сложность должна быть устранена там, где это возможно.

</details>

---

### Вопрос №28: [ Что такое DRY? ]

<details>
<summary>Ответ</summary>

* DRY (Don't Repeat Yourself) — это принцип разработки программного обеспечения, нацеленный на уменьшение повторения информации.
  Любая логика или определение должны существовать в единственном месте в коде, что уменьшает вероятность ошибок и упрощает поддержку.
</details>

---

### Вопрос №29: [ Что такое YAGNI? ]

<details>
<summary>Ответ</summary>

* YAGNI (You Aren't Gonna Need It) — это принцип, который гласит, что программисты не должны добавлять функциональность до тех пор,
  пока это не станет абсолютно необходимым. Это напоминает о том, что добавление ненужных функций может привести к дополнительным затратам
  времени и ресурсов, не принося при этом реальной пользы.

</details>

---

### Вопрос №30: [ Что такое инъекция зависимости (Dependency Injection)? ]

<details>
<summary>Ответ</summary>

* Инъекция зависимостей (Dependency Injection, DI) — это метод проектирования программного обеспечения, который упрощает
  управление зависимостями между объектами. Этот принцип может быть применен в любом языке программирования, включая Go (Golang).
  В Go для инъекции зависимостей часто используются интерфейсы и структуры.

* Это упрощает тестирование (можно использовать мок-объекты) и повышает гибкость кода, делая его менее зависимым от конкретных реализаций.
  В Go встроенных фреймворков для DI нет, но язык предоставляет все необходимые средства для реализации этого принципа
  самостоятельно или с помощью сторонних библиотек.

</details>

---

### Вопрос №31: [ Domain Driven Design ]

<details>
<summary>Ответ</summary>

* Domain-Driven Design (development) — предметно-ориентированное проектирование.
* Domain-Driven Design — подход к проектированию ПО, в основе которого положено тесное сотрудничество клиента и разработчиков.
  Заказчик посвящает команду в бизнес-логику своей компании, объясняет, как устроена ее работа, участвует в проектировании сервиса.
  Основной принцип DDD — разделение приложения на домены.
* Домен — предметная область, которая описывает совокупность проблем и целей бизнеса.
  Можно сказать, что это — отрасль деятельности компании заказчика, которую необходимо автоматизировать с помощью IT-решения.
  Например, если компания занимается грузовыми перевозками, в домен будут входить все связанные с этим проблемы: выбрать машину,
  построить маршрут и другие.
* В проектируемом веб-сервисе выделяют также основной домен — core domain, ядро продукта, его основную функцию.
* Так, если продолжить пример с перевозкой грузов: основная задача приложения — обеспечить доставку посылки из пункта А в пункт Б.
   - Ubiquitous Language— язык описания или универсальный язык.
      Он нужен, чтобы описание домена и моделей было однозначным, не возникало противоречий.
  - Bounded Context — ограниченный контекст. Ограниченная предметная область, которая отвечает за строго определенный функционал,
      со своим языком описания.
  - Репозитории (Repositories): Это механизмы для доступа к сущностям и агрегатам из базы данных.
      Репозитории предоставляют абстракцию для сохранения и извлечения данных.
  - Сервисы (Services): Это объекты, предназначенные для выполнения операций, которые не являются естественной обязанностью какой-либо
      конкретной сущности или значения.
  - Фабрики (Factories): Они отвечают за создание сложных объектов или агрегатов, обеспечивая соблюдение инвариантов.
* Основной принцип работы по DDD — разделение предметной области на ограниченные контексты со своими языками описания. Посетитель, модератор,
  администратор — это разные типы пользователей, каждый из которых относится к своей области.
* Чтобы сервис корректно работал и выполнял все свои функции, между модулями системы нужно настроить связи.
  В DDD-подходе это называется context mapping.
      - Партнерство (Partnership), общее ядро (Shared Kernel): используются для тесно связанных между собой контекстах.
      - Потребитель-поставщик (Customer-Supplier), конформист (Conformist): нужны, когда для разных контекстов приходится использовать
          один универсальный язык.
      - Антикоррупционный слой (Anticorruption Layer): применяется, когда нужна прослойка между универсальным языком источника данных.
      - Open Host Service, опубликованный язык (Published Language): используются на стороне источника данных для стандартизации протокола
          обмена и приема.

</details>

---

### Вопрос №32: [ Гексагональная архитектура ]

<details>
<summary>Ответ</summary>

* Гексагональная архитектура, также известная как порт-адаптерная архитектура или Clean Architecture, предложенная Алистером Кокбёрном,
  представляет собой подход к проектированию программного обеспечения, в котором ядро бизнес-логики выделено в центральную часть (ядро),
  а внешние компоненты, такие как интерфейсы пользователя, базы данных, веб-сервисы, представляются как адаптеры вокруг ядра.
  Это позволяет легко заменять внешние компоненты, не затрагивая ядро системы.

  ```
                      +---------------------+
                      | External Services   |
                      | (Web, Database,     |
                      |  Third-party, etc.) |
                      +----------^----------+
                                 |
                                 |
                                 v
                      +----------------------+
                      |      Adapters        |
                      | (Converters, Drivers)|
                      +----------^-----------+
                                 |
                                 |
            +---------------------|---------------------+
            |                     |                     |
            v                     v                     v
    +------------------+  +-------------------+  +------------------+
    |    Adapter       |  | Application Core  |  |    Adapter       |
    | (Web Interface,  |  | (Business Logic)  |  | (Database Adapter|
    |  CLI Interface,  |  |                   |  |   or Repository) |
    |  etc.)           |  +-------------------+  +------------------+
    +------------------+         ^         |
                                 |         |
                                 |         |
                                 v         |
                    +-------------------+  |
                    |   Domain Logic    |  |
                    | (Entities, Value  |  |
                    | Objects, Services)|  |
                    +-------------------+  |
                                 ^        |
                                 |        |
                                 +--------+
  ```

  Элементы схемы:
  - **External Services** — внешние службы (веб-сервисы, БД, интерфейсы пользователя и т.д.);
  - **Adapters (Converters, Drivers)** — адаптеры, связывающие внешние службы с ядром приложения; преобразуют данные и вызывают методы ядра;
  - **Application Core (Business Logic)** — ядро приложения, содержащее бизнес-логику; сущности, значения и сервисы, представляющие бизнес-правила;
  - **Domain Logic (Entities, Value Objects, Services)** — основная бизнес-логика;
  - **External Interfaces** — интерфейсы для взаимодействия внешних компонентов с приложением (REST API, CLI, GUI и т.д.).

  Гексагональная архитектура помогает создавать гибкие, легко тестируемые и поддерживаемые приложения, разделяя различные аспекты системы
  и уменьшая зависимости между ними.

</details>

---

### Вопрос №33: [ Что такое микросервисы? ]

<details>
<summary>Ответ</summary>

* Микросервисы — это подход к разработке программного обеспечения, при котором большое приложение разбивается на
  меньшие, автономные компоненты. Каждый микросервис представляет собой отдельный модуль, который реализует определенный функционал и может
  работать независимо от других модулей.
  Эти модули обычно взаимодействуют друг с другом через API или событийно-ориентированную архитектуру.

</details>

---

### Вопрос №34: [ Какие преимущества у микросервисной архитектуры по сравнению с монолитом? А какие недостатки? ]

<details>
<summary>Ответ</summary>

* Преимущества:
  - Гибкость: Можно использовать разные технологии и языки программирования для разных микросервисов.
  - Масштабируемость: Легче масштабировать отдельные компоненты.
  - Распределение работы: Разные команды могут работать над разными сервисами параллельно.
  - Быстрый цикл разработки: Изменения в одном микросервисе могут быть развернуты независимо от других.

* Недостатки:
  - Сложность: Взаимодействие между микросервисами может стать сложным и трудным для управления.
  - Проблемы с данными: Труднее обеспечить консистентность данных между сервисами.
  - Сложность тестирования: Тестирование может быть сложнее, особенно для сценариев, которые требуют взаимодействия между множеством сервисов.
</details>

---

### Вопрос №35: [ Как быть с консистентностью данных между несколькими микросервисами? ]

<details>
<summary>Ответ</summary>

* Консистентность данных в микросервисной архитектуре — сложная задача. Один из подходов — использование распределенных транзакций, но это может привести к проблемам производительности и доступности.
  Другой подход — "eventual consistency", где система стремится обеспечить консистентность данных в течение некоторого времени.
  Для этого часто используют шины сообщений и системы очередей, такие как Kafka или RabbitMQ, чтобы синхронизировать данные между сервисами.

</details>

---

### Вопрос №36: [ Что такое service mesh? ]

<details>
<summary>Ответ</summary>

* Это инфраструктурный подход для решения задач взаимодействия между микросервисами. 
  Суть подхода состоит в использовании специальных приложений-прокси (Sidecar) — они запускаются рядом с каждым сервисом, перехватывают его 
      сетевые запросы и передают их "куда нужно".
  Sidecar не только обрабатывают коммуникации между сервисами, но и производят мониторинг и устраняют проблемы безопасности, то есть все, что
      может быть абстрагировано от отдельных сервисов.

  Mesh-сервис может предоставлять следующие возможности:
  - Шифрование.
  - Прозрачность.
  - Трассируемость.
  - Проверку подлинности.
  - Балансировку нагрузки.
  - Обнаружение сервисов.

  Service Mesh имеет свою собственную терминологию для компонентов-сервисов и функций:
  * Фреймворк оркестрации контейнеров. По мере того, как все больше и больше контейнеров добавляется в инфраструктуру приложения, появляется необходимость в отдельном инструменте для мониторинга и управления контейнерами – фреймворке оркестрации контейнеров. Kubernetes плотно занял эту нишу, причем настолько, что даже его основные конкуренты Docker Swarm и Mesosphere DC/OS предлагают в качестве альтернативы интеграцию с Kubernetes.
  * Сервисы и экземпляры (поды Kubernetes). Экземпляр – это единственная запущенная копия микросервиса. Иногда один экземпляр – это один контейнер. В Kubernetes экземпляр состоит из небольшой группы независимых контейнеров, который называется подом. Клиенты редко обращаются напрямую к экземпляру или поду, чаще они обращаются к сервису, который представляет из себя набор идентичных масштабируемых и отказоустойчивых экземпляров или подов (реплик).
  * Sidecar Proxy. Sidecar Proxy работает с одним экземпляром или подом. Смысл Sidecar Proxy в том, чтобы направлять или проксировать трафик, приходящий от контейнера, с которым он работает, и обратный трафик. Sidecar взаимодействует с другими Sidecar Proxy и управляется фреймворком оркестрации. Многие реализации Service Mesh используют Sidecar Proxy для перехвата и управления всем входящим и исходящим трафиком экземпляра или пода.
  * Обнаружение сервисов. Когда экземпляру нужно взаимодействовать с другим сервисом, ему нужно найти (обнаружить) исправный и доступный экземпляр другого сервиса. Как правило экземпляр выполняет поиск по DNS. Фреймворк оркестрации контейнеров хранит список экземпляров, которые готовы к получению запросов, и предоставляет интерфейс для DNS-запросов.
  * Балансировка нагрузки. Большинство фреймворков оркестрации контейнеров обеспечивают балансировку нагрузки на 4 уровне (транспортном). Service Mesh реализует более сложную балансировку нагрузки на 7 уровне (прикладном), богатую алгоритмами и более эффективную в вопросе управления трафиком. Параметры балансировки нагрузки могут быть изменены с помощью API, что позволяет оркестрировать сине-зеленое или канареечное развертывание.
  * Шифрование. Service Mesh может зашифровывать и расшифровывать запросы и ответы, снимая это бремя с сервисов. Service Mesh также может повысить производительность за счет приоритезации или переиспользования существующих постоянных соединений, что снижает необходимость в дорогих вычислениях для создания новых соединений. Наиболее распространенной реализацией шифрования трафика является mutual TLS (mTLS), где инфраструктура открытых ключей (PKI) генерирует и распространяет сертификаты и ключи для использования их в Sidecar Proxy.
  * Аутентификация и авторизация. Service Mesh может авторизовывать и аутентифицировать запросы сделанные снаружи или изнутри приложения, отправляя экземплярам только валидированные запросы.
  * Поддержка шаблона автоматического выключения. Service Mesh поддерживает шаблон автоматического выключания, который изолирует нездоровые экземпляры, а затем постепенно возвращает их в пул здоровых экземпляров при необходимости.

  Для реализации Service Mesh уже существует немало решений. Среди них лидерами являются — Envoy (как data plane), а Istio, Linkerd и Nginx Service Mesh (NSM) — как полноценные платформы Service Mesh поверх Envoy или собственных прокси.
  Также есть программные продукты, упрощающие внедрение стратегии Mesh, например Consul с технологией Connect от Hashicorp.

  Структурно, Service Mesh можно разделить на 2 части:
  * Data Plane — управление сетевым трафиком между всеми прокси.
  * Control Plane — распространение политики с настройками между всеми прокси.

</details>

---

### Вопрос №37: [ Что такое Event-Driven Architecture (EDA)? Плюсы и минусы ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **EDA** — архитектурный стиль, где компоненты общаются через события (асинхронно, decoupled).
  Producer публикует событие, не зная кто его потребляет; consumer реагирует на событие.

* **Плюсы:**
  - Loose coupling: producer не зависит от consumer;
  - Scalability: consumer можно масштабировать независимо;
  - Resilience: если consumer недоступен, события копятся в очереди;
  - Extensibility: новый consumer добавляется без изменения producer.

* **Минусы:**
  - Сложность отладки: event flow распределён, нет единого call stack;
  - Eventual consistency: данные расходятся во времени;
  - Eventual ordering: гарантия порядка сложна (partition key, sequence numbers);
  - Event schema evolution: старые события в новом формате → backward compatibility;
  - "Eventual complexity": со временем event chains становятся запутанными (spaghetti events).

* **Технологии:** Kafka, RabbitMQ, NATS, AWS EventBridge. Для Go: Watermill, Sarama, segmentio/kafka-go.

</details>

---

</details>
