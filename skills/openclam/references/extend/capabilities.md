---
name: extend-capabilities
description: >
  Declarative capabilities an extension contributes beyond procedures —
  `actions`, `events`, `recommendedRules`, `scheduledJobs`, `enrichments`,
  `permissions`, and `commands`. When to use each, with minimal examples.
metadata:
  author: openclam
  version: 0.1.0
---

# Capabilities

Procedures are the main way an extension exposes behaviour to callers, but
a full extension contributes several other declarative pieces. Each is a
field on the `ExtensionDef` you pass to `defineExtension`:

| Field | What it gives you |
| --- | --- |
| `permissions` | Nested `resource:action:description` map referenced by procedures and CLI commands. |
| `actions` | Rule-callable operations — the automation pipeline dispatches events into them. |
| `events` | Domain events this extension emits, with Zod schemas — used by rules and webhooks. |
| `recommendedRules` | Suggested automations offered at extension-enable time. |
| `scheduledJobs` | System cron jobs seeded on enable, removed on disable. |
| `enrichments` | `--expand` options this extension contributes on other extensions' entities. |
| `commands` | CLI subtrees (CommandDefs) this extension contributes. |
| `capabilities` | Cross-extension table access, network, filesystem, system capability tokens. |

## Minimal example (one of each)

```ts
import { defineExtension } from '@openclam/extension-sdk';
import { z } from 'zod';

export const extension = defineExtension({
  id: 'invoicing',
  label: 'Invoicing',
  version: '0.1.0',

  permissions: {
    invoice: {
      create: 'Create invoices',
      read: 'View invoices',
      update: 'Edit invoices',
      delete: 'Delete invoices',
      send: 'Mark invoice as sent',
    },
  },

  capabilities: {
    tables: { users: 'read' },
    network: true,                   // outbound HTTP to payments provider
    systemCapabilities: ['events:emit'],
  },

  actions: {
    'send-invoice': {
      description: 'Mark the target invoice as sent and email the customer',
      paramsSchema: z.object({ invoice_id: z.uuid() }),
      execute: async (_event, params, db, tables) => {
        // ... business logic ...
      },
    },
  },

  events: {
    'invoice.created': {
      schema: z.object({ id: z.uuid(), amount: z.number() }),
      description: 'A new invoice was created',
    },
    'invoice.sent': {
      schema: z.object({ id: z.uuid(), sent_at: z.iso.datetime() }),
      description: 'An invoice was marked as sent',
    },
  },

  recommendedRules: [
    {
      slug: 'invoicing-auto-send-on-approval',
      name: 'Auto-send invoices when approved',
      description: 'When an invoice is approved, send it to the customer.',
      event_pattern: 'invoice.approved',
      actions: [{ extension: 'invoicing', action: 'send-invoice' }],
    },
  ],

  scheduledJobs: [
    {
      slug: 'invoicing-overdue-sweep',
      name: 'Mark overdue invoices daily',
      schedule: '0 8 * * *',
      handler: 'action:invoicing/mark-overdue',
    },
  ],

  enrichments: [
    {
      entityTable: 'persons',
      name: 'invoices',
      table: 'invoices',
      joinColumn: 'customer_id',
      cardinality: 'many',
      label: 'Outstanding and recent invoices for this person',
    },
  ],

  tables: { sqlite: {}, postgres: {} },
});
```

Everything below explains the pieces individually.

## Permissions

```ts
permissions: {
  invoice: {
    create: 'Create invoices',
    read: 'View invoices',
    update: 'Edit invoices',
    delete: 'Delete invoices',
  },
}
```

- Nested shape: `{ <resource>: { <action>: <description> } }`.
- The description text is shown to admins granting permissions to users,
  API keys, and agents.
- Reference them from procedures as the string `'<resource>:<action>'`:
  `defineProcedure({ permission: 'invoice:create', ... })`.
- `'*'` is the admin wildcard — it bypasses all per-procedure checks.
- Use `permission: null` for endpoints callable by any authenticated
  user (e.g. a health check).

## Actions

Actions are rule-callable operations. When a rule matches an event, the
automation pipeline dispatches into one or more actions. An action has a
Zod-validated `paramsSchema` and an `execute` function:

