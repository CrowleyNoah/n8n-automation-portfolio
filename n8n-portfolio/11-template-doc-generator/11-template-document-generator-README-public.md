# 11 — Template-Driven Document Generator

A single n8n workflow that turns a stored template plus a JSON payload into a numbered, archived PDF. It exposes two endpoints: `/documents/generate` (allocate a number, render, archive, deliver) and `/documents/preview` (render only, no side effects).

The interesting part is not the rendering, it is making document generation **safe to retry**. Document numbers are legal and financial identifiers: a retry must never mint a second number, a failure must never leave a gap that can't be explained, and a caller must never be told "success" for a document that wasn't stored.

## Design principle

Nothing is hardcoded. Every service URL, auth header, timeout, tolerance and formatting choice lives in one `CONFIG` node. What is specific to your business (extra document types, allowed templates, required fields, defaults, limits, recipient domains, per-template formats) lives in one data-only `Document Rules` node. The workflow talks to five small HTTP services (template store, document store, PDF renderer, notifier, audit log) and an optional alert webhook, defined by the contract below. Swap any of them by changing a URL.

## Requests

`POST /webhook/documents/generate`

```json
{
  "template_id": "invoice-standard",
  "template_version": "latest",
  "document_type": "invoice",
  "idempotency_key": "order-8841-invoice",
  "recipient": { "name": "Pat Lee", "email": "pat@example.com" },
  "merge_data": {
    "client": { "name": "Acme Ltd" },
    "invoice_date": "2026-01-05",
    "line_items": [ { "description": "Design", "quantity": 2, "unit_price": 100 } ],
    "tax_rate": 0.1
  }
}
```

`POST /webhook/documents/preview` takes the same body (no `idempotency_key` needed) and returns the PDF itself.

| Field | Required | Meaning |
|---|---|---|
| `template_id` | yes | Template to fetch. Letters, digits, `.` `_` `-`. |
| `template_version` | no | `latest` (default) or a version id. |
| `document_type` | yes | `invoice` or `contract`, plus any type you add in `Document Rules`. Selects the number series and, for invoices and totals types, the totals logic. |
| `merge_data` | yes | Object merged into the template. |
| `recipient` | no | `{name?, email?}`. If `email` is present and `SEND_EMAIL` is on, the document is emailed. |
| `idempotency_key` | generate only | 1–128 chars. The identity of the document: same key = same document, always. |

If `CONFIG.GENERATOR_AUTH_TOKEN` is set, callers must send it as the `x-generator-token` header (both endpoints).

## Responses

| Status | Body | When |
|---|---|---|
| 201 | `{status:"completed", document_number, archive_url, delivery:{email}, warnings}` | Generated, archived, recorded. `delivery.email` is `sent`, `failed` or `skipped`. |
| 200 | `{status:"already_generated", document_number, archive_url}` | Same `idempotency_key` already completed. Nothing is re-done. |
| 200 (preview) | the PDF, plus `X-Warning-Count` / `X-Warnings` headers | Preview succeeded. |
| 400 | `{error:"invalid_request", details:[...]}` | Bad input. |
| 401 | `{error:"unauthorized"}` | Missing or wrong `x-generator-token`. |
| 403 | `{error:"template_not_allowed"}` | Only if you set `allowed_template_ids` in `Document Rules`. |
| 404 | `{error:"template_not_found"}` | Unknown template or version. |
| 409 | `{error:"template_inactive"}` | Template exists but is retired. |
| 409 | `{error:"totals_mismatch", supplied_total, computed_total}` | Invoice `total` disagrees with the line items. |
| 409 | `{error:"generation_in_progress"}` | Another request holds this key right now. |
| 422 | `{error:"missing_required_fields", missing_fields}` / `invalid_line_items` / `invalid_tax_rate` / `invalid_total` | Data the template or invoice can't use. |
| 422 | `{error:"recipient_domain_not_allowed" \| "too_many_line_items" \| "total_exceeds_limit"}` | Only if you set the matching `Document Rules` (recipient domains, limits). |
| 500 | `{error:"template_render_failed", detail}` | The template itself is malformed. Record released for retry. |
| 500 | `{error:"generator_misconfigured", details}` | Invalid CONFIG (the message says what to fix). |
| 502 | `{error:"pdf_generation_failed" \| "archive_failed", document_number, retry}` | Failed after a number was allocated. Record released for retry. |
| 503 | `{error:"template_store_unavailable" \| "document_store_unavailable"}` | A required service is down. |

