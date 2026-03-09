# Python SDK Reference

Complete reference for the `murmr` Python SDK. Covers all methods, parameters, and return types for `MurmrClient` (sync) and `AsyncMurmrClient` (async).

## Client

```python
from murmr import MurmrClient, AsyncMurmrClient
```

### Constructor Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `api_key` | `str` | Yes | -- | Your murmr API key. Sent as a Bearer token. |
| `base_url` | `str` | No | `https://api.murmr.dev` | Override the API base URL. |
| `timeout` | `float` | No | `300.0` | Request timeout in seconds (5 min default). |

### Context Managers

Both clients support context managers for automatic cleanup (closes the underlying `httpx` client).

```python
# Sync
with MurmrClient(api_key="murmr_sk_live_...") as client:
    # client.close() called automatically on exit
    pass

# Async
async with AsyncMurmrClient(api_key="murmr_sk_live_...") as client:
    # client.close() called automatically on exit
    pass
```

The client exposes three resource namespaces: `client.speech`, `client.voices`, and `client.jobs`.

---

## Speech

### client.speech.create()

Generate speech from text using a saved voice. Returns audio bytes synchronously by default, or a job ID when `webhook_url` is provided.

**Default (no `webhook_url`):** Returns `SyncAudioResponse` with audio bytes (HTTP 200).
**With `webhook_url`:** Returns `AsyncJobResponse` with a job ID (HTTP 202).

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `input` | `str` | Yes | -- | Text to synthesize. Max 4,096 characters. |
| `voice` | `str` | Yes | -- | Saved voice ID (e.g. `voice_abc123`). |
| `voice_clone_prompt` | `str` | No | `None` | Base64 voice embedding from `extract_embeddings()`. Takes precedence over `voice`. |
| `language` | `str` | No | `"English"` | Output language. 10 supported + `"auto"`. |
| `response_format` | `str` | No | `"wav"` | Audio format: `mp3`, `opus`, `aac`, `flac`, `wav`, or `pcm`. |
| `webhook_url` | `str` | No | `None` | HTTPS URL for async delivery. Returns 202 with job ID. |

#### Return Type

`SyncAudioResponse | AsyncJobResponse`

#### Sync Example

```python
result = client.speech.create(
    input="Hello, world!",
    voice="voice_abc123",
    language="English",
)

if isinstance(result, SyncAudioResponse):
    with open("output.wav", "wb") as f:
        f.write(result.audio)
```

#### Async Example

```python
result = await client.speech.create(
    input="Hello, world!",
    voice="voice_abc123",
    language="English",
)

if isinstance(result, SyncAudioResponse):
    with open("output.wav", "wb") as f:
        f.write(result.audio)
```

### client.speech.create_and_wait()

Convenience method that submits a batch job and polls until completion. If the server returns audio synchronously (default), returns `SyncAudioResponse` immediately. Otherwise polls until done.

#### Parameters

All parameters from `speech.create()` (except `webhook_url`) plus:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `poll_interval_s` | `float` | No | `3.0` | Seconds between status polls (minimum 1.0). |
| `timeout_s` | `float` | No | `900.0` | Maximum wait time in seconds (15 min). |
| `on_poll` | `Callable[[JobStatus], None]` | No | `None` | Callback invoked after each poll. |

#### Return Type

`SyncAudioResponse | JobStatus`

#### Sync Example

```python
result = client.speech.create_and_wait(
    input="Hello, world!",
    voice="voice_abc123",
    language="English",
    response_format="wav",
)

if isinstance(result, SyncAudioResponse):
    with open("output.wav", "wb") as f:
        f.write(result.audio)
elif result.audio_bytes:
    with open("output.wav", "wb") as f:
        f.write(result.audio_bytes)
```

#### Async Example

```python
result = await client.speech.create_and_wait(
    input="Hello, world!",
    voice="voice_abc123",
    response_format="mp3",
)

if isinstance(result, SyncAudioResponse):
    with open("output.mp3", "wb") as f:
        f.write(result.audio)
elif result.audio_bytes:
    with open("output.mp3", "wb") as f:
        f.write(result.audio_bytes)
```

### client.speech.stream()

