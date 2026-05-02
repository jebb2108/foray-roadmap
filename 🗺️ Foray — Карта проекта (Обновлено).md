---
tags: [MOC, foray, overview, updated]
---

# 🗺️ Foray — Карта проекта (Обновлено)

**Foray** — мобильное приложение для поиска единомышленников через текстовый чат на основе общих интересов.

---

## 📋 Справочная архитектура (chat.lllang.site)

Используется как основа для нового проекта. Документы для изучения существующей системы:

| Компонент | Документ | Цель |
|-----------|----------|------|
| **Backend анализ** | [[Анализ Backend chat.lllang.site]] | Разбор текущей архитектуры, уязвимостей, рекомендации |
| **Общая архитектура** | [[Архитектура chat.lllang.site]] | Высокоуровневая схема web-app-service + Gateway |
| **Gateway сервис** | [[Gateway Сервис]] | Роль центрального backend, API эндпоинты |
| **Матчмейкинг** | [[Матчмейкинг]] | Алгоритм подбора пар |
| **WebSocket чат** | [[WebSocket Чат]] | Real-time обмен сообщениями |
| **Жизненный цикл сессии** | [[Жизненный цикл сессии]] | Flow от поиска до завершения |
| **API эндпоинты** | [[API Эндпоинты]] | Полный список Gateway маршрутов |
| **Подписки и платежи** | [[Подписки и Платежи]] | YooKassa интеграция, модели подписок |

---

## 🆕 Переработанная архитектура Foray

Документы для нового мобильного приложения:

### Высокоуровневая архитектура

| Документ | Описание |
|----------|----------|
| **[[Переработанная Архитектура Foray]]** | Новый концепт, микросервисы, диаграммы |
| **[[Gateway и Микросервисы Мессенджера]]** | Детальная архитектура 6 микросервисов + Gateway |

### Детали реализации

| Компонент | Статус | Документ |
|-----------|--------|----------|
| Аутентификация | ✅ Спланировано | [[Foray — Аутентификация]] |
| Технический стек | ✅ Спланировано | [[Foray — Технический стек]] |
| Роадмап | ✅ Спланировано | [[Foray — Роадмап]] |

---

## 🏗️ Архитектура Foray (Упрощённая)

```
┌──────────────────────────────────────┐
│   Mobile Apps (iOS / Android)        │
│                                      │
│  - Profile Management                │
│  - Matchmaking Queue                 │
│  - Messaging                         │
│  - Rating System                     │
│  - Push Notifications                │
└──────────────┬───────────────────────┘
               │
        ┌──────▼──────┐
        │ API Gateway │
        │  (Kong)     │
        └──────┬──────┘
               │
    ┌──────────┼──────────┬─────────┬──────────┐
    │          │          │         │          │
    ▼          ▼          ▼         ▼          ▼
  Auth      Messaging  Matching   Rating   WebSocket
  (3001)    (3002)     (3003)     (3004)   (3005)
    │          │          │         │          │
    └──────────┼──────────┼─────────┼──────────┘
               │
    ┌──────────▼──────────┐
    │ Shared Services:    │
    │ - PostgreSQL (DB)   │
    │ - Redis (cache)     │
    │ - RabbitMQ (events) │
    │ - Elasticsearch     │
    │ - MinIO (files)     │
    └─────────────────────┘
```

---

## 📚 Полный список документов проекта

### Раздел: Существующая система (Справочник)

- [[Архитектура chat.lllang.site]]
- [[Анализ Backend chat.lllang.site]]
- [[Gateway Сервис]]
- [[Матчмейкинг]]
- [[WebSocket Чат]]
- [[Жизненный цикл сессии]]
- [[API Эндпоинты]]
- [[Подписки и Платежи]]

### Раздел: Новая архитектура Foray

- [[Переработанная Архитектура Foray]]
- [[Gateway и Микросервисы Мессенджера]]
- [[Foray — Обзор]]
- [[Foray — Аутентификация]]
- [[Foray — Технический стек]]
- [[Foray — Роадмап]]

---

## 🔄 Миграция от chat.lllang.site к Foray

