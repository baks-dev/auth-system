# 5. Реализация безопасности

## 5.1 Ограничение частоты запросов

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

## 5.2 Безопасность паролей

- **Algorithm:** Argon2id (memory: 64MB, iterations: 3, parallelism: 4)
- **Minimum length:** 8 символов (рекомендуется 12+)
- **No password policy** (не требуем специальные символы для удобства)
- **Timing-safe comparison:** Обязательное константное сравнение хэшей

## 5.3 Безопасность токенов

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

## 5.4 Безопасность email

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

## 5.5 Защита от CSRF

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