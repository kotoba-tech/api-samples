# TTS Sample Data

This directory contains sample input and output files for the Kotoba TTS v1.6 bidirectional streaming service.

## Files

| File                 | Description                             |
| -------------------- | --------------------------------------- |
| `sample_input.json`  | Input event sequence (client → server)  |
| `sample_output.json` | Output event sequence (server → client) |

## sample_input.json

Contains the event sequence that clients send to the TTS service:

1. `open` — Configure session (language, speaker_id, optional `spk_ref_audio_tokens`)
2. `response.create` — Submit the full text for synthesis (one-shot; repeatable per session)

### `open` Parameters

| Parameter              | Required | Default   | Description                                                                                                                                      |
| ---------------------- | -------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `language`             | No       | `ja`      | ISO-639-1 language code. Must be in the server's supported list (`SUPPORTED_LANGUAGES`).                                                         |
| `speaker_id`           | No       | `default` | Preset voice identifier. `default` is always available; additional preset keys depend on the bundle. Unknown ids fail with `unknown_speaker_id`. |
| `format`               | No       | `pcm_f32` | Output audio encoding: `pcm_f32`, `pcm_16`, `mulaw`, or `opus` (see *Output Audio Format*).                                                      |
| `sample_rate`          | No       | `24000`   | Output sample rate (`8000` / `16000` / `24000`). Ignored for `mulaw` (forced 8 kHz) and `opus` (forced 24 kHz).                                  |
| `spk_ref_audio_tokens` | No       | —         | Optional speaker reference tokens for the requested voice.                                                                                       |

The negotiated `format` / `sample_rate` / `channels` are echoed back on
`session.created`. An unsupported `format` or `sample_rate` fails the `open` and
closes the connection.

### `response.create` Parameters

| Parameter     | Required | Default         | Description                                                                                                                                                                                                                                        |
| ------------- | -------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `text`        | **Yes**  | —               | The full text to synthesize. Must be a non-empty string. Sent inside `response.create` — v1.0 does not stream text chunks.                                                                                                                         |
| `response_id` | No       | server-gen UUID | Client-supplied id. Echoed on `response.queued` / `response.created` / every `audio.chunk` / `response.done` for correlation. Must be unique among outstanding (active or queued) responses; reusable once its `response.done` has been delivered. |
| `with_timestamps`  | No       | `false`         | Set to `true` to receive incremental `alignments` records on audio chunks.                                                                                                                  |

You may submit several `response.create` events without **waiting for each
`response.done`** — see *Pipelining* below.

### Timestamps and heard-prefix tracking

When `response.create.with_timestamps` is `true`, an `audio.chunk` includes an
`alignments` field whenever one or more timestamp records have been newly
finalized:

```json
{
  "type": "audio.chunk",
  "audio": "BASE64_ENCODED_PCM_F32_AUDIO_BYTES",
  "isFinal": false,
  "response_id": "resp_001",
  "alignments": [{"start": 0.00, "end": 0.25, "text": "こん"}]
}
```

Each record has the following shape:

| Field   | Type   | Description |
| ------- | ------ | ----------- |
| `start` | number | Start time in seconds from the beginning of this response's output audio. |
| `end`   | number | End time in seconds from the beginning of this response's output audio. |
| `text`  | string | Exact slice of the raw `text` sent in this `response.create`. |

Records are monotone and non-overlapping. Each incremental record is delivered
exactly once, and chunks with no newly finalized records omit `alignments`
entirely. Records normally have roughly token granularity (for example, one
Japanese character or a small multi-character token). Fast speech can finalize
two or more records on one chunk, and a skipped-over token can be represented by
a zero-length record (`start == end`).

The records cover the whole response: once the final `audio.chunk`
(`isFinal: true`) has arrived, the records received across all chunks
concatenate to the full request `text`.

For barge-in or partial-transcript UI, track the index *n* of the `audio.chunk`
currently audible on the speaker, then concatenate `alignments[].text` from
audio chunks `0..n`. That string is what the listener has actually heard. The
recipe works both while audio is being generated and after generation has
completed, with every output format including `opus`; no sample-rate arithmetic
is needed. `response.cancel` is unchanged and only stops generation, so the
client must still use its playback position to choose *n*.

Timestamps are intentionally conservative: a record is emitted only after the
alignment has confirmed that the following text is being spoken. The stream
therefore never over-reports progress, but records typically arrive a few
hundred milliseconds behind the corresponding audio. If a deployment does not
support timestamps, a request with `with_timestamps: true` receives the non-fatal,
response-scoped `timestamps_unavailable` error; the session remains usable.

### Pipelining (queueing multiple responses)

A session synthesizes **one response at a time**, but you do **not** have to wait
for `response.done` before sending the next `response.create`. Extra requests are
**queued** and drained in FIFO order:

