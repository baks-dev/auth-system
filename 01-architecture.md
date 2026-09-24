# 1. Архитектура

## 1.1 Общая архитектура системы

Следующая схема показывает общую структуру системы регистрации и авторизации, включая взаимодействие между компонентами:

```mermaid
flowchart TD
    subgraph Client["Клиент"]
        Browser[Браузер]
    end

    subgraph Gateway["API Gateway / Load Balancer"]
        RateLimit[Rate Limiting<br/><small>Redis</small>]
        SSL[SSL Termination]
    end

    subgraph App["Application Server"]
        Routes["/v1/auth/* Routes"]
        AuthService[Auth Service<br/><small>JWT, Argon2id, Token management</small>]
    end

    subgraph Data["Data Layer"]
        PG[(PostgreSQL)]
        Redis[(Redis)]
        SMTP[SMTP Server]
    end

    Browser -->|HTTPS| Gateway
    Gateway -->|HTTP| App
    App -->|PostgreSQL| PG
    App -->|Redis| Redis
    App -->|SMTP| SMTP

    style Client fill:#e1f5ff
    style Gateway fill:#fff3e0
    style App fill:#e8f5e9
    style Data fill:#fce4ec
```

---

## 1.2 Data Flow Diagrams / Диаграмма потоков данных

### Регистрация пользователя

Следующая диаграмма показывает пошаговый процесс регистрации нового пользователя:

```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL
    participant SMTP as SMTP

    C->>G: POST /v1/auth/register
    G->>R: check rate limit (IP:email)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>A: validate input (email, password, name)
        A->>A: hash password (Argon2id)
        A->>PG: INSERT INTO users
        A->>PG: INSERT INTO email_verification_tokens
        PG-->>A: user_id
        A->>SMTP: send verification email
        SMTP-->>A: queued
        G-->>C: 201 Created
        note right of C: {"message": "...", "email": "..."}
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

### Аутентификация

Следующая диаграмма показывает процесс входа пользователя в систему:

```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL

    C->>G: POST /v1/auth/login
    G->>R: check rate limit (IP:email)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>PG: SELECT * FROM users WHERE email = ?
        PG-->>A: user record / null
        alt User found
            A->>A: verify password (Argon2id)
            A->>R: store session
            A->>A: generate access token (30min)
            A->>A: generate refresh token (7 days)
            A->>PG: INSERT INTO refresh_tokens
            G-->>C: 200 OK
            note right of C: {"access_token": "..."}
            G->>C: Set-Cookie header
        else User not found / wrong password
            A->>A: timing-safe dummy check
            G-->>C: 401 Unauthorized
        end
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

### Обновление токена

Следующая диаграмма показывает процесс обновления access token через refresh token:

```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL

    C->>G: POST /v1/auth/refresh
    note right of C: refresh_token в cookie
    G->>R: check rate limit (IP)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>R: GET revoked_tokens:{hash}
        alt Not revoked
            A->>PG: SELECT * FROM refresh_tokens WHERE jti = ?
            PG-->>A: token record
            alt Token valid
                A->>A: verify signature (RS256)
                A->>A: check expires_at
                A->>A: check ip_address
                A->>A: check user_agent_hash
                alt All checks passed
                    A->>R: SET revoked_tokens:{old_hash}
                    A->>PG: UPDATE refresh_tokens SET revoked = TRUE
                    A->>A: generate new access token
                    A->>A: generate new refresh token
                    A->>PG: INSERT INTO refresh_tokens
                    G-->>C: 200 OK
                    note right of C: {"access_token": "..."}
                    G->>C: Set-Cookie header
                else Token binding failed
                    G-->>C: 401 Unauthorized
                end
            else Token expired/invalid
                G-->>C: 401 Unauthorized
            end
        else Token revoked
            G-->>C: 401 Unauthorized
        end
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

### Выход из системы

Следующая диаграмма показывает процесс выхода пользователя из системы (инвалидация текущей сессии):

```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL

    C->>G: POST /v1/auth/logout
    note right of C: access_token в заголовке<br/>refresh_token в cookie
    G->>R: check rate limit (IP)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>R: GET revoked_tokens:{hash}
        alt Not revoked
            A->>PG: SELECT * FROM refresh_tokens WHERE jti = ?
            PG-->>A: token record
            alt Token valid
                A->>A: verify signature (RS256)
                alt Signature valid
                    A->>R: SET revoked_tokens:{hash}
                    A->>PG: UPDATE refresh_tokens SET revoked = TRUE
                    G->>C: Clear refresh cookie
                    G-->>C: 200 OK
                    note right of C: {"message": "..."}
                else Signature invalid
                    G-->>C: 401 Unauthorized
                end
            else Token expired
                G-->>C: 401 Unauthorized
            end
        else Token already revoked
            G-->>C: 200 OK
        end
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

