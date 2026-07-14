# Развёртывание

Этот документ описывает два практических сценария:

1. новый компьютер разработчика или оператора с локальной LLM через `Ollama`
2. серверное развёртывание с внешним LLM-провайдером по API

## 1. Что это за система

Проект состоит из нескольких частей:

- FastAPI-бэкенд, который принимает отзывы, хранит состояние и отдаёт одобренные ответы
- AI-воркер, который генерирует черновики ответов
- Telegram-интеграция для модерации
- браузерное расширение из папки `extension/`, которое собирает отзывы и публикует одобренные ответы
- PostgreSQL и Redis в серверном режиме

## 2. Общие требования

Для любого сценария понадобятся:

- `Python 3.12`
- `git`
- доступ к Telegram Bot API, если нужна модерация через Telegram

Для локального сценария с Ollama дополнительно:

- установленный `Ollama`
- скачанная локальная модель, например `qwen2.5:7b`

Для серверного сценария дополнительно:

- `Docker` и `Docker Compose`
- домен, указывающий на сервер
- открытые порты `80` и `443`
- API-ключ выбранного LLM-провайдера

## 3. Клонирование проекта

```bash
git clone <URL_репозитория>
cd отзывы
```

## 4. Переменные окружения

Проект читает настройки из файла `.env` в корне репозитория.

Минимальный шаблон:

```env
APP_ENV=development
PORT=8000
APP_BASE_URL=http://localhost:8000

DATABASE_URL=sqlite:///./reviews.db
REDIS_URL=redis://localhost:6379/0
QUEUE_TRANSPORT=database

AI_PROVIDER=ollama
AI_EXAMPLES_LIMIT=3

OLLAMA_URL=http://localhost:11434
OLLAMA_MODEL=qwen2.5:7b
OLLAMA_TIMEOUT_SECONDS=30

OPENAI_API_KEY=
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4.1-mini

YANDEXGPT_API_KEY=
YANDEXGPT_FOLDER_ID=

GIGACHAT_CLIENT_ID=
GIGACHAT_CLIENT_SECRET=
GIGACHAT_SCOPE=GIGACHAT_API_PERS

TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=
TELEGRAM_CHANNEL_ID=
TELEGRAM_API_BASE=https://api.telegram.org

LOG_LEVEL=INFO
EXTERNAL_REQUEST_MAX_RETRIES=2
EXTERNAL_REQUEST_BACKOFF_SECONDS=1
WORKER_POLL_INTERVAL_SECONDS=2
```

Примечания:

- `QUEUE_TRANSPORT=database` подходит для простого локального запуска без Redis.
- `QUEUE_TRANSPORT=redis` нужен для серверного режима и `docker compose`.
- `DATABASE_URL=sqlite:///./reviews.db` подходит для локальной разработки.
- Для сервера лучше использовать PostgreSQL.

## 5. Новый компьютер с локальной LLM через Ollama

Это самый простой сценарий для первого запуска.

### 5.1. Установить зависимости Python

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -e .[test]
```

### 5.2. Установить и подготовить Ollama

Проверьте, что Ollama запущен:

```bash
ollama serve
```

В отдельном окне скачайте модель:

```bash
ollama pull qwen2.5:7b
```

Если будете использовать другую модель, поменяйте `OLLAMA_MODEL` в `.env`.

### 5.3. Подготовить `.env` для локального режима

Рекомендуемое минимальное содержимое:

```env
APP_ENV=development
APP_BASE_URL=http://localhost:8000
DATABASE_URL=sqlite:///./reviews.db
QUEUE_TRANSPORT=database
AI_PROVIDER=ollama
OLLAMA_URL=http://localhost:11434
OLLAMA_MODEL=qwen2.5:7b
LOG_LEVEL=INFO
```

Если Telegram пока не нужен, токены можно оставить пустыми.

### 5.4. Применить миграции

```bash
alembic upgrade head
```

### 5.5. Запустить API

```bash
uvicorn apps.backend.app.main:app --reload
```

После этого API будет доступен на [http://localhost:8000/health](http://localhost:8000/health).

### 5.6. Запустить локальный AI-воркер

```bash
python -m apps.backend.app.workers.ai_worker --forever --limit 10
```

Этот режим использует базу как транспорт задач и не требует Redis.

### 5.7. При необходимости подключить браузерное расширение

1. Откройте `chrome://extensions`
2. Включите `Developer mode`
3. Нажмите `Load unpacked`
4. Выберите папку `extension/`
5. В настройках расширения укажите:

- `Backend URL`: `http://localhost:8000`
- `Account ID`: идентификатор аккаунта, с которым работаете

### 5.8. Если нужна Telegram-модерация локально