## Architecture

```
request → auth + config + shape validation
        → fetch template → required fields present?
        → invoice? validate line items, compute totals in cents, reject a wrong supplied total
generate only:
        → POST /generations  (atomic create-or-return-existing)
            201            → you own this key, here is your number
            409 completed  → replay the cached answer (200)
            409 failed     → compare-and-set failed→pending, reuse the SAME number
            409 pending    → someone else is working on it (409)
        → render template → PDF (Gotenberg) → verify it really is a PDF
preview: return the PDF.
generate: → archive PDF → mark completed → email (best effort) → audit log → 201
any failure after the number exists → mark failed_needs_retry → 5xx
```

1. **Validation first.** Auth token, CONFIG sanity and request shape are checked before anything is fetched or allocated. Every rejection is a specific 4xx/5xx body, never a stack trace.
2. **Template check.** The template store is asked for the template; 404, inactive, store-down and missing required fields each get their own status.
3. **Money is computed in integer cents.** Subtotal, tax and total are derived from the line items, not trusted from the caller. If the caller supplies a `total`, it must match within `TOTALS_TOLERANCE` or the request is rejected. This check runs **before** a number is allocated, so a rejected request never burns a number.
4. **Atomic numbering.** The document store allocates the number inside one `create-or-return-existing` call. The workflow never reads a counter and then writes one (the check-then-act race that produces duplicate numbers under concurrency).
5. **Replay semantics by record status.** `completed` returns the original answer. `failed_needs_retry` is claimed with a compare-and-set so that of several simultaneous retries exactly one proceeds, reusing the same number. `pending` means another request is mid-flight: the caller gets 409 instead of a duplicate document.
6. **Render.** A small template engine (no external library) supports `{{field}}`, `{{currency x}}`, `{{date x}}`, `{{#if}}…{{else}}…{{/if}}` and `{{#each}}` with proper nesting, `{{this}}` and `{{@number}}`. Every substituted value is HTML-escaped.
7. **PDF.** HTML is sent to Gotenberg as a multipart upload (a file part named `files` called `index.html`). The node is deliberately *not* `neverError`, so a 5xx throws and `retryOnFail` genuinely retries (3 tries). The response bytes must start with `%PDF` before they are used.
8. **Archive.** The actual PDF is uploaded (multipart) with its document number, type and key. Only after the store confirms with an `archive_url` is the record marked `completed`.
9. **Delivery is isolated.** Email and audit logging are best effort. If email fails the document is still safe: the caller gets 201 with `delivery.email:"failed"` and a warning, and ops are alerted if a Slack-compatible webhook is configured.
10. **Failures are honest.** Render, PDF and archive failures all converge on one path: the record is marked `failed_needs_retry`, the caller gets a 5xx naming the stage and the document number, and re-sending the same request retries with the same number.

## Edge cases handled

- **Duplicate requests and double-clicks.** Same key = same document, whether the second request arrives after, during, or after a failure of the first.
- **Concurrency.** Six simultaneous identical requests produce one number, one PDF, one archive, one email.
- **No gaps from bad input.** Validation, template, field and totals checks all run before allocation. Numbers are only consumed by requests that passed them.
- **Retry after failure reuses the number.** A PDF or archive failure doesn't leave a hole in the sequence.
- **A 200 from the renderer is not trusted.** The bytes are checked for the PDF signature.
- **Retries that actually retry.** `neverError` is used only where a non-2xx status is meaningful data (404/409 from the stores). The renderer, archive and status-update calls throw on 5xx so `retryOnFail` works.
- **Totals never silently pass.** A non-numeric `total`, a non-numeric quantity, or a percent-style `tax_rate` (7 instead of 0.07) is a 422, not a quiet zero.
- **No markup injection.** Merge data is escaped, so a client named `<script>…` renders as text.
- **Dates don't shift.** Date-only values are formatted in a configured time zone (default UTC), so `2026-01-05` never becomes January 4th on a server west of Greenwich.
- **Input hardening.** `template_id`, version and key are validated and URL-encoded before they reach a URL; prototype-pollution path segments are blocked in the template engine.
- **Helper services fail safe.** An unreachable notifier, audit log or alert webhook never kills the execution or changes the response status.
- **State survives external calls.** Downstream nodes read earlier results by node name, so no HTTP response can overwrite request state.

## Configuration (`CONFIG` node)

