---
name: openclam-build-extension
description: >
  Build custom OpenClam extensions. Use when the user wants to add a new domain,
  entity, or capability to their OpenClam instance — e.g. "add invoicing",
  "I need a tickets extension", "create a custom extension for inventory".
  Covers scaffolding, schema design, service patterns, routes, commands,
  migrations, and the full extension lifecycle.
metadata:
  author: openclam
  version: 0.1.0
tags:
  - extensions
  - scaffolding
  - custom
  - schema
  - services
  - routes
  - commands
---

# Building OpenClam Extensions

This skill guides you through creating custom extensions for OpenClam. Extensions add new domains (entities, services, routes, CLI commands) to an OpenClam instance.

## Quick Start

The fastest path is `clam extension init`:

```bash
# Bare skeleton — empty files, you fill in the logic
clam extension init invoicing

# Scaffolded — generates full CRUD for an entity
clam extension init invoicing --scaffold --entity invoice \
  --fields "number:string, amount:number, status:enum(draft,sent,paid), customer_id:uuid?, due_date:date?"
```

This creates the extension in `~/.openclam/extensions/invoicing/`. Enable it with:

```bash
clam workspace extension enable invoicing
```

## Extension Location

Custom extensions live in `~/.openclam/extensions/<name>/`. The runtime scans this directory on startup and dynamically loads any valid extension packages alongside the built-in ones.

Each extension is a standalone TypeScript package with this structure:

```
~/.openclam/extensions/invoicing/
  package.json
  tsconfig.json
  src/
    index.ts              ← Extension definition (the main export)
    schemas/
      index.ts            ← Zod insert/update/output schemas
    tables/
      sqlite.ts           ← Drizzle table definitions (SQLite)
      postgres.ts         ← Drizzle table definitions (Postgres)
    services/
      invoice.ts          ← CRUD + import/export logic
      index.ts            ← Re-exports
    routes.ts             ← Hono REST API routes
    commands.ts           ← Commander CLI commands
```

## Extension Interface

Every extension must export an object conforming to the `Extension` interface from `@openclam/core`:

```typescript
import type { Extension } from '@openclam/core';

export const extension: Extension = {
  name: 'invoicing',           // Unique identifier
  label: 'Invoicing',          // Human-readable label
  version: '0.1.0',            // Semver
  dependencies: [],             // Other extensions this depends on (by name)

  permissions: {
    invoice: {
      create: 'Create invoices',
      read: 'View invoices',
      update: 'Edit invoices',
      delete: 'Delete invoices',
    },
  },

  actions: {},                  // Automatable rule actions

  tables: {
    sqlite: sqliteTables,       // Drizzle table definitions
    postgres: postgresTables,
  },

  migrations: [
    {
      version: 1,
      description: 'Initial invoices table',
      sqlite: `CREATE TABLE IF NOT EXISTS invoices (...)`,
      postgres: `CREATE TABLE IF NOT EXISTS invoices (...)`,
    },
  ],

  getCommands(ctx) { return getInvoicingCommands(ctx); },
  registerRoutes(app, ctx) { registerInvoicingRoutes(app, ctx); },
};
```

The runtime looks for the export in this order: `extension`, `{name}Extension`, `default`.

## Field Types

When scaffolding with `--fields`, supported types are:

| Type | Zod | SQLite | Postgres | Example |
|------|-----|--------|----------|---------|
| `string` | `z.string()` | `TEXT` | `TEXT` | `name:string` |
| `text` | `z.string()` | `TEXT` | `TEXT` | `body:text` |
| `number` | `z.number()` | `INTEGER` | `INTEGER` | `amount:number` |
| `boolean` | `z.boolean()` | `INTEGER` | `BOOLEAN` | `is_paid:boolean` |
| `uuid` | `z.uuid()` | `TEXT` | `TEXT` | `customer_id:uuid` |
| `date` | `z.string()` | `TEXT` | `TEXT` | `due_date:date` |
| `enum(a,b,c)` | `z.enum([...])` | `TEXT` | `TEXT` | `status:enum(draft,sent,paid)` |

Append `?` to make nullable: `customer_id:uuid?`

## Patterns to Follow

### Schema Pattern

Every entity needs three Zod schemas:

```typescript
// Insert — has defaults for id, timestamps
export const insertInvoiceSchema = z.object({
  id: z.uuid().default(() => uuidv4()),
  number: z.string().min(1),
  amount: z.number(),
  status: z.enum(['draft', 'sent', 'paid']).default('draft'),
  customer_id: z.uuid().nullable().default(null),
  created_at: z.iso.datetime().default(() => new Date().toISOString()),
  updated_at: z.iso.datetime().default(() => new Date().toISOString()),
  deleted_at: z.iso.datetime().nullable().default(null),
});

// Update — all fields optional except updated_at
export const updateInvoiceSchema = z.object({
  number: z.string().min(1).optional(),
  amount: z.number().optional(),
  status: z.enum(['draft', 'sent', 'paid']).optional(),
  customer_id: z.uuid().nullable().optional(),
  updated_at: z.iso.datetime().default(() => new Date().toISOString()),
});

// Output — strict, no defaults
export const invoiceSchema = z.object({
  id: z.uuid(),
  number: z.string(),
  amount: z.number(),
  // ... all fields with their types, no defaults
});

export type Invoice = z.infer<typeof invoiceSchema>;
```

### Table Pattern

Both SQLite and Postgres table definitions are required:

