---
name: openclam-cli
description: >
  Skill for the OpenClam CLI — a data and execution layer for AI agents. Covers setup,
  workspaces, contacts, deals, tasks, messaging, marketing, content, calendar, research,
  storage, tickets, wiki, execution layer (events, rules, actions, workflows, cron, webhooks),
  and cross-cutting patterns. Use when user asks about clam commands, CLI usage, architecture,
  adding features, or understanding the codebase.
metadata:
  author: openclam
  version: 0.1.0
tags:
  - cli
  - architecture
  - contacts
  - deals
  - tasks
  - marketing
  - messaging
  - content
  - calendar
  - research
  - storage
  - tickets
  - wiki
  - extensions
  - services
  - events
  - rules
  - workflows
---

# OpenClam CLI

OpenClam is an extensible, modular system of record built for agents and automation. It gives AI agents and human operators a structured way to manage contacts, deals, tasks, content, messaging, and more — all through a unified CLI or HTTP API backed by a real database.

## Architecture

OpenClam has two distinct layers:

- **System of Record** — persistent domain data (contacts, deals, tasks, etc.) with full CRUD, schemas, and tables
- **Execution Layer** — reacts to state changes, runs workflows, dispatches actions, schedules jobs, and delivers webhooks

```
┌──────────────────────────────────────────────┐  ┌──────────────────────────────────┐
│          SYSTEM OF RECORD                    │  │        EXECUTION LAYER           │
│                                              │  │                                  │
│  Core (always loaded)                        │  │  Events → Rules → Actions        │
│  Workspaces, Users, Projects, Roles, Tags    │  │                ↓                 │
│  Rules, Log, Config, Extension System        │  │           Workflows              │
│                                              │  │                ↓                 │
│  Extensions (per environment)                │  │           Cron Jobs              │
│  Contacts, Deals, Tasks, Messaging      │──│──emit()        ↓                 │
│  Marketing, Content, Calendar, Research       │  │           Webhooks               │
│  Storage, Tickets, Wiki                      │  │                                  │
└──────────────────────────────────────────────┘  └──────────────────────────────────┘
```

Both layers are accessed via CLI (`clam` commands) or HTTP API (Hono). The CLI is the primary interface. All commands support `--json` for machine-readable output.

## How to run

The `clam` command must be installed and available in the user's PATH:

```bash
clam <command> [options]
```

If the `clam` command is not available, tell the user: "You will need to install the OpenClam CLI to get started." Do NOT attempt to use `npx`, `npm exec`, or any other package runner as a fallback.

## When to use

- User asks about `clam` CLI commands or workflows
- User asks about OpenClam's architecture, tables, services, or extension system
- User wants to add features, create extensions, or understand the codebase
- User asks about workspace setup, contacts, deals, tasks, messaging, marketing, content, calendar, research, or storage
- User asks about the execution layer — events, rules, actions, workflows, cron, webhooks

## References

Pick the reference file that matches what you're trying to do:

### Infrastructure / Config

- **Initial setup, workspaces, environments, database, daemon, secrets, AI providers, or config**? → `infrastructure.md`

### System of Record

- **Managing users, roles, projects, tags, custom fields, rules, logs**? → `core.md`
- **Working with contacts** (persons, orgs, emails, phones, notes, segments)? → `contacts.md`
- **Working with deals** (pipelines, stages, deals, forecasting)? → `deals.md`
- **Working with tasks** (create, assign, complete tasks)? → `tasks.md`
- **Sending messages or configuring email** (transports, send)? → `messaging.md`
- **Running outbound sequences** (create sequences, enroll contacts)? → `marketing.md`
- **Managing content** (content types, items, versions, assets, collections)? → `content.md`
- **Working with calendars** (calendars, events, attendees, recurrence)? → `calendar.md`
- **Research and intelligence** (topics, runs, findings, briefings, search)? → `research.md`
- **File storage** (buckets, files, permissions)? → `storage.md`

### Execution Layer

- **Events, rules, actions, workflows, cron, webhooks**? → `execution.md`

### Cross-cutting

- **Understanding how the system works** (extension model, service patterns, table conventions)? → `patterns.md`
- **Querying data** that a single `list`/`get` can't answer (joins, aggregations, cross-domain)? → `natural-query.md`

