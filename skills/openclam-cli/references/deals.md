# Deals & Pipelines Domain Reference

Package: `@openclam/deals`
Extension name: `deals`
Dependencies: `contacts`

---

## Tables

### `pipelines`

| Column       | Type    | Notes                        |
|-------------|---------|------------------------------|
| id          | TEXT PK |                              |
| name        | TEXT    | NOT NULL                     |
| description | TEXT    | nullable                     |
| is_default  | INTEGER | boolean, NOT NULL            |
| created_at  | TEXT    | NOT NULL                     |
| updated_at  | TEXT    | NOT NULL                     |
| deleted_at  | TEXT    | nullable, soft-delete marker |

### `pipeline_stages`

| Column      | Type    | Notes                           |
|------------|---------|----------------------------------|
| id         | TEXT PK |                                  |
| pipeline_id| TEXT    | NOT NULL, FK -> pipelines.id     |
| name       | TEXT    | NOT NULL                         |
| stage_order| INTEGER | NOT NULL                         |
| probability| INTEGER | NOT NULL (0-100)                 |
| is_win     | INTEGER | boolean, NOT NULL                |
| is_loss    | INTEGER | boolean, NOT NULL                |
| created_at | TEXT    | NOT NULL                         |
| updated_at | TEXT    | NOT NULL                         |

### `deals`

| Column             | Type    | Notes                           |
|--------------------|---------|---------------------------------|
| id                 | TEXT PK |                                 |
| name               | TEXT    | NOT NULL                        |
| pipeline_id        | TEXT    | NOT NULL, FK -> pipelines.id    |
| person_id          | TEXT    | nullable                        |
| org_id             | TEXT    | nullable                        |
| stage              | TEXT    | NOT NULL, must match pipeline   |
| value              | REAL    | nullable                        |
| currency           | TEXT    | NOT NULL (default "USD")        |
| probability        | INTEGER | nullable                        |
| expected_close_date| TEXT    | nullable                        |
| notes              | TEXT    | nullable                        |
| created_at         | TEXT    | NOT NULL                        |
| updated_at         | TEXT    | NOT NULL                        |
| deleted_at         | TEXT    | nullable, soft-delete marker    |

### `deal_notes`

| Column     | Type    | Notes                                       |
|-----------|---------|----------------------------------------------|
| id        | TEXT PK |                                              |
| deal_id   | TEXT    | NOT NULL, FK -> deals.id ON DELETE CASCADE    |
| title     | TEXT    | nullable                                     |
| body      | TEXT    | NOT NULL                                     |
| source    | TEXT    | NOT NULL (e.g. "cli")                        |
| created_at| TEXT    | NOT NULL                                     |
| updated_at| TEXT    | NOT NULL                                     |

### `deal_tags`

| Column     | Type    | Notes                                       |
|-----------|---------|----------------------------------------------|
| id        | TEXT PK |                                              |
| deal_id   | TEXT    | NOT NULL, FK -> deals.id ON DELETE CASCADE    |
| tag_id    | TEXT    | NOT NULL, FK -> core tags table              |
| created_at| TEXT    | NOT NULL                                     |

### `deal_comments`

| Column     | Type    | Notes                                       |
|-----------|---------|----------------------------------------------|
| id        | TEXT PK |                                              |
| deal_id   | TEXT    | NOT NULL, FK -> deals.id ON DELETE CASCADE    |
| author_id | TEXT    | NOT NULL                                     |
| parent_id | TEXT    | nullable, self-ref for threading             |
| body      | TEXT    | NOT NULL                                     |
| status    | TEXT    | NOT NULL ("open" or "resolved")              |
| created_at| TEXT    | NOT NULL                                     |
| updated_at| TEXT    | NOT NULL                                     |

### `project_deals`

| Column     | Type    | Notes                                          |
|-----------|---------|------------------------------------------------|
| id        | TEXT PK |                                                |
| project_id| TEXT    | NOT NULL, FK -> projects.id ON DELETE CASCADE   |
| deal_id   | TEXT    | NOT NULL, FK -> deals.id ON DELETE CASCADE      |
| added_at  | TEXT    | NOT NULL                                       |
| added_by  | TEXT    | nullable                                       |

---

## Service Operations

### dealService (`services/deal.ts`)

