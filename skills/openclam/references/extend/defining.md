---
name: extend-defining
description: >
  The three core defines — `defineExtension`, `defineProcedure`,
  `defineService` — and the `ProcedureContext` handlers receive. How one
  procedure descriptor becomes a REST route, OpenAPI entry, and typed-client
  method, and the scope rules for services and cross-extension access.
metadata:
  author: openclam
  version: 0.1.0
---

# Defining extensions

Three functions from `@openclam/extension-sdk` produce the whole authoring
surface:

- `defineExtension(def)` — the descriptor the daemon loader consumes.
- `defineProcedure(def)` — one REST route + OpenAPI entry + typed-client
  method, from a single source of truth.
- `defineService(factory)` — a namespaced bundle of methods handlers can
  call via `ctx.services.<name>`.

Procedures are mounted under `/api/{extension.id}/...` automatically. Slug
uniqueness across installed extensions is enforced at load time. The whole
extension moves as one version unit — procedures are not independently
versioned.

## Minimal example

```ts
// src/procedures/invoice.ts
import { z } from 'zod';
import { defineProcedure } from '@openclam/extension-sdk';

export const create = defineProcedure({
  method: 'POST',
  path: '/invoices',
  summary: 'Create an invoice',
  permission: 'invoice:create',
  input: z.object({
    customer_id: z.uuid(),
    amount: z.number().positive(),
  }),
  output: z.object({
    id: z.uuid(),
    customer_id: z.uuid(),
    amount: z.number(),
    created_at: z.iso.datetime(),
  }),
  handler: async ({ input, ctx }) => {
    return ctx.services.invoice.create(input, ctx.user.id);
  },
});
```

```ts
// src/services/invoice.ts
import { defineService } from '@openclam/extension-sdk';
import { v4 as uuidv4 } from 'uuid';

export const invoiceService = defineService(({ db, tables }) => ({
  async create(input: { customer_id: string; amount: number }, userId: string) {
    const row = {
      id: uuidv4(),
      customer_id: input.customer_id,
      amount: input.amount,
      created_by: userId,
      created_at: new Date().toISOString(),
    };
    await db.insert(tables.invoices).values(row).run();
    return row;
  },
}));
```

```ts
// src/index.ts
import { defineExtension } from '@openclam/extension-sdk';
import * as sqliteTables from './tables/sqlite';
import * as postgresTables from './tables/postgres';
import * as invoiceProcedures from './procedures/invoice';
import { invoiceService } from './services/invoice';

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
    },
  },
  tables: { sqlite: sqliteTables, postgres: postgresTables },
  migrations: [],
  services: { invoice: invoiceService },
  procedures: { invoices: invoiceProcedures },
});
```

That one `create` procedure becomes:
- `POST /api/invoicing/invoices` (REST)
- an OpenAPI entry (request/response schemas, summary, description)
- `client.invoicing.invoices.create({ customer_id, amount })` (typed client)

All derived from the same descriptor.

## `defineExtension`

Signature (from `define.ts`):

```ts
export function defineExtension<
  TServices extends ServiceTree,
  TProcedures extends ProcedureTree,
>(def: ExtensionDef<TServices, TProcedures>): Extension<TServices, TProcedures>
```

Key fields on `ExtensionDef`:

| Field | Purpose |
| --- | --- |
| `id` | Unique slug. Becomes the URL namespace (`/api/{id}/...`). Loader fails on duplicate or invalid slug. |
| `label`, `version`, `description` | Metadata shown in `clam extension list` and the console. |
| `dependencies` | Other extension ids this one needs — enabled before this one. |
| `tables` | `{ sqlite, postgres }` — Drizzle tables per dialect. See `data.md`. |
| `migrations` | Ordered list of idempotent SQL migrations per dialect. See `data.md`. |
| `permissions` | Nested `resource: { action: description }`. Referenced by procedure `permission` strings. See `capabilities.md`. |
| `capabilities` | `ExtensionCapabilities` — `tables` (cross-extension access), `network`, `filesystem`, `systemCapabilities`. Approved at enable time. |
| `trusted` | In-process vs sandboxed. Bundled = true, custom = false. |
| `services` | `ServiceTree` — materialized per request, exposed as `ctx.services`. |
| `procedures` | `ProcedureTree` — nested object; keys become typed-client path segments. |
| `pathPrefix` | Override the `/{id}` URL prefix. Set to `''` to mount procedures at the API root (e.g. `/api/schedules`). |
| `actions`, `events`, `recommendedRules`, `scheduledJobs`, `enrichments`, `commands` | Declarative capabilities — see `capabilities.md`. |

