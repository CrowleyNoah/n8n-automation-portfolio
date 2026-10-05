# 08 — Voice Intake Pipeline

A single n8n workflow that takes one spoken turn (a recorded clip), turns it into text, decides whether it is safe to act on, asks an agent for a reply, speaks the reply back, and returns text and audio together. One endpoint: `POST /webhook/voice/intake`.

A voice interface has failure modes a text chat never sees: silence, mumbled audio, a language the agent wasn't built for, and speech services that fail independently. A naive pipeline crashes on the first bad recording, or feeds garbled text to the agent and confidently speaks nonsense. This one treats "I couldn't understand you" as a normal, first-class outcome: silence, low confidence and unsupported languages get a spoken clarification and **never reach the agent**, and every speech service can fail without the caller being left with a dead line.

## Design principle

Nothing is hardcoded. Every service URL, auth header, language, confidence limit, size limit, spoken message and timeout lives in one `CONFIG` node. The workflow talks to four small HTTP services (speech-to-text, agent, text-to-speech, session store) and an optional QA archive, defined by the contract below. Swap any of them by changing a URL; no single vendor is assumed.

## Request

`POST /webhook/voice/intake`

```json
{
  "audio_base64": "UklGRiQAAABXQVZFZm10IBAAAAABAAEA…",
  "audio_format": "wav",
  "session_id": "d3b07384-d9a0-4c9b-8f3e-2b1c9a1f0e11",
  "turn_id": "client-turn-0042",
  "duration_seconds": 3.4
}
```

