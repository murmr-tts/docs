# SSE Streaming

Stream audio progressively with Server-Sent Events. Start playback in ~450ms instead of waiting for the complete file.

## When to Use SSE

### Good For

- Landing page demos
- Dashboard previews
- Mobile apps (simple integration)
- Any scenario where you want fast first audio

### Consider WebSocket Instead

- Voice agents / chatbots
- LLM streaming integration
- Bidirectional communication
- Text buffering requirements

## Streaming Endpoints

| Endpoint | Mode | Description |
| --- | --- | --- |
| /v1/voices/design | VoiceDesign | SSE streaming (always streams) |
| /v1/voices/design/stream | VoiceDesign | Alias for the above |
| /v1/audio/speech/stream | Saved Voice | Stream with a saved voice ID |

> **ℹ️ VoiceDesign always streams**
> Both `/v1/voices/design` and `/v1/voices/design/stream` return SSE. The `/stream` suffix is optional — it exists for consistency with the Saved Voices API which has separate batch and streaming endpoints.

## Performance

~450ms

Time to first chunk (server-side)

24kHz

PCM sample rate (16-bit mono)

15s

Keepalive interval (SSE comment)

> The server sends an SSE comment (`: keepalive`) every 15 seconds during inactivity to prevent proxies and CDNs from closing the connection. Most SSE clients ignore comments automatically.

## SSE Event Format

The stream returns two event types: audio chunks and a completion marker.

### Audio Chunk

```JSON
data: {
  "chunk": "<base64 PCM int16>",
  "chunk_index": 0,
  "sample_rate": 24000,
  "format": "pcm_s16le",
  "mode": "voicedesign",
  "first_chunk_latency_ms": 450
}
```

### Completion

```JSON
data: {
  "done": true,
  "total_chunks": 5,
  "total_time_ms": 2500,
  "first_chunk_latency_ms": 450,
  "sample_rate": 24000
}
```

#### Field Reference

- `chunk` — Base64-encoded PCM audio (int16, little-endian)
- `chunk_index` — Zero-based chunk sequence number
- `sample_rate` — Always 24000 Hz
- `format` — Always "pcm_s16le"
- `mode` — "voicedesign" or "voice_clone"
- `first_chunk_latency_ms` — Server-side time to first chunk
- `done` — True when stream is complete

## Browser — Web Audio API

Progressive playback with Web Audio API. Audio starts as soon as the first chunk arrives (~450ms):

**VoiceDesign**

```javascript
async function streamTTS(text, voiceDescription) {
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

  // Set up Web Audio for seamless chunk playback
  const audioContext = new AudioContext({ sampleRate: 24000 });
  let nextStartTime = audioContext.currentTime;

  let buffer = '';
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split('\n');
    buffer = lines.pop() ?? '';  // Keep incomplete line

    for (const line of lines) {
      if (!line.startsWith('data: ')) continue;

      const data = JSON.parse(line.slice(6));

      if (data.chunk) {
        // Decode base64 PCM → Int16 → Float32
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

streamTTS('Hello, world!', 'A warm, friendly voice');
```

**Saved Voice**

```javascript
async function streamSavedVoice(text, voiceId) {
  const response = await fetch('https://api.murmr.dev/v1/audio/speech/stream', {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_API_KEY',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      text,
      voice: voiceId,      // e.g. "voice_abc123"
      language: 'English'
    })
  });

  // Same SSE parsing and Web Audio playback as VoiceDesign
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
        const binary = atob(data.chunk);
        const bytes = new Uint8Array(binary.length);
        for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i);
        const int16 = new Int16Array(bytes.buffer);

        const float32 = new Float32Array(int16.length);
        for (let i = 0; i < int16.length; i++) float32[i] = int16[i] / 32768;

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

streamSavedVoice('Hello, world!', 'voice_abc123');
```

## Node.js & Python

The SDKs handle SSE parsing, chunk collection, and WAV header generation automatically.

**Node.js SDK**

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

**Python SDK**

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
        voice="voice_abc123",
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

The stream produces raw PCM without a header. To save as a playable WAV file, wrap the PCM data with a WAV header:

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

## Batch vs Streaming

|  | HTTP Batch | SSE Streaming |
| --- | --- | --- |
| Time to first audio | After full generation | ~450ms |
| Response format | WAV/MP3/Opus/AAC/FLAC | PCM chunks (base64) |
| Request model | Async — 202 + poll | Synchronous stream |
| Best for | Bulk generation, file downloads | Real-time playback, demos |
| Endpoints | /v1/audio/speech | /v1/audio/speech/stream, /v1/voices/design |

## Error Handling

Errors during streaming are returned as SSE events. The stream closes after an error event.

```JSON
data: {"error": "Rate limit exceeded", "done": true}
```

Errors can occur before the stream starts (HTTP error) or mid-stream (error event). The SDK handles both:

**Node.js SDK**

```typescript
import { MurmrClient, MurmrError } from '@murmr/sdk';

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

**Raw fetch**

```javascript
const response = await fetch(url, options);

if (!response.ok) {
  const error = await response.json();
  throw new Error(error.message ?? `HTTP ${response.status}`);
}

// Safe to read SSE stream
const reader = response.body.getReader();
```

## See Also

- [WebSocket Real-time](./realtime.md) — Bidirectional streaming for voice agents
- [Audio Formats](./audio-formats.md) — Batch response formats (MP3, Opus, WAV)
- [VoiceDesign API](./voicedesign.md) — Create voices for streaming
