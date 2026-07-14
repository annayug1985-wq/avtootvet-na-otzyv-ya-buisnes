# Техническое задание для разработчика на Python

## 1. Название проекта

Система полуавтоматической обработки отзывов в Яндекс Бизнес с AI-генерацией ответов, модерацией через Telegram и публикацией approved-ответов.

## 2. Цель проекта

Разработать production-версию системы, которая:

- получает отзывы из интерфейса Яндекс Бизнес;
- передаёт их в backend;
- генерирует черновик ответа через AI;
- отправляет ответ на модерацию в Telegram;
- после одобрения возвращает approved-ответ в браузерный клиент;
- публикует ответ в Яндекс Бизнес;
- хранит все статусы, события и историю в БД;
- поддерживает несколько аккаунтов;
- не теряет состояние после перезапуска.

## 3. Текущий контекст

Уже существует MVP в нескольких версиях:

- ранний вариант: `Tampermonkey + Node.js + Ollama + Telegram`;
- более зрелый вариант: queue-flow через backend;
- launcher-варианты для multi-account;
- Python-перенос части backend на `FastAPI`.

Новая разработка должна не копировать хаотично старые версии, а собрать production-реализацию с чистой архитектурой на Python.

## 4. Целевая архитектура

```text
Browser Extension / Browser Client
    -> Backend API
    -> PostgreSQL
    -> Redis Queue
    -> AI Worker
    -> Telegram Webhook
    -> Approved Reply
    -> Browser publishes reply
```

## 5. Основные требования

### 5.1 Функциональные требования

Система должна:

- принимать отзывы из браузерного клиента;
- вычислять уникальный `review_key`;
- защищать от дублей;
- создавать `review` и `moderation_job`;
- ставить задачу на AI-генерацию в очередь;
- отправлять результат в Telegram на модерацию;
- принимать действия из Telegram:
  - approve;
  - rewrite;
  - edit;
  - ignore;
- менять статус job в БД;
- отдавать клиенту текущий статус и approved-ответ;
- фиксировать факт публикации ответа;
- сохранять approved-ответы как обучающие примеры;
- поддерживать multi-account.

### 5.2 Нефункциональные требования

- Backend должен быть написан на Python.
- Основной API-фреймворк: `FastAPI`.
- База данных: `PostgreSQL`.
- Очереди: `Redis + BullMQ equivalent for Python`, предпочтительно `RQ`, `Celery` или `Arq`.
- Весь state должен храниться в БД, а не в памяти процесса.
- Код должен быть модульным.
- Логи должны быть структурированными.
- Все интеграции должны быть изолированы в отдельных сервисах.
- Система должна быть готова к запуску в Docker.

## 6. Предпочтительный стек

- Python 3.12+
- FastAPI
- SQLAlchemy 2.x
- Alembic
- PostgreSQL
- Redis
- Celery или Arq
- httpx
- pydantic v2
- structlog или standard logging в JSON-формате
- Docker / docker-compose

## 7. Структура проекта

Рекомендуемая структура:

```text
reviews-automation-bot/
├─ README.md
├─ .env.example
├─ docker-compose.yml
├─ pyproject.toml
├─ alembic.ini
├─ migrations/
│  └─ versions/
│
├─ apps/
│  ├─ backend/
│  │  ├─ app/
│  │  │  ├─ main.py
│  │  │  ├─ config/
│  │  │  │  └─ settings.py
│  │  │  ├─ api/
│  │  │  │  ├─ health.py
│  │  │  │  ├─ reviews.py
│  │  │  │  ├─ jobs.py
│  │  │  │  └─ telegram.py
│  │  │  ├─ db/
│  │  │  │  ├─ base.py
│  │  │  │  ├─ session.py
│  │  │  │  └─ models/
│  │  │  │     ├─ review.py
│  │  │  │     ├─ moderation_job.py
│  │  │  │     ├─ approved_example.py
│  │  │  │     └─ telegram_event.py
│  │  │  ├─ schemas/
│  │  │  │  ├─ review.py
│  │  │  │  ├─ job.py
│  │  │  │  └─ telegram.py
│  │  │  ├─ services/
│  │  │  │  ├─ review_service.py
│  │  │  │  ├─ moderation_service.py
│  │  │  │  ├─ ai_service.py
│  │  │  │  ├─ telegram_service.py
│  │  │  │  ├─ dedupe_service.py
│  │  │  │  └─ example_service.py
│  │  │  ├─ repositories/
│  │  │  │  ├─ review_repository.py
│  │  │  │  ├─ moderation_repository.py
│  │  │  │  ├─ approved_example_repository.py
│  │  │  │  └─ telegram_event_repository.py
│  │  │  ├─ workers/
│  │  │  │  ├─ ai_worker.py
│  │  │  │  └─ telegram_worker.py
│  │  │  ├─ providers/
│  │  │  │  ├─ base.py
│  │  │  │  ├─ ollama.py
│  │  │  │  ├─ openai.py
│  │  │  │  ├─ yandexgpt.py
│  │  │  │  └─ gigachat.py
│  │  │  ├─ prompts/
│  │  │  │  └─ review_reply.prompt.txt
│  │  │  └─ utils/
│  │  │     ├─ hash.py
│  │  │     ├─ text.py
│  │  │     ├─ logger.py
│  │  │     └─ enums.py
│  │  └─ tests/
│
└─ docs/
   ├─ architecture.md
   ├─ setup.md
   ├─ api.md
   └─ troubleshooting.md
```

