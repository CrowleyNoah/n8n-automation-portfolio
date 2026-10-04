# 04 — Secured Webhook Order Processor

A single n8n workflow that receives order webhooks safely. It verifies an HMAC signature over the exact bytes the sender signed, rejects replays, records each order atomically so it is processed exactly once however many times it is delivered, replies "accepted" immediately, and then reserves inventory, creates the order, saves the final status and notifies the customer or ops.

Webhook handlers usually fail in two ways: they trust requests they should not (spoofed or replayed calls), and they do all their work before answering, so the sender times out and retries and the order is processed twice. This workflow is built around avoiding both, and around what happens when a downstream system fails after the sender has already been told "200".

## Design principle

Nothing is hardcoded. The signing secret, the header names, every service URL and the timeouts live in the `CONFIG` node. The workflow talks to four small HTTP services (an event store, an inventory service, an order system, a customer-notification service) plus a security log and a Slack-compatible webhook. Any backend that honours the contracts below works.

## The webhook

`POST /webhook/orders/incoming`, JSON body:

```json
{ "order_id": "order-1001", "amount": 49.5, "currency": "USD",
  "line_items": [ {"sku": "WIDGET", "quantity": 2} ], "customer_email": "cust@example.com" }
```

| Field | Required | Meaning |
|---|---|---|
| `order_id` | yes | 1–128 characters: letters, digits and `. _ : -`. The idempotency key. |
| `amount` | yes | Number ≥ 0. |
| `currency` | yes | 3-letter code (stored upper-case). |
| `line_items` | yes | 1–200 objects. If an item has `quantity` it must be a whole number ≥ 1. Other fields are passed through untouched to inventory and the order system. |
| `customer_email` | no | Valid address (stored lower-case). Without it no customer notice is sent. |

**Signature.** `CONFIG.SIGNATURE_SCHEME` picks how the sender signs. In every scheme the signature is an HMAC-SHA256 keyed with `WEBHOOK_SIGNING_SECRET` over the exact bytes received, compared in constant time.

| `SIGNATURE_SCHEME` | Header(s) | What is signed | Timestamp window |
|---|---|---|---|
| `generic` (default) | `SIGNATURE_HEADER` (`x-signature`) and `TIMESTAMP_HEADER` (`x-timestamp`) | `timestamp + "." + rawBody` as hex; `SIGNED_CONTENT=body` and `SIGNATURE_ENCODING=base64` change this for other senders | yes (if a timestamp is signed or `TIMESTAMP_HEADER` is set) |
| `stripe` | `Stripe-Signature: t=<time>,v1=<hex>` | `t + "." + rawBody` as hex. Several `v1` values are allowed (secret rotation): one match is enough. `v0` is ignored. | yes, from `t` |
| `github` | `X-Hub-Signature-256: sha256=<hex>` | `rawBody` as hex. The `sha256=` prefix is required. | no |
| `shopify` | `X-Shopify-Hmac-Sha256: <base64>` | `rawBody` as base64 (compared exactly, capitals matter) | no |

The presets use their own fixed header names and ignore `SIGNATURE_HEADER` / `TIMESTAMP_HEADER`. Put the secret the sender shows you (Stripe's `whsec_...`, the GitHub webhook secret, Shopify's app secret) into `WEBHOOK_SIGNING_SECRET`. The generic scheme accepts a leading `sha256=`, upper-case hex and millisecond timestamps.

**Replay protection differs by scheme.** Where there is no signed timestamp (`github`, `shopify`, a body-only generic scheme) nothing limits how old a captured request can be. What stops a replay is the order record: the same signed request again is a `200 duplicate` and nothing runs twice, for as long as the event store keeps the record. Only the sender's own formats are supported; the payload is not an order in the shape this workflow wants unless you map it (the Order Mapping node; `04-secured-webhook-order-processor-EXTEND.md` recipe 11 shows Shopify's orders).

For schemes with a signed timestamp, it must be within `TIMESTAMP_TOLERANCE_SECONDS` (300) of now, in either direction.