Stream audio using a saved voice via SSE. Returns a context manager that yields an iterator of `AudioStreamChunk` objects. Streaming always returns raw PCM audio (24kHz, mono, 16-bit signed little-endian).

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `input` | `str` | Yes | -- | Text to synthesize. Max 4,096 characters. |
| `voice` | `str` | Yes | -- | Saved voice ID. |
| `voice_clone_prompt` | `str` | No | `None` | Base64 voice embedding. Takes precedence over `voice`. |
| `language` | `str` | No | `"English"` | Output language. |

#### Sync Example

```python
with client.speech.stream(
    input="Real-time audio streaming.",
    voice="voice_abc123",
) as stream:
    for chunk in stream:
        pcm = chunk.audio_bytes  # 24kHz mono 16-bit PCM
        if chunk.done:
            break
```

#### Async Example

```python
async with client.speech.stream(
    input="Real-time audio streaming.",
    voice="voice_abc123",
) as stream:
    async for chunk in stream:
        pcm = chunk.audio_bytes
        if chunk.done:
            break
```

See [streaming](./streaming.md) for more details on SSE audio streaming.

### client.speech.create_long_form()

Generate audio for text of any length. Handles sentence-boundary chunking, sequential generation, retry with exponential backoff, progress callbacks, and WAV concatenation automatically.

For text longer than 4,096 characters, use this method instead of manually splitting text.

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `input` | `str` | Yes | -- | Text of any length. Automatically chunked at sentence boundaries. |
| `voice` | `str` | Yes | -- | Saved voice ID. |
| `voice_clone_prompt` | `str` | No | `None` | Base64 voice embedding. Takes precedence over `voice`. |
| `language` | `str` | No | `"English"` | Output language. |
| `chunk_size` | `int` | No | `3500` | Max characters per chunk. Range: 100--4096. |
| `silence_between_chunks_ms` | `int` | No | `400` | Milliseconds of silence between chunks. |
| `max_retries` | `int` | No | `3` | Retry count per chunk. Backoff: 1s, 2s, 4s. |
| `start_from_chunk` | `int` | No | `0` | Resume from a specific chunk index after failure. |
| `on_progress` | `Callable[[int, int, int], None]` | No | `None` | Callback after each chunk. Receives `(current, total, percent)`. |

#### Return Type: LongFormResult

```python
@dataclass(frozen=True)
class LongFormResult:
    audio: bytes          # WAV audio with silence gaps
    total_chunks: int     # Number of chunks processed
    duration_ms: int      # Total audio duration in ms
    character_count: int  # Total characters processed
```

#### Sync Example

```python
result = client.speech.create_long_form(
    input=long_article_text,
    voice="voice_abc123",
    language="English",
    chunk_size=3500,
    silence_between_chunks_ms=400,
    on_progress=lambda current, total, pct: print(f"{pct}%"),
)

with open("article.wav", "wb") as f:
    f.write(result.audio)
print(f"{result.total_chunks} chunks, {result.duration_ms}ms total")
```

#### Async Example

```python
result = await client.speech.create_long_form(
    input=long_article_text,
    voice="voice_abc123",
    language="English",
    on_progress=lambda current, total, pct: print(f"{pct}%"),
)

with open("article.wav", "wb") as f:
    f.write(result.audio)
```

#### Resuming After Failure

If a chunk fails after all retries, a `MurmrChunkError` is raised with the chunk index. Use `start_from_chunk` to resume:

```python
from murmr import MurmrChunkError

try:
    result = client.speech.create_long_form(input=text, voice="voice_abc123")
except MurmrChunkError as err:
    print(f"Failed at chunk {err.chunk_index}/{err.total_chunks}")
    print(f"{err.completed_chunks} chunks completed")

    # Retry from the failed chunk
    result = client.speech.create_long_form(
        input=text,
        voice="voice_abc123",
        start_from_chunk=err.chunk_index,
    )
```

See [long-form](./long-form.md) for more details.

---

## Voices

### client.voices.design()

Generate audio with a natural-language voice description. Returns WAV audio as `bytes` (24kHz, mono, 16-bit PCM).

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `input` | `str` | Yes | -- | Text to synthesize. Max 4,096 characters. |
| `voice_description` | `str` | Yes | -- | Natural language voice description. Max 500 characters. |
| `language` | `str` | No | `"English"` | Output language. |

#### Sync Example

