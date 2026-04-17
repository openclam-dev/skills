---
name: extend-index
description: >
  Build custom OpenClam extensions with `@openclam/extension-sdk`. Covers the
  extension model, the on-disk layout under `~/.openclam/extensions/`, the
  trusted vs sandboxed distinction, and the `clam extension` CLI for managing
  installed extensions.
metadata:
  author: openclam
  version: 0.1.0
---

# Extend

An extension is a single object built with `defineExtension` from
`@openclam/extension-sdk`. It declares its own tables, migrations, procedures
(HTTP + OpenAPI + typed-client in one go), services, permissions, actions,
events, recommended rules, scheduled jobs, enrichments, and CLI commands.
The daemon loader picks it up, namespaces it under `/api/{id}/...`, and wires
permissions, decision capture, and event emission uniformly.

Everything you write lives under one `defineExtension({...})` call. No Hono,
no oRPC server primitives, no hand-rolled route registration — the SDK is the
only surface you import from.

## Minimal example

```ts
// src/index.ts
import { defineExtension, defineProcedure } from '@openclam/extension-sdk';
import { z } from 'zod';
import * as sqliteTables from './tables/sqlite';
import * as postgresTables from './tables/postgres';

const ping = defineProcedure({
  method: 'GET',
  path: '/ping',
  summary: 'Health check',
  permission: null,                 // any authenticated user
  input: z.object({}),
  output: z.object({ ok: z.literal(true) }),
  handler: async () => ({ ok: true as const }),
});

export const extension = defineExtension({
  id: 'invoicing',
  label: 'Invoicing',
  version: '0.1.0',
  tables: { sqlite: sqliteTables, postgres: postgresTables },
  migrations: [],
  procedures: { ping },
});
```

After `clam extension enable invoicing`, this procedure is reachable as:
- REST: `GET /api/invoicing/ping`
- OpenAPI: entry auto-generated from `input`, `output`, `summary`
- Typed client: `client.invoicing.ping({})`

## The pieces (four files)

- `defining.md` — `defineExtension`, `defineProcedure`, `defineService`, and
  the `ProcedureContext` you get inside handlers.
- `data.md` — Zod schemas (insert/update/output), dual SQLite + Postgres
  Drizzle tables, and the migration rules.
- `capabilities.md` — actions, events, recommended rules, scheduled jobs,
  enrichments, permissions, commands.
- `../surfaces.md` — how procedures become CLI, MCP, HTTP, and typed-client
  invocations.

## File layout on disk

Custom extensions live under `~/.openclam/extensions/<id>/`. The runtime
scans this directory on startup and loads any package that exports a
`defineExtension(...)` value.

```
~/.openclam/extensions/invoicing/
  package.json                      depends on @openclam/extension-sdk
  tsconfig.json
  openclam.extension.json           manifest (id, version, entry point)
  src/
    index.ts                        defineExtension(...) — main export
    tables/
      sqlite.ts                     Drizzle SQLite tables
      postgres.ts                   Drizzle Postgres tables
    schemas/
      index.ts                      Zod insert/update/output schemas
    services/
      invoice.ts                    business logic
      index.ts                      re-exports per defineService
    procedures/
      invoice.ts                    defineProcedure(...) descriptors
      index.ts                      procedure tree (nested object)
    commands.ts                     CLI CommandDef[] contributions
```

`clam extension init <name>` scaffolds this layout. `--scaffold --entity
<name> --fields "..."` additionally generates a CRUD entity with tables,
schemas, service, procedures, and CLI commands wired up.

## Trusted vs sandboxed

Every `ExtensionDef` has a `trusted?: boolean` flag:

- **Trusted** (`trusted: true`) — runs in-process with the daemon. Bundled
  extensions (those shipped with OpenClam) default to trusted. Full access
  to declared capabilities; still gets a scoped db proxy for
  defence-in-depth.
- **Sandboxed** (`trusted: false`, the default for custom extensions) — the
  loader executes it under the declared capability constraints from
  `ExtensionCapabilities`: which tables it reads/writes, whether it can
  reach the network or filesystem, and which system capabilities (e.g.
  `events:emit`, `embeddings:enqueue`) it requires. The user approves
  these at enable time.

Even trusted extensions never see the raw HTTP request, the Hono context,
or another extension's services. `ctx.services` is scoped to the current
extension only — cross-extension calls go through the public REST API or
events, not an in-process call.

## Managing extensions

```bash
# Scaffold a new one in ~/.openclam/extensions/<name>/
clam extension init invoicing
clam extension init invoicing --scaffold --entity invoice \
  --fields "number:string, amount:number, status:enum(draft,sent,paid)"

# Install from npm, a tarball, or symlink a local dir for development
clam extension install @acme/openclam-ext-billing
clam extension install ./path/to/local-ext --link

# See what's available (bundled + custom)
clam extension list

# Enable for the active workspace (runs migrations, seeds scheduled
# jobs, prompts for recommended rules, triggers a daemon reload)
clam extension enable invoicing
clam extension enable invoicing -y          # auto-accept recommended rules

# Disable for the active workspace (removes system cron jobs, reloads)
clam extension disable invoicing

# Remove the installed directory (must be disabled first)
clam extension remove invoicing --force
```

See `../surfaces.md` for surface translation.

## When to build a custom extension

Build one when:
- You have a new domain object (invoice, ticket, shipment) that deserves
  its own tables, permissions, and procedures.
- You want to expose a new integration (a third-party API client) as
  actions callable by rules.
- You want to contribute `--expand` data to another extension's entities
  without owning those entities (use an enrichment).

Don't build one when:
- A bundled extension already covers the concept — extend through its
  existing procedures, recommended rules, or actions instead.
- You just need a one-off script — use a scheduled job on an existing
  extension, or a workflow.

When in doubt, start with `clam extension list` to see what's already
installed. If something fits 80% of the need, extend it via rules or a
thin wrapper extension rather than duplicating it.
