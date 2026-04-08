# Content Domain Reference

Package: `@openclam/content`
Extension name: `content`
Dependencies: none

---

## Tables

### content_types

| Column         | Type | Constraints            |
|----------------|------|------------------------|
| id             | TEXT | PRIMARY KEY            |
| slug           | TEXT | NOT NULL, UNIQUE       |
| name           | TEXT | NOT NULL               |
| description    | TEXT | nullable               |
| icon           | TEXT | nullable               |
| body_format    | TEXT | NOT NULL (markdown, html, plain, etc.) |
| schema_json    | TEXT | nullable (JSON Schema for metadata validation) |
| default_status | TEXT | NOT NULL               |
| created_at     | TEXT | NOT NULL               |
| updated_at     | TEXT | NOT NULL               |
| deleted_at     | TEXT | nullable (soft delete)  |

### content_items

| Column               | Type | Constraints            |
|----------------------|------|------------------------|
| id                   | TEXT | PRIMARY KEY            |
| type                 | TEXT | NOT NULL (content type slug) |
| slug                 | TEXT | UNIQUE, nullable       |
| status               | TEXT | NOT NULL               |
| author_id            | TEXT | nullable               |
| person_id            | TEXT | nullable               |
| org_id               | TEXT | nullable               |
| published_version_id | TEXT | nullable               |
| scheduled_at         | TEXT | nullable               |
| published_at         | TEXT | nullable               |
| created_at           | TEXT | NOT NULL               |
| updated_at           | TEXT | NOT NULL               |
| deleted_at           | TEXT | nullable (soft delete)  |

### content_versions

| Column        | Type    | Constraints                                          |
|---------------|---------|------------------------------------------------------|
| id            | TEXT    | PRIMARY KEY                                          |
| content_id    | TEXT    | NOT NULL, FK -> content_items(id) ON DELETE CASCADE  |
| version       | INTEGER | NOT NULL                                             |
| title         | TEXT    | NOT NULL                                             |
| body          | TEXT    | nullable                                             |
| summary       | TEXT    | nullable                                             |
| metadata_json | TEXT    | nullable                                             |
| author_id     | TEXT    | nullable                                             |
| change_note   | TEXT    | nullable                                             |
| created_at    | TEXT    | NOT NULL                                             |

Immutable snapshots -- versions are never updated, only new ones created.

### content_comments

| Column     | Type | Constraints                                          |
|------------|------|------------------------------------------------------|
| id         | TEXT | PRIMARY KEY                                          |
| content_id | TEXT | NOT NULL, FK -> content_items(id) ON DELETE CASCADE  |
| version_id | TEXT | nullable, FK -> content_versions(id)                 |
| author_id  | TEXT | NOT NULL                                             |
| parent_id  | TEXT | nullable, FK -> content_comments(id) (threading)     |
| body       | TEXT | NOT NULL                                             |
| anchor     | TEXT | nullable (reference to specific content location)    |
| status     | TEXT | NOT NULL (open, resolved)                            |
| created_at | TEXT | NOT NULL                                             |
| updated_at | TEXT | NOT NULL                                             |

### content_notes

| Column     | Type | Constraints                                          |
|------------|------|------------------------------------------------------|
| id         | TEXT | PRIMARY KEY                                          |
| content_id | TEXT | NOT NULL, FK -> content_items(id) ON DELETE CASCADE  |
| title      | TEXT | nullable                                             |
| body       | TEXT | NOT NULL                                             |
| source     | TEXT | NOT NULL (e.g. cli, api, web)                        |
| created_at | TEXT | NOT NULL                                             |
| updated_at | TEXT | NOT NULL                                             |

### content_tags

| Column     | Type | Constraints                                          |
|------------|------|------------------------------------------------------|
| id         | TEXT | PRIMARY KEY                                          |
| content_id | TEXT | NOT NULL, FK -> content_items(id) ON DELETE CASCADE  |
| tag_id     | TEXT | NOT NULL (FK to global tags table)                   |
| created_at | TEXT | NOT NULL                                             |

Join table linking content items to global tags.

### content_assets

| Column     | Type    | Constraints                                          |
|------------|---------|------------------------------------------------------|
| id         | TEXT    | PRIMARY KEY                                          |
| content_id | TEXT    | NOT NULL, FK -> content_items(id) ON DELETE CASCADE  |
| asset_id   | TEXT    | NOT NULL, FK -> assets(id)                           |
| role       | TEXT    | NOT NULL (e.g. hero, thumbnail)                      |
| position   | INTEGER | NOT NULL                                             |
| created_at | TEXT    | NOT NULL                                             |

Join table linking content items to assets.

### content_relations

| Column    | Type    | Constraints                                          |
|-----------|---------|------------------------------------------------------|
| id        | TEXT    | PRIMARY KEY                                          |
| source_id | TEXT    | NOT NULL, FK -> content_items(id) ON DELETE CASCADE  |
| target_id | TEXT    | NOT NULL, FK -> content_items(id) ON DELETE CASCADE  |
| type      | TEXT    | NOT NULL (relation type label)                       |
| position  | INTEGER | nullable                                             |
| created_at| TEXT    | NOT NULL                                             |

### content_channels

| Column       | Type | Constraints                                          |
|--------------|------|------------------------------------------------------|
| id           | TEXT | PRIMARY KEY                                          |
| content_id   | TEXT | NOT NULL, FK -> content_items(id) ON DELETE CASCADE  |
| channel      | TEXT | NOT NULL                                             |
| external_id  | TEXT | nullable                                             |
| status       | TEXT | NOT NULL                                             |
| published_at | TEXT | nullable                                             |
| created_at   | TEXT | NOT NULL                                             |

### content_locales

| Column     | Type | Constraints                                          |
|------------|------|------------------------------------------------------|
| id         | TEXT | PRIMARY KEY                                          |
| content_id | TEXT | NOT NULL, FK -> content_items(id) ON DELETE CASCADE  |
| version_id | TEXT | NOT NULL, FK -> content_versions(id) ON DELETE CASCADE |
| locale     | TEXT | NOT NULL (e.g. en-US, fr-FR)                         |
| created_at | TEXT | NOT NULL                                             |

