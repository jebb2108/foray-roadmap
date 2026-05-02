---
tags: [foray, architecture, new-design, mobile-first]
---

# Переработанная Архитектура Foray

← [[🗺️ Foray — Карта проекта]]

## Новая концепция

**Было:** Веб-сайт для парных разговорных сессий на английском языке  
**Стало:** Мобильное приложение для поиска единомышленников через текстовый чат на основе общих интересов

## Высокоуровневая архитектура

```
┌──────────────────────────────────────────────────────────────────────┐
│                   FORAY Mobile Apps (iOS/Android)                     │
│                                                                        │
│  ├── Authentication Flow (email/password, social)                     │
│  ├── Profile Management (interests, bio, avatar)                      │
│  ├── Matchmaking Screen (visual queue, interests filter)              │
│  ├── Messaging Interface (real-time chat, typing indicators)          │
│  ├── Partner Rating (1-5 stars + comment)                            │
│  ├── Notifications (Push: FCM on Android, APNs on iOS)               │
│  └── Offline Mode (local message cache)                              │
└───────────────────────────────┬────────────────────────────────────────┘
                                │
         ┌──────────────────────┼──────────────────────┐
         │                      │                      │
         ▼                      ▼                      ▼
   REST API (HTTP)      WebSocket (Real-time)   Push Notifications
         │                      │                      │
┌────────▼──────────────────────▼──────────────────────▼─────────────┐
│                  API Gateway & Router                               │
│           (Load Balancer, Request Routing)                          │
│                                                                      │
│  - Rate limiting                                                    │
│  - Request validation                                              │
│  - Centralized logging                                             │
│  - API versioning (/v1/, /v2/)                                     │
└────────┬───────────────────────────────────────────────────────────┘
         │
    ┌────┴────────────────────────────────────────────┐
    │                                                  │
    ▼                                                  ▼
┌────────────────────────┐                ┌──────────────────────────┐
│   Authentication       │                │  Message Broker          │
│   Service              │                │  (Redis Pub/Sub)         │
│                        │                │                          │
│ - Login/Register       │                │ - WebSocket events       │
│ - Token refresh        │                │ - Notifications          │
│ - Email verification   │                │ - Match events           │
│ - Password reset       │                │                          │
│                        │                │                          │
│ JWT + Refresh Token    │                │ Redis Streams            │
└────────┬───────────────┘                └──────────────────────────┘
         │
    ┌────┴──────────────────────────────────────────────────┐
    │                                                        │
    ▼                                                        ▼
┌──────────────────────────────┐                ┌──────────────────────┐
│   Microservices              │                │  Shared Services     │
│   (Business Logic)           │                │                      │
│                              │                │ ├── Redis Cache      │
│ ├── User Service             │                │ │   (profiles,       │
│ │   (profiles, interests)    │                │ │    interests)      │
│ │                            │                │ │                    │
│ ├── Matchmaking Service      │                │ ├── PostgreSQL       │
│ │   (queue, algorithm)       │                │ │   (primary store)  │
│ │                            │                │ │                    │
│ ├── Messaging Service        │                │ ├── Elasticsearch    │
│ │   (chat, history)          │                │ │   (message search) │
│ │                            │                │ │                    │
│ ├── Rating Service           │                │ ├── S3 / MinIO       │
│ │   (partners, reviews)      │                │ │   (avatars, media) │
│ │                            │                │ │                    │
│ ├── Notification Service     │                │ └── Resend           │
│ │   (FCM, APNs, email)       │                │    (email service)  │
│ │                            │                │                     │
│ ├── Payment Service          │                │                     │
│ │   (YooKassa, subscriptions)│                │                     │
│ │                            │                │                     │
│ └── WebSocket Service        │                │                     │
│     (real-time, Redis store) │                │                     │
└──────────────────────────────┘                └──────────────────────┘
```

## Новый концепт: Основные отличия

| Аспект | chat.lllang.site | Foray |
|--------|-----------------|-------|
| **Платформа** | Веб | Мобильное (iOS/Android) |
| **Цель сессии** | Обучение английскому | Поиск единомышленников |
| **Время сессии** | 15 минут (зафиксировано) | Не ограничено или дольше |
| **Матчмейкинг** | По языковому уровню | По интересам + рейтингу |
| **История контактов** | Не сохраняется | Друзья / чёрный список |
| **Рейтинг** | Нет | 1-5 звёзд после сессии |
| **Notifications** | Polling | Push (FCM/APNs) |
| **Offline** | Невозможно | Локальный кэш сообщений |
| **Платежи** | YooKassa | YooKassa + Apple Pay + Google Pay |

