# ERD-диаграмма проекта F-easy

F-easy — фитнес-сайт, где пользователь может смотреть каталог упражнений, собирать тренировочные программы, получать подсказки и отслеживать прогресс.

## Mermaid ERD

```mermaid
erDiagram
    USERS {
        INTEGER id PK
        VARCHAR email UK
        VARCHAR password_hash
        VARCHAR name
        VARCHAR role
        BOOLEAN hints_enabled
        TIMESTAMP created_at
    }

    EXERCISES {
        INTEGER id PK
        VARCHAR name
        VARCHAR exercise_type
        VARCHAR muscle_group
        VARCHAR difficulty
        VARCHAR equipment
        TEXT technique
        VARCHAR source
    }

    WORKOUT_PROGRAMS {
        INTEGER id PK
        INTEGER user_id FK
        VARCHAR title
        TEXT description
        VARCHAR goal
        TIMESTAMP created_at
        TIMESTAMP updated_at
    }

    PROGRAM_EXERCISES {
        INTEGER id PK
        INTEGER program_id FK
        INTEGER exercise_id FK
        INTEGER sets_count
        INTEGER reps_count
        INTEGER sort_order
        TEXT note
    }

    TRAINING_HINTS {
        INTEGER id PK
        INTEGER program_id FK
        VARCHAR hint_type
        TEXT text
        VARCHAR importance
        TIMESTAMP created_at
    }

    BODY_METRICS {
        INTEGER id PK
        INTEGER user_id FK
        DATE recorded_at
        NUMERIC weight_kg
        NUMERIC height_cm
        NUMERIC bmi
        NUMERIC body_fat_percent
    }

    CALCULATOR_RESULTS {
        INTEGER id PK
        INTEGER user_id FK
        VARCHAR calculator_type
        JSONB input_data
        JSONB result_data
        TIMESTAMP calculated_at
    }

    CONTENT_ITEMS {
        INTEGER id PK
        INTEGER author_id FK
        VARCHAR content_type
        VARCHAR title
        TEXT body
        VARCHAR status
        TIMESTAMP created_at
        TIMESTAMP updated_at
    }

    USERS ||--o{ WORKOUT_PROGRAMS : "создает программы"
    WORKOUT_PROGRAMS ||--o{ PROGRAM_EXERCISES : "содержит упражнения"
    EXERCISES ||--o{ PROGRAM_EXERCISES : "используется в программах"
    WORKOUT_PROGRAMS ||--o{ TRAINING_HINTS : "получает подсказки"
    USERS ||--o{ BODY_METRICS : "записывает показатели"
    USERS ||--o{ CALCULATOR_RESULTS : "сохраняет расчеты"
    USERS ||--o{ CONTENT_ITEMS : "создает контент"
```

## Описание связей

| Связь | Тип | Описание |
|-------|-----|----------|
| Users -> WorkoutPrograms | 1:N | Один пользователь может создать много тренировочных программ, но каждая программа принадлежит одному пользователю |
| WorkoutPrograms -> ProgramExercises | 1:N | Одна программа может содержать много упражнений с настройками подходов, повторений и порядка |
| Exercises -> ProgramExercises | 1:N | Одно упражнение из каталога может использоваться во многих тренировочных программах |
| WorkoutPrograms -> TrainingHints | 1:N | Для одной программы может быть создано несколько подсказок умного помощника |
| Users -> BodyMetrics | 1:N | Один пользователь может сохранять много записей с параметрами тела по разным датам |
| Users -> CalculatorResults | 1:N | Один пользователь может сохранить много результатов калькуляторов; для гостя user_id может быть NULL |
| Users -> ContentItems | 1:N | Один администратор или автор может создать много материалов |
| WorkoutPrograms <-> Exercises | M:N | Программа может содержать много упражнений, а одно упражнение может входить во многие программы. Эта связь реализована через промежуточную таблицу ProgramExercises |

## Внешние ключи

| Таблица | Поле | Ссылка |
|---------|------|--------|
| WorkoutPrograms | user_id | Users.id |
| ProgramExercises | program_id | WorkoutPrograms.id |
| ProgramExercises | exercise_id | Exercises.id |
| TrainingHints | program_id | WorkoutPrograms.id |
| BodyMetrics | user_id | Users.id |
| CalculatorResults | user_id | Users.id |
| ContentItems | author_id | Users.id |

## Вопросы для самопроверки

1. Чем связь 1:N отличается от M:N? Приведите пример каждой из вашего проекта.

Связь 1:N означает, что одна запись может иметь много связанных записей, но каждая из этих записей относится только к одной основной записи. Например, один User может иметь много WorkoutPrograms. Связь M:N означает, что много записей одной таблицы могут быть связаны со многими записями другой таблицы. Например, WorkoutPrograms и Exercises: одна программа содержит много упражнений, и одно упражнение может использоваться во многих программах.

2. Почему связь M:N нельзя реализовать двумя таблицами? Зачем нужна промежуточная?

Если хранить M:N только в двух таблицах, будет непонятно, где хранить несколько связей между ними. Промежуточная таблица нужна, чтобы каждая связь была отдельной записью. В проекте F-easy таблица ProgramExercises связывает WorkoutPrograms и Exercises, а также хранит sets_count, reps_count, sort_order и note.

3. Что будет, если удалить запись, на которую ссылается FK?

Обычно база данных не даст удалить такую запись, пока на неё ссылаются другие строки. Например, нельзя просто удалить пользователя, если у него есть программы, если не настроено каскадное удаление или предварительное удаление связанных данных.

4. Может ли FK быть NULL? Когда это полезно?

Да, FK может быть NULL, если связь необязательная. Например, CalculatorResults.user_id может быть NULL для результата калькулятора, который получил гость без регистрации.
