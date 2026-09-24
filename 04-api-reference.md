# 4. Конечные точки API

## 4.0 Общие правила API

### Структура URL

```
https://api.mystore.com/v1/auth/{endpoint}
```

Версия API: `v1`

### Аутентификация

Большинство endpoints используют **Bearer Token** в заголовке `Authorization`:

```
Authorization: Bearer <access_token>
```

Для `/v1/auth/refresh` используется **double-submit cookie pattern**:
- `access_token` передаётся в `Authorization: Bearer <token>`
- `refresh_token` автоматически отправляется как HTTP-only cookie

### Стандартные заголовки

| Заголовок | Обязательный | Описание |
|-----------|--------------|----------|
| `Content-Type` | Для POST/PUT/PATCH | `application/json` |
| `Authorization` | Для защищённых endpoints | `Bearer <access_token>` |
| `Accept` | Опционально | `application/json` |

---

### Общая структура ответа

**Успех (2xx):**
```json
{
  "field1": "value1",
  "field2": "value2"
}
```

**Ошибка (4xx/5xx):**
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Описание ошибки на русском",
    "details": {}
  }
}
```

### Коды ошибок

| Код | Описание | HTTP Status |
|-----|----------|-------------|
| `VALIDATION_ERROR` | Ошибка валидации данных | 400 |
| `EMAIL_ALREADY_REGISTERED` | Email уже зарегистрирован | 409 |
| `RATE_LIMITED` | Превышен лимит запросов | 429 |
| `CONFIRMATION_TOKEN_INVALID` | Неверный или истекший токен | 400 |
| `EMAIL_ALREADY_CONFIRMED` | Email уже подтверждён | 409 |
| `INVALID_CREDENTIALS` | Неверные учётные данные | 401 |
| `EMAIL_NOT_CONFIRMED` | Email не подтверждён | 403 |
| `REFRESH_TOKEN_INVALID` | Неверный токен обновления | 401 |
| `ACCESS_TOKEN_INVALID` | Неверный или истекший токен доступа | 401 |
| `INVALID_TOKEN` | Неверный токен (общий) | 400 |
| `TOKEN_NOT_FOUND` | Токен не найден | 404 |
| `USER_NOT_FOUND` | Пользователь не найден | 404 |
| `WEAK_PASSWORD` | Новый пароль не соответствует требованиям | 400 |
| `EMAIL_ALREADY_VERIFIED` | Email уже подтверждён | 409 |

---

## 4.1 Справочник endpoints

| Метод | Endpoint | Описание | Аутентификация |
|-------|----------|----------|----------------|
| POST | `/v1/auth/register` | Регистрация нового пользователя | Нет |
| POST | `/v1/auth/verify` | Подтверждение email | Нет |
| POST | `/v1/auth/login` | Авторизация пользователя | Нет |
| POST | `/v1/auth/refresh` | Обновление access token | Bearer + Cookie |
| POST | `/v1/auth/logout` | Выход из системы | Bearer |
| GET | `/v1/auth/me` | Получение данных текущего пользователя | Bearer |
| POST | `/v1/auth/resend-verification` | Повторная отправка verification email | Нет |
| POST | `/v1/auth/forgot-password` | Запрос сброса пароля | Нет |
| POST | `/v1/auth/reset-password` | Сброс пароля с токеном | Нет |
| POST | `/v1/auth/unsubscribe` | Отписка от рассылки | Нет |
| PUT | `/v1/auth/change-password` | Изменение пароля | Bearer |
| DELETE | `/v1/auth/me` | Удаление аккаунта | Bearer |

---

## 4.2 POST /v1/auth/register

**Описание:** Регистрация нового пользователя

**Нормализация email:**
- Пробелы по краям строки удаляются
- Строка приводится к нижнему регистру
- Другой нормализации нет: точки (`.`) и часть после символа `+` сохраняются
- Email хранится в нормализованном виде в базе данных

**Пример:**
- Ввод: `"  Ivan.Ivanov+test@Example.COM  "`
- Нормализованный email: `"ivan.ivanov+test@example.com"`

**Описание схемы:**
Схема показывает процесс регистрации пользователя. Клиент отправляет POST-запрос на `/v1/auth/register`. API Gateway проверяет rate limit по комбинации IP и email. При успешной проверке Auth Service нормализует email (удаляет пробелы, приводит к нижнему регистру), валидирует входные данные и хеширует пароль с помощью Argon2id. Затем создается запись пользователя в PostgreSQL и генерируется одноразовый токен для подтверждения email. После этого отправляется письмо с ссылкой для верификации через SMTP-сервер. Клиент получает ответ 201 Created. При превышении rate limit возвращается 429 Too Many Requests.

**Схема:**
```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL
    participant SMTP as SMTP

    C->>G: POST /v1/auth/register
    G->>R: check rate limit (IP:email)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>A: normalize email & validate input
        A->>A: hash password (Argon2id)
        A->>PG: INSERT INTO users
        A->>PG: INSERT INTO email_verification_tokens
        PG-->>A: user_id
        A->>SMTP: send verification email
        SMTP-->>A: queued
        G-->>C: 201 Created
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

