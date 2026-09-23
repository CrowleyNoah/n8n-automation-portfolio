# n8n Automation Engineering Portfolio

12 independently designed and built n8n workflows, each demonstrating a
distinct production-grade automation pattern — not "happy path" demos, but
systems built for how integrations actually fail: race conditions, partial
outages, duplicate requests, malformed input, and the third-party API that
is slow, rate-limited, or just wrong.

Every workflow ships as a runnable `workflow.json` (importable directly into
n8n) plus a `README.md` covering the business problem, architecture,
edge cases handled, and notes for an interview walkthrough.

**Author:** Noah Crowley

---

## Design principles used throughout

A few patterns recur across most of these workflows on purpose — they're
listed once here instead of re-explained in every README:

- **State survives external calls.** Every HTTP call that would overwrite
  `$json` is immediately followed by a `Merge (combineByPosition)` node
  recombining the response with the state that existed before the call, so
  nothing upstream is silently lost.
- **Atomic, not check-then-act.** Idempotency guards, sequence numbering,
  rate-limit token consumption, and lock/probe claiming are all done as a
  single compare-and-set call against the data layer — never a separate
  "check if it exists" followed by a "create it," which is exactly the gap
  a race condition lives in.
- **`neverError` + `fullResponse` on HTTP nodes**, so a 4xx/5xx status is
  read and classified in-workflow instead of throwing and killing the
  execution.
- **Failure classes are kept distinct.** A connection failure (DNS, refused,
  timeout) is not the same as an HTTP error status, which is not the same
  as a client error (4xx, our fault) versus a server error (5xx, their
  fault) — each gets routed and handled differently rather than collapsed
  into one generic "it failed" branch.

## The workflows

| # | Workflow | Nodes | What it demonstrates |
|---|----------|:-:|---|
| 01 | [RAG Knowledge Assistant](./01-rag-knowledge-assistant) | 31 | Retrieval-grounded Q&A over scattered docs (wikis, PDFs, URLs) with source citation and chunking |
| 02 | [Agentic Support Triage](./02-agentic-support-triage) | 40 | LLM-driven ticket classification and routing with tool-calling and escalation guardrails |
| 03 | [Data Validation ETL Pipeline](./03-data-validation-etl) | 29 | Schema validation, quarantine/dead-letter handling, and partial-batch recovery for partner data feeds |
| 04 | [Secured Webhook Order Processor](./04-secured-webhook-order-processor) | 30 | HMAC signature verification, replay-attack windows, and idempotent order intake |
| 05 | [Human-in-the-Loop Approval Gate](./05-human-approval-gate) | 39 | Async approval workflows for high-risk actions with timeout, escalation, and audit logging |
| 06 | [Scheduled Report Pipeline](./06-scheduled-report-pipeline) | 53 | Multi-source data aggregation, rendering, and delivery on a schedule with per-source failure isolation |
| 07 | [Multi-Model LLM Router](./07-multimodel-llm-router) | 52 | Cost-aware routing across LLM providers with fallback chains and budget controls |
| 08 | [Voice Intake Pipeline](./08-voice-intake-pipeline) | 35 | Speech-to-text → intent handling → text-to-speech pipeline with graceful degradation |
| 09 | [CRM Lead Enrichment & Dedup](./09-crm-lead-enrichment-dedup) | 39 | Fuzzy-matching deduplication (hand-rolled Levenshtein) and enrichment for messy inbound lead data |
| 10 | [Self-Healing Execution Monitor](./10-self-healing-monitor) | 92 | Failure polling, storm detection, automated classify/retry/escalate, and recovery detection across a whole n8n instance |
| 11 | [Template-Driven Document Generator](./11-template-doc-generator) | 48 | Atomic document numbering under concurrency, a hand-rolled templating engine, and server-computed financial totals |
| 12 | [Rate-Limited Circuit Breaker Gateway](./12-rate-limited-circuit-breaker) | 46 | Three-state circuit breaker with atomic probe claiming, proactive token-bucket rate limiting, and stale-cache degradation |

## A few worth reading first

If you're skimming rather than reading all 12:

- **[10 — Self-Healing Execution Monitor](./10-self-healing-monitor)** is the
  largest and most systems-heavy: it watches an n8n instance's own
  executions, distinguishes a single flaky failure from a correlated outage
  "storm," and automatically retries or escalates based on incident
  history rather than execution id (which changes on every retry).
- **[12 — Rate-Limited Circuit Breaker Gateway](./12-rate-limited-circuit-breaker)**
  is the clearest single demonstration of the "atomic, not check-then-act"
  principle above — the probe-claiming step is the detail most hand-built
  circuit breakers get wrong.
- **[11 — Template-Driven Document Generator](./11-template-doc-generator)**
  treats a generated invoice/contract as a financial artifact: totals are
  never trusted from the caller, and a document is never reported as
  "success" until it's actually archived.

## Repo layout

```
n8n-portfolio/
├── 01-rag-knowledge-assistant/
│   ├── workflow.json      ← import directly into n8n
│   └── README.md          ← problem, architecture, edge cases, interview notes
├── 02-agentic-support-triage/
│   └── ...
...
└── 12-rate-limited-circuit-breaker/
    └── ...
```

Each `workflow.json` references its external services via environment
variables (listed at the bottom of that workflow's README) rather than
hardcoded URLs or credentials, so nothing here is tied to a specific
vendor or deployment.

## Contact

Reach out via the contact info on my resume/LinkedIn — happy to walk through
the design decisions on any of these in more depth.
