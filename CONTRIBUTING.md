# Contributing to OpenClam Skills

Thanks for your interest in contributing. This repository hosts the official skills for the OpenClam CLI and ecosystem. Skills are markdown files (`SKILL.md`) with YAML frontmatter that follow the [Agent Skills](https://agentskills.io) open standard.

## Adding a new skill

1. **Fork the repo** and create a branch for your skill.
2. **Add a directory** under `skills/` with a kebab-case name (e.g. `skills/openclam-search`).
3. **Create a `SKILL.md`** at the root of that directory with the required frontmatter (see below).
4. **Add reference material** if needed under `references/` inside the skill directory. Keep `SKILL.md` itself focused; offload long content to references.
5. **Bump `metadata.version`** if you're updating an existing skill.
6. **Open a pull request** with a short description of what the skill does and why.

## SKILL.md frontmatter convention

OpenClam skills use this frontmatter shape:

```yaml
---
name: your-skill-name
description: >
  Short, action-oriented description that tells an agent when to use this skill.
  Front-load the key use case in the first sentence.
metadata:
  author: openclam
  version: 0.1.0
tags:
  - cli
  - reference
---
```

**Required**

- `name` — kebab-case, must match the directory name.
- `description` — what the skill does and when an agent should use it. Keep the most important keywords in the first 250 characters since the Claude Code skill picker truncates longer descriptions.

**Project conventions**

- `metadata.author` and `metadata.version` are nested under `metadata:` (not flat top-level fields). This is a project-specific convention used by registries and human readers.
- `tags` is a flat YAML list at the root.

Note: Claude Code itself only reads `name` and `description`. The `metadata` and `tags` blocks are informational; they're surfaced in registries and listings.

## Skill content guidelines

- **Be concrete.** Tell the agent exactly what command to run, what file to edit, what the expected output looks like.
- **No filler.** Skip preamble like "this skill helps you with...". Start with the action.
- **Reference real code paths** when possible. An agent that knows the actual file layout writes better suggestions than one guessing.
- **Use examples generously.** A SKILL.md with three working examples beats one with five paragraphs of theory.
- **Keep it under ~500 lines.** Move detailed reference material to separate files under `references/` and link to them from `SKILL.md`.

## Repository layout

```
.claude-plugin/
  marketplace.json     # Claude Code plugin marketplace catalog
  plugin.json          # Plugin manifest
skills/
  openclam/
    SKILL.md
    references/
      *.md
README.md
LICENSE
CONTRIBUTING.md        # this file
AGENTS.md              # cross-runtime spec
CLAUDE.md              # symlink → AGENTS.md
```

## Testing locally

Before opening a PR, install the skill into your own agent and verify it loads:

```bash
# Using the OpenClam CLI
clam skills install --project --registry "$PWD#$(git branch --show-current)"

# Or manually for Claude Code
mkdir -p ~/.claude/skills
cp -R skills/your-new-skill ~/.claude/skills/
```

Then start a session and ask something the skill should handle. If the agent picks it up automatically (or `/your-new-skill` works), you're good.

## Versioning

We follow [semantic versioning](https://semver.org/) for individual skills:

- **PATCH** (`0.1.0` → `0.1.1`): typo fixes, clarifications, no behavior change.
- **MINOR** (`0.1.0` → `0.2.0`): new examples, new sections, new reference material.
- **MAJOR** (`0.1.0` → `1.0.0`): breaking changes to the skill's invocation patterns or scope.

Bump `metadata.version` in the SKILL.md frontmatter as part of your PR.

## License

By contributing, you agree that your contributions will be licensed under the [MIT License](./LICENSE).
