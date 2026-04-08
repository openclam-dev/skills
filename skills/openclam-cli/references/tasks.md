# Task Management Domain Reference

Package: `@openclam/tasks`
Extension name: `tasks`
Dependencies: none

---

## Tables

### `tasks`

| Column           | Type    | Notes                           |
|-----------------|---------|---------------------------------|
| id              | TEXT PK |                                 |
| title           | TEXT    | NOT NULL                        |
| description     | TEXT    | nullable                        |
| project_id      | TEXT    | nullable                        |
| parent_task_id  | TEXT    | nullable, self-ref for subtasks |
| assignee_id     | TEXT    | nullable                        |
| assignee_type   | TEXT    | nullable                        |
| creator_id      | TEXT    | nullable                        |
| creator_type    | TEXT    | nullable                        |
| person_id       | TEXT    | nullable                        |
| org_id          | TEXT    | nullable                        |
| due_date        | TEXT    | nullable, ISO 8601              |
| priority        | TEXT    | NOT NULL (low, medium, high)    |
| status          | TEXT    | NOT NULL (open, done, blocked, cancelled) |
| position        | INTEGER | NOT NULL, default 0             |
| estimated_effort| TEXT    | nullable                        |
| actual_effort   | TEXT    | nullable                        |
| source          | TEXT    | NOT NULL (e.g. "cli")           |
| created_at      | TEXT    | NOT NULL                        |
| updated_at      | TEXT    | NOT NULL                        |
| completed_at    | TEXT    | nullable, set when done         |
| deleted_at      | TEXT    | nullable, soft-delete marker    |

### `task_comments`

| Column     | Type    | Notes                                        |
|-----------|---------|-----------------------------------------------|
| id        | TEXT PK |                                               |
| task_id   | TEXT    | NOT NULL, FK -> tasks.id ON DELETE CASCADE     |
| author_id | TEXT    | NOT NULL                                      |
| parent_id | TEXT    | nullable, self-ref for threading              |
| body      | TEXT    | NOT NULL                                      |
| status    | TEXT    | NOT NULL ("open" or "resolved")               |
| created_at| TEXT    | NOT NULL                                      |
| updated_at| TEXT    | NOT NULL                                      |

### `project_tasks`

| Column     | Type    | Notes                                          |
|-----------|---------|------------------------------------------------|
| id        | TEXT PK |                                                |
| project_id| TEXT    | NOT NULL, FK -> projects.id ON DELETE CASCADE   |
| task_id   | TEXT    | NOT NULL, FK -> tasks.id ON DELETE CASCADE      |
| added_at  | TEXT    | NOT NULL                                       |
| added_by  | TEXT    | nullable                                       |

---

## Service Operations

### taskService (`services/task.ts`)

| Operation | Signature / Notes                                                                               |
|----------|-------------------------------------------------------------------------------------------------|
| add      | `add(db, tables, input, ctx?)` -- required: title, priority, status, source; optional: description, person_id, org_id, due_date, project_id, parent_task_id, assignee_id, assignee_type, creator_id, creator_type, position, estimated_effort, actual_effort |
| list     | `list(db, tables, filters?, pagination?)` -- filters: search (title+description), status, priority, assigneeId, assigneeType, projectId, personId, orgId, parentTaskId (null = root only), overdue; pagination: offset, limit (default 50) |
| get      | `get(db, tables, idOrSlug)` -- looks up by UUID or title match                                   |
| update   | `update(db, tables, id, input, ctx?)` -- partial update of title, description, person_id, org_id, due_date, priority, assignee_id, assignee_type, position, estimated_effort, actual_effort |
| done     | `done(db, tables, id, ctx?)` -- sets status="done", records completed_at; errors if already done |
| reopen   | `reopen(db, tables, id, ctx?)` -- sets status="open", clears completed_at; errors if already open|
| block    | `block(db, tables, id, ctx?)` -- sets status="blocked"                                           |
| cancel   | `cancel(db, tables, id, ctx?)` -- sets status="cancelled"                                        |
| move     | `move(db, tables, id, projectId)` -- reassigns project_id (null to unassign)                     |
| subtasks | `subtasks(db, tables, parentId)` -- lists child tasks ordered by position                        |
| remove   | `remove(db, tables, id, ctx?)` -- soft delete (sets deleted_at)                                  |
| restore  | `restore(db, tables, id, ctx?)` -- clears deleted_at; errors if not deleted                      |
| purge    | `purge(db, tables, id, ctx?)` -- permanent hard delete                                           |

