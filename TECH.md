# Техническая спецификация: Система регистрации и авторизации

| Файл                         | Описание                                                      |
| ---------------------------- | ------------------------------------------------------------- |
| [README.md](./README.md)     | Главная страница спецификации                                 |
| [PRODUCT.md](./PRODUCT.md)   | Продуктовая спецификация — что делается, для кого             |
| [TECH.md](./TECH.md)         | Техническая спецификация — архитектура, API, БД, безопасность |
| [GLOSSARY.md](./GLOSSARY.md) | Глоссарий терминов                                            |

---

## Оглавление

1. [Архитектура](./01-architecture.md)
   - [Data Flow Diagrams](./01-architecture.md#11-data-flow-diagrams--диаграмма-потоков-данных)

2. [Структура JWT](./02-jwt.md)
   - [Токен доступа](./02-jwt.md#21-токен-доступа)
   - [Проверка iss](./02-jwt.md#проверка-iss-при-валидации-access-token)
   - [Обработка истекших токенов](./02-jwt.md#23-обработка-истекших-токенов-доступа)
   - [Токен обновления](./02-jwt.md#24-токен-обновления)
   - [HTTP-only cookie](./02-jwt.md#25-использование-http-only-cookie-для-refresh-token)

3. [Схема базы данных](./03-database.md)
   - [Таблица users](./03-database.md#31-таблица-users)
   - [email_verification_tokens](./03-database.md#32-таблица-email_verification_tokens)
   - [refresh_tokens](./03-database.md#33-таблица-refresh_tokens)
   - [reset_password_tokens](./03-database.md#34-таблица-reset_password_tokens)
   - [login_attempts](./03-database.md#35-таблица-login_attempts)
   - [Redis ключи](./03-database.md#36-ключи-redis-rate-limiting-session-management)

4. [API Reference](./04-api-reference.md)
   - [POST /v1/auth/register](./04-api-reference.md#41-post-v1authregister)
   - [POST /v1/auth/verify](./04-api-reference.md#42-post-v1authverify)
   - [POST /v1/auth/login](./04-api-reference.md#43-post-v1authlogin)
   - [POST /v1/auth/refresh](./04-api-reference.md#44-post-v1authrefresh)
   - [POST /v1/auth/logout](./04-api-reference.md#45-post-v1authlogout)
   - [GET /v1/auth/me](./04-api-reference.md#46-get-v1authme)
   - [POST /v1/auth/resend-verification](./04-api-reference.md#47-post-v1authresend-verification)
   - [POST /v1/auth/forgot-password](./04-api-reference.md#48-post-v1authforgot-password)
   - [POST /v1/auth/reset-password](./04-api-reference.md#49-post-v1authreset-password)
   - [POST /v1/auth/unsubscribe](./04-api-reference.md#410-post-v1authunsubscribe)
   - [PUT /v1/auth/change-password](./04-api-reference.md#411-put-v1authchange-password)
   - [DELETE /v1/auth/me](./04-api-reference.md#412-delete-v1authme)

5. [Безопасность](./05-security.md)
   - [Rate limiting](./05-security.md#51-ограничение-частоты-запросов)
   - [Пароли](./05-security.md#52-безопасность-паролей)
   - [Токены](./05-security.md#53-безопасность-токенов)
   - [Email](./05-security.md#54-безопасность-email)
   - [Защита от перечисления](./05-security.md#541-защита-от-перечисления-аккаунтов)
   - [CSRF](./05-security.md#55-защита-от-csrf)
   - [Заголовки](./05-security.md#56-заголовки-безопасности)
   - [Модель угроз](./05-security.md#57-модель-угроз)
   - [Чеклист](./05-security.md#510-чеклист-реализации)

6. [Email интеграция](./06-email.md)
   - [SMTP клиент](./06-email.md#61-клиент-smtp)
   - [Шаблоны](./06-email.md#62-шаблон-подтверждение-регистрации)
   - [Очередь email](./06-email.md#64-очередь-email-для-высокой-нагрузки)

7. [Логирование](./07-logging.md)
   - [Схема событий](./07-logging.md#71-схема-событий)
   - [Критические события](./07-logging.md#72-критические-события)

8. [Деплоймент](./08-deployment.md)
   - [Переменные окружения](./08-deployment.md#8-переменные-окружения)
   - [Production пример](./08-deployment.md#81-пример-конфигурации-production)

9. [Чеклист развёртывания](./09-deployment-checklist.md)

10. [Тестирование](./10-testing.md)
    - [Юнит-тесты](./10-testing.md#юнит-тесты)
    - [Интеграционные](./10-testing.md#интеграционные-тесты)
    - [E2E тесты](./10-testing.md#e2e-тесты)
