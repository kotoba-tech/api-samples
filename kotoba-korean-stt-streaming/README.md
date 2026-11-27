# Kotoba Korean Speech-to-Text (STT) Streaming

This directory contains the protocol reference, a sample notebook and sample events for
**Kotoba Korean Speech-to-Text (STT) Streaming**: real-time Korean speech-to-text over SageMaker
bidirectional streaming, with the `/v1/realtime` event protocol. This file and the two sample
files describe the protocol: session fields, events, limits and timeouts.

## Product

| | |
| --- | --- |
| Product | Kotoba Korean Speech-to-Text (STT) Streaming |
| SKU | `kotoba-korean-stt-streaming` |
| Version | `v1.0` |
| Model | Kotoba Korean streaming ASR |
| Pricing | Per audio hour: one inference = 3600 s of decoded input audio |
| Language | `ko` |
| **Text lag** | **about 0.8 s behind the audio** |
| `turn_detection` when omitted | `false` |
| `style_preference` | no Korean styles (`verbatim: false` is rejected; `human_name: "kana"` has no effect) |
| Automatic recovery when the model stops producing text | after 10 s |
| Server session limit | 24 hours |
| Timestamps | not available (`timestamps: true` is rejected) |
| Speaker diarization | not available (`diarization: true` is rejected) |

### Billing

Under per-audio-hour pricing the endpoint counts the decoded input audio of each
`input_audio_buffer.append` (rejected appends and the silence the server adds at commit are
not counted) and reports it to SageMaker during the session, to six decimal places of an
inference. Silence that you send is input audio and is counted.

**What is counted.** The length of the audio you send, after decoding. Connection time is not
counted: an open stream that sends no audio adds no software charge.

| Case | Counted? |
| --- | --- |
| Audio you send in `input_audio_buffer.append`, silence included | Yes |
| Silence the server adds at `input_audio_buffer.commit` | No |
| An `append` that is rejected (for example, audio that cannot be decoded) | No |
| A stream that ends without `commit` (disconnect, expired credentials, the 30-minute limit) | Yes, for the audio already sent |
| An `append` that was decoded but that the server then failed to process (for example, an internal error) | Yes |

**How the listing shows the price.** The AWS Marketplace listing gives the price per "request"
on the dimension `inference.count`. One request there is one inference: 3600 s (one hour) of
audio. It is not one stream and not one event. Shorter audio is billed in proportion. For
example, if the listing price is P per request, a stream that sends 6 minutes of audio is
0.1 inference and costs 0.1 × P. The listing gives the price.

**The other charge.** SageMaker bills the instance hours of the endpoint for as long as the
endpoint runs, also when no audio is sent.

**Tip.** Silence you send is counted, so a client that stops sending during long pauses pays
less. The model writes the text of speech only after it receives some audio that follows it (and,
with turn detection, ends the turn on that audio). Before you stop sending, send silence until
the text of the last speech has arrived, or send `input_audio_buffer.commit`.

Real-time instance types: `ml.g6e.xlarge`, `ml.g6e.2xlarge`, `ml.g6e.4xlarge`, `ml.g6e.8xlarge` and
`ml.g6e.16xlarge` (one NVIDIA L40S), and `ml.g6.xlarge`, `ml.g6.2xlarge`, `ml.g6.4xlarge`,
`ml.g6.8xlarge` and `ml.g6.16xlarge` (one NVIDIA L4). One instance accepts up to **160** concurrent
sessions on L40S (`ml.g6e`) and up to **48** on L4 (`ml.g6`). Both limits were measured on
SageMaker: the L40S limit on `ml.g6e.4xlarge` and `ml.g6e.xlarge`, and the L4 limit on
`ml.g6.4xlarge`. The `ml.g6.xlarge` and `ml.g6.2xlarge` sizes accept the same number of sessions
but were not measured at that load; if sessions end with errors under full load on them, use a
larger size.

### Text lag

The model writes each piece of text about 0.8 s after the audio it belongs to. This lag is a
property of the model, not of the network.

### AWS stream limit

SageMaker keeps a bidirectional stream open for up to **30 minutes**
([AWS documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/realtime-endpoints-test-endpoints.html)).
The stream is signed with the AWS credentials you open it with. **When those credentials expire,
the stream ends**: you receive no `error` event, only the end of the output stream. Temporary
credentials (for example an `aws login` or SSO session that refreshes every 15 minutes) can end a
stream after a few minutes. Open each stream with credentials that outlive the session, or open a
new stream before they expire. The server session limit above never applies on SageMaker.

## Files

| File | Description |
| --- | --- |
| [`kotoba-korean-stt-streaming.ipynb`](./kotoba-korean-stt-streaming.ipynb) | Sample notebook: subscribe, deploy, stream audio, clean up |
| [`data/sample_input.json`](./data/sample_input.json) | Input event sequence (client → server) |
| [`data/sample_output.json`](./data/sample_output.json) | Output event sequence (server → client) |

## data/sample_input.json

1. `transcription_session.update` - Configure session (audio format, language)
2. `input_audio_buffer.append` - Send audio chunks (base64 encoded)
3. `input_audio_buffer.commit` - Signal end of audio

### Session fields

