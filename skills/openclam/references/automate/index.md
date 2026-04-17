---
name: automate-index
description: >
  The OpenClam execution layer — events, rules, scheduled jobs, functions,
  workflows, and webhooks. Map of how they fit together and when to reach
  for which primitive.
metadata:
  author: openclam
  version: 0.1.0
---

# Automate

Every mutation in OpenClam (add a person, update a deal, upload a file) writes
a row to the `events` outbox. The daemon drains that outbox on a tick and fans
the event out to four places in order: in-process handlers, database **rules**,
**workflow** triggers, and **outbound webhook** deliveries. **Scheduled jobs**
(cron) and **inbound webhooks** are the two ways events enter the system from
outside the mutation path.

```
mutation ─▶ events outbox ─▶ rules ─▶ action / function / workflow
                          ├──────▶ workflow event triggers & waiters
                          └──────▶ webhook delivery queue

clock      ─▶ cron tick   ─▶ scheduled handler (action | function | workflow)
http POST  ─▶ inbound webhook endpoint ─▶ events outbox
```

All storage lives in the workspace database; the daemon's event loop, cron
scheduler, workflow executor, and webhook delivery worker all poll on a tick.

## When to pick what

| I want to…                                                | Reach for           |
| --------------------------------------------------------- | ------------------- |
| React to an event with a single known extension action    | Rule → action       |
| Write my own reactive logic in TypeScript                 | Rule → function     |
| Orchestrate multiple steps, sleeps, or wait for an event  | Workflow            |
| Run something on a clock                                  | Schedule            |
| Run a function or workflow on a clock                     | Function with `trigger.cron` / Workflow with cron trigger |
| Push events out to an external HTTP endpoint              | Outbound webhook    |
| Receive signed HTTP from Stripe / GitHub / Standard Webhooks | Inbound webhook  |
| Replay after failure, keep step-level audit, run child workflows | Workflow    |
| Short, stateless TS — no replay, no run history needed    | Function            |

Functions and workflows overlap — the rule of thumb:

- **Function** runs inline, returns a value, has no step memoization. No run
  record is kept. Good for scoring, enrichment, single-shot side effects.
- **Workflow** runs through an executor with per-step attempt records,
  deterministic replay, sleep/wait-for-event primitives, and child workflows.
  Good for anything multi-step or long-lived.

## Routing

| File                           | Covers                                                        |
| ------------------------------ | ------------------------------------------------------------- |
| [events.md](./events.md)       | Outbox schema, emission, pattern matching, `clam event` / `clam events` |
| [rules.md](./rules.md)         | Rule + rule-action tables, conditions, `clam rule`            |
| [schedule.md](./schedule.md)   | Cron jobs, backoff, locking, `clam schedule`                  |
| [functions.md](./functions.md) | `defineFunction`, workspace `functions/` dir, `clam function` |
| [workflows.md](./workflows.md) | `defineWorkflow`, step primitives, triggers, `clam workflow`  |
| [webhooks.md](./webhooks.md)   | Outbound (subscribers, deliveries) + inbound (endpoints, log), `clam webhook` |

See [`../surfaces.md`](../surfaces.md) to translate these operations to MCP,
HTTP, or the typed client.
