# 05 — Human-in-the-Loop Approval Gate

## Business problem
Some automated actions (large refunds, high-value discounts, destructive
data operations) shouldn't execute without a human saying yes — but
"wait for a human" is exactly where naive automations break: duplicate
prompts, no timeout, double-clicked buttons, or a response that arrives
just as the system was about to give up and auto-decline. This workflow
requests approval over Slack, enforces a timeout, and uses one atomic
compare-and-set operation as the single source of truth so a race between
"human clicks approve" and "timeout fires" can never double-execute or
double-notify.

## Architecture
Two webhook entry points share one external approval store.

**A — `POST /webhook/approvals/request`** (fired by the system wanting a
human decision)
1. Validate, then check for an existing request for this `action_id`
   (courtesy dedup) — a retried request returns the existing record
   instead of prompting Slack twice.
2. **Atomically create** the approval record (`status: pending`) — this
   compare-and-set call, not the pre-check, is the real guard against two
   near-simultaneous requests both creating a duplicate prompt.
3. Send the Slack approval prompt and **respond 200 "pending"
   immediately** — this can take hours, so the caller isn't kept waiting.
4. In the same execution, a **Wait node** pauses for `timeout_minutes`,
   then checks the record's status. If still pending, it **atomically
   auto-declines** using the same compare-and-set endpoint the human path
   uses below.

**B — `POST /webhook/approvals/respond`** (Slack button callback or a
direct API call)
1. Normalize either shape into `{ action_id, decision, responder }`.
2. **Atomically resolve** the decision via the same compare-and-set
   endpoint.
3. **Ack fast** (Slack expects a response within ~3s), then continue in
   the background: execute the approved action, or notify the requester
   of a denial.
4. If the approved action's own execution fails, that's flagged
   separately from a denial — an approved-but-unexecuted action is a
   worse silent failure than a clean denial.

## Edge cases handled
- **The core race**: a human approving in the same instant the timeout
  fires. Both paths call the *same* atomic `/resolve` endpoint; whichever
  arrives first wins, the other gets a clean 409 and backs off — no
  double execution, no double notification.
- **Double-clicked Approve button** — the second click hits the same
  compare-and-set endpoint, finds the record already resolved, and
  returns "already resolved" instead of executing twice.
- **Late response after timeout** — same mechanism: the resolve call
  simply fails to claim an already-declined record.
- **Duplicate request creation** — the atomic create rejects a second
  `action_id`; the caller gets the existing record back instead of a
  second Slack prompt.
- **Slack accepting two different payload shapes** — the decision webhook
  normalizes both Slack's native interaction payload (`body.payload` as
  a JSON string) and a plain direct API call into one shape.
- **Fast ack on both entry points** — the request endpoint responds
  before Slack is even notified; the decision endpoint responds before
  the approved action executes. Both continue processing in the same n8n
  execution afterward (same pattern as workflow 04).
- **Approval succeeds, execution fails** — treated as a distinct, more
  urgent failure mode (pages ops) than a denial, since silently losing an
  approved action is worse than a clean "no."
- **Runaway timeout** — `timeout_minutes` is capped at 24h server-side so
  a bad or missing value can't leave a Wait node parked indefinitely.
- **State surviving external calls** — every HTTP call is followed by a
  `Merge (combineByPosition)`, consistent with workflows 01–04.

## Environment variables expected
`APPROVAL_STORE_API_URL`, `SLACK_WEBHOOK_URL`, `NOTIFY_API_URL`,
`ACTION_EXECUTOR_API_URL`
