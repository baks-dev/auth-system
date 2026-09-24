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
  "iss": "mystore-auth-service",
  "sub": "uuid-v7",
  "iat": 1726989600,
  "exp": 1726991400,
  "jti": "uuid-v7"
}
```

**Срок жизни:** 30 минут  
**Алгоритм:** RS256  
**Ключ:** Приватный ключ хранится в переменной окружения `JWT_PRIVATE_KEY` (формат PEM)  
**Верификация:** Публичный ключ доступен через `/.well-known/jwks.json` или из `JWT_PUBLIC_KEY`

### Структура payload

| Клайм | Тип | Описание |
|-------|-----|----------|
| `iss` | string | Идентификатор сервиса (`mystore-auth-service`) |
| `sub` | string | ID пользователя (UUIDv7, time-based) |
| `iat` | number | Время выдачи (Unix timestamp) |
| `exp` | number | Время истечения (Unix timestamp, +30 минут от iat) |
| `jti` | string | Unique JWT ID (UUIDv7 для отслеживания отозванных токенов) |

---

## 2.2 Проверка `iss` при валидации access token

**Ожидаемое значение:** `mystore-auth-service`

**Процесс проверки:**

```mermaid
sequenceDiagram
    participant T as Token
    participant D as Decoder
    participant I as Issuer Check
    participant V as Verify Signature
    participant E as Exp Check
    participant R as Response

    T->>D: Decode (base64url)
    D->>I: Extract iss
    I->>I: Compare with<br/>mystore-auth-service
    alt Issuer valid
        I->>V: Verify RS256 signature
        V->>E: Check exp claim
        alt Not expired
            E->>R: Valid token
        else Expired
            R-->>T: 401 ACCESS_TOKEN_INVALID
        end
    else Issuer invalid
        R-->>T: 401 ACCESS_TOKEN_INVALID
    end
```

### Что происходит при несовпадении `iss`:

- Если `iss` не равен `mystore-auth-service`, токен отклоняется с ошибкой `401 Unauthorized`
- Код ошибки: `ACCESS_TOKEN_INVALID`
- Сообщение: "Неверный или истекший токен доступа"
- **Важно:** Ошибка не раскрывает детали (не указывает, что именно `iss` не совпал) — это предотвращает information disclosure

### Псевдокод валидации access token:

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

---

## 2.3 Обработка истекших токенов доступа

**Стратегия:** Token refresh (авто-обновление) при истечении в середине запроса

```mermaid
stateDiagram-v2
    [*] --> Valid

    Valid --> AutoRefresh: exp - now <= 5s<br/>safe method (GET/HEAD/OPTIONS)
    Valid --> RequestFailed: exp - now < 0<br/>unsafe method (POST/PUT/DELETE)

    AutoRefresh --> Valid: New access token generated<br/>X-Token-Refresh: true
    RequestFailed --> Expired: 401 Unauthorized<br/>X-Token-Status: expired

    Expired --> Revoked: Client performs refresh flow
    Revoked --> [*]
```

### Поведение по времени до истечения

| Время до exp                | Действие                                                  | Ответ                                                  |
|---------------------------|-----------------------------------------------------------|--------------------------------------------------------|
| `exp - now > 5s`            | Обычный запрос                                            | 200 OK                                                 |
| `exp - now <= 5s`           | Авто-обновление (только safe methods: GET, HEAD, OPTIONS) | 200 OK + заголовок `X-Token-Refresh: true`             |
| `exp - now < 0` (просрочен) | Требуется refresh                                         | 401 Unauthorized + заголовок `X-Token-Status: expired` |

### Правила:

- **Safe methods (GET, HEAD, OPTIONS):** Автоматически обновляют access token через refresh, если истек менее 5 секунд назад
- **Unsafe methods (POST, PUT, DELETE):** Возвращают 401 без авто-обновления
- **Header:** При авто-обновлении добавляется `X-Token-Refresh: true` для информирования клиента
- **Логирование:** Все авто-обновления логируются с `event: token.autorefresh`

### Рекомендация для клиентов:

- При получении 401 с `X-Token-Status: expired` выполнить flow refresh token
- При `X-Token-Refresh: true` можно продолжить работу (token обновлён прозрачно)

---

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
  "iss": "mystore-auth-service",
  "sub": "uuid-v7",
  "iat": 1726989600,
  "exp": 1727076000,
  "jti": "uuid-v7"
}
```

**Срок жизни:** 7 дней  
**Хранение:** HTTP-only cookie (домен: `api.mystore.com`, путь: `/`)  
**Token Binding:** Refresh token привязан к `ip_address` и `user_agent_hash` (проверяются при валидации)  
**Rotation:** При каждом refresh генерируется новый refresh token, старый инвалидируется (одноразовость)  
**Revocation:** `jti` добавляется в Redis blacklist с TTL = оставшееся время жизни

### Структура payload

| Клайм | Тип | Описание |
|-------|-----|----------|
| `iss` | string | Идентификатор сервиса |
| `sub` | string | ID пользователя |
| `iat` | number | Время выдачи |
| `exp` | number | Время истечения (+7 дней от iat) |
| `jti` | string | Unique JWT ID для revocation |

### Жизненный цикл refresh token