| Field | Required | Meaning |
|---|---|---|
| `audio_base64` | yes | The recording, base64. A `data:audio/…;base64,` prefix is tolerated. Decoded size limit `MAX_AUDIO_BYTES` (10 MB). |
| `audio_format` | yes | `wav`, `mp3`, `ogg` or `webm` (case-insensitive; the allowed set is a `CONFIG` value). The audio's file signature must match, so mislabelled or garbage uploads are rejected before any service is called. |
| `session_id` | no | Continues a conversation. Unknown or expired ids start a fresh session with a new id (the caller's id is never adopted). 1–128 characters: letters, digits and `. _ : -`. |
| `turn_id` | no | Client-chosen id of this turn. Forwarded to the session store (and the agent) so a retried upload of the same turn can be ignored. Same character rules. |
| `duration_seconds` | no | Client-reported length; must be a number between `MIN_DURATION_SECONDS` (0.3) and `MAX_DURATION_SECONDS` (120). Not verified against the audio. |

If `CONFIG.VOICE_AUTH_TOKEN` is set, callers must send it as the `x-voice-token` header.

## Responses

Every outcome that has something to say returns the same body shape:

```json
{
  "outcome": "answered",
  "session_id": "d3b07384-…",
  "session_status": "existing",
  "transcript": "what are your opening hours",
  "response_text": "We're open nine to five…",
  "language": "en",
  "has_audio": true,
  "response_audio_base64": "UklGR…",
  "response_audio_mime": "audio/wav",
  "truncated_for_speech": false,
  "warnings": []
}
```

| Status | `outcome` | When |
|---|---|---|
| 200 | `answered` | The agent replied. The turn is saved to the session. |
| 200 | `shortcut` | Only if you configure `shortcuts` in the `Voice Rules` node: the caller said one of your phrases and hears your fixed reply (`shortcut` names it). The agent is not called and nothing is saved. |
| 200 | `silent` | Empty transcript, a non-speech marker (`[BLANK_AUDIO]`, `(silence)`, `[Music]`, only dots), or a high no-speech probability. The caller hears `MSG_CLARIFY`. The agent is not called. |
| 200 | `low_confidence` | The speech service reported an average log-probability below `MIN_CONFIDENCE_LOGPROB`. Same spoken clarification. |
| 200 | `unsupported_language` | Detected language not in `SUPPORTED_LANGUAGES_JSON`. The caller hears `MSG_UNSUPPORTED_LANGUAGE`. |
| 503 (`DEGRADED_STATUS_CODE`) | `stt_failed` | The speech-to-text service failed (after one retry). The body still carries the spoken apology, so the client can play it. |
| 503 (`DEGRADED_STATUS_CODE`) | `agent_failed` | The agent errored, timed out, or returned an empty reply. Apology spoken, nothing saved. |
| 400 | — | `{error:"validation_failed", details:[…]}`. Nothing was called. |
| 401 | — | `{error:"unauthorized"}`: missing or wrong `x-voice-token`. |
| 500 | — | `{error:"voice_misconfigured", details:[…]}`: invalid CONFIG, naming every bad setting. |

`warnings` can contain `session_store_unavailable`, `session_not_saved`, `tts_failed` and `tts_invalid_audio`. `session_status` is `new`, `existing` or `unavailable`. `response_audio_base64` is `null` (and `has_audio` false) when synthesis failed or `TTS_ENABLED` is off. Set `DEGRADED_STATUS_CODE=200` if your client treats any non-2xx as "drop the body".

Two cases never reach the workflow because n8n refuses them first: malformed JSON (`422`) and a request over n8n's own size cap, 16 MB by default (an error status; 500 in this version). Keep `MAX_AUDIO_BYTES` below that cap allowing for base64's one-third overhead.

## Async mode: get a job id now, collect the reply later (optional, off by default)

A voice turn waits for speech-to-text, the agent and text-to-speech, which can take a while. With async mode on, `POST /webhook/voice/intake` can answer straight away with a job id, and the caller picks the reply up later. Think of a coat-check ticket: you hand the recording over, get a number, and come back to ask "is it ready?". **Be honest about the fit:** a live phone-style conversation wants the answer now, so async rarely suits it. It is for batch or offline use: voicemail, uploaded recordings, a back-office queue, or a client that cannot hold a connection open for a long turn.

Turn it on in `CONFIG`:

| Key | Default | Meaning |
|---|---|---|
| `ASYNC_MODE` | `off` | `off`: always synchronous, the flags below are ignored. `optional`: synchronous unless the caller asks (`Prefer: respond-async` header or `?async=true`). `always`: every accepted request becomes a job. |
| `JOB_STORE_API_URL` | blank | The job store (contract below). Required unless `off`; must be http or https. |
| `JOB_LEASE_SECONDS` | 300 | How long a job may stay `queued` or `running` before it is reported lost. **Must not be shorter than the slowest possible run** (2 x `STT_TIMEOUT_MS` + `AGENT_TIMEOUT_MS` + 2 x `TTS_TIMEOUT_MS`, 120 s with the defaults), or the check refuses to start. |
| `JOB_RESULT_MAX_BYTES` | 1000000 | Largest result stored (at least 100). Audio is bulky, so a bigger result first loses the reply audio (`result_trimmed: ["audio"]`, `has_audio:false` and the warning `audio_omitted_job_result_too_large`; the text, transcript and everything else stay). If it still does not fit, the job fails with `job_result_too_large`. |
| `ALERT_WEBHOOK_URL` | blank | Optional Slack-style incoming webhook (`POST {text}`). Used only to tell you a job result could not be saved. Blank = no alerts. |

**Submit** exactly as before, adding `?async=true` (or the `Prefer` header). Bad requests (400, 401, 500 bad CONFIG) are still refused on the spot and create no job.

| Status | Body | When |
|---|---|---|
| 202 | `{job_id, state:"queued", status_url, retry_after_seconds}` plus `Location` and `Retry-After` headers | Accepted; the audio will be processed. |
| 200 | `{job_id, state, existing:true, status_url}` | You sent an `Idempotency-Key` header and a live or finished job with that key already exists. Nothing runs twice. |
| 503 | `{error:"job_store_unavailable", retry:true}` | The job store could not create the job. **Nothing was started** (no transcription, no agent call, nothing saved), so retrying is safe. |

**Why jobs are keyed only by the `Idempotency-Key` header:** the same person can legitimately say the same thing twice, so nothing in the audio is used to merge two requests. Without a key, two submissions are two jobs and two agent calls. To make a retried upload harmless, send an `Idempotency-Key` (and, to protect the conversation history, a `turn_id`).

**Collect** with `GET /webhook/voice/intake/jobs?job_id=<id>` (same `x-voice-token` header as the intake endpoint, when `VOICE_AUTH_TOKEN` is set):

| Status | Body | When |
|---|---|---|
| 200 | `{job_id, state, created_at, updated_at, result?, result_trimmed?}` | `state` is `queued`, `running`, `succeeded` or `failed`. `result` is exactly what the synchronous call would have returned: `{http_status, body}`, audio included. A job is `succeeded` when that status is 2xx, so a `503 stt_failed` or `agent_failed` call is a **failed** job carrying the real body (with its spoken apology). A `silent` or `low_confidence` outcome is a 200 and so a succeeded job: read `result.body.outcome`. |
| 200 | `{job_id, state:"failed", error:"worker_lost", message}` | Still queued or running past its lease (n8n restarted or the run died). **Submit again**: a lost job never blocks a new one. |
| 400 / 401 | | Missing or malformed `job_id` / wrong token. |
| 404 | `{error:"async_disabled"}` or `{error:"job_not_found"}` | Async is off, or the id is unknown or belongs to another workflow. |
| 503 | `{error:"job_store_unavailable", retry:true}` | The job store is down. |

The status call returns HTTP 200 for any job that exists; the call's own outcome is in `result.http_status` and `result.body`.

**The job store.** A small HTTP service you run (the mock ships a reference version; put the same four calls in front of Redis, Postgres, Supabase and so on). All JSON, states `queued` → `running` → `succeeded` | `failed`. The kind used here is `voice`.

| Call | Behaviour |
|---|---|
| `POST /jobs {kind, key?, lease_seconds}` | Creates a job; the **store** generates the unguessable `id`. `201 {id, state:"queued", created_at, expires_at}`. If `key` is given and a job with the same `(kind, key)` is queued, running (not expired) or succeeded, return `200 {…that job, existing:true}` instead. A failed or lost job does not block a new one. |
| `PATCH /jobs/{id} {state:"running", lease_seconds}` | `queued` → `running`, extends `expires_at`. `409` if not queued. |
| `PATCH /jobs/{id} {state:"succeeded"\|"failed", result, result_trimmed?}` | Final write. `409` if already final: the first write wins. |
| `GET /jobs/{id}` | `200 {id, kind, state, created_at, updated_at, expires_at, result?, result_trimmed?}` or `404`. |

Send job-store credentials with `SERVICE_HEADERS_JSON`. If saving the result fails three times the call is still processed (and the session saved), you get a notice on `ALERT_WEBHOOK_URL` naming the job if one is set, and the job is left `running` so the lease reports it lost. **The job store holds the transcript, the reply and the reply audio.** That is personal data: give it a short retention and the same care as the session store.

## Architecture

```
request → auth + CONFIG sanity + validation (decode base64, check file signature)   (400 / 401 / 500)
        → session:  id given → fetch   200 → continue with history
                                       404 → new session, NEW id
                                       other → carry on without history ("unavailable")
                    no id    → new session (secure UUID from n8n's Crypto node)
        → build a proper n8n binary → POST to speech-to-text (multipart "file")        retry once
        → judge the transcript:  stt_failed | silent | low_confidence | unsupported_language | proceed
              not proceed → fixed spoken message from CONFIG (agent never called, nothing saved)
              proceed     → agent (JSON: recent history + message + language)
                              reply?  → save the turn to the session → speak it
                              no      → apology, nothing saved
        → cut the spoken text to the TTS limit at a sentence boundary
        → POST to text-to-speech                                                       retry once
              failed or not audio → reply with text only + warning
        → reply (one Respond node for every path)
        → after the reply: archive the interaction (optional)
```

1. **Validation first.** Auth, CONFIG sanity and request shape are checked before anything is called. The audio is decoded and its file signature (`RIFF…WAVE`, `ID3`/MPEG frame, `OggS`, EBML) must match the declared format.
2. **Sessions degrade, they don't fail.** A missing, expired or unknown session starts fresh. A session store that is down means the call continues without history and without saving, with a warning. A real caller's session dropping is routine, not an error. New ids come from n8n's Crypto node (secure random UUIDs); they are the only thing protecting a conversation's history, so they are never taken from the caller and never predictable.
3. **A real binary, a real upload.** The audio is attached with n8n's binary helper and sent as a multipart file (`file`, `input.<format>`, correct MIME type) plus a `language` field. The same helper reads the synthesized audio back, so it works whichever binary storage mode n8n uses (memory, disk).
4. **The transcript is judged before anyone acts on it.** Silence and non-speech markers are caught by empty text, configurable patterns and (when reported) the no-speech probability. Low confidence uses the average log-probability of the segments **when the service reports it**; a service that reports none is not treated as "worst possible confidence". Language names and codes (`English`, `en-US`, `en`) are normalised before the supported-language check. Low confidence is judged before language, because language detection on garbled audio is unreliable.
5. **Fixed replies for fixed situations.** Silence, low confidence, unsupported language, and speech/agent failures each have a message in `CONFIG`, spoken through the same text-to-speech stage as any other reply.
6. **Agent.** Gets a real JSON body: the last `MAX_HISTORY_TURNS` turns, the transcript, the language, and the `turn_id`. It is not retried automatically (agents can have side effects). An error, timeout, malformed body or blank reply is `agent_failed`.
7. **History hygiene.** Only a genuine, successful exchange is written to the session. Clarifications, apologies and failures never are, so they can't corrupt the next turn's context.
8. **Speech length.** The text sent to the TTS engine respects `TTS_MAX_CHARS`, cut at a sentence (or word) boundary, with nothing extra read aloud. The full text is always in the response, with `truncated_for_speech: true`.
9. **Speech failure degrades to text.** A TTS error, an empty body or a non-audio response (even a `200` with a JSON error) gives a `200` with the text, `has_audio:false` and a warning.
10. **Archiving runs after the reply.** A slow or dead archive can never delay or break the answer. Default off.

## Edge cases handled

- **Silence and noise** never reach the agent: empty transcript, `[BLANK_AUDIO]`, `(silence)`, `[Music]`, dots-only, high no-speech probability.
- **Low confidence** is asked to repeat; a transcript at the edge (-0.9 against a -1.0 bar) is let through; an unknown confidence is let through.
- **Unsupported language** gets its own spoken apology; language labels from different services are normalised first.
- **A failing speech-to-text service** is retried once, then answered with a spoken apology and an honest `503`.
- **A failing agent** is never retried blindly, never saved to history, and never replaced by silence.
- **A failing TTS engine** still returns the text; a "successful" response that isn't audio is not trusted.
- **Expired, unknown or missing session**: a fresh server-generated id, never the caller's. Session store down: the turn is answered, with a warning.
- **Retried uploads** with the same `turn_id` are stored once.
- **Long answers**: spoken part trimmed at a sentence end, full text returned.
- **Hostile or broken uploads**: not base64, wrong file signature, oversized, absurd duration, path-like ids are all 400s before any service is called.
- **Concurrency**: ten simultaneous callers each get the answer to their own audio and their own session.
- **State survives external calls.** Downstream nodes read earlier results by node name, so no HTTP response can overwrite request state.

## Configuration (`CONFIG` node)

| Key | Default | Meaning |
|---|---|---|
| `SESSION_API_URL` | `http://localhost:4400` | Base URL of the session store. |
| `STT_API_URL` | `http://localhost:4400/transcribe` | Full URL of the speech-to-text endpoint. |
| `AGENT_API_URL` | `http://localhost:4400/chat` | Full URL of the agent endpoint. |
| `TTS_API_URL` | `http://localhost:4400/synthesize` | Full URL of the text-to-speech endpoint. |
| `QA_ARCHIVE_API_URL` | blank | Full URL for interaction archiving. Blank = off. |
| `SERVICE_HEADERS_JSON` | `{}` | Headers for the session store and the QA archive. |
| `STT_HEADERS_JSON` / `AGENT_HEADERS_JSON` / `TTS_HEADERS_JSON` | `{}` | Headers for each of those services only. |
| `VOICE_AUTH_TOKEN` | blank | If set, callers must send it as `x-voice-token`. |
| `STT_LANGUAGE` | `auto` | Value of the `language` field sent to speech-to-text. |
| `SUPPORTED_LANGUAGES_JSON` | `["en","es","fr","de"]` | Language codes the agent handles. |
| `DEFAULT_LANGUAGE` | `en` | Language for fixed messages and the fallback. Must be in the list above. |
| `MIN_CONFIDENCE_LOGPROB` | -1.0 | Average log-probability below which a transcript is "low confidence" (0 or negative). Only used when the service reports it. |
| `NO_SPEECH_THRESHOLD` | 0.6 | Average `no_speech_prob` above which the clip counts as silence (when reported). |
| `NON_SPEECH_PATTERNS_JSON` | blank / silence / music markers | Regular expressions (case-insensitive) matched against the whole transcript. |
| `ALLOWED_FORMATS_JSON` | `["wav","mp3","ogg","webm"]` | Accepted `audio_format` values. |
| `CHECK_AUDIO_SIGNATURE` | true | Check that the bytes look like the declared format. |
| `MAX_AUDIO_BYTES` | 10485760 | Largest decoded audio. Keep it below n8n's request cap. |
| `MIN_DURATION_SECONDS` / `MAX_DURATION_SECONDS` | 0.3 / 120 | Accepted `duration_seconds`. |
| `MAX_HISTORY_TURNS` | 10 | Turns of history sent to the agent. |
| `TTS_ENABLED` | true | `false` = text-only API (no synthesis, no warning). |
| `TTS_MAX_CHARS` | 2000 | Longest text sent to the TTS engine. |
| `MAX_TTS_AUDIO_BYTES` | 10485760 | Largest accepted synthesized audio. |
| `DEGRADED_STATUS_CODE` | 503 | HTTP status for `stt_failed` / `agent_failed` replies (the body still carries the spoken apology). |
| `MSG_STT_FAILED`, `MSG_CLARIFY`, `MSG_UNSUPPORTED_LANGUAGE`, `MSG_AGENT_FAILED` | English sentences | What the caller hears in each situation. |
| `REQUEST_TIMEOUT_MS` | 8000 | Timeout for the session store and archive. |
| `STT_TIMEOUT_MS` / `AGENT_TIMEOUT_MS` / `TTS_TIMEOUT_MS` | 30000 / 20000 / 20000 | Timeouts for the three AI services. |
| `ASYNC_MODE` / `JOB_STORE_API_URL` / `JOB_LEASE_SECONDS` / `JOB_RESULT_MAX_BYTES` / `ALERT_WEBHOOK_URL` | `off` / blank / 300 / 1000000 / blank | Optional async mode (job id now, reply later); see "Async mode" above. |

## Service contract

All JSON unless noted. Each service gets only its own header set.

**Speech-to-text** (`STT_API_URL`, `STT_HEADERS_JSON`): `POST`, `multipart/form-data` with a file part named `file` (`input.<format>`, `audio/<type>`) and a text field `language`. Response `200 {text: string, language?: string, segments?: [{avg_logprob?: number, no_speech_prob?: number}]}`. This is the shape of Whisper-style servers' verbose output. A service that needs extra fields (a model name, a different part name) needs a small adapter in front of it. Any other status, a malformed body or a missing `text` is a failure.

**Agent** (`AGENT_API_URL`, `AGENT_HEADERS_JSON`): `POST {session_id, history:[{role:"user"|"assistant", text}], message, language, turn_id}` → `200 {reply: string}`. A blank reply counts as a failure.

**Text-to-speech** (`TTS_API_URL`, `TTS_HEADERS_JSON`): `POST {text, language}` → `200` with the audio as the body and an `audio/*` `Content-Type`. A `200` with any other content type, or an empty body, is treated as a failure.

**Session store** (`SESSION_API_URL`, `SERVICE_HEADERS_JSON`)

| Call | Response |
|---|---|
| `GET /sessions/{id}` | `200 {history:[{role:"user"\|"assistant", text}]}`, or `404` if unknown or expired. Anything else = "unavailable". |
| `POST /sessions/{id}/turns` `{turn_id?, user_text, assistant_text}` | `200`. **Creates the session if it doesn't exist**, appends the two entries atomically, and ignores a `turn_id` it has already stored. The store owns expiry (a sliding TTL is typical). |

**QA archive** (`QA_ARCHIVE_API_URL`, `SERVICE_HEADERS_JSON`): `POST {session_id, outcome, transcript, response_text, language, has_audio, warnings, created_at}` → any 2xx. The audio itself is never sent.

## Failure behavior

| What breaks | What happens |
|---|---|
| Speech-to-text fails, times out, returns a non-JSON body | One retry, then `503 stt_failed` with the spoken apology. Agent not called, nothing saved. |
| Agent fails, times out, returns a bad or blank reply | `503 agent_failed` with the spoken apology. Called once. Nothing saved. |
| Text-to-speech fails, returns non-audio or nothing | `200` with the text, `has_audio:false`, `tts_failed` / `tts_invalid_audio`. The turn is still saved. |
| Session store down on lookup | `200`, `session_status:"unavailable"`, `session_store_unavailable`; the turn is not saved. |
| Session store fails on save | `200` with `session_not_saved`. |
| QA archive slow, down, erroring | No effect on the reply (it runs afterwards). |
| Job store down at submit (async) | `503 job_store_unavailable`; nothing transcribed, asked or saved. |
| Job store down when marking "running" (async) | The call still goes ahead; the job just stays `queued` until it finishes. |
| Result cannot be saved after 3 tries (async) | The call is handled and saved; the alert webhook (if set) is told with the job id; the job is reported lost after its lease. |
| Invalid CONFIG | `500 voice_misconfigured` naming every bad setting. |

## Verification

Tested end to end on self-hosted n8n 2.35.7 against an in-memory reference implementation of the service contract above (not included in this repo), using real WAV/MP3/Ogg/WebM container bytes. 200 checks, all passing, across 17 configurations (defaults; auth token + per-service headers + QA archive; text-only; custom limits and messages; signature check off; deliberately bad CONFIG; n8n storing binaries on disk; three rules configurations: the example rules, deliberately broken ones, and rules with a syntax error; and seven async configurations):

- happy path: the audio arrives at the speech service as a multipart file with the right name, type and bytes; the agent gets a real JSON body; TTS speaks exactly the reply; the reply audio is a playable WAV; the turn is saved
- conversation continuity, history cap, unknown or expired session ids, session store down or dropping, save failures, `turn_id` de-duplication
- silence in six forms (empty, `[BLANK_AUDIO]`, dots, `(silence)`, `[Music]`, high no-speech probability); confidence boundary (-1.5 / -0.9) and "no confidence reported"; language normalisation (`Spanish`, `en-US`, `English`), unsupported language, low confidence beating unsupported language
- speech-to-text 500, dropped connection, HTML body, and a flaky first attempt that succeeds on retry; agent 500, dropped, wrong-shape, blank reply (single call, nothing saved); TTS 500, dropped, JSON-with-200, empty body, flaky first attempt
- long replies: full text returned, spoken text cut at a sentence end and within the limit, nothing extra spoken
- mp3, ogg, webm, a `data:` URL prefix and an upper-case format; sixteen kinds of bad input are 400s and touch no service; malformed JSON; an 11 MB clip (over the limit) and a 17 MB request (refused by n8n)
- ten simultaneous callers: no cross-talk, ten distinct sessions
- token 401s; each service receives only its own headers; the archive receives the interaction (without audio), and a slow, failing or dropped archive never affects the reply
- text-only mode; limits, language list, messages, history cap, STT language and degraded status code all taken from CONFIG; bad CONFIG names every problem; n8n's on-disk binary mode
- async mode: off by default (asking for it changes nothing, status endpoint 404); a 202 before the speech service has even answered, with Location and Retry-After; the job moving queued → running → succeeded with a result equal to the synchronous reply, audio included; agent and speech-to-text failures as failed jobs with the real 503 body; bad requests refused with no job; the same `Idempotency-Key` never runs twice, and two submissions without one are two jobs; job store down at submit (nothing transcribed, asked or saved); a failed "mark running" tolerated; the result save retried, then an alert, with the job left running; lost jobs (past their lease) reported as `worker_lost` while finished and in-lease jobs are not; the status endpoint's 401 / 400 / 404 / 503 cases and its refusal to show another workflow's jobs; oversize results losing only the audio, or failing cleanly; `ASYNC_MODE=always`; bad async settings and a lease shorter than the slowest run refused on both endpoints

Not tested: a real job store (only the reference one in the mock); n8n 1.x, S3 or queue-mode binary storage, a real Whisper, TTS engine or LLM, audio recorded by a real browser or phone, real speech (the reference speech service reads scripted audio), sustained load.

## Known limits

- **Synchronous unless you turn async mode on.** By default the caller waits for speech-to-text, the agent and text-to-speech. With the default timeouts and retries the worst case is about two minutes; typical turns are seconds. For live conversation, lower the timeouts (`STT_TIMEOUT_MS`, `AGENT_TIMEOUT_MS`, `TTS_TIMEOUT_MS`) and expect the client to show a "thinking" state.
- **Clip-based, not streaming.** One recording in, one reply out (push-to-talk style).
- **Request size.** Audio travels as base64 inside JSON (about a third larger). n8n's default request cap is 16 MB, so `MAX_AUDIO_BYTES` defaults to 10 MB. To accept more, raise n8n's `N8N_PAYLOAD_SIZE_MAX` as well.
- **Confidence limits are service-specific.** The default -1.0 suits Whisper's `avg_logprob`. A service that reports a different scale needs a different `MIN_CONFIDENCE_LOGPROB`, or should omit `segments` so confidence isn't judged.
- **Non-speech detection is a heuristic.** It catches markers and empties, not the invented phrases ("Thank you.") some speech models produce on silence.
- **Fixed messages are in `DEFAULT_LANGUAGE`.** They are spoken in that language even when the caller spoke another supported one. Per-language messages need a change in `Build Canned Response`.
- **`session_id` is a bearer token.** Anyone holding it can continue (and read the history of) that conversation. Use `VOICE_AUTH_TOKEN`, HTTPS, and short session TTLs.
- **Turns are exactly-once only if the client sends `turn_id`**, and only if your session store and agent honour it. Without it, a client that retries a timed-out upload stores the turn twice.
- **Speech and retries.** A failing speech-to-text or TTS call is retried once, including on a 4xx (a wasted attempt, not a harmful one).
- **`duration_seconds` is client-reported** and is only range-checked. The `webm` signature check only verifies the container header.
- **Async mode needs a job store you run.** The mock ships a reference one, not a production service, and async was tested against that mock only. It rarely suits live conversation (see "Async mode"); it is for batch or offline use.
- **Async work still runs inside one n8n execution.** A restart or n8n's own execution timeout ends it; the lease then reports the job lost and the caller submits again. A turn that already reached the session store before the crash is stored once if the client sends the same `turn_id`.
- **Async has no push notification.** The caller polls the status endpoint. A callback URL is deliberately left out (it needs the same care about which addresses it may call as any outbound URL); adding one means a new node that calls your URL, and the URL must be checked the same way.
- **Anyone holding the workflow token can read that workflow's jobs**, including transcripts and reply audio. Job ids are unguessable but behave like bearer secrets. Many async submissions at once all run at once; use n8n queue mode or a gateway to cap that.
- **Voice is personal data.** The transcript is returned to the caller; archiving is off unless `QA_ARCHIVE_API_URL` is set. Decide retention and consent before turning it on.
