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
            G->>C: Set-Cookie: refresh_token=...
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
    note right of C: refresh_token in HTTP-only cookie
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
                    A->>PG: INSERT INTO refresh_tokens (parent_token_id = old_id)
                    G-->>C: 200 OK
                    note right of C: {"access_token": "..."}
                    G->>C: Set-Cookie: refresh_token=...
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
    note right of C: access_token in Authorization header<br/>refresh_token in cookie
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
                    A->>PG: UPDATE refresh_tokens SET revoked = TRUE, revoked_at = NOW()
                    G->>C: Set-Cookie: refresh_token=; Max-Age=0
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

---

## 1.3 Компоненты системы

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
