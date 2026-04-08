# Tickets Reference

Package: `@openclam/tickets`
Dependencies: contacts

---

## Tables

### tickets
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| number | INTEGER UNIQUE | Auto-increment ticket number |
| subject | TEXT NOT NULL | Ticket subject |
| description | TEXT | Ticket description |
| status | TEXT | `open`, `pending`, `resolved`, `closed` |
| priority | TEXT | `low`, `medium`, `high`, `urgent` |
| channel | TEXT | Source channel |
| category_id | TEXT | FK → ticket_categories |
| team_id | TEXT | FK → ticket_teams |
| requester_id | TEXT | Person who submitted |
| requester_org_id | TEXT | Requester's org |
| assignee_id | TEXT | Assigned agent |
| assignee_type | TEXT | Agent type |
| creator_id | TEXT | Who created the ticket |
| creator_type | TEXT | Creator type |
| project_id | TEXT | Project reference |
| parent_ticket_id | TEXT | Parent ticket (for sub-tickets) |
| merged_into_id | TEXT | Target ticket if merged |
| source | TEXT | Origin source |
| first_response_at | TEXT | When first responded |
| resolved_at | TEXT | When resolved |
| closed_at | TEXT | When closed |
| updated_by | TEXT | Last updater |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete |

### ticket_comments
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| ticket_id | TEXT NOT NULL | FK → tickets (CASCADE) |
| author_id | TEXT | |
| author_type | TEXT | |
| parent_id | TEXT | Self-ref for threading |
| body | TEXT NOT NULL | |
| is_internal | INTEGER | Internal note flag |
| channel | TEXT | |
| macro_id | TEXT | If applied via macro |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete |

### ticket_categories
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | |
| slug | TEXT NOT NULL UNIQUE | |
| description | TEXT | |
| parent_id | TEXT | Hierarchical categories |
| position | INTEGER | Sort order |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete |

### ticket_teams
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | |
| slug | TEXT NOT NULL UNIQUE | |
| description | TEXT | |
| is_default | INTEGER | |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete |

### ticket_team_members
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| team_id | TEXT NOT NULL | FK → ticket_teams (CASCADE) |
| user_id | TEXT NOT NULL | |
| role | TEXT | Member role in team |
| created_at | TEXT NOT NULL | |

### ticket_sla_policies
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | |
| slug | TEXT NOT NULL UNIQUE | |
| description | TEXT | |
| priority | TEXT | Which priority this applies to |
| first_response_minutes | INTEGER | SLA target |
| resolution_minutes | INTEGER | SLA target |
| business_hours_id | TEXT | FK → ticket_business_hours |
| is_default | INTEGER | |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete |

### ticket_sla_trackers
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| ticket_id | TEXT NOT NULL UNIQUE | FK → tickets (CASCADE) |
| sla_policy_id | TEXT | FK → ticket_sla_policies |
| first_response_due_at | TEXT | |
| first_response_met_at | TEXT | |
| first_response_breached | INTEGER | |
| resolution_due_at | TEXT | |
| resolution_met_at | TEXT | |
| resolution_breached | INTEGER | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |

### ticket_business_hours
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | |
| slug | TEXT NOT NULL UNIQUE | |
| timezone | TEXT | |
| is_default | INTEGER | |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete |

### ticket_business_hour_periods
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| business_hours_id | TEXT NOT NULL | FK (CASCADE) |
| day_of_week | INTEGER | 0=Sun, 6=Sat |
| start_time | TEXT | HH:MM |
| end_time | TEXT | HH:MM |

### ticket_business_hour_holidays
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| business_hours_id | TEXT NOT NULL | FK (CASCADE) |
| date | TEXT | ISO date |
| name | TEXT | Holiday name |

### ticket_macros
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| name | TEXT NOT NULL | |
| slug | TEXT NOT NULL UNIQUE | |
| description | TEXT | |
| body | TEXT | Template body |
| set_status | TEXT | Auto-set status |
| set_priority | TEXT | Auto-set priority |
| set_category_id | TEXT | Auto-set category |
| set_team_id | TEXT | Auto-set team |
| is_internal | INTEGER | Internal macro flag |
| position | INTEGER | Sort order |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete |

