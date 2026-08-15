# Observability — Логирование, метрики, трейсинг

Файл содержит вопросы по observability: логированию, метрикам, distributed tracing,
алертингу, SLO/SLI/SLA и инструментам (Prometheus, Grafana, OpenTelemetry, ELK).

Уровни сложности:
- ![Static Badge](https://img.shields.io/badge/Middle-yellow) — Middle разработчик
- ![Static Badge](https://img.shields.io/badge/Senior-red) — Senior разработчик

---

# Оглавление

1. [Observability — основы](#1-observability--основы)
2. [Логирование](#2-логирование)
3. [Метрики](#3-метрики)
4. [Distributed Tracing](#4-distributed-tracing)
5. [Prometheus и Grafana](#5-prometheus-и-grafana)
6. [SLO, SLI, SLA](#6-slo-sli-sla)
7. [Алертинг и инциденты](#7-алертинг-и-инциденты)
8. [OpenTelemetry](#8-opentelemetry)

---

## 1. Observability — основы

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №1: [ Что такое observability? Чем отличается от monitoring? ]

    <details>
      <summary>Ответ</summary>

    * **Monitoring** — сбор и отображение предопределённых метрик. Отвечает на известные вопросы
      ("CPU usage?", "error rate?"). Показывает, **что** произошло.
    * **Observability** — свойство системы, позволяющее понимать её внутреннее состояние по внешним
      сигналам. Отвечает на **любые** вопросы, включая неизвестные заранее ("почему этот запрос
      медленный для конкретного пользователя?").
    * **Три кита observability (три пилара):**
      * **Logs** — дискретные события с контекстом.
      * **Metrics** — агрегированные числовые значения во времени.
      * **Traces** — путь запроса через распределённую систему.
    * **Дополнительно:** structured events, profiling (eBPF, continuous profiling).
    * **Ключевое отличие:** monitoring — вы знаете, что искать. Observability — вы можете исследовать
      неизвестное. Observability = monitoring + ability to debug.

    </details>

  ---

  - Вопрос №2: [ Что такое cardinality? Почему важна для метрик? ]

    <details>
      <summary>Ответ</summary>

    * **Cardinality** — количество уникальных значений метки (label).
    * **High cardinality** — метка с множеством уникальных значений (user_id, email, session_id, IP).
    * **Проблема:** TSDB (Prometheus, InfluxDB) хранят каждый уникальный набор labels как отдельный
      time series.
      * `http_requests_total{user_id="123"}` — если 1M пользователей, то 1M time series.
      * Память и диск растут экспоненциально. Prometheus может OOM.
    * **Правила:**
      * Labels должны иметь **низкую cardinality** (method, status, endpoint → десятки значений).
      * `user_id`, `email`, `session_id` — **никогда** в метриках. Только в logs/traces.
      * Cardinality budget — мониторить `prometheus_tsdb_head_series`.
    * **Лимит:** Prometheus `--storage.tsdb.max-block-duration` и label cardinality limits (через
      relabeling).

    </details>

  ---

  - Вопрос №3: [ Как соотносятся logs, metrics, traces? Когда что использовать? ]

    <details>
      <summary>Ответ</summary>

    | | Logs | Metrics | Traces |
    |--|------|---------|--------|
    | Что | Дискретное событие | Агрегированное значение | Путь запроса |
    | Объём | Большой | Маленький | Средний |
    | Cardinality | Высокая (любые поля) | Низкая (labels) | Высокая (span attributes) |
    | Query | "Покажи error logs за 5 мин" | "Error rate за час" | "Почему запрос медленный?" |
    | Cost | Высокий (хранение) | Низкий | Средний |
    | Использование | Debug, audit | Alerting, dashboards | Latency analysis |

    * **Workflow:**
      1. **Metric** показывает проблему (error rate вырос → alert).
      2. **Trace** показывает, **где** проблема (какой сервис/span медленный).
      3. **Log** показывает, **почему** (детали ошибки в этом сервисе).
    * **Корреляция:** trace_id в logs → найти все логи по запросу. trace_id в metrics (exemplar) →
      найти trace по spike.

    </details>

  ---

  - Вопрос №4: [ Что такое structured logging? Зачем нужен? ]

    <details>
      <summary>Ответ</summary>

    * **Structured logging** — логи в машиночитаемом формате (JSON, не plain text).
    * **Пример:**
      ```json
      {"timestamp":"2024-01-01T12:00:00Z","level":"error","service":"payment",
       "trace_id":"abc123","msg":"payment failed","user_id":"42","amount":100}
      ```
    * **Преимущества:**
      * **Searchable** — фильтр по полям (`level=error AND service=payment`).
      * **Aggregatable** — count, group by в Loki/ELK.
      * **Correlation** — trace_id, user_id для связи с traces.
      * **No parsing** — не нужны regex для извлечения полей.
    * **В Go:** `slog` (стандарт с Go 1.21), `zerolog`, `zap`.
    * **Уровни:** DEBUG, INFO, WARN, ERROR. В prod обычно INFO+, DEBUG по требованию.

    </details>

  ---

</details>

---

## 2. Логирование

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №5: [ Что такое ELK / EFK stack? Как работает? ]

    <details>
      <summary>Ответ</summary>

    * **ELK** — Elasticsearch + Logstash + Kibana.
    * **EFK** — Elasticsearch + Fluentd/Fluent Bit + Kibana (Fluentd вместо Logstash).
    * **Поток данных:**
      1. **Shipper** (Fluent Bit, Filebeat) — собирает логи с серверов/контейнеров.
      2. **Aggregator** (Fluentd, Logstash) — парсит, обогащает, фильтрует.
      3. **Storage** (Elasticsearch) — хранит, индексирует, ищет.
      4. **UI** (Kibana) — поиск, дашборды, алерты.
    * **Logstash vs Fluentd:**
      * Logstash — JVM, тяжёлый, богатый plugin ecosystem.
      * Fluentd — Ruby/C, легче, Kubernetes-native (Fluent Bit ещё легче — на C).
    * **Альтернативы:** Loki + Grafana (дешевле, индексирует только labels, не полный текст).
    * **Архитектура для K8s:** DaemonSet Fluent Bit на каждой ноде → собирает из `/var/log/containers`
      → отправляет в Loki/ES.

    </details>

  ---

  - Вопрос №6: [ Что такое Loki? Чем отличается от Elasticsearch? ]

    <details>
      <summary>Ответ</summary>

    * **Loki** — log aggregation system от Grafana Labs. "Prometheus for logs."
    * **Архитектура:**
      * Индексирует только **labels** (метод Prometheus), не содержимое логов.
      * Содержимое логов хранится compressed (chunks), не индексируется.
      * Поиск: сначала фильтр по labels, потом grep по содержимому.
    * **Преимущества vs Elasticsearch:**
      * **Дешевле** — меньше хранения, меньше RAM.
      * **Проще** — меньше компонентов.
      * **Интеграция** с Grafana и Prometheus (одни labels).
    * **Недостатки:**
      * **Медленнее full-text search** — нет инвертированного индекса по содержимому.
      * Не подходит для сложных аналитических запросов по тексту логов.
    * **Когда выбирать:**
      * Loki — если нужен дешёвый log storage + базовый поиск.
      * ES — если нужен мощный full-text search, сложные запросы.

    </details>

  ---

  - Вопрос №7: [ Как правильно логировать в микросервисах? ]

    <details>
      <summary>Ответ</summary>

    * **Structured logging** — JSON с полями, не plain text.
    * **Correlation IDs:**
      * `trace_id` / `request_id` в каждом логе.
      * Передаётся через HTTP headers (`X-Trace-Id`) / metadata в gRPC / headers в message brokers.
      * Позволяет собрать весь путь запроса по логам.
    * **Контекст:** service name, version, environment, user_id (если есть).
    * **Уровни:**
      * ERROR — что-то сломалось, требует внимания.
      * WARN — необычно, но не сломано.
      * INFO — значимые события (request completed, user registered).
      * DEBUG — детали для отладки (только в dev / при включённом уровне).
    * **Не логировать:**
      * Пароли, токены, PII (GDPR).
      * Тело запроса целиком (может содержать секреты).
      * Каждый шаг цикла (шум).
    * **Sampling:** для high-throughput — логировать не каждый запрос ( probabilistic sampling).
    * **Centralized:** все сервисы → единый log storage (Loki/ES).

    </details>

  ---

  - Вопрос №8: [ Как управлять объёмом логов? ]

    <details>
      <summary>Ответ</summary>

    * **Log levels:** prod = INFO+. DEBUG только при troubleshooting (dynamic log level).
    * **Sampling:** логировать 1% успешных запросов, 100% ошибок.
    * **Retention policies:**
      * Hot data (7 дней) — быстрый поиск.
      * Warm data (30 дней) — медленный поиск.
      * Cold data (90 дней) — archive (S3, GCS).
    * **Compression:** gzip/zstd для archived logs.
    * **Deduplication:** Fluentd `dedot` plugin, или на уровне storage.
    * **Drop noisy logs:** фильтр в shipper (healthcheck logs, static asset requests).
    * **Rate limiting:** ограничить logs/sec per service (защита от log flood при bug).
    * **Cost allocation:** per-team log quotas (Loki `compactor` + tenant limits).

    </details>

  ---

</details>

---

## 3. Метрики

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №9: [ Какие типы метрик бывают? ]

    <details>
      <summary>Ответ</summary>

    * **4 типа метрик (по Prometheus):**
      1. **Counter** — только растёт. `http_requests_total`, `errors_total`.
         * `rate(counter[5m])` — запросов в секунду.
      2. **Gauge** — растёт и падает. `memory_usage_bytes`, `queue_depth`, `active_connections`.
         * Мгновенное значение.
      3. **Histogram** — распределение значений по buckets. `http_request_duration_seconds`.
         * `_bucket{le="0.1"}`, `_count`, `_sum`.
         * `histogram_quantile(0.99, ...)` — p99 latency.
         * Выбор buckets важен — слишком много = дорого, слишком мало = неточно.
      4. **Summary** — как histogram, но quantile считается на клиенте. `go_gc_duration_seconds`.
         * Не агрегируется across instances (нельзя посчитать p99 для нескольких подов).
         * Рекомендуется histogram вместо summary.
    * **USE method** (Utilization, Saturation, Errors) — для ресурсов.
    * **RED method** (Rate, Errors, Duration) — для сервисов.

    </details>

  ---

  - Вопрос №10: [ Что такое RED и USE методы? ]

    <details>
      <summary>Ответ</summary>

    * **RED** — для сервисов (request-driven):
      * **R**ate — запросов в секунду.
      * **E**rrors — доля ошибок.
      * **D**uration — latency (p50, p99).
    * **USE** — для ресурсов (CPU, disk, network):
      * **U**tilization — % использования.
      * **S**aturation — очередь / ожидание.
      * **E**rrors — ошибки (disk errors, packet loss).
    * **Применение:**
      * Каждый сервис → RED metrics.
      * Каждый ресурс → USE metrics.
      * Базовый dashboard: RED для каждого endpoint.
    * **Four Golden Signals (Google SRE):** Latency, Traffic, Errors, Saturation. = RED + Saturation.

    </details>

  ---

  - Вопрос №11: [ Что такое percentile (p50, p90, p99)? Зачем не average? ]

    <details>
      <summary>Ответ</summary>

    * **Percentile** — значение, ниже которого находится N% наблюдений.
      * p50 (median) — половина запросов быстрее этого значения.
      * p99 — 99% запросов быстрее. 1% медленнее.
    * **Почему не average:**
      * Average скрывает tail latency. 100 запросов по 10ms + 1 запрос по 1000ms → average ~20ms, но
        1% пользователей страдают.
      * Распределение latency обычно heavy-tailed (не нормальное). Average искажён outlier'ами.
    * **Какие percentiles:**
      * p50 — типичный опыт.
      * p95 — большинство пользователей.
      * p99 — critical для SLA. 1% от 1M = 10 000 затронутых.
    * **Гистограмма vs Summary:** histogram позволяет агрегировать percentiles across instances.
    * **Внимание:** p99 за 5 минут может скрывать проблему, если она затронула только 1 под из 100.
      Использовать `max` + `avg` + percentiles.

    </details>

  ---

  - Вопрос №12: [ Как правильно выбирать labels в Prometheus? ]

    <details>
      <summary>Ответ</summary>

    * **Правило:** labels должны идентифицировать **что** измеряется, но не **сколько**.
    * **Хорошие labels:**
      * `method` (GET, POST) — низкая cardinality.
      * `status` (200, 500) — низкая cardinality.
      * `endpoint` (/api/users, /api/orders) — умеренная.
      * `service`, `version` — низкая.
    * **Плохие labels (high cardinality):**
      * `user_id`, `session_id`, `email`, `IP` — тысячи/миллионы уникальных значений.
    * **Неизменные labels:** значение label не должно меняться в течение жизни time series.
      * `status` — OK (каждый status — отдельный series).
      * Но не `timestamp` или `random_id`.
    * **Практика:** суммарная cardinality = произведение cardinality всех labels. 3 labels по 10
      значений = 1000 series. Добавить user_id (1M) → 1 billion series. OOM.

    </details>

  ---

</details>

---

## 4. Distributed Tracing

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №13: [ Что такое distributed tracing? Как работает? ]

    <details>
      <summary>Ответ</summary>

    * **Distributed tracing** — отслеживание пути запроса через распределённую систему.
    * **Концепции:**
      * **Trace** — полный путь одного запроса. Уникальный `trace_id`.
      * **Span** — одна операция в trace. Имеет `span_id`, `parent_span_id`, start/end time.
      * **Span context** — `trace_id` + `span_id` + flags. Передаётся между сервисами.
      * **Propagation** — через HTTP headers (`traceparent`, `b3`) / gRPC metadata / message headers.
    * **Дерево spans:**
      ```
      [root] HTTP request (service A)
        ├── DB query (service A → DB)
        ├── gRPC call (service A → service B)
        │     ├── DB query (service B → DB)
        │     └── Redis get (service B → Redis)
        └── HTTP response (service A)
      ```
    * **Что показывает:**
      * Какой сервис/span — bottleneck.
      * Какой сервис вернул ошибку.
      * Параллельные вызовы, retry, таймауты.
    * **Инструменты:** OpenTelemetry, Jaeger, Zipkin, Datadog APM, Honeycomb.

    </details>

  ---

  - Вопрос №14: [ Что такое OpenTelemetry? Зачем нужен? ]

    <details>
      <summary>Ответ</summary>

    * **OpenTelemetry (OTel)** — открытый стандарт для observability (traces, metrics, logs). Результат
      слияния OpenTracing и OpenCensus, под CNCF.
    * **Цель:** единый API + SDK для инструментирования. Не зависит от backend (Jaeger, Prometheus,
      Datadog, etc.).
    * **Архитектура:**
      * **API** — интерфейс (no-op implementation).
      * **SDK** — реализация (sampling, batching, export).
      * **Collector** — прокси: принимает OTLP, экспортирует в разные backend'ы.
      * **Instrumentation libraries** — auto-instrumentation для HTTP, gRPC, DB drivers.
    * **Преимущества:**
      * **Vendor-neutral** — смена backend без изменения кода.
      * **Единый контекст** — trace_id связывает logs, metrics, traces.
      * **Auto-instrumentation** — минимальный код для базового трейсинга.
    * **В Go:**
      ```go
      // Manual span
      ctx, span := tracer.Start(ctx, "processOrder")
      defer span.End()
      span.SetAttributes(attribute.String("order_id", orderID))
      ```

    </details>

  ---

  - Вопрос №15: [ Что такое sampling в tracing? Какие стратегии? ]

    <details>
      <summary>Ответ</summary>

    * **Sampling** — какой процент traces сохранять. Tracing дорог (storage, processing).
    * **Стратегии:**
      * **Head-based sampling** — решение принимается на входе (root span).
        * `probabilistic` — 1% всех запросов.
        * Решение до выполнения → не видит ошибки.
        * Простой, дешёвый, но может пропустить важные traces.
      * **Tail-based sampling** — решение после завершения запроса.
        * Сохранить все errors (status=500).
        * Сохранить slow requests (latency > p99).
        * 1% нормальных запросов.
        * Лучше для debugging, но дороже (нужно держать trace в памяти).
    * **Рекомендация:** tail-based sampling для prod. Head-based для dev/high-volume.
    * **OpenTelemetry Collector** поддерживает tail-based sampling processor.

    </details>

  ---

  - Вопрос №16: [ Как передаётся trace context между сервисами? ]

    <details>
      <summary>Ответ</summary>

    * **Trace context** — `trace_id` + `span_id` + trace flags (sampled?).
    * **W3C Trace Context** (стандарт):
      * Header: `traceparent: 00-{trace-id}-{parent-id}-{trace-flags}`
      * Пример: `traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01`
    * **B3** (Zipkin, широко использовался до W3C):
      * `X-B3-TraceId`, `X-B3-SpanId`, `X-B3-Sampled`.
    * **Propagation в Go (OpenTelemetry):**
      ```go
      // Inject (sender)
      prop := propagation.TraceContext{}
      prop.Inject(ctx, propagation.HeaderCarrier(req.Header))

      // Extract (receiver)
      ctx = prop.Extract(ctx, propagation.HeaderCarrier(req.Header))
      ```
    * **gRPC:** metadata. **Message brokers:** message headers/properties.
    * **Важно:** каждый сервис должен propagate context. Если один сервис "забыл" — trace обрывается.

    </details>

  ---

</details>

---

## 5. Prometheus и Grafana

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №17: [ Как устроен Prometheus? Architecture? ]

    <details>
      <summary>Ответ</summary>

    * **Prometheus** — pull-based TSDB. Сам забирает метрики с таргетов.
    * **Компоненты:**
      * **Prometheus Server** — scrapes metrics, stores в TSDB, executes PromQL.
      * **Exporters** — отдают метрики в `/metrics` (node_exporter, mysqld_exporter, etc.).
      * **Pushgateway** — для push-модели (cron jobs, batch). Прометеус pull'ит из Pushgateway.
      * **Alertmanager** — получает alerts от Prometheus, группирует, отправляет (email, Slack, PagerDuty).
      * **Grafana** — визуализация (dashboard).
    * **Data model:** `metric_name{label1="v1",label2="v2"} value timestamp`.
    * **Scrape:** каждые 15s (по умолчанию). `scrape_interval` в config.
    * **Storage:**
      * Local TSDB (диск). Chunks по 2 часа.
      * Long-term storage: Thanos, Cortex, VictoriaMetrics.
    * **Service discovery:** static configs, Kubernetes SD, Consul, DNS, EC2.

    </details>

  ---

  - Вопрос №18: [ Pull vs Push модель метрик? Плюсы/минусы? ]

    <details>
      <summary>Ответ</summary>

    | | Pull (Prometheus) | Push (StatsD, Datadog) |
    |--|--------------------|-----------------------|
    | Кто инициирует | Server забирает | App отправляет |
    | Firewall | Server → target (нужен доступ) | Target → server (нужен доступ) |
    | Health check | Scrape fail = target down (бесплатный health check) | Нет, нужен отдельный |
    | Batch jobs | Сложно (Pushgateway) | Просто |
    | Serverless | Сложно (нет endpoint) | Просто |
    | Дедупликация | Server знает, что уже собрал | App может отправить дубль |
    | Load balancing | Каждый target scrape отдельно | Может перегрузить server |
    | Authentication | Server → target (mTLS, bearer) | Target → server (API key) |

    * **Prometheus = pull.** Это намеренно: health checking, simpler ops, no data loss on scrape.
    * **Pushgateway** — мост для short-lived jobs. App push'ит метрики → Prometheus pull'ит. Не для
      сервисов.

    </details>

  ---

  - Вопрос №19: [ Основы PromQL. Примеры запросов. ]

    <details>
      <summary>Ответ</summary>

    * **PromQL** — язык запросов Prometheus.
    * **Базовые:**
      * `http_requests_total` — текущее значение.
      * `http_requests_total[5m]` — все значения за 5 минут (range vector).
    * **Rate:**
      * `rate(http_requests_total[5m])` — запросов/сек (для counter).
      * `increase(http_requests_total[1h])` — сколько за час.
    * **Aggregation:**
      * `sum by (service) (rate(http_requests_total[5m]))` — RPS per service.
      * `avg by (method) (http_request_duration_seconds_bucket{le="0.1"})` — avg.
    * **Histogram quantile:**
      * `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` — p99 latency.
    * **Arithmetic:**
      * `rate(http_requests_total{status="500"}[5m]) / rate(http_requests_total[5m])` — error rate.
    * **Comparison:**
      * `rate(http_requests_total{status="500"}[5m]) > 0.1` — больше 0.1 req/s.
    * **Functions:** `rate`, `irate`, `increase`, `histogram_quantile`, `topk`, `avg_over_time`,
      `predict_linear`.

    </details>

  ---

  - Вопрос №20: [ Как работает recording rules и alerting rules в Prometheus? ]

    <details>
      <summary>Ответ</summary>

    * **Recording rules** — предвычисленные метрики. Запускаются периодически, результат сохраняется
      как новая метрика.
      * Зачем: сложный запрос, который используется в нескольких дашбордах. Считать один раз, потом
        использовать.
      ```yaml
      groups:
        - name: example
          rules:
            - record: job:http_requests:rate5m
              expr: sum by (job)(rate(http_requests_total[5m]))
      ```
    * **Alerting rules** — правила, генерирующие alerts.
      ```yaml
            - alert: HighErrorRate
              expr: rate(http_requests_total{status="500"}[5m]) > 0.1
              for: 5m  # должно выполняться 5 минут подряд
              labels:
                severity: critical
              annotations:
                summary: "High error rate on {{ $labels.job }}"
      ```
    * **`for`** — сколько времени условие должно быть true, прежде чем alert сработает. Защита от
      flapping.
    * **Alertmanager** — получает alerts, группирует (по `alertname`, `service`), дедуплицирует,
      отправляет уведомления (inhibition, silencing).

    </details>

  ---

  - Вопрос №21: [ Что такое Thanos? Зачем нужен? ]

    <details>
      <summary>Ответ</summary>

    * **Thanos** — long-term storage и global query для Prometheus.
    * **Проблема:** Prometheus хранит данные локально (диски на одной ноде). Нет HA, нет long-term
      retention, нет global view.
    * **Компоненты Thanos:**
      * **Sidecar** — рядом с Prometheus. Uploads blocks в object storage (S3, GCS).
      * **Store Gateway** — служит метриками из object storage (old data).
      * **Compactor** — downsampling (5m → 1h), compaction.
      * **Query** — единый запрос across all Prometheus instances + Store Gateway. PromQL.
      * **Receive** — push-based ingestion (для remote write).
    * **Преимущества:**
      * **Long-term storage** — S3/GCS вместо локального диска.
      * **Global view** — один Query для всех инстансов.
      * **HA Prometheus** — два Prometheus + Thanos Query → deduplication.
    * **Альтернативы:** VictoriaMetrics, Cortex, Mimir.

    </details>

  ---

</details>

---

## 6. SLO, SLI, SLA

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №22: [ Что такое SLI, SLO, SLA? Чем отличаются? ]

    <details>
      <summary>Ответ</summary>

    * **SLI (Service Level Indicator)** — метрика качества сервиса.
      * Пример: "доля HTTP-ответов со статусом < 500" или "p99 latency < 200ms".
    * **SLO (Service Level Objective)** — целевое значение SLI.
      * Пример: "99.9% запросов за 28 дней успешны" (error budget = 0.1%).
      * Внутренняя цель команды.
    * **SLA (Service Level Agreement)** — контракт с клиентом.
      * Пример: "99.5% uptime в месяц, иначе компенсация $X."
      * SLO всегда строже SLA (буфер безопасности).
    * **Error budget:**
      * SLO 99.9% → error budget = 0.1% = 43.2 мин/мес.
      * Пока error budget не исчерпан — можно рисковать (deploys, features).
      * Исчерпан → freeze, focus на reliability.
    * **Иерархия:** SLI (что меряем) → SLO (цель) → SLA (контракт).

    </details>

  ---

  - Вопрос №23: [ Как выбрать SLO? ]

    <details>
      <summary>Ответ</summary>

    * **Выбор SLI:**
      * **Request-based:** success rate, latency.
      * **Resource-based:** uptime, queue depth.
      * **User-centric:** "удачный поиск" = response < 500ms AND no error.
    * **Выбор target (SLO):**
      * Начать с текущего (baseline) + небольшой запас.
      * 99.9% (three nines) — типичный для web-сервиса. 43 мин downtime/мес.
      * 99.99% (four nines) — дорого. 4.3 мин/мес. Нужна избыточность везде.
      * Не обещать больше, чем нужно. 100% — недостижимо и дорого.
    * **Window:**
      * **Rolling 28-day** — гладко, всегда актуально.
      * **Calendar month** — совпадает с billing, но "сброс" в начале месяца = gaming.
    * **Количество SLO:** 1-3 на сервис. Больше — сложно управлять.
    * **Review:** раз в квартал. Слишком легко → поднять. Слишком сложно → обсудить.

    </details>

  ---

  - Вопрос №24: [ Что такое error budget? Как использовать? ]

    <details>
      <summary>Ответ</summary>

    * **Error budget** = 100% − SLO. При SLO 99.9% → error budget = 0.1%.
    * **Расчёт (rolling 30 дней):**
      * 0.1% от 43200 мин = 43.2 мин допустимого "плохого" времени.
      * Или: 0.1% от 1M запросов = 1000 запросов могут ошибиться.
    * **Использование:**
      * **Green** (budget > 50%) → можно deploys, features, риски.
      * **Yellow** (budget 25-50%) → осторожнее, меньше рисков.
      * **Red** (budget < 25% или исчерпан) → freeze features, focus на reliability, postmortem.
    * **Burn rate:** как быстро тратится budget.
      * `burn_rate = current_error_rate / allowed_error_rate`.
      * burn_rate = 1 → тратится равномерно, хватит на весь window.
      * burn_rate = 10 → тратится в 10 раз быстрее. За 3 дня потратим месячный budget.
    * **Multi-window alerting:** fast burn (1h, 14x) + slow burn (6h, 6x) → alert до исчерпания.

    </details>

  ---

</details>

---

## 7. Алертинг и инциденты

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №25: [ Как писать хорошие alerts? Антипаттерны? ]

    <details>
      <summary>Ответ</summary>

    * **Хороший alert:**
      * **Actionable** — есть действие, которое нужно выполнить. Если действия нет — это dashboard, не
        alert.
      * **Specific** — понятно, что сломалось и где.
      * **Timely** — срабатывает до того, как пользователь заметит.
      * **Low noise** — срабатывает только при реальной проблеме.
    * **Антипаттерны:**
      * Alert на CPU > 80% — что делать? Ничего. Не actionable.
      * Alert на каждую 500-ю ошибку — слишком шумно. Alert на error rate > threshold.
      * Alert на high cardinality (disk space на каждом разделе) — каждый хост → отдельный alert.
      * Alert без runbook — дежурный не знает, что делать.
    * **Правила:**
      * **Symptom-based** — alert на user-visible проблемы (error rate, latency), не на causes (CPU,
        disk).
      * **Pageable vs ticket:** critical → page (PagerDuty). Warning → ticket/Slack.
      * **Runbook** — ссылка на документ с шагами диагностики и лечения.
      * **Aggregation** — группировать related alerts (Alertmanager `group_by`).

    </details>

  ---

  - Вопрос №26: [ Что такое on-call? Как организовать? ]

    <details>
      <summary>Ответ</summary>

    * **On-call** — дежурство инженера для реагирования на инциденты вне рабочего времени.
    * **Организация:**
      * **Rotation** — 1 неделя primary + 1 неделя secondary. Меняются каждую неделю.
      * **Primary** — первый реагирует на page.
      * **Secondary** — backup, escalation если primary не ответил за N минут.
      * **Schedule** — прозрачный, заранее известный. Инструменты: PagerDuty, Opsgenie.
    * **Правила:**
      * **No pages without runbook.**
      * **Follow-the-sun** — распределение по часовым поясам (если команда глобальная).
      * **Compensation** — extra pay / time off за дежурство.
      * **Limit pages** — цель: < 5 pages/week. Больше → alert fatigue → пропустят реальный incident.
      * **Post-incident review** — каждый page, приведший к incident → postmortem.
    * **Health:**
      * Не более 1 недели primary подряд.
      * Защита от burnout: ротация, отгулы, автоматизация (auto-remediation).

    </details>

  ---

  - Вопрос №27: [ Что такое postmortem? Как проводить? ]

    <details>
      <summary>Ответ</summary>

    * **Postmortem** — документ с анализом инцидента после его разрешения.
    * **Цель:** понять **что** произошло, **почему**, и **как предотвратить**. Не найти виновного
      (blameless).
    * **Структура:**
      1. **Summary** — кратко: что случилось, impact (длительность, пользователи).
      2. **Timeline** — хронология событий (UTC). Включая время обнаружения, реакции, фикса.
      3. **Root cause** — техническая причина (не "оператор ошибся", а "не было safeguard").
      4. **Contributing factors** — что усугубило (monitoring gap, slow escalation).
      5. **Action items** — конкретные задачи с owner'ом и deadline. Prevent recurrence.
      6. **Lessons learned** — что работало хорошо, что плохо.
    * **Принципы:**
      * **Blameless** — люди не виноваты, системы/processes виноваты. "Why", не "who".
      * **Concrete action items** — не "будем внимательнее", а "добавим integration test для X".
      * **Track action items** — до закрытия. Не забыть через месяц.
      * **Timely** — postmortem в течение 1-3 дней после инцидента.

    </details>

  ---

  - Вопрос №28: [ Что такое runbook? Каким должен быть? ]

    <details>
      <summary>Ответ</summary>

    * **Runbook** — документ с инструкциями для диагностики и устранения типовых проблем.
    * **Содержание:**
      * **Описание alert** — что значит, насколько критично.
      * **Диагностика** — что проверить (дашборд, логи, команда).
      * **Решение** — пошаговые действия.
      * **Escalation** — кому звонить, если не помогло.
      * **Verification** — как проверить, что проблема решена.
    * **Пример (HighErrorRate):**
      1. Открыть dashboard "Service Overview".
      2. Проверить error logs: `level=error AND service=X` в Loki.
      3. Если новый deploy → откатить: `kubectl rollout undo deployment/X`.
      4. Если БД недоступна → проверить `kubectl get pods -n db`.
      5. Если не помогло → позвать @dba.
    * **Принципы:**
      * **Executable** — конкретные команды, не "проверьте логи".
      * **Up to date** — обновлять при изменениях.
      * **Linked from alert** — Alertmanager annotation со ссылкой.

    </details>

  ---

</details>

---

## 8. OpenTelemetry

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №29: [ Как инструментировать Go-приложение с OpenTelemetry? ]

    <details>
      <summary>Ответ</summary>

    * **Зависимости:**
      ```bash
      go get go.opentelemetry.io/otel
      go get go.opentelemetry.io/otel/sdk/trace
      go get go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc
      ```
    * **Setup:**
      ```go
      func setupTracing(ctx context.Context) (*sdktrace.TracerProvider, error) {
          exporter, err := otlptracegrpc.New(ctx,
              otlptracegrpc.WithEndpoint("otel-collector:4317"),
              otlptracegrpc.WithInsecure(),
          )
          if err != nil { return nil, err }

          tp := sdktrace.NewTracerProvider(
              sdktrace.WithBatcher(exporter),
              sdktrace.WithResource(resource.NewWithAttributes(
                  semconv.SchemaURL,
                  semconv.ServiceName("payment-service"),
              )),
          )
          otel.SetTracerProvider(tp)
          return tp, nil
      }
      ```
    * **Manual span:**
      ```go
      tracer := otel.Tracer("payment")
      ctx, span := tracer.Start(ctx, "processPayment")
      defer span.End()
      span.SetAttributes(attribute.String("order_id", orderID))
      span.AddEvent("payment authorized", trace.WithAttributes(...))
      ```
    * **Auto-instrumentation:** `otelhttp`, `otelgrpc`, `otelsql` — обёртки для HTTP/gRPC/SQL.

    </details>

  ---

  - Вопрос №30: [ Что такое OTel Collector? Зачем нужен? ]

    <details>
      <summary>Ответ</summary>

    * **OTel Collector** — промежуточный компонент между приложением и backend'ами.
    * **Архитектура:**
      * **Receivers** — приём данных (OTLP, Jaeger, Zipkin, Prometheus).
      * **Processors** — обработка (batching, sampling, attributes, filtering).
      * **Exporters** — отправка в backend (Jaeger, Prometheus, Loki, Datadog, S3).
    * **Pipeline:** receiver → processor → exporter.
    * **Зачем:**
      * **Decoupling** — приложение не знает о backend. Смена backend → конфиг collector, не код.
      * **Processing** — sampling, enrichment, filtering вне приложения.
      * **Multi-export** — отправить traces в Jaeger **и** Datadog одновременно.
      * **Buffering** — если backend недоступен, collector буферизует.
      * **Protocol translation** — приём в Jaeger format, экспорт в OTLP.
    * **Deployment:**
      * **Agent** (DaemonSet в K8s) — на каждой ноде, принимает от локальных приложений.
      * **Gateway** (Deployment) — централизованный, финальная обработка + export.

    </details>

  ---

  - Вопрос №31: [ Как связать logs, metrics, traces через OpenTelemetry? ]

    <details>
      <summary>Ответ</summary>

    * **Exemplar** — связь metric → trace:
      * Prometheus поддерживает exemplars (storage-efficient).
      * При spike в metric можно кликнуть → перейти в trace, который соответствует.
      * OTel SDK может прикреплять trace_id к metrics (exemplar).
    * **Trace context в logs:**
      * OTel предоставляет `trace.SpanFromContext(ctx)` для получения `trace_id` и `span_id`.
      * В `slog`: custom handler, добавляющий `trace_id` в каждый log entry.
      ```go
      span := trace.SpanFromContext(ctx)
      logger.Info("processing order",
          "trace_id", span.SpanContext().TraceID().String(),
          "span_id", span.SpanContext().SpanID().String(),
      )
      ```
    * **В Grafana:** trace_id в логе → клик → открыть trace в Jaeger/Tempo. Trace → span → logs by
      trace_id. Единый view.
    * **OTLP logs:** OTel добавляет logs signal. Единый протокол для всех трёх pilars.

    </details>

  ---

  - Вопрос №32: [ Что такое continuous profiling? Зачем нужен? ]

    <details>
      <summary>Ответ</summary>

    * **Continuous profiling** — сбор профилей (CPU, memory, goroutines) в production, непрерывно.
    * **Как работает:**
      * `net/http/pprof` endpoint в Go.
      * Profiler agent (Pyroscope, Parca, Datadog) периодически scrape'ит `/debug/pprof`.
      * Профили агрегируются, хранятся, визуализируются (flame graph).
    * **Зачем:**
      * Найти regression: "latency выросла на 20% после deploy — профиль показывает новая hot path".
      * Memory leaks: heap profile показывает, что растёт.
      * Goroutine leaks: goroutine profile показывает blocked goroutines.
      * Cost optimization: "эта функция потребляет 30% CPU — оптимизировать".
    * **Low overhead:** sampling каждые 10-60s, overhead < 1%. Safe for prod.
    * **Инструменты:** Pyroscope (open source), Parca (eBPF), Datadog Profiler, Grafana Phlare.

    </details>

  ---

</details>

---

## Полезные ресурсы

- [Google SRE Book](https://sre.google/sre-book/table-of-contents/) — SLO, error budget, incidents
- [Prometheus Documentation](https://prometheus.io/docs/)
- [OpenTelemetry](https://opentelemetry.io/docs/)
- [Grafana Loki](https://grafana.com/docs/loki/latest/)
- [Distributed Tracing](https://opentelemetry.io/docs/concepts/observability-primer/)

---

## Связанные файлы

- [go-questions.md](go-questions.md) — Go pprof, runtime metrics
- [redis-devops-questions.md](redis-devops-questions.md) — Prometheus в Docker/K8s
- [system-design-questions.md](system-design-questions.md) — проектирование observable systems
- [linux-internals.md](linux-internals.md) — системные метрики
- [security.md](security.md) — security logging & audit