### assets

| Column           | Type    | Constraints            |
|------------------|---------|------------------------|
| id               | TEXT    | PRIMARY KEY            |
| filename         | TEXT    | NOT NULL               |
| mime_type        | TEXT    | NOT NULL               |
| size_bytes       | INTEGER | nullable               |
| url              | TEXT    | NOT NULL               |
| alt_text         | TEXT    | nullable               |
| width            | INTEGER | nullable               |
| height           | INTEGER | nullable               |
| duration_seconds | INTEGER | nullable               |
| metadata_json    | TEXT    | nullable               |
| uploaded_by      | TEXT    | nullable               |
| created_at       | TEXT    | NOT NULL               |
| deleted_at       | TEXT    | nullable (soft delete)  |

### collections

| Column      | Type | Constraints                                     |
|-------------|------|-------------------------------------------------|
| id          | TEXT | PRIMARY KEY                                     |
| name        | TEXT | NOT NULL                                        |
| slug        | TEXT | UNIQUE, nullable                                |
| type        | TEXT | NOT NULL                                        |
| description | TEXT | nullable                                        |
| parent_id   | TEXT | nullable, FK -> collections(id) (self-ref tree) |
| created_at  | TEXT | NOT NULL                                        |
| updated_at  | TEXT | NOT NULL                                        |
| deleted_at  | TEXT | nullable (soft delete)                          |

### collection_items

| Column        | Type    | Constraints                                          |
|---------------|---------|------------------------------------------------------|
| id            | TEXT    | PRIMARY KEY                                          |
| collection_id | TEXT    | NOT NULL, FK -> collections(id) ON DELETE CASCADE    |
| content_id    | TEXT    | NOT NULL, FK -> content_items(id) ON DELETE CASCADE  |
| position      | INTEGER | NOT NULL                                             |
| created_at    | TEXT    | NOT NULL                                             |

### project_content

| Column     | Type | Constraints                                          |
|------------|------|------------------------------------------------------|
| id         | TEXT | PRIMARY KEY                                          |
| project_id | TEXT | NOT NULL                                             |
| content_id | TEXT | NOT NULL, FK -> content_items(id) ON DELETE CASCADE  |
| added_at   | TEXT | NOT NULL                                             |
| added_by   | TEXT | nullable                                             |

Join table linking projects (from tasks extension) to content items.

---

## Service Operations

### contentTypeService (`services/content-type.ts`)

| Function  | Signature                                                        | Description                              |
|-----------|------------------------------------------------------------------|------------------------------------------|
| `add`     | `(db, tables, input) => ContentType`                             | Create content type. Input: `name`, `slug`, `body_format`, `description?`, `icon?`, `schema_json?`, `default_status`. |
| `list`    | `(db, tables, filters?) => ContentType[]`                        | List non-deleted types. Filter: `search`. |
| `get`     | `(db, tables, id) => ContentType`                                | Get by ID or slug. Excludes soft-deleted. |
| `update`  | `(db, tables, id, input) => ContentType`                         | Update `name`, `slug`, `body_format`, `description`, `icon`. |
| `remove`  | `(db, tables, id) => void`                                       | Soft-delete.                             |
| `restore` | `(db, tables, id) => void`                                       | Restore soft-deleted.                    |
| `purge`   | `(db, tables, id) => void`                                       | Hard-delete.                             |

### contentItemService (`services/content-item.ts`)

| Function  | Signature                                                        | Description                              |
|-----------|------------------------------------------------------------------|------------------------------------------|
| `add`     | `(db, tables, input, ctx?) => ContentItem`                       | Create content item + initial version. Input: `type`, `title`, `body`, `summary?`, `slug?`, `metadata_json?`, `author_id`. |
| `list`    | `(db, tables, filters?, pagination?, opts?) => ContentItem[]`    | List non-deleted items. Filters: `search`, `type`, `status`, `author_id`. Pagination: `limit`, `offset`. Opts: `expand` array (`versions`, `tags`, `notes`). |
| `get`     | `(db, tables, id, opts?) => ContentItem`                         | Get by ID or slug. Opts: `expand` array. |
| `update`  | `(db, tables, id, input, ctx?) => ContentItem`                   | Update `status`, `slug`, `title`, `metadata_json`. |
| `publish` | `(db, tables, id, versionId, ctx?) => ContentItem`               | Set `published_version_id`, status to published, set `published_at`. |
| `remove`  | `(db, tables, id, ctx?) => void`                                 | Soft-delete.                             |
| `restore` | `(db, tables, id, ctx?) => void`                                 | Restore soft-deleted.                    |
| `purge`   | `(db, tables, id, ctx?) => void`                                 | Hard-delete.                             |

### contentVersionService (`services/content-item.ts` or `content-version.ts`)

| Function | Signature                                                        | Description                              |
|----------|------------------------------------------------------------------|------------------------------------------|
| `add`    | `(db, tables, input, ctx?) => ContentVersion`                    | Create version. Input: `content_id`, `title`, `body`, `summary?`, `metadata_json?`, `created_by`. Auto-increments version number. |
| `list`   | `(db, tables, contentId) => ContentVersion[]`                    | List versions for a content item.        |
| `get`    | `(db, tables, versionId) => ContentVersion`                      | Get version by ID.                       |

### contentCommentService (`services/comment.ts`)

| Function    | Signature                                                        | Description                              |
|-------------|------------------------------------------------------------------|------------------------------------------|
| `add`       | `(db, tables, input, ctx?) => ContentComment`                    | Create comment. Input: `content_id`, `author_id`, `body`, `version_id?`, `parent_id?`, `anchor?`. Status defaults to `open`. |
| `list`      | `(db, tables, contentId, filters?) => ContentComment[]`          | List comments. Filters: `status`, `version_id`. |
| `get`       | `(db, tables, id) => ContentComment`                             | Get comment by ID.                       |
| `update`    | `(db, tables, id, input, ctx?) => ContentComment`                | Update `body`, `status`.                 |
| `resolve`   | `(db, tables, id, ctx?) => ContentComment`                       | Set status to `resolved`.                |
| `remove`    | `(db, tables, id, ctx?) => void`                                 | Hard-delete comment.                     |
| `getThread` | `(db, tables, parentId) => ContentComment[]`                     | Get replies to a comment.                |