### Подтверждение email

Следующая диаграмма показывает процесс подтверждения email пользователя:

```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL

    C->>G: POST /v1/auth/verify
    G->>R: check rate limit (IP)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>PG: SELECT * FROM email_verification_tokens WHERE token = ?
        PG-->>A: token record / null
        alt Token found
            alt Token not expired
                A->>PG: SELECT is_email_verified FROM users WHERE id = ?
                alt Email not verified
                    A->>PG: UPDATE users SET is_email_verified = TRUE, email_verified_at = NOW()
                    A->>PG: DELETE FROM email_verification_tokens WHERE id = ?
                    A->>A: generate new access token
                    A->>A: generate new refresh token
                    A->>PG: INSERT INTO refresh_tokens
                    G-->>C: 200 OK
                    note right of C: {"access_token": "...", "refresh_token": "..."}
                else Email already verified
                    G-->>C: 409 Conflict
                    note right of C: {"error": "EMAIL_ALREADY_CONFIRMED"}
                end
            else Token expired
                G-->>C: 400 Bad Request
                note right of C: {"error": "CONFIRMATION_TOKEN_INVALID"}
            end
        else Token not found
            G-->>C: 400 Bad Request
            note right of C: {"error": "CONFIRMATION_TOKEN_INVALID"}
        end
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

### Запрос сброса пароля

Следующая диаграмма показывает процесс запроса сброса пароля:

```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL
    participant SMTP as SMTP

    C->>G: POST /v1/auth/forgot-password
    G->>R: check rate limit (IP:email)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>PG: SELECT * FROM users WHERE email = ?
        PG-->>A: user record / null
        alt User found
            A->>A: generate reset token (UUIDv7)
            A->>PG: INSERT INTO reset_password_tokens
            A->>SMTP: send reset email with token
            SMTP-->>A: queued
            G-->>C: 200 OK
            note right of C: {"message": "..."}
        else User not found
            A->>A: timing-safe dummy operation
            G-->>C: 200 OK
            note right of C: {"message": "..."}
        end
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

### Сброс пароля

Следующая диаграмма показывает процесс сброса пароля пользователя:

```mermaid
sequenceDiagram
    participant C as Клиент
    participant G as API Gateway
    participant R as Redis
    participant A as Auth Service
    participant PG as PostgreSQL

    C->>G: POST /v1/auth/reset-password
    G->>R: check rate limit (IP:email)
    R-->>G: allowed / blocked
    alt Rate limit ok
        A->>PG: SELECT * FROM reset_password_tokens WHERE token = ?
        PG-->>A: token record / null
        alt Token found
            alt Token not expired
                A->>A: hash new password (Argon2id)
                A->>PG: UPDATE users SET password_hash = ?
                A->>PG: UPDATE reset_password_tokens SET used_at = NOW()
                G-->>C: 200 OK
                note right of C: {"message": "..."}
            else Token expired
                G-->>C: 400 Bad Request
                note right of C: {"error": "EXPIRED_TOKEN"}
            end
        else Token not found
            G-->>C: 400 Bad Request
            note right of C: {"error": "INVALID_TOKEN"}
        end
    else Rate limit exceeded
        G-->>C: 429 Too Many Requests
    end
```

---

## 1.3 Жизненный цикл токена

Следующая диаграмма показывает все состояния refresh token и переходы между ними:

