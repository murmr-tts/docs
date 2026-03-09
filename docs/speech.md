# Saved Voices API

Generate speech using voices you've saved from VoiceDesign. Guaranteed consistency across all your content.

> **ℹ️ Creating Saved Voices**
> First, create a voice with [VoiceDesign](./voicedesign.md), then save it using [POST /api/v1/voices](./voices.md). Use the returned voice ID (e.g., `voice_abc123`) with this endpoint.

## Endpoints

`POST /v1/audio/speech`

Batch endpoint — returns [202 with job ID](./async-jobs.md), poll for audio

`POST /v1/audio/speech/stream`

Streaming endpoint — low-latency SSE chunks

> **⚠️ Different response models**
> - **Batch** (`/v1/audio/speech`): Always returns `202 Accepted` with a job ID. Poll `GET /v1/jobs/{jobId}` to retrieve the audio. Supports `response_format` and `webhook_url`.
> - **Streaming** (`/v1/audio/speech/stream`): Returns SSE with base64 PCM chunks for real-time playback.

> **💡 Building a real-time app?**
> The batch endpoint is optimized for bulk generation and file exports, not low-latency playback. If you need audio as fast as possible (voice agents, live previews, interactive apps), use [`/v1/audio/speech/stream`](./streaming.md) instead — it delivers first audio in ~450ms.

## Request Parameters

Send a JSON body with the following parameters:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `text` | string | Yes | -- | The text to synthesize. Maximum 4,096 characters. |
| `voice` | string | No | -- | Saved voice ID (e.g., "voice_abc123"). The API resolves this to the stored voice embeddings automatically. Either voice or voice_clone_prompt is required. |
| `voice_clone_prompt` | string | No | -- | Base64-encoded voice embedding data. Optional — only needed if you store and manage embeddings yourself. Most users should use voice instead. |
| `language` | string | No | Auto | Output language: English, Spanish, Portuguese, German, French, Italian, Chinese, Japanese, Korean, Russian, or "Auto" |
| `response_format` | string | No | wav | Audio format (batch endpoint only): mp3, opus, aac, flac, wav (default), or pcm. See Audio Formats. |
| `webhook_url` | string | No | -- | HTTPS URL for async delivery (batch endpoint only). When provided, completed audio is POSTed to this URL. See Async Jobs. |
| `input` | string | No | -- | Alias for text (OpenAI API compatibility). If both are provided, text takes precedence. |

> **💡 Format your text for better prosody**
> Newlines in your input text act as pause cues — `\n` creates a sentence-level breath pause, `\n\n` creates a paragraph-level pause. See the [Text Formatting Guide](./text-formatting.md) for best practices on natural-sounding output.

## Batch Example

The batch endpoint returns a `202 Accepted` response with a job ID. Poll for the audio result.

**cURL**

```curl
# Submit batch job
curl -X POST "https://api.murmr.dev/v1/audio/speech" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Welcome back to episode 2 of our podcast.",
    "voice": "voice_abc123",
    "response_format": "mp3"
  }'
# Returns: {"id":"job_a1b2c3d4e5f6g7h8","status":"queued","created_at":"..."}

# Poll for result (returns binary audio when complete)
curl "https://api.murmr.dev/v1/jobs/job_a1b2c3d4e5f6g7h8" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --output episode2.mp3
```

**Python**

```python
import time
import requests

# Submit batch job
response = requests.post(
    "https://api.murmr.dev/v1/audio/speech",
    headers={"Authorization": "Bearer YOUR_API_KEY"},
    json={
        "text": "Welcome back to episode 2 of our podcast.",
        "voice": "voice_abc123",
        "response_format": "mp3"
    }
)

job = response.json()
job_id = job["id"]
print(f"Submitted: {job_id}")

# Poll until complete
while True:
    status = requests.get(
        f"https://api.murmr.dev/v1/jobs/{job_id}",
        headers={"Authorization": "Bearer YOUR_API_KEY"}
    )

    # Completed: returns binary audio
    if status.headers.get("Content-Type", "").startswith("audio/"):
        with open("episode2.mp3", "wb") as f:
            f.write(status.content)
        print("Audio saved!")
        break

    # Still in progress or failed
    data = status.json()
    if data["status"] == "failed":
        print(f"Failed: {data.get('error')}")
        break

    print(f"Status: {data['status']}...")
    time.sleep(3)
```

