---
name: automate-functions
description: >
  User-defined functions — lightweight inline TypeScript triggered by rules,
  cron, or manual `clam function run`. Covers `defineFunction`, the workspace
  `functions/` folder, hot-reload, context shape, and the full CLI surface.
metadata:
  author: openclam
  version: 0.1.0
---

# Functions

Functions are the simplest way to author custom TypeScript that participates
in the execution layer. They run **inline**, return a value, and leave **no
run record** behind. Use a function when you'd otherwise wire a single-file
handler; reach for a [workflow](./workflows.md) when you need steps, sleeps,
waiting, retries, or child flows.

## Where they live

Each workspace has a `functions/` directory (created on demand at
`<workspace>/functions/`):

```
<workspace>/
  functions/
    package.json         # auto-scaffolded, private
    tsconfig.json        # auto-scaffolded
    types.d.ts           # codegen: augments FunctionExtensions with enabled extension clients
    score-icp/
      index.ts
    enrich-person/
      index.ts
```

Each function lives in its own `<slug>/` folder; the handler is the
`index.{ts,mts,js,mjs}` inside it. The folder name **is** the slug —
it's the primary identity. The daemon scans `functions/` on startup and
hot-reloads on change. Hidden files, `node_modules`, `package.json`,
`tsconfig.json`, and `*.d.ts` are skipped.

Use `clam function new <slug>` to scaffold a folder — it writes the
shared `package.json` / `tsconfig.json` if missing, then creates
`functions/<slug>/index.ts` with a starter template.

## `defineFunction`

```ts
function defineFunction<TInput, TOutput>(
  spec: FunctionSpec<TInput>,
  handler: FunctionHandler<TInput, TOutput>,
): FunctionDefinition<TInput, TOutput>;

interface FunctionSpec<TInput> {
  slug?: string;                 // auto-derived from name via slugify()
  name: string;                  // required; display label
  description?: string;
  input?: z.ZodType<TInput>;     // optional validator
  trigger?: { cron: string; timezone?: string };   // optional cron trigger
}

interface FunctionContext<TInput> {
  input: TInput;                 // validated against spec.input when provided
  event: unknown | null;         // the triggering event (from a rule); null for manual/cron
  db: OpenClamDb;
  tables: OpenClamTables;
  extensions: FunctionExtensions; // typed via codegen — ctx.extensions.contacts etc.
}

type FunctionHandler<TInput, TOutput> =
  (ctx: FunctionContext<TInput>) => Promise<TOutput>;
```

Functions must be the **default export** of their file.

## Minimal example

```ts
// functions/hello/index.ts
import { defineFunction } from '@openclam/functions';

export default defineFunction(
  { name: 'Hello' },
  async ({ input }) => {
    console.log('hello', input);
    return { ok: true };
  },
);
```

Invoke: `clam function run hello --input '{"who":"world"}'`.

## Realistic example

Scoring function triggered by a rule on `org.updated`:

```ts
// functions/score-icp/index.ts
import { defineFunction } from '@openclam/functions';
import { z } from 'zod';

export default defineFunction(
  {
    slug: 'score-icp',
    name: 'Score enterprise ICP',
    description: 'Attach an ICP score based on industry and headcount.',
    input: z.object({ org_id: z.string().uuid() }),
  },
  async ({ input, event, db, tables, extensions }) => {
    const org = await extensions.contacts.orgs.get(input.org_id);

    let score = 0;
    if (org.industry === 'SaaS') score += 20;
    if ((org.employees ?? 0) >= 100) score += 15;
    if (org.country === 'US') score += 5;

    await extensions.contacts.orgs.update(org.id, {
      custom_fields: { icp_score: score },
    });

    return {
      org_id: org.id,
      score,
      fired_by: event ? (event as { id: string }).id : 'manual',
    };
  },
);
```

Register a rule to drive it:

```bash
clam rule add score-on-org-update --input-json '{
  "slug": "score-on-org-update",
  "name": "Score ICP when an org is updated",
  "event_pattern": "org.updated",
  "actions": [{ "target_type": "function", "function_name": "score-icp", "execution_order": 0 }]
}'
```

When the rule fires, the daemon's function trigger hook finds `score-icp`
in the registry, passes `{ input: <event payload>, event: <full event> }`
into the handler, and records no run row (the rule's `rule_run` captures
the fact that it fired).

## Hot-reload & registry

The daemon keeps an in-memory registry (`registerFunction` /
`lookupFunction` / `clearFunctions` / `listAllFunctions` in
`@openclam/functions`). On file change the daemon clears and re-scans.
Extensions (or tests) can call `refreshFunctions()` to trigger a re-scan
on demand; the daemon registers the concrete implementation via
`onFunctionRegistryRefresh()`.

Typed extension clients come from `types.d.ts` in the functions directory
— the `clam workspace doctor` / workflow `types` commands regenerate this
when the enabled-extension set changes. `ctx.extensions.contacts.orgs.get`
etc. only autocomplete once the file has been written.

## Function vs workflow — when to pick which

| Need                                                 | Function | Workflow |
| ---------------------------------------------------- | -------- | -------- |
| Pure compute, return value, short-lived              | ✓        |          |
| Multi-step with durability across restarts           |          | ✓        |
| `sleep`, `sleepUntil`, `waitForEvent`                |          | ✓        |
| Child workflows                                      |          | ✓        |
| Per-step audit trail + retry with attempt tracking   |          | ✓        |
| Cron trigger                                         | ✓ (via `spec.trigger.cron`) | ✓ |
| Event trigger                                        | via rule | direct (`trigger: { event: … }`) or via rule |

## `clam function` CLI

```bash
clam function new <slug>  [--name <label>] [--description <text>] [--force]
clam function list        [--fields <csv>]
clam function run <slug>  [--input <json>]
```

- `new` scaffolds `<workspace>/functions/<slug>/index.ts` with a starter
  template (and creates the shared `package.json` / `tsconfig.json` in
  `functions/` if missing). Refuses to overwrite without `--force`.
- `list` scans the directory using `jiti` and reports `{ slug, name,
  description, file, valid, error }` for each entry. Invalid files
  (missing `spec.name` or `handler`, import errors) are returned with
  `valid: false` and an error message.
- `run` executes inline with the `extensions` field left empty — the
  typed extension clients are only wired up inside the daemon. For
  realistic local runs use a rule or trigger via the daemon.

All function CLI commands require local DB access and are not available
in remote mode.

### Cron-triggered functions

Set `trigger: { cron: '0 9 * * *' }` in the spec. The daemon registers a
matching `cron_job` with handler `function:<slug>` on load. The triggering
context has `event: null` and `input: undefined` unless `params_json` is
provided on the cron row.

See [`../surfaces.md`](../surfaces.md) to translate these operations to MCP,
HTTP, or the typed client.
