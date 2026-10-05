# 01 — RAG Knowledge Assistant

A single n8n workflow with two endpoints. **Ingest** takes a document (raw text, or a URL to fetch), cleans it, cuts it into overlapping chunks, embeds each chunk and stores it in a vector database. **Query** takes a natural-language question, finds the most similar chunks, and has a language model answer from those chunks only, returning the answer together with the sources it was based on.

The hard part is not the happy path. It is what happens around it: a URL ingest must not become a way to reach your internal network, a re-ingested document must replace its old version instead of leaving stale chunks behind, an embedding or database outage must be reported as an outage (not as "no results" or "ingested"), and retrieved text must reach the model as data, not as instructions.

## Design principle

Nothing is hardcoded. The embedding service, vector store, language model, collection name, chunking, limits, tokens and the hosts that may be fetched all live in the `CONFIG` node. The vector store speaks the Qdrant HTTP API; the embedding service and model are any OpenAI-style / Ollama-style / plain HTTP endpoints (the response shape is detected). Any backend that honours the contracts below works.

## Ingest endpoint

`POST /webhook/rag/ingest`

```json
{ "doc_id": "handbook", "text": "…", "source_url": "https://…", "metadata": {"team": "hr"} }
```

| Field | Required | Meaning |
|---|---|---|
| `doc_id` | yes | Your id for the document, 1–128 characters: letters, digits and `. _ : -`. Ingesting the same id again **replaces** the document. |
| `text` | one of | The document text (max `MAX_DOC_CHARS`, default 500,000). Used when given, even if `source_url` is also present. |
| `source_url` | one of | An `http(s)` page to fetch instead. HTML, plain text, Markdown, JSON and XML only. Subject to the host rules above. |
| `metadata` | no | A JSON object (max 10,000 characters of JSON), stored with every chunk. |

If `CONFIG.INGEST_AUTH_TOKEN` is set, send it as header `x-ingest-token`. The reply comes after the whole document is stored, so the caller must allow for the embedding time (a long document can take many seconds; the limit is `INGEST_MAX_SECONDS`, default 120).

| Status | Body | When |
|---|---|---|
| 200 | `{status:"ingested", doc_id, chunks, characters, dimensions}` | Every chunk is embedded and stored, and leftovers of an older version are removed. |
| 400 | `{error:"validation_failed", details:[...]}` | Bad input, or a `source_url` that is not allowed. Nothing was fetched or stored. |
| 401 | `{error:"unauthorized"}` | Missing or wrong `x-ingest-token` (with `TENANTS_JSON`: missing or wrong `x-tenant-token`). |
| 403 | `{error:"forbidden"}` | Only with `TENANTS_JSON`: the tenant's `can` list does not include `ingest`. |
| 413 | `{error:"document_too_large"}` | The fetched page (or its chunk count) is over the limit. |
| 422 | `{error:"unsupported_content_type"}` or `{error:"empty_or_unreadable_document"}` | A PDF/image/binary page, or a page with no readable text. |
| 500 | `{error:"rag_misconfigured", details:[...]}` | Invalid CONFIG (every problem listed). |
| 502 | `{error:"source_fetch_failed", status?, reason?}` or `{error:"source_redirect_not_allowed"}` | The page could not be fetched (HTTP error, timeout, redirect loop, redirect to a disallowed host). |
| 502 | `{error:"ingest_incomplete", stage, reason, chunks_total, chunks_stored, chunks_failed, failed_chunks:[...], retry:true}` | Something failed while embedding (`stage:"embedding"`), storing (`"vector_store"`), switching an update on (`"commit"`, only with `ATOMIC_UPDATES`) or cleaning up (`"cleanup"`). **Send the same document again**: it overwrites cleanly. Ops are notified in Slack. |

Malformed JSON never reaches the workflow: n8n itself rejects it with a `4xx`.

## Query endpoint

`POST /webhook/rag/query`

```json
{ "query": "How many vacation days do employees get?", "top_k": 5, "filters": { "dept": "hr" } }
```

`query` is required (max `MAX_QUERY_CHARS`, default 2000). `top_k` is optional; it is rounded down and clamped to 1–`TOP_K_MAX` (default 5 if omitted). `filters` is optional: `{key: value}` pairs (string, number or true/false) matched against the `metadata` stored at ingest. Only keys listed in `filterable_metadata` in the RAG Playbook are accepted (none by default; at most 5 filters per query); anything else is a 400, so callers cannot probe other metadata. If `CONFIG.QUERY_AUTH_TOKEN` is set, send it as header `x-query-token`.

