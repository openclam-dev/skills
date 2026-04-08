# Core Domain Reference

Package: `@openclam/core`

---

## Tables

### users
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | Display name |
| type | TEXT NOT NULL | `human` or `agent` |
| identifier | TEXT NOT NULL UNIQUE | Email for humans, slug for agents |
| is_active | INTEGER (boolean) NOT NULL | |
| secret_hash | TEXT | SHA-256 hash of API key |
| password_hash | TEXT | bcrypt hash of login password (human users only) |
| metadata | TEXT | JSON blob |
| created_at | TEXT NOT NULL | ISO 8601 |
| updated_at | TEXT NOT NULL | ISO 8601 |
| deleted_at | TEXT | Soft delete timestamp |

### roles
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| slug | TEXT NOT NULL UNIQUE | |
| name | TEXT NOT NULL | Display name |
| is_system | INTEGER (boolean) NOT NULL | Default false; system roles cannot be modified or deleted |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### role_permissions
| Column | Type | Notes |
|--------|------|-------|
| role_id | TEXT NOT NULL | FK -> roles.id |
| permission | TEXT NOT NULL | Permission string (e.g. `contacts.person.create`, `*`) |
| | | Composite PK: (role_id, permission) |

### user_roles
| Column | Type | Notes |
|--------|------|-------|
| user_id | TEXT NOT NULL | FK -> users.id |
| role_id | TEXT NOT NULL | FK -> roles.id |
| | | Composite PK: (user_id, role_id) |

### rules
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| slug | TEXT NOT NULL UNIQUE | |
| name | TEXT NOT NULL | |
| event_pattern | TEXT NOT NULL | e.g. `person.created`, `person.*`, `*` |
| conditions | TEXT | JSON conditions object |
| action | TEXT | Legacy; replaced by rule_actions |
| priority | INTEGER NOT NULL | Lower runs first |
| is_active | INTEGER (boolean) NOT NULL | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### rule_actions
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| rule_id | TEXT NOT NULL | FK -> rules.id |
| extension | TEXT NOT NULL | Extension name (e.g. `contacts`) |
| action | TEXT NOT NULL | Action name (e.g. `send_welcome`) |
| params_json | TEXT | JSON parameters |
| execution_order | INTEGER NOT NULL | Sequential order |
| created_at | TEXT NOT NULL | |

### projects
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | |
| description | TEXT | |
| status | TEXT NOT NULL | e.g. `active`, `archived` |
| owner_id | TEXT | |
| owner_type | TEXT | e.g. `human`, `agent` |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### log
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| person_id | TEXT | FK reference (not enforced) |
| org_id | TEXT | |
| user_id | TEXT NOT NULL | FK -> users.id |
| type | TEXT NOT NULL | Activity type |
| subject | TEXT | |
| body | TEXT | |
| source | TEXT NOT NULL | e.g. `cli`, `api` |
| created_at | TEXT NOT NULL | |

### config
| Column | Type | Notes |
|--------|------|-------|
| key | TEXT PK | Config key |
| value | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

### events
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| source | TEXT NOT NULL | e.g. `sor/core`, `sor/contacts` |
| event_type | TEXT NOT NULL | e.g. `person.created` |
| entity_type | TEXT NOT NULL | e.g. `person`, `user` |
| entity_id | TEXT NOT NULL | UUID of the entity |
| payload | TEXT | JSON |
| user_id | TEXT | FK -> users.id; nullable for system events |
| created_at | TEXT NOT NULL | |
| processed_at | TEXT | Set when completed |
| status | TEXT NOT NULL | `pending`, `processing`, `completed`, `failed` |
| failure_reason | TEXT | |
| attempts | INTEGER NOT NULL | Default 0 |

### tags
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| label | TEXT NOT NULL | Display label |
| slug | TEXT NOT NULL UNIQUE | Auto-generated from label |
| color | TEXT | |
| created_by | TEXT | FK -> users.id |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

