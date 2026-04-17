---
name: operate-query
description: >
  Cross-domain read paths — `clam sql` for read-only SQL, `clam search` for
  text/semantic/hybrid search, `clam ask` for AI Q&A over your data, and
  `clam embed` for managing the embedding queue. Use this file when you
  want to pull information across extensions rather than through one
  extension's CRUD surface.
metadata:
  author: openclam
  version: 0.1.0
---

# Queries and read paths

Four commands cover everything from "give me the exact rows" (SQL) to "what's
the answer?" (ask). They form a progression — pick the one whose contract
matches your need.

| Command       | Contract                                                                               | Needs AI provider? |
| ------------- | -------------------------------------------------------------------------------------- | ------------------ |
| `clam sql`    | Read-only SQL over the workspace DB. Deterministic, exact rows.                        | No                 |
| `clam search` | Unified text / semantic / hybrid search across entities. Ranked list.                  | Semantic/hybrid yes|
| `clam ask`    | Retrieval-augmented Q&A. Answer + citations.                                           | Yes (completion)   |
| `clam embed`  | Manage the embedding queue (backfill / cleanup / reindex / stats). Not a read command. | Yes (embedding)    |

## `clam sql` — read-only SQL

The escape hatch for any read the typed commands don't cover. Accepts only
`SELECT`, `WITH`, `EXPLAIN`, and `PRAGMA` statements — writes (`INSERT`,
`UPDATE`, `DELETE`, `DROP`, `ALTER`, `CREATE`) are rejected by the service
layer.

```
clam --json sql "SELECT id, name FROM persons LIMIT 5"

# Or via input-json for multi-line queries
clam --json sql --input-json '{"query":"SELECT p.id, p.first_name, p.last_name FROM persons p WHERE p.deleted_at IS NULL ORDER BY p.updated_at DESC LIMIT 20"}'
```

Returns `{ rows: [...] }`.

Requires `config:read` permission (checked server-side).

### Workflow for cross-domain questions

1. `clam --json status` — confirm which extensions are enabled (only enabled
   extensions have their tables populated).
2. Look up the relevant column names in the table map below — do **not**
   guess column names.
3. Compose a `SELECT` and run it via `clam --json sql`.
4. If the data doesn't exist in any enabled extension's tables, say so.
   Never fabricate table or column names.

### Table map by extension

Core tables (always present when a workspace exists):

| Table                         | Purpose                                            |
| ----------------------------- | -------------------------------------------------- |
| `users`                       | All users                                          |
| `roles`, `role_permissions`, `user_roles` | Role-based access                      |
| `rules`, `rule_actions`       | Automation rules                                   |
| `projects`                    | Cross-extension projects                           |
| `tags`                        | Tag catalog                                        |
| `custom_field_definitions`, `custom_field_values` | Custom fields               |
| `log`                         | Activity log                                       |
| `events`                      | Outbox                                             |
| `decisions`, `decision_revisions` | Decision capture                               |
| `config`                      | Key-value workspace config                         |

Always-on infrastructure (loaded even if not called "extensions" in the
manifest):

| Table                                                   | Extension            |
| ------------------------------------------------------- | -------------------- |
| `cron_jobs`, `cron_runs`                                | cron                 |
| `workflow_definitions`, `workflow_runs`, `workflow_step_attempts`, `workflow_event_waiters` | workflows |
| `storage_buckets`, `storage_files`, `storage_bucket_permissions` | storage       |
| `event_subscribers`, `event_deliveries`                 | webhooks-outbound (when enabled)  |
| `webhooks_inbound_*`                                    | webhooks-inbound  (when enabled)  |
| `embedding_queue`, `embeddings`                         | AI runtime (when embedding provider configured)   |

Domain tables (present only when the extension is enabled for this
workspace):

- **contacts** — `persons`, `orgs`, plus per-entity attributes (`person_emails`, `person_phones`, `person_addresses`, `person_urls`, `person_notes`, `person_tags`, `person_comments`, and `org_*` equivalents), `relationships`, `segments`, `project_persons`, `project_orgs`.
- **deals** (requires contacts) — `pipelines`, `pipeline_stages`, `deals`, `deal_notes`, `deal_tags`, `deal_comments`, `project_deals`.
- **tasks** — `tasks`, `task_comments`, `project_tasks`.
- **content** — `content_types`, `content_items`, `content_versions`, `assets`, `content_assets`, `collections`, `collection_items`, `content_tags`, `content_notes`, `content_comments`, `content_channels`, `content_relations`, `content_locales`, `project_content`.
- **calendar** (requires contacts) — `calendars`, `calendar_events`, `calendar_event_attendees`, `calendar_event_comments`, `calendar_event_tags`, `project_calendar_events`.
- **research** — `research_topics`, `research_topic_persons`, `research_topic_orgs`, `research_runs`, `research_sources`, `research_findings`, `research_briefings`, `research_briefing_insights`, `research_briefing_insight_findings`.
- **tickets** — `tickets` and friends.
- **wiki** — `wiki_pages` and friends.
- **icp** — `icp_profiles` and friends.

See each domain's reference under `../` for full column lists.

### Example queries