| Operation  | Signature / Notes                                                                                  |
|-----------|-----------------------------------------------------------------------------------------------------|
| add       | `add(db, tables, input, ctx?)` -- validates stage against pipeline, uses default pipeline if none   |
| list      | `list(db, tables, filters?, pagination?, options?)` -- filters: search, stage, personId, orgId, pipelineId; pagination: offset, limit; options: expand (person, org, pipeline, tags, notes) |
| get       | `get(db, tables, id, options?)` -- returns DealDetail with person, org, logHistory; supports expand  |
| update    | `update(db, tables, id, input, ctx?)` -- partial update of name, person_id, org_id, stage, value, currency |
| remove    | `remove(db, tables, id, ctx?)` -- soft delete (sets deleted_at)                                     |
| restore   | `restore(db, tables, id, ctx?)` -- clears deleted_at                                                |
| purge     | `purge(db, tables, id, ctx?)` -- permanent hard delete                                              |
| pipeline  | `pipeline(db, tables, pipelineId?)` -- returns pipeline overview with stage counts and deal values   |
| forecast  | `forecast(db, tables, pipelineId?)` -- returns weighted forecast by stage probability                |

### dealStatsService (`services/stats.ts`)

| Operation     | Signature / Notes                                                        |
|--------------|---------------------------------------------------------------------------|
| pipelineStats | `pipelineStats(db, tables)` -- returns stages array, total count, totalValue |

### pipelineService (`services/pipeline.ts`)

| Operation       | Signature / Notes                                                                      |
|----------------|------------------------------------------------------------------------------------------|
| add            | `add(db, tables, input, ctx?)` -- if is_default, unsets other defaults                   |
| list           | `list(db, tables, filters?)` -- filters: search                                          |
| get            | `get(db, tables, idOrName)` -- looks up by UUID or name; returns pipeline + stages       |
| update         | `update(db, tables, id, input, ctx?)` -- partial update of name, description, is_default |
| remove         | `remove(db, tables, id, ctx?)` -- soft delete; blocks if only pipeline or has deals      |
| restore        | `restore(db, tables, id, ctx?)` -- clears deleted_at                                     |
| purge          | `purge(db, tables, id, ctx?)` -- hard deletes pipeline + its stages                      |
| seedDefault    | `seedDefault(db, tables)` -- creates "Sales" pipeline with 6 default stages; errors if pipelines exist |
| getDefaultPipeline | `getDefaultPipeline(db, tables)` -- returns default or first pipeline                |
| addStage       | `addStage(db, tables, input)` -- adds stage to pipeline                                  |
| updateStage    | `updateStage(db, tables, stageId, input)` -- partial update of name, order, probability, is_win, is_loss |
| removeStage    | `removeStage(db, tables, stageId)` -- blocks if deals exist in stage                     |
| reorderStages  | `reorderStages(db, tables, pipelineId, stageIds)` -- resets stage_order by array position |

### dealCommentService (`services/deal-comment.ts`)

| Operation | Signature / Notes                                                       |
|----------|--------------------------------------------------------------------------|
| add      | `add(db, tables, input, ctx?)` -- input: deal_id, author_id, parent_id?, body; status defaults to "open" |
| list     | `list(db, tables, dealId, filters?)` -- filters: status                   |
| get      | `get(db, tables, id)` -- single comment by ID                            |
| update   | `update(db, tables, id, input, ctx?)` -- input: body?, status?           |
| remove   | `remove(db, tables, id, ctx?)` -- hard delete                            |
| resolve  | `resolve(db, tables, id, ctx?)` -- sets status to "resolved"             |
| getThread| `getThread(db, tables, parentId)` -- returns child comments              |

### dealNoteService (`services/note.ts`)

| Operation | Signature / Notes                                                       |
|----------|--------------------------------------------------------------------------|
| add      | `add(db, tables, input, ctx?)` -- input: deal_id, title?, body, source   |
| list     | `list(db, tables, dealId, pagination?)` -- offset/limit, ordered desc    |
| get      | `get(db, tables, id)` -- single note by ID                               |
| update   | `update(db, tables, id, input, ctx?)` -- partial update of title, body, source |
| remove   | `remove(db, tables, id, ctx?)` -- hard delete                            |

### dealTagService (`services/tag.ts`)

| Operation          | Signature / Notes                                                   |
|-------------------|----------------------------------------------------------------------|
| apply             | `apply(db, tables, dealId, tagLabel)` -- get-or-create tag, then link; idempotent |
| remove            | `remove(db, tables, dealId, tagLabel)` -- removes link; errors if not found |
| getForDeal        | `getForDeal(db, tables, dealId)` -- returns rows with expanded tag object |
| getTagLabelsForDeal| `getTagLabelsForDeal(db, tables, dealId)` -- returns string[]       |