| Status | Body | When |
|---|---|---|
| 200 | `{status:"accepted", order_id}` | Verified, recorded, and processing has started (it continues after this reply). |
| 200 | `{status:"duplicate", order_id, state}` | This `order_id` with the same contents was already received. `state` is its current status (`received`, `fulfilled`, `backordered`, `pending_manual`). Nothing is reprocessed. |
| 400 | `{error:"validation_failed", details:[...]}` or `{error:"malformed_payload", reason}` | Signed correctly, but the content is bad. Nothing recorded. |
| 422 | `{error:"order_rejected", details:[...]}` | Signed and valid, but one of your Order Mapping `rules` rejected it. Nothing recorded. |
| 401 | `{error:"unauthorized"}` | Missing, wrong or stale signature or timestamp. The reply never says which; the reason goes to the security log. |
| 409 | `{error:"order_conflict", order_id, reason}` | Same `order_id`, different contents. |
| 500 | `{error:"webhook_misconfigured", details:[...]}` | Invalid CONFIG, including a missing or short signing secret (every problem listed). |
| 500 | `{error:"order_playbook_invalid", details:[...]}` or `{error:"order_mapping_failed", details:[...]}` | The Order Mapping notices settings are malformed, or the Order Mapping node cannot run. Nothing recorded. |
| 503 | `{error:"event_store_unavailable", order_id, retry:true}` | The order could not be recorded, so it was **not** accepted. The sender should retry. |

Malformed JSON sent with a JSON content type never reaches the workflow: n8n itself rejects it with a `4xx` (before any signature check). Anything n8n lets through is checked.

## Architecture

```
POST /orders/incoming
  PHASE A: verify, record, acknowledge
  → CONFIG check, then signature + timestamp (constant-time compare)    (401 generic, reason → security log)
  → Order Mapping (your mapOrder / rules / notices) → Validate Order (guard)   (400 / 422 / 500)
  → RECORD the event, atomically (create-or-return-existing)
        new → continue         duplicate → 200 duplicate        different contents → 409
        store down → 503       accepted-but-stale (> RECLAIM_AFTER_SECONDS) → atomic reclaim → continue
  → reply 200 "accepted"
  PHASE B: fulfil (no caller waiting)
  → reserve inventory (1 retry)    409 = out of stock → backordered
                                   anything else → pending_manual, ops paged
  → create the order (1 retry)     failure → pending_manual, ops paged
  → save the final status (1 retry; if that fails, alert ops)
  → customer notice (confirmation / backorder) and ops alert, best effort
```

1. **Raw bytes.** The webhook keeps the raw body and the signature is checked against exactly those bytes. Re-serialising the parsed JSON (what the original fell back to) breaks valid signatures whenever the sender's spacing or key order differs.
2. **Constant-time comparison** (no early exit), and a generic 401 so an attacker learns nothing about which check failed. Failed checks are written to the security log with the real reason.
3. **Replay window.** A correctly signed request outside the timestamp window (past or future) is rejected.
4. **One atomic record = idempotency check + durable log.** The original did a read-then-write (check, later persist), so two simultaneous deliveries could both pass, and nothing ever wrote the idempotency record at all. Here the event store's create is atomic: six simultaneous deliveries produce one `accepted` and five `duplicate`.
5. **Same id, different contents is a conflict (409)**, not a "duplicate". A payload hash is stored for this.
6. **Fail closed.** If the order cannot be recorded, the reply is 503 (retry) rather than "accepted". The original treated a failed idempotency lookup as "new".
7. **Ack fast, then work.** The reply goes out as soon as the event is recorded; fulfilment continues afterwards.
8. **Out of stock is a business outcome; an outage is not.** Only a 409 from inventory means backordered. Any other failure is `pending_manual` with ops paged and **no** misleading backorder email to the customer (the original sent one).
9. **Retries where they are safe.** Inventory, order creation and the final status write each get one retry on transient failure (no response, 5xx, 408, 429), and both downstream calls carry an idempotency key (`order:{order_id}`). A definite answer (2xx, 4xx) is final.
10. **Nothing silently lost.** A failed order creation becomes `pending_manual` with the reason; a failed status write alerts ops.
11. **Takeover of abandoned orders.** If the first delivery was acknowledged but processing died, the event stays `received`. When the sender retries after `RECLAIM_AFTER_SECONDS`, an atomic reclaim lets exactly one delivery take it over and process it.
12. **Notices come last** and never change the outcome.

## Edge cases handled

- **Odd JSON formatting or non-ASCII text** in a signed body: verified correctly.
- **Six simultaneous identical deliveries:** one accepted, five duplicate, one reservation, one order, one confirmation.
- **Four simultaneous deliveries of an abandoned event:** exactly one takes it over.
- **A recent `received` event** is left alone (the first delivery may still be running).
- **Out-of-stock, inventory down, OMS down, OMS rejecting (422, not retried), a transient 500 on either (retried once, succeeds).**
- **Event store down** at receipt (503, nothing reserved or created) and at the final status write (alert).
- **No email on the order, notification service down, Slack down:** the order outcome is unaffected.
- **Slow fulfilment:** the sender's reply takes about the same time as a record call, not the 3.5 s fulfilment in the test.

