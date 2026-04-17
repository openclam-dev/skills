---
name: surfaces
description: >
  Translation rules for invoking OpenClam operations across surfaces — CLI,
  MCP, HTTP API, and typed client. Same operation shape, four delivery
  mechanisms. Load this whenever you need to translate an operation you see
  documented for one surface into a call on another.
metadata:
  author: openclam
  version: 0.1.0
tags:
  - surfaces
  - cli
  - mcp
  - api
  - client
  - auth
  - errors
---

# Surfaces

OpenClam exposes the same business operations through four delivery surfaces.
The unified model is:

- A **namespaced procedure name** — e.g. `contacts.people.create`, or the flat
  CLI form `clam contacts person add`.
- A **Zod input schema** (`ProcedureDef.input`).
- A **Zod output schema** (`ProcedureDef.output`).
- A **handler** that runs inside the daemon against a scoped `ProcedureContext`.

Everything below is a mechanical translation of that triple onto a specific
wire format. Verified in this repo against:

- `packages/extension-sdk/src/define.ts` — `defineProcedure`, `ProcedureDef`
- `packages/extension-sdk/src/loader.ts` — mounting under `/api/{id}/...`
- `packages/api/src/index.ts` — fetch handler, auth, rate limit
- `packages/mcp-server/src/server.ts` / `tools.ts` — MCP tool registrations
- `packages/platform-client/src/index.ts` — typed client (oRPC OpenAPILink)
- `apps/cli/src/index.ts` + `apps/cli/src/utils/db.ts` — CLI flags and auth

See `patterns.md` for the underlying service/table/event conventions that
every surface shares.

---

## 1. Overview — what's identical across surfaces

Regardless of which surface calls a procedure:

- The **input** is one JSON object. Path params, query params, and JSON body
  are all merged into that object before Zod validation.
- The **output** is one JSON object (or an array, or a paginated page).
- **Zod does the validation.** A failure produces the same error envelope
  everywhere — only the transport differs (stderr JSON, MCP tool result,
  HTTP 4xx body, thrown client error).
- The **auth principal** is always a `userId` plus a permission set. How it's
  derived depends on the surface (flag, env, session, header), but the
  enforcement inside the handler is identical.
- **Mutations may carry a `_decision` envelope.** The API loader strips it
  before Zod parses (`packages/api/src/index.ts` → `extractDecision`). It's
  optional and never changes the shape the handler sees.

---

## 2. CLI

Binary: `clam` (from `apps/cli`).

### Invocation shape

```
clam [global flags] <domain> [<entity>] <action> [positional-id] [--input-json '{...}']
```

The top-level command is always the domain (`contacts`, `deals`, `user`,
`workspace`, …). For core entities there's an entity slot (`user`, `role`,
`project`, …); some core commands are flat (`sql`, `status`, `whoami`).

### Global flags

Defined on the root program in `apps/cli/src/index.ts`:

| Flag | Purpose |
|------|---------|
| `--json` | Compact JSON output — use when parsing values programmatically |
| `--markdown` | Markdown tables for lists, key-value blocks for objects |
| `--csv` | CSV output (where applicable) |
| `--token <secret>` | API key for authentication |
| `--no-telemetry` | Disable anonymous telemetry for this invocation |
| `-v, --version` | Print version |

There is no global `--env` on the current root program. Environment
switching is workspace-level (`clam workspace use <slug>`), not a per-command
flag. If you see `--env` referenced anywhere, it's stale.

### The `--input-json` contract

Every command that accepts input takes `--input-json '<json>'`. The JSON
object is the exact input shape the procedure's Zod schema expects.
Individual `--foo` flags DO NOT exist — this is intentional, so agents have
one canonical form. Resolution order inside a command:

```
input.<field> ?? positional ?? (no individual flags)
```

Positional forms are accepted only for identity-only operations
(`remove`, `restore`, `purge`, `get`) so shell usage stays concise:

```bash
clam --json user remove <id>
clam --json user remove --input-json '{"id":"<id>"}'
```

Commands with interactive prompts (`workspace new`, `setup`, …) **will hang**
in a non-interactive shell if `--input-json` is omitted. Agents must always
provide `--input-json` with every required field.

