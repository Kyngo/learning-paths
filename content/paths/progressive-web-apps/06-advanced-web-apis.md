---
title: "Advanced Web APIs"
weight: 6
---

## Project Fugu and the Capabilities Gap

**Project Fugu** is a cross-company effort (Google, Microsoft, Intel, Samsung) to bring native-level capabilities to the web platform while preserving security and user privacy. Named after the Japanese puffer fish — delicious if prepared correctly, lethal if not — the project acknowledges that exposing powerful APIs must be done carefully.

The Fugu tracker at [fugu-tracker.web.app](https://fugu-tracker.web.app) catalogues the status of dozens of APIs. This section covers the most impactful ones for PWA development.

### Capability Check Pattern

Always feature-detect before using any advanced API:

```javascript
if ('share' in navigator) {
  // Web Share API available
} else {
  // Fallback: copy-to-clipboard or manual sharing
}
```

---

## Web Share API

Invokes the native share sheet on mobile (and on desktop where supported), letting users share content to any installed app.

```javascript
async function shareContent() {
  if (!navigator.share) {
    // Fallback: copy link to clipboard
    await navigator.clipboard.writeText(window.location.href);
    showToast('Link copied!');
    return;
  }

  try {
    await navigator.share({
      title: 'Check this article',
      text: 'An excellent guide to PWA development',
      url: 'https://example.com/articles/pwa-guide',
    });
    console.log('Shared successfully');
  } catch (error) {
    if (error.name !== 'AbortError') {
      console.error('Share failed:', error);
    }
  }
}
```

### Sharing Files

```javascript
async function shareImage(imageBlob) {
  const file = new File([imageBlob], 'photo.png', { type: 'image/png' });

  if (navigator.canShare?.({ files: [file] })) {
    await navigator.share({
      title: 'My Photo',
      files: [file],
    });
  }
}
```

| Constraint | Detail |
|-----------|--------|
| User gesture required | Must be triggered by click/tap, not programmatically |
| HTTPS only | Secure origin required |
| File types | Limited to common types (images, text, audio, video, PDF) |

---

## Share Target API

The inverse of Web Share — registers your PWA as a **target** for shares from other apps. Configured in the manifest:

```json
{
  "share_target": {
    "action": "/share-handler",
    "method": "POST",
    "enctype": "multipart/form-data",
    "params": {
      "title": "shared-title",
      "text": "shared-text",
      "url": "shared-url",
      "files": [
        {
          "name": "media",
          "accept": ["image/*", "video/*"]
        }
      ]
    }
  }
}
```

Handle the shared data in a service worker or server-side route:

```javascript
// sw.js — intercept the share POST
self.addEventListener('fetch', (event) => {
  if (event.request.url.endsWith('/share-handler') && event.request.method === 'POST') {
    event.respondWith(
      (async () => {
        const formData = await event.request.formData();
        const title = formData.get('shared-title');
        const text = formData.get('shared-text');
        const url = formData.get('shared-url');
        const files = formData.getAll('media');

        // Store in IndexedDB for the page to pick up
        const db = await openDB('shares', 1);
        await db.put('incoming', { title, text, url, files, timestamp: Date.now() });

        // Redirect to the app
        return Response.redirect('/shared', 303);
      })()
    );
  }
});
```

---

## File System Access API

Read and write to the user's local file system with explicit permission grants. Useful for document editors, IDEs, and media applications.

### Opening a File

```javascript
async function openFile() {
  const [fileHandle] = await window.showOpenFilePicker({
    types: [
      {
        description: 'Text Files',
        accept: { 'text/plain': ['.txt', '.md'] },
      },
    ],
    multiple: false,
  });

  const file = await fileHandle.getFile();
  const contents = await file.text();
  return { fileHandle, contents };
}
```

### Saving a File

```javascript
async function saveFile(fileHandle, contents) {
  // If no handle, prompt for a new file
  if (!fileHandle) {
    fileHandle = await window.showSaveFilePicker({
      suggestedName: 'document.txt',
      types: [{ description: 'Text', accept: { 'text/plain': ['.txt'] } }],
    });
  }

  const writable = await fileHandle.createWritable();
  await writable.write(contents);
  await writable.close();
  return fileHandle;
}
```

### Watching a Directory

```javascript
async function openDirectory() {
  const dirHandle = await window.showDirectoryPicker();

  for await (const [name, handle] of dirHandle) {
    console.log(`${handle.kind}: ${name}`);
    // handle.kind is 'file' or 'directory'
  }
}
```

| Security | Detail |
|----------|--------|
| Permission prompt | User must explicitly select files/directories |
| Persistence | Permissions can be persisted via `queryPermission()` / `requestPermission()` |
| Scope | Only operates on user-selected paths, no arbitrary file system access |

---

## Badging API

Sets a badge on the app icon (taskbar, dock, or home screen). Useful for unread counts and status indicators.

```javascript
// Set a numeric badge
navigator.setAppBadge(5); // Shows "5" on the app icon

// Set a flag badge (no number, just a dot)
navigator.setAppBadge();

// Clear the badge
navigator.clearAppBadge();
```

The badge persists even when the app is not running. Service workers can set badges in response to push events:

```javascript
// sw.js
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? {};
  event.waitUntil(
    Promise.all([
      self.registration.showNotification(data.title, { body: data.body }),
      navigator.setAppBadge(data.unreadCount || 1),
    ])
  );
});
```

---

## Web Bluetooth API

Connect to Bluetooth Low Energy (BLE) devices from the browser. Requires user gesture and explicit device selection.

```javascript
async function connectToHeartRateMonitor() {
  const device = await navigator.bluetooth.requestDevice({
    filters: [{ services: ['heart_rate'] }],
    optionalServices: ['battery_service'],
  });

  const server = await device.gatt.connect();
  const service = await server.getPrimaryService('heart_rate');
  const characteristic = await service.getCharacteristic('heart_rate_measurement');

  characteristic.addEventListener('characteristicvaluechanged', (event) => {
    const value = event.target.value;
    const heartRate = value.getUint8(1);
    console.log('Heart rate:', heartRate);
  });

  await characteristic.startNotifications();
}
```

| Constraint | Detail |
|-----------|--------|
| User gesture | `requestDevice()` must be triggered by a click |
| Pairing | Browser shows a device picker — no silent scanning |
| Protocol | BLE (Bluetooth Low Energy) only, not classic Bluetooth |
| Platform | Chrome on Android, macOS, Windows, Linux. Not iOS Safari. |

---

## Web USB API

Communicate directly with USB devices. Useful for firmware updaters, 3D printers, hardware configuration tools.

```javascript
async function connectToDevice() {
  const device = await navigator.usb.requestDevice({
    filters: [{ vendorId: 0x2341 }], // Arduino
  });

  await device.open();
  await device.selectConfiguration(1);
  await device.claimInterface(0);

  // Send data
  const encoder = new TextEncoder();
  await device.transferOut(1, encoder.encode('LED_ON\n'));

  // Receive data
  const result = await device.transferIn(1, 64);
  const decoder = new TextDecoder();
  console.log('Response:', decoder.decode(result.data));
}
```

---

## Periodic Background Sync

Runs a service worker event at periodic intervals (minimum interval controlled by the browser based on site engagement). Useful for refreshing content in the background.

```javascript
// Register periodic sync
const registration = await navigator.serviceWorker.ready;
const status = await navigator.permissions.query({ name: 'periodic-background-sync' });

if (status.state === 'granted') {
  await registration.periodicSync.register('update-articles', {
    minInterval: 24 * 60 * 60 * 1000, // Once per day minimum
  });
}
```

Handle in the service worker:

```javascript
self.addEventListener('periodicsync', (event) => {
  if (event.tag === 'update-articles') {
    event.waitUntil(updateArticlesCache());
  }
});

async function updateArticlesCache() {
  const response = await fetch('/api/articles/latest');
  const articles = await response.json();

  const db = await openDB('my-app', 1);
  for (const article of articles) {
    await db.put('articles', article);
  }
}
```

| Constraint | Detail |
|-----------|--------|
| Minimum interval | Browser decides actual frequency based on engagement |
| Site engagement | Only works for sites the user visits frequently |
| No guarantee | Browser may delay or skip syncs to preserve battery |
| Chrome-only | Not supported in Firefox or Safari as of 2025 |

---

## API Support Summary

| API | Chrome | Firefox | Safari | Edge |
|-----|--------|---------|--------|------|
| Web Share | ✅ | ✅ (Android) | ✅ | ✅ |
| Share Target | ✅ | ❌ | ❌ | ✅ |
| File System Access | ✅ | ❌ | ❌ | ✅ |
| Badging | ✅ | ❌ | ✅ (17.4+) | ✅ |
| Web Bluetooth | ✅ | ❌ | ❌ | ✅ |
| Web USB | ✅ | ❌ | ❌ | ✅ |
| Periodic Background Sync | ✅ | ❌ | ❌ | ✅ |

**Always feature-detect.** Never assume an API is available based on the browser name.

---

## Key Takeaways

- Project Fugu is systematically closing the capability gap between web and native, but adoption varies widely across browsers — feature detection is mandatory.
- The Web Share API and Share Target API turn your PWA into a first-class participant in the OS sharing ecosystem, both as a sender and receiver.
- The File System Access API enables local file read/write with explicit user permission — powerful for editors and productivity apps, but Chromium-only.
- Hardware APIs (Bluetooth, USB) require user gestures and device pickers for security — no silent device access is possible.
- Periodic Background Sync enables content freshness without the user opening the app, but the browser controls timing based on site engagement.
- Always provide graceful fallbacks for unsupported APIs — a PWA must remain functional in every browser, even if some capabilities are unavailable.
