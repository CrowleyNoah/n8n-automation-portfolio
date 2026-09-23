# 11 — Template-Driven Document Generator

## Business problem
Contracts and invoices generated from templates look simple until the
edge cases show up: a resubmitted request must never burn a second
invoice number, a caller-supplied total that doesn't match the line
items must never be silently trusted, a template with a typo'd id or an
archived version must fail loudly instead of rendering garbage, and a
document that renders fine but fails to archive must never be reported
to the caller as a success. This workflow treats a generated contract or
invoice as a financial/legal artifact, not just a PDF.

## Architecture

Two triggers share the same pipeline: `POST /webhook/documents/generate`
(the real thing — numbered, archived, emailed, audited) and
`POST /webhook/documents/preview` (renders and returns a PDF for a
"preview before you send" UI, and touches nothing durable).

1. **Validate the request shape** — `template_id`, a valid
   `document_type`, an object `merge_data`, and (generate mode only) an
   `idempotency_key`.
2. **Fetch the template** and confirm it's both found *and* active — a
   404 (no such template) and a 200-with-`active:false` (a real template
   whose version was retired) are different business states, but both
   block generation.
3. **Check required fields** against the template's own declared list,
   dot-path aware (`client.name`), and fail with the specific missing
   fields rather than rendering blanks into a legal document.
4. **(Generate mode) Atomically create the generation record** — one
   call assigns the next sequential document number for the series *and*
   creates the pending record keyed by `idempotency_key`, as a single
   compare-and-set. A losing concurrent request gets the winner's number
   back instead of minting its own, which is what actually prevents a
   duplicate invoice number under a race, not just under sequential
   retries.
   - Already `completed` → respond immediately with the cached result,
     no re-render.
   - `pending` / `failed_needs_retry` / `rejected_totals_mismatch` →
     proceed, reusing the same number.
5. **Render** via a small, hand-rolled templating engine (`{{field}}`,
   `{{#if field}}`, `{{#each list}}`, plus `{{currency amount}}` /
   `{{date value}}` helpers) — deliberately minimal rather than pulling
   in a full templating library for three constructs.
6. **Invoice totals are always computed server-side** from
   `quantity * unit_price`, never trusted verbatim from the caller; a
   caller-supplied total that disagrees by more than a cent is a
   discrepancy, not an auto-correction.
7. **Render HTML → PDF** (Gotenberg), with its own longer timeout and
   retry budget, since this step is genuinely slower than the metadata
   calls around it.
8. **(Preview mode)** stops here and returns the PDF directly — no
   archive, no email, no audit entry.
9. **(Generate mode)** archives the PDF; the caller isn't told
   "success" until the archive succeeds — a rendered-but-unarchived
   document is one restart away from being unrecoverable. Email delivery
   and the audit log write happen in parallel, fire-and-forget, once the
   record is marked complete.

## Edge cases handled
- **Idempotent numbering under concurrency** — the atomic
  create-or-return-existing call is the core guarantee; two identical
  requests racing each other can't both mint a number, and a losing
  request transparently gets the winner's result.
- **Never re-numbering a retry** — `failed_needs_retry` and
  `rejected_totals_mismatch` records keep their originally assigned
  number, so fixing and resubmitting reuses it instead of skipping ahead.
- **Financial totals never silently corrected** — a mismatch between
  supplied and computed totals halts generation with both numbers in the
  409 response, rather than picking one and moving on.
- **Archived/retired template versions** — distinct from "template
  doesn't exist," and both are blocking, not warnings.
- **Missing required fields** — checked against the template's own
  declared list (dot-path aware) before any rendering is attempted.
- **Unresolved merge fields inside the template itself** — the rendering
  engine logs a `warnings` entry for any `{{field}}` that resolves to
  `undefined`/`null` rather than either crashing or silently printing
  "undefined" into the document; malformed `{{#each}}` targets (not an
  array) get the same treatment.
- **"Success" gated on archiving, not just rendering** — a PDF that
  renders but fails to archive pages ops and responds `502`, never `201`.
- **Delivery failure isolated from generation failure** — a bounced or
  missing recipient email doesn't undo a successful, archived generation;
  it gets its own distinct ops alert instead.
- **State surviving external calls** — the same
  `Merge(combineByPosition)` pattern used throughout this portfolio,
  applied to every HTTP call in the numbering, rendering, archiving, and
  delivery steps.
- **Preview/generate asymmetry is deliberate** — preview shares every
  validation and rendering step with generate, but is structurally unable
  to touch numbering, the archive, email, or the audit log; that's
  enforced by which nodes preview's branch physically connects to, not by
  a runtime `if (mode !== 'preview')` guard sprinkled through shared code.

## Environment variables expected
`TEMPLATE_STORE_API_URL`, `DOC_STORE_API_URL`, `GOTENBERG_API_URL`,
`NOTIFY_API_URL`, `AUDIT_API_URL`, `SLACK_WEBHOOK_URL`
