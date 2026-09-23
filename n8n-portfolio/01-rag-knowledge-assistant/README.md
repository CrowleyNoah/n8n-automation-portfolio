# 01 — RAG Knowledge Assistant

## Business problem
Teams have documentation scattered across wikis, PDFs, and URLs. Support and
sales reps waste time searching for answers that already exist somewhere.
This workflow exposes two endpoints: one to ingest documents into a vector
store, and one to answer natural-language questions grounded in that store
(retrieval-augmented generation), returning cited sources instead of a
hallucinated answer.

## Architecture

**Ingestion — `POST /webhook/rag/ingest`**
`{ doc_id, text? , source_url?, metadata? }` →
Validate → (fetch URL if no raw text) → strip HTML if present → chunk
(1000 chars, 150 overlap) → batch (5 at a time) → embed each chunk →
upsert vector + payload into the vector store → respond with chunk count.

**Query — `POST /webhook/rag/query`**
`{ query, top_k? }` →
Validate → embed query → similarity search → build a context window capped
at 6,000 chars → generate an answer constrained to the retrieved context →
respond with the answer and its source chunks.

## Edge cases handled
- **Missing/empty required fields** — both endpoints validate and return
  `400` with a specific error list rather than failing deep in the pipeline.
- **Oversized input** — ingestion caps documents at 500k chars, queries at
  2,000 chars, to avoid unbounded memory/API cost.
- **Source URL fetch failure** (timeout, 404, 5xx) — caught via
  `neverError` + explicit status check, returns `502` instead of crashing
  the workflow.
- **HTML-sourced documents** — script/style stripped and tags removed
  before chunking, so markup doesn't pollute embeddings.
- **Empty document after fetch/strip** — returns `422` instead of silently
  ingesting zero-length chunks.
- **Embedding API failures per-chunk** — retried (4x, backoff) and on final
  failure logged rather than aborting the whole batch; one bad chunk
  doesn't fail the entire document.
- **Chunk metadata surviving the embed call** — the embedding API only
  returns a vector, not the original `doc_id`/`chunk_id`/`text`. A `Merge`
  node (`combineByPosition`) recombines each item's original fields with
  its embedding result immediately after the HTTP call, rather than
  reaching back with `$('Chunk Text').item` from inside the batch loop —
  that cross-node `pairedItem` lineage isn't reliable across HTTP Request
  nodes mid-loop and would silently write garbage payloads to the vector
  store.
- **Vector store write failures** — retried (3x, backoff).
- **No similarity matches** — returns a clear "not found" answer instead of
  forcing the LLM to hallucinate from empty context.
- **Context window overflow** — retrieved chunks are truncated to a fixed
  character budget before being sent to the LLM.
- **LLM generation failure** — returns `503` with the sources that *were*
  found, so the caller isn't left with nothing.
- **top_k out of range** — clamped to 1–20 server-side rather than trusting
  client input.

## Environment variables expected
`EMBEDDINGS_API_URL`, `EMBEDDING_MODEL`, `VECTOR_DB_URL`, `LLM_API_URL`,
`CHAT_MODEL`
