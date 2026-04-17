---
name: operate-calendar
description: >
  Calendars and events — calendars, events with recurrence (RRULE), attendees,
  comments, tags, and project links. Covers the `clam calendar` CLI, service
  operations, and database schema. Depends on the `contacts` extension.
metadata:
  author: openclam
  version: 0.1.0
---

# Calendar

Package: `@openclam/calendar` (extension: `calendar`, depends on `contacts`)

---

## Tables

### calendars
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | |
| description | TEXT | |
| color | TEXT | Hex color |
| owner_id | TEXT | User/agent UUID |
| owner_type | TEXT | e.g. `human` or `agent` |
| is_default | INTEGER (boolean) NOT NULL | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### calendar_events
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| calendar_id | TEXT NOT NULL | FK to calendars.id ON DELETE CASCADE |
| title | TEXT NOT NULL | |
| description | TEXT | |
| location | TEXT | |
| start_at | TEXT NOT NULL | ISO 8601 |
| end_at | TEXT | ISO 8601 |
| all_day | INTEGER (boolean) NOT NULL | |
| status | TEXT NOT NULL | `confirmed`, `tentative`, `cancelled` |
| recurrence_rule | TEXT | RFC 5545 RRULE format |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete timestamp |

### calendar_event_attendees
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| event_id | TEXT NOT NULL | FK to calendar_events.id ON DELETE CASCADE |
| person_id | TEXT | FK to contacts persons |
| user_id | TEXT | FK to users |
| status | TEXT NOT NULL | `pending`, `accepted`, `declined`, `tentative` |
| is_organizer | INTEGER (boolean) NOT NULL | |
| created_at | TEXT NOT NULL | |

### calendar_event_comments
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| event_id | TEXT NOT NULL | FK to calendar_events.id ON DELETE CASCADE |
| author_id | TEXT NOT NULL | |
| parent_id | TEXT | Self-ref for threading |
| body | TEXT NOT NULL | |
| status | TEXT NOT NULL | `open` or `resolved` |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

### calendar_event_tags
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| event_id | TEXT NOT NULL | FK to calendar_events.id ON DELETE CASCADE |
| tag_id | TEXT NOT NULL | FK to global tags table |
| created_at | TEXT NOT NULL | |

### project_calendar_events
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| project_id | TEXT NOT NULL | |
| event_id | TEXT NOT NULL | FK to calendar_events.id ON DELETE CASCADE |
| added_at | TEXT NOT NULL | |
| added_by | TEXT | |

---

## Service Operations

### calendarService (`services/calendar.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Create calendar |
| `count(db, tables, filters?)` | Count non-deleted calendars |
| `list(db, tables, filters?, pagination?)` | filters: `search`, `ownerId`, `ownerType` |
| `get(db, tables, id)` | Excludes soft-deleted |
| `update(db, tables, id, input, ctx?)` | Partial update |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore |
| `purge(db, tables, id, ctx?)` | Hard delete |

### calendarEventService (`services/calendar-event.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Auto-sets `created_by` from ctx |
| `count(db, tables, calendarId, filters?)` | |
| `list(db, tables, calendarId, filters?, pagination?)` | filters: `search`, `status`, `from`, `to` |
| `get(db, tables, id)` | |
| `update(db, tables, id, input, ctx?)` | |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore |
| `purge(db, tables, id, ctx?)` | Hard delete |
| `importFromFile(...)` / `exportToFile(...)` | Bulk import/export |

### attendeeService (`services/attendee.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Add attendee to event |
| `list(db, tables, eventId, filters?, pagination?)` | filter: `status` |
| `get(db, tables, id)` | |
| `update(db, tables, id, input, ctx?)` | Update `status`/`is_organizer` |
| `remove(db, tables, id, ctx?)` | |

### eventCommentService (`services/event-comment.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `add(db, tables, input, ctx?)` | Status defaults to `open` |
| `list(db, tables, eventId, filters?, pagination?)` | filter: `status` |
| `get(db, tables, id)` | |
| `update(db, tables, id, input, ctx?)` | Update `body`/`status` |
| `remove(db, tables, id, ctx?)` | Hard delete |
| `resolve(db, tables, id, ctx?)` | Sets status to `resolved` |
| `getThread(db, tables, parentId)` | Child comments |

### eventTagService (`services/event-tag.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `apply(db, tables, eventId, tagLabel)` | Add tag (auto-creates via core) |
| `remove(db, tables, eventId, tagLabel)` | Remove tag |
| `getForEvent(db, tables, eventId)` | List tags for event |

### recurrenceService (`services/recurrence.ts`)
| Operation | Signature / Notes |
|-----------|-------------------|
| `isValidRecurrenceRule(rule)` | Validate an RRULE string |
| `expandOccurrences(event, from, to)` | Materialize occurrences within a window |
| `expandEventList(events, from, to)` | Expand a list of events to their occurrences |

### projectLinkService (`services/project-link.ts`)
`addEventToProject`, `removeEventFromProject`, `listEventsForProject`, `listProjectsForEvent`.

---

## CLI Commands

All list commands return `{ data: [...], nextCursor: string | null, hasMore: boolean }`.

The top-level command is `clam calendar`. Calendars themselves use the `cal` subcommand; events use `event`.

### cal (calendars)