| Field | Default | Notes |
| ----- | ------- | ----- |
| `input_audio_format` | `pcm16` | `pcm16`, `float32`, `mulaw` (`twilio` is an alias) or `opus`. |
| `input_audio_sample_rate` | `24000` | Any positive rate; the server resamples. `mulaw` is always 8 kHz and `opus` is decoded at 48 kHz. |
| `input_audio_number_of_channels` | `1` | PCM input with more channels is downmixed to mono. |
| `input_audio_transcription.target_language` | *(required)* | `"ko"`. Other languages are rejected. |
| `input_audio_transcription.language` | `null` | Optional; if sent, it must equal `target_language`. |
| `input_audio_transcription.keywords` | `null` | Terms to bias recognition toward: up to 1000 terms, each up to 64 characters. |
| `input_audio_transcription.style_preference` | `null` | No Korean styles: `verbatim: false` is rejected, and `human_name: "kana"` is accepted but has no effect. |
| `input_audio_transcription.timestamps` | `false` | Not available: `true` is rejected with `invalid_parameters`. |
| `turn_detection` | `false` | `false`, a semantic VAD object (recommended) or a server VAD object (see *Turn detection*). |
| `diarization` | `false` | Not available: `true` is rejected. |

A `transcription_session.update` with an invalid value is answered with an `error` event
(`error.code`: `invalid_parameters`, `error.param` names the field, for example
`session.input_audio_transcription.target_language`). The server keeps waiting for a corrected
update until 60 s after connecting.

`input_audio_buffer.append` and `input_audio_buffer.commit` take an optional `event_id` of up
to 36 characters (`A-Z`, `a-z`, `0-9`, `_`, `-`).

### Turn detection

`turn_detection` accepts:

- `false` (default) — no automatic turn detection; concatenate `delta` events and use
  `input_audio_buffer.committed` as the "done" signal.
- a semantic VAD object (recommended) — `{ "type": "semantic_vad", "eagerness": "auto" }`.
  `eagerness` is `low`, `medium`, `high` or `auto` (= `medium`).
- a server VAD object — `{ "type": "server_vad", "silence_duration_ms": 800 }`. A turn ends
  after `silence_duration_ms` of model output without text (default **800 ms**).

Semantic VAD ends a turn soon after a sentence is complete and waits longer while it is not;
server VAD ends a turn after every pause of `silence_duration_ms`, which can split a phone
number or an address that is read with pauses. For dictated numbers and addresses, use semantic
VAD or a longer `silence_duration_ms`.

With either VAD object, the server emits `conversation.item.input_audio_transcription.completed`
(the turn's `item_id` and full `transcript`) at each turn boundary. Turn boundaries follow the
model's text, so they come at least one text lag (about 0.8 s) after the speaker stops. **Tune the
turn settings for your application and the pipeline it is embedded in.**

### Audio Format Specifications

| Format    | Sample Rate        | Channels | Encoding                    |
| --------- | ------------------ | -------- | --------------------------- |
| `pcm16`   | any (24 kHz recommended) | 1 or more (downmixed) | Little-endian int16   |
| `float32` | any (24 kHz recommended) | 1 or more (downmixed) | Little-endian float32 |
| `mulaw`   | 8000 Hz            | 1 (mono) | G.711 μ-law                 |
| `opus`    | decoded at 48 kHz  | 1 (mono) | Self-contained Ogg/Opus per append |

### Limits

- Max audio per event: 8 seconds
- Pacing: real time (audio sent more than about 8 s ahead of real time ends the session)
- Recommended chunk size: 80ms
- Recommended sampling rate: 24kHz
- Keywords: up to 1000 terms, each up to 64 characters

### Server-side Timeouts

| Phase               | Server behavior |
| ------------------- | --------------- |
| Session init        | A valid `transcription_session.update` must arrive within **60 s** of connecting. Otherwise the server sends an `error` event (`error.code`: `timeout_error`) and closes the stream. |
| Established session | **No inactivity or result timeout.** A session that never receives `input_audio_buffer.append` stays open until the client or the stream limit closes it. Enforce your own client-side read timeout. |
| After commit        | The server processes the remaining audio (at most 30 s), sends `input_audio_buffer.committed` and closes the stream. |

### Resuming after a disconnect

`input_audio_buffer.committed` carries `audio_processed_ms`, the length of the processed audio.
Deltas also carry `audio_processed_ms`, but on this product it stays `0` until `committed`, so it
cannot mark a resume point mid-stream. After a disconnect, open a new stream and resend the audio
your application has not yet seen transcribed.

## data/sample_output.json

1. `transcription_session.created` - Session initialized
2. `transcription_session.updated` - Configuration confirmed (the applied settings)
3. `conversation.item.created` - Audio processing started
4. `conversation.item.input_audio_transcription.delta` - Transcription fragments (multiple)
5. `input_audio_buffer.committed` - Processing complete

When `turn_detection` is a VAD object, the server additionally emits
`conversation.item.input_audio_transcription.completed` (one per detected turn).

`error` events have the shape `{"type": "error", "event_id": ..., "error": {"type", "code", "message", "param"}}`.

## Known behavior

- **Send audio in real time.** The server keeps about 8 s of audio per session waiting for the model. Audio sent faster than real time (for example a file streamed as fast as possible, or a client catching up after a pause) fills that buffer; the oldest audio is dropped, the session ends with an `error` event (`error.code`: `server_error`, "Internal server error") and the stream closes. Pace appends at the audio's own rate (one 80 ms chunk every 80 ms), and after a client-side pause do not send the missed audio faster than real time.
- Numbers, units and dates are written with digits and symbols (`2005년`, `51%`, `20cm`), not
  spelled out.
- The model sometimes stops producing text for a while. After 10 s the server restarts the model
  context of the session on its own. The connection and the queued audio continue. No event is sent.
- When an instance is full, a new stream is refused, or the server sends an `error` event
  (`error.code`: `system_busy`) and closes it. Retry with backoff, or add instances.

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
  │     (text fragments, ~0.8 s behind)     │
  │◄──── transcription.completed ───────────│
  │     (if turn_detection on, per turn)    │
  │                                         │
  │──── input_audio_buffer.commit ─────────►│
  │                                         │
  │◄──── input_audio_buffer.committed ──────│
  │     (then the server closes the stream) │
```