```ts
import type { ExtensionAction } from '@openclam/extension-sdk';

const markOverdue: ExtensionAction = {
  description: 'Mark every invoice past its due date as overdue',
  paramsSchema: z.object({}),
  execute: async (_event, _params, db, tables) => {
    const today = new Date().toISOString().slice(0, 10);
    await db.update(tables.invoices)
      .set({ status: 'overdue', updated_at: new Date().toISOString() })
      .where(/* due_date < today AND status != 'paid' */)
      .run();
  },
};
```

Shape:

```ts
interface ExtensionAction {
  description: string;
  paramsSchema: z.ZodType;
  execute: (
    event: ActionEvent,
    params: Record<string, unknown>,
    db: OpenClamDb,
    tables: OpenClamTables,
    ctx?: { userId?: string | null },
  ) => Promise<void>;
}
```

Actions are how extensions expose "do this when X happens" to the rule
engine and workflows. They are **not** called by procedure handlers
directly — if procedure code wants the same behaviour, import the
underlying service method and call that.

Action names are local to the extension (`mark-overdue`). Rules and
scheduled jobs address them by the fully-qualified
`<extension>/<action>` form.

## Events

Declare every domain event your extension emits — with a Zod schema so
rules, webhooks, and the typed event catalog know what the payload
shape is:

```ts
events: {
  'invoice.created': {
    schema: z.object({
      id: z.uuid(),
      customer_id: z.uuid(),
      amount: z.number(),
      created_at: z.iso.datetime(),
    }),
    description: 'A new invoice was created',
  },
  'invoice.sent': {
    schema: z.object({
      id: z.uuid(),
      sent_at: z.iso.datetime(),
    }),
    description: 'An invoice was sent to the customer',
  },
},
```

Emit from a service or handler:

```ts
await ctx.emit('invoice.created', { id, customer_id, amount, created_at });
```

Emission goes through the event outbox — rules, webhooks-outbound, and
any other subscriber see it on the next tick. The event type naming
convention is `<resource>.<past-tense-verb>`: `invoice.created`,
`invoice.sent`, `invoice.marked_paid`.

## Recommended rules

Recommended rules are the "wire me up" automations the user is prompted
to enable at `clam extension enable`. They make an extension functional
out-of-the-box while still giving the user control:

```ts
recommendedRules: [
  {
    slug: 'invoicing-auto-send-on-approval',
    name: 'Auto-send invoices when approved',
    description: 'When an invoice is approved, email it to the customer.',
    event_pattern: 'invoice.approved',
    priority: 10,
    actions: [
      { extension: 'invoicing', action: 'send-invoice' },
    ],
    optional: false,               // checked in the prompt by default
  },
  {
    slug: 'invoicing-slack-on-payment',
    name: 'Post to Slack when invoice paid',
    event_pattern: 'invoice.marked_paid',
    actions: [
      { extension: 'slack', action: 'post-message',
        params_json: '{"channel":"#billing"}' },
    ],
    optional: true,                // unchecked by default
  },
],
```

- `slug` is the stable id — idempotent across re-enables. If a rule
  with the same slug already exists in the workspace, it is skipped.
- `event_pattern` supports wildcards (`invoice.*`, `*.created`).
- `actions` can reference actions in any extension, not just the
  declaring one.
- `optional: true` leaves the rule unchecked in the multi-select
  prompt; users opt in explicitly. `optional: false` (or absent) is
  checked by default.

At `clam extension enable <id>` time, the user sees the full list in a
multi-select prompt. `-y` / `--accept-recommended` accepts all; `--no-
recommended` skips the prompt entirely.

## Scheduled jobs

System cron jobs that should run whenever the extension is enabled.
Seeded with `is_system: true` so users see them in `clam schedule list`
but can't edit or delete them:

```ts
scheduledJobs: [
  {
    slug: 'invoicing-overdue-sweep',
    name: 'Mark overdue invoices daily',
    description: 'Runs every morning at 08:00 UTC',
    schedule: '0 8 * * *',                      // croner syntax
    handler: 'action:invoicing/mark-overdue',   // action:<ext>/<name>
    timezone: 'UTC',
  },
  {
    slug: 'invoicing-monthly-rollup',
    name: 'Monthly revenue rollup workflow',
    schedule: '0 2 1 * *',
    handler: 'workflow:monthly-revenue-rollup', // workflow:<slug>
  },
],
```