### contentNoteService (`services/note.ts`)

| Function | Signature                                                        | Description                              |
|----------|------------------------------------------------------------------|------------------------------------------|
| `add`    | `(db, tables, input, ctx?) => ContentNote`                       | Create note. Input: `content_id`, `body`, `title?`, `source`. |
| `list`   | `(db, tables, contentId) => ContentNote[]`                       | List notes for content.                  |
| `get`    | `(db, tables, id) => ContentNote`                                | Get note by ID.                          |
| `update` | `(db, tables, id, input, ctx?) => ContentNote`                   | Update `title`, `body`.                  |
| `remove` | `(db, tables, id, ctx?) => void`                                 | Hard-delete note.                        |

### contentChannelService (`services/channel.ts`)

| Function | Signature                                                        | Description                              |
|----------|------------------------------------------------------------------|------------------------------------------|
| `add`    | `(db, tables, input, ctx?) => ContentChannel`                    | Create channel entry. Input: `content_id`, `channel`, `external_id?`, `status?`, `published_at?`. |
| `list`   | `(db, tables, contentId) => ContentChannel[]`                    | List channels for content.               |
| `get`    | `(db, tables, id) => ContentChannel`                             | Get channel entry by ID.                 |
| `update` | `(db, tables, id, input, ctx?) => ContentChannel`                | Update `channel`, `external_id`, `status`, `published_at`. |
| `remove` | `(db, tables, id, ctx?) => void`                                 | Hard-delete channel entry.               |

### contentRelationService (`services/relation.ts`)

| Function         | Signature                                                   | Description                              |
|------------------|-------------------------------------------------------------|------------------------------------------|
| `add`            | `(db, tables, input) => ContentRelation`                    | Create relation. Input: `source_id`, `target_id`, `type`, `position?`. |
| `listForContent` | `(db, tables, contentId) => ContentRelation[]`              | List all relations where content is source or target. |
| `remove`         | `(db, tables, id) => void`                                  | Hard-delete relation.                    |

### contentLocaleService (`services/locale.ts`)

| Function         | Signature                                                   | Description                              |
|------------------|-------------------------------------------------------------|------------------------------------------|
| `add`            | `(db, tables, input) => ContentLocale`                      | Create locale mapping. Input: `content_id`, `version_id`, `locale`. |
| `listForContent` | `(db, tables, contentId) => ContentLocale[]`                | List locale entries for content.         |
| `getForLocale`   | `(db, tables, contentId, locale) => ContentLocale`          | Get specific locale entry.               |
| `remove`         | `(db, tables, id) => void`                                  | Hard-delete locale entry.                |

### contentAssetService (`services/asset.ts` -- linking)

| Function         | Signature                                                   | Description                              |
|------------------|-------------------------------------------------------------|------------------------------------------|
| `link`           | `(db, tables, input) => ContentAsset`                       | Link asset to content. Input: `content_id`, `asset_id`, `role?`, `position?`. |
| `listForContent` | `(db, tables, contentId) => ContentAsset[]`                 | List assets linked to content.           |
| `unlink`         | `(db, tables, id) => void`                                  | Remove link (hard-delete join row).      |

### assetService (`services/asset.ts`)

| Function  | Signature                                                        | Description                              |
|-----------|------------------------------------------------------------------|------------------------------------------|
| `add`     | `(db, tables, input, ctx?) => Asset`                             | Create asset. Input: `filename`, `mime_type`, `url`, `size_bytes?`, `alt_text?`, `width?`, `height?`, `duration_seconds?`, `metadata_json?`, `uploaded_by`. |
| `list`    | `(db, tables, filters?, pagination?) => Asset[]`                 | List non-deleted assets. Filters: `search`, `mime_type`. Pagination: `limit`, `offset`. |
| `get`     | `(db, tables, id) => Asset`                                      | Get asset by ID.                         |
| `update`  | `(db, tables, id, input) => Asset`                               | Update `alt_text`, `metadata_json`.      |
| `remove`  | `(db, tables, id, ctx?) => void`                                 | Soft-delete.                             |
| `restore` | `(db, tables, id, ctx?) => void`                                 | Restore soft-deleted.                    |
| `purge`   | `(db, tables, id, ctx?) => void`                                 | Hard-delete.                             |

### collectionService (`services/collection.ts`)

| Function  | Signature                                                        | Description                              |
|-----------|------------------------------------------------------------------|------------------------------------------|
| `add`     | `(db, tables, input, ctx?) => Collection`                        | Create collection. Input: `name`, `type`, `slug?`, `description?`, `parent_id?`. |
| `list`    | `(db, tables, filters?, pagination?) => Collection[]`            | List non-deleted. Filters: `search`, `type`, `parent_id`. Pagination: `limit`, `offset`. |
| `get`     | `(db, tables, id) => Collection`                                 | Get by ID or slug.                       |
| `update`  | `(db, tables, id, input, ctx?) => Collection`                    | Update `name`, `slug`, `type`, `description`, `parent_id`. |
| `remove`  | `(db, tables, id, ctx?) => void`                                 | Soft-delete.                             |
| `restore` | `(db, tables, id, ctx?) => void`                                 | Restore soft-deleted.                    |
| `purge`   | `(db, tables, id, ctx?) => void`                                 | Hard-delete.                             |
| `getTree` | `(db, tables, rootId?) => CollectionTree`                        | Build hierarchical tree. Optional root filter. |

### collectionItemService (`services/collection-item.ts`)

| Function           | Signature                                                   | Description                              |
|--------------------|-------------------------------------------------------------|------------------------------------------|
| `add`              | `(db, tables, input) => CollectionItem`                     | Add content to collection. Input: `collection_id`, `content_id`, `position`. |
| `listForCollection`| `(db, tables, collectionId) => CollectionItem[]`            | List items in collection.                |
| `remove`           | `(db, tables, id) => void`                                  | Remove item from collection.             |

