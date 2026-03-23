# Epics and User Stories - TODO App Upgrade

Based on [docs/prd-todo.md](./prd-todo.md).

---

## MVP

---

### Epic 1: Task Priority Field

Enable users to assign a priority level (P1, P2, P3) to each task so urgency is captured alongside the task title.

---

#### Story 1.1: Add `priority` field to the task data model

**As a** developer,
**I want** the tasks database schema to include a `priority` column,
**so that** priority data can be stored and retrieved alongside each task.

**Acceptance Criteria**
- The `tasks` table has a `priority` column that stores one of `P1`, `P2`, or `P3`.
- New tasks that omit `priority` are stored with a default value of `P3`.
- Existing tasks without a priority value are treated as `P3`.

**Technical Requirements**
- In `packages/backend/src/app.js`, extend the `CREATE TABLE` DDL to add `priority TEXT NOT NULL DEFAULT 'P3'`.
- Verify the column is returned in all `SELECT *` responses from `GET /api/tasks` and `GET /api/tasks/:id`.

---

#### Story 1.2: Accept and validate `priority` on task create and update

**As a** developer,
**I want** the API to accept and validate `priority` in `POST /api/tasks` and `PUT /api/tasks/:id`,
**so that** only well-formed values are persisted.

**Acceptance Criteria**
- `POST /api/tasks` and `PUT /api/tasks/:id` accept an optional `priority` field.
- If `priority` is provided, it must be one of `"P1"`, `"P2"`, or `"P3"`; otherwise the API responds with HTTP 400 and a descriptive error message.
- If `priority` is omitted, it defaults to `"P3"`.

**Technical Requirements**
- In `packages/backend/src/app.js`, add a validation check in the `POST` and `PUT` route handlers:
  ```js
  const VALID_PRIORITIES = ['P1', 'P2', 'P3'];
  const priority = req.body.priority ?? 'P3';
  if (!VALID_PRIORITIES.includes(priority)) {
    return res.status(400).json({ error: 'priority must be P1, P2, or P3' });
  }
  ```
- Pass the validated `priority` to the `INSERT` and `UPDATE` SQL statements.

---

#### Story 1.3: Display `priority` badge in the task list

**As a** user,
**I want** to see a color-coded priority badge next to each task,
**so that** I can quickly distinguish high-priority items from lower-priority ones.

**Acceptance Criteria**
- Every task in the list displays a `P1`, `P2`, or `P3` badge.
- P1 badges are red, P2 badges are orange, and P3 badges are gray.
- Tasks without a stored priority are shown with a gray P3 badge.

**Technical Requirements**
- In `packages/frontend/src/TaskList.js`, add a MUI `<Chip>` for `task.priority` inside the `ListItem` render.
- Map priority to color: `{ P1: 'error', P2: 'warning', P3: 'default' }` using the MUI `color` prop.
- Import `Chip` from `@mui/material` (already imported in the file; add `Chip` to the destructured import).

---

#### Story 1.4: Expose `priority` selector in the task form

**As a** user,
**I want** to choose a priority (P1, P2, P3) when creating or editing a task,
**so that** tasks can be submitted with the correct urgency level.

**Acceptance Criteria**
- The task form includes a priority selector with options P1, P2, and P3.
- P3 is selected by default when creating a new task.
- When editing an existing task, the selector is pre-populated with the task's current priority.
- Submitting the form sends `priority` in the request payload.

**Technical Requirements**
- In `packages/frontend/src/TaskForm.js`, add a `priority` state variable initialized to `'P3'`.
- Add a MUI `<Select>` (or `<TextField select>`) with `MenuItem` options for `P1`, `P2`, `P3`.
- Include `priority` in the object passed to `onSave`.
- In the `useEffect` that syncs `initialTask`, also set `priority` from `initialTask?.priority ?? 'P3'`.

---

### Epic 2: Due Date Validation

Ensure that due dates are stored consistently and that invalid values are silently discarded rather than causing errors.

---

#### Story 2.1: Normalize and validate `dueDate` on the backend

**As a** developer,
**I want** the API to ignore invalid `due_date` values rather than storing them,
**so that** the database always contains well-formed dates or `NULL`.

**Acceptance Criteria**
- If `due_date` matches `YYYY-MM-DD` format, it is stored as provided.
- If `due_date` is any other non-empty value, it is treated as absent and stored as `NULL`.
- An absent or `null` `due_date` is stored as `NULL`.

