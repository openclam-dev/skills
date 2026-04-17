---
name: operate-wiki
description: >
  Wiki in OpenClam — hierarchical pages with parent_id, full version
  history, `[[slug]]` wikilinks with auto-maintained backlinks, threaded
  comments, and file attachments. The `clam wiki` CLI.
metadata:
  author: openclam
  version: 0.1.0
---

# Wiki

Package: `@openclam/wiki` (daemon extension).

The wiki is a tree of pages where each page has a mutable current version
(`wiki_page_content`) and an append-only history (`wiki_page_versions`).
Wikilinks of the form `[[page-slug]]` inside a page body are parsed on
every save and materialised into the `wiki_page_links` table so backlink
queries are O(1).

## Tables

### wiki_pages

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| parent_id | TEXT | Self-reference (nullable for top-level) |
| title | TEXT NOT NULL | |
| slug | TEXT NOT NULL UNIQUE | Used by `[[wikilinks]]` |
| icon | TEXT | |
| cover_image | TEXT | |
| position | INTEGER NOT NULL | Default 0 — ordering within parent |
| is_pinned | BOOLEAN NOT NULL | Default false |
| is_archived | BOOLEAN NOT NULL | Default false |
| created_by | TEXT | |
| created_at, updated_at, deleted_at | TEXT | Soft delete via `deleted_at` |

### wiki_page_content

Current version of a page (the one served for `page get`).

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| page_id | TEXT NOT NULL | FK → wiki_pages (CASCADE) |
| title | TEXT NOT NULL | Mirrors the page title at the time of the current edit |
| body | TEXT NOT NULL | Markdown body |

### wiki_page_versions

Append-only history. Version 1 is created on `page add` with change
summary `"Initial version"`.

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| page_id | TEXT NOT NULL | FK → wiki_pages (CASCADE) |
| title, body | TEXT NOT NULL | Snapshot |
| version_number | INTEGER NOT NULL | Monotonic per page |
| edited_by | TEXT | |
| change_summary | TEXT | |
| created_at | TEXT NOT NULL | |

### wiki_page_links

Materialised forward links, refreshed on every page save. Only links whose
target slug resolves to a non-deleted page are stored.

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| source_page_id | TEXT NOT NULL | FK → wiki_pages (CASCADE) |
| target_page_id | TEXT NOT NULL | FK → wiki_pages (CASCADE) |
| created_at | TEXT NOT NULL | |

### wiki_page_tags

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| page_id | TEXT NOT NULL | FK → wiki_pages (CASCADE) |
| tag_id | TEXT NOT NULL | |
| created_at | TEXT NOT NULL | |

### wiki_page_comments

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| page_id | TEXT NOT NULL | FK → wiki_pages (CASCADE) |
| author_id | TEXT | |
| parent_id | TEXT | For reply threads |
| body | TEXT NOT NULL | |
| created_at, updated_at, deleted_at | TEXT | |

### wiki_page_attachments

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| page_id | TEXT NOT NULL | FK → wiki_pages (CASCADE) |
| file_name | TEXT NOT NULL | |
| file_path | TEXT NOT NULL | |
| file_size | INTEGER | |
| mime_type | TEXT | |
| uploaded_by | TEXT | |
| created_at | TEXT NOT NULL | |

## Service Operations

All services live in `@openclam/wiki/services`.

### pageService

```ts
add(db, tables, input, ctx?): Promise<Page>                              // slug auto-generated from title if omitted; creates initial content + version 1; parses wikilinks
list(db, tables, filters?, pagination?, options?): Promise<Paginated<Page>>
count(db, tables, filters?): Promise<{ count: number }>
get(db, tables, id): Promise<Page>                                       // excludes soft-deleted
getBySlug(db, tables, slug): Promise<Page>
getWithContent(db, tables, id): Promise<Page & { content: PageContent }>
update(db, tables, id, input, ctx?): Promise<Page>                       // on body change: writes new version, updates content, re-syncs links
remove(db, tables, id, ctx?): Promise<void>                              // soft delete
restore(db, tables, id, ctx?): Promise<Page>
purge(db, tables, id, ctx?): Promise<void>                               // hard delete; cascades
```