### custom_field_definitions
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| entity_type | TEXT NOT NULL | e.g. `person`, `org`, `deal`, `task` |
| slug | TEXT NOT NULL | Auto-generated from label; UNIQUE per entity_type |
| label | TEXT NOT NULL | Display label |
| description | TEXT | |
| field_type | TEXT NOT NULL | `text`, `number`, `boolean`, `date`, `select`, `multi_select` |
| options_json | TEXT | JSON array of options for select/multi_select |
| required | BOOLEAN NOT NULL | Default false |
| position | INTEGER NOT NULL | Display order; default 0 |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| | | UNIQUE(entity_type, slug) |

### custom_field_values
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| definition_id | TEXT NOT NULL | FK -> custom_field_definitions.id ON DELETE CASCADE |
| entity_type | TEXT NOT NULL | Denormalized for query performance |
| entity_id | TEXT NOT NULL | UUID of the target entity |
| value_text | TEXT | Used for text, select, multi_select |
| value_number | INTEGER | Used for number |
| value_boolean | BOOLEAN | Used for boolean |
| value_date | TEXT | Used for date |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| | | UNIQUE(definition_id, entity_id) |

---

## Service Operations

### userService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create a new user |
| `list(db, tables, filters?, pagination?)` | List users; filters: `type`, `active`; excludes soft-deleted |
| `get(db, tables, idOrIdentifier)` | Get by UUID or identifier; excludes soft-deleted |
| `getByIdentifier(db, tables, identifier)` | Get by identifier (includes soft-deleted) |
| `search(db, tables, query, filters?, pagination?)` | Search by name/identifier |
| `update(db, tables, id, input, ctx?)` | Update user fields |
| `deactivate(db, tables, id, ctx?)` | Set `is_active = false` |
| `remove(db, tables, id, ctx?)` | **Soft delete** (sets `deleted_at`) |
| `restore(db, tables, id, ctx?)` | Restore soft-deleted user |
| `purge(db, tables, id, ctx?)` | **Hard delete** (permanent) |
| `rotateKey(db, tables, identifier, ctx?)` | Generate new API key for agent user; returns `{ api_key }` one-time |

### roleService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create role with permissions |
| `list(db, tables, filters?, pagination?)` | List roles; filter by `search`; excludes soft-deleted |
| `get(db, tables, idOrSlug)` | Get by UUID or slug; excludes soft-deleted |
| `update(db, tables, slug, permissions, ctx?)` | Replace all permissions; blocks system roles |
| `remove(db, tables, slug, ctx?)` | **Soft delete**; blocks system roles |
| `restore(db, tables, idOrSlug, ctx?)` | Restore soft-deleted role |
| `purge(db, tables, idOrSlug, ctx?)` | **Hard delete** (cascades user_roles + role_permissions) |
| `assign(db, tables, userId, roleSlug)` | Assign role to user |
| `revoke(db, tables, userId, roleSlug)` | Revoke role from user |
| `listForUser(db, tables, userId)` | List roles for a user with permissions |
| `hasPermission(db, tables, userId, permission)` | Check if user has permission via roles |
| `ensureAdmin(db, tables)` | Ensure system admin role exists |

### ruleService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input)` | Create rule with actions |
| `list(db, tables, filters?, pagination?)` | List rules; filter by `search`; excludes soft-deleted |
| `get(db, tables, idOrSlug)` | Get by UUID or slug with actions; excludes soft-deleted |
| `update(db, tables, slug, input)` | Update rule fields and/or replace actions |
| `remove(db, tables, slug, ctx?)` | **Soft delete** |
| `restore(db, tables, idOrSlug, ctx?)` | Restore soft-deleted rule |
| `purge(db, tables, idOrSlug, ctx?)` | **Hard delete** (cascades rule_actions) |
| `enable(db, tables, slug)` | Set `is_active = true` |
| `disable(db, tables, slug)` | Set `is_active = false` |

### projectService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create project |
| `list(db, tables, filters?, pagination?)` | List; filters: `status`, `ownerId`, `ownerType`; excludes soft-deleted |
| `get(db, tables, id)` | Get by ID; excludes soft-deleted |
| `update(db, tables, id, input, ctx?)` | Update project fields |
| `archive(db, tables, id, ctx?)` | Set status to `archived` and soft-delete |
| `remove(db, tables, id, ctx?)` | **Soft delete** |
| `restore(db, tables, id, ctx?)` | Restore soft-deleted project |
| `purge(db, tables, id, ctx?)` | **Hard delete** (permanent) |

