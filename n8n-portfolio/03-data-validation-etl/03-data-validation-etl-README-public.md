# 03 — Data Validation ETL Pipeline

A single n8n workflow that takes batches of records from a partner or vendor feed, validates and cleans every record against a schema you define, loads the good ones to a destination API, and sends everything that failed to a dead-letter store with the reason. The reply lists what happened to each record.

Feeds are never clean: bad emails, missing fields, duplicate rows, impossible dates. A naive pipeline either fails the whole batch on the first bad row or silently swallows it. This one handles each record independently, tells a bad record apart from a broken destination, protects a struggling destination from being hammered, and only alerts ops about system trouble, never about bad source data.

## Design principle

Nothing is hardcoded. The record schema (field names, types, rules, defaults) is a JSON value in the `CONFIG` node, and so are the service URLs, retry and breaker settings, batch limits and time budget. The workflow talks to a destination API, an optional shared circuit breaker, an optional dead-letter store and an optional Slack webhook. Any backend that honours the contracts below works.

## Request

`POST /webhook/etl/ingest`

```json
{ "batch_id": "batch-001", "source": "partner-a", "records": [ { "id": "1", "email": "ann@example.com", "amount": 19.99, "created_at": "2026-09-30T12:00:00Z" } ] }
```

| Field | Required | Meaning |
|---|---|---|
| `batch_id` | yes | 1–128 characters: letters, digits and `. _ : -`. Used in idempotency keys and dead letters. |
| `source` | no | Free text label (max 200 characters). Default `unknown`. |
| `records` | yes | 1 to `MAX_BATCH_RECORDS` (500) records. Each is validated against `SCHEMA_JSON`. |

If `CONFIG.ETL_AUTH_TOKEN` is set, callers must send it as header `x-etl-token`.

## Responses

| Status | Body | When |
|---|---|---|
| 200 | `{batch_id, status:"completed"\|"partial"\|"failed", counts, dead_letter, breaker, results:[...]}` | The batch was processed (see below). HTTP stays 200 even when records failed: the per-record results are the answer. |
| 400 | `{error:"batch_validation_failed", details:[...]}` | Bad envelope: nothing was touched. |
| 401 | `{error:"unauthorized"}` | Missing or wrong `x-etl-token`. |
| 500 | `{error:"etl_misconfigured", details:[...]}` | Invalid CONFIG or schema (every problem is listed). |

In the 200 body:

- `status`: `completed` (everything loaded), `partial` (some loaded), `failed` (nothing loaded).
- `counts`: `{total, loaded, invalid, rejected, load_failed, skipped}`.
- `dead_letter`: `saved`, `failed` (could not be saved, ops alerted), `not_needed`, or `not_configured`.
- `breaker`: `{tripped, open_at_start, status_check:"ok"\|"unreachable"\|"not_configured"}`.
- `results`: one entry per record, in input order: `{row, record_id, status, failure_type?, reasons?}`. The data itself is **not** echoed back.

Record statuses:

| Status | Meaning | Dead-lettered | Pages ops |
|---|---|---|---|
| `loaded` | Stored by the destination. | no | no |
| `invalid` | Failed validation (or a duplicate id). Bad source data. | yes | no |
| `rejected` | The destination refused this record (4xx). A data problem, not an outage; not retried and it never trips the breaker. | yes | no |
| `load_failed` | The destination failed (no response, 5xx, 408, 429) after the retries. | yes | yes |
| `skipped` | Not attempted: the breaker stopped the batch, or the time budget ran out. | yes | yes |

Malformed JSON never reaches the workflow: n8n itself rejects it with a `4xx`.

## Replay (dead letters back through the pipeline)

Records that were not loaded are saved to the dead-letter store. Once the cause is fixed (the destination is back, you corrected a rule or the schema, the destination now accepts the data), `POST /webhook/etl/replay` re-runs them. **It is off by default**: set `REPLAY_ENABLED=true` and a `REPLAY_AUTH_TOKEN` (8+ characters, sent as header `x-replay-token`; a separate secret from `ETL_AUTH_TOKEN`). It needs `DEAD_LETTER_API_URL`.

Body (all optional, and `ids` cannot be combined with the others):

