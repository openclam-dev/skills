# Contacts Domain Reference

Package: `@openclam/contacts`

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
| created_at | TEXT NOT NULL | ISO 8601 |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### orgs
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | |
| website | TEXT | Full URL (always starts with `https://`) |
| industry | TEXT | |
| employees | INTEGER | Number of employees |
| location | TEXT | |
| country | TEXT | |
| image_url | TEXT | Logo/image URL |
| accelerator | TEXT | Accelerator program |
| tagline | TEXT | Short tagline |
| description | TEXT | Full description |
| founded_year | INTEGER | Year founded |
| status | TEXT | Enum: `active`, `inactive`, `acquired` |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### person_emails
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id | TEXT NOT NULL | FK -> persons.id ON DELETE CASCADE |
| email | TEXT NOT NULL | |
| label | TEXT NOT NULL | e.g. `work`, `personal` |
| is_primary | INTEGER (boolean) NOT NULL | |
| created_at | TEXT NOT NULL | |

### org_emails
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| org_id | TEXT NOT NULL | FK -> orgs.id ON DELETE CASCADE |
| email | TEXT NOT NULL | |
| label | TEXT NOT NULL | |
| is_primary | INTEGER (boolean) NOT NULL | |
| created_at | TEXT NOT NULL | |

### person_phones
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id | TEXT NOT NULL | FK -> persons.id ON DELETE CASCADE |
| phone | TEXT NOT NULL | |
| label | TEXT NOT NULL | e.g. `mobile`, `work` |
| is_primary | INTEGER (boolean) NOT NULL | |
| created_at | TEXT NOT NULL | |

### org_phones
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| org_id | TEXT NOT NULL | FK -> orgs.id ON DELETE CASCADE |
| phone | TEXT NOT NULL | |
| label | TEXT NOT NULL | |
| is_primary | INTEGER (boolean) NOT NULL | |
| created_at | TEXT NOT NULL | |

### person_addresses
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id | TEXT NOT NULL | FK -> persons.id ON DELETE CASCADE |
| label | TEXT NOT NULL | |
| street | TEXT | |
| city | TEXT | |
| state | TEXT | |
| postal_code | TEXT | |
| country | TEXT | |
| is_primary | INTEGER (boolean) NOT NULL | |
| created_at | TEXT NOT NULL | |

### org_addresses
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| org_id | TEXT NOT NULL | FK -> orgs.id ON DELETE CASCADE |
| label | TEXT NOT NULL | |
| street | TEXT | |
| city | TEXT | |
| state | TEXT | |
| postal_code | TEXT | |
| country | TEXT | |
| is_primary | INTEGER (boolean) NOT NULL | |
| created_at | TEXT NOT NULL | |

### person_urls
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id | TEXT NOT NULL | FK -> persons.id ON DELETE CASCADE |
| url | TEXT NOT NULL | |
| label | TEXT NOT NULL | e.g. `website`, `linkedin` |
| is_primary | INTEGER (boolean) NOT NULL | |
| created_at | TEXT NOT NULL | |

### org_urls
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| org_id | TEXT NOT NULL | FK -> orgs.id ON DELETE CASCADE |
| url | TEXT NOT NULL | |
| label | TEXT NOT NULL | |
| is_primary | INTEGER (boolean) NOT NULL | |
| created_at | TEXT NOT NULL | |

### person_notes
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id | TEXT NOT NULL | FK -> persons.id ON DELETE CASCADE |
| title | TEXT | |
| body | TEXT NOT NULL | |
| source | TEXT NOT NULL | e.g. `cli`, `api` |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### org_notes
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| org_id | TEXT NOT NULL | FK -> orgs.id ON DELETE CASCADE |
| title | TEXT | |
| body | TEXT NOT NULL | |
| source | TEXT NOT NULL | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### person_comments
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id | TEXT NOT NULL | FK -> persons.id ON DELETE CASCADE |
| author_id | TEXT NOT NULL | User ID of the author |
| parent_id | TEXT | Parent comment ID for threading |
| body | TEXT NOT NULL | |
| status | TEXT NOT NULL | `open` or `resolved` |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

