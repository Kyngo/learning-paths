---
title: "Working with Databases (PDO)"
weight: 8
---

## PDO (PHP Data Objects)

PDO is PHP's database abstraction layer. It provides a consistent interface for MySQL, PostgreSQL, SQLite, and other databases.

### Connecting

```php
<?php
$dsn = 'mysql:host=localhost;dbname=myapp;charset=utf8mb4';
$pdo = new PDO($dsn, $user, $password, [
    PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,  // Throw exceptions on error
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,        // Return associative arrays
    PDO::ATTR_EMULATE_PREPARES   => false,                   // Use real prepared statements
]);
```

**Always set these three options.** Without `ERRMODE_EXCEPTION`, errors are silent. Without disabling emulated prepares, you lose type-safe parameter binding.

---

## Prepared Statements (Parameterized Queries)

**Never concatenate user input into SQL.** Use prepared statements to prevent SQL injection:

```php
<?php
// ✅ SAFE — parameterized
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email AND active = :active');
$stmt->execute(['email' => $email, 'active' => 1]);
$user = $stmt->fetch();

// ❌ DANGEROUS — SQL injection vulnerable
$stmt = $pdo->query("SELECT * FROM users WHERE email = '$email'");
```

### Positional vs Named Parameters

```php
<?php
// Named parameters (:name) — self-documenting
$stmt = $pdo->prepare('INSERT INTO users (name, email) VALUES (:name, :email)');
$stmt->execute(['name' => $name, 'email' => $email]);

// Positional parameters (?) — shorter
$stmt = $pdo->prepare('INSERT INTO users (name, email) VALUES (?, ?)');
$stmt->execute([$name, $email]);
```

---

## Fetching Results

```php
<?php
$stmt = $pdo->prepare('SELECT * FROM products WHERE category = ?');
$stmt->execute([$category]);

// Single row
$product = $stmt->fetch();  // returns array or false

// All rows
$products = $stmt->fetchAll();

// Single column value
$count = $pdo->query('SELECT COUNT(*) FROM users')->fetchColumn();

// Fetch into objects
$stmt->setFetchMode(PDO::FETCH_CLASS, Product::class);
$products = $stmt->fetchAll();

// Iterate without loading all into memory
$stmt = $pdo->prepare('SELECT * FROM large_table');
$stmt->execute();
while ($row = $stmt->fetch()) {
    process($row);
}
```

---

## Transactions

Group multiple operations into an atomic unit:

```php
<?php
try {
    $pdo->beginTransaction();
    
    $pdo->prepare('UPDATE accounts SET balance = balance - ? WHERE id = ?')
        ->execute([$amount, $fromId]);
    
    $pdo->prepare('UPDATE accounts SET balance = balance + ? WHERE id = ?')
        ->execute([$amount, $toId]);
    
    $pdo->prepare('INSERT INTO transfers (from_id, to_id, amount) VALUES (?, ?, ?)')
        ->execute([$fromId, $toId, $amount]);
    
    $pdo->commit();
} catch (\Throwable $e) {
    $pdo->rollBack();
    throw $e;
}
```

---

## CRUD Pattern

```php
<?php
class UserRepository
{
    public function __construct(private PDO $pdo) {}

    public function findById(int $id): ?array
    {
        $stmt = $this->pdo->prepare('SELECT * FROM users WHERE id = ?');
        $stmt->execute([$id]);
        return $stmt->fetch() ?: null;
    }

    public function create(string $name, string $email): int
    {
        $stmt = $this->pdo->prepare(
            'INSERT INTO users (name, email, created_at) VALUES (?, ?, NOW())'
        );
        $stmt->execute([$name, $email]);
        return (int) $this->pdo->lastInsertId();
    }

    public function update(int $id, string $name, string $email): bool
    {
        $stmt = $this->pdo->prepare(
            'UPDATE users SET name = ?, email = ?, updated_at = NOW() WHERE id = ?'
        );
        $stmt->execute([$name, $email, $id]);
        return $stmt->rowCount() > 0;
    }

    public function delete(int $id): bool
    {
        $stmt = $this->pdo->prepare('DELETE FROM users WHERE id = ?');
        $stmt->execute([$id]);
        return $stmt->rowCount() > 0;
    }
}
```

---

## Key Takeaways

1. **Always use prepared statements** — never concatenate variables into SQL strings
2. **Set `ERRMODE_EXCEPTION`** — silent failures hide bugs
3. **Disable emulated prepares** — real prepared statements provide type safety
4. **Use transactions** for multi-step operations that must succeed or fail together
5. **Fetch iteratively** for large result sets — `fetchAll()` loads everything into memory
6. **PDO is database-agnostic** — switch between MySQL, PostgreSQL, SQLite by changing the DSN
