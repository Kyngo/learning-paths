---
title: "Testing & Deployment"
weight: 8
---

## Testing Service Workers

Service worker testing is inherently different from unit testing regular JavaScript. A service worker runs in a separate thread, intercepts real network requests, and persists across page loads. You need a combination of unit tests (for strategy logic), integration tests (for service worker behaviour), and manual DevTools verification.

### Unit Testing Caching Logic

Extract caching strategies into pure functions that can be tested outside the service worker:

```javascript
// cache-strategies.js — testable without a service worker
export function shouldCacheResponse(response) {
  if (!response || !response.ok) return false;
  if (response.status === 206) return false; // Partial content
  if (response.headers.get('Cache-Control')?.includes('no-store')) return false;
  return true;
}

export function getCacheNameForRequest(url) {
  const path = new URL(url).pathname;
  if (path.startsWith('/api/')) return 'api-cache';
  if (path.match(/\.(png|jpg|webp|svg)$/)) return 'image-cache';
  return 'static-cache';
}
```

```javascript
// cache-strategies.test.js
import { describe, it, expect } from 'vitest';
import { shouldCacheResponse, getCacheNameForRequest } from './cache-strategies.js';

describe('shouldCacheResponse', () => {
  it('returns true for a 200 response', () => {
    const response = new Response('ok', { status: 200 });
    expect(shouldCacheResponse(response)).toBe(true);
  });

  it('returns false for a 404 response', () => {
    const response = new Response('not found', { status: 404 });
    expect(shouldCacheResponse(response)).toBe(false);
  });

  it('returns false for null response', () => {
    expect(shouldCacheResponse(null)).toBe(false);
  });
});

describe('getCacheNameForRequest', () => {
  it('routes API calls to api-cache', () => {
    expect(getCacheNameForRequest('https://example.com/api/users')).toBe('api-cache');
  });

  it('routes images to image-cache', () => {
    expect(getCacheNameForRequest('https://example.com/photo.webp')).toBe('image-cache');
  });

  it('routes other requests to static-cache', () => {
    expect(getCacheNameForRequest('https://example.com/app.js')).toBe('static-cache');
  });
});
```

### Integration Testing with Playwright

Playwright can control service worker registration and intercept network requests:

```javascript
// tests/pwa.spec.js
import { test, expect } from '@playwright/test';

test.describe('PWA', () => {
  test('registers a service worker', async ({ page }) => {
    await page.goto('/');

    // Wait for the SW to be registered and active
    const swState = await page.evaluate(async () => {
      const reg = await navigator.serviceWorker.ready;
      return reg.active?.state;
    });

    expect(swState).toBe('activated');
  });

  test('serves cached content when offline', async ({ page, context }) => {
    // Load the page online to populate the cache
    await page.goto('/');
    await page.waitForLoadState('networkidle');

    // Go offline
    await context.setOffline(true);

    // Navigate — should serve from cache
    await page.goto('/');
    await expect(page.locator('h1')).toBeVisible();

    // Restore network
    await context.setOffline(false);
  });

  test('shows offline fallback for uncached pages', async ({ page, context }) => {
    await page.goto('/');
    await page.waitForLoadState('networkidle');

    await context.setOffline(true);

    // Navigate to an uncached page
    await page.goto('/never-visited-page');
    await expect(page.locator('text=You are offline')).toBeVisible();

    await context.setOffline(false);
  });
});
```

### Testing with Workbox's Test Utilities

For Workbox-based service workers, use the `workbox-window` package to observe lifecycle events:

```javascript
import { Workbox } from 'workbox-window';

const wb = new Workbox('/sw.js');

wb.addEventListener('installed', (event) => {
  if (event.isUpdate) {
    console.log('New version installed');
  }
});

wb.addEventListener('activated', (event) => {
  console.log('Service worker activated');
});

wb.addEventListener('waiting', (event) => {
  console.log('New version waiting to activate');
});

await wb.register();
```

---

## Manifest Validation

Verify your manifest before deployment:

### Automated Validation

```bash
# Lighthouse checks manifest validity
npx lighthouse https://localhost:3000 --only-categories=pwa --chrome-flags="--ignore-certificate-errors"
```

### Manual Checklist

