# Performance Monitoring & Observability

### A Complete Frontend System Design Guide — Theory, Patterns & Interview Focus

You can build the most elegant UI in the world, but if it takes 5 seconds to load or janks on every scroll, users leave. Performance monitoring isn't a one-time audit — it's a **continuous feedback loop** that tells you **when things are slow, why they're slow, and for whom**.

This guide covers **Core Web Vitals**, **identifying and fixing bottlenecks**, **monitoring strategies**, **error tracking**, **performance budgets**, and **the tools & techniques** (Chrome DevTools, Lighthouse, RUM) that make all of it actionable — all from a **conceptual, interview-ready** perspective.

---

<a id="top"></a>

## Table of Contents

- [Core Web Vitals What They Are and Why They Matter](#core-web-vitals-what-they-are-and-why-they-matter)
- [Identifying Performance Bottlenecks](#identifying-performance-bottlenecks)
- [Fixing Performance Issues A Systematic Approach](#fixing-performance-issues-a-systematic-approach)
- [Chrome DevTools and Available Tools](#chrome-devtools-and-available-tools)
- [Real User Monitoring (RUM) vs Synthetic Monitoring](#real-user-monitoring-rum-vs-synthetic-monitoring)
- [Error Tracking and Logging](#error-tracking-and-logging)
- [Performance Budgets](#performance-budgets)
- [The Three Pillars of Observability](#the-three-pillars-of-observability)
- [Tradeoffs Gotchas and Common Pitfalls](#tradeoffs-gotchas-and-common-pitfalls)
- [Interview Questions and Answers](#interview-questions-and-answers)
[⬆ Back to Top](#top)

---

## Core Web Vitals What They Are and Why They Matter

**Core Web Vitals** are a set of user-centric performance metrics defined by Google. Since **June 2021**, they are **Google ranking signals**, making them critical for both UX and SEO. They answer three fundamental questions about user experience:

```
┌──────────────────────────────────────────────────────────┐
│                   Core Web Vitals                         │
│                                                           │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   │
│   │     LCP     │   │     INP     │   │     CLS     │   │
│   │  (Loading)  │   │(Responsive) │   │ (Stability) │   │
│   │             │   │             │   │             │   │
│   │ "How fast   │   │ "How fast   │   │ "How stable │   │
│   │  does the   │   │  does it    │   │  is the     │   │
│   │  main       │   │  react to   │   │  visual     │   │
│   │  content    │   │  my input?" │   │  layout?"   │   │
│   │  appear?"   │   │             │   │             │   │
│   └─────────────┘   └─────────────┘   └─────────────┘   │
└──────────────────────────────────────────────────────────┘
```

### The 75th Percentile Rule

Google evaluates Core Web Vitals at the **75th percentile** of all page visits. This means 75% of your users must have a "good" experience for you to pass. If your median LCP is 2.0s but your p75 is 3.5s, you **fail** the assessment.

---

### LCP — Largest Contentful Paint

**What it measures:** The time from page load start to when the **largest visible content element** (hero image, heading text, video poster) finishes rendering in the viewport.

| Rating | Time |
|---|---|
| Good | ≤ 2.5s |
| Needs Improvement | 2.5s – 4.0s |
| Poor | > 4.0s |

**LCP candidate elements:** `<img>`, `<video>` poster, elements with `background-image`, and block-level text elements like `<h1>`, `<p>`.

**What causes poor LCP and how to fix it:**

| Cause | Why It's Slow | Fix |
|---|---|---|
| Slow server response (high TTFB) | Server takes too long to return HTML | CDN, edge caching, optimize backend queries |
| Render-blocking CSS/JS | Large CSS/JS files block the parser and delay paint | Inline critical CSS, defer non-critical CSS/JS |
| Slow resource loading | Hero image is unoptimized (2MB PNG) | Compress images, use WebP/AVIF, responsive `srcset` |
| Client-side rendering | SPA waits for JS → API → render cycle | Use SSR/SSG, stream HTML from server |
| Lazy-loaded LCP element | Hero image has `loading="lazy"` — browser delays it | **Never** lazy-load above-the-fold content |
| No resource prioritization | Browser treats LCP image same as everything else | Use `fetchpriority="high"` on the LCP image, `<link rel="preload">` |
| Redirect chains | Multiple 301/302 redirects before actual page | Minimize redirects — each adds a full round trip |

---

### INP — Interaction to Next Paint

**What it measures:** INP measures the latency of **all** user interactions (clicks, taps, key presses) throughout the page's lifetime, and reports a value representing the worst interaction latency. It replaced **FID** in March 2024.

| Rating | Time |
|---|---|
| Good | ≤ 200ms |
| Needs Improvement | 200ms – 500ms |
| Poor | > 500ms |

**Why INP replaced FID:**

| Aspect | FID (Deprecated) | INP (Current) |
|---|---|---|
| What's measured | Delay of **first** input only | **All** interactions throughout page life |
| Scope | Single event | Full session |
| Includes processing time? | No — only input delay | Yes — input delay + processing + presentation delay |
| Real-world coverage | Misses 99% of interactions | Comprehensive |

**The three phases of an interaction:**

```
┌───────────────┬─────────────────────┬──────────────────────┐
│  Input Delay  │  Processing Time    │  Presentation Delay  │
│  (main thread │  (event handlers    │  (render, layout,    │
│   was busy)   │   executing)        │   paint, composite)  │
└───────────────┴─────────────────────┴──────────────────────┘
               ← ──── Total INP Latency ──── →
```

- **Input Delay:** User clicked but the main thread was blocked by another task — the event handler is queued.
- **Processing Time:** How long your event handler code takes to execute.
- **Presentation Delay:** After handler finishes, how long until the browser paints the visual update.

**What causes poor INP and how to fix it:**

| Cause | Why It's Slow | Fix |
|---|---|---|
| Long tasks on main thread | Heavy JS blocks the thread during interaction | Break tasks with `scheduler.yield()` or `setTimeout` chunking |
| Large DOM size | 10,000+ nodes — layout/style recalculation is expensive | Virtualize long lists, simplify DOM structure |
| Expensive re-renders | Framework re-renders entire component tree | Memoize components, colocate state, use selectors |
| Forced synchronous layout | Reading layout props after writes causes "layout thrashing" | Batch DOM reads before writes |
| Third-party scripts | Ad/analytics scripts competing for main thread | Load async, defer, or move to Web Workers |
| No yielding in long handlers | Single handler runs 300ms without breaks | Yield to browser between chunks of work |

---

### CLS — Cumulative Layout Shift

**What it measures:** CLS quantifies how much visible content **shifts unexpectedly** during the page's lifetime. It measures **visual stability** — content jumping around without user interaction.

| Rating | Score |
|---|---|
| Good | ≤ 0.1 |
| Needs Improvement | 0.1 – 0.25 |
| Poor | > 0.25 |

**How CLS is calculated:**
```
Layout Shift Score = Impact Fraction × Distance Fraction
```
- **Impact Fraction:** Percentage of viewport affected by the shift
- **Distance Fraction:** Greatest distance any element moved (as fraction of viewport)

CLS groups shifts into **session windows** (bursts within 1 second, max 5-second window). The **largest session window** is reported.

**What causes poor CLS and how to fix it:**

| Cause | Why It Shifts | Fix |
|---|---|---|
| Images without dimensions | Image loads and pushes content down | Always set `width`/`height` or use CSS `aspect-ratio` |
| Ads/embeds without reserved space | Ad slot loads a 300px banner, pushing everything | Reserve space with min-height/aspect-ratio placeholders |
| Dynamically injected content | Cookie banner or notification pushes page down | Use `transform` animations, fixed/sticky positioning |
| Web font swap (FOUT) | Fallback font renders → web font loads → text reflows | Use `font-display: optional`, preload fonts, use `size-adjust` |
| Client-side rendering | Skeleton loads, then real content arrives with different dimensions | SSR/SSG, skeleton screens with correct dimensions |
| CSS animations using layout properties | Animating `top`/`left`/`width`/`height` triggers layout | Use `transform` (`translateY`, `scale`) — no layout cost |

---

### Supplementary Metrics Worth Knowing

Beyond the three Core Web Vitals, these metrics provide additional context:

| Metric | What It Measures | Good Threshold |
|---|---|---|
| **TTFB** (Time to First Byte) | Server responsiveness — time until first byte of HTML arrives | ≤ 800ms |
| **FCP** (First Contentful Paint) | Time until first text/image is painted on screen | ≤ 1.8s |
| **TTI** (Time to Interactive) | Time until page is fully interactive (no long tasks blocking) | ≤ 3.8s |
| **TBT** (Total Blocking Time) | Total time the main thread was blocked between FCP and TTI | ≤ 200ms |
| **Speed Index** | How quickly visible content is populated (visual completeness) | ≤ 3.4s |

> **Interview insight:** TTFB is not a Core Web Vital, but it's the **root cause** of most LCP issues. If TTFB is 2 seconds, your LCP can never be below 2 seconds. Always check TTFB first when debugging LCP.

[⬆ Back to Top](#top)

---

## Identifying Performance Bottlenecks

The hardest part of performance work isn't fixing issues — it's **finding them**. Here's a systematic approach to identifying bottlenecks across the loading, rendering, and interaction lifecycle.

### The Performance Investigation Funnel

```
Step 1: What's the symptom?
         │
         ├── Page loads slowly        → Focus on loading metrics (LCP, TTFB, FCP)
         ├── Page feels janky/laggy   → Focus on interaction metrics (INP, TBT)
         ├── Content jumps around     → Focus on stability metrics (CLS)
         └── Specific action is slow  → Focus on that interaction's trace
         │
Step 2: Where is the bottleneck?
         │
         ├── Server response (TTFB)   → Backend/CDN issue
         ├── Resource downloading     → Large assets, no compression, no CDN
         ├── Render blocking          → CSS/JS blocking first paint
         ├── Main thread busy         → Heavy JS execution, long tasks
         ├── Layout thrashing         → DOM reads/writes interleaved
         └── Memory leaks             → Memory growing over time
         │
Step 3: Why is it slow?
         │
         Use Chrome DevTools to profile and trace the root cause
```

### The Six Categories of Frontend Bottlenecks

```
┌──────────────────────────────────────────────────────────────────┐
│                Frontend Performance Bottlenecks                   │
│                                                                   │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────────┐  │
│  │  Network   │ │  Parsing   │ │  Rendering │ │  JavaScript  │  │
│  │            │ │  & Loading │ │  & Layout  │ │  Execution   │  │
│  │ • TTFB     │ │ • Large    │ │ • Complex  │ │ • Long tasks │  │
│  │ • Large    │ │   HTML     │ │   selectors│ │ • Unoptimized│  │
│  │   payloads │ │ • Render-  │ │ • Forced   │ │   frameworks │  │
│  │ • No CDN   │ │   blocking │ │   sync     │ │ • Memory     │  │
│  │ • No       │ │   resources│ │   layout   │ │   leaks      │  │
│  │   compress │ │ • Too many │ │ • Excessive│ │ • No code    │  │
│  │ • DNS/TLS  │ │   requests │ │   repaints │ │   splitting  │  │
│  └────────────┘ └────────────┘ └────────────┘ └──────────────┘  │
│                                                                   │
│  ┌────────────┐ ┌──────────────────────────────────────────────┐ │
│  │  Memory    │ │           Third-Party Scripts                 │ │
│  │            │ │                                               │ │
│  │ • Detached │ │ • Analytics, ads, chat widgets, tag managers │ │
│  │   DOM nodes│ │ • Competing for main thread time             │ │
│  │ • Event    │ │ • Blocking critical rendering                │ │
│  │   listener │ │ • Uncontrolled payload sizes                 │ │
│  │   leaks    │ │                                               │ │
│  └────────────┘ └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### How to Identify Each Bottleneck Type

#### 1. Network Bottlenecks
**Symptoms:** High TTFB, slow resource loading, waterfalls with large gaps.

**How to find:**
- Chrome DevTools → **Network panel** → Sort by "Time" column → Look for requests > 1s
- Check the **Waterfall** column — see if resources are loading sequentially instead of parallel
- Look at **TTFB** (green bar in waterfall) — if it's large, the problem is server-side
- Check **Size** column — large uncompressed assets are a red flag
- Filter by "JS" or "CSS" — check if too many files are being loaded

**Signs in the waterfall:**
| Pattern | What It Means |
|---|---|
| Long green bar (TTFB) | Server is slow to respond |
| Long blue bar (download) | Asset is too large |
| Resources loading one after another | No parallelism — likely dependency chain or HTTP/1.1 |
| Many small requests | Consider bundling or using HTTP/2 multiplexing |
| Resources from many different domains | High DNS/TLS overhead per domain |

#### 2. Render-Blocking Resources
**Symptoms:** Long gap between TTFB and FCP. Page is white for too long.

**How to find:**
- Lighthouse audit → "Eliminate render-blocking resources" — lists every CSS/JS file that blocks rendering
- DevTools → **Performance panel** → Record → Look for long "Parse HTML" → "Recalculate Style" → "Layout" before first paint
- DevTools → **Coverage tab** (`Ctrl+Shift+P` → "Show Coverage") — shows what percentage of each CSS/JS file is actually used on this page

**Key insight:** A CSS file is render-blocking by default. A `<script>` tag without `async`/`defer` is parser-blocking. Both delay first paint.

#### 3. Main Thread / JavaScript Bottlenecks
**Symptoms:** Poor INP, janky scrolling, slow interactions, high TBT.

**How to find:**
- DevTools → **Performance panel** → Record an interaction → Look for **Long Tasks** (red-flagged bars > 50ms)
- The "Main" thread flame chart shows exactly which functions are consuming time
- **Bottom-Up tab** → Sort by "Self Time" to find the most expensive individual functions
- **Call Tree tab** → Shows the call hierarchy, find root causes
- Check for **"Recalculate Style"** and **"Layout"** entries — signs of layout thrashing

**What Long Tasks look like:**
```
Main Thread Timeline
─────────────────────────────────────────────
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│ 120ms Long Task (red flag!)
│   ▓▓▓│ 20ms task
│ ▓▓▓▓▓▓▓▓▓│ 60ms Long Task
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│ 200ms Long Task (very bad!)
─────────────────────────────────────────────
Any task > 50ms is a "Long Task" that blocks interactions
```

#### 4. Layout & Rendering Bottlenecks
**Symptoms:** CLS issues, janky animations, slow scroll.

**How to find:**
- DevTools → **Performance panel** → Look for frequent **"Layout"** entries (purple bars) — each is a forced reflow
- DevTools → **Rendering tab** → Enable "Layout Shift Regions" — flashes blue on areas that shift
- DevTools → **Rendering tab** → Enable "Paint flashing" — green flashes show what's being repainted
- DevTools → **Layers panel** — shows which elements have their own compositing layer

**Layout thrashing pattern:** Reading then writing to DOM repeatedly forces the browser to recalculate layout between each read-write pair. The fix is to batch all reads first, then all writes.

#### 5. Memory Issues
**Symptoms:** Page gets slower over time, eventually crashes. Tab's memory in Task Manager keeps growing.

**How to find:**
- DevTools → **Memory panel** → Take heap snapshots at different points → Compare for growing objects
- DevTools → **Performance Monitor** (real-time) → Watch "JS heap size" and "DOM Nodes" over time
- Chrome Task Manager (`Shift+Esc`) → Check if your tab's memory keeps growing

**Common causes:**
| Cause | What Happens | Fix |
|---|---|---|
| Detached DOM nodes | Removed elements still referenced in JS | Null references when removing elements |
| Event listeners not cleaned up | Each re-render adds new listeners without removing old ones | Use `removeEventListener`, or cleanup in React's `useEffect` return |
| Growing arrays/maps | Data accumulates without bounds | Set limits, use LRU eviction |
| Closures retaining large scope | Functions hold references to outer scope variables | Be mindful of what closures capture |
| `setInterval` without `clearInterval` | Timers keep running even after component unmounts | Always clear in cleanup/teardown |

#### 6. Third-Party Script Issues
**Symptoms:** Everything seems optimized but Lighthouse still shows poor scores. Long Tasks you don't recognize.

**How to find:**
- DevTools → **Performance panel** → Look at Long Tasks → Check if the source URL is a third-party domain
- DevTools → **Network panel** → Filter by "Third-party" → Check sizes and timing
- Lighthouse → "Reduce the impact of third-party code" audit
- Use the `web-vitals` attribution build to see if third-party scripts cause poor INP/LCP
- Chrome → `chrome://inspect/#devices` → Check Service Workers for third-party scripts

[⬆ Back to Top](#top)

---

## Fixing Performance Issues A Systematic Approach

Once you've identified the bottleneck, here's a structured approach to fixing each category:

### Loading Performance Fixes (Improve LCP, TTFB, FCP)

| Problem | Fix | Impact |
|---|---|---|
| High TTFB | Deploy CDN, enable edge caching, optimize backend queries, use streaming SSR | High |
| Large JS bundles | Code splitting (route-based), tree shaking, lazy imports for below-fold features | High |
| Large images | Compress, use WebP/AVIF, responsive `srcset`, proper sizing | High |
| Render-blocking CSS | Inline critical CSS (above-fold styles), defer rest with `media="print"` hack or async loading | High |
| Render-blocking JS | Add `defer` or `async` attribute, move scripts to bottom of body | Medium |
| No resource hints | `<link rel="preload">` for LCP image, `<link rel="preconnect">` for critical origins | Medium |
| Too many HTTP requests | Bundle small files, use HTTP/2 (automatic multiplexing), use image sprites or SVG sprites | Medium |
| No compression | Enable Brotli or Gzip on server/CDN | Medium |
| Redirect chains | Eliminate unnecessary 301/302 redirects | Low-Medium |

### Interaction Performance Fixes (Improve INP, TBT)

| Problem | Fix | Impact |
|---|---|---|
| Long event handlers | Break into async chunks with `scheduler.yield()` or `setTimeout(fn, 0)` | High |
| Heavy computation | Move to Web Worker (runs off main thread) | High |
| Framework over-rendering | Memoize components (`React.memo`, `useMemo`), colocate state closer to where it's used | High |
| Large DOM | Virtualize long lists (react-window, TanStack Virtual), simplify nested structures | Medium |
| Layout thrashing | Batch DOM reads before writes, use `requestAnimationFrame` for DOM mutations | Medium |
| Frequent forced reflows | Avoid reading layout properties (`offsetHeight`, `getBoundingClientRect`) inside write loops | Medium |
| Third-party scripts | Load non-essential scripts with `defer`/`async`, use `Partytown` to run them in a Web Worker | Medium |

### Visual Stability Fixes (Improve CLS)

| Problem | Fix | Impact |
|---|---|---|
| Images without dimensions | Always set `width`/`height` attributes or CSS `aspect-ratio` | High |
| Dynamic content insertion | Reserve space with placeholders, use `min-height` on containers | High |
| Font swapping | Use `font-display: optional` (no shift) or `size-adjust` to match fallback metrics | Medium |
| Injected ads/embeds | Reserve exact space with aspect-ratio containers | Medium |
| CSS animations using layout properties | Use `transform` and `opacity` only — they skip layout and paint phases | Medium |
| Late-loading above-fold content | Prioritize critical content loading, SSR/SSG | Medium |

### The Fix Priority Framework

When multiple issues exist, prioritize by **impact × effort**:

```
                        High Impact
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
         │   QUICK WINS     │   BIG PROJECTS   │
         │   (Do first)     │   (Plan & do)    │
         │                  │                  │
         │  • Preload LCP   │  • SSR/SSG       │
Low ─────│  • Image formats │  • Code splitting│───── High
Effort   │  • Add defer/    │  • Web Workers   │  Effort
         │    async         │  • Virtualization │
         │  • Set img dims  │  • Architecture  │
         │                  │    changes       │
         │   FILL-INS       │   AVOID          │
         │   (Do if time)   │   (Low ROI)      │
         │                  │                  │
         │  • DNS prefetch  │  • Rewriting in  │
         │  • Minor CSS     │    new framework │
         │    cleanup       │  • Over-          │
         │                  │    optimization  │
         └──────────────────┼──────────────────┘
                            │
                        Low Impact
```

[⬆ Back to Top](#top)

---

## Chrome DevTools and Available Tools

### Chrome DevTools — Your Primary Performance Tool

Chrome DevTools has several panels specifically designed for performance investigation. Understanding which panel to use for which problem is crucial.

| DevTools Panel | What It Shows | When to Use |
|---|---|---|
| **Performance** | Flame chart of main thread activity, Long Tasks, network waterfall, screenshots | Profiling specific interactions, diagnosing INP/TBT, finding expensive functions |
| **Network** | All network requests, waterfall timing, sizes, headers | Debugging loading issues, finding large assets, checking compression/caching |
| **Lighthouse** | Full page audit — Performance, Accessibility, SEO, Best Practices scores | Quick overall health check, getting actionable recommendations |
| **Memory** | Heap snapshots, allocation timelines, garbage collection | Debugging memory leaks, finding detached DOM nodes |
| **Coverage** | Shows how much of each CSS/JS file is actually used | Finding dead code to remove or defer |
| **Rendering** | Layout shift visualization, paint flashing, layer borders, FPS meter | Debugging CLS, finding unnecessary repaints, checking compositing |
| **Application** | Storage (Cache, IndexedDB, localStorage), Service Workers | Debugging caching strategies, checking storage usage |
| **Performance Monitor** | Real-time CPU, heap size, DOM nodes, event listeners, style recalculations | Quick live overview of performance health |

### How to Use the Performance Panel (Step by Step)

The Performance panel is the **most powerful** tool for diagnosing bottlenecks:

1. **Open DevTools** → **Performance tab**
2. **Click Record** (or `Ctrl+E`) → Perform the slow action → Stop recording
3. **Read the flame chart top-down:**
   - **Network row** — shows when resources loaded
   - **Frames row** — shows frame rate (red = dropped frames)
   - **Main row** — the flame chart of all main thread activity
   - **Red corners on tasks** = Long Tasks (> 50ms) — these are your targets
4. **Click on a Long Task** → See the call stack in the Bottom-Up / Call Tree tabs
5. **Bottom-Up tab (sort by Self Time)** → Find which functions consumed the most time
6. **Summary tab** → Shows time breakdown: Scripting, Rendering, Painting, Idle

**What to look for in the flame chart:**

| Pattern | Meaning | Fix |
|---|---|---|
| Wide yellow blocks | Long JavaScript execution | Break up tasks, optimize algorithm, Web Worker |
| Wide purple blocks | Layout/style recalculation | Reduce DOM complexity, batch DOM writes |
| Wide green blocks | Paint/composite | Reduce painted area, promote to GPU layer |
| Many narrow purple + yellow alternating | Layout thrashing | Batch reads before writes |
| Large red-cornered blocks | Long Tasks blocking interactions | Yield to browser between chunks |

### How to Use the Network Panel Effectively

| Technique | How | What It Reveals |
|---|---|---|
| Sort by **Size** | Click "Size" column header | Largest assets to optimize |
| Sort by **Time** | Click "Time" column header | Slowest requests |
| Check **Waterfall** | Look at the colored bars | Where time is spent (DNS, TLS, TTFB, download) |
| Use **Throttling** | Change "No throttling" to "Slow 3G" or "Fast 3G" | How your site feels on slow networks |
| Check **Disable cache** | Tick the checkbox | See first-visit experience |
| Filter by type | Click JS, CSS, Img, Font, etc. | Focus on specific resource types |
| Check **gzip/Brotli** | Compare "Size" vs "Content" columns | Is compression enabled? |
| Right-click → "Block request URL" | Block specific scripts | Test impact of removing third-party scripts |

### How to Use the Coverage Tab

1. `Ctrl+Shift+P` → Type "Show Coverage" → Enter
2. Click the reload button in the Coverage panel
3. See a list of all CSS/JS files with **percentage used**

| Usage | What It Means | Action |
|---|---|---|
| < 30% used | Heavy dead code | Extract critical portion, lazy-load the rest |
| 30-70% used | Some dead code | Consider code splitting per route |
| > 70% used | Mostly needed | Focus optimization elsewhere |

### How to Use the Rendering Tab for CLS

1. `Ctrl+Shift+P` → "Show Rendering"
2. Enable **"Layout Shift Regions"** — blue rectangles flash where layout shifts occur
3. Slowly scroll and interact with the page — watch for unexpected blue flashes
4. Enable **"Paint Flashing"** — green rectangles show what's being repainted (excessive repaints = performance drain)

### The Full Toolkit Beyond Chrome DevTools

| Tool | Type | Best For |
|---|---|---|
| **Lighthouse** (in DevTools or CLI) | Lab / Synthetic | Quick audit, actionable recommendations, scores |
| **PageSpeed Insights** | Lab + Field (CrUX) | Checking real-user data alongside lab results |
| **WebPageTest** | Lab (real devices) | Detailed waterfall analysis, filmstrip view, multi-step testing |
| **Chrome UX Report (CrUX)** | Field (28-day rolling) | Google's official real-user performance data per origin |
| **Google Search Console** | Field (CrUX) | SEO impact of Core Web Vitals, URL-level reports |
| **`web-vitals` JS library** | Field (RUM) | Custom RUM collection in your own analytics pipeline |
| **Bundle analyzers** (`webpack-bundle-analyzer`, `source-map-explorer`) | Build-time | Visualizing what's in your JS bundles, finding bloat |
| **`size-limit` / `bundlesize`** | CI/CD | Enforcing bundle size budgets in pull requests |
| **Sentry Performance** | RUM + Tracing | Real-user performance monitoring with error correlation |
| **Datadog RUM / New Relic Browser** | RUM | Enterprise-grade real-user monitoring with dashboards |
| **DebugBear / SpeedCurve / Calibre** | Synthetic + Field | Continuous monitoring with historical trends |

### Lab vs Field Data — Critical Distinction

| Aspect | Lab Data | Field Data |
|---|---|---|
| **Source** | Simulated on controlled device | Real users in production |
| **Tools** | Lighthouse, WebPageTest, DevTools | CrUX, web-vitals, Datadog, Sentry |
| **Reproducible** | Yes — same conditions every run | No — varies per user |
| **Covers real diversity** | No — fixed device/network | Yes — all devices, networks, locations |
| **When available** | Anytime (dev, CI/CD) | Only after deployment with real traffic |
| **Use for debugging** | Yes — drill into specific issues | Harder — aggregated data |
| **Use for Google ranking** | No — Google uses field data only | Yes — CrUX field data drives ranking |

> **Interview insight:** Google uses **field data** (CrUX) for ranking, not lab data. Lighthouse can be 100/100 but if real users on slow phones have poor LCP, your ranking still suffers. That's why RUM matters.

[⬆ Back to Top](#top)

---

## Real User Monitoring (RUM) vs Synthetic Monitoring

### Synthetic Monitoring

Runs **automated tests** against your site from controlled environments — specific devices, networks, and locations — on a scheduled basis or in CI/CD.

```
┌──────────────────────────────────────────────────┐
│              Synthetic Monitoring                  │
│                                                   │
│  Scheduled Agent ──→ Load Page ──→ Collect Metrics│
│  (headless browser)  (controlled    (LCP, FCP,    │
│                       conditions)    waterfall)    │
│                                                   │
│  ✅ Same device, network, location every time      │
│  ✅ Reproducible — great for regression detection  │
│  ❌ Doesn't represent real user diversity          │
└──────────────────────────────────────────────────┘
```

**When it runs:** Scheduled intervals (hourly/daily), CI/CD pipeline (every PR), or on-demand audits.

### Real User Monitoring (RUM)

Collects performance data from **actual users** as they interact with your site. Every page load, every interaction, every error — from real devices, networks, and locations.

```
┌──────────────────────────────────────────────────┐
│           Real User Monitoring (RUM)              │
│                                                   │
│  Real User ──→ Loads Page ──→ Browser APIs        │
│  (varied device,  (real network   collect data    │
│   location,        conditions)    automatically)  │
│   browser)              │                         │
│                         ▼                         │
│               Beacon/Fetch ──→ Analytics Backend  │
│               (send data home)  (aggregate, dash) │
│                                                   │
│  ✅ Represents EVERY real user's experience        │
│  ✅ Catches long-tail issues (slow 3G in India)   │
│  ❌ Noisy data — requires percentile analysis     │
└──────────────────────────────────────────────────┘
```

### Head-to-Head Comparison

| Dimension | Synthetic | RUM |
|---|---|---|
| **Data source** | Simulated agents | Real users in production |
| **Environment** | Controlled (fixed device/network) | Diverse (every user's real conditions) |
| **Consistency** | Highly reproducible | Noisy, varies per user |
| **Coverage** | Only pages/flows you configure | Every page every user visits |
| **Catches regressions** | Proactively (before users see them) | Reactively (after deployment) |
| **Third-party impact** | May miss real behavior | Captures actual third-party impact |
| **Geographic coverage** | Limited test locations | Global (wherever users are) |
| **Cost** | Fixed per test run | Scales with traffic volume |

### When to Use Which

| Scenario | Best Approach | Why |
|---|---|---|
| Pre-deployment regression check | **Synthetic** | Catch issues before users see them |
| Understand p75/p95 user experience | **RUM** | Real data from real users |
| Monitor uptime & availability | **Synthetic** | Runs even when no users are online |
| Debug by region/device/connection | **RUM** | Segment by real user dimensions |
| A/B test performance impact | **RUM** | Measure real-world impact |
| CI/CD quality gate | **Synthetic** | Automated, deterministic pass/fail |
| Catch long-tail issues (5% on slow 3G) | **RUM** | Won't show in synthetic at all |

### The Ideal Strategy — Use Both

```
Development ──→ CI/CD Pipeline ──→ Production

Lighthouse        Lighthouse CI       RUM (all users)
in DevTools       assertions          │
(manual)          (fail build if      ├─ Dashboards (p75/p95)
                   LCP > 2.5s)       ├─ Segment by region, device
                                      ├─ Alerts on regressions
                  Scheduled           └─ Correlate with errors
                  Synthetic
                  (hourly)
```

> **Key insight:** Synthetic catches regressions **early** (before users see them). RUM shows the **real impact** on actual users. Together, they provide complete visibility.

### How RUM Data Collection Works (Conceptual)

The core pattern for collecting real user performance data:

1. **Instrument the page** — add the `web-vitals` library (or equivalent) to every page
2. **Collect metrics** — CWV values fire when available (LCP on navigation, CLS on page hide, INP on page hide)
3. **Buffer events** — batch multiple metrics together to reduce network calls
4. **Send via `sendBeacon`** — `navigator.sendBeacon()` is the recommended method because it **survives page unload** (the user leaving the page). Regular `fetch()` might be cancelled.
5. **Enrich with context** — attach user agent, connection type (`navigator.connection.effectiveType`), device memory, geographic info (from server), and page URL
6. **Aggregate on the backend** — compute p50, p75, p95 per route, per device type, per connection speed, per region
7. **Alert on threshold breaches** — if p75 LCP crosses 2.5s, alert the team

> **Why `sendBeacon`?** When a user clicks a link or closes the tab, the browser cancels pending `fetch()` requests. `sendBeacon()` is specifically designed to survive page transitions — it queues the request and the browser completes it even after navigation.

[⬆ Back to Top](#top)

---

## Error Tracking and Logging

Errors in production are inevitable. The difference between a minor issue and a catastrophe is how quickly you **detect, diagnose, and resolve** them.

### Why Frontend Error Tracking Matters

- **90% of frontend errors** go unreported by users — they just leave
- **JavaScript errors** can completely break an SPA's functionality
- **Silent failures** (API errors swallowed by catch blocks) go unnoticed without tracking
- **Cross-browser issues** — an error in Safari might not reproduce in Chrome
- **Third-party scripts** throw errors outside your control

Without error tracking, you're **flying blind** — relying on user complaints to discover issues that may have been happening for days.

### Types of Frontend Errors

| Error Type | Example | How It's Caught |
|---|---|---|
| **Unhandled exceptions** | `TypeError: Cannot read property 'x' of undefined` | `window.onerror` or `window.addEventListener('error')` |
| **Unhandled promise rejections** | Failed `fetch()` without `.catch()` | `window.addEventListener('unhandledrejection')` |
| **Network errors** | API returns 500, CORS failure, timeout | `fetch` wrapper or Axios interceptor |
| **Resource load failures** | CSS/JS/image 404 | `window.addEventListener('error', fn, true)` (capture phase — resource errors don't bubble) |
| **Framework/render errors** | Error during React render crashes component tree | Error Boundaries (React), `Vue.config.errorHandler` (Vue) |
| **Custom business logic** | User can't checkout, payment fails | Manual error capture call in application logic |

### How Error Tracking Tools Work (Conceptual)

```
Error Occurs in Browser
         │
         ▼
┌──────────────────────┐
│  Error Tracking SDK  │
│  (e.g., Sentry)      │
│                       │
│  1. Capture the error │
│  2. Extract stack     │
│     trace             │
│  3. Collect context:  │
│     • Breadcrumbs     │
│       (user actions   │
│       leading up)     │
│     • Browser/OS/     │
│       device info     │
│     • User identity   │
│     • Current URL     │
│     • Console logs    │
│  4. Apply source maps │
│     (minified →       │
│      readable code)   │
│  5. Send to backend   │
└──────────┬────────────┘
           │
           ▼
┌──────────────────────┐
│  Error Tracking       │
│  Backend              │
│                       │
│  1. Deduplicate —     │
│     group similar     │
│     errors into one   │
│     "issue"           │
│  2. Link to release/  │
│     commit that       │
│     introduced it     │
│  3. Track frequency   │
│     and affected      │
│     user count        │
│  4. Send alerts       │
│     (Slack, PagerDuty)│
└──────────────────────┘
```

### Key Error Tracking Concepts

| Concept | What It Means | Why It Matters |
|---|---|---|
| **Issue grouping** | Thousands of the same `TypeError` get grouped into one "issue" | Without grouping, you'd see 10,000 separate errors instead of one issue affecting 10,000 users |
| **Breadcrumbs** | Automatic trail of user actions before the error (clicks, navigations, API calls, console logs) | Shows you what the user did that led to the error — like a flight recorder |
| **Source maps** | Map minified production code back to original source | Without them, stack traces show `a.js:1:42305` instead of `CartTotal.tsx:42` |
| **Release tracking** | Tag errors with the deployment version (`v1.2.3`) | Know exactly which deploy introduced the regression |
| **Suspect commits** | Tool identifies which Git commit likely caused the error | Go from "we have a bug" to "this commit changed the broken code" in seconds |
| **Session replay** | Video-like recording of the user's session (DOM changes, clicks, scrolls) | See exactly what the user experienced — invaluable for reproducing bugs |
| **Sampling** | Only capture a percentage of events (e.g., 10% of performance traces) | Control costs — you don't need 100% of identical errors |

### Error Tracking Tools Comparison

| Feature | Sentry | LogRocket | Datadog |
|---|---|---|---|
| **Primary focus** | Error tracking & alerting | Session replay & debugging | Full-stack observability |
| **Error grouping** | Excellent | Basic | Good |
| **Session replay** | Yes (built-in) | Best-in-class | Yes |
| **Source maps** | Yes | No | Yes |
| **Release tracking** | Yes (suspect commits) | No | Yes |
| **Performance (RUM)** | Yes | Basic | Yes |
| **Alerting** | Advanced (Slack, PagerDuty, rules) | Basic | Advanced |
| **Best for** | Error detection & alerting | Understanding user experience | Full-stack teams |

> **Practical recommendation:** Use an error tracker (like Sentry) with session replay for error-specific sessions. The combination of "what broke" (stack trace) + "what the user saw" (replay) is the fastest path to diagnosing production issues.

### Alert Tiers — The Traffic Light System

Not all errors are equal. Structure your alerts to avoid alert fatigue:

| Tier | Condition | Alert Channel | Response |
|---|---|---|---|
| **P0 — Critical** | Error rate > 5% of sessions, payment/auth flow broken | PagerDuty → On-call engineer | < 15 min |
| **P1 — High** | New error in critical flow, > 100 events/hour | Slack #incidents | < 1 hour |
| **P2 — Medium** | New error in non-critical flow, < 100 events/hour | Slack #frontend-errors | < 1 day |
| **P3 — Low** | Cosmetic errors, third-party script errors | Weekly digest email | Next sprint |

### Structured Logging Principles

Effective frontend logging follows these principles:

1. **Always include context** — timestamp, URL, user ID, session ID, error severity
2. **Use structured format** — JSON objects, not string concatenation. Structured logs are searchable and aggregatable.
3. **Include breadcrumbs** — log navigation events, API calls, and user actions so you can reconstruct what happened
4. **Sanitize sensitive data** — never log passwords, tokens, credit card numbers, or PII
5. **Use `sendBeacon` for delivery** — survives page unload, won't block the main thread
6. **Set severity levels** — `debug`, `info`, `warn`, `error`, `fatal` — filter by level in production
7. **Sample in production** — you don't need 100% of logs. Sample `info` at 10%, capture all `error` and `fatal`.

[⬆ Back to Top](#top)

---

## Performance Budgets

A **performance budget** is an agreed-upon set of limits on metrics that affect user experience. Like a financial budget, it forces trade-off conversations and prevents the gradual performance degradation that happens in every growing project.

### The Boiling Frog Problem

Without budgets, performance degrades invisibly:

```
Sprint 1:   Bundle = 150 KB   ← "Fast enough"
Sprint 5:   Bundle = 220 KB   ← "Just one more library..."
Sprint 10:  Bundle = 380 KB   ← "When did this get slow?"
Sprint 15:  Bundle = 550 KB   ← "We need a performance sprint"

With a budget (200 KB limit):
Sprint 5:   Bundle = 195 KB   ← ⚠️ Warning: approaching budget
Sprint 6:   PR adds 15 KB    ← ❌ BUILD FAILS — exceeds 200 KB
            → Developer must optimize or justify before merging
```

### Types of Performance Budgets

| Budget Type | What It Limits | Examples |
|---|---|---|
| **Quantity-based** | Number/size of resources | Max 170 KB JS (gzipped), max 50 KB CSS, max 5 third-party scripts |
| **Timing-based** | User-centric timing metrics | LCP ≤ 2.5s, FCP ≤ 1.5s, TTI ≤ 3.5s |
| **Score-based** | Audit tool scores | Lighthouse Performance ≥ 90, Accessibility ≥ 95 |

### Recommended Budget Template

| Category | Metric | Budget | Enforcement |
|---|---|---|---|
| **Loading** | LCP | ≤ 2.5s | Lighthouse CI assertion |
| **Interactivity** | INP | ≤ 200ms | RUM alerting |
| **Stability** | CLS | ≤ 0.1 | Lighthouse CI assertion |
| **JS bundle** | Main bundle (gzipped) | ≤ 170 KB | CI size check tool |
| **CSS** | Total CSS (gzipped) | ≤ 50 KB | CI size check tool |
| **Images** | Hero/LCP image | ≤ 100 KB | CI check or manual review |
| **Fonts** | Total web fonts | ≤ 75 KB, max 2 families | Manual review |
| **Third-party** | Total third-party JS | ≤ 50 KB | Lighthouse audit |
| **Total page** | Total page weight | ≤ 500 KB | Lighthouse CI |
| **Lighthouse** | Performance score | ≥ 90 | Lighthouse CI assertion |

### How to Enforce Performance Budgets

| Level | Tool / Method | What It Does |
|---|---|---|
| **IDE** | "Import Cost" VS Code extension | Shows bundle size impact of every import inline as you code |
| **Build** | Webpack `performance.hints: 'error'` / Vite `chunkSizeWarningLimit` | Fails build if bundles exceed configured size |
| **CI/CD** | `bundlesize` or `size-limit` | Checks gzipped bundle sizes against budgets on every PR |
| **CI/CD** | Lighthouse CI assertions | Runs Lighthouse on every PR, fails if metrics exceed thresholds |
| **Production** | RUM alerting | Alerts when real-user p75 metrics cross budget thresholds |

### How to Set Performance Budgets

1. **Benchmark current state** — run Lighthouse and record your current metrics
2. **Check competitors** — benchmark 2-3 competitors for the same metrics
3. **Set goals 20% better than current** — or match Google's "Good" thresholds as minimum
4. **Get team buy-in** — budgets without enforcement are just wishes
5. **Enforce in CI/CD** — automated enforcement removes debates on every PR
6. **Review quarterly** — adjust budgets as the app evolves

[⬆ Back to Top](#top)

---

## The Three Pillars of Observability

Frontend observability goes beyond performance — it's about having complete visibility into what your application is doing in production.

### The Three Pillars

| Pillar | What It Captures | Frontend Examples |
|---|---|---|
| **Metrics** | Numeric measurements over time | LCP values, error rates, API latency, JS heap size |
| **Logs** | Discrete events with context | Errors, user actions, state changes, API responses |
| **Traces** | End-to-end request flow across services | User click → API call → backend processing → response → UI update |

**Why all three matter — they answer different questions:**

```
Metrics tell you SOMETHING is wrong:
  → "Error rate spiked to 5% at 14:30"

Logs tell you WHAT went wrong:
  → "TypeError: Cannot read property 'price' of null in CartTotal.tsx:42"

Traces tell you WHERE in the chain it broke:
  → "User click → /api/cart (200 OK, but price=null) → CartTotal render → crash"
```

### The Frontend Observability Pipeline

```
┌───────────────────────────────────────────────────────────┐
│                        Browser                             │
│                                                            │
│  Performance APIs ──→ web-vitals ──→ RUM Collector         │
│  Error events     ──→ Sentry SDK ──→ Error Tracker         │
│  User actions     ──→ Session Replay ──→ Replay Store      │
│  Custom events    ──→ Analytics SDK ──→ Event Stream        │
│                                                            │
└────────────────────────┬──────────────────────────────────┘
                         │ (sendBeacon / fetch)
                         ▼
┌───────────────────────────────────────────────────────────┐
│                   Ingestion Layer                          │
│            (API Gateway / Event Queue)                     │
└──────┬──────────────┬──────────────┬──────────────────────┘
       │              │              │
┌──────▼─────┐ ┌──────▼─────┐ ┌─────▼────────┐
│  Metrics   │ │   Logs     │ │   Traces     │
│  Store     │ │   Store    │ │   Store      │
└──────┬─────┘ └──────┬─────┘ └──────┬───────┘
       │              │              │
┌──────▼──────────────▼──────────────▼───────────────────────┐
│              Visualization & Alerting                       │
│                                                             │
│  • Dashboards (p75/p95 by route, device, region)           │
│  • Alerts (Slack/PagerDuty on threshold breaches)          │
│  • SLO tracking (99.5% of loads with LCP < 2.5s)          │
└─────────────────────────────────────────────────────────────┘
```

### Distributed Tracing for Frontend

Distributed tracing connects a user interaction in the browser to all the backend services involved. It works by propagating a **trace ID** across service boundaries:

```
Browser (frontend span)
  └─→ trace-id header ──→ API Gateway (span)
      └─→ Order Service (span)
          └─→ Payment Service (span)
              └─→ Database query (span)

Full trace: Click → Validate → API → Payment → DB → Response → UI Update
```

This is how you answer "why was this checkout call slow for this user?" — you see the entire chain with timing for each step. Tools like Sentry Tracing, Datadog APM, and OpenTelemetry enable this.

[⬆ Back to Top](#top)

---

## Tradeoffs Gotchas and Common Pitfalls

### Common Mistakes

| Pitfall | What Goes Wrong | Prevention |
|---|---|---|
| **Optimizing without measuring** | Guessing where slowness is → wasting effort on wrong things | Always profile first — measure, then optimize |
| **Measuring median instead of p75** | Median hides long-tail issues that affect 25%+ of users | Google uses p75. Always track p75 and p95. |
| **Lab-only testing** | Lighthouse is 100 but real users on slow phones have poor experience | Combine Lighthouse (lab) with RUM (field) |
| **No performance budget** | Performance degrades gradually with every sprint | Set budgets, enforce in CI/CD |
| **Alert fatigue** | Too many alerts → team ignores all of them | Tier alerts (P0-P3), filter known noise |
| **Not uploading source maps** | Production stack traces are unreadable minified code | Upload source maps to your error tracker on every deploy |
| **Over-sampling in RUM** | Sending every metric from every user = high cost, no extra value | Sample performance traces at 10-20%, capture all errors at 100% |
| **Ignoring third-party scripts** | Your code is fast but GTM + ads + chat widgets add 500ms | Audit third-party impact regularly, use Lighthouse's third-party audit |
| **Testing on developer machines only** | Fast MacBook Pro ≠ budget Android phone on 3G | Use CPU/network throttling in DevTools, test on real devices |
| **Chasing Lighthouse score** | 100/100 Lighthouse doesn't mean real users are happy | Lighthouse is a starting point. RUM is the truth. |

### Platform-Specific Considerations

| Platform | Consideration |
|---|---|
| **Mobile (general)** | Always test with CPU throttling (4x–6x slowdown). Median phones are much slower than dev machines. |
| **iOS Safari** | No Background Sync, no Periodic Sync, limited Service Worker support. Test on real iOS devices — simulators don't accurately reflect performance. |
| **Low-end Android** | 2-4 GB RAM, slow CPUs. These are your p75/p95 users. Budget Android phones represent the majority of global web traffic. |
| **Slow networks** | Always test on "Fast 3G" and "Slow 3G" throttling. Most of the world isn't on fast broadband. |
| **High-DPI displays** | Serving 2x/3x images without responsive sizing burns bandwidth and hurts LCP. |

[⬆ Back to Top](#top)

---

## Interview Questions and Answers

### Q1: What are Core Web Vitals and why do they matter?

**A:** Core Web Vitals are three Google-defined metrics that measure real-world user experience:
- **LCP (Largest Contentful Paint)** — loading speed. Measures when the largest content element is rendered. Target: ≤ 2.5s.
- **INP (Interaction to Next Paint)** — responsiveness. Measures how quickly the page responds to user interactions. Target: ≤ 200ms.
- **CLS (Cumulative Layout Shift)** — visual stability. Measures unexpected layout shifts. Target: ≤ 0.1.

They matter because: (1) Google uses them as **SEO ranking signals** since June 2021, (2) they're measured at the **75th percentile** of real user data, and (3) they directly correlate with user engagement metrics like bounce rate and conversion.

---

### Q2: Why did INP replace FID?

**A:** FID (First Input Delay) only measured the **delay** of the **first** interaction, and it only captured input delay (not processing or presentation time). This meant:
- It missed slow interactions later in the page lifecycle (e.g., a button that becomes slow after data loads)
- A page could have great FID but terrible responsiveness on the 10th interaction
- It didn't account for how long event handlers took to execute

INP measures **all** interactions throughout the entire page lifecycle and captures the full latency (input delay + processing time + presentation delay). It reports the worst interaction, giving a much more comprehensive picture of responsiveness.

---

### Q3: How would you improve LCP on a slow-loading page?

**A:** I'd follow a systematic approach:

1. **Check TTFB first** — if the server takes 2s to respond, LCP can never be under 2s. Fix with CDN, caching, backend optimization.
2. **Identify the LCP element** — use DevTools Performance panel or `web-vitals` attribution to find which element is the LCP candidate.
3. **Preload the LCP resource** — if it's an image, add `<link rel="preload">` and `fetchpriority="high"`. If it's text, ensure fonts are preloaded.
4. **Remove render-blocking resources** — inline critical CSS, defer non-critical CSS/JS.
5. **Never lazy-load the LCP element** — `loading="lazy"` on the hero image is a common mistake.
6. **Optimize the resource itself** — compress images to WebP/AVIF, use responsive `srcset`, proper sizing.
7. **Consider SSR/SSG** — if the page is client-side rendered, the LCP has to wait for JS → fetch → render. Server rendering eliminates that chain.

---

### Q4: How would you improve INP?

**A:** INP issues come from three phases — I'd diagnose which phase is the problem:

- **Input Delay is high** → The main thread was busy when the user interacted. Break up Long Tasks, defer non-critical work, reduce third-party script impact.
- **Processing Time is high** → Event handlers take too long. Optimize the handler, yield to the browser with `scheduler.yield()` between chunks, move heavy computation to a Web Worker.
- **Presentation Delay is high** → The visual update after the handler is expensive. Reduce DOM complexity, avoid layout thrashing, simplify CSS selectors.

I'd use the `web-vitals` attribution build to identify which interaction is the slowest and which phase dominates, then profile that specific interaction in DevTools Performance panel.

---

### Q5: How do you prevent CLS?

**A:**
1. **Always set dimensions on images/videos** — use `width`/`height` attributes or CSS `aspect-ratio` so the browser reserves space before loading
2. **Reserve space for dynamic content** — ads, embeds, cookie banners should have `min-height` or aspect-ratio containers
3. **Use `font-display: optional`** for web fonts — prevents font swap layout shifts entirely (or use `size-adjust` to match fallback font metrics)
4. **Use `transform` for animations** — never animate `top`, `left`, `width`, `height` (layout triggers). `transform` and `opacity` skip layout and paint entirely.
5. **SSR/SSG** — avoid the client-side rendering pattern where skeleton → data → layout shift

---

### Q6: What's the difference between RUM and Synthetic monitoring?

**A:** **Synthetic monitoring** runs automated tests from controlled environments (fixed device, network, location). It's reproducible, proactive (catches issues before users see them), and great for CI/CD quality gates. But it doesn't represent real user diversity.

**RUM (Real User Monitoring)** collects data from actual users in production — real devices, real networks, real locations. It catches long-tail issues (the 5% of users on slow 3G in India), captures third-party script impact, and shows actual user experience. But it's reactive (data only after deployment) and noisy (requires percentile analysis).

**Best practice:** Use both together. Synthetic in CI/CD to **prevent** regressions. RUM in production to **validate** real impact. Google uses field data (RUM/CrUX) for ranking, not lab data.

---

### Q7: What is a performance budget and how do you enforce it?

**A:** A performance budget is an agreed-upon threshold for metrics that a page must not exceed — like a financial budget for performance. Examples: max 170 KB JS (gzipped), LCP ≤ 2.5s, Lighthouse Performance ≥ 90.

Without budgets, performance degrades gradually as features are added — the "boiling frog" problem.

**Enforcement happens at multiple levels:**
- **IDE level** — "Import Cost" extension shows bundle size impact while coding
- **Build level** — Webpack/Vite config fails the build on oversized bundles
- **CI/CD level** — `bundlesize` or `size-limit` checks bundle sizes on every PR; Lighthouse CI asserts metric thresholds
- **Production level** — RUM dashboards alert when p75 metrics cross budgets

The key is **automated enforcement** — budgets without CI/CD checks are just suggestions that get ignored.

---

### Q8: How would you investigate a report that "the app is slow"?

**A:** I'd follow the investigation funnel:

1. **Quantify "slow"** — which metric is poor? Check RUM dashboards for LCP/INP/CLS p75. Is it a loading problem, interaction problem, or visual stability problem?
2. **Segment the data** — is it slow for all users or specific segments? Filter by device type, connection speed, geographic region, browser, OS. Often "slow" affects budget phones on 3G, not developer MacBooks.
3. **Reproduce locally** — try to reproduce with DevTools throttling (CPU 4x slowdown + Fast 3G). Record a Performance trace.
4. **Analyze the trace** — look for Long Tasks (red flags), heavy Layout/Style recalculations (purple), large network requests. Use Bottom-Up view to find expensive functions.
5. **Check third-parties** — are third-party scripts (analytics, ads, chat widgets) contributing to main thread blocking?
6. **Prioritize fixes** — use the impact × effort matrix. Quick wins first (preload LCP, add image dimensions, defer scripts), then bigger architectural changes (code splitting, SSR, Web Workers).
7. **Validate the fix** — deploy, monitor RUM for p75 improvement, confirm with users.

---

### Q9: What are the three pillars of observability and how do they apply to frontend?

**A:**
1. **Metrics** — numeric measurements over time. Frontend: Core Web Vitals (LCP, INP, CLS), error rates, API latency, JS heap size. Used for dashboards and alerting.
2. **Logs** — discrete events with context. Frontend: error stack traces with breadcrumbs, user action logs, API request/response details. Used for debugging specific issues.
3. **Traces** — end-to-end request flows across services. Frontend: tracing from user click → API call → backend processing → database → response → UI update, with timing at each step. Used for diagnosing latency in distributed systems.

**Metrics** tell you **something** is wrong. **Logs** tell you **what** went wrong. **Traces** tell you **where** in the chain it broke.

In practice: `web-vitals` + analytics for metrics, Sentry for logs + errors, Sentry Tracing or OpenTelemetry for traces.

---

### Q10: How do you prevent cache from growing unbounded in a Service Worker?

**A:** Three approaches:
1. **Entry limit** — keep max N items per cache (e.g., 100 images). Evict oldest on overflow (FIFO/LRU).
2. **TTL (Time-to-Live)** — add a timestamp when caching. On read, check age and delete expired entries.
3. **Version cleanup** — on Service Worker activate, delete all caches not in the current version's allow-list.

---

### Q11: What's the difference between lab data and field data?

**A:** **Lab data** (Lighthouse, WebPageTest) runs on a simulated device with fixed conditions. It's reproducible and great for debugging, but doesn't represent real user diversity. **Field data** (CrUX, RUM) is collected from actual users in production — real devices, networks, and locations. It's noisy but represents the truth.

Critical distinction: **Google uses field data (CrUX) for SEO ranking**, not lab data. You can score 100 on Lighthouse but still fail Google's assessment if real users on slow phones have poor metrics.

---

### Q12: How would you set up error tracking for a production React app?

**A:** The core components:
1. **Global error handlers** — `window.onerror` for uncaught exceptions, `unhandledrejection` listener for promise rejections, capture-phase error listener for resource failures
2. **React Error Boundaries** — wrap critical routes/features with Error Boundaries to catch render errors and show fallback UI
3. **Error tracking service** (e.g., Sentry) — captures errors with full context: stack trace, breadcrumbs (user actions leading to error), user info, browser/OS, release version
4. **Source maps** — upload on every deploy so stack traces show original source, not minified code
5. **Alert tiers** — P0 (critical flows broken → PagerDuty), P1 (new error in important flow → Slack), P2-P3 (lower severity → digest)
6. **Noise filtering** — ignore known harmless errors (`ResizeObserver loop limit exceeded`, browser extension errors)
7. **Session replay on errors** — automatically record sessions where errors occur for easy reproduction

---

### Q13: What tools would you use for a complete frontend performance monitoring strategy?

**A:**

| Stage | Tool | Purpose |
|---|---|---|
| **Development** | Chrome DevTools (Performance, Network, Lighthouse panels) | Profile and debug locally |
| **IDE** | Import Cost extension | See bundle size impact of imports |
| **Build** | Bundle analyzer (webpack-bundle-analyzer) | Visualize what's in the bundle |
| **CI/CD** | Lighthouse CI + `size-limit`/`bundlesize` | Automated regression prevention |
| **Production — RUM** | `web-vitals` library + analytics backend | Real user Core Web Vitals |
| **Production — Errors** | Sentry (with session replay) | Error tracking, alerting, debugging |
| **Production — Field data** | CrUX / PageSpeed Insights | Google's official field data |
| **Ongoing** | Scheduled synthetic tests (WebPageTest, DebugBear) | Baseline monitoring, competitor benchmarking |

---

### Q14: A page has good lab scores but poor field metrics. Why?

**A:** This is a very common scenario. Possible reasons:
1. **User diversity** — lab tests on fast machines; real users on budget phones with slow CPUs and 3G connections
2. **Third-party scripts** — lab might not fully load all third-party scripts (analytics, ads, chat widgets) that compete for main thread time in production
3. **Geographic distance** — lab test from a server close to CDN; real users far from CDN edge nodes experience high TTFB
4. **Caching state** — lab tests with warm cache; many real users are first-time visitors with cold cache
5. **Dynamic content** — lab tests might get different content (fewer ads, no personalization, no A/B test variants)
6. **Long-tail interactions** — lab tests the initial load; real users interact for minutes, hitting slow INP interactions later
7. **Browser/OS diversity** — lab uses latest Chrome; real users include older Safari, Firefox, webviews with different JS engines

**Fix:** Segment RUM data by device type, connection speed, and geography to find which user cohorts have poor experience, then optimize for those conditions.

---

*For related topics, see companion articles: **Critical-Rendering-Path.md**, **css-js-ui-optimization.md**, **network-optimization.md**, **image-optimization.md**, and **Web_Worker_SharedWorker_and_Service_Worker.md**.*

[⬆ Back to Top](#top)

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design

[⬆ Back to Top](#top)
