# Техническая спецификация: Система регистрации и авторизации

| Файл                         | Описание                                                      |
| ---------------------------- | ------------------------------------------------------------- |
| [README.md](./README.md)     | Главная страница спецификации                                 |
| [PRODUCT.md](./PRODUCT.md)   | Продуктовая спецификация — что делается, для кого             |
| [TECH.md](./TECH.md)         | Техническая спецификация — архитектура, API, БД, безопасность |
| [GLOSSARY.md](./GLOSSARY.md) | Глоссарий терминов                                            |

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

## 1.1 Data Flow Diagrams / Диаграмма потоков данных

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

### Аутентификация

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

### Обновление токена

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

### Выход из системы

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

## 2. Структура JWT

### 2.1 Токен доступа

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
  "iss": "mystore-auth-service", // идентификатор сервиса
  "sub": "uuid-v7", // ID пользователя (time-based)
  "iat": 1726989600, // время выдачи (Unix timestamp)
  "exp": 1726991400 // время истечения (Unix timestamp, 30 минут)
}
```

**Срок жизни:** 30 минут  
**Алгоритм:** RS256  
**Ключ:** Приватный ключ хранится в переменной окружения `JWT_PRIVATE_KEY` (формат PEM)  
**Верификация:** Публичный ключ доступен через `/.well-known/jwks.json` или из `JWT_PUBLIC_KEY`

**Пояснение для `iss` (issuer):** Идентификатор сервиса используется при валидации токена для подтверждения, что токен выдан доверенным авторизационным сервером, а не подделан. При проверке подписи также проверяется, что значение `iss` в токене совпадает с ожидаемым идентификатором сервиса.

### Проверка `iss` при валидации access token

**Ожидаемое значение:** `mystore-auth-service`

**Процесс проверки:**

1. **Декодирование JWT** — access token декодируется без проверки подписи (header + payload в base64url)
2. **Проверка `iss`** — значение поля `iss` сравнивается с ожидаемым `mystore-auth-service`
3. **Проверка подписи** — если `iss` валиден, выполняется проверка RS256 подписи с помощью публичного ключа
4. **Проверка `exp`** — проверка срока действия токена

**Что происходит при несовпадении `iss`:**
- Если `iss` не равен `mystore-auth-service`, токен отклоняется с ошибкой `401 Unauthorized`
- Код ошибки: `ACCESS_TOKEN_INVALID`
- Сообщение: "Неверный или истекший токен доступа"
- **Важно:** Ошибка не раскрывает детали (не указывает, что именно `iss` не совпал) — это предотвращает information disclosure

**Псевдокод валидации access token:**

```javascript
function validateAccessToken(token) {
  try {
    // 1. Декодирование без проверки подписи
    const decoded = jwt.decode(token, { complete: true });
    if (!decoded) {
      throw new Error('Invalid token');
    }

    // 2. Проверка iss (issuer)
    if (decoded.payload.iss !== 'mystore-auth-service') {
      throw new Error('Invalid issuer');
    }

    // 3. Проверка подписи (RS256 с публичным ключом)
    jwt.verify(token, publicKey, { algorithms: ['RS256'] });

    // 4. Проверка exp (expiration)
    const now = Math.floor(Date.now() / 1000);
    if (decoded.payload.exp < now) {
      throw new Error('Token expired');
    }

    return decoded.payload;
  } catch (err) {
    throw new Error('ACCESS_TOKEN_INVALID');
  }
}
```

**Примечание:** Email и имя не включаются в access token, так как payload JWT не шифруется, а профиль может измениться за время жизни токена. Профиль клиент получает через `GET /v1/users/me`.

### 2.3 Обработка истекших токенов доступа

**Стратегия:** Token refresh (авто-обновление) при истечении в середине запроса

| Время до exp                | Действие                                                  | Ответ                                                  |
| --------------------------- | --------------------------------------------------------- | ------------------------------------------------------ |
| `exp - now > 5s`            | Обычный запрос                                            | 200 OK                                                 |
| `exp - now <= 5s`           | Авто-обновление (только safe methods: GET, HEAD, OPTIONS) | 200 OK + заголовок `X-Token-Refresh: true`             |
| `exp - now < 0` (просрочен) | Требуется refresh                                         | 401 Unauthorized + заголовок `X-Token-Status: expired` |

**Правила:**

- **Safe methods (GET, HEAD, OPTIONS):** Автоматически обновляют access token через refresh, если истек менее 5 секунд назад
- **Unsafe methods (POST, PUT, DELETE):** Возвращают 401 без авто-обновления
- **Header:** При авто-обновлении добавляется `X-Token-Refresh: true` для информирования клиента
- **Логирование:** Все авто-обновления логируются с `event: token.autorefresh`

**Рекомендация для клиентов:**

- При получении 401 с `X-Token-Status: expired` выполнить flow refresh token
- При `X-Token-Refresh: true` можно продолжить работу (token обновлён прозрачно)

### 2.4 Токен обновления

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
  "iss": "mystore-auth-service", // идентификатор сервиса
  "sub": "uuid-v7", // ID пользователя
  "iat": 1726989600, // время выдачи (Unix timestamp)
  "exp": 1727076000 // время истечения (Unix timestamp, 7 дней)
}
```

**Срок жизни:** 7 дней  
**Хранение:** HTTP-only cookie (домен: `api.mystore.com`, путь: `/`)  
**Token Binding:** Refresh token привязан к `ip_address` и `user_agent_hash` (проверяются при валидации)  
**Rotation:** При каждом refresh генерируется новый refresh token, старый инвалидируется (одноразовость)  
**Revocation:** `jti` добавляется в Redis blacklist с TTL = оставшееся время жизни

**Пояснение для `iss` (issuer):** То же, что и для access token — подтверждение, что токен выдан доверенным авторизационным сервером.

---

## 2.5 Использование HTTP-only cookie для refresh token

**Архитектура хранения:**

| Токен | Место хранения | Доступ | Безопасность |
|-------|----------------|--------|--------------|
| Access token | Response body (JSON) | JS-доступ | XSS уязвим |
| Refresh token | HTTP-only cookie | Недоступен JS | XSS защищён |

**Настройка cookie:**

```javascript
// Пример установки cookie сервером
res.cookie('refresh_token', refreshTokenValue, {
  httpOnly: true,           // Недоступен через JavaScript (защита от XSS)
  secure: true,             // Только HTTPS (в production)
  sameSite: 'Strict',       // Защита от CSRF (не отправляется при cross-site запросах)
  domain: 'api.mystore.com',
  path: '/',
  maxAge: 7 * 24 * 60 * 60 * 1000  // 7 дней в миллисекундах
});
```

**Как работает flow:**

1. **При login/verify:** Сервер устанавливает refresh token в HTTP-only cookie
2. **При refresh/logout:** Браузер автоматически отправляет cookie с каждым запросом
3. **При logout:** Сервер инвалидирует токен и может очистить cookie (через `Set-Cookie: refresh_token=; Max-Age=0`)

**Преимущества HTTP-only cookie:**

- **XSS защита:** JavaScript не может прочитать или украсть refresh token
- **Автоматическая отправка:** Браузер сам добавляет cookie к запросам (не нужно хранить в localStorage/Redux)
- **SameSite protection:** `SameSite=Strict` предотвращает отправку cookie при cross-site запросах

**Ограничения:**