**Request Body:**

```json
{
  "email": "user@example.com",
  "password": "SecurePass123",
  "name": "Иван Иванов"
}
```

**Validation Rules:**

| Поле | Обязательное | Валидация |
|------|-------------|-----------|
| `email` | да | valid email format |
| `password` | да | min 8 characters |
| `name` | да | min 1 character |

**Success Response (201):**

```json
{
  "message": "Registration successful. Please check your email to verify your account.",
  "email": "user@example.com"
}
```

**Error Responses:**

| Код | Описание | Пример |
|-----|----------|--------|
| `400 VALIDATION_ERROR` | Ошибка валидации данных | `{"error": {"code": "VALIDATION_ERROR", "message": "..."}}` |
| `409 EMAIL_ALREADY_REGISTERED` | Email уже зарегистрирован | `{"error": {"code": "EMAIL_ALREADY_REGISTERED", "message": "..."}}` |
| `429 RATE_LIMITED` | Превышен лимит запросов | `{"error": {"code": "RATE_LIMITED", "message": "..."}}` |

**Response 400 Bad Request (VALIDATION_ERROR):**

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Ошибка валидации данных",
    "fields": [
      { "field": "email", "code": "INVALID_FORMAT" },
      { "field": "password", "code": "TOO_SHORT" },
      { "field": "name", "code": "REQUIRED" }
    ]
  }
}
```

**Response 409 Conflict (EMAIL_ALREADY_REGISTERED):**

```json
{
  "error": {
    "code": "EMAIL_ALREADY_REGISTERED",
    "message": "Пользователь с этим email уже существует",
    "details": {}
  }
}
```

**Response 429 Too Many Requests (RATE_LIMITED):**

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Превышен лимит запросов",
    "details": { "retry_after": 3600 }
  }
}
```

**Пример curl:**

```bash
curl -X POST https://api.mystore.com/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "SecurePass123",
    "name": "Иван Иванов"
  }'
```

---

## 4.3 POST /v1/auth/verify

**Описание:** Подтверждение email с одноразовым токеном

**Описание схемы:**
Схема показывает процесс подтверждения email с одноразовым токеном. Клиент отправляет POST-запрос на `/v1/auth/verify` с токеном. API Gateway проверяет rate limit по IP. Auth Service ищет токен в PostgreSQL. Если токен найден и не истек, обновляется флаг `is_email_verified`, удаляется токен, и генерируются новые access token и refresh token (для непрерывности сессии). При истёкшем или не найденном токене возвращается 400 CONFIRMATION_TOKEN_INVALID. При превышении rate limit — 429 Too Many Requests.

**Схема:**
```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL

    C->>G: POST /v1/auth/verify
    G->>R: check rate limit (IP)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>PG: SELECT email_verification_tokens WHERE token = ?
        PG-->>A: token record / null
        alt Token found & valid
            A->>PG: UPDATE users SET is_email_verified = TRUE
            A->>PG: DELETE email_verification_tokens
            A->>PG: INSERT INTO refresh_tokens
            G-->>C: 200 OK (access_token + refresh_token)
        else Token expired/not found
            G-->>C: 400 CONFIRMATION_TOKEN_INVALID
        end
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

**Request Body:**

```json
{
  "token": "abc123..."
}
```

**Success Response (200):**

```json
{
  "access_token": "eyJhbG...",
  "refresh_token": "eyJhbG..."
}
```

**Error Responses:**

| Код | Описание | Пример |
|-----|----------|--------|
| `400 CONFIRMATION_TOKEN_INVALID` | Неверный или истекший токен | `{"error": {"code": "CONFIRMATION_TOKEN_INVALID", "message": "..."}}` |
| `409 EMAIL_ALREADY_CONFIRMED` | Email уже подтверждён | `{"error": {"code": "EMAIL_ALREADY_CONFIRMED", "message": "..."}}` |

**Response 400 Bad Request (CONFIRMATION_TOKEN_INVALID):**

```json
{
  "error": {
    "code": "CONFIRMATION_TOKEN_INVALID",
    "message": "Неверный или истёкший токен подтверждения",
    "details": {}
  }
}
```

**Response 409 Conflict (EMAIL_ALREADY_CONFIRMED):**

```json
{
  "error": {
    "code": "EMAIL_ALREADY_CONFIRMED",
    "message": "Email уже подтверждён",
    "details": {}
  }
}
```

**Пояснение: Почему возвращаются два токена?**

После успешной верификации email возвращаются **оба токена** (`access_token` и `refresh_token`) по следующим причинам:

1. **Непрерывность сессии** — пользователь продолжает работать без повторного логина
2. **Единообразие flow'ев** — верификация — это завершение регистрации, а не отдельный endpoint
3. **Безопасность refresh token** — `refresh_token` передаётся как HTTP-only cookie
4. **Token rotation** — при каждом `/v1/auth/verify` генерируется **новый refresh token**
5. **Off-session verify** — пользователь может подтвердить email в другом браузере

**Пример curl:**

```bash
curl -X POST https://api.mystore.com/v1/auth/verify \
  -H "Content-Type: application/json" \
  -d '{
    "token": "abc123xyz"
  }'
