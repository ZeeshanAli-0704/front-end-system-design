# Frontend System Design: Social Feed

- [Frontend System Design: Social Feed](#frontend-system-design-social-feed)
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
    - [5.2 Feed List Rendering and Infinite Scroll Strategy (Deep Dive)](#52-feed-list-rendering-and-infinite-scroll-strategy-deep-dive)
      - [5.2.1 Infinite Scroll with IntersectionObserver](#521-infinite-scroll-with-intersectionobserver)
      - [5.2.2 Virtualized Feed List for Variable Height Items](#522-virtualized-feed-list-for-variable-height-items)
      - [5.2.3 Scroll Position Restoration](#523-scroll-position-restoration)
      - [5.2.4 New Posts Banner Pattern](#524-new-posts-banner-pattern)
      - [5.2.5 Pull to Refresh (Mobile Web)](#525-pull-to-refresh-mobile-web)
      - [5.2.6 Lazy Loading Media in Feed Posts](#526-lazy-loading-media-in-feed-posts)
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
    - [9.2 Cursor Based Pagination Strategy (Deep Dive)](#92-cursor-based-pagination-strategy-deep-dive)
    - [9.3 Real Time Feed Updates (Deep Dive)](#93-real-time-feed-updates-deep-dive)
    - [9.4 Request and Response Structure](#94-request-and-response-structure)
    - [9.5 Error Handling and Status Codes](#95-error-handling-and-status-codes)
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

*   A social feed is the central content consumption interface of any social platform — Facebook, Twitter/X, LinkedIn, Reddit, etc.
*   It displays a personalized, algorithmically ranked (or chronological) stream of posts from people, pages, and communities the user follows.
*   Target users: all platform users — casual consumers, content creators, businesses, and advertisers.
*   Primary use case: scrolling through an infinite stream of mixed-content posts (text, images, videos, links) and interacting with them via likes, comments, and shares.

---

### 1.2 Key User Personas

*   **Content Consumer**: Scrolls the feed, reads posts, reacts (likes), and occasionally comments or shares. This is the majority of users.
*   **Content Creator**: Creates posts (text, images, videos), engages with comments on their posts, and monitors engagement metrics.
*   **Business / Brand**: Publishes promoted content, tracks analytics, and uses the feed for organic reach.

---

### 1.3 Core User Flows (High Level)

*   **Browsing the Feed (Primary Flow)**:
    1.  User opens the app → feed loads with the first batch of ranked posts.
    2.  User scrolls down → older posts load automatically (infinite scroll).
    3.  New posts arrive in real-time → a "New posts available" banner appears at the top.
    4.  User taps the banner → new posts are prepended to the feed.
    5.  User interacts with a post → like count updates instantly (optimistic update).

*   **Creating a Post (Secondary Flow)**:
    1.  User taps the "Create Post" area at the top of the feed.
    2.  Composes text, attaches media (image/video), adds tags or mentions.
    3.  Taps "Post" → the post appears immediately at the top of their own feed (optimistic).
    4.  Server processes and distributes to followers' feeds.

*   **Interacting with a Post (Secondary Flow)**:
    1.  User taps Like → count increments instantly; API call fires in background.
    2.  User taps Comment → comment input expands; user types and submits.
    3.  User taps Share → share modal opens with options (share to feed, copy link, send in message).

---

## 2. Requirements

### 2.1 Functional Requirements

*   **Feed Display**:
    *   Display a personalized stream of posts (text, image, video, link previews).
    *   Support multiple content types rendered differently (text-only, single image, image carousel, video, shared link with OG preview).
    *   Show author info (avatar, name, timestamp), post content, and engagement counts (likes, comments, shares).
*   **Post Creation**:
    *   Compose text posts with optional media attachments (images, videos).
    *   Support mentions (@user) and hashtags.
*   **Social Interactions**:
    *   Like / Unlike a post (with real-time count update).
    *   Comment on a post (with threaded replies).
    *   Share / Repost a post.
*   **Infinite Scroll**:
    *   Fetch older posts as the user scrolls down (cursor-based pagination).
    *   No explicit "Load More" button — seamless infinite feed.
*   **Real Time Updates**:
    *   New posts from followed users appear as a banner ("5 new posts") without full page refresh.
    *   Live engagement count updates on visible posts (optional, depends on scale).
*   **Feed Freshness Indicators**:
    *   "New posts available" banner at the top when SSE/WebSocket delivers new content.
    *   Pull-to-refresh on mobile web to manually refetch latest feed.

---

### 2.2 Non Functional Requirements

*   **Performance**: FCP < 1.5s; TTI < 3s; smooth 60fps scrolling even with 100+ posts loaded; infinite scroll triggers should feel instantaneous (next batch preloaded).
*   **Scalability**: Support feeds with thousands of posts; users following thousands of accounts; mixed media content at varying resolutions.
*   **Availability**: Graceful degradation — show cached feed if network is down; skeleton placeholders during loading.
*   **Security**: XSS prevention on user-generated content; CSRF tokens on all POST requests; authenticated API calls; media served via signed CDN URLs.
*   **Accessibility**: Full keyboard navigation through posts; screen reader support with semantic HTML; focus management on infinite scroll; reduced motion support.
*   **Device Support**: Mobile web (primary), desktop web (responsive), low-end devices with limited memory.
*   **i18n**: RTL layout support; localized timestamps ("2 hours ago"); locale-aware number formatting (1.2K likes).

---

## 3. Scope Clarification (Interview Scoping)

### 3.1 In Scope

*   Feed rendering with infinite scroll (cursor-based pagination).
*   Post card component design (text, image, video, link previews).
*   Social interactions (like, comment, share) with optimistic updates.
*   Real-time new post notifications via SSE.
*   State management for feed data.
*   Performance optimization for media-heavy feeds.
*   API design from the frontend perspective.

---

### 3.2 Out of Scope

*   Backend ranking algorithm (feed personalization).
*   Rich post editor (mentions autocomplete, media editing, filters).
*   Ads and sponsored posts system.
*   Push notifications.
*   Direct messaging.
*   Admin moderation tools.

---

### 3.3 Assumptions

*   User is authenticated; auth token is available via HTTP-only cookie or Authorization header.
*   APIs return posts pre-ranked by the backend (frontend does not sort or rank).
*   Media (images/videos) are served from a CDN with pre-signed or public URLs.
*   The feed is the main page of the application (not embedded in a secondary view).

---

## 4. High Level Frontend Architecture

### 4.1 Overall Approach

*   **SPA** (Single Page Application) with client-side routing.
*   **SSR** for the initial feed page shell — server renders the first batch of posts for fast FCP and SEO (link previews, shared post URLs).
*   **CSR** for all subsequent interactions — infinite scroll loads, likes, comments, real-time updates are all handled client-side.
*   Feed module is the **primary chunk**; secondary features (post creation modal, share modal) are **code-split** and lazy loaded.

---

### 4.2 Major Architectural Layers

```
┌──────────────────────────────────────────────────────────┐
│  UI Layer                                                │
│  ┌──────────────┐  ┌──────────────────────────────────┐  │
│  │ CreatePostBox│  │ Feed List (Virtual / Windowed)   │  │
│  └──────────────┘  │  ┌────────────┐ ┌────────────┐   │  │
│                    │  │ PostCard   │ │ PostCard   │   │  │
│  ┌──────────────┐  │  │ ┌────────┐ │ │            │   │  │
│  │ NewPosts     │  │  │ │Actions │ │ │            │   │  │
│  │ Banner       │  │  │ │Comments│ │ │            │   │  │
│  └──────────────┘  │  │ └────────┘ │ └────────────┘   │  │
│                    │  └────────────┘                  │  │
│                    │       ┌────────────────┐         │  │
│                    │       │ FeedSentinel   │         │  │
│                    │       │(scroll trigger)│         │  │
│                    │       └────────────────┘         │  │
│                    └──────────────────────────────────┘  │
├──────────────────────────────────────────────────────────┤
│  State Management Layer                                  │
│  (Feed Store, Interaction State, New Posts Buffer,       │
│   User Session, UI Preferences)                          │
├──────────────────────────────────────────────────────────┤
│  API and Data Access Layer                               │
│  (REST Client, SSE Manager, Optimistic Update Queue,     │
│   Request Deduplication, Retry Logic)                    │
├──────────────────────────────────────────────────────────┤
│  Shared / Utility Layer                                  │
│  (IntersectionObserver helpers, Debounce/Throttle,       │
│   Date formatting, Media loader, Analytics tracker)      │
└──────────────────────────────────────────────────────────┘
```

---

### 4.3 External Integrations

*   **CDN**: Serves post images, videos, and user avatars (Cloudfront / Akamai / Fastly).
*   **Analytics SDK**: Track impressions (post viewed), engagement events (like, comment, share), scroll depth, and session duration.
*   **Backend Services**: Feed API (ranked posts), interaction APIs (like/comment/share), user profile service, SSE/WebSocket endpoint for real-time.
*   **Media Player**: Native `<video>` with HLS/DASH for adaptive video streaming in feed.
*   **Link Preview Service**: Backend-generated OG metadata for shared links (title, description, thumbnail).

---

## 5. Component Design and Modularization

### 5.1 Component Hierarchy

```
FeedPage
 ├── CreatePostBox
 │    ├── UserAvatar
 │    ├── TextInput (expands to full editor modal)
 │    └── MediaAttachButtons
 │
 ├── NewPostsBanner ("5 new posts — tap to see")
 │
 ├── FeedList (scrollable container)
 │    └── PostCard[] (repeated for each post)
 │         ├── PostHeader
 │         │    ├── AuthorAvatar
 │         │    ├── AuthorName + Timestamp
 │         │    └── MoreMenu (hide, report, save)
 │         ├── PostContent (text body with "See More" truncation)
 │         ├── PostMedia
 │         │    ├── SingleImage
 │         │    ├── ImageCarousel
 │         │    ├── VideoPlayer
 │         │    └── LinkPreviewCard
 │         ├── EngagementSummary ("12K likes · 340 comments")
 │         ├── PostActions
 │         │    ├── LikeButton
 │         │    ├── CommentButton
 │         │    └── ShareButton
 │         ├── CommentList
 │         │    └── CommentItem[]
 │         │         ├── CommentAvatar
 │         │         ├── CommentBody
 │         │         └── CommentActions (like, reply)
 │         └── CreateComment
 │              ├── UserAvatar
 │              └── CommentInput + PostButton
 │
 └── FeedSentinel (invisible element at bottom — triggers next page load)
```

---

### 5.2 Feed List Rendering and Infinite Scroll Strategy (Deep Dive)

The feed list is the **core of the entire application** — a vertically scrolling, infinitely loading stream of variable-height post cards. Getting this right matters because:
*   The feed is the **primary surface** — users spend 80%+ of their time here.
*   Posts have **variable heights** (text-only posts are short, image carousels are tall, video posts vary).
*   The list can grow to **hundreds of items** during a single session as the user scrolls.
*   Every scroll frame must render at **60fps** — any jank is immediately noticeable.
*   The feed must handle **real-time prepends** (new posts) without disrupting scroll position.

---

#### 5.2.1 Infinite Scroll with IntersectionObserver

##### The Core Pattern — Sentinel Element

Instead of listening to `scroll` events (which fire hundreds of times per second and can cause jank), we use **IntersectionObserver** to watch a single invisible "sentinel" element at the bottom of the feed. When the sentinel enters the viewport, we fetch the next page.

**Why IntersectionObserver instead of scroll events?**

| Approach | How it works | Performance | Complexity |
|----------|-------------|-------------|------------|
| **Scroll event listener** | Listen to `scroll` on window/container, calculate `scrollTop + clientHeight >= scrollHeight - threshold` | Bad — fires on every pixel scrolled; requires throttling/debouncing; forces layout recalculation (`scrollHeight`) on each fire | Moderate — throttle logic, edge cases |
| **IntersectionObserver** | Browser natively observes when a target element enters/exits a root boundary | Excellent — runs off-main-thread in the browser's compositor; fires only when intersection state changes | Simple — observe a sentinel, react to `isIntersecting` |

**The sentinel is a zero-height `<div>` positioned at the bottom of the feed list.** When the user scrolls close enough that the sentinel enters the viewport (or a margin around it), the observer fires and triggers the next page fetch.

```
┌──────────────────────────────┐
│  PostCard 1                  │  ← visible
│  PostCard 2                  │  ← visible
│  PostCard 3                  │  ← visible (bottom of viewport)
├──────────────────────────────┤
│  PostCard 4                  │  ← below viewport
│  PostCard 5                  │  ← below viewport
│  ┌────────────────────────┐  │
│  │ SENTINEL (0px height)  │  │  ← when this enters rootMargin, fetch fires
│  └────────────────────────┘  │
└──────────────────────────────┘
         ↑ rootMargin: "500px" means the observer fires
           when sentinel is 500px below the visible area
```

##### Implementation — Step by Step

```tsx
import { useEffect, useRef, useCallback } from 'react';

function FeedList() {
  const sentinelRef = useRef<HTMLDivElement>(null);
  const [posts, setPosts] = useState<Post[]>([]);
  const [cursor, setCursor] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);

  // Fetch next page of posts
  const loadMore = useCallback(async () => {
    if (isLoading || !hasMore) return;
    setIsLoading(true);

    try {
      const response = await fetch(`/api/feed?cursor=${cursor || ''}&limit=10`);
      const data = await response.json();

      setPosts((prev) => [...prev, ...data.posts]);
      setCursor(data.nextCursor);
      setHasMore(data.hasMore);
    } catch (error) {
      // Retry logic handled in error flow section
    } finally {
      setIsLoading(false);
    }
  }, [cursor, isLoading, hasMore]);

  // Set up IntersectionObserver on the sentinel
  useEffect(() => {
    const sentinel = sentinelRef.current;
    if (!sentinel) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        // When sentinel is visible (or within rootMargin), load more
        if (entry.isIntersecting) {
          loadMore();
        }
      },
      {
        root: null,             // observe relative to the viewport
        rootMargin: '0px 0px 500px 0px',  // fire 500px BEFORE sentinel is visible
        threshold: 0,           // fire as soon as even 1px enters the margin
      }
    );

    observer.observe(sentinel);

    return () => observer.disconnect();
  }, [loadMore]);

  return (
    <div className="feed-list">
      {posts.map((post) => (
        <PostCard key={post.id} post={post} />
      ))}

      {/* Sentinel — invisible trigger element */}
      {hasMore && (
        <div
          ref={sentinelRef}
          className="feed-sentinel"
          style={{ height: 0, overflow: 'hidden' }}
          aria-hidden="true"
        />
      )}

      {/* Loading indicator */}
      {isLoading && <FeedLoadingSkeleton />}

      {/* End of feed */}
      {!hasMore && <EndOfFeedMessage />}
    </div>
  );
}
```

##### Key Configuration Explained

| Option | Value | What it does | Why this value |
|--------|-------|-------------|----------------|
| `root: null` | Viewport | Observes relative to the browser viewport, not a scroll container | The feed scrolls in the main document, not a nested container |
| `rootMargin: '0px 0px 500px 0px'` | 500px bottom margin | Extends the observation zone 500px below the viewport | Starts fetching before the user reaches the bottom — by the time they scroll there, data is likely already loaded. 500px ≈ 1-2 post heights. |
| `threshold: 0` | Any intersection | Fires when even 1px of the sentinel enters the margin zone | We don't need to wait for the sentinel to be fully visible — any intersection means "start loading" |

**Why 500px for rootMargin?**
*   Too small (50px) → user reaches the bottom and sees a loading spinner because the fetch started too late.
*   Too large (2000px) → fetches are triggered too eagerly, wasting bandwidth on posts the user might never reach.
*   500px (~1-2 post heights) is a sweet spot — the fetch completes in 200-500ms, which is usually before the user scrolls 500px further.

##### Why the Sentinel is Removed When `!hasMore`

When the server returns `hasMore: false` (no more posts), we unmount the sentinel. This stops the observer from firing. Without this, the observer would keep firing on every scroll to the bottom, causing unnecessary state updates.

---

#### 5.2.2 Virtualized Feed List for Variable Height Items

##### The Problem — Feeds Are Not Fixed Height Lists

Unlike a story tray (where each avatar is 80px wide) or a data table (where each row is 48px tall), feed posts have **wildly varying heights**:

```
┌─────────────────────────┐
│ Text-only post          │  ← ~120px
│ "Hello world!"          │
└─────────────────────────┘

┌─────────────────────────┐
│ Photo post              │  ← ~500px
│ [Author header]         │
│ [Large image 16:9]      │
│ [Actions bar]           │
│ [2 comments]            │
└─────────────────────────┘

┌─────────────────────────┐
│ Video post              │  ← ~600px
│ [Author header]         │
│ [Video player + controls│
│  with 16:9 aspect ratio]│
│ [Actions bar]           │
│ [5 comments expanded]   │
└─────────────────────────┘

┌─────────────────────────┐
│ Link preview post       │  ← ~350px
│ [Author + long text     │
│  with "See More"]       │
│ [Link card with thumb]  │
│ [Actions bar]           │
└─────────────────────────┘
```

This makes virtualization **significantly harder** than a fixed-size list because:
*   We can't compute `totalHeight = itemCount × itemHeight` upfront — each item's height is unknown until it renders.
*   We can't calculate which items are visible from `scrollTop / itemHeight` — we need cumulative heights.
*   The spacer div's total height changes as items are measured.

##### How Dynamic Measurement Works

The virtualizer uses a **measure-then-position** approach:

```
Step 1: Item mounts for the first time
  ↓
Step 2: Virtualizer measures its actual DOM height via ResizeObserver / ref callback
  ↓
Step 3: Height is stored in an internal cache: { index: 0 → 520px, index: 1 → 135px, ... }
  ↓
Step 4: Total height is recalculated (sum of all measured + estimated unmeasured)
  ↓
Step 5: Spacer div updates its height
  ↓
Step 6: Visible items are repositioned at correct `top` offsets
```

**Before measurement**, the virtualizer uses `estimateSize` as a guess. After measurement, it uses the **real** height. This means the scrollbar thumb might "jump" slightly as items are measured — but in practice, the jump is imperceptible because items near the viewport are measured first.

##### Implementation with Dynamic Sizing

```tsx
import { useRef } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualizedFeed({ posts }: { posts: Post[] }) {
  const scrollRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: posts.length,
    getScrollElement: () => scrollRef.current,
    // Estimate: average post height is ~400px (between text-only 120px and video 600px)
    estimateSize: () => 400,
    // NOT horizontal (vertical feed)
    overscan: 3,                 // render 3 extra items above and below
    // Enable dynamic measurement — items report their actual height
    measureElement: (element) => element.getBoundingClientRect().height,
  });

  return (
    <div
      ref={scrollRef}
      className="feed-scroll-container"
      style={{ height: '100vh', overflow: 'auto' }}
    >
      {/* Spacer — total height of all items (measured + estimated) */}
      <div
        style={{
          height: virtualizer.getTotalSize(),
          width: '100%',
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualItem) => {
          const post = posts[virtualItem.index];

          return (
            <div
              key={post.id}
              // This ref callback lets the virtualizer measure the real height
              ref={virtualizer.measureElement}
              data-index={virtualItem.index}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                // Transform instead of top for GPU-accelerated positioning
                transform: `translateY(${virtualItem.start}px)`,
              }}
            >
              <PostCard post={post} />
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

##### How the Measurement Flow Works Internally

| Step | What happens | DOM result |
|------|-------------|-----------|
| Mount | Virtualizer estimates all 500 posts at 400px each → total height = 200,000px | Spacer = 200,000px; ~8 items rendered (viewport 100vh ≈ 3200px / 400px + overscan) |
| First paint | The 8 rendered PostCards paint with their real heights | Item 0 = 520px, Item 1 = 135px, Item 2 = 480px, etc. |
| Measurement | `measureElement` callback fires for each → virtualizer records real heights | Cache: `{0: 520, 1: 135, 2: 480, 3: 400, ...}` |
| Recalculate | Total height = sum(measured items) + (unmeasured count × 400 estimate) | Spacer adjusts; items reposition at correct `translateY` |
| Scroll | User scrolls down → new items mount, get measured, old items unmount | Cache grows; total height converges to actual value |

##### Why `overscan: 3` for Feeds (Not 5 Like Trays)

*   Feed items are **tall** (120-600px). Three overscan items adds 360-1800px of buffer — plenty to prevent blank flashes.
*   Each PostCard is **heavier** than a tray avatar (complex DOM, media elements, event listeners). Rendering too many off-screen wastes resources.
*   With `overscan: 5` on 600px items, you'd have 3000px of off-screen items — overkill.

##### Performance Impact

| Metric | No virtualization (500 posts) | Virtualized (overscan=3) |
|--------|-------------------------------|--------------------------|
| DOM nodes | ~15,000+ (30 nodes per post average) | ~300 (~10 items × 30 nodes) |
| Initial render time | 500ms-2s | 30-80ms |
| Memory usage | High (all 500 images decoded) | Low (only ~10 images loaded) |
| Scroll smoothness | Degrades with post count | Consistent 60fps |
| Bundle size added | 0 | ~5-8KB (`@tanstack/react-virtual`) |

##### When to Virtualize a Feed

| Scenario | Recommendation | Why |
|----------|---------------|-----|
| **< 20 posts** (dashboard feed, profile page) | **Don't** | Overhead not worth it; posts unmount on navigate anyway |
| **20-50 posts** | **Optional** | Depends on post complexity and target devices |
| **50+ posts** (infinite scroll feed) | **Always** | DOM bloat causes measurable jank; memory grows unbounded |
| **Low-end devices** | **Always (even < 20)** | Limited memory; GPU compositing is slower |

**Alternative to virtualization — DOM recycling:**
Instead of mounting/unmounting, you can **reuse** existing DOM nodes by swapping their content as new items scroll into view. This avoids mount/unmount cost but is significantly more complex. Libraries like `react-virtuoso` handle this internally.

---

#### 5.2.3 Scroll Position Restoration

##### The Problem

User scrolls 50 posts deep → taps on a post to see full comments → navigates to the post detail page → presses the browser back button. **Where should the feed be?**

Without scroll restoration, the feed resets to the top. The user loses their place and has to scroll through 50 posts again. This is one of the most frustrating UX issues in infinite scroll feeds.

##### Approaches

| Approach | How it works | Pros | Cons |
|----------|-------------|------|------|
| **Browser native `scrollRestoration`** | Set `history.scrollRestoration = 'manual'` → save `scrollY` in session storage on navigate → restore on back | Simple; no library | Doesn't work with virtualized lists (items may not exist in DOM yet) |
| **Route-level state preservation** | Cache the feed state (posts array, scroll position) when navigating away; restore on return | Works with virtualized lists | Requires careful state management; memory cost of caching 50+ posts |
| **Keep-alive / Offscreen** | Keep the feed page mounted but hidden when navigating to post detail (CSS `display: none` or React Offscreen) | Perfect restoration (nothing unmounts) | Higher memory usage; hidden page still holds DOM/images |
| **Session storage + scroll-to-index** | Save `lastVisiblePostId` to sessionStorage → on return, fetch feed up to that post → `scrollToIndex()` | Works even after page reload | Re-fetch cost; slight delay before scroll position is restored |

##### Recommended Approach for Virtualized Feeds

```tsx
function useFeedScrollRestoration(virtualizer, posts) {
  const STORAGE_KEY = 'feed_scroll_state';

  // Save state before navigating away
  useEffect(() => {
    const handleBeforeNavigate = () => {
      const scrollEl = virtualizer.scrollElement;
      if (!scrollEl) return;

      const state = {
        scrollTop: scrollEl.scrollTop,
        firstVisiblePostId: getFirstVisiblePostId(virtualizer, posts),
        timestamp: Date.now(),
      };
      sessionStorage.setItem(STORAGE_KEY, JSON.stringify(state));
    };

    // Listen for route changes (framework-specific)
    window.addEventListener('beforeunload', handleBeforeNavigate);
    // Also hook into your router's navigation events
    return () => window.removeEventListener('beforeunload', handleBeforeNavigate);
  }, [virtualizer, posts]);

  // Restore state on mount (back navigation)
  useEffect(() => {
    const saved = sessionStorage.getItem(STORAGE_KEY);
    if (!saved) return;

    const state = JSON.parse(saved);
    // Only restore if navigation was recent (within 30 minutes)
    if (Date.now() - state.timestamp > 30 * 60 * 1000) return;

    // Find the post index and scroll to it
    const index = posts.findIndex((p) => p.id === state.firstVisiblePostId);
    if (index >= 0) {
      virtualizer.scrollToIndex(index, { align: 'start' });
    }

    sessionStorage.removeItem(STORAGE_KEY);
  }, [posts.length]); // trigger when posts are loaded
}
```

**Why `firstVisiblePostId` instead of `scrollTop`?**
*   `scrollTop` is based on pixel position, which depends on all previous items being rendered with their exact heights.
*   In a virtualized list, items above the viewport haven't been measured yet on a fresh mount — the total height is estimated.
*   Using `scrollToIndex` lets the virtualizer jump directly to the right item regardless of estimated heights.

---

#### 5.2.4 New Posts Banner Pattern

##### The Concept

When new posts arrive via SSE/WebSocket while the user is scrolling, we do **NOT** prepend them directly to the feed. Doing so would shift the entire feed down and disrupt the user's scroll position.

Instead, we **buffer** new posts and show a banner: "5 new posts — tap to see."

```
┌──────────────────────────────────┐
│  ┌────────────────────────────┐  │
│  │  ↑ 5 new posts available   │  │  ← banner (clickable)
│  └────────────────────────────┘  │
│                                  │
│  PostCard 1 (current top post)   │  ← user is reading here
│  PostCard 2                      │
│  PostCard 3                      │
│  PostCard 4                      │
└──────────────────────────────────┘
```

When the user clicks the banner → buffer is flushed → new posts are prepended to the feed → user scrolls to top to see them.

##### Implementation

```tsx
function FeedWithNewPostsBanner() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [newPostsBuffer, setNewPostsBuffer] = useState<Post[]>([]);

  // SSE connection for real-time new posts
  useEffect(() => {
    const eventSource = new EventSource('/api/feed/stream');

    eventSource.onmessage = (event) => {
      const newPost = JSON.parse(event.data);
      // DON'T add directly to posts — add to buffer
      setNewPostsBuffer((prev) => [newPost, ...prev]);
    };

    return () => eventSource.close();
  }, []);

  // Flush buffer when user clicks the banner
  const showNewPosts = useCallback(() => {
    setPosts((prev) => [...newPostsBuffer, ...prev]);
    setNewPostsBuffer([]);
    // Scroll to top to show new posts
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }, [newPostsBuffer]);

  return (
    <div className="feed">
      {newPostsBuffer.length > 0 && (
        <button
          className="new-posts-banner"
          onClick={showNewPosts}
          aria-live="polite"
        >
          ↑ {newPostsBuffer.length} new post{newPostsBuffer.length > 1 ? 's' : ''} available
        </button>
      )}

      <div className="feed-list">
        {posts.map((post) => (
          <PostCard key={post.id} post={post} />
        ))}
      </div>
    </div>
  );
}
```

##### Why Buffer Instead of Auto Prepend?

| Behavior | What happens | UX impact |
|----------|-------------|-----------|
| **Auto-prepend** | New post is added at index 0; all other posts shift down by the new post's height | User is reading post #3, which suddenly jumps down. User loses their place. Extremely jarring. |
| **Buffer + banner** | New posts are queued; a small banner appears at the top. User controls when to see them. | User stays in flow. Taps banner when ready. This is what Facebook, Twitter, LinkedIn all do. |

##### Edge Cases

| Scenario | Handling |
|----------|---------|
| **User is at the top of the feed** | Auto-prepend is acceptable here (they're not reading mid-feed). Check `window.scrollY < 100` before deciding. |
| **100+ new posts buffered** | Cap the buffer at ~50 posts. Older buffered posts are dropped (they'll be fetched in the regular feed). Show "50+ new posts". |
| **Banner with keyboard (a11y)** | Banner should be `aria-live="polite"` so screen readers announce it. On click, focus moves to the first new post. |

---

#### 5.2.5 Pull to Refresh (Mobile Web)

On mobile, users expect to swipe down from the top of the feed to refresh it — the "pull-to-refresh" gesture.

**Two approaches:**

| Approach | How it works | Notes |
|----------|-------------|-------|
| **CSS `overscroll-behavior` + browser native** | Set `overscroll-behavior-y: contain` on the feed container; browsers like Chrome Android show a built-in refresh indicator | Zero JS, but you can't customize the indicator or the action |
| **Custom JS implementation** | Listen for `touchstart` / `touchmove` / `touchend` when `scrollTop === 0`; track pull distance; show custom spinner; trigger refetch | Full control over visuals and behavior; more code |

**Simple custom pull-to-refresh:**

```tsx
function usePullToRefresh(onRefresh: () => Promise<void>) {
  const [pulling, setPulling] = useState(false);
  const [pullDistance, setPullDistance] = useState(0);
  const startY = useRef(0);

  const handleTouchStart = (e: TouchEvent) => {
    if (window.scrollY === 0) {
      startY.current = e.touches[0].clientY;
      setPulling(true);
    }
  };

  const handleTouchMove = (e: TouchEvent) => {
    if (!pulling) return;
    const distance = Math.max(0, e.touches[0].clientY - startY.current);
    setPullDistance(Math.min(distance, 150)); // cap at 150px
  };

  const handleTouchEnd = async () => {
    if (pullDistance > 80) { // threshold: 80px to trigger refresh
      await onRefresh();
    }
    setPulling(false);
    setPullDistance(0);
  };

  return { pullDistance, pulling, handleTouchStart, handleTouchMove, handleTouchEnd };
}
```

**Key detail:** Pull-to-refresh should only activate when `scrollY === 0` — if the user is mid-feed and scrolls up, it's regular scrolling, not a refresh gesture.

---

#### 5.2.6 Lazy Loading Media in Feed Posts

##### The Problem

A feed page might render 10 visible posts, each with 1-3 images or a video. Without lazy loading:
*   10 posts × 2 images = **20 image requests** before the user even starts scrolling.
*   Videos might auto-download even when off-screen, wasting **megabytes** of bandwidth.
*   On slow networks, critical content (text, layout) is delayed by competing media downloads.

##### Strategy by Media Type

| Media type | Strategy | Implementation |
|------------|----------|----------------|
| **Images above the fold** (first 2-3 posts) | `loading="eager"` | Load immediately for fast LCP |
| **Images below the fold** | `loading="lazy"` | Browser defers until near viewport |
| **Videos** | Don't load until visible; show thumbnail | IntersectionObserver triggers `video.play()` |
| **Link preview thumbnails** | `loading="lazy"` | Small images; defer safely |
| **User avatars** | `loading="eager"` (small, < 5KB each) | Already cached from previous views |

##### Image Lazy Loading

```tsx
function PostMedia({ media, postIndex }: { media: Media; postIndex: number }) {
  if (media.type === 'image') {
    return (
      <img
        src={media.url}
        alt={media.altText || 'Post image'}
        width={media.width}
        height={media.height}
        // First 3 posts are above the fold — load eagerly
        loading={postIndex < 3 ? 'eager' : 'lazy'}
        // Prevent layout shift by setting aspect ratio
        style={{ aspectRatio: `${media.width} / ${media.height}` }}
      />
    );
  }

  // ...video handling below
}
```

##### Video Autoplay on Visibility

Videos in a feed should only play when visible (saves bandwidth, CPU, battery):

```tsx
function FeedVideoPlayer({ videoUrl, thumbnailUrl }: Props) {
  const videoRef = useRef<HTMLVideoElement>(null);
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        const video = videoRef.current;
        if (!video) return;

        if (entry.isIntersecting && entry.intersectionRatio > 0.5) {
          // More than 50% visible — start playing
          video.play().catch(() => {}); // catch autoplay policy rejection
        } else {
          // Less than 50% visible — pause
          video.pause();
        }
      },
      { threshold: [0, 0.5, 1.0] }  // fire at 0%, 50%, 100% visibility
    );

    observer.observe(container);
    return () => observer.disconnect();
  }, []);

  return (
    <div ref={containerRef}>
      <video
        ref={videoRef}
        src={videoUrl}
        poster={thumbnailUrl}   // show thumbnail before video loads
        muted                   // required for autoplay in most browsers
        playsInline             // prevent fullscreen on iOS
        loop
        preload="none"          // don't preload video data at all
      />
    </div>
  );
}
```

**Why `threshold: [0, 0.5, 1.0]`?**
*   `0` — fires when the video enters/exits the viewport (for pause).
*   `0.5` — fires when 50% is visible (for play — user can actually see it).
*   `1.0` — fires when fully visible (optional — can be used for analytics "impression" tracking).

##### Placeholder Strategies for Feed Images

| Strategy | Visual result | Implementation | Best for |
|----------|--------------|----------------|----------|
| **Aspect ratio box** | Correct-sized empty space → image fills in | CSS `aspect-ratio` or padding-top trick | Preventing CLS (layout shift) |
| **Dominant color** | Solid color matching the image's dominant color | Backend provides `dominantColor: "#3a7bd5"` | Clean, fast, minimal data |
| **Blur-up (LQIP)** | Tiny 16×16 base64 thumbnail → blurred → sharp image loads | Inline base64 in API response + CSS blur | Premium feel (like Medium) |
| **Skeleton shimmer** | Grey box with animated shimmer gradient | CSS animation | When load time is noticeable |

```tsx
// Blur-up placeholder example
function BlurUpImage({ src, placeholder, alt, width, height }) {
  const [loaded, setLoaded] = useState(false);

  return (
    <div style={{ position: 'relative', aspectRatio: `${width}/${height}` }}>
      {/* Tiny blurred placeholder — always visible until real image loads */}
      <img
        src={placeholder}  // data:image/jpeg;base64,/9j/4AA... (tiny 16x16)
        alt=""
        aria-hidden="true"
        style={{
          width: '100%',
          height: '100%',
          filter: loaded ? 'none' : 'blur(20px)',
          transition: 'filter 0.3s ease',
          objectFit: 'cover',
        }}
      />
      {/* Real image — loads on top */}
      <img
        src={src}
        alt={alt}
        loading="lazy"
        onLoad={() => setLoaded(true)}
        style={{
          position: 'absolute',
          top: 0, left: 0,
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

---

#### 5.2.7 Scroll Performance Considerations

| Concern | What happens | Mitigation |
|---------|-------------|------------|
| **Scroll event flooding** | If using scroll events (instead of IntersectionObserver), hundreds of events fire per second | Use IntersectionObserver for infinite scroll; use `passive: true` for any remaining scroll listeners |
| **Layout thrashing** | Reading `scrollTop`, `offsetHeight`, `getBoundingClientRect()` forces the browser to recalculate layout | Batch DOM reads; cache values; use `requestAnimationFrame` for scroll-linked updates |
| **Image decode jank** | Large feed images decoding on main thread during scroll | Use `img.decode()` before displaying; use `content-visibility: auto` for off-screen posts |
| **Video resource drain** | Multiple videos loaded simultaneously consume memory and CPU | Only play visible video; `preload="none"` for off-screen; pause off-screen videos |
| **Unbounded DOM growth** | Without virtualization, every loaded post stays in the DOM forever | Virtualize; or implement a "DOM budget" — remove posts far above the viewport from the DOM |
| **CSS paint complexity** | Complex shadows, gradients, and filters on each post card trigger expensive repaints | Use `will-change: transform` on cards that animate; simplify visual effects |

**`content-visibility: auto` for non-virtualized feeds:**

If you choose not to use a virtualization library, CSS `content-visibility: auto` can provide similar benefits with zero JavaScript:

```css
.post-card {
  content-visibility: auto;
  /* Tell browser the expected size so scrollbar doesn't jump */
  contain-intrinsic-size: auto 400px;
}
```

This tells the browser: "Don't render this element's contents until it's near the viewport." The browser skips layout, paint, and compositing for off-screen posts. When the user scrolls near, the browser renders the content just in time.

| Feature | Virtualization | `content-visibility: auto` |
|---------|---------------|---------------------------|
| DOM node count | Only visible items exist | All items exist, but off-screen are skipped |
| JavaScript overhead | Scroll listener + recalculation | Zero JS |
| Browser support | All browsers (it's JS) | Chrome 85+, Edge 85+, Firefox 125+ (no Safari until 18) |
| Layout stability | Perfect (absolute positioning) | Slight scrollbar jumps (mitigated by `contain-intrinsic-size`) |
| Complexity | Moderate | Trivial (one CSS property) |

**Recommendation:** Use `content-visibility: auto` as a quick win if you're not virtualizing. Use full virtualization for feeds with 50+ posts.

---

#### 5.2.8 Decision Matrix

| Scenario | Strategy | Tool / Approach | Notes |
|----------|----------|----------------|-------|
| **< 20 posts** (profile, dashboard) | Plain DOM rendering + `loading="lazy"` images | No library needed | `content-visibility: auto` for free optimization |
| **20-50 posts** (bounded feed) | `content-visibility: auto` | CSS only | Zero JS overhead; good enough for moderate lists |
| **50+ posts** (infinite scroll) | Full virtualization with dynamic measurement | `@tanstack/react-virtual` or `react-virtuoso` | Handles variable heights; keeps DOM at O(1) |
| **Video-heavy feed** | IntersectionObserver for play/pause | Custom hook | Only one video should play at a time |
| **Mobile web** | Pull-to-refresh + momentum scroll | Custom touch handler or browser native | `-webkit-overflow-scrolling: touch` for iOS |
| **New posts while scrolling** | Buffer + banner pattern | Custom with SSE | Never auto-prepend mid-scroll |
| **Back navigation** | Scroll position restoration with virtualizer | `scrollToIndex` + sessionStorage | Save `firstVisiblePostId`, not `scrollTop` |
| **Slow network** | Skeleton + blur-up placeholders | CSS skeleton + LQIP | Prevent CLS; show something immediately |

---

### 5.3 Reusability Strategy

*   **`PostCard`**: Generic container; delegates to `PostMedia`, `PostActions`, `CommentList`. Configurable for different feed types (main feed, profile feed, search results).
*   **`MediaRenderer`**: Shared image/video component with lazy loading and placeholder support. Used in posts, link previews, and message attachments.
*   **`EngagementBar` / `PostActions`**: Like, Comment, Share buttons reusable across posts, comments, and shared items.
*   **`InfiniteScrollSentinel`**: Generic sentinel + IntersectionObserver hook, reusable for any paginated list (feed, comments, notifications).
*   **`VirtualList`**: Wraps `@tanstack/react-virtual` with project-specific defaults; used for feeds, comment threads, member lists.
*   Design system tokens for colors, spacing, typography, elevation.

---

### 5.4 Module Organization

```
src/
 ├── features/
 │    └── feed/
 │         ├── components/
 │         │    ├── FeedPage.tsx
 │         │    ├── FeedList.tsx
 │         │    ├── PostCard.tsx
 │         │    ├── PostHeader.tsx
 │         │    ├── PostContent.tsx
 │         │    ├── PostMedia.tsx
 │         │    ├── PostActions.tsx
 │         │    ├── CommentList.tsx
 │         │    ├── CommentItem.tsx
 │         │    ├── CreateComment.tsx
 │         │    ├── CreatePostBox.tsx
 │         │    ├── NewPostsBanner.tsx
 │         │    └── FeedSkeleton.tsx
 │         ├── hooks/
 │         │    ├── useFeedInfiniteScroll.ts
 │         │    ├── useFeedSSE.ts
 │         │    ├── useOptimisticLike.ts
 │         │    ├── useVideoAutoplay.ts
 │         │    └── useFeedScrollRestoration.ts
 │         ├── store/
 │         │    └── feedStore.ts
 │         ├── api/
 │         │    └── feedApi.ts
 │         ├── utils/
 │         │    └── feedHelpers.ts
 │         └── types/
 │              └── feed.types.ts
 ├── shared/
 │    ├── components/ (MediaRenderer, InfiniteScrollSentinel, Skeleton, Avatar)
 │    ├── hooks/ (useIntersection, usePullToRefresh, useDebounce)
 │    └── utils/ (dateFormat, numberFormat, mediaLoader)
```

---

## 6. High Level Data Flow Explanation

### 6.1 Initial Load Flow

```
1. User navigates to feed (or opens app)
     ↓
2. Server renders feed shell via SSR (empty skeleton + first 10 posts if SSR'd)
     ↓
3. Client hydrates → FeedPage mounts
     ↓
4. Fetch initial feed: GET /api/feed?limit=10
     ↓
5. Render PostCards; lazy-load images for below-the-fold posts
     ↓
6. Establish SSE connection: GET /api/feed/stream
     ↓
7. Sentinel mounts at bottom → IntersectionObserver starts watching
     ↓
8. User scrolls → sentinel enters viewport → fetch page 2
```

---

### 6.2 User Interaction Flow

**Optimistic Updates — Making Likes Feel Instant**

When a user taps "Like", the UI must respond **instantly**. Waiting 200-500ms for the API round-trip breaks the feeling of direct manipulation.

```
User taps Like button
     ↓
1. Immediately update local state:
   - post.isLiked = true
   - post.likeCount += 1
   - button shows filled heart + animation
     ↓
2. Fire API call (background):
   POST /api/post/:id/like
     ↓
3a. API succeeds → done (state is already correct)
     ↓
3b. API fails → ROLLBACK local state:
   - post.isLiked = false
   - post.likeCount -= 1
   - show subtle error toast
```

**Implementation:**

```tsx
function useOptimisticLike(postId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: () => likePost(postId),
    // Optimistically update BEFORE the server responds
    onMutate: async () => {
      // Cancel any in-flight refetches so they don't overwrite our optimistic update
      await queryClient.cancelQueries({ queryKey: ['feed'] });

      // Snapshot the previous value (for rollback)
      const previousFeed = queryClient.getQueryData(['feed']);

      // Optimistically update
      queryClient.setQueryData(['feed'], (old: FeedData) => ({
        ...old,
        posts: old.posts.map((post) =>
          post.id === postId
            ? { ...post, isLiked: true, likeCount: post.likeCount + 1 }
            : post
        ),
      }));

      return { previousFeed }; // context for rollback
    },
    // On error, rollback to the snapshot
    onError: (err, variables, context) => {
      queryClient.setQueryData(['feed'], context.previousFeed);
    },
    // After success or error, refetch to ensure server state
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['feed'] });
    },
  });
}
```

**Comment flow — Optimistic with placeholder:**

```
User types comment → taps "Post"
     ↓
1. Immediately add a temporary comment to the list:
   { id: 'temp_123', text: userInput, author: currentUser, isPending: true }
     ↓
2. Show comment with a subtle "sending..." indicator
     ↓
3. POST /api/post/:id/comment
     ↓
4a. Success → replace temp comment with server-returned comment (has real ID)
4b. Failure → mark comment as "failed" with retry button
```

---

### 6.3 Error and Retry Flow

*   **Feed fetch fails**: Show cached feed data (React Query stale-while-revalidate) + error banner with retry button.
*   **Image fails to load**: Show broken-image placeholder with alt text; don't crash the entire PostCard.
*   **Like/Comment API fails**: Rollback optimistic update; show toast "Failed to like post — tap to retry."
*   **SSE connection drops**: Auto-reconnect with exponential backoff (EventSource does this natively). Show a subtle "Reconnecting..." indicator after 5s.
*   **Infinite scroll fetch fails**: Show "Failed to load more posts" inline with a retry button (don't block the existing feed).
*   **Network offline**: Show cached feed + "You're offline" banner. Queue interactions (likes, comments) and replay when online.

---

## 7. Data Modelling (Frontend Perspective)

### 7.1 Core Data Entities

*   **Post** — a single feed item (text, images, video, link preview).
*   **Comment** — a reply attached to a post.
*   **User** — author of a post or comment (lightweight profile data).
*   **Media** — image or video attachment on a post.
*   **LinkPreview** — OG metadata for shared links.
*   **Reaction** — like / emoji reaction on a post or comment.

---

### 7.2 Data Shape

```ts
type User = {
  id: string;
  name: string;
  avatarUrl: string;
  profileUrl: string;
};

type Media = {
  id: string;
  type: 'image' | 'video';
  url: string;
  thumbnailUrl?: string;
  width: number;
  height: number;
  altText?: string;
  dominantColor?: string;       // for placeholder
  blurhash?: string;            // for blur-up placeholder
  duration?: number;            // seconds (for video)
};

type LinkPreview = {
  url: string;
  title: string;
  description: string;
  thumbnailUrl?: string;
  siteName: string;
  favicon?: string;
};

type Comment = {
  id: string;
  author: User;
  text: string;
  createdAt: string;
  likeCount: number;
  isLiked: boolean;
  replies?: Comment[];          // for threaded comments
};

type Post = {
  id: string;
  author: User;
  content: string;              // text body (may contain mentions, hashtags)
  media: Media[];               // 0 to many images/videos
  linkPreview?: LinkPreview;    // present if post contains a URL
  likeCount: number;
  commentCount: number;
  shareCount: number;
  isLiked: boolean;
  isSaved: boolean;
  comments: Comment[];          // first 2-3 preview comments
  createdAt: string;
  editedAt?: string;
  visibility: 'public' | 'friends' | 'private';
};
```

---

### 7.3 Entity Relationships

*   **One-to-Many**: One `Post` → many `Media` items; One `Post` → many `Comment` items.
*   **One-to-One**: One `Post` → one optional `LinkPreview`.
*   **Self-referential**: `Comment` → `replies: Comment[]` (threaded comments).
*   **Many-to-One**: Many `Post` / `Comment` → one `User` (author).

**Normalized vs Denormalized:**

| Approach | Storage | Use case | Trade-off |
|----------|---------|----------|-----------|
| **Denormalized** (inline) | `post.author = { id, name, avatarUrl }` | Simple reads; each post is self-contained | Duplicated user data if same author has many posts |
| **Normalized** (by reference) | `post.authorId` + separate `users: { [id]: User }` map | Efficient updates (change user avatar once, all posts reflect it) | More complex selectors; need a store |

**Recommendation:** Use **denormalized** for feed display (each post carries its own author data). Use **normalized** only if the app has a global user cache (common in large SPAs with a Redux/Zustand store).

---

### 7.4 UI Specific Data Models

```ts
// View model for the feed page
type FeedState = {
  posts: Post[];
  cursor: string | null;           // for next page
  hasMore: boolean;
  isLoading: boolean;
  isRefreshing: boolean;           // pull-to-refresh
  error: string | null;
};

// Buffer for real-time new posts (not yet displayed)
type NewPostsBuffer = {
  posts: Post[];
  count: number;                   // displayed in banner
};

// Derived state
type PostViewModel = Post & {
  relativeTime: string;            // "2h ago" — computed from createdAt
  truncatedContent: string;        // first 300 chars + "See More"
  formattedLikeCount: string;      // "1.2K" — locale-aware formatting
  isExpanded: boolean;             // full text shown
  areCommentsExpanded: boolean;    // comment section open
};
```

---

## 8. State Management Strategy

### 8.1 State Classification

| State Type | Examples | Storage |
|---|---|---|
| **Server State** | Feed posts, comments, user profiles | React Query / SWR cache (auto-managed) |
| **Global App State** | Current user session, theme, notification count | Zustand / Redux global store |
| **Feature State** | New posts buffer, SSE connection status | Feature-level Zustand slice or React Context |
| **Component Local State** | Comment input text, "See More" expanded, reply drawer open | `useState` / `useReducer` |
| **Derived / Computed State** | Relative timestamps, formatted counts, truncated content | Selectors / `useMemo` |

---

### 8.2 State Ownership

*   **FeedPage** owns the SSE connection and the new-posts buffer. It connects on mount and disconnects on unmount.
*   **FeedList** owns the infinite scroll state (cursor, hasMore, isLoading) via React Query's `useInfiniteQuery`.
*   **PostCard** owns its own UI state (expanded text, expanded comments) locally. Social interactions (like, comment) are mutations that update the shared React Query cache.
*   **CommentList** manages its own pagination if a post has many comments (separate `useInfiniteQuery` for comments per post).

**Prop drilling is minimal:**
*   Feed passes `post` object to `PostCard` as a single prop.
*   PostCard reads `post.id` and uses it to call mutation hooks (like, comment).
*   No need for deep prop chains — the React Query cache acts as the shared data layer.

---

### 8.3 Persistence Strategy

| Data | Persistence | Reason |
|------|------------|--------|
| Feed posts | React Query in-memory cache | Ephemeral; refetch on focus/revisit; `staleTime: 2min` |
| New posts buffer | Component state (in-memory) | Ephemeral; lost on page navigation |
| Scroll position | sessionStorage | Survive back-navigation; 30-min TTL |
| Draft post text | localStorage | Survive accidental tab close; cleared on successful post |
| User preferences (theme, muted words) | localStorage | Persist across sessions |
| Offline feed cache (PWA) | IndexedDB via Service Worker | Show cached feed when offline |

---

## 9. High Level API Design (Frontend POV)

### 9.1 Required APIs

| API | Method | Description |
|-----|--------|-------------|
| `/api/feed` | **GET** | Fetch paginated feed posts (cursor-based) |
| `/api/feed/stream` | **GET (SSE)** | Stream new posts in real-time |
| `/api/post` | **POST** | Create a new post |
| `/api/post/:id` | **GET** | Fetch single post with full details |
| `/api/post/:id/like` | **POST** | Like / unlike a post |
| `/api/post/:id/comments` | **GET** | Fetch paginated comments for a post |
| `/api/post/:id/comment` | **POST** | Add a comment to a post |
| `/api/post/:id/share` | **POST** | Share / repost a post |

---

### 9.2 Cursor Based Pagination Strategy (Deep Dive)

#### Why Cursor Over Offset for Feeds

Feed data is **highly dynamic** — posts are constantly being added, deleted, and reordered by the ranking algorithm. This makes offset-based pagination unreliable:

```
OFFSET-BASED PROBLEM:

  Page 1 (offset=0, limit=10): Returns posts [A, B, C, D, E, F, G, H, I, J]
  
  While user reads page 1, a new post X is inserted at the top by the ranking algo.
  
  Page 2 (offset=10, limit=10): Returns posts [J, K, L, M, N, O, P, Q, R, S]
                                                 ↑
                                           DUPLICATE! Post J was already shown on page 1.
                                           (Because everything shifted down by 1)
```

With **cursor-based** pagination, we don't use a numeric offset. Instead, we say: "Give me 10 posts **after** this specific post ID (or timestamp)."

```
CURSOR-BASED SOLUTION:

  Page 1: Returns posts [A, B, C, D, E, F, G, H, I, J]
           cursor = "J_timestamp_score"  (an opaque token pointing to the last item)
  
  New post X is inserted at the top. Doesn't matter.
  
  Page 2 (cursor="J_timestamp_score"): Returns posts [K, L, M, N, O, P, Q, R, S, T]
                                                       ↑
                                                 No duplicates! We asked for items AFTER J.
```

#### How Cursor Pagination Works

**The cursor is an opaque string** — the frontend doesn't parse it. It encodes whatever the backend needs to find the next page (typically a combination of the last item's sort key + ID).

```
┌──────────────────────────────────────────────────────────┐
│ Frontend perspective:                                    │
│                                                          │
│   GET /api/feed?limit=10                                 │
│   → Response: { posts: [...], nextCursor: "abc123" }     │
│                                                          │
│   GET /api/feed?cursor=abc123&limit=10                   │
│   → Response: { posts: [...], nextCursor: "def456" }     │
│                                                          │
│   GET /api/feed?cursor=def456&limit=10                   │
│   → Response: { posts: [...], nextCursor: null }         │
│                            ↑ null = no more pages        │
└──────────────────────────────────────────────────────────┘
```

#### Implementation with React Query

```tsx
function useFeed() {
  return useInfiniteQuery({
    queryKey: ['feed'],
    queryFn: async ({ pageParam }) => {
      const url = pageParam
        ? `/api/feed?cursor=${pageParam}&limit=10`
        : `/api/feed?limit=10`;
      const res = await fetch(url);
      return res.json();
    },
    initialPageParam: null as string | null,
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    staleTime: 2 * 60 * 1000,      // 2 minutes before refetch
    refetchOnWindowFocus: true,     // refresh feed when user returns to tab
  });
}

// Usage in FeedList
function FeedList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
  } = useFeed();

  const allPosts = data?.pages.flatMap((page) => page.posts) ?? [];

  // IntersectionObserver triggers fetchNextPage() when sentinel is visible
  // (implementation from section 5.2.1)

  return (
    <div>
      {allPosts.map((post) => (
        <PostCard key={post.id} post={post} />
      ))}
      {hasNextPage && <FeedSentinel onIntersect={fetchNextPage} />}
      {isFetchingNextPage && <FeedLoadingSkeleton />}
    </div>
  );
}
```

#### Comparison: Cursor vs Offset vs Keyset

| Criterion | Offset (`?page=3&limit=10`) | Cursor (`?cursor=abc123`) | Keyset (`?after_id=123&after_score=0.95`) |
|-----------|------|--------|---------|
| Duplicate items on insert | Yes (items shift) | No | No |
| Missing items on delete | Yes | No | No |
| "Jump to page N" | Easy | Impossible | Impossible |
| Total count / progress | Easy (`COUNT(*)`) | Expensive | Expensive |
| Backend complexity | Low | Low-medium | Medium |
| Performance at deep pages | Slow (`OFFSET 10000`) | Fast (seeks from cursor) | Fast |
| **Best for** | Static, rarely-changing lists | Dynamic feeds (social, news) | Dynamic feeds (alternative to cursor) |

**For social feeds, cursor-based is the standard.** Facebook, Twitter/X, Instagram, and LinkedIn all use cursor-based pagination for their feeds.

---

### 9.3 Real Time Feed Updates (Deep Dive)

#### The Requirement

When a followed user publishes a new post, that post should appear in the viewer's feed **without requiring a manual page refresh**. Additionally, engagement counts on visible posts (likes, comments) may need live updates.

#### Three Approaches

##### Approach 1: Polling

```tsx
// Simple interval-based polling
useEffect(() => {
  const interval = setInterval(async () => {
    const res = await fetch('/api/feed/updates?since=' + lastFetchTimestamp);
    const { newPosts } = await res.json();
    if (newPosts.length > 0) {
      addToBuffer(newPosts);
    }
  }, 30000); // every 30 seconds

  return () => clearInterval(interval);
}, []);
```

##### Approach 2: Server Sent Events (SSE)

```tsx
useEffect(() => {
  const eventSource = new EventSource('/api/feed/stream');

  eventSource.addEventListener('new_post', (event) => {
    const post = JSON.parse(event.data);
    addToBuffer(post);  // buffer, don't auto-insert
  });

  eventSource.addEventListener('engagement_update', (event) => {
    const update = JSON.parse(event.data);
    // update like/comment counts on visible post
    updatePostEngagement(update.postId, update.likeCount, update.commentCount);
  });

  eventSource.onerror = () => {
    // Browser auto-reconnects SSE. But log for monitoring.
    console.warn('SSE connection lost, auto-reconnecting...');
  };

  return () => eventSource.close();
}, []);
```

##### Approach 3: WebSocket

```tsx
useEffect(() => {
  const ws = new WebSocket('wss://api.example.com/feed/ws');

  ws.onmessage = (event) => {
    const message = JSON.parse(event.data);
    switch (message.type) {
      case 'new_post':
        addToBuffer(message.post);
        break;
      case 'engagement_update':
        updatePostEngagement(message.postId, message.data);
        break;
    }
  };

  ws.onclose = () => {
    // Manual reconnect needed (unlike SSE)
    setTimeout(() => reconnect(), 3000);
  };

  return () => ws.close();
}, []);
```

#### Comparison Table

| Criterion | Polling | SSE | WebSocket |
|-----------|---------|-----|-----------|
| Direction | Client → Server (pull) | Server → Client (push) | Bidirectional |
| Latency | High (up to polling interval) | Low (server pushes immediately) | Lowest (persistent connection) |
| Server resource usage | Low (stateless requests) | Medium (one open connection per client) | High (persistent connection + state) |
| Auto-reconnect | N/A | Built-in (browser auto-reconnects) | Manual (must implement reconnection logic) |
| Browser support | Universal | All modern browsers | All modern browsers |
| Works with HTTP/2 | Yes | Yes (multiplexed) | Separate TCP connection |
| Proxy / CDN friendly | Yes | Yes (standard HTTP) | Can be problematic (needs upgrade) |
| Battery impact (mobile) | Medium (wake CPU each interval) | Low (idle connection) | Low (idle connection) |
| Complexity | Trivial | Low | Moderate |
| **Best for** | Low-frequency updates; simple setups | Feed updates; one-way server push | Chat; collaborative editing; bidirectional |

#### Recommended Strategy for Feeds

**Use SSE for new post notifications.** Rationale:
*   Feeds only need **server → client** push (new posts). The client doesn't need to push data via the real-time channel (likes/comments use REST).
*   SSE is simpler than WebSocket — auto-reconnects, works with HTTP/2 multiplexing, and is proxy-friendly.
*   SSE uses regular HTTP, so it works through corporate firewalls and CDNs that might block WebSocket upgrades.

**Use polling as a fallback** for environments where SSE isn't supported or for engagement count updates (less time-critical).

##### Backend SSE Endpoint Example

```js
app.get('/api/feed/stream', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.flushHeaders();

  const userId = req.user.id;

  const sendEvent = (type, data) => {
    res.write(`event: ${type}\n`);
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };

  // Listen for new posts relevant to this user
  feedEmitter.on(`newPost:${userId}`, (post) => {
    sendEvent('new_post', post);
  });

  // Listen for engagement updates on posts the user has seen
  feedEmitter.on(`engagement:${userId}`, (update) => {
    sendEvent('engagement_update', update);
  });

  // Heartbeat every 30s to keep connection alive
  const heartbeat = setInterval(() => {
    res.write(': heartbeat\n\n');
  }, 30000);

  req.on('close', () => {
    clearInterval(heartbeat);
    feedEmitter.off(`newPost:${userId}`);
    feedEmitter.off(`engagement:${userId}`);
  });
});
```

---

### 9.4 Request and Response Structure

**GET /api/feed**

```json
// Request
// GET /api/feed?limit=10

// Response
{
  "posts": [
    {
      "id": "p_001",
      "author": {
        "id": "u_123",
        "name": "Zeeshan Ali",
        "avatarUrl": "https://cdn.example.com/avatars/u_123.jpg",
        "profileUrl": "/user/zeeshan_ali"
      },
      "content": "Just published a new article on frontend system design!",
      "media": [
        {
          "id": "m_001",
          "type": "image",
          "url": "https://cdn.example.com/posts/m_001.jpg",
          "thumbnailUrl": "https://cdn.example.com/posts/m_001_thumb.jpg",
          "width": 1200,
          "height": 630,
          "altText": "System design diagram",
          "dominantColor": "#1a73e8"
        }
      ],
      "linkPreview": null,
      "likeCount": 342,
      "commentCount": 28,
      "shareCount": 15,
      "isLiked": false,
      "isSaved": false,
      "comments": [
        {
          "id": "c_001",
          "author": { "id": "u_456", "name": "Ali", "avatarUrl": "..." },
          "text": "Great article!",
          "createdAt": "2026-03-17T08:30:00Z",
          "likeCount": 5,
          "isLiked": false
        }
      ],
      "createdAt": "2026-03-17T08:00:00Z",
      "visibility": "public"
    }
  ],
  "nextCursor": "eyJzY29yZSI6MC45NSwidHMiOiIyMDI2LTAzLTE3In0=",
  "hasMore": true
}
```

**POST /api/post/:id/like**

```json
// Request
// POST /api/post/p_001/like
{
  "action": "like"   // or "unlike"
}

// Response
{
  "success": true,
  "likeCount": 343
}
```

**POST /api/post/:id/comment**

```json
// Request
{
  "text": "This is a great post!",
  "parentCommentId": null   // null for top-level, ID for threaded reply
}

// Response
{
  "comment": {
    "id": "c_099",
    "author": { "id": "u_789", "name": "Sara", "avatarUrl": "..." },
    "text": "This is a great post!",
    "createdAt": "2026-03-17T09:00:00Z",
    "likeCount": 0,
    "isLiked": false
  }
}
```

**SSE Event Examples**

```
event: new_post
data: {"id":"p_305","author":{"id":"u_456","name":"Ali"},"content":"New post here!","media":[],"createdAt":"2026-03-17T09:30:00Z"}

event: engagement_update
data: {"postId":"p_001","likeCount":350,"commentCount":30}
```

---

### 9.5 Error Handling and Status Codes

| Status | Scenario | Frontend Handling |
|--------|----------|-------------------|
| `200` | Success | Render data |
| `304` | Not Modified (cached) | Use cached version |
| `400` | Bad request (invalid cursor, malformed input) | Show error toast; reset pagination |
| `401` | Unauthorized (token expired) | Redirect to login; clear session |
| `403` | Forbidden (private post, blocked user) | Remove post from feed; show "Content not available" |
| `404` | Post deleted | Remove from feed; show toast "Post no longer available" |
| `429` | Rate limit exceeded | Exponential backoff + retry; disable further fetches temporarily |
| `500` | Server error | Show error banner with retry button; keep existing feed visible |
| `503` | Service unavailable | Show cached feed; retry with backoff |

---

## 10. Caching Strategy

### 10.1 What to Cache

*   **Feed data (posts)**: Cached with short TTL (2-5 min) — feed is personalized and changes frequently.
*   **User profiles / avatars**: Cached aggressively (avatars rarely change; `Cache-Control: max-age=86400`).
*   **Post images and videos**: Cached by browser and CDN (`max-age=31536000` with hashed filenames — immutable content).
*   **Comments**: Cached per-post; invalidated on new comment or after 2 min.
*   **API responses**: React Query / SWR handles in-memory cache with stale-while-revalidate.
*   **Link preview metadata**: Cached by the backend; frontend caches with the post data.

### 10.2 Where to Cache

| Data | Cache Location | TTL |
|------|----------------|-----|
| Feed posts (API data) | React Query in-memory cache | 2-5 min (staleTime); refetch on focus |
| User avatars (images) | Browser HTTP cache + CDN | 24h+ (immutable with hash) |
| Post images / videos | Browser HTTP cache + CDN | 1 year (immutable with hash) |
| Comments per post | React Query cache (keyed by postId) | 2 min |
| Scroll position | sessionStorage | 30 min |
| Draft post | localStorage | Until posted or manually cleared |
| Offline feed (PWA) | IndexedDB (via Service Worker) | Until next online sync |

### 10.3 Cache Invalidation

*   **Feed data**: Invalidated on window focus (`refetchOnWindowFocus`); on pull-to-refresh; on SSE `new_post` event for the banner.
*   **Post engagement counts**: Updated optimistically on like/comment; background-reconciled on next feed fetch.
*   **Comments**: Invalidated when user adds a comment; also refetched when user expands comment section.
*   **Images / videos**: Never invalidated (URLs contain content hash — new version = new URL).
*   **Offline cache**: Sync on `navigator.onLine` event; compare timestamps with server.

---

## 11. CDN and Asset Optimization

*   **Media delivery**: All post images and videos served via CDN with global edge caching.
*   **Image optimization**:
    *   Serve WebP/AVIF with fallback to JPEG via `<picture>` or `Accept` header negotiation.
    *   Responsive sizes via `srcset`: `480w` for mobile, `1080w` for desktop, `150w` for thumbnails.
    *   Aspect ratio hints from API (`width`, `height`) to prevent CLS.
    *   LQIP (Low Quality Image Placeholder): API includes a `blurhash` or tiny base64 thumbnail for blur-up effect.
*   **Video optimization**:
    *   HLS adaptive bitrate streaming for longer videos (240p → 1080p based on bandwidth).
    *   Short videos (< 30s) served as MP4 with `preload="none"` (load on visibility).
    *   Video thumbnails (`poster` attribute) generated server-side.
*   **Avatar images**: Small (64-128px), heavily cached, served at fixed size via CDN image resizing.
*   **Cache headers**:
    *   Static assets (JS, CSS, images): `Cache-Control: public, max-age=31536000, immutable` (1 year, content-hashed filenames).
    *   API responses: `Cache-Control: private, no-cache` (always validate, but allow 304).
*   **Compression**: Brotli (preferred) / Gzip for all text responses and static assets.

---

## 12. Rendering Strategy

*   **Feed Page Shell**: **SSR** for the page layout, header, CreatePostBox, and first batch of posts (10). This gives:
    *   Fast FCP — the user sees content immediately.
    *   SEO — shared post URLs can be crawled with OG metadata.
    *   Social preview — when a feed link is shared, crawlers get rendered HTML.
*   **Subsequent feed pages**: **CSR** — fetched client-side via infinite scroll. No SSR benefit for content the user must scroll to.
*   **Post interactions (like, comment, share)**: Fully **CSR** — interactive UI with optimistic updates.
*   **Post creation modal**: **Lazy-loaded CSR chunk** — `React.lazy(() => import('./CreatePostModal'))` with Suspense fallback. Only loaded when user taps "Create Post."
*   **Media rendering**:
    *   Images: Native `<img>` with `loading="lazy"` + blur-up placeholder.
    *   Videos: Native `<video>` with `poster`, `muted`, `playsInline`; HLS via `hls.js` if adaptive streaming is needed.
*   **Hydration**: After SSR, the client hydrates the page and attaches event listeners. Use **selective hydration** (React 18+) — hydrate interactive elements (like buttons, comment inputs) first, defer non-interactive content.

---

## 13. Cross Cutting Non Functional Concerns

### 13.1 Security

*   **XSS mitigation**: All user-generated content (post text, comments) is sanitized before rendering. Use a library like DOMPurify for any HTML content. Escape mentions, hashtags, and links.
*   **CSRF**: All state-changing requests (POST, PUT, DELETE) include a CSRF token via header or cookie.
*   **Content Security Policy (CSP)**: Restrict `img-src` and `media-src` to known CDN origins. Restrict `script-src` to self + trusted domains.
*   **Token storage**: Auth tokens stored in HTTP-only cookies (not localStorage) to prevent XSS theft.
*   **Media URL security**: CDN URLs for private content use signed URLs with short expiry (1h). Public content uses unsigned URLs.
*   **Rate limiting**: Frontend implements client-side throttling on like/comment actions to prevent accidental spam. Backend enforces authoritative rate limits.

---

### 13.2 Accessibility

*   **Semantic HTML**:
    *   Feed list: `<main>` containing `<article>` elements for each post.
    *   Post header: `<header>` with author name and timestamp.
    *   Post actions: `<button>` elements with descriptive `aria-label` (e.g., `aria-label="Like this post by Zeeshan. 342 likes"`).
    *   Comments section: `<section aria-label="Comments">` with `<ul>` for comment list.
*   **Keyboard navigation**:
    *   `Tab` moves focus between posts and interactive elements.
    *   `Enter` / `Space` activates buttons (like, comment, share).
    *   Skip links: "Skip to main content" link at the top of the page.
*   **Screen reader support**:
    *   New posts banner: `aria-live="polite"` so screen readers announce "5 new posts available."
    *   Like button: Toggle state communicated via `aria-pressed="true/false"`.
    *   Engagement counts: `aria-label` provides full context ("342 likes, 28 comments, 15 shares").
    *   Loading states: `aria-busy="true"` on the feed during loading.
*   **Infinite scroll a11y**:
    *   Sentinel element has `aria-hidden="true"` (it's purely visual/mechanical).
    *   When new posts load, announce via an `aria-live` region: "10 more posts loaded."
    *   Provide an alternative paginated view for users who prefer explicit navigation.
*   **Media**:
    *   All post images have descriptive `alt` text (from author or AI-generated).
    *   Videos have captions via `<track>` element when available.
    *   Videos respect `prefers-reduced-motion` — auto-play disabled; show static thumbnail instead.
*   **High contrast**: Ensure like buttons, text, and timestamps meet WCAG AA contrast ratios (4.5:1 for text, 3:1 for large text/icons).

---

### 13.3 Performance Optimization

#### Media Preloading Strategy

```
Current viewport:
  [Post 1 ✅ loaded] [Post 2 ✅ loaded] [Post 3 ✅ loaded]

Below viewport (preloading):
  [Post 4 🔄 image loading] [Post 5 🔄 image loading]

Far below:
  [Post 6 ⏳ not loaded] [Post 7 ⏳ not loaded]
```

*   **Above the fold (first 3 posts)**: Images loaded eagerly (`loading="eager"`); critical for LCP.
*   **Below the fold**: Browser-managed lazy loading (`loading="lazy"`); `rootMargin` on IntersectionObserver-based loaders ensures images start loading ~500px before becoming visible.
*   **Videos**: `preload="none"` until visible; then `preload="metadata"` (loads first frame + duration).

#### Other Optimizations

*   **Code splitting**: Feed page is the main chunk. Post creation modal, share modal, and report dialog are lazy-loaded chunks (50-80KB each, loaded on demand).
*   **Bundle optimization**: Tree-shake unused utilities; split vendor chunk (React, React Query) from app code.
*   **Debounced and batched API calls**:
    *   If user rapidly likes/unlikes, debounce the API call (send final state after 500ms).
    *   Engagement count updates from SSE are batched — update UI at most once per second.
*   **`requestAnimationFrame` for scroll-linked UI**: Any UI updates driven by scroll position (e.g., "back to top" button visibility) use `rAF` to batch to the next frame.
*   **Memoization**: `React.memo` on `PostCard` to prevent re-renders when sibling posts change. `useMemo` for derived values (formatted counts, relative timestamps).
*   **Abort stale fetches**: When user navigates away from the feed, cancel in-flight API requests via `AbortController`.
*   **Image decoding**: Use `img.decode()` for above-the-fold images to ensure they're decoded before paint, avoiding a flash of blank space.

---

### 13.4 Observability and Reliability

*   **Error Boundaries**: Wrap each `PostCard` in an error boundary — if one post's rendering crashes (e.g., bad media URL format), the rest of the feed remains functional. Show a "Something went wrong with this post" fallback.
*   **Logging and Analytics**:
    *   **Impressions**: Track which posts enter the viewport (using IntersectionObserver) for feed ranking feedback.
    *   **Engagement**: Track like, comment, share events with post IDs and user context.
    *   **Scroll depth**: How far the user scrolled (number of posts viewed in a session).
    *   **Time on post**: How long a post was in the viewport (signals content quality).
    *   **Media load failures**: Log URL, error type, and network conditions for debugging CDN issues.
*   **Performance monitoring**:
    *   Track FCP, LCP, CLS, and TTI for the feed page.
    *   Track infinite scroll latency: time from sentinel intersection to new posts rendered.
    *   Track SSE connection health: connection drops, reconnect frequency, message latency.
*   **Feature flags**: Gate new features behind flags:
    *   New post card layout, different content types, experimental ranking signals.
    *   Gradual rollout (1% → 10% → 100%) with monitoring at each stage.
*   **Graceful degradation**:
    *   If SSE fails → fall back to polling every 60s.
    *   If React Query cache is corrupted → clear cache and refetch.
    *   If a specific media type fails → show placeholder, don't crash the post.

---

## 14. Edge Cases and Tradeoffs

| Edge Case | Handling |
|-----------|---------|
| **Post deleted while user is reading** | If the next feed fetch doesn't include it, remove it with a fade-out animation. If the user tries to interact, show "This post has been deleted." |
| **User posts while offline** | Queue the post in localStorage. On reconnect, submit with a "Posted X minutes ago" timestamp. Show "Pending" indicator. |
| **Feed is empty (new user)** | Show an onboarding UI: suggested accounts to follow, trending posts, or a "Your feed is empty — follow people to see their posts" message. |
| **Extremely long post text** | Truncate at ~300 characters with a "See More" button. Full text renders on expansion (no re-fetch needed — full text is in the payload). |
| **Image carousel with 10+ images** | Virtualize the carousel (only render visible + 1 buffer image). Show "1 of 10" indicator. |
| **Video with no sound** | Show a "No audio" indicator. Don't show volume controls. |
| **Rapid like/unlike (double-tap)** | Debounce the API call. Only send the final state after 500ms of no changes. UI responds instantly to each tap. |
| **SSE reconnect storm** | After SSE reconnects, server may send all missed posts at once. Buffer and batch them into the banner. |
| **User follows 5000+ accounts** | Feed API handles ranking server-side; frontend just paginates. The cursor-based approach handles any volume. |
| **Mixed content post (text + image + link)** | Render text, then image, then link preview. If both image and link have thumbnails, prefer the uploaded image. |
| **Concurrent tab sessions** | Like in tab A should reflect in tab B on next focus. React Query's `refetchOnWindowFocus` handles this. |

### Key Tradeoffs

| Decision | Tradeoff |
|----------|----------|
| **SSR first batch of posts** | Better FCP and SEO, but adds server load and time-to-first-byte. CSR-only would be simpler but slower perceived load. |
| **Cursor over offset pagination** | No duplicates/missing items, but can't show "Page 3 of 50" or jump to a page. Acceptable for feeds (users never want "page 47"). |
| **SSE over WebSocket** | Simpler, auto-reconnects, proxy-friendly. But only server → client. If we needed bidirectional real-time (like chat within feed), WebSocket would be needed. |
| **Virtualization** | Constant DOM size regardless of scroll depth. But adds complexity, makes scroll position restoration harder, and requires dynamic height measurement. |
| **Optimistic updates** | Instant UI responsiveness. But introduces rollback complexity and potential for brief inconsistency between client and server state. |
| **Buffer new posts (banner)** | Better UX when scrolling mid-feed. But delays content freshness — user must tap to see new posts. |
| **`content-visibility: auto` vs Virtualization** | `content-visibility` is simpler (CSS-only) but less precise, browser-dependent, and can cause scrollbar jumps. Virtualization is more reliable but adds JS overhead. |
| **Denormalized post data** | Each post is self-contained (simple reads). But if a user changes their avatar, all their cached posts show the old avatar until refetch. |

---

## 15. Summary and Future Improvements

### Key Architectural Decisions

1.  **Cursor-based infinite scroll with IntersectionObserver** — seamless pagination without duplicates; sentinel pattern avoids scroll event overhead.
2.  **SSE for real-time new posts** — simpler than WebSocket; auto-reconnects; new posts buffered with a "tap to see" banner.
3.  **Optimistic updates for all interactions** — likes, comments, and shares feel instant; rollback on failure.
4.  **Virtualized feed list** — constant DOM footprint for infinite scroll sessions; dynamic height measurement for variable post sizes.
5.  **SSR for initial feed** — fast FCP with server-rendered first batch; CSR for all subsequent interactions.
6.  **React Query for server state** — automatic caching, stale-while-revalidate, background refetch, and infinite query support.

### Possible Future Enhancements

*   **Offline feed viewing**: Cache the last 50 posts in IndexedDB via Service Worker. Queue interactions for replay on reconnect.
*   **Shared Element Transitions**: Animate post card expansion into full detail view using the View Transitions API.
*   **Skeleton loading with content-aware shapes**: Instead of generic grey boxes, show skeletons that match the expected content type (text-only vs image post).
*   **Web Workers for feed processing**: Offload feed data normalization, deduplication, and engagements merging to a Web Worker to keep the main thread free for rendering.
*   **Predictive prefetching**: Use heuristics (scroll velocity, time of day) to prefetch the next 2-3 pages before the user reaches the sentinel.
*   **Client-side feed re-ranking**: If the user has "muted words" or custom filters, apply them client-side to the feed data without requiring a server round-trip.
*   **Collaborative real-time features**: Show "X is typing a comment..." indicators using WebSocket for posts the user is actively viewing.

---

### Endpoint Summary

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/feed` | GET | Fetch paginated feed (cursor-based) |
| `/api/feed/stream` | GET (SSE) | Stream new posts and engagement updates |
| `/api/post` | POST | Create new post |
| `/api/post/:id` | GET | Fetch single post with full details |
| `/api/post/:id/like` | POST | Like or unlike a post |
| `/api/post/:id/comments` | GET | Fetch paginated comments for a post |
| `/api/post/:id/comment` | POST | Add a comment |
| `/api/post/:id/share` | POST | Share or repost a post |

---

### Complete Feed Flow

| Direction | Mechanism | Trigger | Endpoint | Action |
|-----------|-----------|---------|----------|--------|
| Initial Load | REST (SSR + CSR) | On mount | `GET /api/feed?limit=10` | Render first 10 posts |
| Infinite Scroll (Older Posts) | REST | Sentinel intersects viewport | `GET /api/feed?cursor=<token>` | Append next batch of posts |
| New Post Updates | SSE | Server push | `GET /api/feed/stream` | Buffer new posts; show banner |
| Like Post | REST + Optimistic | User taps like | `POST /api/post/:id/like` | Instant UI update; background API call |
| Add Comment | REST + Optimistic | User submits comment | `POST /api/post/:id/comment` | Show pending comment; confirm on API success |
| Share Post | REST | User taps share | `POST /api/post/:id/share` | Open share modal; submit on confirmation |
| Create Post | REST + Optimistic | User publishes | `POST /api/post` | Show post at top of own feed; server distributes |
| Refresh Feed | REST | Pull-to-refresh or window focus | `GET /api/feed?limit=10` | Replace stale data with fresh feed |

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design
