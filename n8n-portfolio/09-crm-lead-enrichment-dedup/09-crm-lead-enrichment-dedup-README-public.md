# 09 — CRM Lead Enrichment & Dedup

A single n8n workflow that takes an inbound lead, optionally enriches it, checks the CRM for an existing record, and then does the safe thing: **merge** when it is confident it's the same person, **create-and-flag** when it is not sure, **create** when it is new. Exposed as one endpoint, `POST /webhook/leads/intake`.

The interesting part is the decision, not the plumbing. A naive pipeline either creates a duplicate for every double-submit, or auto-merges on a shaky match and silently fuses two different people into one record. This one scores the match explicitly, treats "maybe the same person" as its own case, never overwrites data already in the CRM, and is safe to retry: a failed write is never remembered as done, and a double-submit never creates two records.

## Design principle

Nothing is hardcoded. Every service URL, auth header, threshold, weight, nickname and TTL lives in one `CONFIG` node. The workflow talks to three small HTTP services (lead store, enrichment, CRM) and an optional Slack-compatible webhook, defined by the contract below. Swap any of them by changing a URL; no single vendor is assumed.

## Request

`POST /webhook/leads/intake`

```json
{
  "email": "ana.rivera@umbrella.io",
  "first_name": "Ana",
  "last_name": "Rivera",
  "company": "Umbrella Labs",
  "title": "Head of Growth",
  "phone": "+1 555 0100",
  "linkedin_url": "https://www.linkedin.com/in/ana-rivera",
  "source": "website",
  "lead_id": "form-8841"
}
```

| Field | Required | Meaning |
|---|---|---|
| `email` | yes | Valid address, 254 characters at most. Lower-cased and trimmed. |
| `first_name`, `last_name` | yes | Non-empty strings, 100 characters at most. |
| `company` | yes | Non-empty string, 200 characters at most. |
| `title`, `phone` | no | Strings (200 / 50 characters at most). |
| `linkedin_url` | no | Must start with `http://` or `https://`. Passed to the enrichment service. |
| `source` | no | Where the lead came from. Defaults to `unknown`. |
| `lead_id` | no | Identity of the lead for duplicate-submit protection. If omitted it is `email:<normalised email>`. If you supply one, it is the identity (1–254 characters: letters, digits and `. _ : @ + -`). |

If `CONFIG.INTAKE_AUTH_TOKEN` is set, callers must send it as the `x-intake-token` header.

## Responses

| Status | Body | When |
|---|---|---|
| 201 | `{status:"created", lead_id, crm_record_id, classification, match_score, needs_review, review_reason, possible_duplicate_of, enrichment_status, company_mismatch}` | A new CRM contact was created (`classification` is `new_lead`, `possible_duplicate` or `search_failed`). Either success reply may include `warnings:["idempotency_record_not_saved"]`. |
| 200 | `{status:"merged", lead_id, crm_record_id, classification, match_score, needs_review:false, fields_filled, conflicts, enrichment_status, company_mismatch}` | Merged into an existing contact (`exact_match` or `high_confidence`). |
| 200 | `{status:"already_processed", lead_id, original_status, crm_record_id, classification, processed_at}` | Same lead already completed inside the idempotency window. Nothing is re-done. |
| 400 | `{error:"validation_failed", details:[...]}` | Bad input. Nothing was called. |
| 422 | `{error:"lead_rejected", details:[...]}` | Only if you configure `blocked_email_domains` in the `Lead Rules` node. Nothing was called. |
| 401 | `{error:"unauthorized"}` | Missing or wrong `x-intake-token`. |
| 409 | `{error:"lead_in_progress", retry_after_seconds}` | Another request holds this lead right now. |
| 500 | `{error:"intake_misconfigured", details:[...]}` | Invalid CONFIG (the message says what to fix). |
| 502 | `{error:"crm_write_failed", status:"create_failed"\|"merge_failed", detail, retry:true}` | The CRM write failed. The claim is released, so retrying works. |
| 503 | `{error:"lead_store_unavailable", retry:true}` | The claim could not be taken. Nothing was enriched or written. |
| 503 | `{error:"crm_search_failed", retry:true}` | Only with `SEARCH_FAILURE_MODE=reject`: the duplicate check could not run, nothing was written. |

Malformed JSON never reaches the workflow: n8n itself rejects it with a `422`.

## Architecture