- **Domain ограничение:** Cookie отправляется только на `api.mystore.com` (или поддомены с `domain=.mystore.com`)
- **HTTPS рекомендация:** `Secure` флаг требует HTTPS в production
- **CSRF защита:** SameSite=Strict предотвращает большинство CSRF атак, но для критичных операций может потребоваться дополнительная защита (CSRF token)

**Клиентская логика:**

```javascript
// При login - сервер устанавливает cookie
fetch('/v1/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ email, password }),
  credentials: 'include'  // Важно: включает cookie в запрос
});

// При refresh - cookie отправляется автоматически
fetch('/v1/auth/refresh', {
  method: 'POST',
  credentials: 'include'
});

// При logout - cookie отправляется автоматически
fetch('/v1/auth/logout', {
  method: 'POST',
  credentials: 'include'
});
```

---

## 3. Схема базы данных

### 3.1 Таблица users

| Поле              | Тип          | Описание                               |
| ----------------- | ------------ | -------------------------------------- |
| id                | UUID         | PRIMARY KEY                            |
| email             | VARCHAR(255) | UNIQUE, NOT NULL                       |
| name              | VARCHAR(255) | NOT NULL                               |
| password_hash     | TEXT         | NOT NULL                               |
| role              | VARCHAR(20)  | DEFAULT 'user'                         |
| is_email_verified | BOOLEAN      | DEFAULT false                          |
| email_verified_at | TIMESTAMP    | NULL                                   |
| unsubscribe_token | VARCHAR(255) | UNIQUE, NULL (для отписки от рассылки) |
| created_at        | TIMESTAMP    | DEFAULT NOW()                          |
| updated_at        | TIMESTAMP    | DEFAULT NOW()                          |

**Описание полей:**

- `id` — уникальный идентификатор пользователя (UUIDv7)
- `email` — email для входа и коммуникации, уникальный
- `name` — отображаемое имя пользователя
- `password_hash` — хэш пароля (Argon2id), без соли
- `role` — роль пользователя: `user` или `admin`
- `is_email_verified` — флаг подтверждённого email
- `email_verified_at` — время подтверждения email (после verify)
- `created_at` — время создания записи
- `updated_at` — время последнего обновления

**Индексы:**

- `idx_users_email` (email) — для быстрого поиска при login/registration (аутентификация/регистрация)
- `idx_users_role` (role) — для фильтрации по ролям
- `idx_users_unsubscribe_token` (unsubscribe_token) — для быстрой отписки

### 3.2 Таблица email_verification_tokens

| Поле       | Тип          | Описание                                |
| ---------- | ------------ | --------------------------------------- |
| id         | UUID         | PRIMARY KEY                             |
| user_id    | UUID         | REFERENCES users(id) ON DELETE CASCADE  |
| token      | VARCHAR(255) | UNIQUE, NOT NULL (URL-safe base64 UUID) |
| expires_at | TIMESTAMP    | NOT NULL                                |
| created_at | TIMESTAMP    | DEFAULT NOW()                           |
| used_at    | TIMESTAMP    | NULL                                    |

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

**Использование:**

- При `/v1/auth/verify` токен удаляется из таблицы (одноразовость)
- Если email уже подтверждён (`is_email_verified = true`), возвращается `409 Conflict`
- Токен может быть использован только один раз:

```sql
DELETE FROM email_verification_tokens 
WHERE token = ? 
AND user_id = ? 
AND used_at IS NULL 
AND expires_at > NOW()
```

**Одноразовость токена:**

- В каждый момент у учётной записи действителен только последний выданный токен
- При повторной генерации токена (например, через `/v1/auth/resend-verification`) прежние токены **аннулируются**:
  - Старые записи обновляются: `used_at = NOW()` (физически не удаляются, но становятся невалидными)
  - Генерируется новый токен с новым `user_id` и `expires_at`
- Проверка валидности токена включает `used_at IS NULL` — это гарантирует, что даже если токен скомпрометирован, он не может быть использован повторно

### 3.3 Таблица refresh_tokens

| Поле            | Тип          | Описание                                                                     |
| --------------- | ------------ | ---------------------------------------------------------------------------- |
| id              | UUID         | PRIMARY KEY                                                                  |
| user_id         | UUID         | REFERENCES users(id) ON DELETE CASCADE                                       |
| token           | VARCHAR(512) | UNIQUE, NOT NULL (hashed)                                                    |
| ip_address      | VARCHAR(45)  | NOT NULL (IP при выдаче токена)                                              |
| user_agent_hash | VARCHAR(64)  | NOT NULL (SHA-256 хэш от агрегированной информации user_agent)               |
| revoked         | BOOLEAN      | DEFAULT false                                                                |
| revoked_at      | TIMESTAMP    | NULL                                                                         |
| expires_at      | TIMESTAMP    | NOT NULL                                                                     |
| parent_token_id | UUID         | REFERENCES refresh_tokens(id) NULL (для отслеживания rotation)               |
| created_at      | TIMESTAMP    | DEFAULT NOW()                                                                |

**Описание полей:**

- `id` — уникальный идентификатор токена (UUID v7)
- `user_id` — референс на пользователя, каскадное удаление (CASCADE)
- `token` — хэшированный SHA-256 refresh token (оригинал хранится в HTTP-only cookie)
- `ip_address` — IP-адрес (IPv4 или IPv6) при выдаче токена (для аудита)
- `user_agent_hash` — SHA-256 хэш от агрегированной информации user_agent (browser/os/device)
- `revoked` — флаг инвалидации (true после logout или компрометации)
- `revoked_at` — время инвалидации токена (NULL если активен)
- `expires_at` — время истечения токена (7 дней от создания)
- `parent_token_id` — ссылка на родительский токен при rotation (NULL для исходного токена)
- `created_at` — время выдачи токена

**Revocation Flow (с token binding):**

1. При logout: `UPDATE refresh_tokens SET revoked = TRUE, revoked_at = NOW() WHERE jti = ? AND user_id = ?`
2. При проверке refresh: `WHERE jti = ? AND revoked = FALSE AND expires_at > NOW() AND ip_address = ? AND user_agent_hash = ?`
3. **Token rotation:** при каждом refresh:
   - Генерируется новый токен с `parent_token_id = old_token_id`
   - Старый токен помечается как `revoked = TRUE`
   - Это создаёт цепочку ротации: `token_3 -> token_2 -> token_1 -> NULL`
4. Очистка: удаление revoked токенов старше N дней (cron-задача)

**Использование `parent_token_id`:**

- Отслеживает историю ротации refresh токенов
- Позволяет найти все токены одной сессии: `SELECT * FROM refresh_tokens WHERE user_id = ? ORDER BY created_at DESC`
- Корневой токен (от login) имеет `parent_token_id = NULL`
- Последний активный токен — это "голова" цепочки ротации

**Где создаются refresh_tokens:**

- `/v1/auth/login` — при успешной авторизации
- `/v1/auth/verify` — при подтверждении email (создаётся **новый** refresh_token, даже если пользователь уже был залогинен)
- `/v1/auth/refresh` — при обновлении токенов (старый инвалидируется, создаётся новый)

**Индексы:**

- `idx_refresh_token` (token) — для быстрого поиска (по хэшу)
- `idx_refresh_user_id` (user_id) — для поиска активных токенов
- `idx_refresh_expires_at` (expires_at) — для очистки истекших токенов
- `idx_refresh_ip` (ip_address) — для аудита по IP
- `idx_refresh_user_agent_hash` (user_agent_hash) — для аудита по типам устройств

