# 07 — Multi-Model LLM Router

## Business problem
Calling the same premium model for every request is slow and expensive;
calling only the cheap local model produces unreliable output for hard
tasks. This router tries the cheapest capable tier first (a free
locally-hosted model), escalates through a
mid-tier and then a premium cloud model only when needed, and treats
cost as a hard constraint rather than an afterthought — with a daily
budget that can actually stop spend, not just log it.

## Architecture

`POST /webhook/llm/complete` → `{ prompt, task_type?, max_tokens?, require_json? }`

1. **Cache check** — an identical prompt+params combination within the
   cache TTL skips every provider entirely.
2. **Classify starting tier** — `simple`/`code` tasks start at Tier 1
   (free local model); `complex` tasks skip straight to Tier 2,
   since starting there would be a guaranteed-weak attempt.
3. **One batched circuit-breaker check** for all three providers up
   front, rather than a separate round-trip per tier.
4. **Tier 1 (local model, free)** → **Tier 2 (mid-tier, paid)** → **Tier 3
   (premium, paid, last resort)** — each tier is skipped if its breaker
   is open, attempted with its own retry budget, and its response is
   quality-checked before being accepted.
5. **Fresh budget check immediately before each paid tier** (not once at
   the start) — a concurrent request could have spent the budget in the
   meantime.
6. On acceptance: cache the response, record the actual cost against the
   budget, and respond. On exhausting every tier: notify ops with the
   specific reason and respond `503`.

## Edge cases handled
- **Three different provider response shapes** — the local model
  (`{response: string}`), an OpenAI-compatible mid-tier
  (`choices[0].message.content`), and an Anthropic-style premium tier
  (`content[0].text`) each get their own extraction step. A router that
  assumes one shape breaks the moment a second provider is added.
- **Soft failure vs hard failure** — an empty or malformed response is
  escalated to the next tier *without* penalizing that provider's
  circuit breaker; a `5xx`/timeout *does* penalize it. Conflating the two
  would trip breakers on quality misses that have nothing to do with the
  provider's health.
- **Task-aware starting tier** — routing isn't purely cost-first; a task
  already known to need real reasoning skips the tier it's known to fail,
  rather than burning a guaranteed-wasted attempt (and its retry budget)
  before escalating anyway.
- **Budget checked fresh per paid tier, not once** — closes the race
  where two concurrent requests both read "budget available" before
  either has actually spent anything.
- **Runaway spend** — a paid tier is skipped (not just slowed down) once
  the daily budget is exhausted; the request either succeeds on a cheaper
  tier or fails cleanly with `budget_exhausted`, never silently overspends.
- **Circuit breakers per provider** — a provider that's actively failing
  is skipped immediately rather than burning the full retry budget on
  every single request while it's down.
- **State surviving external calls** — the same `Merge(combineByPosition)`
  pattern used throughout this portfolio, here carrying `breakers`
  (fetched once) through every subsequent Code node's return object for
  the rest of the run.
- **Cache write vs cost write reliability** — a failed cache write just
  costs a little efficiency next time; a failed cost write means the
  budget guardrail silently drifts from reality, so only the cost write
  gets retried.
- **No further fallback below Tier 3** — its failure paths (budget,
  breaker, hard failure, quality) all terminate in an explicit `503`
  with the specific reason, rather than the request hanging or returning
  something misleading.

## Environment variables expected
`CACHE_API_URL`, `CIRCUIT_BREAKER_API_URL`, `BUDGET_API_URL`,
`LOCAL_LLM_API_URL`, `MIDTIER_API_URL`, `PREMIUM_API_URL`,
`SLACK_WEBHOOK_URL`
