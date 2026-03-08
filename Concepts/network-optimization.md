# Network Optimization – Frontend Performance

Network optimization focuses on reducing the number, size, and latency of HTTP requests. Even with perfectly optimized assets, poor network strategy can bottleneck performance.

## Table of Contents

- [1. HTTP/2 and HTTP/3](#1-http2-and-http3)
- [2. Resource Hints (preload, prefetch, preconnect, dns-prefetch)](#2-resource-hints-preload-prefetch-preconnect-dns-prefetch)
- [3. Caching Strategies](#3-caching-strategies)
- [4. Compression (Gzip, Brotli)](#4-compression-gzip-brotli)
- [5. CDN (Content Delivery Network)](#5-cdn-content-delivery-network)
- [6. Bundle Splitting and Code Splitting](#6-bundle-splitting-and-code-splitting)
- [7. Service Workers and Offline Caching](#7-service-workers-and-offline-caching)
- [Key Takeaways](#key-takeaways)

---

## 1. HTTP/2 and HTTP/3

**Concept:**
HTTP/1.1 sends requests one at a time per connection (head-of-line blocking). HTTP/2 and HTTP/3 remove this bottleneck with multiplexing, allowing many requests over a single connection.

**Evolution:**

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Multiplexing | No (1 req/connection) | Yes (many streams) | Yes (many streams) |
| Header compression | No | HPACK | QPACK |
| Server Push | No | Yes | Yes |
| Transport | TCP | TCP | QUIC (UDP-based) |
| Head-of-line blocking | Per connection | Per TCP connection | None (per-stream) |
| Connection setup | TCP + TLS (2-3 RTT) | TCP + TLS (2-3 RTT) | 0-1 RTT |

**Impact on optimization strategy:**

In HTTP/1.1 era:
- Concatenate files (fewer requests better)
- Use CSS sprites
- Domain sharding (multiple domains for parallel downloads)

In HTTP/2+ era:
- Smaller, granular files are fine (multiplexing handles many requests)
- CSS sprites are less necessary
- Domain sharding actually hurts (breaks multiplexing)
- Focus on reducing total bytes, not total requests

**Verification:**

```javascript
// Check HTTP protocol version in browser DevTools
// Network tab → Right-click header → Enable "Protocol" column
// Look for h2 (HTTP/2) or h3 (HTTP/3)

// Performance API check
const entries = performance.getEntriesByType('resource');
entries.forEach(entry => {
  console.log(entry.name, entry.nextHopProtocol);
  // Output: "script.js" "h2" or "h3"
});
```

---

## 2. Resource Hints (preload, prefetch, preconnect, dns-prefetch)

**Concept:**
Resource hints tell the browser about resources it will need soon, allowing it to start fetching or connecting earlier than it normally would.

**Four resource hints:**

| Hint | Purpose | Priority | When to Use |
|------|---------|----------|-------------|
| `dns-prefetch` | Resolve DNS for a domain | Low | Third-party domains |
| `preconnect` | DNS + TCP + TLS handshake | Medium | Critical third-party origins |
| `preload` | Download specific resource NOW | High | Critical current-page resources |
| `prefetch` | Download resource for NEXT page | Low | Next-page navigation resources |

**Example – All four in practice:**

```html
<head>
  <!-- DNS Prefetch: resolve domain name early -->
  <link rel="dns-prefetch" href="https://analytics.example.com">

  <!-- Preconnect: full connection setup to critical third-party -->
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://cdn.example.com" crossorigin>

  <!-- Preload: fetch critical resources for THIS page immediately -->
  <link rel="preload" href="/fonts/main.woff2" as="font" type="font/woff2" crossorigin>
  <link rel="preload" href="/css/critical.css" as="style">
  <link rel="preload" href="/images/hero.webp" as="image">

  <!-- Prefetch: fetch resources for NEXT likely page -->
  <link rel="prefetch" href="/next-page/bundle.js" as="script">
  <link rel="prefetch" href="/next-page/data.json" as="fetch">
</head>
```

**Timeline comparison:**

```
Without hints:
  HTML → Parse → Discover CSS → Download CSS → Parse CSS → Discover font → Download font
                                                                            ^^^^ Very late

With preconnect + preload:
  HTML → Preconnect to CDN + Preload font (parallel with everything)
       → Parse → Discover CSS → Download CSS (connection already open!)
       → Font already downloading or downloaded
  Savings: 200-500ms per resource
```

**Best practices:**
- Limit preconnect to 2-4 critical origins (each one costs CPU)
- Preload only resources needed within first 3 seconds
- Prefetch only highly likely next-page resources
- dns-prefetch is cheap, use for all known third-party domains
- Over-using preload can hurt – it competes with other critical resources

---

## 3. Caching Strategies

**Concept:**
Caching stores responses locally so repeat visits or requests do not require network round-trips. An effective caching strategy can make second visits near-instant.

**Two levels of caching:**

**1. Browser Cache (HTTP Cache Headers):**

```
Cache-Control: public, max-age=31536000, immutable
```

| Directive | Meaning |
|-----------|---------|
| `public` | Any cache (browser, CDN) can store |
| `private` | Only browser can store (user-specific data) |
| `max-age=N` | Cache is valid for N seconds |
| `immutable` | File will never change (skip revalidation) |
| `no-cache` | Always revalidate with server before using |
| `no-store` | Never cache (sensitive data) |
| `stale-while-revalidate=N` | Use stale cache while revalidating in background |

**Recommended cache strategy by file type:**

```
HTML pages:
  Cache-Control: no-cache
  (Always check for latest version, server may return 304 Not Modified)

CSS/JS with hash in filename (main.a1b2c3.js):
  Cache-Control: public, max-age=31536000, immutable
  (Cache forever – filename changes when content changes)

Images:
  Cache-Control: public, max-age=86400
  (Cache for 1 day – or use immutable with hashed filenames)

API responses:
  Cache-Control: private, max-age=0, must-revalidate
  (User-specific, always fresh)

Fonts:
  Cache-Control: public, max-age=31536000, immutable
  (Fonts rarely change)
```

**2. ETag / Last-Modified (Conditional Requests):**

```
First request:
  Server → ETag: "abc123"
  Browser caches response

Second request:
  Browser → If-None-Match: "abc123"
  Server checks: file unchanged → 304 Not Modified (no body, saves bandwidth)
  Server checks: file changed → 200 OK with new content
```

---

## 4. Compression (Gzip, Brotli)

**Concept:**
Text-based assets (HTML, CSS, JS, JSON, SVG) are compressed on the server before transmission. The browser decompresses them automatically. This reduces transfer size by 60-90%.

**Compression comparison:**

| Algorithm | Compression Ratio | Speed | Browser Support |
|-----------|------------------|-------|-----------------|
| None | 1x (baseline) | N/A | Universal |
| Gzip | 5-8x smaller | Fast | Universal |
| Brotli | 6-10x smaller | Slower to compress | 96%+ browsers |

**Typical savings:**

| Asset | Original | Gzip | Brotli |
|-------|----------|------|--------|
| React bundle (200KB) | 200KB | 55KB | 45KB |
| CSS file (100KB) | 100KB | 18KB | 15KB |
| JSON API (50KB) | 50KB | 8KB | 6KB |

**Server configuration examples:**

```nginx
# Nginx – Enable both Brotli and Gzip
# Brotli (preferred)
brotli on;
brotli_types text/html text/css application/javascript application/json image/svg+xml;
brotli_comp_level 6;

# Gzip (fallback)
gzip on;
gzip_types text/html text/css application/javascript application/json image/svg+xml;
gzip_min_size 256;
```

**How browser negotiation works:**

```
Browser sends:
  Accept-Encoding: br, gzip, deflate

Server responds with best match:
  Content-Encoding: br       (if Brotli supported)
  Content-Encoding: gzip     (fallback if not)
```

**Build-time pre-compression (Webpack):**

```javascript
const CompressionPlugin = require('compression-webpack-plugin');

module.exports = {
  plugins: [
    // Gzip
    new CompressionPlugin({
      algorithm: 'gzip',
      filename: '[path][base].gz',
    }),
    // Brotli
    new CompressionPlugin({
      algorithm: 'brotliCompress',
      filename: '[path][base].br',
      compressionOptions: { level: 11 },
    }),
  ],
};
```

**Benefits of pre-compression:**
- Server serves pre-compressed files instantly (no CPU cost per request)
- Can use maximum Brotli level 11 (too slow for real-time, but fine for build)
- Gzip as fallback for older clients

---

## 5. CDN (Content Delivery Network)

**Concept:**
A CDN distributes copies of your assets to servers spread across the globe (edge locations). Users fetch assets from the nearest edge server instead of your origin server, reducing latency dramatically.

**How CDN reduces latency:**

```
Without CDN:
  User in Tokyo → Request to origin in US → 200ms RTT
  Total for CSS + JS + images = 200ms x 10 resources = 2000ms

With CDN:
  User in Tokyo → Request to CDN edge in Tokyo → 10ms RTT
  Total for CSS + JS + images = 10ms x 10 resources = 100ms
  (Plus HTTP/2 multiplexing reduces this further)
```

**What to serve from CDN:**
- Static assets: CSS, JS, images, fonts, videos
- Build artifacts with content hashes
- Third-party libraries

**What NOT to serve from CDN:**
- HTML pages that change frequently (or use short cache + `stale-while-revalidate`)
- User-specific or authenticated content
- Real-time API endpoints

**CDN configuration example (Cloudflare):**

```
Page Rules:
  *.example.com/static/*
    Cache Level: Cache Everything
    Edge Cache TTL: 1 month
    Browser Cache TTL: 1 year

  example.com/*.html
    Cache Level: Standard
    Edge Cache TTL: 10 minutes
```

**Multi-CDN strategy:**
- Use primary CDN for your assets
- Use specialized Image CDN for image optimization
- Use separate CDN for video streaming
- Implement failover between CDNs

---

## 6. Bundle Splitting and Code Splitting

**Concept:**
Instead of sending one massive JavaScript bundle, split it into smaller chunks that load on demand. Users only download the code they actually need for the current page or interaction.

**Types of splitting:**

| Strategy | What It Does | Example |
|----------|-------------|---------|
| Vendor splitting | Separate third-party libraries | React, Lodash → vendor.js |
| Route splitting | Separate code per page/route | /home → home.js, /about → about.js |
| Component splitting | Lazy load heavy components | Modal, Chart → separate chunks |

**Webpack vendor splitting:**

```javascript
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendor',
          chunks: 'all',
          priority: 10,
        },
        common: {
          minChunks: 2,
          name: 'common',
          chunks: 'all',
          priority: 5,
        },
      },
    },
  },
};
```

**React route-based code splitting:**

```jsx
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

// Each route loads its own chunk
const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<PageLoader />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/settings" element={<Settings />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

**Result:**
- `/` loads `home.[hash].js` (~30KB)
- `/dashboard` loads `dashboard.[hash].js` (~80KB)
- `/settings` loads `settings.[hash].js` (~20KB)
- User visiting only `/` never downloads dashboard or settings code

**Component-level splitting:**

```jsx
import { lazy, Suspense, useState } from 'react';

// Heavy chart library (~200KB) loaded only when user opens analytics
const AnalyticsChart = lazy(() => import('./AnalyticsChart'));

function Dashboard() {
  const [showAnalytics, setShowAnalytics] = useState(false);

  return (
    <div>
      <h1>Dashboard</h1>
      <button onClick={() => setShowAnalytics(true)}>Show Analytics</button>

      {showAnalytics && (
        <Suspense fallback={<ChartSkeleton />}>
          <AnalyticsChart />
        </Suspense>
      )}
    </div>
  );
}
```

---

## 7. Service Workers and Offline Caching

**Concept:**
A Service Worker is a JavaScript file that runs in the background, separate from the web page. It intercepts network requests and can serve cached responses, enabling offline functionality and instant repeat loads.

**Service Worker lifecycle:**

```
1. Register → Browser downloads and installs the SW
2. Install → SW caches critical assets (precaching)
3. Activate → SW takes control of pages
4. Fetch → SW intercepts every network request and decides: cache or network
```

**Basic Service Worker registration:**

```javascript
// main.js – Register service worker
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js')
    .then(reg => console.log('SW registered:', reg.scope))
    .catch(err => console.log('SW registration failed:', err));
}
```

**Service Worker with caching strategies:**

```javascript
// sw.js
const CACHE_NAME = 'app-v1';
const PRECACHE_ASSETS = [
  '/',
  '/css/main.css',
  '/js/app.js',
  '/images/logo.svg',
  '/offline.html',
];

// Install: precache critical assets
self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(PRECACHE_ASSETS))
  );
});

// Activate: clean old caches
self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(keys.filter(k => k !== CACHE_NAME).map(k => caches.delete(k)))
    )
  );
});