## Концептуальные диаграммы по функциям

### 1. Регистрация и Профиль

```
User opens app
    │
    ├─→ Not authenticated?
    │    └─→ Registration Screen
    │        │
    │        ├─ Email input
    │        ├─ Password input
    │        ├─ Nickname input
    │        └─ Interests selection (tags)
    │        │
    │        ▼
    │    POST /auth/register
    │        │
    │        ▼
    │    Authentication Service
    │        ├─ Validate input
    │        ├─ Hash password
    │        ├─ Create user in PostgreSQL
    │        └─ Send verification email
    │        │
    │        ▼
    │    Return JWT + refresh_token
    │        │
    │        ▼
    │    Email Verification Screen
    │        │
    │        ▼
    │    POST /auth/verify?code=XXX
    │        │
    │        ▼
    │    ✅ Authenticated
    │
    └─→ Authenticated?
         └─→ Profile Setup (avatar, bio, detailed interests)
             │
             ▼
         PUT /user/profile
             │
             ▼
         Main Screen (Matchmaking Queue)
```

### 2. Матчмейкинг (Поиск партнёра)

```
User clicks "Find someone"
    │
    ▼
POST /matchmaking/join-queue
    │
    ├─ Check subscription (active?)
    ├─ Check profile (complete?)
    └─ Add to Redis queue
    │
    ▼
UI: Waiting Room
    │
    ├─ Show animated queue (5 people waiting...)
    ├─ Show your interests
    ├─ Show filters applied
    └─ Option to adjust interests
    │
    ▼
[Behind the scenes: Matchmaking Service]
    │
    ├─ Algorithm: similarity(interests) + rating
    │
    └─ When match found:
         │
         ├─ Create room_id
         ├─ Create match in PostgreSQL
         ├─ Publish to Redis Pub/Sub
         │
         ▼
    Push Notification (FCM/APNs)
    "Found a match! Accept?"
         │
         ├─→ User 1 taps notification
         │    └─→ WebSocket connect + sends "confirm"
         │
         ├─→ User 2 taps notification
         │    └─→ WebSocket connect + sends "confirm"
         │
         ▼
    Both confirmed?
         │
         └─→ YES: match_ready event
                 │
                 ▼
             Messaging Screen opens
                 │
                 ├─ Show partner info
                 ├─ Show chat history
                 ├─ Ready for messaging
                 │
                 ▼
             WebSocket: wss://.../ws/chat
                 (Real-time messaging begins)
```

### 3. Реальный Чат

```
User in chat
    │
    ├─ WebSocket connected
    │
    ├─ Can send messages
    │    └─ {type: "message", content: "Hi!"}
    │       │
    │       ▼
    │    Messaging Service
    │       ├─ Encrypt message (Fernet)
    │       ├─ Save to PostgreSQL + Elasticsearch
    │       ├─ Broadcast via Redis Pub/Sub
    │       │
    │       ▼
    │    Partner receives in real-time
    │       │
    │       ▼
    │    If app in background → Push notification
    │
    ├─ Can type
    │    └─ {type: "typing"}
    │       → Partner sees indicator
    │
    ├─ Can upload media (images)
    │    └─ POST /messages/upload
    │       │
    │       ▼
    │    Save to S3/MinIO
    │    Return signed URL
    │    Encrypt metadata
    │
    └─ Session ends when?
         │
         ├─ User exits chat (soft end)
         ├─ Both go offline (hard end)
         ├─ 24 hours pass (automatic)
         │
         ▼
    POST /sessions/{room_id}/end
         │
         ▼
    Rating Screen appears
         │
         ├─ 1-5 stars
         ├─ Optional comment
         ├─ Report? (inappropriate)
         │
         ▼
    POST /ratings/create
         │
         ▼
    Partner added to Friends / Blocked list
```

### 4. Рейтинговая система

