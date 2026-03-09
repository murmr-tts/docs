# SSE Streaming

murmr delivers audio via Server-Sent Events (SSE) for low-latency playback. First chunk latency is typically under 450ms. This guide covers the SSE format, streaming endpoints, browser playback, and Node.js/Python usage patterns.

## Streaming Endpoints

| Endpoint | Mode | Description |
|----------|------|-------------|
| `/v1/voices/design` | VoiceDesign | SSE streaming (always streams) |
| `/v1/voices/design/stream` | VoiceDesign | Alias for the above |
| `/v1/audio/speech/stream` | Saved Voice | Stream with a saved voice ID |

All streaming endpoints return `text/event-stream` responses with base64-encoded PCM audio chunks.

## When to Use SSE vs WebSocket

| SSE (this guide) | WebSocket (`/v1/realtime`) |
|------------------|---------------------------|
| Landing page demos | Voice agents / chatbots |
| Dashboard previews | LLM streaming integration |
| Mobile apps (simple integration) | Bidirectional communication |
| Any scenario where you want fast first audio | Text buffering requirements |

## Audio Specifications

| Property | Value |
|----------|-------|
| Sample rate | 24,000 Hz |
| Bit depth | 16-bit |
| Channels | Mono (1) |
| Encoding | PCM signed 16-bit little-endian (`pcm_s16le`) |
| Chunk encoding | Base64 |
| Chunk size | ~4,800 bytes (~100ms of audio) |

## SSE Event Format

Each SSE event is a `data:` line containing a JSON object.

### Audio Chunk Event

```
data: {"chunk":"<base64 PCM data>","chunk_index":0,"sample_rate":24000,"format":"pcm_s16le","first_chunk_latency_ms":450}
```

### Completion Event

```
data: {"done":true,"total_chunks":42,"total_time_ms":3250,"first_chunk_latency_ms":380,"sample_rate":24000}
```

### Error Event

```
data: {"error":"Rate limit exceeded","done":true}
```

### Field Reference

| Field | Type | Present In | Description |
|-------|------|-----------|-------------|
| `audio` | string | Audio chunks | Base64-encoded PCM data |
| `chunk` | string | Audio chunks | Alias for `audio` (some endpoints use this) |
| `chunk_index` | number | Audio chunks | Zero-based index of the chunk |
| `sample_rate` | number | Audio chunks | Always `24000` |
| `format` | string | Audio chunks | Always `pcm_s16le` |
| `mode` | string | Audio chunks | `voicedesign` or `voice_clone` |
| `first_chunk_latency_ms` | number | First chunk | Time to first byte in milliseconds |
| `done` | boolean | Completion | `true` when stream is complete |
| `total_chunks` | number | Completion | Total audio chunks sent |
| `total_time_ms` | number | Completion | Total generation time |
| `error` | string | Error events | Error message if generation failed mid-stream |

## curl

Stream audio directly from the command line. The response is a `text/event-stream` with JSON-encoded SSE events containing base64 PCM audio:

**curl**
```bash
# VoiceDesign streaming
curl -X POST "https://api.murmr.dev/v1/voices/design" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Hello! This audio is streaming to you in real time.",
    "voice_description": "A confident male narrator voice",
    "language": "English"
  }'

# Saved voice streaming
curl -X POST "https://api.murmr.dev/v1/audio/speech/stream" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Stream audio for real-time playback.",
    "voice": "voice_abc123"
  }'
```

## TypeScript

Collect all streamed PCM chunks and combine them into a WAV file. The `first_chunk_latency_ms` field on the first chunk tells you the time-to-first-chunk (TTFC):

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';
import { writeFileSync } from 'node:fs';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

// VoiceDesign streaming
const stream = await client.voices.designStream({
  input: 'Collect the full stream into a WAV file.',
  voice_description: 'A professional female narrator',
});

const pcmChunks: Buffer[] = [];

for await (const chunk of stream) {
  const audioData = chunk.audio || chunk.chunk;
  if (audioData) {
    pcmChunks.push(Buffer.from(audioData, 'base64'));
  }
  if (chunk.first_chunk_latency_ms) {
    console.log(`TTFC: ${chunk.first_chunk_latency_ms}ms`);
  }
}

// Build WAV using SDK utility
import { createWavHeader } from '@murmr/sdk';

const pcm = Buffer.concat(pcmChunks);
const wav = Buffer.concat([createWavHeader(pcm.length), pcm]);
writeFileSync('output.wav', wav);
```

## Python

Stream audio and collect PCM chunks in Python. The stream context manager handles connection lifecycle automatically:

**Python (sync)**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    # Voice Design streaming
    with client.voices.design_stream(
        input="Streaming delivers audio with minimal latency.",
        voice_description="A clear, professional female narrator",
    ) as stream:
        pcm_parts = []
        for chunk in stream:
            pcm_parts.append(chunk.audio_bytes)

            if chunk.first_chunk_latency_ms is not None:
                print(f"TTFC: {chunk.first_chunk_latency_ms:.0f}ms")

        pcm_audio = b"".join(pcm_parts)

    # Saved voice streaming
    with client.speech.stream(
        input="Using a saved voice for streaming.",
        voice="voice_abc123def456",
    ) as stream:
        for chunk in stream:
            pcm_data = chunk.audio_bytes
```

