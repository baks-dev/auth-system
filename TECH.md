# Техническая спецификация: Система регистрации и авторизации

| Файл         | Описание                                                                      |
|--------------|-------------------------------------------------------------------------------|
| [README.md](./README.md) | Главная страница спецификации                                     |
| [PRODUCT.md](./PRODUCT.md) | Продуктовая спецификация — что делается, для кого               |
| [TECH.md](./TECH.md)    | Техническая спецификация — архитектура, API, БД, безопасность      |

---

## 1. Архитектура

```
┌─────────────┐
│   Browser   │
│  (Client)   │
└──────┬──────┘
       │ HTTPS
       │
┌──────▼────────────────────────────────────┐
│             API Gateway / Load Balancer   │
│             - Rate limiting (Redis)       │
│             - SSL termination             │
└──────┬────────────────────────────────────┘
       │
       │ HTTP
       │
┌──────▼────────────────────────────────────┐
│         Application Server                │
│  ┌─────────────────────────────────────┐  │
│  │  /v1/auth/* Routes                  │  │
│  │  - register, login, verify          │  │
│  │  - refresh, logout, me              │  │
│  │  - resend-verification              │  │
│  │  - forgot-password, reset-password  │  │
│  └─────────────────────────────────────┘  │
│  ┌─────────────────────────────────────┐  │
│  │  Auth Service                       │  │
│  │  - JWT generation/verification      │  │
│  │  - Password hashing (argon2id)      │  │
│  │  - Token management                 │  │
│  │  - Redis client (rate limiting)     │  │
│  │  - SMTP client (email sending)      │  │
│  └─────────────────────────────────────┘  │
└──────┬────────────────────────────────────┘
       │
       │ PostgreSQL                  Redis               SMTP Server
       │                 │            │                  │
┌──────▼────────────────▼────────────▼──────────────────▼────┐
│                   Data Layer                               │
│  ┌──────────────┐  ┌─────────────┐  ┌──────────────────┐  │
│  │  users       │  │  rate_limits│  │  Email Queue     │  │
│  │  tokens      │  │  sessions   │  │  SMTP Client     │  │
│  │  login_attempts│ │  locks      │  │  Templates       │  │
│  └──────────────┘  └─────────────┘  └──────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

---

## 1.1 Диаграмма потоков данных

### Регистрация пользователя

```
┌──────────┐    1. POST /register    ┌──────────────────┐
│  Client  │ ──────────────────────> │   API Gateway    │
└──────────┘                         └────────┬─────────┘
                                               │
                         2. Rate limit check   │
                                         ┌─────▼───────┐
                                         │   Redis       │
                                         │   (check)     │
                                         └───────────────┘
                                               │
                3. Validate & Hash password    │
                                    ┌──────────▼──────────┐
                                    │   Auth Service      │
                                    │   - Validate input  │
                                    │   - Argon2id hash   │
                                    └──────────┬──────────┘
                                               │
                      4. Store user + token    │
                                    ┌──────────▼──────────┐
                                    │   PostgreSQL        │
                                    │   - users table     │
                                    │   - tokens table    │
                                    └──────────┬──────────┘
                                               │
                 5. Send verification email    │
                                    ┌──────────▼──────────┐
                                    │   SMTP Client       │
                                    │   - Email template  │
                                    │   - Queue delivery  │
                                    └─────────────────────┘
                                               │
                     6. Return success         │
                                    ┌──────────▼──────────┐
                                    │   Client response   │
                                    └─────────────────────┘
```

### Аутентификация (Login)

```
┌──────────┐    1. POST /login       ┌──────────────────┐
│  Client  │ ──────────────────────> │   API Gateway    │
└──────────┘                         └────────┬─────────┘
                                               │
                         2. Rate limit check   │
                                    ┌──────────▼──────────┐
                                    │   Redis             │
                                    │   - IP counter      │
                                    └─────────────────────┘
                                               │
              3. Find user + verify password   │
                                    ┌──────────▼──────────┐
                                    │   Auth Service      │
                                    │   - DB query        │
                                    │   - Argon2id check  │
                                    └──────────┬──────────┘
                                               │
               4. Generate tokens + store      │
                                    ┌──────────▼──────────┐
                                    │   Redis             │
                                    │   - Store session   │
                                    └─────────────────────┘
                                               │
                    5. Return tokens           │
                                    ┌──────────▼──────────┐
                                    │   Client response   │
                                    └─────────────────────┘
```

### Refresh токена

```
┌──────────┐    1. POST /refresh     ┌──────────────────┐
│  Client  │ ──────────────────────> │   API Gateway    │
└──────────┘                         └────────┬─────────┘
                                               │
                         2. Rate limit check   │
                                    ┌──────────▼──────────┐
                                    │   Redis             │
                                    └─────────────────────┘
                                               │
            3. Validate refresh token          │
                                    ┌──────────▼──────────┐
                                    │   Auth Service      │
                                    │   - Check Redis     │
                                    │   - Check DB        │
                                    └──────────┬──────────┘
                                               │
        4. Invalidate old + Generate new       │
                                    ┌──────────▼──────────┐
                                    │   Redis             │
                                    │   - Store new       │
                                    └─────────────────────┘
                                               │
                  5. Return new tokens         │
                                    ┌──────────▼──────────┐
                                    │   Client response   │
                                    └─────────────────────┘
