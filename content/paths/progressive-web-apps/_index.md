---
title: "Progressive Web Apps"
weight: 101
bookCollapseSection: true
---

# Progressive Web Apps

A comprehensive learning path covering the design, implementation, and deployment of Progressive Web Apps — from service worker fundamentals and caching strategies through to push notifications, advanced web APIs, and production deployment patterns.

## Overview

Progressive Web Apps bridge the gap between web and native. They load like ordinary web pages but offer capabilities traditionally reserved for platform-specific applications: offline access, push notifications, home screen installation, and background processing. A well-built PWA delivers a fast, reliable, engaging experience regardless of network conditions.

This path takes you from the foundational concepts — what makes a web app "progressive" and how the browser decides it is installable — through the technical machinery that powers offline experiences (service workers, Cache API, IndexedDB) and into the advanced APIs that are rapidly closing the capability gap with native platforms. Every section pairs conceptual understanding with production-ready JavaScript code you can adapt to your own projects.

## Prerequisites

- Solid understanding of HTML, CSS, and modern JavaScript (ES2020+)
- Familiarity with the Fetch API, Promises, and async/await
- Basic understanding of HTTP (methods, headers, status codes, HTTPS)
- Comfort reading and writing JSON
- Familiarity with browser DevTools (Network, Application, Console panels)

## Sections

| # | Section | Topics |
|---|---------|--------|
| 1 | [PWA Fundamentals]({{< relref "01-pwa-fundamentals" >}}) | What makes a PWA, web app manifest, installability criteria, app shell architecture, Lighthouse audit, PWA vs native vs hybrid |
| 2 | [Service Workers]({{< relref "02-service-workers" >}}) | Lifecycle (install, activate, fetch), registration, scope, update flow, debugging with DevTools, navigator.serviceWorker API |
| 3 | [Caching Strategies]({{< relref "03-caching-strategies" >}}) | Cache-first, network-first, stale-while-revalidate, cache-only, network-only, runtime caching, cache versioning, Workbox library |
| 4 | [Offline-First Architecture]({{< relref "04-offline-first" >}}) | Offline detection, IndexedDB for client-side storage, Background Sync API, conflict resolution, offline UX patterns |
| 5 | [Push Notifications]({{< relref "05-push-notifications" >}}) | Notification API, Push API, VAPID keys, subscription management, server-side push (web-push library), notification best practices, permission UX |
| 6 | [Advanced Web APIs]({{< relref "06-advanced-web-apis" >}}) | Web Share API, File System Access API, Web Bluetooth, Web USB, Badging API, Share Target API, periodic background sync, Project Fugu |
| 7 | [Performance & Loading]({{< relref "07-performance-loading" >}}) | Core Web Vitals (LCP, INP, CLS), code splitting, lazy loading, preloading, resource hints, PRPL pattern, performance budgets |
| 8 | [Testing & Deployment]({{< relref "08-testing-deployment" >}}) | Lighthouse CI, Workbox build integration, service worker testing strategies, manifest validation, HTTPS requirements, CDN deployment, update strategies |