## 8. Бизнес-сущности

### 8.1 Review

Поля:

- `id: UUID`
- `review_key: str`
- `source: str`
- `account_id: str | null`
- `author: str | null`
- `rating: int`
- `review_text: str`
- `address: str | null`
- `review_date: str | null`
- `page_url: str | null`
- `status: str`
- `created_at`
- `updated_at`

### 8.2 ModerationJob

Поля:

- `id: UUID`
- `review_id: UUID`
- `status: str`
- `generated_reply: str | null`
- `approved_reply: str | null`
- `rewrite_count: int`
- `approved_by: str | null`
- `created_at`
- `updated_at`

### 8.3 ApprovedExample

Поля:

- `id: UUID`
- `review_text: str`
- `answer_text: str`
- `rating: int | null`
- `created_at`

### 8.4 TelegramEvent

Поля:

- `id: UUID`
- `telegram_update_id: str | null`
- `job_id: UUID | null`
- `action: str`
- `payload: JSONB`
- `created_at`

## 9. Статусы

### 9.1 Статусы review/job

Минимальный набор:

- `new`
- `generating`
- `pending_review`
- `approved`
- `published`
- `failed`
- `ignored`

### 9.2 Переходы

- `new -> generating`
- `generating -> pending_review`
- `pending_review -> approved`
- `pending_review -> generating` при `rewrite`
- `pending_review -> ignored`
- `approved -> published`
- любой этап -> `failed` при ошибке

## 10. Dedupe-логика

Необходимо реализовать стабильный `review_key` по хэшу:

```text
account_id + rating + author + date + address + normalized_review_text
```

Требования:

- нормализация текста обязательна;
- использовать `sha256`;
- одинаковый отзыв не должен создавать новую задачу повторно;
- при повторном получении уже известного отзыва backend должен вернуть существующий `reviewId/jobId`.

## 11. API

### 11.1 Health

`GET /health`

Ответ:

```json
{
  "ok": true
}
```

### 11.2 Создание/синхронизация отзыва

`POST /api/reviews`

Тело:

```json
{
  "accountId": "account_1",
  "rating": 5,
  "author": "Иван",
  "date": "12 апреля 2026",
  "address": "Москва",
  "reviewText": "Все отлично",
  "pageUrl": "https://business.yandex..."
}
```

Ответ:

```json
{
  "ok": true,
  "reviewId": "uuid",
  "jobId": "uuid",
  "status": "generating",
  "duplicate": false
}
```

### 11.3 Получение статуса job

`GET /api/jobs/{job_id}`

Ответ:

```json
{
  "ok": true,
  "jobId": "uuid",
  "status": "approved",
  "reply": "Спасибо за обратную связь..."
}
```

### 11.4 Подтверждение публикации

`POST /api/reviews/{review_id}/published`

Тело:

```json
{
  "published": true
}
```

### 11.5 Telegram webhook

`POST /api/telegram/webhook`

Backend должен:

- принимать update от Telegram;
- определять действие;
- обновлять job;
- логировать событие;
- возвращать `200 OK`.

## 12. AI provider layer

Нужен единый интерфейс провайдера.

Пример:

```python
class BaseAiProvider:
    async def generate_reply(self, *, review_text: str, rating: int, examples: list[dict], previous_reply: str | None = None) -> str:
        raise NotImplementedError
```

Реализации:

- `OllamaProvider`
- `OpenAIProvider`
- `YandexGptProvider`
- `GigaChatProvider`

Выбор через `.env`:

```env
AI_PROVIDER=ollama
```

## 13. Prompt management

Prompt нельзя хардкодить в бизнес-логике.

Нужно:

- хранить его в отдельном файле;
- подгружать на уровне сервиса;
- подмешивать approved examples;
- поддерживать режим rewrite с учётом предыдущего ответа.

Файл:

```text
apps/backend/app/prompts/review_reply.prompt.txt
```

## 14. Очереди и воркеры

### 14.1 Очередь AI

Вход:

- `job_id`

Действия:

- загрузить job и review;
- собрать prompt;
- сгенерировать ответ;
- сохранить `generated_reply`;
- поменять статус на `pending_review`;
- инициировать Telegram-отправку.

### 14.2 Очередь Telegram

Действия:

- отправить сообщение в Telegram;
- прикрепить inline-кнопки:
  - approve
  - rewrite
  - ignore