```

### Logout

```
┌──────────┐    1. POST /logout      ┌──────────────────┐
│  Client  │ ──────────────────────> │   API Gateway    │
└──────────┘                         └────────┬─────────┘
                                               │
                         2. Rate limit check   │
                                    ┌──────────▼──────────┐
                                    │   Redis             │
                                    └─────────────────────┘
                                               │
          3. Revoke refresh token              │
                                    ┌──────────▼──────────┐
                                    │   Auth Service      │
                                    │   - Mark revoked    │
                                    │   - Store in Redis  │
                                    └──────────┬──────────┘
                                               │
                  4. Return success            │
                                    ┌──────────▼──────────┐
                                    │   Client response   │
                                    └─────────────────────┘
```

---

## 2. JWT Структура

### 2.1 Access Token

**Header:**
```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

**Payload:**
```json
{
  "jti": "uuid-v7",           // JWT ID (unique identifier for token tracking)
  "sub": "uuid-v7",           // user ID (time-based)
  "email": "user@example.com",
  "name": "Иван Иванов",
  "role": "user",             // "user" | "admin"
  "is_email_verified": true,
  "auth_time": 1726989600,    // authentication time (unix timestamp)
  "iat": 1726989600,          // issued at (unix timestamp)
  "exp": 1726991400           // expires at (unix timestamp)
}
```

**Срок жизни:** 30 минут  
**Алгоритм:** RS256  
**Ключ:** Приватный ключ хранится в переменной окружения `JWT_PRIVATE_KEY` (PEM format)  
**Верификация:** Публичный ключ доступен через `/.well-known/jwks.json` или из `JWT_PUBLIC_KEY`

### 2.3 Обработка истекших Access Tokens

**Стратегия:** Token refresh при истечении в середине запроса

| Время до exp | Действие | Ответ |
|--------------|----------|-------|
| `exp - now > 5s` | Обычный запрос | 200 OK |
| `exp - now <= 5s` | Авто-обновление (только safe methods) | 200 OK + `X-Token-Refresh: true` header |
| `exp - now < 0` (просрочен) | Требуется refresh | 401 Unauthorized + `X-Token-Status: expired` |

**Правила:**
- **Safe methods (GET, HEAD, OPTIONS):** Автоматически обновляют access token через refresh, если истек менее 5 секунд назад
- **Unsafe methods (POST, PUT, DELETE):** Возвращают 401 без авто-обновления
- **Header:** При авто-обновлении добавляется `X-Token-Refresh: true` для информирования клиента
- **Логирование:** Все авто-обновления логируются с `event: "token.autorefresh"`

**Рекомендация для клиентов:**
- При получении 401 с `X-Token-Status: expired` выполнить refresh token flow
- При `X-Token-Refresh: true` можно продолжить работу (token обновлен прозрачно)

### 2.2 Refresh Token