| Status | Body | When |
|---|---|---|
| 200 | `{answer, grounded:true, cited:[doc_id…], sources:[{doc_id, chunk_index, score, source_url, snippet}]}` | Answered from the retrieved text. `cited` lists which retrieved documents the answer actually names. |
| 200 | `{answer:"No relevant documents were found…" (your `no_match_message`), grounded:false, sources:[]}` | Nothing relevant (nothing found, or everything under `MIN_SCORE`). The model is not called. |
| 400 | `{error:"validation_failed", details}` | Bad input. |
| 401 | `{error:"unauthorized"}` | Missing or wrong `x-query-token` (with `TENANTS_JSON`: `x-tenant-token`). |
| 403 | `{error:"forbidden"}` | Only with `TENANTS_JSON`: the tenant's `can` list does not include `query`. |
| 500 | `{error:"rag_misconfigured", details}` | Invalid CONFIG or RAG Playbook (every problem is listed). |
| 503 | `{error:"embedding_unavailable", retry}` | The question could not be embedded. |
| 503 | `{error:"search_unavailable", retry:true}` | The vector store could not be searched (an outage, or the collection does not exist). **Not** reported as "no results". |
| 503 | `{error:"llm_generation_failed", sources, retry:true}` | Relevant text was found but the model failed or returned nothing; the sources are included. |

## Delete endpoint

`POST /webhook/rag/delete` with `{"doc_id": "handbook"}` removes every stored chunk of that document. **It is off by default**: set `DELETE_ENABLED=true` and a `DELETE_AUTH_TOKEN` (8+ characters; callers send it as header `x-delete-token`, a separate secret from the ingest token).