### org_comments
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| org_id | TEXT NOT NULL | FK -> orgs.id ON DELETE CASCADE |
| author_id | TEXT NOT NULL | User ID of the author |
| parent_id | TEXT | Parent comment ID for threading |
| body | TEXT NOT NULL | |
| status | TEXT NOT NULL | `open` or `resolved` |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

### person_tags
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id | TEXT NOT NULL | FK -> persons.id ON DELETE CASCADE |
| tag_id | TEXT NOT NULL | FK to core tags.id |
| created_at | TEXT NOT NULL | |

### org_tags
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| org_id | TEXT NOT NULL | FK -> orgs.id ON DELETE CASCADE |
| tag_id | TEXT NOT NULL | FK to core tags.id |
| created_at | TEXT NOT NULL | |

### relationships
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id_1 | TEXT NOT NULL | FK -> persons.id |
| person_id_2 | TEXT NOT NULL | FK -> persons.id |
| type | TEXT NOT NULL | `colleague`, `mentor`, `manager`, `referred_by`, `friend`, `other` |
| notes | TEXT | |
| created_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### segments
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | |
| entity_type | TEXT NOT NULL | `person` or `org` |
| filter_json | TEXT NOT NULL | JSON array of filter objects |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### project_persons
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| project_id | TEXT NOT NULL | FK -> projects.id ON DELETE CASCADE |
| person_id | TEXT NOT NULL | FK -> persons.id ON DELETE CASCADE |
| added_at | TEXT NOT NULL | |
| added_by | TEXT | User ID |

### project_orgs
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| project_id | TEXT NOT NULL | FK -> projects.id ON DELETE CASCADE |
| org_id | TEXT NOT NULL | FK -> orgs.id ON DELETE CASCADE |
| added_at | TEXT NOT NULL | |
| added_by | TEXT | User ID |

---

## Service Operations

### personService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, extras?, ctx?)` | Create person; extras: `emails[]`, `phones[]`, `urls[]` |
| `list(db, tables, filters?, pagination?, options?)` | List; filters: `orgId`, `search`, `tag`; options: `expand`; excludes soft-deleted |
| `get(db, tables, id, options?)` | Get with emails, phones, urls, org, tags; options: `expand`; excludes soft-deleted |
| `update(db, tables, id, input, ctx?)` | Update person fields |
| `remove(db, tables, id, ctx?)` | **Soft delete** |
| `restore(db, tables, id, ctx?)` | Restore soft-deleted person |
| `purge(db, tables, id, ctx?)` | **Hard delete** (cascades via ON DELETE CASCADE) |
| `merge(db, tables, winnerId, loserId)` | Merge loser into winner (moves emails/phones/urls/tags, soft-deletes loser) |
| `importFromFile(db, tables, filePath, format?)` | Import from CSV or JSON file |
| `importCsv(db, tables, rows)` | Import from array of row objects |
| `exportData(db, tables)` | Export all persons as flat rows |
| `exportToFile(db, tables, filePath?, format?)` | Export to file or return content |

Expand options for person: `emails`, `phones`, `urls`, `addresses`, `org`, `deals`, `tags`, `notes`, `relationships`, `log`

### orgService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create org |
| `list(db, tables, filters?, pagination?, options?)` | List; filters: `search`, `industry`, `tag`; options: `expand`; excludes soft-deleted |
| `get(db, tables, id, options?)` | Get with linked persons and tags; options: `expand`; excludes soft-deleted |
| `update(db, tables, id, input, ctx?)` | Update org fields |
| `remove(db, tables, id, ctx?)` | **Soft delete** |
| `restore(db, tables, id, ctx?)` | Restore soft-deleted org |
| `purge(db, tables, id, ctx?)` | **Hard delete** (cascades via ON DELETE CASCADE) |
| `importFromFile(db, tables, filePath, format?)` | Import from CSV or JSON file |
| `importCsv(db, tables, rows)` | Import from array of row objects |
| `exportData(db, tables)` | Export all orgs as flat rows |
| `exportToFile(db, tables, filePath?, format?)` | Export to file or return content |

