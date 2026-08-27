---
title: "Collections & Iterators"
weight: 6
---

# Collections & Iterators

Rust's collections are heap-allocated, growable data structures. Its iterator system is the primary way to process collections — zero-cost abstractions that compile to code as fast as hand-written loops.

---

## Vec<T> — Dynamic Array

```rust
// Create
let mut v: Vec<i32> = Vec::new();
let v = vec![1, 2, 3, 4, 5];       // macro shorthand
let v = vec![0; 10];                // 10 zeros
let v = Vec::with_capacity(100);    // pre-allocate

// Access
let first = v[0];                   // panics if out of bounds
let first = v.get(0);               // returns Option<&T>
let last = v.last();                // Option<&T>

// Modify
v.push(6);
v.pop();                            // returns Option<T>
v.insert(0, 10);                    // insert at index
v.remove(2);                        // remove at index
v.retain(|x| x % 2 == 0);          // keep only evens
v.sort();
v.dedup();                          // remove consecutive duplicates

// Iteration
for item in &v { ... }              // borrow
for item in &mut v { ... }          // mutable borrow
for item in v { ... }               // consume (v is moved)

// Slicing
let slice: &[i32] = &v[1..4];
```

### Vec Internals

A `Vec<T>` is three values on the stack: pointer, length, capacity.

```
Stack:                    Heap:
[ptr, len=3, cap=8] ──→ [1, 2, 3, _, _, _, _, _]
```

When length exceeds capacity, Vec allocates a new buffer (typically 2× the old capacity) and copies elements. This makes `push` amortised O(1).

---

## HashMap<K, V>

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();
scores.insert("Alice", 100);
scores.insert("Bob", 85);

// Access
let score = scores.get("Alice");           // Option<&V>
let score = scores["Alice"];                // panics if missing

// Entry API — insert only if absent
scores.entry("Carol").or_insert(90);
scores.entry("Alice").and_modify(|s| *s += 10);

// Iteration
for (name, score) in &scores {
    println!("{name}: {score}");
}

// From iterator of tuples
let scores: HashMap<_, _> = vec![("Alice", 100), ("Bob", 85)]
    .into_iter()
    .collect();
```

### Other Collections

| Collection | Use |
|-----------|-----|
| `Vec<T>` | Ordered, indexed sequence |
| `VecDeque<T>` | Double-ended queue |
| `LinkedList<T>` | Doubly-linked list (rarely used) |
| `HashMap<K, V>` | Key-value lookup (unordered) |
| `BTreeMap<K, V>` | Key-value lookup (sorted by key) |
| `HashSet<T>` | Unique values (unordered) |
| `BTreeSet<T>` | Unique values (sorted) |
| `BinaryHeap<T>` | Priority queue (max-heap) |

---

## Iterators

Iterators are Rust's primary abstraction for processing sequences. They are **lazy** — nothing happens until you consume them.

### The Iterator Trait

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

### Creating Iterators

```rust
let v = vec![1, 2, 3, 4, 5];

v.iter()        // yields &T (borrows)
v.iter_mut()    // yields &mut T (mutable borrows)
v.into_iter()   // yields T (consumes the vec)

// Ranges
(0..10)         // 0 to 9
(0..=10)        // 0 to 10
('a'..='z')     // a to z
```

### Iterator Adaptors (Lazy — Return a New Iterator)

```rust
let v = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

v.iter()
    .filter(|&&x| x % 2 == 0)     // keep evens
    .map(|&x| x * x)              // square them
    .take(3)                       // first 3 results
    .collect::<Vec<_>>()           // [4, 16, 36]