```

---

## 4.4 POST /v1/auth/login

**Описание:** Авторизация пользователя

**Описание схемы:**
Схема показывает процесс авторизации пользователя. Клиент отправляет POST-запрос на `/v1/auth/login`. API Gateway проверяет rate limit по IP и email. При успешной проверке Auth Service ищет пользователя в PostgreSQL по email. Если пользователь найден, происходит верификация пароля с помощью Argon2id (timing-safe сравнение). При успешной верификации создается сессия в Redis, генерируются access token (30 минут) и refresh token (7 дней), который сохраняется в PostgreSQL и устанавливается как HTTP-only cookie. При неверных данных или несуществующем пользователе возвращается 401 Unauthorized (с timing-safe dummy check чтобы не раскрывать наличие пользователя). При превышении rate limit — 429 Too Many Requests.

**Схема:**
```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL

    C->>G: POST /v1/auth/login
    G->>R: check rate limit (IP:email)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>PG: SELECT * FROM users WHERE email = ?
        PG-->>A: user record / null
        alt User found
            A->>A: verify password (Argon2id)
            A->>R: store session
            A->>A: generate access token (30min)
            A->>A: generate refresh token (7 days)
            A->>PG: INSERT INTO refresh_tokens
            G-->>C: 200 OK
            G->>C: Set-Cookie: refresh_token=...
        else User not found / wrong password
            A->>A: timing-safe dummy check
            G-->>C: 401 Unauthorized
        end
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

**Request Body:**

```json
{
  "email": "user@example.com",
  "password": "SecurePass123"
}
```

**Success Response (200):**

```json
{
  "access_token": "eyJhbG..."
}
```

**Error Responses:**

| Код | Описание | Пример |
|-----|----------|--------|
| `401 INVALID_CREDENTIALS` | Неверные учётные данные | `{"error": {"code": "INVALID_CREDENTIALS", "message": "..."}}` |
| `403 EMAIL_NOT_CONFIRMED` | Email не подтверждён | `{"error": {"code": "EMAIL_NOT_CONFIRMED", "message": "..."}}` |
| `429 RATE_LIMITED` | Превышен лимит запросов | `{"error": {"code": "RATE_LIMITED", "message": "..."}}` |

**Response 401 Unauthorized (INVALID_CREDENTIALS):**

```json
{
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "Неверные учётные данные",
    "details": {}
  }
}
```

**Response 403 Forbidden (EMAIL_NOT_CONFIRMED):**

```json
{
  "error": {
    "code": "EMAIL_NOT_CONFIRMED",
    "message": "Пожалуйста, подтвердите ваш email",
    "details": {}
  }
}
```

**Response 429 Too Many Requests (RATE_LIMITED):**

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Превышен лимит запросов",
    "details": { "retry_after": 3600 }
  }
}
```

**Пример curl:**

```bash
curl -X POST https://api.mystore.com/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "SecurePass123"
  }'