**Header:**
```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

**Payload:**
```json
{
  "jti": "uuid-v7",           // JWT ID (unique identifier for revocation tracking)
  "sub": "uuid-v7",           // user ID
  "email": "user@example.com",
  "iat": 1726989600,          // issued at (unix timestamp)
  "exp": 1727076000           // expires at (unix timestamp, 7 days)
}
```

**Срок жизни:** 7 дней  
**Хранение:** HTTP-only cookie (домен: `api.mystore.com`)  
**Одноразовость:** После использования инвалидируется в БД и Redis  
**Revocation:** `jti` добавляется в Redis blacklist с TTL = оставшееся время жизни

---

## 3. Database Schema

### 3.1 users

| Поле | Тип | Описание |
|------|-----|----------|
| id | UUID | PRIMARY KEY |
| email | VARCHAR(255) | UNIQUE, NOT NULL |
| name | VARCHAR(255) | NOT NULL |
| password_hash | TEXT | NOT NULL |
| role | VARCHAR(20) | DEFAULT 'user' |
| is_email_verified | BOOLEAN | DEFAULT false |
| email_verified_at | TIMESTAMP | NULL |
| unsubscribe_token | VARCHAR(255) | UNIQUE, NULL (для отписки от рассылки) |
| created_at | TIMESTAMP | DEFAULT NOW() |
| updated_at | TIMESTAMP | DEFAULT NOW() |

**Описание полей:**
- `id` — уникальный идентификатор пользователя (UUID v7)
- `email` — email для входа и коммуникации, уникальный
- `name` — отображаемое имя пользователя
- `password_hash` — хэш пароля (argon2id), без соли
- `role` — роль пользователя: 'user' или 'admin'
- `is_email_verified` — флаг подтвержденного email
- `email_verified_at` — время подтверждения email (после verification)
- `created_at` — время создания записи
- `updated_at` — время последнего обновления

**Индексы:**
- `idx_users_email` (email) — для быстрого поиска при login/registration
- `idx_users_role` (role) — для фильтрации по ролям
- `idx_users_unsubscribe_token` (unsubscribe_token) — для быстрой отписки

### 3.2 email_verification_tokens

| Поле | Тип | Описание |
|------|-----|----------|
| id | UUID | PRIMARY KEY |
| user_id | UUID | REFERENCES users(id) ON DELETE CASCADE |
| token | VARCHAR(255) | UNIQUE, NOT NULL (URL-safe base64 UUID) |
| expires_at | TIMESTAMP | NOT NULL |
| created_at | TIMESTAMP | DEFAULT NOW() |
| used_at | TIMESTAMP | NULL |

**Описание полей:**
- `id` — уникальный идентификатор токена (UUID v7)
- `user_id` — референс на пользователя, каскадное удаление (CASCADE)
- `token` — одноразовый токен для подтверждения, URL-safe base64 UUID
- `expires_at` — время истечения токена (24 часа от создания)
- `created_at` — время генерации токена
- `used_at` — время использования токена (после verify, NULL если не использован)

**Индексы:**
- `idx_tokens_token` (token) — для быстрого поиска по token
- `idx_tokens_user_id` (user_id) — для поиска активных токенов пользователя
- `idx_tokens_expires_at` (expires_at) — для очистки истекших токенов

### 3.3 refresh_tokens

| Поле | Тип | Описание |
|------|-----|----------|
| id | UUID | PRIMARY KEY |
| user_id | UUID | REFERENCES users(id) ON DELETE CASCADE |
| token | VARCHAR(512) | UNIQUE, NOT NULL (hashed) |
| ip_address | VARCHAR(45) | NOT NULL (IP при выдаче токена) |
| user_agent_hash | VARCHAR(64) | SHA-256 хэш от базовой информации user_agent |
| revoked | BOOLEAN | DEFAULT false |
| revoked_at | TIMESTAMP | NULL |
| expires_at | TIMESTAMP | NOT NULL |
| created_at | TIMESTAMP | DEFAULT NOW() |

**Описание полей:**
- `id` — уникальный идентификатор токена (UUID v7)
- `user_id` — референс на пользователя, каскадное удаление (CASCADE)
- `token` — хэшированный SHA-256 refresh token (оригинал хранится в HTTP-only cookie)
- `ip_address` — IP-адрес (IPv4 или IPv6) при выдаче токена (для аудита)
- `user_agent_hash` — SHA-256 хэш от агрегированной информации user_agent (browser/os/device), без деталей
- `revoked` — флаг инвалидации (true после logout или компрометации)
- `revoked_at` — время инвалидации токена (NULL если активен)
- `expires_at` — время истечения токена (7 дней от создания)
- `created_at` — время выдачи токена

**Revocation Flow:**
1. При logout: `UPDATE refresh_tokens SET revoked = TRUE, revoked_at = NOW() WHERE jti = ?`
2. При проверке refresh: `WHERE jti = ? AND revoked = FALSE AND expires_at > NOW()`
3. Очистка: удаление revoked токенов старше N дней (cron-задача)

**Индексы:**
- `idx_refresh_token` (token) — для быстрого поиска (по хэшу)
- `idx_refresh_user_id` (user_id) — для поиска активных токенов
- `idx_refresh_expires_at` (expires_at) — для очистки истекших токенов
- `idx_refresh_ip` (ip_address) — для аудита по IP

### 3.4 reset_password_tokens

| Поле | Тип | Описание |
|------|-----|----------|
| id | UUID | PRIMARY KEY |
| user_id | UUID | REFERENCES users(id) ON DELETE CASCADE |
| token | VARCHAR(255) | UNIQUE, NOT NULL (UUIDv7) |
| expires_at | TIMESTAMP | NOT NULL (1 час от создания) |
| created_at | TIMESTAMP | DEFAULT NOW() |
| used_at | TIMESTAMP | NULL |

**Описание полей:**
- `id` — уникальный идентификатор токена (UUID v7)
- `user_id` — референс на пользователя, каскадное удаление
- `token` — UUIDv7 для сброса пароля, одноразовый
- `expires_at` — время истечения токена (1 час от создания)
- `created_at` — время генерации токена
- `used_at` — время использования токена (после reset, NULL если не использован)

**Индексы:**
- `idx_reset_token` (token) — для быстрого поиска
- `idx_reset_user_id` (user_id) — для поиска активных токенов
- `idx_reset_expires_at` (expires_at) — для очистки истекших токенов

### 3.5 login_attempts

| Поле | Тип | Описание |
|------|-----|----------|
| id | UUID | PRIMARY KEY |
| email | VARCHAR(255) | NOT NULL |
| ip_address | VARCHAR(45) | NOT NULL |
| user_agent_hash | VARCHAR(64) | SHA-256 хэш от базовой информации user_agent (для аудита без идентификации) |
| success | BOOLEAN | NOT NULL |
| failed_at | TIMESTAMP | DEFAULT NOW() |

**Описание полей:**
- `id` — уникальный идентификатор записи (UUID v7)
- `email` — email, с которого производилась попытка входа
- `ip_address` — IP-адрес (IPv4 или IPv6, VARCHAR(45) для поддержки IPv6)
- `user_agent_hash` — SHA-256 хэш от агрегированной информации user_agent (browser/os/device), без деталей
- `success` — результат попытки: true (успех) или false (неудача)
- `failed_at` — время попытки входа

**Индексы:**
- `idx_attempts_email_ip` (email, ip_address) — для анализа атак
- `idx_attempts_failed_at` (failed_at) — для очистки старых записей
- `idx_attempts_email_success` (email, success) — для статистики
- `idx_attempts_user_agent_hash` (user_agent_hash) — для группировки по типам устройств

**Комментарии:**
- Логируются все попытки входа (успешные и неуспешные)
- Для rate limiting: count за последние N минут по IP
- Для блокировки: count неудачных по email за час
- `user_agent_hash` хранится для аудита без возможности восстановления полного user_agent (GDPR compliance)

---

### 3.5 Redis Keys (для rate limiting и session management)

| Ключ | Тип | Описание | TTL |
|------|-----|----------|-----|
| `rate_limit:login:{ip}` | String | Счетчик логинов по IP | 10 мин |
| `rate_limit:login:block:{email}` | String | Блокировка аккаунта после неудач | 15-24 часа |
| `rate_limit:register:{ip}` | String | Счетчик регистраций по IP | 1 час |
| `rate_limit:verify:{ip}` | String | Счетчик верификаций по IP | 1 час |
| `rate_limit:forgot-password:{ip}` | String | Счетчик запросов сброса по IP | 1 час |
| `rate_limit:reset-password:{ip}` | String | Счетчик сбросов по IP | 1 час |
| `rate_limit:refresh:{ip}` | String | Счетчик refresh запросов по IP | 10 мин |
| `rate_limit:change-password:{ip}` | String | Счетчик смен паролей по IP | 1 час |
| `session:{refresh_token_hash}` | Hash | Информация о сессии (user_id, created_at, jti, user_agent_hash) | 7 дней |
| `revoked_tokens:{refresh_token_hash}` | String | Флаг инвалидации токена | 7 дней |
| `revoked_access_jti:{jti}` | String | Флаг инвалидации access token по jti | 30 минут |
| `lock:register:{email}` | String | Блокировка после неудачной рег-ции | 1 час |
| `lock:forgot-password:{email}` | String | Блокировка после неудачного сброса | 1 час |
| `user_agent:hash:{hash}` | String | Метаинформация по хэшу user_agent (browser/os/device) | 30 дней |

**Redis Commands used:**
- `INCR` / `EXPIRE` — счетчики rate limiting
- `HSET` / `HGET` — хранение session data
- `SET` / `GET` / `GETSET` — флаги revoked tokens и access jti
- `DEL` — удаление истекших записей

**Revocation Flow:**
1. При logout: `SET revoked_tokens:{refresh_token_hash} 1 EX 604800`
2. При истечении access token: проверка `GET revoked_access_jti:{jti}`
3. При refresh: инвалидация старого refresh token + создание нового с новым jti

**Redis Commands used:**
- `INCR` / `EXPIRE` — счетчики rate limiting
- `HSET` / `HGET` — хранение session data
- `SET` / `GET` / `GETSET` — флаги revoked tokens и access jti
- `DEL` — удаление истекших записей

**Revocation Flow:**
1. При logout: `SET revoked_tokens:{refresh_token_hash} 1 EX 604800`
2. При истечении access token: проверка `GET revoked_access_jti:{jti}`
3. При refresh: инвалидация старого refresh token + создание нового с новым jti

---

## 4. Endpoints

### 4.1 POST /v1/auth/register

**Описание:** Регистрация нового пользователя

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123",
  "name": "Иван Иванов"
}
```

