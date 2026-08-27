---
title: "Offline-First Architecture"
weight: 4
---

## The Offline-First Mindset

Offline-first design treats the network as an enhancement rather than a requirement. Instead of building for connectivity and then patching in offline support, you design the application to work without a network from the start. The network, when available, synchronises local state with the server.

This approach matters because network reliability is a spectrum, not a binary. Users encounter:

| Scenario | Reality |
|----------|---------|
| Underground trains, lifts, aeroplanes | Zero connectivity |
| Conference centres, stadiums | Congested, unreliable |
| Rural areas, emerging markets | Slow, intermittent |
| Office Wi-Fi switching to mobile data | Brief disconnection during handoff |

An offline-first PWA handles all of these gracefully.

---

## Offline Detection

### The navigator.onLine Property

```javascript
if (navigator.onLine) {
  console.log('Browser thinks it is online');
} else {
  console.log('Browser is offline');
}
```

**Warning:** `navigator.onLine` only tells you whether the device has a network interface that appears connected. It does **not** guarantee actual internet reachability. A laptop connected to a hotel Wi-Fi captive portal reports `onLine: true` but cannot reach your API.

### Online/Offline Events

```javascript
window.addEventListener('online', () => {
  console.log('Network restored');
  syncPendingChanges();
});

window.addEventListener('offline', () => {
  console.log('Network lost');
  showOfflineBanner();
});
```

### Reliable Connectivity Check

For critical operations, verify actual reachability:

```javascript
async function isReachable(url = '/api/health') {
  try {
    const response = await fetch(url, {
      method: 'HEAD',
      cache: 'no-store',
      signal: AbortSignal.timeout(5000),
    });
    return response.ok;
  } catch {
    return false;
  }
}
```

---

## IndexedDB for Client-Side Storage

IndexedDB is the primary client-side database for structured data in PWAs. Unlike `localStorage` (5 MB, synchronous, strings only), IndexedDB supports large volumes, binary data, indexes, and transactions.

### Storage Comparison

| Storage | Capacity | Data Types | Async | Indexes | Use Case |
|---------|----------|-----------|-------|---------|----------|
| `localStorage` | ~5 MB | Strings only | ❌ | ❌ | Small key-value settings |
| `sessionStorage` | ~5 MB | Strings only | ❌ | ❌ | Tab-scoped temp data |
| IndexedDB | Quota-based (GBs) | Structured clones | ✅ | ✅ | Application data, offline cache |
| Cache API | Quota-based | Request/Response pairs | ✅ | ❌ | HTTP response caching |

### Opening a Database

```javascript
function openDB(name, version) {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open(name, version);

    request.onupgradeneeded = (event) => {
      const db = event.target.result;

      if (!db.objectStoreNames.contains('articles')) {
        const store = db.createObjectStore('articles', { keyPath: 'id' });
        store.createIndex('byDate', 'updatedAt');
        store.createIndex('byCategory', 'category');
      }

      if (!db.objectStoreNames.contains('pendingSync')) {
        db.createObjectStore('pendingSync', { keyPath: 'id', autoIncrement: true });
      }
    };

    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}
```

### CRUD Operations

```javascript
// Write
async function saveArticle(db, article) {
  return new Promise((resolve, reject) => {
    const tx = db.transaction('articles', 'readwrite');
    tx.objectStore('articles').put(article);
    tx.oncomplete = () => resolve();
    tx.onerror = () => reject(tx.error);
  });
}

// Read by key
async function getArticle(db, id) {
  return new Promise((resolve, reject) => {
    const tx = db.transaction('articles', 'readonly');
    const request = tx.objectStore('articles').get(id);
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}

// Read all by index
async function getArticlesByCategory(db, category) {
  return new Promise((resolve, reject) => {
    const tx = db.transaction('articles', 'readonly');
    const index = tx.objectStore('articles').index('byCategory');
    const request = index.getAll(category);
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}

// Delete
async function deleteArticle(db, id) {
  return new Promise((resolve, reject) => {
    const tx = db.transaction('articles', 'readwrite');
    tx.objectStore('articles').delete(id);
    tx.oncomplete = () => resolve();
    tx.onerror = () => reject(tx.error);
  });
}
```

### Using a Wrapper Library

The raw IndexedDB API is verbose. Production apps commonly use a thin wrapper like **idb**:

```javascript
import { openDB } from 'idb';

const db = await openDB('my-app', 1, {
  upgrade(db) {
    const store = db.createObjectStore('articles', { keyPath: 'id' });
    store.createIndex('byDate', 'updatedAt');
  },
});

// Write
await db.put('articles', { id: '1', title: 'Hello', updatedAt: Date.now() });

// Read
const article = await db.get('articles', '1');

// Query by index
const recent = await db.getAllFromIndex('articles', 'byDate');

// Delete
await db.delete('articles', '1');
```

---

## Background Sync API

The Background Sync API allows a service worker to defer actions until the user has a stable network connection. Instead of failing a POST request when offline, you queue the operation and let the browser retry it when connectivity is restored.

### Registering a Sync

