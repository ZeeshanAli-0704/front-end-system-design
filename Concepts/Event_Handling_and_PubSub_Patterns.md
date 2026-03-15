# Event Handling and Pub/Sub Patterns, Frontend System Design Guide

> Event delegation, bubbling, debounce/throttle, requestIdleCallback, custom event buses, AbortController, and memory leak detection, with theory, simple code, ASCII diagrams, and interview ready answers.

---

## Table of Contents

1. [Why Event Handling Matters at Scale](#1-why-event-handling-matters-at-scale)
2. [DOM Event Model, Capture, Target, Bubble](#2-dom-event-model-capture-target-bubble)
3. [Event Delegation, The Scalable Pattern](#3-event-delegation-the-scalable-pattern)
4. [Debounce, Throttle, requestIdleCallback](#4-debounce-throttle-requestidlecallback)
5. [Custom Event Bus / Pub Sub](#5-custom-event-bus--pub-sub-for-decoupled-communication)
6. [AbortController for Cancellable Requests](#6-abortcontroller-for-cancellable-requests)
7. [Memory Leaks, Detection, Diagnosis and Prevention](#7-memory-leaks-detection-diagnosis-and-prevention)
8. [Combining Patterns, Real World Architectures](#8-combining-patterns-real-world-architectures)
9. [Comparison Tables](#9-comparison-tables)
10. [Interview Questions and Answers](#10-interview-questions-and-answers)

---

## 1. Why Event Handling Matters at Scale

Modern frontend apps handle **thousands of events per second**: clicks, scrolls, key presses, network responses, resize, intersection observations, web socket messages, etc.

Naive event handling leads to:

| Problem | Root Cause |
|---|---|
| Memory leaks | Listeners not cleaned up on unmount |
| Layout thrashing | Synchronous DOM reads inside scroll/resize handlers |
| Dropped frames | Too many handlers firing per frame (> 16ms budget) |
| Tight coupling | Components directly calling each other |
| Race conditions | Stale network responses overwriting fresh data |
| Zombie requests | Navigated away but fetch still in-flight |

**Good event architecture** is the difference between a janky prototype and a production-grade application.

### How Events Flow Through a Modern App

```
 User Interaction (click, scroll, type, etc.)
       │
       ▼
 ┌─────────────┐
 │  DOM Event   │ ◄── Browser fires event → capture → target → bubble phases
 │  Pipeline    │     Delegation catches it on a parent element
 └──────┬──────┘
        │
        ▼
 ┌─────────────┐
 │  Rate Limit  │ ◄── debounce / throttle / rIC decide WHEN to process
 │  Layer       │     Prevents flooding the system with too many calls
 └──────┬──────┘
        │
        ▼
 ┌─────────────┐
 │  Event Bus   │ ◄── pub/sub broadcasts to all interested modules
 │  / Pub-Sub   │     Decouples who PRODUCES events from who CONSUMES them
 └──────┬──────┘
        │
        ▼
 ┌─────────────┐
 │  Async Work  │ ◄── AbortController cancels stale/zombie requests
 │  (fetch/WS)  │     Prevents race conditions & wasted bandwidth
 └─────────────┘
```

**How to read this diagram:** A user action enters at the top. Each layer processes or filters the event before passing it down. The DOM layer catches it, the rate-limit layer controls frequency, the event bus broadcasts it to multiple consumers, and the async layer handles network calls with cancellation support. Each layer is independent — you can use any combination.

---

## 2. DOM Event Model, Capture, Target, Bubble

### 2.1 The Three Phases

Every DOM event travels through **three phases**. Think of it like throwing a stone into water — it sinks down (capture), hits the bottom (target), and ripples come back up (bubble).

```
             Window
               │
       ┌───────┴───────┐
       │   CAPTURE ↓    │  Phase 1: Capturing (top → target)
       │   Document     │  The event travels DOWN from Window
       │      │         │  through each ancestor toward the
       │    <html>      │  target element. Rarely used, but
       │      │         │  useful for intercepting events
       │    <body>      │  before they reach children.
       │      │         │
       │    <div>       │
       │      │         │
       │  ┌───┴───┐     │
       │  │<button>│    │  Phase 2: Target
       │  │ TARGET │    │  The event arrives at the element
       │  └───┬───┘     │  that was actually clicked/interacted
       │      │         │  with. Both capture & bubble handlers
       │   BUBBLE ↑     │  on this element fire here.
       │    <div>       │
       │      │         │  Phase 3: Bubbling (target → top)
       │    <body>      │  The event travels BACK UP to Window.
       │      │         │  This is the default phase and the
       │    <html>      │  basis for EVENT DELEGATION.
       │   Document     │
       └───────┴───────┘
             Window
```

**Why this matters:** Event delegation (Section 3) works because of the **bubble phase**. When you click a `<button>` deep in the DOM, the event bubbles up through every ancestor — so a listener on a parent `<div>` can catch clicks on ALL its children.

### 2.2 Listener Registration

```js
// Bubbling phase (default) — most common
element.addEventListener('click', handler);

// Capture phase — fires BEFORE the target
element.addEventListener('click', handler, { capture: true });

// Full options object
element.addEventListener('click', handler, {
  capture: false,   // Listen during bubble phase
  once: true,       // Auto-remove after first invocation
  passive: true,    // Promise not to call preventDefault()
  signal: controller.signal  // AbortController for removal
});
```

### 2.3 `stopPropagation()` vs `stopImmediatePropagation()` vs `preventDefault()`

| Method | What It Does | Use When |
|---|---|---|
| `stopPropagation()` | Stops event from traveling to parent/child elements | You want to handle the event only at this level |
| `stopImmediatePropagation()` | Stops propagation AND prevents other handlers on same element | Multiple handlers on same element, only one should fire |
| `preventDefault()` | Prevents browser's default action (form submit, link navigation) | You want custom behavior instead of default |

```js
// stopPropagation — parent won't see the click
child.addEventListener('click', (e) => {
  e.stopPropagation();
  console.log('Only child handles this');
});

// preventDefault — link won't navigate
link.addEventListener('click', (e) => {
  e.preventDefault();
  router.push(link.href); // custom navigation
});
```

### 2.4 Passive Listeners, The Scroll Optimization

By default, the browser **must wait** for `touchstart`/`touchmove`/`wheel` handlers to finish before it knows whether to scroll (the handler might call `preventDefault()`). This adds latency.

```js
// ❌ Bad — browser waits, scroll feels janky
window.addEventListener('touchstart', handler);

// ✅ Good — browser scrolls immediately (passive = "I won't preventDefault")
window.addEventListener('touchstart', handler, { passive: true });
```

**Chrome, Edge, Firefox** make `touchstart`, `touchmove`, and `wheel` listeners on `document`/`window` **passive by default** since 2017. Declaring `{ passive: true }` explicitly is still best practice for clarity.

### 2.5 Event Properties Cheat Sheet

```js
event.target         // The ACTUAL element that was clicked (innermost)
event.currentTarget  // The element the listener is ATTACHED to
event.type           // 'click', 'keydown', etc.
event.isTrusted      // true = real user action, false = script-dispatched
event.composedPath() // Full path from target to window (crosses Shadow DOM)
```

---

## 3. Event Delegation — The Scalable Pattern

### 3.1 The Problem

Imagine a list of 10,000 items. Attaching a listener to each one is expensive:

```js
// ❌ 10,000 listeners — memory & registration overhead
items.forEach(item => {
  item.addEventListener('click', handleItemClick);
});
```

| Items | Listeners | Impact |
|---|---|---|
| 100 | 100 | Negligible |
| 1,000 | 1,000 | Noticeable |
| 10,000 | 10,000 | Significant |
| 100,000 | 100,000 | 🔴 Problematic |

### 3.2 The Solution, Event Delegation

Attach **one listener** on a common ancestor. The event **bubbles up** from the clicked child, and you use `event.target` to figure out which child was clicked.

```
┌───────────────────────────────────────┐
│ <ul id="list">  ◄── ONE listener here │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐  │
│  │ <li>    │ │ <li>    │ │ <li>    │  │
│  │ Item 1  │ │ Item 2  │ │ Item 3  │  │
│  └─────────┘ └─────────┘ └─────────┘  │
│          ... 10,000 items ...         │
└───────────────────────────────────────┘
       Click on Item 2 bubbles up ↑
       Handled by the single <ul> listener
```

**How it works:** Instead of 10,000 listeners on 10,000 `<li>` elements, you put ONE listener on the `<ul>`. When any `<li>` is clicked, the event bubbles up to the `<ul>`, and your handler checks `event.target` to know which `<li>` was clicked.

```js
// ✅ One listener for ALL items (current & future)
const list = document.getElementById('list');

list.addEventListener('click', (e) => {
  const item = e.target.closest('li'); // find the <li> even if user clicked a child inside it
  if (!item) return;                   // click was on <ul> itself
  if (!list.contains(item)) return;    // safety check

  handleItemClick(item.dataset.id);
});
```

### 3.3 `closest()`, The Key to Robust Delegation

`event.target` might be a **nested element** (e.g., an `<svg>` icon inside a `<button>`). Use `closest()` to walk up and find the real delegate:

```js
container.addEventListener('click', (e) => {
  const btn = e.target.closest('[data-action]');
  if (!btn) return;

  switch (btn.dataset.action) {
    case 'delete': handleDelete(btn); break;
    case 'edit':   handleEdit(btn);   break;
  }
});
```

### 3.4 Why Delegation Handles Dynamic Content Automatically

Since the listener is on the **parent**, any new children added later are automatically covered — the parent listener doesn't care when the child was added, only that the event bubbles up.

### 3.5 When NOT to Use Delegation

| Scenario | Why |
|---|---|
| `focus`/`blur` events | Don't bubble (use `focusin`/`focusout` instead) |
| `mouseenter`/`mouseleave` | Don't bubble (use `mouseover`/`mouseout`) |
| Events on `<canvas>` | No child elements — use coordinates |
| Single specific element handlers | No performance benefit |

### 3.6 React / Framework Event Delegation

React already uses delegation internally:

```
React ≤ 16: All events delegated to document
React 17+:  All events delegated to the React root container
```

You write per-component handlers (`onClick`), but React attaches a **single listener** at the root and routes events using its Synthetic Event system. If you use vanilla `addEventListener` inside `useEffect`, you must manage delegation yourself.

---

## 4. Debounce, Throttle, requestIdleCallback

### 4.1 Mental Model, The Core Difference

```
User typing: a-b-c-d-e-f (6 keystrokes in 600ms)
Time:        0  100  200  300  400  500  600  700  800  900

No limiting:  ✓  ✓   ✓   ✓   ✓   ✓         ← 6 calls (wasteful)

Throttle      ✓              ✓              ✓  ← every 300ms (at most)
(300ms):      │              │              │
              fires          fires          fires

Debounce      ×  ×   ×   ×   ×   ✓         ← 1 call (after 300ms quiet)
(300ms):      resets each time       wait 300ms ──┘

rIC:          ····················✓          ← when browser has free time
```

**How to read this diagram:**
- **No limiting**: Every keystroke fires the function. 6 keystrokes = 6 API calls. Wasteful.
- **Throttle (300ms)**: The function fires on the first call, then ignores subsequent calls until 300ms passes, then fires again. It's like a "rate limit" — at most once per interval.
- **Debounce (300ms)**: The timer resets on every keystroke. The function only fires once the user **stops** for 300ms. It waits for "calm."
- **requestIdleCallback**: Fires whenever the browser has nothing else to do — could be soon or seconds later.

| Technique | When It Fires | Best For |
|---|---|---|
| **Debounce** | After N ms of **silence** | Search input, auto-save, resize end |
| **Throttle** | At most once every N ms | Scroll position, mousemove, analytics |
| **requestIdleCallback** | When browser is **idle** | Non-urgent work, analytics, pre-fetching |

### 4.2 Debounce, Wait Until Calm

Debounce delays execution until the user **stops** triggering the event for a specified period. If the event fires again during the wait, the timer resets.

```
Without debounce (every keystroke triggers API call):
  Type "react" → 5 API calls:  r, re, rea, reac, react  (4 wasted!)

With debounce (300ms):
  Type "react" → 1 API call:   react  (fires 300ms after last keystroke)
```

**When to use:** Anytime you care about the **final** value, not intermediate ones — search input, auto-save, form validation, window resize end.

#### Simple Implementation

```js
function debounce(fn, delay) {
  let timerId;
  return function (...args) {
    clearTimeout(timerId);
    timerId = setTimeout(() => fn.apply(this, args), delay);
  };
}
```

**How it works:** Every time the returned function is called, it clears the previous timer and starts a new one. The original `fn` only runs when there's a pause of `delay` ms.

#### Usage

```js
// Search input — only call API after user stops typing for 300ms
const input = document.getElementById('search');
const debouncedSearch = debounce((query) => {
  fetch(`/api/search?q=${encodeURIComponent(query)}`)
    .then(res => res.json())
    .then(data => renderResults(data));
}, 300);

input.addEventListener('input', (e) => debouncedSearch(e.target.value));

// Auto-save — save 1 second after user stops editing
const debouncedSave = debounce(saveDocument, 1000);
editor.addEventListener('input', debouncedSave);

// Window resize — recalculate layout after resizing stops
window.addEventListener('resize', debounce(recalculateLayout, 200));
```

#### React Hook

```jsx
function useDebounce(fn, delay) {
  const timerRef = useRef(null);
  const fnRef = useRef(fn);
  fnRef.current = fn;

  useEffect(() => () => clearTimeout(timerRef.current), []);

  return useCallback((...args) => {
    clearTimeout(timerRef.current);
    timerRef.current = setTimeout(() => fnRef.current(...args), delay);
  }, [delay]);
}

// Usage
function SearchBar() {
  const [query, setQuery] = useState('');
  const debouncedFetch = useDebounce((q) => {
    fetch(`/api/search?q=${q}`).then(/* ... */);
  }, 300);

  return <input value={query} onChange={(e) => {
    setQuery(e.target.value);
    debouncedFetch(e.target.value);
  }} />;
}
```

### 4.3 Throttle, Rate Limit Execution

Throttle ensures a function is called **at most once** within a given time window. Unlike debounce, it fires periodically DURING continuous activity.

```
Scroll events firing at 60fps = ~60 calls/sec

Throttle (200ms):
  Time: 0   50  100  150  200  250  300  350  400
  Fire: ✓                   ✓                   ✓
        │                   │                   │
        fires immediately   fires again         fires again
```

**When to use:** Anytime you need **periodic updates** during continuous activity — scroll position tracking, drag-and-drop, mousemove, analytics event rate-limiting.

#### Simple Implementation

```js
function throttle(fn, interval) {
  let lastTime = 0;
  return function (...args) {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn.apply(this, args);
    }
  };
}
```

**How it works:** Tracks the last execution time. Only allows the function to run if enough time (`interval`) has passed since the last call.

#### Usage

```js
// Scroll tracking — check every 200ms whether to load more
const throttledScroll = throttle(() => {
  const { scrollTop, scrollHeight, clientHeight } = document.documentElement;
  if (scrollTop + clientHeight > scrollHeight - 500) loadNextPage();
}, 200);
window.addEventListener('scroll', throttledScroll, { passive: true });

// Analytics — send viewport data at most once per second
const throttledTrack = throttle(() => {
  analytics.track('viewport', { sections: getVisibleSections() });
}, 1000);
window.addEventListener('scroll', throttledTrack, { passive: true });
```

### 4.4 Debounce vs Throttle, Decision Guide

```
                       ┌──────────────────────────┐
                       │ Do you need updates      │
                       │ DURING the burst of      │
                       │ events?                  │
                       └─────────┬────────────────┘
                                 │
                    ┌────────────┼────────────────┐
                    │ YES                          │ NO
                    ▼                              ▼
            ┌───────────────┐              ┌──────────────┐
            │   THROTTLE    │              │   DEBOUNCE   │
            │               │              │              │
            │ • scroll pos  │              │ • search     │
            │ • drag/drop   │              │ • auto-save  │
            │ • game loop   │              │ • resize end │
            │ • analytics   │              │ • validation │
            └───────────────┘              └──────────────┘
```

**How to read:** Ask yourself one question — "Do I need feedback WHILE the user is still scrolling/typing/dragging?" If YES → throttle. If NO (you only care about the final value) → debounce.

| | Debounce | Throttle |
|---|---|---|
| Fires | After silence period | At regular intervals |
| First call delayed? | Yes | No (fires immediately) |
| Total calls in burst | 1 | N / interval |
| Real-time feel | Delayed response | Immediate, periodic |
| Good for | Final value | Continuous tracking |

### 4.5 `requestIdleCallback`, Do Work When Browser Is Idle

`requestIdleCallback` schedules work during **browser idle periods** — the gaps between frames when the browser has nothing else to do.

```
Frame Timeline (targeting 60fps = 16.67ms per frame):

│◄── Frame 1 (16.67ms) ──►│◄── Frame 2 (16.67ms) ──►│
│                         │                         │
│ [JS] [Style] [Layout]   │ [JS] [Style]   [IDLE]   │
│ [Paint] [Composite]     │ [Layout] [Paint]        │
│                         │          ▲              │
│ No idle time here       │     rIC fires here      │
```

**How to read:** Each frame takes ~16.67ms to hit 60fps. Within each frame, the browser runs JS, calculates styles, does layout, paints. If all that finishes BEFORE 16.67ms, there's idle time left. `requestIdleCallback` uses that leftover time. If there's no idle time, it waits (or uses the `timeout` fallback).

**When to use:** Non-urgent work that shouldn't block user interaction — analytics, pre-fetching, lazy DOM updates.

#### Syntax

```js
requestIdleCallback((deadline) => {
  // deadline.timeRemaining() → ms left in this idle period
  while (deadline.timeRemaining() > 0) {
    doNextChunkOfWork();
  }
}, { timeout: 2000 }); // Force execution within 2s even if never idle
```

### 4.6 Scheduling API Comparison

```
 High Priority ──────────────────────────────► Low Priority

 ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐
 │  Microtasks  │  │    rAF       │  │     rIC              │
 │  (Promises)  │  │  (next frame │  │  (idle time, may be  │
 │              │  │   ~16ms)     │  │   delayed seconds)   │
 └──────────────┘  └──────────────┘  └──────────────────────┘
```

**How to read:** Left = highest priority (runs first). Microtasks (Promises) run immediately after current JS. `requestAnimationFrame` runs before the next paint (~16ms). `requestIdleCallback` runs only when the browser is free.

| API | When It Fires | Use For |
|---|---|---|
| `queueMicrotask()` | After current JS, before next task | Promise-like work, state sync |
| `requestAnimationFrame` | Before next paint (~16ms) | Animations, visual updates |
| `setTimeout(fn, 0)` | Next task (≥4ms) | Yielding to event loop |
| `requestIdleCallback` | Browser idle time | Analytics, prefetch, non-urgent DOM |

---

## 5. Custom Event Bus / Pub Sub for Decoupled Communication

### 5.1 What is Pub/Sub?

**Pub/Sub (Publish-Subscribe)** is a messaging pattern where:
- **Publishers** emit events without knowing who listens
- **Subscribers** register interest in events without knowing who emits them
- An **Event Bus** is the middleman that routes events from publishers to subscribers

This is the same pattern used by:
- **Node.js EventEmitter** (backend)
- **Redux** (actions dispatched → reducers listen)
- **DOM Events** (element dispatches → listeners catch)
- **WebSocket** message routing
- **Micro-frontends** communication

### 5.2 The Problem: Tight Coupling

Without Pub/Sub, components must import and call each other directly:

```
Without Pub/Sub — Spaghetti dependencies:

  ┌────────┐     ┌────────┐     ┌────────┐
  │ Header │────►│  Cart  │────►│ Toast  │
  │        │◄────│        │◄────│        │
  └───┬────┘     └───┬────┘     └───┬────┘
      │              │              │
      └──────────────┼──────────────┘
                     │
                ┌────┴────┐
                │ Product │
                │  Page   │
                └─────────┘

Every component imports and calls every other component.
Adding a new consumer means editing the producer.
Removing a component breaks all components that depend on it.
```

**Why this is bad:**
- **Adding a feature** (e.g., "show toast on cart add") requires editing the Cart component
- **Removing a component** (e.g., Toast) breaks Cart and Header
- **Testing** requires mocking all dependencies
- Components cannot be reused in other apps without their dependencies

### 5.3 The Solution: Pub/Sub (Event Bus)

```
With Pub/Sub — Decoupled communication:

  ┌────────┐  ┌────────┐  ┌────────┐  ┌──────────┐
  │ Header │  │  Cart  │  │ Toast  │  │ Product  │
  └───┬────┘  └───┬────┘  └───┬────┘  └────┬─────┘
      │           │           │            │
  subscribe   subscribe   subscribe     publish
      │           │           │             │
      └───────────┴───────────┴─────────────┘
                       │
               ┌───────┴───────┐
               │   Event Bus   │
               │ ───────────── │
               │ 'cart:add'    │
               │ 'cart:remove' │
               │ 'auth:login'  │
               │ 'theme:change'│
               └───────────────┘

Components know about EVENTS, not about each other.
```

**How to read this diagram:** The Product page publishes `'cart:add'` to the Event Bus. The Event Bus delivers it to all subscribers — Header (updates count), Cart (adds item), Toast (shows message). No component imports any other component. The Event Bus is the ONLY dependency.

**Why this is better:**
- **Adding a feature** = just subscribe a new listener. No existing code changes.
- **Removing a component** = just remove its subscription. Nothing else breaks.
- **Testing** = publish a fake event and check the response. No mocking needed.
- **Reusability** = components only depend on the event bus, not on each other.

### 5.4 How Pub/Sub Works Internally

The Event Bus is essentially a **map of event names → arrays of callback functions**:

```
Internal data structure:

  listeners = {
    'cart:add':     [fn1, fn2, fn3],   ← 3 subscribers
    'auth:login':   [fn4],             ← 1 subscriber
    'theme:change': [fn5, fn6],        ← 2 subscribers
  }

When emit('cart:add', data) is called:
  → Loop through [fn1, fn2, fn3] and call each with data
```

**Key operations:**
1. **`on(event, fn)`** — Push `fn` into the array for `event`
2. **`emit(event, data)`** — Loop through all functions for `event` and call them with `data`
3. **`off(event, fn)`** — Remove `fn` from the array for `event`

### 5.5 Minimal Implementation (JavaScript)

```js
class EventBus {
  constructor() {
    this.listeners = {};
  }

  // Subscribe — returns an unsubscribe function
  on(event, handler) {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event].push(handler);
    return () => this.off(event, handler);  // cleanup function
  }

  // Subscribe once — auto-removes after first call
  once(event, handler) {
    const wrapper = (data) => {
      this.off(event, wrapper);
      handler(data);
    };
    return this.on(event, wrapper);
  }

  // Unsubscribe
  off(event, handler) {
    if (!this.listeners[event]) return;
    this.listeners[event] = this.listeners[event].filter(h => h !== handler);
  }

  // Publish
  emit(event, data) {
    (this.listeners[event] || []).forEach(handler => {
      try { handler(data); }
      catch (err) { console.error(`EventBus error [${event}]:`, err); }
    });
  }

  // Clear all listeners for an event (or all events)
  clear(event) {
    if (event) delete this.listeners[event];
    else this.listeners = {};
  }
}

// Create a singleton for app-wide use
const eventBus = new EventBus();
```

### 5.6 Usage Pattern

```js
// --- In ProductCard component ---
function handleAddToCart(product) {
  eventBus.emit('cart:add', { productId: product.id, quantity: 1 });
}

// --- In CartIcon component ---
const unsubscribe = eventBus.on('cart:add', ({ quantity }) => {
  cartCount += quantity;
  updateCartBadge(cartCount);
});
// Call unsubscribe() on component destroy to prevent memory leaks

// --- In ToastNotification component ---
eventBus.on('cart:add', ({ productId }) => {
  showToast(`Item added to cart!`);
});

// --- In Analytics module ---
eventBus.on('cart:add', (data) => {
  analytics.track('product_added', data);
});
```

### 5.7 React Integration

```jsx
function useEventBus(event, handler) {
  const handlerRef = useRef(handler);
  handlerRef.current = handler;

  useEffect(() => {
    const unsubscribe = eventBus.on(event, (data) => handlerRef.current(data));
    return unsubscribe; // auto-cleanup on unmount
  }, [event]);
}

// Usage
function CartIcon() {
  const [count, setCount] = useState(0);
  useEventBus('cart:add', ({ quantity }) => setCount(prev => prev + quantity));
  useEventBus('cart:clear', () => setCount(0));
  return <span>🛒 {count}</span>;
}
```

### 5.8 Native Browser APIs for Pub/Sub

You don't always need a custom EventBus. The browser provides built-in pub/sub mechanisms:

#### `CustomEvent`, DOM Based Pub/Sub

```js
// Publish
document.dispatchEvent(new CustomEvent('app:theme-change', {
  detail: { theme: 'dark' },
  bubbles: true,
}));

// Subscribe
document.addEventListener('app:theme-change', (e) => {
  applyTheme(e.detail.theme);
});
```

**When to use:** Web Components, communicating between components that share a DOM tree.

#### `BroadcastChannel`, Cross Tab Pub/Sub

```js
// Tab 1: Listen
const channel = new BroadcastChannel('app-events');
channel.addEventListener('message', (e) => {
  if (e.data.type === 'auth:logout') window.location.href = '/login';
});

// Tab 2: Send to all other tabs
const channel2 = new BroadcastChannel('app-events');
channel2.postMessage({ type: 'auth:logout', reason: 'session_expired' });
```

**When to use:** Syncing auth state, cart, or theme across multiple tabs of the same origin.

### 5.9 When to Use What

| Pattern | Scope | Best For |
|---|---|---|
| **In-memory EventBus** | Single page / SPA | Cross-component communication |
| **CustomEvent on DOM** | DOM tree (bubbles) | Web Component communication |
| **BroadcastChannel** | Cross-tab (same origin) | Auth sync, cart sync across tabs |
| **`window.postMessage`** | Cross-origin / iframes | Embedded widgets, micro-frontends |
| **`storage` event** | Cross-tab (same origin) | Simple cross-tab sync via localStorage |
| **`MessageChannel`** | Two specific endpoints | Worker ↔ Worker, port-based |

### 5.10 Common Pitfalls

| Pitfall | Solution |
|---|---|
| **Memory leaks** — subscribers never removed | Always call `unsubscribe()` on component destroy |
| **Too many events** — hard to track data flow | Use namespaced events (`cart:add`, `auth:login`) and centralize event definitions |
| **Silent failures** — typos in event names | Define event names as constants in one file |
| **Ordering issues** — subscriber added after publisher fires | Use `once()` + replay/cache pattern for initialization events |
| **Debugging difficulty** — "where did this event come from?" | Add a `debug: true` flag that logs all events with stack traces |

---

## 6. AbortController for Cancellable Requests

### 6.1 The Problem: Stale and Zombie Requests

```
User types "re"    → fetch("/search?q=re")     ─── takes 500ms ──┐
User types "rea"   → fetch("/search?q=rea")    ─── takes 200ms ──┼── arrives FIRST  ✓
User types "react" → fetch("/search?q=react")  ─── takes 100ms ──┼── arrives SECOND
                                                                   │
                                                 "re" result ─────┘── arrives LAST!
                                                 Overwrites "react" result! 🐛
```

**How to read this diagram:** The user types "re", "rea", then "react" quickly. Three requests go out. Due to network timing, they come back OUT OF ORDER. The slowest request ("re") arrives last and overwrites the correct "react" result. This is a **race condition**.

**Without cancellation you get:**
- **Race conditions** — old responses overwrite new ones (as shown above)
- **Wasted bandwidth** — requests that are no longer needed still consume network
- **Memory leaks** — unmounted components receive callbacks and try to update destroyed UI
- **Unnecessary server load** — server processes requests nobody will use

### 6.2 What is AbortController?

`AbortController` is a native browser API that lets you **cancel** ongoing async operations — `fetch` requests, event listeners, streams, or any custom async work.

It has two parts:
- **`controller.abort()`** — call this to trigger cancellation
- **`controller.signal`** — pass this to the operation you want to make cancellable

```
┌──────────────────────────────────────────────┐
│                AbortController               │
│                                              │
│  ┌──────────────┐    ┌────────────────────┐  │
│  │   .abort()   │───►│   AbortSignal      │  │
│  │  (trigger)   │    │   .aborted = true  │  │
│  └──────────────┘    │   fires 'abort'    │  │
│                      │   event            │  │
│                      └────────┬───────────┘  │
└───────────────────────────────┼──────────────┘
                                │
           signal passed to:    │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                  ▼
        ┌──────────┐   ┌──────────────┐   ┌────────────┐
        │  fetch() │   │addEventListener│   │ Any custom │
        │  cancels │   │  auto-removes │   │ async work │
        │  request │   │  listener     │   │            │
        └──────────┘   └──────────────┘   └────────────┘
```

**How to read:** You create a controller. You pass its `signal` to one or more operations. When you call `abort()`, the signal fires — and every operation watching that signal cancels itself automatically.

### 6.3 Basic Usage

```js
const controller = new AbortController();

// Pass signal to fetch
fetch('/api/data', { signal: controller.signal })
  .then(res => res.json())
  .then(data => renderData(data))
  .catch(err => {
    if (err.name === 'AbortError') return; // Intentional cancel — not an error
    throw err; // Real error — handle it
  });

// Later — cancel the request
controller.abort();
```

### 6.4 Pattern: Cancel Previous Request on New Input (Search)

This is the most common use case — cancel the old request every time the user types a new character:

```js
let controller = null;

async function search(query) {
  controller?.abort();                    // Cancel previous request
  controller = new AbortController();     // Create new controller

  try {
    const res = await fetch(`/api/search?q=${encodeURIComponent(query)}`, {
      signal: controller.signal,
    });
    return await res.json();
  } catch (err) {
    if (err.name === 'AbortError') return null; // Cancelled — ignore
    throw err;
  }
}

input.addEventListener('input', async (e) => {
  const results = await search(e.target.value);
  if (results) renderResults(results);
});
```

### 6.5 Built-in Timeouts

```js
// Abort if request takes longer than 5 seconds
fetch('/api/data', { signal: AbortSignal.timeout(5000) })
  .catch(err => {
    if (err.name === 'TimeoutError') showToast('Request timed out');
  });
```

### 6.6 Combining Signals, User Cancel + Timeout

```js
const userController = new AbortController();

fetch('/api/large-file', {
  signal: AbortSignal.any([
    userController.signal,        // User clicks "Cancel"
    AbortSignal.timeout(10000),   // 10-second timeout
  ]),
});

cancelButton.onclick = () => userController.abort();
```

### 6.7 Batch Cleanup of Event Listeners

One of the most powerful uses — remove many listeners with a single `abort()`:

```js
const controller = new AbortController();
const { signal } = controller;

// Register multiple listeners with the same signal
window.addEventListener('resize', handleResize, { signal });
window.addEventListener('scroll', handleScroll, { signal });
document.addEventListener('keydown', handleKeydown, { signal });
document.addEventListener('click', handleClick, { signal });

// ONE call removes ALL four listeners
controller.abort();
```

This is especially clean in React:

```jsx
useEffect(() => {
  const controller = new AbortController();
  const { signal } = controller;

  window.addEventListener('resize', handleResize, { signal });
  window.addEventListener('scroll', handleScroll, { signal });

  return () => controller.abort(); // cleanup removes everything
}, []);
```

### 6.8 App Level AbortController Strategy

In a real application, you should have a **centralized approach** to request cancellation rather than managing individual controllers everywhere.

#### Strategy 1: Route Level Controller

Cancel all in-flight requests when the user navigates to a new page:

```js
// In your router/navigation handler
let routeController = new AbortController();

function onRouteChange(newRoute) {
  routeController.abort();                 // Cancel ALL requests from previous page
  routeController = new AbortController(); // Fresh controller for new page
  renderRoute(newRoute, routeController.signal);
}

// Every fetch in the app uses the route signal
async function fetchUserData(signal) {
  const res = await fetch('/api/user', { signal });
  return res.json();
}
```

#### Strategy 2: Fetch Wrapper with Auto Cancellation

Create a wrapper around `fetch` that your entire app uses:

```js
function createFetchClient() {
  const pendingRequests = new Map();

  return {
    async request(key, url, options = {}) {
      // Cancel previous request with the same key
      pendingRequests.get(key)?.abort();

      const controller = new AbortController();
      pendingRequests.set(key, controller);

      try {
        const res = await fetch(url, { ...options, signal: controller.signal });
        return await res.json();
      } catch (err) {
        if (err.name === 'AbortError') return null;
        throw err;
      } finally {
        pendingRequests.delete(key);
      }
    },

    cancelAll() {
      pendingRequests.forEach(controller => controller.abort());
      pendingRequests.clear();
    },
  };
}

// Usage across the app
const api = createFetchClient();

// Same key = auto-cancels previous call
api.request('user-search', `/api/search?q=${query}`);
api.request('user-search', `/api/search?q=${newQuery}`); // cancels previous

// On logout or route change
api.cancelAll();
```

#### Strategy 3: React Hook for Any Component

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!url) return;
    const controller = new AbortController();
    setLoading(true);

    fetch(url, { signal: controller.signal })
      .then(res => res.json())
      .then(json => { setData(json); setLoading(false); })
      .catch(err => {
        if (err.name !== 'AbortError') { setError(err); setLoading(false); }
      });

    return () => controller.abort(); // cleanup on unmount or URL change
  }, [url]);

  return { data, loading, error };
}
```

### 6.9 Debounce + AbortController Together (Search Pattern)

The most common real-world pattern — debounce input AND cancel stale requests:

```js
function createSearch(renderResults, delay = 300) {
  let controller = null;
  let timerId = null;

  return function (query) {
    clearTimeout(timerId);      // Reset debounce timer
    controller?.abort();        // Cancel in-flight request

    if (!query.trim()) { renderResults([]); return; }

    timerId = setTimeout(async () => {
      controller = new AbortController();
      try {
        const res = await fetch(`/api/search?q=${encodeURIComponent(query)}`, {
          signal: controller.signal,
        });
        renderResults(await res.json());
      } catch (err) {
        if (err.name !== 'AbortError') console.error('Search failed:', err);
      }
    }, delay);
  };
}

const search = createSearch(renderResults, 300);
document.getElementById('search').addEventListener('input', (e) => search(e.target.value));
```

---

## 7. Memory Leaks, Detection, Diagnosis and Prevention

### 7.1 What is a Memory Leak?

A memory leak occurs when your application **allocates memory that is never released** back to the system. The garbage collector cannot free the memory because something still holds a reference to it, even though the data is no longer needed.

Over time, this causes:
- **Increasing memory usage** — the app gets slower and slower
- **Page crashes** — "Aw, Snap!" or "Out of Memory" errors
- **Poor user experience** — especially on long-running SPAs where users don't refresh frequently

### 7.2 Common Causes of Memory Leaks in Frontend

```
┌─────────────────────────────────────────────────────┐
│              COMMON MEMORY LEAK SOURCES             │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. FORGOTTEN EVENT LISTENERS                       │
│     Component unmounts but listener stays on        │
│     window/document, holding references to the      │
│     component's closure and all its variables.      │
│                                                     │
│  2. ORPHANED TIMERS                                 │
│     setInterval / setTimeout keeps running after    │
│     component is destroyed. The callback holds      │
│     references to component state.                  │
│                                                     │
│  3. DETACHED DOM NODES                              │
│     A DOM element is removed from the document      │
│     but a JavaScript variable still references      │
│     it — the element stays in memory.               │
│                                                     │
│  4. CLOSURES HOLDING LARGE DATA                     │
│     An event handler or callback closes over a      │
│     large array/object. Even after the function     │
│     is no longer called, the closure + data stay    │
│     in memory because something references          │
│     the function.                                   │
│                                                     │
│  5. GROWING COLLECTIONS                             │
│     Maps, Sets, arrays that grow but never shrink.  │
│     E.g., an event bus where listeners are added    │
│     but never removed.                              │
│                                                     │
│  6. UNCANCELLED FETCH / PROMISES                    │
│     A fetch resolves after component unmounts and   │
│     calls setState on a destroyed component.        │
│     The promise callback keeps the component        │
│     closure alive.                                  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 7.3 How to Detect Memory Leaks

#### Method 1: Chrome DevTools, Performance Monitor

1. Open DevTools → **Performance Monitor** (Ctrl+Shift+P → "Show Performance Monitor")
2. Watch the **JS Heap Size** graph while using your app
3. **Expected:** Memory goes up (work) then comes back down (GC collects)
4. **Leak:** Memory keeps going up and NEVER comes back down, even after GC

```
Normal behavior:                Memory leak:
 Memory                          Memory
  ▲                               ▲
  │    ╱╲   ╱╲   ╱╲               │        ╱
  │  ╱    ╲╱   ╲╱   ╲             │      ╱
  │╱                              │    ╱
  └──────────────────► Time       │  ╱
                                  │╱
  Goes up and comes               └──────────────────► Time
  back down = healthy             Only goes up = LEAK
```

#### Method 2: Chrome DevTools, Memory Heap Snapshots

This is the **most precise** method to identify exactly what is leaking.

**Steps:**
1. Open DevTools → **Memory** tab
2. Select **"Heap Snapshot"**
3. Take **Snapshot 1** (baseline)
4. Perform the action you suspect leaks (e.g., open/close a modal 10 times)
5. Click the **garbage bin icon** (force GC)
6. Take **Snapshot 2**
7. In Snapshot 2, change the dropdown from "Summary" to **"Comparison"** and compare with Snapshot 1
8. Look for objects with **high "Delta"** — these are objects that were created but never released

**What to look for:**
- **Detached HTMLDivElement** → DOM nodes removed from page but still referenced in JS
- **(closure)** → Functions that hold references to destroyed component scope
- **EventListener** → Listeners that were never removed
- Large arrays/objects that shouldn't still exist

#### Method 3: Chrome DevTools, Allocation Timeline

1. Memory tab → select **"Allocation instrumentation on timeline"**
2. Start recording → perform the suspicious action → stop
3. Look at blue bars — blue bars that **remain blue** (not turned gray by GC) are potential leaks
4. Click on persistent blue bars to see what objects were allocated and who retains them

#### Method 4: `performance.measureUserAgentSpecificMemory()` (Programmatic)

```js
// Programmatically check memory usage (requires cross-origin isolation)
async function checkMemory() {
  if (performance.measureUserAgentSpecificMemory) {
    const result = await performance.measureUserAgentSpecificMemory();
    console.log(`Total memory: ${(result.bytes / 1024 / 1024).toFixed(2)} MB`);
  }
}
```

#### Method 5: Monitor Listener Count

```js
// Quick check: are listeners growing over time?
function countListeners() {
  // Check your event bus
  console.log('Event bus listeners:', eventBus.listenerCount?.() || 'N/A');

  // Chrome-only: getEventListeners() in console
  // getEventListeners(document)  — shows all listeners on document
  // getEventListeners(window)    — shows all listeners on window
}
```

### 7.4 Leak Free Patterns

#### Pattern 1: Always Clean Up Listeners

```js
// ❌ LEAK: listener on window never removed
class LeakyWidget {
  constructor() {
    window.addEventListener('resize', this.onResize);
  }
  // No cleanup method!
}

// ✅ SAFE: AbortController removes all listeners
class SafeWidget {
  constructor() {
    this.controller = new AbortController();
    window.addEventListener('resize', this.onResize, { signal: this.controller.signal });
  }
  destroy() {
    this.controller.abort(); // removes ALL listeners
  }
}
```

#### Pattern 2: Always Clear Timers

```js
// ❌ LEAK: interval runs forever after component removed
const id = setInterval(pollServer, 5000);

// ✅ SAFE: clear on destroy
const id = setInterval(pollServer, 5000);
// On cleanup:
clearInterval(id);
```

#### Pattern 3: Always Unsubscribe from Event Bus

```js
// ❌ LEAK: subscription never removed
eventBus.on('data:update', handleUpdate);

// ✅ SAFE: store unsubscribe and call on destroy
const unsubscribe = eventBus.on('data:update', handleUpdate);
// On cleanup:
unsubscribe();
```

#### Pattern 4: Cancel Async Operations

```js
// ❌ LEAK: fetch callback updates destroyed component
fetch('/api/data').then(res => this.setState(res));

// ✅ SAFE: abort on cleanup
const controller = new AbortController();
fetch('/api/data', { signal: controller.signal }).then(/* ... */);
// On cleanup:
controller.abort();
```

### 7.5 React Specific Leak Prevention

In React, the `useEffect` cleanup function is your primary defense:

```jsx
useEffect(() => {
  const controller = new AbortController();
  const unsubscribe = eventBus.on('update', handleUpdate);
  const intervalId = setInterval(poll, 5000);

  fetch('/api/data', { signal: controller.signal });

  // Cleanup function — runs on unmount and before re-run
  return () => {
    controller.abort();    // Cancel fetch
    unsubscribe();         // Remove event bus listener
    clearInterval(intervalId); // Stop timer
  };
}, []);
```

### 7.6 Memory Leak Debugging Checklist

| Check | How |
|---|---|
| **Heap keeps growing?** | Performance Monitor → watch JS Heap Size |
| **What objects are leaking?** | Memory → Heap Snapshot → Comparison view |
| **Who is retaining them?** | Click leaked object → "Retainers" panel shows the reference chain |
| **Detached DOM nodes?** | Heap Snapshot → Filter "Detached" → shows orphaned elements |
| **Event listeners growing?** | Console: `getEventListeners(document)` / `getEventListeners(window)` |
| **Closures holding data?** | Heap Snapshot → look for `(closure)` entries with large retained size |
| **After navigation?** | Navigate away and back → take snapshots before/after → compare delta |

### 7.7 Performance Budget for Memory

| Metric | Budget | How to Stay Within |
|---|---|---|
| Event listeners on `window`/`document` | < 20 | Use delegation, group with AbortController |
| Total event bus subscriptions | < 200 | Track with `listenerCount()`, warn on threshold |
| Handlers per scroll event | < 3 throttled | Combine into single throttled handler |
| Time per event handler | < 5ms | Offload to `requestIdleCallback` or Worker |
| JS Heap growth per navigation | ~0 (should return to baseline) | Heap snapshot comparison after navigating away |

---

## 8. Combining Patterns, Real World Architectures

### 8.1 E Commerce: Cart Events + Delegation + Abort

```
┌──────────────────────────────────────────────────────┐
│  Product Listing Page                                │
│                                                      │
│  ┌─────────────────────────────────────────────────┐ │
│  │ <div id="products">  ◄── ONE delegated listener │ │
│  │   ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐       │ │
│  │   │Product│ │Product│ │Product│ │Product│       │ │
│  │   │  Card │ │  Card │ │  Card │ │  Card │       │ │
│  │   │ [Add] │ │ [Add] │ │ [Add] │ │ [Add] │       │ │
│  │   └───────┘ └───────┘ └───────┘ └───────┘       │ │
│  └─────────────────┬───────────────────────────────┘ │
│                    │ click event bubbles up          │
│                    ▼                                 │
│            eventBus.emit('cart:add')                 │
│                    │                                 │
│     ┌──────────────┼──────────────┐                  │
│     ▼              ▼              ▼                  │
│  CartIcon      Toast          Analytics              │
│  (updates      (shows         (tracks event          │
│   count)        message)       throttled)            │
└──────────────────────────────────────────────────────┘
```

**How to read:** One delegated listener on the product container catches ALL button clicks. It publishes a `'cart:add'` event to the event bus. Three independent subscribers react — CartIcon updates the count, Toast shows a notification, Analytics tracks it (throttled to avoid excess). No component knows about any other component.

```js
// 1. Delegated click handler
document.getElementById('products').addEventListener('click', (e) => {
  const btn = e.target.closest('[data-action="add-to-cart"]');
  if (!btn) return;
  eventBus.emit('cart:add', { productId: btn.dataset.productId });
});

// 2. Subscribers (each in their own module)
eventBus.on('cart:add', ({ productId }) => cartService.addItem(productId));
eventBus.on('cart:add', () => toast.show('Added to cart!'));
eventBus.on('cart:add', throttle((data) => analytics.track('cart_add', data), 1000));
```

### 8.2 Auto Save Form: Debounce + Abort + Event Bus

```js
let controller = null;

const debouncedSave = debounce(async (formData) => {
  controller?.abort();
  controller = new AbortController();
  eventBus.emit('form:status', { status: 'saving' });

  try {
    await fetch('/api/drafts', {
      method: 'PUT',
      body: JSON.stringify(formData),
      signal: controller.signal,
    });
    eventBus.emit('form:status', { status: 'saved' });
  } catch (err) {
    if (err.name !== 'AbortError') {
      eventBus.emit('form:status', { status: 'error' });
    }
  }
}, 1000);

form.addEventListener('input', () => {
  const data = Object.fromEntries(new FormData(form));
  eventBus.emit('form:status', { status: 'unsaved' });
  debouncedSave(data);
});
```

---

## 9. Comparison Tables

### Event Rate Limiting Techniques

| Feature | Debounce | Throttle | requestIdleCallback | requestAnimationFrame |
|---|---|---|---|---|
| **Fires when** | After silence period | At regular intervals | Browser is idle | Before next paint |
| **First call delayed** | Yes | No | Yes | Next frame |
| **Use case** | Search, auto-save | Scroll, resize | Analytics, prefetch | Animations |
| **Cancel** | `clearTimeout` | `clearTimeout` | `cancelIdleCallback()` | `cancelAnimationFrame()` |

### Communication Patterns

| Pattern | Coupling | Scope | Best For |
|---|---|---|---|
| Direct function call | Tight | Same module | Simple parent→child |
| Props/callbacks (React) | Moderate | Component tree | Parent↔child |
| Context/Store (Redux) | Moderate | App-wide | Shared state |
| Event Bus / Pub-Sub | Loose | App-wide | Cross-module events |
| BroadcastChannel | Loose | Cross-tab | Tab sync |

---

## 10. Interview Questions and Answers

### Q1: What is event delegation and why is it useful?

**A:** Event delegation uses **one listener on a parent** instead of individual listeners on each child. It leverages **event bubbling** — when a child is clicked, the event bubbles up to the parent where the handler uses `event.target.closest()` to identify the target.

Benefits: **memory efficiency** (one listener, not thousands), **handles dynamic content** (new children auto-covered), **simpler cleanup** (one listener to remove).

---

### Q2: `event.target` vs `event.currentTarget`?

**A:**
- `event.target` → the **actual element** clicked (the innermost one)
- `event.currentTarget` → the **element the listener is attached to**

In delegation, `currentTarget` is the parent, `target` is the child.

---

### Q3: Debounce vs Throttle?

**A:**
- **Debounce**: Only care about the **final** value after activity stops (search input, auto-save)
- **Throttle**: Need **periodic updates during** continuous activity (scroll, drag, analytics)

Key: Debounce = zero calls during burst (waiting for silence). Throttle = regular calls at intervals.

---

### Q4: How does AbortController prevent race conditions?

**A:** Each new request calls `abort()` on the previous controller. The aborted fetch throws `AbortError` which is caught and ignored. Only the **latest** request's response is processed.

```js
let controller;
async function search(query) {
  controller?.abort();
  controller = new AbortController();
  try {
    const res = await fetch(`/search?q=${query}`, { signal: controller.signal });
    return await res.json();
  } catch (e) {
    if (e.name === 'AbortError') return null;
    throw e;
  }
}
```

---

### Q5: Why `{ passive: true }` on scroll/touch?

**A:** Without it, the browser must **wait** for the handler to finish before scrolling (handler might call `preventDefault()`). With `passive: true`, you promise not to call `preventDefault()`, so the browser scrolls **immediately** while the handler runs in parallel.

---

### Q6: How would you detect a memory leak?

**A:**
1. **Performance Monitor** — watch JS Heap Size. Healthy = up and down. Leak = only goes up.
2. **Heap Snapshots** — take snapshot before/after an action. Compare view shows objects that were created but never released.
3. **Retainers panel** — click a leaked object to see the reference chain holding it in memory.
4. **Filter "Detached"** — finds DOM nodes removed from page but still referenced in JS.
5. **`getEventListeners()`** — Chrome console command to count listeners on an element.

---

### Q7: Design a pub/sub system that prevents memory leaks.

**A:**
1. Return **unsubscribe functions** from `on()` — callers clean up on destroy
2. Set a **`maxListeners` limit** — warn when exceeded (like Node.js EventEmitter)
3. Use **AbortController signals** for DOM listeners — one `abort()` cleanups everything
4. In React, use a `useEventBus` hook that auto-unsubscribes on unmount
5. Add `clear()` method to remove all listeners for an event

---

### Q8: What is `requestIdleCallback`?

**A:** Schedules work during browser **idle periods** — gaps between frames when the browser has no rendering or input to process. Use for non-urgent work: analytics, pre-fetching, lazy computation. Provides `deadline.timeRemaining()` to check available time. Always use `timeout` option as a safety net.

---

### Q9: How does React handle events differently from vanilla DOM?

**A:**
- **Synthetic Events** — cross-browser wrappers around native events
- **Delegation at root** — single listener on root container (React 17+)
- Event names normalized (`onClick` vs `onclick`)
- `onFocus`/`onBlur` use `focusin`/`focusout` (which bubble)

---

### Q10: Real-time dashboard with 100+ WebSocket messages/sec — how?

**A:**
1. **Throttle DOM updates** — batch messages, update UI at most every 100-200ms via `requestAnimationFrame`
2. **Pub/sub layer** between WebSocket and UI — components subscribe to relevant events only
3. **`requestIdleCallback`** for non-critical updates (logging, analytics)
4. **Virtual list** for message history — only render visible items
5. **Web Worker** for data processing — aggregate in worker, post computed results to main thread

---

## Summary, Key Takeaways

| Concept | One-Liner |
|---|---|
| **Event Delegation** | One listener on parent + `closest()` to find target — scales to 100K+ items |
| **Passive Listeners** | `{ passive: true }` on scroll/touch = instant scrolling, no jank |
| **Debounce** | Wait until silence — search, auto-save, resize end |
| **Throttle** | Rate-limit — scroll tracking, drag, analytics |
| **requestIdleCallback** | Non-urgent background work during browser idle time |
| **Event Bus / Pub-Sub** | Decouple producers from consumers — one-to-many communication |
| **AbortController** | Cancel fetch, listeners, streams — prevent race conditions & leaks |
| **Memory Leaks** | Always clean up listeners, timers, subscriptions. Detect with heap snapshots. |
| **Cleanup** | AbortController.abort() + returned unsubscribe functions |
