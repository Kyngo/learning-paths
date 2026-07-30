---
title: "Object-Oriented PHP"
weight: 4
---

## Classes and Objects

```php
<?php
class User
{
    // Properties with visibility and types
    private string $name;
    private string $email;
    private readonly int $id;  // readonly (PHP 8.1+) — set once, never changed

    // Constructor with promotion (PHP 8.0+)
    public function __construct(
        private string $name,
        private string $email,
        private readonly int $id = 0,
    ) {}

    // Methods
    public function getDisplayName(): string
    {
        return "{$this->name} <{$this->email}>";
    }

    // Static methods
    public static function createGuest(): self
    {
        return new self('Guest', 'guest@example.com');
    }
}

$user = new User(name: 'Alice', email: 'alice@example.com', id: 1);
$guest = User::createGuest();
```

### Constructor Property Promotion (PHP 8.0+)

Eliminates the boilerplate of declaring properties, assigning in constructor:

```php
<?php
// Before PHP 8.0 (verbose)
class Point {
    private float $x;
    private float $y;
    
    public function __construct(float $x, float $y) {
        $this->x = $x;
        $this->y = $y;
    }
}

// PHP 8.0+ (promoted — property declaration, type, and assignment in one)
class Point {
    public function __construct(
        private float $x,
        private float $y,
    ) {}
}
```

---

## Visibility

| Modifier | Access |
|----------|--------|
| `public` | Anywhere |
| `protected` | Class + subclasses |
| `private` | Class only |

PHP 8.4 introduces **asymmetric visibility:**

```php
<?php
class User
{
    // Publicly readable, privately writable
    public private(set) string $name;
    
    // Publicly readable, only settable by class and subclasses
    public protected(set) int $age;
}
```

---

## Inheritance

```php
<?php
abstract class Shape
{
    abstract public function area(): float;
    
    public function describe(): string
    {
        return static::class . " with area " . $this->area();
    }
}

class Circle extends Shape
{
    public function __construct(private float $radius) {}
    
    public function area(): float
    {
        return M_PI * $this->radius ** 2;
    }
}

class Rectangle extends Shape
{
    public function __construct(
        private float $width,
        private float $height,
    ) {}
    
    public function area(): float
    {
        return $this->width * $this->height;
    }
}
```

---

## Interfaces

Interfaces define a contract — what methods a class must implement:

```php
<?php
interface Serializable
{
    public function serialize(): string;
    public function unserialize(string $data): void;
}

interface Cacheable
{
    public function getCacheKey(): string;
    public function getCacheTtl(): int;
}

// A class can implement multiple interfaces
class Product implements Serializable, Cacheable
{
    public function serialize(): string { /* ... */ }
    public function unserialize(string $data): void { /* ... */ }
    public function getCacheKey(): string { return "product:{$this->id}"; }
    public function getCacheTtl(): int { return 3600; }
}
```

---

## Traits

Traits provide horizontal code reuse (mixins):

```php
<?php
trait Timestampable
{
    private ?DateTimeImmutable $createdAt = null;
    private ?DateTimeImmutable $updatedAt = null;

    public function markCreated(): void
    {
        $this->createdAt = new DateTimeImmutable();
    }

    public function markUpdated(): void
    {
        $this->updatedAt = new DateTimeImmutable();
    }
}

trait SoftDeletable
{
    private ?DateTimeImmutable $deletedAt = null;
    
    public function softDelete(): void
    {
        $this->deletedAt = new DateTimeImmutable();
    }
    
    public function isDeleted(): bool
    {
        return $this->deletedAt !== null;
    }
}

class Article
{
    use Timestampable, SoftDeletable;
    
    public function __construct(
        private string $title,
        private string $body,
    ) {
        $this->markCreated();
    }
}
```

---

## Enums (PHP 8.1+)

```php
<?php
// Pure enum (no value)
enum Status
{
    case Draft;
    case Published;
    case Archived;
}

// Backed enum (string or int value)
enum Color: string
{
    case Red = 'red';
    case Green = 'green';
    case Blue = 'blue';
    
    // Enums can have methods
    public function label(): string
    {
        return match ($this) {
            self::Red   => 'Red Color',
            self::Green => 'Green Color',
            self::Blue  => 'Blue Color',
        };
    }
}

// Usage
$status = Status::Published;
$color = Color::from('red');         // Color::Red (throws on invalid)
$color = Color::tryFrom('purple');   // null (safe)

// Enums implement interfaces
enum Permission: string implements HasLabel
{
    case Read = 'read';
    case Write = 'write';
    case Admin = 'admin';
    
    public function label(): string
    {
        return ucfirst($this->value);
    }
}
```

---

## Magic Methods

```php
<?php
class Entity
{
    private array $attributes = [];

    public function __get(string $name): mixed
    {
        return $this->attributes[$name] ?? null;
    }

    public function __set(string $name, mixed $value): void
    {
        $this->attributes[$name] = $value;
    }

    public function __isset(string $name): bool
    {
        return isset($this->attributes[$name]);
    }

    public function __toString(): string
    {
        return json_encode($this->attributes);
    }

    public function __clone(): void
    {
        // Deep clone nested objects if needed
    }

    public function __debugInfo(): array
    {
        return ['attributes' => array_keys($this->attributes)];
    }
}
```

---

## Readonly Classes (PHP 8.2+)

All properties become readonly:

```php
<?php
readonly class Coordinate
{
    public function __construct(
        public float $latitude,
        public float $longitude,
    ) {}
}

$coord = new Coordinate(41.3874, 2.1686);
// $coord->latitude = 0;  // Error: cannot modify readonly property
```

---

## Key Takeaways

1. **Constructor promotion** eliminates property boilerplate — use it for all DTOs and value objects
2. **`readonly`** properties enforce immutability at the language level
3. **Interfaces for contracts**, traits for shared behavior, abstract classes for partial implementations
4. **Enums** replace string/int constants — they're type-safe and support methods
5. **Prefer composition** (interfaces + injection) over deep inheritance hierarchies
6. **`static::class`** uses late static binding — gives the actual class name, not the parent
