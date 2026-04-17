---
name: operate-deals
description: >
  Deals and pipelines — pipelines, stages, deals, deal notes, comments, tags,
  forecasting. Covers the `clam deals` CLI, service operations, and database
  schema. Depends on the `contacts` extension.
metadata:
  author: openclam
  version: 0.1.0
---

# Deals

Package: `@openclam/deals` (extension: `deals`, depends on `contacts`)

---

## Tables

### pipelines
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | |
| description | TEXT | |
| is_default | INTEGER (boolean) NOT NULL | |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### pipeline_stages
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| pipeline_id | TEXT NOT NULL | FK to pipelines.id |
| name | TEXT NOT NULL | |
| stage_order | INTEGER NOT NULL | |
| probability | INTEGER NOT NULL | 0-100 |
| is_win | INTEGER (boolean) NOT NULL | |
| is_loss | INTEGER (boolean) NOT NULL | |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

### deals
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | |
| pipeline_id | TEXT NOT NULL | FK to pipelines.id |
| person_id | TEXT | FK to persons.id (contacts) |
| org_id | TEXT | FK to orgs.id (contacts) |
| stage | TEXT NOT NULL | Must match a stage name in the pipeline |
| value | REAL | |
| currency | TEXT NOT NULL | default `"USD"` |
| probability | INTEGER | |
| expected_close_date | TEXT | ISO 8601 |
| notes | TEXT | |
| created_by | TEXT | |
| updated_by | TEXT | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### deal_notes
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| deal_id | TEXT NOT NULL | FK to deals.id ON DELETE CASCADE |
| title | TEXT | |
| body | TEXT NOT NULL | |
| source | TEXT NOT NULL | e.g. `cli` |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

### deal_tags
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| deal_id | TEXT NOT NULL | FK to deals.id ON DELETE CASCADE |
| tag_id | TEXT NOT NULL | FK to global tags table |
| created_at | TEXT NOT NULL | |

### deal_comments
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| deal_id | TEXT NOT NULL | FK to deals.id ON DELETE CASCADE |
| author_id | TEXT NOT NULL | |
| parent_id | TEXT | Self-ref for threading |
| body | TEXT NOT NULL | |
| status | TEXT NOT NULL | `open` or `resolved` |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

### project_deals
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| project_id | TEXT NOT NULL | |
| deal_id | TEXT NOT NULL | FK to deals.id ON DELETE CASCADE |
| added_at | TEXT NOT NULL | |
| added_by | TEXT | |

---

## Service Operations

### dealService (`services/deal.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Validates stage against pipeline; uses default pipeline if `pipeline_id` omitted |
| `count(db, tables, filters?)` | Count non-deleted deals |
| `list(db, tables, filters?, pagination?, options?)` | filters: `search`, `stage`, `personId`, `orgId`, `pipelineId`; options: `expand` (`person`, `org`, `pipeline`, `tags`, `notes`), `fields` |
| `get(db, tables, id, options?)` | Returns `DealDetail` with person, org, logHistory; supports `expand` |
| `update(db, tables, id, input, ctx?)` | Partial update of `name`, `person_id`, `org_id`, `stage`, `value`, `currency`, `probability`, `expected_close_date`, `notes` |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Clear `deleted_at` |
| `purge(db, tables, id, ctx?)` | Hard delete |
| `pipeline(db, tables, pipelineId?)` | Pipeline overview with stage counts and values |
| `forecast(db, tables, pipelineId?)` | Weighted forecast by stage probability |
| `importFromFile(db, tables, filePath, format?)` / `importCsv(...)` | Bulk import |
| `exportData(db, tables)` / `exportToFile(db, tables, filePath?, format?)` | Bulk export |

