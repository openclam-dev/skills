# Execution Layer Reference

The execution layer reacts to state changes, runs workflows, dispatches actions, schedules jobs, and delivers webhooks. It consists of six packages that form a connected pipeline:

```
Domain mutations → Events (outbox) → Rules (evaluate) → Actions (dispatch)
                                                              ↓
                                                        Workflows (orchestrate)
                                                              ↓
                                                        Cron (schedule)
                                                              ↓
                                                        Webhooks (deliver)
```

---

## 1. Events (packages/events)

Central event emission and processing with outbox pattern.

### How it works

1. Domain services call `emitEvent()` after every successful mutation
2. Events are stored as `pending` in the `events` table
3. Processing is triggered explicitly (`clam event process` or `POST /api/events/process`)
4. During processing: run inline handlers → match rules → execute actions → call delivery hooks

### Key Functions

| Function | Description |
|----------|-------------|
| `emitEvent(db, tables, opts)` | Convenience emitter with `sor/` prefix |
| `emit(db, tables, input)` | Low-level direct emission |
| `listPending(db, tables, opts?)` | Get pending/failed events |
| `processEvents(db, tables, opts?)` | Main orchestrator — handlers + rules + hooks |
| `on(pattern, handler)` | Register inline event handler |
| `off(pattern, handler?)` | Unregister handler |
| `registerDeliveryHook(hook)` | External systems hook into processing (used by webhooks) |
| `matchPattern(pattern, eventType)` | Pattern matching with wildcards |

### Event Schema (events table)

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT PK | UUID |
| source | TEXT NOT NULL | e.g. `sor/contacts`, `sor/deals` |
| event_type | TEXT NOT NULL | e.g. `person.created`, `deal.updated` |
| entity_type | TEXT NOT NULL | e.g. `person`, `deal` |
| entity_id | TEXT NOT NULL | UUID of affected entity |
| payload | TEXT | JSON |
| user_id | TEXT | Who caused the event |
| status | TEXT NOT NULL | `pending`, `processing`, `completed`, `failed` |
| created_at | TEXT NOT NULL | |
| processed_at | TEXT | Set when completed |
| failure_reason | TEXT | Error message if failed |
| attempts | INTEGER NOT NULL | Default 0 |

### Pattern Matching

| Pattern | Matches |
|---------|---------|
| `"person.created"` | Exact match |
| `"person.*"` | Any `person.X` event |
| `"*.created"` | Any `X.created` event |
| `"*"` | All events |

### Emission Convention

```typescript
await eventService.emitEvent(db, tables, {
  source: 'contacts',           // auto-prefixed to "sor/contacts"
  eventType: 'person.created',
  entityType: 'person',
  entityId: parsed.id,
  userId: ctx?.userId,
  payload: { stage: 'qualified' },  // optional
});
```

### Inline Handlers

For lightweight reactive logic without database rules:

```typescript
eventService.on('person.created', async (event, db, tables) => { ... });
eventService.off('person.created', handler);
```

Handlers run during `processEvents()`, not at emit time.

---

## 2. Rules (packages/rules)

Define event-based automation rules with conditions and multi-action execution.

### How it works

1. Rules are stored in the database with an `event_pattern` and optional `conditions`
2. During event processing, active rules are matched against each event
3. Matching rules execute their `rule_actions` in `execution_order`
4. Each action is looked up from the global action registry and executed

### Tables

**rules** — id, slug (unique), name, event_pattern, conditions (JSON), priority, is_active, created_at, updated_at, deleted_at

**rule_actions** — id, rule_id (FK→rules), extension, action, params_json, execution_order, created_at

### Key Functions

| Function | Description |
|----------|-------------|
| `add(db, tables, input)` | Create rule with multiple ordered actions |
| `list(db, tables, filters?, pagination?)` | List with search |
| `get(db, tables, idOrSlug)` | Get by ID or slug (includes actions) |
| `update(db, tables, slug, input)` | Update rule and replace actions |
| `enable(db, tables, slug)` | Toggle is_active on |
| `disable(db, tables, slug)` | Toggle is_active off |
| `remove/restore/purge` | Soft/hard delete |

