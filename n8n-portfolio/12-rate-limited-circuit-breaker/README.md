# 12 — Rate-Limited API Integration Gateway with Circuit Breaker

## Business problem
Any workflow that calls a rate-limited third-party API eventually has to
answer the same set of questions, and answering them differently in every
workflow that touches that API is how an integration quietly becomes
fragile: How do we avoid tripping the provider's own rate limit instead
of just reacting to 429s? When the provider is genuinely down, how long
do we keep hammering it before backing off, and how do we test recovery
without a stampede of requests the moment it comes back? And when it's
down, does the caller get nothing, or the last good answer we have? This
workflow is a single front door — other workflows call it instead of the
provider directly — that answers all three consistently, once.

## Architecture

`POST /webhook/gateway/fetch` → `{ integration, resource_path, params?, cache_key, max_staleness_seconds? }`

1. **Validate** the request, then **check the circuit breaker state** for
   this `integration` (`closed` / open-but-cooling-down / open-and-due-
   for-a-probe).
2. **Open, cooldown not elapsed** → skip straight to the degrade path,
   don't even ask for a rate-limit token.
3. **Open, due for a probe** → atomically **claim a probe slot** (10s
   TTL compare-and-set). Exactly one concurrent request wins and makes
   the real call; everyone else falls through to degrade. This is what
   makes "half-open" actually mean one test call instead of a stampede
   the instant the cooldown lapses.
4. **Closed** → atomically **consume a rate-limit token** from a
   provider-sized token bucket, checked and decremented in one call, not
   a separate check-then-consume pair (same race class fixed the same
   way in workflow 11's document numbering). No tokens left → degrade,
   with a `retry_after_seconds` computed from the bucket's own refill
   rate.
5. **Call the provider API.** `onError: 'continueErrorOutput'` on this
   node is deliberate and distinct from `neverError`: `neverError` only
   stops n8n from throwing on a non-2xx *HTTP* status (which still lands
   on the normal output as a readable `statusCode`). It does nothing for
   a connection-level failure — DNS, refused, timeout — that never
   produced an HTTP response at all. Those are routed to a separate
   error output and classified the same as a `5xx` below, rather than
   halting the execution.
6. **Classify the result**: `success` / `rate_limited` (429, kept
   distinct — it means "slow down," not "broken") / `transient` (5xx or
   connection error) / `permanent` (any other 4xx — *our* request is
   wrong, the provider is fine).
7. **Success** → record it against the breaker (a successful probe fully
   closes the breaker), refresh the cache for future degrade use, respond
   fresh.
8. **Rate-limited** → parse `Retry-After` (seconds or HTTP-date form,
   capped at 120s since this is a synchronous webhook), wait, retry up to
   3 attempts — unless this call *was* the probe, which never retries in
   place.
9. **Transient** → record a breaker failure (a failed probe re-opens the
   breaker immediately with a longer cooldown), then exponential backoff
   with jitter (500ms/1s/2s — small on purpose, the caller is waiting)
   and retry up to 3 attempts.
10. **Permanent** → mirrored straight back to the caller with the
    provider's own status code. No retry, no breaker impact — retrying a
    malformed request three times is just slower wrongness.
11. **Degrade path** (every exhausted/skipped branch converges here):
    fetch the cached last-known-good response, check it against the
    caller's `max_staleness_seconds` (any age is fine if the caller
    didn't specify one), and either return it marked `stale: true` with
    its age, or respond `503` with a reason, a `Retry-After` header, and
    whether a cache even exists for this key at all.

## Edge cases handled
- **Proactive rate limiting, not just reactive backoff** — a token
  bucket sized to the provider's documented quota, so the integration
  mostly avoids hitting 429 in the first place rather than only handling
  it after the fact.
- **Connection-level failures vs. HTTP error statuses** — `neverError`
  and `onError: continueErrorOutput` cover two genuinely different
  failure classes; missing either one means either a thrown exception
  killing the execution or a silent gap in classification.
- **429 kept distinct from 5xx** — different `Retry-After`-aware backoff,
  and doesn't count toward the breaker's failure threshold the way a
  real server error does; conflating them would trip the breaker on a
  provider that's actually healthy and just asking for patience.
- **Probe stampede prevention** — the atomic claim-a-probe-slot step is
  what makes half-open mean "one test call," not "every queued request
  becomes a probe the instant the cooldown lapses."
- **A permanent (client) error never retries and never trips the
  breaker** — distinguishing "we're calling it wrong" from "it's down"
  is the single most important classification in this workflow; getting
  it wrong either wastes retries on a bug or masks a real outage behind
  auth-error noise.
- **Idempotent, race-safe token/probe/number-style operations** — rate-
  limit consumption and probe claiming are both single atomic calls, the
  same design principle applied in workflow 11's document numbering,
  reused here for a different resource.
- **Staleness tolerance is caller-controlled** — `max_staleness_seconds`
  lets one caller accept an hour-old cached value while another refuses
  anything older than a minute, from the same gateway.
- **Honest, actionable failure responses** — even the `503` carries a
  `Retry-After` header, a specific reason, and whether any cache exists
  at all for that key, rather than a bare "something went wrong."
- **State surviving external calls** — the same
  `Merge(combineByPosition)` pattern used throughout this portfolio,
  applied at every retry-loop iteration and every breaker/rate-limit
  check.
- **Bounded retry loops** — both the rate-limit and transient retry
  branches cap at 3 attempts and explicitly exclude probe calls from
  retrying in place, so neither loop can run indefinitely.

## Environment variables expected
`BREAKER_STATE_API_URL`, `RATE_LIMIT_API_URL`, `CACHE_API_URL`, plus one
`<INTEGRATION>_API_URL` per integration this gateway fronts (looked up
dynamically as `{{ integration.toUpperCase() }}_API_URL`, e.g.
`STRIPE_API_URL`, `GITHUB_API_URL` — adding a new integration is a config
entry, not a workflow change).
