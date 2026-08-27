---
title: "Web Security"
weight: 131
bookCollapseSection: true
---

# Web Security

A comprehensive learning path covering the security of web applications — from foundational threat modelling and the OWASP Top 10 through to practical attack prevention, authentication and authorisation design, and operational security testing.

## Overview

Every application exposed to the internet is a target. Web security is not an afterthought bolted on at the end of a project — it is a discipline that must be woven into every layer: the HTTP headers your server sends, the queries your ORM builds, the tokens your authentication system issues, and the way your CI pipeline scans for known vulnerabilities.

This path teaches you to think like an attacker so you can build like a defender. Each section pairs real attack payloads with concrete prevention techniques across multiple languages and frameworks. You will learn not just *what* can go wrong, but *why* it goes wrong and *how* to stop it.

## Prerequisites

- Solid understanding of HTTP (methods, headers, status codes, cookies)
- Comfortable reading code in at least one backend language (Python, Java, JavaScript, Go)
- Basic familiarity with HTML, JavaScript, and the browser DOM
- Understanding of relational databases and SQL basics
- Familiarity with command-line tools

## Sections

| # | Section | Topics |
|---|---------|--------|
| 1 | [Web Security Fundamentals]({{< relref "01-fundamentals" >}}) | Threat landscape, CIA triad, attack surfaces, defence in depth, OWASP Top 10 overview |
| 2 | [Injection Attacks]({{< relref "02-injection-attacks" >}}) | SQL injection, NoSQL injection, command injection, LDAP injection, parameterised queries |
| 3 | [Cross-Site Scripting (XSS)]({{< relref "03-cross-site-scripting" >}}) | Reflected, stored, DOM-based XSS, Content Security Policy, sanitisation, encoding |
| 4 | [Authentication]({{< relref "04-authentication" >}}) | Password hashing, session management, MFA, credential stuffing, brute force protection |
| 5 | [Authorisation]({{< relref "05-authorisation" >}}) | RBAC, ABAC, broken access control, IDOR, privilege escalation, JWT pitfalls |
| 6 | [OAuth 2.0 & OpenID Connect]({{< relref "06-oauth-openid-connect" >}}) | Auth code, PKCE, client credentials, tokens, scopes, common mistakes, token storage |
| 7 | [CSRF & Other Attacks]({{< relref "07-csrf-and-other-attacks" >}}) | CSRF tokens, SameSite cookies, clickjacking, SSRF, open redirect |
| 8 | [Transport Security]({{< relref "08-transport-security" >}}) | TLS/HTTPS, HSTS, certificate pinning, mixed content, secure headers, cookie attributes |
| 9 | [API Security]({{< relref "09-api-security" >}}) | Rate limiting, input validation, API keys vs OAuth, CORS, GraphQL security, webhooks |
| 10 | [Security Testing & Operations]({{< relref "10-security-testing-operations" >}}) | SAST, DAST, dependency scanning, penetration testing, security headers audit, incident response |