## Configuration (`CONFIG` node)

| Key | Default | Meaning |
|---|---|---|
| `WEBHOOK_SIGNING_SECRET` | placeholder | The shared secret. **Change it.** Under 8 characters is refused. |
| `SIGNATURE_SCHEME` | `generic` | `generic`, `stripe`, `github` or `shopify` (see Signature above). |
| `SIGNATURE_HEADER` | `x-signature` | Generic scheme only: header carrying the signature. |
| `TIMESTAMP_HEADER` | `x-timestamp` | Generic scheme only: header carrying the timestamp. Can be blank only with `SIGNED_CONTENT=body`. |
| `SIGNED_CONTENT` | `timestamp.body` | Generic scheme only: `timestamp.body` or `body`. |
| `SIGNATURE_ENCODING` | `hex` | Generic scheme only: `hex` or `base64`. |
| `TIMESTAMP_TOLERANCE_SECONDS` | 300 | Replay window. |
| `EVENTS_STORE_API_URL` | `http://localhost:4800` | Base URL of the event store. |
| `INVENTORY_API_URL` | `http://localhost:4800` | Base URL of the inventory service. |
| `OMS_API_URL` | `http://localhost:4800` | Base URL of the order system. |
| `NOTIFY_API_URL` | `http://localhost:4800` | Base URL of the customer-notification service. Blank = no customer notices. |
| `SECURITY_LOG_API_URL` | `http://localhost:4800` | Base URL of the security log. Blank = no logging. |
| `SLACK_WEBHOOK_URL` | blank | Slack-compatible incoming webhook for ops alerts. Blank = none. |
| `SERVICE_HEADERS_JSON` | `{}` | Headers sent to the store, inventory, OMS and notify service, e.g. `{"x-api-key":"..."}`. |
| `RECLAIM_AFTER_SECONDS` | 300 | How old a still-`received` event must be before a re-delivery takes it over. Set it above your slowest legitimate fulfilment. |
| `RECONCILE_ENABLED` | `false` | Turns the reconcile job and the redrive endpoint on. Off = the schedule does nothing and both endpoints answer 403. |
| `RECONCILE_AUTH_TOKEN` | blank | Shared token for the reconcile and redrive endpoints (header `x-reconcile-token`). Required (8+ characters) when enabled. |
| `RECONCILE_ACTION` | `alert` | `alert` only reports stuck orders to Slack; `redrive` also re-runs them. |
| `RECONCILE_STALE_SECONDS` | 900 | How long an event must sit in `received` before it counts as stuck. Not allowed to be below `RECLAIM_AFTER_SECONDS`. |
| `RECONCILE_MAX_PER_RUN` | 20 | Most orders listed (and re-run) per run, 1 to 200. |
| `REDRIVE_BASE_URL` | blank | This n8n instance's own base URL (for example `https://n8n.example.com`). Needed for `redrive`: the job re-runs each order by calling this instance's own redrive webhook. |
| `RELEASE_RESERVATION_ON_OMS_FAILURE` | `off` | What to do with the inventory reservation when order creation fails. `off`: leave it held (a person decides). `rejected`: give it back only when the order system DEFINITELY refused the order (a 4xx other than 408/429). `any`: give it back on any failure, including no answer or a 5xx. Use `any` only if your order system can never create an order after answering with an error. |
| `REQUEST_TIMEOUT_MS` | 10000 | Timeout for store, security log, Slack and notify calls. |
| `FULFILMENT_TIMEOUT_MS` | 15000 | Timeout for inventory and OMS calls. |

## Service contract

All JSON. `SERVICE_HEADERS_JSON` is sent on every store, inventory, OMS and notify call. `{id}` is URL-encoded.

**Event store**

| Call | Body | Response |
|---|---|---|
| `POST /events` | `{order_id, payload_hash, order}` | `201 {record}` if new; `200 {existing:true, record}` if that `order_id` exists. **Must be atomic create-or-return-existing.** A record has `order_id`, `payload_hash`, `status` (`received` at creation), `received_at` (ISO time). |
| `POST /events/{id}/reclaim` | `{stale_after_seconds}` | `200 {claimed:true}` only if the record is still `received` **and** `received_at` is at least that old (and the store then refreshes `received_at`); otherwise `409 {claimed:false, record}`. **Must be atomic.** (A successful claim should also return the `record`; the redrive endpoint uses it.) |
| `GET /events?status=&older_than_seconds=&limit=` | none | `200 {events:[record,...]}`, oldest first. Used only by the reconcile job: `status=received` with the age limit, and `status=pending_manual` with no age limit. |
| `POST /events/{id}/status` | `{status:"fulfilled"\|"backordered"\|"pending_manual", reason}` | `200`. Idempotent. |

