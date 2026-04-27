# Story 1.1: Инициализация проекта EasyGo

Status: ready-for-dev

## Story

Как разработчик,
я хочу развернуть базовый проект EasyGo с БД, системой ролей и API-скелетом,
чтобы все последующие истории имели готовую инфраструктуру.

## Acceptance Criteria

1. **Given** чистый сервер / dev-среда  
   **When** разработчик выполняет первоначальную настройку  
   **Then** проект запускается без ошибок (сервер отвечает на `GET /health` → `200 OK`)

2. **Given** проект запущен  
   **When** разработчик проверяет БД  
   **Then** применены миграции и существуют таблицы: `users`, `roles`  
   **And** таблица `roles` содержит ровно 4 предустановленные записи: `founder`, `admin`, `driver`, `passenger`

3. **Given** проект запущен  
   **When** отправляется `POST /auth/login` с валидными учётными данными  
   **Then** возвращается JWT-токен (access + refresh)  
   **And** токен содержит поле `role` пользователя

4. **Given** JWT-токен получен  
   **When** отправляется запрос на защищённый эндпоинт с заголовком `Authorization: Bearer <token>`  
   **Then** запрос проходит авторизацию  
   **And** запрос без валидного токена возвращает `401 Unauthorized`

5. **Given** проект настроен  
   **When** разработчик проверяет конфигурацию  
   **Then** все секреты (API-ключи, строки подключения к БД, JWT-секрет) хранятся в переменных окружения (`.env`), не в коде  
   **And** `.env` добавлен в `.gitignore`

6. **Given** dev-среда настроена  
   **When** разработчик выполняет команду запуска тестов  
   **Then** базовый тест (`/health`) проходит без ошибок  
   **And** покрытие auth-модуля ≥ 80%

7. **Given** требования NFR5 (ФЗ-152, 242-ФЗ)  
   **When** настраивается конфигурация БД и хостинга  
   **Then** в README задокументировано требование: сервер БД должен физически находиться на территории РФ

## Tasks / Subtasks

- [ ] **T1: Инициализация репозитория и проекта** (AC: 5)
  - [ ] Создать Git-репозиторий с `.gitignore` (включить `.env`, `node_modules/` / `venv/`, `dist/`)
  - [ ] Добавить `.env.example` с описанием всех необходимых переменных без значений
  - [ ] Настроить линтер и форматтер (ESLint + Prettier для Node.js / flake8 + black для Python)

- [ ] **T2: Настройка базы данных и миграций** (AC: 2)
  - [ ] Подключить PostgreSQL через переменную окружения `DATABASE_URL`
  - [ ] Создать миграцию: таблица `roles` (id, name, created_at) + seed 4 ролей: founder, admin, driver, passenger
  - [ ] Создать миграцию: таблица `users` (id, phone, role_id FK→roles, status, created_at, updated_at)
  - [ ] Убедиться, что `npm run migrate` / `alembic upgrade head` применяет миграции на чистой БД

- [ ] **T3: JWT аутентификация** (AC: 3, 4)
  - [ ] Реализовать `POST /auth/login` — принимает phone + password, возвращает `{ access_token, refresh_token }`
  - [ ] Реализовать `POST /auth/refresh` — обновление access-токена по refresh-токену
  - [ ] JWT payload: `{ sub: userId, role: roleName, iat, exp }`
  - [ ] Access-токен TTL: 15 минут; Refresh-токен TTL: 7 дней
  - [ ] Middleware авторизации: проверяет подпись токена, возвращает 401 при невалидном

- [ ] **T4: API-скелет и health-check** (AC: 1)
  - [ ] Реализовать `GET /health` → `200 { status: "ok", timestamp: ISO8601 }`
  - [ ] Настроить CORS (разрешить домены из переменной окружения `ALLOWED_ORIGINS`)
  - [ ] Добавить глобальный обработчик ошибок с форматом ответа `{ error: string, code: string }`
  - [ ] Настроить structured logging (JSON-формат) с уровнями: info, warn, error

- [ ] **T5: Тесты** (AC: 6)
  - [ ] Unit-тест: генерация и валидация JWT
  - [ ] Integration-тест: `POST /auth/login` → успех, неверный пароль → 401
  - [ ] Integration-тест: защищённый маршрут без токена → 401, с токеном → 200
  - [ ] `GET /health` тест