### Conditions

Conditions are a JSON object that matches against event payload fields:

```json
{ "payload.stage": "qualified", "payload.value": 50000 }
```

If conditions is null, the rule matches on event_pattern alone.

### CLI Commands

```
clam --json rule add --input-json '{"slug":"...","name":"...","event_pattern":"person.created","actions":[{"extension":"contacts","action":"add-note","params_json":"{\"body\":\"Welcome!\"}"}]}'
clam --json rule list --input-json '{"search":"...","cursor":"...","limit":25}'
clam --json rule get <slug>
clam --json rule update --input-json '{"slug":"...","event_pattern":"...","actions":[...]}'
clam --json rule enable <slug>
clam --json rule disable <slug>
clam --json rule remove <slug>
clam --json rule restore <slug>
clam --json rule purge <slug>
clam --json rule actions                   # List all available actions
```

---

## 3. Actions (packages/actions)

Central registry for executable actions triggered by rules and workflows.

### How it works

1. Extensions call `registerActions(extensionName, actions)` at bootstrap
2. Rules and workflows look up actions via `lookupAction(extension, action)`
3. The action's `execute()` receives the triggering event, validated params, db, and tables

### Key Functions

| Function | Description |
|----------|-------------|
| `registerActions(extensionName, actions)` | Register actions from an extension |
| `lookupAction(extension, action)` | Get action by `"extension:action"` format |
| `listAllActions()` | List all available actions with descriptions |

### Action Interface

```typescript
interface ExtensionAction {
  description: string;
  paramsSchema: z.ZodType<any>;
  execute: (event, params, db, tables, ctx?) => Promise<void>;
}
```

### Built-in Actions

| Action | Description | Params |
|--------|-------------|--------|
| `core:tag-entity` | Apply tag to event's entity | `{ tag: string }` |
| `core:create-log` | Create log entry for entity | `{ message?, type? }` |
| `contacts:add-note` | Add note to event's entity | `{ title?, body: string }` |
| `contacts:update-deal-stage` | Move deal to new stage | `{ stage: string }` |
| `contacts:assign-to-org` | Assign person to org | `{ org_id: string }` |
| `calendar:cancel-event` | Cancel a calendar event | `{}` |
| `calendar:add-event-comment` | Comment on calendar event | `{ body: string }` |

---

## 4. Workflows (packages/workflows)

Multi-step workflow definitions with deterministic replay, event waiting, and child workflow support.

### How it works

1. Workflows are defined with `defineWorkflow()` specifying a trigger and handler
2. Triggers can be: event patterns, cron expressions, or manual
3. The handler uses `step.*` primitives for durability — each step is memoized
4. On retry, the executor replays completed steps from cache and resumes at the suspension point

### Workflow Definition

```typescript
const myWorkflow = defineWorkflow({
  spec: {
    name: 'onboard-contact',
    trigger: { event: 'person.created' },
    input: z.object({ personId: z.string() }),
  },
  handler: async (ctx) => {
    const person = await ctx.step.run('fetch-person', async () => {
      return ctx.services.personService.get(db, tables, ctx.input.personId);
    });

    await ctx.step.sleep('wait-1h', '1h');

    await ctx.step.run('send-welcome', async () => {
      // send welcome email
    });

    const reply = await ctx.step.waitForEvent('wait-reply', {
      event: 'message.replied',
      filter: { 'payload.person_id': ctx.input.personId },
      timeout: '7d',
    });

    if (reply) {
      await ctx.step.run('mark-engaged', async () => { /* ... */ });
    }
  },
});
```

### Step Primitives

| Primitive | Description |
|-----------|-------------|
| `step.run(name, fn, opts?)` | Execute and memoize a function |
| `step.sleep(name, duration)` | Pause for a duration (`5s`, `2m`, `1h`, `3d`) |
| `step.sleepUntil(name, timestamp)` | Pause until a specific time |
| `step.waitForEvent(name, opts)` | Suspend until a matching event arrives or timeout |
| `step.sendEvent(name, events)` | Emit events from within a workflow |
| `step.runWorkflow(name, workflowName, opts?)` | Start a child workflow and wait for completion |

