---
tags: [foray, architecture, mobile-first]
---

# Архитектура Foray

← [[🗺️ Foray — Карта проекта (Обновлено)]]

Foray — мобильное приложение для поиска единомышленников через текстовый чат по общим интересам.

---

## Общая схема

```
┌─────────────────────────────────────────────────────────────────┐
│                          КЛИЕНТ                                  │
│                  React Native  (iOS + Android)                   │
│                                                                   │
│  Экраны:  Auth · Profile · Queue · Chat · Rating · Settings     │
│                                                                   │
│  Токены:  expo-secure-store                                      │
│  Кэш:     AsyncStorage / MMKV                                    │
│  WS:      socket.io-client                                       │
│  Push:    FCM (Android)  /  APNs (iOS)                          │
└────────────────────────┬────────────────────────────────────────┘
                         │  HTTPS  +  WSS
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                          СЕРВЕР                                  │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │           Gateway  (Nginx + FastAPI  :8000)               │  │
│  │   JWT-валидация · Rate limiting · Маршрутизация           │  │
│  └──────┬──────────────┬────────────┬────────────┬──────────┘  │
│         │              │            │            │              │
│         ▼              ▼            ▼            ▼              │
│  ┌───────────┐  ┌──────────┐  ┌─────────┐  ┌──────────┐       │
│  │   Auth    │  │   Core   │  │  Chat   │  │ Payment  │       │
│  │  Service  │  │  Service │  │ Service │  │ Service  │       │
│  │  :3001    │  │  :3002   │  │  :3003  │  │  :3004   │       │
│  └───────────┘  └──────────┘  └─────────┘  └──────────┘       │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     Infrastructure                        │  │
│  │            PostgreSQL  ·  Redis  ·  S3 / MinIO            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│  Внешние:  Resend (email)  ·  FCM / APNs (push)  ·  YooKassa   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Клиент

**Стек:** React Native + Expo managed workflow

| Слой | Инструмент |
|------|-----------|
| Навигация | expo-router (file-based) |
| Серверное состояние | TanStack Query |
| Локальное состояние | Zustand |
| HTTP-клиент | axios (interceptor для refresh) |
| WebSocket | socket.io-client |
| Токены | expo-secure-store |
| Кэш/офлайн | AsyncStorage + MMKV |
| Push | expo-notifications |
| Медиа | expo-image-picker |

**Структура экранов:**

```
App
├── Auth
│   ├── Welcome
│   ├── Login
│   ├── Register
│   └── EmailVerify
│
└── Main  (требует авторизации)
    ├── Home  ← Queue Screen (точка входа)
    ├── Chat
    │   ├── ChatList
    │   └── ChatScreen
    ├── Profile
    │   ├── MyProfile
    │   └── EditProfile
    ├── Rating       ← появляется после сессии
    └── Settings
        ├── Subscription
        └── Account
```

**Жизненный цикл токенов:**
```
Login → { access_token (15 мин), refresh_token (30 дней) }
             │
             ├─ access_token → SecureStore
             └─ refresh_token → SecureStore (httpOnly аналог)

axios interceptor:
  ошибка 401 → POST /api/auth/refresh → новый access_token → retry
```

---

## Сервер

### Gateway  (:8000)

Единая точка входа для всех запросов.

```
Входящий запрос
    │
    ├── Nginx: TLS, статические ресурсы
    │
    ▼
FastAPI (gateway-service)
    │
    ├── JWT middleware
    │     Проверяет Authorization: Bearer <token>
    │     Публичный ключ RS256 (только верификация, не выдача)
    │     Исключения: /api/auth/register · /api/auth/login · /api/auth/refresh
    │
    ├── Rate limiting  (Redis)
    │     100 req/min  — per IP
    │     1000 req/min — per user_id
    │
    └── Reverse proxy  →  нужный сервис (httpx async)

