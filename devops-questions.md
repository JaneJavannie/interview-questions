# Docker, Kubernetes — Вопросы и ответы для Middle/Senior

Файл содержит вопросы по внутреннему устройству Docker и Kubernetes,
а также практические задачи и сценарии эксплуатации.

Уровни сложности:
- ![Static Badge](https://img.shields.io/badge/Middle-yellow) — Middle разработчик
- ![Static Badge](https://img.shields.io/badge/Senior-red) — Senior разработчик

---

# Оглавление

1. [Docker — внутреннее устройство](#1-docker--внутреннее-устройство)
2. [Kubernetes — архитектура и компоненты](#2-kubernetes--архитектура-и-компоненты)
3. [Kubernetes — эксплуатация и паттерны](#3-kubernetes--эксплуатация-и-паттерны)
4. [Практические задачи (Senior)](#4-практические-задачи-senior)

---

## 1. Docker — внутреннее устройство

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Как устроен Docker внутри? Namespaces, cgroups, unionfs ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

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
│  Image layer 1      │  (base OS)
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

### Вопрос №2: [ Чем отличается COPY от ADD в Dockerfile? Лучшие практики ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

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

### Вопрос №3: [ Что такое multi-stage build? Зачем? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

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

### Вопрос №4: [ Как работают Docker networks? Bridge, host, overlay ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

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

### Вопрос №5: [ Что такое система контейнеризации? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

Система контейнеризации — это метод автоматизации развёртывания, упаковки и управления приложениями в легковесных,
портативных «контейнерах». Эти контейнеры можно легко перемещать между различными средами и хостами, что обеспечивает
гибкость и упрощает процесс разработки, тестирования и распределения приложений.

**Основные характеристики:**
- **Изоляция ресурсов:** контейнеры изолированы друг от друга и используют свои файловые системы, сетевые стеки и процессы, что улучшает безопасность и упрощает управление зависимостями.
- **Портативность:** контейнеры включают в себя всё, что необходимо для работы приложения, что делает их портативными и лёгкими для развёртывания на различных платформах и окружениях.
- **Эффективность:** контейнеры сильно оптимизированы для использования системных ресурсов (CPU, память, диск), что делает их более эффективными по сравнению с традиционными виртуальными машинами.
- **Масштабируемость и оркестровка:** системы контейнеризации часто совместимы с инструментами оркестровки, такими как Kubernetes, которые автоматизируют развёртывание, масштабирование и управление контейнерами.
- **Декларативная конфигурация:** в большинстве систем контейнеризации конфигурации описываются декларативно, что упрощает процесс развёртывания и управления.

</details>

---

### Вопрос №6: [ Какие системы контейнеризации и оркестрации знаешь? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

- Docker
- Kubernetes
- Docker Swarm
- OpenShift
- Mesos

</details>

---

### Вопрос №7: [ Docker — плюсы и минусы ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Плюсы:**
- **Портативность:** все зависимости приложения упакованы в контейнер, что облегчает перенос между различными средами.
- **Изоляция:** контейнеры изолированы друг от друга, что улучшает безопасность.
- **Легковесность:** меньше накладных расходов по сравнению с традиционными виртуальными машинами.
- **Масштабируемость:** легко масштабировать и распределять с помощью оркестровщиков, таких как Kubernetes.
- **Экосистема и сообщество:** обширная библиотека готовых образов и большое сообщество.

**Минусы:**
- **Сложность:** может быть сложно для новичков.
- **Безопасность:** по умолчанию контейнеры имеют определённые уровни изоляции, которые могут быть недостаточными для некоторых приложений.

</details>

---

### Вопрос №8: [ Docker Swarm — плюсы и минусы ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Плюсы:**
- **Простота:** проще в установке и использовании по сравнению с Kubernetes.
- **Интеграция с Docker:** тесная интеграция с Docker обеспечивает лёгкость в управлении.
- **Масштабируемость:** хорошо масштабируется, но менее мощно, чем Kubernetes.

**Минусы:**
- **Функциональность:** меньше функций по сравнению с Kubernetes.
- **Экосистема:** меньше дополнительных инструментов и сообщество не такое большое.

</details>

---

### Вопрос №9: [ Чем отличается Docker от виртуализации? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

Docker и виртуализация — это два различных подхода к управлению контейнерами и виртуализации ресурсов.

| Аспект | Docker (контейнеризация) | Виртуализация (VM) |
|--------|--------------------------|---------------------|
| **Уровень виртуализации** | Контейнеризация на уровне ОС; контейнеры делят ядро ОС с хост-системой | Гипервизоры создают ВМ с полными виртуальными ОС; работают на уровне аппаратного обеспечения |
| **Изоляция** | Изолирует приложения и их зависимости, но используется общее ядро ОС | Полная изоляция; каждая ВМ имеет свою собственную копию ОС |
| **Ресурсы** | Эффективно использует ресурсы — контейнеры делят общее ядро, CPU и память | Требует больше ресурсов — каждая ВМ имеет собственное ядро ОС и запускает полную ОС |
| **Запуск** | На основе образов, включающих все необходимые зависимости и настройки | Требует установки ОС для каждой виртуальной машины |
| **Скорость** | Запускается быстрее, так как используется общее ядро ОС | Запуск может занимать больше времени — требуется загрузка и настройка полной ОС |
| **Использование ресурсов** | Эффективно — не требует запуска полных ОС для каждого контейнера | Больший оверхед из-за полных ОС в каждой виртуальной машине |

</details>

---

</details>

---

## 2. Kubernetes — архитектура и компоненты

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №10: [ Как устроена архитектура Kubernetes? Control Plane и Worker ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

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
- **Container Runtime** — containerd, CRI-O (Docker deprecated since K8s 1.20, удалён в 1.24).

**Поток создания Pod:**
1. `kubectl apply` → API Server → etcd;
2. Scheduler watches → выбирает node → обновляет `pod.spec.nodeName`;
3. Kubelet на выбранной node видит новый pod → pulls image → creates containers;
4. Kubelet reports status → API Server → etcd.

</details>

---

### Вопрос №11: [ Что такое Pod? Почему не «контейнер»? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

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

### Вопрос №12: [ Что такое Service в K8s? Виды (ClusterIP, NodePort, LoadBalancer) ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

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

### Вопрос №13: [ Что такое Ingress? Ingress Controller ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

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

### Вопрос №14: [ Что такое etcd? Почему Raft? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

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
- Compression: etcd хранит историю ревизий → нужно `--auto-compaction` (по умолчанию отключено; рекомендуется `--auto-compaction-retention=5` в часах).

</details>

---

</details>

---

## 3. Kubernetes — эксплуатация и паттерны

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №15: [ Что такое Deployment, StatefulSet, DaemonSet, Job/CronJob? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

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
- **Ordered**: создаются последовательно (0→1→2), удаляются в обратном порядке (2→1→0);
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

### Вопрос №16: [ Как работают resources (requests/limits) в K8s? QoS classes ] ![Static Badge](https://img.shields.io/badge/Senior-red)

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
- Limit → CPU quota (throttle при превышении; cgroups v1: `cpu.cfs_quota_us`, v2: `cpu.max`);
- 100m = 100 millicores = 0.1 CPU.

**Memory:**
- Request → учитывается при scheduling (node allocatable);
- Limit → cgroup (`memory.limit_in_bytes` в v1 / `memory.max` в v2);
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

### Вопрос №17: [ Что такое ConfigMap и Secret? Как безопасно передавать конфиги? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

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

### Вопрос №18: [ Что такое HPA, VPA, Cluster Autoscaler? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

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
- Modes: `Off` (recommend only), `Initial` (только при создании), `Recreate` (пересоздаёт pod при изменении рекомендаций). Режим `Auto` устарел и эквивалентен `Recreate`;
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

### Вопрос №19: [ Что такое Taints, Tolerations, Affinity/Anti-Affinity? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

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

### Вопрос №20: [ Как работает K8s rolling update? Rollback ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

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
1. `preStop` hook → sleep 5s (даёт time для deregister из service endpoints);
2. K8s отправляет `SIGTERM` → app должен завершить обработку текущих запросов;
3. `terminationGracePeriodSeconds` → если не завершился → `SIGKILL`.

</details>

---

### Вопрос №21: [ Что такое Helm? Charts ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

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

### Вопрос №22: [ Что такое система оркестрации контейнеров? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

Системы оркестрации контейнеров, такие как Kubernetes, Docker Swarm или Mesos, используются для автоматизации развёртывания, масштабирования
и управления контейнеризованными приложениями.

**Для чего они нужны:**
- **Автоматизация развёртывания:** один раз описав, как должен работать ваш сервис, вы можете автоматически развернуть его на любом числе машин.
- **Масштабирование:** не нужно вручную добавлять или удалять контейнеры. Оркестратор может делать это автоматически в зависимости от нагрузки.
- **Балансировка нагрузки:** оркестраторы могут автоматически распределять входящий трафик между контейнерами одного сервиса.
- **Высокая доступность:** оркестраторы могут перезапускать упавшие контейнеры и перемещать их между хостами.
- **Обновление и откат:** оркестраторы могут обновлять приложения с минимальными простоями, а также откатывать их до предыдущих версий.

Эти возможности делают системы оркестрации ключевым компонентом для современных облачных и микросервисных архитектур.

</details>

---

### Вопрос №23: [ Kubernetes — плюсы и минусы ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Плюсы:**
- **Масштабируемость:** отлично подходит для управления большим числом контейнеров.
- **Самовосстановление:** автоматически перезапускает контейнеры при сбоях.
- **Балансировка нагрузки:** встроенная балансировка нагрузки для распределения трафика.
- **Экосистема:** множество дополнительных инструментов и большое сообщество.

**Минусы:**
- **Сложность:** высокий порог входа и сложная архитектура.
- **Ресурсоёмкость:** требует больших вычислительных ресурсов для запуска.

</details>

---

### Вопрос №24: [ OpenShift — плюсы и минусы ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Плюсы:**
- **Безопасность:** встроенные функции безопасности, такие как секреты и политики доступа.
- **Полнота решения:** предлагает комплексное решение для CI/CD, мониторинга и логирования.
- **Поддержка:** поддерживается Red Hat, что обеспечивает хорошую корпоративную поддержку.

**Минусы:**
- **Сложность:** может быть сложным для новичков.
- **Цена:** бесплатная версия ограничена, полные версии могут быть дорогими.

</details>

---

### Вопрос №25: [ Mesos — плюсы и минусы ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

**Плюсы:**
- **Масштабируемость:** разработан для очень больших кластеров и больших объёмов данных.
- **Гибкость:** поддерживает множество типов рабочих нагрузок, включая не только контейнеры.
- **Высокая производительность:** оптимизирован для высокой производительности и эффективности.

**Минусы:**
- **Сложность:** сложная архитектура и высокий порог входа.
- **Экосистема:** меньше готовых решений и интеграций по сравнению с Kubernetes.

</details>

---

</details>

---

## 4. Практические задачи (Senior)

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Задача №1: [ Zero-downtime deployment в Kubernetes: checklist ] ![Static Badge](https://img.shields.io/badge/Senior-red)

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
### Задача №2: [ Настройка monitoring и alerting для K8s + Redis ] ![Static Badge](https://img.shields.io/badge/Senior-red)

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
          redis_replication_offset{role="slave"}
            - on() group_right() redis_replication_offset{role="master"} > 10000
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
### Задача №3: [ Миграция монолита на микросервисы в K8s (strangler fig pattern) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

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
            pathType: Prefix
            backend:
              service:
                name: user-service
                port: {number: 8080}
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port: {number: 8080}
          - path: /
            pathType: Prefix
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
### Задача №4: [ Troubleshooting: Pod в CrashLoopBackOff. План диагностики ] ![Static Badge](https://img.shields.io/badge/Senior-red)

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
| **137** | SIGKILL (128+9) — часто OOMKilled, но может быть и ручной `kill -9` | Увеличь memory limit / проверь leak / проверь `kubectl describe` на OOMKilled reason |
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
