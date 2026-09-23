# 04 — Secured Webhook Order Processor

## Business problem
Order/payment webhooks are a favorite attack surface (spoofed requests,
replayed captured requests) and a favorite source of duplicate-processing
bugs (senders retry on timeout, and retries look identical to new events).
This workflow verifies every request cryptographically, rejects replays,
guarantees each order is processed exactly once, and acks fast enough
that the sender never times out and retries unnecessarily — a real
production requirement most webhook handlers get wrong by doing all their
work before responding.

## Architecture

`POST /webhook/orders/incoming` (HMAC-signed)

**Phase A — verify, dedupe, durably log, ack fast:**
1. **Verify signature + timestamp** — HMAC-SHA256 over `timestamp.rawBody`
   using a constant-time comparison; a timestamp more than 5 minutes old
   is rejected as a possible replay even with a technically-valid
   signature.
2. **Parse & shape-check** the payload defensively.
3. **Idempotency check** by `order_id` — a duplicate delivery returns the
   original cached result instead of reprocessing.
4. **Durably persist** the raw event *before* responding — this is what
   makes it safe to ack immediately: if the process dies right after
   responding, the event already survived to disk/DB and can be
   reconciled.
5. **Respond 200 "accepted"** — the sender's webhook call returns here.

**Phase B — continues in the same execution, after the response is sent:**
6. **Reserve inventory** → if insufficient, mark `backordered` and notify
   the customer — not a failure, a normal business outcome.
7. **Create the order** in the OMS → if it fails after retries, mark
   `pending_manual` and page ops for reconciliation, rather than losing
   the order or retrying forever.
8. **Send confirmation** (best-effort) on success.
9. **Update the durable event's final status** either way — this is the
   source of truth a support agent or reconciliation job checks later.

## Edge cases handled
- **Signature verification against the wrong bytes** — the webhook is
  configured with `rawBody: true` specifically so HMAC is checked against
  the exact bytes the sender signed, not a re-serialized
  `JSON.stringify` of the parsed body (a common cause of "valid webhooks
  failing signature checks" in the wild).
- **Timing-attack-safe comparison** — `crypto.timingSafeEqual`, wrapped in
  try/catch since it throws on length mismatch rather than returning
  `false`.
- **Replay attacks** — a signed-but-stale request (>5 min old) is
  rejected even though the signature itself is valid.
- **Duplicate delivery** — idempotency check prevents double inventory
  reservation / double order creation on sender retries.
- **Fast ack, slow work** — the response is sent as soon as the event is
  durably logged; fulfillment continues in the background of the *same*
  n8n execution rather than making the sender wait through inventory,
  OMS, and notification calls (which is exactly the kind of latency that
  causes senders to time out and retry, defeating idempotency in the
  first place if it weren't handled).
- **State surviving external calls** — every HTTP call is followed by a
  `Merge (combineByPosition)` recombining the response with pipeline
  state, consistent with workflows 01–03.
- **Out-of-stock is a business outcome, not an error** — routed to
  `backordered` with a customer notice, not treated as a failure.
- **Downstream failure after ack** — since the sender already got `200`,
  an OMS failure can't be signaled back to them; it's marked
  `pending_manual` and paged to ops instead, with the reason preserved.
- **A failed durable-persist is the one case we don't ack** — if we can't
  even log the event after retries, we deliberately return `500` so the
  sender's own retry gives us another shot, rather than pretending we
  handled something we didn't actually save.
- **Fire-and-forget side effects don't corrupt state** — confirmation
  emails, backorder notices, and ops pings all run in parallel with (not
  chained before) the node that finalizes the order's status.

## Environment variables expected
`WEBHOOK_SIGNING_SECRET`, `SECURITY_LOG_API_URL`, `IDEMPOTENCY_API_URL`,
`EVENTS_STORE_API_URL`, `INVENTORY_API_URL`, `OMS_API_URL`,
`NOTIFY_API_URL`, `SLACK_WEBHOOK_URL`