| Field | Meaning |
|---|---|
| `ids` | Replay exactly these dead letters (ids from the store or from an earlier reply). Unknown or already-resolved ids are ignored. |
| `batch_id` | Only dead letters from that original batch. |
| `failure_type` | Only `validation`, `destination_rejected`, `load`, `circuit_open` or `time_budget`. For example, after an outage: `{"failure_type":"load"}` and `{"failure_type":"circuit_open"}`. |
| `limit` | At most this many (never more than `REPLAY_MAX_PER_RUN`). The oldest go first. |

What it does: it lists the open dead letters, **claims each one** through the store's atomic claim call (two replays at once can never run the same record), and sends the **original record** through the normal chain: validation, `Custom Rules`, `Enforce Rules`, the loader (retries, breaker, time budget). The idempotency key is the one of the **original batch** (`<original batch_id>:<record id>`), so a record that actually reached the destination (and only its reply was lost) is recognised and not stored twice. Then it tells the store the outcome of each record: loaded = resolved; still failing = stays open with the new reason and one more attempt; skipped because the breaker is open = released, not counted as an attempt. A replay **never files new dead letters**.

The reply has the usual `counts` and `results` (each result also names its `dead_letter_id`), a fresh `batch_id` starting `replay-`, and a `replay` block: `{claimed, skipped_already_claimed, loaded, still_open, released, resolve_failed}`. `status:"nothing_to_replay"` means there was nothing open to claim. Ops get one Slack line per replay.

| Status | Body | Meaning |
|---|---|---|
| 200 | like a batch reply, plus `replay` | The replay ran (see `counts`; HTTP stays 200 even if some records still fail). |
| 200 | `{status:"nothing_to_replay", claimed:0, skipped_already_claimed}` | Nothing open, or everything matching was already claimed by another replay. |
| 400 | `{error:"replay_validation_failed", details}` | Unknown field, bad `ids`/`batch_id`/`failure_type`/`limit`, or `ids` combined with filters. |
| 401 | `{error:"unauthorized"}` | Missing or wrong `x-replay-token`. |
| 403 | `{error:"replay_disabled"}` | `REPLAY_ENABLED` is not true. |
| 500 | `{error:"replay_misconfigured", details}` | Enabled with no/short token, no `DEAD_LETTER_API_URL`, a bad `REPLAY_MAX_PER_RUN`, or `REPLAY_LEASE_SECONDS` not larger than `MAX_LOAD_SECONDS`. |
| 502 | `{error:"replay_unavailable"}` | The dead-letter store could not be read; nothing was run. |

If a record loaded but the store could not be told (`resolve_failed`), ops are told which dead letter it is; it appears as open again when its claim lease runs out, and replaying it again is harmless because of the idempotency key.

## Architecture

```
POST /etl/ingest
  → auth (optional) + CONFIG/schema check + envelope validation            (401 / 500 / 400)
  → validate and clean EVERY record against SCHEMA_JSON (nothing loaded yet)
  → Custom Rules (your transforms / rules / payload) → Enforce Rules (guard)   (500 if Custom Rules cannot run)
  → LOAD the valid records one at a time:
        check the shared breaker (if configured)    open → skip everything
        POST each record with an idempotency key    transient failure → retry with back-off
                                                    4xx → rejected (data problem)
                                                    N failures in a row, or breaker opens → skip the rest
                                                    time budget used up → skip the rest
  → ONE dead-letter call for everything that was not loaded (1 retry)
  → reply with per-record results
  → (after the reply) one Slack message per operational problem
```