### taskCommentService (`services/task-comment.ts`)

| Operation | Signature / Notes                                                            |
|----------|-------------------------------------------------------------------------------|
| add      | `add(db, tables, input, ctx?)` -- input: task_id, author_id, parent_id?, body; status defaults to "open" |
| list     | `list(db, tables, taskId, filters?)` -- filters: status                       |
| get      | `get(db, tables, id)` -- single comment by ID                                |
| update   | `update(db, tables, id, input, ctx?)` -- input: body?, status?               |
| remove   | `remove(db, tables, id, ctx?)` -- hard delete                                |
| resolve  | `resolve(db, tables, id, ctx?)` -- sets status to "resolved"                 |
| getThread| `getThread(db, tables, parentId)` -- returns child comments                  |

### projectLinkService (`services/project-link.ts`)

| Operation             | Signature / Notes                                             |
|----------------------|----------------------------------------------------------------|
| addTaskToProject     | `addTaskToProject(db, tables, projectId, taskId, ctx?)`        |
| removeTaskFromProject| `removeTaskFromProject(db, tables, projectId, taskId)`          |
| listTasksForProject  | `listTasksForProject(db, tables, projectId)`                    |
| listProjectsForTask  | `listProjectsForTask(db, tables, taskId)`                       |

---

## CLI Commands

All list commands return a paginated response:
```json
{ "data": [...], "nextCursor": "<string | null>", "hasMore": false }
```

> **Important:** `task list` does NOT support `--input-json`. Use individual flags (`--search`, `--status`, `--priority`, etc.) directly. All other `task` commands accept `--input-json`.

### `task`

```
# add (positional title or --input-json)
clam --json tasks task add "Buy server"
clam --json tasks task add --input-json '{"title":"Buy server","priority":"high","due_date":"2026-04-01"}'

# list (paginated; does NOT support --input-json — use flags directly)
clam --json tasks task list
clam --json tasks task list --search "buy" --status open --priority high --limit 25 --cursor <cursor>

# get (positional id or --input-json)
clam --json tasks task get <id>
clam --json tasks task get --input-json '{"id":"<task-id>"}'

# update (positional id or --input-json)
clam --json tasks task update <id> --priority high
clam --json tasks task update --input-json '{"id":"<task-id>","priority":"high"}'

# status transitions (positional id or --input-json)
clam --json tasks task done <id>
clam --json tasks task done --input-json '{"id":"<task-id>"}'
clam --json tasks task reopen <id>
clam --json tasks task reopen --input-json '{"id":"<task-id>"}'
clam --json tasks task block <id>
clam --json tasks task block --input-json '{"id":"<task-id>"}'
clam --json tasks task cancel <id>
clam --json tasks task cancel --input-json '{"id":"<task-id>"}'

# move (positional id + project-id or --input-json)
clam --json tasks task move <task-id> <project-id>
clam --json tasks task move --input-json '{"id":"<task-id>","project_id":"<project-id>"}'

# subtasks (positional id or --input-json)
clam --json tasks task subtasks <id>
clam --json tasks task subtasks --input-json '{"id":"<task-id>"}'

# remove / restore / purge (positional id or --input-json)
clam --json tasks task remove <id>
clam --json tasks task remove --input-json '{"id":"<task-id>"}'
clam --json tasks task restore <id>
clam --json tasks task restore --input-json '{"id":"<task-id>"}'
clam --json tasks task purge <id>
clam --json tasks task purge --input-json '{"id":"<task-id>"}'

# list projects for a task (positional task-id or --input-json)
clam --json tasks task projects <task-id>
clam --json tasks task projects --input-json '{"task_id":"<task-id>"}'
```