### projectLinkService (`services/project-link.ts`)

| Operation             | Signature / Notes                                           |
|----------------------|--------------------------------------------------------------|
| addDealToProject     | `addDealToProject(db, tables, projectId, dealId, ctx?)`      |
| removeDealFromProject| `removeDealFromProject(db, tables, projectId, dealId)`        |
| listDealsForProject  | `listDealsForProject(db, tables, projectId)`                  |
| listProjectsForDeal  | `listProjectsForDeal(db, tables, dealId)`                     |

---

## CLI Commands

All list commands return a paginated response:
```json
{ "data": [...], "nextCursor": "<string | null>", "hasMore": false }
```

### `deal`

```
# add (positional title or --input-json)
clam --json deals deal add "My Deal" --stage Lead --pipeline-id <id>
clam --json deals deal add --input-json '{"name":"My Deal","stage":"Lead","pipeline_id":"<id>"}'

# list (paginated)
clam --json deals deal list
clam --json deals deal list --search "acme" --pipeline-id <id> --cursor <cursor> --limit 25
clam --json deals deal list --input-json '{"search":"acme","pipeline_id":"<id>","cursor":"<cursor>","limit":25}'

# get (positional id or --input-json)
clam --json deals deal get <id>
clam --json deals deal get --input-json '{"id":"<deal-id>"}'

# update (positional id or --input-json)
clam --json deals deal update <id> --stage "Qualified"
clam --json deals deal update --input-json '{"id":"<deal-id>","stage":"Qualified"}'

# remove / restore / purge (positional id or --input-json)
clam --json deals deal remove <id>
clam --json deals deal remove --input-json '{"id":"<deal-id>"}'
clam --json deals deal restore <id>
clam --json deals deal restore --input-json '{"id":"<deal-id>"}'
clam --json deals deal purge <id>
clam --json deals deal purge --input-json '{"id":"<deal-id>"}'

# pipeline overview (no --input-json; optional --pipeline-id flag)
clam --json deals deal pipeline
clam --json deals deal pipeline --pipeline-id <id>

# forecast (no --input-json; optional --pipeline-id flag)
clam --json deals deal forecast
clam --json deals deal forecast --pipeline-id <id>

# stats
clam --json deals deal stats

# list projects for a deal (positional deal-id or --input-json)
clam --json deals deal projects <deal-id>
clam --json deals deal projects --input-json '{"deal_id":"<deal-id>"}'
```

### `deal comment`

```
# add (positional deal-id or --input-json)
clam --json deals deal comment add <deal-id> --body "Comment text"
clam --json deals deal comment add --input-json '{"deal_id":"<deal-id>","body":"Comment text"}'

# list (paginated; positional deal-id or --input-json)
clam --json deals deal comment list <deal-id>
clam --json deals deal comment list --input-json '{"deal_id":"<deal-id>","status":"open","cursor":"<cursor>","limit":25}'

# get (positional comment-id or --input-json)
clam --json deals deal comment get <comment-id>
clam --json deals deal comment get --input-json '{"id":"<comment-id>"}'

# resolve (positional comment-id or --input-json)
clam --json deals deal comment resolve <comment-id>
clam --json deals deal comment resolve --input-json '{"id":"<comment-id>"}'

# remove (positional comment-id or --input-json)
clam --json deals deal comment remove <comment-id>
clam --json deals deal comment remove --input-json '{"id":"<comment-id>"}'
```

### `deal note`

```
# add (positional deal-id or --input-json)
clam --json deals deal note add <deal-id> --body "Note text"
clam --json deals deal note add --input-json '{"deal_id":"<deal-id>","body":"Note text"}'

# list (paginated; positional deal-id or --input-json)
clam --json deals deal note list <deal-id>
clam --json deals deal note list --input-json '{"deal_id":"<deal-id>","cursor":"<cursor>","limit":25}'
```

### `deal tag`

```
# list (positional deal-id or --input-json)
clam --json deals deal tag list <deal-id>
clam --json deals deal tag list --input-json '{"deal_id":"<deal-id>"}'

# add (positional deal-id or --input-json)
clam --json deals deal tag add <deal-id> --tag "enterprise"
clam --json deals deal tag add --input-json '{"deal_id":"<deal-id>","tag":"enterprise"}'

# remove (positional deal-id or --input-json)
clam --json deals deal tag remove <deal-id> --tag "enterprise"
clam --json deals deal tag remove --input-json '{"deal_id":"<deal-id>","tag":"enterprise"}'
```