### ticket_time_entries
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| ticket_id | TEXT NOT NULL | FK → tickets (CASCADE) |
| user_id | TEXT | |
| minutes | INTEGER | Time logged |
| description | TEXT | |
| billable | INTEGER | 0 or 1 |
| logged_at | TEXT | |
| created_at | TEXT NOT NULL | |
| updated_at | TEXT NOT NULL | |
| deleted_at | TEXT | Soft delete |

### ticket_satisfaction_ratings
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| ticket_id | TEXT NOT NULL UNIQUE | FK → tickets (CASCADE) |
| requester_id | TEXT | |
| score | INTEGER | Rating score |
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

### ticket_tags
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| ticket_id | TEXT NOT NULL | FK → tickets (CASCADE) |
| tag_id | TEXT NOT NULL | |
| created_at | TEXT NOT NULL | |

### ticket_relationships
| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| source_ticket_id | TEXT NOT NULL | FK → tickets (CASCADE) |
| target_ticket_id | TEXT NOT NULL | FK → tickets (CASCADE) |
| relationship | TEXT | e.g. `related`, `blocks`, `duplicates` |
| created_by | TEXT | |
| created_at | TEXT NOT NULL | |

---

## Service Operations

### ticketService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create ticket |
| `list(db, tables, filters?, pagination?)` | List; filters: search, status, priority, assignee_id, team_id, category_id |
| `get(db, tables, id)` | Get by ID |
| `update(db, tables, id, input, ctx?)` | Update fields |
| `remove(db, tables, id, ctx?)` | Soft delete |
| `restore(db, tables, id, ctx?)` | Restore |
| `purge(db, tables, id, ctx?)` | Hard delete |
| `resolve(db, tables, id, ctx?)` | Set status=resolved |
| `close(db, tables, id, ctx?)` | Set status=closed |
| `reopen(db, tables, id, ctx?)` | Set status=open |
| `assign(db, tables, id, assigneeId, ctx?)` | Assign to agent |
| `merge(db, tables, id, targetId, ctx?)` | Merge into another ticket |

### ticketCommentService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Add comment |
| `list(db, tables, ticketId, pagination?)` | List comments |
| `get(db, tables, id)` | Get by ID |
| `update(db, tables, id, input, ctx?)` | Update body |
| `remove(db, tables, id, ctx?)` | Soft delete |

### ticketCategoryService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create category |
| `list(db, tables, filters?, pagination?)` | List categories |
| `get(db, tables, idOrSlug)` | Get by ID or slug |
| `update(db, tables, idOrSlug, input, ctx?)` | Update |
| `remove(db, tables, idOrSlug, ctx?)` | Soft delete |
| `restore(db, tables, idOrSlug, ctx?)` | Restore |
| `purge(db, tables, idOrSlug, ctx?)` | Hard delete |

### ticketTeamService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create team |
| `list(db, tables, filters?, pagination?)` | List teams |
| `get(db, tables, idOrSlug)` | Get by ID or slug |
| `update(db, tables, idOrSlug, input, ctx?)` | Update |
| `remove(db, tables, idOrSlug, ctx?)` | Soft delete |
| `addMember(db, tables, teamId, userId, role?)` | Add member |
| `removeMember(db, tables, teamId, userId)` | Remove member |
| `listMembers(db, tables, teamId)` | List members |

### ticketSlaService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create SLA policy |
| `list(db, tables, pagination?)` | List policies |
| `get(db, tables, idOrSlug)` | Get by ID or slug |
| `update(db, tables, idOrSlug, input, ctx?)` | Update |
| `remove(db, tables, idOrSlug, ctx?)` | Soft delete |
| `attachToTicket(db, tables, ticketId, policyId)` | Attach SLA to ticket |
| `getTracker(db, tables, ticketId)` | Get SLA tracker state |

