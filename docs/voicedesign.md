# VoiceDesign API

Create any voice by describing it in natural language. Generate speech with custom voices — no presets, no limitations.

> **💡 murmr's Key Differentiator**
> VoiceDesign lets you describe any voice in natural language — "A warm, professional female voice, calm and confident" — and generate speech with that voice instantly. Try different descriptions in the [Playground](https://murmr.dev/en/dashboard/playground) before coding.

## Endpoints

`POST /v1/voices/design`

SSE streaming — low-latency audio chunks

`POST /v1/voices/design/stream`

SSE streaming — alias for the above

> **ℹ️ Both endpoints stream**
> Both `/v1/voices/design` and `/v1/voices/design/stream` return Server-Sent Events with PCM audio chunks. The `/stream` suffix is optional — it exists for consistency with the Saved Voices API which has separate batch and streaming endpoints.

## Request Parameters

Send a JSON body with the following parameters:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `text` | string | Yes | -- | The text to synthesize. Maximum 4,096 characters. |
| `voice_description` | string | Yes | -- | Natural language description of the voice (e.g., "A warm, friendly male voice"). Maximum 500 characters. |
| `language` | string | No | Auto | Output language: English, Spanish, Portuguese, German, French, Italian, Chinese, Japanese, Korean, Russian, or "Auto" (detect from text) |
| `input` | string | No | -- | Alias for text (OpenAI API compatibility). If both text and input are provided, text takes precedence. |

## Streaming Request

VoiceDesign always streams audio as Server-Sent Events. Audio chunks arrive progressively for low-latency playback.

**cURL**

```curl
curl -X POST "https://api.murmr.dev/v1/voices/design" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Welcome to our podcast. Today we explore the future of AI.",
    "voice_description": "A warm, professional female voice, 35 years old, calm and confident",
    "language": "English"
  }'
```

**Python**

```python
import requests
import base64
import io
import wave

response = requests.post(
    "https://api.murmr.dev/v1/voices/design",
    headers={
        "Authorization": "Bearer YOUR_API_KEY",
        "Content-Type": "application/json"
    },
    json={
        "text": "Welcome to our podcast. Today we explore the future of AI.",
        "voice_description": "A warm, professional female voice, 35 years old, calm and confident",
        "language": "English"
    },
    stream=True
)

# Collect PCM chunks from SSE stream
pcm_chunks = []
for line in response.iter_lines():
    if line and line.startswith(b"data: "):
        import json
        data = json.loads(line[6:])

        if data.get("chunk"):
            pcm_chunks.append(base64.b64decode(data["chunk"]))

        if data.get("done"):
            print(f"Complete: {data['total_chunks']} chunks, {data['total_time_ms']}ms")

# Save as WAV
all_pcm = b"".join(pcm_chunks)
buf = io.BytesIO()
with wave.open(buf, "wb") as wf:
    wf.setnchannels(1)
    wf.setsampwidth(2)
    wf.setframerate(24000)
    wf.writeframes(all_pcm)

with open("podcast-intro.wav", "wb") as f:
    f.write(buf.getvalue())
print("Saved podcast-intro.wav")
```

**JavaScript**

```javascript
const response = await fetch("https://api.murmr.dev/v1/voices/design", {
  method: "POST",
  headers: {
    "Authorization": "Bearer YOUR_API_KEY",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    text: "Welcome to our podcast. Today we explore the future of AI.",
    voice_description: "A warm, professional female voice, 35 years old, calm and confident",
    language: "English"
  })
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  const chunk = decoder.decode(value);
  const lines = chunk.split('\n');

  for (const line of lines) {
    if (line.startsWith('data: ')) {
      const data = JSON.parse(line.slice(6));

      if (data.chunk) {
        // data.chunk is base64 PCM audio (24kHz, 16-bit, mono)
        console.log(`Chunk ${data.chunk_index}, ${data.sample_rate}Hz`);
      }

      if (data.done) {
        console.log(`Complete: ${data.total_chunks} chunks`);
      }
    }
  }
}
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

Found a voice you love? Save it to ensure consistency across all your content. See the [Voice Management](./voices.md) page for the full API.

```Workflow
1. Generate audio with VoiceDesign
   POST /v1/voices/design → SSE stream with PCM audio

2. Save the voice (send the audio to the Next.js app)
   POST /api/v1/voices with the audio → Get voice_abc123

3. Use saved voice in future requests
   POST /v1/audio/speech with voice: "voice_abc123"
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
