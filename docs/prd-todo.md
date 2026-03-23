# Product Requirements Document (PRD) - TODO App Upgrade (Due Dates, Priority, and Filters)

## 1. Overview

The TODO app currently supports only a task title and completion status. This PRD defines a practical upgrade that improves task planning while keeping implementation simple and teachable.

The MVP introduces due dates, task priorities, and basic date-based filtering, while preserving local-only storage with no backend work. Post-MVP enhancements add richer visual and sorting behavior. Complex features such as notifications and recurring tasks remain out of scope.

---

## 2. MVP Scope

- Add an optional `dueDate` field to each task using ISO format `YYYY-MM-DD`.
- Add a `priority` field with enum values `P1 | P2 | P3`.
- Set default `priority` to `P3` when not specified.
- Support filters: `All`, `Today`, and `Overdue`.
- Keep existing storage local-only (no backend or external storage changes).
- Require `title` for task creation.
- Validate `priority` as one of `P1`, `P2`, or `P3`.
- Treat invalid `dueDate` values as absent (ignore invalid values).
- Show both complete and incomplete tasks in `All` view.
- Show only incomplete tasks in `Today` view.
- Show only incomplete tasks in `Overdue` view.

---

## 3. Post-MVP Scope

- Visually highlight overdue tasks so they are easy to identify.
- Apply deterministic sorting with overdue tasks first.
- Within non-overdue ordering, sort by priority from `P1` to `P3`.
- Within same priority, sort by due date ascending.
- Place tasks without due dates after tasks with due dates.

---

## 4. Out of Scope

- Notifications and reminder delivery.
- Recurring tasks.
- Multi-user or collaboration features.
- Keyboard navigation and expanded accessibility feature work.
- External or cloud-backed storage.
