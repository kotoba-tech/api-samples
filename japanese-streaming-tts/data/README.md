# Japanese TTS Streaming Sample Data

This directory contains sample input and output files for the Kotoba Japanese TTS Streaming bidirectional streaming service.

## Files

| File                 | Description                             |
| -------------------- | --------------------------------------- |
| `sample_input.json`  | Input event sequence (client → server)  |
| `sample_output.json` | Output event sequence (server → client) |

## sample_input.json

Contains the event sequence that clients send to the TTS service:

1. `open` — Configure session (language, speaker_id, optional `voice_settings` defaults)
2. `response.create` — Submit the full text for synthesis (one-shot; repeatable per session)

### `open` Parameters

| Parameter              | Required | Default   | Description                                                                                                                                      |
| ---------------------- | -------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `language`             | No       | `ja`      | ISO-639-1 language code. Must be in the server's supported list (`SUPPORTED_LANGUAGES`).                                                         |
| `speaker_id`           | No       | `default` | Preset voice: `male-01` (male), `female-01`, `female-02`, `female-03`, `female-04` (female). `default` is the same voice as `male-01`. Unknown ids fail with `unknown_speaker_id`. |
| `format`               | No       | `float32` | Output audio encoding: `float32`, `pcm16`, `mulaw`, or `opus` (see *Output Audio Format*).                                                       |
| `sample_rate`          | No       | `24000`   | Output sample rate (`8000` / `16000` / `24000`). Ignored for `mulaw` (forced 8 kHz) and `opus` (forced 24 kHz).                                  |
| `voice_settings`       | No       | —         | Session defaults for `speed`, `pitch` and `volume` (see *Voice settings*). Each `response.create` can override them field by field. An invalid value fails the `open` and closes the connection. |

The negotiated `format` / `sample_rate` / `channels` and the session's
`voice_settings` are echoed back on `session.created`. An unsupported `format` or `sample_rate` fails the `open` and
closes the connection.

### `response.create` Parameters

| Parameter     | Required | Default         | Description                                                                                                                                                                                                                                        |
| ------------- | -------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `text`        | **Yes**  | —               | The full text to synthesize. Must be a non-empty string. Sent inside `response.create` — the model does not stream text chunks. Text past the model's input limit is refused (see *Limits*).                                                            |
| `response_id` | No       | server-gen UUID | Client-supplied id. Echoed on `response.queued` / `response.created` / every `audio.chunk` / `response.done` for correlation. Must be unique among outstanding (active or queued) responses; reusable once its `response.done` has been delivered. |
| `with_timestamps`  | No       | `false`         | Set to `true` to receive incremental `alignments` records on audio chunks. Text of up to 400 characters only (see *Limits*).                                                                |
| `voice_settings`   | No       | session default | Optional object with `speed`, `pitch` and `volume` for this response (see *Voice settings*). Fields left out keep the session default from `open`. An invalid value is rejected with a non-fatal `error` carrying the `response_id`. |

You may submit several `response.create` events without **waiting for each
`response.done`** — see *Pipelining* below.

### Voice settings

`voice_settings` can be set on `open` (the session default) and on each
`response.create` (this response only). Every field is optional:

| Field    | Range            | Default | Effect                                                                                                         |
| -------- | ---------------- | ------- | -------------------------------------------------------------------------------------------------------------- |
| `speed`  | `0.7`–`1.5`      | `1.0`   | Speaking rate without changing the pitch; `1.5` is 1.5× faster.                                                |
| `pitch`  | `-4`–`4`         | `0`     | Pitch shift in semitones without changing the speaking rate.                                                   |
| `volume` | `0.0`–`2.0`      | `1.0`   | Loudness multiplier; `0.0` is silent. Above `1.0` the audio is limited at full scale, so the loudest passages may clip slightly. |

```json
{"type": "response.create", "text": "こんにちは。", "voice_settings": {"speed": 1.2, "pitch": 2, "volume": 0.8}}
```

A value that is out of range or not a number is rejected with an `error`
whose `message` names the field, e.g. `voice_settings.pitch must be between
-4.0 and 4.0` or `voice_settings.speed must be a number`. On `response.create`
the error is non-fatal and carries the `response_id`; on `open` it is fatal.

### Writing `text` for accurate readings

