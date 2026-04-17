---
name: operate-primitives
description: >
  Cross-domain primitives that every extension shares — `clam project`,
  `clam tag`, `clam field`, `clam log`. These are the connective tissue of
  OpenClam: group records across extensions, label them consistently, add
  custom columns without a migration, and inspect activity.
metadata:
  author: openclam
  version: 0.1.0
---

# Cross-domain primitives

Four primitives live in the core module and are available in every workspace
regardless of which extensions are enabled. Each extension declares which
primitives it participates in (project_tasks, person_tags, deal_tags, etc.);
the commands below operate on the shared tables and let you move across
extensions without writing a join by hand.

## Overview

| Primitive                | Table(s)                                    | What it provides                                          |
| ------------------------ | ------------------------------------------- | --------------------------------------------------------- |
| Projects                 | `projects`, `project_*` join tables         | Group disparate entities under one project / workstream.  |
| Tags                     | `tags`, `*_tags`                            | Slug-based labels applied to any entity type.             |
| Custom fields            | `custom_field_definitions`, `custom_field_values` | Per-entity ad-hoc columns without a migration.      |
| Log                      | `log`                                       | Activity log — auth, CLI use, timeline entries.           |

## Projects — `clam project`

A project is a named workstream that any entity (person, org, deal, task,
content, calendar event, etc.) can be attached to via the extension-specific
`project_<entity>` join tables. Useful for cross-extension filtering:
"everything tied to the Acme rollout."

### projects table

| Column       | Notes                                                    |
| ------------ | -------------------------------------------------------- |
| `id`         | UUID                                                     |
| `name`       | Display name                                             |
| `description`|                                                          |
| `status`     | e.g. `active`, `archived`                                |
| `owner_id`   | optional                                                 |
| `owner_type` | `human` or `agent`                                       |
| `deleted_at` | Soft delete                                              |

### Commands

| Sub                    | Purpose                                                           |
| ---------------------- | ----------------------------------------------------------------- |
| `add <name>`           | Create a project.                                                 |
| `list`                 | Paginated with `--status`, `--owner-type` filters.                |
| `count`                | Count matching filters.                                           |
| `get <id>`             | Single project.                                                   |
| `update <id>`          | Update name, description, status.                                 |
| `archive <id>`         | Set status to `archived` and soft-delete.                         |
| `remove <id>`          | Soft delete.                                                      |
| `restore <id>`         | Un-soft-delete.                                                   |
| `purge <id>`           | Hard delete (permanent).                                          |
| `import <file>` / `export <file>` | CSV or JSON bulk (detected from extension).            |

```
clam --json project add "Acme Rollout" --input-json '{"owner_type":"human","description":"Q2 onboarding"}'
clam --json project list --status active --sort created_at --order desc
clam --json project archive 0f3b...
```

Service: `projectService` — `add`, `list`, `get`, `update`, `archive`,
`remove`, `restore`, `purge`.