**Inventory**: `POST /reserve` `{order_id, line_items, idempotency_key}` → `2xx` reserved; `409` not enough stock; anything else = service trouble. De-duplicate on `idempotency_key`.

**Inventory release** (only used when `RELEASE_RESERVATION_ON_OMS_FAILURE` is not `off`): `POST /release` `{order_id, idempotency_key:"release:<order_id>", reason:"oms_create_failed"}` → `2xx` released; `404` = nothing was held (also counted as released). Must be idempotent, and a later `/reserve` for the same order must reserve again (a re-driven order does that).

**Order system**: `POST /orders` `{order_id, amount, currency, line_items, customer_email, idempotency_key}` → `2xx` created. De-duplicate on `order_id` / `idempotency_key`.

**Customer notices**: `POST /send` `{to, template, order_id}` with `template` `order_confirmation` or `backorder_notice`. Best effort.

**Security log**: `POST /security` `{type:"webhook_verification_failed", reason, at}`. Best effort.

**Slack**: `POST` `{text}` to `SLACK_WEBHOOK_URL`.

## Failure behavior

| What breaks | What happens |
|---|---|
| Bad / missing / stale signature or timestamp | 401 generic; reason in the security log; nothing recorded. |
| Bad content after a valid signature | 400; nothing recorded. |
| Event store down at receipt | 503 `retry:true`; nothing reserved or created. |
| Same `order_id`, same contents again | 200 `duplicate`; nothing reprocessed. |
| Same `order_id`, different contents | 409 `order_conflict`. |
| Out of stock (409) | `backordered`; customer gets a backorder notice. |
| Inventory down / erroring | One retry, then `pending_manual`; ops paged; customer not told it is backordered. |
| Order system down / erroring | One retry, then `pending_manual` (inventory stays reserved); ops paged; no confirmation. |
| Order system fails and release is on | The order stays `pending_manual`; the reservation is released (one retry) per `RELEASE_RESERVATION_ON_OMS_FAILURE`, and the reason says so. A release that fails is alerted separately ("stock stays held until someone releases it"). With `rejected`, an uncertain failure (no answer, 5xx) keeps the reservation, because the order may have been created. |
| Order system rejects (4xx) | `pending_manual` immediately (no retry). |
| Final status cannot be saved | One retry, then ops alert (event may still show `received`). |
| Processing dies after the reply | Event stays `received`; the sender's next retry after `RECLAIM_AFTER_SECONDS` takes it over. |
| Slack / notify / security log down or blank | No effect on orders. |
| Invalid CONFIG | 500 `webhook_misconfigured` naming every bad setting. |

## Verification

Tested end to end on self-hosted n8n 2.35.7 against in-memory reference implementations of the services above. The original workflow was run first as a baseline: on current n8n it returns an empty `200` (it loads `crypto` and reads `$env` in a Code node, and both are blocked), so nothing works out of the box. By reading, it also checked the signature against re-serialised JSON rather than the raw bytes, never wrote an idempotency record, treated a failed idempotency lookup as "new", and sent customers a backorder notice when inventory was merely down.

