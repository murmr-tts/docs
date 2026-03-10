# Voice Management

Save voices from VoiceDesign for guaranteed consistency. List and manage your saved voice library.

> **💡 Workflow**
> 1. Generate audio with [VoiceDesign](./voicedesign.md)
> 2. Save the voice with `POST /v1/voices`
> 3. Use the voice ID with [/v1/audio/speech](./speech.md)

`POST /v1/voices`

## Save a Voice

Extract voice embeddings from VoiceDesign audio and save for future use. Requires API key authentication.

### Request Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `name` | string | Yes | -- | Display name for the voice (1-50 characters) |
| `audio` | string | Yes | -- | Base64-encoded WAV audio from VoiceDesign generation |
| `description` | string | Yes | -- | Original voice description (stored for reference) |
| `ref_text` | string | Yes | -- | Transcript of the reference audio. Required for accurate embedding extraction. |
| `language` | string | No | English | Language of the voice (e.g., "English", "Spanish", "Japanese") |

**cURL**

```curl
# First, generate audio with VoiceDesign and save the WAV file
# Then base64 encode it
AUDIO_B64=$(base64 -i sample.wav)

curl -X POST "https://api.murmr.dev/v1/voices" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"My Narrator\",
    \"description\": \"A warm, professional voice\",
    \"audio\": \"$AUDIO_B64\",
    \"ref_text\": \"Sample audio text here.\",
    \"language\": \"English\"
  }"
```

**Node.js SDK**

```typescript
import { MurmrClient } from '@murmr/sdk';

const client = new MurmrClient({
  apiKey: process.env.MURMR_API_KEY!,
});

// Generate reference audio
const inputText = 'This is the reference recording for my new voice.';
const wav = await client.voices.design({
  input: inputText,
  voice_description: 'A soothing female narrator, mid-30s, warm and clear tone',
});

// Save the voice
const saved = await client.voices.save({
  name: 'Soothing Narrator',
  description: 'Female narrator, mid-30s, warm and clear, for audiobooks',
  audio: wav,
  ref_text: inputText,
  language: 'English',
});

console.log(`ID: ${saved.id}`);
console.log(`Embedding size: ${saved.prompt_size_bytes} bytes`);
```

**Python SDK**

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

### Response

```JSON
{
  "id": "voice_abc123def456",
  "name": "My Narrator",
  "description": "A warm, professional voice",
  "language": "English",
  "prompt_size_bytes": 51200,
  "created_at": "2025-01-29T12:00:00Z",
  "success": true,
  "has_audio_preview": true
}
```

`GET /v1/voices`

## List Saved Voices

Retrieve all voices saved to your account. Requires API key authentication.

**cURL**

```curl
curl "https://api.murmr.dev/v1/voices" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

**Node.js SDK**

```typescript
const { voices, saved_count, saved_limit } = await client.voices.list();

console.log(`Using ${saved_count}/${saved_limit} voice slots`);
for (const voice of voices) {
  console.log(`${voice.id}: ${voice.name} (${voice.language})`);
}
```

**Python SDK**

```python
response = client.voices.list_voices()

print(f"Saved: {response.saved_count}/{response.saved_limit}")
for voice in response.voices:
    print(f"  {voice.id}: {voice.name} ({voice.language})")
```

### Response

```JSON
{
  "voices": [
    {
      "id": "voice_abc123def456",
      "name": "My Narrator",
      "description": "A warm, professional voice",
      "language": "English",
      "language_name": "English",
      "audio_preview_url": "https://...",
      "created_at": "2025-01-29T12:00:00Z"
    },
    {
      "id": "voice_xyz789ghi012",
      "name": "Customer Support",
      "description": "Friendly and helpful",
      "language": "English",
      "language_name": "English",
      "audio_preview_url": null,
      "created_at": "2025-01-28T10:30:00Z"
    }
  ],
  "saved_count": 2,
  "saved_limit": 10,
  "total": 2
}
```

The `saved_limit` varies by plan. See limits below.

`DELETE /v1/voices/:id`

## Delete a Voice

Permanently delete a saved voice from your account. Requires API key authentication.

**cURL**

```curl
curl -X DELETE "https://api.murmr.dev/v1/voices/voice_abc123def456" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

**Node.js SDK**

```typescript
const result = await client.voices.delete('voice_abc123def456');
console.log(result.message); // "Voice deleted successfully"
```

**Python SDK**

```python
result = client.voices.delete("voice_abc123def456")
print(f"Deleted: {result.success}")
```

### Response

```JSON
{
  "success": true,
  "id": "voice_abc123def456",
  "message": "Voice \"My Narrator\" deleted"
}
```

`POST /v1/voices/extract-embeddings`

## Extract Embeddings

> **💡 Building a multi-tenant app?**
> Use [Portable Voice Embeddings](./portable-embeddings.md) to store voice data in your own database and pass it via `voice_clone_prompt` — no saved voice IDs needed.

Extract voice embeddings from audio. This endpoint goes through the Worker at `api.murmr.dev` and uses API key auth.

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `audio` | string | Yes | -- | Base64-encoded WAV audio to extract embeddings from |
| `ref_text` | string | Yes | -- | Transcript of the reference audio. Required for accurate embedding extraction. |
| `model` | string | No | base | Model to use: "base" (default) or "base_06b" |

```cURL
curl -X POST "https://api.murmr.dev/v1/voices/extract-embeddings" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"audio": "<base64-wav>", "ref_text": "Transcript of the audio."}'
```

## Voice ID Format

Voice IDs follow the pattern `voice_` followed by a unique identifier:

```Example
voice_abc123def456
voice_xyz789ghi012
```

Voice IDs are case-sensitive and must be used exactly as returned from the API.

## Saved Voice Limits

The number of voices you can save depends on your plan:

Plan

Saved Voices

Free

3

Starter

10

Pro

25

Realtime

50

Scale

100

## Error Responses

| Status | Error | Description |
|--------|-------|-------------|
| 400 | Bad Request | Missing name or audio, invalid base64 encoding |
| 404 | Not Found | Voice ID doesn't exist or doesn't belong to your account |
| 409 | Conflict | Saved voice limit reached for your plan |

## See Also

- [VoiceDesign API](./voicedesign.md) — Create voices from descriptions
- [Saved Voices API](./speech.md) — Generate audio with saved voices
- [Pricing & Usage](./pricing.md) — Plan limits and quotas
