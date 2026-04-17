---
name: operate-tasks
description: >
  Task management — tasks with lifecycle states (open, done, blocked,
  cancelled), subtasks, comments, project links, and assignees. Covers the
  `clam tasks` CLI, service operations, and database schema.
metadata:
  author: openclam
  version: 0.1.0
---

# Tasks

Package: `@openclam/tasks` (extension: `tasks`)

---

## Tables

### tasks
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| title | TEXT NOT NULL | |
| description | TEXT | |
| project_id | TEXT | |
| parent_task_id | TEXT | Self-ref for subtasks |
| assignee_id | TEXT | |
| assignee_type | TEXT | e.g. `human`, `agent` |
| creator_id | TEXT | |
| creator_type | TEXT | |
| person_id | TEXT | FK to contacts persons |
| org_id | TEXT | FK to contacts orgs |
| due_date | TEXT | ISO 8601 |
| priority | TEXT NOT NULL | `low`, `medium`, `high` |
| status | TEXT NOT NULL | `open`, `done`, `blocked`, `cancelled` |
| position | INTEGER NOT NULL | default 0, used for ordering within parent |
| estimated_effort | TEXT | |
| actual_effort | TEXT | |
| source | TEXT NOT NULL | e.g. `cli`, `api`, `web` |
| updated_by | TEXT | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| completed_at | TEXT | Set when status transitions to `done` |
| deleted_at | TEXT | Soft delete timestamp |

### task_comments
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| task_id | TEXT NOT NULL | FK to tasks.id ON DELETE CASCADE |
| author_id | TEXT NOT NULL | |
| parent_id | TEXT | Self-ref for threading |
| body | TEXT NOT NULL | |
| status | TEXT NOT NULL | `open` or `resolved` |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

### project_tasks
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| project_id | TEXT NOT NULL | |
| task_id | TEXT NOT NULL | FK to tasks.id ON DELETE CASCADE |
| added_at | TEXT NOT NULL | |
| added_by | TEXT | |

---

## Service Operations

### taskService (`services/task.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Required: `title`, `priority`, `status`, `source`; optional: `description`, `person_id`, `org_id`, `due_date`, `project_id`, `parent_task_id`, `assignee_id`/`assignee_type`, `creator_id`/`creator_type`, `position`, `estimated_effort`, `actual_effort` |
| `count(db, tables, filters?)` | |
| `list(db, tables, filters?, pagination?)` | filters: `search` (title+description), `status`, `priority`, `assigneeId`, `assigneeType`, `projectId`, `personId`, `orgId`, `parentTaskId` (`null` = root only), `overdue` |
| `get(db, tables, idOrSlug)` | Looks up by UUID or title match |
| `update(db, tables, id, input, ctx?)` | Partial update of `title`, `description`, `person_id`, `org_id`, `due_date`, `priority`, `assignee_id`/`assignee_type`, `position`, `estimated_effort`, `actual_effort` |
| `done(db, tables, id, ctx?)` | Sets status=`done`, records `completed_at`; errors if already done |
| `reopen(db, tables, id, ctx?)` | Sets status=`open`, clears `completed_at`; errors if already open |
| `block(db, tables, id, ctx?)` | Sets status=`blocked` |
| `cancel(db, tables, id, ctx?)` | Sets status=`cancelled` |
| `move(db, tables, id, projectId)` | Reassigns `project_id` (`null` to unassign) |
| `subtasks(db, tables, parentId)` | Child tasks ordered by `position` |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore soft-deleted |
| `purge(db, tables, id, ctx?)` | Hard delete |
| `importFromFile(db, tables, filePath, format?)` / `importCsv(...)` | Bulk import |
| `exportData(db, tables)` / `exportToFile(db, tables, filePath?, format?)` | Bulk export |

### taskCommentService (`services/task-comment.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Input: `task_id`, `author_id`, `parent_id?`, `body`; status defaults to `open` |
| `list(db, tables, taskId, filters?)` | Filter by `status` |
| `get(db, tables, id)` | Single comment |
| `update(db, tables, id, input, ctx?)` | Update `body`, `status` |
| `remove(db, tables, id, ctx?)` | Hard delete |
| `resolve(db, tables, id, ctx?)` | Sets status to `resolved` |
| `getThread(db, tables, parentId)` | Child comments |

### projectLinkService (`services/project-link.ts`)
`addTaskToProject`, `removeTaskFromProject`, `listTasksForProject`, `listProjectsForTask`.

---

## CLI Commands

All list commands return `{ data: [...], nextCursor: string | null, hasMore: boolean }`.

### task