### `pipeline`

```
# add (positional name or --input-json)
clam --json deals pipeline add "Sales Pipeline"
clam --json deals pipeline add --input-json '{"name":"Sales Pipeline","is_default":true}'

# list (paginated)
clam --json deals pipeline list
clam --json deals pipeline list --search "sales" --cursor <cursor> --limit 25
clam --json deals pipeline list --input-json '{"search":"sales","cursor":"<cursor>","limit":25}'

# get (positional id or --input-json)
clam --json deals pipeline get <id>
clam --json deals pipeline get --input-json '{"id":"<pipeline-id>"}'

# update (positional id or --input-json)
clam --json deals pipeline update <id> --name "New Name"
clam --json deals pipeline update --input-json '{"id":"<pipeline-id>","name":"New Name"}'

# remove / restore / purge (positional id or --input-json)
clam --json deals pipeline remove <id>
clam --json deals pipeline remove --input-json '{"id":"<pipeline-id>"}'
clam --json deals pipeline restore <id>
clam --json deals pipeline restore --input-json '{"id":"<pipeline-id>"}'
clam --json deals pipeline purge <id>
clam --json deals pipeline purge --input-json '{"id":"<pipeline-id>"}'

# seed default pipeline
clam --json deals pipeline seed
```

### `pipeline stage`

```
# add (positional pipeline-id + name or --input-json)
clam --json deals pipeline stage add <pipeline-id> "Proposal"
clam --json deals pipeline stage add --input-json '{"pipeline_id":"<id>","name":"Proposal","probability":50}'

# update (positional stage-id or --input-json)
clam --json deals pipeline stage update <stage-id> --name "New Name"
clam --json deals pipeline stage update --input-json '{"id":"<stage-id>","name":"New Name","probability":60}'

# remove (positional stage-id or --input-json)
clam --json deals pipeline stage remove <stage-id>
clam --json deals pipeline stage remove --input-json '{"id":"<stage-id>"}'

# reorder (positional pipeline-id or --input-json)
clam --json deals pipeline stage reorder <pipeline-id> --stage-ids <id1> <id2> <id3>
clam --json deals pipeline stage reorder --input-json '{"pipeline_id":"<id>","stage_ids":["<id1>","<id2>","<id3>"]}'
```

---

## Global Flags

- `--json` -- Produce compact JSON output (must be placed immediately after `clam`, before the subcommand). Required for machine-readable output.
- `--input-json <json>` -- Pass structured JSON input to any command. Overrides positional args and individual flags. The JSON shape must match the TypeScript interfaces below.

---

## Input JSON Shapes

Every command accepts `--input-json <json>`. Field resolution order: `input.<field> ?? positionalArg ?? opts.<flag>`.

### deal add

```typescript
interface DealAddInput {
  name: string;                      // required (also accepted as positional or "title")
  pipeline_id?: string;              // UUID; uses default pipeline if omitted
  person_id?: string | null;         // UUID
  org_id?: string | null;            // UUID
  stage?: string;                    // defaults to "Lead" if omitted
  value?: number | null;
  currency?: string;                 // default: "USD"
}
```

### deal list

```typescript
interface DealListInput {
  search?: string;                   // FTS5 search across: name, notes
  stage?: string;
  person_id?: string;
  org_id?: string;
  pipeline_id?: string;
  cursor?: string;                   // pagination cursor from previous nextCursor
  limit?: number;
  expand?: string;                   // comma-separated: "person,org,pipeline,tags,notes"
  fields?: string;                   // comma-separated columns to return (e.g. "id,name,stage,value")
}
// Returns: { data: Deal[], nextCursor: string | null, hasMore: boolean }
```

### deal get

```typescript
interface DealGetInput {
  id: string;                        // required (also accepted as positional)
  expand?: string;                   // comma-separated expand fields
}
```

### deal update

```typescript
interface DealUpdateInput {
  id: string;                        // required (also accepted as positional)
  name?: string;                     // also accepted as "title"
  pipeline_id?: string;
  person_id?: string | null;
  org_id?: string | null;
  stage?: string;
  value?: number | null;
  currency?: string;
}
```

### deal remove / restore / purge

