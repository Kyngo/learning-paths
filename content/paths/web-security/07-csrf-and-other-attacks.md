---
title: "CSRF & Other Attacks"
weight: 7
---

## Cross-Site Request Forgery (CSRF)

CSRF tricks a victim's browser into making an unwanted request to a site where the victim is authenticated. The browser automatically attaches cookies (including session cookies), so the server cannot distinguish between a legitimate request and a forged one.

### How CSRF Works

```text
1. Victim logs into bank.com (session cookie set)
2. Victim visits evil.com
3. evil.com contains:
   <form action="https://bank.com/transfer" method="POST">
     <input type="hidden" name="to" value="attacker">
     <input type="hidden" name="amount" value="10000">
   </form>
   <script>document.forms[0].submit()</script>
4. Browser sends the POST to bank.com WITH the session cookie
5. bank.com processes the transfer — it sees a valid session
```

### CSRF Attack Variants

| Vector | Technique |
|--------|-----------|
| Auto-submitting form | `<form>` + `<script>document.forms[0].submit()</script>` |
| Image tag (GET only) | `<img src="https://bank.com/transfer?to=attacker&amount=10000">` |
| XMLHttpRequest | Blocked by CORS for non-simple requests, but simple requests (form POST) pass |
| Fetch API | Same CORS restrictions apply |

### Prevention: SameSite Cookies

The `SameSite` cookie attribute controls when cookies are sent with cross-site requests:

| Value | Behaviour | CSRF Protection |
|-------|-----------|-----------------|
| `Strict` | Never sent with cross-site requests | ✅ Complete — but breaks legitimate cross-site navigation |
| `Lax` | Sent only for top-level GET navigations | ✅ Strong — blocks cross-site POST, but allows link navigation |
| `None` | Always sent (requires `Secure`) | ❌ No protection |

```http
Set-Cookie: session_id=abc123; SameSite=Lax; Secure; HttpOnly; Path=/
```

**`Lax` is the recommended default** — it blocks CSRF via forms and AJAX while allowing normal link navigation.

### Prevention: CSRF Tokens

A unique, unpredictable token per session (or per request) that the server validates on state-changing requests:

```text
1. Server generates CSRF token, stores in session
2. Server embeds token in form as hidden field
3. Browser submits form with token
4. Server validates token matches session
5. Attacker's forged form does NOT have the token → rejected
```

**Python (Flask-WTF):**

```python
from flask_wtf.csrf import CSRFProtect

csrf = CSRFProtect(app)

# In template:
# <form method="POST">
#   {{ form.hidden_tag() }}   ← includes CSRF token
#   ...
# </form>
```

**Java (Spring Security) — enabled by default for server-rendered forms:**

```html
<!-- Thymeleaf — token included automatically -->
<form th:action="@{/transfer}" method="post">
    <!-- CSRF token is added automatically by Spring Security -->
    <input type="text" name="amount" />
    <button type="submit">Transfer</button>
</form>
```

**For SPAs (API-based):**

```javascript
// Read CSRF token from meta tag or cookie
const csrfToken = document.querySelector('meta[name="csrf-token"]')?.content;

fetch('/api/transfer', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-CSRF-Token': csrfToken,
  },
  body: JSON.stringify({ to: 'alice', amount: 100 }),
});
```

### Double-Submit Cookie Pattern

When server-side session storage is not available (stateless APIs):

```text
1. Server sets a random CSRF token as a cookie
2. Client reads the cookie and sends the same value in a header
3. Server checks cookie value == header value
4. Attacker cannot read the cookie (different origin) → cannot set header
```

```http
Set-Cookie: csrf_token=random123; SameSite=Strict; Secure; Path=/
```

```javascript
// Client reads cookie and sends as header
const csrfToken = getCookie('csrf_token');
fetch('/api/action', {
  method: 'POST',
  headers: { 'X-CSRF-Token': csrfToken },
  credentials: 'same-origin',
});
```

---

## Clickjacking

Clickjacking loads a target site in a transparent `<iframe>` overlaid on a decoy page. The victim thinks they are clicking on the decoy but are actually clicking on the hidden target.

### How Clickjacking Works

```html
<!-- Attacker's page -->
<style>
  iframe {
    position: absolute;
    top: 0; left: 0;
    width: 100%; height: 100%;
    opacity: 0;          /* Invisible */
    z-index: 10;         /* On top */
  }
</style>
<h1>Click here to win a prize!</h1>
<button>Claim Prize</button>

<!-- Hidden iframe with the target site -->
<iframe src="https://bank.com/settings/delete-account"></iframe>
```

### Prevention

**1. `X-Frame-Options` header:**

```http
X-Frame-Options: DENY           # Cannot be framed by anyone
X-Frame-Options: SAMEORIGIN     # Only framed by same origin
```

**2. CSP `frame-ancestors` directive (modern replacement):**

```http
Content-Security-Policy: frame-ancestors 'none'       # equivalent to DENY
Content-Security-Policy: frame-ancestors 'self'        # equivalent to SAMEORIGIN
Content-Security-Policy: frame-ancestors https://trusted.com
```