---

### 3.4 Таблица reset_password_tokens

| Поле       | Тип          | Описание                               |
| ---------- | ------------ | -------------------------------------- |
| id         | UUID         | PRIMARY KEY                            |
| user_id    | UUID         | REFERENCES users(id) ON DELETE CASCADE |
| token      | VARCHAR(255) | UNIQUE, NOT NULL (UUIDv7)              |
| expires_at | TIMESTAMP    | NOT NULL (1 час от создания)           |
| created_at | TIMESTAMP    | DEFAULT NOW()                          |
| used_at    | TIMESTAMP    | NULL                                   |

**Описание полей:**

- `id` — уникальный идентификатор токена (UUIDv7)
- `user_id` — референс на пользователя, каскадное удаление
- `token` — UUIDv7 для сброса пароля, одноразовый
- `expires_at` — время истечения токена (1 час от создания)
- `created_at` — время генерации токена
- `used_at` — время использования токена (после reset, NULL если не использован)

**Индексы:**

- `idx_reset_token` (token) — для быстрого поиска
- `idx_reset_user_id` (user_id) — для поиска активных токенов
- `idx_reset_expires_at` (expires_at) — для очистки истекших токенов

### 3.5 Таблица login_attempts

| Поле            | Тип          | Описание                                                                    |
| --------------- | ------------ | --------------------------------------------------------------------------- |
| id              | UUID         | PRIMARY KEY                                                                 |
| email           | VARCHAR(255) | NOT NULL                                                                    |
| ip_address      | VARCHAR(45)  | NOT NULL                                                                    |
| user_agent_hash | VARCHAR(64)  | SHA-256 хэш от базовой информации user_agent (для аудита без идентификации) |
| success         | BOOLEAN      | NOT NULL                                                                    |
| failed_at       | TIMESTAMP    | DEFAULT NOW()                                                               |

**Описание полей:**

- `id` — уникальный идентификатор записи (UUIDv7)
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
- `user_agent_hash` хранится для аудита без возможности восстановления полного User-Agent (GDPR compliance)

---

### 3.6 Ключи Redis (rate limiting, session management)

| Ключ                                  | Тип    | Описание                                                                    | TTL        |
| ------------------------------------- | ------ | --------------------------------------------------------------------------- | ---------- |
| `rate_limit:login:{ip}`               | String | Счетчик логинов по IP                                                       | 10 мин     |
| `rate_limit:login:block:{email}`      | String | Блокировка аккаунта после неудач                                            | 15-24 часа |
| `rate_limit:register:{ip}`            | String | Счетчик регистраций по IP                                                   | 1 час      |
| `rate_limit:verify:{ip}`              | String | Счетчик верификаций по IP                                                   | 1 час      |
| `rate_limit:forgot-password:{ip}`     | String | Счетчик запросов сброса по IP                                               | 1 час      |
| `rate_limit:reset-password:{ip}`      | String | Счетчик сбросов по IP                                                       | 1 час      |
| `rate_limit:refresh:{ip}`             | String | Счетчик refresh запросов по IP                                              | 10 мин     |
| `rate_limit:change-password:{ip}`     | String | Счетчик смен паролей по IP                                                  | 1 час      |
| `session:{refresh_token_hash}`        | Hash   | Информация о сессии (user_id, created_at, jti, ip_address, user_agent_hash) | 7 дней     |
| `revoked_tokens:{refresh_token_hash}` | String | Флаг инвалидации токена                                                     | 7 дней     |
| `revoked_access_jti:{jti}`            | String | Флаг инвалидации access token по jti                                        | 30 минут   |
| `lock:register:{email}`               | String | Блокировка после неудачной регистрации                                      | 1 час      |
| `lock:forgot-password:{email}`        | String | Блокировка после неудачного сброса                                          | 1 час      |
| `user_agent:hash:{hash}`              | String | Метаинформация по хэшу User-Agent (browser/os/device)                       | 30 дней    |

**Redis Keys:**

- `session:{refresh_token_hash}` — информация о сессии (user_id, created_at, jti, ip_address, user_agent_hash)
- `revoked_tokens:{refresh_token_hash}` — флаг инвалидации токена
- `revoked_access_jti:{jti}` — флаг инвалидации access token по jti

**Redis Commands used:**

- `INCR` / `EXPIRE` — счетчики rate limiting
- `HSET` / `HGET` — хранение session data
- `SET` / `GET` / `GETSET` — флаги revoked tokens и access jti
- `DEL` — удаление истекших записей

**Revocation Flow:**

1. При logout: `SET revoked_tokens:{refresh_token_hash} 1 EX 604800` + обновление revoked в БД
2. При истечении access token: проверка `GET revoked_access_jti:{jti}`
3. При refresh:
   - Валидация старого refresh token (включая проверку ip_address и user_agent_hash)
   - Инвалидация старого refresh token (revoked в БД + Redis blacklist)
   - Создание нового refresh token с новым jti и сохранением parent_token_id

---

## 4. Конечные точки API

### 4.1 POST /v1/auth/register

**Описание:** Регистрация нового пользователя

**Нормализация email:**
- Пробелы по краям строки удаляются
- Строка приводится к нижнему регистру
- Другой нормализации нет: точки (`.`) и часть после символа `+` сохраняются
- Email хранится в нормализованном виде в базе данных

**Пример:**
- Ввод: `"  Ivan.Ivanov+test@Example.COM  "`
- Нормализованный email: `"ivan.ivanov+test@example.com"`

**Validation Rules:**

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

- `400 VALIDATION_ERROR`: Ошибка валидации данных
- `409 EMAIL_ALREADY_REGISTERED`: Email уже зарегистрирован
- `429 RATE_LIMITED`: Превышен лимит запросов

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

**Правила валидации полей:**
- `REQUIRED` — поле отсутствует, равно null или пустое
- `INVALID_FORMAT` — значение не соответствует формату
- `TOO_SHORT` — длина меньше допустимой
- `TOO_LONG` — длина больше допустимой

---

### 4.2 POST /v1/auth/verify

**Описание:** Подтверждение email с одноразовым токеном

**Request Body:**

```json
{
  "token": "abc123..."
}
```

**Example Request:**

```bash
curl -X POST https://api.mystore.com/v1/auth/verify \
  -H "Content-Type: application/json" \
  -d '{
    "token": "abc123xyz"
  }'
```

**Success Response (200):**

