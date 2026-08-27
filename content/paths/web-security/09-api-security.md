---
title: "API Security"
weight: 9
---

## Why API Security Is Different

APIs are the primary attack surface of modern applications. Unlike server-rendered pages, APIs expose raw business logic directly. They are consumed by browsers, mobile apps, third-party integrations, and automated scripts — each with different trust levels.

The OWASP API Security Top 10 (2023) highlights risks specific to APIs that the standard Top 10 does not adequately cover.

---

## Input Validation

Every API endpoint must validate every input. Never trust the client — not the browser, not the mobile app, not a trusted partner's integration.

### Validation Strategy

| Layer | What to Validate | Example |
|-------|-----------------|---------|
| **Schema** | Types, required fields, formats | JSON Schema, Zod, Pydantic |
| **Business rules** | Ranges, relationships, invariants | `quantity > 0`, `end_date > start_date` |
| **Size limits** | Payload size, string length, array size | `Content-Length < 1MB`, `name.length <= 100` |

### Implementation Examples

**Python (Pydantic):**

```python
from pydantic import BaseModel, Field, field_validator
from datetime import date

class CreateBookingRequest(BaseModel):
    destination: str = Field(min_length=1, max_length=200)
    passengers: int = Field(ge=1, le=50)
    start_date: date
    end_date: date

    @field_validator('end_date')
    @classmethod
    def end_after_start(cls, v, info):
        if 'start_date' in info.data and v <= info.data['start_date']:
            raise ValueError('end_date must be after start_date')
        return v
```

**Java (Jakarta Validation):**

```java
public record CreateBookingRequest(
    @NotBlank @Size(max = 200) String destination,
    @Min(1) @Max(50) int passengers,
    @NotNull @Future LocalDate startDate,
    @NotNull @Future LocalDate endDate
) {}
```

**Node.js (Zod):**

```typescript
import { z } from 'zod';

const CreateBookingSchema = z.object({
  destination: z.string().min(1).max(200),
  passengers: z.number().int().min(1).max(50),
  startDate: z.string().date(),
  endDate: z.string().date(),
}).refine(
  (data) => new Date(data.endDate) > new Date(data.startDate),
  { message: 'endDate must be after startDate' }
);
```

---

## Rate Limiting

Rate limiting prevents abuse, brute force, and denial of service. Apply different limits based on the endpoint's sensitivity.

### Common Strategies

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| **Fixed window** | N requests per time window | Simple, general use |
| **Sliding window** | Smooth rate across window boundaries | More accurate limiting |
| **Token bucket** | Tokens replenish at fixed rate; burst allowed | APIs with occasional bursts |
| **Leaky bucket** | Requests processed at constant rate; excess queued | Consistent throughput |

### Tiered Limits

| Endpoint Category | Limit | Reasoning |
|-------------------|-------|-----------|
| Login / auth | 5/min per IP + 10/min per account | Brute force protection |
| Password reset | 3/hour per email | Prevent abuse |
| Search | 30/min per user | Expensive queries |
| Standard API | 100/min per user | Normal usage |
| Webhook receivers | 1000/min per source | High volume, trusted |

### Response Headers (RFC 6585 / draft-ietf-httpapi-ratelimit-headers)

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 1724770000
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/rate-limit",
  "title": "Rate limit exceeded",
  "status": 429,
  "detail": "You have exceeded 100 requests per minute. Retry after 30 seconds."
}
```

### Implementation (Express.js)

```javascript
import rateLimit from 'express-rate-limit';

const apiLimiter = rateLimit({
  windowMs: 60 * 1000,    // 1 minute
  max: 100,                // 100 requests per window
  standardHeaders: true,   // RateLimit-* headers
  legacyHeaders: false,
  keyGenerator: (req) => req.user?.id || req.ip,
  handler: (req, res) => {
    res.status(429).json({
      type: 'https://api.example.com/errors/rate-limit',
      title: 'Rate limit exceeded',
      status: 429,
    });
  },
});

