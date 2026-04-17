---
name: operate-contacts
description: >
  Contacts and CRM — persons, orgs, sub-entities (emails, phones, addresses,
  urls), notes, comments, tags, relationships, segments, and project links.
  Covers the `clam contacts` CLI, service operations, and database schema.
metadata:
  author: openclam
  version: 0.1.0
---

# Contacts

Package: `@openclam/contacts` (extension: `contacts`)

---

## Tables

### persons
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| org_id | TEXT | FK to orgs.id (not enforced at DB level) |
| first_name | TEXT NOT NULL | |
| last_name | TEXT | |
| title | TEXT | Job title |
| summary | TEXT | |
| image_url | TEXT | Profile image URL |
| created_by | TEXT | User/agent ID that created the row |
| updated_by | TEXT | User/agent ID for last update |
| created_at | TEXT NOT NULL | ISO 8601 |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### orgs
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | |
| website | TEXT UNIQUE | Full URL (always starts with `https://`) |
| industry | TEXT | |
| employees | INTEGER | |
| location | TEXT | |
| country | TEXT | |
| image_url | TEXT | Logo/image URL |
| accelerator | TEXT | Accelerator program |
| tagline | TEXT | Short tagline |
| description | TEXT | Full description |
| founded_year | INTEGER | |
| status | TEXT | Enum: `active`, `inactive`, `acquired` |
| created_by | TEXT | |
| updated_by | TEXT | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### person_emails / org_emails
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id / org_id | TEXT NOT NULL | FK to parent, ON DELETE CASCADE |
| email | TEXT NOT NULL | |
| label | TEXT NOT NULL | e.g. `work`, `personal`, `other` |
| is_primary | INTEGER (boolean) NOT NULL | |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |

### person_phones / org_phones
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id / org_id | TEXT NOT NULL | FK to parent, ON DELETE CASCADE |
| phone | TEXT NOT NULL | E.164 format recommended |
| label | TEXT NOT NULL | e.g. `mobile`, `work`, `home`, `other` |
| is_primary | INTEGER (boolean) NOT NULL | |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |

### person_addresses / org_addresses
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id / org_id | TEXT NOT NULL | FK to parent, ON DELETE CASCADE |
| label | TEXT NOT NULL | e.g. `home`, `office`, `hq` |
| street | TEXT | |
| city | TEXT | |
| state | TEXT | |
| postal_code | TEXT | |
| country | TEXT | |
| is_primary | INTEGER (boolean) NOT NULL | |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |

### person_urls / org_urls
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id / org_id | TEXT NOT NULL | FK to parent, ON DELETE CASCADE |
| url | TEXT NOT NULL | |
| label | TEXT NOT NULL | e.g. `website`, `linkedin`, `github` |
| is_primary | INTEGER (boolean) NOT NULL | |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |

### person_notes / org_notes
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id / org_id | TEXT NOT NULL | FK to parent, ON DELETE CASCADE |
| title | TEXT | |
| body | TEXT NOT NULL | |
| source | TEXT NOT NULL | e.g. `cli`, `api`, `web` |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### person_comments / org_comments
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id / org_id | TEXT NOT NULL | FK to parent, ON DELETE CASCADE |
| author_id | TEXT NOT NULL | |
| parent_id | TEXT | Self-ref for threading |
| body | TEXT NOT NULL | |
| status | TEXT NOT NULL | `open` or `resolved` |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

### person_tags / org_tags
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id / org_id | TEXT NOT NULL | FK to parent, ON DELETE CASCADE |
| tag_id | TEXT NOT NULL | FK to global tags table |
| created_at | TEXT NOT NULL | |

### relationships
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id_1 | TEXT NOT NULL | FK to persons.id |
| person_id_2 | TEXT NOT NULL | FK to persons.id |
| type | TEXT NOT NULL | `colleague`, `mentor`, `manager`, `referred_by`, `friend`, `other` |
| notes | TEXT | |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### segments
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | |
| entity_type | TEXT NOT NULL | `person` or `org` |
| filter_json | TEXT NOT NULL | JSON array of filter objects |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### project_persons / project_orgs
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| project_id | TEXT NOT NULL | |
| person_id / org_id | TEXT NOT NULL | FK to parent, ON DELETE CASCADE |
| added_at | TEXT NOT NULL | |
| added_by | TEXT | |

