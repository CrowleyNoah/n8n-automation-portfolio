# 05 — Human-in-the-Loop Approval Gate

A single n8n workflow that puts a person between a request and a risky action (a refund, a deploy, a data deletion). A system submits a request; the workflow records it, asks approvers in Slack, replies "pending" at once, and waits. An approver clicks Approve or Deny (or calls an API); the decision is applied exactly once, the action runs once with everything it needs, and the requester is told what happened. If nobody answers in time the request is auto-declined and the requester is told that too.

The hard part is not the Slack message. It is making the decision trustworthy: only the right people can approve, two clicks (or a click racing the timeout) can never both win, a store outage is never mistaken for "already resolved", and an action that was approved but failed to run is reported rather than lost.

## Design principle

Nothing is hardcoded. Every service URL, secret, approver list and timeout limit lives in the `CONFIG` node. The workflow talks to three small HTTP services (an approval store, an action executor, a requester-notification service) plus a Slack-compatible webhook. Any backend that honours the contracts below works.

## Request endpoint

`POST /webhook/approvals/request`

```json
{ "action_id": "refund-1001", "action_type": "refund", "requested_by": "bob",
  "summary": "Refund order 42", "details": {"order": 42}, "amount": 25, "timeout_minutes": 60 }
```

| Field | Required | Meaning |
|---|---|---|
| `action_id` | yes | Your unique id for this request, 1–128 characters: letters, digits and `. _ : -`. Submitting the same id twice never creates a second request. |
| `action_type` | yes | What kind of action (1–100 characters). Passed to the executor. |
| `requested_by` | yes | Who asked (1–200 characters). The requester gets the outcome notices; also used for the self-approval check. |
| `summary` | yes | Human-readable description (1–1000 characters), shown to approvers. |
| `details` | no | JSON object passed to the executor and shown to approvers (max 20000 characters of JSON). |
| `amount` | no | Number ≥ 0, passed to the executor. |
| `timeout_minutes` | no | How long approvers have. Between `MIN_TIMEOUT_MINUTES` and `MAX_TIMEOUT_MINUTES`; default `DEFAULT_TIMEOUT_MINUTES`. Out-of-range values are rejected, not silently changed. |

If `CONFIG.REQUEST_AUTH_TOKEN` is set, send it as header `x-request-token`.

| Status | Body | When |
|---|---|---|
| 200 | `{status:"pending", action_id, expires_at, prompt_delivered, prompt:"sent"\|"failed"\|"not_configured"}` | Recorded; approvers asked. `prompt_delivered:false` means nobody was actually asked in Slack (the requester is also notified if `NOTIFY_API_URL` is set). |
| 200 | `{status:"already_requested", action_id, state, expires_at}` | That `action_id` already exists. No second prompt, no second wait. |
| 400 | `{error:"validation_failed", details:[...]}` | Bad input. Nothing was created. |
| 401 | `{error:"unauthorized"}` | Missing or wrong `x-request-token`. |
| 500 | `{error:"approval_misconfigured", details:[...]}` | Invalid CONFIG (every problem listed). |
| 503 | `{error:"approval_store_unavailable", retry:true}` | The request could not be recorded; nothing was sent to approvers; ops alerted. |

Malformed JSON never reaches the workflow: n8n itself rejects it with a `4xx`.

## Decision endpoint

`POST /webhook/approvals/respond` accepts either a **Slack interaction** (the Approve / Deny buttons the workflow posts) or a **direct API call**:

```json
{ "action_id": "refund-1001", "decision": "approve", "responder": "alice" }
```

`decision` is `approve` or `deny`. For Slack clicks the responder is the Slack user (username, name or id).

| Status | Body | When |
|---|---|---|
| 200 | `{status:"approved"\|"denied", action_id, resolved_by, auth:"slack"\|"token"\|"open"}` | This call made the decision. The reply is sent **before** the action runs. |
| 200 | `{status:"already_resolved", action_id, state}` | Someone (or the timeout) decided first. |
| 400 | `{error:"validation_failed", details}` | Unreadable decision. |
| 401 | `{error:"unauthorized"}` | Missing/invalid token or Slack signature, or a stale (over 5 minutes) Slack timestamp. |
| 403 | `{error:"not_an_approver"}` | Authenticated, but not in `ALLOWED_APPROVERS`. |
| 403 | `{error:"self_approval_not_allowed"}` | The requester tried to approve their own request. (Denying is allowed.) |
| 404 | `{error:"unknown_action"}` | No such `action_id`. |
| 410 | `{error:"request_expired"}` | The deadline passed before this answer; the request is auto-declined. |
| 500 | `{error:"approval_misconfigured", details}` | Invalid CONFIG. |
| 503 | `{error:"approval_store_unavailable", retry:true}` | Could not record the decision; **try again**. It is never reported as already resolved. |