### Что переносим из chat.lllang.site ✅

- **WebSocket обработка** — структура событий
- **Шифрование сообщений** — Fernet алгоритм
- **Матчмейкинг алгоритм** — базовая логика (адаптируем под интересы)
- **Email верификация** — flow и шаблоны
- **YooKassa интеграция** — платежный процесс
- **In-memory состояние** → **Redis** (масштабируемость)

### Что меняем 🔄

| Компонент | Было | Станет |
|-----------|------|--------|
| **Платформа** | Веб | Мобильное |
| **Auth система** | Веб-сессия | Email + password + JWT |
| **Notifications** | Polling | Push (FCM/APNs) |
| **Architecture** | Монолит (web-app-service) | Микросервисы (6 сервисов) |
| **State management** | In-memory | Redis |
| **Масштабируемость** | 1 инстанс | N инстансов (load balanced) |

### Что добавляем 🆕

- **Собственная система аккаунтов** (Foray-специфичные)
- **Рейтинговая система** (1-5 звёзд)
- **Система друзей** (контакты, чёрный список)
- **Profile enrichment** (AI рекомендации)
- **Offline режим** (локальный кэш)
- **Микросервисная архитектура** (независимое масштабирование)
- **Message broker** (RabbitMQ для event-driven)
- **Full-text поиск** (Elasticsearch)
- **Object storage** (MinIO/S3 для медиа)

---

## 📊 Сравнение архитектур

### chat.lllang.site (Старое)

```
Web Client
    ↓
web-app-service (монолит)
    ├─ WebSocket handler
    ├─ JWT manager
    ├─ Email service
    └─ Nginx proxy
    ↓
Gateway (основной backend)
    ├─ PostgreSQL
    ├─ Worker (матчмейкинг)
    └─ AI service
    ↓
Внешние: YooKassa, Resend
```

**Проблемы:**
- ❌ Монолитная web-app-service
- ❌ In-memory WebSocket state
- ❌ Нет горизонтального масштабирования
- ❌ Polling каждую секунду
- ❌ Потеря состояния при рестарте

### Foray (Новое)

```
Mobile Client (iOS/Android)
    ↓
API Gateway (Kong)
    ├─ Load balancing
    ├─ Rate limiting
    ├─ JWT validation
    └─ Request routing
    ↓
┌───────────────────────────────────┐
│   Микросервисы (6 сервисов):      │
│                                   │
│  1. Auth Service (3001)           │
│  2. Messaging Service (3002)      │
│  3. Matchmaking Service (3003)    │
│  4. Rating Service (3004)         │
│  5. Notification Service (shared) │
│  6. WebSocket Service (3005)      │
└───────────────────────────────────┘
    ↓
┌───────────────────────────────────┐
│   Shared Infrastructure:           │
│                                   │
│  - PostgreSQL (primary store)     │
│  - Redis (cache, queues)          │
│  - RabbitMQ (event broker)        │
│  - Elasticsearch (search)         │
│  - MinIO (object storage)         │
└───────────────────────────────────┘
    ↓
Внешние: YooKassa, Resend, FCM/APNs
```

**Преимущества:**
- ✅ Микросервисная архитектура
- ✅ Независимое масштабирование каждого сервиса
- ✅ Redis для распределённого состояния
- ✅ Push notifications вместо polling
- ✅ Event-driven (RabbitMQ)
- ✅ Horizontal scaling
- ✅ Лучше готова к отказам

---

## 🎯 Концептуальные отличия Foray

| Аспект | chat.lllang.site | Foray |
|--------|-----------------|-------|
| **Цель** | Обучение английскому | Поиск единомышленников |
| **Платформа** | Веб | Мобильное (нативное) |
| **Матчмейкинг** | По языковому уровню | По интересам + рейтингу |
| **Время сессии** | 15 минут (фиксировано) | Не ограничено |
| **История контактов** | Не сохраняется | Друзья, чёрный список |
| **Рейтинг партнёра** | Нет | 1-5 звёзд (обязателен) |
| **Notifications** | Polling (web) | Push (FCM/APNs) |
| **Offline** | Невозможно | Локальный кэш |
| **Платежи** | YooKassa | YooKassa + Apple Pay + Google Pay |
| **Backend** | Монолит | Микросервисы |
| **Масштабируемость** | 1 инстанс | N инстансов |

