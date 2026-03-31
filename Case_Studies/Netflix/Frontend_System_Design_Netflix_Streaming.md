# Frontend System Design: Netflix (Streaming Platform)

- [Frontend System Design: Netflix (Streaming Platform)](#frontend-system-design-netflix-streaming-platform)
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
    - [5.2 Browse Page Row Carousel and Content Discovery (Deep Dive)](#52-browse-page-row-carousel-and-content-discovery-deep-dive)
      - [5.2.1 Horizontal Carousel Row Architecture](#521-horizontal-carousel-row-architecture)
      - [5.2.2 Virtualized Row Rendering for Dozens of Carousels](#522-virtualized-row-rendering-for-dozens-of-carousels)
      - [5.2.3 Title Card Hover Expansion and Preview](#523-title-card-hover-expansion-and-preview)
      - [5.2.4 Billboard Hero and Autoplay Trailer](#524-billboard-hero-and-autoplay-trailer)
      - [5.2.5 Prefetching and Preloading for Instant Playback](#525-prefetching-and-preloading-for-instant-playback)
      - [5.2.6 Responsive Layout Across Devices](#526-responsive-layout-across-devices)
      - [5.2.7 Keyboard and Remote Control Navigation](#527-keyboard-and-remote-control-navigation)
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
    - [9.2 Phased Row Loading Strategy (Deep Dive)](#92-phased-row-loading-strategy-deep-dive)
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

*   A subscription-based streaming platform that allows users to browse, discover, and watch movies and TV shows — similar to Netflix.
*   Target users: subscribers who consume long-form video content across devices.
*   Primary use case: browsing a personalized catalog of content rows, watching high-quality video with adaptive streaming, and managing profiles and watch progress.

---

### 1.2 Key User Personas

*   **Casual Viewer**: Browses the home page, picks something quickly from "Continue Watching" or billboard hero, watches and moves on.
*   **Binge Watcher**: Watches multi-episode TV series back-to-back; relies on auto-play next episode, skip intro, and resume where they left off.
*   **Browser / Discoverer**: Spends time exploring genres, reading synopses, watching trailers, adding titles to "My List" before committing to watch.
*   **Family Account User**: Switches between profiles (kids, adults); expects profile-specific recommendations and viewing restrictions.

---

### 1.3 Core User Flows (High Level)

*   **Browsing and Selecting Content (Primary Flow)**:
    1.  User signs in → selects a profile → browse page loads with personalized rows.
    2.  Billboard hero autoplays a trailer for a featured title.
    3.  User scrolls through horizontal content rows (Trending, Continue Watching, Top Picks, etc.).
    4.  User hovers on a title card → card expands with synopsis, match score, and mini preview.
    5.  User clicks "Play" → transitions to the full-screen video player.

*   **Watching Content (Primary Flow)**:
    1.  Video player initializes → fetches adaptive streaming manifest (DASH with DRM).
    2.  Video starts playing at the best quality for current bandwidth.
    3.  For TV shows: "Skip Intro" button appears during opening credits → auto-advances to next episode on completion.
    4.  User can seek, adjust audio/subtitle tracks, change quality, enter/exit fullscreen.
    5.  Watch progress is saved continuously; user can resume from any device.

*   **Managing Profiles and Preferences (Secondary Flow)**:
    1.  User selects "Who's Watching?" → picks or creates a profile.
    2.  Each profile has its own watch history, My List, and recommendations.
    3.  User can rate titles (thumbs up/down), manage My List, and set language/subtitle preferences.

---

## 2. Requirements

### 2.1 Functional Requirements

*   **Browse Page**:
    *   Display a personalized billboard hero with autoplay trailer at the top.
    *   Show horizontally scrollable content rows — each row is a genre or algorithmic category (Trending Now, Because You Watched X, New Releases, My List, Continue Watching).
    *   Title cards show poster artwork; on hover, expand with synopsis, match percentage, maturity rating, and genre tags.
    *   Hover preview plays a short muted clip after a brief delay.
*   **Video Playback**:
    *   Adaptive bitrate streaming (DASH/HLS) with DRM (Widevine/FairPlay/PlayReady).
    *   Custom player controls: play/pause, seek bar, volume, audio track selector, subtitle/caption selector, playback speed, fullscreen.
    *   Resume playback from last watched position across devices.
    *   "Skip Intro" and "Skip Recap" buttons with timed appearance.
    *   Auto-play next episode with countdown overlay.
    *   Post-credits "Next Episode" card and recommended titles screen.
*   **Search**:
    *   Real-time search with typeahead suggestions for titles, actors, genres.
    *   Search results as a grid of matching title cards.
*   **My List**:
    *   Add/remove titles from personal watchlist.
    *   Persisted per profile.
*   **Profiles**:
    *   Profile selection screen on app launch.
    *   Kids profile with content filtering (maturity rating).
    *   Profile-specific recommendations and watch history.
*   **Title Detail Page**:
    *   For movies: synopsis, cast, rating, similar titles, trailer.
    *   For TV shows: season/episode selector, episode list with thumbnails and progress bars, synopsis per episode.

---

### 2.2 Non Functional Requirements

*   **Performance**: Time-to-first-frame < 2s; FCP < 1.5s for browse page; smooth 60fps carousel scrolling; billboard trailer starts within 3s of page load; hover preview loads within 500ms.
*   **Scalability**: Support a catalog of 10K+ titles; dozens of personalized rows per user; millions of concurrent viewers.
*   **Availability**: Graceful degradation — show cached catalog if API is slow; fallback to static artwork if trailer fails; skeleton UI during loading.
*   **Security**: DRM-protected content (Widevine L1/L3, FairPlay, PlayReady); signed CDN URLs with short-lived tokens; no client-side URL exposure of raw video streams; authenticated API calls.
*   **Accessibility**: Full keyboard and screen reader navigation for browse and player; closed captions and audio descriptions; focus management across carousels; reduced motion support; high contrast mode.
*   **Device Support**: Desktop web (primary), mobile web (responsive), smart TV web apps (10-foot UI), low-bandwidth and low-end device handling via adaptive streaming.
*   **i18n**: Multi-language UI; localized metadata (titles, synopses); RTL layout support; multi-language audio tracks and subtitle files.

---

## 3. Scope Clarification (Interview Scoping)

### 3.1 In Scope

*   Browse page with hero billboard, horizontal content row carousels, and title card hover previews.
*   Video player with adaptive streaming, DRM, skip intro, auto-play next episode, and resume playback.
*   Title detail page (movie and TV show variants).
*   Profile selection and profile-specific state.
*   State management for catalog, playback, and user preferences.
*   API design from the frontend perspective.
*   Performance optimization for media-heavy browse experience.

---

### 3.2 Out of Scope

*   Backend recommendation algorithms and content ingestion pipeline.
*   Video transcoding and encoding (backend concern).
*   Payment/subscription management and billing.
*   Content upload tooling (admin/internal).
*   Native mobile and TV apps (React Native, native SDKs).
*   Download for offline viewing.
*   Social features (sharing, watch party — brief mention in future improvements).
*   Ads tier implementation.

---

### 3.3 Assumptions

*   User is authenticated and subscribed; auth token is available via HTTP-only cookie.
*   Content catalog is pre-indexed by the backend; APIs return pre-ranked, personalized rows.
*   Videos are pre-transcoded into multiple quality levels and available as DASH manifests with DRM.
*   Artwork (posters, logos, backgrounds) and trailers are served from a CDN.
*   Subtitles and alternative audio tracks are available as TTML/WebVTT files alongside the video manifest.
*   Each profile has its own API context (profile ID in request headers or tokens).

---

## 4. High Level Frontend Architecture

### 4.1 Overall Approach

*   **SPA** (Single Page Application) with client-side routing for seamless transitions between browse, title detail, and player.
*   **SSR** for the browse page shell — server renders the billboard and first 3-4 rows for fast FCP and SEO (canonical title URLs for search engines).
*   **CSR** for all subsequent interactions — carousel scrolling, hover previews, player, title detail, and profile management are client-side.
*   The video player module is **code-split** and loaded only when playback is initiated.
*   Browse page entry is the **primary chunk**; title detail modal, search overlay, and player are separate chunks lazy-loaded on demand.

---

### 4.2 Major Architectural Layers

```
┌────────────────────────────────────────────────────────────────────────┐
│  UI Layer                                                              │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ BrowsePage                                                     │    │
│  │  ┌──────────────────┐  ┌──────────────────────────────────┐    │    │
│  │  │ BillboardHero    │  │ ContentRowCarousel[]             │    │    │
│  │  │ (autoplay trailer│  │  ┌──────────┐ ┌──────────┐       │    │    │
│  │  │   + CTA buttons) │  │  │TitleCard │ │TitleCard │ ...   │    │    │
│  │  └──────────────────┘  │  └──────────┘ └──────────┘       │    │    │
│  │                        └──────────────────────────────────┘    │    │
│  ├────────────────────────────────────────────────────────────────┤    │
│  │ VideoPlayer (full-screen overlay / dedicated route)            │    │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐    │    │
│  │  │ ControlsBar  │ │ Subtitle     │ │ SkipIntro / NextEp   │    │    │
│  │  │ ProgressBar  │ │ Renderer     │ │ Countdown Overlay    │    │    │
│  │  │ AudioMenu    │ │              │ │                      │    │    │
│  │  └──────────────┘ └──────────────┘ └──────────────────────┘    │    │
│  ├────────────────────────────────────────────────────────────────┤    │
│  │ TitleDetailModal / Page                                        │    │
│  │  (Synopsis, Cast, Episodes, Similar Titles)                    │    │
│  └────────────────────────────────────────────────────────────────┘    │
├────────────────────────────────────────────────────────────────────────┤
│  State Management Layer                                                │
│  (Profile State, Browse Catalog, Player State, Watch History,          │
│   My List, UI Preferences, Playback Position)                          │
├────────────────────────────────────────────────────────────────────────┤
│  API and Data Access Layer                                             │
│  (REST/Falcor/GraphQL Client, Streaming Manifest Fetcher,              │
│   DRM License Manager, Request Deduplication, Retry Logic)             │
├────────────────────────────────────────────────────────────────────────┤
│  Shared / Utility Layer                                                │
│  (IntersectionObserver helpers, Debounce/Throttle, Date/Number format, │
│   Media Session API, ABR Controller, DRM Wrapper, Analytics tracker)   │
└────────────────────────────────────────────────────────────────────────┘
```

---

### 4.3 External Integrations

*   **CDN**: Serves video segments (DASH .m4s / HLS .ts), artwork (posters, backgrounds, logos), trailers, and static assets (Cloudfront, Akamai, Netflix Open Connect).
*   **DRM License Server**: Widevine (Chrome, Firefox, Android), FairPlay (Safari, iOS), PlayReady (Edge) — license keys fetched per-session via Encrypted Media Extensions (EME).
*   **Analytics SDK**: Track impressions (title viewed in row), play events, watch time, buffering events, quality switches, search queries, browse patterns, A/B test exposure.
*   **Backend Services**: Browse API (personalized rows), title metadata API, search API, watch history API, My List API, profile API.
*   **Media Player Libraries**: shaka-player (DASH + DRM in one package) or dash.js + custom DRM wrapper.
*   **Subtitle Services**: TTML / WebVTT subtitle and caption files served from CDN alongside the manifest.

---

## 5. Component Design and Modularization

### 5.1 Component Hierarchy

```
App
 ├── ProfileGate (Who's Watching? screen)
 │    └── ProfileAvatar[] (click to select profile)
 │
 ├── Header
 │    ├── Logo (link to home)
 │    ├── NavLinks (Home, TV Shows, Movies, New & Popular, My List)
 │    ├── SearchToggle → SearchOverlay
 │    │    ├── SearchInput (debounced typeahead)
 │    │    └── SearchResultsGrid
 │    ├── NotificationBell
 │    └── ProfileDropdown (switch profile, account, sign out)
 │
 ├── BrowsePage
 │    ├── BillboardHero
 │    │    ├── BackgroundArtwork / TrailerPlayer (muted autoplay)
 │    │    ├── TitleLogo (Netflix-style text logo of the title)
 │    │    ├── Synopsis (2-line truncated)
 │    │    ├── PlayButton (→ VideoPlayer)
 │    │    └── MoreInfoButton (→ TitleDetailModal)
 │    │
 │    └── ContentRowList (vertical list of horizontal rows)
 │         └── ContentRow[] (one per genre/category)
 │              ├── RowTitle ("Trending Now", "Continue Watching", etc.)
 │              ├── CarouselContainer (horizontal scrollable)
 │              │    └── TitleCard[] (per title in the row)
 │              │         ├── PosterImage (portrait or landscape)
 │              │         ├── HoverPreviewCard (expanded on hover)
 │              │         │    ├── PreviewVideo (muted short clip)
 │              │         │    ├── ActionButtons (Play, Add to List, Like, Expand)
 │              │         │    ├── MatchScore + MaturityRating + Duration
 │              │         │    └── GenreTags
 │              │         └── ProgressBar (for Continue Watching row)
 │              └── PaginationArrows (left/right nav for desktop)
 │
 ├── TitleDetailModal (overlay on browse) or TitleDetailPage
 │    ├── HeroSection (trailer autoplay + title logo + synopsis)
 │    ├── MetadataBar (year, maturity, duration/seasons, audio)
 │    ├── ActionButtons (Play, Add to List, Rate)
 │    ├── EpisodeSelector (TV shows only)
 │    │    ├── SeasonDropdown
 │    │    └── EpisodeList
 │    │         └── EpisodeCard[]
 │    │              ├── EpisodeThumbnail + ProgressBar
 │    │              ├── EpisodeTitle + Duration
 │    │              └── EpisodeSynopsis
 │    ├── SimilarTitles (row of title cards)
 │    └── AboutSection (cast, genres, maturity details)
 │
 └── VideoPlayer (full-screen, rendered as overlay or dedicated route)
      ├── VideoElement (HTML5 video with DASH/HLS + DRM)
      ├── ControlsOverlay
      │    ├── BackButton (← back to browse)
      │    ├── ProgressBar (seekable with episode timeline markers)
      │    ├── PlayPauseButton
      │    ├── VolumeControl
      │    ├── AudioTrackSelector (English, Spanish, Hindi...)
      │    ├── SubtitleSelector (Off, English, French...)
      │    ├── PlaybackSpeedMenu
      │    ├── FullscreenButton
      │    └── EpisodeTitle + SeriesName
      ├── SkipIntroButton (timed appearance)
      ├── SkipRecapButton (timed appearance)
      ├── NextEpisodeCountdown (end-of-episode overlay)
      │    ├── NextEpisodeCard (thumbnail + title)
      │    ├── CountdownTimer (15s auto-advance)
      │    └── CancelButton / PlayNowButton
      ├── SubtitleRenderer (TTML/WebVTT overlay)
      ├── BufferingSpinner
      └── PostPlayScreen (shown after final episode or movie end)
           └── RecommendedTitlesGrid
```

---

### 5.2 Browse Page Row Carousel and Content Discovery (Deep Dive)

The browse page is the **heart of the Netflix experience** — a vertically scrolling page filled with horizontally scrolling content rows. Getting this right matters because:
*   The browse page is the **first thing users see** after profile selection — any jank or slow render directly impacts perceived quality.
*   There are typically **40-75 content rows** per browse page, each containing 10-75 titles. Rendering everything up front would mean thousands of DOM nodes.
*   Each title card triggers **complex hover interactions** — expanded card with preview video, synopsis, action buttons — that must feel instant.
*   The page must work across desktop (mouse hover), tablet (touch), and TV (remote/D-pad) input methods.

---

#### 5.2.1 Horizontal Carousel Row Architecture

##### The Core Pattern — CSS Scroll Container with Paginated Navigation

Instead of building a complex JavaScript carousel engine, Netflix uses a **CSS-based horizontal scroll container** with paginated left/right navigation controlled by JavaScript. Each "page" of the carousel shows exactly enough items to fill the viewport width.

**Why paginated (page-by-page navigation) instead of free scroll?**

| Approach | How it works | UX |
|----------|-------------|-----|
| **Free scroll** | User drags/swipes freely; items stop at any position | Familiar on mobile; feels imprecise on desktop; doesn't guarantee items align to edges |
| **Paginated (Netflix approach)** | Left/right arrows advance by one "page" (viewport width of items); scroll-snap keeps items aligned | Clean, controlled navigation; items always align perfectly to viewport edges; works well with remote/D-pad |

```css
.carousel-container {
  display: flex;
  flex-wrap: nowrap;
  overflow-x: hidden;           /* NOT auto — we control scroll via JS, not native scroll */
  gap: 4px;                     /* minimal gap between cards */
  transition: transform 0.75s cubic-bezier(0.5, 0, 0.1, 1);  /* Netflix's signature smooth slide */
}

.carousel-container.touch-device {
  overflow-x: auto;             /* Enable native touch scroll on mobile */
  scroll-snap-type: x mandatory;
  -webkit-overflow-scrolling: touch;
}

.title-card {
  flex: 0 0 calc((100% - 20px) / 6);   /* 6 cards per row on desktop, adjust per breakpoint */
  scroll-snap-align: start;
  position: relative;
}
```

```tsx
function CarouselContainer({ items, rowIndex }: Props) {
  const [currentPage, setCurrentPage] = useState(0);
  const itemsPerPage = useItemsPerPage(); // responsive: 6 on desktop, 3 on tablet, 2 on mobile
  const totalPages = Math.ceil(items.length / itemsPerPage);

  const translateX = -(currentPage * 100); // percentage-based for responsiveness

  const goNext = () => {
    setCurrentPage((prev) => Math.min(prev + 1, totalPages - 1));
  };

  const goPrev = () => {
    setCurrentPage((prev) => Math.max(prev - 1, 0));
  };

  return (
    <div className="carousel-wrapper" role="group" aria-label={`Content row ${rowIndex + 1}`}>
      {currentPage > 0 && (
        <button className="carousel-arrow carousel-arrow-left" onClick={goPrev} aria-label="Previous page">
          ‹
        </button>
      )}

      <div
        className="carousel-container"
        style={{ transform: `translateX(${translateX}%)` }}
      >
        {items.map((item) => (
          <TitleCard key={item.id} title={item} />
        ))}
      </div>

      {currentPage < totalPages - 1 && (
        <button className="carousel-arrow carousel-arrow-right" onClick={goNext} aria-label="Next page">
          ›
        </button>
      )}
    </div>
  );
}
```

##### Why `overflow: hidden` + `transform` Instead of `overflow: auto` + `scrollLeft`?

| Approach | Behavior | Why Netflix chose transform |
|----------|----------|-----------------------------|
| `overflow: auto` + JS `scrollLeft` | Native scroll; scrollbar visible (unless hidden); fires scroll events | Scrollbar appearance is inconsistent across browsers; harder to control animation curve; scroll events are noisy |
| `overflow: hidden` + `transform: translateX()` | No native scroll; movement is a CSS transform animation; arrow buttons control page | **GPU-accelerated** (composited layer); perfectly controlled timing via `cubic-bezier`; no scrollbar to hide; works identically everywhere |

The `cubic-bezier(0.5, 0, 0.1, 1)` easing is Netflix's signature — a fast start, gradual deceleration that feels weighty and premium.

---

#### 5.2.2 Virtualized Row Rendering for Dozens of Carousels

##### The Problem — 50+ Rows with 20+ Items Each

A typical Netflix browse page has 40-75 rows. Each row contains 10-75 title cards. If we render everything:
*   **75 rows × 20 cards × ~5 DOM nodes per card = 7,500+ DOM nodes** — just for title cards.
*   Add row titles, carousel arrows, and hover preview containers → easily **10,000+ DOM nodes**.
*   Each poster image is ~60KB. Loading all 1,500 simultaneously would mean **~90MB of image data**.
*   The user can only see **3-4 rows at a time** on screen. The other 70+ rows are invisible.

##### Two-Level Virtualization Strategy

Netflix uses **two levels** of virtualization:

**Level 1 — Vertical Row Virtualization (which rows are in the DOM)**:
Only the rows near the viewport (visible + 2 above + 2 below) are rendered. All other rows are replaced with empty spacers of the correct height.

**Level 2 — Horizontal Item Windowing (which cards are in a row)**:
Within each rendered row, only the visible page of cards plus one page on each side are in the DOM. Cards far off-screen in a long row are not rendered.

```
┌── Viewport (~1080px tall, shows ~3.5 rows) ─────────────────────┐
│                                                                   │
│  [Billboard Hero]                                                │
│                                                                   │
│  Row 0: "Continue Watching"  [Card][Card][Card][Card][Card][Card]│ ← rendered
│  Row 1: "Trending Now"       [Card][Card][Card][Card][Card][Card]│ ← rendered
│  Row 2: "Top 10 in US"       [Card][Card][Card][Card][Card][Card]│ ← rendered
│  Row 3: "New Releases"       [Card][Card][Card][Card][Card][Card]│ ← rendered (overscan)
│                                                                   │
├───────────────────────────────────────────────────────────────────┤
│  Row 4-6:                    ← rendered (overscan below)         │
│  Row 7-74:                   ← NOT in DOM, just spacer divs     │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

##### Implementation with IntersectionObserver

```tsx
function BrowseRowList({ rows }: { rows: ContentRow[] }) {
  return (
    <div className="browse-rows">
      {rows.map((row, index) => (
        <LazyRow key={row.id} row={row} index={index} />
      ))}
    </div>
  );
}

function LazyRow({ row, index }: { row: ContentRow; index: number }) {
  const [isVisible, setIsVisible] = useState(index < 4); // first 4 rows render immediately
  const rowRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (index < 4) return; // already visible
    const el = rowRef.current;
    if (!el) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true);
          observer.disconnect(); // once visible, always keep it rendered
        }
      },
      { rootMargin: '400px 0px' } // start rendering 400px before it scrolls into view
    );

    observer.observe(el);
    return () => observer.disconnect();
  }, [index]);

  return (
    <div ref={rowRef} className="browse-row-wrapper" style={{ minHeight: 200 }}>
      {isVisible ? (
        <ContentRowCarousel row={row} />
      ) : (
        <div className="row-placeholder" style={{ height: 200 }} aria-hidden="true" />
      )}
    </div>
  );
}
```

##### Performance Impact

| Metric | No virtualization (75 rows) | Virtualized (overscan=2) |
|--------|----------------------------|--------------------------|
| DOM nodes on initial load | ~10,000+ | ~1,500 (billboard + first 6 rows) |
| Images loaded on initial paint | ~1,500 posters | ~36 posters (6 visible per row × 6 rows) |
| Initial render time | 1-3s on mid-range devices | < 300ms |
| Memory usage | Very high (all poster images decoded) | Low (only visible images) |
| Scroll jank | Likely, especially on low-end | Consistent 60fps |

---

#### 5.2.3 Title Card Hover Expansion and Preview

This is one of Netflix's most distinctive UI patterns — when a user hovers over a title card, it **expands** to show more information and plays a short muted preview clip.

##### The Hover Interaction Flow

```
Step 1: Mouse enters title card
  ↓ (delay 500ms — prevents accidental triggers)
Step 2: Card expands (scale transform) above neighboring cards
  ↓
Step 3: After 1-2s delay, muted preview video starts
  ↓
Step 4: Action buttons + metadata appear below preview
  ↓ (on mouse leave)
Step 5: Preview video stops; card shrinks back to original size
```

##### Why a 500ms Delay Before Expansion?

Without the delay, casually moving the mouse across the row would trigger rapid expand/collapse animations for every card the mouse passes over. The delay ensures the user has **intentionally paused** on a card before the preview triggers.

```tsx
function TitleCard({ title }: { title: Title }) {
  const [isHovered, setIsHovered] = useState(false);
  const [isExpanded, setIsExpanded] = useState(false);
  const hoverTimerRef = useRef<ReturnType<typeof setTimeout>>();
  const cardRef = useRef<HTMLDivElement>(null);

  const handleMouseEnter = () => {
    setIsHovered(true);
    hoverTimerRef.current = setTimeout(() => {
      setIsExpanded(true);
    }, 500); // 500ms delay before expansion
  };

  const handleMouseLeave = () => {
    setIsHovered(false);
    setIsExpanded(false);
    clearTimeout(hoverTimerRef.current);
  };

  return (
    <div
      ref={cardRef}
      className={`title-card ${isExpanded ? 'title-card--expanded' : ''}`}
      onMouseEnter={handleMouseEnter}
      onMouseLeave={handleMouseLeave}
      role="button"
      tabIndex={0}
      aria-label={`${title.name}, ${title.year}, ${title.maturityRating}`}
    >
      <img
        src={title.posterUrl}
        alt={title.name}
        loading="lazy"
        width={230}
        height={130}
      />

      {isExpanded && (
        <HoverPreviewCard
          title={title}
          onClose={() => setIsExpanded(false)}
        />
      )}
    </div>
  );
}

function HoverPreviewCard({ title, onClose }: HoverPreviewProps) {
  const videoRef = useRef<HTMLVideoElement>(null);
  const [showVideo, setShowVideo] = useState(false);

  // Delay video start by 1.5s after card expands
  useEffect(() => {
    const timer = setTimeout(() => {
      setShowVideo(true);
    }, 1500);
    return () => clearTimeout(timer);
  }, []);

  return (
    <div className="hover-preview-card">
      {/* Preview video or fallback artwork */}
      <div className="preview-media">
        {showVideo && title.previewUrl ? (
          <video
            ref={videoRef}
            src={title.previewUrl}
            autoPlay
            muted
            playsInline
            loop
          />
        ) : (
          <img src={title.backdropUrl} alt="" />
        )}
      </div>

      {/* Action buttons */}
      <div className="preview-actions">
        <button className="btn-play" aria-label={`Play ${title.name}`}>▶</button>
        <button className="btn-add-list" aria-label="Add to My List">+</button>
        <button className="btn-like" aria-label="Like">👍</button>
        <button className="btn-expand" aria-label="More info" onClick={onClose}>▼</button>
      </div>

      {/* Metadata */}
      <div className="preview-meta">
        <span className="match-score">{title.matchScore}% Match</span>
        <span className="maturity-rating">{title.maturityRating}</span>
        <span className="duration">{title.isShow ? `${title.seasonCount} Seasons` : title.formattedDuration}</span>
      </div>

      <div className="preview-genres">
        {title.genres.slice(0, 3).map((g) => (
          <span key={g} className="genre-tag">{g}</span>
        ))}
      </div>
    </div>
  );
}
```

##### The CSS Expansion Trick

The expanded card must **grow above neighboring cards** without shifting the row layout. This is achieved using `transform: scale()` and `z-index`, NOT by changing width/height (which would reflow the layout).

```css
.title-card {
  position: relative;
  transition: transform 0.3s ease, z-index 0.3s;
  transform-origin: center center;  /* expand from center */
  z-index: 1;
}

/* First item in a row: expand from left edge */
.title-card:first-child {
  transform-origin: left center;
}

/* Last visible item: expand from right edge */
.title-card:last-child {
  transform-origin: right center;
}

.title-card--expanded {
  transform: scale(1.5);           /* 1.5x the original size */
  z-index: 10;                     /* float above neighbors */
  transition-delay: 0s;            /* expand immediately once triggered */
}

.hover-preview-card {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  background: #181818;
  border-radius: 6px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.65);
}
```

**Why `transform: scale()` instead of changing dimensions?**
*   `scale()` is a **compositor-only** operation — it doesn't trigger layout or paint. The browser's GPU handles it.
*   Changing `width`/`height` would cause the entire row to reflow — all sibling cards shift, expensive and janky.
*   With scale, the card's layout position stays fixed; it just visually "grows" above neighbors.

---

#### 5.2.4 Billboard Hero and Autoplay Trailer

The billboard is the **first visual impression** of the browse page — a large hero section featuring one promoted title with autoplay trailer.

##### Billboard Behavior Flow

```
Step 1: Page loads → show static background artwork immediately (fast FCP)
  ↓
Step 2: After 1-2s, start preloading the billboard trailer video
  ↓
Step 3: Once buffered enough, crossfade from static image to muted video
  ↓
Step 4: Trailer plays for ~30s
  ↓
Step 5: Video fades back to static artwork (save bandwidth, avoid looping annoyance)
  ↓
Step 6: If user scrolls past billboard, stop and cleanup the trailer
```

```tsx
function BillboardHero({ featured }: { featured: FeaturedTitle }) {
  const videoRef = useRef<HTMLVideoElement>(null);
  const containerRef = useRef<HTMLDivElement>(null);
  const [showVideo, setShowVideo] = useState(false);
  const [isMuted, setIsMuted] = useState(true);

  // Start trailer after a delay
  useEffect(() => {
    const timer = setTimeout(() => {
      setShowVideo(true);
    }, 2000); // 2s delay after page load
    return () => clearTimeout(timer);
  }, []);

  // Stop trailer when user scrolls past billboard
  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (!entry.isIntersecting) {
          setShowVideo(false);
          videoRef.current?.pause();
        }
      },
      { threshold: 0.3 }
    );

    observer.observe(container);
    return () => observer.disconnect();
  }, []);

  return (
    <div ref={containerRef} className="billboard-hero">
      {/* Static background artwork (always rendered for fast FCP) */}
      <img
        src={featured.backdropUrl}
        alt=""
        className={`billboard-backdrop ${showVideo ? 'billboard-backdrop--hidden' : ''}`}
        loading="eager"
      />

      {/* Trailer video (lazy loaded, plays muted) */}
      {showVideo && featured.trailerUrl && (
        <video
          ref={videoRef}
          className="billboard-trailer"
          src={featured.trailerUrl}
          autoPlay
          muted={isMuted}
          playsInline
          onEnded={() => setShowVideo(false)} // fade back to artwork after trailer ends
        />
      )}

      {/* Gradient overlay for readability */}
      <div className="billboard-gradient" />

      {/* Content overlay */}
      <div className="billboard-content">
        <img src={featured.logoUrl} alt={featured.name} className="billboard-title-logo" />
        <p className="billboard-synopsis">{featured.synopsis}</p>
        <div className="billboard-actions">
          <button className="btn-play-hero">▶ Play</button>
          <button className="btn-more-info">ⓘ More Info</button>
          <button className="btn-mute-toggle" onClick={() => setIsMuted(!isMuted)}>
            {isMuted ? '🔇' : '🔊'}
          </button>
        </div>
      </div>

      {/* Maturity rating badge */}
      <div className="billboard-maturity">{featured.maturityRating}</div>
    </div>
  );
}
```

---

#### 5.2.5 Prefetching and Preloading for Instant Playback

When the user clicks "Play", the expectation is **instant** playback. To achieve time-to-first-frame < 2s, we preload critical resources before the user clicks:

| Resource | When to preload | How |
|----------|----------------|-----|
| **DASH manifest** | When user hovers on "Play" button for > 300ms | `<link rel="prefetch" href="manifest.mpd">` or `fetch()` in background |
| **DRM license** | After manifest is preloaded | Start EME license request proactively |
| **First 2-3 video segments** | After manifest parsed | hls.js / shaka-player can prefetch initial segments |
| **Audio track metadata** | With manifest | Embedded in manifest — no extra request |
| **Subtitle file** | On playback start | Fetch preferred language WebVTT; small file (~50KB) |

```tsx
function usePlaybackPrefetch(manifestUrl: string) {
  const prefetchedRef = useRef(false);

  const startPrefetch = useCallback(() => {
    if (prefetchedRef.current || !manifestUrl) return;
    prefetchedRef.current = true;

    // Prefetch manifest in background
    const link = document.createElement('link');
    link.rel = 'prefetch';
    link.href = manifestUrl;
    link.as = 'fetch';
    link.crossOrigin = 'anonymous';
    document.head.appendChild(link);
  }, [manifestUrl]);

  const handlePlayHover = useCallback(() => {
    // Start prefetch on hover intent (300ms delay to avoid accidental triggers)
    const timer = setTimeout(startPrefetch, 300);
    return () => clearTimeout(timer);
  }, [startPrefetch]);

  return { handlePlayHover };
}
```

##### Why Prefetch the DRM License Eagerly?

DRM license acquisition is a **network round-trip** to the license server (50-200ms). Without pre-fetching:
1.  User clicks Play → manifest fetched → player requests license → **200ms wait** → first segment decrypted → playback starts.

With pre-fetching:
1.  User hovers Play → manifest fetched + license acquired in background → user clicks Play → first segment **already decrypted** → instant playback.

---

#### 5.2.6 Responsive Layout Across Devices

The number of title cards per row and the card dimensions change based on viewport width:

| Breakpoint | Cards per row | Card aspect ratio | Carousel behavior |
|-----------|---------------|-------------------|-------------------|
| **Desktop (> 1400px)** | 6 | 16:9 landscape | Paginated with arrow buttons |
| **Laptop (1100-1400px)** | 5 | 16:9 landscape | Paginated with arrows |
| **Tablet (800-1100px)** | 4 | 16:9 landscape | Touch scroll + scroll-snap |
| **Mobile (500-800px)** | 3 | 16:9 landscape | Touch scroll + scroll-snap |
| **Small mobile (< 500px)** | 2 | 2:3 portrait (poster) | Touch scroll + scroll-snap |

```tsx
function useItemsPerPage(): number {
  const [count, setCount] = useState(6);

  useEffect(() => {
    const update = () => {
      const width = window.innerWidth;
      if (width < 500) setCount(2);
      else if (width < 800) setCount(3);
      else if (width < 1100) setCount(4);
      else if (width < 1400) setCount(5);
      else setCount(6);
    };

    update();
    window.addEventListener('resize', update);
    return () => window.removeEventListener('resize', update);
  }, []);

  return count;
}
```

On mobile/tablet, the carousel switches from JS-controlled pagination (`overflow: hidden` + `transform`) to native touch scrolling (`overflow-x: auto` + `scroll-snap-type: x mandatory`). This gives users the native momentum scroll they expect on touch devices.

---

#### 5.2.7 Keyboard and Remote Control Navigation

For accessibility and smart TV support, the browse page must be fully navigable via keyboard/D-pad:

| Key | Action |
|-----|--------|
| `→` / `←` | Move focus to next / previous card in the current row |
| `↑` / `↓` | Move focus to the same column position in the adjacent row |
| `Enter` | Open title detail or start playback |
| `Tab` | Move focus through interactive elements (header, rows, cards) |
| `Escape` | Close modal / exit player / collapse hover card |

##### Spatial Navigation Pattern

Traditional `Tab` order is linear — it goes left-to-right, top-to-bottom. For a 2D grid of carousels, **spatial navigation** is more natural: arrow keys move in the direction they point.

```tsx
function useSpatialNavigation(gridRef: React.RefObject<HTMLElement>) {
  useEffect(() => {
    const grid = gridRef.current;
    if (!grid) return;

    const handleKeyDown = (e: KeyboardEvent) => {
      const focused = document.activeElement as HTMLElement;
      if (!focused || !grid.contains(focused)) return;

      const row = focused.closest('.content-row');
      const card = focused.closest('.title-card');
      if (!card || !row) return;

      const cards = Array.from(row.querySelectorAll('.title-card'));
      const rows = Array.from(grid.querySelectorAll('.content-row'));
      const cardIndex = cards.indexOf(card);
      const rowIndex = rows.indexOf(row);

      switch (e.key) {
        case 'ArrowRight': {
          e.preventDefault();
          const next = cards[cardIndex + 1] as HTMLElement;
          next?.focus();
          break;
        }
        case 'ArrowLeft': {
          e.preventDefault();
          const prev = cards[cardIndex - 1] as HTMLElement;
          prev?.focus();
          break;
        }
        case 'ArrowDown': {
          e.preventDefault();
          const nextRow = rows[rowIndex + 1];
          if (nextRow) {
            const nextCards = nextRow.querySelectorAll('.title-card');
            const target = (nextCards[cardIndex] || nextCards[nextCards.length - 1]) as HTMLElement;
            target?.focus();
          }
          break;
        }
        case 'ArrowUp': {
          e.preventDefault();
          const prevRow = rows[rowIndex - 1];
          if (prevRow) {
            const prevCards = prevRow.querySelectorAll('.title-card');
            const target = (prevCards[cardIndex] || prevCards[prevCards.length - 1]) as HTMLElement;
            target?.focus();
          }
          break;
        }
      }
    };

    grid.addEventListener('keydown', handleKeyDown);
    return () => grid.removeEventListener('keydown', handleKeyDown);
  }, [gridRef]);
}
```

---

#### 5.2.8 Decision Matrix

| Decision | Options | Chosen | Rationale |
|----------|---------|--------|-----------|
| **Carousel navigation** | Free scroll vs Paginated | Paginated (`transform: translateX`) | Clean alignment; GPU-accelerated; consistent across browsers; works with D-pad |
| **Row virtualization** | Render all vs IntersectionObserver lazy | IntersectionObserver lazy rendering | 75 rows is too many; only 3-4 visible at once; saves 7000+ DOM nodes |
| **Hover expansion** | CSS `transform: scale()` vs width/height change | `transform: scale()` | Compositor-only — no layout reflow; smooth 60fps expansion |
| **Hover delay** | Immediate vs 300-500ms | 500ms | Prevents accidental triggers when mouse passes across row |
| **Preview video** | Autoplay immediately vs Delayed | Delayed (1.5s after hover expand) | Saves bandwidth; user may just be scanning; only committed hover triggers video |
| **Billboard trailer** | Autoplay on load vs After delay | After 2s delay | Let metadata render first (FCP); trailer is enhancement, not critical path |
| **Mobile carousel** | JS paginated vs Native scroll-snap | Native scroll-snap | Better touch physics; native momentum; less JS overhead |

---

### 5.3 Reusability Strategy

*   **TitleCard** component: Reused across browse rows, search results, similar titles, My List, and recommended titles in player post-play screen. Accepts `variant` prop for layout (landscape, portrait, expanded hover, compact list).
*   **CarouselContainer**: Generic horizontal paginated carousel. Reused for content rows, episode lists, and similar titles sections.
*   **VideoPlayer**: Standalone module accepting manifest URL, DRM config, and metadata. Reusable for browse trailer, title detail trailer, and full playback.
*   **ActionButtons** (Play, Add to List, Like): Consistent across hover card, title detail, and billboard.
*   **ProgressBar**: Shared between "Continue Watching" card progress bar and player seek bar (scaled-down version for cards).
*   **Design System**: Button, Modal, Dropdown, Toggle, Badge, SkeletonLoader components from a shared library.

---

### 5.4 Module Organization

```
src/
 ├── pages/
 │    ├── BrowsePage/
 │    │    ├── BrowsePage.tsx
 │    │    ├── BillboardHero.tsx
 │    │    └── ContentRowList.tsx
 │    ├── TitleDetail/
 │    │    ├── TitleDetailModal.tsx
 │    │    ├── EpisodeSelector.tsx
 │    │    ├── EpisodeCard.tsx
 │    │    └── SimilarTitles.tsx
 │    ├── SearchPage/
 │    ├── MyListPage/
 │    └── ProfileSelectPage/
 │
 ├── features/
 │    ├── player/
 │    │    ├── VideoPlayer.tsx
 │    │    ├── ControlsOverlay.tsx
 │    │    ├── ProgressBar.tsx
 │    │    ├── AudioTrackSelector.tsx
 │    │    ├── SubtitleSelector.tsx
 │    │    ├── SubtitleRenderer.tsx
 │    │    ├── SkipIntroButton.tsx
 │    │    ├── NextEpisodeCountdown.tsx
 │    │    ├── PostPlayScreen.tsx
 │    │    ├── useAdaptivePlayer.ts
 │    │    ├── usePlayerState.ts
 │    │    ├── useDRM.ts
 │    │    └── useSkipMarkers.ts
 │    ├── carousel/
 │    │    ├── CarouselContainer.tsx
 │    │    ├── TitleCard.tsx
 │    │    ├── HoverPreviewCard.tsx
 │    │    └── useItemsPerPage.ts
 │    ├── search/
 │    │    ├── SearchOverlay.tsx
 │    │    └── SearchResultsGrid.tsx
 │    └── profiles/
 │         ├── ProfileGate.tsx
 │         └── ProfileAvatar.tsx
 │
 ├── shared/
 │    ├── components/   (Button, Modal, Dropdown, Badge, SkeletonLoader)
 │    ├── hooks/        (useIntersectionObserver, useDebounce, useMediaQuery)
 │    ├── utils/        (formatDuration, formatDate, formatNumber)
 │    └── api/          (apiClient, drmManager, requestDeduplicator)
 │
 └── store/
      ├── profileStore.ts
      ├── browseStore.ts
      ├── playerStore.ts
      ├── myListStore.ts
      ├── watchHistoryStore.ts
      └── searchStore.ts
