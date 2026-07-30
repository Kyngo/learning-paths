---
title: "Java: Object-Oriented Programming"
weight: 3
---

## Classes and Objects

### Anatomy of a Java Class

```java
public class Employee {
    // Fields (instance variables)
    private final String id;
    private String name;
    private double salary;
    private Department department;
    
    // Static field (class-level)
    private static int totalEmployees = 0;
    
    // Constructor
    public Employee(String id, String name, double salary) {
        this.id = id;
        this.name = name;
        this.salary = salary;
        totalEmployees++;
    }
    
    // Overloaded constructor (constructor chaining)
    public Employee(String id, String name) {
        this(id, name, 50000.0);  // Delegates to main constructor
    }
    
    // Getters and setters (encapsulation)
    public String getName() { return name; }
    
    public void setSalary(double salary) {
        if (salary < 0) throw new IllegalArgumentException("Salary cannot be negative");
        this.salary = salary;
    }
    
    // Business method
    public double annualBonus() {
        return salary * 0.10;
    }
    
    // Static method
    public static int getTotalEmployees() {
        return totalEmployees;
    }
    
    // Object methods override
    @Override
    public String toString() {
        return "Employee{id='%s', name='%s', salary=%.2f}".formatted(id, name, salary);
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee other)) return false;
        return id.equals(other.id);
    }
    
    @Override
    public int hashCode() {
        return id.hashCode();
    }
}
```

---

## Inheritance

```java
public abstract class Shape {
    private final String color;
    
    protected Shape(String color) {
        this.color = color;
    }
    
    // Abstract method — subclasses MUST implement
    public abstract double area();
    public abstract double perimeter();
    
    // Concrete method — inherited as-is
    public String describe() {
        return "%s %s (area=%.2f)".formatted(color, getClass().getSimpleName(), area());
    }
    
    // Final method — CANNOT be overridden
    public final String getColor() {
        return color;
    }
}

public class Circle extends Shape {
    private final double radius;
    
    public Circle(String color, double radius) {
        super(color);  // Must call parent constructor
        this.radius = radius;
    }
    
    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
    
    @Override
    public double perimeter() {
        return 2 * Math.PI * radius;
    }
}

public class Rectangle extends Shape {
    private final double width, height;
    
    public Rectangle(String color, double width, double height) {
        super(color);
        this.width = width;
        this.height = height;
    }
    
    @Override
    public double area() { return width * height; }
    
    @Override
    public double perimeter() { return 2 * (width + height); }
}
```

---

## Interfaces

```java
// Interfaces define contracts (what, not how)
public interface Sortable<T> {
    int compareTo(T other);
    
    // Default method (Java 8+) — provides implementation
    default boolean isGreaterThan(T other) {
        return compareTo(other) > 0;
    }
    
    // Static method
    static <T extends Sortable<T>> T max(T a, T b) {
        return a.compareTo(b) >= 0 ? a : b;
    }
}

public interface Serializable {
    byte[] serialize();
    
    // Private method (Java 9+) — helper for default methods
    private String formatHeader() {
        return "v1:" + getClass().getName();
    }
    
    default byte[] serializeWithHeader() {
        return (formatHeader() + ":" + new String(serialize())).getBytes();
    }
}

// A class can implement multiple interfaces
public class Product implements Sortable<Product>, Serializable {
    private final String name;
    private final double price;
    
    @Override
    public int compareTo(Product other) {
        return Double.compare(this.price, other.price);
    }
    
    @Override
    public byte[] serialize() {
        return (name + ":" + price).getBytes();
    }
}
```

### Interface vs Abstract Class

| Feature | Interface | Abstract Class |
|---------|-----------|---------------|
| Multiple inheritance | Yes (implements many) | No (extends one) |
| Fields | Constants only (`static final`) | Any fields |
| Constructors | No | Yes |
| Access modifiers | `public` (default methods) | Any |
| Use when | Defining a capability/contract | Sharing code among related classes |

---

## Polymorphism

```java
// Runtime polymorphism — method dispatch based on actual type
List<Shape> shapes = List.of(
    new Circle("red", 5),
    new Rectangle("blue", 3, 4),
    new Circle("green", 2)
);

for (Shape shape : shapes) {
    // Calls the correct area() based on actual runtime type
    System.out.println(shape.describe());
}

// Covariant return types
class Animal {
    public Animal create() { return new Animal(); }
}

class Dog extends Animal {
    @Override
    public Dog create() { return new Dog(); }  // More specific return type OK
}
```

---

## Composition and Delegation

```java
// Prefer composition over inheritance
public class OrderService {
    private final OrderRepository repository;
    private final PaymentGateway paymentGateway;
    private final NotificationService notifications;
    
    // Dependencies injected via constructor
    public OrderService(
            OrderRepository repository,
            PaymentGateway paymentGateway,
            NotificationService notifications) {
        this.repository = repository;
        this.paymentGateway = paymentGateway;
        this.notifications = notifications;
    }
    
    public Order placeOrder(OrderRequest request) {
        Order order = Order.from(request);
        
        PaymentResult payment = paymentGateway.charge(
            request.paymentMethod(), order.total()
        );
        
        if (!payment.isSuccessful()) {
            throw new PaymentFailedException(payment.errorMessage());
        }
        
        order = order.withStatus(OrderStatus.CONFIRMED);
        repository.save(order);
        notifications.sendOrderConfirmation(order);
        
        return order;
    }
}
```