---

## 📈 Роадмап разработки

```
Фаза 1: Foundation (API Gateway + Auth)
├─ Настроить Kong Gateway
├─ Реализовать Auth Service
├─ Подготовить БД schema
└─ Deploy на staging

Фаза 2: Messaging Core
├─ Messaging Service (REST + WebSocket)
├─ Redis session store
├─ Elasticsearch индексирование
└─ S3/MinIO интеграция

Фаза 3: Matchmaking
├─ Matchmaking Service
├─ Queue management (Redis)
├─ Algorithm optimization
└─ Push notifications (FCM/APNs)

Фаза 4: Rating & Moderation
├─ Rating Service
├─ Moderation system
├─ Report handling
└─ User safety features

Фаза 5: Mobile Apps
├─ iOS app (Swift)
├─ Android app (Kotlin)
├─ Offline mode
└─ Beta testing

Фаза 6: Production
├─ Performance optimization
├─ Security audit
├─ Load testing
└─ Launch
```

---

## 🔗 Связь документов

```
🗺️ Foray — Карта проекта (вы находитесь здесь)
├── Справочные документы (существующая система)
│   ├── Анализ Backend chat.lllang.site
│   ├── Архитектура chat.lllang.site
│   └── ... (другие)
│
├── Новая архитектура
│   ├── Переработанная Архитектура Foray
│   │   ├── Диаграммы сервисов
│   │   ├── Flows регистрации, матчмейкинга, чата
│   │   └── Docker Compose
│   │
│   └── Gateway и Микросервисы Мессенджера
│       ├── API Gateway (Kong)
│       ├── Auth Service (3001)
│       ├── Messaging Service (3002)
│       ├── Matchmaking Service (3003)
│       ├── Rating Service (3004)
│       └── WebSocket Service (3005)
│
└── Детали реализации
    ├── Foray — Аутентификация
    ├── Foray — Технический стек
    └── Foray — Роадмап
```

---

## 📝 Как использовать эту карту

1. **Для новых разработчиков:**
   - Прочитайте [[Foray — Обзор]]
   - Затем перейдите в [[Переработанная Архитектура Foray]]
   - Изучите микросервисы в [[Gateway и Микросервисы Мессенджера]]

2. **Для разработчиков из chat.lllang.site:**
   - Прочитайте [[Анализ Backend chat.lllang.site]] для контекста
   - Посмотрите раздел "Миграция" в этом документе
   - Изучите новую архитектуру

3. **Для backend разработчиков:**
   - Начните с [[Gateway и Микросервисы Мессенджера]]
   - Это полное описание всех API, БД, flows

4. **Для DevOps/инженеров инфраструктуры:**
   - Раздел "Docker Compose" в [[Переработанная Архитектура Foray]]
   - Docker Compose раздел в [[Gateway и Микросервисы Мессенджера]]

---

## ✅ Статус проекта

| Компонент | Статус | Примечание |
|-----------|--------|-----------|
| Архитектура | ✅ Завершено | Все диаграммы и документы готовы |
| API Gateway | ⏳ To Do | Kong configuration |
| Auth Service | ⏳ To Do | JWT + password management |
| Messaging Service | ⏳ To Do | WebSocket + REST API |
| Matchmaking Service | ⏳ To Do | Algorithm + queue management |
| Rating Service | ⏳ To Do | User reviews system |
| WebSocket Service | ⏳ To Do | Real-time events |
| Mobile Apps | ⏳ To Do | iOS + Android |
| DevOps | ⏳ To Do | Docker, Kubernetes setup |
| Security Review | ⏳ To Do | Authentication, encryption |

---

## 📞 Следующие шаги

1. ✅ Пересмотреть диаграммы с командой
2. ⏳ Обсудить выбор технологий ([[Foray — Технический стек]])
3. ⏳ Утвердить роадмап ([[Foray — Роадмап]])
4. ⏳ Начать реализацию Фазы 1