```
clam --json tasks task add --input-json '{"title":"Buy server","priority":"high","due_date":"2026-04-01","description":"...","person_id":"...","org_id":"...","project_id":"..."}'
clam --json tasks task count --input-json '{"search":"...","status":"open","priority":"high"}'
clam --json tasks task list --input-json '{"search":"buy","status":"open","priority":"high","person_id":"...","org_id":"...","assignee_id":"...","overdue":true,"cursor":"...","limit":25}'
clam --json tasks task get --input-json '{"id":"<task-id>"}'
clam --json tasks task update --input-json '{"id":"<task-id>","priority":"high","description":"...","due_date":"..."}'
clam --json tasks task done --input-json '{"id":"<task-id>"}'
clam --json tasks task reopen --input-json '{"id":"<task-id>"}'
clam --json tasks task block --input-json '{"id":"<task-id>"}'
clam --json tasks task cancel --input-json '{"id":"<task-id>"}'
clam --json tasks task move --input-json '{"id":"<task-id>","project_id":"<project-id>"}'
clam --json tasks task subtasks --input-json '{"id":"<task-id>"}'
clam --json tasks task remove --input-json '{"id":"<task-id>"}'
clam --json tasks task restore --input-json '{"id":"<task-id>"}'
clam --json tasks task purge --input-json '{"id":"<task-id>"}'
clam --json tasks task projects --input-json '{"task_id":"<task-id>"}'
clam --json tasks task import --input-json '{"file":"./tasks.csv","format":"csv"}'
clam --json tasks task export --input-json '{"file":"./tasks.csv","format":"csv"}'
```

### comment

Note: comments are registered as a top-level `clam tasks comment` command (not nested under `task`).

```
clam --json tasks comment add --input-json '{"task_id":"<task-id>","body":"...","author_id":"...","parent_id":"..."}'
clam --json tasks comment list --input-json '{"task_id":"<task-id>","status":"open","cursor":"...","limit":25}'
clam --json tasks comment get --input-json '{"id":"<comment-id>"}'
clam --json tasks comment resolve --input-json '{"id":"<comment-id>"}'
clam --json tasks comment remove --input-json '{"id":"<comment-id>"}'
```

---

## Input JSON Shapes

#### task add
```typescript
interface TaskAddInput {
  title: string;
  description?: string | null;
  person_id?: string | null;
  org_id?: string | null;
  project_id?: string | null;
  parent_task_id?: string | null;
  assignee_id?: string | null;
  assignee_type?: string | null;
  due_date?: string | null;            // ISO 8601
  priority?: "low" | "medium" | "high"; // default: "medium"
  position?: number;                    // default: 0
  estimated_effort?: string | null;
  actual_effort?: string | null;
}
// status is always set to "open", source is always set by the command (e.g. "cli")
```

#### task list
```typescript
interface TaskListInput {
  search?: string;                     // title + description
  status?: "open" | "done" | "blocked" | "cancelled";
  priority?: "low" | "medium" | "high";
  person_id?: string;
  org_id?: string;
  project_id?: string;
  assignee_id?: string;
  assignee_type?: string;
  parent_task_id?: string | null;      // null = root-only
  overdue?: boolean;
  cursor?: string;
  limit?: number;
}
```

#### task update
```typescript
interface TaskUpdateInput {
  id: string;
  title?: string;
  description?: string | null;
  person_id?: string | null;
  org_id?: string | null;
  due_date?: string | null;
  priority?: "low" | "medium" | "high";
  assignee_id?: string | null;
  assignee_type?: string | null;
  position?: number;
  estimated_effort?: string | null;
  actual_effort?: string | null;
}
```

#### task done / reopen / block / cancel / remove / restore / purge
```typescript
interface TaskIdInput { id: string; }
// done returns Task with completed_at set
// reopen returns Task with completed_at cleared
// remove returns { ok: true, deleted: string }
// restore returns { ok: true, restored: string }
// purge returns { ok: true, purged: string }
```

#### task move
```typescript
interface TaskMoveInput {
  id: string;
  project_id: string | null;           // null unassigns the task from its project
}
```

#### task subtasks / projects
```typescript
interface TaskSubtasksInput { id: string; }        // returns Task[] ordered by position
interface TaskProjectsInput { task_id: string; }
```

#### comment add
```typescript
interface TaskCommentAddInput {
  task_id: string;
  body: string;
  author_id?: string;                  // defaults to active user
  parent_id?: string | null;
}
```

#### comment list
```typescript
interface TaskCommentListInput {
  task_id: string;
  status?: "open" | "resolved";
  cursor?: string;
  limit?: number;
}
```

---

See `../surfaces.md` to translate these operations to MCP, HTTP, or the typed client.
