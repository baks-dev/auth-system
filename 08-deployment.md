# 8. Переменные окружения

| Переменная               | Обязательная | Описание                                                          |
| ------------------------ | ------------ | ----------------------------------------------------------------- |
| `NODE_ENV`               | Да           | development, staging, production                                  |
| `PORT`                   | Да           | Порт сервера (по умолчанию: 3000)                                 |
| `JWT_PRIVATE_KEY`        | Да           | Приватный ключ для RS256 (PEM format, base64-encoded)             |
| `JWT_PUBLIC_KEY`         | Да           | Публичный ключ для верификации RS256 (PEM format, base64-encoded) |
| `DATABASE_URL`           | Да           | PostgreSQL connection string                                      |
| `REDIS_URL`              | Да           | Redis connection string (включая пароль, если есть)               |
| `SMTP_HOST`              | Да           | SMTP сервер                                                       |
| `SMTP_PORT`              | Да           | SMTP порт (587 для TLS, 465 для SSL)                                              |
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

## 8.1 Пример конфигурации (production)

```
NODE_ENV=production
PORT=3000

JWT_PRIVATE_KEY=LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2Z0lCQURBTkJna3Fqa2lHOXcwQkFRRUZBQVNDQktjd2dnU2pBZ0VBQW9JQkFRREZuWk1QeGNuQlBZ...
JWT_PUBLIC_KEY=LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0KTUlJQklqQU5CZ2txaGtpRzl3MEJBUXNGQUFCQ0NBU0N3Z2dFa01BMEdDU3FHU0liM0RRRUIvVUFNQlR4UXdIeV...

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