**Validation Rules:**
- `email`: required, valid email format
- `password`: required, min 8 characters
- `name`: required, min 1 character

**Success Response (201):**

```bash
curl -X POST https://api.mystore.com/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "SecurePass123",
    "name": "Иван Иванов"
  }'
```

```json
{
  "message": "Registration successful. Please check your email to verify your account.",
  "email": "user@example.com"
}
```

**Error Responses:**
- `400`: Validation errors
- `409`: Email already registered
- `429`: Rate limited

**Response 400 Bad Request:**
```json
{
  "error": "validation_failed",
  "details": {
    "email": "Неверный формат email",
    "password": "Пароль должен содержать минимум 8 символов",
    "name": "Имя не может быть пустым"
  }
}
```

**Response 409 Conflict:**
```json
{
  "error": "email_exists",
  "message": "Пользователь с таким email уже зарегистрирован",
  "details": {}
}
```

**Response 429 Too Many Requests:**
```json
{
  "error": "rate_limited",
  "message": "Превышен лимит запросов",
  "details": { "retry_after": 3600 }
}
```

---

### 4.2 POST /v1/auth/verify

**Описание:** Подтверждение email с одноразовым токеном

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
  "refresh_token": "eyJhbG...",
  "token_type": "bearer",
  "expires_in": 1800
}
```

**Error Responses:**
- `400`: Invalid or expired token
- `409`: Email already verified


**Response 400 Bad Request:**
```json
{
  "error": "invalid_token",
  "message": "Неверный или просроченный токен подтверждения",
  "details": {}
}
```

---

### 4.3 POST /v1/auth/login

**Описание:** Авторизация пользователя

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123"
}
```

**Success Response (200):**

```bash
curl -X POST https://api.mystore.com/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "SecurePass123"
  }'
```

```json
{
  "access_token": "eyJhbG...",
  "refresh_token": "eyJhbG...",
  "token_type": "bearer",
  "expires_in": 1800
}
```

**Error Responses:**
- `401`: Invalid credentials email
- `403`: Email not confirmed 
- `429`: Rate limited


**Response 401 Unauthorized:**
```json
{
  "error": "invalid_credentials",
  "message": "Неверный email или пароль"
}
```

**Response 403 Forbidden:**
```json
{
  "error": "email_not_confirmed",
  "message": "Пожалуйста, подтвердите email перед входом"
}
```

**Response 429 Too Many Requests:**
```json
{
  "error": "rate_limited",
  "message": "Превышен лимит запросов",
  "details": { "retry_after": 3600 }
}
```

