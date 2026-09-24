# 1. Архитектура

```
┌─────────────┐
│   Browser   │
│  (Client)   │
└──────┬──────┘
       │ HTTPS
       │
┌──────▼────────────────────────────────────┐
│             API Gateway / Load Balancer   │
│             - Rate limiting (Redis)       │
│             - SSL termination             │
└──────┬────────────────────────────────────┘
       │
       │ HTTP
       │
┌──────▼────────────────────────────────────┐
│         Application Server                │
│  ┌─────────────────────────────────────┐  │
│  │  /v1/auth/* Routes                  │  │
│  │  - register, login, verify          │  │
│  │  - refresh, logout, me              │  │
│  │  - resend-verification              │  │
│  │  - forgot-password, reset-password  │  │
│  └─────────────────────────────────────┘  │
│  ┌─────────────────────────────────────┐  │
│  │  Auth Service                       │  │
│  │  - JWT generation/verification      │  │
│  │  - Password hashing (argon2id)      │  │
│  │  - Token management                 │  │
│  │  - Redis client (rate limiting)     │  │
│  │  - SMTP client (email sending)      │  │
│  └─────────────────────────────────────┘  │
└──────┬────────────────────────────────────┘
       │
       │ PostgreSQL                  Redis               SMTP Server
       │                 │            │                  │
┌──────▼────────────────▼────────────▼──────────────────▼────┐
│                   Data Layer                               │
│  ┌──────────────┐  ┌─────────────┐  ┌──────────────────┐  │
│  │  users       │  │  rate_limits│  │  Email Queue     │  │
│  │  tokens      │  │  sessions   │  │  SMTP Client     │  │
│  │  login_attempts│ │  locks      │  │  Templates       │  │
│  └──────────────┘  └─────────────┘  └──────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

---

## 1.1 Data Flow Diagrams / Диаграмма потоков данных

### Регистрация пользователя

```
┌──────────┐    1. POST /register    ┌──────────────────┐
│  Client  │ ──────────────────────> │   API Gateway    │
└──────────┘                         └────────┬─────────┘
                                               │
                         2. Rate limit check   │
                                         ┌─────▼───────┐
                                         │   Redis       │
                                         │   (check)     │
                                         └───────────────┘
                                               │
                3. Validate & Hash password    │
                                    ┌──────────▼──────────┐
                                    │   Auth Service      │
                                    │   - Validate input  │
                                    │   - Argon2id hash   │
                                    └──────────┬──────────┘
                                               │
                     4. Store user + token     │
                                    ┌──────────▼──────────┐
                                    │   PostgreSQL        │
                                    │   - users table     │
                                    │   - tokens table    │
                                    └──────────┬──────────┘
                                               │
                  5. Send verification email   │
                                    ┌──────────▼──────────┐
                                    │   SMTP Client       │
                                    │   - Email template  │
                                    │   - Queue delivery  │
                                    └─────────────────────┘
                                               │
                     6. Return success         │
                                    ┌──────────▼──────────┐
                                    │   Client response   │
                                    └─────────────────────┘
```

### Аутентификация

```
┌──────────┐    1. POST /login       ┌──────────────────┐
│  Client  │ ──────────────────────> │   API Gateway    │
└──────────┘                         └────────┬─────────┘
                                               │
                         2. Rate limit check   │
                                    ┌──────────▼──────────┐
                                    │   Redis             │
                                    │   - IP counter      │
                                    └─────────────────────┘
                                               │
              3. Find user + verify password   │
                                    ┌──────────▼──────────┐
                                    │   Auth Service      │
                                    │   - DB query        │
                                    │   - Argon2id check  │
                                    └──────────┬──────────┘
                                               │
               4. Generate tokens + store      │
                                    ┌──────────▼──────────┐
                                    │   Redis             │
                                    │   - Store session   │
                                    └─────────────────────┘
                                               │
                    5. Return tokens           │
                                    ┌──────────▼──────────┐
                                    │   Client response   │
                                    └─────────────────────┘
```

### Обновление токена

```
┌──────────┐    1. POST /refresh     ┌──────────────────┐
│  Client  │ ──────────────────────> │   API Gateway    │
└──────────┘                         └────────┬─────────┘
                                               │
                         2. Rate limit check   │
                                    ┌──────────▼──────────┐
                                    │   Redis             │
                                    └─────────────────────┘
                                               │
            3. Validate refresh token          │
                                    ┌──────────▼──────────┐
                                    │   Auth Service      │
                                    │   - Check Redis     │
                                    │   - Check DB        │
                                    └──────────┬──────────┘
                                               │
        4. Invalidate old + Generate new       │
                                    ┌──────────▼──────────┐
                                    │   Redis             │
                                    │   - Store new       │
                                    └─────────────────────┘
                                               │
                  5. Return new tokens         │
                                    ┌──────────▼──────────┐
                                    │   Client response   │
                                    └─────────────────────┘
```

### Выход из системы

```
┌──────────┐    1. POST /logout      ┌──────────────────┐
│  Client  │ ──────────────────────> │   API Gateway    │
└──────────┘                         └────────┬─────────┘
                                               │
                         2. Rate limit check   │
                                    ┌──────────▼──────────┐
                                    │   Redis             │
                                    └─────────────────────┘
                                               │
          3. Revoke refresh token              │
                                    ┌──────────▼──────────┐
                                    │   Auth Service      │
                                    │   - Mark revoked    │
                                    │   - Store in Redis  │
                                    └──────────┬──────────┘
                                               │
                  4. Return success            │
                                    ┌──────────▼──────────┐
                                    │   Client response   │
                                    └─────────────────────┘
```
