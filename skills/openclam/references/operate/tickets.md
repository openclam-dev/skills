---
name: operate-tickets
description: >
  Support tickets in OpenClam — tickets with auto-generated numbers, SLA
  policies and trackers, teams, categories, macros, time entries, CSAT
  ratings, watchers, relationships, and tags. The `clam tickets` CLI.
metadata:
  author: openclam
  version: 0.1.0
---

# Tickets

Package: `@openclam/tickets` (daemon extension).

The ticketing surface is organised around the `ticket` entity, with several
sibling configuration entities that tickets reference:

- **Config** — `team`, `category`, `sla` policy, `hours` (business hours), `macro`
- **Interactions** — `comment`, `time` (time entry), `rate` (CSAT), `watcher`
- **Graph** — `link` (relationship), `tag`

## Tables

### tickets

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| number | INTEGER NOT NULL | Auto-incremented display number (starts at 1001) |
| subject | TEXT NOT NULL | |
| description | TEXT | |
| status | TEXT NOT NULL | `open` (default), `pending`, `in_progress`, `resolved`, `closed` |
| priority | TEXT NOT NULL | `low`, `medium` (default), `high`, `urgent` |
| channel | TEXT NOT NULL | `web` (default), `email`, `chat`, `phone`, `api` |
| category_id | TEXT | FK → ticket_categories |
| team_id | TEXT | FK → ticket_teams |
| requester_id | TEXT | Contacts person UUID |
| requester_org_id | TEXT | Contacts org UUID |
| assignee_id | TEXT | UUID |
| assignee_type | TEXT | `human` or `agent` |
| creator_id | TEXT | UUID |
| creator_type | TEXT | `human` or `agent` |
| project_id | TEXT | |
| parent_ticket_id | TEXT | Self-reference |
| merged_into_id | TEXT | Set by `merge` |
| source | TEXT NOT NULL | Free-form provenance |
| first_response_at | TEXT | Set by SLA tracking |
| resolved_at | TEXT | Set by `resolve` |
| closed_at | TEXT | Set by `close` / `merge` |
| updated_by | TEXT | |
| created_at, updated_at, deleted_at | TEXT | ISO 8601 |

Ticket subject/description are auto-enqueued for embedding on create and
re-enqueued on update when those fields change.

### ticket_comments

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| ticket_id | TEXT NOT NULL | FK → tickets (CASCADE) |
| author_id | TEXT NOT NULL | |
| author_type | TEXT NOT NULL | `human` (default) or `agent` |
| parent_id | TEXT | For reply threads |
| body | TEXT NOT NULL | |
| is_internal | BOOLEAN NOT NULL | Default false |
| channel | TEXT | |
| macro_id | TEXT | Populated when a macro created the comment |
| created_at, updated_at, deleted_at | TEXT | |

### ticket_categories

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name, slug | TEXT NOT NULL | Slug is the primary CLI handle |
| description | TEXT | |
| parent_id | TEXT | Self-reference |
| position | INTEGER NOT NULL | Default 0 |
| created_by, created_at, updated_at, deleted_at | | |

### ticket_teams

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name, slug | TEXT NOT NULL | |
| description | TEXT | |
| is_default | BOOLEAN NOT NULL | Default false |
| created_by, created_at, updated_at, deleted_at | | |

### ticket_team_members

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| team_id | TEXT NOT NULL | FK → ticket_teams (CASCADE) |
| user_id | TEXT NOT NULL | |
| role | TEXT NOT NULL | `member` (default) or `lead` |
| created_at | TEXT NOT NULL | |

### ticket_sla_policies

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name, slug | TEXT NOT NULL | |
| description | TEXT | |
| priority | TEXT NOT NULL | One of the ticket priorities |
| first_response_minutes | INTEGER | Nullable |
| resolution_minutes | INTEGER | Nullable |
| business_hours_id | TEXT | FK → ticket_business_hours |
| is_default | BOOLEAN NOT NULL | Default false |
| created_by, created_at, updated_at, deleted_at | | |

### ticket_sla_trackers

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| ticket_id | TEXT NOT NULL | FK → tickets (CASCADE) |
| sla_policy_id | TEXT NOT NULL | |
| first_response_due_at | TEXT | |
| first_response_met_at | TEXT | |
| first_response_breached | BOOLEAN NOT NULL | Default false |
| resolution_due_at | TEXT | |
| resolution_met_at | TEXT | |
| resolution_breached | BOOLEAN NOT NULL | Default false |
| created_at, updated_at | TEXT | |

