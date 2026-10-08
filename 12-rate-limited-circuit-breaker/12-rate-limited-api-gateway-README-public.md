# 12 — Rate-Limited API Gateway with Circuit Breaker

A single n8n workflow that sits in front of any rate-limited third-party API. Other workflows call the gateway instead of calling the provider directly, and the gateway answers three questions once, consistently:

1. **How do we avoid tripping the provider's rate limit,** instead of only reacting to 429s?
2. **When the provider is genuinely down, how long do we keep hammering it,** and how do we test recovery without a stampede of requests the moment it comes back?
3. **When it's down, does the caller get nothing, or the last good answer we have?**

Answering these differently in every workflow that touches an API is how an integration quietly becomes fragile.

**Version 1.1.0** (2026-10-07). Changelog: 1.1.0, probe results now carry a claim token, so a late result from a probe whose claim was replaced is ignored. 1.0.0, first release.

## Probe claim token (version 1.1.0), in plain English

Picture a dealership with one test car. When it is time to test, one person gets the keys on a numbered tag, and the tag is only good for a set time. If that person never comes back, the dealership cancels the tag and hands the car to the next person with a new tag number. If the first person shows up late and says "the car drove fine", the desk checks the tag, sees it is not the current one, and writes nothing down. Only the holder of the current tag can report.

In technical terms: the probe claim is a lease, the tag is a `claim_token` (a fencing token), and the state service accepts a probe result only if it carries the token of the current lease. This closes a gap in 1.0.0, where a slow (not dead) probe owner could report after a replacement had taken over and still change the breaker, even re-opening one the replacement had just closed. Credit to Przemek Zielinski for spotting it.

Upgrade order matters: a new workflow works with an older state service (with a logged warning), but an old workflow with a new state service never closes the breaker, because it sends no token. Update the workflow first, or both together.

## Design principle

Nothing is hardcoded. Every URL, auth header, limit, timeout and threshold lives in one `CONFIG` node. Adding another provider is one entry in a JSON setting. Swapping the state store means changing three URLs. What is specific to one API (allowed paths and parameters, fields to hide, staleness cap, no caching, fewer retries, earlier breaker) lives in one data-only `Gateway Rules` node. No engine edits for any of it.

## Request

`POST /webhook/gateway/fetch`

```json
{
  "integration": "stripe",
  "resource_path": "customers/cus_123",
  "params": { "expand": "subscriptions" },
  "cache_key": "customer-cus_123",
  "max_staleness_seconds": 3600
}
```

| Field | Required | Meaning |
|---|---|---|
| `integration` | yes | Key into `CONFIG.PROVIDERS_JSON`. Letters, digits, `-`, `_`. |
| `resource_path` | yes | Appended to the provider's `base_url`. Plain path only: no `..`, `://`, `?`, `#`, spaces. |
| `cache_key` | yes | Name for the cached response. Namespaced per integration so two providers can never collide. |
| `params` | no | Object, sent as the query string. |
| `max_staleness_seconds` | no | Oldest cached answer the caller will accept during an outage. Omit to accept any age. |

If `CONFIG.GATEWAY_AUTH_TOKEN` is set, callers must send it as the `x-gateway-token` header.

## Architecture

```
request → validate → read breaker state
   closed ─────────────→ atomically take a rate-limit token → call provider
   open, cooling down ─→ degrade (provider and limiter never touched)
   open, cooldown over → atomically claim the single probe slot
                           won  → call provider (this request IS the probe)
                           lost → degrade
call provider → classify:
   2xx            success    → record success, refresh cache, respond live
   429            rate limit → wait Retry-After, retry (never counts against the breaker)
   5xx / timeout  transient  → record failure, back off with jitter, retry
   other 4xx      permanent  → return the provider's status as-is, no retry
exhausted or skipped → degrade: last-known-good (if fresh enough) or 503
```