| Category | Reference | Covers |
|----------|-----------|--------|
| Infrastructure | `infrastructure.md` | Setup, workspace, env, db, daemon, secrets, AI providers, reset, config, status, container, token, login/logout |
| Core | `core.md` | Users, roles, rules, projects, tags, custom fields, logs, events, config — tables, services, CLI, API |
| Contacts | `contacts.md` | Persons, orgs, emails, phones, addresses, URLs, notes, comments, relationships, segments |
| Deals | `deals.md` | Deals, pipelines, stages, comments, notes, tags, project links, forecast |
| Tasks | `tasks.md` | Tasks, comments, project links, lifecycle states |
| Messaging | `messaging.md` | Transports, messages, sending email |
| Marketing | `marketing.md` | Sequences, steps, enrollments, comments |
| Content | `content.md` | Content types, items, versions, assets, collections, channels, locales, relations |
| Calendar | `calendar.md` | Calendars, events, attendees, recurrence, comments, tags, project links |
| Research | `research.md` | Topics, runs, sources, findings, briefings, text/semantic/hybrid search |
| Storage | `storage.md` | Buckets, files, permissions, local and S3 backends |
| Execution | `execution.md` | Event outbox, rule engine, action dispatch, workflows, cron, webhooks |
| Patterns | `patterns.md` | Extension system, service conventions, table design, event/rule engine, workspaces |
| Natural queries | `natural-query.md` | `clam sql` — read-only SQL, table schema map, cross-domain query workflow |

### When to use `clam sql` (natural queries)

The `sql` command executes read-only SQL directly against the database. Use it via:

```bash
# Positional query string
clam --json sql "SELECT ..."

# Or via --input-json
clam --json sql --input-json '{"query":"SELECT ..."}'
```

**When to use it:**
- The query **joins an entity with its sub-records** — e.g. "show me each org with its URLs" (orgs live in `orgs`, URLs live in `org_urls` — `list` returns them separately)
- The query **joins data across extensions** — e.g. "persons with open deals worth over $10k"
- The query needs **aggregation** — e.g. "how many contacts were added per month?" or "total deal value by stage"
- The query needs **filtering beyond what `list` supports** — e.g. "persons who haven't been emailed in 30 days"
- The query needs to **correlate across domains** — e.g. "deals created after a sequence enrollment completed"

**When NOT to use it** — a simple `list` or `get` already answers the question:
- "List all persons" → `clam --json contacts person list`
- "Get deal details" → `clam --json deals deal get --input-json '{"id":"..."}'`
- "List tasks by status" → `clam --json tasks task list --input-json '{"status":"open"}'`

Refer to `natural-query.md` for the full table schema map — never guess column names.

## Identity Model

OpenClam has two user types: **human** and **agent**. Both get API keys as the universal internal credential. The difference is how they authenticate interactively.

- **Agents** authenticate via `OPENCLAM_AGENT` env var (vault lookup), `OPENCLAM_TOKEN` env var, or `--token` flag. Agents CANNOT use `clam login`.
- **Humans** authenticate via `clam login` (password-based session). Humans additionally have passwords for interactive login. After login, no token is needed.

### Auth resolution order

1. **`--token` flag** — explicit, per-command
2. **`OPENCLAM_TOKEN` env var** — persistent for a session/process
3. **`OPENCLAM_AGENT` env var** — agent credential lookup from encrypted vault
4. **Active user session** — human login session (via `clam login`)
5. **Fail** — no credential found, command errors

### Output format per user type

Each user has an `outputFormat` stored in config:
- **Humans** default to `text` (colored, human-readable)
- **Agents** default to `json` (compact, machine-parseable)

The format is overridable per-command with `--json` or `--markdown`. Commands output one format only (no double output).

### Agent access

