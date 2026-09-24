# Система регистрации и авторизации

Добро пожаловать в документацию системы регистрации и авторизации MyStore.

## Документация

| Файл                         | Описание                                                      |
| ---------------------------- | ------------------------------------------------------------- |
| **PRODUCT.md**               | Продуктовая спецификация — что делается, для кого             |
| **TECH.md**                  | Оглавление технической спецификации                           |
| **TECH.md → 01-architecture.md** | Архитектура и Data Flow Diagrams                          |
| **TECH.md → 02-jwt.md**      | Структура JWT токенов                                         |
| **TECH.md → 03-database.md** | Схема базы данных                                             |
| **TECH.md → 04-api-reference.md** | Полная спецификация API endpoints                       |
| **TECH.md → 05-security.md** | Реализация безопасности                                      |
| **TECH.md → 06-email.md**    | Интеграция email                                             |
| **TECH.md → 07-logging.md**  | Логирование                                                  |
| **TECH.md → 08-deployment.md** | Переменные окружения и деплоймент                        |
| **TECH.md → 09-deployment-checklist.md** | Чеклист развёртывания                            |
| **TECH.md → 10-testing.md**  | Стратегия тестирования                                        |
| **GLOSSARY.md**              | Глоссарий терминов                                           |

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