// Fetch: serve from cache, fallback to network
self.addEventListener('fetch', event => {
  event.respondWith(
    caches.match(event.request).then(cached => {
      if (cached) return cached;               // Cache hit
      return fetch(event.request).then(response => {
        // Cache new resources for next time
        const clone = response.clone();
        caches.open(CACHE_NAME).then(cache => cache.put(event.request, clone));
        return response;
      });
    }).catch(() => caches.match('/offline.html'))  // Offline fallback
  );
});
```

**Common caching strategies:**

| Strategy | Behavior | Best For |
|----------|----------|----------|
| Cache First | Check cache → fallback to network | Static assets, fonts, images |
| Network First | Check network → fallback to cache | API data, HTML pages |
| Stale While Revalidate | Serve cache immediately → update cache from network | Frequently updated content |
| Cache Only | Only cache, never network | Precached app shell |
| Network Only | Only network, never cache | Real-time data, auth |

**Workbox (Google's SW library) example:**

```javascript
// sw.js using Workbox
import { precacheAndRoute } from 'workbox-precaching';
import { registerRoute } from 'workbox-routing';
import { CacheFirst, StaleWhileRevalidate, NetworkFirst } from 'workbox-strategies';
import { ExpirationPlugin } from 'workbox-expiration';

// Precache build assets
precacheAndRoute(self.__WB_MANIFEST);