- If no response is active, `response.create` starts immediately → `response.created`.
- If a response is already active (or others are queued), the new one is accepted
  and acknowledged with **`response.queued`** (same `response.id`). It later starts
  with `response.created` when its turn comes.
- The pending queue is bounded (`WS_MAX_PENDING_RESPONSES`, default 15). Exceeding
  it returns a non-fatal `error` (`too many queued responses`) carrying the
  rejected `response_id`.

Ordering guarantees (a single client can rely on these):

- `response.done` of response *N* is always delivered **before** `response.created`
  of response *N+1*.
- Queued responses start in the order their `response.create` was sent (FIFO).

### Cancellation

`response.cancel` cancels a response and is acknowledged with a single
`response.done` (`status: cancelled`, same `response.id`).

| Form                                                 | Targets                                                                                                 |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `{"type": "response.cancel"}`                        | the **active** (currently synthesizing) response                                                        |
| `{"type": "response.cancel", "response_id": "<id>"}` | the response with that id — whether it is the **active** one or one still **queued** (see *Pipelining*) |

- Cancelling a **queued** response drops it from the queue without ever starting
  it: you get `response.done` (`status: cancelled`) with **no** preceding
  `response.created` and **no** `usage`. Other queued responses are unaffected.
- Cancelling the **active** response stops the running synthesis, reports any
  `usage` already consumed (omitted if nothing was), and the next queued response
  (if any) starts automatically.
- If the target id is unknown (already finished, or never created) the server
  replies with a **non-fatal** `error` (`no active response` / `response not
  found`) carrying that `response_id`; the connection stays open. A cancel that
  races a natural completion lands here and is harmless.

> In a pipelined client, always cancel **by `response_id`**. The no-id form
> targets "whatever is active right now", which is ambiguous when several
> responses are outstanding.

### Per-response lifecycle

Every transition below carries the same `response.id`. Note the two branches that
trip up clients which assume "`response.created` always comes first":

```
response.create ─► response.created ─► audio.chunk* ─► response.done(completed | failed)
                                  └──► response.done(cancelled)            # cancelled while active
               └─► response.queued ─► response.created ─► … ─► response.done
                                  └──► response.done(cancelled)            # cancelled while queued — NO response.created
               └─► error(fatal:false, response_id)                        # rejected at submission (empty text, duplicate id, queue full)
```

1. A **queued** response cancelled before it starts goes straight to
   `response.done(cancelled)` with no `response.created`.
2. A `response.create` **rejected at submission** terminates with an id-correlated
   `error` (carrying its `response_id`) **instead of** a `response.done`.

### Output Audio Format

The output encoding is negotiated on `open` via the optional `format` /
`sample_rate` fields (both default to `pcm_f32` @ 24000 Hz, the historical
behaviour). The negotiated values are echoed on `session.created`. Audio chunks
are returned as **base64-encoded bytes** in the `audio.chunk.audio` field.

| `format`  | Sample rate (`sample_rate`) | Channels | Encoding                    |
| --------- | --------------------------- | -------- | --------------------------- |
| `pcm_f32` | 8000 / 16000 / 24000 Hz     | 1 (mono) | Little-endian float32       |
| `pcm_16`  | 8000 / 16000 / 24000 Hz     | 1 (mono) | Little-endian signed 16-bit |
| `mulaw`   | 8000 Hz (fixed)             | 1 (mono) | 8-bit G.711 mu-law          |
| `opus`    | 24000 Hz (fixed)            | 1 (mono) | Ogg/Opus                    |

`mulaw` always emits at 8000 Hz and `opus` at 24000 Hz; a `sample_rate`
requested alongside them is ignored.

### Limits

- `open` must be sent within 10s of connection (`WS_OPEN_TIMEOUT`)
- Client idle timeout: 60s (`WS_IDLE_TIMEOUT`). The timer is reset by every
  client message and every major server event (`response.created` / `queued` /
  `done` / `error`) — but **not** by `audio.chunk`. With a normal per-response
  synthesis well under the timeout, draining a queue keeps resetting it, so the
  countdown effectively starts after the last `response.done`.
- Pending queue depth: `WS_MAX_PENDING_RESPONSES` (default 15). A `response.create`
  that would exceed it is rejected with a non-fatal `error` (`too many queued
  responses`).
- Result timeout: 60s (`WS_RESULT_TIMEOUT`); if no audio is produced in time the
  server emits a non-fatal `timeout`, and after `WS_RESULT_TIMEOUT_LIMIT`
  (default 3) consecutive timeouts it emits a `fatal` `error` and closes the
  connection

## sample_output.json

Contains the event sequence that the server sends back:

1. `session.created` — Session established (sent in response to the client's `open`)
2. `response.queued` — A `response.create` was accepted but cannot start yet because another response is active; carries `response.id`. It will later emit `response.created`. Only appears when pipelining (see *Pipelining*).
3. `response.created` — Synthesis response started (carries `response.id`)
4. `audio.chunk` — Audio delta (base64, multiple; each carries `response_id` and an `isFinal` flag). With timestamps enabled, chunks carrying newly finalized records also have `alignments`.
5. `response.done` — Synthesis turn finished (`status`: `completed` / `cancelled` / `failed`; carries `response.id`; carries a `usage` token-count object whenever tokens were consumed)

Additional server events:

- `timeout` — Result timeout notification: the server produced no audio in time (non-fatal; session still alive)
- `error` — Error notification. Carries **`fatal`** (`true` ⇒ the server is closing the connection, reconnect; `false` ⇒ the session continues, keep going) and, for errors tied to a specific `response.create` / `response.cancel`, the **`response_id`** it concerns (the client-supplied id, or `null` when none was provided). Errors with no `response_id` are session-scoped (e.g. malformed `open`, unsupported language); response-scoped non-fatal errors (empty text, duplicate id, queue full, unknown cancel target, or `timestamps_unavailable` when timestamps are requested from a deployment without timestamp support) let you retry on the same connection.

### Response-scoped IDs

v1.0 propagates `response.id` (or `response_id` on `audio.chunk`) so that
clients can correlate streamed audio back to the originating `response.create`.
The id is either the client-supplied `response.create.response_id` or a
server-generated UUID when the client omits it.

### Token usage

`response.done` includes a `usage` object: `input_tokens` (input text),
`output_tokens` (generated audio), and `total_tokens`. On `cancelled` or
`failed` turns the counts reflect what was consumed, and `usage` is omitted when
nothing was consumed.

## Protocol Flow

The client must send `open` first; the server replies with `session.created`
once the language and speaker are validated. If `open` is not sent within
`WS_OPEN_TIMEOUT` (default 10s), the server closes the connection.

```
Client                                    Server
  │                                         │
  │──── open ──────────────────────────────►│
  │     (language, speaker_id,              │
  │      optional spk_ref_audio_tokens)     │
  │                                         │
  │◄──── session.created ───────────────────│
  │     (format, sample_rate, channels,     │
  │      language, speaker_id, client_id)   |
  │                                         │
  │──── response.create ───────────────────►│
  │     (text, response_id?, timestamps?)   │
  │                                         │
  │◄──── response.created ──────────────────│
  │     (response.id)                       │
  │                                         │
  │◄──── audio.chunk (isFinal:false) ───────│
  │     (response_id, base64 audio,          │
  │      alignments? when enabled)           │
  │              ...                        │
  │◄──── audio.chunk (isFinal:true) ────────│
  │     (timestamps when enabled)            │
  │                                         │
  │◄──── response.done ─────────────────────│
  │     (response.id, status: completed)    │
  │                                         │
  │  (session still open; repeat from       │
  │   response.create for another turn)     │
```

### Pipelined Flow (queueing)

Sending a second `response.create` before the first finishes queues it; the
server emits `response.queued` immediately and `response.created` only when the
previous response finishes. `response.done(A)` always precedes `response.created(B)`.

```
Client                                    Server
  │──── response.create (A) -───────────────►│
  │◄──── response.created (A) ───────────────│   A starts immediately
  │──── response.create (B) -───────────────►│
  │◄──── response.queued  (B) ───────────────│   B accepted, waiting
  │◄──── audio.chunk (A) … ──────────────────│
  │◄──── response.done   (A, completed) ─────│   A finishes
  │◄──── response.created (B) ───────────────│   B promoted automatically
  │◄──── audio.chunk (B) … ──────────────────│
  │◄──── response.done   (B, completed) ─────│
```

## Changes from v0.1

If you are migrating from v0.1, the client wire protocol is meaningfully different:

| Aspect                   | v0.1                                                                                              | v1.0                                                    |
| ------------------------ | ------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| Text submission          | `response.create` (no text) + one or more `input_text_buffer.append` + `input_text_buffer.commit` | `response.create` carries the full `text` directly      |
| `input_text_buffer.*`    | Supported                                                                                         | **Removed**                                             |
| `response.create` extras | —                                                                                                 | Optional `response_id`, optional `max_length`           |
| `response.created`       | `{"type": "response.created"}`                                                                    | `{"type": "response.created", "response": {"id": ...}}` |
| `audio.chunk`            | `{"type": "audio.chunk", "audio": ..., "isFinal": ...}`                                           | adds `"response_id": ...`                               |
| `response.done`          | `{"type": "response.done", "response": {"status": ...}}`                                          | adds `"id"`: `{"response": {"id": ..., "status": ...}}` |

The `open` event, `session.created` event, and audio format are unchanged.
v1.0 additionally supports **pipelining** (`response.queued`, FIFO draining),
**cancel-by-`response_id`** (active or queued), and id-correlated, `fatal`-tagged
`error` events — see *Pipelining*, *Cancellation*, and the `error` event above.
The v0.1 protocol reference lives at [../tts-v0.1/](../tts-v0.1/).
