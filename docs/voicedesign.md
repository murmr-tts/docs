# Voice Design

Voice Design lets you describe any voice in natural language and generate speech in a single request. No pre-recorded samples, no voice IDs -- just describe what you want and murmr creates it.

## Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/voices/design` | POST | SSE streaming -- returns audio chunks via Server-Sent Events |
| `/v1/voices/design/stream` | POST | Alias for the above (explicit stream path) |

Both endpoints return Server-Sent Events with PCM audio chunks. The `/stream` suffix is optional -- it exists for consistency with the Saved Voices API which has separate batch and streaming endpoints.

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `text` | string | Yes | -- | Text to synthesize. Max 4,096 characters. |
| `voice_description` | string | Yes | -- | Natural language description of the desired voice. Max 500 characters. |
| `language` | string | No | `Auto` | Language name. SDKs default to `English`; raw API defaults to `Auto`. See [Languages](./languages.md). |
| `input` | string | -- | -- | Alias for `text` (OpenAI API compatibility). |

> **Note:** The `instruct` parameter is **not** available for Voice Design. It only works with saved voices via `/v1/audio/speech` -- see the [Speech guide](./speech.md) for details.

## How It Works

Voice Design always streams via SSE. Both `/v1/voices/design` and `/v1/voices/design/stream` return audio chunks via Server-Sent Events as they are generated. ~450ms to first chunk.

## Streaming Examples

**curl**
```bash
curl -X POST "https://api.murmr.dev/v1/voices/design" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Welcome to our podcast. Today we explore the future of AI.",
    "voice_description": "A warm, professional female voice, 35 years old, calm and confident",
    "language": "English"
  }'
```

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

const stream = await client.voices.designStream({
  input: 'Streaming voice design delivers faster time to first audio.',
  voice_description: 'A warm, friendly female voice with natural inflection',
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

**Python (sync)**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    with client.voices.design_stream(
        input="Streaming voice design delivers audio with minimal latency.",
        voice_description="A deep, authoritative male narrator",
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
            input="Streaming voice design delivers audio with minimal latency.",
            voice_description="A deep, authoritative male narrator",
            language="English",
        ) as stream:
            async for chunk in stream:
                pcm_data = chunk.audio_bytes

asyncio.run(main())
```

## SDK Convenience Methods (Complete WAV)

The SDK's `design()` / `voices.design()` method streams internally, collects all chunks, and returns a complete WAV buffer.

**TypeScript**
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

**Python**
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

**Audio chunk:**
```
data: {"chunk":"<base64 PCM>","chunk_index":0,"sample_rate":24000,"format":"pcm_s16le","first_chunk_latency_ms":450}
```

**Completion event:**
```
data: {"done":true,"total_chunks":5,"total_time_ms":2500,"first_chunk_latency_ms":450,"sample_rate":24000}
```

See the [Streaming guide](./streaming.md) for the full field reference and browser playback integration.

## Voice Description Best Practices

Good voice descriptions are specific about tone, pace, gender, age, and accent.

### Good Descriptions

| Description | Why It Works |
|-------------|-------------|
| "A deep, resonant male voice with a slow, deliberate pace and a slight Southern drawl" | Specific about pitch, pace, and accent |
| "A young woman with a bright, energetic tone, speaking quickly with a London accent" | Covers age, energy, speed, and locale |
| "A calm, authoritative male narrator in his 50s, like a documentary voiceover" | Uses relatable reference for style |
| "A warm grandmother reading a bedtime story, soft and gentle with pauses" | Evokes a specific emotional quality |

### Descriptions to Avoid

| Description | Problem |
|-------------|---------|
| "A nice voice" | Too vague -- no actionable characteristics |
| "Make it sound professional" | "Professional" is subjective and underspecified |
| "Voice #3 from the other API" | References external systems murmr cannot interpret |

### Tips

- **Be specific about gender, age, and accent.** "A 30-year-old British woman" is better than "a female voice."
- **Describe the voice, not the emotion.** "A deep, gravelly baritone" gives better results than "an angry voice."
- **Use familiar archetypes.** "Like a late-night radio host" conveys tone, pace, and register effectively.
- **Keep it under 200 characters.** The API accepts up to 500, but concise descriptions produce more consistent results.
- **Avoid celebrity references.** "Like Morgan Freeman" does not work.

## Saving a Designed Voice

Each Voice Design call generates a unique voice. To reuse the same voice, save it:

**TypeScript**
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

console.log(`Saved as ${saved.id} -- use this ID in future requests`);
```

**Python**
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

Once saved, use the voice ID with `/v1/audio/speech` for consistent output. See the [Voices guide](./voices.md) for the full voice management workflow.

## See Also

- [Speech Generation](./speech.md) -- Generate with saved voices
- [Voices](./voices.md) -- Save, list, and delete voices
- [Streaming](./streaming.md) -- SSE format and playback
- [Languages](./languages.md) -- Supported languages and cross-lingual synthesis