```

---

## 4.5 POST /v1/auth/refresh

**Описание:** Получение нового access token с rotation refresh token

**Описание схемы:**
Схема показывает процесс обновления access token с rotation refresh token. Клиент отправляет POST-запрос на `/v1/auth/refresh`, передавая access token в заголовке Authorization и refresh token как HTTP-only cookie. API Gateway проверяет rate limit по IP. Auth Service сначала проверяет, не находится ли токен в черном списке (Redis). Если токен активен, выполняется валидация: проверка подписи RS256, срок действия, привязка к IP-адресу и user agent hash. При успешной проверке старый токен помещается в черный список с TTL = оставшееся время жизни, помечается как revoked в БД, генерируются новые access token и refresh token (rotation). При несоответствии параметров привязки или истечении срока возвращается 401 Unauthorized. При превышении rate limit — 429 Too Many Requests.

**Схема:**
```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL

    C->>G: POST /v1/auth/refresh
    note right of C: access_token в header<br/>refresh_token в cookie
    G->>R: check rate limit (IP)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>R: GET revoked_tokens:{hash}
        alt Not revoked
            A->>PG: SELECT * FROM refresh_tokens WHERE jti = ?
            PG-->>A: token record
            alt Token valid
                A->>A: verify signature (RS256)
                A->>A: check expires_at
                A->>A: check ip_address
                A->>A: check user_agent_hash
                alt All checks passed
                    A->>R: SET revoked_tokens:{old_hash}
                    A->>PG: UPDATE refresh_tokens SET revoked = TRUE
                    A->>A: generate new tokens
                    A->>PG: INSERT INTO refresh_tokens
                    G-->>C: 200 OK
                    G->>C: Set-Cookie: refresh_token=... (new)
                else Token binding failed
                    G-->>C: 401 Unauthorized
                end
            else Token expired/invalid
                G-->>C: 401 Unauthorized
            end
        else Token revoked
            G-->>C: 401 Unauthorized
        end
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

**Headers:**

```
Authorization: Bearer <access_token>
```

**Refresh Token Location:** HTTP-only cookie (`refresh_token`)

**Success Response (200):**

```json
{
  "access_token": "eyJhbG...",
  "token_type": "bearer",
  "expires_in": 1800
}
```

**Примечание:** `refresh_token` не возвращается в response body, так как он автоматически устанавливается как HTTP-only cookie.

**Error Responses:**

| Код | Описание | Пример |
|-----|----------|--------|
| `401 REFRESH_TOKEN_INVALID` | Неверный или истекший токен обновления | `{"error": {"code": "REFRESH_TOKEN_INVALID", "message": "..."}}` |
| `429 RATE_LIMITED` | Превышен лимит запросов | `{"error": {"code": "RATE_LIMITED", "message": "..."}}` |

**Response 401 Unauthorized (REFRESH_TOKEN_INVALID):**

```json
{
  "error": {
    "code": "REFRESH_TOKEN_INVALID",
    "message": "Неверный или истёкший токен обновления",
    "details": {}
  }
}
```

**Response 429 Too Many Requests (RATE_LIMITED):**

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Превышен лимит запросов",
    "details": { "retry_after": 3600 }
  }
}
```

**Пример curl:**

```bash
curl -X POST https://api.mystore.com/v1/auth/refresh \
  -H "Authorization: Bearer eyJhbG..." \
  --cookie-jar cookies.txt \
  --cookie cookies.txt
```

---

## 4.6 POST /v1/auth/logout

**Описание:** Выход из системы (инвалидация refresh token текущей сессии)

> **Важно:** Logout завершает **только текущую сессию**. В других браузерах или устройствах пользователь остаётся в системе.

**Описание схемы:**
Схема показывает процесс выхода из системы (инвалидация refresh token текущей сессии). Клиент отправляет POST-запрос на `/v1/auth/logout`, передавая access token в заголовке Authorization и refresh token как HTTP-only cookie. API Gateway проверяет rate limit по IP. Auth Service проверяет, не был ли токен уже отозван. При активном токене он добавляется в черный список Redis с TTL = 604800 секунд (7 дней), а запись в PostgreSQL помечается как revoked. HTTP-only cookie очищается через `Set-Cookie: refresh_token=; Max-Age=0`. Важно: logout инвалидирует только текущую сессию — другие активные сессии остаются действительными. При повторном вызове logout для уже отозванного токена возвращается 200 OK (idempotent). При превышении rate limit — 429 Too Many Requests.

**Схема:**
```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL

    C->>G: POST /v1/auth/logout
    note right of C: access_token в header<br/>refresh_token в cookie
    G->>R: check rate limit (IP)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>R: GET revoked_tokens:{hash}
        alt Not revoked
            A->>PG: SELECT refresh_tokens WHERE jti = ?
            PG-->>A: token record
            alt Token valid
                A->>R: SET revoked_tokens:{hash} EX 604800
                A->>PG: UPDATE refresh_tokens SET revoked = TRUE
                G->>C: Clear refresh cookie (Max-Age=0)
                G-->>C: 200 OK
            else Token invalid
                G-->>C: 401 Unauthorized
            end
        else Token already revoked
            G-->>C: 200 OK
        end
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

**Headers:**

```
Authorization: Bearer <access_token>
```

**Success Response (200):**

```json
{
  "message": "Вы успешно вышли из системы"
}
```

**Error Responses:**

