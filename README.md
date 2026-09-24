# Система регистрации и авторизации

Добро пожаловать в документацию системы регистрации и авторизации MyStore.

## Документация

| Файл | Описание |
|------|----------|
| **[PRODUCT.md](./PRODUCT.md)** | Продуктовая спецификация — что делается, для кого |
| **[TECH.md](./TECH.md)** | Оглавление технической спецификации |
| **[01-architecture.md](./01-architecture.md)** | Архитектура и Data Flow Diagrams |
| **[02-jwt.md](./02-jwt.md)** | Структура JWT токенов |
| **[03-database.md](./03-database.md)** | Схема базы данных |
| **[04-api-reference.md](./04-api-reference.md)** | Полная спецификация API endpoints |
| **[05-security.md](./05-security.md)** | Реализация безопасности |
| **[06-email.md](./06-email.md)** | Интеграция email |
| **[07-logging.md](./07-logging.md)** | Логирование |
| **[08-deployment.md](./08-deployment.md)** | Переменные окружения и деплоймент |
| **[09-deployment-checklist.md](./09-deployment-checklist.md)** | Чеклист развёртывания |
| **[10-testing.md](./10-testing.md)** | Стратегия тестирования |
| **[GLOSSARY.md](./GLOSSARY.md)** | Глоссарий терминов |

## Ссылки

- [OpenAPI 3.0 спецификация](./references/openapi.yaml) — для генерации SDK и документации

## Структура проекта

```
specs/auth-system/
├── README.md                  # Этот файл
├── PRODUCT.md                 # Продуктовая спецификация
├── TECH.md                    # Оглавление технической спецификации
├── GLOSSARY.md                # Глоссарий
├── 01-architecture.md         # Архитектура
├── 02-jwt.md                  # JWT токены
├── 03-database.md             # Схема БД
├── 04-api-reference.md        # API Reference
├── 05-security.md             # Безопасность
├── 06-email.md                # Email интеграция
├── 07-logging.md              # Логирование
├── 08-deployment.md           # Деплоймент
├── 09-deployment-checklist.md # Чеклист
├── 10-testing.md              # Тестирование
└── references/
    └── openapi.yaml           # OpenAPI 3.0 спецификация
```

## Контакты

Для вопросов по документации обращайтесь к команде backend или technical writing.
