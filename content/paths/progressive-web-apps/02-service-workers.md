---
title: "Service Workers"
weight: 2
---

## What Is a Service Worker

A service worker is a JavaScript file that the browser runs in the background, separate from the web page. It acts as a **programmable network proxy** — intercepting every network request the page makes and deciding how to respond. This is the foundational technology that enables offline support, background sync, and push notifications.

Key properties:

| Property | Detail |
|----------|--------|
| **Thread** | Runs on its own thread — no access to the DOM |
| **Scope** | Controls all pages under its registered scope |
| **HTTPS only** | Requires a secure origin (localhost exempt for dev) |
| **Event-driven** | Wakes up in response to events, then terminates when idle |
| **No persistent state** | Cannot rely on global variables between events — use IndexedDB or the Cache API |
| **Async only** | Synchronous APIs (e.g. `localStorage`, `XHR`) are not available |

---

## Registration

Register a service worker from your main JavaScript file:

```javascript
// main.js
if ('serviceWorker' in navigator) {
  window.addEventListener('load', async () => {
    try {
      const registration = await navigator.serviceWorker.register('/sw.js', {
        scope: '/',
      });
      console.log('SW registered, scope:', registration.scope);
    } catch (error) {
      console.error('SW registration failed:', error);
    }
  });
}
```

### Scope Rules

The `scope` determines which pages the service worker controls. By default, the scope is the directory containing the service worker file.

| SW location | Default scope | Controls |
|-------------|---------------|----------|
| `/sw.js` | `/` | All pages on the origin |
| `/app/sw.js` | `/app/` | Only pages under `/app/` |
| `/scripts/sw.js` | `/scripts/` | Only pages under `/scripts/` |

You **cannot** register a scope above the service worker's location unless the server sends a `Service-Worker-Allowed` header:

```
Service-Worker-Allowed: /
```

---

## The Service Worker Lifecycle

The lifecycle has three main phases: **install**, **activate**, and **fetch** (plus other functional events). Understanding this lifecycle is critical — misunderstanding it is the most common source of PWA bugs.

```text
┌─────────┐    ┌───────────┐    ┌──────────┐
│ Install  │ → │  Waiting   │ → │ Activate  │ → [Controlling pages]
└─────────┘    └───────────┘    └──────────┘
     │               │                │
     │  Pre-cache    │ Old SW still   │ Clean up old
     │  resources    │ controls tabs  │ caches
```

### Install Event

Fired once when the browser detects a new or changed service worker file. Use it to pre-cache static assets:

```javascript
// sw.js
const CACHE_NAME = 'static-v1';
const PRECACHE_URLS = [
  '/',
  '/index.html',
  '/styles/main.css',
  '/scripts/app.js',
  '/images/logo.svg',
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then((cache) => cache.addAll(PRECACHE_URLS))
      .then(() => self.skipWaiting()) // Activate immediately (see below)
  );
});
```

`event.waitUntil()` tells the browser the install is not complete until the promise resolves. If pre-caching fails, the service worker is discarded.

### Activate Event

Fired after installation, once there are no other service workers controlling pages on this origin. Use it to clean up outdated caches:

```javascript
self.addEventListener('activate', (event) => {
  const currentCaches = [CACHE_NAME];

  event.waitUntil(
    caches.keys()
      .then((cacheNames) =>
        Promise.all(
          cacheNames
            .filter((name) => !currentCaches.includes(name))
            .map((name) => caches.delete(name))
        )
      )
      .then(() => self.clients.claim()) // Take control of open tabs immediately
  );
});
```

### Fetch Event

Fired for every network request made by pages the service worker controls. This is where caching strategies live:

```javascript
self.addEventListener('fetch', (event) => {
  // Only handle same-origin GET requests
  if (event.request.method !== 'GET') return;

  event.respondWith(
    caches.match(event.request).then((cached) => {
      if (cached) return cached;

      return fetch(event.request).then((response) => {
        // Clone the response — it can only be consumed once
        const cloned = response.clone();
        caches.open(CACHE_NAME).then((cache) => cache.put(event.request, cloned));
        return response;
      });
    })
  );
});
```

---

## The Update Flow

When the browser fetches the service worker file and finds even a **single byte** difference from the registered version, it triggers an update:

1. The new service worker is **installed** (install event fires).
2. The new worker enters the **waiting** state — it does not activate until all tabs controlled by the old worker are closed.
3. Once all old tabs close (or `skipWaiting()` is called), the new worker **activates**.
4. Subsequent navigations use the new service worker.

### Forcing Immediate Activation

```javascript
// In the new service worker's install event:
self.addEventListener('install', (event) => {
  self.skipWaiting(); // Skip the waiting phase
});

// In the activate event:
self.addEventListener('activate', (event) => {
  event.waitUntil(self.clients.claim()); // Take over all open tabs
});
```

**Warning:** `skipWaiting()` + `clients.claim()` means the new service worker takes control mid-session. If the cached assets change, the page may load a mix of old HTML and new CSS/JS. Use this pattern only when your code handles such mismatches gracefully, or when a full page reload is triggered.

### Notifying Users of Updates

