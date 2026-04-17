---
name: setup-workspace
description: >
  Create your first OpenClam workspace and user. Covers the interactive
  `clam setup` flow, `clam workspace new` for non-interactive creation,
  choosing a database dialect (SQLite, PGlite, Postgres), and creating the
  first user as a human or agent.
metadata:
  author: openclam
  version: 0.1.0
---

# First workspace

A workspace is the unit of isolation in OpenClam — its own database, its
own manifest, its own enabled extensions, its own users. Everything you do
with `clam` happens in the context of one active workspace. This page
covers the two ways to create one: `clam setup` (interactive, one-shot) and
`clam workspace new` (scriptable, supports `--input-json`).

## Interactive: `clam setup`

First-run experience on a fresh machine. Creates `~/.openclam/openclam.json`,
delegates to `clam workspace new` for the database and user, and then
offers optional password, telemetry opt-in, and skills install.

```bash
clam setup
```

It will ask, in order:

1. **Human or agent?** — determines whether to offer password prompts and
   telemetry opt-in. Agents get neither.
2. All of the prompts from `clam workspace new` (below).
3. **Set a login password?** (humans only, recommended) — stored hashed.
4. **Telemetry?** — defaults to OFF. Anonymous only; never your data.
5. **Install skills?** — delegates to `clam skills`. Covered in
   [skills.md](./skills.md).

If a workspace already exists, `clam setup` prints the current state and
tells you to use `clam workspace new` for another one.

## Non-interactive: `clam workspace new`

For agents or CI. Pass either named flags or a single `--input-json`. With
`--input-json`, every field lives in one JSON blob and flags are ignored.

```bash
clam --json workspace new --input-json '{
  "name": "My Project",
  "slug": "my-project",
  "dialect": "sqlite",
  "extensions": ["contacts", "deals", "tasks"],
  "userName": "Alice",
  "userIdentifier": "alice@example.com",
  "userType": "human"
}'
```

The same thing with flags:

```bash
clam --json workspace new \
  --name "My Project" --slug my-project \
  --dialect sqlite \
  --extensions contacts,deals,tasks \
  --user-name "Alice" --user-identifier alice@example.com --user-type human
```

Agent-owned workspace (no password, just an API key):

```bash
clam --json workspace new --input-json '{
  "name": "Ops",
  "slug": "ops",
  "dialect": "sqlite",
  "userName": "Agent",
  "userIdentifier": "agent@ops.local",
  "userType": "agent"
}'
```

All available flags: `--name`, `--slug`, `--dialect`, `--mode`, `--path`,
`--url`, `--auth-token`, `--extensions`, `--user-name`, `--user-identifier`,
`--user-type`, `--agent-name`, `--agent-identifier`, `--webhooks`,
`--access`, `--daemon-url`, `--input-json`.

## Choosing a database dialect

`--dialect` takes one of three values:

| Dialect | Mode | When to use |
|---|---|---|
| `sqlite` | `local` (default) | Fastest start. One file under `~/.openclam/workspaces/<slug>/data.db`. Perfect for single-user local dev. |
| `sqlite` | `remote` | Turso. Requires `--url libsql://...` and optionally `--auth-token`. |
| `pglite` | `local` | Embedded Postgres, file-based. Zero config, Postgres semantics. Good middle ground. |
| `postgres` | `local` | Local Postgres you manage. Requires `--url postgres://...`. |
| `postgres` | `remote` | Hosted Postgres (Neon, RDS, etc). Requires `--url`. |

Picking Postgres without the `@openclam/adapter-postgres` optional peer
installed will fail pre-flight with an install hint. Install it, or start
with SQLite and migrate later.

## First user: human vs agent

Every workspace needs at least one user. `--user-type` decides the shape:

- **`human`** — can run `clam login`, can set a password with `clam passwd`,
  appears in the interactive login picker. Created with an API key that
  goes into the encrypted credential vault automatically.
- **`agent`** — cannot `clam login` (blocked by type check). Gets an API
  key stored in the vault, keyed by `agentIdentifier`. Reference it via the
  `OPENCLAM_AGENT=<identifier>` env var when the agent runs.

You can add more users later with `clam user create`; `clam workspace new`
only creates the first one.

## Container deployment (optional)

Instead of running the database in-process, you can run it inside a Docker
container started by `clam container start`. Set `--access local-container`
and `--daemon-url http://localhost:3401` on `workspace new`. The CLI will
health-check the daemon, POST to `/bootstrap`, and store the returned API
key locally. Pick this when you want workspace isolation at the OS level.

## Notes and gotchas

- **Slug rules** — lowercase, alphanumeric, hyphens; must be unique across
  workspaces. Auto-suggested from `--name` if omitted.
- **Extensions** — `--extensions` takes a comma-separated list or a JSON
  array via `--input-json`. Infrastructure extensions (`cron`, `storage`,
  `workflows`) are always installed; only domain extensions are opt-in.
- **`clam workspace new` requires `clam setup` first** — unless called via
  `clam setup` itself, a missing config file errors out with a clear hint.
- **Listing and switching** — `clam workspace list`, `clam workspace use <slug>`,
  `clam workspace current`.

See `../surfaces.md` to translate these operations to MCP, HTTP, or the
typed client.
