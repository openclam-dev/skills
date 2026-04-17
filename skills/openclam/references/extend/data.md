---
name: extend-data
description: >
  The data layer for an extension — Zod insert/update/output schemas, dual
  SQLite + Postgres Drizzle tables, and versioned migrations. Covers the
  idempotency and never-modify-an-existing-migration rules.
metadata:
  author: openclam
  version: 0.1.0
---

# Data

An extension owns a slice of the database. Three pieces keep local SQLite
and hosted Postgres in step:

1. **Zod schemas** — separate `insert`, `update`, and `output` variants for
   each entity. Procedures validate against these; services return values
   that match them.
2. **Drizzle tables** — dialect-specific definitions under
   `src/tables/sqlite.ts` and `src/tables/postgres.ts`, both exported
   through `defineExtension({ tables: { sqlite, postgres } })`.
3. **Migrations** — an ordered list of idempotent SQL strings per dialect.
   The daemon applies them on extension enable and on startup.

The extension has implicit read-write on its own tables. To touch other
extensions' tables it must declare them in `capabilities.tables` and get
user approval at enable time (see `capabilities.md`).

## Minimal example (one entity: `invoice`)

### Schemas

```ts
// src/schemas/index.ts
import { z } from 'zod';
import { v4 as uuidv4 } from 'uuid';

export const insertInvoiceSchema = z.object({
  id: z.uuid().default(() => uuidv4()),
  customer_id: z.uuid(),
  amount: z.number().positive(),
  status: z.enum(['draft', 'sent', 'paid']).default('draft'),
  due_date: z.iso.date().nullable().default(null),
  created_at: z.iso.datetime().default(() => new Date().toISOString()),
  updated_at: z.iso.datetime().default(() => new Date().toISOString()),
  deleted_at: z.iso.datetime().nullable().default(null),
});

export const updateInvoiceSchema = z.object({
  customer_id: z.uuid().optional(),
  amount: z.number().positive().optional(),
  status: z.enum(['draft', 'sent', 'paid']).optional(),
  due_date: z.iso.date().nullable().optional(),
  updated_at: z.iso.datetime().default(() => new Date().toISOString()),
});

export const invoiceSchema = z.object({
  id: z.uuid(),
  customer_id: z.uuid(),
  amount: z.number(),
  status: z.enum(['draft', 'sent', 'paid']),
  due_date: z.iso.date().nullable(),
  created_at: z.iso.datetime(),
  updated_at: z.iso.datetime(),
  deleted_at: z.iso.datetime().nullable(),
});

export type Invoice = z.infer<typeof invoiceSchema>;
export type InsertInvoice = z.input<typeof insertInvoiceSchema>;
export type UpdateInvoice = z.infer<typeof updateInvoiceSchema>;
```

Three variants because they have different contracts:

- **`insert*Schema`** — what the caller provides. Defaults fill in `id`,
  timestamps, enum initial state. Use `z.input<...>` for the type so
  callers can omit defaulted fields.
- **`update*Schema`** — everything optional except audit fields. Only
  present keys are updated.
- **`<entity>Schema`** — the wire shape. This is what procedures return
  and what the typed client sees. Every field required (timestamps,
  ids, nullable columns as `.nullable()`).

### Tables (two dialects)

```ts
// src/tables/sqlite.ts
import { sqliteTable, text, integer, real } from 'drizzle-orm/sqlite-core';

export const invoices = sqliteTable('invoices', {
  id: text('id').primaryKey(),
  customer_id: text('customer_id').notNull(),
  amount: real('amount').notNull(),
  status: text('status').notNull(),
  due_date: text('due_date'),
  created_at: text('created_at').notNull(),
  updated_at: text('updated_at').notNull(),
  deleted_at: text('deleted_at'),
});
```

```ts
// src/tables/postgres.ts
import { pgTable, text, numeric, timestamp } from 'drizzle-orm/pg-core';

export const invoices = pgTable('invoices', {
  id: text('id').primaryKey(),
  customer_id: text('customer_id').notNull(),
  amount: numeric('amount').notNull(),
  status: text('status').notNull(),
  due_date: text('due_date'),
  created_at: text('created_at').notNull(),
  updated_at: text('updated_at').notNull(),
  deleted_at: text('deleted_at'),
});
```

Dialects differ on column helpers (`real` vs `numeric`, `integer(...,
{ mode: 'boolean' })` vs `pgBoolean`, `text` vs `timestamp`). Keep
schema-level types portable across dialects so Zod output shapes match.
If you need a richer type in one dialect, coerce at the service
boundary before returning.

