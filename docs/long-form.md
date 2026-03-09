# Long-Form Audio

Generate audio from text of any length — articles, blog posts, book chapters. The SDK handles chunking, generation, retry, and concatenation automatically.

## How It Works

The murmr API has a 4,096-character limit per request. For longer text, the SDK splits your input into chunks at natural sentence boundaries, generates audio for each chunk sequentially, and concatenates the results with configurable silence gaps.

1

Split text at sentence boundaries

Splits at `.!?` and CJK equivalents. Falls back to clause boundaries, then word boundaries.

2

Generate audio for each chunk

Sends each chunk to `/v1/audio/speech` sequentially with your saved voice.

3

Retry failed chunks

Exponential backoff (1s, 2s, 4s) with up to 3 retries per chunk.

4

Concatenate with silence gaps

Inserts 400ms of silence between chunks (WAV/PCM only) and produces a single audio file.

## Quick Example

```typescript
import { MurmrClient } from '@murmr/sdk';
import { writeFileSync } from 'fs';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

const result = await client.speech.createLongForm({
  input: articleText,             // Any length
  voice: "voice_abc123",          // Saved voice ID
  language: "English",
  onProgress: ({ current, total, percent }) => {
    console.log(`Chunk ${current}/${total} (${percent}%)`);
  },
});

writeFileSync("article.wav", Buffer.from(result.audio));
console.log(`Generated ${result.totalChunks} chunks, ${result.durationMs}ms total`);
```

> **ℹ️ Saved voices only**
> Long-form generation uses saved voices (via `voice` ID or `voice_clone_prompt`). VoiceDesign descriptions are not supported because each chunk would produce a slightly different voice. [Save a voice first](./voices.md).

## Text Chunking Strategy

The SDK splits text intelligently to preserve natural speech flow. It never splits mid-sentence.

| Priority | Boundary Type | Characters |
| --- | --- | --- |
| 1 (best) | Sentence endings | . ! ? (and CJK: 。 ！ ？) |
| 2 | Clause boundaries | , ; : — (and CJK: 、 ； ：) |
| 3 (fallback) | Word boundaries | Whitespace |

The default chunk size is 3,500 characters (configurable from 100 to 4,096). Keeping it below the 4,096 API limit leaves room for sentence-boundary splitting to work effectively.

```typescript
// You can also use the chunker standalone
import { splitIntoChunks } from '@murmr/sdk';

const chunks = splitIntoChunks(longText, 3500);
console.log(`Split into ${chunks.length} chunks`);
```

## Audio Format Considerations

| Format | Silence Gaps | Concatenation | Best For |
| --- | --- | --- | --- |
| WAV | Yes (PCM silence) | Header-aware | Highest quality, editing |
| PCM | Yes (raw silence) | Raw concatenation | Processing pipelines |
| MP3 | No | Binary concatenation | Web delivery, small files |
| Opus | No | Binary concatenation | Streaming, real-time |
| AAC | No | Binary concatenation | Mobile apps |
| FLAC | No | Binary concatenation | Lossless archival |

> **⚠️ Silence gaps require WAV or PCM**
> The `silenceBetweenChunksMs` option only works with WAV and PCM formats. Compressed formats (MP3, Opus, AAC, FLAC) are binary-concatenated without silence gaps. If natural pauses between sections matter, use WAV.

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `input` | string | Yes | -- | Text of any length. Automatically chunked at sentence boundaries. |
| `voice` | string | Yes | -- | Saved voice ID (e.g. "voice_abc123"). |
| `voice_clone_prompt` | string | No | -- | Base64-encoded voice embedding. Takes precedence over voice if set. |
| `language` | string | No | English | Output language. |
| `response_format` | AudioFormat | No | wav | Audio format: mp3, opus, aac, flac, wav, or pcm. |
| `chunkSize` | number | No | 3500 | Max characters per chunk. Range: 100–4096. |
| `silenceBetweenChunksMs` | number | No | 400 | Milliseconds of silence between chunks. WAV and PCM only. |
| `maxRetries` | number | No | 3 | Retry count per chunk. Exponential backoff: 1s, 2s, 4s. |
| `startFromChunk` | number | No | 0 | Resume from a specific chunk index after failure. |
| `onProgress` | (progress) => void | No | -- | Progress callback. Receives { current, total, percent }. |

## Return Value

```typescript
interface LongFormResult {
  audio: Buffer;          // Concatenated audio with silence gaps
  totalChunks: number;    // Number of chunks processed
  durationMs: number;     // Total audio duration in ms
  format: AudioFormat;    // Audio format used
  characterCount: number; // Total characters processed
}
```

## Resuming After Failure

If a chunk fails after all retries, a `MurmrChunkError` is thrown with the exact chunk index. Use `startFromChunk` to resume without re-generating completed chunks.

```typescript
import { MurmrChunkError } from '@murmr/sdk';

try {
  const result = await client.speech.createLongForm({
    input: bookChapter,
    voice: "voice_abc123",
  });
} catch (err) {
  if (err instanceof MurmrChunkError) {
    console.log(`Failed at chunk ${err.chunkIndex}/${err.totalChunks}`);
    console.log(`${err.completedChunks} chunks completed successfully`);

    // Resume from the failed chunk
    const result = await client.speech.createLongForm({
      input: bookChapter,
      voice: "voice_abc123",
      startFromChunk: err.chunkIndex,
    });
  }
}
```

> When resuming, the SDK re-chunks the text identically (same input + same chunkSize = same chunks). Skipped chunks still fire `onProgress` callbacks so your progress UI stays accurate.

## Tuning Tips

### Chunk size

Larger chunks (3,500–4,000) produce more natural prosody across sentences. Smaller chunks (500–1,000) reduce the impact of a single chunk failure but may produce more noticeable transitions.

### Silence between chunks

The default 400ms works well for paragraph transitions. Try 200ms for continuous narration or 600–800ms for distinct sections. Set to 0 for seamless concatenation.

### Character usage

Each chunk counts against your monthly character quota. A 10,000-character article split into 3 chunks uses 10,000 characters from your plan, not 3 requests.

## See Also

- [Node.js SDK](./sdk-reference-node.md) — Full SDK reference including createLongForm()
- [Saved Voices API](./speech.md) — Single-request audio generation
- [Text Formatting](./text-formatting.md) — Newlines and formatting for natural prosody
- [Audio Formats](./audio-formats.md) — Format specs and encoding details
- [Rate Limits](./rate-limits.md) — Character quotas and plan limits
