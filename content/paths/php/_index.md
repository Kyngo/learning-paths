---
title: "PHP"
weight: 70
bookFlatSection: false
bookCollapseSection: true
---

PHP is a server-side scripting language that powers over 75% of websites with known server-side technology. Despite its chaotic early history, modern PHP (8.x) is a fast, type-safe, well-designed language with a mature ecosystem. It's the backbone of WordPress, Laravel, Symfony, and Drupal.

## Prerequisites

- Programming Logic (variables, control flow, functions, data structures)
- Basic understanding of HTTP (how web requests work)

---

## Sections

1. [Syntax, Types, and the Runtime]({{< relref "01-syntax-types-runtime" >}})
2. [Control Flow and Functions]({{< relref "02-control-flow-functions" >}})
3. [Arrays and Strings]({{< relref "03-arrays-strings" >}})
4. [Object-Oriented PHP]({{< relref "04-oop" >}})
5. [Error Handling and Exceptions]({{< relref "05-error-handling" >}})
6. [The Type System (PHP 8.x)]({{< relref "06-type-system" >}})
7. [Namespaces, Autoloading, and Composer]({{< relref "07-namespaces-composer" >}})
8. [Working with Databases (PDO)]({{< relref "08-databases-pdo" >}})
9. [HTTP, Sessions, and the Request Lifecycle]({{< relref "09-http-sessions" >}})
10. [Testing with PHPUnit]({{< relref "10-testing" >}})
11. [Modern PHP Patterns and Architecture]({{< relref "11-modern-patterns" >}})
12. [Frameworks: Laravel and Symfony]({{< relref "12-frameworks" >}})
13. [Performance, OpCache, and Deployment]({{< relref "13-performance-deployment" >}})

---

## How PHP Executes

```mermaid
flowchart LR
    A["HTTP Request"] --> B["Web Server<br/>(Apache/Nginx)"]
    B --> C["PHP-FPM<br/>(FastCGI Process Manager)"]
    C --> D["Zend Engine:<br/>Lex → Parse → Compile"]
    D --> E["Opcodes"]
    E --> F["Zend VM Execution"]
    F --> G["Response"]
    G --> B
    B --> H["HTTP Response"]
```

## Why Learn PHP Today?

| Reason | Detail |
|--------|--------|
| **Market share** | 75%+ of web — WordPress alone is 40%+ of all sites |
| **Mature ecosystem** | Composer, Laravel, Symfony, PHPUnit, static analysis (PHPStan, Psalm) |
| **Modern language features** | Enums, fibers, readonly properties, union/intersection types, match expressions |
| **Fast** | PHP 8.x with JIT is competitive with other interpreted languages |
| **Easy deployment** | Shared hosting, Docker, PaaS — runs everywhere |
| **Jobs** | Enormous demand, especially in CMS, e-commerce, and enterprise |
