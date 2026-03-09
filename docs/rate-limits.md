# Rate Limits & Usage

murmr enforces rate limits to ensure fair usage and platform stability. Limits are based on your plan and include monthly character quotas, concurrent request caps, and saved voice limits.

## Per-Plan Limits

| Plan | Price | Characters/Month | Concurrent Requests | Saved Voices | API Keys | WebSocket |
|------|-------|-------------------|---------------------|--------------|----------|-----------|
| Free | $0 | 50,000 | 2 | 3 | 1 | No |
| Starter | $10 | 1,000,000 | 5 | 10 | 3 | No |
| Pro | $25 | 3,000,000 | 10 | 25 | 5 | No |
| Realtime | $49 | 3,000,000 | 10 | 50 | 10 | Yes |
| Scale | $99 | 10,000,000 | 25 | 100 | 25 | Yes |

> WebSocket access (`/v1/realtime`) requires the Realtime or Scale plan.

## Overage Pricing

When you exceed your monthly character allocation on a paid plan, additional characters are billed at the overage rate:

| Plan | Overage Rate |
|------|-------------|
| Starter | $12 per 1M characters |
| Pro | $10 per 1M characters |
| Realtime | $10 per 1M characters |
| Scale | $8 per 1M characters |

The Free plan does not allow overages. Requests are rejected with a `429` error when the limit is reached.

## Character Counting

Characters are counted from the `text` (or `input`) field in each request.

**Rules:**
- Whitespace characters (spaces, newlines, tabs) are counted
- Punctuation is counted
- The `voice_description` field does not count against your quota
- Each API call counts the full text length, even if the request fails during generation
- Streaming and batch requests count the same way
- Voice Design and Speech endpoints count characters the same way
- Unused characters do not roll over to the next month

## Rate Limit Headers

Every HTTP API response includes these headers:

| Header | Description | Example |
|--------|-------------|---------|
| `X-RateLimit-Limit` | Monthly character limit for your plan | `1000000` |
| `X-RateLimit-Remaining` | Characters remaining this period | `847250` |
| `X-RateLimit-Reset` | When the quota resets (ISO 8601) | `2026-04-01T00:00:00.000Z` |

Read rate limit headers from any API response to track your monthly character usage. These headers are present on every successful response:

**TypeScript**
```typescript
import { MurmrClient, isSyncResponse } from '@murmr/sdk';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

const result = await client.speech.create({
  input: 'Check remaining quota.',
  voice: 'voice_abc123',
});

if (isSyncResponse(result)) {
  const remaining = parseInt(result.headers.get('X-RateLimit-Remaining') || '0');
  const limit = parseInt(result.headers.get('X-RateLimit-Limit') || '0');
  const resetAt = result.headers.get('X-RateLimit-Reset');

  console.log(`${remaining.toLocaleString()}/${limit.toLocaleString()} chars remaining`);
  console.log(`Resets at: ${resetAt}`);
}
```

Check usage via raw HTTP headers in Python. The `X-RateLimit-Remaining` header shows characters left in your current billing period:

**Python**
```python
import os
import httpx

response = httpx.post(
    "https://api.murmr.dev/v1/voices/design",
    headers={"Authorization": f"Bearer {os.environ['MURMR_API_KEY']}"},
    json={
        "input": "Quick usage check.",
        "voice_description": "A calm voice",
        "language": "English",
    },
)

remaining = response.headers.get("X-RateLimit-Remaining")
limit = response.headers.get("X-RateLimit-Limit")
reset = response.headers.get("X-RateLimit-Reset")

print(f"Usage: {int(limit) - int(remaining)}/{limit} characters")
print(f"Resets: {reset}")
```

## Concurrent Request Limits

Each plan limits how many requests can be in-flight simultaneously. If you exceed the concurrent limit, the server returns `429` with concurrent limit metadata.

When the concurrent limit is exceeded, the API returns a 429 with `concurrent_limit_exceeded` code. Use the response headers to see how many requests are active:

```json
{
  "error": {
    "type": "rate_limit_error",
    "code": "concurrent_limit_exceeded",
    "message": "Too many concurrent requests. Your plan allows 5 concurrent requests."
  }
}
```

