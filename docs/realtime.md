# Real-time WebSocket

Bidirectional WebSocket API for voice agents and LLM integration with intelligent text buffering.

## When to Use WebSocket

### ✓ Ideal For

- Voice agents / assistants
- LLM streaming output (ChatGPT-style)
- Real-time translation
- Interactive phone systems (IVR)
- Any bidirectional communication

### Consider SSE Instead

- Simple demos / previews
- One-shot generation
- Mobile apps (simpler integration)
- No text buffering needed

## Connection Methods

Choose the right method for your use case:

| Method | Bidirectional | Best For |
| --- | --- | --- |
| HTTP Batch | No | Audiobooks, bulk generation |
| SSE Streaming | No | Demos, previews, mobile |
| WebSocket | Yes | Voice agents, LLM integration |

> **Text Buffering:** WebSocket uniquely supports intelligent text buffering — accumulate tokens from an LLM and generate at natural sentence boundaries for smoother speech.

## Endpoint

`wss://api.murmr.dev/v1/realtime`

After connecting, send a `config` message with your API key within 10 seconds. Time-to-first-chunk is typically ~460ms server-side.

> **⚠️ Plan Requirement**
> WebSocket is available on **Realtime** and **Scale** plans. See [Pricing](./pricing.md) for plan details.

## Quick Example

Basic WebSocket flow with VoiceDesign:

## Key Features

Text Buffering

Smart sentence boundary detection

Binary Mode

Raw PCM for lower latency

API Key Caching

Faster auth after first request

Low Latency

Fast time-to-first-chunk

## Learn More

[WebSocket Protocol Message types and connection flow](./websocket-protocol.md)

[JavaScript Client Vanilla JS examples with Web Audio playback](./installation.md)

[SSE Streaming Simpler alternative for one-shot generation](./streaming.md)

## Rate Limits

WebSocket connections are subject to these limits:

10

Concurrent connections per API key

100

Generations per minute per key

Plan

Character usage counts against quota
