# OpenClam Skills

A collection of agent skills for working with [OpenClam](https://github.com/openclam-dev/openclam) — a system of record and execution layer for AI agents. Built so your AI coding assistant has the same playbook a human OpenClam developer would.

## What are skills?

Skills are markdown files (`SKILL.md`) with YAML frontmatter that follow the [Agent Skills](https://agentskills.io) open standard. Once installed, your AI agent loads them automatically when the conversation matches their description, or you invoke them directly with `/skill-name`.

Without these skills, your agent guesses from whatever scraps of OpenClam appear in its training data. With them, it knows the exact CLI commands, file paths, extension layout, and execution-layer patterns. The result: fewer hallucinated commands, fewer wrong file paths, and far less back-and-forth correcting it.

## Available skills

| Skill | Description |
|---|---|
| [openclam-cli](./skills/openclam-cli) | Reference for the OpenClam CLI — setup, workspaces, contacts, deals, tasks, messaging, marketing, content, calendar, research, storage, tickets, wiki, the execution layer (events, rules, actions, workflows, cron, webhooks), and cross-cutting patterns |
| [openclam-build-extension](./skills/openclam-build-extension) | Build custom OpenClam extensions — scaffolding, schema design, service patterns, routes, commands, migrations, and lifecycle |

## Installation

### Option 1: OpenClam CLI (recommended for OpenClam users)

If you have the OpenClam CLI installed, this is the easiest path:

```bash
clam skills install --all              # install every skill globally
clam skills                            # interactive picker
clam skills check                      # report drift vs. this repo
clam skills update                     # pull updates
```

`clam skills` detects which agents you have installed (Claude Code, Codex, Gemini CLI, OpenCode, OpenClaw) and writes to each one's skills directory. See the [main OpenClam README](https://github.com/openclam-dev/openclam#skills) for full options.

### Option 2: Claude Code Plugin

If you use Claude Code, the official plugin marketplace is the cleanest install:

```
/plugin marketplace add openclam-dev/skills
/plugin install openclam@openclam-skills
```

Skills become invokable as `/openclam:openclam-cli` and `/openclam:openclam-build-extension`. Updates ship automatically when you `/plugin update openclam`.

### Option 3: `npx skills`

The open-source [`skills` CLI](https://github.com/vercel-labs/skills) from Vercel works with any GitHub-hosted skill repo and supports a broad range of agent runtimes:

```bash
npx skills add openclam-dev/skills                              # install everything
npx skills add openclam-dev/skills --skill openclam-cli         # one specific skill
npx skills check                                                # see what's installed
npx skills update                                               # pull updates
```

### Option 4: SkillKit (multi-agent installer)

[SkillKit](https://www.npmjs.com/package/skillkit) is another general-purpose skills installer:

```bash
npx skillkit install openclam-dev/skills
npx skillkit install openclam-dev/skills --skill openclam-cli
npx skillkit install openclam-dev/skills --list
```

### Option 5: Clone and copy

Manual install for any runtime that reads `SKILL.md` files from a directory:

```bash
git clone https://github.com/openclam-dev/skills.git /tmp/openclam-skills

# Claude Code (global)
mkdir -p ~/.claude/skills
cp -R /tmp/openclam-skills/skills/* ~/.claude/skills/

# Codex
mkdir -p ~/.codex/skills
cp -R /tmp/openclam-skills/skills/* ~/.codex/skills/

# Gemini CLI
mkdir -p ~/.gemini/skills
cp -R /tmp/openclam-skills/skills/* ~/.gemini/skills/

# OpenCode
mkdir -p "${XDG_CONFIG_HOME:-$HOME/.config}/opencode/skills"
cp -R /tmp/openclam-skills/skills/* "${XDG_CONFIG_HOME:-$HOME/.config}/opencode/skills/"

# OpenClaw
mkdir -p ~/.openclaw/skills
cp -R /tmp/openclam-skills/skills/* ~/.openclaw/skills/
```

### Option 6: Git submodule

Pin a specific version of the skills as a submodule of your project:

```bash
git submodule add https://github.com/openclam-dev/skills.git .agents/openclam-skills
ln -s .agents/openclam-skills/skills/openclam-cli .claude/skills/openclam-cli
ln -s .agents/openclam-skills/skills/openclam-build-extension .claude/skills/openclam-build-extension
```

Update with `git submodule update --remote .agents/openclam-skills`.

## Usage

Once installed, just ask your agent to do something OpenClam-related and it will reach for the relevant skill automatically. For example:

> "What `clam` command do I use to add a person with an email?"

> "Generate the scaffolding for a tickets extension with a `tickets` table and a `ticket.created` event."

> "Wire a rule that sends a Slack message whenever a deal is moved to the 'Won' stage."

If you prefer explicit invocation, slash commands also work:

```
/openclam-cli   how do I export contacts to CSV?
/openclam-build-extension   create a new extension called inventory
```

## Verifying installation

Start a new session in your agent and ask something like:

> "What's the `clam` command for adding a contact with their email?"

If the skill loaded, the agent will answer with the actual subcommand (`clam contacts person add "Name" --email ...`). If it's guessing or asking you to clarify, the skill isn't being picked up — check that the file landed in the right directory for your runtime.

## Updating

With the OpenClam CLI:

```bash
clam skills check     # see what's outdated
clam skills update    # pull the latest
```

With the Claude Code plugin:

```
/plugin update openclam
```

Manually:

```bash
cd ~/.claude/skills/openclam-cli   # or wherever you installed
git pull
```

## Contributing

We welcome new skills and improvements to existing ones. See [CONTRIBUTING.md](./CONTRIBUTING.md) for the full guide, including the SKILL.md frontmatter convention and the project's versioning policy.

## License

[MIT](./LICENSE) — use these however you want.
