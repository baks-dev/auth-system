# 8. Переменные окружения

**Обзор:** Документ описывает все переменные окружения для настройки системы авторизации. Все переменные делятся на обязательные (без которых приложение не запустится) и опциональные (с разумными значениями по умолчанию).

## 8.1 Базовые настройки

| Переменная | Обязательная | Категория | Описание |
|------------|--------------|-----------|----------|
| `NODE_ENV` | Да | Базовая | Окружение: development, staging, production |
| `PORT` | Да | Базовая | Порт сервера (по умолчанию: 3000) |

## 8.2 Безопасность

| Переменная | Обязательная | Категория | Описание |
|------------|--------------|-----------|----------|
| `JWT_PRIVATE_KEY` | Да | Безопасность | Приватный ключ для RS256 (PEM format, base64-encoded) |
| `JWT_PUBLIC_KEY` | Да | Безопасность | Публичный ключ для верификации RS256 (PEM format, base64-encoded) |

**Примечание:** PEM ключи должны быть закодированы в base64. Для генерации ключей см. [Генерация JWT ключей](#86-генерация-jwt-ключей).

## 8.3 База данных

| Переменная | Обязательная | Категория | Описание |
|------------|--------------|-----------|----------|
| `DATABASE_URL` | Да | База данных | PostgreSQL connection string (формат: `postgresql://user:pass@host:port/db?ssl=true`) |

## 8.4 Кэширование

| Переменная | Обязательная | Категория | Описание |
|------------|--------------|-----------|----------|
| `REDIS_URL` | Да | Кэширование | Redis connection string (формат: `redis://:password@host:port/db`) |

## 8.5 Email

| Переменная | Обязательная | Категория | Описание |
|------------|--------------|-----------|----------|
| `SMTP_HOST` | Да | Email | SMTP сервер (например: smtp.mystore.com) |
| `SMTP_PORT` | Да | Email | SMTP порт (587 для TLS, 465 для SSL) |
| `SMTP_USER` | Да | Email | SMTP пользователь (например: noreply@mystore.com) |
| `SMTP_PASS` | Да | Email | SMTP пароль |
| `SMTP_FROM` | Да | Email | From email адрес (откуда отправляются письма) |
| `SMTP_TIMEOUT` | Нет | Email | Таймаут SMTP соединения в мс (по умолчанию: 30000) |
| `SMTP_TLS_MIN_VERSION` | Нет | Email | Минимальная версия TLS (по умолчанию: TLSv1.2) |

## 8.6 URL

| Переменная | Обязательная | Категория | Описание |
|------------|--------------|-----------|----------|
| `APP_URL` | Да | URL | URL приложения (для email ссылок, без trailing slash) |
| `APP_VERIFY_URL` | Нет | URL | URL страницы верификации (по умолчанию: APP_URL/verify) |
| `APP_RESET_PASSWORD_URL` | Нет | URL | URL страницы сброса пароля (по умолчанию: APP_URL/reset-password) |
| `APP_UNSUBSCRIBE_URL` | Нет | URL | URL страницы отписки (по умолчанию: APP_URL/unsubscribe) |

## 8.7 Rate limiting

| Переменная | Обязательная | Категория | Описание |
|------------|--------------|-----------|----------|
| `RATE_LIMIT_WINDOW_MS` | Нет | Rate limiting | Окно rate limiting в мс (по умолчанию: 60000) |
| `RATE_LIMIT_MAX` | Нет | Rate limiting | Максимальное кол-во запросов в окне (по умолчанию: 100) |

## 8.8 Логирование

| Переменная | Обязательная | Категория | Описание |
|------------|--------------|-----------|----------|
| `LOG_LEVEL` | Нет | Логирование | Уровень логов: debug, info, warn, error (по умолчанию: info) |

---

## 8.9 Пример конфигурации (production)

```
# Окружение
NODE_ENV=production
PORT=3000

# Безопасность (PEM ключи base64-encoded)
JWT_PRIVATE_KEY=LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2Z0lCQURBTkJna3Fqa2lHOXcwQkFRRUZBQVNDQktjd2dnU2pBZ0VBQW9JQkFRREZuWk1QeGNuQlBZ...
JWT_PUBLIC_KEY=LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0KTUlJQklqQU5CZ2txaGtpRzl3MEJBUXNGQUFCQ0NBU0N3Z2dFa01BMEdDU3FHU0liM0RRRUIvVUFNQlR4UXdIeV...

# База данных
DATABASE_URL=postgresql://auth_user:secure_password@db.mystore.com:5432/auth_db?ssl=true

# Redis
REDIS_URL=redis://:secure_redis_password@redis.mystore.com:6379/0

# SMTP
SMTP_HOST=smtp.mystore.com
SMTP_PORT=587
SMTP_USER=noreply@mystore.com
SMTP_PASS=smtp_password_here
SMTP_FROM=noreply@mystore.com
SMTP_TIMEOUT=30000
SMTP_TLS_MIN_VERSION=TLSv1.2

# URL
APP_URL=https://mystore.com
APP_VERIFY_URL=https://mystore.com/verify
APP_RESET_PASSWORD_URL=https://mystore.com/reset-password
APP_UNSUBSCRIBE_URL=https://mystore.com/unsubscribe

# Rate limiting
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=100

# Логирование
LOG_LEVEL=info
```

---

## 8.10 Пример конфигурации (development)

```
# Окружение
NODE_ENV=development
PORT=3000

# Безопасность (используйте test ключи или сгенерируйте свои)
JWT_PRIVATE_KEY=LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2Z0lCQURBTkJna3Fqa2lHOXcwQkFRRUZBQVNDQktjd2dnU2pBZ0VBQW9JQkFRREZuWk1QeGNuQlBZ...
JWT_PUBLIC_KEY=LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0KTUlJQklqQU5CZ2txaGtpRzl3MEJBUXNGQUFCQ0NBU0N3Z2dFa01BMEdDU3FHU0liM0RRRUIvVUFNQlR4UXdIeV...

# База данных (локально без SSL)
DATABASE_URL=postgresql://dev_user:dev_pass@localhost:5432/auth_dev

# Redis (локально без пароля)
REDIS_URL=redis://localhost:6379/0

# SMTP (используйте Mailtrap или аналог для тестов)
SMTP_HOST=smtp.mailtrap.io
SMTP_PORT=587
SMTP_USER=your_mailtrap_user
SMTP_PASS=your_mailtrap_pass
SMTP_FROM=noreply@mystoredev.com
SMTP_TIMEOUT=30000

# URL (используйте localhost)
APP_URL=http://localhost:3000
APP_VERIFY_URL=http://localhost:3000/verify
APP_RESET_PASSWORD_URL=http://localhost:3000/reset-password
APP_UNSUBSCRIBE_URL=http://localhost:3000/unsubscribe

# Rate limiting (для разработки можно увеличить)
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=500

# Логирование (debug для разработки)
LOG_LEVEL=debug
```

---

## 8.11 Генерация JWT ключей

Для генерации пары RSA ключей используйте OpenSSL:

```bash
# Генерация приватного ключа
openssl genrsa -out jwt-private.key 2048

# Извлечение публичного ключа из приватного
openssl rsa -in jwt-private.key -pubout -out jwt-public.key

# Кодирование в base64 для переменных окружения
base64 -w 0 jwt-private.key > jwt-private.key.base64
base64 -w 0 jwt-public.key > jwt-public.key.base64
```

**Важно:**
- Приватный ключ храните в безопасном месте, НЕ коммитьте в Git
- Используйте `.env` файлы с `.gitignore`
- В production используйте secrets manager (Vault, AWS Secrets Manager, etc.)

---

## 8.12 Валидация конфигурации

Приложение должно проверять обязательные переменные при старте:

```javascript
// config/validate.js
const REQUIRED_VARS = [
  'NODE_ENV',
  'PORT',
  'JWT_PRIVATE_KEY',
  'JWT_PUBLIC_KEY',
  'DATABASE_URL',
  'REDIS_URL',
  'SMTP_HOST',
  'SMTP_PORT',
  'SMTP_USER',
  'SMTP_PASS',
  'SMTP_FROM',
  'APP_URL'
];

function validateConfig() {
  const missing = REQUIRED_VARS.filter(varName => !process.env[varName]);

  if (missing.length > 0) {
    console.error('ОШИБКА: Отсутствуют обязательные переменные:');
    missing.forEach(varName => console.error(`  - ${varName}`));
    process.exit(1);
  }

  // Дополнительные проверки
  if (!['development', 'staging', 'production'].includes(process.env.NODE_ENV)) {
    console.error('ОШИБКА: NODE_ENV должен быть development, staging или production');
    process.exit(1);
  }

  if (process.env.NODE_ENV === 'production' && process.env.LOG_LEVEL === 'debug') {
    console.warn('ПРЕДУПРЕЖДЕНИЕ: Уровень логов debug в production может повлиять на производительность');
  }
}

module.exports = { validateConfig };
```

---

## 8.13 Рекомендации по безопасности

1. **Никогда не коммитьте `.env` файлы** — добавьте `.env*` в `.gitignore`
2. **Используйте разные ключи** для каждой среды (development, staging, production)
3. **Ротация ключей** — регулярно обновляйте JWT ключи (рекомендуется: каждые 90 дней)
4. **Переменные с паролями** — храните в secrets manager (Vault, AWS Secrets Manager)
5. **Проверка конфига** — всегда запускайте `validateConfig()` перед стартом приложения
6. **SSL/TLS** — в production всегда используйте `ssl=true` в DATABASE_URL
7. **Network isolation** — Redis и PostgreSQL должны быть недоступны извне VPC
