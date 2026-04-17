---
name: automate-webhooks
description: >
  Inbound and outbound webhooks. Outbound: subscribe to events, deliver via
  signed HTTP with retries. Inbound: verify and log external POSTs, translate
  into outbox events. Covers schemas, signing, providers, and `clam webhook`.
metadata:
  author: openclam
  version: 0.1.0
---

# Webhooks

Webhooks are the I/O seam for OpenClam's execution layer:

- **Outbound** — subscribers listen for event patterns and get a signed
  HTTP POST whenever a matching event is processed.
- **Inbound** — endpoints accept signed POSTs from vendors (Stripe,
  GitHub, Standard Webhooks, Zapier, or no-verify), dedupe, and re-emit
  as OpenClam events so the rest of the execution layer can react.

Both live under `clam webhook`.

---

## Outbound

### `outbound_webhook_subscribers`

| Column          | Type   | Notes                                          |
| --------------- | ------ | ---------------------------------------------- |
| `id`            | uuid   |                                                |
| `slug`          | text   | Stable handle (code-enforced uniqueness)       |
| `name`          | text   | Display name                                   |
| `url`           | text   | Destination URL                                |
| `event_pattern` | text   | `person.created`, `deal.*`, `*`                |
| `conditions`    | text?  | Optional JSON — same operators as rules        |
| `secret`        | text   | HMAC-SHA256 key (supports `$secret:NAME` refs) |
| `headers_json`  | text?  | JSON object — extra request headers            |
| `is_active`     | bool   |                                                |
| `retry_policy`  | text   | `exponential` (default) \| `linear` \| `fixed` |
| `max_retries`   | int    | 0–20, default 5                                |
| `created_at` / `updated_at` / `deleted_at` | text |                              |

### `outbound_webhook_deliveries`

| Column            | Type  | Notes                                                     |
| ----------------- | ----- | --------------------------------------------------------- |
| `id`              | uuid  |                                                           |
| `subscriber_id`   | uuid  |                                                           |
| `event_id`        | uuid  |                                                           |
| `status`          | text  | `pending` \| `delivered` \| `failed` \| `expired`         |
| `attempts`        | int   |                                                           |
| `last_attempt_at` / `next_retry_at` | text? |                                            |
| `status_code`     | int?  | HTTP code from last attempt                               |
| `response_body`   | text? | Truncated response                                        |
| `failure_reason`  | text? |                                                           |

### Lifecycle

1. Event is processed → delivery hook (`queueDeliveries`) matches
   subscribers by `event_pattern` + `conditions` and inserts one `pending`
   delivery row per match.
2. The delivery worker ticks `processDeliveries()`:
   - Picks up `pending` rows and `failed` rows with `next_retry_at <= now`.
   - POSTs JSON; sets `status='delivered'` on 2xx, otherwise computes
     `next_retry_at` per `retry_policy` and `max_retries`.
   - Exponential policy: `2^attempts × 5s`, capped at 5 minutes.
3. `expire` moves deliveries older than 7 days that are still `pending`
   or `failed` to `expired`.

### Delivery request

```http
POST <subscriber.url>
Content-Type: application/json
X-Clam-Signature: sha256=<hex HMAC of raw body using subscriber.secret>
X-Clam-Event-Type: <event.event_type>
X-Clam-Delivery-Id: <delivery.id>
…plus any headers_json entries

{
  "delivery_id": "…",
  "subscriber_id": "…",
  "event": {
    "id": "…", "source": "openclam", "event_type": "person.created",
    "entity_type": "person", "entity_id": "…",
    "payload": { … },
    "user_id": "…", "created_at": "ISO"
  },
  "timestamp": "ISO"
}
```

Consumers verify by recomputing `sha256=HEX(HMAC(secret, rawBody))` and
comparing against `X-Clam-Signature`.

### `clam webhook outbound` CLI

