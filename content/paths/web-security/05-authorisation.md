---
title: "Authorisation"
weight: 5
---

## Authentication vs Authorisation

Authentication proves **who you are**. Authorisation determines **what you can do**. A system can have perfect authentication and still be completely broken if authorisation is missing or bypassable.

Broken access control was the #1 risk in the OWASP Top 10 (2021), overtaking injection for the first time.

---

## Access Control Models

### Role-Based Access Control (RBAC)

Permissions are assigned to roles, and users are assigned to roles. Simple, widely understood, and sufficient for most applications.

```text
┌───────────┐     ┌──────────┐     ┌──────────────┐
│   User    │────▶│   Role   │────▶│  Permission  │
│  (alice)  │     │  (editor) │     │  (post:write) │
└───────────┘     └──────────┘     └──────────────┘
```

**Database schema example:**

```sql
CREATE TABLE roles (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(50) UNIQUE NOT NULL  -- 'admin', 'editor', 'viewer'
);

CREATE TABLE user_roles (
    user_id     INTEGER REFERENCES users(id),
    role_id     INTEGER REFERENCES roles(id),
    PRIMARY KEY (user_id, role_id)
);

CREATE TABLE role_permissions (
    role_id     INTEGER REFERENCES roles(id),
    permission  VARCHAR(100) NOT NULL,       -- 'post:read', 'post:write', 'user:delete'
    PRIMARY KEY (role_id, permission)
);
```

**Checking permissions (Python):**

```python
def has_permission(user_id: int, permission: str) -> bool:
    return db.execute("""
        SELECT 1 FROM user_roles ur
        JOIN role_permissions rp ON ur.role_id = rp.role_id
        WHERE ur.user_id = %s AND rp.permission = %s
        LIMIT 1
    """, (user_id, permission)).fetchone() is not None
```

### Attribute-Based Access Control (ABAC)

Permissions are evaluated against attributes of the user, the resource, the action, and the environment. More flexible than RBAC but more complex.

```text
Policy: "Allow if user.department == resource.department AND time.hour BETWEEN 9 AND 17"
```

| Component | Attributes |
|-----------|-----------|
| **Subject** | role, department, clearance level, location |
| **Resource** | owner, classification, department, type |
| **Action** | read, write, delete, approve |
| **Environment** | time of day, IP address, device type |

**Example policy evaluation:**

```python
def can_access(user, resource, action) -> bool:
    # Rule 1: Admins can do anything
    if 'admin' in user.roles:
        return True

    # Rule 2: Users can read resources in their department
    if action == 'read' and user.department == resource.department:
        return True

    # Rule 3: Owners can edit their own resources
    if action == 'write' and resource.owner_id == user.id:
        return True

    return False
```

### RBAC vs ABAC Comparison

| Aspect | RBAC | ABAC |
|--------|------|------|
| Complexity | Low | High |
| Granularity | Coarse (role-level) | Fine (attribute-level) |
| Scalability | Role explosion with many resources | Scales to complex policies |
| Auditability | Easy — who has which role | Harder — policies can be subtle |
| Best for | Most web applications | Multi-tenant, regulated, complex |

---

## Broken Access Control

### Insecure Direct Object References (IDOR)

IDOR occurs when the application exposes internal object identifiers (database IDs, filenames) and does not verify that the requesting user is authorised to access them.

```text
GET /api/invoices/12345
Authorization: Bearer <token_for_user_alice>
```

If the server returns invoice 12345 without checking that Alice owns it, **any authenticated user** can access **any invoice** by changing the ID.

**Vulnerable code:**

```python
@app.get("/api/invoices/{invoice_id}")
def get_invoice(invoice_id: int, user: User = Depends(get_current_user)):
    # VULNERABLE — no ownership check
    return db.query(Invoice).get(invoice_id)
```

**Fixed code:**

```python
@app.get("/api/invoices/{invoice_id}")
def get_invoice(invoice_id: int, user: User = Depends(get_current_user)):
    invoice = db.query(Invoice).filter(
        Invoice.id == invoice_id,
        Invoice.user_id == user.id  # Ownership check
    ).first()
    if not invoice:
        raise HTTPException(status_code=404)
    return invoice
```

### Horizontal vs Vertical Privilege Escalation

| Type | Definition | Example |
|------|-----------|---------|
| **Horizontal** | Accessing another user's resources at the same privilege level | User A views User B's invoices |
| **Vertical** | Gaining higher privileges than granted | Regular user accesses admin panel |

### Forced Browsing

Accessing pages or endpoints that are not linked in the UI but exist on the server:

```text
/admin
/api/debug
/api/users/export
/backup/database.sql
/.env
```

**Prevention:** authorisation checks on every endpoint, not just the ones visible in the UI.

