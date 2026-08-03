---
title: "Webhooks & Async APIs"
weight: 8
---

# Webhooks & Async APIs

Not all API communication follows the request-response pattern. Webhooks, polling, callbacks, and WebSockets enable asynchronous, event-driven architectures where systems react to changes as they happen rather than repeatedly asking "has anything changed?"

---

## Webhooks

A webhook is a user-defined HTTP callback. When an event occurs in the source system, it sends an HTTP POST to a URL configured by the consumer.

### How Webhooks Work

```
┌──────────┐    1. Register URL     ┌──────────────┐
│  Client  │ ────────────────────── │   Provider   │
│ (Server) │                        │  (e.g. Stripe│
│          │ ◄───────────────────── │   GitHub)    │
└──────────┘    2. POST event data  └──────────────┘
```

1. Consumer registers a callback URL with the provider
2. An event occurs in the provider (payment completed, PR merged, etc.)
3. Provider sends an HTTP POST with event payload to the registered URL
4. Consumer processes the event and returns 2xx to acknowledge

### Webhook Payload Design

```json
{
  "id": "evt_1NqB6e2eZvKY",
  "type": "payment.completed",
  "created": "2024-01-15T10:30:00Z",
  "data": {
    "object": {
      "id": "pay_abc123",
      "amount": 2500,
      "currency": "eur",
      "status": "succeeded"
    }
  },
  "api_version": "2024-01-01"
}
```

**Best practices for payloads:**
- Include an event `id` for deduplication
- Include event `type` for routing
- Include a `created` timestamp
- Keep payloads small — include IDs, not full objects (let the consumer fetch details if needed)
- Version your events

---

## Delivery Guarantees

Webhooks operate over HTTP — delivery is **at-least-once**, not exactly-once.

| Guarantee | Meaning | How to Handle |
|-----------|---------|---------------|
| At-least-once | Provider retries on failure; duplicates possible | Consumer must be idempotent |
| At-most-once | No retries; events may be lost | Acceptable only for non-critical events |
| Exactly-once | Impossible over HTTP alone | Use idempotency keys + deduplication |

### Making Consumers Idempotent

```python
import redis

r = redis.Redis()

def handle_webhook(event: dict) -> bool:
    event_id = event["id"]

    # Check if already processed
    if r.sismember("processed_events", event_id):
        return True  # Already handled, return success

    # Process the event
    process_event(event)

    # Mark as processed (with TTL for cleanup)
    r.sadd("processed_events", event_id)
    r.expire("processed_events", 86400 * 7)  # 7-day dedup window
    return True
```

---

## Retry with Exponential Backoff

Providers typically retry failed deliveries with increasing delays:

| Attempt | Delay | Total Elapsed |
|---------|-------|---------------|
| 1 | Immediate | 0s |
| 2 | 1 minute | 1 min |
| 3 | 5 minutes | 6 min |
| 4 | 30 minutes | 36 min |
| 5 | 2 hours | ~2.5 hours |
| 6 | 8 hours | ~10.5 hours |
| 7 | 24 hours | ~34.5 hours |

After exhausting retries, the provider may disable the webhook endpoint and notify the consumer.

### Provider-Side Retry Logic

```python
import time
import requests

RETRY_DELAYS = [0, 60, 300, 1800, 7200, 28800, 86400]

def deliver_webhook(url: str, payload: dict, secret: str) -> bool:
    for attempt, delay in enumerate(RETRY_DELAYS):
        if delay > 0:
            time.sleep(delay)

        signature = compute_hmac(payload, secret)
        headers = {
            "Content-Type": "application/json",
            "X-Webhook-Signature": signature,
            "X-Webhook-Id": payload["id"],
            "X-Webhook-Attempt": str(attempt + 1),
        }

        try:
            response = requests.post(url, json=payload, headers=headers, timeout=30)
            if 200 <= response.status_code < 300:
                return True
        except requests.RequestException:
            continue

    return False  # All retries exhausted
```

---

## Signature Verification (HMAC)

Webhooks must be verified to prevent spoofing. The standard approach uses HMAC-SHA256.

### Provider: Signing the Payload

```python
import hmac
import hashlib
import json

def compute_hmac(payload: dict, secret: str) -> str:
    body = json.dumps(payload, separators=(',', ':'))
    signature = hmac.new(
        secret.encode(),
        body.encode(),
        hashlib.sha256
    ).hexdigest()
    return f"sha256={signature}"
```

### Consumer: Verifying the Signature

```python
import hmac
import hashlib

def verify_webhook(payload_body: bytes, signature_header: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        payload_body,
        hashlib.sha256
    ).hexdigest()
    expected_signature = f"sha256={expected}"
    return hmac.compare_digest(expected_signature, signature_header)
```

**Critical:** Use `hmac.compare_digest` (constant-time comparison) to prevent timing attacks. Never use `==` for signature comparison.

### Timestamp Validation

Prevent replay attacks by including a timestamp and rejecting old webhooks:

```python
import time

def verify_timestamp(timestamp_header: str, tolerance_seconds: int = 300) -> bool:
    webhook_time = int(timestamp_header)
    current_time = int(time.time())
    return abs(current_time - webhook_time) <= tolerance_seconds
```

---

## Webhook Testing

