---
name: operate-content
description: >
  Versioned content management — content types, items, immutable versions,
  assets, collections, comments, tags, channels, locales, and relations.
  Covers the `clam content` CLI, service operations, and database schema.
metadata:
  author: openclam
  version: 0.1.0
---

# Content

Package: `@openclam/content` (extension: `content`)

---

## Tables

### content_types
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| slug | TEXT NOT NULL UNIQUE | |
| name | TEXT NOT NULL | |
| description | TEXT | |
| icon | TEXT | |
| body_format | TEXT NOT NULL | e.g. `markdown`, `html`, `plain` |
| schema_json | TEXT | JSON Schema for metadata validation |
| default_status | TEXT NOT NULL | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### content_items
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| type | TEXT NOT NULL | Content type slug |
| slug | TEXT UNIQUE | |
| status | TEXT NOT NULL | |
| author_id | TEXT | |
| person_id | TEXT | FK to contacts persons |
| org_id | TEXT | FK to contacts orgs |
| published_version_id | TEXT | FK to content_versions.id |
| scheduled_at | TEXT | ISO 8601 |
| published_at | TEXT | ISO 8601 |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### content_versions
Immutable snapshots — versions are never updated, only new ones created.

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| content_id | TEXT NOT NULL | FK to content_items.id ON DELETE CASCADE |
| version | INTEGER NOT NULL | Auto-incremented per content item |
| title | TEXT NOT NULL | |
| body | TEXT | |
| summary | TEXT | |
| metadata_json | TEXT | |
| author_id | TEXT | |
| change_note | TEXT | |
| created_at | TEXT NOT NULL | |

### assets
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| filename | TEXT NOT NULL | |
| mime_type | TEXT NOT NULL | |
| size_bytes | INTEGER | |
| url | TEXT NOT NULL | |
| alt_text | TEXT | |
| width | INTEGER | |
| height | INTEGER | |
| duration_seconds | INTEGER | |
| metadata_json | TEXT | |
| uploaded_by | TEXT | |
| created_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### content_assets
Join table linking content items to assets.

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| content_id | TEXT NOT NULL | FK to content_items.id ON DELETE CASCADE |
| asset_id | TEXT NOT NULL | FK to assets.id |
| role | TEXT NOT NULL | e.g. `hero`, `thumbnail` |
| position | INTEGER NOT NULL | |
| created_at | TEXT NOT NULL | |

### collections
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | |
| slug | TEXT UNIQUE | |
| type | TEXT NOT NULL | |
| description | TEXT | |
| parent_id | TEXT | Self-ref for nested collections |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### collection_items
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| collection_id | TEXT NOT NULL | FK to collections.id ON DELETE CASCADE |
| content_id | TEXT NOT NULL | FK to content_items.id ON DELETE CASCADE |
| position | INTEGER NOT NULL | |
| created_at | TEXT NOT NULL | |

### content_tags
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| content_id | TEXT NOT NULL | FK to content_items.id ON DELETE CASCADE |
| tag_id | TEXT NOT NULL | FK to global tags table |
| created_at | TEXT NOT NULL | |

### content_notes
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| content_id | TEXT NOT NULL | FK to content_items.id ON DELETE CASCADE |
| title | TEXT | |
| body | TEXT NOT NULL | |
| source | TEXT NOT NULL | e.g. `cli`, `api`, `web` |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

### content_comments
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| content_id | TEXT NOT NULL | FK to content_items.id ON DELETE CASCADE |
| version_id | TEXT | FK to content_versions.id (pin to a version) |
| author_id | TEXT NOT NULL | |
| parent_id | TEXT | Self-ref for threading |
| body | TEXT NOT NULL | |
| anchor | TEXT | Reference to a specific content location |
| status | TEXT NOT NULL | `open` or `resolved` |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

### content_channels
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| content_id | TEXT NOT NULL | FK to content_items.id ON DELETE CASCADE |
| channel | TEXT NOT NULL | e.g. `web`, `email`, `rss` |
| external_id | TEXT | |
| status | TEXT NOT NULL | |
| published_at | TEXT | ISO 8601 |
| created_at | TEXT NOT NULL | |