---

### 4.4 POST /v1/auth/refresh

**Описание:** Получение нового access token

**Headers:**
```
Authorization: Bearer eyJhbG...
```

**Refresh Token Location:** HTTP-only cookie (`refreshToken`)

**Success Response (200):**

```bash
curl -X POST https://api.mystore.com/v1/auth/refresh \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

```json
{
  "access_token": "eyJhbG...",
  "refresh_token": "eyJhbG...",
  "token_type": "bearer",
  "expires_in": 1800
}
```

**Error Responses:**
- `401`: Invalid or expired token
- `403`: Token revoked
- `429`: Rate limited

**Response 429 Too Many Requests:**
```json
{
  "error": "rate_limited",
  "message": "Превышен лимит запросов",
  "details": { "retry_after": 3600 }
}
```


---

### 4.5 POST /v1/auth/logout

**Описание:** Выход из системы (инвалидация refresh token)

**Headers:**
```
Authorization: Bearer eyJhbG...
```

**Success Response (200):**
```json
{
  "message": "Logged out successfully"
}
```

**Error Responses:**
- `401`: Invalid access token

**Response 401 Unauthorized:**
```json
{
  "error": "invalid_token",
  "message": "Неверный или просроченный access token"
}
```

---

### 4.6 GET /v1/auth/me

**Описание:** Получение данных текущего пользователя

**Headers:**
```
Authorization: Bearer eyJhbG...
```

**Success Response (200):**

```bash
curl https://api.mystore.com/v1/auth/me \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

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
- `401`: Invalid access token

**Response 401 Unauthorized:**
```json
{
  "error": "invalid_token",
  "message": "Неверный или просроченный access token"
}
```

---

### 4.7 POST /v1/auth/resend-verification

**Описание:** Повторная отправка email с подтверждением

**Request Body:**
```json
{
  "email": "user@example.com"
}
```

**Success Response (200):**
```json
{
  "message": "Verification email sent. Please check your inbox."
}
```

**Error Responses:**
- `409`: Email already verified
- `429`: Rate limited

**Behavior for already verified email:** Возвращает `200 OK` с сообщением `{"message": "Email already verified"}` (идемпотентное поведение)

---

### 4.8 POST /v1/auth/forgot-password

**Описание:** Запрос на сброс пароля

**Request Body:**
```json
{
  "email": "user@example.com"
}
```

**Success Response (200):**
```json
{
  "message": "Password reset instructions sent to your email."
}
```

**Behavior:** Не раскрывает существование email (для безопасности). Возвращает `200 OK` для любых запросов (существующих и несуществующих email).

**Error Responses:**
- `429`: Rate limited

---

### 4.9 POST /v1/auth/reset-password

