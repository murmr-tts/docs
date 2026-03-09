# Python SDK

Official Python client for the murmr TTS API. Async-first with full sync support, powered by httpx and Pydantic v2.

## Installation

```bash
pip install murmr
```

> **ℹ️ Requirements**
> Python 3.9 or later. Dependencies: `httpx` (HTTP client) and `pydantic` v2 (response models).

## Client

Two client classes are available. Both support context managers for automatic cleanup.

**Sync**

```python
from murmr import MurmrClient

client = MurmrClient(api_key="murmr_sk_live_...")

# Or as a context manager
with MurmrClient(api_key="murmr_sk_live_...") as client:
    wav = client.voices.design(
        input="Hello!",
        voice_description="A warm, friendly voice",
    )
```

**Async**

```python
from murmr import AsyncMurmrClient

async with AsyncMurmrClient(api_key="murmr_sk_live_...") as client:
    wav = await client.voices.design(
        input="Hello!",
        voice_description="A warm, friendly voice",
    )
```

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `api_key` | str | Yes | -- | Your murmr API key. Sent as a Bearer token. |
| `base_url` | str | No | https://api.murmr.dev | Override the API base URL. |
| `timeout` | float | No | 300.0 | Request timeout in seconds (5 minutes default). |

Both clients expose three resource namespaces:

`client.speech`

Generate audio from text

`client.voices`

Design voices with natural language

`client.jobs`

Track async batch jobs

## client.speech.create()

Submits a batch job and returns an `AsyncJobResponse` with a job ID. Use `create_and_wait()` for a synchronous experience.

```python
job = client.speech.create(
    input="Hello, world!",
    voice="voice_abc123",
    language="English",
)

# job.id = "job_xyz", job.status = "queued"

# Poll for completion
result = client.jobs.wait_for_completion(job.id)
with open("output.wav", "wb") as f:
    f.write(result.audio_bytes)
```

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `input` | str | Yes | -- | Text to synthesize. Max 4,096 characters. |
| `voice` | str | Yes | -- | Saved voice ID (e.g. "voice_abc123"). |
| `language` | str | No | English | Output language. 10 supported + "auto". |
| `response_format` | str | No | wav | Audio format: mp3, opus, aac, flac, wav, or pcm. |
| `webhook_url` | str | No | -- | HTTPS URL for async delivery. Returns 202 with job ID. |

## client.speech.create_and_wait()

Convenience method that submits a batch job and polls until completion. Returns a `JobStatus` with audio data.

```python
result = client.speech.create_and_wait(
    input="Hello, world!",
    voice="voice_abc123",
    language="English",
    response_format="wav",
)

with open("output.wav", "wb") as f:
    f.write(result.audio_bytes)
```

## client.speech.stream()

Stream audio using a saved voice via SSE. Returns a context manager that yields PCM audio chunks.

**Sync**

```python
with client.speech.stream(
    input="Real-time audio streaming.",
    voice="voice_abc123",
) as stream:
    for chunk in stream:
        pcm = chunk.audio_bytes  # 24kHz mono 16-bit PCM
        # Process: pipe to speaker, save, etc.
        if chunk.done:
            break
```

**Async**

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

## client.speech.create_long_form()

Generate audio for text of any length. Handles sentence-boundary chunking, sequential generation, retry with exponential backoff, progress callbacks, and WAV concatenation automatically.

```python
result = client.speech.create_long_form(
    input=long_article_text,
    voice="voice_abc123",
    language="English",
    chunk_size=3500,
    silence_between_chunks_ms=400,
    max_retries=3,
    on_progress=lambda current, total, pct: print(f"{pct}%"),
)

with open("article.wav", "wb") as f:
    f.write(result.audio)
print(f"{result.total_chunks} chunks, {result.duration_ms}ms total")
```

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `input` | str | Yes | -- | Text of any length. Automatically chunked at sentence boundaries. |
| `voice` | str | Yes | -- | Saved voice ID. |
| `chunk_size` | int | No | 3500 | Max characters per chunk. Range: 100-4096. |
| `silence_between_chunks_ms` | int | No | 400 | Milliseconds of silence between chunks. |
| `max_retries` | int | No | 3 | Retry count per chunk. Backoff: 1s, 2s, 4s. |
| `start_from_chunk` | int | No | 0 | Resume from a specific chunk index after failure. |
| `on_progress` | Callable | No | -- | Callback after each chunk. Receives (current, total, percent). |