```
request → auth + CONFIG sanity + shape validation          (400 / 401 / 500)
        → CLAIM the lead (atomic create-or-return-existing)
            201            → you own this lead
            409 completed  → reply 200 already_processed
            409 processing → reply 409 lead_in_progress
            anything else  → reply 503 (we cannot guarantee exactly-once)
        → enrich (best effort)         success | not_found | failed | skipped
        → search the CRM for candidates
        → score the candidates         exact_match | high_confidence | possible_duplicate | new_lead | search_failed
        → merge  (exact_match, high_confidence)   → PATCH only the empty fields
          create (new_lead, possible_duplicate)   → POST, flagged if possible_duplicate
          search_failed → create flagged "dedup_unchecked"  (or reject, by CONFIG)
        → write succeeded?  yes → mark the lead completed     no → release the claim
        → reply
        → (after the reply) notifications
```

1. **Validation first.** Auth token, CONFIG sanity and request shape are checked before anything is claimed or called. Every rejection is a specific 4xx/5xx body, never a stack trace.
2. **Atomic claim.** The lead store creates the record or returns the existing one in a single call. The old pattern (read, work for several seconds, write at the very end) lets two simultaneous submits both pass the check.
3. **Enrichment is an enhancement, not a dependency.** A refused connection, a timeout, a 5xx, a "not found" and an HTML captcha page all degrade to "unenriched" and the lead is processed anyway. `enrichment_status` says which.
4. **Company mismatch is surfaced, not resolved.** If the enriched current company differs from the submitted one (compared without case, punctuation, accents or suffixes like Inc/LLC/GmbH), the lead is flagged and sales is told. Neither value overwrites the other; the enriched one is stored separately as `enriched_company`.
5. **Search.** The CRM is asked for candidates by email, last name and company, with proper query encoding (`AT&T`, `pat+tag@x.com` and `O'Neil` arrive intact).
6. **Scoring.** An exact email match against a contact's primary **or secondary** email is a certain match. Otherwise: name similarity (60%) plus company similarity (40%), where similarity is `1 - Levenshtein/longer length`, after normalising case, accents and punctuation, mapping nicknames to formal names (`Bob` → `Robert`, table in `CONFIG`) and stripping company suffixes. Weights and both thresholds are `CONFIG` values.
7. **Three bands, three behaviours.** `>= AUTO_MERGE_THRESHOLD` (0.85): merge. `>= REVIEW_THRESHOLD` (0.6): create a **new** record flagged `needs_review` and linked to the candidate, for a human to decide. Below: genuinely new. Merging wrong silently corrupts two people into one record with no easy undo; a flagged duplicate is cheap to fix.
8. **Merge never overwrites.** Only fields that are empty in the CRM are filled, and only those fields are sent. Anything that disagreed is returned in `conflicts` (`{field, kept, incoming}`). On a fuzzy match with a different email, the new address is kept as `secondary_email` instead of being dropped. If there is nothing to fill, no PATCH is made.
9. **A failed duplicate check is not "no duplicates".** If the search errors, times out or returns a malformed body, the lead is created flagged `dedup_unchecked` and ops is told (`SEARCH_FAILURE_MODE=flag`, default), or rejected with a 503 so the caller retries (`reject`). It is never silently treated as new.
10. **Failures are honest and retryable.** A write failure replies 502 and **releases** the claim. Only successful writes are remembered, so a retry is processed instead of being told "already processed" forever.
11. **Notifications run after the reply.** A slow or dead Slack never delays or changes the response.

## Edge cases handled

- **Double-click and duplicate submits.** Same `lead_id` (or same email) = same lead, whether the second request arrives after, during, or after a failure of the first.
- **Concurrency.** Six simultaneous identical submits produce one CRM contact, one enrichment call; the others get a `200 already_processed` or `409 lead_in_progress`, never a 5xx.
- **No repeated work.** Each service is called once per lead (an earlier design of this pattern ran nodes once per incoming branch).
- **Crash between "created in CRM" and "recorded".** The CRM contact carries `external_ref = lead_id`, and a resubmit finds it by email, so it merges onto itself instead of duplicating.
- **Auto-merge on a typo, not on a stranger.** `Jon Park` / `John Park` at the same company scores about 0.93 and merges; the same name at a different company lands in review.
- **Candidates with missing fields.** A CRM row with no name is scored as "unknown", not as the text `undefined undefined`.
- **A 2xx without an id is a failure.** A create that doesn't return an id is not reported as success.
- **Input hardening.** Non-string fields, over-long values, bad `linkedin_url` and `lead_id` are 400s. Unicode and accents are accepted.
- **Helper services fail safe.** Enrichment, Slack and the completion record never turn a good CRM write into a failed request.
- **State survives external calls.** Downstream nodes read earlier results by node name, so no HTTP response can overwrite request state.

## Configuration (`CONFIG` node)