**Описание жизненного цикла:**
Жизненный цикл refresh token начинается с состояния Created при генерации токена на endpoints `/login` или `/verify`. Токен переходит в состояние Active после сохранения в базе данных и установки как HTTP-only cookie. В состоянии Active токен может быть использован для обновления access token — при этом происходит rotation: старый токен становится Revoked, новый создается как Active. Токен может быть переведен в состояние Revoked через вызов `/logout` (установка `revoked = TRUE` в БД) или автоматически при истечении срока жизни (7 дней). Revoked и Expired токены впоследствии удаляются из БД крон-задачей через несколько дней.

```mermaid
stateDiagram-v2
    [*] --> Created: /login, /verify

    Created --> Active: Token saved to DB<br/>HTTP-only cookie set

    Active --> Active: /refresh<br/>Rotation: old revoked, new created

    Active --> Revoked: /logout<br/>revoked = TRUE in DB

    Active --> Expired: Срок жизни истек<br/>(7 дней)

    Revoked --> [*]: Удален из БД<br/>(cron-задача после N дней)

    Expired --> [*]: Удален из БД
```

---

## 2.5 Использование HTTP-only cookie для refresh token

**Архитектура хранения:**

**Описание архитектуры хранения:**
Схема демонстрирует преимущества использования HTTP-only cookie для хранения refresh token по сравнению с localStorage. В браузере доступны два варианта хранения: localStorage (уязвим к XSS-атакам) и HTTP-only cookie (защищён от XSS, так как недоступен через JavaScript). Серверная часть хранит хэш refresh token в PostgreSQL для проверки валидности и добавляет его в Redis Blacklist при отзыве. Атакующий может украсть токен из localStorage, но не из HTTP-only cookie.

```mermaid
flowchart LR
    subgraph Client["Браузер (Клиент)"]
        LS[localStorage<br/><small>XSS уязвим</small>]
        HT[HTTP-only cookie<br/><small>XSS защищён</small>]
    end

    subgraph Server["Сервер"]
        DB[(PostgreSQL<br/>refresh_tokens)]
        RB[Redis<br/>Blacklist]
    end

    LS -.->|Украсть токен| Attacker["Атакующий"]
    HT -.->|Недоступен| Attacker

    DB -->|Хэш токена| RB
```

### Сравнение хранения токенов

| Токен | Место хранения | Доступ | Безопасность |
|-------|----------------|--------|--------------|
| Access token | Response body (JSON) | JS-доступ | XSS уязвим |
| Refresh token | HTTP-only cookie | Недоступен JS | XSS защищён |

### Настройка cookie:

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

### Как работает flow:

**Описание flow:**
Flow работы refresh token начинается при входе пользователя: сервер генерирует токены и устанавливает HTTP-only cookie с refresh token. При последующих вызовах `/refresh` браузер автоматически отправляет cookie, сервер проверяет его валидность и выдаёт новые токены, обновляя cookie. Важно: для отправки cookie в запросах необходимо использовать `credentials: 'include'` в fetch API.

```mermaid
sequenceDiagram
    participant C as Клиент
    participant S as Сервер

    C->>S: POST /login
    S->>S: Generate tokens
    S->>C: 200 OK
    S-->>C: Set-Cookie: refresh_token=...
    note right of C: Cookie хранится<br/>автоматически

    C->>S: POST /refresh
    note right of C: Cookie отправляется<br/>автоматически
    S->>S: Check cookie + DB
    S->>S: Generate new tokens
    S->>C: 200 OK
    S-->>C: Set-Cookie: refresh_token=... (new)
```

### Преимущества HTTP-only cookie:

- **XSS защита:** JavaScript не может прочитать или украсть refresh token
- **Автоматическая отправка:** Браузер сам добавляет cookie к запросам (не нужно хранить в localStorage/Redux)
- **SameSite protection:** `SameSite=Strict` предотвращает отправку cookie при cross-site запросах

### Ограничения:

- **Domain ограничение:** Cookie отправляется только на `api.mystore.com` (или поддомены с `domain=.mystore.com`)
- **HTTPS рекомендация:** `Secure` флаг требует HTTPS в production
- **CSRF защита:** SameSite=Strict предотвращает большинство CSRF атак, но для критичных операций может потребоваться дополнительная защита (CSRF token)

### Клиентская логика:

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

---

## 2.6 Пример JWT токена

### Структура JWT

JWT состоит из трёх частей, разделённых точкой: `header.payload.signature`

#### Пример (декодированный):

```
[HEADER]
{
  "alg": "RS256",
  "typ": "JWT"
}

[PAYLOAD]
{
  "iss": "mystore-auth-service",
  "sub": "01hv...",
  "iat": 1726989600,
  "exp": 1726991400,
  "jti": "01hv..."
}

[SIGNATURE]
RS256(SIGNING-KEY, [HEADER].[PAYLOAD])
```

#### Полный токен (base64url encoded):

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJteXN0b3JlLWF1dGgtc2VydmljZSIsInN1YiI6IjAxaHYuLi4iLCJpYXQiOjE3MjY5ODk2MDAsImV4cCI6MTcyNjk5MTQwMCwianRpIjoiMDFodi4uLiJ9.SIGNATURE_HERE
```

### Декодирование payload (Node.js):

```javascript
function decodeJwtPayload(token) {
  const base64Url = token.split('.')[1];
  const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/');
  const payload = JSON.parse(atob(base64));
  return payload;
}

// Использование
const payload = decodeJwtPayload(accessToken);
console.log(payload.sub);  // user id
console.log(payload.exp);  // expiration timestamp
```
