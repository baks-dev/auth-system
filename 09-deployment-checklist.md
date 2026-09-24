# 9. Чеклист развёртывания

## Критичные (обязательные к выполнению перед запуском)

### База данных

- [ ] **Создать таблицы** (см. [03-database.md](./03-database.md)):
  - `users` — основная таблица пользователей
  - `email_verification_tokens` — токены верификации
  - `refresh_tokens` — токены обновления
  - `login_attempts` — попытки входа
  - `audit_logs` — аудит логи (см. [07-logging.md](./07-logging.md))
- [ ] **Создать индексы**:
  - `idx_users_email` на `users.email`
  - `idx_users_password_reset_token` на `users.password_reset_token`
  - `idx_refresh_tokens_user_id` на `refresh_tokens.user_id`
  - `idx_refresh_tokens_jti` на `refresh_tokens.jti`
  - `idx_email_verification_tokens_user_id` на `email_verification_tokens.user_id`
  - `idx_login_attempts_ip` на `login_attempts.ip_address`
- [ ] **Настроить резервное копирование** (PgBouncer + WAL-E или аналог)
- [ ] **Проверить подключение** к PostgreSQL из контейнера приложения

### Backend

- [ ] **Проверить переменные окружения** (см. [08-deployment.md](./08-deployment.md)):
  - Все обязательные переменные установлены
  - `NODE_ENV` задан корректно
  - PEM ключи закодированы в base64
- [ ] **Настроить rate limiting** (см. [05-security.md](./05-security.md)):
  - Redis подключен и доступен
  - `RATE_LIMIT_WINDOW_MS` = 60000
  - `RATE_LIMIT_MAX` = 100 (или как в документации)
- [ ] **Настроить email сервер** (см. [06-email.md](./06-email.md)):
  - SMTP параметры верны
  - Проверить подключение к SMTP серверу
  - Проверить отправку тестового письма
- [ ] **Настроить логирование** (см. [07-logging.md](./07-logging.md)):
  - `LOG_LEVEL` установлен (production: info, development: debug)
  - Логи пишутся в stdout/stderr
- [ ] **Включить валидацию конфигурации** при старте приложения

### Безопасность

- [ ] **SSL/TLS на load balancer** — завершение TLS на уровне ingress
- [ ] **HTTP-only cookie для refresh tokens** — проверить настройки cookie в коде
- [ ] **CORS настроен правильно** (см. [05-security.md](./05-security.md)):
  - `origin` ограничен доверенными доменами
  - `credentials: true` для cookie
- [ ] **XSS защита** — заголовки безопасности (см. [05-security.md](./05-security.md) раздел 5.6)
- [ ] **SQL injection защита** — использовать prepared statements во всех запросах

---

## Важные (рекомендуется перед запуском)

### Мониторинг и алертинг

- [ ] **Настроить метрики** (Prometheus/Grafana):
  - Количество запросов
  - Время ответа
  - Количество активных пользователей
  - Количество неудачных попыток входа
- [ ] **Настроить алертинг**:
  - Алерт при >10 неудачных входов за 5 минут
  - Алерт при падении сервиса
  - Алерт при ошибках в логах
- [ ] **Настроить логирование в SIEM** (Sentry/Datadog)

### Инфраструктура

- [ ] **Проверить подключение к Redis** — `redis-cli ping`
- [ ] **Настроить health check endpoint** (`GET /health`)
- [ ] **Настроить readiness probe** — проверка зависимостей (DB, Redis)

---

## Опциональные (после запуска)

### Дополнительные настройки

- [ ] Настроить CDN для статических файлов
- [ ] Настроить кэширование в браузере
- [ ] Настроить gzip/brotli сжатие
- [ ] Настроить отдельный домен для noreply@ email
- [ ] Настроить SPF, DKIM, DMARC для домена email

### Тестирование

- [ ] **Проверить регистрацию пользователя** — полный flow
- [ ] **Проверить верификацию email** — токен работает
- [ ] **Проверить вход** — access и refresh токены выдаются
- [ ] **Проверить авто-обновление токена** — при истечении <5s
- [ ] **Проверить сброс пароля** — токен работает
- [ ] **Проверить rate limiting** — превышение лимита возвращает 429
- [ ] **Проверить логи** — события записываются в audit_logs

---

## Проверка после развёртывания

1. Запустить e2e тесты:
2. Проверить логи на ошибки:
   ```bash
   kubectl logs -f deployment/auth-service
   # или
   docker logs -f auth-service
   ```

3. Проверить метрики в Grafana/Prometheus
4. Выполнить smoke test — вызвать `/health` и `/v1/auth/status`
5. Отправить тестовое письмо на проверку спам-фильтров (Mail-Tester)