### logService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input)` | Create log entry |
| `list(db, tables, filters?, pagination?)` | List; filters: `search`, `personId`, `orgId`, `userId`, `type`, `dateFrom`, `dateTo` |
| `get(db, tables, id)` | Get by ID |
| `update(db, tables, id, input)` | Update subject/body |
| `remove(db, tables, id)` | **Hard delete** (no soft delete) |
| `timeline(db, tables, personId, pagination?)` | Activity timeline for a person |

### tagService
| Operation | Description |
|-----------|-------------|
| `create(db, tables, input)` | Create tag; returns existing if slug matches |
| `list(db, tables, filters?)` | List; filter by `search` |
| `get(db, tables, id)` | Get by ID |
| `getBySlug(db, tables, slug)` | Get by slug |
| `getOrCreate(db, tables, label, color?, created_by?)` | Find or create |
| `update(db, tables, id, input)` | Update label/color |
| `remove(db, tables, id)` | **Hard delete** (no soft delete) |
| `rename(db, tables, oldLabel, newLabel)` | Rename by label |

### customFieldService
| Operation | Description |
|-----------|-------------|
| `createDefinition(db, tables, input)` | Create a custom field definition |
| `listDefinitions(db, tables, filters?, pagination?)` | List definitions; filters: `entity_type`, `field_type`, `search` |
| `getDefinition(db, tables, id)` | Get definition by ID |
| `getDefinitionBySlug(db, tables, entityType, slug)` | Get definition by entity type + slug |
| `updateDefinition(db, tables, id, input)` | Update definition (label, description, required, position) |
| `removeDefinition(db, tables, id)` | **Hard delete** definition + all its values (CASCADE) |
| `setValue(db, tables, definitionId, entityType, entityId, value)` | Set/upsert a single field value |
| `setValues(db, tables, entityType, entityId, fields)` | Set multiple fields by slug; `fields: [{ slug, value }]` |
| `getValuesForEntity(db, tables, entityType, entityId)` | Returns `{ slug: typedValue }` map |
| `getValuesForEntities(db, tables, entityType, entityIds)` | Batch: returns `{ entityId: { slug: value } }` |
| `removeValue(db, tables, definitionId, entityId)` | Remove a single field value |
| `removeAllValues(db, tables, entityType, entityId)` | Remove all field values for an entity |

### eventService
| Operation | Description |
|-----------|-------------|
| `emit(db, tables, input)` | Insert raw event |
| `emitEvent(db, tables, opts)` | Convenience helper with standard `sor/` source prefix |
| `getById(db, tables, id)` | Get by ID |
| `list(db, tables, filters?, pagination?)` | List; filters: `eventType`, `entityType`, `source`, `status` |
| `listPending(db, tables, opts?)` | List pending/failed events |
| `processEvents(db, tables, opts?)` | Process outbox: run handlers + rule actions |
| `on(eventPattern, handler)` | Register in-process event handler |
| `off(eventPattern, handler?)` | Unregister handler |

---

## CLI Commands

> **Note:** Workspace, environment, database, daemon, secrets, and config commands have moved to `infrastructure.md`.

### User

All list commands return `{ data: [...], nextCursor: string | null, hasMore: boolean }`.

