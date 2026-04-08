# Marketing Domain Reference

Package: `@openclam/marketing`
Extension name: `marketing`
Dependencies: `contacts`, `messaging`

---

## Tables

### sequences

| Column      | Type | Constraints            |
|-------------|------|------------------------|
| id          | TEXT | PRIMARY KEY            |
| name        | TEXT | NOT NULL               |
| description | TEXT | nullable               |
| status      | TEXT | NOT NULL               |
| created_at  | TEXT | NOT NULL               |
| updated_at  | TEXT | NOT NULL               |
| deleted_at  | TEXT | nullable (soft delete)  |

### sequence_steps

| Column      | Type    | Constraints                                  |
|-------------|---------|----------------------------------------------|
| id          | TEXT    | PRIMARY KEY                                  |
| sequence_id | TEXT    | NOT NULL, FK -> sequences(id)                |
| step_order  | INTEGER | NOT NULL                                     |
| channel     | TEXT    | NOT NULL (email, sms)                        |
| subject     | TEXT    | nullable                                     |
| body        | TEXT    | NOT NULL                                     |
| delay_days  | INTEGER | NOT NULL                                     |
| created_at  | TEXT    | NOT NULL                                     |
| updated_at  | TEXT    | NOT NULL                                     |

### enrollments

| Column             | Type    | Constraints                   |
|--------------------|---------|-------------------------------|
| id                 | TEXT    | PRIMARY KEY                   |
| person_id          | TEXT    | NOT NULL                      |
| sequence_id        | TEXT    | NOT NULL, FK -> sequences(id) |
| status             | TEXT    | NOT NULL (active, paused, completed, cancelled, replied) |
| current_step_order | INTEGER | NOT NULL                      |
| enrolled_at        | TEXT    | NOT NULL                      |
| paused_at          | TEXT    | nullable                      |
| completed_at       | TEXT    | nullable                      |
| cancelled_at       | TEXT    | nullable                      |

### enrollment_messages

| Column        | Type | Constraints                      |
|---------------|------|----------------------------------|
| id            | TEXT | PRIMARY KEY                      |
| enrollment_id | TEXT | NOT NULL, FK -> enrollments(id)  |
| message_id    | TEXT | NOT NULL                         |
| created_at    | TEXT | NOT NULL                         |

### sequence_comments

| Column      | Type | Constraints                                          |
|-------------|------|------------------------------------------------------|
| id          | TEXT | PRIMARY KEY                                          |
| sequence_id | TEXT | NOT NULL, FK -> sequences(id) ON DELETE CASCADE      |
| author_id   | TEXT | NOT NULL                                             |
| parent_id   | TEXT | nullable (self-ref for threading)                    |
| body        | TEXT | NOT NULL                                             |
| status      | TEXT | NOT NULL (open, resolved)                            |
| created_at  | TEXT | NOT NULL                                             |
| updated_at  | TEXT | NOT NULL                                             |

---

## Service Operations

### sequenceService (`services/sequence.ts`)

| Function       | Signature                                                        | Description                              |
|----------------|------------------------------------------------------------------|------------------------------------------|
| `add`          | `(db, tables, input, ctx?) => Sequence`                          | Create sequence. Input: `name`, `description?`, `status`. Emits `sequence.created`. |
| `list`         | `(db, tables, filters?) => Sequence[]`                           | List non-deleted sequences. Filter: `search` (name LIKE). Ordered by `created_at` desc. |
| `get`          | `(db, tables, idOrSlug) => SequenceDetail`                       | Get sequence by ID or name. Returns sequence + steps (ordered by step_order) + enrollmentStats (`total`, `active`, `completed`, `cancelled`, `paused`, `replied`). Excludes soft-deleted. |
| `update`       | `(db, tables, id, input, ctx?) => Sequence`                      | Update `name`, `description`. Emits `sequence.updated`. |
| `remove`       | `(db, tables, id, ctx?) => void`                                 | Soft-delete. Emits `sequence.deleted`.   |
| `restore`      | `(db, tables, id, ctx?) => void`                                 | Clears `deleted_at`. Throws if not deleted. Emits `sequence.restored`. |
| `purge`        | `(db, tables, id, ctx?) => void`                                 | Hard-delete sequence AND its steps. Emits `sequence.purged`. |
| `addStep`      | `(db, tables, sequenceId, input) => SequenceStep`                | Add step. Input: `channel`, `body`, `step_order`, `subject?`, `delay_days`. |
| `updateStep`   | `(db, tables, stepId, input) => SequenceStep`                    | Update step fields: `channel`, `subject`, `body`, `delay_days`. |
| `removeStep`   | `(db, tables, stepId) => void`                                   | Hard-delete a step.                      |
| `reorderSteps` | `(db, tables, sequenceId, stepIds) => SequenceStep[]`            | Reorder steps by providing ordered array of step IDs. Returns updated steps. |

