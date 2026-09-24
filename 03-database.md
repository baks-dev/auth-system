# 3. Схема базы данных

## 3.1 Таблица users

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

## 3.2 Таблица email_verification_tokens

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

## 3.3 Таблица refresh_tokens

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

## 3.4 Таблица reset_password_tokens

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

## 3.5 Таблица login_attempts

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

## 3.6 Ключи Redis (rate limiting, session management)

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