**Technical Requirements**
- In `packages/backend/src/app.js`, add a helper before the `POST` and `PUT` route handlers:
  ```js
  function sanitizeDueDate(value) {
    if (!value) return null;
    return /^\d{4}-\d{2}-\d{2}$/.test(value) ? value : null;
  }
  ```
- Call `sanitizeDueDate(due_date)` before passing the value to the SQL statement.

---

#### Story 2.2: Normalize `dueDate` in the frontend before submission

**As a** user,
**I want** the form to silently discard dates that cannot be parsed,
**so that** I am never blocked by a validation error for an optional field.

**Acceptance Criteria**
- If the due date input contains a valid `YYYY-MM-DD` date, it is sent to the API.
- If the due date input contains an invalid or unparseable value, an empty string is sent (treated as absent).

**Technical Requirements**
- In `packages/frontend/src/TaskForm.js`, the existing `normalizeDateString` helper already attempts to parse dates. Extend `handleSubmit` to pass `null` when `normalizeDateString` returns an empty string.

---

### Epic 3: Task Filtering by Date

Allow users to switch between three views — All, Today, and Overdue — to focus on relevant tasks without manually scanning the full list.

---

#### Story 3.1: Implement `filter` query parameter on `GET /api/tasks`

**As a** developer,
**I want** the tasks API to support a `filter` query parameter with values `all`, `today`, and `overdue`,
**so that** the frontend can request pre-filtered task lists.

**Acceptance Criteria**
- `GET /api/tasks?filter=all` returns all tasks regardless of completion or due date.
- `GET /api/tasks?filter=today` returns only incomplete tasks whose `due_date` equals today's date in `YYYY-MM-DD`.
- `GET /api/tasks?filter=overdue` returns only incomplete tasks whose `due_date` is earlier than today's date.
- The default behavior when `filter` is absent remains unchanged (returns all tasks).

**Technical Requirements**
- In `packages/backend/src/app.js`, extend `buildTaskQuery` (or the `GET /api/tasks` route handler) to handle a `filter` param:
  ```js
  const today = new Date().toISOString().slice(0, 10);
  if (filter === 'today') {
    where.push('completed = 0 AND due_date = @today');
    params.today = today;
  } else if (filter === 'overdue') {
    where.push('completed = 0 AND due_date < @today');
    params.today = today;
  }
  ```
- Existing `completed` and `search` filters must continue to work when `filter` is absent.

---

#### Story 3.2: Add filter tab bar to the task list UI

**As a** user,
**I want** to click All, Today, or Overdue tabs above the task list,
**so that** I can quickly switch to the view that is most relevant to me.

**Acceptance Criteria**
- Three tabs — **All**, **Today**, and **Overdue** — appear above the task list.
- Clicking a tab immediately updates the list to show only matching tasks.
- The active tab is visually highlighted.
- The All tab is selected by default on page load.

**Technical Requirements**
- In `packages/frontend/src/TaskList.js`, add a `activeFilter` state variable initialized to `'all'`.
- Add a MUI `<Tabs>` / `<Tab>` component above the `<List>`.
- Pass the active filter as a query parameter when calling `fetchTasks`:
  ```js
  const response = await fetch(`/api/tasks?filter=${activeFilter}`);
  ```
- Re-fetch tasks whenever `activeFilter` changes (add it to the `useEffect` dependency array).

---

#### Story 3.3: All view shows both complete and incomplete tasks

**As a** user,
**I want** the All view to include tasks regardless of their completion status,
**so that** I can see the full history of tasks in one place.

**Acceptance Criteria**
- Completed and incomplete tasks both appear when the All filter is active.
- Completed tasks continue to render with their dimmed / strikethrough styling.

**Technical Requirements**
- When `filter=all` (or filter absent), the backend SQL must not apply a `completed` restriction.
- No frontend changes are needed beyond passing `filter=all` to the API.

---

#### Story 3.4: Today and Overdue views show only incomplete tasks

**As a** user,
**I want** the Today and Overdue views to exclude completed tasks,
**so that** I focus on actionable work rather than done items.

**Acceptance Criteria**
- The Today tab shows only incomplete tasks with `due_date` equal to today.
- The Overdue tab shows only incomplete tasks with `due_date` before today.
- Completed tasks never appear in either of these two views.

**Technical Requirements**
- Enforced server-side in the `filter` SQL logic described in Story 3.1.
- No additional frontend filtering is required.

---

## Post-MVP

---

### Epic 4: Visual Overdue Highlighting

Make overdue tasks visually distinctive so users can immediately spot which items need urgent attention.

---

#### Story 4.1: Highlight overdue task rows in red