### `task comment`

```
# add (positional task-id or --input-json)
clam --json tasks task comment add <task-id> --body "Comment text"
clam --json tasks task comment add --input-json '{"task_id":"<task-id>","body":"Comment text"}'

# list (paginated; positional task-id or --input-json)
clam --json tasks task comment list <task-id>
clam --json tasks task comment list --input-json '{"task_id":"<task-id>","status":"open","cursor":"<cursor>","limit":25}'

# get (positional comment-id or --input-json)
clam --json tasks task comment get <comment-id>
clam --json tasks task comment get --input-json '{"id":"<comment-id>"}'

# resolve (positional comment-id or --input-json)
clam --json tasks task comment resolve <comment-id>
clam --json tasks task comment resolve --input-json '{"id":"<comment-id>"}'

# remove (positional comment-id or --input-json)
clam --json tasks task comment remove <comment-id>
clam --json tasks task comment remove --input-json '{"id":"<comment-id>"}'
```

---

## Global Flags

- `--json` -- Produce compact JSON output (must be placed immediately after `clam`, before the subcommand). Required for machine-readable output.
- `--input-json <json>` -- Pass structured JSON input to any command (except `task list`). Overrides positional args and individual flags.

---

## Input JSON Shapes

Every command (except `task list`) accepts `--input-json <json>`. Field resolution order: `input.<field> ?? positionalArg ?? opts.<flag>`.

### task add

```typescript
interface TaskAddInput {
  title: string;                     // required (also accepted as positional)
  description?: string | null;
  person_id?: string | null;         // UUID
  org_id?: string | null;            // UUID
  due_date?: string | null;          // ISO 8601
  priority?: "low" | "medium" | "high";  // default: "medium"
}
// status always set to "open", source always set to "cli"
```

### task list

```
# Does NOT support --input-json. Use flags directly:
--search <query>       search title + description
--status <status>      open | done | blocked | cancelled
--priority <priority>  low | medium | high
--person-id <id>
--org-id <id>
--assignee-id <id>
--overdue              show only overdue tasks
--cursor <cursor>      pagination cursor
--limit <n>
```
Returns: `{ data: Task[], nextCursor: string | null, hasMore: boolean }`

### task get

```typescript
interface TaskGetInput {
  id: string;                        // required (also accepted as positional)
}
```

### task update

```typescript
interface TaskUpdateInput {
  id: string;                        // required (also accepted as positional)
  title?: string;
  description?: string | null;
  person_id?: string | null;
  org_id?: string | null;
  due_date?: string | null;
  priority?: "low" | "medium" | "high";
}
```

### task done / reopen / block / cancel / remove / restore / purge

```typescript
interface TaskIdInput {
  id: string;                        // required (also accepted as positional)
}
// done returns: Task (with completed_at set)
// reopen returns: Task (completed_at cleared)
// remove returns: { ok: true, deleted: string }
// restore returns: { ok: true, restored: string }
// purge returns: { ok: true, purged: string }
```

### task move

```typescript
interface TaskMoveInput {
  id: string;                        // required (also accepted as first positional)
  project_id: string;                // required (also accepted as second positional)
}
```

### task subtasks

```typescript
interface TaskSubtasksInput {
  id: string;                        // required (also accepted as positional)
}
// Returns: Task[] (ordered by position)
```

### task projects

```typescript
interface TaskProjectsInput {
  task_id: string;                   // required (also accepted as positional)
}
```

### task comment add

```typescript
interface TaskCommentAddInput {
  task_id: string;                   // required (also accepted as positional)
  body: string;                      // required
  author_id?: string;                // UUID; defaults to active user
  parent_id?: string | null;         // UUID; for threading
}
```

