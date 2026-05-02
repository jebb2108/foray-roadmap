---
tags: [foray, tech-stack, mobile, new-app]
---

# Foray — Технический стек

← [[🗺️ Foray — Карта проекта]]

## Выбор платформы

### React Native (рекомендуется)

**Почему:**
- Один кодобаза для iOS и Android
- WebSocket-клиенты работают нативно
- Большая экосистема для чат-приложений

**Ключевые библиотеки:**

| Задача | Библиотека |
|--------|-----------|
| Навигация | `react-navigation` |
| WebSocket | встроенный `WebSocket` API |
| Push-уведомления | `react-native-firebase` (FCM + APNs через FCM) |
| Хранение токенов | `react-native-keychain` |
| Платежи | `react-native-yookassa` / `@stripe/stripe-react-native` |
| HTTP-клиент | `axios` или `fetch` |
| Состояние | `zustand` или `redux-toolkit` |

### Альтернатива: Flutter

**Когда выбрать:**
- Нужна максимальная производительность UI
- Команда готова писать на Dart

---

## Backend (изменения минимальны)

Существующий [[Gateway Сервис|Gateway]] (FastAPI) остаётся. Нужно добавить:

| Изменение | Описание |
|-----------|----------|
| Auth эндпоинты | `register`, `login`, `refresh`, `verify` |
| Push токены | Хранить FCM/APNs токены пользователей |
| WebSocket auth | Принимать access-токен вместо чат-токена |

Для [[WebSocket Чат|WebSocket]]-состояния рекомендуется мигрировать с in-memory на **Redis**.

---

## Push уведомления

```
Firebase Cloud Messaging (FCM)
    ├── Android → FCM напрямую
    └── iOS → FCM → APNs → устройство
```

**Триггеры уведомлений:**

| Событие | Уведомление |
|---------|-------------|
| Матч найден | "Партнёр найден! Заходи в чат" |
| Партнёр подключился | "Партнёр в комнате, подтверди участие" |
| Таймер подтверждения | "60 секунд до отмены" |
| Новое сообщение (фон) | "Новое сообщение от {nickname}" |
| Сессия начинается | "Сессия началась!" |

---

## Состояние WebSocket на сервере

| Вариант | Плюсы | Минусы |
|---------|-------|--------|
| In-memory (сейчас) | Просто | Теряется при рестарте, не масштабируется |
| **Redis Pub/Sub** | Масштабируется, персистентно | Нужен Redis |

---

## Безопасность

- Токены в Keychain / Keystore (не AsyncStorage)
- Certificate pinning для API запросов
- Jailbreak/root detection (опционально)
- Шифрование сообщений — оставить Fernet на сервере

---

## CI/CD

| Задача | Инструмент |
|--------|-----------|
| Сборка и дистрибуция | Expo EAS (React Native) |
| iOS | TestFlight → App Store |
| Android | Firebase App Distribution → Play Store |
