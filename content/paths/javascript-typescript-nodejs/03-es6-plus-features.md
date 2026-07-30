---
title: "ES6+ Features"
weight: 3
---

## The Modern JavaScript Revolution

ES6 (ECMAScript 2015) was the biggest single update to JavaScript. Subsequent yearly releases (ES2016–ES2024) added incremental improvements. This file covers the features that fundamentally changed how JavaScript is written.

---

## Destructuring

Destructuring extracts values from objects and arrays into distinct variables using a pattern that mirrors the data structure.

### Object Destructuring

```javascript
const user = { name: "Alice", age: 30, city: "Barcelona", role: "admin" };

// Basic
const { name, age } = user;

// Rename
const { name: userName, age: userAge } = user;

// Default values
const { name, country = "Unknown" } = user; // country = "Unknown"

// Nested
const response = {
    data: {
        user: { id: 1, profile: { avatar: "url" } }
    }
};
const { data: { user: { profile: { avatar } } } } = response;

// Rest (collect remaining)
const { name, ...rest } = user; // rest = { age: 30, city: "Barcelona", role: "admin" }

// Computed property names
const key = "name";
const { [key]: value } = user; // value = "Alice"
```

### Array Destructuring

```javascript
const coords = [41.3851, 2.1734, 12];

// Basic
const [lat, lng] = coords;

// Skip elements
const [, , altitude] = coords;

// Default values
const [x, y, z, w = 0] = coords;

// Rest
const [first, ...remaining] = [1, 2, 3, 4, 5]; // remaining = [2, 3, 4, 5]

// Swap variables (no temp needed)
let a = 1, b = 2;
[a, b] = [b, a]; // a = 2, b = 1
```

### Function Parameter Destructuring

```javascript
// Object parameters with defaults
function createUser({ name, age, role = "user", active = true } = {}) {
    return { name, age, role, active, createdAt: new Date() };
}

createUser({ name: "Alice", age: 30 });
createUser(); // uses all defaults (the = {} handles undefined argument)

// Array parameters
function head([first, ...rest]) {
    return { first, rest };
}

head([1, 2, 3]); // { first: 1, rest: [2, 3] }
```

---

## Spread and Rest Operators

The `...` syntax serves two purposes depending on context:

- **Spread** — expands an iterable into individual elements
- **Rest** — collects multiple elements into a single array/object

### Spread

```javascript
// Array spread
const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];
const merged = [...arr1, ...arr2];        // [1, 2, 3, 4, 5, 6]
const copy = [...arr1];                    // shallow copy
const withInsert = [0, ...arr1, 4];       // [0, 1, 2, 3, 4]

// Object spread (shallow merge)
const defaults = { theme: "dark", lang: "en", fontSize: 14 };
const userPrefs = { theme: "light", fontSize: 16 };
const config = { ...defaults, ...userPrefs };
// { theme: "light", lang: "en", fontSize: 16 } — later spreads win

// Function arguments
const numbers = [5, 2, 8, 1, 9];
Math.max(...numbers); // 9 (equivalent to Math.max(5, 2, 8, 1, 9))
```

### Rest

```javascript
// Function parameters
function sum(...numbers) {
    return numbers.reduce((acc, n) => acc + n, 0);
}
sum(1, 2, 3, 4); // 10

// With leading params
function log(level, ...messages) {
    console.log(`[${level}]`, ...messages);
}
log("ERROR", "Connection failed", "retrying in 5s");
```

### Immutable Update Patterns

```javascript
// Add to array (immutable)
const addItem = (arr, item) => [...arr, item];

// Remove from array (immutable)
const removeAt = (arr, index) => [...arr.slice(0, index), ...arr.slice(index + 1)];

// Update object property (immutable)
const updateUser = (user, updates) => ({ ...user, ...updates });

// Deep update (nested)
const updateAddress = (user, city) => ({
    ...user,
    address: { ...user.address, city }
});
```

---

## Template Literals

```javascript
// Basic interpolation
const name = "Alice";
const greeting = `Hello, ${name}!`; // "Hello, Alice!"

// Expressions
const price = 19.99;
const tax = 0.21;
const total = `Total: €${(price * (1 + tax)).toFixed(2)}`; // "Total: €24.19"

// Multi-line (preserves whitespace)
const html = `
<div class="card">
    <h2>${title}</h2>
    <p>${description}</p>
</div>
`;

// Tagged templates (custom processing)
function sql(strings, ...values) {
    // strings: ["SELECT * FROM users WHERE id = ", " AND active = ", ""]
    // values: [userId, true]
    return {
        text: strings.join("$" + strings.map((_, i) => i + 1).join("$")),
        values: values
    };
}

const query = sql`SELECT * FROM users WHERE id = ${userId} AND active = ${true}`;
// Parameterized query — safe from SQL injection

// Highlight syntax (common in CSS-in-JS)
const styled = css`
    color: ${theme.primary};
    font-size: ${fontSize}px;
`;
```

---

## Iterators and Generators

### The Iterator Protocol

Any object with a `[Symbol.iterator]()` method that returns `{ next() → { value, done } }` is iterable.

