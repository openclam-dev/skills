# Storage Reference

Package: `@openclam/storage`
Dependencies: none

---

## Tables

### storage_buckets
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | Bucket name |
| slug | TEXT NOT NULL UNIQUE | Auto-slugified from name |
| description | TEXT | |
| uri | TEXT NOT NULL | `file://` or `s3://` URI |
| config | TEXT | JSON-encoded S3 credentials (optional) |
| created_by | TEXT NOT NULL | UUID of creator |
| created_at | TEXT NOT NULL | ISO 8601 |
| updated_at | TEXT NOT NULL | |

### storage_files
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| bucket_id | TEXT NOT NULL | FK → storage_buckets (CASCADE) |
| path | TEXT NOT NULL | Relative path in bucket |
| filename | TEXT NOT NULL | |
| mime_type | TEXT NOT NULL | |
| size | INTEGER NOT NULL | Bytes |
| created_by | TEXT | UUID |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

### storage_bucket_permissions
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| bucket_id | TEXT NOT NULL | FK → storage_buckets (CASCADE) |
| user_id | TEXT NOT NULL | UUID |
| permission | TEXT NOT NULL | `read`, `write`, `admin` |
| granted_by | TEXT | UUID |
| created_at | TEXT NOT NULL | |

---

## Service Operations

### bucketService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input)` | Create bucket; auto-grants admin to creator |
| `list(db, tables, filters, pagination?)` | List; filter: `search` |
| `get(db, tables, id)` | Get by ID |
| `getBySlug(db, tables, slug)` | Get by slug |
| `update(db, tables, id, input, ctx?)` | Update fields |
| `remove(db, tables, id, ctx?)` | Delete bucket (cascades files and permissions) |

### fileService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create file record in bucket |
| `list(db, tables, bucketId, filters, pagination?)` | List; filters: `search` (filename), `path` (prefix) |
| `get(db, tables, id)` | Get by ID |
| `update(db, tables, id, input, ctx?)` | Update path/filename/mime_type |
| `remove(db, tables, id, ctx?)` | Delete file record (returns deleted record for cleanup) |

### permissionService
| Operation | Description |
|-----------|-------------|
| `grant(db, tables, input)` | Grant or upgrade permission; hierarchy: read < write < admin |
| `revoke(db, tables, bucketId, userId)` | Revoke access |
| `list(db, tables, bucketId, pagination?)` | List permissions for bucket |
| `check(db, tables, bucketId, userId, requiredLevel)` | Check if user has at least required level |
| `listBucketsForUser(db, tables, userId)` | Get all buckets user has access to |

---

## CLI Commands

### bucket

```
clam --json storage bucket add --input-json '{"name":"...","slug":"...","description":"...","uri":"file://...","created_by":"..."}'
clam --json storage bucket list --input-json '{"search":"...","cursor":"...","limit":25}'
clam --json storage bucket get <id>
clam --json storage bucket update --input-json '{"id":"<id>","name":"...","description":"..."}'
clam --json storage bucket remove <id>
```

### file

```
clam --json storage file add --input-json '{"bucket_id":"...","path":"...","filename":"...","mime_type":"...","size":1024}'
clam --json storage file list --input-json '{"bucket_id":"...","search":"...","path":"...","cursor":"...","limit":25}'
clam --json storage file get <id>
clam storage file get <id> --content [--dest <path>]
clam --json storage file update --input-json '{"id":"<id>","filename":"...","mime_type":"..."}'
clam --json storage file remove <id>
```

### storage-permission

```
clam --json storage permission grant --input-json '{"bucket_id":"...","user_id":"...","permission":"write"}'
clam --json storage permission revoke --input-json '{"bucket_id":"...","user_id":"..."}'
clam --json storage permission list --input-json '{"bucket_id":"..."}'
clam --json storage permission check --input-json '{"bucket_id":"...","user_id":"...","level":"write"}'
```

---

## Input JSON Shapes

#### bucket add
```typescript
interface BucketAddInput {
  name: string;                    // required
  slug?: string;                   // auto-slugified from name if omitted
  description?: string | null;
  uri: string;                     // required — must start with file:// or s3://
  config?: {                       // S3 credentials (optional)
    endpoint?: string;
    region?: string;
    access_key_id?: string;
    secret_access_key?: string;
  } | null;
  created_by: string;             // required — UUID of creator
}
```

#### file add
```typescript
interface FileAddInput {
  bucket_id: string;               // required (UUID)
  path: string;                    // required — relative path in bucket
  filename: string;                // required
  mime_type: string;               // required
  size: number;                    // required — non-negative integer (bytes)
  created_by?: string | null;
}
```

#### permission grant
```typescript
interface PermissionGrantInput {
  bucket_id: string;               // required (UUID)
  user_id: string;                 // required (UUID)
  permission: "read" | "write" | "admin";  // required
  granted_by?: string | null;
}
```

---

## API Routes

### Buckets
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/storage/buckets` | List buckets |
| POST | `/api/storage/buckets` | Create bucket |
| GET | `/api/storage/buckets/:id` | Get bucket |
| GET | `/api/storage/buckets/by-slug/:slug` | Get by slug |
| PUT | `/api/storage/buckets/:id` | Update |
| DELETE | `/api/storage/buckets/:id` | Delete bucket |

### Files
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/storage/buckets/:bucketId/files` | List files |
| POST | `/api/storage/buckets/:bucketId/files` | Add file |
| GET | `/api/storage/files/:id` | Get file |
| PUT | `/api/storage/files/:id` | Update metadata |
| DELETE | `/api/storage/files/:id` | Delete file |

### Permissions
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/storage/buckets/:bucketId/permissions` | List permissions |
| POST | `/api/storage/buckets/:bucketId/permissions` | Grant permission |
| DELETE | `/api/storage/buckets/:bucketId/permissions/:userId` | Revoke |
| GET | `/api/storage/buckets/:bucketId/permissions/check/:userId` | Check access; query: `level` |

---

## Storage Backends

| URI Scheme | Backend | Description |
|------------|---------|-------------|
| `file://` | Local | Files on disk at `~/.openclam/workspaces/<slug>/envs/<env>/buckets/<id>/` |
| `s3://` | S3 | Remote S3-compatible storage; requires `config` with credentials |

Per-bucket `config` overrides workspace-level S3 config. Credentials can reference vault secrets.

## Events Emitted

`storage.bucket.created`, `storage.bucket.updated`, `storage.bucket.deleted`
`storage.file.created`, `storage.file.updated`, `storage.file.deleted`

## Permissions

`storage_bucket.create`, `storage_bucket.read`, `storage_bucket.update`, `storage_bucket.delete`
`storage_file.create`, `storage_file.read`, `storage_file.update`, `storage_file.delete`
`storage_permission.create`, `storage_permission.read`, `storage_permission.delete`