```json
{
  "access_token": "eyJhbG...",
  "refresh_token": "eyJhbG..."
}
```

}`

**Error Responses:**

- `400 CONFIRMATION_TOKEN_INVALID`: Неверный или истекший токен
- `409 EMAIL_ALREADY_CONFIRMED`: Email уже подтверждён

**Response 400 Bad Request (CONFIRMATION_TOKEN_INVALID):**

```json
{
  "error": {
    "code": "CONFIRMATION_TOKEN_INVALID",
    "message": "Неверный или истекший токен подтверждения",
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

**Поведение клиента при 409:**
- Фронтенд должен показать сообщение об успехе (email уже подтверждён)
- Фронтенд должен предложить пользователю войти в систему (форма входа или кнопка "Войти")

---

**Пояснение: Почему возвращается два токена?**

После успешной верификации email возвращаются **оба токена** (`access_token` и `refresh_token`) по следующим причинам:

1. **Непрерывность сессии** — пользователь, который только что подтвердил email, должен продолжать работать без повторного логина. Если возвращать только `access_token`, то через 30 минут (срок его жизни) пользователю придётся логиниться заново, что создаёт плохой UX.

2. **Единообразие flow'ев** — верификация email — это завершение процесса регистрации, а не отдельный endpoint. Логически она должна выдавать те же токены, что и `/v1/auth/login`, чтобы клиент мог работать с API без дополнительных условий.

3. **Безопасность refresh token** — `refresh_token` передаётся как **HTTP-only cookie** (как при login), что защищает его от XSS-атак. Возвращение только `access_token` в response body было бы неполноценным решением.

4. **Token rotation** — при каждом `/v1/auth/verify` генерируется **новый refresh token**, старый (если был) инвалидируется. Это повышает безопасность: даже если токен скомпрометирован, его можно быстро отозвать.

5. **Поддержка off-session verify** — пользователь может подтвердить email в другом браузере или устройстве (через ссылку из email). В этом случае у него может не быть активной сессии, поэтому необходима возможность получить свежие токены.

**Алгоритм работы:**

1. Верифицируется `email_verification_token`
2. Обновляется `users.is_email_verified = true`, `users.email_verified_at = NOW()`
3. Генерируется новый `access_token` (30 мин) и `refresh_token` (7 дней)
4. `refresh_token` сохраняется в БД (хэш) и устанавливается как HTTP-only cookie
5. `email_verification_token` помечается как использованный (удаляется)
6. Возвращаются оба токена в response body (для double-submit cookie pattern или клиентских целей)

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

**Example Request:**

```bash
curl -X POST https://api.mystore.com/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "SecurePass123"
  }'
```

**Success Response (200):**

```json
{
  "access_token": "eyJhbG..."
}
```

**Error Responses:**

- `401 INVALID_CREDENTIALS`: Неверные учётные данные
- `403 EMAIL_NOT_CONFIRMED`: Email не подтверждён
- `429 RATE_LIMITED`: Превышен лимит запросов

**Response 401 Unauthorized (INVALID_CREDENTIALS):**

```json
{
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "Неверный email или пароль"
  }
}
```

**Response 403 Forbidden (EMAIL_NOT_CONFIRMED):**

```json
{
  "error": {
    "code": "EMAIL_NOT_CONFIRMED",
    "message": "Пожалуйста, подтвердите email перед входом"
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

---

### 4.4 POST /v1/auth/refresh

**Описание:** Получение нового access token с rotation refresh token

**Headers:**

```
Authorization: Bearer eyJhbG...   // Access token для идентификации сессии
```

**Refresh Token Location:** HTTP-only cookie (`refreshToken`)

**Поведение:**

- Refresh token **не передается в Authorization header** — он автоматически отправляется браузером как HTTP-only cookie
- Access token передается в `Authorization: Bearer <token>` для идентификации сессии
- При успешном refresh генерируется **новый refresh token** и устанавливается как cookie (старый инвалидируется)

**Example Request:**

```bash
curl -X POST https://api.mystore.com/v1/auth/refresh \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Success Response (200):**

```json
{
  "access_token": "eyJhbG...",
  "token_type": "bearer",
  "expires_in": 1800
}
```

**Примечание:** `refresh_token` не возвращается в response body, так как он автоматически устанавливается как HTTP-only cookie. Это соответствует pattern double-submit cookie: access_token передаётся в Authorization header для идентификации сессии, refresh_token — в cookie для аутентификации. При `/verify` оба токена возвращаются в response body для сохранения непрерывности сессии (см. раздел 4.2).

**Error Responses:**

- `401 REFRESH_TOKEN_INVALID`: Неверный или истекший токен обновления
- `429 RATE_LIMITED`: Превышен лимит запросов

**Response 401 Unauthorized (REFRESH_TOKEN_INVALID):**

```json
{
  "error": {
    "code": "REFRESH_TOKEN_INVALID",
    "message": "Неверный или истекший токен обновления"
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

**Поведение клиента при 401:**
- Все причины отказа (истёк токен, неверная подпись, пользователь не найден) дают одинаковый ответ для клиента
- Клиент должен очистить access token из памяти
- Клиент должен перенаправить пользователя на форму входа
- Конкретная причина 401 фиксируется в логах сервера

---

### 4.5 POST /v1/auth/logout

**Описание:** Выход из системы (инвалидация refresh token текущей сессии)

> **Важно:** Logout завершает **только текущую сессию**. В других браузерах или устройствах пользователь остаётся в системе с действующими токенами.

**Headers:**

```
Authorization: Bearer eyJhbG...
```

**Example Request:**

```bash
curl -X POST https://api.mystore.com/v1/auth/logout \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Success Response (200):**

```json
{
  "message": "Вы успешно вышли из системы"
}
```

**Error Responses:**

- `401 ACCESS_TOKEN_INVALID`: Неверный или истекший токен доступа

```json
{
  "error": {
    "code": "ACCESS_TOKEN_INVALID",
    "message": "Неверный или истекший токен доступа"
  }
}
```

**Как работает logout:**
1. Инвалидирует текущий refresh token (ставит `revoked = TRUE`)
2. Добавляет `jti` в Redis blacklist с TTL = оставшееся время жизни
3. Очищает cookie (через `Set-Cookie: refresh_token=; Max-Age=0`)
4. **Не инвалидирует другие активные сессии** — другие refresh tokens остаются валидными

**Access token после logout:**
- Access token **не инвалидируется** и остаётся действительным до `exp` (30 минут)
- Клиент должен удалить access token из памяти

---

### 4.6 GET /v1/auth/me

**Описание:** Получение данных текущего пользователя

**Headers:**

```
Authorization: Bearer eyJhbG...
```

**Example Request:**

```bash
curl https://api.mystore.com/v1/auth/me \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
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

- `401`: Неверный или истекший токен доступа

**Response 401 Unauthorized (ACCESS_TOKEN_INVALID):**

```json
{
  "error": {
    "code": "ACCESS_TOKEN_INVALID",
    "message": "Неверный или истекший токен доступа"
  }
}
```

**Поведение клиента при 401:**
- Все причины отказа (истёк токен, неверная подпись, пользователь не найден) дают одинаковый ответ для клиента
- Клиент должен очистить access token из памяти
- Клиент должен перенаправить пользователя на форму входа
- Конкретная причина 401 фиксируется в логах сервера

---

### 4.7 POST /v1/auth/resend-verification

**Описание:** Повторная отправка email с подтверждением

**Request Body:**

```json
{
  "email": "user@example.com"
}
```

**Example Request:**

```bash
curl -X POST https://api.mystore.com/v1/auth/resend-verification \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com"
  }'
```

**Success Response (200):**

```json
{
  "message": "Письмо с подтверждением отправлено. Пожалуйста, проверьте почту."
}
```

**Error Responses:**

- `409 EMAIL_ALREADY_VERIFIED`: Email уже подтверждён
- `429 RATE_LIMITED`: Превышен лимит запросов

**Поведение:**

| Состояние учётной записи | Действие | Ответ |
| ----------------------- | -------- | ----- |
| Email уже подтверждён | Письмо не отправляется | `200 OK` с сообщением `{"message": "Email уже подтверждён"}` |
| Существует pending-учётная запись, но за последний час отправлено 3 письма | Письмо не отправляется | `429 RATE_LIMITED` |
| Существует pending-учётная запись, меньше 3 писем за час | Прежние токены аннулируются, выпускается новый | `200 OK` с сообщением `{"message": "Письмо с подтверждением отправлено. Пожалуйста, проверьте почту."}` |
| Учётной записи с таким email не существует | Письмо не отправляется (для безопасности) | `200 OK` с сообщением `{"message": "Если учётная запись существует, письмо отправлено"}` |

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
    "message": "Превышен лимит запросов. Попробуйте позже.",
    "details": { "retry_after": 3600 }
  }
}
```

**Response 200 OK (email already verified):**

```json
{
  "message": "Email уже подтверждён"
}
```

**Response 200 OK (pending account):**

```json
{
  "message": "Письмо с подтверждением отправлено. Пожалуйста, проверьте почту."
}
```

**Response 200 OK (account not found):**

```json
{
  "message": "Если учётная запись существует, письмо отправлено"
}
```

---

### 4.8 POST /v1/auth/forgot-password

**Описание:** Запрос на сброс пароля

**Request Body:**

```json
{
  "email": "user@example.com"
}
```

**Example Request:**

```bash
curl -X POST https://api.mystore.com/v1/auth/forgot-password \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com"
}
```

**Success Response (200):**

```json
{
  "message": "Инструкции по сбросу пароля отправлены на email."
}
```

**Behavior:** Не раскрывает существование email (для безопасности). Возвращает `200 OK` для любых запросов (существующих и несуществующих email).

**Error Responses:**

- `429 RATE_LIMITED`: Превышен лимит запросов

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

**Example Request:**

```bash
curl -X POST https://api.mystore.com/v1/auth/reset-password \
  -H "Content-Type: application/json" \
  -d '{
    "token": "reset-token-from-email",
    "password": "NewSecurePass123"
  }'
```

**Success Response (200):**

```json
{
  "message": "Пароль успешно сброшен."
}
```

**Error Responses:**

- `400 INVALID_TOKEN`: Неверный или истекший токен
- `404 TOKEN_NOT_FOUND`: Токен не найден
- `429 RATE_LIMITED`: Превышен лимит запросов


**Response 400 Bad Request (INVALID_TOKEN):**

```json
{
  "error": {
    "code": "INVALID_TOKEN",
    "message": "Неверный токен сброса пароля"
  }
}
```

**Response 400 Bad Request (EXPIRED_TOKEN):**

```json
{
  "error": {
    "code": "EXPIRED_TOKEN",
    "message": "Токен сброса пароля истёк"
  }
}
```

**Response 404 Not Found (TOKEN_NOT_FOUND):**

```json
{
  "error": {
    "code": "TOKEN_NOT_FOUND",
    "message": "Токен сброса пароля не найден"
  }
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

**Example Request:**

```bash
curl -X POST https://api.mystore.com/v1/auth/unsubscribe \
  -H "Content-Type: application/json" \
  -d '{
    "token": "unsubscribe-token-from-email"
  }'
