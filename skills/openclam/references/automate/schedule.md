---
name: automate-schedule
description: >
  Scheduled jobs — cron-driven handlers that run on a clock. Covers the
  5/6-field cron grammar, handler naming (action / workflow / function),
  exponential backoff on failure, job locking, and the `clam schedule` CLI.
metadata:
  author: openclam
  version: 0.1.0
---

# Schedule (cron)

Scheduled jobs are rows in `cron_jobs` with a cron expression, a named
handler, and optional JSON params. The daemon's `CronScheduler` ticks
periodically: it looks for due jobs, locks them, runs the handler, and
advances `next_run_at`. Failures apply a bounded exponential backoff.

## Grammar

Backed by [croner](https://github.com/hexagon/croner). Both 5-field
(minute) and 6-field (second) expressions are accepted. Lists (`1,15`),
ranges (`1-5`), steps (`*/5`), day/month names (`MON`, `JAN`), and `L`/`W`
flags all work.

| Field order | Min      | Hour  | Day of month | Month | Day of week |
| ----------- | -------- | ----- | ------------ | ----- | ----------- |
| 5-field     | ✓        | ✓     | ✓            | ✓     | ✓           |
| 6-field     | +second prefix | …       | …      | …    | …           |

Timezones are per-job via the `timezone` column (IANA name, e.g.
`America/New_York`). If omitted, the daemon's local time is used.

Helpers:

```ts
import { getNextRun, getNextRuns, isValidCron } from '@openclam/cron';

isValidCron('0 9 * * MON');               // true
getNextRun('0 9 * * MON', new Date());    // next Monday 09:00
getNextRuns('*/5 * * * *', 3, new Date());
```

## Tables

**`cron_jobs`**

| Column                | Type  | Notes                                               |
| --------------------- | ----- | --------------------------------------------------- |
| `id`                  | uuid  |                                                     |
| `slug`                | text  | Unique handle                                       |
| `name`                | text  | Display name                                        |
| `schedule`            | text  | Cron expression                                     |
| `handler`             | text  | Namespaced handler name (see below)                 |
| `params_json`         | text? | JSON params passed to the handler                   |
| `timezone`            | text? | IANA TZ                                             |
| `is_active`           | bool  | Disabled jobs are skipped                           |
| `is_system`           | bool  | Seeded by the daemon; managed, not user-created     |
| `consecutive_errors`  | int   | Drives backoff                                      |
| `running_since`       | text? | Lock timestamp (cleared when the job finishes)      |
| `last_run_at` / `next_run_at` | text? | ISO                                         |
| `created_at` / `updated_at` / `deleted_at` | text | Soft-delete via `deleted_at`   |

**`cron_runs`** — one row per execution: `job_id`, `status` (`running`,
`completed`, `failed`), `started_at`, `finished_at`, `duration_ms`, `error`.

## Handler forms

Handlers are resolved by name. Built-in handlers (e.g. `core.process-events`,
`workflows.tick`) are registered at daemon startup via
`scheduler.registerHandler(name, fn)`. A fallback resolver dispatches
user-facing names to the correct subsystem:

| Handler form           | What runs                                               |
| ---------------------- | ------------------------------------------------------- |
| `action:ext/action`    | Extension action by `<extension>:<action>`              |
| `workflow:<slug>`      | Creates a workflow run with trigger_type `cron`         |
| `function:<slug>`      | Invokes the user function by slug                       |
| `core.process-events`, `workflows.tick`, `webhooks.process`, … | Registered system handlers |

`params_json`, if present, is parsed and passed as the handler's first arg
(or as the workflow/function input).

## Tick loop & backoff

1. `tick()` reads due jobs (`next_run_at <= now`, active, not locked).
2. Up to `maxConcurrentRuns` (default **3**) are locked via
   `running_since` and executed concurrently.
3. On success: `consecutive_errors = 0`, `next_run_at` advances to the
   next natural occurrence.
4. On failure: `consecutive_errors++`, next run = `max(finishedAt +
   backoff, naturalNext)` where backoff follows **30s → 1m → 5m → 15m →
   60m** and caps at 60m.

`recoverMissedJobs()` runs once at daemon startup: it clears stale locks
and advances any job whose `next_run_at` is in the past to the next
future occurrence (missed runs are **skipped, not replayed**).

## Service surface

```ts
import { cronService } from '@openclam/cron';

cronService.addJob(db, tables, { slug, name, schedule, handler, params_json?, timezone? });
cronService.listJobs(db, tables, filters?, pagination?);
cronService.getJobByRef(db, tables, slugOrId);
cronService.updateJob(db, tables, id, patch);
cronService.removeJob(db, tables, id);              // soft delete
cronService.listRuns(db, tables, jobId, pagination?);
cronService.addRun(db, tables, { job_id, status }); // used by `trigger`
cronService.lockJob / unlockJob / clearStaleLocks;
cronService.getDueJobs(db, tables);
```

## `clam schedule` CLI

```bash
clam schedule add <slug> \
  --name "Process events" \
  --schedule "*/1 * * * *" \
  --handler "core.process-events" \
  [--params '{"limit":100}']

clam schedule list    [--include-deleted] [--fields <csv>]
clam schedule get     <slug>
clam schedule enable  <slug>
clam schedule disable <slug>
clam schedule runs    <slug>  [--limit <n>]
clam schedule trigger <slug>           # enqueue a pending run immediately
clam schedule remove  <slug>           # soft delete
```

All commands accept `--input-json`; flag values override `--input-json`
values (except `add`, where `--input-json` wins).

Creation refuses invalid cron expressions (validated via `isValidCron`).
Updates are ID-based; use `update` via the oRPC/HTTP surface for schedule
edits (CLI exposes lifecycle + trigger).

See [`../surfaces.md`](../surfaces.md) to translate these operations to MCP,
HTTP, or the typed client.
