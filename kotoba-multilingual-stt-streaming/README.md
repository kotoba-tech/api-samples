# Kotoba Multilingual Speech-to-Text (STT) Streaming

Real-time speech transcription in eight languages on Amazon SageMaker. You stream audio over a SageMaker bidirectional stream and get the transcript back on the same stream, as it is spoken.

| | |
| --- | --- |
| Product | Kotoba Multilingual Speech-to-Text (STT) Streaming |
| Version | `v1.0` |
| Languages | Japanese (`ja`), English (`en`), Chinese (`zh`), Taiwan Traditional Chinese (`zh-tw`), Korean (`ko`), Cantonese (`yue`), Spanish (`es`), Vietnamese (`vi`) |
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
- [Audio formats](#audio-formats)
- [Errors](#errors)
- [Limits and timeouts](#limits-and-timeouts)
- [Capacity](#capacity)
- [Billing](#billing)

## Highlights

- Streams transcript deltas while audio is still arriving, about 0.8 s behind the audio.
- Eight languages, one per session.
- For Japanese: katakana person names (`style_preference.human_name: "kana"`) and clean transcripts with fillers and restarts (えーと, あの) removed (`style_preference.verbatim: false`).
- Keyword biasing for names, products and jargon: up to 1,000 terms.
- Optional turn detection: sentence-aware (`semantic_vad`, recommended) or silence-based (`server_vad`).
- Accepts `pcm16`, `float32`, `mulaw` / `twilio` and Ogg/Opus input at any sample rate.

Not available in this version: timestamps and speaker diarization.

## Files

| File | Description |
| --- | --- |
| [`kotoba-multilingual-stt-streaming.ipynb`](./kotoba-multilingual-stt-streaming.ipynb) | Sample notebook: subscribe, deploy, stream audio, clean up |
| [`data/sample_input.json`](./data/sample_input.json) | Client → server event sequence |
| [`data/sample_output.json`](./data/sample_output.json) | Server → client event sequence |

## Quick start

1. Subscribe to the product on [AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-ven4djetklnes) and copy the **Model Package ARN** for your region.
2. Deploy a real-time endpoint from the model package on a supported instance (see [Capacity](#capacity)).
3. Open a bidirectional stream with [`aws-sdk-python[sagemaker-runtime-http2]==0.13.1`](https://pypi.org/project/aws-sdk-python/) and `smithy-http[awscrt]` (Python 3.12+), using model invocation path `invocations-bidirectional-stream`.
4. Exchange the JSON events described below. The [sample notebook](./kotoba-multilingual-stt-streaming.ipynb) has a working client.

> [!IMPORTANT]
> The endpoint accepts **bidirectional streaming only**. `POST /invocations` and batch transform jobs are not supported.
>
> Send every event as a UTF-8 text payload (`RequestPayloadPart(bytes_=..., data_type="UTF8")`).

## Protocol flow

```
Client                                                   Server
  │◄──────────── transcription_session.created ────────────│
  │───────────── transcription_session.update ────────────►│  once
  │◄──────────── transcription_session.updated ────────────│
  │───────────── input_audio_buffer.append ───────────────►│  repeated
  │◄──────────── conversation.item.created ────────────────│  once
  │◄──────────── ...input_audio_transcription.delta ───────│  repeated
  │◄──────────── ...input_audio_transcription.completed ───│  per turn, only with turn detection
  │───────────── input_audio_buffer.commit ───────────────►│  end of audio
  │◄──────────── input_audio_buffer.committed ─────────────│
  │                                         (stream closes) │
```

1. Wait for `transcription_session.created`.
2. Send `transcription_session.update` once. Open a new stream to change the settings.
3. Wait for `transcription_session.updated`, then stream audio with `input_audio_buffer.append`.
4. Read `delta` events as they arrive, and `completed` events if turn detection is on.
5. When the audio ends, send `input_audio_buffer.commit`. The server transcribes the remaining audio, sends `input_audio_buffer.committed` and closes the stream. Open a new stream for the next audio.

## Session configuration

The `session` object of `transcription_session.update`:

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `input_audio_format` | `"pcm16"` \| `"float32"` \| `"mulaw"` \| `"twilio"` \| `"opus"` | `"pcm16"` | Encoding of the audio you send. See [Audio formats](#audio-formats). |
| `input_audio_sample_rate` | integer | `24000` | Sample rate of the audio you send, in Hz. Any positive rate; the server resamples. `mulaw` is always 8 kHz and `opus` is decoded at 48 kHz. |
| `input_audio_number_of_channels` | integer | `1` | PCM input with more channels is downmixed to mono. |
| `input_audio_transcription.target_language` | string | *(required)* | One of `"ja"`, `"en"`, `"zh"`, `"zh-tw"`, `"ko"`, `"yue"`, `"es"`, `"vi"`. `"zh-tw"` returns Taiwan Traditional Chinese. Other values are rejected. |
| `input_audio_transcription.language` | string | `null` | Optional. If sent, it must equal `target_language`. |
| `input_audio_transcription.keywords` | string[] | `null` | Terms the transcript should prefer (names, products, jargon). Up to 1,000 terms of up to 64 characters each. |
| `input_audio_transcription.style_preference.human_name` | `"default"` \| `"kana"` | `"default"` | `"kana"` writes Japanese person names in katakana. |
| `input_audio_transcription.style_preference.verbatim` | boolean | `true` | `true` keeps fillers and restarts (えーと, あの) as spoken. `false` removes them; **Japanese only**: with any other `target_language`, `false` is rejected. |
| `input_audio_transcription.timestamps` | boolean | `false` | Not available: `true` is rejected with `invalid_parameters`. |
| `turn_detection` | `false` \| object | `false` | Turn detection. See [Turn detection](#turn-detection). |
| `diarization` | boolean | `false` | Not available: `true` is rejected with `invalid_parameters`. |

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
      "style_preference": { "human_name": "default", "verbatim": false }
    },
    "turn_detection": { "type": "semantic_vad", "eagerness": "medium" }
  }
}
```

`transcription_session.updated` echoes the settings the server applied.

`yue` is written in Traditional characters in most sessions, but some sessions come back in Simplified characters.

## Client events

| Event | Fields | Description |
| --- | --- | --- |
| `transcription_session.update` | `session` | Configures the session. Send once, after `transcription_session.created`. |
| `input_audio_buffer.append` | `audio`, `event_id` (optional) | One chunk of base64-encoded audio in the configured format. |
| `input_audio_buffer.commit` | `event_id` (optional) | No more audio. The server finishes transcribing and replies with `input_audio_buffer.committed`, whose `item_id` is this `event_id`. |

`event_id` is optional: up to 36 characters of `A-Z`, `a-z`, `0-9`, `_` and `-`.

## Server events

| Event | Fields | Description |
| --- | --- | --- |
| `transcription_session.created` | `event_id` | The stream is open. Send `transcription_session.update` next. |
| `transcription_session.updated` | `event_id`, `session` | The settings were accepted. Start sending audio. |
| `conversation.item.created` | `event_id` | Sent once. Informational. |
| `conversation.item.input_audio_transcription.delta` | `item_id`, `delta`, `audio_processed_ms` | A piece of the transcript. |
| `conversation.item.input_audio_transcription.completed` | `event_id`, `item_id`, `transcript` | The full transcript of one detected turn. Only with turn detection on. |
| `input_audio_buffer.committed` | `event_id`, `item_id`, `audio_processed_ms` | All audio is transcribed. The server closes the stream after this event. |
| `error` | `event_id`, `error` | See [Errors](#errors). |

To build the transcript:

- **Without turn detection**, concatenate the `delta` values in order. `input_audio_buffer.committed` marks the end.
- **With turn detection**, use each turn's `completed.transcript`. The deltas of a turn carry the same server-generated `item_id` as its `completed` event.

`audio_processed_ms` stays `0` on deltas. On `committed` it is the length of the processed audio.

## Turn detection

`turn_detection` controls whether the server groups the transcript into turns and sends a `completed` event for each turn.

| Value | Behavior |
| --- | --- |
| `false` (default) | No turns and no `completed` events. |
| `{"type": "semantic_vad", "eagerness": "medium"}` (recommended) | A turn ends soon after the speaker finishes a sentence, and waits longer through pauses inside a sentence. `eagerness` is `"low"`, `"medium"`, `"high"` or `"auto"` (same as `"medium"`). |
| `{"type": "server_vad", "silence_duration_ms": 800}` | A turn ends after `silence_duration_ms` of model output without text (default `800`). |

Semantic VAD ends a turn soon after a sentence is complete and waits longer while it is not; server VAD ends a turn after every pause of `silence_duration_ms`, which can split a phone number or an address that is read with pauses. For dictated numbers and addresses, use semantic VAD or a longer `silence_duration_ms`.

Tune the turn settings for your application and the pipeline the transcript feeds.

## Audio formats

| `input_audio_format` | Encoding | Sample rate | Channels |
| --- | --- | --- | --- |
| `pcm16` | 16-bit signed integer PCM, little-endian | Any (`input_audio_sample_rate`) | 1 or more (downmixed) |
| `float32` | 32-bit float PCM, little-endian, range `[-1.0, 1.0]` | Any (`input_audio_sample_rate`) | 1 or more (downmixed) |
| `mulaw` | G.711 μ-law | 8,000 Hz | 1 |
| `twilio` | G.711 μ-law (same as `mulaw`) | 8,000 Hz | 1 |
| `opus` | Ogg/Opus; each `append` carries a complete, self-contained Ogg/Opus stream | Decoded at 48,000 Hz | 1 |

Recommendations:

- Send small chunks: **80 ms per `append`** is a good default.
- Use **24 kHz** PCM when you control the capture format.
- **Send audio in real time.** The server keeps about 8 s of audio per session waiting for the model. Audio sent faster than real time (a file streamed as fast as possible, or a backlog sent after a client-side pause) fills that buffer, and the session ends with `server_error`. Send one 80 ms chunk every 80 ms.

## Errors

The server reports problems with an `error` event. Match on `error.code`:

```json
{
  "type": "error",
  "event_id": "...",
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
| `invalid_parameters` | `transcription_session.update` is invalid: unsupported `target_language`, `verbatim: false` with a language other than `ja`, keywords over the limits, `timestamps: true`, `diarization: true`, and similar. `param` names the field. | The stream stays open; send a corrected `transcription_session.update` within 60 s of connecting. |
| `timeout_error` | No valid `transcription_session.update` arrived within 60 s of connecting. | The server closes the stream. |
| `system_busy` | The instance is at its session capacity. | The stream is refused, or the server sends this error and closes it. Retry with backoff, or add instances. |
| `server_error` | An internal error, or audio sent faster than real time (see [Audio formats](#audio-formats)). | The server closes the stream. Start a new session. |

## Limits and timeouts

| Item | Limit |
| --- | --- |
| Audio per `append` | 8 seconds |
| Keywords | 1,000 terms, 64 characters each |
| `event_id` | 36 characters, `A-Z a-z 0-9 _ -` |
| Session init | A valid `transcription_session.update` must arrive within **60 s** of connecting. |
| After commit | The server transcribes the remaining audio (at most 30 s), sends `input_audio_buffer.committed`, then closes the stream. |
| AWS credentials | The stream ends when the AWS credentials that signed it expire. |

After `transcription_session.updated` the server does not close a session for inactivity, even when it receives no audio. Enforce your own client-side read timeout and close idle sessions.

## Capacity

The model runs on one GPU. Supported real-time instance types:

| Instance types | GPU | Concurrent sessions per instance |
| --- | --- | --- |
| `ml.g6e.4xlarge`, `ml.g6e.8xlarge`, `ml.g6e.16xlarge` | 1× NVIDIA L40S | up to 72 (measured on `ml.g6e.4xlarge`) |
| `ml.g7e.4xlarge`, `ml.g7e.8xlarge` | 1× NVIDIA RTX PRO 6000 Blackwell | up to 256 |
| `ml.p5.4xlarge` | 1× NVIDIA H100 | up to 320 |

Smaller `ml.g6e` sizes are not supported: their CPU cannot keep up at this load.

A connection over capacity is refused (see [Errors](#errors)). For more sessions, raise the endpoint's instance count.

## Billing

- **Software:** metered per hour of input audio. The AWS Marketplace listing gives the price per "request" (dimension `inference.count`). One request is 3,600 s (one hour) of decoded input audio, not one stream or one event; shorter audio is billed in proportion. For example, 6 minutes of audio is 0.1 request.
  - Counted: the decoded audio of each `append`, including silence you send. Audio already sent is counted even when the stream ends without `commit`.
  - Not counted: rejected chunks, the silence the server adds at `commit`, and connection time.
  - A client that stops sending during long pauses pays less. Before you stop, send silence until the text of the last speech has arrived, or send `commit`.
- **Infrastructure:** SageMaker charges for the endpoint's instances while the endpoint exists, even when idle. Delete endpoints you no longer use.

See the [AWS Marketplace listing](https://aws.amazon.com/marketplace/pp/prodview-ven4djetklnes) for prices.

## Support

Contact kotoba_product+marketplace@kotoba.tech.
