# 10 — Self-Healing Execution Monitor

A single n8n workflow that watches an n8n instance's own execution history, tells transient blips from real bugs, retries the ones worth retrying with backoff, pages a human only for what actually needs one, and closes the loop by noticing and announcing recovery, not just failure.

## Business problem

Automation only pays off if failures get noticed and fixed quickly. Left unattended, a broken workflow either fails silently until someone downstream complains, or floods a channel with a page for every failed run of what is really one root cause. This monitor treats failures as **incidents** (one root cause, one record, one page) and keeps its own state honest even when the things it depends on are down.

## Design principle

Nothing is hardcoded. The n8n address and API key, the state-service address, Slack, thresholds, backoff, alert cooldown and the error-classification patterns all live in one `CONFIG` node. The monitor can watch any n8n instance reachable by its public API, and the state it needs (lock, watermark, incident records) lives behind a small HTTP contract you can back with Redis or SQL. Anything specific to your own workflows (which to ignore, which to retry less or never, nodes that must not be repeated, the wording of the alert) lives in one data-only `Monitor Rules` node, and the lock, window, backoff, storm and cooldown logic stay protected.

## Trigger

Two triggers, one pipeline:

- **Schedule**, every 5 minutes (a node setting; n8n does not allow expressions there, so change it on the node).
- **`POST /webhook/monitor/check-now`**, optional body `{"force": true}`. Runs one cycle now and returns the digest. If `CONFIG.MONITOR_AUTH_TOKEN` is set, callers must send it as the `x-monitor-token` header.

## Architecture

```
trigger → validate config/rules/auth → acquire lock (token) → read watermark → window [since, until)
        → page through failed executions (newest first, stop once a page reaches before `since`)
        → drop: already-retried-successfully, manual test runs, the monitor itself
        → storm? (STORM_THRESHOLD distinct workflows) → one rate-limited storm alert
        → per execution: fetch detail → classify transient / permanent / unknown
        → group into incidents  (workflow + failed node + failure type)
        → per incident: workflow still active? → read incident state → decide
              retry         transient, attempts left, backoff elapsed  → retry every failed execution once, charge ONE attempt
              escalate      permanent / unknown / retries exhausted / retry could not dispatch → page (unless in cooldown)
              wait_backoff  inside the backoff window
              suppress      storm
        → recovery check for every open incident (newest successful run after the last failure → resolve + announce)
        → digest → advance watermark ONLY if nothing was lost → release lock → respond (manual) or end quietly (scheduled)
```

1. **Poll lock.** An atomic, self-expiring lock stops two cycles overlapping. Acquiring it returns a **token**, and release requires that token, so a forced-through manual run can never release another cycle's lock. A crashed cycle cannot deadlock the future: the TTL expires.
2. **State service down is not "busy".** If the lock call itself fails, the cycle ends as `monitor_degraded / state_service_unavailable` (and alerts), never as "poll already in progress".
3. **Watermark window.** `since` is the last fully completed cycle's end (or a 15-minute lookback on the first run). The n8n API has no time filter, so the window is applied to each page, and paging stops as soon as a page reaches before `since`. Capped at `MAX_PAGES`; hitting the cap raises a warning in the digest instead of silently skipping older failures.
4. **Classification.** HTTP status first (5xx, 429, 408 = transient; other 4xx = permanent), then your regex lists from CONFIG (connection resets and timeouts transient; auth, not-found and validation permanent). Anything else is `unknown` and is **never auto-retried**: blindly retrying a deterministic bug only delays the person who needs to see it.
5. **Incidents, not executions.** An incident is workflow + failed node + failure type. Every retry creates a new execution id, so keying by execution id would reset the attempt counter on every try. Ten failed runs of one bug become one incident with ten executions.
6. **One attempt per incident per cycle.** Retrying every failed execution in the incident (up to `MAX_RETRIES_PER_INCIDENT_PER_CYCLE`) costs the incident a single attempt. Backoff is exponential with jitter: 5 min, 15 min, 45 min by default.
7. **Escalation with a cooldown.** A page per incident per `ALERT_COOLDOWN_MINUTES` (30), not one per cycle. If Slack rejects the post, the cooldown is **not** started, so the next cycle pages again instead of the alert silently vanishing.
8. **Storm handling.** `STORM_THRESHOLD` or more distinct workflows failing in one window is treated as one shared-dependency outage: a single storm alert (with its own cooldown) and no retries, so retries do not pile onto something that is already down.
9. **Recovery runs every cycle,** after the failure lane, whether or not this cycle found new failures. An open incident is recovered when its workflow has a successful run that started after the incident's last recorded failure. The incident is resolved first and announced second.
10. **Monitoring the monitor.** An unreachable n8n API or state service is a distinct, alerted failure (rate-limited per reason, so an outage does not page every five minutes). The watermark is not advanced, so the next cycle re-checks the same window. A scheduled run with a broken CONFIG fails loudly instead of doing nothing.

