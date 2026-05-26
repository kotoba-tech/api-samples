# TTS Sample Data

This directory contains sample input and output files for the Kotoba TTS v1.0 bidirectional streaming service.

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

| Parameter    | Required | Default   | Description                                                                                                                                      |
| ------------ | -------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `language`   | No       | `ja`      | ISO-639-1 language code. Must be in the server's supported list (`SUPPORTED_LANGUAGES`).                                                         |
| `speaker_id` | No       | `default` | Preset voice identifier. `default` is always available; additional preset keys depend on the bundle. Unknown ids fail with `unknown_speaker_id`. |

### `response.create` Parameters

| Parameter     | Required | Default         | Description                                                                                                                |
| ------------- | -------- | --------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `text`        | **Yes**  | —               | The full text to synthesize. Must be a non-empty string. Sent inside `response.create` — v1.0 does not stream text chunks. |
| `response_id` | No       | server-gen UUID | Client-supplied id. Echoed back on `response.created`, every `audio.chunk`, and `response.done` for correlation.           |

### Cancellation

Send `{"type": "response.cancel"}` while a response is active to cancel the
current synthesis. The server replies with `response.done` (`status: cancelled`,
same `response.id`) after a short grace period (`WS_CANCEL_GRACE_TIMEOUT`, default 2s).
There is no `response.cancel` payload — at most one response is active at a time.

### Output Audio Format

The server emits a single hardcoded audio format. Audio chunks are returned as
**base64-encoded bytes** in the `audio.chunk.audio` field.

| Format    | Sample Rate | Channels | Encoding              |
| --------- | ----------- | -------- | --------------------- |
| `pcm_f32` | 24000 Hz    | 1 (mono) | Little-endian float32 |

### Limits

- `open` must be sent within 10s of connection (`WS_OPEN_TIMEOUT`)
- Client idle timeout: 60s (`WS_IDLE_TIMEOUT`)
- Worker result timeout: 60s (`WS_RESULT_TIMEOUT`); after
  `WS_RESULT_TIMEOUT_LIMIT` (default 3) consecutive timeouts the server
  emits a fatal `error` and closes the connection

## sample_output.json

Contains the event sequence that the server sends back:

1. `session.created` — Session established (sent in response to the client's `open`)
2. `response.created` — Synthesis response started (carries `response.id`)
3. `audio.chunk` — Audio delta (base64, multiple; each carries `response_id` and an `isFinal` flag)
4. `response.done` — Synthesis turn finished (`status`: `completed` / `cancelled` / `failed`; carries `response.id`)

Additional non-terminal server events:

- `timeout` — Worker result timeout notification (session still alive)
- `error` — Fatal error (followed by WebSocket close)

### Response-scoped IDs

v1.0 propagates `response.id` (or `response_id` on `audio.chunk`) so that
clients can correlate streamed audio back to the originating `response.create`.
The id is either the client-supplied `response.create.response_id` or a
server-generated UUID when the client omits it.

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
  │     (text, response_id?, max_length?)   │
  │                                         │
  │◄──── response.created ──────────────────│
  │     (response.id)                       │
  │                                         │
  │◄──── audio.chunk (isFinal:false) ───────│
  │     (response_id, base64 pcm_f32)       │
  │              ...                        │
  │◄──── audio.chunk (isFinal:true) ────────│
  │                                         │
  │◄──── response.done ─────────────────────│
  │     (response.id, status: completed)    │
  │                                         │
  │  (session still open; repeat from       │
  │   response.create for another turn)     │
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

The `open` event, `session.created` event, audio format, and cancellation
semantics are unchanged. The v0.1 protocol reference lives at
[../tts-v0.1/](../tts-v0.1/).