| Key | Default | Meaning |
|---|---|---|
| `LEAD_STORE_API_URL` | `http://localhost:4300` | Base URL of the lead store (duplicate-submit protection). |
| `ENRICHMENT_API_URL` | `http://localhost:4300` | Base URL of the enrichment service. |
| `CRM_API_URL` | `http://localhost:4300` | Base URL of the CRM. |
| `SERVICE_HEADERS_JSON` | `{}` | Headers sent to the lead store and CRM, e.g. `{"x-api-key":"…"}`. |
| `ENRICHMENT_HEADERS_JSON` | `{}` | Headers sent to the enrichment service only. |
| `SLACK_WEBHOOK_URL` | blank | Slack-compatible incoming webhook. Blank = no notifications. |
| `INTAKE_AUTH_TOKEN` | blank | If set, callers must send it as `x-intake-token`. |
| `ENRICHMENT_ENABLED` | true | `false` skips enrichment entirely. |
| `AUTO_MERGE_THRESHOLD` | 0.85 | Score at or above which a fuzzy match is merged. |
| `AUTO_MERGE_FUZZY` | true | `false` turns fuzzy auto-merge off: only an exact email merges automatically, and every other match at or above `REVIEW_THRESHOLD` is created flagged for review. |
| `REVIEW_THRESHOLD` | 0.6 | Score at or above which a new record is created flagged for review. Must be `<= AUTO_MERGE_THRESHOLD`. |
| `NAME_WEIGHT` / `COMPANY_WEIGHT` | 0.6 / 0.4 | Relative weights in the score (normalised, so they need not sum to 1). |
| `NICKNAMES_JSON` | ~30 common English nicknames | Object of nickname → formal first name. Extend it for your market. |
| `SEARCH_FAILURE_MODE` | `flag` | If the duplicate search fails: `flag` creates the record marked `dedup_unchecked`; `reject` replies 503 and writes nothing. |
| `PROCESSING_TTL_SECONDS` | 120 | How long a claim is held while a lead is being processed. Set it above your slowest legitimate run (enrichment timeout plus CRM time). |
| `IDEMPOTENCY_TTL_SECONDS` | 86400 | How long a completed lead is remembered (the duplicate-submit window). |
| `REQUEST_TIMEOUT_MS` | 10000 | Timeout for lead-store, CRM and Slack calls. |
| `ENRICHMENT_TIMEOUT_MS` | 20000 | Timeout for the enrichment call. The caller waits at most this long for enrichment. |

## Service contract

All bodies are JSON. `SERVICE_HEADERS_JSON` is sent on every lead-store and CRM call.

**Lead store**

| Call | Body | Response |
|---|---|---|
| `POST /leads/claim` | `{lead_id, processing_ttl_seconds}` | `201` if new (the caller now owns it); `409 {existing:{status:"processing"\|"completed", record?}}` if it exists. **Must be atomic.** |
| `POST /leads/{lead_id}/complete` | `{record:{status, crm_record_id, classification, processed_at}, ttl_seconds}` | `200`. Marks it `completed` and keeps `record` for `ttl_seconds`. |
| `POST /leads/{lead_id}/release` | `{}` | `200`. Deletes the claim if it is still `processing`. |

`lead_id` is URL-encoded in the path. By default it is `email:<address>`, so the lead store holds email addresses (as your CRM does).

**Enrichment** (`ENRICHMENT_HEADERS_JSON`): `POST /enrich-profile` `{email, linkedin_url, first_name, last_name, company}` → `200 {found:true, title, current_company, company_size, industry}` or `200 {found:false}`. Anything else is treated as "failed" (a `404` as "not_found"). Any provider that can answer this fits behind a small adapter.

**CRM**

| Call | Response |
|---|---|
| `GET /contacts/search?email=&last_name=&company=` | `200 {candidates:[{id, first_name, last_name, email, secondary_email?, company, title, phone, industry, company_size, …}]}`. Return a **broad net**: contacts whose email equals (primary or secondary), **or** whose last name matches, **or** whose company matches. The workflow does the scoring. Rows without an `id` are ignored. |
| `POST /contacts` | Body: `{external_ref, first_name, last_name, email, company, title, phone, linkedin_url, source, industry, company_size, needs_review, review_reason, possible_duplicate_of, match_score, company_mismatch, enriched_company}`. `201 {id}` (or `200 {id}` if a contact with this `external_ref` already exists). **`external_ref` must be unique and idempotent.** |
| `PATCH /contacts/{id}` | Partial update with only the fields being filled (and possibly `secondary_email`). `200`. |

If your CRM has no such fields as `review_reason` or `secondary_email`, map them in your adapter (custom properties, a notes field, a tag) or drop them.

**Alerts**: `POST` `{text}` to `SLACK_WEBHOOK_URL`.

## Failure behavior

