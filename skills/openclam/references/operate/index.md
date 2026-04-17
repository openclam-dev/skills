---
name: operate-index
description: >
  Ongoing operation of an OpenClam instance — workspace lifecycle, daemon
  control, identity and access, decision capture, cross-domain primitives, and
  read-path queries. Pick the file below that matches what you're trying to do.
  First-time initialization lives in `../setup/`; this directory only covers
  day-to-day use.
metadata:
  author: openclam
  version: 0.1.0
---

# Operate

Day-to-day surfaces once OpenClam is installed and a workspace exists. The
canonical surface for every command below is the CLI (`clam ...`); the same
operations are reachable via MCP, HTTP, and the typed client — see
`../surfaces.md` for the translation rules.

All commands documented here assume an active workspace (`clam workspace
current` returns a row). Commands requiring the daemon note it explicitly.

## Routing

| File                               | Use when you want to…                                                                                             |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| [workspace.md](./workspace.md)     | Create, list, switch, fork, rename, or delete workspaces; manage container mode; deploy to cloud; back up; reset. |
| [daemon.md](./daemon.md)           | Check daemon status, tail logs, restart/stop, reload, tune event processing.                                      |
| [system.md](./system.md)           | Run `clam status`, `clam doctor`, open the admin console, or jump to docs.                                        |
| [auth.md](./auth.md)               | Log in/out, rotate credentials, mint tokens, manage users, roles, API keys, permissions.                          |
| [decisions.md](./decisions.md)     | Attach rationale to mutations via `_decision`; list, inspect, and revise decisions.                               |
| [primitives.md](./primitives.md)   | Use cross-domain primitives — `project`, `tag`, `field`, `log` — that every extension shares.                     |
| [query.md](./query.md)             | Read across the whole system — SQL, text/semantic/hybrid search, AI Q&A, embedding queue.                         |

## Global flags on every `clam` command

| Flag                    | Purpose                                                                                    |
| ----------------------- | ------------------------------------------------------------------------------------------ |
| `--json`                | Compact JSON output (required for agents and scripting).                                   |
| `--markdown`            | Markdown tables / key-value blocks. Lower token count than JSON.                           |
| `--csv`                 | CSV output for list commands.                                                              |
| `--token <api-key>`     | Per-invocation API key; alternative to `OPENCLAM_TOKEN`.                                   |
| `--no-telemetry`        | Disable anonymous telemetry for this invocation.                                           |
| `--input-json '<json>'` | Subcommand-level flag on most mutations — pass the full input as a single JSON object.     |

Environment variables: `OPENCLAM_TOKEN` (session token), `OPENCLAM_AGENT`
(agent credential lookup), `OPENCLAM_VAULT_KEY` (override vault key path),
`OPENCLAM_DEBUG` (print registration errors).

## Paginated list response

Every `list` command returns the same shape:

```json
{ "data": [ ... ], "nextCursor": "<string|null>", "hasMore": true }
```

Pass `nextCursor` back as `cursor` in `--input-json` to walk pages; stop when
`hasMore` is `false`. Use `--all` on commands that support it to skip
pagination entirely.

## Canonical surface note

Every operation here has a CLI command. MCP tool names, HTTP routes, and typed
client methods are all generated from the same underlying procedure. When
working outside the terminal, see `../surfaces.md` to translate these
operations to MCP, HTTP, or the typed client.