### contentTagService (`services/tag.ts`)

| Function       | Signature                                                   | Description                              |
|----------------|-------------------------------------------------------------|------------------------------------------|
| `getForContent`| `(db, tables, contentId) => ContentTag[]`                   | List tags for a content item.            |
| `apply`        | `(db, tables, contentId, tagSlug) => ContentTag`            | Add tag to content (by slug or label).   |
| `remove`       | `(db, tables, contentId, tagSlug) => void`                  | Remove tag from content.                 |

### projectLinkService (`services/project-link.ts`)

| Function                   | Signature                                                   | Description                              |
|----------------------------|-------------------------------------------------------------|------------------------------------------|
| `addContentToProject`      | `(db, tables, projectId, contentId) => ProjectContent`      | Link content to project.                 |
| `removeContentFromProject` | `(db, tables, projectId, contentId) => void`                | Unlink content from project.             |
| `listContentForProject`    | `(db, tables, projectId) => ProjectContent[]`               | List content linked to a project.        |
| `listProjectsForContent`   | `(db, tables, contentId) => ProjectContent[]`               | List projects linked to a content item.  |

---

## CLI Commands

All list commands return a paginated response:
```json
{ "data": [...], "nextCursor": "<string | null>", "hasMore": false }
```

### `content` -- Manage content items

```
# add (positional type + title or --input-json)
clam --json content item add <type> <title> --body "Body text"
clam --json content item add --input-json '{"type":"blog-post","title":"Hello World","body":"Body text"}'

# list (paginated)
clam --json content item list
clam --json content item list --type blog-post --status published --cursor <cursor> --limit 25
clam --json content item list --input-json '{"type":"blog-post","status":"published","cursor":"<cursor>","limit":25}'

# get (positional id or --input-json)
clam --json content item get <id>
clam --json content item get --input-json '{"id":"<content-id>"}'

# update (positional id or --input-json)
clam --json content item update <id> --status published
clam --json content item update --input-json '{"id":"<id>","status":"published"}'

# publish (positional id + version-id or --input-json)
clam --json content item publish <id> <version-id>
clam --json content item publish --input-json '{"id":"<id>","version_id":"<version-id>"}'

# remove / restore / purge (positional id or --input-json)
clam --json content item remove <id>
clam --json content item remove --input-json '{"id":"<id>"}'
clam --json content item restore <id>
clam --json content item restore --input-json '{"id":"<id>"}'
clam --json content item purge <id>
clam --json content item purge --input-json '{"id":"<id>"}'

# list projects for content (positional content-id or --input-json)
clam --json content item projects <content-id>
clam --json content item projects --input-json '{"content_id":"<id>"}'
```

### `content version`

```
# add (positional content-id or --input-json)
clam --json content item version add <content-id> --title "v2" --body "Updated body"
clam --json content item version add --input-json '{"content_id":"<id>","title":"v2","body":"Updated body"}'

# list (paginated; positional content-id or --input-json)
clam --json content item version list <content-id>
clam --json content item version list --input-json '{"content_id":"<id>","cursor":"<cursor>","limit":25}'

# get (positional version-id or --input-json; key is "id")
clam --json content item version get <version-id>
clam --json content item version get --input-json '{"id":"<version-id>"}'
```

### `content note`

```
# add (positional content-id or --input-json)
clam --json content item note add <content-id> --body "Note text"
clam --json content item note add --input-json '{"content_id":"<id>","body":"Note text"}'

# list (paginated; positional content-id or --input-json)
clam --json content item note list <content-id>
clam --json content item note list --input-json '{"content_id":"<id>","cursor":"<cursor>","limit":25}'

# get (positional id or --input-json)
clam --json content item note get <id>
clam --json content item note get --input-json '{"id":"<note-id>"}'

# update (positional id or --input-json)
clam --json content item note update <id> --body "Updated"
clam --json content item note update --input-json '{"id":"<note-id>","body":"Updated"}'

# remove (positional id or --input-json)
clam --json content item note remove <id>
clam --json content item note remove --input-json '{"id":"<note-id>"}'
```

### `content comment`

```
# add (positional content-id or --input-json)
clam --json content item comment add <content-id> --body "Comment"
clam --json content item comment add --input-json '{"content_id":"<id>","body":"Comment","version_id":"<vid>","parent_id":"<pid>","anchor":"section-2"}'

# list (paginated; positional content-id or --input-json)
clam --json content item comment list <content-id>
clam --json content item comment list --input-json '{"content_id":"<id>","status":"open","version_id":"<vid>","cursor":"<cursor>","limit":25}'

# get (positional id or --input-json)
clam --json content item comment get <id>
clam --json content item comment get --input-json '{"id":"<comment-id>"}'

# update (positional id or --input-json)
clam --json content item comment update <id> --body "Updated"
clam --json content item comment update --input-json '{"id":"<id>","body":"Updated","status":"resolved"}'

# resolve (positional id or --input-json)
clam --json content item comment resolve <id>
clam --json content item comment resolve --input-json '{"id":"<id>"}'

# remove (positional id or --input-json)
clam --json content item comment remove <id>
clam --json content item comment remove --input-json '{"id":"<id>"}'

# thread (positional parent-id or --input-json; key is "parent_id")
clam --json content item comment thread <parent-id>
clam --json content item comment thread --input-json '{"parent_id":"<parent-id>"}'
```

### `content channel`

```
# add (positional content-id or --input-json)
clam --json content item channel add <content-id> --channel web --status published
clam --json content item channel add --input-json '{"content_id":"<id>","channel":"web","status":"published"}'

# list (paginated; positional content-id or --input-json)
clam --json content item channel list <content-id>
clam --json content item channel list --input-json '{"content_id":"<id>","cursor":"<cursor>","limit":25}'

# get (positional id or --input-json)
clam --json content item channel get <id>
clam --json content item channel get --input-json '{"id":"<channel-entry-id>"}'

# update (positional id or --input-json)
clam --json content item channel update <id> --status published
clam --json content item channel update --input-json '{"id":"<id>","status":"published","published_at":"2026-03-01T00:00:00Z"}'

# remove (positional id or --input-json)
clam --json content item channel remove <id>
clam --json content item channel remove --input-json '{"id":"<id>"}'
```