```
# add — creates user; response differs by type
# Agent: returns { ok, user, api_key } — API key stored in vault, shown once
# Human: returns { ok, user, setup_code, expires_in } — 6-digit code, 15 min expiry
clam --json user add --input-json '{"name":"Research Agent","type":"agent","identifier":"research-agent-01"}'
clam --json user add --input-json '{"name":"Alice Smith","type":"human","identifier":"alice@co.com"}'

# list — returns paginated response
clam --json user list
clam --json user list --input-json '{"search":"...","type":"human","cursor":"...","limit":25}'

# get — identifier can be positional or in --input-json
clam --json user get <identifier>
clam --json user get --input-json '{"identifier":"..."}'

# update — identifier can be positional or in --input-json
clam --json user update <identifier> --input-json '{"name":"..."}'
clam --json user update --input-json '{"identifier":"...","name":"..."}'

# remove — identifier can be positional or in --input-json
clam --json user remove <identifier>
clam --json user remove --input-json '{"identifier":"..."}'

# restore — id can be positional or in --input-json
clam --json user restore <id>
clam --json user restore --input-json '{"id":"..."}'

# purge — id can be positional or in --input-json
clam --json user purge <id>
clam --json user purge --input-json '{"id":"..."}'

# rotate-key — identifier can be positional or in --input-json (agent users only)
clam --json user rotate-key <identifier>
clam --json user rotate-key --input-json '{"identifier":"..."}'

# passwd — set or change login password (human users only, interactive)
clam passwd

# login — human users only; agents CANNOT use login
clam login                                 # interactive picker (human users only)
clam login <identifier>                    # direct login, prompts for password
clam login <identifier> --code <code>      # activate with setup code, prompts to set password
```

### Role

```
# add — slug can be positional or in --input-json
clam --json role add <slug> --input-json '{"name":"...","permissions":["..."]}'
clam --json role add --input-json '{"slug":"...","name":"...","permissions":["..."]}'

# list — returns paginated response
clam --json role list
clam --json role list --input-json '{"search":"...","cursor":"...","limit":25}'

# get — slug can be positional or in --input-json
clam --json role get <slug>
clam --json role get --input-json '{"slug":"..."}'

# update — slug can be positional or in --input-json
clam --json role update <slug> --input-json '{"permissions":["..."]}'
clam --json role update --input-json '{"slug":"...","permissions":["..."]}'

# remove — slug can be positional or in --input-json
clam --json role remove <slug>
clam --json role remove --input-json '{"slug":"..."}'

# restore — slug can be positional or in --input-json
clam --json role restore <slug>
clam --json role restore --input-json '{"slug":"..."}'

# purge — slug can be positional or in --input-json
clam --json role purge <slug>
clam --json role purge --input-json '{"slug":"..."}'

# assign — args can be positional or in --input-json
clam --json role assign <user-identifier> <role-slug>
clam --json role assign --input-json '{"user_identifier":"...","role_slug":"..."}'

# revoke — args can be positional or in --input-json
clam --json role revoke <user-identifier> <role-slug>
clam --json role revoke --input-json '{"user_identifier":"...","role_slug":"..."}'

clam --json role permissions
```

### Rule

```
# add — slug can be positional or in --input-json
clam --json rule add <slug> --input-json '{"name":"...","event_pattern":"...","actions":[...]}'
clam --json rule add --input-json '{"slug":"...","name":"...","event_pattern":"...","actions":[...]}'

# list — returns paginated response
clam --json rule list
clam --json rule list --input-json '{"search":"...","cursor":"...","limit":25}'

# get — slug can be positional or in --input-json
clam --json rule get <slug>
clam --json rule get --input-json '{"slug":"..."}'

# update — slug can be positional or in --input-json
clam --json rule update <slug> --input-json '{"name":"...","event_pattern":"..."}'
clam --json rule update --input-json '{"slug":"...","name":"...","event_pattern":"..."}'

# remove — slug can be positional or in --input-json
clam --json rule remove <slug>
clam --json rule remove --input-json '{"slug":"..."}'

# restore — slug can be positional or in --input-json
clam --json rule restore <slug>
clam --json rule restore --input-json '{"slug":"..."}'

# purge — slug can be positional or in --input-json
clam --json rule purge <slug>
clam --json rule purge --input-json '{"slug":"..."}'

# enable / disable — slug can be positional or in --input-json
clam --json rule enable <slug>
clam --json rule enable --input-json '{"slug":"..."}'
clam --json rule disable <slug>
clam --json rule disable --input-json '{"slug":"..."}'

clam --json rule actions
```

### Project

