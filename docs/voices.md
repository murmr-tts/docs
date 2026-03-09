# Voice Management

Save voices for consistent, repeatable speech generation. A saved voice captures the vocal characteristics from a Voice Design generation so you can reuse the same voice across requests without re-describing it.

## Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/voices` | GET | List all saved voices |
| `/v1/voices` | POST | Save a new voice |
| `/v1/voices/{voice_id}` | DELETE | Delete a saved voice |
| `/v1/voices/extract-embeddings` | POST | Extract portable embeddings from audio |

## Workflow

1. Generate audio with [VoiceDesign](./voicedesign.md)
2. Save the voice with `POST /v1/voices`
3. Use the voice ID with [/v1/audio/speech](./speech.md)

## Save a Voice

`POST /v1/voices`

Saving a voice requires reference audio (a WAV buffer from a Voice Design call) and its transcript. The API extracts voice embeddings from the audio server-side.

### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `name` | string | Yes | -- | Display name (1-50 characters) |
| `audio` | string | Yes | -- | Base64-encoded WAV audio. SDKs accept binary and encode automatically. |
| `description` | string | Yes | -- | Description of the voice for your reference |
| `ref_text` | string | Yes | -- | Transcript of the reference audio. Improves embedding quality. |
| `language` | string | No | `English` | Language name |

Save a voice using curl by base64-encoding the reference WAV file and sending it with the transcript. The API extracts voice embeddings server-side:

**curl**
```bash
# First, generate audio with VoiceDesign and save the WAV file
# Then base64 encode it
AUDIO_B64=$(base64 -i sample.wav)

curl -X POST "https://api.murmr.dev/v1/voices" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"My Narrator\",
    \"description\": \"A warm, professional voice\",
    \"audio\": \"$AUDIO_B64\",
    \"ref_text\": \"Sample audio text here.\",
    \"language\": \"English\"
  }"
```

Two-step workflow with the Node.js SDK: generate reference audio with Voice Design, then save it. The `ref_text` must match the spoken audio for accurate embedding extraction:

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

// Generate reference audio
const inputText = 'This is the reference recording for my new voice.';
const wav = await client.voices.design({
  input: inputText,
  voice_description: 'A soothing female narrator, mid-30s, neutral American accent',
});

// Save the voice
const saved = await client.voices.save({
  name: 'Soothing Narrator',
  description: 'Female narrator, mid-30s, neutral American, for audiobooks',
  audio: wav,
  ref_text: inputText,
  language: 'English',
});

console.log(`ID: ${saved.id}`);
console.log(`Embedding size: ${saved.prompt_size_bytes} bytes`);
```

Same workflow in Python. The SDK accepts raw `bytes` for the `audio` parameter and handles base64 encoding automatically:

**Python (sync)**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    text = "This is my reference sample for voice extraction."
    description = "A warm, confident male voice with a slight rasp"

    wav = client.voices.design(
        input=text,
        voice_description=description,
        language="English",
    )

    saved = client.voices.save(
        name="Confident Male",
        audio=wav,
        description=description,
        ref_text=text,
        language="English",
    )

    print(f"Voice ID: {saved.id}")
    print(f"Prompt size: {saved.prompt_size_bytes} bytes")
```

**Python (async)**
```python
import asyncio
import os
from murmr import AsyncMurmrClient

async def main():
    async with AsyncMurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
        text = "This is my reference sample for voice extraction."
        description = "A warm, confident male voice with a slight rasp"

        wav = await client.voices.design(
            input=text,
            voice_description=description,
        )

        saved = await client.voices.save(
            name="Confident Male",
            audio=wav,
            description=description,
            ref_text=text,
        )

        print(f"Voice ID: {saved.id}")

asyncio.run(main())
```

### Save Response

A successful save returns the new voice ID, embedding size, and a confirmation. Use the `id` field in subsequent TTS requests:

```json
{
  "id": "voice_a1b2c3d4e5f6",
  "name": "Soothing Narrator",
  "language": "English",
  "description": "Female narrator, mid-30s, neutral American, for audiobooks",
  "prompt_size_bytes": 142380,
  "created_at": "2026-03-01T12:00:00Z",
  "success": true,
  "has_audio_preview": true
}
```

## List Saved Voices

`GET /v1/voices`

Returns all saved voices for your account along with plan limits.

List all saved voices with curl. Returns an array of voice metadata plus your plan's saved voice limits:

**curl**
```bash
curl "https://api.murmr.dev/v1/voices" \
  -H "Authorization: Bearer $MURMR_API_KEY"
```

Check how many voice slots you've used and iterate over saved voices:

**TypeScript**
```typescript
const { voices, saved_count, saved_limit } = await client.voices.list();

console.log(`Using ${saved_count}/${saved_limit} voice slots`);
for (const voice of voices) {
  console.log(`${voice.id}: ${voice.name} (${voice.language})`);
}
```

**Python**
```python
response = client.voices.list_voices()

print(f"Saved: {response.saved_count}/{response.saved_limit}")
for voice in response.voices:
    print(f"  {voice.id}: {voice.name} ({voice.language})")
```