Extensions attach entities via their own commands (e.g. `clam person project
add <project-id> <person-id>` — see each extension's reference).

## Tags — `clam tag`

Slug-based labels. The tag is canonical; each extension has its own
`<entity>_tags` junction table.

### tags table

| Column        | Notes                                  |
| ------------- | -------------------------------------- |
| `id`          | UUID                                   |
| `label`       | Display label                          |
| `slug`        | UNIQUE, auto-generated from label      |
| `color`       | Optional (hex or name)                 |
| `created_by`  | FK → users.id                          |

### Commands

| Sub                   | Purpose                                                     |
| --------------------- | ----------------------------------------------------------- |
| `add <label>`         | Create or return existing (idempotent on slug).             |
| `list`                | Paginated, `--search`.                                      |
| `count`               | Count matching filters.                                     |
| `get <ref>`           | Resolve by UUID, slug, or label.                            |
| `update <ref>`        | Change label or color.                                      |
| `rename <old> <new>`  | Rename by current label.                                    |
| `purge <ref>`         | Hard delete globally (tags have no soft delete).            |
| `reindex`             | Rebuild per-entity tag usage counts.                        |
| `import` / `export`   | CSV / JSON bulk.                                            |

```
clam --json tag add "Strategic" --color "#F97316"
clam --json tag list --search strat --limit 20
clam --json tag rename "VIP" "Priority"
clam --json tag update vip --input-json '{"color":"#3B82F6"}'
clam --json tag purge strategic
```

Service: `tagService` — `create`, `list`, `get`, `getBySlug`, `getOrCreate`,
`update`, `remove`, `rename`.

Attachment to entities is per-extension (e.g. `clam person tag add`, `clam
deal tag add`) and lives in the extension references.

## Custom fields — `clam field`

Per-entity ad-hoc columns without a schema migration. Two tables:
`custom_field_definitions` (the "column") and `custom_field_values` (the
cells).

### custom_field_definitions

| Column         | Notes                                                                           |
| -------------- | ------------------------------------------------------------------------------- |
| `id`           | UUID                                                                            |
| `entity_type`  | e.g. `person`, `org`, `deal`, `task`                                            |
| `slug`         | UNIQUE per entity_type; auto-generated from label                               |
| `label`        | Display label                                                                   |
| `description`  |                                                                                 |
| `field_type`   | `text`, `number`, `boolean`, `date`, `select`, `multi_select`                   |
| `options_json` | JSON array of options (select / multi_select only)                              |
| `required`     | Boolean                                                                         |
| `position`     | Display order                                                                   |

### custom_field_values

| Column          | Notes                                                        |
| --------------- | ------------------------------------------------------------ |
| `id`            | UUID                                                         |
| `definition_id` | FK → `custom_field_definitions.id` ON DELETE CASCADE         |
| `entity_type`   | Denormalized for query speed                                 |
| `entity_id`     | UUID of the target entity                                    |
| `value_text` / `value_number` / `value_boolean` / `value_date` | Typed value columns; exactly one populated per `field_type` |

### Commands

| Sub                                                  | Purpose                                                              |
| ---------------------------------------------------- | -------------------------------------------------------------------- |
| `define <entity-type> <label>`                       | Create a definition. `--type` sets the field_type; `--options` CSV for select fields. |
| `list`                                               | Paginated. Filter by `--entity-type`, `--field-type`, `--search`.    |
| `count`                                              | Count matching filters.                                              |
| `get <ref>`                                          | Ref is UUID or `<entity_type>:<slug>`.                               |
| `update <ref>`                                       | Update label, description, required, position.                       |
| `remove <ref>`                                       | Hard delete (CASCADE to all values).                                 |
| `set <entity-type> <entity-id> <slug> <value>`       | Upsert a single field value.                                         |
| `values <entity-type> <entity-id>`                   | Return `{ slug: typedValue }` map.                                   |

```
# Define a custom field on persons
clam --json field define person "LinkedIn URL" --type text

# Define a select field
clam --json field define deal "Lead Source" --type select \
  --options "inbound,outbound,referral,partner"

# Set a value
clam --json field set person 0f3b... linkedin_url "https://linkedin.com/in/..."

# Bulk set via input-json
clam --json field set --input-json '{
  "entity_type":"person","entity_id":"0f3b...",
  "fields":[
    {"slug":"linkedin_url","value":"https://linkedin.com/in/..."},
    {"slug":"is_vip","value":true}
  ]
}'

# Read all field values for one entity
clam --json field values person 0f3b...
```

Service: `customFieldService` — `createDefinition`, `listDefinitions`,
`getDefinition`, `getDefinitionBySlug`, `updateDefinition`,
`removeDefinition`, `setValue`, `setValues`, `getValuesForEntity`,
`getValuesForEntities`, `removeValue`, `removeAllValues`.

## Log — `clam log`

The `log` table is the shared activity log: auth events, CLI invocations,
arbitrary extension events. Different from the `events` outbox (which feeds
handlers + rules); log entries are immutable notes meant for timelines and
auditing.

### log table

| Column      | Notes                                  |
| ----------- | -------------------------------------- |
| `id`        | UUID                                   |
| `person_id` | Loose FK                               |
| `org_id`    | Loose FK                               |
| `user_id`   | FK → users.id                          |
| `type`      | e.g. `auth.login`, `auth.key_used`     |
| `subject`   |                                        |
| `body`      |                                        |
| `source`    | e.g. `auth`, `cli`, `api`              |
| `created_at`|                                        |

### Commands

| Sub                     | Purpose                                                             |
| ----------------------- | ------------------------------------------------------------------- |
| `add <type> <person-id>`| Create a log entry (optional `--subject`, `--body`, `--org-id`).    |
| `list`                  | Paginated. Filters: `--search`, `--person-id`, `--org-id`, `--type`, `--source`, `--since`, `--until`. |
| `count`                 | Count matching filters.                                             |
| `get <id>`              | Single entry.                                                       |
| `update <id>`           | Update subject/body only.                                           |
| `remove <id>`           | Hard delete (log has no soft delete).                               |
| `timeline <person-id>`  | Paginated activity timeline for one person.                         |

```
clam --json log add touchpoint 0f3b... --input-json '{"subject":"Intro call","body":"30-min intro, agreed to follow up next week."}'

# Find auth events for the last day
clam --json log list --source auth --since "2026-04-15T00:00:00Z"

# Timeline for a contact
clam --json log timeline 0f3b... --limit 25
```

Service: `logService` — `add`, `list`, `get`, `update`, `remove`, `timeline`.

## See also

- Domain references under `../` (e.g. `../contacts.md`, `../deals.md`) —
  where `project_*`, `*_tags`, and custom fields are actually attached.
- [query.md](./query.md) — cross-primitive queries via `clam sql`.
- [auth.md](./auth.md) — auth events land in `log` with `source='auth'`.

See `../surfaces.md` to translate these operations to MCP, HTTP, or the typed
client.