```javascript
// Custom iterable
class Range {
    constructor(start, end, step = 1) {
        this.start = start;
        this.end = end;
        this.step = step;
    }
    
    [Symbol.iterator]() {
        let current = this.start;
        const { end, step } = this;
        
        return {
            next() {
                if (current <= end) {
                    const value = current;
                    current += step;
                    return { value, done: false };
                }
                return { done: true };
            }
        };
    }
}

const range = new Range(1, 10, 2);
for (const n of range) console.log(n); // 1, 3, 5, 7, 9
[...range]; // [1, 3, 5, 7, 9] — spread works on iterables
```

### Generators

Generators are functions that can pause and resume, yielding multiple values lazily.

```javascript
function* fibonacci() {
    let [a, b] = [0, 1];
    while (true) {
        yield a;
        [a, b] = [b, a + b];
    }
}

// Lazy — only computes on demand
const fib = fibonacci();
fib.next(); // { value: 0, done: false }
fib.next(); // { value: 1, done: false }
fib.next(); // { value: 1, done: false }
fib.next(); // { value: 2, done: false }

// Take first N values
function* take(iterable, n) {
    let count = 0;
    for (const item of iterable) {
        if (count >= n) return;
        yield item;
        count++;
    }
}

[...take(fibonacci(), 8)]; // [0, 1, 1, 2, 3, 5, 8, 13]

// Generator delegation
function* concat(...iterables) {
    for (const iter of iterables) {
        yield* iter; // delegates to another iterable
    }
}

[...concat([1, 2], [3, 4], [5])]; // [1, 2, 3, 4, 5]
```

### Async Generators

```javascript
async function* fetchPages(url) {
    let nextUrl = url;
    while (nextUrl) {
        const response = await fetch(nextUrl);
        const data = await response.json();
        yield data.items;
        nextUrl = data.nextPage;
    }
}

// Consume with for-await-of
for await (const page of fetchPages("/api/users?page=1")) {
    for (const user of page) {
        processUser(user);
    }
}
```

---

## Symbols

Symbols are unique, immutable identifiers. Their primary use is as non-colliding property keys.

```javascript
// Creating symbols
const id = Symbol("id");        // description is for debugging only
const id2 = Symbol("id");
id === id2;                     // false — every Symbol is unique

// As property keys (won't collide with string keys)
const user = {
    name: "Alice",
    [id]: 12345  // symbol-keyed property
};

user[id];           // 12345
Object.keys(user);  // ["name"] — symbols are not enumerable

// Well-known symbols (customize language behavior)
class MyArray {
    [Symbol.iterator]() { /* ... */ }        // make iterable
    [Symbol.toPrimitive](hint) { /* ... */ } // customize coercion
    get [Symbol.toStringTag]() { return "MyArray"; } // customize toString
}

// Global symbol registry (shared across realms)
const shared = Symbol.for("app.config"); // creates or retrieves
Symbol.for("app.config") === shared;     // true (same reference)
```

---

## Proxy and Reflect

Proxies intercept fundamental operations on objects (get, set, delete, etc.).

```javascript
const handler = {
    get(target, prop, receiver) {
        console.log(`Accessing ${String(prop)}`);
        return Reflect.get(target, prop, receiver);
    },
    set(target, prop, value, receiver) {
        if (prop === "age" && (value < 0 || value > 150)) {
            throw new RangeError("Invalid age");
        }
        return Reflect.set(target, prop, value, receiver);
    }
};

const user = new Proxy({ name: "Alice", age: 30 }, handler);
user.name;      // logs "Accessing name", returns "Alice"
user.age = 200; // throws RangeError

// Practical: reactive data (Vue 3 uses this internally)
function reactive(obj) {
    return new Proxy(obj, {
        set(target, prop, value) {
            const oldValue = target[prop];
            target[prop] = value;
            if (oldValue !== value) {
                notify(prop, value); // trigger UI update
            }
            return true;
        }
    });
}
```

---

## WeakRef and FinalizationRegistry

```javascript
// WeakRef — holds reference without preventing garbage collection
let obj = { data: "important" };
const weakRef = new WeakRef(obj);

weakRef.deref();  // { data: "important" }
obj = null;       // original reference gone
// After GC: weakRef.deref() → undefined

// FinalizationRegistry — callback when object is GC'd
const registry = new FinalizationRegistry((heldValue) => {
    console.log(`Object with key ${heldValue} was garbage collected`);
});

let cache = { key: "value" };
registry.register(cache, "cache-key");
cache = null; // eventually logs the message
```

---

## Key Takeaways

1. **Destructuring** eliminates verbose property access — use it in function parameters for self-documenting APIs
2. **Spread creates shallow copies** — nested objects are still shared references
3. **Generators enable lazy evaluation** — process infinite sequences without memory issues
4. **Symbols guarantee uniqueness** — use for metadata keys that won't collide with user data
5. **Proxies intercept operations** — foundation for reactivity systems, validation, and logging
6. **Tagged templates** — powerful for DSLs (SQL, CSS, GraphQL) with built-in safety
7. **`for...of` works on any iterable** — arrays, strings, Maps, Sets, generators, custom iterables
