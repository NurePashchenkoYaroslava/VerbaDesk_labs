# API Контракт системи VerbaDesk (API Specification)

Цей документ описує REST API для системи VerbaDesk відповідно до формалізованих вимог (FR-01 — FR-07) та бізнес-правил (D-01 — D-05). Усі запити та відповіді використовують формат `application/json`.
Усі ендпоінти вимагають наявності HTTP-заголовка X-User-Id. Значення 1 або 2 ідентифікує користувача як Translator, значення 3 — як Manager. Якщо заголовок відсутній або містить невідомий ID, система повертає 401 Unauthorized.

---

## 1. Технічні та допоміжні ендпоінти

### 1.1 Перевірка стану системи

- **Метод і шлях:** `GET /health`
- **Призначення:** Технічна перевірка доступності API.
- **Успішна відповідь (200 OK):**
  ```json
  {
    "status": "ok",
    "timestamp": "2026-10-18T12:00:00Z"
  }
  ```
  - **Помилки:**
    - `401 Unauthorized` (Відсутній або недійсний заголовок X-User-Id):
    ```json
    { "detail": "Unauthorized: X-User-Id header missing or invalid" }
    ```

### 1.2 Отримання списку користувачів

- **Метод і шлях:** `GET /users`
- **Призначення:** Повернути статичний список 3 тестових користувачів.
- **Успішна відповідь (200 OK):**
  ```json
  [
    { "id": 1, "name": "Agent A", "role": "Translator" },
    { "id": 2, "name": "Agent B", "role": "Translator" },
    { "id": 3, "name": "Team Lead", "role": "Manager" }
  ]
  ```
- **Помилки:**
    - `401 Unauthorized` (Відсутній або недійсний заголовок X-User-Id):
    ```json
    { "detail": "Unauthorized: X-User-Id header missing or invalid" }
    ```

---

## 2. Робота із заявками (Translation Requests)

### 2.1 Створення нової заявки

- **Метод і шлях:** `POST /issues`
- **Призначення:** Створити нову заявку на переклад (FR-01, US-01).
- **Тіло запиту:**
  ```json
  {
    "title": "Переклад контракту",
    "description": "Потрібен переклад на англійську до п'ятниці. Посилання: http://..."
  }
  ```
- **Успішна відповідь (201 Created):**
  ```json
  {
    "id": 101,
    "title": "Переклад контракту",
    "description": "Потрібен переклад на англійську до п'ятниці. Посилання: http://...",
    "status": "OPEN",
    "priority": null,
    "assignee_id": null,
    "author_id": 1,
    "created_at": "2026-10-18T12:05:00Z"
  }
  ```
- **Помилки:**
  - `422 Unprocessable Entity` (Відсутній `title` або `description`, або некоректна довжина).
  ```json
  {
    "detail": [
      {
        "loc": ["body", "description"],
        "msg": "Field required",
        "type": "missing"
      }
    ]
  }
  ```

### 2.2 Отримання списку заявок (з фільтрацією)

- **Метод і шлях:** `GET /issues`
- **Призначення:** Повернути список заявок (FR-02, US-02). Підтримує фільтрацію (AND).
- **Query-параметри:**
  - `status` (опціонально, напр. `OPEN`)
  - `priority` (опціонально, напр. `High`, або `null` для Not Set)
- **Успішна відповідь (200 OK):** Масив об'єктів заявок (як у 2.1). Якщо збігів немає — повертається порожній масив `[]`.
- **Помилки:**
  - `422 Unprocessable Entity` (Передано невідомий статус або пріоритет у Query-параметрах):
    ```json
    {
      "detail": [
        {
          "loc": ["query", "status"],
          "msg": "Input should be 'OPEN', 'IN_PROGRESS', 'RESOLVED' or 'CLOSED'",
          "type": "enum"
        }
      ]
    }
    ```

### 2.3 Отримання однієї заявки

- **Метод і шлях:** `GET /issues/{id}`
- **Призначення:** Повернути деталі конкретної заявки.
- **Успішна відповідь (200 OK):** Об'єкт заявки.
- **Помилки:**
  - `404 Not Found` (Заявку не знайдено):
    ```json
    { "detail": "Translation request with id 999 not found" }
    ```

### 2.4 Часткове оновлення даних заявки (PATCH)

