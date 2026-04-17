---
name: operate-icp
description: >
  Ideal Customer Profile in OpenClam — profiles with pluggable score and
  enroll functions, per-profile org enrollments, and denormalized scores
  with factor breakdowns. The `clam icp` CLI.
metadata:
  author: openclam
  version: 0.1.0
---

# ICP

Package: `@openclam/icp` (daemon extension).

ICP profiles are declarative: the rules for enrolment and scoring are
written as functions in `@openclam/functions` and referenced by slug. The
daemon validates that those references resolve whenever a profile is
created or updated, flipping `is_valid` / `error` accordingly. Scoring
runs from three action entry points:

- `calculate-score` — event-driven per-org scoring (wired to `org.created`
  and `org.updated` via rules).
- `sweep-changed-orgs` — hourly cron that rescores only orgs updated since
  each profile's `last_swept_at` bookmark (or all orgs with `force_all`).
- `recalculate-profile` — manual recompute of one profile across every
  enrolled org.

## Tables

### icp_profiles

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| slug | TEXT NOT NULL UNIQUE | Primary CLI handle; used as `profile_slug` in function inputs |
| name | TEXT NOT NULL | Display name (denormalized into `icp_scores.profile_name`) |
| description | TEXT | |
| status | TEXT NOT NULL | `active` (default) or `inactive` |
| score_function | TEXT | Function slug that returns `{ score, max_score, factors? }` |
| enroll_function | TEXT | Function slug that returns `true`/`false` for auto-enroll |
| is_valid | BOOLEAN NOT NULL | Default true; flipped when a referenced function can't be resolved |
| error | TEXT | Last validation error, if any |
| last_swept_at | TEXT | Bookmark for incremental sweeps |
| created_by, updated_by | TEXT | |
| created_at, updated_at, deleted_at | TEXT | Soft delete via `deleted_at` |

### icp_profile_members

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| profile_id | TEXT NOT NULL | FK → icp_profiles |
| org_id | TEXT NOT NULL | Contacts org UUID |
| enrolled_by | TEXT NOT NULL | `manual` (default) or `auto` |
| enrolled_at | TEXT NOT NULL | |

Unique index on `(profile_id, org_id)`; additional index on `org_id`.

### icp_scores

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| profile_id | TEXT NOT NULL | FK → icp_profiles |
| profile_name | TEXT NOT NULL | Denormalized copy of the profile name at score time |
| org_id | TEXT NOT NULL | |
| score | REAL NOT NULL | |
| max_score | REAL NOT NULL | |
| factors_json | TEXT | JSON-encoded factor breakdown returned by the score function |
| scored_at | TEXT NOT NULL | |

Unique index on `(profile_id, org_id)`, index on `org_id`, and a composite
`(profile_id, score)` index for descending-score queries. `upsertScore`
keeps one row per `(profile_id, org_id)`.

## Service Operations

### profileService

```ts
add(db, tables, input, ctx?): Promise<Profile>                       // runs checkProfileFunctions first
get(db, tables, idOrSlug): Promise<Profile>                          // excludes soft-deleted
list(db, tables, filters?, pagination?): Promise<Paginated<Profile>>
update(db, tables, idOrSlug, input, ctx?): Promise<Profile>          // re-validates, propagates name to icp_scores.profile_name
remove(db, tables, idOrSlug, ctx?): Promise<void>                    // soft delete
restore(db, tables, idOrSlug, ctx?): Promise<void>
revalidate(db, tables, idOrSlug?): Promise<Array<{ slug, is_valid, error, changed }>>
checkProfileFunctions(input): Promise<{ is_valid, error }>           // utility used by add/update
```

`filters` supports `status`.

### memberService

```ts
enroll(db, tables, input, ctx?): Promise<Member>                     // idempotent
unenroll(db, tables, input, ctx?): Promise<void>                     // also deletes the matching icp_scores row
listMembers(db, tables, profileId): Promise<Member[]>
listProfilesForOrg(db, tables, orgId): Promise<Member[]>
isEnrolled(db, tables, profileId, orgId): Promise<boolean>
```

### scoreService

```ts
upsertScore(db, tables, input, ctx?): Promise<Score>                 // one row per (profile_id, org_id)
getById(db, tables, id): Promise<Score>
getScore(db, tables, profileId, orgId): Promise<Score | null>
listScoresForOrg(db, tables, orgId): Promise<Score[]>                // sorted by score desc
listScoresForProfile(db, tables, profileId, { minScore? }): Promise<Score[]>
```

### Actions (`@openclam/icp/actions`)

Three actions are registered with the extension SDK:

| Action slug | Trigger | What it does |
|-------------|---------|--------------|
| `calculate-score` | Rule: `org.created` / `org.updated` (or manual) | For every active profile, runs `enroll_function` (auto-enroll/unenroll) and then `score_function`, upserting the result. |
| `sweep-changed-orgs` | Hourly cron (built in) | For each active profile, rescores orgs with `updated_at > last_swept_at`; pins `sweepStartedAt` *before* the org read and writes it back as the new bookmark. Pass `force_all: true` to rescore every enrolled org. |
| `recalculate-profile` | Manual (`paramsSchema: { profile_slug }`) | Rescores every enrolled org for the given profile. |

A broken score or enroll function logs a warning and is skipped — one bad
profile never poisons a full sweep.

## CLI Commands

### clam icp

```
clam --json icp profile add    <slug> <name>   --description "..." --score-function org-fit-score --enroll-function org-fit-enroll
clam --json icp profile list                   --status active --fields id,slug,name,status,is_valid
clam --json icp profile get <slugOrId>
clam --json icp profile update <slugOrId>      --name "..." --description "..." --status inactive --score-function <slug> --enroll-function <slug>
clam --json icp profile remove <slugOrId>
clam --json icp profile restore <slugOrId>
clam --json icp profile revalidate [<slugOrId>]          # omit arg to revalidate every active profile

clam --json icp enroll <profile> <orgId>
clam --json icp unenroll <profile> <orgId>
clam --json icp members <profile>

clam --json icp rescore                # incremental: only orgs changed since each profile's last_swept_at
clam --json icp rescore --all          # force-rescore every enrolled org

clam --json icp scores --profile <slugOrId>    # scores for one profile, optional --min-score
clam --json icp scores --org <orgId>           # every profile scoring this org, ranked desc
```

`profile add` takes `slug` and `name` as positional args. `profile
get/update/remove/restore/revalidate` accept either the slug or the UUID
in the `ref` arg. `rescore` is a manual invocation of the
`sweep-changed-orgs` action; the hourly cron runs the same code path.

## Events emitted

`icp.profile.created`, `icp.profile.updated`, `icp.profile.deleted`,
`icp.profile.restored`,
`icp.member.enrolled`, `icp.member.unenrolled`,
`icp.score.calculated`, `icp.score.updated`.

See `../surfaces.md` to translate these operations to MCP, HTTP, or the
typed client.
