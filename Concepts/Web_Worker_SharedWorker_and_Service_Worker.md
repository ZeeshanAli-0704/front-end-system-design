# Web Workers vs Shared Workers vs Service Workers

### A Practical, Mental-Model-First Guide for Modern Web Apps

Modern web applications must be **fast, responsive, resilient, and multi-tab aware**.
To achieve this, browsers provide **three different background execution models**:

* **Web Workers**
* **Shared Workers**
* **Service Workers**

Although they sound similar, they solve **very different problems**.

This blog builds a **clear mental model**, explains **when to use which**, dives into **capabilities & limitations**, and walks through **real-world usage with code examples**.


## Table of Contents

1. [Why Background Workers Exist](#why-background-workers-exist)
2. [Quick Mental Model](#quick-mental-model)
3. [Dedicated Web Worker](#dedicated-web-worker)
4. [Shared Worker](#shared-worker)
5. [Service Worker](#service-worker)
6. [Comparison Table (Web vs Shared vs Service Worker)](#comparison-table-web-vs-shared-vs-service-worker)
7. [Choosing the Right Worker (Decision Guide)](#choosing-the-right-worker-decision-guide)
8. [Real World Usage Scenarios](#real-world-usage-scenarios)
9. [Trade offs & Gotchas](#trade-offs--gotchas)
10. [Security & Compliance Considerations](#security--compliance-considerations)
11. [Final Recommendations](#final-recommendations)


---

## Why Background Workers Exist

JavaScript runs on a **single-threaded event loop**.

If you:

* parse a large file
* encrypt data
* process images
* handle streaming data
* coordinate across multiple tabs

…on the **main thread**, your UI freezes.

**Workers exist to move work off the main thread** while keeping your app responsive.

---

## Quick Mental Model

Think of workers like this:

### 🧠 Mental Model

* **Web Worker (Dedicated Worker)**
  → A background thread for **one page only**
  → Best for **CPU-heavy tasks**

* **Shared Worker**
  → A background thread **shared by multiple tabs**
  → Best for **centralized state or shared resources**

* **Service Worker**
  → A **network proxy + offline engine**
  → Best for **caching, offline, push, background sync**

> ⚠️ Service Workers are **not for long-running compute loops**

---

## Dedicated Web Worker

### What is a Web Worker?

A **Dedicated Web Worker**:

* Runs in a **separate thread**
* Is tied to **one page/tab**
* Has **no DOM or UI access**
* Communicates via `postMessage`

### When to Use Web Workers

Use a Web Worker when:

* One page needs heavy computation
* You want to keep UI smooth
* No cross-tab coordination is needed

### Common Use Cases

* JSON parsing
* Image/video processing
* Cryptography
* Large data transformations
* Search indexing

---

### Architecture

```
Main Thread (UI)
     |
     | postMessage
     v
Dedicated Worker
(CPU-heavy work)
```

---

### Example: Offloading Heavy Compute

#### `worker.js`

```js
self.onmessage = (e) => {
  const { type, payload } = e.data || {};

  if (type === 'sum') {
    const { array } = payload || {};
    const total = array.reduce((a, b) => a + b, 0);
    postMessage({ type: 'sum:result', total });
  }
};
```

#### `main.js`

```js
const worker = new Worker('/worker.js', { type: 'module' });

worker.onmessage = (e) => {
  if (e.data?.type === 'sum:result') {
    console.log('Sum is', e.data.total);
  }
};

worker.postMessage({
  type: 'sum',
  payload: { array: [1, 2, 3, 4] }
});

// worker.terminate(); // cleanup when done
```

---

## Shared Worker

### What is a Shared Worker?

A **SharedWorker**:

* Is shared by **multiple tabs/windows**
* Belongs to the **same origin**
* Maintains **in-memory shared state**
* Uses **MessagePort** instead of `onmessage`

### Why Shared Workers Exist

Without Shared Workers:

* Each tab opens its own WebSocket
* Each tab maintains duplicate state
* Backend load increases

Shared Workers solve this by acting as a **single coordinator**.

---

### When to Use Shared Workers

Use a SharedWorker when:

* Multiple tabs need shared state
* You want one WebSocket per user
* You need cross-tab coordination

---

### Architecture

```
Tab A ─┐
       ├─ SharedWorker ── WebSocket
Tab B ─┘
```

---

### Example: Single WebSocket Shared Across Tabs

#### `shared-worker.js`

```js
const ports = new Set();
let socket;

function broadcast(data, exceptPort) {
  for (const port of ports) {
    if (port !== exceptPort) {
      port.postMessage(data);
    }
  }
}

function ensureSocket() {
  if (socket && socket.readyState === WebSocket.OPEN) return;

  socket = new WebSocket('wss://example.com/realtime');

  socket.onopen = () => broadcast({ type: 'ws:status', status: 'open' });
  socket.onmessage = (evt) =>
    broadcast({ type: 'ws:message', data: evt.data });
  socket.onclose = () =>
    broadcast({ type: 'ws:status', status: 'closed' });
}

onconnect = (e) => {
  const port = e.ports[0];
  ports.add(port);

  ensureSocket();

  port.onmessage = (evt) => {
    if (evt.data?.type === 'ws:send') {
      socket?.send(JSON.stringify(evt.data.payload));
    }
  };

  port.start();
  port.postMessage({ type: 'connected' });

  port.onclose = () => {
    ports.delete(port);
    if (ports.size === 0) socket?.close();
  };
};
```

#### In Each Page

```js
const worker = new SharedWorker('/shared-worker.js', { name: 'shared' });
worker.port.start();

worker.port.onmessage = (e) => {
  console.log('From shared worker:', e.data);
};

worker.port.postMessage({
  type: 'ws:send',
  payload: { message: 'hello' }
});
```

---

## Service Worker

### What is a Service Worker?

A **Service Worker**:

* Runs **outside any page**
* Acts as a **network proxy**
* Is **event-driven**
* Enables **offline, caching, push, sync**

### Why Service Workers Are Different

They:

* Intercept `fetch` requests
* Control multiple pages
* Can outlive page reloads
* Are **not always running**

---

### Lifecycle

```
Install → Activate → Fetch / Push / Sync → Terminate
```

> Service workers wake up **only for events**

---

### Example: Offline Caching + Request Routing

#### `sw.js`

```js
const CACHE_NAME = 'app-cache-v1';
const PRECACHE_URLS = [
  '/',
  '/index.html',
  '/styles.css',
  '/app.js'
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache =>
      cache.addAll(PRECACHE_URLS)
    )
  );
  self.skipWaiting();
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(
        keys.map(k => k !== CACHE_NAME && caches.delete(k))
      )
    )
  );
  self.clients.claim();
});

self.addEventListener('fetch', (event) => {
  if (event.request.method !== 'GET') return;

  event.respondWith(
    caches.match(event.request).then(cached =>
      cached || fetch(event.request)
    )
  );
});
```

#### Registering from the Page

```js
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js')
    .then(reg => console.log('SW registered', reg.scope))
    .catch(console.error);
}
```

---

## Comparison Table

| Feature            | Web Worker       | Shared Worker     | Service Worker |
| ------------------ | ---------------- | ----------------- | -------------- |
| Scope              | One page         | Multiple tabs     | Entire origin  |
| Lifetime           | While referenced | While ports exist | Event-driven   |
| DOM Access         | ❌                | ❌                 | ❌              |
| CPU Work           | ✅                | ✅                 | ❌              |
| Fetch Interception | ❌                | ❌                 | ✅              |
| Offline Support    | ❌                | ❌                 | ✅              |
| Push Notifications | ❌                | ❌                 | ✅              |

---

## Choosing the Right Worker

### Use **Web Worker** if:

* Heavy computation per page
* No shared state needed

### Use **Shared Worker** if:

* Multiple tabs share state
* One WebSocket per user
* Cross-tab coordination

### Use **Service Worker** if:

* Offline-first app
* Network caching
* Push notifications
* Background sync

---

## Real-World Usage Scenarios

* **Google Docs** → SharedWorker for sync + Service Worker for offline
* **Figma** → Web Workers for rendering & computation
* **E-commerce PWA** → Service Worker for caching & resilience
* **Trading dashboards** → SharedWorker for real-time streams

---

## Trade-offs & Gotchas

### Web Worker

* ✔ Simple
* ❌ No sharing across tabs

### Shared Worker

* ✔ Efficient resource sharing
* ❌ Lifecycle complexity
* ❌ Limited support in private browsing

### Service Worker

* ✔ Extremely powerful
* ❌ Complex lifecycle
* ❌ Careful cache invalidation required

---

## Security & Compliance Tips

* Never store secrets in workers
* Validate all incoming messages
* Avoid caching authenticated responses
* Always use HTTPS
* Follow least-privilege principles

> For enterprise apps (e.g., Oracle ecosystems), align with internal security and compliance guidelines before adoption.

---

## Final Recommendations

**Think in responsibilities, not APIs**:

* **Compute → Web Worker**
* **Coordination → Shared Worker**
* **Network & Offline → Service Worker**

If you share your **specific use case**, I can help you design:

* Messaging patterns
* Caching strategies
* Worker composition (yes, they can coexist!)

---