**Описание:** Сброс пароля с токеном

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
  "message": "Password has been reset successfully."
}
```

---

### 4.10 POST /v1/auth/unsubscribe

**Описание:** Отписка от коммерческой рассылки (GDPR/КоАП)

**Request Body:**
```json
{
  "token": "unsubscribe-token-from-email"
}
```

**Success Response (200):**
```json
{
  "message": "Successfully unsubscribed from newsletter"
}
```

**Error Responses:**
- `400`: Invalid or expired token
- `404`: User not found

---

### 4.11 PUT /v1/auth/change-password

**Описание:** Изменение пароля

**Headers:**
```
Authorization: Bearer eyJhbG...
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
  "message": "Password has been changed successfully."
}
```

---

## 5. Security Implementation

### 5.1 Rate Limiting

**Per-endpoint limits:**

| Endpoint | Лимит | Period | Блокировка | Логика сброса |
|----------|-------|--------|------------|---------------|
| `/login` | 5 | 10 мин | 15 мин | При успешном входе |
| `/register` | 3 | 1 час | 1 час | При успешной регистрации |
| `/verify` | 10 | 1 час | 30 мин | При успешной верификации |
| `/forgot-password` | 3 | 1 час | 1 час | При успешном сбросе |
| `/reset-password` | 5 | 1 час | 1 час | При успешном сбросе |
| `/refresh` | 30 | 10 мин | — | Без блокировки |
| `/change-password` | 5 | 1 час | 1 час | При успешной смене |

**Algorithm:**
1. На каждую неудачную попытку логируем IP + email
2. Если за период лимита превышено количество запросов → 429
3. Для `/login` и `/register`: после 5 неудачных попыток включается блокировка
   - 5 неудачных → блокировка 15 минут
   - 10 неудачных → блокировка 24 часа

**Redis Implementation:**
- **Key pattern:** `rate_limit:{endpoint}:{identifier}:{window}`
- **Identifier:** IP для общего лимита, email для блокировки аккаунта
- **TTL:** Period + 5 минут (автоматическая очистка)
- **Counter:** INCR/EXPIRE для каждого identifier
- **Block key:** `lock:{endpoint}:{identifier}` с TTL = duration

**Timing Attack Protection:**
- Использовать константное сравнение для всех критичных проверок:
  - `crypto.timingSafeEqual()` для сравнения хэшей паролей
  - `crypto.timingSafeEqual()` для сравнения токенов
  - `hmac.compare_digest()` (Python) / `ConstantTimeCompare()` (Go)
- Избегать раннего выхода из функций сравнения

### 5.2 Password Security

- **Algorithm:** Argon2id (memory: 64MB, iterations: 3, parallelism: 4)
- **Minimum length:** 8 символов (рекомендуется 12+)
- **No password policy** (не требуем специальные символы для удобства)
- **Timing-safe comparison:** Обязательное константное сравнение хэшей

### 5.3 Token Security

**Refresh Token Revocation:**
- При logout токен **помечается как revoked в БД** (`UPDATE refresh_tokens SET revoked = TRUE, revoked_at = NOW()`)
- Одновременно `jti` токена добавляется в Redis blacklist с TTL = оставшееся время жизни
- **Токен НЕ удаляется из БД** (для аудита и предотвращения reuse)
- Очистка revoked токенов: периодический cron-джоб удаления токенов, revoked = TRUE и expired более N дней назад

**Refresh Token Structure:**
- `revoked` (BOOLEAN, default FALSE) — флаг инвалидации
- `revoked_at` (TIMESTAMP, NULL) — время инвалидации
- При проверке refresh token: `WHERE jti = ? AND revoked = FALSE AND expires_at > NOW()`

**Access Token:**
- RS256, 30 минут, хранение в памяти
- Каждый токен имеет уникальный `jti` (UUIDv7)
- `jti` хранится в Redis с TTL=30 минут для отслеживания
- Авто-обновление при истечении (только safe methods)
- При logout: `revoked_access_jti:{jti}` добавляется в Redis с TTL=30 минут

**Refresh Token:**
- RS256, 7 дней, HTTP-only cookie + Redis blacklist
- После использования инвалидируется в БД и Redis
- Single-use refresh: токен инвалидируется после использования

**Timing Attack Protection:**
- Использовать константное сравнение для всех критичных проверок:
  - `crypto.timingSafeEqual()` для сравнения хэшей паролей
  - `crypto.timingSafeEqual()` для сравнения токенов
  - `hmac.compare_digest()` (Python) / `ConstantTimeCompare()` (Go)
- Избегать раннего выхода из функций сравнения

### 5.4 Email Security

- **Verification tokens:** URL-safe base64 UUID, TTL = 24 часа (см. 3.2 email_verification_tokens)
- **Reset tokens:** UUIDv7, TTL = 1 час (см. 3.4 reset_password_tokens)
- **Forgot-password tokens:** UUIDv7, TTL = 1 час
- **Unsubscribe tokens:** UUIDv7, хранятся в users.unsubscribe_token, не истекают
- **Tokens одноразовые:** После использования помечаются как использованные в БД (used_at)
- **No email enumeration:** Ответы не раскрывают наличие email
- **Timing-safe comparison:** Обязательное константное сравнение токенов при verify/reset

### 5.5 CSRF Protection

**Для endpoints с HTTP-only cookie (refresh token):**

| Endpoint | Cookie | CSRF Protection |
|----------|--------|-----------------|
| `/login` | Нет | CSRF token + SameSite=None (если cross-origin) |
| `/register` | Нет | CSRF token + SameSite=None (если cross-origin) |
| `/refresh` | HTTP-only cookie | SameSite=Strict + Origin validation |
| `/logout` | HTTP-only cookie | SameSite=Strict + Origin validation |
| `/me` | Нет | Origin validation |
| `/change-password` | HTTP-only cookie | SameSite=Strict + CSRF token (опционально для API) |

**Обязательные защиты:**
- `SameSite=Strict` для всех cookie (не отправляются при cross-site запросах)
- `Secure` флаг для cookie (только HTTPS)
- `HttpOnly` флаг для refresh token cookie
- **Origin validation** на сервере для всех запросов
- **Referer/Preferrer policy** заголовки

**Пример проверки Origin:**
```javascript
const allowedOrigins = ['https://mystore.com', 'https://www.mystore.com'];
const origin = req.headers.origin;
if (!allowedOrigins.includes(origin)) {
  return res.status(403).json({ error: 'invalid_origin' });
}
```

---

## 5.7 Threat Model

| Угроза | Митигация |
|--------|-----------|
| Brute force атака | Rate limiting (Redis), блокировка по IP/email, timing-safe comparison |
| Token stealing | HTTP-only cookies, short-lived access tokens, refresh token rotation, revocation tracking |
| SQL Injection | Prepared statements (ORM), parameterized queries |
| XSS | Content-Security-Policy, sanitize user input, HTTP headers |
| CSRF | SameSite=Strict cookies, Origin/Referer validation, CSRF tokens для чувствительных операций |
| Password cracking | Argon2id (memory-hard), rate limiting, breach checking (опционально) |
| Email interception | TLS 1.3+ для SMTP, одноразовые токены, короткий срок действия |
| Timing Attack | Константное сравнение хэшей (`timingSafeEqual`, `ConstantTimeCompare`) для всех критичных проверок |
| Token reuse | Refresh token инвалидируется после использования, revoked флаг в БД + Redis blacklist |
| Account enumeration | Ответы не раскрывают наличие email, timing-safe comparison при verify/login |
| Session hijacking | Short-lived access tokens, HTTP-only cookies, Origin validation |

---

## 5.9 Compliance

- **GDPR:** Возможность удаления аккаунта (см. раздел 4.6 DELETE /v1/auth/me)
- **PII protection:** Email хранится в зашифрованном виде (опционально)
- **Audit logging:** Все операции логируются (см. раздел 7)

---

## 5.10 Implementation Checklist

**Backend:**
- [ ] Константное сравнение для всех критичных проверок (password hash, tokens)
- [ ] Проверка Origin/Referer для всех auth endpoints
- [ ] SameSite=Strict для всех cookie
- [ ] Secure флаг для cookie (только HTTPS в production)
- [ ] HTTP-only флаг для refresh token cookie
- [ ] Rate limiting для всех endpoints (Redis)
- [ ] Блокировка аккаунтов после превышения лимита неудач
- [ ] Revocation flow для logout (UPDATE revoked + Redis blacklist)
- [ ] Проверка revoked флага при refresh
- [ ] Очистка старых revoked токенов (cron-задача)
- [ ] Генерация unsubscribe_token при регистрации пользователя
- [ ] Endpoint POST /v1/auth/unsubscribe для обработки отписок

---

## 6. Email Integration

### 6.1 SMTP Client

**Library:** Nodemailer (Node.js) / SendGrid / AWS SES (production)

**Configuration:**
```javascript
{
  host: process.env.SMTP_HOST,
  port: parseInt(process.env.SMTP_PORT),
  secure: process.env.SMTP_PORT === '465',
  auth: {
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS
  },
  tls: {
    ciphers: 'SSLv3',
    minVersion: 'TLSv1.2'
  }
}
```

**Features:**
- Connection pooling (5-10 connections)
- Retry logic (3 attempts, exponential backoff)
- Queue system (BullMQ) для высокой нагрузки
- Bounce detection и handling

### 6.2 Template: Registration Confirmation

**Subject:** Подтверждение email для MyStore

**Body:**
```
Добро пожаловать в MyStore, {name}!