**Python (async)**
```python
import asyncio
import os
from murmr import AsyncMurmrClient

async def main():
    async with AsyncMurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
        async with client.voices.design_stream(
            input="Async streaming for high-concurrency applications.",
            voice_description="A warm male voice",
        ) as stream:
            async for chunk in stream:
                pcm_data = chunk.audio_bytes

asyncio.run(main())
```

## Saving Streamed Audio as WAV

The stream produces raw PCM without a header. To save as a playable WAV file, wrap the PCM data with a WAV header using Python's `wave` module. Set nchannels=1, sampwidth=2, framerate=24000 to match murmr's output:

**Python**
```python
import io
import wave
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    with client.voices.design_stream(
        input="Save streamed audio as a WAV file.",
        voice_description="A cheerful young voice",
    ) as stream:
        pcm_chunks = [chunk.audio_bytes for chunk in stream]

    pcm_data = b"".join(pcm_chunks)

    buf = io.BytesIO()
    with wave.open(buf, "wb") as wf:
        wf.setnchannels(1)
        wf.setsampwidth(2)
        wf.setframerate(24000)
        wf.writeframes(pcm_data)

    with open("output.wav", "wb") as f:
        f.write(buf.getvalue())
```

## Browser: Web Audio API Playback

Stream audio directly to the browser for real-time playback using the Web Audio API. This example parses SSE events from a `fetch` response, decodes base64 PCM to Float32, and schedules gapless playback:

```javascript
async function playStream(text, voiceDescription) {
  const response = await fetch('https://api.murmr.dev/v1/voices/design', {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_API_KEY',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      text,
      voice_description: voiceDescription,
      language: 'English'
    })
  });

  const reader = response.body.getReader();
  const decoder = new TextDecoder();

  const audioContext = new AudioContext({ sampleRate: 24000 });
  let nextStartTime = audioContext.currentTime;

  let buffer = '';
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split('\n');
    buffer = lines.pop() ?? '';

    for (const line of lines) {
      if (!line.startsWith('data: ')) continue;
      const data = JSON.parse(line.slice(6));

      if (data.chunk) {
        // Decode base64 PCM to Float32
        const binary = atob(data.chunk);
        const bytes = new Uint8Array(binary.length);
        for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i);
        const int16 = new Int16Array(bytes.buffer);

        const float32 = new Float32Array(int16.length);
        for (let i = 0; i < int16.length; i++) float32[i] = int16[i] / 32768;

        // Schedule seamless playback
        const audioBuffer = audioContext.createBuffer(1, float32.length, 24000);
        audioBuffer.getChannelData(0).set(float32);

        const source = audioContext.createBufferSource();
        source.buffer = audioBuffer;
        source.connect(audioContext.destination);
        source.start(Math.max(audioContext.currentTime, nextStartTime));
        nextStartTime = Math.max(audioContext.currentTime, nextStartTime) + audioBuffer.duration;
      }

      if (data.done) {
        console.log(`Done: ${data.total_chunks} chunks, TTFC: ${data.first_chunk_latency_ms}ms`);
      }
    }
  }
}

playStream('Hello, world!', 'A warm, friendly voice');
```

## Batch vs Streaming Comparison

| Feature | Batch (`/v1/audio/speech`) | Streaming (`/v1/audio/speech/stream`) |
|---------|---------------------------|--------------------------------------|
| Latency | Seconds (full generation) | ~450ms to first chunk |
| Response | Complete audio file | SSE with PCM chunks |
| Formats | wav, mp3, opus, aac, flac, pcm | PCM only (24kHz, 16-bit, mono) |
| Max text | 4,096 characters | 4,096 characters |
| Webhook support | Yes | No |
| Best for | File generation, async workflows | Real-time playback, interactive apps |

## Error Handling in Streams

Errors can occur before the stream starts (HTTP error status, thrown as `MurmrError`) or mid-stream (error field in an SSE event). Always check both: catch `MurmrError` for pre-stream failures and check `chunk.error` for mid-stream failures:

**TypeScript**
```typescript
import { MurmrClient, MurmrError } from '@murmr/sdk';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

try {
  const stream = await client.speech.stream({
    input: 'Handle errors gracefully.',
    voice: 'voice_abc123',
  });

  for await (const chunk of stream) {
    if (chunk.error) {
      console.error(`Stream error: ${chunk.error}`);
      break;
    }
    // Process audio
  }
} catch (error) {
  if (error instanceof MurmrError) {
    console.error(`API error ${error.status}: ${error.message}`);
  }
}
```

## See Also

- [Speech Generation](./speech.md) -- Batch and streaming endpoints
- [Voice Design](./voicedesign.md) -- Streaming with voice descriptions
- [Audio Formats](./audio-formats.md) -- PCM specifications and conversion
- [Errors](./errors.md) -- Error handling reference
