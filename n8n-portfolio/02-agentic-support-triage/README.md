# 02 — Agentic Support Triage

## Business problem
Tier-1 support tickets are repetitive (order status, refund eligibility,
KB lookups) but most no-code automations either can't reason about which
action to take, or take actions with no bound and no escape hatch when
they're wrong. This workflow runs a bounded ReAct (reason → act → observe)
loop that can call real tools, self-corrects on bad tool input, and
escalates to a human with full context the moment it's uncertain or stuck
— instead of guessing or looping forever.

## Architecture

`POST /webhook/support/triage` → `{ ticket_id, customer_email, subject, message, channel? }`

1. **Validate** input; `400` on missing/malformed fields.
2. **Idempotency check** — webhook retries from the caller are common in
   production; a duplicate `ticket_id` returns the previously computed
   result instead of re-running classification and tool calls.
3. **Classify** the ticket (category, urgency, confidence, sensitive flag)
   via LLM, with a fail-safe parser.
4. **Immediate escalation gate** — sensitive tickets (legal/abuse/fraud/
   privacy) or low-confidence classifications skip the agent entirely and
   go straight to a human.
5. **ReAct loop** (bounded at 5 iterations): build a prompt from the
   ticket + scratchpad → LLM reasoning step → parse the action → either
   call a tool (`order_lookup`, `kb_search`, `refund_check`), log an error
   observation (unknown tool / unparseable response), or return a final
   answer.
6. **Resolution** — final answer triggers a best-effort customer email and
   a `200` response with the full tool-call trail.
7. **Escalation** — (from the early gate or from exceeding max iterations)
   creates a helpdesk ticket, notifies Slack, and responds `200 escalated`.

## Edge cases handled
- **Webhook retries / duplicate delivery** — idempotency check before any
  work happens; replays return the cached result.
- **Unparseable or malformed LLM output** — both the classifier and the
  agent's step output go through a fail-safe JSON parser (strips code
  fences, extracts the first `{...}` block, falls back cleanly). A
  classification that can't be parsed is treated as *low-confidence +
  sensitive*, which forces escalation rather than silently mishandling
  the ticket.
- **State surviving external calls** — every HTTP call (classify, agent
  reasoning, each tool, helpdesk create, send-reply) is followed by a
  `Merge (combineByPosition)` node that recombines the API response with
  the pipeline state the call would otherwise discard. This is the same
  fix applied to workflow 01's embedding step, applied consistently
  everywhere a round-trip would otherwise drop `ticket_id`, `scratchpad`,
  or `iteration`.
- **Hallucinated tool names** — the agent's requested tool is validated
  against an allowlist; an invalid name is *not* silently ignored, it
  becomes an explicit observation ("tool X is not available...") the
  agent sees on its next turn.
- **Malformed tool input** (e.g. missing `order_id`) — deliberately *not*
  special-cased. The tool API's own error response becomes the next
  observation, letting the agent self-correct — this is the actual point
  of the ReAct loop rather than a gap.
- **Runaway loops** — hard-bounded at `max_iterations` (5). Exceeding it
  escalates with the full scratchpad attached, rather than looping
  forever or timing out silently.
- **Escalation delivery failure** — if the helpdesk API call fails, the
  Slack message explicitly flags it (`[helpdesk ticket creation FAILED]`)
  instead of the failure disappearing silently; Slack itself is
  best-effort and never blocks the customer-facing response.
- **Customer reply delivery** — best-effort; the webhook caller already
  gets the answer in the response body regardless, which is a documented
  design decision (the caller owns actual delivery), not an oversight.

## Environment variables expected
`LLM_API_URL`, `CHAT_MODEL`, `IDEMPOTENCY_API_URL`, `ORDER_API_URL`,
`KB_API_URL`, `REFUND_API_URL`, `HELPDESK_API_URL`, `SLACK_WEBHOOK_URL`,
`EMAIL_API_URL`