## Edge cases handled

- **Converging branches.** A plain node with several incoming branches runs once *per arrival* in n8n, so every convergence point here goes through a Merge node. (The monitor's outcome summary and the recovery summary both do.)
- **Overlapping cycles.** Token-owned TTL lock; concurrent manual runs complete or skip, never double-process.
- **Pagination.** Cursor paging with an early stop at the window edge and a hard cap that warns instead of dropping silently.
- **Already handled.** Executions that were retried successfully (`retrySuccessId`), manual editor test runs and the monitor's own runs are skipped.
- **Deactivated or deleted workflows** are skipped, not nagged about.
- **Re-failure after recovery** opens a fresh incident (new `opened_at`, attempt counter reset).
- **Partial failure of the monitor's own bookkeeping.** If incident state cannot be read or written, the cycle is `completed_with_errors`, the affected incident is not acted on blindly, and the watermark is not advanced.
- **State survives external calls.** Downstream nodes read earlier results by node name or by paired item, so no HTTP response can overwrite context.
- **Dual-trigger responses.** Every terminal path checks which trigger fired: manual calls get a JSON body, scheduled runs end quietly.

## Digest (manual response)

```json
{
  "status": "completed",
  "triggered_by": "manual",
  "window": { "since": "...", "until": "...", "bootstrapped": false },
  "failed_executions_seen": 6,
  "distinct_workflows_failing": 1,
  "storm": false,
  "counts": { "retried": 1, "escalated": 0, "escalation_suppressed_cooldown": 0, "waiting_backoff": 0,
              "suppressed_storm": 0, "skipped": 0, "state_error": 0 },
  "recovered": [],
  "still_open_count": 0,
  "warnings": [],
  "watermark_advanced": true
}
```

| Status | Meaning |
|---|---|
| 200 `completed` | Cycle finished cleanly. |
| 200 `completed_with_errors` | Finished, but something was skipped or not recorded; watermark not advanced. See `warnings`. |
| 200 `skipped` | Another cycle holds the lock (`poll_already_in_progress`). |
| 401 `unauthorized` | Missing or wrong `x-monitor-token`. |
| 500 `monitor_misconfigured` | CONFIG or `Monitor Rules` problem; `details` says what to fix (rules lines start `Rules:`). |
| 503 `monitor_degraded` | `reason` is `state_service_unavailable` or `n8n_api_unavailable`. |

## What the monitor needs from n8n

- An **API key** (Settings > n8n API) with exactly these scopes: `workflow:read`, `workflow:list`, `execution:read`, `execution:list`, `execution:retry`. Nothing else.
- `CONFIG.N8N_API_URL` pointing at the public API root, e.g. `http://localhost:5678/api/v1`.
- Monitored workflows must **save failed and successful executions** (n8n's defaults). Without saved successes, recovery cannot be detected.
- Retrying is the same operation as the Retry button in the n8n editor. Only let it run against workflows that are safe to re-run.

## Configuration (`CONFIG` node)

| Key | Default | Meaning |
|---|---|---|
| `N8N_API_URL` | `http://localhost:5678/api/v1` | Public API root of the instance to watch. |
| `N8N_API_KEY` | blank (required) | API key with the scopes above. Blank = a clear 500 / failed scheduled run. |
| `MONITOR_STATE_API_URL` | `http://localhost:4200` | Base URL of the state service. |
| `MONITOR_STATE_HEADERS_JSON` | `{}` | Headers for the state service, e.g. `{"x-api-key":"..."}`. |
| `SLACK_WEBHOOK_URL` | blank | Slack-compatible incoming webhook. Blank = no Slack messages (everything is still in the digest and the incident records). |
| `MONITOR_AUTH_TOKEN` | blank | If set, manual calls must send it as `x-monitor-token`. |
| `LOCK_TTL_SECONDS` | 600 | How long a cycle may hold the lock before it expires. |
| `BOOTSTRAP_LOOKBACK_MINUTES` | 15 | Window used on the first run, or when the watermark is missing. |
| `PAGE_SIZE` / `MAX_PAGES` | 100 / 20 | Executions per API page; page cap per cycle (warns when hit). |
| `MAX_EXECUTIONS_PER_CYCLE` | 50 | Most executions whose detail is fetched per cycle (newest first). |
| `STORM_THRESHOLD` | 5 | Distinct failing workflows in one window that count as a storm. |
| `MAX_RETRIES` | 3 | Automatic attempts per incident before escalating. |
| `BACKOFF_BASE_MINUTES` / `BACKOFF_MULTIPLIER` / `BACKOFF_JITTER_SECONDS` | 5 / 3 / 30 | Backoff = base × multiplier^(attempt−1) + random jitter. |
| `ALERT_COOLDOWN_MINUTES` | 30 | Minimum gap between pages for the same incident (also storm and monitor-level alerts). |
| `MAX_RETRIES_PER_INCIDENT_PER_CYCLE` | 10 | Most executions retried for one incident in one cycle. |
| `IGNORE_MANUAL_EXECUTIONS` | true | Skip failures from manual runs in the editor. |
| `TRANSIENT_PATTERNS_JSON` / `PERMANENT_PATTERNS_JSON` | see node | JSON arrays of case-insensitive regexes matched against the error text. |
| `REQUEST_TIMEOUT_MS` | 15000 | Timeout for every API call. |

## Per-workflow handling (`Monitor Rules` node)

Everything specific to *your* workflows sits in one data-only node, `Monitor Rules`, checked at the start of every cycle:

| Setting | What it does |
|---|---|
| `ignore_workflows` | Workflows (by id or name pattern) the monitor leaves alone. |
| `workflow_overrides` | Per workflow: fewer retries (never above `CONFIG.MAX_RETRIES`), never retry, an alert prefix such as `P1`, an owner note. |
| `never_retry_nodes` | Node names (patterns) whose transient failures are never repeated automatically, such as a card charge; a person is paged instead. |
| `alert` | Title, icon, which lines, extra line (runbook link) and the wording of each reason. |

Rules can only make the monitor more careful. The lock, window, backoff, storm suppression, alert cooldown and recovery logic cannot be changed from there. A mistake gives `500 monitor_misconfigured` with lines starting `Rules:` before anything is fetched or retried.

## State service contract

All bodies JSON. `MONITOR_STATE_HEADERS_JSON` is sent on every call. Any non-2xx or unreachable response is handled as described under "Failure behavior".

| Call | Body | Response |
|---|---|---|
| `POST /lock` | `{ttl_seconds}` | `{acquired: bool, token?: string}`. **Atomic.** |
| `POST /unlock` | `{token}` | 2xx. Releases only if the token matches; a null or unknown token is a no-op. |
| `GET /last-checked` | | `{last_checked_at: epoch ms \| null}` |
| `POST /last-checked` | `{last_checked_at}` | 2xx |
| `GET /incidents?status=open` | | `{incidents: [record, ...]}` |
| `GET /incidents/{key}` | | `200 record`, or `404` if never seen |
| `POST /incidents/{key}` | any subset of the fields below | **Upsert: merge the given fields.** Create with `status:"open"` and `opened_at = now` if absent. |
| `POST /incidents/{key}/resolve` | | Sets `status:"resolved"`, `resolved_at`. |

Incident record fields the workflow writes and reads: `incident_key`, `status` (`open`, `resolved`, or `system` for the storm-alert pseudo-record `__storm__`), `workflow_id`, `workflow_name`, `failed_node`, `failure_type`, `error_message`, `last_failure_at`, `last_seen_at`, `opened_at`, `phase` (`retrying` / `escalated`), `retries_used`, `next_retry_at`, `escalated`, `escalate_reason`, `last_alerted_at`. The workflow sends `opened_at` itself when a resolved incident re-opens. Keys are URL-encoded by the workflow.

Rules the service must implement:

- **`/lock` is atomic.** Redis: `SET lock <token> NX PX <ttl>`. SQL: one conditional insert/update, checking the affected-row count. Two callers must never both get `acquired:true`.
- **`/unlock` is compare-and-delete** on the token (Redis: a small Lua script).
- `GET /incidents?status=open` must include `workflow_id` and `last_failure_at` on each record; recovery depends on them.
- The watermark is only moved by the workflow after a clean cycle. The service just stores it.

## Failure behavior

| What breaks | What happens |
|---|---|
| State service down at lock time | `monitor_degraded / state_service_unavailable`, alert (rate-limited), nothing else attempted. |
| State service fails mid-cycle | Affected incidents are not acted on blindly; cycle is `completed_with_errors`; watermark not advanced. |
| n8n API unreachable or erroring | `monitor_degraded / n8n_api_unavailable`, alert, watermark not advanced, lock released. |
| Execution detail unavailable | Failure is classified `unknown` and escalated, not retried. |
| Retry call fails | Escalated as `retry_dispatch_failed` (cooldown applies). |
| Slack rejects a post | Cooldown not started; the next cycle pages again. |
| Invalid CONFIG | Manual: 500 naming the key. Scheduled: the run fails visibly. |

## Verification

Tested end to end on self-hosted n8n 2.35.7: the monitor watched the same instance it ran in, through n8n's real executions API, against demo workflows that fail on demand (503, 404, connection reset, a code error) behind an in-memory implementation of the state-service contract and a Slack stand-in (neither included in this repo). 86 checks plus a scheduled-trigger run, all passing, in ten configurations:

- **Defaults (39):** quiet cycle with watermark and lock release; transient failure retried through n8n (the backend really saw the re-run), incident recorded with workflow id, failed node and ~5 min backoff; the next cycle waits inside backoff without touching the backend; attempt 2 with ~15 min backoff; attempt 3, then escalation as retries exhausted with no fourth retry; a failure inside the alert cooldown produces no second page; six failures of one bug are one incident, one attempt, six executions retried; permanent 404 escalated and never retried; unknown code error escalated and never retried; connection reset classified transient; five distinct workflows failing is a storm, five incidents suppressed, zero retries; recovery resolves an incident and a later failure opens a fresh one; a retry that succeeds is recovered in the same cycle and its original execution is skipped afterwards; a held lock skips the cycle, `force` runs through without releasing the other cycle's lock; state service down is degraded, not "busy"; three concurrent manual runs never double-process.
- **Auth + Slack + state headers (11):** 401 on missing and wrong token; one alert naming workflow and node, cooldown suppresses repeats; a failed Slack post does not start the cooldown and the next cycle pages again; one storm alert then cooldown; recovery announced; degraded alerts rate-limited per reason.
- **Pagination (`PAGE_SIZE=2`) (3):** five failures found across three pages; with `MAX_PAGES=2` only four are examined and a warning is raised.
- **n8n API unreachable (3), missing API key (1).**
- **Monitor Rules (29):** an ignored workflow is skipped with no retry, incident or alert; a workflow limited to one retry escalates after one, with its prefix, owner note, custom title, chosen lines and custom reason wording; `auto_retry: false` pages without ever re-running the execution; a transient failure at a never-retry node is not repeated while another workflow's is; unaffected failures keep the default wording; broken rules give a clear 500 naming each mistake and take no lock; a limit above the CONFIG ceiling is refused; a syntax error in the node processes nothing.
- **Scheduled trigger:** a real scheduled run acquired and released the lock, advanced the watermark and ended quietly.

Not tested: n8n 1.x, queue mode, a real Slack workspace, a Redis- or SQL-backed state service, instances with thousands of failures per hour, and executions whose status is `crashed`, `canceled` or `waiting` (only `status=error` is polled).

## Known limits

- **Failures that happen during a storm are not auto-retried afterwards.** They are suppressed for that cycle and the window then moves on; the storm alert is the signal to fix the shared dependency and replay from n8n.
- Only executions with status `error` are monitored.
- At most `MAX_EXECUTIONS_PER_CYCLE` executions get their detail fetched per cycle (newest first); a very bad hour can exceed it. Execution detail that n8n omits for size is classified `unknown`.
- Recovery means "the workflow has a successful run after the last failure". A workflow with several independent paths could look recovered because a different path succeeded.
- Monitor-level alert rate-limiting uses the workflow's own static data, which only persists for an active (published) workflow.
- The schedule interval and the retry/wait settings on nodes are node settings, not CONFIG values.
- The monitor does not monitor itself: use n8n's own Error Workflow setting for that.