Expand options for org: `persons`, `deals`, `urls`, `addresses`, `tags`, `notes`

### noteService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create note; input requires `entity_type` (`person`/`org`), `entity_id`, `body`, `source` |
| `list(db, tables, filters?, pagination?)` | List; filters: `entityType`, `entityId`, `search`; excludes soft-deleted |
| `get(db, tables, id)` | Get by ID across all note tables; excludes soft-deleted |
| `update(db, tables, id, input, ctx?)` | Update title/body |
| `remove(db, tables, id, ctx?)` | **Soft delete** |
| `restore(db, tables, id, ctx?)` | Restore soft-deleted note |
| `purge(db, tables, id, ctx?)` | **Hard delete** |
| `listForEntity(db, tables, entityType, entityId, pagination?)` | List notes for a specific entity |

### commentService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create comment; input: `entity_type`, `entity_id`, `author_id`, `body`, `parent_id?`; status defaults to `open` |
| `list(db, tables, entityType, entityId, filters?)` | List comments for entity; filter by `status` |
| `get(db, tables, entityType, id)` | Get single comment |
| `update(db, tables, entityType, id, input, ctx?)` | Update body/status |
| `remove(db, tables, entityType, id, ctx?)` | **Hard delete** |
| `resolve(db, tables, entityType, id, ctx?)` | Set status to `resolved` |
| `getThread(db, tables, entityType, parentId)` | Get child comments for a parent |

### relationshipService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create relationship; validates both persons exist, prevents self-relationships and duplicates |
| `get(db, tables, id)` | Get by ID; excludes soft-deleted |
| `listForPerson(db, tables, personId, pagination?)` | List relationships for a person with other person details |
| `update(db, tables, id, input, ctx?)` | Update `type` and/or `notes` |
| `remove(db, tables, id, ctx?)` | **Soft delete** |
| `restore(db, tables, id, ctx?)` | Restore soft-deleted relationship |
| `purge(db, tables, id, ctx?)` | **Hard delete** |

### segmentService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create segment; validates `filter_json` is valid JSON |
| `list(db, tables)` | List all; excludes soft-deleted |
| `get(db, tables, idOrName)` | Get by UUID or name with matchingCount; excludes soft-deleted |
| `run(db, tables, id)` | Execute segment filters, return matching persons or orgs |
| `update(db, tables, id, input, ctx?)` | Update name/filter_json/entity_type |
| `remove(db, tables, id, ctx?)` | **Soft delete** |
| `restore(db, tables, id, ctx?)` | Restore soft-deleted segment |
| `purge(db, tables, id, ctx?)` | **Hard delete** |

### projectLinkService
| Operation | Description |
|-----------|-------------|
| `addPersonToProject(db, tables, projectId, personId, ctx?)` | Link person to project |
| `removePersonFromProject(db, tables, projectId, personId)` | Unlink person from project |
| `listPersonsForProject(db, tables, projectId)` | List persons linked to a project |
| `listProjectsForPerson(db, tables, personId)` | List projects for a person |
| `addOrgToProject(db, tables, projectId, orgId, ctx?)` | Link org to project |
| `removeOrgFromProject(db, tables, projectId, orgId)` | Unlink org from project |
| `listOrgsForProject(db, tables, projectId)` | List orgs linked to a project |
| `listProjectsForOrg(db, tables, orgId)` | List projects for an org |

---

## CLI Commands

All list commands return `{ data: [...], nextCursor: string | null, hasMore: boolean }`.

### Person

```
clam --json contacts person add --input-json '{"first_name":"...","last_name":"...","org_id":"...","title":"...","summary":"...","image_url":"https://...","emails":[...],"phones":[...],"urls":[...]}'
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
```

#### Person Sub-entity Commands

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
```

#### Org Sub-entity Commands

```
# emails
clam --json contacts org email add --input-json '{"org_id":"...","email":"...","label":"work","is_primary":false}'
clam --json contacts org email list --input-json '{"org_id":"..."}'
clam --json contacts org email remove --input-json '{"id":"..."}'