Handler forms:
- `action:<extension>/<action>` — dispatches the named action.
- `workflow:<slug>` — starts the named workflow run.
- A bare handler name registered elsewhere (advanced).

Seeded on enable, removed on disable — users never curate system jobs
directly.

## Enrichments

Enrichments let an extension contribute extra data to entities owned by
other extensions, surfaced via `--expand` on list/get queries:

```ts
enrichments: [
  {
    entityTable: 'persons',    // table key of the target entity
    name: 'invoices',          // becomes --expand invoices
    table: 'invoices',         // this extension's table key
    joinColumn: 'customer_id', // column that references persons.id
    cardinality: 'many',       // array of rows (vs 'one' = flat object)
    columns: ['id', 'amount', 'status', 'due_date'],
    label: 'Recent invoices for this person',
  },
],
```

From the CLI:

```bash
clam crm person get <id> --expand invoices
```

`cardinality: 'one'` is for 1:1 enrichments (e.g. an ICP score row per
org). `cardinality: 'many'` is for 1:N (e.g. invoices per person).
`columns` is optional — omit to return every column on the enrichment
table.

Name collisions are checked at load time: two extensions can't both
contribute `--expand foo` for the same entity table.

## Commands

CLI contributions are `CommandDef[]` returned from a function:

```ts
import type { CommandContext, CommandDef } from '@openclam/extension-sdk';

export function getInvoicingCommands(_ctx: CommandContext): CommandDef[] {
  return [{
    name: 'invoice',
    description: 'Manage invoices',
    commands: [
      {
        name: 'list',
        description: 'List invoices',
        options: [
          { flags: '--cursor <cursor>', description: 'Pagination cursor' },
          { flags: '--limit <n>', description: 'Limit', parser: 'int' },
        ],
        handler: async (_args, opts, ctx) => {
          await ctx.requirePermission('invoice:read');
          return invoiceService.list(ctx.db, ctx.tables, { cursor: opts.cursor, limit: opts.limit });
        },
      },
      {
        name: 'send',
        description: 'Mark an invoice as sent',
        args: [{ name: 'id', required: true }],
        handler: async (args, _opts, ctx) => {
          await ctx.requirePermission('invoice:send');
          return invoiceService.send(ctx.db, ctx.tables, args.id);
        },
      },
    ],
  }];
}
```

Wire it into `defineExtension`:

```ts
defineExtension({
  // ...
  commands: getInvoicingCommands,
});
```

Invoked as:

```bash
clam invoicing invoice list
clam invoicing invoice send <id>
```

The same `CommandDef[]` works in both direct (local DB) and forwarded
(daemon HTTP) modes — the CLI shell translates it to a Commander tree
at runtime.

See `../surfaces.md` for surface translation (CLI ↔ MCP ↔ HTTP ↔ typed
client).

## `capabilities` — the sandbox manifest

Used for everything outside the extension's own tables:

```ts
capabilities: {
  tables: {
    users: 'read',               // read-only access to core.users
    tags: 'read-write',          // mutate core.tags
  },
  network: true,                 // can make outbound HTTP
  filesystem: true,              // can read/write outside ~/.openclam/extensions/<id>/
  systemCapabilities: [
    'events:emit',               // may call ctx.emit
    'embeddings:enqueue',        // may schedule embedding jobs
  ],
}
```

- `tables` — cross-extension access. The scoped db proxy enforces this
  at request time for sandboxed extensions. Trusted extensions still
  get the proxy for observability.
- `network`, `filesystem` — physical resource flags.
- `systemCapabilities` — opaque tokens the daemon checks before
  granting access to shared services (event emission, the embeddings
  queue, tag service, API key service, and so on).

The user reviews these at `clam extension enable` time and approves
them. If the hash changes (a new capability added in a later version),
enable re-prompts.

## Choosing where behaviour lives

A quick checklist when adding a new piece of functionality:

- **Users should be able to invoke it over HTTP / CLI / typed client**
  → procedure.
- **Rules should be able to trigger it in response to events** →
  action.
- **Other extensions should react to it happening** → event.
- **It should run on a schedule regardless of user input** → scheduled
  job.
- **It attaches data to another extension's entity** → enrichment.
- **It's an ergonomic shortcut for humans using the CLI** → command.

Most real features need two or three of these working together — the
`invoice.created` event, the `send-invoice` action, and the
`invoicing-auto-send-on-approval` recommended rule tie them together.