- **Postal codes**: write them as `NNN-NNNN` with a hyphen, e.g. `〒100-0001`.
- **Numbers of four or more digits read as quantities**: separate the thousands
  with commas, e.g. `12,345円`. A digit run without commas may be read digit by
  digit, like an order number.

### Timestamps and heard-prefix tracking

When `response.create.with_timestamps` is `true`, an `audio.chunk` includes an
`alignments` field whenever one or more timestamp records have been newly
finalized:

```json
{
  "type": "audio.chunk",
  "audio": "BASE64_ENCODED_FLOAT32_AUDIO_BYTES",
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

Times refer to the audio as delivered, so they already reflect `voice_settings.speed`.
`pitch` and `volume` do not change the timing.

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
hundred milliseconds behind the corresponding audio.

### Pipelining (queueing multiple responses)

A session synthesizes **one response at a time**, but you do **not** have to wait
for `response.done` before sending the next `response.create`. Extra requests are
**queued** and drained in FIFO order:

- If no response is active, `response.create` starts immediately → `response.created`.
- If a response is already active (or others are queued), the new one is accepted
  and acknowledged with **`response.queued`** (same `response.id`). It later starts
  with `response.created` when its turn comes.
- The pending queue holds at most 15 responses. Exceeding
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
               └─► error(fatal:false, response_id)                        # rejected at submission (empty text, invalid voice_settings, duplicate id, queue full)
```

A `text` past the model's input limit is refused after `response.created`:
`response.done(failed)` with `error.code: "invalid_request"` (see *Limits*). A
response that starts while its instance is at capacity ends the same way, with
`error.code: "over_max_concurrency"` (see *Capacity*).

1. A **queued** response cancelled before it starts goes straight to
   `response.done(cancelled)` with no `response.created`.
2. A `response.create` **rejected at submission** terminates with an id-correlated
   `error` (carrying its `response_id`) **instead of** a `response.done`.

### Output Audio Format

The output encoding is negotiated on `open` via the optional `format` /
`sample_rate` fields (both default to `float32` @ 24000 Hz). The negotiated values are echoed on `session.created`. Audio chunks
are returned as **base64-encoded bytes** in the `audio.chunk.audio` field.

| `format`  | Sample rate (`sample_rate`) | Channels | Encoding                    |
| --------- | --------------------------- | -------- | --------------------------- |
| `float32` | 8000 / 16000 / 24000 Hz     | 1 (mono) | Little-endian float32       |
| `pcm16`   | 8000 / 16000 / 24000 Hz     | 1 (mono) | Little-endian signed 16-bit |
| `mulaw`   | 8000 Hz (fixed)             | 1 (mono) | 8-bit G.711 mu-law          |
| `opus`    | 24000 Hz (fixed)            | 1 (mono) | Ogg/Opus                    |

`mulaw` always emits at 8000 Hz and `opus` at 24000 Hz; a `sample_rate`
requested alongside them is ignored.

### Limits

- `open` must be sent within 10s of connection (`WS_OPEN_TIMEOUT`)
- `with_timestamps: true` accepts `text` of up to 400 characters. Longer text is
  rejected with a non-fatal `error` (`with_timestamps supports text up to 400
  characters`) carrying the `response_id`; split it across several
  `response.create` events, or send it without timestamps.
- Text past the model's input limit (about 1,600 tokens) is refused, not cut:
  the response ends with `response.done` (`status: failed`, `error.code:
  "invalid_request"`, `error.message` like `Input text produced 1700 tokens;
  maximum is 1598`) and no audio. The connection stays open and the response is
  not billed. Split long text across several `response.create` events.
- A response stops at about two minutes of audio: `response.done` ends as
  `completed` with `finish_reason: "length"`, and the response is billed in full
  (see *Billing*).
- Client idle timeout: 60s (`WS_IDLE_TIMEOUT`). The timer is reset by every
  client message and every major server event (`response.created` / `queued` /
  `done` / `error`) — but **not** by `audio.chunk`. With a normal per-response
  synthesis well under the timeout, draining a queue keeps resetting it, so the
  countdown effectively starts after the last `response.done`.
- Responses synthesized at once per instance: 32 on L4, 64 on L40S (see
  *Capacity*).