# phones
clam --json contacts org phone add --input-json '{"org_id":"...","phone":"+12155551234","label":"work","is_primary":false}'
clam --json contacts org phone list --input-json '{"org_id":"..."}'
clam --json contacts org phone remove --input-json '{"id":"..."}'

# addresses
clam --json contacts org address add --input-json '{"org_id":"...","label":"hq","street":"...","city":"...","state":"...","postal_code":"...","country":"..."}'
clam --json contacts org address list --input-json '{"org_id":"..."}'
clam --json contacts org address remove --input-json '{"id":"..."}'

# urls
clam --json contacts org url add --input-json '{"org_id":"...","url":"https://...","label":"website","is_primary":false}'
clam --json contacts org url list --input-json '{"org_id":"..."}'
clam --json contacts org url remove --input-json '{"id":"..."}'

# comments
clam --json contacts org comment add --input-json '{"org_id":"...","body":"...","author_id":"...","parent_id":"..."}'
clam --json contacts org comment list --input-json '{"org_id":"...","status":"open","cursor":"...","limit":25}'
clam --json contacts org comment get --input-json '{"id":"..."}'
clam --json contacts org comment resolve --input-json '{"id":"..."}'
clam --json contacts org comment remove --input-json '{"id":"..."}'

# project links
clam --json contacts org project list --input-json '{"org_id":"..."}'
clam --json contacts org project add --input-json '{"org_id":"...","project_id":"..."}'
clam --json contacts org project remove --input-json '{"org_id":"...","project_id":"..."}'
```

### Note

```
clam --json contacts note add --input-json '{"entity_type":"person","entity_id":"...","title":"...","body":"..."}'
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
clam --json contacts relationship list --input-json '{"person_id":"...","cursor":"...","limit":25}'
clam --json contacts relationship update --input-json '{"id":"...","type":"mentor","notes":"..."}'
clam --json contacts relationship remove --input-json '{"id":"..."}'
clam --json contacts relationship restore --input-json '{"id":"..."}'
clam --json contacts relationship purge --input-json '{"id":"..."}'
```

---

## Paginated List Response

All `list` commands return:

```json
{
  "data": [ ... ],
  "nextCursor": "cursor-string-or-null",
  "hasMore": true
}
```

Pass `nextCursor` as `cursor` in `--input-json` to fetch the next page. Stop when `hasMore` is `false`.

---

## Global Flags

- `--json` -- Required. Produces compact JSON output (no pretty printing, no colors). Must be placed before the entity name.
- `--input-json '<json>'` -- Pass all input as structured JSON. Overrides positional args and individual flags.

---

## Input JSON Shapes

All commands accept `--input-json '<json>'`. The JSON shape for each:

#### person add

```typescript
interface PersonAddInput {
  first_name: string;              // required (or positional name arg)
  last_name?: string | null;
  org_id?: string | null;          // UUID
  title?: string | null;
  summary?: string | null;
  image_url?: string | null;       // valid URL
  emails?: Array<{
    email: string;                 // required
    label?: "work" | "personal" | "other";  // default: "work"
    is_primary?: boolean;          // default: false
  }>;
  phones?: Array<{
    phone: string;                 // required, E.164 format (e.g. "+12155551234")
    label?: "mobile" | "work" | "home" | "other";  // default: "mobile"
    is_primary?: boolean;          // default: false
  }>;
  urls?: Array<{
    url: string;                   // required, valid URL
    label?: string;                // default: "website"
    is_primary?: boolean;          // default: false
  }>;
}
```

#### person list

```typescript
interface PersonListInput {
  org_id?: string;
  search?: string;                 // FTS5 search across: first_name, last_name, title, summary
  tag?: string;
  cursor?: string;                 // pagination cursor
  limit?: number;
  expand?: string;                 // comma-separated: "emails,phones,urls,addresses,org,deals,tags,notes,relationships,log"
  fields?: string;                 // comma-separated columns to return (e.g. "id,first_name,last_name")
}
```

#### person update

```typescript
interface PersonUpdateInput {
  id: string;                      // required (or positional)
  first_name?: string;
  last_name?: string | null;
  org_id?: string | null;
  title?: string | null;
  summary?: string | null;
  image_url?: string | null;       // valid URL
}
```

#### person get

```typescript
interface PersonGetInput {
  id: string;                      // required (or positional)
  expand?: string;                 // comma-separated expand fields
}
```

#### person merge

```typescript
interface PersonMergeInput {
  source_id: string;               // required (or first positional) — the loser (gets soft-deleted)
  target_id: string;               // required (or second positional) — the winner
}
```

#### org add

```typescript
interface OrgAddInput {
  name: string;                    // required (or positional)
  website?: string | null;         // auto-prefixed with https:// if missing
  industry?: string | null;
  employees?: number | null;
  location?: string | null;
  country?: string | null;
  image_url?: string | null;       // valid URL
  accelerator?: string | null;
  tagline?: string | null;
  description?: string | null;
  founded_year?: number | null;    // integer
  status?: "active" | "inactive" | "acquired" | null;
}
```

#### org list

```typescript
interface OrgListInput {
  search?: string;                 // FTS5 search across: name, website, industry, location, tagline, description
  industry?: string;
  tag?: string;
  cursor?: string;
  limit?: number;
  expand?: string;                 // comma-separated: "persons,deals,urls,addresses,tags,notes"
  fields?: string;                 // comma-separated columns to return (e.g. "id,name,website")
}
```

#### org update

```typescript
interface OrgUpdateInput {
  id: string;                      // required (or positional)
  name?: string;
  website?: string | null;         // auto-prefixed with https:// if missing
  industry?: string | null;
  employees?: number | null;
  location?: string | null;
  country?: string | null;
  image_url?: string | null;       // valid URL
  accelerator?: string | null;
  tagline?: string | null;
  description?: string | null;
  founded_year?: number | null;    // integer
  status?: "active" | "inactive" | "acquired" | null;
}
```

#### person email add

```typescript
interface PersonEmailAddInput {
  person_id: string;               // required (or first positional)
  email: string;                   // required (or second positional), valid email
  label?: "work" | "personal" | "other";  // default: "work"
  is_primary?: boolean;           // default: false
}
```

#### org email add

```typescript
interface OrgEmailAddInput {
  org_id: string;                  // required (or first positional)
  email: string;                   // required (or second positional), valid email
  label?: "work" | "personal" | "other";  // default: "work"
  is_primary?: boolean;           // default: false
}
```

#### person phone add

```typescript
interface PersonPhoneAddInput {
  person_id: string;               // required (or first positional)
  phone: string;                   // required (or second positional), E.164 format
  label?: "mobile" | "work" | "home" | "other";  // default: "mobile"
  is_primary?: boolean;           // default: false
}
```

#### org phone add

```typescript
interface OrgPhoneAddInput {
  org_id: string;                  // required (or first positional)
  phone: string;                   // required (or second positional), E.164 format
  label?: "mobile" | "work" | "home" | "other";  // default: "mobile"
  is_primary?: boolean;           // default: false
}
```

#### person/org address add

```typescript
interface AddressAddInput {
  person_id?: string;              // required for person address (or positional)
  org_id?: string;                 // required for org address (or positional)
  label?: string;                  // default: "office"
  street?: string | null;
  city?: string | null;
  state?: string | null;
  postal_code?: string | null;
  country?: string | null;
  is_primary?: boolean;           // default: false
}
```

#### person url add

```typescript
interface PersonUrlAddInput {
  person_id: string;               // required (or first positional)
  url: string;                     // required (or second positional), valid URL
  label?: string;                  // default: "website"
  is_primary?: boolean;           // default: false
}
```

#### org url add

```typescript
interface OrgUrlAddInput {
  org_id: string;                  // required (or first positional)
  url: string;                     // required (or second positional), valid URL
  label?: string;                  // default: "website"
  is_primary?: boolean;           // default: false
}
```

#### person/org comment add

```typescript
interface CommentAddInput {
  person_id?: string;              // required for person comment (or positional)
  org_id?: string;                 // required for org comment (or positional)
  body: string;                    // required
  author_id?: string;              // UUID, defaults to active user
  parent_id?: string | null;       // UUID, for threading
}
```

#### person/org comment list

```typescript
interface CommentListInput {
  person_id?: string;              // required for person comment (or positional)
  org_id?: string;                 // required for org comment (or positional)
  status?: "open" | "resolved";
  cursor?: string;
  limit?: number;
}
```

#### note add

```typescript
interface NoteAddInput {
  entity_type: "person" | "org";   // required
  entity_id: string;               // required
  title?: string | null;
  body: string;                    // required
  source?: string;                 // default: "cli"
}
```

#### note list

```typescript
interface NoteListInput {
  entity_type?: "person" | "org";
  entity_id?: string;
  search?: string;
  cursor?: string;
  limit?: number;
}
```

#### note update

```typescript
interface NoteUpdateInput {
  id: string;                      // required (or positional)
  title?: string | null;
  body?: string;
}
```

#### relationship add

```typescript
interface RelationshipAddInput {
  person_id_1: string;             // required, UUID
  person_id_2: string;             // required, UUID
  type?: "colleague" | "mentor" | "manager" | "referred_by" | "friend" | "other";  // default: "other"
  notes?: string | null;
}
```

#### relationship list

```typescript
interface RelationshipListInput {
  person_id: string;               // required (or positional)
  cursor?: string;
  limit?: number;
}
```

#### relationship update

```typescript
interface RelationshipUpdateInput {
  id: string;                      // required (or positional)
  type?: "colleague" | "mentor" | "manager" | "referred_by" | "friend" | "other";
  notes?: string | null;
}
```

#### segment add

```typescript
interface SegmentAddInput {
  name: string;                    // required
  entity_type: "person" | "org";   // required
  filter_json: string;             // required, JSON array string
}
```

#### segment list

```typescript
interface SegmentListInput {
  cursor?: string;
  limit?: number;
}
```

#### segment update

```typescript
interface SegmentUpdateInput {
  id: string;                      // required (or positional)
  name?: string;
  entity_type?: "person" | "org";
  filter_json?: string;
}
```

#### person tag / person untag

```typescript
interface PersonTagInput {
  id: string;                      // required — person UUID
  slug: string;                    // required — tag slug (e.g. "yc", "b2b")
}
```

#### org tag / org untag

```typescript
interface OrgTagInput {
  id: string;                      // required — org UUID
  slug: string;                    // required — tag slug (e.g. "yc", "b2b")
}
```

---

## API Routes

### Persons
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/persons` | List persons; query: `search`, `orgId`, `tag` |
| POST | `/api/persons` | Create person |
| GET | `/api/persons/export` | Export all persons |
| POST | `/api/persons/import` | Import persons from JSON array |
| POST | `/api/persons/merge` | Merge persons; body: `{ winnerId, loserId }` |
| GET | `/api/persons/:id` | Get person with emails, phones, urls, org, tags |
| PUT | `/api/persons/:id` | Update person |
| DELETE | `/api/persons/:id` | Soft delete person |
| PUT | `/api/persons/:id/restore` | Restore soft-deleted person |
| DELETE | `/api/persons/:id/purge` | Permanently delete person |
| GET | `/api/persons/:id/briefing` | Full context briefing |
| GET | `/api/persons/:id/relationships` | List relationships for person |
| GET | `/api/persons/:id/projects` | List projects for person |

