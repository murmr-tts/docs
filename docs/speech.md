# Speech Generation

Generate speech from text using a saved voice. murmr provides two speech endpoints: a batch endpoint that returns complete audio files, and a streaming endpoint that delivers audio chunks via Server-Sent Events (SSE) for low-latency playback.

## Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/audio/speech` | POST | Batch generation -- returns audio (200) by default, or 202 with job ID when `webhook_url` is set |
| `/v1/audio/speech/stream` | POST | Streaming generation -- low-latency SSE chunks |

> First, create a voice with [VoiceDesign](./voicedesign.md), then save it using [POST /v1/voices](./voices.md). Use the returned voice ID (e.g., `voice_abc123`) with these endpoints.

## Request Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `text` | string | Yes | -- | Text to synthesize. Maximum 4,096 characters. |
| `voice` | string | Yes* | -- | Saved voice ID (e.g., `voice_abc123`). |
| `voice_clone_prompt` | string | No | -- | Base64-encoded embedding data. Overrides `voice` if both provided. |
| `language` | string | No | `Auto` | Language name. SDKs default to `English`; raw API defaults to `Auto`. See [Languages](./languages.md). |
| `instruct` | string | No | -- | Prosody/style instruction for the saved voice (e.g., "speak slowly and softly"). Only available for saved voices, not Voice Design. |
| `response_format` | string | No | `wav` | Output format (batch only): `mp3`, `opus`, `aac`, `flac`, `wav`, `pcm`. See [Audio Formats](./audio-formats.md). |
| `webhook_url` | string | No | -- | HTTPS URL for async delivery (batch only). Returns 202 with job ID. |
| `input` | string | -- | -- | Alias for `text` (OpenAI API compatibility). If both are provided, `text` takes precedence. |

> *Either `voice` or `voice_clone_prompt` is required. If you pass `voice_clone_prompt`, the `voice` parameter is ignored.

> Streaming always returns raw PCM audio (24kHz, 16-bit, mono). The `response_format` parameter is not supported for streaming.

## Batch Generation

The batch endpoint (`POST /v1/audio/speech`) returns HTTP 200 with binary audio by default. If you provide a `webhook_url`, it returns HTTP 202 with a job ID instead -- see [Async Jobs](./async-jobs.md) for polling and webhook details.

**curl**
```bash
# Submit batch job
curl -X POST "https://api.murmr.dev/v1/audio/speech" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Welcome back to episode 2 of our podcast.",
    "voice": "voice_abc123",
    "response_format": "mp3"
  }'
# Returns: {"id":"job_a1b2c3d4e5f6g7h8","status":"queued","created_at":"..."}

# Poll for result (returns binary audio when complete)
curl "https://api.murmr.dev/v1/jobs/job_a1b2c3d4e5f6g7h8" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  --output episode2.mp3
```

**TypeScript**
```typescript
import { MurmrClient, isSyncResponse } from '@murmr/sdk';
import { writeFileSync } from 'node:fs';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

// Option 1: Sync -- returns audio directly (default, no webhook_url)
const result = await client.speech.create({
  input: 'Hello from the murmr TTS API.',
  voice: 'voice_abc123',
  response_format: 'mp3',
});

if (isSyncResponse(result)) {
  const buffer = Buffer.from(await result.arrayBuffer());
  writeFileSync('output.mp3', buffer);
}

// Option 2: Poll until done (convenience wrapper)
const waited = await client.speech.createAndWait({
  input: 'Wait for the audio to be ready.',
  voice: 'voice_abc123',
  response_format: 'wav',
  onPoll: (status) => console.log(`Status: ${status.status}`),
});

// Option 3: Async with webhook
const async_result = await client.speech.create({
  input: 'This will be processed asynchronously.',
  voice: 'voice_abc123',
  webhook_url: 'https://yourapp.com/webhooks/tts',
});
```

**Python (sync)**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    result = client.speech.create_and_wait(
        input="Your package has been shipped and will arrive by Friday.",
        voice="voice_abc123def456",
        language="English",
        response_format="mp3",
    )

    with open("notification.mp3", "wb") as f:
        f.write(result.audio)

    print(f"Duration: {result.duration_ms}ms")
```

**Python (async)**
```python
import asyncio
import os
from murmr import AsyncMurmrClient

async def main():
    async with AsyncMurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
        result = await client.speech.create(
            input="Your package has been shipped.",
            voice="voice_abc123def456",
            response_format="wav",
        )

        with open("notification.wav", "wb") as f:
            f.write(result.audio)

asyncio.run(main())
```

## Streaming Generation

The streaming endpoint (`POST /v1/audio/speech/stream`) returns audio chunks via SSE. First chunk latency is typically under 450ms.

**curl**
```bash
curl -X POST "https://api.murmr.dev/v1/audio/speech/stream" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{
    "text": "Stream audio for real-time playback.",
    "voice": "voice_abc123"
  }'
```

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

const stream = await client.speech.stream({
  input: 'Stream audio for real-time playback.',
  voice: 'voice_abc123',
});

for await (const chunk of stream) {
  const audioData = chunk.audio || chunk.chunk;
  if (audioData) {
    const pcm = Buffer.from(audioData, 'base64');
    // Send to audio player, write to file, etc.
  }
  if (chunk.done) {
    console.log(`Done in ${chunk.total_time_ms}ms`);
  }
}
```

**Python (sync)**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    with client.speech.stream(
        input="Real-time audio streaming for interactive applications.",
        voice="voice_abc123def456",
        language="English",
    ) as stream:
        pcm_chunks = []
        for chunk in stream:
            pcm_chunks.append(chunk.audio_bytes)

            if chunk.first_chunk_latency_ms is not None:
                print(f"Time to first chunk: {chunk.first_chunk_latency_ms:.0f}ms")

            if chunk.done:
                print(f"Total time: {chunk.total_time_ms:.0f}ms")
```

**Python (async)**
```python
import asyncio
import os
from murmr import AsyncMurmrClient

async def main():
    async with AsyncMurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
        async with client.speech.stream(
            input="Real-time audio streaming for interactive applications.",
            voice="voice_abc123def456",
        ) as stream:
            async for chunk in stream:
                pcm_data = chunk.audio_bytes
                # Send to audio player or write to buffer

asyncio.run(main())
```

## OpenAI Compatibility

This endpoint is compatible with OpenAI's `/v1/audio/speech` API. The `input` parameter is accepted as an alias for `text`. See the [OpenAI Migration Guide](https://murmr.dev/en/docs/openai) for details.

## Error Codes

| Status | Meaning | Common Causes |
|--------|---------|---------------|
| 400 | Bad Request | Missing `text`, text exceeds 4,096 chars, invalid `response_format` |
| 401 | Unauthorized | Missing or invalid API key |
| 404 | Not Found | Voice ID does not exist or belongs to another user |
| 429 | Rate Limited | Monthly character limit exceeded or too many concurrent requests |

See the [Errors guide](./errors.md) for detailed error handling.

## See Also

- [Voice Design](./voicedesign.md) -- Generate with a voice description instead of a saved voice
- [Streaming](./streaming.md) -- SSE format details and browser playback
- [Voices](./voices.md) -- Save, list, and delete voices
- [Audio Formats](./audio-formats.md) -- Format comparison and conversion
- [Text Formatting](./text-formatting.md) -- Newlines, pauses, and prosody best practices
