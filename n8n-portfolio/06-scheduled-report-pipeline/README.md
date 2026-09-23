# 06 — Scheduled Report Pipeline

## Business problem
A weekly ops report aggregates data from several internal systems, renders
it to PDF, archives it, and emails it to stakeholders. The naive version
breaks in predictable ways: a slow data source delays or corrupts the
whole report, a PDF render timeout loses the run entirely, one bad email
address blocks delivery to everyone else, and a manual re-run (e.g. to
backfill after fixing an outage) either does nothing useful or sends a
duplicate. This workflow treats each of those as a first-class case
rather than an afterthought.

## Architecture
Two triggers converge on one pipeline: a **weekly schedule** (computes the
previous Mon–Sun automatically) and a **manual/backfill webhook** (for
on-demand runs, with `force: true` to bypass the duplicate guard).

1. **Idempotency check** — a period already marked `sent` is skipped
   (unless forced).
2. **Fetch three data sources in parallel** (sales, support, infra), each
   independently retried; each response is **namespaced** before merging
   siblings together, specifically to avoid one source's `statusCode`
   silently overwriting another's during the merge.
3. **Assess completeness** — if *every* source is down, abort and page
   ops rather than emailing an all-placeholder report. If *some* are
   down, the report still generates with those sections explicitly
   marked unavailable.
4. **Render HTML → PDF** via Gotenberg's
   `POST /forms/chromium/convert/html` endpoint.
5. **Archive and distribute run independently** — archiving to the
   report store and emailing recipients are unrelated concerns and
   shouldn't gate each other.
6. **Per-recipient delivery** — each email is sent and tracked in
   isolation, aggregated back into a summary; a bad address becomes a
   `partial_failure` status, not a whole-batch failure.
7. **Every termination point checks which trigger fired** — the schedule
   trigger has no caller to respond to; only the manual/webhook path
   sends an HTTP response back. The scheduled path just ends quietly.

## Edge cases handled
- **Merge field collisions** — three sibling API responses are wrapped
  under unique keys (`sales`/`support`/`infra`) *before* being combined,
  since a naive `combineByPosition` on three raw responses sharing a
  `statusCode` field would let each merge clobber the last one's status.
  This is a distinct failure mode from the request/response merge pattern
  used elsewhere in this portfolio (workflows 01–05) — same tool, a
  different reason to need it.
- **Partial data availability** — a down source doesn't block the report;
  its section is explicitly labeled "unavailable," which is a different
  fact from "zero activity" and must never be conflated.
- **Total data unavailability** — aborts entirely rather than sending a
  content-free report, and pages ops.
- **PDF generation is genuinely slow** — a longer timeout (30s) and its
  own retry budget, separate from the faster API calls elsewhere in the
  pipeline; a failure here is reported distinctly from a data-fetch
  failure.
- **Binary data through a Code-node fan-out** — the PDF's binary payload
  doesn't automatically follow when one item is split into N recipient
  items; it's explicitly copied onto each output item.
- **One bad recipient doesn't block the rest** — per-recipient isolation,
  same principle as workflow 03's per-record ETL processing, applied to
  a notification fan-out instead of a data pipeline.
- **Empty/failed subscriber list** — falls back to a known ops address
  rather than the report silently reaching nobody.
- **Archive and delivery are independent failure domains** — one failing
  never blocks or is blocked by the other; each gets its own ops
  notification with its own specific failure reason.
- **Dual-trigger response handling** — every terminal branch explicitly
  checks `triggered_by` before deciding whether to send an HTTP response
  (manual) or just end (scheduled), rather than assuming one trigger
  type throughout.
- **State surviving external calls** — the same `Merge(combineByPosition)`
  pattern from workflows 01–05, applied nine times across this pipeline.

## Environment variables expected
`SALES_API_URL`, `SUPPORT_API_URL`, `INFRA_API_URL`, `GOTENBERG_API_URL`,
`REPORT_STORE_API_URL`, `DISTRIBUTION_API_URL`, `NOTIFY_API_URL`,
`SLACK_WEBHOOK_URL`, `FALLBACK_RECIPIENT_EMAIL`
