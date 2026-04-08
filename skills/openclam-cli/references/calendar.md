# Calendar Reference

Package: `@openclam/calendar`
Dependencies: `contacts`

---

## Tables

### calendars
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | Display name |
| description | TEXT | |
| color | TEXT | Hex color |
| owner_id | TEXT | User UUID |
| owner_type | TEXT | `human` or `agent` |
| is_default | INTEGER (boolean) | |
| created_at | TEXT NOT NULL | ISO 8601 |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete |

### calendar_events
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| calendar_id | TEXT NOT NULL | FK → calendars (CASCADE) |
| title | TEXT NOT NULL | |
| description | TEXT | |
| location | TEXT | |
| start_at | TEXT NOT NULL | ISO 8601 |
| end_at | TEXT | |
| all_day | INTEGER (boolean) | |
| status | TEXT | `confirmed`, `tentative`, `cancelled` |
| recurrence_rule | TEXT | RRULE format |
| created_by | TEXT | User UUID |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete |

### calendar_event_attendees
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| event_id | TEXT NOT NULL | FK → calendar_events (CASCADE) |
| person_id | TEXT | FK reference to persons |
| user_id | TEXT | FK reference to users |
| status | TEXT | `pending`, `accepted`, `declined`, `tentative` |
| is_organizer | INTEGER (boolean) | |
| created_at | TEXT NOT NULL | |

### calendar_event_comments
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| event_id | TEXT NOT NULL | FK → calendar_events (CASCADE) |
| author_id | TEXT NOT NULL | |
| parent_id | TEXT | For threaded comments |
| body | TEXT NOT NULL | |
| status | TEXT | `open`, `resolved` |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

### calendar_event_tags
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| event_id | TEXT NOT NULL | FK → calendar_events (CASCADE) |
| tag_id | TEXT NOT NULL | FK reference to core tags |
| created_at | TEXT NOT NULL | |

### project_calendar_events
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| project_id | TEXT NOT NULL | |
| event_id | TEXT NOT NULL | FK → calendar_events (CASCADE) |
| added_at | TEXT | |
| added_by | TEXT | |

---

## Service Operations

### calendarService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create calendar |
| `list(db, tables, filters?, pagination?)` | List; filters: `search`, `ownerId`, `ownerType` |
| `get(db, tables, id)` | Get by ID |
| `update(db, tables, id, input, ctx?)` | Update fields |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore |
| `purge(db, tables, id, ctx?)` | Hard delete |

### calendarEventService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create event; auto-sets `created_by` from ctx |
| `list(db, tables, calendarId, filters?, pagination?)` | List; filters: `search`, `status`, `from`, `to` |
| `get(db, tables, id)` | Get by ID |
| `update(db, tables, id, input, ctx?)` | Update fields |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore |
| `purge(db, tables, id, ctx?)` | Hard delete |

### attendeeService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Add attendee to event |
| `list(db, tables, eventId, filters?, pagination?)` | List; filter: `status` |
| `get(db, tables, id)` | Get by ID |
| `update(db, tables, id, input, ctx?)` | Update status/is_organizer |
| `remove(db, tables, id, ctx?)` | Remove attendee |

### eventCommentService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create comment on event |
| `list(db, tables, eventId, filters?, pagination?)` | List; filter: `status` |
| `get(db, tables, id)` | Get by ID |
| `update(db, tables, id, input, ctx?)` | Update body/status |
| `remove(db, tables, id, ctx?)` | Delete comment |
| `resolve(db, tables, id, ctx?)` | Mark as resolved |
| `getThread(db, tables, parentId)` | Get child comments (replies) |

### eventTagService
| Operation | Description |
|-----------|-------------|
| `apply(db, tables, eventId, tagLabel)` | Add tag (auto-creates via core) |
| `remove(db, tables, eventId, tagLabel)` | Remove tag |
| `getForEvent(db, tables, eventId)` | List tags for event |

### projectLinkService
| Operation | Description |
|-----------|-------------|
| `addEventToProject(db, tables, projectId, eventId, ctx?)` | Link event to project |
| `removeEventFromProject(db, tables, projectId, eventId)` | Unlink |
| `listEventsForProject(db, tables, projectId)` | Events for project |
| `listProjectsForEvent(db, tables, eventId)` | Projects for event |

---

## CLI Commands

### calendar

```
clam --json calendar cal add --input-json '{"name":"...","description":"...","color":"...","owner_id":"...","owner_type":"human"}'
clam --json calendar cal list --input-json '{"search":"...","owner_id":"...","cursor":"...","limit":25}'
clam --json calendar cal get <id>
clam --json calendar cal update --input-json '{"id":"<id>","name":"...","color":"..."}'
clam --json calendar cal remove <id>
clam --json calendar cal restore <id>
clam --json calendar cal purge <id>
```

### cal-event

```
clam --json calendar event add --input-json '{"calendar_id":"...","title":"...","start_at":"...","end_at":"...","all_day":false,"status":"confirmed","recurrence_rule":"..."}'
clam --json calendar event list --input-json '{"calendar_id":"...","search":"...","status":"confirmed","from":"...","to":"...","cursor":"...","limit":25}'
clam --json calendar event get <id>
clam --json calendar event update --input-json '{"id":"<id>","title":"...","status":"..."}'
clam --json calendar event remove <id>
clam --json calendar event restore <id>
clam --json calendar event purge <id>
```

### cal-event attendee

