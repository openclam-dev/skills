---
name: operate-storage
description: >
  File storage in OpenClam — buckets, files, permissions, backend adapters
  (local and S3), per-bucket vector indexing. Storage is a first-class
  primitive in the core runtime. Covers `clam bucket`, `clam file`, and
  `clam storage`.
metadata:
  author: openclam
  version: 0.1.0
---

# Storage

Package: `@openclam/storage` (first-class primitive in the core runtime — the
tables ship with every workspace, not as an optional extension).

Storage is split into three pieces that map to the CLI surface:

- **Buckets** (`clam bucket …`) — a named, URI-addressed collection of
  files with optional per-bucket S3 credentials and vector-indexing config.
  Creating a bucket auto-grants the creator `admin` permission.
- **Files** (`clam file …`) — catalog records for files that live on a
  backend (local disk or S3). Files are moved/copied through a
  `StorageAdapter` so the DB row and the byte stream stay consistent.
- **Storage admin** (`clam storage …`) — permissions, indexing, semantic
  search, and filesystem sync for local buckets.

## Tables

### storage_buckets

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | Display name |
| slug | TEXT NOT NULL UNIQUE | Auto-slugified from name |
| description | TEXT | |
| uri | TEXT NOT NULL | Must start with `file://` or `s3://` |
| config | TEXT | JSON — S3 credentials + indexing config (optional) |
| created_by | TEXT NOT NULL | UUID of creator |
| created_at | TEXT NOT NULL | ISO 8601 |
| updated_at | TEXT NOT NULL | |

### storage_files

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| bucket_id | TEXT NOT NULL | FK → storage_buckets (CASCADE) |
| path | TEXT NOT NULL | Relative path in bucket |
| filename | TEXT NOT NULL | Display filename |
| mime_type | TEXT NOT NULL | |
| size | INTEGER NOT NULL | Bytes |
| backend_modified_at | TEXT | mtime observed on the backend |
| created_by | TEXT | UUID (nullable) |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

Unique index on `(bucket_id, path)`.

### storage_bucket_permissions

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| bucket_id | TEXT NOT NULL | FK → storage_buckets (CASCADE) |
| user_id | TEXT NOT NULL | UUID |
| permission | TEXT NOT NULL | `read`, `write`, or `admin` |
| granted_by | TEXT | UUID |
| created_at | TEXT NOT NULL | |

Permission hierarchy: `read < write < admin`. Granting a higher level
on an existing row upgrades it; granting a lower level is a no-op.

## Service Operations

### bucketService (`@openclam/storage/services/bucket`)

```ts
add(db, tables, input, ctx?): Promise<Bucket>                         // auto-grants admin to created_by
list(db, tables, filters, pagination?, options?): Promise<Paginated<Bucket>>
count(db, tables, filters?): Promise<{ count: number }>
get(db, tables, id): Promise<Bucket>                                  // throws if missing
getBySlug(db, tables, slug): Promise<Bucket>
update(db, tables, id, input, ctx?): Promise<Bucket>
remove(db, tables, id, ctx?): Promise<void>                           // cascades files + permissions
```

`filters` supports `search`. Pagination is
`{ cursor, limit, sort, order, all }`; sort is any column that exists
on `storageBuckets` (defaults to `created_at`).

### fileService (`@openclam/storage/services/file`)

```ts
add(db, tables, input, ctx?): Promise<StorageFile>                    // enqueues embedding if bucket is indexed
list(db, tables, bucketId|null, filters, pagination?, options?): Promise<Paginated<StorageFile>>
count(db, tables, bucketId, filters?): Promise<{ count: number }>
get(db, tables, id): Promise<StorageFile>
getByPath(db, tables, bucketId, filePath): Promise<StorageFile | null>
update(db, tables, id, input, ctx?): Promise<StorageFile>             // filename / mime_type only
moveRecord(db, tables, id, input, ctx?): Promise<StorageFile>         // DB half of a move; path + size + mtime
remove(db, tables, id, ctx?): Promise<StorageFile>                    // returns the deleted row for backend cleanup
```

`list` accepts `bucketId = null` to enumerate files across every bucket in
the workspace. `filters` supports `search` (on filename) and `path`
(prefix match). Deleting a file also removes any `embeddings` rows for
that file.

### permissionService (`@openclam/storage/services/permission`)

```ts
grant(db, tables, input): Promise<BucketPermission>                   // idempotent; upgrades if higher
revoke(db, tables, bucketId, userId): Promise<void>                   // throws if missing
list(db, tables, bucketId, pagination?): Promise<Paginated<BucketPermission>>
check(db, tables, bucketId, userId, requiredLevel): Promise<boolean>
listBucketsForUser(db, tables, userId): Promise<BucketPermission[]>
```

### indexing services (`@openclam/storage/services/indexing`)

