# Kotoba Inference Runtime - AWS Marketplace Documentation

This repository contains sample notebooks and data files for Kotoba ML products available on AWS Marketplace.

## Products

| Product                        | Description                                                | Status      |
| ------------------------------ | ---------------------------------------------------------- | ----------- |
| [STT (Speech-to-Text)](./stt/) | Real-time speech transcription via bidirectional streaming | Available   |
| TTS (Text-to-Speech)           | Real-time speech synthesis via bidirectional streaming     | Coming Soon |

## Important Notice

All Kotoba products use **SageMaker bidirectional streaming only**:

- Standard `POST /invocations` is **NOT supported**
- Batch transform jobs are **NOT supported**
- Requires Python 3.12+ with `aws-sdk-python[sagemaker-runtime-http2]`

## Quick Links

### STT (Speech-to-Text)

- **Sample Notebook**: [kotoba-stt-streaming.ipynb](./stt/kotoba-stt-streaming.ipynb)
- **Input Event Sample**: [sample_input.json](./stt/data/sample_input.json)
- **Output Event Sample**: [sample_output.json](./stt/data/sample_output.json)

## Getting Started

1. Subscribe to the product on AWS Marketplace
2. Deploy an endpoint using the sample notebook
3. Start transcribing audio in real-time

## Requirements

- AWS Account with SageMaker permissions
- Python 3.12+
- `aws-sdk-python[sagemaker-runtime-http2]`