### ticket_business_hours

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name, slug | TEXT NOT NULL | |
| timezone | TEXT NOT NULL | IANA tz (e.g. `America/New_York`) |
| is_default | BOOLEAN NOT NULL | Default false |
| created_by, created_at, updated_at, deleted_at | | |

### ticket_business_hour_periods

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| business_hours_id | TEXT NOT NULL | FK → ticket_business_hours (CASCADE) |
| day_of_week | INTEGER NOT NULL | 0 = Sunday … 6 = Saturday |
| start_time, end_time | TEXT NOT NULL | `HH:MM` 24-hour |

### ticket_business_hour_holidays

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| business_hours_id | TEXT NOT NULL | FK → ticket_business_hours (CASCADE) |
| date | TEXT NOT NULL | `YYYY-MM-DD` |
| name | TEXT | |

### ticket_macros

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name, slug | TEXT NOT NULL | |
| description | TEXT | |
| body | TEXT | Comment body the macro posts |
| set_status, set_priority | TEXT | Applied to the ticket when the macro runs |
| set_category_id, set_team_id | TEXT | Applied to the ticket |
| is_internal | BOOLEAN NOT NULL | Posts the comment as internal |
| position | INTEGER NOT NULL | Default 0 |
| created_by, created_at, updated_at, deleted_at | | |

### ticket_time_entries

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| ticket_id | TEXT NOT NULL | FK → tickets (CASCADE) |
| user_id | TEXT NOT NULL | |
| minutes | INTEGER NOT NULL | |
| description | TEXT | |
| billable | BOOLEAN NOT NULL | Default false |
| logged_at, created_at, updated_at, deleted_at | TEXT | |

### ticket_satisfaction_ratings

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| ticket_id | TEXT NOT NULL | FK → tickets (CASCADE) |
| requester_id | TEXT NOT NULL | |
| score | INTEGER NOT NULL | 1–5 |
| comment | TEXT | |
| created_at | TEXT NOT NULL | |

### ticket_watchers

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| ticket_id | TEXT NOT NULL | FK → tickets (CASCADE) |
| user_id | TEXT NOT NULL | |
| added_by | TEXT | |
| created_at | TEXT NOT NULL | |

### ticket_relationships

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| source_ticket_id | TEXT NOT NULL | FK → tickets (CASCADE) |
| target_ticket_id | TEXT NOT NULL | FK → tickets (CASCADE) |
| relationship | TEXT NOT NULL | `related_to`, `blocked_by`, `duplicated_by` |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |

### ticket_tags

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| ticket_id | TEXT NOT NULL | FK → tickets (CASCADE) |
| tag_id | TEXT NOT NULL | |
| created_at | TEXT NOT NULL | |

## Service Operations

All services live in `@openclam/tickets/services`. Signatures take the
daemon's `db` and `tables` handles.

### ticketService

```ts
add(db, tables, input, ctx?): Promise<Ticket>                      // assigns the next `number`
list(db, tables, filters?, pagination?, options?): Promise<Paginated<Ticket>>
count(db, tables, filters?): Promise<{ count: number }>
get(db, tables, idOrNumber): Promise<Ticket>                       // accepts UUID or numeric
update(db, tables, id, input, ctx?): Promise<Ticket>
resolve(db, tables, id, ctx?): Promise<Ticket>                     // status=resolved, resolved_at=now
close(db, tables, id, ctx?): Promise<Ticket>                       // status=closed, closed_at=now
reopen(db, tables, id, ctx?): Promise<Ticket>                      // clears resolved_at/closed_at
assign(db, tables, id, input, ctx?): Promise<Ticket>               // assignee_id/type + optional team_id
merge(db, tables, sourceId, targetId, ctx?): Promise<Ticket>       // source → closed, merged_into_id set
remove(db, tables, id, ctx?): Promise<void>                        // soft delete
restore(db, tables, id, ctx?): Promise<void>
purge(db, tables, id, ctx?): Promise<void>                         // hard delete
```

`filters` supports `search` (subject+description LIKE), `status`,
`priority`, `channel`, `assigneeId`, `assigneeType`, `teamId`, `categoryId`,
`requesterId`, `requesterOrgId`, `projectId`.

### ticketCommentService

```ts
add(db, tables, input, ctx?): Promise<TicketComment>
list(db, tables, ticketId, filters?, pagination?): Promise<Paginated<TicketComment>>
get(db, tables, id): Promise<TicketComment>
update(db, tables, id, input, ctx?): Promise<TicketComment>
remove(db, tables, id, ctx?): Promise<void>
```

### ticketCategoryService