| Код | Описание | Пример |
|-----|----------|--------|
| `401 ACCESS_TOKEN_INVALID` | Неверный или истекший токен доступа | `{"error": {"code": "ACCESS_TOKEN_INVALID", "message": "..."}}` |

**Response 401 Unauthorized (ACCESS_TOKEN_INVALID):**

```json
{
  "error": {
    "code": "ACCESS_TOKEN_INVALID",
    "message": "Неверный или истёкший токен доступа",
    "details": {}
  }
}
```

**Как работает logout:**

1. Инвалидирует текущий refresh token (`revoked = TRUE`)
2. Добавляет `jti` в Redis blacklist с TTL = оставшееся время жизни
3. Очищает cookie через `Set-Cookie: refresh_token=; Max-Age=0`
4. **Не инвалидирует другие активные сессии**

**Access token после logout:**
- Access token **не инвалидируется** и остаётся действительным до `exp` (30 минут)
- Клиент должен удалить access token из памяти

**Пример curl:**

```bash
curl -X POST https://api.mystore.com/v1/auth/logout \
  -H "Authorization: Bearer eyJhbG..."
```

---

## 4.7 GET /v1/auth/me

**Описание:** Получение данных текущего пользователя

**Описание схемы:**
Схема показывает процесс получения данных текущего пользователя. Клиент отправляет GET-запрос на `/v1/auth/me`, передавая access token в заголовке Authorization. API Gateway или Auth Service валидирует токен (проверка подписи RS256 и срока действия). При валидном токене из PostgreSQL извлекаются данные пользователя (id, email, name, role, флаг верификации, created_at) и возвращаются клиенту. При невалидном токене возвращается 401 Unauthorized.

**Схема:**
```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant A as Auth Service
    participant PG as PostgreSQL

    C->>G: GET /v1/auth/me
    note right of C: access_token в header
    G->>A: Validate access token
    alt Token valid
        A->>PG: SELECT * FROM users WHERE id = ?
        PG-->>A: user record
        A-->>C: 200 OK (user data)
    else Token invalid
        A-->>C: 401 Unauthorized
    end
```

**Headers:**

```
Authorization: Bearer <access_token>
```

**Success Response (200):**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "name": "Иван Иванов",
  "role": "user",
  "is_email_verified": true,
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Error Responses:**

| Код | Описание | Пример |
|-----|----------|--------|
| `401 ACCESS_TOKEN_INVALID` | Неверный или истекший токен доступа | `{"error": {"code": "ACCESS_TOKEN_INVALID", "message": "..."}}` |

**Response 401 Unauthorized (ACCESS_TOKEN_INVALID):**

```json
{
  "error": {
    "code": "ACCESS_TOKEN_INVALID",
    "message": "Неверный или истёкший токен доступа",
    "details": {}
  }
}
```

**Пример curl:**

```bash
curl https://api.mystore.com/v1/auth/me \
  -H "Authorization: Bearer eyJhbG..."
```

---

## 4.8 POST /v1/auth/resend-verification

**Описание:** Повторная отправка email с подтверждением

**Описание схемы:**
Схема показывает процесс повторной отправки email с подтверждением. Клиент отправляет POST-запрос на `/v1/auth/resend-verification` с email. API Gateway проверяет rate limit по IP. Auth Service ищет пользователя по email. Если пользователь существует, проверяется наличие pending токена. Если есть pending токен и за последний час отправлено менее 3 писем, старые токены аннулируются, выпускается новый и отправляется email. Если pending токена нет, создается новый токен и отправляется email. Если пользователь не найден, возвращается 200 OK (но письмо не отправляется для безопасности). При превышении лимита 3 писем за час — 429 RATE_LIMITED. При превышении общего rate limit — 429 Too Many Requests.

**Схема:**
```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL
    participant SMTP as SMTP

    C->>G: POST /v1/auth/resend-verification
    G->>R: check rate limit (IP)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>PG: SELECT * FROM users WHERE email = ?
        PG-->>A: user record / null
        alt User exists
            A->>PG: SELECT * FROM email_verification_tokens WHERE user_id = ?
            alt Has pending token
                A->>A: check tokens count (last hour)
                alt Count < 3
                    A->>PG: UPDATE tokens SET used_at = NOW() (old)
                    A->>PG: INSERT INTO email_verification_tokens (new)
                    A->>SMTP: send verification email
                    G-->>C: 200 OK
                else Count >= 3
                    G-->>C: 429 RATE_LIMITED
                end
            else No pending token
                A->>PG: INSERT INTO email_verification_tokens
                A->>SMTP: send verification email
                G-->>C: 200 OK
            end
        else User not found
            G-->>C: 200 OK (no email sent for security)
        end
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

**Request Body:**

```json
{
  "email": "user@example.com"
}
```

**Success Response (200):**

```json
{
  "message": "Письмо с подтверждением отправлено. Пожалуйста, проверьте почту."
}
```

**Error Responses:**

| Код | Описание |
|-----|----------|
| `409 EMAIL_ALREADY_VERIFIED` | Email уже подтверждён |
| `429 RATE_LIMITED` | Превышен лимит запросов |

**Response 409 Conflict (EMAIL_ALREADY_VERIFIED):**

```json
{
  "error": {
    "code": "EMAIL_ALREADY_VERIFIED",
    "message": "Email уже подтверждён",
    "details": {}
  }
}
```

**Response 429 Too Many Requests (RATE_LIMITED):**

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Превышен лимит запросов",
    "details": { "retry_after": 3600 }
  }
}
```

