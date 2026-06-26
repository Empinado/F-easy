# Архитектура и структура проекта F-easy

Монорепозиторий: фронтенд и бэкенд в одном git-репозитории, общая документация в `docs/`.

```
F-easy/
├── client/                 # React + TypeScript (FSD)
├── server/                 # Node.js + Express (трёхслойная архитектура)
├── docs/                   # API, архитектура
├── erd.md
├── README.md
├── .gitignore
└── docker-compose.yml      # позже — PostgreSQL для локальной разработки
```

---

## Стек

| Слой | Технологии |
|------|------------|
| Frontend | React, TypeScript, Vite, React Router, Zustand (или Context для auth) |
| UI | CSS modules или TailwindCSS |
| Backend | Node.js, Express, TypeScript |
| БД | PostgreSQL |
| ORM / запросы | Prisma (или `pg` + SQL в repository) |
| Валидация | Zod (бэкенд и опционально фронт) |
| Auth | JWT (access + refresh), bcrypt |
| Внешний API | WorkoutX (упражнения) |

Фронт общается только с нашим API (`/api`). Ключ WorkoutX — только на бэкенде.

---

## Backend — трёхслойная архитектура

Поток запроса:

```
HTTP запрос
    → router (маршрут + middleware)
    → controller (парсинг req, status code, ответ)
    → service (бизнес-логика, права, подсказки, расчёты)
    → repository (SQL / Prisma)
    → PostgreSQL
```

**Правила:**
- Router/controller не содержит бизнес-логику и SQL.
- Service не знает про `req`/`res` — только данные и ошибки.
- Repository не знает про HTTP — только CRUD и запросы к БД.

### Структура `server/`

```
server/
├── src/
│   ├── index.ts                 # запуск сервера
│   ├── app.ts                   # Express app, middleware, роуты
│   ├── config/
│   │   ├── env.ts               # переменные из .env
│   │   └── constants.ts         # лимиты подсказок, пагинация
│   ├── db/
│   │   ├── client.ts            # подключение Prisma / pg pool
│   │   └── migrations/          # или prisma/migrations на корне server
│   ├── middlewares/
│   │   ├── auth.middleware.ts   # проверка JWT, req.user
│   │   ├── admin.middleware.ts  # role === admin
│   │   ├── validate.middleware.ts
│   │   └── error.middleware.ts  # единый формат ошибок API
│   ├── routes/
│   │   ├── index.ts             # сборка всех роутов под /api
│   │   ├── auth.routes.ts
│   │   ├── exercises.routes.ts
│   │   ├── programs.routes.ts
│   │   ├── body-metrics.routes.ts
│   │   ├── calculators.routes.ts
│   │   ├── profile.routes.ts
│   │   ├── content.routes.ts
│   │   └── admin.routes.ts
│   ├── controllers/
│   │   ├── auth.controller.ts
│   │   ├── exercises.controller.ts
│   │   ├── programs.controller.ts
│   │   ├── hints.controller.ts
│   │   ├── body-metrics.controller.ts
│   │   ├── calculators.controller.ts
│   │   ├── profile.controller.ts
│   │   ├── content.controller.ts
│   │   └── admin.controller.ts
│   ├── services/
│   │   ├── auth.service.ts
│   │   ├── exercise.service.ts
│   │   ├── workoutx.service.ts      # запросы к WorkoutX API
│   │   ├── program.service.ts
│   │   ├── program-exercise.service.ts
│   │   ├── hint.service.ts          # правила умного помощника
│   │   ├── body-metrics.service.ts
│   │   ├── calculator.service.ts    # ИМТ, калории, БЖУ
│   │   ├── profile.service.ts
│   │   ├── content.service.ts
│   │   ├── admin.service.ts
│   │   └── activity.service.ts      # user_activities
│   ├── repositories/
│   │   ├── user.repository.ts
│   │   ├── exercise.repository.ts
│   │   ├── program.repository.ts
│   │   ├── program-exercise.repository.ts
│   │   ├── hint.repository.ts
│   │   ├── body-metrics.repository.ts
│   │   ├── calculator.repository.ts
│   │   ├── content.repository.ts
│   │   └── activity.repository.ts
│   ├── validators/
│   │   ├── auth.schema.ts
│   │   ├── program.schema.ts
│   │   ├── body-metrics.schema.ts
│   │   └── ...
│   ├── types/
│   │   ├── express.d.ts             # расширение Request (user)
│   │   └── api.types.ts
│   └── utils/
│       ├── jwt.ts
│       ├── password.ts
│       └── pagination.ts
├── prisma/
│   └── schema.prisma                # если Prisma
├── .env.example
├── package.json
└── tsconfig.json
```

