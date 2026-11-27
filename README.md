# Kotoba Inference Runtime - AWS Marketplace Documentation

This repository contains sample notebooks and data files for Kotoba ML products available on AWS Marketplace.

## Products

| Product                                                                             | Description                                                                   | Status    |
| ----------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- | --------- |
| [STT (Speech-to-Text)](./stt/)                                                      | Real-time speech transcription via bidirectional streaming                    | Available |
| [Japanese STT (Speech-to-Text)](./japanese-streaming-stt/)                          | Real-time Japanese speech transcription, billed per hour of audio             | Limited   |
| [TTS (Text-to-Speech)](./tts/)                                                      | Real-time speech synthesis via bidirectional streaming                        | Available |
| [Japanese TTS (Text-to-Speech)](./japanese-streaming-tts/)                          | Real-time Japanese speech synthesis, billed per million bytes of text         | Limited   |
| [Multilingual Speech-to-Text (STT) Streaming](./kotoba-multilingual-stt-streaming/) | Real-time speech transcription in eight languages via bidirectional streaming | Limited   |
| [Korean Speech-to-Text (STT) Streaming](./kotoba-korean-stt-streaming/)             | Real-time Korean speech transcription via bidirectional streaming             | Limited   |

## Important Notice

All Kotoba products use **SageMaker bidirectional streaming only**:

- Standard `POST /invocations` is **NOT supported**
- Batch transform jobs are **NOT supported**
- Requires Python 3.12+ with `aws-sdk-python[sagemaker-runtime-http2]==0.13.1` and `smithy-http[awscrt]` (the sample notebooks use the 0.13 client API)

## Quick Links

### STT (Speech-to-Text)

- **Sample Notebook**: [kotoba-stt-streaming.ipynb](./stt/kotoba-stt-streaming.ipynb)
- **Input Event Sample**: [sample_input.json](./stt/data/sample_input.json)
- **Output Event Sample**: [sample_output.json](./stt/data/sample_output.json)

### Japanese STT (Speech-to-Text)

- **Sample Notebook**: [kotoba-japanese-streaming-stt.ipynb](./japanese-streaming-stt/kotoba-japanese-streaming-stt.ipynb)
- **Input Event Sample**: [sample_input.json](./japanese-streaming-stt/data/sample_input.json)
- **Output Event Sample**: [sample_output.json](./japanese-streaming-stt/data/sample_output.json)

### TTS (Text-to-Speech)

- **Sample Notebook**: [kotoba-tts-streaming.ipynb](./tts/kotoba-tts-streaming.ipynb)
- **Input Event Sample**: [sample_input.json](./tts/data/sample_input.json)
- **Output Event Sample**: [sample_output.json](./tts/data/sample_output.json)

### Japanese TTS (Text-to-Speech)

- **Sample Notebook**: [kotoba-japanese-streaming-tts.ipynb](./japanese-streaming-tts/kotoba-japanese-streaming-tts.ipynb)
- **Input Event Sample**: [sample_input.json](./japanese-streaming-tts/data/sample_input.json)
- **Output Event Sample**: [sample_output.json](./japanese-streaming-tts/data/sample_output.json)

### Multilingual Speech-to-Text (STT) Streaming

- **Protocol Reference**: [README.md](./kotoba-multilingual-stt-streaming/README.md)
- **Sample Notebook**: [kotoba-multilingual-stt-streaming.ipynb](./kotoba-multilingual-stt-streaming/kotoba-multilingual-stt-streaming.ipynb)
- **Input Event Sample**: [sample_input.json](./kotoba-multilingual-stt-streaming/data/sample_input.json)
- **Output Event Sample**: [sample_output.json](./kotoba-multilingual-stt-streaming/data/sample_output.json)

### Korean Speech-to-Text (STT) Streaming

- **Protocol Reference**: [README.md](./kotoba-korean-stt-streaming/README.md)
- **Sample Notebook**: [kotoba-korean-stt-streaming.ipynb](./kotoba-korean-stt-streaming/kotoba-korean-stt-streaming.ipynb)
- **Input Event Sample**: [sample_input.json](./kotoba-korean-stt-streaming/data/sample_input.json)
- **Output Event Sample**: [sample_output.json](./kotoba-korean-stt-streaming/data/sample_output.json)

## Getting Started

1. Subscribe to the product on AWS Marketplace
2. Deploy an endpoint using the sample notebook
3. Start transcribing audio or synthesizing speech in real-time

## Requirements

- AWS Account with SageMaker permissions
- Python 3.12+
- `aws-sdk-python[sagemaker-runtime-http2]==0.13.1` and `smithy-http[awscrt]`