```

---

## 6. High Level Data Flow Explanation

### 6.1 Initial Load Flow

```
User selects profile on "Who's Watching?" screen
        │
        ▼
┌─── Server (SSR) ─────────────────────────────────────────────┐
│  1. Identify profile from session/token                       │
│  2. Fetch billboard data (featured title + trailer URL)       │
│  3. Fetch first 4 content rows (personalized)                 │
│  4. Render HTML shell with billboard + first rows (fast FCP)  │
│  5. Include <link rel="preconnect"> to CDN                    │
│  6. Send HTML to browser                                      │
└───────────────────────────────────────────────────────────────┘
        │
        ▼
┌─── Browser ───────────────────────────────────────────────────┐
│  1. Parse HTML → paint billboard artwork + first row posters  │
│  2. Load JS bundle → hydrate React app                        │
│  3. Billboard: after 2s delay, start loading trailer video    │
│  4. Remaining rows: loaded lazily via IntersectionObserver    │
│  5. Row data fetched as user scrolls (per row API call or     │
│     batched fetch for next 5 rows)                            │
│  6. Poster images lazy-loaded per row visibility              │
│  7. Hover preview: video fetched only on hover intent         │
│  8. Analytics: track browse session start, row impressions    │
└───────────────────────────────────────────────────────────────┘
```

---

### 6.2 User Interaction Flow

| Action | Frontend Flow |
|--------|--------------|
| **Hover on title card** | 500ms delay → expand card (`transform: scale(1.5)`) → 1.5s delay → fetch and play muted preview clip → show metadata |
| **Click "Play"** | Transition to player route/overlay → fetch DASH manifest → acquire DRM license → start adaptive playback → restore watch position from history |
| **Add to My List** | Optimistic: UI immediately shows checkmark → POST /api/mylist/add → on success: confirmed; on failure: revert + toast |
| **Skip Intro** | Button appears at `skipIntroStart` timestamp → click → `video.currentTime = skipIntroEnd` → button disappears |
| **Auto-play next episode** | End of episode → 15s countdown overlay appears with next episode thumbnail → on countdown end: load next episode manifest → seamless playback → update watch history |
| **Change audio/subtitle track** | User selects from menu → shaka-player switches active track → no rebuffering for subtitles; audio may rebuffer briefly |
| **Search** | Debounced input (300ms) → GET /api/search/suggestions → show dropdown; on submit → GET /api/search → render results grid |
| **Rate title (thumbs up/down)** | Optimistic: icon fills immediately → POST /api/ratings → confirmed or reverted |

---

### 6.3 Error and Retry Flow

| Error Type | Detection | Recovery |
|------------|-----------|----------|
| **Manifest fetch failure** | shaka-player `error` event | Retry with exponential backoff (1s, 2s, 4s); after 3 retries show "Something went wrong" screen with retry button |
| **Segment fetch failure** | Player `SEGMENT_LOAD_ERROR` | Auto-retry by player library; if persistent, drop to lower quality; if all fail, show error |
| **DRM license failure** | EME error event | Show "Playback error. Please try again." message; retry once; if persistent, suggest trying a different browser |
| **Browse API failure** | HTTP 5xx | Show cached rows if available from React Query; skeleton placeholders for uncached rows; retry in background |
| **Network offline** | `navigator.onLine` + `offline` event | Continue playback from buffer; show "No connection" toast; attempt reconnection for API calls; pause if buffer exhausts |
| **Trailer autoplay blocked** | `video.play()` promise rejection | Silently fall back to static artwork; do not show error (trailer is optional enhancement) |
| **Image load failure** | `<img>` `onerror` event | Show fallback gradient placeholder with title text overlay |

---

## 7. Data Modelling (Frontend Perspective)

### 7.1 Core Data Entities

*   **Title**: Movie or TV show — the primary content entity with metadata, artwork URLs, streaming info.
*   **Episode**: A single episode within a TV show season — its own metadata, manifest, and watch progress.
*   **ContentRow**: A horizontal row of titles on the browse page — has a display name, row type, and ordered list of title references.
*   **Profile**: A user profile within an account — with avatar, name, maturity settings, and preferences.
*   **WatchProgress**: Per-profile progress for a title or episode — positions, completion status.
*   **MyListItem**: A title saved to a profile's personal watchlist.

---

### 7.2 Data Shape

```ts
interface Title {
  id: string;
  type: 'movie' | 'show';
  name: string;
  synopsis: string;
  year: number;
  maturityRating: string;          // "PG-13", "TV-MA", "R"
  matchScore: number;              // 0-100 percent match
  genres: string[];
  cast: string[];
  posterUrl: string;               // portrait artwork
  backdropUrl: string;             // landscape background
  logoUrl: string;                 // title treatment image
  trailerUrl: string | null;       // short trailer for preview
  manifestUrl: string;             // DASH/HLS manifest for full playback
  duration: number | null;         // seconds (movies only)
  seasonCount: number | null;      // (shows only)
  seasons: Season[] | null;        // (shows only, fetched on detail)
  audioTracks: AudioTrack[];
  subtitleTracks: SubtitleTrack[];
  isInMyList: boolean;
  userRating: 'liked' | 'disliked' | null;
  watchProgress: WatchProgress | null;
}

