# API-контракт F-easy

Описание REST API бэкенда. Фронтенд работает только с этим API, не с внешними сервисами напрямую.

**Base URL:** `/api`

**Формат:** JSON

**Авторизация:** JWT в заголовке `Authorization: Bearer <token>` (для защищённых эндпоинтов).

---

## Общие правила

### Пагинация (списки)

Query-параметры:

| Параметр | Тип | По умолчанию | Описание |
|----------|-----|--------------|----------|
| `page` | integer | 1 | Номер страницы |
| `limit` | integer | 20 | Количество записей (макс. 100) |

Формат ответа:

```json
{
  "data": [],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 1327,
    "totalPages": 67
  }
}
```

### Ошибки

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Описание ошибки",
    "details": []
  }
}
```

| HTTP | code | Когда |
|------|------|-------|
| 400 | `VALIDATION_ERROR` | Невалидные данные |
| 401 | `UNAUTHORIZED` | Нет или невалидный токен |
| 403 | `FORBIDDEN` | Нет прав |
| 404 | `NOT_FOUND` | Ресурс не найден |
| 429 | `RATE_LIMIT` | Превышен лимит |
| 500 | `INTERNAL_ERROR` | Ошибка сервера |
| 409 | `CONFLICT` | Конфликт данных (например, email уже занят) |

---

## Auth — регистрация и авторизация

Аутентификация через email и пароль. После успешного входа или регистрации бэкенд возвращает JWT. Роли: `user` (по умолчанию), `admin`.

Токены:
- **access token** — короткий срок жизни (например, 15 мин), передаётся в `Authorization: Bearer`
- **refresh token** — длинный срок (например, 7 дней), используется только для обновления access token

Пароль в БД хранится как bcrypt-хеш. В ответах API пароль никогда не возвращается.

### Модель `User` (публичная, без пароля)

```json
{
  "id": 1,
  "email": "user@example.com",
  "name": "Алексей",
  "role": "user",
  "hintsEnabled": true,
  "createdAt": "2026-06-01T10:00:00.000Z"
}
```

### Модель `AuthTokens`

```json
{
  "accessToken": "eyJhbG...",
  "refreshToken": "eyJhbG...",
  "expiresIn": 900,
  "user": { /* User */ }
}
```

---

### `POST /api/auth/register`

Регистрация нового пользователя.

**Доступ:** гость

**Body:**

```json
{
  "email": "user@example.com",
  "password": "securePass123",
  "name": "Алексей"
}
```

| Поле | Тип | Обязательно | Правила |
|------|-----|-------------|---------|
| `email` | string | да | Валидный email, уникальный |
| `password` | string | да | Мин. 8 символов |
| `name` | string | нет | Мин. 2, макс. 100 символов |

**Пример ответа `201`:** `AuthTokens`

**Ошибки:**
- `400 VALIDATION_ERROR` — невалидные поля
- `409 CONFLICT` — email уже зарегистрирован

---

### `POST /api/auth/login`

Вход по email и паролю.

**Доступ:** гость

**Body:**

```json
{
  "email": "user@example.com",
  "password": "securePass123"
}
```

**Пример ответа `200`:** `AuthTokens`

**Ошибки:**
- `400 VALIDATION_ERROR` — пустые поля
- `401 UNAUTHORIZED` — неверный email или пароль

---

### `POST /api/auth/refresh`

Получение нового access token по refresh token.

**Доступ:** гость (токен в body, не в заголовке)

**Body:**

```json
{
  "refreshToken": "eyJhbG..."
}
```

**Пример ответа `200`:**

```json
{
  "accessToken": "eyJhbG...",
  "expiresIn": 900
}
```

**Ошибки:**
- `401 UNAUTHORIZED` — refresh token невалиден или истёк

---

### `POST /api/auth/logout`

Выход. Refresh token помечается как отозванный на бэкенде (если хранится в БД).

**Доступ:** user, admin

**Body (опционально):**

```json
{
  "refreshToken": "eyJhbG..."
}
```

**Пример ответа `200`:**

```json
{
  "message": "Logged out"
}
```

---

### `GET /api/auth/me`

Текущий авторизованный пользователь.

**Доступ:** user, admin

**Headers:** `Authorization: Bearer <accessToken>`

**Пример ответа `200`:** модель `User`

**Ошибки:**
- `401 UNAUTHORIZED` — токен отсутствует, истёк или невалиден

---

### `PATCH /api/auth/me`

Обновление профиля.

**Доступ:** user, admin

**Body (все поля опциональны):**

```json
{
  "name": "Алексей Иванов",
  "hintsEnabled": false
}
```

| Поле | Тип | Правила |
|------|-----|---------|
| `name` | string | Мин. 2, макс. 100 символов |
| `hintsEnabled` | boolean | Вкл/выкл подсказки умного помощника |

**Пример ответа `200`:** обновлённая модель `User`

**Ошибки:**
- `400 VALIDATION_ERROR`
- `401 UNAUTHORIZED`

---

### `PATCH /api/auth/password`

Смена пароля.

**Доступ:** user, admin

**Body:**

```json
{
  "currentPassword": "oldPass123",
  "newPassword": "newSecurePass456"
}
```

| Поле | Тип | Обязательно | Правила |
|------|-----|-------------|---------|
| `currentPassword` | string | да | Текущий пароль |
| `newPassword` | string | да | Мин. 8 символов, отличается от текущего |

**Пример ответа `200`:**

```json
{
  "message": "Password updated"
}
```

**Ошибки:**
- `400 VALIDATION_ERROR` — новый пароль слишком короткий или совпадает с текущим
- `401 UNAUTHORIZED` — неверный текущий пароль или невалидный токен

После смены пароля все refresh token пользователя отзываются — нужно войти заново.

---

### `GET /api/auth/activity`

История действий пользователя в личном кабинете. Записи создаются на бэкенде при ключевых операциях (вход, создание программы, сохранение метрик и т.д.).

**Доступ:** user, admin (только свои записи)

**Query-параметры:** `page`, `limit`

**Пример ответа `200`:**

```json
{
  "data": [
    {
      "id": 15,
      "action": "program_created",
      "description": "Создана программа «Моя программа на гипертрофию»",
      "metadata": {
        "programId": 3
      },
      "createdAt": "2026-06-15T14:30:00.000Z"
    },
    {
      "id": 14,
      "action": "login",
      "description": "Вход в аккаунт",
      "metadata": {},
      "createdAt": "2026-06-15T14:28:00.000Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 42,
    "totalPages": 3
  }
}
```

**Типы действий (`action`):**

| action | Когда записывается |
|--------|-------------------|
| `register` | Регистрация |
| `login` | Успешный вход |
| `profile_updated` | Изменение имени или настроек |
| `password_changed` | Смена пароля |
| `program_created` | Создание программы |
| `program_updated` | Изменение программы |
| `program_deleted` | Удаление программы |
| `body_metric_added` | Добавление записи прогресса |
| `calculator_saved` | Сохранение результата калькулятора |

**Ошибки:**
- `401 UNAUTHORIZED`

---

### Таблица `user_activities` (для истории действий)

| Колонка | Тип | Описание |
|---------|-----|----------|
| `id` | INTEGER PK | |
| `user_id` | INTEGER FK → users | |
| `action` | VARCHAR | Тип действия |
| `description` | TEXT | Текст для отображения в UI |
| `metadata` | JSONB | Доп. данные (id программы и т.д.) |
| `created_at` | TIMESTAMP | |

---

### Env-переменные (auth)

| Переменная | Описание |
|------------|----------|
| `JWT_ACCESS_SECRET` | Секрет для access token |
| `JWT_REFRESH_SECRET` | Секрет для refresh token |
| `JWT_ACCESS_EXPIRES` | Срок access token, например `15m` |
| `JWT_REFRESH_EXPIRES` | Срок refresh token, например `7d` |

---

## Exercises — каталог упражнений

Упражнения хранятся в PostgreSQL. Источник данных — [WorkoutX API](https://workoutxapp.com). Синхронизация выполняется на бэкенде, фронт читает только локальный каталог.

### Модель `Exercise` (ответ API)

```json
{
  "id": 1,
  "externalId": "0001",
  "name": "3/4 Sit-up",
  "bodyPart": "Waist",
  "targetMuscle": "Abs",
  "secondaryMuscles": ["Hip Flexors", "Lower Back"],
  "exerciseType": "strength",
  "difficulty": "beginner",
  "equipment": "Body Weight",
  "description": "3/4 Sit-up is a beginner...",
  "instructions": ["Lie flat on your back...", "..."],
  "gifUrl": "https://api.workoutxapp.com/v1/gifs/0001.gif",
  "source": "workoutx"
}
```

---

### `GET /api/exercises`

Список упражнений из локальной БД с поиском, фильтрами и пагинацией.

**Доступ:** гость, user, admin

**Query-параметры:**

| Параметр | Тип | Описание |
|----------|-----|----------|
| `search` | string | Поиск по названию (частичное совпадение) |
| `bodyPart` | string | Часть тела (Waist, Chest, Back...) |
| `target` | string | Целевая мышца (Abs, Biceps, Glutes...) |
| `equipment` | string | Оборудование (Barbell, Body Weight...) |
| `difficulty` | string | `beginner`, `intermediate`, `expert` |
| `exerciseType` | string | `strength`, `cardio`, `stretching` и др. |
| `page` | integer | Страница |
| `limit` | integer | Размер страницы |

**Пример запроса:**

```
GET /api/exercises?target=Biceps&difficulty=beginner&page=1&limit=20
```

**Пример ответа `200`:**

```json
{
  "data": [
    {
      "id": 42,
      "externalId": "0001",
      "name": "3/4 Sit-up",
      "bodyPart": "Waist",
      "targetMuscle": "Abs",
      "exerciseType": "strength",
      "difficulty": "beginner",
      "equipment": "Body Weight",
      "gifUrl": "https://api.workoutxapp.com/v1/gifs/0001.gif"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 1327,
    "totalPages": 67
  }
}
```

---

### `GET /api/exercises/:id`

Одно упражнение по внутреннему id (PK в PostgreSQL).

**Доступ:** гость, user, admin

**Пример ответа `200`:** полная модель `Exercise` (см. выше).

**Ошибки:** `404 NOT_FOUND` — упражнение не существует.

---

### `GET /api/exercises/filters`

Списки значений для фильтров на фронте. Берутся из уникальных значений в БД.

**Доступ:** гость, user, admin

**Пример ответа `200`:**

```json
{
  "bodyParts": ["Back", "Chest", "Waist", "Upper Legs"],
  "targets": ["Abs", "Biceps", "Glutes", "Lats"],
  "equipment": ["Barbell", "Body Weight", "Dumbbell"],
  "difficulties": ["beginner", "intermediate", "expert"],
  "exerciseTypes": ["strength", "cardio", "stretching"]
}
```

---

### `POST /api/admin/exercises/sync`

Синхронизация каталога с WorkoutX API. Бэкенд запрашивает упражнения пачками (`limit` + `offset`), upsert в таблицу `exercises` по `external_id`.

**Доступ:** admin

**Body:** пустой или опционально:

```json
{
  "fullSync": true
}
```

**Пример ответа `200`:**

```json
{
  "synced": 1327,
  "created": 1327,
  "updated": 0,
  "skipped": 0
}
```

**Ошибки:**
- `403 FORBIDDEN` — не admin
- `502 EXTERNAL_API_ERROR` — WorkoutX недоступен или ключ невалиден

**Заметка по лимитам:** free план WorkoutX ограничивает число запросов. Синхронизацию запускать редко (при первом деплое или вручную из админки), не при каждом заходе пользователя.

---

## WorkoutX — внешний API (только бэкенд)

Используется в service-слое при sync. Не вызывается с фронта.

| Параметр | Значение |
|----------|----------|
| Base URL | `https://api.workoutxapp.com/v1` |
| Заголовок | `X-WorkoutX-Key: <WORKOUTX_API_KEY>` |
| Env | `WORKOUTX_API_KEY` в `.env` |

**Эндпоинты для sync:**

| WorkoutX | Назначение |
|----------|------------|
| `GET /exercises?limit=100&offset=N` | Пагинация всего каталога |
| `GET /exercises/exercise/:id` | Одно упражнение по external id |

**Маппинг полей WorkoutX → PostgreSQL:**

| WorkoutX | Колонка в `exercises` |
|----------|----------------------|
| `id` | `external_id` |
| `name` | `name` |
| `bodyPart` | `body_part` |
| `target` | `target_muscle` |
| `secondaryMuscles` | `secondary_muscles` (JSONB) |
| `category` | `exercise_type` |
| `difficulty` | `difficulty` |
| `equipment` | `equipment` |
| `instructions` | `instructions` (JSONB) |
| `description` | `description` |
| `gifUrl` | `gif_url` |
| — | `source` = `'workoutx'` |

---

## Programs — тренировочные программы

Программа принадлежит одному пользователу. Доступ только владельцу (или admin при необходимости в админке). При создании, изменении и удалении записывается событие в `user_activities`.

### Модель `WorkoutProgram` (список)

```json
{
  "id": 3,
  "title": "Моя программа на гипертрофию",
  "description": "Тренировки 3 раза в неделю",
  "goal": "hypertrophy",
  "exerciseCount": 8,
  "createdAt": "2026-06-10T12:00:00.000Z",
  "updatedAt": "2026-06-15T14:30:00.000Z"
}
```

### Модель `ProgramExercise`

```json
{
  "id": 12,
  "exerciseId": 42,
  "setsCount": 3,
  "repsCount": 12,
  "sortOrder": 1,
  "note": "Медленный негатив",
  "exercise": {
    "id": 42,
    "name": "3/4 Sit-up",
    "targetMuscle": "Abs",
    "bodyPart": "Waist",
    "difficulty": "beginner",
    "gifUrl": "https://api.workoutxapp.com/v1/gifs/0001.gif"
  }
}
```

### Модель `WorkoutProgramDetail` (полная программа)

```json
{
  "id": 3,
  "title": "Моя программа на гипертрофию",
  "description": "Тренировки 3 раза в неделю",
  "goal": "hypertrophy",
  "createdAt": "2026-06-10T12:00:00.000Z",
  "updatedAt": "2026-06-15T14:30:00.000Z",
  "exercises": [ /* ProgramExercise[] */ ]
}
```

**Цели программы (`goal`):** `hypertrophy`, `strength`, `weight_loss`, `endurance`, `general` (или `null`).

---

### `GET /api/programs`

Список программ текущего пользователя.

**Доступ:** user, admin

**Query-параметры:** `page`, `limit`, `search` (поиск по названию)

**Пример ответа `200`:**

```json
{
  "data": [
    {
      "id": 3,
      "title": "Моя программа на гипертрофию",
      "description": "Тренировки 3 раза в неделю",
      "goal": "hypertrophy",
      "exerciseCount": 8,
      "createdAt": "2026-06-10T12:00:00.000Z",
      "updatedAt": "2026-06-15T14:30:00.000Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 5,
    "totalPages": 1
  }
}
```

---

### `POST /api/programs`

Создание новой программы.

**Доступ:** user, admin

**Body:**

```json
{
  "title": "Моя программа на гипертрофию",
  "description": "Тренировки 3 раза в неделю",
  "goal": "hypertrophy"
}
```

| Поле | Тип | Обязательно | Правила |
|------|-----|-------------|---------|
| `title` | string | да | Мин. 2, макс. 150 символов |
| `description` | string | нет | Макс. 2000 символов |
| `goal` | string | нет | Одно из значений `goal` |

**Пример ответа `201`:** `WorkoutProgramDetail` (с пустым `exercises: []`)

**Ошибки:** `400 VALIDATION_ERROR`, `401 UNAUTHORIZED`

---

### `GET /api/programs/:id`

Программа с упражнениями, отсортированными по `sortOrder`.

**Доступ:** user, admin (владелец)

**Пример ответа `200`:** `WorkoutProgramDetail`

**Ошибки:**
- `401 UNAUTHORIZED`
- `403 FORBIDDEN` — программа другого пользователя
- `404 NOT_FOUND`

---

### `PATCH /api/programs/:id`

Обновление названия, описания или цели.

**Доступ:** user, admin (владелец)

**Body (все поля опциональны):**

```json
{
  "title": "Новое название",
  "description": "Обновлённое описание",
  "goal": "strength"
}
```

**Пример ответа `200`:** `WorkoutProgramDetail`

**Ошибки:** `400`, `401`, `403`, `404`

---

### `DELETE /api/programs/:id`

Удаление программы и всех связанных `program_exercises` и `training_hints` (каскад или явное удаление в service).

**Доступ:** user, admin (владелец)

**Пример ответа `200`:**

```json
{
  "message": "Program deleted"
}
```

**Ошибки:** `401`, `403`, `404`

---

### `GET /api/programs/:id/export`

Экспорт программы в текстовом формате для копирования или скачивания `.txt` на фронте.

**Доступ:** user, admin (владелец)

**Query-параметры:**

| Параметр | Тип | По умолчанию | Описание |
|----------|-----|--------------|----------|
| `format` | string | `json` | `json` — объект с текстом; `text` — plain text в body |

**Пример ответа `200` (`format=json`):**

```json
{
  "title": "Моя программа на гипертрофию",
  "text": "Моя программа на гипертрофию\n\n1. 3/4 Sit-up — 3 x 12\n   Заметка: Медленный негатив\n\n2. Barbell Curl — 4 x 10\n"
}
```

**Пример ответа `200` (`format=text`):** Content-Type `text/plain; charset=utf-8`

**Ошибки:** `401`, `403`, `404`

---

## Program exercises — упражнения в программе

Упражнения добавляются из локального каталога (`exercises.id`). Одно и то же упражнение можно добавить в программу только один раз.

---

### `POST /api/programs/:programId/exercises`

Добавить упражнение в программу.

**Доступ:** user, admin (владелец программы)

**Body:**

```json
{
  "exerciseId": 42,
  "setsCount": 3,
  "repsCount": 12,
  "note": "Медленный негатив"
}
```

| Поле | Тип | Обязательно | Правила |
|------|-----|-------------|---------|
| `exerciseId` | integer | да | Существующий id в `exercises` |
| `setsCount` | integer | нет | 1–20, по умолчанию 3 |
| `repsCount` | integer | нет | 1–100, по умолчанию 10 |
| `note` | string | нет | Макс. 500 символов |

`sortOrder` назначается автоматически (max + 1).

**Пример ответа `201`:** `ProgramExercise`

**Ошибки:**
- `400 VALIDATION_ERROR` — упражнение уже в программе
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `404 NOT_FOUND` — программа или упражнение не найдены

После добавления пересчитываются подсказки умного помощника (если `hintsEnabled`).

---

### `PATCH /api/programs/:programId/exercises/:id`

Изменить подходы, повторения, заметку или порядок одного упражнения в программе.

**Доступ:** user, admin (владелец)

**Body (все поля опциональны):**

```json
{
  "setsCount": 4,
  "repsCount": 10,
  "sortOrder": 2,
  "note": "Пауза 2 сек внизу"
}
```

| Поле | Тип | Правила |
|------|-----|---------|
| `setsCount` | integer | 1–20 |
| `repsCount` | integer | 1–100 |
| `sortOrder` | integer | ≥ 1 |
| `note` | string | Макс. 500 символов |

**Пример ответа `200`:** `ProgramExercise`

**Ошибки:** `400`, `401`, `403`, `404`

---

### `PATCH /api/programs/:programId/exercises/reorder`

Массовое изменение порядка упражнений в конструкторе (drag-and-drop).

**Доступ:** user, admin (владелец)

**Body:**

```json
{
  "order": [
    { "id": 12, "sortOrder": 1 },
    { "id": 15, "sortOrder": 2 },
    { "id": 18, "sortOrder": 3 }
  ]
}
```

`order` — полный список `program_exercises.id` программы с новыми позициями.

**Пример ответа `200`:** `WorkoutProgramDetail`

**Ошибки:** `400 VALIDATION_ERROR` — id не из этой программы, `401`, `403`, `404`

---

### `DELETE /api/programs/:programId/exercises/:id`

Удалить упражнение из программы (не из каталога).

**Доступ:** user, admin (владелец)

**Пример ответа `200`:**

```json
{
  "message": "Exercise removed from program"
}
```

**Ошибки:** `401`, `403`, `404`

После удаления пересчитываются подсказки.

---

## Hints — умный помощник программы

Анализирует состав программы и генерирует подсказки для новичка. Логика живёт в **service-слое** бэкенда. Результаты сохраняются в `training_hints` и обновляются при изменении программы (добавление, удаление, изменение упражнений, reorder).

Если у пользователя `hintsEnabled = false` (настройка в профиле), подсказки не показываются — API возвращает пустой список.

Подсказки не заменяют консультацию тренера или врача — это эвристики для базовой проверки баланса.

### Модель `TrainingHint`

```json
{
  "id": 7,
  "hintType": "muscle_overload",
  "text": "В программе 5 упражнений на Biceps — возможна перегрузка одной группы мышц.",
  "importance": "warning",
  "metadata": {
    "targetMuscle": "Biceps",
    "exerciseCount": 5
  },
  "createdAt": "2026-06-15T14:35:00.000Z"
}
```

**Типы подсказок (`hintType`):**

| hintType | Описание |
|----------|----------|
| `program_empty` | В программе нет упражнений |
| `program_short` | Мало упражнений для полноценной тренировки |
| `muscle_overload` | Слишком много упражнений на одну целевую мышцу |
| `muscle_missing` | Почти нет упражнений на важную группу |
| `difficulty_high` | Слишком много сложных упражнений |
| `volume_high` | Слишком большой объём подходов на одну группу |

**Уровни важности (`importance`):** `info`, `warning`, `critical`

---

### Правила анализа (бизнес-логика)

Константы задаются в service (можно вынести в config). Анализ идёт по `target_muscle` упражнений в программе.

| Правило | Условие | hintType | importance |
|---------|---------|----------|------------|
| Пустая программа | 0 упражнений | `program_empty` | `info` |
| Короткая программа | 1–2 упражнения | `program_short` | `info` |
| Перегруз мышцы | ≥ 4 упражнений на одну `target_muscle` | `muscle_overload` | `warning` |
| Пропущена группа | ≥ 5 упражнений в программе, но 0 на ноги (`Quadriceps`, `Hamstrings`, `Glutes`) или спину (`Lats`, `Middle Back`) | `muscle_missing` | `warning` |
| Высокая сложность | ≥ 3 упражнений с `difficulty = expert` | `difficulty_high` | `warning` |
| Большой объём | сумма `setsCount` по одной `target_muscle` ≥ 20 | `volume_high` | `critical` |

При нескольких нарушений для одной мышцы — одна подсказка с наиболее важным типом (volume > overload).

Текст подсказки (`text`) генерируется на бэкенде по шаблону с подстановкой названия мышцы и чисел.

**Когда пересчитывать:**
- `POST /api/programs/:programId/exercises`
- `PATCH /api/programs/:programId/exercises/:id`
- `DELETE /api/programs/:programId/exercises/:id`
- `PATCH /api/programs/:programId/exercises/reorder`
- `POST /api/programs/:id/hints/recalculate` (явный запрос)

При пересчёте старые подсказки программы удаляются и записываются новые.

---

### `GET /api/programs/:programId/hints`

Список актуальных подсказок для программы.

**Доступ:** user, admin (владелец программы)

**Пример ответа `200` (подсказки включены):**

```json
{
  "programId": 3,
  "hintsEnabled": true,
  "analyzedAt": "2026-06-15T14:35:00.000Z",
  "hints": [
    {
      "id": 7,
      "hintType": "muscle_overload",
      "text": "В программе 5 упражнений на Biceps — возможна перегрузка одной группы мышц.",
      "importance": "warning",
      "metadata": {
        "targetMuscle": "Biceps",
        "exerciseCount": 5
      },
      "createdAt": "2026-06-15T14:35:00.000Z"
    },
    {
      "id": 8,
      "hintType": "muscle_missing",
      "text": "В программе почти нет упражнений на спину. Для баланса добавьте работу на Lats или Middle Back.",
      "importance": "warning",
      "metadata": {
        "missingGroups": ["Lats", "Middle Back"]
      },
      "createdAt": "2026-06-15T14:35:00.000Z"
    }
  ]
}
```

**Пример ответа `200` (подсказки отключены в профиле):**

```json
{
  "programId": 3,
  "hintsEnabled": false,
  "analyzedAt": null,
  "hints": []
}
```

**Ошибки:** `401`, `403`, `404`

---

### `POST /api/programs/:programId/hints/recalculate`

Принудительный пересчёт подсказок. Полезно для отладки и если нужно обновить без изменения упражнений.

**Доступ:** user, admin (владелец)

**Body:** пустой

**Пример ответа `200`:** тот же формат, что `GET /api/programs/:programId/hints`

Если `hintsEnabled = false`, пересчёт не выполняется — возвращается пустой список.

**Ошибки:** `401`, `403`, `404`

---

### Встраивание в конструктор (фронт)

1. После изменения программы фронт может запросить `GET /api/programs/:id/hints` или использовать подсказки из ответа после mutate.
2. Подсказки отображаются в блоке помощника рядом со списком упражнений.
3. Цвет/иконка по `importance`: `info` — нейтральный, `warning` — жёлтый, `critical` — красный.
4. При `hintsEnabled = false` блок помощника скрыт или показывает «Подсказки отключены в настройках».

---

## Body metrics — прогресс в личном кабинете

Записи параметров тела по датам. Раздел «Прогресс» в профиле: таблица и график динамики веса, ИМТ и других показателей.

ИМТ (`bmi`) рассчитывается на бэкенде при сохранении, если указаны вес и рост:

`bmi = weightKg / (heightCm / 100)²`

### Модель `BodyMetric`

```json
{
  "id": 10,
  "recordedAt": "2026-06-15",
  "weightKg": 78.5,
  "heightCm": 175,
  "bmi": 25.6,
  "bodyFatPercent": 18.2,
  "createdAt": "2026-06-15T10:00:00.000Z"
}
```

| Поле | Тип | Описание |
|------|-----|----------|
| `recordedAt` | date | Дата записи (не дата создания в системе) |
| `weightKg` | number | Вес, кг |
| `heightCm` | number | Рост, см |
| `bmi` | number | Рассчитанный ИМТ |
| `bodyFatPercent` | number | Процент жира, опционально |

---

### `GET /api/body-metrics`

Список записей прогресса текущего пользователя.

**Доступ:** user, admin

**Query-параметры:**

| Параметр | Тип | Описание |
|----------|-----|----------|
| `page`, `limit` | integer | Пагинация |
| `from`, `to` | date | Фильтр по `recordedAt` (ISO date) |
| `sort` | string | `recordedAt` (по умолчанию `-recordedAt` — новые первые) |

**Пример ответа `200`:**

```json
{
  "data": [
    {
      "id": 10,
      "recordedAt": "2026-06-15",
      "weightKg": 78.5,
      "heightCm": 175,
      "bmi": 25.6,
      "bodyFatPercent": 18.2,
      "createdAt": "2026-06-15T10:00:00.000Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 12,
    "totalPages": 1
  }
}
```

---

### `GET /api/body-metrics/chart`

Данные для графика в личном кабинете (упрощённый формат для фронта).

**Доступ:** user, admin

**Query-параметры:** `from`, `to` (опционально)

**Пример ответа `200`:**

```json
{
  "weight": [
    { "date": "2026-05-01", "value": 80.0 },
    { "date": "2026-06-01", "value": 78.5 },
    { "date": "2026-06-15", "value": 77.8 }
  ],
  "bmi": [
    { "date": "2026-05-01", "value": 26.1 },
    { "date": "2026-06-01", "value": 25.6 },
    { "date": "2026-06-15", "value": 25.4 }
  ],
  "bodyFatPercent": [
    { "date": "2026-06-01", "value": 19.0 },
    { "date": "2026-06-15", "value": 18.2 }
  ]
}
```

Точки без значения (например, нет `bodyFatPercent`) не включаются в соответствующий массив.

---

### `POST /api/body-metrics`

Добавить запису прогресса.

**Доступ:** user, admin

**Body:**

```json
{
  "recordedAt": "2026-06-15",
  "weightKg": 78.5,
  "heightCm": 175,
  "bodyFatPercent": 18.2
}
```

| Поле | Тип | Обязательно | Правила |
|------|-----|-------------|---------|
| `recordedAt` | date | да | Не в будущем |
| `weightKg` | number | нет* | 30–300 кг |
| `heightCm` | number | нет* | 100–250 см |
| `bodyFatPercent` | number | нет | 3–60 |

\* Для расчёта ИМТ нужны `weightKg` и `heightCm`. Можно сохранить только вес или только процент жира.

**Пример ответа `201`:** `BodyMetric`

**Ошибки:** `400 VALIDATION_ERROR`, `401 UNAUTHORIZED`

Записывается событие `body_metric_added` в `user_activities`.

---

### `GET /api/body-metrics/:id`

Одна записа.

**Доступ:** user, admin (владелец)

**Ошибки:** `401`, `403`, `404`

---

### `PATCH /api/body-metrics/:id`

Обновление записи. При изменении веса/роста ИМТ пересчитывается.

**Доступ:** user, admin (владелец)

**Body:** любые поля из `POST` (все опциональны)

**Пример ответа `200`:** `BodyMetric`

---

### `DELETE /api/body-metrics/:id`

**Доступ:** user, admin (владелец)

**Пример ответа `200`:**

```json
{
  "message": "Body metric deleted"
}
```

---

## Calculators — калькуляторы и личный кабинет

Три калькулятора: ИМТ, суточная норма калорий, БЖУ. Гость может **рассчитать** без авторизации. **Сохранить** результат в профиль — только авторизованный пользователь (`saveToProfile: true`).

Расчёт выполняется в service-слое. Формулы:

**ИМТ:** `weight / (height/100)²`. Категории: `<18.5` — низкий, `18.5–24.9` — норма, `25–29.9` — избыточный, `≥30` — высокий.

**Калории (Mifflin–St Jeor):**
- муж: `BMR = 10×вес + 6.25×рост − 5×возраст + 5`
- жен: `BMR = 10×вес + 6.25×рост − 5×возраст − 161`
- `TDEE = BMR × множитель активности` (1.2 / 1.375 / 1.55 / 1.725 / 1.9)

**БЖУ:** из `dailyCalories` и цели (`balanced`, `low_carb`, `high_protein`) — проценты белков, жиров, углеводов и граммы.

### Модель `CalculatorResult` (сохранённый расчёт)

```json
{
  "id": 5,
  "calculatorType": "calories",
  "inputData": {
    "weightKg": 78,
    "heightCm": 175,
    "age": 25,
    "gender": "male",
    "activityLevel": "moderate"
  },
  "resultData": {
    "bmr": 1756,
    "tdee": 2722,
    "goalCalories": 2722
  },
  "calculatedAt": "2026-06-15T11:00:00.000Z"
}
```

**Типы (`calculatorType`):** `bmi`, `calories`, `macros`

---

### `POST /api/calculators/bmi`

**Доступ:** гость, user, admin

**Body:**

```json
{
  "weightKg": 78,
  "heightCm": 175,
  "saveToProfile": false
}
```

| Поле | Тип | Обязательно | Правила |
|------|-----|-------------|---------|
| `weightKg` | number | да | 30–300 |
| `heightCm` | number | да | 100–250 |
| `saveToProfile` | boolean | нет | Только для авторизованных; сохраняет в `calculator_results` |

**Пример ответа `200`:**

```json
{
  "bmi": 25.6,
  "category": "overweight",
  "categoryLabel": "Избыточный вес",
  "saved": false,
  "savedResultId": null
}
```

При `saveToProfile: true` и авторизации: `saved: true`, `savedResultId: 5`. Опционально создаётся записа в `body_metrics` с теми же весом, ростом и ИМТ на текущую дату.

---

### `POST /api/calculators/calories`

Суточная норма калорий.

**Доступ:** гость, user, admin

**Body:**

```json
{
  "weightKg": 78,
  "heightCm": 175,
  "age": 25,
  "gender": "male",
  "activityLevel": "moderate",
  "goal": "maintain",
  "saveToProfile": false
}
```

| Поле | Тип | Обязательно | Значения |
|------|-----|-------------|----------|
| `gender` | string | да | `male`, `female` |
| `activityLevel` | string | да | `sedentary`, `light`, `moderate`, `active`, `very_active` |
| `goal` | string | нет | `maintain`, `lose`, `gain` (корректировка ±300–500 ккал) |

**Пример ответа `200`:**

```json
{
  "bmr": 1756,
  "tdee": 2722,
  "goalCalories": 2722,
  "goal": "maintain",
  "saved": false,
  "savedResultId": null
}
```

---

### `POST /api/calculators/macros`

Расчёт БЖУ.

**Доступ:** гость, user, admin

**Body:**

```json
{
  "dailyCalories": 2200,
  "goal": "balanced",
  "weightKg": 78,
  "saveToProfile": false
}
```

| Поле | Тип | Обязательно | Значения |
|------|-----|-------------|----------|
| `dailyCalories` | number | да | 800–6000 |
| `goal` | string | нет | `balanced`, `low_carb`, `high_protein` |
| `weightKg` | number | нет | Для `high_protein` — расчёт белка от веса |

**Пример ответа `200`:**

```json
{
  "dailyCalories": 2200,
  "proteinGrams": 165,
  "fatGrams": 73,
  "carbsGrams": 248,
  "proteinPercent": 30,
  "fatPercent": 30,
  "carbsPercent": 40,
  "saved": false,
  "savedResultId": null
}
```

---

### `GET /api/calculators/results`

Сохранённые расчёты в личном кабинете.

**Доступ:** user, admin

**Query-параметры:** `page`, `limit`, `calculatorType` (фильтр: `bmi`, `calories`, `macros`)

**Пример ответа `200`:**

```json
{
  "data": [
    {
      "id": 5,
      "calculatorType": "calories",
      "inputData": { "weightKg": 78, "heightCm": 175, "age": 25 },
      "resultData": { "tdee": 2722, "goalCalories": 2722 },
      "calculatedAt": "2026-06-15T11:00:00.000Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 3,
    "totalPages": 1
  }
}
```

---

### `DELETE /api/calculators/results/:id`

Удалить сохранённый расчёт из профиля.

**Доступ:** user, admin (владелец)

**Пример ответа `200`:**

```json
{
  "message": "Calculator result deleted"
}
```

---

### `GET /api/profile/summary`

Сводка для личного кабинета: программы, последний прогресс, недавние расчёты.

**Доступ:** user, admin

**Пример ответа `200`:**

```json
{
  "programsCount": 3,
  "latestBodyMetric": {
    "recordedAt": "2026-06-15",
    "weightKg": 77.8,
    "bmi": 25.4
  },
  "recentCalculatorResults": [
    {
      "id": 5,
      "calculatorType": "calories",
      "calculatedAt": "2026-06-15T11:00:00.000Z",
      "summary": "2722 ккал / день"
    }
  ]
}
```

---

## Admin — админ-панель

Все эндпоинты ниже доступны только с ролью `admin`. Проверка в middleware: JWT + `role === 'admin'`.

Базовый путь: `/api/admin`

Уже описанные admin-эндпоинты:
- `POST /api/admin/exercises/sync` — синхронизация каталога упражнений (см. блок Exercises)

---

### `GET /api/admin/stats`

Сводка для дашборда админки.

**Доступ:** admin

**Пример ответа `200`:**

```json
{
  "usersCount": 128,
  "programsCount": 342,
  "exercisesCount": 1327,
  "contentItemsCount": 12,
  "publishedContentCount": 8
}
```

---

## Admin — пользователи

### Модель `AdminUser` (без пароля)

```json
{
  "id": 5,
  "email": "user@example.com",
  "name": "Алексей",
  "role": "user",
  "hintsEnabled": true,
  "programsCount": 3,
  "createdAt": "2026-05-01T08:00:00.000Z"
}
```

---

### `GET /api/admin/users`

Список пользователей.

**Доступ:** admin

**Query-параметры:**

| Параметр | Тип | Описание |
|----------|-----|----------|
| `page`, `limit` | integer | Пагинация |
| `search` | string | Поиск по email или имени |
| `role` | string | Фильтр: `user`, `admin` |

**Пример ответа `200`:**

```json
{
  "data": [
    {
      "id": 5,
      "email": "user@example.com",
      "name": "Алексей",
      "role": "user",
      "hintsEnabled": true,
      "programsCount": 3,
      "createdAt": "2026-05-01T08:00:00.000Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 128,
    "totalPages": 7
  }
}
```

---

### `GET /api/admin/users/:id`

Детали пользователя + счётчики.

**Доступ:** admin

**Пример ответа `200`:**

```json
{
  "id": 5,
  "email": "user@example.com",
  "name": "Алексей",
  "role": "user",
  "hintsEnabled": true,
  "createdAt": "2026-05-01T08:00:00.000Z",
  "stats": {
    "programsCount": 3,
    "bodyMetricsCount": 12,
    "calculatorResultsCount": 5
  }
}
```

**Ошибки:** `404 NOT_FOUND`

---

### `PATCH /api/admin/users/:id`

Изменение роли пользователя.

**Доступ:** admin

**Body:**

```json
{
  "role": "admin"
}
```

| Поле | Тип | Значения |
|------|-----|----------|
| `role` | string | `user`, `admin` |

Админ не может понизить свою собственную роль до `user` (защита от случайной блокировки админки).

**Пример ответа `200`:** `AdminUser`

**Ошибки:** `400`, `403 FORBIDDEN` — попытка изменить свою роль, `404`

---

### `DELETE /api/admin/users/:id`

Удаление пользователя и связанных данных (программы, метрики, расчёты, активность). Каскад в service или FK ON DELETE CASCADE.

**Доступ:** admin

Не допускается удаление собственного аккаунта.

**Пример ответа `200`:**

```json
{
  "message": "User deleted"
}
```

**Ошибки:** `403`, `404`

---

## Admin — контент

Материалы для главной страницы, гайдов и будущих разделов портала. Админ создаёт, редактирует, скрывает.

### Модель `ContentItem`

```json
{
  "id": 1,
  "contentType": "article",
  "title": "Как начать тренироваться",
  "body": "Текст материала в Markdown или plain text...",
  "status": "published",
  "authorId": 1,
  "authorName": "Админ",
  "createdAt": "2026-06-01T10:00:00.000Z",
  "updatedAt": "2026-06-10T12:00:00.000Z"
}
```

**Типы (`contentType`):** `article`, `guide`, `announcement`

**Статусы (`status`):** `draft`, `published`, `hidden`

| status | Видимость |
|--------|-----------|
| `draft` | Только админка |
| `published` | Публичный API и сайт |
| `hidden` | Скрыт с сайта, виден в админке |

---

### `GET /api/admin/content`

Список всех материалов (включая draft и hidden).

**Доступ:** admin

**Query-параметры:** `page`, `limit`, `search`, `contentType`, `status`

**Пример ответа `200`:** пагинированный список `ContentItem`

---

### `POST /api/admin/content`

Создание материала.

**Доступ:** admin

**Body:**

```json
{
  "contentType": "article",
  "title": "Как начать тренироваться",
  "body": "Текст...",
  "status": "draft"
}
```

| Поле | Тип | Обязательно | Правила |
|------|-----|-------------|---------|
| `contentType` | string | да | `article`, `guide`, `announcement` |
| `title` | string | да | Мин. 2, макс. 200 |
| `body` | string | да | Мин. 10, макс. 50000 |
| `status` | string | нет | По умолчанию `draft` |

`authorId` — текущий admin из JWT.

**Пример ответа `201`:** `ContentItem`

---

### `GET /api/admin/content/:id`

**Доступ:** admin

**Ошибки:** `404`

---

### `PATCH /api/admin/content/:id`

Редактирование материала.

**Доступ:** admin

**Body:** любые поля из `POST` (все опциональны)

**Пример ответа `200`:** `ContentItem`

---

### `DELETE /api/admin/content/:id`

**Доступ:** admin

**Пример ответа `200`:**

```json
{
  "message": "Content item deleted"
}
```

---

## Content — публичный API

Опубликованные материалы для главной и информационных страниц. Без авторизации.

---

### `GET /api/content`

Список опубликованных материалов (`status = published`).

**Доступ:** гость, user, admin

**Query-параметры:** `page`, `limit`, `contentType`

**Пример ответа `200`:**

```json
{
  "data": [
    {
      "id": 1,
      "contentType": "article",
      "title": "Как начать тренироваться",
      "body": "Краткий или полный текст...",
      "publishedAt": "2026-06-10T12:00:00.000Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 8,
    "totalPages": 1
  }
}
```

В списке `body` может быть урезан (превью) — полный текст в `GET /api/content/:id`.

---

### `GET /api/content/:id`

Один опубликованный материал.

**Доступ:** гость, user, admin

**Ошибки:** `404` — не найден или не `published`

---

## Обзор API

| Блок | Префикс | Основные операции |
|------|---------|-------------------|
| Auth | `/api/auth` | register, login, me, password, activity |
| Exercises | `/api/exercises` | каталог, фильтры, sync (admin) |
| Programs | `/api/programs` | CRUD программ, export |
| Program exercises | `/api/programs/:id/exercises` | add, update, reorder, delete |
| Hints | `/api/programs/:id/hints` | подсказки помощника |
| Body metrics | `/api/body-metrics` | прогресс, chart |
| Calculators | `/api/calculators` | bmi, calories, macros, results |
| Profile | `/api/profile` | summary личного кабинета |
| Admin | `/api/admin` | users, content, stats, exercises sync |
| Content | `/api/content` | публичные материалы |
