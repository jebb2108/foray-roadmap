---
tags: [websocket, chat, realtime, existing-system, core-feature]
---

# WebSocket Чат

← [[🗺️ Foray — Карта проекта]] | Связан: [[Жизненный цикл сессии]]

## Подключение

```
wss://chat.lllang.site/api/sockets/ws/chat?user_id=<id>&token=<JWT>
```

При подключении сервер:
1. Декодирует JWT (`convert_token`)
2. Проверяет соответствие `user_id` и токена (`validate_access`)
3. Регистрирует соединение в `ConnectionService`
4. Загружает историю сообщений из Gateway (расшифровывает Fernet)
5. Отправляет `partner_status` и `match_info`

## Типы сообщений (сервер → клиент)

| Тип | Описание |
|-----|----------|
| `partner_status` | Онлайн/оффлайн статус партнёра |
| `message_history` | История предыдущих сообщений (расшифрованная) |
| `match_info` | Данные партнёра: ник, возраст, пол, интересы, интро |
| `match_ready` | Оба нажали Accept — сессия начинается |
| `session_started` | Таймер запущен, передаётся `expires_at` |
| `session_ended` | Сессия завершена (таймаут или пользователь вышел) |
| `new_message` | Новое сообщение от партнёра |
| `typing` | Партнёр печатает |

## Типы сообщений (клиент → сервер)

| Тип | Описание |
|-----|----------|
| `confirm` | Пользователь нажал Accept |
| `cancel` | Пользователь нажал Decline / Cancel |
| `timeout` | Истёк таймер подтверждения (60 сек) |
| `message` | Отправить сообщение |
| `typing` | Индикатор печати |

## In-memory состояние (ConnectionService)

```python
active_connections: Dict[room_id, Dict[nickname, WebSocket]]
sessions: Dict[WebSocket, {
    room_id: str,
    is_ready: bool,
    nickname: str,
    token: dict
}]
```

**Важно:** состояние живёт в памяти процесса. При рестарте — потеря всех активных сессий.

## Шифрование сообщений

- Алгоритм: **Fernet** (симметричное AES-128-CBC + HMAC-SHA256)
- Ключ: `FERNET_KEY` env var
- История сообщений хранится в Gateway в зашифрованном виде
- При загрузке истории сервер расшифровывает перед отправкой клиенту

## Для Foray: что учесть

- Заменить in-memory state на **Redis** для масштабируемости
- Добавить **reconnection logic** на мобильном (сеть прерывается чаще)
- Push notification при `new_message` когда приложение в фоне
- Поддержка **offline-режима** — локальный кэш истории
