# 07 — Multi-Model LLM Router

A single n8n workflow that answers an LLM request with the **cheapest model that gives a usable answer**, and treats cost as a hard limit. Exposed as one endpoint, `POST /webhook/llm/complete`.

You list your models as tiers (cheapest first). For each request the router skips tiers that are down or unsuitable, reserves budget before paying for a call, checks what comes back, and moves to the next tier only when it has to. It returns the answer together with a trace of what it tried and what it cost.

The interesting part is the failure handling. A naive router either keeps hammering a dead provider, treats a model's bad answer as an outage, or checks the budget and then lets ten concurrent requests all spend it. This one separates "the provider is unhealthy" from "the answer was unusable", reserves worst-case cost atomically before every paid call, and settles it to the real cost afterwards.

## Design principle

Nothing is hardcoded. The tiers themselves (how many, in what order, which API style, which URL, model, price and credentials) are a JSON value in the `CONFIG` node, and so are every service URL, threshold and timeout. The router talks to three small HTTP services (circuit breaker, cache, budget), each of which can be switched off, plus an optional Slack-compatible webhook. Swap any of them by changing a URL; no single vendor is assumed. Three API styles are built in: `ollama` (`/api/generate`), `openai` (`/v1/chat/completions`, which many hosts and gateways speak) and `anthropic` (`/v1/messages`).

## Request

`POST /webhook/llm/complete`

```json
{
  "prompt": "Extract the invoice number and total as JSON: ...",
  "task_type": "code",
  "max_tokens": 512,
  "require_json": true
}
```

| Field | Required | Meaning |
|---|---|---|
| `prompt` | yes | Non-empty string, at most `MAX_PROMPT_CHARS` (32,000) characters. |
| `task_type` | no | `simple` (default), `code` or `complex`. A tier is only used for task types in its `handles` list. If you leave it out, the `Routing Policy` node's `classify` rules may pick one; a value you send always wins. |
| `max_tokens` | no | Integer from 1 to `MAX_ALLOWED_TOKENS` (8,192). Default `DEFAULT_MAX_TOKENS` (1,024). Also sets the worst-case cost reserved for paid tiers. |
| `require_json` | no | `true` makes the router ask the model for JSON only, strip a ```` ```json ```` fence, and accept an answer only if it parses. Default `false`. |

If `CONFIG.ROUTER_AUTH_TOKEN` is set, callers must send it as the `x-router-token` header.

## Responses

| Status | Body | When |
|---|---|---|
| 200 | `{request_id, text, task_type, provider, tier_index, cached:false, cost_usd, usage:{input_tokens, output_tokens}, json?, warnings?, attempts?}` | A tier produced an acceptable answer. `provider` is the tier's `name`. `cost_usd` is the total over every call this request made (a paid tier that returned unusable text still cost money). `json` is the parsed value when `require_json` is true. `attempts` is omitted if `INCLUDE_ATTEMPTS=false`. |
| 200 | `{request_id, text, task_type, provider, tier_index, cached:true, cost_usd:0, json?}` | Served from the cache. No provider was called. |
| 400 | `{error:"validation_failed", details:[...]}` | Bad input. Nothing was called. |
| 401 | `{error:"unauthorized"}` | Missing or wrong `x-router-token`. |
| 500 | `{error:"router_misconfigured", details:[...]}` | Invalid CONFIG or `TIERS_JSON` (every problem is listed, with the tier index). |
| 503 | `{error:"all_providers_unavailable", reason, request_id, attempts?}` | No tier produced an acceptable answer. `reason` is one of the values below. |

`warnings` can contain `budget_settle_failed` (the answer is fine, but the budget service did not record the final cost; ops is alerted), `budget_unavailable_fail_open` (`BUDGET_FAIL_MODE=open` and the budget service was unreachable, so the limit was not enforced) or `breaker_unavailable_fail_open`.

**503 reasons**, first match wins: `daily_budget_exhausted` (a paid tier was refused by the budget service), `quality_check_failed_all_tiers` (every tier that answered gave an empty or invalid answer), `all_providers_failed` (at least one tier was tried and failed hard), `budget_service_unavailable` (budget service unreachable, fail-closed), `all_circuits_open` (every eligible tier is skipped by its breaker), `no_eligible_tier` (no tier handles this `task_type`).

**An `attempts` entry** is `{tier, outcome, reason?, http_status?, attempts_made, latency_ms, cost_usd, probe?, notes?}`. `outcome` is `accepted`, `soft_failure` (answered, but empty or invalid JSON), `hard_failure` (network error, timeout, 5xx, 429, 401/403/404, wrong response shape), `rejected` (the provider refused this request with another 4xx) or `skipped` (`reason`: `task_type_not_handled`, `breaker_open`, `budget_denied`, `budget_unavailable`).

Malformed JSON never reaches the workflow: n8n itself rejects it with a `4xx`, and a body over 16 MB with an error status.

## Architecture

```
request → auth + CONFIG/TIERS sanity + shape validation            (400 / 401 / 500)
        → request id (UUID) and cache key (SHA-256 of prompt, task type, max_tokens, require_json, models)
        → cache lookup   hit and younger than CACHE_TTL_SECONDS → reply 200 cached
        → for each tier in TIERS_JSON order that handles this task_type:
              breaker:  closed → go   |  open, cooling → skip   |  open, cooldown over → claim THE probe, or skip
              budget:   paid tier → atomically RESERVE worst-case cost  (refused → skip)
              call:     provider-specific request, own timeout; transient failure → retry (RETRIES_PER_TIER)
              assess:   hard failure | soft failure (empty / invalid JSON) | rejected | accepted
              record:   breaker success/failure (not for quality misses or rejected requests)
              settle:   replace the reservation with the real cost (0 for a failed call)
              accepted → reply 200            otherwise → next tier
        → no tier left → reply 503 with a reason
        → (after the reply) store in the cache, send throttled alerts
