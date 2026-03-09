# Authentication

All requests to the murmr API require an API key. This guide covers key formats, authentication methods, usage limits, and security best practices.

## API Key Format

murmr API keys follow a predictable format:

| Prefix | Environment | Example |
|--------|-------------|---------|
| `murmr_sk_live_` | Production | `murmr_sk_live_abc123def456...` |
| `murmr_sk_test_` | Testing | `murmr_sk_test_abc123def456...` |

Test keys have the same rate limits as your plan but do not count toward your monthly character usage. Use test keys during development and CI.

## Bearer Token Authentication

Every HTTP API request requires the `Authorization` header with a Bearer token.

Pass your API key as a Bearer token in the `Authorization` header on every request:

**curl**
```bash
curl -X POST https://api.murmr.dev/v1/voices/design \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"text": "Hello world", "voice_description": "A calm voice"}'
```

The SDKs handle the `Authorization` header automatically. Just pass your key at initialization:

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});
// The SDK sets the Authorization header automatically.
```

**Python**
```python
import os
from murmr import MurmrClient

client = MurmrClient(api_key=os.environ["MURMR_API_KEY"])
# The SDK sends the key as a Bearer token automatically.
```

### WebSocket Connections

For the realtime WebSocket endpoint, connect without credentials and send your API key in the first `config` message within 10 seconds. The key is sent over the encrypted WebSocket connection, not as a URL parameter:

```
wss://api.murmr.dev/v1/realtime
```

```json
{
  "type": "config",
  "api_key": "murmr_sk_live_your_key_here",
  "voice_description": "A warm, friendly voice",
  "language": "English"
}
```

> Do not pass the API key as a query parameter -- query parameters may be logged by proxies and CDNs.

## Environment Variables

Never commit API keys to source control. Use environment variables:

```bash
# .env file (add to .gitignore)
MURMR_API_KEY=murmr_sk_live_your_key_here
```

Always load API keys from environment variables. Hardcoding keys risks leaking them in version control:

**TypeScript**
```typescript
// Correct: environment variable
const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

// Wrong: hardcoded key
const client = new MurmrClient({
  apiKey: 'murmr_sk_live_abc123...',  // DO NOT DO THIS
});
```

Validate that the environment variable is set before creating the client to get a clear error message at startup:

**Python**
```python
import os
from murmr import MurmrClient

api_key = os.environ.get("MURMR_API_KEY")
if not api_key:
    raise RuntimeError("MURMR_API_KEY environment variable is not set")

client = MurmrClient(api_key=api_key)
```

## Usage Limit Headers

Every API response includes headers that report your current usage:

| Header | Description | Example |
|--------|-------------|---------|
| `X-RateLimit-Limit` | Monthly character limit for your plan | `1000000` |
| `X-RateLimit-Remaining` | Characters remaining this period | `847250` |
| `X-RateLimit-Reset` | When the quota resets (ISO 8601) | `2026-04-01T00:00:00.000Z` |

Read the rate limit headers from any API response to monitor your monthly character usage:

**TypeScript**
```typescript
import { MurmrClient, isSyncResponse } from '@murmr/sdk';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

const result = await client.speech.create({
  input: 'Check my usage.',
  voice: 'voice_abc123',
});

if (isSyncResponse(result)) {
  const remaining = result.headers.get('X-RateLimit-Remaining');
  const limit = result.headers.get('X-RateLimit-Limit');
  const reset = result.headers.get('X-RateLimit-Reset');
  console.log(`${remaining}/${limit} characters remaining (resets ${reset})`);
}
```

Check usage via raw HTTP response headers. The `X-RateLimit-Reset` header shows when your quota resets (ISO 8601):

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
print(f"Usage: {int(limit) - int(remaining)}/{limit} characters (resets {reset})")
```

## Plans and Limits

| Plan | Price | Characters/Month | Saved Voices | Concurrent Requests | API Keys | WebSocket |
|------|-------|-------------------|--------------|---------------------|----------|-----------|
| Free | $0 | 50,000 | 3 | 2 | 1 | No |
| Starter | $10 | 1,000,000 | 10 | 5 | 3 | No |
| Pro | $25 | 3,000,000 | 25 | 10 | 5 | No |
| Realtime | $49 | 3,000,000 | 50 | 10 | 10 | Yes |
| Scale | $99 | 10,000,000 | 100 | 25 | 25 | Yes |

> The Realtime plan includes WebSocket access (`/v1/realtime`). All other plans are limited to HTTP endpoints. See [Rate Limits](./rate-limits.md) for concurrent request limits and overage pricing.

## Security Best Practices

### Server-Side Only

API keys should only be used in server-side code. Never expose them in client-side JavaScript, mobile apps, or public repositories. If you need browser playback, proxy requests through your backend.

### Separate Keys by Environment

Use different API keys for development, staging, and production. Rotate keys immediately if one is exposed.

### Rotate Compromised Keys

If a key is exposed:

1. Generate a new key in the [murmr dashboard](https://murmr.dev/en/dashboard/api-keys)
2. Update your environment variables
3. Delete the compromised key
4. Check your usage logs for unauthorized activity

## Client Configuration

Both SDKs accept these options:

| Option | TypeScript | Python | Default | Description |
|--------|-----------|--------|---------|-------------|
| API Key | `apiKey` | `api_key` | (required) | Your murmr API key |
| Base URL | `baseUrl` | `base_url` | `https://api.murmr.dev` | API base URL |
| Timeout | `timeout` | `timeout` | `300000ms` / `300.0s` | Request timeout |

Override the default 5-minute timeout for faster failure detection. Set a shorter timeout for interactive applications:

**TypeScript**
```typescript
const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
  timeout: 60_000, // 1 minute timeout
});
```

**Python**
```python
client = MurmrClient(
    api_key=os.environ["MURMR_API_KEY"],
    timeout=60.0,  # 60 second timeout
)
```

## See Also

- [Quickstart](./quickstart.md) -- Get started in 5 minutes
- [Rate Limits](./rate-limits.md) -- Concurrent limits and overage pricing
- [Errors](./errors.md) -- Handling 401 and 429 responses
