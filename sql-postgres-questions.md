# SQL / PostgreSQL — Вопросы и ответы для Middle/Senior

Репозиторий предназначен для подготовки к собеседованиям на позицию Backend разработчика.
В данном файле собраны вопросы по SQL и PostgreSQL (внутреннее устройство, оптимизация,
практические задачи с решениями).

Уровни сложности:
- ![Static Badge](https://img.shields.io/badge/Middle-yellow) — Middle разработчик
- ![Static Badge](https://img.shields.io/badge/Senior-red) — Senior разработчик

---

# Оглавление

1. [Теория и внутреннее устройство PostgreSQL](#1-теория-и-внутреннее-устройство-postgresql)
2. [Индексы](#2-индексы)
3. [Транзакции и MVCC](#3-транзакции-и-mvcc)
4. [Оптимизация запросов](#4-оптимизация-запросов)
5. [Практические задачи (Middle)](#5-практические-задачи-middle)
6. [Практические задачи (Senior)](#6-практические-задачи-senior)
7. [Администрирование и эксплуатация](#7-администрирование-и-эксплуатация)

---

## 1. Теория и внутреннее устройство PostgreSQL

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Как устроена архитектура PostgreSQL? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

PostgreSQL использует **клиент-серверную архитектуру** с процессной моделью (не потоковой,
как MySQL).

**Основные компоненты:**

```
                    ┌──────────────────────────────────────────┐
                    │              PostgreSQL Server             │
                    │                                           │
  Client ──conn──►  │  postmaster (главный процесс)              │
                    │     ├── backend (1 на соединение)          │
                    │     ├── background writer                  │
                    │     ├── WAL writer / WAL sender            │
                    │     ├── checkpointer                       │
                    │     ├── autovacuum launcher / workers      │
                    │     ├── stats collector                    │
                    │     ├── logical replication worker         │
                    │     └── parallel query workers             │
                    │                                           │
                    │  Shared Buffers (shared memory)            │
                    │  WAL Buffers                               │
                    └──────────────────────────────────────────┘
```

- **postmaster** — слушает порт, принимает соединения, форкает backend-процесс на каждое
  соединение (process-per-connection, не thread-per-connection);
- **backend** — обслуживает один коннект: парсит, планирует, выполняет SQL, возвращает результат;
- **background writer** — сбрасывает грязные страницы из shared buffers на диск;
- **WAL writer** — записывает WAL (Write-Ahead Log) на диск;
- **checkpointer** — создаёт контрольные точки (синхронизация буферов с диском);
- **autovacuum** — автоматически очищает «мёртвые» версии строк (MVCC garbage collection).

**Память:**
- **Shared buffers** — общий кэш страниц данных (по умолчанию 128MB, рекомендуется 25% RAM);
- **WAL buffers** — буфер WAL (по умолчанию ~3MB);
- **Local memory (per backend)** — work_mem, maintenance_work_mem, temp_buffers.

</details>

---

### Вопрос №2: [ Что такое MVCC? Как это работает в PostgreSQL? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**MVCC (Multi-Version Concurrency Control)** — механизм управления конкурентным доступом,
при котором читатели не блокируют писателей, а писатели — читателей. Каждая транзакция видит
**снимок (snapshot)** данных на момент её начала.

**Реализация в PostgreSQL:**

Каждая строка (tuple) в таблице содержит скрытые системные колонки:
- **xmin** — ID транзакции, которая вставила эту версию строки;
- **xmax** — ID транзакции, которая удалила/обновила эту версию (0 = активна);
- **cmin / cmax** — ID команды внутри транзакции;
- **ctid** — физический адрес строки (page + offset), используется для chain обновлений.

**Как работает UPDATE:**
UPDATE в PostgreSQL = DELETE старой версии + INSERT новой. Старая версия не затирается —
помечается `xmax`, новая версия вставляется с новым `xmin`. Обе версии существуют одновременно,
пока vacuum не очистит старую.

**Как работает SELECT:**
Транзакция T берёт snapshot: записывает текущие активные транзакции. Строка видна, если:
- `xmin` закоммичен и был до начала snapshot;
- `xmax` не закоммичен ИЛИ был после начала snapshot (или 0).

**Виды снимков:**
- `REPEATABLE READ` — один snapshot на всю транзакцию;
- `READ COMMITTED` — новый snapshot на каждый оператор.

**Последствия:**
- Читатели **никогда не блокируются** писателями (нет read locks);
- UPDATE/DELETE создают **dead tuples** → нужен **VACUUM** для очистки;
- Длительные транзакции мешают vacuum → **bloat** (раздувание таблиц/индексов);
- Нет rollback сегментов (как в Oracle/InnoDB) — старые версии в той же таблице.

</details>

---

### Вопрос №3: [ Что такое VACUUM? В чём разница между VACUUM, VACUUM FULL и autovacuum? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**VACUUM** — процесс очистки «мёртвых» версий строк (dead tuples), созданных при UPDATE/DELETE
вследствие MVCC. Vacuum не удаляет данные физически — он помечает пространство как доступное
для повторного использования (free space map).

| Операция | Что делает | Блокирует таблицу? | Возвращает место ОС? |
|---|---|---|---|
| `VACUUM` | Помечает dead tuples как free space | Нет | Нет (место остаётся в таблице) |
| `VACUUM FULL` | Перестраивает таблицу целиком, убирая dead space | **Да** (AccessExclusiveLock) | **Да** |
| `VACUUM ANALYZE` | VACUUM + обновляет статистику для planner | Нет | Нет |
| `autovacuum` | Автоматический VACUUM/ANALYZE по триггерам | Нет | Нет |

**Когда нужен VACUUM FULL:**
- Таблица сильно раздута (bloat) после массовых DELETE;
- Нужно вернуть место ОС;
- **Минус:** блокирует таблицу на время выполнения → делать в maintenance window.

**Альтернатива без блокировки:**
```sql
-- pg_repack (расширение) — перестраивает таблицу без долгой блокировки:
-- CREATE EXTENSION pg_repack;
-- SELECT repack_table('my_table');
```

**Параметры autovacuum:**
- `autovacuum_vacuum_threshold` — базовое число изменений (default 50);
- `autovacuum_vacuum_scale_factor` — доля строк (default 0.2 = 20%);
- Триггер: `changes > threshold + scale_factor * reltuples`;
- Для больших таблиц 20% — слишком редко → нужно снижать scale_factor или использовать
  per-table настройки.

</details>

---

### Вопрос №4: [ Что такое WAL и checkpoint? Зачем нужны? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**WAL (Write-Ahead Log)** — журнал предзаписи. Перед изменением данных на диске, PostgreSQL
сначала записывает описание изменения в WAL. Это обеспечивает:
- **Durability** — восстановление после сбоя (recovery);
- **Replication** — streaming replication передаёт WAL на реплики;
- **Point-in-time recovery (PITR)** — восстановление на любой момент времени.

**Как работает:**
1. Изменение данных → сначала в WAL buffers (в памяти);
2. WAL buffers сбрасываются на диск при COMMIT (синхронно, если `synchronous_commit=on`);
3. Грязные страницы данных (shared buffers) сбрасываются позже (background writer / checkpoint).

**Checkpoint** — момент, когда гарантируется, что все данные до этой точки записаны на диск.
При восстановлении PostgreSQL проигрывает WAL начиная с последнего checkpoint.

**Параметры:**
- `wal_level` — minimal / replica / logical (определяет, сколько информации в WAL);
- `synchronous_commit` — on/off/local (влияет на latency vs durability);
- `max_wal_size` — максимальный размер WAL между checkpoints (default 1GB);
- `checkpoint_timeout` — максимальное время между checkpoints (default 5min).

**Trade-off:** Более частые checkpoints → быстрее recovery, но больше I/O. Более редкие →
дольше recovery, но меньше overhead.

</details>

---

### Вопрос №5: [ Как работает query planner в PostgreSQL? Из чего состоит? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

PostgreSQL использует **cost-based optimizer (CBO)** — планировщик на основе стоимости.

**Этапы выполнения запроса:**
```
SQL text → Parser → Parse Tree → Analyzer (semantic) → Rewritten Tree
         → Planner → Plan Tree → Executor → Result
```

**Parser** — синтаксический разбор (проверка SQL-синтаксиса, построение parse tree).
**Analyzer/Rewriter** — семантический анализ (резолв таблиц, колонок, типов), применение
правил (rules), раскрытие views.
**Planner** — генерация и выбор плана выполнения.
**Executor** — выполнение плана.

**Планировщик (Planner):**
1. Оценивает **стоимость** каждого возможного плана (cost estimation);
2. Выбирает план с минимальной total_cost.

**Компоненты стоимости:**
- **startup_cost** — стоимость до возвращения первой строки;
- **total_cost** — полная стоимость (startup + обработка всех строк);
- Стоимость зависит от:
  - **Статистики** (pg_statistic, собирается ANALYZE): распределение значений, гистограммы,
    most_common_values, correlation;
  - **Размера таблицы** (pg_class.reltuples);
  - **Параметров** (seq_page_cost, random_page_cost, cpu_tuple_cost, parallel_*).

**Стратегии соединения (Join methods):**
- **Nested Loop** — для маленьких таблиц или когда есть индекс на inner table;
- **Hash Join** — для больших таблиц без индексов (build hash table на smaller side);
- **Merge Join** — если обе стороны отсортированы (или можно отсортировать).

**Методы доступа к данным:**
- **Sequential Scan** — полное сканирование таблицы;
- **Index Scan** — чтение через индекс + чтение строк таблицы;
- **Index Only Scan** — чтение только из индекса (если все нужные колонки в индексе + visibility map);
- **Bitmap Index Scan + Bitmap Heap Scan** — индекс строит битмап страниц, потом batch-чтение.

**EXPLAIN:**
```sql
EXPLAIN SELECT * FROM users WHERE email = 'a@b.c';
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'a@b.c'; -- + реальное выполнение
EXPLAIN (ANALYZE, BUFFERS) SELECT ...; -- + статистика I/O
```

</details>

---

### Вопрос №6: [ Чем отличается NULL в SQL? Как с ним работать? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**NULL** — не значение, а **отсутствие значения** (unknown). Это создаёт неинтуитивное поведение:

1. **Сравнения с NULL всегда дают NULL (не false):**
   ```sql
   NULL = NULL   → NULL (не true!)
   NULL <> NULL  → NULL (не true!)
   NULL > 0      → NULL
   ```

2. **WHERE отфильтровывает NULL:**
   ```sql
   SELECT * FROM users WHERE phone != '+123';  -- не вернёт строки с phone IS NULL!
   -- Нужно:
   SELECT * FROM users WHERE phone IS DISTINCT FROM '+123';
   -- или:
   SELECT * FROM users WHERE phone != '+123' OR phone IS NULL;
   ```

3. **Агрегатные функции игнорируют NULL:**
   ```sql
   AVG(col) = SUM(col) / COUNT(col)   -- COUNT считает только non-NULL!
   COUNT(*) считает все строки, COUNT(col) — только non-NULL col
   ```

4. **NULL в сортировке:**
   ```sql
   ORDER BY col ASC   → NULLs last (по умолчанию в PostgreSQL)
   ORDER BY col DESC  → NULLs first
   -- Явно:
   ORDER BY col NULLS FIRST / NULLS LAST
   ```

5. **COALESCE / NULLIF:**
   ```sql
   COALESCE(NULL, NULL, 'default') → 'default'
   NULLIF(col, 0) → NULL если col = 0 (защита от деления на ноль)
   ```

6. **NULL в unique constraint:**
   - В PostgreSQL: `UNIQUE(col)` — **разрешает** несколько NULL (NULL != NULL);
   - `UNIQUE NON NULL col` или partial index для запрета NULL.

7. **NULL в JOIN:**
   ```sql
   -- NULL не соединяется при INNER JOIN (NULL != NULL)
   -- Для соединения по NULL используйте:
   SELECT * FROM a JOIN b ON a.id IS NOT DISTINCT FROM b.id;
   ```

</details>

---

### Вопрос №7: [ Что такое CTE и рекурсивные CTE? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**CTE (Common Table Expression)** — временный именованный результат, существующий в рамках
одного запроса. Улучшает читаемость и позволяет рекурсию.

**Простое CTE:**
```sql
WITH active_users AS (
    SELECT * FROM users WHERE status = 'active'
)
SELECT au.name, COUNT(o.id) as orders
FROM active_users au
LEFT JOIN orders o ON o.user_id = au.id
GROUP BY au.name;
```

**Рекурсивное CTE** — для работы с иерархическими данными (деревья, графы):

```sql
-- Найти всех подчинённых (включая вложенных) менеджера с id=1
WITH RECURSIVE subordinates AS (
    -- Базовый случай (anchor)
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE id = 1

    UNION ALL

    -- Рекурсивный шаг
    SELECT e.id, e.name, e.manager_id, s.level + 1
    FROM employees e
    INNER JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates ORDER BY level;
```

**Структура рекурсивного CTE:**
1. **Anchor** — нерекурсивная часть (стартовая точка);
2. `UNION [ALL]` — объединение (ALL быстрее, но может дать дубликаты);
3. **Recursive** — ссылается на само себя; выполняется пока есть новые строки.

**Ограничение глубины:**
```sql
-- Защита от бесконечной рекурсии:
SET statement_timeout = '10s';
-- Или ограничение в запросе:
... WHERE level <= 10
```

**PostgreSQL 12+:** CTE по умолчанию **inline** (могут быть оптимизированы planner'ом).
До 12 — CTE были optimization fence (выполнялись отдельно). Для принудительного fence:
```sql
WITH cte AS MATERIALIZED (...) ...
WITH cte AS NOT MATERIALIZED (...) ...
```

</details>

---

### Вопрос №8: [ Что такое partitioning в PostgreSQL? Какие виды бывают? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**Partitioning** — разбиение большой таблицы на меньшие физические части (partition),
оставаясь одной логической таблицей для запросов.

**Виды партиционирования (PostgreSQL 10+, декларативное):**

1. **RANGE** — по диапазону значений:
```sql
CREATE TABLE events (
    id BIGSERIAL,
    created_at TIMESTAMPTZ NOT NULL,
    data JSONB
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

2. **LIST** — по списку значений:
```sql
CREATE TABLE users (...) PARTITION BY LIST (country);
CREATE TABLE users_us PARTITION OF users FOR VALUES IN ('US');
CREATE TABLE users_eu PARTITION OF users FOR VALUES IN ('DE', 'FR', 'UK');
```

3. **HASH** — по хэшу (равномерное распределение):
```sql
CREATE TABLE users (...) PARTITION BY HASH (id);
CREATE TABLE users_0 PARTITION OF users FOR VALUES WITH (modulus 4, remainder 0);
CREATE TABLE users_1 PARTITION OF users FOR VALUES WITH (modulus 4, remainder 1);
```

**Когда использовать:**
- Таблица > 10–50 млн строк;
- Запросы обычно фильтруются по partition key (например, по дате);
- Нужно быстро удалять старые данные (`DROP TABLE partition` vs `DELETE`).

**Partition pruning** — planner исключает ненужные партиции при запросе:
```sql
-- Если запрос: WHERE created_at >= '2024-02-01'
-- Planner сканирует только events_2024_02
```

**Подводные камни:**
- **Global unique constraint** — нельзя создать UNIQUE constraint на партиционированную таблицу
  (только если partition key входит в constraint). Решение: partial unique indexes per partition;
- **Foreign keys ON partitioned table** — поддерживается только с PostgreSQL 12+ (как referencing);
- **DEFAULT partition** — ловушка для невалидных данных (всё, что не попало в другие партиции);
- **Pruning работает только если** partition key в условии — константа, а не параметр
  (prepared statements могут не prune без generic plan).

**Автоматическое создание партиций (pg_partman):**
```sql
-- Расширение pg_partman автоматически создаёт новые партиции
-- и удаляет старые
```

</details>

---

</details>

---

## 2. Индексы

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Какие типы индексов есть в PostgreSQL? Когда какой использовать? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

| Тип | Индекс | Когда использовать |
|---|---|---|
| **B-Tree** | по умолчанию | Точечный поиск, диапазоны, сортировка. Универсальный |
| **Hash** | `USING HASH` | Только точное равенство (`=`). Не поддерживает диапазоны/сортировку |
| **GIN** | `USING GIN` | Полнотекстовый поиск, массивы, JSONB, составные ключи. Быстрый поиск, медленная запись |
| **GiST** | `USING GIST` | Геометрические данные (PostGIS), диапазоны, full-text search (tsvector) |
| **SP-GiST** | `USING SPGIST` | Пространственные данные, телефонные номера, неравномерные деревья |
| **BRIN** | `USING BRIN` | Огромные таблицы с естественной упорядоченностью (time-series). Минимальный размер |
| **Bloom** | расширение | Множественные колонки, любые комбинации фильтров (approximate) |

**Примеры:**

```sql
-- B-Tree (по умолчанию) — точечный поиск и диапазоны
CREATE INDEX idx_users_email ON users(email);

-- Hash — только точное равенство
CREATE INDEX idx_users_id_hash ON users USING HASH (id);

-- GIN — полнотекстовый поиск и JSONB
CREATE INDEX idx_docs_body ON docs USING GIN (to_tsvector('russian', body));
CREATE INDEX idx_data_tags ON events USING GIN (tags);          -- массив
CREATE INDEX idx_data_json ON events USING GIN (data);          -- JSONB

-- GiST — геометрия и диапазоны
CREATE INDEX idx_geo ON places USING GIST (location);
CREATE INDEX idx_schedule ON schedule USING GIST (during);      -- tsrange

-- BRIN — большие таблицы, упорядоченные по дате
CREATE INDEX idx_events_brin ON events USING BRIN (created_at);

-- Partial index — индекс на подмножество строк
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';

-- Expression index — индекс по выражению
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
```

</details>

---

### Вопрос №2: [ Чем Index Scan отличается от Index Only Scan и Bitmap Scan? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**1. Sequential Scan:**
- Читает всю таблицу страницу за страницей;
- Используется, когда нет подходящего индекса или нужно прочитать большую часть таблицы;
- Для больших таблиц с фильтром по ~5% строк → Sequential может быть медленнее Index Scan.

**2. Index Scan:**
- Читает индекс → находит ctid (адрес строки) → читает строку из таблицы (heap fetch);
- Два I/O на строку: индекс + таблица;
- Хорошо для точечного поиска (несколько строк);
- **Может** возвращать данные в порядке индекса (если ORDER BY совпадает).

**3. Index Only Scan:**
- Все нужные колонки есть в индексе → **не нужно читать таблицу** (no heap fetch);
- **Требование:** Visibility Map должен показывать, что страница all-visible (не было изменений
  после последнего vacuum). Если страница не all-visible → fallback к heap fetch;
- Поэтому важен регулярный VACUUM для обновления visibility map.

**4. Bitmap Index Scan + Bitmap Heap Scan:**
- Index Scan: для каждой найденной строки → random I/O в таблицу (медленно при большом количестве строк);
- Bitmap: индекс строит **битмап страниц** (не строк) → затем batch-чтение страниц последовательно;
- Хорошо для промежуточного количества строк (~5–25% таблицы);
- Bitmap теряет порядок (если нужен ORDER BY — может потребоваться Sort).

```sql
-- Index Scan (планировщик выбирает при малом количестве строк)
EXPLAIN SELECT * FROM users WHERE id = 42;
-- Index Scan using users_pkey on users

-- Index Only Scan (все колонки в индексе + страница all-visible)
EXPLAIN SELECT id FROM users WHERE id = 42;
-- Index Only Scan using users_pkey on users

-- Bitmap Scan (промежуточное количество строк)
EXPLAIN SELECT * FROM users WHERE status = 'active';
-- Bitmap Heap Scan on users
--   -> Bitmap Index Scan on idx_users_status
```

</details>

---

### Вопрос №3: [ Что такое покрывающий индекс (covering index)? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Покрывающий индекс** — индекс, содержащий все колонки, нужные запросу. Планировщик может
использовать **Index Only Scan** — читать только индекс, не обращаясь к таблице.

В PostgreSQL для этого используется `INCLUDE`:

```sql
-- Запрос:
SELECT id, email, name FROM users WHERE email = 'a@b.c';

-- Обычный индекс: Index Scan (нужно читать таблицу для name)
CREATE INDEX idx_email ON users(email);

-- Покрывающий индекс: Index Only Scan (name в INCLUDE)
CREATE INDEX idx_email_covering ON users(email) INCLUDE (name, id);
-- email — key column (по ней фильтруем)
-- name, id — included columns (только для чтения, не участвуют в B-Tree)
```

**Разница INCLUDE vs key column:**
- **Key column** — участвует в B-Tree (сортировка, поиск), но ограничивает типы
  (не все типы можно в B-Tree key);
- **INCLUDE column** — хранится в leaf-страницах индекса, не участвует в tree structure,
  нет ограничений по типам (можно даже expressions, JSONB без операторного класса).

**Когда использовать:**
- Запрос возвращает небольшое число колонок;
- Чтение из индекса значительно быстрее (избегаем random I/O heap fetch);
- Таблица большая, индекс помещается в кэш.

</details>

---

### Вопрос №4: [ Когда индекс НЕ будет использоваться? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Случаи, когда индекс не используется (даже если он есть):**

1. **Функция на колонке (без expression index):**
   ```sql
   -- Индекс: CREATE INDEX idx ON users(email);
   SELECT * FROM users WHERE LOWER(email) = 'a@b.c';  -- НЕ использует индекс!
   -- Нужно: CREATE INDEX idx ON users(LOWER(email));
   ```

2. **Несовпадение типов (implicit cast):**
   ```sql
   -- Индекс: CREATE INDEX idx ON orders(code);  -- code VARCHAR
   SELECT * FROM orders WHERE code = 123;  -- 123 INT → implicit cast → НЕ использует индекс
   SELECT * FROM orders WHERE code = '123';  -- использует
   ```

3. **LIKE с wildcard в начале:**
   ```sql
   SELECT * FROM users WHERE name LIKE '%ivan';  -- НЕ использует B-Tree индекс
   SELECT * FROM users WHERE name LIKE 'ivan%';  -- использует
   -- Решение: pg_trgm extension + GIN index
   ```

4. **OR без индексов на обе стороны:**
   ```sql
   SELECT * FROM users WHERE email = 'a@b.c' OR unknown_col = 'x';
   -- Если на unknown_col нет индекса → может выбрать Seq Scan
   ```

5. **Планировщик решает, что Seq Scan дешевле:**
   - Возвращается > ~5–20% строк таблицы;
   - Таблица маленькая;
   - Статистика устарела (нужен ANALYZE).

6. **Низкая селективность:**
   ```sql
   WHERE gender = 'male'  -- 50% строк → индекс не имеет смысла
   ```

7. **ORDER BY с другим направлением:**
   ```sql
   -- Индекс: CREATE INDEX idx ON users(created_at);
   SELECT * FROM users ORDER BY created_at DESC;
   -- Использует (PostgreSQL умеет обратный scan B-Tree)
   -- Но: CREATE INDEX idx ON users(a ASC, b DESC);
   -- ORDER BY a DESC, b ASC — не использует (направления не совпадают)
   ```

</details>

---

### Вопрос №5: [ Что такое bloat (раздувание) индексов и таблиц? Как бороться? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**Bloat** — неиспользуемое пространство внутри таблиц/индексов, образующееся из-за MVCC.
При UPDATE/DELETE старые версии строк (dead tuples) помечаются как свободные, но место
не возвращается ОС (только VACUUM FULL или REINDEX).

**Причины bloat:**
- Длительные транзакции (мешают autovacuum очищать dead tuples);
- Insufficient autovacuum settings (слишком редкий);
- Массовые UPDATE без последующего vacuum;
- Index-only scans деградируют (visibility map не all-visible).

**Как измерить:**

```sql
-- Bloat таблиц (приблизительная оценка):
SELECT
    schemaname, relname,
    n_live_tup,
    n_dead_tup,
    ROUND(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_live_tup > 1000
ORDER BY dead_pct DESC;

-- Точный размер bloat через pgstattuple (расширение):
CREATE EXTENSION pgstattuple;
SELECT * FROM pgstattuple('my_table');
-- table_len, tuple_len, dead_tuple_len, free_space

-- Bloat индексов:
SELECT * FROM pgstatindex('my_index');
```

**Борьба:**
1. **Настройка autovacuum** — более агрессивные параметры для горячих таблиц:
   ```sql
   ALTER TABLE my_table SET (autovacuum_vacuum_scale_factor = 0.05);
   ALTER TABLE my_table SET (autovacuum_analyze_scale_factor = 0.02);
   ```
2. **REINDEX** — перестроение индекса (CONCURRENTLY — без блокировки):
   ```sql
   REINDEX INDEX CONCURRENTLY my_index;
   ```
3. **pg_repack** — перестроение таблицы без блокировки:
   ```sql
   SELECT repack_table('my_table');
   ```
4. **VACUUM FULL** — перестроение таблицы (блокирует!):
   ```sql
   VACUUM FULL my_table;
   ```

</details>

---

### Вопрос №6: [ Что такое partial index и expression index? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Partial index** — индекс на подмножество строк (WHERE условие):
```sql
-- Только активные пользователи (если 90% неактивных — индекс в 10 раз меньше)
CREATE INDEX idx_active_emails ON users(email) WHERE status = 'active';

-- Только непрочитанные сообщения
CREATE INDEX idx_unread ON messages(user_id) WHERE read_at IS NULL;
```
Плюс: меньше размер, быстрее сканирование, меньше overhead на запись.
Минус: используется только если запрос содержит то же условие.

**Expression index** — индекс по результату функции/выражения:
```sql
-- Case-insensitive поиск
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'a@b.c';  -- использует индекс

-- Индекс по длине массива
CREATE INDEX idx_tags_count ON posts(ARRAY_LENGTH(tags, 1));

-- Индекс по JSONB полю
CREATE INDEX idx_data_user_id ON events((data->>'user_id'));
SELECT * FROM events WHERE data->>'user_id' = '123';
```

**Важно:** запрос должен использовать **то же выражение** для использования индекса.

</details>

---

</details>

---

## 3. Транзакции и MVCC

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Объясни уровни изоляции в PostgreSQL. Какие аномалии возможны? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

PostgreSQL реализует 3 из 4 стандартных уровня (Read Uncommitted = Read Committed):

| Уровень | Dirty Read | Non-Repeatable Read | Phantom Read | Serialization Anomaly |
|---|---|---|---|---|
| Read Uncommitted* | Невозможно | Возможно | Возможно | Возможно |
| Read Committed (default) | Невозможно | Возможно | Возможно | Возможно |
| Repeatable Read | Невозможно | Невозможно | Невозможно* | Возможно |
| Serializable | Невозможно | Невозможно | Невозможно | Невозможно |

\* PostgreSQL: Read Uncommitted ведёт себя как Read Committed.
\* PostgreSQL: Repeatable Read = Snapshot Isolation — фантомы **невозможны** (строже стандарта SQL).

**Read Committed (default):**
- Каждый оператор видит данные, закоммиченные на момент его начала;
- Non-repeatable read: в одной транзакции два SELECT могут вернуть разные данные;
- Подходит для большинства случаев.

**Repeatable Read (Snapshot Isolation):**
- Один snapshot на всю транзакцию (первый SELECT);
- Write skew возможен (две транзакции читают пересекающиеся данные и пишут разные строки);
- При конфликте записи (одна строка изменена другой транзакцией) → ошибка:
  `ERROR: could not serialize access due to concurrent update`.

**Serializable:**
- Serializable Snapshot Isolation (SSI) — отслеживает зависимости между транзакциями;
- При обнаружении hazardous pattern → `ERROR: could not serialize access due to read/write
  dependencies among transactions`;
- Приложение должно быть готово к retry при serialization failure (SQLSTATE 40001).

</details>

---

### Вопрос №2: [ Что такое advisory locks? Виды и применение ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**Advisory locks** — приложение-уровневые блокировки, предоставляемые PostgreSQL. Не связаны
с конкретными строками или таблицами — идентифицируются произвольным ключом (bigint или 2 int).

**Виды:**

```sql
-- Transaction-level (освобождаются при COMMIT/ROLLBACK)
pg_advisory_xact_lock(key)           -- блокирует и ждёт
pg_try_advisory_xact_lock(key)       -- пытается, возвращает bool

-- Session-level (освобождаются при закрытии соединения)
pg_advisory_lock(key)                -- блокирует и ждёт
pg_try_advisory_lock(key)            -- пытается, возвращает bool
pg_advisory_unlock(key)              -- явное освобождение

-- С двумя ключами (int4, int4) — удобно для namespace
pg_advisory_lock(class_id, key_id)
```

**Применения:**

1. **Распределённая блокировка (coordination):**
```sql
-- Только один воркер выполняет задачу
BEGIN;
SELECT pg_advisory_xact_lock(42);  -- lock id=42
-- ... критическая секция ...
COMMIT;  -- lock освобождён
```

2. **Ограничение уникальности, которое нельзя выразить через constraint:**
```sql
-- Например, "не более одного активного заказа на пользователя"
-- (условная уникальность)
SELECT pg_advisory_xact_lock(user_id);
INSERT INTO orders (...) VALUES (...);
-- Другая транзакция для того же user_id будет ждать
```

3. **Leader election / cron job guard:**
```sql
-- Только одна нода выполняет cron-задачу
SELECT pg_try_advisory_lock(hashtext('daily_report'));
-- true → выполняем, false → другая нода уже делает
```

**Преимущества перед row-level locks:**
- Не требуют реальной строки в таблице;
- Transaction-level автоматически освобождаются (нет утечки);
- Очень быстрые (в памяти).

**Осторожно:** Session-level locks могут протекать если соединение переиспользуется
(connection pool) — всегда явно unlock или используйте xact-level.

</details>

---

### Вопрос №3: [ Что такое SELECT FOR UPDATE? Виды и поведение ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

`SELECT ... FOR UPDATE` — пессимистичная блокировка строк на чтение+запись.

**Виды:**

```sql
-- FOR UPDATE — блокирует строки от изменения другими транзакциями
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

-- FOR NO KEY UPDATE — слабее: блокирует от UPDATE, но не от UPDATE ключа
-- (другие транзакции могут UPDATE неключевых колонок)
SELECT * FROM accounts WHERE id = 1 FOR NO KEY UPDATE;

-- FOR SHARE — разделяемая блокировка: другие могут FOR SHARE, но не UPDATE/DELETE
SELECT * FROM accounts WHERE id = 1 FOR SHARE;

-- FOR KEY SHARE — слабейший: другие не могут UPDATE ключевых колонок или DELETE
SELECT * FROM accounts WHERE id = 1 FOR KEY SHARE;
```

**Опции конфликта:**

```sql
-- NOWAIT — не ждать, сразу ошибка если заблокировано
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;

-- SKIP LOCKED — пропустить заблокированные строки (не ошибка)
SELECT * FROM queue WHERE status = 'pending' FOR UPDATE SKIP LOCKED LIMIT 10;
-- Идеально для job queue!

-- WAIT — ждать (по умолчанию)
```

**Применение SKIP LOCKED (job queue pattern):**
```sql
-- Атомарно взять 10 задач из очереди без блокировки других воркеров
BEGIN;
SELECT id, payload FROM job_queue
WHERE status = 'pending'
ORDER BY created_at
FOR UPDATE SKIP LOCKED
LIMIT 10;

UPDATE job_queue SET status = 'processing', worker_id = $1
WHERE id IN (...);
COMMIT;
```

</details>

---

### Вопрос №4: [ Что такое deadlock? Как избежать? Как PostgreSQL его обнаруживает? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Deadlock** — ситуация, когда две или более транзакций ждут блокировки, удерживаемые друг
другом. Ни одна не может продолжить.

```
T1: LOCK(A) → ждёт LOCK(B)
T2: LOCK(B) → ждёт LOCK(A)
→ Deadlock!
```

**Обнаружение в PostgreSQL:**
- Background-процесс **deadlock detector** запускается по таймеру (`deadlock_timeout`,
  default 1s);
- Строит граф ожидания (wait-for graph);
- Если найден цикл → одна транзакция получает `ERROR: deadlock detected` и откатывается
  (жертва выбирается произвольно);
- Другая транзакция продолжает.

**Как избежать:**

1. **Фиксированный порядок блокировок** — всегда блокируйте ресурсы в одном порядке:
```sql
-- Все транзакции блокируют accounts по возрастанию id
SELECT * FROM accounts WHERE id IN (1, 2, 3) ORDER BY id FOR UPDATE;
```

2. **Короткие транзакции** — меньше время удержания блокировок;

3. **Минимальные блокировки** — не блокируйте то, что не нужно;

4. **Advisory locks для координации** — вместо FOR UPDATE на нескольких таблицах;

5. **Использовать SKIP LOCKED / NOWAIT** — не ждать бесконечно;

6. **Оптимистичная блокировка** (version column) вместо пессимистичной:
```sql
UPDATE products SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 5;
-- Если 0 rows affected → кто-то изменил раньше → retry
```

7. **Установка lock_timeout:**
```sql
SET lock_timeout = '5s';
-- Если блокировка не получена за 5с → ERROR (а не бесконечное ожидание)
```

</details>

---

### Вопрос №5: [ Что такое SAVEPOINT? Как работает nested transaction? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**SAVEPOINT** — точка внутри транзакции, к которой можно откатиться, не прерывая всю транзакцию.
Не является настоящей вложенной транзакцией (это точка отката), но даёт подобный эффект.

```sql
BEGIN;

INSERT INTO orders (...) VALUES (...);
SAVEPOINT sp1;

INSERT INTO order_items (...) VALUES (...);
-- Ошибка! (например, constraint violation)

ROLLBACK TO sp1;  -- Откат к точке sp1, ORDER остаётся

-- Попробуем иначе
INSERT INTO order_items (...) VALUES (...);

COMMIT;
```

**Поведение при ошибке:**
- В PostgreSQL после любой ошибки транзакция переходит в **aborted state** — все последующие
  операторы дают `ERROR: current transaction is aborted`;
- `ROLLBACK TO savepoint` выводит транзакцию из aborted state → можно продолжать;
- Поэтому SAVEPOINT критичен для обработки ошибок внутри транзакции.

**Паттерн "batch with error handling":**
```sql
BEGIN;
FOREACH item IN batch:
    SAVEPOINT sp;
    BEGIN;
        INSERT INTO items VALUES (item);
    EXCEPTION WHEN others THEN
        ROLLBACK TO sp;
        -- логируем ошибку, продолжаем
    END;
COMMIT;
```

</details>

---

</details>

---

## 4. Оптимизация запросов

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Как читать EXPLAIN (ANALYZE)? Что значат cost, rows, actual time? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE email = 'a@b.c';
```

```
Index Scan using users_email_idx on users  (cost=0.29..8.31 rows=1 width=148) (actual time=0.023..0.025 rows=1 loops=1)
  Index Cond: (email = 'a@b.c'::text)
  Buffers: shared hit=4
Planning Time: 0.083 ms
Execution Time: 0.041 ms
```

**Поля:**

- **cost = startup..total** — оценочная стоимость (условные единицы, ~произвольные):
  - startup — до первой строки;
  - total — полная стоимость;
  - `seq_page_cost = 1.0` — стоимость последовательного чтения страницы;
  - `random_page_cost = 4.0` — стоимость случайного чтения (SSD: снизить до 1.1–2.0).

- **rows** — оценочное количество строк. **Сравнивай с actual rows!**
  - Если estimate сильно отличается от actual → проблема со статистикой (нужен ANALYZE
    или ручная настройка).

- **width** — средний размер строки в байтах.

- **actual time = first..last** (только с ANALYZE) — реальное время в миллисекундах:
  - first — до первой строки;
  - last — до последней строки.

- **loops** — сколько раз узел выполнялся (actual time × loops = суммарное время).

- **Buffers:**
  - shared hit — страниц прочитано из кэша (shared buffers);
  - shared read — страниц прочитано с диска;
  - shared dirtied / written — изменено/записано.
  - **hit=100%** — всё из кэша (хорошо).

**Что искать:**
1. **seq scan на большой таблице** без фильтра → нужен индекс;
2. **estimate vs actual rows сильно расходится** → ANALYZE или проблема со статистикой;
3. **nested loop с большим actual rows на outer** → O(N×M), возможно нужен hash join;
4. **sort с большим actual time** → нужен индекс для ORDER BY;
5. **high shared read** → много дискового I/O → может нужен индекс или больше shared_buffers.

</details>

---

### Вопрос №2: [ Что такое N+1 проблема? Как исправить? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**N+1** — когда вместо одного запроса с JOIN выполняется 1 запрос на родительские записи + N
запросов на дочерние (по одному на каждую родительскую).

**Пример (плохо):**
```sql
-- 1 запрос: получить 100 заказов
SELECT * FROM orders LIMIT 100;
-- Затем в цикле для каждого заказа:
SELECT * FROM users WHERE id = ?;  -- × 100 раз
-- Итого: 1 + 100 = 101 запрос
```

**Решение 1: JOIN**
```sql
SELECT o.*, u.name
FROM orders o
JOIN users u ON u.id = o.user_id
LIMIT 100;
```

**Решение 2: Batch loading (IN)**
```sql
-- Шаг 1: получить заказы
SELECT * FROM orders LIMIT 100;
-- Шаг 2: получить всех пользователей одним запросом
SELECT * FROM users WHERE id IN (1, 2, 3, ..., 100);
-- Итого: 2 запроса
```

**Решение 3: LATERAL JOIN (PostgreSQL)**
```sql
SELECT o.*, u_data.*
FROM orders o
LEFT JOIN LATERAL (
    SELECT name, email FROM users WHERE id = o.user_id
) u_data ON true
LIMIT 100;
```

**На уровне приложения (ORM):**
- Go (GORM): `Preload("User")` или `Joins`
- Python (SQLAlchemy): `selectinload(User.orders)` или `joinedload`
- Hibernate (Java): `@Fetch(FetchMode.JOIN)` или `@EntityGraph`

</details>

---

### Вопрос №3: [ Как оптимизировать медленный запрос? Пошаговый алгоритм ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**Шаг 1: EXPLAIN ANALYZE**
```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT ...;
```
Смотрим:
- Где больше всего actual time;
- Seq Scan на большой таблице (нужен индекс?);
- Estimate vs actual rows (статистика врёт?);
- Buffers: read (с диска) vs hit (из кэша).

**Шаг 2: Проверить индексы**
- Есть ли индекс по колонкам из WHERE / JOIN / ORDER BY?
- Используется ли индекс? Если нет — почему? (см. вопрос "Когда индекс не используется");
- Поможет ли covering index (INCLUDE)?

**Шаг 3: Проверить статистику**
```sql
ANALYZE my_table;  -- обновить статистику
-- Проверить:
SELECT * FROM pg_stats WHERE tablename = 'my_table';
```

**Шаг 4: Переписать запрос**
- Заменить subquery на JOIN (или наоборот);
- Избегать `SELECT *` — выбирать только нужные колонки;
- Разбить сложный запрос на этапы (CTE, temp table);
- Использовать EXISTS вместо IN для больших списков;
- LIMIT + late row lookup:
```sql
-- Плохо: JOIN на больших таблицах с LIMIT
SELECT * FROM orders o JOIN users u ON ... ORDER BY o.created_at LIMIT 10;

-- Хорошо: сначала найти 10 заказов, потом join
SELECT * FROM (
    SELECT * FROM orders ORDER BY created_at LIMIT 10
) o JOIN users u ON ...;
```

**Шаг 5: Настройки planner'а**
```sql
-- Увеличить work_mem для больших сортировок/хэшей
SET work_mem = '64MB';
-- Включить parallel query
SET max_parallel_workers_per_gather = 4;
```

**Шаг 6: Архитектурные изменения**
- Материализованные представления для тяжёлых агрегаций;
- Партиционирование для больших таблиц;
- Денормализация (pre-computed counters);
- Read replica для аналитики.

</details>

---

### Вопрос №4: [ Что такое materialized view? Чем отличается от обычного view? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

| | View | Materialized View |
|---|---|---|
| Хранение | Только определение запроса | Результат запроса (данные) |
| Актуальность | Всегда свежие | Обновляются при REFRESH |
| Скорость чтения | Зависит от запроса | Быстро (чтение физической таблицы) |
| Запись | Нет (read-only) | Нет (read-only, можно через INSTEAD OF rules) |
| Индексы | Нет | Можно создавать |

```sql
-- Обычный view — пересчитывается каждый раз
CREATE VIEW daily_stats AS
SELECT date, COUNT(*) FROM events GROUP BY date;

-- Materialized view — хранит результат
CREATE MATERIALIZED VIEW daily_stats_mv AS
SELECT date, COUNT(*) FROM events GROUP BY date;

-- Индекс на materialized view
CREATE INDEX idx_daily_stats_date ON daily_stats_mv(date);

-- Обновление
REFRESH MATERIALIZED VIEW daily_stats_mv;
-- Без блокировки (CONCURRENTLY — требует unique index):
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_stats_mv;
```

**Когда использовать:**
- Тяжёлые агрегации, которые не нужны в real-time;
- Отчёты, дашборды;
- Денормализация для чтения.

**Подводные камни:**
- `REFRESH MATERIALIZED VIEW` блокирует чтение (без CONCURRENTLY);
- `CONCURRENTLY` требует unique index и не может быть внутри транзакции;
- Нужно планировать refresh (cron, pg_cron, triggers).

</details>

---

### Вопрос №5: [ Как работают LATERAL JOIN и WHEN (window functions в деталях)? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**LATERAL JOIN** — позволяет подзапросу ссылаться на колонки из левой таблицы (как correlated
subquery, но в JOIN):

```sql
-- Найти 3 последних заказа для каждого пользователя
SELECT u.name, recent_orders.*
FROM users u
LEFT JOIN LATERAL (
    SELECT * FROM orders
    WHERE orders.user_id = u.id
    ORDER BY created_at DESC
    LIMIT 3
) recent_orders ON true;
```
Без LATERAL пришлось бы использовать window functions или несколько запросов.

**Оконные функции (window functions):**

```sql
-- ROW_NUMBER: номер строки в партии
SELECT name, salary,
       ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
FROM employees;

-- RANK vs DENSE_RANK:
-- RANK: 1, 2, 2, 4 (gap after ties)
-- DENSE_RANK: 1, 2, 2, 3 (no gap)
SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC) AS rank,
       DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;

-- LAG / LEAD: доступ к предыдущей/следующей строке
SELECT date, revenue,
       revenue - LAG(revenue) OVER (ORDER BY date) AS daily_growth,
       revenue - LAG(revenue, 7) OVER (ORDER BY date) AS week_over_week
FROM daily_sales;

-- Running total (кумулятивная сумма)
SELECT date, revenue,
       SUM(revenue) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative
FROM daily_sales;

-- MOVING AVERAGE (скользящее среднее за 7 дней)
SELECT date, revenue,
       AVG(revenue) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma_7
FROM daily_sales;

-- FIRST_VALUE / LAST_VALUE / NTH_VALUE
SELECT name, salary,
       FIRST_VALUE(name) OVER (PARTITION BY dept ORDER BY salary DESC) AS top_earner
FROM employees;
```

**PARTITION BY** — определяет партию (группу).
**ORDER BY** — сортировка внутри партии.
**Frame clause** — диапазон строк для вычисления:
- `ROWS BETWEEN` — по физическому положению;
- `RANGE BETWEEN` — по значению;
- `UNBOUNDED PRECEDING/FOLLOWING` — от начала/до конца партии.

</details>

---

</details>

---

## 5. Практические задачи (Middle)

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Задача №1: [ Найти дубликаты в таблице ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Условие</summary>

Есть таблица `users(id, email, created_at)`. Найти все email, которые встречаются более
одного раза, и количество повторений.

</details>

<details>
<summary>Решение</summary>

```sql
SELECT email, COUNT(*) AS cnt
FROM users
GROUP BY email
HAVING COUNT(*) > 1
ORDER BY cnt DESC;
```

**Удаление дубликатов (оставить только последнего по created_at):**

```sql
DELETE FROM users
WHERE id NOT IN (
    SELECT DISTINCT ON (email) id
    FROM users
    ORDER BY email, created_at DESC
);
```

`DISTINCT ON (email)` — PostgreSQL-специфичная конструкция: возвращает первую строку для
каждого уникального значения email (с учётом ORDER BY).

</details>

---

### Задача №2: [ Найти N-ый по величине элемент ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Условие</summary>

Найти 3-ю по величине зарплату среди всех сотрудников.
Таблица: `employees(id, name, salary)`.

</details>

<details>
<summary>Решение</summary>

**Вариант 1: DENSE_RANK (корректно с дубликатами зарплат)**
```sql
SELECT salary
FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) t
WHERE rnk = 3;
```

**Вариант 2: OFFSET/FETCH (проще, но дубликаты считаются отдельно)**
```sql
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
OFFSET 2 LIMIT 1;  -- пропускаем 2, берём 1 = 3-й
```

**Разница:**
- Если зарплаты: 100, 100, 90, 80, 70
- DENSE_RANK: 1=100, 2=90, 3=80
- OFFSET (с DISTINCT): 1=100, 2=90, 3=80 (та же логика)
- Без DISTINCT OFFSET: пропустит 2 строки (100, 100) → вернёт 90 (2-я уникальная)

</details>

---

### Задача №3: [ Конвертация строк в колонки (pivot) ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Условие</summary>

Таблица `sales(product, month, revenue)`:
```
A | Jan | 100
A | Feb | 150
B | Jan | 200
B | Feb | 250
```

Получить:
```
product | Jan | Feb
A       | 100 | 150
B       | 200 | 250
```

</details>

<details>
<summary>Решение</summary>

**Вариант 1: FILTER (PostgreSQL-специфичный, рекомендуемый)**
```sql
SELECT
    product,
    SUM(revenue) FILTER (WHERE month = 'Jan') AS "Jan",
    SUM(revenue) FILTER (WHERE month = 'Feb') AS "Feb",
    SUM(revenue) FILTER (WHERE month = 'Mar') AS "Mar"
FROM sales
GROUP BY product
ORDER BY product;
```

**Вариант 2: CASE (стандартный SQL)**
```sql
SELECT
    product,
    SUM(CASE WHEN month = 'Jan' THEN revenue ELSE 0 END) AS "Jan",
    SUM(CASE WHEN month = 'Feb' THEN revenue ELSE 0 END) AS "Feb"
FROM sales
GROUP BY product
ORDER BY product;
```

**Вариант 3: crosstab (расширение tablefunc)**
```sql
CREATE EXTENSION tablefunc;

SELECT *
FROM crosstab(
    'SELECT product, month, revenue FROM sales ORDER BY product, month'
) AS ct(product TEXT, "Jan" INT, "Feb" INT);
```

</details>

---

### Задача №4: [ Найти пропущенные значения в последовательности ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Условие</summary>

Таблица `entries(id)` содержит не все целые числа от 1 до N. Найти пропущенные (gaps).
Например: 1, 2, 5, 6, 7, 10 → пропущены 3, 4, 8, 9.

</details>

<details>
<summary>Решение</summary>

**Вариант 1: generate_series + LEFT JOIN**
```sql
SELECT s.num AS missing
FROM generate_series(1, (SELECT MAX(id) FROM entries)) AS s(num)
LEFT JOIN entries e ON e.id = s.num
WHERE e.id IS NULL;
```

**Вариант 2: LAG (найти границы gap'ов)**
```sql
SELECT prev_id + 1 AS gap_start, curr_id - 1 AS gap_end
FROM (
    SELECT id AS curr_id, LAG(id) OVER (ORDER BY id) AS prev_id
    FROM entries
) t
WHERE curr_id - prev_id > 1;
```

**Вариант 3: Раскрыть диапазоны gap'ов в отдельные числа**
```sql
SELECT generate_series(gap_start, gap_end) AS missing
FROM (
    SELECT prev_id + 1 AS gap_start, curr_id - 1 AS gap_end
    FROM (
        SELECT id AS curr_id, LAG(id) OVER (ORDER BY id) AS prev_id
        FROM entries
    ) t
    WHERE curr_id - prev_id > 1
) gaps;
```

</details>

---

### Задача №5: [ Накопительный итог и процент от общего ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Условие</summary>

Таблица `daily_sales(date, revenue)`. Для каждой даты вывести:
- revenue за день;
- накопительный итог с начала периода;
- процент от общей суммы за период.

</details>

<details>
<summary>Решение</summary>

```sql
SELECT
    date,
    revenue,
    SUM(revenue) OVER (ORDER BY date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative,
    ROUND(
        revenue * 100.0 / SUM(revenue) OVER (),
        2
    ) AS pct_of_total
FROM daily_sales
ORDER BY date;
```

**Объяснение:**
- `SUM(...) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` —
  кумулятивная сумма от первой строки до текущей;
- `SUM(revenue) OVER ()` — сумма по всем строкам (без partition/order = весь набор).

</details>

---

### Задача №6: [ Топ-N в каждой группе ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Условие</summary>

Таблица `orders(id, user_id, amount, created_at)`. Для каждого пользователя найти
3 самых дорогих заказа.

</details>

<details>
<summary>Решение</summary>

**Вариант 1: ROW_NUMBER (классический)**
```sql
SELECT user_id, id, amount, created_at
FROM (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rn
    FROM orders
) t
WHERE rn <= 3
ORDER BY user_id, rn;
```

**Вариант 2: LATERAL JOIN (быстрее при наличии индекса на (user_id, amount DESC))**
```sql
SELECT u.user_id, o.id, o.amount, o.created_at
FROM (SELECT DISTINCT user_id FROM orders) u
CROSS JOIN LATERAL (
    SELECT id, amount, created_at
    FROM orders
    WHERE user_id = u.user_id
    ORDER BY amount DESC
    LIMIT 3
) o
ORDER BY u.user_id, o.amount DESC;
```

**Вариант 3: ARRAY_AGG + unnest (PostgreSQL-специфичный)**
```sql
SELECT
    user_id,
    (ARRAY_AGG(amount ORDER BY amount DESC))[1:3] AS top3
FROM orders
GROUP BY user_id;
```

</details>

---

### Задача №7: [ Объединение пересекающихся интервалов ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Условие</summary>

Таблица `bookings(room_id, start_time, end_time)`. Объединить пересекающиеся интервалы
бронирования для каждой комнаты.

```
room 1 | 10:00-11:00
room 1 | 10:30-12:00  ← пересекается с предыдущим
room 1 | 13:00-14:00  ← не пересекается
```

Результат:
```
room 1 | 10:00-12:00
room 1 | 13:00-14:00
```

</details>

<details>
<summary>Решение</summary>

Классический **gaps and islands** подход:

```sql
SELECT
    room_id,
    MIN(start_time) AS merged_start,
    MAX(end_time) AS merged_end
FROM (
    SELECT *,
        -- Группа (island) = cumulative count of gaps
        SUM(is_gap) OVER (PARTITION BY room_id ORDER BY start_time) AS grp
    FROM (
        SELECT *,
            -- Если start_time > prev_end_time → это новый интервал (gap = 1)
            CASE
                WHEN start_time > LAG(end_time) OVER (
                    PARTITION BY room_id ORDER BY start_time
                )
                THEN 1
                ELSE 0
            END AS is_gap
        FROM bookings
    ) b1
) b2
GROUP BY room_id, grp
ORDER BY room_id, merged_start;
```

**Логика:**
1. Для каждой строки смотрим, начинается ли она после конца предыдущей (gap);
2. Кумулятивная сумма gap'ов → номер группы (island);
3. GROUP BY по группе → MIN(start), MAX(end) = объединённый интервал.

</details>

---

</details>

---

## 6. Практические задачи (Senior)

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Задача №1: [ Реализовать конкурентный счётчик (account balance) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Условие</summary>

Таблица `accounts(id, balance)`. Много параллельных транзакций списывают средства.
Обеспечить: (1) баланс не уходит в минус, (2) минимальные блокировки, (3) отсутствие lost update.

</details>

<details>
<summary>Решение</summary>

**Плохо (блокирует всю строку, deadlock-prone):**
```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- приложение проверяет balance >= amount
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

**Вариант 1: Atomic UPDATE с условием (рекомендуется)**
```sql
UPDATE accounts
SET balance = balance - 100
WHERE id = 1 AND balance >= 100
RETURNING balance;
-- Если вернулась строка → успешно
-- Если 0 строк → недостаточно средств (или аккаунт не найден)
```
Одна атомарная операция, без отдельного SELECT. Блокировка строки минимальное время.

**Вариант 2: Оптимистичная блокировка (version)**
```sql
-- Таблица: accounts(id, balance, version)
UPDATE accounts
SET balance = balance - 100, version = version + 1
WHERE id = 1 AND version = $current_version AND balance >= 100
RETURNING balance;
-- 0 rows → retry (кто-то изменил раньше)
```

**Вариант 3: Append-only (event sourcing)**
```sql
-- Вместо UPDATE — INSERT в таблицу операций
INSERT INTO account_transactions (account_id, amount) VALUES (1, -100);

-- Баланс = сумма операций
SELECT SUM(amount) FROM account_transactions WHERE account_id = 1;
-- Или материализованный view / кэш для актуального баланса
```
Плюс: полный audit log, нет блокировок. Минус: нужно агрегировать.

**Защита от гонки при transfer (A → B):**
```sql
-- Всегда блокировать в порядке возрастания id → нет deadlock
SELECT id FROM accounts WHERE id IN ($from, $to) ORDER BY id FOR UPDATE;
-- Затем:
UPDATE accounts SET balance = balance - $amount WHERE id = $from;
UPDATE accounts SET balance = balance + $amount WHERE id = $to;
```

</details>

---

### Задача №2: [ Построить job queue на PostgreSQL ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Условие</summary>

Реализовать очередь задач на PostgreSQL. Требования:
- Множество воркеров забирают задачи параллельно;
- Задача не может быть взята двумя воркерами;
- Если воркер упал — задача должна быть доступна повторно (после timeout);
- Приоритеты задач.

</details>

<details>
<summary>Решение</summary>

**Схема:**
```sql
CREATE TABLE jobs (
    id BIGSERIAL PRIMARY KEY,
    payload JSONB NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
        -- pending → processing → done / failed
    priority INT NOT NULL DEFAULT 0,
    worker_id TEXT,
    started_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jobs_claim
    ON jobs(priority DESC, created_at)
    WHERE status = 'pending';
```

**Захват задачи (атомарный, без блокировок других задач):**
```sql
-- Воркер забирает одну задачу
WITH next_job AS (
    SELECT id FROM jobs
    WHERE status = 'pending'
      AND (worker_id IS NULL OR started_at < now() - INTERVAL '5 minutes')
    ORDER BY priority DESC, created_at
    FOR UPDATE SKIP LOCKED
    LIMIT 1
)
UPDATE jobs
SET status = 'processing',
    worker_id = $worker_id,
    started_at = now()
FROM next_job
WHERE jobs.id = next_job.id
RETURNING jobs.*;
```

**Ключевые моменты:**
- `FOR UPDATE SKIP LOCKED` — если задача уже захвачена другим воркером, пропускаем (не ждём);
- `LIMIT 1` — берём одну;
- CTE + UPDATE в одном операторе → атомарность;
- Timeout: `started_at < now() - INTERVAL '5 minutes'` — если воркер завис, задача доступна.

**Завершение:**
```sql
UPDATE jobs SET status = 'done' WHERE id = $1 AND worker_id = $worker_id;
-- Проверка: если 0 rows → кто-то перехватил задачу (timeout истёк)
```

**Cleanup (запускать периодически):**
```sql
-- Вернуть зависшие задачи в очередь
UPDATE jobs
SET status = 'pending', worker_id = NULL, started_at = NULL
WHERE status = 'processing'
  AND started_at < now() - INTERVAL '10 minutes';
```

**Плюсы PostgreSQL-очереди:**
- Транзакционность (задача + бизнес-данные в одной транзакции);
- Нет отдельной инфраструктуры (Redis/RabbitMQ);
- Простота.

**Минусы:**
- Polling (воркеры периодически опрашивают) → I/O;
- LISTEN/NOTIFY можно использовать для уведомления, но не для high-throughput;
- При > 1000 RPS задач → лучше специализированная очередь (Redis Streams, Kafka).

</details>

---

### Задача №3: [ Иерархический запрос: найти всех потомков и уровень вложенности ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Условие</summary>

Таблица `categories(id, name, parent_id)`. Дерево категорий.
Найти всех потомков категории с id=5, включая уровень вложенности и полный путь.

</details>

<details>
<summary>Решение</summary>

```sql
WITH RECURSIVE descendants AS (
    -- Anchor: прямые потомки
    SELECT
        id,
        name,
        parent_id,
        1 AS level,
        name::TEXT AS path,
        ARRAY[id] AS id_path
    FROM categories
    WHERE parent_id = 5

    UNION ALL

    -- Recursive: потомки потомков
    SELECT
        c.id,
        c.name,
        c.parent_id,
        d.level + 1,
        d.path || ' > ' || c.name,
        d.id_path || c.id
    FROM categories c
    INNER JOIN descendants d ON c.parent_id = d.id
)
SELECT id, name, level, path, id_path
FROM descendants
ORDER BY id_path;
-- LIMIT: защита от бесконечной рекурсии
-- WHERE level <= 10
```

**Для произвольного корня (включая саму категорию):**
```sql
WITH RECURSIVE descendants AS (
    SELECT id, name, parent_id, 0 AS level, name::TEXT AS path, ARRAY[id] AS id_path
    FROM categories WHERE id = 5
    UNION ALL
    SELECT c.id, c.name, c.parent_id, d.level + 1, d.path || ' > ' || c.name, d.id_path || c.id
    FROM categories c JOIN descendants d ON c.parent_id = d.id
)
SELECT * FROM descendants ORDER BY id_path;
```

**Альтернатива: ltree (расширение PostgreSQL для деревьев)**
```sql
CREATE EXTENSION ltree;

ALTER TABLE categories ADD COLUMN path ltree;
-- path = '5.12.34.56' (путь от корня)

CREATE INDEX idx_categories_path ON categories USING GIST (path);

-- Все потомки категории 5:
SELECT * FROM categories WHERE path <@ '5';

-- Прямые потомки:
SELECT * FROM categories WHERE path ~ '5.*{1}';

-- Глубина:
SELECT nlevel(path) - nlevel('5') AS depth FROM categories WHERE path <@ '5';
```

</details>

---

### Задача №4: [ Time-series: сжатие и агрегация больших объёмов ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Условие</summary>

Таблица `metrics(timestamp, device_id, value)` растёт на 10M строк/день.
Нужно: (1) быстрый запрос агрегаций по часам/дням, (2) сжатие старых данных.

</details>

<details>
<summary>Решение</summary>

**1. Партиционирование по времени:**
```sql
CREATE TABLE metrics (
    timestamp TIMESTAMPTZ NOT NULL,
    device_id INT NOT NULL,
    value DOUBLE PRECISION NOT NULL
) PARTITION BY RANGE (timestamp);

-- Ежедневные партиции (создаются автоматически через pg_partman или cron)
CREATE TABLE metrics_2024_01_01 PARTITION OF metrics
    FOR VALUES FROM ('2024-01-01') TO ('2024-01-02');
```

**2. BRIN index (минимальный размер для time-series):**
```sql
CREATE INDEX idx_metrics_ts_brin ON metrics USING BRIN (timestamp);
-- BRIN хранит только min/max на блок страниц → ~1000× меньше B-Tree
```

**3. Continuous Aggregates (TimescaleDB) или Materialized Views:**
```sql
-- Стандартный PostgreSQL: материализованное представление
CREATE MATERIALIZED VIEW metrics_hourly AS
SELECT
    date_trunc('hour', timestamp) AS hour,
    device_id,
    AVG(value) AS avg_value,
    MIN(value) AS min_value,
    MAX(value) AS max_value,
    COUNT(*) AS cnt
FROM metrics
GROUP BY 1, 2;

CREATE UNIQUE INDEX ON metrics_hourly(hour, device_id);

-- Обновление (инкрементальное — только новые данные):
REFRESH MATERIALIZED VIEW CONCURRENTLY metrics_hourly;
```

**4. Сжатие старых партиций (через pg_compress или TimescaleDB):**
```sql
-- TimescaleDB native compression:
-- ALTER TABLE metrics SET (timescaledb.compress);
-- SELECT compress_chunk(i) FROM show_chunks('metrics', older_than => '7 days'::interval) i;

-- Стандартный PostgreSQL: DROP старых партиций или миграция в columnar storage
```

**5. Запрос агрегаций:**
```sql
-- Быстрый запрос по часам (из материализованного представления):
SELECT hour, device_id, avg_value
FROM metrics_hourly
WHERE hour >= '2024-01-01' AND hour < '2024-02-01'
ORDER BY hour, device_id;

-- Partition pruning: сканируются только нужные партиции
SELECT date_trunc('hour', timestamp), AVG(value)
FROM metrics
WHERE timestamp >= '2024-01-01' AND timestamp < '2024-01-02'
GROUP BY 1;
```

**Альтернатива: TimescaleDB** (расширение PostgreSQL для time-series):
- Hypertables (авто-партиционирование);
- Native compression (columnar, 10–20× сжатие);
- Continuous aggregates (auto-refresh);
- Skip scan для sparse data.

</details>

---

### Задача №5: [ Полный текстовый поиск с ранжированием ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Условие</summary>

Таблица `articles(id, title, body)`. Реализовать полнотекстовый поиск с:
- Морфологией (русский язык);
- Ранжированием (title важнее body);
- Выделением найденных фрагментов (highlight).

</details>

<details>
<summary>Решение</summary>

**1. GIN-индекс на tsvector:**
```sql
CREATE INDEX idx_articles_search
    ON articles
    USING GIN (to_tsvector('russian', coalesce(title, '') || ' ' || coalesce(body, '')));
```

Или с весами через отдельную колонку:
```sql
ALTER TABLE articles ADD COLUMN search_vector tsvector;

UPDATE articles SET search_vector =
    setweight(to_tsvector('russian', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('russian', coalesce(body, '')), 'B');

CREATE INDEX idx_articles_search ON articles USING GIN (search_vector);

-- Триггер для автообновления:
CREATE TRIGGER trg_articles_search
    BEFORE INSERT OR UPDATE ON articles
    FOR EACH ROW EXECUTE FUNCTION
    tsvector_update_trigger(search_vector, 'pg_catalog.russian', title, body);
```

**2. Поиск с ранжированием:**
```sql
SELECT
    id,
    title,
    ts_rank(search_vector, query) AS rank,
    ts_headline('russian', body, query,
        'StartSel=<b>, StopSel=</b>, MaxWords=35, MinWords=15'
    ) AS snippet
FROM articles,
    websearch_to_tsquery('russian', 'базы данных производительность') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

**Объяснение:**
- `to_tsvector` — разбиение текста на лексемы (с нормализацией/стеммингом);
- `websearch_to_tsquery` — поисковый запрос (поддерживает кавычки, OR, минус);
- `setweight(..., 'A')` — вес лексемы: A > B > C > D (title=A, body=B);
- `ts_rank` — вычисляет релевантность (учитывает вес и частоту);
- `ts_headline` — выделяет найденные фрагменты (highlight).

**3. Расширенный поиск с фразами:**
```sql
-- Phrase search: "базы данных" как фраза (слова рядом)
SELECT * FROM articles
WHERE search_vector @@ phraseto_tsquery('russian', 'базы данных');

-- websearch_to_tsquery (Google-подобный синтаксис):
-- "базы данных" -MySQL PostgreSQL
SELECT * FROM articles
WHERE search_vector @@ websearch_to_tsquery('russian', '"базы данных" -MySQL PostgreSQL');
```

</details>

---

### Задача №6: [ JSONB: запросы и индексация ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Условие</summary>

Таблица `events(id, data JSONB)`. Пример data:
```json
{"user": {"id": 42, "name": "Alice"}, "tags": ["click", "view"], "amount": 1500}
```

Нужно: (1) искать по вложенным полям, (2) искать по элементам массива, (3) эффективная индексация.

</details>

<details>
<summary>Решение</summary>

**1. Запросы по JSONB:**
```sql
-- Доступ к вложенным полям (-> для JSON, ->> для TEXT)
SELECT * FROM events WHERE data->'user'->>'name' = 'Alice';
SELECT * FROM events WHERE (data->>'amount')::int > 1000;

-- Существует ли ключ:
SELECT * FROM events WHERE data ? 'tags';

-- Поиск по элементам массива:
SELECT * FROM events WHERE data->'tags' ? 'click';

-- Содержит ли JSONB (@>):
SELECT * FROM events WHERE data @> '{"user": {"id": 42}}';
SELECT * FROM events WHERE data @> '{"tags": ["view"]}';

-- Любой из ключей (||):
SELECT * FROM events WHERE data ?| array['tags', 'amount'];

-- Все из ключей (?&):
SELECT * FROM events WHERE data ?& array['tags', 'amount'];
```

**2. Индексация JSONB:**

```sql
-- GIN-индекс на весь JSONB (поддерживает @>, ?, ?|, ?&)
CREATE INDEX idx_events_data_gin ON events USING GIN (data);

-- GIN с jsonb_path_ops (меньше размер, только @>, но быстрее)
CREATE INDEX idx_events_data_path ON events USING GIN (data jsonb_path_ops);

-- Expression index на конкретное поле (для точечного поиска)
CREATE INDEX idx_events_user_id ON events((data->'user'->>'id'));
-- Используется для: WHERE data->'user'->>'id' = '42'

-- B-Tree на конкретное поле (для диапазонов/сортировки)
CREATE INDEX idx_events_amount ON events(((data->>'amount')::int));
-- Используется для: WHERE (data->>'amount')::int > 1000
```

**3. JSON Path (PostgreSQL 12+):**
```sql
-- SQL/JSON Path Language
SELECT * FROM events
WHERE data @? '$.user[*] ? (@.name == "Alice")';

-- Возвращает значения через JSONBPath:
SELECT jsonb_path_query(data, '$.tags[*]') AS tag FROM events;
-- "click"
-- "view"

-- Агрегация:
SELECT
    jsonb_path_query_first(data, '$.user.name') AS user_name,
    COUNT(*) AS cnt
FROM events
GROUP BY 1;
```

**Когда использовать JSONB vs реляционные таблицы:**
- JSONB: schemaless данные, varying attributes, event payloads, config;
- Реляционные: structured data с связями, когда нужна целостность (FK, constraints),
  когда нужны сложные JOIN'ы.

</details>

---

### Задача №7: [ UPSERT (INSERT ... ON CONFLICT) — insertion or update ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Условие</summary>

Таблица `product_stats(product_id, date, views, purchases)`.
При каждом просмотре товара нужно увеличить `views` или создать запись, если её нет.
Обеспечить атомарность и отсутствие race condition.

</details>

<details>
<summary>Решение</summary>

```sql
-- Таблица с unique constraint на (product_id, date)
CREATE TABLE product_stats (
    product_id INT NOT NULL,
    date DATE NOT NULL,
    views BIGINT NOT NULL DEFAULT 0,
    purchases BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (product_id, date)
);

-- UPSERT: вставить или обновить
INSERT INTO product_stats (product_id, date, views)
VALUES (42, '2024-01-15', 1)
ON CONFLICT (product_id, date)
DO UPDATE SET views = product_stats.views + 1
RETURNING (xmax = 0) AS was_inserted;
-- xmax = 0 → строка была вставлена (новая)
-- xmax != 0 → строка была обновлена (существовала)
```

**Особенности:**
- `ON CONFLICT (column)` — требует unique constraint или index на указанные колонки;
- `DO NOTHING` — просто пропустить конфликт:
  ```sql
  INSERT INTO users (email) VALUES ('a@b.c')
  ON CONFLICT (email) DO NOTHING;
  ```
- `DO UPDATE SET` — обновить существующую строку;
- Доступ к существующей строке через таблицу `excluded`:
  ```sql
  INSERT INTO products (id, name, price)
  VALUES (1, 'Widget', 100)
  ON CONFLICT (id)
  DO UPDATE SET name = excluded.name, price = excluded.price;
  ```
- `RETURNING` работает для обоих случаев (insert и update).

**Partial unique constraint:**
```sql
-- Только один активный продукт на категорию
CREATE UNIQUE INDEX idx_one_active_per_category
    ON products(category_id) WHERE status = 'active';

INSERT INTO products (category_id, status) VALUES (1, 'active')
ON CONFLICT (category_id) WHERE status = 'active'
DO UPDATE SET name = excluded.name;
```

</details>

---

### Задача №8: [ Реализация optimistic locking через version field ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Условие</summary>

Таблица `documents(id, title, body, version)`. Несколько пользователей редактируют документы.
Реализовать optimistic locking: при конфликте — пользователь получает ошибку и может merge/retry.

</details>

<details>
<summary>Решение</summary>

```sql
-- Пользователь читает документ (получает version=5)
SELECT id, title, body, version FROM documents WHERE id = 1;

-- Пользователь сохраняет изменения
UPDATE documents
SET title = $1, body = $2, version = version + 1
WHERE id = 1 AND version = 5
RETURNING *;
-- Если вернулась строка → успешно, новый version=6
-- Если 0 строк → кто-то изменил документ раньше (version уже != 5) → конфликт
```

**На уровне приложения:**
```go
result, err := db.ExecContext(ctx, `
    UPDATE documents
    SET title = $1, body = $2, version = version + 1
    WHERE id = $3 AND version = $4
`, newTitle, newBody, docID, currentVersion)

rowsAffected, _ := result.RowsAffected()
if rowsAffected == 0 {
    // OptimisticLockError → предложить пользователю merge
}
```

**Преимущества перед SELECT FOR UPDATE (пессимистичная):**
- Нет блокировок → выше параллелизм;
- Подходит для длительных сессий редактирования (формы в UI);
- Нет удержания DB-блокировки всё время редактирования.

**Недостатки:**
- При высокой конкуренции → много retry (thundering herd);
- Нужно передавать version в каждом запросе.

**Альтернатива: xmin (системная колонка)**
```sql
-- xmin — ID транзакции, создавшей версию строки
SELECT id, title, body, xmin FROM documents WHERE id = 1;

UPDATE documents SET title = $1
WHERE id = $2 AND xmin = $3;
-- Но: xmin может меняться при VACUUM → ненадёжно для долгоживущих сессий
```

</details>

---

### Задача №9: [ Миграция большой таблицы без простоя (zero-downtime) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Условие</summary>

Нужно добавить NOT NULL колонку с default value в таблицу 500M строк без простоя.
Прямой `ALTER TABLE` заблокирует таблицу на часы.

</details>

<details>
<summary>Решение</summary>

**Стандартный подход (PostgreSQL 11+ для простых default):**

PostgreSQL 11+ оптимизировал добавление колонки с constant default — метаданные only,
не переписывает таблицу:
```sql
-- Быстро (мгновенно) для constant default:
ALTER TABLE big_table ADD COLUMN status TEXT NOT NULL DEFAULT 'active';
```

**Для non-constant default (вычисляемый) — expand/contract pattern:**

**Шаг 1: Добавить колонку как nullable (мгновенно)**
```sql
ALTER TABLE big_table ADD COLUMN new_col TEXT;
```

**Шаг 2: Заполнить данными батчами (без блокировки)**
```sql
-- Батчами по 10K строк, с паузами
DO $$
DECLARE
    affected INT;
BEGIN
    LOOP
        UPDATE big_table
        SET new_col = compute_value(id)
        WHERE id IN (
            SELECT id FROM big_table WHERE new_col IS NULL LIMIT 10000
        );
        GET DIAGNOSTICS affected = ROW_COUNT;
        EXIT WHEN affected = 0;
        PERFORM pg_sleep(0.1);  -- пауза 100ms между батчами
    END LOOP;
END $$;
```

**Шаг 3: Установить default и NOT NULL**
```sql
ALTER TABLE big_table ALTER COLUMN new_col SET DEFAULT 'default_val';
-- Проверить что нет NULL (с всех строк уже заполнено):
SELECT 1 FROM big_table WHERE new_col IS NULL LIMIT 1;
-- Если пусто:
ALTER TABLE big_table ALTER COLUMN new_col SET NOT NULL;
```

**Шаг 4: Обновить код приложения** (если нужно — write to both old and new).

**Для создания индекса без блокировки:**
```sql
CREATE INDEX CONCURRENTLY idx_new ON big_table(new_col);
-- Не блокирует запись, но занимает дольше и не может быть в транзакции
```

**Для изменения типа колонки:**
```sql
-- Добавить новую колонку нужного типа → заполнить → переключить → удалить старую
ALTER TABLE big_table ADD COLUMN amount_new NUMERIC(12,2);
-- ... batch update ...
ALTER TABLE big_table DROP COLUMN amount;
ALTER TABLE big_table RENAME COLUMN amount_new TO amount;
```

</details>

---

### Задача №10: [ Audit log: триггеры vs application-level ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Условие</summary>

Реализовать audit log для таблицы `orders`: отслеживать все изменения (INSERT/UPDATE/DELETE)
с хранением старых и новых значений, пользователя и времени.

</details>

<details>
<summary>Решение</summary>

**Вариант 1: Trigger-based (надёжный, DB-level)**

```sql
CREATE TABLE orders_audit (
    id BIGSERIAL PRIMARY KEY,
    operation TEXT NOT NULL,       -- INSERT / UPDATE / DELETE
    order_id BIGINT NOT NULL,
    old_data JSONB,                -- старые значения (NULL для INSERT)
    new_data JSONB,                -- новые значения (NULL для DELETE)
    changed_by TEXT DEFAULT current_user,
    changed_at TIMESTAMPTZ DEFAULT now()
);

CREATE OR REPLACE FUNCTION audit_orders() RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO orders_audit (operation, order_id, new_data)
        VALUES ('INSERT', NEW.id, to_jsonb(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO orders_audit (operation, order_id, old_data, new_data)
        VALUES ('UPDATE', NEW.id, to_jsonb(OLD), to_jsonb(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO orders_audit (operation, order_id, old_data)
        VALUES ('DELETE', OLD.id, to_jsonb(OLD));
        RETURN OLD;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_orders
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION audit_orders();
```

**Плюсы trigger-based:**
- Невозможно обойти (даже прямой SQL-запрос);
- Не зависит от приложения;
- Atomicity: audit + change в одной транзакции.

**Минусы:**
- Overhead на каждую операцию (запись в audit table);
- Сложность отладки (логика в DB);
- Репликация: аудит только на primary (или отдельная логика).

**Вариант 2: Application-level (гибкий)**

```go
func UpdateOrder(ctx context.Context, id int64, updates map[string]any) error {
    return db.Tx(ctx, func(tx *sql.Tx) error {
        // 1. Прочитать старые значения
        var old Order
        tx.QueryRowContext(ctx, "SELECT * FROM orders WHERE id = $1", id).Scan(...)

        // 2. Обновить
        tx.ExecContext(ctx, "UPDATE orders SET ... WHERE id = $1", id)

        // 3. Записать audit
        tx.ExecContext(ctx,
            "INSERT INTO orders_audit (operation, order_id, old_data, new_data) VALUES (...)")

        return nil
    })
}
```

**Плюсы:** гибкость (фильтрация, enrichment, async), тестируемость.
**Минусы:** можно забыть добавить аудит в новом коде; разные приложения могут вести себя по-разному.

**Вариант 3: Logical Decoding / CDC (для масштабных систем)**
```sql
-- Настройка logical replication slot
-- Плагин: pglogical, wal2json, Debezium
-- Захватывает изменения из WAL → публикует в Kafka
-- Плюс: zero application impact, real-time, ordered
-- Минус: сложная инфраструктура
```

</details>

---

### Задача №11: [ Агрегация по скользящему окну с разными группировками ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Условие</summary>

Таблица `sales(date, store_id, product_id, quantity, revenue)`.

В одном запросе вывести для каждого магазина и продукта:
- Сумма за последние 7 дней (включая текущий день);
- Сумма за последние 30 дней;
- Среднедневной показатель за 30 дней;
- Рост в % по сравнению с предыдущим 7-дневным периодом.

</details>

<details>
<summary>Решение</summary>

```sql
WITH daily AS (
    -- Агрегация по дням (если есть несколько записей в день)
    SELECT
        date,
        store_id,
        product_id,
        SUM(quantity) AS qty,
        SUM(revenue) AS rev
    FROM sales
    GROUP BY date, store_id, product_id
),
windowed AS (
    SELECT
        date,
        store_id,
        product_id,
        qty,
        rev,
        -- Скользящая сумма за 7 дней
        SUM(qty) OVER w7  AS qty_7d,
        SUM(rev) OVER w7  AS rev_7d,
        -- Скользящая сумма за 30 дней
        SUM(qty) OVER w30 AS qty_30d,
        SUM(rev) OVER w30 AS rev_30d,
        -- Предыдущий 7-дневный период (дни -14..-8)
        SUM(qty) OVER (
            PARTITION BY store_id, product_id
            ORDER BY date
            ROWS BETWEEN 14 PRECEDING AND 8 PRECEDING
        ) AS qty_prev_7d
    FROM daily
    WINDOW
        w7  AS (PARTITION BY store_id, product_id ORDER BY date
                ROWS BETWEEN 6 PRECEDING AND CURRENT ROW),
        w30 AS (PARTITION BY store_id, product_id ORDER BY date
                ROWS BETWEEN 29 PRECEDING AND CURRENT ROW)
)
SELECT
    date,
    store_id,
    product_id,
    qty_7d,
    rev_7d,
    qty_30d,
    rev_30d,
    ROUND(rev_30d / 30.0, 2) AS avg_daily_rev_30d,
    CASE
        WHEN qty_prev_7d > 0
        THEN ROUND((qty_7d - qty_prev_7d) * 100.0 / qty_prev_7d, 2)
        ELSE NULL
    END AS growth_pct
FROM windowed
ORDER BY store_id, product_id, date;
```

**Ключевые приёмы:**
- `WINDOW` clause — определение окон один раз, переиспользование;
- `ROWS BETWEEN N PRECEDING AND CURRENT ROW` — скользящее окно по физическим строкам;
- Для расчёта "предыдущего периода" — смещаем окно (14 PRECEDING..8 PRECEDING);
- `NULLIF` / `CASE` для защиты от деления на ноль.

</details>

---

### Задача №12: [ Репликация PostgreSQL: streaming vs logical ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Условие</summary>

Сравнить streaming и logical репликацию. Когда какую использовать?

</details>

<details>
<summary>Решение</summary>

| | Streaming Replication | Logical Replication |
|---|---|---|
| **Уровень** | Физический (блоки WAL) | Логический (INSERT/UPDATE/DELETE) |
| **Что реплицируется** | Вся БД (или кластер) | Отдельные таблицы |
| **Версия PG** | Должна совпадать | Может отличаться (major version) |
| **ОС / архитектура** | Должна совпадать | Может отличаться |
| **Replica writable** | Нет (read-only) | Да (можно писать) |
| **Разные схемы** | Нет | Да (частичная) |
| **Триггеры на replica** | Нет (Hot Standby не запускает) | Да (запускаются) |
| **TRUNCATE** | Реплицируется | Реплицируется (PG 11+) |
| **DDL** | Реплицируется | **Нет** (нужно вручную) |
| **SEQUENCE** | Реплицируется | **Нет** (нужно синхронизировать) |
| **Lag** | Низкий (WAL streaming) | Чуть выше (parsing + apply) |

**Streaming Replication:**
```ini
# master: postgresql.conf
wal_level = replica
max_wal_senders = 10

# replica: postgresql.conf
hot_standby = on

# replica: recovery.conf / standby.signal
primary_conninfo = 'host=master port=5432'
```
Use cases: HA (failover), read replicas, PITR.

**Logical Replication:**
```sql
-- Publisher (master)
CREATE PUBLICATION my_pub FOR TABLE users, orders;

-- Subscriber (target)
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=master port=5432 dbname=mydb'
    PUBLICATION my_pub;
```
Use cases:
- Major version upgrade (zero-downtime);
- Частичная репликация (только нужные таблицы);
- Replication между разными средами;
- CDC (Change Data Capture) → Kafka через wal2json/pgoutput.

**Подводные камни logical replication:**
- DDL не реплицируется → синхронизация schema вручную;
- sequences не реплицируются → `pg_logical_emit_message` или ручная синхронизация;
- Large objects не поддерживаются;
- Без primary key → UPDATE/DELETE не реплицируются (нужен replica identity);
- Конфликты: при INSERT с тем же PK → replication останавливается (нужно ручное разрешение).

</details>

---

</details>

---

## 7. Администрирование и эксплуатация

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Как настроить connection pooling? pgBouncer vs pgcat ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

PostgreSQL использует **process-per-connection** — каждое соединение = отдельный OS-процесс
(~5–10MB RAM). При 1000+ соединений → overhead на context switching и memory.

**pgBouncer** — легковесный пулер соединений:

**Режимы:**
- **Session pooling** — клиенту выделяется backend на всё время сессии (освобождается при
  отключении). Как без пулера, но ограничивает max connections;
- **Transaction pooling** — backend выделяется на время транзакции (освобождается при COMMIT/ROLLBACK).
  **Максимальная эффективность.** Ограничения: SET, temp tables, prepared statements, LISTEN/NOTIFY
  не работают корректно (state не сохраняется между транзакциями);
- **Statement pooling** — на каждый оператор (ограничения ещё больше, редко используется).

**Когда transaction pooling НЕ подходит:**
- Prepared statements (session-level) → нужно использовать prepared statements protocol или
  pgbouncer 1.21+ с поддержкой protocol-level prepared statements;
- Temporary tables (session-level);
- SET (session variables);
- LISTEN/NOTIFY;
- Advisory locks (session-level) → использовать transaction-level advisory locks.

**Конфигурация (типичная):**
```ini
[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
pool_mode = transaction
max_client_conn = 5000       # клиенты
default_pool_size = 25       # backend connections per db per user
reserve_pool_size = 5        # extra connections under load
server_idle_timeout = 600    # закрыть idle backend через 10 мин
```

**Pgcat** — альтернатива (Rust), поддерживает multi-tenant, sharding, Query cancellation.

**Alternatives:**
- **Application-level pooling** (Go: database/sql с max_open_conns; Java: HikariCP);
- **Supavisor** (Elixir, от Supabase) — multi-tenant pooling.

</details>

---

### Вопрос №2: [ Что делать при высоком load average / медленных запросах в production? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**Диагностика:**

1. **Текущие активные запросы:**
```sql
SELECT pid, state, wait_event_type, wait_event,
       now() - query_start AS duration,
       LEFT(query, 200) AS query_preview
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;
```

2. **Блокировки:**
```sql
SELECT
    blocked.pid     AS blocked_pid,
    blocked.query   AS blocked_query,
    blocking.pid    AS blocking_pid,
    blocking.query  AS blocking_query,
    now() - blocked.query_start AS blocked_duration
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

3. **pg_stat_statements** — top медленных запросов:
```sql
SELECT
    queryid,
    LEFT(query, 100) AS query,
    calls,
    mean_exec_time,
    total_exec_time,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

4. **Размеры таблиц и bloat:**
```sql
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;
```

**Экстренные действия:**

```sql
-- Отменить конкретный запрос (мягко, позволяет COMMIT/ROLLBACK):
SELECT pg_cancel_backend(pid);

-- Терминация процесса (жёстко, убивает соединение):
SELECT pg_terminate_backend(pid);

-- Отменить все запросы длиннее 5 минут:
SELECT pg_cancel_backend(pid)
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > INTERVAL '5 minutes'
  AND pid != pg_backend_pid();
```

**Профилактика:**
```sql
-- Timeout на запросы (per session):
SET statement_timeout = '30s';

-- Timeout на блокировки:
SET lock_timeout = '5s';

-- Idle in transaction timeout (убивает забывшие COMMIT):
SET idle_in_transaction_session_timeout = '60s';
```

</details>

---

### Вопрос №3: [ Как настроить shared_buffers, work_mem, effective_cache_size? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**shared_buffers** — размер кэша страниц в shared memory:
- Default: 128MB (слишком мало);
- Рекомендация: **25% RAM** (для dedicated DB server);
- Больше не лучше — ОС также кэширует страницы (double caching).

```ini
shared_buffers = 4GB  # при 16GB RAM
```

**effective_cache_size** — оценка planner'ом размера OS cache:
- Не выделяет память! Только подсказка для planner'а;
- Рекомендация: **50–75% RAM**;
- Влияет на выбор Index Scan vs Seq Scan (больше = planner думает, что данные в кэше).

```ini
effective_cache_size = 12GB  # при 16GB RAM
```

**work_mem** — память на сортировку/хэш per operation:
- Default: 4MB (мало для тяжёлых запросов);
- Риск: `work_mem × max_connections × concurrent_sorts` может превысить RAM;
- Рекомендация: базовое 16–64MB, per-session для тяжёлых запросов.

```ini
work_mem = 16MB  # глобально
-- Для конкретного запроса:
SET LOCAL work_mem = '256MB';
```

**maintenance_work_mem** — память для VACUUM, CREATE INDEX, ALTER TABLE:
- Default: 64MB;
- Рекомендация: 256MB–1GB (больше = быстрее VACUUM/REINDEX).

```ini
maintenance_work_mem = 512MB
```

**Контрольный список (для 16GB RAM dedicated PG):**
```ini
shared_buffers = 4GB
effective_cache_size = 12GB
work_mem = 16MB
maintenance_work_mem = 512MB
wal_buffers = 16MB
max_connections = 100  # + pgBouncer
random_page_cost = 1.1  # для SSD
```

</details>

---

### Вопрос №4: [ Backup стратегии: pg_dump vs pg_basebackup vs PITR ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

| Метод | Что | Восстановление | Consistency | Размер |
|---|---|---|---|---|
| `pg_dump` | Логический дамп (SQL или custom) | На любой сервер/версию | Точка во времени | Маленький (текст) |
| `pg_basebackup` | Физическая копия (binary) | Только та же версия PG | Точка во времени | = размер БД |
| **PITR** | pg_basebackup + WAL archive | На любой момент времени | Любая точка | = размер + WAL |

**pg_dump (логический):**
```bash
# Custom format (быстрее, поддерживает параллельное восстановление)
pg_dump -Fc -j 4 -f backup.dump mydb

# Восстановление
pg_restore -j 4 -d newdb backup.dump

# Только схема
pg_dump -s -f schema.sql mydb
```
- Плюс: перенос между версиями, ОС, архитектурой; selective restore;
- Минус: медленно для больших БД; нет PITR.

**pg_basebackup (физический):**
```bash
pg_basebackup -D /backup -Ft -z -P -h master -U replicator
# -Ft: tar, -z: gzip, -P: progress
```
- Плюс: быстро; полная копия (включая конфиг);
- Минус: только та же версия; нет PITR без WAL archive.

**PITR (Point-in-Time Recovery):**
```ini
# postgresql.conf
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
wal_level = replica
```
```bash
# Восстановление на момент времени:
# recovery.conf (PG < 12) или postgresql.conf (PG 12+):
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
recovery_target_action = 'promote'  # или 'pause' для проверки
```

**Стратегия (рекомендуемая):**
1. **Ежедневный pg_basebackup** (полная физическая копия);
2. **Continuous WAL archiving** (для PITR);
3. **Еженедельный pg_dump** (на случай logical restore / migration);
4. **Тестирование восстановления** (бэкап не существует, пока не восстановлен!).

**Инструменты:**
- **pgBackRest** — backup, restore, WAL archive, delta restore, retention;
- **Barman** — backup management, remote backup;
- **WAL-G** — WAL archiving to S3/GCS, backup compression.

</details>

---

### Вопрос №5: [ Как мониторить PostgreSQL? Ключевые метрики ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

**Ключевые метрики:**

1. **Connections:**
```sql
SELECT count(*), state FROM pg_stat_activity GROUP BY state;
-- active, idle, idle in transaction
-- Алёрм: idle_in_transaction > 0 долгое время (утечки транзакций)
-- Алёрм: total connections > 80% max_connections
```

2. **pg_stat_statements (top queries):**
```sql
-- Топ по total time:
SELECT query, calls, mean_exec_time, total_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 20;

-- Топ по I/O:
SELECT query, shared_blks_read + shared_blks_hit AS buffers,
       shared_blks_read AS reads
FROM pg_stat_statements
ORDER BY shared_blks_read DESC LIMIT 20;
```

3. **Cache hit ratio:**
```sql
SELECT
    sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) AS hit_ratio
FROM pg_statio_user_tables;
-- Цель: > 99% (ниже → нужен индекс или больше shared_buffers)
```

4. **Replication lag:**
```sql
-- На master:
SELECT client_addr, state, write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
-- На replica:
SELECT now() - pg_last_xact_replay_timestamp() AS lag;
```

5. **Dead tuples (autovacuum effectiveness):**
```sql
SELECT relname, n_live_tup, n_dead_tup,
       ROUND(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) AS dead_pct,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY dead_pct DESC;
```

6. **Database size growth:**
```sql
SELECT datname, pg_size_pretty(pg_database_size(datname))
FROM pg_database;
```

7. **Locks waiting:**
```sql
SELECT count(*) FROM pg_stat_activity WHERE wait_event IS NOT NULL;
```

8. **Transaction wraparound (критично!):**
```sql
SELECT datname, age(datfrozenxid) AS xid_age
FROM pg_database
ORDER BY xid_age DESC;
-- При xid_age > 1.5 млрд → срочно VACUUM (иначе БД остановится!)
```

**Инструменты мониторинга:**
- **pg_stat_statements** — собирает статистику запросов;
- **Prometheus + postgres_exporter** — метрики → Grafana;
- **pgBadger** — анализ логов (slow queries, errors, locks);
- **pgCenter** — top-like мониторинг в терминале.

</details>

---

</details>

---

## Шпаргалка: полезные SQL-сниппеты

<details>
<summary><b>Развернуть</b></summary>

```sql
-- Размер таблицы (с индексами)
SELECT pg_size_pretty(pg_total_relation_size('my_table'));

-- Размер всех таблиц (топ-10)
SELECT relname, pg_size_pretty(pg_total_relation_size(relid))
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC LIMIT 10;

-- Размер всех БД
SELECT datname, pg_size_pretty(pg_database_size(datname))
FROM pg_database ORDER BY pg_database_size(datname) DESC;

-- Активные запросы
SELECT pid, state, wait_event, now()-query_start AS duration, LEFT(query,200)
FROM pg_stat_activity WHERE state != 'idle' ORDER BY duration DESC;

-- Убить запрос
SELECT pg_cancel_backend(pid);    -- мягко (SIGINT)
SELECT pg_terminate_backend(pid); -- жёстко (SIGTERM)

-- Проверить cache hit ratio
SELECT sum(heap_blks_hit)::float / NULLIF(sum(heap_blks_hit)+sum(heap_blks_read),0)
FROM pg_statio_user_tables;

-- Список индексов и их размер
SELECT schemaname, relname, indexrelname,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size,
       idx_scan AS scans
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Неиспользуемые индексы (кандидаты на удаление)
SELECT relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Текущие блокировки
SELECT pid, locktype, relation::regclass, mode, granted
FROM pg_locks WHERE NOT granted;

-- Статистика по БД (tps, commits, rollbacks)
SELECT datname, xact_commit, xact_rollback,
       blks_read, blks_hit, tup_returned, tup_fetched, tup_inserted
FROM pg_stat_database;

-- Обновить статистику (для planner'а)
ANALYZE my_table;

-- Перестроить индекс без блокировки
REINDEX INDEX CONCURRENTLY my_index;

-- Проверить версию PostgreSQL
SELECT version();
```

</details>
