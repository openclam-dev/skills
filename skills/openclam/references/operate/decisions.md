---
name: operate-decisions
description: >
  Decisions are OpenClam's first-class audit / explainability layer. Every
  mutation can carry a `_decision` field capturing *why* it happened, who
  decided, with what confidence, and against which proposal. The resulting
  decisions table links rationales to the events they produced and builds
  an enterprise context graph across every change in the system.
metadata:
  author: openclam
  version: 0.1.0
---

# Decisions

A decision is a structured rationale attached to a mutation: who decided,
what they were considering, how confident they were, and why. Decision
capture is a product differentiator — instead of burying "why" in chat
history or commit messages, OpenClam stores it next to the data change,
linked via `event_id` to the event the mutation emitted.

The intended use is **agents annotating their own actions**. When an agent
updates a deal stage, creates a contact, or disables a rule, it passes a
`_decision` object alongside the normal input — the runtime records the
mutation as an event *and* writes a paired decision row referencing that
event.

## Tables

### decisions

| Column            | Type           | Notes                                                                      |
| ----------------- | -------------- | -------------------------------------------------------------------------- |
| `id`              | TEXT PK        | UUID                                                                       |
| `event_id`        | TEXT           | FK → `events.id` — the mutation this decision explains                     |
| `actor_type`      | TEXT           | `human`, `agent`, or `human-via-agent`                                     |
| `actor_id`        | TEXT           | FK → `users.id` (nullable for pure-agent decisions)                        |
| `rationale`       | TEXT           | Free-form explanation                                                      |
| `confidence`      | TEXT           | `high`, `medium`, `low`                                                    |
| `proposal`        | TEXT           | JSON — the options the actor considered                                    |
| `final_action`    | TEXT           | JSON — what was ultimately done                                            |
| `metadata`        | TEXT           | JSON — anything else (sources, prompts, cost)                              |
| `source_context`  | TEXT           | Free-form pointer back to the originating context (PR URL, chat ID, etc.) |
| `revision`        | INTEGER        | Starts at 1; bumped on every `decision revise`                             |
| `created_at`      | TEXT NOT NULL  | ISO 8601                                                                   |
| `updated_at`      | TEXT NOT NULL  | ISO 8601                                                                   |

### decision_revisions

Append-only log of prior versions so decisions can be revised without losing
history.

| Column         | Type          | Notes                                            |
| -------------- | ------------- | ------------------------------------------------ |
| `id`           | TEXT PK       |                                                  |
| `decision_id`  | TEXT          | FK → `decisions.id`                              |
| `revision`     | INTEGER       | 1, 2, 3…                                         |
| `rationale`    | TEXT          | Snapshot at that revision                        |
| `confidence`   | TEXT          | Snapshot at that revision                        |
| `metadata`     | TEXT          | JSON snapshot                                    |
| `edited_by`    | TEXT          | FK → `users.id`                                  |
| `edit_reason`  | TEXT          | Reason the actor gave for this revision          |
| `created_at`   | TEXT NOT NULL |                                                  |

## The `_decision` parameter

Every mutation that goes through the typed client, HTTP API, MCP, or `clam
<thing> <action> --input-json` accepts an optional `_decision` field
alongside the normal payload:

```json
{
  "name": "Acme Corp Deal",
  "stage": "closed_won",
  "_decision": {
    "rationale": "Customer countersigned MSA this morning; unblocks Q2 renewal.",
    "confidence": "high",
    "actorType": "human-via-agent",
    "proposal": { "alternatives": ["closed_won", "negotiation"] },
    "metadata": { "source_msg": "slack://C12/p1744812341" },
    "sourceContext": "https://slack.com/archives/C12/p1744812341"
  }
}
```

Fields on `_decision`:

| Field           | Type                                           | Meaning                                    |
| --------------- | ---------------------------------------------- | ------------------------------------------ |
| `rationale`     | string                                         | Plain-language explanation.                |
| `confidence`    | `'high' \| 'medium' \| 'low'`                  | How sure the actor is.                     |
| `actorType`     | `'human' \| 'agent' \| 'human-via-agent'`      | Defaults from the auth context.            |
| `proposal`      | object                                         | JSON of considered options.                |
| `metadata`      | object                                         | JSON for anything else.                    |
| `sourceContext` | string                                         | Pointer back to originating context.       |

At the CLI/HTTP/MCP boundary the runtime extracts `_decision` via
`extractDecision()` (`packages/core/src/services/decision-helper.ts`) and
passes it as `ctx.decision` into the mutation. After the mutation emits its
event, `maybeCreateDecision()` writes the decision row linked to that event.
Mutations that don't emit events (rare) silently drop the decision.

## Operations — `clam decision`

The `decision` command is read and revise only; **you do not create decisions
directly** via `clam decision add` — they are always created as a side effect
of another mutation that carries `_decision`.

