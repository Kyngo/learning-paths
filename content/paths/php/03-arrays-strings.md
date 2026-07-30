---
title: "Arrays and Strings"
weight: 3
---

## Arrays

PHP arrays are the most versatile data structure in the language. They're actually **ordered hash maps** — they can act as lists, dictionaries, stacks, queues, sets, and more.

### Creating Arrays

```php
<?php
// Short syntax (preferred)
$fruits = ['apple', 'banana', 'cherry'];

// Associative array (key => value)
$person = [
    'name' => 'Alice',
    'age'  => 30,
    'city' => 'Barcelona',
];

// Mixed keys (possible but confusing — avoid)
$mixed = [0 => 'zero', 'key' => 'value', 1 => 'one'];

// Nested arrays
$matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9],
];
```

### Accessing and Modifying

```php
<?php
$fruits[0];            // 'apple'
$person['name'];       // 'Alice'
$matrix[1][2];         // 6

// Append
$fruits[] = 'date';    // adds at next integer key

// Modify
$person['age'] = 31;

// Remove
unset($fruits[1]);     // removes 'banana', does NOT reindex!
// After unset: [0 => 'apple', 2 => 'cherry', 3 => 'date']

// Reindex after unset
$fruits = array_values($fruits);
```

### Array Destructuring

```php
<?php
// List assignment (positional)
[$first, $second] = ['hello', 'world'];

// Skip elements
[, , $third] = [1, 2, 3];  // $third = 3

// Associative destructuring
['name' => $name, 'age' => $age] = $person;

// In foreach
$points = [['x' => 1, 'y' => 2], ['x' => 3, 'y' => 4]];
foreach ($points as ['x' => $x, 'y' => $y]) {
    echo "($x, $y) ";
}
```

### Essential Array Functions

#### Searching and Filtering

```php
<?php
in_array('apple', $fruits);              // true (loose comparison)
in_array('apple', $fruits, true);        // true (strict — preferred)
array_key_exists('name', $person);       // true
array_search('cherry', $fruits);         // returns key or false

// Filter with callback
$adults = array_filter($users, fn($u) => $u['age'] >= 18);

// Filter without callback removes falsy values
$clean = array_filter([0, '', null, 'hello', false, 42]);  // ['hello', 42]
```

#### Transforming

```php
<?php
// Map: transform each element
$upper = array_map(fn($s) => strtoupper($s), $fruits);

// Map with keys (use array_map with null + multiple arrays, or foreach)
$result = [];
foreach ($data as $key => $value) {
    $result[$key] = transform($value);
}

// Reduce: fold into a single value
$total = array_reduce($prices, fn($carry, $item) => $carry + $item, 0);

// Column: extract a single column from nested arrays
$names = array_column($users, 'name');
$indexed = array_column($users, null, 'id');  // re-index by 'id' field
```

#### Sorting

```php
<?php
sort($arr);        // by value, reindex
rsort($arr);       // reverse, reindex
asort($arr);       // by value, preserve keys
arsort($arr);      // reverse by value, preserve keys
ksort($arr);       // by key
krsort($arr);      // reverse by key

// Custom sort
usort($users, fn($a, $b) => $a['age'] <=> $b['age']);

// Sort by multiple criteria
usort($users, function ($a, $b) {
    return $a['city'] <=> $b['city']
        ?: $a['name'] <=> $b['name'];
});
```

#### Combining and Splitting

```php
<?php
$merged = array_merge($a, $b);         // reindexes numeric keys
$combined = $a + $b;                    // preserves keys, left wins on conflict
$replaced = array_replace($a, $b);     // right wins on conflict

$chunk = array_chunk($large, 100);     // split into groups of 100
$slice = array_slice($arr, 2, 5);      // 5 elements starting at index 2

$keys = array_keys($person);           // ['name', 'age', 'city']
$values = array_values($person);       // ['Alice', 30, 'Barcelona']
$flipped = array_flip($arr);           // swap keys and values
$unique = array_unique($arr);          // remove duplicates
```

### The Spread Operator in Arrays (PHP 7.4+)

```php
<?php
$first = [1, 2, 3];
$second = [4, 5, 6];
$all = [...$first, ...$second];  // [1, 2, 3, 4, 5, 6]

// Works with named keys (PHP 8.1+)
$defaults = ['color' => 'blue', 'size' => 'M'];
$override = ['color' => 'red'];
$config = [...$defaults, ...$override];  // ['color' => 'red', 'size' => 'M']
```

---

## Strings

### String Functions (Most Common)

```php
<?php
strlen($str);                    // Length (bytes, not characters!)
mb_strlen($str);                 // Length (multibyte-safe — use this for UTF-8)
strtolower($str);                // Lowercase
strtoupper($str);                // Uppercase
trim($str);                      // Strip whitespace from both ends
ltrim($str); rtrim($str);        // Left/right trim

str_contains($hay, $needle);     // PHP 8.0+ (previously: strpos !== false)
str_starts_with($str, 'http');   // PHP 8.0+
str_ends_with($str, '.php');     // PHP 8.0+

substr($str, 0, 5);             // First 5 characters
str_replace('old', 'new', $str); // Replace all occurrences
str_pad($str, 10, '0', STR_PAD_LEFT);  // Left-pad with zeros

explode(',', $csv);              // Split string → array
implode(', ', $arr);             // Join array → string

sprintf('Hello, %s! You are %d.', $name, $age);  // Formatted string
```

### String Interpolation

```php
<?php
$name = "World";

// Simple variable
echo "Hello, $name!";

// Complex expressions need curly braces
echo "Item: {$items[0]->name}";
echo "Result: {$obj->getLabel()}";  // Method calls need braces

// Array access in strings
echo "Value: {$config['key']}";
```

### Regular Expressions (PCRE)

```php
<?php
// Match
preg_match('/^[a-z]+$/i', $input, $matches);

// Match all
preg_match_all('/\d+/', $text, $matches);

// Replace
$clean = preg_replace('/\s+/', ' ', $text);

// Replace with callback
$result = preg_replace_callback(
    '/\{(\w+)\}/',
    fn($m) => $replacements[$m[1]] ?? $m[0],
    $template
);

// Split
$words = preg_split('/[\s,;]+/', $text);
```

### Multibyte Strings (UTF-8)

PHP's standard string functions are byte-oriented. For UTF-8 text, always use `mb_*` functions:

```php
<?php
$emoji = "Hello 🌍";

strlen($emoji);      // 10 (bytes — emoji is 4 bytes in UTF-8)
mb_strlen($emoji);   // 7 (characters)

mb_strtolower('ÜBER');  // 'über' (handles non-ASCII)
mb_substr($str, 0, 5); // First 5 characters (not bytes)
```

---

## Key Takeaways

1. **PHP arrays are ordered hash maps** — they serve as list, dict, set, stack, and queue
2. **`unset()` doesn't reindex** — use `array_values()` if you need a sequential list after removal
3. **Use strict mode** in `in_array($val, $arr, true)` — loose comparison causes subtle bugs
4. **Array destructuring** with `['key' => $var]` makes code cleaner than repeated `$arr['key']`
5. **`array_column()`** is incredibly useful for extracting or re-indexing nested data
6. **Always use `mb_*` functions** for user-facing text — `strlen()` counts bytes, not characters
7. **`str_contains/starts_with/ends_with`** (PHP 8.0+) replace the old `strpos !== false` pattern
