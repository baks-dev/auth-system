# 10. Стратегия тестирования

**Обзор:** Документ описывает подход к тестированию системы авторизации. Стратегия включает три уровня тестов: юнит-тесты для изолированной логики, интеграционные тесты для проверки взаимодействия с зависимостями и E2E тесты для проверки полных пользовательских сценариев.

---

## 10.1 Юнит-тесты

**Цель:** Проверка изолированной бизнес-логики без внешних зависимостей (база данных, Redis, SMTP).

**Целевое покрытие:**
- **≥85%** для критических модулей (auth, token handling, password hashing)
- **≥70%** для остальных модулей

**Что тестируется:**
- Хэширование паролей (argon2id)
- Подпись/проверка JWT (RS256)
- Генерация токенов электронной почты
- Ограничение запросов (rate limiting)
- JWT ID (jti) генерация и уникальность
- Revocation check по jti в Redis (mock)
- Валидация токенов (expired, revoked, issuer)
- Сравнение токенов (timing-safe)

**Подход:**
- Использовать моки для внешних зависимостей
- Тестировать позитивные и негативные сценарии
- Проверять граничные случаи (edge cases)

**Примеры тестовых сценариев:**

| Сценарий | Вход | Ожидаемый результат |
|----------|------|---------------------|
| Валидный JWT | Токен с валидной подписью | `isValid = true` |
| Истекший JWT | Токен с прошедшим `exp` | `isValid = false`, ошибка `TOKEN_EXPIRED` |
| Неверный issuer | `iss` не равен `mystore-auth-service` | `isValid = false`, ошибка `ACCESS_TOKEN_INVALID` |
| Revoked token | `jti` в blacklist Redis | `isValid = false`, ошибка `TOKEN_REVOKED` |
| Правильный пароль | `argon2id` хеш совпадает | `isValid = true` |
| Неверный пароль | Хеш не совпадает | `isValid = false` |

---

## 10.2 Интеграционные тесты

**Цель:** Проверка взаимодействия с внешними системами (PostgreSQL, Redis, SMTP).

**Что тестируется:**

| Сценарий | Зависимости | Описание |
|----------|-------------|----------|
| Полный процесс регистрации | PostgreSQL, SMTP | Регистрация, токен, письмо |
| Подтверждение email | PostgreSQL | Валидация токена, статус user |
| Вход в систему | PostgreSQL, Redis | Генерация токенов, cookie |
| Обновление токена | PostgreSQL, Redis | Rotation токенов |
| Выход из системы | PostgreSQL, Redis | Revocation токенов |
| Rate limiting | Redis | Блокировка после лимита |
| Auto-refresh token | PostgreSQL, Redis | Обновление при истечении <5s |
| Revocation по jti | Redis | Проверка черного списка |
| Edge: Refresh token истек | PostgreSQL | Возврат 401 |
| Edge: Access token при refresh | PostgreSQL | Возврат 401 без авто-обновления |
| Edge: Reuse атака | PostgreSQL, Redis | Возврат 401 для повторного использования |

**Подход:**
- Использовать test containers для изолированной базы данных
- Использовать in-memory Redis (например, ioredis-mock)
- Для SMTP использовать Mailtrap или аналог
- Каждый тест должен очищать данные после завершения

---

## 10.3 E2E тесты

**Цель:** Проверка полных пользовательских сценариев через HTTP API.

**Что тестируется:**

| Сценарий | API endpoints | Проверка |
|----------|---------------|----------|
| Регистрация | POST /v1/auth/register | 201 Created, email отправлено |
| Верификация | GET /v1/auth/verify | 200 OK, user.verified = true |
| Вход | POST /v1/auth/login | 200 OK, access + refresh токены |
| Refresh | POST /v1/auth/refresh | 200 OK, новые токены |
| Logout | POST /v1/auth/logout | 200 OK, токены отозваны |
| Get user | GET /v1/users/me | 200 OK, профиль пользователя |
| Сброс пароля | POST /v1/auth/reset-password | 200 OK, письмо отправлено |
| Установка пароля | POST /v1/auth/set-password | 200 OK, пароль обновлен |

**Security scenarios:**

| Тип атаки | Описание | Ожидаемый результат |
|-----------|----------|---------------------|
| XSS | Ввод `<script>alert(1)</script>` | Экранирование, 200 OK без JS |
| SQLi | `email=' OR 1=1 --` | Возврат пустого результата, 401 |
| Header injection | `\r\nSet-Cookie: ...` | Header удаляется/экранируется |
| CSRF | Cross-site запрос без токена | 403 Forbidden |
| Token leakage | Проверка console.log/network logs | Токены не попадают в логи |
| Session fixation | Проверка смены jti при refresh | `jti` изменяется |

**Подход:**
- Использовать Cypress или Playwright для browser-based тестов
- Использовать fetch/axios для API-only тестов
- Все тесты должны работать с реальным API
- Использовать test users с уникальными email

---

## 10.4 Тестовая инфраструктура

**Фикстуры:**
- Тестовые данные пользователей (с валидными хешами паролей)
- Валидные JWT токены для тестов
- Валидные refresh tokens
- Валидные email verification tokens

**Моки:**
- SMTP сервер (Mailtrap или аналог)
- Redis (in-memory или docker container)
- PostgreSQL (in-memory или docker container)

**Test containers:**
- PostgreSQL latest
- Redis latest
- Mailtrap (или аналог)

**Порядок запуска:**
1. Запустить test containers
2. Применить миграции
3. Загрузить фикстуры
4. Запустить тесты
5. Очистить данные
6. Остановить containers

---

## 10.5 CI/CD интеграция

**Pipeline этапы:**

| Этап | Команда | Условие |
|------|---------|---------|
| Install | `npm ci` | Всегда |
| Lint | `npm run lint` | Всегда |
| Unit tests | `npm run test:unit` | Всегда |
| Integration tests | `npm run test:integration` | После unit |
| E2E tests | `npm run test:e2e` | После integration |
| Coverage report | `npm run coverage` | После всех тестов |

**Quality gates:**
- Unit tests: coverage ≥ 70% (критичных модулей ≥ 85%)
- Integration tests: все проходят
- E2E tests: все проходят
- Линт: без ошибок

**Отчеты:**
- Coverage report (HTML + JSON)
- Test results (JUnit XML)
- Security scan (SonarQube/Snyk)

---

## 10.6 Security testing

**Ручное тестирование (включить в CI):**

| Тест | Описание | Инструмент |
|------|----------|------------|
| JWT签名验证 | Проверка подписи RS256 | Custom script |
| Token revocation | Проверка черного списка | Custom script |
| Rate limiting | Проверка блокировки | Custom script |
| SQL injection | Ввод `'; DROP TABLE users; --` | Custom script |
| XSS | Ввод `<script>` | Custom script |
| CSRF | Cross-site запрос | Custom script |
| Timing attacks | Проверка timing-safe | Custom script |

**Проверки:**
- Токены не хранятся в localStorage
- HTTP-only cookie для refresh tokens
- Сессия инвалидируется при logout
- Пароли не хранятся в логах
- Токены не попадают в error messages