**JavaScript**

```javascript
// Submit batch job
const submitResp = await fetch("https://api.murmr.dev/v1/audio/speech", {
  method: "POST",
  headers: {
    "Authorization": "Bearer YOUR_API_KEY",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    text: "Welcome back to episode 2 of our podcast.",
    voice: "voice_abc123",
    response_format: "mp3"
  })
});
const job = await submitResp.json();

// Poll until complete
async function pollJob(jobId) {
  while (true) {
    const resp = await fetch(`https://api.murmr.dev/v1/jobs/${jobId}`, {
      headers: { "Authorization": "Bearer YOUR_API_KEY" }
    });

    // Completed: returns binary audio
    const ct = resp.headers.get("Content-Type") || "";
    if (ct.startsWith("audio/")) {
      return await resp.blob();
    }

    const data = await resp.json();
    if (data.status === "failed") throw new Error(data.error);

    await new Promise(r => setTimeout(r, 3000));
  }
}

const audioBlob = await pollJob(job.id);
const audio = new Audio(URL.createObjectURL(audioBlob));
audio.play();
```

## Streaming Example

For low-latency playback, use the streaming endpoint. Audio arrives as Server-Sent Events with base64 PCM chunks.

**cURL**

```curl
curl -X POST "https://api.murmr.dev/v1/audio/speech/stream" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Hello, this is streaming audio.",
    "voice": "voice_abc123"
  }'
```

**JavaScript**

```javascript
const response = await fetch("https://api.murmr.dev/v1/audio/speech/stream", {
  method: "POST",
  headers: {
    "Authorization": "Bearer YOUR_API_KEY",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    text: "Hello, this is streaming audio.",
    voice: "voice_abc123"
  })
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  const chunk = decoder.decode(value);
  for (const line of chunk.split("\n")) {
    if (line.startsWith("data: ")) {
      const data = JSON.parse(line.slice(6));
      if (data.chunk) {
        // base64 PCM int16 at 24kHz mono
        console.log(`Chunk ${data.chunk_index}`);
      }
      if (data.done) {
        console.log(`Complete: ${data.total_chunks} chunks`);
      }
    }
  }
}
```

See [SSE Streaming](./streaming.md) for complete integration guide with Web Audio API playback.

## OpenAI Compatibility

This endpoint is compatible with OpenAI's `/v1/audio/speech` API. The `input` parameter is accepted as an alias for `text`. See the [OpenAI Migration Guide](./openai-migration.md) for details.

## Error Responses

```JSON
{
  "error": "Saved voice not found: voice_abc123"
}
```

| Status | Error | Description |
|--------|-------|-------------|
| 400 | Bad Request | Missing text, text too long (>4096 chars), invalid response_format |
| 401 | Unauthorized | Missing or invalid API key |
| 404 | Not Found | Saved voice not found or doesn't belong to your account |
| 429 | Rate Limit Exceeded | Monthly character quota exceeded, or server at capacity |

See [Error Reference](./errors.md) for complete error handling guidance.

## See Also

- [VoiceDesign API](./voicedesign.md) — Create voices from descriptions
- [Voice Management](./voices.md) — Save, list, delete voices
- [Text Formatting](./text-formatting.md) — Newlines, pauses, and prosody best practices
- [SSE Streaming](./streaming.md) — Low-latency streaming integration
- [Audio Formats](./audio-formats.md) — Supported formats and encoding specs
- [Async Jobs](./async-jobs.md) — Job polling and webhook delivery
