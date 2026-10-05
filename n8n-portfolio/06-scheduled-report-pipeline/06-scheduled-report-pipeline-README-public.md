# 06 — Scheduled Report Pipeline

A single n8n workflow that builds a periodic report from several data sources, renders it to PDF, archives it, and emails it to a subscriber list. It runs on a weekly schedule (the previous Monday to Sunday) and can also be triggered on demand for a backfill: `POST /webhook/reports/generate`.

The interesting part is what happens when things go wrong. A naive version lets a slow source corrupt the whole report, loses the run when the PDF renderer times out, lets one bad address block everyone else, and either does nothing or sends duplicates when you re-run it after an outage. This one claims each period atomically, treats a missing section as different from "no activity", releases the claim when a run fails, and resumes a partly delivered run by emailing only the people who did not get it.

## Design principle

Nothing is hardcoded. The data sources (any number: name, label, URL, headers, timeout, required or optional) are a JSON value in the `CONFIG` node, and so is every service URL, limit and recipient fallback. The workflow talks to four small HTTP services (data sources, report store, subscriber list, mail sender), a Gotenberg PDF renderer and an optional Slack-compatible webhook, defined by the contract below. Swap any of them by changing a URL; no single vendor is assumed.

## Request

`POST /webhook/reports/generate`

```json
{ "period_start": "2026-09-21", "period_end": "2026-09-27", "force": false }
```

| Field | Required | Meaning |
|---|---|---|
| `period_start`, `period_end` | yes | Real ISO dates (`YYYY-MM-DD`), start not after end, at most `MAX_PERIOD_DAYS` (92) days apart, inclusive. Passed to every data source as `?start=&end=`. |
| `force` | no | `true` re-runs a period that was already sent and sends to everyone again. Default `false`. |

If `CONFIG.REPORT_AUTH_TOKEN` is set, callers must send it as the `x-report-token` header. The scheduled path has no caller: it computes the previous Monday to Sunday (UTC dates) itself.

The **run id** is `REPORT_NAME-period_start_period_end` (for example `weekly_ops-2026-09-21_2026-09-27`). It names the exact period, so a custom backfill can never collide with the weekly run.

## Responses

| Status | Body | When |
|---|---|---|
| 200 | `{status:"sent"\|"partial_failure", run_id, period_start, period_end, total, sent, already_delivered, failed_count, failed_recipients, missing_sections, archive_url, warnings?}` | The report was delivered. `partial_failure` means some recipients failed (they are listed); re-running the same request retries only them. `sent` counts this run, `already_delivered` those who got it in an earlier run of the same period. `warnings` can contain `archive_failed`, `fallback_recipient_used`, `invalid_recipients_skipped`, `status_not_recorded`. |
| 200 | `{status:"skipped_already_sent", run_id}` | This period was already sent and `force` is not set. Nothing was fetched or sent. |
| 400 | `{error:"validation_failed", details:[...]}` | Bad input. Nothing was claimed or called. |
| 401 | `{error:"unauthorized"}` | Missing or wrong `x-report-token`. |
| 409 | `{error:"run_in_progress", run_id, status}` | Another run holds this period right now. |
| 500 | `{error:"report_misconfigured", details:[...]}` | Invalid CONFIG or `DATA_SOURCES_JSON` (every problem is listed, with the source index). |
| 502 | `{status:"failed_no_data"\|"failed_required_source_unavailable"\|"failed_pdf_generation"\|"failed_no_recipients"\|"failed_too_many_recipients", run_id, detail, retry:true}` | The run stopped before sending anything. The claim is released, so fixing the cause and re-running the same period just works. |
| 502 | `{error:"delivery_failed", status:"partial_failure", ...}` | Every single delivery failed. |
| 503 | `{error:"report_store_unavailable", run_id, retry:true}` | The period could not be claimed, so nothing was fetched or sent. |

Malformed JSON never reaches the workflow: n8n itself rejects it with a `4xx`.

