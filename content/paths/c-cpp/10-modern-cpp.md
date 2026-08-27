---
title: "Modern C++"
weight: 10
---

# Modern C++

C++11 through C++23 transformed the language. Smart pointers, move semantics, lambdas, `auto`, and `std::optional` make modern C++ drastically safer and more expressive than pre-2011 code.

---

## Smart Pointers

Replace raw `new`/`delete` with RAII-managed pointers:

### `std::unique_ptr` — Exclusive Ownership

```cpp
#include <memory>

auto user = std::make_unique<User>("Alice", 30);  // allocates on heap
user->greet();  // use like a raw pointer

// Automatically deleted when unique_ptr goes out of scope
// Cannot be copied — only moved
auto user2 = std::move(user);  // user is now nullptr
```

### `std::shared_ptr` — Shared Ownership

```cpp
auto data = std::make_shared<std::vector<int>>(1000);
auto copy = data;  // reference count = 2

data.use_count();  // 2
// Deleted when last shared_ptr is destroyed
```

### `std::weak_ptr` — Non-Owning Observer

```cpp
std::weak_ptr<User> observer = shared_user;
if (auto locked = observer.lock()) {
    // locked is a shared_ptr — object still alive
    locked->greet();
}
// If the object was deleted, lock() returns nullptr
```

### When to Use What

| Type | Use |
|------|-----|
| `unique_ptr` | Default. Single owner, deterministic destruction |
| `shared_ptr` | Multiple owners needed (rare — think carefully) |
| `weak_ptr` | Break reference cycles, caches, observers |
| Raw pointer | Non-owning reference (never `delete` a raw pointer) |
| Raw `new`/`delete` | Almost never. Only in allocator implementations |

---

## Move Semantics

Move semantics transfer resources instead of copying them:

```cpp
std::vector<int> create_data() {
    std::vector<int> v(1000000);
    // ... fill v ...
    return v;  // MOVED, not copied (compiler optimises this — NRVO)
}

std::string a = "hello";
std::string b = std::move(a);  // a is now in a "valid but unspecified" state
// a might be empty, or might contain something — don't use it
```

### Move vs Copy

| | Copy | Move |
|-|------|------|
| Operation | Duplicate all data | Transfer ownership of data |
| Performance | O(n) for containers | O(1) for containers |
| Source after | Unchanged | Valid but unspecified |

### Rvalue References (`&&`)

```cpp
void process(const std::string& s);   // accepts lvalues (named variables)
void process(std::string&& s);         // accepts rvalues (temporaries, std::move'd)
```

---

## Lambdas

Anonymous functions with capture:

```cpp
// Basic lambda
auto add = [](int a, int b) { return a + b; };
add(3, 4);  // 7

// Capture by reference
int threshold = 10;
auto above = [&threshold](int x) { return x > threshold; };

// Capture by value
auto snapshot = [threshold](int x) { return x > threshold; };

// Capture all by reference
auto f = [&](int x) { return x + threshold; };

// Capture all by value
auto g = [=](int x) { return x + threshold; };

// Generic lambda (C++14)
auto print = [](const auto& x) { std::cout << x << std::endl; };
```

### Lambdas with STL

```cpp
std::vector<int> v = {5, 3, 1, 4, 2};

std::sort(v.begin(), v.end(), [](int a, int b) { return a > b; });
// v = {5, 4, 3, 2, 1}

auto it = std::find_if(v.begin(), v.end(), [](int x) { return x == 3; });

int sum = std::accumulate(v.begin(), v.end(), 0,
    [](int acc, int x) { return acc + x; });
```

---

## `auto` and Type Deduction

```cpp
auto x = 42;                          // int
auto pi = 3.14;                       // double
auto name = std::string("Alice");     // std::string
auto v = std::vector<int>{1, 2, 3};   // std::vector<int>

// With references
const auto& ref = v;                  // const std::vector<int>&

// Return type deduction (C++14)
auto add(int a, int b) { return a + b; }

// Structured bindings (C++17)
std::map<std::string, int> ages = {{"Alice", 30}, {"Bob", 25}};
for (const auto& [name, age] : ages) {
    std::cout << name << ": " << age << std::endl;
}
```

---

## `std::optional` (C++17)

A value that may or may not be present — replaces `nullptr` and sentinel values:

```cpp
#include <optional>

std::optional<User> find_user(int id) {
    if (/* found */) return User{...};
    return std::nullopt;
}

auto user = find_user(42);
if (user.has_value()) {
    std::cout << user->name() << std::endl;
}

// Or with value_or
std::string name = find_user(42)
    .transform([](const User& u) { return u.name(); })
    .value_or("unknown");
```

---

## `std::variant` (C++17) — Type-Safe Union

```cpp
#include <variant>

std::variant<int, double, std::string> value;
value = 42;
value = "hello";

// Visit pattern
std::visit([](const auto& v) {
    std::cout << v << std::endl;
}, value);

// Type check
if (std::holds_alternative<int>(value)) {
    int n = std::get<int>(value);
}
```

---

## `constexpr` — Compile-Time Computation

```cpp
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}

constexpr int result = factorial(10);  // computed at compile time
static_assert(factorial(5) == 120);    // compile-time assertion

// constexpr if (C++17) — compile-time branch
template<typename T>
std::string to_string(T value) {
    if constexpr (std::is_same_v<T, bool>) {
        return value ? "true" : "false";
    } else {
        return std::to_string(value);
    }
}
```

---

## Key Takeaways

- `std::unique_ptr` is the default for heap allocation. `shared_ptr` only when multiple owners are genuinely needed.
- Move semantics make returning large objects by value efficient — the compiler transfers, not copies.
- Lambdas replace function pointers and functors. They are the standard way to pass behaviour to algorithms.
- `auto` reduces verbosity and avoids type mismatch bugs. Use it liberally but keep function signatures explicit.
- `std::optional` replaces null pointers for optional values. `std::variant` replaces unions with type safety.
- `constexpr` moves computation to compile time. Use it for constants, lookup tables, and config validation.