### Output

`--json` (or the user's configured default when they're an agent type) makes
the CLI emit a single JSON value on stdout. `--markdown` emits a markdown
table for lists and key-value blocks for single objects. Errors go to
stderr as JSON.

### Auth resolution (verified in `apps/cli/src/utils/db.ts → requireUser`)

1. `--token <secret>` (highest priority)
2. `OPENCLAM_TOKEN` env var
3. `OPENCLAM_AGENT` env var — looks the agent's key up in the namespaced
   credentials store for this workspace
4. `activeUser` from the global config — session credential lookup
5. Fail with `No user logged in. Run 'clam login', set OPENCLAM_TOKEN, set
   OPENCLAM_AGENT, or pass --token.`

The resolved token is always an API key, and the downstream step is always
`apiKeyService.authenticateApiKey(...)`, so there's one code path whether
the user is a human with a session or an agent with a stored key.

---

## 3. MCP

Package: `packages/mcp-server/` — transport-agnostic server definition.
Binary: `apps/mcp-service/` — wires it to stdio for Claude Desktop, Codex,
and similar local clients. Daemon-hosted flavor uses the same server over
Streamable HTTP.

### Tools (verified in `packages/mcp-server/src/server.ts`)

OpenClam exposes **three** tools — not one tool per procedure. This is a
deliberate design choice: everything reachable over REST is reachable via
`clam_api`, and the procedure catalog is discovered through resources.

| Tool | Purpose |
|------|---------|
| `clam_status` | Orientation. Returns workspace config, enabled extensions, dialect. Call first. |
| `clam_api` | Generic REST passthrough. Takes `method`, `path` (must start with `/api/`), optional `body`, optional `query`. |
| `clam_sql` | Read-only SQL against the workspace DB. Useful for joins and aggregations the REST surface doesn't expose. |

Resources served alongside the tools:

- `openclam://mcp-primer` — read before calling `clam_api`. Explains the
  command→route mapping.
- `openclam://skill/<domain>` — per-domain skill references (the
  `--input-json` shapes are the REST request bodies verbatim).
- `openclam://schema` — table/column map for `clam_sql`.

### Input and output

`clam_api` inputs are validated by the Zod schema in
`packages/mcp-server/src/server.ts`. The `body` parameter on POST/PUT is
forwarded verbatim to the daemon as the JSON body, so it matches the
procedure's input schema 1:1. Tool output is the response body as a JSON
string in a single `text` content block (see `tools.ts → successResult`).

### Auth

MCP tools don't carry auth themselves. The `DaemonClient` the MCP server was
constructed with already holds the token; the server passes it through to
the daemon on every request. Token sources, in order:

- `OPENCLAM_TOKEN` env var on the MCP server process
- Whatever credential the client config (e.g. Claude Desktop's
  `mcpServers` block) injected when launching the server

### Translation example (from the MCP primer)

Skill shows:

```
clam --json deals deal add --input-json '{"name":"Acme","stage":"Lead","value":50000}'
```

Over MCP:

```
clam_api(
  method = "POST",
  path   = "/api/deals/deals",
  body   = { "name": "Acme", "stage": "Lead", "value": 50000 }
)
```

---

## 4. HTTP API

Factory: `createApp(ctx)` in `packages/api/src/index.ts` returns a plain
`FetchHandler` — `(req: Request) => Promise<Response>`. Hono is no longer
involved; the daemon proxies raw fetch requests through this handler.

### Mounting

Procedures mount under `/api/<extensionId>/...`. The default prefix is the
extension's `id`. Two extensions override it with `pathPrefix: ''`:

- `core` — so user/role/rule/etc. live at `/api/users`, `/api/roles`, … not
  `/api/core/users`.
- `secrets` — so secret-vault routes live at `/api/secrets/...` without a
  doubled namespace.

This is enforced in `packages/extension-sdk/src/loader.ts`:

```ts
const prefix = ext.def.pathPrefix ?? `/${ext.def.id}`;
merged[ext.def.id] = walkProcedures(ext, ext.def.procedures, prefix);
```