The `X-Concurrent-Limit` and `X-Concurrent-Active` headers show your plan's limit and current in-flight count:

```
X-Concurrent-Limit: 5
X-Concurrent-Active: 5
```

## WebSocket Limits

The Realtime WebSocket endpoint has additional limits:

| Limit | Value | Scope |
|-------|-------|-------|
| Concurrent connections | 10 | Per API key |
| Generations per minute | 100 | Per API key |
| Auth timeout | 10 seconds | Per connection |
| Max buffer size | 4,096 chars | Per connection |

## Handling 429 Errors

Retry with exponential backoff when rate-limited, but do not retry monthly limit exhaustion (upgrade your plan instead):

**TypeScript**
```typescript
import { MurmrClient, MurmrError } from '@murmr/sdk';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

async function generateWithRetry(input: string, voice: string, maxRetries = 3) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await client.speech.create({ input, voice });
    } catch (err) {
      if (err instanceof MurmrError && err.status === 429) {
        if (err.code === 'monthly_limit') {
          throw err; // Do not retry monthly limits
        }
        if (attempt === maxRetries) throw err;

        const delay = 1000 * Math.pow(2, attempt);
        console.log(`Rate limited. Retrying in ${delay}ms...`);
        await new Promise(r => setTimeout(r, delay));
        continue;
      }
      throw err;
    }
  }
}
```

Same retry pattern in Python. The `e.code` field distinguishes concurrent limits (retryable) from monthly quota exhaustion (not retryable):

**Python**
```python
import os
import time
from murmr import MurmrClient, MurmrError

def generate_with_backoff(
    client: MurmrClient,
    text: str,
    voice: str,
    max_retries: int = 5,
) -> bytes:
    for attempt in range(max_retries + 1):
        try:
            result = client.speech.create_and_wait(input=text, voice=voice)
            if hasattr(result, "audio"):
                return result.audio
            return result.audio_bytes
        except MurmrError as e:
            if e.status != 429 or attempt == max_retries:
                raise
            if e.code == "rate_limit_exceeded":
                raise  # Monthly limit -- no point retrying

            delay = min(2 ** attempt, 60)
            print(f"Rate limited ({e.code}). Retrying in {delay}s...")
            time.sleep(delay)

    raise MurmrError("Max retries exceeded")

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    audio = generate_with_backoff(client, "Hello world", "voice_abc123def456")
```

## Batch Processing with Concurrency Control

When processing many items, use a semaphore to stay within your plan's concurrent request limit. This prevents 429 errors and ensures smooth throughput:

**Python (async)**
```python
import asyncio
import os
from murmr import AsyncMurmrClient

async def batch_generate(
    texts: list[str],
    voice: str,
    max_concurrent: int = 3,
) -> list[bytes]:
    semaphore = asyncio.Semaphore(max_concurrent)
    results: list[bytes] = [b""] * len(texts)

    async def process_one(index: int, text: str):
        async with semaphore:
            async with AsyncMurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
                result = await client.speech.create_and_wait(
                    input=text,
                    voice=voice,
                    response_format="mp3",
                )
                if hasattr(result, "audio"):
                    results[index] = result.audio
                else:
                    results[index] = result.audio_bytes

    tasks = [process_one(i, t) for i, t in enumerate(texts)]
    await asyncio.gather(*tasks)
    return results

texts = [
    "First notification message.",
    "Second notification message.",
    "Third notification message.",
]

asyncio.run(batch_generate(texts, "voice_abc123def456", max_concurrent=3))
```

## Monthly Reset

- Usage resets on the first day of each calendar month at 00:00 UTC
- Unused characters do not roll over to the next month
- Upgrading your plan takes effect immediately with the new limits
- The exact reset timestamp is in the `X-RateLimit-Reset` header

## See Also

- [Authentication](./authentication.md) -- Plan details and API key management
- [Errors](./errors.md) -- Full error reference including 429 sub-types
- [Speech Generation](./speech.md) -- Async jobs for background processing