**Поведение:**

| Состояние учётной записи | Действие | Ответ |
| ----------------------- | -------- | ----- |
| Email уже подтверждён | Письмо не отправляется | `200 OK` с сообщением `{"message": "Email уже подтверждён"}` |
| 3+ писем за последний час | Письмо не отправляется | `429 RATE_LIMITED` |
| Есть pending-токен, < 3 писем | Старые токены аннулируются, выпускается новый | `200 OK` |
| Учётной записи не существует | Письмо не отправляется | `200 OK` (для безопасности) |

**Пример curl:**

```bash
curl -X POST https://api.mystore.com/v1/auth/resend-verification \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com"
  }'
```

---

## 4.9 POST /v1/auth/forgot-password

**Описание:** Запрос на сброс пароля

**Описание схемы:**
Схема показывает процесс запроса сброса пароля. Клиент отправляет POST-запрос на `/v1/auth/forgot-password`. API Gateway проверяет rate limit по комбинации IP и email. Auth Service ищет пользователя в PostgreSQL по email. Если пользователь найден, генерируется UUIDv7-токен сброса, сохраняется в базе со сроком действия 1 час, и отправляется email со ссылкой для сброса. Если пользователь не найден, выполняется timing-safe dummy операция (чтобы не раскрывать существование аккаунта) и всё равно возвращается 200 OK. Важно: система не раскрывает, существует ли email — это защита от enumeration атак. При превышении rate limit — 429 Too Many Requests.

**Схема:**
```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL
    participant SMTP as SMTP

    C->>G: POST /v1/auth/forgot-password
    G->>R: check rate limit (IP:email)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>PG: SELECT * FROM users WHERE email = ?
        PG-->>A: user record / null
        alt User found
            A->>A: generate reset token (UUIDv7)
            A->>PG: INSERT INTO reset_password_tokens
            A->>SMTP: send reset email with token
            SMTP-->>A: queued
            G-->>C: 200 OK
        else User not found
            A->>A: timing-safe dummy operation
            G-->>C: 200 OK (no email for security)
        end
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

**Request Body:**

```json
{
  "email": "user@example.com"
}
```

**Success Response (200):**

```json
{
  "message": "Инструкции по сбросу пароля отправлены на email."
}
```

**Behavior:** Не раскрывает существование email (для безопасности). Возвращает `200 OK` для любых запросов.

**Error Responses:**

| Код | Описание |
|-----|----------|
| `429 RATE_LIMITED` | Превышен лимит запросов |

**Response 429 Too Many Requests (RATE_LIMITED):**

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Превышен лимит запросов",
    "details": { "retry_after": 3600 }
  }
}
```

**Пример curl:**

```bash
curl -X POST https://api.mystore.com/v1/auth/forgot-password \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com"
  }'
```

---

## 4.10 POST /v1/auth/reset-password

**Описание:** Сброс пароля с токеном

**Описание схемы:**
Схема показывает процесс сброса пароля с использованием токена. Клиент отправляет POST-запрос на `/v1/auth/reset-password` с токеном из email и новым паролем. API Gateway проверяет rate limit по комбинации IP и email. Auth Service ищет токен в базе данных PostgreSQL. Если токен найден и не истёк, пароль хешируется с помощью Argon2id, обновляется в базе, а токен помечается как использованный. Если токен не найден или истёк, возвращается ошибка 400. При превышении rate limit — 429 Too Many Requests.