Пожалуйста, подтвердите свой email, перейдя по ссылке:
https://mystore.com/verify?token={token}

Ссылка действует 24 часа.

--- 
Управление рассылкой:
Если вы больше не хотите получать письма от MyStore, отпишитесь:
https://mystore.com/unsubscribe?token={unsubscribe_token}

Если вы не регистрировались в MyStore, проигнорируйте это письмо.
```

**Template variables:**
- `{name}` — имя пользователя
- `{token}` — email verification token (для GET-страницы verify)
- `{unsubscribe_token}` — токен для отписки от рассылки (UUIDv7 из users.unsubscribe_token)
- `{company}` — название компании (MyStore)

### 6.3 Template: Password Reset

**Subject:** Сброс пароля для MyStore

**Body:**
```
{email}, вы запросили сброс пароля.

Перейдите по ссылке для установки нового пароля:
https://mystore.com/reset-password?token={token}

Ссылка действует 1 час.

--- 
Управление рассылкой:
Если вы больше не хотите получать письма от MyStore, отпишитесь:
https://mystore.com/unsubscribe?token={unsubscribe_token}

Если вы не запрашивали сброс пароля, проигнорируйте это письмо.
```

**Template variables:**
- `{email}` — email пользователя
- `{token}` — password reset token (для GET-страницы reset-password)
- `{unsubscribe_token}` — токен для отписки от рассылки (UUIDv7 из users.unsubscribe_token)
- `{company}` — название компании

### 6.4 Email Queue (для высокой нагрузки)

**Technology:** BullMQ (Redis-based queue)

**Queue name:** `emails`

**Job types:**
- `verification` — email verification
- `password_reset` — password reset
- `unsubscribe` — handling unsubscribe requests
- `notification` — general notifications

**Worker configuration:**
```javascript
{
  concurrency: 10,        // parallel workers
  attempts: 3,            // retry on failure
  delay: 5000,            // 5s delay between retries
  backoff: 'exponential'  // exponential backoff
}
```

### 6.5 Environment Variables

```
# SMTP Configuration
SMTP_HOST=smtp.mystore.com
SMTP_PORT=587
SMTP_USER=noreply@mystore.com
SMTP_PASS=***
SMTP_FROM=noreply@mystore.com

# Email URLs (для ссылок в письмах)
APP_URL=https://mystore.com
APP_VERIFY_URL=https://mystore.com/verify
APP_RESET_PASSWORD_URL=https://mystore.com/reset-password
APP_UNSUBSCRIBE_URL=https://mystore.com/unsubscribe