| Key | Default | Meaning |
|---|---|---|
| `TEMPLATE_STORE_API_URL` | `http://localhost:4100` | Base URL of the template service. |
| `DOC_STORE_API_URL` | `http://localhost:4100` | Base URL of the document store (numbering, records, archive). |
| `GOTENBERG_API_URL` | `http://localhost:4100` | Base URL of Gotenberg. |
| `NOTIFY_API_URL` | `http://localhost:4100` | Base URL of the email service. |
| `AUDIT_API_URL` | `http://localhost:4100` | Base URL of the audit log. |
| `SLACK_WEBHOOK_URL` | blank | Slack-compatible incoming webhook for ops alerts. Blank = no alerts. |
| `SERVICE_HEADERS_JSON` | `{}` | Headers sent to the template, document, notify and audit services, e.g. `{"x-api-key":"…"}`. |
| `GOTENBERG_HEADERS_JSON` | `{}` | Headers sent to the renderer, if it sits behind auth. |
| `GENERATOR_AUTH_TOKEN` | blank | If set, callers must send it as `x-generator-token`. |
| `SEND_EMAIL` | true | Turn recipient email on or off. |
| `TOTALS_TOLERANCE` | 0.01 | Largest accepted difference between supplied and computed invoice total. |
| `CURRENCY_SYMBOL` | `$` | Prefix used by `{{currency x}}`. Number format is fixed at `1,234.50`. |
| `DATE_LOCALE` / `DATE_TIME_ZONE` | `en-US` / `UTC` | Used by `{{date x}}`. |
| `REQUEST_TIMEOUT_MS` | 10000 | Timeout for the store, notify and audit calls. |
| `PDF_TIMEOUT_MS` | 30000 | Timeout for rendering and archive upload. |

## Business rules (`Document Rules` node)

Everything specific to *your* business sits in one data-only node, `Document Rules`, checked on every request before a template is fetched or a number is allocated:

| Setting | What it does |
|---|---|
| `extra_document_types` / `totals_document_types` | Add plain document types (letter, NDA) or types with line items and the full invoice maths (quote, credit note). |
| `allowed_template_ids` | Only these templates can be used (`403 template_not_allowed`). |
| `required_fields` | Fields every request must carry, for all types or one type, on top of what the template requires. |
| `defaults` | Values used when the request leaves a field out (company name, payment terms). The request always wins. |
| `limits` | `max_line_items`, `max_total`, `max_tax_rate`. |
| `recipient_domains_only` / `recipient_domains_blocked` | Which email domains a document may be addressed to (`422 recipient_domain_not_allowed`). |
| `no_email_types` | Types that are archived but never emailed. |
| `template_formats` | Currency symbol, date locale and time zone for one template. |

Rules can only add types and data or make the generator stricter. Number allocation, idempotency, the money maths, escaping and the PDF checks cannot be changed from there. A mistake gives `500 generator_misconfigured` with lines starting `Rules:` before anything is fetched or allocated.

## Service contract

All bodies are JSON unless noted. When `SERVICE_HEADERS_JSON` is set it is sent on every call to the first four services.

**Template store**

| Call | Response |
|---|---|
| `GET /templates/{id}?version=` | `200 {active: bool, required_fields: string[], html: string}`; `404` if unknown. A retired template is `200` with `active:false`. |

**Document store**

| Call | Body | Response |
|---|---|---|
| `POST /generations` | `{idempotency_key, document_type}` | `201 {document_number}` if new; `409 {existing:{document_number, status, archive_url}}` if the key exists. **Must be atomic.** |
| `POST /generations/{key}` | `{status, archive_url?, expect_status?}` | `200`. If `expect_status` is given and the record's status differs: `409`, nothing changed (compare-and-set). |
| `POST /archive` | multipart: file part `pdf`, fields `document_number`, `document_type`, `idempotency_key` | `201 {archive_url}`. Must be idempotent per `document_number` (it can be retried). |

Record statuses: `pending` → `completed` or `failed_needs_retry`. A request that fails after allocation sets `failed_needs_retry`; a retry claims it back to `pending` with `expect_status:"failed_needs_retry"`.

**Renderer**: Gotenberg's `POST /forms/chromium/convert/html`, multipart, one file part named `files` with filename `index.html`; returns `application/pdf`.

**Notifier**: `POST /send-document-email` `{to, recipient_name, document_number, document_type, template_id, archive_url}` → any 2xx.

**Audit**: `POST /log` `{event, document_number, document_type, idempotency_key, template_id, archive_url, warnings}` → any 2xx.