```ts
add, list, get (by slug or id), update, remove, restore, purge
```

### ticketTeamService

```ts
add, list, get (by slug or id), update, remove
addMember(db, tables, input): Promise<TeamMember>
removeMember(db, tables, teamId, userId): Promise<void>
listMembers(db, tables, teamId): Promise<TeamMember[]>
```

### ticketSlaService

```ts
add, list, get (by slug or id), update, remove
attachToTicket(db, tables, ticketId, policyId, ctx?): Promise<SlaTracker>   // creates/updates tracker
getTracker(db, tables, ticketId): Promise<SlaTracker | null>
```

### ticketBusinessHoursService

```ts
add, list, get, update, remove
setPeriods(db, tables, hoursId, periods[]): Promise<Period[]>               // replaces all periods
addHoliday(db, tables, input): Promise<Holiday>
removeHoliday(db, tables, holidayId): Promise<void>
listHolidays(db, tables, hoursId): Promise<Holiday[]>
```

### ticketMacroService

```ts
add, list, get (by slug or id), update, remove
apply(db, tables, macroSlug, ticketId, ctx?): Promise<void>                 // posts comment + applies side effects
```

### ticketTimeEntryService

```ts
add, list, get, update, remove
total(db, tables, ticketId): Promise<{ minutes: number; billable_minutes: number }>
```

### ticketSatisfactionService

```ts
submit(db, tables, input, ctx?): Promise<SatisfactionRating>
get(db, tables, ticketId): Promise<SatisfactionRating | null>
```

### ticketWatcherService / ticketRelationshipService / ticketTagService

```ts
watchers: add, remove(ticketId, userId), list(ticketId)
relationships: link, unlink(id), list(ticketId)
tags: add, remove(ticketId, tagId), list(ticketId)
```

## CLI Commands

### clam tickets

All commands are nested under `clam tickets <group> …`. The groups are
`ticket`, `comment`, `category`, `team`, `sla`, `hours`, `macro`, `time`,
`rate`, `watcher`, `link`, and `tag`.

#### ticket

```
clam --json tickets ticket add <subject>    --input-json '{
  "description":"...","priority":"high","channel":"email",
  "category_id":"<uuid>","team_id":"<uuid>","requester_id":"<uuid>","requester_org_id":"<uuid>",
  "assignee_id":"<uuid>","assignee_type":"human","project_id":"<uuid>","source":"support-email"
}'
clam --json tickets ticket list   --input-json '{"search":"...","status":"open","priority":"high","channel":"web","assignee_id":"<uuid>","team_id":"<uuid>","category_id":"<uuid>","requester_id":"<uuid>","requester_org_id":"<uuid>","project_id":"<uuid>","cursor":"...","limit":25,"sort":"created_at","order":"desc","all":false,"fields":"id,number,subject,status"}'
clam --json tickets ticket count  --input-json '{"status":"open"}'
clam --json tickets ticket get <idOrNumber>          # accepts UUID or ticket number
clam --json tickets ticket update <id>   --input-json '{"subject":"...","priority":"urgent","team_id":"<uuid>"}'
clam --json tickets ticket assign <id> <assignee>    --input-json '{"assignee_type":"human","team_id":"<uuid>"}'
clam --json tickets ticket resolve <id>
clam --json tickets ticket close <id>
clam --json tickets ticket reopen <id>
clam --json tickets ticket merge <sourceId> <targetId>
clam --json tickets ticket remove <id>   --input-json '{"dry_run":false}'
clam --json tickets ticket restore <id>
clam --json tickets ticket purge <id>    --input-json '{"dry_run":false}'
```

#### comment

```
clam --json tickets comment add <ticketId> <body>   --input-json '{"is_internal":false,"channel":"email","author_id":"<uuid>","author_type":"human"}'
clam --json tickets comment list <ticketId>         --input-json '{"include_internal":true,"cursor":"...","limit":25,"fields":"id,body,is_internal,author_id"}'
clam --json tickets comment get <id>
clam --json tickets comment update <id>             --input-json '{"body":"..."}'
clam --json tickets comment remove <id>
```

#### category

```
clam --json tickets category add <slug>   --input-json '{"name":"Billing","description":"...","parent_id":"<uuid>"}'
clam --json tickets category list         --input-json '{"search":"...","parent_id":"<uuid>","cursor":"...","limit":25,"sort":"position","order":"asc"}'
clam --json tickets category get <slug>
clam --json tickets category update <slug> --input-json '{"name":"...","description":"...","parent_id":"<uuid>","position":3}'
clam --json tickets category remove <slug>
```

#### team

