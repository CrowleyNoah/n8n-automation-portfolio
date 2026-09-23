# 03 — Data Validation ETL Pipeline

## Business problem
Partner/vendor feeds send batches of records (orders, transactions,
whatever the source system produces) that are never perfectly clean: bad
emails, missing fields, duplicate rows, unparseable dates. A naive "load
everything, fail the batch on the first bad row" pipeline either loses
good data or silently swallows bad data. This workflow validates,
transforms, and loads each record independently, routes every failure to
a durable dead-letter store with a reason, and protects the destination
system from being hammered when it's already struggling.

## Architecture

`POST /webhook/etl/ingest` → `{ batch_id, source?, records: [...] }`

1. **Validate the envelope** — `batch_id` required, `records` must be a
   non-empty array capped at 500 items per batch.
2. **Detect duplicates** — records sharing an `id` within the batch are
   flagged (first occurrence wins); flagged copies fail validation later
   with an explicit reason instead of silently overwriting each other at
   the destination.
3. **Split into per-record items** so one bad record can never take down
   the rest of the batch.
4. **Schema validation** per record (required fields, email format,
   non-negative amount, optional 3-letter currency, date presence).
5. **Transform** valid records (trim/lowercase email, round currency,
   parse and normalize the date) — a bad date throws inside a `try/catch`
   and becomes a typed transform failure, not a crashed execution.
6. **Circuit breaker check** before every load attempt — if the
   destination has been failing repeatedly (tracked externally), the
   record is skipped rather than piling another failing request onto an
   active outage.
7. **Load** to the destination API with retry/backoff on transient
   failures; a final failure records a breaker-failure signal and the
   record is dead-lettered.
8. **Every non-loaded record** (invalid, transform error, skipped, load
   failed) is logged to a durable dead-letter store as a side effect —
   this never blocks the batch response even if the dead-letter store
   itself is down.
9. **Aggregate** all per-record outcomes back into one batch result and
   **summarize** counts (loaded / invalid / load_failed / skipped).
10. **Ops alerting** — only *operational* failures (load_failed, skipped)
    page Slack; bad source data (invalid) is logged for review but
    doesn't page anyone, since it isn't a system problem.
11. **Respond once** with the full per-record breakdown.

## Edge cases handled
- **Oversized/malformed batch envelope** — rejected up front with `400`,
  before any record is touched.
- **Duplicate records within a batch** — flagged and rejected explicitly
  rather than silently overwriting each other at the destination.
- **One bad record doesn't fail the batch** — records are split and
  processed independently; failures are isolated and reported per-record.
- **Unparseable dates / malformed field values** — caught in a
  `try/catch` inside the transform step, converted into a typed failure
  instead of throwing.
- **State surviving external calls** — every HTTP call (breaker check,
  load, Slack) is followed by a `Merge (combineByPosition)` that
  recombines the response with the record/summary state the call would
  otherwise discard — the same pattern used in workflows 01 and 02.
- **Circuit breaker** — breaker state is delegated to an external service
  (shared across records/batches, not per-execution) so a struggling
  destination gets a break instead of a batch-sized pile-on.
- **Reading HTTP status codes correctly** — `fullResponse: true` is set
  on the load call specifically because `neverError` alone only exposes
  the parsed response body, not the status code, which is a common n8n
  gotcha worth knowing.
- **Fire-and-forget side effects don't corrupt the main path** — the
  breaker-failure ping and the dead-letter log both run in parallel with
  (not chained before) the node that builds the record's final outcome,
  so their own HTTP responses never overwrite the record's data.
- **Dead-letter store outage** — best-effort; doesn't block the batch
  summary response.
- **Alert fatigue** — bad source data alone never pages ops; only
  destination/system trouble does.

## Environment variables expected
`CIRCUIT_BREAKER_API_URL`, `DESTINATION_API_URL`, `DEAD_LETTER_API_URL`,
`SLACK_WEBHOOK_URL`