| Check | Requirement |
|-------|-------------|
| `name` or `short_name` present | Required for install prompt |
| `start_url` set | Required — the entry point of the installed app |
| `display` is `standalone`, `fullscreen`, or `minimal-ui` | `browser` mode prevents installation |
| Icons include 192×192 and 512×512 | Required for install prompt and splash screen |
| At least one maskable icon | Recommended for adaptive icon shapes |
| `theme_color` matches `<meta name="theme-color">` | Consistent status bar tint |
| `background_color` set | Used during splash screen before CSS loads |
| All icon files are reachable | 404 on an icon URL fails the audit |
| `start_url` is within `scope` | Navigation outside scope opens in the browser |

### Programmatic Validation

```javascript
async function validateManifest() {
  const response = await fetch('/manifest.webmanifest');
  const manifest = await response.json();
  const errors = [];

  if (!manifest.name && !manifest.short_name) {
    errors.push('Missing name or short_name');
  }
  if (!manifest.start_url) {
    errors.push('Missing start_url');
  }
  if (!['standalone', 'fullscreen', 'minimal-ui'].includes(manifest.display)) {
    errors.push(`Invalid display mode: ${manifest.display}`);
  }
  const icons = manifest.icons || [];
  const sizes = icons.map((i) => i.sizes);
  if (!sizes.some((s) => s?.includes('192x192'))) {
    errors.push('Missing 192x192 icon');
  }
  if (!sizes.some((s) => s?.includes('512x512'))) {
    errors.push('Missing 512x512 icon');
  }

  return errors;
}
```

---

## Lighthouse CI

Integrate Lighthouse audits into your CI pipeline to catch regressions before deployment.

### Installation and Configuration

```bash
npm install -D @lhci/cli
```

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: ['http://localhost:3000/', 'http://localhost:3000/about'],
      startServerCommand: 'npm run preview',
      numberOfRuns: 3,
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'categories:accessibility': ['error', { minScore: 0.9 }],
        'categories:best-practices': ['error', { minScore: 0.9 }],
        'categories:pwa': ['warn', { minScore: 0.9 }],
        'service-worker': 'error',
        'installable-manifest': 'error',
        'splash-screen': 'warn',
        'themed-omnibox': 'warn',
      },
    },
    upload: {
      target: 'temporary-public-storage',
    },
  },
};
```

### Running in CI (GitHub Actions Example)

```yaml
name: Lighthouse CI
on: [push]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build
      - run: npx @lhci/cli autorun
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
```

---

## HTTPS Requirements

PWAs require HTTPS without exception (except `localhost` for development).

| Requirement | Detail |
|-------------|--------|
| Service worker registration | Blocked on non-secure origins |
| Push API | Requires secure origin |
| Cache API | Available only on secure origins |
| Geolocation, camera, mic | Require secure origin |
| Install prompt | Only shown on HTTPS pages |

### Local Development

For development with HTTPS:

```bash
# Generate self-signed cert for local dev
mkcert -install
mkcert localhost 127.0.0.1

# Use with a dev server (Vite example)
# vite.config.js
import fs from 'fs';

export default {
  server: {
    https: {
      key: fs.readFileSync('./localhost-key.pem'),
      cert: fs.readFileSync('./localhost.pem'),
    },
  },
};
```

---

## CDN Deployment

### Service Worker Caching Headers

The service worker file itself must not be aggressively cached by the CDN or browser HTTP cache:

```nginx
# Nginx — no-cache for the service worker file
location = /sw.js {
    add_header Cache-Control "no-cache, no-store, must-revalidate";
    add_header Pragma "no-cache";
    expires 0;
}

# Long cache for hashed static assets
location /assets/ {
    add_header Cache-Control "public, max-age=31536000, immutable";
}
```

### Deployment Checklist

| Step | Why |
|------|-----|
| Set `Cache-Control: no-cache` on `sw.js` | Ensures browsers check for updates on every visit |
| Set `Cache-Control: immutable` on hashed assets | Maximises CDN cache hit rate |
| Set `Cache-Control: no-cache` on `manifest.webmanifest` | Ensures manifest changes propagate quickly |
| Verify HTTPS redirect (HTTP → HTTPS) | Required for all PWA features |
| Set `Service-Worker-Allowed: /` if sw.js is in a subdirectory | Expands service worker scope |
| Test with CDN's cache purge | Verify the new SW is served after deployment |

---

## Update Strategies

Managing service worker updates in production is the most operationally critical PWA concern. A bad update can leave users stuck on a broken version.

### Strategy Comparison

| Strategy | User Experience | Risk |
|----------|----------------|------|
| **Prompt to reload** | Banner: "New version available. Reload?" | Low — user controls timing |
| **Skip waiting** | Auto-activate new SW immediately | Medium — mid-session asset mismatch |
| **Navigation preload** | New SW serves fresh HTML on next navigation | Low — clean transition |
| **Versioned precache (Workbox)** | Workbox handles versioning and cleanup automatically | Low — proven in production |

### Recommended: Prompt with Workbox-Window

```javascript
import { Workbox } from 'workbox-window';