```typescript
interface DealIdInput {
  id: string;                        // required (also accepted as positional)
}
// remove returns: { ok: true, deleted: string }
// restore returns: Deal
// purge returns: { ok: true, purged: string }
```

### deal projects

```typescript
interface DealProjectsInput {
  deal_id?: string;                  // required (also accepted as "id" or positional)
}
```

### deal comment add

```typescript
interface DealCommentAddInput {
  deal_id: string;                   // required (also accepted as positional)
  body: string;                      // required
  author_id?: string;                // UUID; defaults to active user
  parent_id?: string | null;         // UUID; for threading
}
```

### deal comment list

```typescript
interface DealCommentListInput {
  deal_id: string;                   // required (also accepted as positional)
  status?: "open" | "resolved";
  cursor?: string;
  limit?: number;
}
// Returns: { data: DealComment[], nextCursor: string | null, hasMore: boolean }
```

### deal comment get / resolve / remove

```typescript
interface DealCommentIdInput {
  id: string;                        // required (also accepted as positional)
}
```

### deal note add

```typescript
interface DealNoteAddInput {
  deal_id: string;                   // required (also accepted as positional)
  body: string;                      // required
  title?: string | null;
}
// source is always set to "cli" by the command
```

### deal note list

```typescript
interface DealNoteListInput {
  deal_id: string;                   // required (also accepted as positional)
  cursor?: string;
  limit?: number;
}
// Returns: { data: DealNote[], nextCursor: string | null, hasMore: boolean }
```

### deal tag list / add / remove

```typescript
interface DealTagInput {
  deal_id: string;                   // required (also accepted as "id" or positional)
  tag?: string;                      // required for add/remove; tag slug or label
}
```

### pipeline add

```typescript
interface PipelineAddInput {
  name: string;                      // required (also accepted as positional)
  description?: string | null;
  is_default?: boolean;              // default: false
}
```

### pipeline list

```typescript
interface PipelineListInput {
  search?: string;
  cursor?: string;
  limit?: number;
}
// Returns: { data: Pipeline[], nextCursor: string | null, hasMore: boolean }
```

### pipeline get

```typescript
interface PipelineGetInput {
  id: string;                        // required (also accepted as positional); UUID or name
}
```

### pipeline update

```typescript
interface PipelineUpdateInput {
  id: string;                        // required (also accepted as positional)
  name?: string;
  description?: string | null;
  is_default?: boolean;
}
```

### pipeline remove / restore / purge

```typescript
interface PipelineIdInput {
  id: string;                        // required (also accepted as positional)
}
// remove returns: { ok: true, deleted: string }
// restore returns: Pipeline
// purge returns: { ok: true, purged: string }
```

### pipeline stage add

```typescript
interface PipelineStageAddInput {
  pipeline_id: string;               // required (also accepted as first positional)
  name: string;                      // required (also accepted as second positional)
  stage_order?: number;              // default: 0
  probability?: number;              // 0-100, default: 0
  is_win?: boolean;                  // default: false
  is_loss?: boolean;                 // default: false
}
```

### pipeline stage update

```typescript
interface PipelineStageUpdateInput {
  id: string;                        // required (also accepted as positional, key is "id")
  name?: string;
  stage_order?: number;
  probability?: number;              // 0-100
  is_win?: boolean;
  is_loss?: boolean;
}
```

### pipeline stage remove

```typescript
interface PipelineStageRemoveInput {
  id: string;                        // required (also accepted as positional)
}
// Returns: { ok: true, deleted: string }
```

### pipeline stage reorder

```typescript
interface PipelineStageReorderInput {
  pipeline_id: string;               // required (also accepted as positional)
  stage_ids: string[];               // required; array of stage UUIDs in desired order
}
```

---

## API Routes

### Deals

| Method | Path                          | Description                                | Query Params                                 |
|--------|-------------------------------|--------------------------------------------|----------------------------------------------|
| GET    | `/api/deals`                  | List deals                                 | stage, personId, orgId, pipelineId           |
| POST   | `/api/deals`                  | Create a deal                              |                                              |
| GET    | `/api/deals/pipeline`         | Pipeline overview (stages + deal counts)   | pipelineId                                   |
| GET    | `/api/deals/forecast`         | Weighted forecast by stage                 | pipelineId                                   |
| GET    | `/api/deals/stats`            | Aggregate pipeline statistics              |                                              |
| GET    | `/api/deals/:id`              | Get deal detail                            |                                              |
| PUT    | `/api/deals/:id`              | Update a deal                              |                                              |
| DELETE | `/api/deals/:id`              | Soft-delete a deal                         |                                              |
| PUT    | `/api/deals/:id/restore`      | Restore a soft-deleted deal                |                                              |
| DELETE | `/api/deals/:id/purge`        | Permanently delete a deal                  |                                              |

