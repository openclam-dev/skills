---
name: patterns
description: >
  Cross-cutting architecture patterns for the OpenClam monorepo. Covers extensions,
  services, table design, event/rule engine, workspaces, and CLI/API surfaces.
metadata:
  tags:
    - architecture
    - extensions
    - services
    - tables
    - events
    - workspaces
    - surfaces
---

# Cross-Cutting Patterns

---

## 1. Extension System

Extensions are the primary mechanism for adding domain functionality. Each extension declares tables, migrations, permissions, actions, API routes, and CLI commands.

### Extension Interface

Defined in `packages/core/src/extension/types.ts`:

```typescript
interface Extension {
  name: string;                   // unique id: "contacts", "tasks"
  label: string;                  // human label: "Contacts & CRM"
  version: string;                // semver
  dependencies: string[];         // other extension names
  tables: {
    sqlite: Record<string, any>;
    postgres: Record<string, any>;
  };
  migrations: ExtensionMigration[];
  permissions?: Record<string, Record<string, string>>;  // resource -> action -> desc
  actions?: Record<string, ExtensionAction>;
  registerRoutes?: (app: Hono, ctx: ExtensionContext) => void;
  getCommands?: (ctx: CommandContext) => Command[];
}

interface ExtensionMigration {
  version: number;
  description: string;
  sqlite: string;   // raw SQL
  postgres: string;  // raw SQL
}

interface ExtensionAction {
  description: string;
  paramsSchema: z.ZodType<any>;
  execute: (event: Event, params: any, db: any, tables: any, ctx?: { userId?: string | null }) => Promise<void>;
}
```

### Registration

Extensions register at startup via `registerBundledExtension(ext)` in `packages/core/src/extension/registry.ts`. All bundled extensions are registered in `apps/cli/src/utils/db.ts` → `ensureBundledExtensions()`:

```typescript
async function ensureBundledExtensions(): Promise<void> {
  const { contactsExtension } = await import('@openclam/contacts');
  const { tasksExtension } = await import('@openclam/tasks');
  const { messagingExtension } = await import('@openclam/messaging');
  const { marketingExtension } = await import('@openclam/marketing');
  const { contentExtension } = await import('@openclam/content');

  registerBundledExtension(contactsExtension);
  registerBundledExtension(tasksExtension);
  registerBundledExtension(messagingExtension);
  registerBundledExtension(marketingExtension);
  registerBundledExtension(contentExtension);

  registerCoreActions();
}
```

### Resolution

`resolveExtensions(enabledNames)` validates dependencies, detects cycles, and returns extensions in topological order.

### Table Merging

`mergeTables(coreTables, extensions, dialect)` combines core adapter tables with extension tables into one flat object. Services reference tables by key (e.g., `tables.persons`, `tables.contentItems`).

### Migrations

`initExtensionTables(db, extensions, dialect)` runs each extension's raw SQL migrations. Statements use `CREATE TABLE IF NOT EXISTS` (idempotent). When adding tables to an existing extension, add a new migration entry — NEVER modify existing entries.

### Permissions

Extensions declare permissions as `resource -> action -> description`. Key functions:

| Function | Purpose |
|----------|---------|
| `gatherPermissions(extensions)` | Collect permission groups from core + extensions |
| `flattenPermissions(groups)` | Flat list of `"resource:action"` keys |
| `permissionMatches(granted, required)` | Check with wildcard support (`"*"`, `"resource:*"`) |
| `checkPermissionCollisions(newExt, enabledExts)` | Validate no duplicate keys |

### Collision Checking

| Function | Purpose |
|----------|---------|
| `checkTableCollisions(newExt, enabledExts)` | Detect table name conflicts |
| `checkPermissionCollisions(newExt, enabledExts)` | Detect duplicate permission keys |
| `canDisable(name, enabledNames)` | Check if other extensions depend on this one |
| `getRequiredDependencies(name, alreadyEnabled)` | Transitive deps needed to enable |

### Creating a New Extension

