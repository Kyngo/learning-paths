---
title: "Authentication & Authorization"
weight: 3
---

# Authentication & Authorization

Authentication (AuthN) verifies *who you are*. Authorization (AuthZ) determines *what you can do*. Every API needs both. This section covers the mechanisms from simple API keys to full OAuth2 flows.

## Authentication vs Authorization

| Aspect | Authentication (AuthN) | Authorization (AuthZ) |
|--------|----------------------|---------------------|
| Question | "Who is this?" | "Can they do this?" |
| Happens | First | After authentication |
| Mechanism | Credentials, tokens | Permissions, roles, policies |
| Failure code | 401 Unauthorized | 403 Forbidden |

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant AuthServer

    Client->>API: Request with credentials
    API->>AuthServer: Validate credentials (AuthN)
    AuthServer-->>API: Identity confirmed
    API->>API: Check permissions (AuthZ)
    API-->>Client: 200 OK or 403 Forbidden
```

## API Keys

The simplest authentication mechanism. A long, random string shared between client and server.

```http
# Via header (preferred)
GET /api/weather?city=barcelona
X-API-Key: sk_live_abc123def456ghi789

# Via query parameter (less secure — appears in logs)
GET /api/weather?city=barcelona&api_key=sk_live_abc123def456ghi789
```

### API Key Best Practices

| Practice | Reason |
|----------|--------|
| Use headers, not query params | Query params appear in server logs, browser history |
| Prefix keys with purpose | `sk_live_`, `sk_test_`, `pk_` (like Stripe) |
| Hash before storing | Never store plaintext keys in your database |
| Support key rotation | Allow multiple active keys during transition |
| Scope keys to permissions | Read-only key vs read-write key |
| Set expiry dates | Keys without expiry become security debt |

### When to Use API Keys

- Server-to-server communication
- Public APIs with usage tracking
- Simple integrations that don't need user-level permissions
- NOT for user-facing applications (keys can't represent user identity)

## OAuth 2.0

OAuth 2.0 is an authorization framework that enables third-party applications to obtain limited access to a service. It separates the role of the client from the resource owner (user).

### OAuth2 Roles

| Role | Description | Example |
|------|-------------|---------|
| Resource Owner | The user who owns the data | End user |
| Client | The application requesting access | Mobile app, SPA |
| Authorization Server | Issues tokens after authentication | Auth0, Keycloak |
| Resource Server | Hosts the protected API | Your API |

### OAuth2 Grant Types

```mermaid
flowchart TD
    A[What type of client?] --> B{Has a backend?}
    B -->|Yes, web server| C[Authorization Code]
    B -->|No, SPA/mobile| D[Authorization Code + PKCE]
    A --> E{Machine-to-machine?}
    E -->|Yes| F[Client Credentials]
    A --> G{Legacy, trusted first-party?}
    G -->|Yes| H[Resource Owner Password<br/>⚠️ Deprecated]
```

### Authorization Code Flow

Used by web applications with a server-side component:

```mermaid
sequenceDiagram
    participant User
    participant App
    participant AuthServer
    participant API

    User->>App: Click "Login"
    App->>AuthServer: Redirect to /authorize
    AuthServer->>User: Show login page
    User->>AuthServer: Enter credentials
    AuthServer->>App: Redirect with authorization code
    App->>AuthServer: Exchange code for tokens (+ client_secret)
    AuthServer-->>App: Access token + Refresh token
    App->>API: Request with access token
    API-->>App: Protected resource
```

```http
# Step 1: Redirect user to authorization server
GET https://auth.example.com/authorize?
  response_type=code&
  client_id=my_app&
  redirect_uri=https://myapp.com/callback&
  scope=read:orders write:orders&
  state=random_csrf_token

# Step 2: Exchange code for token
POST https://auth.example.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=AUTH_CODE_FROM_CALLBACK&
client_id=my_app&
client_secret=SECRET&
redirect_uri=https://myapp.com/callback
```

### Authorization Code with PKCE

PKCE (Proof Key for Code Exchange) adds security for public clients (SPAs, mobile apps) that can't safely store a `client_secret`:

```python
import hashlib, base64, secrets

# Client generates a code_verifier (random string)
code_verifier = secrets.token_urlsafe(64)

# Client derives code_challenge from verifier
code_challenge = base64.urlsafe_b64encode(
    hashlib.sha256(code_verifier.encode()).digest()
).decode().rstrip("=")

# Authorization request includes challenge
# GET /authorize?...&code_challenge=CHALLENGE&code_challenge_method=S256

# Token exchange includes verifier (proves same client)
# POST /token ... &code_verifier=VERIFIER
```

The authorization server verifies that `SHA256(code_verifier) == code_challenge`, proving the token requester is the same entity that initiated the flow.

### Client Credentials Flow

Machine-to-machine authentication. No user involved:

```http
POST https://auth.example.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&
client_id=service_orders&
client_secret=SECRET&
scope=read:inventory
```

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "read:inventory"
}
```

## JSON Web Tokens (JWT)

JWTs are the standard format for OAuth2 access tokens (and sometimes ID tokens). A JWT has three parts separated by dots:

```
header.payload.signature
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.signature_here
```

### JWT Structure