Заполните в `.env`:

```env
TELEGRAM_BOT_TOKEN=...
TELEGRAM_CHAT_ID=...
```

Для настоящего webhook Telegram нужен публичный HTTPS URL. Локально это обычно делают через туннель, например `ngrok`, а затем:

```bash
python -m apps.backend.app.scripts.telegram_webhook set
python -m apps.backend.app.scripts.telegram_webhook info
```

Перед этим выставьте:

```env
APP_BASE_URL=https://ваш-публичный-url
```

## 6. Серверное развёртывание с LLM по API

Этот сценарий рассчитан на сервер с постоянной работой, PostgreSQL, Redis и HTTPS.

### 6.1. Подготовить сервер

На сервере должны быть:

- Linux-сервер с публичным IP
- DNS-запись домена на этот сервер
- установленные `Docker` и `Docker Compose`

### 6.2. Подготовить `.env` для сервера

Пример для OpenAI API:

```env
APP_ENV=production
APP_BASE_URL=https://reviews.example.com
APP_DOMAIN=reviews.example.com
ACME_EMAIL=admin@example.com

DATABASE_URL=postgresql+psycopg://reviews:reviews@postgres:5432/reviews_bot
REDIS_URL=redis://redis:6379/0
QUEUE_TRANSPORT=redis

AI_PROVIDER=openai
OPENAI_API_KEY=sk-...
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_MODEL=gpt-4.1-mini

TELEGRAM_BOT_TOKEN=...
TELEGRAM_CHAT_ID=...
TELEGRAM_CHANNEL_ID=

LOG_LEVEL=INFO
EXTERNAL_REQUEST_MAX_RETRIES=2
EXTERNAL_REQUEST_BACKOFF_SECONDS=1
```

Если используете не OpenAI, а другой API-провайдер:

- для YandexGPT задайте `AI_PROVIDER=yandexgpt`, `YANDEXGPT_API_KEY`, `YANDEXGPT_FOLDER_ID`
- для GigaChat задайте `AI_PROVIDER=gigachat`, `GIGACHAT_CLIENT_ID`, `GIGACHAT_CLIENT_SECRET`

### 6.3. Запустить стек

```bash
docker compose up --build -d
```

Контейнеры поднимутся в таком порядке:

1. `postgres`
2. `redis`
3. `migrate`
4. `backend`
5. `worker`
6. `caddy`

### 6.4. Проверить здоровье сервисов

```bash
docker compose ps
docker compose logs backend --tail=100
docker compose logs worker --tail=100
```

Проверьте endpoint:

```bash
curl https://reviews.example.com/health
```

Ожидаемый ответ:

```json
{"ok":true}
```

### 6.5. Зарегистрировать Telegram webhook

Когда домен уже доступен по HTTPS:

```bash
docker compose exec backend python -m apps.backend.app.scripts.telegram_webhook set
docker compose exec backend python -m apps.backend.app.scripts.telegram_webhook info
```

Webhook должен смотреть на:

```text
https://reviews.example.com/api/telegram/webhook
```

## 7. Как переключить провайдера LLM

Смена провайдера делается без изменения кода, только через `.env`.

Локально через Ollama:

```env
AI_PROVIDER=ollama
OLLAMA_URL=http://localhost:11434
OLLAMA_MODEL=qwen2.5:7b
```

Через OpenAI API:

```env
AI_PROVIDER=openai
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4.1-mini
```

Через YandexGPT API:

```env
AI_PROVIDER=yandexgpt
YANDEXGPT_API_KEY=...
YANDEXGPT_FOLDER_ID=...
```

Через GigaChat API:

```env
AI_PROVIDER=gigachat
GIGACHAT_CLIENT_ID=...
GIGACHAT_CLIENT_SECRET=...
```

После изменения переменных перезапустите API и воркер.

## 8. Минимальная проверка после запуска

После развёртывания полезно проверить цепочку целиком:

1. открыть `/health`
2. отправить тестовый отзыв в `POST /api/reviews`
3. убедиться, что задача получила статус `generating`, затем `pending_review`
4. одобрить или переписать ответ через Telegram
5. проверить, что расширение или клиент получает ответ через `GET /api/reviews/next-approved`
6. подтвердить публикацию через `POST /api/reviews/{review_id}/published`

## 9. Что важно помнить

- Локальный режим без Redis проще для первого запуска, но не является боевым.
- Для продакшена лучше использовать только PostgreSQL + Redis + Docker Compose.
- Telegram webhook требует публичный HTTPS URL.
- Браузерное расширение нужно загружать отдельно из папки `extension/`.
- Если модель в Ollama тяжёлая, локальный AI-воркер может отвечать заметно медленнее, чем облачный API.
