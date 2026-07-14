# Последняя ошибка: Telegram callback не доходит до backend

## Что происходит

Текстовые сообщения от бота в Telegram доходят до backend через webhook, но нажатия на inline-кнопку `Approve` для новых moderation-сообщений не приводят к смене статуса job.

## Симптомы

- В Telegram кнопка `Approve` визуально реагирует на нажатие.
- В backend после нажатия не появляется новый `POST /api/telegram/webhook`.
- В таблице `telegram_events` для новых отзывов есть только `sent_for_moderation`.
- Новых событий `approve` для этих job не появляется.
- После `Refresh Statuses` новые job остаются в статусе `pending_review`.

## Подтверждение

### В backend-логах

- Есть входящие webhook для обычных текстовых сообщений Telegram.
- Нет новых webhook-вызовов после нажатия `Approve` на свежих moderation-сообщениях.

### В PostgreSQL

Проверка последних moderation-событий:

```sql
select created_at, action, telegram_update_id, job_id
from telegram_events
order by created_at desc
limit 10;
```

Результат: для новых job виден только `sent_for_moderation`.

Проверка callback-действий:

```sql
select created_at, action, telegram_update_id, job_id
from telegram_events
where action in ('approve', 'rewrite', 'ignore', 'edit', 'update_error')
order by created_at desc
limit 20;
```

Результат: есть только старые `approve` из прошлых тестов, новых нет.

## Что уже работает

- Backend принимает отзывы.
- Расширение забирает реальные отзывы со страницы `yandex.ru/sprav/...`.
- Worker генерирует ответы.
- Telegram получает новые moderation-сообщения.
- Webhook для обычных Telegram message-update работает.

## Где именно сбой

Сбой находится в контуре:

`inline button click in Telegram -> callback_query -> /api/telegram/webhook`

А не в контуре:

`text message in Telegram -> message update -> /api/telegram/webhook`

## Найденная причина

В `apps/backend/app/scripts/telegram_webhook.py` webhook регистрировался только с `url`, без явного `allowed_updates`.

Для Telegram это критично: если webhook раньше был установлен с ограничением типов апдейтов, повторный `setWebhook` без `allowed_updates` может сохранить старый фильтр. В таком состоянии backend продолжит получать `message`, но не будет получать `callback_query`.

Это точно совпадает с наблюдаемым поведением:

- текстовые webhook-апдейты доходят;
- нажатие на inline-кнопку визуально отрабатывает в Telegram;
- новый `POST /api/telegram/webhook` для `callback_query` не появляется.

## Что изменено в коде

- В `setWebhook` добавлена явная регистрация `allowed_updates=["message","callback_query"]`.
- Добавлен тест, который проверяет этот контракт.

## Следующие действия

1. Перерегистрировать webhook после обновления командой `python -m apps.backend.app.scripts.telegram_webhook set`.
2. Проверить текущую регистрацию командой `python -m apps.backend.app.scripts.telegram_webhook info`.
3. Повторить один клик `Approve` на свежем moderation-сообщении.
4. Убедиться, что:
   - в backend-логах появился `POST /api/telegram/webhook`;
   - в `telegram_events` появилась запись `approve`;
   - job перешёл из `pending_review` в `approved`.
