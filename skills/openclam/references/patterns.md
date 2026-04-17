---
name: patterns
description: >
  Cross-cutting architecture patterns for OpenClam — service conventions,
  pagination, soft-delete, event emission, Zod validation, table design,
  workspace layout, auth resolution, and surface wrappers. Load this when
  authoring an extension or investigating how something is wired up.
metadata:
  author: openclam
  version: 0.1.0
tags:
  - architecture
  - extensions
  - services
  - tables
  - events
  - workspaces
  - surfaces
---

# Cross-cutting patterns

Patterns here are verified against the current codebase. Key sources:

- `packages/extension-sdk/src/define.ts` — `defineProcedure`, `defineService`,
  `defineExtension`.
- `packages/extension-sdk/src/loader.ts` — how procedures mount under
  `/api/{id}/...`, how `ProcedureContext` is constructed.
- `packages/core/src/services/user.ts` — canonical CRUD + soft-delete +
  event emission.
- `packages/core/src/procedures/users.ts` — thin procedure wrappers over
  services.
- `packages/core/src/schemas/user.ts`, `query.ts` — Zod patterns.
- `packages/schemas/src/pagination.ts` — cursor-based pagination.
- `packages/core/src/connection/types.ts`, `paths.ts` — workspace layout.
- `packages/events/src/service.ts` — `emitEvent` signature.
- `apps/cli/src/utils/db.ts` → `requireUser` — auth resolution.

If you need to translate a pattern onto a specific invocation surface, see
`surfaces.md`.

---

## 1. Service conventions

Services are the domain logic layer. Procedures and CLI commands are thin
wrappers that validate input, call a service, and format output.

### Function signature

Every service function takes `db` and `tables` first, then domain input,
then (for mutations) a `MutationContext`. `list` variants also take
`pagination` and `options`. This is stable across every service:

```ts
async function list(
  db: OpenClamDb,
  tables: OpenClamTables,
  filters: { type?: string; active?: boolean } = {},
  pagination: PaginationInput = {},
  options: { fields?: string[] } = {},
): Promise<PaginatedResult<User>>
```

```ts
async function add(
  db: OpenClamDb,
  tables: OpenClamTables,
  input: Omit<InsertUser, 'id' | 'created_at' | 'updated_at'>,
  ctx?: MutationContext,   // { userId?, decision? }
): Promise<User & { api_key?: string }>
```

Services are **user-agnostic**. User identity flows in as `ctx.userId`.
This is what lets `defineService` factories live behind
`services` in `defineExtension` (`packages/extension-sdk/src/define.ts`) —
the factory only closes over `{ db, tables, dialect }`, not the caller.

### CRUD pattern

Every primary entity implements the same set of operations:

| Operation | Function | Semantics |
|-----------|----------|-----------|
| Create | `add(db, tables, input, ctx?)` | Parse insert schema, insert, emit `<entity>.created` |
| List | `list(db, tables, filters?, pagination?, options?)` | Returns `PaginatedResult<T>` |
| Get | `get(db, tables, idOrSlug, options?)` | Throws `<Entity> not found: <id>` |
| Update | `update(db, tables, id, input, ctx?)` | Parse update schema, set, emit `<entity>.updated` |
| Soft delete | `remove(db, tables, id, ctx?)` | Sets `deleted_at`, emits `<entity>.deleted` |
| Restore | `restore(db, tables, id, ctx?)` | Throws if not deleted, clears `deleted_at`, emits `<entity>.restored` |
| Purge | `purge(db, tables, id, ctx?)` | Hard delete + cascade, emits `<entity>.purged` |

Canonical example: `packages/core/src/services/user.ts`.

### Rules

- Validate every mutation input with Zod **before** writing.
- Emit an event after **every** successful mutation.
- Throw descriptive errors: `"User not found: <id>"`, `"Slug already in use"`.
- Never swallow errors in the service; let the procedure/CLI layer format.
- Never reach for `ctx.request` or Hono — services know nothing about HTTP.

---

## 2. Zod validation

Every domain has three schemas in `src/schemas/<entity>.ts`:

- `insert<Entity>Schema` — used on create. Auto-generates `id`
  (`z.uuid().default(() => uuidv4())`), `created_at`, `updated_at`, and
  defaults `deleted_at` to `null`.
- `update<Entity>Schema` — partial; always auto-sets `updated_at` so every
  update bumps the row.
- `<entity>Schema` — the select/output shape.

From `packages/core/src/schemas/user.ts`:

```ts
export const insertUserSchema = z.object({
  id: z.uuid().default(() => uuidv4()),
  name: z.string().min(1),
  slug: SlugSchema,
  type: UserTypeSchema,
  email: z.string().email().nullable().default(null),
  is_active: z.boolean().default(true),
  status: UserStatusSchema.default('active'),
  secret_hash: z.string().nullable().default(null),
  metadata: z.string().nullable().default(null),
  created_at: z.iso.datetime().default(() => new Date().toISOString()),
  updated_at: z.iso.datetime().default(() => new Date().toISOString()),
  deleted_at: z.iso.datetime().nullable().default(null),
});

export const updateUserSchema = z.object({
  // ...fields optional...
  updated_at: z.iso.datetime().default(() => new Date().toISOString()),
});
```

Usage in a service:

```ts
const parsed = insertUserSchema.parse(input);
await db.insert(tables.users).values(parsed);
```

Shared query primitives (`idParamSchema`, `paginationSchema`, `searchSchema`)
live in `@openclam/schemas` and are re-exported from
`packages/core/src/schemas/query.ts`. Domain-specific list schemas extend
those with filters:

```ts
export const listUsersQuerySchema = paginationSchema.extend({
  type: z.enum(['human', 'agent']).optional(),
  sort: z.enum(['created_at', 'name', 'type', 'identifier']).default('created_at').optional(),
});
```

---

## 3. Pagination

All `list` operations return a page object, never a raw array. Source of
truth: `packages/schemas/src/pagination.ts`.

```ts
export interface PaginationInput {
  cursor?: string | null;
  limit?: number;
  sort?: string;
  order?: 'asc' | 'desc';
  all?: boolean;
}

export interface PaginatedResult<T> {
  data: T[];
  nextCursor: string | null;
  hasMore: boolean;
}

export const DEFAULT_LIMIT = 50;
export const MAX_LIMIT = 200;
```

Helpers:

- `clampLimit(limit?)` — `[1, MAX_LIMIT]`, default `DEFAULT_LIMIT = 50`.
- `encodeCursor(sortValue, id)` — base64url of `{ v, i }`, where `v` is the
  sort-column value at the last row.
- `decodeCursor(cursor)` — returns `{ v, i }`.
- `buildCursorCondition({ cursor, sortCol, idCol, order })` — produces the
  keyset WHERE predicate for the active sort column + direction.
- `applyPagination(rows, limit, getSortValue)` — consumes rows fetched with
  `limit + 1`, returns a `PaginatedResult<T>`. `getSortValue` must read the
  same column used in `buildCursorCondition`'s `sortCol`, or pagination
  silently overlaps/skips rows.

Canonical list loop (from `userService.list`):

```ts
const limit = clampLimit(pagination.limit);
const conditions = [isNull(tables.users.deleted_at)];

const sortKey = pagination.sort && tables.users[pagination.sort] ? pagination.sort : 'created_at';
const sortCol = tables.users[sortKey];
const order: SortOrder = pagination.order ?? 'desc';
const sortDir = order === 'asc' ? asc : desc;

if (pagination.cursor) {
  conditions.push(buildCursorCondition({
    cursor: pagination.cursor,
    sortCol,
    idCol: tables.users.id,
    order,
  }));
}

const rows = await db
  .select()
  .from(tables.users)
  .where(and(...conditions))
  .orderBy(sortDir(sortCol), sortDir(tables.users.id))
  .limit(limit + 1);

return applyPagination(rows, limit, (r) => (r as Record<string, unknown>)[sortKey]);
```

Clients paginate by passing `nextCursor` back in as `cursor` and stopping
when `hasMore === false`. See `surfaces.md` for how this surfaces on each
transport.

---

## 4. Soft delete

Primary entities carry a nullable `deleted_at TEXT` column. `list` and
`get` filter with `isNull(table.deleted_at)` by default.

- `remove(db, tables, id, ctx?)`
  - Verifies the row exists; throws `"<Entity> not found: <id>"` otherwise.
  - Sets `deleted_at = now()` and `updated_at = now()`.
  - Emits `<entity>.deleted`.
- `restore(db, tables, id, ctx?)`
  - Throws if `deleted_at` is already `null` (`"<Entity> is not deleted"`).
  - Clears `deleted_at`, bumps `updated_at`.
  - Emits `<entity>.restored`.
- `purge(db, tables, id, ctx?)`
  - Deletes the row. `ON DELETE CASCADE` on sub-tables cleans up dependents.
  - Irreversible.
  - Emits `<entity>.purged`.