### pipelineService (`services/pipeline.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | If `is_default`, unsets other defaults |
| `count(db, tables, filters?)` | |
| `list(db, tables, filters?)` | filters: `search` |
| `get(db, tables, idOrName)` | Lookup by UUID or name; returns pipeline + stages |
| `update(db, tables, id, input, ctx?)` | Partial update of `name`, `description`, `is_default` |
| `remove(db, tables, id, ctx?)` | Soft delete; blocked if only pipeline or has deals |
| `restore(db, tables, id, ctx?)` | Restore |
| `purge(db, tables, id, ctx?)` | Hard delete pipeline + stages |
| `seedDefault(db, tables)` | Create "Sales" pipeline with 6 default stages; errors if pipelines exist |
| `getDefaultPipeline(db, tables)` | Default or first pipeline |
| `addStage(db, tables, input)` | Add stage to pipeline |
| `updateStage(db, tables, stageId, input)` | Partial update |
| `removeStage(db, tables, stageId)` | Blocked if deals exist in stage |
| `reorderStages(db, tables, pipelineId, stageIds)` | Reset `stage_order` by array position |
| `importFromFile(...)` / `exportToFile(...)` | Pipeline import/export |

### dealCommentService (`services/deal-comment.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Input: `deal_id`, `author_id`, `parent_id?`, `body`; status defaults to `open` |
| `list(db, tables, dealId, filters?)` | Filter by `status` |
| `get(db, tables, id)` | Single comment |
| `update(db, tables, id, input, ctx?)` | Update `body`, `status` |
| `remove(db, tables, id, ctx?)` | Hard delete |
| `resolve(db, tables, id, ctx?)` | Sets status to `resolved` |
| `getThread(db, tables, parentId)` | Child comments |

### dealNoteService (`services/note.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Input: `deal_id`, `title?`, `body`, `source` |
| `list(db, tables, dealId, pagination?)` | Cursor pagination |
| `get(db, tables, id)` | Single note |
| `update(db, tables, id, input, ctx?)` | Update `title`, `body`, `source` |
| `remove(db, tables, id, ctx?)` | Hard delete |

### dealTagService (`services/tag.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `apply(db, tables, dealId, tagLabel)` | Get-or-create tag, then link; idempotent |
| `remove(db, tables, dealId, tagLabel)` | Remove link |
| `getForDeal(db, tables, dealId)` | Rows with expanded tag |
| `getTagLabelsForDeal(db, tables, dealId)` | `string[]` |

### dealStatsService (`services/stats.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `pipelineStats(db, tables)` | Stages array + total count + total value |

### projectLinkService (`services/project-link.ts`)
`addDealToProject`, `removeDealFromProject`, `listDealsForProject`, `listProjectsForDeal`.

---

## CLI Commands

All list commands return `{ data: [...], nextCursor: string | null, hasMore: boolean }`.

### deal

```
clam --json deals deal add --input-json '{"name":"Acme expansion","stage":"Lead","pipeline_id":"<id>","person_id":"<id>","org_id":"<id>","value":50000,"currency":"USD"}'
clam --json deals deal count --input-json '{"search":"...","stage":"...","pipeline_id":"..."}'
clam --json deals deal list --input-json '{"search":"acme","pipeline_id":"<id>","cursor":"...","limit":25,"expand":"person,org,pipeline,tags,notes"}'
clam --json deals deal get --input-json '{"id":"<deal-id>","expand":"person,org,pipeline,tags,notes"}'
clam --json deals deal update --input-json '{"id":"<deal-id>","stage":"Qualified","value":75000}'
clam --json deals deal remove --input-json '{"id":"<deal-id>"}'
clam --json deals deal restore --input-json '{"id":"<deal-id>"}'
clam --json deals deal purge --input-json '{"id":"<deal-id>"}'
clam --json deals deal pipeline --input-json '{"pipeline_id":"<id>"}'
clam --json deals deal forecast --input-json '{"pipeline_id":"<id>"}'
clam --json deals deal stats
clam --json deals deal projects --input-json '{"deal_id":"<deal-id>"}'
clam --json deals deal import --input-json '{"file":"./deals.csv","format":"csv"}'
clam --json deals deal export --input-json '{"file":"./deals.csv","format":"csv"}'
```

### deal comment

```
clam --json deals deal comment add --input-json '{"deal_id":"<deal-id>","body":"...","author_id":"...","parent_id":"..."}'
clam --json deals deal comment list --input-json '{"deal_id":"<deal-id>","status":"open","cursor":"...","limit":25}'
clam --json deals deal comment get --input-json '{"id":"<comment-id>"}'
clam --json deals deal comment resolve --input-json '{"id":"<comment-id>"}'
clam --json deals deal comment remove --input-json '{"id":"<comment-id>"}'
```

