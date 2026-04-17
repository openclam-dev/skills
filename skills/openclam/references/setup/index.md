---
name: setup-index
description: >
  Setup orientation — start here. Ordered checklist that links to each setup
  topic. Use when the user is installing OpenClam for the first time or asks
  "how do I get started?"
metadata:
  author: openclam
  version: 0.1.0
---

# Setup

Bringing OpenClam up on a new machine takes about five minutes and seven
steps. Work through them in order: the later steps assume the earlier ones
are done. On a fresh machine, `clam setup` drives most of 2–7 interactively;
the individual commands below are what agents and scripts reach for when
they need to do one piece non-interactively.

## Checklist

1. [Install the CLI](./install.md) — `clam --version` works
2. [Create your first workspace](./workspace.md) — `clam setup` or `clam workspace new`
3. [Install the daemon as a service](./daemon.md) — `clam daemon start` registers launchd / systemd
4. [Configure AI providers](./ai-providers.md) — `clam ai` for embedding, completion, reranker
5. [Initialize the secret vault](./secrets.md) — `clam secrets init`
6. [Install skills into your agent runtime](./skills.md) — `clam skills install --all`
7. [Wire up the MCP server](./mcp.md) — the agent can now call OpenClam as tools

## What `clam setup` covers for you

Running `clam setup` once on a new machine does:

- Step 2 (creates the first workspace by delegating to `clam workspace new`)
- Optional password for the first human user
- Telemetry opt-in (defaults OFF)
- Optional skills install (step 6)

It does not start the daemon, configure AI providers, or initialize the
vault. Those are separate deliberate actions.

## Shape of each page

Every page here follows the same pattern: one-paragraph overview, concrete
`clam` commands with `--json` / `--input-json` where applicable, then
notes. Commands work locally and against remote daemons the same way.

See `../surfaces.md` to translate these operations to MCP, HTTP, or the
typed client.
