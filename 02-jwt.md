# 2. Структура JWT

## 2.1 Токен доступа

**Header:**

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

**Payload:**

```json
{
  "iss": "mystore-auth-service", // идентификатор сервиса
  "sub": "uuid-v7", // ID пользователя (time-based)
  "iat": 1726989600, // время выдачи (Unix timestamp)
  "exp": 1726991400 // время истечения (Unix timestamp, 30 минут)
}
```

**Срок жизни:** 30 минут  
**Алгоритм:** RS256  
**Ключ:** Приватный ключ хранится в переменной окружения `JWT_PRIVATE_KEY` (формат PEM)  
**Верификация:** Публичный ключ доступен через `/.well-known/jwks.json` или из `JWT_PUBLIC_KEY`

**Пояснение для `iss` (issuer):** Идентификатор сервиса используется при валидации токена для подтверждения, что токен выдан доверенным авторизационным сервером, а не подделан. При проверке подписи также проверяется, что значение `iss` в токене совпадает с ожидаемым идентификатором сервиса.

## Проверка `iss` при валидации access token

**Ожидаемое значение:** `mystore-auth-service`

**Процесс проверки:**

1. **Декодирование JWT** — access token декодируется без проверки подписи (header + payload в base64url)
2. **Проверка `iss`** — значение поля `iss` сравнивается с ожидаемым `mystore-auth-service`
3. **Проверка подписи** — если `iss` валиден, выполняется проверка RS256 подписи с помощью публичного ключа
4. **Проверка `exp`** — проверка срока действия токена

**Что происходит при несовпадении `iss`:**
- Если `iss` не равен `mystore-auth-service`, токен отклоняется с ошибкой `401 Unauthorized`
- Код ошибки: `ACCESS_TOKEN_INVALID`
- Сообщение: "Неверный или истекший токен доступа"
- **Важно:** Ошибка не раскрывает детали (не указывает, что именно `iss` не совпал) — это предотвращает information disclosure

**Псевдокод валидации access token:**

```javascript
function validateAccessToken(token) {
  try {
    // 1. Декодирование без проверки подписи
    const decoded = jwt.decode(token, { complete: true });
    if (!decoded) {
      throw new Error('Invalid token');
    }

    // 2. Проверка iss (issuer)
    if (decoded.payload.iss !== 'mystore-auth-service') {
      throw new Error('Invalid issuer');
    }

    // 3. Проверка подписи (RS256 с публичным ключом)
    jwt.verify(token, publicKey, { algorithms: ['RS256'] });

    // 4. Проверка exp (expiration)
    const now = Math.floor(Date.now() / 1000);
    if (decoded.payload.exp < now) {
      throw new Error('Token expired');
    }

    return decoded.payload;
  } catch (err) {
    throw new Error('ACCESS_TOKEN_INVALID');
  }
}
```

**Примечание:** Email и имя не включаются в access token, так как payload JWT не шифруется, а профиль может измениться за время жизни токена. Профиль клиент получает через `GET /v1/users/me`.

## 2.3 Обработка истекших токенов доступа

**Стратегия:** Token refresh (авто-обновление) при истечении в середине запроса

| Время до exp                | Действие                                                  | Ответ                                                  |
| --------------------------- | --------------------------------------------------------- | ------------------------------------------------------ |
| `exp - now > 5s`            | Обычный запрос                                            | 200 OK                                                 |
| `exp - now <= 5s`           | Авто-обновление (только safe methods: GET, HEAD, OPTIONS) | 200 OK + заголовок `X-Token-Refresh: true`             |
| `exp - now < 0` (просрочен) | Требуется refresh                                         | 401 Unauthorized + заголовок `X-Token-Status: expired` |

**Правила:**

- **Safe methods (GET, HEAD, OPTIONS):** Автоматически обновляют access token через refresh, если истек менее 5 секунд назад
- **Unsafe methods (POST, PUT, DELETE):** Возвращают 401 без авто-обновления
- **Header:** При авто-обновлении добавляется `X-Token-Refresh: true` для информирования клиента
- **Логирование:** Все авто-обновления логируются с `event: token.autorefresh`

**Рекомендация для клиентов:**

- При получении 401 с `X-Token-Status: expired` выполнить flow refresh token
- При `X-Token-Refresh: true` можно продолжить работу (token обновлён прозрачно)

## 2.4 Токен обновления

**Header:**

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

**Payload:**

```json
{
  "iss": "mystore-auth-service", // идентификатор сервиса
  "sub": "uuid-v7", // ID пользователя
  "iat": 1726989600, // время выдачи (Unix timestamp)
  "exp": 1727076000 // время истечения (Unix timestamp, 7 дней)
}
```

**Срок жизни:** 7 дней  
**Хранение:** HTTP-only cookie (домен: `api.mystore.com`, путь: `/`)  
**Token Binding:** Refresh token привязан к `ip_address` и `user_agent_hash` (проверяются при валидации)  
**Rotation:** При каждом refresh генерируется новый refresh token, старый инвалидируется (одноразовость)  
**Revocation:** `jti` добавляется в Redis blacklist с TTL = оставшееся время жизни

**Пояснение для `iss` (issuer):** То же, что и для access token — подтверждение, что токен выдан доверенным авторизационным сервером.

---

## 2.5 Использование HTTP-only cookie для refresh token

**Архитектура хранения:**

| Токен | Место хранения | Доступ | Безопасность |
|-------|----------------|--------|--------------|
| Access token | Response body (JSON) | JS-доступ | XSS уязвим |
| Refresh token | HTTP-only cookie | Недоступен JS | XSS защищён |

**Настройка cookie:**

```javascript
// Пример установки cookie сервером
res.cookie('refresh_token', refreshTokenValue, {
  httpOnly: true,           // Недоступен через JavaScript (защита от XSS)
  secure: true,             // Только HTTPS (в production)
  sameSite: 'Strict',       // Защита от CSRF (не отправляется при cross-site запросах)
  domain: 'api.mystore.com',
  path: '/',
  maxAge: 7 * 24 * 60 * 60 * 1000  // 7 дней в миллисекундах
});
```

**Как работает flow:**

1. **При login/verify:** Сервер устанавливает refresh token в HTTP-only cookie
2. **При refresh/logout:** Браузер автоматически отправляет cookie с каждым запросом
3. **При logout:** Сервер инвалидирует токен и может очистить cookie (через `Set-Cookie: refresh_token=; Max-Age=0`)

**Преимущества HTTP-only cookie:**

- **XSS защита:** JavaScript не может прочитать или украсть refresh token
- **Автоматическая отправка:** Браузер сам добавляет cookie к запросам (не нужно хранить в localStorage/Redux)
- **SameSite protection:** `SameSite=Strict` предотвращает отправку cookie при cross-site запросах

**Ограничения:**

- **Domain ограничение:** Cookie отправляется только на `api.mystore.com` (или поддомены с `domain=.mystore.com`)
- **HTTPS рекомендация:** `Secure` флаг требует HTTPS в production
- **CSRF защита:** SameSite=Strict предотвращает большинство CSRF атак, но для критичных операций может потребоваться дополнительная защита (CSRF token)

**Клиентская логика:**

```javascript
// При login - сервер устанавливает cookie
fetch('/v1/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ email, password }),
  credentials: 'include'  // Важно: включает cookie в запрос
});

// При refresh - cookie отправляется автоматически
fetch('/v1/auth/refresh', {
  method: 'POST',
  credentials: 'include'
});

// При logout - cookie отправляется автоматически
fetch('/v1/auth/logout', {
  method: 'POST',
  credentials: 'include'
});
```
