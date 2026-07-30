---
title: "Namespaces, Autoloading, and Composer"
weight: 7
---

## Namespaces

Namespaces prevent class name collisions and organize code into logical hierarchies:

```php
<?php
// File: src/Payment/Stripe/Client.php
namespace App\Payment\Stripe;

class Client
{
    public function charge(float $amount): Receipt { /* ... */ }
}
```

```php
<?php
// File: src/Http/Client.php
namespace App\Http;

class Client
{
    public function get(string $url): Response { /* ... */ }
}
```

Both classes are named `Client`, but they don't conflict because they're in different namespaces.

### Using Namespaced Classes

```php
<?php
namespace App\Service;

// Import with 'use'
use App\Payment\Stripe\Client as StripeClient;
use App\Http\Client as HttpClient;
use App\Entity\{User, Order, Product};  // Group imports

class PaymentService
{
    public function __construct(
        private StripeClient $stripe,
        private HttpClient $http,
    ) {}
}
```

### Convention: PSR-4

The standard maps namespace to directory structure:

```
Namespace: App\Payment\Stripe\Client
File path: src/Payment/Stripe/Client.php
```

One class per file. File name matches class name exactly (case-sensitive).

---

## Autoloading

PHP doesn't automatically know where classes live. Autoloading registers a callback that loads the file when a class is first used:

```php
<?php
// Manual autoloading (don't do this in practice)
spl_autoload_register(function (string $class): void {
    $path = __DIR__ . '/src/' . str_replace('\\', '/', $class) . '.php';
    if (file_exists($path)) {
        require $path;
    }
});
```

In practice, **Composer handles autoloading** for you.

---

## Composer

Composer is PHP's dependency manager and autoloader. Every modern PHP project uses it.

### Initializing a Project

```bash
composer init
```

### composer.json

```json
{
    "name": "acme/my-project",
    "description": "My PHP project",
    "type": "project",
    "require": {
        "php": ">=8.2",
        "guzzlehttp/guzzle": "^7.8",
        "monolog/monolog": "^3.5"
    },
    "require-dev": {
        "phpunit/phpunit": "^11.0",
        "phpstan/phpstan": "^1.10"
    },
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "App\\Tests\\": "tests/"
        }
    }
}
```

### Essential Composer Commands

```bash
composer install              # Install dependencies from lock file
composer update              # Update dependencies to latest matching versions
composer require vendor/pkg  # Add a dependency
composer require --dev vendor/pkg  # Add a dev dependency
composer dump-autoload       # Regenerate autoload files
composer dump-autoload -o    # Optimized (classmap) for production
```

### Using the Autoloader

```php
<?php
// This single line loads everything
require __DIR__ . '/vendor/autoload.php';

// Now all classes from your project (App\*) and dependencies are available
$client = new \GuzzleHttp\Client();
$logger = new \Monolog\Logger('app');
```

### Version Constraints

| Constraint | Meaning |
|-----------|---------|
| `^7.8` | >=7.8.0, <8.0.0 (recommended — allows minor updates) |
| `~7.8` | >=7.8.0, <7.9.0 (allows patch updates only) |
| `>=7.8` | 7.8 or higher (no upper bound — risky) |
| `7.8.*` | Any 7.8.x patch version |
| `7.8.2` | Exact version (too restrictive for libraries) |

### composer.lock

- **Always commit `composer.lock`** for applications (ensures everyone gets identical versions)
- Run `composer install` (not `update`) in CI/CD and deployment
- Run `composer update` when you intentionally want newer versions

---

## Packages Every PHP Developer Should Know

| Package | Purpose |
|---------|---------|
| `guzzlehttp/guzzle` | HTTP client |
| `monolog/monolog` | Logging (PSR-3 compatible) |
| `symfony/console` | CLI applications |
| `league/flysystem` | Filesystem abstraction (local, S3, FTP) |
| `ramsey/uuid` | UUID generation |
| `carbon/carbon` | Date/time manipulation |
| `phpstan/phpstan` | Static analysis |
| `phpunit/phpunit` | Testing framework |
| `php-cs-fixer/shim` | Code style fixer |

---

## PSR Standards

PHP-FIG (Framework Interop Group) defines PSR standards for interoperability:

| PSR | Topic | Key Point |
|-----|-------|-----------|
| PSR-1 | Basic coding standard | Classes in PascalCase, methods in camelCase |
| PSR-4 | Autoloading | Namespace ↔ directory mapping |
| PSR-7 | HTTP messages | Request/Response interfaces |
| PSR-11 | Container | Dependency injection container interface |
| PSR-12 | Extended coding style | Braces, spacing, line length (superseded by PER-CS) |
| PSR-15 | HTTP handlers | Middleware interfaces |
| PSR-18 | HTTP client | Client interface (Guzzle, Symfony HTTP) |

---

## Key Takeaways

1. **One class per file**, file name = class name, namespace = directory path (PSR-4)
2. **Composer is non-negotiable** — every PHP project starts with `composer init`
3. **Commit `composer.lock`** for apps, use `composer install` in CI/production
4. **Use `^` version constraints** — they allow compatible updates without breaking changes
5. **`vendor/autoload.php`** is the only `require` you need — everything else autoloads
6. **PSR standards** ensure libraries work together — follow PSR-4 and PSR-12/PER-CS