```
# add — name can be positional or in --input-json
clam --json project add <name> --input-json '{"description":"...","owner_id":"...","owner_type":"human"}'
clam --json project add --input-json '{"name":"...","description":"...","owner_id":"...","owner_type":"human"}'

# list — returns paginated response
clam --json project list
clam --json project list --input-json '{"status":"active","owner_type":"human","cursor":"...","limit":25}'

# get — id can be positional or in --input-json
clam --json project get <id>
clam --json project get --input-json '{"id":"..."}'

# update — id can be positional or in --input-json
clam --json project update <id> --input-json '{"name":"...","description":"...","status":"..."}'
clam --json project update --input-json '{"id":"...","name":"...","description":"...","status":"..."}'

# archive — id can be positional or in --input-json
clam --json project archive <id>
clam --json project archive --input-json '{"id":"..."}'

# remove — id can be positional or in --input-json
clam --json project remove <id>
clam --json project remove --input-json '{"id":"..."}'

# restore — id can be positional or in --input-json
clam --json project restore <id>
clam --json project restore --input-json '{"id":"..."}'

# purge — id can be positional or in --input-json
clam --json project purge <id>
clam --json project purge --input-json '{"id":"..."}'
```

### Event

```
# list — returns paginated response
clam --json event list
clam --json event list --input-json '{"event_type":"person.created","entity_type":"person","source":"sor/contacts","status":"pending","cursor":"...","limit":25}'

# get — id can be positional or in --input-json
clam --json event get <id>
clam --json event get --input-json '{"id":"..."}'

# process — limit can be in --input-json
clam --json event process
clam --json event process --input-json '{"limit":10}'
```

### Log

```
# add — type and person_id can be positional or in --input-json
clam --json log add <type> <person-id> --input-json '{"subject":"...","body":"...","org_id":"..."}'
clam --json log add --input-json '{"type":"...","person_id":"...","subject":"...","body":"...","org_id":"..."}'

# list — returns paginated response
clam --json log list
clam --json log list --input-json '{"search":"...","person_id":"...","org_id":"...","type":"...","source":"auth","date_from":"...","date_to":"...","cursor":"...","limit":25}'

# get — id can be positional or in --input-json
clam --json log get <id>
clam --json log get --input-json '{"id":"..."}'

# update — id can be positional or in --input-json
clam --json log update <id> --input-json '{"subject":"...","body":"..."}'
clam --json log update --input-json '{"id":"...","subject":"...","body":"..."}'

# remove — id can be positional or in --input-json
clam --json log remove <id>
clam --json log remove --input-json '{"id":"..."}'

# timeline — person_id can be positional or in --input-json; returns paginated response
clam --json log timeline <person-id>
clam --json log timeline --input-json '{"person_id":"...","cursor":"...","limit":25}'
```

### Tag

```
clam --json tag add --input-json '{"label":"YC","color":"#F97316"}'
clam --json tag list --input-json '{"search":"...","cursor":"...","limit":25}'
clam --json tag rename --input-json '{"old_name":"...","new_name":"..."}'
clam --json tag purge --input-json '{"name":"..."}'
```

### API Key

```
clam --json api-key create --input-json '{"name":"my-agent"}'
clam --json api-key list
clam --json api-key get --input-json '{"id":"..."}'
clam --json api-key revoke --input-json '{"id":"..."}'
clam --json api-key rotate --input-json '{"id":"..."}'
```

### SQL

```
clam --json sql --input-json '{"query":"SELECT ..."}'
```

---

## Paginated List Response

All `list` commands return this shape:

```json
{
  "data": [ ... ],
  "nextCursor": "cursor-string-or-null",
  "hasMore": true
}
```

To fetch the next page, pass `nextCursor` as `cursor` in `--input-json`. Stop when `hasMore` is `false`.

---

## Global Flags

- `--json` -- Required. Produce compact JSON output (no pretty printing, no colors). All commands must include this flag.
- `--markdown` -- Output as markdown tables (lists) or key-value blocks (single objects). Pairs well with `--fields`. Lower token count than JSON.
- `--token <api-key>` -- API key for authentication. Alternative to `OPENCLAM_TOKEN` env var.
- `--env <env>` -- Override the active environment.
- `--input-json '<json>'` -- Pass all structured input as a single JSON object. Overrides positional args and individual flags. See shapes below.

