# TTS Sample Data

This directory contains sample input and output files for the Kotoba TTS bidirectional streaming service.

## Files

| File                 | Description                             |
| -------------------- | --------------------------------------- |
| `sample_input.json`  | Input event sequence (client → server)  |
| `sample_output.json` | Output event sequence (server → client) |

## sample_input.json

Contains the event sequence that clients send to the TTS service:

1. `open` - Configure session (language, speaker, optional custom voice)
2. `response.create` - Start a synthesis response (repeatable)
3. `input_text_buffer.append` - Send text chunks
4. `input_text_buffer.commit` - Signal end of text input

### Session Parameters

| Parameter    | Required | Default   | Description                                                                              |
| ------------ | -------- | --------- | ---------------------------------------------------------------------------------------- |
| `language`   | No       | `ja`      | ISO-639-1 language code. Must be in the server's supported list (`SUPPORTED_LANGUAGES`). |
| `speaker_id` | No       | `default` | Preset voice identifier. Currently only `default` is supported.                          |

### Output Audio Format

The server emits a single hardcoded audio format. Audio chunks are returned as
**base64-encoded bytes** in the `audio.chunk.audio` field.

| Format    | Sample Rate | Channels | Encoding              |
| --------- | ----------- | -------- | --------------------- |
| `pcm_f32` | 24000 Hz    | 1 (mono) | Little-endian float32 |

### Limits

- `open` must be sent within 10s of connection (`WS_OPEN_TIMEOUT`)
- Client idle timeout: 60s (`WS_IDLE_TIMEOUT`)
- Worker result timeout: 60s (`WS_RESULT_TIMEOUT`)

## sample_output.json

Contains the event sequence that the server sends back:

1. `session.created` - Session established (sent in response to the client's `open` event)
2. `response.created` - Synthesis response started
3. `audio.chunk` - Audio delta (base64, multiple, `isFinal` flag on last)
4. `response.done` - Synthesis turn finished (`status`: `completed` / `cancelled` / `failed`)

Additional non-terminal server events:

- `timeout` - Worker result timeout notification (session still alive)
- `error` - Fatal error (followed by WebSocket close)

## Protocol Flow

The client must send `open` first; the server replies with `session.created`
once the language and speaker are validated. If `open` is not sent within
`WS_OPEN_TIMEOUT` (default 10s), the server closes the connection.

```
Client                                    Server
  │                                         │
  │──── open ──────────────────────────────►│
  │     (language, speaker_id)              │
  │                                         │
  │◄──── session.created ───────────────────│
  │                                         │
  │──── response.create ───────────────────►│
  │◄──── response.created ──────────────────│
  │                                         │
  │──── input_text_buffer.append ──────────►│
  │     (text chunks)                       │
  │◄──── audio.chunk (isFinal:false) ───────│
  │     (base64 pcm_f32)                    │
  │                                         │
  │──── input_text_buffer.commit ──────────►│
  │                                         │
  │◄──── audio.chunk (isFinal:true) ────────│
  │◄──── response.done ─────────────────────│
  │     (status: completed)                 │
  │                                         │
  │  (session still open; repeat from       │
  │   response.create for another turn)     │
```

## Cancellation

Send `{"type": "response.cancel"}` while a response is active to cancel the
current synthesis. The server responds with `response.done` (`status: cancelled`)
after a short grace period (`WS_CANCEL_GRACE_TIMEOUT`, default 2s).
