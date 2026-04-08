# AGENTS.md

This file describes the contract this repository follows so any AI agent runtime can consume it.

## What this repository contains

A collection of [Agent Skills](https://agentskills.io) for working with [OpenClam](https://github.com/openclam-dev/openclam) — a system of record and execution layer for AI agents. Each skill is a directory under `skills/` containing a `SKILL.md` file plus optional reference material.

## Repository layout

```
.claude-plugin/
  marketplace.json      # Claude Code plugin marketplace catalog
  plugin.json           # Plugin manifest
skills/
  <skill-name>/
    SKILL.md            # Required — main instructions with YAML frontmatter
    references/         # Optional — supporting documentation loaded on demand
      *.md
README.md
CONTRIBUTING.md
LICENSE
AGENTS.md               # this file
CLAUDE.md               # symlink → AGENTS.md
```

## SKILL.md format

Every skill is a markdown file with YAML frontmatter between `---` markers, followed by free-form instructions.

```markdown
---
name: <kebab-case-name>
description: >
  What the skill does and when to use it. Front-load the key use case.
metadata:
  author: openclam
  version: 0.1.0
tags:
  - tag1
  - tag2
---

# Instructions

Free-form markdown content the agent reads when this skill is loaded.
```

**Fields read by Claude Code:** `name`, `description`. Other documented Claude Code fields (`disable-model-invocation`, `allowed-tools`, `model`, `effort`, `paths`, etc.) may also appear in individual skills.

**Project conventions (informational only):** `metadata.author`, `metadata.version`, `tags`. These are nested under `metadata:` rather than flat at the root.

## Discovery

Two patterns work for discovering skills in this repo:

- **Claude Code plugin** — read `.claude-plugin/marketplace.json` and `.claude-plugin/plugin.json`. The plugin's `source` is the repo root; skills are auto-discovered from the `skills/` directory.
- **Direct enumeration** — walk `skills/*/SKILL.md`. The directory name is the skill name. Each directory may contain additional supporting files referenced from `SKILL.md`.

## Install paths

This repo doesn't prescribe install paths; it relies on the consumer (CLI, plugin manager, or manual `git clone`) to copy skill directories into the appropriate location for the target runtime. See [README.md](./README.md) for the supported install methods.

## Versioning

Each skill is versioned independently via the `metadata.version` field in its `SKILL.md`. Skill version bumps follow [semver](https://semver.org/). Repository-level versioning uses git tags on the default branch.

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for the process and conventions.

## License

[MIT](./LICENSE).
