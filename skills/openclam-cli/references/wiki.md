# Wiki Reference

Package: `@openclam/wiki`
Dependencies: none

---

## Tables

### wiki_pages
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| parent_id | TEXT | Self-ref for nesting (null = root) |
| title | TEXT NOT NULL | Page title |
| slug | TEXT NOT NULL UNIQUE | URL-friendly identifier |
| icon | TEXT | Emoji or identifier |
| cover_image | TEXT | Cover image path |
| position | INTEGER | Sort order within siblings |
| is_pinned | INTEGER | 0 or 1 |
| is_archived | INTEGER | 0 or 1 |
| created_by | TEXT | User UUID |
| created_at | TEXT NOT NULL | ISO 8601 |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete |

### wiki_page_content
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| page_id | TEXT NOT NULL | FK → wiki_pages (CASCADE), 1:1 |
| title | TEXT NOT NULL | Current title (denormalized for FTS) |
| body | TEXT NOT NULL | Current markdown content |

FTS5 indexed (title + body). Vector embeddings registered via embedding registry.

### wiki_page_versions
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| page_id | TEXT NOT NULL | FK → wiki_pages (CASCADE) |
| title | TEXT NOT NULL | Title at this version |
| body | TEXT NOT NULL | Content at this version |
| version_number | INTEGER NOT NULL | Monotonically increasing per page |
| edited_by | TEXT | User UUID |
| change_summary | TEXT | What changed |
| created_at | TEXT NOT NULL | |

### wiki_page_tags
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| page_id | TEXT NOT NULL | FK → wiki_pages (CASCADE) |
| tag_id | TEXT NOT NULL | Reference to core tags |
| created_at | TEXT NOT NULL | |

### wiki_page_comments
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| page_id | TEXT NOT NULL | FK → wiki_pages (CASCADE) |
| author_id | TEXT | User UUID |
| parent_id | TEXT | Self-ref for threading |
| body | TEXT NOT NULL | Comment text |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete |

### wiki_page_links
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| source_page_id | TEXT NOT NULL | FK → wiki_pages (CASCADE) |
| target_page_id | TEXT NOT NULL | FK → wiki_pages (CASCADE) |
| created_at | TEXT NOT NULL | |

Populated automatically by parsing `[[slug]]` patterns in page body.

### wiki_page_attachments
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| page_id | TEXT NOT NULL | FK → wiki_pages (CASCADE) |
| file_name | TEXT NOT NULL | |
| file_path | TEXT NOT NULL | |
| file_size | INTEGER | Bytes |
| mime_type | TEXT | |
| uploaded_by | TEXT | User UUID |
| created_at | TEXT NOT NULL | |

---

## Service Operations

### pageService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create page + content + first version + sync links |
| `list(db, tables, filters?, pagination?)` | List; filters: `search`, `parent_id`, `is_pinned`, `is_archived` |
| `count(db, tables, filters?)` | Count matching pages |
| `get(db, tables, id)` | Get by ID (metadata only) |
| `getBySlug(db, tables, slug)` | Get by slug |
| `getWithContent(db, tables, id)` | Get page + current content |
| `update(db, tables, id, input, ctx?)` | Update; creates new version if title/body changed |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore |
| `purge(db, tables, id, ctx?)` | Hard delete |

### versionService
| Operation | Description |
|-----------|-------------|
| `list(db, tables, pageId, pagination?)` | List versions for page (newest first) |
| `get(db, tables, id)` | Get version by ID |
| `getByNumber(db, tables, pageId, versionNumber)` | Get specific version by number |

### commentService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Add comment to page |
| `list(db, tables, pageId, pagination?)` | List comments for page |
| `get(db, tables, id)` | Get by ID |
| `update(db, tables, id, input, ctx?)` | Update body |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore |

### linkService
| Operation | Description |
|-----------|-------------|
| `listBacklinks(db, tables, targetPageId, pagination?)` | Pages linking TO this page |
| `listForwardLinks(db, tables, sourcePageId, pagination?)` | Pages this page links TO |

### attachmentService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Add attachment to page |
| `list(db, tables, pageId, pagination?)` | List attachments for page |
| `get(db, tables, id)` | Get by ID |
| `remove(db, tables, id, ctx?)` | Hard delete |

---

## CLI Commands

### wiki page

```
clam --json wiki page add --input-json '{"title":"...","body":"...","parent_id":"...","slug":"..."}'
clam --json wiki page list --input-json '{"search":"...","parent_id":"...","cursor":"...","limit":25}'
clam --json wiki page count --input-json '{"search":"...","parent_id":"..."}'
clam --json wiki page get <id>
clam --json wiki page update --input-json '{"id":"<id>","title":"...","body":"...","change_summary":"..."}'
clam --json wiki page remove <id>
clam --json wiki page restore <id>
```

### wiki version

```
clam --json wiki version list <page-id>
clam --json wiki version get <id>
```

### wiki comment

```
clam --json wiki comment add --input-json '{"page_id":"...","body":"...","parent_id":"..."}'
clam --json wiki comment list <page-id>
clam --json wiki comment update --input-json '{"id":"<id>","body":"..."}'
clam --json wiki comment remove <id>
```

### wiki attachment

```
clam --json wiki attachment add --input-json '{"page_id":"...","file_name":"...","file_path":"...","mime_type":"...","file_size":1024}'
clam --json wiki attachment list <page-id>
clam --json wiki attachment remove <id>
```

---

## API Routes

### Pages
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/wiki/pages` | Create page |
| GET | `/api/wiki/pages` | List pages |
| GET | `/api/wiki/pages/count` | Count pages |
| GET | `/api/wiki/pages/by-slug/:slug` | Get by slug |
| GET | `/api/wiki/pages/:id` | Get page with content |
| PUT | `/api/wiki/pages/:id` | Update page |
| DELETE | `/api/wiki/pages/:id` | Soft delete |
| PUT | `/api/wiki/pages/:id/restore` | Restore |
| DELETE | `/api/wiki/pages/:id/purge` | Hard delete |

### Versions
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/wiki/pages/:pageId/versions` | List versions |
| GET | `/api/wiki/versions/:id` | Get version |

### Comments
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/wiki/pages/:pageId/comments` | List comments |
| POST | `/api/wiki/pages/:pageId/comments` | Add comment |
| PUT | `/api/wiki/comments/:id` | Update comment |
| DELETE | `/api/wiki/comments/:id` | Soft delete |

### Links
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/wiki/pages/:id/backlinks` | Pages linking to this page |
| GET | `/api/wiki/pages/:id/links` | Pages this page links to |

### Tags
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/wiki/pages/:pageId/tags` | List tags |
| POST | `/api/wiki/pages/:pageId/tags` | Add tag |
| DELETE | `/api/wiki/pages/:pageId/tags/:tagId` | Remove tag |

### Attachments
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/wiki/pages/:pageId/attachments` | List attachments |
| POST | `/api/wiki/pages/:pageId/attachments` | Add attachment |
| DELETE | `/api/wiki/attachments/:id` | Delete attachment |

---

## Events Emitted

`wiki.page.created`, `wiki.page.updated`, `wiki.page.deleted`
`wiki.comment.created`, `wiki.comment.updated`, `wiki.comment.deleted`
`wiki.attachment.created`, `wiki.attachment.deleted`

## Permissions

`page.create`, `page.read`, `page.update`, `page.delete`
`comment.create`, `comment.read`, `comment.update`, `comment.delete`
`attachment.create`, `attachment.read`, `attachment.delete`
