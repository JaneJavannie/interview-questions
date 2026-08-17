# Дублирующиеся вопросы (перенесены в sql-postgres-questions.md)

> **Внимание:** Этот файл содержит вопросы, которые дублируют содержание
> `sql-postgres-questions.md`. Уникальные вопросы уже перенесены туда.
> Удалите этот файл после проверки дубликатов.

---

## 1. Индексы

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

### Вопрос №2: [ Что такое connection pooling и зачем нужен PgBouncer? ]

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

</details>

---

## 6. Мониторинг и деградация

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

## 7. Шардинг и масштабирование

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ CAP-теорема и как она применима к БД? ]

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

## 8. Сравнение БД: SQL vs NoSQL vs columnar vs search

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

### Вопрос №2: [ Когда выбрать columnar БД (ClickHouse, Snowflake)? ]

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

</details>

---