### content_relations
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| source_id | TEXT NOT NULL | FK to content_items.id ON DELETE CASCADE |
| target_id | TEXT NOT NULL | FK to content_items.id ON DELETE CASCADE |
| type | TEXT NOT NULL | Relation type label |
| position | INTEGER | |
| created_at | TEXT NOT NULL | |

### content_locales
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| content_id | TEXT NOT NULL | FK to content_items.id ON DELETE CASCADE |
| version_id | TEXT NOT NULL | FK to content_versions.id ON DELETE CASCADE |
| locale | TEXT NOT NULL | e.g. `en-US`, `fr-FR` |
| created_at | TEXT NOT NULL | |

### project_content
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| project_id | TEXT NOT NULL | |
| content_id | TEXT NOT NULL | FK to content_items.id ON DELETE CASCADE |
| added_at | TEXT NOT NULL | |
| added_by | TEXT | |

---

## Service Operations

### contentTypeService (`services/content-type.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input)` | Input: `name`, `slug`, `body_format`, `default_status`, plus `description?`, `icon?`, `schema_json?` |
| `list(db, tables, filters?)` | filter: `search`; excludes soft-deleted |
| `get(db, tables, id)` | By ID or slug |
| `update(db, tables, id, input)` | Update `name`, `slug`, `body_format`, `description`, `icon` |
| `remove(db, tables, id)` | Soft delete |
| `restore(db, tables, id)` | Restore |
| `purge(db, tables, id)` | Hard delete |

### contentItemService (`services/content-item.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Creates the content item and an initial version |
| `count(db, tables, filters?)` | |
| `list(db, tables, filters?, pagination?, opts?)` | filters: `search`, `type`, `status`, `author_id`; opts: `expand` (`versions`, `tags`, `notes`) |
| `get(db, tables, id, opts?)` | By ID or slug; supports `expand` |
| `update(db, tables, id, input, ctx?)` | Update `status`, `slug`, `title`, `metadata_json` |
| `publish(db, tables, id, versionId, ctx?)` | Sets `published_version_id`, status to published, sets `published_at` |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore |
| `purge(db, tables, id, ctx?)` | Hard delete |
| `importFromFile(...)` / `exportToFile(...)` | Bulk import/export |

### contentVersionService (`services/content-version.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Creates a new version; auto-increments version number |
| `list(db, tables, contentId, pagination?)` | Versions for a content item |
| `get(db, tables, versionId)` | Single version |

### contentCommentService (`services/comment.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Input: `content_id`, `author_id`, `body`, `version_id?`, `parent_id?`, `anchor?`; status defaults to `open` |
| `list(db, tables, contentId, filters?, pagination?)` | filters: `status`, `version_id` |
| `get(db, tables, id)` | |
| `update(db, tables, id, input, ctx?)` | Update `body`, `status` |
| `resolve(db, tables, id, ctx?)` | Sets status to `resolved` |
| `remove(db, tables, id, ctx?)` | Hard delete |
| `getThread(db, tables, parentId)` | Child comments |

### contentNoteService (`services/note.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Input: `content_id`, `body`, `title?`, `source` |
| `list(db, tables, contentId, pagination?)` | |
| `get(db, tables, id)` | |
| `update(db, tables, id, input, ctx?)` | Update `title`, `body` |
| `remove(db, tables, id, ctx?)` | Hard delete |

### contentChannelService (`services/channel.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Input: `content_id`, `channel`, `external_id?`, `status?`, `published_at?` |
| `list(db, tables, contentId, pagination?)` | |
| `get(db, tables, id)` | |
| `update(db, tables, id, input, ctx?)` | Update `channel`, `external_id`, `status`, `published_at` |
| `remove(db, tables, id, ctx?)` | Hard delete |

### contentRelationService (`services/relation.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input)` | Input: `source_id`, `target_id`, `type`, `position?` |
| `listForContent(db, tables, contentId)` | Relations where content is source or target |
| `remove(db, tables, id)` | Hard delete |

