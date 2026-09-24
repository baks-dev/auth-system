# 5. Реализация безопасности

## 5.1 Ограничение частоты запросов

**Лимиты запросов по endpoints:**

| Endpoint           | Лимит | Период | Identifier для лимита | Логика сброса            |
| ------------------ | ----- | ------ | --------------------- | ------------------------ |
| `/login`           | 5     | 10 мин | IP + email            | При успешном входе       |
| `/register`        | 3     | 1 час  | IP + email            | При успешной регистрации |
| `/verify`          | 10    | 1 час  | IP                    | При успешной верификации |
| `/forgot-password` | 3     | 1 час  | IP + email            | При успешном сбросе      |
| `/reset-password`  | 5     | 1 час  | IP + email            | При успешном сбросе      |
| `/refresh`         | 30    | 10 мин | IP                    | Без блокировки           |
| `/change-password` | 5     | 1 час  | IP + email            | При успешной смене       |
| `/logout`          | 30    | 10 мин | IP                    | Без блокировки           |

**Блокировка после неудачных попыток:**

| Endpoint                        | 5 неудачных | 10 неудачных | Identifier |
| ------------------------------- | ----------- | ------------ | ---------- |
| `/login`, `/register`           | 15 мин      | 24 часа      | IP         |
| `/forgot-password`, `/reset-password` | 15 мин      | 24 часа      | IP         |
| `/change-password`              | 15 мин      | 24 часа      | IP         |
| `/verify`                       | —           | —            | —          |
| `/refresh`, `/logout`           | —           | —            | —          |

**Примечания:**
- Блокировка применяется только к аутентификационным endpoints (`/login`, `/register`, `/forgot-password`, `/reset-password`, `/change-password`)
- Неудачные попытки на `/verify` не приводят к блокировке (только к общему лимиту)
- При успешной операции счётчик неудачных попыток сбрасывается

**Алгоритм работы лимитов и блокировок:**

1. При каждом запросе проверяется лимит по соответствующему identifier (IP для общих эндпоинтов, IP + email для аутентификационных)
2. Если лимит превышен → возвращается ответ 429 RATE_LIMITED с заголовком `Retry-After`
3. Для аутентификационных endpoints (`/login`, `/register`, `/forgot-password`, `/reset-password`, `/change-password`):
   - Неудачные попытки (неверные credentials, несуществующий email и т.д.) увеличивают счётчик блокировки
   - После 5 неудачных попыток активируется блокировка по IP на 15 минут
   - После 10 неудачных попыток блокировка увеличивается до 24 часов
   - При успешной операции счётчик неудачных попыток сбрасывается
4. Для `/verify`, `/refresh`, `/logout` применяется только общий лимит по IP, блокировка не используется

**Реализация в Redis:**

| Параметр | Формат |
|----------|--------|
| Key для лимита | `rate_limit:{endpoint}:{identifier}:{window}` |
| Key для блокировки | `lock:{endpoint}:{identifier}` |
| TTL | Period + 5 минут (автоматическая очистка) |
| Счётчик | INCR/EXPIRE для каждого identifier |
| Block key | `lock:{endpoint}:{identifier}` с TTL = duration блокировки |

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

**Алгоритм хеширования:** Argon2id
- Memory cost: 64MB
- Time cost: 3 итерации
- Parallelism: 4 параллельных потока
- Type: Hybrid (Argon2id)

**Политика паролей:**
- Минимальная длина: 8 символов
- Рекомендуемая длина: 12+ символов
- Без принудительных требований к специальным символам (удобство пользователя)

**Защита от timing attack:**
- Константное сравнение хэшей паролей с помощью `crypto.timingSafeEqual()`
- Константное сравнение токенов
- Избегание раннего выхода из функций сравнения

**Хранение:**
- Пароли хранятся только в виде хэша (никакого шифрования)
- Соль включена в параметры Argon2id (auto-generated)

