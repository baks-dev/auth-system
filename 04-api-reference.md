# 4. Конечные точки API

## 4.1 POST /v1/auth/register

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

## 4.2 POST /v1/auth/verify

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

## 4.3 POST /v1/auth/login

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

## 4.4 POST /v1/auth/refresh

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

## 4.5 POST /v1/auth/logout

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

## 4.6 GET /v1/auth/me

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

## 4.7 POST /v1/auth/resend-verification

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

## 4.8 POST /v1/auth/forgot-password

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
}'
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

## 4.9 POST /v1/auth/reset-password

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

## 4.10 POST /v1/auth/unsubscribe

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

## 4.11 PUT /v1/auth/change-password

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
```

---

## 4.12 DELETE /v1/auth/me

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
