---
tags: [backend, analysis, existing-system, architecture]
---

# Анализ Backend chat.lllang.site

← [[🗺️ Foray — Карта проекта (Обновлено)]]

## Текущая архитектура

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend (Web)                            │
│                                                                   │
│  Vanilla HTML/CSS/JS (без фреймворков)                          │
│  - /chat → Зал ожидания + Chat                                 │
│  - /main → Лендинг                                              │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                    WebSocket & REST
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│        web-app-service (FastAPI + Uvicorn)                      │
│  172.20.0.2:8000 (докер-контейнер)                              │
│                                                                   │
│  ├── Routers:                                                    │
│  │   ├── waiting_room.py     → /api/user/*                      │
│  │   ├── matchmaking.py      → /api/worker/*                    │
│  │   ├── websockets.py       → /api/sockets/ws/chat             │
│  │   └── payments.py         → /api/payments/webhook/*          │
│  │                                                                │
│  ├── Services:                                                   │
│  │   └── connection.py       → In-memory WebSocket state        │
│  │                                                                │
│  └── Validators:                                                 │
│      └── tokens.py          → JWT create/validate               │
│                                                                   │
│  Функции ВЕБ-сервиса:                                           │
│  ✓ Выдаёт JWT (генерирует локально)                            │
│  ✓ Управляет WebSocket соединениями (in-memory)                │
│  ✓ Отправляет email через Resend                               │
│  ✓ Проксирует запросы к Gateway                                │
│  ✓ Валидирует payment webhooks YooKassa                         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                    HTTP (REST)
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│           Gateway Service (217.149.29.173:1000)                  │
│                                                                   │
│  ├── /api/*         → Пользователи, профили, история            │
│  ├── /api/worker/*  → Матчмейкинг, очередь, сессии             │
│  ├── /api/payments/*→ Подписки, YooKassa интеграция             │
│  └── /api/ai/*      → AI функции                                │
│                                                                   │
│  Функции GATEWAY:                                                │
│  ✓ Хранит ВСЕ данные (главная БД)                              │
│  ✓ Матчмейкинг алгоритм (Worker)                               │
│  ✓ Управление подписками                                        │
│  ✓ Обработка платежей                                           │
│  ✓ AI интеграции                                                │
│                                                                   │
│  Содержит:                                                       │
│  └─ PostgreSQL база данных                                      │
│  └─ Worker-сервис (фоновые задачи)                              │
│  └─ AI-сервис (рекомендации)                                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
    PostgreSQL         YooKassa           Resend
      (DB)           (платежи)            (email)
```

## Ключевые компоненты

### 1. Веб-сервис (web-app-service)

**Роль:** Тонкий прокси-слой и реалтайм сервер

| Компонент | Функция | Технология |
|-----------|---------|-----------|
| **JWT Manager** | Генерирует/валидирует токены | PyJWT |
| **WebSocket Handler** | Управляет соединениями, распределяет сообщения | FastAPI WebSockets |
| **Connection Service** | In-memory состояние активных сессий | Dict в памяти процесса |
| **Email Service** | Отправляет email верификации, уведомления | Resend API |
| **Payment Handler** | Обрабатывает webhook'и YooKassa | FastAPI router |
| **Nginx** | Reverse proxy, SSL, статика | Nginx config |

**Уязвимости текущей реализации:**
- ❌ In-memory state теряется при рестарте
- ❌ JWT секрет захардкодирован ("lambo")
- ❌ Нет горизонтального масштабирования (только 1 инстанс)
- ❌ Polling каждую секунду неэффективен
- ❌ Нет система аккаунтов (сессия основана на IP + cookie)

### 2. Gateway Service

**Роль:** Центральный backend, хранит ВСЕ данные

| Компонент | Функция |
|-----------|---------|
| **API Gateway** | Маршрутизация запросов |
| **User Service** | Регистрация, профили, верификация |
| **Matchmaking Engine** | Алгоритм подбора пар |
| **Session Manager** | Управление сессиями, тайм-ауты |
| **Payment Service** | Интеграция с YooKassa, подписки |
| **Message Store** | Хранение/шифрование сообщений (Fernet) |
| **AI Service** | Рекомендации, анализ |
| **Worker** | Фоновые задачи, матчмейкинг |

**Структура данных в БД (примерно):**

```sql
-- Пользователи
users (id, email, password_hash, nickname, age, gender, interests, intro, created_at)

-- Подписки
subscriptions (id, user_id, status, due_to, plan_type, payment_id)

-- Сессии/матчи
matches (id, user1_id, user2_id, room_id, status, created_at, ended_at)

-- Сообщения (зашифрованы Fernet)
messages (id, room_id, sender_id, content_encrypted, created_at)

-- История очереди
queue_history (id, user_id, queue_time, match_found_at)
```

### 3. Внешние сервисы

| Сервис | Использование | Интеграция |
|--------|--------------|-----------|
| **YooKassa** | Обработка платежей | Webhook + REST API |
| **Resend** | Отправка email | REST API |
| **PostgreSQL** | Хранение данных | Прямая БД |

## Критический путь обработки запросов

### Сценарий 1: Регистрация + Вход в очередь

```
1. Web Client
   POST /api/user (ник, возраст, интересы, email)
   ↓
2. web-app-service
   - Валидирует входные данные
   - Проксирует к Gateway
   ↓
3. Gateway
   - Создаёт профиль в БД
   - Отправляет email верификацию (через сервис)
   - Возвращает user_id
   ↓
4. web-app-service
   - Генерирует JWT
   - Возвращает клиенту
   ↓
5. User входит в очередь
   POST /api/worker/match/toggle
   ↓
6. web-app-service → Gateway
   - Проверяет подписку
   - Добавляет в очередь
   ↓
7. Gateway
   - Пользователь в очереди
   - Worker начинает матчмейкинг
```

### Сценарий 2: Чат в реальном времени

```
1. Web Client
   WebSocket подключение
   wss://chat.lllang.site/api/sockets/ws/chat?user_id=X&token=JWT
   ↓
2. web-app-service (WebSocket Handler)
   - Декодирует JWT
   - Валидирует соответствие user_id
   - Загружает историю из Gateway (расшифровывает)
   - Регистрирует соединение в ConnectionService
   ↓
3. User A отправляет сообщение
   JSON { type: "message", content: "Hi" }
   ↓
4. web-app-service
   - Шифрует сообщение (Fernet)
   - Сохраняет в Gateway
   - Отправляет User B через WebSocket
   ↓
5. User B получает в реальном времени
```

### Сценарий 3: Платёж

```
1. User нажимает "Купить подписку"
   POST /api/payments/purchase
   ↓
2. web-app-service → Gateway
   - Создаёт платёж в YooKassa
   - Возвращает payment_url
   ↓
3. User переходит на YooKassa и платит
   ↓
4. YooKassa отправляет webhook
   POST /api/payments/webhook/yookassa
   ↓
5. web-app-service
   - Валидирует подпись webhook
   - Обновляет статус в Gateway
   - Активирует подписку в БД
```

## Проблемы для масштабирования

| Проблема | Причина | Решение |
|----------|---------|---------|
| In-memory state | WebSocket состояние в памяти процесса | Redis для распределённого state |
| Нет горизонтального масштабирования | Один инстанс web-app-service | Load balancer + несколько инстансов |
| Polling матчмейкинга | Клиент опрашивает каждую секунду | Push notifications (WebSocket или FCM) |
| Потеря соединений | Если сервер упадёт, все потеряют сессию | Connection recovery + persistence |
| JWT тайм-ауты | Нет refresh token | Реализовать refresh token + sliding window |
| Шифрование Fernet | Одноразовый ключ для всех | Rotation policy + key management |

## Рекомендации для Foray

✅ **Взять из chat.lllang.site:**
- Структуру WebSocket обработки
- Систему шифрования сообщений (Fernet)
- Матчмейкинг алгоритм
- Email верификацию
- Интеграцию YooKassa

🔄 **Переработать:**
- JWT система → собственная система аккаунтов (email + пароль)
- In-memory state → Redis
- Polling → Push notifications + WebSocket
- Однопроцессная → Многопроцессная архитектура

🆕 **Добавить:**
- Система рейтинга партнёров
- Push notifications (FCM + APNs)
- Offline-режим (локальный кэш)
- Microservices для мессенджера
- Система друзей / контактов
- Profile enrichment (AI recommendations)
