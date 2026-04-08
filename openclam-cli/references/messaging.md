# Messaging Domain Reference

Package: `@openclam/messaging`
Extension name: `messaging`
Dependencies: `contacts`

---

## Tables

### transports

| Column      | Type    | Constraints              |
|-------------|---------|--------------------------|
| id          | TEXT    | PRIMARY KEY              |
| name        | TEXT    | NOT NULL, UNIQUE         |
| type        | TEXT    | NOT NULL (`ses`, `resend`) |
| channel     | TEXT    | NOT NULL (`email`)       |
| config_json | TEXT    | NOT NULL                 |
| is_default  | INTEGER | NOT NULL (boolean)       |
| created_at  | TEXT    | NOT NULL                 |
| updated_at  | TEXT    | NOT NULL                 |
| deleted_at  | TEXT    | nullable (soft delete)   |

### messages

| Column         | Type | Constraints            |
|----------------|------|------------------------|
| id             | TEXT | PRIMARY KEY            |
| person_id      | TEXT | NOT NULL               |
| transport_name | TEXT | NOT NULL               |
| channel        | TEXT | NOT NULL               |
| to_address     | TEXT | NOT NULL               |
| subject        | TEXT | nullable               |
| body           | TEXT | NOT NULL               |
| status         | TEXT | NOT NULL (pending, sent, delivered, opened, replied, bounced, complained) |
| sent_at        | TEXT | nullable               |
| delivered_at   | TEXT | nullable               |
| opened_at      | TEXT | nullable               |
| replied_at     | TEXT | nullable               |
| bounced_at     | TEXT | nullable               |
| external_id    | TEXT | nullable               |
| created_at     | TEXT | NOT NULL               |
| deleted_at     | TEXT | nullable (soft delete)  |

---

## Service Operations

### messageService (`services/message.ts`)

| Function  | Signature                                                        | Description                              |
|-----------|------------------------------------------------------------------|------------------------------------------|
| `list`    | `(db, tables, filters?, pagination?) => Message[]`               | List messages. Filters: `personId`, `status`, `channel`. Pagination: `offset` (default 0), `limit` (default 50). Excludes soft-deleted. |
| `get`     | `(db, tables, id) => Message`                                    | Get single message by ID. Excludes soft-deleted. Throws if not found. |
| `stats`   | `(db, tables) => MessageStats`                                   | Aggregate counts by status: total, pending, sent, delivered, opened, replied, bounced, complained. Includes all messages (even soft-deleted). |
| `remove`  | `(db, tables, id, ctx?) => void`                                 | Soft-delete: sets `deleted_at`. Emits `message.deleted` event. |
| `restore` | `(db, tables, id, ctx?) => void`                                 | Clears `deleted_at`. Throws if not already deleted. Emits `message.restored` event. |
| `purge`   | `(db, tables, id, ctx?) => void`                                 | Hard-delete: removes row permanently. Emits `message.purged` event. |

### transportAdminService (`services/transport-admin.ts`)

| Function     | Signature                                                        | Description                              |
|--------------|------------------------------------------------------------------|------------------------------------------|
| `add`        | `(db, tables, input, ctx?) => TransportRecord`                   | Create transport. Input: `name`, `type` (ses/resend), `channel` (default email), `config` (JSON object), `is_default?`. If `is_default`, clears previous default. Throws on duplicate name. Emits `transport.created`. |
| `list`       | `(db, tables) => TransportRecord[]`                              | List all non-deleted transports.         |
| `get`        | `(db, tables, nameOrId) => TransportRecord`                      | Lookup by name or UUID. Excludes soft-deleted. Throws if not found. |
| `update`     | `(db, tables, name, input, ctx?) => TransportRecord`             | Update `config` and/or `is_default`. If setting default, clears previous. Emits `transport.updated`. |
| `remove`     | `(db, tables, name, ctx?) => void`                               | Soft-delete. Emits `transport.deleted`.  |
| `restore`    | `(db, tables, nameOrId, ctx?) => void`                           | Clears `deleted_at`. Throws if not deleted. Emits `transport.restored`. |
| `purge`      | `(db, tables, nameOrId, ctx?) => void`                           | Hard-delete row. Emits `transport.purged`. |
| `setDefault` | `(db, tables, name) => void`                                     | Clears all defaults, then sets the named transport as default. |

### sendService (`services/send.ts`)

