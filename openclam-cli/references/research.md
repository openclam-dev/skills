# Research Reference

Package: `@openclam/research`
Dependencies: none

---

## Tables

### research_topics
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | Topic name |
| description | TEXT | |
| color | TEXT | Hex color |
| status | TEXT | e.g. `active` |
| created_by | TEXT | User UUID |
| created_at | TEXT NOT NULL | ISO 8601 |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete |

### research_topic_persons
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| topic_id | TEXT NOT NULL | FK → research_topics (CASCADE) |
| person_id | TEXT NOT NULL | Reference to contacts person |
| created_at | TEXT NOT NULL | |

### research_topic_orgs
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| topic_id | TEXT NOT NULL | FK → research_topics (CASCADE) |
| org_id | TEXT NOT NULL | Reference to contacts org |
| created_at | TEXT NOT NULL | |

### research_runs
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| topic_id | TEXT NOT NULL | FK → research_topics (CASCADE) |
| status | TEXT | `running`, `completed`, `failed` |
| summary | TEXT | |
| started_at | TEXT | |
| completed_at | TEXT | |
| created_at | TEXT NOT NULL | |

### research_sources
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| run_id | TEXT NOT NULL | FK → research_runs (CASCADE) |
| url | TEXT | Source URL |
| source_type | TEXT NOT NULL | Type identifier |
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
| content | TEXT NOT NULL | Finding content (min 1 char) |
| confidence | INTEGER | 1–5 scale |
| embedding | TEXT | JSON-encoded number[] |
| embedding_model | TEXT | Model name used for embedding |
| embedding_dimensions | INTEGER | Dimension count |
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
| content | TEXT NOT NULL | |
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

---

## Service Operations

### topicService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create topic |
| `list(db, tables, filters?, pagination?)` | List; filters: `search`, `status` |
| `get(db, tables, id)` | Get by ID |
| `update(db, tables, id, input, ctx?)` | Update fields |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore |
| `purge(db, tables, id, ctx?)` | Hard delete |
| `linkPerson(db, tables, topicId, personId)` | Link person to topic |
| `unlinkPerson(db, tables, topicId, personId)` | Unlink person |
| `listPersons(db, tables, topicId)` | List linked persons |
| `linkOrg(db, tables, topicId, orgId)` | Link org to topic |
| `unlinkOrg(db, tables, topicId, orgId)` | Unlink org |
| `listOrgs(db, tables, topicId)` | List linked orgs |

### runService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create run for topic |
| `list(db, tables, topicId, pagination?)` | List runs for topic |
| `get(db, tables, id)` | Get by ID |
| `update(db, tables, id, input, ctx?)` | Update status/summary |
| `complete(db, tables, id, summary?, ctx?)` | Set status=completed, completed_at=now |
| `fail(db, tables, id, summary?, ctx?)` | Set status=failed |
| `remove(db, tables, id, ctx?)` | Delete run |

### sourceService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Add source to run |
| `list(db, tables, runId, pagination?)` | List sources for run |
| `get(db, tables, id)` | Get by ID |
| `update(db, tables, id, input, ctx?)` | Update fields |
| `remove(db, tables, id, ctx?)` | Delete source |

### findingService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Add finding to run |
| `list(db, tables, runId, pagination?)` | List findings for run |
| `get(db, tables, id)` | Get by ID |
| `update(db, tables, id, input, ctx?)` | Update fields |
| `remove(db, tables, id, ctx?)` | Delete finding |
| `ingest(db, tables, provider, input, ctx?)` | Add finding + auto-embed content via AI |
| `ingestBatch(db, tables, provider, inputs[], ctx?)` | Batch add + embed |
| `embedMissing(db, tables, provider, runId, ctx?)` | Backfill embeddings for findings without them |

### briefingService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create briefing for topic |
| `list(db, tables, topicId, pagination?)` | List briefings |
| `get(db, tables, id)` | Get by ID |
| `update(db, tables, id, input, ctx?)` | Update title/summary |
| `remove(db, tables, id, ctx?)` | Delete briefing |
| `addInsight(db, tables, input, ctx?)` | Add insight with optional finding_ids |
| `listInsights(db, tables, briefingId)` | List insights |
| `getInsight(db, tables, id)` | Get insight |
| `updateInsight(db, tables, id, input, ctx?)` | Update insight |
| `removeInsight(db, tables, id, ctx?)` | Delete insight |
| `linkInsightFinding(db, tables, insightId, findingId, relevanceScore?)` | Link finding to insight |
| `unlinkInsightFinding(db, tables, insightId, findingId)` | Unlink |
| `listInsightFindings(db, tables, insightId)` | List linked findings |