## Async mode: get a job id now, collect the answer later (optional, off by default)

A manual or backfill run waits for the sources, the PDF, the archive and every email. With async mode on, `POST /webhook/reports/generate` can answer straight away with a job id, and the caller picks up the result later. Think of a coat-check ticket: you hand the work over, get a number, and come back to the desk to ask "is it ready?". **Only the manual webhook can become a job**: the weekly schedule has no caller to answer, so scheduled runs always behave exactly as before (even with `ASYNC_MODE=always`), and alert instead.

Turn it on in `CONFIG`:

| Key | Default | Meaning |
|---|---|---|
| `ASYNC_MODE` | `off` | `off`: always synchronous, the flags below are ignored. `optional`: synchronous unless the caller asks (`Prefer: respond-async` header or `?async=true`). `always`: every accepted manual request becomes a job. |
| `JOB_STORE_API_URL` | blank | The job store (contract below). Required unless `off`; must be http or https. |
| `JOB_LEASE_SECONDS` | 900 | How long a job may stay `queued` or `running` before it is reported lost. **Must not be shorter than `PROCESSING_TTL_SECONDS`** (the longest a run is expected to take), or the check refuses to start. |
| `JOB_RESULT_MAX_BYTES` | 1000000 | Largest result stored (at least 100). A bigger result first loses the `failed_recipients` list (`result_trimmed` says so; `failed_count` stays). If it still does not fit, the job fails with `job_result_too_large`. |

**Submit** exactly as before, adding `?async=true` (or the `Prefer` header). Bad requests (400, 401, 500 bad CONFIG) are still refused on the spot and create no job.

| Status | Body | When |
|---|---|---|
| 202 | `{job_id, state:"queued", status_url, retry_after_seconds}` plus `Location` and `Retry-After` headers | Accepted; the run has started. |
| 200 | `{job_id, state, existing:true, status_url}` | You sent an `Idempotency-Key` header and a live or finished job with that key already exists. Nothing runs twice. |
| 503 | `{error:"job_store_unavailable", retry:true}` | The job store could not create the job. **Nothing was started** (no claim, no source fetched, nothing sent), so retrying is safe. |

**Why jobs are keyed only by the `Idempotency-Key` header, not by the period:** re-running a period is a normal way to retry (re-running the same period without `force` retries only the failed deliveries), and a finished job for that period would block it. The run claim already guarantees exactly-once delivery: if you submit the same period twice with no key, both get jobs, the first runs, and the second ends as a **failed** job carrying `409 run_in_progress` (or, later, a succeeded one saying `skipped_already_sent`).

**Collect** with `GET /webhook/reports/jobs?job_id=<id>` (same `x-report-token` header as the generate endpoint, when `REPORT_AUTH_TOKEN` is set):

| Status | Body | When |
|---|---|---|
| 200 | `{job_id, state, created_at, updated_at, result?, result_trimmed?}` | `state` is `queued`, `running`, `succeeded` or `failed`. `result` is exactly what the synchronous call would have returned: `{http_status, body}`. A job is `succeeded` when that status is 2xx. A run that ended `partial_failure` is still a succeeded job: look at `result.body.status` and `failed_recipients`. A `502 failed_no_data` run is a **failed** job carrying the real body. |
| 200 | `{job_id, state:"failed", error:"worker_lost", message}` | Still queued or running past its lease (n8n restarted or the run died). **Submit again**: a lost job never blocks a new one, and the run claim expires after `PROCESSING_TTL_SECONDS`. |
| 400 / 401 | | Missing or malformed `job_id` / wrong token. |
| 404 | `{error:"async_disabled"}` or `{error:"job_not_found"}` | Async is off, or the id is unknown or belongs to another workflow. |
| 503 | `{error:"job_store_unavailable", retry:true}` | The job store is down. |

The status call returns HTTP 200 for any job that exists; the run's own outcome is in `result.http_status` and `result.body`.

