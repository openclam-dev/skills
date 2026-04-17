---
name: operate-daemon
description: >
  Ongoing daemon control — status, logs, restart, stop, hot-reload, and
  per-workspace enable/disable. Plus event processing configuration via
  `clam events`. Daemon install and first start live in `../setup/`.
metadata:
  author: openclam
  version: 0.1.0
---

# Daemon

The OpenClam daemon is one long-running process per host that serves:

- An HTTP API on a configurable port (default `3400`).
- A Unix control socket at `~/.openclam/daemon.sock` that the CLI uses for
  fast local forwarding.
- The **event processor** — a loop that drains the `events` outbox, running
  registered handlers and rule actions.
- The **cron scheduler** — ticks `cron_jobs` across every enabled workspace.
- The **file watcher** — notices changes in storage bucket directories.
- The **workflow worker** — picks up pending `workflow_runs`.
- An **AI provider manager** with reference counting and idle disposal.

The daemon holds a **workspace pool**: it can service many workspaces
concurrently, each with its own DB connection, Hono app, cron jobs, and
optional AI provider. Which workspaces the daemon services is controlled by
`clam daemon enable <workspace>` / `disable <workspace>`.

## Lifecycle commands

> `clam daemon start` always installs a launchd (macOS) or systemd-user
> (Linux) service so the daemon survives reboots and auto-restarts on crash.
> There is no separate `run` command — use `--foreground` for debugging.

### `clam daemon`

| Sub                   | What it does                                                                    |
| --------------------- | ------------------------------------------------------------------------------- |
| `start`               | Install the OS service if missing, then start it. Idempotent.                   |
| `start --foreground`  | Run in the terminal instead of backgrounding (debug only).                      |
| `start -p <port>`     | Override HTTP port (default 3400).                                              |
| `stop`                | Gracefully stop the daemon (keeps the service unit so `start` resumes cleanly). |
| `restart`             | Stop + start. Accepts `-p <port>`.                                              |
| `reload`              | Hot-reload config + extensions without dropping the process.                    |
| `status`              | Show running state, PID, port, and which workspaces are enabled in the pool.    |
| `logs`                | Tail the daemon log.                                                            |
| `logs --tail <n>`     | Show the last N lines then follow.                                              |
| `logs -f`             | Follow mode (equivalent to `tail -f`).                                          |
| `enable <workspace>`  | Add a workspace to the daemon's active pool.                                    |
| `enable --all`        | Enable every workspace on this host.                                            |
| `disable <workspace>` | Remove a workspace from the pool (data untouched, just stops serving it).       |

### Typical checks

```
# Is it up and what's it serving?
clam --json daemon status

# Tail logs while reproducing an issue
clam daemon logs -f

# Make a manifest edit take effect without a bounce
clam daemon reload

# Enable a newly-created workspace in the pool
clam --json daemon enable acme
```

### Installed service unit paths

- macOS: `~/Library/LaunchAgents/com.openclam.daemon.plist`
- Linux: `~/.config/systemd/user/openclam-daemon.service`

Logs stream to `~/.openclam/daemon.log`. The PID file is at
`~/.openclam/daemon.pid`; the Unix control socket is at
`~/.openclam/daemon.sock`.

## HTTP API auth

Every HTTP request to the daemon needs:

```
Authorization: Bearer <api-key>
X-Clam-Workspace: <workspace-slug>   # Optional — targets a specific workspace
X-Clam-Env:       <env>              # Optional — targets a specific environment
```

Mint a long-lived JWT for non-CLI integrations (e.g. an MCP server) with
`clam token mint` — see [auth.md](./auth.md#tokens).

## Event processing config — `clam events`

Every workspace has its own event-processing knobs stored in the workspace
manifest. They control how aggressively the daemon drains the `events` table
outbox for that workspace.

| Sub     | Purpose                                                                                 |
| ------- | --------------------------------------------------------------------------------------- |
| `get`   | Show the current config (defaults: `enabled: true`, `processIntervalSeconds: 10`).      |
| `set`   | Update `enabled` and/or `processIntervalSeconds` (range 1–300s).                        |
| `reset` | Remove workspace-level overrides and fall back to defaults.                             |

> These are exposed under both `clam events` and `clam config events` (the
> latter nests the same command under the `config` group in help output).

```
# Read current config
clam --json events get

# Slow the processor to once every 60 seconds
clam --json events set --input-json '{"enabled":true,"processIntervalSeconds":60}'

# Turn off processing entirely (events still enqueue, just don't drain)
clam --json events set --enabled false

# Back to defaults
clam --json events reset
```

When processing is disabled you can still drain manually with
`clam event process --limit <n>` (see [query.md](./query.md) for the cross-
domain read-path commands; event CRUD itself is under the core event
procedures — see `../automate/` for automation semantics).

## Forwarding behavior

By default the CLI talks directly to the workspace DB for speed. For container
and remote workspaces, the CLI **forwards** to the daemon over HTTP instead —
completely transparent from the caller's perspective. The decision is made
per invocation based on the active workspace's `access` mode. You can force
direct-mode debugging with `OPENCLAM_DEBUG=1` to surface the forwarding
trace.

## See also

See `../surfaces.md` to translate these operations to MCP, HTTP, or the typed
client.