**As a** user,
**I want** overdue tasks to appear with a red background in the list,
**so that** I can identify them at a glance without switching to the Overdue filter.

**Acceptance Criteria**
- A task is considered overdue when its `due_date` is before today's date and it is not yet completed.
- Overdue tasks render with a red-tinted background distinct from normal and completed task backgrounds.
- Completed tasks are never highlighted red, even if their due date is in the past.

**Technical Requirements**
- In `packages/frontend/src/TaskList.js`, compute an `isOverdue` boolean per task:
  ```js
  const today = new Date().toISOString().slice(0, 10);
  const isOverdue = !task.completed && task.due_date && task.due_date < today;
  ```
- Adjust the `ListItem` `sx` `background` and `borderColor` props to use a red tint when `isOverdue` is `true` (e.g., `rgba(211, 47, 47, 0.08)` and `rgba(211, 47, 47, 0.25)`).

---

#### Story 4.2: Show overdue label chip on overdue tasks

**As a** user,
**I want** overdue tasks to display an "Overdue" label,
**so that** the status is explicit even when I am on the All view.

**Acceptance Criteria**
- An "Overdue" chip/badge appears on tasks that are past their due date and not completed.
- The chip is red and positioned near the due date display.
- Non-overdue and completed tasks do not show this chip.

**Technical Requirements**
- In `packages/frontend/src/TaskList.js`, conditionally render a MUI `<Chip label="Overdue" color="error" size="small" />` when `isOverdue` is `true`.

---

### Epic 5: Deterministic Task Sorting

Apply a consistent sort order so the most urgent and highest-priority tasks always appear at the top of the list.

---

#### Story 5.1: Sort overdue tasks before all other tasks

**As a** user,
**I want** overdue tasks to appear at the top of the list,
**so that** the most time-sensitive items are immediately visible.

**Acceptance Criteria**
- Incomplete tasks with a `due_date` before today always appear above tasks that are not overdue.
- Completed tasks follow all incomplete tasks.

**Technical Requirements**
- In `packages/backend/src/app.js`, replace the current `ORDER BY` clause in `GET /api/tasks` with a multi-level sort that uses a CASE expression to detect overdue status:
  ```sql
  ORDER BY
    CASE WHEN completed = 0 AND due_date IS NOT NULL AND due_date < DATE('now') THEN 0 ELSE 1 END ASC,
    ...
  ```

---

#### Story 5.2: Sort non-overdue tasks by priority (P1 → P3)

**As a** user,
**I want** non-overdue tasks ordered from highest (P1) to lowest (P3) priority,
**so that** the most important work is surfaced after overdue items.

**Acceptance Criteria**
- Among non-overdue tasks, P1 tasks come before P2, which come before P3.
- Tasks missing a priority value are treated as P3.

**Technical Requirements**
- Extend the `ORDER BY` clause with a priority rank:
  ```sql
  CASE priority WHEN 'P1' THEN 1 WHEN 'P2' THEN 2 ELSE 3 END ASC,
  ```

---

#### Story 5.3: Sort same-priority tasks by due date ascending

**As a** user,
**I want** tasks with equal priority ordered by their due date (earliest first),
**so that** the most time-constrained task of a given priority appears first.

**Acceptance Criteria**
- When two non-overdue tasks share the same priority, the one with the earlier `due_date` appears first.
- Tasks with no `due_date` appear after tasks that have a due date.

**Technical Requirements**
- Extend the `ORDER BY` clause:
  ```sql
  due_date IS NULL ASC,
  due_date ASC,
  created_at ASC
  ```
- Full composed `ORDER BY` for `GET /api/tasks`:
  ```sql
  ORDER BY
    CASE WHEN completed = 0 AND due_date IS NOT NULL AND due_date < DATE('now') THEN 0 ELSE 1 END ASC,
    CASE priority WHEN 'P1' THEN 1 WHEN 'P2' THEN 2 ELSE 3 END ASC,
    due_date IS NULL ASC,
    due_date ASC,
    created_at ASC
  ```

---

#### Story 5.4: Place tasks without due dates last within their priority group

**As a** user,
**I want** undated tasks to appear after dated tasks at the same priority level,
**so that** tasks with concrete deadlines are always prioritized over open-ended ones.

**Acceptance Criteria**
- A task with no due date sorts after any task of the same priority that has a due date.
- Undated tasks within the same priority are ordered by `created_at` ascending.

**Technical Requirements**
- Enforced by `due_date IS NULL ASC, due_date ASC, created_at ASC` in the `ORDER BY` as defined in Story 5.3 — no separate change required.
