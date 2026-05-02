---
tags: [foray, auth, mobile, new-app]
---

# Foray — Аутентификация

← [[🗺️ Foray — Карта проекта (Обновлено)]]

## Схема: Email + Access/Refresh JWT

### Регистрация

```
POST /api/auth/register
{
  "email": "user@example.com",
  "password": "...",
  "nickname": "...",
  "birthday": "2000-01-01",
  "gender": "male",
  "topics": ["travel", "tech"],
  "intro": "Hi!"
}
    ↓
Письмо подтверждения (Resend — уже есть в Gateway)
    ↓
GET /api/auth/verify?token=<email_verification_token>
    ↓
Аккаунт активирован
```

### Логин

```
POST /api/auth/login
{ "email": "...", "password": "..." }
    ↓
{
  "access_token": "...",   // TTL: 15-30 мин
  "refresh_token": "..."   // TTL: 30 дней
}
```

### Refresh

```
POST /api/auth/refresh
{ "refresh_token": "..." }
    ↓
{ "access_token": "..." }
```

---

## JWT структура

```json
{
  "user_id": "uuid",
  "nickname": "...",
  "type": "access",
  "expires_at": 1234567890
}
```

Токен не содержит данных матча — они передаются отдельно при подключении к [[WebSocket Чат|WebSocket]].

---

## Хранение токенов на мобильном

| Платформа | Хранилище |
|-----------|-----------|
| iOS | Keychain |
| Android | EncryptedSharedPreferences / Keystore |
| **Никогда** | AsyncStorage / localStorage |

---

## Нужные изменения в Gateway

1. Новая таблица `accounts` с `email`, `password_hash`, `user_id`
2. `POST /api/auth/register`
3. `POST /api/auth/login`
4. `POST /api/auth/refresh`
5. `GET /api/auth/verify`

---

## Безопасность

- Пароли: **bcrypt**
- Refresh токены хранить в БД (возможность отзыва)
- Access токены: HS256 с надёжным секретом
- Rate limiting на `/api/auth/login` против brute force
