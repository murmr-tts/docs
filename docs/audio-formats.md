# Audio Formats

murmr supports six audio output formats via the `response_format` parameter. All formats are available on all plans.

## Supported Formats

Set `response_format` in your request body. Defaults to `mp3` (via Worker; `wav` on pod direct).

| Format | Codec | Bitrate | Content-Type | Best For |
| --- | --- | --- | --- | --- |
| wav | — | Lossless | audio/wav | Highest quality, editing |
| pcm | — | Raw | audio/pcm | Direct playback, real-time pipelines |
| mp3 | libmp3lame | 128k | audio/mpeg | Default, downloads, broadest compatibility |
| opus | libopus | 64k | audio/opus | Low bandwidth, WebRTC, voice agents |
| aac | aac | 64k | audio/aac | iOS/Safari, mobile apps |
| flac | flac | Lossless | audio/flac | Archival, lossless compression |

> **ℹ️ Batch Only**
> - **Batch endpoint** (`/v1/audio/speech`) supports all 6 formats via the `response_format` parameter. Format conversion uses ffmpeg server-side.
> - **Streaming endpoints** (`/stream`) always return raw PCM chunks (base64 encoded in SSE JSON), regardless of `response_format`.

## Requesting a Format

**cURL**

```curl
curl -X POST "https://api.murmr.dev/v1/audio/speech" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Hello, world!",
    "voice": "voice_abc123",
    "response_format": "mp3"
  }' --output output.mp3
# Returns 200 with binary audio (mp3 default)
```

**Python**

```python
import requests

response = requests.post(
    "https://api.murmr.dev/v1/audio/speech",
    headers={"Authorization": "Bearer YOUR_API_KEY"},
    json={
        "text": "Hello, world!",
        "voice": "voice_abc123",
        "response_format": "mp3"
    }
)

# response.content contains binary audio (audio/mpeg)
with open("output.mp3", "wb") as f:
    f.write(response.content)
```

**JavaScript**

```javascript
const response = await fetch("https://api.murmr.dev/v1/audio/speech", {
  method: "POST",
  headers: {
    "Authorization": "Bearer YOUR_API_KEY",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    text: "Hello, world!",
    voice: "voice_abc123",
    response_format: "mp3"
  })
});

// response.body contains binary audio (audio/mpeg)
const blob = await response.blob();
```

## Native Audio Specifications

All audio is generated at these native specifications before encoding:

| Property | Value |
| --- | --- |
| Sample Rate | 24,000 Hz |
| Bit Depth | 16-bit signed |
| Channels | Mono (1) |
| Encoding | PCM (pcm_s16le) |
| Byte Order | Little-endian |

## PCM Format Details

Raw PCM audio is signed 16-bit integers in little-endian byte order:

```Binary
Each sample: 2 bytes (16 bits)
  - Range: -32768 to +32767
  - Little-endian: low byte first, high byte second

Example (silence): 0x00 0x00
Example (max positive): 0xFF 0x7F = +32767
Example (max negative): 0x00 0x80 = -32768

Duration calculation:
  duration_seconds = num_bytes / (sample_rate * 2)
  duration_seconds = num_bytes / 48000
```

## Base64 Decoding

Streaming endpoints return PCM data as base64-encoded strings in JSON. Decode to raw bytes:

**JavaScript**

```javascript
// Decode base64 to PCM Int16Array
function decodeBase64PCM(base64: string): Int16Array {
  const binaryString = atob(base64);
  const bytes = new Uint8Array(binaryString.length);

  for (let i = 0; i < binaryString.length; i++) {
    bytes[i] = binaryString.charCodeAt(i);
  }

  // Convert to Int16Array (2 bytes per sample)
  return new Int16Array(bytes.buffer);
}

// Usage with SSE chunk
const data = JSON.parse(event.data.slice(6));  // Remove "data: "
const pcmSamples = decodeBase64PCM(data.chunk);

// Convert to Float32 for Web Audio API (-1.0 to +1.0)
const float32 = new Float32Array(pcmSamples.length);
for (let i = 0; i < pcmSamples.length; i++) {
  float32[i] = pcmSamples[i] / 32768;
}
```

**Python**

```python
import base64
import struct

def decode_base64_pcm(base64_string: str) -> bytes:
    """Decode base64 to raw PCM bytes."""
    return base64.b64decode(base64_string)

def pcm_to_samples(pcm_bytes: bytes) -> list[int]:
    """Convert PCM bytes to list of 16-bit signed integers."""
    # '<' = little-endian, 'h' = signed short (2 bytes)
    num_samples = len(pcm_bytes) // 2
    return list(struct.unpack(f'<{num_samples}h', pcm_bytes))

# Usage
pcm_bytes = decode_base64_pcm(chunk_data['chunk'])
samples = pcm_to_samples(pcm_bytes)
print(f"Decoded {len(samples)} samples")
```