### `content relation`

```
# add (positional source-id + target-id or --input-json)
clam --json content item relation add <source-id> <target-id> --type "related"
clam --json content item relation add --input-json '{"source_id":"<id>","target_id":"<id>","type":"related"}'

# list (positional content-id or --input-json)
clam --json content item relation list <content-id>
clam --json content item relation list --input-json '{"content_id":"<id>"}'

# remove (positional id or --input-json)
clam --json content item relation remove <id>
clam --json content item relation remove --input-json '{"id":"<relation-id>"}'
```

### `content locale`

```
# add (positional content-id or --input-json; version-id + locale required)
clam --json content item locale add <content-id> --version-id <vid> --locale en-US
clam --json content item locale add --input-json '{"content_id":"<id>","version_id":"<vid>","locale":"en-US"}'

# list (positional content-id or --input-json)
clam --json content item locale list <content-id>
clam --json content item locale list --input-json '{"content_id":"<id>"}'

# get (positional content-id + locale or --input-json)
clam --json content item locale get <content-id> en-US
clam --json content item locale get --input-json '{"content_id":"<id>","locale":"en-US"}'

# remove (positional id or --input-json)
clam --json content item locale remove <id>
clam --json content item locale remove --input-json '{"id":"<locale-entry-id>"}'
```

### `content asset`

```
# link (positional content-id + asset-id or --input-json)
clam --json content asset link <content-id> <asset-id> --role hero
clam --json content asset link --input-json '{"content_id":"<id>","asset_id":"<id>","role":"hero"}'

# list (positional content-id or --input-json)
clam --json content asset list <content-id>
clam --json content asset list --input-json '{"content_id":"<id>"}'

# unlink (positional link-id or --input-json)
clam --json content asset unlink <id>
clam --json content asset unlink --input-json '{"id":"<content-asset-id>"}'
```

### `content tag`

```
# list (positional content-id or --input-json; key is "content_id")
clam --json content item tag list <content-id>
clam --json content item tag list --input-json '{"content_id":"<id>"}'

# add (positional content-id or --input-json)
clam --json content item tag add <content-id> --tag "featured"
clam --json content item tag add --input-json '{"content_id":"<id>","tag":"featured"}'

# remove (positional content-id or --input-json)
clam --json content item tag remove <content-id> --tag "featured"
clam --json content item tag remove --input-json '{"content_id":"<id>","tag":"featured"}'
```

---

### `content-type` -- Manage content types

```
# add (positional name or --input-json)
clam --json content type add "Blog Post" --body-format markdown
clam --json content type add --input-json '{"name":"Blog Post","body_format":"markdown","default_status":"draft"}'

# list (paginated)
clam --json content type list
clam --json content type list --search "blog" --cursor <cursor> --limit 25
clam --json content type list --input-json '{"search":"blog","cursor":"<cursor>","limit":25}'

# get (positional id or --input-json)
clam --json content type get <id>
clam --json content type get --input-json '{"id":"<content-type-id>"}'

# update (positional id or --input-json)
clam --json content type update <id> --name "Article"
clam --json content type update --input-json '{"id":"<id>","name":"Article"}'

# remove / restore / purge (positional id or --input-json)
clam --json content type remove <id>
clam --json content type remove --input-json '{"id":"<id>"}'
clam --json content type restore <id>
clam --json content type restore --input-json '{"id":"<id>"}'
clam --json content type purge <id>
clam --json content type purge --input-json '{"id":"<id>"}'
```

---

### `asset` -- Manage assets

```
# add (positional filename or --input-json)
clam --json content asset add "photo.jpg" --mime-type image/jpeg --url https://cdn.example.com/photo.jpg
clam --json content asset add --input-json '{"filename":"photo.jpg","mime_type":"image/jpeg","url":"https://cdn.example.com/photo.jpg"}'

# list (paginated)
clam --json content asset list
clam --json content asset list --search "photo" --mime-type image/jpeg --cursor <cursor> --limit 25
clam --json content asset list --input-json '{"search":"photo","mime_type":"image/jpeg","cursor":"<cursor>","limit":25}'

# get (positional id or --input-json)
clam --json content asset get <id>
clam --json content asset get --input-json '{"id":"<asset-id>"}'

# update (positional id or --input-json)
clam --json content asset update <id> --alt-text "A photo"
clam --json content asset update --input-json '{"id":"<id>","alt_text":"A photo"}'

# remove / restore / purge (positional id or --input-json)
clam --json content asset remove <id>
clam --json content asset remove --input-json '{"id":"<id>"}'
clam --json content asset restore <id>
clam --json content asset restore --input-json '{"id":"<id>"}'
clam --json content asset purge <id>
clam --json content asset purge --input-json '{"id":"<id>"}'
```

---

### `collection` -- Manage collections

```
# add (positional name or --input-json)
clam --json content collection add "My Series" --type series
clam --json content collection add --input-json '{"name":"My Series","type":"series","description":"A series of articles"}'

# list (paginated)
clam --json content collection list
clam --json content collection list --type series --cursor <cursor> --limit 25
clam --json content collection list --input-json '{"type":"series","search":"my","cursor":"<cursor>","limit":25}'

# get (positional id or --input-json)
clam --json content collection get <id>
clam --json content collection get --input-json '{"id":"<collection-id>"}'

# update (positional id or --input-json)
clam --json content collection update <id> --name "Updated Series"
clam --json content collection update --input-json '{"id":"<id>","name":"Updated Series"}'

# remove / restore / purge (positional id or --input-json)
clam --json content collection remove <id>
clam --json content collection remove --input-json '{"id":"<id>"}'
clam --json content collection restore <id>
clam --json content collection restore --input-json '{"id":"<id>"}'
clam --json content collection purge <id>
clam --json content collection purge --input-json '{"id":"<id>"}'

# tree (no --input-json; optional --root flag)
clam --json content collection tree
clam --json content collection tree --root <root-id>
```

