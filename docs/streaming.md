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

**Node.js**

```javascript
import fs from 'fs';

async function streamToFile(text, voiceDescription) {
  const response = await fetch('https://api.murmr.dev/v1/voices/design', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.MURMR_API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      text,
      voice_description: voiceDescription,
      language: 'English'
    })
  });

  const chunks = [];

  for await (const line of response.body.pipeThrough(new TextDecoderStream()).pipeThrough(
    new TransformStream({
      transform(chunk, controller) {
        for (const line of chunk.split('\n')) {
          if (line.startsWith('data: ')) controller.enqueue(line.slice(6));
        }
      }
    })
  )) {
    const data = JSON.parse(line);
    if (data.chunk) chunks.push(Buffer.from(data.chunk, 'base64'));
    if (data.done) console.log(`TTFC: ${data.first_chunk_latency_ms}ms`);
  }

  // Write WAV file
  const pcm = Buffer.concat(chunks);
  const wavHeader = createWavHeader(pcm.length, 24000, 1, 16);
  fs.writeFileSync('output.wav', Buffer.concat([wavHeader, pcm]));
}

function createWavHeader(dataSize, sampleRate, channels, bitsPerSample) {
  const header = Buffer.alloc(44);
  const byteRate = sampleRate * channels * (bitsPerSample / 8);
  const blockAlign = channels * (bitsPerSample / 8);

  header.write('RIFF', 0);
  header.writeUInt32LE(36 + dataSize, 4);
  header.write('WAVE', 8);
  header.write('fmt ', 12);
  header.writeUInt32LE(16, 16);
  header.writeUInt16LE(1, 20);
  header.writeUInt16LE(channels, 22);
  header.writeUInt32LE(sampleRate, 24);
  header.writeUInt32LE(byteRate, 28);
  header.writeUInt16LE(blockAlign, 32);
  header.writeUInt16LE(bitsPerSample, 34);
  header.write('data', 36);
  header.writeUInt32LE(dataSize, 40);
  return header;
}
```

**Python**

```python
import requests
import base64
import json
import io
import wave

def stream_tts(text: str, voice_description: str) -> bytes:
    """Stream VoiceDesign audio and return PCM bytes."""
    response = requests.post(
        "https://api.murmr.dev/v1/voices/design",
        headers={
            "Authorization": "Bearer YOUR_API_KEY",
            "Content-Type": "application/json",
        },
        json={
            "text": text,
            "voice_description": voice_description,
            "language": "English",
        },
        stream=True,
    )

    chunks = []
    for line in response.iter_lines():
        if not line:
            continue

        line = line.decode("utf-8")
        if not line.startswith("data: "):
            continue

        data = json.loads(line[6:])

        if "chunk" in data:
            chunks.append(base64.b64decode(data["chunk"]))
            print(f"Chunk {data['chunk_index']}: {len(chunks[-1])} bytes")

        if data.get("done"):
            print(f"TTFC: {data['first_chunk_latency_ms']}ms")
            break

    return b"".join(chunks)

# Save as WAV
pcm_data = stream_tts("Hello, world!", "A warm, friendly voice")

buf = io.BytesIO()
with wave.open(buf, "wb") as wf:
    wf.setnchannels(1)
    wf.setsampwidth(2)  # 16-bit
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

If the request fails before streaming starts (bad parameters, auth failure), you get a standard HTTP error response instead of SSE. Always check `response.ok` before reading the stream.

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