**Alerts**: `POST` `{text}` to `SLACK_WEBHOOK_URL`.

Rules the document store must implement:

- **`POST /generations` is atomic.** The number is allocated in the same step that creates the record. Use a database sequence (or `INSERT … ON CONFLICT (idempotency_key) DO NOTHING RETURNING …` followed by a read) rather than read-then-write.
- **Status updates honour `expect_status` atomically** (one conditional `UPDATE … WHERE status = …`, check the affected rows).
- **Stale `pending` records should expire.** If a worker dies after allocating a number, the record stays `pending`, and callers see `409 generation_in_progress`. The store should move `pending` records older than your longest legitimate run (a few minutes) to `failed_needs_retry`. The workflow does not do this itself.
- Numbers are per `document_type` series (for example `INV-0001`, `CON-0001`); the format is entirely the store's choice.

## Failure behavior

| What breaks | What happens |
|---|---|
| Template store down | 503 `template_store_unavailable`. Nothing allocated. |
| Document store down at allocation | 503 `document_store_unavailable`. Nothing rendered. |
| Renderer error or non-PDF response | 3 attempts, then record → `failed_needs_retry`, 502 `pdf_generation_failed`. |
| Archive failure | 3 attempts, then record → `failed_needs_retry`, 502 `archive_failed`, Slack alert if configured. Never reported as success. |
| Status update to `completed` fails | Document is archived and delivered. Response is 201 with a `record_status_update_failed` warning; the record stays `pending` until the store expires it. |
| Email failure | 201 with `delivery.email:"failed"`, warning, and an alert. |
| Audit log failure | 201 with an `audit_log_failed` warning. |
| Template syntax error | 500 `template_render_failed`, record released. |
| Invalid CONFIG | 500 `generator_misconfigured` naming the problem. |

## Verification

Tested end to end on self-hosted n8n 2.35.7 against an in-memory reference implementation of the service contract above (not included in this repo). 120 checks, all passing, in six configurations (defaults; auth token + service headers + Slack alerts; and four with the `Document Rules` node filled in, deliberately broken, over a limit, or with a syntax error):

- invoice and contract happy paths, number series, archived PDF bytes, email and audit payloads
- integer-cent totals; wrong, non-numeric and within-tolerance totals; garbage quantity; percent-style tax rate
- idempotent replay returns the original answer and re-does nothing
- six simultaneous identical requests: one number, one PDF, one archive, one email
- validation and auth: missing fields, bad types, path-like ids, malformed JSON, bad email, 401s on both endpoints
- template 404 / inactive / unknown version / store down / missing and empty required fields
- HTML escaping of markup in merge data
- preview: returns a PDF, allocates nothing, archives nothing, sends nothing, reports template warnings in headers
- renderer failure: three attempts, 502, record released, retry reuses the same number
- five simultaneous retries of a failed record: exactly one succeeds
- archive failure: 502, no email, Slack alert when configured
- email failure isolated, no-recipient skip, document store down, stuck `pending` record, malformed template, nested `each`/`if`/`else`
- CONFIG validation (bad URL → 500 naming the key), `SEND_EMAIL=false`, locale and currency settings
- **Document Rules (45):** an extra document type gets its own series, fills defaults and is archived without email; a quote gets the invoice maths, a wrong total is refused and it is formatted in euros and German dates while invoices keep CONFIG formatting; required fields per type and for all types; line-item, total and tax-rate limits (exactly at the limit is accepted) with no numbers burned; disallowed templates refused before the template store is called; recipient domains outside the allow list or on the block list refused; broken rules give a clear 500 naming every mistake and fetch nothing; a tax ceiling above 1 is refused; a syntax error in the node processes nothing.

Not tested: n8n 1.x, a real Gotenberg instance (the reference renderer implements the same multipart contract), a database-backed document store, sustained load.

## Known limits

- Synchronous: the caller waits for render, archive and delivery. Worst case on a failing renderer is about three attempts of `PDF_TIMEOUT_MS`.
- Retry count and wait (3 tries, 1 s) are node settings, not CONFIG values; n8n does not allow expressions there.
- The idempotency key identifies the document, not its contents. Reusing a key with different data returns the original document.
- `{{currency x}}` formats numbers as `1,234.50` regardless of locale; only the symbol is configurable.
- The template engine is deliberately small (variables, `if`/`else`, `each`, two helpers). It is not Handlebars.