---

## Enums — Type-Safe Constants

```java
public enum HttpStatus {
    OK(200, "OK"),
    CREATED(201, "Created"),
    BAD_REQUEST(400, "Bad Request"),
    UNAUTHORIZED(401, "Unauthorized"),
    NOT_FOUND(404, "Not Found"),
    INTERNAL_ERROR(500, "Internal Server Error");
    
    private final int code;
    private final String message;
    
    HttpStatus(int code, String message) {
        this.code = code;
        this.message = message;
    }
    
    public int code() { return code; }
    public String message() { return message; }
    
    public boolean isSuccess() { return code >= 200 && code < 300; }
    public boolean isClientError() { return code >= 400 && code < 500; }
    public boolean isServerError() { return code >= 500; }
    
    // Lookup by code
    public static HttpStatus fromCode(int code) {
        for (HttpStatus status : values()) {
            if (status.code == code) return status;
        }
        throw new IllegalArgumentException("Unknown HTTP status: " + code);
    }
}

// Enum with abstract method (strategy pattern)
public enum Operation {
    ADD {
        @Override public double apply(double a, double b) { return a + b; }
    },
    SUBTRACT {
        @Override public double apply(double a, double b) { return a - b; }
    },
    MULTIPLY {
        @Override public double apply(double a, double b) { return a * b; }
    };
    
    public abstract double apply(double a, double b);
}

double result = Operation.ADD.apply(3, 4);  // 7.0
```

---

## SOLID Principles in Java

### Single Responsibility

```java
// BAD: UserService does too much
class UserService {
    void createUser(User u) { /* ... */ }
    void sendEmail(User u, String msg) { /* ... */ }
    String generateReport(List<User> users) { /* ... */ }
}

// GOOD: Each class has one reason to change
class UserService { void createUser(User u) { /* ... */ } }
class EmailService { void send(String to, String msg) { /* ... */ } }
class UserReportGenerator { String generate(List<User> users) { /* ... */ } }
```

### Open/Closed (via interfaces)

```java
// Open for extension, closed for modification
interface DiscountStrategy {
    double apply(double price);
}

class PercentageDiscount implements DiscountStrategy {
    private final double percent;
    PercentageDiscount(double percent) { this.percent = percent; }
    public double apply(double price) { return price * (1 - percent / 100); }
}

class FixedDiscount implements DiscountStrategy {
    private final double amount;
    FixedDiscount(double amount) { this.amount = amount; }
    public double apply(double price) { return Math.max(0, price - amount); }
}

// Adding new discount types doesn't modify existing code
class BuyOneGetOneFree implements DiscountStrategy {
    public double apply(double price) { return price / 2; }
}
```

### Dependency Inversion

```java
// HIGH-level modules should not depend on LOW-level modules.
// Both should depend on abstractions.

// BAD: OrderService depends on concrete MySQLRepository
class OrderService {
    private MySQLOrderRepository repo = new MySQLOrderRepository();
}

// GOOD: Depend on abstraction
class OrderService {
    private final OrderRepository repo;  // Interface
    
    OrderService(OrderRepository repo) {  // Injected
        this.repo = repo;
    }
}
// Now works with MySQL, Postgres, InMemory, Mock...
```

---

## Hypothetical Use Case: Event-Driven Domain Model

```java
public sealed interface DomainEvent permits OrderCreated, OrderShipped, OrderCancelled {
    String orderId();
    Instant occurredAt();
}

public record OrderCreated(String orderId, String customerId, Instant occurredAt) 
    implements DomainEvent {}

public record OrderShipped(String orderId, String trackingNumber, Instant occurredAt) 
    implements DomainEvent {}

public record OrderCancelled(String orderId, String reason, Instant occurredAt) 
    implements DomainEvent {}

public class Order {
    private final String id;
    private OrderStatus status;
    private final List<DomainEvent> events = new ArrayList<>();
    
    public void ship(String trackingNumber) {
        if (status != OrderStatus.CONFIRMED) {
            throw new IllegalStateException("Cannot ship " + status + " order");
        }
        status = OrderStatus.SHIPPED;
        events.add(new OrderShipped(id, trackingNumber, Instant.now()));
    }
    
    public List<DomainEvent> drainEvents() {
        var drained = List.copyOf(events);
        events.clear();
        return drained;
    }
}
```

---

## Key Takeaways

1. **Encapsulation** — fields private, access via methods (getters/setters or business methods)
2. **Favor composition over inheritance** — inject dependencies, delegate behavior
3. **Interfaces define contracts** — use them for polymorphism and testability
4. **Enums are powerful** — they can have fields, methods, and implement interfaces
5. **Records** (Java 16+) eliminate data class boilerplate
6. **Sealed types** (Java 17+) enable exhaustive pattern matching
7. **SOLID principles** guide class design — especially Dependency Inversion for testability