The CLI surface for these is positional-or-JSON:

```
clam --json <domain> <entity> remove  <id>
clam --json <domain> <entity> restore <id>
clam --json <domain> <entity> purge   <id>
```

(Both forms accepted; agents should prefer `--input-json '{"id":"..."}'`.)

---

## 5. Event emission

The canonical emitter is `emitEvent` from `@openclam/events`:

```ts
// packages/events/src/service.ts
export async function emitEvent(
  db: OpenClamDb,
  tables: OpenClamTables,
  opts: {
    source: string;
    module?: string | null;
    eventType: string;
    entityType: string;
    entityId: string;
    userId?: string | null;
    payload?: Record<string, unknown>;
    metadata?: Record<string, unknown> | null;
    capabilityContext?: CapabilityCheck;
  },
): Promise<Event | null>
```

Conventions observed across core services (see `userService.add`):

- `source` — coarse origin. Core uses `'openclam'`.
- `module` — sub-origin. Core uses `'core'`. Extensions use their `id`.
- `eventType` — `<entity>.<verb>`, e.g. `user.created`, `deal.moved`.
- `entityType` / `entityId` — the primary record affected.
- `userId` — from `ctx.userId`; null means system-initiated.
- `payload` — the post-state row for creates/updates, pre-state for deletes,
  plus `_before` for updates. Outbox consumers should not need to re-read.

Example from `userService.add`:

```ts
const event = await emitEvent(tx, tables, {
  source: 'openclam',
  module: 'core',
  eventType: 'user.created',
  entityType: 'user',
  entityId: parsed.id,
  userId: ctx?.userId,
  payload: {
    id: parsed.id,
    name: parsed.name,
    slug: parsed.slug,
    type: parsed.type,
    email: parsed.email ?? null,
    is_active: parsed.is_active,
    status: parsed.status,
    metadata: parsed.metadata ?? null,
    created_at: parsed.created_at,
    updated_at: parsed.updated_at,
  },
});
await maybeCreateDecision(tx, tables, event, ctx);
```

Events are written inside the same transaction as the mutation
(outbox pattern). A separate processor drains the outbox asynchronously —
the service never runs handlers inline, which keeps mutations fast and
predictable.

Pattern matching for rules is `exact`, `<domain>.*`, or `*`.

---

## 6. Decision traces

Mutations accept an optional `_decision` envelope on the input. The API
layer strips it before Zod parsing (`packages/api/src/index.ts →
extractDecision`) and attaches a `DecisionContext` to `ctx.decision`.
Services forward it through `MutationContext.decision`, and
`maybeCreateDecision(...)` persists a decision row linked to the emitted
event when one is present. No decision → no row; the core data path is
unaffected.

This is how agents get an auditable "why" without sprinkling bespoke logging.

---

## 7. Entity expansion and `fields` projection

Two list/get options commonly appear on services:

- `options.fields?: string[]` — column projection. `userService.list`
  filters by `tables.users[f]` so unknown columns are silently dropped.
  Surfaces in the CLI as `--input-json '{"fields":"id,name,type"}'`.
- `options.expand?: string[]` — related data pull-in. Implementations use a
  whitelist; callers get what they ask for and nothing else. `core` ships
  the generic mechanism via the extension SDK `enrichments` hook
  (`EnrichmentDef` in `define.ts`), where any extension can contribute an
  `--expand` key for an entity owned by another extension.

Rule: always validate `expand` entries against a known-good list. Never
interpolate them into SQL.

---

## 8. Procedure pattern (thin wrapper)

Procedures are authored with `defineProcedure`. They:

1. Declare a Zod `input` (merging path/query/body).
2. Declare a Zod `output` (usually `z.record(z.string(), z.unknown())` for
   dynamic rows or the domain schema for strict outputs).
3. Declare a `permission` string (or `null` for any-authenticated-user).
4. Call into a service inside `handler`, passing `ctx.user.id` and
   `ctx.decision` through.

From `packages/core/src/procedures/users.ts`:

```ts
export const create = defineProcedure({
  method: 'POST',
  path: '/users',
  summary: 'Create a user',
  permission: 'user:create',
  input: insertUserSchema.extend({ _decision: z.unknown().optional() }),
  output: anyRecord,
  handler: async ({ input, ctx }) => {
    const { _decision, ...data } = input;
    void _decision;
    return await userService.add(ctx.db, ctx.tables, data, {
      userId: ctx.user.id,
      decision: ctx.decision,
    });
  },
});
```