1. **Everything is validated before anything is loaded**, and each record stands alone. Row order is preserved in the reply.
2. **Schema-driven.** `SCHEMA_JSON` lists the fields: `id`, `string`, `email`, `number`, `integer`, `currency`, `date`, `boolean`, with `required`, `default`, `min`, `max`, `decimals`, `maxLength`, `pattern`, `enum`. Values are cleaned as they are checked (trimmed, e-mails lower-cased, currencies upper-cased, numbers rounded, dates converted to UTC ISO).
3. **Real dates only.** Dates must be ISO 8601 (`2026-09-30`, or with a time and optional offset). `2026-02-30`, month 13, hour 25, `yesterday` and `1` are all rejected. JavaScript's own date parser would accept some of them and silently roll others over.
4. **Duplicates inside a batch.** The first record with an id wins; later ones (even with different whitespace) are rejected with the row of the first.
5. **One loader node with memory.** n8n runs node by node over all items, so a breaker check placed before a load node sees the same state for all 500 records: the original could never stop mid-batch. The loader keeps state between records instead.
6. **Bad data is not an outage.** A 4xx from the destination marks that record `rejected` and moves on. It is never retried and never counts toward the breaker. (In the original, 8 records the destination disliked would open the breaker and block the good ones.)
7. **Retries with back-off** for transient failures (no response, 5xx, 408, 429): `LOAD_MAX_ATTEMPTS` tries, doubling the wait each time.
8. **Breaker, two layers.** Consecutive failures within the batch (`BREAKER_TRIP_AFTER`) stop the rest. If you also run a shared breaker (`BREAKER_API_URL`), it is checked before the batch and after each failure, and failures are reported to it. A breaker that cannot be reached is ignored (reported as `unreachable`).
9. **Time budget.** After `MAX_LOAD_SECONDS` the rest are skipped, so the caller is never held for ever.
10. **Idempotency.** Every load carries `idempotency-key: batch_id:id`. A retry, or a resent batch, therefore does not create duplicates in an idempotent destination.
11. **One dead-letter call per batch**, with the original records and reasons, and one retry. If it still fails the reply says so and ops is alerted. (The original made one call per record and ignored every failure.)
12. **No data echoed.** The reply carries ids, statuses and reasons, not the records (which can hold personal data).
13. **Alerts only for operations.** Load failures, skips and lost dead letters page Slack; bad source data does not. Alerts are sent after the reply.

## Edge cases handled

- Null, string and array "records"; missing and null fields; numeric ids (stored as strings).
- Amounts like `1.005` rounded correctly (`1.01`), not by naive floating-point multiplication.
- Unknown fields dropped (or passed through with `ALLOW_EXTRA_FIELDS`).
- A transient destination error on a record: retried, and the other records are unaffected.
- A destination that is down: five records fail (15 requests), the remaining seven are skipped, not 36 requests.
- The shared breaker opening part-way through a batch, or already open.
- Breaker lookup, dead-letter store or Slack failing.
- Re-sending a whole batch.
- A 500-record batch.
- Missing optional services (no Slack, no shared breaker, no dead-letter store).

## Configuration (`CONFIG` node)

| Key | Default | Meaning |
|---|---|---|
| `SCHEMA_JSON` | id, email, amount, currency, created_at | The record schema (below). |
| `ID_FIELD` | `id` | The schema field that identifies a record (duplicate detection, idempotency key). |
| `ALLOW_EXTRA_FIELDS` | `false` | `true` passes fields that are not in the schema through to the destination. |
| `MAX_BATCH_RECORDS` | 500 | Larger batches are rejected with 400. |
| `ETL_AUTH_TOKEN` | blank | If set, callers send it as `x-etl-token`. |
| `DESTINATION_API_URL` | `http://localhost:4900` | Base URL of the destination. |
| `BREAKER_API_URL` | `http://localhost:4900` | Shared circuit breaker. Blank = only the in-batch breaker. |
| `DEAD_LETTER_API_URL` | `http://localhost:4900` | Dead-letter store. Blank = failed records appear only in the reply. |
| `SLACK_WEBHOOK_URL` | blank | Slack-compatible incoming webhook. Blank = no alerts. |
| `SERVICE_HEADERS_JSON` | `{}` | Headers sent to the destination, breaker and dead-letter store, e.g. `{"x-api-key":"..."}`. |
| `LOAD_MAX_ATTEMPTS` | 3 | Tries per record for transient failures. |
| `LOAD_RETRY_BACKOFF_MS` | 500 | First wait between tries; doubles each time. |
| `BREAKER_TRIP_AFTER` | 5 | Consecutive records that fail (after their retries) before the rest are skipped. |
| `LOAD_TIMEOUT_MS` | 10000 | Timeout per destination request. |
| `MAX_LOAD_SECONDS` | 120 | Time budget for the load phase. Keep it under your caller's timeout. |
| `REPLAY_ENABLED` | `false` | Turns `POST /webhook/etl/replay` on. Off = 403. |
| `REPLAY_AUTH_TOKEN` | blank | Required (8+ characters) when enabled; sent as header `x-replay-token`. |
| `REPLAY_MAX_PER_RUN` | 50 | Most dead letters replayed per call (1 up to `MAX_BATCH_RECORDS`, at most 200). |
| `REPLAY_LEASE_SECONDS` | 600 | How long a claimed dead letter stays claimed. Must be larger than `MAX_LOAD_SECONDS`. |
| `REQUEST_TIMEOUT_MS` | 10000 | Timeout for breaker, dead-letter and Slack calls. |

