---
title: "Transport Security"
weight: 8
---

## Why Transport Security Matters

Data in transit is vulnerable to interception, modification, and replay. Without transport security, an attacker on the same network (coffee shop Wi-Fi, compromised router, ISP-level surveillance) can read passwords, session tokens, and personal data in plaintext.

Transport security is not optional — it is the foundation on which all other web security controls depend.

---

## TLS (Transport Layer Security)

TLS encrypts the communication channel between client and server. HTTPS is HTTP over TLS.

### TLS Handshake (TLS 1.3, Simplified)

```text
Client                                     Server
  │                                          │
  │── ClientHello ──────────────────────────▶│
  │   (supported cipher suites, key share)   │
  │                                          │
  │◀── ServerHello ──────────────────────────│
  │    (chosen cipher, key share,            │
  │     certificate, finished)               │
  │                                          │
  │── Finished ─────────────────────────────▶│
  │                                          │
  │◀═══════ Encrypted Application Data ═════▶│
```

TLS 1.3 completes the handshake in **1 round trip** (1-RTT), compared to 2 round trips in TLS 1.2. It also supports **0-RTT** for resumed connections (with replay risk caveats).

### TLS Version Comparison

| Version | Status | Key Exchange | Cipher Suites | Round Trips |
|---------|--------|-------------|---------------|-------------|
| TLS 1.0 | ❌ Deprecated | RSA, DHE | RC4, 3DES, AES | 2-RTT |
| TLS 1.1 | ❌ Deprecated | RSA, DHE | 3DES, AES | 2-RTT |
| TLS 1.2 | ✅ Acceptable | RSA, DHE, ECDHE | AES-GCM, ChaCha20 | 2-RTT |
| TLS 1.3 | ✅ Recommended | ECDHE, DHE only | AES-128-GCM, AES-256-GCM, ChaCha20-Poly1305 | 1-RTT |

**Recommendation:** require TLS 1.2 minimum, prefer TLS 1.3. Disable TLS 1.0 and 1.1 entirely.

### Nginx TLS Configuration

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/ssl/certs/example.com.pem;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    # Protocol versions
    ssl_protocols TLSv1.2 TLSv1.3;

    # Cipher suites (TLS 1.2 — TLS 1.3 suites are fixed by the protocol)
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers on;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 1.1.1.1 8.8.8.8 valid=300s;

    # Session resumption
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;  # Disabled for forward secrecy
}
```

---

## HSTS (HTTP Strict Transport Security)

HSTS tells browsers to **only** connect to the site over HTTPS — even if the user types `http://`. It prevents SSL-stripping attacks where an attacker downgrades the connection to HTTP.

```http
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

| Directive | Meaning |
|-----------|---------|
| `max-age=63072000` | Remember for 2 years (in seconds) |
| `includeSubDomains` | Apply to all subdomains |
| `preload` | Request inclusion in browser HSTS preload lists |

### HSTS Preloading

Even with HSTS, the **first** visit to a site is vulnerable (the browser has not yet received the header). HSTS preloading solves this by hardcoding your domain into the browser itself.

Submit at https://hstspreload.org/ — requirements:
1. Valid HTTPS certificate
2. Redirect HTTP to HTTPS on the same host
3. HSTS header on the HTTPS response with `max-age >= 31536000`, `includeSubDomains`, and `preload`
4. All subdomains must support HTTPS

---

## Certificate Pinning

Certificate pinning binds a server to a specific certificate or public key, preventing man-in-the-middle attacks using fraudulent certificates.

### Where Pinning Makes Sense

| Context | Recommendation |
|---------|---------------|
| Mobile apps (native) | ✅ Pin to the leaf or intermediate certificate |
| Browsers (web) | ❌ HTTP Public Key Pinning (HPKP) is deprecated — too risky |
| Service-to-service (internal) | ✅ Pin to internal CA or specific certificate |

### Mobile App Pinning (Android)

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2027-01-01">
            <pin digest="SHA-256">base64EncodedSHA256OfSubjectPublicKeyInfo=</pin>
            <pin digest="SHA-256">backupPinForCertificateRotation=</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

---

## Mixed Content

Mixed content occurs when an HTTPS page loads resources over HTTP. This breaks the security guarantee — an attacker can modify the HTTP resource.

| Type | Behaviour | Example |
|------|-----------|---------|
| **Active** (scripts, iframes) | Blocked by browsers | `<script src="http://cdn.com/app.js">` |
| **Passive** (images, video) | Loaded with warning | `<img src="http://cdn.com/photo.jpg">` |

### Prevention

```http
Content-Security-Policy: upgrade-insecure-requests
```

This directive tells the browser to automatically upgrade HTTP requests to HTTPS. Combine with a CSP that restricts sources:

```http
Content-Security-Policy: default-src https:; upgrade-insecure-requests
```

---

## Security Headers

A comprehensive set of HTTP security headers for production:

```http
# Transport
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload

