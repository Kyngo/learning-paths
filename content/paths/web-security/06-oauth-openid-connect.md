---
title: "OAuth 2.0 & OpenID Connect"
weight: 6
---

## Overview

**OAuth 2.0** is an authorisation framework that allows third-party applications to obtain limited access to a user's resources without exposing their credentials. **OpenID Connect (OIDC)** is an identity layer built on top of OAuth 2.0 that adds authentication.

| Protocol | Purpose | Result |
|----------|---------|--------|
| OAuth 2.0 | Authorisation (what can the app do?) | Access Token |
| OpenID Connect | Authentication (who is the user?) | ID Token + Access Token |

---

## OAuth 2.0 Roles

| Role | Description |
|------|-------------|
| **Resource Owner** | The user who owns the data |
| **Client** | The application requesting access |
| **Authorization Server** | Issues tokens (e.g., Keycloak, Auth0, Okta) |
| **Resource Server** | The API that holds protected resources |

---

## Grant Types

### Authorization Code (with PKCE) — Recommended for All Clients

This is the **only recommended grant** for browser-based and mobile applications. PKCE (Proof Key for Code Exchange) prevents authorisation code interception.

```text
┌──────────┐                              ┌──────────────────┐
│  Browser  │                              │  Auth Server      │
│  (Client) │                              │  (Keycloak/Auth0) │
└─────┬─────┘                              └────────┬─────────┘
      │                                             │
      │  1. Generate code_verifier (random)         │
      │     code_challenge = SHA256(code_verifier)  │
      │                                             │
      │  2. GET /authorize                          │
      │     ?response_type=code                     │
      │     &client_id=my-app                       │
      │     &redirect_uri=https://app.com/callback  │
      │     &scope=openid profile email             │
      │     &state=random_csrf_value                │
      │     &code_challenge=...                     │
      │     &code_challenge_method=S256             │
      │────────────────────────────────────────────▶│
      │                                             │
      │  3. User logs in + consents                 │
      │◀────────────────────────────────────────────│
      │                                             │
      │  4. Redirect to callback with code          │
      │     https://app.com/callback                │
      │     ?code=AUTH_CODE&state=random_csrf_value  │
      │                                             │
      │  5. POST /token                             │
      │     grant_type=authorization_code            │
      │     &code=AUTH_CODE                          │
      │     &redirect_uri=https://app.com/callback  │
      │     &code_verifier=original_verifier         │
      │────────────────────────────────────────────▶│
      │                                             │
      │  6. { access_token, id_token, refresh_token }│
      │◀────────────────────────────────────────────│
```

### Client Credentials — Machine-to-Machine

For server-to-server communication where no user is involved.

```text
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=backend-service
&client_secret=<secret>
&scope=api.read
```

Returns an access token representing the **application itself**, not a user.

### Implicit Grant — Deprecated

The implicit grant returned tokens directly in the URL fragment. It is **deprecated** in the OAuth 2.0 Security Best Current Practice (RFC 9700) because:

- Tokens exposed in browser history and logs
- No mechanism to verify the client
- No refresh tokens

**Always use Authorization Code + PKCE instead.**

### Grant Type Decision

| Scenario | Grant Type |
|----------|-----------|
| Single-page app (SPA) | Authorization Code + PKCE |
| Mobile app | Authorization Code + PKCE |
| Server-side web app | Authorization Code (+ PKCE recommended) |
| Service-to-service | Client Credentials |
| CLI tool (interactive) | Device Authorization Grant |

---

## Tokens

### Access Token

A short-lived credential that authorises API requests. Can be opaque (random string) or a JWT.