### ticketMacroService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create macro |
| `list(db, tables, pagination?)` | List macros |
| `get(db, tables, idOrSlug)` | Get by ID or slug |
| `update(db, tables, idOrSlug, input, ctx?)` | Update |
| `remove(db, tables, idOrSlug, ctx?)` | Soft delete |
| `apply(db, tables, macroSlug, ticketId, ctx?)` | Apply macro to ticket |

### ticketBusinessHoursService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Create business hours |
| `list(db, tables, pagination?)` | List |
| `get(db, tables, idOrSlug)` | Get by ID or slug |
| `update(db, tables, idOrSlug, input, ctx?)` | Update |
| `remove(db, tables, idOrSlug, ctx?)` | Soft delete |
| `setPeriods(db, tables, id, periods[])` | Set working hour periods |
| `addHoliday(db, tables, id, holiday)` | Add holiday |
| `removeHoliday(db, tables, id, holidayId)` | Remove holiday |
| `listHolidays(db, tables, id)` | List holidays |

### ticketTimeEntryService
| Operation | Description |
|-----------|-------------|
| `add(db, tables, input, ctx?)` | Log time |
| `list(db, tables, ticketId, pagination?)` | List entries |
| `get(db, tables, id)` | Get by ID |
| `update(db, tables, id, input, ctx?)` | Update |
| `remove(db, tables, id, ctx?)` | Soft delete |

### ticketSatisfactionService
| Operation | Description |
|-----------|-------------|
| `submit(db, tables, ticketId, input)` | Submit rating |
| `get(db, tables, ticketId)` | Get rating |

### ticketRelationshipService, ticketWatcherService, ticketTagService
Standard add/list/remove operations for junction tables.

---

## API Routes

### Tickets
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/tickets` | Create ticket |
| GET | `/api/tickets` | List tickets |
| GET | `/api/tickets/:id` | Get ticket |
| PATCH | `/api/tickets/:id` | Update ticket |
| DELETE | `/api/tickets/:id` | Soft delete |
| POST | `/api/tickets/:id/resolve` | Resolve |
| POST | `/api/tickets/:id/close` | Close |
| POST | `/api/tickets/:id/reopen` | Reopen |
| POST | `/api/tickets/:id/assign` | Assign |
| POST | `/api/tickets/:id/merge` | Merge |

### Comments
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/tickets/:id/comments` | Add comment |
| GET | `/api/tickets/:id/comments` | List comments |
| GET | `/api/tickets/comments/:commentId` | Get comment |
| PATCH | `/api/tickets/comments/:commentId` | Update |
| DELETE | `/api/tickets/comments/:commentId` | Delete |

### Categories
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/tickets/categories` | Create |
| GET | `/api/tickets/categories` | List |
| GET | `/api/tickets/categories/:slug` | Get |
| PATCH | `/api/tickets/categories/:slug` | Update |
| DELETE | `/api/tickets/categories/:slug` | Delete |

### Teams
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/tickets/teams` | Create |
| GET | `/api/tickets/teams` | List |
| GET | `/api/tickets/teams/:slug` | Get |
| PATCH | `/api/tickets/teams/:slug` | Update |
| DELETE | `/api/tickets/teams/:slug` | Delete |
| POST | `/api/tickets/teams/:slug/members` | Add member |
| GET | `/api/tickets/teams/:slug/members` | List members |
| DELETE | `/api/tickets/teams/:slug/members/:userId` | Remove member |

### SLA
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/tickets/sla` | Create policy |
| GET | `/api/tickets/sla` | List policies |
| GET | `/api/tickets/sla/:slug` | Get policy |
| PATCH | `/api/tickets/sla/:slug` | Update |
| DELETE | `/api/tickets/sla/:slug` | Delete |
| GET | `/api/tickets/:id/sla` | Get ticket SLA tracker |

### Business Hours
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/tickets/business-hours` | Create |
| GET | `/api/tickets/business-hours` | List |
| GET | `/api/tickets/business-hours/:slug` | Get |
| PATCH | `/api/tickets/business-hours/:slug` | Update |
| DELETE | `/api/tickets/business-hours/:slug` | Delete |