- No business logic in the handler — it's a pass-through.
- Permission is enforced by the loader (`loader.ts` checks
  `runtime.user.permissions`) **before** the handler runs.
- `AppError` from `@openclam/extension-sdk` maps to the right HTTP status.
- `pathPrefix` on `defineExtension` is the mounting lever. Default:
  `/${id}`. `core` uses `pathPrefix: ''` so `/api/users` isn't
  `/api/core/users`.

---

## 9. Table design

All tables live in per-dialect files:

```
packages/<extension>/src/tables/sqlite.ts   // drizzle-orm/sqlite-core
packages/<extension>/src/tables/postgres.ts // drizzle-orm/pg-core
```

Core tables in this repo: `packages/core/src/tables/{sqlite,postgres}.ts`.

### Column conventions

| Column | Type | Rule |
|--------|------|------|
| `id` | `TEXT PRIMARY KEY` | UUIDv4, populated by the insert schema |
| `slug` | `TEXT UNIQUE` | Where applicable. Enforced by `SlugSchema` in core |
| `created_at` | `TEXT NOT NULL` | ISO datetime string, set on insert |
| `updated_at` | `TEXT NOT NULL` | ISO datetime, bumped on every update |
| `deleted_at` | `TEXT` (nullable) | `NULL` = active, ISO datetime = soft-deleted |
| FKs | `TEXT NOT NULL` (or nullable) | `REFERENCES parent(id) ON DELETE CASCADE` for sub-tables |

### Dedicated per-entity sub-tables

OpenClam never uses polymorphic `entity_type` + `entity_id` patterns.
Sub-tables are FK'd into exactly one parent. Canonical shape (when contacts
is in play): `persons`, `person_emails`, `person_phones`, … Each sub-table
cascades on parent delete.

Why: FK integrity, CASCADE cleanup, full type safety in Drizzle queries,
and no runtime `entity_type` dispatch.

### Dialect differences

SQLite uses `INTEGER { mode: 'boolean' }` for booleans and `TEXT` for ISO
datetimes; Postgres uses native types. Both dialects must exist — the
extension-sdk's `ExtensionDef.tables` mandates `{ sqlite, postgres }`.

### Vector columns

Core defines `float32Vector` in `packages/core/src/tables/sqlite.ts` as a
libSQL `F32_BLOB` custom type. Extensions that need embeddings should reuse
this pattern rather than invent their own.

---

## 10. Workspace layout

Verified against `packages/core/src/connection/paths.ts` and `types.ts`.
**Important correction vs older docs:** there are no separate
`environments` anymore — the workspace has a single DB at the workspace
root, and `GlobalConfig` is `version: 3`, not 2. If other docs in this repo
still say "v2 + envs", treat them as stale.

### Filesystem

```
~/.openclam/                       # getClamHome() — override via OPENCLAM_HOME
  openclam.json                    # GlobalConfig v3
  workspaces/
    <workspace-slug>/
      manifest.json                # WorkspaceManifest
      data.db                      # SQLite DB (local mode, default path)
  extensions/                      # user-installed extensions
  backups/                         # getBackupsDir()
```

Path helpers: `getClamHome`, `getConfigPath`, `getWorkspacesDir`,
`getWorkspaceDir(slug)`, `getWorkspaceManifestPath(slug)`,
`getDefaultDbPath(workspaceSlug)`, `getExtensionsDir`, `getBackupsDir`,
`assertSafeBackupPath`.

### GlobalConfig v3

```ts
// packages/core/src/connection/types.ts
export const GlobalConfigSchema = z.object({
  version: z.literal(3).default(3),
  activeWorkspace: z.string().nullable(),
  activeUser: z.string().nullable().default(null),
  users: z.record(z.string(), UserEntrySchema).default({}),
  serve: ServeConfigSchema,
  vaultKeyCmd: z.string().optional(),
  remoteAccount: RemoteAccountSchema.nullable().optional(),
  telemetry: TelemetryConfigSchema,
});
```

`UserEntry` is `{ name, type: 'human' | 'agent', outputFormat?,
passwordHash?, createdAt }`. The config holds the list of known local
users plus the active pointer and the serve config.

### WorkspaceManifest

