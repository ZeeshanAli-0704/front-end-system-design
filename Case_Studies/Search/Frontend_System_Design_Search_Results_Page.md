# Frontend System Design: Search Results Page

- [Frontend System Design: Search Results Page](#frontend-system-design-search-results-page)
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
    - [5.2 Search Results Rendering and Pagination Strategy (Deep Dive)](#52-search-results-rendering-and-pagination-strategy-deep-dive)
      - [5.2.1 Result Card Polymorphism and Mixed Content Types](#521-result-card-polymorphism-and-mixed-content-types)
      - [5.2.2 Query Highlighting in Results](#522-query-highlighting-in-results)
      - [5.2.3 Pagination Strategy: Offset vs Cursor vs Infinite Scroll](#523-pagination-strategy-offset-vs-cursor-vs-infinite-scroll)
      - [5.2.4 Faceted Filtering and URL State Synchronization](#524-faceted-filtering-and-url-state-synchronization)
      - [5.2.5 Sort and Layout Toggle](#525-sort-and-layout-toggle)
      - [5.2.6 Zero Results and Spell Correction](#526-zero-results-and-spell-correction)
      - [5.2.7 Skeleton Loading and Perceived Performance](#527-skeleton-loading-and-perceived-performance)
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
    - [9.2 Search Query Lifecycle and Optimization (Deep Dive)](#92-search-query-lifecycle-and-optimization-deep-dive)
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

*   A search results page (SERP) is the primary interface users see after submitting a query — whether on a web search engine (Google), an e-commerce platform (Amazon), or a content site (YouTube, GitHub).
*   It displays a ranked list of results matching the user's query, enriched with metadata, filters, pagination, and contextual enhancements like spell corrections, featured snippets, and related searches.
*   Target users: every user of a search-enabled platform — from casual consumers browsing products to power users performing research.
*   Primary use case: finding the most relevant result as quickly as possible and either consuming it in place or navigating to the detail page.

---

### 1.2 Key User Personas

*   **Casual Searcher**: Submits a query, scans the first few results, clicks the most relevant one. Rarely goes past page 1. Represents the majority of users.
*   **Power / Refining Searcher**: Uses filters, sorts, and pagination to narrow down from thousands of results. Common in e-commerce (price range, brand, rating) and job search platforms.
*   **Browser / Explorer**: Doesn't have a specific target. Browses results across pages, opens multiple links. Common in research, news, and shopping.
*   **Mobile User**: Searches on a small screen with limited bandwidth. Expects fewer but highly relevant results with compact UI and fast load times.

---

### 1.3 Core User Flows (High Level)

*   **Search and Click (Primary Flow)**:
    1.  User types a query in the search bar and submits (press Enter or tap Search).
    2.  SERP loads with the first page of results, optional featured snippet at the top, and filter/facet sidebar.
    3.  User scans results (title, URL, snippet) and clicks a result to navigate to the detail/landing page.
    4.  If the user wants to refine: uses the back button to return to SERP (scroll position preserved) and modifies filters or query.

*   **Filter and Sort Flow (Secondary Flow)**:
    1.  User views initial results and applies a filter (e.g., price range $50–$100, brand "Nike", rating ≥ 4 stars).
    2.  URL updates to reflect filter state (`?q=shoes&brand=nike&price=50-100`).
    3.  Results reload with filtered data — result count updates, filters show active selections.
    4.  User changes sort order (relevance → price low-to-high) — results re-fetch and re-render.

*   **Paginate and Explore Flow (Secondary Flow)**:
    1.  User scrolls to the bottom of page 1 results.
    2.  Clicks "Next" or page 2 — URL updates to `?q=shoes&page=2`.
    3.  New results load; user can navigate back to page 1 freely.
    4.  Alternatively (infinite scroll): user scrolls and more results load automatically.

---

## 2. Requirements

### 2.1 Functional Requirements

*   **Results Display**:
    *   Render a ranked list of search results — each result shows a title, URL/breadcrumb, text snippet, and optional thumbnail/image.
    *   Support multiple result types rendered differently: web links, product cards, image results, video results, knowledge panels, featured snippets.
    *   Highlight the matched query terms in the result title and snippet.
*   **Search Input**:
    *   Persistent search bar at the top of the SERP (pre-filled with the current query).
    *   User can modify the query and re-submit directly from the SERP.
*   **Faceted Filters (E-Commerce / Vertical Search)**:
    *   Sidebar or top-bar filters: category, brand, price range, rating, availability, seller, etc.
    *   Multi-select and range filters.
    *   Active filter chips (removable).
    *   Result count updates dynamically as filters change.
*   **Sorting**:
    *   Sort options: relevance (default), price (low-to-high, high-to-low), rating, newest, popularity.
    *   Sort selection triggers a re-fetch (server-side sort).
*   **Pagination**:
    *   Numbered page navigation (Google-style) or infinite scroll (Amazon/Pinterest-style).
    *   Page number reflected in the URL (`?page=2`) for shareability and SEO.
*   **Query Enhancements**:
    *   "Did you mean: {corrected query}?" — spell correction suggestion.
    *   "Showing results for {corrected}. Search instead for {original}?" — auto-correction.
    *   Related searches at the bottom of results.
    *   Featured snippet / answer box for informational queries.
*   **Result Actions**:
    *   Click to navigate to the result detail page.
    *   Quick preview on hover (optional — knowledge panel, product quick-view).
    *   Save / Wishlist button on e-commerce results.
    *   Share result link.
*   **Layout Options**:
    *   List view (default for web search).
    *   Grid view (default for image/product search).
    *   Toggle between list and grid on e-commerce SERPs.

---

### 2.2 Non Functional Requirements

*   **Performance**: FCP < 1.2s; TTI < 2.5s; search results visible within 500ms of query submission. Pagination should render new results within 300ms.
*   **Scalability**: Handle millions of concurrent searchers; result sets of millions of items with fast server-side ranking. Frontend handles any result count gracefully.
*   **Availability**: Graceful degradation — show cached results if the search backend is slow. Skeleton loaders during loading. Never show a blank page.
*   **Security**: XSS prevention on user query reflection in the page. Sanitize all result snippets (they contain HTML bold tags for highlights). CSRF on any state-changing actions (save/wishlist).
*   **Accessibility**: Full keyboard navigation through results. Screen reader support with semantic HTML. Focus management on pagination and filter changes. High contrast for result links and metadata.
*   **Device Support**: Desktop (primary), mobile web (responsive), tablet. Touch-friendly pagination and filter controls on mobile.
*   **SEO**: SERP pages should be crawlable for platforms where results themselves are content (e-commerce category pages, directory listings). Clean URL structure.
*   **i18n**: RTL layout support; locale-aware number and currency formatting (1,200 results / ₹1,299 / $49.99); localized filter labels.

---

## 3. Scope Clarification (Interview Scoping)

### 3.1 In Scope

*   Search results page rendering with mixed result types (web links, product cards, images).
*   Faceted filtering with URL synchronization.
*   Pagination (offset-based with numbered pages).
*   Sort controls.
*   Query term highlighting in results.
*   Spell correction and zero-result handling.
*   State management for results, filters, sort, and pagination.
*   API design from the frontend perspective.
*   Performance optimization for fast SERP rendering.
*   Responsive layout (list + grid views).

---

### 3.2 Out of Scope

*   Backend search engine internals (indexing, ranking, Elasticsearch/Solr configuration).
*   Typeahead / autocomplete component (covered in the companion article).
*   Ads and sponsored results system.
*   Natural language processing and query understanding.
*   Voice search and visual search.
*   Rich interactive widgets (calculator, weather, flight tracker in search results).

---

### 3.3 Assumptions

*   Backend search API is available and returns ranked, paginated results with pre-computed snippets.
*   User may or may not be authenticated (search works for anonymous users; auth enables personalization and wishlist).
*   Filters and facet counts are computed server-side and returned alongside results.
*   Images and thumbnails in results are served via CDN.

---

## 4. High Level Frontend Architecture

### 4.1 Overall Approach

*   **SPA** (Single Page Application) with client-side routing — navigating between pages, applying filters, and sorting does not cause full page reloads.
*   **SSR** (Server-Side Rendering) for the initial SERP load — critical for SEO (e-commerce category pages indexed by crawlers) and fast FCP (results visible without waiting for client-side JS to fetch data).
*   **CSR** for subsequent interactions — pagination, filter changes, sort changes, and re-searches are handled client-side with API calls.
*   The search results module is the **primary chunk**; filter sidebar, grid layout, and quick-view modals are **code-split** where possible.

---

### 4.2 Major Architectural Layers

```
┌───────────────────────────────────────────────────────────────────┐
│  UI Layer                                                         │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ SearchHeader                                                │  │
│  │  ┌───────────────────────┐  ┌────────────────────────────┐  │  │
│  │  │ SearchBar (pre-filled)│  │ SortDropdown + LayoutToggle│  │  │
│  │  └───────────────────────┘  └────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────┐  ┌───────────────────────────────────────┐  │
│  │ FilterSidebar    │  │ ResultsList                           │  │
│  │  ┌────────────┐  │  │  ┌─────────────────────────────────┐ │  │
│  │  │ CategoryFil│  │  │  │ SpellCorrection / FeaturedSnip  │ │  │
│  │  │ PriceRange │  │  │  │ ResultCard[] (polymorphic)      │ │  │
│  │  │ BrandFilter│  │  │  │  ┌─────────┐  ┌─────────┐      │ │  │
│  │  │ RatingFilt │  │  │  │  │WebResult│  │ProdCard │      │ │  │
│  │  │ MoreFilters│  │  │  │  │ImageRes │  │VideoRes │      │ │  │
│  │  └────────────┘  │  │  │  └─────────┘  └─────────┘      │ │  │
│  │  ┌────────────┐  │  │  └─────────────────────────────────┘ │  │
│  │  │ActiveFilter│  │  │  ┌─────────────────────────────────┐ │  │
│  │  │  Chips     │  │  │  │ PaginationBar (1 2 3 ... 10 >) │ │  │
│  │  └────────────┘  │  │  └─────────────────────────────────┘ │  │
│  └──────────────────┘  │  ┌─────────────────────────────────┐ │  │
│                        │  │ RelatedSearches                 │ │  │
│                        │  └─────────────────────────────────┘ │  │
│                        └───────────────────────────────────────┘  │
├───────────────────────────────────────────────────────────────────┤
│  State Management Layer                                           │
│  (Search Query, Results, Filters, Sort, Pagination, URL Sync,     │
│   Loading States, User Preferences)                               │
├───────────────────────────────────────────────────────────────────┤
│  API and Data Access Layer                                        │
│  (REST/GraphQL Client, Request Deduplication, AbortController,    │
│   Response Normalization, Retry Logic)                            │
├───────────────────────────────────────────────────────────────────┤
│  Shared / Utility Layer                                           │
│  (URL serialization, String highlighting, Number/Currency format, │
│   Debounce, Analytics tracker, A/B test framework)               │
└───────────────────────────────────────────────────────────────────┘
```

---

### 4.3 External Integrations

*   **Search API**: Primary backend service that accepts queries, filters, sort, and pagination parameters. Returns ranked results with snippets and facet counts.
*   **CDN**: Serves result thumbnails, product images, and static assets (JS/CSS bundles).
*   **Analytics SDK**: Track search impressions, click-through rates (CTR), filter usage, pagination depth, and result position of clicked items.
*   **A/B Testing Framework**: Test different result layouts, sort defaults, filter UIs, and ranking signals.
*   **Spell Correction Service**: Returns corrected queries (may be part of the search API response).
*   **Ads Service (out of scope)**: In production, sponsored results would be injected into the result list.

---

## 5. Component Design and Modularization

### 5.1 Component Hierarchy

```
SearchResultsPage
 ├── SearchHeader
 │    ├── Logo / BackLink
 │    ├── SearchBar (pre-filled with current query)
 │    │    ├── SearchInput (controlled)
 │    │    └── SearchButton
 │    ├── SortDropdown
 │    └── LayoutToggle (List / Grid)
 │
 ├── ActiveFilterChips (horizontal strip of active filters with × to remove)
 │
 ├── ContentArea (flex row: sidebar + results)
 │    ├── FilterSidebar (desktop) / FilterDrawer (mobile)
 │    │    ├── CategoryFilter (tree / accordion)
 │    │    ├── PriceRangeFilter (slider + inputs)
 │    │    ├── BrandFilter (checkbox list with search)
 │    │    ├── RatingFilter (star rating selector)
 │    │    ├── AvailabilityFilter (in-stock toggle)
 │    │    └── MoreFilters (expandable)
 │    │
 │    └── ResultsContainer
 │         ├── ResultsMeta ("Showing 1–10 of 1,234 results for 'shoes'")
 │         ├── SpellCorrection ("Did you mean: running shoes?")
 │         ├── FeaturedSnippet (answer box — for informational queries)
 │         ├── ResultsList (list or grid layout)
 │         │    └── ResultCard[] (polymorphic — renders differently per type)
 │         │         ├── WebResultCard (title, URL, snippet)
 │         │         ├── ProductResultCard (image, title, price, rating, seller)
 │         │         ├── ImageResultCard (thumbnail grid)
 │         │         ├── VideoResultCard (thumbnail, duration, channel)
 │         │         └── KnowledgePanel (side card with entity info)
 │         │
 │         ├── PaginationBar (1 2 3 ... 10 Next >)
 │         └── RelatedSearches ("People also search for...")
 │
 └── NoResultsView (when result count is 0)
      ├── Suggestions to broaden search
      ├── Popular / trending items
      └── Spell correction link
```

---

### 5.2 Search Results Rendering and Pagination Strategy (Deep Dive)

The search results list is the **heart of the SERP** — a ranked list of heterogeneous content items that must render fast, support diverse layouts, and communicate relevance through visual hierarchy. Getting this right matters because:
*   Users decide within **3-5 seconds** whether to click a result or refine their query.
*   The first 3 results capture **60-70%** of all clicks (position bias) — they must be visible at FCP.
*   Results are **polymorphic** — a single SERP may contain web links, product cards, images, videos, and knowledge panels, each requiring a different component.
*   Filters and pagination changes must feel **instant** — any perceived delay after a filter click triggers user frustration.

---

#### 5.2.1 Result Card Polymorphism and Mixed Content Types

##### The Problem — One SERP, Many Result Types

Unlike a social feed where every item is a "post," a search results page renders **different components** depending on the result type:

```
┌─────────────────────────────────────────────────┐
│ Featured Snippet (answer box)                   │  ← type: "featured_snippet"
│ "A running shoe is a type of footwear..."       │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ 🔗 Best Running Shoes 2026 - Runner's World     │  ← type: "web"
│ runnersworld.com › gear › running-shoes          │
│ Top picks for runners: Nike Pegasus, ASICS...    │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ 🛒 Nike Air Zoom Pegasus 41 — ₹12,999          │  ← type: "product"
│ ★★★★☆ (4.3) · 2,340 reviews · Free delivery    │
│ [Image] [Add to Cart]                           │
└─────────────────────────────────────────────────┘

┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐              │  ← type: "image_pack"
│ img1 │ │ img2 │ │ img3 │ │ img4 │              │
└──────┘ └──────┘ └──────┘ └──────┘              │

┌─────────────────────────────────────────────────┐
│ 🎥 Top 10 Running Shoes Reviewed — 12:34        │  ← type: "video"
│ YouTube · RunnerChannel · 1.2M views · 2 mo     │
│ [Thumbnail]                                     │
└─────────────────────────────────────────────────┘
```

##### Strategy — Registry-Based Polymorphic Rendering

Use a **result type → component map** to render the correct card for each result. This avoids a giant switch/if-else chain and makes adding new result types trivial.

```tsx
import { lazy } from 'react';

// Registry: result type → component
const RESULT_RENDERERS: Record<string, React.ComponentType<{ result: SearchResult }>> = {
  web: WebResultCard,
  product: ProductResultCard,
  video: VideoResultCard,
  image_pack: ImagePackCard,
  featured_snippet: FeaturedSnippetCard,
  knowledge_panel: KnowledgePanelCard,
};

// Fallback for unknown types
const DefaultResultCard = ({ result }: { result: SearchResult }) => (
  <WebResultCard result={result} />
);

function ResultsList({ results }: { results: SearchResult[] }) {
  return (
    <ol className="results-list" role="list" aria-label="Search results">
      {results.map((result, index) => {
        const Renderer = RESULT_RENDERERS[result.type] || DefaultResultCard;
        return (
          <li key={result.id} className="result-item">
            <Renderer result={result} />
          </li>
        );
      })}
    </ol>
  );
}
```

##### Why a Registry Over a Switch Statement?

| Approach | Adding a new result type | Code organization | Runtime behavior |
|----------|------------------------|-------------------|-----------------|
| **Switch/if-else** | Edit the switch — grows into a 200-line monster | All types in one file | Same |
| **Registry map** | Add one entry: `myType: MyComponent` | Each type in its own file | Same |
| **Lazy registry** | `lazy(() => import('./MyCard'))` | Per-type code splitting | Only load components for types in the current result set |

For a SERP with many result types (Google has 20+), the registry pattern keeps the code organized and extensible.

##### Each Result Card Component

**WebResultCard** — the classic blue-link result:

```tsx
function WebResultCard({ result }: { result: WebResult }) {
  return (
    <article className="web-result">
      <div className="result-breadcrumb">
        <img
          src={result.favicon}
          alt=""
          width={16}
          height={16}
          loading="lazy"
        />
        <span className="result-url">{result.displayUrl}</span>
      </div>
      <h3 className="result-title">
        <a href={result.url} target="_blank" rel="noopener noreferrer">
          <HighlightedText text={result.title} highlights={result.titleHighlights} />
        </a>
      </h3>
      <p className="result-snippet">
        <HighlightedText text={result.snippet} highlights={result.snippetHighlights} />
      </p>
      {result.sitelinks && <SiteLinks links={result.sitelinks} />}
    </article>
  );
}
```

**ProductResultCard** — e-commerce result:

```tsx
function ProductResultCard({ result }: { result: ProductResult }) {
  return (
    <article className="product-result">
      <a href={result.url} className="product-link">
        <div className="product-image">
          <img
            src={result.thumbnailUrl}
            alt={result.title}
            width={200}
            height={200}
            loading="lazy"
          />
          {result.badge && <span className="product-badge">{result.badge}</span>}
        </div>
        <div className="product-info">
          <h3 className="product-title">
            <HighlightedText text={result.title} highlights={result.titleHighlights} />
          </h3>
          <div className="product-rating">
            <StarRating value={result.rating} />
            <span className="rating-count">({result.reviewCount.toLocaleString()})</span>
          </div>
          <div className="product-price">
            {result.originalPrice && (
              <span className="original-price">
                {formatCurrency(result.originalPrice, result.currency)}
              </span>
            )}
            <span className="sale-price">
              {formatCurrency(result.price, result.currency)}
            </span>
            {result.discount && (
              <span className="discount-badge">-{result.discount}%</span>
            )}
          </div>
          <div className="product-meta">
            {result.freeDelivery && <span className="free-delivery">Free delivery</span>}
            {result.seller && <span className="seller">by {result.seller}</span>}
          </div>
        </div>
      </a>
    </article>
  );
}
```

---

#### 5.2.2 Query Highlighting in Results

##### The Concept

When the user searches for "running shoes", the matching terms in result titles and snippets should be visually distinguished:

```
Title:  Best **Running Shoes** for Marathon Training 2026
Snippet: Discover the top **running shoes** that offer comfort,
         durability, and speed for long-distance **running**.
```

This helps users quickly identify **why** a result matched their query and scan results faster.

##### Two Approaches to Highlighting

**Approach 1: Server-Side Highlights (Recommended)**

The search backend (Elasticsearch, Solr, etc.) already identifies matching terms during scoring. It returns highlight ranges or pre-marked HTML:

```json
{
  "title": "Best Running Shoes for Marathon Training",
  "titleHighlights": [
    { "start": 5, "end": 18 }
  ],
  "snippet": "Discover the top running shoes that offer...",
  "snippetHighlights": [
    { "start": 16, "end": 29 },
    { "start": 87, "end": 94 }
  ]
}
```

**Why server-side is better:**
*   The server knows the **actual tokens** used for matching (after stemming, synonyms, etc.). "Running" might also match "run" and "runner" — the server knows this, the client doesn't.
*   Highlights are consistent with the ranking — the server highlights the same terms it scored.
*   No client-side computation cost.

**Approach 2: Client-Side Highlighting (Fallback)**

If the API doesn't return highlight ranges, the frontend can perform basic highlighting by splitting on query terms:

```tsx
function highlightQueryTerms(text: string, query: string): React.ReactNode[] {
  if (!query.trim()) return [text];

  // Split query into individual terms
  const terms = query
    .trim()
    .split(/\s+/)
    .filter(Boolean)
    .map((t) => t.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')); // escape regex chars

  if (terms.length === 0) return [text];

  const regex = new RegExp(`(${terms.join('|')})`, 'gi');
  const parts = text.split(regex);

  return parts.map((part, i) =>
    regex.test(part) ? <mark key={i}>{part}</mark> : part
  );
}
```

##### Implementation — Rendering Server-Side Highlights Safely

```tsx
type HighlightRange = { start: number; end: number };

function HighlightedText({
  text,
  highlights,
}: {
  text: string;
  highlights?: HighlightRange[];
}) {
  if (!highlights || highlights.length === 0) {
    return <span>{text}</span>;
  }

  // Sort highlights by start position
  const sorted = [...highlights].sort((a, b) => a.start - b.start);
  const parts: React.ReactNode[] = [];
  let lastIndex = 0;

  sorted.forEach((highlight, i) => {
    // Non-highlighted text before this highlight
    if (highlight.start > lastIndex) {
      parts.push(text.slice(lastIndex, highlight.start));
    }
    // Highlighted text
    parts.push(
      <mark key={i} className="query-highlight">
        {text.slice(highlight.start, highlight.end)}
      </mark>
    );
    lastIndex = highlight.end;
  });

  // Remaining text after the last highlight
  if (lastIndex < text.length) {
    parts.push(text.slice(lastIndex));
  }

  return <span>{parts}</span>;
}
```

**Why use `<mark>` instead of `<strong>` or `<b>`?**
*   `<mark>` is **semantically correct** — it indicates text that is "highlighted" or marked for reference. Screen readers announce it as "highlighted."
*   `<strong>` implies importance, not search relevance.
*   CSS can style `<mark>` specifically: `mark.query-highlight { background: #fff3cd; padding: 0 2px; }`.

---

#### 5.2.3 Pagination Strategy: Offset vs Cursor vs Infinite Scroll

##### Why This Matters

Pagination is one of the **most debated design decisions** in SERP design. The choice affects URL structure, SEO, user behavior, and backend complexity.

##### Three Approaches

| Pagination Type | How it works | URL example | SEO | UX pattern |
|----------------|-------------|-------------|-----|-----------|
| **Offset (numbered pages)** | `page=2` → server returns results 11-20 | `?q=shoes&page=2` | Excellent (each page is a crawlable URL) | Google, Bing, traditional e-commerce |
| **Cursor-based** | `cursor=abc123` → server returns next batch | `?q=shoes&cursor=abc123` | Bad (opaque cursor is meaningless to crawlers) | Dynamic feeds, timelines |
| **Infinite scroll** | User scrolls → next batch loads automatically | `?q=shoes` (no page in URL) | Bad (all results on one "page" for crawlers) | Pinterest, some mobile e-commerce |

##### Why Offset Is Best for SERPs

```
User searches "shoes" on Google:

  Page 1: ?q=shoes           → Results 1-10
  Page 2: ?q=shoes&page=2    → Results 11-20
  Page 3: ?q=shoes&page=3    → Results 21-30

Benefits:
  ✅ User can bookmark "page 3 of shoes results"
  ✅ User can share the URL — recipient sees the same page
  ✅ Browser back/forward works perfectly (each page is a history entry)
  ✅ Google bot can crawl page 2, 3, 4... (SEO for e-commerce category pages)
  ✅ User sees "Page 2 of 124" — progress indicator
  ✅ No scroll position complexity (page starts at top)

Infinite scroll problems for SERPs:
  ❌ No progress indicator ("how far am I?")
  ❌ Can't share "I was looking at result #47"
  ❌ Back button loses scroll position (complex to restore)
  ❌ SEO — crawlers can't trigger scroll events
  ❌ Filter changes must reset an unbounded scroll
```

**Exception:** Infinite scroll works well on **mobile** for product browsing (Amazon mobile uses "load more" + infinite scroll hybrid). Desktop SERPs almost universally use numbered pagination.

##### Implementation — Numbered Pagination Component

```tsx
function PaginationBar({
  currentPage,
  totalPages,
  onPageChange,
}: {
  currentPage: number;
  totalPages: number;
  onPageChange: (page: number) => void;
}) {
  const pages = generatePageNumbers(currentPage, totalPages);

  return (
    <nav aria-label="Search results pagination">
      <ul className="pagination">
        {/* Previous button */}
        <li>
          <button
            onClick={() => onPageChange(currentPage - 1)}
            disabled={currentPage === 1}
            aria-label="Previous page"
          >
            ← Prev
          </button>
        </li>

        {/* Page numbers */}
        {pages.map((page, index) =>
          page === '...' ? (
            <li key={`ellipsis-${index}`} className="pagination-ellipsis">
              <span aria-hidden="true">...</span>
            </li>
          ) : (
            <li key={page}>
              <button
                onClick={() => onPageChange(page as number)}
                aria-current={page === currentPage ? 'page' : undefined}
                className={page === currentPage ? 'active' : ''}
              >
                {page}
              </button>
            </li>
          )
        )}

        {/* Next button */}
        <li>
          <button
            onClick={() => onPageChange(currentPage + 1)}
            disabled={currentPage === totalPages}
            aria-label="Next page"
          >
            Next →
          </button>
        </li>
      </ul>
    </nav>
  );
}

// Generate page numbers with ellipsis
// [1, 2, 3, '...', 8, 9, 10] when on page 2 of 10
function generatePageNumbers(
  current: number,
  total: number
): (number | '...')[] {
  if (total <= 7) {
    return Array.from({ length: total }, (_, i) => i + 1);
  }

  const pages: (number | '...')[] = [];

  // Always show first page
  pages.push(1);

  if (current > 3) {
    pages.push('...');
  }

  // Pages around current
  const start = Math.max(2, current - 1);
  const end = Math.min(total - 1, current + 1);
  for (let i = start; i <= end; i++) {
    pages.push(i);
  }

  if (current < total - 2) {
    pages.push('...');
  }

  // Always show last page
  pages.push(total);

  return pages;
}
```

##### Why Scroll to Top on Page Change

When the user clicks "Next" or a page number, the results should:
1.  Scroll to the top of the results container.
2.  Focus the first result (for keyboard/screen reader users).

```tsx
function handlePageChange(page: number) {
  setCurrentPage(page);
  // Scroll to top of results
  window.scrollTo({ top: 0, behavior: 'instant' });
  // Move focus to first result (accessibility)
  document.querySelector('.result-item:first-child a')?.focus();
}
```

**Why `behavior: 'instant'` instead of `'smooth'`?** On page navigation, users expect immediate repositioning. Smooth scrolling during page change feels laggy and confusing.

---

#### 5.2.4 Faceted Filtering and URL State Synchronization

##### The Core Challenge — URL Is the Source of Truth

On a SERP, the URL must encode the **entire search state**: query, filters, sort, and page. This enables:
*   **Shareability**: Copy the URL → recipient sees the exact same results.
*   **Back/Forward navigation**: Each filter change creates a history entry.
*   **SEO**: `/search?q=shoes&brand=nike&page=2` is a crawlable URL.
*   **Persistence**: Refresh the page → same results (state is in the URL, not in memory).

```
URL encodes full state:
  /search?q=shoes&brand=nike,adidas&price=50-200&rating=4&sort=price_asc&page=2

Parsed state:
  {
    query: "shoes",
    filters: {
      brand: ["nike", "adidas"],
      price: { min: 50, max: 200 },
      rating: 4
    },
    sort: "price_asc",
    page: 2
  }
```

##### Implementation — Bidirectional URL ↔ State Sync

```tsx
import { useSearchParams, useNavigate } from 'react-router-dom';

// Parse URL search params into structured state
function parseSearchParams(params: URLSearchParams): SearchState {
  return {
    query: params.get('q') || '',
    filters: {
      brand: params.get('brand')?.split(',').filter(Boolean) || [],
      price: {
        min: params.has('price_min') ? Number(params.get('price_min')) : undefined,
        max: params.has('price_max') ? Number(params.get('price_max')) : undefined,
      },
      rating: params.has('rating') ? Number(params.get('rating')) : undefined,
      category: params.get('category') || undefined,
    },
    sort: (params.get('sort') as SortOption) || 'relevance',
    page: Number(params.get('page')) || 1,
  };
}

// Serialize state back to URL search params
function serializeSearchState(state: SearchState): URLSearchParams {
  const params = new URLSearchParams();

  params.set('q', state.query);

  if (state.filters.brand.length > 0) {
    params.set('brand', state.filters.brand.join(','));
  }
  if (state.filters.price.min !== undefined) {
    params.set('price_min', String(state.filters.price.min));
  }
  if (state.filters.price.max !== undefined) {
    params.set('price_max', String(state.filters.price.max));
  }
  if (state.filters.rating !== undefined) {
    params.set('rating', String(state.filters.rating));
  }
  if (state.filters.category) {
    params.set('category', state.filters.category);
  }
  if (state.sort !== 'relevance') {
    params.set('sort', state.sort);
  }
  if (state.page > 1) {
    params.set('page', String(state.page));
  }

  return params;
}

// Hook that syncs URL ↔ search state
function useSearchState() {
  const [searchParams, setSearchParams] = useSearchParams();
  const state = useMemo(() => parseSearchParams(searchParams), [searchParams]);

  const updateState = useCallback(
    (updates: Partial<SearchState>) => {
      const newState = { ...state, ...updates };

      // Reset page to 1 when filters or sort change
      if (updates.filters || updates.sort || updates.query) {
        newState.page = 1;
      }

      setSearchParams(serializeSearchState(newState), { replace: false });
    },
    [state, setSearchParams]
  );

  return { state, updateState };
}
```

##### Why Reset Page on Filter Change?

When the user applies a new filter:
*   The result set changes entirely (different count, different ranking).
*   Being on "page 5" of the old result set is meaningless for the new result set.
*   The user expects to see the **best matches** for their refined query — that's page 1.

```
User is on page 3 of "shoes" results.
User applies filter: brand = Nike.
  → Page resets to 1.
  → Results count drops from 12,000 to 800.
  → Page 3 of the old results had nothing to do with Nike-filtered results.
```

##### Filter Change Flow — Replace vs Push History

| Interaction | History behavior | Why |
|-------------|-----------------|-----|
| Apply a filter | **Push** new history entry | User can press Back to undo the filter |
| Change page | **Push** new history entry | User can go Back to previous page |
| Remove a filter | **Push** | Undoing a filter removal via Back feels natural |
| Change sort | **Push** | User might want to go back to previous sort |
| Typing in price range (each keystroke) | **Replace** | Don't create 10 history entries while typing "$1", "$12", "$120" |
| Debounced price range (final value) | **Push** once | One history entry per completed price input |

##### Facet Counts and Dynamic Filters

The search API should return **facet counts** alongside results — these tell the user how many results exist for each filter value:

```json
{
  "results": [...],
  "facets": {
    "brand": [
      { "value": "Nike", "count": 342, "selected": true },
      { "value": "Adidas", "count": 218, "selected": false },
      { "value": "Puma", "count": 95, "selected": false }
    ],
    "priceRange": {
      "min": 29,
      "max": 499,
      "histogram": [
        { "range": "0-50", "count": 120 },
        { "range": "50-100", "count": 340 },
        { "range": "100-200", "count": 280 }
      ]
    },
    "rating": [
      { "value": 4, "label": "4★ & up", "count": 580 },
      { "value": 3, "label": "3★ & up", "count": 790 }
    ]
  }
}
```

Displaying counts helps users make informed filter decisions: "Nike (342)" vs "Puma (95)" — the user knows Nike has more results before clicking.

**Important UX detail:** Disable (grey out) filter options with count 0. Don't hide them — hiding causes the filter list to jump/reflow, which is disorienting.

---

#### 5.2.5 Sort and Layout Toggle

##### Sort Dropdown

```tsx
type SortOption = 'relevance' | 'price_asc' | 'price_desc' | 'rating' | 'newest' | 'popularity';

const SORT_OPTIONS: { value: SortOption; label: string }[] = [
  { value: 'relevance', label: 'Most Relevant' },
  { value: 'price_asc', label: 'Price: Low to High' },
  { value: 'price_desc', label: 'Price: High to Low' },
  { value: 'rating', label: 'Customer Rating' },
  { value: 'newest', label: 'Newest First' },
  { value: 'popularity', label: 'Most Popular' },
];

function SortDropdown({
  currentSort,
  onSortChange,
}: {
  currentSort: SortOption;
  onSortChange: (sort: SortOption) => void;
}) {
  return (
    <label className="sort-control">
      <span className="sort-label">Sort by:</span>
      <select
        value={currentSort}
        onChange={(e) => onSortChange(e.target.value as SortOption)}
        aria-label="Sort results"
      >
        {SORT_OPTIONS.map((option) => (
          <option key={option.value} value={option.value}>
            {option.label}
          </option>
        ))}
      </select>
    </label>
  );
}
```

##### Layout Toggle (List vs Grid)

For e-commerce SERPs, provide a list/grid toggle:

```tsx
type LayoutMode = 'list' | 'grid';

function LayoutToggle({
  layout,
  onLayoutChange,
}: {
  layout: LayoutMode;
  onLayoutChange: (layout: LayoutMode) => void;
}) {
  return (
    <div className="layout-toggle" role="radiogroup" aria-label="Results layout">
      <button
        role="radio"
        aria-checked={layout === 'list'}
        onClick={() => onLayoutChange('list')}
        aria-label="List view"
        className={layout === 'list' ? 'active' : ''}
      >
        ☰
      </button>
      <button
        role="radio"
        aria-checked={layout === 'grid'}
        onClick={() => onLayoutChange('grid')}
        aria-label="Grid view"
        className={layout === 'grid' ? 'active' : ''}
      >
        ⊞
      </button>
    </div>
  );
}
```

**Layout persistence:** Save layout preference in localStorage. Don't encode it in the URL — layout is a user preference, not a search parameter.

```tsx
function useLayoutPreference(): [LayoutMode, (layout: LayoutMode) => void] {
  const [layout, setLayout] = useState<LayoutMode>(() => {
    return (localStorage.getItem('search_layout') as LayoutMode) || 'list';
  });

  const updateLayout = useCallback((newLayout: LayoutMode) => {
    setLayout(newLayout);
    localStorage.setItem('search_layout', newLayout);
  }, []);

  return [layout, updateLayout];
}
```

---

#### 5.2.6 Zero Results and Spell Correction

##### Zero Results Page

When a query returns no results, the page should **not** be empty. Provide actionable alternatives:

```tsx
function NoResultsView({
  query,
  spellSuggestion,
  popularItems,
}: {
  query: string;
  spellSuggestion?: string;
  popularItems?: SearchResult[];
}) {
  return (
    <div className="no-results" role="alert">
      <h2>No results found for "{query}"</h2>

      <div className="no-results-suggestions">
        <h3>Suggestions:</h3>
        <ul>
          <li>Check your spelling</li>
          <li>Try broader keywords (e.g., "shoes" instead of "running shoes size 10 blue")</li>
          <li>Remove some filters</li>
        </ul>
      </div>

      {spellSuggestion && (
        <div className="spell-suggestion">
          <p>
            Did you mean:{' '}
            <a href={`/search?q=${encodeURIComponent(spellSuggestion)}`}>
              <strong>{spellSuggestion}</strong>
            </a>
            ?
          </p>
        </div>
      )}

      {popularItems && popularItems.length > 0 && (
        <div className="popular-items">
          <h3>Popular right now</h3>
          <ResultsList results={popularItems} />
        </div>
      )}
    </div>
  );
}
```

##### Spell Correction — Two Patterns

**Pattern 1: Suggestion-only** (Google-style):
```
Search results for: "runnig shoes"

  Did you mean: running shoes?     ← clickable link
  
  [Results for "runnig shoes" shown below...]
```

**Pattern 2: Auto-correct with escape** (Google/Amazon-style):
```
Showing results for: running shoes     ← auto-corrected
Search instead for: runnig shoes        ← link to original query

  [Results for "running shoes" shown below]
```

```tsx
function SpellCorrection({
  originalQuery,
  correctedQuery,
  autoCorrect,
}: {
  originalQuery: string;
  correctedQuery: string;
  autoCorrect: boolean;
}) {
  if (autoCorrect) {
    return (
      <div className="spell-correction" role="status">
        <p>
          Showing results for{' '}
          <strong>{correctedQuery}</strong>
        </p>
        <p>
          Search instead for{' '}
          <a href={`/search?q=${encodeURIComponent(originalQuery)}&nospell=1`}>
            {originalQuery}
          </a>
        </p>
      </div>
    );
  }

  return (
    <div className="spell-correction" role="status">
      <p>
        Did you mean:{' '}
        <a href={`/search?q=${encodeURIComponent(correctedQuery)}`}>
          <strong>{correctedQuery}</strong>
        </a>
        ?
      </p>
    </div>
  );
}
```

---

#### 5.2.7 Skeleton Loading and Perceived Performance

##### The Problem — Empty Page During Fetch

When the user submits a search or changes page/filters, there's a 200-800ms window where data is loading. Showing a blank page or a generic spinner is jarring.

##### Strategy — Content-Aware Skeleton

Show skeleton cards that match the **shape** of real results. This gives users a sense of what's coming and reduces perceived wait time.

```tsx
function ResultsListSkeleton({ count = 10 }: { count?: number }) {
  return (
    <div className="results-skeleton" aria-busy="true" aria-label="Loading results">
      {Array.from({ length: count }, (_, i) => (
        <div key={i} className="skeleton-card">
          <div className="skeleton-line skeleton-url" style={{ width: '30%' }} />
          <div className="skeleton-line skeleton-title" style={{ width: '70%' }} />
          <div className="skeleton-line skeleton-snippet" style={{ width: '90%' }} />
          <div className="skeleton-line skeleton-snippet" style={{ width: '60%' }} />
        </div>
      ))}
    </div>
  );
}
```

```css
.skeleton-line {
  height: 14px;
  background: linear-gradient(90deg, #e0e0e0 25%, #f0f0f0 50%, #e0e0e0 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s ease-in-out infinite;
  border-radius: 4px;
  margin-bottom: 8px;
}

.skeleton-title {
  height: 20px;
  margin-bottom: 12px;
}

@keyframes shimmer {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}
```

##### When to Show Skeleton vs Stale Results

| Scenario | Strategy |
|----------|----------|
| **Initial search (no previous results)** | Show skeleton immediately |
| **Page change** | Show skeleton — new page content is completely different |
| **Filter change** | Option A: Show skeleton (clean transition) Option B: Keep old results with opacity + spinner overlay (stale-while-revalidate feel) |
| **Re-search (new query)** | Show skeleton — old results are irrelevant |

**The stale-while-revalidate pattern for filters** (keeps the page from feeling blank):

```tsx
function ResultsContainer({ results, isLoading }: Props) {
  return (
    <div className={`results-container ${isLoading ? 'results-loading' : ''}`}>
      {results.map((result) => (
        <ResultCard key={result.id} result={result} />
      ))}
    </div>
  );
}

// CSS
.results-loading {
  opacity: 0.5;
  pointer-events: none;
  position: relative;
}
.results-loading::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 32px;
  height: 32px;
  border: 3px solid #ccc;
  border-top-color: #333;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}
```

---

#### 5.2.8 Decision Matrix

| Scenario | Strategy | Implementation | Notes |
|----------|----------|---------------|-------|
| **Multiple result types** | Registry-based polymorphic renderer | Type → component map | Extensible; code-split per type |
| **Query highlighting** | Server-side highlight ranges | `HighlightedText` component with `<mark>` | Semantically correct; accessible |
| **Pagination** | Offset-based numbered pages | `PaginationBar` with ellipsis | SEO-friendly; URL encodes page |
| **URL ↔ state sync** | Bidirectional serialization | `useSearchState` hook with `useSearchParams` | URL is single source of truth |
| **Filter change** | Push history entry + reset page to 1 | `setSearchParams(newState, { replace: false })` | Back button undoes last filter |
| **Sort** | Server-side sort with re-fetch | Sort param in URL and API | Never sort large result sets client-side |
| **Layout preference** | localStorage (not in URL) | `useLayoutPreference` hook | User preference, not search state |
| **Zero results** | Helpful alternatives + spell correction | `NoResultsView` + `SpellCorrection` | Never show an empty page |
| **Loading state** | Skeleton for new search; fade for filter change | `ResultsListSkeleton` or opacity overlay | Match skeleton to result card shape |

---

### 5.3 Reusability Strategy

*   **`ResultCard` (polymorphic)**: Registry maps result type → component. New types are added by registering one entry.
*   **`HighlightedText`**: Shared highlighting component, reusable for any text with match ranges (search results, autocomplete, log viewers).
*   **`PaginationBar`**: Generic pagination with ellipsis generation — reusable for any paginated list (results, orders, comments).
*   **`FilterSidebar`**: Composed of generic filter primitives: `CheckboxFilter`, `RangeFilter`, `RatingFilter`. Each is reusable independently.
*   **`useSearchState`**: URL ↔ state synchronization hook. Reusable for any page with URL-driven state.
*   **`SortDropdown` / `LayoutToggle`**: Generic controls, configurable via props.
*   Design system tokens for result card typography, link colors, highlight colors, and spacing.

---

### 5.4 Module Organization

```
src/
 ├── features/
 │    └── search/
 │         ├── components/
 │         │    ├── SearchResultsPage.tsx
 │         │    ├── SearchHeader.tsx
 │         │    ├── ResultsList.tsx
 │         │    ├── resultCards/
 │         │    │    ├── WebResultCard.tsx
 │         │    │    ├── ProductResultCard.tsx
 │         │    │    ├── ImagePackCard.tsx
 │         │    │    ├── VideoResultCard.tsx
 │         │    │    ├── FeaturedSnippetCard.tsx
 │         │    │    ├── KnowledgePanelCard.tsx
 │         │    │    └── resultCardRegistry.ts
 │         │    ├── FilterSidebar.tsx
 │         │    ├── filters/
 │         │    │    ├── CheckboxFilter.tsx
 │         │    │    ├── RangeFilter.tsx
 │         │    │    ├── RatingFilter.tsx
 │         │    │    └── ActiveFilterChips.tsx
 │         │    ├── PaginationBar.tsx
 │         │    ├── SortDropdown.tsx
 │         │    ├── LayoutToggle.tsx
 │         │    ├── SpellCorrection.tsx
 │         │    ├── NoResultsView.tsx
 │         │    ├── RelatedSearches.tsx
 │         │    ├── ResultsMeta.tsx
 │         │    └── ResultsSkeleton.tsx
 │         ├── hooks/
 │         │    ├── useSearchState.ts
 │         │    ├── useSearchResults.ts
 │         │    ├── useLayoutPreference.ts
 │         │    └── useFilterSidebar.ts
 │         ├── api/
 │         │    └── searchApi.ts
 │         ├── utils/
 │         │    ├── urlSerializer.ts
 │         │    ├── highlightText.ts
 │         │    ├── paginationHelper.ts
 │         │    └── currencyFormatter.ts
 │         └── types/
 │              └── search.types.ts
 ├── shared/
 │    ├── components/ (HighlightedText, Skeleton, StarRating, Dropdown)
 │    ├── hooks/ (useDebounce, useMediaQuery, useLocalStorage)
 │    └── utils/ (formatNumber, formatCurrency, sanitizeHTML)
```

---

## 6. High Level Data Flow Explanation

### 6.1 Initial Load Flow

```
1. User submits search query (from homepage or re-search on SERP)
     ↓
2. Router navigates to /search?q=shoes
     ↓
3. SSR: Server fetches search results for "shoes" (page 1, default sort, no filters)
     ↓
4. Server renders the SERP shell: SearchHeader, FilterSidebar, ResultsList, PaginationBar
     ↓
5. HTML streams to browser → FCP: user sees results immediately
     ↓
6. Client hydrates → event listeners attach (filter clicks, pagination, sort)
     ↓
7. If URL has filters/sort params, they are already applied in the server fetch
     ↓
8. Analytics: fire "search_impression" event with query, result count, and first page result IDs
```

---

### 6.2 User Interaction Flow

**Filter Application Flow:**

```
User clicks "Nike" in brand filter
     ↓
1. URL updates: /search?q=shoes&brand=nike (page resets to 1)
     ↓
2. useSearchState hook detects URL change → derives new state
     ↓
3. useSearchResults hook fires new API call:
   GET /api/search?q=shoes&brand=nike&page=1&sort=relevance
     ↓
4. Previous results shown with loading overlay (opacity 0.5 + spinner)
     ↓
5. API responds → results update, facet counts update, total count updates
     ↓
6. ResultsList re-renders; FilterSidebar updates facet counts
     ↓
7. Scroll to top of results
```

**Pagination Flow:**

```
User clicks "Page 3" in pagination bar
     ↓
1. URL updates: /search?q=shoes&page=3
     ↓
2. API call: GET /api/search?q=shoes&page=3
     ↓
3. Skeleton replaces old results (new page = completely different content)
     ↓
4. API responds → new results render
     ↓
5. Scroll to top; focus first result
     ↓
6. Analytics: fire "search_page_view" event with page number
```

**Sort Change Flow:**

```
User selects "Price: Low to High" from sort dropdown
     ↓
1. URL updates: /search?q=shoes&sort=price_asc (page resets to 1)
     ↓
2. API call with new sort parameter
     ↓
3. Results re-render in new order
```

---

### 6.3 Error and Retry Flow

*   **Search API fails (500/503)**: Show error state with "Search is temporarily unavailable. Please try again." + Retry button. Keep the search bar functional.
*   **Search times out (> 5s)**: Show "Search is taking longer than expected" with a "Try Again" button. Don't auto-retry — the user might want to modify their query.
*   **Filter API fails**: Keep current results visible. Show toast: "Failed to update filters. Try again."
*   **Network offline**: Show "You're offline" banner. If previous results are cached (React Query/SWR), show them with a stale indicator. Search bar still usable — queue the search for when connectivity returns.
*   **Rate limited (429)**: Back off; show "Please wait a moment before searching again." Implement exponential backoff for retries.
*   **Partial failure (some result enrichments fail)**: Render results without enrichment (e.g., show product without image if image CDN is down). Use fallback UI for failed thumbnails.

---

## 7. Data Modelling (Frontend Perspective)

### 7.1 Core Data Entities

*   **SearchResult** — a single result item (polymorphic: web, product, video, image, snippet).
*   **SearchResponse** — the API response containing results, facets, pagination info, and query metadata.
*   **Facet** — a filter dimension with possible values and counts.
*   **SpellCorrection** — a corrected query suggestion.
*   **RelatedSearch** — a related query suggestion.
*   **FeaturedSnippet** — an answer box with extracted content.

---

### 7.2 Data Shape

```ts
// Base result — all result types share these fields
type BaseResult = {
  id: string;
  type: 'web' | 'product' | 'video' | 'image_pack' | 'featured_snippet' | 'knowledge_panel';
  url: string;
  title: string;
  titleHighlights?: HighlightRange[];
};

type HighlightRange = { start: number; end: number };

// Web result (classic blue link)
type WebResult = BaseResult & {
  type: 'web';
  displayUrl: string;
  favicon?: string;
  snippet: string;
  snippetHighlights?: HighlightRange[];
  sitelinks?: { title: string; url: string }[];
  publishedDate?: string;
};

// Product result (e-commerce)
type ProductResult = BaseResult & {
  type: 'product';
  thumbnailUrl: string;
  price: number;
  originalPrice?: number;
  currency: string;
  discount?: number;
  rating: number;
  reviewCount: number;
  seller?: string;
  freeDelivery?: boolean;
  inStock: boolean;
  badge?: string;              // "Best Seller", "New Arrival", etc.
};

// Video result
type VideoResult = BaseResult & {
  type: 'video';
  thumbnailUrl: string;
  duration: string;            // "12:34"
  channel: string;
  platform: string;            // "YouTube", "Vimeo"
  views: number;
  publishedDate: string;
};

// Image pack (horizontal set of images)
type ImagePackResult = BaseResult & {
  type: 'image_pack';
  images: {
    thumbnailUrl: string;
    fullUrl: string;
    alt: string;
    width: number;
    height: number;
  }[];
};

// Featured snippet (answer box)
type FeaturedSnippetResult = BaseResult & {
  type: 'featured_snippet';
  answer: string;
  sourceUrl: string;
  sourceName: string;
};

// Union type
type SearchResult = WebResult | ProductResult | VideoResult | ImagePackResult | FeaturedSnippetResult;

// Facet types
type FacetValue = {
  value: string;
  label: string;
  count: number;
  selected: boolean;
};

type RangeFacet = {
  min: number;
  max: number;
  selectedMin?: number;
  selectedMax?: number;
  histogram?: { range: string; count: number }[];
};

type Facets = {
  category?: FacetValue[];
  brand?: FacetValue[];
  price?: RangeFacet;
  rating?: FacetValue[];
  availability?: FacetValue[];
  [key: string]: FacetValue[] | RangeFacet | undefined;
};

// Full API response
type SearchResponse = {
  query: string;
  correctedQuery?: string;
  autoCorrect: boolean;
  results: SearchResult[];
  totalCount: number;
  page: number;
  pageSize: number;
  totalPages: number;
  facets: Facets;
  relatedSearches: string[];
  featuredSnippet?: FeaturedSnippetResult;
};
```

---

### 7.3 Entity Relationships

*   **One-to-Many**: One `SearchResponse` → many `SearchResult` items; One `SearchResponse` → many `Facet` dimensions.
*   **One-to-One**: One `SearchResponse` → one optional `FeaturedSnippet`; One `SearchResponse` → one optional `SpellCorrection`.
*   **Many-to-Many**: `Facet` values relate to `SearchResult` items (a result can match multiple facet values; a facet value applies to multiple results). But this relationship is computed server-side — the frontend only sees counts.

**Normalized vs Denormalized:**

| Approach | For SERPs |
|----------|-----------|
| **Denormalized** (each result is self-contained) | ✅ Correct choice. Results are read-only and don't share nested entities across items. |
| **Normalized** | ❌ Unnecessary. Results don't reference shared sub-entities (unlike a social feed where posts share author data). |

---

### 7.4 UI Specific Data Models

```ts
// Full page state
type SearchPageState = {
  query: string;
  filters: FilterState;
  sort: SortOption;
  page: number;
  layout: LayoutMode;
};

type FilterState = {
  brand: string[];
  category?: string;
  price: { min?: number; max?: number };
  rating?: number;
  [key: string]: unknown;
};

type SortOption = 'relevance' | 'price_asc' | 'price_desc' | 'rating' | 'newest' | 'popularity';
type LayoutMode = 'list' | 'grid';

// Derived / view state
type SearchPageViewModel = {
  resultsDisplay: string;          // "Showing 1–10 of 1,234 results"
  activeFilterCount: number;       // for "Filters (3)" badge on mobile
  hasActiveFilters: boolean;
  isFirstPage: boolean;
  isLastPage: boolean;
  canClearFilters: boolean;
};
```

---

## 8. State Management Strategy

### 8.1 State Classification

| State Type | Examples | Storage |
|---|---|---|
| **URL State (source of truth)** | Query, filters, sort, page | URL search params via `useSearchParams` |
| **Server State** | Search results, facets, total count, spell correction | React Query / SWR cache (keyed by URL params) |
| **User Preference State** | Layout mode (list/grid), recent searches | localStorage |
| **Component Local State** | Filter sidebar open/closed (mobile), price range input intermediate value | `useState` |
| **Derived State** | Results display string, active filter count, pagination range | Computed via `useMemo` |

---

### 8.2 State Ownership

*   **SearchResultsPage** is the top-level container. It reads state from the URL via `useSearchState` and passes it down.
*   **URL is the single source of truth** for all search-related state. This is critical. No component stores search state locally that isn't reflected in the URL.
*   **FilterSidebar** receives current filter state from the URL and fires `updateState({ filters: {...} })` on change — which updates the URL.
*   **ResultsList** receives results from React Query (keyed by URL params). It is a pure rendering component.
*   **PaginationBar** receives `currentPage` and `totalPages` from the search response and fires `updateState({ page: n })`.

**Why URL as source of truth?**
*   Every possible SERP state is a **URL** — shareable, bookmarkable, crawlable.
*   Browser navigation (back/forward) automatically works because each state change is a URL change.
*   Page refresh restores the exact state (no session/memory loss).
*   React Query uses the URL-derived params as cache keys. Different filter combinations = different cache entries.

---

### 8.3 Persistence Strategy

| Data | Persistence | Reason |
|------|------------|--------|
| Search query, filters, sort, page | URL search params | Shareable, bookmarkable, SEO-friendly |
| Search results | React Query in-memory cache | `staleTime: 2min`; refetch on URL change |
| Layout preference (list/grid) | localStorage | User preference; persists across sessions |
| Recent searches | localStorage | Displayed in typeahead; user-managed |
| Filter sidebar state (mobile) | Component state | UI-only; not persisted |

---

## 9. High Level API Design (Frontend POV)

### 9.1 Required APIs

| API | Method | Description |
|-----|--------|-------------|
| `/api/search` | **GET** | Fetch paginated search results with filters, sort, and facets |
| `/api/search/suggest` | **GET** | Fetch related search suggestions (for "People also search for") |
| `/api/search/trending` | **GET** | Fetch trending/popular searches (for empty state) |

---

### 9.2 Search Query Lifecycle and Optimization (Deep Dive)

#### The Complete Request Timeline

```
User types "running shoes" in search bar → presses Enter

Time  0ms                           ~200ms                              ~700ms
      │                              │                                   │
  [Enter pressed]              [SSR fetch starts]                  [HTML streams to browser]
      │                              │                                   │
      └──→ Router navigates to       └──→ Backend receives               └──→ FCP: results visible
           /search?q=running+shoes        query + params;                     Client hydrates
                                          returns results + facets            Interactions active

Total time to visible results (SSR): ~700ms ✅
Total time to visible results (CSR only): ~1200ms ❌ (JS parse + fetch + render)
```

#### Request Deduplication

React Query automatically deduplicates requests with the same key. If the user rapidly clicks between page 2 and page 3 and back to page 2, only one request fires for each unique page (the second request for page 2 uses the cached response).

```tsx
function useSearchResults(state: SearchState) {
  return useQuery({
    queryKey: ['search', state.query, state.filters, state.sort, state.page],
    queryFn: () => fetchSearchResults(state),
    staleTime: 2 * 60 * 1000,        // 2 minutes
    keepPreviousData: true,            // show old results while fetching new
    refetchOnWindowFocus: false,       // search results don't change while tab is hidden
  });
}
```

**`keepPreviousData: true`** is important: while a new search is loading, the UI shows the **previous** results (faded) instead of a blank skeleton. This feels faster.

#### Prefetching Adjacent Pages

When the user is on page 2, prefetch page 3 in the background so navigation feels instant:

```tsx
function usePrefetchNextPage(state: SearchState) {
  const queryClient = useQueryClient();

  useEffect(() => {
    if (state.page < totalPages) {
      const nextPageState = { ...state, page: state.page + 1 };
      queryClient.prefetchQuery({
        queryKey: ['search', nextPageState.query, nextPageState.filters, nextPageState.sort, nextPageState.page],
        queryFn: () => fetchSearchResults(nextPageState),
        staleTime: 2 * 60 * 1000,
      });
    }
  }, [state.page]);
}
```

---

### 9.3 Request and Response Structure

**GET /api/search**

```json
// Request
// GET /api/search?q=running+shoes&brand=nike&price_min=50&price_max=200&sort=relevance&page=1&pageSize=10

// Response
{
  "query": "running shoes",
  "correctedQuery": null,
  "autoCorrect": false,
  "results": [
    {
      "id": "r_001",
      "type": "featured_snippet",
      "url": "https://runnersworld.com/best-running-shoes",
      "title": "Best Running Shoes 2026",
      "answer": "The best running shoes for 2026 combine responsive cushioning with lightweight design...",
      "sourceUrl": "https://runnersworld.com/best-running-shoes",
      "sourceName": "Runner's World"
    },
    {
      "id": "r_002",
      "type": "product",
      "url": "https://www.amazon.com/dp/B0ABC123",
      "title": "Nike Air Zoom Pegasus 41 Men's Running Shoes",
      "titleHighlights": [{ "start": 0, "end": 4 }, { "start": 20, "end": 34 }],
      "thumbnailUrl": "https://cdn.example.com/products/pegasus41.jpg",
      "price": 12999,
      "originalPrice": 14999,
      "currency": "INR",
      "discount": 13,
      "rating": 4.3,
      "reviewCount": 2340,
      "seller": "Nike Official",
      "freeDelivery": true,
      "inStock": true,
      "badge": "Best Seller"
    },
    {
      "id": "r_003",
      "type": "web",
      "url": "https://runnersworld.com/best-running-shoes",
      "displayUrl": "runnersworld.com › gear › running-shoes",
      "favicon": "https://runnersworld.com/favicon.ico",
      "title": "15 Best Running Shoes for Every Type of Runner",
      "titleHighlights": [{ "start": 8, "end": 21 }],
      "snippet": "Our experts tested 50+ running shoes. Top picks include the Nike Pegasus for daily runs...",
      "snippetHighlights": [{ "start": 25, "end": 38 }],
      "publishedDate": "2026-02-15"
    },
    {
      "id": "r_004",
      "type": "video",
      "url": "https://youtube.com/watch?v=abc123",
      "title": "Top 10 Running Shoes Reviewed and Compared",
      "titleHighlights": [{ "start": 7, "end": 20 }],
      "thumbnailUrl": "https://i.ytimg.com/vi/abc123/hqdefault.jpg",
      "duration": "12:34",
      "channel": "RunnerReviews",
      "platform": "YouTube",
      "views": 1200000,
      "publishedDate": "2026-01-10"
    }
  ],
  "totalCount": 1234,
  "page": 1,
  "pageSize": 10,
  "totalPages": 124,
  "facets": {
    "brand": [
      { "value": "nike", "label": "Nike", "count": 342, "selected": true },
      { "value": "adidas", "label": "Adidas", "count": 218, "selected": false },
      { "value": "asics", "label": "ASICS", "count": 167, "selected": false },
      { "value": "brooks", "label": "Brooks", "count": 134, "selected": false }
    ],
    "price": {
      "min": 29,
      "max": 499,
      "selectedMin": 50,
      "selectedMax": 200,
      "histogram": [
        { "range": "0-50", "count": 120 },
        { "range": "50-100", "count": 340 },
        { "range": "100-200", "count": 280 },
        { "range": "200-500", "count": 94 }
      ]
    },
    "rating": [
      { "value": "4", "label": "4★ & up", "count": 580, "selected": false },
      { "value": "3", "label": "3★ & up", "count": 790, "selected": false }
    ]
  },
  "relatedSearches": [
    "running shoes for flat feet",
    "best running shoes for beginners",
    "nike running shoes men",
    "running shoes on sale"
  ]
}
```

---

### 9.4 Error Handling and Status Codes

| Status | Scenario | Frontend Handling |
|--------|----------|-------------------|
| `200` | Success | Render results, facets, pagination |
| `200` + empty results | Query matched nothing | Show `NoResultsView` with suggestions |
| `304` | Not Modified (HTTP cache) | Use cached response |
| `400` | Invalid query/params (malformed page, bad filter key) | Show error toast; reset to page 1 with no filters |
| `401` | Unauthorized (personalized search requires auth) | Fall back to anonymous search results |
| `429` | Rate limited | Show "Please wait" message; exponential backoff |
| `500` | Server error | Show error state with retry button; keep search bar functional |
| `503` | Service unavailable (indexing, maintenance) | Show "Search is temporarily unavailable"; retry with backoff |
| `504` | Gateway timeout (slow query) | Show "Search took too long — try a simpler query" + Retry |

---

## 10. Caching Strategy

### 10.1 What to Cache

*   **Search results per query+filters+sort+page**: Each unique combination is a separate cache entry. Short TTL — results can change as inventory updates, new content is indexed, etc.
*   **Facet metadata**: Cached with results (same response). Facet counts change with every filter application.
*   **Product thumbnails and images**: Aggressively cached via CDN (images don't change for a given product version).
*   **Static assets**: JS/CSS bundles with immutable caching.
*   **Related searches**: Cached per query with medium TTL (these change slowly).

### 10.2 Where to Cache

| Data | Cache Location | TTL |
|------|----------------|-----|
| Search results (API data) | React Query in-memory cache | 2 min (staleTime); keyed by full URL params |
| Product images / thumbnails | Browser HTTP cache + CDN | 24h+ (content-addressable or versioned URLs) |
| Favicons | Browser HTTP cache | 7 days |
| JS/CSS bundles | Browser HTTP cache + CDN | 1 year (immutable, content-hashed) |
| Layout preference | localStorage | Indefinite (user-managed) |
| Recent searches | localStorage | 30 days, user-managed |
| Adjacent page (prefetched) | React Query in-memory cache | 2 min |

### 10.3 Cache Invalidation

*   **Search results**: Automatically stale after 2 min. New search (different URL) = different cache key. Page refresh triggers revalidation.
*   **Facets**: Invalidated with each new search request (facet counts depend on the current query + other active filters).
*   **Product images**: Content-addressable URLs (hash in filename). New image = new URL = automatic invalidation.
*   **Static assets**: Cache-busted via webpack content hashing (`bundle.[contenthash].js`).
*   **Prefetched pages**: Same TTL as regular results. If the user takes > 2 min to click "Next," the prefetched data is revalidated.

---

## 11. CDN and Asset Optimization

*   **Product images**: Served via CDN with responsive sizes (`srcset: 100w, 200w, 400w`). WebP/AVIF with JPEG fallback. Aspect ratio hints from API (`width`, `height`) to prevent CLS.
*   **Thumbnails**: Small (100-200px), heavily cached. Lazy loaded for results below the fold.
*   **Favicons**: Served from the CDN (not fetched from individual sites). Proxied through a favicon service: `https://cdn.example.com/favicons/runnersworld.com`.
*   **Cache headers**:
    *   Static assets: `Cache-Control: public, max-age=31536000, immutable`.
    *   API responses: `Cache-Control: private, max-age=120` (2-min browser cache for identical queries).
    *   Product images: `Cache-Control: public, max-age=86400` (24h).
*   **Compression**: Brotli for all text responses (HTML, JSON, JS, CSS). API responses compress from ~5-20KB to ~1-4KB.
*   **Preconnect**: `<link rel="preconnect" href="https://cdn.example.com">` for CDN image domain. `<link rel="dns-prefetch" href="https://images.productcdn.com">` for third-party image hosts.
*   **Image optimization**:
    *   First 3 product images: `loading="eager"` (above the fold — critical for LCP).
    *   Rest: `loading="lazy"`.
    *   Use `<picture>` for format negotiation or let the CDN handle it via `Accept` header.

---

## 12. Rendering Strategy

*   **Initial SERP load**: **SSR** for the full first page of results. This is critical for:
    *   **SEO**: E-commerce category pages (`/search?q=shoes`) need to be crawlable with product data in the HTML.
    *   **FCP**: Results are visible immediately without waiting for client JS to fetch data.
    *   **Social previews**: When a search URL is shared, crawlers get rendered HTML with OG metadata.
*   **Subsequent interactions (filter, sort, paginate)**: **CSR** — client-side fetch and re-render. The page shell (header, sidebar structure, footer) is already rendered; only the results area updates.
*   **Filter sidebar**: SSR'd with the initial facet data. Interactive behavior (expand/collapse, range inputs) hydrates on client.
*   **Code splitting**:
    *   Core SERP (result list, pagination, filters) = main chunk.
    *   Uncommon result types (KnowledgePanelCard, FeaturedSnippetCard) = lazy-loaded chunks (only loaded if the API returns those result types).
    *   Filter drawer (mobile) = lazy-loaded (loaded on hamburger menu tap).
*   **Hydration**: Selective hydration (React 18+) — hydrate the search bar and filter controls first (interactive), then the result links (less urgent).
*   **Streaming SSR**: Use `renderToPipeableStream` to stream the SERP HTML. Send the header + search bar + first 3 results immediately, then stream remaining results as they complete.

---

## 13. Cross Cutting Non Functional Concerns

### 13.1 Security

*   **XSS mitigation — query reflection**:
    *   The current query is displayed in: (a) the search input, (b) "Showing results for 'X'", (c) spell correction. All must be **escaped**.
    *   React JSX auto-escapes: `{query}` in JSX is safe.
    *   **Danger zone**: If the API returns snippet HTML with `<b>` tags for highlighting (e.g., Elasticsearch `highlight` feature), we must sanitize with DOMPurify before using `dangerouslySetInnerHTML`. Better: use offset-based highlights (see section 5.2.2) and avoid raw HTML entirely.
    *   URL parameter reflection: Ensure `?q=<script>alert(1)</script>` is harmless — React handles this automatically, but server-rendered HTML must also escape.
*   **CSRF**: Search is read-only (GET requests) — CSRF is not a concern for search. State-changing actions (save to wishlist, add to cart from SERP) require CSRF tokens.
*   **Content Security Policy**: `img-src` restricted to CDN origins. `script-src self` prevents injection of external scripts.
*   **URL injection**: Validate and sanitize URL parameters before constructing API requests. Never pass raw URL params directly to `fetch` without encoding.
*   **Rate limiting**: Client-side debounce on re-search (if user hits Enter repeatedly). Backend enforces per-IP rate limits.

---

### 13.2 Accessibility

*   **Semantic HTML**:
    *   Results list: `<ol role="list" aria-label="Search results">` — ordered list because results are ranked.
    *   Each result: `<li>` containing an `<article>` with heading (`<h3>`) for the title.
    *   Filter sidebar: `<aside aria-label="Filters">` with `<fieldset>` and `<legend>` for each filter group.
    *   Pagination: `<nav aria-label="Search results pagination">` with `aria-current="page"` on the active page button.
*   **Keyboard navigation**:
    *   `Tab` moves through result links, filter controls, and pagination buttons.
    *   Skip links: "Skip to results" and "Skip to filters" at the top.
    *   Filter checkboxes are natively keyboard-accessible (Space to toggle).
    *   Price range slider has `aria-valuemin`, `aria-valuemax`, `aria-valuenow`.
*   **Screen reader support**:
    *   Results count announced: `<p aria-live="polite">Showing 1–10 of 1,234 results for "shoes"</p>`.
    *   When results update (filter/page change): `aria-busy="true"` on the results container during loading; `aria-live="polite"` region announces "Results updated. Showing 1-10 of 800 results."
    *   Spell correction: `role="status"` so screen readers announce the correction.
    *   Star rating: `aria-label="Rated 4.3 out of 5 stars"` (not just visual stars).
    *   Product price: `aria-label="Price: 12,999 rupees"` with currency spoken correctly.
*   **Focus management**:
    *   After page change: focus moves to the first result.
    *   After filter change: focus stays on the filter control (user might apply more filters).
    *   After clearing all filters: focus returns to the results area.
*   **High contrast**:
    *   Result links are blue/purple (visited) — meets WCAG AA contrast (4.5:1).
    *   Query highlights use `<mark>` with sufficient contrast (not just light yellow).
    *   Filter checkboxes have visible focus indicators.
*   **Reduced motion**: Skeleton shimmer animation respects `prefers-reduced-motion: reduce`.

---

### 13.3 Performance Optimization

#### Critical Rendering Path for SERPs

```
Priority order (what to render first):

1. Search bar (user might want to re-search immediately)
2. First 3 results (above the fold — captures 60-70% of clicks)
3. Pagination controls (user needs to know there are more pages)
4. Filter sidebar (for refinement)
5. Results 4-10 (below fold)
6. Related searches (bottom of page)
7. Knowledge panel / featured snippet (nice-to-have enhancements)
```

#### Specific Optimizations

*   **SSR first page**: Server-render the first page of results for fast FCP. All subsequent pages are CSR.
*   **Prefetch next page**: When user is on page N, prefetch page N+1 in background. Navigation to next page feels instant.
*   **`keepPreviousData`**: React Query shows old results (faded) while fetching new results on filter/sort change. No blank screen.
*   **Image lazy loading**:
    *   First 3 product images: `loading="eager"` (above fold, critical for LCP).
    *   Rest: `loading="lazy"`.
*   **Code split uncommon result types**: Knowledge panels, image packs, and video result cards are less common. Load their components only when needed.
*   **Memoization**:
    *   `React.memo` on each `ResultCard` — prevent re-renders when sibling results don't change.
    *   `useMemo` for derived values: formatted prices, relative dates, pagination ranges.
*   **AbortController on re-search**: When user changes query/filters/sort, abort the previous in-flight search request.
*   **Font subsetting**: If using a custom font for the SERP, subset it to Latin characters + numbers (search results are text-heavy; every ms of font load matters).
*   **`content-visibility: auto`** on results below the fold — browser skips rendering off-screen result cards.
*   **Debounce price range filter**: As user drags the price slider, don't fire API on every pixel change. Debounce → fire one API call when user stops dragging.

#### Performance Budget

| Metric | Target | Strategy |
|--------|--------|----------|
| FCP | < 1.2s | SSR first page |
| LCP | < 2.0s | Eager load first 3 product images; SSR result titles |
| TTI | < 2.5s | Code split; minimal JS for initial hydration |
| CLS | < 0.05 | Image dimensions from API; skeleton matches result card shape |
| Page change time | < 300ms | Prefetch next page; `keepPreviousData` |
| Filter change time | < 500ms | `keepPreviousData`; server-side < 200ms |

---

### 13.4 Observability and Reliability

*   **Search Analytics (CTR and Engagement)**:
    *   **Impression tracking**: Which results were visible (entered viewport). Critical for ranking quality feedback.
    *   **Click tracking**: Which result was clicked, at which position, with which query. The most important signal for search quality.
    *   **CTR by position**: Click-through rate at positions 1-10. If position 1 CTR drops, ranking may have degraded.
    *   **Refinement rate**: How often users modify the query after seeing results. High refinement = poor initial results.
    *   **Pagination depth**: How far users paginate. If many users go to page 5+, results quality may need improvement.
    *   **Filter usage**: Which filters are used most, which filter combinations lead to conversions.
    *   **Zero result rate**: Percentage of queries returning no results. Spike = backend indexing issue or new content gap.
*   **Performance monitoring**:
    *   Track FCP, LCP, CLS for the SERP page.
    *   Track search API latency (p50, p90, p99) from the client perspective.
    *   Track time from Enter keypress to first result visible.
    *   Track filter change → results update latency.
*   **Error Boundaries**:
    *   Wrap each `ResultCard` in an error boundary — if one result's rendering crashes (malformed data), the rest remain functional.
    *   Wrap `FilterSidebar` in an error boundary — if filters crash, results are still usable.
*   **Feature flags**:
    *   New result card layouts, different pagination styles, filter UI experiments.
    *   A/B test sort defaults (relevance vs personalized), result count per page (10 vs 20), and featured snippet placement.
*   **Graceful degradation**:
    *   If facets API fails → show results without filter sidebar.
    *   If one result type component fails to load → use `DefaultResultCard` fallback.
    *   If spell correction service is down → show results for the raw query.
    *   If CDN for product images is down → show placeholder images with alt text.

---

## 14. Edge Cases and Tradeoffs

| Edge Case | Handling |
|-----------|---------|
| **Query is empty** | Redirect to homepage or show trending/popular items. Don't fetch with empty query. |
| **Query contains only special characters** | URL-encode with `encodeURIComponent`. Let the backend decide whether to return results or zero-result state. |
| **Extremely long query (500+ chars)** | Truncate display in the search bar but send full query to API. Backend may truncate for performance. |
| **Page number exceeds total pages** | `?page=999` when only 10 pages exist → redirect to last valid page or page 1. |
| **Filter combination yields 0 results** | Show zero-result state with suggestion to remove filters. Show facet counts of 0 for impossible combinations. |
| **User clicks result, presses Back** | Browser restores page from bfcache or React Query cache. Scroll position restored. |
| **Very slow search (> 3s)** | Show "This is taking longer than expected" with option to cancel. Don't auto-timeout; let the user decide. |
| **Product goes out of stock between search and click** | Not a frontend concern (product detail page handles this). But if real-time inventory is available, gray out "Out of stock" items in results. |
| **Price in different currencies** | API returns currency code. Frontend formats with `Intl.NumberFormat`. Never assume a single currency. |
| **RTL language** | Filter sidebar moves to the right. Results text aligns right. Pagination direction reverses. Use `dir="auto"` on result text. |
| **Screen is very narrow (< 320px)** | Collapse filter sidebar into a full-screen drawer. Stack result cards vertically. Reduce pagination to "Prev/Next" only. |
| **User has JavaScript disabled** | SSR ensures first page of results is visible. Pagination links are `<a>` tags with real URLs (progressive enhancement). Filters require JS — show a message or use `<form>` with server-side handling. |
| **Concurrent filter and page change** | If user clicks page 3 then immediately applies a filter: abort the page 3 request. The filter change resets to page 1 — new request supersedes. |

### Key Tradeoffs

| Decision | Tradeoff |
|----------|----------|
| **SSR for initial SERP** | Better FCP and SEO, but adds server complexity and TTFB. CSR-only is simpler but slower and not crawlable. |
| **Offset pagination over cursor** | Bookmarkable, SEO-friendly, "Page 3 of 100" progress. But slight risk of duplicate results on highly dynamic data (acceptable for search). |
| **URL as single source of truth** | Shareable, back/forward works, SEO-friendly. But URL serialization/deserialization adds boilerplate. Every state change requires URL update. |
| **Server-side sort over client-side** | Correct for large result sets (can't sort 1M results on client). But adds a round-trip for sort changes. Acceptable because network latency (~200ms) is faster than sorting large datasets in the browser. |
| **keepPreviousData on filter change** | Old results visible during loading (no blank screen). But briefly shows stale data — user must visually recognize the loading state (opacity fade). |
| **Skeleton vs spinner** | Skeleton matches result shape (lower perceived latency). But requires per-result-type skeleton components — more code. |
| **Registry pattern for result types** | Clean extensibility — add a type in one place. But indirection — harder to trace which component renders for a given type without reading the registry. |
| **Facet counts from server** | Accurate counts that reflect the current filter state. But requires the backend to compute counts on every request — adds latency and server load. |
| **Prefetching next page** | Instant page navigation. But wastes bandwidth if user never clicks "Next" (prefetched but unused). Acceptable because search result payloads are small (~10-20KB). |

---

## 15. Summary and Future Improvements

### Key Architectural Decisions

1.  **SSR for initial SERP + CSR for interactions** — fast FCP with SEO-friendly crawlable pages; client-side filter, sort, and pagination for instant feel.
2.  **URL as single source of truth** — query, filters, sort, and page are all encoded in the URL. Shareable, bookmarkable, and back/forward navigation works perfectly.
3.  **Offset-based numbered pagination** — SEO-friendly; progress indicator; each page is a crawlable URL. Cursor-based avoided due to opaque URLs.
4.  **Registry-based polymorphic result cards** — type → component map for extensible, code-split result rendering.
5.  **Server-side highlights** — search backend provides match ranges; frontend renders `<mark>` elements. Semantically correct and accessible.
6.  **React Query with `keepPreviousData`** — show old results (faded) while fetching new results on filter/sort changes. Prefetch adjacent pages for instant navigation.
7.  **Bidirectional URL ↔ state synchronization** — `useSearchState` hook ensures URL and component state are always in sync with proper history management.

### Possible Future Enhancements

*   **Instant search (search-as-you-type)**: Update results as the user types in the search bar (debounced). Blurs the line between typeahead and SERP.
*   **Saved searches / alerts**: Allow users to save a search with filters and receive notifications when new results match.
*   **Compare feature**: Select multiple product results and compare them side-by-side.
*   **Visual search**: "Search by image" — upload a photo and find visually similar products.
*   **Personalized ranking**: Use user's search history, purchase history, and browsing behavior to re-rank results client-side or via personalized API.
*   **Result previews on hover**: Show a popover card with more details (product specs, article summary) without navigating away.
*   **Voice search**: Microphone integration in the search bar; transcribed query feeds into the search flow.
*   **Offline search**: Cache recent search results in IndexedDB; serve cached results when offline with a "Results may be outdated" disclaimer.
*   **Web Workers for highlight computation**: Offload client-side highlighting (fallback mode) to a Web Worker for large result sets.
*   **View Transitions API**: Smooth transition animation between search results page and product detail page.

---

### Endpoint Summary

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/search` | GET | Fetch paginated results with filters, sort, facets, and spell correction |
| `/api/search/suggest` | GET | Fetch related search suggestions |
| `/api/search/trending` | GET | Fetch trending searches (for empty/homepage state) |

---

### Complete Search Flow

| Direction | Mechanism | Trigger | Endpoint | Action |
|-----------|-----------|---------|----------|--------|
| Initial Search | SSR + REST | Enter keypress / form submit | `GET /api/search?q=X` | SSR renders first page; client hydrates |
| Filter Change | CSR + REST | Filter checkbox / slider | `GET /api/search?q=X&brand=Y` | URL updates; fetch new results; page resets to 1 |
| Sort Change | CSR + REST | Sort dropdown selection | `GET /api/search?q=X&sort=Z` | URL updates; fetch re-sorted results; page resets to 1 |
| Page Change | CSR + REST | Pagination button click | `GET /api/search?q=X&page=N` | URL updates; fetch page N; scroll to top |
| Re-Search | CSR + REST | New query in search bar | `GET /api/search?q=newQuery` | URL updates; fresh search; filters reset |
| Prefetch Next Page | Background REST | On current page render | `GET /api/search?q=X&page=N+1` | Silent background fetch; cached for instant nav |
| Back Navigation | Browser Cache | Browser back button | — | React Query cache hit; results from memory |

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design