## Reconcile (stuck requests)

Two things can go wrong when n8n is down at the wrong moment: a request can stay `pending` past its deadline (the timer was lost), and a request can be recorded `approved` but never run. The reconcile job finds both. It is **off by default** (`RECONCILE_ENABLED=false`) and runs two ways: on a schedule (the `Schedule: Reconcile` node, every 15 minutes, edit it there) and on demand with `POST /webhook/approvals/reconcile` and the header `x-reconcile-token`.

1. **Expire.** (`RECONCILE_EXPIRE`, default true.) Requests still `pending` after `expires_at` are auto-declined through the same atomic `timeout` resolve the normal timer uses, and the requester is told with the policy's `requester_timeout` wording.
2. **Run what was approved but never run.** Approved requests with no `execution` and a decision older than `RECONCILE_STALE_SECONDS`:
   - with `RECONCILE_EXECUTE=false` (default) they are only **reported** to Slack and in the reply (`not_executed`);
   - with `RECONCILE_EXECUTE=true` each one is claimed first (`POST /approvals/{id}/execution-claim`, atomic with a lease), then sent to the executor **once** with the original idempotency key `approval:<action_id>`, and the outcome is recorded with `via:"reconcile"`.

What it will never do: run a request that has fewer approvals than the policy requires (reported as `underapproved`), touch anything the store returns that is not really `pending`/past deadline or `approved`/unexecuted/old enough (it re-checks every record itself, so a store that ignores the filters cannot make it act on the wrong ones), run the same request twice (the claim, plus the idempotency key), or retry a request whose execution is already recorded `failed` or `unknown` (a person decides those; ops get an ATTENTION line).

Reply: `{status:"ok"|"incomplete", expire, execute, expired, expire_skipped, executed, execution_failed, execution_unknown, not_executed, underapproved, skipped, problems}`. `incomplete` means the store could not list something; nothing in that part was run. At most `RECONCILE_MAX_PER_RUN` records are handled per step per run.

| Status | Body | When |
|---|---|---|
| 200 | the summary above | The job ran. |
| 401 | `{error:"unauthorized"}` | Missing or wrong `x-reconcile-token` (API calls only; the schedule needs no token). |
| 403 | `{error:"reconcile_disabled"}` | `RECONCILE_ENABLED` is not `true`. |
| 500 | `{error:"reconcile_misconfigured", details}` | Token under 8 characters, `RECONCILE_STALE_SECONDS` not larger than the executor timeout, `RECONCILE_MAX_PER_RUN` not 1 to 200, or a non-boolean switch. A scheduled run tells ops in Slack. |
| 500 | `{error:"approval_misconfigured", details}` | Invalid CONFIG or Approval Policy. |

Order of checks: switch, reconcile settings, token, CONFIG and policy.

## Architecture

```
REQUEST  POST /approvals/request
  → auth (optional token) + CONFIG + input validation                       (400 / 401 / 500)
  → create the record (atomic create-or-return-existing)   duplicate → 200 already_requested
                                                          store down → 503 + ops alert
  → Slack prompt (Approve / Deny, escaped text, 3 tries)
  → reply 200 "pending" (+ whether the prompt was delivered)
  → [prompt failed → tell the requester]
  → Wait until the deadline (saved to the database if over ~65 s)
  → atomic "timeout" resolve: claimed → tell requester + Slack     already answered → nothing
                              store error → alert ops it may be stuck pending

DECISION POST /approvals/respond   (Slack button  or  API call)
  → parse → verify Slack signature (HMAC) → authorize: signature OR token, then ALLOWED_APPROVERS   (401 / 403)
  → look up the record: unknown 404 · store error 503 · already decided → already_resolved · self-approval 403
  → atomic resolve: claimed → approved / denied      too late → 410      lost the race → already_resolved
  → reply
  → denied: tell requester + Slack
  → approved: run the action ONCE (idempotency key) → record the result → tell requester; alert ops if it failed
```

