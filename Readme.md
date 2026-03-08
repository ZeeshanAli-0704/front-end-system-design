
# Front End System Design with Zeeshan Ali

Welcome to the **System Design with Zeeshan Ali** repository! This collection is dedicated to in-depth articles and resources on system design, covering foundational concepts, architecture patterns, optimization techniques, and real-world case studies. Each section is carefully curated to provide clear explanations, practical examples, and best practices for building scalable, reliable, and efficient systems.

* **GitHub**: [System Design with Zeeshan Ali](https://github.com/ZeeshanAli-0704/front-end-system-design)
* **Dev.to**: [System Design with Zeeshan Ali](https://dev.to/t/systemdesignwithzeeshanali)

---

## Folder Structure

### Case Studies
* **[FaceBook](./Case_Studies/Facebook/Frontend-System-Design-Facebook-News-Feed.md)**: FaceBook News Feed Design, Components, API, DataModel

* **[Pinterest](./Case_Studies/Pinterest/Frontend-System-Design-Pinterest.md)**: Pinterest Design, Components, API, DataModel

* **[TypeHead](./Case_Studies/TypeHead)**: Auto complete backend & Front end Design, LLD, HLD, UI usage

### Concepts

* **[CSS, CSSOM, and DOM — Browser Rendering Deep Dive](./Concepts/CSS-CSSOM-and-DOM-Rendering-in-Browser.md)**: How browsers build the DOM, CSSOM, and Render Tree — the full rendering pipeline (parse → layout → paint → composite), reflow vs repaint, Virtual DOM, progressive rendering, debugging tools, and framework-specific rendering behavior.

* **[Critical Rendering Path](./Concepts/Critical-Rendering-Path.md)**: The **Critical Rendering Path** refers to the sequence of steps the browser takes to convert your HTML, CSS, and JavaScript into **pixels rendered on the user’s screen**.

* **[Web Worker, Shared Worker & Service Worker](./Concepts/Web_Worker_SharedWorker_and_Service_Worker.md)**: A mental-model-first guide to all three browser workers — what they are, when to use each, communication patterns (postMessage, MessagePort, clients API, BroadcastChannel), code snippets (promise wrappers, transferable objects, worker pools, caching strategies, push notifications, background sync), real-world architectures (Figma, Google Docs, trading dashboards, e-commerce PWAs), trade-offs, gotchas, and a decision flowchart.

* **[Modern Large Scale Reducer Design](./Concepts/Modern_Large_Scale_Reducer.md)**: A complete guide to scalable state architecture — feature-sliced design, Redux Toolkit slices, RTK Query, normalized state with Entity Adapter, memoized selectors, async patterns, cross-feature communication, middleware, code-splitting reducers, testing, migration strategies, and framework-specific patterns (React, Angular, Vue).

* **[Redux Toolkit vs Zustand vs Jotai](./Concepts/Redux_Toolkit-vs-Zustand-vs-Jotai.md)**: A complete guide to choosing the right React state management — mental models (centralized vs hook-based vs atomic), architecture patterns, side-by-side feature comparison, performance & re-render behavior, middleware & DevTools, TypeScript & SSR support, scalability for teams, decision frameworks, real-world use-case mapping, and migration guidance.

* **[Scalable CSS Architecture](./Concepts/Scalable-CSS-Architecture.md)**: A complete guide to scalable CSS — BEM, SMACSS, OOCSS, CSS Modules, CSS-in-JS, Tailwind, Vanilla Extract, CSS @layer, design tokens, theming, migration strategies, performance, auditing, and choosing the right approach for your team.

* **[Web Accessibility (A11y) Complete Guide](./Concepts/Web_Accessibility_A11y_Complete_Guide.md)**: A comprehensive guide to building inclusive web apps — WCAG standards, auditing, fixing issues, keyboard/screen reader testing, ARIA, accessible forms, SPAs, and CI/CD integration.

* **[Rendering Patterns — CSR, SSR, SSG & Prerendering](./Concepts/Rendering-Patterns.md)**: A deep-dive into every rendering strategy — Client-Side Rendering, Server-Side Rendering, Universal/Isomorphic Rendering, Static Site Generation, and Prerendering — with request/response flows, code samples, comparison matrix, and a decision flowchart.

* **[Optimization — Frontend Performance](./Concepts/Optimization.md)**: Hub page linking to all five asset optimization articles with a quick-reference performance metrics impact table (LCP, FCP, CLS, TTI, TTFB).

  * **[Image Optimization](./Concepts/image-optimization.md)**: Image formats (JPEG, PNG, WebP, AVIF, SVG), responsive images (srcset, sizes), lazy loading, progressive images, compression (lossy vs lossless), Image CDN, art direction (`<picture>`), and placeholder strategies (LQIP, BlurHash, dominant color).

  * **[Video Optimization](./Concepts/video-optimization.md)**: Progressive enhancement, video formats (WebM, MP4, AV1), poster images, removing audio, adaptive streaming (HLS/DASH), platform-based dimensions, and preloading strategy.

  * **[Font Optimization](./Concepts/font-optimization.md)**: FOUT/FOIT issues, `font-display` strategy, WOFF2/WOFF formats, font preloading, subsetting, variable fonts, and Font Face Observer.

  * **[CSS, JavaScript & UI Optimization](./Concepts/css-js-ui-optimization.md)**: Critical CSS, async/lazy CSS loading, JS async & defer, module loading & tree shaking, configurable UI (conditional rendering), and streaming UI rendering with React Suspense.

  * **[Network Optimization](./Concepts/network-optimization.md)**: HTTP/2 & HTTP/3, resource hints (preload, prefetch, preconnect, dns-prefetch), caching strategies, Gzip/Brotli compression, CDN, bundle & code splitting, and Service Workers.