```
clam --json calendar event attendee add --input-json '{"event_id":"...","person_id":"...","status":"pending","is_organizer":false}'
clam --json calendar event attendee list --input-json '{"event_id":"...","status":"...","cursor":"...","limit":25}'
clam --json calendar event attendee get <id>
clam --json calendar event attendee update --input-json '{"id":"<id>","status":"accepted"}'
clam --json calendar event attendee remove <id>
```

### cal-event comment

```
clam --json calendar event comment add --input-json '{"event_id":"...","body":"...","author_id":"...","parent_id":"..."}'
clam --json calendar event comment list --input-json '{"event_id":"...","status":"open","cursor":"...","limit":25}'
clam --json calendar event comment get <id>
clam --json calendar event comment update --input-json '{"id":"<id>","body":"..."}'
clam --json calendar event comment resolve <id>
clam --json calendar event comment remove <id>
clam --json calendar event comment thread <id>
```

### cal-event tag

```
clam --json calendar event tag list --input-json '{"event_id":"..."}'
clam --json calendar event tag add --input-json '{"event_id":"...","tag":"<tag-label>"}'
clam --json calendar event tag remove --input-json '{"event_id":"...","tag":"<tag-label>"}'
```

---

## Input JSON Shapes

#### calendar add
```typescript
interface CalendarAddInput {
  name: string;                    // required
  description?: string | null;
  color?: string | null;
  owner_id?: string | null;
  owner_type?: "human" | "agent" | null;
  is_default?: boolean;
}
```

#### cal-event add
```typescript
interface CalendarEventAddInput {
  calendar_id: string;             // required (UUID)
  title: string;                   // required
  start_at: string;                // required (ISO 8601)
  description?: string | null;
  location?: string | null;
  end_at?: string | null;
  all_day?: boolean;
  status?: "confirmed" | "tentative" | "cancelled";
  recurrence_rule?: string | null; // RRULE format
}
```

#### attendee add
```typescript
interface AttendeeAddInput {
  event_id: string;                // required (UUID)
  person_id?: string | null;       // reference to contacts person
  user_id?: string | null;         // reference to users
  status?: "pending" | "accepted" | "declined" | "tentative";
  is_organizer?: boolean;
}
```

---

## API Routes

### Calendars
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/calendars` | List calendars |
| POST | `/api/calendars` | Create calendar |
| GET | `/api/calendars/:id` | Get calendar |
| PUT | `/api/calendars/:id` | Update calendar |
| DELETE | `/api/calendars/:id` | Soft delete |
| PUT | `/api/calendars/:id/restore` | Restore |
| DELETE | `/api/calendars/:id/purge` | Hard delete |

### Calendar Events
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/calendars/:calendarId/events` | List events for calendar |
| POST | `/api/calendars/:calendarId/events` | Create event |
| GET | `/api/calendar-events/:id` | Get event |
| PUT | `/api/calendar-events/:id` | Update event |
| DELETE | `/api/calendar-events/:id` | Soft delete |
| PUT | `/api/calendar-events/:id/restore` | Restore |
| DELETE | `/api/calendar-events/:id/purge` | Hard delete |

### Attendees
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/calendar-events/:eventId/attendees` | List attendees |
| POST | `/api/calendar-events/:eventId/attendees` | Add attendee |
| GET | `/api/calendar-event-attendees/:id` | Get attendee |
| PUT | `/api/calendar-event-attendees/:id` | Update attendee |
| DELETE | `/api/calendar-event-attendees/:id` | Remove attendee |

### Comments
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/calendar-events/:eventId/comments` | List comments |
| POST | `/api/calendar-events/:eventId/comments` | Create comment |
| GET | `/api/calendar-event-comments/:id` | Get comment |
| PUT | `/api/calendar-event-comments/:id` | Update comment |
| DELETE | `/api/calendar-event-comments/:id` | Delete comment |
| PUT | `/api/calendar-event-comments/:id/resolve` | Resolve comment |
| GET | `/api/calendar-event-comments/:id/thread` | Get thread |

### Tags
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/calendar-events/:eventId/tags` | List tags |
| POST | `/api/calendar-events/:eventId/tags` | Add tag; body: `{ tag }` |
| DELETE | `/api/calendar-events/:eventId/tags/:tag` | Remove tag |

### Project Links
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/calendar-events/:id/projects` | Projects for event |
| GET | `/api/projects/:id/calendar-events` | Events for project |
| POST | `/api/projects/:id/calendar-events` | Link event; body: `{ eventId }` |
| DELETE | `/api/projects/:id/calendar-events/:eventId` | Unlink |

---

## Events Emitted

`calendar.created`, `calendar.updated`, `calendar.deleted`, `calendar.restored`
`calendar.event.created`, `calendar.event.updated`, `calendar.event.deleted`, `calendar.event.restored`
`calendar.event.attendee.added`, `calendar.event.attendee.updated`, `calendar.event.attendee.removed`
`calendar.event.comment.created`, `calendar.event.comment.updated`, `calendar.event.comment.deleted`, `calendar.event.comment.resolved`

## Actions

| Action | Description | Params |
|--------|-------------|--------|
| `calendar:cancel-event` | Sets event status to `cancelled` | `{}` |
| `calendar:add-event-comment` | Adds comment to triggering event | `{ body: string }` |

## Permissions

`calendar.create`, `calendar.read`, `calendar.update`, `calendar.delete`
`calendar_event.create`, `calendar_event.read`, `calendar_event.update`, `calendar_event.delete`
`calendar_event_comment.create`, `calendar_event_comment.read`, `calendar_event_comment.update`, `calendar_event_comment.delete`