### Migrations

```ts
// src/index.ts (excerpt)
import { defineExtension } from '@openclam/extension-sdk';
import * as sqliteTables from './tables/sqlite';
import * as postgresTables from './tables/postgres';

export const extension = defineExtension({
  id: 'invoicing',
  label: 'Invoicing',
  version: '0.1.0',
  tables: { sqlite: sqliteTables, postgres: postgresTables },
  migrations: [
    {
      version: 1,
      description: 'Initial invoices table',
      sqlite: `
        CREATE TABLE IF NOT EXISTS invoices (
          id TEXT PRIMARY KEY,
          customer_id TEXT NOT NULL,
          amount REAL NOT NULL,
          status TEXT NOT NULL,
          due_date TEXT,
          created_at TEXT NOT NULL,
          updated_at TEXT NOT NULL,
          deleted_at TEXT
        );
        CREATE INDEX IF NOT EXISTS invoices_customer_idx
          ON invoices(customer_id);
      `,
      postgres: `
        CREATE TABLE IF NOT EXISTS invoices (
          id TEXT PRIMARY KEY,
          customer_id TEXT NOT NULL,
          amount NUMERIC NOT NULL,
          status TEXT NOT NULL,
          due_date TEXT,
          created_at TEXT NOT NULL,
          updated_at TEXT NOT NULL,
          deleted_at TEXT
        );
        CREATE INDEX IF NOT EXISTS invoices_customer_idx
          ON invoices(customer_id);
      `,
    },
  ],
});
```

## Migration rules

### 1. Never modify an existing migration

Once a migration has landed (merged, released, or even run locally on a
developer machine), its SQL is frozen. To change the schema, append a
new migration with the next `version`. The daemon tracks applied
versions per extension and will not re-run an existing one, so edits to
an already-applied migration are silently lost.

### 2. Make every statement idempotent

Migrations may be retried after a partial failure, re-applied after a
dialect switch, or run on a workspace that already has some of the
tables. Every DDL should tolerate that:

- `CREATE TABLE IF NOT EXISTS`
- `CREATE INDEX IF NOT EXISTS`
- SQLite: `ALTER TABLE ... ADD COLUMN ...` (plain — SQLite errors if the
  column exists, so a column add must go in its own version and stay
  unmodified after that).
- Postgres: `ALTER TABLE ... ADD COLUMN IF NOT EXISTS ...`

### 3. `version` is monotonic per extension

Start at `1`, increment by `1` each migration. The daemon compares
against the `extensions_applied_versions` row for this extension and
runs every unapplied migration in order.

### 4. Both dialects must ship

Every migration entry has `sqlite` and `postgres` fields. Even if you
develop only against SQLite locally, the hosted Postgres path must be
kept in sync or the workspace will fail to start in cloud mode.

### 5. Prefer additive changes

Dropping or renaming columns in a running workspace is expensive. When
you need to replace a column:

1. Add the new column in version N.
2. Backfill via a follow-up data migration or a one-off script.
3. Stop reading the old column in the service (a code change, not a
   migration).
4. Drop the old column in a later version — only after every deployed
   workspace has run step 3.

A worked example pattern is `cron`'s `is_system` addition and
`webhooks-outbound`'s `slug` backfill — both visible in those packages.

## Soft delete

The convention is a nullable `deleted_at` timestamp column on user-
editable entities. Service list/get methods filter
`WHERE deleted_at IS NULL`; delete handlers set `deleted_at = now()`.
This keeps decision traces attached to rows that would otherwise
disappear.

If you have rows that must be hard-deleted (audit logs with retention
limits, encrypted payloads), say so explicitly in the service method
name (`purge`, not `remove`) so callers know which they're getting.

## Capability declarations for cross-extension tables

Even if you can technically query `tables.users` from your handler,
don't — declare the intent:

```ts
export const extension = defineExtension({
  id: 'invoicing',
  // ...
  capabilities: {
    tables: {
      users: 'read',              // read user names/emails for rendering
      entities: 'read',           // resolve customer_id → display name
    },
    network: true,                // call a payments provider
    systemCapabilities: ['events:emit'],
  },
});
```

The user sees these at `clam extension enable invoicing` and approves
them. Sandboxed extensions that touch an undeclared table at runtime
get an immediate proxy error. Trusted extensions still get a scoped db
so the same violation shows up in logs.

See `capabilities.md` for the full set of capability fields.