The loader validates `id` against the slug regex and rejects duplicates.
It also pre-builds the oRPC contract from every procedure so the console's
typed client can call `buildContract([exts])` and get exact URL paths for
each method.

## `defineProcedure`

Signature:

```ts
export function defineProcedure<
  TInputSchema extends ZodTypeAny,
  TOutputSchema extends ZodTypeAny,
  TServices,
>(def: ProcedureDef<TInputSchema, TOutputSchema, TServices>):
  Procedure<TInputSchema, TOutputSchema, TServices>
```

Fields on `ProcedureDef`:

| Field | Purpose |
| --- | --- |
| `method` | `'GET' \| 'POST' \| 'PUT' \| 'PATCH' \| 'DELETE'`. |
| `path` | Relative to the extension's namespace. Must start with `/`. Path params use `{name}` syntax. |
| `summary`, `description` | Surfaced in the OpenAPI spec. |
| `permission` | `string \| null`. If a string, loader enforces it before the handler runs (403 on failure). `null` means any authenticated user. |
| `input` | Zod schema. Path, query, and body are merged and validated against it. |
| `output` | Zod schema for the response body. |
| `handler` | `({ input, ctx }) => Promise<output>`. See below. |

### From descriptor to runtime

One `defineProcedure({...})` call becomes:

1. A tagged `Procedure` object (`__type: 'procedure'`) the extension
   exports. It also carries a pre-built `contract` piece.
2. At daemon startup, `loadExtensions([ext])` walks the `procedures`
   tree and for each descriptor:
   - Prepends `/api/{id}` (or `pathPrefix`) to `path`.
   - Wraps the handler with a permission check: if the user's
     permissions don't include `def.permission` (and don't include the
     `'*'` admin wildcard), throws `AppError.forbidden()` before the
     handler runs.
   - Materializes `ctx.services` from this extension's `defineService`
     factories using the request-scoped db.
   - Invokes the handler, catching `AppError` and mapping to an oRPC
     `ORPCError` with the right HTTP status code.
3. The merged router is wrapped in an `OpenAPIHandler` (fetch adapter)
   so the daemon calls `handle(request)` and gets
   `{ matched: true, response }` for a hit.
4. Separately, `buildContract([extensions])` walks the same tree at
   module load time to produce the static oRPC contract the console's
   typed client consumes.

### Path params and input merging

```ts
defineProcedure({
  method: 'GET',
  path: '/invoices/{id}',
  input: z.object({
    id: z.uuid(),                             // path param
    include_deleted: z.coerce.boolean().optional(),  // query string
  }),
  output: invoiceSchema,
  // ...
});
```

Path params, query string, and request body are merged into one object
before Zod validation. The handler sees the merged, parsed result as
`input`. Use `z.coerce.number()` / `z.coerce.boolean()` for query
strings since they arrive as strings over the wire.

### Errors

Throw `AppError` from `@openclam/extension-sdk` for any non-2xx outcome.
The loader maps the `code` to the right HTTP status:

```ts
import { AppError } from '@openclam/extension-sdk';

if (!row) throw AppError.notFound('Invoice', id);         // 404
if (row.locked) throw AppError.conflict('Already sent');  // 409
if (!isValid(input)) throw AppError.badRequest('Bad amount', { amount }); // 400
```

Available helpers: `badRequest`, `unauthorized`, `forbidden`, `notFound`,
`conflict`, `unprocessable`, `internal`, `notImplemented`. The full code
set includes `RATE_LIMITED`, `PRECONDITION_FAILED`, and `UNAVAILABLE` too.

Any non-`AppError` thrown from a handler propagates as a 500.

## The `ProcedureContext`

Every handler gets `{ input, ctx }`. `ctx` is narrow by design — no raw
request, no Hono context, no framework primitives:

```ts
interface ProcedureContext<TServices = unknown> {
  db: OpenClamDb;                 // scoped (capability-proxied) db
  tables: OpenClamTables;         // scoped table refs
  dialect: string;                // 'sqlite' | 'postgres'
  user: CurrentUser;              // { id, permissions, workspaceId, apiKeyId }
  workspace: { id: string; slug: string };
  services: TServices;            // THIS extension's services only
  emit: (eventType: string, payload: unknown) => Promise<void>;
  decision?: DecisionContext;     // extracted from `_decision` in body
  ai?: unknown;                   // AI provider if configured
}
```

Notes:

- **`db` and `tables` are request-scoped.** The daemon proxies the db to
  enforce the extension's declared `capabilities.tables` access levels.
  Never cache `ctx.db` across requests.
- **`user` is always present.** Authentication happens before the
  handler runs; an unauthenticated request never reaches it.
- **`services` is scoped to this extension only.** See below.
- **`emit`** writes a domain event into the outbox. For most domain
  events, prefer emitting from inside a service method (so the event
  fires regardless of which surface called the service). Use handler-
  level `ctx.emit` for surface-specific events.
- **`decision`** is extracted from the `_decision` property on the
  request body by the loader. Pass it to `maybeCreateDecision(...)`
  inside services that mutate data; see automation references for the
  full decision-trace pattern.
- **`ai`** is present only when the workspace has an AI provider
  configured (e.g. an OpenAI key). Handlers that need AI should check
  for its presence and return `AppError.unavailable(...)` otherwise.

## `defineService`

Signature:

```ts
export function defineService<TMethods extends Record<string, unknown>>(
  factory: ServiceFactory<TMethods>,
): Service<TMethods>

type ServiceFactory<TMethods> = (ctx: ServiceContext) => TMethods;

interface ServiceContext {
  db: OpenClamDb;
  tables: OpenClamTables;
  dialect: string;
}
```

The factory runs per request (because `db` is request-scoped) and returns
the methods handlers can call. It intentionally does not receive `user`
— services are user-agnostic. If a method needs the caller's identity,
take it as an argument:

```ts
export const invoiceService = defineService(({ db, tables }) => ({
  async create(input: CreateInvoiceInput, userId: string) {
    // ...
  },
  async listForUser(userId: string) {
    // ...
  },
}));
```

### Scope rule: no cross-extension service calls

`ctx.services` contains only the services declared by the current
extension. There is no `ctx.otherExtension` or shared service registry —
it is impossible to reach into another extension's internals in-process.

When you need behaviour from another extension:

1. **Call its public REST API** — via an HTTP client, using the
   workspace's auth. Most relevant for cloud deployments.
2. **Emit a domain event** — the other extension subscribes via a rule
   or webhook. The preferred decoupled pattern; see `capabilities.md`.
3. **Dispatch one of its actions** — declare the action call in a rule
   (recommended or user-defined) and let the action pipeline route it.

This scope rule is what makes extensions swappable — the sandboxed
runtime can enforce the capability boundary, and an extension's internal
shape is free to change without breaking other extensions.

## Putting it together (surface mapping)

For the example `create` procedure above, the four surfaces are:

```bash
# CLI (via extension-contributed command or generic invoke)
clam invoicing invoice add --input-json '{"customer_id":"...","amount":120}'

# HTTP
curl -X POST https://daemon/api/invoicing/invoices \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -d '{"customer_id":"...","amount":120}'

# MCP (one tool per procedure; name derived from procedure path)
# tool: invoicing.invoices.create
# input: { customer_id, amount }

# Typed client (console, other extensions via HTTP)
const invoice = await client.invoicing.invoices.create({
  customer_id: '...',
  amount: 120,
});
```

See `../surfaces.md` for surface translation.

## Keeping the surface narrow

Two discipline rules:

- **Never import from `hono`, `@orpc/contract`, or `@orpc/server` in
  extension code.** If you feel the urge, the SDK probably already has
  what you need — or the thing you want doesn't belong in an extension.
- **Never import from another extension's non-public entry point.**
  Extension packages export only their `defineExtension` result. If you
  need a service's output shape, import it from the typed client
  contract, not the internal service module.

These rules keep extensions swappable and the sandbox enforcement
tractable.