---

## Privilege Escalation Patterns

### Parameter Tampering

```text
# Request to update profile
POST /api/users/me
{
  "name": "Alice",
  "email": "alice@example.com",
  "role": "admin"              ← attacker adds this field
}
```

**Prevention — allowlist updatable fields:**

```python
# Python (Pydantic)
class UpdateProfileRequest(BaseModel):
    name: str
    email: EmailStr
    # 'role' is NOT included — cannot be set via this endpoint

@app.put("/api/users/me")
def update_profile(data: UpdateProfileRequest, user: User = Depends(get_current_user)):
    user.name = data.name
    user.email = data.email
    db.commit()
```

```java
// Java — use a DTO without the role field
public record UpdateProfileRequest(
    @NotBlank String name,
    @Email String email
    // No 'role' field — cannot be set
) {}
```

### JWT Role Manipulation

If the application trusts the JWT payload without server-side verification:

```json
{
  "sub": "alice",
  "role": "user"     ← attacker changes to "admin"
}
```

**Prevention:** always verify the JWT signature server-side. Never trust payload claims without cryptographic verification.

---

## JWT Security Pitfalls

### Common JWT Mistakes

| Mistake | Consequence | Prevention |
|---------|-------------|------------|
| `alg: none` accepted | Attacker sends unsigned token | Reject `none` algorithm; validate `alg` against allowlist |
| Symmetric key too short | Brute-forceable with hashcat | ≥256-bit keys for HMAC; prefer RS256/ES256 |
| Secret in source code | Key leaked via git | Store in secrets manager (AWS SSM, Vault) |
| No expiration (`exp`) | Token valid forever | Always set `exp`; keep short (15–60 min) |
| Token in URL | Logged in server logs, referer headers | Send in `Authorization` header or `HttpOnly` cookie |
| Trusting `kid` header blindly | SQL injection or path traversal via `kid` | Validate `kid` against allowlist |

### `alg: none` Attack

```json
// Original token header
{ "alg": "HS256", "typ": "JWT" }

// Attacker changes to
{ "alg": "none", "typ": "JWT" }
```

If the library accepts `none`, the token is treated as valid without a signature.

**Prevention (Node.js — jose):**

```javascript
import { jwtVerify } from 'jose';

const { payload } = await jwtVerify(token, secretKey, {
  algorithms: ['HS256'],  // Explicit allowlist — rejects 'none'
});
```

### Key Confusion Attack (RS256 → HS256)

If a server uses RS256 (asymmetric), an attacker can:

1. Obtain the public key (often exposed at `/.well-known/jwks.json`)
2. Forge a token signed with HMAC-SHA256 using the public key as the HMAC secret
3. If the library uses the `alg` header to select verification, it verifies successfully

**Prevention:** always enforce the expected algorithm server-side.

---

## Authorisation Enforcement Patterns

### Middleware / Decorator Pattern

**Python (FastAPI):**

```python
from functools import wraps

def require_permission(permission: str):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, user: User = Depends(get_current_user), **kwargs):
            if not has_permission(user.id, permission):
                raise HTTPException(status_code=403, detail="Forbidden")
            return await func(*args, user=user, **kwargs)
        return wrapper
    return decorator

@app.delete("/api/posts/{post_id}")
@require_permission("post:delete")
async def delete_post(post_id: int, user: User = Depends(get_current_user)):
    ...
```

**Java (Spring Security):**

```java
@RestController
@RequestMapping("/api/posts")
public class PostController {

    @DeleteMapping("/{id}")
    @PreAuthorize("hasAuthority('post:delete')")
    public ResponseEntity<Void> deletePost(@PathVariable UUID id) {
        // ...
    }
}
```

### Row-Level Security

Database-enforced access control — PostgreSQL example:

```sql
-- Enable RLS on the invoices table
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;

-- Users can only see their own invoices
CREATE POLICY user_isolation ON invoices
    USING (user_id = current_setting('app.current_user_id')::int);
```

---

## Key Takeaways

- **Broken access control** is the #1 web application risk — authorisation checks must exist on every endpoint, not just the UI.
- **IDOR** is prevented by always including an ownership or permission check in database queries — never trust user-supplied IDs alone.
- **RBAC** is sufficient for most applications; use **ABAC** when policies depend on contextual attributes (department, time, resource classification).
- **Allowlist updatable fields** in request DTOs to prevent parameter tampering and mass assignment attacks.
- **JWT tokens** must be verified server-side with an explicit algorithm allowlist — reject `alg: none` and enforce the expected signing algorithm.
- **Enforce authorisation at the data layer** in addition to the application layer — PostgreSQL Row-Level Security provides a strong last line of defence.
