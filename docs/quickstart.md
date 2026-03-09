# Quickstart

Get started with the murmr TTS API in under 5 minutes. Generate your first audio with a natural language voice description, stream audio, save a voice, and reuse it.

## Prerequisites

1. A murmr account -- [sign up free](https://murmr.dev/en/signup)
2. An API key -- create one in your [dashboard](https://murmr.dev/en/dashboard/api-keys)
3. Node.js 18+ (for the TypeScript SDK), Python 3.9+ (for the Python SDK), or any language that supports HTTP

## Install the SDK

**TypeScript**
```bash
npm install @murmr/sdk
# or: pnpm add @murmr/sdk / yarn add @murmr/sdk
```

**Python**
```bash
pip install murmr
```

No SDK? You can use the REST API directly with any HTTP client.

## Set Your API Key

API keys follow the format `murmr_sk_live_xxx`. Set it as an environment variable:

```bash
export MURMR_API_KEY="murmr_sk_live_your_key_here"
```

> Never hardcode API keys. Always use environment variables. See the [Authentication guide](./authentication.md) for best practices.

## Initialize the Client

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});
```

**Python**
```python
import os
from murmr import MurmrClient

client = MurmrClient(api_key=os.environ["MURMR_API_KEY"])
```

## Generate Audio with Voice Design

Describe any voice in natural language and generate speech in a single call. This is murmr's core feature -- no presets, no cloning, just describe what you want.

**curl**
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

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';
import { writeFileSync } from 'node:fs';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

const wav = await client.voices.design({
  input: 'Welcome to our app. Let me show you around.',
  voice_description: 'A warm, friendly female voice, calm and clear',
  language: 'English',
});

writeFileSync('welcome.wav', wav);
console.log('Audio saved to welcome.wav');
```

**Python**
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
    print("Audio saved to welcome.wav")
```

## Stream Audio (Low Latency)

For real-time playback, use the streaming endpoint. Audio chunks arrive via Server-Sent Events as they are generated. First chunk typically arrives in ~450ms.

**curl**
```bash
curl -X POST "https://api.murmr.dev/v1/voices/design/stream" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Hello! This audio is streaming to you in real time.",
    "voice_description": "A confident male narrator voice",
    "language": "English"
  }'
```

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';
import { createWriteStream } from 'node:fs';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

const stream = await client.voices.designStream({
  input: 'Streaming delivers audio with minimal latency.',
  voice_description: 'A calm female voice, clear and articulate',
});

const file = createWriteStream('streamed.pcm');

for await (const chunk of stream) {
  const audioData = chunk.audio || chunk.chunk;
  if (audioData) {
    file.write(Buffer.from(audioData, 'base64'));
  }
  if (chunk.done) {
    console.log(`Stream complete in ${chunk.total_time_ms}ms`);
  }
}

file.end();
```

**Python**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    with client.voices.design_stream(
        input="Streaming delivers audio with minimal latency.",
        voice_description="A calm female voice, clear and articulate",
        language="English",
    ) as stream:
        for chunk in stream:
            pcm_data = chunk.audio_bytes
            print(f"Chunk {chunk.chunk_index}: {len(pcm_data)} bytes")
```

> Streamed audio is raw PCM (24kHz, 16-bit, mono). See the [Streaming guide](./streaming.md) for details on collecting chunks into a WAV file.

## Save a Voice for Reuse

Found a voice you like? Save it so every request sounds the same. Saved voices use voice embeddings -- they are fast and consistent.

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

// Step 1: Generate audio with voice design
const inputText = 'This is my reference audio for saving a voice.';
const wav = await client.voices.design({
  input: inputText,
  voice_description: 'A friendly, upbeat female voice',
});

// Step 2: Save the voice
const saved = await client.voices.save({
  name: 'Friendly Female',
  description: 'A friendly, upbeat female voice for product demos',
  audio: wav,
  ref_text: inputText,
  language: 'English',
});

console.log(`Voice saved: ${saved.id}`);
// Output: Voice saved: voice_a1b2c3d4e5f6
```

**Python**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    # Step 1: Generate audio with a voice description
    text = "This is a sample of my voice for saving."
    wav = client.voices.design(
        input=text,
        voice_description="A calm, professional male voice",
        language="English",
    )

    # Step 2: Save the voice
    saved = client.voices.save(
        name="Professional Narrator",
        audio=wav,
        description="A calm, professional male voice",
        ref_text=text,
        language="English",
    )

    print(f"Voice saved: {saved.id}")
    # Output: Voice saved: voice_abc123def456
```

## Generate with a Saved Voice

Use the saved voice ID for consistent, repeatable speech generation.

**curl**
```bash
curl -X POST "https://api.murmr.dev/v1/audio/speech" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input": "This is my saved voice. It sounds the same every time.",
    "voice": "voice_abc123"
  }' \
  --output output.wav
```

**TypeScript**
```typescript
import { MurmrClient, isSyncResponse } from '@murmr/sdk';
import { writeFileSync } from 'node:fs';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

// Batch generation (returns complete audio)
const result = await client.speech.create({
  input: 'Every call with this voice ID sounds the same.',
  voice: 'voice_a1b2c3d4e5f6',
});

if (isSyncResponse(result)) {
  const buffer = Buffer.from(await result.arrayBuffer());
  writeFileSync('consistent.wav', buffer);
}

// Or stream for lower latency
const stream = await client.speech.stream({
  input: 'Streaming with a saved voice is just as easy.',
  voice: 'voice_a1b2c3d4e5f6',
});

for await (const chunk of stream) {
  if (chunk.audio || chunk.chunk) {
    // Process PCM audio chunk
  }
}
```

**Python**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    # Batch generation (returns complete audio)
    result = client.speech.create_and_wait(
        input="Your order has been confirmed and will arrive tomorrow.",
        voice="voice_abc123def456",
        language="English",
        response_format="mp3",
    )

    with open("confirmation.mp3", "wb") as f:
        f.write(result.audio)

    # Streaming generation (low latency)
    with client.speech.stream(
        input="Your order has been confirmed and will arrive tomorrow.",
        voice="voice_abc123def456",
        language="English",
    ) as stream:
        for chunk in stream:
            pcm_data = chunk.audio_bytes
            # Process audio chunks as they arrive
```

## Next Steps

| Topic | Description |
|-------|-------------|
| [Authentication](./authentication.md) | API key management and security |
| [Voice Design](./voicedesign.md) | Advanced voice description techniques |
| [Speech Generation](./speech.md) | Saved voice generation deep dive |
| [Streaming](./streaming.md) | SSE streaming deep dive |
| [Voices](./voices.md) | Save, list, and delete voices |
| [Audio Formats](./audio-formats.md) | WAV, MP3, Opus, and more |
| [Error Handling](./errors.md) | Robust error handling patterns |