The inner procedure path is declared on the procedure itself
(`defineProcedure({ method, path: '/users/{id}', ... })`) and must start
with `/`.

### Verb mapping

Each `defineProcedure` declares its own `method`. In practice the conventions
from `packages/core/src/procedures/` are:

| Purpose | Verb | Example |
|---------|------|---------|
| List | `GET` | `GET /api/users` |
| Get one | `GET` | `GET /api/users/{id}` |
| Create | `POST` | `POST /api/users` |
| Update | `PUT` | `PUT /api/users/{id}` |
| Soft-delete | `DELETE` | `DELETE /api/users/{id}` |
| Restore | `PUT` | `PUT /api/users/{id}/restore` |
| Hard-delete / purge | `DELETE` | `DELETE /api/users/{id}/purge` |
| Enable / disable / other state transitions | `PUT` | `PUT /api/rules/{slug}/enable` |
| Non-CRUD operations | `POST` | `POST /api/sql`, `POST /api/users/import` |

### Request body

For `POST`/`PUT`/`PATCH` with `Content-Type: application/json`, the parsed
body, merged with path and query params, is validated against the
procedure's `input` Zod schema. A mutation body may include a `_decision`
property — `extractDecision(body)` pulls it out before validation and
attaches it to `ctx.decision` for the handler.

### Auth (verified in `packages/api/src/index.ts`)

Three modes, resolved per-request:

1. `x-clam-auth-mode: daemon-jwt` with `x-clam-user-id` and
   `x-clam-key-permissions` — internal daemon trust boundary.
2. `x-clam-auth-mode: platform-trust` — the managed-platform edge injects
   this after verifying the caller. Grants `['*']` admin permissions.
3. Default path: `Authorization: Bearer <api-key>` →
   `apiKeyService.authenticateApiKey()` resolves the userId and the key's
   declared permission scope.

Effective permissions for an external bearer caller are:

```
rolePermissions ∩ apiKeyScopes     (scopes narrow roles)
```

The per-procedure `permission` declared in `defineProcedure` is checked
inside the loader with `permissionMatches()`. `'*'` is the admin wildcard.

### Other headers and behavior

- `OPTIONS /api/*` — CORS preflight. Origin allowlist is `OPENCLAM_CORS_ORIGIN`.
- `/health` — no auth, no rate limit, returns `{ status: 'ok' }`.
- `/api/openapi.json` — no auth, cached. Generated from every extension's
  `defineProcedure` entries via `generateOpenApiSpec`.
- Rate limit — sliding window per IP, `OPENCLAM_RATE_LIMIT_RPM` (default 300).
  Exceeding returns `429 { "error": "Too many requests. Try again later." }`.
- Security headers — `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`,
  `Referrer-Policy`, and `Strict-Transport-Security` in production.

---

## 5. Typed client

Package: `packages/platform-client/` — `createPlatformClient({ token, baseUrl })`
returns a `ContractRouterClient<typeof contract>` (oRPC). Implementation uses
`OpenAPILink`, so a typed call and a curl call hit the exact same REST
endpoint. The client is purely a compile-time guarantee layer.

The contract it speaks is the **platform** contract (managed-platform plane),
exported from `@openclam/contracts`, not a per-workspace API. Shape:

```ts
const client = createPlatformClient({ token: 'ocp_…' });
await client.workspaces.list({ organizationId: 'org-id' });
await client.backups.create({ slug: 'my-workspace' });
```

For the **per-workspace** API described in §4, there is no published typed
client today. Consumers hit it with:

- `fetch` / `curl` against `/api/...`
- The CLI (which builds either a `DirectClient` or an `HttpClient` via
  `createDirectClient` / `createHttpClient` in `@openclam/core`, see
  `apps/cli/src/index.ts`)
- The MCP `clam_api` tool (documented in §3)

### Auth

- Platform client: `Authorization: Bearer <token>` injected by `OpenAPILink`
  on every request. Token is an OpenClam Platform PAT (`ocp_…` prefix) or a
  BetterAuth session bearer.
- Workspace direct/HTTP client (CLI internal): same resolution chain as the
  CLI itself — token, env vars, session.