```python
wav = client.voices.design(
    input="Welcome to the show.",
    voice_description="A deep, gravelly male voice, slow and deliberate",
    language="English",
)

with open("output.wav", "wb") as f:
    f.write(wav)
```

#### Async Example

```python
wav = await client.voices.design(
    input="Welcome to the show.",
    voice_description="A deep, gravelly male voice, slow and deliberate",
    language="English",
)

with open("output.wav", "wb") as f:
    f.write(wav)
```

### client.voices.design_stream()

Stream audio with a voice description via SSE. Returns a context manager yielding an iterator of `AudioStreamChunk` objects with raw PCM audio.

#### Parameters

Same as `voices.design()`.

#### Sync Example

```python
with client.voices.design_stream(
    input="Streaming with a designed voice.",
    voice_description="A deep, authoritative male news anchor",
) as stream:
    for chunk in stream:
        pcm = chunk.audio_bytes
        if chunk.done:
            break
```

#### Async Example

```python
async with client.voices.design_stream(
    input="Streaming with a designed voice.",
    voice_description="A deep, authoritative male news anchor",
) as stream:
    async for chunk in stream:
        pcm = chunk.audio_bytes
        if chunk.done:
            break
```

### client.voices.list_voices()

Retrieve all saved voices for the authenticated user.

#### Sync Example

```python
response = client.voices.list_voices()

print(f"Saved: {response.saved_count}/{response.saved_limit}")
for voice in response.voices:
    print(f"  {voice.id}: {voice.name} ({voice.language})")
```

#### Async Example

```python
response = await client.voices.list_voices()

print(f"Saved: {response.saved_count}/{response.saved_limit}")
for voice in response.voices:
    print(f"  {voice.id}: {voice.name} ({voice.language})")
```

#### Return Type: VoiceListResponse

| Field | Type | Description |
|-------|------|-------------|
| `voices` | `list[SavedVoice]` | List of saved voices. |
| `saved_count` | `int` | Number of voices saved. |
| `saved_limit` | `int` | Maximum voices allowed on your plan. |
| `total` | `int` | Total voice count. |

Each `SavedVoice` has: `id`, `name`, `description`, `language`, `language_name`, `audio_preview_url`, `created_at`.

### client.voices.save()

Save a voice from audio generated by Voice Design for reuse with saved-voice endpoints.

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `name` | `str` | Yes | -- | Display name (1--50 characters). |
| `audio` | `bytes` | Yes | -- | WAV audio bytes from a VoiceDesign generation. |
| `description` | `str` | Yes | -- | The voice description used to generate the audio. |
| `ref_text` | `str` | Yes | -- | Transcript of the reference audio (required for ICL extraction). |
| `language` | `str` | No | `"English"` | Language code. |

Voice limits by plan: Free: 3, Starter: 10, Pro: 25, Realtime: 50, Scale: 100.

#### Sync Example

```python
wav = client.voices.design(
    input="This is my reference audio for voice saving.",
    voice_description="A calm, professional male voice",
)

saved = client.voices.save(
    name="Professional Narrator",
    audio=wav,
    description="A calm, professional male voice",
    ref_text="This is my reference audio for voice saving.",
    language="English",
)

print(f"Voice ID: {saved.id}")
```

#### Async Example

```python
wav = await client.voices.design(
    input="This is my reference audio for voice saving.",
    voice_description="A calm, professional male voice",
)

saved = await client.voices.save(
    name="Professional Narrator",
    audio=wav,
    description="A calm, professional male voice",
    ref_text="This is my reference audio for voice saving.",
)

print(f"Voice ID: {saved.id}")
```

#### Return Type: VoiceSaveResponse

| Field | Type | Description |
|-------|------|-------------|
| `id` | `str` | Voice ID (e.g. `voice_abc123def456`). |
| `name` | `str` | Display name. |
| `language` | `str` | Language code. |
| `description` | `str` | Voice description. |
| `prompt_size_bytes` | `int` | Size of stored embeddings. |
| `created_at` | `str` | ISO 8601 timestamp. |
| `success` | `bool` | Whether the save succeeded. |
| `has_audio_preview` | `bool` | Whether a preview was stored. |

### client.voices.delete()

Permanently delete a saved voice by ID.

#### Sync Example

```python
result = client.voices.delete("voice_abc123def456")
print(f"Deleted: {result.success}")
```

