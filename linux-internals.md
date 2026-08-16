# Linux и операционная система — Вопросы и ответы

Файл содержит вопросы по Linux, операционной системе, процессам, памяти, сети на уровне ОС,
IO, системным вызовам. Эти темы часто встречаются на backend-собеседованиях.

Уровни сложности:
- ![Static Badge](https://img.shields.io/badge/Middle-yellow) — Middle разработчик
- ![Static Badge](https://img.shields.io/badge/Senior-red) — Senior разработчик

---

# Оглавление

1. [Процессы и потоки](#1-процессы-и-потоки)
2. [Память и виртуальная память](#2-память-и-виртуальная-память)
3. [Файловая система и I/O](#3-файловая-система-и-io)
4. [Сеть на уровне ОС](#4-сеть-на-уровне-ос)
5. [Сигналы и межпроцессное взаимодействие (IPC)](#5-сигналы-и-межпроцессное-взаимодействие-ipc)
6. [Производительность и диагностика](#6-производительность-и-диагностика)
7. [Systemd и управление сервисами](#7-systemd-и-управление-сервисами)
8. [Shell и командная строка](#8-shell-и-командная-строка)

---

## 1. Процессы и потоки

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ В чём разница между процессом и потоком? ]

<details>
<summary>Ответ</summary>

* **Процесс** — экземпляр выполняемой программы. Имеет собственное адресное пространство, таблицу файловых
  дескрипторов, PID, окружение.
* **Поток (thread)** — единица выполнения внутри процесса. Потоки одного процесса **разделяют** адресное
  пространство, файловые дескрипторы, но имеют собственные: стек, регистры, PC (program counter), TLS.
* Создание потока дешевле процесса (не нужно копировать page table, не нужен `fork`).
* Переключение контекста между потоками одного процесса дешевле, чем между процессами (same address space,
  не нужно менять TLB).
* В Linux потоки реализованы как "легковесные процессы" (LWP) — `clone()` с флагами `CLONE_VM | CLONE_FILES`.

</details>

---

### Вопрос №2: [ Что происходит при системном вызове fork()? ]

<details>
<summary>Ответ</summary>

* `fork()` создаёт **точную копию** текущего процесса:
  - Новый PID, PPID = PID родителя.
  - Копия адресного пространства (Copy-on-Write — физически не копируется, пока не запишут).
  - Копия таблицы файловых дескрипторов (разделяют file offset/flags).
  - Копия сигналов, окружения, cwd.
* Возвращает:
  - В родителе: PID ребёнка.
  - В ребёнке: 0.
  - При ошибке: -1.
* `fork()` → `exec()` — типичный паттерн: создать копию, сразу заменить новой программой.
* `vfork()` — оптимизация: ребёнок не копирует адресное пространство, сразу делает `exec()`. Не рекомендуется
  (ребёнок не должен менять память родителя).

</details>

---

### Вопрос №3: [ Что такое Copy-on-Write (CoW)? ]

<details>
<summary>Ответ</summary>

* **Copy-on-Write** — оптимизация: при `fork()` страницы памяти **не копируются** физически, а разделяются
  между родителем и ребёнком с пометкой read-only.
* Когда один из процессов пытается **записать** в страницу — page fault → ядро копирует страницу,
  снимает read-only, процесс продолжает запись.
* Позволяет `fork()` не копировать физические страницы данных сразу (экономит память и время), но копирование таблиц страниц всё равно требуется — O(n) по количеству виртуальных страниц.
* Если ребёнок сразу делает `exec()` — страницы вообще не копируются.
* CoW используется и в других местах: `mmap(MAP_PRIVATE)`, std::string в некоторых реализациях C++.

</details>

---

### Вопрос №4: [ Что такое zombie-процесс и как с ним бороться? ]

<details>
<summary>Ответ</summary>

* **Zombie (defunct)** — процесс завершился, но родитель ещё не вызвал `wait()`/`waitpid()`.
  Запись в process table остаётся (PID, exit status, usage stats).
* Zombie не потребляет CPU и память (кроме небольшой записи в process table).
* Если родитель не вызывает `wait()` — zombie остаётся до завершения родителя, затем переходит к `init` (PID 1),
  который вызывает `wait()`.
* Много zombie → исчерпание PID → нельзя создавать новые процессы.
* **Решение**:
  - Родитель должен вызывать `wait()`/`waitpid()`.
  - Игнорировать `SIGCHLD`: `signal(SIGCHLD, SIG_IGN)` — ядро автоматически очищает zombie.
  - Установить обработчик `SIGCHLD`, в нём вызывать `waitpid(-1, &status, WNOHANG)` в цикле.
* Если родитель завис — `kill -9` родителя, zombie уйдут к `init`.

</details>

---

### Вопрос №5: [ Что такое orphan-процесс? ]

<details>
<summary>Ответ</summary>

* **Orphan** — процесс, чей родитель завершился. ОС "переподчиняет" его процессу `init` (PID 1) или
  subreaper'у (если установлен через `prctl(PR_SET_CHILD_SUBREAPER)`).
* В отличие от zombie, orphan **выполняется** нормально, просто его новый родитель — `init`.
* `init` корректно вызывает `wait()`, поэтому orphan не станет zombie.
* **Double fork** — паттерн для демонизации: процесс fork'ает ребёнка, ребёнок fork'ает внука, родитель
  завершается → внук становится orphan → init забирает его. Используется для создания демонов.

</details>

---

### Вопрос №6: [ Чем отличается nice/renice от cgroups? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

* **nice / renice** — устанавливает приоритет планировщика CPU для одного процесса (значение -20..19,
  меньше = выше приоритет). Влияет только на CPU time, не ограничивает память/IO. Относительный приоритет
  среди процессов.
* **cgroups (control groups)** — механизм для ограничения и учёта ресурсов группы процессов:
  - CPU quota (cpu.max) — жёсткий лимит CPU.
  - Memory limit (memory.max) — OOM kill при превышении.
  - IO throttling (io).
  - Network bandwidth — не контролируется через cgroups; используется `tc` (Traffic Control) отдельно.
* nice — простой приоритет для одного процесса. cgroups — изоляция и лимиты для групп (основа контейнеров).
* systemd использует cgroups v2 через `CPUQuota=`, `MemoryMax=`, `TasksMax=`.

</details>

---

### Вопрос №7: [ Что такое context switch и сколько он стоит? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

* **Context switch** — переключение процессора с одного процесса/потока на другой:
  1. Сохранение регистров, PC, SP текущего процесса в PCB (Process Control Block).
  2. Загрузка регистров следующего процесса.
  3. Переключение адресного пространства (page table, TLB flush — если другой процесс).
  4. Возврат в user-space.
* **Стоимость**:
  - Прямые затраты: ~1–10 микросекунд (сохранение/восстановление регистров).
  - Косвенные: TLB flush → cache miss → memory stall. Может быть 10–100µs эффективной потери.
* Переключение между потоками одного процесса дешевле (не нужен TLB flush).
* Переключение user → kernel (syscall) — это **mode switch**, а не context switch (процесс тот же, меняется только режим CPU).
* Много context switches = CPU тратит время на переключение, а не на работу. `vmstat` колонка `cs`.

</details>

---

### Вопрос №8: [ Можно ли убить поток внутри определенного процесса командой kill? ]

<details>
<summary>Ответ</summary>

* Обычно команда kill убивает процессы, а не отдельные потоки. В Linux потоки являются частью процесса и не могут быть убиты независимо от него командой kill.

</details>

---

### Вопрос №9: [ Что такое операционная система? ]

<details>
<summary>Ответ</summary>

* Операционная система — это программное обеспечение, которое управляет аппаратными средствами компьютера и предоставляет различные сервисы для выполнения приложений.

</details>

---

### Вопрос №10: [ Что такое многозадачность? ]

<details>
<summary>Ответ</summary>

* Многозадачность — это способность операционной системы выполнять несколько задач (процессов) одновременно.

</details>

---

### Вопрос №11: [ Что такое ядро ОС? ]

<details>
<summary>Ответ</summary>

* Ядро ОС — это основной компонент операционной системы, который управляет аппаратными и программными ресурсами компьютера.

</details>

---

### Вопрос №12: [ Что такое системный вызов? ]

<details>
<summary>Ответ</summary>

* Системный вызов — это программный интерфейс между ядром операционной системы и пользовательскими программами. 
  Это способ, которым программы могут запрашивать сервисы, предоставляемые операционной системой.

</details>

---

</details>

---

## 2. Память и виртуальная память

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Что такое виртуальная память и зачем она нужна? ]

<details>
<summary>Ответ</summary>

* **Виртуальная память** — каждый процесс видит непрерывное адресное пространство, не зависящее от физической RAM.
* **Page** — единица памяти (обычно 4 KB). Виртуальные страницы маппятся на физические frame'ы через page table.
* **MMU** (Memory Management Unit) — аппаратный блок, транслирующий виртуальный адрес в физический.
* **TLB** (Translation Lookaside Buffer) — кэш для page table entries, чтобы не ходить в RAM на каждый доступ.
* **Зачем**:
  - Изоляция процессов (каждый имеет своё адресное пространство).
  - Swap — можно использовать больше памяти, чем физически есть.
  - Lazy allocation — страницы выделяются при первом доступе.
  - Memory-mapped files (`mmap`).
  - Copy-on-Write.
* Page fault — обращение к странице, которой нет в физической RAM. Ядро загружает её с диска (swap) или
  выделяет новую (demand paging).

</details>

---

### Вопрос №2: [ Что такое page, page frame, page table? ]

<details>
<summary>Ответ</summary>

* **Page** — единица виртуальной памяти (обычно 4 KB на x86_64, но поддерживаются huge pages 2 MB / 1 GB).
* **Page frame** (физическая страница) — единица физической памяти (RAM).
* **Page table** — таблица, маппящая виртуальные страницы на физические frame'ы.
  Многоуровневая (4 уровня на x86_64: PML4 → PDPT → PD → PT) для экономии памяти.
* **Page table entry (PTE)** — запись: физический адрес frame + флаги (present, read/write, user/supervisor,
  dirty, accessed, NX).
* Процесс имеет свою page table. CR3 register указывает на корень.
* **Huge pages** — 2 MB / 1 GB страницы. Меньше PTE → меньше TLB pressure → меньше page walks.

</details>

---

### Вопрос №3: [ Что происходит при нехватке памяти (OOM)? ]

<details>
<summary>Ответ</summary>

* При нехватке RAM ядро:
  1. Освобождает page cache (clean pages — можно перечитать с диска).
  2. Swap (если есть) — выгружает cold pages на диск.
  3. Если swap нет или исчерпан — **OOM Killer**.
* **OOM Killer** — выбирает процесс для убийства по алгоритму (oom_score):
  - Больше памяти = выше шанс быть убитым.
  - Root-процессы имеют штраф (меньше шанс).
  - oom_score_adj (-1000..1000) — ручная настройка (например, -1000 = никогда не убивать).
* В контейнерах (cgroups) — OOM kill убивает процесс внутри cgroup при превышении `memory.max`.
* **`oom_score_adj`** можно выставить в systemd unit: `OOMScoreAdjust=500`.

</details>

---

### Вопрос №4: [ Что такое swap и нужен ли он? ]

<details>
<summary>Ответ</summary>

* **Swap** — область на диске, куда ядро выгружает неактивные (cold) страницы памяти.
* **Зачем**:
  - Расширяет доступную память (медленный диск вместо RAM).
  - Позволяет освободить RAM для более полезных данных (page cache, active pages).
* **swappiness** (0..200, по умолчанию 60) — насколько агрессивно ядро использует swap.
  - 0 — почти не использует (только при OOM).
  - 100 — активно.
  - Для БД-серверов обычно ставят 1–10 (swap вредит производительности БД).
* **Swap file vs swap partition**: файл гибче (можно увеличить/уменьшить). Разницы в производительности практически нет: начиная с ядра 2.6 swap-файл использует прямой доступ к блокам (bypass FS).
* **zswap** — сжатый swap в RAM (быстрее диска, но использует CPU).
* **zram** — блочное устройство в RAM со сжатием, используется как swap.

</details>

---

### Вопрос №5: [ Что такое mmap и в каких случаях используется? ]

<details>
<summary>Ответ</summary>

* **mmap** — системный вызов, отображающий файл или устройство в память процесса.
* Виды:
  - `MAP_SHARED` — изменения видны другим процессам и записываются в файл.
  - `MAP_PRIVATE` — Copy-on-Write, изменения не видны другим и не пишутся в файл.
  - `MAP_ANONYMOUS` — не связан с файлом (просто выделение памяти).
* **Преимущества**:
  - Нет `read`/`write` syscalls на каждый доступ — page fault обрабатывает ядро.
  - Kernel может использовать page cache эффективно.
  - Несколько процессов могут разделять `MAP_SHARED` отображение (IPC).
* **Использование**:
  - Загрузка больших файлов (БД, index files).
  - Shared memory IPC.
  - `malloc` для больших аллокаций (glibc malloc использует mmap для блоков > 128 KB).
  - Execution of binaries (ELF segments mmap'ятся).
* **Минусы**: page fault на первом доступе (latency), сложно контролировать flushing (`msync`).

</details>

---

### Вопрос №6: [ RSS vs VSZ vs PSS — что показывают? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

* **VSZ (Virtual Size)** — всё виртуальное адресное пространство процесса. Включает mmap, swap, uncommitted pages.
  Может быть огромным (100+ GB) и не означает реального потребления.
* **RSS (Resident Set Size)** — физические страницы в RAM. Включает shared libraries.
  Не учитывает, что shared pages могут считаться для нескольких процессов (double counting).
* **PSS (Proportional Set Size)** — RSS, но shared pages делятся поровну между процессами.
  Точнее отражает реальное потребление.
* **USS (Unique Set Size)** — только private pages (не shared). Сколько памяти освободится, если процесс завершится.
* `ps` показывает VSZ и RSS. `smem` показывает PSS и USS.

</details>

---

</details>

---

## 3. Файловая система и I/O

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Что такое file descriptor и как он работает? ]

<details>
<summary>Ответ</summary>

* **File descriptor (fd)** — целое число (индекс в таблице fd процесса), указывающее на открытый файл/сокет/pipe.
* Стандартные fd:
  - 0 — stdin
  - 1 — stdout
  - 2 — stderr
* Каждый процесс имеет таблицу fd → file table (system-wide) → inode table.
* `open()` возвращает fd, `read(fd, ...)`, `write(fd, ...)`, `close(fd)`.
* Лимиты:
  - `ulimit -n` (soft) — обычно 1024.
  - `/proc/sys/fs/file-max` (system-wide).
  - Можно увеличить: `ulimit -n 1000000` или через systemd `LimitNOFILE=`.
* При `fork()` ребёнок наследует копию таблицы fd. `close-on-exec` (FD_CLOEXEC) — закрывает fd при `exec()`.

</details>

---

### Вопрос №2: [ Что такое inode? ]

<details>
<summary>Ответ</summary>

* **inode** (index node) — структура данных в файловой системе, описывающая файл/директорию.
* inode содержит:
  - Тип файла (regular, directory, symlink, device...).
  - Права доступа (rwx for user/group/other).
  - Owner UID, GID.
  - Размер.
  - Timestamps (atime, mtime, ctime).
  - Список блоков на диске (direct, single-indirect, double-indirect, triple-indirect).
  - Количество hard links.
* inode **не** содержит имя файла — имя хранится в директории (mapping: name → inode number).
* `ls -i` — показать inode numbers. `df -i` — использование inodes (может закончиться, даже если есть место).
* Hard link — ещё одно имя, указывающее на тот же inode. Нельзя создать на директорию или на другую файловую систему.

</details>

---

### Вопрос №3: [ Буферизованный vs небуферизованный I/O ]

<details>
<summary>Ответ</summary>

* **Небуферизованный I/O (без user-space буфера)** — `read()`/`write()` через syscall.
  Данные идут: user buffer → kernel page cache → disk (через `fsync` или writeback).
* **Буферизованный I/O** — stdio (C `fread`/`fwrite`, Go `bufio`).
  Дополнительный буфер в user-space. `fwrite` копирует в user buffer, `flush` вызывает `write()`.
* **Page cache** (kernel) — ядро кэширует файловые блоки в RAM. `read()` может отдать из cache без дискового I/O.
* `O_DIRECT` — обход page cache (прямая запись на диск). Используется в БД (PostgreSQL, Oracle), которые
  сами управляют кэшем.
* `O_SYNC` — `write()` не возвращается, пока данные физически не на диске (медленно, но надёжно).
* `fsync(fd)` — сброс page cache для конкретного fd на диск. `fdatasync` — без metadata (быстрее).

</details>

---

### Вопрос №4: [ Что такое epoll и чем отличается от select/poll? ]

<details>
<summary>Ответ</summary>

* **select/poll/epoll** — механизмы мультиплексирования I/O (один поток ждёт несколько fd).
* **select**:
  - Передаёт bitmap fd_set (ограничение FD_SETSIZE=1024).
  - Linear scan всех fd каждый раз O(n).
  - Возвращает только "готовых есть", нужно перебирать все.
* **poll**:
  - Нет лимита 1024 (массив структур).
  - Всё равно O(n) scan.
* **epoll** (Linux):
  - `epoll_create` → `epoll_ctl` (add/modify/delete) → `epoll_wait`.
  - O(1) для получения готовых fd (red-black tree + ready list).
  - **Level-triggered** (по умолчанию): сообщает, пока fd готов (есть данные).
  - **Edge-triggered** (`EPOLLET`): сообщает только при **изменении** состояния. Нужно читать до `EAGAIN`.
  - Используется в nginx, Redis, Go runtime (netpoller), Netty.
* macOS аналог — **kqueue**. Windows — **IOCP**.

</details>

---

### Вопрос №5: [ synchronous vs asynchronous I/O, blocking vs non-blocking ]

<details>
<summary>Ответ</summary>

| | Blocking | Non-blocking |
|---|---|---|
| **Synchronous** | `read()` ждёт данных | `read()` возвращает `EAGAIN`, poll позже |
| **Asynchronous** | — | `aio_read()` / `io_uring` — уведомляет по готовности |

* **Blocking** — `read()` блокирует поток, пока данные не доступны.
* **Non-blocking** — `read()` сразу возвращает `EAGAIN`/`EWOULDBLOCK`, если данных нет.
  Нужно poll'ить (select/poll/epoll).
* **Asynchronous I/O** — инициировать I/O, получить уведомление по завершении (callback/event).
  Linux: `io_uring` (современный), POSIX AIO (устаревший). Windows: IOCP.
* **Reactor pattern** (epoll + non-blocking) — event loop, обработка готовых I/O.
* **Proactor pattern** (IOCP, io_uring) — initiate I/O, completion callback.

</details>

---

### Вопрос №6: [ Что такое inotify? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

* **inotify** — подсистема Linux для мониторинга изменений файловой системы.
* `inotify_init()` → `inotify_add_watch(path, mask)` → `read()` события.
* События: `IN_CREATE`, `IN_MODIFY`, `IN_DELETE`, `IN_MOVE`, `IN_CLOSE_WRITE`, ...
* Используется: file sync (Syncthing), hot reload (webpack, Vite), log shipping, autotest runners.
* Ограничения: не работает рекурсивно (нужно add_watch на каждую директорию), лимит watches
  (`/proc/sys/fs/inotify/max_user_watches`, по умолчанию 8192 — часто увеличивают).

</details>

---

### Вопрос №7: [ Что такое драйвер? ]

<details>
<summary>Ответ</summary>

* Драйвер — это специальная программа, позволяющая операционной системе взаимодействовать с конкретным аппаратным обеспечением.

</details>

---

</details>

---

## 4. Сеть на уровне ОС

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Что происходит при TCP-рукопожатии (3-way handshake)? ]

<details>
<summary>Ответ</summary>

1. **SYN** — клиент → сервер: "хочу подключиться, мой initial sequence number = X".
   Флаг SYN=1, seq=X.
2. **SYN-ACK** — сервер → клиент: "ок, мой ISN = Y, ack=X+1".
   Флаг SYN=1, ACK=1, seq=Y, ack=X+1.
3. **ACK** — клиент → сервер: "принято, ack=Y+1".
   Флаг ACK=1, seq=X+1, ack=Y+1.
* После этого соединение **established**.
* Почему 3 шага, а не 2? Стороны должны синхронизировать sequence numbers в обе стороны. SYN-ACK подтверждает
  SYN клиента и одновременно отправляет свой SYN. ACK подтверждает SYN сервера.
* **SYN flood** — атака: много SYN без завершения handshake. Защита: SYN cookies (сервер не выделяет
  состояние до ACK).

</details>

---

### Вопрос №2: [ Что происходит при TCP-разрыве (4-way handshake / FIN)? ]

<details>
<summary>Ответ</summary>

* **FIN** — сторона A: "я закончил отправлять". (half-close: A не шлёт, но может принимать.)
* **ACK** — сторона B подтверждает FIN.
* (B может ещё отправлять данные — B в CLOSE_WAIT, A в FIN_WAIT_2.)
* **FIN** — сторона B: "я тоже закончил".
* **ACK** — сторона A подтверждает. A → TIME_WAIT (2*MSL), B → CLOSED.
* **TIME_WAIT** (2*MSL ≈ 60 сек):
  - Чтобы последний ACK дошёл до B (если потерялся — B resend FIN, A resend ACK).
  - Чтобы старые сегменты из этого соединения не попали в новое соединение с тем же 4-tuple.
* **Много TIME_WAIT** — исчерпание портов. Решения: `SO_REUSEADDR`, `tcp_tw_reuse`, longer port range,
  keep-alive.

</details>

---

### Вопрос №3: [ Что такое TCP keepalive и чем отличается от HTTP keep-alive? ]

<details>
<summary>Ответ</summary>

* **TCP keepalive** — на уровне TCP. Ядро отправляет empty ACK-сегменты для проверки, жива ли другая сторона.
  - Параметры: `tcp_keepalive_time` (по умолчанию 7200 сек = 2 часа), `tcp_keepalive_intvl` (75 сек),
    `tcp_keepalive_probes` (9 раз).
  - Включается per-socket: `SO_KEEPALIVE`.
  - Назначение: обнаружить "мёртвые" соединения (например, NAT timeout, обрыв кабеля).
* **HTTP keep-alive** — на уровне HTTP. Соединение **не закрывается** после одного запроса, а используется
  для нескольких (header `Connection: keep-alive`). Избегает overhead TCP handshake на каждый запрос.
  - `Keep-Alive: timeout=5, max=100`.
* Разные уровни: TCP keepalive — проверка живости, HTTP keep-alive — переиспользование соединения.

</details>

---

### Вопрос №4: [ Что такое backlog в TCP и как связан с listen()? ]

<details>
<summary>Ответ</summary>

* `listen(fd, backlog)` — backlog — размер **accept queue** (завершённых, но не accepted соединений).
  SYN queue управляется отдельно через `tcp_max_syn_backlog`.
* В Linux: `tcp_max_syn_backlog` — размер SYN queue. `somaxconn` — максимальный accept queue.
  Реальный backlog = min(`listen backlog`, `somaxconn`).
* Если accept queue полон — новые соединения: 
  - Если `tcp_abort_on_overflow=1` → RST.
  - По умолчанию → финальный ACK дропается, сервер ретранслирует SYN-ACK (до `tcp_synack_retries` раз).
* При high load с медленным `accept()` — соединения дропаются. Решение: увеличить backlog, быстрее accept'ить
  (передать в worker pool).

</details>

---

### Вопрос №5: [ NAT, SNAT, DNAT — в чём разница? ]

<details>
<summary>Ответ</summary>

* **NAT** (Network Address Translation) — подмена адресов в пакетах.
* **SNAT** (Source NAT) — подмена source IP. Используется для выхода в интернет из private сети
  (MASQUERADE — частный случай SNAT с динамическим IP интерфейса).
  ```
  iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE
  ```
* **DNAT** (Destination NAT) — подмена destination IP. Используется для port forwarding.
  ```
  iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.0.0.5:8080
  ```
* **PAT** (Port Address Translation) — много внутренних IP → один внешний IP с разными портами.
  Это то, что делает домашний роутер.

</details>

---

### Вопрос №6: [ Что такое conntrack и почему может переполниться? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

* **conntrack** (connection tracking) — модуль netfilter, отслеживающий состояния всех сетевых соединений.
  Используется для stateful firewall (`-m state --state ESTABLISHED`) и NAT.
* Каждое соединение (4-tuple: src_ip, src_port, dst_ip, dst_port) → запись в conntrack table.
* Лимит: `nf_conntrack_max` (по умолчанию ~65536, зависит от RAM).
* При переполнении — новые соединения дропаются, в логе: `nf_conntrack: table full, dropping packet`.
* Решение:
  - Увеличить `nf_conntrack_max`.
  - Использовать `NOTRACK` для трафика, не требующего state (например, raw UDP).
  - Оптимизировать timeout'ы: `nf_conntrack_tcp_timeout_established` (по умолчанию 5 дней!).
* Проблема актуальна для high-traffic серверов, особенно с большим числом short-lived соединений.

</details>

---

</details>

---

## 5. Сигналы и межпроцессное взаимодействие (IPC)

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Какие основные сигналы в Linux и что они делают? ]

<details>
<summary>Ответ</summary>

| Сигнал | № | Действие по умолчанию | Описание |
|--------|---|----------------------|----------|
| SIGHUP | 1 | Terminate | Hangup (закрытие терминала). Часто используется для reload конфига. |
| SIGINT | 2 | Terminate | Interrupt (Ctrl+C). |
| SIGQUIT | 3 | Core dump | Quit (Ctrl+\). |
| SIGILL | 4 | Core dump | Illegal instruction. |
| SIGABRT | 6 | Core dump | Abort (от `abort()`, `assert`). |
| SIGFPE | 8 | Core dump | Floating point exception (div by zero). |
| SIGKILL | 9 | Terminate | Kill (нельзя перехватить/игнорировать). |
| SIGSEGV | 11 | Core dump | Segmentation fault. |
| SIGPIPE | 13 | Terminate | Write to broken pipe. |
| SIGALRM | 14 | Terminate | Timer alarm (`alarm()`). |
| SIGTERM | 15 | Terminate | Termination (default for `kill`). |
| SIGSTOP | 19 | Stop | Stop (нельзя перехватить/игнорировать). |
| SIGCONT | 18 | Continue | Continue after stop. |
| SIGCHLD | 17 | Ignore | Child process terminated. |
| SIGUSR1 | 10 | Terminate | User-defined. |
| SIGUSR2 | 12 | Terminate | User-defined. |

* `kill -9` = SIGKILL (нельзя перехватить). `kill -15` = SIGTERM (можно перехватить, graceful shutdown).
* Go runtime перехватывает SIGINT/SIGTERM по умолчанию (если не обработан — программа завершается с exit code 2).

</details>

---

### Вопрос №2: [ Какие виды IPC (межпроцессное взаимодействие) существуют? ]

<details>
<summary>Ответ</summary>

* **Pipes** — `pipe()` (unnamed, parent-child), `mkfifo()` (named, any process).
  Однонаправленный поток байт. Буфер ограничен (64 KB по умолчанию).
* **Shared memory** (`shmget`/`shmat` или `mmap(MAP_SHARED)`) — самый быстрый IPC (zero-copy).
  Нужна синхронизация (семафоры/мьютексы).
* **Message queues** — `msgget`/`msgsnd`/`msgrcv` (System V) или `mq_open` (POSIX). Структурированные сообщения.
* **Semaphores** — `semget`/`semop` (System V) или `sem_open` (POSIX). Для синхронизации.
* **Unix domain sockets** (`AF_UNIX`) — сокеты внутри ядра. Надёжные, двунаправленные, поддерживают
  SCM_RIGHTS (передача fd). Используются Docker, X11, systemd.
* **Signals** — простые уведомления (только номер сигнала, без данных). Не подходят для передачи данных.
* **Eventfd / timerfd** — lightweight уведомления (event counter) и таймеры через fd (можно poll/epoll).

</details>

  -----

### Вопрос №3: [ Что такое SIGPIPE и как с ним бороться в Go? ]

<details>
<summary>Ответ</summary>

* **SIGPIPE** — отправляется процессу, когда он пишет в pipe/socket, читающий конец которого закрыт.
* По умолчанию — процесс завершается.
* В Go: runtime перехватывает SIGPIPE и вместо завершения возвращает `io.ErrClosedPipe` / `broken pipe` error
  для записей в socket. Для записей в pipe (stdout/stderr) — стандартное поведение (завершение).
* `signal.Ignore(syscall.SIGPIPE)` — игнорировать (тогда `write` вернёт `EPIPE` error).
* На практике: при записи в HTTP response writer, если клиент закрыл соединение, Go вернёт ошибку,
  а не завершит процесс.

</details>

---

### Вопрос №4: [ Как реализовать graceful shutdown приложения? ]

<details>
<summary>Ответ</summary>

* **Graceful shutdown** — корректное завершение: остановить приём новых запросов, дождаться завершения
  текущих, закрыть соединения, flush buffers, освободить ресурсы.
* Сигналы: SIGTERM (от systemd/k8s), SIGINT (Ctrl+C).
* **В Go**:

```go
ctx, stop := signal.NotifyContext(context.Background(), 
    syscall.SIGINT, syscall.SIGTERM)
defer stop()

server := &http.Server{Addr: ":8080"}
go server.ListenAndServe()

<-ctx.Done()  // ждём сигнала
shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
server.Shutdown(shutdownCtx)  // остановить, дождаться текущих
// закрыть БД-соединения, flush логи, и т.д.
```

* В k8s: Pod получает SIGTERM, через `terminationGracePeriodSeconds` (30 сек по умолчанию) — SIGKILL.
* В systemd: `TimeoutStopSec=` (90 сек по умолчанию), затем SIGKILL.
* Важно: установить `readiness probe = false` **до** начала shutdown, чтобы LB перестал слать трафик.

</details>

---

</details>

---

## 6. Производительность и диагностика

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Какие инструменты диагностики производительности вы знаете? ]

<details>
<summary>Ответ</summary>

* **CPU**:
  - `top` / `htop` — процессы по CPU.
  - `vmstat` — context switches, interrupts, run queue.
  - `mpstat` — CPU per-core.
  - `perf` — profiling (PMU events, call stacks).
  - `pidstat` — per-process CPU usage.
* **Память**:
  - `free -h` — RAM usage.
  - `vmstat` — swap, page faults.
  - `smem` — PSS/USS per process.
  - `/proc/meminfo`, `/proc/<pid>/status`.
* **I/O**:
  - `iostat` — I/O per device (await, svctm, %util).
  - `iotop` — per-process I/O.
  - `biolatency` (bcc/eBPF) — I/O latency histogram.
* **Сеть**:
  - `sar -n DEV` — throughput per interface.
  - `tcpdump` / `wireshark` — packet capture.
  - `ss` / `netstat` — socket states.
  - `iperf3` — bandwidth testing.
* **Tracing**:
  - `strace` — syscall tracing (медленный!).
  - `ltrace` — library call tracing.
  - `bpftrace` / `bcc` — eBPF tracing (production-safe, low overhead).
  - `ftrace` — kernel function tracing.

</details>

---

### Вопрос №2: [ Что такое load average и как его интерпретировать? ]

<details>
<summary>Ответ</summary>

* **Load average** — среднее число процессов в состоянии **runnable** (running on CPU) + **uninterruptible sleep**
  (D-state, waiting for I/O) за 1, 5, 15 минут.
* Показывается: `uptime`, `top`, `/proc/loadavg`.
* LA = 1.0 на 1-CPU системе = 100% утилизация.
* LA = 4.0 на 4-CPU системе = 100% утилизация.
* LA > кол-ва CPU → процессы ждут CPU (CPU-bound) или I/O (I/O-bound).
* **Важно**: LA включает D-state (I/O wait). Высокий LA при низком CPU = I/O bottleneck.
* `vmstat` колонки `r` (runnable) и `b` (blocked) — моментальные значения.

</details>

---

### Вопрос №3: [ Как найти причину высокой нагрузки на сервер? ]

<details>
<summary>Ответ</summary>

* **Шаг 1: top/htop** — понять, что грузит (CPU, memory, I/O). Сортировка по CPU%, MEM%.
* **Шаг 2: vmstat 1** — `r` > CPU count → CPU-bound. `b` > 0 → I/O-bound. `si`/`so` > 0 → swapping.
* **Шаг 3: iostat -xz 1** — `%util` ~100% → диск перегружен. `await` > `svctm` → очередь.
* **Шаг 4: ss -s** / `netstat` — много ESTABLISHED/TIME_WAIT → проверить connection handling.
* **Шаг 5: perf top** — где CPU тратит время (функция/системный вызов).
* **Шаг 6: strace -c -p <pid>** — какие syscalls и сколько времени (осторожно — замедляет).
* **Шаг 7: bpftrace / bcc tools** — production-safe tracing.
* **Шаг 8: dmesg** — сообщения ядра (OOM, errors, warnings).
* **Шаг 9: /proc** — `/proc/<pid>/status`, `/proc/<pid>/io`, `/proc/<pid>/fd`.

</details>

---

### Вопрос №4: [ Что такое strace и когда использовать? ]

<details>
<summary>Ответ</summary>

* **strace** — tracing системных вызовов и сигналов процесса.
* `-c` — summary (сколько времени в каждом syscall).
* `-e trace=openat,read,write` — фильтр по syscall.
* `-p <pid>` — attach к работающему процессу.
* `-f` — следить за children (fork).
* `-T` — время каждого syscall.
* **Когда использовать**:
  - "Программа зависла" — посмотреть, на каком syscall блокируется.
  - "Программа не может открыть файл" — посмотреть openat и errno.
  - "Медленный запуск" — `-c` покажет, где время.
* **Осторожно**: strace использует ptrace, который замедляет процесс в 2–10 раз. Не использовать на production
  под нагрузкой. Альтернатива: eBPF (`bpftrace`, `bcc`).

</details>

---

### Вопрос №5: [ Что такое eBPF и почему это важно? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **eBPF** (extended Berkeley Packet Filter) — механизм загрузки sandboxed программ в ядро Linux.
* Программа eBPF компилируется в bytecode, проходит верификатор (safety check), JIT-компилируется в native.
* **Применения**:
  - **Tracing/profiling**: tracepoints, kprobes, uprobes — наблюдение за ядром и user-space.
  - **Networking**: XDP (eXpress Data Path) — packet processing на уровне драйвера. TC (Traffic Control).
  - **Security**: seccomp-bpf, AppArmor, Falco — мониторинг и блокировка.
  - **Observability**: bcc tools, bpftrace, Hubble, Cilium.
* **Преимущества**:
  - Production-safe (low overhead, нет замедления как у strace).
  - Не требует перекомпиляции ядра или модулей.
  - Гибкий — можно trace'ить что угодно.
* **Инструменты**: bcc (Python + BPF), bpftrace (one-liners), Cilium (networking/security),
  Pixie (k8s observability).

</details>

---

### Вопрос №6: [ Что такое CPU-bound и IO-bound задачи? ]

<details>
<summary>Ответ</summary>

* Задачи, выполняемые в программе или системе, часто классифицируются как CPU-bound или IO-bound, в зависимости от того, 
  какие ресурсы являются основными узкими местами для их выполнения.
* CPU-bound задачи:
  Это задачи, для которых основное ограничение — это скорость процессора (CPU). Они требуют много вычислительных ресурсов и 
  часто занимают процессор интенсивными вычислениями.
  Такие задачи будут быстрее выполняться на более быстром процессоре или при использовании оптимизированных алгоритмов.
  Примеры включают в себя сложные математические вычисления, рендеринг графики, обработку данных и так далее.
* IO-bound задачи:
  Это задачи, для которых основное ограничение — это скорость ввода/вывода (I/O), такая, как операции чтения/записи на диск, 
  сетевые операции или обращение к базе данных. Улучшение производительности таких задач обычно достигается путем оптимизации 
  системы I/O, уменьшения количества I/O операций или асинхронного выполнения I/O, чтобы другие части программы могли продолжать 
  работу, пока I/O операция завершается.
  Примеры включают в себя загрузку файлов, запросы к базе данных или сетевые запросы.
  В реальном мире многие задачи являются комбинацией CPU-bound и IO-bound операций. Определение типа вашей задачи может помочь вам определить, какие оптимизации могут дать наилучший результат. Например, добавление дополнительных ядер процессора может помочь с CPU-bound задачами, в то время как асинхронное программирование или использование быстрого SSD может помочь с IO-bound задачами.

</details>

---

</details>

---

## 7. Systemd и управление сервисами

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Что такое systemd и его основные компоненты? ]

<details>
<summary>Ответ</summary>

* **systemd** — система инициализации (init system) и service manager для Linux. PID 1.
* **Компоненты**:
  - **systemd** (PID 1) — управляет service units.
  - **journald** — структурированный логгер (binary journal, `journalctl`).
  - **networkd** — управление сетью.
  - **resolved** — DNS resolver.
  - **logind** — управление сессиями.
  - **timedated**, **hostnamectl**, **localectl**, **timedatectl** — системные настройки.
  - **udev** — device management.
* **Units** — типы ресурсов: `.service`, `.socket`, `.timer`, `.target`, `.mount`, `.device`, `.slice`.
* `systemctl` — управление units (start, stop, enable, disable, status).
* `journalctl -u <service>` — логи сервиса.

</details>

---

### Вопрос №2: [ Как написать systemd unit файл для сервиса? ]

<details>
<summary>Ответ</summary>

```ini
[Unit]
Description=My Backend Service
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yaml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s
TimeoutStopSec=30s

# Ограничения ресурсов
MemoryMax=2G
CPUQuota=200%
LimitNOFILE=65536

# Безопасность
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/myapp /var/log/myapp

# Окружение
Environment=GIN_MODE=release
EnvironmentFile=/etc/myapp/env

[Install]
WantedBy=multi-user.target
```

* `Type=simple` — процесс не fork'ает (foreground). `Type=forking` — процесс fork'ает и завершается.
* `Restart=always` — перезапускать всегда. `on-failure` — при ненулевом exit code, убийстве сигналом или таймауте.
* `After` + `Requires` — запуск после зависимостей. `Wants` — желательно, но не обязательно.

</details>

---

### Вопрос №3: [ Чем systemd timer лучше cron? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

* **systemd timer** — `.timer` unit, запускает `.service` по расписанию.
* **Преимущества над cron**:
  - Логи в journald (с контекстом: exit code, stdout/stderr, resource usage).
  - Зависимости: `After=network.target`.
  - Resource control: `MemoryMax`, `CPUQuota`.
  - Случайный offset (`RandomizedDelaySec=10m`) — избегать thundering herd.
  - `OnBootSec=` — запуск через N секунд после загрузки.
  - Persistent: `Persistent=true` — если пропустили (система была выключена), запустит при включении.
  - Мониторинг: `systemctl list-timers`.
* **cron** — проще, универсален, но: нет логов, нет зависимостей, нет resource limits, нет retry.
* На практике: для production — systemd timers, для простых задач — cron.

</details>

---

</details>

---

## 8. Shell и командная строка

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Чем отличается bash от sh? ]

<details>
<summary>Ответ</summary>

* **sh** — POSIX shell. Стандарт, минимальный набор возможностей. На разных системах `sh` → разные реализации
  (на Debian — dash, на macOS — bash в POSIX mode).
* **bash** (Bourne Again Shell) — расширенный sh. Массивы, `[[ ]]`, process substitution `<(...)`,
  `$( )`, brace expansion `{1..10}`, associative arrays.
* **Другие shells**:
  - **zsh** — расширенный bash, лучший autocomplete, spelling correction.
  - **fish** — не POSIX, user-friendly, синтаксис отличается.
  - **dash** — минималистичный, быстрый, используется для `/bin/sh` в Debian/Ubuntu.
* Для скриптов: использовать `#!/bin/sh` если не нужны bash-фичи (более портативно). Иначе `#!/usr/bin/env bash`.
* **set -euo pipefail** — best practice для bash скриптов:
  - `-e` — exit при ошибке.
  - `-u` — error при unset variable.
  - `-o pipefail` — pipe возвращает ошибку, если любая команда упала.

</details>

---

### Вопрос №2: [ Как найти процесс, который слушает определённый порт? ]

<details>
<summary>Ответ</summary>

```bash
# ss (современный, рекомендуется)
ss -tlnp | grep :8080

# lsof
lsof -i :8080

# netstat (устаревший)
netstat -tlnp | grep :8080

# fuser
fuser 8080/tcp
```

* `ss` быстрее netstat (не сканирует /proc).
* `-t` — TCP, `-u` — UDP, `-l` — listening, `-n` — numeric (no DNS), `-p` — process.

</details>

---

### Вопрос №3: [ Как найти файлы, модифицированные за последние N минут? ]

<details>
<summary>Ответ</summary>

```bash
# Изменённые за последние 30 минут
find /var/log -mmin -30 -type f

# Изменённые более 7 дней назад
find /tmp -mtime +7 -type f

# Изменённые сегодня
find /home -daystart -mtime 0 -type f
```

* `-mmin` — modification time в минутах. `-` = менее, `+` = более.
* `-mtime` — в днях (24-часовых периодах).
* `-type f` — только файлы. `-type d` — только директории.

</details>

---

### Вопрос №4: [ Что такое exit code и зачем $? ]

<details>
<summary>Ответ</summary>

* **Exit code** — целое число (0–255), которое процесс возвращает при завершении.
* **0** — успех. **1–255** — ошибка.
* Стандартные конвенции (sysexits.h):
  - 0 — success
  - 1 — generic error
  - 2 — misuse
  - 126 — command not executable
  - 127 — command not found
  - 128+N — killed by signal N (например, 137 = SIGKILL, 130 = SIGINT)
* `$?` — exit code последней команды в shell.
* `set -e` — скрипт завершается при ненулевом exit code любой команды.
* `cmd1 || cmd2` — cmd2 выполнится, если cmd1 завершился с ошибкой.
* `cmd1 && cmd2` — cmd2 выполнится, только если cmd1 успешно.

</details>

---

### Вопрос №5: [ Что такое umask? ]

<details>
<summary>Ответ</summary>

* **umask** — маска, определяющая права по умолчанию для новых файлов и директорий.
* Формула: `actual_permissions = requested_permissions & ~umask`.
* Обычно umask = `022`:
  - Файл: `0666 & ~022 = 0644` (rw-r--r--).
  - Директория: `0777 & ~022 = 0755` (rwxr-xr-x).
* Для безопасной настройки: umask = `077` → файлы `0600`, директории `0700` (только owner).
* `umask` команда — посмотреть/установить. `umask -S` — symbolic format.
* Устанавливается в `~/.bashrc`, `/etc/profile`, или в systemd unit: `UMask=0077`.

</details>

---

</details>

---