### Person URLs
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/persons/:id/urls` | List URLs for person |
| POST | `/api/persons/:id/urls` | Add URL; body: `{ url, label }` |
| DELETE | `/api/person-urls/:id` | Delete URL by row ID |

### Person Comments
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/persons/:id/comments` | List comments; query: `status` |
| POST | `/api/persons/:id/comments` | Add comment; body: `{ author_id, body, parent_id? }` |
| GET | `/api/person-comments/:id` | Get comment |
| PUT | `/api/person-comments/:id` | Update comment; body: `{ body?, status? }` |
| DELETE | `/api/person-comments/:id` | Delete comment (hard delete) |
| PUT | `/api/person-comments/:id/resolve` | Resolve comment |

### Orgs
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/orgs` | List orgs; query: `search`, `industry`, `tag` |
| POST | `/api/orgs` | Create org |
| GET | `/api/orgs/export` | Export all orgs |
| POST | `/api/orgs/import` | Import orgs from JSON array |
| GET | `/api/orgs/:id` | Get org with persons and tags |
| PUT | `/api/orgs/:id` | Update org |
| DELETE | `/api/orgs/:id` | Soft delete org |
| PUT | `/api/orgs/:id/restore` | Restore soft-deleted org |
| DELETE | `/api/orgs/:id/purge` | Permanently delete org |
| GET | `/api/orgs/:id/projects` | List projects for org |

### Org URLs
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/orgs/:id/urls` | List URLs for org |
| POST | `/api/orgs/:id/urls` | Add URL; body: `{ url, label }` |
| DELETE | `/api/org-urls/:id` | Delete URL by row ID |