### Search Services

#### textSearch
| Operation | Description |
|-----------|-------------|
| `textSearch(db, tables, query, opts?)` | FTS5 search with LIKE fallback; opts: `limit`, `runId`, `topicId` |

#### vectorSearch
| Operation | Description |
|-----------|-------------|
| `searchSimilar(db, tables, queryEmbedding, opts?)` | Cosine similarity on embeddings; opts: `limit`, `threshold`, `runId`, `topicId` |
| `search(db, tables, provider, query, opts?)` | Auto-embeds query text, then searches |

#### hybridSearch
| Operation | Description |
|-----------|-------------|
| `search(db, tables, query, opts)` | Multi-mode search with RRF fusion |

Search modes (passed in `opts.mode`):
- `text` — FTS only (no AI provider needed)
- `semantic` — Vector similarity only (requires embedding provider)
- `hybrid` — Text + semantic merged via Reciprocal Rank Fusion
- `reranked` — Hybrid + LLM reranking (requires reranker provider)

---

## CLI Commands

### research topic

```
clam --json research topic add --input-json '{"name":"...","description":"...","color":"...","status":"active"}'
clam --json research topic list --input-json '{"search":"...","status":"...","cursor":"...","limit":25}'
clam --json research topic get <id>
clam --json research topic update --input-json '{"id":"<id>","name":"...","status":"..."}'
clam --json research topic remove <id>
clam --json research topic restore <id>
clam --json research topic purge <id>
clam --json research topic attach-person --input-json '{"topic_id":"...","person_id":"..."}'
clam --json research topic detach-person --input-json '{"topic_id":"...","person_id":"..."}'
clam --json research topic attach-org --input-json '{"topic_id":"...","org_id":"..."}'
clam --json research topic detach-org --input-json '{"topic_id":"...","org_id":"..."}'
```

### research run

```
clam --json research run add --input-json '{"topic_id":"...","summary":"..."}'
clam --json research run list --input-json '{"topic_id":"...","cursor":"...","limit":25}'
clam --json research run get <id>
clam --json research run update --input-json '{"id":"<id>","status":"...","summary":"..."}'
clam --json research run complete --input-json '{"id":"<id>","summary":"..."}'
clam --json research run fail --input-json '{"id":"<id>","summary":"..."}'
clam --json research run remove <id>
```

### research source

```
clam --json research source add --input-json '{"run_id":"...","source_type":"...","url":"...","title":"...","author":"...","summary":"..."}'
clam --json research source list --input-json '{"run_id":"...","cursor":"...","limit":25}'
clam --json research source get <id>
clam --json research source update --input-json '{"id":"<id>","title":"...","summary":"..."}'
clam --json research source remove <id>
```

### research finding

```
clam --json research finding add --input-json '{"run_id":"...","content":"...","title":"...","source_id":"...","confidence":4}'
clam --json research finding list --input-json '{"run_id":"...","cursor":"...","limit":25}'
clam --json research finding get <id>
clam --json research finding update --input-json '{"id":"<id>","title":"...","content":"...","confidence":5}'
clam --json research finding remove <id>
clam --json research finding text-search --input-json '{"query":"...","run_id":"...","topic_id":"...","limit":10}'
clam --json research finding search --input-json '{"query":"...","mode":"hybrid","run_id":"...","topic_id":"...","limit":10}'
```

### research briefing

```
clam --json research briefing add --input-json '{"topic_id":"...","title":"...","summary":"..."}'
clam --json research briefing list --input-json '{"topic_id":"...","cursor":"...","limit":25}'
clam --json research briefing get <id>
clam --json research briefing update --input-json '{"id":"<id>","title":"...","summary":"..."}'
clam --json research briefing remove <id>
```

### research insight

