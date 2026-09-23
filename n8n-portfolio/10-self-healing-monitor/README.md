# 10 — Self-Healing Execution Monitor

## Business problem
Automation only pays off if failures get noticed and fixed quickly. Left
unattended, a broken workflow either fails silently until someone
downstream complains, or floods a Slack channel with a page for every
single failed run of an outage that's really one root cause. This
workflow watches an n8n instance's own execution history, tells transient
blips from real bugs, retries the ones worth retrying with backoff, pages
a human only for what actually needs one, and closes the loop by noticing
and announcing recovery — not just failure.

## Architecture

Two triggers converge on one pipeline: a **5-minute schedule** and a
**manual/on-demand webhook** (`POST /webhook/monitor/check-now`, with an
optional `{force: true}`).

1. **Poll lock** — an atomic, TTL'd lock (self-expires after 10 minutes)
   prevents two poll cycles from overlapping if one runs long; `force`
   lets an operator push a manual run through anyway.
2. **Compute the poll window** from an externally-stored watermark
   (`since` = last successful checkpoint, falling back to a 15-minute
   lookback on first run).
3. **Two lanes run in parallel** from that point:
   - **Failure lane**: paginate through the n8n API's failed-executions
     list for the window, capped at 20 pages as a runaway-loop guard.
   - **Recovery lane**: pull every currently-open incident and check
     whether its workflow has a recent successful run — independent of
     whether *this* cycle found any new failures, since a workflow can
     heal between polls.
4. **Batch-level storm check** — 5+ distinct workflows failing in the same
   window is treated as a likely shared-dependency outage: one systemic
   alert, individual auto-retries suppressed for the cycle, rather than a
   wave of retries and pages that would just pile onto an active outage.
5. **Per execution**: fetch full detail → classify the error
   (transient/permanent/unknown, keyword-based) → confirm the workflow is
   still active (skip ones a human deliberately turned off) → compute a
   stable **incident key** (workflow + failed node + failure type, *not*
   the execution id, since every retry gets a new id) → look up that
   incident's retry state → decide: retry / escalate / wait-in-backoff /
   suppress-as-storm.
6. **Retry** dispatches via the n8n API and records exponential backoff
   (5m → 15m → 45m, plus jitter) before the next attempt is even
   considered.
7. **Escalate** (permanent error, unknown error, retries exhausted, or the
   retry dispatch itself failing) checks a per-incident alert cooldown
   before paging — one page per incident per 30 minutes, not one per poll
   cycle.
8. **Recovery** posts a distinct "recovered" notice and resolves the
   incident once a fresh success shows up for a workflow that had one
   open.
9. Both lanes' summaries are merged into one digest; the lock is released
   and the watermark is advanced — **but only on a fully completed
   cycle**. If the monitor's own call to the n8n API fails, that's treated
   as the monitor itself being degraded: a distinct alert fires and the
   watermark is deliberately **not** advanced, so the next cycle re-checks
   the same window instead of silently skipping it.

## Edge cases handled
- **Overlapping poll cycles** — a TTL'd lock, not just "don't schedule
  it twice," since a crash mid-cycle must not deadlock every future run.
- **Pagination** — a busy window can have more failures than one page;
  capped at 20 pages so a non-terminating cursor can't loop forever.
- **Retry state keyed by incident, not execution id** — every retry
  produces a brand-new execution id, so tracking attempts by execution id
  would reset the count on every single try.
- **Transient vs. permanent vs. unknown** — only transient failures get
  auto-retried; permanent (validation, auth, 4xx) and unknown failures go
  straight to a human, since blindly retrying a deterministic bug just
  delays the person who needs to see it.
- **Exponential backoff with jitter** — spreads out retries for
  simultaneously-failing executions of the same incident instead of
  hammering the destination again in lockstep.
- **Failure storms** — a shared-dependency outage produces one systemic
  alert and suppressed individual retries, instead of N pages and N
  retry-storms against something already down.
- **Alert cooldown per incident** — a workflow stuck failing every poll
  gets one page, not one every 5 minutes, until it either recovers or the
  cooldown lapses.
- **Respecting intentional deactivation** — a workflow someone turned off
  while fixing it is skipped, not nagged about.
- **Monitoring the monitor** — if the n8n API itself is unreachable, that
  is its own distinct failure mode (a degraded-monitor alert), never
  silently counted as "zero failures this cycle," and the watermark isn't
  advanced so nothing gets missed.
- **Closed-loop recovery** — checked every cycle independent of new
  failures, so a workflow that quietly healed gets an explicit
  "recovered" notice and its incident closed, not left open forever.
- **State surviving external calls** — the same
  `Merge(combineByPosition)` pattern used throughout this portfolio,
  applied to every HTTP call in both lanes.
- **Dual-trigger response handling** — every terminal branch (skipped,
  degraded, completed) checks which trigger fired before deciding to
  respond (manual) or end quietly (scheduled).

## Environment variables expected
`N8N_API_URL`, `N8N_API_KEY` (sent as a header on every n8n API call —
omitted from the JSON here since header auth is normally configured via
n8n's built-in HTTP credential, not a literal value in the workflow),
`MONITOR_STATE_API_URL`, `SLACK_WEBHOOK_URL`
