# Redis, Docker, Kubernetes — Вопросы и ответы для Middle/Senior

Файл содержит вопросы по внутреннему устройству Redis, Docker и Kubernetes,
а также практические задачи и сценарии эксплуатации.

Уровни сложности:
- ![Static Badge](https://img.shields.io/badge/Middle-yellow) — Middle разработчик
- ![Static Badge](https://img.shields.io/badge/Senior-red) — Senior разработчик

---

# Оглавление

1. [Redis — внутреннее устройство](#1-redis--внутреннее-устройство)
2. [Redis — репликация и кластер](#2-redis--репликация-и-кластер)
3. [Redis — паттерны использования](#3-redis--паттерны-использования)
4. [Docker — внутреннее устройство](#4-docker--внутреннее-устройство)
5. [Kubernetes — архитектура и компоненты](#5-kubernetes--архитектура-и-компоненты)
6. [Kubernetes — эксплуатация и паттерны](#6-kubernetes--эксплуатация-и-паттерны)
7. [Практические задачи (Senior)](#7-практические-задачи-senior)
8. [Сравнение in-memory хранилищ: Redis vs Aerospike vs Memcached](#8-сравнение-in-memory-хранилищ-redis-vs-aerospike-vs-memcached)

---

## 1. Redis — внутреннее устройство

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Как устроен Redis внутри? Почему он такой быстрый? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Redis (REmote DIctionary Server)** — in-memory key-value хранилище, написанное на C.

**Почему быстрый:**
1. **In-memory:** все данные в RAM → нет дискового I/O для чтения (диск только для персистентности);
2. **Single-threaded event loop:** один поток обрабатывает все команды → нет overhead на блокировки, context switching, cache thrashing;
3. **I/O multiplexing:** epoll/kqueue → один поток обрабатывает тысячи соединений;
4. **Простые структуры данных:** Redis реализует собственные оптимизированные структуры (SDS, ziplist, quicklist, skiplist, hashtable);
5. **Нет парсинга SQL:** простой протокол RESP (REdis Serialization Protocol).

**Single-threaded — не значит медленный:**
- Команды выполняются последовательно, но каждая O(1) или O(log n);
- Узкое место — сеть и память, не CPU;
- В Redis 6+ — I/O threading (сеть в отдельных потоках, выполнение команд — в одном).

**Ограничения single-threaded:**
- Одна долгая команда (KEYS *, SUNION больших множеств) блокирует всех;
- Не использует многоядерность (нужен Redis Cluster для горизонтального масштабирования).

</details>

---

### Вопрос №2: [ Какие структуры данных используются внутри Redis? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

Redis экспонирует абстрактные типы, но внутри использует разные реализации в зависимости от размера:

| Тип Redis | Внутренняя реализация | Когда переключается |
|---|---|---|
| **String** | SDS (Simple Dynamic String) | всегда |
| **List** | ziplist (маленькие) → quicklist (большие) | > 128 элементов или > 64 байт/элемент |
| **Hash** | ziplist → hashtable | > 128 полей или > 64 байт/значение |
| **Set** | intset (только int) → hashtable | > 512 элементов или не-int значения |
| **ZSet** | ziplist → skiplist + hashtable | > 128 элементов или > 64 байт/значение |
| **Stream** | radix tree | всегда |
| **HyperLogLog** | dense/sparse representation | по плотности |

**Ziplist** — компактный contiguous блок памяти, отлично для маленьких списков (cache friendly, мало overhead). Минус: O(n) вставка/удаление в середину.

**Quicklist** — связный список ziplist'ов (компромисс между списком и массивом).

**Skiplist** — вероятностная структура для упорядоченных множеств. O(log n) поиск/вставка/удаление. Проще чем balanced BST, и range queries эффективны.

**intset** — отсортированный массив целых чисел. Бинарный поиск O(log n). Компактнее hashtable для маленьких int-множеств.

**Механизм переключения** — `hash-max-ziplist-entries`, `list-max-ziplist-size` и т.д. При превышении — автоматическая конвертация в более мощную структуру (односторонняя, обратно не возвращается).

</details>

---

### Вопрос №3: [ Как работает персистентность в Redis? RDB vs AOF ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**RDB (Redis Database) — снапшоты:**
- Периодически дампит весь dataset в бинарный файл `.rdb`;
- Триггеры: `save 900 1` (через 900с если ≥1 изменение), `save 300 10`, `save 60 10000`;
- **BGSAVE** — fork процесса, дочерний пишет RDB (parent продолжает обслуживать);
- Плюс: компактный, быстрый restore, хороший для backup;
- Минус: можно потерять данные между снапшотами (до нескольких минут).

**AOF (Append-Only File) — журнал команд:**
- Каждая записывающая команда дописывается в `.aof` файл;
- `appendfsync`:
  - `always` — sync на каждую команду (максимальная durability, медленно);
  - `everysec` (по умолчанию) — sync раз в секунду (компромисс);
  - `no` — OS решает когда sync (быстро, можно потерять до 30с данных);
- **AOF Rewrite** — периодически переписывает AOF (убирает устаревшие команды, компактно);
- Плюс: минимальная потеря данных (до 1с при everysec);
- Минус: файл больше RDB, restore медленнее.

**Гибрид (рекомендуется):**
- RDB + AOF одновременно;
- При restart: загружает AOF (в нём последняя актуальная часть, при `aof-use-rdb-preamble yes` — AOF начинается с RDB-снапшота + последующие команды).

**Redis 4.0+:** `aof-use-rdb-preamble` — AOF rewrite создаёт RDB-префикс + incrementальные команды. Быстрее восстановление.

</details>

---

### Вопрос №4: [ Что такое eviction policies в Redis? Какие бывают? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Eviction policy** — стратегия удаления ключей при нехватке памяти (`maxmemory`).

| Политика | Описание |
|---|---|
| `noeviction` | Не удалять — возвращать ошибку на запись (по умолчанию в Redis 4+) |
| `allkeys-lru` | LRU среди всех ключей |
| `allkeys-lfu` | LFU среди всех ключей (Redis 4.0+) |
| `allkeys-random` | Случайное удаление среди всех |
| `volatile-lru` | LRU только среди ключей с TTL |
| `volatile-lfu` | LFU только среди ключей с TTL |
| `volatile-random` | Случайное среди ключей с TTL |
| `volatile-ttl` | Удалять ключи с ближайшим TTL |

**LRU (Least Recently Used):**
- Redis использует **approximated LRU** — не точный LRU (дорого), а выборка из N случайных ключей (`maxmemory-samples`, default 5);
- Увеличение `maxmemory-samples` → точнее, но медленнее.

**LFU (Least Frequently Used) — Redis 4.0+:**
- Отслеживает частоту доступа (не время);
- Морицованный счётчик (8 бит): логарифмическая вероятность, затухание со временем;
- Лучше для skew-распределения (одни ключи очень популярны, другие — нет).

**Когда что:**
- **Cache (general):** `allkeys-lru` или `allkeys-lfu`;
- **Session store:** `volatile-lru` (только сессии с TTL);
- **Mixed (cache + persistent):** `volatile-lru` + явный TTL для cache-ключей.

</details>

---

### Вопрос №5: [ Что такое Pub/Sub в Redis? Ограничения ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Pub/Sub** — механизм публикации/подписки на сообщения.

```redis
# Подписчик
SUBSCRIBE channel1

# Издатель
PUBLISH channel1 "Hello"
```

**Особенности:**
- **Fire-and-forget:** если подписчик не在线 — сообщение потеряно (нет persistence);
- **Fan-out:** сообщение доставляется всем активным подписчикам;
- **No ack:** нет подтверждения получения;
- **Шаблоны:** `PSUBSCRIBE chan*` — подписка по pattern;
- **Streams** (Redis 5.0+) — более мощная альтернатива с persistence, consumer groups, ack.

**Ограничения:**
- Сообщения не сохраняются → нет redelivery;
- Нет consumer groups (только в Streams);
- Медленные подписчики не блокируют publisher, но теряют данные;
- В кластере: сообщения не шардируются — публикуются на все узлы, знающие о канале.

**Redis Streams (рекомендуется вместо Pub/Sub):**
```redis
XADD mystream * field1 value1
XREAD COUNT 10 STREAMS mystream 0
XGROUP CREATE mystream group1 0
XREADGROUP GROUP group1 consumer1 COUNT 10 STREAMS mystream >
XACK mystream group1 <id>
```
- Persistence, consumer groups, at-least-once delivery, pending entries list (PEL).

</details>

---

### Вопрос №6: [ Как работают Lua-скрипты в Redis? Зачем? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**Lua-скрипты** — атомарное выполнение нескольких команд без конкурентного вмешательства.

```redis
EVAL "return redis.call('INCR', KEYS[1])" 1 counter
```

**Зачем:**
1. **Атомарность:** скрипт выполняется как единая операция (никакая другая команда не вклинится);
2. **Производительность:** несколько команд за один round-trip (network latency);
3. **Условная логика:** CAS-подобные операции, условные обновления.

**Пример — атомарный compare-and-swap:**
```lua
-- key = "balance", expected = 100, new = 50
local current = redis.call('GET', KEYS[1])
if current == ARGV[1] then
    redis.call('SET', KEYS[1], ARGV[2])
    return 1
else
    return 0
end
```

```redis
EVAL "local c=redis.call('GET',KEYS[1]) if c==ARGV[1] then redis.call('SET',KEYS[1],ARGV[2]) return 1 else return 0 end" 1 balance 100 50
```

**EVALSHA** — хэш скрипта вместо текста (компактнее):
```redis
SCRIPT LOAD "return ..."  → "a1b2c3..."
EVALSHA a1b2c3... 1 key1 arg1
```

**Ограничения:**
- **Блокирует event loop** — долгий скрипт парализует Redis (максимум `lua-time-limit`, default 5s);
- **Нет I/O:** скрипт не может делать сетевые запросы;
- **Determinism:** нельзя использовать `TIME`, `RANDOMKEY` (результат должен быть детерминированным для replication);
- **Отладка:** `redis-cli --ldb` — step debugger для Lua.

**Redis 7.0+:** Redis Functions — улучшенные Lua-скрипты с библиотеками, better management.

</details>

---

</details>

---

## 2. Redis — репликация и кластер

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №7: [ Как работает репликация в Redis? Master-replica ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Redis replication** — async master-replica репликация.

**Процесс:**
1. Replica отправляет `REPLICAOF master_ip master_port`;
2. Master начинает **full sync**: делает BGSAVE → отправляет RDB → replica загружает;
3. После RDB — master стримит **backlog** (команды, накопленные во время sync);
4. Далее — **continuous sync**: master отправляет каждую write-команду репликам.

**Replication backlog:**
- Кольцевой буфер в памяти master (default 1MB);
- Если replica отключилась и переподключилась — может догнать из backlog (partial resync);
- Если backlog переполнен → full resync (дорого);

**Параметры:**
- `repl-backlog-size` — размер backlog (увеличить для стабильности);
- `repl-backlog-ttl` — TTL backlog без реплик (default 3600s);
- `min-replicas-to-write` — мастер не принимает запись, если меньше N реплик (semi-sync).

**Топологии:**
- **Star:** master → N replicas (просто, но master нагружен);
- **Chain:** master → replica1 → replica2 (снижает нагрузку master, но добавляет latency);
- **Sentinel:** для HA (см. вопрос №8).

**Важно:**
- Репликация **асинхронная** → replication lag → eventual consistency;
- **Read from replica** — можно, но данные могут быть устаревшими;
- Реплика не поддерживает запись (read-only по умолчанию).

</details>

---

### Вопрос №8: [ Что такое Redis Sentinel? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Redis Sentinel** — система для HA (High Availability) Redis: мониторинг, automatic failover, configuration provider.

**Функции:**
1. **Monitoring:** Sentinels постоянно проверяют master и replicas (PING);
2. **Notification:** уведомляют администратора/приложение о событиях;
3. **Automatic failover:** если master недоступен → выбирают новый master из replicas;
4. **Configuration provider:** клиенты спрашивают Sentinel "кто сейчас master?".

**Архитектура:**
```
                   ┌──────────┐
         ┌────────►│ Sentinel1│◄───────┐
         │         └──────────┘        │
    ┌────┴───┐    ┌──────────┐    ┌────┴───┐
    │ Master │◄──►│ Sentinel2│◄──►│Replica1│
    └────────┘    └──────────┘    └────────┘
         │         ┌──────────┐        │
         └────────►│ Sentinel3│◄───────┘
                   └──────────┘
```

**Failover process:**
1. Sentinels отмечают master как `SDOWN` (subjective down — один sentinel не отвечает);
2. Кворум Sentinels переводит в `ODOWN` (objective down — несколько согласны);
3. Выбирается leader-sentinel для failover;
4. Leader выбирает лучшую replica (по приоритету, offset, runid);
5. Replica → master: `REPLICAOF NO ONE`;
6. Другие реплики переключаются на нового master;
7. Клиенты уведомляются (через pub/sub или запрос к Sentinel).

**Параметры:**
- `quorum` — сколько Sentinels должны согласиться для ODOWN (обычно N/2+1);
- `down-after-milliseconds` — через сколько считать down (default 30s);
- `failover-timeout` — максимальное время failover (default 180s);

**Минимум 3 Sentinel'а** в разных AZ для кворума без split-brain.

</details>

---

### Вопрос №9: [ Как устроен Redis Cluster? Шардинг ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**Redis Cluster** — встроенное шардирование с automatic failover.

**Шардинг — hash slots:**
- 16384 hash slots (0–16383);
- `slot = CRC16(key) % 16384`;
- Каждый узел владеет диапазоном слотов (например, узел 1: 0–5460, узел 2: 5461–10922, узел 3: 10923–16383);

**Multi-key операции:**
- `MGET k1 k2 k3` — работает только если все ключи на одном слоте;
- **Hash tags:** `{user:1}:profile`, `{user:1}:orders` — скобки `{}` определяют хэшируемую часть → один слот;

**Топология:**
- Минимум 3 master + 3 replica (6 узлов);
- Каждый master имеет хотя бы одну replica;
- Gossip protocol для обнаружения узлов и health checks.

**MOVED redirect:**
```redis
SET k1 v1
→ (error) MOVED 1234 192.168.1.5:6379  # слот 1234 на узле 192.168.1.5
```
Клиент кэширует слот-маппинг и при MOVED обновляет его.

**ASK redirect (во время resharding):**
```redis
GET k1
→ (error) ASK 1234 192.168.1.5:6379
```
Временный redirect — клиент должен отправить `ASKING` перед командой. Не обновляет кэш.

**Failover в Cluster:**
- Replicas обнаруживают master down через gossip;
- Election: replica запрашивает голоса у master-узлов;
- Majority голосов → replica становится master;
- Быстрее чем Sentinel (встроено, без отдельного процесса).

**Ограничения Cluster:**
- **Нет DB 1-15:** только DB 0;
- **Multi-key:** только same-slot;
- **Transactions (MULTI/EXEC):** только same-slot;
- **Pub/Sub:** broadcast на все узлы (в Redis 7.0+ — sharded pub/sub);
- **Lua-скрипты:** только same-slot ключи.

</details>

---

### Вопрос №10: [ Что такое Redis Stream? Consumer Groups ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**Redis Stream** — log data structure для messaging (Redis 5.0+). Альтернатива Kafka для лёгких use cases.

**Создание и добавление:**
```redis
XADD orders * id 1001 amount 500
# * = auto-generated ID (timestamp-sequence)
# → "1672531200000-0"
```

**Чтение:**
```redis
XRANGE orders - +           # все записи
XRANGE orders 1672531200000-0 + COUNT 10  # с ID, лимит 10
XLEN orders                 # количество записей
```

**Consumer Groups:**
```redis
# Создать группу
XGROUP CREATE orders order_processor $  # $ = с конца, 0 = с начала

# Читать (> = только новые, ещё не прочитанные этой группой)
XREADGROUP GROUP order_processor worker-1 COUNT 10 BLOCK 5000 STREAMS orders >

# Подтвердить обработку
XACK orders order_processor 1672531200000-0

# Pending entries (не acked) — для recovery
XPENDING orders order_processor
```

**Guarantees:**
- **At-least-once:** сообщение остаётся в PEL пока не acked;
- **Exactly-once:** нужно idempotent consumer (dedup по message ID);
- **Ordering:** гарантирован в пределах одного stream;
- **Persistence:** данные в памяти + RDB/AOF;

**Dead Letter Queue pattern:**
```redis
# Если сообщение не обработано после N попыток → переместить в DLQ
XCLAIM orders order_processor worker-2 3600000 <id>  # забрать через 1h
# Если и тогда не обработано → XADD в DLQ stream + XACK из основного
```

**Redis Stream vs Kafka:**

| | Redis Stream | Kafka |
|---|---|---|
| Persistence | Memory + RDB/AOF | Disk (WAL) |
| Volume | GB (RAM) | TB (disk) |
| Retention | MAXLEN / trimming | Time/size based |
| Consumer groups | Да | Да |
| Partitions | Нет (один stream) | Да (параллелизм) |
| Throughput | ~100K/s | ~1M/s+ |
| Latency | < 1ms | ~5-10ms |

</details>

---

</details>

---

## 3. Redis — паттерны использования

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №11: [ Паттерн Cache-Aside vs Write-Through vs Write-Behind ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**1. Cache-Aside (Lazy Loading):**
```
App → Cache? → MISS → DB → Cache → App
App → Cache? → HIT → App
```
- Приложение само управляет кэшем;
- Запись: обновить DB, инвалидировать кэш (DELETE);
- Плюс: просто, кэш содержит только нужные данные;
- Минус: cache miss → 2 запроса (cache + DB), stale data возможна.

**2. Write-Through:**
```
App → Cache → DB → Cache → App
```
- Запись идёт через кэш → кэш всегда актуален;
- Плюс: нет stale data, простой read (всегда из кэша);
- Минус: latency записи (2 операции), кэш содержит всё (даже редко читаемое).

**3. Write-Behind (Write-Back):**
```
App → Cache → (async) → DB
```
- Запись в кэш → асинхронно в DB;
- Плюс: минимальная latency записи, batch writes в DB;
- Минус: потеря данных при crash кэша, complex implementation.

**Инвалидация кэша (cache-aside):**
```redis
# При обновлении в DB — удалить из кэша (не обновить!)
DEL user:123
# Следующий read — cache miss → загрузит свежие данные из DB
```
**Почему DELETE, а не SET?** При DELETE — следующий read гарантированно загрузит свежие данные. При SET — race condition: два writer'а могут перезаписать друг друга в неправильном порядке.

</details>

---

### Вопрос №12: [ Как реализовать распределённый lock в Redis? Redlock ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**Базовый lock:**
```redis
SET lock:resource {unique_value} NX PX 30000
# NX = only if not exists, PX = TTL 30s
# unique_value = UUID (для безопасного снятия lock)
```

**Снятие lock (atomic, с проверкой value):**
```lua
-- unlock.lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
```

**Проблема базового lock (single instance):**
- Если Redis упал → lock потерян → multiple holders;
- Репликация async → failover может дать lock другому.

**Redlock algorithm (Martin Kleppmann critique):**
1. Клиент запрашивает lock на N (обычно 5) независимых Redis-узлов;
2. `SET NX PX` на каждый с коротким timeout;
3. Если получено большинство (≥3 из 5) за разумное время → lock acquired;
4. Время действия lock = TTL - время на acquire;
5. Release: отправить DEL на все узлы.

**Критика Redlock (Kleppmann):**
- **GC pause:** клиент получил lock → GC pause → lock истёк → другой клиент получил → два holder'а;
- **Clock drift:** NTP sync может сдвинуть время → lock истекает раньше;
- **Network partition:** аналогичные проблемы.

**Когда Redlock OK:**
- Для efficiency (не correctness) — например, предотвращение дублирования работы;
- Не для correctness-critical сценариев (финансы, booking) — там нужен fencing token или consensus (etcd/ZooKeeper).

**Fencing token (надёжная альтернатива):**
- Каждый lock содержит monotonically increasing token;
- Сервис хранит последний виденный token → отклоняет старые;
- Защита от stale lock holder'а даже при GC pause.

</details>

---

### Вопрос №13: [ Как избежать cache stampede? Singleflight в Redis ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**Cache stampede (thundering herd):**
- TTL ключа истёк → 1000 запросов одновременно → cache miss → все идут в DB → DB падает.

**Решения:**

**1. Request coalescing (singleflight):**
```redis
# Первый запрос устанавливает lock, остальные ждут
SET lock:resource 1 NX EX 5
# Если OK → загрузить из DB → SET в кэш → DEL lock
# Если не OK → спать 100ms → повторить GET из кэша
```

В Go: `golang.org/x/sync/singleflight` — in-process, для distributed — Redis lock.

**2. Probabilistic early expiration (XFetch):**
```go
// Обновить кэш раньше истечения TTL с определённой вероятностью
func shouldRefresh(ttl, expiration time.Duration) bool {
    if ttl > expiration {
        return false // ещё рано
    }
    // Вероятность растёт по мере приближения к expiration
    probability := 1.0 - float64(ttl) / float64(expiration)
    return rand.Float64() < probability
}
```
- Распределяет refresh во времени → нет одновременного expiry.

**3. Stale-while-revalidate:**
```redis
# Хранить два значения: data + "stale" флаг
SET cache:resource {data} EX 60    # TTL 60s
SET stale:resource 1 EX 120        # TTL 120s (дольше)

# Read:
GET cache:resource
# Если miss, но stale:resource есть → вернуть stale + async refresh
# Если оба miss → sync load from DB
```

**4. Jittered TTL:**
```redis
# Добавить случайный offset к TTL — ключи не истекают одновременно
SET cache:resource {data} EX $((60 + RANDOM % 30))  # 60-90s
```

</details>

---

### Вопрос №14: [ Rate limiter на Redis. Token bucket через Lua ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**Token bucket через Lua (атомарный):**
```lua
-- rate_limiter.lua
-- KEYS[1] = bucket key
-- ARGV[1] = capacity (max tokens)
-- ARGV[2] = refill_rate (tokens per second)
-- ARGV[3] = now (current timestamp in ms)
-- ARGV[4] = requested (tokens to consume, usually 1)

local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

-- Get current bucket state
local bucket = redis.call('HMGET', key, 'tokens', 'timestamp')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Refill tokens based on elapsed time
local elapsed = math.max(0, now - last_refill) / 1000.0  -- seconds
tokens = math.min(capacity, tokens + elapsed * refill_rate)

-- Check if enough tokens
local allowed = 0
if tokens >= requested then
    tokens = tokens - requested
    allowed = 1
end

-- Save state
redis.call('HMSET', key, 'tokens', tokens, 'timestamp', now)
redis.call('EXPIRE', key, math.ceil(capacity / refill_rate * 2))  -- auto-cleanup

return allowed
```

**Использование:**
```redis
SCRIPT LOAD "..."  → "sha123..."
EVALSHA sha123... 1 rate:user:123 100 10 1690000000000 1
# → 1 (allowed) или 0 (denied)
```

**Почему Lua:**
- HMGET + вычисления + HMSET — атомарны (никакая другая команда не вклинится);
- Без Lua: race condition между read и write (два клиента видят одинаковые tokens).

**Sliding window counter (проще):**
```redis
-- Каждый запрос инкрементирует счётчик в окне
INCR rate:user:123:1690000000   # ключ = user + second
EXPIRE rate:user:123:1690000000 2
# Если INCR > limit → deny
```

</details>

---

</details>

---

## 4. Docker — внутреннее устройство

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №15: [ Как устроен Docker внутри? Namespaces, cgroups, unionfs ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

Docker — не виртуальная машина. Это набор Linux-технологий для изоляции процессов:

**1. Namespaces — изоляция ресурсов:**
| Namespace | Что изолирует |
|---|---|
| **PID** | Идентификаторы процессов (контейнер видит только свои) |
| **NET** | Сетевой стек (интерфейсы, routing, ports) |
| **IPC** | Межпроцессное взаимодействие (shared memory, semaphores) |
| **MNT** | Точки монтирования файловых систем |
| **UTS** | Hostname, domain name |
| **USER** | UID/GID mapping (root в контейнере ≠ root на хосте) |
| **Cgroup** | Вид cgroup иерархии |

**2. Cgroups (Control Groups) — ограничение ресурсов:**
- CPU quota (`--cpus=2`);
- Memory limit (`--memory=512m`);
- I/O limits (`--device-write-bps`);
- PIDs limit (`--pids-limit=200`).

**3. UnionFS (Overlay2) — слоистая файловая система:**
```
┌─────────────────────┐
│  Container layer    │  (read-write, ephemeral)
├─────────────────────┤
│  Image layer 3      │  (app code)
├─────────────────────┤
│  Image layer 2      │  (dependencies)
├─────────────────────┤
│  Image layer 3      │  (base OS)
└─────────────────────┘
```
- Каждый слой — только отличия от предыдущего;
- Нижние слои read-only, верхний — read-write (copy-on-write);
- При модификации файла из нижнего слоя → копируется в верхний (CoW).

**Docker daemon (dockerd):**
- REST API;
- Управляет images, containers, networks, volumes;
- containerd — runtime (runC) для непосредственного запуска;
- В Docker 1.11+ — split на dockerd → containerd → runC.

</details>

---

### Вопрос №16: [ Чем отличается COPY от ADD в Dockerfile? Лучшие практики ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

| | COPY | ADD |
|---|---|---|
| Копирование файлов | Да | Да |
| Распаковка архивов (.tar.gz) | Нет | Да (автоматически) |
| URLs | Нет | Да (скачивает) |
| Рекомендуется | **Да** | Только для архивов/URLs |

**Лучшие практики Dockerfile:**

```dockerfile
# 1. Используй конкретный tag, не :latest
FROM golang:1.22-alpine AS builder

# 2. COPY вместо ADD
COPY go.mod go.sum ./
RUN go mod download
COPY . .

# 3. Объединяй RUN (меньше слоёв)
RUN apk add --no-cache git ca-certificates && \
    update-ca-certificates

# 4. Multi-stage build (маленький финальный image)
FROM alpine:3.19
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]

# 5. .dockerignore (не копировать .git, vendor, кэш)
# .dockerignore:
# .git
# vendor
# *.md

# 6. Non-root user
RUN adduser -D -u 1001 appuser
USER appuser

# 7. EXPOSE для документации (не публикует порт)
EXPOSE 8080

# 8. ENTRYPOINT + CMD для гибкости
ENTRYPOINT ["/server"]
CMD ["--config", "/etc/config.yaml"]

# 9. HEALTHCHECK
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD wget -qO- http://localhost:8080/health || exit 1
```

**Порядок слоёв — важен для кэша:**
- `COPY go.mod go.sum` + `RUN go mod download` — отдельный слой (кэшируется, пока deps не изменились);
- `COPY . .` — меняется часто (код) → invalidate кэш для последующих слоёв.

</details>

---

### Вопрос №17: [ Что такое multi-stage build? Зачем? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Multi-stage build** — несколько `FROM` в одном Dockerfile, финальный image содержит только нужное.

```dockerfile
# Stage 1: Build
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o server ./cmd/server

# Stage 2: Runtime (минимальный image)
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

**Размер image:**
- `golang:1.22` — ~800MB (компилятор, исходники, инструменты);
- `alpine:3.19` — ~7MB;
- `distroless/static` — ~2MB;
- **Multi-stage итог:** ~2MB + бинарник (~15MB) = ~17MB.

**Зачем:**
1. **Меньше image** → быстрее pull/push, меньше storage;
2. **Безопасность** — нет компилятора, shell, лишних пакетов в production;
3. **Меньше attack surface** — distroless не имеет shell → нельзя `docker exec -it ... sh`.

**distroless vs scratch vs alpine:**
- `scratch` — пустой image (0MB), нужно статически слинковать (CGO_ENABLED=0);
- `distroless` — scratch + CA certificates + timezone + minimal libraries;
- `alpine` — ~7MB, есть `apk`, shell, busybox — удобнее для debugging, но больше attack surface.

</details>

---

### Вопрос №18: [ Как работают Docker networks? Bridge, host, overlay ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**1. Bridge (default):**
- Виртуальный мост `docker0` на хосте;
- Каждый контейнер получает veth-пару (виртуальный ethernet);
- NAT (iptables MASQUERADE) для исходящего трафика;
- Port publishing: `-p 8080:80` → DNAT правило;
- Контейнеры на одном bridge могут общаться по IP (не по имени, если нет DNS);
- **User-defined bridge** (рекомендуется): `docker network create mynet` — встроенный DNS, контейнеры резолвятся по имени.

**2. Host:**
- Контейнер использует сеть хоста напрямую (нет изоляции);
- `--network host` → порт 80 в контейнере = порт 80 на хосте;
- Быстрее (нет NAT), но нет изоляции;
- Полезно для performance-critical приложений.

**3. Overlay:**
- Для Docker Swarm / multi-host;
- VXLAN-туннели между хостами;
- Контейнеры на разных машинах общаются как в одной сети;
- Требует key-value store (встроен в Swarm mode).

**4. Macvlan:**
- Контейнеру назначается MAC-адрес → выглядит как отдельное устройство в LAN;
- Полезно для legacy-приложений, ожидающих физический интерфейс.

**5. None:**
- Нет сети (`--network none`);
- Только loopback;
- Для изолированных задач (вычисления, batch).

</details>

---

</details>

---

## 5. Kubernetes — архитектура и компоненты

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №19: [ Как устроена архитектура Kubernetes? Control Plane и Worker ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

```
┌─────────────────────────── Control Plane (Master) ────────────────────────────┐
│                                                                                │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ API Server│  │   etcd   │  │ Scheduler │  │CM (Cloud)│  │  Kube        │  │
│  │          │  │          │  │           │  │          │  │  Controller  │  │
│  │ (gateway)│  │ (state)  │  │ (assigns) │  │ (cloud)  │  │  Mgr         │  │
│  └────┬─────┘  └──────────┘  └───────────┘  └──────────┘  └──────────────┘  │
│       │                                                                        │
└───────┼────────────────────────────────────────────────────────────────────────┘
        │
    ┌───┴─────────────────┐
    │                     │
┌───▼──── Worker Node ───▼───┐       ┌──────── Worker Node ────────┐
│                             │       │                              │
│  ┌──────────┐  ┌─────────┐ │       │  ┌──────────┐  ┌─────────┐  │
│  │ Kubelet  │  │ Kube-   │ │       │  │ Kubelet  │  │ Kube-   │  │
│  │          │  │ Proxy   │ │       │  │          │  │ Proxy   │  │
│  └────┬─────┘  └─────────┘ │       │  └────┬─────┘  └─────────┘  │
│       │                     │       │       │                     │
│  ┌────▼──────────────────┐ │       │  ┌────▼──────────────────┐  │
│  │ Container Runtime     │ │       │  │ Container Runtime     │  │
│  │ (containerd / CRI-O)  │ │       │  │ (containerd / CRI-O)  │  │
│  └───────────────────────┘ │       │  └───────────────────────┘  │
└─────────────────────────────┘       └─────────────────────────────┘
```

**Control Plane:**
- **API Server** — единственная точка входа (REST API); все компоненты общаются через него;
- **etcd** — distributed key-value store (Raft consensus); единственный source of truth;
- **Scheduler** — выбирает node для новых pods (resource requests, taints, affinity);
- **Controller Manager** — runs controllers (Deployment, ReplicaSet, Node, Endpoint);
- **Cloud Controller Manager** — интеграция с cloud (load balancers, volumes, routes).

**Worker Node:**
- **Kubelet** — агент на node; следит за pods (health, restart), сообщает статус API Server;
- **Kube Proxy** — управляет iptables/IPVS rules для service discovery и load balancing;
- **Container Runtime** — containerd, CRI-O (Docker deprecated since K8s 1.24).

**Поток создания Pod:**
1. `kubectl apply` → API Server → etcd;
2. Scheduler watches → выбирает node → обновляет `pod.spec.nodeName`;
3. Kubelet на выбранной node видит новый pod → pulls image → creates containers;
4. Kubelet reports status → API Server → etcd.

</details>

---

### Вопрос №20: [ Что такое Pod? Почему не «контейнер»? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Pod** — минимальная развёртываемая единица в K8s. Содержит 1+ контейнеров, разделяющих:
- Network namespace (один IP, общие порты);
- IPC namespace (shared memory);
- Volumes (shared storage);
- Но **не** PID namespace (по умолчанию; можно включить `shareProcessNamespace: true`).

**Почему Pod, а не контейнер:**
- K8s абстрагирует контейнерный runtime → Pod — wrapper над контейнером;
- Multi-container pods: main container + sidecar (logging, proxy, config reloader);
- Pod — unit of scaling (replicas = pod copies).

**Multi-container patterns:**

**1. Sidecar:**
```yaml
spec:
  containers:
    - name: app
      image: myapp
    - name: logger
      image: fluent-bit
      # Читает логи app из shared volume
```

**2. Ambassador:**
```yaml
spec:
  containers:
    - name: app
      image: myapp
    - name: proxy
      image: envoy
      # app обращается к localhost:5432 → proxy → внешняя БД
```

**3. Adapter:**
```yaml
spec:
  containers:
    - name: app
      image: myapp
    - name: adapter
      image: prometheus-exporter
      # Трансформирует метрики app в формат Prometheus
```

**Pod lifecycle:**
```
Pending → Running → Succeeded / Failed
                ↕
            CrashLoopBackOff (repeated failures)
```

</details>

---

### Вопрос №21: [ Что такое Service в K8s? Виды (ClusterIP, NodePort, LoadBalancer) ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Service** — абстракция, обеспечивающая стабильный сетевой адрес для группы pods (которые могут создаваться/удаляться).

**Виды:**

**1. ClusterIP (default):**
- Виртуальный IP внутри кластера;
- Недоступен снаружи;
- Для внутреннего общения между сервисами.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 8080
      targetPort: 8080
```

**2. NodePort:**
- Открывает порт (30000–32767) на всех nodes;
- `nodeIP:nodePort` → service;
- Базовый external access (обычно не используется в prod напрямую).

**3. LoadBalancer:**
- Создаёт cloud load balancer (AWS ELB, GCP LB, etc.);
- `externalIP` → service;
- Используется для prod external access;
- В bare-metal — MetalLB или ingress controller.

**4. ExternalName:**
- CNAME-запись → внешний DNS;
- Не создаёт proxy → просто DNS alias;
- Полезно для migration (переключение с внешнего сервиса на внутренний).

**Endpoints:**
- Service → набор endpoints (pod IP:port);
- Kube Proxy управляет iptables/IPVS rules;
- **Headless service** (`clusterIP: None`) — нет виртуального IP, DNS возвращает pod IPs напрямую (для StatefulSet).

</details>

---

### Вопрос №22: [ Что такое Ingress? Ingress Controller ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Ingress** — HTTP/L7 routing правилa, направляющие внешний трафик в сервисы кластера.

```
Internet → [Ingress Controller] → Ingress rules → Service → Pod
                (NGINX, Traefik, ALB)
```

**Ingress resource:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts: [api.example.com]
      secretName: tls-secret
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 8080
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8080
```

**Ingress Controller** — pod, реализующий Ingress rules (по умолчанию K8s не включает):
- **NGINX Ingress** — самый популярный, nginx + Lua;
- **Traefik** — Go-based, auto-discovery, Let's Encrypt;
- **AWS ALB Ingress** — native AWS ALB;
- **HAProxy Ingress** — для high-throughput.

**Зачем:**
- Один LB для множества сервисов (экономия cloud LB);
- TLS termination в одном месте;
- Path-based routing, virtual hosts;
- Rate limiting, auth, rewriting (через annotations).

</details>

---

### Вопрос №23: [ Что такое etcd? Почему Raft? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**etcd** — distributed key-value store, используемый K8s для хранения всего состояния кластера.

**Raft consensus algorithm:**
- Один узел — **leader**, остальные — **followers**;
- Все записи идут через leader → followers → quorum ack → commit;
- Leader election при отказе (randomized timeout);
- **Quorum = N/2 + 1** (3 узла → терпит 1 отказ, 5 узлов → 2 отказа).

**Что хранит etcd в K8s:**
- Pods, Services, Deployments, ConfigMaps, Secrets;
- Node status, cluster config;
- Всё — в etcd, K8s stateless без него.

**Рекомендации:**
- **3 или 5 узлов** (нечётное — для кворума);
- **Dedicated nodes** (не смешивать с workload);
- **SSD storage** (etcd чувствителен к disk latency);
- **Backup** — `etcdctl snapshot save` регулярно;
- Восстановление: `etcdctl snapshot restore`.

**Опасности:**
- etcd недоступен → K8s "замораживается" (нет новых pods, но существующие работают);
- etcd corruption → потеря всего состояния кластера (нужен backup!);
- Compression: etcd хранит историю ревизий → нужно `--auto-compaction` (default 5m revisions).

</details>

---

</details>

---

## 6. Kubernetes — эксплуатация и паттерны

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №24: [ Что такое Deployment, StatefulSet, DaemonSet, Job/CronJob? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

| Resource | Назначение | When to use |
|---|---|---|
| **Deployment** | Stateless pods | Web servers, API, workers |
| **StatefulSet** | Stateful pods (stable identity + storage) | Databases, message queues |
| **DaemonSet** | Один pod на каждой node | Logging agent, monitoring, CNI |
| **Job** | Run to completion (batch) | Migrations, data processing |
| **CronJob** | Scheduled Job | Backups, cleanup, reports |

**Deployment:**
```yaml
spec:
  replicas: 3
  selector:
    matchLabels: {app: api}
  template:
    metadata:
      labels: {app: api}
    spec:
      containers:
        - name: api
          image: api:v2
          resources:
            requests: {cpu: 100m, memory: 128Mi}
            limits: {cpu: 500m, memory: 256Mi}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```
- Pods безымянные (`api-abc123`, `api-def456`) — можно заменять свободно;
- `RollingUpdate` (по умолчанию) или `Recreate`.

**StatefulSet:**
- Pods имеют **stable identity**: `db-0`, `db-1`, `db-2`;
- **Stable storage**: PVC `data-db-0`, `data-db-1` (не теряется при пересоздании pod);
- **Ordered**: создаются/удаляются последовательно (0→1→2);
- **Headless Service** (`clusterIP: None`) → DNS: `db-0.db.default.svc.cluster.local`;
- Для: PostgreSQL, Redis, Kafka, Elasticsearch.

**DaemonSet:**
```yaml
spec:
  selector:
    matchLabels: {app: fluent-bit}
  template:
    spec:
      containers:
        - name: fluent-bit
          image: fluent-bit:3.0
```
- Один pod на каждой (или выбранной) node;
- Автоматически добавляется при добавлении новой node;
- Для: logging, monitoring, network plugins (CNI), storage plugins (CSI).

**Job:**
```yaml
spec:
  completions: 1       # сколько успешных выполнений
  parallelism: 1       # параллельно
  backoffLimit: 3      # retry при failure
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: migrate:v1
```

</details>

---

### Вопрос №25: [ Как работают resources (requests/limits) в K8s? QoS classes ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**Requests** — что scheduler гарантирует (для размещения pod на node).
**Limits** — максимальный предел (cgroup enforcement).

```yaml
resources:
  requests:
    cpu: 100m      # 0.1 core (millicores)
    memory: 128Mi
  limits:
    cpu: 500m      # 0.5 core max
    memory: 256Mi  # hard limit
```

**CPU:**
- Request → CPU shares (CFS scheduler);
- Limit → CPU quota (throttle при превышении, `cpu.cfs_quota_us`);
- 100m = 100 millicores = 0.1 CPU.

**Memory:**
- Request → учитывается при scheduling (node allocatable);
- Limit → cgroup `memory.limit_in_bytes`;
- Превышение → **OOMKill** (container перезапускается).

**QoS (Quality of Service) classes:**

| Class | Условие | Eviction priority |
|---|---|---|
| **Guaranteed** | requests == limits (CPU и memory) | Последний (best) |
| **Burstable** | requests < limits (или не все set) | Средний |
| **BestEffort** | Нет requests и limits | Первый (first evicted) |

**Node pressure → eviction order:**
1. BestEffort pods;
2. Burstable pods (превышившие request);
3. Burstable pods (в рамках request) — редко.

**CPU throttling vs OOMKill:**
- CPU throttle → замедление (признак: high `container_cpu_cfs_throttled_seconds_total`);
- Memory OOMKill → container restart (признак: `OOMKilled` in `kubectl describe pod`).

**Рекомендации:**
- Всегда ставь requests (без них scheduler не учитывает нагрузку → overcommit);
- Limits для CPU — осторожно (throttle может деградировать latency);
- Limits для memory — обязательно (без limit → OOMKill node);
- HPA (Horizontal Pod Autoscaler) работает от requests (CPU utilization = actual / request).

</details>

---

### Вопрос №26: [ Что такое ConfigMap и Secret? Как безопасно передавать конфиги? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**ConfigMap** — конфигурация в виде ключ-значение (plaintext).
**Secret** — то же, но для чувствительных данных (base64-encoded).

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "debug"
  config.yaml: |
    server:
      port: 8080
```

```yaml
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=   # base64("password123")
```

**Способы подключения:**

**1. Environment variables:**
```yaml
spec:
  containers:
    - name: app
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
      envFrom:
        - configMapRef:
            name: app-config
```

**2. Volume mount:**
```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - name: config
          mountPath: /etc/config
          readOnly: true
  volumes:
    - name: config
      secret:
        secretName: db-secret
```

**Безопасность Secrets:**
- Base64 ≠ encryption (это encoding);
- **etcd encryption at rest** — `EncryptionConfiguration` в API Server;
- **RBAC** — ограничить доступ к Secret (ServiceAccount + RoleBinding);
- **External secrets** — HashiCorp Vault, AWS Secrets Manager, GSM → External Secrets Operator;
- **Sealed Secrets** — шифрование для gitops (Bitnami Sealed Secrets).

</details>

---

### Вопрос №27: [ Что такое HPA, VPA, Cluster Autoscaler? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

| Mechanism | What scales | Based on |
|---|---|---|
| **HPA** (Horizontal Pod Autoscaler) | Количество pods | CPU, memory, custom metrics |
| **VPA** (Vertical Pod Autoscaler) | Requests/limits pod'а | Historical usage |
| **Cluster Autoscaler** | Количество nodes | Pending pods (unschedulable) |

**HPA:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```
- CPU utilization = actual usage / request;
- `cooldown` / `stabilizationWindowSeconds` — предотвращает flapping;
- Custom metrics — Prometheus Adapter, KEDA (Kubernetes Event-Driven Autoscaling).

**VPA:**
- Рекомендует requests на основе истории;
- Modes: `Off` (recommend only), `Initial` (только при создании), `Auto` (пересоздаёт pod);
- **Несовместим с HPA** на CPU/memory (используй HPA с custom metrics + VPA in Off mode).

**Cluster Autoscaler:**
- Watch: pods in `Pending` state (scheduler не нашёл node с достаточными resources);
- Action: добавляет node в cloud provider;
- Scale down: если node underutilized → drain → terminate;
- Медленный (минуты) → для burst лучше KEDA + overprovisioning.

**KEDA (Kubernetes Event-Driven Autoscaling):**
- Scale based on: Kafka lag, SQS queue depth, Redis stream length, Prometheus query;
- Scale to zero (0 → N pods);
- Работает как HPA wrapper (внутри использует HPA + custom metrics).

</details>

---

### Вопрос №28: [ Что такое Taints, Tolerations, Affinity/Anti-Affinity? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**Taints & Tolerations** — управление на какие nodes могут быть размещены pods.

**Taint** на node (запрет):
```bash
kubectl taint nodes node1 dedicated=gpu:NoSchedule
# NoSchedule — не размещать новые pods
# NoExecute — выселить существующие pods
# PreferNoSchedule — стараться не размещать
```

**Toleration** в pod (исключение):
```yaml
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
```

**Node Affinity** — предпочтение по labels (вместо nodeSelector):
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values: [ssd]
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: zone
                operator: In
                values: [us-east-1a]
```

**Pod Anti-Affinity** — распределить replicas по разным nodes (HA):
```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: postgres
          topologyKey: kubernetes.io/hostname
      # → каждый postgres pod на отдельной node
```

**Topology Spread Constraints** (современная альтернатива anti-affinity):
```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: api
    # → равномерно распределить pods по zones
```

</details>

---

### Вопрос №29: [ Как работает K8s rolling update? Rollback ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Rolling Update (default for Deployments):**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # сколько новых pods сверх replicas
    maxUnavailable: 0    # сколько старых pods можно убрать одновременно
```

- `maxSurge: 1, maxUnavailable: 0` → создаёт 1 новый → ждёт Ready → удаляет 1 старый → ... (zero downtime);
- `maxSurge: 0, maxUnavailable: 1` → удаляет 1 старый → создаёт 1 новый → ... (быстрее, но capacity снижена);

**Readiness Probe** — критична для rolling update:
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```
- Новый pod считается Ready только когда readinessProbe проходит;
- Без readiness probe → traffic идёт в ещё не готовый pod → 502;

**Rollback:**
```bash
# История
kubectl rollout history deployment/api

# Откат к предыдущей
kubectl rollout undo deployment/api

# Откат к конкретной
kubectl rollout undo deployment/api --to-revision=3

# Статус
kubectl rollout status deployment/api
```

**Graceful shutdown (критично для zero-downtime):**
```yaml
spec:
  terminationGracePeriodSeconds: 30
  containers:
    - name: app
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]
```
1. K8s отправляет `SIGTERM` → app должен завершить обработку текущих запросов;
2. `preStop` hook → sleep 5s (даёт time для deregister из service endpoints);
3. `terminationGracePeriodSeconds` → если не завершился → `SIGKILL`.

</details>

---

### Вопрос №30: [ Что такое Helm? Charts ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Helm** — package manager для Kubernetes. Chart = набор templated YAML + values.

**Структура chart:**
```
my-chart/
├── Chart.yaml          # metadata (name, version)
├── values.yaml         # default values
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl    # template helpers
└── charts/             # dependencies
```

**Пример template:**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

**values.yaml:**
```yaml
replicaCount: 3
image:
  repository: myapp
  tag: v1.2.0
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

**Использование:**
```bash
helm install my-release my-chart/ -f custom-values.yaml
helm upgrade my-release my-chart/ -f custom-values.yaml
helm rollback my-release 1
helm list
helm uninstall my-release
```

**Helm vs Kustomize:**
- **Helm** — templating (Go templates), chart dependencies, releases;
- **Kustomize** — overlay-based (base + patches), no templates, built-in `kubectl`;
- Выбор: Helm — для distributing charts; Kustomize — для environment-specific overlays.

</details>

---

</details>

---

## 7. Практические задачи (Senior)

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Задача №1: [ Спроектировать cache layer с Redis для high-load API ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Решение</summary>

**Сценарий:** API с 50K RPS, PostgreSQL не справляется с read-нагрузкой.

**Архитектура:**
```
Client → API → Redis (cache-aside) → PostgreSQL (source of truth)
                ↑
            Redis Cluster (3 master + 3 replica)
```

**Ключевые решения:**

**1. Шардинг — Redis Cluster:**
- 3 master + 3 replica (один replica на master для failover);
- Hash tags для multi-key: `{user:42}:profile`, `{user:42}:orders`;
- `maxmemory 8gb` на узел → 24gb total;

**2. Eviction policy:**
- `allkeys-lru` (cache только, не persistent store);
- `maxmemory-samples 10` (точнее LRU);

**3. TTL strategy:**
```go
// Базовый TTL + jitter
ttl := 300 + rand.Intn(60)  // 300-360 seconds
cache.Set(ctx, key, value, time.Duration(ttl)*time.Second)
```

**4. Singleflight (защита от stampede):**
```go
import "golang.org/x/sync/singleflight"

var g singleflight.Group

func GetUser(ctx context.Context, id int) (*User, error) {
    // Cache first
    if u, err := cache.Get(ctx, fmt.Sprintf("user:%d", id)); err == nil {
        return u, nil
    }
    // Singleflight: только один запрос в DB
    v, err, _ := g.Do(fmt.Sprintf("user:%d", id), func() (interface{}, error) {
        return db.GetUser(ctx, id)
    })
    if err != nil {
        return nil, err
    }
    u := v.(*User)
    cache.Set(ctx, fmt.Sprintf("user:%d", id), u, ttl)
    return u, nil
}
```

**5. Circuit breaker:**
```go
// Если Redis недоступен — fallback на DB (но с rate limiting)
breaker := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        return counts.ConsecutiveFailures > 5
    },
})

func cacheGet(key string) ([]byte, error) {
    return breaker.Execute(func() (interface{}, error) {
        return redis.Get(key)
    })
}
```

**Подводные камни:**

| Проблема | Решение |
|---|---|
| Hot key (1 ключ → 10K RPS на один шард) | Client-side L1 cache (in-process LRU, 1s TTL) |
| Big value (1MB JSON) блокирует event loop | Compress (gzip), разбить на части, или использовать RedisJSON |
| Cache inconsistency (DB updated, cache stale) | Cache-aside + DELETE on update + short TTL (safety net) |
| Redis failover → connection errors | Client with retry + connection pool к Sentinel/Cluster |
| Memory bloat | Monitor `used_memory` / `maxmemory`, alerting при > 80% |

</details>

---

### Задача №2: [ Zero-downtime deployment в Kubernetes: checklist ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Решение</summary>

**Deployment YAML:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0        # zero downtime: всегда есть replicas
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: api
          image: api:v2
          ports:
            - containerPort: 8080
          # 1. Readiness probe — новый pod не получает traffic пока не готов
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 3
            failureThreshold: 3
          # 2. Liveness probe — перезапуск при зависании
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          # 3. Resources — scheduler корректно размещает
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              memory: 512Mi
          # 4. Graceful shutdown
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]
          # 5. App handles SIGTERM
          # In Go: signal.Notify(sigChan, syscall.SIGTERM)
```

**App-level requirements:**

```go
// 6. Graceful shutdown в коде
func main() {
    srv := &http.Server{Addr: ":8080", Handler: handler}
    go srv.ListenAndServe()

    // Ждём SIGTERM
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGTERM, syscall.SIGINT)
    <-sigChan

    // Даём time для deregister из endpoints
    // preStop sleep 5s + K8s удаляет из endpoints ~5s
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    srv.Shutdown(ctx)  // ждём завершения in-flight requests
}
```

**Checklist:**
1. [x] `maxUnavailable: 0` — всегда есть capacity;
2. [x] Readiness probe — новый pod не получает traffic до готовности;
3. [x] `preStop` sleep — deregister из K8s endpoints до SIGTERM;
4. [x] App handles SIGTERM — graceful shutdown;
5. [x] `terminationGracePeriodSeconds` > max request timeout;
6. [x] DB migrations backward-compatible (expand-contract pattern);
7. [x] Pod anti-affinity — replicas на разных nodes;
8. [x] PDB (PodDisruptionBudget) — `minAvailable: 50%`.

</details>

---

### Задача №3: [ Настройка monitoring и alerting для K8s + Redis ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Решение</summary>

**Стек: Prometheus + Grafana + AlertManager**

**K8s metrics (kube-prometheus-stack):**
```yaml
# ServiceMonitor для app
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-monitor
spec:
  selector:
    matchLabels: {app: api}
  endpoints:
    - port: metrics
      interval: 15s
```

**Ключевые метрики K8s:**

| Метрика | Что показывает | Alert |
|---|---|---|
| `kube_pod_status_phase` | Pod phases | Pending > 5m |
| `container_cpu_cfs_throttled_seconds_total` | CPU throttle | rate > 0 sustained |
| `container_memory_working_set_bytes` | Memory usage | > 90% limit |
| `kube_deployment_status_replicas_available` | Available replicas | < desired |
| `kube_node_status_condition` | Node status | NotReady > 1m |
| `container_cpu_usage_seconds_total` / request | HPA utilization | > 80% |

**Redis metrics (redis_exporter):**

| Метрика | Что показывает | Alert |
|---|---|---|
| `redis_memory_used_bytes` / `maxmemory` | Memory usage | > 80% |
| `redis_connected_clients` | Connections | > maxclients × 0.8 |
| `redis_keyspace_hits_total` / `redis_keyspace_misses_total` | Hit rate | < 95% |
| `redis_replication_offset_diff` | Replication lag | > 1MB |
| `redis_slowlog_length` | Slow queries | increasing |
| `redis_rejected_connections_total` | Rejected (maxclients) | > 0 |

**Alert rules (Prometheus):**
```yaml
groups:
  - name: k8s-alerts
    rules:
      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[5m]) > 0
        for: 10m
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"

      - alert: HighMemoryUsage
        expr: |
          container_memory_working_set_bytes
          / container_spec_memory_limit_bytes > 0.9
        for: 5m
        annotations:
          summary: "Memory > 90% on {{ $labels.pod }}"

  - name: redis-alerts
    rules:
      - alert: RedisMemoryHigh
        expr: |
          redis_memory_used_bytes / redis_memory_max_bytes > 0.8
        for: 2m
        annotations:
          summary: "Redis memory > 80%"

      - alert: RedisReplicationLag
        expr: |
          (redis_connected_clients{role="master"}
           - redis_connected_clients{role="slave"}) > 0
        for: 5m
        annotations:
          summary: "Redis replication lag detected"

      - alert: RedisLowHitRate
        expr: |
          rate(redis_keyspace_misses_total[5m])
          / (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m])) > 0.05
        for: 10m
        annotations:
          summary: "Redis hit rate < 95%"
```

**Golden signals dashboard:**
- **Latency:** p50, p95, p99 histogram;
- **Traffic:** RPS per endpoint;
- **Errors:** 5xx rate, error budget burn;
- **Saturation:** CPU, memory, connection pool.

</details>

---

### Задача №4: [ Миграция монолита на микросервисы в K8s (strangler fig pattern) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Решение</summary>

**Strangler Fig Pattern** — постепенное "удушение" монолита: новые функции — микросервисы, трафик постепенно переводится.

**Фазы:**

**1. API Gateway перед монолитом:**
```
Client → [API Gateway / Ingress] → Monolith
```
- Все запросы идут через gateway;
- Монолит работает как раньше.

**2. Выделение первого сервиса (strangler):**
```
Client → [API Gateway]
            ├─ /users → User Service (new microservice, K8s)
            └─ /*     → Monolith (legacy)
```
- Gateway routing по path/header;
- Новый сервис в K8s, монолит — где был (VM или K8s);

**3. Постепенный перенос:**
- Каждый sprint — новый сервис;
- Данные: DB per service (выделение из shared DB);
- Sync: temporary dual-write или CDC;

**4. Полное замещение:**
```
Client → [API Gateway]
            ├─ /users    → User Service
            ├─ /orders   → Order Service
            ├─ /payments → Payment Service
            └─ /reports  → Report Service (последний → монолит удалён)
```

**K8s Ingress routing:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /users
            backend:
              service:
                name: user-service
                port: {number: 8080}
          - path: /orders
            backend:
              service:
                name: order-service
                port: {number: 8080}
          - path: /
            backend:
              service:
                name: monolith
                port: {number: 80}
```

**Подводные камни:**

| Проблема | Решение |
|---|---|
| **Shared database** (монолит и сервисы читают одну DB) | CDC (Debezium) для синхронизации; eventual consistency; gradual extraction |
| **Distributed transactions** | Saga pattern; идемпотентность; outbox pattern |
| **Authentication** | Gateway → JWT validation → forward to services |
| **Data consistency** | Eventual consistency + idempotent consumers; reconciliation jobs |
| **Backward compatibility** | API versioning (v1/v2); контрактное тестирование |
| **Monitoring** | Distributed tracing (Jaeger); per-service dashboards |
| **Canary** | Gateway weighted routing (10% → new service, 90% → monolith) |

</details>

---

### Задача №5: [ Настройка Redis Cluster в Kubernetes (StatefulSet) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Решение</summary>

**StatefulSet для Redis Cluster:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: redis-cluster
  replicas: 6              # 3 master + 3 replica
  selector:
    matchLabels: {app: redis-cluster}
  template:
    metadata:
      labels: {app: redis-cluster}
    spec:
      containers:
        - name: redis
          image: redis:7.2-alpine
          command:
            - redis-server
            - --cluster-enabled yes
            - --cluster-config-file nodes.conf
            - --cluster-node-timeout 5000
            - --appendonly yes
            - --maxmemory 4gb
            - --maxmemory-policy allkeys-lru
          ports:
            - containerPort: 6379
              name: client
            - containerPort: 16379
              name: gossip
          resources:
            requests: {cpu: 500m, memory: 5Gi}
            limits: {memory: 6Gi}
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests: {storage: 10Gi}
        storageClassName: fast-ssd
```

**Headless Service для DNS resolution:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
spec:
  clusterIP: None          # headless
  selector:
    app: redis-cluster
  ports:
    - port: 6379
      name: client
    - port: 16379
      name: gossip
```

**Pod anti-affinity (HA — разные nodes):**
```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels: {app: redis-cluster}
          topologyKey: kubernetes.io/hostname
```

**Инициализация cluster (один раз):**
```bash
kubectl exec redis-cluster-0 -- redis-cli --cluster create \
  redis-cluster-0.redis-cluster:6379 \
  redis-cluster-1.redis-cluster:6379 \
  redis-cluster-2.redis-cluster:6379 \
  redis-cluster-3.redis-cluster:6379 \
  redis-cluster-4.redis-cluster:6379 \
  redis-cluster-5.redis-cluster:6379 \
  --cluster-replicas 1 --cluster-yes
```

**Альтернатива: Redis Operator**
- `redis-operator` (OT Container Kit) или `Redis Enterprise Operator`;
- Автоматическое создание, scaling, failover, backup;
- Упрощает operational overhead.

**Подводные камни:**

| Проблема | Решение |
|---|---|
| **Persistent volume** при удалении StatefulSet | PVC не удаляется автоматически → data retention; но при `persistentVolumeReclaimPolicy: Delete` → потеря |
| **Cluster rebalancing** при scaling | `redis-cli --cluster rebalance` вручную или Operator |
| **Pod IP changes** (restart) | Redis Cluster хранит IP в nodes.conf → `--cluster-announce-ip` через initContainer (reads pod IP) |
| **Network partition** | Split-brain → Redis Cluster переводит minority partition в read-only |
| **Memory limit > maxmemory** | K8s OOMKill: set `maxmemory` на 80% от container memory limit |

</details>

---

### Задача №6: [ Troubleshooting: Pod в CrashLoopBackOff. План диагностики ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Решение</summary>

**CrashLoopBackOff** — pod перезапускается и снова падает.

**Step-by-step диагностика:**

```bash
# 1. Что произошло?
kubectl describe pod <pod-name>
# Смотри Events, Last State, Exit Code

# 2. Логи текущего контейнера
kubectl logs <pod-name>

# 3. Логи предыдущего (упавшего) контейнера
kubectl logs <pod-name> --previous

# 4. Логи всех контейнеров в pod
kubectl logs <pod-name> --all-containers

# 5. Events в namespace
kubectl get events --sort-by=.lastTimestamp | tail -20
```

**Частые причины:**

| Exit Code | Причина | Решение |
|---|---|---|
| **0** | Процесс завершился нормально (Job done) | Проверь `restartPolicy` |
| **1** | Application error | Смотри логи `--previous` |
| **125** | Docker failed to start container | Проверь image, entrypoint |
| **126** | Permission denied (command not executable) | `chmod +x` или проверь USER |
| **127** | Command not found | Проверь PATH, entrypoint |
| **137** | OOMKilled (memory limit exceeded) | Увеличь memory limit / проверь leak |
| **139** | Segmentation fault | Баг в code (CGO, unsafe) |
| **143** | SIGTERM (graceful shutdown) | Нормально при rolling update |

**Проверить OOMKill:**
```bash
kubectl describe pod <pod-name> | grep -A5 "Last State"
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137

# Memory usage:
kubectl top pod <pod-name>
```

**Проверить resources:**
```bash
kubectl describe node <node-name> | grep -A10 "Allocated"
# Возможно, node перегружена
```

**Debug pod (если логи недоступны):**
```yaml
# Запустить тот же image с другой command
spec:
  containers:
    - name: debug
      image: myapp:v2
      command: ["sh", "-c", "sleep 3600"]  # не падает
```
```bash
kubectl exec -it <debug-pod> -- sh
# Ручная диагностика: проверить config, env, connectivity
```

**Ephemeral debug container (K8s 1.25+):**
```bash
kubectl debug <pod-name> -it --image=busybox --target=<container-name>
```

</details>

---

</details>

---

## Шпаргалка: полезные команды

<details>
<summary><b>Развернуть</b></summary>

### Redis

```bash
# Подключение
redis-cli -h host -p 6379 -a password

# Информация
INFO                    # общая информация
INFO memory             # память
INFO replication        # репликация
INFO stats              # статистика

# Мониторинг
MONITOR                 # все команды в real-time (только debug!)
SLOWLOG GET 10          # медленные запросы
CLIENT LIST             # активные соединения
MEMORY USAGE key        # размер ключа в байтах
MEMORY STATS            # статистика памяти

# Управление
CONFIG GET maxmemory    # текущее значение
CONFIG SET maxmemory 4gb
FLUSHDB                 # очистить текущую DB (ОСТОРОЖНО!)
FLUSHALL                # очистить все DB (ОСТОРОЖНО!)
BGSAVE                  # снапшот (background)
BGREWRITEAOF            # rewrite AOF

# Cluster
CLUSTER INFO            # состояние cluster
CLUSTER NODES           # все узлы
CLUSTER COUNTKEYSINSLOT 1234  # ключей в слоте
CLUSTER KEYSLOT mykey   # слот для ключа

# Анализ ключей
SCAN 0 MATCH user:* COUNT 100   # безопасный scan (не KEYS!)
DBSIZE                  # количество ключей
TYPE key                # тип ключа
OBJECT ENCODING key     # внутренняя структура (ziplist, hashtable, etc.)
```

### Docker

```bash
# Управление
docker build -t myapp:v1 .
docker run -d --name app -p 8080:8080 myapp:v1
docker stop app && docker rm app

# Инспекция
docker ps -a                    # все контейнеры
docker images                   # образы
docker logs -f app              # логи
docker exec -it app sh          # shell в контейнере
docker inspect app              # полная информация
docker stats                    # ресурсы в real-time

# Очистка
docker system prune -a          # удалить всё неиспользуемое
docker image prune              # удалить dangling images
docker volume prune             # удалить unused volumes

# Слои image
docker history myapp:v1         # слои и их размер
docker history --no-trunc myapp:v1  # подробные команды
```

### Kubernetes

```bash
# Базовые
kubectl get pods -o wide
kubectl get pods -A                     # все namespaces
kubectl get svc,deploy,ingress
kubectl describe pod <pod>

# Логи
kubectl logs <pod>
kubectl logs <pod> --previous           # упавший контейнер
kubectl logs -f <pod> -c <container>     # follow + конкретный контейнер

# Exec
kubectl exec -it <pod> -- sh
kubectl debug <pod> -it --image=busybox  # ephemeral debug container

# Scale / rollout
kubectl scale deploy api --replicas=5
kubectl rollout status deploy/api
kubectl rollout undo deploy/api
kubectl rollout history deploy/api

# Resources
kubectl top pod                          # CPU/memory (needs metrics-server)
kubectl top node
kubectl get resourcequotas
kubectl describe node <node> | grep -A15 "Allocated"

# Labels / selectors
kubectl get pods -l app=api,tier=backend
kubectl label pod <pod> env=prod
kubectl get pods --field-selector status.phase=Running

# Port forward (debug)
kubectl port-forward svc/api 8080:8080

# ConfigMap / Secret
kubectl create configmap cfg --from-file=config.yaml
kubectl create secret tls tls-secret --cert=cert.pem --key=key.pem
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d

# Drain / cordon
kubectl cordon <node>                    # запретить scheduling
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data  # выселить pods
kubectl uncordon <node>                  # вернуть в строй

# Cleanup
kubectl delete pod <pod> --force --grace-period=0  # принудительно
kubectl delete pods --field-selector status.phase=Failed  # удалить failed pods
```

</details>

---

## 8. Сравнение in-memory хранилищ: Redis vs Aerospike vs Memcached

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №1: [ Redis vs Memcached — принципиальная разница ]

    <details>
      <summary>Ответ</summary>

    | Критерий | Redis | Memcached |
    |----------|-------|-----------|
    | **Тип данных** | String, List, Hash, Set, Sorted Set, Stream, HyperLogLog, Geo | Только String (key-value) |
    | **Persistence** | RDB (snapshot) + AOF (write log) | Нет (только RAM) |
    | **Replication** | Master-replica, Sentinel, Cluster | Нет (каждый node независим) |
    | **Threading** | Single-threaded (Redis 6+ I/O threads) | Multi-threaded |
    | **Max object size** | 512 МБ | 1 МБ (по умолчанию) |
    | **Eviction** | LRU, LFU, TTL, noeviction | LRU, TTL |
    | **Pub/Sub** | Да | Нет |
    | **Lua scripting** | Да | Нет |
    | **Cluster** | Redis Cluster (hash slots) | Client-side hashing (consistent hashing) |
    | **Memory efficiency** | Выше (ziplist, intset) | Ниже для тех же данных (но меньше overhead) |

    * **Memcached:** простейший key-value cache, multi-threaded, очень быстрый для простых
      GET/SET. Нет persistence — чистый cache.
    * **Redis:** data-structure server — списки, множества, сортированные множества, streams.
      Persistence, replication, pub/sub, Lua.

    </details>

  ---

  - Вопрос №2: [ Когда выбрать Redis, а когда Memcached? ]

    <details>
      <summary>Ответ</summary>

    * **Выбирайте Redis, если:**
      - Нужны структуры данных (list, set, sorted set, hash) — rate limiter, leaderboard,
        queue (LPUSH/RPOP).
      - Нужна persistence (cache warm restart, session store).
      - Нужна репликация (HA, read scaling).
      - Нужен pub/sub или streams.
      - Нужен Lua scripting (atomic complex operations).
      - TTL с миллисекундной точностью.

    * **Выбирайте Memcached, если:**
      - Простой key-value cache (HTML fragments, SQL results).
      - Multi-threaded важно (Redis single-threaded — 1 core, Memcached — все cores).
      - Много маленьких объектов — Memcached эффективнее по памяти (меньше metadata overhead).
      - Cache-only — потеря данных при рестарте допустима.

    * **На практике:** Redis стал де-факто стандартом — Memcached выбирают только для
      простейшего caching слоя с максимальным multi-core использованием.

    </details>

  ---

  - Вопрос №3: [ Redis vs Aerospike — когда in-memory не хватает ]

    <details>
      <summary>Ответ</summary>

    | Критерий | Redis | Aerospike |
    |----------|-------|-----------|
    | **Архитектура** | In-memory (опционально disk persistence) | Hybrid: RAM + SSD/NVMe (primary storage) |
    | **Объём данных** | RAM-limited (десятки ГБ на node) | TB на node (данные на SSD) |
    | **Latency** | <1 мс (RAM) | ~1–5 мс (SSD), <1 мс (RAM namespace) |
    | **Throughput** | ~100K–200K ops/сек (single-threaded) | ~200K–1M ops/сек (multi-threaded) |
    | **Threading** | Single-threaded (Redis 6+ I/O threads) | Multi-threaded |
    | **Replication** | Async (по умолчанию), sync (WAIT) | Sync / async (configurable per namespace) |
    | **Sharding** | Redis Cluster (16384 hash slots) | Auto-clustering (consistent hashing + paxos) |
    | **Data model** | Key-value, data structures | Key-value (bins), secondary indexes, aggregation |
    | **Indexes** | Нет (только key lookup) | Primary + secondary indexes (sindex) |
    | **Strong consistency** | Нет (eventual consistency) | Да (strongly consistent mode, CP) |
    | **Cross-DC** | Redis CRDTs (Active-Active) | XDR (Cross-Datacenter Replication) |
    | **License** | RSALv2 / SSPL (Valkey — BSD) | AGPL / Enterprise |

    * **Aerospike** создан для **больших объёмов данных на SSD** — данные хранятся на диске,
      RAM используется только для primary index. Это позволяет хранить TB на один node при
      sub-millisecond latency.
    * **Redis** — in-memory: все данные в RAM. Для больших объёмов нужен cluster с множеством
      nodes, что дорого (RAM дороже SSD).

    </details>

  ---

  - Вопрос №4: [ Когда выбрать Aerospike вместо Redis? ]

    <details>
      <summary>Ответ</summary>

    * **Выбирайте Aerospike, если:**
      - **Объём данных > RAM** — terabytes данных, которые не влезают в RAM.
      - **Cost efficiency** — SSD дешевле RAM (Aerospike на 1 TB = 1 сервер, Redis = много
        серверов с RAM).
      - **High write throughput** — multi-threaded, линейное масштабирование.
      - **Strong consistency** — Aerospike поддерживает CP mode (linearizable reads/writes).
      - **Secondary indexes** — запросы по не-key полям (Redis — только scan или app-level index).
      - **Cross-DC replication** — XDR для active-active multi-region.
      - **Session store для 100M+ users** — большой объём, low latency, HA.

    * **Выбирайте Redis, если:**
      - **Data structures** — sorted sets (leaderboard), lists (queue), streams (event log),
        hyperloglog (cardinality).
      - **Sub-millisecond latency** — всё в RAM.
      - **Ecosystem** — огромная экосистема, библиотеки на всех языках, Redis Stack (Search,
        JSON, TimeSeries, Bloom).
      - **Small-medium dataset** — до десятков ГБ на node.
      - **Pub/Sub** — real-time notifications.
      - **Lua scripting** — atomic complex operations.

    </details>

  ---

  - Вопрос №5: [ Redis vs Aerospike: стоимость владения (TCO) ]

    <details>
      <summary>Ответ</summary>

    * **Сценарий:** 1 TB hot data, ~500K ops/сек, 99-й перцентиль < 5 мс.

    | | Redis Cluster | Aerospike |
    |--|---------------|-----------|
    | **Nodes** | ~25–30 nodes (по 40–48 ГБ RAM каждая) | ~5–7 nodes (по 2 TB SSD + 64 ГБ RAM) |
    | **Hardware cost** | ~25 × high-RAM servers (дорого) | ~5 × standard servers (дешевле) |
    | **Network** | 25 nodes → больше cross-node traffic (resharding) | 5 nodes → меньше overhead |
    | **Operations** | 25 nodes → больше patching, monitoring | 5 nodes → проще управлять |
    | **License** | RSALv2 / SSPL (Valkey — BSD) | Open source (AGPL) / Enterprise (paid) |

    * **Вывод:** для больших объёмов данных (>100 ГБ) Aerospike значительно дешевле по hardware.
      Для малых объёмов (<50 ГБ) Redis проще и дешевле (free license, нет enterprise license).

    </details>

  ---

  - Вопрос №6: [ Практический сценарий: выбор хранилища для разных задач ]

    <details>
      <summary>Ответ</summary>

    | Задача | Хранилище | Почему |
    |--------|-----------|-------|
    | Session store (10M users, <10 ГБ) | Redis | In-memory, low latency, TTL, simple |
    | Session store (200M users, 500 ГБ) | Aerospike | TB на SSD, low latency, cheaper than RAM |
    | Leaderboard / real-time ranking | Redis | Sorted Set (ZADD/ZRANGE) — built-in |
    | Rate limiter | Redis | INCR + EXPIRE, sliding window via Lua |
    | Cache layer (SQL results) | Redis / Memcached | Simple key-value, TTL |
    | User profile store (1 TB, indexed) | Aerospike | Secondary indexes, SSD-backed |
    | Real-time bidding (RTB) | Aerospike | <1 мс, high write throughput, strong consistency |
    | Pub/Sub notifications | Redis | Built-in pub/sub + streams |
    | Job queue (delayed tasks) | Redis | List + ZSET (delayed queue) |
    | Fraud detection (large graph) | Aerospike | Large dataset, sindex, UDF (Lua) |
    | Geo-fencing / geospatial | Redis | Geo commands (GEOADD, GEORADIUS) |
    | Time-series (metrics, IoT) | Redis (TimeSeries module) / Aerospike | Redis TS — purpose-built; Aerospike — large volume |

    </details>

  ---

</details>

---