### `collection item`

```
# add (positional collection-id + content-id or --input-json)
clam --json content collection item add <collection-id> <content-id>
clam --json content collection item add --input-json '{"collection_id":"<id>","content_id":"<id>","position":0}'

# list (positional collection-id or --input-json)
clam --json content collection item list <collection-id>
clam --json content collection item list --input-json '{"collection_id":"<id>"}'

# remove (positional item-id or --input-json; key is "id")
clam --json content collection item remove <item-id>
clam --json content collection item remove --input-json '{"id":"<collection-item-id>"}'
```

---

## Global Flags

- `--json` -- Produce compact JSON output (must be placed immediately after `clam`, before the subcommand). Required for machine-readable output.
- `--input-json <json>` -- Pass structured JSON input to any command. Overrides positional args and individual flags.

---

## Input JSON Shapes

Every command accepts `--input-json <json>`. Field resolution order: `input.<field> ?? positionalArg ?? opts.<flag>`.

### content add

```typescript
interface ContentAddInput {
  type: string;                      // required (also accepted as first positional); content type slug
  title: string;                     // required (also accepted as second positional)
  body: string;                      // required (also accepted via --body flag)
  summary?: string | null;
  slug?: string | null;
  metadata_json?: string | null;
  author_id?: string;                // UUID; defaults to active user
}
```

### content list

```typescript
interface ContentListInput {
  search?: string;
  type?: string;                     // content type slug
  status?: string;
  author_id?: string;
  cursor?: string;
  limit?: number;
  expand?: string | string[];        // comma-separated or array: "versions,tags,notes"
}
// Returns: { data: ContentItem[], nextCursor: string | null, hasMore: boolean }
```

### content get

```typescript
interface ContentGetInput {
  id: string;                        // required (also accepted as positional); UUID or slug
  expand?: string | string[];
}
```

### content update

```typescript
interface ContentUpdateInput {
  id: string;                        // required (also accepted as positional)
  status?: string;
  slug?: string;
  title?: string;
  metadata_json?: string | null;
}
```

### content publish

```typescript
interface ContentPublishInput {
  id: string;                        // required (also accepted as first positional)
  version_id: string;                // required (also accepted as second positional)
}
```

### content remove / restore / purge

```typescript
interface ContentIdInput {
  id: string;                        // required (also accepted as positional)
}
// remove returns: { ok: true, deleted: string }
// restore returns: { ok: true, restored: string }
// purge returns: { ok: true, purged: string }
```

### content projects

```typescript
interface ContentProjectsInput {
  content_id: string;                // required (also accepted as positional)
}
```

### content version add

```typescript
interface ContentVersionAddInput {
  content_id: string;                // required (also accepted as positional)
  title: string;                     // required
  body: string;                      // required
  summary?: string | null;
  metadata_json?: string | null;
}
```

### content version list

```typescript
interface ContentVersionListInput {
  content_id: string;                // required (also accepted as positional)
  cursor?: string;
  limit?: number;
}
// Returns: { data: ContentVersion[], nextCursor: string | null, hasMore: boolean }
```

### content version get

```typescript
interface ContentVersionGetInput {
  id: string;                        // required (also accepted as positional); version UUID
}
```

### content note add

```typescript
interface ContentNoteAddInput {
  content_id: string;                // required (also accepted as positional)
  body: string;                      // required
  title?: string | null;
  source?: string;                   // default: "cli"
}
```

### content note list

```typescript
interface ContentNoteListInput {
  content_id: string;                // required (also accepted as positional)
  cursor?: string;
  limit?: number;
}
// Returns: { data: ContentNote[], nextCursor: string | null, hasMore: boolean }
```

### content note get / remove

```typescript
interface ContentNoteIdInput {
  id: string;                        // required (also accepted as positional)
}
```

### content note update

```typescript
interface ContentNoteUpdateInput {
  id: string;                        // required (also accepted as positional)
  title?: string | null;
  body?: string;
}
```

### content comment add

```typescript
interface ContentCommentAddInput {
  content_id: string;                // required (also accepted as positional)
  body: string;                      // required
  author_id?: string;                // UUID; defaults to active user
  version_id?: string | null;        // UUID; pin comment to a version
  parent_id?: string | null;         // UUID; for threading
  anchor?: string | null;            // reference to a specific content location
}
```

### content comment list

```typescript
interface ContentCommentListInput {
  content_id: string;                // required (also accepted as positional)
  status?: "open" | "resolved";
  version_id?: string;
  cursor?: string;
  limit?: number;
}
// Returns: { data: ContentComment[], nextCursor: string | null, hasMore: boolean }
```

### content comment get / resolve / remove

```typescript
interface ContentCommentIdInput {
  id: string;                        // required (also accepted as positional)
}
```

### content comment update

```typescript
interface ContentCommentUpdateInput {
  id: string;                        // required (also accepted as positional)
  body?: string;
  status?: "open" | "resolved";
}
```

### content comment thread

```typescript
interface ContentCommentThreadInput {
  parent_id: string;                 // required (also accepted as positional)
}
```

### content channel add

```typescript
interface ContentChannelAddInput {
  content_id: string;                // required (also accepted as positional)
  channel: string;                   // required; channel name (e.g. "web", "email", "rss")
  external_id?: string | null;
  status?: string;
  published_at?: string | null;      // ISO 8601
}
```

### content channel list

```typescript
interface ContentChannelListInput {
  content_id: string;                // required (also accepted as positional)
  cursor?: string;
  limit?: number;
}
// Returns: { data: ContentChannel[], nextCursor: string | null, hasMore: boolean }
```

### content channel get / remove

```typescript
interface ContentChannelIdInput {
  id: string;                        // required (also accepted as positional)
}
```

### content channel update

```typescript
interface ContentChannelUpdateInput {
  id: string;                        // required (also accepted as positional)
  channel?: string;
  external_id?: string | null;
  status?: string;
  published_at?: string | null;
}
```

### content relation add