1. Create package under `packages/<name>/` with `package.json`, `tsconfig.json`, `eslint.config.mjs`
2. Define tables in `src/tables/sqlite.ts` and `src/tables/postgres.ts` (Drizzle dialect-specific)
3. Define Zod schemas in `src/schemas/` (insert, update, select)
4. Implement services in `src/services/` following the service conventions below
5. Create `extension.ts` exporting the Extension object
6. Register in `ensureBundledExtensions()` in `apps/cli/src/utils/db.ts`
7. Add CLI commands and API routes (see Surface Conventions)
8. Run: `pnpm turbo check-types && pnpm turbo test`

### Key Files

- `packages/core/src/extension/types.ts` — Extension, ExtensionAction, ExtensionMigration interfaces
- `packages/core/src/extension/registry.ts` — Registration, resolution, merging, permissions, actions

---

## 2. Service Conventions

Services are the domain logic layer. CLI commands and API routes are thin wrappers.

### Function Signature

```typescript
async function operation(
  db: any,           // Drizzle instance (dialect-agnostic)
  tables: any,       // merged table definitions
  input: InputType,
  pagination?: { cursor?: string; limit?: number },
  options?: { expand?: string[] },
): Promise<ResultType>
```

### CRUD Pattern

Every entity implements these standard operations:

| Operation | Name | Returns |
|-----------|------|---------|
| Create | `add(db, tables, input, extras?, ctx?)` | Created entity |
| List | `list(db, tables, filters?, pagination?, options?)` | Paginated page |
| Get | `get(db, tables, id, options?)` | Single entity (throws if not found) |
| Update | `update(db, tables, id, input, ctx?)` | Updated entity |
| Soft delete | `remove(db, tables, id, ctx?)` | void — sets `deleted_at` |
| Restore | `restore(db, tables, id, ctx?)` | void — clears `deleted_at` |
| Hard delete | `purge(db, tables, id, ctx?)` | void — permanent, irreversible |

### Pagination

All `list` operations return a **page object**, not a raw array:

```typescript
interface Page<T> {
  data: T[];              // the items for this page
  nextCursor: string | null;  // pass as cursor for the next page; null = no more pages
  hasMore: boolean;       // true if there is at least one more page
}
```

Callers pass `{ cursor, limit }` in the pagination argument. `cursor` is an opaque string returned by the previous call's `nextCursor`. `limit` defaults vary by service (typically 25–50).

### Validation

Every mutation MUST validate input with Zod before writing:

```typescript
const parsed = insertPersonSchema.parse(input);  // throws ZodError on invalid
await db.insert(tables.persons).values(parsed).run();
```

- `insertXxxSchema` — auto-generates `id` (UUIDv4), `created_at`, `updated_at`
- `updateXxxSchema` — auto-sets `updated_at`

Schemas live in each extension's `src/schemas/` directory.

### Event Emission

After every successful mutation, emit an event:

```typescript
await eventService.emitEvent(db, tables, {
  source: 'contacts',
  eventType: 'person.created',
  entityType: 'person',
  entityId: parsed.id,
  userId: ctx?.userId,
});
```

### Entity Expansion

List and get operations support `options.expand` for related data:

```typescript
const VALID_EXPANDS = ['emails', 'phones', 'addresses', 'org', 'deals', 'tags', 'notes', 'relationships', 'log'];
```

ALWAYS validate expand options against a whitelist.

### Soft Delete Pattern

Primary entities use `deleted_at TEXT` for soft deletion:

- `remove()` — sets `deleted_at = now()`, emits `<entity>.deleted` event
- `restore()` — clears `deleted_at`, emits `<entity>.restored` event. Throws if entity is not deleted.
- `purge()` — permanently deletes the row + cascades. Irreversible. Emits `<entity>.purged` event.
- All list/get queries filter with `isNull(table.deleted_at)` by default

### Error Handling

Services throw descriptive errors: `"Person not found: <id>"`, `"Transport already exists"`, etc. Surfaces catch and format. Services NEVER catch and swallow errors.

### Rules

- ALWAYS validate input with Zod before DB writes
- ALWAYS emit events after successful mutations
- ALWAYS pass `ctx` with `userId` when available
- ALWAYS use soft deletes for primary entities; filter with `isNull(table.deleted_at)`
- NEVER put domain logic in CLI commands or API handlers
- NEVER access `db` or `tables` from outside the service layer