### Deal Notes

| Method | Path                          | Description          |
|--------|-------------------------------|----------------------|
| GET    | `/api/deals/:id/notes`        | List notes for deal  |
| POST   | `/api/deals/:id/notes`        | Add note to deal     |

### Deal Comments

| Method | Path                              | Description                   | Query Params |
|--------|-----------------------------------|-------------------------------|--------------|
| GET    | `/api/deals/:id/comments`         | List comments on a deal       | status       |
| POST   | `/api/deals/:id/comments`         | Add comment to a deal         |              |
| GET    | `/api/deal-comments/:id`          | Get single comment            |              |
| PUT    | `/api/deal-comments/:id`          | Update comment (body, status) |              |
| DELETE | `/api/deal-comments/:id`          | Delete comment                |              |
| PUT    | `/api/deal-comments/:id/resolve`  | Resolve a comment             |              |

### Deal Tags

| Method | Path                          | Description               | Body            |
|--------|-------------------------------|---------------------------|-----------------|
| GET    | `/api/deals/:id/tags`         | List tags for deal        |                 |
| POST   | `/api/deals/:id/tags`         | Apply tag to deal         | `{ tag: "..." }`|

### Pipelines

| Method | Path                                    | Description                          |
|--------|-----------------------------------------|--------------------------------------|
| GET    | `/api/pipelines`                        | List pipelines                       |
| POST   | `/api/pipelines`                        | Create a pipeline                    |
| POST   | `/api/pipelines/seed`                   | Seed default pipeline                |
| GET    | `/api/pipelines/:id`                    | Get pipeline with stages             |
| PUT    | `/api/pipelines/:id`                    | Update a pipeline                    |
| DELETE | `/api/pipelines/:id`                    | Soft-delete a pipeline               |
| PUT    | `/api/pipelines/:id/restore`            | Restore a soft-deleted pipeline      |
| DELETE | `/api/pipelines/:id/purge`              | Permanently delete pipeline + stages |

### Pipeline Stages

| Method | Path                                    | Description              | Body                   |
|--------|-----------------------------------------|--------------------------|------------------------|
| POST   | `/api/pipelines/:id/stages`             | Add stage to pipeline    |                        |
| PUT    | `/api/pipelines/:id/stages/reorder`     | Reorder stages           | `{ stageIds: [...] }`  |
| PUT    | `/api/pipelines/stages/:id`             | Update a stage           |                        |
| DELETE | `/api/pipelines/stages/:id`             | Remove a stage           |                        |

### Project-Deal Links

| Method | Path                                    | Description                  | Body                  |
|--------|-----------------------------------------|------------------------------|-----------------------|
| POST   | `/api/projects/:id/deals`               | Link deal to project         | `{ dealId: "..." }`   |
| DELETE | `/api/projects/:id/deals/:dealId`       | Unlink deal from project     |                       |
| GET    | `/api/projects/:id/deals`               | List deals for project       |                       |
| GET    | `/api/deals/:id/projects`               | List projects for deal       |                       |

### Stats

| Method | Path                  | Description               |
|--------|-----------------------|---------------------------|
| GET    | `/api/stats/pipeline` | Pipeline statistics (alias)|

---

## Events Emitted

| Event Type             | Entity Type    |
|------------------------|----------------|
| deal.created           | deal           |
| deal.updated           | deal           |
| deal.deleted           | deal           |
| deal.restored          | deal           |
| deal.purged            | deal           |
| deal.comment.created   | deal_comment   |
| deal.comment.updated   | deal_comment   |
| deal.comment.deleted   | deal_comment   |
| deal.comment.resolved  | deal_comment   |
| note.created           | deal           |
| note.updated           | deal           |
| note.deleted           | deal           |
| pipeline.created       | pipeline       |
| pipeline.updated       | pipeline       |
| pipeline.deleted       | pipeline       |
| pipeline.restored      | pipeline       |
| pipeline.purged        | pipeline       |

## Permissions

| Resource      | Actions                        |
|--------------|--------------------------------|
| deal         | create, read, update, delete   |
| deal_comment | create, read, update, delete   |
| pipeline     | create, read, update, delete   |