### contentLocaleService (`services/locale.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input)` | Input: `content_id`, `version_id`, `locale` |
| `listForContent(db, tables, contentId)` | |
| `getForLocale(db, tables, contentId, locale)` | |
| `remove(db, tables, id)` | Hard delete |

### contentAssetService (`services/content-asset.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `link(db, tables, input)` | Link asset to content. Input: `content_id`, `asset_id`, `role?`, `position?` |
| `listForContent(db, tables, contentId)` | |
| `unlink(db, tables, id)` | Hard delete of the join row |

### assetService (`services/asset.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Input: `filename`, `mime_type`, `url`, plus optional size/alt/width/height/duration/metadata/uploaded_by |
| `count(db, tables, filters?)` | |
| `list(db, tables, filters?, pagination?)` | filters: `search`, `mime_type` |
| `get(db, tables, id)` | |
| `update(db, tables, id, input)` | Update `alt_text`, `metadata_json` |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore |
| `purge(db, tables, id, ctx?)` | Hard delete |

### collectionService (`services/collection.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Input: `name`, `type`, `slug?`, `description?`, `parent_id?` |
| `count(db, tables, filters?)` | |
| `list(db, tables, filters?, pagination?)` | filters: `search`, `type`, `parent_id` |
| `get(db, tables, id)` | By ID or slug |
| `update(db, tables, id, input, ctx?)` | |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore |
| `purge(db, tables, id, ctx?)` | Hard delete |
| `getTree(db, tables, rootId?)` | Hierarchical tree |

### collectionItemService (`services/collection-item.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input)` | Input: `collection_id`, `content_id`, `position?` |
| `listForCollection(db, tables, collectionId)` | Ordered by `position` |
| `remove(db, tables, id)` | |

### contentTagService (`services/tag.ts`)
`getForContent`, `apply(db, tables, contentId, tagSlug)`, `remove(db, tables, contentId, tagSlug)`.

### projectLinkService (`services/project-link.ts`)
`addContentToProject`, `removeContentFromProject`, `listContentForProject`, `listProjectsForContent`.

---

## CLI Commands

All list commands return `{ data: [...], nextCursor: string | null, hasMore: boolean }`.

Top-level command groups under `clam content`:
- `item` — content items (with nested `version`, `note`, `comment`, `channel`, `relation`, `locale`, `asset`, `tag`)
- `type` — content types
- `asset` — assets
- `collection` — collections (with nested `item`)

### content item

```
clam --json content item add --input-json '{"type":"blog-post","title":"Hello","body":"...","summary":"...","slug":"hello","metadata_json":"{...}","author_id":"..."}'
clam --json content item count --input-json '{"type":"blog-post","status":"published"}'
clam --json content item list --input-json '{"type":"blog-post","status":"published","search":"...","author_id":"...","cursor":"...","limit":25,"expand":"versions,tags,notes"}'
clam --json content item get --input-json '{"id":"<content-id>","expand":"versions,tags,notes"}'
clam --json content item update --input-json '{"id":"<content-id>","status":"published","title":"...","slug":"...","metadata_json":"{...}"}'
clam --json content item publish --input-json '{"id":"<content-id>","version_id":"<version-id>"}'
clam --json content item remove --input-json '{"id":"<content-id>"}'
clam --json content item restore --input-json '{"id":"<content-id>"}'
clam --json content item purge --input-json '{"id":"<content-id>"}'
clam --json content item import --input-json '{"file":"./content.json","format":"json"}'
clam --json content item export --input-json '{"file":"./content.json","format":"json"}'
clam --json content item projects --input-json '{"content_id":"<content-id>"}'
```

### content item version

```
clam --json content item version add --input-json '{"content_id":"<id>","title":"v2","body":"...","summary":"...","metadata_json":"{...}","change_note":"..."}'
clam --json content item version list --input-json '{"content_id":"<id>","cursor":"...","limit":25}'
clam --json content item version get --input-json '{"id":"<version-id>"}'
```

### content item note