## 5.3 Безопасность токенов

**Refresh Token Revocation:**
При logout токен помечается как revoked в БД (`UPDATE refresh_tokens SET revoked = TRUE, revoked_at = NOW() WHERE jti = ? AND user_id = ?`) и одновременно `jti` токена добавляется в Redis blacklist с TTL = оставшееся время жизни. Токен НЕ удаляется из БД (для аудита и предотвращения reuse). Очистка revoked токенов: периодический cron-джоб удаления токенов, у которых revoked = TRUE и срок истёк более N дней назад.

**Refresh Token Structure:**

| Поле | Тип | Описание |
|------|-----|----------|
| `revoked` | BOOLEAN | Флаг инвалидации (default FALSE) |
| `revoked_at` | TIMESTAMP | Время инвалидации (NULL, если активен) |
| `parent_token_id` | UUID | Ссылка на родительский токен при rotation |

**Token Binding:**
При проверке refresh token проверяется привязка к клиенту: `jti` валидность, `revoked = FALSE`, `expires_at > NOW()`, а также совпадение `ip_address` с IP запроса и `user_agent_hash` с хэшем User-Agent запроса. Это защищает от кражи токена — даже если токен украден, он не сработает с другого IP или устройства.

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

## 5.4 Безопасность email

**Email Verification Token:**
- Срок жизни: 24 часа
- Одноразовость: после использования токен удаляется из таблицы `email_verification_tokens`
- Проверка дублирования: если `is_email_verified = true`, возвращается `409 Conflict`
- Токен генерируется как URL-safe base64 UUID (без хэширования, так как токен короткий и имеет высокую энтропию)

**Reset Password Token:**
- Срок жизни: 1 час
- Тип: UUIDv7
- Одноразовость: после использования помечается как `used_at` в БД

**Forgot Password Token:**
- Срок жизни: 1 час
- Тип: UUIDv7
- Одноразовость: после использования помечается как `used_at` в БД

**Unsubscribe Token:**
- Хранится в `users.unsubscribe_token`
- Не истекает (постоянный токен для отписки)
- Используется для GDPR-совместимой отписки от рассылок

**Типы токенов и их назначение:**

| Тип токена | Длительность | Хранение | Одноразовый |
|------------|--------------|----------|-------------|
| Email verification | 24 часа | БД | Да |
| Reset password | 1 час | БД | Да |
| Forgot password | 1 час | БД | Да |
| Unsubscribe | Постоянный | users table | Нет |

**Token binding:**
- Применяется к `/refresh` и `/logout` — проверка `ip_address` и `user_agent_hash`
- **Не применяется к `/verify`** — пользователь может подтвердить email с любого устройства
- **Не применяется к `/forgot-password` и `/reset-password`** — пользователь может сбросить пароль с любого устройства

**Timing-safe comparison:**
- Обязательное константное сравнение токенов при verify/reset для защиты от timing attack

---

## 5.4.1 Защита от перечисления аккаунтов

**Цель:** Предотвратить возможность для злоумышленника определить, зарегистрирован ли определённый email в системе (email enumeration attack).

### Принципы защиты:

| Принцип | Описание |
|---------|----------|
| Единый формат ответа | Все неудачные попытки возвращают 401 с одним и тем же сообщением |
| Константное сравнение | Даже для несуществующих email выполняется хэширование и сравнение |
| Фиктивная проверка | Если email не найден, генерируется фиктивный хэш для одинакового времени ответа |

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
|----------|-----------|
| `/login` | 401 для несуществующего email и неверного пароля — одинаковый ответ |
| `/verify` | 401 для неверного токена — фиктивная проверка email |
| `/resend-verification` | 204 для любого email (даже несуществующего) — не раскрывает наличие аккаунта |
| `/forgot-password` | 204 для любого email — не раскрывает наличие аккаунта |

### Запрещённые поведения:

| Поведение | Риски |
|-----------|-------|
| `404 Not Found` | Раскрывает существование/несуществование аккаунта |
| `409 Email already registered` в `/login` | Раскрывает существование аккаунта |
| `400 Email not found` в `/login` | Раскрывает несуществование аккаунта |
| Различие во времени ответа | Может раскрыть наличие аккаунта через timing attack |

---

## 5.5 Защита от CSRF

**Стратегии защиты:**

Для endpoints без cookie (login, register) используется CSRF token, который генерируется сервером и должен быть отправлен в заголовке или теле запроса. Для endpoints с HTTP-only cookie (refresh, logout) используется `SameSite=Strict` и проверка Origin.

| Endpoint           | Cookie           | CSRF Protection |
|--------------------|------------------|-----------------|
| `/login`           | Нет              | CSRF token в заголовке `X-CSRF-Token` + `SameSite=None` (если cross-origin) |
| `/register`        | Нет              | CSRF token в заголовке `X-CSRF-Token` + `SameSite=None` (если cross-origin) |
| `/refresh`         | HTTP-only cookie | `SameSite=Strict` + проверка Origin |
| `/logout`          | HTTP-only cookie | `SameSite=Strict` + проверка Origin |
| `/me`              | Нет              | Проверка Origin |
| `/change-password` | HTTP-only cookie | `SameSite=Strict` + проверка Origin |

**Обязательные защиты:**

- `SameSite=Strict` для cookie (не отправляются при cross-site запросах)
- `Secure` флаг для cookie (только HTTPS)
- `HttpOnly` флаг для refresh token cookie
- **Origin validation** на сервере для всех запросов (список разрешённых origins)
- **Referer/Referrer policy** заголовки: `strict-origin-when-cross-origin`

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

**HTTP заголовки для защиты от различных атак:**

| Заголовок | Значение | Описание |
|-----------|----------|----------|
| `Content-Security-Policy` | `default-src 'self'` | Ограничивает источники контента (script, style, img) |
| `X-Content-Type-Options` | `nosniff` | Отключает MIME-sniffing (защита от某些 XSS) |
| `X-Frame-Options` | `DENY` | Защита от clickjacking (не разрешает фреймы) |
| `X-XSS-Protection` | `1; mode=block` | XSS фильтр браузера (legacy, но рекомендуется) |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Контроль referrer (не передаёт путь при cross-origin) |
| `Permissions-Policy` | `geolocation=(), microphone=()` | Ограничение доступа к API устройства |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | HSTS — обязательный HTTPS на 1 год |

**Рекомендации:**
- Устанавливать заголовки на всех ответах (глобальный middleware)
- Для `Content-Security-Policy` начните с `report-uri` для мониторинга нарушений
- `Strict-Transport-Security` добавляется только по HTTPS

---

## 5.7 Модель угроз

**Цель:** Идентифицировать потенциальные угрозы безопасности и описать митигации (меры защиты) для каждой.

| Угроза | Описание | Митигация |
|---------|----------|-----------|
| Brute force атака | Попытки подбора пароля/токена через множественные запросы | Ограничение частоты запросов (Redis), блокировка по IP/email, константное сравнение |
| Кража токенов | Украденный refresh token может использоваться злоумышленником | HTTP-only cookies, короткоживущие access tokens, rotation refresh tokens |
| Привязка токена | Токен используется с другого IP/устройства | Привязка IP + User-Agent для refresh tokens, проверка при каждом использовании |
| SQL-инъекция | Вредоносный SQL в user input | Prepared statements (ORM), параметризованные запросы |
| XSS | Вредоносный JavaScript в ответе браузеру | Content-Security-Policy, очистка input, заголовки безопасности |
| CSRF | Автоматический запрос с другого сайта | SameSite=Strict cookies, проверка Origin/Referer, CSRF-токены |
| Взлом паролей | Угадывание паролей из словаря/утечек | Argon2id (memory-hard), rate limiting, проверка на утечки |
| Перехват email | SMTP соединение перехватывается | TLS 1.3+ для SMTP, одноразовые токены, короткий срок действия |
| Атаки по времени | Время ответа раскрывает наличие email/пароля | Константное сравнение хэшей (timingSafeEqual) |
| Повторное использование токена | Украденный token используется повторно | Refresh token инвалидируется после usage, revoked + Redis blacklist |
| Перечисление аккаунтов | Злоумышленник определяет существование email | Ответы без деталей, timing-safe сравнение |
| Перехват сессии | Access token используется с другого IP | Короткоживущие access tokens, HTTP-only cookies, проверка Origin |
| DoS при logout | Массированные запросы logout для нагрузки | Rate limiting (30/10 мин по IP) для logout |

