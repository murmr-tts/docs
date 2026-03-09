# Error Reference

Complete guide to API errors, their causes, and how to resolve them.

## Error Response Format

Most errors return a JSON object with a string `error` field:

```JSON
{
  "error": "Description of what went wrong"
}
```

Rate limit errors (429) return a structured object with additional metadata to help you handle retries programmatically:

```JSON
{
  "error": {
    "type": "rate_limit_exceeded",
    "message": "Maximum 5 concurrent requests. Retry when a current request completes.",
    "concurrent_limit": 5,
    "concurrent_active": 5
  }
}
```

Check `error.type` to programmatically distinguish rate limits from other errors.

## 429 Error Types

The 429 status code covers several rate-limiting scenarios. Check `error.type` or the error message to determine the cause:

| Type / Message | Cause | Action |
| --- | --- | --- |
| type: "rate_limit_exceeded" | Concurrent request limit for your plan (Free=1, Starter=3, Pro/Realtime=5, Scale=10) | Wait for a current request to complete, or upgrade plan |
| "Usage limit exceeded." | Monthly character quota exhausted | Upgrade plan or wait for monthly reset |
| "Voice design limit exceeded." | Voice design quota for the period | Upgrade plan or wait for reset |
| "Server is at capacity." | All backend GPU slots are busy | Retry in a few seconds |
| "Server busy, please retry" | Batch queue is full | Retry after Retry-After header |

## HTTP Error Codes

### 400 Bad Request

The request was malformed or missing required parameters.

**Common causes:**

- Missing required parameter (text or voice_description)
- Input text exceeds 4,096 characters
- Voice description exceeds 500 characters
- Invalid response_format (must be mp3, opus, aac, flac, wav, or pcm)
- Invalid webhook_url (not HTTPS, private IP, or too long)
- Invalid JSON body

**Solutions:**

- Check required parameters: text + voice_description (VoiceDesign) or text + voice (Saved Voices)
- Split long text into multiple requests (<4096 chars each)
- Use a valid response_format: mp3, opus, aac, flac, wav, or pcm
- Ensure webhook_url is a valid external HTTPS URL
- Ensure request body is valid JSON

### 401 Unauthorized

Authentication failed — missing or invalid API key.

**Common causes:**

- Missing Authorization header
- Invalid API key format (must start with murmr_sk_)
- Expired or revoked API key
- Using wrong key type (Supabase JWT for Worker endpoints)

**Solutions:**

- Include header: Authorization: Bearer YOUR_API_KEY
- Verify key was copied correctly (no extra spaces)
- Check if key was revoked in dashboard settings
- Generate a new API key if needed

### 404 Not Found

The requested resource was not found.

**Common causes:**

- Unknown API endpoint
- Saved voice ID not found or belongs to another account
- Job ID not found, expired (24h TTL), or belongs to another account
- Invalid job ID format (must match job_[a-f0-9]{16})

**Solutions:**

- Check the endpoint URL is correct
- Verify the voice or job ID is exact (case-sensitive)
- Job data expires after 24 hours — re-submit if needed

### 405 Method Not Allowed

The HTTP method is not supported for this endpoint.

**Common causes:**

- Using GET on a POST-only endpoint
- Using POST on a GET-only endpoint (e.g., /v1/jobs/:id)

**Solutions:**

- TTS endpoints require POST
- Job status endpoint requires GET
- Check the API reference for the correct method

### 429 Rate Limit / Quota Exceeded

You have exceeded your usage quota or the server is at capacity.

**Common causes:**

- Monthly character limit reached
- Voice design quota exceeded
- Per-plan concurrent request limit exceeded (Free=1, Starter=3, Pro/Realtime=5, Scale=10)
- Server at GPU capacity (all slots busy)

**Solutions:**

- Check the error message and error.type field for the specific reason
- Upgrade to a higher plan for more characters or concurrency
- Wait for the next billing period (limits reset monthly)
- For capacity errors, retry after a few seconds
- Check the Retry-After header when present

### 410 Gone

The completed job's audio data has been purged from the backend.

**Common causes:**

- Audio result was not retrieved promptly after job completion
- RunPod purges completed job data after a short retention period