```
clam --json content item note add --input-json '{"content_id":"<id>","body":"...","title":"..."}'
clam --json content item note list --input-json '{"content_id":"<id>","cursor":"...","limit":25}'
clam --json content item note get --input-json '{"id":"<note-id>"}'
clam --json content item note update --input-json '{"id":"<note-id>","body":"...","title":"..."}'
clam --json content item note remove --input-json '{"id":"<note-id>"}'
```

### content item comment

```
clam --json content item comment add --input-json '{"content_id":"<id>","body":"...","version_id":"<vid>","parent_id":"<pid>","anchor":"section-2","author_id":"..."}'
clam --json content item comment list --input-json '{"content_id":"<id>","status":"open","version_id":"<vid>","cursor":"...","limit":25}'
clam --json content item comment get --input-json '{"id":"<comment-id>"}'
clam --json content item comment update --input-json '{"id":"<comment-id>","body":"...","status":"resolved"}'
clam --json content item comment resolve --input-json '{"id":"<comment-id>"}'
clam --json content item comment remove --input-json '{"id":"<comment-id>"}'
clam --json content item comment thread --input-json '{"parent_id":"<parent-id>"}'
```

### content item channel

```
clam --json content item channel add --input-json '{"content_id":"<id>","channel":"web","status":"published","external_id":"...","published_at":"..."}'
clam --json content item channel list --input-json '{"content_id":"<id>","cursor":"...","limit":25}'
clam --json content item channel get --input-json '{"id":"<channel-entry-id>"}'
clam --json content item channel update --input-json '{"id":"<id>","channel":"...","status":"...","external_id":"...","published_at":"..."}'
clam --json content item channel remove --input-json '{"id":"<id>"}'
```

### content item relation

```
clam --json content item relation add --input-json '{"source_id":"<id>","target_id":"<id>","type":"related","position":0}'
clam --json content item relation list --input-json '{"content_id":"<id>"}'
clam --json content item relation remove --input-json '{"id":"<relation-id>"}'
```

### content item locale

```
clam --json content item locale add --input-json '{"content_id":"<id>","version_id":"<vid>","locale":"en-US"}'
clam --json content item locale list --input-json '{"content_id":"<id>"}'
clam --json content item locale get --input-json '{"content_id":"<id>","locale":"en-US"}'
clam --json content item locale remove --input-json '{"id":"<locale-entry-id>"}'
```

### content item asset

```
clam --json content item asset link --input-json '{"content_id":"<id>","asset_id":"<id>","role":"hero","position":0}'
clam --json content item asset list --input-json '{"content_id":"<id>"}'
clam --json content item asset unlink --input-json '{"id":"<content-asset-id>"}'
```

### content item tag

```
clam --json content item tag list --input-json '{"content_id":"<id>"}'
clam --json content item tag add --input-json '{"content_id":"<id>","tag":"featured"}'
clam --json content item tag remove --input-json '{"content_id":"<id>","tag":"featured"}'
```

### content type

```
clam --json content type add --input-json '{"name":"Blog Post","slug":"blog-post","body_format":"markdown","default_status":"draft","description":"...","icon":"...","schema_json":"{...}"}'
clam --json content type count
clam --json content type list --input-json '{"search":"blog","cursor":"...","limit":25}'
clam --json content type get --input-json '{"id":"<id-or-slug>"}'
clam --json content type update --input-json '{"id":"<id>","name":"Article","slug":"article","body_format":"html"}'
clam --json content type remove --input-json '{"id":"<id>"}'
clam --json content type restore --input-json '{"id":"<id>"}'
clam --json content type purge --input-json '{"id":"<id>"}'
```

### content asset

```
clam --json content asset add --input-json '{"filename":"photo.jpg","mime_type":"image/jpeg","url":"https://cdn.example.com/photo.jpg","size_bytes":12345,"alt_text":"...","width":1200,"height":800}'
clam --json content asset count
clam --json content asset list --input-json '{"search":"photo","mime_type":"image/jpeg","cursor":"...","limit":25}'
clam --json content asset get --input-json '{"id":"<asset-id>"}'
clam --json content asset update --input-json '{"id":"<id>","alt_text":"...","metadata_json":"{...}"}'
clam --json content asset remove --input-json '{"id":"<id>"}'
clam --json content asset restore --input-json '{"id":"<id>"}'
clam --json content asset purge --input-json '{"id":"<id>"}'
```

