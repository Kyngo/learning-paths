---
title: "Control Flow and Functions"
weight: 2
---

## Conditionals

### if / elseif / else

```php
<?php
if ($age >= 18) {
    echo "Adult";
} elseif ($age >= 13) {
    echo "Teenager";
} else {
    echo "Child";
}
```

### match Expression (PHP 8.0+)

`match` is a strict-comparison alternative to `switch` that returns a value and doesn't fall through:

```php
<?php
$status = match ($code) {
    200     => 'OK',
    301     => 'Moved Permanently',
    404     => 'Not Found',
    500     => 'Internal Server Error',
    default => 'Unknown',
};

// match with complex conditions
$label = match (true) {
    $temp > 35  => 'Hot',
    $temp > 20  => 'Warm',
    $temp > 10  => 'Cool',
    default     => 'Cold',
};
```

**match vs switch:**

| Feature | switch | match |
|---------|--------|-------|
| Comparison | Loose (`==`) | Strict (`===`) |
| Fall-through | Yes (needs `break`) | No |
| Returns value | No | Yes |
| Exhaustive | No | Yes (throws `UnhandledMatchError`) |

### Ternary and Short Ternary

```php
<?php
$label = $count > 0 ? 'has items' : 'empty';

// Short ternary (Elvis operator) — returns left if truthy, else right
$name = $input ?: 'Anonymous';
// Equivalent to: $name = $input ? $input : 'Anonymous';
```

---

## Loops

### for

```php
<?php
for ($i = 0; $i < 10; $i++) {
    echo $i;
}
```

### foreach

The workhorse loop for arrays:

```php
<?php
$colors = ['red', 'green', 'blue'];

// Values only
foreach ($colors as $color) {
    echo $color;
}

// Keys and values
foreach ($colors as $index => $color) {
    echo "$index: $color";
}

// Modifying values by reference
foreach ($colors as &$color) {
    $color = strtoupper($color);
}
unset($color); // Always unset reference variable after foreach!
```

> ⚠️ Forgetting `unset($color)` after a reference foreach is a classic PHP bug — the variable remains a reference and can be silently overwritten later.

### while and do-while

```php
<?php
while ($row = $statement->fetch()) {
    processRow($row);
}

do {
    $input = readline('Enter command: ');
} while ($input !== 'quit');
```

### Loop Control

```php
<?php
// break — exit the loop
// continue — skip to next iteration
// break 2 / continue 2 — affect outer loop (nested loops)

foreach ($items as $item) {
    if ($item->isInvalid()) {
        continue;
    }
    if ($item->isTerminator()) {
        break;
    }
    process($item);
}
```

---

## Functions

### Declaring Functions

```php
<?php
function calculateTax(float $amount, float $rate = 0.21): float
{
    return $amount * $rate;
}

$tax = calculateTax(100.00);        // 21.0
$tax = calculateTax(100.00, 0.10);  // 10.0
```

### Named Arguments (PHP 8.0+)

```php
<?php
// Skip optional params, pass in any order
$tax = calculateTax(rate: 0.10, amount: 100.00);

// Especially useful for functions with many optional params
array_slice(array: $data, offset: 2, length: 5, preserve_keys: true);
```

### Type Declarations

```php
<?php
function divide(int|float $a, int|float $b): float
{
    if ($b === 0) {
        throw new \DivisionByZeroError('Cannot divide by zero');
    }
    return $a / $b;
}

// Nullable types
function findUser(int $id): ?User
{
    return $this->repository->find($id); // returns User or null
}

// void return type
function logMessage(string $message): void
{
    file_put_contents('app.log', $message . PHP_EOL, FILE_APPEND);
}

// never return type (PHP 8.1+) — function never returns normally
function abort(int $code): never
{
    http_response_code($code);
    exit;
}
```

### Variadic Functions

```php
<?php
function sum(int|float ...$numbers): float
{
    return array_sum($numbers);
}

sum(1, 2, 3, 4, 5);  // 15

// Spread operator to pass array as arguments
$values = [1, 2, 3];
sum(...$values);  // 6
```

### Arrow Functions (PHP 7.4+)

Short closures that automatically capture variables from the outer scope by value:

```php
<?php
$multiplier = 3;

// Arrow function (implicit capture, single expression)
$triple = fn(int $n): int => $n * $multiplier;

// Equivalent long closure (explicit capture with 'use')
$triple = function (int $n) use ($multiplier): int {
    return $n * $multiplier;
};

// Common use: callbacks
$evens = array_filter($numbers, fn($n) => $n % 2 === 0);
$names = array_map(fn($user) => $user->name, $users);
```

### Closures and Variable Capture

```php
<?php
function makeCounter(): Closure
{
    $count = 0;
    
    return function () use (&$count): int {
        return ++$count;
    };
}

$counter = makeCounter();
echo $counter(); // 1
echo $counter(); // 2
echo $counter(); // 3
```

**Important:** `use ($var)` captures by value (snapshot). `use (&$var)` captures by reference (shared state).

### First-Class Callable Syntax (PHP 8.1+)

```php
<?php
// Create a Closure from any callable
$strlen = strlen(...);    // Closure wrapping strlen()
$upper = strtoupper(...);

$lengths = array_map($strlen, ['foo', 'bar', 'baz']);  // [3, 3, 3]

// Works with methods too
$formatter = $dateObject->format(...);
```

---

## Scope

### Variable Scope

PHP has function-level scope (no block scope for variables):

```php
<?php
$outer = 'visible';

function inner(): void
{
    // $outer is NOT accessible here — different scope
    echo $outer; // Warning: undefined variable
}

// To access outer variables, you must:
// 1. Pass as parameter (preferred)
// 2. Use 'use' in a closure
// 3. Use 'global' keyword (avoid — makes testing impossible)
```

### Static Variables

Persist across function calls without being global:

```php
<?php
function getNextId(): int
{
    static $id = 0;
    return ++$id;
}

echo getNextId(); // 1
echo getNextId(); // 2
echo getNextId(); // 3
```

---

## Generators

Generators allow lazy iteration without building entire arrays in memory:

```php
<?php
function fibonacci(): Generator
{
    $a = 0;
    $b = 1;
    
    while (true) {
        yield $a;
        [$a, $b] = [$b, $a + $b];
    }
}

// Only computes values on demand
$fib = fibonacci();
for ($i = 0; $i < 10; $i++) {
    echo $fib->current() . ' ';
    $fib->next();
}
// Output: 0 1 1 2 3 5 8 13 21 34

// Reading a large file line-by-line without loading it all into memory
function readLines(string $file): Generator
{
    $handle = fopen($file, 'r');
    try {
        while (($line = fgets($handle)) !== false) {
            yield trim($line);
        }
    } finally {
        fclose($handle);
    }
}
```

---

## Key Takeaways

1. **`match` over `switch`** — strict comparison, returns a value, no fall-through bugs
2. **Always `unset()` reference variables** after `foreach (&$item)` loops
3. **Named arguments** make code self-documenting and let you skip optional params
4. **Arrow functions (`fn()`)** auto-capture from outer scope — use for short callbacks
5. **`use (&$var)` for reference capture** in closures when you need shared mutable state
6. **Generators** solve memory problems — use them for large datasets and infinite sequences
7. **Type declarations** on all function signatures — they catch bugs at call time, not deep in logic