- [ ] **T6: Документация** (AC: 7)
  - [ ] Создать `README.md`: требования к среде (Node ≥ 20 / Python ≥ 3.11), команды запуска, migrate, test
  - [ ] Задокументировать требование NFR5: "БД должна быть развёрнута на серверах в РФ (ФЗ-152, 242-ФЗ)"
  - [ ] Описать переменные окружения (ссылка на `.env.example`)

## Dev Notes

### Выбор технологического стека

**Стек не зафиксирован в архитектуре** — команда выбирает самостоятельно. Рекомендации:

| Компонент | Вариант A (Node.js) | Вариант B (Python) |
|-----------|---------------------|---------------------|
| Framework | NestJS + TypeScript | FastAPI + Python 3.11 |
| ORM / миграции | TypeORM + миграции | SQLAlchemy + Alembic |
| БД | PostgreSQL 15+ | PostgreSQL 15+ |
| Auth | `@nestjs/jwt` + Passport | `python-jose` + FastAPI Security |
| Тесты | Jest + Supertest | pytest + httpx |

**PostgreSQL обязателен** — последующие истории (синхронизация TaxiMaster, рефералы) требуют сложных JOIN-запросов и транзакций.

### Схема таблиц (минимум для этой истории)

```sql
-- roles
CREATE TABLE roles (
  id   SERIAL PRIMARY KEY,
  name VARCHAR(50) UNIQUE NOT NULL  -- founder, admin, driver, passenger
);

-- users (базовая структура; расширяется в историях 1.3, 1.5)
CREATE TABLE users (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone      VARCHAR(20) UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  role_id    INTEGER REFERENCES roles(id),
  status     VARCHAR(20) DEFAULT 'active',  -- active, suspended, blocked
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

**Важно:** `id` должен быть UUID (не serial integer) — таблица будет использоваться во всех последующих историях и в реферальных ссылках.

### JWT структура

```json
{
  "sub": "uuid-пользователя",
  "role": "driver",
  "iat": 1713600000,
  "exp": 1713600900
}
```

Секрет JWT из `process.env.JWT_SECRET` / `os.environ["JWT_SECRET"]`. **Никогда не хардкодить.**

### Переменные окружения (`.env.example`)

```
DATABASE_URL=postgresql://user:password@localhost:5432/easygo
JWT_SECRET=your-secret-key-min-32-chars
JWT_REFRESH_SECRET=your-refresh-secret-min-32-chars
ALLOWED_ORIGINS=http://localhost:3000
PORT=3000
NODE_ENV=development
```

### Требования к безопасности (NFR6, NFR7)

- Пароли хэшируются через bcrypt (cost factor ≥ 12) — **не MD5, не SHA1**
- HTTPS обязателен в prod-окружении (настраивается на уровне reverse proxy / nginx)
- JWT-секрет ≥ 32 символов

### Связь с последующими историями

| История | Что потребует из этой |
|---------|----------------------|
| 1.2 Верификация телефона | Таблица `users`, endpoint POST /auth/* |
| 1.3 Регистрация водителя | Таблица `users`, роль `driver` |
| 1.4 Топливный бонус | Аудит-лог (NFR8) — добавить таблицу `audit_log` или расширить users |
| 2.2 Sync TaxiMaster | Таблица `rides` (создаётся в 2.2), схема БД |
| 5.1 Мастер-панель | Роли `founder`, `admin` из этой истории |

**⚠️ Не создавать таблицы rides, referrals, ratings здесь** — каждая история создаёт свои таблицы через миграции.

### Project Structure Notes

Greenfield-проект: структура папок создаётся с нуля. Рекомендуемая структура (NestJS):

```
easygo-backend/
├── src/
│   ├── auth/           # JWT, login, refresh
│   ├── users/          # user entity (расширяется в 1.3, 1.5)
│   ├── common/         # guards, decorators, filters
│   └── main.ts
├── migrations/         # TypeORM или Alembic миграции
├── test/               # integration tests
├── .env.example
├── .gitignore
└── README.md
```

### TaxiMaster API контекст (справочно для этой истории)

TaxiMaster использует MD5-аутентификацию с заголовком `Signature`. Сервис интеграции разрабатывается в Истории 2.2 — в этой истории API TaxiMaster **не используется**.

### References

- [integration.md] — Архитектура интеграции EasyGo + TaxiMaster: двухуровневая схема
- [master-admin.md] — Роли доступа: founder / admin / driver (Принцип доступа)
- [epics.md — Story 1.1] — Acceptance Criteria исходной истории
- [epics.md — Additional Requirements] — NFR5 (серверы в РФ), NFR6 (HTTPS/TLS), NFR8 (аудит-лог)

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List