### Triggers

```typescript
// Event trigger
{ event: 'person.created', filter?: { 'payload.source': 'import' } }

// Cron trigger
{ cron: '0 9 * * MON', timezone?: 'America/New_York' }

// Manual trigger
{ manual: true }
```

### Run Status Lifecycle

`pending` → `running` → `sleeping` / `waiting_for_event` / `waiting_for_child` → `running` → `completed` / `failed` / `canceled`

### Tables

**workflow_definitions** — id, name (unique), description, file_path, triggers_json, is_valid, error, is_active, loaded_at, created_at, updated_at

**workflow_runs** — id, workflow_name, status, input_json, output_json, error, trigger_type, trigger_event_id, parent_run_id, parent_step_name, worker_id, available_at, consecutive_errors, attempt_number, max_attempts, started_at, finished_at, created_at

**workflow_step_attempts** — id, workflow_run_id (FK→runs), name, ordinal, step_type, status, output_json, error, started_at, finished_at, duration_ms

**workflow_event_waiters** — id, workflow_run_id (FK→runs), step_name, event_pattern, match_json, timeout_at, created_at

### Executor Tick Loop

1. Expire timed-out event waiters
2. Resolve completed child workflows
3. Claim claimable runs (pending/sleeping/waiting, available_at <= now)
4. Execute runs in parallel (max concurrent)

Retry uses exponential backoff up to `max_attempts`.

---

## 5. Cron (packages/cron)

Job scheduling and execution with cron expressions.

### How it works

1. Jobs are registered with a cron schedule and handler name
2. The daemon's cron scheduler calls `tick()` periodically
3. `tick()` finds due jobs (`next_run_at <= now`), locks them, executes handlers, updates `next_run_at`
4. Failed jobs use backoff: 30s → 1m → 5m → 15m → 60m (capped)

### Tables

**cron_jobs** — id, slug (unique), name, schedule, handler, params_json, is_active, consecutive_errors, running_since, timezone, last_run_at, next_run_at, created_at, updated_at, deleted_at

**cron_runs** — id, job_id (FK→cron_jobs), status, started_at, finished_at, duration_ms, error, created_at

### Key Functions

| Function | Description |
|----------|-------------|
| `cronService.addJob()` | Create a cron job |
| `cronService.listJobs()` | List jobs with pagination |
| `cronService.getJobBySlug()` | Get by slug |
| `cronService.updateJob()` | Update schedule/handler/params |
| `cronService.removeJob()` | Delete job |
| `CronScheduler.tick()` | Execute due jobs |
| `CronScheduler.registerHandler()` | Register execution handler by name |
| `getNextRun(schedule)` | Calculate next execution time |
| `isValidCron(expr)` | Validate cron expression |

Supports 5-field (minute-level) and 6-field (second-level) cron expressions.

### CLI Commands

```
clam --json cron job add --input-json '{"slug":"...","name":"...","schedule":"0 9 * * *","handler":"...","params_json":"..."}'
clam --json cron job list --input-json '{"cursor":"...","limit":25}'
clam --json cron job get <slug>
clam --json cron job update --input-json '{"slug":"...","schedule":"...","handler":"..."}'
clam --json cron job enable <slug>
clam --json cron job disable <slug>
clam --json cron job remove <slug>
clam --json cron job trigger <slug>
```

