---
name: operate-system
description: >
  Small utility commands for poking at a running OpenClam instance — `clam
  status`, `clam doctor`, `clam console`, `clam docs`. Use these when you
  need to answer "is my install healthy?", "what's enabled here?", or "open
  the admin UI."
metadata:
  author: openclam
  version: 0.1.0
---

# System utilities

Four small commands that surface health, inventory, and admin affordances for
a running OpenClam instance. None of them modify data.

## Operations

| Command        | What it does                                                                          |
| -------------- | ------------------------------------------------------------------------------------- |
| `clam status`  | Reports the active workspace, environment, connection, user, extensions, AI, storage. |
| `clam doctor`  | Runs health checks against the workspace DB; optionally applies fixes.                |
| `clam console` | Opens the admin console in your browser (starts the daemon if needed).                |
| `clam docs`    | Opens https://openclam.dev in your browser.                                           |

## `clam status`

A one-shot summary of where you are and what's wired up.

```
clam --json status                  # machine-readable
clam --markdown status              # compact key-value for agents
clam status                         # human-friendly colored output
```

Typical fields: active workspace slug, connection dialect/mode/provider,
active user, enabled extensions, daemon running state + port, configured AI
providers (embedding / completion / reranker), webhooks subsystem state.

Add `--records` to also print row counts per table. Useful for quickly
confirming "did that import actually land?"

## `clam doctor`

Schema- and integrity-level health checks. Good to run after upgrades or
when something feels off. With no flags it reports; with `--fix` it applies
every fix it finds; with `--scope` it narrows to specific scopes.

| Flag               | Purpose                                                                      |
| ------------------ | ---------------------------------------------------------------------------- |
| `--fix`            | Apply all fixes without prompting.                                           |
| `--status`         | Report-only mode; do not offer fixes.                                        |
| `--scope <list>`   | Comma-separated scopes (e.g. `core,contacts,cron`).                          |
| `--deep`           | Include FTS index and embedding queue integrity checks (slower).             |
| `--input-json`     | Non-interactive input.                                                       |

Typical things doctor catches: missing columns after an extension upgrade,
orphaned FK targets, FTS tables out of sync, stale embedding jobs,
infrastructure migrations pending (cron / storage / workflows / webhooks).

```
# Just tell me what's wrong
clam --json doctor --status

# Non-interactively fix core and cron
clam --json doctor --fix --scope core,cron

# Full deep pass, apply everything
clam doctor --fix --deep
```

## `clam console`

Opens the admin web console in your browser, authenticated as your active
user.

Under the hood: the command starts the daemon if it isn't running, registers
a one-time session code over the control socket, and opens
`http://localhost:<port>/auth/activate?code=<code>`. The code is single-use
and short-lived.

```
clam console                 # start daemon if needed + open browser
clam console --no-open       # print the URL without opening a browser
```

For container and remote workspaces the same flow runs against the container
daemon or the cloud URL respectively.

## `clam docs`

Minimal shortcut — shells out to `open` / `xdg-open` / `start` pointing at
`https://openclam.dev`. No flags, no network from the CLI itself.

```
clam docs
```

## See also

See `../surfaces.md` to translate these operations to MCP, HTTP, or the typed
client.