app.use('/api/', apiLimiter);
```

---

## API Keys vs OAuth

| Aspect | API Key | OAuth 2.0 Token |
|--------|---------|-----------------|
| Represents | An application | A user (or application for client_credentials) |
| Granularity | Coarse (entire app) | Fine (scoped per user, per permission) |
| Revocation | Rotate the key (all users affected) | Revoke individual tokens |
| User context | ❌ No user identity | ✅ User identity in token claims |
| Expiration | Usually long-lived | Short-lived (minutes to hours) |
| Best for | Server-to-server, rate limiting, analytics | User-facing APIs, delegated access |

**API keys are not authentication** — they identify the calling application but not the user. For user-facing APIs, use OAuth 2.0.

---

## CORS (Cross-Origin Resource Sharing)

CORS controls which origins can make requests to your API from a browser. Without CORS headers, browsers block cross-origin AJAX requests.

### How CORS Works

```text
Browser (https://app.com)              API (https://api.example.com)
  │                                        │
  │── Preflight (OPTIONS) ────────────────▶│
  │   Origin: https://app.com              │
  │   Access-Control-Request-Method: POST  │
  │                                        │
  │◀── Preflight Response ────────────────│
  │    Access-Control-Allow-Origin: https://app.com
  │    Access-Control-Allow-Methods: GET, POST
  │    Access-Control-Allow-Headers: Content-Type, Authorization
  │    Access-Control-Max-Age: 86400       │
  │                                        │
  │── Actual Request (POST) ──────────────▶│
  │   Origin: https://app.com              │
  │                                        │
  │◀── Response ──────────────────────────│
  │    Access-Control-Allow-Origin: https://app.com
```

### CORS Configuration

```javascript
// Express.js
import cors from 'cors';

app.use(cors({
  origin: ['https://app.example.com', 'https://admin.example.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,        // Allow cookies
  maxAge: 86400,            // Cache preflight for 24 hours
}));
```

### CORS Mistakes

| Mistake | Risk |
|---------|------|
| `Access-Control-Allow-Origin: *` with `credentials: true` | Browsers block this — but trying it reveals misunderstanding |
| Reflecting `Origin` header without validation | Any site can make authenticated requests |
| Allowing `null` origin | `file://` pages and sandboxed iframes match `null` |
| Overly broad allowlist | Subdomain takeover on `*.example.com` compromises the API |

**Safe pattern — validate against an explicit allowlist:**

```javascript
const ALLOWED_ORIGINS = new Set([
  'https://app.example.com',
  'https://admin.example.com',
]);

app.use(cors({
  origin: (origin, callback) => {
    if (!origin || ALLOWED_ORIGINS.has(origin)) {
      callback(null, true);
    } else {
      callback(new Error('CORS not allowed'));
    }
  },
}));
```

---

## GraphQL Security

GraphQL introduces unique security concerns because clients control query structure.

### Key Risks

| Risk | Description | Mitigation |
|------|-------------|------------|
| **Deeply nested queries** | Query depth causes exponential DB queries | Query depth limiting (max 10–15 levels) |
| **Expensive queries** | Single request fetches massive data | Query complexity analysis, cost limits |
| **Introspection** | Schema exposed to attackers | Disable in production |
| **Batching attacks** | Multiple mutations in one request | Limit batch size, rate limit per operation |
| **Injection** | Arguments passed to resolvers unsafely | Parameterised queries in resolvers |

### Query Depth and Complexity Limits

```javascript
// Apollo Server with depth limiting
import depthLimit from 'graphql-depth-limit';
import { createComplexityLimitRule } from 'graphql-validation-complexity';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    depthLimit(10),
    createComplexityLimitRule(1000),
  ],
  introspection: process.env.NODE_ENV !== 'production',
});
```

### Disable Introspection in Production

```graphql
# Attacker's introspection query — should be blocked
{
  __schema {
    types {
      name
      fields { name type { name } }
    }
  }
}
```

---

## Webhook Verification

When receiving webhooks from external services, verify the request is authentic:

### HMAC Signature Verification

```python
import hmac
import hashlib

def verify_webhook(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256,
    ).hexdigest()

    # Constant-time comparison to prevent timing attacks
    return hmac.compare_digest(f"sha256={expected}", signature)

# In your handler:
@app.post("/webhooks/stripe")
async def stripe_webhook(request: Request):
    body = await request.body()
    signature = request.headers.get("Stripe-Signature", "")

    if not verify_webhook(body, signature, STRIPE_WEBHOOK_SECRET):
        raise HTTPException(status_code=401, detail="Invalid signature")

    event = json.loads(body)
    # Process the event
```

### Webhook Security Checklist

| ✅ | Control |
|---|---------|
| ☐ | Verify HMAC signature on every request |
| ☐ | Use constant-time comparison (`hmac.compare_digest`) |
| ☐ | Reject requests with missing or invalid signatures |
| ☐ | Store webhook secrets in secrets manager |
| ☐ | Validate event freshness (reject events older than N minutes) |
| ☐ | Return 2xx quickly, process asynchronously |
| ☐ | Idempotent processing (handle duplicate deliveries) |

---

## Key Takeaways

- **Validate all input** with schema validation libraries (Pydantic, Zod, Jakarta Validation) — type checks, range limits, and business rules on every endpoint.
- **Rate limit by endpoint sensitivity** — authentication endpoints need strict per-IP and per-account limits; standard endpoints need per-user limits.
- **API keys identify applications, not users** — use OAuth 2.0 tokens for user-facing APIs with scoped permissions.
- **CORS must use an explicit origin allowlist** — never reflect the `Origin` header without validation, and never use `*` with credentials.
- **GraphQL requires additional controls** — depth limiting, complexity analysis, and disabled introspection in production.
- **Verify webhook signatures** using HMAC with constant-time comparison — never trust the payload without cryptographic proof of authenticity.
