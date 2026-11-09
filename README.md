# Kittygram — практическая работа 4

Приложение для регистрации пользователей и создания карточек котиков с фотографиями. Исходный учебный проект: https://github.com/yandex-praktikum/kittygram_final.

## Запуск на Mac

Открыть Docker Desktop и дождаться запуска. В терминале:

```bash
cd /Users/mandarin/Dev/VUZ_Projects/Back/pz4/kittygram_final
docker compose up -d --build
docker compose ps -a
```

Открыть http://localhost:9000 и зарегистрироваться через сайт. Миграции и сбор статики выполняются автоматически. Фронтенд один раз копирует собранные файлы и завершается с кодом 0 — это нормально. Backend и db должны быть healthy, gateway — running.

Локальный `.env` уже создан. Он содержит случайные ключ и пароль базы; файл исключён из Git. После клонирования на другой компьютер:

```bash
cp .env.example .env
```

В новом `.env` заменить POSTGRES_PASSWORD и SECRET_KEY своими случайными значениями. Пароль уже созданной базы меняется отдельно средствами PostgreSQL.

Порт задаётся APP_PORT в `.env`. При смене порта обновить также CSRF_TRUSTED_ORIGINS. Админка: http://localhost:9000/admin/. Учётную запись администратора при необходимости создать:

```bash
docker compose exec backend python manage.py createsuperuser
```

## Проверки

```bash
docker compose config --quiet
docker compose exec backend ruff check .
docker compose exec backend python manage.py check
docker compose exec backend python manage.py makemigrations --check --dry-run
docker compose exec backend python manage.py test cats --verbosity 2
docker compose run --rm --no-deps -e CI=true frontend npm test -- --watchAll=false --runInBand
```

API-тесты Django создают отдельную тестовую БД и удаляют её после проверки. Фронтенд запускает исходный Jest-тест.

Локальные проверки файлов учебного проекта через pytest:

```bash
docker compose run --rm --no-deps -v "$PWD:/repo:ro" -w /repo backend python -m pytest tests/test_files.py -k 'not deploy_info and not edu_provisioning'
```

Это проверки структуры файлов. Полный `pytest` в исходном репозитории дополнительно проверяет публичный сервер Taski, Kittygram, учебный сервер и образы Docker Hub. Для него требуются реальные адреса и аккаунты в `tests.yml`; пример — `tests.yml.example`. Эти внешние проверки не подтверждаются локальным запуском.

## Остановка и повторный запуск

```bash
docker compose down
docker compose up -d
```

Данные сохраняются в pg_data, фотографии — в media, статика — в static. Параметр `-v` у down удаляет хранилища вместе с данными.

Диагностика:

```bash
docker compose logs --tail=100 backend gateway db
```

## Устройство проекта

- db: PostgreSQL 13 по методичке; порт базы не опубликован на Mac.
- backend: Django и Gunicorn, соединение с БД через переменные окружения.
- frontend: React, сборка в общий volume static.
- gateway: Nginx, проксирует /api/ и /admin/, раздаёт фронтенд, статику админки и фотографии.

Сайт работает в корне http://localhost:9000. /kittygram перенаправляет в корень. Совместное размещение с Taski на удалённом сервере требует отдельной настройки внешнего Nginx.

## GitHub Actions и Docker Hub

Workflow `.github/workflows/main.yml` проверяет backend через Ruff и Django, запускает тест фронтенда, затем публикует три образа и отправляет уведомление Telegram. `kittygram_workflow.yml` — его точная копия для проверки по методичке.

Перед push в main добавить в GitHub Settings → Secrets and variables → Actions:

| Тип | Имя | Значение |
|---|---|---|
| Variable | DOCKERHUB_USERNAME | Логин Docker Hub |
| Secret | DOCKERHUB_TOKEN | Токен Docker Hub с правом записи |
| Secret | TELEGRAM_TOKEN | Токен Telegram-бота |
| Secret | TELEGRAM_TO | ID чата, в котором боту уже отправлено сообщение |

На pull request запускаются только тесты. Публикация разрешена для main после успешных тестов. Образы получают теги latest и SHA коммита, собираются для amd64 и arm64.

Для запуска опубликованных образов настроить DOCKERHUB_USERNAME в `.env`:

```bash
docker compose -f docker-compose.production.yml pull
docker compose -f docker-compose.production.yml up -d
```

Локальная и опубликованная конфигурации используют разные имена Compose-проектов и разные хранилища. Остановить локальную перед запуском опубликованной, чтобы освободить порт. Данные между ними автоматически не копируются.

## Коммит и отправка

После проверки пользователь самостоятельно выполняет:

```bash
git status --short
git add README.md backend frontend nginx docker-compose.yml docker-compose.production.yml .env.example .github kittygram_workflow.yml ruff.toml tests.yml.example "ПОРЯДОК СДАЧИ.md"
git diff --cached --stat
git commit -m "Configure Kittygram Docker and CI workflow"
git push origin main
```

Проверить состав коммита перед отправкой. `.env`, пароли и токены не включать. Настоящий запуск Actions и публикация Docker Hub проверяются после настройки аккаунтов и push.

Сценарий демонстрации преподавателю: [ПОРЯДОК СДАЧИ.md](ПОРЯДОК%20СДАЧИ.md).
