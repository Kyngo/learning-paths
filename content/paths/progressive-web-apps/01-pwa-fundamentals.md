---
title: "PWA Fundamentals"
weight: 1
---

## What Makes a Progressive Web App

A Progressive Web App is a web application that uses modern browser APIs and a strategic design approach to deliver a native-app-like experience. The term was coined by Alex Russell and Frances Berriman in 2015, but the underlying principles are rooted in the web's core strengths: linkability, discoverability, and universal access.

A PWA is not a framework or a specific technology. It is a set of characteristics:

| Characteristic | Meaning |
|---------------|---------|
| **Progressive** | Works for every user regardless of browser, using progressive enhancement |
| **Responsive** | Adapts to any form factor — phone, tablet, desktop |
| **Connectivity-independent** | Functions offline or on unreliable networks via service workers |
| **App-like** | Feels like a native app with app-shell architecture and smooth interactions |
| **Fresh** | Always up to date thanks to service worker update mechanisms |
| **Safe** | Served over HTTPS to prevent snooping and content tampering |
| **Discoverable** | Identifiable as an "application" by search engines via the web app manifest |
| **Re-engageable** | Supports push notifications to bring users back |
| **Installable** | Users can add it to their home screen without an app store |
| **Linkable** | Shareable via URL with zero installation friction |

### The Three Technical Pillars

Every PWA rests on three technical requirements:

1. **HTTPS** — the entire origin must be served over a secure connection (localhost is exempted for development).
2. **A web app manifest** — a JSON file that describes the application to the browser.
3. **A service worker** — a JavaScript file that acts as a programmable network proxy.

---

## The Web App Manifest

The manifest is a JSON file (conventionally `manifest.json` or `manifest.webmanifest`) linked from every HTML page:

```html
<link rel="manifest" href="/manifest.webmanifest">
```

### Minimal Manifest

```json
{
  "name": "My Progressive App",
  "short_name": "MyApp",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#1a73e8",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

### Key Manifest Fields

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | Yes | Full application name shown during install |
| `short_name` | Recommended | Displayed on the home screen (max ~12 characters) |
| `start_url` | Yes | URL opened when the user launches the installed app |
| `display` | Yes | `standalone`, `fullscreen`, `minimal-ui`, or `browser` |
| `background_color` | Recommended | Splash screen background while the app loads |
| `theme_color` | Recommended | Browser toolbar colour / OS status bar tint |
| `icons` | Yes | Array of icon objects; at minimum 192×192 and 512×512 |
| `scope` | Optional | URL scope that constrains navigation within the app |
| `description` | Optional | Used by app stores and install prompts |
| `screenshots` | Optional | Rich install UI on Android (richer install dialog) |

### Display Modes

```text
fullscreen   → No browser UI at all. Entire screen.
standalone   → Looks like a native app. No URL bar.
minimal-ui   → Small navigation controls visible.
browser      → Standard browser tab. Not installable.
```

You can query the current display mode in CSS:

```css
@media (display-mode: standalone) {
  /* Styles applied only when running as installed PWA */
  .install-banner { display: none; }
}
```

---

## Installability Criteria

Browsers use a set of heuristics to decide whether a web app qualifies for the install prompt. Chrome's current criteria:

1. The page is served over **HTTPS**.
2. A valid **web app manifest** is present with `name` or `short_name`, `start_url`, `display` set to `standalone`/`fullscreen`/`minimal-ui`, and at least one icon ≥ 192×192.
3. A **service worker** is registered with a `fetch` event handler.
4. The user has engaged with the page (interacted at least once).

### Controlling the Install Prompt

Browsers fire a `beforeinstallprompt` event that you can intercept:

```javascript
let deferredPrompt;

window.addEventListener('beforeinstallprompt', (e) => {
  // Prevent the default mini-infobar
  e.preventDefault();
  deferredPrompt = e;

  // Show your custom install button
  document.getElementById('install-btn').style.display = 'block';
});

