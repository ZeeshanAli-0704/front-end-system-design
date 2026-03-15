# Web Workers vs Shared Workers vs Service Workers

### A Practical, Mental-Model-First Guide for Modern Web Apps

Modern web applications must be **fast, responsive, resilient, and multi-tab aware**.
To achieve this, browsers provide **three different background execution models**:

* **Web Workers** (Dedicated Workers)
* **Shared Workers**
* **Service Workers**

Although they sound similar, they solve **very different problems**, have **different lifecycles**, and communicate with the main thread in **fundamentally different ways**.

This guide builds a **clear mental model**, explains **when to use which**, dives into **capabilities & limitations**, and walks through **real-world usage with production-grade code examples**. We pay special attention to **communication patterns** — the most common source of confusion.

---

<a id="top"></a>

## Table of Contents

1. [Why Background Workers Exist](#why-background-workers-exist)
2. [Quick Mental Model](#quick-mental-model)
3. [Dedicated Web Worker](#dedicated-web-worker)
4. [Shared Worker](#shared-worker)
5. [Service Worker](#service-worker)
6. [Communication Patterns Deep Dive](#communication-patterns-deep-dive)
7. [Comparison Table (Web vs Shared vs Service Worker)](#comparison-table-web-vs-shared-vs-service-worker)
8. [Choosing the Right Worker (Decision Guide)](#choosing-the-right-worker-decision-guide)
9. [Real-World Usage Scenarios (with Code)](#real-world-usage-scenarios-with-code)
10. [Tradeoffs and Gotchas](#tradeoffs-and-gotchas)
11. [Security and Compliance Considerations](#security-and-compliance-considerations)
12. [Final Recommendations](#final-recommendations)

[⬆ Back to Top](#top)

---

## Why Background Workers Exist

JavaScript runs on a **single-threaded event loop**.

Everything — DOM updates, event handling, network callbacks, timers — shares **one thread**. If a single operation takes too long, **everything else waits**.

```
User clicks button
   → Event handler starts
   → Heavy computation (500ms) 🔴 UI frozen
   → DOM update finally happens
```

If you:

* parse a 50 MB JSON file
* encrypt / hash data (bcrypt, AES)
* process images / video frames
* run ML inference on the client
* handle streaming data
* coordinate across multiple tabs

…on the **main thread**, your UI freezes. Animations jank. Input feels laggy. Users leave.

### The Solution: Workers

**Workers exist to move work off the main thread** while keeping your app responsive.

```
Main Thread                     Worker Thread
┌──────────────┐               ┌──────────────┐
│  UI Rendering │               │ Heavy Compute │
│  Event Loop   │  postMessage  │ No DOM access │
│  DOM Access   │ ◄──────────► │ Own scope     │
└──────────────┘               └──────────────┘
```

Workers run in an **isolated global context** (`self` instead of `window`). They have:
- **No access** to DOM, `document`, `window`, `localStorage`
- **Their own event loop**
- **Structured clone** for data transfer (or `Transferable` objects for zero-copy)

[⬆ Back to Top](#top)

---

## Quick Mental Model

Think of workers like **employees with different job descriptions**:

### 🧠 Mental Model

| Worker | Analogy | Scope | Key Job |
|--------|---------|-------|---------|
| **Web Worker** | A **private assistant** for one person | One page/tab | CPU-heavy tasks |
| **Shared Worker** | A **team assistant** shared by everyone on the floor | Multiple tabs (same origin) | Centralized state, shared connections |
| **Service Worker** | The **building's mail room** | Entire origin (all pages) | Network proxy, caching, offline, push |

```
┌─────────────────────────────────────────────────────┐
│                    Browser (Same Origin)             │
│                                                      │
│  Tab A ──── Web Worker A   (private, dies with tab)  │
│    │                                                 │
│    ├──── Shared Worker ◄──── Tab B                   │
│    │       (lives while any tab is connected)        │
│    │                                                 │
│  Service Worker (intercepts ALL network requests)    │
│    │  (event-driven, wakes up & sleeps)              │
│    │                                                 │
│  Tab A ── fetch() ──► Service Worker ──► Network     │
│  Tab B ── fetch() ──► Service Worker ──► Cache       │
└─────────────────────────────────────────────────────┘
```

> **Key Insight**: Web Workers are about **compute**, Shared Workers are about **coordination**, Service Workers are about **the network**.

> ⚠️ Service Workers are **not for long-running compute loops** — the browser terminates them when idle.

[⬆ Back to Top](#top)

---

## Dedicated Web Worker

### What is a Web Worker?

A **Dedicated Web Worker**:

* Runs in a **separate OS-level thread**
* Is tied to **one page/tab** (1:1 relationship)
* Has **no DOM or UI access** (no `document`, no `window`)
* Communicates via **`postMessage` / `onmessage`**
* Can import scripts via `importScripts()` or ES modules
* Gets **garbage collected** when the page closes or `worker.terminate()` is called

### When to Use Web Workers

Use a Web Worker when:

* One page needs **heavy computation** (>16ms tasks that block 60fps)
* You want to keep **UI smooth** during processing
* **No cross-tab coordination** is needed
* You need **synchronous-style code** without blocking the main thread

### Common Use Cases

| Use Case | Example |
|----------|---------|
| JSON parsing | Parse 10MB API response |
| Image processing | Resize, crop, apply filters |
| Cryptography | Client-side encryption/hashing |
| Data transformation | CSV → JSON, sorting large arrays |
| Search indexing | Full-text search over local data |
| ML inference | TensorFlow.js model execution |
| Markdown rendering | Convert MD → HTML for previews |
| Syntax highlighting | Parse and tokenize code blocks |

---

### Architecture

```
Main Thread (UI)
     │
     │  postMessage({ type, payload })
     ▼
┌────────────────────┐
│  Dedicated Worker   │
│  Own global scope   │
│  Own event loop     │
│  CPU-heavy work     │
└────────────────────┘
     │
     │  postMessage({ type, result })
     ▼
Main Thread receives result → updates DOM
```

---

### Example 1: Offloading Heavy Compute

#### `worker.js`

```js
// Workers have their own global scope: `self`
self.onmessage = (e) => {
  const { type, payload } = e.data || {};

  switch (type) {
    case 'sum': {
      const total = payload.array.reduce((a, b) => a + b, 0);
      self.postMessage({ type: 'sum:result', total });
      break;
    }

    case 'sort': {
      const sorted = [...payload.array].sort((a, b) => a - b);
      self.postMessage({ type: 'sort:result', sorted });
      break;
    }

    default:
      self.postMessage({ type: 'error', message: `Unknown type: ${type}` });
  }
};
```

#### `main.js`

```js
const worker = new Worker('/worker.js');

// Listen for results
worker.onmessage = (e) => {
  const { type } = e.data;
  if (type === 'sum:result') {
    console.log('Sum:', e.data.total);
  } else if (type === 'sort:result') {
    console.log('Sorted:', e.data.sorted);
  }
};

// Handle errors
worker.onerror = (err) => {
  console.error('Worker error:', err.message);
};

// Send work to the worker
worker.postMessage({
  type: 'sum',
  payload: { array: [1, 2, 3, 4, 5] },
});

// Cleanup when done
// worker.terminate();
```

---

### Example 2: Promise-Based Worker Wrapper (Request/Response Pattern)

Raw `postMessage` is fire-and-forget. For real apps, you often need a **request → response** pattern:

```js
// workerClient.js — wraps postMessage as Promises
function createWorkerClient(url) {
  const worker = new Worker(url);
  const pending = new Map();
  let nextId = 0;

  worker.onmessage = (e) => {
    const { id, result, error } = e.data;
    const resolver = pending.get(id);
    if (!resolver) return;

    pending.delete(id);
    error ? resolver.reject(new Error(error)) : resolver.resolve(result);
  };

  return {
    request(type, payload) {
      const id = nextId++;
      return new Promise((resolve, reject) => {
        pending.set(id, { resolve, reject });
        worker.postMessage({ id, type, payload });
      });
    },
    terminate() {
      worker.terminate();
    },
  };
}

// Usage
const client = createWorkerClient('/worker.js');
const total = await client.request('sum', { array: [1, 2, 3] });
console.log('Total:', total); // 6
```

#### Matching worker that supports request IDs:

```js
// worker.js — supports request/response with IDs
self.onmessage = (e) => {
  const { id, type, payload } = e.data;

  try {
    let result;
    switch (type) {
      case 'sum':
        result = payload.array.reduce((a, b) => a + b, 0);
        break;
      default:
        throw new Error(`Unknown type: ${type}`);
    }
    self.postMessage({ id, result });
  } catch (err) {
    self.postMessage({ id, error: err.message });
  }
};
```

---

### Example 3: Transferable Objects (Zero-Copy Performance)

When sending large data (images, ArrayBuffers), **structured clone** copies the data. Use **Transferable** to avoid the copy:

```js
// Main thread — transfer an ArrayBuffer (zero-copy)
const buffer = new ArrayBuffer(1024 * 1024); // 1MB
console.log(buffer.byteLength); // 1048576

// Transfer ownership — buffer becomes empty in main thread
worker.postMessage({ type: 'process', buffer }, [buffer]);
console.log(buffer.byteLength); // 0 (transferred!)

// Worker side
self.onmessage = (e) => {
  const { buffer } = e.data;
  const view = new Uint8Array(buffer);
  // ... process the buffer ...

  // Transfer it back
  self.postMessage({ type: 'done', buffer }, [buffer]);
};
```

> **When to use Transferable**: ArrayBuffer, MessagePort, ImageBitmap, OffscreenCanvas. Use when data is large (>1MB) and you don't need it in the sender after transfer.

[⬆ Back to Top](#top)

---

## Shared Worker

### What is a Shared Worker?

A **SharedWorker**:

* Is shared by **multiple tabs, iframes, or windows** of the same origin
* Maintains **in-memory shared state** across all connected contexts
* Uses **`MessagePort`** for communication (not direct `onmessage`)
* Stays alive as long as **at least one port is connected**
* Has **one instance per URL + name combination** per origin

### Why Shared Workers Exist

Without Shared Workers, **every tab is an island**:

```
❌ Without Shared Worker:
Tab A ── WebSocket ──► Server
Tab B ── WebSocket ──► Server    (duplicate connections!)
Tab C ── WebSocket ──► Server    (3x server load!)
Each tab: own state, own connection, own memory
```

```
✅ With Shared Worker:
Tab A ─┐
Tab B ─┼── SharedWorker ── 1 WebSocket ──► Server
Tab C ─┘   (single connection, shared state, 1/3 server load)
```

**Problems Shared Workers solve**:
- **Duplicate WebSocket connections** (1 per user instead of 1 per tab)
- **State drift** between tabs (single source of truth)
- **Resource waste** (one copy of data in memory, not N)
- **Race conditions** (centralized write coordination)

---

### When to Use Shared Workers

Use a SharedWorker when:

* Multiple tabs need **shared in-memory state**
* You want **one WebSocket per user** (not per tab)
* You need **cross-tab coordination** (e.g., "only one tab shows notification")
* You want to **deduplicate API calls** across tabs

### Common Use Cases

| Use Case | Why Shared Worker? |
|----------|-------------------|
| Real-time dashboard | One WebSocket feeds all tabs |
| Auth session management | One tab refreshes token, all tabs benefit |
| Collaborative editing | Centralized OT/CRDT state |
| Shopping cart | Cart state synced across product pages |
| Notification dedup | "New message" shown in one tab only |

---

### Architecture

```
Tab A ──► port A ─┐
                   │
Tab B ──► port B ─┼──► SharedWorker
                   │     │
Tab C ──► port C ─┘     │
                          ▼
                    WebSocket / State / Cache
```

Each tab gets a **MessagePort**. The worker keeps a **Set of ports** and can broadcast to all, or send to specific ones.

---

### Example 1: Single WebSocket Shared Across Tabs

#### `shared-worker.js`

```js
const ports = new Set();
let socket = null;

// Broadcast a message to all connected tabs (optionally excluding one)
function broadcast(data, exceptPort = null) {
  for (const port of ports) {
    if (port !== exceptPort) {
      port.postMessage(data);
    }
  }
}

// Maintain a single WebSocket connection
function ensureSocket() {
  if (socket && socket.readyState <= WebSocket.OPEN) return;

  socket = new WebSocket('wss://api.example.com/realtime');

  socket.onopen = () => {
    broadcast({ type: 'ws:status', status: 'connected' });
  };

  socket.onmessage = (evt) => {
    // Parse and fan out to all tabs
    const data = JSON.parse(evt.data);
    broadcast({ type: 'ws:message', data });
  };

  socket.onclose = () => {
    broadcast({ type: 'ws:status', status: 'disconnected' });
    // Auto-reconnect after 3s
    setTimeout(ensureSocket, 3000);
  };

  socket.onerror = () => socket.close();
}

// Called every time a new tab connects
self.onconnect = (e) => {
  const port = e.ports[0];
  ports.add(port);

  // Ensure WebSocket is alive
  ensureSocket();

  // Handle messages FROM this tab
  port.onmessage = (evt) => {
    const { type, payload } = evt.data;

    switch (type) {
      case 'ws:send':
        // Forward to WebSocket
        if (socket?.readyState === WebSocket.OPEN) {
          socket.send(JSON.stringify(payload));
        }
        break;

      case 'broadcast':
        // Tab wants to notify other tabs
        broadcast({ type: 'tab:message', data: payload }, port);
        break;

      case 'get:status':
        port.postMessage({
          type: 'ws:status',
          status: socket?.readyState === WebSocket.OPEN ? 'connected' : 'disconnected',
          tabCount: ports.size,
        });
        break;
    }
  };

  port.start();

  // Welcome message
  port.postMessage({
    type: 'connected',
    tabCount: ports.size,
  });
};
```

#### `main.js` (In Each Page/Tab)

```js
const worker = new SharedWorker('/shared-worker.js', { name: 'app-shared' });

// IMPORTANT: Must call port.start() or use onmessage (which auto-starts)
worker.port.start();

worker.port.onmessage = (e) => {
  const { type, data, status, tabCount } = e.data;

  switch (type) {
    case 'connected':
      console.log(`Connected! ${tabCount} tab(s) active`);
      break;

    case 'ws:status':
      updateConnectionIndicator(status);
      break;

    case 'ws:message':
      handleRealtimeUpdate(data);
      break;

    case 'tab:message':
      console.log('Message from another tab:', data);
      break;
  }
};

// Send a message through the shared WebSocket
worker.port.postMessage({
  type: 'ws:send',
  payload: { action: 'subscribe', channel: 'prices' },
});

// Broadcast to other tabs (not via WebSocket)
worker.port.postMessage({
  type: 'broadcast',
  payload: { message: 'User updated their profile' },
});
```

---

### Example 2: Shared State Store Across Tabs

```js
// shared-state-worker.js — Acts as a centralized store
const ports = new Set();
const state = { cart: [], user: null, theme: 'light' };

function broadcast(data, except = null) {
  for (const p of ports) {
    if (p !== except) p.postMessage(data);
  }
}

self.onconnect = (e) => {
  const port = e.ports[0];
  ports.add(port);

  // New tab gets current state immediately
  port.postMessage({ type: 'state:sync', state: structuredClone(state) });

  port.onmessage = (evt) => {
    const { type, key, value } = evt.data;

    if (type === 'state:set') {
      state[key] = value;
      // Notify ALL tabs (including sender) for consistency
      broadcast({ type: 'state:update', key, value });
    }

    if (type === 'state:get') {
      port.postMessage({ type: 'state:value', key, value: state[key] });
    }
  };

  port.start();
};
```

```js
// Usage in any tab
const store = new SharedWorker('/shared-state-worker.js');
store.port.start();

// Update cart (all tabs see the change)
store.port.postMessage({
  type: 'state:set',
  key: 'cart',
  value: [{ id: 1, name: 'Widget', qty: 2 }],
});

// Listen for state changes from any tab
store.port.onmessage = (e) => {
  if (e.data.type === 'state:update') {
    console.log(`State "${e.data.key}" changed:`, e.data.value);
    renderUI(e.data.key, e.data.value);
  }
};
```

---

### ⚠️ Shared Worker Gotcha: Debugging

Shared Workers **don't appear** in the regular DevTools. To debug:
- **Chrome**: Navigate to `chrome://inspect/#workers`
- **Firefox**: Navigate to `about:debugging#/runtime/this-firefox`
- Add `console.log` liberally — they appear in the worker's own console

[⬆ Back to Top](#top)

---

## Service Worker

### What is a Service Worker?

A **Service Worker**:

* Runs **outside any page** — it's a background process for your origin
* Acts as a **programmable network proxy** between browser and server
* Is **event-driven** — wakes up for events, sleeps when idle
* Enables **offline experiences, caching, push notifications, background sync**
* Has a strict **lifecycle** (install → activate → fetch)
* **Requires HTTPS** (except `localhost` for development)

### Why Service Workers Are Different

Unlike Web/Shared Workers:

* They **intercept every `fetch()` request** from controlled pages
* They **persist across page navigations** and even browser restarts
* They **are not always running** — the browser terminates idle ones
* They **cannot do heavy CPU work** (will be killed if blocking)
* They control **all pages** of the origin (after activation)

```
Without Service Worker:
Page ──► fetch('/api/data') ──► Network ──► Server ──► Response

With Service Worker:
Page ──► fetch('/api/data') ──► Service Worker (intercepts!)
                                  │
                                  ├── Cache hit? → Return cached response (instant!)
                                  │
                                  └── Cache miss? → fetch from Network → Cache → Return
```

---

### Lifecycle (Critical to Understand!)

```
┌──────────────────────────────────────────────────────────┐
│                 Service Worker Lifecycle                   │
│                                                           │
│  1. REGISTER    navigator.serviceWorker.register()        │
│       │                                                   │
│  2. INSTALL     'install' event → precache assets         │
│       │         (runs ONCE per SW version)                │
│       │                                                   │
│  3. WAIT        Waits for old SW to release all clients   │
│       │         (unless skipWaiting() is called)          │
│       │                                                   │
│  4. ACTIVATE    'activate' event → clean old caches       │
│       │         (runs ONCE, then controls pages)          │
│       │                                                   │
│  5. IDLE        Sleeps... waiting for events              │
│       │                                                   │
│  6. FETCH/PUSH  Wakes up for 'fetch', 'push', 'sync'     │
│       │                                                   │
│  7. TERMINATE   Browser kills idle SW to save memory      │
│                 (will wake again for next event)          │
└──────────────────────────────────────────────────────────┘
```

> **Key insight**: Service Workers are NOT continuously running. They wake up for events and go back to sleep. Do NOT store in-memory state in them — it will be lost!

---

### Caching Strategies

| Strategy | Description | Best For |
|----------|-------------|----------|
| **Cache First** | Check cache → fallback to network | Static assets (CSS, JS, images) |
| **Network First** | Try network → fallback to cache | API data that should be fresh |
| **Stale While Revalidate** | Return cache immediately → update cache from network in background | Frequently updated content |
| **Cache Only** | Only serve from cache | Precached app shell |
| **Network Only** | Always go to network | Auth endpoints, real-time data |

---

### Example 1: Offline Caching with Multiple Strategies

#### `sw.js`

```js
const CACHE_NAME = 'app-cache-v1';

// Assets to precache during install
const PRECACHE_URLS = [
  '/',
  '/index.html',
  '/styles.css',
  '/app.js',
  '/offline.html',
];

// ─── INSTALL: Precache critical assets ───
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.addAll(PRECACHE_URLS))
  );
  self.skipWaiting(); // Activate immediately, don't wait
});

// ─── ACTIVATE: Clean up old caches ───
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(
        keys
          .filter((key) => key !== CACHE_NAME)
          .map((key) => caches.delete(key))
      )
    )
  );
  self.clients.claim(); // Control all open tabs immediately
});

// ─── FETCH: Route requests to appropriate strategy ───
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);

  // Skip non-GET requests
  if (request.method !== 'GET') return;

  // API calls → Network First
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirst(request));
    return;
  }

  // Static assets → Cache First
  if (url.pathname.match(/\.(css|js|png|jpg|svg|woff2)$/)) {
    event.respondWith(cacheFirst(request));
    return;
  }

  // HTML pages → Stale While Revalidate
  event.respondWith(staleWhileRevalidate(request));
});

// ─── STRATEGIES ───

async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;

  try {
    const response = await fetch(request);
    const cache = await caches.open(CACHE_NAME);
    cache.put(request, response.clone());
    return response;
  } catch {
    return new Response('Offline', { status: 503 });
  }
}

async function networkFirst(request) {
  try {
    const response = await fetch(request);
    const cache = await caches.open(CACHE_NAME);
    cache.put(request, response.clone());
    return response;
  } catch {
    const cached = await caches.match(request);
    return cached || new Response(JSON.stringify({ error: 'Offline' }), {
      status: 503,
      headers: { 'Content-Type': 'application/json' },
    });
  }
}

async function staleWhileRevalidate(request) {
  const cache = await caches.open(CACHE_NAME);
  const cached = await cache.match(request);

  // Fetch fresh copy in the background
  const networkPromise = fetch(request)
    .then((response) => {
      cache.put(request, response.clone());
      return response;
    })
    .catch(() => cached); // If network fails, fall back to cache

  // Return cached immediately, or wait for network
  return cached || networkPromise;
}
```

#### Registering from the Page

```js
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker
      .register('/sw.js', { scope: '/' })
      .then((reg) => {
        console.log('SW registered, scope:', reg.scope);

        // Detect updates
        reg.onupdatefound = () => {
          const newSW = reg.installing;
          newSW.onstatechange = () => {
            if (newSW.state === 'activated') {
              // Notify user: "New version available, refresh?"
              showUpdateBanner();
            }
          };
        };
      })
      .catch((err) => console.error('SW registration failed:', err));
  });
}
```

---

### Example 2: Background Sync (Offline Form Submission)

```js
// sw.js — Retry failed form submissions when back online
self.addEventListener('sync', (event) => {
  if (event.tag === 'submit-form') {
    event.waitUntil(retryPendingSubmissions());
  }
});

async function retryPendingSubmissions() {
  const db = await openIndexedDB();
  const pending = await db.getAll('pending-submissions');

  for (const submission of pending) {
    try {
      await fetch('/api/submit', {
        method: 'POST',
        body: JSON.stringify(submission.data),
        headers: { 'Content-Type': 'application/json' },
      });
      await db.delete('pending-submissions', submission.id);
    } catch {
      // Will retry on next sync event
      break;
    }
  }
}
```

```js
// main.js — Queue submission when offline
async function submitForm(data) {
  try {
    await fetch('/api/submit', {
      method: 'POST',
      body: JSON.stringify(data),
      headers: { 'Content-Type': 'application/json' },
    });
  } catch {
    // Offline! Save to IndexedDB and register sync
    const db = await openIndexedDB();
    await db.put('pending-submissions', { data, timestamp: Date.now() });

    const reg = await navigator.serviceWorker.ready;
    await reg.sync.register('submit-form');
    showToast('Saved offline. Will submit when back online.');
  }
}
```

---

### Example 3: Push Notifications

```js
// sw.js — Handle push events
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? { title: 'New Notification' };

  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icon-192.png',
      badge: '/badge-72.png',
      data: { url: data.url || '/' },
    })
  );
});

// Handle notification click
self.addEventListener('notificationclick', (event) => {
  event.notification.close();

  event.waitUntil(
    clients.matchAll({ type: 'window' }).then((windowClients) => {
      // Focus existing tab or open new one
      const existing = windowClients.find(
        (c) => c.url === event.notification.data.url
      );
      return existing
        ? existing.focus()
        : clients.openWindow(event.notification.data.url);
    })
  );
});
```

```js
// main.js — Subscribe to push
async function subscribeToPush() {
  const reg = await navigator.serviceWorker.ready;
  const subscription = await reg.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey: urlBase64ToUint8Array(VAPID_PUBLIC_KEY),
  });

  // Send subscription to your backend
  await fetch('/api/push/subscribe', {
    method: 'POST',
    body: JSON.stringify(subscription),
    headers: { 'Content-Type': 'application/json' },
  });
}
```

[⬆ Back to Top](#top)

---

## Communication Patterns Deep Dive

This is the **most important section** for system design interviews. Each worker type communicates differently.

---

### Communication Summary

| Pattern | Web Worker | Shared Worker | Service Worker |
|---------|-----------|---------------|----------------|
| **Main → Worker** | `worker.postMessage()` | `worker.port.postMessage()` | `navigator.serviceWorker.controller.postMessage()` |
| **Worker → Main** | `self.postMessage()` | `port.postMessage()` | `client.postMessage()` |
| **Worker → All Tabs** | ❌ (only its own page) | `broadcast()` over ports | `self.clients.matchAll()` → loop `postMessage` |
| **Tab → Tab** | ❌ | Via SharedWorker relay | Via SW relay or `BroadcastChannel` |
| **Data Transfer** | Structured clone / Transferable | Structured clone | Structured clone |

---

### Pattern 1: Web Worker — Simple `postMessage` (1:1)

```
Main Thread ◄──── postMessage ────► Web Worker
  (1 page)                           (1 worker)
```

```js
// Main → Worker
worker.postMessage({ type: 'compute', data: [1, 2, 3] });

// Worker → Main
self.postMessage({ type: 'result', data: 6 });
```

**Use when**: One page needs to offload work. Simplest model.

---

### Pattern 2: Shared Worker — Port-Based (N:1)

```
Tab A ──port A──┐
Tab B ──port B──┼──► SharedWorker (maintains port set)
Tab C ──port C──┘
```

```js
// Tab → SharedWorker (via port)
worker.port.postMessage({ type: 'subscribe', channel: 'updates' });

// SharedWorker → Specific Tab
port.postMessage({ type: 'data', payload: {...} });

// SharedWorker → ALL Tabs (broadcast)
for (const p of ports) {
  p.postMessage({ type: 'broadcast', data: {...} });
}
```

**Use when**: Multiple tabs need the same data stream or shared state.

**Key difference from Web Worker**: You MUST use `port.start()` and handle the `onconnect` event.

---

### Pattern 3: Service Worker — Client-Based (1:N, event-driven)

```
Service Worker ──► clients.matchAll() ──► [Tab A, Tab B, Tab C]
                                               │
Page ──► navigator.serviceWorker.controller ──►│
```

#### Service Worker → All Pages:

```js
// Inside Service Worker: Notify all controlled pages
async function notifyAllClients(data) {
  const allClients = await self.clients.matchAll({
    type: 'window',
    includeUncontrolled: false,
  });

  for (const client of allClients) {
    client.postMessage(data);
  }
}

// Triggered by push event, for example
self.addEventListener('push', (event) => {
  event.waitUntil(
    notifyAllClients({
      type: 'push:received',
      payload: event.data?.json(),
    })
  );
});
```

#### Page → Service Worker:

```js
// Send message to Service Worker from a page
navigator.serviceWorker.controller?.postMessage({
  type: 'cache:clear',
  pattern: '/api/user/*',
});

// Listen for messages FROM Service Worker
navigator.serviceWorker.addEventListener('message', (event) => {
  console.log('From SW:', event.data);
});
```

#### Service Worker as Inter-Tab Message Relay:

```js
// sw.js — Relay messages between tabs
self.addEventListener('message', async (event) => {
  if (event.data?.type === 'relay') {
    const allClients = await self.clients.matchAll({ type: 'window' });

    for (const client of allClients) {
      // Don't echo back to sender
      if (client.id !== event.source.id) {
        client.postMessage({
          type: 'relayed',
          from: event.source.id,
          data: event.data.payload,
        });
      }
    }
  }
});
```

---

### Pattern 4: `BroadcastChannel` — The Simpler Alternative for Cross-Tab

If you **only** need cross-tab messaging (no shared state, no shared WebSocket), `BroadcastChannel` is simpler than SharedWorker:

```js
// In any tab — create channel with same name
const channel = new BroadcastChannel('app-events');

// Send to all other tabs (not to self)
channel.postMessage({ type: 'theme:changed', theme: 'dark' });

// Receive from other tabs
channel.onmessage = (e) => {
  if (e.data.type === 'theme:changed') {
    document.body.className = e.data.theme;
  }
};

// Cleanup
channel.close();
```

#### When to use BroadcastChannel vs SharedWorker:

| Need | Use |
|------|-----|
| Just send messages between tabs | `BroadcastChannel` ✅ |
| Shared WebSocket connection | `SharedWorker` ✅ |
| Centralized state management | `SharedWorker` ✅ |
| Coordinate who does what | `SharedWorker` ✅ |

---

### Pattern 5: Combining Workers (Real Architecture)

In production, you often **combine all three**:

```
┌──────────────────────────────────────────────────────────┐
│                    Browser (your origin)                   │
│                                                           │
│  Tab A                          Tab B                     │
│  ├── Web Worker (image resize)  ├── Web Worker (search)   │
│  │                              │                         │
│  └── port A ──┐                 └── port B ──┐            │
│               ▼                              ▼            │
│          SharedWorker (WebSocket, shared state)           │
│                                                           │
│  ──────────── Service Worker (network proxy) ───────────  │
│    │  Intercepts all fetch()                              │
│    │  Caches responses                                    │
│    │  Handles push notifications                          │
│    │  Background sync                                     │
└──────────────────────────────────────────────────────────┘
```

**Example: Chat app architecture**:
- **Web Worker** per tab: encrypts/decrypts messages (CPU work)
- **SharedWorker**: maintains single WebSocket, broadcasts to all tabs
- **Service Worker**: caches chat history for offline reading, handles push notifications

[⬆ Back to Top](#top)

---

## Comparison Table (Web vs Shared vs Service Worker)

| Feature | Web Worker | Shared Worker | Service Worker |
|---------|-----------|---------------|----------------|
| **Scope** | One page/tab | Multiple tabs (same origin) | Entire origin |
| **Lifetime** | While page is open | While any port is connected | Event-driven (sleeps/wakes) |
| **Survives page reload** | ❌ | ✅ (if other tabs open) | ✅ |
| **DOM Access** | ❌ | ❌ | ❌ |
| **CPU-heavy Work** | ✅ Best choice | ✅ Possible | ❌ Will be terminated |
| **Fetch Interception** | ❌ | ❌ | ✅ Only one that can |
| **Offline Support** | ❌ | ❌ | ✅ |
| **Push Notifications** | ❌ | ❌ | ✅ |
| **Background Sync** | ❌ | ❌ | ✅ |
| **Cross-tab Communication** | ❌ | ✅ Via ports | ✅ Via clients API |
| **Shared State** | ❌ | ✅ In-memory | ❌ (no persistent memory) |
| **HTTPS Required** | ❌ | ❌ | ✅ |
| **Communication API** | `postMessage` direct | `MessagePort` | `clients.postMessage` |
| **Browser Support** | ✅ Excellent | ⚠️ No Safari iOS < 16 | ✅ Excellent |
| **DevTools Debugging** | Easy (Sources tab) | Hard (`chrome://inspect`) | Medium (Application tab) |
| **Max Instances** | Many per page | 1 per URL+name | 1 per scope |

[⬆ Back to Top](#top)

---

## Choosing the Right Worker (Decision Guide)

### Decision Flowchart

```
                    What problem are you solving?
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        Heavy CPU work?   Cross-tab needs?  Network/Offline?
              │               │               │
              ▼               ▼               ▼
       ┌──────────┐    ┌──────────┐    ┌──────────┐
       │Web Worker │    │  Shared  │    │ Service  │
       │           │    │  Worker  │    │  Worker  │
       └──────────┘    └──────────┘    └──────────┘
              │               │               │
   One tab only?       Just messaging?   Need caching?
     Yes → Web Worker   Yes → BroadcastChannel  Yes → SW
     No → SharedWorker  Shared state? → SharedWorker
```

### Use **Web Worker** if:

- ✅ Heavy computation on **one page** (image processing, crypto, parsing)
- ✅ You want to keep the **UI responsive** during expensive operations
- ✅ No cross-tab coordination needed
- ✅ You need a **simple** setup

### Use **Shared Worker** if:

- ✅ Multiple tabs need to **share a WebSocket** connection
- ✅ You want **centralized in-memory state** across tabs
- ✅ You need to **deduplicate** backend connections or API calls
- ✅ Cross-tab coordination (e.g., "leader election" — one tab does polling)

### Use **Service Worker** if:

- ✅ **Offline-first** experience (PWA)
- ✅ **Smart caching** of assets and API responses
- ✅ **Push notifications**
- ✅ **Background sync** (retry failed operations)
- ✅ Network request **routing & transformation**

### Use **BroadcastChannel** if:

- ✅ You **only** need tab-to-tab messaging (no shared state, no shared connection)
- ✅ Simplest possible cross-tab communication

[⬆ Back to Top](#top)

---

## Real-World Usage Scenarios (with Code)

### 1. Figma — Web Workers for Rendering

Figma uses **Web Workers** for its canvas rendering engine. The main thread handles UI, while workers handle:
- Layout calculations
- Constraint solving
- Rendering to OffscreenCanvas

```js
// Simplified Figma-like rendering worker
// render-worker.js
self.onmessage = (e) => {
  const { type, canvas, scene } = e.data;

  if (type === 'render') {
    const ctx = canvas.getContext('2d');

    // Heavy rendering work — doesn't block main thread!
    for (const shape of scene.shapes) {
      ctx.fillStyle = shape.color;
      ctx.fillRect(shape.x, shape.y, shape.width, shape.height);
    }

    // Signal completion
    self.postMessage({ type: 'render:done', frameTime: performance.now() });
  }
};
```

```js
// main.js — Transfer OffscreenCanvas to worker
const canvas = document.getElementById('viewport');
const offscreen = canvas.transferControlToOffscreen();

const renderWorker = new Worker('/render-worker.js');

// Transfer canvas ownership to worker (zero-copy)
renderWorker.postMessage(
  { type: 'render', canvas: offscreen, scene: currentScene },
  [offscreen] // Transferable!
);
```

---

### 2. Trading Dashboard — SharedWorker for Real-Time Streams

A stock trading app opens in multiple tabs. **One WebSocket** feeds all of them:

```js
// trading-shared-worker.js
const ports = new Set();
let ws = null;
const latestPrices = new Map(); // Shared state: latest price per symbol

function broadcast(msg) {
  for (const p of ports) p.postMessage(msg);
}

function connectWS() {
  ws = new WebSocket('wss://stream.exchange.com/prices');

  ws.onmessage = (e) => {
    const tick = JSON.parse(e.data);
    latestPrices.set(tick.symbol, tick); // Update shared state
    broadcast({ type: 'tick', data: tick });
  };

  ws.onclose = () => setTimeout(connectWS, 2000);
}

self.onconnect = (e) => {
  const port = e.ports[0];
  ports.add(port);

  if (!ws) connectWS();

  // New tab gets ALL current prices immediately (no waiting!)
  port.postMessage({
    type: 'snapshot',
    prices: Object.fromEntries(latestPrices),
  });

  port.onmessage = (evt) => {
    if (evt.data.type === 'subscribe') {
      ws?.send(JSON.stringify({ action: 'sub', symbols: evt.data.symbols }));
    }
  };

  port.start();
};
```

---

### 3. E-Commerce PWA — Service Worker for Offline + Performance

```js
// sw.js — E-commerce caching strategy
self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);

  // Product images → Cache First (they rarely change)
  if (url.pathname.startsWith('/images/products/')) {
    event.respondWith(cacheFirst(event.request, 'product-images-v1'));
    return;
  }

  // Product catalog API → Stale While Revalidate
  if (url.pathname.startsWith('/api/products')) {
    event.respondWith(staleWhileRevalidate(event.request, 'api-cache-v1'));
    return;
  }

  // Cart / Checkout → Network Only (must be fresh)
  if (url.pathname.startsWith('/api/cart') || url.pathname.startsWith('/api/checkout')) {
    return; // Let it go to network normally
  }

  // App shell → Cache First
  event.respondWith(cacheFirst(event.request, 'app-shell-v1'));
});
```

---

### 4. Google Docs — All Three Workers Together

```
┌─────────────────────────────────────────────────┐
│  Google Docs Architecture (Simplified)           │
│                                                  │
│  Tab 1 (Document A)        Tab 2 (Document B)   │
│  ├── Web Worker             ├── Web Worker       │
│  │   (spell check,          │   (spell check,    │
│  │    formatting calc)       │    formatting)     │
│  │                          │                     │
│  └──── SharedWorker ◄───────┘                    │
│         │ (shared auth, OT sync,                  │
│         │  single WebSocket to Docs server)       │
│         │                                         │
│  Service Worker                                   │
│  ├── Caches documents for offline editing         │
│  ├── Background sync: pushes pending edits        │
│  └── Push notification: "Someone commented"       │
└─────────────────────────────────────────────────┘
```

[⬆ Back to Top](#top)

---

## Tradeoffs and Gotchas

### Web Worker

| ✅ Pros | ❌ Cons |
|---------|---------|
| Simplest to set up | Each tab creates its own (no sharing) |
| True parallel execution | Data must be cloned (cost for large objects) |
| Great debugging support | No DOM access (need to plan API carefully) |
| Any number per page | Each one = extra thread + memory |
| Works in all browsers | Can't intercept fetch or go offline |

**Gotcha**: Creating a new worker for every small task is expensive. **Pool workers** for repeated tasks:

```js
// Simple worker pool
class WorkerPool {
  #workers = [];
  #queue = [];
  #busy = new Set();

  constructor(url, size = navigator.hardwareConcurrency || 4) {
    for (let i = 0; i < size; i++) {
      this.#workers.push(new Worker(url));
    }
  }

  async exec(data) {
    const worker = await this.#getAvailable();
    return new Promise((resolve, reject) => {
      worker.onmessage = (e) => {
        this.#busy.delete(worker);
        this.#processQueue();
        resolve(e.data);
      };
      worker.onerror = (e) => {
        this.#busy.delete(worker);
        this.#processQueue();
        reject(e);
      };
      this.#busy.add(worker);
      worker.postMessage(data);
    });
  }

  #getAvailable() {
    const free = this.#workers.find((w) => !this.#busy.has(w));
    if (free) return Promise.resolve(free);
    return new Promise((resolve) => this.#queue.push(resolve));
  }

  #processQueue() {
    if (this.#queue.length === 0) return;
    const free = this.#workers.find((w) => !this.#busy.has(w));
    if (free) this.#queue.shift()(free);
  }

  terminate() {
    this.#workers.forEach((w) => w.terminate());
  }
}

// Usage
const pool = new WorkerPool('/compute-worker.js', 4);
const results = await Promise.all([
  pool.exec({ task: 'hash', data: file1 }),
  pool.exec({ task: 'hash', data: file2 }),
  pool.exec({ task: 'hash', data: file3 }),
  pool.exec({ task: 'hash', data: file4 }),
]);
```

---

### Shared Worker

| ✅ Pros | ❌ Cons |
|---------|---------|
| Single WebSocket for all tabs | Debugging is painful (`chrome://inspect`) |
| Shared in-memory state | Port-based API is more complex |
| Reduces server load | Limited Safari support (added iOS 16) |
| Efficient resource usage | No way to "force close" from one tab |
| Deduplicates work | Lifecycle: dies when ALL tabs close |

**Gotcha**: Port cleanup is tricky. There's no reliable `port.onclose` event in all browsers. Use heartbeats:

```js
// shared-worker.js — Heartbeat-based port cleanup
const ports = new Map(); // port → lastSeen timestamp

setInterval(() => {
  const now = Date.now();
  for (const [port, lastSeen] of ports) {
    if (now - lastSeen > 10000) {
      ports.delete(port);
      console.log('Cleaned up stale port');
    }
  }
}, 5000);

self.onconnect = (e) => {
  const port = e.ports[0];
  ports.set(port, Date.now());

  port.onmessage = (evt) => {
    ports.set(port, Date.now()); // Update heartbeat

    if (evt.data.type === 'heartbeat') return; // Just keep-alive
    // ... handle other messages
  };

  port.start();
};
```

```js
// main.js — Send heartbeats
const worker = new SharedWorker('/shared-worker.js');
worker.port.start();

setInterval(() => {
  worker.port.postMessage({ type: 'heartbeat' });
}, 5000);
```

---

### Service Worker

| ✅ Pros | ❌ Cons |
|---------|---------|
| Offline support (PWA) | Complex lifecycle (install/activate/wait) |
| Smart caching strategies | Cache invalidation is HARD |
| Push notifications | Can't store in-memory state (may be terminated) |
| Background sync | Requires HTTPS |
| Controls all pages of origin | Update propagation can confuse users |
| Intercepts all fetch requests | Debugging requires understanding lifecycle |

**Gotcha: The "Stuck on Old Version" Problem**

By default, a new Service Worker **waits** until all tabs using the old version are closed. This confuses users:

```js
// Fix 1: skipWaiting + clients.claim (aggressive update)
self.addEventListener('install', () => self.skipWaiting());
self.addEventListener('activate', () => self.clients.claim());

// Fix 2: Prompt user to update (better UX)
// In registration code:
reg.onupdatefound = () => {
  const newWorker = reg.installing;
  newWorker.onstatechange = () => {
    if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
      // New version available!
      if (confirm('New version available. Reload?')) {
        newWorker.postMessage({ type: 'skipWaiting' });
        window.location.reload();
      }
    }
  };
};
```

**Gotcha: Cache Storage is not unlimited**

Browsers can evict cached data under storage pressure. Always handle cache misses gracefully:

```js
// Always have a fallback!
async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;

  try {
    return await fetch(request);
  } catch {
    // Even network failed — return an offline page
    return caches.match('/offline.html');
  }
}
```

---

### Comparison of Trade-offs at a Glance

| Dimension | Web Worker | Shared Worker | Service Worker |
|-----------|-----------|---------------|----------------|
| **Setup Complexity** | 🟢 Low | 🟡 Medium | 🔴 High |
| **Debugging** | 🟢 Easy | 🔴 Hard | 🟡 Medium |
| **Browser Support** | 🟢 Universal | 🟡 Good (Safari 16+) | 🟢 Universal |
| **Memory Overhead** | 🟡 Per-tab | 🟢 Shared | 🟢 Shared |
| **Learning Curve** | 🟢 Low | 🟡 Medium | 🔴 Steep |
| **Power** | 🟡 Compute only | 🟡 Coordination | 🟢 Network + Offline |

[⬆ Back to Top](#top)

---

## Security and Compliance Considerations

### General Rules (All Workers)

1. **Never store secrets** (API keys, tokens) in worker code — it's downloadable JavaScript
2. **Validate ALL incoming messages** — any page on the origin can postMessage to your worker
3. **Use HTTPS** — required for Service Workers, recommended for all
4. **Same-origin policy** — workers can only be loaded from the same origin

### Service Worker Specific

5. **Never cache authenticated responses** — or you'll serve User A's data to User B
6. **Be careful with `clients.claim()`** — it can cause inconsistent behavior if new SW has different API expectations
7. **Set `Cache-Control` headers on SW file** — browsers check for updates; stale SW = stale app

### Shared Worker Specific

8. **Sanitize port messages** — malicious tabs on same origin can send crafted messages
9. **Implement rate limiting** — one rogue tab shouldn't overwhelm the shared worker

```js
// Message validation in any worker
self.onmessage = (e) => {
  // Validate structure
  if (!e.data || typeof e.data.type !== 'string') {
    console.warn('Invalid message format:', e.data);
    return;
  }

  // Whitelist allowed message types
  const ALLOWED = ['compute', 'subscribe', 'heartbeat'];
  if (!ALLOWED.includes(e.data.type)) {
    console.warn('Unknown message type:', e.data.type);
    return;
  }

  // Process valid message...
};
```

> For enterprise apps (e.g., Oracle, banking), align with internal security and compliance guidelines before adoption. Service Workers especially need review since they intercept all network traffic.

[⬆ Back to Top](#top)

---

## Final Recommendations

### Think in Responsibilities, Not APIs

| Responsibility | Worker |
|---------------|--------|
| **Compute** (heavy CPU) | Web Worker |
| **Coordination** (cross-tab state) | Shared Worker |
| **Network** (caching, offline, push) | Service Worker |
| **Simple cross-tab messaging** | BroadcastChannel |

### Production Checklist

- [ ] **Web Worker**: Use pool pattern for repeated tasks. Transfer large data, don't clone.
- [ ] **Shared Worker**: Implement heartbeat cleanup. Test in Safari. Plan for debugging.
- [ ] **Service Worker**: Version your caches. Handle updates gracefully. Test offline thoroughly.
- [ ] **All Workers**: Validate messages. Handle errors. Plan for graceful degradation.

### They Coexist — Use the Right One for Each Job

```js
// In a real production app, you might have ALL THREE:

// 1. Service Worker — registered once, intercepts network
navigator.serviceWorker.register('/sw.js');

// 2. Shared Worker — one per user across tabs
const shared = new SharedWorker('/shared-worker.js');
shared.port.start();

// 3. Web Worker — per-page for heavy computation
const compute = new Worker('/compute-worker.js');
compute.postMessage({ task: 'processImage', imageData });
```

> **The best architecture uses each worker for its intended purpose.** Don't try to make a Service Worker do CPU work, or a Web Worker do cross-tab coordination.

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli


[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design

[⬆ Back to Top](#top)
