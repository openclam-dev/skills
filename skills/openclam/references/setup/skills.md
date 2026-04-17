---
name: setup-skills
description: >
  Install official OpenClam skills into one or more agent runtimes — Claude
  Code, Codex, Gemini CLI, OpenCode, OpenClaw. Covers `clam skills`, global
  vs per-project install, the `OPENCLAM_SKILLS_REGISTRY` override, and
  alternate install paths.
metadata:
  author: openclam
  version: 0.1.0
---

# Skills

Skills are markdown files (one per skill, with YAML frontmatter) that teach
an agent the OpenClam vocabulary: exact CLI commands, file paths, extension
layout, event/rule/workflow patterns. Without them, the agent guesses from
whatever OpenClam detritus is in its training data. With them, fewer
hallucinated flags, fewer wrong paths, fewer follow-ups.

The `clam skills` command manages these installs: fetching from the
registry (`openclam-dev/skills` on GitHub by default), symlinking or
copying into each agent's skills directory, and keeping a lockfile so you
can see what's installed and what's drifted.

## Supported agent runtimes

OpenClaw (formerly Moltbot / Clawdbot) is first-class alongside the
others.

| Runtime | Global dir | Project dir |
|---|---|---|
| Claude Code | `~/.claude/skills` | `.claude/skills` |
| Codex (OpenAI) | `~/.codex/skills` | `.agents/skills` |
| Gemini CLI | `~/.gemini/skills` | `.agents/skills` |
| OpenCode | `$XDG_CONFIG_HOME/opencode/skills` | `.agents/skills` |
| OpenClaw | `~/.openclaw/skills` (falls back to `~/.clawdbot` / `~/.moltbot` if present) | `skills/` |

Global paths honor `CLAUDE_CONFIG_DIR`, `CODEX_HOME`, and `XDG_CONFIG_HOME`
env vars. The interactive installer detects which runtimes are installed
on your machine and pre-selects them.

## Interactive

```bash
clam skills
```

Prompts for:
1. Which skills to install (multiselect, pre-selects already-installed)
2. Scope — `global` (default) or `project`
3. Which agents — universal `.agents/skills` (Codex, Gemini CLI, OpenCode)
   plus Claude Code and OpenClaw as separate entries in project scope;
   each agent individually in global scope

## Non-interactive

Install one or many:

```bash
clam skills install openclam          # one skill
clam skills install openclam openclam-build-extension
clam skills install --all             # everything in the registry
```

Scope and agents:

```bash
clam skills install openclam -g                        # global (default)
clam skills install openclam --project                 # project-local
clam skills install openclam --agent claude-code       # only Claude Code
clam skills install --all --agent claude-code codex    # multiple agents
```

Useful flags: `--copy` (copy files instead of symlinking — slower but
survives `rm -rf` of the source), `--dry-run` (print what would happen),
`-y` / `--yes` (skip confirmation), `--registry <url>` (override).

List and diff:

```bash
clam skills list                   # what's installed, with drift status
clam skills list --remote          # what's in the registry
clam --json skills list
clam skills check                  # exits non-zero if any drift
clam skills check --quiet          # only non-zero exit, for CI
```

Update and remove:

```bash
clam skills update                 # pull latest for everything outdated
clam skills update openclam        # just one
clam skills update --dry-run
clam skills remove openclam        # uninstall from global scope (default)
clam skills remove openclam --project
```

## Global vs project

- **Global (`-g`, default)** — writes into the agent's home skills dir.
  Applies to every project on the machine. Good for skills you always
  want loaded (language references, OpenClam itself).
- **Project (`--project`)** — writes into the current directory's
  `.claude/skills`, `.agents/skills`, or `skills/` depending on agent.
  Use this when a skill is specific to one codebase and you want it
  committed to the repo.

Skills can coexist in both scopes — `clam skills list` will show
`installed: global, project` when that happens.

## Overriding the registry

By default the installer reads from `openclam-dev/skills#main` on GitHub
via the Git Trees and Blobs APIs (no local `git` dependency).

```bash
OPENCLAM_SKILLS_REGISTRY=myorg/custom-skills clam skills install --all
OPENCLAM_SKILLS_REGISTRY=myorg/custom-skills#develop clam skills install --all
clam skills install --all --registry myorg/custom-skills#v1.2.0
```

Set `GITHUB_TOKEN` (or `GH_TOKEN`) to lift the 60/hr unauthenticated
rate limit. Accepts `owner/repo[#ref]` or a full `https://github.com/owner/repo`
URL.

## Alternate install paths

If you don't have (or don't want) the OpenClam CLI:

- **Claude Code plugin marketplace:**
  `/plugin marketplace add openclam-dev/skills`
  `/plugin install openclam@openclam-skills`
- **`npx skills`** (Vercel Labs CLI):
  `npx skills add https://github.com/openclam-dev/skills`
- **`npx skillkit install openclam-dev/skills`**
- **Clone-and-copy:** `cp -r skills/skills/* .agents/skills/`
- **Git submodule:** for repos that want pinned updates

## Verifying

Start a fresh agent session and ask something OpenClam-specific:

> What's the `clam` command to add a person with their email?

If the skill loaded, it quotes the exact subcommand. If it's guessing or
asking for clarification, the skill isn't being picked up — check that
the expected skill files landed in the right directory for your runtime.

## Notes

- Symlinks are the default for speed and auto-update; pass `--copy` when
  your agent runtime can't follow symlinks (rare).
- `clam skills check` exits non-zero when any installed skill is outdated
  or removed upstream — useful in CI to catch drift.
- The setup flow (`clam setup`) offers to run the interactive skills
  install at the end. It's the same code path as bare `clam skills`.

See `../surfaces.md` to translate these operations to MCP, HTTP, or the
typed client.
