---
tags: [gateway, microservices, architecture, messaging]
---

# Gateway и Микросервисы Мессенджера

← [[🗺️ Foray — Карта проекта]]

## Архитектура в разрезе

```
┌────────────────────────────────────────────────────────────┐
│                 Mobile Apps (iOS/Android)                   │
│                                                              │
│  REST API + WebSocket                                      │
└──────────────────────────┬─────────────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ▼                 ▼                 ▼
      HTTP/1.1         HTTP/2            WebSocket
      (REST)           (gRPC)            (Realtime)
         │                 │                 │
┌────────▼─────────────────▼─────────────────▼──────────────┐
│                                                             │
│         API GATEWAY (Kong / NGINX Plus)                    │
│                                                             │
│  ├─ TLS termination                                        │
│  ├─ Request validation (OpenAPI schema)                    │
│  ├─ JWT authentication (bearer token)                      │
│  ├─ Rate limiting (per user, per endpoint)                 │
│  ├─ Request routing (path-based)                           │
│  ├─ Response compression (gzip)                            │
│  ├─ Logging (ELK stack)                                    │
│  ├─ Tracing (Jaeger/Zipkin)                               │
│  └─ Metrics (Prometheus)                                   │
│                                                             │
│  Routing table:                                            │
│  /auth/*        → Auth Service (3001)                     │
│  /messages/*    → Messaging Service (3002)                │
│  /queue/*       → Matchmaking Service (3003)              │
│  /ratings/*     → Rating Service (3004)                   │
│  /notifications/*→ Notification Service (shared)          │
│  /ws/*          → WebSocket Service (3005)                │
│                                                             │
└────────┬────────┬────────┬────────┬────────┬──────────────┘
         │        │        │        │        │
         ▼        ▼        ▼        ▼        ▼
      (3001)  (3002)  (3003)  (3004)  (3005)
```

## 1. API Gateway (Входная точка)

### Функции Gateway