### Соответствие routes ↔ API-контракт

| routes | Префикс |
|--------|---------|
| `auth.routes` | `/api/auth` |
| `exercises.routes` | `/api/exercises` |
| `programs.routes` | `/api/programs` (+ exercises, hints, export) |
| `body-metrics.routes` | `/api/body-metrics` |
| `calculators.routes` | `/api/calculators` |
| `profile.routes` | `/api/profile` |
| `content.routes` | `/api/content` |
| `admin.routes` | `/api/admin` |

---

## Frontend — Feature-Sliced Design (FSD)

Слои (сверху вниз по зависимостям):

```
app → pages → widgets → features → entities → shared
```

Нижний слой **не импортирует** верхний. `shared` не знает про `features`. `entities` не знает про `pages`.

### Структура `client/`

```
client/
├── public/
├── src/
│   ├── app/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── providers/
│   │   │   ├── AuthProvider.tsx
│   │   │   └── RouterProvider.tsx
│   │   ├── routes/
│   │   │   ├── index.tsx            # конфиг React Router
│   │   │   └── ProtectedRoute.tsx
│   │   └── styles/
│   │       └── global.css
│   │
│   ├── pages/
│   │   ├── home/
│   │   │   └── ui/HomePage.tsx
│   │   ├── exercises/
│   │   │   ├── ui/ExercisesPage.tsx
│   │   │   └── ui/ExerciseDetailPage.tsx
│   │   ├── programs/
│   │   │   ├── ui/ProgramsPage.tsx
│   │   │   └── ui/ProgramBuilderPage.tsx
│   │   ├── profile/
│   │   │   └── ui/ProfilePage.tsx
│   │   ├── calculators/
│   │   │   └── ui/CalculatorsPage.tsx
│   │   ├── auth/
│   │   │   ├── ui/LoginPage.tsx
│   │   │   └── ui/RegisterPage.tsx
│   │   ├── admin/
│   │   │   ├── ui/AdminDashboardPage.tsx
│   │   │   ├── ui/AdminUsersPage.tsx
│   │   │   └── ui/AdminContentPage.tsx
│   │   └── not-found/
│   │       └── ui/NotFoundPage.tsx
│   │
│   ├── widgets/
│   │   ├── header/
│   │   │   └── ui/Header.tsx
│   │   ├── footer/
│   │   │   └── ui/Footer.tsx
│   │   ├── exercise-catalog/
│   │   │   └── ui/ExerciseCatalog.tsx
│   │   ├── program-builder/
│   │   │   └── ui/ProgramBuilder.tsx
│   │   ├── hints-panel/
│   │   │   └── ui/HintsPanel.tsx
│   │   ├── body-metrics-chart/
│   │   │   └── ui/BodyMetricsChart.tsx
│   │   └── profile-summary/
│   │       └── ui/ProfileSummary.tsx
│   │
│   ├── features/
│   │   ├── auth/
│   │   │   ├── login/
│   │   │   │   ├── ui/LoginForm.tsx
│   │   │   │   └── model/useLogin.ts
│   │   │   ├── register/
│   │   │   │   ├── ui/RegisterForm.tsx
│   │   │   │   └── model/useRegister.ts
│   │   │   └── logout/
│   │   │       └── ui/LogoutButton.tsx
│   │   ├── create-program/
│   │   │   ├── ui/CreateProgramForm.tsx
│   │   │   └── model/useCreateProgram.ts
│   │   ├── add-exercise-to-program/
│   │   │   └── ui/AddExerciseButton.tsx
│   │   ├── edit-program-exercise/
│   │   │   └── ui/ProgramExerciseEditor.tsx
│   │   ├── export-program/
│   │   │   └── ui/ExportProgramButton.tsx
│   │   ├── toggle-hints/
│   │   │   └── ui/HintsToggle.tsx
│   │   ├── save-body-metric/
│   │   │   └── ui/BodyMetricForm.tsx
│   │   ├── calculator-bmi/
│   │   │   └── ui/BmiCalculator.tsx
│   │   ├── calculator-calories/
│   │   │   └── ui/CaloriesCalculator.tsx
│   │   ├── calculator-macros/
│   │   │   └── ui/MacrosCalculator.tsx
│   │   └── admin-manage-content/
│   │       └── ui/ContentEditor.tsx
│   │
│   ├── entities/
│   │   ├── user/
│   │   │   ├── model/types.ts
│   │   │   └── api/userApi.ts
│   │   ├── exercise/
│   │   │   ├── model/types.ts
│   │   │   ├── ui/ExerciseCard.tsx
│   │   │   └── api/exerciseApi.ts
│   │   ├── workout-program/
│   │   │   ├── model/types.ts
│   │   │   ├── ui/ProgramCard.tsx
│   │   │   └── api/programApi.ts
│   │   ├── training-hint/
│   │   │   ├── model/types.ts
│   │   │   └── ui/HintBadge.tsx
│   │   ├── body-metric/
│   │   │   ├── model/types.ts
│   │   │   └── api/bodyMetricApi.ts
│   │   └── content-item/
│   │       ├── model/types.ts
│   │       └── api/contentApi.ts
│   │
│   └── shared/
│       ├── api/
│       │   ├── baseApi.ts           # axios/fetch + interceptors (token)
│       │   └── types.ts             # ApiError, PaginatedResponse
│       ├── ui/
│       │   ├── Button/
│       │   ├── Input/
│       │   ├── Modal/
│       │   ├── Loader/
│       │   └── Pagination/
│       ├── lib/
│       │   ├── formatDate.ts
│       │   └── cn.ts
│       └── config/
│           └── env.ts               # VITE_API_URL
│
├── index.html
├── package.json
├── tsconfig.json
└── vite.config.ts
```