### Key Files

- `packages/contacts/src/services/person.ts` — Canonical CRUD + expand + event example
- `packages/core/src/services/event.ts` — Event emission and outbox processing

---

## 3. CLI Patterns

### The --json and --markdown flags

ALWAYS place `--json` or `--markdown` immediately after `clam`. `--json` switches all output to compact JSON (no colors, no formatting). `--markdown` renders lists as markdown tables and single objects as key-value blocks (lower token count, better for display). Use `--json` when parsing values programmatically; use `--markdown` when presenting results to the user.

```bash
clam --json contacts person list
clam --markdown contacts person list
```

### The --input-json pattern

ALL commands that accept input (add, update, get, list, and most others) accept `--input-json '<json>'`. Pass all fields as a single JSON object. Individual named flags also exist but `--input-json` overrides them.

```bash
# Pass all fields as JSON
clam --json contacts person add --input-json '{"first_name":"Alice","last_name":"Smith"}'
clam --json contacts person list --input-json '{"search":"alice","limit":10}'
clam --json deals deal get --input-json '{"id":"<uuid>"}'
```

Commands resolve values as: `input.<field> ?? posArg ?? opts.<flag>`. This means `--input-json` takes precedence over positional args, which take precedence over individual flags.

### Positional args as alternatives

All commands that previously required positional args now accept them as **optional** positionals OR via `--input-json`. Both forms work:

```bash
# Positional form
clam --json contacts person get <person-id>

# --input-json form (preferred for agents)
clam --json contacts person get --input-json '{"id":"<person-id>"}'
```

### remove / restore / purge

These commands accept **either** a positional arg OR `--input-json`. Both work:

```bash
# Positional (concise)
clam --json contacts person remove <person-id>
clam --json contacts person restore <person-id>
clam --json contacts person purge <person-id>

# --input-json (explicit)
clam --json contacts person remove --input-json '{"id":"<person-id>"}'
clam --json contacts person restore --input-json '{"id":"<person-id>"}'
clam --json contacts person purge --input-json '{"id":"<person-id>"}'
```

Note: For `role`, `rule`, `project` the identifier is a `slug` or `id` depending on the entity. Check the domain reference for the exact key name.

### Pagination pattern

All `list` commands return a paginated response:

```json
{
  "data": [ ... ],
  "nextCursor": "cursor-string-or-null",
  "hasMore": true
}
```

To paginate, pass `cursor` and optionally `limit` in `--input-json`:

```bash
# First page
clam --json contacts person list --input-json '{"limit":25}'

# Next page (use nextCursor from previous response)
clam --json contacts person list --input-json '{"cursor":"<nextCursor>","limit":25}'
```

Stop fetching when `hasMore` is `false` or `nextCursor` is `null`.

**Pagination fields in list --input-json:**

```typescript
interface PaginationInput {
  cursor?: string;   // opaque cursor from previous nextCursor
  limit?: number;    // max items per page (default varies: 25–50)
}
```

### Standard workflow

```bash
# 1. Always use --json
# 2. Pass data via --input-json
# 3. Parse output as JSON to extract IDs for follow-up commands

PERSON=$(clam --json contacts person add --input-json '{"first_name":"Alice","last_name":"Smith"}')
PERSON_ID=$(echo "$PERSON" | jq -r '.id')

# 4. Use cursor pagination for list commands
PAGE=$(clam --json contacts person list --input-json '{"limit":50}')
NEXT=$(echo "$PAGE" | jq -r '.nextCursor')
HAS_MORE=$(echo "$PAGE" | jq -r '.hasMore')
```

---

## 4. Table Design

Drizzle ORM with dialect-specific definitions. Dedicated per-entity tables (not polymorphic).

### Dedicated Per-Entity Tables

Each primary entity has its own sub-tables:

```
persons
  person_emails, person_phones, person_addresses, person_urls
  person_notes, person_tags, person_comments

orgs
  org_emails, org_phones, org_addresses, org_notes, org_tags

deals
  deal_notes, deal_tags, deal_comments

content_items
  content_versions, content_assets, content_tags, content_notes
  content_comments, content_channels, content_relations, content_locales
```

Why not polymorphic: FK constraints enforce integrity, CASCADE deletes auto-cleanup, full type safety in Drizzle queries, no runtime `entity_type` filtering.

### Dual Adapter Pattern

Every table MUST be defined in both dialect files:

```
packages/<extension>/src/tables/sqlite.ts   — drizzle-orm/sqlite-core
packages/<extension>/src/tables/postgres.ts — drizzle-orm/pg-core
```

Key dialect differences: `INTEGER` vs `BOOLEAN`, `REAL` vs `DOUBLE PRECISION`.

At runtime, `mergeTables()` selects the correct dialect tables.

### Column Conventions

| Column | Type | Rule |
|--------|------|------|
| `id` | TEXT PRIMARY KEY | UUIDv4, generated in insert schema |
| `created_at` | TEXT NOT NULL | ISO datetime, set on insert |
| `updated_at` | TEXT NOT NULL | ISO datetime, updated on every mutation |
| `deleted_at` | TEXT | NULL = active, ISO datetime = soft-deleted |
| FKs | TEXT NOT NULL | `REFERENCES parent(id) ON DELETE CASCADE` |

### Core Tables

Defined in `packages/adapter-sqlite/` and `packages/adapter-postgres/`:

`users`, `log`, `config`, `events`, `rules`, `rule_actions`, `projects`, `roles`, `role_permissions`, `user_roles`

### Extension Tables

| Extension | Tables |
|-----------|--------|
| contacts | persons, person_emails, person_phones, person_addresses, person_urls, person_notes, person_tags, person_comments, orgs, org_emails, org_phones, org_addresses, org_notes, org_tags, relationships, segments, project_persons, project_orgs |
| deals | pipelines, pipeline_stages, deals, deal_notes, deal_tags, deal_comments, project_deals |
| messaging | transports, messages |
| marketing | sequences, sequence_steps, enrollments, sequence_comments |
| tasks | tasks, task_comments, project_tasks |
| content | content_types, content_items, content_versions, assets, content_assets, collections, collection_items, content_tags, content_notes, content_comments, content_channels, content_relations, content_locales, project_content |
| calendar | calendars, calendar_events, calendar_event_attendees, calendar_event_comments, calendar_event_tags, project_calendar_events |
| research | research_topics, research_topic_persons, research_topic_orgs, research_runs, research_sources, research_findings, research_briefings, research_briefing_insights, research_briefing_insight_findings |
| storage | storage_buckets, storage_files, storage_bucket_permissions |
| cron | cron_jobs, cron_runs |
| webhooks | event_subscribers, event_deliveries |
| workflows | workflow_definitions, workflow_runs, workflow_step_attempts, workflow_event_waiters |

### FTS / Search

**SQLite**: FTS5 virtual tables (`persons_fts`, `orgs_fts`, `deals_fts`, etc.) with auto-sync triggers. Created by `createFtsTables(db)` in `packages/adapter-sqlite/src/fts.ts`.

**Postgres**: Generated `tsvector` columns with GIN indexes. Created by `createSearchIndex(db)` in `packages/adapter-postgres/src/search.ts`. No manual rebuild needed.

Both adapters export `search(db, query)` returning `SearchResult[]`, with LIKE/ILIKE fallback.

### Migration Pattern

```typescript
migrations: [
  {
    version: 1,
    description: 'Initial tables',
    sqlite: `CREATE TABLE IF NOT EXISTS persons (...);`,
    postgres: `CREATE TABLE IF NOT EXISTS persons (...);`,
  },
]
```

`initExtensionTables()` splits on `;` and executes each statement. Add new entries for new tables — NEVER modify existing migration entries.

### Rules

- ALWAYS define tables for both `sqlite` and `postgres`
- ALWAYS use `ON DELETE CASCADE` for entity sub-tables
- NEVER use polymorphic `entity_type` / `entity_id` patterns
- ALWAYS use `TEXT PRIMARY KEY` with UUIDv4
- ALWAYS include `created_at`, `updated_at` on mutable entities; `deleted_at` on soft-deletable entities