### API Routes

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/cron/jobs` | List jobs |
| POST | `/api/cron/jobs` | Create job |
| GET | `/api/cron/jobs/:slug` | Get job |
| PUT | `/api/cron/jobs/:slug` | Update |
| DELETE | `/api/cron/jobs/:slug` | Delete |
| PUT | `/api/cron/jobs/:slug/enable` | Enable |
| PUT | `/api/cron/jobs/:slug/disable` | Disable |
| POST | `/api/cron/jobs/:slug/trigger` | Trigger immediately |
| GET | `/api/cron/jobs/:slug/runs` | List runs for job |

---

## 6. Webhooks (packages/webhooks)

Subscribe to events and deliver them via HTTP POST with retries.

### How it works

1. Subscribers are registered with an `event_pattern` and target `url`
2. When events are processed, `queueDeliveries()` is called as a delivery hook
3. Active subscribers matching the event get a `pending` delivery queued
4. `processDeliveries()` sends HTTP POST with HMAC-SHA256 signature
5. Failed deliveries retry with exponential backoff (2^attempts * 5s, capped at 5min)

### Tables

**event_subscribers** — id, name, url, event_pattern, conditions (JSON), secret, headers_json, is_active, retry_policy (`exponential`/`linear`/`fixed`), max_retries (0–20, default 5), created_at, updated_at, deleted_at

**event_deliveries** — id, subscriber_id, event_id, status (`pending`/`delivered`/`failed`/`expired`), attempts, last_attempt_at, next_retry_at, status_code, response_body, failure_reason, created_at

### Delivery Payload

```json
{
  "delivery_id": "uuid",
  "subscriber_id": "uuid",
  "event": {
    "id": "...", "source": "...", "event_type": "...",
    "entity_type": "...", "entity_id": "...",
    "payload": { ... },
    "user_id": "...", "created_at": "..."
  },
  "timestamp": "ISO-8601"
}
```

### Delivery Headers

```
Content-Type: application/json
X-Clam-Signature: sha256=<hmac>
X-Clam-Event-Type: <event_type>
X-Clam-Delivery-Id: <delivery_id>
```

Plus any custom headers from `headers_json`.

### Key Functions

| Function | Description |
|----------|-------------|
| `subscriberService.add()` | Create subscriber |
| `subscriberService.list()` | List with pagination |
| `subscriberService.get()` | Get by ID or name |
| `subscriberService.update()` | Update |
| `subscriberService.remove()` | Delete |
| `subscriberService.enable/disable()` | Toggle active |
| `deliveryService.queueDeliveries(event)` | Queue matching deliveries |
| `deliveryService.processDeliveries()` | Send pending + retry failed |
| `deliveryService.retryDelivery(id)` | Manual retry |
| `deliveryService.expireStaleDeliveries()` | Expire old deliveries |
| `deliveryService.listDeliveries(filters, pagination)` | Query deliveries |

### API Routes

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/webhooks/subscribers` | List subscribers |
| POST | `/api/webhooks/subscribers` | Create subscriber |
| GET | `/api/webhooks/subscribers/:id` | Get subscriber |
| PUT | `/api/webhooks/subscribers/:id` | Update |
| DELETE | `/api/webhooks/subscribers/:id` | Delete |
| PUT | `/api/webhooks/subscribers/:id/enable` | Enable |
| PUT | `/api/webhooks/subscribers/:id/disable` | Disable |
| GET | `/api/webhooks/deliveries` | List deliveries |
| POST | `/api/webhooks/deliveries/:id/retry` | Retry delivery |
| POST | `/api/webhooks/process` | Process pending deliveries |

---

## End-to-End Flow Example

```bash
# 1. Create a rule that fires when a deal is created
clam --json rule add --input-json '{"slug":"deal-welcome","name":"Welcome on deal","event_pattern":"deal.created","actions":[{"extension":"contacts","action":"add-note","params_json":"{\"body\":\"New deal created!\"}"}]}'

# 2. Create a webhook subscriber for deal events
clam --json webhook subscriber add --input-json '{"name":"slack-notify","url":"https://hooks.slack.com/...","event_pattern":"deal.*","secret":"my-secret"}'

# 3. Create a deal — this emits a deal.created event
clam --json deals deal add --input-json '{"name":"Big Deal","stage":"Lead","value":100000}'

# 4. Process events — rule fires (adds note), webhook queued
clam --json event process

# 5. Deliver webhooks
# (happens automatically in daemon mode, or manually via API)
```
