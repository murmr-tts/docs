# Voice Agents

Build conversational voice agents by streaming LLM tokens directly into murmr's WebSocket API. Text goes in, audio comes out -- sub-600ms end-to-end.

## Architecture

A voice agent pipes LLM output tokens into murmr as they arrive. murmr buffers text to natural boundaries, generates speech, and streams audio back to the client for playback.

```
User speaks
    |
    v
+---------+     +----------+     +---------------+     +----------+
|  STT /  |---->|   LLM    |---->|  murmr WS     |---->|  Audio   |
|  Input  |     | (stream) |     |  /v1/realtime  |     | Playback |
+---------+     +----------+     +---------------+     +----------+
                  tokens            audio chunks
                  as they           as they're
                  arrive            generated
```

> **Plan requirement:** WebSocket access requires the **Realtime** ($49/mo) or **Scale** ($99/mo) plan.

## Integration Example

End-to-end voice agent that streams GPT-4o tokens into murmr's WebSocket for real-time audio output. The flow is: connect to murmr, authenticate, start the LLM stream, forward each token, flush at the end, and play audio as it arrives:

**TypeScript**
```typescript
import OpenAI from 'openai';
import WebSocket from 'ws';

const openai = new OpenAI();

// 1. Connect to murmr WebSocket
const ws = new WebSocket('wss://api.murmr.dev/v1/realtime');

ws.on('open', () => {
  // 2. Send config (auth + voice setup)
  ws.send(JSON.stringify({
    type: 'config',
    api_key: process.env.MURMR_API_KEY,
    voice_description: 'A calm, professional female voice',
    language: 'English',
  }));
});

ws.on('message', (data) => {
  const msg = JSON.parse(data.toString());

  if (msg.type === 'config_ack') {
    // 3. Start LLM stream after config acknowledged
    streamLLMResponse(ws);
  }

  if (msg.type === 'audio') {
    // 5. Play audio chunk (base64 PCM, 24kHz mono 16-bit)
    const pcm = Buffer.from(msg.chunk, 'base64');
    playAudio(pcm);
  }

  if (msg.type === 'done') {
    console.log(`Audio complete: ${msg.duration_ms}ms`);
  }
});

async function streamLLMResponse(ws: WebSocket) {
  const stream = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [{ role: 'user', content: 'Explain quantum computing briefly' }],
    stream: true,
  });

  // 4. Forward each token to murmr
  for await (const chunk of stream) {
    const token = chunk.choices[0]?.delta?.content;
    if (token) {
      ws.send(JSON.stringify({ type: 'text', text: token }));
    }
  }

  // Signal end of text
  ws.send(JSON.stringify({ type: 'flush' }));
}
```

Same voice agent in Python using `websockets` and the OpenAI async client. Tokens are forwarded as they arrive and audio chunks are collected:

**Python**
```python
import asyncio
import base64
import json
import os

import websockets
from openai import AsyncOpenAI

openai_client = AsyncOpenAI()

async def voice_agent(user_input: str):
    api_key = os.environ["MURMR_API_KEY"]
    uri = "wss://api.murmr.dev/v1/realtime"

    async with websockets.connect(uri) as ws:
        # 1. Send config (auth + voice setup)
        await ws.send(json.dumps({
            "type": "config",
            "api_key": api_key,
            "voice_description": "A calm, professional female voice",
            "language": "English",
        }))

        # 2. Wait for config_ack
        ack = json.loads(await ws.recv())
        assert ack["type"] == "config_ack"

        # 3. Start LLM stream and forward tokens
        stream = await openai_client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": user_input}],
            stream=True,
        )

        async for chunk in stream:
            token = chunk.choices[0].delta.content
            if token:
                await ws.send(json.dumps({"type": "text", "text": token}))

        # 4. Signal end of text
        await ws.send(json.dumps({"type": "flush"}))

        # 5. Receive audio chunks
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

asyncio.run(voice_agent("Explain quantum computing in one sentence."))
```

## Text Buffering

murmr doesn't generate audio for every token. It buffers incoming text and triggers generation at natural speech boundaries for optimal quality.

| Rule | Condition | Behavior |
|------|-----------|----------|
| Force flush | Buffer >= 200 chars | Generates immediately at best available boundary |
| Sentence flush | Buffer >= 50 chars + sentence end (`.!?`) | Generates at sentence boundary |
| Clause flush | Buffer >= 50 chars + clause end (`,;:`) | Generates at clause boundary |
| Explicit flush | Client sends `{"type":"flush"}` | Generates all buffered text immediately |

You can send tokens one at a time -- murmr accumulates them and generates speech when it has a meaningful phrase.

> **Always send a final flush.** After the LLM stream ends, send `{"type":"flush"}` to ensure any remaining buffered text is generated. Without this, trailing text like "Thank you" (no period, under 50 chars) stays buffered.

## Handling Interruptions

In a conversational agent, the user may interrupt while audio is still playing. Each WebSocket connection handles one conversation turn. When interrupted, close the current connection (which cancels in-progress generation server-side) and open a new one for the next turn:

**TypeScript**
```typescript
// When user starts speaking (interrupt detected):

// 1. Stop audio playback
audioPlayer.stop();

// 2. Close the current WebSocket connection
ws.close();

// 3. Open a new connection for the next response
const newWs = new WebSocket('wss://api.murmr.dev/v1/realtime');
// ... configure and start new LLM stream
```

Each conversation turn uses a fresh WebSocket connection. Closing the connection cancels any in-progress generation on the server:

**Python**
```python
async def create_turn(text: str):
    """One conversation turn = one WebSocket connection."""
    async with websockets.connect(uri) as ws:
        await ws.send(json.dumps({
            "type": "config",
            "api_key": api_key,
            "voice_description": "A helpful assistant voice",
        }))

        ack = json.loads(await ws.recv())
        assert ack["type"] == "config_ack"

        await ws.send(json.dumps({"type": "text", "text": text}))
        await ws.send(json.dumps({"type": "flush"}))

        async for message in ws:
            data = json.loads(message)
            if data["type"] == "done":
                break
```

The server cancels any in-progress generation on disconnect.

## Latency Expectations

| Metric | Typical Value | Notes |
|--------|---------------|-------|
| Server TTFC | ~550ms | Time from text received to first audio chunk generated |
| Client TTFC | ~600-700ms | Includes network round-trip |
| Binary mode savings | ~50ms | Skips base64 encoding overhead |
| Auth overhead | ~0ms | Parallel auth -- no blocking wait |
| Subsequent chunks | ~80ms apart | Continuous generation after first chunk |

Total voice agent latency = LLM TTFT + murmr TTFC + network. With a fast LLM (~300ms TTFT) and murmr (~600ms TTFC), expect ~900ms from user input to first audio -- well under the 1-second threshold for natural conversation.

## Best Practices

- **Use saved voices for consistency.** Pass `voice` (saved voice ID) or `voice_clone_prompt` instead of `voice_description`. Saved voices produce consistent audio across turns.
- **Send tokens immediately.** Don't wait for complete sentences from the LLM. Send each token as it arrives -- murmr's text buffer handles accumulation and boundary detection.
- **Always send a final flush.** After the LLM stream ends, send `{"type":"flush"}` to ensure any remaining buffered text is generated.
- **Monitor the done message.** The `done` message includes `first_chunk_latency_ms` and `duration_ms`. Log these to monitor performance in production.

## See Also

- [WebSocket Protocol](./websocket-protocol.md) -- Complete message type reference
- [Browser Client](./browser-client.md) -- Browser-side WebSocket with Web Audio API
- [Real-time Overview](./realtime.md) -- When to use WebSocket vs SSE