```ts
export const WorkspaceManifestSchema = z.object({
  workspaceId: z.uuid(),
  name: z.string().min(1),
  connection: ConnectionSchema.nullable().default(null),
  extensions: z.array(z.string()).default([]),
  events: EventsConfigSchema.optional(),
  webhooks: WebhooksConfigSchema.optional(),
  access: AccessModeSchema.optional(),          // 'local-direct' | 'local-container' | 'remote'
  daemonUrl: z.string().url().optional(),
  extensionTrust: z.record(z.string(), ExtensionTrustEntrySchema).optional(),
  ai: AiConfigSchema.optional(),
  storage: StorageConfigSchema.optional(),
  adminKeyHash: z.string().nullable().optional(),
  createdAt: z.iso.datetime(),
});
```

### Connection types

Discriminated on `{ dialect, mode, provider }`. Verified in
`connection/types.ts`:

| Dialect | Mode | Provider | Fields |
|---------|------|----------|--------|
| `sqlite` | `local` | `local` | `path` |
| `sqlite` | `remote` | `turso` \| `d1` | `url`, `authToken?` |
| `postgres` | `local` | `local` | `url` |
| `postgres` | `local` | `pglite` | `path` |
| `postgres` | `remote` | `neon` | `url` |
| `mysql` | `local` | `local` | `url` |
| `mysql` | `remote` | `planetscale` | `url` |

### Access modes

`access` on the manifest controls how the CLI reaches the workspace:

- `local-direct` — CLI opens the SQLite/PGlite file itself.
- `local-container` — CLI talks to the local daemon over its socket.
- `remote` — CLI talks to a remote daemon over HTTPS with a platform token.

### Slugs

`SlugSchema`: lowercase, starts with a letter, `[a-z0-9]` with single
hyphens, max 128 chars, must not look like a UUID. Use `toSlug(name)` to
derive one from a display name.

---

## 11. Auth resolution

Single canonical chain, verified in
`apps/cli/src/utils/db.ts → requireUser`:

1. `--token <secret>` flag
2. `OPENCLAM_TOKEN` env var
3. `OPENCLAM_AGENT` env var → namespaced credentials store lookup, scoped
   to the active workspace (`{ workspaceSlug, workspaceId }`)
4. `activeUser` from `GlobalConfig` → session credential lookup scoped to
   the same workspace
5. Fail: `No user logged in. Run 'clam login', set OPENCLAM_TOKEN, set
   OPENCLAM_AGENT, or pass --token.`

Every resolved token is fed into `apiKeyService.authenticateApiKey(db,
tables, token)` which returns `{ userId, keyId, permissions }`. The HTTP
server does the same resolution on the `Authorization: Bearer <key>`
header (see `surfaces.md` §4 for the daemon/platform-trust variants).

Two workspace access modes imply different credential paths:

- `local-direct` — `requireUser` runs against the local DB.
- `local-container` / `remote` — `getContainerClient()` resolves the token
  and attaches it to a `DaemonClient`; the daemon runs the auth step.

---

## 12. Extension definition

All the pieces above come together in `defineExtension`:

```ts
// packages/extension-sdk/src/define.ts
export function defineExtension<TServices, TProcedures>(
  def: ExtensionDef<TServices, TProcedures>,
): Extension<TServices, TProcedures>
```

`ExtensionDef` key fields:

- `id` — slug; also the default `/api/<id>/` prefix. Slug-validated at
  define time.
- `label`, `version`, `description`, `dependencies`.
- `tables: { sqlite, postgres }` — Drizzle tables, both dialects required.
- `migrations: ExtensionMigration[]` — ordered `{ version, description,
  sqlite, postgres }` entries, idempotent (`CREATE TABLE IF NOT EXISTS`).
  Add new entries, never mutate existing ones.
- `permissions` — nested `resource → action → description`. These become
  the permission strings checked in `defineProcedure({ permission: ... })`.
- `capabilities: ExtensionCapabilities` — declared access to other
  extensions' tables, network, filesystem, and system tokens like
  `events:emit`, `cron:register`, `embeddings:enqueue`. Enforced by the
  daemon proxy.
- `trusted` — bundled extensions default `true` (in-process). Custom
  extensions default `false` (sandboxed).
- `services` — `defineService(factory)` entries. Exposed to handlers as
  `ctx.services.<name>`. Scoped to this extension only — no cross-extension
  service calls.
- `procedures` — tree of `defineProcedure(...)` nodes. Mounted by the
  loader at `pathPrefix + path`.
- `pathPrefix` — override for the URL prefix. Empty string mounts
  directly under `/api/...` (`core`, `secrets`).
