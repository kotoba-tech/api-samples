# Kotoba Japanese Speech-to-Text (STT) Streaming

Real-time Japanese speech transcription on Amazon SageMaker. You stream audio over a SageMaker bidirectional stream and get the transcript back on the same stream, as it is spoken.

| | |
| --- | --- |
| Product | Kotoba Japanese Speech-to-Text (STT) Streaming |
| Language | Japanese (`ja`) |
| Transport | SageMaker bidirectional streaming (HTTP/2), JSON events |
| Pricing | Per hour of input audio (see [Billing](#billing)) |

## Contents

- [Highlights](#highlights)
- [Files](#files)
- [Quick start](#quick-start)
- [Protocol flow](#protocol-flow)
- [Session configuration](#session-configuration)
- [Client events](#client-events)
- [Server events](#server-events)
- [Turn detection](#turn-detection)
- [Timestamps](#timestamps)
- [Audio formats](#audio-formats)
- [Errors](#errors)
- [Limits and timeouts](#limits-and-timeouts)
- [Capacity](#capacity)
- [Billing](#billing)

## Highlights

- Streams transcript deltas while audio is still arriving.
- Katakana person names (`style_preference.human_name: "kana"`).
- Clean transcripts with fillers and restarts (えーと, あの) removed (`style_preference.verbatim: false`).
- Keyword biasing for names, products and jargon: up to 1,000 terms.
- Optional turn detection: silence-based (`server_vad`) or sentence-aware (`semantic_vad`).
- Optional timestamps: where each delta and turn falls in the input audio (`start_ms` / `end_ms`).
- Accepts `pcm16`, `float32`, `mulaw` / `twilio` and Ogg/Opus input at any sample rate.

## Files

| File | Description |
| --- | --- |
| [`kotoba-japanese-streaming-stt.ipynb`](../kotoba-japanese-streaming-stt.ipynb) | Sample notebook: subscribe, deploy, stream audio, clean up |
| [`sample_input.json`](./sample_input.json) | Client → server event sequence |
| [`sample_output.json`](./sample_output.json) | Server → client event sequence |

## Quick start

1. Subscribe to the product on [AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-vo2nehk65vyn4) and copy the **Model Package ARN** for your region.
2. Deploy a real-time endpoint from the model package on a supported instance (see [Capacity](#capacity)).
3. Open a bidirectional stream with [`aws-sdk-python[sagemaker-runtime-http2]==0.13.1`](https://pypi.org/project/aws-sdk-python/) and `smithy-http[awscrt]` (Python 3.12+), using model invocation path `invocations-bidirectional-stream`.
4. Exchange the JSON events described below. The [sample notebook](../kotoba-japanese-streaming-stt.ipynb) has a working client.

> [!IMPORTANT]
> The endpoint accepts **bidirectional streaming only**. `POST /invocations` and batch transform jobs do not transcribe audio.
>
> Send every event as a UTF-8 text payload (`RequestPayloadPart(bytes_=..., data_type="UTF8")`). A payload sent without `data_type="UTF8"` reaches the server as a binary frame, which it cannot read as an event, and the session fails.

## Protocol flow

```
Client                                                   Server
  │◄──────────── transcription_session.created ────────────│
  │───────────── transcription_session.update ────────────►│  once
  │◄──────────── transcription_session.updated ────────────│
  │───────────── input_audio_buffer.append ───────────────►│  repeated
  │◄──────────── conversation.item.created ────────────────│  once, on the first append
  │◄──────────── ...input_audio_transcription.delta ───────│  repeated
  │◄──────────── ...input_audio_transcription.completed ───│  per turn, only with turn detection
  │───────────── input_audio_buffer.commit ───────────────►│  end of audio
  │◄──────────── ...delta / ...completed ──────────────────│  remaining audio
  │◄──────────── input_audio_buffer.committed ─────────────│
  │                                         (stream closes) │
```

1. Wait for `transcription_session.created`.
2. Send `transcription_session.update` once. The settings cannot be changed later; open a new stream to change them.
3. Wait for `transcription_session.updated`, then stream audio with `input_audio_buffer.append`.
4. Read `delta` events as they arrive, and `completed` events if turn detection is on.
5. When the audio ends, send `input_audio_buffer.commit`. The server transcribes the remaining audio, sends `input_audio_buffer.committed` and closes the stream. A session handles one commit.

## Session configuration

The `session` object of `transcription_session.update`:

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `input_audio_format` | `"pcm16"` \| `"float32"` \| `"mulaw"` \| `"twilio"` \| `"opus"` | `"pcm16"` | Encoding of the audio you send. See [Audio formats](#audio-formats). |
| `input_audio_sample_rate` | integer | `24000` | Sample rate of the audio you send, in Hz. Any positive rate; the server resamples. Ignored for `mulaw` / `twilio` (8 kHz) and `opus`. |
| `input_audio_number_of_channels` | integer | `1` | Interleaved channels in the `pcm16`, `float32`, `mulaw` or `twilio` audio you send; the server downmixes them to mono. Ignored for `opus`, whose channels come from the stream. |
| `input_audio_transcription.target_language` | string | *(required)* | `"ja"`. |
| `input_audio_transcription.language` | string | `null` | Spoken language: `"ja"`. Optional. |
| `input_audio_transcription.keywords` | string[] | `null` | Terms the transcript should prefer, written as they should appear (names, products, jargon). Up to 1,000 terms of up to 64 characters each, after trimming whitespace; blank terms are dropped. |
| `input_audio_transcription.style_preference.human_name` | `"default"` \| `"kana"` | `"default"` | `"kana"` writes person names in katakana. |
| `input_audio_transcription.style_preference.verbatim` | boolean | `true` | `true` keeps fillers and restarts (えーと, あの) as spoken. `false` removes them for a clean transcript. |
| `input_audio_transcription.timestamps` | boolean | `false` | `true` adds `start_ms` / `end_ms` to deltas and turns. See [Timestamps](#timestamps). |
| `turn_detection` | `false` \| object | `false` | Turn detection. See [Turn detection](#turn-detection). |

Example:

```json
{
  "type": "transcription_session.update",
  "session": {
    "input_audio_format": "pcm16",
    "input_audio_sample_rate": 24000,
    "input_audio_number_of_channels": 1,
    "input_audio_transcription": {
      "language": "ja",
      "target_language": "ja",
      "keywords": ["Kotoba", "音声認識"],
      "style_preference": { "human_name": "kana", "verbatim": false },
      "timestamps": true
    },
    "turn_detection": { "type": "semantic_vad", "eagerness": "medium" }
  }
}
```

`transcription_session.updated` echoes the settings the server applied, defaults included. It can carry fields not listed here; ignore any field you do not recognize.

## Client events

| Event | Fields | Description |
| --- | --- | --- |
| `transcription_session.update` | `session` | Configures the session. Send once, after `transcription_session.created`. |
| `input_audio_buffer.append` | `audio`, `event_id` (optional) | One chunk of base64-encoded audio in the configured format. |
| `input_audio_buffer.commit` | `event_id` (optional) | No more audio. The server finishes transcribing and replies with `input_audio_buffer.committed`. |

`event_id` is optional: up to 36 characters of `A-Z`, `a-z`, `0-9`, `_` and `-`.

- On `append` without turn detection, it is returned as the `item_id` of the deltas for that audio. When buffered audio from several appends lands in one delta, the first `event_id` is used. With turn detection, `item_id` is a server-generated turn ID instead.
- On `commit`, it is returned as the `item_id` of `input_audio_buffer.committed`.

The server sends no acknowledgement for `append`, and ignores events of any other `type`.

## Server events

| Event | Fields | Description |
| --- | --- | --- |
| `transcription_session.created` | `event_id` | The stream is open. Send `transcription_session.update` next. |
| `transcription_session.updated` | `event_id`, `session` | The settings were accepted. Start sending audio. |
| `conversation.item.created` | `event_id` | Sent once, on the first `append`. Informational. |
| `conversation.item.input_audio_transcription.delta` | `item_id`, `delta`, `audio_processed_ms`, `start_ms` / `end_ms` (with timestamps) | A piece of the transcript. Deltas can be empty, for example during silence. |
| `conversation.item.input_audio_transcription.completed` | `event_id`, `item_id`, `transcript`, `start_ms` / `end_ms` (with timestamps) | The full transcript of one detected turn. Only with turn detection on. |
| `input_audio_buffer.committed` | `event_id`, `item_id`, `audio_processed_ms` | All audio is transcribed. The server closes the stream after this event. |
| `error` | `event_id`, `error`, `item_id` | See [Errors](#errors). |

To build the transcript:

- **Without turn detection**, concatenate the `delta` values in order. `input_audio_buffer.committed` marks the end.
- **With turn detection**, either show deltas live and replace them with each turn's `completed.transcript`, or use only the `completed` events. Deltas and their `completed` event share an `item_id`. An empty delta sent before a turn starts has `item_id: null`.

`audio_processed_ms` is a position in the input audio, in milliseconds from the start of the session, before which the server no longer needs the audio. It trails the audio you have sent and can stay `0` until the end. On `committed` it equals the length of the audio you sent.

## Turn detection

`turn_detection` controls whether the server groups the transcript into turns and sends a `completed` event for each turn.

| Value | Behavior |
| --- | --- |
| `false` (default) | No turns and no `completed` events. |
| `{"type": "server_vad", "silence_duration_ms": 800}` | A turn ends after `silence_duration_ms` of audio in which no new words come out. `silence_duration_ms` defaults to `800` when omitted or `null`; `0` ends a turn at the first silent 80 ms chunk. |
| `{"type": "semantic_vad", "eagerness": "medium"}` | A turn ends soon after the speaker finishes a sentence, and waits longer through pauses inside a sentence. `eagerness` is `"low"`, `"medium"`, `"high"` or `"auto"` (default, same as `"medium"`). |

Pauses that end a `semantic_vad` turn:

| `eagerness` | After a finished sentence | Inside a sentence |
| --- | --- | --- |
| `high` | 80 ms | 1,280 ms |
| `medium` / `auto` | 80 ms | 2,400 ms |
| `low` | 400 ms | 4,000 ms |

Tune these for your application and the pipeline the transcript feeds. Shorter values give faster turn-taking; longer values avoid splitting a turn on a natural pause. `input_audio_buffer.commit` ends any turn still open, and its `completed` event arrives before `input_audio_buffer.committed`.

## Timestamps

Set `input_audio_transcription.timestamps` to `true` to learn where the transcript falls in the audio, for example to place subtitles or to jump to a passage in a recording.

- Each `delta` with text carries `start_ms` and `end_ms`: the span of input audio its text was spoken in. Empty or whitespace-only deltas carry neither.
- With turn detection on, each `completed` event carries the span of its turn, from the start of its first delta to the end of its last.
- Times are integer milliseconds from the start of the audio you sent in this session (silence included), not wall-clock time. They never go backwards: each delta's `start_ms` and `end_ms` are at or after the previous delta's.
- Spans are estimated by the model and are approximate. They are calibrated for this model and work with keywords and katakana names.
- Treat both fields as optional. If the server loses the alignment partway through a session, for example while recovering from an internal fault, later deltas and turns omit them and the transcript continues.

```json
{
  "type": "conversation.item.input_audio_transcription.delta",
  "item_id": "6f1d2c3b-4a5e-4f60-8b7a-9c0d1e2f3a4b",
  "delta": "Kotobaの",
  "audio_processed_ms": 0,
  "start_ms": 320,
  "end_ms": 880
}
```

## Audio formats

| `input_audio_format` | Encoding | Sample rate |
| --- | --- | --- |
| `pcm16` | 16-bit signed integer PCM, little-endian | Any (`input_audio_sample_rate`) |
| `float32` | 32-bit float PCM, little-endian, range `[-1.0, 1.0]` | Any (`input_audio_sample_rate`) |
| `mulaw` | G.711 μ-law | 8,000 Hz |
| `twilio` | G.711 μ-law (same as `mulaw`; Twilio Media Streams) | 8,000 Hz |
| `opus` | Ogg/Opus; each `append` carries a complete, self-contained Ogg/Opus stream | Decoded at 48,000 Hz |

Recommendations:

- Send small chunks in real time. **80 ms per `append`** is a good default: the model works in 80 ms frames, and larger chunks add latency.
- Use **24 kHz** PCM when you control the capture format.

## Errors

The server reports problems with an `error` event. Most carry an `error` object; match on `error.code`:

```json
{
  "type": "error",
  "event_id": "sess_...",
  "item_id": null,
  "error": {
    "type": "invalid_request_error",
    "code": "invalid_parameters",
    "message": "...",
    "param": "session.input_audio_transcription.target_language"
  }
}
```

| `error.code` | When | What happens |
| --- | --- | --- |
| `invalid_parameters` | `transcription_session.update` is invalid: unsupported `target_language`, keywords over the limits, an unknown audio format, and similar. `param` names the field when known. | The stream stays open; send a corrected `transcription_session.update` within the init window. |
| `invalid_parameters` | An `append` or `commit` event does not match its schema (for example an `event_id` over 36 characters). | That event is ignored; the session continues. |
| `payload_too_large` | One `append` carries more than 8 seconds of audio. | That chunk is dropped and not billed; the session continues. Split audio into smaller chunks. |
| `invalid_json` | After `transcription_session.updated`, an event is not valid JSON. | That event is ignored; the session continues. |
| `invalid_event` | After `transcription_session.updated`, an event has no `type`. | That event is ignored; the session continues. |
| `timeout_error` | `transcription_session.update` was not accepted within 60 seconds of connecting. | The server closes the stream. |
| `system_busy` | The session could not join the model after it was admitted. | The server closes the stream. Retry with backoff. |
| `server_error` | An internal error. | The server closes the stream. Start a new session. |

Some failures end the session with an `error` event whose `error` is a plain string instead of an object, then close the stream:

- An `append` carries audio that cannot be decoded: invalid base64, PCM whose length is not a whole number of samples × channels, or a corrupt Ogg/Opus stream.
- Before `transcription_session.updated`, an event is not valid JSON.
- Any other unexpected failure.

When the endpoint is already serving its maximum number of sessions, the stream does not open: the connection is refused with HTTP 503 before `transcription_session.created`, with error code `system_busy` (at capacity) or `worker_unavailable` (the model is not ready). Retry with backoff, or add instances.

## Limits and timeouts

| Item | Limit |
| --- | --- |
| Audio per `append` | 8 seconds |
| Keywords | 1,000 terms, 64 characters each |
| `event_id` | 36 characters, `A-Z a-z 0-9 _ -` |
| Session init | `transcription_session.update` must be accepted within **60 s** of connecting. |
| After commit | The server sends `input_audio_buffer.committed` once the remaining audio is transcribed, then closes the stream. |

After `transcription_session.updated` the server does not close a session for inactivity, even when it receives no audio. Enforce your own client-side read timeout and close idle sessions.

## Capacity

The model runs on one GPU. Supported real-time instance types:

| Instance types | GPU | Concurrent sessions per instance |
| --- | --- | --- |
| `ml.g6.xlarge`, `ml.g6.2xlarge`, `ml.g6.4xlarge`, `ml.g6.8xlarge`, `ml.g6.16xlarge` | 1× NVIDIA L4 | up to 48 |
| `ml.g6e.xlarge`, `ml.g6e.2xlarge`, `ml.g6e.4xlarge`, `ml.g6e.8xlarge`, `ml.g6e.16xlarge` | 1× NVIDIA L40S | up to 128 |

A connection over capacity is refused before the stream opens (see [Errors](#errors)). For more sessions, raise the endpoint's instance count.

## Billing

- **Software:** metered per hour of input audio. Usage is counted as each `append` is decoded, so you pay for the audio you send, not for how long the stream stays open. Chunks rejected with an error are not counted.
- **Infrastructure:** SageMaker charges for the endpoint's instances while the endpoint exists, even when idle. Delete endpoints you no longer use.

See the [AWS Marketplace listing](https://aws.amazon.com/marketplace/pp/prodview-vo2nehk65vyn4) for prices.

## Support

Contact kotoba_product+marketplace@kotoba.tech.
