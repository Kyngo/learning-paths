---
title: "C++ Object-Oriented"
weight: 8
---

# C++ Object-Oriented Programming

C++ supports classical OOP — inheritance, polymorphism, encapsulation, and abstract classes. Modern C++ uses OOP selectively, favouring composition and generic programming for most designs.

---

## Inheritance

```cpp
class Shape {
public:
    Shape(double x, double y) : x_(x), y_(y) {}
    virtual ~Shape() = default;  // virtual destructor — essential for polymorphism

    double x() const { return x_; }
    double y() const { return y_; }

    virtual double area() const = 0;        // pure virtual — must be overridden
    virtual std::string name() const = 0;

    virtual void describe() const {
        std::cout << name() << " at (" << x_ << ", " << y_
                  << ") area=" << area() << std::endl;
    }

private:
    double x_, y_;
};

class Circle : public Shape {
public:
    Circle(double x, double y, double radius)
        : Shape(x, y), radius_(radius) {}

    double area() const override { return M_PI * radius_ * radius_; }
    std::string name() const override { return "Circle"; }

private:
    double radius_;
};

class Rectangle : public Shape {
public:
    Rectangle(double x, double y, double w, double h)
        : Shape(x, y), width_(w), height_(h) {}

    double area() const override { return width_ * height_; }
    std::string name() const override { return "Rectangle"; }

private:
    double width_, height_;
};
```

### Key Rules

| Rule | Why |
|------|-----|
| Always make base class destructor `virtual` | Prevents undefined behaviour when deleting through base pointer |
| Use `override` keyword | Catches signature mismatches at compile time |
| Use `= 0` for pure virtual (abstract) | Forces derived classes to implement |
| Prefer `public` inheritance | Models "is-a" relationship |

---

## Polymorphism

```cpp
void print_shapes(const std::vector<std::unique_ptr<Shape>>& shapes) {
    for (const auto& shape : shapes) {
        shape->describe();  // virtual dispatch — correct method called
    }
}

std::vector<std::unique_ptr<Shape>> shapes;
shapes.push_back(std::make_unique<Circle>(0, 0, 5));
shapes.push_back(std::make_unique<Rectangle>(1, 1, 3, 4));
print_shapes(shapes);
```

### Virtual Functions Internals

Each class with virtual functions has a **vtable** — a table of function pointers. Each object carries a hidden **vptr** pointing to its class's vtable. Virtual calls are indirect function calls through this table.

| | Static Binding | Dynamic Binding (Virtual) |
|-|---------------|--------------------------|
| Resolved | Compile time | Runtime |
| Overhead | None (direct call) | One pointer indirection |
| Use | Non-virtual methods | `virtual` methods |

---

## Operator Overloading

```cpp
class Vector2D {
public:
    double x, y;

    Vector2D(double x, double y) : x(x), y(y) {}

    Vector2D operator+(const Vector2D& other) const {
        return {x + other.x, y + other.y};
    }

    Vector2D operator*(double scalar) const {
        return {x * scalar, y * scalar};
    }

    bool operator==(const Vector2D& other) const {
        return x == other.x && y == other.y;
    }

    friend std::ostream& operator<<(std::ostream& os, const Vector2D& v) {
        return os << "(" << v.x << ", " << v.y << ")";
    }
};

Vector2D a(1, 2), b(3, 4);
Vector2D c = a + b;         // (4, 6)
Vector2D d = a * 3;         // (3, 6)
std::cout << c << std::endl; // (4, 6)
```

### Overloadable Operators

| Category | Operators |
|----------|----------|
| Arithmetic | `+`, `-`, `*`, `/`, `%`, `++`, `--` |
| Comparison | `==`, `!=`, `<`, `>`, `<=`, `>=`, `<=>` (C++20) |
| Assignment | `=`, `+=`, `-=`, `*=`, `/=` |
| Subscript | `[]` |
| Function call | `()` |
| Stream | `<<`, `>>` |
| Dereference | `*`, `->` |

---

## Abstract Classes and Interfaces

```cpp
// Abstract class (has at least one pure virtual function)
class Serializable {
public:
    virtual ~Serializable() = default;
    virtual std::string to_json() const = 0;
    virtual void from_json(const std::string& json) = 0;
};

// C++ interface pattern (all pure virtual, no data)
class Repository {
public:
    virtual ~Repository() = default;
    virtual std::optional<User> find(int id) = 0;
    virtual void save(const User& user) = 0;
    virtual void remove(int id) = 0;
};

// Concrete implementation
class PostgresRepository : public Repository {
public:
    std::optional<User> find(int id) override { /* SQL query */ }
    void save(const User& user) override { /* SQL insert/update */ }
    void remove(int id) override { /* SQL delete */ }
};
```

---

## Multiple Inheritance

C++ allows inheriting from multiple base classes:

```cpp
class Printable {
public:
    virtual void print() const = 0;
};

class Serializable {
public:
    virtual std::string serialize() const = 0;
};

class Document : public Printable, public Serializable {
public:
    void print() const override { std::cout << content_ << std::endl; }
    std::string serialize() const override { return content_; }
private:
    std::string content_;
};
```

### The Diamond Problem

```
    Animal
   /      \
  Dog     Pet
   \      /
   PetDog        ← which Animal's data does PetDog have?
```

Solution: **virtual inheritance**

```cpp
class Animal { /* ... */ };
class Dog : virtual public Animal { /* ... */ };
class Pet : virtual public Animal { /* ... */ };
class PetDog : public Dog, public Pet { /* ... */ };  // single Animal subobject
```

**In practice:** Avoid deep hierarchies and diamond inheritance. Prefer composition.

---

## Composition over Inheritance

```cpp
// PREFER THIS — composition
class Logger { /* ... */ };
class Database { /* ... */ };

class UserService {
public:
    UserService(Logger& log, Database& db) : log_(log), db_(db) {}
    // ...
private:
    Logger& log_;
    Database& db_;
};

// AVOID THIS — inheritance for code reuse
class UserService : public Logger, public Database { /* ... */ };
```

Use inheritance for **is-a** relationships and polymorphism. Use composition for **has-a** relationships and code reuse.

---

## Key Takeaways

- Always declare base class destructors as `virtual`. Always use `override` on derived methods.
- Virtual functions enable runtime polymorphism via vtable dispatch — small cost, huge flexibility.
- Pure virtual functions (`= 0`) create abstract classes. All-pure-virtual classes serve as interfaces.
- Operator overloading makes custom types feel native. Overload `==`, `<`, and `<<` for standard library compatibility.
- Avoid deep inheritance hierarchies and the diamond problem. Prefer composition.
- Use `std::unique_ptr<Base>` for polymorphic ownership — no manual `delete`, no leaks.