#### Async Example

```python
result = await client.voices.delete("voice_abc123def456")
print(f"Deleted: {result.success}")
```

#### Return Type: VoiceDeleteResponse

| Field | Type | Description |
|-------|------|-------------|
| `success` | `bool` | Whether the deletion succeeded. |
| `id` | `str` | The deleted voice ID. |
| `message` | `str` | Confirmation message. |

### client.voices.extract_embeddings()

Extract portable voice embeddings from audio without saving the voice. Store the returned `prompt_data` in your own database and pass it via `voice_clone_prompt` in any TTS request. See [portable embeddings](./portable-embeddings.md).

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `audio` | `bytes` | Yes | WAV audio to extract embeddings from. |
| `ref_text` | `str` | Yes | Transcript of the reference audio (improves extraction quality). |

#### Sync Example

```python
result = client.voices.extract_embeddings(
    audio=open("reference.wav", "rb").read(),
    ref_text="Transcript of the reference audio.",
)

# Use the embedding in a TTS request
with client.speech.stream(
    input="Hello from a portable voice!",
    voice="inline",
    voice_clone_prompt=result.prompt_data,
) as stream:
    for chunk in stream:
        pcm = chunk.audio_bytes
```

#### Async Example

```python
result = await client.voices.extract_embeddings(
    audio=open("reference.wav", "rb").read(),
    ref_text="Transcript of the reference audio.",
)

async with client.speech.stream(
    input="Hello from a portable voice!",
    voice="inline",
    voice_clone_prompt=result.prompt_data,
) as stream:
    async for chunk in stream:
        pcm = chunk.audio_bytes
```

#### Return Type: ExtractEmbeddingsResponse

| Field | Type | Description |
|-------|------|-------------|
| `prompt_data` | `str` | Base64-encoded voice embedding data. |
| `prompt_size_bytes` | `int` | Size of the embedding in bytes. |

---

## Jobs

### client.jobs.get()

Get the status of an async batch job.

#### Sync Example

```python
status = client.jobs.get("job_xyz")
print(f"Status: {status.status}")
```

#### Async Example

```python
status = await client.jobs.get("job_xyz")
print(f"Status: {status.status}")
```

#### Return Type: JobStatus

| Field | Type | Description |
|-------|------|-------------|
| `id` | `str` | Job ID. |
| `status` | `str` | `"queued"`, `"processing"`, `"completed"`, or `"failed"`. |
| `created_at` | `str` | ISO 8601 timestamp. |
| `completed_at` | `str \| None` | Completion timestamp. |
| `duration_ms` | `int \| None` | Audio duration in ms. |
| `error` | `str \| None` | Error message if failed. |
| `audio_base64` | `str \| None` | Base64-encoded audio (when completed). |
| `content_type` | `str \| None` | Audio content type (e.g. `audio/wav`). |
| `response_format` | `str \| None` | Audio format used. |

The `JobStatus` model also provides an `audio_bytes` property that decodes `audio_base64` into raw `bytes`:

```python
if status.audio_bytes:
    with open("output.wav", "wb") as f:
        f.write(status.audio_bytes)
```

### client.jobs.wait_for_completion()

Poll until the job reaches `completed` or `failed`. Raises `MurmrError` with `code='JOB_FAILED'` if the job fails, or `code='TIMEOUT'` if the deadline is exceeded.

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `job_id` | `str` | Yes | -- | The job ID to poll. |
| `poll_interval_s` | `float` | No | `3.0` | Seconds between polls (minimum 1.0). |
| `timeout_s` | `float` | No | `900.0` | Maximum wait time in seconds (15 min). |
| `on_poll` | `Callable[[JobStatus], None]` | No | `None` | Callback after each poll. |

#### Sync Example

```python
result = client.jobs.wait_for_completion(
    "job_xyz",
    poll_interval_s=3.0,
    timeout_s=900.0,
    on_poll=lambda status: print(f"Status: {status.status}"),
)

if result.audio_bytes:
    with open("output.wav", "wb") as f:
        f.write(result.audio_bytes)
```

#### Async Example

```python
result = await client.jobs.wait_for_completion(
    "job_xyz",
    poll_interval_s=3.0,
    timeout_s=900.0,
)

if result.audio_bytes:
    with open("output.wav", "wb") as f:
        f.write(result.audio_bytes)
```

