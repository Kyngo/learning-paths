---
title: "Testing with PHPUnit"
weight: 10
---

## PHPUnit Basics

PHPUnit is the standard testing framework for PHP.

### Setup

```bash
composer require --dev phpunit/phpunit
```

```xml
<!-- phpunit.xml -->
<phpunit bootstrap="vendor/autoload.php" colors="true">
    <testsuites>
        <testsuite name="Unit">
            <directory>tests/</directory>
        </testsuite>
    </testsuites>
    <source>
        <include>
            <directory>src/</directory>
        </include>
    </source>
</phpunit>
```

### Writing Tests

```php
<?php
// tests/Calculator/CalculatorTest.php
namespace App\Tests\Calculator;

use App\Calculator\Calculator;
use PHPUnit\Framework\TestCase;

class CalculatorTest extends TestCase
{
    private Calculator $calculator;

    protected function setUp(): void
    {
        $this->calculator = new Calculator();
    }

    public function testAddPositiveNumbers(): void
    {
        $result = $this->calculator->add(2, 3);
        $this->assertSame(5, $result);
    }

    public function testDivideByZeroThrowsException(): void
    {
        $this->expectException(\DivisionByZeroError::class);
        $this->calculator->divide(10, 0);
    }

    /**
     * @dataProvider additionProvider
     */
    public function testAddWithDataProvider(int $a, int $b, int $expected): void
    {
        $this->assertSame($expected, $this->calculator->add($a, $b));
    }

    public static function additionProvider(): array
    {
        return [
            'positive numbers' => [1, 2, 3],
            'negative numbers' => [-1, -2, -3],
            'zero'             => [0, 0, 0],
            'mixed signs'      => [-5, 3, -2],
        ];
    }
}
```

### Running Tests

```bash
vendor/bin/phpunit                    # All tests
vendor/bin/phpunit tests/Calculator/  # Specific directory
vendor/bin/phpunit --filter=testAdd   # Specific test method
vendor/bin/phpunit --coverage-html=coverage/  # With code coverage report
```

---

## Assertions

| Assertion | Purpose |
|-----------|---------|
| `assertSame($expected, $actual)` | Strict equality (===) |
| `assertEquals($expected, $actual)` | Loose equality (==) |
| `assertTrue($value)` | Assert truthy |
| `assertFalse($value)` | Assert falsy |
| `assertNull($value)` | Assert null |
| `assertCount($n, $array)` | Array/Countable has exactly N items |
| `assertContains($needle, $haystack)` | Array contains value |
| `assertArrayHasKey($key, $arr)` | Array has specific key |
| `assertInstanceOf(Class::class, $obj)` | Object is instance of class |
| `assertStringContainsString($sub, $str)` | String contains substring |
| `assertMatchesRegularExpression($re, $str)` | Regex match |

**Prefer `assertSame` over `assertEquals`** — strict comparison catches type bugs.

---

## Mocking

Mock external dependencies to test in isolation:

```php
<?php
use PHPUnit\Framework\TestCase;

class OrderServiceTest extends TestCase
{
    public function testPlaceOrderSendsNotification(): void
    {
        // Create mock
        $notifier = $this->createMock(NotifierInterface::class);
        
        // Set expectation: notify() must be called exactly once with any Order
        $notifier->expects($this->once())
            ->method('notify')
            ->with($this->isInstanceOf(Order::class));
        
        // Inject mock
        $service = new OrderService($notifier);
        $service->placeOrder(new Order(amount: 99.99));
    }

    public function testGetUserReturnsFromRepository(): void
    {
        $user = new User(id: 1, name: 'Alice');
        
        $repo = $this->createMock(UserRepository::class);
        $repo->method('find')
            ->with(1)
            ->willReturn($user);
        
        $service = new UserService($repo);
        $result = $service->getUser(1);
        
        $this->assertSame($user, $result);
    }
}
```

---

## Test Organization

```
tests/
├── Unit/                    # Fast, isolated, no I/O
│   ├── Service/
│   │   └── OrderServiceTest.php
│   └── Calculator/
│       └── CalculatorTest.php
├── Integration/             # Tests with real database/APIs
│   └── Repository/
│       └── UserRepositoryTest.php
└── Functional/              # Full request/response cycle
    └── Api/
        └── OrderEndpointTest.php
```

---

## Key Takeaways

1. **One test class per production class**, mirroring the `src/` structure
2. **Use `assertSame`** (strict) over `assertEquals` (loose) by default
3. **Data providers** eliminate repetitive test methods — test many inputs in one method
4. **Mock interfaces, not concrete classes** — this enforces proper dependency inversion
5. **Test behavior, not implementation** — verify what a method does, not how it does it
6. **`setUp()` for shared setup**, `tearDown()` for cleanup — runs before/after each test