```json
// Header (algorithm + type)
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "key-2024-03"
}

// Payload (claims)
{
  "iss": "https://auth.example.com",
  "sub": "user_42",
  "aud": "https://api.example.com",
  "exp": 1710500000,
  "iat": 1710496400,
  "scope": "read:orders write:orders",
  "roles": ["customer", "premium"]
}

// Signature
RSASHA256(base64url(header) + "." + base64url(payload), private_key)
```

### Standard JWT Claims

| Claim | Name | Purpose |
|-------|------|---------|
| `iss` | Issuer | Who created the token |
| `sub` | Subject | Who the token represents (user ID) |
| `aud` | Audience | Intended recipient (your API) |
| `exp` | Expiration | When the token expires (Unix timestamp) |
| `iat` | Issued At | When the token was created |
| `nbf` | Not Before | Token not valid before this time |
| `jti` | JWT ID | Unique token identifier (for revocation) |

### JWT Validation Checklist

```python
def validate_jwt(token: str) -> dict:
    """Validate a JWT. Raises on any failure."""
    # 1. Decode header (without verification) to get algorithm + key ID
    header = decode_header(token)

    # 2. Fetch signing key (from JWKS endpoint, cached)
    key = get_signing_key(header["kid"])

    # 3. Verify signature + standard claims
    payload = jwt.decode(
        token,
        key=key,
        algorithms=["RS256"],       # Never allow "none" or HS256 with public key
        audience="https://api.example.com",
        issuer="https://auth.example.com",
    )

    # 4. Check expiration is handled by library (exp claim)
    # 5. Verify required custom claims
    if "scope" not in payload:
        raise InvalidToken("Missing scope claim")

    return payload
```

### Token Validation Flow

```mermaid
flowchart TD
    A[Receive JWT] --> B[Decode header]
    B --> C[Fetch public key via JWKS]
    C --> D{Signature valid?}
    D -->|No| E[401 Unauthorized]
    D -->|Yes| F{Token expired?}
    F -->|Yes| E
    F -->|No| G{Correct audience?}
    G -->|No| E
    G -->|Yes| H{Correct issuer?}
    H -->|No| E
    H -->|Yes| I[Token valid — extract claims]
    I --> J[Proceed to authorization]
```

## Token Refresh

Access tokens are short-lived (minutes to hours). Refresh tokens are long-lived and used to obtain new access tokens without re-authentication:

```http
POST https://auth.example.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&
refresh_token=REFRESH_TOKEN_VALUE&
client_id=my_app
```

| Token Type | Lifetime | Storage | Purpose |
|-----------|----------|---------|---------|
| Access token | 15 min – 1 hour | Memory | API access |
| Refresh token | Days – weeks | Secure HTTP-only cookie or keychain | Get new access tokens |
| ID token | Minutes | Memory | User profile info (OIDC) |

### Refresh Token Rotation

For extra security, issue a new refresh token with each use and invalidate the old one. If a stolen refresh token is used, the legitimate user's next refresh will fail — detecting the compromise.

## Scopes

Scopes limit what an access token can do:

```http
# Request specific scopes
GET /authorize?...&scope=read:orders write:orders read:profile

# Token will only have granted scopes
{
  "scope": "read:orders write:orders read:profile"
}
```

### Scope Design Patterns

| Pattern | Example | Use Case |
|---------|---------|----------|
| Resource:action | `read:orders`, `write:users` | Fine-grained per resource |
| Hierarchical | `orders`, `orders.read`, `orders.write` | Implied permissions |
| Feature-based | `analytics`, `admin`, `export` | Feature gating |

## RBAC vs ABAC

### Role-Based Access Control (RBAC)

Permissions are assigned to roles. Users are assigned roles.

```json
{
  "roles": {
    "viewer": ["read:orders", "read:products"],
    "editor": ["read:orders", "write:orders", "read:products", "write:products"],
    "admin": ["*"]
  },
  "users": {
    "user_42": ["editor"],
    "user_99": ["viewer", "editor"]
  }
}
```

### Attribute-Based Access Control (ABAC)

Access decisions based on attributes of the user, resource, action, and environment:

```python
# ABAC policy example
def can_access_order(user, order, action):
    # Users can only read their own orders
    if action == "read" and order.customer_id == user.id:
        return True
    # Managers can read orders from their department
    if action == "read" and user.role == "manager" and order.department == user.department:
        return True
    # Admins can do anything
    if user.role == "admin":
        return True
    return False
```

### RBAC vs ABAC Comparison

| Aspect | RBAC | ABAC |
|--------|------|------|
| Complexity | Low | High |
| Flexibility | Limited | Very flexible |
| Audit trail | Easy (who has what role) | Complex (evaluate policies) |
| Best for | Simple hierarchies | Multi-tenant, contextual access |
| Example | "Editors can write posts" | "Users can edit their own posts created in the last 24h" |

## Key Takeaways

1. **API keys are not user authentication** — use them for service identity and rate limiting, not for user sessions
2. **Always use PKCE for public clients** — SPAs and mobile apps cannot securely store client secrets
3. **Keep access tokens short-lived** — minutes, not days; use refresh tokens for longevity
4. **Validate JWTs completely** — signature, expiration, audience, and issuer; never trust unverified claims
5. **Design scopes early** — retrofitting fine-grained permissions is painful
6. **RBAC covers 80% of cases** — reach for ABAC when you need contextual, attribute-based decisions
7. **Never log tokens** — tokens are credentials; log the user ID and scope, not the bearer string
