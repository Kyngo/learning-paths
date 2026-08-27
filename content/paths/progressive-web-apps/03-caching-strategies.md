---
title: "Caching Strategies"
weight: 3
---

## The Cache API

The Cache API provides a storage mechanism for request/response pairs, designed specifically for service workers. Unlike the browser's HTTP cache, you have full programmatic control over what goes in and what comes out.

```javascript
// Open (or create) a named cache
const cache = await caches.open('my-cache-v1');

// Store a response
await cache.put(request, response);

// Store by fetching the URL
await cache.add('/styles/main.css');

// Store multiple URLs at once
await cache.addAll(['/index.html', '/app.js', '/styles.css']);

// Retrieve a cached response
const response = await caches.match(request);

// Delete an entry
await cache.delete(request);

// List all keys
const keys = await cache.keys();
```

Important: `cache.add()` and `cache.addAll()` will reject if **any** response is not a successful (2xx) HTTP response. A single 404 in the list fails the entire batch.

---

## Caching Strategy Patterns

Each strategy makes a different trade-off between freshness and speed. Choose based on the resource type and how stale data impacts user experience.

### Cache-First (Cache Falling Back to Network)

Serve from cache if available; fetch from network only on cache miss. Best for versioned static assets that do not change at a given URL.

```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) => {
      return cached || fetch(event.request);
    })
  );
});
```

| Pros | Cons |
|------|------|
| Fastest response for cached resources | Stale content until cache is explicitly updated |
| Works fully offline for cached URLs | Cache can grow unbounded without eviction logic |

**Best for:** CSS/JS with hashed filenames, font files, images with versioned URLs.

### Network-First (Network Falling Back to Cache)

Always try the network; fall back to cache on failure. Ensures users get the freshest content when online.

```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(
    fetch(event.request)
      .then((response) => {
        // Update the cache with the fresh response
        const cloned = response.clone();
        caches.open('dynamic-v1').then((cache) => cache.put(event.request, cloned));
        return response;
      })
      .catch(() => caches.match(event.request))
  );
});
```

| Pros | Cons |
|------|------|
| Always fresh when online | Slower — waits for network before responding |
| Graceful offline fallback | Network timeout can delay response significantly |

**Best for:** API responses, HTML pages, news feeds, any content where freshness matters more than speed.

### Stale-While-Revalidate

Respond immediately from cache, then fetch from the network in the background and update the cache for next time. The user gets instant speed now and fresh content on the next visit.

```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.open('swr-cache-v1').then((cache) => {
      return cache.match(event.request).then((cached) => {
        const networkFetch = fetch(event.request).then((response) => {
          cache.put(event.request, response.clone());
          return response;
        });

        return cached || networkFetch;
      });
    })
  );
});
```

| Pros | Cons |
|------|------|
| Instant response from cache | Content may be one version behind |
| Background update keeps cache fresh | More complex logic |

**Best for:** User avatars, social media feeds, product listings, analytics scripts — anything where slightly stale data is acceptable.

### Cache-Only

Serve exclusively from cache. If the resource is not cached, the request fails.

```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(caches.match(event.request));
});
```

**Best for:** Pre-cached app shell resources that are guaranteed to be in the cache after installation.

### Network-Only

Bypass the cache entirely. The service worker does not intervene.

```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(fetch(event.request));
});
```

**Best for:** Non-GET requests, analytics pings, real-time API calls that must never be stale.

---

## Strategy Comparison

| Strategy | Speed | Freshness | Offline | Best For |
|----------|-------|-----------|---------|----------|
| Cache-first | ★★★★★ | ★★☆☆☆ | ✅ Cached items | Versioned static assets |
| Network-first | ★★☆☆☆ | ★★★★★ | ✅ If cached | HTML, API data |
| Stale-while-revalidate | ★★★★☆ | ★★★★☆ | ✅ If cached | Feeds, avatars, listings |
| Cache-only | ★★★★★ | ★☆☆☆☆ | ✅ If cached | App shell |
| Network-only | ★★☆☆☆ | ★★★★★ | ❌ | Analytics, POST requests |

---

## Runtime Caching

Pre-caching handles known static assets. Runtime caching handles requests encountered during normal use — API calls, images loaded on demand, third-party resources.

### Adding a Network Timeout

Prevent slow networks from stalling the response:

```javascript
function networkFirstWithTimeout(request, timeoutMs, cacheName) {
  return new Promise((resolve, reject) => {
    const timeoutId = setTimeout(() => {
      caches.match(request).then((cached) => {
        if (cached) resolve(cached);
      });
    }, timeoutMs);

    fetch(request)
      .then((response) => {
        clearTimeout(timeoutId);
        const cloned = response.clone();
        caches.open(cacheName).then((cache) => cache.put(request, cloned));
        resolve(response);
      })
      .catch(() => {
        clearTimeout(timeoutId);
        caches.match(request).then((cached) => {
          cached ? resolve(cached) : reject(new Error('No cache and network failed'));
        });
      });
  });
}
```