interface Season {
  seasonNumber: number;
  episodeCount: number;
  episodes: Episode[];
}

interface Episode {
  id: string;
  titleId: string;
  seasonNumber: number;
  episodeNumber: number;
  name: string;
  synopsis: string;
  duration: number;
  thumbnailUrl: string;
  manifestUrl: string;
  watchProgress: WatchProgress | null;
  skipIntro: TimeRange | null;     // { start: 32, end: 95 }
  skipRecap: TimeRange | null;     // { start: 0, end: 45 }
  skipCredits: number | null;      // timestamp where credits start
}

interface TimeRange {
  start: number;   // seconds
  end: number;     // seconds
}

interface WatchProgress {
  position: number;                // seconds watched
  duration: number;                // total duration
  completed: boolean;              // true if watched > 90%
  lastWatchedAt: string;           // ISO 8601
}

interface ContentRow {
  id: string;
  title: string;                   // "Trending Now", "Because You Watched: Stranger Things"
  rowType: 'algorithmic' | 'continue_watching' | 'my_list' | 'top_10' | 'new_releases';
  items: TitleCardData[];
}

interface TitleCardData {
  id: string;
  name: string;
  type: 'movie' | 'show';
  posterUrl: string;
  backdropUrl: string;
  previewUrl: string | null;       // short muted clip for hover
  matchScore: number;
  maturityRating: string;
  duration: number | null;
  seasonCount: number | null;
  genres: string[];
  isInMyList: boolean;
  watchProgress: WatchProgress | null;
  rank: number | null;             // for "Top 10" rows
}