### Org Comments
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/orgs/:id/comments` | List comments; query: `status` |
| POST | `/api/orgs/:id/comments` | Add comment; body: `{ author_id, body, parent_id? }` |
| GET | `/api/org-comments/:id` | Get comment |
| PUT | `/api/org-comments/:id` | Update comment; body: `{ body?, status? }` |
| DELETE | `/api/org-comments/:id` | Delete comment (hard delete) |
| PUT | `/api/org-comments/:id/resolve` | Resolve comment |

### Notes
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/notes` | List notes; query: `entityType`, `entityId` |
| POST | `/api/notes` | Create note |
| GET | `/api/notes/:id` | Get note |
| PUT | `/api/notes/:id` | Update note |
| DELETE | `/api/notes/:id` | Soft delete note |
| PUT | `/api/notes/:id/restore` | Restore soft-deleted note |
| DELETE | `/api/notes/:id/purge` | Permanently delete note |

### Relationships
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/relationships` | Create relationship |
| GET | `/api/relationships/:id` | Get relationship |
| PUT | `/api/relationships/:id` | Update relationship; body: `{ type?, notes? }` |
| DELETE | `/api/relationships/:id` | Soft delete relationship |
| PUT | `/api/relationships/:id/restore` | Restore soft-deleted relationship |
| DELETE | `/api/relationships/:id/purge` | Permanently delete relationship |

### Segments
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/segments` | List segments |
| POST | `/api/segments` | Create segment |
| GET | `/api/segments/:id` | Get segment with matching count |
| PUT | `/api/segments/:id` | Update segment |
| DELETE | `/api/segments/:id` | Soft delete segment |
| PUT | `/api/segments/:id/restore` | Restore soft-deleted segment |
| DELETE | `/api/segments/:id/purge` | Permanently delete segment |
| POST | `/api/segments/:id/run` | Run segment, return matching entities |

### Tags (entity-level, registered by contacts)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/tags/:entityType/:entityId` | Get tags for entity |
| POST | `/api/tags` | Apply tag; body: `{ entityType, entityId, tag }` |
| DELETE | `/api/tags/entity` | Remove tag; body: `{ entityType, entityId, tag }` |

### Project Links
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/projects/:id/persons` | Add person to project; body: `{ personId }` |
| DELETE | `/api/projects/:id/persons/:personId` | Remove person from project |
| GET | `/api/projects/:id/persons` | List persons for project |
| POST | `/api/projects/:id/orgs` | Add org to project; body: `{ orgId }` |
| DELETE | `/api/projects/:id/orgs/:orgId` | Remove org from project |
| GET | `/api/projects/:id/orgs` | List orgs for project |

### Stats
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stats/overview` | Overview stats |
| GET | `/api/stats/log` | Log metrics |
