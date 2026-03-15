# Micro Frontends, Module Federation and Cross Communication

A guide to **Monolithic vs Micro Frontend** architectures, composition strategies, cross-MFE communication patterns, and **Webpack Module Federation** — focused on when to use what and why.

---

<a id="top"></a>

## Table of Contents

- [Monolithic Frontend Architecture](#monolithic-frontend-architecture)
- [What Are Micro Frontends](#what-are-micro-frontends)
- [Monolith vs Micro Frontend Tradeoffs](#monolith-vs-micro-frontend-tradeoffs)
- [MFE Integration Composition Approaches](#mfe-integration-composition-approaches)
- [Hosting Multiple MFEs Under One UI](#hosting-multiple-mfes-under-one-ui)
- [Cross Communication Between MFEs](#cross-communication-between-mfes)
- [Module Federation Deep Dive](#module-federation-deep-dive)
- [Module Federation 2.0](#module-federation-20)
- [Shared Dependencies and Versioning](#shared-dependencies-and-versioning)
- [Routing in Micro Frontends](#routing-in-micro-frontends)
- [Deployment and CI CD](#deployment-and-ci-cd)
- [Real World Examples](#real-world-examples)
- [Decision Flowchart](#decision-flowchart)
- [Key Takeaways](#key-takeaways)
[⬆ Back to Top](#top)

---

## Monolithic Frontend Architecture

A **monolithic frontend** is a single, unified codebase where every page, feature, and route lives together. One team (or multiple teams in the same repo) builds, tests, and deploys the entire application as a single unit.

```
┌──────────────────────────────────────────────────┐
│                Monolithic SPA                     │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │  Auth     │  │  Product │  │  Cart    │        │
│  │  Module   │  │  Catalog │  │  Module  │        │
│  └──────────┘  └──────────┘  └──────────┘        │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │  Search   │  │  Profile │  │  Orders  │        │
│  │  Module   │  │  Module  │  │  Module  │        │
│  └──────────┘  └──────────┘  └──────────┘        │
│                                                   │
│     Single Build  →  Single Bundle  →  Deploy     │
└──────────────────────────────────────────────────┘
```

| Aspect | Detail |
|---|---|
| **Codebase** | Single repository, single build pipeline |
| **Deployment** | All-or-nothing deploy for every change |
| **Team Coupling** | High — every team touches the same code |
| **Bundle Size** | Grows linearly with features (mitigated by code-splitting) |
| **Consistency** | Natural — shared design system, shared state |

### Build & Deploy Cycle

```
Developer A (Auth)  ──┐
Developer B (Cart)  ──┼──>  Single Repo  ──>  CI Pipeline
Developer C (Search)──┘    (merge to main)       │
                                                  ▼
                                        Lint + Test (ALL code)
                                                  │
                                                  ▼
                                         Build (Entire App)
                                                  │
                                                  ▼
                                      Deploy (All or Nothing)
```

Developer A's one-line auth fix must wait for Developer B's half-finished cart feature to be merged or reverted.

### When Monolith Works Well

- Small-to-medium app with **1–3 teams**
- Features are **tightly coupled** (e.g., dashboard where every widget shares state)
- Your org **doesn't need independent deployment** cadences
- Early-stage products where **speed of iteration** matters most

Monoliths are not bad — they are the correct starting point. The problems only emerge at **organizational scale**, not application scale. A well-structured monolith with lazy routes and code-splitting can serve a product for years. Most of the benefits attributed to MFEs can be partially achieved within a well-structured monolith: lazy loading, monorepo tooling (Nx, Turborepo), and CODEOWNERS files. The one thing a monolith **cannot** give you is independent deployment.

**Rule of thumb:** If your team can ship a feature from code review to production in under a day, your monolith is not the bottleneck.

### Pain Points at Scale

```
App grows  →  Build times spike (5–15 min+)
           →  Merge conflicts multiply
           →  One bug blocks entire deploy
           →  Teams step on each other's code
           →  Testing surface explodes
```

### The Coupling Cascade

Nothing prevents cross-module coupling in a monolith. Any file can import any other file. Developers under deadline pressure take shortcuts — import a utility from another module, share a React context, add a field to a shared Redux store. Each shortcut is harmless individually, but collectively they create a web of implicit dependencies that makes independent changes impossible.

```
Year 1:  Clean modules, fast builds, small team
Year 2:  "Just import that util from the other module"
Year 3:  "We need a shared context for user data"
Year 4:  "Changing the cart broke the search page somehow"
Year 5:  Fear-driven development, massive test suites, 3-week releases
```

Micro-frontends solve this by creating **hard boundaries** — separate repos, separate builds, separate deploys. You physically cannot `import` from another MFE's source code, forcing teams to communicate through explicit contracts.

[⬆ Back to Top](#top)

---

## What Are Micro Frontends

**Micro-frontends** extend the microservices idea to the frontend: split a large monolithic UI into **independently developed, tested, deployed, and hosted** frontend applications that are **composed together** in the browser to feel like one cohesive product.

*"An architectural style where independently deliverable frontend applications are composed into a greater whole."* — Martin Fowler

### Vertical Slices, Not Horizontal Layers

The most common mistake is thinking of MFEs as horizontal layers (UI team, API team, data team). Instead, MFEs are **vertical slices** — each team owns an entire feature end-to-end.

```
  ❌ WRONG: Horizontal Layers         ✅ CORRECT: Vertical Slices

  ┌──────────────────────┐         ┌──────┐  ┌──────┐  ┌──────┐
  │   UI Team            │         │Search│  │Produc│  │ Cart │
  ├──────────────────────┤         │ Team │  │ Team │  │ Team │
  │   API Team           │         │      │  │      │  │      │
  ├──────────────────────┤         │  UI  │  │  UI  │  │  UI  │
  │   Data Team          │         │  API │  │  API │  │  API │
  └──────────────────────┘         │  DB  │  │  DB  │  │  DB  │
                                   └──────┘  └──────┘  └──────┘
  Teams organized by layer          Teams own full features
  → lots of cross-team work         → autonomous delivery
```

Each MFE owns its own UI, state, routes, and ideally communicates with its own backend microservice. The Search team owns the search UI (React app), search API (Node/Go service), and search database (Elasticsearch). They can ship without coordinating with anyone.

### Core Principles

| Principle | Meaning |
|---|---|
| **Team Autonomy** | Each team owns a vertical slice end-to-end. They can make local decisions without cross-team approval. |
| **Technology Agnostic** | Teams can use different frameworks (though most orgs standardize). Mainly useful for **incremental migration**. |
| **Independent Deployment** | Ship your MFE without coordinating with other teams. Each MFE has its own CI/CD. This is the #1 reason companies adopt MFEs. |
| **Isolation** | One MFE's crash shouldn't take down others. `ErrorBoundary` around each MFE ensures fault containment. |
| **No Shared State** | Prefer explicit contracts (events, APIs, props) over shared global state. Shared state re-introduces the coupling MFEs are meant to solve. |

**Technology Agnostic** sounds appealing but is almost always the wrong default. Running React + Vue means double framework bundles, two tooling sets, and difficulty sharing a design system. The real value is for **incremental migration** — moving from Angular to React MFE by MFE.

### Architecture Overview

```
                    ┌─────────────────────────────┐
                    │       App Shell / Host       │
                    │  (Routing, Layout, Auth)     │
                    └─────────┬───────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                    │
 ┌────────▼──────┐  ┌────────▼──────┐  ┌─────────▼─────┐
 │  MFE: Search   │  │  MFE: Product │  │  MFE: Cart    │
 │  (Team Alpha)  │  │  (Team Beta)  │  │  (Team Gamma) │
 │  React 18      │  │  React 18     │  │  Vue 3        │
 └────────────────┘  └───────────────┘  └───────────────┘
          │                   │                    │
          ▼                   ▼                    ▼
 Independent CI/CD   Independent CI/CD   Independent CI/CD
```

[⬆ Back to Top](#top)

---

## Monolith vs Micro Frontend Tradeoffs

| Dimension | Monolith | Micro-Frontend |
|---|---|---|
| **Team scalability** | Hard beyond 5–8 devs | Scales to dozens of teams |
| **Deployment** | All-or-nothing | Per-MFE independent |
| **Build time** | Grows with app size (10–15 min for 500K LoC) | Per-MFE (stays small, ~30 sec) |
| **Technology freedom** | Single stack | Mix-and-match (with caution) |
| **UX Consistency** | Natural | Requires shared design system |
| **Performance** | One optimized bundle | Risk of duplicate deps |
| **Complexity** | Low | High (orchestration, contracts, versioning) |
| **Testing** | Single E2E suite | Per-MFE + integration layer |
| **Fault isolation** | One bug can break all | Bug scoped to one MFE |
| **Shared state** | Easy (Redux, context) | Hard (events, props, URL) |

### The Inflection Point

```
          Pain / Overhead
          │
 MFE      │        /   Monolith Pain
 Overhead │       /
 ------   │      /
          │     /
 ─────────┼────/──────── ← Crossover point (4-6 teams, 100K+ LoC)
          │   /
          │  /
 MFE Cost │ ──────────── MFE Overhead (relatively flat)
          │
          └────────────────────────>
          1 team    4-6 teams    10+ teams
```

In a monolith, coordination cost scales **O(n²)** with team count (every team's changes can conflict with every other's). MFEs reduce this to **O(n)** because teams only coordinate at well-defined boundaries.

### When to Choose MFEs

✅ Large org with **5+ autonomous teams**
✅ Teams need **different release cadences**
✅ **Clear domain boundaries** (e.g., search, product, cart, checkout)
✅ Need to **incrementally migrate** a legacy monolith

### When NOT to Choose MFEs

❌ Small team (< 5 devs)
❌ Features are **heavily coupled** with lots of shared state
❌ Org doesn't have **DevOps maturity** for multiple pipelines
❌ Building a **prototype or MVP**

### Strangler Fig Migration Pattern

Most real-world MFE adoption is **gradually migrating a monolith**, not greenfield. The Strangler Fig pattern wraps the old system and replaces it piece by piece:

```
Phase 1:  Monolith serves everything
          ┌────────────────────────────────┐
          │  MONOLITH (Search|Product|Cart) │
          └────────────────────────────────┘

Phase 2:  Add App Shell, extract first MFE
          ┌────────────────────────────────┐
          │           APP SHELL            │
          │  /search  → Search MFE (new)   │
          │  /* else  → Monolith (legacy)  │
          └────────────────────────────────┘

Phase 3:  Extract more MFEs over months
          ┌────────────────────────────────┐
          │           APP SHELL            │
          │  /search  → Search MFE         │
          │  /product → Product MFE        │
          │  /cart    → Cart MFE           │
          │  /* else  → Monolith (shrink)  │
          └────────────────────────────────┘

Phase 4:  Monolith fully decomposed
          ┌────────────────────────────────┐
          │           APP SHELL            │
          │  All routes → dedicated MFEs   │
          │  Monolith is gone ✓            │
          └────────────────────────────────┘
```

At every phase the system is fully functional. If a new MFE has bugs, route it back to the monolith with a config change. Each phase delivers **immediate measurable value** — unlike "big bang" rewrites that deliver zero value until 100% complete.

The hardest part is **shared authentication and layout**. The monolith and new MFEs must share the same login session (via cookies or `localStorage` tokens) and maintain a consistent header/footer. Build the App Shell and auth integration first.

[⬆ Back to Top](#top)

---

## MFE Integration Composition Approaches

### Composition Spectrum

```
Build-Time                                             Runtime
(tight coupling,                                (loose coupling,
 best performance)                               more flexibility)
     │                                                  │
     ▼                                                  ▼
┌─────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐  ┌─────────┐
│   npm   │  │  Module  │  │ JS       │  │  Web    │  │ iframes │
│ packages│  │  Federat-│  │ Dynamic  │  │ Compo-  │  │         │
│         │  │  ion     │  │ Remotes  │  │ nents   │  │         │
└─────────┘  └──────────┘  └──────────┘  └─────────┘  └─────────┘

  Less autonomy                                More autonomy
  Better perf                                  More isolation
  Simpler setup                                More complexity
```

### 4.1 Build Time Integration (npm Packages)

MFEs are published as **npm packages** and the host installs them as dependencies. The host imports and bundles them at compile time.

| Pros | Cons |
|---|---|
| Simple to set up | Loses independent deployment (host must rebuild) |
| Strong typing across MFEs | Version coupling — lock-step releases |
| Single optimized bundle | Not truly "micro" — just well-organized monolith |

**Why this isn't true MFE:** Every time the Search team ships a new version, the Host must bump the version, rebuild, run CI/CD, and deploy. The Host becomes a bottleneck. In a 10-team org, the Host could rebuild 5 times a day just to pick up others' changes. Works great for **shared component libraries** where you want explicit opt-in to new versions, not for independent deployment.

### 4.2 Runtime Integration via iframes

Each MFE is loaded inside an `<iframe>`. Each iframe gets a separate browsing context with its own DOM, CSS engine, JS heap, and event loop.

| Pros | Cons |
|---|---|
| **Complete isolation** (CSS, JS, DOM) | Poor UX (no shared scrolling, awkward resizing) |
| Technology agnostic | Cross-iframe communication is clunky (`postMessage`) |
| Security sandbox | Performance overhead (~4x memory for 3 iframes) |
| | Cannot share dependencies (duplicate React) |
| | SEO-unfriendly |

**When to actually use iframes:**
1. **Third-party widget embedding** (Intercom, payment forms) — you don't control the code
2. **Legacy migration** — wrap the old jQuery monolith in an iframe while building new MFEs
3. **Security-critical sections** — iframe the payment page so XSS in the main app can't steal card data

### 4.3 Runtime Integration via JavaScript (Dynamic Remotes)

The host loads each MFE's JS bundle at runtime. Each MFE exports a `mount(container)` function. This is the pattern **Single-SPA** popularized.

**How it works:**

```
Host App Shell
     │
     ├── Fetch manifest.json → { search: "cdn/.../search.js" }
     │
     ├── <script src="search.js">
     │       └── window.searchMFE.mount(#search-root)
     │
     └── <script src="cart.js">
             └── window.cartMFE.mount(#cart-root)
```

**Lifecycle flow:**
1. Each MFE exports `mount(container)` and optionally `unmount()`, `bootstrap()`
2. The Host decides when to call them based on routes
3. Each MFE is framework-agnostic — mount can call `ReactDOM.render()`, `createApp().mount()` (Vue), or vanilla DOM
4. The cleanup function is critical — without proper unmount, you get memory leaks

```
User navigates to /search:
  Host → fetch search.js → call mount(#search-root) → Search MFE renders

User navigates to /cart:
  Host → call search cleanup() → fetch cart.js → call mount(#cart-root)
```

| Pros | Cons |
|---|---|
| True independent deployment | Manual dependency management |
| Each MFE can use different tech | Global namespace pollution risk |
| Runtime flexibility | No built-in shared dependency dedup |

### 4.4 Runtime Integration via Web Components

Each MFE wraps itself as a **Custom Element** using the browser-native Web Components API. Shadow DOM provides style isolation without iframe overhead.

| Pros | Cons |
|---|---|
| Framework-agnostic (browser standard) | Shadow DOM CSS isolation can be tricky |
| Encapsulated via Shadow DOM | Event bubbling across shadow boundaries needs `composed: true` |
| Native browser API | SSR support is limited |
| | Passing complex data via attributes is awkward (string-only) |

Web Components act as a **framework-agnostic wrapper**: your React MFE renders inside a custom element's Shadow DOM, and from the host's perspective it's just a `<mfe-search>` tag. The practical challenge is passing complex data — HTML attributes are strings, so you either serialize to JSON or use JS properties.

### 4.5 Runtime Integration via Module Federation (Webpack 5)

The **most popular modern approach**. See [Section 7](#7-module-federation-deep-dive) for the deep dive.

### 4.6 Server-Side Composition

MFEs are composed on the **server** before HTML reaches the browser. A composition layer (edge function, CDN, Node.js middleware) fetches HTML fragments from each MFE's SSR service and stitches them into one page.

```
Browser: GET /product/laptop-123
              │
              ▼
    Composition Server (Edge)
         │         │         │
         ▼         ▼         ▼
    Header SSR  Product SSR  Footer SSR
    (50ms)      (120ms)      (30ms)
         │         │         │
         └─────────┬─────────┘
                   ▼
     Stitched HTML (130ms total) → Browser
```

**Technologies:** Podium, Piral, Tailor (Zalando), ESI (Edge Side Includes)

| Pros | Cons |
|---|---|
| Fast FCP (server-rendered) | More complex infrastructure |
| SEO-friendly | Hydration coordination is hard |
| No client-side orchestration | Technology mixing is harder |

The critical challenge is **hydration coordination**: each fragment was rendered by a different framework instance. Each MFE must hydrate only its own fragment with separate hydration scripts.

### Comparison Matrix

| Approach | Indep. Deploy | Isolation | Shared Deps | Performance | Complexity | SEO |
|---|---|---|---|---|---|---|
| **npm packages** | ❌ | ❌ | ✅ Natural | ✅ Best | Low | ✅ |
| **iframes** | ✅ | ✅ Full | ❌ Duplicated | ❌ Heavy | Low | ❌ |
| **JS Dynamic Remotes** | ✅ | ⚠️ Partial | ⚠️ Manual | ⚠️ Depends | Medium | ⚠️ |
| **Web Components** | ✅ | ✅ Shadow DOM | ⚠️ Manual | ⚠️ Depends | Medium | ⚠️ |
| **Module Federation** | ✅ | ⚠️ Partial | ✅ Built-in | ✅ Good | Medium-High | ⚠️ |
| **Server-side** | ✅ | ✅ Process | ⚠️ Varies | ✅ Good FCP | High | ✅ |

### Three Underlying Tensions

Don't memorize the table. Understand the three tensions that drive every cell:

1. **Coupling vs Autonomy:** Build-time approaches (npm) give tight integration and best performance but sacrifice independent deployment. Runtime approaches (iframes, Module Federation) give full autonomy but add network requests and potential failure points.

2. **Isolation vs Shared Resources:** iframes give perfect isolation but zero resource sharing. Module Federation gives shared dependencies but imperfect isolation (MFEs share the same DOM and can accidentally interfere with CSS).

3. **Performance vs Flexibility:** Best performance comes from a single optimized bundle. Most flexibility comes from runtime code loading. No approach maximizes both simultaneously.

**Practical recommendation:** Module Federation for Webpack/Vite + React teams. Server-side composition when SEO is a hard requirement. iframes only for truly untrusted third-party content.

[⬆ Back to Top](#top)

---

## Hosting Multiple MFEs Under One UI

The key challenge: how do you make 3–10 independently deployed apps feel like **one product**?

### The App Shell Pattern

The **App Shell** (Host/Container) is a lightweight application responsible for:

1. **Routing** — which MFE to load for a given URL
2. **Layout** — shared chrome (header, sidebar, footer)
3. **Authentication** — auth tokens/context for all MFEs
4. **Loading MFEs** — fetching and mounting the correct MFE
5. **Shared Services** — analytics, error tracking, feature flags

The shell should be **thin and stable** — it deploys rarely while MFEs deploy frequently. Think of it like an OS kernel: small, rarely updated. If the shell had significant business logic, it would become a bottleneck.

```
┌────────────────────────────────────────────────────────┐
│                    App Shell (Host)                    │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Header / Navigation Bar                         │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  ┌──────────┐  ┌─────────────────────────────────┐     │
│  │          │  │       MFE Content Area          │     │
│  │ Sidebar  │  │                                 │     │
│  │ (shared) │  │  /search  → Search MFE          │     │
│  │          │  │  /product → Product MFE         │     │
│  │          │  │  /cart    → Cart MFE            │     │
│  │          │  │                                 │     │
│  └──────────┘  └─────────────────────────────────┘     │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Footer                                          │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

### Hosting Strategies

| Strategy | How It Works | When to Use |
|---|---|---|
| **Separate origins** | Each MFE on its own subdomain (`search.myapp.com`) | Max autonomy, separate cloud accounts. Requires CORS config. |
| **Path-based CDN** | Same CDN, different prefixes (`cdn.myapp.com/search/`) | Most common. Same-origin = no CORS headaches. |
| **Monorepo deploy** | One repo (Nx/Turborepo), each MFE builds/deploys independently | Orgs starting their MFE journey. Shared tooling + independent deploys. |
| **Container registry** | Docker containers behind reverse proxy (K8s + Nginx) | Orgs with existing Kubernetes infra. |

### Discovery Service / Manifest

In production, the host doesn't hardcode MFE URLs. It fetches a **manifest** at runtime — the same pattern as **service discovery** in backend microservices (Consul, etcd, K8s DNS).

```
Hardcoded (bad):
  Host config → "searchApp@cdn/search/v2.3.1/remoteEntry.js"
  Search deploys v2.4.0 → Host still points to v2.3.1!
  Must rebuild + redeploy host ❌

Manifest (good):
  Host fetches "/mfe-manifest.json" → { search: "cdn/search/v2.4.0/..." }
  Search deploys v2.4.0 → Updates manifest only
  Host automatically picks up new version ✓
```

The manifest enables **canary deployments** (5% of users see new version), **instant rollbacks** (update manifest to previous version — no rebuild), **A/B testing** at MFE level, and an **emergency kill switch** (revert within CDN cache TTL of ~60 seconds).

**Cache strategy:** Manifest served with a **short TTL (60 seconds)** so updates propagate quickly. MFE bundles served with **long TTLs (1 year)** using content-hashed filenames since they're immutable.

[⬆ Back to Top](#top)

---

## Cross Communication Between MFEs

One of the hardest MFE problems. MFEs must be independent (no shared imports), yet the UX demands they feel connected (clicking "Add to Cart" in Product MFE should update the cart badge in Cart MFE).

### Communication Rules

```
✘ NEVER:  import { addToCart } from '../cart-mfe/utils';
          (creates build-time dependency)

✘ NEVER:  window.cartMFE.state.items.push(newItem);
          (reaches into another MFE's internal state)

✓ GOOD:   eventBus.emit('cart:add-item', { productId, qty });
          (loose coupling via agreed contract)

✓ GOOD:   <CartMFE items={cartItems} onCheckout={handleCheckout} />
          (props from host — explicit, typed, traceable)

✓ GOOD:   URL: /product/123?addedToCart=true
          (natural, shareable, bookmarkable state)
```

### 6.1 Custom Events (DOM Events)

Uses the browser's native `CustomEvent` API. One MFE dispatches on `document`, another listens.

| Pros | Cons |
|---|---|
| Zero dependencies, native API | No type safety |
| Framework agnostic | Easy to create spaghetti event chains |
| Works across Shadow DOM | Debugging is harder |

**When to use:** 2–3 MFEs with a handful of fire-and-forget notifications like "user logged in" or "item added to cart".

**When NOT to use:** 10+ event types across 5+ MFEs — you lose track of who's emitting what.

### 6.2 Pub/Sub Event Bus (Typed)

A lightweight event bus shared across MFEs via a singleton on `window`. The key improvement over raw CustomEvents is a **typed event contract** — a single interface that defines all cross-MFE events and their payload shapes.

The `MFEEvents` interface acts as a shared contract living in a shared package (`@org/mfe-contracts`). When one team changes a payload shape, TypeScript flags all consumers at build time instead of silently breaking in production.

```
Without types:  Product emits { productId: '123', qty: 1 }
                Cart expects { quantity: 1 }  ← silent bug in prod!

With types:     MFEEvents['cart:add-item'] = { productId: string; quantity: number }
                Product tries to emit { qty: 1 } → TypeScript error ✅
```

| Pros | Cons |
|---|---|
| Type-safe with TS interfaces | Debugging event flows needs logging infra |
| Loose coupling | Events can fail silently if nobody listens |
| Framework agnostic | Shared contract package needed |

**Best for:** Most MFE architectures. The go-to pattern for peer-to-peer communication.

### 6.3 Props Down from Host

The host passes data and callbacks to MFEs as props. Explicit, traceable, type-safe with TypeScript.

| Pros | Cons |
|---|---|
| Explicit data flow, easy to debug | Host becomes a bottleneck/middleman |
| Type-safe with TypeScript | Tight coupling between host and MFE interfaces |
| Easy to debug | Doesn't scale for many events |

**Best for:** Few MFEs, simple data flow. Auth context, user info, feature flags.

### 6.4 URL / Query Params

The URL is a **natural, framework-agnostic state container**. Any MFE can read/write query params. It's bookmarkable, shareable, and survives page refreshes.

**Best for:** Filter state, search queries, pagination, sort order — any state multiple MFEs need to read and that should survive refresh.

**Limits:** String-only, publicly visible, ~2000 char limit, can't hold complex nested objects.

### 6.5 BroadcastChannel API

For cross-tab communication or MFEs in separate iframes. Example: user logs out in one tab → BroadcastChannel notifies all tabs → all redirect to login. Impossible with events or event bus since those only work within a single document.

### 6.6 Shared Backend / APIs

MFEs don't talk to each other — they share the same backend APIs. Most **loosely coupled** pattern, but the trade-off is **latency** (Cart badge won't update instantly on "Add to Cart"). Pair with optimistic UI updates.

### Comparison

| Pattern | Coupling | Type Safety | Best For |
|---|---|---|---|
| **Custom Events** | Loose | ❌ | Simple, few events |
| **Typed Event Bus** | Loose | ✅ | Most MFE architectures |
| **Props from Host** | Medium | ✅ | Few MFEs, simple data |
| **URL/Query Params** | Loose | ❌ | Filters, navigation state |
| **BroadcastChannel** | Loose | ❌ | Cross-tab, iframes |
| **Shared Backend** | Loose | N/A | Eventually consistent data |

### Best Practice: Layer Your Communication

```
┌────────────────────────────────────────────────────────┐
│  LAYER 1: Props from Host (top-down)                    │
│  Auth token, user profile, theme, feature flags         │
│  Strongest contract — compile-time enforced              │
├────────────────────────────────────────────────────────┤
│  LAYER 2: Typed Event Bus (peer-to-peer)                │
│  'cart:add-item', 'cart:updated', 'search:filter-change'│
│  Medium-strength — typed but runtime-enforced            │
├────────────────────────────────────────────────────────┤
│  LAYER 3: URL State (shared, bookmarkable)              │
│  ?q=laptop&sort=price&page=2                            │
│  Implicit contract — survives refresh, shareable         │
├────────────────────────────────────────────────────────┤
│  LAYER 4: Shared Backend API (eventually consistent)    │
│  Cart count, user preferences, order history            │
│  Loosest contract — persistent data, latency trade-off   │
└────────────────────────────────────────────────────────┘
```

**Anti-pattern:** Reaching for the Event Bus when Props would be simpler. If the host already has the data, just pass it as props. Save the event bus for genuinely peer-to-peer interactions.

[⬆ Back to Top](#top)

---

## Module Federation Deep Dive

### What Is Module Federation?

A Webpack 5 feature that lets an app **dynamically load code from another independently built and deployed app at runtime** while sharing dependencies to avoid duplication. Think of it as **runtime npm**.

Before Module Federation (pre-2020), choices were: npm packages (loses independent deployment), iframes (terrible UX), or custom script-loading hacks (no dependency sharing). Module Federation was created to solve the **shared dependency problem** — multiple independent apps sharing React at runtime without duplicating it.

### Key Concepts

| Concept | What It Is |
|---|---|
| **Host** | App that **consumes** remote modules (the main aggregator) |
| **Remote** | App that **exposes** modules for others to consume |
| **Shared** | Dependencies shared between host and remotes to avoid duplication |
| **Container** | Runtime entry point (`remoteEntry.js`) the host uses to access a remote's modules |
| **Expose** | Declaring which modules a remote makes available (its public API) |
| **Scope** | Isolated namespace to avoid conflicts between remotes |

### Runtime Flow (Step by Step)

This is the critical flow to understand — how code actually loads at runtime:

```
┌───────────────────────────────────────────────────────────┐
│         Module Federation Runtime Flow                     │
├───────────────────────────────────────────────────────────┤
│                                                            │
│  Step 1: Browser loads Host's index.html + main.js         │
│                    │                                       │
│                    ▼                                       │
│  Step 2: main.js calls import('./bootstrap')               │
│          This creates the ASYNC BOUNDARY                   │
│                    │                                       │
│                    ▼                                       │
│  Step 3: Webpack initializes the sharing scope             │
│          Registers Host's versions: React 18.2             │
│                    │                                       │
│                    ▼                                       │
│  Step 4: User navigates to /search                         │
│          lazy(() => import('searchApp/SearchPage'))         │
│                    │                                       │
│                    ▼                                       │
│  Step 5: Webpack fetches remoteEntry.js (~5KB) from CDN    │
│                    │                                       │
│                    ▼                                       │
│  Step 6: SHARING NEGOTIATION:                              │
│          Remote: "I need React ^18.0.0"                    │
│          Host:   "I have React 18.2.0"                     │
│          18.2 satisfies ^18.0 → Remote reuses Host's React │
│                    │                                       │
│                    ▼                                       │
│  Step 7: container.get('./SearchPage') fetches only        │
│          the SearchPage chunk (~50KB), NOT entire bundle   │
│                    │                                       │
│                    ▼                                       │
│  Step 8: SearchPage renders as a normal React component    │
│          Shared React → hooks, context all work correctly  │
│                                                            │
└───────────────────────────────────────────────────────────┘
```

### Build Time vs Runtime

```
Build Time:
┌──────────────────────┐     ┌───────────────────────┐
│   Host App            │     │   Remote: Search MFE   │
│                       │     │                        │
│ webpack.config.js:    │     │ webpack.config.js:     │
│  remotes: {           │     │  name: 'searchApp'     │
│    searchApp: '...'   │     │  exposes: {            │
│  }                    │     │    './SearchPage'      │
│  shared: ['react']    │     │  }                     │
│                       │     │  shared: ['react']     │
└──────────────────────┘     └───────────────────────┘

Runtime:
┌──────────────────────┐      ┌───────────────────────┐
│  Host App (browser)   │      │   CDN                  │
│                       │      │                        │
│ 1. User visits /search│      │  searchApp/            │
│ 2. Loads remoteEntry  │─────>│    remoteEntry.js      │
│ 3. Negotiates shared  │      │    src_SearchPage.js   │
│    deps (React)       │      │                        │
│ 4. Downloads only the │<─────│                        │
│    SearchPage chunk   │      │                        │
│ 5. Renders SearchPage │      │                        │
└──────────────────────┘      └───────────────────────┘
```

### The Async Boundary (Critical!)

Module Federation **requires** an async boundary at the entry point. Without it, shared dependency negotiation fails.

```
index.js  →  import('./bootstrap')  →  bootstrap.js (actual app code)
          │
          └── This gap is where sharing negotiation happens
```

Without the async boundary, the host's main entry executes **synchronously** before the sharing scope is initialized. Remote modules try to resolve `react` from the sharing scope, find nothing, and either crash with "Shared module is not available for eager consumption" or load their own separate copy (defeating sharing).

The `import('./bootstrap')` creates an async chunk boundary, giving Webpack time to: (1) parse all `remoteEntry.js` files, (2) build the sharing scope, (3) resolve version negotiations, (4) then start rendering.

### Dynamic Remotes

Instead of hardcoding remote URLs at build time, load them from a manifest at runtime. This enables true independent deployment — the host doesn't need to rebuild when a remote deploys a new version. The flow: inject `<script>` for `remoteEntry.js`, call `__webpack_init_sharing__`, initialize the container, then get the specific module.

### Bidirectional Federation

Any app can be **both host AND remote** simultaneously. App A exposes Header/Footer while consuming ProductCatalog from App B, and vice versa.

**When to use:** Shared layout components. The Platform team exposes Header/Footer as a remote, but their admin dashboard consumes ProductCatalog from Product MFE.

**Caution:** Circular dependencies can cascade failures. Keep bi-directional surface area small (shared UI shells, not business logic).

[⬆ Back to Top](#top)

---

## Module Federation 2.0

MF 1.0 was Webpack-specific, had no type safety, and required Webpack internal hacks for advanced use cases. MF 2.0 decouples the federation runtime from the bundler.

### Key Improvements

| Feature | MF 1.0 | MF 2.0 |
|---|---|---|
| **Bundler support** | Webpack only | Webpack, Vite, Rspack, Rollup |
| **Type safety** | None | Auto-generated `.d.ts` from remote exports |
| **Manifest** | `remoteEntry.js` only | `mf-manifest.json` with metadata |
| **Runtime API** | Webpack internals (`__webpack_init_sharing__`) | Clean SDK: `init()` and `loadRemote()` |
| **Versioning** | Manual shared config | Automatic negotiation |
| **DevTools** | None | Chrome extension for inspecting remotes |

The biggest win is **type safety**. In MF 1.0, `import('searchApp/SearchPage')` was completely untyped. MF 2.0 generates TypeScript declarations from the remote's actual exports — full autocomplete and compile-time errors.

MF 2.0 uses `@module-federation/enhanced` as a drop-in replacement for `ModuleFederationPlugin`. For new projects, start with MF 2.0. For existing MF 1.0 setups, migration is straightforward.

[⬆ Back to Top](#top)

---

## Shared Dependencies and Versioning

The biggest performance concern in MFEs: **duplicate dependencies**.

### How Sharing Works

```
Host has React 18.2, Remote has React 18.3 in package.json

Sharing negotiation:
  Remote: "I need React ^18.0.0"
  Host:   "I have React 18.2.0"
  18.2 satisfies ^18.0 → Remote uses Host's React ✅ (ONE copy)

If incompatible:
  Remote: "I need React ^19.0.0"
  Host:   "I have React 18.2.0"
  18.2 does NOT satisfy ^19.0 → Remote loads own React ⚠️ (TWO copies)
```

### Why Duplicate React is Catastrophic

It's not just bundle size — it's a **runtime correctness** issue. React uses module-level state for its hooks system (fiber tree, hook queue). Two React instances means:

1. **Hooks break:** `useState` registers in one React's registry. If a component renders in another React's tree, you get "Invalid hook call" — the #1 Module Federation debugging nightmare.
2. **Context doesn't work:** `useContext` across different React instances can't share context. A `ThemeProvider` wrapping the host's React won't provide values to a remote using a different instance.
3. **Events break:** React's synthetic event system is per-instance.

This is why `singleton: true` is **mandatory** for React, ReactDOM, and any library with module-level state.

### Sharing Negotiation Algorithm

```
Remote requests 'react':
  │
  ├── Is singleton: true?
  │    ├── YES → Host has react in share scope?
  │    │    ├── YES → Host version satisfies requiredVersion?
  │    │    │    ├── YES → ✅ Use Host's version (shared!)
  │    │    │    └── NO  → strictVersion: true?
  │    │    │         ├── YES → 💥 RUNTIME ERROR
  │    │    │         └── NO  → ⚠️ Warning, use Host's anyway
  │    │    └── NO  → Remote loads its own (bad!)
  │    │
  │    └── NO (not singleton) →
  │         Version satisfied? → Share (save bandwidth)
  │         Not satisfied?     → Load own copy (OK for stateless libs)
```

### Version Strategy

| Library Type | `singleton` | `strictVersion` | Why |
|---|---|---|---|
| **React / ReactDOM** | `true` | `false` | Multiple instances break hooks, context, events |
| **State libs (Redux)** | `true` | `false` | Need single store |
| **Router** | `true` | `false` | Need single history object |
| **Design System** | `true` | `false` | Consistent styling |
| **Utility libs (lodash)** | `false` | `false` | Stateless — duplication is harmless, just bundle bloat |

### Organizational Strategy

The most dangerous moment is when one team upgrades React and others haven't. Establish a **shared dependency upgrade cadence** — all teams upgrade React together once per quarter. The Platform team proposes the version, tests it in staging, and coordinates rollout.

Set `strictVersion: false` (the default) as a safety net. On version mismatch, Webpack logs a warning but still uses the Host's version. Minor version differences (18.2 vs 18.3) rarely cause real issues. Only use `strictVersion: true` if a mismatch would cause data corruption.

[⬆ Back to Top](#top)

---

## Routing in Micro Frontends

Each MFE has its own routes, but the user sees **one URL bar**.

### Routing Delegation Model

```
URL: /search/results?q=laptop&sort=price
      │        │
      │        └──────────────────┐
      ▼                           ▼
 Host Router                 MFE Router
 matches /search/*           matches /results
 → loads Search MFE          → renders SearchResults

 HOST RESPONSIBILITY:         MFE RESPONSIBILITY:
 - Top-level route matching   - Sub-route matching
 - Loading/unloading MFEs     - Internal navigation
 - Cross-MFE navigation       - Query param management
```

### Approach: Nested Route Delegation

The Host owns top-level routes with wildcard `/*`. Each MFE owns everything under its prefix:

```
Host:
  /search/*   → Search MFE
  /product/*  → Product MFE
  /cart/*     → Cart MFE

Search MFE sub-routes:
  /search/          → SearchHome
  /search/results   → SearchResults
  /search/filters   → SearchFilters
```

### The Single Router Rule

If each MFE uses its own `BrowserRouter`, **multiple router instances fight over the URL bar**. When Search MFE calls `navigate('/cart')`, its own router processes it but the Host's router doesn't know — Cart MFE never loads.

**Solutions:**
1. **MFEs don't have their own router** — receive route info as props
2. **Use `MemoryRouter`** for internal navigation (doesn't touch URL bar), delegate cross-MFE navigation to the Host via callback or event bus
3. **Use `basename`** prop and the Host listens for `popstate` events

Option 2 is the most common in production. The MFE owns internal navigation (tabs, filters) via MemoryRouter. For "leave this MFE" navigation, it fires an event or calls the Host's `navigate()`.

### Cross-MFE Navigation

Three options:
- `window.history.pushState({}, '', '/cart')` + dispatch `popstate` event
- `eventBus.emit('nav:navigate', { path: '/cart' })`
- Call `props.navigate('/cart')` from host-provided callback

### Common Routing Bugs

- **Double router bug:** MFE updates URL but Host's router doesn't see it → wrong MFE renders
- **Back button confusion:** MemoryRouter changes don't touch browser history → back button skips MFE-internal navigation
- **Deep linking failure:** MFE doesn't read URL params on mount → shared URLs show default view instead of expected state

**Golden rule:** The Host owns the URL bar. MFEs can **read** the URL to initialize state, but should only **write** through the Host's navigation API.

[⬆ Back to Top](#top)

---

## Deployment and CI CD

### Independent Pipelines

```
Search MFE:   push → lint → test → build → deploy to CDN → update manifest
Product MFE:  push → lint → test → build → deploy to CDN → update manifest
Host:         push → lint → test → build → deploy (rarely changes)
```

Each MFE has its own pipeline. The Host shell changes rarely.

### Versioning & Rollback

```
CDN structure:
cdn.myapp.com/search/
  v2.3.0/remoteEntry.js   ← Previous
  v2.3.1/remoteEntry.js   ← Current

Manifest points to current. Rollback = update manifest to previous version.
No rebuild needed. Old bundles stay on CDN.
```

**Rollback comparison:**

MFE rollback: Update manifest (one config change, ~60 seconds propagation)
Monolith rollback: Revert interleaved git commits, cherry-pick, rebuild (15 min), redeploy (30–60 min)

**Canary deployments:** The manifest serves different versions to different user cohorts. Start at 1% traffic, monitor errors, scale to 100% if healthy.

### Testing Strategy

- **Per-MFE:** Unit + integration tests in each pipeline (fast, isolated)
- **Composition testing:** Staging environment running latest of every MFE from manifest. Run E2E tests (Playwright) on every MFE deploy. Block manifest update if E2E fails.
- **Contract testing:** Verify cross-MFE event payload shapes. If Product changes `cart:add-item` payload, Cart's contract test fails before deployment. Use Pact or JSON Schema validation.
- **Breaking changes:** Deploy the **consumer** first (handles old + new format), then deploy the **producer**. Same "expand and contract" pattern as database migrations.

[⬆ Back to Top](#top)

---

## Real World Examples

Every company below adopted MFEs because of an **organizational problem**, not a technology problem.

| Company | Architecture | Why |
|---|---|---|
| **IKEA** | Module Federation + React | 30+ country teams need geographic deployment autonomy |
| **Spotify** | iframes → Web Components | 100+ squads need maximum isolation. Migrating from iframes for better performance. |
| **Zalando** | Server-side composition (Tailor) | SEO-critical e-commerce. Streams HTML fragments from multiple services. |
| **Amazon** | Server-side + client-side JS | Thousands of engineers. Product details, reviews, recommendations = separate teams. |
| **Banking (Empresas)** | Single-SPA + Module Federation | Regulatory compliance. Each product (loans, cards) has different audit cycles. |

The pattern: IKEA needed geographic autonomy, Spotify needed squad independence, Zalando needed SEO with team autonomy, Amazon needed thousands of engineers to not step on each other, banking needed regulatory isolation.

If asked "Should we use MFEs?", the right first question is: **"What organizational problem are you solving?"** If the answer is "slow builds" or "large bundles", solve those within a monolith (code-splitting, CDN caching). If the answer is "teams blocking each other's deployments", that's when MFEs earn their complexity tax.

[⬆ Back to Top](#top)

---

## Decision Flowchart

```
Start
  │
  ├── How many teams?
  │     │
  │     ├── 1-3 teams → MONOLITH (with code-splitting)
  │     │                 Lazy routes, good folder structure
  │     │
  │     └── 4+ teams →
  │           │
  │           ├── Clear domain boundaries?
  │           │     │
  │           │     ├── Yes → MICRO-FRONTENDS
  │           │     │          │
  │           │     │          ├── Need framework mixing?
  │           │     │          │     ├── Yes → Web Components / Single-SPA
  │           │     │          │     └── No  → Module Federation
  │           │     │          │
  │           │     │          ├── SEO critical?
  │           │     │          │     ├── Yes → Server-side composition
  │           │     │          │     └── No  → Client-side composition
  │           │     │          │
  │           │     │          └── Legacy migration?
  │           │     │                ├── Yes → iframes → migrate to MF
  │           │     │                └── No  → Module Federation from day 1
  │           │     │
  │           │     └── No (highly coupled) → MONOLITH (monorepo + Nx/Turborepo)
  │           │
  │           └── Not sure → Monolith with CLEAR MODULE BOUNDARIES
  │                          Migrate to MFE when pain points emerge
  └── End
```

### Quick Decision Matrix

| Situation | Recommendation |
|---|---|
| Small app, 1-2 teams | Monolith. MFE overhead far exceeds benefits. |
| Growing app, 3-5 teams | Monorepo with clear boundaries (Nx/Turborepo). Evaluate MFE. |
| Large app, 5+ teams, clear domains | Module Federation. Sweet spot for MFE. |
| Legacy migration | Strangler Fig. iframes first, extract MFEs gradually. |
| SEO-critical | Server-side composition (Podium / custom SSR). |
| Third-party widgets | iframes or Web Components. Maximum isolation. |
| Different tech stacks | Single-SPA or Web Components. |

[⬆ Back to Top](#top)

---

## Key Takeaways

1. **Monolith first.** Don't adopt MFE until you feel the pain at organizational scale. MFE is an organizational architecture decision, not a technical one.

2. **Module Federation** is the most popular approach for React-based MFEs. It solves the shared dependency problem. Use MF 2.0 for type safety and bundler flexibility (Vite, Rspack).

3. **App Shell** is the standard hosting pattern — thin host handles routing, layout, auth. Keep it stable so it rarely redeploys.

4. **Cross-MFE communication** should be layered: Props for auth/config, Typed Event Bus for peer-to-peer, URL for filters/navigation, Shared API for persistent data. Never import another MFE's code directly.

5. **Shared singletons** (`singleton: true` for React, Redux, Router) prevent duplicate instances that break hooks, context, and events.

6. **Async boundary** (`import('./bootstrap')`) is mandatory for Module Federation's sharing negotiation to work.

7. **Manifest + CDN** enables true independent deployment, instant rollbacks, canary releases, and A/B testing without rebuilding the host.

8. The right answer to "Should we use MFEs?" is: **"It depends on your organization's size, structure, DevOps maturity, and whether monolith pain exceeds MFE overhead."**

---

[⬆ Back to Top](#top)

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design

[⬆ Back to Top](#top)
