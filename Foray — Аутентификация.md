---
tags: [foray, auth, mobile, new-app]
---

# Foray — Аутентификация (без Telegram)

← [[🗺️ Foray — Карта проекта]] | Текущая система: [[Аутентификация — Telegram]]

## Задача

Заменить Telegram ID как единственный identity provider на собственную систему аккаунтов. Telegram может оставаться **опциональным** (например, для уведомлений), но не обязательным.

---

## Рекомендуемая схема: Email + Access/Refresh JWT

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
Отправить письмо подтверждения (Resend — уже есть)
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

## JWT структура для Foray

```json
{
  "user_id": "uuid",
  "nickname": "...",
  "type": "access",         // или "refresh"
  "expires_at": 1234567890
}
```

**Отличие от текущего:** токен не содержит данных матча (они передаются отдельно при подключении к WebSocket).

---

## Хранение токенов на мобильном

| Платформа | Хранилище |
|-----------|-----------|
| iOS | Keychain |
| Android | EncryptedSharedPreferences / Keystore |
| **Никогда** | AsyncStorage / localStorage (небезопасно) |

---

## Опциональный Telegram

Пользователь может **привязать** Telegram аккаунт в настройках:
- Для получения уведомлений через бота
- Для импорта существующего профиля (миграция с chat.lllang.site)

```
POST /api/auth/link-telegram
{ "telegram_id": "...", "telegram_init_data": "..." }
```

---

## Нужные изменения в Gateway

1. Новая таблица `accounts` с `email`, `password_hash`, `user_id`
2. Эндпоинт `POST /api/auth/register`
3. Эндпоинт `POST /api/auth/login`
4. Эндпоинт `POST /api/auth/refresh`
5. Эндпоинт `GET /api/auth/verify`
6. Поле `telegram_id` остаётся опциональным в профиле

---

## Безопасность

- Пароли: **bcrypt** (не md5/sha)
- Refresh токены хранить в БД (возможность отзыва)
- Access токены: HS256 с нормальным секретом (не `"lambo"`)
- Rate limiting на `/api/auth/login` против brute force
