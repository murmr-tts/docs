# Async Jobs

Submit batch TTS jobs, poll for results, and receive webhook notifications when audio is ready. Ideal for background processing and server-to-server workflows.

## How It Works

1. **Submit** -- `POST /v1/audio/speech` with a `webhook_url` returns `202 Accepted` with a job ID. Without `webhook_url`, the endpoint returns HTTP 200 with binary audio directly (synchronous mode).
2. **Process** -- The job runs on GPU workers. Status progresses: `queued` -> `processing` -> `completed` or `failed`
3. **Retrieve** -- Poll `GET /v1/jobs/{jobId}` for the audio, or receive it via webhook

## Job Statuses

| Status | Description |
|--------|-------------|
| `queued` | Job is waiting for an available GPU worker |
| `processing` | GPU worker is generating audio |
| `completed` | Audio generated -- poll to download, or check webhook |
| `failed` | Job failed -- check the `error` field for details |

## Submitting a Job

Submit a batch TTS job. Without `webhook_url`, the API returns synchronous audio (HTTP 200). With `webhook_url`, it returns a job ID (HTTP 202) for async processing:

**curl**
```bash
curl -X POST "https://api.murmr.dev/v1/audio/speech" \
  -H "Authorization: Bearer $MURMR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "A long article to convert to speech...",
    "voice": "voice_abc123",
    "response_format": "mp3"
  }'
# Returns 202:
# {"id": "job_a1b2c3d4e5f6g7h8", "status": "queued", "created_at": "..."}
```

Submit a job with `webhook_url` to receive results asynchronously. Use `isAsyncResponse()` to type-narrow the result:

**TypeScript**
```typescript
import { MurmrClient, isAsyncResponse } from '@murmr/sdk';

const client = new MurmrClient({ apiKey: process.env.MURMR_API_KEY! });

const result = await client.speech.create({
  input: 'Generate this audio asynchronously.',
  voice: 'voice_abc123',
  response_format: 'mp3',
  webhook_url: 'https://yourapp.com/webhooks/tts',
});

if (isAsyncResponse(result)) {
  console.log(`Job ID: ${result.id}`);
  console.log(`Status: ${result.status}`);  // "queued"
}
```

**Python**
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

## Polling for Results

`GET /v1/jobs/{jobId}` returns the current job status. When the job is completed, it returns binary audio directly.

Poll for a job's status with curl. Completed jobs return binary audio directly; in-progress jobs return JSON with status:

**curl**
```bash
# Poll for status
curl -s "https://api.murmr.dev/v1/jobs/$JOB_ID" \
  -H "Authorization: Bearer $MURMR_API_KEY"

# When completed: returns binary audio (Content-Type: audio/*)
# When in progress: returns JSON with status
# When failed: returns JSON with error
# When expired: returns 410 Gone
```

Three polling strategies in TypeScript: manual single poll, automatic wait-for-completion, or the all-in-one `createAndWait`:

**TypeScript**
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

**Python (sync)**
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

## Webhook Delivery

Add `webhook_url` to your request to receive the audio via POST when the job completes. Polling still works as a fallback.

### Requirements

- URL must use **HTTPS** (HTTP is rejected)
- URL must be publicly accessible (no private IPs or localhost)
- Your endpoint must return a 2xx status within 10 seconds
- Max URL length: 2,048 characters

### Success Payload

When the job completes, your webhook receives a POST with base64-encoded audio and metadata:

```json
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

If the job fails, the webhook payload includes the error message. Check `status === "failed"` and read the `error` field:

```json
{
  "id": "job_a1b2c3d4e5f6g7h8",
  "status": "failed",
  "error": "Voice not found: voice_invalid"
}
```

### Webhook Handler

Receive webhook deliveries in your backend. Decode the base64 audio and save it. Set a large JSON body limit (50MB+) since audio payloads can be substantial:

**TypeScript (Express)**
```typescript
import express from 'express';
import { writeFile } from 'fs/promises';

const app = express();
app.use(express.json({ limit: '50mb' }));

app.post('/webhooks/tts', async (req, res) => {
  const payload = req.body;

  if (payload.status === 'completed') {
    const audioBuffer = Buffer.from(payload.audio, 'base64');
    const filename = `${payload.id}.${payload.response_format}`;
    await writeFile(filename, audioBuffer);
    console.log(`Saved ${filename} (${payload.duration_ms}ms audio)`);
  } else {
    console.error(`Job failed: ${payload.error}`);
  }

  res.sendStatus(200);
});
```

Same webhook handler in Python using Flask. Decode the base64 audio payload and write it to disk:

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

## Job Expiration

Job metadata expires after **24 hours**. Audio data on the backend may be purged sooner. Retrieve completed audio promptly to avoid a `410 Gone` response.

## See Also

- [Long-Form Audio](./long-form.md) -- Generate audio from text of any length
- [Portable Embeddings](./portable-embeddings.md) -- Store and reuse voice embeddings