### sequenceCommentService (`services/sequence-comment.ts`)

| Function    | Signature                                                        | Description                              |
|-------------|------------------------------------------------------------------|------------------------------------------|
| `add`       | `(db, tables, input, ctx?) => SequenceComment`                   | Create comment. Input: `sequence_id`, `author_id`, `body`, `parent_id?`. Status defaults to `open`. Emits `sequence.comment.created`. |
| `list`      | `(db, tables, sequenceId, filters?) => SequenceComment[]`        | List comments for sequence. Filter: `status`. Ordered by `created_at` desc. |
| `get`       | `(db, tables, id) => SequenceComment`                            | Get comment by ID. Throws if not found.  |
| `update`    | `(db, tables, id, input, ctx?) => SequenceComment`               | Update `body` and/or `status`. Emits `sequence.comment.updated`. |
| `remove`    | `(db, tables, id, ctx?) => void`                                 | Hard-delete comment. Emits `sequence.comment.deleted`. |
| `resolve`   | `(db, tables, id, ctx?) => SequenceComment`                      | Set status to `resolved`. Emits `sequence.comment.resolved`. |
| `getThread` | `(db, tables, parentId) => SequenceComment[]`                    | Get replies to a comment (by `parent_id`). Ordered by `created_at` desc. |

### enrollmentService (`services/enrollment.ts`)

| Function       | Signature                                                        | Description                              |
|----------------|------------------------------------------------------------------|------------------------------------------|
| `add`          | `(db, tables, input, ctx?) => Enrollment`                        | Enroll person. Input: `sequence_id`, `person_id`, `status`, `current_step_order`. Validates person and sequence exist. Prevents duplicate active enrollment. Emits `enrollment.created`. |
| `list`         | `(db, tables, filters?, pagination?) => Enrollment[]`            | List enrollments. Filters: `sequenceId`, `personId`, `status`. Pagination: `offset` (default 0), `limit` (default 50). |
| `get`          | `(db, tables, id) => EnrollmentDetail`                           | Get enrollment with expanded `person` (id, first_name, last_name), `sequence` (id, name), and `messages` (via enrollment_messages join). |
| `pause`        | `(db, tables, id, ctx?) => Enrollment`                           | Pause active enrollment. Sets `paused_at`. Throws if not active. Emits `enrollment.paused`. |
| `resume`       | `(db, tables, id, ctx?) => Enrollment`                           | Resume paused enrollment. Clears `paused_at`. Throws if not paused. Emits `enrollment.resumed`. |
| `cancel`       | `(db, tables, id, ctx?) => Enrollment`                           | Cancel enrollment. Sets `cancelled_at`. Throws if already completed/cancelled. Emits `enrollment.cancelled`. |
| `advance`      | `(db, tables) => PendingStep[]`                                  | Process all active enrollments: check delay, advance step order, mark completed when no more steps. Returns pending steps (enrollment + step + personEmail). |
| `linkMessage`  | `(db, tables, enrollmentId, messageId) => void`                  | Link a sent message to an enrollment via enrollment_messages. |
| `checkReplies` | `(db, tables) => void`                                           | Check for replies and update statuses. (TODO: not yet implemented) |

---

## CLI Commands

All list commands return a paginated response:
```json
{ "data": [...], "nextCursor": "<string | null>", "hasMore": false }
```

### `sequence` -- Manage sequences