**Схема:**
```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL

    C->>G: POST /v1/auth/reset-password
    G->>R: check rate limit (IP:email)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>PG: SELECT * FROM reset_password_tokens WHERE token = ?
        PG-->>A: token record / null
        alt Token found & valid
            A->>A: hash new password (Argon2id)
            A->>PG: UPDATE users SET password_hash = ?
            A->>PG: UPDATE reset_password_tokens SET used_at = NOW()
            G-->>C: 200 OK
        else Token expired/not found
            G-->>C: 400 INVALID_TOKEN
        end
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

**Request Body:**

```json
{
  "token": "reset-token-from-email",
  "password": "NewSecurePass123"
}
```

**Success Response (200):**

```json
{
  "message": "Пароль успешно сброшен."
}
```

**Error Responses:**

| Код | Описание |
|-----|----------|
| `400 INVALID_TOKEN` | Неверный или истекший токен |
| `400 EXPIRED_TOKEN` | Токен сброса пароля истёк |
| `404 TOKEN_NOT_FOUND` | Токен не найден |
| `429 RATE_LIMITED` | Превышен лимит запросов |

**Response 400 Bad Request (INVALID_TOKEN):**

```json
{
  "error": {
    "code": "INVALID_TOKEN",
    "message": "Неверный или истёкший токен сброса пароля",
    "details": {}
  }
}
```

**Response 400 Bad Request (EXPIRED_TOKEN):**

```json
{
  "error": {
    "code": "EXPIRED_TOKEN",
    "message": "Токен сброса пароля истёк",
    "details": {}
  }
}
```

**Response 404 Not Found (TOKEN_NOT_FOUND):**

```json
{
  "error": {
    "code": "TOKEN_NOT_FOUND",
    "message": "Токен сброса пароля не найден",
    "details": {}
  }
}
```

**Response 429 Too Many Requests (RATE_LIMITED):**

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Превышен лимит запросов",
    "details": { "retry_after": 3600 }
  }
}
```

**Пример curl:**

```bash
curl -X POST https://api.mystore.com/v1/auth/reset-password \
  -H "Content-Type: application/json" \
  -d '{
    "token": "reset-token-from-email",
    "password": "NewSecurePass123"
  }'
```

---

## 4.11 POST /v1/auth/unsubscribe

**Описание:** Отписка от коммерческой рассылки (GDPR/КоАП)

**Описание схемы:**
Схема показывает процесс отписки от email-рассылки. Клиент отправляет POST-запрос на `/v1/auth/unsubscribe` с токеном отписки из email. Auth Service ищет пользователя в PostgreSQL по unsubscribe_token. Если пользователь найден, токен обнуляется в базе (удаляется), и возвращается 200 OK. Если пользователь не найден, возвращается 404 USER_NOT_FOUND. Эндпоинт не требует аутентификации — токен является достаточным доказательством прав на отписку.

**Схема:**
```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant A as Auth Service
    participant PG as PostgreSQL

    C->>G: POST /v1/auth/unsubscribe
    G->>A: Process unsubscribe
    A->>PG: SELECT * FROM users WHERE unsubscribe_token = ?
    PG-->>A: user record / null
    alt User found
        A->>PG: UPDATE users SET unsubscribe_token = NULL
        G-->>C: 200 OK
    else User not found
        G-->>C: 404 USER_NOT_FOUND
    end
```

**Request Body:**

```json
{
  "token": "unsubscribe-token-from-email"
}
```

**Success Response (200):**

```json
{
  "message": "Успешная отписка от рассылки"
}
```

**Error Responses:**

| Код | Описание |
|-----|----------|
| `400 INVALID_TOKEN` | Неверный или истекший токен |
| `404 USER_NOT_FOUND` | Пользователь не найден |

**Response 400 Bad Request (INVALID_TOKEN):**

```json
{
  "error": {
    "code": "INVALID_TOKEN",
    "message": "Неверный или истёкший токен отписки",
    "details": {}
  }
}
```

**Response 404 Not Found (USER_NOT_FOUND):**

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "Пользователь не найден",
    "details": {}
  }
}
```

**Пример curl:**

```bash
curl -X POST https://api.mystore.com/v1/auth/unsubscribe \
  -H "Content-Type: application/json" \
  -d '{
    "token": "unsubscribe-token-from-email"
  }'
```

---

## 4.12 PUT /v1/auth/change-password

**Описание:** Изменение пароля

**Описание схемы:**
Схема показывает процесс изменения пароля пользователем. Клиент отправляет PUT-запрос на `/v1/auth/change-password` с текущим паролем и новым паролем в теле запроса. Authorization header содержит access token. API Gateway проверяет rate limit по IP. Auth Service валидирует access token, затем проверяет текущий пароль. Если текущий пароль верный, новый пароль хешируется и обновляется в базе. Если текущий пароль неверный, возвращается 400 INVALID_CREDENTIALS. При превышении rate limit — 429 Too Many Requests.

**Схема:**
```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL

    C->>G: PUT /v1/auth/change-password
    note right of C: access_token в header
    G->>R: check rate limit (IP)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>A: validate access token
        A->>A: verify current_password
        alt Current password valid
            A->>A: hash new password
            A->>PG: UPDATE users SET password_hash = ?
            G-->>C: 200 OK
        else Current password invalid
            G-->>C: 400 INVALID_CREDENTIALS
        end
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request Body:**

