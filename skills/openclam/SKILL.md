---
name: openclam
description: >
  Use when the user runs or asks about any OpenClam commands (`clam`), uses
  OpenClam MCP tools, or calls the OpenClam HTTP API. Also use when the user
  asks about setting up or configuring OpenClam, checking its status, or
  working with any of its built-in extensions in the OpenClam context — or
  building custom extensions, custom functions, or custom workflows for use
  with OpenClam.
metadata:
  author: openclam
  version: 0.1.0
tags:
  - openclam
  - cli
  - mcp
  - http-api
  - operate
  - automate
  - extend
---

# OpenClam

OpenClam is a data and execution layer for AI agents: a system of record (contacts, deals, tasks, content, calendar, research, storage, tickets, wiki) plus an execution layer (events, rules, schedules, functions, workflows, webhooks). The same operations are reachable from four surfaces — CLI (`clam`), MCP tools, HTTP API, and a typed oRPC client.

## First-contact flow

Run through these steps before acting on any user ask. Each step feeds the next.

### 1. Confirm OpenClam is installed

```
clam --json status
```

- **`command not found` / not on PATH** → OpenClam isn't installed. Tell the user, ask if they want to install it, then load [`references/setup/install.md`](references/setup/install.md).
- **Runs but reports no workspace / not configured** → first-time setup needed. Load [`references/setup/index.md`](references/setup/index.md).
- **Runs cleanly** → capture from the output: is the daemon running, the active workspace slug, which extensions are enabled, which storage buckets exist. Continue.

### 2. Survey available workspaces

```
clam --json workspace list --fields workspaceId,slug,name
```

Lets you confirm the active workspace and see the alternatives.

### 3. Confirm which workspace to operate on

Mutations should always target a specific workspace. If an active workspace exists, ask the user:

> "Should I work against the active workspace `<slug>`, or a different one?"

For one-off calls you can also pass `--workspace <slug>` instead of switching the active workspace globally.

### 4. Establish identity

```
clam --json whoami
clam --json user list --fields id,name,type
```

- `whoami` tells you who is currently authenticated — usually a human user.
- `user list` shows every user in the workspace, including any `type=agent` service accounts.

If there is an agent account that represents this session, ask the human:

> "Should I act in my own capacity (as agent user `<name>`) or on your behalf (as `<human>`)?"

This matters for audit trails, `decisions` attribution, and RBAC. Once established, set `OPENCLAM_TOKEN` to the chosen identity's API key for subsequent calls.

### 5. Classify the intent and load the right reference

| User wants to… | Load |
|---|---|
| install, configure, or change setup (AI providers, secrets, daemon, MCP, skills) | [`references/setup/`](references/setup/) — start with `index.md` |
| work with data (contacts, deals, tasks, content, calendar, research, storage, tickets, wiki, ICP) | [`references/operate/`](references/operate/) — pick the domain file |
| query across domains (SQL, full-text/semantic/hybrid search, ask) | [`references/operate/query.md`](references/operate/query.md) |
| use cross-extension primitives (projects, tags, custom fields, activity log) | [`references/operate/primitives.md`](references/operate/primitives.md) |
| manage workspaces, users, roles, API keys, or capture decisions | `references/operate/workspace.md`, `auth.md`, `decisions.md` |
| wire declarative automation (events, rules, schedules, webhooks) | [`references/automate/index.md`](references/automate/index.md) |
| build, edit, or debug a **custom function** (TypeScript handler) | [`references/automate/functions.md`](references/automate/functions.md) |
| build, edit, or debug a **custom workflow** (multi-step TypeScript) | [`references/automate/workflows.md`](references/automate/workflows.md) |
| build, edit, or debug a **custom extension** (new tables/services/procedures) | [`references/extend/index.md`](references/extend/index.md) |
| translate an operation between CLI, MCP, HTTP, and the typed client | [`references/surfaces.md`](references/surfaces.md) |
| understand architecture patterns, pagination, soft-delete, events | [`references/patterns.md`](references/patterns.md) |

If the user hasn't expressed intent yet, ask — don't guess.

## Non-negotiables

These apply on every call, regardless of surface.

- **Always `--json`.** Pass it before the subcommand: `clam --json <command> …`. Never parse human-readable output.
- **Use `--input-json '{...}'`** for complex inputs. Don't spread them across many individual flags.
- **Auth via `OPENCLAM_TOKEN`** env var. Set it in the agent environment; don't hardcode tokens. Full resolution order in [`references/surfaces.md`](references/surfaces.md).
- **Never guess table or column names.** Look them up in the domain reference, or introspect with `clam --json sql "PRAGMA table_info(<table>)"`.
- **Record rationale on mutations** with a `_decision` envelope when AI-driven. Shape in [`references/operate/decisions.md`](references/operate/decisions.md).
- **Only enabled extensions have populated tables.** If `status` shows an extension disabled, don't query its tables or call its procedures.

## Error envelope

All surfaces return `{ "error": { "code": "...", "message": "..." } }`. Retry only transient errors (network, `RATE_LIMITED`). Do not retry `NOT_FOUND`, `UNAUTHORIZED`, `FORBIDDEN`, or `VALIDATION_ERROR`.
