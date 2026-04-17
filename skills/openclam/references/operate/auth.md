---
name: operate-auth
description: >
  Identity and access control — logging in/out, switching users, setting
  passwords, minting tokens, managing API keys, users, and roles with their
  permission sets. OpenClam has two user types (human and agent) and a
  role-based permission model with wildcard support.
metadata:
  author: openclam
  version: 0.1.0
---

# Auth and identity

OpenClam has two user types, both backed by API keys:

| Type    | How they authenticate                                                                      |
| ------- | ------------------------------------------------------------------------------------------ |
| `human` | Interactive login with password via `clam login`. First activation uses a 6-digit setup code. Session is stored in an encrypted credential vault. |
| `agent` | Always uses an API key (no password, no login flow). Key stored in vault or provided via `OPENCLAM_TOKEN` / `--token`. |

Every user gets an API key as the universal internal credential. Humans *also*
have a login password for interactive sessions.

## Auth resolution order

When a CLI invocation needs a credential:

1. `--token <api-key>` flag on the command.
2. `OPENCLAM_TOKEN` env var.
3. `OPENCLAM_AGENT` env var — looks up the named agent's key in the vault.
4. Active user session (set by `clam login`).
5. Fail with "no credential found."

## Operations overview

| Command                       | What it does                                                             |
| ----------------------------- | ------------------------------------------------------------------------ |
| `clam login`                  | Interactive login picker (humans only). Remote: opens browser for OAuth. |
| `clam login <identifier>`     | Direct login. Prompts for password if one is set.                        |
| `clam login <id> --code <c>`  | First-time activation using a setup code; prompts to set a password.     |
| `clam login --remote`         | Log in to OpenClam Cloud via browser OAuth.                              |
| `clam login --token <key>`    | Re-login using an existing API key.                                      |
| `clam logout`                 | Clear the active session (revoke on remote).                             |
| `clam whoami`                 | Print the active user with their DB record.                              |
| `clam passwd`                 | Set or change the login password (human users only).                     |
| `clam token mint`             | Issue a long-lived JWT for external integrations (e.g. MCP).             |
| `clam user`                   | CRUD on workspace users.                                                 |
| `clam role`                   | CRUD on roles and role assignments.                                      |
| `clam api-key`                | CRUD on API keys for a user.                                             |

## `clam login` and siblings

Humans:

```
# Pick from existing humans in the active workspace
clam login

# Direct login
clam login alice@acme.com

# First-time activation (admin just ran `clam user add --type human`)
clam login alice@acme.com --code 482913
```

Agents cannot use `clam login`. Either set `OPENCLAM_AGENT=<agent-slug>` in
the environment (the vault lookup is automatic) or pass `--token` on every
invocation.

Remote (OpenClam Cloud): `clam login --remote` opens a browser OAuth flow and
persists the remote account credential separately from workspace credentials.
Remote logout revokes the server-side session.

### Sessions & storage

- Sessions are persisted in an encrypted credentials store at
  `~/.openclam/credentials.enc` (AES-256-GCM with scrypt-derived keys).
- OS keychain is preferred when available, with the encrypted file as
  fallback.
- Session credentials are **scoped per workspace** so switching workspaces
  picks up the right session automatically.
- All auth events are logged to the `log` table with `source='auth'` — see
  [primitives.md](./primitives.md#log).

## Tokens — `clam token mint`

Long-lived JWTs for integrations that can't run the CLI (e.g. a remote MCP
server). Trades an API key for a signed token via the daemon's
`/auth/exchange` endpoint.

```
clam --json token mint                           # 30d default, active workspace
clam --json token mint --ttl 7d
clam --json token mint --ttl 365d
clam --json token mint --workspace acme --daemon-url http://localhost:3400
```

TTL format: `<number><s|m|h|d>` (e.g. `30d`, `8h`). Requires the daemon
running.

## API keys — `clam api-key`

Per-user, per-purpose API keys. A user's first key is created automatically
on workspace creation; use these commands to issue additional ones.

| Sub                  | Purpose                                                            |
| -------------------- | ------------------------------------------------------------------ |
| `add <name>`         | Mint a new key. Shown once — copy it immediately.                  |
| `list`               | List keys for the current user (or `--user <identifier>`).         |
| `count`              | Count keys.                                                        |
| `get <id>`           | Details for one key.                                               |
| `revoke <id>`        | Disable a key.                                                     |
| `rotate <id>`        | Revoke old key, create a replacement, return the new secret once.  |

```
# New key with inherited permissions, 90-day expiry
clam --json api-key add automation --input-json '{
  "permissions": null,
  "expiresAt": "2026-07-15T00:00:00Z"
}'

# Key scoped down to just read operations
clam --json api-key add reader-only --input-json '{
  "permissions": ["contacts.person:read","deals.deal:read"]
}'

# List with --all to skip pagination
clam --json api-key list --user alice@acme.com --all
```

Service: `apiKeyService` in `@openclam/core` — methods `create`, `list`,
`count`, `get`, `revoke`, `rotate`.

## Users — `clam user`

`human` vs `agent` determines the credential model but both live in the same
`users` table:

| users column   | Notes                                                            |
| -------------- | ---------------------------------------------------------------- |
| `id`           | UUID                                                             |
| `name`         | Display name                                                     |
| `type`         | `human` or `agent`                                               |
| `identifier`   | Email for humans, slug for agents — UNIQUE                       |
| `is_active`    | Boolean                                                          |
| `secret_hash`  | SHA-256 hash of API key                                          |
| `password_hash`| bcrypt hash of login password (humans only)                      |
| `metadata`     | JSON blob                                                        |
| `deleted_at`   | Soft delete timestamp                                            |

Service: `userService` — `add`, `list`, `get`, `getByIdentifier`, `search`,
`update`, `deactivate`, `remove` (soft), `restore`, `purge` (hard),
`rotateKey`.

```
# Create an agent user (returns api_key once, stored in vault)
clam --json user add 'Research Agent' --type agent --identifier research-agent-01

# Create a human user (returns setup_code with 15-min expiry)
clam --json user add 'Alice Smith' --type human --email alice@acme.com

# List humans only
clam --json user list --type human --limit 25

# Rotate an agent's API key
clam --json user rotate-key research-agent-01

# Generate a fresh setup code for a human
clam --json user invite alice@acme.com

# Soft delete
clam --json user remove alice@acme.com

# Hard delete (irreversible)
clam --json user purge <user-id>

# See which keys a user has (with stale filter)
clam --json user list-keys alice@acme.com --stale --days 90
```

`clam user import` / `clam user export` handle CSV/JSON for bulk migrations.

## Roles and permissions — `clam role`

OpenClam uses role-based permissions with **resource:action** strings and
wildcards.

| roles column | Notes                                                                        |
| ------------ | ---------------------------------------------------------------------------- |
| `id`         | UUID                                                                         |
| `slug`       | UNIQUE                                                                       |
| `name`       | Display name                                                                 |
| `is_system`  | System roles (`admin`) cannot be modified or deleted                         |
| `deleted_at` | Soft delete                                                                  |

### Permission strings

Format: `<resource>:<action>`. Both parts accept the `*` wildcard.

Examples:

- `*` — everything (what the system `admin` role holds).
- `contacts.person:*` — all actions on person records.
- `contacts.person:read` — read-only on person.
- `deals.deal:create`, `deals.deal:update` — narrower.
- `api_key:create` — permission governing `clam api-key add` (permission string is still `:create`).

List every permission the runtime knows about:

```
clam --json role permissions
```

### Role commands

| Sub                          | Purpose                                                              |
| ---------------------------- | -------------------------------------------------------------------- |
| `add <slug>`                 | Create a role with an initial permission list.                       |
| `list`                       | Paginated.                                                           |
| `get <slug>`                 | One role with its full permission set.                               |
| `update <slug>`              | **Replaces** all permissions (pass full list).                       |
| `remove <slug>`              | Soft delete (blocked for system roles).                              |
| `restore <slug>` / `purge <slug>` | Restore soft-deleted / hard delete.                             |
| `assign <user-id> <slug>`    | Assign role to user.                                                 |
| `revoke <user-id> <slug>`    | Revoke.                                                              |
| `permissions`                | List all available permission strings.                               |

```
# Create a custom role
clam --json role add researcher --input-json '{
  "name": "Researcher",
  "permissions": [
    "contacts.person:*",
    "contacts.org:*",
    "research.*:*",
    "tag:read",
    "log:read"
  ]
}'

# Add a permission (remember: update REPLACES the full list)
clam --json role update researcher --input-json '{
  "permissions": [
    "contacts.person:*","contacts.org:*","research.*:*",
    "tag:read","log:read","deals.deal:read"
  ]
}'

# Assign to a user
clam --json role assign alice@acme.com researcher

# Revoke
clam --json role revoke alice@acme.com researcher
```

Service: `roleService` — `add`, `list`, `get`, `update`, `remove`, `restore`,
`purge`, `assign`, `revoke`, `listForUser`, `hasPermission`, `ensureAdmin`.

At runtime, a user's effective permissions are the union across all assigned
roles. `roleService.hasPermission(db, tables, userId, 'contacts.person:create')`
resolves wildcards automatically.

## Typical TypeScript signatures

```ts
// userService
add(db, tables, input: { name, type, identifier, ... }, ctx?): Promise<User>
list(db, tables, filters?: { type, active }, pagination?): Promise<PaginatedResult<User>>
getByIdentifier(db, tables, identifier: string): Promise<User | null>
rotateKey(db, tables, identifier: string, ctx?): Promise<{ api_key: string }>
remove(db, tables, id: string, ctx?): Promise<void>  // soft
purge(db, tables, id: string, ctx?): Promise<void>   // hard

// roleService
add(db, tables, input: { slug, name?, permissions? }, ctx?): Promise<Role>
update(db, tables, slug: string, permissions: string[], ctx?): Promise<Role>
assign(db, tables, userId: string, roleSlug: string): Promise<void>
hasPermission(db, tables, userId: string, permission: string): Promise<boolean>

// apiKeyService
create(db, tables, input: { userId, name, permissions, expiresAt }): Promise<{ key, record }>
rotate(db, tables, id: string): Promise<{ key, record }>
revoke(db, tables, id: string): Promise<void>
```

## See also

- [decisions.md](./decisions.md) — every mutation listed here accepts an
  optional `_decision` field for audit / rationale capture.
- [primitives.md](./primitives.md#log) — auth events land in the `log` table
  with `source='auth'`.

See `../surfaces.md` to translate these operations to MCP, HTTP, or the typed
client.