```

**Success Response (200):**

```json
{
  "message": "Успешная отписка от рассылки"
}
```

**Error Responses:**

- `400 INVALID_TOKEN`: Неверный или истекший токен
- `404 USER_NOT_FOUND`: Пользователь не найден

**Response 400 Bad Request (INVALID_TOKEN):**

```json
{
  "error": {
    "code": "INVALID_TOKEN",
    "message": "Неверный или истекший токен отписки"
  }
}
```

**Response 404 Not Found (USER_NOT_FOUND):**

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "Пользователь не найден"
  }
}
```

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

**Example Request:**

```bash
curl -X PUT https://api.mystore.com/v1/auth/change-password \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "current_password": "OldPass123",
    "new_password": "NewSecurePass456"
  }'
```

**Success Response (200):**

```json
{
  "message": "Пароль успешно изменен."
}
```

**Error Responses:**

- `400 INVALID_CREDENTIALS`: Текущий пароль неверный
- `400 WEAK_PASSWORD`: Новый пароль не соответствует требованиям
- `401 ACCESS_TOKEN_INVALID`: Неверный или истекший токен доступа
- `429 RATE_LIMITED`: Превышен лимит запросов

**Response 400 Bad Request (INVALID_CREDENTIALS):**

```json
{
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "Текущий пароль неверный"
  }
}
```

**Response 400 Bad Request (WEAK_PASSWORD):**

```json
{
  "error": {
    "code": "WEAK_PASSWORD",
    "message": "Новый пароль не соответствует требованиям (минимум 8 символов)"
  }
}
```

**Response 401 Unauthorized (ACCESS_TOKEN_INVALID):**

```json
{
  "error": {
    "code": "ACCESS_TOKEN_INVALID",
    "message": "Неверный или истекший токен доступа"
  }
}

---

### 4.12 DELETE /v1/auth/me

**Описание:** Удаление аккаунта текущего пользователя (GDPR compliance)

**Headers:**

```
Authorization: Bearer eyJhbG...
```

**Example Request:**

```bash
curl -X DELETE https://api.mystore.com/v1/auth/me \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Success Response (200):**

```json
{
  "message": "Аккаунт удалён навсегда."
}
```

**Error Responses:**

- `401 ACCESS_TOKEN_INVALID`: Неверный или истекший токен доступа
- `403 EMAIL_NOT_VERIFIED`: Email не подтверждён

**Response 401 Unauthorized (ACCESS_TOKEN_INVALID):**

```json
{
  "error": {
    "code": "ACCESS_TOKEN_INVALID",
    "message": "Неверный или истекший токен доступа"
  }
}
```

**Response 403 Forbidden (EMAIL_NOT_VERIFIED):**

```json
{
  "error": {
    "code": "EMAIL_NOT_VERIFIED",
    "message": "Пожалуйста, подтвердите email перед удалением аккаунта"
  }
}
```

---

## 5. Реализация безопасности

### 5.1 Ограничение частоты запросов

**Per-endpoint limits (для всех запросов к эндпоинту):**

| Endpoint           | Лимит | Период | Тайм-аут блокировки | Identifier для лимита | Логика сброса            |
| ------------------ | ----- | ------ | ------------------- | --------------------- | ------------------------ |
| `/login`           | 5     | 10 мин | 15 мин              | IP + email            | При успешном входе       |
| `/register`        | 3     | 1 час  | 1 час               | IP + email            | При успешной регистрации |
| `/verify`          | 10    | 1 час  | 30 мин              | IP                    | При успешной верификации |
| `/forgot-password` | 3     | 1 час  | 1 час               | IP + email            | При успешном сбросе      |
| `/reset-password`  | 5     | 1 час  | 1 час               | IP + email            | При успешном сбросе      |
| `/refresh`         | 30    | 10 мин | —                   | IP                    | Без блокировки           |
| `/change-password` | 5     | 1 час  | 1 час               | IP + email            | При успешной смене       |
| `/logout`          | 30    | 10 мин | —                   | IP                    | Без блокировки           |

**Algorithm:**

1. Для каждого запроса проверяется лимит по соответствующему identifier (IP для общих эндпоинтов, IP + email для аутентификационных)
2. Если лимит превышен → ответ 429 RATE_LIMITED с телом `{ "error": "RATE_LIMITED", "message": "Превышен лимит запросов" }`
3. Для аутентификационных эндпоинтов (`/login`, `/register`, `/forgot-password`, `/reset-password`, `/change-password`):
   - Неудачные попытки (неверные credentials, несуществующий email и т.д.) увеличивают счётчик блокировки
   - После 5 неудачных попыток включается блокировка по IP:
     - 5 неудачных → блокировка 15 минут
     - 10 неудачных → блокировка 24 часа
   - При успешной операции счётчик сбрасывается
