---
name: automate-rules
description: >
  Rules match events by pattern and optional conditions, then dispatch to one
  or more ordered targets — an extension action, a workflow, or a user-defined
  function. Covers the rule + rule_action schema, dot-notation conditions,
  target types, and the `clam rule` CLI.
metadata:
  author: openclam
  version: 0.1.0
---

# Rules

A rule is a database row that says "when `<event pattern>` fires and
`<conditions>` hold, execute these targets in order". Matching happens during
outbox processing. Unlike inline `events.on()` handlers, rules are persisted,
editable from any surface, and can target actions, workflows, or functions.

## Tables

**`rules`**

| Column           | Type  | Notes                                           |
| ---------------- | ----- | ----------------------------------------------- |
| `id`             | uuid  |                                                 |
| `slug`           | text  | Unique, stable handle                           |
| `name`           | text  | Display name                                    |
| `event_pattern`  | text  | Exact match, `*`, or `prefix.*`                 |
| `conditions`     | text? | JSON; see below                                 |
| `priority`       | int   | Lower runs first; default `0`                   |
| `is_active`      | bool  | Disabled rules are skipped                      |
| `created_by`     | text? |                                                 |
| `created_at` / `updated_at` / `deleted_at` | text | Soft-delete via `deleted_at` |

**`rule_actions`** — one row per target, evaluated in `execution_order`:

| Column            | Type  | Notes                                                 |
| ----------------- | ----- | ----------------------------------------------------- |
| `id`              | uuid  |                                                       |
| `rule_id`         | uuid  | FK → `rules.id`                                       |
| `target_type`     | text  | `'action'` (default) \| `'workflow'` \| `'function'`  |
| `extension`       | text  | For `action` targets — extension name                 |
| `action`          | text  | For `action` targets — action name within the extension |
| `workflow_name`   | text? | For `workflow` targets — slug or name                 |
| `function_name`   | text? | For `function` targets — slug                         |
| `params_json`     | text? | JSON params passed to the target                      |
| `execution_order` | int   | Ascending                                             |
| `delay_ms`        | int?  | Delay before enqueue (action targets)                 |
| `schedule_at`     | text? | Absolute ISO time to run (action targets)             |

**`rule_runs`** — audit row per (rule, event) match with `targets_fired`,
`status`, and `error`.

## Event patterns

Same matcher as events: `person.created`, `person.*`, `*`. Trailing `.*`
only — wildcards inside a pattern are not supported.

## Conditions

Conditions are a JSON object of `"dot.path": matcher` pairs evaluated
against the full event object `{ source, event_type, entity_type, entity_id,
payload, … }` with `payload` parsed as JSON. A rule fires only if **every**
condition matches.

Shorthand: a plain value means `$eq`:

```json
{ "payload.stage": "qualified" }
```

Operator form uses a `{ "$op": value }` object:

| Operator    | Semantics                                   |
| ----------- | ------------------------------------------- |
| `$eq`       | `actual === value`                          |
| `$ne`       | `actual !== value`                          |
| `$gt` / `$gte` / `$lt` / `$lte` | Numeric comparisons       |
| `$in`       | `value` is array containing `actual`        |
| `$nin`      | `value` is array NOT containing `actual`    |
| `$contains` | `actual` (string) includes `value`          |
| `$exists`   | `true` → actual is defined, `false` → null/undefined |

```json
{
  "payload.stage":  { "$in": ["qualified", "demo"] },
  "payload.value":  { "$gte": 50000 },
  "entity_type":    "deal"
}
```

A missing path returns `undefined`; operators that require a type (e.g.
`$gt` on a non-number) evaluate to `false`.

## Target types

### Action (default)

Looks up `<extension>:<action>` in the global action registry. Params
validated by the action's Zod schema. Enqueued to `scheduled_actions` and
run by the daemon's ScheduledActionRunner on the next tick — never inline.
`delay_ms` and `schedule_at` adjust the enqueue time.

```json
{ "target_type": "action",
  "extension":   "contacts",
  "action":      "add-note",
  "params_json": "{\"body\":\"Welcome!\"}" }
```

### Workflow

Triggers a new workflow run via the daemon's workflow trigger hook. The
workflow receives the event as its input.

```json
{ "target_type": "workflow", "workflow_name": "onboard-contact" }
```

### Function

Executes the registered user function by slug via the function trigger
hook. The triggering event is passed as `ctx.event`.

```json
{ "target_type": "function", "function_name": "score-icp" }
```

If a target fails (missing handler, trigger hook not registered, enqueue
error), the rule's `rule_run` row is marked `failed` with the error
captured but other matching rules continue.

## `clam rule` CLI

```bash
clam rule add <slug> \
  --name "Welcome on deal" \
  --event "deal.created" \
  [--conditions '{"payload.value":{"$gte":50000}}'] \
  [--priority 10] \
  --action contacts/add-note:'{"body":"Welcome!"}' \
  [--action deals/move-stage:'{"stage":"engaged"}']

clam rule list    [--search <q>] [--cursor <c>] [--limit <n>] \
                  [--sort created_at|name|priority|is_active] [--order asc|desc] \
                  [--all] [--fields <csv>]
clam rule count   [--search <q>]
clam rule get     <slug>
clam rule update  <slug> [--name …] [--event …] [--conditions …] \
                         [--priority …] [--action <spec>…]
clam rule enable  <slug>
clam rule disable <slug>
clam rule remove  <slug>  [--dry-run]
clam rule restore <slug>
clam rule purge   <slug>  [--dry-run]
clam rule actions                    # list all registered actions
clam rule import  <file>  [--format csv|json]
clam rule export  [file]  [--format csv|json]     # stdout if no file
```

`--action` specs are repeatable and take the form
`extension/action[:params_json]`. For workflow or function targets use
`--input-json` with the full array of action objects:

```bash
clam rule add welcome-deal --input-json '{
  "slug": "welcome-deal",
  "name": "Welcome on deal",
  "event_pattern": "deal.created",
  "actions": [
    { "target_type": "workflow", "workflow_name": "onboard-deal", "execution_order": 0 },
    { "target_type": "function", "function_name": "score-icp",    "execution_order": 1 }
  ]
}'
```

All commands accept `--input-json` which overrides individual flags.
`--schema` on `list`/`get` prints the Zod schema as JSON Schema.

See [`../surfaces.md`](../surfaces.md) to translate these operations to MCP,
HTTP, or the typed client.
