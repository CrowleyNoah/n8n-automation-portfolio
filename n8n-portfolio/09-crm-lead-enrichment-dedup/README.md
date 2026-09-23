# 09 — CRM Lead Enrichment & Dedup

## Business problem
Inbound leads arrive with messy, inconsistent data — typos, nicknames,
slightly different company name formatting — and get enriched from
external sources that don't always agree with what the lead submitted.
A naive pipeline either creates duplicate CRM records for the same
person, or blindly auto-merges on a shaky match and corrupts two
different people's data into one record. This one scores match
confidence explicitly and treats "maybe the same person" as a distinct,
safer case from both "definitely the same person" and "definitely new."

## Architecture

`POST /webhook/leads/intake` → `{ email, first_name, last_name, company, title?, phone?, linkedin_url?, source }`

1. **Validate**, then **idempotency check** — a deterministic id derived
   from the email (when the caller doesn't supply one) catches a
   double-submitted form even before fuzzy matching runs.
2. **Enrich** via a self-hosted LinkedIn profile lookup (a
   Playwright+Stealth scraping bridge standing in for a paid
   third-party enrichment API). A scraper failure or "not found"
   degrades the lead to unenriched rather than blocking intake.
3. **Company mismatch check** — if enrichment's current company differs
   from what the lead submitted, it's flagged, not silently overwritten
   either direction.
4. **Search the CRM** for candidate matches, then **score similarity**
   with a hand-rolled Levenshtein-based comparison on name + company
   (weighted 60/40). An exact email match short-circuits to a certain
   match; otherwise the score lands in one of three bands.
5. **Route by confidence**: `exact_match`/`high_confidence` (≥0.85) auto-
   merge into the existing record; `possible_duplicate` (0.6–0.85)
   creates a new record but flags it, linked to the candidate it
   resembles, for a human to decide; below 0.6 is treated as genuinely
   new.
6. **Merge rule**: existing non-empty CRM fields are never overwritten by
   incoming data — only empty fields get filled in.
7. Three independent post-write checks (CRM failure, needs-review,
   company-mismatch) each fire their own Slack notification without
   gating each other or the final response.

## Edge cases handled
- **Confidence-tiered deduplication** — the three-way split
  (auto-merge / flag-for-review / treat-as-new) is the core design
  decision: auto-merging on an uncertain score is worse than creating a
  flagged duplicate, because merging wrong silently corrupts two people's
  data into one record with no easy undo.
- **Merge conflicts** — "fill empty fields, never overwrite existing
  ones" avoids a lower-quality new submission clobbering previously
  verified CRM data.
- **Conflicting enrichment vs. self-reported data** — a company mismatch
  is surfaced, not resolved automatically in either direction; it could
  be a job change or a bad submission, and guessing wrong either way is
  worse than asking.
- **Enrichment as an enhancement, not a dependency** — a down or slow
  scraper, or a legitimate "not found," both degrade to processing the
  lead unenriched rather than failing intake entirely.
- **Duplicate intake (double form submit)** — caught by a deterministic,
  email-derived idempotency key even when the caller never supplies an
  explicit lead id.
- **Distinct notification channels for distinct concerns** — a CRM write
  failure, a flagged possible-duplicate, and a company mismatch are three
  unrelated facts about the outcome; each gets checked and notified
  independently rather than collapsed into one generic "something to
  look at" alert.
- **State surviving external calls** — the same `Merge(combineByPosition)`
  pattern used throughout this portfolio, applied six times across
  enrichment, search, and CRM write calls.

## Environment variables expected
`LEAD_STORE_API_URL`, `ENRICHMENT_API_URL`, `CRM_API_URL`,
`SLACK_WEBHOOK_URL`