```
clam --json tickets team add <slug>   --input-json '{"name":"Support","description":"...","is_default":true}'
clam --json tickets team list         --input-json '{"search":"...","cursor":"...","limit":25}'
clam --json tickets team get <slug>
clam --json tickets team update <slug>         --input-json '{"name":"...","description":"...","is_default":false}'
clam --json tickets team remove <slug>
clam --json tickets team add-member <teamSlug> <userId>   --input-json '{"role":"lead"}'
clam --json tickets team remove-member <teamSlug> <userId>
clam --json tickets team members <teamSlug>
```

#### sla

```
clam --json tickets sla add <slug>   --input-json '{"name":"Premium","priority":"urgent","first_response_minutes":15,"resolution_minutes":240,"business_hours_id":"<uuid>","is_default":true}'
clam --json tickets sla list         --input-json '{"priority":"urgent","search":"...","cursor":"...","limit":25}'
clam --json tickets sla get <slug>
clam --json tickets sla update <slug>  --input-json '{"name":"...","priority":"high","first_response_minutes":30,"resolution_minutes":480,"business_hours_id":"<uuid>","is_default":false}'
clam --json tickets sla remove <slug>
clam --json tickets sla status <ticketId>         # returns the tracker row (or a no-tracker message)
```

#### hours (business hours)

```
clam --json tickets hours add <slug>   --input-json '{"name":"US East","timezone":"America/New_York","is_default":true}'
clam --json tickets hours list         --input-json '{"search":"...","cursor":"...","limit":25}'
clam --json tickets hours get <slug>
clam --json tickets hours update <slug>  --input-json '{"name":"...","timezone":"Europe/London","is_default":false}'
clam --json tickets hours remove <slug>
clam --json tickets hours set-periods <slug>   --input-json '{"periods":[{"day_of_week":1,"start_time":"09:00","end_time":"17:00"}]}'
clam --json tickets hours add-holiday <slug>   --input-json '{"date":"2026-12-25","name":"Christmas"}'
clam --json tickets hours remove-holiday <holidayId>
clam --json tickets hours holidays <slug>
```

#### macro

```
clam --json tickets macro add <slug>   --input-json '{"name":"Close stale","body":"We haven\u0027t heard back, closing.","set_status":"closed","set_priority":"low","set_category_id":"<uuid>","set_team_id":"<uuid>","is_internal":false}'
clam --json tickets macro list         --input-json '{"search":"...","cursor":"...","limit":25}'
clam --json tickets macro get <slug>
clam --json tickets macro update <slug> --input-json '{"name":"...","body":"...","set_status":"resolved","is_internal":true}'
clam --json tickets macro remove <slug>
clam --json tickets macro apply <macroSlug> <ticketId>
```

#### time

```
clam --json tickets time add <ticketId>   --input-json '{"minutes":30,"description":"Investigated logs","billable":true}'
clam --json tickets time list <ticketId>  --input-json '{"billable":true,"user_id":"<uuid>","cursor":"...","limit":25}'
clam --json tickets time get <id>
clam --json tickets time update <id>      --input-json '{"minutes":45,"description":"...","billable":false}'
clam --json tickets time remove <id>
clam --json tickets time total <ticketId>
```

#### rate (CSAT)

```
clam --json tickets rate rate <ticketId>   --input-json '{"score":5,"comment":"..."}'
clam --json tickets rate rating <ticketId>
```

#### watcher, link, tag

```
clam --json tickets watcher add <ticketId> <userId>
clam --json tickets watcher remove <ticketId> <userId>
clam --json tickets watcher list <ticketId>

clam --json tickets link add <ticketId> <targetId>   --input-json '{"type":"blocked_by"}'
clam --json tickets link remove <ticketId> <targetId> --input-json '{"type":"blocked_by"}'
clam --json tickets link list <ticketId>

clam --json tickets tag add <ticketId> <tagSlug>
clam --json tickets tag remove <ticketId> <tagSlug>
clam --json tickets tag list <ticketId>
```

Every mutation accepts an optional `decision` field inside `--input-json`
for decision-trace capture. Every resource exposes `--schema` on `list`
and `get` to emit its JSON Schema.

## Events emitted

`ticket.created`, `ticket.updated`, `ticket.resolved`, `ticket.closed`,
`ticket.reopened`, `ticket.assigned`, `ticket.merged`, `ticket.status_changed`,
`ticket.deleted`, `ticket.restored`, `ticket.purged`,
plus per-sub-entity events emitted by each service.

See `../surfaces.md` to translate these operations to MCP, HTTP, or the
typed client.
