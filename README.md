# Reviews Automation Bot

Продакшен-ориентированный FastAPI-бэкенд для приёма отзывов из Яндекс Бизнеса, генерации ответов через LLM, модерации в Telegram и публикации одобренных ответов.

## Что уже реализовано

- FastAPI API с маршрутами `GET /health`, `POST /api/reviews`, `GET /api/jobs/{job_id}`, `POST /api/jobs/{job_id}/retry`, `GET /api/reviews/next-approved` и `POST /api/reviews/{review_id}/published`
- SQLAlchemy-модели и Alembic-миграции для отзывов, задач модерации, примеров одобренных ответов, событий Telegram и AI-задач
- дедупликация по стабильному `review_key`
- слой провайдеров LLM с адаптерами для `Ollama`, `OpenAI`, `YandexGPT` и `GigaChat`
- сценарий модерации в Telegram с командами `approve`, `rewrite`, `ignore` и ручным `edit:`
- JSON-логирование для API, воркеров, AI-генерации и Telegram-обработчиков
- ретраи и таймауты для исходящих вызовов в LLM и Telegram
- транспорт фоновых задач через Redis/RQ для продакшен-режима
- `docker compose`-стек для `postgres`, `redis`, `backend`, `worker` и `caddy`
- базовые тесты для дедупликации, API, AI-воркера, Telegram webhook и retry-логики

## Быстрый локальный запуск

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -e .[test]
alembic upgrade head
uvicorn apps.backend.app.main:app --reload
```

Фоновый AI-воркер без Redis:

```bash
python -m apps.backend.app.workers.ai_worker --forever --limit 10
```

Redis/RQ-воркер:

```bash
python -m apps.backend.app.workers.rq_worker ai telegram
```

## Запуск через Docker

```bash
docker compose up --build
```

Порядок старта:

- `postgres` и `redis` проходят healthcheck
- `migrate` выполняет `alembic upgrade head`
- `backend` поднимает HTTP API
- `worker` начинает читать очереди `ai` и `telegram`
- `caddy` публикует приложение по HTTP/HTTPS

## Тесты

```bash
pytest
```

## Документация

- Архитектура: `docs/architecture.md`
- API: `docs/api.md`
- Базовый setup: `docs/setup.md`
- Развёртывание на новом ПК и сервере: `docs/deployment.md`
- Расширение браузера: `extension/README.md`
