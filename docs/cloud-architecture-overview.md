# Cloud Architecture Overview

## System Context

The TODO App is a monorepo containing a React single-page application and a Node.js/Express REST API that shares a single in-memory SQLite store. There is no external database, cloud service, or authentication layer — all state lives in process memory for the lifetime of the API server.

```mermaid
graph LR
    User["User\n(Browser)"]

    subgraph Monorepo["TODO App — Monorepo"]
        Frontend["React Frontend\nReact 18 + MUI\npackages/frontend\nport 3000"]
        Backend["Express API\nNode.js + Express\npackages/backend\nport 3030"]
        Store[("In-Memory Store\nSQLite via better-sqlite3")]
    end

    User -->|"HTTP"| Frontend
    Frontend -->|"HTTP JSON /api/*\nproxied by CRA dev server"| Backend
    Backend -->|"SQL queries"| Store
```

### Component Responsibilities

| Component | Technology | Role |
|-----------|-----------|------|
| **React Frontend** | React 18, MUI v7, CRA | Renders the task list and form; proxies API calls |
| **Express API** | Node.js, Express 4 | Validates input and executes CRUD operations |
| **In-Memory Store** | better-sqlite3 (`:memory:`) | Persists tasks for the lifetime of the API process |

---

## Sequence Diagram — Creating a TODO

The following sequence describes the full flow from the moment a user submits a new task through to the refreshed task list being displayed.

```mermaid
sequenceDiagram
    actor User
    participant TaskForm
    participant App
    participant API as Express API
    participant DB as SQLite Store

    User->>TaskForm: Enter title, description, due date
    User->>TaskForm: Click "Add Task"
    TaskForm->>TaskForm: Validate title is not empty
    TaskForm->>App: onSave({ title, description, due_date })

    App->>API: POST /api/tasks\nContent-Type: application/json
    API->>API: Validate title is present
    API->>DB: INSERT INTO tasks\n(title, description, due_date)
    DB-->>API: New task row with generated id
    API-->>App: 201 Created\n{ id, title, description, due_date, completed, created_at }

    App->>App: Increment refreshKey\n(unmounts and remounts TaskList)
    App->>TaskForm: Reset form fields to empty

    App->>API: GET /api/tasks
    API->>DB: SELECT * FROM tasks\nORDER BY due_date ASC
    DB-->>API: Array of task rows
    API-->>App: 200 OK [ ...tasks ]

    App->>User: Render updated task list
```

### Flow Summary

1. **Input** — The user fills in `TaskForm` and clicks submit.
2. **Client-side validation** — `TaskForm` rejects an empty title before any network call is made.
3. **API call** — `App.handleSave` sends `POST /api/tasks` with the task payload as JSON.
4. **Server-side validation** — The Express route returns HTTP 400 if `title` is missing or blank.
5. **Persistence** — The validated task is inserted into the in-memory SQLite table.
6. **Response** — The API returns the fully-formed task row (including generated `id` and `created_at`) with HTTP 201.
7. **List refresh** — `App` increments `refreshKey`, which remounts `TaskList` and triggers a `GET /api/tasks` to reload the full list.
8. **Render** — The browser displays the newly created task in the list.
