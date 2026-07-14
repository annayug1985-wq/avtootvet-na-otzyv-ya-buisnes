# API

## Health

`GET /health`

Response:

```json
{
  "ok": true
}
```

## Create Or Sync Review

`POST /api/reviews`

Request:

```json
{
  "accountId": "account_1",
  "rating": 5,
  "author": "Иван",
  "date": "2026-06-18",
  "address": "Москва",
  "reviewText": "Все отлично",
  "pageUrl": "https://business.yandex.ru/review/1"
}
```

Response:

```json
{
  "ok": true,
  "reviewId": "uuid",
  "jobId": "uuid",
  "status": "generating",
  "duplicate": false
}
```

## Get Job Status

`GET /api/jobs/{job_id}`

Response:

```json
{
  "ok": true,
  "jobId": "uuid",
  "status": "approved",
  "reply": "Спасибо за обратную связь..."
}
```

`reply` is returned only after backend approval.

## Retry Failed Job

`POST /api/jobs/{job_id}/retry`

Retries a job only when its current status is `failed`.

Response:

```json
{
  "ok": true,
  "jobId": "uuid",
  "status": "generating",
  "reply": null
}
```

## Mark Reply As Published

`POST /api/reviews/{review_id}/published`

Request:

```json
{
  "published": true
}
```

Response:

```json
{
  "ok": true,
  "reviewId": "uuid",
  "jobId": "uuid",
  "status": "published"
}
```

## Telegram Webhook

`POST /api/telegram/webhook`

The endpoint accepts Telegram `callback_query` and manual `edit:` commands, stores inbound events, ignores duplicate `update_id`, and always returns:

```json
{
  "ok": true
}
```