| Tool | Purpose |
|------|---------|
| [webhook.site](https://webhook.site) | Inspect incoming webhook payloads |
| [ngrok](https://ngrok.com) | Expose local server for webhook delivery |
| [Svix](https://www.svix.com) | Webhook infrastructure (sending, retries, monitoring) |
| Mock servers | Unit/integration test webhook handlers locally |

### Testing Your Webhook Handler

```python
import pytest
from unittest.mock import patch

def test_webhook_handler_valid_signature():
    payload = {"id": "evt_123", "type": "order.created", "data": {}}
    body = json.dumps(payload).encode()
    secret = "whsec_test123"
    signature = compute_hmac(payload, secret)

    response = client.post(
        "/webhooks",
        data=body,
        headers={
            "X-Webhook-Signature": signature,
            "Content-Type": "application/json",
        },
    )
    assert response.status_code == 200

def test_webhook_handler_invalid_signature():
    response = client.post(
        "/webhooks",
        json={"id": "evt_123"},
        headers={"X-Webhook-Signature": "sha256=invalid"},
    )
    assert response.status_code == 401
```

---

## Async API Patterns

### Polling

The client periodically checks for updates. Simple but inefficient.

```python
import time
import requests

def poll_for_result(job_url: str, interval: int = 5, timeout: int = 300):
    start = time.time()
    while time.time() - start < timeout:
        response = requests.get(job_url)
        data = response.json()

        if data["status"] == "completed":
            return data["result"]
        elif data["status"] == "failed":
            raise RuntimeError(f"Job failed: {data['error']}")

        time.sleep(interval)
    raise TimeoutError("Polling timed out")
```

### Callback Pattern

Client provides a callback URL when initiating a long-running operation:

```http
POST /api/reports/generate
Content-Type: application/json

{
  "parameters": {"date_range": "2024-01"},
  "callback_url": "https://myapp.com/callbacks/report-ready"
}
```

Response (immediate):
```http
HTTP/1.1 202 Accepted
Location: /api/reports/jobs/abc123

{
  "job_id": "abc123",
  "status": "processing",
  "estimated_completion": "2024-01-15T10:35:00Z"
}
```

### WebSockets

Persistent bidirectional connection for real-time communication:

```python
import asyncio
import websockets

async def connect():
    async with websockets.connect("wss://api.example.com/ws") as ws:
        # Subscribe to events
        await ws.send(json.dumps({
            "action": "subscribe",
            "channels": ["orders", "notifications"]
        }))

        # Listen for messages
        async for message in ws:
            event = json.loads(message)
            handle_event(event)
```

### Server-Sent Events (SSE)

Unidirectional server-to-client streaming over HTTP:

```python
# Server (Flask)
from flask import Response

@app.route('/events')
def stream_events():
    def generate():
        while True:
            event = get_next_event()  # blocks until event available
            yield f"event: {event['type']}\ndata: {json.dumps(event['data'])}\n\n"
    return Response(generate(), mimetype='text/event-stream')
```

---

## Comparison of Async Patterns

| Pattern | Direction | Connection | Use Case |
|---------|-----------|-----------|----------|
| Polling | Client → Server | Stateless (repeated) | Simple status checks |
| Webhooks | Server → Client | Stateless (per event) | Event notifications |
| Callbacks | Server → Client | Stateless (one-time) | Long-running job completion |
| WebSockets | Bidirectional | Persistent | Chat, gaming, live data |
| SSE | Server → Client | Persistent (one-way) | Live feeds, dashboards |

---

## Event-Driven API Design Patterns

### Event Notification

Minimal payload — tells the consumer something happened, consumer fetches details:

```json
{
  "type": "order.updated",
  "resource_url": "/api/orders/12345"
}
```

### Event-Carried State Transfer

Full state included — consumer doesn't need to call back:

```json
{
  "type": "order.updated",
  "data": {
    "id": "12345",
    "status": "shipped",
    "tracking_number": "1Z999AA10123456784"
  }
}
```

### Event Sourcing via API

Expose an event log that consumers can replay:

```http
GET /api/events?after=evt_500&limit=100

{
  "events": [...],
  "has_more": true,
  "next_cursor": "evt_600"
}
```

---

## Webhook Security Checklist

- [ ] Verify HMAC signature on every request
- [ ] Validate timestamp to prevent replay attacks
- [ ] Use HTTPS endpoints only
- [ ] Return 2xx quickly, process asynchronously if needed
- [ ] Implement idempotency (deduplicate by event ID)
- [ ] Set reasonable timeouts (5-30 seconds)
- [ ] Log failed verifications for security monitoring
- [ ] Rotate webhook secrets periodically

---

## Key Takeaways

- Webhooks reverse the communication flow — the server pushes events to the client instead of the client polling
- **At-least-once delivery** is the practical guarantee; consumers must be idempotent
- **HMAC signature verification** is non-negotiable — unsigned webhooks are a security vulnerability
- **Exponential backoff** with reasonable retry limits handles transient failures gracefully
- Choose the async pattern based on your needs: webhooks for event notification, WebSockets for real-time bidirectional, SSE for server-to-client streams
- **Event-carried state transfer** reduces coupling (no callback needed) but increases payload size and coupling to schema
- Always respond to webhooks quickly (< 5s) — offload heavy processing to a background queue
