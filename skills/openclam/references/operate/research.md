---
name: operate-research
description: >
  Research & intelligence in OpenClam — topics, runs, sources, findings,
  briefings, and hybrid (text + semantic) search over findings. The `clam
  research` CLI.
metadata:
  author: openclam
  version: 0.1.0
---

# Research

Package: `@openclam/research` (ships as a daemon extension; tables and
commands are loaded through the extension SDK alongside the core runtime).

Research is organised as a tree:

```
topic            (the thing you're researching)
└── run          (a bounded pass of data collection)
    ├── source   (where a finding came from)
    └── finding  (an atomic piece of information, embeddable)
briefing         (a synthesized document attached to a topic)
└── insight      (a Q&A inside a briefing)
    └── finding* (links back to findings that support the insight)
```

Topics can be linked to contacts (persons, orgs) so you can pin research to
the entities it's about. Findings are auto-embedded and searchable via
`text`, `semantic`, `hybrid`, and `reranked` modes.

## Tables

### research_topics

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | Topic name |
| description | TEXT | |
| color | TEXT | Hex color |
| status | TEXT NOT NULL | e.g. `active` (default) |
| created_by | TEXT | User UUID |
| created_at | TEXT NOT NULL | ISO 8601 |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete |

### research_topic_persons

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| topic_id | TEXT NOT NULL | FK → research_topics (CASCADE) |
| person_id | TEXT NOT NULL | References a contacts person |
| created_at | TEXT NOT NULL | |

### research_topic_orgs

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| topic_id | TEXT NOT NULL | FK → research_topics (CASCADE) |
| org_id | TEXT NOT NULL | References a contacts org |
| created_at | TEXT NOT NULL | |

### research_runs

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| topic_id | TEXT NOT NULL | FK → research_topics (CASCADE) |
| status | TEXT NOT NULL | `running`, `completed`, `failed` |
| summary | TEXT | |
| started_at | TEXT NOT NULL | ISO 8601 — set on insert |
| completed_at | TEXT | Set by `complete` / `fail` |
| created_at | TEXT NOT NULL | |

### research_sources

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| run_id | TEXT NOT NULL | FK → research_runs (CASCADE) |
| url | TEXT | |
| source_type | TEXT NOT NULL | `webpage`, `article`, etc. |
| title | TEXT | |
| author | TEXT | |
| published_at | TEXT | |
| summary | TEXT | |
| created_at | TEXT NOT NULL | |

### research_findings

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| run_id | TEXT NOT NULL | FK → research_runs (CASCADE) |
| source_id | TEXT | FK → research_sources (SET NULL) |
| title | TEXT | |
| content | TEXT NOT NULL | Min length 1 |
| confidence | INTEGER | 1–5 scale |
| embedding | F32_BLOB(1536) | libSQL native vector column |
| embedding_model | TEXT | e.g. `text-embedding-3-small` |
| embedding_dimensions | INTEGER | |
| created_at | TEXT NOT NULL | |

### research_briefings

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| topic_id | TEXT NOT NULL | FK → research_topics (CASCADE) |
| title | TEXT NOT NULL | |
| summary | TEXT | |
| created_at | TEXT NOT NULL | |

### research_briefing_insights

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| briefing_id | TEXT NOT NULL | FK → research_briefings (CASCADE) |
| question | TEXT NOT NULL | |
| content | TEXT NOT NULL | Answer / synthesis |
| order | INTEGER NOT NULL | Display order |
| created_at | TEXT NOT NULL | |

### research_briefing_insight_findings

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| insight_id | TEXT NOT NULL | FK → research_briefing_insights (CASCADE) |
| finding_id | TEXT NOT NULL | FK → research_findings (CASCADE) |
| relevance_score | INTEGER | 0–100 |
| created_at | TEXT NOT NULL | |

## Service Operations

All services live in `@openclam/research/services`. Signatures take the
daemon's `db` and `tables` handles from the extension context; the HTTP
client variants expose the same shapes through `createHttpResearchClient`.

### topicService