```
# add (positional name or --input-json)
clam --json marketing sequence add "Welcome Series"
clam --json marketing sequence add --input-json '{"name":"Welcome Series","description":"Onboarding flow"}'

# list (paginated)
clam --json marketing sequence list
clam --json marketing sequence list --search "welcome" --cursor <cursor> --limit 25
clam --json marketing sequence list --input-json '{"search":"welcome","cursor":"<cursor>","limit":25}'

# get (positional id or --input-json)
clam --json marketing sequence get <id>
clam --json marketing sequence get --input-json '{"id":"<sequence-id>"}'

# update (positional id or --input-json)
clam --json marketing sequence update <id> --name "New Name"
clam --json marketing sequence update --input-json '{"id":"<sequence-id>","name":"New Name"}'

# remove / restore / purge (positional id or --input-json)
clam --json marketing sequence remove <id>
clam --json marketing sequence remove --input-json '{"id":"<sequence-id>"}'
clam --json marketing sequence restore <id>
clam --json marketing sequence restore --input-json '{"id":"<sequence-id>"}'
clam --json marketing sequence purge <id>
clam --json marketing sequence purge --input-json '{"id":"<sequence-id>"}'

# add-step (positional sequence-id or --input-json)
clam --json marketing sequence add-step <sequence-id> --channel email --body "Hello {{first_name}}" --step-order 1 --delay-days 0
clam --json marketing sequence add-step --input-json '{"sequence_id":"<id>","channel":"email","body":"Hello {{first_name}}","step_order":1,"delay_days":0}'

# update-step (positional step-id or --input-json; key is "id")
clam --json marketing sequence update-step <step-id> --body "Updated body"
clam --json marketing sequence update-step --input-json '{"id":"<step-id>","body":"Updated body","delay_days":3}'

# remove-step (positional step-id or --input-json; key is "step_id")
clam --json marketing sequence remove-step <step-id>
clam --json marketing sequence remove-step --input-json '{"step_id":"<step-id>"}'

# reorder (positional sequence-id or --input-json; "order" field accepts array or comma-separated string)
clam --json marketing sequence reorder <sequence-id> --order "<id1>,<id2>,<id3>"
clam --json marketing sequence reorder --input-json '{"sequence_id":"<id>","order":["<id1>","<id2>","<id3>"]}'
```

---

### `sequence comment` -- Manage sequence comments

> Note: `sequence comment get`, `resolve`, and `remove` use `comment_id` as the key in `--input-json` (not `id`).

```
# add (positional sequence-id or --input-json)
clam --json marketing sequence comment add <sequence-id> --body "Comment text"
clam --json marketing sequence comment add --input-json '{"sequence_id":"<id>","body":"Comment text"}'

# list (paginated; positional sequence-id or --input-json)
clam --json marketing sequence comment list <sequence-id>
clam --json marketing sequence comment list --input-json '{"sequence_id":"<id>","status":"open","cursor":"<cursor>","limit":25}'

# get (positional comment-id or --input-json; key is "comment_id")
clam --json marketing sequence comment get <comment-id>
clam --json marketing sequence comment get --input-json '{"comment_id":"<comment-id>"}'

# resolve (positional comment-id or --input-json; key is "comment_id")
clam --json marketing sequence comment resolve <comment-id>
clam --json marketing sequence comment resolve --input-json '{"comment_id":"<comment-id>"}'

# remove (positional comment-id or --input-json; key is "comment_id")
clam --json marketing sequence comment remove <comment-id>
clam --json marketing sequence comment remove --input-json '{"comment_id":"<comment-id>"}'
```

---

### `enroll` -- Manage sequence enrollments

```
# add (positional sequence-id + person-id or --input-json)
clam --json marketing enroll add <sequence-id> <person-id>
clam --json marketing enroll add --input-json '{"sequence_id":"<id>","person_id":"<id>"}'

# list (paginated)
clam --json marketing enroll list
clam --json marketing enroll list --sequence-id <id> --status active --cursor <cursor> --limit 25
clam --json marketing enroll list --input-json '{"sequence_id":"<id>","person_id":"<id>","status":"active","cursor":"<cursor>","limit":25}'

# get (positional id or --input-json)
clam --json marketing enroll get <id>
clam --json marketing enroll get --input-json '{"id":"<enrollment-id>"}'

# pause / resume / cancel (positional id or --input-json)
clam --json marketing enroll pause <id>
clam --json marketing enroll pause --input-json '{"id":"<enrollment-id>"}'
clam --json marketing enroll resume <id>
clam --json marketing enroll resume --input-json '{"id":"<enrollment-id>"}'
clam --json marketing enroll cancel <id>
clam --json marketing enroll cancel --input-json '{"id":"<enrollment-id>"}'

# advance (no args)
clam --json marketing enroll advance

# check-replies (no args)
clam --json marketing enroll check-replies
```