- `actions` — `ExtensionAction` map (rule-engine actions).
- `events` — declared event catalog, one Zod schema per event type.
- `commands` — `(ctx) => CommandDef[]`, contributing CLI subcommands.
- `enrichments: EnrichmentDef[]` — per-entity expand entries.
- `recommendedRules: RecommendedRule[]` — suggested rule rows prompted at
  enable time.
- `scheduledJobs: ScheduledJob[]` — system cron rows seeded at enable,
  removed at disable.

The loader (`loader.ts`) walks `procedures`, enforces slug uniqueness,
checks permissions, materializes services per request, and builds the
`ProcedureContext` that handlers receive.

---

## 13. Surface conventions (summary)

Full cross-surface translation rules are in `surfaces.md`. Constraints that
matter when authoring:

- **Thin wrappers only.** CLI commands and procedures should do three
  things: validate input, call a service, format output. No DB access and
  no domain logic in surfaces.
- **Route ordering.** Static paths (`/api/persons/merge`) must be declared
  before parameterized siblings (`/api/persons/{id}`) so the router
  doesn't treat `merge` as an id.
- **Dynamic imports in handlers.** CLI and event actions import service
  modules lazily to avoid circular dependencies between `core`, extensions,
  and the CLI shell.
- **Never touch `openclam.json` or `manifest.json` directly.** Use the
  load/save helpers in `@openclam/core` (`loadConfig`, `loadManifest`,
  `setActiveWorkspace`, etc.) so migrations and validation run.
- **Every Zod validation error has the same envelope** regardless of
  surface — see `surfaces.md §7`.

---

## 14. Package map

High level — verified by `ls packages/` on the current branch.

### System of record (core + daemon-owned)

| Package | Role |
|---------|------|
| `core` | Workspaces, users, projects, roles, tags, rules, logs, config, decision traces, extension system |
| `events` | Event outbox, `emitEvent`, pattern matching |
| `rules` | Rule engine — conditions, actions |
| `actions` | Action registry |
| `cron` | Scheduled jobs |
| `workflows` | Workflow definitions, runs, step attempts |
| `webhooks-inbound` / `webhooks-outbound` | Inbound/outbound webhook handling |
| `storage` / `storage-s3` | Bucket + file metadata, S3 backend |

Domain extensions (`contacts`, `deals`, `tasks`, `messaging`, `marketing`,
`content`, `calendar`, `research`, `tickets`, `wiki`) are maintained as
separate packages outside this monorepo. Treat their slugs as canonical
when constructing paths — `/api/contacts/people`, `/api/deals/deals`, etc.

### Surfaces and supporting

| Package | Role |
|---------|------|
| `api` | Fetch-handler factory (`createApp`); auth, rate limiting, CORS, dispatch |
| `extension-sdk` | `defineProcedure` / `defineService` / `defineExtension`, loader, OpenAPI generator, `AppError` |
| `mcp-server` | MCP server definition (`createMcpServer`); tools (`clam_status`, `clam_api`, `clam_sql`) and resources |
| `platform-client` | Typed client for the managed-platform plane (oRPC `OpenAPILink`) |
| `runtime` | Workspace bootstrap: connect DB, resolve extensions, initialize tables |
| `contracts` | Shared platform oRPC contract |
| `schemas` | Pagination, common query schemas, validation helpers |
| `adapter-sqlite` / `adapter-postgres` | Dialect adapters, search, FTS |
| `provider-sqlite-local` / `provider-postgres-local` / `provider-pglite` | DB connection factories |
| `ai` | LLM provider abstraction |
| `vault` | Encrypted secret store |
| `sandbox` | Extension sandbox runtime |
| `ui` | React components (Base UI + Tailwind) |
| `lib` | Shared utilities (`isValidSlug`, ids, time helpers) |

Apps:

| App | Role |
|-----|------|
| `apps/cli` | `clam` binary, thin wrapper over services / HTTP |
| `apps/daemon` | Long-running workspace daemon (HTTP, cron, watcher) |
| `apps/mcp-service` | stdio MCP transport wrapper over `packages/mcp-server` |
| `apps/console` | Web console |
| `apps/docs` | Documentation site |

---

## 15. See also

- `surfaces.md` — how these patterns appear on CLI, MCP, HTTP, and typed
  client surfaces.
- `skills/openclam-cli/SKILL.md` and its per-domain references — the
  `--input-json` shapes are the Zod input schemas verbatim.
- `packages/extension-sdk/src/define.ts` — canonical definitions for every
  type referenced above.
- `packages/core/src/services/user.ts` — best example of the full service
  pattern end to end.