```

1. **Validation first.** Auth, CONFIG, every tier in `TIERS_JSON` and the request are checked before anything is called. Every rejection is a specific 4xx/5xx body.
2. **Tiers are data.** A tier is `{name, kind, url, model, handles, paid, prices, timeout, credentials}`. The loop runs over however many you list, in the order you list them. There is no "tier 1/2/3" in the workflow.
3. **Task-aware start.** A tier declares which task types it `handles`. The default local tier handles `simple` and `code`, so a `complex` request skips straight past it instead of burning an attempt that is known to be weak.
4. **Per-tier circuit breaker.** Each tier has its own breaker, using the same contract as the Circuit Breaker gateway workflow in this portfolio. After `BREAKER_FAILURE_THRESHOLD` consecutive failures it opens and the tier is skipped without a call. When the cooldown ends, exactly one request wins an atomic claim and tests the provider; success closes the breaker, failure re-opens it with a longer cooldown (doubling up to `BREAKER_MAX_COOLDOWN_SECONDS`).
5. **Provider health is not answer quality.** A timeout, 5xx, 429, 401/403/404 or a 200 in the wrong shape counts against the breaker. An empty answer, invalid JSON, or a 400-type refusal of this particular request does not: the provider is fine, the answer just was not usable, so the router escalates without punishing it.
6. **Transient failures are retried, configuration errors are not.** Network errors, timeouts, 5xx, 429 and 408 are retried up to `RETRIES_PER_TIER` times. 401/403/404 and wrong-shape replies are not retried (retrying cannot fix a bad key or URL). The breaker hears about the tier once per request, after retries. Between retries it waits (see "Back-off"): exponential, with jitter, honouring the provider's `Retry-After`, within a total wait budget per request.
7. **Budget is reserved, not just checked.** Before each paid call the router asks the budget service to atomically reserve the worst case (estimated prompt tokens plus the full `max_tokens` at that tier's prices). If the reservation is refused the tier is skipped. After the call the reservation is replaced by the actual cost from the provider's `usage` figures (prices come from `TIERS_JSON`); a failed call settles at 0. Check-then-spend lets concurrent requests all pass the check; reserve-then-settle does not.
8. **Quality check.** Empty or whitespace-only text is a soft failure. With `require_json`, the model is told to answer in JSON only, a surrounding code fence is stripped, and the text must parse (any JSON value). A soft failure moves to the next tier.
9. **Cache.** Identical requests (same prompt, task type, `max_tokens`, `require_json` and configured models) are answered from the cache without touching any provider. The router checks the entry's age itself, so the store needs no expiry feature, and an entry that is malformed, expired or not valid JSON for a `require_json` request is a miss, never an error. Changing your models changes the key.
10. **Everything after the reply is after the reply.** Cache writes and Slack alerts happen after the response is sent, so a slow cache or dead Slack never delays or changes an answer.
11. **Alerts are throttled.** The same alert (e.g. `all_providers_failed`) is sent at most once per `ALERT_COOLDOWN_SECONDS`, and the cooldown only starts when Slack actually accepted the message.

### Back-off between retries

A retry used to follow the failure immediately, which hammers a provider that is already struggling and burns the retry on the very moment it is still down. Now each retry on the same tier waits first:

- **Exponential**: `RETRY_BACKOFF_MS`, then double, then double again, never more than `RETRY_BACKOFF_MAX_MS`.
- **Jitter**: each wait varies by up to `RETRY_JITTER_PERCENT` either way.
- **Retry-After**: on a `429` or `503` with a `Retry-After` header, the wait is at least that long. If it is longer than the cap, the tier is not retried at all and the router moves straight to the next tier. A header that cannot be understood is ignored; `500`s are never treated as rate limits.
- **A total budget**: all the waits in one request add up to at most `RETRY_MAX_TOTAL_WAIT_MS`. When the next wait would go over, the tier is not retried again.
- Only transient failures wait (network errors, timeouts, 5xx, 429, 408). A 401/403/404 or a 400 is never retried, so never waited for.
- The attempts trace shows what happened: `retry_waits_ms` (the waits before each retry on that tier) and, when a retry was refused, `retry_skipped` (`retry_after_exceeds_cap` or `retry_wait_budget_exhausted`). The waiting is not counted in a call's `latency_ms`.

The caller waits too: worst case a request can take the sum over tiers of `(RETRIES_PER_TIER + 1) x timeout_ms`, plus at most `RETRY_MAX_TOTAL_WAIT_MS`.

## Edge cases handled

- **Three provider response shapes.** Ollama (`{response}`), OpenAI-style (`choices[0].message.content`) and Anthropic-style (`content[].text`) each have their own request builder and extractor. A 200 that is not in the shape the tier is configured for is a hard failure ("misconfigured tier"), not an empty answer.
- **Quality miss vs outage.** Covered above; a model that keeps giving bad JSON never trips its own breaker.
- **Paid calls that return junk still cost money.** They are settled at their real cost and included in `cost_usd`, so the budget reflects reality.
- **Concurrent requests at the limit.** Six simultaneous requests with room for two paid calls: two are served, four get `daily_budget_exhausted`, and the daily limit holds.
- **One probe at a time.** Two simultaneous requests against a breaker whose cooldown just ended: one tests the provider, the other goes to the next tier.
- **Dependencies that fail are not allowed to take the router down.** Cache down: every request is a miss. Breaker service down: fail open (and the trace says so). Budget service down: fail closed by default (paid tiers refused, free tier still works), or `BUDGET_FAIL_MODE=open` to keep serving without the limit.
- **Settle failure.** The answer is still delivered; the settle is retried 3 times, and if it still fails the reply carries `budget_settle_failed` and Slack is told, so a drifting budget is never silent.
- **Hung providers.** Each tier has its own `timeout_ms`; a hang is a retried, then escalated, failure.
- **Input hardening.** Non-string prompts, over-long prompts, bad `task_type`, non-integer or out-of-range `max_tokens` and non-boolean `require_json` are 400s.
- **State survives external calls.** Downstream nodes read earlier results by node name, so no HTTP response can overwrite request state.

## Tiers (`TIERS_JSON`)

A JSON array in `CONFIG`, **cheapest first**. The default:

```json
[
  {"name": "local",   "kind": "ollama",    "url": "http://localhost:4500/ollama/api/generate", "model": "llama3",
   "handles": ["simple", "code"], "paid": false, "timeout_ms": 20000},
  {"name": "midtier", "kind": "openai",    "url": "http://localhost:4500/openai/v1/chat/completions", "model": "mid-model", "api_key": "",
   "paid": true, "price_in_per_mtok": 0.5, "price_out_per_mtok": 1.5, "timeout_ms": 30000},
  {"name": "premium", "kind": "anthropic", "url": "http://localhost:4500/anthropic/v1/messages", "model": "premium-model", "api_key": "",
   "paid": true, "price_in_per_mtok": 3, "price_out_per_mtok": 15, "timeout_ms": 60000}
]
```

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | 1–64 characters (letters, digits, `.` `_` `-`), unique. Used as the breaker and budget key and as `provider` in replies. |
| `kind` | yes | `ollama`, `openai` or `anthropic`: the request/response style. Any server that speaks one of them works (a gateway, vLLM, LM Studio, another vendor's compatible endpoint). |
| `url` | yes | The **full endpoint URL** (not a base URL). |
| `model` | yes | Model name sent to that endpoint. |
| `handles` | no | Task types this tier serves. Default all three. |
| `paid` | no | `true` makes the router reserve and settle budget around calls to this tier. Default `false`. |
| `price_in_per_mtok`, `price_out_per_mtok` | no | USD per million input / output tokens. Used for the reservation and the final cost. Default 0. **Keep these current: the budget is only as accurate as these numbers.** |
| `timeout_ms` | no | Per-attempt timeout. Default 30,000. |
| `api_key` | no | Sent as `Authorization: Bearer …` (`ollama`, `openai`) or `x-api-key` (`anthropic`). |
| `headers` | no | Extra headers (e.g. an organisation id). Applied last, so they can override. |
| `anthropic_version` | no | Value of `anthropic-version`. Default `2023-06-01`. |

Worst-case latency of a request is the sum over tiers of `(RETRIES_PER_TIER + 1) × timeout_ms`, plus at most `RETRY_MAX_TOTAL_WAIT_MS` of back-off; size your timeouts and retries with the caller's patience in mind.

**Credentials.** `api_key` values are stored in the workflow's `CONFIG` node, in plain text. That is convenient for a self-hosted instance you control; do not export or commit a workflow that holds real keys. If you prefer n8n credentials or a secrets store, keep keys out of `TIERS_JSON` and put a small authenticating gateway in front of the provider (then `url` points at the gateway and `api_key` is blank).

## Configuration (`CONFIG` node)

| Key | Default | Meaning |
|---|---|---|
| `TIERS_JSON` | three tiers (above) | The providers, cheapest first. |
| `CACHE_API_URL` | `http://localhost:4500` | Base URL of the cache service. |
| `BREAKER_API_URL` | `http://localhost:4500` | Base URL of the circuit-breaker service. |
| `BUDGET_API_URL` | `http://localhost:4500` | Base URL of the budget service. |
| `SERVICE_HEADERS_JSON` | `{}` | Headers sent to the three services, e.g. `{"x-api-key":"…"}`. |
| `SLACK_WEBHOOK_URL` | blank | Slack-compatible incoming webhook. Blank = no alerts. |
| `ROUTER_AUTH_TOKEN` | blank | If set, callers must send it as `x-router-token`. |
| `CACHE_ENABLED` / `BREAKER_ENABLED` / `BUDGET_ENABLED` | true | Switch a service off entirely (its URL is then not needed). With the budget off, paid tiers are called without a limit but cost is still reported. |
| `CACHE_TTL_SECONDS` | 3600 | How old a cache entry may be and still be served. |
| `DAILY_BUDGET_USD` | 5 | Limit passed to the budget service with every reservation. |
| `BUDGET_FAIL_MODE` | `closed` | If the budget service cannot be reached: `closed` refuses paid tiers, `open` calls them anyway. |
| `BREAKER_FAILURE_THRESHOLD` | 3 | Consecutive failures that open a tier's breaker. |
| `BREAKER_COOLDOWN_SECONDS` | 30 | First cooldown before a probe; doubles after each failed probe. |
| `BREAKER_MAX_COOLDOWN_SECONDS` | 600 | Cap on the cooldown. |
| `PROBE_CLAIM_TTL_SECONDS` | 60 | How long a probe claim is held. Set it above your slowest tier's `timeout_ms` × attempts. |
| `RETRIES_PER_TIER` | 1 | Extra attempts after a transient failure (so 1 = at most 2 calls per tier). |
| `RETRY_BACKOFF_MS` | 500 | Wait before the first retry; it doubles for each further retry on the same tier (500, 1000, 2000, ...). 0 = retry at once. |
| `RETRY_BACKOFF_MAX_MS` | 4000 | No single wait is longer than this. Also the longest `Retry-After` the router will wait for. |
| `RETRY_JITTER_PERCENT` | 20 | Each wait varies randomly by up to this percent either way, so many requests do not all retry at the same instant. 0 = exact. |
| `RETRY_MAX_TOTAL_WAIT_MS` | 8000 | The most time one request will spend waiting between retries, across all its tiers. When the next wait would go over, that tier is not retried again and the request moves on. |
| `RETRY_HONOUR_RETRY_AFTER` | `true` | On a 429 or 503 that carries a `Retry-After` header (seconds, or an HTTP date), wait at least that long. If it is longer than `RETRY_BACKOFF_MAX_MS`, do not retry that tier (it would only fail again) and move on at once. |
| `MAX_PROMPT_CHARS` | 32000 | Longest accepted prompt. |
| `DEFAULT_MAX_TOKENS` / `MAX_ALLOWED_TOKENS` | 1024 / 8192 | Default and ceiling for `max_tokens`. |
| `MIN_RESPONSE_CHARS` | 1 | Shortest answer that counts as non-empty. |
| `INCLUDE_ATTEMPTS` | true | Include the `attempts` trace in replies. Set false if you do not want to expose tier names and costs to callers. |
| `ALERT_COOLDOWN_SECONDS` | 300 | Minimum gap between identical alerts. |
| `REQUEST_TIMEOUT_MS` | 5000 | Timeout for cache, breaker, budget and Slack calls. |