document.getElementById('install-btn').addEventListener('click', async () => {
  if (!deferredPrompt) return;

  deferredPrompt.prompt();
  const { outcome } = await deferredPrompt.userChoice;
  console.log(`User ${outcome === 'accepted' ? 'accepted' : 'dismissed'} install`);
  deferredPrompt = null;
});
```

Detecting whether the app is already installed:

```javascript
window.addEventListener('appinstalled', () => {
  console.log('PWA installed successfully');
  // Hide the install button, log analytics event
  document.getElementById('install-btn').style.display = 'none';
});
```

---

## App Shell Architecture

The app shell model separates the **minimal UI skeleton** (HTML, CSS, core JavaScript) from the **dynamic content**. The shell is cached aggressively on first visit so subsequent loads are instant — even offline.

```text
┌──────────────────────────────────────┐
│           App Shell (cached)         │
│  ┌──────────────────────────────┐    │
│  │   Header / Navigation        │    │
│  ├──────────────────────────────┤    │
│  │                              │    │
│  │   Content Area               │    │ ← Loaded dynamically
│  │   (fetched from network      │    │   from API / cache
│  │    or cache at runtime)      │    │
│  │                              │    │
│  ├──────────────────────────────┤    │
│  │   Footer / Tab Bar           │    │
│  └──────────────────────────────┘    │
└──────────────────────────────────────┘
```

### Implementing an App Shell

In your service worker install event, pre-cache the shell resources:

```javascript
const SHELL_CACHE = 'app-shell-v1';
const SHELL_FILES = [
  '/',
  '/index.html',
  '/styles/app.css',
  '/scripts/app.js',
  '/icons/icon-192.png',
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(SHELL_CACHE).then((cache) => cache.addAll(SHELL_FILES))
  );
});
```

Then serve the shell from cache in the fetch handler:

```javascript
self.addEventListener('fetch', (event) => {
  if (event.request.mode === 'navigate') {
    event.respondWith(
      caches.match('/index.html').then((response) => response || fetch(event.request))
    );
    return;
  }

  event.respondWith(
    caches.match(event.request).then((response) => response || fetch(event.request))
  );
});
```

---

## Lighthouse PWA Audit

Google's Lighthouse tool audits PWA compliance. Run it from Chrome DevTools (Lighthouse panel) or the command line:

```bash
npx lighthouse https://example.com --only-categories=pwa --output=json
```

### Key Audit Checks

| Audit | What It Verifies |
|-------|-----------------|
| Installable | Valid manifest, service worker with fetch handler, HTTPS |
| Splash screen | `background_color`, `theme_color`, `name`, 512×512 icon |
| Themed address bar | `theme_color` in manifest and `<meta name="theme-color">` |
| Content sized correctly | Viewport meta tag present, content fits without horizontal scroll |
| Redirects HTTP → HTTPS | All HTTP requests redirect to HTTPS |
| Offline fallback | Service worker returns a 200 when offline |
| Maskable icon | At least one icon with `"purpose": "maskable"` for adaptive display |

### Maskable Icons

Android adaptive icons require a "safe zone" — the inner 80% of the image. Add a maskable icon to your manifest:

```json
{
  "src": "/icons/icon-maskable-512.png",
  "sizes": "512x512",
  "type": "image/png",
  "purpose": "maskable"
}
```

Use [maskable.app](https://maskable.app) to preview how your icon looks in different shapes.

---

## PWA vs Native vs Hybrid

| Dimension | PWA | Native | Hybrid (Capacitor/Cordova) |
|-----------|-----|--------|---------------------------|
| Distribution | URL, no app store required | App Store / Play Store | App Store / Play Store |
| Installation | Browser prompt, instant | Download + install | Download + install |
| Update mechanism | Service worker, instant | Store review + user update | Store review + user update |
| Offline support | Service worker + Cache API | Full device API access | Plugin-dependent |
| Hardware access | Limited (camera, geolocation, sensors) | Full (Bluetooth, NFC, file system) | Plugin-dependent |
| Performance | Near-native with good patterns | Best possible | Worse than native |
| Development cost | Single codebase (web) | Per-platform codebase | Single codebase + plugins |
| Discoverability | Search engines + URL sharing | App store search | App store search |
| Push notifications | Supported (not on iOS Safari before 16.4) | Full support | Full support |
| Storage quota | Browser-managed, can be evicted | Unlimited | Plugin-dependent |

### When to Choose a PWA

- Your audience is primarily on the web and you want zero-friction access.
- You need fast iteration cycles without app store review delays.
- Your functionality can be achieved with available web APIs.
- You want a single codebase serving desktop and mobile.

### When Native Is Still Necessary

- You need deep hardware integration (Bluetooth LE peripherals, ARKit/ARCore, NFC writes).
- Your app is performance-critical (3D games, real-time video processing).
- You require persistent background execution beyond what the web platform allows.
- Your business model depends on app store visibility and in-app purchases.

---

## Key Takeaways

- A PWA is defined by characteristics (reliable, fast, engaging), not by a specific framework or library — any web app can become progressive incrementally.
- The three technical pillars are HTTPS, a web app manifest, and a service worker with a fetch event handler.
- The web app manifest controls how the installed app appears: its name, icon, splash screen, display mode, and theme colour.
- App shell architecture separates the cacheable UI skeleton from dynamic content, enabling instant loads on repeat visits and offline access.
- Lighthouse provides automated auditing of PWA compliance — run it early and often during development.
- PWAs trade some native capabilities for zero-friction distribution, instant updates, and a single codebase — the right choice depends on your audience and required device APIs.
