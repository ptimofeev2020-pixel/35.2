# LMS — 34.2 Docker Compose

Платформа онлайн-обучения на Django REST Framework. Все части проекта запускаются одной командой через Docker Compose.

## Сервисы

| Сервис | Назначение | Доступ |
|---|---|---|
| `web` | Django-приложение | порт `8000` наружу |
| `db` | PostgreSQL 16 | только внутри сети (`expose 5432`), данные в томе `postgres_data` |
| `redis` | брокер Celery | только внутри сети (`expose 6379`), данные в томе `redis_data` |
| `celery` | воркер фоновых задач | без портов |
| `celery-beat` | планировщик периодических задач | без портов |

Сервисы `web`, `celery` и `celery-beat` стартуют только после того, как база и Redis прошли healthcheck.
При запуске `web` автоматически применяет миграции и собирает статику.

## Запуск

1. Установите Docker и Docker Compose.
2. Создайте файл `.env` по образцу:
   ```
   cp .env.sample .env
   ```
   В `.env` для Docker адреса сервисов указываются по именам: `POSTGRES_HOST=db`, `CELERY_BROKER_URL=redis://redis:6379/0`.
3. Соберите и запустите проект:
   ```
   docker compose up -d --build
   ```
4. Создайте суперпользователя и загрузите фикстуры:
   ```
   docker compose exec web python manage.py createsuperuser
   docker compose exec web python manage.py loaddata groups.json
   ```

Приложение: http://localhost:8000/
Документация API: http://localhost:8000/api/docs/

## Полезные команды

```
docker compose ps                 # статус сервисов
docker compose logs -f web        # логи приложения
docker compose logs -f celery     # логи воркера
docker compose exec web python manage.py test   # тесты внутри контейнера
docker compose down               # остановить
docker compose down -v            # остановить и удалить тома с данными
```

## Переменные окружения

Все настройки (секретный ключ, доступ к базе, Redis, Stripe, почта) задаются в `.env`.
Файл подключён ко всем сервисам через `env_file` и не попадает в репозиторий (`.gitignore`).
