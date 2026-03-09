# Node.js SDK Reference

Complete reference for the `@murmr/sdk` package. All methods, parameters, return types, and utility functions.

## MurmrClient

The main entry point. Create one instance and reuse it across your application.

```typescript
import { MurmrClient } from '@murmr/sdk';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
  // baseUrl: 'https://api.murmr.dev', // default
  // timeout: 300_000,                 // 5 min default
});
```

### Constructor Options

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `apiKey` | `string` | Yes | -- | Your murmr API key. Sent as a Bearer token. |
| `baseUrl` | `string` | No | `https://api.murmr.dev` | Override the API base URL. |
| `timeout` | `number` | No | `300000` | Request timeout in milliseconds (5 min default). |

The client exposes three resource namespaces: `client.speech`, `client.voices`, and `client.jobs`.

---

## Speech

### client.speech.create()

Generate speech from text using a saved voice. Returns audio bytes synchronously by default, or a job ID when `webhook_url` is provided.

**Default (no `webhook_url`):** Returns a `Response` with binary audio bytes (HTTP 200). Use `isSyncResponse()` type guard, then `.arrayBuffer()` or `.blob()` to consume.

**With `webhook_url`:** Returns `AsyncJobResponse` with a job ID (HTTP 202). Poll with `client.jobs.get()`.

```typescript
import { isSyncResponse } from '@murmr/sdk';

// Sync (default) -- get audio directly
const response = await client.speech.create({
  input: 'Hello, world!',
  voice: 'voice_abc123',
  language: 'English',
});

if (isSyncResponse(response)) {
  const buffer = Buffer.from(await response.arrayBuffer());
  writeFileSync('output.wav', buffer);
}
```

```typescript
// Async (webhook) -- get job ID for later retrieval
const job = await client.speech.create({
  input: 'A long piece of text...',
  voice: 'voice_abc123',
  webhook_url: 'https://your-app.com/webhooks/murmr',
});
// job = { id: "job_xyz", status: "queued", created_at: "..." }
```

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `input` | `string` | Yes | -- | Text to synthesize. Max 4,096 characters. |
| `voice` | `string` | Yes | -- | Saved voice ID (e.g. `voice_abc123`). |
| `voice_clone_prompt` | `string` | No | -- | Base64 voice embedding from `extractEmbeddings()`. Takes precedence over `voice`. |
| `language` | `string` | No | `"English"` | Output language. 10 supported + `"auto"`. |
| `response_format` | `AudioFormat` | No | `"wav"` | Audio format: `mp3`, `opus`, `aac`, `flac`, `wav`, or `pcm`. |
| `webhook_url` | `string` | No | -- | HTTPS URL for async delivery. Returns 202 with job ID. |

#### Return Type

`Promise<Response | AsyncJobResponse>`

Use the type guards `isSyncResponse()` and `isAsyncResponse()` to narrow the type.

### client.speech.createAndWait()

Convenience method that submits a batch job and polls until completion. If the server returns audio synchronously (default), returns immediately. Otherwise polls until done.

```typescript
const result = await client.speech.createAndWait({
  input: 'Hello, world!',
  voice: 'voice_abc123',
  language: 'English',
  response_format: 'wav',
  onPoll: (status) => console.log(status.status),
});

if (result instanceof Response) {
  writeFileSync('output.wav', Buffer.from(await result.arrayBuffer()));
} else {
  writeFileSync('output.wav', Buffer.from(result.audio_base64!, 'base64'));
}
```

#### Parameters

All parameters from `speech.create()` plus:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `pollIntervalMs` | `number` | No | `3000` | Milliseconds between status polls (minimum 1000). |
| `timeoutMs` | `number` | No | `900000` | Maximum wait time in milliseconds (15 min). |
| `onPoll` | `(status: JobStatus) => void` | No | -- | Callback after each poll. |

#### Return Type

`Promise<JobStatus | Response>`

### client.speech.stream()

Stream audio using a saved voice via SSE. Returns an async generator of PCM audio chunks (24kHz, mono, 16-bit).

