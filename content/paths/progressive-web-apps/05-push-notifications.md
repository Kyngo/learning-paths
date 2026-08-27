---
title: "Push Notifications"
weight: 5
---

## The Push Architecture

Push notifications on the web involve three actors and two separate APIs:

```text
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│  Your Server  │ ──► │  Push Service     │ ──► │  Browser +   │
│  (backend)    │     │  (vendor-operated)│     │  Service Worker│
└──────────────┘     └──────────────────┘     └──────────────┘
      │                     │                        │
  Web Push Protocol     Vendor endpoint          push event
  (RFC 8030)            (FCM, Mozilla, etc.)     in SW
```

| Component | Role |
|-----------|------|
| **Notification API** | Displays notifications to the user (can be used without push) |
| **Push API** | Allows the server to send messages to the browser, even when the page is closed |
| **Push Service** | Vendor-operated server (FCM for Chrome, autopush for Firefox) that queues and delivers messages |
| **VAPID** | Voluntary Application Server Identification — authenticates your server to the push service |

---

## The Notification API

The Notification API works independently of push — you can show notifications from page JavaScript without involving a server:

```javascript
// Request permission
const permission = await Notification.requestPermission();
// Returns: 'granted', 'denied', or 'default'

if (permission === 'granted') {
  new Notification('Hello!', {
    body: 'This is a local notification',
    icon: '/icons/icon-192.png',
    badge: '/icons/badge-72.png',
  });
}
```

### Notification Options

| Option | Type | Purpose |
|--------|------|---------|
| `body` | string | Main notification text |
| `icon` | URL | Large icon displayed alongside the notification |
| `badge` | URL | Small monochrome icon (Android status bar) |
| `image` | URL | Large image displayed in the notification body |
| `tag` | string | Identifier for replacing/grouping notifications |
| `renotify` | boolean | Vibrate/sound again when replacing a tagged notification |
| `actions` | array | Up to two action buttons |
| `data` | any | Custom data accessible in the notification click handler |
| `silent` | boolean | Suppress sound and vibration |
| `requireInteraction` | boolean | Notification persists until user interacts (desktop) |

### Service Worker Notifications

For push-triggered notifications, use `self.registration.showNotification()` inside the service worker:

```javascript
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? {};

  event.waitUntil(
    self.registration.showNotification(data.title || 'New Update', {
      body: data.body || 'You have a new notification',
      icon: '/icons/icon-192.png',
      badge: '/icons/badge-72.png',
      tag: data.tag || 'default',
      data: { url: data.url || '/' },
      actions: [
        { action: 'open', title: 'Open' },
        { action: 'dismiss', title: 'Dismiss' },
      ],
    })
  );
});
```

### Handling Notification Clicks

```javascript
self.addEventListener('notificationclick', (event) => {
  event.notification.close();

  if (event.action === 'dismiss') return;

  const targetUrl = event.notification.data?.url || '/';

  event.waitUntil(
    clients.matchAll({ type: 'window', includeUncontrolled: true }).then((windowClients) => {
      // Focus an existing tab if one is open
      for (const client of windowClients) {
        if (client.url === targetUrl && 'focus' in client) {
          return client.focus();
        }
      }
      // Otherwise open a new tab
      return clients.openWindow(targetUrl);
    })
  );
});
```

---

## VAPID Keys

VAPID (Voluntary Application Server Identification) uses a public/private key pair to authenticate your server to the push service. This prevents other servers from sending notifications to your subscribers.

### Generating VAPID Keys

```bash
npx web-push generate-vapid-keys
```

Output:

```text
Public Key:  BNbxGYNMhEIi3hYH0kfJft1_6Gzlhm...
Private Key: 3KzvKMdXaFGkYLaPQMrwBP5pnGkD3R...
```

Store the **private key** securely on your server. The **public key** is sent to the browser during subscription.

---

## Subscription Management

### Subscribing a User

```javascript
async function subscribeToPush() {
  const registration = await navigator.serviceWorker.ready;

  // Check existing subscription
  let subscription = await registration.pushManager.getSubscription();
  if (subscription) return subscription;

  // Convert VAPID public key from base64 to Uint8Array
  const vapidPublicKey = 'BNbxGYNMhEIi3hYH0kfJft1_6Gzlhm...';
  const convertedKey = urlBase64ToUint8Array(vapidPublicKey);

  subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true, // Required: promise to show a notification for every push
    applicationServerKey: convertedKey,
  });

  // Send subscription to your backend
  await fetch('/api/push/subscribe', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(subscription),
  });

  return subscription;
}

function urlBase64ToUint8Array(base64String) {
  const padding = '='.repeat((4 - (base64String.length % 4)) % 4);
  const base64 = (base64String + padding).replace(/-/g, '+').replace(/_/g, '/');
  const raw = atob(base64);
  return Uint8Array.from([...raw].map((char) => char.charCodeAt(0)));
}
```

