# Redis — Вопросы и ответы для Middle/Senior

Файл содержит вопросы по внутреннему устройству Redis, репликации, кластеру,
паттернам использования, а также практические задачи и сравнение с другими in-memory хранилищами.

Уровни сложности:
- ![Static Badge](https://img.shields.io/badge/Middle-yellow) — Middle разработчик
- ![Static Badge](https://img.shields.io/badge/Senior-red) — Senior разработчик

---

# Оглавление

1. [Redis — внутреннее устройство](#1-redis--внутреннее-устройство)
2. [Redis — репликация и кластер](#2-redis--репликация-и-кластер)
3. [Redis — паттерны использования](#3-redis--паттерны-использования)
4. [Практические задачи (Senior)](#4-практические-задачи-senior)
5. [Сравнение in-memory хранилищ: Redis vs Aerospike vs Memcached](#5-сравнение-in-memory-хранилищ-redis-vs-aerospike-vs-memcached)

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
- Вероятностный счётчик (8 бит): логарифмическая вероятность, затухание со временем;
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
- **Fire-and-forget:** если подписчик не в сети — сообщение потеряно (нет persistence);
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

## 4. Практические задачи (Senior)

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
### Задача №2: [ Настройка Redis Cluster в Kubernetes (StatefulSet) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

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

</details>

---

## 5. Сравнение in-memory хранилищ: Redis vs Aerospike vs Memcached

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №15: [ Redis vs Memcached — принципиальная разница ]

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

### Вопрос №16: [ Когда выбрать Redis, а когда Memcached? ]

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

### Вопрос №17: [ Redis vs Aerospike — когда in-memory не хватает ]

<details>
<summary>Ответ</summary>

| Критерий | Redis | Aerospike |
|----------|-------|-----------|
| **Архитектура** | In-memory (опционально disk persistence) | Hybrid: RAM + SSD/NVMe (primary storage) |
| **Объём данных** | RAM-limited (десятки ГБ на node) | TB на node (данные на SSD) |
| **Latency** | <1 мс (RAM) | ~1–5 мс (SSD), <1 мс (RAM namespace) |
| **Throughput** | ~100K ops/сек (single-thread), ~500K+ с I/O threads | ~200K–1M ops/сек (multi-threaded) |
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

### Вопрос №18: [ Когда выбрать Aerospike вместо Redis? ]

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

### Вопрос №19: [ Redis vs Aerospike: стоимость владения (TCO) ]

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

### Вопрос №20: [ Практический сценарий: выбор хранилища для разных задач ]

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