**The job store.** A small HTTP service you run (the mock ships a reference version; put the same four calls in front of Redis, Postgres, Supabase and so on). All JSON, states `queued` → `running` → `succeeded` | `failed`. The kind used here is `report`.

| Call | Behaviour |
|---|---|
| `POST /jobs {kind, key?, lease_seconds}` | Creates a job; the **store** generates the unguessable `id`. `201 {id, state:"queued", created_at, expires_at}`. If `key` is given and a job with the same `(kind, key)` is queued, running (not expired) or succeeded, return `200 {…that job, existing:true}` instead. A failed or lost job does not block a new one. |
| `PATCH /jobs/{id} {state:"running", lease_seconds}` | `queued` → `running`, extends `expires_at`. `409` if not queued. |
| `PATCH /jobs/{id} {state:"succeeded"\|"failed", result, result_trimmed?}` | Final write. `409` if already final: the first write wins. |
| `GET /jobs/{id}` | `200 {id, kind, state, created_at, updated_at, expires_at, result?, result_trimmed?}` or `404`. |

Send job-store credentials with `SERVICE_HEADERS_JSON`. If saving the result fails three times the report is still handled, ops get a Slack notice naming the job, and the job is left `running` so the lease reports it lost.

## Architecture

```
trigger  schedule (previous Mon-Sun, UTC dates)   |   POST /webhook/reports/generate
        → auth (manual) + CONFIG/sources sanity + period validation          (400 / 401 / 500)
        → CLAIM the period (atomic)       claimed → carry on (possibly resuming a partial run)
                                          already sent → 200 skipped         in progress → 409
                                          store unreachable → 503
        → fetch every data source (one retry pass for transient failures)
        → assess: all down | a `required` one down → release the claim, 502, alert
                  some down → continue, sections marked "unavailable"
        → render HTML (escaped) → PDF via Gotenberg (3 tries) → verify it really is a PDF   (failure → release, 502)
        → archive the PDF (3 tries)            failure = warning + alert, delivery continues
        → recipients: subscriber list → validate / de-duplicate → fallback if unavailable → skip those who already have it
        → email each recipient (its own call, one retry pass for transient failures)
        → record the outcome  sent | partial_failure  (3 tries)
        → reply (manual runs only)
        → (after the reply) one Slack message per problem
```

1. **Validation first.** Auth (manual), CONFIG, every data source and the period are checked before anything is claimed or called. Impossible dates (`2026-02-30`) and over-long periods are rejected.
2. **Atomic claim.** The report store grants a period to exactly one run at a time. The usual pattern (read the status, work for a minute, write at the end) lets two runs both pass the check, e.g. a schedule firing while someone backfills. The claim carries a TTL (`PROCESSING_TTL_SECONDS`), so a crashed run cannot block the period forever.
3. **A hollow report is never emailed.** If every source is down, or a source you marked `required` is down, the run stops, releases the claim, alerts, and replies 502. Otherwise the report goes out with the missing sections named in the PDF, in the email body and in the alert.
4. **Unavailable is not zero.** A source that failed shows "Data unavailable for this period (reason)"; a source that answered with nothing shows "No data returned for this period". These are different facts and are never mixed up.
5. **HTML is escaped.** Every value from a data source is escaped before it goes into the page, so data can never inject markup into the PDF.
6. **PDF is verified.** Gotenberg is called the way it requires (a multipart file part named `files` called `index.html`), with its own timeout and 3 tries, and the response must start with `%PDF` before anything is archived or sent.
7. **Archive and delivery are separate.** An archive failure does not stop the emails (and vice versa): it becomes a warning in the reply and an alert saying to save a copy by hand.
8. **Recipients are cleaned.** Addresses are trimmed, lower-cased, validated and de-duplicated; invalid ones are skipped and reported. If the subscriber list is unreachable, empty or malformed the report goes to `FALLBACK_RECIPIENT_EMAIL` only (and says so). If there is nobody to send to, or more than `MAX_RECIPIENTS`, nothing is sent: a runaway list is a bug to look at, not a reason to email 40,000 people.
9. **Per-recipient delivery.** Each recipient gets its own call, with the PDF attached to each (the attachment is copied onto every item explicitly; it does not follow a fan-out by itself). One bounced address becomes one failed result, never a failed batch.
10. **Resume, don't resend.** The store remembers who has already been delivered the period. A re-run of a `partial_failure` period emails only the others; `force:true` is the explicit "send to everybody again", with fresh idempotency keys.
11. **Idempotency keys.** Every email carries `idempotency_key = run_id|address` so a mail service can drop a duplicate if a retry (or a crash and re-run) repeats a send that already went out.
12. **Failures are honest and retryable.** A run that cannot deliver is recorded as failed and the claim released; only a run that actually delivered is remembered as `sent`.
13. **Alerts run after the reply.** A slow or dead Slack never delays or changes the response. Scheduled runs have no caller, so for them the alerts are the only signal.