- при наличии канала-копии отправить копию без кнопок.

### 14.3 Retry policy

Должны быть retries для:

- AI provider errors;
- Telegram API errors;
- временных сетевых ошибок.

## 15. Telegram-модерация

Требования:

- inline-кнопки для approve/rewrite/ignore;
- возможность ручного edit через отдельный flow;
- защита от повторной обработки одного и того же update;
- хранение `telegram_update_id` и payload в БД.

Желательно:

- formatter для текста сообщений;
- безопасная обработка `callback_data`;
- idempotency.

## 16. Multi-account

Система должна поддерживать несколько аккаунтов.

Требования:

- `account_id` обязателен в API;
- все dedupe keys учитывают аккаунт;
- статусы и очереди должны различаться по аккаунтам;
- должна быть возможность вводить разные лимиты или маршрутизацию по аккаунтам позже.

## 17. Логирование

Нужно логировать события:

- `review_received`
- `duplicate_detected`
- `job_created`
- `ai_generation_started`
- `ai_generation_failed`
- `telegram_sent`
- `telegram_action_received`
- `reply_approved`
- `reply_rewritten`
- `reply_ignored`
- `reply_published`

Требования:

- structured logs;
- `request_id` или `correlation_id` желательно;
- ошибки с traceback;
- отдельный логгер для workers.

## 18. Конфигурация

Пример `.env.example`:

```env
APP_ENV=development
PORT=3000
APP_BASE_URL=http://localhost:3000

DATABASE_URL=postgresql+psycopg://reviews:reviews@postgres:5432/reviews_bot
REDIS_URL=redis://redis:6379/0

AI_PROVIDER=ollama
OLLAMA_URL=http://host.docker.internal:11434
OLLAMA_MODEL=qwen2.5:7b

OPENAI_API_KEY=
YANDEXGPT_API_KEY=
GIGACHAT_CLIENT_ID=
GIGACHAT_CLIENT_SECRET=

TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=
TELEGRAM_CHANNEL_ID=

PUBLISH_DELAY_MIN_MS=5000
PUBLISH_DELAY_MAX_MS=10000
```

## 19. База данных

Нужно подготовить Alembic migrations для таблиц:

- `reviews`
- `moderation_jobs`
- `approved_examples`
- `telegram_events`

Требования:

- UUID primary keys;
- индексы на `review_key`, `status`, `account_id`;
- foreign key между `moderation_jobs.review_id` и `reviews.id`.

## 20. Docker

Нужно подготовить `docker-compose.yml` минимум для:

- `postgres`
- `redis`
- `backend`
- `worker`

Если worker выделяется в отдельный сервис, он должен запускаться отдельно от HTTP API.

## 21. Тестирование

Минимум нужно покрыть:

- нормализацию текста;
- генерацию `review_key`;
- dedupe;
- создание review/job;
- смену статусов;
- обработку Telegram approve/rewrite;
- AI provider interface через mock;
- API smoke tests.

## 22. Ограничения и правила

- Не хранить рабочее состояние в `dict/Map` как в MVP.
- Не завязывать логику на Tampermonkey-специфику.
- Не смешивать HTTP routes и бизнес-логику в одном файле.
- Не хардкодить prompt в коде.
- Не принимать решение о публикации в браузере без статуса `approved` от backend.
- Любой внешний вызов должен быть обработан с timeout и retry-политикой.

## 23. Этапы реализации

### Этап 1. Базовый backend

Сделать:

- FastAPI app;
- PostgreSQL models;
- migrations;
- `POST /api/reviews`;
- `GET /api/jobs/{id}`;
- dedupe;
- создание review/job.

### Этап 2. AI и очередь

Сделать:

- AI provider abstraction;
- Ollama provider;
- очередь AI generation;
- сохранение generated reply;
- перевод job в `pending_review`.

### Этап 3. Telegram

Сделать:

- Telegram service;
- отправку сообщений с кнопками;
- webhook endpoint;
- обработку approve/rewrite/ignore;
- сохранение telegram events.

### Этап 4. Публикационный flow

Сделать:

- endpoint подтверждения публикации;
- выдачу approved reply клиенту;
- перевод job в `published`.

### Этап 5. Hardening

Сделать:

- structured logging;
- retry policies;
- docker;
- тесты;
- документацию.

## 24. Ожидаемый результат

На выходе разработчик должен предоставить:

- рабочий backend на Python;
- миграции БД;
- docker-compose;
- `.env.example`;
- README по запуску;
- документацию по API;
- базовые тесты;
- код, готовый к подключению браузерного клиента.

## 25. Критерии приёмки

Система считается принятой, если:

- отзыв создаётся через API;
- дубли корректно определяются;
- job создаётся и живёт в БД;
- AI-ответ генерируется асинхронно;
- Telegram-модерация меняет статус job;
- approved reply можно получить через API;
- публикация фиксируется в БД;
- после перезапуска backend данные не теряются;
- проект запускается локально через Docker.
