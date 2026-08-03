---
title: "REST Principles"
weight: 1
---

# REST Principles

REST (Representational State Transfer) is an architectural style defined by Roy Fielding in his 2000 doctoral dissertation. It describes a set of constraints that, when applied to web services, yield systems that are scalable, loosely coupled, and simple to understand.

## The Six REST Constraints

REST is not a protocol or a standard — it's a set of architectural constraints. A service that satisfies all constraints is called *RESTful*.

| Constraint | Description | Benefit |
|-----------|-------------|---------|
| Client-Server | Separation of concerns between UI and data storage | Independent evolution of client and server |
| Stateless | Each request contains all information needed to process it | Scalability, reliability, visibility |
| Cacheable | Responses must define themselves as cacheable or not | Reduced latency, improved efficiency |
| Uniform Interface | Standardised way to interact with resources | Simplicity, decoupling |
| Layered System | Client cannot tell if connected directly to server | Load balancing, shared caches, security |
| Code on Demand (optional) | Server can extend client functionality with executable code | Client extensibility |

### Statelessness in Practice

```http
# BAD — relies on server-side session state
GET /api/next-page
Cookie: session_id=abc123

# GOOD — request is self-contained
GET /api/orders?page=3&per_page=20
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

Every request carries its own authentication, pagination state, and context. The server never stores conversational state between requests.

## Resources and URIs

In REST, everything is a **resource** — an entity that can be identified, named, and manipulated. Resources are identified by URIs (Uniform Resource Identifiers).

### Resource Naming Conventions

| Convention | Example | Rule |
|-----------|---------|------|
| Nouns, not verbs | `/orders` not `/getOrders` | Resources are things, not actions |
| Plural collection names | `/users`, `/products` | Collections contain multiple items |
| Hierarchical relationships | `/users/42/orders` | Express ownership/containment |
| Lowercase with hyphens | `/order-items` not `/orderItems` | URI standard convention |
| No trailing slashes | `/users` not `/users/` | Consistency |
| No file extensions | `/users/42` not `/users/42.json` | Content negotiation via headers |

### URI Design Examples

```
# Collection
GET /api/v1/customers

# Single resource
GET /api/v1/customers/42

# Sub-resource (nested relationship)
GET /api/v1/customers/42/orders

# Specific sub-resource
GET /api/v1/customers/42/orders/7

# Filtering on a collection
GET /api/v1/orders?status=shipped&created_after=2024-01-01
```

## HTTP Methods (Verbs)

REST maps CRUD operations to HTTP methods. Each method has defined semantics:

| Method | Purpose | Safe | Idempotent | Request Body | Typical Status |
|--------|---------|------|------------|--------------|----------------|
| GET | Retrieve resource(s) | ✅ | ✅ | No | 200 OK |
| POST | Create a new resource | ❌ | ❌ | Yes | 201 Created |
| PUT | Replace a resource entirely | ❌ | ✅ | Yes | 200 OK |
| PATCH | Partially update a resource | ❌ | ❌* | Yes | 200 OK |
| DELETE | Remove a resource | ❌ | ✅ | No | 204 No Content |
| HEAD | Same as GET without body | ✅ | ✅ | No | 200 OK |
| OPTIONS | Describe communication options | ✅ | ✅ | No | 204 No Content |

*PATCH can be made idempotent depending on the patch format used.

**Safe** = does not modify server state. **Idempotent** = calling N times has the same effect as calling once.

### Method Examples

```http
# Create a new order
POST /api/orders
Content-Type: application/json

{
  "customer_id": 42,
  "items": [{"product_id": 101, "quantity": 2}]
}

# Response
HTTP/1.1 201 Created
Location: /api/orders/1234

# Full replacement
PUT /api/orders/1234
Content-Type: application/json

{
  "customer_id": 42,
  "items": [{"product_id": 101, "quantity": 3}],
  "status": "confirmed"
}

# Partial update
PATCH /api/orders/1234
Content-Type: application/json

{
  "status": "shipped"
}
```

## HTTP Status Codes

Status codes communicate the result of a request. Use them precisely:

| Range | Category | Common Codes |
|-------|----------|-------------|
| 2xx | Success | 200 OK, 201 Created, 202 Accepted, 204 No Content |
| 3xx | Redirection | 301 Moved Permanently, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 429 Too Many Requests |
| 5xx | Server Error | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout |

### Choosing the Right Status Code

```mermaid
flowchart TD
    A[Request received] --> B{Valid request?}
    B -->|No| C{Auth issue?}
    C -->|No credentials| D[401 Unauthorized]
    C -->|No permission| E[403 Forbidden]
    C -->|Bad syntax| F[400 Bad Request]
    C -->|Semantic error| G[422 Unprocessable Entity]
    B -->|Yes| H{Resource exists?}
    H -->|No| I[404 Not Found]
    H -->|Yes| J{Operation succeeded?}
    J -->|Yes, data returned| K[200 OK]
    J -->|Yes, created| L[201 Created]
    J -->|Yes, no body| M[204 No Content]
    J -->|Accepted for later| N[202 Accepted]
    J -->|No, conflict| O[409 Conflict]
    J -->|No, server error| P[500 Internal Server Error]
