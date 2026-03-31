# Frontend System Design: Pinterest Image Discovery and Pin Grid

- [Frontend System Design: Pinterest Image Discovery and Pin Grid](#frontend-system-design-pinterest-image-discovery-and-pin-grid)
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
    - [5.2 Masonry Grid Layout and Infinite Scroll Strategy (Deep Dive)](#52-masonry-grid-layout-and-infinite-scroll-strategy-deep-dive)
      - [5.2.1 What is a Masonry Layout and Why Pinterest Uses It](#521-what-is-a-masonry-layout-and-why-pinterest-uses-it)
      - [5.2.2 Column Assignment Algorithm (Shortest Column First)](#522-column-assignment-algorithm-shortest-column-first)
      - [5.2.3 CSS Based Masonry Approaches](#523-css-based-masonry-approaches)
      - [5.2.4 JavaScript Based Masonry with Absolute Positioning](#524-javascript-based-masonry-with-absolute-positioning)
      - [5.2.5 Responsive Column Count](#525-responsive-column-count)
      - [5.2.6 Virtualized Masonry Grid for Infinite Scroll](#526-virtualized-masonry-grid-for-infinite-scroll)
      - [5.2.7 Image Loading and Placeholder Strategy](#527-image-loading-and-placeholder-strategy)
      - [5.2.8 Scroll Performance Considerations](#528-scroll-performance-considerations)
      - [5.2.9 Decision Matrix](#529-decision-matrix)
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
    - [9.2 Cursor Based Pagination for Grid Feed (Deep Dive)](#92-cursor-based-pagination-for-grid-feed-deep-dive)
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

*   Pinterest is a visual discovery and bookmarking platform where users browse, save, and organize images (called "Pins") into themed collections (called "Boards").
*   Target users: casual browsers seeking inspiration, content creators sharing visual content, businesses promoting products visually.
*   Primary use case: browsing an infinite, image-heavy masonry grid of pins — discovering, saving, and organizing visual content across categories like home decor, recipes, fashion, and DIY.

---

### 1.2 Key User Personas

*   **Browser / Discoverer**: Scrolls the home feed or search results, saves pins to boards, and follows topics or creators for inspiration.
*   **Content Creator / Pinner**: Creates and uploads pins (images with descriptions and links), organizes boards, and tracks engagement.
*   **Business / Brand**: Promotes products via rich pins (with pricing, availability), runs promoted pin campaigns, and uses analytics to measure reach.

---

### 1.3 Core User Flows (High Level)

*   **Browsing the Home Feed (Primary Flow)**:
    1.  User opens Pinterest → home feed loads with the first batch of personalized pins in a masonry grid.
    2.  User scrolls down → more pins load automatically (infinite scroll).
    3.  User hovers over a pin → quick-action overlay appears (Save, Link out, Share).
    4.  User taps a pin → pin detail modal/page opens with full image, description, related pins, and comments.
    5.  User taps "Save" → selects a board → pin is saved (optimistic update).

*   **Searching for Pins (Secondary Flow)**:
    1.  User types in the search bar → autocomplete suggestions appear.
    2.  User selects or submits a query → search results grid loads.
    3.  Results are filterable by category, color, and media type.
    4.  User saves or clicks through to external links.

*   **Board Management (Secondary Flow)**:
    1.  User navigates to their profile → sees a grid of boards.
    2.  Taps a board → sees all saved pins in that board (masonry grid).
    3.  Can reorder pins, move pins between boards, create new boards.

---

## 2. Requirements

### 2.1 Functional Requirements

*   **Pin Grid Display**:
    *   Display a responsive masonry grid of pins (variable-height image cards) with infinite scroll.
    *   Each pin card shows: image, title (optional), author avatar, save button, link icon.
    *   Support multiple column counts based on viewport width (2 on mobile, 4-6 on desktop).
*   **Pin Interaction**:
    *   Hover overlay with Save, Link, and Share actions (desktop).
    *   Tap to open pin detail view (modal or page).
    *   Save pin to a board (with board picker dropdown).
    *   Click-through to external source URL.
*   **Pin Detail View**:
    *   Full-size image with zoom capability.
    *   Pin description, source link, author info.
    *   "More like this" related pins grid below.
    *   Comments section.
*   **Search and Discovery**:
    *   Search bar with autocomplete/typeahead.
    *   Visual search results in masonry layout.
    *   Topic-based explore page.
*   **Board Management**:
    *   Create, rename, delete boards.
    *   Save pins to boards.
    *   View board contents in masonry grid.
*   **Infinite Scroll**:
    *   Fetch more pins as the user scrolls (cursor-based pagination).
    *   Seamless loading with no explicit "Load More" button.

---

### 2.2 Non Functional Requirements

*   **Performance**: FCP < 1.5s; LCP < 2.5s (largest pin image above the fold); smooth 60fps scrolling with 200+ pins loaded; masonry layout recalculation must be imperceptible (< 16ms).
*   **Scalability**: Support grids with thousands of pins per session; millions of pin images at variable aspect ratios; responsive from 320px mobile to 2560px ultrawide.
*   **Availability**: Graceful degradation — show cached pins if network is down; skeleton placeholders during loading.
*   **Security**: XSS prevention on user-generated content (pin descriptions, comments); CSRF tokens on all state-changing requests; CDN-served images with signed URLs for private boards.
*   **Accessibility**: Full keyboard navigation through pin grid; screen reader support with semantic HTML and alt text on all images; focus management on modal open/close; reduced motion support.
*   **Device Support**: Mobile web (primary), desktop web (responsive), tablet; low-end devices with limited memory and GPU.
*   **i18n**: RTL layout support; localized UI strings and timestamps; locale-aware number formatting.

---

## 3. Scope Clarification (Interview Scoping)

### 3.1 In Scope

*   Masonry grid layout algorithm and rendering strategy (the core deep-dive).
*   Pin card component design with hover interactions and image optimization.
*   Infinite scroll with cursor-based pagination in a multi-column layout.
*   Pin detail modal with related pins.
*   Save-to-board interaction with optimistic updates.
*   State management for grid data and board selections.
*   Performance optimization for image-heavy, variable-height grids.
*   API design from the frontend perspective.

---

### 3.2 Out of Scope

*   Backend recommendation/ranking algorithm (feed personalization).
*   Pin creation / upload flow (image processing, metadata extraction).
*   Visual search (image-based search using ML).
*   Promoted pins / ads system.
*   Push notifications.
*   Direct messaging.
*   Rich pins (product, recipe, article types — deep integration with source sites).

---

### 3.3 Assumptions

*   User is authenticated; auth token is available via HTTP-only cookie or Authorization header.
*   APIs return pins pre-ranked by the backend (frontend does not rank or sort).
*   Pin images are served from a CDN with multiple size variants (thumbnail, medium, original).
*   The API provides image dimensions (`width`, `height`) in the response so the frontend can calculate aspect ratios **before** images load (critical for masonry layout).

---

## 4. High Level Frontend Architecture

### 4.1 Overall Approach

*   **SPA** (Single Page Application) with client-side routing.
*   **SSR** for the initial home feed and pin detail pages — server renders the first batch of pins for fast FCP, SEO (pin pages should be indexable), and social preview (OG metadata for shared pin URLs).
*   **CSR** for all subsequent interactions — infinite scroll loads, board management, search interactions.
*   Pin grid module is the **primary chunk**; pin detail modal, board picker, and search are **code-split** and lazy loaded.

---

### 4.2 Major Architectural Layers

```
┌──────────────────────────────────────────────────────────────────┐
│  UI Layer                                                        │
│  ┌────────────────┐  ┌───────────────────────────────────────┐   │
│  │ SearchBar      │  │  MasonryGrid                          │   │
│  │ (typeahead)    │  │  ┌──────────┐ ┌──────────┐ ┌──────┐  │   │
│  └────────────────┘  │  │ PinCard  │ │ PinCard  │ │ Pin  │  │   │
│                      │  │ ┌──────┐ │ │          │ │ Card │  │   │
│  ┌────────────────┐  │  │ │Hover │ │ │          │ │      │  │   │
│  │ CategoryTabs   │  │  │ │Overl.│ │ │          │ │      │  │   │
│  │ (explore)      │  │  │ └──────┘ │ │          │ │      │  │   │
│  └────────────────┘  │  └──────────┘ └──────────┘ └──────┘  │   │
│                      │       ┌────────────────┐              │   │
│  ┌────────────────┐  │       │ GridSentinel   │              │   │
│  │ PinDetailModal │  │       │ (infinite scr.)│              │   │
│  │ (lazy loaded)  │  │       └────────────────┘              │   │
│  └────────────────┘  └───────────────────────────────────────┘   │
├──────────────────────────────────────────────────────────────────┤
│  State Management Layer                                          │
│  (Pin Grid Store, Board Store, Search State, User Session,       │
│   Save/Unsave Queue, UI Preferences)                             │
├──────────────────────────────────────────────────────────────────┤
│  API and Data Access Layer                                       │
│  (REST Client, Optimistic Update Queue, Request Deduplication,   │
│   Image Preloader, Retry Logic)                                  │
├──────────────────────────────────────────────────────────────────┤
│  Shared / Utility Layer                                          │
│  (IntersectionObserver helpers, Masonry calculator, Debounce,    │
│   Image sizing utils, Analytics tracker, Date formatting)        │
└──────────────────────────────────────────────────────────────────┘
```

---

### 4.3 External Integrations

*   **CDN**: Serves pin images at multiple resolutions (Cloudfront / Akamai / Fastly). URL-based resizing (e.g., `image.jpg?w=236` for thumbnails, `?w=736` for detail).
*   **Analytics SDK**: Track pin impressions (viewed in grid), click-through events, save events, scroll depth, and session duration.
*   **Backend Services**: Feed API (ranked pins), search API, board API, save/unsave API, user profile service.
*   **Image Processing Service**: Backend generates multiple image sizes on upload; provides `width`, `height`, `dominantColor`, and `blurhash` in API responses.
*   **Link Preview Service**: Extracts OG metadata from source URLs for rich pin display.

---

## 5. Component Design and Modularization

### 5.1 Component Hierarchy

```
HomePage
 ├── NavBar
 │    ├── Logo
 │    ├── SearchBar (typeahead input)
 │    └── UserMenu (profile, settings)
 │
 ├── CategoryTabs (optional — explore page)
 │
 ├── MasonryGrid (main pin grid container)
 │    └── PinCard[] (repeated for each pin, absolutely positioned)
 │         ├── PinImage (responsive image with placeholder)
 │         ├── PinHoverOverlay (desktop only)
 │         │    ├── SaveButton (with board picker dropdown)
 │         │    ├── SourceLinkBadge
 │         │    └── ShareButton
 │         ├── PinFooter
 │         │    ├── PinTitle (truncated)
 │         │    ├── AuthorAvatar + AuthorName
 │         │    └── SaveCount (optional)
 │         └── PinBoardPicker (dropdown on save)
 │
 ├── GridSentinel (invisible element — triggers next page load)
 │
 └── PinDetailModal (Portal/Overlay — rendered on pin tap)
      ├── PinDetailImage (full-size with zoom)
      ├── PinMetadata (description, source link, author)
      ├── PinActions (Save, Share, Visit Link)
      ├── CommentSection
      │    ├── CommentList
      │    │    └── CommentItem[]
      │    └── CommentInput
      └── RelatedPinsGrid (masonry — "More like this")
```

---

### 5.2 Masonry Grid Layout and Infinite Scroll Strategy (Deep Dive)

The masonry grid is the **defining UI pattern of Pinterest** — a multi-column, variable-height grid where items (pin cards) are packed tightly without equal row heights. Getting this right matters because:
*   The grid is the **primary content surface** — users spend 90%+ of their time browsing pins in this layout.
*   Pin images have **wildly varying aspect ratios** (portrait, landscape, square, tall infographics), making a standard CSS grid with equal rows unusable.
*   The grid can grow to **hundreds or thousands of items** during a single session via infinite scroll.
*   Every scroll frame must render at **60fps** — layout recalculations on new pin batches must be invisible to the user.
*   The layout must be **responsive** — dynamically adjusting column count from 2 (mobile) to 6+ (ultrawide desktop) without breaking positions.

---

#### 5.2.1 What is a Masonry Layout and Why Pinterest Uses It

##### What Makes Masonry Different from a Regular Grid

In a standard CSS Grid or Flexbox grid, all items in a row share the **same height**:

```
Standard Grid (equal rows):
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│  200px   │ │  200px   │ │  200px   │ │  200px   │
│  tall    │ │  tall    │ │  tall    │ │  tall    │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│  200px   │ │  200px   │ │  200px   │ │  200px   │
│  tall    │ │  tall    │ │  tall    │ │  tall    │
└──────────┘ └──────────┘ └──────────┘ └──────────┘

  → Images are cropped or stretched to fit equal row height.
  → Lots of wasted whitespace if images have different aspect ratios.
  → Pinterest images vary from 1:1 to 1:3+ — forced cropping would destroy content.
```

In a **masonry layout** (also called "waterfall" or "Pinterest-style" grid), each item retains its **natural aspect ratio**, and the next item is placed in the **shortest column**:

```
Masonry Grid (variable heights):
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│          │ │          │ │          │ │          │
│  300px   │ │  180px   │ │  400px   │ │  220px   │
│          │ └──────────┘ │          │ │          │
│          │ ┌──────────┐ │          │ └──────────┘
└──────────┘ │  250px   │ │          │ ┌──────────┐
┌──────────┐ │          │ └──────────┘ │  350px   │
│  200px   │ └──────────┘ ┌──────────┐ │          │
│          │ ┌──────────┐ │  150px   │ │          │
└──────────┘ │  180px   │ └──────────┘ └──────────┘
             └──────────┘

  → Each image uses its FULL natural height — no cropping.
  → New items fill the shortest column — minimizes wasted vertical space.
  → The grid has a "waterfall" or "staggered" visual effect.
```

##### Why Pinterest Uses Masonry

| Reason | Explanation |
|--------|-------------|
| **No content cropping** | Pin images range from square (1:1) to extremely tall infographics (1:4). Forced cropping would hide critical content. Masonry preserves every pixel. |
| **Maximum content density** | Items pack tightly with minimal whitespace. This matters for a discovery platform — more pins visible = more engagement. |
| **Visual variety** | The staggered heights create an organic, visually interesting layout that encourages browsing. A uniform grid feels rigid and monotonous for image content. |
| **Engagement** | Users tend to scroll longer in masonry layouts because there is no clear "row boundary" — the eye flows naturally between columns. |

---

#### 5.2.2 Column Assignment Algorithm (Shortest Column First)

The core algorithm for masonry layout is simple: **always place the next item in the column with the least total height**. This ensures columns stay visually balanced.

##### The Algorithm — Step by Step

```
Given: columnCount = 4, columnWidth = 236px, gap = 16px
Maintain: columnHeights = [0, 0, 0, 0]  (cumulative height of each column)

For each pin in the batch:
  1. Compute pin card height:
     cardHeight = (pinImageHeight / pinImageWidth) * columnWidth + footerHeight
     Example: image is 600×900 → cardHeight = (900/600) * 236 + 40 = 394px

  2. Find the shortest column:
     shortestColumnIndex = argmin(columnHeights)
     Example: columnHeights = [800, 650, 720, 690] → shortestColumnIndex = 1

  3. Compute pin position:
     top  = columnHeights[shortestColumnIndex]
     left = shortestColumnIndex * (columnWidth + gap)

  4. Place pin at (left, top) using position: absolute + transform

  5. Update column height:
     columnHeights[shortestColumnIndex] += cardHeight + gap
```

##### Visual Walkthrough

```
Step 1: Place Pin A (height: 300px) → shortest column: 0 (all are 0)
  Heights: [300, 0, 0, 0]
  
  Col 0    Col 1    Col 2    Col 3
  ┌─────┐
  │  A  │
  │300px│
  └─────┘

Step 2: Place Pin B (height: 180px) → shortest column: 1 (height 0)
  Heights: [300, 180, 0, 0]

  Col 0    Col 1    Col 2    Col 3
  ┌─────┐ ┌─────┐
  │  A  │ │  B  │
  │300px│ │180px│
  └─────┘ └─────┘

Step 3: Place Pin C (height: 400px) → shortest column: 2 (height 0)
  Heights: [300, 180, 400, 0]

Step 4: Place Pin D (height: 220px) → shortest column: 3 (height 0)
  Heights: [300, 180, 400, 220]

Step 5: Place Pin E (height: 250px) → shortest column: 1 (height 180, lowest)
  Heights: [300, 430, 400, 220]

  Col 0    Col 1    Col 2    Col 3
  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
  │  A  │ │  B  │ │     │ │  D  │
  │300px│ │180px│ │  C  │ │220px│
  └─────┘ ├─────┤ │400px│ └─────┘
           │  E  │ │     │
           │250px│ └─────┘
           └─────┘
```

##### Implementation

```tsx
type PinPosition = {
  top: number;
  left: number;
  width: number;
  height: number;
};

function calculateMasonryLayout(
  pins: Pin[],
  columnCount: number,
  columnWidth: number,
  gap: number,
  footerHeight: number = 40
): { positions: PinPosition[]; totalHeight: number } {
  const columnHeights = new Array(columnCount).fill(0);
  const positions: PinPosition[] = [];

  for (const pin of pins) {
    // 1. Compute card height from image aspect ratio
    const aspectRatio = pin.image.height / pin.image.width;
    const imageHeight = aspectRatio * columnWidth;
    const cardHeight = imageHeight + footerHeight;

    // 2. Find the shortest column
    let shortestColumn = 0;
    let minHeight = columnHeights[0];
    for (let i = 1; i < columnCount; i++) {
      if (columnHeights[i] < minHeight) {
        minHeight = columnHeights[i];
        shortestColumn = i;
      }
    }

    // 3. Compute position
    const top = columnHeights[shortestColumn];
    const left = shortestColumn * (columnWidth + gap);

    positions.push({
      top,
      left,
      width: columnWidth,
      height: cardHeight,
    });

    // 4. Update column height
    columnHeights[shortestColumn] += cardHeight + gap;
  }

  const totalHeight = Math.max(...columnHeights);
  return { positions, totalHeight };
}
```

##### Why "Shortest Column First" and Not Other Approaches

| Approach | How it works | Visual result | Problem |
|----------|-------------|---------------|---------|
| **Round-robin** (pin 0→col 0, pin 1→col 1, ...) | Cycle through columns in order | Unbalanced — one column may be much taller if it receives tall images | Ugly gaps; not visually balanced |
| **Shortest column first** (Pinterest's approach) | Always pick the column with the least cumulative height | Balanced — all columns stay roughly the same height | None significant; this is optimal for visual balance |
| **Left-to-right filling** | Fill each row left to right | Requires equal row height (standard grid) | Can't preserve aspect ratios |

**Shortest column first is optimal** because it minimizes the height difference between columns at every step. The final grid has the most even bottom edge possible for the given set of images.

##### Edge Case: What If Multiple Columns Have the Same Height?

When two or more columns share the same minimum height, pick the **leftmost** one. This creates a left-to-right visual flow that feels natural in LTR layouts. In RTL, pick the rightmost.

```tsx
// Modified shortest column finder for stable left-to-right placement
let shortestColumn = 0;
let minHeight = columnHeights[0];
for (let i = 1; i < columnCount; i++) {
  // Strict less-than (not <=) ensures leftmost column wins ties
  if (columnHeights[i] < minHeight) {
    minHeight = columnHeights[i];
    shortestColumn = i;
  }
}
```

---

#### 5.2.3 CSS Based Masonry Approaches

Before reaching for JavaScript, consider CSS-only options:

##### Approach 1: CSS `column-count` (Multi Column Layout)

```css
.masonry-grid {
  column-count: 4;
  column-gap: 16px;
}

.pin-card {
  break-inside: avoid;        /* prevent card from splitting across columns */
  margin-bottom: 16px;
}
```

**How it works:** The browser's multi-column layout engine flows content vertically (top-to-bottom) within each column, then moves to the next column. It naturally creates a masonry-like effect.

| Pros | Cons |
|------|------|
| Pure CSS, zero JS | Items flow **top-to-bottom, then left-to-right** — not left-to-right, then top-to-bottom. Pin order is wrong (item 2 is below item 1, not next to it). |
| Simple to implement | No control over which column receives which pin |
| Good browser support | Cannot combine with virtualization (items must be in DOM order) |
| | Reflow is expensive on large grids (browser recalculates all column flows) |
| | Infinite scroll appends disrupt layout (all columns reflow on each batch) |

**Why Pinterest does NOT use this:** The reading order is wrong. In `column-count`, items flow **down** first:

```
column-count flow:           Pinterest expected flow:
  Col 0  Col 1  Col 2         Col 0  Col 1  Col 2
  [1]    [4]    [7]            [1]    [2]    [3]
  [2]    [5]    [8]            [4]    [5]    [6]
  [3]    [6]    [9]            [7]    [8]    [9]
```

Users expect a left-to-right reading order (1, 2, 3 across the top), but `column-count` gives a top-to-bottom order (1, 2, 3 down the first column). This is confusing for discovery — the "most important" (highest-ranked) pins should appear at the top-left, not stacked vertically in column 1.

##### Approach 2: CSS Grid with `grid-auto-rows` and `span`

```css
.masonry-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  grid-auto-rows: 10px;      /* small row unit */
  gap: 0 16px;                /* column gap only; row gap handled by span */
}

.pin-card {
  /* span a number of rows proportional to the image height */
  grid-row: span var(--row-span);
}
```

Each pin calculates `--row-span` based on its aspect ratio:
```tsx
const rowHeight = 10; // px per grid row
const span = Math.ceil((cardHeight + gap) / rowHeight);
// Apply as inline style: style={{ gridRow: `span ${span}` }}
```

| Pros | Cons |
|------|------|
| CSS Grid handles positioning | Requires JS to calculate `--row-span` (not truly CSS-only) |
| Left-to-right reading order (correct!) | Gaps appear when spans don't perfectly align |
| Browser handles reflow | Slight visual inaccuracies (rounding to nearest row unit) |
| | Performance degrades with 100+ items (recalculates grid) |
| | Hard to combine with virtualization |

**This approach is viable for small grids** (< 50 pins) but the rounding artifacts and reflow cost make it impractical for Pinterest-scale infinite scroll.

##### Approach 3: CSS Masonry (Upcoming Specification)

```css
.masonry-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  grid-template-rows: masonry;      /* proposed spec */
}
```

This is the **ideal** solution — native CSS masonry layout with correct reading order. However:
*   As of March 2026, it is only supported behind flags in Firefox (Nightly) and Safari (Technology Preview).
*   **Not production-ready** — no Chrome support, specification still evolving.
*   When it ships in all browsers, it will replace JavaScript masonry entirely.

**Bottom line:** For production Pinterest-style grids today, **JavaScript-based absolute positioning** is the standard approach.

---

#### 5.2.4 JavaScript Based Masonry with Absolute Positioning

This is the approach Pinterest itself uses: a JavaScript layout engine that **calculates positions** and places each pin using `position: absolute` with `transform: translate(x, y)`.

##### The Three Layer Architecture (Same as Virtualization)

```
┌─ Outer Container (scrollable) ────────────────────────────────┐
│  overflow: auto;  width: 100%                                  │
│                                                                │
│  ┌─ Inner Spacer (creates total scroll height) ─────────────┐ │
│  │  height: max(columnHeights)  (e.g., 45,000px)            │ │
│  │  No visible content — just occupies space for scrollbar   │ │
│  │                                                           │ │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                      │ │
│  │  │Pin 0 │ │Pin 1 │ │Pin 2 │ │Pin 3 │ ← Absolutely          │ │
│  │  │pos:  │ │pos:  │ │pos:  │ │pos:  │   positioned           │ │
│  │  │0,0   │ │0,252 │ │0,504 │ │0,756 │   rendered items       │ │
│  │  │      │ └──────┘ │      │ │      │                        │ │
│  │  │300px │ ┌──────┐ │400px │ │220px │                        │ │
│  │  └──────┘ │Pin 4 │ └──────┘ └──────┘                        │ │
│  │  ┌──────┐ │250px │         ┌──────┐                          │ │
│  │  │Pin 5 │ └──────┘         │Pin 6 │                          │ │
│  │  └──────┘                  └──────┘                          │ │
│  └───────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

##### Implementation

```tsx
import { useMemo, useRef, useState, useEffect, useCallback } from 'react';

function MasonryGrid({ pins }: { pins: Pin[] }) {
  const containerRef = useRef<HTMLDivElement>(null);
  const [containerWidth, setContainerWidth] = useState(0);

  // Measure container width (responsive)
  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    const observer = new ResizeObserver(([entry]) => {
      setContainerWidth(entry.contentRect.width);
    });
    observer.observe(container);
    return () => observer.disconnect();
  }, []);

  // Calculate layout
  const { positions, totalHeight, columnCount, columnWidth } = useMemo(() => {
    if (containerWidth === 0) return { positions: [], totalHeight: 0, columnCount: 0, columnWidth: 0 };

    const gap = 16;
    const minColumnWidth = 236;
    const count = Math.max(2, Math.floor((containerWidth + gap) / (minColumnWidth + gap)));
    const width = (containerWidth - gap * (count - 1)) / count;

    const { positions, totalHeight } = calculateMasonryLayout(pins, count, width, gap);

    return { positions, totalHeight, columnCount: count, columnWidth: width };
  }, [pins, containerWidth]);

  return (
    <div
      ref={containerRef}
      className="masonry-container"
      style={{ position: 'relative', width: '100%' }}
    >
      {/* Spacer — creates total scrollable height */}
      <div style={{ height: totalHeight, position: 'relative' }}>
        {pins.map((pin, index) => {
          const pos = positions[index];
          if (!pos) return null;

          return (
            <div
              key={pin.id}
              style={{
                position: 'absolute',
                // Use transform for GPU-accelerated positioning
                transform: `translate(${pos.left}px, ${pos.top}px)`,
                width: pos.width,
                height: pos.height,
              }}
            >
              <PinCard pin={pin} />
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

##### Why `transform` Instead of `top` / `left`

| Property | How browser handles it | Performance |
|----------|----------------------|-------------|
| `top` / `left` | Changes trigger **layout recalculation** for the element and potentially siblings | Slower — layout is a main-thread operation |
| `transform: translate()` | Changes are handled by the **compositor** — GPU-accelerated, runs off-main-thread | Faster — no layout recalculation; only a composite step |

Using `transform` means that when we reposition pins (e.g., on window resize or new batch), the browser only needs to re-composite — not re-layout the entire grid.

##### Performance of Layout Calculation

The `calculateMasonryLayout` function runs in **O(n × c)** time where `n` is the number of pins and `c` is the column count. For typical values:
*   100 pins × 4 columns = 400 operations (< 1ms)
*   1000 pins × 6 columns = 6000 operations (< 5ms)

This is well within the 16ms frame budget. The calculation is pure JavaScript (no DOM access), so it runs entirely on the main thread without causing layout thrashing.

---

#### 5.2.5 Responsive Column Count

Pinterest dynamically adjusts the number of columns based on viewport width:

```
Mobile (< 640px):     2 columns, card width ~160px
Tablet (640-1024px):  3-4 columns, card width ~200px
Desktop (1024-1440px): 4-5 columns, card width ~236px
Ultrawide (> 1440px):  5-7 columns, card width ~236px (max width capped)
```

##### Why Not Simply Media Queries?

CSS media queries can change `column-count`, but JavaScript masonry requires the layout engine to know the column count to compute positions. Using `ResizeObserver` + JavaScript gives us:
*   Exact pixel-level breakpoints (not just CSS breakpoints).
*   Dynamic column width calculation based on actual container width.
*   Instant re-layout when the window resizes or side panels open/close.

##### Implementation

```tsx
function useColumnLayout(containerWidth: number) {
  return useMemo(() => {
    const gap = 16;
    const minColumnWidth = 236;  // Pinterest's standard card width
    const maxColumnWidth = 280;  // Don't stretch cards too wide

    // Calculate optimal column count
    let columnCount = Math.max(2, Math.floor((containerWidth + gap) / (minColumnWidth + gap)));

    // Calculate actual column width from available space
    let columnWidth = (containerWidth - gap * (columnCount - 1)) / columnCount;

    // If columns are too wide, add another column
    if (columnWidth > maxColumnWidth && columnCount < 8) {
      columnCount += 1;
      columnWidth = (containerWidth - gap * (columnCount - 1)) / columnCount;
    }

    return { columnCount, columnWidth, gap };
  }, [containerWidth]);
}
```

##### What Happens on Resize

```
User resizes browser from 1400px → 1000px
  ↓
1. ResizeObserver fires with new container width (1000px)
  ↓
2. Column count recalculates: 5 cols → 4 cols
  ↓
3. All pin positions recalculate (O(n) — fast for even 500 pins)
  ↓
4. React re-renders — pins animate to new positions via:
   - CSS transition on `transform` (smooth slide to new position)
   - Or instant snap (simpler, what Pinterest actually does)
  ↓
5. Total grid height changes (recalculated from new column heights)
```

**Should we animate the re-layout?**
*   Animation looks smooth but can cause jank with 200+ pin cards transitioning simultaneously.
*   Pinterest **does not animate** — pins snap to their new positions instantly on resize. The visual change is fast enough that animation isn't needed.
*   If animation is desired, use `will-change: transform` on pin cards and CSS `transition: transform 0.3s ease`.

---

#### 5.2.6 Virtualized Masonry Grid for Infinite Scroll

##### The Problem — Unbounded DOM Growth

Without virtualization, every pin loaded via infinite scroll stays in the DOM forever:
*   After scrolling 50 pages × 25 pins = **1,250 pin cards** in the DOM.
*   Each card has ~10-15 DOM nodes (image, overlay, footer, buttons) = **~15,000+ DOM nodes**.
*   Each image consumes decoded bitmap memory (~1MB for a 236×400 pin image).
*   1,250 images decoded = **~1.25GB of GPU memory** — devices will crash.

##### The Challenge — Masonry + Virtualization Is Harder Than Flat Lists

Virtualizing a masonry grid is **significantly harder** than virtualizing a flat list because:

1. **Items are not in rows.** In a flat list, you can calculate visible items from `scrollTop / rowHeight`. In masonry, each column has a different height, and items are scattered at arbitrary `top` positions.

2. **Visible range must consider all columns.** A viewport window at `scrollTop = 2000px` might show items from different "generations" of the grid — an item placed early (in a short column) might be at `top: 1800px`, while an item placed later (in a tall column) might also be at `top: 1800px`.

3. **Total height changes as items load.** Each new batch extends column heights unevenly.

##### The Approach — Pre-Calculated Position Lookup

Since we already calculate positions for all pins via `calculateMasonryLayout`, we can use those positions to determine visibility:

```
Pre-calculated positions:
  Pin 0: { top: 0, height: 300 }
  Pin 1: { top: 0, height: 180 }
  Pin 2: { top: 0, height: 400 }
  ...
  Pin 200: { top: 8500, height: 250 }

Viewport: scrollTop = 2000, viewportHeight = 900
Visible range: top = 2000 - overscan, bottom = 2900 + overscan
                    = 1500 to 3400

Visible pins: all pins where (pin.top + pin.height > 1500) AND (pin.top < 3400)
```

##### Implementation — Virtualized Masonry

```tsx
function VirtualizedMasonryGrid({ pins }: { pins: Pin[] }) {
  const containerRef = useRef<HTMLDivElement>(null);
  const [containerWidth, setContainerWidth] = useState(0);
  const [scrollTop, setScrollTop] = useState(0);
  const [viewportHeight, setViewportHeight] = useState(window.innerHeight);

  // ResizeObserver for container width
  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;
    const obs = new ResizeObserver(([entry]) => {
      setContainerWidth(entry.contentRect.width);
      setViewportHeight(entry.contentRect.height);
    });
    obs.observe(container);
    return () => obs.disconnect();
  }, []);

  // Scroll listener (passive, RAF-batched)
  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    let ticking = false;
    const handleScroll = () => {
      if (!ticking) {
        requestAnimationFrame(() => {
          setScrollTop(container.scrollTop);
          ticking = false;
        });
        ticking = true;
      }
    };

    container.addEventListener('scroll', handleScroll, { passive: true });
    return () => container.removeEventListener('scroll', handleScroll);
  }, []);

  // Calculate all positions (O(n) — runs on pin list change or resize)
  const { positions, totalHeight, columnWidth } = useMemo(() => {
    if (containerWidth === 0) return { positions: [], totalHeight: 0, columnWidth: 0 };
    const gap = 16;
    const count = Math.max(2, Math.floor((containerWidth + gap) / (236 + gap)));
    const width = (containerWidth - gap * (count - 1)) / count;
    const result = calculateMasonryLayout(pins, count, width, gap);
    return { ...result, columnWidth: width };
  }, [pins, containerWidth]);

  // Determine visible pins (O(n) scan — optimizable with spatial index)
  const overscan = 500; // px above and below viewport
  const visiblePins = useMemo(() => {
    const top = scrollTop - overscan;
    const bottom = scrollTop + viewportHeight + overscan;

    const visible: number[] = [];
    for (let i = 0; i < positions.length; i++) {
      const pos = positions[i];
      // Pin is visible if any part of it is within the visible range
      if (pos.top + pos.height > top && pos.top < bottom) {
        visible.push(i);
      }
    }
    return visible;
  }, [positions, scrollTop, viewportHeight]);

  return (
    <div
      ref={containerRef}
      style={{ height: '100vh', overflow: 'auto', position: 'relative' }}
    >
      {/* Spacer — total grid height */}
      <div style={{ height: totalHeight, position: 'relative' }}>
        {/* Only render visible pins */}
        {visiblePins.map((index) => {
          const pin = pins[index];
          const pos = positions[index];

          return (
            <div
              key={pin.id}
              style={{
                position: 'absolute',
                transform: `translate(${pos.left}px, ${pos.top}px)`,
                width: pos.width,
                height: pos.height,
              }}
            >
              <PinCard pin={pin} />
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

##### Optimizing Visible Pin Lookup

The naive O(n) scan through all positions on every scroll works fine for up to ~1,000 pins. Beyond that, we can optimize:

**Option 1: Binary Search on Sorted Positions**

Since positions are roughly sorted by `top` (earlier pins tend to have smaller top values, though not strictly), we can partition pins into "bands" by top position.

**Option 2: Spatial Bucketing**

Divide the total height into fixed-size buckets (e.g., 500px each). Each bucket contains the indices of pins that overlap that vertical range. On scroll, only check pins in the relevant buckets.

```tsx
function buildSpatialIndex(positions: PinPosition[], bucketSize: number = 500) {
  const buckets = new Map<number, number[]>();

  for (let i = 0; i < positions.length; i++) {
    const pos = positions[i];
    const startBucket = Math.floor(pos.top / bucketSize);
    const endBucket = Math.floor((pos.top + pos.height) / bucketSize);

    for (let b = startBucket; b <= endBucket; b++) {
      if (!buckets.has(b)) buckets.set(b, []);
      buckets.get(b)!.push(i);
    }
  }

  return buckets;
}

// On scroll: only check pins in visible buckets
function getVisiblePins(
  buckets: Map<number, number[]>,
  scrollTop: number,
  viewportHeight: number,
  overscan: number,
  bucketSize: number
): Set<number> {
  const top = scrollTop - overscan;
  const bottom = scrollTop + viewportHeight + overscan;
  const startBucket = Math.floor(top / bucketSize);
  const endBucket = Math.floor(bottom / bucketSize);

  const visible = new Set<number>();
  for (let b = startBucket; b <= endBucket; b++) {
    const indices = buckets.get(b) || [];
    for (const i of indices) {
      visible.add(i);
    }
  }
  return visible;
}
```

This reduces per-scroll computation from O(n) to O(visible pins + bucket overhead) — typically **O(50-100)** regardless of total pin count.

##### Performance Impact

| Metric | No virtualization (1000 pins) | Virtualized (overscan=500px) |
|--------|-------------------------------|------------------------------|
| DOM nodes | ~15,000 | ~300-600 (20-40 visible pins) |
| Decoded images in memory | ~1GB+ | ~30-60MB (20-40 images) |
| Initial render time (after layout calc) | 200-500ms | 20-50ms |
| Scroll smoothness | Degrades after ~200 pins | Consistent 60fps |
| Layout calculation time | ~5ms for 1000 pins | Same (positions pre-calculated for all) |
| Memory growth over session | Unbounded | Bounded to viewport |

---

#### 5.2.7 Image Loading and Placeholder Strategy

Images are the **primary content** in Pinterest. Every pin card is essentially an image with metadata. Getting image loading right is critical because:
*   Pin images vary enormously in size (100KB for small illustrations to 2MB+ for high-res photos).
*   The masonry grid can display 20-40 images simultaneously in the viewport.
*   Slow image loading creates a "blank grid" that feels broken.
*   Layout shift (CLS) from images loading with unknown dimensions destroys the masonry layout.

##### Why Pre-Known Dimensions Are Critical

The masonry algorithm **must know each pin's aspect ratio before the image loads**. Without it:
*   The layout positions would be wrong (calculated with estimated heights).
*   When images load, their real heights differ from estimates → all subsequent pins shift → massive CLS.
*   Pinterest's API **always** includes `width` and `height` in pin data for this exact reason.

```tsx
// API response includes dimensions — layout is calculated immediately
{
  "id": "pin_123",
  "image": {
    "url": "https://cdn.example.com/pins/pin_123.jpg",
    "width": 600,
    "height": 900,
    "dominantColor": "#3a7bd5",
    "blurhash": "LEHV6nWB2yk8pyo0adR*.7kCMdnj"
  }
}
```

##### Placeholder Strategies for Pin Images

| Strategy | How it works | Visual result | Performance |
|----------|-------------|---------------|-------------|
| **Aspect ratio box + dominant color** | Set `background-color` to `dominantColor`; reserve space with `aspect-ratio` | Colored rectangle → image fades in | Fast; 7 bytes of data per pin (hex color) |
| **Blurhash placeholder** | Decode blurhash string to a tiny canvas image; show blurred until real image loads | Blurred preview of image → sharp image | ~1-3ms decode time; 20-30 byte string per pin |
| **Low Quality Image Placeholder (LQIP)** | Show a tiny (16x16) base64 thumbnail, blurred; then load full image | Very blurred thumbnail → sharp | ~200 bytes per pin (inline base64) |
| **Skeleton shimmer** | CSS animated grey box | Grey shimmer → image | No extra data needed; generic look |

**Pinterest uses dominant color + fade-in** as the primary strategy. It is the best tradeoff: minimal data overhead, instant rendering, and a visually coherent placeholder that matches the image's color palette.

##### Implementation

```tsx
function PinImage({ image, eager = false }: { image: PinImageData; eager?: boolean }) {
  const [loaded, setLoaded] = useState(false);

  return (
    <div
      style={{
        // Reserve exact space using aspect ratio — prevents CLS
        aspectRatio: `${image.width} / ${image.height}`,
        backgroundColor: image.dominantColor || '#e0e0e0',
        borderRadius: '16px',
        overflow: 'hidden',
        position: 'relative',
      }}
    >
      <img
        src={image.url}
        alt={image.altText || 'Pin image'}
        loading={eager ? 'eager' : 'lazy'}
        onLoad={() => setLoaded(true)}
        style={{
          width: '100%',
          height: '100%',
          objectFit: 'cover',
          opacity: loaded ? 1 : 0,
          transition: 'opacity 0.3s ease',
        }}
      />
    </div>
  );
}
```

##### Image Size Variants via CDN

Pinterest serves images at multiple widths via CDN URL parameters:

```
Thumbnail (grid card):  https://cdn.example.com/pins/pin_123.jpg?w=236
Medium (2x retina):     https://cdn.example.com/pins/pin_123.jpg?w=474
Full size (detail):     https://cdn.example.com/pins/pin_123.jpg?w=736
Original:               https://cdn.example.com/pins/pin_123.jpg
```

For the grid, we use the **`?w=236` variant** (matching the column width). For retina displays, we use `srcset`:

```tsx
<img
  src={`${image.url}?w=236&q=80`}
  srcSet={`${image.url}?w=236&q=80 1x, ${image.url}?w=474&q=75 2x`}
  alt={image.altText}
  loading="lazy"
/>
```

##### First Batch Loading Strategy

```
First 8-12 pins (above the fold, 2 rows):
  → loading="eager" — load immediately for fast LCP
  → Preconnect to CDN: <link rel="preconnect" href="https://cdn.example.com">

Remaining pins (below the fold):
  → loading="lazy" — browser defers until near viewport

Virtualized off-screen pins:
  → Not in DOM — no <img> tag exists — zero load cost
```

---

#### 5.2.8 Scroll Performance Considerations

| Concern | What happens | Mitigation |
|---------|-------------|------------|
| **Layout recalc on batch append** | New batch of 25 pins triggers `calculateMasonryLayout` for the entire grid | Only recalculate positions for new pins; append to existing positions array. Column heights carry over from previous batch. |
| **Scroll event flooding** | Scroll listener fires hundreds of times per second | Use `requestAnimationFrame` batching; only update `scrollTop` once per frame |
| **Image decode jank** | Large pin images decoding on main thread during scroll | Use `loading="lazy"` to defer; use `img.decode()` for above-fold images; consider `content-visibility: auto` |
| **Hover overlay paint cost** | Each pin card has a hover overlay with opacity transition | Use `will-change: opacity` on the overlay element; only render overlay on hover (not in DOM until needed) |
| **Resize reflow** | Window resize triggers full position recalculation + re-render | Debounce resize (200ms); recalculate positions in `useMemo` (pure JS, fast); use `transform` for positioning |
| **GPU memory exhaustion** | 200+ decoded images consume > 500MB of GPU memory | Virtualize — only images in/near viewport are in the DOM and decoded |

##### Incremental Layout Calculation on New Batches

Instead of recalculating positions for all pins when a new batch loads, we can **continue** from where the previous calculation left off:

```tsx
function appendToMasonryLayout(
  existingPositions: PinPosition[],
  existingColumnHeights: number[],
  newPins: Pin[],
  columnCount: number,
  columnWidth: number,
  gap: number,
  footerHeight: number = 40
): { newPositions: PinPosition[]; updatedColumnHeights: number[] } {
  const columnHeights = [...existingColumnHeights]; // copy
  const newPositions: PinPosition[] = [];

  for (const pin of newPins) {
    const aspectRatio = pin.image.height / pin.image.width;
    const cardHeight = aspectRatio * columnWidth + footerHeight;

    let shortestColumn = 0;
    let minHeight = columnHeights[0];
    for (let i = 1; i < columnCount; i++) {
      if (columnHeights[i] < minHeight) {
        minHeight = columnHeights[i];
        shortestColumn = i;
      }
    }

    newPositions.push({
      top: columnHeights[shortestColumn],
      left: shortestColumn * (columnWidth + gap),
      width: columnWidth,
      height: cardHeight,
    });

    columnHeights[shortestColumn] += cardHeight + gap;
  }

  return { newPositions, updatedColumnHeights: columnHeights };
}
```

This is **O(newBatchSize)** instead of **O(totalPins)** — critical for maintaining smooth infinite scroll with large datasets.

---

#### 5.2.9 Decision Matrix

| Scenario | Strategy | Tool / Approach | Notes |
|----------|----------|----------------|-------|
| **< 30 pins** (board page, search results) | CSS Grid with `grid-row: span` | CSS + minimal JS for span calc | Good enough; no virtualization needed |
| **30-100 pins** (bounded grid) | JS masonry with absolute positioning | Custom `calculateMasonryLayout` | Simple; no virtualization needed |
| **100+ pins** (infinite scroll) | Virtualized JS masonry | Custom virtualizer with spatial bucketing | Keep DOM at O(viewport) |
| **Responsive resize** | ResizeObserver + full re-layout | `useMemo` + `transform` positioning | Instant snap (no animation) |
| **Image loading** | Dominant color placeholder + lazy loading | `loading="lazy"` + `aspectRatio` CSS | First 8-12 images `loading="eager"` |
| **Retina displays** | `srcset` with 2x variant | CDN resizing (`?w=236` / `?w=474`) | Auto browser selection |
| **Mobile (2 columns)** | Same JS masonry, fewer columns | Responsive column count from container width | Touch scroll; no hover overlays |
| **Slow network** | Dominant color immediate + progressive image load | API provides `dominantColor` per pin | Prevents blank grid |

---

### 5.3 Reusability Strategy

*   **`PinCard`**: Generic pin card component; configurable for home grid, board grid, search results, and related pins. Accepts `showHoverOverlay`, `showFooter`, `size` props.
*   **`MasonryGrid`**: Reusable masonry layout engine; used for home feed, board view, search results, and "More like this" sections. Accepts `pins`, `columnMinWidth`, `gap` as props.
*   **`PinImage`**: Shared image component with lazy loading, placeholder, fade-in. Used in pin cards, pin detail, board thumbnails.
*   **`InfiniteScrollSentinel`**: Generic sentinel + IntersectionObserver hook, reusable for any paginated list.
*   **`BoardPicker`**: Dropdown component for selecting a board on save; reusable from grid hover overlay, pin detail, and extension.
*   Design system tokens for colors, spacing, typography, border-radius (Pinterest's signature rounded corners).

---

### 5.4 Module Organization

```
src/
 ├── features/
 │    ├── grid/
 │    │    ├── components/
 │    │    │    ├── MasonryGrid.tsx
 │    │    │    ├── VirtualizedMasonryGrid.tsx
 │    │    │    ├── PinCard.tsx
 │    │    │    ├── PinImage.tsx
 │    │    │    ├── PinHoverOverlay.tsx
 │    │    │    ├── PinFooter.tsx
 │    │    │    ├── GridSentinel.tsx
 │    │    │    └── GridSkeleton.tsx
 │    │    ├── hooks/
 │    │    │    ├── useMasonryLayout.ts
 │    │    │    ├── useColumnLayout.ts
 │    │    │    ├── useGridInfiniteScroll.ts
 │    │    │    └── useGridVirtualizer.ts
 │    │    ├── utils/
 │    │    │    ├── masonryCalculator.ts
 │    │    │    └── spatialIndex.ts
 │    │    └── types/
 │    │         └── grid.types.ts
 │    ├── pin-detail/
 │    │    ├── components/
 │    │    │    ├── PinDetailModal.tsx
 │    │    │    ├── PinDetailImage.tsx
 │    │    │    ├── PinMetadata.tsx
 │    │    │    ├── RelatedPinsGrid.tsx
 │    │    │    └── CommentSection.tsx
 │    │    ├── hooks/
 │    │    │    └── usePinDetail.ts
 │    │    └── api/
 │    │         └── pinDetailApi.ts
 │    ├── boards/
 │    │    ├── components/
 │    │    │    ├── BoardGrid.tsx
 │    │    │    ├── BoardCard.tsx
 │    │    │    └── BoardPicker.tsx
 │    │    ├── hooks/
 │    │    │    └── useBoardManagement.ts
 │    │    └── api/
 │    │         └── boardApi.ts
 │    └── search/
 │         ├── components/
 │         │    ├── SearchBar.tsx
 │         │    └── SearchResultsGrid.tsx
 │         └── hooks/
 │              └── useSearch.ts
 ├── shared/
 │    ├── components/ (Avatar, Button, Modal, Skeleton, LazyImage)
 │    ├── hooks/ (useIntersection, useDebounce, useResizeObserver)
 │    └── utils/ (dateFormat, numberFormat, imageUrl)
```

---

## 6. High Level Data Flow Explanation

### 6.1 Initial Load Flow

```
1. User navigates to Pinterest home (or opens app)
     ↓
2. Server renders page shell via SSR (nav bar + skeleton grid + first 25 pins)
     ↓
3. Client hydrates → MasonryGrid mounts
     ↓
4. ResizeObserver measures container width → column count determined
     ↓
5. calculateMasonryLayout runs → positions computed for 25 pins
     ↓
6. Pins render at absolute positions; images start loading
     ↓
7. First 8-12 images (above fold) load eagerly → LCP fires
     ↓
8. GridSentinel mounts at bottom → IntersectionObserver watches
     ↓
9. User scrolls → sentinel enters rootMargin → fetch page 2
     ↓
10. New batch positions are appended (incremental calculation)
```

---

### 6.2 User Interaction Flow

**Save Pin to Board — Optimistic Update**

```
User hovers pin → overlay appears → taps "Save"
     ↓
1. Board picker dropdown opens
     ↓
2. User selects "Home Decor" board
     ↓
3. Immediately update local state:
   - pin.isSaved = true
   - pin.savedToBoard = "Home Decor"
   - Save button changes to "Saved ✓"
     ↓
4. Fire API call (background):
   POST /api/pin/:id/save { boardId: "board_123" }
     ↓
5a. API succeeds → done (state already correct)
     ↓
5b. API fails → ROLLBACK local state:
   - pin.isSaved = false
   - Show error toast: "Failed to save pin"
```

**Implementation:**

```tsx
function useOptimisticSave(pinId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ boardId }: { boardId: string }) => savePin(pinId, boardId),
    onMutate: async ({ boardId }) => {
      await queryClient.cancelQueries({ queryKey: ['grid'] });
      const previousGrid = queryClient.getQueryData(['grid']);

      queryClient.setQueryData(['grid'], (old: GridData) => ({
        ...old,
        pins: old.pins.map((pin) =>
          pin.id === pinId
            ? { ...pin, isSaved: true, savedToBoardId: boardId }
            : pin
        ),
      }));

      return { previousGrid };
    },
    onError: (err, variables, context) => {
      queryClient.setQueryData(['grid'], context.previousGrid);
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['grid'] });
    },
  });
}
```

**Pin Detail Modal Flow:**

```
User taps pin card in grid
     ↓
1. Lazy-load PinDetailModal chunk (React.lazy)
     ↓
2. Modal opens as overlay (React Portal)
     ↓
3. Fetch pin detail: GET /api/pin/:id
   (includes full description, source URL, comments, related pin IDs)
     ↓
4. Render full image, metadata, comments
     ↓
5. Below the detail, fetch and render "More like this" pins in a mini masonry grid
     ↓
6. User closes modal → focus returns to the pin card in the grid
```

---

### 6.3 Error and Retry Flow

*   **Grid feed fetch fails**: Show cached grid (React Query stale-while-revalidate) + error banner with retry button.
*   **Image fails to load**: Keep the dominant color placeholder visible; show a subtle "Image unavailable" icon. Don't collapse the card (it would shift the entire grid).
*   **Save API fails**: Rollback optimistic save; show toast "Failed to save — tap to retry."
*   **Pin detail fetch fails**: Show modal with error state and retry button; don't close the modal.
*   **Infinite scroll fetch fails**: Show "Failed to load more pins" inline at the bottom with retry button. Don't block existing grid.
*   **Network offline**: Show cached grid + "You're offline" banner. Queue save actions and replay when online.

---

## 7. Data Modelling (Frontend Perspective)

### 7.1 Core Data Entities

*   **Pin** — a single image card in the grid with metadata.
*   **Board** — a user's collection of saved pins, organized by theme.
*   **User** — creator or saver of a pin (lightweight profile data).
*   **PinImage** — image metadata including dimensions, CDN URL, and placeholders.
*   **Comment** — a text reply on a pin detail page.

---

### 7.2 Data Shape

```ts
type User = {
  id: string;
  name: string;
  avatarUrl: string;
  profileUrl: string;
};

type PinImage = {
  url: string;                     // CDN base URL (append ?w=236 for sizes)
  width: number;                   // original width (critical for aspect ratio)
  height: number;                  // original height (critical for aspect ratio)
  dominantColor: string;           // hex color for placeholder
  blurhash?: string;               // optional blur-up placeholder
  altText?: string;
};

type Pin = {
  id: string;
  image: PinImage;
  title?: string;                  // optional — many pins have no title
  description?: string;
  sourceUrl?: string;              // link to original source
  sourceDomain?: string;           // "example.com" — shown on card
  author: User;
  isSaved: boolean;
  savedToBoardId?: string;
  saveCount: number;
  commentCount: number;
  createdAt: string;
  tags?: string[];
};

type Board = {
  id: string;
  name: string;
  description?: string;
  coverImageUrl?: string;
  pinCount: number;
  isSecret: boolean;
  owner: User;
  createdAt: string;
};

type Comment = {
  id: string;
  author: User;
  text: string;
  createdAt: string;
  likeCount: number;
  isLiked: boolean;
};
```

---

### 7.3 Entity Relationships

*   **Many-to-Many**: Pins ↔ Boards (a pin can be saved to many boards; a board contains many pins).
*   **Many-to-One**: Many Pins → one User (author/creator).
*   **One-to-Many**: One Pin → many Comments.
*   **One-to-One**: One Pin → one PinImage.

**Normalized vs Denormalized:**

| Approach | Usage |
|----------|-------|
| **Denormalized** | Pin includes inline `author: { id, name, avatarUrl }`. Self-contained for rendering. This is what the grid needs. |
| **Normalized** | Board references `pinIds: string[]`; separate `pins` map. Useful for managing boards where pins appear in multiple boards. |

**Recommendation:** Denormalized for the grid feed (each pin carries its own author data). Board management may use normalized references.

---

### 7.4 UI Specific Data Models

```ts
// Computed position from masonry layout
type PinPosition = {
  top: number;
  left: number;
  width: number;
  height: number;
};

// Grid page state
type GridFeedState = {
  pins: Pin[];
  positions: PinPosition[];
  columnHeights: number[];
  cursor: string | null;
  hasMore: boolean;
  isLoading: boolean;
  error: string | null;
};

// Pin detail modal state
type PinDetailState = {
  isOpen: boolean;
  pinId: string | null;
  pinData: PinDetail | null;
  relatedPins: Pin[];
  isLoading: boolean;
};

// Board picker state (used when saving a pin)
type BoardPickerState = {
  isOpen: boolean;
  targetPinId: string | null;
  boards: Board[];
  recentBoardId: string | null;     // last used board for quick save
};
```

---

## 8. State Management Strategy

### 8.1 State Classification

| State Type | Examples | Storage |
|---|---|---|
| **Server State** | Grid pins, pin detail, board lists, comments | React Query / SWR cache (auto-managed) |
| **Global App State** | Current user session, theme, board list for save picker | Zustand / Redux global store |
| **Feature State** | Masonry layout positions, column heights, grid page cursor | Feature-level Zustand slice or component state |
| **Component Local State** | Hover overlay visible, board picker open, image loaded | `useState` / `useReducer` |
| **Derived / Computed State** | Visible pin indices (from scroll position + positions), column count | `useMemo` with dependencies |

---

### 8.2 State Ownership

*   **HomePage** owns the infinite scroll state (cursor, hasMore) via React Query's `useInfiniteQuery`.
*   **MasonryGrid** owns the layout positions and column heights. It recalculates on pin list change or container resize.
*   **VirtualizedMasonryGrid** additionally owns scroll state and visible pin computation.
*   **PinCard** owns its own hover state and image loaded state locally.
*   **PinDetailModal** owns its own visibility and fetches pin detail data independently via `useQuery`.
*   **BoardPicker** reads from a global board list store (Zustand) and manages dropdown open/close locally.

**Prop drilling is minimal:**
*   Grid passes `pin` object and `position` to each `PinCard`.
*   Save triggers update the React Query cache directly (optimistic update).
*   Board picker reads the global board list via `useBoardStore()`.

---

### 8.3 Persistence Strategy

| Data | Persistence | Reason |
|------|------------|--------|
| Grid pins (API data) | React Query in-memory cache | Ephemeral; refetch on focus/revisit; `staleTime: 5min` |
| Masonry positions | Component state (in-memory) | Recomputed on mount; no persistence needed |
| Board list | React Query cache + global store | Shared across components; cached for board picker |
| Recent board (last saved to) | localStorage | Quick save to same board without re-selecting |
| Scroll position | sessionStorage | Survive back-navigation from pin detail; 30-min TTL |
| User preferences (grid density) | localStorage | Persist across sessions |
| Offline pin cache (PWA) | IndexedDB via Service Worker | Show cached pins when offline |

---

## 9. High Level API Design (Frontend POV)

### 9.1 Required APIs

| API | Method | Description |
|-----|--------|-------------|
| `/api/feed` | **GET** | Fetch home feed pins (cursor-based, personalized) |
| `/api/search` | **GET** | Search pins by query (cursor-based) |
| `/api/pin/:id` | **GET** | Fetch single pin with full detail |
| `/api/pin/:id/related` | **GET** | Fetch related / similar pins |
| `/api/pin/:id/save` | **POST** | Save pin to a board |
| `/api/pin/:id/unsave` | **DELETE** | Remove pin from a board |
| `/api/pin/:id/comments` | **GET** | Fetch comments for a pin |
| `/api/pin/:id/comment` | **POST** | Add a comment to a pin |
| `/api/boards` | **GET** | Fetch user's boards (for board picker) |
| `/api/board/:id/pins` | **GET** | Fetch pins in a specific board |
| `/api/board` | **POST** | Create a new board |

---

### 9.2 Cursor Based Pagination for Grid Feed (Deep Dive)

#### Why Cursor for Masonry Grids

The same arguments for cursor-based pagination in feeds apply here (see Facebook Social Feed article), but masonry grids add extra sensitivity:

*   **Duplicate pins are especially noticeable** in a grid layout. In a vertical feed, a duplicate post might be 10 screens apart. In a compact grid, duplicates could appear **side by side** if offset pagination shifts results.
*   **Pin ranking changes frequently** — the backend continuously re-ranks based on engagement and recency. Offset-based pagination would produce inconsistent pages.

#### Implementation with React Query

```tsx
function useGridFeed() {
  return useInfiniteQuery({
    queryKey: ['grid', 'home'],
    queryFn: async ({ pageParam }) => {
      const url = pageParam
        ? `/api/feed?cursor=${pageParam}&limit=25`
        : `/api/feed?limit=25`;
      const res = await fetch(url);
      return res.json();
    },
    initialPageParam: null as string | null,
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    staleTime: 5 * 60 * 1000,       // 5 minutes before refetch
    refetchOnWindowFocus: true,
  });
}

// Usage in grid
function HomePage() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useGridFeed();
  const allPins = data?.pages.flatMap((page) => page.pins) ?? [];

  return (
    <>
      <VirtualizedMasonryGrid pins={allPins} />
      {hasNextPage && <GridSentinel onIntersect={fetchNextPage} />}
      {isFetchingNextPage && <GridLoadingSkeleton />}
    </>
  );
}
```

#### Batch Size for Masonry Grids

| Column Count | Visible Pins (approx) | Recommended Batch Size | Reasoning |
|-------------|----------------------|----------------------|-----------|
| 2 (mobile) | ~4-6 | 15-20 | ~3-4 screens worth |
| 4 (desktop) | ~8-12 | 25-30 | ~2-3 screens worth |
| 6 (ultrawide) | ~12-18 | 25-30 | ~1.5-2 screens worth |

**25 pins per batch is the sweet spot** — enough to fill 2+ screens on most devices, small enough to keep API response times fast (~10-20KB of JSON).

---

### 9.3 Request and Response Structure

**GET /api/feed**

```json
// Request
// GET /api/feed?limit=25

// Response
{
  "pins": [
    {
      "id": "pin_001",
      "image": {
        "url": "https://cdn.example.com/pins/pin_001.jpg",
        "width": 600,
        "height": 900,
        "dominantColor": "#3a7bd5",
        "altText": "Minimalist living room with white sofa"
      },
      "title": "30 Minimalist Living Room Ideas",
      "description": "Explore clean, modern living room designs...",
      "sourceUrl": "https://example.com/article/minimalist-rooms",
      "sourceDomain": "example.com",
      "author": {
        "id": "u_123",
        "name": "Zeeshan Ali",
        "avatarUrl": "https://cdn.example.com/avatars/u_123.jpg",
        "profileUrl": "/user/zeeshan_ali"
      },
      "isSaved": false,
      "saveCount": 1523,
      "commentCount": 42,
      "createdAt": "2026-03-15T10:00:00Z",
      "tags": ["home-decor", "minimalist", "living-room"]
    }
  ],
  "nextCursor": "eyJzY29yZSI6MC44NywidHMiOiIyMDI2LTAzLTE1In0=",
  "hasMore": true
}
```

**POST /api/pin/:id/save**

```json
// Request
{
  "boardId": "board_456"
}

// Response
{
  "success": true,
  "savedPin": {
    "pinId": "pin_001",
    "boardId": "board_456",
    "savedAt": "2026-03-17T09:00:00Z"
  }
}
```

**GET /api/pin/:id**

```json
// Response
{
  "id": "pin_001",
  "image": {
    "url": "https://cdn.example.com/pins/pin_001.jpg",
    "width": 2400,
    "height": 3600,
    "dominantColor": "#3a7bd5"
  },
  "title": "30 Minimalist Living Room Ideas",
  "description": "A comprehensive guide to creating a clean, modern living room...",
  "sourceUrl": "https://example.com/article/minimalist-rooms",
  "sourceDomain": "example.com",
  "author": {
    "id": "u_123",
    "name": "Zeeshan Ali",
    "avatarUrl": "https://cdn.example.com/avatars/u_123.jpg"
  },
  "isSaved": false,
  "saveCount": 1523,
  "commentCount": 42,
  "comments": [
    {
      "id": "c_001",
      "author": { "id": "u_789", "name": "Sara", "avatarUrl": "..." },
      "text": "Love these ideas!",
      "createdAt": "2026-03-16T14:30:00Z",
      "likeCount": 8,
      "isLiked": false
    }
  ],
  "relatedPinIds": ["pin_055", "pin_112", "pin_087"],
  "createdAt": "2026-03-15T10:00:00Z",
  "tags": ["home-decor", "minimalist", "living-room"]
}
```

---

### 9.4 Error Handling and Status Codes

| Status | Scenario | Frontend Handling |
|--------|----------|-------------------|
| `200` | Success | Render data |
| `304` | Not Modified (cached) | Use cached version |
| `400` | Bad request (invalid cursor) | Show error toast; reset pagination |
| `401` | Unauthorized (token expired) | Redirect to login; clear session |
| `403` | Forbidden (private board) | Show "This board is private" message |
| `404` | Pin deleted or board removed | Remove from grid; show toast "Pin no longer available" |
| `429` | Rate limit exceeded | Exponential backoff + retry; disable further fetches temporarily |
| `500` | Server error | Show error banner with retry; keep existing grid visible |
| `503` | Service unavailable | Show cached grid; retry with backoff |

---

## 10. Caching Strategy

### 10.1 What to Cache

*   **Grid pins (API data)**: Cached with moderate TTL (5 min) — feed changes based on engagement but not as rapidly as social feeds.
*   **Pin detail**: Cached per pin ID; invalidated on save/comment actions.
*   **Board list**: Cached aggressively (boards change infrequently); invalidated on board create/delete.
*   **Pin images**: Cached aggressively by browser and CDN (immutable content-hashed URLs).
*   **User avatars**: Long-lived cache (avatars rarely change).
*   **Search results**: Short cache (2 min); search results evolve quickly.

### 10.2 Where to Cache

| Data | Cache Location | TTL |
|------|----------------|-----|
| Grid pins | React Query in-memory cache | 5 min (`staleTime`) |
| Pin detail | React Query cache (keyed by pinId) | 10 min |
| Board list | React Query + Zustand global store | 30 min |
| Pin images | Browser HTTP cache + CDN | 1 year (`immutable`, content-hashed) |
| Search results | React Query cache (keyed by query) | 2 min |
| Scroll position / layout state | sessionStorage | 30 min |
| Recent board choice | localStorage | Until changed |
| Offline pins (PWA) | IndexedDB via Service Worker | Until next online sync |

### 10.3 Cache Invalidation

*   **Grid feed**: Invalidated on pull-to-refresh; refetch on window focus.
*   **Pin detail**: Invalidated when user saves/unsaves or adds a comment.
*   **Board list**: Invalidated when user creates/deletes/renames a board.
*   **Pin images**: Never invalidated (new image = new URL from CDN).
*   **Search results**: Invalidated on new search query; stale after 2 min.

---

## 11. CDN and Asset Optimization

*   **Media delivery**: All pin images served via CDN with global edge caching and on-the-fly resizing.
*   **Image optimization**:
    *   Serve WebP/AVIF with fallback to JPEG via `Accept` header negotiation or `<picture>` element.
    *   URL-based resizing: `?w=236` for grid thumbnails, `?w=736` for detail view, original for download.
    *   `srcset` for retina: `?w=236 1x, ?w=474 2x`.
    *   API includes `width`, `height`, `dominantColor` for every pin image — prevents CLS and enables instant placeholders.
*   **Avatar images**: Small (48-64px), heavily cached, served at fixed size via CDN.
*   **Cache headers**:
    *   Pin images: `Cache-Control: public, max-age=31536000, immutable` (content-hashed URLs).
    *   Static assets (JS, CSS): Same immutable caching.
    *   API responses: `Cache-Control: private, no-cache` (allow 304).
*   **Compression**: Brotli (preferred) / Gzip for all text responses and static assets.
*   **Preconnect**: `<link rel="preconnect" href="https://cdn.example.com">` in `<head>` to speed up first image load.

---

## 12. Rendering Strategy

*   **Home Grid Page**: **SSR** for the page shell and first batch of 25 pins. This gives:
    *   Fast FCP — user sees the grid layout immediately.
    *   SEO — the home page and pin pages are indexable for organic search.
    *   Social preview — shared Pinterest URLs get proper OG metadata.
*   **Subsequent grid pages**: **CSR** — fetched and appended client-side via infinite scroll.
*   **Pin Detail Modal**: **Lazy-loaded CSR chunk** — `React.lazy(() => import('./PinDetailModal'))` with Suspense skeleton fallback.
*   **Pin Detail Page** (direct URL): **SSR** — for SEO and social sharing. When a user shares a pin URL, the server renders the full pin detail with OG tags.
*   **Search Results**: **CSR** — search is interactive; no SEO benefit from SSR on user-initiated searches.
*   **Media rendering**:
    *   Images: Native `<img>` with `loading="lazy"`, `srcset` for retina, `aspectRatio` CSS for CLS prevention.
    *   No video in the standard grid (pins are images); video pins play on tap in detail view.
*   **Hydration**: After SSR, selective hydration (React 18+) — hydrate hover overlays and save buttons first, defer non-interactive image containers.

---

## 13. Cross Cutting Non Functional Concerns

### 13.1 Security

*   **XSS mitigation**: All user-generated content (pin titles, descriptions, comments) sanitized via DOMPurify before rendering.
*   **CSRF**: All state-changing requests (POST, DELETE) include CSRF token.
*   **Content Security Policy (CSP)**: Restrict `img-src` to known CDN origins. Restrict `script-src` to self + trusted domains.
*   **Token storage**: Auth tokens in HTTP-only cookies (not localStorage).
*   **Source URL validation**: External links from pin `sourceUrl` are validated and displayed with the domain name; user prompted before leaving Pinterest.
*   **Private boards**: Images from secret boards use signed CDN URLs with short expiry.

---

### 13.2 Accessibility

*   **Semantic HTML**:
    *   Grid container: `role="list"` with each pin card as `role="listitem"`.
    *   Pin cards: `<article>` elements with `aria-label` describing the pin.
    *   Images: `alt` text from API (author-provided or AI-generated).
*   **Keyboard navigation**:
    *   Arrow keys move between pins in the grid (left/right for same row, up/down for column).
    *   `Tab` moves to interactive elements within a pin (save button, link).
    *   `Enter` opens pin detail modal.
    *   `Escape` closes modal.
*   **Screen reader support**:
    *   Pin cards: `aria-label="Pin: 30 Minimalist Living Room Ideas by Zeeshan Ali. 1523 saves."`.
    *   Save button: `aria-pressed="true/false"` for saved state.
    *   Infinite scroll: `aria-live="polite"` region announces "25 more pins loaded."
    *   Loading state: `aria-busy="true"` on grid during fetch.
*   **Focus management**:
    *   On modal open, focus moves to the modal.
    *   On modal close, focus returns to the triggering pin card.
    *   Skip link: "Skip to main content" at the top of the page.
*   **Reduced motion**: Respect `prefers-reduced-motion` — disable image fade-in transitions; show images immediately.
*   **High contrast**: Ensure hover overlay text meets WCAG AA contrast ratios. Save button visible against all image backgrounds.

---

### 13.3 Performance Optimization

#### Image Loading Strategy

```
First 8-12 pins (above the fold):
  → loading="eager" — critical for LCP
  → Preconnect to CDN in <head>

Remaining visible pins (below fold):
  → loading="lazy" — browser defers until near viewport

Off-screen pins (virtualized):
  → Not in DOM — zero load cost

Pin detail image:
  → Preload on hover: <link rel="preload" as="image" href="...?w=736">
```

#### Other Optimizations

*   **Code splitting**: Home grid is the main chunk (~80KB). Pin detail modal, board manager, and search are lazy-loaded chunks (30-60KB each).
*   **Incremental masonry layout**: On new batch load, only calculate positions for new pins (O(batch size), not O(total pins)).
*   **Spatial bucketing for virtualization**: O(1) visible pin lookup instead of O(n) scan on every scroll.
*   **Debounced resize**: Container resize triggers layout recalculation debounced at 200ms.
*   **`requestAnimationFrame` for scroll-linked updates**: Scroll position for virtualization updates at most once per frame.
*   **Memoization**: `React.memo` on `PinCard` — pin data rarely changes during a session. `useMemo` on masonry positions.
*   **Image preload on hover** (pin detail): When user hovers a pin for 200ms, preload the full-size image for the detail view.
*   **Abort stale fetches**: On search query change, abort the previous search request via `AbortController`.

---

### 13.4 Observability and Reliability

*   **Error Boundaries**: Wrap each `PinCard` in an error boundary — if one pin's rendering crashes, the rest of the grid remains functional. Show a "Something went wrong" placeholder in that card's position.
*   **Logging and Analytics**:
    *   **Impressions**: Track which pins enter the viewport (IntersectionObserver on each visible card) — critical for recommendation ranking.
    *   **Click-through**: Track pin detail opens and external link clicks.
    *   **Save events**: Track save/unsave with board context.
    *   **Scroll depth**: How many pins the user scrolled past (pages loaded × batch size).
    *   **Image load failures**: Log URL, error type, and network conditions for CDN monitoring.
*   **Performance monitoring**:
    *   Track FCP, LCP, CLS, and TTI for the grid page.
    *   Track masonry layout calculation time per batch.
    *   Track infinite scroll latency: time from sentinel intersection to new pins rendered.
    *   Track image load time distribution (p50, p90, p99).
*   **Feature flags**: Gate new features behind flags:
    *   New grid layout algorithm, new pin card design, experimental column counts.
    *   Gradual rollout (1% → 10% → 100%) with monitoring.
*   **Graceful degradation**:
    *   If masonry calculation fails → fall back to CSS Grid with estimated row spans.
    *   If images fail → show dominant color placeholder indefinitely.
    *   If React Query cache corrupts → clear cache and refetch.

---

## 14. Edge Cases and Tradeoffs

| Edge Case | Handling |
|-----------|---------|
| **Pin with extremely tall image (1:5 aspect ratio)** | Cap maximum card height at 3x the column width. Apply `object-fit: cover` and crop the excess. Show full image only in detail view. |
| **Pin with no image dimensions in API** | Use a default aspect ratio (3:4) as fallback. When image loads, measure real dimensions and relayout. This will cause CLS — should be rare if API is well-designed. |
| **Grid becomes empty (no pins match search)** | Show "No pins found" empty state with suggested searches or trending topics. |
| **User saves pin while offline** | Queue the save action in localStorage. Replay when online. Show "Pending" indicator on saved state. |
| **Window resize mid-scroll** | Full position recalculation. Scroll position is preserved by mapping the first visible pin to its new position. |
| **Very narrow viewport (< 320px)** | Force 1 column. Full-width pin cards. Masonry degrades gracefully to a simple list. |
| **RTL layout** | Masonry column assignment starts from the right. `left` positions are mirrored. Shortest column tie-breaking picks rightmost. |
| **Pin detail opened via direct URL (not from grid)** | SSR renders full pin detail page (not a modal). No grid context to return to. |
| **Browser back from pin detail modal** | Close modal, restore scroll position using sessionStorage + scrollToIndex. |
| **1000+ pins loaded in session** | Virtualization keeps DOM bounded. Spatial bucketing keeps scroll performance constant. |

### Key Tradeoffs

| Decision | Tradeoff |
|----------|----------|
| **JS masonry over CSS masonry** | Full control, correct reading order, virtualization support. But adds ~5-10KB of JS and requires layout calculation. CSS masonry (when supported) would be simpler. |
| **Absolute positioning over CSS Grid** | GPU-accelerated via `transform`; compatible with virtualization. But items are not in DOM flow — accessibility and SEO require extra care. |
| **Dominant color placeholder over blurhash** | Almost zero data overhead (7 bytes per pin). But less visually accurate than blurhash. Acceptable because speed trumps preview fidelity in a grid of 20+ items. |
| **Virtualization** | Bounded DOM and memory regardless of scroll depth. But adds complexity: spatial indexing, scroll position management, harder debugging. |
| **SSR first batch** | Better FCP, SEO, and social preview. But adds server load and hydration complexity. CSR-only would be simpler but slower perceived load. |
| **Optimistic saves** | Instant UI feedback. But rollback on failure can confuse users ("I thought I saved that"). Mitigated with clear error toasts. |
| **Cursor over offset pagination** | No duplicates in dynamic grids. But can't show "Page X of Y" or jump to arbitrary pages. Acceptable — users never want "page 47" in a discovery grid. |
| **250px overscan for virtualization** | Balance between early rendering (no blank flashes) and over-rendering (wasting resources). Lower than feed overscan (500px) because grid items are shorter. |

---

## 15. Summary and Future Improvements

### Key Architectural Decisions

1.  **JavaScript masonry layout with shortest-column-first algorithm** — correct left-to-right reading order; pre-calculated positions from API-provided dimensions; GPU-accelerated `transform` positioning.
2.  **Virtualized masonry grid** — bounded DOM footprint for infinite scroll; spatial bucketing for O(1) visible pin lookup.
3.  **Cursor-based infinite scroll with IntersectionObserver** — seamless pagination without duplicates; sentinel pattern avoids scroll event overhead.
4.  **Dominant color placeholders** — instant visual feedback with almost zero data overhead; prevents CLS via `aspectRatio` CSS.
5.  **SSR for initial grid** — fast FCP with server-rendered first batch; CSR for all subsequent scroll loads.
6.  **Optimistic save-to-board** — instant "Saved" feedback; rollback on failure.
7.  **React Query for server state** — automatic caching, stale-while-revalidate, infinite query support.

### Possible Future Enhancements

*   **CSS Native Masonry**: When `grid-template-rows: masonry` ships in all browsers, migrate from JS-based layout for simpler code and potentially better performance.
*   **Shared Element Transitions**: Animate pin card expanding to pin detail modal using the View Transitions API — smooth visual connection between grid and detail.
*   **Offline browsing**: Cache recent pins and boards in IndexedDB via Service Worker for offline access.
*   **Web Workers for layout**: Offload masonry position calculation and spatial index building to a Web Worker for zero main-thread cost.
*   **Visual search integration**: On pin hover, offer "Find similar" that uses image ML to search for visually similar pins.
*   **Client-side filtering**: Apply tag/category filters client-side to already-loaded pins without a server round-trip.
*   **Skeleton with aspect ratios**: Instead of generic grey boxes, show skeletons that match estimated aspect ratios from the incoming batch metadata.
*   **Predictive prefetching**: Use scroll velocity to prefetch the next 2-3 batches before the sentinel fires.

---

### Endpoint Summary

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/feed` | GET | Fetch personalized home feed pins (cursor-based) |
| `/api/search` | GET | Search pins by query |
| `/api/pin/:id` | GET | Fetch single pin with full detail |
| `/api/pin/:id/related` | GET | Fetch related/similar pins |
| `/api/pin/:id/save` | POST | Save pin to a board |
| `/api/pin/:id/unsave` | DELETE | Remove pin from a board |
| `/api/pin/:id/comments` | GET | Fetch comments for a pin |
| `/api/pin/:id/comment` | POST | Add a comment |
| `/api/boards` | GET | Fetch user's board list |
| `/api/board/:id/pins` | GET | Fetch pins in a board |
| `/api/board` | POST | Create a new board |

---

### Complete Pin Grid Flow

| Direction | Mechanism | Trigger | Endpoint | Action |
|-----------|-----------|---------|----------|--------|
| Initial Load | REST (SSR + CSR) | On mount | `GET /api/feed?limit=25` | Render first 25 pins in masonry grid |
| Infinite Scroll | REST | Sentinel intersects viewport | `GET /api/feed?cursor=<token>` | Append next batch; incremental layout calc |
| Pin Detail | REST + Lazy Load | User taps pin | `GET /api/pin/:id` | Open detail modal with full data |
| Related Pins | REST | Pin detail loads | `GET /api/pin/:id/related` | Show "More like this" mini grid |
| Save Pin | REST + Optimistic | User taps Save | `POST /api/pin/:id/save` | Instant UI update; background API call |
| Unsave Pin | REST + Optimistic | User taps Unsave | `DELETE /api/pin/:id/unsave` | Instant UI update; background API call |
| Search | REST | User submits query | `GET /api/search?q=<query>` | New masonry grid with results |
| Board View | REST | User navigates to board | `GET /api/board/:id/pins` | Masonry grid with board's pins |
| Resize | Client Only | Window resize | — | Recalculate column count + positions |

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design
