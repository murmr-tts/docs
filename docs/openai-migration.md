# OpenAI Compatibility

murmr is designed as a drop-in replacement for the OpenAI TTS API. Migrate your existing code with minimal changes and unlock powerful new features.

## Quick Migration

The fastest migration path uses VoiceDesign — describe any voice in natural language instead of choosing from preset names:

**Python**

```python
import requests

# murmr VoiceDesign — describe any voice you want
response = requests.post(
    "https://api.murmr.dev/v1/voices/design",
    headers={"Authorization": "Bearer YOUR_MURMR_API_KEY"},
    json={
        "text": "Hello, world!",
        "voice_description": "A warm, professional female voice",
        "language": "English"
    }
)

with open("output.wav", "wb") as f:
    f.write(response.content)
```

**JavaScript/Node**

```javascript
// murmr VoiceDesign — describe any voice you want
const response = await fetch("https://api.murmr.dev/v1/voices/design", {
  method: "POST",
  headers: {
    "Authorization": "Bearer YOUR_MURMR_API_KEY",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    text: "Hello, world!",
    voice_description: "A warm, professional female voice",
    language: "English"
  })
});

const buffer = Buffer.from(await response.arrayBuffer());
fs.writeFileSync("output.wav", buffer);
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

- **Different — Batch is async:** The batch endpoint (`/v1/audio/speech`) returns `202` with a job ID. Poll `GET /v1/jobs/{jobId}` to retrieve the audio when ready. This endpoint is optimized for bulk generation and file exports — for low-latency playback, use [`/v1/audio/speech/stream`](./streaming.md) instead (~450ms to first audio).

- **Different — Language parameter:** murmr adds a `language` parameter using full names (`"English"`, `"French"`, `"Japanese"`). Defaults to `"Auto"` for automatic detection. See [Language Support](./languages.md).

> **⚠️ OpenAI SDK Compatibility**
> The OpenAI SDK's `client.audio.speech.create()` targets `/v1/audio/speech`, which is murmr's async batch endpoint (returns `202` JSON, not audio). For direct SDK compatibility, use the [VoiceDesign](./voicedesign.md) or [Streaming](./streaming.md) endpoints instead, or use the REST API directly as shown below.

## REST API Migration

If you're using the REST API directly, here's how the requests compare:

OpenAI:

```cURL
curl https://api.openai.com/v1/audio/speech \
  -H "Authorization: Bearer sk-..." \
  -H "Content-Type: application/json" \
  -d '{"model": "tts-1", "voice": "alloy", "input": "Hello!"}'
```

murmr (VoiceDesign — returns audio directly):

```cURL
curl -X POST "https://api.murmr.dev/v1/voices/design" \
  -H "Authorization: Bearer YOUR_MURMR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Hello!",
    "voice_description": "A neutral, clear voice",
    "language": "English"
  }' --output hello.wav
```

murmr (saved voice — streaming, recommended for real-time):

```cURL
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

murmr (saved voice — async batch, for bulk generation):

```cURL
# Submit job (returns 202 with job ID)
curl -X POST "https://api.murmr.dev/v1/audio/speech" \
  -H "Authorization: Bearer YOUR_MURMR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Hello!",
    "voice": "voice_abc123",
    "language": "English",
    "response_format": "mp3"
  }'
# → {"id": "job_abc123", "status": "queued", "created_at": "..."}

# Poll for result
curl "https://api.murmr.dev/v1/jobs/job_abc123" \
  -H "Authorization: Bearer YOUR_MURMR_KEY"
```

Both `text` and `input` field names are accepted for OpenAI compatibility.

## Using murmr Extras

After migrating, take advantage of murmr-specific features:

```Python
import requests

# Explicit language control
response = requests.post(
    "https://api.murmr.dev/v1/voices/design",
    headers={"Authorization": "Bearer YOUR_MURMR_KEY"},
    json={
        "text": "Bonjour, comment allez-vous?",
        "voice_description": "A warm, friendly French woman",
        "language": "French"
    }
)

# Streaming for low-latency playback (~450ms TTFC)
response = requests.post(
    "https://api.murmr.dev/v1/voices/design/stream",
    headers={"Authorization": "Bearer YOUR_MURMR_KEY"},
    json={
        "text": "Welcome to the future of voice.",
        "voice_description": "An enthusiastic narrator, building anticipation",
        "language": "English"
    },
    stream=True
)
for line in response.iter_lines():
    # SSE chunks with base64 PCM audio
    pass
```

## Direct VoiceDesign

For one-off generations or experimentation, use VoiceDesign directly without saving:

```cURL
curl -X POST "https://api.murmr.dev/v1/voices/design" \
  -H "Authorization: Bearer YOUR_MURMR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Welcome to murmr. Creating voices has never been easier.",
    "voice_description": "A warm, professional female voice, 35 years old, calm and clear",
    "language": "English"
  }' --output welcome.wav
```

See [VoiceDesign API](./voicedesign.md) for streaming options and full parameter reference.

> **ℹ️ Pricing**
> murmr offers competitive pricing compared to OpenAI TTS, especially at higher volumes. Check the [Pricing page](./pricing.md) for details.