4. Для `/verify`, `/refresh`, `/logout` блокировка не применяется, только общий лимит по IP

**Redis Implementation:**

- **Key pattern для лимита:** `rate_limit:{endpoint}:{identifier}:{window}`
- **Key pattern для блокировки:** `lock:{endpoint}:{identifier}`
- **Identifier для лимита:**
  - `/login`, `/register`, `/forgot-password`, `/reset-password`, `/change-password` — `IP:email`
  - `/verify`, `/refresh`, `/logout` — `IP`
- **TTL:** Period + 5 минут (автоматическая очистка)
- **Счётчик:** INCR/EXPIRE для каждого identifier
- **Block key:** `lock:{endpoint}:{identifier}` с TTL = duration блокировки
- **Структура данных:** Для каждого identifier используется отдельный счётчик с автоматическим сбросом по истечении TTL

**Примеры ключей:**
- `rate_limit:login:192.168.1.1:user@example.com:1727100000` — лимит для login от конкретного IP и email
- `lock:login:192.168.1.1:1727148000` — блокировка IP на 15 минут после 5 неудачных попыток

**Timing Attack Protection:**

- Использовать константное сравнение для всех критичных проверок:
  - `crypto.timingSafeEqual()` для сравнения хэшей паролей
  - `crypto.timingSafeEqual()` для сравнения токенов
  - `hmac.compare_digest()` (Python) / `ConstantTimeCompare()` (Go)
- Избегать раннего выхода из функций сравнения

### 5.2 Безопасность паролей

- **Algorithm:** Argon2id (memory: 64MB, iterations: 3, parallelism: 4)
- **Minimum length:** 8 символов (рекомендуется 12+)
- **No password policy** (не требуем специальные символы для удобства)
- **Timing-safe comparison:** Обязательное константное сравнение хэшей

### 5.3 Безопасность токенов

**Refresh Token Revocation:**

- При logout токен помечается как revoked в БД (`UPDATE refresh_tokens SET revoked = TRUE, revoked_at = NOW() WHERE jti = ? AND user_id = ?`)
- Одновременно `jti` токена добавляется в Redis blacklist с TTL = оставшееся время жизни
- **Токен НЕ удаляется из БД** (для аудита и предотвращения reuse)
- Очистка revoked токенов: периодический cron-джоб удаления токенов, revoked = TRUE и expired более N дней назад

**Refresh Token Structure:**

- `revoked` (BOOLEAN, default FALSE) — флаг инвалидации
- `revoked_at` (TIMESTAMP, NULL) — время инвалидации
- `parent_token_id` (UUID) — ссылка на родительский токен при rotation

**Token Binding:**

- При проверке refresh token проверяются:
  - `jti` валидность
  - `revoked = FALSE`
  - `expires_at > NOW()`
  - **`ip_address` совпадает с IP запроса**
  - **`user_agent_hash` совпадает с хэшем User-Agent запроса**

**Access Token:**

- RS256, 30 минут, хранение в памяти
- Каждый токен имеет уникальный `jti` (UUIDv7)
- `jti` хранится в Redis с TTL=30 минут для отслеживания
- Авто-обновление при истечении (только safe methods)
- При logout: `revoked_access_jti:{jti}` добавляется в Redis с TTL=30 минут

**Access Token НЕ может быть отозван:**

- Access token — это **stateless** токен с фиксированным сроком жизни (30 минут)
- При logout или смене пароля access token **не инвалидируется** напрямую
- `revoked_access_jti:{jti}` в Redis используется только для защиты от повторного использования **точно этого же токена** в рамках одного запроса (защита от replay), но токен всё равно будет валиден до истечения `exp`
- **После истечения `exp` (30 минут)** токен автоматически становится недействительным
- **Для полной инвалидации** необходимо использовать refresh token flow:
  1. Вызвать logout с refresh token (инвалидирует refresh token)
  2. Пользователь должен выполнить повторный login
- **Исключение:** При критических событиях (утечка ключа, компрометация) можно использовать `revoked_access_jti` для конкретных `jti`, но это работает только до истечения `exp`

**Refresh Token Rotation:**

- RS256, 7 дней, HTTP-only cookie + Redis blacklist
- **При каждом refresh:**
  1. Старый refresh token инвалидируется (revoked в БД + Redis blacklist)
  2. Генерируется новый refresh token с новым jti
  3. Новый токен сохраняется с `parent_token_id`, указывающим на старый токен
  4. Новый токен устанавливается как HTTP-only cookie
- Single-use refresh: токен инвалидируется после использования (реализован через rotation)

**Timing Attack Protection:**

- Использовать константное сравнение для всех критичных проверок:
  - `crypto.timingSafeEqual()` для сравнения хэшей паролей
  - `crypto.timingSafeEqual()` для сравнения токенов
  - `hmac.compare_digest()` (Python) / `ConstantTimeCompare()` (Go)
- Избегать раннего выхода из функций сравнения

### 5.4 Безопасность email

**Email Verification Token:**

- Срок жизни: 24 часа
- Одноразовость: после использования токен удаляется из таблицы `email_verification_tokens`
- Проверка дублирования: если `is_email_verified = true`, возвращается `409 Conflict`
- Токен генерируется как URL-safe base64 UUID (без хэширования, так как токен короткий и имеет высокую энтропию)

**Security Considerations for /v1/auth/verify:**

- При verify создаётся **новый refresh_token** (даже если пользователь уже был залогинен) — это обеспечивает непрерывность сессии
- Если verify выполняется в другом браузере/устройстве (не там, где был login), создаётся **новая сессия** с новым refresh_token
- **Token binding (IP + user_agent_hash) НЕ применяется к /verify** — это специально, чтобы пользователь мог подтвердить email из любого устройства (off-session verify через ссылку в email)
- **Token binding применим ТОЛЬКО к /refresh и /logout** — эти endpoints проверяют IP и user_agent_hash для защиты от кражи токенов
- Rate limiting: 10 запросов в час, с 30-минутной блокировкой при превышении

**Почему возвращается refresh_token после verify?**

1. **User Experience (UX):** Пользователь, который только что подтвердил email, должен продолжать работать без повторного логина
2. **Consistency:** Verify — это завершение регистрации, а не отдельный endpoint. Логически он должен выдавать те же токены, что и login
3. **Security:** Refresh token в HTTP-only cookie защищён от XSS. Возвращение только access_token в response body было бы неполноценным решением
4. **Token Rotation:** При verify генерируется новый refresh_token, старый (если был) инвалидируется — это повышает безопасность

- **Verification tokens:** URL-safe base64 UUID, TTL = 24 часа (см. 3.2 email_verification_tokens)
- **Reset tokens:** UUIDv7, TTL = 1 час (см. 3.4 reset_password_tokens)
- **Forgot-password tokens:** UUIDv7, TTL = 1 час
- **Unsubscribe tokens:** UUIDv7, хранятся в users.unsubscribe_token, не истекают
- **Tokens одноразовые:** После использования помечаются как использованные в БД (used_at)
- **No email enumeration:** Ответы не раскрывают наличие email (см. 5.4.1)
- **Timing-safe comparison:** Обязательное константное сравнение токенов при verify/reset

---

## 5.4.1 Защита от перечисления аккаунтов

**Цель:** Предотвратить возможность для злоумышленника определить, зарегистрирован ли определённый email в системе.

### Принципы защиты:

1. **Единый формат ответа 401** для всех случаев неудачной аутентификации:
   - Несуществующий email
   - Неверный пароль для существующего аккаунта
   - Не подтверждённый email

2. **Константное сравнение пароля** — даже для несуществующих email сервер должен выполнить операцию хэширования и сравнения, чтобы время ответа не раскрывало наличие аккаунта.

3. **Фиктивная проверка пароля** — если email не найден, сервер генерирует фиктивный хэш (например, от случайной строки) и выполняет константное сравнение, чтобы время ответа было одинаковым.

### Реализация в `/v1/auth/login`:

```javascript
// Псевдокод
const user = await db.query('SELECT * FROM users WHERE email = ?', [normalizedEmail]);

if (!user) {
  // Фиктивная проверка пароля для защиты от перечисления
  await argon2.hash(dummyPassword, argon2Options);
  return res.status(401).json({
    error: {
      code: 'INVALID_CREDENTIALS',
      message: 'Неверный email или пароль',
      details: {}
    }
  });
}

const isValid = await argon2.verify(user.password_hash, password);
if (!isValid) {
  return res.status(401).json({
    error: {
      code: 'INVALID_CREDENTIALS',
      message: 'Неверный email или пароль',
      details: {}
    }
  });
}
```

### Реализация в `/v1/auth/verify`:

```javascript
// Псевдокод
const tokenRecord = await db.query(
  'SELECT * FROM email_verification_tokens WHERE token = ? AND used_at IS NULL AND expires_at > NOW()',
  [token]
);

if (!tokenRecord) {
  // Фиктивная проверка для защиты от перечисления
  await db.query('SELECT id FROM users WHERE email = ?', [dummyEmail]);
  return res.status(401).json({
    error: {
      code: 'INVALID_TOKEN',
      message: 'Неверный токен подтверждения',
      details: {}
    }
  });
}
```

### Эндпоинты с защитой от перечисления:

| Endpoint | Поведение |
| -------- | ---------- |
| `/login` | 401 для несуществующего email и неверного пароля — одинаковый ответ |
| `/verify` | 401 для неверного токена — фиктивная проверка email |
| `/resend-verification` | 204 для любого email (даже несуществующего) — не раскрывает наличие аккаунта |
| `/forgot-password` | 204 для любого email — не раскрывает наличие аккаунта |

### Запрещённые поведения (раскрывают перечисление):

- `404 Not Found` — всегда запрещён
- `409 Email already registered` в `/login`
- `400 Email not found` в `/login`
- Различие во времени ответа для существующих/несуществующих аккаунтов

---

### 5.5 Защита от CSRF

**Для endpoints с HTTP-only cookie (refresh token):**

| Endpoint           | Cookie           | CSRF Protection                                |
| ------------------ | ---------------- | ---------------------------------------------- |
| `/login`           | Нет              | CSRF token + SameSite=None (если cross-origin) |
| `/register`        | Нет              | CSRF token + SameSite=None (если cross-origin) |
| `/refresh`         | HTTP-only cookie | SameSite=Strict + Origin validation            |
| `/logout`          | HTTP-only cookie | SameSite=Strict + Origin validation            |
| `/me`              | Нет              | Origin validation                              |
| `/change-password` | HTTP-only cookie | SameSite=Strict + Origin validation            |

**Обязательные защиты:**

- `SameSite=Strict` для всех cookie (не отправляются при cross-site запросах)
- `Secure` флаг для cookie (только HTTPS)
- `HttpOnly` флаг для refresh token cookie
- **Origin validation** на сервере для всех запросов
- **Referer/Referrer policy** заголовки

**Пример проверки Origin:**

```javascript
const allowedOrigins = ["https://mystore.com", "https://www.mystore.com"];
const origin = req.headers.origin;
if (!allowedOrigins.includes(origin)) {
  return res.status(403).json({ error: "invalid_origin" });
}
```

---

## 5.6 Заголовки безопасности

Дополнительные HTTP заголовки для защиты:

| Заголовок                   | Значение                              | Описание                        |
| --------------------------- | ------------------------------------- | ------------------------------- |
| `Content-Security-Policy`   | `default-src 'self'`                  | Ограничивает источники контента |
| `X-Content-Type-Options`    | `nosniff`                             | Отключает MIME-sniffing         |
| `X-Frame-Options`           | `DENY`                                | Защита от clickjacking          |
| `X-XSS-Protection`          | `1; mode=block`                       | XSS фильтр (legacy)             |
| `Referrer-Policy`           | `strict-origin-when-cross-origin`     | Контроль referrer               |
| `Permissions-Policy`        | `geolocation=(), microphone=()`       | Ограничение API                 |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | HSTS для HTTPS                  |

---

## 5.7 Модель угроз

| Угроза                         | Митигация                                                                                                        |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| Brute force атака              | Ограничение частоты запросов (Redis, rate limiting), блокировка по IP/email, константное сравнение (timing-safe) |
| Кража токенов                  | HTTP-only cookies, короткоживущие access tokens, rotation refresh tokens, отслеживание инвалидации               |
| Атака через привязку токена    | Привязка IP + User-Agent для refresh tokens, проверка при каждом использовании                                   |
| SQL-инъекция                   | Подготовленные выражения (prepared statements/ORM), параметризованные запросы                                    |
| XSS                            | Content-Security-Policy, очистка пользовательского ввода, заголовки безопасности                                 |
| CSRF                           | SameSite=Strict cookies, проверка Origin/Referer, CSRF-токены для чувствительных операций                        |
| Взлом паролей                  | Argon2id (memory-hard), ограничение частоты запросов, проверка на утечки (опционально)                           |
| Перехват email                 | TLS 1.3+ для SMTP, одноразовые токены, короткий срок действия                                                    |
| Атаки по времени               | Константное сравнение хэшей (timingSafeEqual, ConstantTimeCompare) для всех критичных проверок                   |
| Повторное использование токена | Refresh token инвалидируется после использования, revoked флаг в БД + Redis blacklist                            |
| Перечисление аккаунтов         | Ответы не раскрывают наличие email, константное сравнение (timing-safe) при verify/login                         |
| Перехват сессии                | Короткоживущие access tokens, HTTP-only cookies, проверка Origin                                                 |
| DoS при logout                 | Ограничение частоты запросов (30/10 мин по IP) для logout endpoint                                               |

---

## 5.8 Краткое резюме реализации безопасности

**Ключевые механизмы безопасности:**

| Механизм                         | Описание                          | Реализация                     |
| -------------------------------- | --------------------------------- | ------------------------------ |
| **Ограничение частоты запросов** | Защита от brute force и DoS       | Redis + лимиты на endpoint     |
| **Rotation токенов**             | Одноразовость refresh tokens      | Revoked флаг + Redis blacklist |
| **Привязка токена**              | Привязка к IP + User-Agent        | Проверка при каждом refresh    |
| **Константное сравнение**        | Защита от атак по времени         | crypto.timingSafeEqual()       |
| **Защита от CSRF**               | SameSite=Strict + проверка Origin | HTTP middleware                |
| **Защищённые cookies**           | HttpOnly + Secure флаги           | HTTP-only для refresh token    |
| **Короткоживущие токены**        | Access token 30 минут             | JWT exp claim                  |
| **Аудит логирование**            | Логирование всех операций         | Event schema (раздел 7)        |

---

## 5.9 Соответствие стандартам