## Service contract

All bodies are JSON. `SERVICE_HEADERS_JSON` is sent on every cache, breaker and budget call. `{name}` is the tier name (URL-encoded).

**Circuit breaker** (same contract as the Circuit Breaker gateway workflow)

| Call | Body | Response |
|---|---|---|
| `GET /breaker/{name}` | none | `200 {status:"closed"\|"open", next_probe_at (ms epoch or null), trip_count}`. An unknown tier is `closed`. |
| `POST /breaker/{name}/claim-probe` | `{ttl_seconds}` | `200 {claimed:true\|false}`. **Must be atomic:** true only if the breaker is open, the cooldown has passed and nobody holds a live claim. |
| `POST /breaker/{name}/record-success` | `{is_probe}` | `200`. A probe success closes the breaker and resets the trip count; an ordinary success resets the failure streak. |
| `POST /breaker/{name}/record-failure` | `{is_probe, error_message, failure_threshold, cooldown_seconds, max_cooldown_seconds}` | `200`. A failed probe re-opens with a longer cooldown; an ordinary failure counts toward the threshold. |

**Cache**: `GET /cache/{key}` → `200 {body, cached_at}` or `404`; `POST /cache/{key}` `{body, cached_at}` → `200`. `key` is a 64-character hex string. `body` is `{text, provider, tier_index}`. The router enforces the TTL itself.