### content collection

```
clam --json content collection add --input-json '{"name":"My Series","type":"series","slug":"my-series","description":"...","parent_id":"..."}'
clam --json content collection count
clam --json content collection list --input-json '{"type":"series","search":"my","parent_id":"...","cursor":"...","limit":25}'
clam --json content collection get --input-json '{"id":"<id-or-slug>"}'
clam --json content collection update --input-json '{"id":"<id>","name":"Updated","slug":"updated","type":"series","description":"...","parent_id":"..."}'
clam --json content collection remove --input-json '{"id":"<id>"}'
clam --json content collection restore --input-json '{"id":"<id>"}'
clam --json content collection purge --input-json '{"id":"<id>"}'
clam --json content collection tree --input-json '{"root_id":"..."}'
```

### content collection item

```
clam --json content collection item add --input-json '{"collection_id":"<id>","content_id":"<id>","position":0}'
clam --json content collection item list --input-json '{"collection_id":"<id>"}'
clam --json content collection item remove --input-json '{"id":"<collection-item-id>"}'
```

---

## Input JSON Shapes

#### content item add
```typescript
interface ContentAddInput {
  type: string;                        // content type slug
  title: string;
  body: string;
  summary?: string | null;
  slug?: string | null;
  metadata_json?: string | null;
  author_id?: string;                  // defaults to active user
}
```

#### content item list
```typescript
interface ContentListInput {
  search?: string;
  type?: string;
  status?: string;
  author_id?: string;
  cursor?: string;
  limit?: number;
  expand?: string | string[];          // "versions,tags,notes"
}
```

#### content item publish
```typescript
interface ContentPublishInput {
  id: string;
  version_id: string;
}
```

#### content item version add
```typescript
interface ContentVersionAddInput {
  content_id: string;
  title: string;
  body: string;
  summary?: string | null;
  metadata_json?: string | null;
  change_note?: string | null;
}
```

#### content item comment add
```typescript
interface ContentCommentAddInput {
  content_id: string;
  body: string;
  author_id?: string;
  version_id?: string | null;
  parent_id?: string | null;
  anchor?: string | null;
}
```

#### content item channel add / update
```typescript
interface ContentChannelAddInput {
  content_id: string;
  channel: string;
  external_id?: string | null;
  status?: string;
  published_at?: string | null;
}
```

#### content item relation add
```typescript
interface ContentRelationAddInput {
  source_id: string;
  target_id: string;
  type: string;
  position?: number | null;
}
```

#### content item locale add
```typescript
interface ContentLocaleAddInput {
  content_id: string;
  version_id: string;
  locale: string;
}
```

#### content item asset link
```typescript
interface ContentAssetLinkInput {
  content_id: string;
  asset_id: string;
  role?: string;                       // e.g. "hero", "thumbnail"
  position?: number;
}
```

#### content type add
```typescript
interface ContentTypeAddInput {
  name: string;
  slug?: string;                       // auto-generated from name if omitted
  body_format: string;                 // e.g. "markdown", "html", "plain"
  default_status?: string;             // default: "draft"
  description?: string | null;
  icon?: string | null;
  schema_json?: string | null;
}
```

#### asset add
```typescript
interface AssetAddInput {
  filename: string;
  mime_type: string;
  url: string;
  size_bytes?: number | null;
  alt_text?: string | null;
  width?: number | null;
  height?: number | null;
  duration_seconds?: number | null;
  metadata_json?: string | null;
}
```

#### collection add
```typescript
interface CollectionAddInput {
  name: string;
  type: string;
  slug?: string;                       // auto-generated from name if omitted
  description?: string | null;
  parent_id?: string | null;
}
```

#### collection item add
```typescript
interface CollectionItemAddInput {
  collection_id: string;
  content_id: string;
  position?: number;                   // default: 0
}
```

---

See `../surfaces.md` to translate these operations to MCP, HTTP, or the typed client.