```json
{
  "current_password": "OldPass123",
  "new_password": "NewSecurePass456"
}
```

**Success Response (200):**

```json
{
  "message": "Пароль успешно изменен."
}
```

**Error Responses:**

| Код | Описание |
|-----|----------|
| `400 INVALID_CREDENTIALS` | Текущий пароль неверный |
| `400 WEAK_PASSWORD` | Новый пароль не соответствует требованиям |
| `401 ACCESS_TOKEN_INVALID` | Неверный или истекший токен доступа |
| `429 RATE_LIMITED` | Превышен лимит запросов |

**Response 400 Bad Request (INVALID_CREDENTIALS):**

```json
{
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "Текущий пароль неверный",
    "details": {}
  }
}
```

**Response 400 Bad Request (WEAK_PASSWORD):**

```json
{
  "error": {
    "code": "WEAK_PASSWORD",
    "message": "Новый пароль не соответствует требованиям безопасности",
    "details": {}
  }
}
```

**Response 401 Unauthorized (ACCESS_TOKEN_INVALID):**

```json
{
  "error": {
    "code": "ACCESS_TOKEN_INVALID",
    "message": "Неверный или истёкший токен доступа",
    "details": {}
  }
}
```

**Response 429 Too Many Requests (RATE_LIMITED):**

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Превышен лимит запросов",
    "details": { "retry_after": 3600 }
  }
}
```

**Пример curl:**

```bash
curl -X PUT https://api.mystore.com/v1/auth/change-password \
  -H "Authorization: Bearer eyJhbG..." \
  -H "Content-Type: application/json" \
  -d '{
    "current_password": "OldPass123",
    "new_password": "NewSecurePass456"
  }'
```

---

## 4.13 DELETE /v1/auth/me

**Описание:** Удаление аккаунта текущего пользователя (GDPR compliance)

**Описание схемы:**
Схема показывает процесс полного удаления аккаунта пользователя (GDPR right to be forgotten). Клиент отправляет DELETE-запрос на `/v1/auth/me` с access token в header. API Gateway валидирует токен. Если токен валиден, Auth Service проверяет, подтверждён ли email (обязательное условие для удаления). Если email подтверждён, последовательно удаляются: refresh tokens, email verification tokens, reset password tokens, login attempts и сама запись пользователя из БД. Также очищаются все сессии пользователя в Redis. Если email не подтверждён, возвращается 403 EMAIL_NOT_VERIFIED.

**Схема:**
```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant A as Auth Service
    participant PG as PostgreSQL
    participant R as Redis

    C->>G: DELETE /v1/auth/me
    note right of C: access_token в header
    G->>A: Validate access token
    alt Token valid
        A->>A: check is_email_verified
        alt Email verified
            A->>PG: DELETE FROM refresh_tokens WHERE user_id = ?
            A->>PG: DELETE FROM email_verification_tokens WHERE user_id = ?
            A->>PG: DELETE FROM reset_password_tokens WHERE user_id = ?
            A->>PG: DELETE FROM login_attempts WHERE email = ?
            A->>PG: DELETE FROM users WHERE id = ?
            A->>R: DEL session:*
            G-->>C: 200 OK
        else Email not verified
            G-->>C: 403 EMAIL_NOT_VERIFIED
        end
    else Token invalid
        G-->>C: 401 Unauthorized
    end
```

**Headers:**

```
Authorization: Bearer <access_token>
```

**Success Response (200):**

```json
{
  "message": "Аккаунт удалён навсегда."
}
```

**Error Responses:**

| Код | Описание |
|-----|----------|
| `401 ACCESS_TOKEN_INVALID` | Неверный или истекший токен доступа |
| `403 EMAIL_NOT_VERIFIED` | Email не подтверждён |

**Response 401 Unauthorized (ACCESS_TOKEN_INVALID):**

```json
{
  "error": {
    "code": "ACCESS_TOKEN_INVALID",
    "message": "Неверный или истёкший токен доступа",
    "details": {}
  }
}
```

**Response 403 Forbidden (EMAIL_NOT_VERIFIED):**

```json
{
  "error": {
    "code": "EMAIL_NOT_VERIFIED",
    "message": "Пожалуйста, подтвердите ваш email перед удалением аккаунта",
    "details": {}
  }
}
```

**Пример curl:**

```bash
curl -X DELETE https://api.mystore.com/v1/auth/me \
  -H "Authorization: Bearer eyJhbG..."
```