---

## Input JSON Shapes

All commands accept `--input-json '<json>'`. The JSON shape for each:

#### user add

```typescript
interface UserAddInput {
  name: string;                    // required (or positional)
  type: "human" | "agent";        // required
  identifier: string;             // required
}
```

#### user list

```typescript
interface UserListInput {
  search?: string;                 // search by name or identifier
  type?: "human" | "agent";
  cursor?: string;                 // pagination cursor from previous nextCursor
  limit?: number;                  // max items per page
}
```

#### user update

```typescript
interface UserUpdateInput {
  identifier: string;              // required (or positional)
  name?: string;
}
```

#### user get / remove / restore / purge

```typescript
// get and remove use identifier; restore and purge use id
{ identifier: string }  // for get, remove
{ id: string }          // for restore, purge
```

#### user rotate-key

```typescript
{ identifier: string }  // agent user identifier (or positional)
```

Returns `{ ok: true, identifier: string, api_key: string }`. The `api_key` is shown once — store it immediately.

> **User creation responses differ by type:**
> - **Agent** (`type: "agent"`): response includes a one-time `api_key` field (stored in vault). Store it securely; it cannot be retrieved again.
> - **Human** (`type: "human"`): response includes a `setup_code` (6-digit, expires in 15 minutes) and `expires_in`. The human activates their account via `clam login <identifier> --code <code>`, which prompts them to set a password. Subsequent logins require the password.

#### passwd

```bash
clam passwd                                # Set or change login password (human users only)
```

Interactive command — prompts for current password (if one is set) and new password. Only available to human users.

#### role add

```typescript
interface RoleAddInput {
  slug: string;                    // required (or positional)
  name?: string;                   // defaults to slug
  permissions?: string[];          // array of permission strings
}
```

#### role list

```typescript
interface RoleListInput {
  search?: string;
  cursor?: string;
  limit?: number;
}
```

#### role update

```typescript
interface RoleUpdateInput {
  slug: string;                    // required (or positional)
  permissions: string[];           // required — replaces all existing permissions
}
```

#### role assign / revoke

```typescript
interface RoleAssignInput {
  user_identifier: string;         // required (or first positional)
  role_slug: string;               // required (or second positional)
}
```

#### rule add

```typescript
interface RuleAddInput {
  slug: string;                    // required (or positional)
  name?: string;                   // defaults to slug
  event_pattern: string;           // required (e.g. "person.created", "person.*", "*")
  conditions?: string | null;      // JSON conditions object as string
  priority?: number;               // default: 0, lower runs first
  actions?: Array<{
    extension: string;             // required
    action: string;                // required
    params_json?: string | null;
  }>;
}
```

#### rule list

```typescript
interface RuleListInput {
  search?: string;
  cursor?: string;
  limit?: number;
}
```

#### rule update

```typescript
interface RuleUpdateInput {
  slug: string;                    // required (or positional)
  name?: string;
  event_pattern?: string;
  conditions?: string | null;
  priority?: number;
  actions?: Array<{
    extension: string;
    action: string;
    params_json?: string | null;
  }>;
}
```

#### project add

```typescript
interface ProjectAddInput {
  name: string;                    // required (or positional)
  description?: string | null;
  status?: string;                 // default: "active"
  owner_id?: string | null;
  owner_type?: "human" | "agent" | null;
}
```

#### project list

```typescript
interface ProjectListInput {
  status?: string;
  owner_type?: string;
  cursor?: string;
  limit?: number;
}
```

#### project update

```typescript
interface ProjectUpdateInput {
  id: string;                      // required (or positional)
  name?: string;
  description?: string | null;
  status?: string;
}
```

#### event list

```typescript
interface EventListInput {
  event_type?: string;             // e.g. "person.created"
  entity_type?: string;            // e.g. "person"
  source?: string;                 // e.g. "sor/contacts"
  status?: "pending" | "processing" | "completed" | "failed";
  cursor?: string;
  limit?: number;
}
```