```typescript
const stream = await client.speech.stream({
  input: 'Real-time audio streaming.',
  voice: 'voice_abc123',
});

for await (const chunk of stream) {
  const audioData = chunk.audio || chunk.chunk;
  if (audioData) {
    const pcm = Buffer.from(audioData, 'base64');
    // Process PCM: pipe to speaker, ffmpeg, etc.
  }
  if (chunk.done) break;
}
```

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `input` | `string` | Yes | -- | Text to synthesize. Max 4,096 characters. |
| `voice` | `string` | Yes | -- | Saved voice ID. |
| `voice_clone_prompt` | `string` | No | -- | Base64 voice embedding. Takes precedence over `voice`. |
| `language` | `string` | No | `"English"` | Output language. |

#### Return Type

`Promise<AsyncGenerator<AudioStreamChunk>>`

Streaming always returns raw PCM audio (24kHz, mono, 16-bit signed little-endian). The `response_format` parameter is not available for streaming. See [streaming](./streaming.md) for details.

### client.speech.createLongForm()

Generate audio for text of any length. Handles sentence-boundary chunking, sequential generation, retry with exponential backoff, progress callbacks, and audio concatenation automatically. Always produces WAV output.

For text longer than 4,096 characters, use this method instead of manually splitting text.

```typescript
const result = await client.speech.createLongForm({
  input: longArticleText,          // No length limit
  voice: 'voice_abc123',
  language: 'English',
  chunkSize: 3500,                 // chars per chunk (default: 3500, max: 4096)
  silenceBetweenChunksMs: 400,     // silence gap (default: 400ms)
  maxRetries: 3,                   // retries per chunk (default: 3)
  onProgress: ({ current, total, percent }) => {
    console.log(`Chunk ${current}/${total} (${percent}%)`);
  },
});

writeFileSync('article.wav', Buffer.from(result.audio));
console.log(`${result.totalChunks} chunks, ${result.durationMs}ms total`);
```

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `input` | `string` | Yes | -- | Text of any length. Automatically chunked at sentence boundaries. |
| `voice` | `string` | Yes | -- | Saved voice ID. |
| `voice_clone_prompt` | `string` | No | -- | Base64 voice embedding. Takes precedence over `voice`. |
| `language` | `string` | No | `"English"` | Output language. |
| `chunkSize` | `number` | No | `3500` | Max characters per chunk. Range: 100--4096. |
| `silenceBetweenChunksMs` | `number` | No | `400` | Milliseconds of silence between chunks (WAV/PCM only). |
| `maxRetries` | `number` | No | `3` | Retry count per chunk. Backoff: 1s, 2s, 4s. |
| `startFromChunk` | `number` | No | `0` | Resume from a specific chunk index after failure. |
| `onProgress` | `(progress: LongFormProgress) => void` | No | -- | Callback after each chunk. Receives `{ current, total, percent }`. |

#### Return Type: LongFormResult

```typescript
interface LongFormResult {
  audio: Buffer;          // WAV audio with silence gaps
  totalChunks: number;    // Number of chunks processed
  durationMs: number;     // Total audio duration in ms
  characterCount: number; // Total characters processed
}
```

#### Resuming After Failure

If a chunk fails after all retries, a `MurmrChunkError` is thrown with the chunk index. Use `startFromChunk` to resume:

```typescript
import { MurmrChunkError } from '@murmr/sdk';

try {
  const result = await client.speech.createLongForm({ input, voice: 'voice_abc123' });
} catch (err) {
  if (err instanceof MurmrChunkError) {
    console.log(`Failed at chunk ${err.chunkIndex}/${err.totalChunks}`);
    console.log(`${err.completedChunks} chunks completed`);

    // Retry from the failed chunk
    const result = await client.speech.createLongForm({
      input,
      voice: 'voice_abc123',
      startFromChunk: err.chunkIndex,
    });
  }
}
```

See [long-form](./long-form.md) for more details.

---

## Voices

### client.voices.design()