| Status | Body | Meaning |
|---|---|---|
| 200 | `{status:"deleted", doc_id, chunks_deleted}` | Every chunk of that document is gone (the workflow counted again after deleting and found zero). |
| 400 | `{error:"validation_failed", details}` | `doc_id` missing or not 1-128 characters of letters, digits and `. _ : -`. |
| 401 | `{error:"unauthorized"}` | Missing or wrong `x-delete-token` (with `TENANTS_JSON`: `x-tenant-token`, and the tenant's `can` must include `delete`, otherwise `403 forbidden`; the `DELETE_AUTH_TOKEN` is then not used). |
| 403 | `{error:"delete_disabled"}` | `DELETE_ENABLED` is not true. |
| 404 | `{error:"doc_not_found", doc_id}` | The document has no chunks (never ingested, already deleted, or a typo). Nothing was deleted. |
| 500 | `{error:"delete_misconfigured"}` or `rag_misconfigured` | Enabled with no/short token, or another bad CONFIG value. |
| 502 | `{error:"delete_incomplete", stage, reason, retry:true, chunks_found?, chunks_remaining?}` | The vector store failed while counting (`count`), deleting (`delete`), or the chunks were still there afterwards (`verify`). It is safe to repeat the call. |

How it stays safe: the delete filter is an **exact** match on `doc_id` (deleting `abc` never touches `abc-2`); the workflow counts first and last, and only says `deleted` when zero chunks remain, so a store that answers "ok" without deleting is caught; a delete skips the RAG Playbook entirely, so a mistake in your playbook cannot stop or change it. Ops get one Slack line for every delete (an audit trail) and one for every failure. A document that is deleted can be ingested again at any time.

## Per-tenant access (optional)

Set `TENANTS_JSON` and the workflow stops using the three single tokens. Every caller must send a **tenant token** in the header `x-tenant-token`, and a token only ever sees, changes or deletes **its own** documents. Think of one apartment building with one key per flat: the same front door (one workflow, one collection), but your key opens only your flat.

```json
{
  "acme":   { "token": "acme-token-at-least-16-chars", "can": ["ingest", "query", "delete"] },
  "globex": { "token": "globex-token-at-least-16-chars" },
  "widget": { "token": "widget-token-at-least-16-chars", "can": ["query"] }
}
```

- A tenant id is 1-64 characters (letters, digits, `.` `_` `-`). A token is at least 16 characters and unique. `can` is optional (default `["ingest","query"]`); `delete` also needs `DELETE_ENABLED=true`.
- **The tenant comes only from the token.** A `tenant` field in the body, in `metadata` or in `filters` is ignored or refused.
- Two tenants can use the same `doc_id` ("handbook") without clashing: each tenant's documents get their own point ids, and re-ingesting, cleaning up leftovers, atomic updates and deletes are all scoped to the caller's tenant.
- Every search carries the caller's tenant filter **and** the workflow re-checks each returned chunk, so a vector store that ignores a filter still cannot leak another tenant's text.
- A tenant deleting a `doc_id` that only another tenant has gets `404 doc_not_found`, the same as a document that does not exist. It learns nothing.
- Wrong or missing token: `401`. A valid token without the right `can`: `403 forbidden`. A broken `TENANTS_JSON`: `500 rag_misconfigured` (the problems are listed, token values never are).
- Ops notices are tagged `[tenant acme]`.
- Atomic updates (`ATOMIC_UPDATES`) work per tenant. Their version ids are already unique, so the tenant filter on the hidden-chunk switch is an extra safeguard rather than the thing that keeps tenants apart.
- **Plan the switch.** Documents stored before you turned this on have no tenant, so no tenant can see or delete them (they are not removed). Turning it off again would make tenant documents visible to everyone. Use a fresh collection (change `VECTOR_COLLECTION`) when you switch either way.

## Architecture

```
INGEST   POST /rag/ingest
  → auth (optional token, or the tenant token when TENANTS_JSON is set) + CONFIG + validation (doc_id, text|source_url, metadata)      (400 / 401 / 500)
  → source_url: host rules (allow-list, or no localhost / private / metadata addresses)
  → fetch: ≤3 redirects, each hop re-checked · text types only · size limit               (400 / 413 / 422 / 502)
  → clean (HTML → text, entities decoded) → normalise → chunk at paragraph / sentence / word boundaries   (422 if empty)
  → embed every chunk (4 at a time, 1 retry each, time budget, early stop if the service is down)
  → store in batches of 50 (deterministic UUIDs, 1 retry)
  → ONLY if everything was stored: delete chunks beyond the new count (older, longer version)
  → reply 200, or 502 ingest_incomplete → THEN Slack notice

ATOMIC UPDATES (only when ATOMIC_UPDATES=true) replace the two "store" lines above:
  → store every chunk as a NEW version, hidden (staged=true, version-specific ids) · the old version is not touched
  → COMMIT: one set-payload call switches the new version on, then an exact count must equal the chunk count
  → failed or short: ROLL BACK (delete only this version); the old version is still whole and still served
  → committed: delete older versions only (no version_ts, or a smaller one); a newer one is never touched
  → searches hide staged chunks and show only the newest version of each document

DELETE   POST /rag/delete                                   (off unless DELETE_ENABLED)
  → switch (403) → token set + long enough (500) → token matches (401) → CONFIG (500) → doc_id (400)
  → count chunks (404 if none) → delete by exact doc_id filter → count again
  → reply 200 deleted only when zero remain, else 502 delete_incomplete → THEN Slack line

QUERY    POST /rag/query
  → auth (or tenant token) + CONFIG + Playbook + validation (query, top_k, filters)                                            (400 / 401 / 500)
  → embed the question (1 retry)          failure → 503 embedding_unavailable
  → vector search (1 retry)               failure → 503 search_unavailable
  → drop weak / malformed hits, sort, de-duplicate, pack into MAX_CONTEXT_CHARS, wrap each chunk in <document> tags
  → nothing left → 200 "no relevant documents" (no model call)
  → model (1 retry), answer from the documents only     failure → 503 with the sources
  → reply: answer, grounded, cited, sources
```

1. **Failures are reported, not swallowed.** The original called its embedding, vector-store and model services with `neverError` and `retryOnFail`; with `neverError` nothing counts as an error, so the promised retries never ran, a failed upsert was ignored, embedding failures were "logged" by a node that only built an item and looped on, so nothing was ever reported, and the reply said "ingested" with the planned chunk count. Query-side outages were reported as "No relevant documents were found". Here every call is checked, retried once when the failure is transient, and a failure is a `502`/`503` that says what happened.
2. **Re-ingesting replaces.** Point ids are UUIDs derived from `doc_id` and the chunk index, so the same document always lands on the same points. After a complete store, chunks beyond the new count are deleted. The original used `doc_id::chunk_N` as the point id (which Qdrant rejects) and never removed leftovers, so shrinking a document left stale chunks answering questions forever. With `ATOMIC_UPDATES=true` an update is instead staged hidden, switched on in one step, verified, and only then is the old version removed.
3. **Cleanup only after success.** If any chunk failed, nothing is deleted, so a failed update cannot also wipe the previous version. Retrying the same document finishes the job.
4. **URL ingest is guarded.** The original fetched any URL the caller supplied (cloud metadata, internal admin pages) and followed redirects blindly. Now: `ALLOWED_SOURCE_HOSTS` (when set, only those hosts), otherwise localhost, `.local`/`.internal`/single-label hosts, private, loopback, link-local, carrier-grade and multicast IPv4 ranges, IPv6 literals, odd numeric forms (`2130706433`, `0x7f000001`) and `user:pass@` URLs are refused; redirects are followed by hand (max 3) with every hop re-checked.
5. **Only readable content.** A PDF or image used to be embedded as garbage; now it is a `422`. A page with no text is a `422` (the original had a 422 node that nothing was connected to, so an empty document went on to be "embedded").
6. **Cleaner chunks.** Scripts, styles, comments and tags are removed from HTML, entities decoded, whitespace normalised; chunks end at paragraph, then sentence, then word boundaries (not mid-word), overlap their neighbour, are never cut inside an emoji, and every character of the text lands in some chunk.
7. **Vectors are checked.** The embedding response may be `{embedding}`, `{embeddings:[[…]]}` or `{data:[{embedding}]}`; anything that is not a non-empty list of finite numbers, or has a different length from the rest (or from `EXPECTED_DIMENSIONS`), is a failure for that chunk.
8. **The model gets data, not orders.** Each chunk is wrapped in `<document doc_id=… chunk=…>` tags (a document that contains a closing tag is neutralised), and the system prompt says the documents and the question are data and must never be obeyed. This reduces prompt-injection risk; it does not remove it.
9. **Relevance is enforceable.** `MIN_SCORE` drops weak matches before the model sees them. Results are re-sorted, de-duplicated and packed into a fixed context size, and the sources list matches exactly what was sent.
10. **Alerts after the reply.** An incomplete ingest posts one Slack notice (text escaped), never delaying the reply. Slack is optional.

## Edge cases handled

- **Same document ingested three times at once.** Deterministic ids mean one clean copy, no duplicates.
- **Many different documents at once.** All complete; the store holds exactly their chunks.
- **One chunk that cannot be embedded.** The rest are stored, the reply names how many failed and which, nothing is deleted, and a retry completes it.
- **A transient error.** One retry for embedding, vector store and model calls.
- **A service that is plainly down.** Ingest stops early (after the first few failures) instead of making hundreds of doomed calls.
- **A slow service.** Per-call timeouts and an overall ingest time budget.
- **Text with no spaces, emoji, CRLF line endings, control characters.** All handled.
- **A question with nothing relevant.** A plain "no relevant documents", and the model is not called.

## Configuration (`CONFIG` node)

| Key | Default | Meaning |
|---|---|---|
| `EMBEDDINGS_API_URL` | `http://localhost:5200` | Base URL of the embedding service. |
| `EMBEDDINGS_PATH` | `/embed` | Path appended to it (`/v1/embeddings`, `/api/embed`, …). The request is `{input, model}`. |
| `EMBEDDING_MODEL` / `EMBEDDING_HEADERS_JSON` | `nomic-embed-text` / `{}` | Model name; extra headers such as `{"authorization":"Bearer …"}`. |
| `VECTOR_DB_URL` / `VECTOR_COLLECTION` / `VECTOR_HEADERS_JSON` | `http://localhost:5200` / `knowledge_base` / `{}` | Qdrant-style vector store, the collection, and headers such as `{"api-key":"…"}`. |
| `LLM_API_URL` / `CHAT_MODEL` / `LLM_HEADERS_JSON` | `http://localhost:5200` / `local-model` / `{}` | OpenAI-compatible chat endpoint (`/v1/chat/completions` is appended). |
| `SLACK_WEBHOOK_URL` | blank | Slack-compatible webhook for ingest-failure notices. Blank = none. |
| `INGEST_AUTH_TOKEN` / `QUERY_AUTH_TOKEN` | blank | If set, callers send `x-ingest-token` / `x-query-token`. |
| `ATOMIC_UPDATES` | `false` | `true` = all-or-nothing document updates (see Known limits and Failure behavior). Needs a vector store that supports set-payload by filter, `must_not` and `is_empty` (Qdrant does). Off = the original in-place overwrite. |
| `TENANTS_JSON` | blank | Per-tenant access (see "Per-tenant access"). A JSON object `{"acme": {"token": "…16+ chars…", "can": ["ingest","query","delete"]}}`. When set, callers send `x-tenant-token` and the three single tokens are not used. Blank = off. |
| `DELETE_ENABLED` / `DELETE_AUTH_TOKEN` | `false` / blank | Turns `POST /webhook/rag/delete` on. Required token (8+ characters) is sent as header `x-delete-token`. Off = every delete is refused with 403. |
| `ALLOWED_SOURCE_HOSTS` | blank | Comma-separated hostnames `source_url` may use. Blank = any public host. |
| `CHUNK_SIZE` / `CHUNK_OVERLAP` | 1000 / 150 | Characters per chunk (200–8000) and overlap (0 to half the size). |
| `MAX_DOC_CHARS` / `MAX_QUERY_CHARS` | 500000 / 2000 | Input limits. |
| `TOP_K_DEFAULT` / `TOP_K_MAX` | 5 / 20 | Sources per question when `top_k` is omitted / the cap. |
| `MIN_SCORE` | 0 | Drop matches scoring below this (0 = off). Scores depend on your embedding model and distance metric, so tune it on real questions. |
| `MAX_CONTEXT_CHARS` | 6000 | Text sent to the model per question. |
| `EXPECTED_DIMENSIONS` | 0 | Required vector length (0 = accept any, but all chunks of a document must agree). |
| `EMBED_CONCURRENCY` | 4 | Chunks embedded at once. |
| `INGEST_MAX_SECONDS` | 120 | Time budget for embedding one document. |
| `FETCH_TIMEOUT_MS` / `EMBED_TIMEOUT_MS` / `VECTOR_TIMEOUT_MS` / `LLM_TIMEOUT_MS` / `REQUEST_TIMEOUT_MS` | 15000 / 20000 / 15000 / 30000 / 10000 | Per-call timeouts (page fetch / embedding / vector store / model / Slack). |

## Service contract

All JSON. `{c}` is the URL-encoded `VECTOR_COLLECTION`.

| Service | Call | Response |
|---|---|---|
| Embeddings | `POST {EMBEDDINGS_API_URL}{EMBEDDINGS_PATH} {input, model}` | `2xx` with `{embedding:[…]}`, `{embeddings:[[…]]}` or `{data:[{embedding:[…]}]}`. |
| Vector store | `PUT /collections/{c}/points?wait=true {points:[{id, vector, payload}]}` | `2xx`. `id` is a UUID; `payload` has `doc_id, chunk_id, chunk_index, total_chunks, text, source_url, metadata` (plus `tenant` with `TENANTS_JSON`). |
| | `POST /collections/{c}/points/search {vector, limit, with_payload:true, score_threshold?}` | `2xx {result:[{id, score, payload}]}`, higher score = more similar. `404` (no collection) is treated as unavailable. |
| | `POST /collections/{c}/points/delete?wait=true {filter:{must:[{key:"doc_id", match:{value}}, {key:"chunk_index", range:{gte}}]}}` | `2xx`. |
| | `POST /collections/{c}/points/payload?wait=true {payload:{staged:false}, filter:{must:[doc_id, version]}}` | `2xx`. **Only with `ATOMIC_UPDATES`**: set a payload field on every point matching the filter. A store that acknowledges but ignores it is caught by the count that follows. Atomic updates also use `{must_not:[{key:"staged", match:{value:true}}]}` in searches and `{is_empty:{key:"version_ts"}}` / `range:{lt}` in deletes. |
| | `POST /collections/{c}/points/count {filter:{must:[{key:"doc_id", match:{value}}]}, exact:true}` | `2xx {result:{count}}`. Used by the delete endpoint (before and after) and, with `ATOMIC_UPDATES`, to check that the whole new version is live. A `404` means the collection does not exist, so the document is reported as not found. |
| Model | `POST {LLM_API_URL}/v1/chat/completions {model, temperature, messages}` | `choices[0].message.content`. |
| Slack | `POST {text}` | Anything; failures are ignored on purpose. |
| Pages | `GET source_url` | `2xx` text, HTML, Markdown, JSON or XML. |

These are the real Qdrant request shapes. The workflow was run against a real Qdrant 1.19.1 (ingest, query, atomic update, delete, tenants, missing collection). Any other vector database needs a thin adapter exposing this API.

## Failure behavior

| What breaks | What happens |
|---|---|
| Embedding service down (ingest) | One retry per chunk, early stop, `502 ingest_incomplete` (stage `embedding`), nothing deleted, Slack notice. |
| One chunk cannot be embedded | The others are stored; `502` naming the failed chunks; nothing deleted; retry completes. |
| Vector store fails during a delete | One retry, then `502 delete_incomplete` with the stage (`count`, `delete` or `verify`) and `retry:true`; a store that acknowledges but leaves chunks behind is `stage: verify`, never "deleted". Slack notice. |
| Vector store write fails | One retry, then `502` (stage `vector_store`) with how many chunks were stored. |
| Cleanup fails after a full store | `502` (stage `cleanup`); the document is stored; retry finishes the cleanup. |
| Update fails part-way (`ATOMIC_UPDATES`) | `502`, the new chunks are removed again, the previous version stays whole and is still served (`previous_version_kept:true`). |
| Store ignores or rejects the "switch on" step (`ATOMIC_UPDATES`) | `502` stage `commit`, rolled back the same way; never reported as updated. |
| Old version cannot be removed after the switch (`ATOMIC_UPDATES`) | `502` stage `cleanup`, `new_version_live:true`; searches show only the newest version; sending the document again finishes the cleanup. |
| Rollback itself fails (`ATOMIC_UPDATES`) | `502` with `hidden_chunks_left_behind:true` and a Slack notice. Hidden chunks are never searched; the next successful update of the same document removes them. |
| Page cannot be fetched | `502 source_fetch_failed`, with the HTTP status or reason. |
| Embedding service / vector store down (query) | `503 embedding_unavailable` / `search_unavailable`. |
| Model down or empty answer | `503 llm_generation_failed` with the sources found. |
| Slack down | No effect. |
| Invalid CONFIG | `500 rag_misconfigured` naming every bad setting. |

## Verification

Tested end to end on self-hosted n8n 2.35.7 against in-memory reference implementations of the services above. The original workflow was run first as a baseline: on current n8n it returns an empty `200` for real ingest and query requests (its `$env` reads are blocked); the other defects listed in the architecture notes were found by reading it.

- ingest: chunk count, deterministic UUID ids (checked against Node's SHA-256), payloads, chunk size, overlap, boundaries, no lost text, metadata, normalisation, emoji, no-space text, a 70-chunk document (one embedding call per chunk)
- query: grounded answer with sources best-first, `cited`, no-match path without a model call, `top_k` rounding and clamping, ranking
- prompts: documents in `<document>` tags, injected closing/opening tags neutralised, instructions in the system prompt
- re-ingest: identical (no change), shorter (leftovers removed), changed (old content gone), other documents untouched
- URL ingest: HTML cleaned (script, style, comments, tags, entities), text, JSON, relative and absolute redirects, 404, 500, PDF (422), empty page (422), redirect loop, redirect to a disallowed host, text-over-URL precedence, host outside the allow-list, timeout, oversized page
- 23 internal / reserved / tricky URLs refused with no request made (localhost, loopback, RFC 1918, link-local and cloud metadata, carrier-grade NAT, IPv6 literals, `.local`/`.internal`/single-label hosts, decimal / hex / octal IPs, `user:pass@`)
- 12 invalid ingest shapes, 5 invalid query shapes, a JSON array and malformed JSON rejected before any service is called
- failures: embedding down, one bad chunk, transient errors (retried), vector store write and cleanup failures with the right stage, unusable / non-numeric / wrong-length vectors, time budget, per-call timeouts, Slack down; query-side embedding, vector store and model outages, empty model answer
- all-or-nothing updates (`ATOMIC_UPDATES`): versioned live chunks, a failure half-way through a 140-chunk update leaves the old version whole and served, a store that ignores or rejects the switch-on step is caught and rolled back, a failed rollback is reported, hidden chunks are never searched, a failed cleanup shows only the newest version, a newer version saved by someone else is never deleted, pre-feature documents are replaced
- per-tenant access (`TENANTS_JSON`): no / wrong / near-miss / old single tokens refused on all three endpoints before any service is called, 403 for a tenant without the right `can`, two tenants with the same `doc_id` stay separate (ids, payloads, searches), the tenant cannot be chosen from the body, metadata or filters, updates, leftover cleanup, atomic update rollback and deletes never touch another tenant, a store that ignores filters still cannot leak, documents without a tenant are invisible, ops notices name the tenant, a broken `TENANTS_JSON` is reported without echoing tokens
- concurrency: six different documents at once, three copies of one document at once
- secured configuration (separate tokens, API keys to the services, model header), OpenAI-style and Ollama-style embedding endpoints with a custom collection name, expected-dimension mismatch, tight limits (chunk size, score threshold, context size, top_k cap), broken CONFIG reported with every problem named

Not tested: a real embedding model or language model (retrieval and answer quality are unproven here), Qdrant versions other than 1.19.1, Qdrant Cloud and clustered Qdrant, any other vector database, n8n 1.x, queue mode, collections with millions of chunks.

## Known limits

- **Access control is per tenant, not per user or per document.** Without `TENANTS_JSON` anyone who may query reads everything. With it, everyone holding one tenant's token shares that tenant's documents; there is no finer permission inside a tenant. Tenant tokens live in the `CONFIG` node, so protect who can open the workflow in n8n, and rotate a token by editing `TENANTS_JSON`. Tenant separation is a filter on one shared collection, enforced by this workflow and double-checked on every result; if you need hard isolation (regulated data, a tenant that must never share storage), use a separate collection or vector store per tenant.
- **Switching `TENANTS_JSON` on or off on a collection that already has documents** hides the old ones (on) or exposes tenant documents to everyone (off). Use a fresh collection.
- **The URL guard checks names, not DNS.** A hostname that resolves to a private address is not caught when `ALLOWED_SOURCE_HOSTS` is blank (the Code node cannot resolve DNS). Set `ALLOWED_SOURCE_HOSTS` for production, and block internal ranges at the network level too.
- **The size limit on a fetched page is checked after it is downloaded**, so a huge page still costs memory and time before it is refused.
- **Text extraction is simple.** HTML boilerplate (menus, footers) is kept, pages built by JavaScript return little or nothing, and PDFs, Word files and images are not supported (convert them to text first).
- **Updates are all-or-nothing only with `ATOMIC_UPDATES=true`.** Off (the default), a failed update can leave a mix: some chunks new, some old until the retry succeeds (the reply says `retry:true`), and two simultaneous ingests of one `doc_id` can interleave. On, the old version stays whole until the new one is complete and verified. It is not a database transaction: the switch is one set-payload call plus a count check, so a store that is not strongly consistent could briefly show both versions (searches then show only the newest). Two updates of one document at the same instant: the newer wins and the other may report `commit`; still send updates for one document one at a time.
- **Switching `ATOMIC_UPDATES` back off** while hidden chunks exist would make those chunks searchable. If you turn it off, delete the affected documents and ingest them again. Switching it on is safe: old documents are replaced on their next update.
- **Atomic updates need a store that supports** set-payload by filter, `must_not` filters and `is_empty` (Qdrant does). The test suite proves the workflow's logic against the mock, not against a real Qdrant.
- **Delete is by `doc_id` only.** There is no "delete everything matching this metadata" call (by design: a wide delete should be a deliberate act in your vector store). A delete is permanent; keep your source documents so you can ingest again.
- **Answers are not verified against the sources.** `cited` only shows which retrieved documents the answer names; a real model can still misread or ignore them. Prompt-injection protection is reduced, not eliminated.
- **The caller waits for the whole ingest.** A long document takes as long as its embeddings do (up to `INGEST_MAX_SECONDS`).
- **The collection must already exist** with the right vector size; this workflow does not create it.
- n8n's own `422` for malformed JSON includes a stack trace in its body; put a gateway in front if that matters.