**Budget**

| Call | Body | Response |
|---|---|---|
| `POST /budget/reserve` | `{request_id, tier, amount_usd, daily_limit_usd}` | `200 {granted:true\|false, remaining_usd}`. **Must be atomic check-and-reserve** against the current day's total (reservations count). Idempotent for the same `(request_id, tier)`. Defining "the current day" (time zone, reset time) is the service's job. |
| `POST /budget/settle` | `{request_id, tier, actual_usd}` | `200`. Replaces that reservation with the actual cost. Idempotent. A missing reservation should be a `404`. |

**Alerts**: `POST` `{text}` to `SLACK_WEBHOOK_URL`.

## Failure behavior

| What breaks | What happens |
|---|---|
| A tier times out, refuses the connection, returns 5xx/429 | Retried up to `RETRIES_PER_TIER` (with back-off between tries), then the next tier; the breaker is told once; after enough failures the tier is skipped without a call. |
| A tier returns 401/403/404 or a wrong-shape 200 | Not retried; counts against the breaker; next tier; the trace says what to check (URL, key, `kind`). |
| A tier refuses this request (other 4xx) | Not retried, no breaker penalty, next tier. |
| A tier answers with empty text or invalid JSON | Next tier; no breaker penalty; paid cost is still settled. |
| Every tier fails | 503 with a specific reason and the trace; one throttled Slack alert. |
| Budget exhausted | Paid tiers skipped; a free tier can still answer; otherwise 503 `daily_budget_exhausted`. |
| Budget service unreachable | `closed` (default): paid tiers refused, free tier still works. `open`: paid tiers used, reply warns. |
| Budget settle fails | Answer delivered with `budget_settle_failed`, retried 3 times, Slack alert. |
| Breaker service unreachable | Fail open: the tier is tried anyway; the trace notes it. |
| Cache unreachable, slow or returning junk | Treated as a miss; cache writes failing is ignored. |
| Slack down or not configured | No effect on the reply. |
| Invalid CONFIG / TIERS_JSON | 500 `router_misconfigured` naming every bad setting. |