```
After chat ends
    │
    ▼
Rating Screen
    │
    ├─ Show partner avatar + name
    ├─ "How was this conversation?"
    │
    ├─ Star rating (1-5)
    ├─ Optional text comment
    ├─ "Report user" button
    │
    ▼
POST /ratings/create
    │
    ├─ Validate user participated
    ├─ Save rating to PostgreSQL
    ├─ Update partner's average_rating
    ├─ Update matchmaking algorithm weights
    │
    ▼
✅ Saved
    │
    ├─ Can view partner's profile + reviews
    ├─ Add to "Liked" or "Blocked"
    │
    ▼
Matchmaking uses rating:
    │
    ├─ Higher rated users matched first
    ├─ Blocked users not matched with each other
    └─ Low-rated accounts get restricted (warning → shadowban)
```

### 5. Система аккаунтов и безопасность

```
Authentication Flow:

1. Registration
   └─ POST /auth/register (email, password, nickname, interests)
      │
      ├─ Hash password with bcrypt
      ├─ Create user in PostgreSQL
      ├─ Generate email verification code
      ├─ Send via Resend
      │
      ▼
      Return: { status: "verify_email" }

2. Email Verification
   └─ POST /auth/verify-email (code)
      │
      ├─ Check code validity (5 min expiry)
      ├─ Mark user as verified
      │
      ▼
      Return: { access_token, refresh_token }

3. Login
   └─ POST /auth/login (email, password)
      │
      ├─ Verify password
      ├─ Generate JWT (access_token) + refresh_token
      │
      ▼
      Return: { access_token, refresh_token, expires_in }

4. Token Management
   └─ Access token (JWT)
      ├─ Short-lived (15 min)
      ├─ Signed with RS256 (asymmetric)
      │
      └─ Refresh token
         ├─ Longer-lived (30 days)
         ├─ Stored in secure HTTP-only cookie
         │
      When access token expires:
      └─ POST /auth/refresh
         │
         ├─ Validate refresh token
         ├─ Issue new access_token
         │
         ▼
         Return: { access_token }

5. Password Reset
   └─ POST /auth/forgot-password (email)
      │
      ├─ Generate reset code
      ├─ Send via Resend
      │
      ▼
      POST /auth/reset-password (code, new_password)
      │
      ├─ Verify code
      ├─ Hash new password
      ├─ Update in PostgreSQL
      │
      ▼
      ✅ Password changed
```

## Микросервисная архитектура мессенджера

```
┌──────────────────────────────────────────────────────────────┐
│              API Gateway (Kong / NGINX)                       │
│  - Authentication (JWT)                                       │
│  - Rate limiting                                              │
│  - Request routing                                            │
└─────────────┬────────────────────────────────────────────────┘
              │
    ┌─────────┴──────────┬──────────────┬───────────────┐
    │                    │              │               │
    ▼                    ▼              ▼               ▼
┌─────────────┐  ┌─────────────┐ ┌──────────┐  ┌──────────┐
│ Auth        │  │ Messaging   │ │Matchmake │  │ Rating   │
│ Service     │  │ Service     │ │ Service  │  │ Service  │
│ (Port 3001)│  │ (Port 3002) │ │(Port3003)│  │(Port3004)│
│             │  │             │ │          │  │          │
│ Endpoints:  │  │ Endpoints:  │ │Endpoints:│  │Endpoints:│
│ /auth/*     │  │ /messages/* │ │/queue/*  │  │/ratings/*│
│             │  │ /chat/*     │ │/matches/*│  │/reviews/*│
│             │  │ /upload/*   │ │          │  │          │
│             │  │             │ │ Uses:    │  │ Uses:    │
│ Uses:       │  │ Uses:       │ │ -Redis   │  │ -Postgre │
│ -PostgreSQL │  │ -Redis      │ │ -RabbitMQ  │ │-SQL      │
│ -Redis      │  │ -PostgreSQL │ │          │  │          │
│ -Resend     │  │ -S3/MinIO   │ │          │  │          │
│ -JWT lib    │  │ -Elasticsearch          │  │          │
└──────┬──────┘  └──────┬──────┘ └────┬─────┘  └──────────┘
       │                │             │
       │                ▼             │
       │         ┌──────────────┐     │
       │         │WebSocket     │     │
       │         │Handler Srv   │     │
       │         │(Port 3005)   │     │
       │         │              │     │
       │         │Maintains:    │     │
       │         │- Connections │     │
       │         │- Rooms       │     │
       │         │- Sessions    │     │
       │         │              │     │
       │         │Uses:         │     │
       │         │- Redis       │     │
       │         │- Socket.io   │     │
       │         └──────┬───────┘     │
       │                │             │
       │                ▼             │
       │         ┌─────────────────┐  │
       │         │Message Broker   │  │
       │         │(RabbitMQ/Redis) │  │
       │         │                 │  │
       │         │Queues:          │  │
       │         │- chat.messages  │  │
       │         │- notifications  │  │
       │         │- match.events   │  │
       │         │- ratings.created│  │
       │         └────────┬────────┘  │
       │                  │           │
       └──────────────────┼───────────┘
                          │
            ┌─────────────┼─────────────┐
            │             │             │
            ▼             ▼             ▼
        PostgreSQL    Redis Cache    Elasticsearch
        (Main DB)    (Session,       (Message
                      Queues)        Indexing)
```

