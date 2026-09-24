# 7. Логирование

## 7.1 Схема событий

```json
{
  "timestamp": "2024-09-22T10:30:00.000Z",
  "event": "user.registered" | "user.verified" | "user.logged_in" | "user.logged_out" | "auth.failed" | "token.autorefresh" | "auth.token_refreshed",
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
    "jti": "01hv...",      // JWT ID (для token.autorefresh и auth.token_refreshed)
    "jti_old": "01hv...",  // старый JWT ID (для auth.token_refreshed)
    "request_method": "GET",
    "path": "/api/users"
  }
}
```

**Примечание:** `user_agent` хранится в агрегированном виде без деталей, которые могут идентифицировать пользователя (согласно GDPR принципу минимизации данных).

## 7.2 Критические события

- `user.registered` — логировать user_id, email, ip_address, user_agent (базовая информация)
- `user.verified` — логировать user_id, email, ip_address
- `user.logged_in` — логировать user_id, email, ip_address, user_agent (базовая информация)
- `user.logged_out` — логировать user_id, email, ip_address
- `auth.failed` — логировать email, ip_address, user_agent (базовая информация)
- `token.autorefresh` — логировать jti, метод запроса и путь (при авто-обновлении access token)
- `auth.token_refreshed` — логировать user_id, email, ip_address, jti (новый токен), jti_old (старый токен), user_agent (базовая информация)
