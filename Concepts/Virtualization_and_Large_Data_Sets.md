# Virtualization & Handling Large Data Sets

> Render only what's visible — lists, tables, infinite scroll, canvas grids, and architecture patterns.

---

<a id="top"></a>

## Table of Contents

- [Why Virtualization](#why-virtualization)
- [How Virtualization Works](#how-virtualization-works)
- [Library Comparison and Decision Guide](#library-comparison-and-decision-guide)
- [Infinite Scrolling Patterns](#infinite-scrolling-patterns)
- [Large Table Rendering](#large-table-rendering)
- [Canvas Based Rendering (Extreme Data)](#canvas-based-rendering-extreme-data)
- [Performance Optimization](#performance-optimization)
- [Accessibility (A11y)](#accessibility-a11y)
- [Architecture Decision Guide](#architecture-decision-guide)
- [Real World Case Studies](#real-world-case-studies)
- [Common Pitfalls](#common-pitfalls)
- [Interview Questions](#interview-questions)
- [Quick Reference Card](#quick-reference-card)
[⬆ Back to Top](#top)

---

## Why Virtualization

Modern web applications routinely deal with large data sets — product catalogs with 50K items, analytics dashboards with 100K rows, chat logs spanning years. The natural instinct is to render everything into the DOM and let the browser handle it. But the browser's rendering pipeline was never designed for tens of thousands of nodes simultaneously. Every DOM node passes through **style calculation → layout → paint → composite** on every frame. When you have 50,000 nodes, the browser spends more time on off-screen elements than on what the user actually sees. Virtualization exists to solve this fundamental mismatch between data size and rendering capacity.

### The Problem — DOM Is Expensive

```
10,000 rows × ~5 DOM nodes each = 50,000 DOM nodes

Each node costs:
  ├── Memory:  ~0.5–1 KB per node
  ├── Layout:  Browser calculates position for ALL nodes
  ├── Paint:   Renders ALL visible + off-screen nodes
  └── GC:      Tracks all references

Result: 50 MB+ memory, 2–5s initial render, scroll jank, input lag
```

### Without vs With Virtualization

```
Without Virtualization              With Virtualization
┌──────────────┐                    ┌──────────────┐
│ Row 1        │ ← rendered         │              │ ← padding-top (spacer)
│ Row 2        │ ← rendered         │╔════════════╗│
│ Row 3        │ ← rendered         │║ Row 5      ║│ ← rendered (visible)
│ ...          │                    │║ Row 6      ║│ ← rendered (visible)
│ Row 9999     │ ← rendered         │║ Row 7      ║│ ← rendered (visible)
│ Row 10K      │ ← rendered         │╚════════════╝│
└──────────────┘                    │              │ ← padding-bottom (spacer)
                                    └──────────────┘
DOM: 10,000 nodes                   DOM: ~10 nodes + spacers
Memory: O(n)                        Memory: O(viewport) — constant!
```

### Performance Numbers

| Rows | DOM Nodes | Render Time | Scroll FPS | Memory |
|------|-----------|-------------|------------|--------|
| 100 | 500 | ~20ms | 60 fps | ~5 MB |
| 10,000 | 50,000 | ~2,000ms | 20-30 fps | ~80 MB |
| 100,000 | 500,000 | ~15,000ms+ | < 10 fps | ~500 MB+ |
| **100K virtualized** | **~500** | **~50ms** | **60 fps** | **~10 MB** |

> **Key Insight:** Virtualization keeps DOM node count near-constant regardless of data size.

[⬆ Back to Top](#top)

---

## How Virtualization Works

Virtualization (also called **windowing**) is the technique of **only rendering elements currently visible** in the viewport, plus a small overscan buffer, while maintaining the **illusion** of a full scrollable list. The browser's native scrollbar still behaves as if all items exist — the trick is that a tall "phantom" container (set to the total height of all items) creates the scrollbar, but actual DOM nodes are only created for the ~20–30 items the user can see.

This is conceptually similar to how a movie projector works: the audience sees a continuous stream of images, but only one frame is displayed at any given moment. The rest of the film reel exists but isn't being projected.

### Core Math (Fixed Height)

The core algorithm is surprisingly simple for fixed-height items. Since every item has the same height, we can compute the exact position of any item with basic arithmetic — no measurement needed. The virtualizer listens to the `scroll` event, reads `scrollTop`, divides by `itemHeight` to find which item the user is looking at, then renders only a small window around that index.

```
Given:
  containerHeight = 600px,  itemHeight = 40px
  totalItems = 50,000,      scrollTop = 8,000px,  overscan = 5

Calculate:
  totalHeight = 50,000 × 40 = 2,000,000px       ← virtual scroll height
  startIndex  = floor(8000 / 40) = 200
  visibleCount = ceil(600 / 40) = 15
  endIndex    = 200 + 15 = 215

With overscan:
  renderStart = max(0, 200 - 5) = 195
  renderEnd   = min(50000, 215 + 5) = 220

→ Render items[195..220] = 25 DOM nodes instead of 50,000
```

### Fixed vs Variable Height

Fixed-height virtualization is straightforward because `offset = index × height` gives O(1) random access to any item's position. **Variable-height** items are significantly harder. Since each item can be a different height (think tweets with images vs text-only), you can't use simple multiplication. Instead, you must maintain a **prefix sum array** — a running total of heights — so that `offset[i] = sum of heights from item 0 to i-1`. Finding which item is at a given scroll position then requires a **binary search** on this prefix sum array, giving O(log n) lookup.

The additional challenge with variable heights is that you often **don't know the height until the item renders**. Libraries solve this with an **estimate-then-measure** approach: provide an `estimateSize` for initial layout, render the items, measure their actual DOM height via `getBoundingClientRect()` or `ResizeObserver`, then correct the offsets.

```
Fixed Height:                       Variable Height:
┌────────────────┐  40px           ┌────────────────┐  30px
├────────────────┤  40px           ├────────────────┤  80px  ← expanded
├────────────────┤  40px           ├────────────────┤  120px ← has image
├────────────────┤  40px           ├────────────────┤  30px

offset = index × height            offset = prefixSum[index]
O(1) lookup                         O(log n) binary search on prefix sums
```

### Architecture of a Virtualizer

Every virtualizer has the same fundamental structure: an **outer scroll container** (with `overflow: auto`) that detects scroll events, an **inner container** sized to the total virtual height of all items (creating the scrollbar), and a small set of **rendered items** positioned within that inner container. The space above and below rendered items is filled by spacers (padding, empty divs, or CSS transforms) to maintain the correct scroll position.

```
┌───────────────────────────────────────────────────────┐
│                  Scroll Container                      │
│  ┌─────────────────────────────────────────────────┐  │
│  │         Inner Container (total height)           │  │
│  │                                                  │  │
│  │  ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐      │  │
│  │    Spacer Top (padding / transform)              │  │
│  │  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘      │  │
│  │  ┌─────────────────────────────────────────┐    │  │
│  │  │  Rendered Item (startIndex)             │    │  │
│  │  │  Rendered Item (startIndex + 1)         │    │  │
│  │  │  ...                                    │    │  │
│  │  │  Rendered Item (endIndex)               │    │  │
│  │  └─────────────────────────────────────────┘    │  │
│  │  ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐      │  │
│  │    Spacer Bottom                                 │  │
│  │  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘      │  │
│  └─────────────────────────────────────────────────┘  │
│                   ▲ scrollTop                          │
└───────────────────────────────────────────────────────┘
```

### Three Positioning Strategies

Once the virtualizer knows which items to render, it needs to **position them at the correct scroll offset**. There are three approaches, each with different performance characteristics. The key insight is about **which phase of the rendering pipeline** each approach triggers:

- **Layout phase** (expensive): Browser recalculates geometry of all affected elements
- **Paint phase** (moderate): Browser redraws pixels for affected areas
- **Composite phase** (cheap): GPU moves already-painted layers — no CPU recalculation

`transform: translateY()` wins because it only triggers compositing — the browser promotes the element to its own GPU layer and just repositions it, skipping layout and paint entirely.

| Strategy | How | Used By | Why |
|----------|-----|---------|-----|
| **Padding** | `paddingTop` / `paddingBottom` on inner container | Simple custom builds | Easy, but triggers layout on change |
| **Absolute** | `position: absolute; top: offset` per item | react-window | Precise, each item independently placed |
| **Transform** | `translateY(offset)` on wrapper | react-virtuoso | **GPU-accelerated**, no layout — only compositing |

```
Positioning comparison:

Padding:                  Absolute:                  Transform:
┌────────────┐           ┌────────────┐             ┌────────────┐
│ paddingTop │           │ relative   │             │ relative   │
│╔══════════╗│           │   abs top:0│             │ translateY │
│║ items    ║│           │   abs top:X│             │  ┌──────┐  │
│╚══════════╝│           │   abs top:Y│             │  │items │  │
│ paddingBot │           │            │             │  └──────┘  │
└────────────┘           └────────────┘             └────────────┘
Triggers layout          Triggers layout            GPU compositing only ✅
```

### Core Principles Summary

| Principle | What It Means | Why It Matters |
|-----------|---------------|----------------|
| **Viewport-only rendering** | Only items visible (± overscan) exist in DOM | Keeps DOM node count constant O(viewport), regardless of data size — eliminates the root cause of slowness |
| **Spacer/padding trick** | Total scroll height preserved using padding or a sentinel div | Creates the native scrollbar illusion — browser thinks all items are rendered, user sees correct scroll thumb size and position |
| **Scroll sync** | `scrollTop / itemHeight` → data index mapping | Connects the physical scroll position to the logical data index — this is the core calculation that drives which items get rendered |
| **Overscan** | Extra items rendered above/below visible area | Prevents blank flashes during fast scrolling — items are pre-rendered just off-screen so they appear instantly when scrolled into view |
| **Recycling** | Some implementations reuse DOM nodes instead of creating/destroying | Reduces garbage collection pressure — instead of creating 20 new nodes and destroying 20 old ones per scroll, the same nodes get new content |

[⬆ Back to Top](#top)

---

## Library Comparison and Decision Guide

Choosing a virtualization library is one of the most impactful architectural decisions for data-heavy UIs. The core trade-off is between **simplicity** (less control, more opinions) and **flexibility** (more control, more boilerplate). There are two fundamental API philosophies:

- **Component-based** (react-window, react-virtuoso): You render a `<FixedSizeList>` or `<Virtuoso>` component that owns the scroll container, the positioning logic, and the rendering. You just provide an `itemContent` renderer. Less code, but the library controls the DOM structure.
- **Headless** (TanStack Virtual): The library gives you a hook (`useVirtualizer`) that returns mathematical calculations — which items are visible, what their offsets are, total height. You own 100% of the markup and styling. More boilerplate, but total flexibility.

The right choice depends on your constraints: Do you need to match a custom design system? Use headless. Do you need grouped headers and infinite scroll working in 10 minutes? Use batteries-included.

### Library Evolution

```
react-virtualized (2015) ← Legacy, large bundle, feature-rich
       │
react-window (2018)      ← Lighter rewrite by same author
       │
@tanstack/react-virtual  ← Framework-agnostic, headless, modern
       │
react-virtuoso (2019+)   ← Zero-config, dynamic heights, batteries-included
```

### Comparison Table

| Feature | react-window | react-virtuoso | TanStack Virtual |
|---------|:----------:|:-----------:|:-------------:|
| **Bundle** | ~6 KB | ~16 KB | ~4 KB |
| **API Style** | Component | Component | Headless hook |
| **Fixed-size items** | ✅ | ✅ | ✅ |
| **Variable-size** | ⚠️ must know sizes | ✅ auto-measured | ✅ via `measureElement` |
| **Infinite scroll** | Via addon | ✅ built-in | Manual |
| **Sticky headers** | ❌ | ✅ built-in | Manual |
| **Table support** | Grid component | ✅ `TableVirtuoso` | Manual (headless) |
| **Horizontal** | ✅ Grid | ❌ | ✅ |
| **Bidirectional** (chat) | ❌ | ✅ | Manual |
| **SSR** | ⚠️ Limited | ✅ | ✅ |
| **Frameworks** | React only | React only | React, Vue, Solid, Svelte |
| **Maintained (2024+)** | Low | Active | Very active |

### Decision Flowchart

```
Need list virtualization?
│
├─ Fixed-height items, simple list?
│  └─ react-window ✅ (smallest, simplest)
│
├─ Dynamic heights, zero-config needed?
│  ├─ Need infinite scroll, grouped headers, tables?
│  │  └─ react-virtuoso ✅ (batteries included)
│  └─ Just simple dynamic list?
│     └─ react-virtuoso or TanStack Virtual
│
├─ Need full markup/styling control?
│  └─ @tanstack/react-virtual ✅ (headless)
│
├─ Not using React? (Vue, Solid, Svelte)
│  └─ @tanstack/virtual ✅ (framework-agnostic)
│
├─ Custom table with column virtualization?
│  └─ TanStack Virtual + TanStack Table ✅
│
└─ Chat UI with bidirectional scroll?
   └─ react-virtuoso ✅ (reversed mode)
```

### Usage Syntax At a Glance

**react-window** — Component-based, sizes upfront:

```jsx
<FixedSizeList height={600} itemCount={items.length} itemSize={50} overscanCount={5}>
  {({ index, style }) => <div style={style}>{items[index].name}</div>}
</FixedSizeList>
```

**react-virtuoso** — Zero-config, auto-measures:

```jsx
<Virtuoso
  style={{ height: '600px' }}
  totalCount={items.length}
  itemContent={(index) => <div>{items[index].name}</div>}
/>
```

**TanStack Virtual** — Headless hook, you own the markup:

```jsx
const virtualizer = useVirtualizer({
  count: items.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 50,
});

// You render: virtualizer.getVirtualItems().map(row => ...)
// You position: transform: `translateY(${row.start}px)`
```

[⬆ Back to Top](#top)

---

## Infinite Scrolling Patterns

Infinite scrolling is a UX pattern where new data loads automatically as the user scrolls, eliminating explicit "next page" buttons. But it introduces two fundamental challenges:

1. **When to fetch:** The app must detect that the user is approaching the end of loaded content and trigger a network request *before* they see blank space. Too late and users see a loading spinner; too early and you waste bandwidth.

2. **DOM growth**: Without virtualization, infinite scroll creates an ever-growing DOM. A user who scrolls through 500 pages has 25,000 DOM nodes in memory. Combining infinite scroll with virtualization solves this — the data grows in memory but the DOM stays constant.

The **gold standard** in production is: **cursor-based pagination** on the API + **`useInfiniteQuery`** for caching + **virtualization** for rendering. This gives you bounded DOM, efficient network usage, and a seamless user experience.

### Pattern Overview

```
Infinite Scroll Approaches:

1. Intersection Observer (Modern ✅ Recommended)
   └─ Sentinel element at bottom → fires callback when visible

2. Scroll Position (Legacy)
   └─ scrollTop + clientHeight >= scrollHeight - threshold

3. Virtualized + Infinite Scroll (Best for Large Lists ✅)
   └─ Combine virtualization + data fetching

4. Bidirectional (Chat UIs)
   └─ Load older messages at top, new at bottom
   └─ Maintain scroll position when prepending
```

### Pattern 1: Intersection Observer

The **Intersection Observer API** is a browser-native mechanism for efficiently detecting when an element enters or exits the viewport (or any ancestor scroll container). Unlike scroll event listeners — which fire synchronously on every pixel of scroll and require manual math to determine element visibility — Intersection Observer is **asynchronous** and **batched** by the browser. The browser itself determines when elements cross visibility thresholds during its rendering cycle, making it significantly cheaper than polling scroll positions.

The pattern uses a **sentinel element** — a tiny 1px `div` placed after the last item. The observer watches this sentinel. The `rootMargin` property extends the detection zone (e.g., `200px` below the viewport), so the fetch triggers *before* the user reaches the end — giving the network request time to complete before the user sees empty space.

```
How it works:

  ┌─────────────────────────┐
  │     Scroll Container     │
  │  ┌───────────────────┐  │
  │  │   Item 1          │  │
  │  │   ...             │  │  Visible Viewport
  │  │   Item 20         │  │
  │  └───────────────────┘  │
  │  ╔═══════════════════╗  │ ← rootMargin: 200px (pre-trigger zone)
  │  ╚═══════════════════╝  │
  │  ┌ ─ ─ ─ ─ ─ ─ ─ ─ ┐  │
  │    Sentinel (1px div)   │ ← When enters rootMargin → fetchMore()
  │  └ ─ ─ ─ ─ ─ ─ ─ ─ ┘  │
  └─────────────────────────┘
```

**Approach:** Watch a 1px sentinel div. When it enters rootMargin zone, trigger fetch.

```jsx
// Hook: useInfiniteScroll
const observer = new IntersectionObserver(
  ([entry]) => { if (entry.isIntersecting) fetchMore(); },
  { rootMargin: '200px', threshold: 0 }
);
observer.observe(sentinelRef.current);

// Component: Just place sentinel after items
{items.map(item => <div key={item.id}>{item.name}</div>)}
<div ref={sentinelRef} style={{ height: 1 }} />  {/* ← trigger */}
```

> **Why better than scroll events?** No event listener overhead, no manual scroll math, built-in thresholding, more performant.

### Pattern 2: Virtualized + Infinite Scroll (Gold Standard)

**Approach:** Virtualization for DOM + cursor-based pagination for data.

```
Data Flow:

  API (cursor-paginated)
       ↓
  useInfiniteQuery({ queryKey, queryFn, getNextPageParam })
       ↓
  allItems = pages.flatMap(p => p.data)
       ↓
  Virtualizer (only renders ~20-30 visible items)
       ↓
  When last virtual item visible → fetchNextPage()
```

**react-virtuoso** — Built-in:

```jsx
<Virtuoso
  data={items}
  endReached={loadMore}   // ← fires when scrolled near bottom
  itemContent={(index, item) => <Item data={item} />}
  components={{ Footer: () => loading ? <Spinner /> : null }}
/>
```

**TanStack Virtual + TanStack Query** — Manual wiring:

```jsx
const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({ ... });
const allItems = data?.pages.flatMap(p => p.data) ?? [];

const virtualizer = useVirtualizer({
  count: hasNextPage ? allItems.length + 1 : allItems.length, // +1 loader row
  ...
});

// In useEffect: when last virtualItem.index >= allItems.length → fetchNextPage()
```

### Pattern 3: Bidirectional Scroll (Chat UIs)

Bidirectional scrolling is the hardest infinite scroll pattern. In a chat app, old messages load at the **top** when the user scrolls up, while new messages appear at the **bottom** in real-time. The critical challenge is **scroll position stability**: when you prepend 50 older messages above the current viewport, the browser naturally shifts everything down by the combined height of those new items, causing the viewport to "jump" — the message the user was reading disappears.

The solution is the **`firstItemIndex` pattern**: instead of array indices starting at 0, you start with a large synthetic index (like 10,000). When older messages are prepended, you decrement `firstItemIndex`. The virtualizer uses this to compute stable offsets, effectively "inserting" items above the viewport without shifting visible content.

```
Challenge — Prepending messages without scroll jump:

  Before prepend:               After prepend (WRONG):
  ┌─────────────────┐          ┌─────────────────┐
  │  Message 50     │ ← here   │  Message -49 ▲  │ ← viewport jumps!
  │  Message 51     │          │  Message -48    │
  └─────────────────┘          └─────────────────┘

  After prepend (CORRECT — scroll position preserved):
  ┌─────────────────┐
  │  Message -49     │ ← added above (off-screen)
  │  ...             │
  │  Message 50      │ ← still visible ✅
  └─────────────────┘
```

**Approach:** Use `firstItemIndex` pattern — start with large index, decrement on prepend:

```jsx
<Virtuoso
  data={messages}
  firstItemIndex={firstItemIndex}           // Start at 10000, decrement on prepend
  initialTopMostItemIndex={messages.length - 1}  // Start at bottom
  startReached={loadOlderMessages}          // Called when scrolled to top
  followOutput="smooth"                     // Auto-scroll on new messages
/>
```

### Cursor-Based vs Offset-Based Pagination

How you paginate your API directly affects the reliability of infinite scroll. **Offset-based** (`?offset=200&limit=50`) is simple but fragile — if items are inserted or deleted between requests, items get skipped or duplicated because the offset now points to a different position. **Cursor-based** (`?cursor=abc123&limit=50`) uses an opaque pointer (typically the last item's ID or a timestamp) to resume exactly where the previous page left off, regardless of insertions or deletions. For any real-time feed (social, chat, notifications), cursor-based is essential.

| Aspect | Cursor-Based | Offset-Based |
|--------|:----------:|:----------:|
| **Consistency** | ✅ Stable with real-time data | ❌ Skips/duplicates on insert/delete |
| **Performance** | ✅ O(1) seek with index | ❌ O(n) for large offsets |
| **Jump to page** | ❌ Not possible | ✅ Easy |
| **Use when** | Infinite scroll, feeds | Paginated tables |

```
Cursor: GET /items?cursor=abc123&limit=50  → { data, nextCursor }
Offset: GET /items?offset=200&limit=50     → { data, total }
```

[⬆ Back to Top](#top)

---

## Large Table Rendering

Tables amplify the virtualization problem because data is **two-dimensional**. A list with 100K items has one axis to virtualize; a table with 100K rows × 50 columns has **two axes**. The cell count grows multiplicatively, not additively. This means you often need **both** a row virtualizer (vertical, based on `scrollTop`) and a column virtualizer (horizontal, based on `scrollLeft`) working simultaneously, rendering only the rectangular intersection of visible rows and visible columns.

Additionally, tables have unique UX requirements: **sticky headers** must remain visible while scrolling vertically, **sticky columns** (like an ID or name column) must remain visible while scrolling horizontally, and the **corner cell** where both sticky axes meet needs the highest `z-index` so it sits above everything.

### The Challenge — 2D Virtualization

```
100,000 rows × 50 columns = 5,000,000 cells!

With Row Virtualization only (20 visible rows):
  20 × 50 = 1,000 cells → manageable

With Row + Column Virtualization (20 rows × 10 cols):
  200 cells → fast! ✅
```

### Row Virtualization

```
┌──────────────────────────────────────────────────────┐
│  Col A    │  Col B    │  Col C    │  ...  │  Col Z   │
├───────────┼───────────┼───────────┼───────┼──────────┤
│  (padding-top: from hidden rows above)               │
├───────────┼───────────┼───────────┼───────┼──────────┤
│  Row 45   │  data     │  data     │  ...  │  data    │ ← Rendered
│  Row 46   │  data     │  data     │  ...  │  data    │ ← Rendered
│  ...      │  ...      │  ...      │  ...  │  ...     │
│  Row 64   │  data     │  data     │  ...  │  data    │ ← Rendered
├───────────┼───────────┼───────────┼───────┼──────────┤
│  (padding-bottom: from hidden rows below)            │
└──────────────────────────────────────────────────────┘
```

### Column Virtualization

```
Full table: │ Col 1 │ Col 2 │ Col 3 │ ··· │ Col 50 │

Viewport (columns 5–15 visible):
              ┌─────────────────────────────┐
  padding-left│ Col 5 │ Col 6 │ ··· │Col 15│ padding-right
              └─────────────────────────────┘
                        ▲
                  scrollLeft
```

### Approach: TanStack Table + TanStack Virtual

The modern standard for building production virtualized tables is to separate **table logic** from **rendering logic**. **TanStack Table** handles column definitions, sorting state, filtering, row selection, column resizing — all the *data operations*. **TanStack Virtual** handles *which rows and columns are visible* given the current scroll position. Neither library renders any DOM — they both return data/calculations that you render yourself. This separation gives you full control over markup, styling, and behavior while letting each library focus on what it does best.

```
Architecture:

  TanStack Table → column defs, sorting, filtering, selection
       ↓
  rows = table.getRowModel().rows
       ↓
  Row Virtualizer  → which rows to render (based on scrollTop)
  Col Virtualizer  → which cols to render (based on scrollLeft)
       ↓
  Render intersection: visible rows × visible columns
  + padding cells for unrendered columns
  + spacer rows for unrendered rows
```

```jsx
// Two virtualizers working together
const rowVirtualizer = useVirtualizer({
  count: rows.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 40,
});

const colVirtualizer = useVirtualizer({
  horizontal: true,
  count: visibleColumns.length,
  getScrollElement: () => parentRef.current,
  estimateSize: (i) => visibleColumns[i].getSize(),
});

// Render: only intersection of virtualRows × virtualCols
// + paddingLeft/paddingRight cells for hidden columns
// + spacer <tr> for hidden rows above/below
```

### Sticky Headers & Columns

Sticky positioning is one of CSS's most powerful features for data grids. `position: sticky` creates an element that behaves like `relative` positioning until it reaches a scroll threshold, then switches to behaving like `fixed` — all without JavaScript. The key implementation detail is **z-index layering**: the sticky header needs `z-index: 2` (above scrolling rows), sticky columns need `z-index: 1`, and the corner cell (where sticky header meets sticky column) needs `z-index: 3` so it sits above both. All sticky elements **must have an opaque background** — otherwise scrolling content bleeds through underneath them.

```
Normal scroll:                     Sticky header:
┌─────────────────┐               ┌─────────────────┐
│  Header Row     │ ← scrolls up  │  Header Row     │ ← stays fixed ✅
├─────────────────┤               ├─────────────────┤
│  Row 1          │               │  Row 50         │
│  Row 2          │               │  Row 51         │
└─────────────────┘               └─────────────────┘

Sticky column (horizontal scroll):
┌──────────┬──────────┬──────────┬──────────┐
│ Name     │ Col 5    │ Col 6    │ Col 7    │
│ (sticky) │          │          │          │  ← Col 1-4 scrolled out
├──────────┼──────────┼──────────┼──────────┤
│ Alice    │ data     │ data     │ data     │
│ Bob      │ data     │ data     │ data     │
└──────────┴──────────┴──────────┴──────────┘
  ▲ stays fixed
```

**CSS approach:**

```css
/* Sticky header */
thead th { position: sticky; top: 0; z-index: 2; background: white; }

/* Sticky first column */
td:first-child, th:first-child { position: sticky; left: 0; z-index: 1; background: white; }

/* Corner cell (both axes) */
thead th:first-child { z-index: 3; }  /* Above both sticky header & column */
```

[⬆ Back to Top](#top)

---

## Canvas Based Rendering (Extreme Data)

When even virtualized DOM can't keep up — millions of cells, real-time streaming updates, complex conditional formatting — **canvas rendering** bypasses the DOM entirely. Instead of creating DOM nodes that the browser must manage through its rendering pipeline (style → layout → paint → composite), you draw pixels directly onto a `<canvas>` element using its 2D drawing API (`fillRect`, `fillText`, `strokeRect`).

The trade-off is fundamental: you gain **raw speed** (the browser doesn't track individual cells, no layout recalculation, no style resolution) but you lose **everything the DOM provides for free** — text selection, accessibility, event delegation, CSS styling, form elements, and SEO. You must manually implement hit-testing ("which cell did the user click?"), keyboard navigation, clipboard operations, and accessibility overlays. This is why canvas grids are reserved for extreme cases where DOM virtualization is genuinely insufficient.

### When DOM vs Canvas

```
DOM Rendering (Virtualized)         Canvas Rendering
├── < 100K cells                    ├── > 100K cells
├── Rich interactions (forms)       ├── Display-heavy (read-only)
├── A11y required                   ├── Real-time data (trading, monitoring)
├── SEO needed                      ├── High update frequency
└── Standard table features         └── GPU-accelerated drawing
```

### How Canvas Grids Work

A canvas grid treats the entire visible area as a single `<canvas>` element and **draws cell content as pixels** using the Canvas 2D API (`ctx.fillText`, `ctx.fillRect`, `ctx.strokeRect`). On scroll, the grid reads the new scroll position, calculates which rows/columns are visible (the same math as DOM virtualization), clears the canvas, and redraws only the visible cells in a single `requestAnimationFrame` callback.

The key architectural challenges are: **event handling** (canvas receives a single click event with `clientX`/`clientY` coordinates that must be translated to `(row, col)` via math), **text editing** (an invisible `<input>` element is overlaid on the canvas, positioned over the active cell), and **accessibility** (an ARIA grid overlay with visually hidden DOM nodes must mirror the visible canvas content for screen readers).

```
┌──────────────────────────────────────────┐
│          Canvas Grid Architecture         │
│                                           │
│  ┌────────────────────────────────────┐  │
│  │          <canvas>                   │  │
│  │   Painted pixels (NOT DOM nodes)    │  │
│  │   ┌─────┬─────┬─────┬─────┐       │  │
│  │   │ A1  │ B1  │ C1  │ D1  │       │  │
│  │   ├─────┼─────┼─────┼─────┤       │  │
│  │   │ A2  │ B2  │ C2  │ D2  │       │  │
│  │   └─────┴─────┴─────┴─────┘       │  │
│  └────────────────────────────────────┘  │
│                                           │
│  Events: Manual hit-testing               │
│    (clientX, clientY) → (row, col)        │
│  Scroll: requestAnimationFrame repaint    │
│  Edit:   Overlay <input> over cell        │
└──────────────────────────────────────────┘
```

**Approach:** Calculate visible range from scroll position → draw only visible cells on canvas → manual hit-testing for interactions → overlay native `<input>` for editing.

### Canvas Libraries

| Library | Use Case | Scale |
|---------|----------|-------|
| **Glide Data Grid** | React canvas grid | 1M+ cells |
| **AG Grid** (Enterprise) | Hybrid DOM/Canvas | Enterprise apps |
| **Handsontable** | Excel-like grid | Spreadsheets |
| **PixiJS / Konva** | Custom visualizations | Custom |

### Canvas vs DOM Trade-offs

| Aspect | DOM (Virtualized) | Canvas |
|--------|:------------------:|:------:|
| **Max cells** | ~100K | Millions |
| **Accessibility** | Native | Must implement manually |
| **Text selection** | Native | Manual clipboard |
| **Rich content** | Any HTML/CSS | Must paint everything |
| **Event handling** | DOM events | Manual hit testing |
| **Complexity** | Low-Medium | High |

[⬆ Back to Top](#top)

---

## Performance Optimization

Virtualization alone gets you 80% of the way to a smooth experience, but the remaining 20% comes from **fine-tuning the rendering pipeline**. The core issue: even though the virtualizer limits DOM nodes to ~20–30, each scroll event can still trigger re-renders, layout recalculations, and garbage collection. These micro-costs accumulate at 60fps (one frame every 16.6ms), and a single slow frame causes visible jank.

The optimization strategy is to minimize work at every level: (1) reduce how often React re-renders (memoization), (2) reduce what the browser recalculates per frame (CSS containment, transform positioning), (3) reduce GC pressure (stable keys, avoid inline allocations), and (4) move expensive work off the main thread (Web Workers for sorting/filtering).

### Overscan Tuning

**Overscan** is the number of extra items rendered above and below the visible area. It's a direct trade-off: more overscan means fewer blank flashes during fast scrolling (items are pre-rendered off-screen), but too much overscan defeats the purpose of virtualization by keeping too many DOM nodes alive. The sweet spot depends on row complexity — simple text rows need less overscan than rows with images and interactive controls.

```
Too little (0):              Right (3-10):              Too much (50):
┌──────────────┐            ┌──────────────┐           ┌──────────────┐
│              │ ← blank!   │▒▒▒▒▒▒▒▒▒▒▒▒│ ← buffer  │▒▒▒▒▒▒▒▒▒▒▒▒│
│╔════════════╗│            │╔════════════╗│            │▒▒▒▒▒▒▒▒▒▒▒▒│
│║  Visible   ║│            │║  Visible   ║│            │╔════════════╗│
│╚════════════╝│            │╚════════════╝│            │╚════════════╝│
│              │ ← blank!   │▒▒▒▒▒▒▒▒▒▒▒▒│ ← buffer  │▒▒▒▒▒▒▒▒▒▒▒▒│
└──────────────┘            └──────────────┘           └──────────────┘
Fast but flickers           Smooth scrolling ✅        Defeats purpose
```

| Scenario | Recommended Overscan |
|----------|---------------------|
| Simple text rows | 3–5 items |
| Complex row renderers | 5–10 items |
| Fast scroll (mobile) | 10–20 items or 200–400px |

### Key Optimization Techniques

These techniques are ordered by impact-to-effort ratio. The first few are simple CSS/React changes that provide large gains; the later ones require more effort but handle edge cases.

| Technique | Impact | Approach |
|-----------|--------|----------|
| `React.memo` on row components | High | Prevent re-renders on scroll |
| `useMemo` for data transforms | Medium | Avoid recomputing filtered/sorted data |
| `transform: translateY()` not `top` | Medium | GPU compositing, no layout trigger |
| `contain: layout style paint` | High | Isolate layout recalc per row |
| Reserve space for images | High | Set explicit `width`/`height` to avoid layout shift |
| `{ passive: true }` scroll listeners | Medium | Prevent scroll jank |
| `requestAnimationFrame` scroll batching | Medium | Batch scroll state updates |
| Web Worker for data processing | High | Offload filtering/sorting from main thread |
| `content-visibility: auto` | High | Browser-native lazy rendering (non-virtualized) |
| Avoid inline objects/functions in render | Medium | Prevents child re-renders |

### Scroll Performance Pattern

Scroll events fire at 60+ times per second. If each event calls `setState`, React schedules a re-render for every event — potentially 60+ re-renders per second. `requestAnimationFrame` batching ensures only **one state update per visual frame**, matching the browser's actual repaint rate. Production libraries like TanStack Virtual handle this internally using `observeElementOffset`, so you typically don't need to implement this yourself.

```jsx
// ❌ Bad: setState on every scroll → re-render every frame
const handleScroll = (e) => setScrollTop(e.currentTarget.scrollTop);

// ✅ Better: Batch with requestAnimationFrame
const rafRef = useRef(null);
const handleScroll = useCallback((e) => {
  if (rafRef.current) cancelAnimationFrame(rafRef.current);
  rafRef.current = requestAnimationFrame(() => {
    setScrollTop(e.currentTarget.scrollTop);
  });
}, []);

// ✅ Best: Libraries (TanStack Virtual) handle this internally
```

[⬆ Back to Top](#top)

---

## Accessibility (A11y)

Virtualization creates a fundamental tension with accessibility. Screen readers rely on the DOM to understand content — they traverse DOM nodes to announce content, count items, and navigate. But virtualization **removes most items from the DOM** to improve performance. A screen reader visiting a virtualized list with 10,000 items would only "see" 20–30 DOM nodes and have no knowledge of the other 9,970+.

The solution is **ARIA attributes** that communicate the *logical structure* of the list (total count, each item's position) without requiring all items to be in the DOM. Combined with **keyboard navigation** that programmatically scrolls to items on arrow key presses, screen reader users can navigate the full list even though most items aren't rendered.

### Challenge

Screen readers can only see items currently in the DOM. Off-screen virtualized items are invisible.

### ARIA Attributes Approach

`aria-rowcount` on the container tells the screen reader the **total number of items** in the list, even though most aren't in the DOM. `aria-rowindex` on each rendered item tells the screen reader **where this item sits** in the full list (using 1-based indexing). Together, these attributes let the screen reader announce things like "Item 156 of 10,000" — giving the user full positional awareness despite only ~20 items existing in the DOM.

```
Container:
  role="list"
  aria-label="Search results"
  aria-rowcount={totalItems}           ← total count

Each item:
  role="listitem"
  aria-rowindex={index + 1}            ← 1-based position
  aria-setsize={totalItems}            ← total count
  aria-posinset={index + 1}            ← position in set
```

### Keyboard Navigation

Keyboard navigation in virtualized lists requires extra work because the target item **might not exist in the DOM** yet. When the user presses Arrow Down to move to the next item, you must: (1) update the focused index in state, (2) tell the virtualizer to `scrollToIndex` so the target item gets rendered, and (3) set focus on the newly rendered DOM node. This three-step process — state update, scroll, focus — must happen in sequence, and the focus step often needs a `requestAnimationFrame` delay to wait for the item to actually render.

| Key | Action |
|-----|--------|
| `↓` Arrow Down | Move focus to next item + `scrollToIndex` |
| `↑` Arrow Up | Move focus to prev item + `scrollToIndex` |
| `Home` | Jump to first item |
| `End` | Jump to last item |
| `Page Up/Down` | Scroll by page |

### A11y Checklist

The key principle is: **tell the screen reader what it can't see**. Use ARIA to communicate the full logical structure, keyboard handlers to enable navigation, and live regions to announce dynamic changes.

| Requirement | How |
|-------------|-----|
| Total count announced | `aria-rowcount` on container |
| Item position | `aria-rowindex`, `aria-posinset`, `aria-setsize` |
| Keyboard navigation | Arrow keys + `scrollToIndex` |
| Focus management | `tabIndex`, scroll-to-focused |
| Loading states | `aria-live="polite"` region |
| Table semantics | `<table>`, `<thead>`, `role="grid"` |

[⬆ Back to Top](#top)

---

## Architecture Decision Guide

The biggest mistake teams make is reaching for virtualization too early (adding complexity to a 100-item list) or too late (discovering at scale that their non-virtualized table is unusable). The thresholds below are based on real-world measurements across typical React applications. Below ~200 items, the browser handles rendering efficiently and virtualization adds unnecessary complexity. Between 200-5K, simple virtualization provides a massive improvement. Beyond 5K, you need to combine virtualization with strategic data fetching. Beyond 100K in a table context, you need 2D virtualization. And at 1M+ cells, even virtualized DOM may not keep up — that's canvas territory.

### Choosing the Right Pattern

```
How many items?

  < 200 items
  └─ Plain rendering. No virtualization needed.
     (CSS content-visibility: auto can help)

  200 – 5,000 items
  └─ Simple virtualization
     ├─ Fixed heights → react-window
     └─ Dynamic heights → react-virtuoso

  5,000 – 100,000 items
  └─ Virtualization + infinite scroll
     ├─ react-virtuoso (built-in endReached)
     └─ TanStack Virtual + TanStack Query

  100,000+ items (tables)
  └─ Row + Column virtualization
     ├─ TanStack Table + TanStack Virtual
     └─ AG Grid, Glide Data Grid

  1,000,000+ cells (extreme)
  └─ Canvas-based rendering
     └─ Glide Data Grid, custom canvas
```

### Full Architecture for Data-Heavy App

The modern production stack for data-heavy applications follows a clear separation of concerns: **data fetching** (TanStack Query manages caching, pagination, and deduplication), **data operations** (TanStack Table handles sorting, filtering, column state), **rendering optimization** (TanStack Virtual calculates which items to render), and **DOM output** (your components render only the visible intersection). Each layer is independently testable and replaceable.

```
┌─────────────────────────────────────────────────────────┐
│                 Frontend Architecture                    │
│                                                          │
│  Search/Filter (debounced 300ms) + Sort Controls         │
│           │                                              │
│           ▼                                              │
│  ┌─────────────────────────────────────┐                │
│  │  TanStack Query (useInfiniteQuery)  │                │
│  │  Pages loaded on demand             │  Cursor-based  │
│  │  maxPages: 10 (memory limit)        │  pagination    │
│  └──────────────┬──────────────────────┘                │
│                 │ allItems = pages.flatMap(p => p.data)  │
│                 ▼                                        │
│  ┌─────────────────────────────────────┐                │
│  │  TanStack Table                     │                │
│  │  (columns, sorting, filtering)      │                │
│  └──────────────┬──────────────────────┘                │
│                 │ rows = table.getRowModel().rows        │
│                 ▼                                        │
│  ┌─────────────────────────────────────┐                │
│  │  TanStack Virtual                   │                │
│  │  Row virtualizer + Col virtualizer  │                │
│  │  Renders ~20-30 visible rows        │                │
│  └──────────────┬──────────────────────┘                │
│                 ▼                                        │
│  ┌─────────────────────────────────────┐                │
│  │  DOM: <table>                       │                │
│  │  Sticky <thead> (position: sticky)  │                │
│  │  Virtualized <tbody> (translateY)   │                │
│  └─────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────┘
```

[⬆ Back to Top](#top)

---

## Real World Case Studies

These case studies show how real products solve virtualization challenges. The common thread is that **each product hit a specific scale threshold** where naive DOM rendering failed, and their solution was tailored to their specific constraints — dynamic heights for feeds, bidirectional scroll for chat, canvas for real-time finance.

### Twitter/X Feed

```
Problem: Infinite feed with variable-height tweets
Solution:
  ├── Virtualized list (dynamic height measurement)
  ├── Intersection Observer for infinite loading
  ├── Cursor-based pagination (tweet ID as cursor)
  ├── Height pre-estimation + correction after paint
  └── Scroll position restoration on back-nav
```

### Google Sheets

```
Problem: 1M+ cells, real-time collaboration, complex formatting
Solution:
  ├── Canvas-based rendering (only way at 1M+ scale)
  ├── Overlay <input> for cell editing
  ├── Only repaints dirty regions (not full canvas)
  ├── Web Worker for formula calculation
  └── ARIA grid overlay for screen readers
```

### Slack Messages

```
Problem: Bidirectional scroll (old up, new down)
Solution:
  ├── react-virtuoso with reversed mode
  ├── firstItemIndex pattern for prepend stability
  ├── Auto-scroll on new messages (followOutput)
  └── "Jump to latest" button when scrolled up
```

### Bloomberg Terminal

```
Problem: Real-time streaming data, thousands of instruments
Solution:
  ├── Canvas grid (DOM can't handle per-second updates)
  ├── Cell-level dirty tracking (repaint only changed cells)
  ├── WebSocket for streaming prices
  ├── Double-buffering canvas (avoid flicker)
  └── requestAnimationFrame batching
```

[⬆ Back to Top](#top)

---

## Common Pitfalls

Virtualization looks straightforward in demos but has subtle failure modes in production. Most pitfalls fall into three categories: **measurement errors** (wrong heights cause scroll position drift), **React rendering issues** (unnecessary re-renders kill scroll performance), and **memory management** (data grows unbounded). Understanding these patterns saves hours of debugging.

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Wrong item heights | Scroll jumping, gaps | Accurate `estimateSize`, use `measureElement` |
| Array index as key | Items swap/flash on scroll | Use stable IDs (`item.id`) |
| Unbounded data growth | Memory grows over time | Page eviction, `maxPages` in TanStack Query |
| No overscan | Blank flashes on fast scroll | Set overscan to 5–10 |
| Inline functions in render | Unnecessary re-renders | `React.memo`, extract components |
| `overflow: hidden` parent | Scroll doesn't work | Ensure `overflow: auto` on scroll container |
| Dynamic content (images) | Layout shift after render | Reserve space with explicit dimensions |
| SSR mismatch | Hydration errors | `initialItemCount`, server render first N items |

### Memory Management for Infinite Scroll

Virtualization solves the **DOM problem** (only ~20 nodes in the DOM at any time) but doesn't solve the **data problem**. If a user scrolls through 500 pages of results, all 25,000 items are still in JavaScript memory even though only 20 are rendered. For most applications this is fine — JavaScript objects are lightweight compared to DOM nodes. But for very long sessions or memory-constrained devices, you need **page eviction**: keep only N pages of data in memory (e.g., `maxPages: 10` in TanStack Query), and re-fetch evicted pages if the user scrolls back to them. This adds complexity but bounds total memory usage.

```
Problem: items array grows unbounded as user scrolls

  Page 1:   50 items   → fine
  Page 50:  2,500      → acceptable
  Page 500: 25,000     → memory problem!

Solutions:
  1. Virtualization handles DOM ✅ (only ~20 nodes)
  2. Limit data in memory:
     - TanStack Query: maxPages: 10
     - Custom: keep pages [current-5, current+2], evict rest
     - Re-fetch evicted pages if user scrolls back
```

[⬆ Back to Top](#top)

---

## Interview Questions

Virtualization is a common system design interview topic because it tests multiple skills simultaneously: understanding of browser rendering (why the DOM is expensive), algorithmic thinking (prefix sums, binary search for variable heights), API design (cursor vs offset pagination), and architectural judgment (when to use what tool). The questions below progress from conceptual understanding to system design.

### Q1: What is virtualization and why is it needed?

**Virtualization** (windowing) = render only visible items + small overscan buffer. Needed because DOM is expensive — 10K+ nodes cause high memory, slow render, scroll jank. Keeps DOM count near-constant (~20–50 nodes) regardless of data size → O(1) render cost.

---

### Q2: How does a virtualizer calculate which items to render?

**Fixed height:** `startIndex = floor(scrollTop / itemHeight)`, `visibleCount = ceil(containerHeight / itemHeight)`, add overscan. Total height = `totalItems × itemHeight`.

**Variable height:** Maintain prefix sum array of heights. Binary search on prefix sum to find startIndex for given scrollTop. Measure items post-render, update sizes dynamically.

---

### Q3: How would you implement infinite scroll with virtualization?

1. Cursor-based API (`?cursor=abc&limit=50`)
2. `useInfiniteQuery` manages paginated cache
3. Flatten pages → single items array
4. Virtualizer renders visible items only
5. When last virtual item visible → `fetchNextPage()`
6. `maxPages` to limit memory

---

### Q4: How do you handle column virtualization?

Two separate virtualizers:
- **Row virtualizer** (vertical): which rows based on `scrollTop`
- **Column virtualizer** (horizontal): which cols based on `scrollLeft`

Render only the intersection. Padding cells for non-rendered columns. TanStack Table + TanStack Virtual is the standard approach.

---

### Q5: What are the accessibility challenges?

Screen readers can't see off-screen items. Solutions: `aria-rowcount` (total), `aria-rowindex` / `aria-posinset` (position), keyboard nav with `scrollToIndex`, `aria-live` for loading states, semantic `<table>` elements.

---

### Q6: When would you use canvas over DOM virtualization?

Canvas when: 1M+ cells, real-time streaming data, high update frequency, read-only display.
Avoid canvas when: rich interactions, accessibility required, SEO needed, team lacks canvas expertise.

---

### Q7: How do you prevent scroll position jumping?

1. Reserve space for media (explicit `width`/`height`)
2. Accurate `estimateSize` close to actual
3. Use `measureElement` to correct post-render
4. CSS `contain: layout style paint` on rows
5. Skeleton loading matching expected height

---

### Q8: Design a virtualized grid — 100K rows, 50 columns, sortable, filterable.

```
API (cursor-paginated) → TanStack Query (cache + fetch)
  ↓
TanStack Table (column defs, sorting, filtering)
  ↓
Row Virtualizer + Column Virtualizer (TanStack Virtual)
  ↓
DOM <table>: sticky <thead>, virtualized <tbody>

Key decisions:
  - Server-side sort/filter (100K too much for client)
  - Row virtualizer: ~20-30 visible rows
  - Col virtualizer: ~8-10 visible columns
  - Fixed row height (40px) for predictability
  - Sticky header + sticky first column
  - Debounced filter (300ms) → new query
  - maxPages for memory management
```

---

### Q9: How does "scroll to item" work with variable heights?

Virtualizer maintains offset map (prefix sum). To scroll to index N:
- `scrollTop = prefixSum[N]` (start align)
- Center: `scrollTop = prefixSum[N] - containerHeight/2 + itemHeight/2`
- If unmeasured, use estimates → render → measure → correct position

```
APIs:
  react-window:   listRef.current.scrollToItem(index, 'center')
  react-virtuoso: virtuosoRef.current.scrollToIndex({ index, align: 'center' })
  TanStack:       virtualizer.scrollToIndex(index, { align: 'center' })
```

---

### Q10: Compare `transform: translateY()` vs `position: absolute; top:`

| Aspect | `translateY()` | `absolute + top` |
|--------|:-----------:|:-------------:|
| **Rendering** | GPU compositing only | Triggers layout |
| **Performance** | Faster ✅ | Slower |
| **Used by** | react-virtuoso, TanStack Virtual | react-window |
| **Why better** | Skips layout & paint phases, only compositing step |

[⬆ Back to Top](#top)

---

## Quick Reference Card

```
Virtualization = render only visible items + overscan buffer
  ├── Math: startIndex = floor(scrollTop / itemHeight)
  ├── Position: translateY (GPU) > absolute top > padding
  └── DOM stays O(viewport), never O(n)

Libraries:
  react-window   → simple, fixed-size, ~6KB
  react-virtuoso → zero-config, dynamic, batteries, ~16KB
  TanStack Virtual → headless, any framework, ~4KB

Infinite Scroll:
  Intersection Observer + sentinel div (simple)
  Virtualized + useInfiniteQuery (production)
  Bidirectional + firstItemIndex (chat)

Tables:
  Row + Column virtualization for 2D
  TanStack Table (logic) + TanStack Virtual (rendering)
  Sticky: position: sticky + z-index layering

Canvas: 1M+ cells, real-time, read-only heavy
When NOT to virtualize: < 200 items
```

[⬆ Back to Top](#top)

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design

[⬆ Back to Top](#top)
