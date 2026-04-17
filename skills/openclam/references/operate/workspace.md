---
name: operate-workspace
description: >
  Ongoing workspace operations — create, list, switch, rename, fork, and
  delete workspaces; manage extensions and the webhooks subsystem; run the
  container or cloud deployment modes; create and restore backups; perform
  selective resets. First-time initialization (`clam setup`) lives in
  `../setup/`.
metadata:
  author: openclam
  version: 0.1.0
---

# Workspace

A workspace is an isolated OpenClam instance: its own database, its own set of
enabled extensions, its own users/roles, its own secrets vault. A single host
can hold many workspaces; exactly one is active at a time and every `clam`
invocation runs against that one unless `--env` or a `--workspace` flag
overrides it.

Workspaces have an **access mode** that controls where their data lives:

| Access mode       | DB location                             | How you reach it                                 |
| ----------------- | --------------------------------------- | ------------------------------------------------ |
| `local-direct`    | File on this machine (SQLite / PGlite / local Postgres) | CLI talks to the DB directly.                    |
| `local-container` | Inside a Docker/Podman container on this machine        | CLI forwards to the container daemon over HTTP.  |
| `remote`          | OpenClam Cloud (Fly machine + volume)   | CLI forwards to `https://<slug>.openclam.app`.   |

> **Note on "environments."** OpenClam does not currently expose multi-env
> subcommands (`clam env add`, `clam env use`, etc.). The manifest schema
> reserves space for environments and the `--env <env>` global flag is parsed,
> but environment lifecycle commands are **command surface pending**. Treat
> each workspace as a single environment today.

## Operations

### `clam workspace`

| Sub         | What it does                                                                 |
| ----------- | ---------------------------------------------------------------------------- |
| `new`       | Create a workspace. Interactive by default; fully scriptable via flags / `--input-json`. |
| `list`      | Paginated list of all workspaces on this host (incl. remote synced).         |
| `use <ref>` | Switch the active workspace. Accepts slug or `workspaceId`.                  |
| `current`   | Print the active workspace and its connection/webhooks config.               |
| `rename <old> <new>` | Rename (slug change). `old` may be slug or ID; `new` must be a slug. |
| `fork <src> <dst>`   | Clone manifest (and optionally data with `--with-data`) into a new slug.     |
| `remove <ref>`       | Soft-delete workspace (pairs with `restore`). Requires 6-digit OTP unless `--force`. `--dry-run` previews. |
| `extensions`         | Interactive or scripted — enable/disable extensions for the workspace.       |
| `webhooks enable\|disable` | Toggle outbound webhooks subsystem on the workspace.                  |
| `sync`               | Pull remote workspaces from OpenClam Cloud into the local config.            |
| `upgrade-security`   | One-time migration: approve capabilities for already-enabled extensions.     |

### Typical flows

```
# Create a new local SQLite workspace, interactive
clam workspace new

# Fully scripted workspace creation (agents)
clam --json workspace new --input-json '{
  "name": "Acme Research",
  "slug": "acme",
  "dialect": "sqlite",
  "extensions": ["contacts","deals","tasks"],
  "userName": "Alice",
  "userIdentifier": "alice@acme.com",
  "userType": "human"
}'

# List
clam --json workspace list --input-json '{"cursor":"0","limit":20}'

# Switch
clam --json workspace use acme

# Fork with data copy
clam --json workspace fork --input-json '{"source":"acme","target":"acme-staging","with_data":true}'

# Remove (interactive OTP) — soft-delete, pairs with `restore`
clam workspace remove acme
# Or non-interactive (careful):
clam --json workspace remove --input-json '{"slug":"acme","force":true}'
```

### Supported dialects/providers

| Dialect  | Mode   | Provider    | Where data lives                                        |
| -------- | ------ | ----------- | ------------------------------------------------------- |
| sqlite   | local  | —           | `~/.openclam/workspaces/<slug>/envs/<env>/data.db`      |
| sqlite   | remote | Turso (d1)  | `libsql://...`                                          |
| postgres | local  | —           | Local PG at the URL you provide                         |
| postgres | local  | pglite      | Embedded, file-based Postgres on disk                   |
| postgres | remote | Neon        | Hosted serverless Postgres                              |

## Extensions

`clam workspace extensions` is the interactive entry point; for scripting use
`--input-json`:

```
# Interactive toggle UI
clam workspace extensions

# Replace the exact set of enabled extensions
clam --json workspace extensions --input-json '{"extensions":["contacts","deals","tasks"]}'

# Incremental: add or remove
clam --json workspace extensions --input-json '{"enable":["tasks"],"disable":["deals"],"drop_disabled_tables":false}'
```

When disabling an extension you'll be asked whether to drop its tables.
Default (`"drop_disabled_tables": false`) preserves data so the extension can
be re-enabled later.

Webhooks is toggled separately because it adds its own subsystem:

```
clam --json workspace webhooks enable
clam --json workspace webhooks disable
```

## Container mode

Run a workspace inside a Docker or Podman container. The CLI transparently
forwards operations to a daemon inside the container. See `../setup/` for
first-time container install; ongoing commands:

```
clam container start                 # Build + start (default host port 3401)
clam container start --port 3500
clam container stop
clam --json container status
clam container logs                  # Follow
clam container logs --tail 100
```

To move an existing workspace into container mode, create a new workspace
with `--access local-container --daemon-url http://localhost:3401` and point
it at the same DB.

## Cloud deploy

`clam deploy` ships the active local SQLite workspace to OpenClam Cloud as a
managed `remote` workspace. Requires `clam login --remote` first.

```
clam deploy --schema-only              # Tables only, no rows
clam deploy --full                     # Tables + all data
clam deploy --slug acme-prod           # Override remote slug
clam deploy --force                    # Skip confirmation
```

After deploy the workspace's `access` becomes `remote` and the CLI transparently
forwards to `https://<slug>.openclam.app`.

## Backup

`clam backup` works on both local and remote workspaces:

| Sub                 | Purpose                                                                           |
| ------------------- | --------------------------------------------------------------------------------- |
| `add`               | Write an archive (local) or trigger a snapshot (remote).                          |
| `restore <archive>` | Restore from a local archive path or a remote backup ID.                          |
| `list <dir>`        | List archives in a directory.                                                     |
| `verify <archive>`  | Validate archive integrity without restoring.                                     |

Scopes (`--scope`): `workspace` (just the active one), `all-workspaces`
(every workspace on this host), `full` (everything including vault key, CLI
config, and custom extensions — see flags below).

```
# Single workspace archive to a directory
clam --json backup add --scope workspace --output ~/backups

# Full system backup including vault key (needed to restore on another host)
clam --json backup add --scope full --include-vault-key --verify

# Restore (local)
clam --json backup restore ~/backups/openclam-2026-04-16.tar.zst --dry-run
clam --json backup restore ~/backups/openclam-2026-04-16.tar.zst --force
```

Remote workspaces: `backup add` calls the platform API to trigger an
on-demand snapshot; `backup restore <id>` restores from a snapshot ID.

## Selective reset

`clam reset` interactively chooses what to nuke — local workspaces, container
workspaces, or both. It requires a 6-digit OTP confirmation and is
irreversible:

- **Local** — deletes every workspace directory under `~/.openclam/workspaces/`.
- **Container** — stops Docker/Podman, removes volumes, deletes container manifests.
- **Config** — if you reset everything, the global config is removed too.

There are no flags; reset is always interactive because the blast radius is
large.

## See also

See `../surfaces.md` to translate these operations to MCP, HTTP, or the typed
client.