```
clam --json research insight add --input-json '{"briefing_id":"...","question":"...","content":"...","order":1}'
clam --json research insight list --input-json '{"briefing_id":"..."}'
clam --json research insight get <id>
clam --json research insight update --input-json '{"id":"<id>","question":"...","content":"..."}'
clam --json research insight remove <id>
clam --json research insight attach-finding --input-json '{"insight_id":"...","finding_id":"...","relevance_score":85}'
clam --json research insight detach-finding --input-json '{"insight_id":"...","finding_id":"..."}'
clam --json research insight findings --input-json '{"insight_id":"..."}'
```

---

## API Routes

### Topics
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/research/topics` | List topics |
| POST | `/api/research/topics` | Create topic |
| GET | `/api/research/topics/:id` | Get topic |
| PUT | `/api/research/topics/:id` | Update |
| DELETE | `/api/research/topics/:id` | Soft delete |
| PUT | `/api/research/topics/:id/restore` | Restore |
| DELETE | `/api/research/topics/:id/purge` | Hard delete |
| GET | `/api/research/topics/:id/persons` | List linked persons |
| POST | `/api/research/topics/:id/persons` | Link person |
| DELETE | `/api/research/topics/:id/persons/:personId` | Unlink |
| GET | `/api/research/topics/:id/orgs` | List linked orgs |
| POST | `/api/research/topics/:id/orgs` | Link org |
| DELETE | `/api/research/topics/:id/orgs/:orgId` | Unlink |

### Runs
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/research/topics/:topicId/runs` | List runs |
| POST | `/api/research/topics/:topicId/runs` | Create run |
| GET | `/api/research/runs/:id` | Get run |
| PUT | `/api/research/runs/:id` | Update |
| PUT | `/api/research/runs/:id/complete` | Complete run |
| PUT | `/api/research/runs/:id/fail` | Fail run |
| DELETE | `/api/research/runs/:id` | Delete |

### Sources
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/research/runs/:runId/sources` | List sources |
| POST | `/api/research/runs/:runId/sources` | Add source |
| GET | `/api/research/sources/:id` | Get source |
| PUT | `/api/research/sources/:id` | Update |
| DELETE | `/api/research/sources/:id` | Delete |

### Findings
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/research/runs/:runId/findings` | List findings |
| POST | `/api/research/runs/:runId/findings` | Add finding |
| GET | `/api/research/findings/:id` | Get finding |
| PUT | `/api/research/findings/:id` | Update |
| DELETE | `/api/research/findings/:id` | Delete |
| GET | `/api/research/findings/text-search` | FTS search; query: `q`, `run_id`, `topic_id`, `limit` |
| POST | `/api/research/findings/search` | Unified search; body: `{ query, mode, run_id?, topic_id?, limit? }` |

### Briefings & Insights
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/research/topics/:topicId/briefings` | List briefings |
| POST | `/api/research/topics/:topicId/briefings` | Create briefing |
| GET | `/api/research/briefings/:id` | Get briefing |
| PUT | `/api/research/briefings/:id` | Update |
| DELETE | `/api/research/briefings/:id` | Delete |
| GET | `/api/research/briefings/:id/insights` | List insights |
| POST | `/api/research/briefings/:id/insights` | Create insight |
| GET | `/api/research/briefing-insights/:id` | Get insight |
| PUT | `/api/research/briefing-insights/:id` | Update |
| DELETE | `/api/research/briefing-insights/:id` | Delete |
| GET | `/api/research/briefing-insights/:id/findings` | List linked findings |
| POST | `/api/research/briefing-insights/:id/findings` | Link finding |
| DELETE | `/api/research/briefing-insights/:insightId/findings/:findingId` | Unlink |

---

## Events Emitted

`topic.created`, `topic.updated`, `topic.deleted`, `topic.restored`, `topic.purged`
`run.created`, `run.updated`, `run.deleted`
`source.created`, `source.updated`, `source.deleted`
`finding.created`, `finding.updated`, `finding.deleted`

## Permissions

`topic.create`, `topic.read`, `topic.update`, `topic.delete`
`run.create`, `run.read`, `run.update`, `run.delete`
`source.create`, `source.read`, `source.update`, `source.delete`
`finding.create`, `finding.read`, `finding.update`, `finding.delete`
`briefing.create`, `briefing.read`, `briefing.update`, `briefing.delete`