1. **The store decides, not the workflow.** `resolve` is a single atomic compare-and-set in the store. Two clicks, or a click racing the timeout, can never both win: exactly one call gets `claimed:true`. The workflow never trusts its own earlier read.
2. **The timer does not resume approvals.** The Wait node only auto-declines what is *still pending*. An approval is handled entirely by the decision flow, so a late timer after an approval is a harmless no-op (no second execution, no decline notice).
3. **Authenticated decisions.** The original had an open respond endpoint with a self-declared responder. Now Slack clicks need a valid, fresh HMAC signature over the raw body, API calls need a token, and `ALLOWED_APPROVERS` limits who counts. The self-approval check stops a requester approving their own request.
4. **Outage is not "already resolved".** The original treated any 4xx/5xx from the store as "already resolved", silently losing the approval. Here a store error is a `503 retry:true`.
5. **Prompt failure is visible.** If the Slack message cannot be delivered (3 tries), the reply says `prompt_delivered:false` and the requester is told. The request stays approvable through the API.
6. **The executor gets what it needs.** `action_type`, `details`, `amount`, the approver and an idempotency key `approval:{action_id}`. It is not retried by the workflow: it has side effects, so retries are the executor's decision (use the key).
7. **Definite failure vs unknown.** An HTTP error from the executor is recorded `failed`; no response at all is recorded `unknown` (it may have run), with the key to check by. Both alert ops and tell the requester.
8. **The outcome is written back** to the store (`executed` / `failed` / `unknown`), so the record shows what actually happened.
9. **Strict input checks.** Invalid ids, timeouts, amounts and details are rejected with 400 instead of coerced.
10. **Slack text is escaped** (`& < >`), so a request cannot inject `@channel` or links into the approvers' channel.
11. **Notices run after the reply**, so slow Slack or notify services never delay or change a decision.

## Edge cases handled

- **Double click / five simultaneous approvals.** One `approved`, four `already_resolved`, one execution, one notice.
- **Approve and deny at the same instant.** One winner; the executor runs only if approve won.
- **Approval arriving at the deadline.** Both outcomes are possible, but each request ends in exactly one consistent state: approved means executed once and no decline notice; declined means never executed and exactly one decline notice.
- **Duplicate request ids.** Two simultaneous identical requests: one pending, one `already_requested`, one Slack prompt, one auto-decline.
- **Answering after the deadline but before the timer fires.** `410 request_expired`; the store records it as auto-declined and the requester is told.
- **Requester approving themselves.** 403 (case-insensitive); they may still deny.
- **Replay of an old Slack click.** A correctly signed request older than 5 minutes is refused.

## Configuration (`CONFIG` node)

| Key | Default | Meaning |
|---|---|---|
| `APPROVAL_STORE_API_URL` | `http://localhost:4700` | Base URL of the approval store. |
| `ACTION_EXECUTOR_API_URL` | `http://localhost:4700` | Base URL of the service that performs approved actions. |
| `NOTIFY_API_URL` | `http://localhost:4700` | Base URL of the requester-notification service. Blank = no requester notices. |
| `SERVICE_HEADERS_JSON` | `{}` | Headers sent to the store, executor and notify service, e.g. `{"x-api-key":"..."}`. |
| `SLACK_WEBHOOK_URL` | blank | Slack-compatible incoming webhook for prompts and alerts. Blank = no Slack (decisions still work through the API). |
| `SLACK_SIGNING_SECRET` | blank | Verifies Slack button clicks. |
| `REQUEST_AUTH_TOKEN` | blank | If set, request callers send `x-request-token`. |
| `APPROVER_API_TOKEN` | blank | If set, API approvers send `x-approver-token`. |
| `ALLOWED_APPROVERS` | blank | Comma-separated approver names (blank = anyone who passes authentication). |
| `SELF_APPROVAL_ALLOWED` | `false` | `true` lets a requester approve their own request. |
| `DEFAULT_TIMEOUT_MINUTES` | 60 | Used when a request omits `timeout_minutes`. |
| `MIN_TIMEOUT_MINUTES` / `MAX_TIMEOUT_MINUTES` | 1 / 1440 | Accepted range. |
| `REQUEST_TIMEOUT_MS` | 10000 | Timeout for store, Slack and notify calls. |
| `EXECUTOR_TIMEOUT_MS` | 30000 | Timeout for the executor call. |
| `RECONCILE_ENABLED` | `false` | Switch for the reconcile job (schedule and endpoint). |
| `RECONCILE_AUTH_TOKEN` | blank | Required when enabled (8+ characters); API callers send it as `x-reconcile-token`. |
| `RECONCILE_EXPIRE` | `true` | Auto-decline requests still pending after their deadline. |
| `RECONCILE_EXECUTE` | `false` | `true` runs approved-but-never-run actions; `false` only reports them. |
| `RECONCILE_STALE_SECONDS` | 900 | How old an approved, unexecuted request must be before reconcile touches it. Must be larger than `EXECUTOR_TIMEOUT_MS` in seconds. |
| `RECONCILE_MAX_PER_RUN` | 20 | Most records handled per step per run (1 to 200). |