## Verification

Tested end to end on self-hosted n8n 2.35.7 against an in-memory reference implementation of the service contracts above and fake providers in all three API styles (not included in this repo). 136 checks, all passing, across fourteen configurations (defaults with Slack on; a 2 s breaker cooldown; auth token + service headers + per-tier keys; cache, breaker and budget all off; all three services unreachable; `BUDGET_FAIL_MODE=open`; `RETRIES_PER_TIER=0`; an 800 ms tier timeout; two tiers; four tiers; deliberately bad CONFIG; and three routing-policy configurations: the example policy, a deliberately broken one, and one with a syntax error):

- request bodies and auth headers for each API style; usage-based cost; JSON-only instruction for each style
- cache: hit with a fresh request id, no provider call; different `max_tokens`/`task_type`/`require_json` are different keys; stale, malformed and not-valid-JSON entries are misses
- complex tasks skip the local tier; reservation = worst-case, settle = actual, budget service holds the actual figure
- `require_json`: fenced JSON, arrays, invalid JSON escalating without a breaker failure; empty and whitespace answers
- 500, 429 (retried), 401 (not retried, breaker hears), 400 (neither), wrong shape, non-JSON body, one blip then success
- breaker: opens after 3 failing requests, skips without calling, no probe while cooling, probe success closes, probe failure re-opens with a longer cooldown, two simultaneous requests share one probe, a real 2 s cooldown elapsing, backoff cap, independent breakers per tier
- everything failing: 503 `all_providers_failed`, reservations released, one alert, no second alert inside the cooldown, alert again after it
- soft failures on every tier: `quality_check_failed_all_tiers`, paid junk settled at real cost
- budget exhausted: paid tiers refused, free tier still served; six concurrent requests with room for two: exactly two served, limit holds
- settle failing (3 attempts, warning, alert); reserve failing (fail closed); budget service unreachable (closed and open)
- a hung provider (timeout, retry, escalate); two-tier and four-tier configurations
- ten kinds of bad input are 400s and touch nothing; malformed JSON refused by n8n; a prompt exactly at the limit accepted; 401s; bad CONFIG and bad `TIERS_JSON` entries named with their index