#### event process

```typescript
interface EventProcessInput {
  limit?: number;                  // max events to process
}
```

#### log add

```typescript
interface LogAddInput {
  type: string;                    // required (or first positional)
  person_id: string;               // required (or second positional)
  subject?: string | null;
  body?: string | null;
  org_id?: string | null;
}
```

#### log list

```typescript
interface LogListInput {
  search?: string;
  person_id?: string;
  org_id?: string;
  type?: string;                   // e.g. "auth.login", "auth.key_used"
  source?: string;                 // e.g. "auth", "cli" — filter by log source
  date_from?: string;              // ISO date string
  date_to?: string;                // ISO date string
  cursor?: string;
  limit?: number;
}
```

#### log update

```typescript
interface LogUpdateInput {
  id: string;                      // required (or positional)
  subject?: string | null;
  body?: string | null;
}
```

#### log timeline

```typescript
interface LogTimelineInput {
  person_id: string;               // required (or positional)
  cursor?: string;
  limit?: number;
}
```

#### tag add

```typescript
interface TagAddInput {
  label: string;                   // required — display name (slug auto-generated)
  color?: string | null;           // hex color (e.g. "#F97316")
}
```

#### tag list

```typescript
interface TagListInput {
  search?: string;
  cursor?: string;
  limit?: number;
}
```

#### tag rename

```typescript
interface TagRenameInput {
  old_name: string;                // required (or first positional)
  new_name: string;                // required (or second positional)
}
```

#### tag purge

```typescript
interface TagPurgeInput {
  name: string;                    // required (or positional) — tag label
}
```

#### field define

```typescript
interface FieldDefineInput {
  entity_type: string;             // required (positional) — e.g. "person", "org", "deal"
  label: string;                   // required (positional) — display label
  field_type: string;              // required (--type or field_type) — text, number, boolean, date, select, multi_select
  slug?: string;                   // auto-generated from label if omitted
  description?: string;            // field description
  options_json?: string;           // JSON array or comma-separated for select/multi_select
  required?: boolean;              // default false
  position?: number;               // display order, default 0
}
```

#### field list

```typescript
interface FieldListInput {
  entity_type?: string;            // filter by entity type
  field_type?: string;             // filter by field type
  search?: string;                 // search by label
  cursor?: string;
  limit?: number;
}
```

#### field get

```typescript
interface FieldGetInput {
  id: string;                      // required (positional) — definition UUID
}
```

#### field update

```typescript
interface FieldUpdateInput {
  id: string;                      // required (positional) — definition UUID
  label?: string;
  description?: string;
  required?: boolean;
  position?: number;
}
```

#### field remove

```typescript
interface FieldRemoveInput {
  id: string;                      // required (positional) — cascades to all values
}
```

#### field set

Single value:
```typescript
interface FieldSetInput {
  entity_type: string;             // required (positional) — e.g. "person"
  entity_id: string;               // required (positional) — target entity UUID
  slug: string;                    // required (positional) — field slug
  value: string | number | boolean | null;  // required (positional) — value to set
}
```

Bulk set via `--input-json`:
```typescript
interface FieldSetBulkInput {
  entity_type: string;             // required
  entity_id: string;               // required
  fields: Array<{                  // required — sets multiple fields at once
    slug: string;
    value: string | number | boolean | null;
  }>;
}
```

#### field values

```typescript
interface FieldValuesInput {
  entity_type: string;             // required (positional) — e.g. "person"
  entity_id: string;               // required (positional) — target entity UUID
}
```

Returns: `{ slug: typedValue, ... }` — e.g. `{ "linkedin_url": "https://...", "is_vip": true }`

#### api-key create

```typescript
interface ApiKeyCreateInput {
  name: string;                    // required — key display name
}
```

#### sql

```typescript
// Positional: clam sql "SELECT ..."
// Or: clam sql --input-json '{"query":"SELECT ..."}'
interface SqlInput {
  query: string;                   // required (positional or in --input-json) — a read-only SELECT statement
}
```

---

## API Routes

