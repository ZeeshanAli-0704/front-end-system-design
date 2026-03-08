# CSS, JavaScript & UI Optimization – Frontend Performance

CSS and JavaScript are render-blocking resources by default. The browser cannot paint anything until CSS is parsed (CSSOM) and synchronous JS is executed. Optimizing how these resources load is critical for First Contentful Paint (FCP) and Time to Interactive (TTI).

## Table of Contents

- [1. Critical CSS Rendering](#1-critical-css-rendering)
- [2. Non-Critical CSS (Async Loading)](#2-non-critical-css-async-loading)
- [3. Lazy Loading CSS (Media-Based)](#3-lazy-loading-css-media-based)
- [4. JS-Based CSS Loading](#4-js-based-css-loading)
- [5. Async and Defer Strategy](#5-async-and-defer-strategy)
- [6. Module Loading and Tree Shaking](#6-module-loading-and-tree-shaking)
- [7. Configurable UI (Conditional Rendering)](#7-configurable-ui-conditional-rendering)
- [8. Streaming UI Rendering](#8-streaming-ui-rendering)
- [Key Takeaways](#key-takeaways)

---

## 1. Critical CSS Rendering

**Concept:**
Extract only the CSS rules needed to render above-the-fold content and inline them directly in the HTML. The rest of the CSS loads asynchronously. This eliminates the render-blocking CSS download for initial paint.

**Normal CSS loading:**

```
HTML downloaded
  → External CSS file requested (blocks rendering)
  → Entire CSS downloaded (50-200KB)
  → CSSOM built
  → First paint happens
  ⏱ Total blocking time: 200-800ms
```

**With Critical CSS:**

```
HTML downloaded (includes inlined critical CSS ~5-15KB)
  → CSSOM for above-the-fold built immediately
  → First paint happens FAST
  → Full CSS loads asynchronously in background
  ⏱ Total blocking time: ~0ms
```

**Example:**

```html
<head>
  <!-- Critical CSS inlined -->
  <style>
    /* Only above-the-fold styles (~5-15KB) */
    body { margin: 0; font-family: system-ui, sans-serif; }
    .header { background: #1a1a2e; color: white; padding: 16px; }
    .hero { max-width: 1200px; margin: 0 auto; padding: 40px 20px; }
    .hero h1 { font-size: 2.5rem; margin-bottom: 16px; }
    .hero img { width: 100%; height: auto; }
  </style>

  <!-- Full CSS loads asynchronously -->
  <link rel="preload" href="/css/main.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
  <noscript><link rel="stylesheet" href="/css/main.css"></noscript>
</head>
```

**How to extract Critical CSS automatically:**

```javascript
// Using the 'critical' npm package in build step
const critical = require('critical');

critical.generate({
  inline: true,
  base: 'dist/',
  src: 'index.html',
  target: 'index-critical.html',
  width: 1300,
  height: 900,
  dimensions: [
    { width: 414, height: 896 },   // Mobile
    { width: 1300, height: 900 }   // Desktop
  ]
});
```

**Tools for Critical CSS extraction:**
- `critical` (npm) – Automated extraction and inlining
- `critters` (Webpack plugin) – Build-time critical CSS
- `penthouse` – Headless browser-based extraction
- Next.js – Automatic CSS inlining for pages

---

## 2. Non-Critical CSS (Async Loading)

**Concept:**
CSS that is not needed for the initial viewport (below-the-fold styles, modal styles, footer styles) should load without blocking rendering.

**Technique 1 – Preload then apply:**

```html
<link rel="preload" href="non-critical.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
<noscript><link rel="stylesheet" href="non-critical.css"></noscript>
```

**How it works:**
1. `rel="preload"` downloads the file without applying it
2. `onload` fires when download completes
3. `this.rel='stylesheet'` applies the CSS
4. `this.onload=null` prevents infinite loop in some browsers
5. `noscript` fallback for JavaScript-disabled browsers

**Technique 2 – Media attribute trick:**

```html
<!-- Browser downloads but does not apply (media doesn't match) -->
<link rel="stylesheet" href="non-critical.css" media="print" onload="this.media='all'">
```

**How it works:**
1. `media="print"` tells browser this CSS is only for printing → non-blocking
2. Browser still downloads it (lower priority)
3. `onload` fires → `this.media='all'` applies it for all media types

---

## 3. Lazy Loading CSS (Media-Based)

**Concept:**
Load CSS files only under specific conditions using the `media` attribute. The browser downloads all stylesheets but only applies those whose media query matches.

**Example – Load only when conditions match:**

```html
<!-- Always loaded and applied -->
<link rel="stylesheet" href="base.css">

<!-- Downloaded but applied ONLY when printing -->
<link rel="stylesheet" href="print.css" media="print">

<!-- Applied only on screens wider than 768px -->
<link rel="stylesheet" href="desktop.css" media="(min-width: 768px)">

<!-- Applied only in portrait orientation -->
<link rel="stylesheet" href="portrait.css" media="(orientation: portrait)">

<!-- Applied only when user prefers dark mode -->
<link rel="stylesheet" href="dark-theme.css" media="(prefers-color-scheme: dark)">
```

**Key behavior:**
- Browser downloads ALL linked stylesheets regardless of media match
- But non-matching stylesheets are downloaded at lowest priority
- Non-matching stylesheets do NOT block rendering
- This is a native browser optimization, no JavaScript needed

**Practical example – Conditional component CSS:**

```html
<!-- Critical above-the-fold styles – render-blocking (intentional) -->
<link rel="stylesheet" href="header.css">
<link rel="stylesheet" href="hero.css">

<!-- Below-the-fold – non-blocking via media trick -->
<link rel="stylesheet" href="footer.css" media="print" onload="this.media='all'">
<link rel="stylesheet" href="comments.css" media="print" onload="this.media='all'">
```

---

## 4. JS-Based CSS Loading

**Concept:**
Use JavaScript to load additional stylesheets after the initial page render. This gives complete programmatic control over when and which CSS files are loaded.

**Example – Dynamic stylesheet injection:**

```javascript
// Load CSS after page is interactive
function loadCSS(href) {
  const link = document.createElement('link');
  link.rel = 'stylesheet';
  link.href = href;
  document.head.appendChild(link);
}

// Load non-critical styles after DOM is ready
document.addEventListener('DOMContentLoaded', () => {
  loadCSS('/css/animations.css');
  loadCSS('/css/below-fold.css');
});
```

**Example – Load CSS on user interaction:**

```javascript
// Load modal styles only when modal is triggered
document.querySelector('.open-modal').addEventListener('click', () => {
  loadCSS('/css/modal.css');
  // Then open modal after styles are ready
}, { once: true });
```

**Example – Load CSS based on feature detection:**

```javascript
// Load CSS only if the browser supports a feature
if ('IntersectionObserver' in window) {
  loadCSS('/css/lazy-images.css');
}

// Load CSS based on viewport
if (window.innerWidth > 1024) {
  loadCSS('/css/desktop-sidebar.css');
}
```

**Benefits:**
- Complete control over loading conditions
- Load styles only when actually needed
- Avoid blocking page render with unused CSS
- Can combine with feature detection and user interaction

---

## 5. Async and Defer Strategy

**Concept:**
By default, `<script>` tags block HTML parsing. The browser stops building the DOM, downloads the script, executes it, then resumes parsing. `async` and `defer` change this behavior.

**Three loading modes:**

```
Normal <script>:
HTML parsing ████████░░░░░░░░░░░████████████
Script download      ░░░█████░░░░░░░░
Script execution         ░░░░░███░░░░░

async <script>:
HTML parsing ████████████████░░░█████████████
Script download  ░░░█████░░░░░░░░░░░
Script execution         ░░░███░░░░░░░

defer <script>:
HTML parsing ████████████████████████████████
Script download  ░░░█████░░░░░░░░░░░░░░░░░░
Script execution                             ███
                                             ^ After HTML parsing complete
```

**Comparison:**

| Feature | Normal | async | defer |
|---------|--------|-------|-------|
| Blocks HTML parsing | Yes | Only during execution | No |
| Download | Sequential | Parallel | Parallel |
| Execution timing | Immediately | When downloaded | After DOM ready |
| Execution order | Ordered | NOT guaranteed | Guaranteed order |
| DOMContentLoaded | Waits for script | May fire before/after | Fires after scripts |

**Example:**

```html
<head>
  <!-- Critical: must run before DOM (rare) -->
  <script src="critical-polyfill.js"></script>

  <!-- Analytics: order doesn't matter, run when ready -->
  <script src="analytics.js" async></script>
  <script src="tracking.js" async></script>

  <!-- App scripts: need DOM, order matters -->
  <script src="vendor.js" defer></script>
  <script src="utils.js" defer></script>
  <script src="app.js" defer></script>
</head>
```

**When to use which:**

| Use Case | Strategy |
|----------|----------|
| Third-party analytics/ads | `async` |
| Chat widgets, social embeds | `async` |
| Main app bundle | `defer` |
| Library dependencies (React, Vue) | `defer` |
| Critical polyfills | Normal (no attribute) |
| A/B testing scripts | Normal (must run before render) |

---

## 6. Module Loading and Tree Shaking

**Concept:**
ES Modules enable the browser (and bundlers) to understand dependency relationships between files. This unlocks tree shaking – the ability to remove unused code from the final bundle.

**ES Module loading in browser:**

```html
<!-- Module script: automatically deferred, strict mode -->
<script type="module" src="app.js"></script>

<!-- Fallback for old browsers -->
<script nomodule src="app-legacy.js"></script>
```

**Module behavior:**
- `type="module"` scripts are deferred by default (no need for `defer`)
- They run in strict mode automatically
- They support `import` / `export` syntax
- `nomodule` is ignored by modern browsers, runs in legacy browsers

**Tree shaking example:**

```javascript
// math.js – Library with multiple exports
export function add(a, b) { return a + b; }
export function subtract(a, b) { return a - b; }
export function multiply(a, b) { return a * b; }
export function divide(a, b) { return a / b; }

// app.js – Only imports 'add'
import { add } from './math.js';
console.log(add(2, 3));

// After tree shaking:
// subtract, multiply, divide are REMOVED from bundle
// Only 'add' function is included
```

**Requirements for tree shaking to work:**
- Must use ES Module syntax (`import` / `export`)
- Must NOT use CommonJS (`require` / `module.exports`)
- Bundler must be configured for production mode
- Code must be free of side effects (or marked via `sideEffects` in package.json)

**package.json sideEffects configuration:**

```json
{
  "name": "my-library",
  "sideEffects": false
}
```

```json
{
  "name": "my-library",
  "sideEffects": [
    "**/*.css",
    "./src/polyfills.js"
  ]
}
```

**Dynamic imports for code splitting:**

```javascript
// Static import – always included in bundle
import { heavyFunction } from './heavy-module.js';

// Dynamic import – loaded only when needed (separate chunk)
async function handleUserAction() {
  const { heavyFunction } = await import('./heavy-module.js');
  heavyFunction();
}

// React lazy loading
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));
```

---

## 7. Configurable UI (Conditional Rendering)

**Concept:**
Render only the components and features that are actually needed for the current user, device, or context. Avoid rendering invisible or unnecessary UI elements that consume memory, CPU, and bandwidth.

**Core principles:**
- Render only what is visible
- Load components on demand
- Adapt UI based on device, network, and user context

**Example 1 – Conditional component rendering:**

```jsx
function SearchPage({ config }) {
  return (
    <div>
      {/* Render search bar only if enabled in config */}
      {config.showSearchBar && <SearchBar />}

      {/* Render filters only on desktop */}
      {!isMobile && <FilterSidebar />}

      {/* Render recommendations only for logged-in users */}
      {user.isLoggedIn && <Recommendations userId={user.id} />}

      <SearchResults />
    </div>
  );
}
```

**Example 2 – Feature flags for conditional loading:**

```javascript
// Feature flag configuration (from API or config file)
const featureFlags = {
  enableChat: true,
  enableVideoPlayer: false,
  enableAdvancedSearch: true,
};

function App() {
  return (
    <div>
      <Header />
      <MainContent />

      {/* Components only load if feature is enabled */}
      {featureFlags.enableChat && <ChatWidget />}
      {featureFlags.enableVideoPlayer && <VideoPlayer />}
      {featureFlags.enableAdvancedSearch && <AdvancedSearch />}
    </div>
  );
}
```

**Example 3 – Network-aware rendering:**

```javascript
function MediaSection() {
  const connection = navigator.connection;
  const isSlowNetwork = connection?.effectiveType === '2g' || connection?.effectiveType === '3g';

  if (isSlowNetwork) {
    // Show lightweight version on slow networks
    return <ImageThumbnails />;
  }

  // Show rich media on fast networks
  return <VideoCarousel />;
}
```

**Benefits:**
- Smaller initial DOM size → Faster rendering
- Less JavaScript parsed and executed
- Reduced memory usage
- Better experience on low-end devices

---

## 8. Streaming UI Rendering

**Concept:**
Instead of waiting for all data and components to be ready before showing anything, stream the UI in chunks. Show important sections first, then progressively fill in secondary content.

**Traditional rendering vs Streaming:**

```
Traditional SSR:
  Server fetches ALL data → Server renders ALL HTML → Sends complete HTML → Browser shows page
  User sees blank page for entire duration ████████████████████████░░

Streaming SSR:
  Server sends shell immediately → Streams components as data arrives → Browser shows incrementally
  User sees header/shell instantly ██░░ → content fills in ████ → complete ████████████████████
```

**React 18 Streaming SSR with Suspense:**

```jsx
import { Suspense } from 'react';

function ProductPage() {
  return (
    <div>
      {/* Streamed immediately – no data dependency */}
      <Header />
      <ProductTitle />

      {/* Streamed when product details data is ready */}
      <Suspense fallback={<ProductDetailsSkeleton />}>
        <ProductDetails />
      </Suspense>

      {/* Streamed when reviews data is ready */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <ProductReviews />
      </Suspense>

      {/* Streamed when recommendations data is ready */}
      <Suspense fallback={<RecommendationsSkeleton />}>
        <Recommendations />
      </Suspense>

      {/* Streamed immediately – no data dependency */}
      <Footer />
    </div>
  );
}
```

**How React Streaming works:**
1. Server sends HTML shell with `Header`, `ProductTitle`, `Footer` immediately
2. `ProductDetailsSkeleton` placeholder is sent while data fetches
3. When product details data is ready, server streams the real HTML to replace skeleton
4. Same for reviews and recommendations – each streams independently
5. Browser progressively replaces skeletons with real content using inline scripts

**Next.js App Router streaming example:**

```jsx
// app/product/[id]/page.js
import { Suspense } from 'react';
import { ProductDetails, Reviews, Recommendations } from './components';
import { Skeleton } from './skeletons';

export default function ProductPage({ params }) {
  return (
    <main>
      <h1>Product Page</h1>

      <Suspense fallback={<Skeleton type="details" />}>
        <ProductDetails id={params.id} />
      </Suspense>

      <Suspense fallback={<Skeleton type="reviews" />}>
        <Reviews productId={params.id} />
      </Suspense>

      <Suspense fallback={<Skeleton type="recommendations" />}>
        <Recommendations productId={params.id} />
      </Suspense>
    </main>
  );
}
```

**Benefits of streaming UI:**
- TTFB (Time to First Byte) drops dramatically – shell arrives in milliseconds
- Users see meaningful content while data-heavy sections load
- Slow API calls don't block the entire page
- Each section is independent – one slow section doesn't delay others
- Skeleton screens provide visual structure immediately

---

## Key Takeaways

### CSS Optimization
- Inline critical CSS for above-the-fold content, async-load the rest
- Use the media attribute trick to make non-critical CSS non-blocking
- Use media-based lazy loading for conditional CSS (print, dark mode, orientation)
- Load CSS via JavaScript for interaction-dependent styles

### JavaScript Optimization
- Use `defer` for app scripts (preserves order, runs after DOM parsing)
- Use `async` for independent third-party scripts (analytics, ads)
- Enable tree shaking with ES Modules and `sideEffects: false`
- Use dynamic `import()` for code splitting heavy features

### UI Optimization
- Conditionally render components based on device, network, and user context
- Use feature flags to control which components load
- Stream UI with React 18 Suspense for progressive rendering
- Show skeletons while streaming data-heavy sections

### Performance Metrics Impact

| Optimization | LCP | FCP | CLS | TTI | TTFB |
|-------------|-----|-----|-----|-----|------|
| Critical CSS | | +++ | + | | |
| Async CSS loading | | ++ | | + | |
| JS async/defer | | + | | +++ | |
| Tree shaking | | | | ++ | |
| Code splitting | | | | +++ | |
| Conditional rendering | | + | | ++ | |
| Streaming SSR | | ++ | | ++ | +++ |

`+++` = Major impact, `++` = Moderate impact, `+` = Minor impact