**Solutions:**

- Re-submit the job to generate the audio again
- Retrieve completed audio promptly after polling shows completion
- Use webhooks for reliable delivery of completed audio

### 500 Internal Server Error

An unexpected error occurred on our servers.

**Common causes:**

- Temporary server issue
- Auth service unavailable
- Unexpected processing error

**Solutions:**

- Retry the request after a few seconds
- If persistent, contact support@murmr.dev

### 502 Bad Gateway

The TTS backend is unreachable or returned an error.

**Common causes:**

- TTS backend is starting up
- Backend returned an error during processing
- Network issue between gateway and backend

**Solutions:**

- Retry the request
- Try with shorter input text
- If persistent, the backend may be undergoing maintenance

### 503 Service Unavailable

The TTS service is temporarily unavailable.

**Common causes:**

- Cold start — first request after idle period
- Batch processing service not configured
- No GPU workers available

**Solutions:**

- Wait 10-30 seconds and retry
- Cold starts are normal — first request may take longer
- Implement exponential backoff for retries

### 504 Gateway Timeout

The backend did not respond in time.

**Common causes:**

- TTS generation taking too long
- Backend network timeout
- Very long text near the 4096 char limit

**Solutions:**

- Retry the request
- Use shorter text for faster generation
- Use the batch endpoint for long text (processes asynchronously)
- Set HTTP client timeout to at least 60 seconds

## Implementing Retry Logic

For 5xx errors and 429 rate limits, implement exponential backoff:

```Python
import time
import requests

def generate_speech_with_retry(text, voice, max_retries=3):
    """Generate speech with exponential backoff on failure."""
    for attempt in range(max_retries):
        response = requests.post(
            "https://api.murmr.dev/v1/audio/speech",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json"
            },
            json={"text": text, "voice": voice}
        )

        if response.status_code == 202:
            return response.json()  # Job submitted

        # Retry on 5xx errors or 429
        if response.status_code >= 500 or response.status_code == 429:
            wait_time = (2 ** attempt) * 2  # 2, 4, 8 seconds
            retry_after = response.headers.get("Retry-After")
            if retry_after:
                wait_time = int(retry_after)
            print(f"Retry {attempt + 1}/{max_retries} in {wait_time}s...")
            time.sleep(wait_time)
            continue

        # Don't retry 4xx client errors (except 429)
        response.raise_for_status()

    raise Exception("Max retries exceeded")
```

## Troubleshooting Tips

First request is slow?

Cold starts are normal. The first request after an idle period may take 20-30 seconds as the TTS worker spins up. Subsequent requests are faster.

Getting 401 with correct key?

Ensure the format is exactly `Bearer YOUR_KEY` — note the space after "Bearer". Check for leading/trailing whitespace in your key.

Invalid voice error?

Saved voice IDs must be exactly as returned from the API (e.g., voice_abc123). For custom voices, use VoiceDesign with a voice_description instead.

Request timing out?

Set your HTTP client timeout to at least 60 seconds. For the batch endpoint, note that it always returns 202 immediately — the actual generation happens asynchronously.

## WebSocket Close Codes

For WebSocket connections (`/v1/realtime`), the server may close with these codes:

| Code | Name | Description |
| --- | --- | --- |
| 4001 | AUTH_TIMEOUT | No config message received within 10 seconds of connecting |
| 4002 | AUTH_FAILED | Invalid API key in the config message |
| 4003 | INVALID_MESSAGE | Malformed JSON or unexpected message type |
| 4004 | RATE_LIMITED | Too many concurrent connections or generation rate exceeded |
| 4005 | SERVER_ERROR | Internal server error during processing |

Code `4006` is sent as an error message (not a close code) when all GPU slots are occupied:

```JSON
{
  "type": "error",
  "message": "All slots occupied, try again shortly",
  "code": 4006
}
```

See [WebSocket Protocol](./websocket-protocol.md) for connection details.

> **ℹ️ Need Help?**
> If you're experiencing persistent issues not covered here, contact us at [support@murmr.dev](mailto:support@murmr.dev) with:
> - The error message you're receiving
> - The request you're making (without your API key)
> - Approximate timestamp of the error