### Соответствие pages ↔ экраны из README

| Page | URL (пример) |
|------|----------------|
| HomePage | `/` |
| ExercisesPage | `/exercises` |
| ExerciseDetailPage | `/exercises/:id` |
| ProgramsPage | `/programs` |
| ProgramBuilderPage | `/programs/:id` |
| ProfilePage | `/profile` |
| CalculatorsPage | `/calculators` |
| LoginPage | `/login` |
| RegisterPage | `/register` |
| AdminDashboardPage | `/admin` |
| AdminUsersPage | `/admin/users` |
| AdminContentPage | `/admin/content` |

### Layout

Страницы с `Header` + `Footer` оборачиваются в layout в `app/routes`. Админка — отдельный layout с сайдбаром (widget `admin-sidebar`).

---

## Переменные окружения

### `server/.env`

```
PORT=3001
DATABASE_URL=postgresql://user:pass@localhost:5432/feasy
JWT_ACCESS_SECRET=
JWT_REFRESH_SECRET=
JWT_ACCESS_EXPIRES=15m
JWT_REFRESH_EXPIRES=7d
WORKOUTX_API_KEY=
```

### `client/.env`

```
VITE_API_URL=http://localhost:3001/api
```

---

## Порядок реализации (после каркаса)

1. БД: Prisma schema по ERD, миграции.
2. Auth: register, login, middleware.
3. Exercises: sync WorkoutX, GET каталога.
4. Programs + program exercises.
5. Hints service.
6. Body metrics + calculators + profile summary.
7. Admin + content.
8. Фронт: auth → каталог → конструктор → профиль → админка.
