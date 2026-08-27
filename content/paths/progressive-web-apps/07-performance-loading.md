---
title: "Performance & Loading"
weight: 7
---

## Why Performance Matters for PWAs

A PWA that loads slowly on first visit will never become an installed app. Performance is not a separate concern — it is a prerequisite for every other PWA capability. Users on mobile networks with 3G-equivalent speeds will abandon a page that takes more than 3 seconds to become interactive. Google uses Core Web Vitals as a ranking signal, directly linking performance to discoverability.

---

## Core Web Vitals

Google's Core Web Vitals are three metrics that measure real-user experience:

| Metric | Full Name | What It Measures | Good | Needs Improvement | Poor |
|--------|-----------|-----------------|------|-------------------|------|
| **LCP** | Largest Contentful Paint | Time until the largest visible element renders | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| **INP** | Interaction to Next Paint | Responsiveness — time from user input to visual update | ≤ 200ms | ≤ 500ms | > 500ms |
| **CLS** | Cumulative Layout Shift | Visual stability — how much content shifts unexpectedly | ≤ 0.1 | ≤ 0.25 | > 0.25 |

### Measuring Core Web Vitals

```javascript
import { onLCP, onINP, onCLS } from 'web-vitals';

onLCP((metric) => {
  console.log('LCP:', metric.value, 'ms');
  sendToAnalytics('LCP', metric);
});

onINP((metric) => {
  console.log('INP:', metric.value, 'ms');
  sendToAnalytics('INP', metric);
});

onCLS((metric) => {
  console.log('CLS:', metric.value);
  sendToAnalytics('CLS', metric);
});

function sendToAnalytics(name, metric) {
  navigator.sendBeacon('/api/analytics', JSON.stringify({
    name,
    value: metric.value,
    id: metric.id,
    navigationType: metric.navigationType,
  }));
}
```

---

## Optimising LCP

The largest contentful paint is usually a hero image, heading, or large text block. Common causes of poor LCP:

| Cause | Fix |
|-------|-----|
| Slow server response (TTFB > 800ms) | CDN, edge caching, server-side caching |
| Render-blocking CSS/JS | Inline critical CSS, defer non-critical scripts |
| Large unoptimised images | Compress, use modern formats (WebP, AVIF), responsive sizes |
| Web fonts blocking render | `font-display: swap`, preload key fonts |
| Client-side rendering delay | Pre-render or SSR the initial HTML |

### Preloading the LCP Image

```html
<head>
  <!-- Tell the browser to fetch the hero image immediately -->
  <link rel="preload" as="image" href="/images/hero.webp"
        fetchpriority="high" type="image/webp">
</head>
```

### Responsive Images

```html
<img
  src="/images/hero-800.webp"
  srcset="
    /images/hero-400.webp 400w,
    /images/hero-800.webp 800w,
    /images/hero-1200.webp 1200w
  "
  sizes="(max-width: 600px) 100vw, 800px"
  alt="Hero banner"
  width="800"
  height="400"
  loading="eager"
  fetchpriority="high"
>
```

---

## Optimising INP

INP measures the worst-case responsiveness across all interactions during a page visit. Poor INP means the main thread is blocked during user interactions.

| Cause | Fix |
|-------|-----|
| Long JavaScript tasks (> 50ms) | Break into smaller tasks using `scheduler.yield()` |
| Heavy event handlers | Debounce, defer non-visual work |
| Layout thrashing | Batch DOM reads before writes |
| Excessive re-renders | Virtualise long lists, memoise components |

### Breaking Up Long Tasks

```javascript
// Before: one long blocking task
function processLargeList(items) {
  for (const item of items) {
    expensiveOperation(item); // Blocks for 200ms total
  }
}

// After: yield to the browser between chunks
async function processLargeList(items) {
  const chunkSize = 50;
  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);
    for (const item of chunk) {
      expensiveOperation(item);
    }
    // Yield control so the browser can handle input/rendering
    await scheduler.yield();
  }
}
```

If `scheduler.yield()` is not available, use a fallback:

```javascript
function yieldToMain() {
  return new Promise((resolve) => setTimeout(resolve, 0));
}
```

---

## Optimising CLS

Layout shifts occur when visible elements move after the initial render. Common offenders:

| Cause | Fix |
|-------|-----|
| Images without dimensions | Always set `width` and `height` attributes |
| Ads or embeds injected later | Reserve space with CSS `aspect-ratio` or min-height |
| Web fonts causing reflow | Use `font-display: swap` + `size-adjust` |
| Dynamic content inserted above viewport | Insert below the fold or use CSS `content-visibility` |

### Reserving Space for Dynamic Content

```css
/* Reserve space for an ad slot */
.ad-slot {
  min-height: 250px;
  aspect-ratio: 300 / 250;
  background: #f0f0f0;
}

/* Prevent font swap layout shift */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap;
  size-adjust: 105%; /* Match fallback font metrics */
}
```

---

## Code Splitting and Lazy Loading

### Dynamic Imports

Split code at route or feature boundaries so the browser downloads only what is needed:

```javascript
// Instead of importing everything upfront
// import { heavyChart } from './charts.js';

// Load on demand
async function showChart(data) {
  const { heavyChart } = await import('./charts.js');
  heavyChart.render(data);
}
```

### Lazy Loading Images

```html
<!-- Browser-native lazy loading -->
<img src="/images/product.webp" alt="Product" loading="lazy" width="400" height="300">
```

For below-the-fold images, `loading="lazy"` defers the network request until the image is near the viewport. **Never** lazy-load the LCP image.

### Lazy Loading Routes (Framework Example)

```javascript
// Vite / webpack — route-based code splitting
const routes = [
  { path: '/', component: () => import('./pages/Home.js') },
  { path: '/dashboard', component: () => import('./pages/Dashboard.js') },
  { path: '/settings', component: () => import('./pages/Settings.js') },
];
```

---

## Resource Hints

Tell the browser what to prioritise before it discovers resources naturally:

| Hint | Purpose | Example |
|------|---------|---------|
| `preload` | Fetch this resource now — it is needed for the current page | Critical fonts, hero images |
| `prefetch` | Fetch this resource at low priority — it may be needed for the next navigation | Next-page JS bundle |
| `preconnect` | Establish connection early (DNS + TCP + TLS) | Third-party API origins |
| `dns-prefetch` | Resolve DNS only | Analytics domains |
| `modulepreload` | Preload an ES module and its dependencies | Critical JS modules |

```html
<head>
  <!-- Preconnect to API origin -->
  <link rel="preconnect" href="https://api.example.com">

  <!-- Preload critical font -->
  <link rel="preload" as="font" type="font/woff2"
        href="/fonts/main.woff2" crossorigin>

  <!-- Prefetch next page's bundle -->
  <link rel="prefetch" href="/js/dashboard.js">

  <!-- Preload critical module -->
  <link rel="modulepreload" href="/js/app.js">
</head>
```

---

## The PRPL Pattern

PRPL is a loading architecture designed for PWAs:

| Letter | Action | How |
|--------|--------|-----|
| **P** | **Push** critical resources for the initial route | HTTP/2 server push or `<link rel="preload">` |
| **R** | **Render** the initial route as quickly as possible | Inline critical CSS, stream HTML |
| **P** | **Pre-cache** remaining routes | Service worker pre-caches other route shells |
| **L** | **Lazy-load** remaining routes on demand | Dynamic imports triggered by navigation |

### Implementation Sketch

```javascript
// 1. Service worker pre-caches route shells during install
const ROUTE_SHELLS = [
  '/shell/home.html',
  '/shell/dashboard.html',
  '/shell/settings.html',
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('route-shells-v1').then((cache) => cache.addAll(ROUTE_SHELLS))
  );
});

// 2. Navigations serve pre-cached shells instantly
self.addEventListener('fetch', (event) => {
  if (event.request.mode === 'navigate') {
    const url = new URL(event.request.url);
    const shellPath = `/shell${url.pathname === '/' ? '/home' : url.pathname}.html`;

    event.respondWith(
      caches.match(shellPath).then((cached) => cached || fetch(event.request))
    );
  }
});
```

---

## Performance Budgets

Set explicit limits on resource sizes and loading times. Fail the build if budgets are exceeded.

### Budget Configuration (Lighthouse CI)

```json
{
  "ci": {
    "assert": {
      "assertions": {
        "resource-summary:script:size": ["error", { "maxNumericValue": 300000 }],
        "resource-summary:stylesheet:size": ["error", { "maxNumericValue": 100000 }],
        "resource-summary:image:size": ["warn", { "maxNumericValue": 500000 }],
        "resource-summary:total:size": ["error", { "maxNumericValue": 1000000 }],
        "first-contentful-paint": ["error", { "maxNumericValue": 2000 }],
        "interactive": ["error", { "maxNumericValue": 5000 }]
      }
    }
  }
}
```

### Webpack Bundle Analyser

```bash
npx webpack-bundle-analyzer dist/stats.json
```

### Custom Performance Observer

```javascript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.transferSize > 200_000) {
      console.warn(`Large resource: ${entry.name} (${(entry.transferSize / 1024).toFixed(1)} KB)`);
    }
  }
});

observer.observe({ type: 'resource', buffered: true });
```

---

## Key Takeaways

- Core Web Vitals (LCP, INP, CLS) are the three metrics that matter most — measure them with the `web-vitals` library and send data to your analytics.
- LCP is usually blocked by slow server response, render-blocking resources, or unoptimised images — preload the LCP element and use modern image formats.
- INP improves by breaking long JavaScript tasks into smaller chunks using `scheduler.yield()`, so the browser can respond to user input between them.
- CLS is prevented by always setting explicit dimensions on images and embeds, and reserving space for dynamically injected content.
- The PRPL pattern (Push, Render, Pre-cache, Lazy-load) is the canonical loading architecture for PWAs — pre-cache route shells and lazy-load everything else.
- Performance budgets make regressions visible by failing the build when resource sizes or timing thresholds are exceeded.