```ts
add(db, tables, input, ctx?): Promise<Topic>
list(db, tables, filters?, pagination?, options?): Promise<Paginated<Topic>>
count(db, tables, filters?): Promise<{ count: number }>
get(db, tables, id): Promise<Topic>
update(db, tables, id, input, ctx?): Promise<Topic>
remove(db, tables, id, ctx?): Promise<void>    // soft delete
restore(db, tables, id, ctx?): Promise<void>
purge(db, tables, id, ctx?): Promise<void>     // hard delete
linkPerson(db, tables, topicId, personId): Promise<TopicPerson>
unlinkPerson(db, tables, topicId, personId): Promise<void>
listPersons(db, tables, topicId): Promise<TopicPerson[]>
linkOrg(db, tables, topicId, orgId): Promise<TopicOrg>
unlinkOrg(db, tables, topicId, orgId): Promise<void>
listOrgs(db, tables, topicId): Promise<TopicOrg[]>
importFromFile(db, tables, filePath, format?): Promise<ImportResult>
exportToFile(db, tables, filePath?, format?): Promise<ExportResult>
```

`filters` supports `search` and `status`. Pagination accepts
`{ cursor, limit, sort, order, all }` where `sort` is one of
`created_at | name | status`.

### runService

```ts
add(db, tables, input, ctx?): Promise<Run>        // started_at defaulted to now
list(db, tables, topicId, pagination?): Promise<Paginated<Run>>
get(db, tables, id): Promise<Run>
update(db, tables, id, input, ctx?): Promise<Run>
complete(db, tables, id, summary?, ctx?): Promise<Run>   // sets completed_at=now
fail(db, tables, id, summary?, ctx?): Promise<Run>
remove(db, tables, id, ctx?): Promise<void>
```

### sourceService

```ts
add(db, tables, input, ctx?): Promise<Source>
list(db, tables, runId, pagination?): Promise<Paginated<Source>>
get(db, tables, id): Promise<Source>
update(db, tables, id, input, ctx?): Promise<Source>
remove(db, tables, id, ctx?): Promise<void>
```

### findingService

```ts
add(db, tables, input, ctx?): Promise<Finding>
list(db, tables, runId, pagination?): Promise<Paginated<Finding>>
count(db, tables, runId): Promise<{ count: number }>
get(db, tables, id): Promise<Finding>
update(db, tables, id, input, ctx?): Promise<Finding>
remove(db, tables, id, ctx?): Promise<void>
textSearch(db, tables, query, opts?): Promise<Finding[]>
search(db, tables, query, opts): Promise<SearchResult[]>
```

`opts` for `search` accepts `{ mode, runId?, topicId?, limit? }` where mode
is one of `text | semantic | hybrid | reranked`. Embeddings are written
into `embedding` as F32_BLOB via libSQL's `vector32()`.

### briefingService

```ts
add(db, tables, input, ctx?): Promise<Briefing>
list(db, tables, topicId, pagination?): Promise<Paginated<Briefing>>
count(db, tables, topicId): Promise<{ count: number }>
get(db, tables, id): Promise<Briefing>
update(db, tables, id, input, ctx?): Promise<Briefing>
remove(db, tables, id, ctx?): Promise<void>

// Insights (live under briefingService)
addInsight(db, tables, input, ctx?): Promise<Insight>
listInsights(db, tables, briefingId): Promise<Insight[]>
getInsight(db, tables, id): Promise<Insight>
updateInsight(db, tables, id, input, ctx?): Promise<Insight>
removeInsight(db, tables, id, ctx?): Promise<void>
linkInsightFinding(db, tables, insightId, findingId, relevanceScore?): Promise<InsightFinding>
unlinkInsightFinding(db, tables, insightId, findingId): Promise<void>
listInsightFindings(db, tables, insightId): Promise<InsightFinding[]>
```

### Search services

`textSearch`, `vectorSearch`, and `hybridSearch` live as sibling services
in `@openclam/research/services`.

```ts
// textSearch (FTS5 with LIKE fallback)
textSearch(db, tables, query, opts?): Promise<Finding[]>

// vectorSearch
searchSimilar(db, tables, queryEmbedding, opts?): Promise<SearchResult[]>
search(db, tables, provider, query, opts?): Promise<SearchResult[]>

// hybridSearch (multi-mode, RRF fusion, optional LLM rerank)
search(db, tables, query, opts): Promise<SearchResult[]>
```

Common `opts`: `{ limit, threshold, runId, topicId, mode }`.

## CLI Commands

### clam research