| What breaks | What happens |
|---|---|
| Lead store down or unreachable at the claim | 503 `lead_store_unavailable`; nothing enriched or written; ops alert. |
| Enrichment down, slow, 5xx, captcha page | Lead processed unenriched; `enrichment_status:"failed"`. A slow service is cut off at `ENRICHMENT_TIMEOUT_MS`. |
| Enrichment says "not found" | Processed unenriched; `enrichment_status:"not_found"`. |
| CRM search fails or returns a malformed body | `flag` (default): created flagged `dedup_unchecked` + ops alert. `reject`: 503 `crm_search_failed`, nothing written, claim released. |
| CRM create or update fails, is unreachable, or returns no id | 502 `crm_write_failed`; claim released so a retry works; ops alert. |
| Completion record fails after a good CRM write | 201/200 with `warnings:["idempotency_record_not_saved"]` + ops alert. A resubmit inside the window will find the contact by email and merge, not duplicate. |
| Claim release fails | Reply is unchanged; ops alert; the claim frees itself after `PROCESSING_TTL_SECONDS`. |
| Slack down or not configured | No effect on the reply. |
| Invalid CONFIG | 500 `intake_misconfigured` naming every bad setting. |

## Verification

Tested end to end on self-hosted n8n 2.35.7 against an in-memory reference implementation of the service contract above (not included in this repo). 137 checks, all passing, across twelve configurations (defaults with Slack on; auth token + service headers; `SEARCH_FAILURE_MODE=reject`; enrichment off; stricter thresholds; fuzzy auto-merge off; no Slack; a 1 s enrichment timeout; deliberately bad CONFIG; and three rules configurations: the example rules, deliberately broken ones, and rules with a syntax error):

- new lead with enrichment: real JSON bodies reach the CRM, one call per service, `external_ref` set, no notification
- idempotent replay: only the claim call is made; case-insensitive email
- exact match: nothing overwritten, `conflicts` reported, no pointless PATCH; gap-filling sends only the empty fields; company names compared without suffixes
- fuzzy matching: nickname (`Bob` = `Robert`), typo-level (`Jon`/`John` ≈ 0.93), `secondary_email` kept and then matched as an exact email, best of several candidates, same last name only → new, candidate rows with no name
- possible duplicate: created, flagged, linked, sales notified; thresholds changed through CONFIG change the outcome
- six simultaneous identical submits: one contact, one enrichment call, no 5xx; twenty-five different leads at once: no cross-talk
- CRM create failure, 2xx-without-id, dropped connection, update failure: 502, claim released, retry creates exactly one contact
- duplicate search returning 500, a malformed body, or a dropped connection: flagged `dedup_unchecked` (and rejected with `reject`)
- enrichment 500, dropped connection, HTML captcha page, 404, 4 s response against a 1 s timeout; enrichment disabled
- company mismatch flagged without overwriting; `Acme, Inc.` vs `ACME` is not a mismatch
- lead store down / dropped at claim; completion failing; release failing; claim expiry; idempotency-window expiry
- explicit `lead_id`; `AT&T`, `+`, apostrophes, accents
- eleven kinds of bad input are 400s and touch no service; malformed JSON is refused by n8n; 401s; Slack failing or not configured; bad CONFIG names every problem

Not tested: n8n 1.x, queue mode, a real CRM, a real enrichment provider, real Slack, a database-backed lead store, sustained load.

## Known limits

- **Matching is simple on purpose.** Levenshtein on normalised names and companies, plus a nickname table and common titles and suffixes (Dr., Jr.). It does not handle initials (`J. Park`), middle names, or names in non-Latin scripts (those normalise to nothing, so only an exact email can match them). The `Lead Rules` node has further data-only matching settings (swapped names, name weights, company aliases, placeholder companies) that this README does not document.
- **Same name at the same company auto-merges.** Two different people called `John Park` at one company score 1.0 and merge. The new email is kept as `secondary_email` and the disagreement is listed in `conflicts`, so nothing is lost, but if wrongly merging matters more to you than reviewing duplicates, set `AUTO_MERGE_FUZZY=false` and only exact emails will merge automatically.
- **The search decides what can be found.** A duplicate the CRM search doesn't return is never compared.
- **Not atomic across systems.** CRM write and completion record are two calls. A crash between them is covered by `external_ref` and the email lookup, but if your CRM can't enforce `external_ref` idempotency, add that to your adapter.
- **Synchronous.** The caller waits for enrichment (up to `ENRICHMENT_TIMEOUT_MS`) and the CRM write. Notifications are sent after the reply and are lost if n8n dies in that instant.
- **Identical resubmits are ignored for `IDEMPOTENCY_TTL_SECONDS`.** A returning lead with new information inside that window gets the original answer. After it, the lead is processed again and merges onto its own record.
- **Title preference.** For a new record the submitted title wins over the enriched one; when merging, an existing CRM title wins over both.
- n8n's own `422` for malformed JSON includes a stack trace in its body; put a gateway in front if that matters.