## Описание каждого микросервиса мессенджера

### 1. Authentication Service (Port 3001)

```
/auth/register              POST  Register new user
/auth/login                 POST  Login user
/auth/logout                POST  Logout (invalidate token)
/auth/refresh               POST  Get new access token
/auth/verify-email          POST  Verify email with code
/auth/forgot-password       POST  Send reset email
/auth/reset-password        POST  Reset password
/auth/profile               GET   Get current user profile
/auth/profile               PUT   Update profile (interests, bio)
/auth/delete-account        DELETE Delete account

Database:
- users (id, email, password_hash, nickname, avatar_url, bio, interests[], 
         verified, created_at, updated_at, deleted_at)
- refresh_tokens (token, user_id, expires_at)
- email_verification (code, user_id, expires_at)
- password_resets (code, user_id, expires_at)

Internal APIs used:
- PostgreSQL
- Redis (cache, token blacklist)
- Resend (email service)
```

### 2. Messaging Service (Port 3002)

```
WebSocket: /chat/ws?user_id=X&token=JWT
  ├─ On connect: Load message history, partner status
  ├─ Messages: {type: "message", content, attachments[]}
  ├─ Typing: {type: "typing"}
  └─ Disconnect: Cleanup, save session

REST endpoints:
/messages/list              GET   Get chat history (paginated)
/messages/search            GET   Search in messages
/messages/upload            POST  Upload image/file
/messages/mark-read         POST  Mark messages as read
/messages/{id}/delete       DELETE Delete message (soft)

Database:
- messages (id, room_id, sender_id, content, content_encrypted,
            attachments[], read_by[], created_at, deleted_at)
- chat_rooms (id, user1_id, user2_id, created_at, ended_at, is_active)
- read_receipts (message_id, user_id, read_at)

Internal APIs used:
- PostgreSQL
- Elasticsearch (full-text search)
- Redis Pub/Sub (real-time delivery)
- Redis Cache (active connections)
- S3/MinIO (media storage)
```

### 3. Matchmaking Service (Port 3003)

```
/queue/join                 POST  Join queue
/queue/leave                POST  Leave queue
/queue/status               GET   Get current queue status
/queue/user-status          GET   Get your position in queue
/matches/list               GET   Your match history
/matches/{id}               GET   Match details
/matches/{id}/end           POST  End match/session

Database:
- queue (user_id, joined_at, interests_filter[], status)
- matches (id, user1_id, user2_id, room_id, created_at, ended_at, 
          status: pending/accepted/rejected/completed)
- match_history (user_id, match_id, joined_at, left_at)

Matchmaking Algorithm:
1. Get top 5 users from queue
2. For each pair calculate similarity:
   - Shared interests (Jaccard similarity)
   - Rating difference (prefer similar ratings)
   - Geography (if available)
   - Time in queue (fairness)
3. Create match when score > threshold
4. Send push notifications to both

Internal APIs used:
- PostgreSQL
- Redis (queue management)
- RabbitMQ (async matching)
- Notification Service (push)
```

### 4. Rating Service (Port 3004)