---

## Service Operations

Services live under `extensions/contacts/src/services/*.ts` and are re-exported as
`personsService`, `orgsService`, `personSubResourcesService`,
`orgSubResourcesService`, `relationshipsService`, `segmentsService`,
`notesService`, `tagsService`, `commentsService`, `contextsService`,
`contactStatsService`, and `projectsService` from `services/index.ts`.

### personService
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, extras?, ctx?)` | Create person; extras: `emails[]`, `phones[]`, `urls[]` |
| `count(db, tables, filters?)` | Count persons |
| `list(db, tables, filters?, pagination?, options?)` | filters: `orgId`, `search`, `tag`; options: `expand`, `fields`; excludes soft-deleted |
| `get(db, tables, id, options?)` | Get with emails, phones, urls, org, tags; supports `expand`; excludes soft-deleted |
| `getByName(db, tables, name)` | Lookup by first+last name |
| `getByEmail(db, tables, email)` | Lookup by email address |
| `update(db, tables, id, input, ctx?)` | Update person fields |
| `remove(db, tables, id, ctx?)` | Soft delete (sets `deleted_at`) |
| `restore(db, tables, id, ctx?)` | Restore soft-deleted person |
| `purge(db, tables, id, ctx?)` | Hard delete (cascades via ON DELETE CASCADE) |
| `merge(db, tables, winnerId, loserId)` | Merge loser into winner (moves emails/phones/urls/tags, soft-deletes loser) |
| `importFromFile(db, tables, filePath, format?)` | Import from CSV or JSON file |
| `importCsv(db, tables, rows)` | Import from array of row objects |
| `exportData(db, tables)` | Export all persons as flat rows |
| `exportToFile(db, tables, filePath?, format?)` | Export to file or return content |

Expand options for person: `emails`, `phones`, `urls`, `addresses`, `org`, `deals`, `tags`, `notes`, `relationships`, `log`.

### orgService
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Create org |
| `count(db, tables, filters?)` | Count orgs |
| `list(db, tables, filters?, pagination?, options?)` | filters: `search`, `industry`, `tag`; options: `expand`, `fields`; excludes soft-deleted |
| `get(db, tables, id, options?)` | Get with linked persons and tags; supports `expand`; excludes soft-deleted |
| `getByField(db, tables, field, value)` | Lookup by website/name |
| `update(db, tables, id, input, ctx?)` | Update org fields |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore soft-deleted org |
| `purge(db, tables, id, ctx?)` | Hard delete |
| `importFromFile(db, tables, filePath, format?)` | Import from CSV or JSON file |
| `importCsv(db, tables, rows)` | Import from array of row objects |
| `exportData(db, tables)` | Export all orgs as flat rows |
| `exportToFile(db, tables, filePath?, format?)` | Export to file or return content |

Expand options for org: `persons`, `deals`, `urls`, `addresses`, `tags`, `notes`, plus any registered enrichments (see `@openclam/core` `listEnrichmentNames('orgs')`).

### noteService
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Input: `entity_type` (`person`/`org`), `entity_id`, `body`, `title?`, `source` |
| `count(db, tables, filters?)` | Count notes |
| `list(db, tables, filters?, pagination?)` | filters: `entityType`, `entityId`, `search`; excludes soft-deleted |
| `get(db, tables, id)` | Get by ID across all note tables; excludes soft-deleted |
| `update(db, tables, id, input, ctx?)` | Update title/body |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore soft-deleted note |
| `purge(db, tables, id, ctx?)` | Hard delete |
| `listForEntity(db, tables, entityType, entityId, pagination?)` | Notes for a specific entity |

### commentService
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Input: `entity_type`, `entity_id`, `author_id`, `body`, `parent_id?`; status defaults to `open` |
| `list(db, tables, entityType, entityId, filters?)` | Filter by `status` |
| `get(db, tables, entityType, id)` | Single comment |
| `update(db, tables, entityType, id, input, ctx?)` | Update body/status |
| `remove(db, tables, entityType, id, ctx?)` | Hard delete |
| `resolve(db, tables, entityType, id, ctx?)` | Set status to `resolved` |
| `getThread(db, tables, entityType, parentId)` | Child comments for a parent |

### relationshipService
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Validates both persons exist, prevents self-refs and duplicates |
| `get(db, tables, id)` | Excludes soft-deleted |
| `countForPerson(db, tables, personId)` | Count of non-deleted relationships |
| `listForPerson(db, tables, personId, pagination?)` | With other-person details |
| `update(db, tables, id, input, ctx?)` | Update `type` and/or `notes` |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore |
| `purge(db, tables, id, ctx?)` | Hard delete |

### segmentService
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Validates `filter_json` is a valid JSON array |
| `count(db, tables)` | Count non-deleted segments |
| `list(db, tables, pagination?)` | Excludes soft-deleted |
| `get(db, tables, idOrName)` | Lookup by UUID or name; returns `matchingCount` |
| `run(db, tables, id)` | Execute filters, return matching persons or orgs |
| `update(db, tables, id, input, ctx?)` | Update name/filter_json/entity_type |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore |
| `purge(db, tables, id, ctx?)` | Hard delete |

### tagService
| Operation | Signature / Notes |
|-----------|-------------------|
| `applyBySlug(db, tables, entityType, entityId, slug)` | Apply tag by slug |
| `apply(db, tables, entityType, entityId, tagLabel)` | Get-or-create tag, then link; idempotent |
| `removeBySlug(db, tables, entityType, entityId, slug)` | Remove tag by slug |
| `remove(db, tables, entityType, entityId, tagLabel)` | Remove tag link |
| `getForEntity(db, tables, entityType, entityId)` | Tag rows with tag object |
| `getTagLabelsForEntity(db, tables, entityType, entityId)` | `string[]` |

### projectLinkService
| Operation | Signature / Notes |
|-----------|-------------------|
| `addPersonToProject(db, tables, projectId, personId, ctx?)` | |
| `removePersonFromProject(db, tables, projectId, personId)` | |
| `listPersonsForProject(db, tables, projectId)` | |
| `listProjectsForPerson(db, tables, personId)` | |
| `addOrgToProject(db, tables, projectId, orgId, ctx?)` | |
| `removeOrgFromProject(db, tables, projectId, orgId)` | |
| `listOrgsForProject(db, tables, projectId)` | |
| `listProjectsForOrg(db, tables, orgId)` | |

### contextService / statsService
- `contextService.briefing(db, tables, personId)` — full context briefing
- `statsService.overview(db, tables)` / `statsService.logMetrics(db, tables)`

---

## CLI Commands

All list commands return `{ data: [...], nextCursor: string | null, hasMore: boolean }`.

### Person

```
clam --json contacts person add --input-json '{"first_name":"...","last_name":"...","org_id":"...","title":"...","summary":"...","image_url":"https://...","emails":[...],"phones":[...],"urls":[...]}'
clam --json contacts person count --input-json '{"search":"...","org_id":"...","tag":"..."}'
clam --json contacts person list --input-json '{"search":"...","org_id":"...","tag":"...","fields":"id,first_name,last_name","limit":25,"cursor":"..."}'
clam --json contacts person get --input-json '{"id":"...","expand":"emails,phones,urls,addresses,org,deals,tags,notes,relationships,log"}'
clam --json contacts person update --input-json '{"id":"...","first_name":"...","last_name":"...","org_id":"...","title":"...","summary":"...","image_url":"..."}'
clam --json contacts person remove --input-json '{"id":"..."}'
clam --json contacts person restore --input-json '{"id":"..."}'
clam --json contacts person purge --input-json '{"id":"..."}'
clam --json contacts person merge --input-json '{"source_id":"...","target_id":"..."}'
clam --json contacts person import --input-json '{"file":"...","format":"csv"}'
clam --json contacts person export --input-json '{"file":"...","format":"csv"}'
clam --json contacts person context --input-json '{"id":"..."}'
clam --json contacts person stats
clam --json contacts person tag --input-json '{"id":"<person-id>","slug":"yc"}'
clam --json contacts person untag --input-json '{"id":"<person-id>","slug":"yc"}'
clam --json contacts person bulk-tag --input-json '{"filter":"...","slug":"..."}'
```

#### Person sub-entities

```
# emails
clam --json contacts person email add --input-json '{"person_id":"...","email":"...","label":"work","is_primary":false}'
clam --json contacts person email list --input-json '{"person_id":"..."}'
clam --json contacts person email remove --input-json '{"id":"..."}'