## Service contract

All JSON. `SERVICE_HEADERS_JSON` is sent on every store, executor and notify call. `{id}` is URL-encoded.

**Approval store**

| Call | Body | Response |
|---|---|---|
| `POST /approvals` | `{action_id, action_type, requested_by, summary, details, amount, timeout_minutes}` | `201 {record}` if new; `200 {existing:true, record}` if that `action_id` exists. **Must be atomic create-or-return-existing.** The record has `status` (`pending` at creation) and `expires_at`. |
| `GET /approvals/{id}` | | `200 {record}` or `404`. |
| `POST /approvals/{id}/resolve` | `{decision:"approve"\|"deny"\|"timeout", resolved_by}` | `200 {claimed:true, record}` if it was `pending` and is now decided; `409 {claimed:false, record}` if already decided; `409 {claimed:false, expired:true, record}` if a human decision arrives after `expires_at` (the store should mark it auto-declined); `404` if unknown. **Must be one atomic compare-and-set** (a conditional `UPDATE ... WHERE status='pending' RETURNING`, or Redis `SET NX`/Lua). `timeout` is always allowed after expiry. |
| `POST /approvals/{id}/execution` | `{status:"executed"\|"failed"\|"unknown", detail, via?}` | `200`. Records what happened. |
| `GET /approvals?status=&expired=&unexecuted=&older_than_seconds=&limit=` | | `200 {approvals:[record]}`, oldest first. Only used by reconcile; `expired=true` means `expires_at` has passed, `unexecuted=true` means no `execution`, `older_than_seconds` is measured from `resolved_at`. Records must carry `resolved_at`, `approvals` and `execution`. The workflow re-checks every record, so a store may be loose, never relied on. |
| `POST /approvals/{id}/execution-claim` | `{lease_seconds}` | `200 {claimed:true, record}` or `409 {claimed:false, record}`. **Must be atomic** and succeed only when the record is `approved`, has no `execution` and has no live lease (`UPDATE ... WHERE status='approved' AND execution IS NULL AND (lease_until IS NULL OR lease_until < now()) RETURNING`). Only used by reconcile. |

**Action executor**: `POST /execute` `{action_id, action_type, details, amount, approved_by, idempotency_key}`. `2xx` = done; an HTTP error = failed; no response = unknown. **De-duplicate on `idempotency_key`.**

**Requester notices**: `POST {NOTIFY_API_URL}/send` `{to, message}`. Failures are ignored on purpose.

**Slack**: prompts and alerts are `POST` `{text, blocks?}` to `SLACK_WEBHOOK_URL`. Buttons use `action_id` `approve_btn` / `deny_btn` and the `action_id` of the request as `value`.

## Failure behavior

| What breaks | What happens |
|---|---|
| Store down at request time | 503; nothing sent to approvers; ops alerted. |
| Store down at decision time | 503 `retry:true`; nothing executes; the same approval works after recovery. |
| Store down at the deadline | Ops alerted that the request may be stuck pending. |
| Slack prompt fails (3 tries) | `prompt_delivered:false`; requester told; still approvable by API. |
| Slack down for notices | No effect on decisions. |
| Executor returns an error | Approval stands; recorded `failed`; ops alert; requester told. |
| Executor does not answer | Recorded `unknown` (may have run); ops alert with the idempotency key; requester told; not retried. |
| Outcome cannot be recorded | Ops alert. |
| Notify service down / blank | Requester notices skipped; everything else unaffected. |
| n8n dies after the decision, before the action runs | The record stays `approved` with no `execution`. With reconcile on, it is reported (or run once, per `RECONCILE_EXECUTE`). |
| n8n is down at the deadline | The request stays pending past `expires_at`; no one can approve it (410). With reconcile on, it is auto-declined on the next run. |
| Invalid CONFIG | 500 `approval_misconfigured` naming every bad setting. |