### Return Value

```python
@dataclass(frozen=True)
class LongFormResult:
    audio: bytes          # WAV audio with silence gaps
    total_chunks: int     # Number of chunks processed
    duration_ms: float    # Total audio duration
    character_count: int  # Total characters processed
```

### Resuming After Failure

If a chunk fails after all retries, a `MurmrChunkError` is thrown with the chunk index. Use `start_from_chunk` to resume.

```python
from murmr import MurmrChunkError

try:
    result = client.speech.create_long_form(input=text, voice=voice)
except MurmrChunkError as err:
    print(f"Failed at chunk {err.chunk_index}/{err.total_chunks}")

    # Retry from the failed chunk
    result = client.speech.create_long_form(
        input=text,
        voice=voice,
        start_from_chunk=err.chunk_index,
    )
```

## client.voices.design()

Generate audio with a natural-language voice description. Returns WAV audio as `bytes`.

```python
wav = client.voices.design(
    input="Welcome to the show.",
    voice_description="A deep, gravelly male voice, slow and deliberate",
    language="English",
)

with open("output.wav", "wb") as f:
    f.write(wav)
```

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `input` | str | Yes | -- | Text to synthesize. Max 4,096 characters. |
| `voice_description` | str | Yes | -- | Natural language voice description. Max 500 characters. |
| `language` | str | No | English | Output language. |

## client.voices.design_stream()

Stream audio with a voice description via SSE. Returns a context manager yielding PCM audio chunks.

**Sync**

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

**Async**

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

## Async Jobs

`speech.create()` always returns a job ID. Use these methods to poll for completion, or pass a `webhook_url` for async delivery.

### client.jobs.get()

```python
status = client.jobs.get("job_xyz")
# status.id, status.status ("queued"|"processing"|"completed"|"failed")
```

### client.jobs.wait_for_completion()

Polls until the job reaches `completed` or `failed`.

```python
result = client.jobs.wait_for_completion(
    "job_xyz",
    poll_interval=3.0,     # default: 3s
    timeout=900.0,         # default: 15 min
)
```

> Raises `MurmrError` with `code='JOB_FAILED'` if the job fails, or `code='TIMEOUT'` if the deadline is exceeded.

## Error Handling

```python
from murmr import MurmrError, MurmrChunkError

try:
    audio = client.speech.create(input=text, voice=voice)
except MurmrError as err:
    print(err.message)    # "Usage limit exceeded..."
    print(err.status)     # 429
    print(err.code)       # "JOB_FAILED", "TIMEOUT", etc.
```

| Class | When Raised | Extra Properties |
| --- | --- | --- |
| MurmrError | API errors, validation, timeouts | status, code, cause |
| MurmrChunkError | Long-form chunk failure after retries | chunk_index, completed_chunks, total_chunks |

## Response Models

All response types are immutable Pydantic v2 models (`frozen=True`).

```python
from murmr import (
    MurmrClient,
    AsyncMurmrClient,
    MurmrError,
    MurmrChunkError,
)

# Response models (Pydantic v2, frozen)
from murmr._types import (
    AsyncJobResponse,     # id, status, created_at
    JobStatus,            # id, status, audio_base64, audio_bytes
    AudioStreamChunk,     # audio_bytes, done
    LongFormResult,       # audio, total_chunks, duration_ms, character_count
)
```

## Audio Constants

| Value | Description |
| --- | --- |
| 24,000 Hz | Sample rate (Qwen3-TTS native) |
| 1 channel | Mono |
| 16-bit | PCM bit depth |
| 2 bytes/sample | Bytes per sample |
| 44 bytes | WAV header size |

## See Also

- [Node.js SDK](./sdk-reference-node.md) — TypeScript/JavaScript client
- [Quickstart](./quickstart.md) — Get started in 5 minutes
- [Async Jobs](./async-jobs.md) — Webhooks, polling, and job lifecycle
- [Error Reference](./errors.md) — All HTTP and WebSocket error codes
