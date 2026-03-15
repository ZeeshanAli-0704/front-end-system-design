# Rendering Patterns — CSR, SSR, CSR+SSR (Universal), SSG, ISR, Prerendering & React Server Components

> A deep-dive into **how**, **when**, and **why** each rendering strategy exists, the request/response flow for every pattern, and concise code samples.

---

<a id="top"></a>

## Table of Contents

- [Client Side Rendering (CSR)](#client-side-rendering-csr)
- [Server Side Rendering (SSR)](#server-side-rendering-ssr)
- [Universal Isomorphic Rendering (CSR and SSR)](#universal-isomorphic-rendering-csr-and-ssr)
- [Static Site Generation (SSG Static Rendering)](#static-site-generation-ssg-static-rendering)
- [Incremental Static Regeneration (ISR)](#incremental-static-regeneration-isr)
- [Prerendering](#prerendering)
- [React Server Components (RSC)](#react-server-components-rsc)
- [Comparison Matrix](#comparison-matrix)
- [Decision Flowchart When to Pick What](#decision-flowchart-when-to-pick-what)
- [Real World Pattern Usage](#real-world-pattern-usage)
- [Common Pitfalls](#common-pitfalls)
- [The Modern Approach Streaming SSR and Selective Hydration](#the-modern-approach-streaming-ssr-and-selective-hydration)
- [Summary](#summary)
[⬆ Back to Top](#top)

---

## Client Side Rendering (CSR)

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

[⬆ Back to Top](#top)

---

## Server Side Rendering (SSR)

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

[⬆ Back to Top](#top)

---

## Universal Isomorphic Rendering (CSR and SSR)

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

[⬆ Back to Top](#top)

---

## Static Site Generation (SSG Static Rendering)

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

### When to Use SSG

- Content **changes infrequently** (blogs, docs, portfolios).
- Pages are **not personalized** per user.
- You want **maximum performance** and **minimal infrastructure**.
- The number of pages is **manageable** at build time (< tens of thousands).

### Sample Code (Next.js SSG)

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

[⬆ Back to Top](#top)

---

## Incremental Static Regeneration (ISR)

### What Is It?

ISR (pioneered by Next.js) extends SSG by allowing pages to be **re-generated in the background** after a configurable time interval — **without a full site rebuild**. It combines the performance of static pages with the freshness of server-rendered pages.

### How It Works — Step by Step

```
                        BUILD TIME (same as SSG)

  1. Build generates static HTML for all known routes
  2. Pages are deployed to CDN as static files


                        REQUEST TIME

Browser                            CDN / Edge                    Origin Server
  |                                     |                              |
  |  1. GET /products/42                |                              |
  |------------------------------------>|                              |
  |                                     |                              |
  |  2. CDN checks: is cached page      |                              |
  |     still within revalidate window? |                              |
  |                                     |                              |
  |     IF FRESH (within TTL):          |                              |
  |  3a. Serve cached HTML instantly    |                              |
  |<------------------------------------|                              |
  |                                     |                              |
  |     IF STALE (TTL expired):         |                              |
  |  3b. Serve stale HTML instantly     |                              |
  |<------------------------------------|                              |
  |                                     |                              |
  |     4. Trigger BACKGROUND regen --->|---regen request------------>|
  |        (user does NOT wait)         |                              |
  |                                     |   5. Server re-renders page  |
  |                                     |      with fresh data         |
  |                                     |<--- new HTML ----------------|  
  |                                     |                              |
  |     6. CDN replaces cached page     |                              |
  |        with fresh version           |                              |
  |                                     |                              |
  |  NEXT REQUEST:                      |                              |
  |  7. GET /products/42                |                              |
  |------------------------------------>|                              |
  |  8. Serves FRESH re-generated HTML  |                              |
  |<------------------------------------|                              |
```

**Key behavior:** ISR follows a **stale-while-revalidate** model. The user who triggers regeneration still gets the stale (but fast) page. The **next** user gets the fresh page. No user ever waits for regeneration.

### Key Characteristics

| Aspect | Detail |
|---|---|
| **FCP** | Fastest — same as SSG (pre-built HTML from CDN edge) |
| **TTI** | Fast — minimal JS for mostly static content |
| **SEO** | Excellent — full HTML available to crawlers |
| **Server Cost** | Low — only regenerates when stale + requested (not every request like SSR) |
| **Freshness** | Near-fresh — stale for at most `revalidate` seconds, then updated |
| **Build Time** | Same as SSG for initial build; no full rebuild needed for updates |
| **Scalability** | Excellent — CDN serves most traffic; origin only handles regenerations |
| **Best For** | E-commerce product pages, CMS-driven marketing sites, blogs with frequent updates |

### ISR vs SSG vs SSR — The Trade-off

```
SSG:  Build once → serve forever → stale until full rebuild
ISR:  Build once → serve → regenerate in background after TTL
SSR:  No build → render fresh on every single request

Freshness:    SSR > ISR > SSG
Performance:  SSG = ISR > SSR
Server Cost:  SSG < ISR << SSR
```

### On-Demand Revalidation

Beyond time-based ISR, modern frameworks support **on-demand revalidation** — triggering a page regeneration via an API call (e.g., from a CMS webhook) instead of waiting for the TTL to expire.

```
CMS content updated → Webhook fires → POST /api/revalidate?path=/blog/my-post
                                        → Server regenerates that specific page
                                        → Next request gets fresh content immediately
```

This gives you **SSG performance** with **near-instant freshness** — the best of both worlds.

### When to Use ISR

- Content updates **periodically but not on every request** (product pages, blog posts, landing pages).
- You need **SSG performance** but can't tolerate stale content for hours/days.
- You have **too many pages** to rebuild the entire site on every content change.
- You want to **avoid the server cost** of SSR while keeping content relatively fresh.

### When NOT to Use ISR

- Content is **personalized per user** (cart, profile, recommendations) → use SSR.
- Content must be **real-time fresh** (stock prices, live scores) → use SSR or CSR with API.
- No server/edge runtime available (pure static hosting like GitHub Pages) → use SSG.

### Sample Code (Next.js App Router)

```tsx
// app/products/[id]/page.tsx

// Generate static pages for known products at build time
export async function generateStaticParams() {
  const products = await fetch('https://api.example.com/products').then(r => r.json());
  return products.map(p => ({ id: String(p.id) }));
}

// This runs at BUILD TIME and again during ISR re-generation
export default async function ProductPage({ params }) {
  const product = await fetch(`https://api.example.com/products/${params.id}`, {
    next: { revalidate: 3600 },  // ISR: regenerate at most every 60 minutes
  }).then(r => r.json());

  return (
    <main>
      <h1>{product.name}</h1>
      <p>${product.price}</p>
      <p>{product.description}</p>
    </main>
  );
}
```

```ts
// app/api/revalidate/route.ts — On-demand revalidation via webhook
import { revalidatePath } from 'next/cache';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  const { path, secret } = await request.json();

  // Verify the webhook secret
  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ error: 'Invalid secret' }, { status: 401 });
  }

  // Regenerate the specific page
  revalidatePath(path); // e.g., '/products/42'
  return NextResponse.json({ revalidated: true, path });
}
```

[⬆ Back to Top](#top)

---

## Prerendering

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

### Sample Code (Prerendering)

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

[⬆ Back to Top](#top)

---

## React Server Components (RSC)

### What Is It?

React Server Components (RSC) are a **new rendering paradigm** introduced by the React team and shipped as the default in **Next.js App Router (13.4+)**. Unlike traditional rendering patterns where the entire component tree is either server-rendered or client-rendered, RSC allows you to **mix server and client components** in the same tree.

Server Components **run only on the server** — their code is never sent to the browser. They can directly access databases, file systems, and server-only APIs. They produce **zero client-side JavaScript** for their own rendering.

### How It Works — Step by Step

```
Browser                            Server
  |                                     |
  |  1. GET /products/42                |
  |------------------------------------>|
  |                                     |
  |  2. Server renders the component    |
  |     tree:                           |
  |                                     |
  |     <ProductPage>          ← Server Component (runs on server)
  |       <ProductInfo />      ← Server Component (DB query, zero JS)
  |       <Reviews />          ← Server Component (DB query, zero JS)
  |       <AddToCartButton />  ← Client Component (interactive, JS shipped)
  |                                     |
  |  3. Server sends:                   |
  |     - Full HTML for immediate paint |
  |     - RSC Payload (serialized tree) |
  |     - JS bundle ONLY for Client     |
  |       Components                    |
  |<------------------------------------|
  |                                     |
  |  4. Browser paints HTML instantly   |
  |                                     |
  |  5. JS hydrates ONLY the Client     |
  |     Components (AddToCartButton)    |
  |     Server Components need NO JS    |
  |                                     |
```

### Server Components vs Client Components

| Aspect | Server Component | Client Component |
|---|---|---|
| **Runs on** | Server only | Server (SSR) + Client (hydration) |
| **JS sent to browser** | None (zero KB) | Full component code |
| **Can access** | DB, filesystem, server secrets, APIs | Browser APIs, DOM, event handlers |
| **Can use** | `async/await` directly, Node.js APIs | `useState`, `useEffect`, `onClick`, etc. |
| **Interactivity** | None — pure render output | Full — state, effects, event handling |
| **Default in App Router** | ✅ Yes (all components are Server by default) | Must opt-in with `'use client'` directive |

### The Mental Model

Think of your component tree as a **map with two colors**:

```
┌─────────────────────────────────────────────────────┐
│  Layout (Server)                                     │
│  ┌────────────────────────────────────────────────┐  │
│  │  Header (Server) — no JS                       │  │
│  │  ┌──────────────────────────────────────────┐  │  │
│  │  │  SearchBar (Client) — 'use client'       │  │  │
│  │  │  (needs onChange, useState)               │  │  │
│  │  └──────────────────────────────────────────┘  │  │
│  ├────────────────────────────────────────────────┤  │
│  │  ProductGrid (Server) — DB query, no JS        │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐     │  │
│  │  │ Card     │  │ Card     │  │ Card     │     │  │
│  │  │ (Server) │  │ (Server) │  │ (Server) │     │  │
│  │  │          │  │          │  │          │     │  │
│  │  │ [Buy]    │  │ [Buy]    │  │ [Buy]    │     │  │
│  │  │ (Client) │  │ (Client) │  │ (Client) │     │  │
│  │  └──────────┘  └──────────┘  └──────────┘     │  │
│  ├────────────────────────────────────────────────┤  │
│  │  Footer (Server) — no JS                       │  │
│  └────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘

  █ Server Component = zero JS shipped
  █ Client Component = JS shipped + hydrated
```

**Key rule:** Server Components can import Client Components. Client Components **cannot** import Server Components (but can accept them as `children` props).

### Key Characteristics

| Aspect | Detail |
|---|---|
| **FCP** | 🟢 Fast — full HTML from server, no waiting for JS |
| **Bundle Size** | 🟢 Significantly smaller — only Client Components ship JS |
| **SEO** | 🟢 Excellent — full HTML response |
| **Data Fetching** | Direct server access — no API round-trips, no waterfalls |
| **Interactivity** | Client Components handle all user interactions |
| **DX** | `async/await` directly in components — no `useEffect` data fetching |
| **Complexity** | 🟡 Medium — must understand Server/Client boundary rules |
| **Best For** | Data-heavy pages with selective interactivity (e-commerce, dashboards, content sites) |

### Pros & Cons

| ✅ Pros | ❌ Cons |
|---|---|
| Zero JS for Server Components (massive bundle reduction) | Mental model shift — must learn Server/Client boundary |
| Direct DB/API access (no fetch waterfalls) | Cannot use hooks (`useState`, `useEffect`) in Server Components |
| Faster page loads (less JS to download, parse, execute) | Ecosystem still adapting — some libraries need `'use client'` wrappers |
| Eliminates many client-side data fetching patterns | Currently only fully supported in Next.js App Router |
| Streaming + Suspense integration out of the box | Server infrastructure required (not pure static) |
| Reduced hydration cost (only Client Components hydrate) | Debugging can be harder (server + client execution) |

### When to Use RSC

- Building with **Next.js App Router** (13.4+) — RSC is the default.
- Pages have **lots of data-fetching** and only **pockets of interactivity** (e.g., product pages with a buy button).
- You want to **drastically reduce JS bundle size**.
- You need **direct server access** (DB queries, secret keys) without building separate API routes.

### When NOT to Use RSC

- **Highly interactive apps** where most components need state/effects (Figma, Google Docs) — most components would be Client Components anyway.
- **No server runtime available** — RSC requires a server or edge runtime.
- Your framework **doesn't support RSC** — currently only Next.js has full RSC support. Remix, Astro, and others are exploring it.

### Sample Code (Next.js App Router)

```tsx
// app/products/[id]/page.tsx — Server Component (default)
// ✅ Can use async/await, access DB directly, zero JS shipped

import { db } from '@/lib/db';
import { AddToCart } from './AddToCart'; // Client Component

export default async function ProductPage({ params }) {
  // Direct DB query — no API route needed, no useEffect, no loading state
  const product = await db.products.findUnique({ where: { id: params.id } });
  const reviews = await db.reviews.findMany({ where: { productId: params.id } });

  return (
    <main>
      {/* All of this is Server-rendered, zero JS */}
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <p className="price">${product.price}</p>

      {/* Only this component ships JS to the browser */}
      <AddToCart productId={product.id} />

      <section>
        <h2>Reviews ({reviews.length})</h2>
        {reviews.map(r => (
          <div key={r.id}>
            <strong>{r.author}</strong>
            <p>{r.text}</p>
          </div>
        ))}
      </section>
    </main>
  );
}
```

```tsx
// app/products/[id]/AddToCart.tsx — Client Component
'use client';  // ← This directive marks it as a Client Component

import { useState } from 'react';

export function AddToCart({ productId }: { productId: string }) {
  const [adding, setAdding] = useState(false);

  async function handleClick() {
    setAdding(true);
    await fetch('/api/cart', {
      method: 'POST',
      body: JSON.stringify({ productId }),
    });
    setAdding(false);
  }

  return (
    <button onClick={handleClick} disabled={adding}>
      {adding ? 'Adding...' : 'Add to Cart'}
    </button>
  );
}
```

> **Key takeaway:** The product name, description, price, and all reviews are rendered on the server with **zero JavaScript**. Only the tiny `AddToCart` button ships JS. On a page with 50 reviews, this could mean **hundreds of KB less JavaScript** compared to a fully client-rendered approach.

[⬆ Back to Top](#top)

---

## Comparison Matrix

| Pattern | Render Location | Render Timing | FCP | TTI | SEO | Server Cost | Freshness | Complexity |
|---|---|---|---|---|---|---|---|---|
| **CSR** | Browser | On every visit | 🔴 Slow | 🟢 Fast (= FCP) | 🔴 Poor | 🟢 None | 🟢 Always fresh | 🟢 Low |
| **SSR** | Server | On every request | 🟢 Fast | 🟡 Delayed (hydration) | 🟢 Excellent | 🔴 High | 🟢 Always fresh | 🟡 Medium |
| **CSR + SSR (Universal)** | Server → Browser | First load: server; then client | 🟢 Fast | 🟡 Delayed (hydration) | 🟢 Excellent | 🟡 Medium | 🟢 Always fresh | 🔴 High |
| **SSG** | Build machine | At build time | 🟢 Fastest | 🟢 Fast | 🟢 Excellent | 🟢 None | 🔴 Stale (until rebuild) | 🟢 Low |
| **ISR** | Build → Server (bg) | Build + on-demand bg regen | 🟢 Fastest | 🟢 Fast | 🟢 Excellent | 🟡 Low | 🟡 Near-fresh | 🟡 Medium |
| **Prerendering** | Headless browser | Build or on-demand | 🟡 Medium | 🟡 Medium | 🟢 Good | 🟡 Medium | 🟡 Depends | 🟡 Medium |
| **RSC** | Server (components) + Client (interactive) | On request (Server) + hydrate (Client) | 🟢 Fast | 🟢 Fast (less JS) | 🟢 Excellent | 🟡 Medium | 🟢 Always fresh | 🟡 Medium |

[⬆ Back to Top](#top)

---

## Decision Flowchart When to Pick What

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
       └─ NO  → Using Next.js App Router?
                  │
                  ├─ YES → ✅ RSC (Server Components by default + Client where needed)
                  └─ NO  → Pick from above based on requirements
```

[⬆ Back to Top](#top)

---

## Real World Pattern Usage

| Company / Product | Pattern | Why |
|---|---|---|
| **Gmail, Figma** | CSR | Behind auth, highly interactive, no SEO needed |
| **Amazon Product Pages** | SSR | SEO critical, personalized (price, recommendations), data changes constantly |
| **Next.js Docs, Gatsby blogs** | SSG | Content from markdown/CMS, changes infrequently, maximum performance |
| **Vercel, Shopify Storefronts** | Universal (CSR+SSR) + ISR | SEO + interactivity + near-fresh content without full SSR cost |
| **Next.js App Router apps** | RSC + Streaming | Minimal JS, direct DB access, selective hydration |
| **Legacy Angular SPAs** | Prerendering | Quick SEO fix without rewriting to SSR |

[⬆ Back to Top](#top)

---

## Common Pitfalls

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

[⬆ Back to Top](#top)

---

## The Modern Approach Streaming SSR and Selective Hydration

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

[⬆ Back to Top](#top)

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
| Minimal JS + direct server access + selective interactivity | **RSC (React Server Components)** |
| Best of everything (modern) | **Streaming SSR + RSC + Selective Hydration** |

[⬆ Back to Top](#top)

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design

[⬆ Back to Top](#top)
