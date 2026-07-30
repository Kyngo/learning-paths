---
title: "The Type System (PHP 8.x)"
weight: 6
---

## Evolution of PHP Types

PHP started as a loosely-typed scripting language. Modern PHP has a sophisticated gradual type system:

| PHP Version | Type Feature Added |
|-------------|-------------------|
| 5.0 | Class/interface type hints (parameters only) |
| 7.0 | Scalar types (int, string, float, bool), return types, strict mode |
| 7.1 | Nullable (`?Type`), `void`, `iterable` |
| 7.2 | `object` type |
| 7.4 | Typed properties |
| 8.0 | Union types (`int\|string`), `mixed`, `static` return, `null`, `false` as standalone |
| 8.1 | Intersection types (`A&B`), `never`, enums, fibers |
| 8.2 | DNF types (Disjunctive Normal Form), `null`, `true`, `false` standalone |
| 8.3 | Typed class constants |
| 8.4 | Property hooks |

---

## Strict Types

By default, PHP coerces values. Enable strict mode per-file:

```php
<?php
declare(strict_types=1);

function add(int $a, int $b): int
{
    return $a + $b;
}

add(1, 2);     // OK
add(1, "2");   // TypeError! (without strict_types, "2" would be coerced to 2)
add(1, 2.0);   // TypeError! (float is not int in strict mode)
```

**Always use `declare(strict_types=1)`** at the top of every PHP file. It catches type bugs at the call site.

---

## Union Types (PHP 8.0+)

A value can be one of several types:

```php
<?php
function process(int|string $id): void { /* ... */ }

function find(int $id): User|null  // equivalent to ?User
{
    return $this->repo->find($id);
}

// Common union: accepting array or single value
function normalize(string|array $input): array
{
    return is_array($input) ? $input : [$input];
}
```

---

## Intersection Types (PHP 8.1+)

A value must satisfy ALL listed types simultaneously:

```php
<?php
// Must implement BOTH Countable AND Iterator
function processCollection(Countable&Iterator $items): void
{
    echo "Processing {$items->count()} items...";
    foreach ($items as $item) {
        handle($item);
    }
}
```

---

## DNF Types (PHP 8.2+)

Combine union and intersection (Disjunctive Normal Form):

```php
<?php
// Must be (A & B) OR null
function getCollection(): (Countable&Iterator)|null
{
    // ...
}
```

---

## Type Narrowing

PHP narrows types after checks:

```php
<?php
function process(string|int|null $value): string
{
    if ($value === null) {
        return 'nothing';  // narrowed: $value is null here
    }
    
    if (is_int($value)) {
        return (string) ($value * 2);  // narrowed: $value is int here
    }
    
    return strtoupper($value);  // narrowed: $value is string here
}
```

### instanceof for Objects

```php
<?php
if ($shape instanceof Circle) {
    // PHP knows $shape is Circle — can access Circle-specific methods
    echo $shape->getRadius();
}
```

---

## Generics (Missing — Workarounds)

PHP does not have generics at the language level. Workarounds:

```php
<?php
/**
 * @template T
 * @param class-string<T> $class
 * @return T
 */
function create(string $class): object
{
    return new $class();
}

/**
 * @template T
 * @param T[] $items
 * @return T|null
 */
function first(array $items): mixed
{
    return $items[0] ?? null;
}
```

Static analysis tools (PHPStan, Psalm) understand these `@template` annotations and enforce them at analysis time.

---

## Static Analysis

Since PHP's type system is gradual (runtime-checked), static analysis tools fill the gaps:

### PHPStan

```bash
# Install
composer require --dev phpstan/phpstan

# Run
vendor/bin/phpstan analyse src/ --level=9
```

PHPStan levels (0–9) — higher = stricter:

| Level | Checks |
|-------|--------|
| 0 | Basic checks (unknown classes, functions) |
| 1 | Possibly undefined variables |
| 3 | Return types |
| 5 | Dead code, unreachable statements |
| 6 | Missing typehints |
| 8 | Report nullable issues |
| 9 | Maximum strictness |

### Psalm

```bash
composer require --dev vimeo/psalm
vendor/bin/psalm --init
vendor/bin/psalm
```

Both tools support generics via PHPDoc annotations, taint analysis (security), and custom rules.

---

## Key Takeaways

1. **`declare(strict_types=1)`** on every file — catches type coercion bugs
2. **Union types** for parameters that accept multiple types, intersection for contracts requiring multiple interfaces
3. **PHP's type system is gradual** — types are checked at runtime, not compile time
4. **Static analysis (PHPStan/Psalm)** is essential — it provides the safety net PHP's runtime can't
5. **No generics at language level** — use `@template` PHPDoc annotations for static analysis tools
6. **Type narrowing** happens automatically after `is_*()` checks and `instanceof`
