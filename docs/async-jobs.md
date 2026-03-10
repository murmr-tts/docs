# Async Jobs

All batch TTS requests are processed asynchronously. Submit a job, get a job ID, then poll for the audio result or receive it via webhook.

> **ℹ️ When to Use Async**
> `POST /v1/audio/speech` returns `200` with binary audio by default (sync). When you provide a `webhook_url` parameter, it returns `202 Accepted` with a job ID for async processing. Use `GET /v1/jobs/{jobId}` to poll, or wait for the webhook callback.

## How It Works

1

Submit a batch request

`POST /v1/audio/speech` with a `webhook_url` parameter returns `202` immediately with a job ID like `job_a1b2c3d4e5f6g7h8`. Without `webhook_url`, it returns `200` with binary audio (sync).

2

Job processes on GPU

The job enters the RunPod queue. Status progresses: `queued` -> `processing` -> `completed` or `failed`.

3

Retrieve the audio

Poll `GET /v1/jobs/{jobId}` — returns **binary audio** when complete. Optionally add `webhook_url` to receive the result via POST.

## Submitting a Job

**cURL**

```curl
curl -X POST "https://api.murmr.dev/v1/audio/speech" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "A long article to convert to speech...",
    "voice": "voice_abc123",
    "response_format": "mp3"
  }'
# Returns 202:
# {"id": "job_a1b2c3d4e5f6g7h8", "status": "queued", "created_at": "..."}
```

**Node.js SDK**

```typescript
import { MurmrClient, isAsyncResponse } from '@murmr/sdk';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

const result = await client.speech.create({
  input: 'A long article to convert to speech...',
  voice: 'voice_abc123',
  response_format: 'mp3',
  webhook_url: 'https://yourapp.com/webhooks/tts',
});

if (isAsyncResponse(result)) {
  console.log(`Job ID: ${result.id}`);
  console.log(`Status: ${result.status}`);  // "queued"
}
```

**Python SDK**

```python
import os
from murmr import MurmrClient

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    job = client.speech.create(
        input="A long article to convert to speech...",
        voice="voice_abc123",
        response_format="mp3",
        webhook_url="https://yourapp.com/webhooks/tts",
    )

    print(f"Job ID: {job.id}")
    print(f"Status: {job.status}")  # "queued"
```

**Python (async)**

```python
import asyncio
import os
from murmr import AsyncMurmrClient

async def main():
    async with AsyncMurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
        job = await client.speech.create(
            input="Async job submission.",
            voice="voice_abc123",
            response_format="opus",
            webhook_url="https://yourapp.com/webhooks/tts",
        )

        result = await client.jobs.wait_for_completion(
            job.id,
            poll_interval_s=2.0,
        )

        with open("output.opus", "wb") as f:
            f.write(result.audio_bytes)

asyncio.run(main())
```

Optionally add `webhook_url` to receive the completed audio via POST to your server. See [Webhook Delivery](#webhook-delivery) below.

## Submit Response

202 Accepted

application/json

```JSON
{
  "id": "job_a1b2c3d4e5f6g7h8",
  "status": "queued",
  "created_at": "2026-02-13T10:30:00.000Z"
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | string | Yes | Unique job identifier (job_ + 16 hex chars). Use this to poll for status. |
| `status` | string | Yes | Always "queued" on submission. |
| `created_at` | string | Yes | ISO 8601 timestamp of when the job was submitted. |

## Polling for Results

`GET /v1/jobs/{jobId}`

Returns current job status. Requires the same API key used to submit the job. Only the job owner can access their jobs.

> **ℹ️ Response varies by status**
> - **In progress:** Returns JSON with `status` field
> - **Completed:** Returns **binary audio** (Content-Type: audio/*) — save directly to file
> - **Failed:** Returns JSON with `error` field
> - **Expired:** Returns `410 Gone` if audio has been purged

### In-Progress Response

```JSON
{
  "id": "job_a1b2c3d4e5f6g7h8",
  "status": "processing",
  "created_at": "2026-02-13T10:30:00.000Z"
}
```

### Completed Response

- **200 OK — audio/mpeg (or requested format):** Binary audio data. Check the `Content-Type` header to determine the format. Response headers include `X-Audio-Duration-Ms` and `X-Total-Time-Ms`.

### Failed Response

```JSON
{
  "id": "job_a1b2c3d4e5f6g7h8",
  "status": "failed",
  "error": "Voice not found: voice_invalid",
  "created_at": "2026-02-13T10:30:00.000Z"
}
```

## Polling Example

**Node.js SDK**

```typescript
import { MurmrClient, MurmrError } from '@murmr/sdk';
import { writeFileSync } from 'fs';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

// Option 1: Manual polling
const status = await client.jobs.get('job_a1b2c3d4e5f6g7h8');

if (status.status === 'completed' && status.audio_base64) {
  const audio = Buffer.from(status.audio_base64, 'base64');
  writeFileSync(`output.${status.response_format || 'wav'}`, audio);
}

// Option 2: Wait for completion (polls automatically)
try {
  const result = await client.jobs.waitForCompletion('job_a1b2c3d4e5f6g7h8', {
    pollIntervalMs: 3000,
    timeoutMs: 900_000,
    onPoll: (status) => console.log(`Status: ${status.status}`),
  });

  if (result.audio_base64) {
    writeFileSync('output.wav', Buffer.from(result.audio_base64, 'base64'));
  }
} catch (error) {
  if (error instanceof MurmrError && error.code === 'JOB_FAILED') {
    console.error('Job failed:', error.message);
  }
}

// Option 3: Submit and wait in one call
const final = await client.speech.createAndWait({
  input: 'Submit and wait in one call.',
  voice: 'voice_abc123',
  response_format: 'mp3',
});
```

**Python SDK**

```python
import os
from murmr import MurmrClient, MurmrError

with MurmrClient(api_key=os.environ["MURMR_API_KEY"]) as client:
    # Option 1: Manual polling
    status = client.jobs.get("job_a1b2c3d4e5f6g7h8")

    if status.status == "completed":
        with open("output.mp3", "wb") as f:
            f.write(status.audio_bytes)

    # Option 2: Wait for completion
    try:
        result = client.jobs.wait_for_completion(
            "job_a1b2c3d4e5f6g7h8",
            poll_interval_s=3.0,
            timeout_s=120.0,
            on_poll=lambda s: print(f"Polling: {s.status}"),
        )
        with open("output.mp3", "wb") as f:
            f.write(result.audio_bytes)
    except MurmrError as e:
        if e.code == "JOB_FAILED":
            print(f"Job failed: {e.message}")

    # Option 3: Submit and wait in one call
    result = client.speech.create_and_wait(
        input="Submit and wait in one call.",
        voice="voice_abc123",
        response_format="mp3",
    )
```

**cURL**

```curl
# Poll for status
curl -s "https://api.murmr.dev/v1/jobs/job_a1b2c3d4e5f6g7h8" \
  -H "Authorization: Bearer YOUR_API_KEY"

# When completed: returns binary audio (Content-Type: audio/*)
# When in progress: returns JSON with status
# When failed: returns JSON with error
# When expired: returns 410 Gone

# Save completed audio to file
curl "https://api.murmr.dev/v1/jobs/job_a1b2c3d4e5f6g7h8" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --output output.mp3
```

> **💡 Polling Best Practice**
> Poll every 3-5 seconds. Job metadata expires after 24 hours. Audio data on the backend may be purged sooner — retrieve promptly after completion to avoid a `410 Gone` response.

## Webhook Delivery

Add `webhook_url` to your request to receive the audio via POST when the job completes. Polling still works as a fallback.

```cURL
curl -X POST "https://api.murmr.dev/v1/audio/speech" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Text to synthesize...",
    "voice": "voice_abc123",
    "response_format": "mp3",
    "webhook_url": "https://your-server.com/webhooks/tts"
  }'