### task comment list

```typescript
interface TaskCommentListInput {
  task_id: string;                   // required (also accepted as positional)
  status?: "open" | "resolved";
  cursor?: string;
  limit?: number;
}
// Returns: { data: TaskComment[], nextCursor: string | null, hasMore: boolean }
```

### task comment get / resolve / remove

```typescript
interface TaskCommentIdInput {
  id: string;                        // required (also accepted as positional)
}
```

---

## API Routes

### Tasks

| Method | Path                          | Description                        | Query Params                                                       |
|--------|-------------------------------|------------------------------------|--------------------------------------------------------------------|
| GET    | `/api/tasks`                  | List tasks                         | search, status, priority, assigneeId, projectId, personId, orgId   |
| POST   | `/api/tasks`                  | Create a task                      |                                                                    |
| GET    | `/api/tasks/:id`              | Get task detail                    |                                                                    |
| PUT    | `/api/tasks/:id`              | Update a task                      |                                                                    |
| DELETE | `/api/tasks/:id`              | Soft-delete a task                 |                                                                    |
| PUT    | `/api/tasks/:id/restore`      | Restore a soft-deleted task        |                                                                    |
| DELETE | `/api/tasks/:id/purge`        | Permanently delete a task          |                                                                    |

### Task Status Transitions

| Method | Path                          | Description              | Body                      |
|--------|-------------------------------|--------------------------|---------------------------|
| PUT    | `/api/tasks/:id/done`         | Mark task as done        |                           |
| PUT    | `/api/tasks/:id/reopen`       | Reopen a completed task  |                           |
| PUT    | `/api/tasks/:id/block`        | Mark task as blocked     |                           |
| PUT    | `/api/tasks/:id/cancel`       | Cancel a task            |                           |
| PUT    | `/api/tasks/:id/move`         | Move task to project     | `{ projectId: "..." }`   |

### Task Subtasks

| Method | Path                          | Description              |
|--------|-------------------------------|--------------------------|
| GET    | `/api/tasks/:id/subtasks`     | List subtasks of a task  |

### Task Comments

| Method | Path                              | Description                    | Query Params |
|--------|-----------------------------------|--------------------------------|--------------|
| GET    | `/api/tasks/:id/comments`         | List comments on a task        | status       |
| POST   | `/api/tasks/:id/comments`         | Add comment to a task          |              |
| GET    | `/api/task-comments/:id`          | Get single comment             |              |
| PUT    | `/api/task-comments/:id`          | Update comment (body, status)  |              |
| DELETE | `/api/task-comments/:id`          | Delete comment                 |              |
| PUT    | `/api/task-comments/:id/resolve`  | Resolve a comment              |              |

### Project-Task Links

| Method | Path                                    | Description                  | Body                  |
|--------|-----------------------------------------|------------------------------|-----------------------|
| POST   | `/api/projects/:id/tasks`               | Link task to project         | `{ taskId: "..." }`   |
| DELETE | `/api/projects/:id/tasks/:taskId`       | Unlink task from project     |                       |
| GET    | `/api/projects/:id/tasks`               | List tasks for project       |                       |
| GET    | `/api/tasks/:id/projects`               | List projects for task       |                       |

---

## Events Emitted

| Event Type             | Entity Type    |
|------------------------|----------------|
| task.created           | task           |
| task.updated           | task           |
| task.completed         | task           |
| task.reopened          | task           |
| task.blocked           | task           |
| task.cancelled         | task           |
| task.deleted           | task           |
| task.restored          | task           |
| task.purged            | task           |
| task.comment.created   | task_comment   |
| task.comment.updated   | task_comment   |
| task.comment.deleted   | task_comment   |
| task.comment.resolved  | task_comment   |

## Permissions

| Resource      | Actions                               |
|--------------|---------------------------------------|
| task         | create, read, update, delete, complete|
| task_comment | create, read, update, delete          |
