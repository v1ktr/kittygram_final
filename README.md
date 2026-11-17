# Kittygram

Сервис карточек котов: регистрация, список питомцев, фото и достижения. Бэкенд на Django REST Framework, фронтенд на React. Всё запускается в Docker и публикуется где угодно, в проектной работе рядом с Taski на одном домене.

Продакшен: [https://taskivktr.webhop.me/kittygram](https://taskivktr.webhop.me/kittygram)

| | |
| --- | --- |
| API | `/kittygram/api/` |
| Админка | `/kittygram/admin/` |
| Загруженные фото | `/kittygram/media/` |
| Taski на том же сервере | [https://taskivktr.webhop.me/taski](https://taskivktr.webhop.me/taski) |

Корень домена перенаправляется на Kittygram. Taski продолжает отвечать по `/taski`, а его уже собранный фронтенд по-прежнему ходит в `/api/` и `/static/`.

## Стек

- Python 3.12, Django, Django REST Framework, Djoser, Gunicorn
- PostgreSQL 13
- React (Create React App), база путей `/kittygram`
- Nginx в роли gateway
- Docker Compose и GitHub Actions

## Структура

```
kittygram_final
├── backend          Django-проект и Dockerfile образа kittygram_backend
├── frontend         React-приложение и Dockerfile образа kittygram_frontend
├── nginx            Конфигурация и Dockerfile образа kittygram_gateway
├── docker-compose.yml
├── docker-compose.production.yml
└── .github/workflows/main.yml
```

Три тома:

- `pg_data` — данные PostgreSQL
- `static` — собранный фронтенд и статика Django, её раздаёт gateway
- `media` — файлы, загруженные пользователями; том доступен backend и gateway

Gateway слушает `127.0.0.1:9000`. Снаружи до него доходит только Nginx сервера по HTTPS.

## Локальный запуск

Нужны Docker и Docker Compose.

```bash
cp .env.example .env
sudo docker compose up --build
```

Приложение откроется на [http://127.0.0.1:9000/kittygram/](http://127.0.0.1:9000/kittygram/).

Администратора можно создать после старта базы:

```bash
sudo docker compose exec backend python manage.py createsuperuser
```

Остановка:

```bash
sudo docker compose down
```

## Переменные окружения

Шаблон лежит в `.env.example`. Файл `.env` в репозиторий не попадает: Compose читает его у контейнеров `db` и `backend`.

| Переменная | Назначение |
| --- | --- |
| `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` | Имя базы, пользователь и пароль PostgreSQL |
| `DB_HOST`, `DB_PORT` | Адрес базы для Django. В Compose это сервис `db` и порт `5432` |
| `SECRET_KEY` | Секрет Django |
| `DEBUG` | `True` или `False` |
| `ALLOWED_HOSTS` | Список хостов через запятую |
| `CSRF_TRUSTED_ORIGINS` | Доверенные origin для админки, со схемой `http://` или `https://` |
| `FORCE_SCRIPT_NAME` | Префикс приложения, в этом проекте `/kittygram` |
| `STATIC_URL`, `MEDIA_URL` | Публичные пути статики и загрузок |
| `MEDIA_ROOT` | Каталог загрузок внутри контейнера backend |

`POSTGRES_PASSWORD` PostgreSQL применяет только при первом создании тома `pg_data`.

## Продакшен

На сервере проект лежит в `/home/ubuntu/kittygram`. Рядом в `/home/ubuntu/taski` работает Taski на порту `8000`. Nginx хоста отправляет `/kittygram/` на `127.0.0.1:9000`, а пути Taski — на `127.0.0.1:8000`.

Продакшен-файл `docker-compose.production.yml` скачивает готовые образы:

- `v1ktr/kittygram_backend`
- `v1ktr/kittygram_frontend`
- `v1ktr/kittygram_gateway`

Перед первым запуском в каталоге на сервере должен лежать `.env`. Образы на Docker Hub публичные.

Ручной перезапуск на сервере:

```bash
cd /home/ubuntu/kittygram
sudo docker compose -f docker-compose.production.yml pull
sudo docker compose -f docker-compose.production.yml up -d
sudo docker compose -f docker-compose.production.yml exec backend python manage.py migrate
sudo docker compose -f docker-compose.production.yml exec backend python manage.py collectstatic --noinput
```


## CI/CD

Push в ветку `main` запускает `.github/workflows/main.yml`.

1. Проверка бэкенда flake8 и `python manage.py test`, затем тесты фронтенда.
2. Сборка трёх образов и отправка их в Docker Hub.
3. Копирование `docker-compose.production.yml` на сервер, обновление контейнеров, миграции и выкладка статики в том.
4. Сообщение в Telegram об успешном деплое.

Секреты репозитория:

| Секрет | Назначение |
| --- | --- |
| `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN` | Вход в Docker Hub. Логин должен быть `v1ktr` |
| `HOST`, `USER`, `SSH_KEY` | SSH на сервер |
| `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` | Уведомление после деплоя |

Копия workflow для проверки лежит в `kittygram_workflow.yml`. Параметры опубликованного стенда — в `tests.yml`.