- **Метод і шлях:** `PATCH /issues/{id}`
- **Призначення:** Оновлення основних текстових полів, пріоритету та призначення (FR-03, FR-05, US-03, US-05). Змінюються ТІЛЬКИ передані поля (атомарно).
- **Особливості поведінки:**
  - Якщо передано порожнє тіло `{}`, система не вносить жодних змін і повертає поточну заявку з кодом `200 OK`.
  - Спроба передати системні поля (`id`, `created_at`) у тілі PATCH запиту ігнорується системою.
- **Тіло запиту (приклад оновлення всіх дозволених полів):**
  ```json
  {
    "title": "Виправлений переклад контракту",
    "priority": "High",
    "assignee_id": 2
  }
  ```
- **Успішна відповідь (200 OK):** Об'єкт оновленої заявки.
- **Помилки:**
  - `404 Not Found` (Заявку не знайдено):
    ```json
    { "detail": "Translation request with id 999 not found" }
    ```
  - `409 Conflict` (Спроба редагувати заявку у статусі CLOSED):
    ```json
    { "detail": "Cannot update a request in CLOSED status" }
    ```
  - `403 Forbidden` (Спроба змінити priority/assignee_id користувачем без ролі Manager):
    ```json
    { "detail": "Only Manager can change priority or assignee" }
    ```
  - `422 Unprocessable Entity` (Передано невідомий assignee_id):
    ```json
    {
      "detail": [
        {
          "loc": ["body", "assignee_id"],
          "msg": "Translator with id 99 does not exist",
          "type": "value_error"
        }
      ]
    }
    ```

### 2.5 Окрема операція зміни статусу

- **Метод і шлях:** `PATCH /issues/{id}/status`
- **Призначення:** Переведення заявки в інший статус суворо за матрицею переходів (FR-04, US-04).
- **Тіло запиту:**
  ```json
  {
    "status": "IN_PROGRESS"
  }
  ```
- **Успішна відповідь (200 OK):**
  Об'єкт оновленої заявки. (Якщо передано поточний статус, відповідь також `200 OK` без фактичних змін у базі).
- **Помилки:**
  - `404 Not Found` (Заявку не знайдено):
    ```json
    { "detail": "Translation request with id 999 not found" }
    ```
  - `409 Conflict` (Заборонений перехід статусу АБО діє не поточний Assignee):
    ```json
    { "detail": "Invalid status transition from OPEN to RESOLVED" }
    ```
  - `422 Unprocessable Entity` (Передано невідомий статус або null):
    ```json
    {
      "detail": [
        {
          "loc": ["body", "status"],
          "msg": "Input should be 'OPEN', 'IN_PROGRESS', 'RESOLVED' or 'CLOSED'",
          "type": "enum"
        }
      ]
    }
    ```

---

## 3. Коментарі (Comments)

### 3.1 Отримання коментарів до заявки

- **Метод і шлях:** `GET /issues/{id}/comments`
- **Призначення:** Отримати список усіх коментарів, пов'язаних із заявкою (FR-06, US-06).
- **Успішна відповідь (200 OK):**
  ```json
  [
    {
      "id": 1,
      "text": "Почав роботу, терміни встигаємо.",
      "request_id": 101,
      "author_id": 2,
      "created_at": "2026-10-18T13:00:00Z"
    }
  ]
  ```
- **Помилки:**
  - `404 Not Found` (Якщо самої заявки не існує):
    ```json
    { "detail": "Translation request with id 999 not found" }
    ```

### 3.2 Додавання коментаря

- **Метод і шлях:** `POST /issues/{id}/comments`
- **Призначення:** Додати новий коментар до заявки (включно із заявками `CLOSED`) (FR-06, US-06).
- **Тіло запиту:**
  ```json
  {
    "text": "Клієнт просив звернути увагу на терміни в пункті 4."
  }
  ```
- **Успішна відповідь (201 Created):** Об'єкт новоствореного коментаря (як у 3.1).
- **Помилки:**
  - `404 Not Found` (Заявки не існує):
    ```json
    { "detail": "Translation request with id 999 not found" }
    ```
  - `422 Unprocessable Entity` (Текст коментаря порожній або містить лише пробіли):
    ```json
    {
      "detail": [
        {
          "loc": ["body", "text"],
          "msg": "String should have at least 1 character",
          "type": "string_too_short"
        }
      ]
    }
    ```