#### Return Type

`JobStatus` -- Always has `status == "completed"` and `audio_base64` populated.

---

## Error Handling

All errors raised by the SDK are instances of `MurmrError` or its subclass `MurmrChunkError`.

```python
from murmr import MurmrError, MurmrChunkError

try:
    result = client.speech.create(input=text, voice="voice_abc123")
except MurmrChunkError as err:
    print(f"Long-form failed at chunk {err.chunk_index}/{err.total_chunks}")
except MurmrError as err:
    print(err.message)    # "Usage limit exceeded..."
    print(err.status)     # 429
    print(err.code)       # "JOB_FAILED", "TIMEOUT", etc.
```

| Class | When Raised | Properties |
|-------|-------------|------------|
| `MurmrError` | API errors, validation, timeouts | `message`, `status`, `code`, `type`, `concurrent_limit`, `concurrent_active` |
| `MurmrChunkError` | Long-form chunk failure after retries | `chunk_index`, `completed_chunks`, `total_chunks` (plus all `MurmrError` fields) |

See [errors](./errors.md) for all error codes and retry strategies.

---

## Response Models

All response types are immutable Pydantic v2 models (`frozen=True`).

```python
from murmr._types import (
    SyncAudioResponse,        # audio (bytes), content_type, duration_ms, total_time_ms
    AsyncJobResponse,         # id, status ("queued"), created_at
    JobStatus,                # id, status, audio_base64, audio_bytes (property)
    AudioStreamChunk,         # audio_bytes (property), done, chunk_index
    LongFormResult,           # audio (bytes), total_chunks, duration_ms, character_count
    SavedVoice,               # id, name, description, language, created_at
    VoiceListResponse,        # voices, saved_count, saved_limit, total
    VoiceSaveResponse,        # id, name, success, prompt_size_bytes
    VoiceDeleteResponse,      # success, id, message
    ExtractEmbeddingsResponse,# prompt_data, prompt_size_bytes
)
```

### SyncAudioResponse

Returned by `speech.create()` when the server responds synchronously (HTTP 200).

| Field | Type | Description |
|-------|------|-------------|
| `audio` | `bytes` | Raw audio bytes in the requested format. |
| `content_type` | `str` | MIME type (e.g. `audio/wav`). |
| `duration_ms` | `int` | Audio duration in milliseconds. |
| `total_time_ms` | `int` | Total server processing time in milliseconds. |

### AudioStreamChunk

Yielded by `speech.stream()` and `voices.design_stream()`.

| Field | Type | Description |
|-------|------|-------------|
| `audio` | `str \| None` | Base64-encoded PCM audio data. |
| `chunk` | `str \| None` | Alias for audio (some SSE events use this field). |
| `chunk_index` | `int \| None` | Zero-based index of this chunk. |
| `sample_rate` | `int \| None` | Sample rate (always 24000). |
| `first_chunk_latency_ms` | `float \| None` | Time to first chunk in milliseconds. |
| `done` | `bool \| None` | `True` when the stream is complete. |
| `total_chunks` | `int \| None` | Total chunks in the stream. |
| `total_time_ms` | `float \| None` | Total generation time. |
| `error` | `str \| None` | Error message, if any. |

The `audio_bytes` property decodes the `audio` or `chunk` field from base64:

```python
for chunk in stream:
    pcm = chunk.audio_bytes  # bytes, ready to pipe to a speaker
```

---

## Audio Constants

| Value | Description |
|-------|-------------|
| 24,000 Hz | Sample rate (Qwen3-TTS native). |
| 1 channel | Mono. |
| 16-bit | PCM bit depth. |
| 2 bytes/sample | Bytes per sample. |
| 44 bytes | WAV header size. |

---

## See Also

- [Installation](./installation.md) -- Setup and configuration
- [Node.js SDK Reference](./sdk-reference-node.md) -- TypeScript equivalent
- [Streaming](./streaming.md) -- SSE streaming details
- [Long-Form Generation](./long-form.md) -- Text of any length
- [Async Jobs](./async-jobs.md) -- Webhooks, polling, job lifecycle
- [Audio Formats](./audio-formats.md) -- Format specs and encoding
- [Errors](./errors.md) -- All error codes and retry strategies
