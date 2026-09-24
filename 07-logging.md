# 7. Логирование

## 7.1 Схема событий

**Описание:** Стандартизированная схема логирования событий аутентификации для аудита, мониторинга и анализа безопасности.

```json
{
  "timestamp": "2024-09-22T10:30:00.000Z",
  "event": "user.registered",
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
    "jti": "01hv...",
    "request_method": "GET",
    "path": "/api/users"
  }
}
```

### Описание полей

| Поле | Тип | Обязательное | Описание |
|------|-----|--------------|----------|
| `timestamp` | string (ISO 8601) | Да | Время события в UTC |
| `event` | string | Да | Тип события (enum) |
| `user_id` | string (UUIDv7) | Нет | ID пользователя (null для неавторизованных событий) |
| `email` | string | Нет | Email пользователя |
| `ip_address` | string | Да | IP-адрес клиента |
| `user_agent` | object | Да | Детали User-Agent (агрегированные, без деталей для GDPR) |
| `details` | object | Нет | Дополнительные данные события |

**Примечание:** `user_agent` хранится в агрегированном виде без деталей, которые могут идентифицировать пользователя (согласно GDPR принципу минимизации данных).

---

## 7.2 Критические события

| Событие | Уровень | Основные поля | Описание |
|---------|---------|---------------|----------|
| `user.registered` | high | user_id, email, ip_address, user_agent | Новая регистрация пользователя |
| `user.verified` | medium | user_id, email, ip_address | Подтверждение email |
| `user.logged_in` | medium | user_id, email, ip_address, user_agent | Успешный вход |
| `user.logged_out` | low | user_id, email, ip_address | Выход из системы |
| `auth.failed` | critical | email, ip_address, user_agent | Неудачная попытка входа |
| `token.autorefresh` | low | jti, request_method, path | Авто-обновление access token |
| `auth.token_refreshed` | medium | user_id, email, ip_address, jti, jti_old | Обновление токенов |

### Уровни важности

- **critical** — события требующие немедленного внимания (возможная атака)
- **high** — важные события аудита пользователей
- **medium** — события для аудита и мониторинга
- **low** — события для анализа сессий и поведения

---

## 7.3 Примеры логов

### User registered
```json
{
  "timestamp": "2024-09-22T10:30:00.000Z",
  "event": "user.registered",
  "user_id": "01hv5n6q9r2s3t4u5v6w7x8y9z",
  "email": "user@example.com",
  "ip_address": "192.168.1.100",
  "user_agent": {
    "browser": "Chrome",
    "browser_version": "128.0",
    "os": "Windows",
    "os_version": "11",
    "device": "Desktop"
  }
}
```

### Auth failed (critical)
```json
{
  "timestamp": "2024-09-22T11:15:30.000Z",
  "event": "auth.failed",
  "user_id": null,
  "email": "admin@target.com",
  "ip_address": "203.0.113.45",
  "user_agent": {
    "browser": "curl",
    "browser_version": "8.0",
    "os": "Linux",
    "os_version": null,
    "device": "Server"
  },
  "details": {
    "reason": "invalid_password"
  }
}
```

### Token autorefresh
```json
{
  "timestamp": "2024-09-22T11:45:58.000Z",
  "event": "token.autorefresh",
  "user_id": "01hv5n6q9r2s3t4u5v6w7x8y9z",
  "email": "user@example.com",
  "ip_address": "192.168.1.100",
  "user_agent": {
    "browser": "Chrome",
    "browser_version": "128.0",
    "os": "Windows",
    "os_version": "11",
    "device": "Desktop"
  },
  "details": {
    "jti": "01hv5a6b7c8d9e0f1g2h3i4j5k",
    "request_method": "GET",
    "path": "/api/users/me"
  }
}
```

---

## 7.4 Хранение логов

**Технологии:**

| Назначение | Технология | TTL / Retention |
|------------|------------|-----------------|
| Быстрый поиск | Elasticsearch | 90 дней |
| Архив | PostgreSQL (table: audit_logs) | 7 лет |
| SIEM | Sentry / Datadog | 30 дней |

**Формат вывода в файл:**
```
[ISO_TIMESTAMP] [LEVEL] [event] [user_id] [ip] - message {details}
```

**Пример:**
```
[2024-09-22T10:30:00.000Z] [INFO] [user.registered] [01hv5n6q9r2s3t4u5v6w7x8y9z] [192.168.1.100] - User registered {email: "user@example.com"}
```
