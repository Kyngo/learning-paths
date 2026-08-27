# Software Engineering Learning Paths

A structured collection of learning paths covering fundamental and advanced software engineering topics. Built as a [Hugo](https://gohugo.io/) static site with the [Book](https://themes.gohugo.io/themes/hugo-book/) theme.

**Live site:** https://kyngo.github.io/learning-paths/

## Purpose

This repository provides in-depth, technically rigorous learning material for software engineers at any level. The paths are designed to be:

- **Progressive** — each topic builds from foundational concepts to advanced patterns
- **Practical** — theory is paired with real-world examples and exercises
- **Clear** — complex topics are explained as simply as the subject allows, without sacrificing technical accuracy

## Learning Paths

| Path | Description |
|------|-------------|
| Programming Logic | Algorithms, data structures, control flow, computational thinking |
| DSA Interview Prep | Sliding window, two pointers, trees, heaps, backtracking, binary search |
| Databases & SQL | Relational design, SQL mastery, indexing, transactions, NoSQL |
| System Design | Scalability, load balancing, caching, distributed systems |
| Software Architecture | Design patterns, clean architecture, DDD, event-driven systems |
| APIs & Web Services | REST, authentication, GraphQL, webhooks, rate limiting |
| Testing & Quality | Unit/integration/e2e testing, TDD, property testing, performance |
| Operating Systems | Processes, memory, file systems, I/O, concurrency, containers |
| Python | Python language from basics to advanced patterns |
| Java | Java ecosystem, OOP, concurrency, enterprise patterns |
| PHP | PHP from basics to modern frameworks (Laravel, Symfony) |
| Go | Goroutines, channels, interfaces, standard library, modules, patterns |
| JavaScript / TypeScript / Node.js | Full JS ecosystem including types and server-side |
| Swift and SwiftUI | Apple development — Swift language and SwiftUI framework |
| Frontend Development | CSS, browser APIs, responsive design, accessibility |
| Terraform | Infrastructure as Code with Terraform |
| Amazon Web Services | AWS cloud services and architecture |
| DevOps & CI/CD | Pipelines, GitLab CI, GitHub Actions, GitOps, DORA metrics |
| Kubernetes | Pods, deployments, networking, Helm, security, operations |
| Scripting | Shell scripting, automation, text processing |
| Tools | Git, Docker, SSH, Apache, Nginx, package management |
| Git Advanced | Internals, rebase, reflog, hooks, workflows at scale |
| Networking | TCP/IP, DNS, HTTP, subnetting, network architecture |
| Mathematics for Engineers | Number systems, algebra, discrete math, linear algebra, probability, statistics, calculus, information theory |
| IT Fundamentals | Operating systems, hardware, computer assembly, security basics |
| Cybersecurity | Threat landscape, cryptography, OSINT, offensive/defensive security |
| AI and Large Language Models | Neural networks, transformers, LLMs, RAG, agents, MCP, prompt engineering |
| Soft Skills | Communication, leadership, collaboration, career growth |

## Structure

Content lives under `content/paths/`. Each learning path has its own folder with an `_index.md` and numbered section files:

```
content/paths/
├── python/
│   ├── _index.md
│   ├── 01-basics.md
│   ├── 02-data-structures.md
│   └── ...
├── tools/
│   ├── _index.md
│   ├── 01-git.md
│   ├── 02-docker.md
│   └── ...
└── ...
```

## Local Development

```bash
# Prerequisites: Hugo extended (v0.164.0+)
brew install hugo

# Run dev server
hugo server

# Build
hugo --gc --minify
```

## Recommended Order for Beginners

```text
IT Fundamentals → Programming Logic → Python → Scripting → Networking → AWS
```

Or for web development:

```text
Programming Logic → JavaScript/TypeScript/Node.js → Frontend Development → Terraform → AWS
```

## Deployment

Deployed automatically to GitHub Pages via GitHub Actions on push to `master`.