Set the `OPENCLAM_AGENT` environment variable (preferred — looks up the agent's API key from the encrypted vault) or `OPENCLAM_TOKEN`:

```bash
export OPENCLAM_AGENT=research-agent-01
clam --json contacts person list
clam --json deals deal add --input-json '{"name":"..."}'
```

Or pass `--token` per-command:

```bash
clam --json --token clam_abc123... contacts person list
```

**For HTTP API access**, include a single header:

```
Authorization: Bearer <api-key>
```

### Human access

Humans authenticate via `clam login`, which requires a password (if one has been set). After login, a long-lived session is created. Sessions are revocable server-side.

```bash
clam login                         # interactive picker — shows only human users
clam login alice@co.com            # direct login — prompts for password
clam --json contacts person list   # works automatically via session
```

### Creating users

#### Agent users

An admin creates an agent user. The response includes a one-time `api_key` field stored in the encrypted vault:

```bash
clam --json user add --input-json '{"name":"Research Agent","type":"agent","identifier":"research-agent-01"}'
# Response: { "ok": true, "user": {...}, "api_key": "clam_a1b2c3..." }
```

Store the API key immediately; it cannot be retrieved again.

#### Human users

An admin creates a human user. The response includes a **setup code** (6-digit, expires in 15 minutes):

```bash
clam --json user add --input-json '{"name":"Alice Smith","type":"human","identifier":"alice@co.com"}'
# Response: { "ok": true, "user": {...}, "setup_code": "482901", "expires_in": "15m" }
```

The human activates their account by logging in with the setup code, which prompts them to set a password:

```bash
clam login alice@co.com --code 482901
# Prompted: "Set your password: ********"
# Prompted: "Confirm password: ********"
```

After activation, subsequent logins require the password.

### Password management

```bash
clam passwd                        # Set or change login password (human users only)
```

### Key rotation

To rotate a compromised or expired agent API key:

```bash
clam --json user rotate-key <identifier>
```

This generates a new key and invalidates the old one.

### Security model

- **Agents cannot impersonate humans** — `clam login` is blocked if `OPENCLAM_AGENT` is set or user type is agent. Password is required for human login.
- **OS keychain preferred** for credential storage, with encrypted vault file as fallback.
- **Sessions are long-lived, revocable server-side.**
- **All auth events are logged** to the `log` table with `source='auth'` (login, logout, key rotation, failed attempts).

### Roles

`admin` is the only built-in role. Custom roles are created by an admin, optionally from templates (`editor`, `viewer`):

```bash
clam --json role add --input-json '{"slug":"editor","name":"Editor","permissions":["contacts.*","deals.*","tasks.*"]}'
clam --json role add --input-json '{"slug":"viewer","name":"Viewer","permissions":["contacts.person.read","deals.deal.read"]}'
clam --json role assign --input-json '{"user_identifier":"alice@co.com","role_slug":"editor"}'
```

---

## Agent usage rules

**You MUST follow these rules for every command:**

1. **`--json` or `--markdown`** on every command. `--json` produces compact JSON (required when parsing values programmatically). `--markdown` produces markdown tables for lists and key-value blocks for single objects (lower token count, better for display). Place the flag immediately after `clam`. Use `--json` when you need to extract IDs or values for subsequent commands; use `--markdown` when presenting results to the user. Combine `--markdown` with `--fields` for concise reports: `clam --markdown contacts org list --input-json '{"fields":"id,name,industry","limit":10}'`. Note: agent users default to JSON output, so `--json` is implicit but still recommended for clarity.
2. **Authentication** — set `OPENCLAM_AGENT` env var (preferred — vault lookup), `OPENCLAM_TOKEN` env var, or pass `--token` on every command. Agents CANNOT use `clam login`. See [Identity Model](#identity-model) above.
3. **`--input-json <json>` is the ONLY way to pass arguments.** Every command that accepts input — `add`, `update`, `get`, `list`, `send`, `enroll`, `sql`, `workspace new`, and all others — takes a single `--input-json '{ ... }'` flag with a JSON object. **Individual flags DO NOT EXIST.** There are no `--name`, `--email`, `--search`, `--limit`, `--status`, `--cursor`, or any other per-field flags. The only global flags are `--json`, `--markdown`, `--token`, `--env`, and `--input-json`. If a command takes no arguments (e.g. bare `list` with no filters), you can omit `--input-json`. See each domain's reference file for the exact TypeScript interface accepted by each command.
4. **NEVER run a command without `--input-json` if it requires input.** Commands like `workspace new`, `env add`, and `extensions` have interactive prompts that will hang in a non-interactive environment. Always provide `--input-json` with all required fields to skip prompts entirely.
5. **`remove`, `restore`, and `purge` accept BOTH forms** — positional ID or `--input-json`. For agents, still prefer `--input-json`.
6. **List commands return paginated results.** Every `list` command returns `{ "data": [...], "nextCursor": "...", "hasMore": true/false }`. To paginate, pass `cursor` and `limit` inside `--input-json`: `--input-json '{"cursor":"<nextCursor>","limit":20}'`. Parse `data` for results, use `nextCursor` for the next page.
7. **All output is JSON.** Every command returns a JSON object when `--json` is used. Parse the output to extract IDs and values for subsequent commands.

```bash
# CORRECT — --input-json for ALL arguments (assumes OPENCLAM_TOKEN is set)
clam --json contacts person add --input-json '{"first_name":"Alice","last_name":"Smith"}'
clam --json contacts person get --input-json '{"id":"<person-id>"}'
clam --json contacts person list --input-json '{"search":"Alice","limit":10}'
clam --json deals deal list --input-json '{"status":"open","limit":50}'
clam --json sql --input-json '{"query":"SELECT id, first_name FROM persons LIMIT 5"}'

# CORRECT — workspace new with --input-json (NEVER run without it)
clam --json workspace new --input-json '{"name":"My Project","slug":"my-project","dialect":"sqlite","extensions":["contacts","deals","tasks"],"userName":"Alice","userIdentifier":"alice@co.com","userType":"human"}'

# CORRECT — list with no filters (--input-json optional)
clam --json contacts person list

# PREFERRED for agents — explicit input-json even for remove/restore/purge
clam --json contacts person remove --input-json '{"id":"<person-id>"}'
clam --json contacts person restore --input-json '{"id":"<person-id>"}'
clam --json contacts person purge --input-json '{"id":"<person-id>"}'

# ALSO SUPPORTED — positional shorthand
clam --json contacts person remove <person-id>
clam --json contacts person restore <person-id>
clam --json contacts person purge <person-id>

# WRONG — individual flags DO NOT EXIST
clam --json contacts person add --first-name Alice --last-name Smith

# WRONG — running workspace new without --input-json (will hang on prompts)
clam --json workspace new

# WRONG — malformed input-json
clam --json contacts person purge --input-json '{"bad":"shape"}'
```

## Error handling

All errors are returned as JSON on stderr. There are two error shapes you must handle:

### Validation errors (Zod)

When input fails schema validation (wrong types, missing required fields, invalid formats), the CLI returns a structured error with per-field details:

```json
{
  "error": "Validation failed",
  "issues": [
    { "field": "email", "message": "Invalid email", "code": "invalid_string" },
    { "field": "phone", "message": "Phone must be in E.164 format (e.g. +12155551234)", "code": "custom" },
    { "field": "first_name", "message": "Required", "code": "invalid_type" }
  ]
}
```

**How to recover:** Read the `issues` array. Each issue tells you exactly which `field` failed and `message` explains why. Fix the specific fields in your `--input-json` and retry. Common validation rules:
- **UUIDs** (`id`, `person_id`, `org_id`, `deal_id`, etc.) — must be valid RFC 4122 UUIDs
- **Emails** — must be valid email format
- **Phone numbers** — must be E.164 format: `+` followed by country code and digits, e.g. `+12155551234` (no dashes, spaces, or letters)
- **Required fields** — check the TypeScript interface in the domain reference; any field without `?` is required

### General errors

All other errors (not found, permission, state errors) return:

```json
{ "error": "Person not found: <id>" }
```

**Common errors and recovery:**

| Error message | Cause | Recovery |
|---------------|-------|----------|
| `"Validation failed"` | Input failed schema validation | Read `issues` array, fix the named fields, retry |
| `"<Entity> not found: <id>"` | ID doesn't exist or was purged | Verify the ID is correct; if soft-deleted, `restore` first |
| `"No active workspace"` | No workspace configured | Run `clam setup` or `clam --json workspace new` |
| `"No user logged in"` | No authenticated user | Set `OPENCLAM_AGENT` or `OPENCLAM_TOKEN` env var, pass `--token`, or run `clam login` (humans only) |
| `"Authentication failed: invalid API key."` | Wrong or expired API key | Verify the token is correct. If lost, rotate with `clam --json user rotate-key <identifier>` |
| `"Agents cannot use interactive login"` | Agent tried `clam login` | Use `OPENCLAM_AGENT` env var or `--token` instead |
| `"Invalid or expired setup code"` | Setup code wrong or past 15 min | Admin must create a new user or generate a new setup code |
| `"Password required"` | Human user has password set but none provided | Enter password at the prompt during `clam login` |
| `"Table not found"` | Extension tables not created | Run `clam --json db migrate` |
| `"<field> is required (positional or in --input-json)"` | A required field is missing from the JSON | Add the named field to your `--input-json` object |
| `"Entity is not deleted"` on restore | Entity isn't soft-deleted | Nothing to restore — it's already active |
| `"Duplicate contacts"` | Contact already exists | Use `clam --json contacts person merge --input-json '{"source":"<id>","target":"<id>"}'` |

**General recovery strategy:** Always check the exit code. Non-zero means failure. Parse stderr as JSON. If `error` is `"Validation failed"`, iterate over `issues` to fix input. For all other errors, read the `error` string and take the appropriate action from the table above.

## End-to-end examples

### 1. New workspace setup
```
# First-time setup (interactive — asks "Are you a human or an agent?")
# Human installers are prompted to set a password after workspace creation
# Post-setup guidance uses namespaced commands (contacts person add, deals deal add, etc.)
clam setup

# Or non-interactive (for agents)
clam --json workspace new --input-json '{"name":"My Project","slug":"my-project","dialect":"sqlite","extensions":["contacts","deals","tasks"],"userName":"Alice","userIdentifier":"alice@co.com","userType":"human"}'
clam --json status
clam --json deals pipeline seed
```

### 2. Contact-to-deal workflow with comments
```
clam --json contacts person add --input-json '{"first_name":"Alice","last_name":"Smith","emails":[{"email":"alice@example.com","label":"work"}],"urls":[{"url":"https://linkedin.com/in/alice","label":"linkedin"}]}'
# parse person id from output
clam --json contacts org add --input-json '{"name":"Acme Corp","domain":"acme.com"}'
# parse org id from output
clam --json contacts person update --input-json '{"id":"<person-id>","org_id":"<org-id>"}'
clam --json deals deal add --input-json '{"name":"Acme Enterprise","person_id":"<person-id>","org_id":"<org-id>","stage":"Lead","value":50000}'
# parse deal id from output
clam --json deals deal comment add --input-json '{"deal_id":"<deal-id>","body":"Initial call went well, scheduling demo"}'
clam --json contacts person project add --input-json '{"person_id":"<person-id>","project_id":"<project-id>"}'
clam --json deals deal project add --input-json '{"deal_id":"<deal-id>","project_id":"<project-id>"}'
```

### 3. Outbound sequence with enrollment
```
clam --json marketing sequence add --input-json '{"name":"Q1 Outreach","description":"Cold outreach campaign"}'
# parse sequence id from output
clam --json marketing sequence add-step --input-json '{"sequence_id":"<seq-id>","channel":"email","subject":"Quick intro","body":"Hi {{first_name}}...","step_order":1,"delay_days":0}'
clam --json marketing sequence add-step --input-json '{"sequence_id":"<seq-id>","channel":"email","subject":"Following up","body":"Just checking in...","step_order":2,"delay_days":3}'
clam --json marketing enroll add --input-json '{"sequence_id":"<seq-id>","person_id":"<person-id>"}'
clam --json marketing enroll advance
clam --json marketing sequence comment add --input-json '{"sequence_id":"<seq-id>","body":"Response rate looking good at 15%"}'
```

### 4. Ad-hoc data query
```
clam --json status  # check enabled extensions
clam --json sql --input-json '{"query":"SELECT p.first_name, p.last_name, COUNT(d.id) as deals FROM persons p LEFT JOIN deals d ON d.person_id = p.id AND d.deleted_at IS NULL WHERE p.deleted_at IS NULL GROUP BY p.id ORDER BY deals DESC LIMIT 10"}'
```

### 5. Provision an agent and use it
```
# Admin creates agent user — API key is returned and stored in vault
clam --json user add --input-json '{"name":"Research Agent","type":"agent","identifier":"research-agent-01"}'
# Response: { "ok": true, "user": {...}, "api_key": "clam_a1b2c3..." }

# Agent authenticates via OPENCLAM_AGENT (preferred — vault lookup)
export OPENCLAM_AGENT=research-agent-01
clam --json contacts person list
clam --json deals deal add --input-json '{"name":"New Deal","stage":"Lead","value":10000}'

# Or via OPENCLAM_TOKEN (direct API key)
export OPENCLAM_TOKEN=clam_a1b2c3...
clam --json contacts person list

# Or pass per-command
clam --json --token clam_a1b2c3... contacts person list

# Rotate key if compromised
clam --json user rotate-key research-agent-01
# Response: { "ok": true, "identifier": "research-agent-01", "api_key": "clam_d4e5f6..." }
```

### 6. Provision a human user
```
# Admin creates human user — returns a setup code (not an API key)
clam --json user add --input-json '{"name":"Bob Jones","type":"human","identifier":"bob@co.com"}'
# Response: { "ok": true, "user": {...}, "setup_code": "482901", "expires_in": "15m" }

# Human activates account with the setup code (prompted to set password)
clam login bob@co.com --code 482901

# After activation, login uses password
clam login bob@co.com
# Prompted: "Password: ********"

# Change password
clam passwd
```

### 7. Soft delete and recovery
```
clam --json contacts person remove <person-id>
clam --json contacts person list
clam --json contacts person restore <person-id>
clam --json contacts person purge <person-id>
```