### The Subscription Object

The `PushSubscription` returned by `subscribe()` looks like this:

```json
{
  "endpoint": "https://fcm.googleapis.com/fcm/send/fD1Jc...",
  "expirationTime": null,
  "keys": {
    "p256dh": "BLQELIDm-6b2Iwh...",
    "auth": "4vQK-SvR..."
  }
}
```

| Field | Purpose |
|-------|---------|
| `endpoint` | The push service URL unique to this subscription |
| `keys.p256dh` | Public encryption key for message payload |
| `keys.auth` | Authentication secret |

Store these on your server, associated with the user.

### Unsubscribing

```javascript
async function unsubscribe() {
  const registration = await navigator.serviceWorker.ready;
  const subscription = await registration.pushManager.getSubscription();

  if (subscription) {
    await subscription.unsubscribe();
    // Notify your backend to remove the subscription
    await fetch('/api/push/unsubscribe', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ endpoint: subscription.endpoint }),
    });
  }
}
```

---

## Server-Side Push

### Using the web-push Library (Node.js)

```bash
npm install web-push
```

```javascript
import webPush from 'web-push';

webPush.setVapidDetails(
  'mailto:admin@example.com',
  process.env.VAPID_PUBLIC_KEY,
  process.env.VAPID_PRIVATE_KEY
);

async function sendPushNotification(subscription, payload) {
  try {
    await webPush.sendNotification(
      subscription,
      JSON.stringify(payload),
      { TTL: 60 * 60 } // Time-to-live in seconds
    );
    console.log('Push sent successfully');
  } catch (error) {
    if (error.statusCode === 410 || error.statusCode === 404) {
      // Subscription expired or invalid — remove from database
      await removeSubscription(subscription.endpoint);
    } else {
      console.error('Push failed:', error);
    }
  }
}

// Usage
await sendPushNotification(userSubscription, {
  title: 'New Message',
  body: 'You have a new message from Alice',
  url: '/messages/123',
  tag: 'message-123',
});
```

### Push Service Response Codes

| Status | Meaning | Action |
|--------|---------|--------|
| 201 | Message accepted | Success |
| 404 | Subscription not found | Remove from database |
| 410 | Subscription expired | Remove from database |
| 413 | Payload too large | Max ~4 KB encrypted |
| 429 | Too many requests | Retry with backoff |

---

## Permission UX Best Practices

The notification permission prompt is one-shot in many browsers — if the user dismisses or denies it, you cannot ask again easily. Poor timing destroys your opt-in rate.

### What NOT to Do

- ❌ Request permission on page load before the user has engaged.
- ❌ Show the browser prompt without context or explanation.
- ❌ Use dark patterns ("Allow notifications to continue reading").

### Recommended Pattern: Two-Step Permission

```javascript
function showPermissionDialog() {
  const dialog = document.getElementById('push-dialog');
  dialog.innerHTML = `
    <h3>Stay Updated</h3>
    <p>Get notified when your order ships or when items go on sale.</p>
    <button id="push-accept">Enable Notifications</button>
    <button id="push-decline">Not Now</button>
  `;
  dialog.hidden = false;

  document.getElementById('push-accept').addEventListener('click', async () => {
    dialog.hidden = true;
    const permission = await Notification.requestPermission();
    if (permission === 'granted') {
      await subscribeToPush();
    }
  });

  document.getElementById('push-decline').addEventListener('click', () => {
    dialog.hidden = true;
    localStorage.setItem('push-declined', Date.now().toString());
  });
}

// Only show after meaningful engagement
function maybePromptForPush() {
  if (Notification.permission !== 'default') return; // Already decided
  const declined = localStorage.getItem('push-declined');
  if (declined && Date.now() - parseInt(declined) < 30 * 24 * 60 * 60 * 1000) return; // Wait 30 days

  showPermissionDialog();
}
```

### Timing Recommendations

| Trigger | Why It Works |
|---------|-------------|
| After a purchase completes | User wants order updates |
| After following a topic | User wants content updates |
| After using the app 3+ times | Demonstrated engagement |
| When the user clicks a "notify me" button | Explicit intent |

---

## Key Takeaways

- Push notifications involve two APIs: the **Notification API** (display) and the **Push API** (server-to-browser delivery), connected through vendor-operated push services.
- VAPID keys authenticate your server to the push service — the public key is shared with the browser, the private key stays on the server.
- The `PushSubscription` object (endpoint + encryption keys) must be stored on your server and removed when the push service returns 404 or 410.
- Always handle notification clicks by focusing an existing tab or opening a new one — an unhandled click is a missed engagement opportunity.
- Permission UX is critical: use a two-step dialog that explains value before triggering the browser prompt, and time it to a moment of demonstrated user intent.
- Server-side push payloads are encrypted and limited to approximately 4 KB — send a minimal data object and let the service worker construct the full notification.
