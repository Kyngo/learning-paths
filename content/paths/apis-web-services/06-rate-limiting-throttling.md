---
title: "Rate Limiting & Throttling"
weight: 6
---

# Rate Limiting & Throttling

Rate limiting protects APIs from abuse, ensures fair resource allocation, and maintains service stability under heavy load. Understanding rate limiting algorithms and their trade-offs is essential for both API providers and consumers.

---

## Why Rate Limit?

- **Protect infrastructure** from overload and cascading failures
- **Ensure fairness** among consumers sharing a service
- **Prevent abuse** (scraping, brute-force attacks, DDoS)
- **Control costs** for metered or pay-per-use services
- **Meet SLAs** by preserving capacity for critical traffic

---

## Rate Limiting Algorithms

### Token Bucket

The token bucket algorithm allows bursts while enforcing an average rate. Tokens are added to a bucket at a fixed rate; each request consumes one token. If the bucket is empty, the request is rejected.

```python
import time

class TokenBucket:
    def __init__(self, capacity: int, refill_rate: float):
        """
        capacity: maximum tokens in the bucket
        refill_rate: tokens added per second
        """
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate
        self.last_refill = time.monotonic()

    def _refill(self):
        now = time.monotonic()
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)
        self.last_refill = now

    def allow_request(self) -> bool:
        self._refill()
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False
```

**Characteristics:** allows bursts up to bucket capacity, smooth average rate, simple to implement.

### Leaky Bucket

The leaky bucket processes requests at a constant rate regardless of burst. Incoming requests are queued; if the queue overflows, requests are dropped.

```python
from collections import deque
import time

class LeakyBucket:
    def __init__(self, capacity: int, leak_rate: float):
        """
        capacity: maximum queue size
        leak_rate: requests processed per second
        """
        self.capacity = capacity
        self.leak_rate = leak_rate
        self.queue = deque()
        self.last_leak = time.monotonic()

    def _leak(self):
        now = time.monotonic()
        elapsed = now - self.last_leak
        to_remove = int(elapsed * self.leak_rate)
        for _ in range(min(to_remove, len(self.queue))):
            self.queue.popleft()
        self.last_leak = now

    def allow_request(self, request_id: str) -> bool:
        self._leak()
        if len(self.queue) < self.capacity:
            self.queue.append(request_id)
            return True
        return False
```

**Characteristics:** smooths output to a constant rate, no bursts allowed, acts as a traffic shaper.

### Fixed Window Counter

Divides time into fixed intervals (e.g., 1-minute windows). A counter tracks requests per window and resets at each boundary.

```python
import time

class FixedWindowCounter:
    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window_seconds = window_seconds
        self.counter = 0
        self.window_start = int(time.time()) // window_seconds

    def allow_request(self) -> bool:
        current_window = int(time.time()) // self.window_seconds
        if current_window != self.window_start:
            self.window_start = current_window
            self.counter = 0
        if self.counter < self.limit:
            self.counter += 1
            return True
        return False
```

**Problem:** burst at window boundaries — a client can send 2× the limit by hitting the end of one window and start of the next.

### Sliding Window Log

Tracks the timestamp of each request. Counts requests within the trailing window by removing expired entries.

```python
import time
from collections import deque

class SlidingWindowLog:
    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window_seconds = window_seconds
        self.timestamps = deque()

    def allow_request(self) -> bool:
        now = time.time()
        # Remove expired entries
        while self.timestamps and self.timestamps[0] <= now - self.window_seconds:
            self.timestamps.popleft()
        if len(self.timestamps) < self.limit:
            self.timestamps.append(now)
            return True
        return False
```

**Characteristics:** precise, no boundary bursts, but higher memory usage (stores each timestamp).

### Sliding Window Counter

A hybrid: combines fixed window counters with weighted overlap from the previous window. Lower memory than the log approach, nearly as accurate.

```
current_count = prev_window_count * overlap_percentage + current_window_count
```

---

## Algorithm Comparison

| Algorithm | Burst Handling | Memory | Accuracy | Complexity |
|-----------|---------------|--------|----------|------------|
| Token Bucket | Allows controlled bursts | Low (counter + timestamp) | Good | Low |
| Leaky Bucket | No bursts (constant output) | Medium (queue) | Exact | Low |
| Fixed Window | Boundary burst problem | Very low (counter) | Approximate | Very low |
| Sliding Window Log | No boundary issue | High (all timestamps) | Exact | Medium |
| Sliding Window Counter | Minimal boundary issue | Low (two counters) | Near-exact | Low |

---

## HTTP Response Headers

Standard headers communicate rate limit status to clients:

| Header | Description | Example |
|--------|-------------|---------|
| `X-RateLimit-Limit` | Maximum requests allowed in the window | `1000` |
| `X-RateLimit-Remaining` | Requests remaining in current window | `742` |
| `X-RateLimit-Reset` | Unix timestamp when the window resets | `1672531200` |
| `Retry-After` | Seconds to wait before retrying (on 429) | `30` |

### Response When Rate Limited

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1672531260
Retry-After: 60

{
  "error": "rate_limit_exceeded",
  "message": "Rate limit of 100 requests per minute exceeded",
  "retry_after": 60
}
```

---

## Client-Side Handling

### Exponential Backoff with Jitter

```python
import time
import random
import requests

def request_with_backoff(url: str, max_retries: int = 5) -> requests.Response:
    for attempt in range(max_retries):
        response = requests.get(url)

        if response.status_code != 429:
            return response

        # Use Retry-After header if available
        retry_after = response.headers.get("Retry-After")
        if retry_after:
            wait = int(retry_after)
        else:
            # Exponential backoff with full jitter
            wait = random.uniform(0, min(60, 2 ** attempt))

        time.sleep(wait)

    raise RuntimeError(f"Exceeded max retries for {url}")
```

### Client-Side Rate Limiter

Proactively limit outgoing requests to avoid hitting server limits:

```python
import time
import threading

class ClientRateLimiter:
    def __init__(self, requests_per_second: float):
        self.min_interval = 1.0 / requests_per_second
        self.last_request = 0.0
        self.lock = threading.Lock()

    def wait(self):
        with self.lock:
            now = time.monotonic()
            elapsed = now - self.last_request
            if elapsed < self.min_interval:
                time.sleep(self.min_interval - elapsed)
            self.last_request = time.monotonic()
```

---

## Distributed Rate Limiting

In multi-instance deployments, rate limits must be shared across nodes.

### Redis-Based Sliding Window

```python
import redis
import time

r = redis.Redis()

def is_rate_limited(client_id: str, limit: int, window: int) -> bool:
    key = f"rate:{client_id}"
    now = time.time()
    pipe = r.pipeline()
    pipe.zremrangebyscore(key, 0, now - window)  # Remove expired
    pipe.zadd(key, {str(now): now})               # Add current
    pipe.zcard(key)                                # Count
    pipe.expire(key, window)                       # TTL for cleanup
    results = pipe.execute()
    return results[2] > limit
```

### Strategies for Distributed Systems

| Strategy | Pros | Cons |
|----------|------|------|
| Centralized (Redis) | Globally consistent | Single point of failure, latency |
| Local + sync | Fast decisions | Eventually consistent, may overshoot |
| Sticky sessions | Simple | Uneven distribution |
| Token bucket with Redis | Burst-friendly, consistent | Redis dependency |

---

## Quota Management

Quotas define usage limits over longer periods (daily, monthly) and are typically tied to subscription tiers.

### Tiered Rate Limiting

| Tier | Requests/min | Requests/day | Burst Allowed |
|------|-------------|--------------|---------------|
| Free | 10 | 1,000 | No |
| Basic | 100 | 50,000 | 20 req burst |
| Pro | 1,000 | 500,000 | 100 req burst |
| Enterprise | 10,000 | Unlimited | 500 req burst |

### Multi-Level Limiting

Apply multiple limits simultaneously:

```python
def check_all_limits(client_id: str, tier: str) -> bool:
    limits = {
        "per_second": (tier_config[tier]["rps"], 1),
        "per_minute": (tier_config[tier]["rpm"], 60),
        "per_day": (tier_config[tier]["rpd"], 86400),
    }
    for name, (limit, window) in limits.items():
        if is_rate_limited(client_id, limit, window):
            return False
    return True
```

---

## Best Practices

1. **Always return rate limit headers** — clients need visibility into their usage
2. **Use 429 status code** — never use 503 for rate limiting
3. **Provide clear error messages** — include when the client can retry
4. **Identify clients consistently** — API key, JWT subject, or IP address
5. **Apply limits at multiple levels** — per-user, per-endpoint, global
6. **Monitor and alert** — track rejection rates to detect abuse or misconfiguration
7. **Allow quota increases** — provide a path for legitimate high-volume users
8. **Consider cost-based limiting** — expensive endpoints get lower limits

---

## Key Takeaways

- **Token bucket** is the most versatile algorithm — allows bursts while maintaining an average rate
- **Fixed window** is simple but vulnerable to boundary bursts; use sliding window variants for precision
- **Always communicate limits** via standard HTTP headers so clients can self-regulate
- **Exponential backoff with jitter** is the standard client-side strategy for handling 429 responses
- **Distributed rate limiting** requires shared state (typically Redis) for consistency across instances
- **Quota management** operates at a higher level than rate limiting — it governs total usage over business-relevant time periods
- Design rate limits as a **protection mechanism first**, revenue tool second