- **GDPR:** Возможность удаления аккаунта (см. раздел 4.12 DELETE /v1/auth/me)
- **PII protection:** Email хранится в зашифрованном виде (опционально)
- **Audit logging:** Все операции логируются (см. раздел 7)

---

## 5.10 Чеклист реализации

**Backend:**

- [ ] Константное сравнение для всех критичных проверок (password hash, tokens)
- Проверка Origin/Referer для всех auth endpoints
- Security Headers (CSP, HSTS, X-Frame-Options)
- [ ] SameSite=Strict для всех cookie
- [ ] Secure флаг для cookie (только HTTPS в production)
- [ ] HTTP-only флаг для refresh token cookie
- [ ] Rate limiting для всех endpoints (Redis), включая logout (30/10min по IP)
- [ ] Блокировка аккаунтов после превышения лимита неудач
- [ ] Revocation flow для logout (UPDATE revoked + Redis blacklist + проверка user_id)
- [ ] Проверка revoked флага, ip_address и user_agent_hash при refresh
- [ ] Очистка старых revoked токенов (cron-задача)
- [ ] Генерация unsubscribe_token при регистрации пользователя
- [ ] Endpoint POST /v1/auth/unsubscribe для обработки отписок

---

## 6. Интеграция email

### 6.1 Клиент SMTP

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
    ciphers: 'TLSv1.2',
    minVersion: 'TLSv1.2'
  }
}
```

**Features:**

- Connection pooling (5-10 connections)
- Retry logic (3 attempts, exponential backoff)
- Queue system (BullMQ) для высокой нагрузки
- Bounce detection и handling

### 6.2 Шаблон: Подтверждение регистрации

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

### 6.3 Шаблон: Сброс пароля

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

### 6.4 Очередь email (для высокой нагрузки)

**Technology:** BullMQ (Redis-based queue)

**Queue name:** `emails`

**Job types:**

- `verification` — email verification (подтверждение)
- `password_reset` — password reset (сброс пароля)
- `unsubscribe` — handling unsubscribe requests (обработка отписок)
- `notification` — general notifications (уведомления)

**Worker configuration:**

```javascript
{
  concurrency: 10,        // параллельные воркеры
  attempts: 3,            // повтор при ошибке
  delay: 5000,            // 5 сек задержка между повторами
  backoff: 'exponential'  // экспоненциальная задержка
}
```

### 6.5 Переменные окружения

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

## 7. Логирование

### 7.1 Схема событий

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

### 7.2 Критические события

- `user.registered` — логировать user_id, email, ip_address, user_agent (базовая информация)
- `user.verified` — логировать user_id, email, ip_address
- `user.logged_in` — логировать user_id, email, ip_address, user_agent (базовая информация)
- `user.logged_out` — логировать user_id, email, ip_address
- `auth.failed` — логировать email, ip_address, user_agent (базовая информация)
- `token.autorefresh` — логировать jti, метод запроса и путь (при авто-обновлении access token)
- `auth.token_refreshed` — логировать user_id, email, ip_address, jti (новый токен), jti_old (старый токен), user_agent (базовая информация)

---

## 8. Переменные окружения

| Переменная               | Обязательная | Описание                                                          |
| ------------------------ | ------------ | ----------------------------------------------------------------- |
| `NODE_ENV`               | Да           | development, staging, production                                  |
| `PORT`                   | Да           | Порт сервера (по умолчанию: 3000)                                 |
| `JWT_PRIVATE_KEY`        | Да           | Приватный ключ для RS256 (PEM format, base64-encoded)             |
| `JWT_PUBLIC_KEY`         | Да           | Публичный ключ для верификации RS256 (PEM format, base64-encoded) |
| `DATABASE_URL`           | Да           | PostgreSQL connection string                                      |
| `REDIS_URL`              | Да           | Redis connection string (включая пароль, если есть)               |
| `SMTP_HOST`              | Да           | SMTP сервер                                                       |
| `SMTP_PORT`              | Да           | SMTP порт (587 для TLS, 465 для SSL)                              |
| `SMTP_USER`              | Да           | SMTP пользователь                                                 |
| `SMTP_PASS`              | Да           | SMTP пароль                                                       |
| `SMTP_FROM`              | Да           | From email адрес                                                  |
| `SMTP_TIMEOUT`           | Нет          | Таймаут SMTP соединения (по умолчанию: 30s)                       |
| `SMTP_TLS_MIN_VERSION`   | Нет          | Минимальная версия TLS (по умолчанию: TLSv1.2)                    |
| `APP_URL`                | Да           | URL приложения (для email ссылок)                                 |
| `APP_VERIFY_URL`         | Нет          | URL страницы верификации (по умолчанию: APP_URL/verify)           |
| `APP_RESET_PASSWORD_URL` | Нет          | URL страницы сброса пароля (по умолчанию: APP_URL/reset-password) |
| `APP_UNSUBSCRIBE_URL`    | Нет          | URL страницы отписки (по умолчанию: APP_URL/unsubscribe)          |
| `RATE_LIMIT_WINDOW_MS`   | Нет          | Окно rate limiting в мс (по умолчанию: 60000)                     |
| `RATE_LIMIT_MAX`         | Нет          | Максимальное кол-во запросов (по умолчанию: 100)                  |
| `LOG_LEVEL`              | Нет          | debug, info, warn, error (по умолчанию: info)                     |

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

## 9. Чеклист развёртывания

### База данных

- [ ] Создать таблицы (users, email_verification_tokens, refresh_tokens, login_attempts)
- [ ] Создать индексы
- [ ] Настроить резервное копирование

### Backend

- [ ] Настроить переменные окружения
- [ ] Настроить rate limiting (Redis)
- [ ] Настроить email сервер
- [ ] Настроить логирование
- [ ] Настроить мониторинг (метрики)

### Безопасность

- [ ] SSL/TLS на load balancer
- [ ] HTTP-only cookie для refresh tokens
- [ ] CORS настроен правильно
- [ ] XSS защита (см. раздел 5.6 Заголовки безопасности)
- [ ] SQL injection защита (prepared statements)

---

## 10. Стратегия тестирования

### Юнит-тесты

- Целевое покрытие: **≥85%** для критических модулей (auth, token handling, password hashing)
- Целевое покрытие: **≥70%** для остальных модулей
- Хэширование паролей (argon2id)
- Подпись/проверка JWT (RS256)
- Генерация токенов электронной почты
- Ограничение запросов (rate limiting)
- JWT ID (jti) генерация и уникальность
- Revocation check по jti в Redis

### Интеграционные тесты

- Полный процесс регистрации
- Процесс подтверждения электронной почты
- Процесс входа в систему
- Обновления токена
- Процесс выхода из системы
- Ограничение запросов (rate limiting)
- Auto-refresh access token при истечении (safe methods)
- Revocation по jti (logout и истечение срока)
- Edge case: Refresh token истек в момент запроса refresh
- Edge case: Access token истек в момент refresh (должен вернуть 401)
- Edge case: Использованный refresh token (reuse attack)

### E2E тесты

- Фреймворки: Cypress / Playwright
- Проверка UI (если есть)
- Security Scenarios:
  - XSS: Ввод вредоносного JS в поля name/email/password, проверка экранирования
  - SQLi: Injection в email (e.g. `' OR 1=1 --`)
  - Header injection в User-Agent
  - CSRF: Проверка отсутствия токена в заголовках
  - Token leakage: Проверка, что токены не попадают в console.log и network logs
  - Session fixation: Проверка смены jti при refresh