Generate audio with a natural-language voice description. Sends the text to the VoiceDesign endpoint, collects the SSE stream, and returns a complete WAV buffer (24kHz, mono, 16-bit PCM).

```typescript
const audio = await client.voices.design({
  input: 'Welcome to the show.',
  voice_description: 'A deep, gravelly male voice, slow and deliberate',
  language: 'English',
});

writeFileSync('output.wav', audio);
```

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `input` | `string` | Yes | -- | Text to synthesize. Max 4,096 characters. |
| `voice_description` | `string` | Yes | -- | Natural language voice description. Max 500 characters. |
| `language` | `string` | No | `"English"` | Output language. |

#### Return Type

`Promise<Buffer>` -- Complete WAV audio.

### client.voices.designStream()

Stream audio with a voice description via SSE. Returns an async generator of PCM audio chunks.

```typescript
const stream = await client.voices.designStream({
  input: 'Streaming with a designed voice.',
  voice_description: 'A deep, authoritative male news anchor',
  language: 'English',
});

for await (const chunk of stream) {
  const audioData = chunk.audio || chunk.chunk;
  if (audioData) {
    const pcm = Buffer.from(audioData, 'base64');
  }
  if (chunk.done) break;
}
```

#### Parameters

Same as `voices.design()`.

#### Return Type

`Promise<AsyncGenerator<AudioStreamChunk>>`

### client.voices.list()

List all saved voices for the authenticated user. Returns voice metadata and plan limits.

```typescript
const { voices, saved_count, saved_limit } = await client.voices.list();

console.log(`Saved: ${saved_count}/${saved_limit}`);
for (const voice of voices) {
  console.log(`${voice.name} (${voice.id})`);
}
```

#### Return Type: VoiceListResponse

| Field | Type | Description |
|-------|------|-------------|
| `voices` | `SavedVoice[]` | List of saved voices. |
| `saved_count` | `number` | Number of voices saved. |
| `saved_limit` | `number` | Maximum voices allowed on your plan. |
| `total` | `number` | Total voice count. |

Each `SavedVoice` has: `id`, `name`, `description`, `language`, `language_name`, `audio_preview_url`, `created_at`.

### client.voices.save()

Save a VoiceDesign output for reuse. Accepts WAV audio (`Buffer` or `Uint8Array`), base64-encodes it, and extracts embeddings server-side.

```typescript
const wav = await client.voices.design({
  input: 'This is my reference audio for voice saving.',
  voice_description: 'A warm narrator',
});

const saved = await client.voices.save({
  name: 'My Narrator',
  audio: wav,
  description: 'A warm narrator',
  ref_text: 'This is my reference audio for voice saving.',
});
console.log(saved.id);  // voice_abc123def456
```

Voice limits by plan: Free: 3, Starter: 10, Pro: 25, Realtime: 50, Scale: 100.

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `name` | `string` | Yes | -- | Display name (1--50 characters). |
| `audio` | `Buffer \| Uint8Array` | Yes | -- | WAV audio bytes from a VoiceDesign generation. |
| `description` | `string` | Yes | -- | The voice description used to generate the audio. |
| `ref_text` | `string` | Yes | -- | Transcript of the reference audio (required for ICL extraction). |
| `language` | `string` | No | `"English"` | Language code. |

#### Return Type: VoiceSaveResponse

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | Voice ID (e.g. `voice_abc123def456`). |
| `name` | `string` | Display name. |
| `language` | `string` | Language code. |
| `description` | `string` | Voice description. |
| `prompt_size_bytes` | `number` | Size of stored embeddings. |
| `created_at` | `string` | ISO 8601 timestamp. |
| `success` | `boolean` | Whether the save succeeded. |
| `has_audio_preview` | `boolean` | Whether a preview was stored. |

### client.voices.delete()

Delete a saved voice by ID.

```typescript
await client.voices.delete('voice_abc123def456');
```

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `voiceId` | `string` | Yes | The voice ID to delete (e.g. `voice_abc123def456`). |