### Key Files

- `packages/contacts/src/tables/sqlite.ts` / `postgres.ts` — Contacts tables
- `packages/content/src/tables/sqlite.ts` / `postgres.ts` — Content tables
- `packages/adapter-sqlite/src/fts.ts` — FTS5 DDL and triggers
- `packages/adapter-sqlite/src/search.ts` — SQLite search
- `packages/adapter-postgres/src/search.ts` — Postgres search

---

## 5. Event & Rule Engine

Event outbox pattern for automation. Services emit events; events are stored as `pending` and processed in batch against active rules.

### Event Schema (`events` table)

| Column | Type | Description |
|--------|------|-------------|
| `id` | TEXT PK | UUIDv4 |
| `source` | TEXT | Origin, e.g. `"sor/contacts"` |
| `event_type` | TEXT | `"person.created"`, `"task.completed"`, etc. |
| `entity_type` | TEXT | `"person"`, `"deal"`, `"task"`, etc. |
| `entity_id` | TEXT | UUID of affected entity |
| `payload` | TEXT | Optional JSON |
| `user_id` | TEXT | Who caused the event (null for system) |
| `status` | TEXT | `pending` / `processing` / `completed` / `failed` |
| `created_at` | TEXT | ISO datetime |
| `processed_at` | TEXT | Set when completed |
| `failure_reason` | TEXT | Error message if failed |
| `attempts` | INTEGER | Processing attempt count |

### Emission

ALWAYS use the convenience wrapper in service code:

```typescript
await eventService.emitEvent(db, tables, {
  source: 'contacts',           // auto-prefixed to "sor/contacts"
  eventType: 'person.created',
  entityType: 'person',
  entityId: parsed.id,
  userId: ctx?.userId,
  payload: { stage: 'qualified' },  // optional, auto-stringified
});
```

### Outbox Processing

Events are NOT processed synchronously. Processing is triggered explicitly:

- CLI: `clam event process`
- API: `POST /api/events/process`

Flow: fetch pending/failed events -> for each: mark processing -> run in-process handlers -> match rules -> execute actions -> mark completed/failed.

### Pattern Matching

`matchPattern(pattern, eventType)`:

| Pattern | Matches |
|---------|---------|
| `"person.created"` | Exact match |
| `"person.*"` | Any `"person.X"` event |
| `"*"` | All events |

### Rule Structure

**`rules` table**: `id`, `slug` (unique), `name`, `event_pattern`, `conditions` (JSON), `priority`, `is_active`

**`rule_actions` table**: `id`, `rule_id` (FK), `extension`, `action`, `params_json`, `execution_order`

### In-Process Handlers

For lightweight reactive logic without database rules:

```typescript
eventService.on('person.created', async (event, db, tables) => { ... });
eventService.off('person.created', handler);
```

Handlers run during `processEvents()`, not at emit time.

### Built-in Actions

| Action | Description | Params |
|--------|-------------|--------|
| `core:tag-entity` | Apply tag to event's entity | `{ tag: string }` |
| `core:create-log` | Create log entry for entity | `{ message?, type? }` |
| `contacts:add-note` | Add note to event's entity | `{ title?, body: string }` |
| `contacts:update-deal-stage` | Move deal to new stage | `{ stage: string }` |
| `contacts:assign-to-org` | Assign person to org | `{ org_id: string }` |

### Writing New Actions

```typescript
const myAction: ExtensionAction = {
  description: 'Do something useful',
  paramsSchema: z.object({ someParam: z.string() }),
  execute: async (event, params, db, tables, ctx) => {
    const { myService } = await import('./services/index.js');  // dynamic import
    await myService.doSomething(db, tables, event.entity_id, params.someParam);
  },
};
```

Add to the extension's `actions` property. Registered automatically via `registerBundledExtension()`.

### Rules

- ALWAYS emit events after successful mutations
- NEVER process events synchronously in the request path
- ALWAYS handle action failures gracefully
- ALWAYS use dynamic imports in action `execute()` to avoid circular deps
- NEVER modify events during processing — they are immutable

