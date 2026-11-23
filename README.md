<div align="center">

# 🐾 Kittygram

**Социальная сеть для владельцев кошек с полным CI/CD: тесты → Docker Hub → автодеплой на сервер → уведомление в Telegram**

![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)
![Django](https://img.shields.io/badge/Django-DRF-092E20?logo=django&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-13-4169E1?logo=postgresql&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-3_images-2496ED?logo=docker&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?logo=githubactions&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-gateway-009639?logo=nginx&logoColor=white)

</div>

## О проекте

Kittygram — сервис, где пользователи публикуют карточки своих кошек: фото, имя, год рождения, цвет и достижения.
Главное в этом репозитории — **инфраструктура**: приложение разбито на контейнеры, образы собираются и публикуются автоматически, а каждый push в `main` без ручных действий доезжает до production-сервера.

## CI/CD-пайплайн

Workflow [`.github/workflows/main.yml`](.github/workflows/main.yml) запускается на каждый push в `main` и состоит из трёх последовательных jobs:

```
push в main
   │
   ▼
1. tests ─────────── flake8 (backend) → pytest (backend) → Jest (frontend)
   │
   ▼
2. build_and_push ── сборка и публикация 3 образов в Docker Hub:
   │                 kittygram_backend · kittygram_frontend · kittygram_gateway
   ▼
3. deploy ────────── SSH на сервер → docker compose pull → up
                     → migrate → collectstatic → уведомление в Telegram
```

- Следующий этап запускается, только если предыдущий прошёл (`needs`).
- Секреты (Docker Hub, SSH-ключ, токен Telegram-бота) хранятся в GitHub Secrets и не попадают в код.
- Образы опубликованы в Docker Hub: [`evingb/kittygram_backend`](https://hub.docker.com/r/evingb/kittygram_backend), [`evingb/kittygram_frontend`](https://hub.docker.com/r/evingb/kittygram_frontend), [`evingb/kittygram_gateway`](https://hub.docker.com/r/evingb/kittygram_gateway).

## Архитектура

| Сервис | Образ | Назначение |
|---|---|---|
| `db` | `postgres:13` | база данных, данные в volume `pg_data` |
| `backend` | `kittygram_backend` | Django REST API на Gunicorn |
| `frontend` | `kittygram_frontend` | собирает React-приложение и копирует сборку в общий volume |
| `gateway` | `kittygram_gateway` | Nginx: единая точка входа, проксирует `/api/` и `/admin/`, раздаёт статику и медиа |

Два compose-файла:

- `docker-compose.yml` — локальная сборка образов из исходников;
- `docker-compose.production.yml` — запуск готовых образов из Docker Hub на сервере.

## Что сделано

- Написаны Dockerfile для backend, frontend и gateway, конфигурация Nginx и два compose-файла.
- Настроен GitHub Actions workflow из 3 jobs: линтинг и тесты, сборка и публикация образов, деплой по SSH с Telegram-уведомлением.
- Все настройки Django (БД, `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS`) вынесены в переменные окружения.
- Проект был развёрнут на облачном сервере Ubuntu с HTTPS (Let's Encrypt) и обновлялся только через пайплайн.

## Стек

Python 3.11 · Django · Django REST Framework · Djoser · PostgreSQL · Gunicorn · Nginx · Docker · Docker Compose · Docker Hub · GitHub Actions · React

## Запуск локально

**Требования:** Docker 24+ и Docker Compose v2.

```bash
git clone https://github.com/vadim-guseinov/kittygram_final.git
cd kittygram_final

cp .env.example .env          # заполните пароли и SECRET_KEY
docker compose up -d --build

docker compose exec backend python manage.py migrate
docker compose exec backend python manage.py collectstatic --noinput
docker compose exec backend python manage.py createsuperuser
```

Приложение будет доступно на http://localhost:9000.

### Запуск на сервере из готовых образов

```bash
docker compose -f docker-compose.production.yml pull
docker compose -f docker-compose.production.yml up -d
docker compose -f docker-compose.production.yml exec backend python manage.py migrate
docker compose -f docker-compose.production.yml exec backend python manage.py collectstatic --noinput
```

### Секреты репозитория для CI/CD

| Секрет | Назначение |
|---|---|
| `DOCKER_USERNAME`, `DOCKER_PASSWORD` | публикация образов в Docker Hub |
| `SSH_PRIVATE_KEY` | доступ к серверу для деплоя |
| `TELEGRAM_TOKEN`, `TELEGRAM_CHAT_ID` | уведомление об успешном деплое |

### Тесты

В CI выполняются flake8 по backend, `pytest` (проверки инфраструктуры: Dockerfile, compose-файлы, workflow, наличие образов в Docker Hub, доступность развёрнутого сервиса) и Jest-тесты фронтенда. Локально:

```bash
pip install -r backend/requirements.txt flake8 pytest
flake8 backend --ignore=E501
pytest
```

## Пример запроса к API

```bash
# получить токен
curl -X POST http://localhost:9000/api/token/login/ \
  -H "Content-Type: application/json" \
  -d '{"username": "user", "password": "strongpassword"}'

# список кошек
curl http://localhost:9000/api/cats/ -H "Authorization: Token <token>"
```

## Автор

**Вадим Гусейнов** — Python-разработчик (Backend) · [Telegram](https://t.me/eviliing)
