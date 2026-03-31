# Frontend System Design with Zeeshan Ali

> A comprehensive collection of frontend system design concepts, architecture patterns, optimization techniques, and real world case studies. Each article provides clear explanations, practical examples, and best practices for building scalable, reliable, and efficient systems.

- **GitHub**: [System Design with Zeeshan Ali](https://github.com/ZeeshanAli-0704/front-end-system-design)
- **Dev.to**: [System Design with Zeeshan Ali](https://dev.to/t/systemdesignwithzeeshanali)

---

<a id="top"></a>

## Table of Contents

- [Case Studies](#case-studies)
- [Concepts](#concepts)
  - [Browser Rendering and CRP](#browser-rendering-and-crp)
  - [Asset Optimization](#asset-optimization)
  - [Rendering Patterns and CSS Architecture](#rendering-patterns-and-css-architecture)
  - [State Management](#state-management)
  - [Communication Protocols and Data](#communication-protocols-and-data)
  - [Authentication and Security](#authentication-and-security)
  - [Performance and Monitoring](#performance-and-monitoring)
  - [Offline Support and PWAs](#offline-support-and-pwas)
  - [Architecture Patterns](#architecture-patterns)
  - [Accessibility](#accessibility)
  - [Workers](#workers)
  - [Logging Analytics and Feature Flags](#logging-analytics-and-feature-flags)
  - [Miscellaneous](#miscellaneous)

[⬆ Back to Top](#top)

---

## Case Studies

| # | Case Study | Description |
|---|-----------|-------------|
| 1 | [Facebook News Feed](./Case_Studies/Facebook/Frontend_System_Design_Facebook_News_Feed.md) | Facebook News Feed Design, Components, API, Data Model |
| 2 | [Pinterest](./Case_Studies/Pinterest/Frontend_System_Design_Pinterest.md) | Pinterest Design, Components, API, Data Model |
| 3 | [Typeahead / Autocomplete](./Case_Studies/Typeahead) | Autocomplete backend and frontend design, LLD, HLD, UI usage |
| 4 | [Search Results Page](./Case_Studies/Search) | Search Results Page (Google / Amazon style) — polymorphic result cards, faceted filtering, URL state sync, pagination, query highlighting |
| 5 | [Excalidraw (Collaborative Whiteboard)](./Case_Studies/Excalidraw/Frontend_System_Design_Excalidraw.md) | Excalidraw Design — Canvas 2D rendering engine, IndexedDB local-first storage, Rough.js hand-drawn style, CRDT collaboration, undo/redo, E2E encryption |
| 6 | [Google Docs (Collaborative Document Editor)](./Case_Studies/GoogleDocs/Frontend_System_Design_Google_Docs.md) | Google Docs Design — Rich text editor engine, Operational Transformation (OT) deep dive, CRDTs vs OT, real-time cursor presence, document model, collaborative undo/redo |

[⬆ Back to Top](#top)

---

## Concepts

### Browser Rendering and CRP

| # | Topic | Description |
|---|-------|-------------|
| 1 | [Critical Rendering Path](./Concepts/Critical_Rendering_Path.md) | The sequence of steps the browser takes to convert HTML, CSS, and JS into pixels. Covers CRP optimization techniques, resource hints, inline critical CSS, defer JS, and performance metrics. |
| 2 | [DOM, CSSOM and Browser Rendering Pipeline](./Concepts/CSS_CSSOM_and_DOM_Rendering_in_Browser.md) | How browsers build the DOM, CSSOM, and Render Tree. Full rendering pipeline (parse, layout, paint, composite), reflow vs repaint, Virtual DOM, progressive rendering, and debugging tools. |

[⬆ Back to Top](#top)

### Asset Optimization

| # | Topic | Description |
|---|-------|-------------|
| 3 | [Optimization Index](./Concepts/Optimization.md) | Hub page linking to all asset optimization articles with a quick reference performance metrics impact table (LCP, FCP, CLS, TTI, TTFB). |
| 4 | [Image Optimization](./Concepts/Image_Optimization.md) | Image formats (JPEG, PNG, WebP, AVIF, SVG), responsive images (srcset, sizes), lazy loading, compression, Image CDN, art direction, and placeholder strategies (LQIP, BlurHash). |
| 5 | [Video Optimization](./Concepts/Video_Optimization.md) | Progressive enhancement, video formats (WebM, MP4, AV1), poster images, adaptive streaming (HLS, DASH), platform based dimensions, preloading, and accessibility. |
| 6 | [Font Optimization](./Concepts/Font_Optimization.md) | FOUT and FOIT issues, font display strategy, WOFF2 and WOFF formats, font preloading, subsetting, variable fonts, and Font Face Observer. |
| 7 | [CSS, JS and UI Optimization](./Concepts/CSS_JS_UI_Optimization.md) | Critical CSS, async and lazy CSS loading, JS async and defer, module loading and tree shaking, configurable UI, streaming UI rendering with React Suspense. |
| 8 | [Network Optimization](./Concepts/Network_Optimization.md) | HTTP 2 and HTTP 3, resource hints (preload, prefetch, preconnect), caching strategies, Gzip and Brotli compression, CDN, bundle and code splitting, Service Workers, 103 Early Hints, stale while revalidate. |

[⬆ Back to Top](#top)

### Rendering Patterns and CSS Architecture

| # | Topic | Description |
|---|-------|-------------|
| 9 | [Rendering Patterns](./Concepts/Rendering_Patterns.md) | CSR, SSR, Universal Isomorphic Rendering, SSG, ISR, Prerendering, React Server Components, Streaming SSR with request response flows, comparison matrix, and decision flowchart. |
| 10 | [Scalable CSS Architecture](./Concepts/Scalable_CSS_Architecture.md) | BEM, SMACSS, OOCSS, CSS Modules, CSS in JS, Tailwind, Vanilla Extract, CSS layers, design tokens, theming, migration strategies, performance, and auditing. |

[⬆ Back to Top](#top)

### State Management

| # | Topic | Description |
|---|-------|-------------|
| 11 | [Modern Large Scale Reducer Design](./Concepts/Modern_Large_Scale_Reducer.md) | Scalable state architecture with feature sliced design, Redux Toolkit slices, RTK Query, normalized state, memoized selectors, async patterns, and cross feature communication. |
| 12 | [Redux Toolkit vs Zustand vs Jotai](./Concepts/Redux_Toolkit_vs_Zustand_vs_Jotai.md) | Mental models (centralized vs hook based vs atomic), side by side feature comparison, performance and re render behavior, middleware, DevTools, TypeScript, SSR support, and decision frameworks. |

[⬆ Back to Top](#top)

### Communication Protocols and Data

| # | Topic | Description |
|---|-------|-------------|
| 13 | [Communication Protocols and Real Time Data](./Concepts/Communication_Protocols_and_Real_Time_Data.md) | HTTP, Short Polling, Long Polling, SSE, WebSockets, WebRTC, gRPC Web, GraphQL Subscriptions, microservice communication patterns, comparison table, and decision flowchart. |
| 14 | [Pagination Patterns](./Concepts/Pagination_Patterns.md) | Offset based, Cursor based, Keyset, Page number pagination. Frontend patterns: Infinite Scroll, Load More, Virtual Scroll. Real world architecture examples and edge cases. |
| 15 | [Browser Caching for Web Apps](./Concepts/Browser_Caching_Web_Apps.md) | Cache Control headers, content hashed filenames, deployment order, CDN best practices, Service Worker caching, ETags, and a deployment checklist. |
| 16 | [Evolution of the Web HTTP Protocol](./Concepts/Evolution_the_Web_HTTP_Protocol.md) | Comparing HTTP 1.1, HTTP 2, and HTTP 3. Head of line blocking, multiplexing, QUIC, 0 RTT handshake, and connection migration. |

[⬆ Back to Top](#top)

### Authentication and Security

| # | Topic | Description |
|---|-------|-------------|
| 17 | [Authentication Flows](./Concepts/Authentication_Flows.md) | Session based, JWT, OAuth 2.0, SSO. Token storage, refresh token rotation, protected routes, Axios interceptor patterns, and role based UI rendering. |
| 18 | [Frontend Security](./Concepts/Frontend_Security.md) | XSS, CSRF, Clickjacking prevention. CSP headers, CORS deep dive, secure cookie handling, input sanitization, JWT security, iframe security, SRI, and dependency security. |

[⬆ Back to Top](#top)

### Performance and Monitoring

| # | Topic | Description |
|---|-------|-------------|
| 19 | [Performance Monitoring and Observability](./Concepts/Performance_Monitoring_and_Observability.md) | Core Web Vitals (LCP, INP, CLS), identifying and fixing bottlenecks, Chrome DevTools, RUM vs Synthetic monitoring, error tracking, performance budgets, and the three pillars of observability. |
| 20 | [Virtualization and Large Data Sets](./Concepts/Virtualization_and_Large_Data_Sets.md) | List and table virtualization, infinite scrolling patterns, canvas based rendering for extreme data, library comparison (react window, react virtuoso, TanStack Virtual), and accessibility. |

[⬆ Back to Top](#top)

### Offline Support and PWAs

| # | Topic | Description |
|---|-------|-------------|
| 21 | [Offline Support and PWAs](./Concepts/Offline_Support_and_PWA.md) | Service Worker lifecycle, cache strategies (Cache First, Network First, Stale While Revalidate), IndexedDB, App Shell architecture, Background Sync, Push Notifications, Web App Manifest, and testing. |

[⬆ Back to Top](#top)

### Architecture Patterns

| # | Topic | Description |
|---|-------|-------------|
| 22 | [Micro Frontends and Module Federation](./Concepts/Micro_Frontends_and_Module_Federation.md) | Monolith vs MFE tradeoffs, integration approaches (iframes, JS remotes, Web Components, Module Federation, server side composition), App Shell, cross MFE communication, MF 2.0, routing, deployment, and real world examples (IKEA, Spotify, Zalando, Amazon). |
| 23 | [Event Handling and Pub Sub Patterns](./Concepts/Event_Handling_and_PubSub_Patterns.md) | Event delegation, debounce, throttle, requestIdleCallback, custom typed event bus, AbortController for cancellable requests, memory leak detection, and real world architecture patterns. |

[⬆ Back to Top](#top)

### Accessibility

| # | Topic | Description |
|---|-------|-------------|
| 24 | [Web Accessibility (A11y)](./Concepts/Web_Accessibility_A11y_Complete_Guide.md) | WCAG standards, POUR principles, semantic HTML, ARIA, keyboard accessibility, focus management, color contrast, accessible forms, images, media, SPAs, modals, and CI CD integration with axe and Pa11y. |

[⬆ Back to Top](#top)

### Workers

| # | Topic | Description |
|---|-------|-------------|
| 25 | [Web Worker, Shared Worker and Service Worker](./Concepts/Web_Worker_SharedWorker_and_Service_Worker.md) | Mental model for all three workers, communication patterns (postMessage, MessagePort, clients API, BroadcastChannel), worker pools, transferable objects, caching strategies, push notifications, background sync, and real world architectures. |

[⬆ Back to Top](#top)

### Logging Analytics and Feature Flags

| # | Topic | Description |
|---|-------|-------------|
| 26 | [Logging, Analytics and Feature Flags](./Concepts/Logging_Analytics_and_Feature_Flags.md) | Analytics architecture (event taxonomy, batching, funnel tracking), A B testing infrastructure, feature flag systems (LaunchDarkly, Unleash), session replay and heatmaps, frontend error tracking (Sentry, source maps, Error Boundaries), and full logging pipeline. |

[⬆ Back to Top](#top)

---

## Contributing

### File Naming Convention

All files and folders in this repository follow **Snake_Case** — each word is capitalized and separated by underscores.

| Rule | Example |
|------|---------|
| Capitalize each word | `Image_Optimization.md` |
| Separate words with underscores | `Critical_Rendering_Path.md` |
| Acronyms stay uppercase | `CSS_JS_UI_Optimization.md` |
| Connectors (`and`, `in`, `the`, `vs`) stay lowercase | `Redux_Toolkit_vs_Zustand_vs_Jotai.md` |

Feel free to open issues or submit pull requests if you find any errors or want to suggest improvements.

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design

[⬆ Back to Top](#top)