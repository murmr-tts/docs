# Error Reference

Complete guide to API errors, their causes, and how to resolve them. murmr uses standard HTTP status codes and structured error responses.

## Error Response Format

All errors use the structured OpenAI-compatible format. Parse the `error.type` and `error.code` fields to determine the cause programmatically:

```json
{
  "error": {
    "message": "Text exceeds maximum length of 4096 characters",
    "type": "invalid_request_error",
    "code": "invalid_parameter",
    "param": null
  }
}
```

Rate limit errors (429) include additional metadata showing your concurrent request limits, which helps you implement proper throttling:

```json
{
  "error": {
    "type": "rate_limit_error",
    "code": "concurrent_limit_exceeded",
    "message": "Too many concurrent requests. Your plan allows 5 concurrent requests."
  }
}
```

## HTTP Error Codes

### 400 Bad Request

The request is malformed or contains invalid parameters.

**Common causes:**
- Missing `text` or `input` field
- Text exceeds 4,096 characters
- Missing `voice` and `voice_description`
- `voice_description` exceeds 500 characters
- Invalid `response_format` value
- Invalid `webhook_url` (not HTTPS, private IP, or too long)
- Invalid JSON body

### 401 Unauthorized

Authentication failed.

**Common causes:**
- Missing `Authorization` header
- Invalid API key format (must start with `murmr_sk_`)
- Expired or revoked API key
- Missing space after "Bearer" in the header

### 404 Not Found

The requested resource does not exist.

**Common causes:**
- Voice ID does not exist or belongs to a different user account
- Job ID not found or expired (jobs expire after 24 hours)
- Invalid endpoint path

### 405 Method Not Allowed

Wrong HTTP method for the endpoint. TTS endpoints require POST. Job status endpoint requires GET.

### 410 Gone

The resource existed but has been removed. Completed job data is retained for a limited time. Re-submit the request or use webhooks for reliable delivery.

### 429 Rate Limited / Quota Exceeded

You have exceeded a rate limit. Check `error.type` or the error message to determine the cause:

| Type / Message | Cause | Action |
|----------------|-------|--------|
| `rate_limit_error` (concurrent) | Concurrent request limit for your plan | Wait for a current request to complete |
| `"Usage limit exceeded."` | Monthly character quota exhausted | Upgrade plan or wait for monthly reset |
| `"Voice design limit exceeded."` | Voice design quota for the period | Upgrade plan or wait for reset |
| `"Server is at capacity."` | All backend GPU slots are busy | Retry in a few seconds |

Headers on 429 responses:

| Header | Description |
|--------|-------------|
| `X-Concurrent-Limit` | Maximum concurrent requests allowed |
| `X-Concurrent-Active` | Current number of in-flight requests |

### 500 Internal Server Error

An unexpected error occurred on the server. Retry with exponential backoff. If persistent, contact support@murmr.dev.

### 502 Bad Gateway

The API gateway could not reach the backend. Usually transient. Retry after a short delay.

### 503 Service Unavailable

The backend is temporarily unavailable (scaling up, cold start, maintenance). Retry with backoff.

### 504 Gateway Timeout

The backend did not respond in time. Common with very long text near the 4,096 character limit. Retry, or use streaming for long text.

## SDK Error Classes

### TypeScript (`@murmr/sdk`)

The SDK wraps all API errors in typed error classes. Use `instanceof` checks to handle different error types. `MurmrError` is the base class; `MurmrChunkError` extends it for long-form generation failures:

```typescript
import { MurmrError, MurmrChunkError } from '@murmr/sdk';

// MurmrError -- base error for all API errors
class MurmrError extends Error {
  readonly status?: number;
  readonly code?: string;
  readonly type?: string;
  readonly concurrentLimit?: number;
  readonly concurrentActive?: number;
}

// MurmrChunkError -- thrown during long-form generation
class MurmrChunkError extends MurmrError {
  readonly chunkIndex: number;
  readonly completedChunks: number;
  readonly totalChunks: number;
}
```

### Python (`murmr`)

Python error classes mirror the TypeScript SDK. Access error details via attributes like `.status`, `.code`, and `.message`:

```python
from murmr import MurmrError, MurmrChunkError

# MurmrError attributes:
# .message (str) -- Human-readable error description
# .status (int) -- HTTP status code
# .code (str) -- Machine-readable error code
# .type (str) -- Error type category
# .concurrent_limit (int) -- Max concurrent requests (rate limit errors)
# .concurrent_active (int) -- Current in-flight requests (rate limit errors)

# MurmrChunkError (extends MurmrError) attributes:
# .chunk_index (int) -- Zero-based index of the failed chunk
# .completed_chunks (int) -- Chunks generated before the failure
# .total_chunks (int) -- Total chunks in the long-form request
```

## Error Handling Examples

Handle different error types by checking `status` and `code`. This pattern covers auth failures, rate limits, and bad requests in a single try/catch:

**TypeScript**
```typescript
import { MurmrClient, MurmrError } from '@murmr/sdk';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

try {
  await client.speech.stream({ input: 'Hello', voice: 'voice_abc123' });
} catch (error) {
  if (error instanceof MurmrError) {
    if (error.status === 429) {
      console.error(`Rate limited: ${error.code}`);
      if (error.concurrentLimit) {
        console.error(`Active: ${error.concurrentActive}/${error.concurrentLimit}`);
      }
    } else if (error.status === 400) {
      console.error('Bad request:', error.message);
    } else if (error.status === 401) {
      console.error('Check your API key');
    }
  }
}
```

Equivalent error handling in Python. The `concurrent_active` and `concurrent_limit` fields are only populated on 429 rate limit errors:

**Python**
```python
import os
from murmr import MurmrClient, MurmrError

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    try:
        wav = client.voices.design(
            input="Hello world",
            voice_description="A calm voice",
        )
    except MurmrError as e:
        print(f"Status: {e.status}")
        print(f"Code: {e.code}")
        print(f"Type: {e.type}")
        print(f"Message: {e.message}")

        if e.status == 429:
            print(f"Active: {e.concurrent_active}/{e.concurrent_limit}")
```

## Retry Logic

Implement exponential backoff for transient errors (429, 500, 502, 503, 504). Do **not** retry 400, 401, or 404 errors -- fix the request instead.

Retry a streaming request with exponential backoff. Only retries on transient HTTP status codes; all other errors propagate immediately:

**TypeScript**
```typescript
import { MurmrClient, MurmrError } from '@murmr/sdk';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

async function generateWithRetry(
  input: string,
  voice: string,
  maxRetries: number = 3,
): Promise<Buffer> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const stream = await client.speech.stream({ input, voice });
      const chunks: Buffer[] = [];
      for await (const chunk of stream) {
        const audio = chunk.audio || chunk.chunk;
        if (audio) chunks.push(Buffer.from(audio, 'base64'));
      }
      return Buffer.concat(chunks);
    } catch (error) {
      if (error instanceof MurmrError) {
        const retryable = [429, 500, 502, 503, 504].includes(error.status ?? 0);
        if (!retryable || attempt === maxRetries) throw error;

        const delay = Math.min(1000 * Math.pow(2, attempt), 30_000);
        console.log(`Retry ${attempt + 1}/${maxRetries} in ${delay}ms...`);
        await new Promise((r) => setTimeout(r, delay));
      } else {
        throw error;
      }
    }
  }
  throw new Error('Unreachable');
}
```

Same retry pattern in Python. For 429 errors, uses a longer minimum delay (5 seconds) since rate limits need more cooldown:

**Python**
```python
import os
import time
from murmr import MurmrClient, MurmrError

def generate_with_retry(client, text: str, voice: str, max_retries: int = 3):
    for attempt in range(max_retries + 1):
        try:
            return client.speech.create_and_wait(input=text, voice=voice)
        except MurmrError as e:
            if e.status in (429, 500, 502, 503) and attempt < max_retries:
                delay = min(2 ** attempt, 60)
                if e.status == 429:
                    delay = max(5.0, delay)
                print(f"Retry {attempt + 1}/{max_retries} in {delay}s: {e.message}")
                time.sleep(delay)
            else:
                raise

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    result = generate_with_retry(client, "Hello world", "voice_abc123def456")
```

## WebSocket Close Codes

For the realtime WebSocket endpoint (`/v1/realtime`):

| Code | Name | Description | Retryable |
|------|------|-------------|-----------|
| 4001 | AUTH_TIMEOUT | No config message received within 10 seconds | No |
| 4002 | AUTH_FAILED | Invalid API key in the config message | No |
| 4003 | INVALID_MESSAGE | Malformed JSON or unexpected message type | No (fix message) |
| 4004 | RATE_LIMITED | Too many concurrent connections or generation rate exceeded | Yes (after delay) |
| 4005 | SERVER_ERROR | Internal server error during processing | Yes (with backoff) |
| 4006 | CAPACITY | All GPU slots occupied (sent as error message, not close code) | Yes (retry shortly) |

## Troubleshooting

| Symptom | Likely Cause | Action |
|---------|-------------|--------|
| All requests return 401 | API key issue | Verify key in environment variable, check for whitespace |
| Intermittent 502/503 | Backend scaling | Retry with backoff |
| 429 with `concurrent_limit` | Too many parallel calls | Reduce concurrency or queue requests |
| 429 with `monthly_limit` | Usage exhausted | Upgrade plan or wait for monthly reset |
| Stream cuts off mid-audio | Network interruption | Check connection, retry |
| First request is slow | Cold start | Normal -- first request after idle may take 20-30 seconds |
| Empty audio returned | Empty input text | Validate input is non-empty |
| Request timing out | Long text or high load | Set HTTP timeout to at least 60 seconds |

## See Also

- [Authentication](./authentication.md) -- API key setup and plans
- [Rate Limits](./rate-limits.md) -- Detailed limit tables
- [Streaming](./streaming.md) -- Error handling in SSE streams