```
clam --json calendar cal add --input-json '{"name":"Team","description":"...","color":"#4f46e5","owner_id":"...","owner_type":"human","is_default":false}'
clam --json calendar cal count --input-json '{"search":"...","owner_id":"..."}'
clam --json calendar cal list --input-json '{"search":"...","owner_id":"...","owner_type":"human","cursor":"...","limit":25}'
clam --json calendar cal get --input-json '{"id":"<calendar-id>"}'
clam --json calendar cal update --input-json '{"id":"<calendar-id>","name":"...","color":"..."}'
clam --json calendar cal remove --input-json '{"id":"<calendar-id>"}'
clam --json calendar cal restore --input-json '{"id":"<calendar-id>"}'
clam --json calendar cal purge --input-json '{"id":"<calendar-id>"}'
```

### event

```
clam --json calendar event add --input-json '{"calendar_id":"<id>","title":"...","start_at":"2026-05-01T10:00:00Z","end_at":"2026-05-01T11:00:00Z","all_day":false,"status":"confirmed","location":"...","description":"...","recurrence_rule":"FREQ=WEEKLY;BYDAY=MO"}'
clam --json calendar event count --input-json '{"calendar_id":"<id>","status":"confirmed"}'
clam --json calendar event list --input-json '{"calendar_id":"<id>","search":"...","status":"confirmed","from":"...","to":"...","cursor":"...","limit":25}'
clam --json calendar event get --input-json '{"id":"<event-id>"}'
clam --json calendar event update --input-json '{"id":"<event-id>","title":"...","status":"cancelled","start_at":"...","end_at":"..."}'
clam --json calendar event remove --input-json '{"id":"<event-id>"}'
clam --json calendar event restore --input-json '{"id":"<event-id>"}'
clam --json calendar event purge --input-json '{"id":"<event-id>"}'
clam --json calendar event import --input-json '{"file":"./events.json","format":"json"}'
clam --json calendar event export --input-json '{"file":"./events.json","format":"json"}'
```

### event attendee

```
clam --json calendar event attendee add --input-json '{"event_id":"<id>","person_id":"...","user_id":"...","status":"pending","is_organizer":false}'
clam --json calendar event attendee list --input-json '{"event_id":"<id>","status":"accepted","cursor":"...","limit":25}'
clam --json calendar event attendee get --input-json '{"id":"<attendee-id>"}'
clam --json calendar event attendee update --input-json '{"id":"<attendee-id>","status":"accepted"}'
clam --json calendar event attendee remove --input-json '{"id":"<attendee-id>"}'
```

### event comment

```
clam --json calendar event comment add --input-json '{"event_id":"<id>","body":"...","author_id":"...","parent_id":"..."}'
clam --json calendar event comment list --input-json '{"event_id":"<id>","status":"open","cursor":"...","limit":25}'
clam --json calendar event comment get --input-json '{"id":"<comment-id>"}'
clam --json calendar event comment update --input-json '{"id":"<comment-id>","body":"...","status":"resolved"}'
clam --json calendar event comment resolve --input-json '{"id":"<comment-id>"}'
clam --json calendar event comment remove --input-json '{"id":"<comment-id>"}'
clam --json calendar event comment thread --input-json '{"id":"<parent-id>"}'
```

### event tag

```
clam --json calendar event tag list --input-json '{"event_id":"<id>"}'
clam --json calendar event tag add --input-json '{"event_id":"<id>","tag":"all-hands"}'
clam --json calendar event tag remove --input-json '{"event_id":"<id>","tag":"all-hands"}'
```

---

## Input JSON Shapes

#### cal add / update
```typescript
interface CalendarAddInput {
  name: string;
  description?: string | null;
  color?: string | null;
  owner_id?: string | null;
  owner_type?: string | null;           // e.g. "human", "agent"
  is_default?: boolean;                 // default: false
}
```

#### event add
```typescript
interface CalendarEventAddInput {
  calendar_id: string;
  title: string;
  start_at: string;                     // ISO 8601
  end_at?: string | null;               // ISO 8601
  description?: string | null;
  location?: string | null;
  all_day?: boolean;                    // default: false
  status?: "confirmed" | "tentative" | "cancelled"; // default: "confirmed"
  recurrence_rule?: string | null;      // RFC 5545 RRULE
}
```

#### event list
```typescript
interface CalendarEventListInput {
  calendar_id: string;
  search?: string;
  status?: "confirmed" | "tentative" | "cancelled";
  from?: string;                        // ISO 8601
  to?: string;                          // ISO 8601
  cursor?: string;
  limit?: number;
}
```

#### attendee add / update
```typescript
interface AttendeeAddInput {
  event_id: string;
  person_id?: string | null;
  user_id?: string | null;
  status?: "pending" | "accepted" | "declined" | "tentative"; // default: "pending"
  is_organizer?: boolean;               // default: false
}
```

#### event comment add
```typescript
interface EventCommentAddInput {
  event_id: string;
  body: string;
  author_id?: string;                   // defaults to active user
  parent_id?: string | null;
}
```

#### event tag add / remove
```typescript
interface EventTagInput {
  event_id: string;
  tag: string;                          // tag slug or label
}
```

---

## Actions

| Action | Description | Params |
|--------|-------------|--------|
| `calendar:cancel-event` | Sets event status to `cancelled` | `{}` |
| `calendar:add-event-comment` | Adds a comment to the triggering event | `{ body: string }` |

---

See `../surfaces.md` to translate these operations to MCP, HTTP, or the typed client.