```javascript
// In the page — queue a sync request
async function submitForm(data) {
  // Save data to IndexedDB first
  const db = await openDB('my-app', 1);
  await db.put('pendingSync', { ...data, createdAt: Date.now() });

  // Register a background sync
  const registration = await navigator.serviceWorker.ready;
  await registration.sync.register('sync-form-data');

  showNotification('Saved! Will sync when online.');
}
```

### Handling the Sync Event

```javascript
// sw.js
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-form-data') {
    event.waitUntil(syncFormData());
  }
});

async function syncFormData() {
  const db = await openDB('my-app', 1);
  const pending = await db.getAll('pendingSync');

  for (const item of pending) {
    try {
      await fetch('/api/submit', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(item),
      });
      await db.delete('pendingSync', item.id);
    } catch (error) {
      // Throwing causes the browser to retry the sync later
      throw error;
    }
  }
}
```

The browser retries the sync event with exponential backoff. If the `waitUntil` promise rejects, the sync is retried. If it resolves, the sync is complete.

---

## Conflict Resolution

When multiple clients modify the same data offline, conflicts are inevitable when they sync. There is no universal solution — the strategy depends on your data model and business requirements.

### Common Strategies

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| **Last-write-wins** | Most recent timestamp overwrites | Simple settings, user preferences |
| **Server-wins** | Server version always takes precedence | Authoritative data sources |
| **Client-wins** | Client version overwrites server | Draft documents, local-first apps |
| **Manual merge** | Present both versions to the user | Collaborative editing |
| **CRDT** | Conflict-free replicated data types | Real-time collaborative apps |
| **Operational transform** | Transform concurrent operations | Google Docs-style collaboration |

### Last-Write-Wins Implementation

```javascript
async function syncArticle(local, remote) {
  if (!remote) {
    // New local article — push to server
    return pushToServer(local);
  }

  if (local.updatedAt > remote.updatedAt) {
    // Local is newer — push to server
    return pushToServer(local);
  }

  if (remote.updatedAt > local.updatedAt) {
    // Remote is newer — update local
    return saveLocally(remote);
  }

  // Same timestamp — no conflict
  return local;
}
```

### Version Vector Approach

For multi-device sync, track a version counter per device:

```javascript
const article = {
  id: 'article-1',
  title: 'My Post',
  content: '...',
  versions: {
    'device-a': 3,
    'device-b': 2,
  },
};

function isNewer(incoming, existing) {
  // Incoming is newer if ALL its version counters are ≥ the existing ones
  return Object.entries(incoming.versions).every(
    ([device, version]) => version >= (existing.versions[device] || 0)
  );
}
```

---

## Offline UX Patterns

### Show Current State Clearly

Always indicate the connectivity state and data freshness:

```javascript
function updateStatusUI() {
  const banner = document.getElementById('status-banner');

  if (!navigator.onLine) {
    banner.textContent = 'You are offline. Changes will sync when reconnected.';
    banner.className = 'banner banner-warning';
    banner.hidden = false;
  } else {
    banner.hidden = true;
  }
}

window.addEventListener('online', updateStatusUI);
window.addEventListener('offline', updateStatusUI);
```

### Optimistic UI Updates

Apply changes to the UI immediately, then confirm with the server in the background:

```javascript
async function likePost(postId) {
  // 1. Update UI immediately
  updateLikeCount(postId, +1);

  // 2. Queue the server request
  try {
    await fetch(`/api/posts/${postId}/like`, { method: 'POST' });
  } catch {
    // 3. If offline, queue for background sync
    await queueForSync({ action: 'like', postId });
    showToast('Like saved — will sync when online');
  }
}
```

### Offline Fallback Page

Serve a custom offline page when the user navigates to an uncached URL:

```javascript
// sw.js — pre-cache an offline fallback
const OFFLINE_PAGE = '/offline.html';

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('offline-v1').then((cache) => cache.add(OFFLINE_PAGE))
  );
});

self.addEventListener('fetch', (event) => {
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request).catch(() => caches.match(OFFLINE_PAGE))
    );
  }
});
```

### Key UX Principles

| Principle | Implementation |
|-----------|---------------|
| Never show a blank screen | Pre-cache the app shell and an offline fallback page |
| Show data age | Display "Last updated: 5 minutes ago" alongside cached content |
| Queue user actions | Save writes to IndexedDB and sync later instead of showing errors |
| Confirm sync completion | Notify users when background sync completes successfully |
| Degrade gracefully | Disable features that require connectivity rather than hiding them |

---

## Key Takeaways

- Offline-first treats the network as an enhancement — design for no connectivity first, then layer on network access.
- `navigator.onLine` is unreliable for actual internet reachability; use a fetch-based health check for critical operations.
- IndexedDB is the primary offline data store for structured data — use a wrapper library like `idb` to avoid the verbose raw API.
- The Background Sync API queues operations until the browser has a stable connection, with automatic retry and exponential backoff.
- Conflict resolution must be an explicit design decision — last-write-wins is simple but not always correct; consider version vectors or CRDTs for collaborative scenarios.
- Clear status indicators, optimistic UI updates, and offline fallback pages are essential UX patterns that make offline behaviour feel intentional rather than broken.
