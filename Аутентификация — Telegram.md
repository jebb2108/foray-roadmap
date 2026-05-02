---
tags: [auth, telegram, existing-system]
---

# Аутентификация — Telegram

← [[🗺️ Foray — Карта проекта]] | Замена → [[Foray — Аутентификация]]

## Как работает сейчас

```
Пользователь открывает бота @lllangbot в Telegram
    ↓
Telegram открывает Mini App (chat.lllang.site)
    ↓
JS читает window.Telegram.WebApp.initDataUnsafe.user.id
    ↓ (fallback)
URL параметр ?user_id=
    ↓
GET /api/user/check_user?user_id=<telegram_id>
    ↓ если нет профиля
Форма регистрации (ник, email, дата рождения, пол, интересы)
    ↓
Email верификация (Resend)
    ↓
Профиль создан в Gateway
```

## JWT (сессионный токен)

Токен выдаётся **только в момент нахождения матча**:

```python
# src/validators/tokens.py
payload = {
    "user_id": user_id,
    "nickname": ...,
    "age": ...,
    "intro": ...,
    "gender": ...,
    "topics": [...],
    "room_id": ...,
    "match_id": ...,
    "expires_at": unix_timestamp  # TTL: 15 минут
}
# Алгоритм: HS256
# Секрет: SECRET_KEY env var
```

## Верификация email

- Отправляется через Resend
- Шаблон: `front/main/email_templates/confirmation.html`
- Плейсхолдер `{{ confirmation_url }}`
- Эндпоинт подтверждения: `GET /api/user/verify`

## Защита эндпоинтов

- **WebSocket:** токен валидируется при подключении → закрывается с `WS_1008_POLICY_VIOLATION` при ошибке
- **Словарь и матчмейкинг:** дополнительно проверяется активность [[Подписки и Платежи|подписки]]

## Проблемы для Foray

| Проблема | Следствие |
|----------|-----------|
| Telegram ID = единственный идентификатор | Нет способа войти без Telegram |
| Нет серверной валидации `initData` | Уязвимость к подделке user_id |
| JWT без refresh токена | Нужно перевыпускать каждый раз |
| Короткий TTL (15 мин) жёстко привязан к сессии чата | Нельзя использовать для общей авторизации |