## Edge cases handled

- **Schedule and backfill at the same time.** Both go through the same atomic claim; the second gets `409 run_in_progress` (or `skipped_already_sent`).
- **Five simultaneous requests for one period.** One runs; the other four are told `run_in_progress`. One set of fetches, one PDF, one email per recipient.
- **A claim left behind by a crash.** It expires after `PROCESSING_TTL_SECONDS` and the next run takes the period over.
- **A source that returns an HTML maintenance page, a 404, a 500 or hangs.** All are "unavailable"; the others are unaffected, and transient failures (network, timeout, 5xx, 429) get one more try.
- **Partial delivery.** `partial_failure` plus the exact failed list, an alert, and a safe re-run.
- **Everything bounces.** `502 delivery_failed` (not a cheerful 200).
- **The store forgets the outcome.** The run still replies with what happened plus `status_not_recorded` and an alert; the claim then expires and the mail service's idempotency keys protect against a duplicate.
- **Merge-field collisions and lost state.** Downstream nodes read earlier results by node name, so one HTTP response (with its own `statusCode`, `body`, ...) can never overwrite another's data or the run's state.

## Data sources (`DATA_SOURCES_JSON`)

A JSON array in `CONFIG`; order = order in the report. The default:

```json
[
  {"name": "sales",   "label": "Sales",          "url": "http://localhost:4600/src/sales"},
  {"name": "support", "label": "Support",        "url": "http://localhost:4600/src/support"},
  {"name": "infra",   "label": "Infrastructure", "url": "http://localhost:4600/src/infra"}
]
```

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | 1–64 characters (letters, digits, `.` `_` `-`), unique. |
| `url` | yes | The full URL. The workflow adds `?start=YYYY-MM-DD&end=YYYY-MM-DD` (or `&start=...` if it already has a query string). It must answer with a JSON object or array. |
| `label` | no | Section heading in the PDF. Default: `name`. |
| `headers` | no | Headers sent to this source only, e.g. `{"Authorization":"Bearer ..."}`. |
| `required` | no | `true`: if this source is down the whole run stops instead of sending an incomplete report. Default `false`. |
| `timeout_ms` | no | Per-request timeout. Default `SOURCE_TIMEOUT_MS`. |

How a source's JSON is shown: an object becomes a key/value table (nested objects become nested tables, up to two levels), an array of objects becomes a table (up to 12 columns and 50 rows, with a "more rows not shown" note), an array of values becomes a list. Every value is HTML-escaped. Sources are fetched one after another, so the worst case is about twice the sum of their timeouts.

## Configuration (`CONFIG` node)