# phones
clam --json contacts person phone add --input-json '{"person_id":"...","phone":"+12155551234","label":"mobile","is_primary":false}'
clam --json contacts person phone list --input-json '{"person_id":"..."}'
clam --json contacts person phone remove --input-json '{"id":"..."}'

# addresses
clam --json contacts person address add --input-json '{"person_id":"...","label":"office","street":"...","city":"...","state":"...","postal_code":"...","country":"..."}'
clam --json contacts person address list --input-json '{"person_id":"..."}'
clam --json contacts person address remove --input-json '{"id":"..."}'

# urls
clam --json contacts person url add --input-json '{"person_id":"...","url":"https://...","label":"website","is_primary":false}'
clam --json contacts person url list --input-json '{"person_id":"..."}'
clam --json contacts person url remove --input-json '{"id":"..."}'

# comments
clam --json contacts person comment add --input-json '{"person_id":"...","body":"...","author_id":"...","parent_id":"..."}'
clam --json contacts person comment list --input-json '{"person_id":"...","status":"open","cursor":"...","limit":25}'
clam --json contacts person comment get --input-json '{"id":"..."}'
clam --json contacts person comment resolve --input-json '{"id":"..."}'
clam --json contacts person comment remove --input-json '{"id":"..."}'

# project links
clam --json contacts person project list --input-json '{"person_id":"..."}'
clam --json contacts person project add --input-json '{"person_id":"...","project_id":"..."}'
clam --json contacts person project remove --input-json '{"person_id":"...","project_id":"..."}'
```

### Org

```
clam --json contacts org add --input-json '{"name":"...","website":"https://...","industry":"...","employees":100,"location":"...","country":"...","image_url":"https://...","accelerator":"...","tagline":"...","description":"...","founded_year":2024,"status":"active"}'
clam --json contacts org count --input-json '{"search":"...","industry":"...","tag":"..."}'
clam --json contacts org list --input-json '{"search":"...","industry":"...","tag":"...","fields":"id,name,website","limit":25,"cursor":"...","expand":"persons,deals,urls,addresses,tags,notes"}'
clam --json contacts org get --input-json '{"id":"...","expand":"persons,deals,urls,addresses,tags,notes"}'
clam --json contacts org update --input-json '{"id":"...","name":"...","website":"...","tagline":"...","description":"...","status":"acquired"}'
clam --json contacts org remove --input-json '{"id":"..."}'
clam --json contacts org restore --input-json '{"id":"..."}'
clam --json contacts org purge --input-json '{"id":"..."}'
clam --json contacts org import --input-json '{"file":"...","format":"csv"}'
clam --json contacts org export --input-json '{"file":"...","format":"csv"}'
clam --json contacts org tag --input-json '{"id":"<org-id>","slug":"yc"}'
clam --json contacts org untag --input-json '{"id":"<org-id>","slug":"yc"}'
clam --json contacts org bulk-tag --input-json '{"filter":"...","slug":"..."}'
```

#### Org sub-entities

Mirror the person sub-entity commands with `org_id` substituted for `person_id`:
`org email`, `org phone`, `org address`, `org url`, `org comment`, and `org project` follow the same `add`/`list`/`get`/`update`/`resolve`/`remove` shapes as their person counterparts.

### Note

```
clam --json contacts note add --input-json '{"entity_type":"person","entity_id":"...","title":"...","body":"..."}'
clam --json contacts note count --input-json '{"entity_type":"person","entity_id":"..."}'
clam --json contacts note list --input-json '{"entity_type":"person","entity_id":"...","search":"...","cursor":"...","limit":25}'
clam --json contacts note get --input-json '{"id":"..."}'
clam --json contacts note update --input-json '{"id":"...","title":"...","body":"..."}'
clam --json contacts note remove --input-json '{"id":"..."}'
clam --json contacts note restore --input-json '{"id":"..."}'
clam --json contacts note purge --input-json '{"id":"..."}'
```

### Segment

```
clam --json contacts segment add --input-json '{"name":"...","entity_type":"person","filter_json":"[...]"}'
clam --json contacts segment count
clam --json contacts segment list --input-json '{"cursor":"...","limit":25}'
clam --json contacts segment get --input-json '{"id":"..."}'
clam --json contacts segment run --input-json '{"id":"..."}'
clam --json contacts segment update --input-json '{"id":"...","name":"...","filter_json":"[...]","entity_type":"person"}'
clam --json contacts segment remove --input-json '{"id":"..."}'
clam --json contacts segment restore --input-json '{"id":"..."}'
clam --json contacts segment purge --input-json '{"id":"..."}'
```

### Relationship

```
clam --json contacts relationship add --input-json '{"person_id_1":"...","person_id_2":"...","type":"colleague","notes":"..."}'
clam --json contacts relationship count --input-json '{"person_id":"..."}'
clam --json contacts relationship list --input-json '{"person_id":"...","cursor":"...","limit":25}'
clam --json contacts relationship update --input-json '{"id":"...","type":"mentor","notes":"..."}'
clam --json contacts relationship remove --input-json '{"id":"..."}'
clam --json contacts relationship restore --input-json '{"id":"..."}'
clam --json contacts relationship purge --input-json '{"id":"..."}'
```

---

## Input JSON Shapes

#### person add
```typescript
interface PersonAddInput {
  first_name: string;
  last_name?: string | null;
  org_id?: string | null;
  title?: string | null;
  summary?: string | null;
  image_url?: string | null;
  emails?: Array<{ email: string; label?: "work" | "personal" | "other"; is_primary?: boolean }>;
  phones?: Array<{ phone: string; label?: "mobile" | "work" | "home" | "other"; is_primary?: boolean }>;
  urls?: Array<{ url: string; label?: string; is_primary?: boolean }>;
}
```

#### person list / org list
```typescript
interface ListInput {
  search?: string;
  cursor?: string;
  limit?: number;
  expand?: string;   // comma-separated
  fields?: string;   // comma-separated column projection
  tag?: string;
  // person-only: org_id
  // org-only: industry
}
```

#### person merge
```typescript
interface PersonMergeInput {
  source_id: string;  // loser (soft-deleted)
  target_id: string;  // winner
}
```

#### org add / update
```typescript
interface OrgAddInput {
  name: string;
  website?: string | null;       // auto-prefixed with https:// if missing
  industry?: string | null;
  employees?: number | null;
  location?: string | null;
  country?: string | null;
  image_url?: string | null;
  accelerator?: string | null;
  tagline?: string | null;
  description?: string | null;
  founded_year?: number | null;
  status?: "active" | "inactive" | "acquired" | null;
}
```

#### note add
```typescript
interface NoteAddInput {
  entity_type: "person" | "org";
  entity_id: string;
  title?: string | null;
  body: string;
  source?: string; // default: "cli"
}
```

#### relationship add
```typescript
interface RelationshipAddInput {
  person_id_1: string;
  person_id_2: string;
  type?: "colleague" | "mentor" | "manager" | "referred_by" | "friend" | "other";
  notes?: string | null;
}
```

#### segment add
```typescript
interface SegmentAddInput {
  name: string;
  entity_type: "person" | "org";
  filter_json: string; // JSON array string
}
```

#### tag / untag
```typescript
interface TagInput {
  id: string;    // person or org UUID
  slug: string;  // tag slug
}
```

---

See `../surfaces.md` to translate these operations to MCP, HTTP, or the typed client.