const wb = new Workbox('/sw.js');

wb.addEventListener('waiting', () => {
  const shouldUpdate = confirm('A new version is available. Reload to update?');
  if (shouldUpdate) {
    wb.messageSkipWaiting();
  }
});

wb.addEventListener('controlling', () => {
  window.location.reload();
});

wb.register();
```

### Emergency Kill Switch

If a broken service worker reaches production, you need a way to unregister it:

```javascript
// kill-sw.js — serve this as sw.js to unregister
self.addEventListener('install', () => self.skipWaiting());
self.addEventListener('activate', async () => {
  // Clear all caches
  const cacheNames = await caches.keys();
  await Promise.all(cacheNames.map((name) => caches.delete(name)));

  // Unregister self
  await self.registration.unregister();

  // Reload all clients
  const clients = await self.clients.matchAll();
  clients.forEach((client) => client.navigate(client.url));
});
```

Deploy this as your `sw.js` file, and the broken service worker will be replaced by one that immediately self-destructs and clears all caches.

### Clear-Site-Data Header

For a server-side nuclear option, send this header to wipe everything:

```
Clear-Site-Data: "cache", "storage"
```

This clears the HTTP cache, Cache API storage, IndexedDB, localStorage, and sessionStorage for the origin. Use as a last resort.

---

## Workbox Build Integration

### Build Modes

| Mode | When to Use |
|------|-------------|
| `generateSW` | Workbox writes the entire SW — no custom logic needed |
| `injectManifest` | You write the SW, Workbox injects the precache manifest |

### Vite + Workbox (vite-plugin-pwa)

```bash
npm install -D vite-plugin-pwa
```

```javascript
// vite.config.js
import { VitePWA } from 'vite-plugin-pwa';

export default {
  plugins: [
    VitePWA({
      registerType: 'prompt',
      includeAssets: ['favicon.ico', 'robots.txt', 'icons/*.png'],
      manifest: {
        name: 'My PWA',
        short_name: 'MyPWA',
        start_url: '/',
        display: 'standalone',
        background_color: '#ffffff',
        theme_color: '#1a73e8',
        icons: [
          { src: '/icons/icon-192.png', sizes: '192x192', type: 'image/png' },
          { src: '/icons/icon-512.png', sizes: '512x512', type: 'image/png' },
          { src: '/icons/icon-512-maskable.png', sizes: '512x512', type: 'image/png', purpose: 'maskable' },
        ],
      },
      workbox: {
        globPatterns: ['**/*.{js,css,html,png,svg,woff2}'],
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/api\.example\.com\/.*/i,
            handler: 'NetworkFirst',
            options: {
              cacheName: 'api-cache',
              expiration: { maxEntries: 50, maxAgeSeconds: 300 },
            },
          },
        ],
      },
    }),
  ],
};
```

### Webpack + Workbox

```javascript
// webpack.config.js
const { InjectManifest } = require('workbox-webpack-plugin');

module.exports = {
  plugins: [
    new InjectManifest({
      swSrc: './src/sw.js',
      swDest: 'sw.js',
      maximumFileSizeToCacheInBytes: 5 * 1024 * 1024,
    }),
  ],
};
```

---

## Key Takeaways

- Extract caching strategy logic into pure functions for unit testing; use Playwright's `context.setOffline()` for integration tests that verify offline behaviour.
- Validate the web app manifest programmatically and via Lighthouse — a single missing field or unreachable icon can prevent installation.
- Lighthouse CI in your pipeline catches PWA, performance, and accessibility regressions before they reach production.
- Set `Cache-Control: no-cache` on `sw.js` and `manifest.webmanifest` so updates propagate immediately; use `immutable` only for content-hashed assets.
- Always have an emergency kill switch — a self-destructing service worker that clears all caches and unregisters itself in case a broken version reaches production.
- Use Workbox's build integration (via vite-plugin-pwa or workbox-webpack-plugin) to automate precache manifest generation and runtime caching configuration.
