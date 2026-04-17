---
name: automate-events
description: >
  Event emission and outbox processing. Event schema, `source/module` convention,
  pattern matching with wildcards, dead-letter queue, and the `clam event` /
  `clam events` CLI.
metadata:
  author: openclam
  version: 0.1.0
---

# Events

Every domain mutation writes an immutable row to the `events` outbox. A
background tick in the daemon picks up pending rows and fans each event to
in-process handlers, database rules, workflow triggers, and delivery hooks
(outbound webhooks). Events are the integration seam across the whole
execution layer.

## Event schema (`events` table)

| Column            | Type    | Notes                                                                |
| ----------------- | ------- | -------------------------------------------------------------------- |
| `id`              | uuid PK |                                                                      |
| `source`          | text    | Producing system — `'openclam'` for internal events; vendor name (`'stripe'`, `'resend'`) for inbound webhook events |
| `module`          | text?   | Subsystem inside OpenClam (`'contacts'`, `'cron'`, `'workflows'`). Null for external |
| `event_type`      | text    | `resource.action` — `person.created`, `deal.updated`                 |
| `entity_type`     | text    | Affected entity, e.g. `person`                                       |
| `entity_id`       | text    | UUID of the entity                                                   |
| `payload`         | text?   | JSON. Optional extra data                                            |
| `metadata`        | text?   | JSON. Producer context (workflow_name, inbound_endpoint_slug, …)     |
| `user_id`         | text?   | Actor who caused the event (null for system)                         |
| `status`          | text    | `pending` → `processing` → `completed` / `failed` / `dead`           |
| `attempts`        | int     | Processing attempts (max 5, then `dead`)                             |
| `failure_reason`  | text?   | Error message if failed                                              |
| `created_at`      | text    | ISO                                                                  |
| `processed_at`    | text?   | ISO, set on completion                                               |
| `schema_version`  | int     | Reserved, default `1`                                                |

## Pattern matching

Patterns are compared against `event_type` (not `source/event_type`):

| Pattern          | Matches                       |
| ---------------- | ----------------------------- |
| `"person.created"` | Exact match                 |
| `"person.*"`     | Any `person.*` event          |
| `"*"`            | All events                    |

Rules, webhook subscribers, workflow triggers, and `events.on()` handlers all
use this matcher (`matchPattern` in `@openclam/events`). Note: `*.created`
(wildcard prefix) is **not** supported — only `*` and trailing `.*`.

## Outbox processing lifecycle

1. `emitEvent()` inserts a row with `status='pending'`.
2. The daemon's event-loop worker coalesces notifications and calls
   `processEvents()`.
3. For each pending row, status flips to `processing`, then:
   - In-process `on()` handlers matching the pattern run inline.
   - Active rules are matched (pattern + optional conditions); each matching
     rule's `rule_actions` are enqueued to `scheduled_actions` for the
     ScheduledActionRunner to execute on the next tick. Workflow and function
     targets fire via their registered trigger hooks.
   - Registered delivery hooks run (outbound webhook delivery queues
     matching subscribers).
4. Row is marked `completed`. Any enqueue failures are stored in
   `failure_reason` but the event still counts as processed.
5. Thrown errors bump `attempts`; after 5 the row goes `dead`. `completed`
   events older than 30 days are auto-purged by a seeded system cron.

Concurrency: single daemon per database. Running multiple daemons against
the same DB is not supported.

## Emission convention

```ts
import { emitEvent } from '@openclam/events';

await emitEvent(db, tables, {
  source: 'openclam',        // or 'stripe' / 'resend' for inbound
  module: 'contacts',        // subsystem; null for external
  eventType: 'person.created',
  entityType: 'person',
  entityId: person.id,
  userId: ctx?.userId ?? null,
  payload: { stage: 'qualified' },
  metadata: { workflow_name: 'onboard-contact' },
});
```

Extensions call `emitEvent` after every successful mutation. Disable the
whole outbox (test scenarios) via `setEventsEnabled(false)`.

## Inline handlers

Register TS handlers without a DB row — run in the daemon process only:

```ts
import { on, off } from '@openclam/events';

on('person.created', async (event, db, tables) => { /* … */ });
off('person.created');              // clear all
off('person.created', handlerRef);  // clear one
```

Handlers run during `processEvents()`, not at emit time.

## Dead-letter queue & retry

```ts
listDead(db, tables, pagination?);     // status === 'failed'
retry(db, tables, eventId);            // reset attempts=0, status=pending
retryAll(db, tables);                  // requeue every failed row
```

## `clam event` CLI

```bash
clam event count [--type <t>] [--entity-type <et>] [--source <s>] [--status <st>]
clam event list  [--type …] [--entity-type …] [--source …] [--status …] \
                 [--cursor <c>] [--limit <n>] [--sort created_at|event_type|status] \
                 [--order asc|desc] [--all] [--fields <csv>]
clam event get <id>
clam event process   [--limit <n>]
clam event dead      [--cursor <c>] [--limit <n>] [--all]
clam event retry <id>
clam event retry-all
```

All commands accept `--input-json` for machine callers. `--schema` on `list`
and `get` prints the Zod schema as JSON Schema.

## `clam events` CLI (config)

Configure the daemon's event-processing loop:

```bash
clam events get
clam events set --enabled <bool> --interval <seconds>   # interval 1–300
clam events reset
```

Changes take effect on the daemon's next reconciliation.

See [`../surfaces.md`](../surfaces.md) to translate these operations to MCP,
HTTP, or the typed client.
