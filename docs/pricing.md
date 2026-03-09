# Pricing & Usage

Simple, transparent pricing based on character count. No hidden fees.

## Plans

> **ℹ️ WebSocket Access**
> The [real-time WebSocket API](./realtime.md) is available on Realtime and Scale plans. All other endpoints (REST, SSE streaming, batch) work on every plan. All [audio formats](./audio-formats.md) are included on every plan.

Manage your subscription in the [Billing dashboard](https://murmr.dev/en/dashboard/billing).

## Character Counting

Usage is measured in characters of input text:

All characters count

— letters, numbers, spaces, punctuation

Maximum per request

— 4,096 characters per API call

Failed requests

— not counted against your quota

> **💡 Character estimation**
> Average English sentence: ~80-100 characters Average paragraph: ~500-700 characters 10,000 characters ≈ 2-3 pages of text

## Overage Pricing

If you exceed your monthly limit on a paid plan, overage charges apply based on your plan tier:

| Plan | Overage Rate |
| --- | --- |
| Starter | $15 per 1M characters |
| Pro | $12 per 1M characters |
| Realtime | $12 per 1M characters |
| Scale | $10 per 1M characters |

> Overage is only available on paid plans. Free tier users will receive a 429 error when quota is exceeded. Upgrade to continue usage or wait for the next billing period.

## Concurrent Request Limits

Each plan has a limit on simultaneous in-flight requests per user. If you exceed the limit, the API returns a `429` with a structured error until a current request completes.

| Plan | Concurrent Requests |
| --- | --- |
| Free | 1 |
| Starter | 3 |
| Pro | 5 |
| Realtime | 5 |
| Scale | 10 |

Batch jobs (`/v1/audio/speech` without streaming) are processed asynchronously and do not count toward the concurrent limit.

## Tracking Usage

Monitor your usage through multiple channels:

Dashboard

View usage charts and history on the [Usage page](https://murmr.dev/en/dashboard/usage)

Response headers

Every API response includes rate limit headers showing remaining quota

## Rate Limit Headers

| Header | Description |
| --- | --- |
| X-RateLimit-Limit | Total characters allowed this billing period |
| X-RateLimit-Remaining | Characters remaining in current period |
| X-RateLimit-Reset | Unix timestamp when the limit resets |

## OpenAI Comparison

murmr offers ~60% savings compared to OpenAI TTS:

| Provider | Price per 1M characters |
| --- | --- |
| OpenAI TTS-1 | $15.00 |
| OpenAI TTS-1-HD | $30.00 |
| murmr Starter (subscription) | $10.00 |
| murmr Pro (subscription) | $8.33 |
| murmr Scale (subscription) | $9.90 |
| murmr Scale (overage) | $10.00 |

Starter plan at $10/mo for 1M characters is the best value for most developers.
