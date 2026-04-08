# OpenClam Skills

Agent skills for working with [OpenClam](https://github.com/openclam-dev/openclam) — a data and execution layer for AI agents.

Skills follow the [Agent Skills](https://agentskills.io) open standard and work with Claude Code and other compatible AI tools.

## Skills

| Skill | Purpose |
|---|---|
| [openclam-cli](./openclam-cli) | Reference for the OpenClam CLI — setup, workspaces, extensions, execution layer (events, rules, actions, workflows, cron, webhooks), and cross-cutting patterns |
| [openclam-build-extension](./openclam-build-extension) | Build custom OpenClam extensions — scaffolding, schema design, service patterns, routes, commands, migrations |

## Install

### Claude Code

Clone into your personal skills directory:

```bash
git clone https://github.com/openclam-dev/skills.git ~/.claude/skills/openclam
```

Or install a single skill:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/openclam-dev/skills.git /tmp/openclam-skills
cp -R /tmp/openclam-skills/openclam-cli ~/.claude/skills/
```

### Other tools

Each skill is a directory containing a `SKILL.md` file plus optional reference material. Point your agent runtime at the directory you want to use.

## License

MIT
