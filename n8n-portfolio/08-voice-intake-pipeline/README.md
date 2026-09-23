# 08 — Voice Intake Pipeline

## Business problem
A voice interface (built here on Whisper STT and Coqui TTS, fed by a
push-to-talk style client) has to handle a lot that a text chat interface
never does: silence, mumbled or low-confidence audio, unsupported
languages, and STT/TTS services that can independently fail. A naive
pipeline either crashes on the first bad recording or blindly feeds
garbled text to the agent and confidently speaks back nonsense. This one
treats "I couldn't understand you" as a normal, first-class outcome.

## Architecture

`POST /webhook/voice/intake` → `{ audio_base64, audio_format, session_id?, duration_seconds? }`

1. **Validate** the payload (format, rough size cap, duration bounds).
2. **Resolve session** — an existing `session_id` fetches conversation
   history; missing, absent, or expired degrades to a fresh session
   rather than failing.
3. **Transcribe** via Whisper; a hard STT failure still produces a
   spoken/text apology rather than a dead end.
4. **Assess transcript quality** — silence or low average confidence
   (from Whisper's segment log-probabilities) routes to a clarification
   prompt *before* ever reaching the agent.
5. **Language check** — an unsupported detected language gets an apology
   in a default language rather than being forced through the agent.
6. **Agent call** — only a real, successful exchange gets recorded into
   session history; clarifications, apologies, and error responses don't
   pollute the context for the next turn.
7. **Truncate for TTS** — the spoken response respects the TTS engine's
   own length limit, independent of the LLM's; the full text still goes
   back in the JSON response either way.
8. **Synthesize speech**; a TTS failure degrades to a text-only response
   rather than failing the whole interaction.
9. **Archive for QA** (best-effort, parallel with the response) and
   respond.

## Edge cases handled
- **Silence** — an empty transcript never reaches the agent; it gets a
  "could you repeat that?" prompt instead of the agent improvising a
  response to nothing.
- **Low-confidence transcription** — acting on a likely-garbled transcript
  is worse than asking the person to repeat themselves, so it's gated the
  same way as silence, before the agent ever sees it.
- **Unsupported language** — handled explicitly with its own apology
  rather than passing garbage-in-a-language-the-agent-doesn't-expect
  through to the LLM.
- **Expired/missing session** — degrades gracefully to a new session
  instead of erroring, since a real caller's session dropping (a call
  disconnecting, an app being backgrounded) is routine, not exceptional.
- **Conversation history hygiene** — only genuine agent exchanges get
  written to history; transient system responses (clarifications, error
  apologies) are explicitly excluded so they don't corrupt future context.
- **STT hard failure** — still produces a spoken/text response (an
  apology) through the same downstream TTS stage as every other outcome,
  rather than a bare error the caller can't act on.
- **TTS engine length limits** — a separate, explicit truncation step
  from any LLM-side length limit; the full text is preserved in the
  response even when the spoken version is shortened.
- **TTS hard failure** — degrades to a text-only response instead of
  failing the interaction outright — a real accessibility/robustness
  requirement, not just a nice-to-have.
- **Manually attaching binary data** — decoding base64 audio into
  `$binary` inside a Code node isn't automatic; it's constructed
  explicitly, and reading synthesized audio back out afterward is called
  out as something to verify against this n8n version's specific binary
  representation rather than assumed.
- **QA archiving never blocks the response** — runs in parallel with
  (not before) responding to the caller.
- **State surviving external calls** — the same
  `Merge(combineByPosition)` pattern used throughout this portfolio,
  applied five times across STT, agent, session, and TTS calls.

## Environment variables expected
`SESSION_API_URL`, `WHISPER_API_URL`, `AGENT_API_URL`,
`COQUI_TTS_API_URL`, `QA_ARCHIVE_API_URL`