interface Profile {
  id: string;
  name: string;
  avatarUrl: string;
  isKids: boolean;
  maturityLevel: 'all' | 'kids' | 'teen' | 'adult';
  preferredLanguage: string;
  preferredSubtitleLanguage: string | null;
  autoplayEnabled: boolean;
}

interface AudioTrack {
  language: string;
  label: string;                   // "English [Original]", "Spanish"
  isDefault: boolean;
}

interface SubtitleTrack {
  language: string;
  label: string;
  url: string;                     // WebVTT/TTML file URL
  isForced: boolean;               // forced narrative subtitles (foreign language in an English title)
}
```

---

### 7.3 Entity Relationships

```
Profile  ──(1:many)──→  WatchProgress
Profile  ──(1:many)──→  MyListItem
Profile  ──(1:many)──→  ContentRow (personalized per profile)
Title    ──(1:many)──→  Season    ──(1:many)──→  Episode
Title    ──(1:many)──→  AudioTrack
Title    ──(1:many)──→  SubtitleTrack
Title    ──(1:1)────→   WatchProgress (per profile)
Episode  ──(1:1)────→   WatchProgress (per profile)
ContentRow ──(1:many)──→  TitleCardData (ordered items)
```

*   **Denormalized in API responses**: Browse API returns `TitleCardData` (lightweight) embedded in `ContentRow`. Full `Title` with seasons/episodes is fetched only when the user opens the title detail view.
*   **Normalized in client store**: Titles are stored in a map by ID so multiple rows referencing the same title share one data object. Watch progress is a separate map keyed by `profileId + titleId`.

---

### 7.4 UI Specific Data Models

```ts
// Derived state for a title card in a row
interface TitleCardUIState {
  formattedDuration: string;           // "1h 45m" or "3 Seasons"
  formattedMatchScore: string;         // "97% Match"
  progressPercent: number | null;      // 0-100 for "Continue Watching" progress bar
  isTop10: boolean;                    // shows rank badge
  rankDisplay: string | null;          // "1", "2", "10"
}

