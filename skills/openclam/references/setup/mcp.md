---
name: setup-mcp
description: >
  Wire up the OpenClam MCP server in an agent runtime so agents can call
  OpenClam operations as tools. Covers the `clam-mcp` stdio binary, the
  daemon-hosted streamable-HTTP endpoint, per-runtime config, and auth.
metadata:
  author: openclam
  version: 0.1.0
---

# MCP server

MCP (Model Context Protocol) lets agents call OpenClam as a set of typed
tools instead of shelling out. It exposes three tools — `clam_status`,
`clam_api` (generic REST passthrough to the daemon), and `clam_sql`
(read-only SQL) — plus resources like `openclam://mcp-primer` and per-skill
references. Everything routes through the daemon, so the daemon must be
running.

Two ways to run the server:

- **stdio** — the `clam-mcp` binary from `@openclam/mcp-service`, spawned
  per-session by the agent. Best fit for local CLIs like Claude Code,
  Cursor, and Codex.
- **Streamable HTTP** — served by `@openclam/daemon` at `/mcp`. Use this
  for remote agents or when stdio isn't practical.

## Stdio (recommended for local)

The `clam-mcp` binary reads OpenClam's CLI config, resolves a daemon URL
and credential, and bridges stdio ↔ daemon. It works with zero
configuration if your active workspace is local and you're logged in.

### Claude Code

Add to `~/.claude/claude_desktop_config.json` (or the project
`.mcp.json`):

```json
{
  "mcpServers": {
    "openclam": {
      "command": "clam-mcp"
    }
  }
}
```

Or, from within a cloned monorepo:

```json
{
  "mcpServers": {
    "openclam": {
      "command": "tsx",
      "args": ["apps/mcp-service/src/index.ts"]
    }
  }
}
```

### Codex / Cursor / other stdio hosts

Same binary, whichever file that host reads. Most hosts use the same
`command` + `args` shape as Claude Code.

### Env-based override (bypass CLI config)

When you want the MCP server to hit an explicit daemon — CI, remote
machines, a non-default workspace — set both env vars and `clam-mcp`
skips the CLI config lookup:

```bash
export OPENCLAM_DAEMON_URL=https://daemon.example.com
export OPENCLAM_TOKEN=<pre-exchanged-jwt>
clam-mcp
```

`OPENCLAM_TOKEN` is treated as a pre-exchanged JWT in this mode (no
key-for-token swap).

## Streamable HTTP (daemon-hosted)

The running daemon exposes MCP directly at `POST /mcp` on its HTTP port
(default 3400). Use this when an agent isn't local to the daemon.

Point the agent at `http://localhost:3400/mcp` (or the remote daemon's
URL) with an `Authorization: Bearer <jwt>` header. Mint a long-lived JWT
with:

```bash
clam --json token mint --ttl 30d
clam --json token mint --workspace <slug> --ttl 90d
```

Exact per-agent config varies; the daemon speaks the MCP streamable-HTTP
transport spec and needs nothing OpenClam-specific from the client.

## Auth resolution (stdio)

`clam-mcp` walks through these in order:

1. `OPENCLAM_DAEMON_URL` + `OPENCLAM_TOKEN` env — both set → used as-is.
2. Active workspace from `~/.openclam/openclam.json`. Daemon URL from
   the workspace manifest. Credential:
   - `OPENCLAM_AGENT=<identifier>` → agent API key from the vault
   - else active user session → session API key from the vault
   - else (remote workspace) → platform access token
3. No credential → prints an actionable error on startup.

The credential is swapped at the daemon's `/auth/exchange` endpoint for a
short-lived JWT before any tool call goes out. Pre-exchanged tokens
(from case 1) skip that step.

## Per-workspace scoping

The exchange is workspace-scoped by workspace ID, which is read from the
active workspace's manifest. If you want the MCP server to target a
different workspace than the CLI's active one, either:

- `clam workspace use <slug>` before starting the agent session, or
- set `OPENCLAM_DAEMON_URL` + `OPENCLAM_TOKEN` manually.

## Verify

From the agent side:

- Call `clam_status` — returns the workspace, dialect, enabled extensions.
- Read `openclam://mcp-primer` — a resource that explains the `clam_api`
  tool's command-to-route mapping.
- Try `clam_api` with `method=GET`, `path=/api/contacts/people` — should
  return a page of contacts (or `[]` for an empty workspace).

If you get a 401, the daemon isn't reading the token. If startup dies
with "No active workspace", run `clam workspace use <slug>` or set the
env vars.

## Notes

- **The daemon must be running.** `clam daemon status` to check; see
  [daemon.md](./daemon.md) to start it.
- **Read-only SQL only.** `clam_sql` rejects `INSERT`, `UPDATE`, `DELETE`,
  DDL. Use `clam_api` for writes.
- **Resources before tools.** For `clam_api`, agents should read
  `openclam://mcp-primer` first, then `openclam://skill/<domain>` for the
  specific command's input shape. The skill references are the same
  content shipped with the `openclam` skill — reading them via MCP saves
  agents from needing the skill installed locally.

See `../surfaces.md` for the full invocation surface (CLI, MCP, HTTP, typed
client) and how operations map across them.
