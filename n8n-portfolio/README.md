# n8n Automation Engineering Portfolio

12 independently designed and built n8n workflows, each demonstrating a
distinct production-grade automation pattern. Not "happy path" demos: these
are built for how integrations actually fail, with race conditions, partial
outages, duplicate requests, malformed input, and the third-party API that
is slow, rate-limited, or just wrong.

Every workflow is a ready-to-import n8n workflow file plus a README that
covers the business problem, the exact request and response each backing
service must answer, every setting, what happens when each part fails, and
what was and was not tested.

**Author:** Noah Crowley

---

## How these were checked

Each workflow was run end to end on a real n8n instance (version 2.35.7),
not just read or imported. The external services it calls (databases, CRMs,
mail, speech, language models, Slack) were replaced by small test stand-ins
that can be told to fail on demand, and each workflow was exercised across
many configurations with well over 2,000 automated checks in total: normal
traffic, duplicates, simultaneous requests, malformed input, outages, slow
responses, and restarts in the middle of a run.

Each README ends with a plain "Not tested" list. In short: no live
third-party accounts, no sustained load testing, and no n8n 1.x or queue
mode. The stand-ins follow the contracts written in each README, so any real
backend that honours the same contract should drop in by changing a URL.

## Design principles used throughout

A few patterns recur across most of these workflows on purpose. They are
listed once here instead of re-explained in every README:

- **Nothing hardcoded.** Every service URL, secret, limit and message lives
  in one `CONFIG` node at the start of the workflow. There are no environment
  variables to set and no credentials inside the file, so nothing is tied to
  a specific vendor or deployment.
- **Atomic, not check-then-act.** Idempotency guards, document numbering,
  budget reservation, approvals and circuit-breaker probes are each done as a
  single claim against the data layer, never a separate "check if it exists"
  followed by a "create it". That gap is exactly where a race condition lives.
- **Failure classes are kept distinct.** A connection failure is not an HTTP
  error, and a client error (4xx, the caller's fault) is not a server error
  (5xx, the dependency's fault). Each is routed differently instead of being
  collapsed into one generic "it failed" branch.
- **Honest answers.** A lookup that failed is never reported as "not found",
  a write that failed is never reported as success, and an outage is never
  disguised as "no results". Where a workflow can only be sure of part of a
  result, it says which part.
- **HTTP nodes read the status instead of throwing.** Calls use
  `neverError` and `fullResponse`, so a 4xx or 5xx is classified inside the
  workflow instead of killing the execution.
- **One safe place to customise.** Each workflow has a small data-only rules
  node (a playbook, policy, layout or rules node, named in its README) that
  holds what is specific to your business. The checks that keep it safe sit
  in separate nodes marked as the engine, so changing wording or adding a
  rule never means touching them.

## The workflows

| # | Workflow | Nodes | What it demonstrates |
|---|----------|:-:|---|
| 01 | [RAG Knowledge Assistant](./01-rag-knowledge-assistant) | 32 | Retrieval-grounded Q&A over documents with source citation, safe URL ingest, atomic re-ingest, delete, and optional per-tenant access |
| 02 | [Agentic Support Triage](./02-agentic-support-triage) | 45 | A bounded AI agent with tools: ticket classification, own-orders-only access, refund guard, human escalation, optional async jobs |
| 03 | [Data Validation ETL Pipeline](./03-data-validation-etl) | 44 | Schema-driven validation, dead-letter handling and replay, circuit-breaker loading, parallel loading, optional async jobs |
| 04 | [Secured Webhook Order Processor](./04-secured-webhook-order-processor) | 75 | HMAC verification (generic, Stripe, GitHub, Shopify presets), replay windows, exactly-once intake, reconciliation of abandoned orders |
| 05 | [Human-in-the-Loop Approval Gate](./05-human-approval-gate) | 61 | Approvals for risky actions: Slack signature checks, multi-approver quorum, escalation, timeouts, reconcile job |
| 06 | [Scheduled Report Pipeline](./06-scheduled-report-pipeline) | 61 | Multi-source reports to PDF and email on a schedule, per-source failure isolation, resumable runs, configurable layouts, optional async jobs |
| 07 | [Multi-Model LLM Router](./07-multimodel-llm-router) | 45 | Cost-aware routing across model tiers with circuit breakers, atomic budget reservation, caching and retry back-off |
| 08 | [Voice Intake Pipeline](./08-voice-intake-pipeline) | 54 | Speech-to-text, safety checks, agent reply and text-to-speech, with graceful handling of silence, low confidence and failures |
| 09 | [CRM Lead Enrichment & Dedup](./09-crm-lead-enrichment-dedup) | 37 | Scored fuzzy matching (Levenshtein, nicknames, company suffixes), merge vs flag vs create, never overwriting CRM data |
| 10 | [Self-Healing Execution Monitor](./10-self-healing-monitor) | 93 | Failure polling, incident grouping, storm detection, classify/retry/escalate, and recovery detection across a whole n8n instance |
| 11 | [Template-Driven Document Generator](./11-template-doc-generator) | 48 | Gap-free document numbering under concurrency, safe templating, server-computed totals, and no "success" before the archive exists |
| 12 | [Rate-Limited Circuit Breaker Gateway](./12-rate-limited-circuit-breaker) | 51 | Three-state circuit breaker with atomic probe claiming, proactive token-bucket rate limiting, and stale-cache degradation |

## A few worth reading first

If you're skimming rather than reading all 12:

- **[10 — Self-Healing Execution Monitor](./10-self-healing-monitor)**
  is the largest and most systems-heavy: it watches an n8n instance's own
  executions, distinguishes a single flaky failure from a correlated outage
  "storm", and retries or escalates based on incident history rather than
  execution id (which changes on every retry).
- **[12 — Rate-Limited Circuit Breaker Gateway](./12-rate-limited-circuit-breaker)**
  is the clearest single demonstration of the "atomic, not check-then-act"
  principle above. The probe-claiming step is the detail most hand-built
  circuit breakers get wrong.
- **[04 — Secured Webhook Order Processor](./04-secured-webhook-order-processor)**
  shows the full lifecycle of a trusted-but-unreliable webhook: verify the
  exact signed bytes, accept fast, process exactly once, and recover orders
  whose processing died after the sender was already told "200".
- **[11 — Template-Driven Document Generator](./11-template-doc-generator)**
  treats a generated invoice or contract as a financial artifact: totals are
  never trusted from the caller, and a document is never reported as
  "success" until it is actually archived.

## Repo layout

```
n8n-portfolio/
├── README.md                                         ← this file
├── 01-rag-knowledge-assistant/
│   ├── 01-rag-knowledge-assistant-workflow.json      ← import into n8n
│   └── 01-rag-knowledge-assistant-README-public.md   ← problem, contracts, settings, failures, tests
├── 02-agentic-support-triage/
│   └── ...
...
└── 12-rate-limited-circuit-breaker/
    └── ...
```

To try one: import its `-workflow.json` into n8n, open the `CONFIG` node,
point the service URLs at your own services (the README lists the exact
request and response each one must answer), and activate it.

## Contact

Reach out via the contact info on my resume/LinkedIn. I'm happy to walk
through the design decisions on any of these in more depth.
