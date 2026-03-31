# Frontend System Design: Instagram Stories

- [Frontend System Design: Instagram Stories](#frontend-system-design-instagram-stories)
  - [1. Concept, Idea, Product Overview](#1-concept-idea-product-overview)
    - [1.1 Product Description](#11-product-description)
    - [1.2 Key User Personas](#12-key-user-personas)
    - [1.3 Core User Flows (High Level)](#13-core-user-flows-high-level)
  - [2. Requirements](#2-requirements)
    - [2.1 Functional Requirements](#21-functional-requirements)
    - [2.2 Non Functional Requirements](#22-non-functional-requirements)
  - [3. Scope Clarification (Interview Scoping)](#3-scope-clarification-interview-scoping)
    - [3.1 In Scope](#31-in-scope)
    - [3.2 Out of Scope](#32-out-of-scope)
    - [3.3 Assumptions](#33-assumptions)
  - [4. High Level Frontend Architecture](#4-high-level-frontend-architecture)
    - [4.1 Overall Approach](#41-overall-approach)
    - [4.2 Major Architectural Layers](#42-major-architectural-layers)
    - [4.3 External Integrations](#43-external-integrations)
  - [5. Component Design and Modularization](#5-component-design-and-modularization)
    - [5.1 Component Hierarchy](#51-component-hierarchy)
    - [5.2 Horizontal Story Tray Scrolling Strategy and Pattern (Deep Dive)](#52-horizontal-story-tray-scrolling-strategy-and-pattern-deep-dive)
      - [5.2.1 Base CSS Approach](#521-base-css-approach)
      - [5.2.2 Virtualized Horizontal List](#522-virtualized-horizontal-list)
      - [5.2.3 Desktop Arrow Navigation](#523-desktop-arrow-navigation)
      - [5.2.4 Tray Ordering Unseen First, Seen Pushed to Back](#524-tray-ordering-unseen-first-seen-pushed-to-back)
      - [5.2.5 RTL Language Support](#525-rtl-language-support)
      - [5.2.6 Lazy Loading Avatars in the Tray](#526-lazy-loading-avatars-in-the-tray)
      - [5.2.7 Scroll Performance Considerations](#527-scroll-performance-considerations)
      - [5.2.8 Decision Matrix](#528-decision-matrix)
    - [5.3 Reusability Strategy](#53-reusability-strategy)
    - [5.4 Module Organization](#54-module-organization)
  - [6. High Level Data Flow Explanation](#6-high-level-data-flow-explanation)
    - [6.1 Initial Load Flow](#61-initial-load-flow)
    - [6.2 User Interaction Flow](#62-user-interaction-flow)
    - [6.3 Error and Retry Flow](#63-error-and-retry-flow)
  - [7. Data Modelling (Frontend Perspective)](#7-data-modelling-frontend-perspective)
    - [7.1 Core Data Entities](#71-core-data-entities)
    - [7.2 Data Shape](#72-data-shape)
    - [7.3 Entity Relationships](#73-entity-relationships)
    - [7.4 UI Specific Data Models](#74-ui-specific-data-models)
  - [8. State Management Strategy](#8-state-management-strategy)
    - [8.1 State Classification](#81-state-classification)
    - [8.2 State Ownership](#82-state-ownership)
    - [8.3 Persistence Strategy](#83-persistence-strategy)
  - [9. High Level API Design (Frontend POV)](#9-high-level-api-design-frontend-pov)
    - [9.1 Required APIs](#91-required-apis)
    - [9.2 Two Phase Fetching Strategy](#92-two-phase-fetching-strategy)
    - [9.3 Request and Response Structure](#93-request-and-response-structure)
    - [9.4 Error Handling and Status Codes](#94-error-handling-and-status-codes)
  - [10. Caching Strategy](#10-caching-strategy)
  - [11. CDN and Asset Optimization](#11-cdn-and-asset-optimization)
  - [12. Rendering Strategy](#12-rendering-strategy)
  - [13. Cross Cutting Non Functional Concerns](#13-cross-cutting-non-functional-concerns)
    - [13.1 Security](#131-security)
    - [13.2 Accessibility](#132-accessibility)
    - [13.3 Performance Optimization](#133-performance-optimization)
    - [13.4 Observability and Reliability](#134-observability-and-reliability)
  - [14. Edge Cases and Tradeoffs](#14-edge-cases-and-tradeoffs)
  - [15. Summary and Future Improvements](#15-summary-and-future-improvements)

---

## 1. Concept, Idea, Product Overview

### 1.1 Product Description

*   Instagram Stories is a feature that lets users post ephemeral, full-screen media (photos, videos, text) that auto-expire after **24 hours**.
*   Target users: all Instagram users — creators, casual users, and businesses.
*   Primary use case: consuming and creating short-lived, immersive media in a horizontally swipeable, auto-advancing viewer.

---

### 1.2 Key User Personas

*   **Viewer**: Browses the story tray, taps into stories, navigates between slides and users, reacts/replies.
*   **Creator**: Captures or uploads photos/videos, applies text/filters, and publishes a story.
*   **Business / Influencer**: Uses stories with links and analytics to engage their audience.

---

### 1.3 Core User Flows (High Level)

*   **Viewing Stories (Primary Flow)**:
    1. User sees the story tray (horizontal ring list) at the top of the feed.
    2. Taps on a user's avatar → full-screen story viewer opens.
    3. Story slides auto-advance on a timer (5s for images, video duration for videos).
    4. Tap right → next slide; Tap left → previous slide; tap and hold → pause.
    5. Auto-advance to next user when all slides are consumed.
    6. Reply input → send a reply or reaction.
    7. When all stories are consumed → viewer closes or returns to feed.

*   **Creating Stories (Secondary Flow)**:
    1. User taps the "+" icon on their avatar in the story tray.
    2. Camera/gallery opens → capture or select media.
    3. Apply text, filters, mentions.
    4. Publish → story appears in followers' trays.

---

## 2. Requirements

### 2.1 Functional Requirements

*   **Story Tray**:
    *   Display horizontal list of users who have active (unseen) stories, sorted by recency / relevance.
    *   Current user's own story (or "Add" button) pinned at the start.
    *   Visual ring indicator: gradient ring = unseen, grey ring = seen.
*   **Story Viewer**:
    *   Full-screen immersive overlay.
    *   Auto-advance with a segmented progress bar at the top.
    *   Tap-based navigation between slides and users.
    *   Pause on long-press or when reply drawer is open.
    *   Support image, video, and text-only story types.
*   **Interactions**:
    *   Reply to a story (text message).
    *   Quick emoji reaction.
*   **Story Creation**:
    *   Capture photo/video or pick from gallery.
    *   Add text, filters.
    *   Publish to "My Story" or "Close Friends".
*   **Seen / View Tracking**:
    *   Mark stories as seen locally and report to server.
    *   Story owner can view the list of viewers.

---

### 2.2 Non Functional Requirements

*   **Performance**: Story tray visible within FCP < 1.5s; story viewer opens in < 300ms; media preloaded for zero-wait transitions.
*   **Scalability**: Support users following thousands of accounts, each potentially having stories; tray can contain 100+ avatars.
*   **Availability**: Graceful degradation — show cached stories if network is unavailable.
*   **Security**: Media served over authenticated CDN URLs with short-lived tokens; no client-side URL leaking.
*   **Accessibility**: Full keyboard & screen-reader navigation for tray and viewer; pause controls; captions on video.
*   **Device Support**: Mobile web (primary), desktop web (responsive), low-end devices.
*   **i18n**: RTL layout support; localized timestamps ("2h ago").

---

## 3. Scope Clarification (Interview Scoping)

### 3.1 In Scope

*   Story tray component (horizontal list with unseen/seen state).
*   Story viewer (full-screen, auto-advance, navigation, progress bar).
*   Media preloading and playback strategy.
*   State management for seen/unseen tracking.
*   API design for fetching stories and reporting views.
*   Performance optimization for media-heavy experience.

---

### 3.2 Out of Scope

*   Story creation / camera / editor UI (rich editing is a separate deep-dive).
*   Backend ranking algorithm for story ordering.
*   Ads in stories.
*   Instagram Highlights (archived stories).
*   Push notifications for new stories.

---

### 3.3 Assumptions

*   User is authenticated; auth token is available.
*   APIs return stories grouped by user, ordered by platform ranking.
*   Media (images/videos) are served from a CDN with pre-signed URLs.
*   The story feature is embedded as a section within a larger feed page (not a standalone app).

---

## 4. High Level Frontend Architecture

### 4.1 Overall Approach

*   **SPA** with the story viewer rendered as a full-screen **overlay/portal** on top of the feed.
*   **CSR** for the story viewer (client-side rendering) — it's an interactive, media-heavy overlay that doesn't benefit from SSR.
*   **SSR / SSG** for the feed page shell (tray placeholder rendered server-side for fast FCP).
*   Story module is **code-split** and lazily loaded when the user taps into a story.

---

### 4.2 Major Architectural Layers

```
┌──────────────────────────────────────────────────────┐
│  UI Layer                                            │
│  ┌─────────────┐   ┌──────────────────────────────┐  │
│  │ Story Tray  │   │ Story Viewer (Portal/Overlay)│  │
│  │ (Avatars +  │   │  ┌──────────┐ ┌───────────┐  │  │
│  │  Ring UI)   │   │  │ Progress │ │  Media    │  │  │
│  └─────────────┘   │  │ Bar      │ │  Renderer │  │  │
│                    │  └──────────┘ └───────────┘  │  │
│                    │         ┌──────────┐         │  │
│                    │         │ Reply    │         │  │
│                    │         │ Drawer   │         │  │
│                    │         └──────────┘         │  │
│                    └──────────────────────────────┘  │
├─────────────────────────────────────────────────────┤
│  State Management Layer                             │
│  (Stories Store, Seen State, Active Viewer State)   │
├─────────────────────────────────────────────────────┤
│  API & Data Access Layer                            │
│  (Story APIs, View Reporting, Media Preloader)      │
├─────────────────────────────────────────────────────┤
│  Shared / Utility Layer                             │
│  (Timer utils, Media loader,                        │
│   Intersection Observer, Analytics)                 │
└─────────────────────────────────────────────────────┘
```

---

### 4.3 External Integrations

*   **CDN**: Serves story images and videos (Cloudfront / Akamai).
*   **Analytics SDK**: Track story views, completion rates, interaction events.
*   **Media Player**: Native `<video>` with HLS/DASH for adaptive streaming.
*   **Backend Services**: Story feed API, view tracking API, reaction/reply API.

---

## 5. Component Design and Modularization

### 5.1 Component Hierarchy

```
FeedPage
 ├── StoryTray (horizontal scrollable container)
 │    ├── OwnStoryAvatar (pinned first — "Add" or "Your Story")
 │    └── StoryAvatarItem[] (scrollable list)
 │         ├── AvatarRing (gradient = unseen, grey = seen)
 │         └── Username
 │
 └── StoryViewer (Portal/Overlay — rendered on tap)
      ├── StoryProgressBar
      │    └── ProgressSegment[] (one per slide)
      ├── StoryHeader
      │    ├── UserAvatar + Username + Timestamp
      │    └── CloseButton / MoreMenu (mute, report)
      ├── StoryMediaRenderer
      │    ├── ImageSlide
      │    └── VideoSlide
      ├── StoryNavigationLayer (invisible tap zones: left/right)
      └── StoryReplyDrawer
           ├── ReplyInput
           └── EmojiReactionBar
```

---

### 5.2 Horizontal Story Tray Scrolling Strategy and Pattern (Deep Dive)

The story tray is the **first thing users see** at the top of the Instagram feed — a horizontally scrollable row of circular avatars. Getting this right matters because:
*   It's above the fold — any jank or slow render directly impacts **perceived performance (FCP/LCP)**.
*   It contains dynamic, user-specific data (seen/unseen states) that changes frequently.
*   It must work across mobile touch, desktop mouse/trackpad, and screen readers.
*   The number of items is unpredictable — could be 5 avatars or 500+.

---

#### 5.2.1 Base CSS Approach

**The Core Idea:**  
Instead of building a custom scroll engine in JavaScript, we rely on the browser's **native horizontal scroll** behavior. A flex container with `overflow-x: auto` makes its children scrollable when they exceed the container's width. The browser handles all touch gestures, trackpad physics, and scroll inertia natively — no JS needed.

**Why Flexbox (not CSS Grid)?**
*   Each avatar is a **same-size item flowing in one direction** — this is exactly what `flex-direction: row` is designed for.
*   `gap` provides consistent spacing without margin hacks.
*   Flex items naturally **don't wrap** (default `flex-wrap: nowrap`), which is what we want — a single scrollable row.
*   Grid would work but is overkill for a single-row layout and requires explicit column definitions.

```css
.story-tray {
  display: flex;
  flex-direction: row;
  flex-wrap: nowrap;             /* single row — items overflow instead of wrapping */
  overflow-x: auto;             /* enable horizontal scroll when content exceeds width */
  overflow-y: hidden;           /* prevent accidental vertical scroll */
  gap: 12px;                    /* consistent spacing between avatars */
  padding: 8px 16px;            /* breathing room on edges */

  /* ---- Smooth scroll behavior ---- */
  scroll-behavior: smooth;
  -webkit-overflow-scrolling: touch;  /* momentum-based scroll on iOS Safari */

  /* ---- Hidden scrollbar (aesthetic choice) ---- */
  scrollbar-width: none;              /* Firefox */
  -ms-overflow-style: none;           /* IE / legacy Edge */
}
.story-tray::-webkit-scrollbar {
  display: none;                      /* Chrome, Safari, newer Edge */
}
```

```html
<div class="story-tray" role="list">
  <div class="story-avatar-item" role="listitem"><!-- own story --></div>
  <div class="story-avatar-item" role="listitem"><!-- user 1 --></div>
  <div class="story-avatar-item" role="listitem"><!-- user 2 --></div>
  <!-- ... more items ... -->
</div>
```

**Key CSS properties explained:**

| Property | What it does | Why it matters |
|----------|-------------|----------------|
| `overflow-x: auto` | Shows scrollbar only when content overflows | Enables native horizontal scroll without JS |
| `overflow-y: hidden` | Prevents vertical scroll on the tray | Avoids diagonal scroll on touch devices |
| `flex-wrap: nowrap` | All items stay in one row, excess overflows | Creates the scrollable region |
| `scroll-behavior: smooth` | Animates programmatic `scrollTo()` / `scrollBy()` calls | Smooth auto-scroll-to-unseen, arrow button navigation |
| `-webkit-overflow-scrolling: touch` | Enables momentum (inertia) scrolling on iOS | Without this, iOS scroll feels "sticky" and stops immediately |
| `scrollbar-width: none` | Hides the scrollbar track | Instagram-style clean UI — scroll via touch/drag, not scrollbar |
| `gap: 12px` | Space between flex children | Cleaner than `margin-right` on each item (no trailing margin issue) |

**Why hide the scrollbar?**
*   The tray is a **touch/swipe-first** UI pattern (similar to carousels). Users expect to scroll by dragging, not by clicking a scrollbar.
*   Visible scrollbars take vertical space and look out of place in a compact avatar row.
*   On desktop, we add explicit arrow buttons instead (covered below) — they're more discoverable than a thin scrollbar.

**What `scroll-behavior: smooth` does NOT do:**
*   It only affects **programmatic** scrolls (`scrollTo`, `scrollBy`, `scrollIntoView`). It does **NOT** affect finger/touch scrolling — that's always native-smooth.
*   If you need animated scrolling from JS without `scroll-behavior: smooth`, you'd use `requestAnimationFrame` + easing — but the CSS property is simpler and sufficient for our use case.

**What `-webkit-overflow-scrolling: touch` does (iOS):**
*   Without it, scrolling inside a `overflow` container on older iOS Safari is **synchronous** — the scroll stops the moment you lift your finger.
*   With it, iOS applies **momentum physics** — a fast swipe continues scrolling with deceleration (like the native iOS scroll).
*   On modern iOS (15+), this is largely automatic, but the property provides backward compatibility.

**When this approach is enough:**
*   **< ~50 avatars**: The DOM has ~50 nodes in the tray. Each avatar is just an `<img>` + `<span>` wrapped in a `<button>` — lightweight. The browser can paint and composite this efficiently.
*   Fast initial render: No JS overhead for scroll management.
*   Zero bundle cost: Pure CSS, no library dependency.

---

#### 5.2.2 Virtualized Horizontal List

##### The Problem — Why We Can't Just Render Everything

When a power user follows 2,000+ accounts and 300 of them have active stories, rendering 300 avatar items means:
*   **~900+ DOM nodes** just for the tray (each avatar = image + ring SVG/border + label ≈ 3 nodes).
*   Browser must **layout, paint, and composite** all 900 nodes even though only ~5-7 are visible at a time on screen.
*   On low-end Android devices, this causes **200-500ms jank** on initial mount.
*   More DOM nodes = more memory usage, more garbage collection pressure, slower `querySelector` operations.

The core insight: **the user can only see ~5-7 avatars at a time**. Rendering the other 293 is pure waste.

---

##### The Concept — What is Virtualization?

Virtualization (also called **windowing**) is a rendering technique where you **only mount DOM elements for items that are currently visible** (or about to become visible). Everything outside the visible "window" simply doesn't exist in the DOM.

Think of it like a theatre stage with a **moving spotlight**:
*   The stage (scrollable area) is 24,000px wide (300 avatars × 80px each).
*   The spotlight (viewport) is only ~400px wide (~5 avatars).
*   Only the actors (DOM nodes) standing **in the spotlight** are real. Everyone else is waiting backstage (unmounted).
*   As the spotlight moves (user scrolls), new actors walk on stage and old ones walk off.

**The key trick**: The scrollable area **feels** like all 300 items exist (the scroll thumb is proportionally small, momentum scroll works, total width is correct) — but the browser is only doing work for ~17 items at any given moment.

---

##### The Approach — How Virtualization Works (Conceptual Flow)

There are **three layers** that work together to create the illusion of a full list:

```
┌─ Outer Container (what the user scrolls) ──────────────────────────────────┐
│  overflow-x: auto;  width: 100% of screen (~400px)                         │
│                                                                            │
│  ┌─ Inner Spacer (creates full scroll width) ────────────────────────────┐ │
│  │  width: 24,000px  (300 items × 80px)                                  │ │
│  │  No visible content — just occupies space for scrollbar physics       │ │
│  │                                                                       │ │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                         │ │
│  │  │ Av 4 │ │ Av 5 │ │ Av 6 │ │ Av 7 │ │ Av 8 │  ← Absolutely           │ │
│  │  │pos:  │ │pos:  │ │pos:  │ │pos:  │ │pos:  │    positioned           │ │
│  │  │320px │ │400px │ │480px │ │560px │ │640px │    rendered items       │ │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘                         │ │
│  │                                                                       │ │
│  │  Items 0-3: NOT in DOM          Items 9-299: NOT in DOM               │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────────┘
                         ▲ viewport shows items 4-8 ▲
```

**Layer 1 — Outer Scroll Container:**
*   A `<div>` with `overflow-x: auto` and a fixed width (screen width).
*   This is the element the user physically scrolls. The browser manages all native scroll physics (momentum, touch gestures) on this element.
*   A `scroll` event listener on this div tells the virtualizer where the user is.

**Layer 2 — Inner Spacer:**
*   A single empty `<div>` whose width = `totalItems × itemWidth`.
*   It has **zero visible content**. Its only job is to tell the browser: "the scrollable content is 24,000px wide."
*   This makes the scrollbar thumb size and scroll range behave correctly — without it, the scroll container would think there's nothing to scroll.

**Layer 3 — Absolutely Positioned Visible Items:**
*   Only the items within (and near) the viewport are rendered as real DOM nodes.
*   Each item is `position: absolute` and placed at its computed `left` pixel offset.
*   They "float" on top of the spacer at exactly the right position.

---

##### The Flow — What Happens on Each Scroll Event

```
User scrolls the tray
       ↓
1. Browser fires `scroll` event on the outer container
       ↓
2. Virtualizer reads `scrollLeft` from the container
   (e.g., scrollLeft = 480px)
       ↓
3. Virtualizer calculates which items are in the viewport:
   - Viewport start: scrollLeft = 480px
   - Viewport end:   scrollLeft + clientWidth = 480 + 400 = 880px
   - Item size: 80px
   - Visible range: index 6 to index 11  (480/80 = 6,  880/80 = 11)
       ↓
4. Add overscan buffer (e.g., 5 on each side):
   - Render range: index 1 to index 16
       ↓
5. Virtualizer returns the list of "virtual items" with their positions:
   [{ index: 1, start: 80 }, { index: 2, start: 160 }, ... { index: 16, start: 1280 }]
       ↓
6. React re-renders ONLY these 16 items as <div style="position:absolute; left:{start}px">
       ↓
7. Items that were previously rendered but are now outside the range are UNMOUNTED
   (their DOM nodes are removed, images are unloaded, memory is freed)
       ↓
8. User sees a smooth, continuous-looking list — indistinguishable from rendering all 300
```

**The math is simple:**
*   `startIndex = Math.floor(scrollLeft / itemWidth) - overscan`
*   `endIndex = Math.ceil((scrollLeft + viewportWidth) / itemWidth) + overscan`
*   Clamp both to `[0, totalCount - 1]`

---

##### Why This Doesn't Feel Broken — The Illusion

You might wonder: "If items are being mounted and unmounted constantly, won't the user see flickering or blank spaces?"

**No, because of three things:**

1. **The overscan buffer** — We render 5 extra items on each side that the user can't see yet. By the time an overscan item scrolls into view, it's already been in the DOM for a while (fully painted and composited). The user never sees it "pop in."

2. **Absolute positioning** — Items don't affect each other's layout. Mounting item #15 doesn't cause items #6-#14 to shift or re-flow. Each item is independently placed at its computed `left` offset.

3. **The spacer maintains scroll physics** — Because the inner `<div>` is always the full 24,000px wide, the browser's scroll thumb, momentum, and scroll events behave identically to a non-virtualized list.

```
Visual: What the virtualizer renders at any moment

  Total list:  [0] [1] [2] [3] [4] [5] [6] [7] [8] [9] [10] [11] [12] [13] ... [299]
                         ↑                                       ↑
  Rendered:          [buffer] [buffer] [visible...visible] [buffer] [buffer]
  In DOM:              items 2-12  (5 overscan + ~5 visible + 5 overscan ≈ 15 items)
  NOT in DOM:          items 0-1, items 13-299  (285 items saved!)
```

---

##### Simple Implementation — Step by Step

Here's the simplest possible virtualized horizontal list, explained line by line:

```tsx
import { useRef } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

function StoryTray({ trayItems }: { trayItems: StoryTrayItem[] }) {
  // Step 1: Create a ref for the scrollable container
  // The virtualizer needs direct access to read scrollLeft
  const scrollRef = useRef<HTMLDivElement>(null);

  // Step 2: Initialize the virtualizer
  const virtualizer = useVirtualizer({
    count: trayItems.length,        // total items (300) — NOT rendered, just counted
    getScrollElement: () => scrollRef.current,  // which DOM element to track scroll on
    estimateSize: () => 80,         // each avatar is ~80px wide
    horizontal: true,               // horizontal mode (tracks scrollLeft, not scrollTop)
    overscan: 5,                    // render 5 extra items on each side as buffer
  });

  // Step 3: Render the three-layer structure
  return (
    // Layer 1: Outer scrollable container (fixed width = screen width)
    <div
      ref={scrollRef}
      className="story-tray"     // has overflow-x: auto
      role="list"
      style={{ overflow: 'auto', position: 'relative' }}
    >
      {/* Layer 2: Inner spacer — creates the full scroll width */}
      <div
        style={{
          // This is the magic: a 24,000px wide empty div
          // makes the scrollbar and scroll physics work correctly
          width: virtualizer.getTotalSize(),  // 300 × 80 = 24,000px
          height: '100%',
          position: 'relative',
        }}
      >
        {/* Layer 3: Only the visible + buffer items */}
        {virtualizer.getVirtualItems().map((virtualItem) => {
          // virtualItem contains:
          //   .index  → which item in the original array (e.g., 6)
          //   .start  → pixel offset from left (e.g., 480px)
          //   .size   → width of this item (e.g., 80px)

          const item = trayItems[virtualItem.index];

          return (
            <div
              key={item.userId}
              role="listitem"
              style={{
                // Absolutely positioned at the correct offset
                // This is why items don't affect each other's layout
                position: 'absolute',
                top: 0,
                left: virtualItem.start,   // e.g., 480px for item #6
                width: virtualItem.size,   // 80px
                height: '100%',
              }}
            >
              <StoryAvatarItem item={item} />
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

**What happens when this renders:**

| Step | What happens | DOM result |
|------|-------------|------------|
| Mount | Virtualizer calculates visible range (items 0-11 with overscan=5) | ~12 DOM nodes created |
| User scrolls right | `scroll` event fires → virtualizer recalculates range (e.g., 3-16) | Items 0-2 unmounted, items 12-16 mounted |
| User scrolls fast | Multiple recalculations per frame → batched via `requestAnimationFrame` | Smooth transition, ~15-17 nodes always |
| User reaches end | Range becomes (283-299 with overscan) | Still only ~17 nodes |

---

##### Key Configuration — What Each Option Does

| Option | Value | What it does | What happens if wrong |
|--------|-------|-------------|----------------------|
| `count` | `trayItems.length` | Tells virtualizer the total number of items. Used to calculate total scroll width. | Too high → scroll area extends past actual content. Too low → items at the end are unreachable. |
| `estimateSize` |  `() => 80` | Estimated pixel width per item. Used for initial layout before items are measured. | Too small → items overlap or scroll area is too short. Too large → gaps between items or scroll area is too long. **Must be close to actual size.** |
| `horizontal` | `true` | Switches virtualizer to track `scrollLeft` instead of `scrollTop`. Items are positioned with `left` instead of `top`. | If `false` (default), items stack vertically — wrong axis for a horizontal tray. |
| `overscan` | `5` | Extra items rendered outside viewport on each side. Acts as a buffer against blank flashes during fast scrolling. | Too low (0-1) → user sees blank space during fast scroll. Too high (20+) → too many DOM nodes, defeats purpose. **5 is a good default.** |

**Choosing the right `overscan` value:**

| overscan | DOM nodes (viewport=7) | Fast-scroll experience | Memory |
|----------|----------------------|----------------------|--------|
| `0` | 7 | Blank flashes on fast scroll | Minimal |
| `3` | 13 | Occasional flash on very fast scroll | Low |
| `5` | 17 | Smooth for normal-to-fast scroll | Low |
| `10` | 27 | No visible flashing even at extreme speed | Moderate |
| `20+` | 47+ | Overkill — approaches non-virtualized | Defeats purpose |

---

##### Performance Impact — With vs Without Virtualization

| Metric | No virtualization (300 items) | Virtualized (overscan=5) |
|--------|-------------------------------|--------------------------|
| DOM nodes in tray | ~900 | ~51 (17 items × 3 nodes) |
| Initial render time | 150-500ms | 10-30ms |
| Memory usage | Higher (all 300 images loaded) | Lower (only ~17 images loaded) |
| Scroll smoothness | Can jank on low-end devices | Consistent 60fps |
| Bundle size added | 0 | ~5-8KB (`@tanstack/react-virtual`) |
| Code complexity | Simple `.map()` | Virtualizer setup + absolute positioning |

---

##### When to Virtualize vs Not

| Situation | Recommendation | Why |
|-----------|---------------|-----|
| **< 30 items** | **Don't virtualize** | Overhead of scroll listeners + absolute positioning + re-renders on scroll isn't worth it. Plain CSS is faster and simpler. |
| **30-50 items** | **Optional** — depends on item complexity | If each item is lightweight (one `<img>` + `<span>`), skip it. If each item has heavy SVG rings or multiple images, consider it. |
| **50-100 items** | **Recommended** | DOM savings become noticeable. ~150 nodes saved. |
| **100+ items** | **Always virtualize** | Without it, render time visibly degrades. 300+ items = major jank. |
| **Unbounded count** | **Always virtualize** | If you can't guarantee a max, you must assume worst case. |

**The trade-off is clear:**
*   Virtualization adds ~5-8KB to bundle + some code complexity.
*   In return, you get **O(1) DOM nodes** regardless of list size — the tray renders identically whether there are 5 stories or 5,000.

---

#### 5.2.3 Desktop Arrow Navigation

**Why arrows are needed on desktop:**
*   Desktop users don't have touch/swipe. They can use trackpad swipe or scrollbar drag, but both are **non-obvious** (scrollbar is hidden, trackpad requires horizontal gesture support).
*   Left/right arrow buttons are a **clear affordance** — the user immediately understands the tray is scrollable.
*   Instagram web uses this exact pattern: semi-transparent arrow overlays that appear on hover.

**Implementation:**

```tsx
function StoryTrayWithArrows({ trayItems }: { trayItems: StoryTrayItem[] }) {
  const scrollRef = useRef<HTMLDivElement>(null);
  const [canScrollLeft, setCanScrollLeft] = useState(false);
  const [canScrollRight, setCanScrollRight] = useState(true);

  // Update arrow visibility on scroll
  const handleScroll = useCallback(() => {
    const el = scrollRef.current;
    if (!el) return;
    setCanScrollLeft(el.scrollLeft > 0);
    setCanScrollRight(el.scrollLeft + el.clientWidth < el.scrollWidth - 1);
    // ↑ the "-1" accounts for sub-pixel rounding
  }, []);

  useEffect(() => {
    const el = scrollRef.current;
    if (!el) return;
    el.addEventListener('scroll', handleScroll, { passive: true });
    handleScroll(); // initial check
    return () => el.removeEventListener('scroll', handleScroll);
  }, [handleScroll]);

  const scrollTray = (direction: 'left' | 'right') => {
    const el = scrollRef.current;
    if (!el) return;
    const scrollAmount = el.clientWidth * 0.8; // scroll by 80% of visible area
    el.scrollBy({
      left: direction === 'left' ? -scrollAmount : scrollAmount,
      behavior: 'smooth',  // relies on CSS scroll-behavior or built-in smooth scroll
    });
  };

  return (
    <div className="story-tray-wrapper">
      {canScrollLeft && (
        <button
          className="tray-arrow tray-arrow--left"
          onClick={() => scrollTray('left')}
          aria-label="Scroll stories left"
        >‹</button>
      )}

      <div ref={scrollRef} className="story-tray" role="list">
        {/* ... avatar items ... */}
      </div>

      {canScrollRight && (
        <button
          className="tray-arrow tray-arrow--right"
          onClick={() => scrollTray('right')}
          aria-label="Scroll stories right"
        >›</button>
      )}
    </div>
  );
}
```

**Key design decisions:**

| Decision | Rationale |
|----------|-----------|
| **`passive: true`** on scroll listener | Tells the browser the handler won't call `preventDefault()`. This allows the browser to **optimize scroll performance** and not wait for JS before scrolling. Critical for 60fps. |
| **80% of `clientWidth`** as scroll amount | Scrolling by the full width would disorient the user (entire visible set changes). 80% keeps ~1-2 items from the previous view as a visual anchor. |
| **`behavior: 'smooth'`** on `scrollBy` | Animates the scroll so users can track the motion. Instant jumps are disorienting. |
| **Hide arrows conditionally** | Left arrow hidden at start (nothing to scroll back to). Right arrow hidden at end. Prevents dead clicks. |
| **`scrollWidth - 1` edge check** | Browsers sometimes have sub-pixel differences between `scrollLeft + clientWidth` and `scrollWidth`. The `-1` prevents the right arrow from disappearing prematurely. |
| **Show arrows on hover only (CSS)** | On desktop, arrows appear when the user hovers over the tray area. This keeps the UI clean when the tray isn't being interacted with. |

**Arrow visibility CSS:**

```css
.story-tray-wrapper {
  position: relative;                /* anchor for absolute-positioned arrows */
}

.tray-arrow {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  z-index: 2;
  background: rgba(255, 255, 255, 0.9);
  border-radius: 50%;
  width: 32px;
  height: 32px;
  border: none;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(0,0,0,0.15);
  opacity: 0;
  transition: opacity 0.2s ease;
}

.story-tray-wrapper:hover .tray-arrow {
  opacity: 1;                        /* show arrows on hover */
}

.tray-arrow--left  { left: 8px; }
.tray-arrow--right { right: 8px; }
```

**Detecting desktop vs mobile (for arrow rendering):**
*   Use CSS media query `@media (hover: hover) and (pointer: fine)` — this matches devices with a mouse/trackpad (desktop) and excludes touch-only devices.
*   Or use JS: `window.matchMedia('(hover: hover)').matches`.
*   Don't show arrows on mobile — they clutter the UI and touch scroll is the primary interaction.

---

#### 5.2.4 Tray Ordering Unseen First, Seen Pushed to Back

##### How Instagram Actually Works

Instagram's story tray does **NOT** scroll to the first unseen story. Instead, the **backend sorts the tray** so that unseen stories always come first. When you view someone's story:
*   That user's avatar **moves to the end** of the tray (after all unseen stories).
*   The tray always starts at scroll position 0 (leftmost).
*   Unseen stories naturally occupy the visible area — no programmatic scrolling needed.

This is a **server-driven sort**, not a client-side trick.

##### The Tray Sort Order (How Instagram Ranks It)

```
Position 0        Position 1...N              Position N+1...M
┌──────────┐    ┌──────────────────────┐    ┌──────────────────────┐
│ Your Own │    │   Unseen Stories     │    │   Seen Stories       │
│  Story   │    │   (sorted by ranking │    │   (sorted by when    │
│ (pinned) │    │    algorithm)        │    │    you viewed them)  │
└──────────┘    └──────────────────────┘    └──────────────────────┘
  Always first   ← User sees these first     Pushed to the right →
```

**Detailed sort priority:**
1. **Own story** — Always pinned at index 0. Shows "Add" button if no active story, or a preview ring if story exists.
2. **Unseen stories** — Users whose stories you haven't viewed yet. Within this group, ranked by:
   *   Engagement score (how often you interact with this person — likes, DMs, comments).
   *   Recency (newer stories ranked higher).
   *   Close Friends stories may be boosted.
3. **Seen stories** — Users whose stories you've already fully viewed. Sorted by:
   *   When you viewed them (most recently viewed first) — so you can easily re-watch.
   *   They get a **grey ring** instead of the gradient ring.

##### What Happens When You View a Story — The Re-Sort Flow

```
BEFORE viewing user B's story:

  [You] [B 🟣] [C 🟣] [D 🟣] [A ⚪] [E ⚪]
         ↑ unseen (gradient ring)      ↑ seen (grey ring)

User taps B → views all of B's slides → closes viewer

AFTER viewing:

  [You] [C 🟣] [D 🟣] [B ⚪] [A ⚪] [E ⚪]
                        ↑ B moved to "seen" section
                          ring changed from gradient → grey
```

**Step-by-step what the frontend does:**

1. **User finishes viewing B's story** (all slides seen, or user exits viewer).
2. **Local state update (immediate):**
   *   Set `B.hasUnseenStory = false` → ring changes from gradient to grey.
   *   Move B from the unseen group to the **front** of the seen group (most recently viewed).
3. **Report to server:**
   *   `POST /api/stories/seen/batch` with B's slide IDs.
4. **On next tray refetch** (background refetch / pull-to-refresh):
   *   Server returns the re-ranked tray with B in the seen section.
   *   Client reconciles (usually already matches local state).

##### Frontend Implementation — Optimistic Re-Sort

```tsx
function moveToSeen(userId: string) {
  setTrayItems((prev) => {
    const item = prev.find((i) => i.userId === userId);
    if (!item || !item.hasUnseenStory) return prev; // already seen

    // Mark as seen
    const updatedItem = { ...item, hasUnseenStory: false };

    // Remove from current position
    const without = prev.filter((i) => i.userId !== userId);

    // Find the boundary between unseen and seen
    const firstSeenIndex = without.findIndex(
      (i) => !i.isOwnStory && !i.hasUnseenStory
    );

    // Insert at the beginning of the seen section
    const insertAt = firstSeenIndex === -1 ? without.length : firstSeenIndex;
    const newTray = [
      ...without.slice(0, insertAt),
      updatedItem,
      ...without.slice(insertAt),
    ];

    return newTray;
  });
}
```

**Why do it optimistically (client-side) instead of waiting for the server?**
*   The server re-sort arrives on the **next tray refetch** (could be seconds or minutes later).
*   Without optimistic re-sort, the user would see B with a grey ring sitting in the unseen section — confusing.
*   The local re-sort keeps the tray visually consistent immediately.

##### Why This Design Eliminates "Scroll-to-Unseen"

Because unseen stories are always at the left (positions 1...N), the tray **starts at the right place by default**:
*   Scroll position 0 = your story + first unseen stories = exactly what the user wants.
*   No need for `scrollToIndex()` or `scrollIntoView()`.
*   No jarring auto-scroll animation on tray load.

**The only case where programmatic scrolling might help:**
*   If the user has already scrolled the tray to the right (to seen stories), then **new unseen stories arrive** via background refetch or WebSocket push → you could nudge the user by scrolling back to position 0, or showing a "New stories" indicator at the left edge.

```tsx
// Only scroll back to start if new unseen stories appeared
useEffect(() => {
  const hasNewUnseen = trayItems.some(
    (item) => !item.isOwnStory && item.hasUnseenStory
  );
  const userHasScrolled = scrollRef.current?.scrollLeft > 0;

  if (hasNewUnseen && userHasScrolled) {
    // Option A: Auto-scroll back (can feel intrusive)
    // scrollRef.current?.scrollTo({ left: 0, behavior: 'smooth' });

    // Option B: Show a small "← New stories" pill at the left edge (less intrusive)
    setShowNewStoriesIndicator(true);
  }
}, [trayItems]);
```

##### Edge Cases

| Scenario | What happens |
|----------|-------------|
| **User views a story, then pulls to refresh** | Server returns re-sorted tray; B is now in seen section. Local state matches. No visual jump. |
| **User views a story in the middle of the unseen group** | That user moves to the seen section. Remaining unseen stories shift left to fill the gap. The tray might visually "jump" — mitigate with `layout animation` or accept the shift. |
| **Multiple stories viewed in quick succession** | Batch the re-sorts. Don't re-sort after each slide — wait until the user exits the viewer, then move all viewed users at once. |
| **User opens app after a long time (all stories are new)** | Entire tray is unseen stories. Tray starts at position 0. Perfect — no action needed. |
| **User has seen all stories** | Entire tray is grey rings. Tray starts at position 0 showing own story + most recently viewed. |
| **New story from a user already in seen section** | That user moves back to the unseen section (server handles this on next refetch). Frontend should detect `hasUnseenStory` change and re-sort locally. |

---

#### 5.2.5 RTL Language Support

In RTL languages (Arabic, Hebrew, Urdu, Persian), the tray must scroll in the **opposite direction**:
*   Start position is the **right** edge.
*   "Next" stories are to the **left**.
*   Arrow buttons flip: right arrow goes "back", left arrow goes "forward".

**How it works with CSS:**

```css
.story-tray[dir="rtl"] {
  direction: rtl;   /* or set on a parent element */
}
```

When `direction: rtl` is applied:
*   `scrollLeft` starts at `0` but counts in the **negative** direction (in Firefox) or starts at `scrollWidth - clientWidth` and counts down (in Chrome). **This is inconsistent across browsers**.
*   Flexbox reverses the main axis — items flow from right to left naturally.

**Handling scroll direction inconsistency:**

```tsx
function getNormalizedScrollLeft(el: HTMLElement): number {
  if (el.dir === 'rtl' || getComputedStyle(el).direction === 'rtl') {
    // Normalize: Firefox uses negative scrollLeft for RTL, Chrome uses positive
    return Math.abs(el.scrollLeft);
  }
  return el.scrollLeft;
}
```

**Arrow logic must flip:**
*   In LTR: Left arrow = scroll toward start, Right arrow = scroll toward end.
*   In RTL: Left arrow = scroll toward **end** (more content), Right arrow = scroll toward **start**.

```tsx
const isRTL = getComputedStyle(scrollRef.current).direction === 'rtl';
const scrollAmount = el.clientWidth * 0.8;

el.scrollBy({
  left: direction === 'left'
    ? (isRTL ? scrollAmount : -scrollAmount)
    : (isRTL ? -scrollAmount : scrollAmount),
  behavior: 'smooth',
});
```

---

#### 5.2.6 Lazy Loading Avatars in the Tray

##### The Problem — Too Many Image Requests

Each avatar in the tray is a small profile photo (~64×64px). Without any optimization:
*   300 stories = **300 HTTP requests** fired at once on page load.
*   Even though each image is tiny (~2-5KB), 300 simultaneous requests **saturate the browser's connection pool** (browsers limit to ~6 concurrent requests per domain).
*   This delays loading of more important resources (story media, feed images, JS bundles).
*   On slow networks, the tray avatars load in a random order as requests complete — users see blank circles gradually filling in.

##### Two Approaches — Depending on Whether You Use Virtualization

The strategy differs based on whether the tray is virtualized or not:

```
┌─────────────────────────────────────────────────────────────────┐
│                     WITHOUT Virtualization                      │
│  All 300 items are in the DOM                                   │
│  → Need lazy loading to prevent 300 simultaneous image fetches  │
│  → Use: loading="lazy" or IntersectionObserver                  │
├─────────────────────────────────────────────────────────────────┤
│                      WITH Virtualization                        │
│  Only ~17 items are in the DOM at any time                      │
│  → Images load only when the item mounts (enters viewport)      │
│  → Lazy loading is automatic — no extra work needed!            │
└─────────────────────────────────────────────────────────────────┘
```

---

##### Approach 1: `loading="lazy"` (Without Virtualization)

The HTML `loading="lazy"` attribute tells the browser: **"Don't fetch this image until it's about to become visible in the viewport."**

```tsx
function StoryAvatarItem({ item, index }: { item: StoryTrayItem; index: number }) {
  return (
    <button
      role="listitem"
      aria-label={`View story by ${item.username}`}
      onClick={() => openStory(item.userId)}
    >
      <img
        src={item.avatarUrl}
        alt={`${item.username}'s avatar`}
        width={64}
        height={64}
        // First ~7 items are visible on load — load them immediately
        // Remaining items: defer until they scroll into view
        loading={index < 7 ? 'eager' : 'lazy'}
      />
      <span>{item.username}</span>
    </button>
  );
}
```

**How `loading="lazy"` works under the hood:**

1. Browser parses the HTML and sees `<img loading="lazy">`.
2. Instead of immediately fetching the image, the browser registers an **internal IntersectionObserver** on the `<img>` element.
3. The image fetch is deferred until the element is **within a threshold distance** from the viewport (the exact threshold varies by browser — Chrome uses ~1250px on fast connections, ~2500px on slow 3G).
4. Once the image enters the threshold zone, the browser begins the fetch.
5. When the fetch completes, the image is decoded and painted.

```
                    ← viewport (~400px) →
  [loaded] [loaded] [loaded] [loaded] [loaded] [threshold zone ~1250px] [not loaded] [not loaded]
  ↑ eager (first 7)                             ↑ browser starts        ↑ completely deferred
                                                  fetching here
```

**Why `index < 7` gets `loading="eager"` (critical):**

*   The first ~7 avatars are **above the fold** — visible immediately when the feed loads.
*   If they use `loading="lazy"`, the browser **waits** for the IntersectionObserver to confirm they are in the viewport before fetching. This adds a **100-200ms delay** to what should be an instant load.
*   This would hurt **LCP** (Largest Contentful Paint) — the tray rings would appear empty briefly.
*   `loading="eager"` (the default) tells the browser to fetch immediately, just like a normal image.

**Browser support:** `loading="lazy"` is supported in all modern browsers (Chrome 77+, Firefox 75+, Safari 15.4+, Edge 79+). For older browsers, it simply loads eagerly (graceful fallback — no broken behavior).

---

##### Approach 2: IntersectionObserver (Manual Lazy Loading)

If you need more control than `loading="lazy"` provides (e.g., custom placeholder, fade-in animation, exact threshold control), use IntersectionObserver directly:

```tsx
function LazyAvatar({ avatarUrl, username }: { avatarUrl: string; username: string }) {
  const imgRef = useRef<HTMLImageElement>(null);
  const [isLoaded, setIsLoaded] = useState(false);

  useEffect(() => {
    const img = imgRef.current;
    if (!img) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          // Image is near the viewport — start loading
          img.src = avatarUrl;
          observer.unobserve(img);  // only need to observe once
        }
      },
      {
        root: null,               // observe relative to the viewport
        rootMargin: '200px',      // start loading 200px before it's visible
        threshold: 0,             // trigger as soon as even 1px enters the margin
      }
    );

    observer.observe(img);
    return () => observer.disconnect();
  }, [avatarUrl]);

  return (
    <img
      ref={imgRef}
      src="data:image/svg+xml,..."  // inline SVG placeholder (grey circle)
      alt={`${username}'s avatar`}
      onLoad={() => setIsLoaded(true)}
      width={64}
      height={64}
      style={{
        opacity: isLoaded ? 1 : 0.5,        // fade in when loaded
        transition: 'opacity 0.2s ease',
      }}
    />
  );
}
```

**When to use IntersectionObserver over `loading="lazy"`:**

| Feature | `loading="lazy"` | IntersectionObserver |
|---------|------------------|---------------------|
| Setup complexity | Zero (HTML attribute) | Moderate (hook + observer) |
| Placeholder control | Browser shows empty space | Full control (custom SVG, blur-up, shimmer) |
| Threshold control | Browser decides (~1250px) | You decide (`rootMargin: '200px'`) |
| Fade-in animation | Not possible natively | Yes (via `onLoad` + CSS transition) |
| Horizontal scroll | Works but threshold optimized for vertical | Works perfectly; set `root` to scroll container |
| Browser support | Chrome 77+, Safari 15.4+ | Chrome 51+, Safari 12.1+ (broader) |

**For the story tray, `loading="lazy"` is sufficient** — avatars are simple, small, and don't need fancy placeholders.

---

##### Approach 3: Virtualization Makes It Automatic

With virtualization (section 5.2.2), lazy loading is a **free side effect**:

```
Without virtualization:
  DOM: [img1] [img2] [img3] ... [img300]  ← all 300 <img> tags exist
  Browser must decide which to fetch      ← needs loading="lazy" to prevent 300 fetches

With virtualization:
  DOM: [img4] [img5] [img6] [img7] [img8]  ← only 5-17 <img> tags exist
  Only mounted images can be fetched       ← lazy loading is inherent!
  Unmounted items = no <img> tag = no fetch
```

When the user scrolls and new avatar items mount, their `<img>` tags enter the DOM for the first time. The browser fetches the image immediately (since it's now in/near the viewport). When the user scrolls past and the item unmounts, the `<img>` is removed — the browser can garbage-collect the decoded image data.

**Even with virtualization, there's a subtle benefit to `loading="lazy"` on overscan items:**
*   Overscan items (the buffer) are in the DOM but NOT visible yet.
*   Without `loading="lazy"`, the browser fetches their images immediately when mounted.
*   With `loading="lazy"`, the browser may defer them further until they're closer to the viewport edge.
*   In practice, the difference is minimal (overscan items are only ~5 items away), but it's a free optimization.

---

##### Avatar Placeholder Strategy

While the real avatar is loading, what does the user see?

| Strategy | Implementation | Visual result | Best for |
|----------|---------------|--------------|----------|
| **Empty space** | No `src` attribute until loaded | Jarring jump when image appears | Never — bad UX |
| **Grey circle** | Inline SVG/CSS as default `src` | Clean, matches ring UI | Story tray (simple & fast) |
| **Blur-up (LQIP)** | Tiny 8×8 base64 thumbnail → full image | Smooth progressive reveal | Feed posts (not needed for 64px avatars) |
| **Shimmer / skeleton** | CSS animation on placeholder | Modern "loading" feel | When load times are noticeable (>200ms) |

**For the story tray, a simple grey circle placeholder is ideal** — avatars are small (64px), load fast (2-5KB), and the gradient ring already provides visual structure.

---

#### 5.2.7 Scroll Performance Considerations

| Concern | What happens | Mitigation |
|---------|-------------|------------|
| **Expensive scroll handler** | JS runs on every scroll pixel → blocks compositor → jank | Use `passive: true` on listener; debounce arrow show/hide with `requestAnimationFrame` |
| **Layout thrashing** | Reading `scrollLeft` + `scrollWidth` forces layout recalc | Batch reads; cache `scrollWidth` (it doesn't change unless items change) |
| **Repaints on avatar mount** | mounting new items in virtualizer triggers paint | Items are `position: absolute` — they exist on their own compositing layer; mounting one doesn't affect others |
| **Image decode jank** | Large avatar images decoding on main thread during scroll | Use `img.decode()` promise or `content-visibility: auto` for off-screen items |
| **Touch scroll blocking** | Active (non-passive) touch listeners delay scroll start by up to 300ms | Never use non-passive listeners on the scroll container |

**`requestAnimationFrame` for scroll-linked updates:**

```tsx
const handleScroll = useCallback(() => {
  // Avoid calling setState on every scroll pixel — batch to next frame
  requestAnimationFrame(() => {
    const el = scrollRef.current;
    if (!el) return;
    setCanScrollLeft(el.scrollLeft > 0);
    setCanScrollRight(el.scrollLeft + el.clientWidth < el.scrollWidth - 1);
  });
}, []);
```

This ensures arrow visibility updates at most once per frame (60fps) instead of potentially hundreds of times per second during a fast scroll.

---

#### 5.2.8 Decision Matrix

| Scenario | Strategy | Library / Tool | Notes |
|----------|----------|---------------|-------|
| **< 50 avatars** | Plain CSS `overflow-x: auto` + flexbox | None (native CSS) | Zero JS overhead; fastest initial render |
| **50–500+ avatars** | Virtualized horizontal list | `@tanstack/react-virtual` / `react-window` | Only ~17 DOM nodes regardless of total count |
| **Desktop users** | Arrow buttons with `scrollBy()` | Custom (show on `@media (hover: hover)`) | Passive scroll listener + rAF for arrow state |
| **Mobile users** | Native touch scroll | Built-in (CSS `-webkit-overflow-scrolling: touch`) | No arrows needed; momentum scroll is default |
| **RTL languages** | CSS `direction: rtl` + normalized `scrollLeft` | Native CSS + small JS helper | Must handle browser inconsistency in RTL `scrollLeft` |
| **Seen story re-sort** | Optimistic client-side re-sort on view (move seen to back) | Custom `moveToSeen()` logic | Server-driven sort; client mirrors it optimistically |
| **Avatar image loading** | `loading="lazy"` or auto via virtualization | Native HTML attribute | First ~7 visible avatars should be `loading="eager"` |
| **Scroll performance** | `passive: true` + `rAF` batching | Native APIs | Never block compositor; batch DOM reads |

---

### 5.3 Reusability Strategy

*   **`AvatarRing`**: Reused across story tray, DM list, and profile. Configurable via props: `size`, `ringColor`, `hasUnseenStory`.
*   **`ProgressBar` / `ProgressSegment`**: Generic timed-progress component, reusable for Reels or any auto-advancing UI.
*   **`MediaRenderer`**: Shared image/video component with lazy loading, used in feed posts and stories.
*   Design-system tokens for colors, spacing, typography.

---

### 5.4 Module Organization

```
src/
 ├── features/
 │    └── stories/
 │         ├── components/
 │         │    ├── StoryTray.tsx
 │         │    ├── StoryAvatarItem.tsx
 │         │    ├── StoryViewer.tsx
 │         │    ├── StoryProgressBar.tsx
 │         │    ├── StoryMediaRenderer.tsx
 │         │    ├── StoryReplyDrawer.tsx
 │         │    └── StoryNavigationLayer.tsx
 │         ├── hooks/
 │         │    ├── useStoryTimer.ts
 │         │    ├── useStoryNavigation.ts
 │         │    └── useStoryPreloader.ts
 │         ├── store/
 │         │    └── storyStore.ts
 │         ├── api/
 │         │    └── storyApi.ts
 │         ├── utils/
 │         │    └── storyHelpers.ts
 │         └── types/
 │              └── story.types.ts
 ├── shared/
 │    ├── components/ (AvatarRing, MediaRenderer, ProgressBar)
 │    ├── hooks/ (useIntersection)
 │    └── utils/ (timer, mediaLoader)
```

---

## 6. High Level Data Flow Explanation

### 6.1 Initial Load Flow

```
1. Feed page loads (SSR shell)
     ↓
2. Client hydrates → StoryTray mounts
     ↓
3. Fetch story tray data: GET /api/stories/tray
     ↓
4. Render avatars with seen/unseen rings
     ↓
5. Prefetch first story media for the first 3-5 users
   (using <link rel="preload"> or Image() constructor)
```

---

### 6.2 User Interaction Flow

```
User taps avatar in tray
     ↓
1. Open StoryViewer overlay (lazy-loaded chunk)
     ↓
2. Fetch full stories for that user: GET /api/stories/user/:userId
   (if not already prefetched)
     ↓
3. Render first slide → start timer
     ↓
4. Timer completes → advance to next slide (progress bar fills)
     ↓
5. Last slide → auto-advance to next user's stories
     ↓
6. User swipes right → go back to previous user
     ↓
7. Report view: POST /api/stories/:storyId/seen (debounced/batched)
     ↓
8. Update local seen state → tray ring turns grey
```

**Optimistic Updates**:
*   Mark story as "seen" in local state immediately on view, then confirm with server.

---

### 6.3 Error and Retry Flow

*   **Media fails to load**: Show a placeholder with a retry button; skip to next slide after timeout.
*   **API failure on tray fetch**: Show cached tray data (if available) or a subtle error banner with retry.
*   **View tracking fails**: Queue in memory/localStorage, retry on next successful API call (fire-and-forget pattern).
*   **Network lost during viewer**: Pause timer, show "No connection" toast, resume when online.

---

## 7. Data Modelling (Frontend Perspective)

### 7.1 Core Data Entities

*   **StoryTrayItem** — a user's avatar + metadata for the tray
*   **StoryGroup** — a user's collection of story slides
*   **StorySlide** — individual image/video/text slide
*   **StoryReaction** — emoji or text reply to a slide

---

### 7.2 Data Shape

```ts
type StoryTrayItem = {
  userId: string;
  username: string;
  avatarUrl: string;
  hasUnseenStory: boolean;
  latestStoryTimestamp: string; // for sorting
  isCloseFriend: boolean;
  isOwnStory: boolean;
};

type StorySlide = {
  id: string;
  type: 'image' | 'video' | 'text';
  mediaUrl?: string;           // CDN URL with signed token
  thumbnailUrl?: string;       // low-res placeholder
  videoDuration?: number;      // seconds (for video type)
  displayDuration: number;     // seconds (default 5 for images)
  backgroundColor?: string;    // for text-only stories
  createdAt: string;
  expiresAt: string;
  isSeen: boolean;
};

type StoryGroup = {
  userId: string;
  username: string;
  avatarUrl: string;
  slides: StorySlide[];
  currentSlideIndex: number;  // UI state: which slide the user is on
};
```

---

### 7.3 Entity Relationships

*   **One‑to‑Many**: One `StoryTrayItem` → one `StoryGroup` → many `StorySlide`s.
*   **Normalized storage**: Story groups stored by `userId` key in a map for O(1) lookup.

---

### 7.4 UI Specific Data Models

```ts
// Derived view model for the story viewer
type StoryViewerState = {
  isOpen: boolean;
  userQueue: string[];              // ordered list of userIds to navigate through
  currentUserIndex: number;
  currentSlideIndex: number;
  isPaused: boolean;
  isMuted: boolean;                 // global mute toggle for videos
  elapsedTime: number;              // ms elapsed on current slide (for progress bar)
  replyDrawerOpen: boolean;
};
```

---

## 8. State Management Strategy

### 8.1 State Classification

| State Type | Examples | Storage |
|---|---|---|
| **Global App State** | Authenticated user, mute preference | Zustand / Redux global store |
| **Feature State** | Story tray data, story groups, seen map | Feature-level store (Zustand slice / React Query cache) |
| **Viewer UI State** | Current user/slide index, paused, elapsed time | Component-local state (useReducer) |
| **Derived State** | "Has unseen stories" flag, progress % | Computed selectors |

---

### 8.2 State Ownership

*   **StoryTray** owns the tray list and fetch logic; passes `userId` to viewer on tap.
*   **StoryViewer** owns navigation state (current user, slide index, paused) internally using `useReducer`.
*   **Story Store** (Zustand / React Query) owns the cached story groups and seen state, shared between tray and viewer.
*   Prop drilling is minimal: tray passes `userId` → viewer reads from store.

---

### 8.3 Persistence Strategy

*   **In‑memory**: Story groups, viewer state (ephemeral, no persistence needed).
*   **React Query / SWR cache**: Story tray data cached with `staleTime: 5min`; background refetch on focus.
*   **localStorage**: Seen story IDs (fallback for fast tray rendering before API responds); mute preference.
*   **IndexedDB (optional)**: Prefetched media blobs for offline-capable PWA mode.

---

## 9. High Level API Design (Frontend POV)

### 9.1 Required APIs

| API | Method | Description |
|-----|--------|-------------|
| `/api/stories/tray` | **GET** | Fetch story tray (list of users with active stories) |
| `/api/stories/user/:userId` | **GET** | Fetch all story slides for a specific user |
| `/api/stories/:storyId/seen` | **POST** | Report that the user has seen a story slide |
| `/api/stories/seen/batch` | **POST** | Batch report seen stories (debounced) |
| `/api/stories/:storyId/react` | **POST** | Send emoji reaction to a story |
| `/api/stories/:storyId/reply` | **POST** | Send text reply to a story |
| `/api/stories/create` | **POST** | Upload and publish a new story |

---

### 9.2 Two Phase Fetching Strategy

**Short answer:** We do **NOT** fetch all stories for all users upfront. We use a **two-phase approach** — tray metadata first, individual story slides on demand.

#### Why Not Fetch Everything At Once?

If a user follows 500 accounts and 200 have active stories, each with an average of 3-5 slides:
*   200 users × 4 slides = **800 slide objects** with media URLs, metadata, overlays.
*   Each slide might have a 200-500 byte JSON payload → ~200-400KB of JSON, just for data the user might never look at.
*   The user typically only views **5-15 users' stories** per session.
*   Fetching everything upfront wastes **bandwidth, memory, and server resources**.

#### Phase 1: Tray Metadata (On Feed Load)

When the feed page loads, we fetch **only the tray** — a lightweight list of users who have active stories. No individual slides, no media URLs.

```
GET /api/stories/tray
```

**What this returns:**

```
┌─────────────────────────────────────────────────────────────────┐
│  TRAY DATA (lightweight — ~50 bytes per user)                   │
│                                                                 │
│  For each user:                                                 │
│   • userId, username, avatarUrl                                 │
│   • hasUnseenStory (boolean)                                    │
│   • latestStoryTimestamp                                        │
│   • isCloseFriend                                               │
│   • storyCount (optional — to show number of segments)          │
│                                                                 │
│  What it does NOT include:                                      │
│   ✗ Individual slide data (mediaUrl, duration, etc.)            │
│   ✗ Actual story media (images, videos)                         │
│   ✗ Seen/unseen state per slide                                 │
└─────────────────────────────────────────────────────────────────┘
```

This is enough to render the entire tray — avatars, usernames, gradient/grey rings. **Total payload: ~10-20KB for 200 users.**

#### Phase 2: Story Slides (On User Tap — Lazy Fetch)

When the user taps on a specific user's avatar, **only then** do we fetch that user's actual story slides:

```
GET /api/stories/user/:userId
```

**What this returns:**

```
┌─────────────────────────────────────────────────────────────────┐
│  STORY SLIDES for user "zeeshan_ali"                            │
│                                                                 │
│  Slide 1: { id, type: "image", mediaUrl, displayDuration: 5 }  │
│  Slide 2: { id, type: "video", mediaUrl, videoDuration: 12 }   │
│  Slide 3: { id, type: "text",  backgroundColor: "#ff5733" }    │
│                                                                 │
│  ~500 bytes - 2KB per user                                      │
└─────────────────────────────────────────────────────────────────┘
```

#### The Complete Fetch Timeline

```
User opens Instagram feed
  │
  ├─ Phase 1: GET /api/stories/tray ──────────────────────── ~200ms
  │    → Returns: 200 users with avatars, seen/unseen flags
  │    → Tray renders immediately
  │
  │  ... user scrolls feed, reads posts ...
  │
  ├─ User taps on user B's avatar ────────────────────────── ~100ms
  │    → Phase 2: GET /api/stories/user/B
  │    → Returns: B's 4 slides with media URLs
  │    → Viewer opens, first slide renders
  │
  │  ... user watches B's stories, auto-advances to user C ...
  │
  ├─ Auto-advance to user C ──────────────────────────────── ~0ms (prefetched!)
  │    → Phase 2: GET /api/stories/user/C (already fetched ahead)
  │    → Seamless transition, no loading spinner
  │
  │  ... user continues watching ...
  │
  ├─ Auto-advance to user D ──────────────────────────────── ~100ms (or prefetched)
  │    → Phase 2: GET /api/stories/user/D
  │
  └─ User closes viewer
       → Only 3 users' slides were fetched (B, C, D)
       → The other 197 users' slides were NEVER loaded
```

#### Prefetching Making Phase 2 Feel Instant

The problem with strict on-demand fetching: when the user taps an avatar, there's a **100-200ms delay** while the slide data loads. For a media-heavy, instant-feeling UI like stories, that's noticeable.

**Solution: Prefetch the next user's slides while the current user's stories are playing.**

```
User is viewing B's stories (4 slides)
  │
  ├─ Slide 1 playing (5 seconds)
  │    → Background: prefetch C's slides: GET /api/stories/user/C
  │    → Background: preload C's first slide image
  │
  ├─ Slide 2 playing
  │    → C's slides already cached ✅
  │    → Background: preload C's second slide image
  │
  ├─ Slide 3 playing
  │    → Background: prefetch D's slides: GET /api/stories/user/D
  │
  ├─ Slide 4 (last slide)
  │    → D's slides already cached ✅
  │
  └─ Auto-advance to C → instant! (data + first image already loaded)
```

**Implementation with React Query:**

```tsx
function useStoryViewer(currentUserId: string, nextUserId: string | null) {
  // Fetch current user's slides (active query)
  const { data: currentStories } = useQuery({
    queryKey: ['stories', currentUserId],
    queryFn: () => fetchStorySlides(currentUserId),
  });

  // Prefetch next user's slides (background, non-blocking)
  const queryClient = useQueryClient();
  useEffect(() => {
    if (nextUserId) {
      queryClient.prefetchQuery({
        queryKey: ['stories', nextUserId],
        queryFn: () => fetchStorySlides(nextUserId),
        staleTime: 5 * 60 * 1000,  // cache for 5 minutes
      });
    }
  }, [nextUserId]);

  return currentStories;
}
```

#### What About Prefetching on Tray Load?

For the first **3-5 users** in the tray, we can proactively prefetch their slide data **before the user taps anything**. This makes the first story tap feel instant.

```tsx
// After tray data loads, prefetch top users' slides
useEffect(() => {
  if (!trayItems.length) return;

  // Prefetch slide data for first 3 unseen users
  const topUsers = trayItems
    .filter((item) => !item.isOwnStory && item.hasUnseenStory)
    .slice(0, 3);

  topUsers.forEach((user) => {
    queryClient.prefetchQuery({
      queryKey: ['stories', user.userId],
      queryFn: () => fetchStorySlides(user.userId),
    });
  });
}, [trayItems]);
```

**Why only 3-5?**
*   Prefetching too many wastes bandwidth — the user may never tap into stories at all.
*   3-5 covers the most likely first taps (highest-ranked unseen stories).
*   Each prefetch is ~500 bytes-2KB of JSON — very cheap.

#### Summary Fetching Decision Matrix

| What | When fetched | API | Payload size | Cache TTL |
|------|-------------|-----|-------------|-----------|
| **Tray metadata** (200 users) | Feed page load | `GET /api/stories/tray` | ~10-20KB total | 2-5 min (refetch on focus) |
| **Top 3-5 users' slides** | After tray loads (background prefetch) | `GET /api/stories/user/:id` | ~1-5KB each | 5 min |
| **Tapped user's slides** | On avatar tap (or already prefetched) | `GET /api/stories/user/:id` | ~500B-2KB | 5 min |
| **Next user's slides** | While current user's story plays | `GET /api/stories/user/:id` | ~500B-2KB | 5 min |
| **Story media (images)** | When slide is about to be shown | CDN URL from slide data | 50-500KB each | 24h (CDN + browser) |
| **Story media (videos)** | When video slide starts | CDN URL (HLS/MP4) | 1-5MB each | 24h (CDN + browser) |

**The key insight:** There are actually **three levels** of lazy loading happening:
1. **Slide JSON data** — fetched per-user, on demand or prefetched.
2. **Slide images** — preloaded 1-2 ahead using `new Image().src`.
3. **Slide videos** — only start loading when the video slide is about to play.

---

### 9.3 Request and Response Structure

**GET /api/stories/tray**

```json
// Response
{
  "tray": [
    {
      "userId": "u_123",
      "username": "zeeshan_ali",
      "avatarUrl": "https://cdn.example.com/avatars/u_123.jpg",
      "hasUnseenStory": true,
      "latestStoryTimestamp": "2026-03-16T08:30:00Z",
      "isCloseFriend": false,
      "previewMediaUrl": "https://cdn.example.com/stories/preview_123.jpg"
    }
  ],
  "nextCursor": "c_abc123"
}
```

**GET /api/stories/user/:userId**

```json
// Response
{
  "userId": "u_123",
  "username": "zeeshan_ali",
  "avatarUrl": "https://cdn.example.com/avatars/u_123.jpg",
  "slides": [
    {
      "id": "s_001",
      "type": "image",
      "mediaUrl": "https://cdn.example.com/stories/s_001.jpg?token=abc",
      "thumbnailUrl": "https://cdn.example.com/stories/s_001_thumb.jpg",
      "displayDuration": 5,
      "createdAt": "2026-03-16T08:30:00Z",
      "expiresAt": "2026-03-17T08:30:00Z",
      "isSeen": false
    }
  ]
}
```

**POST /api/stories/seen/batch**

```json
// Request
{
  "seenStories": [
    { "storyId": "s_001", "seenAt": "2026-03-16T09:00:00Z" },
    { "storyId": "s_002", "seenAt": "2026-03-16T09:00:02Z" }
  ]
}

// Response
{ "success": true }
```

---

### 9.4 Error Handling and Status Codes

| Status | Scenario | Frontend Handling |
|--------|----------|-------------------|
| `200` | Success | Render data |
| `304` | Not Modified | Use cached version |
| `401` | Unauthorized | Redirect to login |
| `404` | Story expired / deleted | Remove from tray, skip in viewer |
| `429` | Rate limit | Backoff + retry with exponential delay |
| `500` | Server error | Show error toast, retry button |

---

## 10. Caching Strategy

### 10.1 What to Cache

*   **Story tray**: Cached with short TTL (2-5 min) — changes frequently.
*   **Story slides**: Cached per user; invalidate when story expires or new story is added.
*   **Media assets**: Aggressively cached by CDN and browser (`Cache-Control: max-age=86400` since stories expire in 24h anyway).
*   **Seen state**: Cached in localStorage for instant tray rendering.

### 10.2 Where to Cache

| Data | Cache Location |
|------|----------------|
| Story tray list | React Query / SWR in-memory cache |
| Story slides per user | React Query cache (keyed by userId) |
| Images / Videos | Browser HTTP cache + CDN |
| Seen story IDs | localStorage |
| Prefetched next-user media | In-memory blob / browser cache |

### 10.3 Cache Invalidation

*   **Story tray**: Refetch on app focus (`refetchOnWindowFocus`), and on SSE/WebSocket event for new stories.
*   **Slides**: Invalidate when `expiresAt` is passed (client-side TTL check).
*   **Seen state**: Sync from server on tray fetch; local seen state is source of truth until next server sync.

---

## 11. CDN and Asset Optimization

*   **Media delivery**: All story images/videos served via CDN with edge caching.
*   **Image optimization**:
    *   Serve WebP/AVIF with fallback to JPEG.
    *   Responsive sizes: `1080w` for full-screen, `150w` thumbnails for tray avatars.
    *   Use `<picture>` element or `srcset` for format/size negotiation.
*   **Video optimization**:
    *   HLS adaptive bitrate for videos (240p → 1080p based on network).
    *   Short videos (< 15s) can be served as MP4 with `preload="auto"`.
*   **Signed URLs**: Media URLs include short-lived tokens (1h expiry) to prevent unauthorized access.
*   **Cache headers**: `Cache-Control: public, max-age=86400` for story media (24h matches story lifetime).

---

## 12. Rendering Strategy

*   **Feed Page (including Story Tray)**: SSR for the shell → fast FCP; hydrate on client.
*   **Story Viewer**: Fully **CSR** — loaded as a **lazy-loaded chunk** when user taps into a story.
    *   `React.lazy(() => import('./StoryViewer'))` with `Suspense` fallback (skeleton overlay).
*   **Media rendering**:
    *   Images: Native `<img>` with `loading="eager"` (within viewer) + blur-up placeholder.
    *   Videos: `<video>` with `preload="auto"`, `playsinline`, `muted` (auto-play policy).
*   **Portal rendering**: Story viewer rendered in a React Portal attached to `document.body` to escape parent overflow/z-index stacking.
*   **No SSR for viewer**: The viewer is interactive-only content; SSR adds no SEO or FCP value.

---

## 13. Cross Cutting Non Functional Concerns

### 13.1 Security

*   **Signed media URLs**: CDN URLs expire after 1h; tokens are tied to the user's session.
*   **No client-side URL exposure**: Media URLs are not persisted in localStorage or logged.
*   **XSS mitigation**: All user-generated text is sanitized before rendering (DOMPurify).
*   **CSRF**: All POST requests (reactions, replies, seen) include CSRF tokens.
*   **Content Security Policy**: Restrict media loading to known CDN origins.

---

### 13.2 Accessibility

*   **Story Tray**:
    *   `role="list"` with `role="listitem"` for each avatar.
    *   Each avatar is a `<button>` with `aria-label="View story by {username}"`.
    *   Tray is keyboard-navigable with arrow keys.
*   **Story Viewer**:
    *   `role="dialog"` with `aria-modal="true"`.
    *   Progress bar as `role="progressbar"` with `aria-valuenow`.
    *   Keyboard shortcuts: `→` next, `←` previous, `Space` pause/resume, `Esc` close.
    *   Screen reader announces: "Story by {username}, slide {n} of {total}".
    *   `aria-live="polite"` region for slide transitions.
*   **Video**: Captions via `<track>` element when available.
*   **Reduced motion**: Respect `prefers-reduced-motion` — disable auto-advance, let user tap manually.
*   **High contrast**: Ensure progress bar and overlay text meet WCAG AA contrast ratios.

---

### 13.3 Performance Optimization

#### Media Preloading Strategy (Critical)

```
Current User's Stories:
  [Slide 1 ✅ loaded] [Slide 2 🔄 preloading] [Slide 3 ⏳ queued]

Next User in Queue:
  [Slide 1 🔄 preloading] [Slide 2 ⏳ queued]
```

*   **Preload next slide** while current slide is displayed.
*   **Preload first slide of next user** when on the last slide of current user.
*   **Preload tray**: On feed load, preload first slide images for the first 3-5 users in tray.
*   Use `new Image().src = url` for images and `<link rel="preload" as="video">` for videos.

#### Implementation

```js
function useStoryPreloader(currentGroup, currentSlideIndex, nextGroup) {
  useEffect(() => {
    // Preload next slide in current group
    const nextSlide = currentGroup.slides[currentSlideIndex + 1];
    if (nextSlide?.mediaUrl) {
      const img = new Image();
      img.src = nextSlide.mediaUrl;
    }

    // Preload first slide of next user
    if (currentSlideIndex === currentGroup.slides.length - 1 && nextGroup) {
      const firstSlide = nextGroup.slides[0];
      if (firstSlide?.mediaUrl) {
        const img = new Image();
        img.src = firstSlide.mediaUrl;
      }
    }
  }, [currentSlideIndex, currentGroup, nextGroup]);
}
```

#### Other Optimizations

*   **Code splitting**: Story viewer chunk loaded on first tap (~50-80KB).
*   **Virtualized tray**: Only render visible avatars + buffer in the horizontal scroll (if 100+ stories).
*   **Debounced seen reporting**: Batch seen API calls every 3-5 seconds instead of per-slide.
*   **RAF-based progress bar**: Use `requestAnimationFrame` for smooth 60fps progress animation.
*   **Memoization**: `React.memo` on StorySlide to prevent re-renders during timer ticks.
*   **Abort stale fetches**: Cancel in-flight story fetches when user swipes past a user quickly.

---

### 13.4 Observability and Reliability

*   **Error Boundaries**: Wrap `StoryViewer` in an error boundary — if a slide crashes, skip to next.
*   **Logging**: Track key events:
    *   Story viewed (impression), story completed, story skipped, reply sent, reaction sent.
    *   Media load failures (with URL, error type, network info).
    *   Time spent per slide.
*   **Performance monitoring**:
    *   Track viewer open latency (tap → first slide rendered).
    *   Track media load time per slide.
    *   Core Web Vitals for the feed page (LCP, CLS with tray rendering).
*   **Feature flags**: Gate new features (e.g., new story types, reply formats) behind flags.

---

## 14. Edge Cases and Tradeoffs

| Edge Case | Handling |
|-----------|----------|
| **Story expires while viewing** | Check `expiresAt` before rendering; if expired, skip slide and show toast "This story is no longer available". |
| **User has 50+ stories** | Paginate slides; show progress segments grouped (e.g., first 10 visible, rest as a thin bar). |
| **Video fails to load** | Show thumbnail with play-retry button; auto-skip after 5s timeout. |
| **Slow network** | Show low-res thumbnail immediately; load full-res in background; don't auto-advance until loaded. |
| **User rapidly taps** | Debounce navigation; cancel preloads for skipped slides; prioritize loading target slide. |
| **Two tabs open** | Seen state may diverge; reconcile on next tray fetch. |

### Key Trade-offs

| Decision | Trade-off |
|----------|-----------|
| **Preload next user's stories eagerly** | Faster transitions but more bandwidth usage (wasted if user exits early). |
| **Batch seen reporting** | Fewer API calls but slight delay in seen state accuracy on server. |
| **Lazy-load viewer chunk** | Smaller initial bundle but ~100-200ms delay on first story open. Mitigated with prefetch on hover/tray visibility. |
| **CSR-only viewer** | No SSR overhead, simpler code, but no SEO (stories aren't SEO content anyway). |
| **localStorage for seen state** | Instant tray rendering without API, but can go stale across devices. |

---

## 15. Summary and Future Improvements

### Key Architectural Decisions

1. **Portal-based full-screen viewer** — clean separation from feed, avoids z-index wars.
2. **Aggressive media preloading** — preload next slide + next user for seamless experience.
3. **Batched seen reporting** — reduces API calls from per-slide to periodic batches.
4. **Code-split viewer** — keeps feed page bundle small; viewer loaded on demand.
5. **React Query for story data** — automatic caching, background refetch, stale-while-revalidate.

### Possible Future Enhancements

*   **Offline story viewing**: Cache recent stories in IndexedDB via Service Worker for offline access.
*   **Shared Element Transition**: Animate avatar from tray to viewer header using View Transitions API.
*   **Skeleton loading**: Show story-shaped skeleton while chunk + data loads.
*   **WebSocket for real-time tray updates**: Push new story notifications to update the tray without polling.
*   **Collaborative stories**: Multiple users contributing slides to a shared story (requires conflict resolution).
*   **Web Workers for media processing**: Offload image/video thumbnail generation during story creation.

---

### Endpoint Summary

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/stories/tray` | GET | Fetch story tray |
| `/api/stories/user/:userId` | GET | Fetch user's story slides |
| `/api/stories/seen/batch` | POST | Batch report seen stories |
| `/api/stories/:storyId/react` | POST | Send reaction |
| `/api/stories/:storyId/reply` | POST | Send reply |
| `/api/stories/create` | POST | Publish new story |

---

### Complete Story Flow

| Direction | Mechanism | Trigger | Endpoint | Action |
|-----------|-----------|---------|----------|--------|
| Tray Load | REST | On mount | `/api/stories/tray` | Show avatar ring list |
| Open Viewer | REST | Tap avatar | `/api/stories/user/:userId` | Full-screen story viewer |
| Auto-Advance | Client Timer | Slide duration | — | Next slide / next user |
| Navigation | User Tap | Tap left / right | — | Move between slides/users |
| Seen Tracking | REST (batched) | Every 3-5s | `/api/stories/seen/batch` | Mark slides as seen |
| Reaction | REST | User taps emoji | `/api/stories/:id/react` | Send reaction to author |
| New Stories | SSE / WebSocket | Server push | `/api/stories/stream` | Update tray with new ring |

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design