### deal note

```
clam --json deals deal note add --input-json '{"deal_id":"<deal-id>","body":"...","title":"..."}'
clam --json deals deal note list --input-json '{"deal_id":"<deal-id>","cursor":"...","limit":25}'
```

### deal tag

```
clam --json deals deal tag list --input-json '{"deal_id":"<deal-id>"}'
clam --json deals deal tag add --input-json '{"deal_id":"<deal-id>","tag":"enterprise"}'
clam --json deals deal tag remove --input-json '{"deal_id":"<deal-id>","tag":"enterprise"}'
```

### pipeline

```
clam --json deals pipeline add --input-json '{"name":"Sales Pipeline","description":"...","is_default":true}'
clam --json deals pipeline count
clam --json deals pipeline list --input-json '{"search":"sales","cursor":"...","limit":25}'
clam --json deals pipeline get --input-json '{"id":"<pipeline-id>"}'
clam --json deals pipeline update --input-json '{"id":"<pipeline-id>","name":"New Name","is_default":false}'
clam --json deals pipeline remove --input-json '{"id":"<pipeline-id>"}'
clam --json deals pipeline restore --input-json '{"id":"<pipeline-id>"}'
clam --json deals pipeline purge --input-json '{"id":"<pipeline-id>"}'
clam --json deals pipeline seed
clam --json deals pipeline import --input-json '{"file":"./pipelines.json","format":"json"}'
clam --json deals pipeline export --input-json '{"file":"./pipelines.json","format":"json"}'
```

### pipeline stage

```
clam --json deals pipeline stage add --input-json '{"pipeline_id":"<id>","name":"Proposal","probability":50,"stage_order":3}'
clam --json deals pipeline stage update --input-json '{"id":"<stage-id>","name":"New Name","probability":60,"is_win":false,"is_loss":false}'
clam --json deals pipeline stage remove --input-json '{"id":"<stage-id>"}'
clam --json deals pipeline stage reorder --input-json '{"pipeline_id":"<id>","stage_ids":["<id1>","<id2>","<id3>"]}'
```

---

## Input JSON Shapes

#### deal add
```typescript
interface DealAddInput {
  name: string;                      // also accepted as positional or "title"
  pipeline_id?: string;              // uses default pipeline if omitted
  person_id?: string | null;
  org_id?: string | null;
  stage?: string;                    // defaults to "Lead"
  value?: number | null;
  currency?: string;                 // default: "USD"
  probability?: number | null;
  expected_close_date?: string | null; // ISO 8601
  notes?: string | null;
}
```

#### deal list
```typescript
interface DealListInput {
  search?: string;                   // FTS5 across name, notes
  stage?: string;
  person_id?: string;
  org_id?: string;
  pipeline_id?: string;
  cursor?: string;
  limit?: number;
  expand?: string;                   // comma-separated: "person,org,pipeline,tags,notes"
  fields?: string;                   // e.g. "id,name,stage,value"
}
```

#### pipeline add
```typescript
interface PipelineAddInput {
  name: string;
  description?: string | null;
  is_default?: boolean;              // default: false
}
```

#### pipeline stage add
```typescript
interface PipelineStageAddInput {
  pipeline_id: string;
  name: string;
  stage_order?: number;              // default: 0
  probability?: number;              // 0-100, default: 0
  is_win?: boolean;                  // default: false
  is_loss?: boolean;                 // default: false
}
```

#### pipeline stage reorder
```typescript
interface PipelineStageReorderInput {
  pipeline_id: string;
  stage_ids: string[];               // UUIDs in desired order
}
```

#### deal comment add
```typescript
interface DealCommentAddInput {
  deal_id: string;
  body: string;
  author_id?: string;                // defaults to active user
  parent_id?: string | null;
}
```

#### deal tag list / add / remove
```typescript
interface DealTagInput {
  deal_id: string;
  tag?: string;                      // required for add/remove; slug or label
}
```

---

See `../surfaces.md` to translate these operations to MCP, HTTP, or the typed client.