**Примечание:** Модель угроз должна обновляться при изменении архитектуры или добавлении новых endpoints.

---

## 5.8 Краткое резюме реализации безопасности

**Архитектура безопасности строится на трёх принципах:**

1. **Минимизация ущерба:** Короткоживущие токены (30 мин), rotation, revocation
2. **Препятствование атакам:** Rate limiting, timing-safe сравнения, token binding
3. **Прозрачность:** Аудит логирование, чёткие ответы для пользователя

| Механизм | Цель | Реализация |
|----------|------|------------|
| Ограничение частоты запросов | Защита от brute force и DoS | Redis + лимиты на endpoint |
| Rotation токенов | Одноразовость refresh tokens | Revoked флаг + Redis blacklist |
| Привязка токена | Препятствие краже токена | Проверка IP + User-Agent при refresh |
| Константное сравнение | Защита от timing attack | crypto.timingSafeEqual() |
| Защита от CSRF | Предотвращение cross-site запросов | SameSite=Strict + Origin validation |
| Защищённые cookies | XSS защита для refresh token | HttpOnly + Secure флаги |
| Короткоживущие токены | Минимизация окна атаки | Access token 30 минут (JWT exp) |
| Аудит логирование | Отслеживание подозрительных действий | Event schema (раздел 7) |

---

## 5.9 Соответствие стандартам

**Документация соответствия требованиям законодательства и стандартов безопасности:**

| Стандарт/Требование | Реализация | Примечание |
|--------------------|------------|------------|
| **GDPR** (General Data Protection Regulation) | Возможность полного удаления аккаунта через `/v1/auth/me` (DELETE) | Право на забвение — данные удаляются из БД и Redis |
| **PII protection** (Защита персональных данных) | Email хранится в plain text (опционально: зашифровано через column encryption) | При шифровании ключи хранятся в KMS |
| **Audit logging** | Все операции логируются через event schema (см. раздел 7) | Логи включают: ip, user_agent, timestamp, event_type, user_id |
| **TLS/SSL** | Обязательный TLS 1.3+ для всех внешних соединений (SMTP, DB) | Шифрование в транспорте |
| **Rate limiting** | Ограничение частоты запросов для предотвращения злоупотреблений | Соответствует рекомендациям OWASP |

---

## 5.10 Чеклист реализации

**Цель:** Проверка полноты реализации механизмов безопасности перед деплоем в production.

### Backend

- [ ] Константное сравнение для всех критичных проверок (password hash, tokens)
- [ ] Проверка Origin/Referer для всех auth endpoints
- [ ] Security Headers (CSP, HSTS, X-Frame-Options, X-XSS-Protection, Referrer-Policy)
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

### Database

- [ ] Индексы на email, token полям для быстрого поиска
- [ ] Foreign key связи с cascade delete
- [ ] TTL автоматическая очистка истёкших токенов
- [ ] Encrypt column для email (опционально)

### Monitoring

- [ ] Алерты на блокировки по IP/email
- [ ] Алерты на превышение rate limit
- [ ] Мониторинг revoked токенов (подозрительное использование)
- [ ] Логирование всех событий (см. раздел 7)