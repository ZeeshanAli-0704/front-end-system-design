# Rendering Patterns — CSR, SSR, CSR+SSR (Universal), Static Rendering & Prerendering

> A deep-dive into **how**, **when**, and **why** each rendering strategy exists, the request/response flow for every pattern, and concise code samples.

---

## Table of Contents

1. [Client-Side Rendering (CSR)](#1-client-side-rendering-csr)
2. [Server-Side Rendering (SSR)](#2-server-side-rendering-ssr)
3. [Universal / Isomorphic Rendering (CSR + SSR)](#3-universal--isomorphic-rendering-csr--ssr)
4. [Static Site Generation (SSG / Static Rendering)](#4-static-site-generation-ssg--static-rendering)
5. [Prerendering](#5-prerendering)
6. [Comparison Matrix](#6-comparison-matrix)
7. [Decision Flowchart — When to Pick What](#7-decision-flowchart--when-to-pick-what)

---

## 1. Client-Side Rendering (CSR)

### What Is It?

The server sends a **minimal, empty HTML shell** plus a JavaScript bundle. The browser downloads, parses, and **executes JS** to build the DOM, fetch data, and paint pixels — all on the client.

### How It Works — Step by Step

```
Browser                            Server / CDN
  |                                     |
  |  1. GET /dashboard                  |
  |------------------------------------>|
  |                                     |
  |  2. Returns bare HTML shell         |
  |     (empty <div id="root">)         |
  |<------------------------------------|
  |                                     |
  |  3. Browser parses HTML,            |
  |     finds <script src="bundle.js">  |
  |                                     |
  |  4. GET /bundle.js (+ chunks)       |
  |------------------------------------>|
  |                                     |
  |  5. Returns JS bundle               |
  |<------------------------------------|
  |                                     |
  |  6. JS executes:                    |
  |     - Framework boots               |
  |     - Components mount              |
  |     - Fetches data via API          |
  |------------------------------------>| (API calls)
  |                                     |
  |  7. API returns JSON                |
  |<------------------------------------|
  |                                     |
  |  8. JS renders DOM, page is         |
  |     now visible & interactive       |
  |                                     |
```

### Key Characteristics

| Aspect | Detail |
|---|---|
| **First Paint** | Slow — user sees blank/spinner until JS loads |
| **Time to Interactive (TTI)** | Equal to first meaningful paint (once JS runs, page is already interactive) |
| **SEO** | Poor by default (empty HTML; search bots may not execute JS) |
| **Server Load** | Minimal — server only serves static files |
| **Caching** | Easy — entire app is a set of static assets on a CDN |
| **Best For** | Dashboards, admin panels, SPAs behind auth (SEO not needed) |

### When to Use CSR

- Content is **behind authentication** (no SEO requirement).
- Highly **interactive** apps (dashboards, editors, Figma-like tools).
- You want the **simplest deployment** (just static files on a CDN).

### Sample Code (React + Vite)

```html
<!-- index.html — the shell -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>My App</title>
</head>
<body>
  <div id="root"></div>          <!-- empty! -->
  <script type="module" src="/src/main.tsx"></script>
</body>
</html>
```

```tsx
// main.tsx — JS takes over
import { createRoot } from 'react-dom/client';
import App from './App';

createRoot(document.getElementById('root')!).render(<App />);
```

```tsx
// App.tsx — data fetched on the client
import { useEffect, useState } from 'react';

export default function App() {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    fetch('/api/posts')
      .then(res => res.json())
      .then(setPosts);
  }, []);

  return (
    <ul>
      {posts.map(p => <li key={p.id}>{p.title}</li>)}
    </ul>
  );
}
```

> **Notice:** Data fetching happens *after* the JS bundle loads — this is the CSR waterfall.

---

## 2. Server-Side Rendering (SSR)

### What Is It?

On **every request**, the server runs the application code, fetches data, and generates **fully populated HTML**. The browser receives a complete page and renders it immediately.

### How It Works — Step by Step

```
Browser                            Server (Node.js / Edge)
  |                                     |
  |  1. GET /products/42                |
  |------------------------------------>|
  |                                     |
  |  2. Server executes app code:       |
  |     - Matches route                 |
  |     - Fetches data from DB/API      |
  |     - Renders component tree → HTML |
  |                                     |
  |  3. Returns FULL HTML               |
  |     (with product data baked in)    |
  |<------------------------------------|
  |                                     |
  |  4. Browser paints immediately      |
  |     → User sees content (FCP ✓)     |
  |                                     |
  |  5. Browser downloads JS bundle     |
  |------------------------------------>|
  |                                     |
  |  6. Returns JS bundle               |
  |<------------------------------------|
  |                                     |
  |  7. JS hydrates the DOM:            |
  |     - Attaches event listeners      |
  |     - Makes page interactive        |
  |     → TTI ✓                         |
  |                                     |
```

### Key Characteristics

| Aspect | Detail |
|---|---|
| **First Contentful Paint (FCP)** | Fast — HTML arrives fully formed |
| **Time to Interactive (TTI)** | Delayed — page is *visible* but *not interactive* until hydration completes (the "uncanny valley") |
| **SEO** | Excellent — crawlers receive complete HTML |
| **Server Load** | High — every request triggers rendering on the server |
| **Caching** | Harder — responses are dynamic; need CDN strategies (stale-while-revalidate, edge caching) |
| **Best For** | E-commerce product pages, news articles, marketing pages needing SEO + fresh data |

### When to Use SSR

- Pages need strong **SEO** and contain **frequently changing data**.
- **First paint speed** is critical (e-commerce, landing pages).
- Content is **personalized per user** (cannot be statically generated).

### Sample Code (Next.js App Router)

```tsx
// app/products/[id]/page.tsx — runs on the server per request
export default async function ProductPage({ params }) {
  // This fetch runs on the SERVER for every request
  const product = await fetch(`https://api.example.com/products/${params.id}`, {
    cache: 'no-store',    // ← forces SSR (no caching)
  }).then(r => r.json());

  return (
    <main>
      <h1>{product.name}</h1>
      <p>${product.price}</p>
      <AddToCartButton id={product.id} />  {/* Client Component */}
    </main>
  );
}
```

```tsx
// components/AddToCartButton.tsx — hydrated on the client
'use client';
import { useState } from 'react';

export function AddToCartButton({ id }) {
  const [added, setAdded] = useState(false);

  return (
    <button onClick={() => { addToCart(id); setAdded(true); }}>
      {added ? 'Added ✓' : 'Add to Cart'}
    </button>
  );
}
```

> **Notice:** The page HTML is ready instantly. The `AddToCartButton` becomes clickable only after hydration.

---

## 3. Universal / Isomorphic Rendering (CSR + SSR)

### What Is It?

The **same JavaScript code** runs on both the server (first render) and the client (subsequent navigation). The first page load is SSR'd for speed + SEO; after that, the app takes over as a **SPA** with client-side routing — no full-page reloads.

This is the pattern used by **Next.js, Nuxt, Remix, Angular Universal, SvelteKit**, etc.

### How It Works — Step by Step

```
Browser                            Server (Node.js)                CDN / API
  |                                     |                              |
  |  1. GET /home (first visit)         |                              |
  |------------------------------------>|                              |
  |                                     |                              |
  |  2. Server-side renders HTML        |--- fetch data from API ----->|
  |     using shared component code     |<----- JSON response ---------|
  |                                     |                              |
  |  3. Full HTML + serialized state    |                              |
  |     (window.__INITIAL_STATE__)      |                              |
  |<------------------------------------|                              |
  |                                     |                              |
  |  4. Browser paints page (FCP ✓)     |                              |
  |                                     |                              |
  |  5. Download JS bundle              |                              |
  |------------------------------------>|                              |
  |<------------------------------------|                              |
  |                                     |                              |
  |  6. HYDRATION:                      |                              |
  |     - React attaches to existing DOM|                              |
  |     - Reads __INITIAL_STATE__       |                              |
  |     - Page becomes interactive      |                              |
  |                                     |                              |
  |  === From here on: CSR mode ===     |                              |
  |                                     |                              |
  |  7. User clicks link → /about       |                              |
  |     (client-side route transition — |                              |
  |      NO server round-trip for HTML) |                              |
  |                                     |                              |
  |  8. Client fetches /api/about-data  |                              |
  |------------------------------------>|----------fetch data---------->|
  |<------------------------------------|<---------JSON response--------|
  |                                     |                              |
  |  9. Client renders new view         |                              |
  |     in the browser (SPA behavior)   |                              |
```

### The Hydration Concept

| Phase | What Happens |
|---|---|
| **Server Render** | Components execute on Node.js → produce HTML string |
| **Serialize State** | Data fetched on server is embedded as JSON in the HTML (`<script>window.__DATA__ = {...}</script>`) |
| **Client Boot** | React/Vue/Angular boots in the browser, finds existing DOM |
| **Hydration** | Framework **attaches event listeners** to the server-rendered DOM instead of re-creating it. It reconciles the server HTML with the client component tree |
| **SPA Takeover** | All subsequent navigation is handled by the client-side router — only API calls hit the server |

### Key Characteristics

| Aspect | Detail |
|---|---|
| **FCP** | Fast (SSR for first load) |
| **TTI** | Slight delay (hydration cost), but better perceived performance |
| **Subsequent Navigations** | Instant (client-side routing, no full reload) |
| **SEO** | Excellent (first load is full HTML) |
| **Complexity** | Highest — code must work in both Node.js and Browser environments |
| **Best For** | Large-scale apps needing both SEO and rich interactivity (e-commerce, social media, content platforms) |

### When to Use Universal Rendering

- You need **SEO + interactivity**.
- App has many **page transitions** that should feel instant.
- You're building with a framework that supports it out of the box (Next.js, Nuxt, Remix).

### Sample Code (Next.js — showing the SSR → SPA handoff)

```tsx
// app/layout.tsx — shared layout (server component)
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <nav>
          {/* These Links do CLIENT-SIDE navigation after first load */}
          <Link href="/">Home</Link>
          <Link href="/about">About</Link>
        </nav>
        {children}
      </body>
    </html>
  );
}
```

```tsx
// app/page.tsx — SSR'd on first load, CSR'd on subsequent navigations
export default async function HomePage() {
  const posts = await fetch('https://api.example.com/posts', {
    next: { revalidate: 60 },  // ISR: re-generate every 60s
  }).then(r => r.json());

  return (
    <section>
      <h1>Latest Posts</h1>
      {posts.map(p => (
        <article key={p.id}>
          <Link href={`/post/${p.id}`}>{p.title}</Link>
        </article>
      ))}
    </section>
  );
}
```

> **Key insight:** The first request is fully server-rendered. Clicking `<Link href="/about">` triggers a **client-side fetch** for the route data — the browser never reloads.

---

## 4. Static Site Generation (SSG / Static Rendering)

### What Is It?

Pages are rendered to HTML **at build time**. The output is a set of plain `.html` files deployed to a CDN. No server-side computation happens at request time.

### How It Works — Step by Step

```
                        BUILD TIME
Developer Machine / CI
  |
  |  1. Build command runs (next build, gatsby build, etc.)
  |
  |  2. For each route:
  |     - Fetch data from CMS / API / filesystem
  |     - Render component → HTML file
  |     - Generate associated .js chunks for hydration
  |
  |  3. Output:
  |     /out/index.html
  |     /out/about.html
  |     /out/blog/my-post.html
  |     /out/_next/static/chunks/...
  |
  |  4. Deploy all files to CDN
  |


                        REQUEST TIME
Browser                            CDN (Cloudflare, Vercel Edge, S3+CF)
  |                                     |
  |  1. GET /blog/my-post               |
  |------------------------------------>|
  |                                     |
  |  2. CDN serves pre-built HTML       |
  |     (no computation, just file I/O) |
  |<------------------------------------|
  |                                     |
  |  3. Browser paints immediately      |
  |     → Fastest possible FCP          |
  |                                     |
  |  4. JS loads, hydration runs        |
  |     → Page becomes interactive      |
  |                                     |
```

### Key Characteristics

| Aspect | Detail |
|---|---|
| **FCP** | Fastest possible — pre-built HTML served from edge |
| **TTI** | Fast — minimal JS needed for mostly static content |
| **SEO** | Excellent — full HTML available to crawlers |
| **Server Load** | Zero at runtime — everything is on the CDN |
| **Freshness** | Stale — content only updates on rebuild (unless using ISR) |
| **Build Time** | Can be very long for sites with thousands of pages |
| **Best For** | Blogs, docs, marketing sites, landing pages — content that changes infrequently |

### Incremental Static Regeneration (ISR) — The Hybrid

ISR (pioneered by Next.js) extends SSG: pages are statically generated but **re-generated in the background** after a configurable time interval, without a full rebuild.

```
Request 1 → serves cached HTML (built at deploy)
                                               ← stale-while-revalidate timer expires
Request 2 → serves stale cached HTML
             (triggers background re-generation)
Request 3 → serves FRESH re-generated HTML
```

### When to Use SSG

- Content **changes infrequently** (blogs, docs, portfolios).
- Pages are **not personalized** per user.
- You want **maximum performance** and **minimal infrastructure**.
- The number of pages is **manageable** at build time (< tens of thousands).

### Sample Code (Next.js SSG + ISR)

```tsx
// app/blog/[slug]/page.tsx

// Tell Next.js which slugs to pre-render at build time
export async function generateStaticParams() {
  const posts = await fetch('https://cms.example.com/posts').then(r => r.json());
  return posts.map(post => ({ slug: post.slug }));
}

// This runs at BUILD TIME (or during ISR re-generation)
export default async function BlogPost({ params }) {
  const post = await fetch(`https://cms.example.com/posts/${params.slug}`, {
    next: { revalidate: 3600 },  // ISR: regenerate every hour
  }).then(r => r.json());

  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  );
}
```

```tsx
// Pure SSG (no revalidation) — Astro example
// src/pages/about.astro
---
const team = await fetch('https://cms.example.com/team').then(r => r.json());
---
<html>
  <body>
    <h1>Our Team</h1>
    {team.map(member => (
      <div>
        <h2>{member.name}</h2>
        <p>{member.role}</p>
      </div>
    ))}
  </body>
</html>
<!-- This entire file becomes a static .html at build time. Zero JS shipped. -->
```

---

## 5. Prerendering

### What Is It?

Prerendering is a technique where a **headless browser** (e.g., Puppeteer, Rendertron) visits your CSR app at build time or on-demand, **executes the JavaScript**, and saves the resulting HTML. This HTML snapshot is served to crawlers (and optionally to users) while the actual app remains a CSR SPA.

> **Key difference from SSG:** SSG uses your framework's rendering pipeline at build time. Prerendering **runs a real browser** to capture the output of an already-built CSR app.

### How It Works — Step by Step

```
                  BUILD-TIME PRERENDERING
                  
Build System                    Headless Browser (Puppeteer)
  |                                     |
  |  1. Build CSR app normally          |
  |     (produces index.html + bundle)  |
  |                                     |
  |  2. For each route to prerender:    |
  |     Launch headless browser ------->|
  |                                     |
  |     3. Headless browser loads       |
  |        http://localhost:3000/about  |
  |        → JS executes               |
  |        → DOM is built              |
  |        → Data is fetched           |
  |                                     |
  |     4. Capture final HTML snapshot  |
  |<------------------------------------|
  |                                     |
  |  5. Save as /about/index.html       |
  |                                     |
  |  6. Deploy static files to CDN      |
  |                                     |


                  ON-DEMAND PRERENDERING (Dynamic Rendering)

Browser / Crawler                Middleware (Rendertron)           Origin Server
  |                                     |                              |
  |  1. GET /products/42                |                              |
  |------------------------------------>|                              |
  |                                     |                              |
  |  2. Middleware checks User-Agent    |                              |
  |     Is it a bot? (Googlebot, etc.)  |                              |
  |                                     |                              |
  |     IF BOT:                         |                              |
  |     3a. Renders page in headless    |                              |
  |         browser, returns HTML ------+--fetch CSR app-------------->|
  |         snapshot to crawler         |<----- CSR app files ---------|
  |<---------full HTML------------------|                              |
  |                                     |                              |
  |     IF REAL USER:                   |                              |
  |     3b. Proxy request as-is --------+--forward to origin---------->|
  |<---------normal CSR response--------|<----- CSR HTML shell --------|
  |                                     |                              |
```

### Key Characteristics

| Aspect | Detail |
|---|---|
| **FCP** | Same as CSR for users (unless build-time snapshots are served) |
| **SEO** | Good — bots receive fully rendered HTML |
| **Complexity** | Moderate — requires headless browser tooling |
| **Freshness** | Depends on re-render frequency |
| **Use Case** | Retrofit SEO onto an existing CSR/SPA without rewriting it |
| **Best For** | Legacy SPAs that need SEO, apps where migrating to SSR/SSG is too costly |

### When to Use Prerendering

- You have an **existing CSR SPA** and need to add SEO without a rewrite.
- The number of routes to prerender is **limited and known**.
- You want a **quick SEO fix** while planning a longer-term migration to SSR/SSG.

### Sample Code

#### Build-Time Prerendering (using `prerender-spa-plugin` with Webpack)

```js
// webpack.config.js
const PrerenderSPAPlugin = require('prerender-spa-plugin');
const path = require('path');

module.exports = {
  // ... normal webpack config
  plugins: [
    new PrerenderSPAPlugin({
      staticDir: path.join(__dirname, 'dist'),
      // Routes to prerender
      routes: ['/', '/about', '/contact', '/blog'],
      renderer: new PrerenderSPAPlugin.PuppeteerRenderer({
        renderAfterDocumentEvent: 'render-complete',
      }),
    }),
  ],
};
```

```js
// In your app — signal when rendering is done
document.addEventListener('DOMContentLoaded', () => {
  // ... app initialization
  mountApp().then(() => {
    document.dispatchEvent(new Event('render-complete'));
  });
});
```

#### On-Demand Prerendering (Express + Rendertron middleware)

```js
// server.js
const express = require('express');
const rendertron = require('rendertron-middleware');

const app = express();

// Serve prerendered HTML to bots, normal SPA to users
app.use(rendertron.makeMiddleware({
  proxyUrl: 'https://my-rendertron-instance.example.com/render',
  userAgentPattern: /Googlebot|bingbot|Slackbot|LinkedInBot/i,
}));

// Serve the CSR app
app.use(express.static('dist'));

app.listen(3000);
```

---

## 6. Comparison Matrix

| Pattern | Render Location | Render Timing | FCP | TTI | SEO | Server Cost | Freshness | Complexity |
|---|---|---|---|---|---|---|---|---|
| **CSR** | Browser | On every visit | 🔴 Slow | 🟢 Fast (= FCP) | 🔴 Poor | 🟢 None | 🟢 Always fresh | 🟢 Low |
| **SSR** | Server | On every request | 🟢 Fast | 🟡 Delayed (hydration) | 🟢 Excellent | 🔴 High | 🟢 Always fresh | 🟡 Medium |
| **CSR + SSR (Universal)** | Server → Browser | First load: server; then client | 🟢 Fast | 🟡 Delayed (hydration) | 🟢 Excellent | 🟡 Medium | 🟢 Always fresh | 🔴 High |
| **SSG** | Build machine | At build time | 🟢 Fastest | 🟢 Fast | 🟢 Excellent | 🟢 None | 🔴 Stale (until rebuild) | 🟢 Low |
| **SSG + ISR** | Build → Server (bg) | Build + on-demand bg regen | 🟢 Fastest | 🟢 Fast | 🟢 Excellent | 🟡 Low | 🟡 Near-fresh | 🟡 Medium |
| **Prerendering** | Headless browser | Build or on-demand | 🟡 Medium | 🟡 Medium | 🟢 Good | 🟡 Medium | 🟡 Depends | 🟡 Medium |

---

## 7. Decision Flowchart — When to Pick What

```
Start
  │
  ├─ Does the page need SEO?
  │    │
  │    ├─ NO → Is it highly interactive (dashboard, editor)?
  │    │         │
  │    │         ├─ YES → ✅ CSR
  │    │         └─ NO  → ✅ CSR (simplest deployment)
  │    │
  │    └─ YES → Does content change frequently?
  │               │
  │               ├─ NO (or rarely) → Can you rebuild on change?
  │               │                     │
  │               │                     ├─ YES → ✅ SSG (+ ISR for scale)
  │               │                     └─ NO  → ✅ ISR
  │               │
  │               └─ YES → Is content personalized per user?
  │                          │
  │                          ├─ YES → ✅ SSR (or Universal CSR+SSR)
  │                          └─ NO  → ✅ ISR or SSR with CDN caching
  │
  └─ Is it a legacy SPA you can't rewrite?
       │
       ├─ YES → ✅ Prerendering (Rendertron / Puppeteer)
       └─ NO  → Pick from above based on requirements
```

---

## 8. Real-World Pattern Usage

| Company / Product | Pattern | Why |
|---|---|---|
| **Gmail, Figma** | CSR | Behind auth, highly interactive, no SEO needed |
| **Amazon Product Pages** | SSR | SEO critical, personalized (price, recommendations), data changes constantly |
| **Next.js Docs, Gatsby blogs** | SSG | Content from markdown/CMS, changes infrequently, maximum performance |
| **Vercel, Shopify Storefronts** | Universal (CSR+SSR) + ISR | SEO + interactivity + near-fresh content without full SSR cost |
| **Legacy Angular SPAs** | Prerendering | Quick SEO fix without rewriting to SSR |

---

## 9. Common Pitfalls

### Hydration Mismatch (SSR / Universal)
Server HTML and client-rendered DOM must match exactly, or React will throw warnings and re-render from scratch.

```tsx
// ❌ BAD — different output on server vs client
function Greeting() {
  return <p>Hello at {new Date().toLocaleTimeString()}</p>;
  // Server: "Hello at 10:00:00 AM"
  // Client: "Hello at 10:00:01 AM" → MISMATCH
}

// ✅ FIX — defer client-only values
function Greeting() {
  const [time, setTime] = useState(null);
  useEffect(() => setTime(new Date().toLocaleTimeString()), []);
  return <p>Hello {time ? `at ${time}` : ''}</p>;
}
```

### Bundle Size Bloat (CSR)
Shipping too much JS delays FCP and TTI. Mitigate with:
- **Code splitting** — `React.lazy()` + `Suspense`
- **Tree-shaking** — remove unused exports
- **Dynamic imports** — `import('module')` only when needed

### Stale Content (SSG)
Users see outdated data until the next build. Mitigate with:
- **ISR** — background re-generation after a TTL
- **Client-side fetching** — hydrate with static data, then refresh on-demand
- **Webhooks** — CMS triggers rebuild on content change

### Server Cost Spikes (SSR)
Every request triggers rendering. Mitigate with:
- **CDN caching** with `stale-while-revalidate`
- **Edge rendering** (Cloudflare Workers, Vercel Edge Functions) — lower latency, distributed load
- **Streaming SSR** — send HTML in chunks as components resolve

---

## 10. The Modern Approach: Streaming SSR + Selective Hydration

Modern frameworks (React 18+, Next.js 13+) combine the best of all patterns:

```
Browser                            Server
  |                                     |
  |  GET /page                          |
  |------------------------------------>|
  |                                     |
  |  Server starts STREAMING HTML:      |
  |  ┌──────────────────────────┐       |
  |  │ <html><head>...</head>   │       |
  |  │ <body>                   │  ←──── sent immediately
  |  │   <header>Nav</header>   │
  |  │   <Suspense fallback>    │  ←──── placeholder for slow data
  |  │     Loading...           │
  |  │   </Suspense>            │
  |  └──────────────────────────┘       |
  |<------------------------------------|
  |                                     |
  |  Browser starts painting!           |
  |                                     |
  |  ... server fetches slow data ...   |
  |                                     |
  |  ┌──────────────────────────┐       |
  |  │ <template data-suspense> │       |
  |  │   <ProductList ... />    │  ←──── streamed in later
  |  │ </template>              │
  |  │ <script>replace()</script>│
  |  └──────────────────────────┘       |
  |<------------------------------------|
  |                                     |
  |  Browser swaps placeholder          |
  |  with real content inline           |
```

```tsx
// Streaming SSR with React 18 Suspense
import { Suspense } from 'react';

export default function Page() {
  return (
    <main>
      <h1>Product Page</h1>

      {/* This renders immediately */}
      <Header />

      {/* This streams in when the data resolves */}
      <Suspense fallback={<Skeleton />}>
        <ProductDetails />   {/* async server component */}
      </Suspense>

      {/* This also renders immediately */}
      <Footer />
    </main>
  );
}

async function ProductDetails() {
  const product = await fetchProduct(); // slow DB query
  return <div>{product.name} — ${product.price}</div>;
}
```

> **Selective Hydration:** React 18 can hydrate components **independently** — if the user clicks a not-yet-hydrated component, React **prioritizes hydrating that component first**.

---

## Summary

| If you need... | Use... |
|---|---|
| Simplest setup, no SEO | **CSR** |
| SEO + always-fresh data | **SSR** |
| SEO + interactivity + SPA feel | **Universal (CSR+SSR)** |
| Maximum speed, infrequent updates | **SSG** |
| SSG + freshish data without full rebuild | **ISR** |
| SEO for legacy SPA, no rewrite budget | **Prerendering** |
| Best of everything (modern) | **Streaming SSR + Selective Hydration** |



Get all articles related to system design 
Hastag: SystemDesignWithZeeshanAli


[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design