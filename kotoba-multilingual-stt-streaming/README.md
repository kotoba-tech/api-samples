# Kotoba Multilingual Speech-to-Text (STT) Streaming

This directory contains the protocol reference, a sample notebook and sample events for
**Kotoba Multilingual Speech-to-Text (STT) Streaming**: real-time speech-to-text in eight languages over
SageMaker bidirectional streaming, with the `/v1/realtime` event protocol. This file and the two
sample files describe the protocol: session fields, events, limits and timeouts.

## Product

| | |
| --- | --- |
| Product | Kotoba Multilingual Speech-to-Text (STT) Streaming |
| SKU | `kotoba-multilingual-stt-streaming` |
| Version | `v1.0` |
| Model | Kotoba streaming ASR v2.0 |
| Pricing | Per audio hour: one inference = 3600 s of decoded input audio |
| Languages | `ja`, `en`, `zh`, `zh-tw`, `ko`, `yue`, `es`, `vi` |
| `turn_detection` when omitted | semantic VAD, `eagerness` `medium` |
| `style_preference.verbatim` when omitted | `true` (fillers kept). `false` is accepted for `ja` only. |
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

Real-time instance types and the concurrent sessions one instance accepts:

| Instance types | GPU | Concurrent sessions per instance |
| --- | --- | --- |
| `ml.g6e.4xlarge`, `ml.g6e.8xlarge`, `ml.g6e.16xlarge` | one NVIDIA L40S | up to **72** (measured on `ml.g6e.4xlarge`) |
| `ml.g7e.4xlarge`, `ml.g7e.8xlarge` | one NVIDIA RTX PRO 6000 Blackwell | up to **256** |
| `ml.p5.4xlarge` | one NVIDIA H100 | up to **320** |

Smaller `ml.g6e` sizes are not supported: their CPU cannot keep up at this load.

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
| [`kotoba-multilingual-stt-streaming.ipynb`](./kotoba-multilingual-stt-streaming.ipynb) | Sample notebook: subscribe, deploy, stream audio, clean up |
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
| `input_audio_transcription.target_language` | *(required)* | One of `ja`, `en`, `zh`, `zh-tw`, `ko`, `yue`, `es`, `vi`. `zh-tw` returns Taiwan Traditional Chinese. Other values are rejected. |
| `input_audio_transcription.language` | `null` | Optional; if sent, it must equal `target_language`. |
| `input_audio_transcription.keywords` | `null` | Terms to bias recognition toward: up to 1000 terms, each up to 64 characters. |
| `input_audio_transcription.style_preference.human_name` | `"default"` | `"kana"` transcribes Japanese person names in katakana. |
| `input_audio_transcription.style_preference.verbatim` | `true` | `false` omits fillers and restarts (e.g. えーと, あの). **Japanese only**: with any other `target_language`, `false` is rejected. |
| `input_audio_transcription.timestamps` | `false` | Not available: `true` is rejected with `invalid_parameters`. |
| `turn_detection` | semantic VAD (`medium`) | `false`, a semantic VAD object (recommended) or a server VAD object (see *Turn detection*). |
| `diarization` | `false` | Not available: `true` is rejected. |

A `transcription_session.update` with an invalid value is answered with an `error` event
(`error.code`: `invalid_parameters`, `error.param` names the field, for example
`session.input_audio_transcription.target_language`). The server keeps waiting for a corrected
update until 60 s after connecting.

`input_audio_buffer.append` and `input_audio_buffer.commit` take an optional `event_id` of up
to 36 characters (`A-Z`, `a-z`, `0-9`, `_`, `-`).

### Turn detection

`turn_detection` accepts:

- omitted (default) — semantic VAD with `eagerness` `medium`.
- `false` — no automatic turn detection; concatenate `delta` events and use
  `input_audio_buffer.committed` as the "done" signal.
- a semantic VAD object (recommended) — `{ "type": "semantic_vad", "eagerness": "auto" }`.
  A turn ends sooner after a finished sentence and later in mid-sentence. `eagerness` is `low`,
  `medium`, `high` or `auto` (= `medium`).
- a server VAD object — `{ "type": "server_vad", "silence_duration_ms": 800 }`. A turn ends
  after `silence_duration_ms` of model output without text (default **800 ms**).

Semantic VAD ends a turn soon after a sentence is complete and waits longer while it is not;
server VAD ends a turn after every pause of `silence_duration_ms`, which can split a phone
number or an address that is read with pauses. For dictated numbers and addresses, use semantic
VAD or a longer `silence_duration_ms`.

With either VAD, the server emits `conversation.item.input_audio_transcription.completed`
(the turn's `item_id` and full `transcript`) at each turn boundary. **Tune the turn settings for
your application and the pipeline it is embedded in.**

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
2. `transcription_session.updated` - Configuration confirmed (the applied settings, including the default `turn_detection`)
3. `conversation.item.created` - Audio processing started
4. `conversation.item.input_audio_transcription.delta` - Transcription fragments (multiple)
5. `conversation.item.input_audio_transcription.completed` - One per detected turn (turn detection on, the default)
6. `input_audio_buffer.committed` - Processing complete

`error` events have the shape `{"type": "error", "event_id": ..., "error": {"type", "code", "message", "param"}}`.

## Known behavior

- **Send audio in real time.** The server keeps about 8 s of audio per session waiting for the model. Audio sent faster than real time (for example a file streamed as fast as possible, or a client catching up after a pause) fills that buffer; the oldest audio is dropped, the session ends with an `error` event (`error.code`: `server_error`, "Internal server error") and the stream closes. Pace appends at the audio's own rate (one 80 ms chunk every 80 ms), and after a client-side pause do not send the missed audio faster than real time.
- Text arrives about 0.8 s after the audio it transcribes. This lag is a property of the model, not of the network.
- The model sometimes stops producing text for a while. After 10 s the server restarts the model
  context of the session on its own. The connection and the queued audio continue. No event is sent.
- Cantonese (`yue`) is written in Traditional characters in most sessions, but some sessions come back
  in Simplified characters.
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
  │     (text fragments)                    │
  │◄──── transcription.completed ───────────│
  │     (per turn, if turn_detection is on) │
  │                                         │
  │──── input_audio_buffer.commit ─────────►│
  │                                         │
  │◄──── input_audio_buffer.committed ──────│
  │     (then the server closes the stream) │
```