| Function        | Signature                                                          | Description                              |
|-----------------|--------------------------------------------------------------------|------------------------------------------|
| `sendEmail`     | `(db, tables, personId, opts, ctx?) => SendResult`                 | Send email to a person. Resolves primary email from `person_emails`. Uses named or default transport. Creates message row (status: pending), sends via transport, updates to sent/bounced. Emits `message.sent` on success. Opts: `subject`, `body`, `from`, `transportName?`, `replyTo?`. Returns: `{ messageId, success, externalId?, error? }`. |
| `preview`       | `(db, tables, personId, opts) => PreviewResult`                    | Preview message with template substitution (`{{first_name}}`, `{{last_name}}`, `{{title}}`). Does not send. Returns: `{ to, from, subject, body }`. |
| `testTransport` | `(db, tables, address) => SendResult`                              | Send test message to self via default transport. Subject: "OpenClam Transport Test". |

---

## CLI Commands

All list commands return a paginated response:
```json
{ "data": [...], "nextCursor": "<string | null>", "hasMore": false }
```

### `message` -- View messages

```
# list (paginated; positional or --input-json)
clam --json messaging message list
clam --json messaging message list --person-id <id> --status sent --limit 25 --cursor <cursor>
clam --json messaging message list --input-json '{"person_id":"<id>","channel":"email","status":"sent","cursor":"<cursor>","limit":25}'

# get (positional id or --input-json)
clam --json messaging message get <id>
clam --json messaging message get --input-json '{"id":"<message-id>"}'

# stats
clam --json messaging message stats

# remove / restore / purge (positional id or --input-json)
clam --json messaging message remove <id>
clam --json messaging message remove --input-json '{"id":"<message-id>"}'
clam --json messaging message restore <id>
clam --json messaging message restore --input-json '{"id":"<message-id>"}'
clam --json messaging message purge <id>
clam --json messaging message purge --input-json '{"id":"<message-id>"}'
```

---

### `transport` -- Manage message transports

> Transport commands use `name` (not `id`) as the identifier for get, update, remove, restore, purge, and set-default.

```
# add (positional name or --input-json)
# Note: config is a JSON object (not a string) in --input-json
clam --json messaging transport add "my-ses" --type ses --config '{"region":"us-east-1","key":"...","secret":"..."}'
clam --json messaging transport add --input-json '{"name":"my-ses","type":"ses","config":{"region":"us-east-1","key":"...","secret":"..."}}'

# list (paginated)
clam --json messaging transport list
clam --json messaging transport list --cursor <cursor> --limit 25
clam --json messaging transport list --input-json '{"cursor":"<cursor>","limit":25}'

# get (positional name or --input-json)
clam --json messaging transport get <name>
clam --json messaging transport get --input-json '{"name":"<transport-name>"}'

# update (positional name or --input-json)
clam --json messaging transport update <name> --default
clam --json messaging transport update --input-json '{"name":"<transport-name>","is_default":true}'

# remove / restore / purge (positional name or --input-json)
clam --json messaging transport remove <name>
clam --json messaging transport remove --input-json '{"name":"<transport-name>"}'
clam --json messaging transport restore <name>
clam --json messaging transport restore --input-json '{"name":"<transport-name>"}'
clam --json messaging transport purge <name>
clam --json messaging transport purge --input-json '{"name":"<transport-name>"}'

# set-default (positional name or --input-json)
clam --json messaging transport set-default <name>
clam --json messaging transport set-default --input-json '{"name":"<transport-name>"}'
```

---

### `send` -- Send messages

```
# email (positional person-id or --input-json)
clam --json messaging send email <person-id> --subject "Hello" --body "Hi there" --from "me@example.com"
clam --json messaging send email --input-json '{"person_id":"<id>","subject":"Hello","body":"Hi there","from":"me@example.com","transport":"my-ses"}'

# preview (positional person-id or --input-json)
clam --json messaging send preview <person-id> --subject "Hello" --body "Hi {{first_name}}" --from "me@example.com"
clam --json messaging send preview --input-json '{"person_id":"<id>","subject":"Hello","body":"Hi {{first_name}}","from":"me@example.com"}'

# test (uses --to flag or --input-json)
clam --json messaging send test --to test@example.com
clam --json messaging send test --input-json '{"to":"test@example.com"}'
```

---

## Global Flags

- `--json` -- Produce compact JSON output (must be placed immediately after `clam`, before the subcommand). Required for machine-readable output.
- `--input-json <json>` -- Pass structured JSON input to any command. Overrides positional args and individual flags.

---

## Input JSON Shapes

Every command accepts `--input-json <json>`. Field resolution order: `input.<field> ?? positionalArg ?? opts.<flag>`.

### message list