# Email Queue (BullMQ)
REDIS_URL=redis://localhost:6379
EMAIL_QUEUE_PREFIX=emails
```

---

## 7. Logging

### 7.1 Event Schema

```json
{
  "timestamp": "2024-09-22T10:30:00.000Z",
  "event": "user.registered" | "user.verified" | "user.logged_in" | "user.logged_out" | "auth.failed" | "token.autorefresh" | "auth.token_refreshed",
  "user_id": "01a0c637-3aa0-73d4-b85e-8f89aa81e711",
  "email": "user@example.com",
  "ip_address": "192.168.1.1",
  "user_agent": {
    "browser": "Chrome",
    "browser_version": "128.0",
    "os": "Windows",
    "os_version": "11",
    "device": "Desktop"
  },
  "details": {
    "jti": "01hv...",      // JWT ID (для token.autorefresh и auth.token_refreshed)
    "jti_old": "01hv...",  // старый JWT ID (для auth.token_refreshed)
    "request_method": "GET",
    "path": "/api/users"
  }
}
```

**Примечание:** `user_agent` хранится в агрегированном виде без деталей, которые могут идентифицировать пользователя (согласно GDPR принципу минимизации данных).

### 7.2 Critical Events

- `user.registered` — логировать user_id, email, ip_address, user_agent (базовая информация)
- `user.verified` — логировать user_id, email, ip_address
- `user.logged_in` — логировать user_id, email, ip_address, user_agent (базовая информация)
- `user.logged_out` — логировать user_id, email, ip_address
- `auth.failed` — логировать email, ip_address, user_agent (базовая информация)
- `token.autorefresh` — логировать jti, метод запроса и путь (при авто-обновлении access token)
- `auth.token_refreshed` — логировать user_id, email, ip_address, jti (новый токен), jti_old (старый токен), user_agent (базовая информация)

---

## 8. Environment Variables

| Переменная | Обязательная | Описание |
|------------|--------------|----------|
| `NODE_ENV` | Да | development, staging, production |
| `PORT` | Да | Порт сервера (по умолчанию: 3000) |
| `JWT_PRIVATE_KEY` | Да | Приватный ключ для RS256 (PEM format, base64-encoded) |
| `JWT_PUBLIC_KEY` | Да | Публичный ключ для верификации RS256 (PEM format, base64-encoded) |
| `DATABASE_URL` | Да | PostgreSQL connection string |
| `REDIS_URL` | Да | Redis connection string (включая пароль, если есть) |
| `SMTP_HOST` | Да | SMTP сервер |
| `SMTP_PORT` | Да | SMTP порт (587 для TLS, 465 для SSL) |
| `SMTP_USER` | Да | SMTP пользователь |
| `SMTP_PASS` | Да | SMTP пароль |
| `SMTP_FROM` | Да | From email адрес |
| `SMTP_TIMEOUT` | Нет | Таймаут SMTP соединения (по умолчанию: 30s) |
| `SMTP_TLS_MIN_VERSION` | Нет | Минимальная версия TLS (по умолчанию: TLSv1.2) |
| `APP_URL` | Да | URL приложения (для email ссылок) |
| `APP_VERIFY_URL` | Нет | URL страницы верификации (по умолчанию: APP_URL/verify) |
| `APP_RESET_PASSWORD_URL` | Нет | URL страницы сброса пароля (по умолчанию: APP_URL/reset-password) |
| `APP_UNSUBSCRIBE_URL` | Нет | URL страницы отписки (по умолчанию: APP_URL/unsubscribe) |
| `RATE_LIMIT_WINDOW_MS` | Нет | Окно rate limiting в мс (по умолчанию: 60000) |
| `RATE_LIMIT_MAX` | Нет | Максимальное кол-во запросов (по умолчанию: 100) |
| `LOG_LEVEL` | Нет | debug, info, warn, error (по умолчанию: info) |

### 8.1 Пример конфигурации (production)

```
NODE_ENV=production
PORT=3000

JWT_PRIVATE_KEY=LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2Z0lCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQktjd2dnU2pBZ0VBQW9JQkFRREZuWk1QeGNuQlBZ...
JWT_PUBLIC_KEY=LS0tLS1CRUdJTiBQVUJMSWMgS0VZLS0tLS0KTUlJQklqQU5CZ2txaGtpRzl3MEJBUXNGQUFCQ0NBU0N3Z2dFa01BMEdDU3FHU0liM0RRRUIvVUFNQlR4UXdIeV...

DATABASE_URL=postgresql://user:pass@localhost:5432/auth_db?ssl=true

REDIS_URL=redis://:password@localhost:6379/0

SMTP_HOST=smtp.mystore.com
SMTP_PORT=587
SMTP_USER=noreply@mystore.com
SMTP_PASS=smtp-password-here
SMTP_FROM=noreply@mystore.com
SMTP_TIMEOUT=30000

APP_URL=https://mystore.com
APP_VERIFY_URL=https://mystore.com/verify
APP_RESET_PASSWORD_URL=https://mystore.com/reset-password
APP_UNSUBSCRIBE_URL=https://mystore.com/unsubscribe
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=100

LOG_LEVEL=info
```

---

## 9. Deployment Checklist

### Database
- [ ] Создать таблицы (users, email_verification_tokens, refresh_tokens, login_attempts)
- [ ] Создать индексы
- [ ] Настроить резервное копирование

### Backend
- [ ] Настроить переменные окружения
- [ ] Настроить rate limiting (Redis)
- [ ] Настроить email сервер
- [ ] Настроить логирование
- [ ] Настроить мониторинг (метрики)

### Security
- [ ] SSL/TLS на load balancer
- [ ] HTTP-only cookie для refresh tokens
- [ ] CORS настроен правильно
- [ ] XSS защита (см. раздел 5.5 Security Headers)
- [ ] SQL injection защита (prepared statements)

---

## 10. Testing Strategy

### Unit Tests
- Хэширование паролей (argon2id)
- Подпись/проверка JWT (RS256)
- Генерация токенов электронной почты 
- Ограничение запросов / Rate limiting
- **JWT ID (jti) генерация и уникальность**
- **Revocation check по jti в Redis**

### Интеграционные тесты
- Полный процесс регистрации / registration
- Процесс подтверждения электронной почты / verification
- Процесс входа в систему / login
- Обновления токена / Token refresh
- Процесс выхода из системы / logout
- Ограничение запросов / Rate limiting
- **Auto-refresh access token при истечении (safe methods)**
- **Revocation по jti (logout и истечение срока)**

### E2E Tests
- сценарии используя фреймворки для E2E тестирования (Cypress/Playwright)
- Проверка UI (если есть)