```

### Common Adaptors

| Adaptor | Description | Example |
|---------|-------------|---------|
| `map(f)` | Transform each element | `.map(\|x\| x * 2)` |
| `filter(f)` | Keep elements matching predicate | `.filter(\|x\| x > &0)` |
| `filter_map(f)` | Map + filter None | `.filter_map(\|s\| s.parse().ok())` |
| `flat_map(f)` | Map then flatten | `.flat_map(\|v\| v.iter())` |
| `flatten()` | Flatten nested iterators | `vec![vec![1], vec![2, 3]].into_iter().flatten()` |
| `enumerate()` | Add index | `.enumerate()` → `(0, item), (1, item)...` |
| `zip(other)` | Pair with another iterator | `.zip(other.iter())` |
| `chain(other)` | Concatenate iterators | `.chain(other.iter())` |
| `take(n)` | First n elements | `.take(5)` |
| `skip(n)` | Skip first n elements | `.skip(2)` |
| `peekable()` | Peek at next without consuming | `.peekable()` |
| `inspect(f)` | Debug: side-effect without modifying | `.inspect(\|x\| println!("{x}"))` |
| `cloned()` | Clone borrowed items | `.cloned()` — `&T → T` |

### Consumers (Eager — Produce a Result)

| Consumer | Description | Example |
|----------|-------------|---------|
| `collect()` | Build a collection | `.collect::<Vec<_>>()` |
| `for_each(f)` | Apply function to each | `.for_each(\|x\| println!("{x}"))` |
| `count()` | Count elements | `.count()` |
| `sum()` | Sum elements | `.sum::<i32>()` |
| `product()` | Multiply elements | `.product::<i32>()` |
| `min()` / `max()` | Minimum / maximum | `.max()` → `Option<T>` |
| `any(f)` | True if any match | `.any(\|x\| x > &10)` |
| `all(f)` | True if all match | `.all(\|x\| x > &0)` |
| `find(f)` | First matching element | `.find(\|x\| x > &&5)` |
| `position(f)` | Index of first match | `.position(\|x\| x == &5)` |
| `fold(init, f)` | Reduce with accumulator | `.fold(0, \|acc, x\| acc + x)` |
| `reduce(f)` | Fold without initial value | `.reduce(\|a, b\| a + b)` |

---

## Closures

Closures are anonymous functions that capture their environment:

```rust
let threshold = 10;
let above = |x: &i32| *x > threshold;  // captures threshold by reference

let big_numbers: Vec<_> = numbers.iter().filter(above).collect();
```

### Closure Capture

| Capture | How | Trait |
|---------|-----|-------|
| By reference | `&variable` | `Fn` |
| By mutable reference | `&mut variable` | `FnMut` |
| By value (move) | `variable` consumed | `FnOnce` |

```rust
let name = String::from("Alice");

// Move capture — closure takes ownership
let greeting = move || format!("Hello, {name}!");
// name is no longer available here
```

### Fn Trait Hierarchy

```
FnOnce  ⊃  FnMut  ⊃  Fn
```

- `FnOnce` — can be called once (may consume captured values)
- `FnMut` — can be called multiple times (may mutate captured values)
- `Fn` — can be called multiple times (read-only access to captures)

---

## Practical Examples

### Word Frequency Count

```rust
use std::collections::HashMap;

fn word_frequencies(text: &str) -> HashMap<&str, usize> {
    let mut freqs = HashMap::new();
    for word in text.split_whitespace() {
        *freqs.entry(word).or_insert(0) += 1;
    }
    freqs
}
```

### Top N Elements

```rust
fn top_n<T: Ord>(items: Vec<T>, n: usize) -> Vec<T> {
    let mut sorted = items;
    sorted.sort_unstable_by(|a, b| b.cmp(a));
    sorted.into_iter().take(n).collect()
}
```

### Iterator Pipeline

```rust
let total_revenue: f64 = orders.iter()
    .filter(|o| o.status == Status::Completed)
    .filter(|o| o.date >= start_date)
    .map(|o| o.total)
    .sum();
```

---

## Key Takeaways

- `Vec<T>` is the default collection. Use `HashMap` for key-value lookups, `HashSet` for uniqueness.
- Iterators are lazy — adaptors like `map` and `filter` build a chain that executes only when consumed.
- Iterator chains compile to code as fast as hand-written loops — zero-cost abstraction.
- `collect()` is the universal consumer — it can produce `Vec`, `HashMap`, `String`, `Result<Vec<_>, _>`, and more.
- Closures capture their environment automatically. Use `move` to take ownership of captured values.
- The entry API (`map.entry(key).or_insert(default)`) is the idiomatic way to handle missing keys.