```bash
clam webhook outbound list    [--cursor <c>] [--limit <n>] [--fields <csv>]
clam webhook outbound get     <ref>              # slug or uuid
clam webhook outbound add     --input-json '{ slug, name, url, event_pattern, secret, … }'
clam webhook outbound update  <ref>  --input-json '{ … }'
clam webhook outbound remove  <ref>
clam webhook outbound enable  <ref>
clam webhook outbound disable <ref>
clam webhook outbound import  <file> [--format csv|json]
clam webhook outbound export  [file] [--format csv|json]

clam webhook outbound delivery list   [--subscriber-id <id>] [--event-id <id>] [--status <s>] [--cursor] [--limit] [--fields]
clam webhook outbound delivery process [--limit <n>]
clam webhook outbound delivery retry  <id>
clam webhook outbound delivery expire                        # expires >7d old
```

`add` requires `--input-json`. `--schema` on `list` / `get` prints the
subscriber Zod schema as JSON Schema.

---

## Inbound

### `inbound_webhook_endpoints`

| Column                | Type  | Notes                                                      |
| --------------------- | ----- | ---------------------------------------------------------- |
| `id`                  | uuid  |                                                            |
| `slug`                | text  | Used in the public URL: `/api/webhooks/inbound/receive/<slug>` |
| `name`                | text  |                                                            |
| `provider`            | text  | `standard-webhooks` \| `stripe` \| `github` \| `zapier` \| `none` |
| `secret`              | text  | Raw secret or `$secret:NAME` vault ref                     |
| `headers_config`      | text? | JSON — provider-specific (e.g. `headerPrefix` for Standard Webhooks, `tokenHeader` for Zapier) |
| `event_type_override` | text? | Fallback when the verifier can't infer an event type       |
| `description`         | text? |                                                            |
| `is_active`           | bool  |                                                            |
| `created_at` / `updated_at` / `deleted_at` | text |                               |

### `inbound_webhook_log`

Idempotency + audit. `(endpoint_id, dedup_key)` is unique; retries of
the same source payload are short-circuited.

| Column        | Type   | Notes                                                                |
| ------------- | ------ | -------------------------------------------------------------------- |
| `id`          | uuid   |                                                                      |
| `endpoint_id` | uuid   |                                                                      |
| `dedup_key`   | text   | Stripe event id / GitHub delivery UUID / SWH msg id / sha256(rawBody) |
| `source_ip`   | text?  |                                                                      |
| `received_at` | text   | ISO                                                                  |

### Verification & translation

Each provider has a verifier that:

1. Validates the signature against `secret` (via vault if `$secret:NAME`).
2. Extracts a deterministic `dedup_key`.
3. Derives an `event_type` (e.g. `stripe.customer.created`,
   `github.push`) — falling back to `event_type_override` if set.
4. Writes an `inbound_webhook_log` row (skipped on dedup hit) and calls
   `emitEvent` with `source=<provider>`, `module=null`, the derived
   `event_type`, and the raw body as payload.

From that point the event flows through the normal outbox — rules,
workflow triggers, outbound webhook subscribers can all react.

### `clam webhook inbound` CLI

```bash
clam webhook inbound list    [--provider <p>] [--cursor <c>] [--limit <n>] [--fields <csv>]
clam webhook inbound get     <ref>
clam webhook inbound add     [--slug …] [--name …] [--provider standard-webhooks|stripe|github|zapier|none] \
                             [--secret <literal-or-$secret:NAME>] [--description <text>] \
                             [--input-json '{ …full config… }']
clam webhook inbound update  <ref> --input-json '{ … }'
clam webhook inbound remove  <ref>
clam webhook inbound enable  <ref>
clam webhook inbound disable <ref>
clam webhook inbound test    <ref> [--input-json <synthetic payload>]

clam webhook inbound log list    [--endpoint <ref>] [--cursor <c>] [--limit <n>] [--fields <csv>]
clam webhook inbound log cleanup --older-than-ms <ms> | --older-than-days <days>
```

`inbound test` sends a synthetic payload through the same verifier +
dedupe + translate pipeline an external provider would hit — use it to
wire rules without waiting on live traffic.

All inbound-webhook CLI commands require local DB access; the
`/api/webhooks/inbound/receive/<slug>` route is the only surface that
handles external POSTs.

See [`../surfaces.md`](../surfaces.md) to translate these operations to MCP,
HTTP, or the typed client.