```
clam --json research topic add <name>        --input-json '{"description":"...","color":"#aabbcc","status":"active"}'
clam --json research topic list              --input-json '{"search":"...","status":"active","cursor":"...","limit":25,"sort":"created_at","order":"desc","all":false,"fields":"id,name,status"}'
clam --json research topic count             --input-json '{"search":"...","status":"active"}'
clam --json research topic get <id>
clam --json research topic update <id>       --input-json '{"name":"...","status":"archived"}'
clam --json research topic remove <id>       --input-json '{"dry_run":false}'
clam --json research topic restore <id>
clam --json research topic purge <id>        --input-json '{"dry_run":false}'
clam --json research topic attach-person <topicId>   --input-json '{"person_id":"<uuid>"}'
clam --json research topic detach-person <topicId>   --input-json '{"person_id":"<uuid>"}'
clam --json research topic attach-org <topicId>      --input-json '{"org_id":"<uuid>"}'
clam --json research topic detach-org <topicId>      --input-json '{"org_id":"<uuid>"}'
clam --json research topic import <file>     --input-json '{"format":"csv"}'
clam --json research topic export <file>     --input-json '{"format":"json"}'

clam --json research run add <topicId>       --input-json '{"summary":"..."}'
clam --json research run list <topicId>      --input-json '{"cursor":"...","limit":25}'
clam --json research run get <id>
clam --json research run update <id>         --input-json '{"status":"completed","summary":"..."}'
clam --json research run complete <id>       --input-json '{"summary":"..."}'
clam --json research run fail <id>           --input-json '{"summary":"..."}'
clam --json research run remove <id>         --input-json '{"dry_run":false}'

clam --json research source add <runId>      --input-json '{"source_type":"webpage","url":"https://...","title":"...","author":"...","summary":"..."}'
clam --json research source list <runId>     --input-json '{"cursor":"...","limit":25,"fields":"id,title,url"}'
clam --json research source get <id>
clam --json research source update <id>      --input-json '{"title":"...","summary":"..."}'
clam --json research source remove <id>      --input-json '{"dry_run":false}'

clam --json research finding add <runId>     --input-json '{"content":"...","title":"...","source_id":"<uuid>","confidence":4}'
clam --json research finding list <runId>    --input-json '{"cursor":"...","limit":25,"sort":"created_at","order":"desc"}'
clam --json research finding count <runId>
clam --json research finding get <id>
clam --json research finding update <id>     --input-json '{"title":"...","content":"...","confidence":5}'
clam --json research finding remove <id>     --input-json '{"dry_run":false}'
clam --json research finding text-search     --input-json '{"query":"...","run_id":"...","topic_id":"...","limit":20}'
clam --json research finding search          --input-json '{"query":"...","mode":"hybrid","run_id":"...","topic_id":"...","limit":10}'

clam --json research briefing add <topicId>  --input-json '{"title":"...","summary":"..."}'
clam --json research briefing list <topicId> --input-json '{"cursor":"...","limit":25,"sort":"created_at","order":"desc"}'
clam --json research briefing count <topicId>
clam --json research briefing get <id>
clam --json research briefing update <id>    --input-json '{"title":"...","summary":"..."}'
clam --json research briefing remove <id>    --input-json '{"dry_run":false}'

clam --json research briefing insight add <briefingId>      --input-json '{"question":"...","content":"...","order":1,"finding_ids":["<uuid>"]}'
clam --json research briefing insight list <briefingId>
clam --json research briefing insight get <id>
clam --json research briefing insight update <id>           --input-json '{"question":"...","content":"...","order":2}'
clam --json research briefing insight remove <id>           --input-json '{"dry_run":false}'
clam --json research briefing insight attach-finding <insightId>   --input-json '{"finding_id":"<uuid>","relevance_score":85}'
clam --json research briefing insight detach-finding <insightId>   --input-json '{"finding_id":"<uuid>"}'
clam --json research briefing insight findings <insightId>
```

Every mutation subcommand accepts `--input-json` with an optional top-level
`decision` field carrying the decision trace (see `decisions.md`). Every
list/get on a single resource also accepts `--schema` which emits the JSON
Schema for that resource instead of running the query.

## Events emitted

`topic.created`, `topic.updated`, `topic.deleted`, `topic.restored`, `topic.purged`,
`run.created`, `run.updated`, `run.deleted`,
`source.created`, `source.updated`, `source.deleted`,
`finding.created`, `finding.updated`, `finding.deleted`.

See `../surfaces.md` to translate these operations to MCP, HTTP, or the
typed client.
