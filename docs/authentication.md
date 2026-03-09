# Authentication

All API requests require authentication via an API key in the Authorization header.

## Getting an API Key

To get an API key, create an account and generate one from the dashboard:

1. Sign up or log in at [murmr.dev/dashboard](https://murmr.dev/en/dashboard)
2. Navigate to [API Keys](https://murmr.dev/en/dashboard/api-keys)
3. Click "Create new key" and give it a descriptive name
4. Copy your key immediately — it won't be shown again

> **⚠️ Keep your API key secure**
> Your API key grants access to your account and usage quota. Never share it publicly, commit it to version control, or expose it in client-side code.

## API Key Format

murmr API keys follow a structured format for easy identification:

```Text
murmr_sk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345   # Production key
murmr_sk_test_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345   # Test key (coming soon)
```

Keys are 32 alphanumeric characters after the prefix. The `murmr_sk_` prefix makes them easy to identify in code reviews and secret scanning tools.

## Using Your API Key

Include your API key in the `Authorization` header using the Bearer token format:

```HTTP Header
Authorization: Bearer murmr_sk_live_...
```

Complete request example:

```cURL
curl -X POST "https://api.murmr.dev/v1/voices/design" \
  -H "Authorization: Bearer murmr_sk_live_aBcDeFgHiJkLmNoPqRsT..." \
  -H "Content-Type: application/json" \
  -d '{"text": "Hello world", "voice_description": "A warm voice", "language": "English"}' \
  --output speech.wav
```

## Usage Limits

Usage limits vary by plan. When you exceed your monthly character limit, you'll receive a `429` error.

| Plan | Price | Monthly Characters |
| --- | --- | --- |
| Free | $0 | 50,000 |
| Starter | $10/mo | 1,000,000 |
| Pro | $25/mo | 3,000,000 |
| Realtime | $49/mo | 3,000,000 |
| Scale | $99/mo | 10,000,000 |

Realtime and Scale plans include WebSocket access for voice agent use cases. See the [Pricing page](./pricing.md) for overage rates and full plan details.

Usage information is included in response headers:

| Header | Description | Example |
| --- | --- | --- |
| X-RateLimit-Limit | Total characters allowed this period | 1000000 |
| X-RateLimit-Remaining | Characters remaining | 847293 |
| X-RateLimit-Reset | ISO 8601 timestamp when limit resets | 2026-03-01T00:00:00.000Z |

The reset timestamp is always the first of the next month at midnight UTC.

## Managing API Keys

You can create multiple API keys and revoke them at any time from the dashboard.

Multiple keys

— Create separate keys for different environments (dev, staging, prod)

Descriptive names

— Name keys by purpose (e.g., "Production App", "Development")

Instant revocation

— If a key is compromised, revoke it immediately from the dashboard

## Security Best Practices

**Do:** Store API keys in environment variables or a secrets manager

**Do:** Make API calls from your backend, never from client-side code

**Do:** Use different keys for development and production

**Don't:** Commit API keys to version control

**Don't:** Share keys via email, Slack, or other messaging

**Don't:** Expose keys in client-side JavaScript or mobile apps