```sql
-- Persons with the most open deals
SELECT p.first_name, p.last_name, COUNT(d.id) AS deal_count
FROM persons p
LEFT JOIN deals d
  ON d.person_id = p.id
  AND d.deleted_at IS NULL
WHERE p.deleted_at IS NULL
GROUP BY p.id
ORDER BY deal_count DESC
LIMIT 10;
```

```sql
-- Decisions made by agents in the last 7 days, joined to their events
SELECT d.id, d.rationale, d.confidence,
       e.event_type, e.entity_type, e.entity_id
FROM decisions d
JOIN events e ON e.id = d.event_id
WHERE d.actor_type = 'agent'
  AND d.created_at >= datetime('now', '-7 days')
ORDER BY d.created_at DESC;
```

```sql
-- Tasks by status with assignee name
SELECT t.title, t.status, t.priority, t.due_date, u.name AS assignee
FROM tasks t
LEFT JOIN users u ON t.assignee_id = u.id
WHERE t.deleted_at IS NULL
ORDER BY t.priority, t.due_date;
```

```sql
-- SQLite table introspection
PRAGMA table_info(persons);
```

## `clam search` — unified search

One search interface across every entity type. Three modes:

| `--mode`   | What it uses                                                        |
| ---------- | ------------------------------------------------------------------- |
| `text`     | Full-text search (FTS5 on SQLite, tsvector on Postgres). Default.   |
| `semantic` | Vector similarity against `embeddings`. Requires embedding provider + indexed content. |
| `hybrid`   | RRF-fused text + semantic ranking.                                  |

```
# Default text search, all entity types
clam --json search "acme corp" --limit 20

# Semantic search scoped to contacts only
clam --json search "strategic account in biotech" \
  --mode semantic \
  --types person,org \
  --limit 10

# Hybrid, with a similarity floor
clam --json search "renewal risk" --mode hybrid --threshold 0.25

# Or pass everything via input-json
clam --json search --input-json '{
  "query":"strategic account",
  "mode":"hybrid",
  "types":["person","org","deal"],
  "limit":20,
  "threshold":0.2
}'
```

Returns ranked results with the matched entity ID, type, score, and a
preview snippet.

Semantic and hybrid modes require:

1. An embedding provider configured — see the AI config in `../setup/`.
2. Entity content already embedded in the `embeddings` table — use
   `clam embed reindex` or let content embed on write.

## `clam ask` — AI Q&A

Retrieval-augmented generation over your workspace data. Uses `search` to
pull context, then `completion` to answer.

```
# Default: semantic retrieval, 5 context docs
clam ask "Who has the most open deals?"

# Hybrid mode with more context
clam ask "What's the current state of the Acme rollout?" --mode hybrid --limit 10

# Stream the response (great for interactive shells)
clam ask "Summarize today's activity" --stream

# Show raw context + answer for debugging
clam ask "Why did we move Acme to closed_won?" --raw
```

Requires both an embedding provider (for retrieval) and a completion
provider (for the answer). Raw mode returns the retrieved chunks + scores
alongside the final answer — useful for verifying the model isn't
hallucinating.

## `clam embed` — embedding queue

The queue that keeps vector indexes in sync with entity content. Not a read
command, but belongs in the query family because `search --mode semantic`
and `ask` depend on it.

| Sub         | Purpose                                                                              |
| ----------- | ------------------------------------------------------------------------------------ |
| `status`    | Show queue statistics (pending / processing / done / failed counts).                 |
| `backfill`  | Synchronously drain pending jobs. Useful after bulk imports.                         |
| `cleanup`   | Remove completed jobs older than N days (default 7).                                 |
| `reindex`   | Enqueue jobs for every entity whose embedding is stale against the target model.     |
| `stats`     | Embedding counts grouped by model and entity type.                                   |

```
# What's in the queue?
clam --json embed status

# Drain synchronously (agent workflow after a bulk import)
clam --json embed backfill --batch-size 100

# Reindex everything against the currently configured model
clam --json embed reindex

# Reindex only persons against a specific model
clam --json embed reindex \
  --entity-type person \
  --model text-embedding-3-small

# Tidy up finished jobs older than 30 days
clam --json embed cleanup --days 30

# Stats by model
clam --json embed stats
```

Service: `embeddingQueueService` — `stats`, `dequeue`, `markDone`,
`markFailed`, `cleanup`, `reindex`, `embeddingStats`.

## Picking the right tool

- **I know the columns and need exact rows.** → `clam sql`.
- **I have a vague string and want ranked matches.** → `clam search` (text
  mode for keywords, semantic for concept match, hybrid when unsure).
- **I have a question and want an answer, not rows.** → `clam ask`.
- **Semantic search is empty or stale.** → `clam embed reindex` then retry.
- **Performance audit of the embedding subsystem.** → `clam embed stats` +
  `clam embed status`.

## See also

- [primitives.md](./primitives.md) — shared tables used across extensions.
- [decisions.md](./decisions.md) — query the decision history via `clam sql`.
- Each domain's reference under `../` — column-level table details.

See `../surfaces.md` to translate these operations to MCP, HTTP, or the typed
client.
