# LMS — 35.2 CI/CD и GitHub Actions

Платформа онлайн-обучения на Django REST Framework. Проект запускается в Docker (Gunicorn + Nginx, PostgreSQL, Redis, Celery, celery-beat). При каждом push GitHub Actions прогоняет тесты, линтер, собирает Docker-образ и деплоит его на удалённый сервер.

## Состав

| Сервис | Назначение | Порт |
|---|---|---|
| `nginx` | принимает запросы, отдаёт статику и медиа, проксирует в Gunicorn | `80` наружу |
| `web` | Django + Gunicorn | `8000` только внутри сети |
| `db` | PostgreSQL 16 | `5432` только внутри сети |
| `redis` | брокер Celery | `6379` только внутри сети |
| `celery` | воркер фоновых задач | — |
| `celery-beat` | периодические задачи | — |

У всех контейнеров `restart: always`, поэтому после падения или перезагрузки сервера они поднимаются сами.

## Локальный запуск

```
cp .env.template .env
docker compose up -d --build
docker compose exec web python manage.py createsuperuser
```

Приложение: http://localhost/ , документация API: http://localhost/api/docs/

Без Docker:

```
python -m venv venv
venv\Scripts\activate            (Linux/macOS: source venv/bin/activate)
pip install -r requirements.txt
cp .env.template .env            (указать POSTGRES_HOST=localhost)
python manage.py migrate
python manage.py runserver
```

Тесты и линтер:

```
python manage.py test
flake8 .
```

## CI/CD

Файл `.github/workflows/ci.yml`. Запускается при push в `main`, `develop`, `feature/**` и при pull request в `main` и `develop`.

1. **test** — поднимает PostgreSQL и Redis как сервисы, ставит зависимости, запускает `python manage.py test`. Если хоть один тест упал, pipeline останавливается.
2. **lint** — `flake8 .` (настройки в `.flake8`).
3. **build** — собирает Docker-образ, проверяет его командой `manage.py check`, сохраняет образ как артефакт.
4. **deploy** — только для push: копирует образ, `docker-compose.yml` и конфиг Nginx на сервер по SSH, создаёт там `.env` из GitHub Secrets, загружает образ и перезапускает контейнеры, затем проверяет, что приложение отвечает.

Каждый этап выполняется только после успешного предыдущего (`needs`).

### GitHub Secrets

Settings → Secrets and variables → Actions → New repository secret:

| Secret | Что указать |
|---|---|
| `SSH_HOST` | публичный IP сервера |
| `SSH_USER` | пользователь на сервере |
| `SSH_KEY` | приватный SSH-ключ целиком (от `-----BEGIN` до `-----END`) |
| `SECRET_KEY` | секретный ключ Django |
| `POSTGRES_PASSWORD` | пароль базы данных |
| `STRIPE_API_KEY` | тестовый ключ Stripe |

Секреты в репозиторий не попадают: `.env` создаётся на сервере во время деплоя, файл `.env` исключён в `.gitignore`.

## Настройка сервера (Yandex Cloud, Ubuntu 22.04/24.04)

1. Создать SSH-ключ на своём компьютере:
   ```
   ssh-keygen -t ed25519 -f ~/.ssh/lms_deploy -N ""
   ```
   Публичный ключ `lms_deploy.pub` указать при создании виртуальной машины, приватный `lms_deploy` положить в секрет `SSH_KEY`.

2. Подключиться к серверу:
   ```
   ssh -i ~/.ssh/lms_deploy USER@SERVER_IP
   ```

3. Установить Docker и разрешить запуск без sudo:
   ```
   sudo apt update && sudo apt upgrade -y
   curl -fsSL https://get.docker.com | sudo sh
   sudo usermod -aG docker $USER
   sudo systemctl enable --now docker
   ```
   После этого выйти и зайти по SSH заново.

4. Закрыть лишние порты, оставить только SSH и HTTP:
   ```
   sudo ufw default deny incoming
   sudo ufw default allow outgoing
   sudo ufw allow OpenSSH
   sudo ufw allow 80/tcp
   sudo ufw enable
   ```

5. Запретить вход по паролю (только по ключу): в `/etc/ssh/sshd_config` выставить
   ```
   PasswordAuthentication no
   PermitRootLogin no
   ```
   и выполнить `sudo systemctl restart ssh`.

6. Добавить секреты в GitHub и сделать push — workflow сам развернёт проект в папке `~/lms`.

Проверка на сервере:

```
cd ~/lms
docker compose ps
docker compose logs -f web
docker compose exec web python manage.py createsuperuser
```

Приложение будет доступно по адресу `http://SERVER_IP/`, документация — `http://SERVER_IP/api/docs/`.

Автозапуск: Docker включён в systemd (`systemctl enable docker`), контейнеры имеют `restart: always`, поэтому после `sudo reboot` проект поднимается сам.
