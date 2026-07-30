---
title: "Modern PHP Patterns and Architecture"
weight: 11
---

## Dependency Injection

The most important pattern in modern PHP. Instead of creating dependencies internally, inject them from outside:

```php
<?php
// ❌ Tight coupling — hard to test, can't swap implementations
class OrderService
{
    public function process(Order $order): void
    {
        $mailer = new SmtpMailer();  // hardcoded dependency
        $mailer->send($order->getEmail(), 'Order confirmed');
    }
}

// ✅ Dependency injection — testable, flexible
class OrderService
{
    public function __construct(
        private readonly MailerInterface $mailer,
        private readonly OrderRepository $repository,
    ) {}

    public function process(Order $order): void
    {
        $this->repository->save($order);
        $this->mailer->send($order->getEmail(), 'Order confirmed');
    }
}
```

### DI Containers

A container autowires dependencies based on type declarations:

```php
<?php
// Symfony DI Container (simplified)
$container = new ContainerBuilder();
$container->autowire(OrderService::class)
    ->setPublic(true);
$container->autowire(SmtpMailer::class)
    ->setAlias(MailerInterface::class);

$orderService = $container->get(OrderService::class);
// Container automatically injects MailerInterface and OrderRepository
```

---

## Value Objects

Immutable objects representing a concept with no identity (two VOs with same values are equal):

```php
<?php
readonly class Money
{
    public function __construct(
        public int $amount,       // Store in cents to avoid float issues
        public string $currency,
    ) {
        if ($amount < 0) {
            throw new \InvalidArgumentException('Amount cannot be negative');
        }
    }

    public function add(self $other): self
    {
        if ($this->currency !== $other->currency) {
            throw new \DomainException('Cannot add different currencies');
        }
        return new self($this->amount + $other->amount, $this->currency);
    }

    public function multiply(int $factor): self
    {
        return new self($this->amount * $factor, $this->currency);
    }

    public function format(): string
    {
        return number_format($this->amount / 100, 2) . ' ' . $this->currency;
    }
}

$price = new Money(1999, 'EUR');     // €19.99
$total = $price->multiply(3);        // €59.97
```

---

## DTOs (Data Transfer Objects)

Simple structures for passing data between layers:

```php
<?php
readonly class CreateUserRequest
{
    public function __construct(
        public string $name,
        public string $email,
        public ?string $phone = null,
    ) {}

    public static function fromArray(array $data): self
    {
        return new self(
            name: $data['name'] ?? throw new \InvalidArgumentException('Name required'),
            email: $data['email'] ?? throw new \InvalidArgumentException('Email required'),
            phone: $data['phone'] ?? null,
        );
    }
}
```

---

## Repository Pattern

Abstract data access behind an interface:

```php
<?php
interface UserRepository
{
    public function findById(int $id): ?User;
    public function findByEmail(string $email): ?User;
    public function save(User $user): void;
    public function delete(User $user): void;
}

class DoctrineUserRepository implements UserRepository
{
    public function __construct(private EntityManagerInterface $em) {}

    public function findById(int $id): ?User
    {
        return $this->em->find(User::class, $id);
    }

    public function save(User $user): void
    {
        $this->em->persist($user);
        $this->em->flush();
    }
    // ...
}
```

---

## Service Layer

Encapsulate business logic in service classes:

```php
<?php
class RegistrationService
{
    public function __construct(
        private readonly UserRepository $users,
        private readonly PasswordHasher $hasher,
        private readonly MailerInterface $mailer,
        private readonly EventDispatcher $events,
    ) {}

    public function register(CreateUserRequest $request): User
    {
        // Business rules
        if ($this->users->findByEmail($request->email)) {
            throw new DuplicateEmailException($request->email);
        }

        $user = new User(
            name: $request->name,
            email: $request->email,
            password: $this->hasher->hash($request->password),
        );

        $this->users->save($user);
        $this->events->dispatch(new UserRegistered($user));
        $this->mailer->send($user->email, 'Welcome!');

        return $user;
    }
}
```

---

## SOLID Principles in PHP

| Principle | PHP Application |
|-----------|----------------|
| **S** — Single Responsibility | One class = one reason to change. `OrderService` doesn't send emails directly. |
| **O** — Open/Closed | Extend behavior via interfaces and DI, not by modifying existing code |
| **L** — Liskov Substitution | Any `MailerInterface` implementation must be swappable without breaking callers |
| **I** — Interface Segregation | Small, focused interfaces (`Readable`, `Writable`) over fat ones (`FileSystem`) |
| **D** — Dependency Inversion | Depend on abstractions (interfaces), not concrete classes |

---

## Key Takeaways

1. **Dependency injection** is the foundation — inject interfaces, not concrete classes
2. **Value objects** make invalid states unrepresentable (Money can't be negative)
3. **`readonly`** classes are perfect for DTOs and value objects — immutability by default
4. **Repository pattern** isolates data access — swap DB for in-memory in tests
5. **Service classes** orchestrate business logic — keep controllers thin
6. **SOLID** isn't academic — it directly determines testability and maintainability