# Content restrictions
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; upgrade-insecure-requests
X-Content-Type-Options: nosniff
X-Frame-Options: DENY

# Information leakage
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=()

# Legacy (still useful for older browsers)
X-XSS-Protection: 0
```

### Header Reference

| Header | Purpose | Recommended Value |
|--------|---------|-------------------|
| `Strict-Transport-Security` | Force HTTPS | `max-age=63072000; includeSubDomains; preload` |
| `Content-Security-Policy` | Restrict content sources | Customise per application |
| `X-Content-Type-Options` | Prevent MIME type sniffing | `nosniff` |
| `X-Frame-Options` | Prevent clickjacking | `DENY` |
| `Referrer-Policy` | Control referrer information | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | Disable browser features | Deny unused features |
| `X-XSS-Protection` | Legacy XSS filter | `0` (disable — modern CSP is better) |

### Setting Headers in Application Frameworks

**Express.js (Helmet):**

```javascript
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
      frameAncestors: ["'none'"],
    },
  },
  strictTransportSecurity: {
    maxAge: 63072000,
    includeSubDomains: true,
    preload: true,
  },
}));
```

**Spring Boot:**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.headers(headers -> headers
            .httpStrictTransportSecurity(hsts -> hsts
                .includeSubDomains(true)
                .maxAgeInSeconds(63072000)
                .preload(true))
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; frame-ancestors 'none'"))
            .frameOptions(frame -> frame.deny())
            .referrerPolicy(referrer -> referrer
                .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
            .permissionsPolicy(permissions -> permissions
                .policy("camera=(), microphone=(), geolocation=()"))
        );
        return http.build();
    }
}
```

---

## Cookie Security Attributes

| Attribute | Purpose | Recommended |
|-----------|---------|-------------|
| `Secure` | Only sent over HTTPS | Always |
| `HttpOnly` | Not accessible via JavaScript | Always for session cookies |
| `SameSite=Lax` | Limits cross-site sending | Default for most cookies |
| `SameSite=Strict` | Never sent cross-site | For highly sensitive cookies |
| `Path=/` | Scopes to entire site | Avoid overly narrow paths |
| `Domain` | Omit to limit to exact host | Do not set unless needed for subdomains |
| `Max-Age` / `Expires` | Cookie lifetime | As short as practical |
| `__Host-` prefix | Enforces Secure + no Domain + Path=/ | Best for session cookies |
| `__Secure-` prefix | Enforces Secure | Good for other cookies |

### Cookie Prefixes

```http
# __Host- prefix: most restrictive
# Must be: Secure, no Domain attribute, Path=/
Set-Cookie: __Host-session=abc123; Secure; HttpOnly; SameSite=Lax; Path=/

# __Secure- prefix: requires Secure only
Set-Cookie: __Secure-token=xyz789; Secure; HttpOnly; SameSite=Lax; Path=/
```

---

## Testing Transport Security

**Check TLS configuration:**

```bash
# SSL Labs (comprehensive online test)
# https://www.ssllabs.com/ssltest/

# Command-line with testssl.sh
./testssl.sh https://example.com

# OpenSSL client
openssl s_client -connect example.com:443 -tls1_3

# Check certificate details
openssl s_client -connect example.com:443 </dev/null 2>/dev/null | openssl x509 -text -noout
```

**Check security headers:**

```bash
# curl
curl -I https://example.com

# securityheaders.com (online scanner)
# https://securityheaders.com/?q=example.com
```

---

## Key Takeaways

- **TLS 1.2 minimum, TLS 1.3 preferred** — disable TLS 1.0 and 1.1 entirely; use ECDHE key exchange for forward secrecy.
- **HSTS with preloading** prevents SSL-stripping attacks — set `max-age` to at least 2 years with `includeSubDomains` and submit for preload.
- **Security headers** are a low-effort, high-impact defence — deploy `Content-Security-Policy`, `Strict-Transport-Security`, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, and `Permissions-Policy` on every response.
- **Cookie security attributes** (`Secure`, `HttpOnly`, `SameSite`, `__Host-` prefix) are essential — misconfigured cookies are the root cause of many authentication and CSRF vulnerabilities.
- **Mixed content** breaks the HTTPS security guarantee — use `upgrade-insecure-requests` in CSP and audit all resource URLs.
- **Test your configuration** with SSL Labs, testssl.sh, and securityheaders.com before and after deployment.