---

## Global Flags

- `--json` -- Produce compact JSON output (must be placed immediately after `clam`, before the subcommand). Required for machine-readable output.
- `--input-json <json>` -- Pass structured JSON input to any command. Overrides positional args and individual flags.

---

## Input JSON Shapes

Every command accepts `--input-json <json>`. Field resolution order: `input.<field> ?? positionalArg ?? opts.<flag>`.

### sequence add

```typescript
interface SequenceAddInput {
  name: string;                      // required (also accepted as positional)
  description?: string | null;
}
// status always set to "active" by the command
```

### sequence list

```typescript
interface SequenceListInput {
  search?: string;
  cursor?: string;
  limit?: number;
}
// Returns: { data: Sequence[], nextCursor: string | null, hasMore: boolean }
```

### sequence get

```typescript
interface SequenceGetInput {
  id: string;                        // required (also accepted as positional); UUID or name
}
// Returns: SequenceDetail with steps + enrollmentStats
```

### sequence update

```typescript
interface SequenceUpdateInput {
  id: string;                        // required (also accepted as positional)
  name?: string;
  description?: string | null;
}
```

### sequence remove / restore / purge

```typescript
interface SequenceIdInput {
  id: string;                        // required (also accepted as positional)
}
// remove returns: { ok: true, deleted: string }
// restore returns: { ok: true, restored: string }
// purge returns: { ok: true, purged: string }
```

### sequence add-step

```typescript
interface SequenceStepAddInput {
  sequence_id: string;               // required (also accepted as positional)
  channel: "email" | "sms";          // required
  body: string;                      // required; supports {{first_name}}, {{last_name}}, {{title}}
  step_order: number;                // required
  subject?: string | null;
  delay_days?: number;               // default: 0
}
```

### sequence update-step

```typescript
interface SequenceStepUpdateInput {
  id: string;                        // required (also accepted as positional); step UUID
  channel?: "email" | "sms";
  subject?: string | null;
  body?: string;
  delay_days?: number;
}
```

### sequence remove-step

```typescript
interface SequenceStepRemoveInput {
  step_id: string;                   // required (also accepted as positional); note: key is "step_id" not "id"
}
// Returns: { ok: true, removed: string }
```

### sequence reorder

```typescript
interface SequenceReorderInput {
  sequence_id: string;               // required (also accepted as positional)
  order: string[] | string;          // required; array of step UUIDs in desired order, or comma-separated string
}
```

### sequence comment add

```typescript
interface SequenceCommentAddInput {
  sequence_id: string;               // required (also accepted as positional)
  body: string;                      // required
  author_id?: string;                // UUID; defaults to active user
  parent_id?: string | null;         // UUID; for threading
}
```

### sequence comment list

```typescript
interface SequenceCommentListInput {
  sequence_id: string;               // required (also accepted as positional)
  status?: "open" | "resolved";
  cursor?: string;
  limit?: number;
}
// Returns: { data: SequenceComment[], nextCursor: string | null, hasMore: boolean }
```

### sequence comment get / resolve / remove

```typescript
interface SequenceCommentIdInput {
  comment_id: string;                // required (also accepted as positional); note: key is "comment_id" not "id"
}
```

### enroll add

```typescript
interface EnrollAddInput {
  sequence_id: string;               // required (also accepted as first positional)
  person_id: string;                 // required (also accepted as second positional)
}
// status always "active", current_step_order always 1
```

### enroll list

```typescript
interface EnrollListInput {
  sequence_id?: string;
  person_id?: string;
  status?: "active" | "paused" | "completed" | "cancelled" | "replied";
  cursor?: string;
  limit?: number;
}
// Returns: { data: Enrollment[], nextCursor: string | null, hasMore: boolean }
```

