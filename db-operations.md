# Базы данных: эксплуатация и оптимизация — Вопросы и ответы

Файл содержит вопросы по эксплуатации БД: индексы, оптимизация запросов, транзакции,
репликация, бэкапы, миграции, мониторинг. Дополняет `sql-postgres-questions.md` темами
по операционной практике работы с БД.

Уровни сложности:
- ![Static Badge](https://img.shields.io/badge/Middle-yellow) — Middle разработчик
- ![Static Badge](https://img.shields.io/badge/Senior-red) — Senior разработчик

---

# Оглавление

1. [Индексы: глубокое погружение](#1-индексы-глубокое-погружение)
2. [Оптимизация запросов](#2-оптимизация-запросов)
3. [Транзакции и изоляция](#3-транзакции-и-изоляция)
4. [Репликация и High Availability](#4-репликация-и-high-availability)
5. [Бэкапы и восстановление](#5-бэкапы-и-восстановление)
6. [Миграции схемы](#6-миграции-схемы)
7. [Мониторинг и деградация](#7-мониторинг-и-деградация)
8. [Шардинг и масштабирование](#8-шардинг-и-масштабирование)
9. [Сравнение БД: SQL vs NoSQL vs columnar vs search](#9-сравнение-бд-sql-vs-nosql-vs-columnar-vs-search)

---

## 1. Индексы: глубокое погружение

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Какие типы индексов поддерживает PostgreSQL? ]

<details>
<summary>Ответ</summary>

* **B-tree** — по умолчанию. Для =, <, >, BETWEEN, IS NULL, ORDER BY, prefix LIKE.
* **Hash** — только = (равенство). Не поддерживает range. Быстрее B-tree для точного match, но не используется
  для ORDER BY. До PostgreSQL 10 не был WAL-logged → не реплицировался.
* **GIN** (Generalized Inverted Index) — для составных значений: массивы, full-text search (tsvector),
  JSONB. "Какое значение в каких строках".
* **GiST** (Generalized Search Tree) — для геометрических данных (PostGIS), range types, full-text search
  (с триграммами). Поддерживает `<<`, `@>`, `<@` и др.
* **SP-GiST** (Space-Partitioned GiST) — для неравномерных данных (телефонные номера, маршрутизация).
* **BRIN** (Block Range Index) — для огромных таблиц с естественным упорядочиванием (time-series).
  Хранит min/max для блоков (128 страниц). Крошечный размер, но неточный.
* **Partial index** — индекс по подмножеству строк: `WHERE active = true`.
* **Expression index** — индекс по выражению: `CREATE INDEX ON users (lower(email))`.
* **Covering index** — `INCLUDE` columns: `CREATE INDEX ON orders (user_id) INCLUDE (total, status)`.

</details>

---

### Вопрос №2: [ Когда индекс не используется (index scan → seq scan)? ]

<details>
<summary>Ответ</summary>

* **Низкая селективность** — если предикат возвращает > ~5–10% строк, планировщик выбирает seq scan
  (прочитать всю таблицу дешевле, чем index + heap fetch).
* **Функция на колонке** — `WHERE lower(email) = '...'` не использует обычный индекс на email.
  Нужен expression index: `CREATE INDEX ON users (lower(email))`.
* **Неявное приведение типов** — `WHERE varchar_col = 123` (int vs varchar) может не использовать индекс.
* **LIKE с wildcards в начале** — `WHERE name LIKE '%ivan'` не использует B-tree. GiST с pg_trgm — может.
* **OR с разными колонками** — `WHERE a = 1 OR b = 2` может не использовать индексы (но bitmap OR scan может).
* **Устаревшая статистика** — `ANALYZE` не запускался давно. `VACUUM ANALYZE` обновит.
* **Несоответствие типа колонки и выражения** — `WHERE date_col = '2024-01-01'::text` (явное приведение к другому типу) может не использовать индекс. В PostgreSQL unknown-литералы автоматически приводятся к типу колонки, поэтому `WHERE bigint_col = '123'` индекс использовать будет.
* Проверить: `EXPLAIN ANALYZE` — посмотреть, используется ли index scan / bitmap scan / seq scan.

</details>

---

### Вопрос №3: [ Что такое covering index и INCLUDE? ]

<details>
<summary>Ответ</summary>

* **Covering index** — индекс, содержащий все колонки, нужные запросу, чтобы не ходить в heap (таблицу).
* **Index-only scan** — если все нужные колонки в индексе, PostgreSQL читает только индекс (быстрее).
* `INCLUDE` (PostgreSQL 11+):
```sql
CREATE INDEX idx_orders_user ON orders (user_id) INCLUDE (total, status);
-- Запрос SELECT total, status FROM orders WHERE user_id = 5 → index-only scan
```
* Включённые (`INCLUDE`) колонки не участвуют в сортировке/search, но хранятся в leaf-нодах.
* До PostgreSQL 11 — covering index через composite index: `CREATE INDEX ON orders (user_id, total, status)`
  (но тогда total и status участвуют в B-tree order, что может быть не нужно).

</details>

---

### Вопрос №4: [ Что такое index bloat и как с ним бороться? ]

<details>
<summary>Ответ</summary>

* **Index bloat** — индекс фрагментируется из-за UPDATE/DELETE. Dead tuples занимают место в индексе.
* Симптомы: индекс намного больше, чем должен быть, медленные index scans.
* **Проверка**:
```sql
SELECT schemaname, tablename, indexname, pg_size_pretty(pg_relation_size(indexname::regclass))
FROM pg_indexes;
```
* Использовать расширение `pgstattuple` для детального анализа.
* **Решение**:
  - `REINDEX INDEX index_name;` — пересоздаёт индекс (блокирует запись в старых версиях).
  - `REINDEX INDEX CONCURRENTLY index_name;` (PostgreSQL 12+) — без блокировки (дольше, но безопасно).
  - `pg_repack` / `pg_squeeze` — перестроение без долгих блокировок.
* **Автоматизация**: `autovacuum` очищает dead tuples, но не дефрагментирует индекс полностью.
  Периодический `REINDEX` для сильно фрагментированных.

</details>

---

### Вопрос №5: [ Как работают partial и expression индексы? ]

<details>
<summary>Ответ</summary>

* **Partial index** — индекс только по строкам, удовлетворяющим условие:
```sql
CREATE INDEX idx_active_users ON users (email) WHERE active = true;
-- Используется только для запросов, включающим active = true
SELECT * FROM users WHERE active = true AND email = 'a@b.com';  -- ✅
SELECT * FROM users WHERE email = 'a@b.com';                     -- ❌ не использует
```
* Плюсы: меньше размер, быстрее scan, меньше overhead на INSERT/UPDATE.
* **Expression index** — индекс по результату функции:
```sql
CREATE INDEX idx_users_email_lower ON users (lower(email));
SELECT * FROM users WHERE lower(email) = 'alice@example.com';  -- ✅ использует
```
* Пример: JSONB path — `CREATE INDEX ON events ((data->>'type'))`.

</details>

---

</details>

---

## 2. Оптимизация запросов

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Как читать EXPLAIN ANALYZE в PostgreSQL? ]

<details>
<summary>Ответ</summary>

* `EXPLAIN` — план выполнения (без выполнения). `EXPLAIN ANALYZE` — выполняет и показывает реальные метрики.
* `EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)` — добавляет информацию о буферах (hit/read/dirtied/written).
* **Ключевые узлы плана**:
  - **Seq Scan** — полное сканирование таблицы. Часто плохо для больших таблиц.
  - **Index Scan** — чтение из индекса + heap fetch.
  - **Index Only Scan** — только индекс (если covering). Лучший вариант.
  - **Bitmap Index Scan → Bitmap Heap Scan** — сначала bitmap из индекса, потом heap fetch партиями.
  - **Nested Loop** — для join'а: для каждой строки outer table — scan inner. Хорошо для малых таблиц
    или indexed inner.
  - **Hash Join** — хэш-таблица меньшей стороны, probe большей. Хорошо для больших таблиц без индекса.
  - **Merge Join** — обе стороны отсортированы, merge. Хорошо для отсортированных данных (index).
* **Метрики**:
  - `cost=0.00..341.00` — estimated cost (startup..total). Не время, а условные единицы.
  - `rows=1000` — estimated (EXPLAIN) / actual (ANALYZE) rows.
  - `actual time=0.1..5.3` — реальное время (мс) на startup..total.
  - `loops=5` — узел выполнен 5 раз. time/rows — per loop.
  - `Buffers: shared hit=100 read=20` — pages из cache / с диска.
* **На что смотреть**:
  - Большая разница estimated vs actual rows → устаревшая статистика → `ANALYZE`.
  - Seq Scan на большой таблице → нужен индекс.
  - Nested Loop с большими rows → возможно лучше Hash Join.
  - `read=` много → данные не в cache.

</details>

---

### Вопрос №2: [ Что такое N+1 проблема и как её решить? ]

<details>
<summary>Ответ</summary>

* **N+1** — вместо одного запроса с JOIN, делается 1 запрос на список + N запросов на каждую связанную сущность.
* Пример (ORM):
```go
// N+1: 1 запрос на users + N запросов на orders для каждого user
users := db.Find(&users)
for _, u := range users {
    db.Model(&u).Related(&u.Orders)  // N запросов!
}
```
* **Решение**:
  - **JOIN** (eager loading): `SELECT u.*, o.* FROM users u LEFT JOIN orders o ON o.user_id = u.id`.
  - **Batch loading**: `SELECT * FROM orders WHERE user_id IN (1, 2, 3, ...)` — один запрос для всех.
  - ORM: GORM `Preload("Orders")`, Django `select_related` (JOIN) / `prefetch_related` (batch).
* **Обнаружение**: query logging, APM (DataDog, NewRelic) показывает много одинаковых запросов.

</details>

---

### Вопрос №3: [ Как оптимизировать медленный запрос? ]

<details>
<summary>Ответ</summary>

* **Шаг 1**: `EXPLAIN (ANALYZE, BUFFERS)` — понять план.
* **Шаг 2**: Найти самый дорогой узел (по `actual time` или `rows`).
* **Шаг 3**: Проверить:
  - **Seq Scan на большой таблице** → добавить индекс на filter/join колонки.
  - **Большая разница estimated vs actual rows** → `ANALYZE table` (обновить статистику).
  - **Filesort / Sort** → индекс на ORDER BY колонки.
  - **Temporary tables** → слишком много данных в памяти → индекс или переписать запрос.
  - **Nested Loop с большим rows** → может, Hash Join лучше (изменить селективность, статистику).
* **Шаг 4**: Переписать запрос:
  - Убрать `SELECT *`, оставить только нужные колонки (covering index).
  - Заменить `IN (subquery)` на `EXISTS` или JOIN.
  - Разбить сложный запрос на несколько (CTE / temp tables).
  - Убрать функции на колонках в WHERE (нельзя использовать индекс).
* **Шаг 5**: Проверить лимиты — `LIMIT` без `ORDER BY` может дать нестабильные результаты.
* **Шаг 6**: Партиционирование для огромных таблиц.

</details>

---

### Вопрос №4: [ Что такое prepared statements и когда использовать? ]

<details>
<summary>Ответ</summary>

* **Prepared statement** — запрос с параметрами, который парсится и планируется один раз, выполняется много раз.
* `PREPARE get_user AS SELECT * FROM users WHERE id = $1;` → `EXECUTE get_user(42);`
* **Плюсы**:
  - Парсинг/планирование один раз → быстрее при повторных вызовах.
  - Защита от SQL injection (параметры не интерполируются в SQL).
* **В PostgreSQL**:
  - **Custom plan** — план с конкретными значениями параметров (можно лучше оценить).
  - **Generic plan** — план без значений (универсальный).
  - После 5 выполнений PostgreSQL решает: использовать generic (если stable) или custom.
* **Минусы**:
  - `SET` / `RESET` — prepared statements не видят изменений в search_path и других SET.
  - Session-level: живёт до DISCARD или закрытия сессии. PgBouncer в transaction mode — проблемы с prepared
    statements (нужен `max_prepared_statements` в PgBouncer).
* В Go: `database/sql` с `db.QueryContext(ctx, "SELECT ... WHERE id = $1", id)` использует prepared statements
  (зависит от драйвера).

</details>

---

### Вопрос №5: [ Чем отличается CTE от подзапроса? ]

<details>
<summary>Ответ</summary>

* **CTE** (Common Table Expression, `WITH`):
```sql
WITH active_users AS (
    SELECT * FROM users WHERE active = true
)
SELECT * FROM active_users WHERE created_at > '2025-01-01';
```
* **Подзапрос**:
```sql
SELECT * FROM (SELECT * FROM users WHERE active = true) au
WHERE created_at > '2025-01-01';
```
* В PostgreSQL ≤ 11: CTE — **optimization fence** (материализуется, выполняется один раз, планировщик
  не может "протолкнуть" условия внутрь). С 12+ — по умолчанию inline (если не `MATERIALIZED`).
* `WITH ... AS MATERIALIZED (...)` — принудительная материализация (выполнить один раз, сохранить).
  Полезно, если CE дорогой и используется несколько раз.
* `WITH ... AS NOT MATERIALIZED (...)` — inline (протолкнуть условия).
* **Recursive CTE** — для иерархических запросов (деревья, графы):
```sql
WITH RECURSIVE descendants AS (
    SELECT id, parent_id, name FROM categories WHERE id = 1
    UNION ALL
    SELECT c.id, c.parent_id, c.name FROM categories c
    JOIN descendants d ON c.parent_id = d.id
)
SELECT * FROM descendants;
```

</details>

---

</details>

---

## 3. Транзакции и изоляция

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Какие уровни изоляции транзакций поддерживает PostgreSQL? ]

<details>
<summary>Ответ</summary>

| Уровень | Dirty Read | Non-Repeatable Read | Phantom Read | Serialization Anomaly |
|---------|-----------|---------------------|--------------|----------------------|
| Read Uncommitted | Не возможно* | Возможно | Возможно | Возможно |
| Read Committed (default) | Не возможно | Возможно | Возможно | Возможно |
| Repeatable Read | Не возможно | Не возможно | Не возможно* | Возможно |
| Serializable | Не возможно | Не возможно | Не возможно | Не возможно |

* *В PostgreSQL Read Uncommitted работает как Read Committed (dirty reads невозможны).
* *В PostgreSQL Repeatable Read предотвращает и phantom reads (использует snapshot isolation).
* **Read Committed** — каждое `SELECT` видит свежий snapshot. Внутри одной транзакции два SELECT могут
  дать разные результаты.
* **Repeatable Read** — snapshot на момент первого `SELECT` в транзакции. Все последующие `SELECT` видят
  те же данные.
* **Serializable** —True serializability через SSI (Serializable Snapshot Isolation). Если обнаружен конфликт —
  `ERROR: could not serialize access due to read/write dependencies`.

</details>

---

### Вопрос №2: [ Что такое MVCC и как оно работает в PostgreSQL? ]

<details>
<summary>Ответ</summary>

* **MVCC** (Multi-Version Concurrency Control) — каждый UPDATE создаёт **новую версию** строки, а не изменяет
  существующую. DELETE помечает строку как dead.
* Readers видят snapshot (версии строк на момент начала транзакции) — не блокируются writers.
* Writers не блокируют readers, writers блокируют writers (через row-level locks).
* **xmin** — ID транзакции, создавшей версию. **xmax** — ID транзакции, удалившей/изменившей версию.
* **cmin/cmax** — command ID внутри транзакции.
* **Dead tuples** — старые версии строк, не видные ни одной активной транзакции. Занимают место (bloat).
* **VACUUM** — удаляет dead tuples, освобождает место для повторного использования (не возвращает ОС,
  кроме `VACUUM FULL`).
* **autovacuum** — автоматический vacuum, работает в фоне. Настраивается через параметры
  `autovacuum_vacuum_threshold`, `autovacuum_vacuum_scale_factor`.
* **Transaction ID wraparound** — XID — 32-битный. При переполнении — необходимость vacuum, иначе данные
  становятся невидимы. `VACUUM FREEZE` устанавливает `xmin = FrozenXID`.

</details>

---

### Вопрос №3: [ Что такое deadlock и как его предотвратить? ]

<details>
<summary>Ответ</summary>

* **Deadlock** — две транзакции ждут блокировки друг друга. PostgreSQL обнаруживает и убивает одну
  (`ERROR: deadlock detected`).
* ```
  T1: BEGIN; UPDATE accounts SET balance = balance - 100 WHERE id = 1; -- lock on id=1
  T2: BEGIN; UPDATE accounts SET balance = balance + 100 WHERE id = 2; -- lock on id=2
  T1: UPDATE accounts SET balance = balance + 100 WHERE id = 2; -- wait for T2
  T2: UPDATE accounts SET balance = balance - 100 WHERE id = 1; -- wait for T1 → DEADLOCK
  ```
* **Предотвращение**:
  - **Одинаковый порядок блокировок** — всегда блокировать ресурсы в одном порядке (например, по ID).
  - **Короткие транзакции** — меньше времени удержания → меньше шанс.
  - **`LOCK TABLE ... IN SHARE MODE`** — явно заблокировать заранее.
  - **Advisory locks** — `pg_advisory_lock(key)` — application-level locks.
  - **Повтор с retry** — при deadlock, откатить и повторить транзакцию.
* **Мониторинг**: `pg_stat_database.deadlocks` — счётчик deadlock'ов. Alert, если растёт.

</details>

---

### Вопрос №4: [ Что такое SELECT FOR UPDATE и когда использовать? ]

<details>
<summary>Ответ</summary>

* `SELECT ... FOR UPDATE` — блокирует выбранные строки (row-level lock) до конца транзакции.
* Аналоги: `FOR SHARE` (shared lock), `FOR NO KEY UPDATE`, `FOR KEY SHARE`.
* **Использование**: read-modify-write паттерн:
```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- блокирует строку
-- application: проверка, вычисления
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```
* Без `FOR UPDATE` другая транзакция может изменить строку между SELECT и UPDATE (lost update).
* **`NOWAIT`** — `SELECT ... FOR UPDATE NOWAIT` — если строка заблокирована, сразу `ERROR`.
* **`SKIP LOCKED`** — `SELECT ... FOR UPDATE SKIP LOCKED` — пропустить заблокированные строки.
  Используется для job queues:
```sql
SELECT * FROM jobs WHERE status = 'pending'
ORDER BY created_at LIMIT 10 FOR UPDATE SKIP LOCKED;
```

</details>

---

### Вопрос №5: [ Как работают advisory locks? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

* **Advisory locks** — application-level блокировки, не привязанные к конкретным строкам/таблицам.
* `pg_advisory_lock(key)` — блокировка по 64-bit (или двух 32-bit) ключу. Держится до `pg_advisory_unlock(key)`
  или конца сессии.
* `pg_advisory_xact_lock(key)` — автоснимается при COMMIT/ROLLBACK.
* `pg_try_advisory_lock(key)` — неблокирующий, возвращает true/false.
* **Применения**:
  - Координация между процессами (лидер-выборы, single-writer).
  - Кратковременная блокировка ресурса по ID без `SELECT FOR UPDATE`.
  - Distributed lock (если несколько реплик — но только для primary).
  - Миграции: `pg_advisory_lock(42)` — только одна миграция одновременно.
* **Важно**: advisory locks — не механизм БД, а контракт приложения. Не блокируют обычные запросы.
  Нужны только для координации между клиентами одной БД.

</details>

---

</details>

---

## 4. Репликация и High Availability

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Как работает репликация в PostgreSQL? ]

<details>
<summary>Ответ</summary>

* **Streaming replication** — primary отправляет WAL (Write-Ahead Log) записиси репликам по TCP.
* **Физическая репликация** — побайтовая копия. Реплика — точная копия primary (схема, данные, индексы).
  Реплика — read-only (hot standby).
* ** WAL sender** (на primary) → **WAL receiver** (на replica). Реплика применяет WAL записиси.
* **Synchronous** — primary ждёт подтверждения от реплики (минимум одной) перед COMMIT.
  `synchronous_commit = on` + `synchronous_standby_names`.
  Гарантия: no data loss при падении primary. Минус: latency выше.
* **Asynchronous** — primary не ждёт. При падении primary последние транзакции могут быть потеряны
  (small replication lag). Быстрее.
* **Логическая репликация** (PostgreSQL 10+) — publisher/subscriber. Реплицирует на уровне строк (logical
  decoding). Можно реплицировать отдельные таблицы, разную схему, разные версии PostgreSQL.
* **Cascading replication** — реплика может быть upstream для другой реплики (снижает нагрузку на primary).

</details>

---

### Вопрос №2: [ Что такое split-brain и как предотвратить? ]

<details>
<summary>Ответ</summary>

* **Split-brain** — два узла считают себя primary одновременно. Возникает при network partition.
* Оба primary принимают записи → данные расходятся → конфликт при восстановлении.
* **Предотвращение**:
  - **Quorum** — большинство узлов должно согласовать failover. Если узел не может получить кворум —
    не становится primary.
  - **STONITH** (Shoot The Other Node In The Head) — перед продвижением реплики, убедиться, что старый
    primary выключен (fence).
  - **Witness server** — треться сторона (arbiter), которая помогает кворуму.
  - **Distributed consensus** — Raft, Paxos — для выбора лидера.
* В PostgreSQL: **Patroni** (использует etcd/Consul/ZooKeeper для кворума), **repmgr** (использует witness),
  **pg_auto_failover**.
* В k8s: operator (CrunchyData, Zalando) управляет failover, использует кворум через k8s API.

</details>

---

### Вопрос №3: [ Как настроить read replicas для масштабирования чтения? ]

<details>
<summary>Ответ</summary>

* **Архитектура**: primary (write) + N read replicas.
* **Маршрутизация**:
  - HAProxy / PgBouncer / ProxySQL — направляет write-запросы на primary, read — на реплики.
  - Application-level: разные connection pools для read/write.
* **Проблема replication lag**:
  - Пользователь пишет на primary → сразу читает с реплики → не видит своих данных (read-after-write consistency).
  - Решения:
    - **Read-after-write через primary**: критичные чтения (свой профиль) — с primary.
    - **Session stickiness**: после записи, N секунд читать с primary.
    - **WAL flush check**: приложение проверяет, догнала ли реплика (pg_wal_lsn_diff).
    - **Causal consistency**: track causality tokens.
* **Connection pooler** (PgBouncer) — обязательно для большого числа клиентов. `pool_mode = transaction`
  для большинства случаев.

</details>

---

### Вопрос №4: [ Что такое connection pooling и зачем нужен PgBouncer? ]

<details>
<summary>Ответ</summary>

* **Проблема**: каждое соединение в PostgreSQL = отдельный процесс (fork). ~10 MB memory per connection.
  1000 соединений = 10 GB RAM + context switch overhead.
* **PgBouncer** — lightweight connection pooler. Много клиентских соединений → мало серверных.
* **Режимы**:
  - **Session pooling** — серверное соединение привязано к клиентскому на всю сессию.
  - **Transaction pooling** — серверное соединение привязано на время транзакции. Между транзакциями —
    connection может быть переиспользован. **Рекомендуемый режим** (но: prepared statements, SET, TEMP tables
    не работают корректно).
  - **Statement pooling** — на каждый запрос. Не работает с транзакциями.
* **Плюсы**: меньше соединений к PostgreSQL → меньше memory → меньше context switches → выше throughput.
* **Минусы**: не поддерживает все фичи (session-level features в transaction mode), single-threaded
  (но PgBouncer 1.21+ — multi-threaded).

</details>

---

</details>

---

## 5. Бэкапы и восстановление

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Какие стратегии бэкапа PostgreSQL существуют? ]

<details>
<summary>Ответ</summary>

* **pg_dump / pg_dumpall** — логический бэкап (SQL или custom format).
  - `pg_dump -Fc -f backup.dump dbname` — custom format (compressed, parallel restore).
  - `pg_dumpall --globals-only` — roles, tablespaces.
  - Плюсы: переносимость (между версиями, архитектурами), выборочный restore.
  - Минусы: медленно для больших БД, не point-in-time.
* **pg_basebackup** — физический бэкап (binary copy of data directory + WAL).
  - `pg_basebackup -D /backup -Ft -z -P` — tar + gzip.
  - Используется для initial replica setup и PITR.
* **PITR** (Point-in-Time Recovery) — восстановление на момент времени:
  - Base backup + WAL archive (archive_mode=on, archive_command='cp %p /archive/%f').
  - `recovery_target_time = '2025-06-15 14:30:00'`.
* **Continuous WAL archiving** — `archive_command` копирует каждый WAL segment в архив (S3, NFS).
* **Snapshot-based** — если на файловом уровне (LVM, ZFS, EBS snapshot): freeze + snapshot + unfreeze.
  `pg_start_backup()` / `pg_stop_backup()` (или `pg_backup_start()` / `pg_backup_stop()` в PG 15+).
* **Рекомендация**: комбинировать — pg_basebackup + WAL archive (PITR) для полного восстановления +
  pg_dump для логического бэкапа (перенос между версиями).

</details>

---

### Вопрос №2: [ Что такое WAL и как он работает? ]

<details>
<summary>Ответ</summary>

* **WAL** (Write-Ahead Log) — журнал транзакций. Все изменения сначала записываются в WAL, потом применяются
  к data files.
* **Принцип**: write-ahead — данные записываются в WAL **до** изменения data pages. При crash recovery —
  replay WAL с последней checkpoint.
* **WAL segments** — файлы по 16 MB (настраивается). Имена: `000000010000000000000001`.
* **LSN** (Log Sequence Number) — позиция в WAL. `pg_current_wal_lsn()`.
* **Checkpoint** — момент, когда все изменённые data pages сброшены на диск. После checkpoint — WAL до
  checkpoint'а можно удалить (если не нужен для репликации/archive).
* **archive_mode = on** — копировать отработанные WAL segments в архив (`archive_command`).
* **pg_wal** (ранее pg_xlog) — директория с WAL файлами.
* **Replication slot** — гарантирует, что primary не удалит WAL, пока реплика не получила (предотвращает
  WAL buildup при отставании реплики).

</details>

---

### Вопрос №3: [ Как протестировать восстановление из бэкапа? ]

<details>
<summary>Ответ</summary>

* **Регулярный drill** — не реже раза в квартал.
* **Шаги**:
  1. Поднять тестовый сервер (или Docker контейнер).
  2. Восстановить base backup: `pg_basebackup` или распаковать tar.
  3. Настроить `recovery.signal` (PG 12+) или `standby.signal` + `restore_command` для WAL replay.
  4. Указать `recovery_target_time` / `recovery_target_lsn` для PITR.
  5. Запустить PostgreSQL, дождаться recovery complete.
  6. Проверить целостность: `pg_dump`, сравнить row counts, бизнес-запросы.
  7. Замерить время восстановления (RTO).
* **Автоматизация**: скрипт, который раз в неделю восстанавливает бэкап на staging и проверяет.
* **Метрики**: RTO (Recovery Time Objective), RPO (Recovery Point Objective).

</details>

---

### Вопрос №4: [ Что такое RPO и RTO? ]

<details>
<summary>Ответ</summary>

* **RPO** (Recovery Point Objective) — максимально допустимая потеря данных (по времени).
  - RPO = 0 — потеря данных недопустима (synchronous replication, WAL archiving).
  - RPO = 5 мин — допустимо потерять до 5 минут данных (async replication lag).
* **RTO** (Recovery Time Objective) — максимально допустимое время простоя при восстановлении.
  - RTO = 0 — недопустим простой (active-active, hot standby с автоматическим failover).
  - RTO = 1 час — восстановление в течение часа (автоматический failover + quick restore).
  - RTO = 24 часа — бэкап раз в день, ручное восстановление.
* **Связь с архитектурой**:
  - RPO/RTO = 0 → очень дорого (multi-region active-active).
  - RPO = минуты, RTO = минуты → standby + автоматический failover (Patroni).
  - RPO = часы, RTO = часы → бэкапы + ручное восстановление.

</details>

---

</details>

---

## 6. Миграции схемы

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Как безопасно добавить колонку с DEFAULT в большой таблице? ]

<details>
<summary>Ответ</summary>

* **Проблема**: `ALTER TABLE big_table ADD COLUMN status VARCHAR DEFAULT 'active'` в PostgreSQL < 11 —
  переписывает всю таблицу (блокирует на долгое время).
* **PostgreSQL 11+** — `ADD COLUMN ... DEFAULT` — metadata-only operation (хранит default в каталоге).
  Новые строки получают default при INSERT, старые — лениво при чтении. **Не блокирует**.
* **NOT NULL constraint с DEFAULT**:
  - PG 11+: `ADD COLUMN ... NOT NULL DEFAULT 'active'` — metadata-only (быстро).
  - PG < 11: добавить колонку (nullable, без default) → обратно обновить по батчам → добавить default →
    добавить NOT NULL.
* **Большая таблица + индекс на новую колонку**: `CREATE INDEX CONCURRENTLY` — без блокировки записи.
* **Backfill данных**:
```sql
-- Плохо: блокирует таблицу
UPDATE big_table SET new_col = old_col;

-- Хорошо: батчами
UPDATE big_table SET new_col = old_col WHERE id BETWEEN 1 AND 10000 AND new_col IS NULL;
-- ... повторять
```

</details>

---

### Вопрос №2: [ Как безопасно переименовать колонку? ]

<details>
<summary>Ответ</summary>

* **Проблема**: `ALTER TABLE ... RENAME COLUMN` — мгновенно в PostgreSQL, но **приложение, использующее
  старое имя, сломается**.
* **Expand → Migrate → Contract**:
  1. **Expand**: добавить новую колонку (с триггером или app-level dual-write).
  2. **Migrate**: backfill — скопировать данные.
  3. **Contract**: перевести приложение на новую колонку.
  4. **Cleanup**: удалить старую колонку.
* Для простых случаев (если есть окно обслуживания) — `RENAME COLUMN` + деплой приложения одновременно.
* **View** — создать view с новым именем, переключить приложение, потом переименовать в таблице.

</details>

---

### Вопрос №3: [ Как безопасно удалить большую таблицу в production? ]

<details>
<summary>Ответ</summary>

* **Проблема**: `DROP TABLE` — блокирует на время cleanup (освобождение extents, удаление индексов).
  Для огромной таблицы — секунды/минуты блокировки.
* **Решение**:
  1. Убедиться, что никто не использует таблицу (мониторинг запросов).
  2. `RENAME TABLE big_table TO big_table_to_drop;` — мгновенно (atomic rename).
  3. Приложение больше не видит `big_table`.
  4. `DROP TABLE big_table_to_drop;` — можно не спешить, никто не блокирует.
* Альтернатива для partitioned table — drop old partitions по одной: `DROP TABLE big_table_2023_01;`.
* В PostgreSQL `DROP TABLE` обычно быстро ( metadata + unlink файлов). Но на большой таблице с
  огромными индексами на нагруженном сервере — может вызвать IO spike.

</details>

---

### Вопрос №4: [ Какие инструменты для миграций вы знаете? ]

<details>
<summary>Ответ</summary>

* **golang-migrate** — Go-инструмент, SQL-файлы (up/down), поддерживает много БД. Embed в Go binary.
* **goose** — Go-инструмент, SQL или Go-функции для миграций.
* **Flyway** — Java-экосистема, SQL-файлы, versioned/repeatable migrations.
* **Liquibase** — XML/YAML/SQL changelog, supports rollback, diff.
* **Atlas** — declarative schema management (HCL), plan/diff/apply.
* **dbmate** — легковесный, SQL-файлы, Go.
* **Sqitch** — dependency-based, no up/down (change + revert scripts).
* **Prisma Migrate** — TypeScript-экосистема, schema-first.
* Best practices:
  - Миграции — immutable (не изменять применённую миграцию, создавать новую).
  - Forward-only (не использовать rollback в production, создавать revert миграцию).
  - Idempotent при возможности.
  - Тестировать на копии production данных.

</details>

---

</details>

---

## 7. Мониторинг и деградация

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Какие ключевые метрики мониторинга PostgreSQL? ]

<details>
<summary>Ответ</summary>

* **Подключения**: `pg_stat_activity` — active/idle connections, max_connections.
  Alert: connections > 80% of max.
* **Транзакции**: `pg_stat_database` — `xact_commit`, `xact_rollback`. TPS, rollback rate.
* **Replication lag**: `pg_stat_replication` — `replay_lag`, `write_lag`, `flush_lag` (seconds/bytes).
  Alert: lag > threshold.
* **Cache hit ratio**: `pg_stat_database` — `blks_hit / (blks_hit + blks_read)`. < 99% → увеличить
  `shared_buffers` или оптимизировать запросы.
* **Deadlocks**: `pg_stat_database.deadlocks`. Alert: > 0.
* **Table bloat**: `pg_stat_user_tables` — `n_dead_tup`, `last_vacuum`, `last_autovacuum`.
* **Slow queries**: `pg_stat_statements` — `total_exec_time`, `calls`, `mean_exec_time`, `rows` (до PG 13: `total_time`, `mean_time`).
* **WAL generation**: `pg_stat_wal` — WAL bytes generated. Sudden spike → investigate.
* **Autovacuum**: `pg_stat_user_tables` — `n_dead_tup`, `autovacuum_count`.
* **Locks**: `pg_locks` — waiting locks. `pg_blocking_pids(pid)` — кто блокирует.
* **Database size**: рост размера — alert, если аномальный (возможно bloat или missing cleanup).

</details>

---

### Вопрос №2: [ Что делать, если PostgreSQL тормозит? ]

<details>
<summary>Ответ</summary>

* **Шаг 1: pg_stat_activity** — найти long-running queries:
```sql
SELECT pid, now() - query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle' AND now() - query_start > interval '1 minute'
ORDER BY duration DESC;
```
* **Шаг 2: Locks** — кто кого блокирует:
```sql
SELECT blocked.pid, blocked.query, blocking.pid, blocking.query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON pg_blocking_pids(blocked.pid) @> ARRAY[blocking.pid];
```
* **Шаг 3: pg_stat_statements** — самые медленные запросы:
```sql
SELECT query, mean_exec_time, calls, total_exec_time
FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;
```
* **Шаг 4: Connections** — если `max_connections` исчерпан → проверить leak в приложении,
  увеличить pool или PgBouncer.
* **Шаг 5: I/O** — `iostat`, `vmstat` — disk насыщен? `pg_stat_database.blks_read` растёт быстро → cache miss.
* **Шаг 6: `EXPLAIN ANALYZE`** на проблемный запрос → оптимизация (индексы, переписывание).
* **Шаг 7: `pg_terminate_backend(pid)`** — убить проблемный запрос (крайняя мера).

</details>

---

### Вопрос №3: [ Что такое pg_stat_statements и как использовать? ]

<details>
<summary>Ответ</summary>

* **pg_stat_statements** — расширение PostgreSQL, собирает статистику по всем SQL-запросам.
* `shared_preload_libraries = 'pg_stat_statements'` в `postgresql.conf`.
* `CREATE EXTENSION pg_stat_statements;`
* Представление `pg_stat_statements`:
  - `query` — текст запроса (нормализованный, параметры → $1, $2).
  - `calls` — количество вызовов.
  - `total_exec_time` — суммарное время выполнения.
  - `mean_exec_time` — среднее время.
  - `rows` — суммарное количество строк.
  - `shared_blks_hit/read` — блоки из cache / с диска.
  - `temp_blks_read/written` — temporary files.
* **Использование**:
```sql
-- Топ-10 самых медленных запросов по среднему времени
SELECT query, mean_exec_time, calls, total_exec_time, rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Топ-10 по total time (что больше всего грузит сервер)
SELECT query, total_exec_time, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```
* `pg_stat_statements_reset()` — сброс статистики (например, после деплоя оптимизаций).

</details>

---

</details>

---

## 8. Шардинг и масштабирование

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Когда нужно шардировать БД и какие признаки? ]

<details>
<summary>Ответ</summary>

* **Признаки, что пора**:
  - CPU/IO на primary постоянно > 80%, оптимизация запросов не помогает.
  - Data size — десятки TB, индексы не влезают в RAM (shared_buffers).
  - Write throughput упёрся в IOPS диска.
  - Backup/restore время > RTO.
  - Таблица > 1 TB, vacuum занимает часы.
* **До шардинга** попробовать:
  - Оптимизация запросов, индексы.
  - Read replicas (разгрузить чтение).
  - Партиционирование (разгрузить I/O на одной машине).
  - Vertical scaling (больше CPU/RAM/SSD).
  - Архивация старых данных (detach old partitions).
* **Шардинг** — последнее средство, дорогое в эксплуатации.

</details>

---

### Вопрос №2: [ Какие стратегии шардинга существуют? ]

<details>
<summary>Ответ</summary>

* **Range-based** — по диапазону ключа. `shard 1: id 1-1M`, `shard 2: id 1M-2M`.
  Плюс: просто. Минус: hotspots (последний shard — самый активный).
* **Hash-based** — `shard = hash(key) % N`. Равномерно распределяет. Минус: добавление нового shard'а
  требует перераспределения всех данных (consistent hashing решает).
* **Consistent hashing** — ключи и шарды на кольце. Добавление/удаление шарда перемещает минимум данных.
  Используется в Cassandra, Redis Cluster.
* **Directory-based** — отдельный lookup-сервис (routing table) маппит key → shard.
  Гибко, но lookup — bottleneck (кэшировать).
* **Geo-based** — по географии (EU users → EU shard, US → US shard). Для data residency (GDPR).
* **Vertical sharding** (разделение по таблицам/функциям) — users на одном сервере, orders на другом.
  Проще, но не решает проблему одной огромной таблицы.

</details>

---

### Вопрос №3: [ Что такое Citus и как он работает? ]

<details>
<summary>Ответ</summary>

* **Citus** — расширение PostgreSQL для горизонтального масштабирования (sharding).
* Распределяет большие таблицы по нескольким PostgreSQL-узлам (worker nodes).
* **Distributed table** — таблица партиционируется по shard key (hash). Каждый shard — обычная PostgreSQL
  таблица на worker-узле.
* **Reference table** — реплицируется на все узлы (для small lookup tables, JOINs).
* **Coordinator node** — принимает запросы, маршрутизирует на worker'ы, собирает результаты.
* Поддерживает: distributed JOINs (co-located или broadcast), distributed aggregates, INSERT/UPDATE/DELETE.
* **Когда использовать**: multi-tenant SaaS (shard by tenant_id), time-series, real-time analytics.
* Плюсы: SQL-совместимость (почти), нет необходимости переписывать приложение полностью.
* Минусы: не все PostgreSQL-фичи работают (foreign keys на distributed tables, некоторые типы JOINs).

</details>

---

### Вопрос №4: [ Что такое Vitess и как он работает? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Vitess** — система шардинга для MySQL (разработана YouTube/Google).
* **VTGate** — proxy, маршрутизирует SQL-запросы на нужные shard'ы (прозрачно для приложения).
* **VTTablet** — sidecar для MySQL, управляет репликацией, backup, query serving.
* **VTOrc** — управление failover и репликацией (аналог Orchestrator).
* **Topodata** — топология (в etcd) — какие shard'ы где.
* **Sharding**: hash, range, custom. Resharding online (без даунтайма) — VReplication.
* **Vertical split** — перенос таблиц между keyspaces (без даунтайма).
* Используется: YouTube, Slack, GitHub (ранее), Square, Pinterest.
* Плюсы: transparent sharding для MySQL, battle-tested, online resharding.
* Минусы: сложность настройки, не все MySQL-фичи (savepoints, XA transactions), только MySQL.

</details>

---

### Вопрос №5: [ CAP-теорема и как она применима к БД? ]

<details>
<summary>Ответ</summary>

* **CAP** (Consistency, Availability, Partition tolerance) — при network partition нужно выбрать:
  - **CP** — сохранить консистентность, пожертвовать доступностью (отклонять запросы).
  - **AP** — сохранить доступность, пожертвовать консистентностью (принимать, но данные могут расходиться).
* **Partition tolerance** — не выбирается, network partition **могут** произойти.
* **PostgreSQL** — CP (при partition — реплики отстают, primary остаётся consistent).
* **Cassandra** — AP (tunable consistency, при partition — принимает записи, eventual consistency).
* **MongoDB** — CP (при partition — только primary принимает записи, если нет большинства — step down).
* **PACELC** — расширение: при Partition → CAP, Else (no partition) → Latency vs Consistency.
  - PostgreSQL — PC/EC (consistent всегда, но нет partition tolerance по умолчанию).
  - Cassandra — PA/EL (available + low latency, eventual consistency).

</details>

---

</details>

---

## 9. Сравнение БД: SQL vs NoSQL vs columnar vs search

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ SQL (реляционные) vs NoSQL — принципиальная разница ]

<details>
<summary>Ответ</summary>

| Критерий | SQL (PostgreSQL, MySQL) | NoSQL |
|----------|--------------------------|-------|
| **Схема** | Жёсткая (schema-on-write) | Гибкая/отсутствует (schema-on-read) |
| **Транзакции** | ACID | BASE (eventual consistency), некоторые ACID |
| **JOIN** | Да (сложные запросы) | Нет или ограниченные |
| **Масштабирование** | Вертикальное (+ read replicas) | Горизонтальное (sharding built-in) |
| **Data model** | Таблицы, relations | Key-value, document, column-family, graph |
| **Запросы** | SQL (декларативный) | Разные (MongoDB query, Cassandra CQL, Cypher) |
| **Consistency** | Strong (по умолчанию) | Tunable / eventual |

* **SQL** — когда данные реляционные, нужны ACID-транзакции, сложные JOIN, аналитика.
* **NoSQL** — когда данные нереляционные, нужен горизонтальный масштаб, гибкая схема,
  high write throughput.

</details>

---

### Вопрос №2: [ Типы NoSQL БД и когда что выбрать ]

<details>
<summary>Ответ</summary>

| Тип | Примеры | Data model | Когда выбирать |
|-----|---------|------------|----------------|
| **Key-value** | Redis, Memcached, Aerospike, DynamoDB | key → value | Cache, session store, simple lookups, high throughput |
| **Document** | MongoDB, CouchDB, Firestore | key → JSON/BSON document | Каталоги, CMS, user profiles, content management, гибкая схема |
| **Column-family** | Cassandra, HBase, ScyllaDB | row key → column families | Time-series, IoT, write-heavy, massive scale, multi-DC |
| **Graph** | Neo4j, ArangoDB, Dgraph | nodes + edges + properties | Social graph, fraud detection, recommendation, knowledge graph |
| **Search** | Elasticsearch, OpenSearch, Solr | inverted index | Full-text search, faceted search, log analytics |

* **Key-value** — простейший, самый быстрый. Нет сложных запросов — только GET/SET по key.
* **Document** — гибкая схема, вложенные объекты, индексы по полям. Хорош для эволюционирующих
  данных (prototype → production).
* **Column-family** — write-optimized, linear scalability. Данные моделируются вокруг query
  patterns (query-driven design).
* **Graph** — когда отношения важнее самих данных. JOIN в SQL работает плохо для deep graphs
  (N+1 problem, exponential explosion).
* **Search** — full-text search, fuzzy matching, aggregations. Не primary DB — secondary index.

</details>

---

### Вопрос №3: [ Когда выбрать PostgreSQL, а когда MongoDB? ]

<details>
<summary>Ответ</summary>

* **Выбирайте PostgreSQL, если:**
  - **ACID-транзакции** критичны (финансы, заказы, биллинг).
  - **Сложные запросы** — JOIN по нескольким таблицам, subqueries, window functions, CTE.
  - **Data integrity** — foreign keys, check constraints, unique constraints, triggers.
  - **Аналитика** — OLAP-запросы (GROUP BY, HAVING, аналитические функции).
  - **JSON нужен ограниченно** — PostgreSQL имеет JSONB (с индексами GIN).
  - **Consistency** — strong consistency важнее availability при partition.

* **Выбирайте MongoDB, если:**
  - **Гибкая схема** — документы одной коллекции могут иметь разные поля (catalog, CMS).
  - **Embedded documents** — вложенные объекты вместо JOIN (denormalization).
  - **Horizontal scaling** — built-in sharding (MongoDB Sharded Cluster).
  - **Write-heavy** — высокая write нагрузка, eventual consistency приемлема.
  - **Rapid prototyping** — схема меняется часто, нет времени на миграции.
  - **Geo-distributed** — MongoDB Atlas Global Cluster (data locality по geo-sharding).

* **На практике:**
  - PostgreSQL + JSONB покрывает большинство use cases MongoDB (с ACID).
  - MongoDB выбирают, когда реляционная модель не подходит (deeply nested, variable schema,
    massive horizontal scale).

</details>

---

### Вопрос №4: [ Когда выбрать columnar БД (ClickHouse, Snowflake)? ]

<details>
<summary>Ответ</summary>

| Критерий | Row-oriented (PostgreSQL) | Column-oriented (ClickHouse) |
|----------|---------------------------|------------------------------|
| **Storage** | Строки (вся строка рядом) | Колонки (значения одной колонки рядом) |
| **Чтение** | Вся строка (или index scan) | Только нужные колонки |
| **Запись** | Быстрая (одна строка) | Медленная (обновить несколько колонок) |
| **Compression** | Ниже | Выше (одинаковый тип данных в колонке) |
| **Aggregation** | Медленнее (scan all columns) | Очень быстро (scan only needed columns) |
| **Point lookup** | Быстрый (by PK) | Медленный (нужно собрать строку из колонок) |
| **OLTP** | Да | Нет |
| **OLAP** | Нет (медленно для больших данных) | Да |

* **Выбирайте columnar (ClickHouse, Snowflake, BigQuery), если:**
  - **OLAP / Analytics** — агрегации по миллионам/миллиардам строк (SUM, COUNT, GROUP BY).
  - **Time-series** — метрики, логи, события (ClickHouse идеален для log analytics).
  - **Read-heavy** — много аналитических запросов, мало точечных записей.
  - **Compression** — данные сжимаются в 5–10 раз лучше, чем row-oriented.
  - **Big data** — терабайты/петабайты аналитических данных.

* **Выбирайте row-oriented (PostgreSQL, MySQL), если:**
  - **OLTP** — транзакции, точечные чтения/записи по primary key.
  - **ACID** — нужны транзакции с изоляцией.
  - **UPDATE/DELETE** — частые обновления строк (columnar — append-only, update = rewrite).

* **Hybrid:** PostgreSQL (OLTP) + ClickHouse (OLAP) — CDC через Kafka/Debezium.

</details>

---

### Вопрос №5: [ Когда выбрать Cassandra / ScyllaDB? ]

<details>
<summary>Ответ</summary>

* **Cassandra** — wide-column store, распределённая, masterless, AP (highly available).

* **Выбирайте Cassandra/ScyllaDB, если:**
  - **Massive write throughput** — миллионы writes/сек (IoT, event logging, time-series).
  - **Multi-datacenter** — native multi-DC replication (active-active, geo-distributed).
  - **Linear scalability** — добавь node → больше throughput (нет re-sharding).
  - **Always-on** — masterless, нет single point of failure, нет failover downtime.
  - **Tunable consistency** — ONE (fast, eventual) до ALL (slow, strong).
  - **Append-only / time-series** — данные пишутся по partition key, сортировка по clustering
    key (time-ordered).

* **НЕ выбирайте Cassandra, если:**
  - Нужны **JOIN** — Cassandra не поддерживает.
  - Нужны **ACID-транзакции** — нет (только lightweight transactions / Paxos — медленно).
  - Нужны **ad-hoc запросы** — Cassandra query-driven (запросы = primary key), ad-hoc
    требуют secondary indexes (медленные) или Spark.
  - Нужны **UPDATE/DELETE** — Cassandra оптимизирована для append, tombstones замедляют.
  - **Small dataset** — overhead Cassandra не оправдана (< 100 ГБ → PostgreSQL).

* **ScyllaDB** — Cassandra-совместимая, написана на C++ (вместо Java), в 5–10 раз выше
  throughput, ниже latency. Drop-in replacement.

</details>

---

### Вопрос №6: [ Когда выбрать Elasticsearch / OpenSearch? ]

<details>
<summary>Ответ</summary>

* **Elasticsearch** — distributed search engine, built on Lucene, inverted index.

* **Выбирайте Elasticsearch, если:**
  - **Full-text search** — поиск по тексту (токенизация, стемминг, fuzzy, synonyms).
  - **Faceted search** — фильтры по категориям (e-commerce: brand, price range, color).
  - **Log analytics** — ELK/EFK stack (логи, метрики, APM).
  - **Autocomplete / suggest** — поиск с автодополнением, suggesters.
  - **Geo search** — поиск в радиусе, bounding box, geo aggregations.
  - **Vector search** — semantic search, kNN (с версии 7.x + 8.x).

* **НЕ выбирайте Elasticsearch как primary DB, если:**
  - **ACID-транзакции** — нет (eventual consistency).
  - **Relational data** — нет JOIN (parent-child — медленно и ограниченно).
  - **Point lookups** — медленнее, чем key-value или SQL by PK.
  - **Frequent updates** — Lucene segments immutable, update = delete + reindex.
  - **Cost** — Elasticsearch тяжёлый (JVM, много RAM для heap + filesystem cache).

* **Pattern:** Primary DB (PostgreSQL/MongoDB) → CDC/async → Elasticsearch (search index).
  Приложение пишет в primary DB, search index обновляется асинхронно.

</details>

---

### Вопрос №7: [ Полная таблица: какая БД для какой задачи ]

<details>
<summary>Ответ</summary>

| Задача | БД | Почему |
|--------|-----|--------|
| OLTP (транзакции, заказы, биллинг) | PostgreSQL, MySQL | ACID, JOIN, constraints, point lookups |
| OLAP (аналитика, reporting) | ClickHouse, Snowflake, BigQuery | Columnar, fast aggregations, compression |
| Cache | Redis, Memcached | In-memory, sub-ms latency |
| Session store (small) | Redis | TTL, in-memory, simple |
| Session store (large, TB) | Aerospike | SSD-backed, low latency, cheaper |
| Document store (CMS, catalog) | MongoDB | Flexible schema, embedded docs |
| Time-series (IoT, metrics) | ClickHouse, TimescaleDB, InfluxDB | Optimized for time-ordered, high write |
| Wide-column (massive writes) | Cassandra, ScyllaDB | Multi-DC, linear scale, always-on |
| Full-text search | Elasticsearch, OpenSearch | Inverted index, fuzzy, faceted |
| Graph (social, fraud) | Neo4j, ArangoDB | Traversals, path queries |
| Key-value (simple lookups) | Redis, DynamoDB | Fast, simple, scalable |
| Vector / semantic search | Elasticsearch (kNN), Pinecone, Milvus | Embedding similarity, ANN |
| Event log / audit | Kafka (→ ClickHouse/S3 for archive) | Append-only, replay, retention |
| Counter / leaderboard | Redis (Sorted Set) | Built-in ZADD/ZRANGE |
| Geo / spatial | PostgreSQL (PostGIS), Elasticsearch | PostGIS — spatial queries, ES — geo search |

</details>

---

### Вопрос №8: [ Практический сценарий: выбор БД для e-commerce платформы ]

<details>
<summary>Ответ</summary>

| Компонент | БД | Почему |
|-----------|-----|--------|
| Users, orders, payments | PostgreSQL | ACID, FK, transactions, OLTP |
| Product catalog (variable attributes) | MongoDB / PostgreSQL (JSONB) | Flexible schema (разные товары — разные атрибуты) |
| Search (products, autocomplete) | Elasticsearch | Full-text, faceted, fuzzy, synonyms |
| Cart / session | Redis | TTL, in-memory, low latency |
| Analytics (sales, funnels) | ClickHouse | Columnar, fast aggregations over millions of rows |
| Recommendations (user-item graph) | Neo4j / Redis (collaborative filtering) | Graph traversals / sorted sets |
| Inventory (high write, multi-DC) | Cassandra / ScyllaDB | Write-heavy, multi-DC, always-on |
| Audit log / event sourcing | Kafka → S3/ClickHouse | Append-only, replay, long retention |
| Rate limiter | Redis | INCR + EXPIRE, Lua |
| Image / file storage | S3 / MinIO | Object storage, CDN |

* **Поток данных:**
  1. API → PostgreSQL (order) + Kafka (order_created event).
  2. Kafka → ClickHouse (analytics), Elasticsearch (search index update).
  3. API → Redis (cart, session, rate limit).
  4. API → MongoDB (product catalog) → Elasticsearch (search index).
  5. Cassandra → inventory updates (real-time, multi-DC).

* **Ключевой принцип:** **polyglot persistence** — разные БД для разных задач, связанные
  через events (Kafka) и CDC. Не пытаться уместить всё в одну БД.

</details>

---

</details>

---