```typescript
interface MessageListInput {
  person_id?: string;
  channel?: string;                  // "email"
  status?: string;                   // pending | sent | delivered | opened | replied | bounced | complained
  cursor?: string;
  limit?: number;
}
// Returns: { data: Message[], nextCursor: string | null, hasMore: boolean }
```

### message get / remove / restore / purge

```typescript
interface MessageIdInput {
  id: string;                        // required (also accepted as positional)
}
// remove returns: { ok: true, deleted: string }
// restore returns: { ok: true, restored: string }
// purge returns: { ok: true, purged: string }
```

### transport add

```typescript
interface TransportAddInput {
  name: string;                      // required (also accepted as positional)
  type: "ses" | "resend";            // required
  config: Record<string, unknown>;   // required; JSON object (NOT a string)
  is_default?: boolean;              // default: false
}
```

### transport list

```typescript
interface TransportListInput {
  cursor?: string;
  limit?: number;
}
// Returns: { data: TransportRecord[], nextCursor: string | null, hasMore: boolean }
```

### transport get / remove / restore / purge / set-default

```typescript
interface TransportNameInput {
  name: string;                      // required (also accepted as positional); transport name (not UUID)
}
// remove returns: { ok: true, removed: string }
// restore returns: { ok: true, restored: string }
// purge returns: { ok: true, purged: string }
// set-default returns: { ok: true, default: string }
```

### transport update

```typescript
interface TransportUpdateInput {
  name: string;                      // required (also accepted as positional); transport name
  config?: Record<string, unknown>;  // JSON object (NOT a string)
  is_default?: boolean;
}
```

### send email

```typescript
interface SendEmailInput {
  person_id: string;                 // required (also accepted as positional)
  subject: string;                   // required
  body: string;                      // required
  from: string;                      // required; sender address
  transport?: string;                // transport name; uses default if omitted
}
// Returns: { messageId, success, externalId?, error? }
```

### send preview

```typescript
interface SendPreviewInput {
  person_id: string;                 // required (also accepted as positional)
  subject: string;                   // required
  body: string;                      // required; supports {{first_name}}, {{last_name}}, {{title}}
  from: string;                      // required
}
// Returns: { to, from, subject, body } — no message is sent
```

### send test

```typescript
interface SendTestInput {
  to: string;                        // required; recipient address
}
// Sends test message via default transport
// Returns: { messageId, success, externalId?, error? }
```

---

## API Routes

### Messages

| Method | Path                        | Description              |
|--------|-----------------------------|--------------------------|
| GET    | `/api/messages`             | List messages. Query: `personId`, `status`, `channel`. |
| GET    | `/api/messages/stats`       | Get message stats.       |
| GET    | `/api/messages/:id`         | Get message by ID.       |
| DELETE | `/api/messages/:id`         | Soft-delete message.     |
| PUT    | `/api/messages/:id/restore` | Restore soft-deleted message. |
| DELETE | `/api/messages/:id/purge`   | Hard-delete message.     |

### Send

| Method | Path                 | Description              |
|--------|----------------------|--------------------------|
| POST   | `/api/send`          | Send email. Body: `{ personId, subject, body, from, transportName? }`. Returns 201. |
| POST   | `/api/send/preview`  | Preview message. Body: `{ personId, subject, body, from }`. |
| POST   | `/api/send/test`     | Test default transport. Body: `{ address }`. |

### Transports

| Method | Path                             | Description              |
|--------|----------------------------------|--------------------------|
| GET    | `/api/transports`                | List transports.         |
| POST   | `/api/transports`                | Create transport. Returns 201. |
| GET    | `/api/transports/:name`          | Get transport by name.   |
| PUT    | `/api/transports/:name`          | Update transport.        |
| PUT    | `/api/transports/:name/default`  | Set transport as default. |
| DELETE | `/api/transports/:name`          | Soft-delete transport.   |
| PUT    | `/api/transports/:name/restore`  | Restore soft-deleted transport. |
| DELETE | `/api/transports/:name/purge`    | Hard-delete transport.   |

---

## Events Emitted

| Event Type           | Entity Type | Trigger          |
|----------------------|-------------|------------------|
| `message.sent`       | message     | Email sent       |
| `message.deleted`    | message     | Soft-delete      |
| `message.restored`   | message     | Restore          |
| `message.purged`     | message     | Hard-delete      |
| `transport.created`  | transport   | Add transport    |
| `transport.updated`  | transport   | Update transport |
| `transport.deleted`  | transport   | Soft-delete      |
| `transport.restored` | transport   | Restore          |
| `transport.purged`   | transport   | Hard-delete      |

## Permissions

| Resource  | Actions                    |
|-----------|----------------------------|
| transport | create, read, update, delete |
| message   | create, read               |
