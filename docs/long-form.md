# Long-Form Audio

Generate audio from text of any length -- articles, blog posts, book chapters. The SDK handles chunking, generation, retry, and concatenation automatically.

## How It Works

The murmr API has a 4,096-character limit per request. For longer text, the SDK splits your input into chunks at natural sentence boundaries, generates audio for each chunk sequentially, and concatenates the results with configurable silence gaps.

1. **Split** -- Text is split at sentence boundaries (`.!?` and CJK equivalents), falling back to clause boundaries, then word boundaries.
2. **Generate** -- Each chunk is sent to the streaming endpoint sequentially.
3. **Retry** -- Failed chunks are retried with exponential backoff (up to 3 retries by default).
4. **Concatenate** -- PCM audio from all chunks is joined with configurable silence gaps and wrapped in a WAV header.

> **Saved voices only.** Long-form generation uses saved voices (via `voice` ID or `voice_clone_prompt`). VoiceDesign descriptions are not supported because each chunk would produce a slightly different voice.

## Quick Example

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';
import { writeFileSync } from 'fs';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

const result = await client.speech.createLongForm({
  input: articleText,             // Any length
  voice: 'voice_abc123',         // Saved voice ID
  language: 'English',
  onProgress: ({ current, total, percent }) => {
    console.log(`Chunk ${current}/${total} (${percent}%)`);
  },
});

writeFileSync('article.wav', Buffer.from(result.audio));
console.log(`Generated ${result.totalChunks} chunks, ${result.durationMs}ms total`);
```

**Python (sync)**
```python
import os
from murmr import MurmrClient

def on_progress(current: int, total: int, percent: int) -> None:
    print(f"Chunk {current}/{total} ({percent}%)")

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    result = client.speech.create_long_form(
        input=article_text,
        voice="voice_abc123",
        language="English",
        on_progress=on_progress,
    )

    with open("article.wav", "wb") as f:
        f.write(result.audio)

    print(f"Chunks: {result.total_chunks}")
    print(f"Duration: {result.duration_ms}ms")
    print(f"Characters: {result.character_count}")
```

**Python (async)**
```python
import asyncio
import os
from murmr import AsyncMurmrClient

async def main():
    async with AsyncMurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
        result = await client.speech.create_long_form(
            input=article_text,
            voice="voice_abc123",
            language="English",
        )

        with open("article.wav", "wb") as f:
            f.write(result.audio)

asyncio.run(main())
```

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `input` | string | (required) | Text of any length. Automatically chunked at sentence boundaries. |
| `voice` | string | (required) | Saved voice ID (e.g., `voice_abc123`). |
| `voice_clone_prompt` | string | -- | Base64-encoded voice embedding. Takes precedence over `voice` if set. |
| `language` | string | `English` | Output language. |
| `chunkSize` / `chunk_size` | number | `3500` | Max characters per chunk. Range: 100-4096. |
| `silenceBetweenChunksMs` / `silence_between_chunks_ms` | number | `400` | Silence between chunks in milliseconds. |
| `maxRetries` / `max_retries` | number | `3` | Retry count per chunk. Exponential backoff: 1s, 2s, 4s. |
| `startFromChunk` / `start_from_chunk` | number | `0` | Resume from a specific chunk index after failure. |
| `onProgress` / `on_progress` | function | -- | Progress callback. Receives `{ current, total, percent }`. |

## Chunking Strategy

The SDK splits text intelligently to preserve natural speech flow:

| Priority | Boundary Type | Characters |
|----------|---------------|------------|
| 1 (best) | Sentence endings | `. ! ?` (and CJK equivalents) |
| 2 | Clause boundaries | `, ; : --` (and CJK equivalents) |
| 3 (fallback) | Word boundaries | Whitespace |

The default chunk size of 3,500 characters keeps chunks below the 4,096 API limit while leaving room for sentence-boundary splitting.

**TypeScript**
```typescript
import { splitIntoChunks } from '@murmr/sdk';

const chunks = splitIntoChunks(longText, 3500);
console.log(`Split into ${chunks.length} chunks`);
```

## Silence Between Chunks

| Value | Effect |
|-------|--------|
| 0 | No gap (chunks joined seamlessly) |
| 200 | Short pause (sentence boundary feel) |
| 400 | Default (paragraph boundary feel) |
| 800 | Long pause (chapter or section break feel) |

## Error Recovery with start_from_chunk

If a chunk fails after all retries, a `MurmrChunkError` is thrown with the exact chunk index. Use `startFromChunk` / `start_from_chunk` to resume without re-generating completed chunks.

**TypeScript**
```typescript
import { MurmrChunkError } from '@murmr/sdk';

try {
  const result = await client.speech.createLongForm({
    input: bookChapter,
    voice: 'voice_abc123',
  });
} catch (err) {
  if (err instanceof MurmrChunkError) {
    console.log(`Failed at chunk ${err.chunkIndex}/${err.totalChunks}`);
    console.log(`${err.completedChunks} chunks completed successfully`);

    // Resume from the failed chunk
    const result = await client.speech.createLongForm({
      input: bookChapter,
      voice: 'voice_abc123',
      startFromChunk: err.chunkIndex,
    });
  }
}
```

**Python**
```python
from murmr import MurmrClient, MurmrChunkError

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    try:
        result = client.speech.create_long_form(
            input=book_chapter,
            voice="voice_abc123",
        )
    except MurmrChunkError as e:
        print(f"Failed at chunk {e.chunk_index} ({e.completed_chunks}/{e.total_chunks} completed)")

        # Resume from the failed chunk
        result = client.speech.create_long_form(
            input=book_chapter,
            voice="voice_abc123",
            start_from_chunk=e.chunk_index,
        )
```

> When resuming, the SDK re-chunks the text identically (same input + same chunk_size = same chunks). Skipped chunks still fire progress callbacks so your progress UI stays accurate.

## Audio Format

Long-form generation always produces a WAV file (24kHz, 16-bit, mono PCM). To convert to other formats:

**Python**
```python
from pydub import AudioSegment
import io

audio = AudioSegment.from_wav(io.BytesIO(result.audio))
audio.export("output.mp3", format="mp3", bitrate="192k")
```

## See Also

- [Async Jobs](./async-jobs.md) -- Batch processing with polling and webhooks
- [Voice Crafting](./voice-crafting.md) -- Create effective voice descriptions
- [Style Instructions](./style-instructions.md) -- Control voice delivery