### Key Files

- `packages/core/src/services/event.ts` — Emission, outbox processing, handler registry
- `packages/core/src/actions/core-actions.ts` — Built-in `tag-entity` and `create-log`
- `packages/contacts/src/actions.ts` — Contacts extension actions

---

## 6. Workspaces

A workspace is an isolated data context. Each workspace has one or more environments, each with its own DB connection and enabled extensions.

### Filesystem Layout

```
~/.openclam/
  openclam.json                        # GlobalConfig v2
  workspaces/
    <workspace-slug>/
      manifest.json                    # WorkspaceManifest
      envs/
        <env-slug>/
          data.db                      # SQLite database (local)
```

Path helpers in `packages/core/src/connection/paths.ts`: `getClamHome()`, `getConfigPath()`, `getWorkspacesDir()`, `getWorkspaceDir(slug)`, `getWorkspaceManifestPath(slug)`, `getEnvironmentDir(wsSlug, envSlug)`, `getDefaultDbPath(wsSlug, envSlug)`.

### GlobalConfig v2

Stored in `~/.openclam/openclam.json`:

```typescript
interface GlobalConfig {
  version: 2;
  activeWorkspace: string | null;
  activeUser: string | null;
  users: Record<string, UserEntry>;  // keyed by identifier
}

interface UserEntry {
  name: string;
  type: 'human' | 'agent';
  createdAt: string;
}
```

Workspaces are NOT stored inline. Each has its own `manifest.json`.

### WorkspaceManifest

Stored in `~/.openclam/workspaces/<slug>/manifest.json`:

```typescript
interface WorkspaceManifest {
  name: string;
  activeEnvironment: string | null;
  environments: Record<string, EnvironmentEntry>;
  createdAt: string;
}

interface EnvironmentEntry {
  name: string;
  connection: Connection;
  extensions: string[];
  createdAt: string;
}
```

### Connection Types

| Dialect | Mode | Provider | Fields |
|---------|------|----------|--------|
| sqlite | local | local | `path` |
| sqlite | remote | turso, d1 | `url`, `authToken?` |
| postgres | local | local | `url` |
| postgres | remote | neon | `url` |
| mysql | local | local | `url` |
| mysql | remote | planetscale | `url` |

### Workspace Operations (`packages/core/src/connection/workspace.ts`)

| Function | Purpose |
|----------|---------|
| `addWorkspace(slug, name, envSlug, envName, connection, extensions)` | Create with initial env |
| `removeWorkspace(slug)` | Delete workspace dir + update config |
| `renameWorkspace(oldSlug, newSlug, newName?)` | Rename/move |
| `setActiveWorkspace(slug)` | Set active in config |
| `getActiveWorkspace()` | Returns active with resolved env |
| `listWorkspaces()` | List all with metadata |

### Environment Operations

| Function | Purpose |
|----------|---------|
| `addEnvironment(wsSlug, envSlug, envName, connection, extensions)` | Add env to workspace |
| `removeEnvironment(wsSlug, envSlug)` | Remove (cannot remove last) |
| `setActiveEnvironment(wsSlug, envSlug)` | Switch active env |
| `enableExtension(wsSlug, extName, envSlug?)` | Enable extension |
| `disableExtension(wsSlug, extName, envSlug?)` | Disable extension |

### Clone & Fork

- `cloneEnvironment(wsSlug, sourceEnv, targetEnv, targetName, { withData })` — Clone env within workspace
- `forkWorkspace(sourceSlug, targetSlug, targetName, { withData })` — New workspace from existing
- `diffEnvironments(wsSlug, envA, envB)` — Compare two envs

### Environment Override

CLI supports `--env <env>` on any command. A `preAction` hook in `apps/cli/src/index.ts` calls `setEnvOverride(opts.env)`, and `getDbContext()` checks the override before resolving.

### Authentication

Auth is resolved in `requireUser()` (`apps/cli/src/utils/db.ts`) in priority order:

1. **`--token` flag** — API key passed per-command
2. **`OPENCLAM_TOKEN` env var** — API key set for the session/process
3. **`activeUser` from config** — human login session (via `clam login`). Syncs the config user into the workspace DB if needed.

API key auth uses `apiKeyService.authenticateApiKey()` which looks up the key by prefix, verifies the SHA-256 hash with timing-safe comparison, and returns the associated userId.

For HTTP API access: `Authorization: Bearer <api-key>` header. A single header — the key identifies the user.

Agent users get an API key auto-created in the `api_keys` table at creation time (`type: "agent"` on `user add`). The plaintext is returned once. Keys can be rotated via `user rotate-key`.

### v1 to v2 Migration

Automatic on `loadConfig()`. When config has `workspaces` record but no `version` field, each inline workspace becomes a `manifest.json` with a `default` environment. Legacy flat DB files are moved to `envs/default/data.db`.

### Rules

- ALWAYS use `addWorkspace()` / `addEnvironment()` — not raw filesystem ops
- NEVER modify `openclam.json` or `manifest.json` directly — use load/save helpers
- ALWAYS validate slugs with `SlugSchema` (lowercase, starts with letter, hyphens only)

### Key Files

- `packages/core/src/connection/types.ts` — Type definitions and Zod schemas
- `packages/core/src/connection/workspace.ts` — Workspace/environment CRUD and clone/fork
- `packages/core/src/connection/paths.ts` — Filesystem path helpers

---

## 7. Surface Conventions

Two surfaces: CLI (Commander.js) and API (Hono). Both are thin wrappers over services. Every service operation MUST be reachable from both.

### CLI Pattern

```typescript
// apps/cli/src/commands/person.ts
import { Command } from 'commander';
import { getDbContext } from '../utils/db.js';

export const personCommand = new Command('person').description('Manage contacts');

personCommand
  .command('add [name]')
  .option('--input-json <json>', 'JSON input (overrides flags)')
  .action(async (nameArg, opts) => {
    const { db, tables } = await getDbContext();
    const { personService } = await import('@openclam/contacts');
    const input = opts.inputJson ? JSON.parse(opts.inputJson) : {};
    const firstName = input.first_name ?? nameArg;
    const person = await personService.add(db, tables, { first_name: firstName, ... });
    output(person);
  });
```

Conventions:
- Each command file exports a single `Command`
- Use `getDbContext()` for `{ db, tables, dialect, extensions }`
- Use dynamic imports for service modules
- `--env <env>` flag overrides active environment (preAction hook)
- `--token <api-key>` or `OPENCLAM_TOKEN` env var for agent auth (preAction hook)
- `requireUser(db, tables)` checks token/env var first, then falls back to active user from config

### API Pattern

```typescript
// Core routes in packages/api/src/index.ts
app.post('/api/users', async (c) => {
  const { userService } = await import('@openclam/core');
  return c.json(await userService.add(db, tables, await c.req.json()), 201);
});

// Extension routes via extension.registerRoutes(app, ctx)
registerRoutes(app: Hono, ctx: ExtensionContext) {
  const { db, tables } = ctx;
  app.get('/api/persons', async (c) => { ... });
}
```

Conventions:
- `createApp(ctx)` receives `{ db, tables, extensions }` already resolved
- Dynamic imports for services in route handlers
- Status codes: `200` success, `201` creation, `c.json({ ok: true })` for void ops

### REST Verb Mapping

| Verb | Purpose | Example |
|------|---------|---------|
| GET | List / get | `GET /api/persons`, `GET /api/persons/:id` |
| POST | Create | `POST /api/persons` |
| PUT | Update | `PUT /api/persons/:id` |
| DELETE | Soft delete | `DELETE /api/persons/:id` |
| PUT | Restore | `PUT /api/persons/:id/restore` |
| DELETE | Purge | `DELETE /api/persons/:id/purge` |
| PUT | Status actions | `PUT /api/rules/:slug/enable` |
| POST | Special ops | `POST /api/persons/merge`, `POST /api/events/process` |

### Route Ordering

CRITICAL: Static paths MUST come before parameterized paths:

