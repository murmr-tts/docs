# Installation

Get started with the murmr TTS API. Choose your integration method: REST API with any HTTP client, or use the official Node.js or Python SDK for built-in streaming, long-form chunking, and error recovery.

## REST API (No SDK)

You only need an API key and an HTTP client. No SDK installation required.

### Get Your API Key

Sign up at [murmr.dev](https://murmr.dev) to get your API key. Keys follow the format `murmr_sk_live_xxx`. The Free plan includes 50,000 characters per month.

```bash
export MURMR_API_KEY="murmr_sk_live_your_key_here"
```

### Verify with curl

```bash
curl -X POST https://api.murmr.dev/v1/voices/design \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Hello from murmr!",
    "voice_description": "A warm, friendly narrator",
    "language": "English"
  }' \
  --output hello.wav

# Play the audio
afplay hello.wav    # macOS
aplay hello.wav     # Linux
```

If you hear audio, your API key is working. See the [API reference](./api-reference.md) for all available endpoints.

## Node.js SDK (@murmr/sdk)

TypeScript-first with zero runtime dependencies. Handles auth, streaming, long-form chunking, and error recovery automatically.

### Requirements

- **Node.js 18+** (uses native `fetch` and `AbortSignal.timeout`)
- A **murmr API key** ([sign up at murmr.dev](https://murmr.dev))

### Install

```bash
npm install @murmr/sdk
```

Or with your preferred package manager:

```bash
pnpm add @murmr/sdk
yarn add @murmr/sdk
```

### Set Your API Key

Store your API key as an environment variable. Never hardcode it.

```bash
# .env or shell profile
export MURMR_API_KEY="murmr_sk_live_..."
```

### Initialize the Client

```typescript
import { MurmrClient } from '@murmr/sdk';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `apiKey` | `string` | (required) | Your murmr API key. Sent as a Bearer token. |
| `baseUrl` | `string` | `https://api.murmr.dev` | Override the API base URL. |
| `timeout` | `number` | `300000` | Request timeout in milliseconds (5 min default). |

### Verify It Works

```typescript
import { MurmrClient } from '@murmr/sdk';
import { writeFileSync } from 'node:fs';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

const wav = await client.voices.design({
  input: 'Hello from murmr!',
  voice_description: 'A warm, friendly narrator',
  language: 'English',
});

writeFileSync('hello.wav', wav);
console.log(`Generated ${wav.length} bytes`);
```

### Project Structure

The client exposes three resource namespaces:

- **`client.speech`** -- Generate audio from text (batch, streaming, long-form)
- **`client.voices`** -- Design voices, save/list/delete, extract embeddings
- **`client.jobs`** -- Track async batch jobs

See the full [Node.js SDK Reference](./sdk-reference-node.md) for all methods and parameters.

## Python SDK (murmr)

Async-first with sync support. Built on `httpx` and `pydantic` v2.

### Requirements

- **Python 3.9+**
- Dependencies (installed automatically): `httpx`, `pydantic` v2

### Install

```bash
pip install murmr
```

### Set Your API Key

```bash
export MURMR_API_KEY="murmr_sk_live_your_key_here"
```

### Initialize the Client

Two client classes are available. Both support context managers for automatic cleanup.

**Sync client:**

```python
import os
from murmr import MurmrClient

client = MurmrClient(api_key=os.environ["MURMR_API_KEY"])

# Or as a context manager (recommended)
with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    wav = client.voices.design(
        input="Hello!",
        voice_description="A warm, friendly voice",
    )
```

**Async client:**

```python
import os
from murmr import AsyncMurmrClient

async with AsyncMurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    wav = await client.voices.design(
        input="Hello!",
        voice_description="A warm, friendly voice",
    )
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `api_key` | `str` | (required) | Your murmr API key. Sent as a Bearer token. |
| `base_url` | `str` | `https://api.murmr.dev` | Override the API base URL. |
| `timeout` | `float` | `300.0` | Request timeout in seconds (5 min default). |

### Verify It Works

```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    wav = client.voices.design(
        input="murmr is working.",
        voice_description="A clear, professional voice",
        language="English",
    )

    with open("test.wav", "wb") as f:
        f.write(wav)

    print(f"Success! Wrote {len(wav)} bytes to test.wav")
```

### Project Structure

Both sync and async clients expose three resource namespaces:

| Namespace | Description |
|-----------|-------------|
| `client.speech` | Generate audio from text with saved voices |
| `client.voices` | Design voices with natural language, manage saved voices |
| `client.jobs` | Track async batch jobs |

See the full [Python SDK Reference](./sdk-reference-python.md) for all methods and parameters.

## OpenAI Compatibility

murmr is a drop-in replacement for OpenAI's TTS API. Change the base URL and API key to migrate:

**Node.js (OpenAI SDK):**

```typescript
import OpenAI from 'openai';

const client = new OpenAI({
  apiKey: process.env.MURMR_API_KEY,
  baseURL: 'https://api.murmr.dev',
});

const response = await client.audio.speech.create({
  model: 'murmr-tts',         // ignored, but required by OpenAI SDK
  voice: 'voice_abc123',       // your saved voice ID
  input: 'Hello from murmr!',
});
```

**Python (OpenAI SDK):**

```python
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["MURMR_API_KEY"],
    base_url="https://api.murmr.dev",
)

response = client.audio.speech.create(
    model="murmr-tts",
    voice="voice_abc123",
    input="Hello from murmr!",
)
response.stream_to_file("output.wav")
```

See [OpenAI Migration](./openai-migration.md) for a full migration guide.

## Next Steps

- [Streaming](./streaming.md) -- Real-time audio streaming via SSE
- [Voice Design](./voicedesign.md) -- Create voices with natural language descriptions
- [Long-Form Generation](./long-form.md) -- Generate audio for text of any length
- [Errors](./errors.md) -- Error handling and retry strategies
