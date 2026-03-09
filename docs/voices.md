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
| `language` | string | No | English | Language of the voice (e.g., "English", "Spanish", "Japanese") |

**cURL**

```curl
# First, generate audio with VoiceDesign
curl -X POST "https://api.murmr.dev/v1/voices/design" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{"text": "Sample", "voice_description": "A warm voice", "language": "English"}' \
  > stream_output.txt

# Collect PCM from SSE, convert to WAV, then base64 encode
# (The SDK handles this automatically)
AUDIO_B64=$(base64 -i sample.wav)

# Save the voice
curl -X POST "https://api.murmr.dev/v1/voices" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"My Narrator\",
    \"description\": \"A warm, professional voice\",
    \"audio\": \"$AUDIO_B64\"
  }"
```

**Python**

```python
from murmr import MurmrClient

client = MurmrClient(api_key="YOUR_API_KEY")

# 1. Generate audio with VoiceDesign
wav = client.voices.design(
    input="Sample audio for voice extraction",
    voice_description="A warm, professional voice",
    language="English",
)

# 2. Save the voice for reuse
voice = client.voices.save(
    name="My Narrator",
    audio=wav,
    description="A warm, professional voice",
)
print(f"Saved as: {voice.id}")  # voice_abc123def456

# 3. Use the saved voice
job = client.speech.create(input="Hello!", voice=voice.id)
```

**JavaScript (SDK)**

```javascript
import { MurmrClient } from "@murmr/sdk";

const client = new MurmrClient({ apiKey: "YOUR_API_KEY" });

// 1. Generate audio with VoiceDesign
const wav = await client.voices.design({
  input: "Sample audio for voice extraction",
  voice_description: "A warm, professional voice",
});

// 2. Save the voice for reuse
const voice = await client.voices.save({
  name: "My Narrator",
  audio: wav,
  description: "A warm, professional voice",
});

console.log(`Saved as: ${voice.id}`);  // voice_abc123def456
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

```cURL
curl "https://api.murmr.dev/v1/voices" \
  -H "Authorization: Bearer YOUR_API_KEY"
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

```cURL
curl -X DELETE "https://api.murmr.dev/v1/voices/voice_abc123def456" \
  -H "Authorization: Bearer YOUR_API_KEY"
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

Extract voice embeddings from audio. This endpoint goes through the Worker at `api.murmr.dev` and uses API key auth.

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `audio` | string | Yes | -- | Base64-encoded WAV audio to extract embeddings from |
| `model` | string | No | base | Model to use: "base" (default) or "base_06b" |

```cURL
curl -X POST "https://api.murmr.dev/v1/voices/extract-embeddings" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"audio": "<base64-wav>"}'
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