## Verification

Tested end to end on self-hosted n8n 2.35.7 against in-memory reference implementations of the services above. The original workflow was run first as a baseline: on current n8n it returns an empty `200` (its `$env` reads are blocked), so nothing works out of the box.

- happy path: record, one Slack prompt (text escaped, buttons carry the id), approve, executor called once with action type, details, amount, approver and idempotency key, outcome recorded, requester told
- duplicate requests (simultaneous): one pending, one `already_requested`, one prompt, one auto-decline
- auto-decline at the deadline (requester and Slack told once), late approval refused
- 13 invalid request shapes and 4 invalid decisions rejected; none reach the store
- Slack button payloads (approve and deny); deny notices
- five simultaneous approvals; approve-vs-deny races; unknown action
- store outage at request and decision time (503, recovery works, never "already resolved")
- executor 500, 422, and no response (failed vs unknown, not retried)
- Slack prompt failing (3 tries, requester told, still approvable)
- self-approval blocked (case-insensitive), deny allowed, and allowed when configured
- approval after the deadline but before the timer: 410, nothing executes
- 14 approve-vs-timeout races fired at 2.3 to 3.3 s against a 3 s deadline: every one ends in a single consistent state
- secured configuration: request and approver tokens, `ALLOWED_APPROVERS`, Slack signatures (missing, wrong, wrong secret, 10 minutes old, tampered body, valid)
- a 72 s wait with n8n killed and restarted while waiting: requests stayed pending, could still be approved, and unanswered ones were auto-declined at the deadline
- broken CONFIG reported with every problem named; no Slack and no notify configured

Reconcile (the stuck-request job) was tested for token and switch checks, expiry of overdue requests, running approved-but-never-run actions once with the original idempotency key, de-duplication, failure handling, simultaneous runs, the per-run cap, store listing failures, report-only mode and the approval-count floor, against the mock store only (the schedule trigger itself was not waited on).

Not tested: a live Slack app (clicks were simulated with correctly signed payloads), a real approval store, a real executor, n8n 1.x, queue mode, waits longer than a few minutes.

## Known limits

- **Approved but not run if n8n dies at the wrong moment.** The decision is recorded and the reply sent before the action runs. If n8n is killed between those two steps, the record says `approved` with no `execution`. Switch on the reconcile job to find these; it needs your store to support the list and `execution-claim` calls (see the service contract). It runs them only with `RECONCILE_EXECUTE=true`, and only safely if your executor honours the idempotency key.
- **The timer lives in n8n.** Waits over about 65 seconds are saved to the database and survived a restart in testing, but if n8n is down at the deadline the auto-decline happens late. Because the store refuses human answers after `expires_at` (410), a late timer can never let an expired request be approved. The reconcile job (when on) auto-declines overdue requests, relying on the store's `expires_at`.
- **Reconcile does not retry failures.** A request whose execution is recorded `failed` or `unknown` is left to a person; so is anything the store will not list.
- **The executor is called once, never retried.** An unknown result needs a human or your executor's idempotency key to resolve.
- **Open by default.** With no tokens or signing secret the decision endpoint trusts anyone, and `responder` is self-declared. Set the secrets.
- **Authenticated API callers choose their own `responder` name.** The token proves the caller is trusted, not which person it is; `ALLOWED_APPROVERS` is only as strong as that trust. Slack clicks carry a verified identity.
- **One approval, one approver by default.** The `Approval Policy` node can require several different approvers and escalate a stalled request, but this README does not document those settings and the approval store you plug in must support them; there are no reminders.
- **Slack interactivity needs a Slack app**, not just an incoming webhook.
- **Notices are best effort.** A failed requester or Slack notice is not retried.
- n8n's own `422` for malformed JSON includes a stack trace in its body; put a gateway in front if that matters.