```
/ratings/create             POST  Submit rating for partner
/ratings/user/{id}          GET   Get user's ratings (public profile)
/ratings/my-ratings         GET   Get your ratings
/ratings/{id}/update        PUT   Update your rating (within 24h)
/ratings/{id}/delete        DELETE Delete rating (within 24h)
/reviews/list               GET   List reviews for user

Database:
- ratings (id, from_user_id, to_user_id, match_id, stars: 1-5,
          comment, created_at, updated_at)
- user_stats (user_id, avg_rating, total_ratings, total_matches, 
             matches_completed, flag_count, banned_at)

Triggers:
- After rating created: Update user_stats.avg_rating
- After 3+ low ratings (<=2 stars): Flag account for review
- After 5+ low ratings: Shadow ban (hidden from queue)

Internal APIs used:
- PostgreSQL
```

### 5. Notification Service (Shared)

```
Endpoints:
/notifications/register-token  POST  Register FCM/APNs token
/notifications/send-test       POST  Send test notification
/notifications/history         GET   Get notification history

Event Listeners (from RabbitMQ):
- match.found        → Send "Match found!" push
- message.received   → Send "New message" push (if in background)
- session.starting   → Send "Chat starts now" push
- rating.reminder    → Send "Rate this match" push (after 1h)

Supported channels:
- FCM (Android)
- APNs (iOS)
- Email (Resend)
- In-app (via WebSocket)

Database:
- push_tokens (user_id, token, platform: ios/android, created_at)
- notification_history (user_id, type, payload, sent_at, read_at)
```

### 6. WebSocket Service (Port 3005)

```
Maintains real-time connections using Socket.io

Rooms:
- /chat/{room_id}          For active chat
- /queue/{user_id}         For matching updates
- /notifications           For user notifications

Events (client → server):
- connect                  WebSocket connected
- message                  Send message
- typing                   User is typing
- read-receipt             Message read
- disconnect               Cleanup

Events (server → client):
- partner-status           "online" / "offline"
- message-received         New message
- typing-indicator         Partner typing
- read-receipt             Message read by partner
- session-ended            Session finished

Internal APIs used:
- Redis (session store for horizontal scaling)
- PostgreSQL (persistence)
- RabbitMQ (event broadcasting)
```

## Развёртывание (Docker Compose / Kubernetes)

```yaml
services:
  # Gateway
  api-gateway:
    image: kong:latest
    ports: [8000:8000]
    depends_on: [postgres, redis]
    
  # Microservices
  auth-service:
    build: ./services/auth
    ports: [3001:3001]
    env: [DATABASE_URL, JWT_SECRET, RESEND_API_KEY]
    depends_on: [postgres, redis]
    
  messaging-service:
    build: ./services/messaging
    ports: [3002:3002]
    env: [DATABASE_URL, REDIS_URL, ELASTICSEARCH_URL, S3_*]
    depends_on: [postgres, redis, elasticsearch, minio]
    
  matchmaking-service:
    build: ./services/matchmaking
    ports: [3003:3003]
    env: [DATABASE_URL, REDIS_URL, RABBITMQ_URL]
    depends_on: [postgres, redis, rabbitmq]
    
  rating-service:
    build: ./services/rating
    ports: [3004:3004]
    env: [DATABASE_URL]
    depends_on: [postgres]
    
  websocket-service:
    build: ./services/websocket
    ports: [3005:3005]
    env: [DATABASE_URL, REDIS_URL, RABBITMQ_URL]
    depends_on: [postgres, redis, rabbitmq]
    
  # Infrastructure
  postgres:
    image: postgres:15
    environment: [POSTGRES_DB, POSTGRES_PASSWORD]
    volumes: [postgres_data:/var/lib/postgresql/data]
    
  redis:
    image: redis:7-alpine
    volumes: [redis_data:/data]
    
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.0.0
    environment: [discovery.type=single-node]
    volumes: [es_data:/usr/share/elasticsearch/data]
    
  rabbitmq:
    image: rabbitmq:3.12-management
    ports: [5672:5672, 15672:15672]
    volumes: [rabbitmq_data:/var/lib/rabbitmq]
    
  minio:
    image: minio/minio
    ports: [9000:9000, 9001:9001]
    environment: [MINIO_ROOT_USER, MINIO_ROOT_PASSWORD]
    volumes: [minio_data:/minio_data]

volumes:
  postgres_data:
  redis_data:
  es_data:
  rabbitmq_data:
  minio_data:
```