// Derived state for the video player
interface PlayerUIState {
  formattedCurrentTime: string;        // "1:23:45"
  formattedDuration: string;           // "2:15:30"
  formattedRemaining: string;          // "-51:45" (remaining time)
  progressPercent: number;
  bufferedPercent: number;
  showSkipIntro: boolean;              // true when currentTime is within skipIntro range
  showSkipRecap: boolean;
  showNextEpisode: boolean;            // true when currentTime >= skipCredits
  nextEpisodeCountdown: number;        // seconds until auto-advance
  currentAudioLabel: string;
  currentSubtitleLabel: string;
  qualityLabel: string;
}

// Browse page aggregated state
interface BrowseUIState {
  billboard: FeaturedTitle | null;
  rows: ContentRow[];
  loadedRowCount: number;
  isLoadingMoreRows: boolean;
}
```

---

## 8. State Management Strategy

### 8.1 State Classification

| State Type | What it contains | Tool |
|------------|-----------------|------|
| **Server state** | Browse rows, title metadata, episode lists, search results | React Query / TanStack Query (cache, refetch, stale-while-revalidate) |
| **Player state** | Current time, duration, buffered, playing/paused, audio track, subtitle track, quality | Local component state via `usePlayerState` hook (high-frequency updates) |
| **Global app state** | Active profile, auth session, theme, UI preferences | Zustand or React Context |
| **Feature state** | My List items, user ratings, search query/filters | Zustand slices or React Query mutations |
| **Persisted state** | Watch progress per title/episode, volume, subtitle preference, autoplay preference | localStorage synced to server via API |

---

### 8.2 State Ownership

| State | Owner | Why |
|-------|-------|-----|
| **Active profile** | Global Zustand store | Used by every API call (profile ID in headers); affects all personalized data |
| **Browse catalog rows** | React Query cache (keyed by profileId) | Server-owned personalized data; invalidated on profile switch |
| **Title detail** | React Query cache (keyed by titleId) | Fetched on demand; cached for back-navigation |
| **Player playback state** | Local `usePlayerState` hook | High-frequency `timeupdate` (~4/sec) must not propagate to global store |
| **Watch progress** | localStorage + React Query mutation for server sync | Local for instant resume; server sync for cross-device |
| **My List** | React Query with optimistic mutations | Server-owned but needs instant UI feedback |
| **Skip markers (intro/recap/credits)** | Episode metadata from API (immutable per episode) | Fetched once with episode data; drives timed UI (skip buttons, next episode overlay) |

##### Why Player State is Local (Same Pattern as YouTube)

The video `timeupdate` event fires ~4 times per second. Putting this in a global store would flood all subscribers with updates. By keeping it within the `VideoPlayer` component tree, only the progress bar and time display re-render. Components outside the player (browse page, header) are completely unaffected.

---

### 8.3 Persistence Strategy

| Data | Storage | Sync Strategy |
|------|---------|---------------|
| **Watch progress** | `localStorage: { [profileId_titleId]: { position, updatedAt } }` | Write on `timeupdate` (throttled every 10s); server sync via PUT every 30s and on pause/exit |
| **Volume and mute** | `localStorage: volume` | Write on `volumechange` |
| **Subtitle preference** | `localStorage: { preferredSubLang, subsEnabled }` | Write on change; auto-apply to next title |
| **Audio language preference** | `localStorage: preferredAudioLang` | Write on change; auto-select matching track |
| **Autoplay next episode** | `localStorage: autoplayEnabled` | Synced with profile setting |
| **Active profile** | `sessionStorage: activeProfileId` | Cleared on sign out; used to skip "Who's Watching?" on page refresh |
| **Search history** | `localStorage: searchHistory[]` (capped at 10) | Write on search submit; used for suggestions |

```tsx
// Watch progress sync with debounced server update
function useWatchProgressSync(
  profileId: string,
  contentId: string,
  videoRef: React.RefObject<HTMLVideoElement>
) {
  const lastSyncRef = useRef(0);

  // Throttled localStorage write (every 10s)
  useEffect(() => {
    const video = videoRef.current;
    if (!video) return;

    const handleTimeUpdate = () => {
      const now = Date.now();
      if (now - lastSyncRef.current < 10000) return;
      lastSyncRef.current = now;

      const key = `wp_${profileId}_${contentId}`;
      localStorage.setItem(key, JSON.stringify({
        position: Math.floor(video.currentTime),
        duration: Math.floor(video.duration),
        updatedAt: new Date().toISOString(),
      }));
    };

    video.addEventListener('timeupdate', handleTimeUpdate);
    return () => video.removeEventListener('timeupdate', handleTimeUpdate);
  }, [profileId, contentId, videoRef]);

  // Server sync on pause and component unmount
  useEffect(() => {
    const video = videoRef.current;
    if (!video) return;

    const syncToServer = () => {
      const position = Math.floor(video.currentTime);
      if (position < 5) return; // don't save very start positions

      fetch(`/api/profiles/${profileId}/history/${contentId}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ position, duration: Math.floor(video.duration) }),
        keepalive: true, // ensures request completes even if page is unloading
      }).catch(() => {}); // best-effort
    };

    video.addEventListener('pause', syncToServer);
    return () => {
      syncToServer(); // sync on unmount (navigate away)
      video.removeEventListener('pause', syncToServer);
    };
  }, [profileId, contentId, videoRef]);

  // Restore on mount
  useEffect(() => {
    const video = videoRef.current;
    const key = `wp_${profileId}_${contentId}`;
    const saved = localStorage.getItem(key);
    if (!video || !saved) return;

    const { position, duration } = JSON.parse(saved);
    // Only resume if not near the end
    if (position > 5 && position < duration * 0.95) {
      video.currentTime = position;
    }
  }, [profileId, contentId, videoRef]);
}
```

---

## 9. High Level API Design (Frontend POV)

### 9.1 Required APIs

| API | Method | Purpose |
|-----|--------|---------|
| `GET /api/profiles` | GET | List all profiles for the account |
| `GET /api/browse` | GET | Fetch personalized browse page (billboard + rows) |
| `GET /api/browse/rows` | GET | Fetch additional rows (pagination for lazy-loaded rows) |
| `GET /api/titles/:id` | GET | Fetch full title detail (movie or show with seasons/episodes) |
| `GET /api/titles/:id/episodes` | GET | Fetch episodes for a specific season |
| `GET /api/search` | GET | Search titles by query with filters |
| `GET /api/search/suggestions` | GET | Typeahead suggestions |
| `GET /api/mylist` | GET | Fetch user's My List |
| `POST /api/mylist/:titleId` | POST | Add to My List |
| `DELETE /api/mylist/:titleId` | DELETE | Remove from My List |
| `POST /api/ratings/:titleId` | POST | Rate a title (like/dislike) |
| `PUT /api/history/:contentId` | PUT | Update watch progress |
| `GET /api/history` | GET | Fetch watch history for profile |
| `GET /api/titles/:id/similar` | GET | Fetch similar/recommended titles |

---

### 9.2 Phased Row Loading Strategy (Deep Dive)

Loading all 75 browse rows at once would be wasteful and slow. Netflix uses a **phased loading strategy** that prioritizes above-the-fold content.

```
Phase 1: SSR (server side, critical path)
────────────────────────────────────────
  Server fetches: billboard + first 4 rows (Continue Watching, Trending, Top 10, New Releases)
  Renders: HTML with billboard artwork + first 4 row placeholders with poster images
  Result: FCP in < 1.5s — user sees content immediately

Phase 2: Client Hydration (after JS loads, ~1-2s after FCP)
────────────────────────────────────────────────────────
  React hydrates → makes billboard trailer playable → hover interactions enabled
  Fetches: next 5 rows (GET /api/browse/rows?offset=4&limit=5)
  Result: 9 total rows available; billboard trailer starts playing

Phase 3: On-Scroll Lazy Loading (as user scrolls down)
────────────────────────────────────────────────────
  IntersectionObserver detects scroll approaching row 8-9
  Fetches: next batch of rows (GET /api/browse/rows?offset=9&limit=5)
  Result: rows load just-in-time as user scrolls

Phase 4: Background Prefetch (after 5s of idle time)
────────────────────────────────────────────────────
  If user hasn't scrolled, prefetch rows 10-14 in background
  Low priority fetch (requestIdleCallback or setTimeout)
  Result: if user eventually scrolls, rows are already cached
```

##### Why Not Load All Rows in One API Call?

| Approach | Payload | Time to first row visible | Wasted bandwidth |
|----------|---------|--------------------------|------------------|
| **All 75 rows at once** | ~200-500KB JSON | 1-3s (blocked on huge response) | ~90% (user sees < 10 rows in average session) |
| **Phased loading (4 + 5 + 5...)** | ~15KB per batch | < 500ms for first batch | Minimal (only load what's needed) |

---

### 9.3 Request and Response Structure

##### Get Browse Page (Initial)

```json
// GET /api/browse?profile=prof_abc

// Response:
{
  "billboard": {
    "titleId": "tt_stranger",
    "name": "Stranger Things",
    "synopsis": "When a young boy vanishes, a town uncovers...",
    "year": 2025,
    "maturityRating": "TV-14",
    "backdropUrl": "https://cdn.example.com/art/stranger_backdrop.webp",
    "logoUrl": "https://cdn.example.com/art/stranger_logo.png",
    "trailerUrl": "https://cdn.example.com/trailers/stranger_s5.mp4",
    "manifestUrl": "https://cdn.example.com/v/stranger_s5e1/manifest.mpd"
  },
  "rows": [
    {
      "id": "row_continue",
      "title": "Continue Watching for Zeeshan",
      "rowType": "continue_watching",
      "items": [
        {
          "id": "tt_001",
          "name": "Wednesday",
          "type": "show",
          "posterUrl": "https://cdn.example.com/posters/wednesday.webp",
          "backdropUrl": "https://cdn.example.com/backdrops/wednesday.webp",
          "previewUrl": "https://cdn.example.com/previews/wednesday_s2.mp4",
          "matchScore": 95,
          "maturityRating": "TV-14",
          "seasonCount": 2,
          "genres": ["Mystery", "Comedy", "Fantasy"],
          "isInMyList": true,
          "watchProgress": {
            "position": 1845,
            "duration": 2940,
            "completed": false,
            "lastWatchedAt": "2026-03-16T22:30:00Z"
          }
        }
      ]
    },
    {
      "id": "row_trending",
      "title": "Trending Now",
      "rowType": "algorithmic",
      "items": []
    }
  ],
  "pagination": {
    "offset": 0,
    "limit": 4,
    "totalRows": 62,
    "hasMore": true
  }
}
```

##### Get Title Detail

```json
// GET /api/titles/tt_stranger

// Response:
{
  "data": {
    "id": "tt_stranger",
    "type": "show",
    "name": "Stranger Things",
    "synopsis": "When a young boy vanishes, a small town uncovers a mystery...",
    "year": 2016,
    "maturityRating": "TV-14",
    "matchScore": 97,
    "genres": ["Sci-Fi", "Horror", "Drama"],
    "cast": ["Millie Bobby Brown", "Finn Wolfhard", "Winona Ryder"],
    "posterUrl": "https://cdn.example.com/posters/stranger.webp",
    "backdropUrl": "https://cdn.example.com/backdrops/stranger.webp",
    "logoUrl": "https://cdn.example.com/logos/stranger.png",
    "trailerUrl": "https://cdn.example.com/trailers/stranger_main.mp4",
    "seasonCount": 5,
    "seasons": [
      {
        "seasonNumber": 5,
        "episodeCount": 8,
        "episodes": [
          {
            "id": "ep_s5e1",
            "seasonNumber": 5,
            "episodeNumber": 1,
            "name": "The Crawl",
            "synopsis": "The gang reunites to face...",
            "duration": 3120,
            "thumbnailUrl": "https://cdn.example.com/thumbs/s5e1.webp",
            "manifestUrl": "https://cdn.example.com/v/stranger_s5e1/manifest.mpd",
            "watchProgress": { "position": 1845, "duration": 3120, "completed": false, "lastWatchedAt": "2026-03-16T22:30:00Z" },
            "skipIntro": { "start": 5, "end": 67 },
            "skipRecap": null,
            "skipCredits": 2980
          }
        ]
      }
    ],
    "audioTracks": [
      { "language": "en", "label": "English [Original]", "isDefault": true },
      { "language": "es", "label": "Spanish", "isDefault": false }
    ],
    "subtitleTracks": [
      { "language": "en", "label": "English", "url": "https://cdn.example.com/subs/s5e1_en.vtt", "isForced": false },
      { "language": "es", "label": "Spanish", "url": "https://cdn.example.com/subs/s5e1_es.vtt", "isForced": false }
    ],
    "isInMyList": true,
    "userRating": "liked"
  }
}
```

---

### 9.4 Error Handling and Status Codes

| Status | Meaning | Frontend Behavior |
|--------|---------|-------------------|
| `200` | Success | Render data normally |
| `400` | Bad request | Show inline error; log to monitoring |
| `401` | Unauthorized / session expired | Redirect to login page |
| `403` | Forbidden (maturity, geo-block) | Show "This title is not available" with reason (e.g., "Content not available in your region") |
| `404` | Title/episode not found | Show "This title is no longer available" page |
| `429` | Rate limited | Back off; queue retry; show loading state |
| `500` | Server error | Show cached data if available; generic error with retry |
| `503` | Service unavailable | Show cached browse page; retry with backoff |

---

## 10. Caching Strategy

### 10.1 What to Cache

| Data | Cache? | Reason |
|------|--------|--------|
| **Browse rows** | Yes (stale-while-revalidate, 5min staleTime) | Show cached browsing experience instantly on revisit; refetch in background |
| **Title detail** | Yes (per titleId, 10min TTL) | Rarely changes; user may navigate back and forth |
| **Episode list** | Yes (per titleId + season, 10min) | Same as title detail |
| **Search results** | Yes (per query, 2min TTL) | Quick back-navigation to search results |
| **My List** | Yes (per profileId) | Invalidated on add/remove mutation |
| **Watch progress** | localStorage + server sync | Must survive session; needs cross-device access |
| **Video segments** | Yes (HTTP cache, immutable, long TTL) | Segments never change after transcoding |
| **Artwork (posters, backdrops)** | Yes (HTTP cache, long TTL, CDN) | Static assets; URL changes when artwork updates |
| **Trailer clips** | Yes (HTTP cache, moderate TTL) | May be updated for new seasons |

### 10.2 Where to Cache

| Layer | What | How |
|-------|------|-----|
| **React Query in-memory** | API responses (rows, titles, episodes, search) | Automatic with `staleTime` / `gcTime` |
| **HTTP browser cache** | Video segments (.m4s), artwork (.webp), subtitle files (.vtt) | `Cache-Control: public, max-age=31536000, immutable` for versioned assets |
| **Service Worker** | App shell, JS/CSS bundles, fonts | Precache on install; network-first for API, cache-first for static |
| **localStorage** | Watch progress, volume, subtitle prefs, search history | Written on user action; read on mount |
| **CDN edge** | All static assets and video segments | Multi-region edge servers; long TTL |

### 10.3 Cache Invalidation

| Data | Invalidation Strategy |
|------|----------------------|
| **Browse rows** | Time-based (5min staleTime); manual refetch on profile switch |
| **Title detail** | Time-based (10min); refetch on mutation (rate, My List change) |
| **My List** | Optimistic update on add/remove; server-confirmed |
| **Watch progress** | Overwritten on each sync; no invalidation needed |
| **Video segments** | Immutable — never invalidated; new transcode = new URLs |
| **Artwork** | URL-versioned; long cache; new URL on artwork update |
| **Profile change** | Clear all profile-specific React Query cache; refetch browse rows |

---

## 11. CDN and Asset Optimization

### 11.1 Video Delivery

*   Video segments served from **multi-region CDN edge servers** (Netflix uses Open Connect Appliances placed inside ISP networks for extreme locality).
*   Each segment is 2-4 seconds of video at a specific quality level.
*   `Cache-Control: public, max-age=31536000, immutable` — segments are immutable.
*   Manifest files (`.mpd`) have short TTL for live content but can be cached longer for VOD.

### 11.2 Artwork Optimization

| Strategy | Implementation |
|----------|----------------|
| **Format** | WebP (primary), AVIF (progressive enhancement), JPEG fallback |
| **Responsive sizes** | Multiple artwork sizes: 160px (small card), 342px (row card), 800px (billboard thumbnail), 1920px (billboard backdrop) |
| **Lazy loading** | `loading="lazy"` for all images outside the first 2 visible rows; `loading="eager"` for billboard and first row |
| **Dominant color placeholder** | API provides `dominantColor: "#1a1a2e"` per title; used as CSS `background-color` until poster loads |
| **CDN image transforms** | Dynamic resizing via CDN (e.g., `posters/tt_001.webp?w=342&q=80`) |

```tsx
function TitlePoster({ title, size }: { title: TitleCardData; size: 'small' | 'medium' | 'large' }) {
  const widthMap = { small: 160, medium: 342, large: 800 };
  const width = widthMap[size];

  return (
    <div
      className="poster-container"
      style={{ backgroundColor: title.dominantColor || '#141414', aspectRatio: '16/9' }}
    >
      <img
        src={`${title.posterUrl}?w=${width}&q=80`}
        alt={title.name}
        width={width}
        loading="lazy"
        decoding="async"
      />
    </div>
  );
}
```

### 11.3 Static Asset Optimization

*   **JS bundles**: Code-split per route; main bundle < 120KB gzipped; player chunk loaded on playback; title detail loaded on click.
*   **CSS**: Critical CSS inlined in SSR; browse-specific CSS in main chunk; player CSS code-split.
*   **Fonts**: Netflix Sans font subsetted; `font-display: swap`; preloaded via `<link rel="preload">`.
*   **Compression**: Brotli (preferred) or gzip for all text assets.

---

## 12. Rendering Strategy

| Page | Strategy | Rationale |
|------|----------|-----------|
| **Browse page** | **SSR** (billboard + first 4 rows) + **CSR** (remaining rows, hover interactions, trailer) | Fast FCP with content visible; subsequent rows lazy loaded client-side |
| **Title detail** | **CSR** (rendered as modal overlay or client-navigated page) | Opened from browse — data fetched on demand; no SEO benefit from modal |
| **Video player** | **CSR** (full-screen overlay) | Pure interactive experience; no SSR benefit |
| **Search** | **CSR** (overlay on browse page) | Real-time typing interaction; no SSR benefit |
| **Profile select** | **SSR** (simple static page) | Fast initial load; minimal JS needed |
| **Title canonical URL** (e.g., /title/stranger-things) | **SSR** | SEO for search engines and social sharing (OG tags) |

### Hydration Strategy

*   Server renders billboard artwork + first 4 rows with poster images → browsers paints immediately.
*   Client hydrates React → interactive elements become functional (hover, play, carousel arrows).
*   **Progressive hydration**: Billboard hydrates first (above fold); carousel rows hydrate next; title detail and player modules are not loaded until needed.
*   Use `<Suspense>` boundaries for code-split chunks (player, title detail, search overlay).

---

## 13. Cross Cutting Non Functional Concerns

### 13.1 Security

*   **DRM (Digital Rights Management)**: All premium content encrypted. Uses Encrypted Media Extensions (EME) with:
    *   Widevine L1 (Chrome, Firefox, Android) — hardware-backed decryption
    *   FairPlay (Safari, iOS, macOS)
    *   PlayReady (Edge, Windows)
    *   License keys are session-specific, short-lived, and never exposed to JavaScript.
*   **Signed CDN URLs**: Video manifests and segments use short-lived signed tokens (e.g., `?token=abc&expires=1710000000`).
*   **HDCP enforcement**: For HD/4K content, browser must support HDCP (hardware output protection). If not, player caps at 720p.
*   **XSS prevention**: All user-generated content (profile names, search queries) rendered via React JSX (auto-escaped).
*   **CSRF**: SameSite=Strict cookies; CSRF tokens on mutations.
*   **CSP headers**: Restrict script sources; only allow CDN domains for media; no inline scripts.
*   **Token storage**: Auth tokens in HTTP-only, Secure, SameSite=Lax cookies.

---

### 13.2 Accessibility

*   **Browse page**:
    *   Full keyboard navigation: arrow keys for spatial navigation across rows and cards.
    *   Screen reader: each row is a `role="group"` with `aria-label`; each card has descriptive `aria-label` (title name, maturity, genre).
    *   Focus indicators: visible focus ring on all interactive elements.
    *   Skip link: "Skip to content" bypasses header and navigation.
    *   Reduced motion: respect `prefers-reduced-motion` — disable carousel animations, autoplay trailers, hover expansions.
*   **Video player**:
    *   All controls keyboard-operable; focus trapped within player when in fullscreen.
    *   `role="slider"` for seek and volume with `aria-valuenow`, `aria-valuetext`.
    *   Closed captions (CC): user-controllable; multiple languages; customizable size and background.
    *   Audio descriptions (AD): alternative audio track for visually impaired users.
    *   Screen reader announcements for state changes (buffering, quality change, next episode) via `aria-live`.
*   **Color contrast**: WCAG AA (4.5:1) minimum. Netflix's dark theme makes this challenging — careful attention to text on artwork overlays.

---

### 13.3 Performance Optimization

*   **Code splitting**: Route-based (browse, player, search, title detail); player module (~80KB) only loaded on playback.
*   **Image lazy loading**: IntersectionObserver-based row rendering ensures only visible rows' images load.
*   **Hover preview optimization**: Preview videos fetched only after 1.5s hover (not on every mouse-enter).
*   **Billboard trailer**: Delayed 2s; preloaded via `<link rel="preload">`; stopped when scrolled past.
*   **Row prefetch on idle**: After initial rows render and 5s of inactivity, prefetch next batch of rows in background.
*   **DRM license pre-acquisition**: On hover over Play button, start license handshake to eliminate 200ms wait.
*   **Virtualized row rendering**: Only 6-8 rows in DOM at a time out of 75 total.
*   **Metrics tracked**: FCP, LCP (billboard), TTI, CLS, time-to-first-frame (playback), buffering ratio, carousel scroll fps.

---

### 13.4 Observability and Reliability

*   **Error boundaries**: Independent error boundaries for billboard, each carousel row, title detail modal, and player. A crash in one row doesn't take down the page.
*   **Logging**: Capture player errors (DRM failures, segment errors, manifest issues), API errors, JS exceptions. Send to Sentry / Datadog.
*   **Video quality metrics**: Per-session: average bitrate, quality switches, buffering duration, stall count, time-to-first-frame, play failures.
*   **A/B testing**: Rows, billboard content, player UI changes, recommendation algorithms all behind feature flags with gradual rollout.
*   **Browse metrics**: Row impression tracking (which rows/titles are seen), click-through rate, hover-to-play conversion, time-to-first-play from browse.

```tsx
function BrowsePage() {
  return (
    <div className="browse-page">
      <ErrorBoundary fallback={<BillboardFallback />}>
        <BillboardHero featured={billboard} />
      </ErrorBoundary>

      {rows.map((row) => (
        <ErrorBoundary key={row.id} fallback={<RowErrorFallback />}>
          <LazyRow row={row} />
        </ErrorBoundary>
      ))}

      <ErrorBoundary fallback={<PlayerErrorFallback />}>
        <Suspense fallback={<PlayerLoadingScreen />}>
          {isPlayerActive && <VideoPlayer {...playerProps} />}
        </Suspense>
      </ErrorBoundary>
    </div>
  );
}
```

---

## 14. Edge Cases and Tradeoffs

| Edge Case | Handling |
|-----------|---------|
| **Very slow network (2G)** | ABR drops to lowest quality (240p); poster images compressed to smallest variant; hover previews disabled; billboard trailer disabled |
| **DRM not supported (older browser)** | Detect via `navigator.requestMediaKeySystemAccess()` check; show "Your browser doesn't support playback" message with browser upgrade suggestion |
| **HDCP not available** | Player caps maximum quality at 720p SD; user is informed "HD not available on this display" |
| **User has 200+ items in My List** | Paginate My List API (cursor-based); virtualize the My List page grid |
| **Autoplay blocked by browser** | Billboard trailer: fall back to static artwork silently. Player: show "Click to play" overlay (same as YouTube). Muted autoplay is always allowed. |
| **Multiple devices streaming** | API returns `concurrentStreamLimit` error; show "Too many streams" modal with option to stop other sessions |
| **Profile switch mid-session** | Clear all profile-specific React Query cache; reset player; re-fetch browse data for new profile |
| **Episode out of order** | User can pick any episode; update watch progress for that specific episode; "Continue Watching" should surface the next unwatched episode |
| **Title removed from catalog** | API returns 404; remove from local cache; show "no longer available" toast; remove from My List automatically |
| **Subtitles out of sync** | Provide subtitle offset control (rare in web; common in TV apps); report mechanism |
| **Very long movie (3+ hours)** | Buffer only 30s ahead; progress bar precision at seconds; thumbnail preview sprites paginated |
| **Kids profile** | API filters out mature content; UI hides maturity badges; browse rows are kid-specific; player omits certain controls |
| **RTL languages (Arabic, Hebrew)** | Mirror carousel navigation (right arrow = previous); flip arrow buttons; adjust text alignment |
| **Offline / intermittent connection** | Continue playback from buffer; show connection lost banner; pause API syncs; resume when back online |

---

## 15. Summary and Future Improvements

### Key Architectural Decisions

| Decision | Rationale |
|----------|-----------|
| SPA with SSR for browse page | SEO for canonical title URLs; fast FCP with artwork visible; rich client-side interactivity |
| DASH with DRM (shaka-player) | Industry standard for premium content protection; multi-DRM support in one library |
| Paginated carousel with CSS transform | GPU-accelerated; no layout reflow; consistent animation curves; works with D-pad |
| Two-level virtualization (rows + cards) | 75 rows is too many for DOM; only 3-4 visible at once; saves thousands of DOM nodes |
| Hover expand with `transform: scale()` | Compositor-only animation; no reflow; smooth 60fps expansion |
| React Query for server state | Stale-while-revalidate for instant revisits; optimistic mutations for My List |
| Local state for player internals | High-frequency `timeupdate` isolated from global store; prevents cascade re-renders |
| Phased row loading (SSR 4 + lazy batches) | Minimize initial payload; load rows just-in-time as user scrolls |
| DRM license pre-acquisition | Eliminates 200ms license wait on playback start |

### Major Tradeoffs

| Tradeoff | Chosen | Alternative | Why |
|----------|--------|-------------|-----|
| DRM complexity vs open playback | DRM required | Unencrypted streams | Content licensing obligations; piracy prevention |
| Paginated navigation vs free scroll | Paginated | Free scroll with snap | Cleaner grid alignment; better D-pad/remote support |
| Hover video delay vs instant | 1.5s delay | Immediate autoplay | Saves significant bandwidth; prevents accidental triggers |
| SSR for browse vs full CSR | SSR first 4 rows | Full CSR | 500ms faster FCP; SEO for title pages |
| Virtualized rows vs render all | Virtualize | Render all | 75 rows = 10K+ DOM nodes without virtualization |
| localStorage for progress vs server-only | Local + server sync | Server-only | Instant resume; server sync for cross-device |

### Future Enhancements

*   **Watch party**: Synchronized playback across multiple users via WebSocket with shared chat.
*   **Interactive content**: Choose-your-own-adventure episodes (like Bandersnatch) with branching manifest segments.
*   **Spatial audio**: Dolby Atmos support via Web Audio API for compatible browsers.
*   **Picture-in-Picture browse**: Continue watching in PiP while browsing catalog.
*   **Smart downloads**: Service Worker pre-caches next episode segments on WiFi for uninterrupted viewing.
*   **Personalized thumbnails**: A/B test different artwork per user; API returns user-specific poster URLs.
*   **Voice search**: Web Speech API integration for hands-free search input.
*   **10-foot UI mode**: Detect TV browser (Samsung Tizen, LG webOS) and switch to large-text, D-pad-optimized layout automatically.
*   **Ambient mode**: Extract video colors for background glow effect while browsing with mini player.

---

Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: [https://github.com/ZeeshanAli-0704/front-end-system-design](https://github.com/ZeeshanAli-0704/front-end-system-design)
