# STT Sample Data

This directory contains sample input and output files for the Kotoba STT bidirectional streaming service.

## Files

| File                 | Description                             |
| -------------------- | --------------------------------------- |
| `sample_input.json`  | Input event sequence (client → server)  |
| `sample_output.json` | Output event sequence (server → client) |

## sample_input.json

Contains the event sequence that clients send to the STT service:

1. `transcription_session.update` - Configure session (audio format, language)
2. `input_audio_buffer.append` - Send audio chunks (base64 encoded)
3. `input_audio_buffer.commit` - Signal end of audio

Set `input_audio_transcription.kana` to `true` to transcribe person names in katakana.
Set `input_audio_transcription.keywords` to bias recognition toward the listed terms.

### Turn detection (server VAD)

`turn_detection` accepts either:

- `false` (default) — no automatic turn detection; concatenate `delta` events and use `input_audio_buffer.committed` as the "done" signal.
- a server VAD config object — `{ "type": "server_vad", "silence_duration_ms": 400 }`. The server then emits a `conversation.item.input_audio_transcription.completed` event (carrying the turn's `item_id` and full `transcript`) at each detected turn boundary.

`silence_duration_ms` is the trailing silence that ends a turn. **It defaults to 400 ms, but should be tuned for your application and the pipeline it is embedded in** (e.g. shorter for snappy turn-taking, longer to avoid splitting on natural pauses).

### Audio Format Specifications

| Format    | Sample Rate | Channels | Encoding              |
| --------- | ----------- | -------- | --------------------- |
| `pcm16`   |             | 1 (mono) | Little-endian int16   |
| `float32` |             | 1 (mono) | Little-endian float32 |
| `mulaw`   | 8000 Hz     | 1 (mono) | G.711 μ-law           |
| `opus`    | 24000/48000 | 1 (mono) | Ogg/Opus per append   |

### Limits

- Max audio per event: 8 seconds (~1MB)
- Recommended chunk size: 80ms
- Recommended sampling rate: 24kHz

## sample_output.json

Contains the event sequence that the server sends back:

1. `transcription_session.created` - Session initialized
2. `transcription_session.updated` - Configuration confirmed
3. `conversation.item.created` - Audio processing started
4. `conversation.item.input_audio_transcription.delta` - Transcription fragments (multiple)
5. `input_audio_buffer.committed` - Processing complete

When `turn_detection` is a `server_vad` object, the server additionally emits `conversation.item.input_audio_transcription.completed` (one per detected turn, with the turn's full `transcript`).

## Protocol Flow

```
Client                                    Server
  │                                         │
  │◄──── transcription_session.created ─────│
  │                                         │
  │──── transcription_session.update ──────►│
  │                                         │
  │◄──── transcription_session.updated ─────│
  │                                         │
  │──── input_audio_buffer.append ─────────►│
  │     (audio chunks)                      │
  │◄──── transcription.delta ───────────────│
  │     (text fragments)                    │
  │◄──── transcription.completed ───────────│
  │     (if turn_detection on, per turn)    │
  │                                         │
  │──── input_audio_buffer.commit ─────────►│
  │                                         │
  │◄──── input_audio_buffer.committed ──────│
  │                                         │
```