```

## HATEOAS

Hypermedia As The Engine Of Application State — the most misunderstood (and least implemented) REST constraint. The server provides links in responses that tell the client what actions are available.

```json
{
  "id": 1234,
  "status": "confirmed",
  "total": 89.99,
  "_links": {
    "self": { "href": "/api/orders/1234" },
    "customer": { "href": "/api/customers/42" },
    "cancel": { "href": "/api/orders/1234/cancel", "method": "POST" },
    "items": { "href": "/api/orders/1234/items" }
  }
}
```

The `cancel` link only appears when the order is in a cancellable state. The client doesn't need to encode business rules about when cancellation is allowed — the presence of the link communicates it.

### Benefits of HATEOAS

- Client discovers available actions dynamically
- Server can change URLs without breaking clients
- API becomes self-documenting at runtime
- Workflow state is communicated through available transitions

## Richardson Maturity Model

Leonard Richardson proposed a model that grades APIs by how well they use HTTP and REST principles:

| Level | Name | Description | Example |
|-------|------|-------------|---------|
| 0 | The Swamp of POX | Single URI, single method (usually POST) | SOAP-style XML-RPC |
| 1 | Resources | Multiple URIs, but only POST | `/createUser`, `/getUser` |
| 2 | HTTP Verbs | Multiple URIs with proper HTTP methods | `GET /users`, `POST /users` |
| 3 | Hypermedia Controls | HATEOAS — responses include links | Full REST with `_links` |

Most production APIs sit at Level 2. Level 3 adds discoverability but increases response size and complexity.

```mermaid
graph LR
    L0[Level 0<br/>POX] --> L1[Level 1<br/>Resources]
    L1 --> L2[Level 2<br/>HTTP Verbs]
    L2 --> L3[Level 3<br/>HATEOAS]
    style L2 fill:#e3f2fd,stroke:#1565c0
    style L3 fill:#e8f5e1,stroke:#1b5e20
```

## Idempotency

An operation is **idempotent** if performing it multiple times produces the same result as performing it once. This is critical for reliability — if a network failure occurs, the client can safely retry without side effects.

| Method | Idempotent | Why |
|--------|-----------|-----|
| GET | ✅ | Reading doesn't change state |
| PUT | ✅ | Replaces entire resource — same input = same result |
| DELETE | ✅ | Deleting an already-deleted resource = no change |
| POST | ❌ | Creates a new resource each time |
| PATCH | ❌* | Depends on the patch semantics |

### Making Non-Idempotent Operations Safe

Use **idempotency keys** for POST requests:

```http
POST /api/payments
Idempotency-Key: 7c4d8f2e-1a3b-4c5d-9e8f-0a1b2c3d4e5f
Content-Type: application/json

{
  "amount": 99.99,
  "currency": "EUR",
  "recipient": "merchant_42"
}
```

The server checks if it has already processed a request with this key. If yes, it returns the original response without processing again. This prevents double-charging even if the client retries.

### Idempotency Key Flow

```mermaid
sequenceDiagram
    participant Client
    participant Server
    participant DB

    Client->>Server: POST /payments (Key: abc-123)
    Server->>DB: Check key abc-123
    DB-->>Server: Not found
    Server->>DB: Process payment + store key
    Server-->>Client: 201 Created

    Note over Client: Network timeout, client retries

    Client->>Server: POST /payments (Key: abc-123)
    Server->>DB: Check key abc-123
    DB-->>Server: Found (already processed)
    Server-->>Client: 201 Created (cached response)
```

## Content Negotiation

Clients and servers negotiate the representation format using HTTP headers:

```http
# Client requests JSON
GET /api/orders/42
Accept: application/json

# Client requests XML
GET /api/orders/42
Accept: application/xml

# Server responds with chosen format
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
```

Use `Accept` to request a format and `Content-Type` to declare what you're sending. Don't encode format in the URL (`/api/orders/42.json` — this breaks the uniform interface).

## Key Takeaways

1. **REST is constraints, not a specification** — there's no "REST standard" to comply with, only principles to follow
2. **Statelessness enables scale** — every request is independent, so any server instance can handle it
3. **HTTP methods have semantics** — use them correctly (GET reads, POST creates, PUT replaces, DELETE removes)
4. **Status codes communicate intent** — a 404 means "not found", a 403 means "you can't", a 409 means "conflict"
5. **Idempotency enables reliability** — safe retries require idempotent operations or idempotency keys
6. **Resources are nouns, methods are verbs** — `POST /orders` not `POST /createOrder`
7. **HATEOAS decouples clients from URL structures** — but pragmatically, most APIs stop at Richardson Level 2