| Key | Default | Meaning |
|---|---|---|
| `DATA_SOURCES_JSON` | three sources (above) | The data sources. |
| `REPORT_NAME` | `weekly_ops` | Identifier for this report: part of the run id and the `report` parameter sent to the subscriber service. |
| `REPORT_TITLE` | `Weekly Ops Report` | Heading of the PDF, and the start of the email subject. |
| `REPORT_STORE_API_URL` | `http://localhost:4600` | Base URL of the report store. |
| `GOTENBERG_API_URL` | `http://localhost:4600` | Base URL of Gotenberg (the default is the mock's stand-in). |
| `DISTRIBUTION_API_URL` | `http://localhost:4600` | Base URL of the subscriber list service. |
| `NOTIFY_API_URL` | `http://localhost:4600` | Base URL of the mail sender. |
| `SERVICE_HEADERS_JSON` | `{}` | Headers sent to the report store, subscriber service and mail sender, e.g. `{"x-api-key":"..."}`. |
| `GOTENBERG_HEADERS_JSON` | `{}` | Headers sent to Gotenberg only. |
| `SLACK_WEBHOOK_URL` | blank | Slack-compatible incoming webhook. Blank = no alerts. |
| `REPORT_AUTH_TOKEN` | blank | If set, manual callers must send it as `x-report-token`. |
| `FALLBACK_RECIPIENT_EMAIL` | `ops@example.com` | Used when the subscriber list is unavailable or empty. **Change this.** Blank = no fallback (the run then fails with `failed_no_recipients`). |
| `MAX_RECIPIENTS` | 200 | More recipients than this and nothing is sent. |
| `MAX_PERIOD_DAYS` | 92 | Longest accepted backfill period. |
| `PROCESSING_TTL_SECONDS` | 900 | How long a claim is held. Set it above your slowest legitimate run (fetches + PDF + all emails). |
| `SOURCE_TIMEOUT_MS` | 20000 | Default timeout per data-source request. |
| `PDF_TIMEOUT_MS` | 30000 | Timeout for Gotenberg and for the archive upload. |
| `EMAIL_TIMEOUT_MS` | 15000 | Timeout per email send. |
| `REQUEST_TIMEOUT_MS` | 10000 | Timeout for report-store, subscriber and Slack calls. |
| `ASYNC_MODE` / `JOB_STORE_API_URL` / `JOB_LEASE_SECONDS` / `JOB_RESULT_MAX_BYTES` | `off` / blank / 900 / 1000000 | Optional async mode for manual runs (job id now, result later); see "Async mode" above. |

## Service contract

All JSON unless noted. `SERVICE_HEADERS_JSON` is sent on every report-store, subscriber and mail call. `{run_id}` is URL-encoded.

**Report store**

| Call | Body | Response |
|---|---|---|
| `POST /reports/{run_id}/claim` | `{ttl_seconds, force}` | `200 {claimed:true\|false, record:{status, delivered:[emails]}}`. **Must be atomic.** Grant (`claimed:true`) when there is no record, the status is `failed` or `partial_failure`, a `processing` claim has expired, or `force` is true and nobody holds a live claim. A granted claim becomes `processing` for `ttl_seconds` and keeps `delivered` (cleared when `force` is true). Otherwise `claimed:false` with the current record (`status:"sent"`, or a live `processing`). |
| `POST /reports/{run_id}/archive` | multipart: `pdf` (file), `period_start`, `period_end`, `missing_sections` | `200/201 {archive_url}`. **Idempotent per run_id** (it is retried). |
| `POST /reports/{run_id}/complete` | `{status:"sent"\|"partial_failure"\|"failed", delivered:[emails], failed:[emails], sent_at?, archive_url?, detail?}` | `200`. Stores the outcome and releases the claim. |

**Subscribers**: `GET /subscribers?report=NAME` → `200 {emails:[...]}`. Anything else (or an empty list) means "use the fallback".

**Mail sender**: `POST /send-with-attachment`, multipart: `to`, `subject`, `body`, `idempotency_key`, `attachment` (the PDF file). `2xx` = sent. A `4xx` (rejected mailbox) is final for that recipient; a network error, timeout, `5xx`, `408` or `429` gets one more try. **It should drop a repeat of an `idempotency_key` it has already sent** (return `2xx` without sending again): that is what makes retries and crash recovery duplicate-proof.

**Gotenberg**: `POST /forms/chromium/convert/html` with a multipart file part named `files` called `index.html`; the response is the PDF. This is Gotenberg's own API.

**Alerts**: `POST` `{text}` to `SLACK_WEBHOOK_URL`.

## Failure behavior

| What breaks | What happens |
|---|---|
| A data source is down, slow, returns 4xx/5xx or non-JSON | Marked unavailable in the report; transient failures get one more try; the other sources are unaffected. Alert: "report sent INCOMPLETE". |
| Every source is down, or a `required` one is | 502 `failed_no_data` / `failed_required_source_unavailable`; nothing sent; claim released; alert. |
| Gotenberg fails or returns something that is not a PDF | 3 tries, then 502 `failed_pdf_generation`; nothing archived or sent; claim released; alert. |
| Archive fails | 3 tries, then delivery continues; `archive_failed` warning and an alert to save a copy by hand. |
| Subscriber list unreachable, empty or malformed | Sent to `FALLBACK_RECIPIENT_EMAIL`; `fallback_recipient_used` warning and alert. |
| No valid recipients / more than `MAX_RECIPIENTS` | 502 `failed_no_recipients` / `failed_too_many_recipients`; nothing sent; claim released. |
| Some emails fail | `partial_failure`: the others are delivered; failed list in the reply and the alert; re-running the same period retries only them. |
| All emails fail | 502 `delivery_failed`. |
| Report store unreachable at the claim | 503 `report_store_unavailable`; nothing fetched or sent; alert. |
| Report store rejects the final record | The reply says what happened with `status_not_recorded`; alert; the claim expires on its own. |
| Job store down at submit (async) | 503 `job_store_unavailable`; nothing started, nothing claimed. |
| Job store down when marking "running" (async) | The run still goes ahead; the job just stays `queued` until it finishes. |
| Result cannot be saved after 3 tries (async) | Report handled; ops alerted with the job id; the job is reported lost after its lease. |
| Slack down or not configured | No effect on the run. |
| Invalid CONFIG / `DATA_SOURCES_JSON` | 500 `report_misconfigured` naming every bad setting (a scheduled run alerts instead). |

## Verification

Tested end to end on self-hosted n8n 2.35.7 against in-memory reference implementations of the services above, including a Gotenberg stand-in that enforces the real API's `index.html` file-part requirement and returns a small PDF (not included in this repo). 155 checks, all passing, across fourteen configurations (defaults with Slack on; webhook token + service key + per-source headers; a required source; five sources with custom labels, title and report name; deliberately bad CONFIG; no fallback address; `MAX_RECIPIENTS=2`; no Slack; the schedule trigger built to fire every minute; and three layout configurations: the example layout, a deliberately broken one, and one with a syntax error):

- happy path: three source fetches with the period, a real `index.html` to the renderer, the real PDF archived and attached to every email, store records who got it, no alerts
- idempotency: the same period again is skipped; `force` re-runs and really re-sends (fresh keys)
- eleven kinds of bad input are 400s and touch nothing; a one-day and a 92-day period are accepted
- one source down (retried once, section marked, email says incomplete, alert), an HTML maintenance page, a 404, an empty answer ("no data returned" vs "unavailable")
- every source down: 502, claim released, alert, and the same period then runs normally
- markup in source data is escaped
- Gotenberg erroring (3 tries) and returning a non-PDF
- archive failing (3 tries, delivery continues, alert)
- subscriber list failing, empty, malformed; invalid, mixed-case and duplicate addresses
- one bounced mailbox, resume (only the failed one is emailed, nobody twice), everything bouncing, one transient mail 500 retried
- report store down at the claim; outcome record failing (3 tries, warning, alert)
- five simultaneous requests: one run; a claim left behind by a crash is taken over after it expires
- the schedule trigger: runs the previous Monday-Sunday, skips the already-sent period on its next firing without a second fetch, email or alert, and alerts when every source is down; required source; five sources; no-fallback and too-many-recipients stops; no Slack
- async mode (manual webhook only): off by default (flags ignored, status endpoint 404); a 202 before the sources are even fetched, with Location and Retry-After; the job moving queued → running → succeeded with a result equal to the synchronous reply; a failed run as a failed job with the real 502; a partial delivery as a succeeded job; bad requests refused with no job or claim; the same `Idempotency-Key` never runs twice; two simultaneous submissions of one period sending three emails in all (the second job ends 409); job store down at submit (no claim, nothing fetched); a failed "mark running"; result saves retried, then a Slack alert; lost jobs reported after the lease; oversize results trimmed or failed; the status endpoint's token, bad ids, other workflows' jobs, store down; `always` mode; a scheduled run never turning into a job; bad settings (including a lease shorter than the run limit) named in the `500`. Deliberately weakening the async logic in 14 ways makes the suite fail

Not tested: a real job store (only the reference one in the mock); a real Gotenberg, a real mail service, a real report store, n8n 1.x, queue mode, the actual weekly Monday 08:00 firing (the test fires it every minute instead; the period maths is the same), large attachments, sustained load.

## Known limits

- **Synchronous unless you turn async mode on.** By default the caller of a manual run waits for everything: sources (fetched one after another), PDF, archive and every email. Worst case is roughly twice the sum of your timeouts plus one send per recipient; keep the list modest or call the endpoint from something that can wait.
- **n8n's own node retry is not used for the multi-item steps.** in my tests it did not retry the failing source fetch, and it re-sent every item in the email step, so those two steps use an explicit second pass instead: one more try, only for transient failures. Gotenberg, the archive upload and the final record use n8n's built-in retry (3 tries).
- **Duplicate protection depends on your mail service.** The workflow sends an `idempotency_key` with every email and never re-sends to someone the store lists as delivered, but if the process dies after a send and before the outcome is recorded, only a mail service that drops repeated keys prevents a duplicate on the re-run.
- **A crash holds the period.** If a run dies mid-way, the period stays claimed until `PROCESSING_TTL_SECONDS` passes (15 minutes by default), then the next run takes over.
- **Schedule time zone.** The trigger fires in the n8n instance's time zone; the reported period is computed from UTC dates. Near midnight in a far-from-UTC zone the two can disagree by a day.
- **The schedule is set in the trigger node,** not in `CONFIG` (n8n does not allow that).
- **Plain layout.** Sources are shown as tables, cards and lists of whatever JSON they return. The Report Layout node sets page size, accent colour, font, headline figures, sorting, totals and highlights, but there are no charts or images, no repeated page headers or footers, and one attachment for everyone. How `page` and `theme` look in a real Chromium render has not been checked.
- **The whole PDF is held in memory** and attached to every email.
- **Recipients are not personalised** and `MAX_RECIPIENTS` stops a run rather than truncating the list.
- **Async mode needs a job store you run.** The mock ships a reference one, not a production service, and async was tested against that mock only. Only manual runs can be jobs; the schedule is unchanged.
- **Async work still runs inside one n8n execution.** A restart or n8n's own execution timeout ends it; the lease then reports the job lost and the caller submits again (a crashed run's period stays claimed until `PROCESSING_TTL_SECONDS`, so an immediate resubmit may end as a job whose result is `409 run_in_progress`). Emails already sent are protected by the store's delivered list and the mail service's idempotency key.
- **Async has no push notification.** The caller polls the status endpoint. A callback URL is deliberately left out (it needs the same care about which addresses it may call as any outbound URL); adding one means a new node that calls your URL, and the URL must be checked the same way.
- **Anyone holding the workflow token can read that workflow's jobs.** Job ids are unguessable but behave like bearer secrets. Many async submissions at once all run at once; use n8n queue mode or a gateway to cap that.
- n8n's own `422` for malformed JSON includes a stack trace in its body; put a gateway in front if that matters.
