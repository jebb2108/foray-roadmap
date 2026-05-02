---
tags: [architecture, existing-system, reference]
---

# Архитектура chat.lllang.site

← [[🗺️ Foray — Карта проекта (Обновлено)]]

## Обзор

`chat.lllang.site` — веб-приложение для парных разговорных сессий на английском языке. Реализован как тонкий прокси-слой поверх внутреннего [[Gateway Сервис|Gateway]].

## Технический стек

| Слой | Технология |
|------|------------|
| Backend | Python 3.13 + FastAPI + Uvicorn |
| Frontend | Vanilla HTML/CSS/JS (без фреймворков) |
| Инфраструктура | Docker Compose (app + Nginx) |
| Реалтайм | WebSocket (FastAPI native) |
| Шифрование | Fernet (симметричное) |
| Email | Resend |
| Платежи | YooKassa |

## Структура сервиса

```
web-app-service/
├── src/
│   ├── main.py              # Точка входа FastAPI
│   ├── config.py            # Конфиг через env vars
│   ├── dependencies.py      # DI: connection service singleton
│   ├── routers/
│   │   ├── waiting_room.py  # /api/user/*
│   │   ├── matchmaking.py   # /api/worker/*
│   │   ├── websockets.py    # /api/sockets/ws/chat
│   │   └── payments.py      # /api/payments/webhook/*
│   ├── services/
│   │   └── connection.py    # In-memory WebSocket state
│   └── validators/
│       └── tokens.py        # JWT create/validate
└── front/
    ├── chat/                # Зал ожидания + чат
    └── main/                # Лендинг + email templates
```

## Субдомены

| Субдомен | Назначение |
|----------|------------|
| `lllang.site` | Лендинг |
| `chat.lllang.site` | Зал ожидания + чат (основной продукт) |

## Ключевые зависимости

- **[[Gateway Сервис]]** — хранит все данные: пользователи, матчи, сообщения, платежи
- **[[Матчмейкинг]]** — алгоритм подбора пар
- **[[WebSocket Чат]]** — реалтайм коммуникация
- **[[Подписки и Платежи]]** — монетизация

## Слабые места (для Foray)

- In-memory WebSocket state → теряется при рестарте
- Нет собственной системы аккаунтов
- Секрет JWT (`lambo`) — нужно сменить
- Нет мобильного нативного опыта
- Polling каждую секунду для матчмейкинга (неэффективно для мобильных)