```http
GET /api/bookings
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

### ID Token (OpenID Connect)

A JWT containing claims about the authenticated user. **Never send the ID token to an API** — it is for the client application to identify the user.

```json
{
  "iss": "https://auth.example.com",
  "sub": "user-123",
  "aud": "my-client-app",
  "exp": 1724770000,
  "iat": 1724766400,
  "name": "Alice Smith",
  "email": "alice@example.com",
  "email_verified": true
}
```

### Refresh Token

A long-lived token used to obtain new access tokens without re-authenticating. Must be stored securely.

| Token | Lifetime | Storage | Sent To |
|-------|----------|---------|---------|
| Access Token | 5–60 minutes | Memory (SPA) or HttpOnly cookie | Resource Server (API) |
| ID Token | 5–60 minutes | Memory (SPA) | Client only — never sent to API |
| Refresh Token | Hours to days | HttpOnly cookie (rotation required) | Auth Server only |

---

## Scopes

Scopes limit what an access token can do:

```text
scope=openid profile email read:bookings write:bookings
```

| Scope | Purpose |
|-------|---------|
| `openid` | Required for OIDC — returns an ID token |
| `profile` | User's name, picture, etc. |
| `email` | User's email and verified status |
| `offline_access` | Issues a refresh token |
| Custom (`read:bookings`) | Application-specific permissions |

**Principle of least privilege applies:** request only the scopes your application actually needs.

---

## Token Storage (Client-Side)

This is one of the most debated topics in web security. The right answer depends on your architecture.

| Storage | XSS Risk | CSRF Risk | Recommendation |
|---------|----------|-----------|----------------|
| `localStorage` | ❌ Vulnerable (JS can read) | ✅ Immune | ❌ Avoid for access tokens |
| `sessionStorage` | ❌ Vulnerable (JS can read) | ✅ Immune | ❌ Avoid for access tokens |
| `HttpOnly` cookie | ✅ Immune (JS cannot read) | ❌ Vulnerable | ✅ Use with SameSite + CSRF token |
| In-memory variable | ✅ Immune (lost on refresh) | ✅ Immune | ✅ Best for SPAs (short sessions) |
| BFF pattern (server-side) | ✅ Immune | ✅ Managed server-side | ✅ Best overall for SPAs |

### Backend-for-Frontend (BFF) Pattern

The SPA never handles tokens directly. A backend proxy manages the OAuth flow and stores tokens in server-side sessions:

```text
┌──────────┐     ┌─────────────┐     ┌──────────────┐     ┌──────────┐
│  Browser  │────▶│  BFF (Nuxt  │────▶│  Auth Server  │     │  API     │
│  (SPA)    │◀────│  Server)    │◀────│  (Keycloak)   │     │  Server  │
│           │     │             │────────────────────────▶  │          │
│  Cookie:  │     │  Stores     │     │               │     │          │
│  session  │     │  tokens in  │     └──────────────┘     └──────────┘
│  only     │     │  server     │
└──────────┘     │  session    │
                  └─────────────┘
```

---

## Common OAuth Mistakes

### 1. Missing `state` Parameter

Without `state`, an attacker can initiate an OAuth flow and have the victim complete it, linking the attacker's account.

```text
# Always include state
state = generate_random_string(32)
session['oauth_state'] = state
# Include in /authorize request, verify on callback
```

### 2. Open Redirect via `redirect_uri`

If the auth server accepts arbitrary redirect URIs, an attacker can steal the authorisation code:

```text
/authorize?redirect_uri=https://evil.com/steal
```

**Prevention:** register redirect URIs in advance; match exactly (no wildcards).

### 3. Token Leakage in Logs

Access tokens in URLs are logged by proxies, CDNs, and browsers:

```text
# DANGEROUS — token in URL
GET /api/data?access_token=eyJ...

# SAFE — token in header
GET /api/data
Authorization: Bearer eyJ...
```

### 4. Insufficient Token Validation

| Check | Why |
|-------|-----|
| Verify `iss` (issuer) | Reject tokens from unknown auth servers |
| Verify `aud` (audience) | Reject tokens intended for other clients |
| Verify `exp` (expiration) | Reject expired tokens |
| Verify signature | Reject tampered or forged tokens |
| Check `scope` or `permissions` | Enforce least privilege |

**Node.js validation example:**

```javascript
import { jwtVerify } from 'jose';

const JWKS = createRemoteJWKSet(new URL('https://auth.example.com/.well-known/jwks.json'));

async function validateToken(token) {
  const { payload } = await jwtVerify(token, JWKS, {
    issuer: 'https://auth.example.com',
    audience: 'my-api',
    algorithms: ['RS256'],
  });
  return payload;
}
```

### 5. Refresh Token Without Rotation

If a refresh token is stolen and never rotated, the attacker has indefinite access.

**Prevention:** enable **refresh token rotation** — each use of a refresh token invalidates the old one and issues a new one. Detect reuse of an already-rotated token as a sign of compromise and revoke the entire family.

---

## OIDC Discovery

Compliant providers expose their configuration at a well-known endpoint:

```text
GET https://auth.example.com/.well-known/openid-configuration
```

```json
{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "https://auth.example.com/authorize",
  "token_endpoint": "https://auth.example.com/token",
  "userinfo_endpoint": "https://auth.example.com/userinfo",
  "jwks_uri": "https://auth.example.com/.well-known/jwks.json",
  "scopes_supported": ["openid", "profile", "email"],
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "client_credentials", "refresh_token"]
}
```

Use this endpoint to dynamically configure your OAuth client — do not hardcode endpoint URLs.

---

## Key Takeaways

- **Authorization Code + PKCE** is the only recommended grant for browser and mobile applications — the Implicit Grant is deprecated.
- **Client Credentials** is for machine-to-machine flows where no user is involved.
- **Never store access tokens in `localStorage`** — use HttpOnly cookies with SameSite, in-memory variables, or the BFF pattern.
- **Always validate tokens** server-side: verify issuer, audience, expiration, signature, and scope.
- **Enable refresh token rotation** and detect reuse to limit the blast radius of token theft.
- **Register exact redirect URIs** with the authorisation server — open redirects enable authorisation code theft.