```typescript
interface ContentRelationAddInput {
  source_id: string;                 // required (also accepted as first positional)
  target_id: string;                 // required (also accepted as second positional)
  type: string;                      // required; relation type label (e.g. "related", "parent")
  position?: number | null;
}
```

### content relation list

```typescript
interface ContentRelationListInput {
  content_id: string;                // required (also accepted as positional)
}
// Returns: ContentRelation[]
```

### content relation remove

```typescript
interface ContentRelationRemoveInput {
  id: string;                        // required (also accepted as positional)
}
```

### content locale add

```typescript
interface ContentLocaleAddInput {
  content_id: string;                // required (also accepted as positional)
  version_id: string;                // required
  locale: string;                    // required; e.g. "en-US", "fr-FR"
}
```

### content locale list

```typescript
interface ContentLocaleListInput {
  content_id: string;                // required (also accepted as positional)
}
```

### content locale get

```typescript
interface ContentLocaleGetInput {
  content_id: string;                // required (also accepted as first positional)
  locale: string;                    // required (also accepted as second positional)
}
```

### content locale remove

```typescript
interface ContentLocaleRemoveInput {
  id: string;                        // required (also accepted as positional); locale entry UUID
}
```

### content asset link

```typescript
interface ContentAssetLinkInput {
  content_id: string;                // required (also accepted as first positional)
  asset_id: string;                  // required (also accepted as second positional)
  role?: string;                     // e.g. "hero", "thumbnail"
  position?: number;
}
```

### content asset list

```typescript
interface ContentAssetListInput {
  content_id: string;                // required (also accepted as positional)
}
```

### content asset unlink

```typescript
interface ContentAssetUnlinkInput {
  id: string;                        // required (also accepted as positional); content_assets row UUID
}
```

### content tag list / add / remove

```typescript
interface ContentTagInput {
  content_id: string;                // required (also accepted as positional)
  tag?: string;                      // required for add/remove; tag slug or label
}
```

### content-type add

```typescript
interface ContentTypeAddInput {
  name: string;                      // required (also accepted as positional)
  slug?: string;                     // auto-generated from name if omitted
  body_format: string;               // required; e.g. "markdown", "html", "plain"
  description?: string | null;
  icon?: string | null;
  schema_json?: string | null;       // JSON Schema for metadata validation
  default_status?: string;           // default: "draft"
}
```

### content-type list

```typescript
interface ContentTypeListInput {
  search?: string;
  cursor?: string;
  limit?: number;
}
// Returns: { data: ContentType[], nextCursor: string | null, hasMore: boolean }
```

### content-type get

```typescript
interface ContentTypeGetInput {
  id: string;                        // required (also accepted as positional); UUID or slug
}
```

### content-type update

```typescript
interface ContentTypeUpdateInput {
  id: string;                        // required (also accepted as positional)
  name?: string;
  slug?: string;
  description?: string | null;
  icon?: string | null;
  body_format?: string;
}
```

### content-type remove / restore / purge

```typescript
interface ContentTypeIdInput {
  id: string;                        // required (also accepted as positional)
}
```

### asset add

```typescript
interface AssetAddInput {
  filename: string;                  // required (also accepted as positional)
  mime_type: string;                 // required
  url: string;                       // required
  size_bytes?: number | null;
  alt_text?: string | null;
  width?: number | null;
  height?: number | null;
  duration_seconds?: number | null;
  metadata_json?: string | null;
}
```

### asset list

```typescript
interface AssetListInput {
  search?: string;
  mime_type?: string;
  cursor?: string;
  limit?: number;
}
// Returns: { data: Asset[], nextCursor: string | null, hasMore: boolean }
```

### asset get / remove / restore / purge

```typescript
interface AssetIdInput {
  id: string;                        // required (also accepted as positional)
}
```

### asset update

```typescript
interface AssetUpdateInput {
  id: string;                        // required (also accepted as positional)
  alt_text?: string | null;
  metadata_json?: string | null;
}
```

### collection add

```typescript
interface CollectionAddInput {
  name: string;                      // required (also accepted as positional)
  type: string;                      // required
  slug?: string;                     // auto-generated from name if omitted
  description?: string | null;
  parent_id?: string | null;         // UUID; for nested collections
}
```

### collection list

```typescript
interface CollectionListInput {
  search?: string;
  type?: string;
  parent_id?: string | null;
  cursor?: string;
  limit?: number;
}
// Returns: { data: Collection[], nextCursor: string | null, hasMore: boolean }
```

### collection get / remove / restore / purge

```typescript
interface CollectionIdInput {
  id: string;                        // required (also accepted as positional)
}
```

### collection update

```typescript
interface CollectionUpdateInput {
  id: string;                        // required (also accepted as positional)
  name?: string;
  slug?: string;
  type?: string;
  description?: string | null;
  parent_id?: string | null;
}
```

### collection item add

```typescript
interface CollectionItemAddInput {
  collection_id: string;             // required (also accepted as first positional)
  content_id: string;                // required (also accepted as second positional)
  position?: number;                 // default: 0
}
```

### collection item list

```typescript
interface CollectionItemListInput {
  collection_id: string;             // required (also accepted as positional)
}
// Returns: CollectionItem[] (ordered by position)
```

### collection item remove

```typescript
interface CollectionItemRemoveInput {
  id: string;                        // required (also accepted as positional); collection_items row UUID
}
```

---

## API Routes

### Content Types

| Method | Path                              | Description              |
|--------|-----------------------------------|--------------------------|
| GET    | `/api/content-types`              | List content types. Query: `search`. |
| POST   | `/api/content-types`              | Create content type. Returns 201. |
| GET    | `/api/content-types/:id`          | Get content type.        |
| PUT    | `/api/content-types/:id`          | Update content type.     |
| DELETE | `/api/content-types/:id`          | Soft-delete content type. |
| PUT    | `/api/content-types/:id/restore`  | Restore content type.    |
| DELETE | `/api/content-types/:id/purge`    | Hard-delete content type. |

### Content Items

