# OpenAI Migration Guide

murmr is designed as a drop-in replacement for OpenAI's TTS API. Migrate with minimal code changes and gain access to natural voice descriptions, 10 languages, and lower cost per character.

## What Changes

| | OpenAI | murmr |
|---|---|---|
| Base URL | `https://api.openai.com/v1` | `https://api.murmr.dev` |
| API key | `sk-...` | `murmr_sk_live_...` |
| Model | `tts-1` or `tts-1-hd` | Not needed (single model) |
| Voice | 6 fixed names (`alloy`, `echo`, etc.) | Unlimited custom voices via description or saved IDs |

## What Stays the Same

- Endpoint path: `/v1/audio/speech`
- Auth header: `Authorization: Bearer <key>`
- Audio formats: `mp3`, `opus`, `aac`, `flac`, `wav`, `pcm`
- `input` / `text` field name (both accepted)
- Max 4,096 characters per request

## Quick Migration

### Before (OpenAI)

**curl**
```bash
curl https://api.openai.com/v1/audio/speech \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "tts-1", "voice": "nova", "input": "Hello world"}' \
  --output output.mp3
```

**TypeScript**
```typescript
import OpenAI from 'openai';
import { writeFileSync } from 'fs';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const response = await openai.audio.speech.create({
  model: 'tts-1',
  voice: 'nova',
  input: 'Hello, this is a test.',
});

const buffer = Buffer.from(await response.arrayBuffer());
writeFileSync('output.mp3', buffer);
```

**Python**
```python
from openai import OpenAI

client = OpenAI()

response = client.audio.speech.create(
    model="tts-1",
    voice="alloy",
    input="Hello, welcome to our application.",
)

response.stream_to_file("output.mp3")
```

### After (murmr with VoiceDesign)

**curl**
```bash
curl -X POST "https://api.murmr.dev/v1/voices/design" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Hello world",
    "voice_description": "A warm, professional female voice",
    "language": "English"
  }' --output output.wav
```

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';
import { writeFileSync } from 'fs';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

const wav = await client.voices.design({
  input: 'Hello, this is a test.',
  voice_description: 'A warm, friendly female voice similar to a podcast host',
});

writeFileSync('output.wav', wav);
```

**Python**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    wav = client.voices.design(
        input="Hello, welcome to our application.",
        voice_description="A friendly, professional female voice",
        language="English",
    )

    with open("output.wav", "wb") as f:
        f.write(wav)
```

### After (murmr with Saved Voice)

For a workflow closer to OpenAI's fixed voice model, save a voice once and reuse it by ID:

**curl**
```bash
# Submit job (returns 202 with job ID)
curl -X POST "https://api.murmr.dev/v1/audio/speech" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Hello world",
    "voice": "voice_abc123",
    "language": "English",
    "response_format": "mp3"
  }'

# Poll for result
curl "https://api.murmr.dev/v1/jobs/$JOB_ID" \
  -H "Authorization: Bearer $MURMR_API_KEY"
```

**TypeScript**
```typescript
import { MurmrClient, isSyncResponse } from '@murmr/sdk';
import { writeFileSync } from 'fs';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

const result = await client.speech.create({
  input: 'Hello, this is a test.',
  voice: 'voice_abc123',
  response_format: 'mp3',
});

if (isSyncResponse(result)) {
  writeFileSync('output.mp3', Buffer.from(await result.arrayBuffer()));
}
```

**Python**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    result = client.speech.create_and_wait(
        input="Hello, welcome to our application.",
        voice="voice_abc123",
        response_format="mp3",
    )

    with open("output.mp3", "wb") as f:
        f.write(result.audio)
```

## Voice Strategy

OpenAI provides 6 fixed voices. murmr lets you create unlimited custom voices.

**Recommended migration path:**

1. Use VoiceDesign to describe each voice you need
2. Generate a reference audio sample
3. Save the voice for a persistent ID
4. Map your old OpenAI voice names to the new murmr IDs

**TypeScript**
```typescript
// One-time setup: create and save your voices
const wav = await client.voices.design({
  input: 'This is a reference recording for the Nova replacement voice.',
  voice_description: 'A warm, friendly female voice, mid-20s, American',
});

const saved = await client.voices.save({
  name: 'Nova Replacement',
  description: 'A warm, friendly female voice, mid-20s, American',
  audio: wav,
  ref_text: 'This is a reference recording for the Nova replacement voice.',
});

console.log(`Nova replacement: ${saved.id}`);
```

**Python**
```python
# Map OpenAI voices to your saved murmr voices
VOICE_MAP = {
    "alloy": "voice_abc123",
    "echo": "voice_def456",
    "nova": "voice_ghi789",
}

def generate_speech(text: str, voice: str = "alloy", fmt: str = "mp3") -> bytes:
    """Drop-in replacement for OpenAI TTS."""
    murmr_voice = VOICE_MAP[voice]

    with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
        result = client.speech.create_and_wait(
            input=text,
            voice=murmr_voice,
            response_format=fmt,
        )
        return result.audio
```

## Key Differences

| Feature | OpenAI TTS | murmr |
|---------|-----------|-------|
| Voice selection | 6 fixed voices by name | Unlimited custom voices via description or saved IDs |
| Batch response | 200 with audio bytes | 202 async with job ID (poll or webhook) |
| Streaming | Proprietary chunked response | Standard SSE (`text/event-stream`) |
| Audio formats | mp3, opus, aac, flac, wav, pcm | mp3, opus, aac, flac, wav, pcm |
| WebSocket realtime | Yes (separate API) | Yes (`/v1/realtime`) |
| Languages | Auto-detect only | 10 languages with explicit `language` parameter |
| Long-form | Manual chunking required | Built-in `createLongForm()` / `create_long_form()` with auto-chunking |
| Max text length | 4,096 characters | 4,096 characters (unlimited via long-form) |
| `speed` parameter | Supported | Not supported (natural pacing) |

## Additional murmr Features

After migrating, take advantage of murmr-specific capabilities:

- **VoiceDesign** -- Describe any voice instead of choosing from a fixed list
- **Voice Saving** -- Save and reuse voices with consistent IDs
- **Long-Form Audio** -- Built-in chunking for text of any length
- **10 Languages** -- Explicit language parameter with cross-lingual synthesis
- **Typed Streaming** -- Audio chunks with metadata (TTFC, chunk index)
- **WebSocket Realtime** -- Persistent connection for voice agent applications
- **Async Jobs** -- Submit and poll or use webhooks
- **Portable Embeddings** -- Store voice embeddings in your own database

## OpenAI SDK Compatibility Note

If you are using the OpenAI Node.js or Python SDK pointed at murmr's base URL, be aware that the batch endpoint returns HTTP 202 (async) instead of 200. The OpenAI SDK expects 200 and may not handle 202 responses correctly. Use the `@murmr/sdk` or `murmr` packages for full compatibility.

```typescript
// May not work correctly for all request types:
import OpenAI from 'openai';
const openai = new OpenAI({
  apiKey: process.env.MURMR_API_KEY,
  baseURL: 'https://api.murmr.dev/v1',
});

// Use the murmr SDK instead:
import { MurmrClient } from '@murmr/sdk';
const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });
```

## See Also

- [Voice Crafting](./voice-crafting.md) -- Create effective voice descriptions
- [Style Instructions](./style-instructions.md) -- Control voice delivery
- [Async Jobs](./async-jobs.md) -- Batch processing with polling and webhooks
- [Long-Form Audio](./long-form.md) -- Generate audio from text of any length
