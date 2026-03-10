# OpenAI Compatibility

murmr is designed as a drop-in replacement for the OpenAI TTS API. Migrate your existing code with minimal changes and unlock powerful new features.

## Quick Migration

The fastest migration path uses VoiceDesign — describe any voice in natural language instead of choosing from preset names:

**cURL**

```curl
curl -X POST "https://api.murmr.dev/v1/voices/design" \
  -H "Authorization: Bearer YOUR_MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Hello, world!",
    "voice_description": "A warm, professional female voice",
    "language": "English"
  }' --output output.wav
```

**Node.js SDK**

```typescript
import { MurmrClient } from '@murmr/sdk';
import { writeFileSync } from 'fs';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

const wav = await client.voices.design({
  input: 'Hello, world!',
  voice_description: 'A warm, professional female voice',
});

writeFileSync('output.wav', wav);
```

**Python SDK**

```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    wav = client.voices.design(
        input="Hello, world!",
        voice_description="A warm, professional female voice",
        language="English",
    )

    with open("output.wav", "wb") as f:
        f.write(wav)
```

> **💡 OpenAI Voice Equivalents**
> Instead of mapping OpenAI voices 1:1, describe what you want:
> - OpenAI `alloy` → "A neutral, clear voice for general use"
> - OpenAI `echo` → "A warm, resonant male voice"
> - OpenAI `nova` → "A bright, energetic female voice"
> - OpenAI `shimmer` → "A playful, upbeat voice"

## Voice Strategy

Unlike OpenAI's fixed voices, murmr uses **VoiceDesign** — describe any voice in natural language and generate speech with it. Here's the recommended migration approach:

1. **Create voices with VoiceDesign** — Use the [Playground](https://murmr.dev/en/dashboard/playground) or [VoiceDesign API](./voicedesign.md) to describe the voice you want: "A warm, professional female voice, calm and clear"

2. **Save voices you like** — Found a voice you love? Save it via the [Voices API](./voices.md) to get a stable ID like `voice_abc123`

3. **Use saved voices in production** — Use your saved voice IDs with the batch or streaming endpoints for consistent, repeatable output.

## Key Differences

- **Same — Endpoint structure & auth:** `/v1/audio/speech` with Bearer token in Authorization header — same as OpenAI.

- **Better — Voice flexibility:** Instead of 6 fixed voices, create any voice with VoiceDesign. Describe exactly what you need.

- **Better — Streaming support:** SSE streaming for low-latency playback (~450ms TTFC), or batch mode for complete audio files.

- **Extra — Response format options:** Batch endpoint supports `response_format`: `mp3`, `opus`, `aac`, `flac`, `wav` (default), `pcm`

- **Extra — WebSocket real-time:** WebSocket API for voice agents and LLM integration. See [Real-time docs](./realtime.md).

- **Same — Sync batch:** `/v1/audio/speech` returns `200` with binary audio by default — same as OpenAI. For async processing, pass `webhook_url` to get a `202` with a job ID. For low-latency streaming, use [`/v1/audio/speech/stream`](./streaming.md) (~450ms to first audio).

- **Different — Language parameter:** murmr adds a `language` parameter using full names (`"English"`, `"French"`, `"Japanese"`). Defaults to `"Auto"` for automatic detection. See [Language Support](./languages.md).

> **ℹ️ OpenAI SDK Compatibility**
> `/v1/audio/speech` now returns `200` with binary audio by default — the OpenAI SDK's `client.audio.speech.create()` works as a drop-in (with a saved voice ID as the `voice` parameter).

## After: murmr with Saved Voice

Once you've saved a voice, use it with the streaming endpoint for real-time playback or the batch endpoint for bulk generation:

### Streaming (recommended for real-time)

**cURL**

```curl
curl -X POST "https://api.murmr.dev/v1/audio/speech/stream" \
  -H "Authorization: Bearer YOUR_MURMR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Hello!",
    "voice": "voice_abc123",
    "language": "English"
  }'
# → SSE stream with base64 PCM chunks (~450ms to first audio)
```

**Node.js SDK**

```typescript
import { MurmrClient } from '@murmr/sdk';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

const stream = await client.speech.stream({
  input: 'Hello, this is a test.',
  voice: 'voice_abc123',
});

for await (const chunk of stream) {
  if (chunk.audio) {
    const pcm = Buffer.from(chunk.audio, 'base64');
    // Send to audio player, write to file, etc.
  }
}
```

**Python SDK**

```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    with client.speech.stream(
        input="Hello, this is a test.",
        voice="voice_abc123",
        language="English",
    ) as stream:
        for chunk in stream:
            pcm = chunk.audio_bytes
            # Process PCM audio data
```

### Batch (for bulk generation)

**cURL**

```curl
# Returns 200 with binary audio
curl -X POST "https://api.murmr.dev/v1/audio/speech" \
  -H "Authorization: Bearer YOUR_MURMR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Hello!",
    "voice": "voice_abc123",
    "response_format": "mp3"
  }' --output hello.mp3
```

**Node.js SDK**

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

**Python SDK**

```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    result = client.speech.create_and_wait(
        input="Hello, this is a test.",
        voice="voice_abc123",
        response_format="mp3",
    )

    with open("output.mp3", "wb") as f:
        f.write(result.audio)
```

Both `text` and `input` field names are accepted for OpenAI compatibility.

## Voice Mapping

Create a replacement voice for each OpenAI voice you use. This is a one-time setup — save the voice ID and use it in production:

**Node.js SDK**

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

**Python SDK**

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

> **ℹ️ Pricing**
> murmr offers competitive pricing compared to OpenAI TTS, especially at higher volumes. Check the [Pricing page](./pricing.md) for details.