```ts
registerFileEmbeddable(): void                                         // called at command boot
parseIndexingConfig(configStr): IndexingConfig | null
enqueueFileIfIndexable(db, tables, file): Promise<boolean>
removeFileEmbeddings(db, tables, fileId): Promise<void>
reindexBucket(db, tables, bucketId): Promise<{ enqueued: number; skipped: number }>
```

Indexing config lives under `bucket.config.indexing` and defaults to
`{ enabled: false, mime_types: ['text/plain','text/markdown'], max_file_size: 524288 }`.
When enabled, the daemon chunks file content (markdown-aware for
markdown MIME types), hashes it, and enqueues embedding jobs through the
shared `embeddingQueueService`.

## Backends

| URI scheme | Adapter | Resolution |
|------------|---------|------------|
| `file://` | Local | Files on disk, default at `~/.openclam/workspaces/<slug>/envs/<env>/buckets/<id>/` |
| `s3://` | S3 | Uses `@openclam/storage-s3`; reads workspace-level S3 config, overridden by per-bucket `config` |

Per-bucket S3 `config` fields (`endpoint`, `region`, `access_key_id`,
`secret_access_key`) can either be literal strings or `$secret:NAME`
references that resolve through the workspace vault.

## CLI Commands

### clam bucket

```
clam --json bucket add   --input-json '{
  "name":"Docs",
  "slug":"docs",
  "description":"...",
  "uri":"s3://my-bucket/prefix",
  "endpoint":"...","region":"...","access_key_id":"$secret:s3_key","secret_access_key":"$secret:s3_secret",
  "enable_indexing":true,"index_mime_types":"text/plain,text/markdown"
}'
clam --json bucket list   --input-json '{"search":"...","cursor":"...","limit":25,"sort":"created_at","order":"desc","all":false,"fields":"id,name,slug,uri"}'
clam --json bucket count  --input-json '{"search":"..."}'
clam --json bucket get <id|slug>
clam --json bucket update --input-json '{"id":"<id>","name":"...","description":"...","uri":"s3://...","endpoint":"..."}'
clam --json bucket remove <id>
```

If `--uri` is omitted at creation, a local `file://` URI is auto-generated
under the active environment's bucket root. The adapter's `ensureDir()` is
invoked immediately so the backend is ready before the DB insert commits.

### clam file

```
clam --json file add      --input-json '{"bucket_id":"<id>","source":"/local/path","path":"docs/intro.md","filename":"intro.md","mime_type":"text/markdown"}'
clam --json file register --input-json '{"bucket_id":"<id>","path":"existing/on/backend.pdf","filename":"backend.pdf","mime_type":"application/pdf"}'
clam --json file list     --input-json '{"bucket":"<id|slug>","search":"...","path":"docs/","cursor":"...","limit":25,"sort":"created_at","order":"desc","all":false,"fields":"id,filename,path,size"}'
clam --json file count    --input-json '{"bucket":"<id|slug>","search":"...","path":"docs/"}'
clam --json file get <id>
clam         file get <id> --content [--dest <path>]     # streams bytes to stdout or writes to disk
clam --json file update <id> --input-json '{"filename":"...","mime_type":"..."}'
clam --json file move <id>   --input-json '{"path":"new/location.md","filename":"location.md"}'
clam --json file remove <id> --input-json '{"dry_run":false}'
```

`file add` requires a local `--source` path; use `file register` to catalog
a file that was placed on the backend out-of-band. `file list` without
`--bucket` lists across every bucket in the workspace. `file get --content`
is the only command that writes raw bytes (not JSON) to stdout.

### clam storage

```
clam --json storage permission grant   --input-json '{"bucket_id":"<id>","user_id":"<id>","permission":"write"}'
clam --json storage permission revoke  --input-json '{"bucket_id":"<id>","user_id":"<id>"}'
clam --json storage permission list <bucketId>
clam --json storage permission check   --input-json '{"bucket_id":"<id>","user_id":"<id>","level":"write"}'

clam --json storage index run <bucket>       # enqueue every eligible file for embedding
clam --json storage index status <bucket>    # {indexing_enabled, total_files, embedded_files, total_chunks, pending_jobs}

clam --json storage search "<query>" --input-json '{"bucket":"<slug|id>","limit":10,"threshold":0.2}'

clam --json storage sync <bucket>    --input-json '{"dry_run":false}'   # local file:// buckets only
```

`storage sync` walks the filesystem under a local bucket's URI and
reconciles it with the catalog:

- files on disk but not in DB → inserted as new catalog rows
- size / mtime changes → row updated and embedding re-enqueued
- rows with no disk file → removed

## Events emitted

`storage.bucket.created`, `storage.bucket.updated`, `storage.bucket.deleted`,
`storage.file.created`, `storage.file.updated`, `storage.file.moved`,
`storage.file.deleted`.

See `../surfaces.md` to translate these operations to MCP, HTTP, or the
typed client.
