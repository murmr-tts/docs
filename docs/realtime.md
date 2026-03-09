# Real-time WebSocket

Bidirectional WebSocket API for voice agents and LLM integration with intelligent text buffering. The lowest-latency path for text-to-speech.

## When to Use WebSocket vs SSE

| Method | Bidirectional | Best For |
|--------|---------------|----------|
| HTTP Batch | No | Audiobooks, bulk generation |
| SSE Streaming | No | Demos, previews, mobile apps |
| WebSocket | Yes | Voice agents, LLM integration, real-time translation |

WebSocket uniquely supports intelligent text buffering -- accumulate tokens from an LLM and generate speech at natural sentence boundaries for smoother prosody.

## Endpoint

```
wss://api.murmr.dev/v1/realtime
```

After connecting, send a `config` message with your API key within 10 seconds. Time-to-first-chunk is typically ~460ms server-side.

> **Plan requirement:** WebSocket is available on **Realtime** ($49/mo) and **Scale** ($99/mo) plans.

## Connection Lifecycle

1. **Connect** to `wss://api.murmr.dev/v1/realtime`
2. **Send `config`** with API key and voice settings
3. **Receive `config_ack`** -- connection is authenticated and ready, you can start sending text
4. **Send `text` messages** -- server buffers and generates at natural boundaries
5. **Send `flush`** -- forces generation of remaining buffered text
6. **Receive `audio` chunks** -- base64 JSON or raw binary PCM (24kHz, 16-bit, mono)
7. **Receive `done`** -- generation complete for the current utterance
8. Repeat steps 4-7 for multiple utterances on the same connection

## Quick Example

**TypeScript**
```typescript
import WebSocket from 'ws';

const ws = new WebSocket('wss://api.murmr.dev/v1/realtime');

ws.on('open', () => {
  ws.send(JSON.stringify({
    type: 'config',
    api_key: process.env.MURMR_API_KEY,
    voice_description: 'A warm, friendly voice',
    language: 'English',
  }));
});

ws.on('message', (data, isBinary) => {
  if (isBinary) {
    // Binary mode: raw PCM audio
    const pcm = data as Buffer;
    return;
  }

  const msg = JSON.parse(data.toString());

  if (msg.type === 'config_ack') {
    ws.send(JSON.stringify({ type: 'text', text: 'Hello, world!' }));
    ws.send(JSON.stringify({ type: 'flush' }));
  }

  if (msg.type === 'audio') {
    const pcm = Buffer.from(msg.chunk, 'base64');
    // PCM: 24kHz, 16-bit signed LE, mono
  }

  if (msg.type === 'done') {
    console.log(`TTFC: ${msg.first_chunk_latency_ms}ms`);
  }
});
```

**Python**
```python
import asyncio
import base64
import json
import os
import websockets

async def realtime_tts():
    api_key = os.environ["MURMR_API_KEY"]
    uri = "wss://api.murmr.dev/v1/realtime"

    async with websockets.connect(uri) as ws:
        # Configure session
        await ws.send(json.dumps({
            "type": "config",
            "api_key": api_key,
            "voice_description": "A warm, friendly voice",
            "language": "English",
        }))

        ack = json.loads(await ws.recv())
        assert ack["type"] == "config_ack"

        # Send text and flush
        await ws.send(json.dumps({"type": "text", "text": "Hello, world!"}))
        await ws.send(json.dumps({"type": "flush"}))

        # Receive audio chunks
        pcm_parts = []
        async for message in ws:
            if isinstance(message, bytes):
                pcm_parts.append(message)
            else:
                data = json.loads(message)
                if data["type"] == "audio":
                    pcm_parts.append(base64.b64decode(data["chunk"]))
                elif data["type"] == "done":
                    print(f"TTFC: {data['first_chunk_latency_ms']}ms")
                    break

        audio = b"".join(pcm_parts)

asyncio.run(realtime_tts())
```

## Key Features

- **Text Buffering** -- Smart sentence boundary detection for natural prosody
- **Binary Mode** -- Raw PCM frames for ~50ms lower latency per chunk
- **Session Auth** -- `config_ack` confirms the connection is authenticated and ready
- **Multiple Generations** -- One connection supports multiple text-to-audio cycles

## Rate Limits

| Limit | Value |
|-------|-------|
| Concurrent WebSocket connections | 10 per API key |
| Generations per minute | 100 per API key |
| Global connections | 500 total (server-wide) |

Character usage on WebSocket counts against your plan's monthly character quota.

## See Also

- [WebSocket Protocol](./websocket-protocol.md) -- Full message type reference, close codes
- [Browser Client](./browser-client.md) -- Web Audio playback, React hook, LLM integration
- [Voice Agents](./voice-agents.md) -- Building conversational voice agents
