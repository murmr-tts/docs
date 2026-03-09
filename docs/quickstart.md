# Quickstart

Generate your first audio in under 5 minutes. This guide covers installation, authentication, and making your first API call.

## Prerequisites

1. A murmr account — [sign up free](https://murmr.dev/en/signup)
2. An API key — create one in your [dashboard](https://murmr.dev/en/dashboard/api-keys)
3. Node.js 18+ (for the SDK) or any language that supports HTTP

## 1. Install the SDK

**npm**

```bash
npm install @murmr/sdk
```

**pnpm**

```bash
pnpm add @murmr/sdk
```

**yarn**

```bash
yarn add @murmr/sdk
```

No SDK? You can use the [REST API directly](./voicedesign.md) with any HTTP client.

## 2. Initialize the Client

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
import requests

API_KEY = os.environ["MURMR_API_KEY"]
BASE_URL = "https://api.murmr.dev"
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}
```

> **⚠️ Keep your API key secret**
> Never expose your API key in client-side code or commit it to version control. Use environment variables.

## 3. Design a Voice and Generate Audio

Describe any voice in natural language and generate audio in a single call. This is murmr's core feature — no presets, no cloning, just describe what you want.

**TypeScript**

```typescript
import { writeFileSync } from 'fs';

const audio = await client.voices.design({
  input: "Welcome to our app. Let me show you around.",
  voice_description: "A warm, friendly female voice, calm and clear",
  language: "English",
});

writeFileSync("welcome.wav", Buffer.from(audio));
console.log("Audio saved to welcome.wav");
```

**Python**

```python
response = requests.post(
    f"{BASE_URL}/v1/voices/design",
    headers=HEADERS,
    json={
        "text": "Welcome to our app. Let me show you around.",
        "voice_description": "A warm, friendly female voice, calm and clear",
        "language": "English",
    },
)

with open("welcome.wav", "wb") as f:
    f.write(response.content)
print("Audio saved to welcome.wav")
```

**cURL**

```curl
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

## 4. Stream Audio (Low Latency)

For real-time playback, use the streaming endpoint. Audio chunks arrive via Server-Sent Events as they're generated — no waiting for the full file.

**TypeScript**

```typescript
const response = await fetch("https://api.murmr.dev/v1/voices/design/stream", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${process.env.MURMR_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    text: "Hello! This audio is streaming to you in real time.",
    voice_description: "A confident male narrator voice",
    language: "English",
  }),
});

const reader = response.body!.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  for (const line of decoder.decode(value).split("\n")) {
    if (!line.startsWith("data: ")) continue;
    const data = JSON.parse(line.slice(6));

    if (data.chunk) {
      // Base64-encoded PCM audio chunk (24kHz, 16-bit, mono)
      const pcm = Buffer.from(data.chunk, "base64");
      console.log(`Received ${pcm.length} bytes`);
    }

    if (data.done) {
      console.log(`Complete in ${data.total_time_ms}ms`);
      console.log(`Time to first chunk: ${data.first_chunk_latency_ms}ms`);
    }
  }
}
```

**cURL**

```curl
curl -X POST "https://api.murmr.dev/v1/voices/design/stream" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Hello! This audio is streaming to you in real time.",
    "voice_description": "A confident male narrator voice",
    "language": "English"
  }'
```

> **💡 ~450ms time-to-first-chunk**
> Streaming starts delivering audio in ~450ms. See the [SSE Streaming guide](./streaming.md) for browser playback with Web Audio API.

## 5. Save a Voice for Reuse

Found a voice you like? Save it so every request sounds the same. Saved voices use voice embeddings — they're fast and consistent.

**TypeScript**

```typescript
// Step 1: Design a voice and get the audio
const audio = await client.voices.design({
  input: "Testing this voice for my project.",
  voice_description: "A calm, professional narrator",
});

// Step 2: Save it via REST API
const response = await fetch("https://murmr.dev/api/v1/voices", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${process.env.MURMR_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    name: "Project Narrator",
    description: "Calm, professional narrator for product videos",
    audio: Buffer.from(audio).toString("base64"),
    language: "English",
  }),
});

const voice = await response.json();
console.log(`Saved as ${voice.id}`);
// e.g. "voice_abc123"
```

**Python**

```python
import base64

# Step 1: Design a voice and get the audio
response = requests.post(
    f"{BASE_URL}/v1/voices/design",
    headers=HEADERS,
    json={
        "text": "Testing this voice for my project.",
        "voice_description": "A calm, professional narrator",
    },
)
audio = response.content

# Step 2: Save it
save_response = requests.post(
    f"{BASE_URL}/api/v1/voices",
    headers=HEADERS,
    json={
        "name": "Project Narrator",
        "description": "Calm, professional narrator for product videos",
        "audio": base64.b64encode(audio).decode(),
        "language": "English",
    },
)
voice_id = save_response.json()["id"]
print(f"Saved as {voice_id}")
```

## 6. Generate with a Saved Voice

Once saved, use the voice ID in any speech request. Every call with the same voice ID produces the same voice.

**TypeScript**

```typescript
import { writeFileSync } from 'fs';

const result = await client.speech.createAndWait({
  input: "This is my saved voice. It sounds the same every time.",
  voice: "voice_abc123", // Your saved voice ID
});

writeFileSync("output.wav", Buffer.from(result.audio_base64!, "base64"));
```

**cURL**

```curl
curl -X POST "https://api.murmr.dev/v1/audio/speech" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input": "This is my saved voice. It sounds the same every time.",
    "voice": "voice_abc123"
  }' \
  --output output.wav
```

## Next Steps

- [Node.js SDK Reference](./sdk-reference-node.md) — Full API reference with long-form audio, async jobs, and error handling
- [VoiceDesign API](./voicedesign.md) — Detailed endpoint reference with all parameters
- [Crafting Voices](./voice-crafting.md) — Tips for writing effective voice descriptions
- [Real-time WebSocket](./realtime.md) — Build voice agents with ~460ms latency
- [OpenAI Migration](./openai-migration.md) — Switching from OpenAI TTS? Start here
