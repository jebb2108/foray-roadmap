---
tags: [api, reference, existing-system]
---

# API Эндпоинты

← [[🗺️ Foray — Карта проекта]]

Полный справочник эндпоинтов `web-app-service`. Для эндпоинтов самого [[Gateway Сервис|Gateway]] — см. отдельную заметку.

---

## `/api/user` — Пользователи

| Метод | Путь | Параметры | Описание |
|-------|------|-----------|----------|
| GET | `/api/user/check_user` | `?user_id=` | Существует ли пользователь в Gateway |
| GET | `/api/user/check_profile` | `?user_id=` | Профиль + статус верификации email |
| GET | `/api/user/check_nickname` | `?nickname=` | Уникальность никнейма |
| GET | `/api/user/check_email` | `?email=` | Доступность email |
| PUT | `/api/user/register` | body: профиль | Сохранить профиль в Gateway |
| GET | `/api/user/verify` | `?token=` | Ссылка подтверждения email → возвращает HTML |
| GET | `/api/user/partner_info` | `?match_id=` | Профиль партнёра по match_id |
| POST | `/api/user/send_email` | body: `{user_id, email}` | Отправить письмо подтверждения через Resend |
| GET | `/api/user/create_token` | `?user_id=&match_id=&room_id=` | Выдать JWT для чат-сессии |
| POST | `/api/user/exchange_contacts` | body: `{user_id, match_id}` | Запрос обмена Telegram-контактами |

---

## `/api/worker` — Матчмейкинг

| Метод | Путь | Параметры | Описание |
|-------|------|-----------|----------|
| GET | `/api/worker/check_match` | `?user_id=` | Найден ли матч |
| GET | `/api/worker/queue/status` | — | Размер очереди |
| GET | `/api/worker/queue/{user_id}/status` | — | Статус пользователя в очереди |
| POST | `/api/worker/match/toggle` | body: `{user_id}` | Войти/выйти из очереди |
| PUT | `/api/worker/cancel_match` | body: `{user_id}` | Отменить матч |
| GET | `/api/worker/chat/rooms/{room_id}/status` | — | Онлайн пользователи в комнате |
| POST | `/api/worker/notify_session_end` | body: `{room_id}` | Broadcast завершения сессии |

---

## `/api/sockets` — WebSocket

| Протокол | Путь | Query-параметры | Описание |
|----------|------|-----------------|----------|
| WS | `/api/sockets/ws/chat` | `?user_id=&token=` | Основной чат канал |

---

## `/api/dict` — Словарь

| Метод | Путь | Доступ | Описание |
|-------|------|--------|----------|
| GET | `/api/dict/words` | Открытый | Мои слова |
| POST | `/api/dict/words` | Подписка | Добавить слово |
| PUT | `/api/dict/words` | Подписка | Изменить слово |
| DELETE | `/api/dict/words` | Подписка | Удалить слово |
| GET | `/api/dict/words/search` | Подписка | Поиск (личные + публичные) |
| GET | `/api/dict/stats` | Открытый | Статистика |

---

## `/api/payments/webhook` — Платежи

| Метод | Путь | Описание |
|-------|------|----------|
| POST | `/api/payments/webhook/yookassa` | Получить webhook от YooKassa |
