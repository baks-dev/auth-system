# 9. Чеклист развёртывания

## База данных

- [ ] Создать таблицы (users, email_verification_tokens, refresh_tokens, login_attempts)
- [ ] Создать индексы
- [ ] Настроить резервное копирование

## Backend

- [ ] Настроить переменные окружения
- [ ] Настроить rate limiting (Redis)
- [ ] Настроить email сервер
- [ ] Настроить логирование
- [ ] Настроить мониторинг (метрики)

## Безопасность

- [ ] SSL/TLS на load balancer
- [ ] HTTP-only cookie для refresh tokens
- [ ] CORS настроен правильно
- [ ] XSS защита (см. раздел 5.6 Заголовки безопасности)
- [ ] SQL injection защита (prepared statements)