Таблица маршрутов:
  /api/auth/*      →  Auth Service    :3001
  /api/users/*     →  Core Service    :3002
  /api/match/*     →  Core Service    :3002
  /api/ratings/*   →  Core Service    :3002
  /api/messages/*  →  Chat Service    :3003
  /api/ws/*        →  Chat Service    :3003
  /api/payments/*  →  Payment Service :3004
```

---

### Auth Service  (:3001)

Аутентификация и управление сессиями.

```
POST  /api/auth/register          регистрация
POST  /api/auth/login             вход
POST  /api/auth/logout            выход (инвалидация refresh)
POST  /api/auth/refresh           обновить access_token
POST  /api/auth/verify-email      подтверждение email
POST  /api/auth/forgot-password   запрос сброса пароля
POST  /api/auth/reset-password    сброс пароля

Схема БД (schema: auth):
  users          (id, email, password_hash, nickname, verified, created_at)
  refresh_tokens (token_hash, user_id, expires_at, revoked)
  email_codes    (code_hash, user_id, type, expires_at)

Redis:
  auth:blacklist:{jti}    — отозванные access-токены (TTL = exp)
  auth:code:{user_id}     — rate limit на отправку кодов

Зависимости:
  PostgreSQL · Redis · Resend
```

---

### Core Service  (:3002)

Профили, матчмейкинг, рейтинги.

**Профили:**
```
GET    /api/users/me          мой профиль
PUT    /api/users/me          обновить (интересы, аватар, bio)
GET    /api/users/:id         профиль пользователя (публичный)
DELETE /api/users/me          удалить аккаунт
```

**Матчмейкинг:**
```
POST   /api/match/join        войти в очередь
DELETE /api/match/leave       выйти из очереди
GET    /api/match/status      позиция и размер очереди
GET    /api/matches/:id       данные матча (room_id, партнёр)
POST   /api/matches/:id/end   завершить сессию

Воркер  (asyncio background task, каждые 5 сек):
  1. ZRANGEBYSCORE match:queue → первые N пользователей
  2. Для каждой пары считаем score:
       interests_sim  = Jaccard(A.interests, B.interests)  × 0.5
       rating_sim     = 1 − |A.rating − B.rating| / 5      × 0.3
       wait_bonus     = min(wait_sec / 300, 1.0)            × 0.2
  3. Лучшая пара с score > 0.5 → создать match в БД
  4. PUBLISH match:{user_id} для обоих
  5. Push: "Нашли партнёра — подтвердите!"
```

**Рейтинги:**
```
POST   /api/ratings           оценить партнёра (после сессии)
GET    /api/ratings/user/:id  оценки пользователя

После оценки:
  → пересчёт avg_rating в user_stats
  → ≥ 3 оценок ≤ 2★  → флаг "под проверкой"
  → ≥ 5 оценок ≤ 2★  → shadow ban (скрытие из очереди)
```

**Схема БД (schema: core):**
```
profiles     (user_id, nickname, avatar_url, bio, avg_rating, ban_status)
interests    (user_id, tag)
matches      (id, user1_id, user2_id, room_id, status, created_at, ended_at)
ratings      (id, from_user_id, to_user_id, match_id, stars, comment, created_at)
user_stats   (user_id, total_matches, flag_count, shadow_banned)
```

**Redis:**
```
match:queue          Sorted Set  (score = join_timestamp)
match:{user_id}      Pub/Sub канал уведомлений матча
profile:{user_id}    кэш профиля (TTL 5 мин)
```

---

### Chat Service  (:3003)

Реальный времени чат + история сообщений.

**WebSocket:**
```
WS  /api/ws/chat?token=<JWT>

Клиент → сервер:
  { type: "message",  content: "...", attachments: [] }
  { type: "typing" }
  { type: "read",     message_id: "..." }

Сервер → клиент:
  { type: "message",        ...payload }
  { type: "typing",         user_id }
  { type: "read",           message_id }
  { type: "partner_status", status: "online|offline" }
  { type: "match_ready",    room_id, partner }
  { type: "session_ended" }

Масштабирование:
  Каждый инстанс Chat Service подписан на Redis Pub/Sub
  channel room:{room_id} → рассылает всем подключённым клиентам
```

**REST:**
```
GET    /api/messages/:room_id     история (cursor-based pagination)
POST   /api/messages/upload       загрузить медиафайл → S3/MinIO
DELETE /api/messages/:id          soft delete (только свои, 24 ч)
POST   /api/sessions/:id/end      завершить сессию
```

**Схема БД (schema: chat):**
```
chat_rooms     (id, user1_id, user2_id, created_at, ended_at, is_active)
messages       (id, room_id, sender_id, content, attachments[], created_at, deleted_at)
read_receipts  (message_id, user_id, read_at)
```

**Redis:**
```
room:{room_id}       Pub/Sub канал комнаты
session:{user_id}    активное WS-соединение (TTL 1 мин, heartbeat)
```

---

### Payment Service  (:3004)

Подписки через YooKassa. Основа — tutor2/payment-service.

```
GET   /api/payments/link         получить ссылку на оплату (YooKassa)
GET   /api/payments/due_to       дата окончания подписки
POST  /api/payments/activate     активировать подписку
POST  /api/payments/deactivate   деaktivировать подписку
POST  /api/payments/yookassa     webhook от YooKassa

Фоновая задача:
  Scheduler (из tutor2) — ежедневно 00:00
  → проверить истёкшие подписки → продлить / деaktivировать

Схема БД (schema: payment):
  subscriptions  (user_id, status, activated_at, due_to)
  payments       (id, user_id, amount, yookassa_id, status, created_at)
```

---

## Инфраструктура

### PostgreSQL

Единый инстанс. Изоляция через схемы — просто, без сложности мульти-DB.

```sql
CREATE SCHEMA auth;     -- Auth Service
CREATE SCHEMA core;     -- Core Service
CREATE SCHEMA chat;     -- Chat Service
CREATE SCHEMA payment;  -- Payment Service
```

Каждый сервис подключается с `search_path=<schema>` — видит только свои таблицы.

### Redis

```
Namespace       Тип           Назначение
──────────────────────────────────────────────────
auth:*          String/Set    Blacklist токенов, коды верификации
ratelimit:*     String        Счётчики rate limit (с TTL)
match:queue     Sorted Set    Очередь матчмейкинга
match:{uid}     Pub/Sub       Уведомления о матче
room:{id}       Pub/Sub       Реальный времени чат
session:{uid}   String        Heartbeat WS-соединения
profile:{uid}   String (JSON) Кэш профиля
```

### S3 / MinIO

Аватарки и медиафайлы в чате. MinIO для self-hosted, S3 для cloud.

```
Buckets:
  foray-avatars   — аватарки пользователей (public read)
  foray-chat      — медиафайлы в чатах (presigned URLs)
```

---

## Ключевые флоу

### Регистрация

```
Клиент                     Auth Service
  │                              │
  ├─ POST /api/auth/register ───►│
  │  { email, pwd, nick,         │── bcrypt(pwd) → PostgreSQL
  │    interests }               │── email_code → Redis (5 мин)
  │◄─ { status:"verify_email" }──┤── Resend: письмо верификации
  │                              │
  ├─ POST /api/auth/verify-email►│
  │  { code }                    │── code valid → verified=true
  │◄─ { access_token,            │
  │     refresh_token }──────────┤
  │                              │
  └─ → Core: PUT /api/users/me   (заполнить профиль, загрузить аватар)
```

### Матчмейкинг

```
Клиент            Gateway         Core Service          Redis
  │                  │                 │                   │
  ├─ POST join ──────►├─── proxy ──────►│                   │
  │◄─ { queued } ─────┤                 ├─ ZADD queue ──────►│
  │                   │                 │                   │
  │  [воркер каждые 5 сек]              │                   │
  │                   │                 ├─ ZRANGE ──────────►│
  │                   │                 │◄── [user_a, user_b]─┤
  │                   │                 ├─ score > 0.5?      │
  │                   │                 ├─ INSERT match      │
  │◄── Push: "Нашли!" │                 ├─ PUBLISH match:uid►│
  │                   │                 │                   │
  ├─ GET match status►│                 │                   │
  │◄─ { room_id }──────┤                │                   │
  │                   │                 │                   │
  └─── WS /api/ws/chat ─────────────────────────────────────►│
```

### Чат

```
User A                  Chat Service                    User B
  │                          │                             │
  ├─ WS connect ────────────►│◄──── WS connect ────────────┤
  │                          ├─ подписка room:{id} (Redis)  │
  │                          │                             │
  ├─ { type:"message",       │                             │
  │    content:"Привет" } ──►│                             │
  │                          ├─ INSERT messages            │
  │                          ├─ PUBLISH room:{id} ─────────►│
  │                          │                   │         │
  │                          │  User B offline:  │         │
  │                          ├─ Push notification►         │
  │                          │                             │
  │◄── { type:"read" }───────┤◄──── { type:"read" }────────┤
```

### Завершение сессии и рейтинг

```
Клиент                     Chat / Core Service
  │                               │
  ├─ POST /api/sessions/:id/end ─►│
  │                               ├─ ended_at = now
  │                               ├─ PUBLISH room:{id}: session_ended
  │                               │
  │◄─ WS: { type:"session_ended"}─┤
  │                               │
  │  (UI: Rating Screen)          │
  ├─ POST /api/ratings ──────────►│
  │  { match_id, stars, comment } ├─ INSERT ratings
  │                               ├─ UPDATE user_stats.avg_rating
  │◄─ { ok } ─────────────────────┤
```

---

## Развёртывание  (Docker Compose)

```yaml
services:
  nginx:
    image: nginx:alpine
    ports: ["443:443", "80:80"]
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/certs
    depends_on: [gateway]

  gateway:
    build: ./gateway-service
    environment:
      AUTH_URL:        http://auth:3001
      CORE_URL:        http://core:3002
      CHAT_URL:        http://chat:3003
      PAYMENT_URL:     http://payment:3004
      REDIS_URL:       redis://redis:6379
      JWT_PUBLIC_KEY:  ${JWT_PUBLIC_KEY}
    depends_on: [redis]

  auth:
    build: ./auth-service
    environment:
      DATABASE_URL:    ${PG_URL}?options=-csearch_path%3Dauth
      REDIS_URL:       redis://redis:6379
      JWT_PRIVATE_KEY: ${JWT_PRIVATE_KEY}
      RESEND_API_KEY:  ${RESEND_API_KEY}
    depends_on: [postgres, redis]

  core:
    build: ./core-service
    environment:
      DATABASE_URL:    ${PG_URL}?options=-csearch_path%3Dcore
      REDIS_URL:       redis://redis:6379
      FCM_SERVER_KEY:  ${FCM_SERVER_KEY}
    depends_on: [postgres, redis]

  chat:
    build: ./chat-service
    environment:
      DATABASE_URL:    ${PG_URL}?options=-csearch_path%3Dchat
      REDIS_URL:       redis://redis:6379
      S3_ENDPOINT:     http://minio:9000
      S3_ACCESS_KEY:   ${MINIO_USER}
      S3_SECRET_KEY:   ${MINIO_PASSWORD}
    depends_on: [postgres, redis, minio]

  payment:
    build: ./payment-service      # tutor2/payment-service
    environment:
      DATABASE_URL:           ${PG_URL}?options=-csearch_path%3Dpayment
      YOOKASSA_SHOP_ID:       ${YOOKASSA_SHOP_ID}
      YOOKASSA_SECRET_KEY:    ${YOOKASSA_SECRET_KEY}
    depends_on: [postgres]

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB:       foray
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes: [postgres_data:/var/lib/postgresql/data]

  redis:
    image: redis:7-alpine
    volumes: [redis_data:/data]

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER:     ${MINIO_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_PASSWORD}
    volumes: [minio_data:/data]

volumes:
  postgres_data:
  redis_data:
  minio_data:
```

---

## Что берём из tutor2

| Компонент | Источник в tutor2 | Изменения |
|-----------|------------------|-----------|
| Gateway | `gateway-service` | Новые роуты, JWT middleware |
| Auth | `auth-service` | Расширить: interests, avatar |
| Payment | `payment-service` | Без изменений (YooKassa + Scheduler) |
| Matchmaking API | `gateway-service/worker_router` | Перенести в Core Service |
| User API | `gateway-service/users_router` | Адаптировать под Foray |
| DB pattern | `db-storage-service` | asyncpg + SQLAlchemy (без RabbitMQ) |
| Push | нет | Новое: FCM/APNs через Core Service |
| Chat WebSocket | `web-app-service` | Перенести в Chat Service |
| Rating | нет | Новое в Core Service |
