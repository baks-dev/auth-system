# 10. Стратегия тестирования

## Юнит-тесты

- Целевое покрытие: **≥85%** для критических модулей (auth, token handling, password hashing)
- Целевое покрытие: **≥70%** для остальных модулей
- Хэширование паролей (argon2id)
- Подпись/проверка JWT (RS256)
- Генерация токенов электронной почты
- Ограничение запросов (rate limiting)
- JWT ID (jti) генерация и уникальность
- Revocation check по jti в Redis

## Интеграционные тесты

- Полный процесс регистрации
- Процесс подтверждения электронной почты
- Процесс входа в систему
- Обновления токена
- Процесс выхода из системы
- Ограничение запросов (rate limiting)
- Auto-refresh access token при истечении (safe methods)
- Revocation по jti (logout и истечение срока)
- Edge case: Refresh token истек в момент запроса refresh
- Edge case: Access token истек в момент refresh (должен вернуть 401)
- Edge case: Использованный refresh token (reuse attack)

## E2E тесты

- Фреймворки: Cypress / Playwright
- Проверка UI (если есть)
- Security Scenarios:
  - XSS: Ввод вредоносного JS в поля name/email/password, проверка экранирования
  - SQLi: Injection в email (e.g. `' OR 1=1 --`)
  - Header injection в User-Agent
  - CSRF: Проверка отсутствия токена в заголовках
  - Token leakage: Проверка, что токены не попадают в console.log и network logs
  - Session fixation: Проверка смены jti при refresh