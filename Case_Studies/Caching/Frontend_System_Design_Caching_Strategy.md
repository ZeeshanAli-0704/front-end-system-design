# Frontend System Design: Caching Strategy

- [Frontend System Design: Caching Strategy](#frontend-system-design-caching-strategy)
  - [1. Concept, Idea, Product Overview](#1-concept-idea-product-overview)
    - [1.1 Product Description](#11-product-description)
    - [1.2 Key User Personas](#12-key-user-personas)
    - [1.3 Core User Flows (High Level)](#13-core-user-flows-high-level)
  - [2. Requirements](#2-requirements)
    - [2.1 Functional Requirements](#21-functional-requirements)
    - [2.2 Non Functional Requirements](#22-non-functional-requirements)
  - [3. Scope Clarification (Interview Scoping)](#3-scope-clarification-interview-scoping)
    - [3.1 In Scope](#31-in-scope)
    - [3.2 Out of Scope](#32-out-of-scope)
    - [3.3 Assumptions](#33-assumptions)
  - [4. High Level Frontend Architecture](#4-high-level-frontend-architecture)
    - [4.1 Overall Approach](#41-overall-approach)
    - [4.2 Major Architectural Layers](#42-major-architectural-layers)
    - [4.3 External Integrations](#43-external-integrations)
  - [5. Component Design and Modularization](#5-component-design-and-modularization)
    - [5.1 Component Hierarchy](#51-component-hierarchy)
    - [5.2 Multi Layer Cache Architecture and Invalidation Strategy (Deep Dive)](#52-multi-layer-cache-architecture-and-invalidation-strategy-deep-dive)
      - [5.2.1 HTTP Cache and Cache Control Headers](#521-http-cache-and-cache-control-headers)
      - [5.2.2 Service Worker Cache with Workbox Strategies](#522-service-worker-cache-with-workbox-strategies)
      - [5.2.3 In Memory API Response Cache (Stale While Revalidate)](#523-in-memory-api-response-cache-stale-while-revalidate)
      - [5.2.4 IndexedDB for Structured Offline Data](#524-indexeddb-for-structured-offline-data)
      - [5.2.5 LocalStorage and SessionStorage Use Cases](#525-localstorage-and-sessionstorage-use-cases)
      - [5.2.6 Cache Invalidation Patterns and TTL Management](#526-cache-invalidation-patterns-and-ttl-management)
      - [5.2.7 Cache Warming and Prefetching](#527-cache-warming-and-prefetching)
      - [5.2.8 Cache Size Management and Eviction](#528-cache-size-management-and-eviction)
      - [5.2.9 Decision Matrix](#529-decision-matrix)
    - [5.3 Reusability Strategy](#53-reusability-strategy)
    - [5.4 Module Organization](#54-module-organization)
  - [6. High Level Data Flow Explanation](#6-high-level-data-flow-explanation)
    - [6.1 Initial Load Flow](#61-initial-load-flow)
    - [6.2 User Interaction Flow](#62-user-interaction-flow)
    - [6.3 Error and Retry Flow](#63-error-and-retry-flow)
  - [7. Data Modelling (Frontend Perspective)](#7-data-modelling-frontend-perspective)
    - [7.1 Core Data Entities](#71-core-data-entities)
    - [7.2 Data Shape](#72-data-shape)
    - [7.3 Entity Relationships](#73-entity-relationships)
    - [7.4 UI Specific Data Models](#74-ui-specific-data-models)
  - [8. State Management Strategy](#8-state-management-strategy)
    - [8.1 State Classification](#81-state-classification)
    - [8.2 State Ownership](#82-state-ownership)
    - [8.3 Persistence Strategy](#83-persistence-strategy)
  - [9. High Level API Design (Frontend POV)](#9-high-level-api-design-frontend-pov)
    - [9.1 Required APIs](#91-required-apis)
    - [9.2 Cache Aware Fetching Pipeline (Deep Dive)](#92-cache-aware-fetching-pipeline-deep-dive)
    - [9.3 Request and Response Structure](#93-request-and-response-structure)
    - [9.4 Error Handling and Status Codes](#94-error-handling-and-status-codes)
  - [10. Caching Strategy](#10-caching-strategy)
  - [11. CDN and Asset Optimization](#11-cdn-and-asset-optimization)
  - [12. Rendering Strategy](#12-rendering-strategy)
  - [13. Cross Cutting Non Functional Concerns](#13-cross-cutting-non-functional-concerns)
    - [13.1 Security](#131-security)
    - [13.2 Accessibility](#132-accessibility)
    - [13.3 Performance Optimization](#133-performance-optimization)
    - [13.4 Observability and Reliability](#134-observability-and-reliability)
  - [14. Edge Cases and Tradeoffs](#14-edge-cases-and-tradeoffs)
  - [15. Summary and Future Improvements](#15-summary-and-future-improvements)

---

## 1. Concept, Idea, Product Overview

### 1.1 Product Description

*   A frontend caching strategy is a **cross-cutting architectural concern** that applies to any web application — e-commerce, social feeds, dashboards, SaaS tools, and content platforms.
*   It defines how the client stores, retrieves, invalidates, and evicts data at multiple layers (HTTP cache, Service Worker, in-memory, browser storage) to minimize network requests, reduce latency, and enable offline access.
*   Target systems: any frontend application that makes API calls, loads static assets, or needs to function reliably on slow or unreliable networks.
*   Primary use case: reducing time-to-interactive (TTI), enabling instant navigation between pages, providing offline-capable experiences, and lowering backend load through intelligent client-side caching.

---

### 1.2 Key User Personas

*   **End User on Fast Network**: Expects instant page transitions, no duplicate loading spinners, and data that feels "already there." Benefits from in-memory and HTTP caching for sub-100ms repeat visits.
*   **End User on Slow or Offline Network**: Relies on Service Worker cache and IndexedDB for functional offline access. Expects the app to show stale data with a refresh option rather than a blank error screen.
*   **Developer / Platform Engineer**: Configures caching strategies per data type, monitors cache hit ratios, and debugs stale data issues. Needs clear invalidation APIs and observability tools.

---

### 1.3 Core User Flows (High Level)

*   **Repeat Page Visit (Primary Flow)**:
    1.  User visits a page for the first time → data fetched from server, cached in memory and HTTP cache.
    2.  User navigates away, then returns to the same page.
    3.  Cached data is served instantly (stale-while-revalidate) → page renders in < 50ms.
    4.  Background revalidation fetches fresh data from server → if changed, UI updates seamlessly.

*   **Offline Access (Secondary Flow)**:
    1.  User loads the app while online → critical data cached in Service Worker + IndexedDB.
    2.  Network drops → user continues using the app with cached data.
    3.  User takes an action (e.g., submits a form) → action queued locally.
    4.  Network restored → queued actions sync to server; caches revalidate.

*   **Cache Invalidation on Mutation (Secondary Flow)**:
    1.  User updates their profile (name, avatar).
    2.  Mutation succeeds → profile cache is invalidated.
    3.  Next read of profile data fetches fresh from server, re-caches.
    4.  All components displaying profile data reflect the update instantly.

---

## 2. Requirements

### 2.1 Functional Requirements

*   **Static Asset Caching**:
    *   Cache JavaScript bundles, CSS, fonts, and images with long-lived cache headers.
    *   Use content-hash-based filenames for immutable caching and instant invalidation on deploy.
*   **API Response Caching**:
    *   Cache GET API responses in memory for instant repeat access within a session.
    *   Support configurable TTL (time-to-live) per endpoint or data type.
    *   Stale-while-revalidate pattern: serve cached data instantly, revalidate in background.
*   **Offline Support**:
    *   Service Worker intercepts network requests and serves from cache when offline.
    *   Critical data (user profile, recent content, navigation) available offline via IndexedDB.
*   **Cache Invalidation**:
    *   Automatic invalidation when the user performs a mutation (POST, PUT, DELETE).
    *   Event-driven invalidation from server push (WebSocket/SSE) for multi-tab or multi-device consistency.
    *   Manual refresh button to force-fetch fresh data.
*   **User Preference Caching**:
    *   Theme, language, notification settings cached in localStorage for instant hydration on app boot.
*   **Prefetching and Cache Warming**:
    *   Prefetch likely-next-page data on hover or viewport proximity.
    *   Warm critical caches during idle periods.

---

### 2.2 Non Functional Requirements

*   **Performance**: Cached data served in < 5ms (memory) or < 50ms (IndexedDB/Service Worker). Cache hit ratio > 80% for repeat visits. No perceptible delay for stale-while-revalidate updates.
*   **Scalability**: Handle applications with 500+ unique API endpoints; manage cache storage within browser limits (50MB for Cache API, 5-10MB for localStorage, variable for IndexedDB).
*   **Availability**: The app must be functional (at minimum read-only) when the network is unavailable. Graceful fallback from failed revalidation.
*   **Security**: Never cache sensitive data (auth tokens, PII) in world-readable storage. Respect `Cache-Control: no-store` headers. Clear caches on logout.
*   **Consistency**: Stale data must not cause incorrect user actions (e.g., showing outdated inventory as "in stock"). Configurable freshness guarantees per data type.
*   **Device Support**: Desktop and mobile web. Low-end devices with limited storage.
*   **Debuggability**: Cache state inspectable via DevTools. Cache hit/miss logging for performance monitoring.

---

## 3. Scope Clarification (Interview Scoping)

### 3.1 In Scope

*   Multi-layer cache architecture (HTTP, Service Worker, in-memory, browser storage).
*   Cache-aware data fetching patterns (stale-while-revalidate, cache-first, network-first).
*   Cache invalidation strategies (mutation-based, TTL-based, event-driven).
*   Static asset caching with content hashing.
*   Offline data access via Service Worker and IndexedDB.
*   Prefetching and cache warming patterns.
*   Cache size management and eviction policies.

---

### 3.2 Out of Scope

*   Backend caching (Redis, CDN origin cache, reverse proxy caches).
*   Database query caching.
*   Full PWA implementation (manifest, install prompts, push notifications).
*   GraphQL-specific caching (Apollo normalized cache) — mentioned but not deep-dived.
*   Server-side rendering cache (ISR/SSG cache at the framework level).

---

### 3.3 Assumptions

*   The application is a modern SPA or SSR+CSR hybrid (React, Next.js, Vue, etc.).
*   APIs return standard HTTP cache headers (`Cache-Control`, `ETag`, `Last-Modified`).
*   The browser supports Service Worker, Cache API, and IndexedDB.
*   HTTPS is enforced (required for Service Worker).

---

## 4. High Level Frontend Architecture

### 4.1 Overall Approach

*   **Multi-layer caching** — each layer serves a specific purpose and speed profile.
*   **Cache-through pattern** — all network requests flow through the cache layer; the UI never directly calls `fetch()`.
*   **Configurable per data type** — static assets use Cache-First; API data uses Stale-While-Revalidate; user mutations bypass cache entirely.
*   The caching system is a **shared infrastructure module**, not per-feature code. Every feature consumes it through a unified API (React Query, SWR, or custom).

---

### 4.2 Major Architectural Layers

```
┌──────────────────────────────────────────────────────────────┐
│  UI Layer (React Components)                                 │
│  Components read data via hooks (useQuery, useSWR, etc.)     │
│  Components never call fetch() directly                      │
├──────────────────────────────────────────────────────────────┤
│  Data Fetching Layer (React Query / SWR / Custom)            │
│  ┌───────────────────────────────────────────────────────┐   │
│  │  In-Memory Cache (L1)                                 │   │
│  │  • Fastest: < 1ms read                                │   │
│  │  • Per-session, lost on tab close                     │   │
│  │  • Keyed by query key + params                        │   │
│  │  • Configurable staleTime / gcTime                    │   │
│  └───────────────────────────────────────────────────────┘   │
├──────────────────────────────────────────────────────────────┤
│  Service Worker Layer (L2)                                   │
│  ┌───────────────────────────────────────────────────────┐   │
│  │  Cache API (sw.js)                                    │   │
│  │  • Fast: < 10ms read                                  │   │
│  │  • Persists across sessions                           │   │
│  │  • Strategies: CacheFirst, NetworkFirst, StaleWR      │   │
│  │  • Workbox for declarative configuration              │   │
│  └───────────────────────────────────────────────────────┘   │
├──────────────────────────────────────────────────────────────┤
│  Browser Storage Layer (L3)                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌───────────────┐  │
│  │  IndexedDB      │  │  localStorage  │  │ sessionStorage│  │
│  │  • Structured   │  │  • Key-value   │  │ • Per-tab     │  │
│  │  • Large (GB)   │  │  • 5-10MB      │  │ • ~5MB        │  │
│  │  • Async API    │  │  • Sync API    │  │ • Sync API    │  │
│  │  • Offline data │  │  • Preferences │  │ • Temp state  │  │
│  └────────────────┘  └────────────────┘  └───────────────┘  │
├──────────────────────────────────────────────────────────────┤
│  HTTP Cache Layer (L4 — Browser Managed)                     │
│  ┌───────────────────────────────────────────────────────┐   │
│  │  Browser HTTP Cache                                   │   │
│  │  • Controlled by Cache-Control, ETag, Last-Modified   │   │
│  │  • Automatic — no JS code needed                      │   │
│  │  • Static assets: immutable + max-age=31536000        │   │
│  │  • API responses: max-age=0, must-revalidate + ETag   │   │
│  └───────────────────────────────────────────────────────┘   │
├──────────────────────────────────────────────────────────────┤
│  CDN Edge Cache (L5 — Server Infrastructure)                 │
│  ┌───────────────────────────────────────────────────────┐   │
│  │  CDN (Cloudfront / Akamai / Fastly)                   │   │
│  │  • Nearest edge PoP, ~5-50ms                          │   │
│  │  • JS/CSS/Image bundles                               │   │
│  │  • s-maxage for CDN, max-age for browser              │   │
│  └───────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

**Request flow through layers (cache lookup order):**

```
Component calls useQuery('user-profile')
       ↓
1. In-Memory Cache (L1) → HIT? Return instantly (<1ms). MISS? ↓
       ↓
2. Service Worker Cache (L2) → HIT? Return (<10ms). MISS? ↓
       ↓
3. HTTP Cache (L4 — browser checks ETag/Last-Modified) → HIT? Return. MISS? ↓
       ↓
4. CDN Edge (L5) → HIT? Return. MISS? ↓
       ↓
5. Origin Server → Fetch, return response, populate all layers on the way back
```

---

### 4.3 External Integrations

*   **CDN**: Cloudfront, Akamai, Fastly, or Vercel Edge — serves static assets and cacheable API responses.
*   **Service Worker Tooling**: Workbox (Google) for declarative caching strategies.
*   **Data Fetching Library**: React Query (TanStack Query), SWR, or Apollo Client — manages in-memory cache and revalidation.
*   **Analytics SDK**: Track cache hit/miss ratios, stale data rates, and offline usage metrics.
*   **Backend Services**: Must return proper HTTP cache headers (`Cache-Control`, `ETag`, `Vary`).

---

## 5. Component Design and Modularization

### 5.1 Component Hierarchy

```
App
 ├── CacheProvider (wraps entire app — React Query / SWR provider)
 │    ├── ServiceWorkerRegistration (registers sw.js on mount)
 │    └── CacheDebugPanel (DevTools overlay — dev only)
 │
 ├── Any Feature Page
 │    └── DataConsumingComponent
 │         └── uses useQuery / useSWR hooks
 │              → reads from in-memory cache (L1)
 │              → triggers background revalidation if stale
 │              → falls back to Service Worker cache (L2) on network error
 │
 ├── PrefetchManager (invisible — prefetches on hover/idle)
 │    └── uses queryClient.prefetchQuery on route links
 │
 └── OfflineBanner (shows "You're offline" when no network)
      └── reads navigator.onLine + Service Worker status
```

---

### 5.2 Multi Layer Cache Architecture and Invalidation Strategy (Deep Dive)

Caching is the **single most impactful performance optimization** a frontend can implement. Getting it right matters because:
*   The average web page makes **70+ HTTP requests**. Without caching, every navigation repeats all of them.
*   A cache HIT at the memory layer returns in **< 1ms** vs 200-500ms for a network round trip — a **200-500x improvement**.
*   Offline-capable apps require persistent caching; without it, any network interruption renders the app useless.
*   Incorrect caching (stale data shown as fresh) can cause **data integrity issues** — showing outdated prices, stale permissions, or wrong user data.
*   Cache invalidation is famously "one of the two hard things in computer science" — a well-designed strategy prevents the most common pitfalls.

---

#### 5.2.1 HTTP Cache and Cache Control Headers

##### How the Browser HTTP Cache Works

The HTTP cache is the **lowest-effort, highest-impact** caching layer. It's built into every browser, requires zero JavaScript, and is controlled entirely by response headers.

**The two fundamental caching models:**

| Model | Headers | How it works | Best for |
|-------|---------|-------------|----------|
| **Strong caching** (no revalidation) | `Cache-Control: max-age=31536000, immutable` | Browser caches the response for the specified duration. No server request is made until the cache expires. | Static assets with content hashes: `app.a1b2c3.js`, `styles.d4e5f6.css` |
| **Conditional caching** (revalidate every time) | `Cache-Control: no-cache` + `ETag` or `Last-Modified` | Browser always asks the server "has this changed?" via a conditional request. Server returns `304 Not Modified` (no body) if unchanged, saving bandwidth. | API responses, HTML documents |

**Common `Cache-Control` directives explained:**

| Directive | Meaning | When to use |
|-----------|---------|-------------|
| `max-age=N` | Cache for N seconds without revalidation | Static assets (long TTL), API responses (short TTL) |
| `s-maxage=N` | CDN-specific max-age (overrides max-age for CDN only) | When CDN and browser need different TTLs |
| `no-cache` | Must revalidate with server before using cached version | API data that changes frequently |
| `no-store` | Never cache this response anywhere | Sensitive data (auth tokens, PII, financial data) |
| `immutable` | Response will never change — don't even check | Content-hashed assets: `bundle.a1b2c3.js` |
| `private` | Only browser cache (not CDN/proxies) can store | User-specific API responses |
| `public` | CDN and proxies may cache | Shared assets, non-user-specific data |
| `must-revalidate` | Don't use stale version even if offline | Data where staleness is harmful |
| `stale-while-revalidate=N` | Serve stale for N seconds while revalidating in background | API responses where slight staleness is acceptable |

##### Content-Hashed Static Assets — The Best Pattern

Modern bundlers (Webpack, Vite, esbuild) generate filenames with content hashes:

```
dist/
  ├── app.a1b2c3d4.js         → content hash in filename
  ├── styles.e5f6g7h8.css
  ├── vendor.i9j0k1l2.js
  └── logo.m3n4o5p6.png
```

**Why this is powerful:**
*   The hash changes **only when the file content changes**.
*   You can set `Cache-Control: max-age=31536000, immutable` (cache for 1 year).
*   The browser **never** revalidates these files — they're cached until evicted.
*   When you deploy a new version, the HTML references a **new filename** (e.g., `app.q7r8s9.js`), so the browser fetches the new file and caches it.
*   Old files remain cached harmlessly and are eventually evicted by the browser's LRU policy.

```
Deploy v1: index.html → <script src="app.a1b2c3.js">
                                ↓
           Browser caches app.a1b2c3.js for 1 year

Deploy v2: index.html → <script src="app.x9y8z7.js">  ← NEW hash
                                ↓
           Browser fetches app.x9y8z7.js (cache miss)
           Old app.a1b2c3.js still cached but never referenced
```

##### The HTML File — The One Exception

The `index.html` file **must not be cached** with `max-age` because it's the entry point that references all hashed assets. If HTML is stale, it references old JS/CSS bundles:

```
index.html:
  Cache-Control: no-cache
  ETag: "v1-abc123"

This ensures:
  1. Browser always revalidates index.html with the server
  2. If unchanged → 304 Not Modified (very fast, no body transfer)
  3. If changed → new HTML with new asset references → browser fetches new bundles
```

##### Implementation — Server Configuration

```
# Nginx example

# Hashed static assets — cache forever
location /static/ {
    add_header Cache-Control "public, max-age=31536000, immutable";
}

# HTML entry point — always revalidate
location / {
    add_header Cache-Control "no-cache";
    add_header ETag $request_id;
}

# API responses — short cache with revalidation
location /api/ {
    add_header Cache-Control "private, no-cache";
    add_header Vary "Authorization, Accept-Language";
}
```

##### ETag Deep Dive — Conditional Requests

**What is an ETag?** An opaque string (fingerprint) that uniquely identifies a specific version of a resource.

```
First request:
  GET /api/user/profile
  → 200 OK
  → ETag: "abc123"
  → Cache-Control: no-cache
  → Body: { name: "Alice", avatar: "..." }

Second request (browser sends conditional request):
  GET /api/user/profile
  If-None-Match: "abc123"     ← "do you still have this version?"
  → 304 Not Modified          ← "yes, use your cached copy"
  → No body transferred       ← saves bandwidth

  OR if changed:
  → 200 OK
  → ETag: "def456"            ← new fingerprint
  → Body: { name: "Alice (updated)", avatar: "..." }
```

**Why ETags matter for API caching:**
*   The request still hits the server (latency), but the **response body is skipped** on 304 — saving bandwidth.
*   For large payloads (e.g., a 500KB product catalog), an ETag saves 500KB of transfer on every revalidation.
*   Combined with `stale-while-revalidate` in the Service Worker, you can serve the cached version immediately while the ETag check happens in the background.

---

#### 5.2.2 Service Worker Cache with Workbox Strategies

##### What the Service Worker Cache Layer Does

The Service Worker acts as a **programmable network proxy** between the browser and the network. It intercepts every `fetch()` request and can:
*   Serve a response from the Cache API (no network).
*   Forward the request to the network (normal behavior).
*   Serve from cache AND revalidate with network simultaneously (stale-while-revalidate).
*   Queue failed requests for later retry (background sync).

```
┌─────────────────────────────────────────────────────┐
│  Browser Main Thread                                │
│  fetch('/api/products')                             │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│  Service Worker (sw.js)                              │
│                                                      │
│  Intercepts the request via addEventListener('fetch')│
│                                                      │
│  Decision: Which strategy to use?                    │
│  ┌────────────┐  ┌────────────┐  ┌───────────────┐  │
│  │ CacheFirst │  │ NetworkFst │  │ StaleWhileRev │  │
│  │ (static)   │  │ (API)      │  │ (API)         │  │
│  └────────────┘  └────────────┘  └───────────────┘  │
│                                                      │
│  → Cache API (persistent, disk-backed)               │
│  → Network (if cache miss or revalidation needed)    │
└──────────────────────────────────────────────────────┘
```

##### The Five Workbox Strategies

| Strategy | Flow | Best For | Tradeoff |
|----------|------|----------|----------|
| **CacheFirst** | Check cache → if HIT, return. If MISS, fetch from network, cache, return. | Static assets (JS, CSS, images, fonts) | Stale if asset updates (solved by content hashing) |
| **NetworkFirst** | Try network → if success, cache + return. If fail, return from cache. | Fresh API data (user profile, order status) | Slower on slow networks (waits for network timeout) |
| **StaleWhileRevalidate** | Return from cache immediately (if exists) → fetch from network in background → update cache. | API data where slight staleness is OK (product listings, feeds) | User sees stale data briefly; UI may update after background fetch |
| **NetworkOnly** | Always fetch from network. Never cache. | Mutations (POST/PUT/DELETE), auth endpoints | No offline support; every request hits the network |
| **CacheOnly** | Return from cache. Never fetch network. | Pre-cached assets during install event | Fails if not in cache; useful for app shell assets |

##### Implementation with Workbox

```js
// sw.js (Service Worker)
import { registerRoute } from 'workbox-routing';
import {
  CacheFirst,
  NetworkFirst,
  StaleWhileRevalidate,
} from 'workbox-strategies';
import { ExpirationPlugin } from 'workbox-expiration';
import { CacheableResponsePlugin } from 'workbox-cacheable-response';
import { precacheAndRoute } from 'workbox-precaching';

// 1. Precache app shell (built assets injected by build tool)
precacheAndRoute(self.__WB_MANIFEST);

// 2. CacheFirst for static assets (images, fonts)
registerRoute(
  ({ request }) =>
    request.destination === 'image' ||
    request.destination === 'font',
  new CacheFirst({
    cacheName: 'static-assets',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 200,              // max 200 images/fonts cached
        maxAgeSeconds: 30 * 24 * 60 * 60, // 30 days
      }),
      new CacheableResponsePlugin({
        statuses: [0, 200],           // cache opaque responses too (cross-origin)
      }),
    ],
  })
);

// 3. StaleWhileRevalidate for API data (product listings, feed)
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/products') ||
               url.pathname.startsWith('/api/feed'),
  new StaleWhileRevalidate({
    cacheName: 'api-data',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 100,
        maxAgeSeconds: 5 * 60,       // 5 minutes max staleness
      }),
      new CacheableResponsePlugin({
        statuses: [200],              // only cache successful responses
      }),
    ],
  })
);

// 4. NetworkFirst for user-specific data (profile, cart, orders)
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/user') ||
               url.pathname.startsWith('/api/cart') ||
               url.pathname.startsWith('/api/orders'),
  new NetworkFirst({
    cacheName: 'user-data',
    networkTimeoutSeconds: 3,         // fall back to cache after 3s
    plugins: [
      new ExpirationPlugin({
        maxEntries: 50,
        maxAgeSeconds: 10 * 60,       // 10 minutes
      }),
    ],
  })
);

// 5. Never cache authentication endpoints
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/auth'),
  new NetworkOnly()
);
```

##### Why Service Worker Cache Survives Browser Restart

Unlike the in-memory cache (React Query), the Service Worker cache is **persistent**:
*   It uses the **Cache API**, which stores responses on disk.
*   The cache survives tab close, browser restart, and even system reboot.
*   The Service Worker itself persists — it starts up automatically when the user returns to the site.
*   This makes it ideal for offline support: even if the user has been away for days, the cached data is still there.

**Limitation:** Browsers set **storage quotas** per origin. Chrome allows up to ~60% of available disk space. If the quota is exceeded, the oldest caches are evicted (LRU).

---

#### 5.2.3 In Memory API Response Cache (Stale While Revalidate)

##### The Problem — API Data Needs the Fastest Possible Access

Service Worker cache provides offline persistence, but it's still a **disk read** (~5-10ms). For the absolute fastest repeat access within a session, we need **in-memory** caching — data stored as JavaScript objects in the app's heap.

This is what libraries like **React Query (TanStack Query)** and **SWR** provide out of the box.

##### How React Query's Cache Works

```
┌──────────────────────────────────────────────────────┐
│  React Query Cache (in-memory Map)                   │
│                                                      │
│  Key: ['user', 'profile']  → { data, staleTime, ... }
│  Key: ['products', { page: 1 }] → { data, ... }     │
│  Key: ['cart']              → { data, ... }          │
│                                                      │
│  On read (useQuery):                                 │
│   1. Check cache for key                             │
│   2. If fresh → return immediately (no fetch)        │
│   3. If stale → return cached + fetch in background  │
│   4. If missing → fetch and cache                    │
│                                                      │
│  On mutation (useMutation):                          │
│   1. Call API                                        │
│   2. Invalidate related cache keys                   │
│   3. Affected queries refetch automatically          │
└──────────────────────────────────────────────────────┘
```

##### Configuration Options Explained

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 2 * 60 * 1000,       // 2 minutes — data is "fresh" for this long
      gcTime: 10 * 60 * 1000,         // 10 minutes — unused data is garbage collected
      refetchOnWindowFocus: true,      // refetch when user returns to tab
      refetchOnReconnect: true,        // refetch when network is restored
      retry: 3,                        // retry failed requests 3 times
      retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30000),
    },
  },
});
```

| Option | Default | What it does | Why this value |
|--------|---------|-------------|----------------|
| `staleTime` | `0` | How long data is considered fresh (no refetch). | 2 min: balances freshness with reducing redundant API calls. Set to `Infinity` for truly static data. |
| `gcTime` | `5 min` | How long inactive (no subscribers) data stays in cache before garbage collection. | 10 min: user navigates away and back within 10 min → cache hit. Beyond 10 min, memory is freed. |
| `refetchOnWindowFocus` | `true` | When user returns to the tab, refetch stale queries. | Ensures data is fresh after the user was away (checking email, etc.). |
| `refetchOnReconnect` | `true` | When network is restored, refetch stale queries. | Critical after an offline period — data might have changed. |
| `retry` | `3` | Number of automatic retries on fetch failure. | 3 retries with exponential backoff covers most transient errors. |

##### Per-Query Configuration

Different data types need different cache policies:

```tsx
// User profile — rarely changes, cache for 10 minutes
const { data: profile } = useQuery({
  queryKey: ['user', 'profile'],
  queryFn: fetchUserProfile,
  staleTime: 10 * 60 * 1000,       // 10 min fresh
  gcTime: 30 * 60 * 1000,          // 30 min before GC
});

// Product listing — changes moderately, cache for 2 minutes
const { data: products } = useQuery({
  queryKey: ['products', { page, category }],
  queryFn: () => fetchProducts({ page, category }),
  staleTime: 2 * 60 * 1000,        // 2 min fresh
});

// Stock levels — changes frequently, always revalidate
const { data: stock } = useQuery({
  queryKey: ['stock', productId],
  queryFn: () => fetchStock(productId),
  staleTime: 0,                     // always stale (revalidate on every mount)
  refetchInterval: 30_000,          // also refetch every 30 seconds
});

// App config — rarely changes, cache for entire session
const { data: config } = useQuery({
  queryKey: ['app-config'],
  queryFn: fetchAppConfig,
  staleTime: Infinity,              // never stale within session
});
```

##### Mutation-Based Cache Invalidation

When the user performs a write operation, related cached data must be invalidated:

```tsx
const updateProfile = useMutation({
  mutationFn: (newProfile) => fetch('/api/user/profile', {
    method: 'PUT',
    body: JSON.stringify(newProfile),
  }),
  onSuccess: () => {
    // Invalidate the user profile cache — triggers automatic refetch
    queryClient.invalidateQueries({ queryKey: ['user', 'profile'] });
    // Also invalidate anything that shows the user's name/avatar
    queryClient.invalidateQueries({ queryKey: ['feed'] }); // feed shows user's posts
  },
});
```

**Optimistic updates** — update the cache before the server responds:

```tsx
const addToCart = useMutation({
  mutationFn: (item) => fetch('/api/cart', {
    method: 'POST',
    body: JSON.stringify(item),
  }),
  // Optimistically update the cart cache
  onMutate: async (newItem) => {
    await queryClient.cancelQueries({ queryKey: ['cart'] });
    const previousCart = queryClient.getQueryData(['cart']);

    queryClient.setQueryData(['cart'], (old) => ({
      ...old,
      items: [...old.items, { ...newItem, status: 'pending' }],
    }));

    return { previousCart }; // for rollback
  },
  // If mutation fails, rollback to previous cache
  onError: (_err, _newItem, context) => {
    queryClient.setQueryData(['cart'], context.previousCart);
  },
  // Always refetch cart data on success or error to ensure consistency
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['cart'] });
  },
});
```

---

#### 5.2.4 IndexedDB for Structured Offline Data

##### When to Use IndexedDB vs Other Storage

| Storage | Capacity | API | Persistence | Data Type | Best For |
|---------|----------|-----|-------------|-----------|----------|
| **In-memory** (React Query) | ~50-100MB | Sync (JS Object) | Session only | Any JS object | Active session data |
| **localStorage** | 5-10MB | Sync | Persistent | Strings only | Preferences, tokens |
| **sessionStorage** | 5-10MB | Sync | Tab only | Strings only | Temporary form data |
| **Cache API** (SW) | ~60% disk | Async | Persistent | Request/Response pairs | HTTP responses |
| **IndexedDB** | ~GB range | Async | Persistent | Structured objects | Offline data, large datasets |

##### Why IndexedDB for Offline Apps

IndexedDB is the only browser storage that supports:
*   **Large data volumes** (GBs of structured data).
*   **Indexed queries** (find records by any field, not just key).
*   **Transactions** (atomic reads and writes).
*   **Structured data** (objects, arrays, Blobs, Files — not just strings).

This makes it ideal for caching structured API data for offline access:

```tsx
// Using idb (a tiny Promise-based wrapper around IndexedDB)
import { openDB } from 'idb';

const dbPromise = openDB('app-cache', 1, {
  upgrade(db) {
    // Create object stores (like tables)
    const productStore = db.createObjectStore('products', { keyPath: 'id' });
    productStore.createIndex('category', 'category');
    productStore.createIndex('updatedAt', 'updatedAt');

    db.createObjectStore('conversations', { keyPath: 'id' });
    db.createObjectStore('user-profile', { keyPath: 'userId' });
  },
});

// Write products to IndexedDB (after successful API fetch)
async function cacheProducts(products: Product[]) {
  const db = await dbPromise;
  const tx = db.transaction('products', 'readwrite');
  await Promise.all([
    ...products.map((p) => tx.store.put(p)),
    tx.done,
  ]);
}

// Read products from IndexedDB (when offline)
async function getCachedProducts(category?: string): Promise<Product[]> {
  const db = await dbPromise;
  if (category) {
    return db.getAllFromIndex('products', 'category', category);
  }
  return db.getAll('products');
}

// Usage with React Query — IndexedDB as fallback
const { data: products } = useQuery({
  queryKey: ['products', category],
  queryFn: async () => {
    try {
      const response = await fetch(`/api/products?category=${category}`);
      const data = await response.json();
      // Cache in IndexedDB for offline access
      await cacheProducts(data.products);
      return data.products;
    } catch (error) {
      // Network failed — try IndexedDB
      const cached = await getCachedProducts(category);
      if (cached.length > 0) return cached;
      throw error; // truly no data available
    }
  },
});
```

##### Syncing IndexedDB with Server

When the user comes back online, IndexedDB data may be stale. The sync strategy:

```
User returns online (navigator.onLine === true)
       ↓
1. App detects online status via 'online' event
       ↓
2. Fetch fresh data from server for critical entities
       ↓
3. Compare server data with IndexedDB data
   (by updatedAt timestamp or version number)
       ↓
4. Update IndexedDB with fresh data
       ↓
5. Notify React Query cache to refetch (invalidateQueries)
       ↓
6. UI updates with fresh data
```

---

#### 5.2.5 LocalStorage and SessionStorage Use Cases

##### What Belongs in localStorage

localStorage is **synchronous, persistent, and limited to 5-10MB**. Use it for small, frequently-accessed data that must survive page reloads:

| Data | Key | Example Value | Why localStorage |
|------|-----|--------------|------------------|
| **User preferences** | `theme` | `"dark"` | Prevent FOUC (flash of unstyled content) on load |
| **Locale** | `locale` | `"en-US"` | Hydrate i18n before API responds |
| **Auth state flag** | `isLoggedIn` | `"true"` | Show correct UI shell instantly (not the token itself!) |
| **Onboarding state** | `onboarding-complete` | `"true"` | Don't show onboarding again |
| **Last visited route** | `lastRoute` | `"/dashboard"` | Restore navigation on app reopen |
| **Draft content** | `draft:post-123` | `"Hello world..."` | Preserve unsaved work |
| **Feature flags (cache)** | `feature-flags` | `{"newUI": true}` | Show correct features before server responds |
| **Unread count** | `unreadCount` | `"5"` | Show badge instantly on app boot |

```tsx
// Hydrate theme before React renders to prevent FOUC
// This runs in a <script> tag in <head>, before any React code
const theme = localStorage.getItem('theme') || 'light';
document.documentElement.setAttribute('data-theme', theme);
```

##### What Belongs in sessionStorage

sessionStorage is **per-tab and cleared on tab close**. Use it for temporary data within a single browsing session:

| Data | Use Case |
|------|----------|
| **Form wizard state** | Multi-step form progress (lost on tab close is OK) |
| **Scroll positions** | Restore scroll on back navigation within the session |
| **One-time banners** | "Welcome back" banner shown once per session |
| **Temporary filters** | Search filters applied during this session only |

##### What Does NOT Belong in localStorage

| Data | Why NOT |
|------|---------|
| **Auth tokens** (JWT, access token) | Readable by any JS on the page — XSS can steal it. Use `httpOnly` cookies instead. |
| **Large datasets** (> 1MB) | Synchronous API blocks the main thread. Use IndexedDB. |
| **Sensitive PII** | localStorage is not encrypted and persists indefinitely. |
| **Frequently updated data** | Sync API + JSON.parse/stringify on every read/write is slow. Use in-memory. |

---

#### 5.2.6 Cache Invalidation Patterns and TTL Management

##### The Three Invalidation Triggers

```
┌───────────────────────────────────────────────────────────┐
│  1. TIME-BASED (TTL)                                      │
│     Data expires after a fixed duration.                  │
│     staleTime: 2min, maxAgeSeconds: 300                   │
│     ✅ Simple   ❌ Can serve stale until TTL expires     │
├───────────────────────────────────────────────────────────┤
│  2. MUTATION-BASED (Event-Driven)                         │
│     Cache invalidated when user performs a write action.  │
│     invalidateQueries(['cart']) after addToCart mutation  │
│     ✅ Precise   ❌ Only handles current user's mutations│
├───────────────────────────────────────────────────────────┤
│  3. SERVER-PUSH (Cross-Device/Tab)                        │
│     Server notifies client of data change via             │
│     WebSocket/SSE → client invalidates affected caches.   │
│     ✅ Real-time   ❌ Requires server infrastructure     │
└───────────────────────────────────────────────────────────┘
```

##### Pattern 1: TTL-Based Invalidation

The simplest approach — data expires automatically after a configured duration.

```tsx
// React Query: staleTime controls TTL
useQuery({
  queryKey: ['products'],
  queryFn: fetchProducts,
  staleTime: 5 * 60 * 1000,  // fresh for 5 minutes
});
```

**Choosing the right TTL per data type:**

| Data Type | Recommended TTL | Reasoning |
|-----------|----------------|-----------|
| **App config** (feature flags, UI config) | 30 min – `Infinity` | Rarely changes; re-deploy or flag change triggers manual invalidation |
| **User profile** | 5–10 min | Changes infrequently; user expects their own changes to reflect instantly (mutation-based) |
| **Product catalog** | 2–5 min | Changes moderately; slight staleness acceptable (price changes handled by checkout validation) |
| **Search results** | 1–2 min | Ephemeral; re-search is expected to produce fresh results |
| **Stock levels / Inventory** | 0 (always revalidate) | Changes constantly; stale stock causes user frustration ("added to cart but out of stock") |
| **Social feed** | 1–2 min | Stale-while-revalidate is fine; "new posts" banner handles real-time updates |
| **Chat messages** | 0 (always fresh via WebSocket) | Real-time by nature; cached only for offline/history access |

##### Pattern 2: Mutation-Based Invalidation

When the user changes data, invalidate related caches immediately:

```tsx
// When user updates their name, invalidate all caches that show the name
const updateName = useMutation({
  mutationFn: (name) => api.updateProfile({ name }),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['user'] });
    queryClient.invalidateQueries({ queryKey: ['feed'] }); // feed shows user's avatar + name
    queryClient.invalidateQueries({ queryKey: ['comments'] }); // comments show user's name
  },
});
```

**Partial cache update (better than invalidation for UX):**

```tsx
onSuccess: (updatedProfile) => {
  // Instead of invalidating (which causes refetch + loading state),
  // directly update the cache with the mutation response
  queryClient.setQueryData(['user', 'profile'], updatedProfile);
};
```

##### Pattern 3: Server-Push Invalidation

For multi-tab or multi-device consistency, the server pushes invalidation events:

```tsx
// Listen for server-sent cache invalidation events via WebSocket
useEffect(() => {
  const ws = new WebSocket('wss://api.example.com/events');

  ws.onmessage = (event) => {
    const { type, keys } = JSON.parse(event.data);

    if (type === 'cache.invalidate') {
      // Server says these cache keys are stale
      keys.forEach((key: string[]) => {
        queryClient.invalidateQueries({ queryKey: key });
      });
    }
  };

  return () => ws.close();
}, []);
```

**When this matters:**
*   User updates profile on their phone → WebSocket push to desktop tab → desktop tab refetches profile.
*   Admin changes product price → push to all connected clients → product listings update.
*   Another user sends a message → push to recipient → conversation cache updated.

##### Cross-Tab Invalidation (BroadcastChannel)

Without server push, two open tabs can have divergent caches:

```tsx
// Tab A: user updates profile → invalidate cache in Tab A
// Tab B: still showing old profile

// Solution: BroadcastChannel to sync invalidation across tabs
const channel = new BroadcastChannel('cache-invalidation');

// In Tab A (after mutation):
channel.postMessage({ type: 'invalidate', keys: [['user', 'profile']] });

// In all tabs (listener):
channel.onmessage = (event) => {
  if (event.data.type === 'invalidate') {
    event.data.keys.forEach((key: string[]) => {
      queryClient.invalidateQueries({ queryKey: key });
    });
  }
};
```

---

#### 5.2.7 Cache Warming and Prefetching

##### The Idea — Load Data Before the User Needs It

Cache warming means **proactively populating the cache** so that when the user navigates to a page, the data is already there.

```
Without prefetching:
  User clicks "Products" → fetch starts → 300ms loading → data renders

With prefetching:
  User hovers over "Products" → prefetch starts in background
  User clicks "Products" → data already cached → instant render (< 10ms)
```

##### Strategy 1: Prefetch on Hover

```tsx
function NavLink({ to, queryKey, queryFn, children }) {
  const queryClient = useQueryClient();

  const prefetch = () => {
    queryClient.prefetchQuery({
      queryKey,
      queryFn,
      staleTime: 5 * 60 * 1000, // don't re-prefetch if already cached
    });
  };

  return (
    <Link
      to={to}
      onMouseEnter={prefetch}  // start fetching when cursor hovers
      onFocus={prefetch}       // also on keyboard focus (accessibility)
    >
      {children}
    </Link>
  );
}

// Usage
<NavLink
  to="/products"
  queryKey={['products', { page: 1 }]}
  queryFn={() => fetchProducts({ page: 1 })}
>
  Products
</NavLink>
```

**Why hover, not click?** The average hover-to-click time is **200-400ms** — enough to complete most API requests. By the time the user clicks, the data is already cached.

##### Strategy 2: Prefetch on Viewport Proximity (IntersectionObserver)

For content below the fold (e.g., a "Related Products" section), prefetch when the section is about to scroll into view:

```tsx
function usePrefetchOnVisible(queryKey, queryFn) {
  const ref = useRef<HTMLDivElement>(null);
  const queryClient = useQueryClient();

  useEffect(() => {
    if (!ref.current) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          queryClient.prefetchQuery({ queryKey, queryFn });
          observer.disconnect(); // only prefetch once
        }
      },
      { rootMargin: '200px' } // prefetch 200px before visible
    );

    observer.observe(ref.current);
    return () => observer.disconnect();
  }, [queryKey, queryFn, queryClient]);

  return ref;
}
```

##### Strategy 3: Idle-Time Warming

Prefetch critical data during browser idle periods:

```tsx
function useCacheWarming() {
  useEffect(() => {
    // Wait until the browser is idle (no user interaction, no rendering work)
    if ('requestIdleCallback' in window) {
      const id = requestIdleCallback(() => {
        // Warm caches for likely-next-needed data
        queryClient.prefetchQuery({
          queryKey: ['user', 'profile'],
          queryFn: fetchUserProfile,
        });
        queryClient.prefetchQuery({
          queryKey: ['notifications', 'count'],
          queryFn: fetchNotificationCount,
        });
      }, { timeout: 5000 }); // ensure it runs within 5s even if never idle

      return () => cancelIdleCallback(id);
    }
  }, []);
}
```

---

#### 5.2.8 Cache Size Management and Eviction

##### The Problem — Caches Grow Unbounded

Without eviction, caches grow indefinitely:
*   In-memory: Every unique query key adds an entry. A paginated product listing generates `['products', {page: 1}]`, `['products', {page: 2}]`, ... `['products', {page: 100}]` — 100 cache entries.
*   Service Worker Cache API: Every cached response stays until manually deleted or quota is exceeded.
*   IndexedDB: Data accumulates across sessions unless explicitly cleaned.

##### Eviction Strategies

| Layer | Eviction Strategy | Configuration |
|-------|------------------|---------------|
| **In-memory (React Query)** | GC after `gcTime` when no active subscribers | `gcTime: 10 * 60 * 1000` (10 min) |
| **Service Worker (Workbox)** | LRU by `maxEntries` or TTL by `maxAgeSeconds` | `maxEntries: 100, maxAgeSeconds: 86400` |
| **IndexedDB** | Custom: delete records older than N days | Manual cleanup in `requestIdleCallback` |
| **localStorage** | Manual: LRU wrapper or key-prefix cleanup | Custom `LRUStorage` class |
| **HTTP Cache** | Browser-managed LRU (no control) | Controlled indirectly via `max-age` |

##### Implementing a Bounded LRU Cache for localStorage

```tsx
class LRUStorage {
  private prefix: string;
  private maxEntries: number;
  private orderKey: string;

  constructor(prefix: string, maxEntries: number) {
    this.prefix = prefix;
    this.maxEntries = maxEntries;
    this.orderKey = `${prefix}:__order__`;
  }

  get(key: string): string | null {
    const fullKey = `${this.prefix}:${key}`;
    const value = localStorage.getItem(fullKey);
    if (value !== null) {
      this.touch(key); // move to most-recently-used
    }
    return value;
  }

  set(key: string, value: string): void {
    const fullKey = `${this.prefix}:${key}`;
    localStorage.setItem(fullKey, value);
    this.touch(key);
    this.evict();
  }

  private touch(key: string): void {
    const order = this.getOrder();
    const filtered = order.filter((k) => k !== key);
    filtered.push(key); // most recent at end
    localStorage.setItem(this.orderKey, JSON.stringify(filtered));
  }

  private evict(): void {
    const order = this.getOrder();
    while (order.length > this.maxEntries) {
      const oldest = order.shift()!;
      localStorage.removeItem(`${this.prefix}:${oldest}`);
    }
    localStorage.setItem(this.orderKey, JSON.stringify(order));
  }

  private getOrder(): string[] {
    try {
      return JSON.parse(localStorage.getItem(this.orderKey) || '[]');
    } catch {
      return [];
    }
  }
}

// Usage
const draftCache = new LRUStorage('drafts', 20); // max 20 drafts
draftCache.set('post-123', 'Hello world...');
```

##### IndexedDB Cleanup

```tsx
async function cleanupOldCachedData() {
  const db = await dbPromise;
  const cutoff = Date.now() - 7 * 24 * 60 * 60 * 1000; // 7 days ago

  const tx = db.transaction('products', 'readwrite');
  const index = tx.store.index('updatedAt');
  let cursor = await index.openCursor();

  while (cursor) {
    if (new Date(cursor.value.updatedAt).getTime() < cutoff) {
      await cursor.delete();
    }
    cursor = await cursor.continue();
  }
}

// Run cleanup during idle time
if ('requestIdleCallback' in window) {
  requestIdleCallback(() => cleanupOldCachedData());
}
```

---

#### 5.2.9 Decision Matrix

| Data Type | Cache Layer | Strategy | TTL | Invalidation | Notes |
|-----------|-----------|----------|-----|-------------|-------|
| **JS/CSS bundles** | HTTP Cache + CDN | CacheFirst (immutable) | 1 year (`max-age=31536000`) | Content hash in filename | Never revalidated; new deploy = new filename |
| **Images, fonts** | HTTP Cache + SW (CacheFirst) | CacheFirst | 30 days | URL change or manual purge | SW limits entries with `maxEntries` |
| **HTML entry point** | HTTP Cache (no-cache + ETag) | NetworkFirst | 0 (always revalidate) | Every request checks ETag | Must be fresh to reference correct JS bundles |
| **Product catalog** | In-memory + SW (StaleWR) | StaleWhileRevalidate | 2-5 min | Mutation + TTL | Show cached, update in background |
| **User profile** | In-memory | StaleWhileRevalidate | 5-10 min | Mutation-based | Invalidate on profile update |
| **Stock levels** | In-memory (no cache) | NetworkFirst | 0 | Always fetch fresh | Staleness causes UX problems |
| **User preferences** | localStorage | Read on boot | Persistent | On explicit save | Prevents FOUC; tiny data |
| **Drafts** | localStorage (LRU) | Write-through | Persistent | On send or explicit discard | Bounded by LRU eviction |
| **Offline data** | IndexedDB | Write-behind (async) | 7 days | Sync on reconnect | Large structured data |
| **Auth tokens** | httpOnly cookie | Server-managed | Session or refresh TTL | Server-side | Never in localStorage! |
| **App config / flags** | In-memory + localStorage | StaleWhileRevalidate | 30 min | Server push or deploy | localStorage for instant boot |

---

### 5.3 Reusability Strategy

*   **`useCachedQuery`**: A wrapper around `useQuery` with sensible defaults per data category (static, dynamic, real-time). Configures `staleTime`, `gcTime`, and retry based on data classification.
*   **`ServiceWorkerManager`**: Shared module that registers the SW, handles updates, and exposes cache status.
*   **`OfflineFallback`**: Higher-order component or wrapper that shows cached content when network is unavailable with an "Offline" banner.
*   **`PrefetchLink`**: Drop-in replacement for `<Link>` that prefetches the target page's data on hover/focus.
*   **`LRUStorage`**: Reusable bounded localStorage wrapper for any feature (drafts, recent searches, notification count).

---

### 5.4 Module Organization

```
src/
 ├── cache/
 │    ├── config/
 │    │    ├── cacheConfig.ts         (TTL and strategy per data type)
 │    │    └── queryClientConfig.ts   (React Query default options)
 │    ├── service-worker/
 │    │    ├── sw.ts                  (Workbox strategies)
 │    │    ├── swRegistration.ts      (registration + update handling)
 │    │    └── swBridge.ts            (postMessage communication)
 │    ├── storage/
 │    │    ├── indexedDB.ts           (idb wrapper + object stores)
 │    │    ├── lruStorage.ts          (bounded localStorage)
 │    │    └── storageCleanup.ts      (idle-time eviction)
 │    ├── hooks/
 │    │    ├── useCachedQuery.ts      (smart defaults per data type)
 │    │    ├── usePrefetch.ts         (hover + viewport prefetch)
 │    │    ├── useOfflineStatus.ts    (network state)
 │    │    └── useCacheWarming.ts     (idle-time warming)
 │    ├── invalidation/
 │    │    ├── mutationInvalidation.ts (query key mapping)
 │    │    ├── broadcastSync.ts       (cross-tab via BroadcastChannel)
 │    │    └── serverPushSync.ts      (WebSocket invalidation)
 │    └── types/
 │         └── cache.types.ts
 ├── shared/
 │    ├── components/ (OfflineBanner, PrefetchLink)
 │    └── providers/ (CacheProvider wrapping QueryClientProvider)
```

---

## 6. High Level Data Flow Explanation

### 6.1 Initial Load Flow

```
1. Browser requests index.html
     ↓
2. HTML served with no-cache + ETag → browser checks freshness with server
     ↓
3. HTML references hashed JS/CSS bundles → browser checks HTTP cache
   → HIT: served from disk cache (0ms network) | MISS: fetched from CDN
     ↓
4. App boots → React QueryClientProvider mounts with default cache config
     ↓
5. Service Worker registers (if not already active)
     ↓
6. Critical data loaded:
   a. localStorage: theme, locale, isLoggedIn → hydrate app shell instantly
   b. useQuery('user-profile') → check in-memory (miss) → fetch from server → cache
   c. useQuery('notifications-count') → same flow
     ↓
7. Idle-time cache warming: prefetch likely-next-needed data
```

---

### 6.2 User Interaction Flow

```
User navigates to /products (already visited earlier this session)
     ↓
1. useQuery(['products', { page: 1 }]) fires
     ↓
2. In-memory cache HIT (data < 2 min old) → return instantly (< 1ms)
     ↓
3. If data > 2 min old → return stale data + refetch in background
     ↓
4. Background fetch completes → cache updated → components re-render with fresh data
     ↓
5. If background fetch fails → stale data remains displayed + error logged
```

**Mutation flow:**

```
User adds product to cart
     ↓
1. Optimistic: cache updated immediately (cart shows new item)
     ↓
2. POST /api/cart → server processes
     ↓
3. Success → invalidate ['cart'] cache key → refetch confirms
   OR
   Failure → rollback optimistic update → show error toast
```

---

### 6.3 Error and Retry Flow

*   **Network failure during fetch**: React Query retries 3 times with exponential backoff (1s, 2s, 4s). If all fail, serve stale data from cache (if available) + show error indicator.
*   **Service Worker cache miss + network offline**: Display "You're offline" banner. Show last-known data from IndexedDB. Queue mutations for background sync.
*   **ETag revalidation fails**: Serve the cached response (if `stale-while-revalidate` is configured). Log the failure.
*   **IndexedDB storage quota exceeded**: Evict oldest data (LRU). Log a warning for monitoring.
*   **Cache corruption**: Detect via try-catch on read. On corruption, clear the affected cache and refetch. Never crash the app due to a cache read error.

---

## 7. Data Modelling (Frontend Perspective)

### 7.1 Core Data Entities

*   **CacheEntry** — a single cached piece of data (query result, response, or stored value)
*   **CacheKey** — the unique identifier for a cache entry (array or string)
*   **CachePolicy** — the configuration (TTL, strategy, invalidation rules) for a data category
*   **InvalidationEvent** — a trigger to mark cache entries as stale

---

### 7.2 Data Shape

```ts
type CachePolicy = {
  staleTime: number;          // ms — how long data is considered fresh
  gcTime: number;             // ms — how long to keep inactive entries
  strategy: 'cache-first' | 'network-first' | 'stale-while-revalidate' | 'network-only';
  maxEntries?: number;        // for bounded caches (SW, localStorage)
  maxAge?: number;            // ms — absolute maximum cache lifetime
  revalidateOnFocus: boolean;
  revalidateOnReconnect: boolean;
  retryCount: number;
  retryDelay: number;         // ms — base delay for exponential backoff
};

type CacheEntry<T> = {
  key: string[];
  data: T;
  fetchedAt: number;          // timestamp of last fetch
  staleAt: number;            // timestamp when data becomes stale
  expiresAt: number;          // timestamp when data is evicted
  etag?: string;              // server ETag for conditional requests
  isStale: boolean;           // computed: Date.now() > staleAt
};

type InvalidationEvent = {
  type: 'mutation' | 'ttl-expired' | 'server-push' | 'manual';
  affectedKeys: string[][];   // list of cache keys to invalidate
  timestamp: number;
};

type StorageQuota = {
  usage: number;              // bytes used
  quota: number;              // bytes available
  percentUsed: number;        // computed
};
```

---

### 7.3 Entity Relationships

*   **One-to-Many**: One CachePolicy → many CacheEntries (a policy applies to a category of data).
*   **One-to-Many**: One InvalidationEvent → many CacheEntries (a mutation can invalidate multiple cache keys).
*   **Many-to-Many**: CacheEntries can exist in multiple layers simultaneously (in-memory + Service Worker + IndexedDB for the same data).

---

### 7.4 UI Specific Data Models

```ts
// Cache status for developer tools / debug panel
type CacheStatus = {
  layer: 'memory' | 'service-worker' | 'indexeddb' | 'localstorage' | 'http';
  entryCount: number;
  totalSizeBytes: number;
  hitCount: number;
  missCount: number;
  hitRatio: number;           // computed: hitCount / (hitCount + missCount)
};

// Per-query cache metadata (exposed in DevTools)
type QueryCacheInfo = {
  queryKey: string[];
  status: 'fresh' | 'stale' | 'fetching' | 'error' | 'none';
  dataUpdatedAt: number;      // last successful fetch time
  fetchCount: number;
  cacheHitCount: number;
  lastError?: string;
};
```

---

## 8. State Management Strategy

### 8.1 State Classification

| State Type | Examples | Storage |
|---|---|---|
| **Cached Server Data** | API responses (products, profile, feed) | React Query in-memory cache |
| **Persistent Caches** | Offline data, Service Worker cache entries | IndexedDB, Cache API |
| **User Preferences** | Theme, locale, sidebar state | localStorage |
| **Session State** | Form wizard progress, scroll positions | sessionStorage |
| **Cache Metadata** | Hit/miss counts, storage usage | Zustand store (dev only) |
| **Network State** | Online/offline, WebSocket status | Zustand or React context |
| **Derived State** | isStale, isFromCache, cacheAge | Computed selectors |

---

### 8.2 State Ownership

*   **React Query** owns all server-state caches (API data). Components subscribe via `useQuery` hooks. Mutations invalidate through `queryClient`.
*   **Service Worker** owns persistent response caches. Communicates with main thread via `postMessage`.
*   **IndexedDB module** owns structured offline data. Exposed through async helper functions.
*   **localStorage** is read directly by early-boot scripts (theme, locale) and managed by feature-specific modules.
*   **Network status** is owned by a global context/store, updated by `navigator.onLine` events.

---

### 8.3 Persistence Strategy

*   **In-memory (React Query)**: Per-session. Lost on tab close. Re-populated from server on next visit(fastest access).
*   **Service Worker Cache API**: Persists across sessions. Survives browser restart. Evicted by browser under storage pressure or by Workbox expiration rules.
*   **IndexedDB**: Persists across sessions. Large capacity. Used for structured offline data.
*   **localStorage**: Persists across sessions. Small capacity (5-10MB). Synchronous. Used for preferences and small flags.
*   **sessionStorage**: Per-tab. Cleared on tab close. Temporary state.
*   **HTTP Cache**: Browser-managed. Heuristic eviction. Controlled by response headers.

---

## 9. High Level API Design (Frontend POV)

### 9.1 Required APIs

| API | Method | Cache Strategy | Notes |
|-----|--------|---------------|-------|
| `/api/user/profile` | **GET** | StaleWhileRevalidate (10 min) | Invalidated on profile mutation |
| `/api/products?page=N` | **GET** | StaleWhileRevalidate (2 min) | Prefetched on hover; paginated |
| `/api/cart` | **GET** | NetworkFirst (0 staleTime) | Optimistic updates on mutations |
| `/api/notifications/count` | **GET** | StaleWhileRevalidate (1 min) | Server push updates via WebSocket |
| `/api/app-config` | **GET** | CacheFirst (30 min) | Cached in localStorage for boot |
| `/api/search?q=X` | **GET** | StaleWhileRevalidate (1 min) | Short TTL; new search = new key |

---

### 9.2 Cache Aware Fetching Pipeline (Deep Dive)

#### The Unified Fetch Pipeline

Instead of each component calling `fetch()` directly, all data access flows through a **cache-aware pipeline**:

```
Component → useQuery(key, fn) → React Query Cache (L1) → Service Worker (L2)
                                     ↓ (miss)                ↓ (miss)
                              Background Fetch  ←───── HTTP Cache (L4)
                                     ↓ (miss)                ↓ (miss)
                              Origin Server  ←───────── CDN Edge (L5)
                                     ↓
                              Response flows back through all layers
                              Each layer caches the response
```

#### Implementing the Pipeline

```tsx
// Unified fetch function that integrates all cache layers
async function cachedFetch<T>(
  url: string,
  options: {
    cachePolicy: CachePolicy;
    indexedDBStore?: string;       // also persist to IndexedDB
    localStorageKey?: string;     // also persist to localStorage
  }
): Promise<T> {
  const { cachePolicy, indexedDBStore, localStorageKey } = options;

  // L1: Check in-memory (handled by React Query — this function is the queryFn)

  // L3: Check IndexedDB if configured (for offline support)
  if (indexedDBStore) {
    try {
      const cached = await getFromIndexedDB(indexedDBStore, url);
      if (cached && !isExpired(cached, cachePolicy.maxAge)) {
        // Return IndexedDB data, but also trigger background revalidation
        return cached.data;
      }
    } catch {
      // IndexedDB error — continue to network
    }
  }

  // L2/L4/L5: Fetch from network (Service Worker intercepts → HTTP cache → CDN → Origin)
  const response = await fetch(url);
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  const data = await response.json();

  // Persist to IndexedDB for offline access (async, non-blocking)
  if (indexedDBStore) {
    putToIndexedDB(indexedDBStore, url, data).catch(() => {});
  }

  // Persist to localStorage for instant boot (small data only)
  if (localStorageKey) {
    try {
      localStorage.setItem(localStorageKey, JSON.stringify(data));
    } catch {
      // localStorage full — ignore
    }
  }

  return data;
}
```

#### Cache Key Design

Good cache keys are **unique, predictable, and hierarchical**:

```tsx
// Good keys — hierarchical, easy to invalidate at any level
['user', 'profile']                    // invalidate: ['user'] → clears all user data
['products', { page: 1, category: 'shoes' }]  // invalidate: ['products'] → clears all product pages
['cart']
['notifications', 'count']
['search', { query: 'react hooks' }]   // invalidate: ['search'] → clears all searches

// Invalidation: specify prefix to invalidate all matching keys
queryClient.invalidateQueries({ queryKey: ['products'] });
// → Invalidates ['products', {page: 1}], ['products', {page: 2}], etc.
```

---

### 9.3 Request and Response Structure

**GET /api/products?page=1&category=shoes**

```json
// Response Headers
// Cache-Control: private, no-cache
// ETag: "abc123"
// Vary: Accept-Language, Authorization

// Response Body
{
  "data": [
    {
      "id": "prod_001",
      "name": "Running Shoes",
      "price": 129.99,
      "imageUrl": "https://cdn.example.com/products/prod_001.jpg",
      "inStock": true
    }
  ],
  "pagination": {
    "page": 1,
    "totalPages": 10,
    "nextCursor": "c_eyh78"
  },
  "meta": {
    "cachedAt": "2026-03-17T10:30:00Z",
    "version": "v2"
  }
}
```

**Conditional request (on repeat visit):**

```
GET /api/products?page=1&category=shoes
If-None-Match: "abc123"

→ 304 Not Modified (0 bytes body transferred)
→ Browser uses cached response
```

---

### 9.4 Error Handling and Status Codes

| Status | Scenario | Cache Behavior |
|--------|----------|----------------|
| `200` | Success | Cache the response |
| `304` | Not Modified (ETag match) | Use cached version |
| `400` | Bad request | Do not cache; show error |
| `401` | Unauthorized | Clear user caches; redirect to login |
| `404` | Not found | Remove from cache if previously cached |
| `429` | Rate limited | Serve stale from cache; retry after backoff |
| `500` | Server error | Serve stale from cache (if available); retry |
| `503` | Service unavailable | Serve stale from cache; show degraded mode banner |

---

## 10. Caching Strategy

*(This section is the meta-summary of the entire design — the overview of all layers together.)*

### 10.1 The Five Layers

| Layer | Speed | Persistence | Size Limit | Controlled By |
|-------|-------|------------|------------|---------------|
| **L1: In-Memory** (React Query) | < 1ms | Session | ~100MB (heap) | `staleTime`, `gcTime` |
| **L2: Service Worker** (Cache API) | < 10ms | Persistent | ~60% disk | Workbox strategies + plugins |
| **L3: Browser Storage** (IndexedDB/LS) | < 50ms | Persistent | GB (IDB), 5-10MB (LS) | Application code |
| **L4: HTTP Cache** (Browser) | < 5ms | Session/persistent | Browser-managed | `Cache-Control`, `ETag` |
| **L5: CDN Edge** | 5-50ms | Persistent | CDN-managed | `s-maxage`, purge API |

### 10.2 Strategy Summary by Data Type

| Data | L1 (Memory) | L2 (SW) | L3 (Storage) | L4 (HTTP) | L5 (CDN) |
|------|------------|---------|-------------|----------|---------|
| **JS/CSS bundles** | — | CacheFirst | — | immutable 1yr | ✅ |
| **Images** | — | CacheFirst | — | max-age 30d | ✅ |
| **API: products** | SWR 2min | StaleWR 5min | IndexedDB | no-cache+ETag | — |
| **API: user profile** | SWR 10min | NetworkFirst 10min | — | private, no-cache | — |
| **API: stock** | 0 (no cache) | NetworkOnly | — | no-store | — |
| **Preferences** | — | — | localStorage | — | — |
| **Offline data** | — | — | IndexedDB | — | — |

### 10.3 Invalidation Summary

| Trigger | Mechanism | Scope |
|---------|-----------|-------|
| User mutation | `queryClient.invalidateQueries` | Affected keys + related keys |
| TTL expiry | React Query `staleTime` / Workbox `maxAgeSeconds` | Individual entry |
| Server push | WebSocket message → `invalidateQueries` | Broadcast to all connected tabs |
| Cross-tab | `BroadcastChannel` → `invalidateQueries` | All tabs for same origin |
| App deploy | New HTML → new JS bundle hashes | All static assets |
| Manual refresh | User pulls to refresh or clicks retry | Current page's queries |

---

## 11. CDN and Asset Optimization

*   **Static assets** (JS, CSS, images, fonts) served from CDN edge PoPs (Points of Presence) for ~5-50ms latency globally.
*   **Content hashing**: All bundled assets use content hashes in filenames → `Cache-Control: public, max-age=31536000, immutable` → never refetched until new deploy.
*   **Image optimization**: Serve WebP/AVIF with JPEG fallback. Responsive `srcset` for device-appropriate sizes. Lazy load below-fold images.
*   **Font optimization**: `font-display: swap` to prevent FOIT (Flash of Invisible Text). Preload critical fonts via `<link rel="preload">`. Subset fonts to include only used characters.
*   **Compression**: Brotli (preferred) or Gzip for text assets. ~60-80% size reduction for JS/CSS.
*   **Preconnect**: `<link rel="preconnect" href="https://cdn.example.com">` to establish early DNS + TCP + TLS connections.
*   **Cache headers split**: `s-maxage=86400` for CDN (24h), `max-age=3600` for browser (1h) — allows CDN purge without waiting for browser cache expiry.

---

## 12. Rendering Strategy

*   **App Shell (SSR or SSG)**: The outer layout (header, sidebar, footer) is server-rendered or statically generated — available immediately from CDN. Inner content hydrates on the client.
*   **API Data (CSR)**: All cacheable API data is fetched client-side through React Query. CSR means the cache layer is fully in the client's control.
*   **Static Pages (SSG/ISR)**: Marketing pages, docs, and blog posts generated at build time or incrementally. Served from CDN with long cache TTL.
*   **Streaming SSR**: For critical above-the-fold content, stream HTML from the server with initial data embedded — avoids the CSR fetch waterfall.
*   **Cache-aware hydration**: On SSR pages, the server's initial data is dehydrated into the HTML → React Query hydrates it on the client → no duplicate fetch.

```tsx
// Server-side: dehydrate React Query cache into HTML
const dehydratedState = dehydrate(queryClient);

// Client-side: hydrate from server data — no refetch
<HydrationBoundary state={dehydratedState}>
  <App />
</HydrationBoundary>
```

---

## 13. Cross Cutting Non Functional Concerns

### 13.1 Security

*   **Never cache auth tokens in localStorage**: Use `httpOnly` cookies for session management. XSS cannot read `httpOnly` cookies.
*   **`Cache-Control: no-store` for sensitive data**: Ensure API responses with PII, financial data, or auth tokens include `no-store` to prevent any caching.
*   **Clear caches on logout**: On user logout, clear in-memory cache (`queryClient.clear()`), localStorage (`localStorage.clear()` for app-prefixed keys), IndexedDB stores, and Service Worker caches.
*   **Signed CDN URLs**: For protected media, use short-lived signed URLs that expire. Cache the content (not the URL) with appropriate TTL.
*   **Subresource Integrity (SRI)**: Add `integrity` attributes to `<script>` and `<link>` tags for CDN-hosted assets to prevent tampering.
*   **CSP (Content Security Policy)**: Configure CSP headers to restrict which origins can serve cacheable content.

```tsx
// Logout cache cleanup
async function onLogout() {
  // Clear in-memory
  queryClient.clear();

  // Clear localStorage (only app-specific keys)
  Object.keys(localStorage)
    .filter((key) => key.startsWith('app:'))
    .forEach((key) => localStorage.removeItem(key));

  // Clear IndexedDB
  const db = await dbPromise;
  await db.clear('user-profile');
  await db.clear('conversations');

  // Clear Service Worker caches
  const cacheNames = await caches.keys();
  await Promise.all(
    cacheNames
      .filter((name) => name.startsWith('user-'))
      .map((name) => caches.delete(name))
  );
}
```

---

### 13.2 Accessibility

*   **Loading states**: When data is refetching (stale-while-revalidate), use `aria-busy="true"` on containers.
*   **Offline banner**: `role="alert"` with `aria-live="assertive"` for "You're offline" message.
*   **Stale data indicator**: Screen readers should announce when displayed data may be outdated: `aria-label="Product price shown may be outdated. Last updated 5 minutes ago."`.
*   **Prefetch links**: Prefetching on hover/focus must not interfere with keyboard navigation or screen reader flow.
*   **Cache-related errors**: Error toasts for cache failures should be announced via `aria-live="polite"`.

---

### 13.3 Performance Optimization

*   **Cache-aware code splitting**: Only load the Service Worker registration code on first visit. On repeat visits, the SW is already active.
*   **Deferred hydration**: For non-critical cached data, defer hydration until after LCP. Use `React.lazy` + `Suspense` with cache-primed data.
*   **Stale-while-revalidate eliminates loading states**: Users see data instantly (from cache); background revalidation updates silently. Fewer spinners = better perceived performance.
*   **Prefetching budget**: Limit prefetching to 3-5 concurrent requests to avoid saturating the network and competing with active user requests.
*   **Compression + caching**: Brotli-compressed responses are cached in their compressed form by the HTTP cache, saving both bandwidth and disk space.

**Performance budget:**

```
First visit (cold cache):
  HTML:     ~10KB (gzipped) — from CDN in 50ms
  JS:       ~200KB (gzipped) — from CDN in 100ms
  CSS:      ~30KB (gzipped) — from CDN in 30ms
  API data: ~5-50KB — from origin in 200ms
  Total: ~300ms to interactive

Repeat visit (warm cache):
  HTML:     304 Not Modified — 50ms round trip
  JS/CSS:   From HTTP cache — 0ms
  API data: From React Query cache — <1ms
  Total: ~50ms to interactive (6x faster!)
```

---

### 13.4 Observability and Reliability

*   **Cache hit/miss logging**: Track per-layer cache hit ratios (in-memory, Service Worker, HTTP). Alert if hit ratio drops below threshold (e.g., < 60%).
*   **Stale data monitoring**: Track how often users see stale data and for how long. Set SLOs (e.g., "< 5% of page views show data older than 10 minutes").
*   **Storage quota monitoring**: Use `navigator.storage.estimate()` to track storage usage and warn before hitting limits.
*   **Service Worker health**: Monitor SW registration, update, and activation events. Alert on SW errors or unexpected unregistrations.
*   **Error boundaries**: Wrap cache-dependent components in error boundaries. A corrupted cache entry should not crash the app.
*   **Feature flags**: Gate new caching strategies (e.g., IndexedDB offline mode, aggressive prefetching) behind flags for gradual rollout.

```tsx
// Monitor storage usage
async function logStorageQuota() {
  if ('storage' in navigator && 'estimate' in navigator.storage) {
    const { usage, quota } = await navigator.storage.estimate();
    analytics.track('storage_usage', {
      usageBytes: usage,
      quotaBytes: quota,
      percentUsed: ((usage / quota) * 100).toFixed(2),
    });
  }
}
```

---

## 14. Edge Cases and Tradeoffs

| Edge Case | Handling |
|-----------|----------|
| **User sees outdated price, adds to cart** | Cart API validates current price server-side. If changed, show "Price updated" message with new price. |
| **Browser clears cache unexpectedly** | Service Worker re-installs on next visit. React Query refetches on mount. App degrades gracefully. |
| **User has two tabs with different cached data** | BroadcastChannel syncs invalidation across tabs. Both tabs converge on mutation. |
| **Storage quota exceeded** | Workbox's ExpirationPlugin evicts oldest entries. IndexedDB cleanup runs in `requestIdleCallback`. App continues working with smaller cache. |
| **Stale Service Worker serves old JS bundle** | `skipWaiting` + `clients.claim` on new SW activation. Or prompt user: "New version available — refresh?" |
| **Third-party CDN is down** | Service Worker serves from Cache API. If not cached, show fallback UI. |
| **User on flaky connection (2G)** | NetworkFirst with 3s timeout → falls back to cache quickly. Show "Loading slowly" indicator. |
| **Race condition: revalidation response arrives after newer mutation** | React Query uses timestamps — ignores stale background fetch if data was updated more recently by a mutation. |
| **Cache corruption (malformed JSON in IDB)** | Wrap all cache reads in try-catch. On error, delete corrupted entry and refetch from network. |
| **Offline user modifies data** | Queue mutations in IndexedDB. Sync on reconnect with conflict resolution (last-write-wins or merge). |

### Key Tradeoffs

| Decision | Tradeoff |
|----------|----------|
| **Stale-while-revalidate as default** | Instant perceived load, but user may briefly see outdated data. Acceptable for most data (catalogs, feeds), dangerous for financial data or inventory. |
| **Service Worker adds complexity** | Offline support and persistent caching, but adds a new lifecycle to manage (install, activate, update). Stale SW can serve old code. |
| **IndexedDB for offline** | Large structured storage, but async API adds complexity. Schema migrations needed on app updates. |
| **localStorage for boot data** | Instant synchronous access before React renders, but 5-10MB limit and synchronous API blocks main thread for large reads. Keep data small (< 50KB). |
| **Aggressive prefetching** | Instant navigation, but wastes bandwidth if user doesn't visit prefetched pages. Cap at 3-5 concurrent prefetches. |
| **Content hashing + immutable** | Perfect cache invalidation for static assets, but requires a build pipeline. No cache busting for assets without hashes. |
| **BroadcastChannel cross-tab sync** | Consistent cache across tabs, but adds code complexity. Falls back to independent caches if not supported. |
| **React Query as cache layer** | Excellent DX, built-in stale-while-revalidate, auto-GC, devtools. But it's another dependency (~13KB). SWR (~4KB) is lighter but less feature-rich. |

---

## 15. Summary and Future Improvements

### Key Architectural Decisions

1.  **Five-layer cache architecture** (Memory → Service Worker → Browser Storage → HTTP Cache → CDN) — each layer serves a specific speed and persistence profile.
2.  **Stale-while-revalidate as default strategy** — instant data access with background freshness; overridden with NetworkFirst or NetworkOnly for sensitive/critical data.
3.  **Content-hashed static assets with immutable caching** — deploy-triggered invalidation via new filenames, never via cache busting or manual purge.
4.  **React Query as the in-memory cache manager** — provides unified API for fetching, caching, invalidation, optimistic updates, and background revalidation.
5.  **Service Worker with Workbox strategies** — persistent offline cache, configurable per-route strategies, automatic eviction.
6.  **Three invalidation patterns (TTL + Mutation + Server Push)** — covers all scenarios from time-based expiry to real-time cross-device sync.

### Possible Future Enhancements

*   **Shared Worker for multi-tab cache**: Use a `SharedWorker` to share a single in-memory cache across all tabs, reducing memory usage and ensuring consistency.
*   **Background Sync API**: Queue failed mutations in the Service Worker and automatically retry when connectivity is restored (beyond simple reconnect).
*   **Streaming cache responses**: Use `ReadableStream` in Service Worker to stream partial cached responses while fetching the rest from the network.
*   **Delta/patch caching**: Instead of refetching entire API responses, fetch only the diff (JSON Patch) since the last cached version — dramatically reduces bandwidth.
*   **Predictive prefetching with ML**: Use navigation pattern data to predict which pages the user will visit next and prefetch them with higher confidence.
*   **Edge compute caching**: Move cache logic to Cloudflare Workers or Vercel Edge Functions — personalized responses cached at the edge.
*   **Cache analytics dashboard**: Real-time dashboard showing per-layer hit ratios, stale data rates, storage usage, and user impact metrics.
*   **Automatic cache policy tuning**: Auto-adjust TTLs based on observed data change frequency — data that changes every 30min doesn't need a 24h cache.

---

### Cache Layer Summary

| Layer | Technology | Speed | Persistence | Best For |
|-------|-----------|-------|-------------|----------|
| L1 | React Query / SWR | < 1ms | Session | Active page data |
| L2 | Service Worker (Cache API) | < 10ms | Persistent | Offline access, static assets |
| L3 | IndexedDB / localStorage | < 50ms | Persistent | Structured offline data, preferences |
| L4 | Browser HTTP Cache | < 5ms | Varies | All HTTP responses |
| L5 | CDN Edge | 5-50ms | Persistent | Static assets, public API responses |

---

### Complete Cache Flow

| Trigger | Layer Check Order | Miss Behavior | Cache Write Order |
|---------|------------------|---------------|-------------------|
| Initial page load | L4 (HTML) → L5 (CDN) → Origin | Fetch HTML → load hashed JS/CSS from L4/L5 | L4 + L5 auto-cache |
| useQuery fires | L1 (memory) → L2 (SW) → L4 → L5 → Origin | Fetch from origin, populate all layers | L1 → L2 → L4 (automatic) |
| Repeat navigation | L1 (memory) → return instantly | — (L1 HIT most likely) | Background revalidation updates L1 |
| Network offline | L1 → L2 → L3 (IndexedDB) | Show offline banner if all miss | — |
| User mutation | Skip cache → POST to origin | — | Invalidate L1 keys, refetch |
| App deploy | New HTML with new JS hashes | L4 miss for new bundles → fetch from L5/origin | L4 + L5 cache new assets |
| Tab refocus | L1 check staleTime | If stale, background revalidate | Update L1 with fresh response |

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design
