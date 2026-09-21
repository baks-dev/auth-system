# Техническая спецификация: Система регистрации и авторизации

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
│             - Rate limiting               │
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
│  └─────────────────────────────────────┘  │
└──────┬────────────────────────────────────┘
       │
       │ PostgreSQL
       │
┌──────▼────────────────────────────────────┐
│              Database Layer               │
│  - users                                  │
│  - email_verification_tokens              │
│  - refresh_tokens                         │
│  - login_attempts                         │
└───────────────────────────────────────────┘
```

---

## 2. JWT Структура

### 2.1 Access Token

**Header:**
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload:**
```json
{
  "sub": "uuid-v7",           // user ID (time-based)
  "email": "user@example.com",
  "name": "Иван Иванов",
  "role": "user",             // "user" | "admin"
  "is_email_verified": true,
  "iat": 1726989600,          // issued at (unix timestamp)
  "exp": 1726991400           // expires at (unix timestamp)
}
```

**Срок жизни:** 30 минут  
**Секрет:** Хранится в переменной окружения `JWT_ACCESS_SECRET`  
**Алгоритм:** HS256

### 2.2 Refresh Token

**Payload аналогичен Access Token**  
**Срок жизни:** 7 дней  
**Хранение:** HTTP-only cookie (домен: `api.mystore.com`)  
**Одноразовость:** После использования инвалидируется в БД

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
| revoked | BOOLEAN | DEFAULT false |
| expires_at | TIMESTAMP | NOT NULL |
| created_at | TIMESTAMP | DEFAULT NOW() |

**Описание полей:**
- `id` — уникальный идентификатор токена (UUID v7)
- `user_id` — референс на пользователя, каскадное удаление (CASCADE)
- `token` — хэшированный SHA-256 refresh token (оригинал хранится в HTTP-only cookie)
- `revoked` — флаг инвалидации (true после logout или компрометации)
- `expires_at` — время истечения токена (7 дней от создания)
- `created_at` — время выдачи токена

**Индексы:**
- `idx_refresh_token` (token) — для быстрого поиска (по хэшу)
- `idx_refresh_user_id` (user_id) — для поиска активных токенов
- `idx_refresh_expires_at` (expires_at) — для очистки истекших токенов

### 3.4 login_attempts

| Поле | Тип | Описание |
|------|-----|----------|
| id | UUID | PRIMARY KEY |
| email | VARCHAR(255) | NOT NULL |
| ip_address | VARCHAR(45) | NOT NULL |
| success | BOOLEAN | NOT NULL |
| failed_at | TIMESTAMP | DEFAULT NOW() |

**Описание полей:**
- `id` — уникальный идентификатор записи (UUID v7)
- `email` — email, с которого производилась попытка входа
- `ip_address` — IP-адрес (IPv4 или IPv6, VARCHAR(45) для поддержки IPv6)
- `success` — результат попытки: true (успех) или false (неудача)
- `failed_at` — время попытки входа

**Индексы:**
- `idx_attempts_email_ip` (email, ip_address) — для анализа атак
- `idx_attempts_failed_at` (failed_at) — для очистки старых записей
- `idx_attempts_email_success` (email, success) — для статистики

**Комментарии:**
- Логируются все попытки входа (успешные и неуспешные)
- Для rate limiting: count за последние N минут по IP
- Для блокировки: count неудачных по email за час

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
- `401`: Invalid credentials or unverified email
- `429`: Rate limited

---

### 4.4 POST /v1/auth/refresh

**Описание:** Получение нового access token

**Headers:**
```
Refresh-Token: eyJhbG...
```

**Success Response (200):**

```bash
curl -X POST https://api.mystore.com/v1/auth/refresh \
  -H "Refresh-Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
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
- `404`: User not found
- `409`: Email already verified
- `429`: Rate limited

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

**Note:** Не раскрывает существование email (для безопасности)

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

### 4.10 PUT /v1/auth/change-password

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

**Algorithm:**
1. На каждую неудачную попытку логируем IP + email
2. Если за последний час:
   - 5 неудачных → блокировка 15 минут
   - 10 неудачных → блокировка 24 часа

---

## 6. Email Integration

### 6.1 Template: Registration Confirmation

**Subject:** Подтверждение email для MyStore

**Body:**
```
Добро пожаловать в MyStore, {name}!

Пожалуйста, подтвердите свой email, перейдя по ссылке:
https://mystore.com/verify?token={token}

Ссылка действует 24 часа.

Если вы не регистрировались в MyStore, проигнорируйте это письмо.
```

### 6.2 Template: Password Reset

**Subject:** Сброс пароля для MyStore

**Body:**
```
{email}, вы запросили сброс пароля.

Перейдите по ссылке для установки нового пароля:
https://mystore.com/reset-password?token={token}

Ссылка действует 1 час.

Если вы не запрашивали сброс пароля, проигнорируйте это письмо.
```

### 6.3 Environment Variables

```
SMTP_HOST=smtp.mystore.com
SMTP_PORT=587
SMTP_USER=noreply@mystore.com
SMTP_PASS=***
SMTP_FROM=noreply@mystore.com

EMAIL_VERIFY_URL=https://mystore.com/verify
PASSWORD_RESET_URL=https://mystore.com/reset-password
```

---

## 7. Logging

### 7.1 Event Schema

```json
{
  "timestamp": "2024-09-22T10:30:00.000Z",
  "event": "user.registered" | "user.verified" | "user.logged_in" | "user.logged_out" | "auth.failed",
  "user_id": "01a0c637-3aa0-73d4-b85e-8f89aa81e711",
  "email": "user@example.com",
  "ip_address": "192.168.1.1",
  "user_agent": "Mozilla/5.0...",
  "details": {}
}
```

### 7.2 Critical Events

- `auth.failed` — логировать с IP и email
- `user.verified` — логировать user_id
- `user.logged_in` — логировать user_id и IP
- `user.logged_out` — логировать user_id

---

## 8. Environment Variables

| Переменная | Обязательная | Описание |
|------------|--------------|----------|
| `NODE_ENV` | Да | development, staging, production |
| `PORT` | Да | Порт сервера |
| `JWT_ACCESS_SECRET` | Да | Секрет для access tokens |
| `JWT_REFRESH_SECRET` | Да | Секрет для refresh tokens |
| `DATABASE_URL` | Да | PostgreSQL connection string |
| `REDIS_URL` | Да | Redis connection string |
| `SMTP_HOST` | Да | SMTP сервер |
| `SMTP_PORT` | Да | SMTP порт |
| `SMTP_USER` | Да | SMTP пользователь |
| `SMTP_PASS` | Да | SMTP пароль |
| `SMTP_FROM` | Да | From email адрес |
| `APP_URL` | Да | URL приложения (для email ссылок) |

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
- [ ] XSS защита
- [ ] SQL injection защита (prepared statements)

---

## 10. Testing Strategy

### Unit Tests
- Хэширование паролей (argon2id)
- Подпись/проверка JWT
- Генерация токенов электронной почты 
- Ограничение запросов / Rate limiting

### Интеграционные тесты
- Полный процесс регистрации / registration
- Процесс подтверждения электронной почты / verification
- Процесс входа в систему / login
- Обновления токена / Token refresh
- Процесс выхода из системы / logout
- Ограничение запросов / Rate limiting

### E2E Tests
- сценарии используя фреймворки для E2E тестирования (Cypress/Playwright)
- Проверка UI (если есть)