- happy path: one reservation and one OMS order, each with an idempotency key; one confirmation; no alerts
- signature over odd formatting and non-ASCII bodies; `sha256=` prefix
- signature presets: Stripe (`t=`, `v1=`, several v1, a v0 ignored, 12 kinds of rejection including a changed timestamp, a stale one and a signature that skips the timestamp), GitHub (`sha256=` required, no timestamp needed, SHA-1 header refused) and Shopify (base64 compared exactly, a hex digest of the same bytes refused) with a real-shaped Shopify order mapped end to end; a re-delivery of the same signed request is a duplicate in all three; a generic body-only base64 sender; unknown and inconsistent signature settings refused with a clear 500; deliberately weakening five of these rules (timestamp window, several-signature handling, base64 case, required prefix, scheme check) made a test fail
- nine kinds of rejected request (no/short/wrong/other-secret signature, no/non-numeric timestamp, altered body, 10-minute-old and 10-minute-future timestamps): generic 401, specific reason logged, nothing created
- eleven kinds of bad content: 400, nothing recorded
- re-delivery after completion; six simultaneous deliveries (one accepted); same id with different contents (409)
- out of stock; inventory down (pending_manual, retried, no backorder email); transient inventory and OMS errors retried; OMS down; OMS 422 not retried
- event store down at receipt (503, then the sender's retry works); status write failing (retried, ops alerted)
- an abandoned `received` event taken over; a fresh one left alone; four simultaneous takeovers produce one
- the reply arrives in about a second while fulfilment takes 3.5 s
- no email, notify down; custom header names and tolerance; service API keys; no Slack / notify / security log; broken CONFIG

Not tested: a real event store, inventory or order system; real deliveries from Stripe, GitHub or Shopify (the presets follow each vendor's documented format and were tested with signatures computed by the test script, not with live webhooks); n8n 1.x; queue mode; high volume.

## Reconcile job (stuck orders)

If processing dies after the `200` reply, the event stays `received` until the sender retries. Some senders never retry, so the workflow can look for those orders itself. **It is off by default.** Set `RECONCILE_ENABLED=true` and a `RECONCILE_AUTH_TOKEN` to use it.

- **Schedule: Reconcile** runs every 15 minutes (edit the interval on that node). `POST /webhook/orders/reconcile` with header `x-reconcile-token` runs the same job on demand and returns a summary.
- It lists `received` events older than `RECONCILE_STALE_SECONDS` and `pending_manual` events from the event store.
- `RECONCILE_ACTION=alert` only tells ops (Slack). `RECONCILE_ACTION=redrive` also re-runs each stuck `received` order: the job calls this workflow's own `POST /webhook/orders/redrive {"order_id": "..."}` (same token), which takes the order over through the store's atomic `reclaim` and runs the normal fulfilment chain (inventory, order system, status, customer notice, ops alert). Calling `redrive` by hand works too.
- **`pending_manual` orders are never re-run.** They need a person, so every run only lists them in the Slack digest (it repeats each run until someone resolves them).
- Safe by construction: only one caller can win the `reclaim`, so overlapping runs, a manual call and a sender retry cannot process an order twice. The workflow re-checks the status and age of everything the store returns, so a store that ignores the filters cannot make it re-run a finished order. If either list call fails, nothing is re-run and ops are told.
- A wrong or missing token is `401`, a disabled job `403 reconcile_disabled`, and bad settings `500 reconcile_misconfigured` listing every problem. Order intake is unaffected by any of these.
- Downstream services must still honour the idempotency key: a re-run repeats inventory and order creation.

## Releasing the reservation

By default a failed order creation leaves the inventory reservation held and the alert says so. Set `RELEASE_RESERVATION_ON_OMS_FAILURE` to give the stock back automatically:

- `rejected` (recommended if you turn it on): only after a definite refusal from the order system (a 4xx other than 408/429). The order system did not create the order, so the stock is safe to free.
- `any`: after any failure. If the order system answered with an error but had already created the order, the stock would be freed while the order exists. Use this only if your order system cannot do that.

The order itself stays `pending_manual` either way; a person still decides what to do with it. The release call is retried once, a 404 from inventory counts as released, and a release that fails is alerted. A re-driven order reserves again.

## Known limits

- **Re-driving stuck orders is optional and off by default.** Turn on the reconcile job (see "Reconcile job") to alert on, or re-run, orders stuck in `received`. Orders in `pending_manual` are only ever reported.
- **Reprocessing relies on your downstream services.** A takeover runs inventory and order creation again, so they must honour the idempotency key.
- **A reservation is released only if you switch that on** (`RELEASE_RESERVATION_ON_OMS_FAILURE`). The default leaves it held and the alert says so. Even when on, an uncertain failure keeps it under `rejected`, and the order still needs a person.
- **Four signature schemes only.** `generic` (configurable header, hex or base64, with or without a timestamp), `stripe`, `github` and `shopify`. A sender with another format (asymmetric signatures, a different hash, a signature over something other than the body) needs a small edit to `Prepare Verification` and `Check Request`.
- **GitHub and Shopify signatures carry no timestamp**, so there is no replay window for them; a replay is neutralised by the order record only (the same order id with the same body is a duplicate). Stripe and the generic scheme keep the window.
- **Stripe events are not orders.** The preset only checks the signature; Stripe event payloads do not carry line items, so map them yourself (for example from checkout-session metadata) in Order Mapping.
- **No rate limiting.** Unsigned traffic is cheap to reject but still fills the security log; rate-limit in front of n8n.
- **Retries are immediate and single.** One retry with no back-off; a longer outage becomes `pending_manual`.
- **Notices are best effort** and are not retried.
- **The duplicate reply reports the current state**, not the original response body.
- n8n's own `422` for malformed JSON includes a stack trace in its body; put a gateway in front if that matters.
