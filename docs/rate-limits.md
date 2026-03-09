# Rate Limits & Usage

murmr enforces rate limits per API key based on your plan. Limits reset monthly on your billing cycle date.

## Per-Plan Limits

| Plan | Characters/mo | Concurrent Requests | Saved Voices | API Keys | WebSocket |
| --- | --- | --- | --- | --- | --- |
| Free | 50,000 | 1 | 3 | 2 | No |
| Starter ($10) | 1,000,000 | 3 | 10 | 5 | No |
| Pro ($25) | 3,000,000 | 5 | 25 | 10 | No |
| Realtime ($49) | 3,000,000 | 5 | 50 | 10 | Yes |
| Scale ($99) | 10,000,000 | 10 | 100 | 25 | Yes |

> **ℹ️ Overage pricing**
> Paid plans can exceed their monthly character limit. Overage is billed at: Starter $15/1M, Pro $12/1M, Realtime $12/1M, Scale $10/1M. Free plans receive a `429` error when the limit is reached.

## Character Counting

Characters are counted from the `text` (or `input`) field of your request. Whitespace and punctuation count. The `voice_description` field does not count against your quota.

### Single request

A request with `"Hello, world!"` (13 characters) uses 13 characters from your quota.

### Long-form generation

A 10,000-character article split into 3 chunks uses 10,000 characters total, not 3 requests. Characters are counted per chunk.

### WebSocket

Each text buffer flush counts the characters in that chunk. A WebSocket session generating 5 flushes of ~100 characters each uses ~500 characters.

## Concurrent Request Limits

Each plan has a maximum number of in-flight requests per API key. If you exceed the limit, the API returns `429` with details about the limit.

```json
// 429 response for concurrent limit exceeded
{
  "error": {
    "type": "rate_limit_exceeded",
    "message": "Concurrent request limit exceeded",
    "concurrent_limit": 3,
    "concurrent_active": 3
  }
}
```

Wait for in-flight requests to complete before sending more, or upgrade your plan for higher concurrency.

## WebSocket Limits

WebSocket connections have additional rate limits beyond character quotas.

| Limit | Value | Scope |
| --- | --- | --- |
| Concurrent connections | 10 | Per API key |
| Generations per minute | 100 | Per API key |
| Auth timeout | 10 seconds | Per connection |
| Max buffer size | 4,096 chars | Per connection |

If you exceed the generation rate limit, the server sends an error message on the WebSocket (code `4004`) rather than closing the connection, so you can wait and retry.

## Rate Limit Headers

Every HTTP API response includes headers showing your current usage:

| Header | Description | Example |
| --- | --- | --- |
| X-RateLimit-Limit | Monthly character limit for your plan | 1000000 |
| X-RateLimit-Remaining | Characters remaining this period | 847250 |
| X-RateLimit-Reset | When the quota resets (ISO 8601) | 2026-03-01T00:00:00.000Z |

```typescript
// Check remaining quota from response headers
const response = await fetch("https://api.murmr.dev/v1/audio/speech", {
  method: "POST",
  headers: {
    "Authorization": "Bearer murmr_sk_live_...",
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ input: "Hello", voice: "voice_abc123" }),
});

const remaining = response.headers.get("X-RateLimit-Remaining");
const limit = response.headers.get("X-RateLimit-Limit");
const reset = response.headers.get("X-RateLimit-Reset");

console.log(`${remaining}/${limit} characters remaining (resets ${reset})`);
```

> Rate limit headers are only included on HTTP API responses. WebSocket connections do not include these headers. Use the [Dashboard usage page](./pricing.md) to monitor WebSocket usage.

## Handling 429 Errors

When you exceed a rate limit, the API returns HTTP `429 Too Many Requests`. Handle this with exponential backoff:

```typescript
import { MurmrClient, MurmrError } from '@murmr/sdk';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

async function generateWithRetry(input: string, voice: string, maxRetries = 3) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await client.speech.create({ input, voice });
    } catch (err) {
      if (err instanceof MurmrError && err.status === 429) {
        if (attempt === maxRetries) throw err;

        // Exponential backoff: 1s, 2s, 4s
        const delay = 1000 * Math.pow(2, attempt);
        console.log(`Rate limited. Retrying in ${delay}ms...`);
        await new Promise(r => setTimeout(r, delay));
        continue;
      }
      throw err; // Non-rate-limit error
    }
  }
}
```

For concurrent limit errors specifically, check the `concurrent_active` field in the error response to understand how many requests are in flight.

## Monthly Reset

Character quotas reset on the first of each month (UTC). The exact reset timestamp is included in the `X-RateLimit-Reset` header on every API response.

Unused characters do not roll over to the next month.

## See Also

- [Pricing](./pricing.md) — Plan details and overage pricing
- [Authentication](./authentication.md) — API key management
- [Error Reference](./errors.md) — All HTTP and WebSocket error codes
