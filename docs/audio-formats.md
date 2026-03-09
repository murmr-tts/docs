# Audio Formats

murmr supports six audio output formats for batch generation. Streaming endpoints always deliver raw PCM. This guide covers format specifications, when to use each format, and how to convert between them.

## Format Comparison

| Format | MIME Type | Codec | Bitrate | Lossless | Batch | Streaming |
|--------|-----------|-------|---------|----------|-------|-----------|
| `wav` | `audio/wav` | PCM | ~384 kbps | Yes | Default | -- |
| `pcm` | `audio/pcm` | Raw PCM | ~384 kbps | Yes | Yes | Always |
| `mp3` | `audio/mpeg` | LAME | 128 kbps | No | Yes | -- |
| `opus` | `audio/opus` | Opus | 64 kbps | No | Yes | -- |
| `aac` | `audio/aac` | AAC-LC | 64 kbps | No | Yes | -- |
| `flac` | `audio/flac` | FLAC | ~200 kbps | Yes | Yes | -- |

## Native Audio Specifications

All murmr audio is generated at these native specs regardless of output format:

| Property | Value |
|----------|-------|
| Sample rate | 24,000 Hz |
| Bit depth | 16-bit |
| Channels | Mono (1 channel) |
| PCM encoding | Signed 16-bit little-endian (`pcm_s16le`) |

## When to Use Which Format

| Requirement | Recommended Format |
|-------------|-------------------|
| Maximum quality / archival | `wav` |
| Web delivery | `mp3` |
| iOS / Safari | `aac` |
| Bandwidth constrained | `opus` |
| Lossless with smaller file size | `flac` |
| Audio processing pipeline | `pcm` |
| Real-time playback | Use streaming (always PCM) |

## Format Details

### WAV (default)

Lossless, universally supported. Includes a 44-byte RIFF header followed by raw PCM data. Best for maximum quality, server-side processing, and archival. Drawback: large file size.

### PCM (raw)

Same audio data as WAV but without the header. Raw PCM samples in `pcm_s16le` format. Best for custom audio pipelines and real-time processing. Drawback: no metadata -- you must know the sample rate and format to decode.

### MP3

Lossy compression at 128 kbps. Widely supported across all platforms and browsers. Best for web delivery, mobile apps, and bandwidth-constrained environments.

### Opus

Highly efficient lossy codec at 64 kbps. Excellent quality-to-size ratio. Best for WebRTC, real-time communication, and low-bandwidth scenarios. Drawback: not supported in all legacy browsers/players.

### AAC

Lossy compression at 64 kbps. Native to Apple platforms and widely supported. Best for iOS/macOS apps and podcast distribution.

### FLAC

Lossless compression. Typically 40-60% smaller than WAV with zero quality loss. Best for high-quality delivery with smaller file size than WAV. Drawback: not supported in all browsers.

## Using response_format

The `response_format` parameter is only available on the **batch** endpoints (`/v1/audio/speech` and `/v1/voices/design`). Streaming endpoints always return PCM.

**curl**
```bash
curl -X POST "https://api.murmr.dev/v1/audio/speech" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Generate in Opus format for efficient delivery.",
    "voice": "voice_abc123",
    "response_format": "opus"
  }' \
  --output output.opus
```

**TypeScript**
```typescript
import { MurmrClient, isSyncResponse } from '@murmr/sdk';
import { writeFileSync } from 'node:fs';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

const result = await client.speech.create({
  input: 'Generate in Opus format for efficient delivery.',
  voice: 'voice_abc123',
  response_format: 'opus',
});

if (isSyncResponse(result)) {
  const buffer = Buffer.from(await result.arrayBuffer());
  writeFileSync('output.opus', buffer);
}
```

**Python**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    # WAV (default)
    result = client.speech.create_and_wait(
        input="Hello world",
        voice="voice_abc123def456",
        response_format="wav",
    )

    # MP3
    result = client.speech.create_and_wait(
        input="Hello world",
        voice="voice_abc123def456",
        response_format="mp3",
    )
    with open("output.mp3", "wb") as f:
        f.write(result.audio)

    # Opus
    result = client.speech.create_and_wait(
        input="Hello world",
        voice="voice_abc123def456",
        response_format="opus",
    )
    with open("output.opus", "wb") as f:
        f.write(result.audio)
```

## PCM to WAV Conversion

Streaming endpoints return raw PCM. To create a valid WAV file, add a header.

**TypeScript**
```typescript
import { MurmrClient, createWavHeader } from '@murmr/sdk';
import { writeFileSync } from 'node:fs';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

const stream = await client.speech.stream({
  input: 'Convert this PCM stream to a WAV file.',
  voice: 'voice_abc123',
});

const pcmChunks: Buffer[] = [];
for await (const chunk of stream) {
  const audioData = chunk.audio || chunk.chunk;
  if (audioData) {
    pcmChunks.push(Buffer.from(audioData, 'base64'));
  }
}

const pcm = Buffer.concat(pcmChunks);
const header = createWavHeader(pcm.length);
const wav = Buffer.concat([header, pcm]);
writeFileSync('from-stream.wav', wav);
```

**Python**
```python
import io
import wave
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    with client.speech.stream(
        input="Convert this streamed audio to WAV.",
        voice="voice_abc123def456",
    ) as stream:
        pcm_chunks = [chunk.audio_bytes for chunk in stream]

    pcm_data = b"".join(pcm_chunks)

    buf = io.BytesIO()
    with wave.open(buf, "wb") as wf:
        wf.setnchannels(1)
        wf.setsampwidth(2)  # 16-bit = 2 bytes
        wf.setframerate(24000)
        wf.writeframes(pcm_data)

    with open("from_stream.wav", "wb") as f:
        f.write(buf.getvalue())
```

## File Size Estimation

For 24kHz mono 16-bit audio:

| Duration | WAV/PCM | MP3 (128k) | Opus (64k) | AAC (64k) | FLAC |
|----------|---------|------------|------------|-----------|------|
| 10 sec | 480 KB | 160 KB | 80 KB | 80 KB | ~240 KB |
| 1 min | 2.9 MB | 960 KB | 480 KB | 480 KB | ~1.4 MB |
| 10 min | 28.8 MB | 9.6 MB | 4.8 MB | 4.8 MB | ~14 MB |
| 1 hour | 172.8 MB | 57.6 MB | 28.8 MB | 28.8 MB | ~86 MB |

> Compressed format sizes vary by audio content. Speech typically compresses better than music. FLAC sizes are approximate.

## See Also

- [Speech Generation](./speech.md) -- Using `response_format` in batch requests
- [Streaming](./streaming.md) -- PCM streaming format details
- [Voice Design](./voicedesign.md) -- Batch voice design with format selection
