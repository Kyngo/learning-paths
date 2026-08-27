---
title: "Templates & STL"
weight: 9
---

# Templates & STL

Templates are C++'s generic programming mechanism — compile-time code generation that produces type-specific implementations with zero runtime overhead. The Standard Template Library (STL) is built entirely on templates and provides containers, algorithms, and iterators.

---

## Function Templates

```cpp
template<typename T>
T max_of(T a, T b) {
    return (a > b) ? a : b;
}

max_of(3, 5);           // int
max_of(3.14, 2.71);     // double
max_of<std::string>("alice", "bob");  // explicit type
```

The compiler generates a separate function for each type used — this is **monomorphisation**. Zero overhead at runtime; binary size may increase.

---

## Class Templates

```cpp
template<typename T>
class Stack {
public:
    void push(const T& item) { data_.push_back(item); }

    T pop() {
        T top = data_.back();
        data_.pop_back();
        return top;
    }

    bool empty() const { return data_.empty(); }
    size_t size() const { return data_.size(); }

private:
    std::vector<T> data_;
};

Stack<int> int_stack;
Stack<std::string> str_stack;
```

---

## STL Containers

### Sequence Containers

| Container | Description | Access | Insert/Remove |
|-----------|-------------|--------|--------------|
| `std::vector<T>` | Dynamic array | O(1) random | O(1) back, O(n) middle |
| `std::deque<T>` | Double-ended queue | O(1) random | O(1) front/back |
| `std::list<T>` | Doubly-linked list | O(n) | O(1) at iterator position |
| `std::array<T, N>` | Fixed-size array | O(1) random | N/A (fixed size) |

### Associative Containers (Sorted, Tree-Based)

| Container | Description | Lookup | Insert |
|-----------|-------------|--------|--------|
| `std::set<T>` | Unique sorted values | O(log n) | O(log n) |
| `std::map<K, V>` | Sorted key-value pairs | O(log n) | O(log n) |
| `std::multiset<T>` | Sorted, allows duplicates | O(log n) | O(log n) |
| `std::multimap<K, V>` | Sorted, duplicate keys | O(log n) | O(log n) |

### Unordered Containers (Hash-Based)

| Container | Description | Lookup | Insert |
|-----------|-------------|--------|--------|
| `std::unordered_set<T>` | Unique values, hashed | O(1) avg | O(1) avg |
| `std::unordered_map<K, V>` | Key-value, hashed | O(1) avg | O(1) avg |

### Container Adaptors

| Adaptor | Underlying | Interface |
|---------|-----------|-----------|
| `std::stack<T>` | deque | push, pop, top |
| `std::queue<T>` | deque | push, pop, front |
| `std::priority_queue<T>` | vector | push, pop, top (max-heap) |

---

## std::vector (The Default Container)

```cpp
#include <vector>

std::vector<int> v = {1, 2, 3, 4, 5};

v.push_back(6);
v.pop_back();
v.size();           // 5
v.empty();          // false
v.capacity();       // >= 5
v.reserve(100);     // pre-allocate

v[0];               // 1 (no bounds check)
v.at(0);            // 1 (bounds check — throws std::out_of_range)

v.front();          // first element
v.back();           // last element

// Range-based for loop (C++11)
for (const auto& item : v) {
    std::cout << item << " ";
}

// Erase-remove idiom
v.erase(std::remove(v.begin(), v.end(), 3), v.end());
```

---

## std::map and std::unordered_map

```cpp
#include <map>
#include <unordered_map>

std::unordered_map<std::string, int> ages;
ages["Alice"] = 30;
ages["Bob"] = 25;
ages.insert({"Carol", 28});

// Check existence
if (ages.count("Alice")) { /* exists */ }
if (auto it = ages.find("Alice"); it != ages.end()) {
    std::cout << it->second << std::endl;  // 30
}

// Iterate
for (const auto& [name, age] : ages) {  // structured bindings (C++17)
    std::cout << name << ": " << age << std::endl;
}

// Default value on access (CAUTION: inserts if missing)
int age = ages["Unknown"];  // inserts "Unknown" → 0
```

---

## Iterators

Iterators are the glue between containers and algorithms:

```cpp
std::vector<int> v = {5, 3, 1, 4, 2};

// Iterator types
auto begin = v.begin();     // points to first element
auto end = v.end();         // points past last element
auto rbegin = v.rbegin();   // reverse: points to last element

// Manual iteration
for (auto it = v.begin(); it != v.end(); ++it) {
    std::cout << *it << " ";
}
```

---

## STL Algorithms (`<algorithm>`)

Algorithms operate on iterator ranges, not containers directly:

```cpp
#include <algorithm>
#include <numeric>

std::vector<int> v = {5, 3, 1, 4, 2};

// Sort
std::sort(v.begin(), v.end());              // ascending
std::sort(v.begin(), v.end(), std::greater<>());  // descending

// Search
auto it = std::find(v.begin(), v.end(), 3);
bool found = std::binary_search(v.begin(), v.end(), 3);  // requires sorted

// Transform
std::vector<int> doubled(v.size());
std::transform(v.begin(), v.end(), doubled.begin(), [](int x) { return x * 2; });

// Accumulate
int sum = std::accumulate(v.begin(), v.end(), 0);

// Min/Max
auto [min_it, max_it] = std::minmax_element(v.begin(), v.end());

// Count
int count = std::count_if(v.begin(), v.end(), [](int x) { return x > 3; });

// Remove (erase-remove idiom)
v.erase(std::remove_if(v.begin(), v.end(), [](int x) { return x < 3; }), v.end());

// Any/All/None
bool any = std::any_of(v.begin(), v.end(), [](int x) { return x > 10; });
bool all = std::all_of(v.begin(), v.end(), [](int x) { return x > 0; });
```

---

## Template Specialisation

```cpp
// General template
template<typename T>
std::string to_string(const T& value) {
    std::ostringstream oss;
    oss << value;
    return oss.str();
}

// Specialisation for bool
template<>
std::string to_string<bool>(const bool& value) {
    return value ? "true" : "false";
}
```

---

## Key Takeaways

- `std::vector` is the default container. Use it unless you have a specific reason not to.
- `std::unordered_map` for O(1) lookup; `std::map` for sorted iteration.
- STL algorithms (`sort`, `find`, `transform`, `accumulate`) work on iterator ranges. Prefer them over hand-written loops.
- Templates are compile-time code generation — zero runtime overhead but potentially larger binaries.
- Use structured bindings (`auto& [key, value]`) for cleaner iteration over maps (C++17).
- The erase-remove idiom is the standard way to remove elements from a vector.