### enroll get / pause / resume / cancel

```typescript
interface EnrollIdInput {
  id: string;                        // required (also accepted as positional)
}
```

---

## API Routes

### Sequences

| Method | Path                              | Description              |
|--------|-----------------------------------|--------------------------|
| GET    | `/api/sequences`                  | List sequences.          |
| POST   | `/api/sequences`                  | Create sequence. Returns 201. |
| GET    | `/api/sequences/:id`              | Get sequence with steps + enrollment stats. |
| PUT    | `/api/sequences/:id`              | Update sequence.         |
| DELETE | `/api/sequences/:id`              | Soft-delete sequence.    |
| PUT    | `/api/sequences/:id/restore`      | Restore soft-deleted sequence. |
| DELETE | `/api/sequences/:id/purge`        | Hard-delete sequence + steps. |

### Sequence Steps

| Method | Path                              | Description              |
|--------|-----------------------------------|--------------------------|
| POST   | `/api/sequences/:id/steps`        | Add step. Returns 201.   |
| PUT    | `/api/sequence-steps/:id`         | Update step.             |
| DELETE | `/api/sequence-steps/:id`         | Remove step.             |
| PUT    | `/api/sequences/:id/steps/reorder`| Reorder steps. Body: `{ stepIds: string[] }`. |

### Enrollments

| Method | Path                              | Description              |
|--------|-----------------------------------|--------------------------|
| GET    | `/api/enrollments`                | List enrollments. Query: `sequenceId`, `personId`, `status`. |
| POST   | `/api/enrollments`                | Create enrollment. Returns 201. |
| GET    | `/api/enrollments/:id`            | Get enrollment detail.   |
| PUT    | `/api/enrollments/:id/pause`      | Pause enrollment.        |
| PUT    | `/api/enrollments/:id/resume`     | Resume enrollment.       |
| PUT    | `/api/enrollments/:id/cancel`     | Cancel enrollment.       |
| POST   | `/api/enrollments/advance`        | Advance all active enrollments. |
| POST   | `/api/enrollments/check-replies`  | Check replies.           |

### Sequence Comments

| Method | Path                                | Description              |
|--------|-------------------------------------|--------------------------|
| GET    | `/api/sequences/:id/comments`       | List comments for sequence. Query: `status`. |
| POST   | `/api/sequences/:id/comments`       | Create comment. Body: `{ author_id, body, parent_id? }`. Returns 201. |
| GET    | `/api/sequence-comments/:id`        | Get comment.             |
| PUT    | `/api/sequence-comments/:id`        | Update comment.          |
| DELETE | `/api/sequence-comments/:id`        | Delete comment.          |
| PUT    | `/api/sequence-comments/:id/resolve`| Resolve comment.         |

### Marketing Stats

| Method | Path                               | Description              |
|--------|------------------------------------|--------------------------|
| GET    | `/api/marketing/stats/sequences`   | Sequence performance stats. |
| GET    | `/api/marketing/stats/transports`  | Transport metrics.       |

---

## Events Emitted

| Event Type                  | Entity Type      | Trigger          |
|-----------------------------|------------------|------------------|
| `sequence.created`          | sequence         | Add              |
| `sequence.updated`          | sequence         | Update           |
| `sequence.deleted`          | sequence         | Soft-delete      |
| `sequence.restored`         | sequence         | Restore          |
| `sequence.purged`           | sequence         | Hard-delete      |
| `sequence.comment.created`  | sequence_comment | Add comment      |
| `sequence.comment.updated`  | sequence_comment | Update comment   |
| `sequence.comment.deleted`  | sequence_comment | Delete comment   |
| `sequence.comment.resolved` | sequence_comment | Resolve comment  |
| `enrollment.created`        | enrollment       | Enroll           |
| `enrollment.paused`         | enrollment       | Pause            |
| `enrollment.resumed`        | enrollment       | Resume           |
| `enrollment.cancelled`      | enrollment       | Cancel           |

## Permissions

| Resource   | Actions                          |
|------------|----------------------------------|
| sequence   | create, read, update, delete     |
| enrollment | create, read, pause, resume, cancel |
| comment    | create, read, update, delete     |
