# Состояние проекта `отзывы`

## Что это за сервис

Сервис автоматизирует цикл обработки отзывов:

1. Забрать реальный отзыв с площадки.
2. Отправить его в backend.
3. Сгенерировать AI-ответ.
4. Отправить ответ в Telegram на модерацию.
5. После одобрения вернуть approved reply в браузерный клиент.
6. Опубликовать ответ обратно на площадке.
7. Подтвердить публикацию в backend.

Целевой бизнес-процесс:

`площадка -> backend -> AI -> Telegram moderation -> approved reply -> публикация на площадке -> published`

## Что уже реально работает

### Backend и инфраструктура

- Docker Compose стек поднимается.
- `backend`, `worker`, `postgres`, `redis` работают.
- `GET /health` отвечает `200 OK`.
- База и очереди подключены.

### Backend workflow

- `POST /api/reviews` создаёт review и job.
- Dedupe работает.
- AI generation работает.
- Job переходит в `pending_review`.
- Telegram webhook принимает обычные updates.
- Ручной тест approve callback раньше уже успешно проходил.
- `POST /api/reviews/{review_id}/published` переводит job в `published`.

### Browser-side

- Создан Chrome MV3 extension-клиент в папке `extension/`.
- Extension подключается к странице `https://yandex.ru/sprav/...`.
- Extension видит реальные карточки отзывов.
- Extension отправляет реальные отзывы в backend.
- После `Collect Page` backend создаёт реальные jobs.
- Эти jobs доходят до `pending_review`.
- Telegram получает реальные moderation-сообщения по этим отзывам.

## Что уже доказано end-to-end

Следующие участки контура подтверждены:

- локальный backend workflow;
- реальный Telegram webhook для обычных входящих сообщений;
- реальный сбор отзывов со страницы Яндекс Бизнес;
- реальная отправка этих отзывов в backend;
- реальная доставка moderation-сообщений в Telegram.

## Что сейчас не работает стабильно

### Критическая проблема

Для новой пачки реальных отзывов нажатие `Approve` в Telegram пока не меняет status в backend.

Симптом:

- кнопка в Telegram визуально отправляется;
- но в backend не появляется новый `POST /api/telegram/webhook`;
- в `telegram_events` нет новых `approve`;
- jobs остаются в `pending_review`.

То есть сейчас не замкнут реальный контур:

`новый moderation message -> inline approve callback -> backend approved`

### Из-за этого пока не доказано

- что approved reply по новым реальным отзывам стабильно возвращается в extension;
- что extension реально публикует approved reply обратно в Яндекс Бизнес;
- что публикация по реальным отзывам доходит до `published` без ручного обхода.

## Что ещё не завершено как продукт

- Нет стабильного production webhook URL.
- Используется временный tunnel.
- Нет постоянного сервера/домена.
- DOM-логика расширения перенесена частично и требует проверки на реальной публикации.
- Нет подтверждённого автопубликатора на новых real jobs.

## Главный текущий статус

Проект больше не находится в стадии "ничего не работает".

Сейчас состояние такое:

- backend ядро работает;
- Telegram доставка в целом работает;
- browser extension начал забирать реальные отзывы;
- но полноценный сервис ещё не завершён, потому что новый real moderation callback пока не проходит до backend стабильно.

## Что осталось до полноценного сервиса

### Ближайшая цель

Добить один реальный полный цикл на настоящем отзыве:

1. real review collected from Yandex Business
2. job created
3. AI reply generated
4. Telegram approve callback received by backend
5. job becomes `approved`
6. extension получает approved reply
7. extension публикует reply в Яндекс
8. backend получает `published`

### После этого

- стабилизировать callback flow;
- стабилизировать publish flow;
- перевести webhook на постоянный публичный адрес;
- довести сервис до повторяемого рабочего режима.

## Связанные файлы

- [latest-telegram-callback-error.md](C:/Users/a_elkin/Documents/отзывы/latest-telegram-callback-error.md)
- [sammary.md](C:/Users/a_elkin/Documents/отзывы/sammary.md)
- [extension/README.md](C:/Users/a_elkin/Documents/отзывы/extension/README.md)
