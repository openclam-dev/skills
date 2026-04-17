---
name: setup-install
description: >
  Install the OpenClam CLI. Covers prerequisites, supported install methods,
  PATH setup, and verifying the installation.
metadata:
  author: openclam
  version: 0.1.0
---

# Install the CLI

The `clam` CLI is the primary surface for OpenClam — everything the console,
MCP server, and HTTP API do is also available as a command. Installing it
puts two binaries on your PATH: `clam` (short form) and `openclam` (aliased
to the same entry point).

## Prerequisites

- **Node.js 18+** — the CLI declares `engines.node >= 18` and tests against
  Node 20. Verify with `node --version`.
- **pnpm** (only for the from-source install below) — install with
  `npm i -g pnpm` or `corepack enable`.
- **Postgres client libraries** (optional) — only needed if you plan to
  use the `postgres` or `pglite` dialect. The CLI ships without them by
  default and surfaces an install hint when you choose a Postgres dialect.
- **Ollama** (optional) — install separately from [ollama.com](https://ollama.com)
  if you plan to run local completion/embedding models.

## Install methods

### From source (current preferred path)

Until a published npm tarball / homebrew tap lands, the canonical install is
the monorepo build:

```bash
git clone https://github.com/openclam-dev/openclam.git
cd openclam
pnpm install
pnpm --filter @openclam/cli exec tsup
```

That produces `apps/cli/dist/index.js` and wires the `clam` / `openclam`
bins in `apps/cli/bin/`. Link them onto your PATH:

```bash
# pnpm already places them if you pnpm i -g from the apps/cli dir
cd apps/cli
pnpm link --global
```

Or add `apps/cli/bin` to your `PATH` directly in `.zshrc` / `.bashrc`.

### Claude Code plugin (skills only)

If you only want the skills and not the full CLI, the Claude Code plugin
marketplace is the cleanest path. It will not install the `clam` binary.

```
/plugin marketplace add openclam-dev/skills
/plugin install openclam@openclam-skills
```

## Verify

```bash
clam --version
clam --help
```

You should see version `0.1.0` (or later) and the full command tree. Then
run the health check — it works before a workspace exists and reports
what's missing:

```bash
clam doctor
```

`clam doctor --json` is the agent-friendly form.

## Troubleshooting

- **`command not found: clam`** — the bin directory isn't on your PATH.
  Check `pnpm bin -g` and add that to PATH, or re-run `pnpm link --global`
  from `apps/cli`.
- **`Cannot find module '@openclam/adapter-postgres'`** — you picked a
  Postgres-flavored dialect without the optional peer. Install Postgres
  peers explicitly: `pnpm add -w @openclam/adapter-postgres @openclam/provider-postgres-local`.
- **Old Node** — Node 16 will fail with an obscure syntax error. Upgrade
  to 18+.
- **Permission denied on bin** — `chmod +x apps/cli/bin/clam.js`.

## Next step

Once `clam --version` works, move on to [workspace.md](./workspace.md) to
create your first workspace.

See `../surfaces.md` to translate these operations to MCP, HTTP, or the
typed client.