```javascript
// main.js — after registration
const registration = await navigator.serviceWorker.register('/sw.js');

registration.addEventListener('updatefound', () => {
  const newWorker = registration.installing;

  newWorker.addEventListener('statechange', () => {
    if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
      // A new SW is installed but waiting — show update banner
      showUpdateBanner(() => {
        newWorker.postMessage({ type: 'SKIP_WAITING' });
      });
    }
  });
});

// Reload once the new SW takes over
navigator.serviceWorker.addEventListener('controllerchange', () => {
  window.location.reload();
});
```

In the service worker:

```javascript
self.addEventListener('message', (event) => {
  if (event.data?.type === 'SKIP_WAITING') {
    self.skipWaiting();
  }
});
```

---

## The navigator.serviceWorker API

The `navigator.serviceWorker` interface provides control over service worker registration from the page:

| Property / Method | Purpose |
|-------------------|---------|
| `navigator.serviceWorker.register(url, options)` | Register a service worker |
| `navigator.serviceWorker.ready` | Promise that resolves when a SW is active and controlling the page |
| `navigator.serviceWorker.controller` | The active SW controlling the current page (`null` on first load) |
| `navigator.serviceWorker.getRegistrations()` | Returns all registrations for this origin |
| `registration.update()` | Force-check for an updated SW file |
| `registration.unregister()` | Unregister the service worker |
| `registration.installing` | SW in the installing state |
| `registration.waiting` | SW installed but waiting to activate |
| `registration.active` | The currently active SW |

### Communication Between Page and Service Worker

Use `postMessage` for bidirectional communication:

```javascript
// Page → Service Worker
navigator.serviceWorker.controller.postMessage({
  type: 'CACHE_URLS',
  urls: ['/api/articles/1', '/api/articles/2'],
});

// Service Worker → Page
self.addEventListener('message', (event) => {
  if (event.data?.type === 'CACHE_URLS') {
    const cache = await caches.open('dynamic-v1');
    await cache.addAll(event.data.urls);
    event.source.postMessage({ type: 'CACHE_COMPLETE' });
  }
});

// Page listens for messages from SW
navigator.serviceWorker.addEventListener('message', (event) => {
  if (event.data?.type === 'CACHE_COMPLETE') {
    console.log('URLs cached successfully');
  }
});
```

---

## Debugging with DevTools

Chrome DevTools provides comprehensive service worker tooling under **Application → Service Workers**:

### Key Actions

| DevTools Feature | What It Does |
|-----------------|-------------|
| **Update on reload** | Forces the new SW to install and activate on every page reload — essential for development |
| **Bypass for network** | Ignores the SW's fetch handler — requests go directly to the network |
| **Offline** | Simulates offline mode to test cache behaviour |
| **Unregister** | Removes the service worker registration |
| **Stop / Start** | Manually control the SW lifecycle |
| **skipWaiting** | Manually advance a waiting SW to active |

### Cache Storage Inspector

Under **Application → Cache Storage**, you can browse every named cache, inspect individual entries, and delete items.

### Common Debugging Issues

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| SW not updating | Browser cached the SW file | Set `updateViaCache: 'none'` on registration, or add cache-busting headers on the server |
| Old content served | Old cache not cleaned up in activate | Delete outdated caches in the activate event |
| Fetch handler not firing | Request outside the SW's scope | Check scope matches, or move SW file |
| `install` event fails silently | One URL in `addAll()` returns 404 | Verify all pre-cache URLs are reachable |
| Infinite redirect loop | SW intercepts navigation to its own scope incorrectly | Exclude the SW file itself from fetch handling |

### Checking SW Status Programmatically

```javascript
async function logSWStatus() {
  const registrations = await navigator.serviceWorker.getRegistrations();
  for (const reg of registrations) {
    console.log('Scope:', reg.scope);
    console.log('Installing:', reg.installing?.state);
    console.log('Waiting:', reg.waiting?.state);
    console.log('Active:', reg.active?.state);
  }
}
```

---

## Service Worker Lifecycle Summary

```text
Browser detects new/changed SW file
        │
        ▼
   ┌──────────┐
   │ INSTALL   │ → Pre-cache assets via cache.addAll()
   └────┬─────┘
        │ success
        ▼
   ┌──────────┐
   │ WAITING   │ → Old SW still controls tabs
   └────┬─────┘   (or skipWaiting() bypasses this)
        │ all old tabs closed
        ▼
   ┌──────────┐
   │ ACTIVATE  │ → Clean up old caches, clients.claim()
   └────┬─────┘
        │
        ▼
   ┌──────────┐
   │ IDLE      │ → Listens for fetch, push, sync, message events
   └────┬─────┘
        │ no events for a while
        ▼
   ┌──────────┐
   │ TERMINATED│ → Browser reclaims memory; wakes on next event
   └──────────┘
```

---

## Key Takeaways

- A service worker is a programmable network proxy that runs on a separate thread — it has no DOM access, no synchronous APIs, and no persistent in-memory state.
- The lifecycle (install → waiting → activate) ensures that a single version of the service worker controls all open tabs, preventing asset version mismatches.
- `skipWaiting()` and `clients.claim()` bypass the waiting phase for faster activation, but must be used carefully to avoid serving mismatched resources.
- Every network request from controlled pages flows through the `fetch` event, which is where you implement caching strategies.
- `postMessage` provides bidirectional communication between the page and the service worker — use it for update notifications, cache preloading, and state synchronisation.
- Always enable "Update on reload" during development, and test with the "Offline" checkbox to verify cache behaviour.
