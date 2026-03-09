# Async Jobs

All batch TTS requests are processed asynchronously. Submit a job, get a job ID, then poll for the audio result or receive it via webhook.

> **ℹ️ Always Async**
> The batch endpoint `POST /v1/audio/speech` always returns `202 Accepted` with a job ID — it never returns audio directly. Use `GET /v1/jobs/{jobId}` to retrieve the audio when the job completes. For real-time playback, use the [streaming endpoint](./speech.md) instead.

## How It Works

1

Submit a batch request

`POST /v1/audio/speech` returns `202` immediately with a job ID like `job_a1b2c3d4e5f6g7h8`.

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

**Python**

```python
import requests

response = requests.post(
    "https://api.murmr.dev/v1/audio/speech",
    headers={"Authorization": "Bearer YOUR_API_KEY"},
    json={
        "text": "A long article to convert to speech...",
        "voice": "voice_abc123",
        "response_format": "mp3"
    }
)

job = response.json()
print(f"Job submitted: {job['id']}")
# {"id": "job_a1b2c3d4e5f6g7h8", "status": "queued", "created_at": "..."}
```

**JavaScript**

```javascript
const response = await fetch("https://api.murmr.dev/v1/audio/speech", {
  method: "POST",
  headers: {
    "Authorization": "Bearer YOUR_API_KEY",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    text: "A long article to convert to speech...",
    voice: "voice_abc123",
    response_format: "mp3"
  })
});

const job = await response.json();
// { id: "job_a1b2c3d4e5f6g7h8", status: "queued", created_at: "..." }
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

**Python**

```python
import time
import requests

API_KEY = "YOUR_API_KEY"
BASE_URL = "https://api.murmr.dev"

# 1. Submit job
resp = requests.post(
    f"{BASE_URL}/v1/audio/speech",
    headers={"Authorization": f"Bearer {API_KEY}"},
    json={
        "text": "Long text to synthesize...",
        "voice": "voice_abc123",
        "response_format": "mp3"
    }
)
job = resp.json()
job_id = job["id"]
print(f"Submitted: {job_id}")

# 2. Poll for completion
while True:
    poll = requests.get(
        f"{BASE_URL}/v1/jobs/{job_id}",
        headers={"Authorization": f"Bearer {API_KEY}"}
    )

    # Completed: binary audio returned
    content_type = poll.headers.get("Content-Type", "")
    if content_type.startswith("audio/"):
        ext = "mp3" if "mpeg" in content_type else content_type.split("/")[-1]
        filename = f"{job_id}.{ext}"
        with open(filename, "wb") as f:
            f.write(poll.content)
        duration = poll.headers.get("X-Audio-Duration-Ms", "?")
        print(f"Saved {filename} ({duration}ms audio)")
        break

    # Still in progress or failed
    data = poll.json()
    if data["status"] == "failed":
        print(f"Failed: {data.get('error')}")
        break

    print(f"Status: {data['status']}...")
    time.sleep(3)
```

**JavaScript**

```javascript
const API_KEY = "YOUR_API_KEY";
const BASE_URL = "https://api.murmr.dev";

// 1. Submit job
const submitResp = await fetch(`${BASE_URL}/v1/audio/speech`, {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${API_KEY}`,
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    text: "Long text to synthesize...",
    voice: "voice_abc123",
    response_format: "mp3"
  })
});
const job = await submitResp.json();

// 2. Poll for completion
async function pollJob(jobId) {
  while (true) {
    const resp = await fetch(`${BASE_URL}/v1/jobs/${jobId}`, {
      headers: { "Authorization": `Bearer ${API_KEY}` }
    });

    const ct = resp.headers.get("Content-Type") || "";

    // Completed: binary audio
    if (ct.startsWith("audio/")) {
      return await resp.blob();
    }

    // In progress or failed
    const data = await resp.json();
    if (data.status === "failed") throw new Error(data.error);

    await new Promise(r => setTimeout(r, 3000));
  }
}

const audioBlob = await pollJob(job.id);
const audio = new Audio(URL.createObjectURL(audioBlob));
audio.play();
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