---

## 6. Worked example — creating a user

Procedure definition (lives in `packages/core/src/procedures/users.ts`):

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

Input fields (from `packages/core/src/schemas/user.ts → insertUserSchema`):
`name`, `slug`, `type` (`human`|`agent`), `email?`, `is_active?`, `status?`,
`metadata?`.

### CLI

```bash
clam --json user add --input-json '{
  "name": "Alice Example",
  "slug": "alice",
  "type": "human",
  "email": "alice@example.com"
}'
```

Auth: `OPENCLAM_TOKEN`, `OPENCLAM_AGENT`, `--token`, or active session.

### MCP

```
clam_api(
  method = "POST",
  path   = "/api/users",
  body   = {
    "name": "Alice Example",
    "slug": "alice",
    "type": "human",
    "email": "alice@example.com"
  }
)
```

Auth: carried by the MCP server's `DaemonClient`.

### HTTP API (curl)

```bash
curl -X POST https://<daemon-host>/api/users \
  -H "Authorization: Bearer $OPENCLAM_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Alice Example",
    "slug": "alice",
    "type": "human",
    "email": "alice@example.com"
  }'
```

### Typed client (platform plane)

Only applies to platform-level resources. For a workspace-level user
creation there's currently no typed entry point; agents should use the CLI,
`clam_api`, or raw HTTP. The `workspaces`, `backups`, organization, and
billing calls on `client` are the ones exposed through `@openclam/contracts`.

---

## 7. Error model

Validation vs general errors, with the same structural shape across surfaces.

### Validation errors (Zod)

When input fails the procedure's `input` schema, the error envelope carries
per-field issues:

```json
{
  "error": "Validation failed",
  "issues": [
    { "field": "email", "message": "Invalid email", "code": "invalid_string" },
    { "field": "slug",  "message": "Slug must be lowercase, start with a letter...", "code": "custom" },
    { "field": "name",  "message": "Required", "code": "invalid_type" }
  ]
}
```

Recover by reading the `issues` array, fixing the named fields in
`--input-json` / body, and retrying.

### General errors

Everything else (not found, conflict, forbidden, state) returns:

```json
{ "error": "User not found: <id>" }
```

Standard codes are from `AppError` in `packages/extension-sdk/src/errors.ts`:

| Code | Status | Typical cause |
|------|--------|---------------|
| `BAD_REQUEST` | 400 | Business-rule failure |
| `UNAUTHORIZED` | 401 | Missing/invalid credentials |
| `FORBIDDEN` | 403 | Authenticated but missing permission |
| `NOT_FOUND` | 404 | Resource doesn't exist (or was purged) |
| `CONFLICT` | 409 | Uniqueness / optimistic lock |
| `PRECONDITION_FAILED` | 412 | State precondition not met |
| `UNPROCESSABLE` | 422 | Semantically invalid |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL` | 500 | Unhandled |
| `NOT_IMPLEMENTED` | 501 | Procedure missing |
| `UNAVAILABLE` | 503 | Temporary outage |

### Transport wrapping

| Surface | Success | Error |
|---------|---------|-------|
| CLI | JSON on stdout, exit 0 | JSON on stderr, non-zero exit |
| MCP | `{ content: [{ type: 'text', text: '<json>' }] }` | Same shape with `isError: true` |
| HTTP API | 2xx body | 4xx/5xx body, same JSON envelope |
| Typed client | Promise resolves | Promise rejects with oRPC error wrapping the envelope |

The body shape is the same everywhere — only the outer envelope differs.
When an agent sees `issues` it's a validation error; when it sees `error`
without `issues`, it's a general error that needs the recovery table in
the CLI SKILL.md.

---

## 8. See also

- `patterns.md` — service, table, pagination, soft-delete, and event
  conventions that power every procedure.
- `skills/openclam-cli/SKILL.md` — full CLI usage rules and per-domain
  references. The `--input-json` shapes there are the procedure input
  schemas verbatim, so they work unchanged on every surface in this file.
- `packages/mcp-server/src/mcp-primer.md` — the version of §3 that ships
  with the MCP server for on-the-fly translation.