| Функция | Описание | Реализация |
|---------|---------|-----------|
| **Load Balancing** | Распределение нагрузки между instances | Round-robin, Least connections |
| **TLS/SSL** | Шифрование в пути | ACME (Let's Encrypt) |
| **Authentication** | Проверка JWT токенов | validate-jwt plugin |
| **Rate Limiting** | Защита от DDoS | Redis-based rate limiter |
| **Request Validation** | Проверка schema (OpenAPI) | OpenAPI validator plugin |
| **Request Logging** | Логирование всех запросов | Syslog plugin → ELK |
| **Tracing** | Распределённое трейсирование | Jaeger plugin |
| **Metrics** | Сбор метрик Prometheus | Prometheus plugin |
| **CORS** | Управление cross-origin | CORS plugin |
| **Response Transformation** | Трансформация ответов | Response transformer plugin |

### Kong Configuration Example

```yaml
# kong.yml

# Upstream = группа backend сервисов
upstreams:
  auth_upstream:
    targets:
      - { target: "auth-service:3001", weight: 10 }
      - { target: "auth-service-2:3001", weight: 10 }
  
  messaging_upstream:
    targets:
      - { target: "messaging-service:3002", weight: 10 }
  
  matchmaking_upstream:
    targets:
      - { target: "matchmaking-service:3003", weight: 10 }

# Services = логические сервисы
services:
  auth_service:
    url: "http://auth_upstream"
    routes:
      - { paths: ["/auth"], methods: [POST, GET] }
    plugins:
      - name: jwt
        config:
          secret: ${JWT_SECRET}
          header_names: [Authorization]
      - name: rate-limiting
        config:
          minute: 100  # 100 requests per minute
          policy: redis
      - name: request-transformer
        config:
          add:
            headers: ["X-Service: auth"]
      - name: cors
        config:
          origins: ["*"]
          credentials: true
  
  messaging_service:
    url: "http://messaging_upstream"
    routes:
      - { paths: ["/messages"], methods: [GET, POST, DELETE] }
    plugins:
      - name: jwt
      - name: rate-limiting
        config:
          minute: 1000  # More generous for messaging
```

### Rate Limiting Strategy

```python
# Токен-бакет алгоритм

Per user:
- Auth endpoints: 10 requests/min
- Messaging: 1000 requests/min
- Matchmaking: 100 requests/min
- Ratings: 50 requests/min

Per IP (DDoS protection):
- Total: 10000 requests/min
- Burst: 50 requests/sec

Per endpoint (capacity protection):
- /messages/upload: 10 MB/min
- /queue/join: 100 users/sec
```

---

## 2. Auth Service (Port 3001)

### Задачи

- Регистрация пользователей
- Логин / логаут
- Генерация и валидация JWT токенов
- Email верификация
- Управление профилями
- Password reset

### API Endpoints

```python
# Registration
POST /auth/register
{
    "email": "user@example.com",
    "password": "SecurePass123",
    "nickname": "John_Doe",
    "interests": ["gaming", "music", "tech"],
    "avatar": "base64_or_url"
}
Response:
{
    "status": "verify_email",
    "message": "Verification code sent to email"
}

# Email Verification
POST /auth/verify-email
{
    "code": "123456"
}
Response:
{
    "access_token": "eyJhbGc...",
    "refresh_token": "eyJhbGc...",
    "expires_in": 900  // 15 minutes
}

# Login
POST /auth/login
{
    "email": "user@example.com",
    "password": "SecurePass123"
}
Response:
{
    "access_token": "eyJhbGc...",
    "refresh_token": "eyJhbGc...",
    "expires_in": 900,
    "user": {
        "id": "user_123",
        "email": "user@example.com",
        "nickname": "John_Doe",
        "avatar_url": "https://..."
    }
}

# Token Refresh
POST /auth/refresh
{
    "refresh_token": "eyJhbGc..."
}
Response:
{
    "access_token": "eyJhbGc...",
    "expires_in": 900
}

# Get Current User
GET /auth/me
Headers: { Authorization: "Bearer eyJhbGc..." }
Response:
{
    "id": "user_123",
    "email": "user@example.com",
    "nickname": "John_Doe",
    "avatar_url": "https://...",
    "bio": "I love gaming",
    "interests": ["gaming", "music"],
    "created_at": "2024-01-15T10:00:00Z",
    "verified": true
}

# Update Profile
PUT /auth/profile
{
    "nickname": "NewName",
    "bio": "New bio",
    "avatar": "base64",
    "interests": ["gaming", "tech"]
}
Response: { status: "ok" }

# Logout
POST /auth/logout
Response: { status: "ok" }
```

### Token Strategy

```
Access Token (JWT):
├─ Type: RS256 (asymmetric, signed with private key)
├─ Payload:
│  ├─ sub: user_id
│  ├─ email: user@example.com
│  ├─ nickname: John_Doe
│  ├─ iat: 1234567890
│  ├─ exp: 1234568790 (15 minutes)
│  └─ jti: unique_token_id
├─ Verify: Public key (no secret needed)
├─ Revocation: Via Redis blacklist
└─ Used for: All API calls

Refresh Token:
├─ Type: Opaque string (generated with secrets.token_urlsafe)
├─ Stored in: Redis (key: refresh_token:{token})
├─ Payload:
│  ├─ user_id: user_123
│  ├─ created_at: timestamp
│  ├─ expires_at: timestamp + 30 days
│  └─ rotation_count: 0
├─ Rotation: New refresh token every 7 days
├─ Revocation: DELETE from Redis
└─ Used for: Getting new access tokens
```

### Password Security

```python
# Хеширование (bcrypt)
password_hash = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))

# Верификация
bcrypt.checkpw(provided_password.encode(), stored_hash)

# Password reset flow
1. User requests reset: POST /auth/forgot-password
2. System generates reset code (6 alphanumeric, 15 min TTL)
3. Send code via email (Resend)
4. User enters code: POST /auth/reset-password
5. System validates code, hashes new password
6. Invalidate all refresh tokens for this user
7. Force re-login on all devices
```

### Database Schema

```sql
-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    nickname VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    avatar_url VARCHAR(500),
    bio TEXT,
    interests TEXT[] DEFAULT '{}',  -- Array of interest tags
    verified BOOLEAN DEFAULT false,
    verification_code VARCHAR(6),
    verification_code_expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT now(),
    updated_at TIMESTAMP DEFAULT now(),
    deleted_at TIMESTAMP,
    
    INDEX email_idx (email),
    INDEX nickname_idx (nickname),
    INDEX verified_idx (verified)
);

-- Refresh tokens
CREATE TABLE refresh_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    token VARCHAR(500) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT now(),
    expires_at TIMESTAMP NOT NULL,
    rotation_count INT DEFAULT 0,
    last_used_at TIMESTAMP,
    ip_address INET,
    user_agent VARCHAR(500),
    revoked BOOLEAN DEFAULT false,
    
    INDEX user_id_idx (user_id),
    INDEX token_idx (token)
);

-- Email verification codes
CREATE TABLE email_verifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    code VARCHAR(6) NOT NULL,
    created_at TIMESTAMP DEFAULT now(),
    expires_at TIMESTAMP NOT NULL,
    verified_at TIMESTAMP,
    
    INDEX user_id_idx (user_id)
);

-- Password resets
CREATE TABLE password_resets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    token VARCHAR(500) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT now(),
    expires_at TIMESTAMP NOT NULL,
    used_at TIMESTAMP,
    
    INDEX user_id_idx (user_id)
);
```

---

## 3. Messaging Service (Port 3002)

### Задачи

- Отправка и получение сообщений
- История чатов
- Шифрование сообщений
- Управление вложениями (images, files)
- Поиск в сообщениях
- Индикаторы прочитанности

### API Endpoints

```python
# Get message history
GET /messages?room_id=room_123&limit=50&offset=0
Response:
{
    "total": 450,
    "messages": [
        {
            "id": "msg_123",
            "room_id": "room_123",
            "sender_id": "user_123",
            "sender_nickname": "John_Doe",
            "content": "Hi there!",
            "created_at": "2024-01-15T10:00:00Z",
            "read_by": ["user_456"],
            "attachments": []
        },
        ...
    ]
}

# Search messages
GET /messages/search?q=hello&room_id=room_123
Response:
{
    "results": [
        {
            "id": "msg_123",
            "content": "Hi there, hello world!",
            "created_at": "2024-01-15T10:00:00Z",
            "snippet": "Hi there, <b>hello</b> world!"
        }
    ]
}

# Upload attachment
POST /messages/upload
{
    "file": <binary>,
    "room_id": "room_123",
    "type": "image"  // or "file"
}
Response:
{
    "id": "attach_123",
    "url": "https://s3.example.com/messages/attach_123.jpg",
    "size": 102400,
    "type": "image",
    "created_at": "2024-01-15T10:00:00Z"
}

# Mark messages as read
POST /messages/mark-read
{
    "message_ids": ["msg_123", "msg_124"],
    "room_id": "room_123"
}
Response: { status: "ok" }

# Delete message (soft)
DELETE /messages/{message_id}
Response: { status: "ok" }
```

### WebSocket Events

```javascript
// Connection
ws = new WebSocket(
    'wss://api.foray.app/chat/ws?room_id=room_123&token=JWT'
);

// Client → Server events

ws.send(JSON.stringify({
    type: 'message',
    content: 'Hello!',
    attachments: [],
    timestamp: Date.now()
}));

ws.send(JSON.stringify({
    type: 'typing',
    timestamp: Date.now()
}));

ws.send(JSON.stringify({
    type: 'stop-typing',
    timestamp: Date.now()
}));

ws.send(JSON.stringify({
    type: 'read-receipt',
    message_ids: ['msg_123'],
    timestamp: Date.now()
}));

// Server → Client events

ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);
    
    if (msg.type === 'message') {
        // {id, sender_id, sender_nickname, content, created_at, attachments}
    } else if (msg.type === 'typing') {
        // {sender_id, sender_nickname}
    } else if (msg.type === 'read-receipt') {
        // {message_ids, read_by, read_at}
    } else if (msg.type === 'partner-status') {
        // {status: 'online' | 'offline', timestamp}
    } else if (msg.type === 'session-ended') {
        // {reason: 'user-left' | 'timeout' | 'other', timestamp}
    }
};
```

### Message Encryption (Fernet)

```python
from cryptography.fernet import Fernet

# Generate key once and store in env
KEY = os.getenv('FERNET_KEY')
cipher_suite = Fernet(KEY.encode())

# Encrypt message before storing
plaintext = "Hello, world!"
ciphertext = cipher_suite.encrypt(plaintext.encode())
# ciphertext is bytes, store as base64

# Decrypt when retrieving
retrieved_ciphertext = base64.b64decode(stored_value)
plaintext = cipher_suite.decrypt(retrieved_ciphertext).decode()
```

### Database Schema

```sql
-- Chat rooms
CREATE TABLE chat_rooms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user1_id UUID NOT NULL REFERENCES users(id),
    user2_id UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMP DEFAULT now(),
    ended_at TIMESTAMP,
    is_active BOOLEAN DEFAULT true,
    
    UNIQUE(user1_id, user2_id),
    INDEX active_idx (is_active),
    INDEX user1_idx (user1_id),
    INDEX user2_idx (user2_id)
);

-- Messages
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    room_id UUID NOT NULL REFERENCES chat_rooms(id),
    sender_id UUID NOT NULL REFERENCES users(id),
    content_encrypted TEXT NOT NULL,  -- Fernet encrypted
    content_length INT,  -- For statistics
    created_at TIMESTAMP DEFAULT now(),
    updated_at TIMESTAMP DEFAULT now(),
    deleted_at TIMESTAMP,
    
    INDEX room_idx (room_id, created_at),
    INDEX sender_idx (sender_id),
    INDEX time_idx (created_at)
);

-- Message attachments
CREATE TABLE message_attachments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id UUID NOT NULL REFERENCES messages(id),
    file_url VARCHAR(500) NOT NULL,
    file_size INT NOT NULL,
    file_type VARCHAR(50),  -- 'image', 'video', 'file'
    created_at TIMESTAMP DEFAULT now(),
    
    INDEX message_idx (message_id)
);

-- Read receipts
CREATE TABLE read_receipts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id UUID NOT NULL REFERENCES messages(id),
    user_id UUID NOT NULL REFERENCES users(id),
    read_at TIMESTAMP NOT NULL,
    
    UNIQUE(message_id, user_id),
    INDEX message_idx (message_id),
    INDEX user_idx (user_id)
);

-- Message search index (Elasticsearch)
-- Synced from PostgreSQL via Logstash
{
    "id": "msg_123",
    "room_id": "room_123",
    "sender_id": "user_123",
    "content": "Hello world!",
    "created_at": "2024-01-15T10:00:00Z"
}
```

### S3/MinIO Configuration

```python
# Upload image to S3
from boto3 import client

s3_client = client(
    's3',
    endpoint_url='https://s3.amazonaws.com',
    aws_access_key_id=AWS_ACCESS_KEY,
    aws_secret_access_key=AWS_SECRET_KEY
)

def upload_message_attachment(file, room_id):
    filename = f"messages/{room_id}/{uuid4()}.jpg"
    
    s3_client.upload_fileobj(
        file,
        bucket_name='foray-media',
        key=filename,
        ExtraArgs={
            'ContentType': 'image/jpeg',
            'ACL': 'private',
            'Metadata': {
                'room-id': room_id,
                'uploaded-by': current_user.id
            }
        }
    )
    
    # Generate signed URL (valid for 1 hour)
    url = s3_client.generate_presigned_url(
        'get_object',
        Params={'Bucket': 'foray-media', 'Key': filename},
        ExpiresIn=3600
    )
    
    return { 'url': url, 'key': filename }
```

---

## 4. Matchmaking Service (Port 3003)

### Задачи

- Управление очередью
- Алгоритм подбора пар
- Создание матчей
- Управление сессиями

### API Endpoints

```python
# Join queue
POST /queue/join
{
    "interests_filter": ["gaming", "music"],  // Optional
    "location": "Russia"  // Optional
}
Response:
{
    "status": "in_queue",
    "position": 5,
    "estimated_wait": 120,  // seconds
    "queue_size": 12
}

# Check queue status
GET /queue/status
Response:
{
    "position": 5,
    "queue_size": 12,
    "queue_growth_rate": 0.8,  // people/sec
    "estimated_wait": 120,
    "is_matching": true
}

# Leave queue
POST /queue/leave
Response: { status: "ok" }

# Check if match found (polling fallback)
GET /queue/check-match
Response:
{
    "found": true,
    "match": {
        "id": "match_123",
        "partner": {
            "id": "user_456",
            "nickname": "Jane_Doe",
            "avatar_url": "https://...",
            "age": 25,
            "interests": ["gaming", "music"],
            "rating": 4.8
        },
        "room_id": "room_789",
        "expires_at": "2024-01-15T10:05:00Z"
    }
}

# Accept/Decline match
POST /matches/{match_id}/accept
Response: { status: "ok" }

POST /matches/{match_id}/decline
Response: { status: "ok" }

# End session
POST /matches/{match_id}/end
Response: { status: "ok" }

# Get match history
GET /matches?limit=20&offset=0
Response:
{
    "total": 50,
    "matches": [
        {
            "id": "match_123",
            "partner_id": "user_456",
            "partner_nickname": "Jane_Doe",
            "created_at": "2024-01-15T10:00:00Z",
            "ended_at": "2024-01-15T10:15:00Z",
            "duration": 900,
            "status": "completed",
            "your_rating": 5,
            "partner_rating": 4
        }
    ]
}
```

### Matchmaking Algorithm

```python
def find_best_match(user_id: str) -> Optional[Match]:
    """
    Algorithm:
    1. Get waiting queue
    2. Filter by subscription status
    3. Calculate compatibility score
    4. Match if score > threshold
    """
    
    # Step 1: Get users waiting for match
    queue = redis.zrange('matchmaking:queue', 0, -1)  # Sorted by join_time
    
    # Step 2: Get user profile
    user = db.users.find_by_id(user_id)
    
    # Step 3: For each candidate, calculate score
    best_match = None
    best_score = 0
    
    for candidate_id in queue:
        if candidate_id == user_id:
            continue
        
        candidate = db.users.find_by_id(candidate_id)
        
        # A. Interest similarity (Jaccard)
        shared_interests = len(set(user.interests) & set(candidate.interests))
        total_interests = len(set(user.interests) | set(candidate.interests))
        interest_score = shared_interests / total_interests if total_interests > 0 else 0
        
        # B. Rating similarity
        rating_diff = abs(user.avg_rating - candidate.avg_rating)
        rating_score = max(0, 1 - (rating_diff / 5))  # Normalized to 0-1
        
        # C. Time-in-queue (fairness)
        user_wait = now() - redis.hget(f'queue:{user_id}', 'joined_at')
        candidate_wait = now() - redis.hget(f'queue:{candidate_id}', 'joined_at')
        wait_diff = abs(user_wait - candidate_wait)
        wait_score = max(0, 1 - (wait_diff / 600))  # Fair if waited similar time
        
        # D. Location (if available)
        location_score = 1.0 if user.location == candidate.location else 0.7
        
        # E. Response time (users who reply fast are better matches)
        response_time = get_avg_message_response_time(candidate_id)
        response_score = max(0.5, 1 - (response_time / 120))  # 2 min threshold
        
        # Composite score (weighted average)
        score = (
            interest_score * 0.40 +
            rating_score * 0.25 +
            wait_score * 0.15 +
            location_score * 0.10 +
            response_score * 0.10
        )
        
        if score > best_score:
            best_score = score
            best_match = candidate_id
    
    # Step 4: Match if score > threshold (0.6)
    if best_score > 0.6 and best_match:
        return create_match(user_id, best_match)
    
    return None

def create_match(user1_id: str, user2_id: str) -> Match:
    """Create match and notify both users"""
    
    # Create room and match
    room_id = str(uuid4())
    match = db.matches.insert({
        'id': str(uuid4()),
        'user1_id': user1_id,
        'user2_id': user2_id,
        'room_id': room_id,
        'status': 'pending',
        'created_at': datetime.now(),
        'expires_at': datetime.now() + timedelta(minutes=1)  # 1 min to accept
    })
    
    # Remove both from queue
    redis.zrem('matchmaking:queue', user1_id)
    redis.zrem('matchmaking:queue', user2_id)
    
    # Publish match event
    redis.publish('matches:found', json.dumps({
        'match_id': match['id'],
        'user1_id': user1_id,
        'user2_id': user2_id,
        'room_id': room_id
    }))
    
    # Send push notifications
    notify_service.send_push(user1_id, 'Match found! Accept within 1 minute.')
    notify_service.send_push(user2_id, 'Match found! Accept within 1 minute.')
    
    return match
```

### Database Schema

```sql
-- Queue (managed in Redis)
-- Redis Sorted Set: matchmaking:queue
-- Members: user_id
-- Score: join_time (unix timestamp)

-- Matches
CREATE TABLE matches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user1_id UUID NOT NULL REFERENCES users(id),
    user2_id UUID NOT NULL REFERENCES users(id),
    room_id UUID NOT NULL UNIQUE REFERENCES chat_rooms(id),
    status VARCHAR(20) DEFAULT 'pending',  -- pending, accepted, rejected, completed
    created_at TIMESTAMP DEFAULT now(),
    user1_accepted_at TIMESTAMP,
    user2_accepted_at TIMESTAMP,
    session_started_at TIMESTAMP,
    ended_at TIMESTAMP,
    end_reason VARCHAR(50),  -- user_left, timeout, both_agreed
    duration INT,  -- seconds
    
    INDEX user1_idx (user1_id),
    INDEX user2_idx (user2_id),
    INDEX status_idx (status),
    INDEX created_idx (created_at)
);

-- Match statistics
CREATE TABLE match_statistics (
    user_id UUID NOT NULL REFERENCES users(id),
    total_matches INT DEFAULT 0,
    completed_matches INT DEFAULT 0,
    avg_session_duration INT DEFAULT 0,
    abandoned_matches INT DEFAULT 0,
    
    PRIMARY KEY (user_id)
);
```

---

## 5. Rating Service (Port 3004)

### Задачи

- Приём оценок от пользователей
- Расчёт рейтинга
- Управление чёрным списком
- Модерация

### API Endpoints

```python
# Submit rating
POST /ratings/create
{
    "match_id": "match_123",
    "stars": 5,
    "comment": "Great conversation!",
    "tags": ["friendly", "interesting"]
}
Response:
{
    "id": "rating_123",
    "created_at": "2024-01-15T10:15:00Z"
}

# Get user's ratings (public)
GET /users/{user_id}/ratings
Response:
{
    "user": {
        "id": "user_456",
        "nickname": "Jane_Doe",
        "avatar_url": "https://..."
    },
    "stats": {
        "avg_rating": 4.8,
        "total_ratings": 25,
        "distribution": {
            "5": 20,
            "4": 4,
            "3": 1,
            "2": 0,
            "1": 0
        }
    },
    "reviews": [
        {
            "id": "rating_123",
            "from_user_nickname": "John_Doe",
            "stars": 5,
            "comment": "Great conversation!",
            "tags": ["friendly", "interesting"],
            "created_at": "2024-01-15T10:15:00Z"
        }
    ]
}

# Update rating (within 24h)
PUT /ratings/{rating_id}
{
    "stars": 4,
    "comment": "Actually, was good."
}
Response: { status: "ok" }

# Delete rating (within 24h)
DELETE /ratings/{rating_id}
Response: { status: "ok" }

# Report user
POST /reports/create
{
    "match_id": "match_123",
    "reason": "inappropriate_language",  // or "harassment", "spam", "other"
    "description": "User was rude"
}
Response: { status: "ok" }

# Get my ratings
GET /ratings/my-ratings
Response:
{
    "given": [...],
    "received": [...]
}
```

### Rating & Moderation System

```python
def update_user_rating(user_id: str):
    """Update user's average rating and apply moderation rules"""
    
    ratings = db.ratings.find({'to_user_id': user_id})
    
    if not ratings:
        return
    
    # Calculate average
    avg_rating = sum(r.stars for r in ratings) / len(ratings)
    
    low_rating_count = sum(1 for r in ratings if r.stars <= 2)
    
    # Update user stats
    db.users.update(user_id, {
        'avg_rating': round(avg_rating, 2),
        'total_ratings': len(ratings)
    })
    
    # Moderation rules
    if low_rating_count >= 3 and len(ratings) >= 5:
        # Flag account for review
        db.flags.insert({
            'user_id': user_id,
            'reason': 'low_ratings',
            'low_rating_count': low_rating_count,
            'created_at': datetime.now()
        })
    
    if low_rating_count >= 5 and len(ratings) >= 10:
        # Shadow ban (hidden from queue, can't find matches)
        db.users.update(user_id, {
            'shadow_banned': True,
            'banned_at': datetime.now()
        })
        
        # Notify admins
        notify_admins(f"User {user_id} shadow banned (low ratings)")
```

### Database Schema

```sql
-- Ratings
CREATE TABLE ratings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    from_user_id UUID NOT NULL REFERENCES users(id),
    to_user_id UUID NOT NULL REFERENCES users(id),
    match_id UUID NOT NULL REFERENCES matches(id),
    stars INT NOT NULL CHECK (stars >= 1 AND stars <= 5),
    comment TEXT,
    tags TEXT[] DEFAULT '{}',
    created_at TIMESTAMP DEFAULT now(),
    updated_at TIMESTAMP DEFAULT now(),
    
    UNIQUE(match_id, from_user_id),  -- One rating per match per user
    INDEX user_idx (to_user_id),
    INDEX created_idx (created_at)
);

-- User reports
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    from_user_id UUID NOT NULL REFERENCES users(id),
    reported_user_id UUID NOT NULL REFERENCES users(id),
    match_id UUID REFERENCES matches(id),
    reason VARCHAR(50) NOT NULL,  -- inappropriate_language, harassment, spam, other
    description TEXT,
    status VARCHAR(20) DEFAULT 'open',  -- open, investigating, resolved, dismissed
    created_at TIMESTAMP DEFAULT now(),
    resolved_at TIMESTAMP,
    
    INDEX reported_user_idx (reported_user_id),
    INDEX status_idx (status)
);

-- User flags and moderation
CREATE TABLE user_flags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    reason VARCHAR(50),  -- low_ratings, report_count, spam, other
    low_rating_count INT,
    report_count INT,
    created_at TIMESTAMP DEFAULT now(),
    reviewed_at TIMESTAMP,
    resolution VARCHAR(50),  -- none, warning, shadow_ban, ban
    
    INDEX user_idx (user_id)
);
```

---

## 6. WebSocket Service (Port 3005)

### Задачи

- Управление WebSocket соединениями
- Real-time события
- Состояние сессии
- Распределённость (Redis session store)

### Implementation (Socket.io)

```python
# server.py
from fastapi import FastAPI
from fastapi_socketio import SocketManager
import socketio

app = FastAPI()

# Redis adapter для горизонтального масштабирования
mgr = socketio.AsyncRedisManager('redis://redis:6379')
sio = socketio.AsyncServer(
    async_mode='asgi',
    client_manager=mgr,
    cors_allowed_origins=['*']
)

@sio.on('connect')
async def connect(sid, environ):
    # Extract JWT from query params
    token = environ['QUERY_STRING'].split('token=')[1]
    user_id = validate_jwt(token)
    
    # Store user connection
    await sio.save_session(sid, {'user_id': user_id})
    
    # Join user's notification room
    sio.enter_room(sid, f'user:{user_id}')
    
    # Join chat room if provided
    room_id = environ['QUERY_STRING'].split('room_id=')[1]
    if room_id:
        sio.enter_room(sid, f'room:{room_id}')
        # Notify others: user joined
        await sio.emit('user-joined', {'user_id': user_id}, room=f'room:{room_id}')

@sio.on('message')
async def on_message(sid, data):
    session = await sio.get_session(sid)
    user_id = session['user_id']
    room_id = data['room_id']
    content = data['content']
    
    # Save message
    msg = db.messages.insert({
        'room_id': room_id,
        'sender_id': user_id,
        'content_encrypted': encrypt(content),
        'created_at': datetime.now()
    })
    
    # Broadcast to room
    await sio.emit('message', {
        'id': msg['id'],
        'sender_id': user_id,
        'content': content,
        'created_at': msg['created_at']
    }, room=f'room:{room_id}')
    
    # Publish to message broker (for persistence, search, etc)
    redis.publish('messages:new', json.dumps(msg))

@sio.on('typing')
async def on_typing(sid, data):
    session = await sio.get_session(sid)
    user_id = session['user_id']
    room_id = data['room_id']
    
    # Notify others
    await sio.emit('typing', {'user_id': user_id}, room=f'room:{room_id}', skip_sid=sid)

@sio.on('disconnect')
async def disconnect(sid):
    session = await sio.get_session(sid)
    if session:
        user_id = session['user_id']
        # Notify rooms: user left
        sio.rooms(sid).forEach(room => {
            sio.emit('user-left', {'user_id': user_id}, room=room)
        })
```

---

## Взаимодействие сервисов

### Пример: User joins queue

```mermaid
sequenceDiagram
    participant Client
    participant Gateway
    participant Matchmaking Service
    participant Notification Service
    participant WebSocket Service
    participant Redis
    participant PostgreSQL

    Client->>Gateway: POST /queue/join
    Gateway->>Matchmaking Service: Forward request
    
    Matchmaking Service->>PostgreSQL: Check subscription
    Matchmaking Service->>Redis: Add to queue (sorted set)
    
    Matchmaking Service->>Redis: Publish match.join event
    Matchmaking Service->>Client: Response: in_queue, position=5
    
    Note over Matchmaking Service: Background worker runs every 5 sec
    Matchmaking Service->>Redis: Get queue
    Matchmaking Service->>PostgreSQL: Load user profiles
    Matchmaking Service->>Matchmaking Service: Calculate scores
    
    alt Match found
        Matchmaking Service->>PostgreSQL: Create match record
        Matchmaking Service->>Redis: Remove both from queue
        Matchmaking Service->>Redis: Publish match.found event
        
        Notification Service->>Notification Service: Listen match.found
        Notification Service->>Redis: Get push tokens
        Notification Service->>FCM/APNs: Send push notification
        
        WebSocket Service->>WebSocket Service: Listen match.found
        WebSocket Service->>Client: WebSocket event: match-found
    end
```

---

## Развёртывание

### Docker Compose

```yaml
version: '3.8'

services:
  # Gateway (Kong)
  kong:
    image: kong:latest
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: postgres
      KONG_PG_PASSWORD: example
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
    ports:
      - "8000:8000"   # Proxy
      - "8001:8001"   # Admin
    depends_on:
      - postgres
    networks:
      - foray

  # Microservices
  auth-service:
    build: ./services/auth
    environment:
      DATABASE_URL: postgresql://user:pass@postgres/foray
      JWT_SECRET_KEY: your-secret-key
      JWT_ALGORITHM: RS256
      RESEND_API_KEY: ${RESEND_API_KEY}
    ports:
      - "3001:3001"
    depends_on:
      - postgres
      - redis
    networks:
      - foray

  messaging-service:
    build: ./services/messaging
    environment:
      DATABASE_URL: postgresql://user:pass@postgres/foray
      REDIS_URL: redis://redis:6379
      ELASTICSEARCH_HOST: elasticsearch
      S3_BUCKET: foray-media
      S3_REGION: us-east-1
      FERNET_KEY: ${FERNET_KEY}
    ports:
      - "3002:3002"
    depends_on:
      - postgres
      - redis
      - elasticsearch
      - minio
    networks:
      - foray

  matchmaking-service:
    build: ./services/matchmaking
    environment:
      DATABASE_URL: postgresql://user:pass@postgres/foray
      REDIS_URL: redis://redis:6379
      RABBITMQ_URL: amqp://user:pass@rabbitmq:5672
    ports:
      - "3003:3003"
    depends_on:
      - postgres
      - redis
      - rabbitmq
    networks:
      - foray

  rating-service:
    build: ./services/rating
    environment:
      DATABASE_URL: postgresql://user:pass@postgres/foray
    ports:
      - "3004:3004"
    depends_on:
      - postgres
    networks:
      - foray

  websocket-service:
    build: ./services/websocket
    environment:
      DATABASE_URL: postgresql://user:pass@postgres/foray
      REDIS_URL: redis://redis:6379
      RABBITMQ_URL: amqp://user:pass@rabbitmq:5672
    ports:
      - "3005:3005"
    depends_on:
      - postgres
      - redis
      - rabbitmq
    networks:
      - foray

  # Infrastructure
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: foray
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - foray

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    networks:
      - foray

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.0.0
    environment:
      discovery.type: single-node
      xpack.security.enabled: "false"
    volumes:
      - es_data:/usr/share/elasticsearch/data
    networks:
      - foray

  rabbitmq:
    image: rabbitmq:3.12-management
    environment:
      RABBITMQ_DEFAULT_USER: user
      RABBITMQ_DEFAULT_PASS: pass
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    networks:
      - foray

  minio:
    image: minio/minio
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - minio_data:/minio_data
    ports:
      - "9000:9000"
      - "9001:9001"
    networks:
      - foray

volumes:
  postgres_data:
  redis_data:
  es_data:
  rabbitmq_data:
  minio_data:

networks:
  foray:
    driver: bridge
```

---

## Summary

| Компонент | Port | Функция | Масштабируемость |
|-----------|------|---------|-----------------|
| API Gateway | 8000 | Входная точка | ✅ Load balanced |
| Auth Service | 3001 | Аутентификация | ✅ Stateless |
| Messaging Service | 3002 | Чаты | ✅ Redis-backed |
| Matchmaking Service | 3003 | Подбор пар | ✅ Queue-based |
| Rating Service | 3004 | Рейтинги | ✅ Stateless |
| WebSocket Service | 3005 | Real-time | ✅ Redis adapter |
| PostgreSQL | 5432 | Primary store | ✅ Replicas + Read replicas |
| Redis | 6379 | Cache, Queues | ✅ Sentinel/Cluster |
| Elasticsearch | 9200 | Full-text search | ✅ Sharding |
| RabbitMQ | 5672 | Message broker | ✅ Clustering |
| MinIO | 9000 | Object storage | ✅ Distributed |