```typescript
app.post('/api/persons/merge', ...);   // static first
app.get('/api/persons/:id', ...);      // parameterized last
```

### Adding a New Feature End-to-End

1. **Schema** — Zod schemas in `packages/<ext>/src/schemas/`
2. **Table** — Drizzle definitions in `src/tables/sqlite.ts` + `postgres.ts`
3. **Migration** — `CREATE TABLE IF NOT EXISTS` in extension `migrations` (both dialects)
4. **Service** — CRUD functions in `src/services/`
5. **Events** — Register types, emit from mutations
6. **CLI command** — Command file in `apps/cli/src/commands/`, register in `apps/cli/src/index.ts`
7. **API route** — Routes in extension's `registerRoutes()`
8. **Export** — Service from `src/services/index.ts`
9. **Check** — `pnpm turbo check-types && pnpm turbo test`

### Rules

- ALWAYS add both CLI command and API route for new features
- ALWAYS use `getDbContext()` in CLI commands
- ALWAYS use dynamic imports for services in handlers
- ALWAYS place static routes before parameterized routes
- NEVER put domain logic in surfaces — keep them thin
- NEVER access the database directly from a surface

### Key Files

- `apps/cli/src/index.ts` — CLI command registration
- `apps/cli/src/utils/db.ts` — `getDbContext()`, `requireUser()`, extension registration
- `packages/api/src/index.ts` — Core API routes and extension route registration

---

## Package Map

### Apps

| Package | Description |
|---------|-------------|
| `apps/cli` | Commander.js CLI application; thin wrapper over services |
| `apps/daemon` | Long-running background service (HTTP + cron + file watching) |
| `apps/docs` | Documentation site (Fumadocs + Vite + React) |

### System of Record (domain data, CRUD, schemas, tables)

| Package | Description |
|---------|-------------|
| `packages/core` | Always loaded: workspaces, users, projects, roles, tags, rules, logs, config, extension system |
| `packages/contacts` | People, organizations, relationships, segments |
| `packages/deals` | Pipelines, stages, deals, forecasting |
| `packages/tasks` | Task management with subtasks and assignees |
| `packages/messaging` | Message records, transport configs, delivery logs |
| `packages/marketing` | Sequences, steps, enrollments |
| `packages/content` | Versioned CMS: content types, items, versions, assets, collections, channels, locales |
| `packages/calendar` | Calendars, events, attendees, recurrence |
| `packages/research` | Research topics, runs, sources, findings, briefings |
| `packages/storage` | Buckets and file metadata |

### Execution Layer (state mutation, event processing, orchestration)

| Package | Description |
|---------|-------------|
| `packages/events` | Event outbox — emission, processing, dispatch to subscribers |
| `packages/rules` | Rule engine — evaluates conditions, triggers actions on events |
| `packages/actions` | Action registry — executes domain actions from rules/workflows |
| `packages/workflows` | Workflow definitions, step orchestration, retries, async state |
| `packages/cron` | Job scheduling and execution |
| `packages/webhooks` | Subscriber management, async delivery with retries |

### Supporting Packages

| Package | Description |
|---------|-------------|
| `packages/api` | Hono HTTP app factory; core + extension REST routes |
| `packages/runtime` | Workspace bootstrap: connects DB, resolves extensions, initializes tables |
| `packages/vault` | Encrypted secret store (AES-256-GCM, file-based) |
| `packages/ai` | LLM provider abstraction (OpenAI, Anthropic, Ollama, OpenRouter, local) |
| `packages/providers` | External service transports (Resend, AWS SES) |
| `packages/schemas` | Unified re-export of all domain Zod schemas |
| `packages/ui` | React component library (Base UI + Tailwind) |
| `packages/adapter-sqlite` | SQLite adapter: tables, FTS5 search, migrations |
| `packages/adapter-postgres` | Postgres adapter: tables, tsvector search, migrations |
| `packages/provider-sqlite-local` | Local SQLite connection factory (LibSQL) |
| `packages/provider-postgres-local` | Local Postgres connection factory |
| `packages/provider-sqlite-d1` | Cloudflare D1 connection factory |
| `packages/storage-s3` | S3 storage backend |