## Converting PCM to WAV

To save streaming audio as a WAV file, prepend a 44-byte header to the raw PCM data:

**JavaScript**

```javascript
function pcmToWav(pcmData: ArrayBuffer, sampleRate = 24000): Blob {
  const numChannels = 1;
  const bitsPerSample = 16;
  const byteRate = sampleRate * numChannels * bitsPerSample / 8;
  const blockAlign = numChannels * bitsPerSample / 8;
  const dataSize = pcmData.byteLength;

  // Create 44-byte WAV header
  const header = new ArrayBuffer(44);
  const view = new DataView(header);

  // "RIFF" chunk descriptor
  writeString(view, 0, 'RIFF');
  view.setUint32(4, 36 + dataSize, true);  // File size - 8
  writeString(view, 8, 'WAVE');

  // "fmt " sub-chunk
  writeString(view, 12, 'fmt ');
  view.setUint32(16, 16, true);            // Subchunk1 size (16 for PCM)
  view.setUint16(20, 1, true);             // Audio format (1 = PCM)
  view.setUint16(22, numChannels, true);
  view.setUint32(24, sampleRate, true);
  view.setUint32(28, byteRate, true);
  view.setUint16(32, blockAlign, true);
  view.setUint16(34, bitsPerSample, true);

  // "data" sub-chunk
  writeString(view, 36, 'data');
  view.setUint32(40, dataSize, true);

  return new Blob([header, pcmData], { type: 'audio/wav' });
}

function writeString(view: DataView, offset: number, str: string) {
  for (let i = 0; i < str.length; i++) {
    view.setUint8(offset + i, str.charCodeAt(i));
  }
}

// Usage: Concatenate all PCM chunks, then convert
const allPcmData = concatenateArrayBuffers(pcmChunks);
const wavBlob = pcmToWav(allPcmData, 24000);

// Download
const url = URL.createObjectURL(wavBlob);
const a = document.createElement('a');
a.href = url;
a.download = 'audio.wav';
a.click();
```

**Python**

```python
import wave
import io

def pcm_to_wav(pcm_bytes: bytes, sample_rate: int = 24000) -> bytes:
    """Convert raw PCM bytes to WAV format."""
    buffer = io.BytesIO()

    with wave.open(buffer, 'wb') as wav_file:
        wav_file.setnchannels(1)         # Mono
        wav_file.setsampwidth(2)          # 16-bit = 2 bytes
        wav_file.setframerate(sample_rate)
        wav_file.writeframes(pcm_bytes)

    return buffer.getvalue()

# Usage: Collect all PCM chunks from streaming
pcm_chunks = []
for chunk in stream_response():
    pcm_chunks.append(base64.b64decode(chunk['chunk']))

all_pcm = b''.join(pcm_chunks)
wav_data = pcm_to_wav(all_pcm, sample_rate=24000)

with open('output.wav', 'wb') as f:
    f.write(wav_data)
```

## Web Audio API Playback

For real-time playback in the browser, schedule PCM chunks with the Web Audio API:

```javascript
const audioContext = new AudioContext();
let nextStartTime = audioContext.currentTime;

function playPCMChunk(pcmSamples: Float32Array, sampleRate: number) {
  // Create audio buffer
  const buffer = audioContext.createBuffer(1, pcmSamples.length, sampleRate);
  buffer.getChannelData(0).set(pcmSamples);

  // Create and schedule source
  const source = audioContext.createBufferSource();
  source.buffer = buffer;
  source.connect(audioContext.destination);

  // Schedule playback (gapless)
  const startTime = Math.max(nextStartTime, audioContext.currentTime);
  source.start(startTime);

  // Update next start time
  nextStartTime = startTime + buffer.duration;
}

// Important: Convert Int16 PCM to Float32 first
function int16ToFloat32(pcm: Int16Array): Float32Array {
  const float = new Float32Array(pcm.length);
  for (let i = 0; i < pcm.length; i++) {
    float[i] = pcm[i] / 32768;
  }
  return float;
}
```

> **User gesture required:** AudioContext must be created or resumed after a user interaction (click, tap) due to browser autoplay policies.

## File Size Calculation

Estimate audio file sizes for capacity planning:

```Formula
PCM bytes = duration_seconds × sample_rate × bytes_per_sample
PCM bytes = duration_seconds × 24000 × 2
PCM bytes = duration_seconds × 48000

WAV bytes = PCM bytes + 44 (header)

Examples:
  1 second  → 48 KB
  10 seconds → 480 KB
  1 minute  → 2.88 MB
  10 minutes → 28.8 MB
```

## See Also

- [Saved Voices API](./speech.md) — Generate speech with response_format
- [SSE Streaming](./streaming.md) — Receive PCM chunks via Server-Sent Events
- [WebSocket Protocol](./websocket-protocol.md) — Real-time audio with binary mode