### Users
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/users` | List users; query: `type` |
| POST | `/api/users` | Create user |
| GET | `/api/users/by-identifier/:identifier` | Get user by identifier |
| GET | `/api/users/search` | Search users; query: `q`, `type` |
| GET | `/api/users/:id` | Get user by ID |
| PUT | `/api/users/:id` | Update user |
| DELETE | `/api/users/:id` | Soft delete user |
| PUT | `/api/users/:id/restore` | Restore soft-deleted user |
| DELETE | `/api/users/:id/purge` | Permanently delete user |
| POST | `/api/users/by-identifier/:identifier/rotate-key` | Rotate API key for agent user; returns new `api_key` |
| GET | `/api/users/:id/roles` | List roles for user |

### Config
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/config` | Show all config |
| GET | `/api/config/:key` | Get config value |
| PUT | `/api/config/:key` | Set config value; body: `{ value }` |

### Roles
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/roles` | List roles; query: `search` |
| POST | `/api/roles` | Create role |
| GET | `/api/roles/:slug` | Get role |
| PUT | `/api/roles/:slug` | Update role permissions; body: `{ permissions }` |
| DELETE | `/api/roles/:slug` | Soft delete role |
| PUT | `/api/roles/:slug/restore` | Restore soft-deleted role |
| DELETE | `/api/roles/:slug/purge` | Permanently delete role |
| POST | `/api/roles/assign` | Assign role; body: `{ userId, roleSlug }` |
| POST | `/api/roles/revoke` | Revoke role; body: `{ userId, roleSlug }` |

### Rules
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/rules` | List rules; query: `search` |
| POST | `/api/rules` | Create rule |
| GET | `/api/rules/:slug` | Get rule with actions |
| PUT | `/api/rules/:slug` | Update rule |
| DELETE | `/api/rules/:slug` | Soft delete rule |
| PUT | `/api/rules/:slug/restore` | Restore soft-deleted rule |
| DELETE | `/api/rules/:slug/purge` | Permanently delete rule |
| PUT | `/api/rules/:slug/enable` | Enable rule |
| PUT | `/api/rules/:slug/disable` | Disable rule |

### Log
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/log` | List log entries; query: `type`, `personId`, `orgId` |
| POST | `/api/log` | Create log entry |
| GET | `/api/log/:id` | Get log entry |
| PUT | `/api/log/:id` | Update log entry |
| DELETE | `/api/log/:id` | Delete log entry (hard delete) |
| GET | `/api/log/timeline/:personId` | Timeline for a person |

### Projects
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/projects` | List projects; query: `status`, `ownerId`, `ownerType` |
| POST | `/api/projects` | Create project |
| GET | `/api/projects/:id` | Get project |
| PUT | `/api/projects/:id` | Update project |
| PUT | `/api/projects/:id/archive` | Archive project |
| DELETE | `/api/projects/:id` | Soft delete project |
| PUT | `/api/projects/:id/restore` | Restore soft-deleted project |
| DELETE | `/api/projects/:id/purge` | Permanently delete project |

### Tags
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/tags` | List tags; query: `search` |
| PUT | `/api/tags/rename` | Rename tag; body: `{ oldName, newName }` |
| DELETE | `/api/tags/:tag` | Delete tag by slug (hard delete) |

### Events
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/events` | List events; query: `entityType`, `eventType`, `source` |
| GET | `/api/events/:id` | Get event |
| POST | `/api/events/process` | Process pending events |

### Custom Fields
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/custom-fields` | List definitions; query: `entityType`, `fieldType`, `search` |
| POST | `/api/custom-fields` | Create definition |
| GET | `/api/custom-fields/:id` | Get definition |
| PUT | `/api/custom-fields/:id` | Update definition |
| DELETE | `/api/custom-fields/:id` | Delete definition (cascades to all values) |
| GET | `/api/custom-fields/values/:entityType/:entityId` | Get all custom field values for an entity |
| PUT | `/api/custom-fields/values/:entityType/:entityId` | Set custom field values; body: `{ fields: [{ slug, value }] }` |

### Search
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/search` | Global search; query: `q` |
