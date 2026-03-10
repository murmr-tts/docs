# VoiceDesign API

Create any voice by describing it in natural language. Generate speech with custom voices — no presets, no limitations.

> **💡 murmr's Key Differentiator**
> VoiceDesign lets you describe any voice in natural language — "A warm, professional female voice, calm and confident" — and generate speech with that voice instantly. Try different descriptions in the [Playground](https://murmr.dev/en/dashboard/playground) before coding.

## Endpoints

`POST /v1/voices/design`

Sync binary audio — returns complete audio file with `response_format` support (mp3 default)

`POST /v1/voices/design/stream`

SSE streaming — low-latency PCM audio chunks

> **ℹ️ Two modes**
> `/v1/voices/design` returns `200` with binary audio (mp3 default). `/v1/voices/design/stream` returns Server-Sent Events with PCM audio chunks for low-latency playback. Both accept the same JSON request body.

## Request Parameters

Send a JSON body with the following parameters:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `text` | string | Yes | -- | The text to synthesize. Maximum 4,096 characters. |
| `voice_description` | string | Yes | -- | Natural language description of the voice (e.g., "A warm, friendly male voice"). Maximum 500 characters. |
| `language` | string | No | Auto | Output language: English, Spanish, Portuguese, German, French, Italian, Chinese, Japanese, Korean, Russian, or "Auto" (detect from text) |
| `input` | string | No | -- | Alias for text (OpenAI API compatibility). If both text and input are provided, text takes precedence. |

## Sync Request (Binary Audio)

`/v1/voices/design` returns a complete audio file. Default format is mp3. Use this when you want the full file without handling SSE chunks.

**cURL**

```curl
curl -X POST "https://api.murmr.dev/v1/voices/design" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Bienvenue dans notre podcast. Explorons ensemble le futur de l\'IA.",
    "voice_description": "A French female narrator with a Parisian accent, warm and articulate",
    "language": "French"
  }' --output french_narrator.mp3
```

**Node.js SDK**

```typescript
import { MurmrClient } from '@murmr/sdk';
import { writeFileSync } from 'fs';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

// design() streams internally and returns a complete WAV buffer
const wav = await client.voices.design({
  input: "Bienvenue dans notre podcast.",
  voice_description: "A French female narrator with a Parisian accent, warm and articulate",
  language: "French",
});

writeFileSync("french_narrator.wav", wav);
```

**Python SDK**

```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    wav = client.voices.design(
        input="Bienvenue dans notre podcast.",
        voice_description="A French female narrator with a Parisian accent, warm and articulate",
        language="French",
    )

    with open("french_narrator.wav", "wb") as f:
        f.write(wav)
```

## Streaming Request (SSE)

`/v1/voices/design/stream` returns audio chunks via Server-Sent Events for low-latency playback (~450ms to first chunk).

**cURL**

```curl
curl -X POST "https://api.murmr.dev/v1/voices/design/stream" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Welcome to our podcast. Today we explore the future of AI.",
    "voice_description": "A warm, professional female voice, 35 years old, calm and confident",
    "language": "English"
  }'
```

**Node.js SDK**

```typescript
import { MurmrClient } from '@murmr/sdk';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

const stream = await client.voices.designStream({
  input: 'Welcome to our podcast. Today we explore the future of AI.',
  voice_description: 'A warm, professional female voice, 35 years old, calm and confident',
});

const pcmChunks: Buffer[] = [];

for await (const chunk of stream) {
  const audioData = chunk.audio || chunk.chunk;
  if (audioData) {
    pcmChunks.push(Buffer.from(audioData, 'base64'));
  }
  if (chunk.first_chunk_latency_ms) {
    console.log(`First chunk in ${chunk.first_chunk_latency_ms}ms`);
  }
  if (chunk.done) {
    console.log(`Complete: ${chunk.total_chunks} chunks in ${chunk.total_time_ms}ms`);
  }
}
```

**Python SDK**

```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    with client.voices.design_stream(
        input="Welcome to our podcast. Today we explore the future of AI.",
        voice_description="A warm, professional female voice, 35 years old, calm and confident",
        language="English",
    ) as stream:
        pcm_chunks = []
        for chunk in stream:
            pcm_chunks.append(chunk.audio_bytes)

        pcm_audio = b"".join(pcm_chunks)
```

**Python (async)**

```python
import asyncio
import os
from murmr import AsyncMurmrClient

async def main():
    async with AsyncMurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
        async with client.voices.design_stream(
            input="Welcome to our podcast. Today we explore the future of AI.",
            voice_description="A warm, professional female voice, 35 years old, calm and confident",
            language="English",
        ) as stream:
            async for chunk in stream:
                pcm_data = chunk.audio_bytes

asyncio.run(main())
```

## Complete WAV (SDK)

The SDK's `design()` method streams internally, collects all chunks, and returns a complete WAV buffer. Use this when you want the full audio file without handling chunks manually.

**Node.js SDK**

```typescript
import { MurmrClient } from '@murmr/sdk';
import { writeFileSync } from 'node:fs';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

const wav = await client.voices.design({
  input: 'The quick brown fox jumps over the lazy dog.',
  voice_description: 'A deep, resonant male voice with a slow, deliberate pace',
  language: 'English',
});

writeFileSync('voicedesign.wav', wav);
```

**Python SDK**

```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    wav = client.voices.design(
        input="The quick brown fox jumps over the lazy dog.",
        voice_description="A young woman with a clear, upbeat tone and American accent",
        language="English",
    )

    with open("output.wav", "wb") as f:
        f.write(wav)
```

## SSE Event Format

```JSON
// Audio chunk
data: {
  "chunk": "<base64 PCM int16>",
  "chunk_index": 0,
  "sample_rate": 24000,
  "format": "pcm_s16le",
  "mode": "voicedesign",
  "first_chunk_latency_ms": 450
}

// Completion
data: {
  "done": true,
  "total_chunks": 5,
  "total_time_ms": 2500,
  "first_chunk_latency_ms": 450,
  "sample_rate": 24000
}
```

See [SSE Streaming](./streaming.md) for complete integration guide including Web Audio API playback and PCM-to-WAV conversion.

## Voice Description Best Practices

> **💡 Comprehensive Guide Available**
> For detailed guidance with examples directly from Qwen3-TTS documentation, see our [Crafting Voices Guide](./voice-crafting.md).

### Effective Descriptions

- "A wise elderly wizard with a deep, mystical voice. Speaks slowly and deliberately with gravitas."
- "Professional male CEO voice, confident and authoritative, measured pace"
- "Warm grandmother voice, gentle and soothing, perfect for bedtime stories"
- "Excited teenage girl, high-pitched voice with lots of energy"

### Avoid

- Celebrity references — "like Morgan Freeman"
- Contradictory traits — "high-pitched deep voice"
- Overly long descriptions (>500 chars)

## Save Voice for Reuse

Each VoiceDesign call generates a unique voice. To reuse the same voice consistently, save it with [`voices.save()`](./voices.md). The saved voice ID can then be used with [`/v1/audio/speech`](./speech.md).

**Node.js SDK**

```typescript
const inputText = 'This is the reference audio for my saved voice.';

const wav = await client.voices.design({
  input: inputText,
  voice_description: 'A confident male tech presenter, mid-30s, American',
});

const saved = await client.voices.save({
  name: 'Tech Presenter',
  description: 'Confident male, mid-30s, American, for product demos',
  audio: wav,
  ref_text: inputText,
  language: 'English',
});

console.log(`Saved as ${saved.id} — use this ID in future requests`);
```

**Python SDK**

```python
text = "This is my reference audio for voice saving."
description = "A calm, professional female voice with an American accent"

wav = client.voices.design(
    input=text,
    voice_description=description,
    language="English",
)

saved = client.voices.save(
    name="Professional Female",
    audio=wav,
    description=description,
    ref_text=text,
    language="English",
)

print(f"Saved voice ID: {saved.id}")
```

Saved voice limits by plan: Free (3), Starter (10), Pro (25), Realtime (50), Scale (100)

## Error Responses

| Status | Error | Description |
|--------|-------|-------------|
| 400 | Bad Request | Missing text or voice_description, text too long (>4096), description too long (>500) |
| 401 | Unauthorized | Missing or invalid API key |
| 429 | Quota Exceeded | Character quota or voice design quota exceeded for this billing period |

See [Error Reference](./errors.md) for all error codes.

## See Also

- [Saved Voices API](./speech.md) — Use saved voices for consistent output
- [SSE Streaming](./streaming.md) — Deep dive into streaming integration
- [Audio Formats](./audio-formats.md) — PCM decoding and WAV conversion