Use both headers for backward compatibility:

```http
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'
```

---

## Server-Side Request Forgery (SSRF)

SSRF occurs when an attacker tricks the server into making HTTP requests to internal or unintended destinations. The server acts as a proxy, bypassing firewalls and network controls.

### How SSRF Works

```text
# Application fetches a user-provided URL (e.g., "import from URL" feature)
POST /api/import
{ "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/" }
```

The server fetches the AWS metadata endpoint and returns IAM credentials to the attacker.

### SSRF Targets

| Target | What the Attacker Gets |
|--------|----------------------|
| `http://169.254.169.254/` | AWS instance metadata (IAM credentials) |
| `http://metadata.google.internal/` | GCP metadata |
| `http://localhost:8080/admin` | Internal admin panels |
| `http://10.0.0.5:6379/` | Internal Redis/database |
| `file:///etc/passwd` | Local file read (if `file://` protocol allowed) |

### SSRF Bypass Techniques

Attackers bypass naive URL validation:

| Bypass | Technique |
|--------|-----------|
| Decimal IP | `http://2130706433/` (127.0.0.1 in decimal) |
| IPv6 shorthand | `http://[::1]/` (localhost) |
| DNS rebinding | Domain resolves to public IP first, then internal IP |
| Redirect chain | `http://attacker.com` → 302 → `http://169.254.169.254/` |
| URL encoding | `http://127.0.0.%31/` |

### Prevention

```python
import ipaddress
from urllib.parse import urlparse

BLOCKED_NETWORKS = [
    ipaddress.ip_network('127.0.0.0/8'),
    ipaddress.ip_network('10.0.0.0/8'),
    ipaddress.ip_network('172.16.0.0/12'),
    ipaddress.ip_network('192.168.0.0/16'),
    ipaddress.ip_network('169.254.0.0/16'),   # Link-local (AWS metadata)
    ipaddress.ip_network('::1/128'),
]

def is_safe_url(url: str) -> bool:
    parsed = urlparse(url)

    # 1. Allowlist protocols
    if parsed.scheme not in ('http', 'https'):
        return False

    # 2. Resolve hostname to IP BEFORE making the request
    import socket
    try:
        ip = ipaddress.ip_address(socket.gethostbyname(parsed.hostname))
    except (socket.gaierror, ValueError):
        return False

    # 3. Block private/internal IPs
    for network in BLOCKED_NETWORKS:
        if ip in network:
            return False

    return True
```

**Additional controls:**
- Use IMDSv2 on AWS (requires token for metadata access)
- Network-level isolation (do not allow application servers to reach metadata endpoints)
- Allowlist target domains when possible

---

## Open Redirect

An open redirect occurs when the application redirects users to a URL specified in a parameter without validation. Attackers use this to phish users or steal OAuth tokens.

```text
https://trusted.com/login?redirect=https://evil.com/phishing
```

After login, the user is sent to `evil.com` — which looks like the trusted site.

### Prevention

```python
from urllib.parse import urlparse

ALLOWED_HOSTS = {'example.com', 'app.example.com'}

def is_safe_redirect(url: str) -> bool:
    parsed = urlparse(url)
    # Only allow relative paths or known hosts
    if not parsed.netloc:
        return True  # Relative URL
    return parsed.netloc in ALLOWED_HOSTS
```

**Alternatively, use only relative paths for redirects:**

```python
# Instead of: /login?redirect=https://example.com/dashboard
# Use:        /login?redirect=/dashboard

redirect_path = request.args.get('redirect', '/')
if redirect_path.startswith('/') and not redirect_path.startswith('//'):
    return redirect(redirect_path)
return redirect('/')
```

---

## Defence Summary

| Attack | Primary Defence | Secondary Defence |
|--------|----------------|-------------------|
| CSRF | SameSite=Lax cookies | CSRF tokens, double-submit cookie |
| Clickjacking | `frame-ancestors 'none'` (CSP) | `X-Frame-Options: DENY` |
| SSRF | Allowlist target URLs/domains | Block private IPs, IMDSv2, network isolation |
| Open Redirect | Allowlist redirect destinations | Use relative paths only |

---

## Key Takeaways

- **CSRF** exploits the browser's automatic cookie attachment — `SameSite=Lax` is the strongest single defence; combine with CSRF tokens for defence in depth.
- **Clickjacking** is prevented by `Content-Security-Policy: frame-ancestors 'none'` — set this header on all pages that should not be framed.
- **SSRF** turns your server into a proxy for attacking internal networks — resolve hostnames to IPs before making requests and block all private/reserved ranges.
- **Open redirects** are used for phishing and OAuth token theft — allowlist redirect destinations or restrict to relative paths.
- **No single defence is complete** — combine SameSite cookies, CSRF tokens, CSP frame-ancestors, and network-level controls for robust protection.
- **AWS IMDSv2** is essential when your application handles user-provided URLs — it requires a session token that SSRF attacks cannot easily obtain.
