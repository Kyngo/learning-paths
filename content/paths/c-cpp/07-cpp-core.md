---
title: "C++ Core"
weight: 7
---

# C++ Core

C++ extends C with classes, references, namespaces, and RAII (Resource Acquisition Is Initialization). This section covers the foundational C++ features that replace C's manual patterns with safer, more expressive abstractions.

---

## Classes

```cpp
class User {
public:
    // Constructor
    User(std::string name, int age)
        : name_(std::move(name)), age_(age) {}

    // Destructor (called automatically when object is destroyed)
    ~User() = default;

    // Getter
    const std::string& name() const { return name_; }
    int age() const { return age_; }

    // Method
    std::string greeting() const {
        return "Hello, " + name_ + "!";
    }

private:
    std::string name_;
    int age_;
};

User alice("Alice", 30);
std::cout << alice.greeting() << std::endl;
```

### Constructor Initialiser Lists

Always use initialiser lists — they initialise members directly instead of default-constructing then assigning:

```cpp
// GOOD — direct initialisation
User(std::string name, int age) : name_(std::move(name)), age_(age) {}

// BAD — default construct then assign (wastes work for non-trivial types)
User(std::string name, int age) {
    name_ = name;
    age_ = age;
}
```

### Special Member Functions

| Function | Signature | When Called |
|----------|-----------|------------|
| Default constructor | `User()` | `User u;` |
| Parameterised constructor | `User(string, int)` | `User u("Alice", 30);` |
| Copy constructor | `User(const User&)` | `User b = a;` |
| Move constructor | `User(User&&)` | `User b = std::move(a);` |
| Copy assignment | `User& operator=(const User&)` | `b = a;` |
| Move assignment | `User& operator=(User&&)` | `b = std::move(a);` |
| Destructor | `~User()` | Object goes out of scope |

### The Rule of Zero / Three / Five

| Rule | When | Define |
|------|------|--------|
| **Rule of Zero** | Class manages no resources | Nothing — use defaults |
| **Rule of Three** | Class manages a resource (raw pointer) | Copy ctor, copy assign, destructor |
| **Rule of Five** | Rule of Three + move semantics | + move ctor, move assign |

**Prefer the Rule of Zero.** Use smart pointers and standard containers — they handle their own resources.

---

## RAII (Resource Acquisition Is Initialization)

The most important C++ pattern: tie resource lifetime to object lifetime.

```cpp
void process_file(const std::string& path) {
    std::ifstream file(path);  // resource acquired (file opened)
    if (!file.is_open()) {
        throw std::runtime_error("cannot open " + path);
    }

    std::string line;
    while (std::getline(file, line)) {
        // process line
    }
}  // file automatically closed here (destructor called)
// No explicit close(), no cleanup code, no leaks — even if an exception is thrown
```

### RAII for Any Resource

```cpp
class MutexGuard {
public:
    explicit MutexGuard(std::mutex& m) : mutex_(m) { mutex_.lock(); }
    ~MutexGuard() { mutex_.unlock(); }

    // Non-copyable
    MutexGuard(const MutexGuard&) = delete;
    MutexGuard& operator=(const MutexGuard&) = delete;

private:
    std::mutex& mutex_;
};

// Usage
{
    MutexGuard guard(my_mutex);  // locked
    // ... critical section ...
}  // unlocked automatically

// Standard library equivalent: std::lock_guard, std::unique_lock
```

---

## References

References are aliases — they refer to an existing object without copying it:

```cpp
int x = 42;
int& ref = x;    // ref IS x (not a copy)
ref = 100;
std::cout << x;   // 100

// const reference — read-only alias
const int& cref = x;
// cref = 50;  // error: cannot modify through const ref
```

### References vs Pointers

| | Reference | Pointer |
|-|-----------|---------|
| Nullable | No — must refer to an object | Yes — can be `nullptr` |
| Reassignable | No — bound once | Yes |
| Syntax | `ref.member` | `ptr->member` |
| Arithmetic | No | Yes |
| Typical use | Function params, return values | Optional values, dynamic allocation |

### Pass by Reference

```cpp
// Pass by value — copies the vector (expensive!)
void process(std::vector<int> data) { ... }

// Pass by const reference — no copy, read-only (preferred for read access)
void process(const std::vector<int>& data) { ... }

// Pass by reference — no copy, can modify
void populate(std::vector<int>& data) { data.push_back(42); }
```

---

## Namespaces

```cpp
namespace myapp {
namespace http {

class Server {
public:
    void start(int port);
};

}  // namespace http
}  // namespace myapp

myapp::http::Server server;

// Using declaration
using myapp::http::Server;
Server server;

// NEVER use `using namespace std;` in headers — it pollutes every file that includes them
```

---

## `const` Correctness

```cpp
// Const method — does not modify the object
class User {
public:
    const std::string& name() const { return name_; }  // callable on const User
    void set_name(const std::string& n) { name_ = n; } // NOT callable on const User
};

const User alice("Alice", 30);
alice.name();       // OK — const method
// alice.set_name("Bob");  // error: non-const method on const object
```

**Mark everything `const` unless it needs to be mutable.** This is documentation, compiler-enforced correctness, and optimisation opportunity in one.

---

## Strings (`std::string`)

```cpp
#include <string>

std::string s = "Hello";
s += ", World!";           // concatenation
s.size();                  // 13
s.empty();                 // false
s.substr(0, 5);            // "Hello"
s.find("World");           // 7 (position)
s.find("xyz");             // std::string::npos (not found)
s.c_str();                 // const char* (for C APIs)

// Comparison
s == "Hello, World!"       // true
s < "Zello"                // true (lexicographic)

// String view (C++17) — non-owning reference to string data
std::string_view sv = s;   // no allocation
```

---

## I/O Streams

```cpp
#include <iostream>
#include <fstream>
#include <sstream>

// Console
std::cout << "Name: " << name << ", Age: " << age << std::endl;
std::cerr << "Error: " << message << std::endl;

std::string input;
std::getline(std::cin, input);

// File
std::ofstream out("output.txt");
out << "Hello, file!" << std::endl;

std::ifstream in("input.txt");
std::string line;
while (std::getline(in, line)) {
    std::cout << line << std::endl;
}

// String stream (build strings)
std::ostringstream oss;
oss << "User " << id << ": " << name;
std::string result = oss.str();
```

---

## Key Takeaways

- RAII is the foundation of C++ resource management. Acquire in constructors, release in destructors — no leaks, even with exceptions.
- Prefer the Rule of Zero. Use `std::string`, `std::vector`, and smart pointers — they handle their resources.
- References are non-nullable, non-reassignable aliases. Pass by `const&` for read access, by `&` for modification.
- Mark everything `const` unless mutation is needed. `const` methods are callable on `const` objects.
- `std::string` replaces `char*`. `std::string_view` provides a lightweight, non-owning view.
- Namespaces prevent name collisions. Never use `using namespace std;` in headers.