#### Return Type: VoiceDeleteResponse

| Field | Type | Description |
|-------|------|-------------|
| `success` | `boolean` | Whether the deletion succeeded. |
| `id` | `string` | The deleted voice ID. |
| `message` | `string` | Confirmation message. |

### client.voices.extractEmbeddings()

Extract portable voice embeddings from audio. Store the returned `prompt_data` in your own database and pass it via `voice_clone_prompt` in any TTS request -- no saved voice ID needed. See [portable embeddings](./portable-embeddings.md).

```typescript
const { prompt_data, prompt_size_bytes } = await client.voices.extractEmbeddings({
  audio: wavBuffer,
  ref_text: 'Transcript of the reference audio.',
});

// Use the embedding in a TTS request
const stream = await client.speech.stream({
  input: 'Hello from a portable voice!',
  voice: 'inline',
  voice_clone_prompt: prompt_data,
});
```

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `audio` | `Buffer \| Uint8Array` | Yes | WAV audio to extract embeddings from. |
| `ref_text` | `string` | Yes | Transcript of the reference audio (improves extraction quality). |

#### Return Type: ExtractEmbeddingsResponse

| Field | Type | Description |
|-------|------|-------------|
| `prompt_data` | `string` | Base64-encoded voice embedding data. |
| `prompt_size_bytes` | `number` | Size of the embedding in bytes. |

---

## Jobs

### client.jobs.get()

Get the status of an async batch job.

```typescript
const status = await client.jobs.get('job_xyz');
// status.id, status.status: "queued" | "processing" | "completed" | "failed"
```

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `jobId` | `string` | Yes | The job ID to query. |

#### Return Type: JobStatus

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | Job ID. |
| `status` | `string` | `"queued"`, `"processing"`, `"completed"`, or `"failed"`. |
| `created_at` | `string` | ISO 8601 timestamp. |
| `completed_at` | `string \| null` | Completion timestamp. |
| `duration_ms` | `number \| null` | Audio duration in ms. |
| `error` | `string \| null` | Error message if failed. |
| `audio_base64` | `string \| undefined` | Base64-encoded audio (when completed). |
| `content_type` | `string \| undefined` | Audio content type (e.g. `audio/wav`). |
| `response_format` | `string \| undefined` | Audio format used. |

### client.jobs.waitForCompletion()

Poll until the job reaches `completed` or `failed`. Throws `MurmrError` with `code: 'JOB_FAILED'` if the job fails, or `code: 'TIMEOUT'` if the deadline is exceeded.

```typescript
const result = await client.jobs.waitForCompletion('job_xyz', {
  pollIntervalMs: 3000,    // default: 3s (min: 1s)
  timeoutMs: 900_000,      // default: 15 min
  onPoll: (status) => {
    console.log(`Job status: ${status.status}`);
  },
});

writeFileSync('output.wav', Buffer.from(result.audio_base64!, 'base64'));
```

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `jobId` | `string` | Yes | -- | The job ID to poll. |
| `pollIntervalMs` | `number` | No | `3000` | Milliseconds between polls (minimum 1000). |
| `timeoutMs` | `number` | No | `900000` | Maximum wait time in milliseconds (15 min). |
| `onPoll` | `(status: JobStatus) => void` | No | -- | Callback after each poll. |

#### Return Type

`Promise<JobStatus>` -- Always has `status: 'completed'` and `audio_base64` populated.

---

## Error Handling

All errors thrown by the SDK are instances of `MurmrError` or its subclass `MurmrChunkError`.

```typescript
import { MurmrError, MurmrChunkError } from '@murmr/sdk';

try {
  const audio = await client.speech.create({ input, voice: 'voice_abc123' });
} catch (err) {
  if (err instanceof MurmrChunkError) {
    console.error(`Long-form failed at chunk ${err.chunkIndex}/${err.totalChunks}`);
  } else if (err instanceof MurmrError) {
    console.error(err.message);    // "Usage limit exceeded..."
    console.error(err.status);     // 429
    console.error(err.code);       // "JOB_FAILED", "TIMEOUT", etc.
  }
}
```