1. **Validate** the request and the config, then **check breaker state** for this integration: `closed`, open-but-cooling-down, or open-and-due-for-a-probe.
2. **Open, cooldown not elapsed:** straight to the degrade path. It doesn't even ask for a rate-limit token.
3. **Open, due for a probe:** atomically **claim a probe slot** (TTL compare-and-set). Exactly one concurrent request wins and makes the real call. Everyone else degrades. This is what makes "half-open" mean one test call instead of a retry stampede the instant the cooldown lapses. The winner also receives a unique `claim_token`, which it must send back with its result.
4. **Closed:** atomically **consume a rate-limit token** from a provider-sized token bucket, checked and decremented in one call rather than a separate check-then-consume pair (a time-of-check/time-of-use race, the same class of fix as in workflow 11's document numbering). No tokens left: degrade, with a `retry_after_seconds` computed from the bucket's refill rate.
5. **Call the provider.** The HTTP node uses both `neverError` and `onError: continueErrorOutput`, and they are not interchangeable. `neverError` keeps non-2xx HTTP statuses on the normal output as a readable `statusCode`. It does nothing for a connection-level failure (DNS, refused, timeout) that never produced a response. Those go to the node's separate error output and are classified like a 5xx instead of halting the execution.
6. **Classify:** `success` / `rate_limited` (429, kept distinct: it means "slow down," not "broken") / `transient` (5xx or connection error) / `permanent` (any other 4xx: *our* request is wrong, the provider is fine).
7. **Success:** record it against the breaker (a successful probe, sending its claim token, fully closes it), refresh the cache, respond fresh. If the state service answers `accepted:false` (the claim was replaced), the breaker is left alone, the node output is marked `breaker_update: "ignored_stale_claim"`, and the provider answer is still returned.
8. **Rate-limited:** parse `Retry-After` (seconds or HTTP-date form, capped by config since this is a synchronous webhook), wait, retry up to `MAX_ATTEMPTS`, unless this call was the probe, which never retries in place.
9. **Transient:** record a breaker failure (a failed probe re-opens the breaker immediately with a doubled cooldown), then exponential backoff with jitter and retry.
10. **Permanent:** mirrored back to the caller with the provider's own status. No retry, no breaker impact. Retrying a malformed request three times is just slower wrongness.
11. **Degrade path** (every exhausted or skipped branch converges here): fetch the cached last-known-good response and check it against the caller's `max_staleness_seconds`. Return it marked `stale: true` with its age, or respond `503` with a reason, a `Retry-After` header, and whether any cache exists for that key at all.

## Responses

| Status | Body | When |
|---|---|---|
| 200 | `{data, stale:false, served_from:"live"}` | Provider answered. |
| 200 | `{data, stale:true, served_from:"cache", age_seconds, degrade_reason}` | Provider unavailable; usable cached answer exists. |
| 503 | `{error:"upstream_unavailable", reason, retry_after_seconds, cache_available}` + `Retry-After` | Provider unavailable and no usable cache. `cache_available:false` means this key has never succeeded. |
| provider's 4xx | `{error:"upstream_client_error", upstream_status, upstream_body}` | The provider rejected the request (any 4xx except 429). |
| 400 / 401 / 500 | `{error, details}` | Invalid request or a path/parameter your rules refuse / bad gateway token / invalid CONFIG or rules (the message says what to fix). |

`degrade_reason`: `breaker_open`, `probe_in_progress`, `probe_failed`, `rate_limit_exhausted`, `rate_limited_retries_exhausted`, `transient_retries_exhausted`.

## Edge cases handled

- **Proactive rate limiting, not just reactive backoff.** A token bucket sized to the provider's documented quota means the integration mostly avoids 429s instead of only handling them.
- **Connection-level failures vs HTTP error statuses.** `neverError` and `continueErrorOutput` cover two different failure classes. Missing either means a thrown exception kills the execution or classification has a silent gap.
- **429 is distinct from 5xx.** Different `Retry-After`-aware backoff, and it doesn't count toward the breaker's failure threshold. Conflating them trips the breaker on a provider that is healthy and just asking for patience.
- **Probe stampede prevention.** The atomic claim is what makes half-open mean "one test call".
- **A permanent client error never retries and never trips the breaker.** Telling "we're calling it wrong" apart from "it's down" is the most important classification here. Getting it wrong either wastes retries on a bug or hides a real outage behind auth-error noise.
- **Late probe results cannot change the breaker.** A probe that stalls past its claim TTL and is replaced may still finish. Its result carries an old token, so the state service rejects it (`accepted:false`, `stale_claim`) without changing state, releasing the claim or counting a failure. A late failure can no longer re-open a breaker the replacement already closed.
- **Failed probe escalates.** One bad probe is enough evidence: the breaker re-opens at once with a doubled cooldown, capped.
- **Staleness tolerance is caller-controlled.** One caller can accept an hour-old value while another refuses anything older than a minute, from the same gateway.
- **Honest failure responses.** Even the 503 carries `Retry-After`, a specific reason, and whether a cache exists at all.
- **Bounded retry loops.** Both retry branches cap at `MAX_ATTEMPTS` and exclude probe calls from retrying in place.
- **State survives external calls.** `Merge (combineByPosition)` re-attaches the request state after every HTTP call, including inside the retry loops.
- **Input hardening.** Provider and path are validated: unknown integrations, path traversal and absolute URLs are rejected, so a caller can't point the gateway anywhere but the configured providers.
- **Helper services failing safe.** An unreachable breaker, rate-limit or cache service never kills the execution (see below).

## Per-API rules (`Gateway Rules` node)

Everything specific to *one API* sits in one data-only node, `Gateway Rules`, checked on every request before any service or provider is called. `defaults` applies to every integration; `integrations` holds one entry per name in `PROVIDERS_JSON`, and an entry's own values win key by key.

| Setting | What it does |
|---|---|
| `allowed_path_prefixes` | `resource_path` must start with one of these, otherwise 400. |
| `allowed_params` / `blocked_params` | Which query parameters may be sent. |
| `max_staleness_seconds` | Oldest cached answer ever served for this API (the lower of this and the caller's value). |
| `remove_fields` | Dotted paths removed from every answer and from what is cached (and from older cache entries when read). |
| `use_cache` | `false` = nothing stored, no cached copy served for this API. |
| `max_attempts` / `failure_threshold` | Can only be lowered below `CONFIG`. |
| `cooldown_seconds` | Can only be raised above `CONFIG`. |

Rules can only make the gateway stricter. The token bucket, the single-probe claim, the retry and `Retry-After` handling and the degrade path cannot be changed from there. A mistake gives `500 gateway_misconfigured` with lines starting `Rules:` before anything is called.

## Configuration (`CONFIG` node)

| Key | Default | Meaning |
|---|---|---|
| `GATEWAY_AUTH_TOKEN` | blank | If set, callers send it as `x-gateway-token`. |
| `BREAKER_API_URL`, `RATE_LIMIT_API_URL`, `CACHE_API_URL` | `http://localhost:4000` | Base URLs of the three state services (may be one server). |
| `STATE_API_HEADERS_JSON` | `{}` | Headers sent to the state services, e.g. an API key. |
| `PROVIDERS_JSON` | demo entry | One entry per API: `base_url`, `headers`, `rate_limit: {capacity, refill_per_second}`. |
| `PROVIDER_TIMEOUT_MS` | 10000 | Per-call provider timeout. |
| `FAILURE_THRESHOLD` | 5 | Failed provider calls (while closed) before opening. Each retry attempt counts. |
| `COOLDOWN_SECONDS` / `MAX_COOLDOWN_SECONDS` | 30 / 600 | Initial cooldown; doubles per failed probe up to the cap. |
| `PROBE_CLAIM_TTL_SECONDS` | 15 | Probe claim lifetime. Keep above the provider timeout. |
| `MAX_ATTEMPTS` | 3 | Total tries per request (429 and transient). |
| `BACKOFF_BASE_MS` / `BACKOFF_JITTER_MS` | 500 / 250 | Exponential backoff base and random jitter. |
| `RETRY_AFTER_DEFAULT_SECONDS` / `RETRY_AFTER_CAP_SECONDS` | 5 / 30 | Wait when a 429 has no usable header; longest wait honored. Keep the cap under about 60s, since n8n offloads longer waits and a webhook caller may time out. |
| `DEGRADE_RETRY_AFTER_SECONDS` / `PROBE_IN_PROGRESS_RETRY_AFTER_SECONDS` | 30 / 10 | Retry hints reported when degrading. |
| `RATE_LIMIT_FAIL_OPEN` | true | If the rate-limit service is unreachable: true lets the call through, false degrades. |

`PROVIDERS_JSON` example:

```json
{
  "stripe": {
    "base_url": "https://api.stripe.com/v1",
    "headers": { "Authorization": "Bearer YOUR_KEY" },
    "rate_limit": { "capacity": 25, "refill_per_second": 10 }
  }
}
```

## State service contract

The breaker, rate limiter and cache are three small HTTP services. This workflow only needs the contract below, so back it with whatever you already run. Bodies are JSON.

| Call | Body | Response |
|---|---|---|
| `GET /breaker/{integration}` | | `{status:"closed"\|"open", next_probe_at: epoch ms \| null, trip_count}`; unknown = closed |
| `POST /breaker/{integration}/claim-probe` | `{ttl_seconds}` | `{claimed: true, claim_token: string}` or `{claimed: false}` (`claim_token` is new in 1.1.0) |
| `POST /breaker/{integration}/record-success` | `{is_probe, claim_token}` (token required when `is_probe` is true, omitted otherwise) | HTTP 200 `{accepted: true, ...}` or, for a stale probe result, HTTP 200 `{accepted: false, reason: "stale_claim"}` |
| `POST /breaker/{integration}/record-failure` | `{is_probe, claim_token (probe only), error_message, failure_threshold, cooldown_seconds, max_cooldown_seconds}` | same as record-success |
| `POST /rate-limit/{integration}/consume` | `{tokens, capacity, refill_per_second}` | `{allowed: bool, retry_after_seconds}` |
| `GET /cache/{key}` | | `200 {body: string, cached_at: epoch ms}` or `404` |
| `POST /cache/{key}` | `{body: string, cached_at: epoch ms}` | 2xx |

Rules the service must implement:

- **`claim-probe` is atomic.** `claimed:true` only if the breaker is open, `next_probe_at` has passed, and no unexpired claim exists, with the claim recorded in the same step. If two callers can both win, half-open becomes a stampede.
- **`claim-probe` issues a fresh unique `claim_token` per successful claim** and stores it with the expiry. A new claim replaces the stored token.
- **Probe results are token-checked.** A probe `record-success` or `record-failure` is accepted only if its `claim_token` equals the stored one. Missing, empty, wrong or replaced tokens get HTTP 200 `{accepted:false, reason:"stale_claim"}` and must not change state, release the claim or count as a failure. The token is cleared when the breaker trips or closes, so it works once. (The mock server does not reject a token merely for being expired, only for being replaced; a stricter service may.)
- **`consume` is one atomic step:** refill, check, decrement.
- An accepted `record-success` with `is_probe:true` fully closes the breaker and resets counters; while closed, any success resets the failure streak; while open, ignore it.
- `record-failure` while closed increments failures and opens at `failure_threshold`. An accepted one with `is_probe:true` re-opens at once with the cooldown doubled per trip, capped at `max_cooldown_seconds`. While open, ignore it.
- Calls without `is_probe:true` behave as before and need no token.

Redis fits all three: `SET key <uuid> NX PX ttl` for the probe claim (the stored value can be the token; the result handler compares and acts in one Lua script, which is a sketch, not tested here), a Lua script for the token bucket, and `SET`/`GET` for the cache. A SQL store works if `claim-probe` is a single conditional `UPDATE ... WHERE` and you check the affected-row count. Thresholds and cooldowns arrive with each request from CONFIG, so the service stays simple and all tuning happens in n8n.

## Failure behavior

| What breaks | What the gateway does |
|---|---|
| Provider 5xx or timeout | Retries up to `MAX_ATTEMPTS` with backoff, records failures, serves cache or 503. |
| Provider 429 | Honors `Retry-After`, retries, never counts toward the breaker. |
| Provider other 4xx | Returns that status. No retry. |
| Breaker service unreachable | Treated as closed. Calls still go through, with breaker protection off until it's back. |
| Rate-limit service unreachable | Per `RATE_LIMIT_FAIL_OPEN`. |
| Cache service unreachable | Treated as a cache miss. |
| Probe-claim service unreachable | Request degrades. It never risks an unprotected probe. |
| Late result from a replaced probe | Rejected by the state service. Breaker unchanged, not a failure, provider answer still returned, output marked `ignored_stale_claim`. |
| State service older than 1.1.0 (no `claim_token` in the claim response) | Falls back to the old behaviour with a logged warning. Late results are not protected until the service is upgraded. |
| Invalid CONFIG | 500 `gateway_misconfigured` naming the problem. |

## Verification

Tested end to end on self-hosted n8n 2.35.7 against a local in-memory state service and a controllable fake provider (neither is included in this repo). Version 1.0.0: 81 checks in 8 configurations, all passing; each protected rule was also removed in turn (15 mutations) and a test failed every time. Version 1.1.0 (2026-10-07): 131 checks in 10 configurations (95 through the workflow in n8n, 36 against the state service alone), 131 of 131 passing on n8n 2.35.7.

- probe claim token, state service alone (36 checks): normal path unchanged; a killed probe owner is replaced after the TTL; a late success from a replaced claim is rejected and changes nothing; a late failure is rejected and cannot re-trip the breaker, including after the replacement already closed it; wrong, missing, empty or non-string tokens are rejected; a token works once
- probe claim token, through the workflow (14 checks): the same late success and late failure scenarios with a real slow provider, the caller still gets the provider's answer, and the breaker follows the replacement only
- the same new tests fail against the 1.0.0 code (29 of 36 against the old state service, 4 of 14 against the old workflow), so they do detect the gap
- a state service older than 1.1.0 (no `claim_token`): the 40 default checks still pass, with a warning on the node output

- request validation and auth (missing fields, unknown integration, path traversal, absolute URL, bad params, missing/wrong token)
- permanent 4xx passthrough: one provider call, breaker untouched
- transient 5xx: three attempts, breaker trips at threshold, stale cache served
- open breaker: zero provider calls, stale or 503 with `Retry-After` and `cache_available:false`
- half-open: six simultaneous requests after the cooldown produce exactly one provider call; the successful probe closes the breaker
- failed probe: exactly one call (no in-place retry), cooldown doubled
- 429: `Retry-After` honored across three attempts, breaker unaffected
- token bucket exhaustion: degrades without touching the provider
- provider hang: timeout handled as a transient failure
- all three state services unreachable (its own configuration): live answer when the provider is healthy, 503 with `Retry-After` otherwise
- Gateway Rules: paths, parameters, staleness cap, field removal (answer, cache write and old cache entries), no-cache, lowered attempts and threshold, raised cooldown; broken or loosening rules refused with a 500 before anything is called

Not tested: n8n 1.x, a Redis-backed state service (the Redis token check above is a sketch), sustained load.

## Known limits

- `GET` only, and synchronous: the caller waits through retries.
- A request already past the breaker check when it trips keeps its remaining attempts.
- `breaker_update` (`applied`, `ignored_stale_claim`, `applied_without_claim_token`, `record_failed`) shows in the n8n execution data only, not in the response body.
- Cached responses are stored as-is unless you set `remove_fields` or `use_cache:false`; entries cached before a rule existed keep raw content until served or expired.
- A syntax error inside the rules node gives an empty reply; n8n shows the error and nothing is called.