// Cache images: Cache First, expire after 30 days
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'images',
    plugins: [new ExpirationPlugin({ maxEntries: 100, maxAgeSeconds: 30 * 24 * 60 * 60 })],
  })
);

// Cache API: Network First
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({ cacheName: 'api-cache' })
);

// Cache CSS/JS: Stale While Revalidate
registerRoute(
  ({ request }) => request.destination === 'style' || request.destination === 'script',
  new StaleWhileRevalidate({ cacheName: 'static-resources' })
);
```

---

## Key Takeaways

- Leverage HTTP/2+ multiplexing (many small files are fine, avoid bundling everything)
- Use resource hints strategically: `preconnect` for critical origins, `preload` for critical assets, `prefetch` for next-page resources
- Cache aggressively with content-hashed filenames (`max-age=31536000, immutable`)
- Enable Brotli compression (Gzip as fallback) – 60-90% smaller text assets
- Serve static assets from CDN edge locations for minimal latency
- Split bundles by vendor, route, and component – users download only what they need
- Use Service Workers for offline support, instant repeat loads, and background sync

### Performance Metrics Impact

| Optimization | LCP | FCP | CLS | TTI | TTFB |
|-------------|-----|-----|-----|-----|------|
| HTTP/2-3 | + | + | | + | ++ |
| Resource hints | ++ | ++ | | + | + |
| Caching | +++ | +++ | | +++ | +++ |
| Compression | + | + | | + | ++ |
| CDN | ++ | ++ | | + | +++ |
| Code splitting | | | | +++ | |
| Service Workers | +++ | +++ | | +++ | +++ |

`+++` = Major impact, `++` = Moderate impact, `+` = Minor impact