| Sub           | Purpose                                                            |
| ------------- | ------------------------------------------------------------------ |
| `list`        | List decisions with rich filters.                                  |
| `get <id>`    | One decision plus its full revision history.                       |
| `revise <id>` | Append a new revision (rationale / confidence / metadata).         |
| `count`       | Count decisions matching filters.                                  |

### `clam decision list`

| Filter flag                  | Matches                                                          |
| ---------------------------- | ---------------------------------------------------------------- |
| `--actor-type <type>`        | `human`, `agent`, `human-via-agent`                              |
| `--confidence <level>`       | `high`, `medium`, `low`                                          |
| `--event-type <type>`        | Joins to events: e.g. `deal.updated`                             |
| `--source <source>`          | Joins to events: e.g. `sor/deals`                                |
| `--entity-type <t>` + `--entity-id <id>` | Both required together — decisions for one entity    |
| `--expand event,revisions`   | Include joined event details and full revision history           |
| `--cursor / --limit / --order / --all / --fields` | Standard list-command pagination & shaping  |

```
# All low-confidence agent decisions in the last page
clam --json decision list --actor-type agent --confidence low --limit 20

# All decisions about deal <id>
clam --json decision list \
  --entity-type deal --entity-id 0f3b... \
  --expand event,revisions

# All recent decisions from the deals module
clam --json decision list --source sor/deals --order desc
```

### `clam decision get <id>`

Returns the decision joined to its originating event, plus the complete
`revisions[]` array.

```
clam --json decision get 9c12...
```

### `clam decision revise <id>`

Revision is the approved way to edit a decision — the old version is
preserved in `decision_revisions`, `revision` increments, and a
`decision.revised` event is emitted.

```
clam --json decision revise 9c12... --input-json '{
  "rationale": "Revised after legal review rejected the countersignature.",
  "confidence": "medium",
  "edit_reason": "Legal review flagged term language on 2026-04-16."
}'
```

Flags: `--rationale`, `--confidence`, `--reason` (aliased to `edit_reason` in
the JSON schema). `metadata` can only be set via `--input-json`.

## Service signatures

```ts
// Create is called internally by maybeCreateDecision — not from user code
decisionService.create(
  db, tables,
  input: {
    event_id: string,
    actor_type: 'human' | 'agent' | 'human-via-agent',
    actor_id: string | null,
    rationale: string | null,
    confidence: 'high' | 'medium' | 'low' | null,
    proposal: string | null,     // JSON-serialized
    metadata: string | null,
    source_context: string | null,
  }
): Promise<Decision>

decisionService.get(db, tables, id): Promise<EnrichedDecision>
// → decision + joined event fields + revisions[]

decisionService.list(
  db, tables,
  filters: { actor_type?, confidence?, event_type?, source? },
  pagination?,
  options?: { expand?: ('event'|'revisions')[] }
): Promise<PaginatedResult<EnrichedDecision>>

decisionService.listForEntity(
  db, tables,
  entityType: string, entityId: string,
  pagination?
): Promise<PaginatedResult<Decision>>

decisionService.revise(
  db, tables, id,
  input: { rationale?, confidence?, metadata?, editReason? },
  ctx?: { userId?: string }
): Promise<Decision>

decisionService.getForEvent(db, tables, eventId): Promise<Decision | null>
```

And the boundary helper used by CLI / HTTP / MCP:

```ts
// services/decision-helper.ts
extractDecision<T>(input: T): { data: Omit<T, '_decision'>, decision?: DecisionContext }
maybeCreateDecision(db, tables, event: Event | null, ctx?: MutationContext): Promise<void>
```

## The enterprise context graph

Over time the `events` + `decisions` + `decision_revisions` tables form a
queryable graph:

- Every mutation is an event (`events.entity_type`, `entity_id`).
- Most agent-driven mutations carry a decision linked 1:1 by `event_id`.
- Decisions revise over time, producing a visible reasoning history.
- `source_context` lets you jump back to the originating
  conversation/PR/thread that triggered the change.

Typical queries — authored via `clam sql` (see [query.md](./query.md)):

```sql
-- Every low-confidence mutation to deals this week
SELECT d.id, d.rationale, d.confidence, e.event_type, e.entity_id, d.created_at
FROM decisions d
JOIN events e ON e.id = d.event_id
WHERE d.confidence = 'low'
  AND e.source = 'sor/deals'
  AND d.created_at >= datetime('now','-7 days')
ORDER BY d.created_at DESC;

-- Audit trail for one contact
SELECT d.revision, d.rationale, d.confidence, d.created_at, e.event_type
FROM decisions d
JOIN events e ON e.id = d.event_id
WHERE e.entity_type = 'person' AND e.entity_id = ?
ORDER BY d.created_at;
```

## See also

- [auth.md](./auth.md) — `actor_id` is a users.id reference; the auth
  chain determines the default `actor_type`.
- [query.md](./query.md) — the cross-domain query surfaces for mining the
  decision history.

See `../surfaces.md` to translate these operations to MCP, HTTP, or the typed
client.