```typescript
// tables/sqlite.ts
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core';

export const invoices = sqliteTable('invoices', {
  id: text('id').primaryKey(),
  number: text('number').notNull(),
  amount: integer('amount').notNull(),
  status: text('status').notNull().default('draft'),
  customer_id: text('customer_id'),
  created_at: text('created_at').notNull(),
  updated_at: text('updated_at').notNull(),
  deleted_at: text('deleted_at'),
});
```

### Service Pattern

Every entity service follows this structure:

```typescript
// Core CRUD
export async function add(db, tables, input) { ... }
export async function list(db, tables, filters, pagination) { ... }
export async function get(db, tables, id) { ... }
export async function update(db, tables, id, input) { ... }
export async function remove(db, tables, id) { ... }  // soft delete

// Import/Export
export async function importCsv(db, tables, rows) { ... }
export async function importFromFile(db, tables, filePath, format?) { ... }
export async function exportData(db, tables) { ... }
export async function exportToFile(db, tables, filePath?, format?) { ... }
```

Key conventions:
- All services receive `db` and `tables` as first two args (not bound to a specific instance)
- Use `isNull(tables.xxx.deleted_at)` for soft-delete filtering
- Use `@openclam/schemas` pagination helpers: `clampLimit`, `decodeCursor`, `applyPagination`
- Import/export uses `papaparse` for CSV

### Routes Pattern

```typescript
export function registerInvoicingRoutes(app: Hono<AppEnv>, ctx: ExtensionContext): void {
  const { db, tables } = ctx;

  // Collection routes BEFORE :id routes (to avoid param capture)
  app.get('/api/invoices', requirePermission('invoice:read'), async (c) => { ... });
  app.post('/api/invoices', requirePermission('invoice:create'), async (c) => { ... });
  app.post('/api/invoices/import', requirePermission('invoice:create'), async (c) => { ... });
  app.get('/api/invoices/export', requirePermission('invoice:read'), async (c) => { ... });

  // Instance routes
  app.get('/api/invoices/:id', requirePermission('invoice:read'), async (c) => { ... });
  app.put('/api/invoices/:id', requirePermission('invoice:update'), async (c) => { ... });
  app.delete('/api/invoices/:id', requirePermission('invoice:delete'), async (c) => { ... });
}
```

### Commands Pattern

```typescript
export function getInvoicingCommands(ctx: CommandContext): Command[] {
  const { db, tables, output, outputError, requirePermission } = ctx;
  const cmd = new Command('invoice').description('Manage invoices');

  cmd.command('add').description('Create an invoice')
    .option('--input-json <json>', 'JSON input')
    .action(async (opts) => {
      await requirePermission('invoice:create');
      const input = parseInputJson(opts.inputJson);
      output(await invoiceService.add(db, tables, input));
    });

  // list, get, update, remove, import, export ...

  return [new Command('invoicing').description('Invoicing').addCommand(cmd)];
}
```

### Migration Pattern

Migrations are versioned and ordered. Each migration has both SQLite and Postgres SQL:

```typescript
migrations: [
  {
    version: 1,
    description: 'Initial invoices table',
    sqlite: `CREATE TABLE IF NOT EXISTS invoices (
      id TEXT PRIMARY KEY,
      number TEXT NOT NULL,
      amount INTEGER NOT NULL,
      ...
    );`,
    postgres: `CREATE TABLE IF NOT EXISTS invoices (
      id TEXT PRIMARY KEY,
      number TEXT NOT NULL,
      amount INTEGER NOT NULL,
      ...
    );`,
  },
  {
    version: 2,
    description: 'Add line_items table',
    sqlite: `CREATE TABLE IF NOT EXISTS invoice_line_items (...);`,
    postgres: `CREATE TABLE IF NOT EXISTS invoice_line_items (...);`,
  },
],
```

When adding new tables or columns to an existing extension, add a NEW migration with an incremented version number. Never modify existing migrations.

## Adding Features to an Existing Extension

To add a new entity to an existing custom extension:

1. Add the table definition to `tables/sqlite.ts` and `tables/postgres.ts`
2. Add schemas to `schemas/index.ts`
3. Create a new service file in `services/`
4. Add routes to `routes.ts`
5. Add CLI commands to `commands.ts`
6. Add a new migration to the extension's `migrations` array
7. Add permissions for the new entity

## Events Integration

To emit events when entities change:

```typescript
import { eventService } from '@openclam/events';

// Inside your service function, after a successful insert/update/delete:
await eventService.emitEvent(db, tables, {
  source: 'invoicing',           // Your extension name
  eventType: 'invoice.created',  // entity.action
  entityType: 'invoice',
  entityId: parsed.id,
  userId: ctx?.userId,
  payload: { id: parsed.id, number: parsed.number },
});
```

## Dependencies

If your extension depends on another extension's data (e.g., reading persons from contacts):

```typescript
export const extension: Extension = {
  name: 'invoicing',
  dependencies: ['contacts'],  // Ensures contacts is enabled first
  // ...
};
```

You can then read from contacts tables (e.g., `tables.persons`) in your services. The dependency declaration ensures the tables exist.

## Checklist

When building or modifying a custom extension, verify:

- [ ] Extension exports a valid `Extension` object from `src/index.ts`
- [ ] Both SQLite and Postgres table definitions exist
- [ ] Migrations match table definitions
- [ ] Schemas have insert, update, and output variants
- [ ] Services follow the `(db, tables, ...)` pattern
- [ ] Routes place collection endpoints before `:id` params
- [ ] Permissions are declared for all entity operations
- [ ] Import/export functions exist for user-facing entities
- [ ] CLI commands follow the `--input-json` pattern