### Route-Based Strategy Selection

A real service worker combines multiple strategies based on the request URL:

```javascript
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);

  // App shell — cache-first
  if (url.origin === location.origin && SHELL_FILES.includes(url.pathname)) {
    event.respondWith(caches.match(request));
    return;
  }

  // API calls — network-first with timeout
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirstWithTimeout(request, 3000, 'api-cache-v1'));
    return;
  }

  // Images — stale-while-revalidate
  if (request.destination === 'image') {
    event.respondWith(
      caches.open('image-cache-v1').then((cache) =>
        cache.match(request).then((cached) => {
          const fresh = fetch(request).then((response) => {
            cache.put(request, response.clone());
            return response;
          });
          return cached || fresh;
        })
      )
    );
    return;
  }

  // Everything else — network-first
  event.respondWith(
    fetch(request).catch(() => caches.match(request))
  );
});
```

---

## Cache Versioning and Cleanup

When you change cached resources, increment the cache version and delete old caches during activation:

```javascript
const CACHE_VERSION = 'v3';
const STATIC_CACHE = `static-${CACHE_VERSION}`;
const DYNAMIC_CACHE = `dynamic-${CACHE_VERSION}`;
const EXPECTED_CACHES = [STATIC_CACHE, DYNAMIC_CACHE];

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((names) =>
      Promise.all(
        names
          .filter((name) => !EXPECTED_CACHES.includes(name))
          .map((name) => {
            console.log('Deleting old cache:', name);
            return caches.delete(name);
          })
      )
    )
  );
});
```

### Cache Size Management

Browsers impose per-origin storage quotas. Prevent unbounded growth:

```javascript
async function trimCache(cacheName, maxEntries) {
  const cache = await caches.open(cacheName);
  const keys = await cache.keys();
  if (keys.length > maxEntries) {
    await cache.delete(keys[0]); // Delete oldest entry (FIFO)
    await trimCache(cacheName, maxEntries); // Recurse until under limit
  }
}
```

---

## Workbox Library

[Workbox](https://developer.chrome.com/docs/workbox/) is Google's production-grade library for service worker caching. It abstracts the manual patterns above into a declarative API.

### Installation

```bash
npm install workbox-precaching workbox-routing workbox-strategies workbox-expiration
```

### Workbox Service Worker

```javascript
// sw.js using Workbox modules
import { precacheAndRoute } from 'workbox-precaching';
import { registerRoute } from 'workbox-routing';
import { CacheFirst, NetworkFirst, StaleWhileRevalidate } from 'workbox-strategies';
import { ExpirationPlugin } from 'workbox-expiration';

// Pre-cache the app shell (injected at build time)
precacheAndRoute(self.__WB_MANIFEST);

// Images — cache-first with expiration
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'images',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 60,
        maxAgeSeconds: 30 * 24 * 60 * 60, // 30 days
      }),
    ],
  })
);

// API calls — network-first
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({
    cacheName: 'api-responses',
    networkTimeoutSeconds: 3,
    plugins: [
      new ExpirationPlugin({
        maxEntries: 50,
        maxAgeSeconds: 5 * 60, // 5 minutes
      }),
    ],
  })
);

// CSS/JS — stale-while-revalidate
registerRoute(
  ({ request }) => request.destination === 'style' || request.destination === 'script',
  new StaleWhileRevalidate({
    cacheName: 'static-resources',
  })
);
```

### Workbox Build Integration

Generate the precache manifest at build time:

```javascript
// workbox-config.js
module.exports = {
  globDirectory: 'dist/',
  globPatterns: ['**/*.{html,js,css,png,svg,woff2}'],
  swSrc: 'src/sw.js',
  swDest: 'dist/sw.js',
  maximumFileSizeToCacheInBytes: 5 * 1024 * 1024,
};
```

```bash
npx workbox-cli injectManifest workbox-config.js
```

This replaces `self.__WB_MANIFEST` in your source with the actual list of versioned URLs.

### Workbox Strategies Summary

| Workbox Class | Manual Equivalent |
|---------------|-------------------|
| `CacheFirst` | Cache-first |
| `NetworkFirst` | Network-first (with optional timeout) |
| `StaleWhileRevalidate` | Stale-while-revalidate |
| `CacheOnly` | Cache-only |
| `NetworkOnly` | Network-only |

---

## Key Takeaways

- The Cache API gives you full programmatic control over stored request/response pairs — unlike the HTTP cache, entries never expire unless you delete them.
- Choose strategies by resource type: cache-first for versioned static assets, network-first for API data, stale-while-revalidate for content where slight staleness is acceptable.
- A production service worker combines multiple strategies using URL pattern matching — a single strategy for all requests is almost never correct.
- Version your caches and clean up old versions in the `activate` event to prevent stale resources and unbounded storage growth.
- Workbox abstracts manual caching patterns into a declarative, plugin-based API with built-in expiration, precaching, and build integration.
- Always set cache size limits (max entries and max age) on runtime caches to prevent quota eviction by the browser.