```mermaid
stateDiagram-v2
    [*] --> Created

    Created --> Active: Token выдан<br/>login/verify/refresh

    Active --> Active: Token использован<br/>rotation (новый токен)

    Active --> Revoked: Logout вызван
    Active --> Revoked: Token скомпрометирован
    Active --> Revoked: Периодическая очистка

    Active --> Expired: Срок жизни истек<br/>(7 дней)

    Revoked --> [*]: Удален из БД<br/>(после N дней)

    Expired --> [*]: Удален из БД
```

### Описание состояний

| Состояние | Описание |
|-----------|----------|
| `Created` | Токен создан и хранится в БД, но ещё не использован |
| `Active` | Токен активен, может быть использован для refresh |
| `Revoked` | Токен инвалидирован (logout или компрометация) |
| `Expired` | Срок жизни токена истёк |

### Transition события

- **Token выдан** — при `/login`, `/verify` или `/refresh`
- **Rotation** — при каждом `/refresh` старый токен становится `Revoked`, создаётся новый `Active`
- **Logout** — текущий токен помечается как `Revoked`
- **Истечение срока** — автоматически через 7 дней

---

## 1.4 ER-диаграмма связей таблиц

Следующая диаграмма показывает связи между таблицами базы данных:

```mermaid
erDiagram
    USERS ||--o{ EMAIL_VERIFICATION_TOKENS : "has"
    USERS ||--o{ REFRESH_TOKENS : "has"
    USERS ||--o{ RESET_PASSWORD_TOKENS : "has"
    USERS ||--o{ LOGIN_ATTEMPTS : "has"

    USERS {
        uuid id PK
        varchar(255) email
        varchar(255) name
        text password_hash
        varchar(20) role
        boolean is_email_verified
        timestamp email_verified_at
        varchar(255) unsubscribe_token
        timestamp created_at
        timestamp updated_at
    }

    EMAIL_VERIFICATION_TOKENS {
        uuid id PK
        uuid user_id FK
        varchar(255) token
        timestamp expires_at
        timestamp created_at
        timestamp used_at
    }

    REFRESH_TOKENS {
        uuid id PK
        uuid user_id FK
        varchar(512) token_hash
        varchar(45) ip_address
        varchar(64) user_agent_hash
        boolean revoked
        timestamp revoked_at
        timestamp expires_at
        uuid parent_token_id FK
        timestamp created_at
    }

    RESET_PASSWORD_TOKENS {
        uuid id PK
        uuid user_id FK
        varchar(255) token
        timestamp expires_at
        timestamp created_at
        timestamp used_at
    }

    LOGIN_ATTEMPTS {
        uuid id PK
        varchar(255) email
        varchar(45) ip_address
        varchar(64) user_agent_hash
        boolean success
        timestamp failed_at
    }
```

### Описание таблиц

| Таблица | Описание |
|---------|----------|
| `users` | Основная таблица пользователей |
| `email_verification_tokens` | Токены для подтверждения email (24 часа) |
| `refresh_tokens` | Токены обновления (7 дней), с поддержкой rotation |
| `reset_password_tokens` | Токены для сброса пароля (1 час) |
| `login_attempts` | Лог попыток входа для аудита |

### Связи

- `users` -> `email_verification_tokens` — один ко многим (CASCADED DELETE)
- `users` -> `refresh_tokens` — один ко многим (CASCADED DELETE)
- `users` -> `reset_password_tokens` — один ко многим (CASCADED DELETE)
- `users` -> `login_attempts` — один ко многим (для аудита)

---

## 1.5 Компоненты системы

### Базы данных

| Компонент | Назначение |
|-----------|------------|
| PostgreSQL | Хранение пользователей, токенов, логов |
| Redis | Rate limiting, сессии, черные списки токенов |

### Основные сервисы

| Компонент | Назначение |
|-----------|------------|
| API Gateway | Rate limiting, SSL termination |
| Auth Service | JWT generation/verification, password hashing, token management |

### Внешние сервисы

| Компонент | Назначение |
|-----------|------------|
| SMTP Server | Отправка email (верификация, сброс пароля) |