### Macros
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/tickets/macros` | Create |
| GET | `/api/tickets/macros` | List |
| GET | `/api/tickets/macros/:slug` | Get |
| PATCH | `/api/tickets/macros/:slug` | Update |
| DELETE | `/api/tickets/macros/:slug` | Delete |
| POST | `/api/tickets/macros/:slug/apply/:ticketId` | Apply to ticket |

### Time, Ratings, Watchers, Relationships, Tags
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/tickets/:id/time` | Log time |
| GET | `/api/tickets/:id/time` | List time entries |
| POST | `/api/tickets/:id/rate` | Submit satisfaction rating |
| GET | `/api/tickets/:id/rating` | Get rating |
| POST | `/api/tickets/:id/watchers` | Add watcher |
| GET | `/api/tickets/:id/watchers` | List watchers |
| DELETE | `/api/tickets/:id/watchers/:userId` | Remove watcher |
| POST | `/api/tickets/:id/links` | Create relationship |
| GET | `/api/tickets/:id/links` | List relationships |
| DELETE | `/api/tickets/:id/links/:linkId` | Delete relationship |
| POST | `/api/tickets/:id/tags` | Add tag |
| GET | `/api/tickets/:id/tags` | List tags |
| DELETE | `/api/tickets/:id/tags/:tagSlug` | Remove tag |

---

## Events Emitted

`ticket.created`, `ticket.updated`, `ticket.resolved`, `ticket.status_changed`, `ticket.closed`, `ticket.reopened`, `ticket.assigned`, `ticket.merged`, `ticket.deleted`, `ticket.restored`, `ticket.purged`
`ticket.comment.created`, `ticket.comment.updated`, `ticket.comment.deleted`
`ticket.category.created`, `ticket.category.updated`, `ticket.category.deleted`, `ticket.category.restored`, `ticket.category.purged`
`ticket.team.created`, `ticket.team.updated`, `ticket.team.deleted`, `ticket.team.member_added`, `ticket.team.member_removed`
`ticket.sla_policy.created`, `ticket.sla_policy.updated`, `ticket.sla_policy.deleted`, `ticket.sla.attached`
`ticket.macro.created`, `ticket.macro.updated`, `ticket.macro.deleted`, `ticket.macro.applied`
`ticket.time_entry.created`, `ticket.time_entry.updated`, `ticket.time_entry.deleted`
`ticket.satisfaction.submitted`
`ticket.watcher.added`, `ticket.watcher.removed`
`ticket.tag.added`, `ticket.tag.removed`
`ticket.relationship.created`, `ticket.relationship.deleted`
`ticket.business_hours.created`, `ticket.business_hours.updated`, `ticket.business_hours.deleted`, `ticket.business_hours.periods_set`, `ticket.business_hours.holiday_added`, `ticket.business_hours.holiday_removed`

## Permissions

`ticket.create`, `ticket.read`, `ticket.update`, `ticket.delete`, `ticket.assign`, `ticket.resolve`, `ticket.close`, `ticket.merge`
`ticket_comment.create`, `ticket_comment.read`, `ticket_comment.update`, `ticket_comment.delete`
`ticket_category.create`, `ticket_category.read`, `ticket_category.update`, `ticket_category.delete`
`ticket_team.create`, `ticket_team.read`, `ticket_team.update`, `ticket_team.delete`, `ticket_team.assign`
`ticket_sla.create`, `ticket_sla.read`, `ticket_sla.update`, `ticket_sla.delete`
`ticket_macro.create`, `ticket_macro.read`, `ticket_macro.update`, `ticket_macro.delete`
`ticket_time.create`, `ticket_time.read`, `ticket_time.update`, `ticket_time.delete`
`ticket_business_hours.create`, `ticket_business_hours.read`, `ticket_business_hours.update`, `ticket_business_hours.delete`