### List Response

```json
{
  "voices": [
    {
      "id": "voice_a1b2c3d4e5f6",
      "name": "Soothing Narrator",
      "description": "Female narrator, mid-30s, neutral American, for audiobooks",
      "language": "English",
      "language_name": "English",
      "audio_preview_url": "https://...",
      "created_at": "2026-03-01T12:00:00Z"
    }
  ],
  "saved_count": 1,
  "saved_limit": 10,
  "total": 1
}
```

## Delete a Voice

`DELETE /v1/voices/:id`

Permanently deletes a saved voice.

Delete a voice by ID. This is permanent and frees one slot toward your plan's limit:

**curl**
```bash
curl -X DELETE "https://api.murmr.dev/v1/voices/voice_a1b2c3d4e5f6" \
  -H "Authorization: Bearer $MURMR_API_KEY"
```

**TypeScript**
```typescript
const result = await client.voices.delete('voice_a1b2c3d4e5f6');
console.log(result.message); // "Voice deleted successfully"
```

**Python**
```python
result = client.voices.delete("voice_a1b2c3d4e5f6")
print(f"Deleted: {result.success}")
```

### Delete Response

```json
{
  "success": true,
  "id": "voice_a1b2c3d4e5f6",
  "message": "Voice \"Soothing Narrator\" deleted"
}
```

## Extract Embeddings

`POST /v1/voices/extract-embeddings`

Extract portable voice embeddings from audio without saving the voice. The returned `prompt_data` can be stored in your own database and passed as `voice_clone_prompt` in any TTS request -- no saved voice ID needed, and it does not count against your saved voice limit.

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `audio` | string | Yes | Base64-encoded WAV audio. SDKs accept binary. |
| `ref_text` | string | Yes | Transcript of the reference audio. Required for accurate embedding extraction. |

### Response

| Field | Type | Description |
|-------|------|-------------|
| `prompt_data` | string | Base64-encoded voice embedding data. Pass this as `voice_clone_prompt` in TTS requests. |
| `prompt_size_bytes` | number | Size of the embedding data in bytes (typically 50-200KB). |

Extract embeddings from reference audio via curl. Send base64-encoded WAV audio and the transcript:

**curl**
```bash
curl -X POST "https://api.murmr.dev/v1/voices/extract-embeddings" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"audio": "<base64-wav>", "ref_text": "Transcript of the audio."}'
```

Extract embeddings with the Node.js SDK and use them inline for TTS. The `voice` parameter is required but ignored when `voice_clone_prompt` is provided:

**TypeScript**
```typescript
import { MurmrClient } from '@murmr/sdk';
import { readFileSync } from 'node:fs';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

const audioBuffer = readFileSync('reference.wav');

const { prompt_data, prompt_size_bytes } = await client.voices.extractEmbeddings({
  audio: audioBuffer,
  ref_text: 'The transcript of the reference audio goes here.',
});

console.log(`Embedding size: ${prompt_size_bytes} bytes`);

// Store prompt_data in your database, then use it in requests:
const stream = await client.speech.stream({
  input: 'Generate speech with the extracted embedding.',
  voice: 'unused', // Required field, but voice_clone_prompt takes precedence
  voice_clone_prompt: prompt_data,
});
```

Extract embeddings in Python, then use them directly in a TTS request without saving the voice. This avoids counting against your saved voice limit:

**Python**
```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    text = "A sample for embedding extraction."
    wav = client.voices.design(
        input=text,
        voice_description="A clear female voice",
    )

    embeddings = client.voices.extract_embeddings(
        audio=wav,
        ref_text=text,
    )

    print(f"Prompt size: {embeddings.prompt_size_bytes} bytes")

    # Use embeddings directly (no saved voice needed)
    result = client.speech.create_and_wait(
        input="Using extracted embeddings for voice cloning.",
        voice="unused",
        voice_clone_prompt=embeddings.prompt_data,
    )
```

> **When to use embeddings vs saved voices:** Use saved voices when you want murmr to store and manage the voice. Use extracted embeddings when you need to store voice data in your own system (e.g., multi-tenant apps), or when you want to avoid the saved voice limit on your plan.

## Voice Limits by Plan

| Plan | Saved Voice Limit |
|------|-------------------|
| Free | 3 |
| Starter | 10 |
| Pro | 25 |
| Realtime | 50 |
| Scale | 100 |

Attempting to save a voice beyond your limit returns an error. Delete unused voices or upgrade your plan to increase the limit.

## Voice ID Format

Voice IDs follow the pattern `voice_` followed by a unique identifier:

```
voice_abc123def456
voice_xyz789ghi012
```

Voice IDs are case-sensitive and must be used exactly as returned from the API.

## See Also

- [Voice Design](./voicedesign.md) -- Natural language voice descriptions
- [Speech Generation](./speech.md) -- Using saved voices in TTS requests
- [Authentication](./authentication.md) -- Plan limits and API keys
- [Rate Limits](./rate-limits.md) -- Voice save limits by plan
