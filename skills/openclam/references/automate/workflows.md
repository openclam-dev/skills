---
name: automate-workflows
description: >
  Multi-step durable workflows. `defineWorkflow`, step primitives
  (run / sleep / sleepUntil / waitForEvent / sendEvent / runWorkflow),
  triggers (event / cron / manual with arrays and filters), the workspace
  `workflows/` folder, hot-reload, run status lifecycle, and `clam workflow`.
metadata:
  author: openclam
  version: 0.1.0
---

# Workflows

A workflow is an author-owned TypeScript file whose default export is a
`defineWorkflow({ … }, handler)` value. The daemon loads workflows from
the workspace, registers their triggers, and executes runs with
**deterministic replay** — every call to a `step.*` primitive is
persisted, and on retry the executor skips completed steps and resumes
at the first unfinished one.

Pick a workflow over a [function](./functions.md) when you need any of:
sleeping, waiting for events, retry with attempt tracking, child
workflows, or a visible run history.

## Where they live

```
<workspace>/
  workflows/
    package.json         # auto-scaffolded
    tsconfig.json        # auto-scaffolded
    types.d.ts           # codegen: augments WorkflowExtensions
    onboard-contact/
      index.ts
    reconcile-stripe/
      index.ts
```

Each workflow lives in its own `<slug>/` folder; the handler is the
`index.{ts,mts,js,mjs}` inside it. The folder name **is** the slug.

Regenerate `types.d.ts` with `clam workflow types` after enabling a new
extension. Install npm dependencies into `workflows/` via
`clam workflow install <pkg>` / `clam workflow uninstall <pkg>` — they
run inside the `workflows/` folder.

The daemon watches this directory and hot-reloads on save. Each workflow
is upserted into `workflow_definitions` (keyed by `slug`) with its
`triggers_json` and validity state.

## `defineWorkflow`

```ts
function defineWorkflow<TInput, TOutput>(
  spec: WorkflowSpec<TInput>,
  handler: WorkflowHandler<TInput, TOutput>,
): WorkflowDefinition<TInput, TOutput>;

interface WorkflowSpec<TInput> {
  slug?: string;                       // auto-derived from name
  name: string;                        // required
  description?: string;
  trigger: WorkflowTrigger | WorkflowTrigger[];
  input?: z.ZodType<TInput>;
  retry?: { maxAttempts?, initialDelayMs?, backoffMultiplier?, maxDelayMs? };
}

type WorkflowTrigger =
  | { event: string; filter?: Record<string, unknown> }
  | { cron:  string; timezone?: string }
  | { manual: true };
```

An array of triggers is legal — the workflow fires when **any** of them
matches. Event filters are the same dot-notation condition language used
by rules (see [rules.md](./rules.md)).

### Context & handler

```ts
type WorkflowHandler<TInput, TOutput> =
  (ctx: WorkflowContext<TInput>) => Promise<TOutput>;

interface WorkflowContext<TInput> {
  input:      TInput;
  event:      unknown | null;      // original triggering event (event trigger only)
  step:       StepAPI;
  db:         OpenClamDb;
  tables:     OpenClamTables;
  extensions: WorkflowExtensions;  // typed via codegen
  run: { id: string; workflowName: string; attemptNumber: number; startedAt: string };
}
```

Anything your handler does that isn't wrapped in `step.run(...)` will
re-execute on replay — always put side effects inside a step.

## Step primitives

```ts
interface StepAPI {
  run<T>(name: string, fn: () => Promise<T>, opts?: { retry?: RetryConfig }): Promise<T>;
  sleep(name: string, duration: string): Promise<void>;
  sleepUntil(name: string, timestamp: Date | string): Promise<void>;
  waitForEvent<T>(name: string, opts: { event: string; match?: Record<string, unknown>; timeout: string }): Promise<T | null>;
  sendEvent(name: string, events: Array<{ type: string; entityType?: string; entityId?: string; payload?: Record<string, unknown> }>): Promise<void>;
  runWorkflow<T>(name: string, workflowName: string, opts?: { input?: unknown; timeout?: string }): Promise<T>;
}
```

| Primitive       | Persists in `workflow_step_attempts` as | Suspend?                                    |
| --------------- | --------------------------------------- | ------------------------------------------- |
| `run`           | `run`                                   | No                                          |
| `sleep`         | `sleep`                                 | Yes — run status `sleeping`                 |
| `sleepUntil`    | `sleep_until`                           | Yes — `sleeping`                            |
| `waitForEvent`  | `wait_for_event`                        | Yes — `waiting_for_event`; row in `workflow_event_waiters` with `timeout_at` |
| `sendEvent`     | `send_event`                            | No                                          |
| `runWorkflow`   | `run_workflow`                          | Yes — `waiting_for_child`                   |

Duration strings: `100ms`, `5s`, `2m`, `1h`, `3d`, `1w`, `3mo`, `1yr`
(months = 30d, years = 365d).

`waitForEvent` returns the matching event or `null` on timeout.
`sendEvent` forwards to `emitEvent` with `source='openclam'`,
`module='workflows'`, and metadata tagging the emitting run.

## Minimal example

```ts
// workflows/hello/index.ts
import { defineWorkflow } from '@openclam/workflows';

export default defineWorkflow(
  { name: 'Hello', trigger: { manual: true } },
  async ({ input, step }) => {
    await step.run('log', async () => { console.log('hello', input); });
    return { ok: true };
  },
);
```

Invoke: `clam workflow run hello --input '{"who":"world"}'`.

## Realistic example

Onboard a person: greet, wait for a reply, either mark engaged or send a
follow-up:

```ts
// workflows/onboard-contact/index.ts
import { defineWorkflow } from '@openclam/workflows';
import { z } from 'zod';

export default defineWorkflow(
  {
    slug: 'onboard-contact',
    name: 'Onboard contact',
    description: 'Welcome email, wait 7d for reply, follow up if silent.',
    trigger: [
      { event: 'person.created', filter: { 'payload.source': 'import' } },
      { manual: true },
    ],
    input: z.object({ personId: z.string().uuid() }),
    retry: { maxAttempts: 3, initialDelayMs: 30_000, backoffMultiplier: 2 },
  },
  async ({ input, step, extensions, run }) => {
    const person = await step.run('fetch-person', async () =>
      extensions.contacts.persons.get(input.personId),
    );

    await step.run('send-welcome', async () => {
      // side effect wrapped in a step so replay skips it
      return extensions.contacts.persons.addNote(person.id, {
        body: `Welcome ${person.first_name}!`,
      });
    });

    const reply = await step.waitForEvent<{ payload: { body: string } }>(
      'await-reply',
      {
        event: 'message.replied',
        match: { 'payload.person_id': person.id },
        timeout: '7d',
      },
    );

    if (reply) {
      await step.run('tag-engaged', async () =>
        extensions.contacts.persons.tag(person.id, 'engaged'),
      );
      await step.sendEvent('announce', [
        { type: 'person.engaged', entityType: 'person', entityId: person.id },
      ]);
      return { engaged: true, run: run.id };
    }

    await step.sleep('cool-off', '1h');
    await step.runWorkflow('nudge', 'follow-up-drip', {
      input: { personId: person.id },
    });

    return { engaged: false };
  },
);
```

## Triggers

| Shape                                    | Notes                                             |
| ---------------------------------------- | ------------------------------------------------- |
| `{ event: 'person.created' }`            | Same matcher as rules (`*`, `prefix.*`)           |
| `{ event: 'person.created', filter: { 'payload.stage': 'qualified' } }` | Dot-notation condition language from rules |
| `{ cron: '0 9 * * MON', timezone: 'America/New_York' }` | Registered with the cron scheduler   |
| `{ manual: true }`                       | Only via `clam workflow run` or the API           |
| `[ …, …, … ]`                            | Array — fires on any match                        |

Event-triggered runs receive the whole event (`{ …event, payload:
parsed }`) as `input`. Cron and manual runs use the configured input.

Only **one run per definition per event** is created on an event hit
(the first matching trigger wins).

## Run lifecycle

Columns on `workflow_runs` of note: `status`, `input_json`, `output_json`,
`error`, `trigger_type` (`event` | `cron` | `manual`), `trigger_event_id`,
`rule_id`, `parent_run_id`, `parent_step_name`, `worker_id`,
`available_at`, `consecutive_errors`, `attempt_number`, `max_attempts`
(default 3), `started_at`, `finished_at`.

```
pending ─▶ running ─┬▶ sleeping           ─▶ running ─▶ …
                     ├▶ waiting_for_event  ─▶ running ─▶ …
                     ├▶ waiting_for_child  ─▶ running ─▶ …
                     ├▶ completed
                     ├▶ failed
                     └▶ canceled
```

The executor tick (`ExecutorOptions.maxConcurrentRuns`, default 10):

1. Expire timed-out event waiters — matching runs fail their step and
   move on.
2. Resolve completed child workflows — parent runs become `pending` with
   the child output piped back into the step attempt.
3. Claim claimable runs (`pending` / `sleeping` / `waiting*` with
   `available_at <= now`).
4. Execute up to `maxConcurrentRuns` in parallel. Failures retry with
   exponential backoff up to `max_attempts`; after that the run is
   `failed`.

## Underlying tables

- `workflow_definitions` — loaded code (`slug`, `name`, `file_path`,
  `triggers_json`, `is_valid`, `error`, `is_active`).
- `workflow_runs` — every invocation.
- `workflow_step_attempts` — per-step record with `step_type`, `status`
  (`running` / `completed` / `failed`), `output_json`, `error`,
  `duration_ms`, `ordinal`.
- `workflow_event_waiters` — `(run_id, step_name, event_pattern,
  match_json, timeout_at)` rows, resolved or expired by the tick.

## `clam workflow` CLI

```bash
clam workflow new         <slug>   [--name <label>] [--description <text>] [--force]
clam workflow list        [--include-invalid] [--fields <csv>]
clam workflow get         <ref>                # slug | name | uuid
clam workflow run         <ref>    [--input <json>]
clam workflow runs        [<ref>]  [--status <s>] [--limit <n>]
clam workflow run-detail  <run-id>             # includes step attempts
clam workflow pause       <ref>
clam workflow resume      <ref>
clam workflow cancel      <run-id>             # pending|running|sleeping|waiting_*
clam workflow import      <file>   [--format csv|json]
clam workflow export      [file]   [--format csv|json]    # stdout if no file
clam workflow install     <package>            # npm install inside workflows/
clam workflow uninstall   <package>
clam workflow types                            # regenerate workflows/types.d.ts
```

`new` scaffolds `<workspace>/workflows/<slug>/index.ts` with a starter
template (and creates the shared `package.json` / `tsconfig.json` in
`workflows/` if missing). Refuses to overwrite without `--force`.

`pause` / `resume` flip `is_active` on the definition (new triggers stop
firing; in-flight runs continue). `cancel` moves a single run to
`canceled`. All workflow commands require local DB access and are not
available in remote mode.

See [`../surfaces.md`](../surfaces.md) to translate these operations to MCP,
HTTP, or the typed client.