| Class | When Thrown | Properties |
|-------|------------|------------|
| `MurmrError` | API errors, validation, timeouts | `status`, `code`, `type`, `cause` |
| `MurmrChunkError` | Long-form chunk failure after retries | `chunkIndex`, `completedChunks`, `totalChunks` (plus all `MurmrError` fields) |

See [errors](./errors.md) for all error codes and retry strategies.

---

## Utility Functions

Standalone exports for advanced use cases.

### splitIntoChunks()

Split text at sentence boundaries. Supports Latin and CJK punctuation. Falls back to clause boundaries, then word boundaries.

```typescript
import { splitIntoChunks } from '@murmr/sdk';

const chunks = splitIntoChunks(longText, 3500);
// Returns string[] — each chunk <= 3500 characters
```

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `text` | `string` | Yes | -- | Text to split. |
| `maxChunkSize` | `number` | No | `3500` | Max characters per chunk (100--4096). |

### concatenateAudio()

Concatenate multiple audio buffers with optional silence gaps.

```typescript
import { concatenateAudio } from '@murmr/sdk';

const combined = concatenateAudio(audioBuffers, 'wav', 400);
// WAV: strips headers, concatenates PCM, adds silence, writes new header
// PCM: concatenates raw data with silence
// MP3/Opus/AAC/FLAC: binary concatenation (no silence)
```

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `buffers` | `Buffer[]` | Yes | -- | Audio buffers to concatenate. |
| `format` | `AudioFormat` | No | `"wav"` | Audio format. |
| `silenceMs` | `number` | No | `0` | Milliseconds of silence between chunks. |

### Type Guards

```typescript
import { isSyncResponse, isAsyncResponse } from '@murmr/sdk';

const result = await client.speech.create({ input: 'Hello', voice: 'voice_abc123' });

if (isSyncResponse(result)) {
  // result is Response — audio bytes available
  const buffer = Buffer.from(await result.arrayBuffer());
}

if (isAsyncResponse(result)) {
  // result is AsyncJobResponse — poll for completion
  const completed = await client.jobs.waitForCompletion(result.id);
}
```

---

## Audio Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `SAMPLE_RATE` | `24000` | 24 kHz (Qwen3-TTS native rate). |
| `CHANNELS` | `1` | Mono. |
| `BITS_PER_SAMPLE` | `16` | 16-bit PCM. |
| `BYTES_PER_SAMPLE` | `2` | 2 bytes per sample. |
| `WAV_HEADER_SIZE` | `44` | Standard RIFF/WAV header. |

---

## Type Exports

All types are exported for use in your TypeScript code.

```typescript
import type {
  MurmrClientOptions,
  AudioFormat,              // 'mp3' | 'opus' | 'aac' | 'flac' | 'wav' | 'pcm'
  SpeechCreateOptions,
  SpeechStreamOptions,
  CreateAndWaitOptions,
  VoiceDesignOptions,
  VoiceDesignStreamOptions,
  AsyncJobResponse,
  JobStatus,
  LongFormOptions,
  LongFormProgress,
  LongFormResult,
  AudioStreamChunk,
  SavedVoice,
  VoiceListResponse,
  VoiceSaveOptions,
  VoiceSaveResponse,
  VoiceDeleteResponse,
  ExtractEmbeddingsOptions,
  ExtractEmbeddingsResponse,
  WebhookPayload,
} from '@murmr/sdk';
```

---

## See Also

- [Installation](./installation.md) -- Setup and configuration
- [Python SDK Reference](./sdk-reference-python.md) -- Python equivalent
- [Streaming](./streaming.md) -- SSE streaming details
- [Long-Form Generation](./long-form.md) -- Text of any length
- [Async Jobs](./async-jobs.md) -- Webhooks, polling, job lifecycle
- [Audio Formats](./audio-formats.md) -- Format specs and encoding
- [Errors](./errors.md) -- All error codes and retry strategies
