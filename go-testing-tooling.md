# Go — Тестирование и инструментирование

Файл содержит вопросы по тестированию в Go, бенчмаркам, мокам, профилированию,
Go Modules и антипаттернам.

Уровни сложности:
- ![Static Badge](https://img.shields.io/badge/Middle-yellow) — Middle разработчик
- ![Static Badge](https://img.shields.io/badge/Senior-red) — Senior разработчик

---

# Оглавление

1. [Тестирование — основы](#1-тестирование--основы)
2. [Бенчмарки и покрытие](#2-бенчмарки-и-покрытие)
3. [Моки и тестовые двойники](#3-моки-и-тестовые-двойники)
4. [Go Modules и зависимости](#4-go-modules-и-зависимости)
5. [Профилирование и отладка](#5-профилирование-и-отладка)
6. [Антипаттерны и лучшие практики](#6-антипаттерны-и-лучшие-практики)

---

## 1. Тестирование — основы

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Какую сигнатуру должен иметь тест в Go? ]

<details>
<summary>Ответ</summary>

```go
func TestXxx(t *testing.T) {
    // ...
}
```

* Имя функции должно начинаться с `Test`, затем идёт имя (с заглавной буквы).
* Файл теста должен иметь суффикс `_test.go` и быть в том же пакете (или `package_test`).
* `testing.T` предоставляет методы: `t.Error`, `t.Fatal`, `t.Skip`, `t.Run` (сабтесты), `t.Parallel`.

</details>

---

### Вопрос №2: [ Чем отличаются t.Error и t.Fatal? ]

<details>
<summary>Ответ</summary>

* `t.Error` / `t.Errorf` — отмечает тест как проваленный, но **продолжает** выполнение.
* `t.Fatal` / `t.Fatalf` — отмечает тест как проваленный и **немедленно останавливает** выполнение текущего теста
  (вызывает `runtime.Goexit`).
* `t.Fatal` стоит использовать когда продолжение не имеет смысла (например, не удалось создать ресурс).
* В сабтестах `t.Fatal` останавливает только текущий сабтест, а не весь родительский тест.

</details>

---

### Вопрос №3: [ Что такое сабтесты (subtests) и для чего они нужны? ]

<details>
<summary>Ответ</summary>

* Сабтесты создаются через `t.Run(name, func(t *testing.T) { ... })`.
* Позволяют группировать связанные тесты и запускать их по фильтру: `go test -run TestFoo/bar`.
* Каждый сабтест имеет собственный `*testing.T`, что даёт независимый статус и лог.
* Сабтесты можно делать параллельными: `t.Parallel()` внутри сабтеста.

```go
func TestSplit(t *testing.T) {
    cases := []struct{ in, sep string; want []string }{
        {"a,b,c", ",", []string{"a", "b", "c"}},
        {"hello world", " ", []string{"hello", "world"}},
    }
    for _, tc := range cases {
        t.Run(tc.in, func(t *testing.T) {
            got := strings.Split(tc.in, tc.sep)
            assert.Equal(t, tc.want, got)
        })
    }
}
```

</details>

---

### Вопрос №4: [ Как запустить тесты параллельно? ]

<details>
<summary>Ответ</summary>

* Вызов `t.Parallel()` в начале теста сообщает, что он может выполняться параллельно с другими параллельными тестами.
* По умолчанию `go test` запускает параллельные тесты с `GOMAXPROCS` горутинами.
* Можно ограничить: `go test -parallel 4`.
* Сабтесты с `t.Parallel()` внутри одного родителя также могут выполняться параллельно, если родитель дожидается их через `t.Run`.

```go
func TestParallel(t *testing.T) {
    t.Parallel()
    // ...
}
```

</details>

---

### Вопрос №5: [ Что такое table-driven тесты? ]

<details>
<summary>Ответ</summary>

* Паттерн, при котором тестовые данные выносятся в таблицу (slice структур), и тест пробегает по ней в цикле.
* Плюсы: лёгкое добавление новых кейсов, единая логика проверки, читаемость.
* Идиоматичный способ тестирования в Go.

```go
func TestAdd(t *testing.T) {
    cases := []struct {
        name     string
        a, b, want int
    }{
        {"positive", 2, 3, 5},
        {"negative", -1, 1, 0},
        {"zero", 0, 0, 0},
    }
    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            got := Add(tc.a, tc.b)
            if got != tc.want {
                t.Errorf("Add(%d, %d) = %d, want %d", tc.a, tc.b, got, tc.want)
            }
        })
    }
}
```

</details>

---

### Вопрос №6: [ Что такое t.Cleanup и чем он лучше defer? ]

<details>
<summary>Ответ</summary>

* `t.Cleanup(func())` регистрирует функцию, которая вызовется после завершения теста (в порядке LIFO).
* В отличие от `defer`, `t.Cleanup` работает корректно с сабтестами и параллельными тестами:
  cleanup родителя выполняется только после завершения всех сабтестов.
* Полезно для освобождения ресурсов, созданных в хелперах.

</details>

---

### Вопрос №7: [ Что такое test helpers и соглашение об именовании? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

* Хелперы помечаются вызовом `t.Helper()` в начале функции.
* Это сообщает testing-фреймворку, что при падении (`t.Error`, `t.Fatal`) строка ошибки указывает на
  **вызывающий** код, а не на строку внутри хелпера.
* Соглашение: имена хелперов часто начинаются с заглавной буквы, если экспортируемые, или со строчной, если внутренние.

```go
func setupDB(t *testing.T) *sql.DB {
    t.Helper()
    db, err := sql.Open("pgx", testDSN)
    if err != nil {
        t.Fatal(err)
    }
    t.Cleanup(func() { db.Close() })
    return db
}
```

</details>

---

### Вопрос №8: [ Как тестировать с флагами и переменными окружения? ]

<details>
<summary>Ответ</summary>

* Флаги: можно использовать `flag.Int("workers", 4, "")` в `_test.go` и передавать `go test -workers=8`.
* Переменные окружения: `t.Setenv(key, value)` — устанавливает env для теста и автоматически восстанавливает после.
* `t.Setenv` автоматически делает тест непараллельным (вызовет panic если `t.Parallel()` уже вызван).

</details>

---

### Вопрос №9: [ Что такое testing.TB? ]

<details>
<summary>Ответ</summary>

* `testing.TB` — интерфейс, объединяющий `testing.T`, `testing.B` и `testing.F` (fuzz test).
* Полезен для написания хелперов, которые работают и в обычных тестах, и в бенчмарках, и в fuzz-тестах.
* Содержит общие методы: `Error`, `Fatal`, `Skip`, `Helper`, `Cleanup`, `TempDir` и т.д.

</details>

---

### Вопрос №10: [ Что такое fuzzing в Go? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* Fuzzing (фаззинг) — автоматическая генерация случайных входных данных для поиска паник и unexpected behavior.
* Доступен с Go 1.18 (тип `testing.F` в пакете `testing`).
* Функция `FuzzXxx(f *testing.F)` принимает seed-корпусы через `f.Add(...)`, затем движок генерирует новые данные.

```go
func FuzzParseURL(f *testing.F) {
    f.Add("http://example.com")
    f.Add("https://google.com?q=1")

    f.Fuzz(func(t *testing.T, raw string) {
        u, err := ParseURL(raw)
        if err != nil {
            return // ignore invalid
        }
        // проверка: ParseURL(u.String()) не должно падать
        if _, err2 := ParseURL(u.String()); err2 != nil {
            t.Errorf("round-trip failed: %v", err2)
        }
    })
}
```

* Запуск: `go test -fuzz=FuzzParseURL -fuzztime=30s`.
* Seed-корпусы сохраняются в `testdata/fuzz/` и воспроизводятся при обычном `go test`.

</details>

---

### Вопрос №11: [ Как детерминированно тестировать конкурентный код без time.Sleep? ]

<details>
<summary>Ответ</summary>

* Ожидайте **событие**, а не предполагаемую длительность работы: канал `ready` для готовности,
  `ack` после обработки задания, `done` для завершения. Отправка задания сама по себе
  не подтверждает, что обработка закончилась; подтверждение отправляют после нужного действия.
* `time.Sleep` не создаёт отношения happens-before и не гарантирует, что worker успел выполниться.
  Для наблюдения результата нужны каналы, `WaitGroup` или другая подходящая синхронизация.
* **`testing/synctest.Test` (стабильный API с Go 1.25)** запускает функцию и созданные ею
  горутины в изолированной bubble. Внутри `time` использует fake clock: таймеры и deadlines
  можно проверять без ожидания соответствующего реального времени.
* Fake time продвигается к следующему таймеру, когда все горутины bubble **durably blocked**
  и нет ожидающего `synctest.Wait`. Примеры: ожидание на канале, созданном внутри bubble,
  `time.Sleep`, `sync.Cond.Wait`. После выхода корневой горутины время больше не продвигается.
* Не всякая блокировка подходит: ожидание `Mutex`/`RWMutex`, сетевого I/O или syscall
  не считается durably blocked. Внешние процессы и реальные соединения не становятся fake;
  для таких зависимостей нужны тестовые двойники либо отдельные интеграционные тесты.
* `synctest.Wait()` вызывают внутри bubble: он ждёт, пока остальные горутины завершатся
  или станут durably blocked. Это точка синхронизации, **не команда завершить горутины**
  и не способ перемотать время. Fake `Sleep` сам по себе тоже не синхронизирует доступ к памяти.
* `synctest.Test` ждёт выхода всех горутин bubble; deadlock приводит к провалу теста.
  `synctest` не перебирает все варианты планирования и не заменяет `go test -race`.

Полный пример для `_test.go` (Go 1.25+): проверяем отсутствие результата до отмены
и получение именно `context.Canceled` после неё. Результат передаётся через канал.

```go
package tooling_test

import (
    "context"
    "testing"
    "testing/synctest"
)

func TestCancellationSynctest(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        ctx, cancel := context.WithCancel(t.Context())
        defer cancel()
        done := make(chan error, 1)
        go func() {
            <-ctx.Done()
            done <- ctx.Err()
        }()

        synctest.Wait()
        select {
        case err := <-done:
            t.Fatalf("worker finished before cancellation: %v", err)
        default:
        }

        cancel()
        synctest.Wait()
        select {
        case err := <-done:
            if err != context.Canceled {
                t.Fatalf("worker error = %v, want %v", err, context.Canceled)
            }
        default:
            t.Fatal("worker did not report cancellation")
        }
    })
}
```

Документация: [testing/synctest](https://pkg.go.dev/testing/synctest).

</details>

---

### Вопрос №12: [ Как связать жизненный цикл worker с T.Context? ]

<details>
<summary>Ответ</summary>

* **`t.Context()` доступен с Go 1.24**. Его контекст отменяется непосредственно перед
  выполнением функций `t.Cleanup`, поэтому cleanup может дождаться остановки worker (join).
* Нельзя синхронно ждать `<-t.Context().Done()` в теле теста до его возврата:
  тест не дойдёт до фазы, в которой этот контекст отменяется. Ожидание переносите в cleanup.
* Сама отмена не означает завершение worker: он должен обработать `ctx.Done()` и сообщить
  о выходе через `done`. Регистрируйте cleanup до запуска worker, чтобы join состоялся и при `t.Fatal`.
* Для **явной проверки cancellation в теле теста** создайте дочерний контекст через
  `context.WithCancel(t.Context())`, вызовите `cancel()` и проверьте ответ worker, как выше.
  Отмена дочернего контекста не отменяет контекст самого теста; `defer cancel()` освобождает ресурсы.

Полный lifecycle-тест для `_test.go` (Go 1.24+): worker ждёт отмены и завершается,
а cleanup подтверждает его выход. Закрытие `done` синхронизирует чтение `workerErr`.

```go
package tooling_test

import (
    "context"
    "testing"
)

func TestWorkerLifecycle(t *testing.T) {
    ctx := t.Context()
    ready := make(chan struct{})
    done := make(chan struct{})
    var workerErr error
    t.Cleanup(func() {
        <-done
        if workerErr != context.Canceled {
            t.Errorf("worker error = %v, want %v", workerErr, context.Canceled)
        }
    })
    go func() {
        defer close(done)
        close(ready)
        <-ctx.Done()
        workerErr = ctx.Err()
    }()
    <-ready
    if err := ctx.Err(); err != nil {
        t.Fatalf("test context canceled before cleanup: %v", err)
    }
}
```

Если worker использует общий ресурс, учитывайте LIFO: cleanup с join должен выполниться
до закрытия этого ресурса. Документация: [T.Context](https://pkg.go.dev/testing#T.Context).

</details>

---

</details>

---

## 2. Бенчмарки и покрытие

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Как написать бенчмарк в Go? ]

<details>
<summary>Ответ</summary>

```go
func BenchmarkFoo(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Foo()
    }
}
```

* Имя начинается с `Benchmark`, параметр `*testing.B`.
* `b.N` — количество итераций, которое testing-фреймворк подбирает автоматически так, чтобы бенчмарк шёл ~1 секунду.
* Запуск: `go test -bench=.`.

</details>

---

### Вопрос №2: [ Зачем нужен b.ResetTimer и b.ReportAllocs? ]

<details>
<summary>Ответ</summary>

* `b.ResetTimer()` — сбрасывает таймер после подготовительной фазы (инициализация данных, не являющаяся частью измеряемого кода).
* `b.ReportAllocs()` — включает отчёт об аллокациях (кол-во аллокаций и байт на итерацию).
* `b.ReportMetric(float64, "unit")` — добавить кастомную метрику.

```go
func BenchmarkSort(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        b.StopTimer()
        data := generateData(1000) // подготовка — не измеряем
        b.StartTimer()
        sort.Ints(data) // измеряем только сортировку
    }
}
```

</details>

---

### Вопрос №3: [ Как измерить покрытие кода тестами? ]

<details>
<summary>Ответ</summary>

```bash
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out         # HTML-отчёт
go tool cover -func=coverage.out         # текстовый отчёт по функциям
```

* `-coverprofile` записывает, какие строки были выполнены во время тестов.
* `go tool cover -html` открывает интерактивный отчёт в браузере.
* `-covermode=atomic` — режим подсчёта покрытия (atomic нужен при использовании `-race`).
  Целевое покрытие задаётся через `-coverprofile` + анализ, либо `go test -cover` показывает процент.

</details>

---

### Вопрос №4: [ Чем отличаются covermode: set, count, atomic? ]

<details>
<summary>Ответ</summary>

* `set` (по умолчанию) — только был ли выполнен блок (да/нет).
* `count` — сколько раз каждый блок был выполнен.
* `atomic` — то же, что count, но с атомарными счётчиками (нужно для параллельных тестов).
* Для race-тестов: `-covermode=atomic`.

</details>

---

### Вопрос №5: [ Что такое b.RunParallel и когда его использовать? ]

<details>
<summary>Ответ</summary>

* `b.RunParallel(func(pb *testing.PB) { for pb.Next() { Foo() } })` — запускает бенчмарк параллельно на нескольких горутинах.
* Полезно для измерения throughput при конкурентном доступе (например, sync.Map vs map+Mutex).
* Количество горутин = `runtime.GOMAXPROCS(0)` (можно переопределить через `b.SetParallelism(n)`).

</details>

---

### Вопрос №6: [ Как сравнить результаты бенчмарков? ]

<details>
<summary>Ответ</summary>

```bash
go test -bench=. -benchmem -count=10 > old.txt
# ... внесение изменений ...
go test -bench=. -benchmem -count=10 > new.txt
benchstat old.txt new.txt
```

* `benchstat` — утилита для статистического сравнения результатов бенчмарков.
* `-count=10` запускает бенчмарк 10 раз для уменьшения шума.
* `-benchmem` включает статистику по аллокациям.

</details>

---

### Вопрос №7: [ Чем B.Loop удобнее цикла по b.N? ]

<details>
<summary>Ответ</summary>

* **`b.Loop()` доступен с Go 1.24** и сам определяет продолжительность измерения.
  Функция бенчмарка вызывается один раз на измерение, а не повторно при подборе `b.N`.
* Первый вызов `Loop` сбрасывает таймер и счётчики аллокаций: setup до цикла не измеряется.
  Возвращая `false`, `Loop` останавливает таймер, поэтому cleanup после цикла тоже исключён.
* Подготовка **внутри** цикла автоматически не исключается: для неё используйте пару
  `b.StopTimer()` / `b.StartTimer()`. `b.ResetTimer()` обнуляет накопленные измерения,
  но не меняет состояние таймера; вызывать его на каждой итерации неправильно.
* Пишите именно **`for b.Loop() { ... }`**. Компилятор защищает вызовы в теле такой формы
  от полного удаления из-за неиспользуемого результата; не прячьте условие в helper.
* Не смешивайте `Loop` с циклом по `b.N` или с `b.RunParallel` в одном бенчмарке.
  Для параллельного измерения используйте отдельный бенчмарк с `RunParallel` / `pb.Next()`.
  После выхода из `Loop` значение `b.N` можно читать для расчёта дополнительных метрик.

Полный пример для `_test.go` (Go 1.24+): измеряем SHA-256, а не подготовку входных данных.

```go
package tooling_test

import (
    "crypto/sha256"
    "testing"
)

func BenchmarkSHA256Loop(b *testing.B) {
    data := []byte("payload for SHA-256 benchmark")
    b.ReportAllocs()
    b.SetBytes(int64(len(data)))
    for b.Loop() {
        sha256.Sum256(data)
    }
}
```

Ограниченный запуск: `go test -run='^$' -bench='^BenchmarkSHA256Loop$' -benchtime=100x -count=1 -benchmem`.
Документация: [B.Loop](https://pkg.go.dev/testing#B.Loop).

</details>

---

</details>

---

## 3. Моки и тестовые двойники

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Какие бывают типы test doubles? ]

<details>
<summary>Ответ</summary>

* **Dummy** — передаётся, но не используется (заполнение аргументов).
* **Stub** — возвращает предопределённые ответы на вызовы.
* **Spy** — записывает информацию о вызовах (аргументы, количество).
* **Mock** — stub + проверки ожиданий (был ли вызван, с какими аргументами, сколько раз).
* **Fake** — упрощённая, но рабочая реализация (in-memory БД вместо реальной).

</details>

---

### Вопрос №2: [ Как тестировать без мок-фреймворков в Go? ]

<details>
<summary>Ответ</summary>

* В Go принято использовать **интерфейсы** для подмены зависимостей.
* Создаётся интерфейс с нужными методами, в продакшене используется реальная реализация, в тестах — простая структура-заглушка.
* Это часто предпочтительнее мок-фреймворков (mockery, gomock, testify/mock), т.к. проще и не требует кодогенерации.

```go
// Интерфейс
type UserRepository interface {
    GetByID(id int) (*User, error)
}

// Реальная реализация
type pgUserRepo struct { db *sql.DB }
func (r *pgUserRepo) GetByID(id int) (*User, error) { /* SQL */ }

// Тестовая заглушка
type fakeUserRepo struct { user *User; err error }
func (f *fakeUserRepo) GetByID(id int) (*User, error) {
    return f.user, f.err
}
```

</details>

---

### Вопрос №3: [ Что такое testify? ]

<details>
<summary>Ответ</summary>

* `github.com/stretchr/testify` — популярная библиотека для тестирования в Go.
* Подпакеты:
  - `assert` — assertions без остановки теста (`assert.Equal(t, want, got)`).
  - `require` — assertions с остановкой теста (`require.NoError(t, err)`).
  - `mock` — мок-фреймворк.
  - `suite` — тестовые наборы с setup/teardown.
* `assert` использует `t.Helper()`, поэтому ошибки указывают на строку вызова.

</details>

---

### Вопрос №4: [ Как использовать gomock / mockery? ] ![Static Badge](https://img.shields.io/badge/Middle-yellow)

<details>
<summary>Ответ</summary>

* **gomock** (от Google) — кодогенерация моков по интерфейсам.

```bash
go install go.uber.org/mock/mockgen@latest
mockgen -source=user_repo.go -destination=mock_user_repo.go -package=mocks
```

```go
func TestUserService(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockRepo := mocks.NewMockUserRepository(ctrl)
    mockRepo.EXPECT().GetByID(1).Return(&User{Name: "Alice"}, nil)

    svc := NewUserService(mockRepo)
    user, err := svc.GetUser(1)
    require.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
}
```

* **mockery** — альтернатива, генерирует моки по именам интерфейсов или через конфиг `.mockery.yaml`.
* `ctrl.Finish()` проверяет, что все ожидания были выполнены (в новых версиях gomock можно не вызывать — делается автоматически через `t.Cleanup`).

</details>

---

### Вопрос №5: [ Что такое httptest и как тестировать HTTP-хендлеры? ]

<details>
<summary>Ответ</summary>

* `net/http/httptest` — пакет для тестирования HTTP-кода без запуска реального сервера.

```go
func TestHandler(t *testing.T) {
    req := httptest.NewRequest("GET", "/users/1", nil)
    w := httptest.NewRecorder()

    handler(w, req)

    resp := w.Result()
    require.Equal(t, http.StatusOK, resp.StatusCode)
    // body := w.Body.String()
}
```

* `httptest.NewServer(handler)` — запускает реальный HTTP-сервер на случайном порту, удобно для интеграционных тестов.
* `httptest.NewTLSServer` — для HTTPS.

</details>

---

### Вопрос №6: [ Как тестировать с реальной базой данных (testcontainers)? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **testcontainers-go** — библиотека, поднимающая Docker-контейнеры в тестах (PostgreSQL, Redis, Kafka и т.д.).
* Альтернативы: embedded PostgreSQL, in-memory SQLite (с оговорками по совместимости SQL).

```go
func TestRepository(t *testing.T) {
    ctx := context.Background()
    pgContainer, err := postgres.Run(ctx,
        "postgres:16-alpine",
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).WithStartupTimeout(30*time.Second)),
    )
    require.NoError(t, err)
    defer pgContainer.Terminate(ctx)

    dsn, _ := pgContainer.ConnectionString(ctx)
    db, _ := sql.Open("pgx", dsn)
    // запуск миграций, тесты...
}
```

* Плюсы: реальная СУБД, полный SQL-диалект.
* Минусы: медленнее unit-тестов, требует Docker в CI.

</details>

---

</details>

---

## 4. Go Modules и зависимости

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Что такое go.mod и go.sum? ]

<details>
<summary>Ответ</summary>

* `go.mod` — файл модуля: объявляет module path, версию Go и зависимости.
* `go.sum` — криптографические хэши (SHA-256) всех зависимостей и их транзитивных зависимостей для верификации целостности.
* `go.sum` должен быть в VCS, чтобы все разработчики и CI получали одинаковые версии.

```
module github.com/example/myapp

go 1.23

require (
    github.com/lib/pq v1.10.9
    github.com/stretchr/testify v1.9.0
)
```

</details>

---

### Вопрос №2: [ Что такое MVS (Minimal Version Selection)? ]

<details>
<summary>Ответ</summary>

* MVS — алгоритм разрешения зависимостей в Go Modules.
* Выбирает **минимальную** версию, удовлетворяющую всем требованиям (а не максимальную, как в других системах).
* Если зависимость A требует lib v1.2, а B требует lib v1.3, Go выберет v1.3 (минимальная из удовлетворяющих).
* Плюсы: детерминизм, воспроизводимость, отсутствие "dependency hell" resolver'а.
* Минусы: не обновляет автоматически до более новых версий (нужно `go get`).

</details>

---

### Вопрос №3: [ Чем отличаются команды go get и go install? ]

<details>
<summary>Ответ</summary>

* `go get` — добавляет/обновляет зависимость в `go.mod` (добавляет в `require`).
* `go install` — компилирует и устанавливает бинарник в `$GOBIN` (или `$GOPATH/bin`).
* Начиная с Go 1.16, `go install example.com/tool@v1.2.3` устанавливает бинарник независимо
  от текущего модуля и не изменяет `go.mod`.
* Начиная с Go 1.17, `go get` больше не собирает и не устанавливает пакеты — используется
  только для управления зависимостями в `go.mod`.

</details>

---

### Вопрос №4: [ Что такое replace, retract и exclude в go.mod? ]

<details>
<summary>Ответ</summary>

* `replace` — подменяет модуль на локальный путь или другую версию. Удобно для локальной разработки.
  ```
  replace github.com/example/lib => ../lib
  replace github.com/example/lib v1.2.3 => github.com/example/lib v1.3.0
  ```
* `exclude` — исключает конкретную версию зависимости (например, с критическим багом).
  ```
  exclude github.com/example/lib v1.2.4
  ```
* `retract` (Go 1.16+) — помечает собственную версию модуля как отозванную. `go get` выдаст предупреждение.
  ```
  retract v1.2.3 // serious bug, see issue #42
  ```

</details>

---

### Вопрос №5: [ Что такое vendor директория и когда её использовать? ]

<details>
<summary>Ответ</summary>

* `go mod vendor` — копирует все зависимости в `vendor/` директорию.
* При наличии `vendor/`, `go build` использует её вместо загрузки модулей из кэша/сети.
* `-mod=vendor` (включается автоматически при наличии `vendor/`, если в `go.mod` указана версия 1.14+).
* Полезно для: CI без интернета, air-gap окружения, гарантия воспроизводимости.
* Минусы: большой объём в VCS, нужно обновлять вручную через `go mod vendor`.

</details>

---

### Вопрос №6: [ Что такое Go toolchain (Go 1.21+)? ]

<details>
<summary>Ответ</summary>

* Начиная с Go 1.21, `go.mod` может содержать директиву `toolchain go1.23.0`.
* Если установленный Go старее, чем требует `toolchain`, Go автоматически скачает и использует нужную версию.
* `GOTOOLCHAIN` env var управляет поведением: `auto` (по умолчанию), `local`, или конкретная версия.
* Это решает проблему "разные разработчики с разными версиями Go".

</details>

---

### Вопрос №7: [ Как работает module proxy и GOPROXY? ]

<details>
<summary>Ответ</summary>

* `GOPROXY` — URL прокси-сервера для загрузки модулей (по умолчанию `https://proxy.golang.org,direct`).
* Прокси кэширует модули и обеспечивает доступность даже если оригинальный репозиторий удалён.
* `GOFLAGS=-mod=mod` — разрешает автоматическое обновление `go.mod`.
* `GOSUMDB=sum.golang.org` — база данных контрольных сумм для проверки целостности.
* Для приватных репозиториев: `GOPRIVATE=git.company.com` (обходит прокси и sumdb).

</details>

---

</details>

---

## 5. Профилирование и отладка

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Как сделать CPU-профиль в Go? ]

<details>
<summary>Ответ</summary>

Способ 1 — через тесты:
```bash
go test -cpuprofile=cpu.out -bench=.
go tool pprof cpu.out
```

Способ 2 — через `pprof` в коде:
```go
import _ "net/http/pprof"

go func() {
    http.ListenAndServe("localhost:6060", nil)
}()
```
```bash
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

* `go tool pprof` открывает интерактивный режим: `top`, `list funcName`, `web` (граф в браузере).
* `-http=:8080` запускает web-интерфейс pprof.

</details>

---

### Вопрос №2: [ Как получить профиль памяти (heap)? ]

<details>
<summary>Ответ</summary>

```bash
# Через pprof endpoint
go tool pprof http://localhost:6060/debug/pprof/heap

# Через тесты
go test -memprofile=mem.out -bench=.
go tool pprof mem.out
```

* Heap-профиль показывает, где аллокируется память, которая **не была освобождена** на момент снятия профиля.
* `alloc_space` — все аллокации за время работы, `inuse_space` — текущее использование.

```bash
(pprof) top
(pprof) list MyFunc
```

</details>

---

### Вопрос №3: [ Что такое go trace и trace viewer? ]

<details>
<summary>Ответ</summary>

* `runtime/trace` — инструмент для анализа временной шкалы работы Go-программы.
* Показывает: планирование горутин, GC events, syscall blocking, network blocking.

```bash
go test -trace=trace.out -bench=.
go tool trace trace.out
```

* Открывает web-интерфейс с timeline, где видно:
  - Когда горутины создавались, блокировались, заканчивались.
  - Причины блокировок (GC, channel, syscall, network).
  - Проблемы планировщика (например, горутина не вытесняется).

</details>

---

### Вопрос №4: [ Как найти goroutine leak? ]

<details>
<summary>Ответ</summary>

* Способ 1: `go tool pprof http://localhost:6060/debug/pprof/goroutine` — показывает все стеки горутин.
* Способ 2: `runtime.NumGoroutine()` в тестах — проверить, что количество горутин не растёт.
* Способ 3: `go test -run TestXxx -race` — race detector может выявить некоторые проблемы.
* Способ 4: в тесте использовать `goleak` пакет от Uber:

```go
import "go.uber.org/goleak"

func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```

</details>

---

### Вопрос №5: [ Что такое -race и как работает race detector? ]

<details>
<summary>Ответ</summary>

* `go test -race` / `go build -race` — включает детектор гонок данных.
* Работает на основе алгоритма ThreadSanitizer (TSan): инструментирует обращения к памяти и отслеживает happens-before.
* Добавляет ~2x к времени выполнения и ~10x к потреблению памяти.
* Находит: одновременные чтение+запись или запись+запись без синхронизации.
* **Не находит**: гонки, которые не произошли во время конкретного запуска (он динамический, не статический).

</details>

---

### Вопрос №6: [ Что такое escape analysis и как его проверить? ]

<details>
<summary>Ответ</summary>

* Escape analysis определяет, может ли переменная быть выделена на стеке или "утекает" (escape) в heap.
* Если переменная не покидает функцию (адрес не возвращается, не сохраняется) — она на стеке (быстро, бесплатно для GC).
* Проверка:

```bash
go build -gcflags="-m" ./...
go build -gcflags="-m -m" ./...  # более подробно
```

* Примеры "утечки": возврат указателя на локальную переменную, запись в interface{}, сохранение в замыкании.
* `//go:noescape` — существующая давно директива компилятора (не начиная с какой-то версии):
  указывает, что аргументы asm-функции не «утекают» в кучу, применяется только к функциям на ассемблере.

</details>

---

### Вопрос №7: [ Как использовать delve (dlv) для отладки? ]

<details>
<summary>Ответ</summary>

* Delve — отладчик для Go, лучше работает с горутинами, чем gdb.
* Установка: `go install github.com/go-delve/delve/cmd/dlv@latest`.

```bash
dlv debug ./cmd/app           # отладка main пакета
dlv test ./pkg/service        # отладка тестов
dlv attach <pid>              # подключение к работающему процессу
```

* Команды: `break` (b), `continue` (c), `step` (s), `next` (n), `print` (p), `goroutines`, `threads`, `locals`.

</details>

---

### Вопрос №8: [ Как проводить нагрузочное тестирование? (vegeta, k6, wrk) ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Цель**: найти предел пропускной способности, деградацию латентности, утечки (см. вопрос про
  утечки памяти в go-questions) до продакшена.
* **Инструменты**:
  - **vegeta** (Go) — атака с постоянной частотой (constant rate): `vegeta attack -rate=1000 -duration=60s | vegeta report`.
    Показывает, что система не справляется уже при заданной нагрузке (а не «упирается и молчит»);
  - **k6** — сценарии на JS: этапы (ramp-up, пик, спад), пороги (thresholds: `p99 < 300ms`),
    интеграция в CI, экспорт метрик в Prometheus;
  - **wrk / wrk2** — компактные бенчмарки для быстрых прогонов (корректная координация задержек в wrk2).
* **Виды нагрузки**:
  - **нагрузочное** — ожидаемый продакшен-трафик;
  - **стресс** — поиск предела и поведения за ним (деградация, очереди, ошибки);
  - **пиковое (spike)** — резкий рост в разы;
  - **длительное (soak)** — часы под нагрузкой, выявляет утечки и фрагментацию.
* **Что измерять**: RPS, латентность **процентилями** (p50/p95/p99, не средним), долю ошибок,
  потребление CPU/RAM/GC-паузы цели.
* **Профилирование под нагрузкой**: включить `net/http/pprof` на отдельном порту и снять
  CPU/heap-профили во время прогона.
* **Антипаттерн**: гнать нагрузку на один инстанс через тот же узел, где стоит генератор —
  упрётся в клиент, а не в сервер.

</details>

---

</details>

---

## 6. Антипаттерны и лучшие практики

<details>
<summary><b>Развернуть раздел</b></summary>

---

### Вопрос №1: [ Назовите антипаттерны тестирования в Go ]

<details>
<summary>Ответ</summary>

* **Тестирование реализаций, а не поведения** — тестирование приватных функций вместо публичного API.
* **Мокирование всего** — вместо моков лучше использовать интерфейсы и fakes.
* **Отсутствие table-driven тестов** — дублирование логики в разных тестах.
* **Игнорирование ошибок в тестах** — `err != nil` проверка опущена "потому что это тест".
* **Тесты зависят от порядка выполнения** — использование глобального состояния.
* **Тесты зависят от внешних ресурсов** — без cleanup, без изоляции (общая БД, общие файлы).
* **Слишком широкие assert'ы** — `assert.NotNil(t, result)` вместо проверки конкретных полей.
* **Sleep в тестах** — `time.Sleep(100 * time.Millisecond)` вместо каналов или `Eventually`.
* **Тесты без t.Helper()** — ошибки указывают на строку в хелпере, а не в вызывающем коде.

</details>

---

### Вопрос №2: [ Что такое golden files и когда их использовать? ]

<details>
<summary>Ответ</summary>

* Golden files — предопределённые файлы с ожидаемым результатом, с которыми сравнивается вывод теста.
* Удобно для: сериализации (JSON, XML), рендеринга шаблонов, генерации кода.

```go
func TestRender(t *testing.T) {
    got := Render(data)
    golden := filepath.Join("testdata", "expected.golden")
    if *update {
        os.WriteFile(golden, got, 0644)
        return
    }
    want, _ := os.ReadFile(golden)
    assert.Equal(t, want, got)
}
```

* Флаг `-update` перегенерирует golden files: `go test -run TestRender -update`.

</details>

---

### Вопрос №3: [ Как структурировать тесты: white-box vs black-box? ]

<details>
<summary>Ответ</summary>

* **White-box** (internal test): `package myservice` — тест в том же пакете, доступ к приватным функциям.
* **Black-box** (external test): `package myservice_test` — тест в отдельном пакете, только публичный API.
* Best practice: **по умолчанию black-box**, white-box только когда нужно протестировать внутреннюю логику, недоступную снаружи.
* Внешние тесты не создают циклических импортов при использовании тестовых утилит из других пакетов.

</details>

---

### Вопрос №4: [ Что такое build tags и как использовать для тестов? ]

<details>
<summary>Ответ</summary>

* Build tags (build constraints) — комментарии в начале файла, управляющие компиляцией.

```go
//go:build integration
// +build integration

package myservice_test

func TestDatabaseIntegration(t *testing.T) { /* ... */ }
```

* Запуск: `go test -tags=integration ./...`.
* Обычный `go test` пропустит эти файлы.
* Альтернатива: `testing.Short()` — `if testing.Short() { t.Skip() }`, запуск: `go test -short`.

</details>

---

### Вопрос №5: [ Что такое t.TempDir()? ]

<details>
<summary>Ответ</summary>

* `t.TempDir()` — создаёт временную директорию и автоматически удаляет её после теста.
* Не нужно писать `os.MkdirTemp` + `defer os.RemoveAll`.
* Каждый тест получает уникальную директорию, что исключает гонки между тестами.

```go
func TestWriteFile(t *testing.T) {
    dir := t.TempDir()
    path := filepath.Join(dir, "output.txt")
    err := WriteFile(path, data)
    require.NoError(t, err)
}
```

</details>

---

### Вопрос №6: [ Как писать тестируемый код? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **Зависимости через интерфейсы** — позволяет подменять реализации в тестах.
* **Чистые функции** — без side effects, результат зависит только от входных данных.
* **Внедрение зависимостей (DI)** — через конструкторы, а не глобальные переменные.
* **Маленькие функции** — проще тестировать, проще понимать.
* **`time.Now()` за интерфейсом** — или `clock.Worker` для контроля времени в тестах.
* **Явные ошибки** — `if err != nil` всегда, без паник в бизнес-логике.
* **Минимум ветвлений** — меньше edge cases для тестирования.
* **Separation of I/O** — отделить чтение/запись от бизнес-логики.

</details>

---

### Вопрос №7: [ Что такое golangci-lint? Какие линтеры включать? ] ![Static Badge](https://img.shields.io/badge/Senior-red)

<details>
<summary>Ответ</summary>

* **golangci-lint** — агрегатор десятков линтеров: запуск параллельно, кэширование, единый
  конфиг `.golangci.yml`. Установка: `go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@latest`.
* **Обязательный минимум** (обычно включён по умолчанию):
  - `govet` — подозрительные конструкции;
  - `errcheck` — необработанные ошибки;
  - `staticcheck` — самый мощный набор проверок (включая SA, S, ST);
  - `unused` — мёртвый код.
* **Для сеньор-проектов дополнительно**:
  - `bodyclose` / `noctx` — незакрытые `resp.Body`, запросы без контекста;
  - `gosec` — security-проверки (слабая криптография, инъекции);
  - `gocritic`, `revive` — стилевые идиомы;
  - `fieldalignment` (через go vet) — паддинг структур;
  - `prealloc`, `copyloopvar` — микрооптимизации и ловушки циклов.
* **Практика**: линтеры в CI обязательным шагом; точечные подавления `//nolint:<linter> // причина`,
  глобальные исключения в конфиге держать минимальными.
* **Важно**: линтер не заменяет ревью — он находит классы ошибок, но не дизайн-проблемы.

</details>

---

</details>

---
