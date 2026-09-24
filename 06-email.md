# 6. Интеграция email

## 6.1 Клиент SMTP

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

## 6.2 Шаблон: Подтверждение регистрации

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

## 6.3 Шаблон: Сброс пароля

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

## 6.4 Очередь email (для высокой нагрузки)

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

## 6.5 Переменные окружения

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
