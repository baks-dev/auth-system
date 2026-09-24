# 6. Интеграция email

**Обзор:** Документ описывает настройку SMTP-клиента, шаблоны email-писем для системы авторизации и очередь email-сообщений для высоконагруженных систем.

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

## 6.2 Шаблоны email-писем

| Тип письма | Назначение | Срок действия | Template variables |
|------------|------------|---------------|-------------------|
| Verification | Подтверждение email при регистрации | 24 часа | name, token, unsubscribe_token |
| Password Reset | Сброс пароля по запросу | 1 час | email, token, unsubscribe_token |
| Unsubscribe | Отписка от рассылки | Постоянный | email, unsubscribe_token |

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

| Параметр | Значение | Описание |
|----------|----------|----------|
| `concurrency` | 10 | Параллельные воркеры (одновременных писем) |
| `attempts` | 3 | Повтор при ошибке |
| `delay` | 5000 мс | Задержка между повторами |
| `backoff` | exponential | Экспоненциальная задержка между попытками |

**Job lifecycle:**
1. Job создается в очереди с типом `verification`/`password_reset`/`unsubscribe`/`notification`
2. Worker забирает job из очереди
3. Формируется email на основе шаблона
4. Письмо отправляется через SMTP-клиент
5. При успехе job удаляется из очереди
6. При неудаче (3 попытки) job перемещается в "dead letter queue" для анализа

---

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

---

## 6.6 Безопасность email

**Защита от спама и фишинга:**

| Механизм | Описание |
|----------|----------|
| SPF (Sender Policy Framework) | DNS-запись разрешает отправку с конкретных серверов |
| DKIM (DomainKeys Identified Mail) | Подпись писем приватным ключом домена |
| DMARC (Domain-based Message Authentication) | Политика обработки неавторизованных писем |
| TLS 1.3+ | Шифрование соединения с SMTP-сервером |
| No personal data | В письмах только необходимые данные (email, token) |
| One-time tokens | Токены одноразовые и имеют ограниченный срок действия |

**Рекомендации:**
- Использовать отдельный домен для noreply@ (например, noreply@mystore.com)
- Настроить SPF, DKIM, DMARC в DNS провайдера
- Логировать все попытки отправки (успех/ошибка)
- Не отправлять пароли в письмах (только ссылки для сброса)
```