- Pending queue depth: 15 responses. A `response.create`
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
5. `response.done` — Synthesis turn finished (`status`: `completed` / `cancelled` / `failed`; carries `response.id`; carries a `usage` token-count object whenever tokens were consumed). A response that reached the model's output length limit ends as `completed` with `finish_reason: "length"`; the audio already sent is kept.

Additional server events:

- `timeout` — Result timeout notification: the server produced no audio in time (non-fatal; session still alive)
- `error` — Error notification. Carries **`fatal`** (`true` ⇒ the server is closing the connection, reconnect; `false` ⇒ the session continues, keep going) and, for errors tied to a specific `response.create` / `response.cancel`, the **`response_id`** it concerns (the client-supplied id, or `null` when none was provided). Errors with no `response_id` are session-scoped (e.g. malformed `open`, unsupported language); response-scoped non-fatal errors (empty text, invalid `voice_settings`, timestamps on text over 400 characters, duplicate id, queue full, or unknown cancel target) let you retry on the same connection.

### Response-scoped IDs

The server propagates `response.id` (or `response_id` on `audio.chunk`) so that
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
  │      voice_settings?)                   │
  │                                         │
  │◄──── session.created ───────────────────│
  │     (format, sample_rate, channels,     │
  │      language, speaker_id,              │
  │      voice_settings, client_id)         │
  │                                         │
  │──── response.create ───────────────────►│
  │     (text, response_id?,                │
  │      with_timestamps?, voice_settings?) │
  │                                         │
  │◄──── response.created ──────────────────│
  │     (response.id)                       │
  │                                         │
  │◄──── audio.chunk (isFinal:false) ───────│
  │     (response_id, base64 audio,          │
  │      alignments? when enabled)           │
  │              ...                        │
  │◄──── audio.chunk (isFinal:true) ────────│
  │     (alignments? when enabled)           │
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

## Capacity

The model runs on one GPU. Supported real-time instance types:

| Instance types | GPU | Responses synthesized at once per instance |
| --- | --- | --- |
| `ml.g6.xlarge`, `ml.g6.2xlarge`, `ml.g6.4xlarge`, `ml.g6.8xlarge`, `ml.g6.16xlarge` | 1× NVIDIA L4 | up to 32 |
| `ml.g6e.xlarge`, `ml.g6e.2xlarge`, `ml.g6e.4xlarge`, `ml.g6e.8xlarge`, `ml.g6e.16xlarge` | 1× NVIDIA L40S | up to 64 |

The limit counts responses being synthesized, across every session on the
instance. An open session with no active response does not count, nor do
responses waiting in a session's queue: each session synthesizes one response at
a time.

Capacity never refuses a session at `open`. A response that starts while the instance is
already synthesizing its maximum gets `response.created` and then ends with
`response.done` (`status: failed`, `error.code: "over_max_concurrency"`) and no
audio. It is not billed, and the session stays open. Retry it with backoff, or
raise the endpoint's instance count. The limit applies to the instance that
serves each session, and sessions are not always spread evenly across
instances, so a response can be refused even while the endpoint as a whole is
below the instance count times the limit: plan for headroom.

## Billing

- **Software:** metered per inference. One inference is 1,000,000 UTF-8 bytes
  of input text: the `text` of each `response.create`, counted before
  normalization. See the [AWS Marketplace listing](https://aws.amazon.com/marketplace/pp/prodview-y3o5u6umuskum) for the price.
- **Infrastructure:** SageMaker charges for the endpoint's instances while the
  endpoint exists, even when idle. Delete endpoints you no longer use.

| Case | Billed |
| ---- | ------ |
| The response reaches the model: its first audio arrives, or it is cancelled after `response.created` | All of its input bytes |
| Audio cut at the maximum length (`finish_reason: "length"`) | All of its input bytes |
| You disconnect before a result, or the server fails after the request reached the model | All of its input bytes, at session end |
| Cancelled while still queued (no `response.created`) | Not billed |
| Refused before any audio (e.g. text past the input limit, see *Limits*, or the instance at capacity, see *Capacity*) | Not billed |
| Rejected at submission (`error` with `fatal: false`) | Not billed |

The endpoint reports usage to AWS Marketplace itself; you do not need to do
anything.

## Support

Contact kotoba_product+marketplace@kotoba.tech.