Not tested: live OpenAI / Anthropic / Ollama endpoints (response shapes follow their public formats), n8n 1.x, queue mode, a database-backed cache/breaker/budget, real Slack, sustained load, streaming.

## Known limits

- **Only tested against fakes.** The three API styles are implemented from their public formats. Run a few requests against your real endpoints before relying on it; providers differ in small ways (e.g. an OpenAI-compatible host that returns `content: null` is treated as an empty answer).
- **Back-off waits inside the request.** The caller waits while a retry backs off (at most `RETRY_MAX_TOTAL_WAIT_MS` in total), so keep that budget below your caller's timeout. The router does not remember a provider's `Retry-After` for later requests; the circuit breaker is what keeps traffic off a failing tier.
- **Cost accuracy depends on the prices you enter.** Reservations use your configured prices and an estimate of 1 token per 4 characters; the final cost uses the provider's reported usage (or the same estimate if the provider reports none).
- **A probe claim can be left unused.** If a request wins the breaker probe and then its budget reservation is refused, the claim is held until `PROBE_CLAIM_TTL_SECONDS` expires, which delays the next probe.
- **No streaming.** The caller gets the whole answer at once.
- **One user message per call.** The request is a single prompt, not a chat history or system prompt (the router adds a system instruction only for `require_json`).
- **The day boundary belongs to the budget service.** The router passes a daily limit; what counts as a day is implemented by your service.
- **Keys sit in `CONFIG`.** See "Credentials" above.
- **The cache stores answers.** Keys are hashes, but the cached text is stored as returned. Do not cache prompts whose answers must not be stored, or set `CACHE_ENABLED=false`.
- **Synchronous.** The caller waits for every tier tried; see the worst-case latency formula.
- **Alert throttle memory** lives in n8n's workflow static data: it survives restarts of an active workflow, and in our tests it also survived re-importing the workflow over itself (delete the workflow first for a clean slate).
- n8n's own `422` for malformed JSON includes a stack trace in its body; put a gateway in front if that matters.