| Method | Path                          | Description              |
|--------|-------------------------------|--------------------------|
| GET    | `/api/content`                | List items. Query: `search`, `type`, `status`, `author_id`. |
| POST   | `/api/content`                | Create item. Returns 201. |
| POST   | `/api/content/import`         | Import/create item (alias). Returns 201. |
| GET    | `/api/content/:id`            | Get item. Query: `expand` (comma-separated). |
| PUT    | `/api/content/:id`            | Update item.             |
| PUT    | `/api/content/:id/publish`    | Publish item. Body: `{ versionId }`. |
| DELETE | `/api/content/:id`            | Soft-delete item.        |
| PUT    | `/api/content/:id/restore`    | Restore item.            |
| DELETE | `/api/content/:id/purge`      | Hard-delete item.        |

### Content Versions

| Method | Path                          | Description              |
|--------|-------------------------------|--------------------------|
| GET    | `/api/content/:id/versions`   | List versions for content. |
| POST   | `/api/content/:id/versions`   | Create version. Returns 201. |
| GET    | `/api/versions/:id`           | Get version by ID.       |

### Content Assets (linking)

| Method | Path                          | Description              |
|--------|-------------------------------|--------------------------|
| GET    | `/api/content/:id/assets`     | List assets linked to content. |
| POST   | `/api/content/:id/assets`     | Link asset. Returns 201. |
| DELETE | `/api/content-assets/:id`     | Unlink asset.            |

### Content Comments

| Method | Path                          | Description              |
|--------|-------------------------------|--------------------------|
| GET    | `/api/content/:id/comments`   | List comments for content. |
| POST   | `/api/content/:id/comments`   | Create comment. Returns 201. |
| PUT    | `/api/comments/:id`           | Update comment.          |
| PUT    | `/api/comments/:id/resolve`   | Resolve comment.         |
| DELETE | `/api/comments/:id`           | Delete comment.          |

### Content Notes

| Method | Path                          | Description              |
|--------|-------------------------------|--------------------------|
| GET    | `/api/content/:id/notes`      | List notes for content.  |
| POST   | `/api/content/:id/notes`      | Create note. Returns 201. |
| GET    | `/api/content-notes/:id`      | Get note by ID.          |
| PUT    | `/api/content-notes/:id`      | Update note.             |
| DELETE | `/api/content-notes/:id`      | Delete note.             |

### Content Tags

| Method | Path                          | Description              |
|--------|-------------------------------|--------------------------|
| GET    | `/api/content/:id/tags`       | List tags for content.   |
| POST   | `/api/content/:id/tags`       | Add tag. Body: `{ tag }`. Returns 201. |
| DELETE | `/api/content/:id/tags`       | Remove tag. Body: `{ tag }`. |

### Content Relations

| Method | Path                          | Description              |
|--------|-------------------------------|--------------------------|
| GET    | `/api/content/:id/relations`  | List relations for content. |
| POST   | `/api/content/:id/relations`  | Create relation. Returns 201. |
| DELETE | `/api/content-relations/:id`  | Delete relation.         |

### Content Channels

| Method | Path                          | Description              |
|--------|-------------------------------|--------------------------|
| GET    | `/api/content/:id/channels`   | List channels for content. |
| POST   | `/api/content/:id/channels`   | Create channel entry. Returns 201. |
| GET    | `/api/content-channels/:id`   | Get channel entry.       |
| PUT    | `/api/content-channels/:id`   | Update channel entry.    |
| DELETE | `/api/content-channels/:id`   | Delete channel entry.    |

### Content Locales

| Method | Path                               | Description              |
|--------|-------------------------------------|--------------------------|
| GET    | `/api/content/:id/locales`          | List locales for content. |
| GET    | `/api/content/:id/locales/:locale`  | Get specific locale version. |
| POST   | `/api/content/:id/locales`          | Create locale entry. Returns 201. |

### Assets

| Method | Path                        | Description              |
|--------|-----------------------------|--------------------------|
| GET    | `/api/assets`               | List assets. Query: `search`, `mime_type`. |
| POST   | `/api/assets`               | Create asset. Returns 201. |
| GET    | `/api/assets/:id`           | Get asset.               |
| PUT    | `/api/assets/:id`           | Update asset.            |
| DELETE | `/api/assets/:id`           | Soft-delete asset.       |
| PUT    | `/api/assets/:id/restore`   | Restore asset.           |
| DELETE | `/api/assets/:id/purge`     | Hard-delete asset.       |

### Collections

| Method | Path                            | Description              |
|--------|---------------------------------|--------------------------|
| GET    | `/api/collections/tree`         | Get collection tree. Query: `rootId`. |
| GET    | `/api/collections`              | List collections. Query: `search`, `type`. |
| POST   | `/api/collections`              | Create collection. Returns 201. |
| GET    | `/api/collections/:id`          | Get collection.          |
| PUT    | `/api/collections/:id`          | Update collection.       |
| DELETE | `/api/collections/:id`          | Soft-delete collection.  |
| PUT    | `/api/collections/:id/restore`  | Restore collection.      |
| DELETE | `/api/collections/:id/purge`    | Hard-delete collection.  |

### Collection Items

| Method | Path                            | Description              |
|--------|---------------------------------|--------------------------|
| GET    | `/api/collections/:id/items`    | List items in collection. |
| POST   | `/api/collections/:id/items`    | Add item. Returns 201.   |
| DELETE | `/api/collection-items/:id`     | Remove item.             |

### Project-Content Links

| Method | Path                                    | Description              |
|--------|-----------------------------------------|--------------------------|
| POST   | `/api/projects/:id/content`             | Link content to project. Body: `{ contentId }`. Returns 201. |
| DELETE | `/api/projects/:id/content/:contentId`  | Unlink content from project. |
| GET    | `/api/projects/:id/content`             | List content for project. |
| GET    | `/api/content/:id/projects`             | List projects for content. |

---

## Permissions

| Resource        | Actions                          |
|-----------------|----------------------------------|
| content_type    | create, read, update, delete     |
| content_item    | create, read, update, delete, publish |
| asset           | create, read, delete             |
| collection      | create, read, update, delete     |
| content_comment | create, read, update, delete     |