```

### Success Payload

```JSON
{
  "id": "job_a1b2c3d4e5f6g7h8",
  "status": "completed",
  "audio": "<base64-encoded audio>",
  "content_type": "audio/mpeg",
  "response_format": "mp3",
  "duration_ms": 4520,
  "total_time_ms": 3200
}
```

### Failure Payload

```JSON
{
  "id": "job_a1b2c3d4e5f6g7h8",
  "status": "failed",
  "error": "Voice not found: voice_invalid"
}
```

### Webhook Handler Example

**Python (Flask)**

```python
import base64
from flask import Flask, request

app = Flask(__name__)

@app.route("/webhooks/tts", methods=["POST"])
def handle_tts_webhook():
    payload = request.json

    if payload["status"] == "completed":
        audio_bytes = base64.b64decode(payload["audio"])
        filename = f"{payload['id']}.{payload['response_format']}"

        with open(filename, "wb") as f:
            f.write(audio_bytes)

        print(f"Saved {filename} ({payload['duration_ms']}ms audio)")
    else:
        print(f"Job failed: {payload.get('error')}")

    return "OK", 200
```

**Node.js (Express)**

```javascript
import express from "express";
import { writeFile } from "fs/promises";

const app = express();
app.use(express.json({ limit: "50mb" }));

app.post("/webhooks/tts", async (req, res) => {
  const payload = req.body;

  if (payload.status === "completed") {
    const audioBuffer = Buffer.from(payload.audio, "base64");
    const filename = `${payload.id}.${payload.response_format}`;

    await writeFile(filename, audioBuffer);
    console.log(`Saved ${filename} (${payload.duration_ms}ms audio)`);
  } else {
    console.error(`Job failed: ${payload.error}`);
  }

  res.sendStatus(200);
});
```

> **⚠️ Webhook Requirements**
> - **HTTPS only** — webhook_url must use HTTPS protocol
> - **Max URL length** — 2,048 characters
> - **No private IPs** — localhost, 127.0.0.1, 10.x, 172.16-31.x, 192.168.x are blocked
> - **No internal domains** — murmr.dev subdomains are blocked
> - **Payload size** — audio is base64-encoded (~1.33x raw file size). Set your server's body size limit accordingly.

## Job ID Format

Job IDs follow the pattern `job_` followed by 16 hexadecimal characters:

```Regex
/^job_[a-f0-9]{16}$/

Example: job_a1b2c3d4e5f6g7h8
```

## Job Status Values

| Status | Description |
| --- | --- |
| queued | Job is waiting for an available GPU worker |
| processing | GPU worker is generating audio |
| completed | Audio generated — poll to download, or check webhook |
| failed | Job failed — check the error field for details |

Job metadata expires after **24 hours**. Audio data on the backend may be purged sooner. Retrieve completed audio promptly.

## Error Responses

| Status | Error | Description |
|--------|-------|-------------|
| 400 | Bad Request | Invalid webhook_url (not HTTPS, private IP, too long), invalid response_format, missing text |
| 404 | Not Found | Job ID not found, expired (24h TTL), or belongs to a different account |
| 410 | Gone | Job completed but audio data has been purged from the backend. Re-submit the job. |
| 429 | Server Busy | GPU queue is full. Check the `Retry-After` header. |
| 503 | Service Unavailable | Batch processing is temporarily unavailable |

## See Also

- [Saved Voices API](./speech.md) — Batch and streaming TTS endpoints
- [Audio Formats](./audio-formats.md) — Supported response_format values
- [Error Reference](./errors.md) — Complete error handling guide