`filters` supports `search` (title LIKE), `parent_id` (pass `null` to
filter top-level pages), `is_pinned`, `is_archived`. Pagination is
`{ cursor, limit, sort, order, all }`; sort defaults to `created_at` and
accepts any real column on `wikiPages` (typically `title`, `position`,
`updated_at`).

### versionService

```ts
list(db, tables, pageId, pagination?): Promise<Paginated<PageVersion>>   // newest first
get(db, tables, id): Promise<PageVersion>
getByNumber(db, tables, pageId, versionNumber): Promise<PageVersion>
```

### linkService

```ts
listBacklinks(db, tables, pageId): Promise<Page[]>                       // pages whose body links to pageId
listForwardLinks(db, tables, pageId): Promise<Page[]>                    // pages this page links to
```

### commentService

```ts
add, list(pageId, pagination?), get, update, remove, restore
```

### attachmentService

```ts
add, list(pageId, pagination?), get, remove
```

## CLI Commands

### clam wiki

Namespaces: `page`, `version`, `comment`, `attachment`.

#### page

```
clam --json wiki page add <title>   --input-json '{"body":"Hello [[other-page]]","parent_id":"<uuid>","slug":"my-page","icon":"📘"}'
clam --json wiki page list          --input-json '{"search":"...","parent_id":"<uuid>","is_pinned":1,"cursor":"...","limit":25,"sort":"updated_at","order":"desc","all":false,"fields":"id,title,slug,position"}'
clam --json wiki page count         --input-json '{"search":"...","parent_id":"<uuid>"}'
clam --json wiki page get <id>
clam --json wiki page update <id>   --input-json '{"title":"...","body":"...","slug":"...","change_summary":"Reworded the intro"}'
clam --json wiki page remove <id>                     # soft delete
clam --json wiki page restore <id>
```

Updating `body` automatically:

1. Appends a new row to `wiki_page_versions` with the next version number
   and the supplied `change_summary`.
2. Overwrites `wiki_page_content` with the new title/body.
3. Re-parses `[[wikilinks]]` and reconciles `wiki_page_links`.

Pass `--schema` on `list` or `get` to emit the JSON Schema.

#### version

```
clam --json wiki version list <pageId>   --input-json '{"cursor":"...","limit":25,"fields":"id,version_number,change_summary,edited_by,created_at"}'
clam --json wiki version get <id>
```

Use this to diff history or roll back by feeding an older version's body
back through `wiki page update`.

#### comment

```
clam --json wiki comment add <pageId>   --input-json '{"body":"...","parent_id":"<commentId>"}'
clam --json wiki comment list <pageId>  --input-json '{"cursor":"...","limit":25}'
clam --json wiki comment update <id>    --input-json '{"body":"..."}'
clam --json wiki comment remove <id>
```

#### attachment

```
clam --json wiki attachment add <pageId>    --input-json '{"file_name":"spec.pdf","file_path":"wiki/attachments/spec.pdf","mime_type":"application/pdf","file_size":12345}'
clam --json wiki attachment list <pageId>   --input-json '{"cursor":"...","limit":25,"fields":"id,file_name,mime_type,file_size"}'
clam --json wiki attachment remove <id>
```

The wiki attachment row only stores metadata; the bytes live in the
storage bucket addressed by `file_path`. See `storage.md` for file upload
commands.

## Search

There is no dedicated wiki search command today. Use the common read-path
query primitives (`../query.md`) over `wiki_pages` / `wiki_page_content`,
or wire an embedding via the shared `embeddings` table if needed.

## Events emitted

`wiki.page.created`, `wiki.page.updated`, `wiki.page.deleted`,
`wiki.page.restored`, `wiki.page.purged`,
plus comment and attachment events in the same namespace.

See `../surfaces.md` to translate these operations to MCP, HTTP, or the
typed client.