### The schema

```json
[
  {"name": "id",         "type": "id",       "required": true},
  {"name": "email",      "type": "email",    "required": true},
  {"name": "amount",     "type": "number",   "required": true, "min": 0, "decimals": 2},
  {"name": "currency",   "type": "currency", "default": "USD"},
  {"name": "created_at", "type": "date",     "required": true}
]
```

| Type | Accepts | Cleaned to |
|---|---|---|
| `id` | non-empty string or whole number, max 128 characters | trimmed string |
| `string` | string; optional `maxLength` (default 1000), `pattern` (regex), `enum` | trimmed |
| `email` | valid address, max 254 characters | trimmed, lower-case |
| `number` | finite number; optional `min`, `max`, `decimals` | rounded to `decimals` |
| `integer` | whole number; optional `min`, `max` | as is |
| `currency` | 3 letters | upper-case |
| `date` | ISO 8601 date or date-time (no offset = UTC) | UTC ISO string |
| `boolean` | `true` / `false` | as is |

Any field: `required` (default `false`), `default` (used when the value is missing or null), `from` (the name the sender uses, if it differs from `name`; the sender's name is what must be present).

## Service contract

All JSON. `SERVICE_HEADERS_JSON` is sent on every call to the destination, breaker and dead-letter store.

**Destination**: `POST /records` with the cleaned record as the body and header `idempotency-key: batch_id:id`. `2xx` = stored. `4xx` = refused (this record). `5xx`, `408`, `429` or no response = trouble. **Should de-duplicate on the idempotency key (or upsert on the id).**

**Shared breaker** (optional): `GET /status` → `200 {open:true|false}`; `POST /record-failure` `{reason, status, batch_id}`.

**Dead-letter store** (optional): `POST /dead-letters` `{batch_id, source, received_at, items:[{row, record_id, status, failure_type, reasons, record}]}` → `2xx`. `record` is the original record as received.

**Dead-letter store, for replay** (only if you turn replay on). The store gives every saved item an `id` and a `state` (`open` or `resolved`):
- `GET /dead-letters?state=open&batch_id=&failure_type=&limit=` → `200 {items:[{id, state, batch_id, record_id, failure_type, status, reasons, record, attempts}]}`, oldest first, not including items that are currently claimed.
- `GET /dead-letters/{id}` → `200 {item}` or `404`.
- `POST /dead-letters/{id}/claim` `{lease_seconds}` → `200 {claimed:true, item}` only if the item is `open` and not claimed by anyone else; otherwise `409 {claimed:false}`. **Must be atomic** (one conditional update). The workflow uses the `item` in the reply.
- `POST /dead-letters/{id}/resolve` `{outcome:"loaded"|"failed"|"released", status?, failure_type?, reasons?, replay_batch_id}` → `2xx`. `loaded` marks it resolved; `failed` keeps it open, adds one attempt and stores the new reason; `released` just drops the claim. Every outcome releases the claim.

The workflow re-checks everything the store returns (state, batch, failure type), so a store that ignores a filter cannot make it replay the wrong record.

**Slack**: `POST` `{text}` to `SLACK_WEBHOOK_URL`.

## Failure behavior

| What breaks | What happens |
|---|---|
| Bad envelope | 400; nothing touched. |
| A record fails validation | `invalid`; dead-lettered; ops not paged. |
| Destination refuses a record (4xx) | `rejected`; not retried; no breaker effect; dead-lettered; ops not paged. |
| Destination fails on a record | Retried with back-off, then `load_failed`; breaker told; dead-lettered; ops paged. |
| `BREAKER_TRIP_AFTER` failures in a row | The rest are `skipped` (`circuit_open`); ops paged once. |
| Shared breaker opens or is open | Same: skipped. |
| Shared breaker unreachable | Ignored; the load proceeds. |
| Time budget used up | The rest are `skipped` (`time_budget`); dead-lettered for re-submission; ops paged. |
| Dead-letter store down | One retry, then `dead_letter:"failed"` in the reply and an alert saying how many records exist only in the reply. |
| Replay: dead-letter store cannot be read | `502 replay_unavailable`, nothing run. |
| Replay: a record loads but the store cannot be told | The reply lists it under `resolve_failed` and ops are told which dead letter; replaying it again is harmless (same idempotency key). |
| Slack down or blank | No effect. |
| Invalid CONFIG or schema | 500 `etl_misconfigured` naming every bad setting. |

## Verification

Tested end to end on self-hosted n8n 2.35.7 against in-memory reference implementations of the services above. The original workflow was run first as a baseline: bad envelopes got their 400, but any real batch returned an empty `200`, because every service call read `$env`, which is blocked. By reading, it also checked the breaker for all records before loading any (so it could never open mid-batch), counted destination 4xx responses on bad data as outages, never retried despite saying so, made one dead-letter call per record and ignored their failures, accepted dates such as `1` and rolled others over, and echoed full records (e-mail addresses) back in the reply.

- happy path; cleaning (trim, lower-case, rounding `1.005` to `1.01`, `eur` to `EUR`, numeric id, date-only, `+02:00` offset converted, unknown field dropped)
- 20 mixed records in one batch: 5 loaded, 15 invalid with specific reasons (impossible dates, month 13, hour 25, `yesterday`, missing fields, negative and string amounts, bad currency, null/string/array records, duplicate id with different whitespace); one dead-letter call; no page
- nine kinds of bad envelope including 501 records, and malformed JSON
- two transient failures retried, same idempotency key each time; a record that always fails (3 attempts, load_failed, dead-lettered, one alert); eight records refused with 422 in a row (no retries, breaker untouched, the rest still load)
- destination down: 15 requests instead of 36, seven records skipped; the shared breaker opening mid-batch, already open, and unreachable
- dead-letter store failing once (retried) and down (reply says so, ops alerted)
- the same batch sent twice; a 500-record batch
- a different schema (pattern, integer limits, enum, boolean, defaults, extra fields, a different `ID_FIELD`)
- caller token and service API keys; no Slack / shared breaker / dead-letter store; a 2 s time budget; broken CONFIG and schema

Not tested: a real destination, breaker or dead-letter service; n8n 1.x; queue mode; batches over 500 records; a destination slower than the timeouts in a long batch beyond the time-budget test.

## Known limits

- **Synchronous.** The caller waits while the valid records are loaded one at a time. 500 records took about 1.3 seconds against a local mock; against a real API it is roughly records times latency, capped by `MAX_LOAD_SECONDS`. If your sender's timeout is shorter, lower the cap or send smaller batches.
- **The loader is one Code node.** That is deliberate (the breaker needs state between records, which n8n's node-by-node flow does not give), but it is less visual than a chain of nodes.
- **Sequential loading.** One record at a time keeps the breaker honest and the destination calm, and is slower than parallel loading.
- **Resending a batch re-attempts every record.** It is safe only if the destination is idempotent on the key or the record id. There is no batch-level "already processed" memory.
- **No half-way recovery.** If n8n dies during the load, the caller gets an error and the batch must be resent; the records already loaded are protected only by the destination's idempotency.
- **Replay is a request, not a schedule.** Something (you, a cron job, a person reviewing the dead letters) has to call `POST /webhook/etl/replay`; the workflow does not retry dead letters on its own. A replay only helps when the cause has been fixed, and a record that is invalid under the current schema simply stays open with its new reason.
- **Dates without a time zone are treated as UTC.**
- **Rounding is half away from zero** at the configured decimals; the workflow does not do currency-aware arithmetic.
- **The breaker is simple:** consecutive failures within one batch plus an optional shared one. It does not half-open or probe by itself.
- n8n's own `422` for malformed JSON includes a stack trace in its body; put a gateway in front if that matters.
