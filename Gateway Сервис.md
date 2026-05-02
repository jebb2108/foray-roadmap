---
tags: [gateway, backend, infrastructure, existing-system]
---

# Gateway Сервис

← [[🗺️ Foray — Карта проекта]]

## Роль

Gateway — центральный backend сервис, который хранит ВСЕ данные. `web-app-service` является тонким прокси-слоем над ним.

```
Клиент → web-app-service → Gateway (217.149.29.173:1000)
                                 ↕
                            База данных
                            Worker (матчмейкинг)
                            AI сервис
                            Платёжный сервис
```

## Адрес

```
http://217.149.29.173:1000
```

## Префиксы роутов

| Префикс | Назначение |
|---------|------------|
| `/api` | Пользователи, профили, история |
| `/api/worker` | Матчмейкинг, очередь, сессии |
| `/api/payments` | Подписки, YooKassa |
| `/api/dict` | Словарь |
| `/api/ai` | AI функции (объявлен, не используется в web-app) |

## Ключевые Gateway эндпоинты (используемые web-app)

| Путь | Назначение |
|------|------------|
| `GET /api/user?user_id=` | Получить профиль пользователя |
| `POST /api/user` | Создать профиль |
| `GET /api/user/partner?match_id=` | Данные партнёра по match_id |
| `GET /api/worker/check_match?user_id=` | Статус матча |
| `POST /api/worker/match/toggle` | Войти/выйти из очереди |
| `GET /api/payments/due_to?user_id=` | Дата окончания подписки |
| `POST /api/payments/webhook/yookassa` | Принять платёжный webhook |
| `GET /api/dict/words?user_id=` | Слова пользователя |
| `GET /api/sockets/chat/rooms/{room_id}/messages` | История сообщений (зашифровано Fernet) |

## Что делает web-app-service сам (без Gateway)

1. **Выдаёт JWT** — генерируется локально на основе данных из Gateway
2. **WebSocket соединения** — состояние в памяти процесса
3. **Отправка email** — через Resend напрямую
4. **Nginx / статика** — фронтенд файлы

## Для Foray

Gateway остаётся как есть. Мобильное приложение будет обращаться к тем же Gateway эндпоинтам, но через новый мобильный API-слой, который заменит `web-app-service` в части аутентификации.

Нужно добавить в Gateway:
- Эндпоинт регистрации через email/телефон (не Telegram)
- Push-токены (FCM/APNs) для уведомлений о матче
- Refresh token эндпоинт
