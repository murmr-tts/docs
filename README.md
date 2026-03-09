# murmr

**Text-to-speech where you describe any voice in natural language.**

[![Context7](https://img.shields.io/badge/Context7-indexed-blue)](https://context7.com/murmr-tts/docs)
[![npm](https://img.shields.io/npm/v/@murmr/sdk)](https://www.npmjs.com/package/@murmr/sdk)
[![PyPI](https://img.shields.io/pypi/v/murmr)](https://pypi.org/project/murmr/)
[![Website](https://img.shields.io/badge/website-murmr.dev-black)](https://murmr.dev)

## What is murmr?

murmr is a text-to-speech API that creates voices from natural language descriptions. No preset voices, no audio samples -- just describe what you want and murmr generates it.

- **Voice Design** -- describe any voice in plain English (age, accent, tone, pace) and get speech back
- **OpenAI TTS-compatible** -- drop-in replacement, change the base URL and key to migrate
- **10 languages** -- Chinese, English, Japanese, Korean, German, French, Russian, Portuguese, Spanish, Italian
- **Multiple delivery modes** -- SSE streaming (~450ms to first chunk), real-time WebSocket, and batch
- **Official SDKs** -- Node.js (`@murmr/sdk`) and Python (`murmr`) with streaming, long-form chunking, and error recovery built in

## Quick Start

### curl

```bash
curl -X POST "https://api.murmr.dev/v1/voices/design" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Welcome to our app. Let me show you around.",
    "voice_description": "A warm, friendly female voice, calm and clear",
    "language": "English"
  }' \
  --output welcome.wav
```

### TypeScript

```typescript
import { MurmrClient } from '@murmr/sdk';
import { writeFileSync } from 'node:fs';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

const wav = await client.voices.design({
  input: 'Welcome to our app. Let me show you around.',
  voice_description: 'A warm, friendly female voice, calm and clear',
  language: 'English',
});

writeFileSync('welcome.wav', wav);
```

### Python

```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    wav = client.voices.design(
        input="Welcome to our app. Let me show you around.",
        voice_description="A warm, friendly female voice, calm and clear",
        language="English",
    )

    with open("welcome.wav", "wb") as f:
        f.write(wav)
```

Sign up at [murmr.dev](https://murmr.dev) for a free API key (50,000 characters/month).

## Documentation

### Getting Started

| Document | Description |
|----------|-------------|
| [Installation](docs/installation.md) | SDK setup for Node.js and Python, REST API basics |
| [Quickstart](docs/quickstart.md) | Generate audio, stream it, save a voice, and reuse it in 5 minutes |
| [Authentication](docs/authentication.md) | API key format, Bearer auth, usage headers, security best practices |

### API Reference

| Document | Description |
|----------|-------------|
| [Speech Generation](docs/speech.md) | Batch and streaming endpoints for saved voices (`/v1/audio/speech`) |
| [Voice Design](docs/voicedesign.md) | Create voices from natural language descriptions (`/v1/voices/design`) |
| [Streaming](docs/streaming.md) | SSE format, chunk structure, browser playback with Web Audio API |
| [Voice Management](docs/voices.md) | Save, list, and delete voices |
| [Audio Formats](docs/audio-formats.md) | WAV, MP3, Opus, AAC, FLAC, PCM -- specs and when to use each |

### Real-time

| Document | Description |
|----------|-------------|
| [Real-time WebSocket](docs/realtime.md) | Bidirectional WebSocket API for voice agents and LLM integration |
| [WebSocket Protocol](docs/websocket-protocol.md) | Full protocol reference: message types, binary mode, text buffering, close codes |
| [Browser Client](docs/browser-client.md) | Connect to the WebSocket endpoint from the browser with native APIs |

### Advanced

| Document | Description |
|----------|-------------|
| [Long-Form Audio](docs/long-form.md) | Generate audio from text of any length with automatic chunking |
| [Async Jobs](docs/async-jobs.md) | Submit batch jobs, poll for results, receive webhook notifications |
| [Portable Embeddings](docs/portable-embeddings.md) | Extract voice identity as compact embeddings for multi-tenant apps |
| [Voice Agents](docs/voice-agents.md) | Build conversational agents by streaming LLM tokens into murmr |

### Guides

| Document | Description |
|----------|-------------|
| [Crafting Voice Descriptions](docs/voice-crafting.md) | What works in voice descriptions: demographics, emotion, pace, accent |
| [Style Instructions](docs/style-instructions.md) | Control delivery style with the `instruct` parameter for saved voices |
| [Text Formatting](docs/text-formatting.md) | How whitespace and punctuation affect prosody and pacing |
| [OpenAI Migration](docs/openai-migration.md) | Migrate from OpenAI TTS with minimal code changes |
| [Languages](docs/languages.md) | Supported languages, cross-lingual synthesis, language auto-detection |

### Reference

| Document | Description |
|----------|-------------|
| [Errors](docs/errors.md) | HTTP status codes, structured error format, troubleshooting |
| [Rate Limits](docs/rate-limits.md) | Per-plan limits, concurrent requests, usage headers, overage pricing |
| [Node.js SDK Reference](docs/sdk-reference-node.md) | Complete `@murmr/sdk` API: all methods, parameters, and return types |
| [Python SDK Reference](docs/sdk-reference-python.md) | Complete `murmr` SDK API: sync and async clients, all methods and types |

## Using with AI Assistants

These docs are indexed by [Context7](https://context7.com/murmr-tts/docs), which means AI coding assistants can pull them automatically when you work with murmr.

**In Cursor or Claude Code**, just mention murmr in your prompt:

> "Use murmr to add text-to-speech to my Express app"

The assistant will fetch the relevant docs via Context7 and generate working code with correct API usage.

**Direct MCP query** (for custom setups):

```
Use context7 to look up murmr-tts/docs, then show me how to stream audio with voice design
```

## Links

| | |
|---|---|
| Website | [murmr.dev](https://murmr.dev) |
| Dashboard | [murmr.dev/en/dashboard](https://murmr.dev/en/dashboard) |
| npm | [@murmr/sdk](https://www.npmjs.com/package/@murmr/sdk) |
| PyPI | [murmr](https://pypi.org/project/murmr/) |
| Context7 | [context7.com/murmr-tts/docs](https://context7.com/murmr-tts/docs) |
