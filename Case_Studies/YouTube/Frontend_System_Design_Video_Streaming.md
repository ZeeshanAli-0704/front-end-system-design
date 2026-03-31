# Frontend System Design: Video Streaming (YouTube)

- [Frontend System Design: Video Streaming (YouTube)](#frontend-system-design-video-streaming-youtube)
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
    - [5.2 Video Player Architecture and Adaptive Streaming (Deep Dive)](#52-video-player-architecture-and-adaptive-streaming-deep-dive)
      - [5.2.1 HLS and DASH Adaptive Bitrate Streaming](#521-hls-and-dash-adaptive-bitrate-streaming)
      - [5.2.2 Video Player State Machine](#522-video-player-state-machine)
      - [5.2.3 Custom Controls Overlay](#523-custom-controls-overlay)
      - [5.2.4 Buffering and Preload Strategy](#524-buffering-and-preload-strategy)
      - [5.2.5 Quality Selection and Auto Resolution](#525-quality-selection-and-auto-resolution)
      - [5.2.6 Picture in Picture and Mini Player](#526-picture-in-picture-and-mini-player)
      - [5.2.7 Keyboard Shortcuts and Accessibility](#527-keyboard-shortcuts-and-accessibility)
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
    - [9.2 Adaptive Streaming Manifest Fetching Strategy (Deep Dive)](#92-adaptive-streaming-manifest-fetching-strategy-deep-dive)
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

*   A web application that allows users to discover, watch, and interact with video content — similar to YouTube.
*   Target users: content consumers (watchers), content creators (uploaders), and advertisers.
*   Primary use case: browsing a personalized video feed, watching videos with adaptive streaming, and engaging through likes, comments, and subscriptions.

---

### 1.2 Key User Personas

*   **Viewer**: Browses the home feed, searches for content, watches videos, interacts with likes/comments, subscribes to channels, manages watch history and playlists.
*   **Creator**: Uploads videos, manages channel page, reads comments, tracks analytics (views, watch time, subscribers).
*   **Casual Browser**: Lands on a shared link, watches a single video, may or may not sign in.

---

### 1.3 Core User Flows (High Level)

*   **Watching a Video (Primary Flow)**:
    1.  User opens the app or clicks a shared link → video page loads.
    2.  Video player initializes → fetches adaptive streaming manifest (HLS/DASH).
    3.  Video starts playing at an appropriate quality based on network conditions.
    4.  User interacts with player controls (play/pause, seek, volume, fullscreen, quality selector).
    5.  Recommended videos load in the sidebar or below the player.
    6.  User scrolls to comments section, reads or posts a comment.

*   **Browsing the Home Feed (Secondary Flow)**:
    1.  User opens the home page → personalized video grid loads.
    2.  Video thumbnails display with title, channel name, view count, and upload date.
    3.  User hovers on a thumbnail → silent video preview plays (desktop).
    4.  User clicks a thumbnail → navigates to the video watch page.
    5.  Infinite scroll loads more recommendations as user scrolls.

*   **Searching for Content (Secondary Flow)**:
    1.  User types in the search bar → typeahead suggestions appear.
    2.  User presses enter → search results page with video cards, channel results, and filters.
    3.  User clicks a result → navigates to the video watch page.

---

## 2. Requirements

### 2.1 Functional Requirements

*   **Video Playback**:
    *   Adaptive bitrate streaming (HLS/DASH) with automatic quality switching.
    *   Custom video player controls (play/pause, seek bar, volume, fullscreen, playback speed, quality selector, captions toggle).
    *   Resume playback from last watched position ("Continue watching").
    *   Picture-in-Picture (PiP) and mini-player on scroll.
    *   Keyboard shortcuts (space = play/pause, arrow keys = seek, f = fullscreen, m = mute, c = captions).
*   **Home Feed**:
    *   Personalized video grid (thumbnail, title, channel info, view count, timestamp).
    *   Category chips for filtering (Trending, Music, Gaming, etc.).
    *   Infinite scroll with cursor-based pagination.
    *   Thumbnail hover preview (silent autoplay on desktop).
*   **Video Page**:
    *   Video player (primary content).
    *   Video metadata (title, description with expandable "Show More", tags, publish date).
    *   Engagement actions: like, dislike, share, save to playlist, subscribe/unsubscribe.
    *   Comments section with threaded replies, sort options (Top, Newest), infinite scroll.
    *   Recommended videos sidebar (desktop) or below player (mobile).
*   **Search**:
    *   Typeahead suggestions with debounced input.
    *   Search results with filtering (upload date, type, duration, sort order).
*   **Channel Page**:
    *   Channel banner, avatar, name, subscriber count, subscribe button.
    *   Tabbed content: Videos, Shorts, Playlists, Community, About.

---

### 2.2 Non Functional Requirements

*   **Performance**: Video playback start (time-to-first-frame) < 2s; FCP < 1.5s for feed pages; TTI < 3s; smooth 60fps scrolling for video grid; buffer ahead of playback to prevent stalls.
*   **Scalability**: Support millions of videos in search/recommendations; video pages with millions of comments; home feed with thousands of personalized entries.
*   **Availability**: Graceful degradation — show cached thumbnails if offline; fallback to lower quality if bandwidth drops; skeleton UI during loading.
*   **Security**: Signed CDN URLs for video segments; DRM integration (Widevine/FairPlay) for protected content; XSS prevention on user-generated comments/descriptions; CSRF tokens on all mutations.
*   **Accessibility**: Full keyboard navigation for player and UI; screen reader support with ARIA roles; closed captions and subtitles; focus management for modals and overlays; reduced motion support.
*   **Device Support**: Desktop web (primary), mobile web (responsive), tablet; low-bandwidth and low-end device handling via adaptive streaming.
*   **i18n**: RTL layout support; localized timestamps ("2 hours ago"); locale-aware number formatting (1.2M views); multi-language subtitle/caption support.

---

## 3. Scope Clarification (Interview Scoping)

### 3.1 In Scope

*   Video player component with adaptive streaming (HLS/DASH).
*   Custom player controls (seek, quality, captions, PiP, keyboard shortcuts).
*   Home feed video grid with infinite scroll.
*   Video watch page (player + metadata + engagement + recommendations + comments).
*   State management for playback, feed, and user interactions.
*   API design from the frontend perspective.
*   Performance optimization for video-heavy pages.

---

### 3.2 Out of Scope

*   Video upload and transcoding pipeline (backend concern).
*   Backend recommendation and ranking algorithms.
*   Ads and monetization system (pre-roll, mid-roll ad insertion).
*   Live streaming (a separate deep-dive).
*   Shorts / vertical video player.
*   Push notifications.
*   Creator Studio / Analytics dashboard.
*   Native mobile apps.

---

### 3.3 Assumptions

*   User may or may not be authenticated; unauthenticated users can watch videos but cannot interact (like, comment, subscribe).
*   Videos are pre-transcoded by the backend into multiple quality levels (240p to 4K) and available as HLS/DASH manifests.
*   Video segments and thumbnails are served from a CDN with signed or public URLs.
*   APIs return pre-ranked recommendations and search results (frontend does not rank).
*   Captions/subtitles are available as WebVTT files served alongside video manifests.

---

## 4. High Level Frontend Architecture

### 4.1 Overall Approach

*   **SPA** (Single Page Application) with client-side routing.
*   **SSR** for the video watch page shell — server renders the video metadata (title, description, channel info) for fast FCP and SEO (shared links, search engine indexing).
*   **CSR** for all interactive elements — video player, comments, recommendations, likes, and subscriptions are hydrated and managed client-side.
*   The video player module is the **primary chunk**; secondary features (comments, share modal, playlist modal) are **code-split** and lazy loaded.
*   Home feed page uses **SSR** for the initial grid shell with **CSR** for infinite scroll and hover previews.

---

### 4.2 Major Architectural Layers

```
┌────────────────────────────────────────────────────────────────────┐
│  UI Layer                                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ VideoWatchPage                                                │  │
│  │  ┌─────────────────────┐  ┌──────────────────────────────┐   │  │
│  │  │ VideoPlayer         │  │ RecommendedVideosSidebar     │   │  │
│  │  │  ┌───────────────┐  │  │  ┌────────────────────────┐  │   │  │
│  │  │  │ ControlsBar   │  │  │  │ VideoCard[]            │  │   │  │
│  │  │  │ ProgressBar   │  │  │  └────────────────────────┘  │   │  │
│  │  │  │ QualityMenu   │  │  └──────────────────────────────┘   │  │
│  │  │  │ CaptionsLayer │  │                                     │  │
│  │  │  └───────────────┘  │  ┌──────────────────────────────┐   │  │
│  │  └─────────────────────┘  │ CommentsSection               │   │  │
│  │  ┌─────────────────────┐  │  ┌────────────────────────┐  │   │  │
│  │  │ VideoMetadata       │  │  │ CommentItem[]           │  │   │  │
│  │  │ EngagementBar       │  │  └────────────────────────┘  │   │  │
│  │  └─────────────────────┘  └──────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ HomeFeedPage                                                  │  │
│  │  ┌────────────────┐  ┌────────────────────────────────────┐  │  │
│  │  │ CategoryChips  │  │ VideoGrid (responsive grid)        │  │  │
│  │  └────────────────┘  │  ┌──────────┐ ┌──────────┐        │  │  │
│  │                      │  │VideoCard │ │VideoCard │ ...     │  │  │
│  │                      │  └──────────┘ └──────────┘        │  │  │
│  │                      └────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
├────────────────────────────────────────────────────────────────────┤
│  State Management Layer                                            │
│  (Player State, Feed Store, Watch History, User Session,           │
│   Comments State, Subscription State, UI Preferences)              │
├────────────────────────────────────────────────────────────────────┤
│  API and Data Access Layer                                         │
│  (REST/GraphQL Client, Streaming Manifest Fetcher,                 │
│   Optimistic Update Queue, Request Deduplication, Retry Logic)     │
├────────────────────────────────────────────────────────────────────┤
│  Shared / Utility Layer                                            │
│  (IntersectionObserver helpers, Debounce/Throttle, Date formatting,│
│   Number formatting, Media Session API, Analytics tracker,         │
│   ABR Controller, DRM Manager)                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

### 4.3 External Integrations

*   **CDN**: Serves video segments (HLS .ts / DASH .m4s), thumbnails, channel avatars, and static assets (Cloudfront / Akamai / Fastly / Google CDN).
*   **Analytics SDK**: Track video impressions, play events, watch time, buffering events, quality switches, engagement actions, and session duration.
*   **Backend Services**: Video metadata API, recommendations API, comments API, search API, subscription API, watch history API.
*   **Media Player Libraries**: hls.js (HLS playback), dash.js (DASH playback), or shaka-player (unified HLS/DASH + DRM).
*   **DRM Services**: Widevine (Chrome, Android), FairPlay (Safari, iOS), PlayReady (Edge) for protected content licensing.
*   **Caption Services**: WebVTT subtitle files served alongside video manifests.

---

## 5. Component Design and Modularization

### 5.1 Component Hierarchy

```
App
 ├── Header
 │    ├── Logo (link to home)
 │    ├── SearchBar
 │    │    ├── SearchInput (with debounced typeahead)
 │    │    └── SuggestionsDropdown
 │    ├── VoiceSearchButton
 │    └── UserMenu (avatar, sign in, settings)
 │
 ├── Sidebar (collapsible navigation)
 │    ├── NavItem[] (Home, Trending, Subscriptions, Library, History)
 │    └── SubscriptionsList (channel links)
 │
 ├── HomeFeedPage
 │    ├── CategoryChips (horizontal scrollable filter chips)
 │    ├── VideoGrid (responsive CSS grid)
 │    │    └── VideoCard[] (repeated)
 │    │         ├── ThumbnailContainer
 │    │         │    ├── ThumbnailImage
 │    │         │    ├── HoverPreview (silent video on hover, desktop only)
 │    │         │    ├── DurationBadge ("12:34")
 │    │         │    └── WatchProgressBar (if partially watched)
 │    │         ├── VideoInfo
 │    │         │    ├── ChannelAvatar
 │    │         │    ├── VideoTitle (2 line clamp)
 │    │         │    ├── ChannelName
 │    │         │    └── MetaInfo ("1.2M views · 3 days ago")
 │    │         └── MoreMenu (Save, Add to queue, Not interested)
 │    └── FeedSentinel (IntersectionObserver for infinite scroll)
 │
 └── VideoWatchPage
      ├── VideoPlayer
      │    ├── VideoElement (HTML5 video with HLS/DASH source)
      │    ├── ControlsOverlay
      │    │    ├── PlayPauseButton
      │    │    ├── ProgressBar (seekable with preview thumbnails)
      │    │    ├── VolumeControl (slider + mute toggle)
      │    │    ├── TimeDisplay ("2:34 / 12:56")
      │    │    ├── PlaybackSpeedMenu
      │    │    ├── QualitySelector (Auto, 240p to 4K)
      │    │    ├── CaptionsToggle
      │    │    ├── PiPButton
      │    │    ├── TheaterModeButton
      │    │    └── FullscreenButton
      │    ├── CaptionsRenderer (WebVTT overlay)
      │    ├── BufferingSpinner
      │    └── SeekPreviewThumbnail (on hover over progress bar)
      │
      ├── VideoMetadata
      │    ├── VideoTitle
      │    ├── ViewCount + PublishDate
      │    └── ExpandableDescription ("Show More" / "Show Less")
      │
      ├── EngagementBar
      │    ├── LikeButton (with count)
      │    ├── DislikeButton
      │    ├── ShareButton → ShareModal
      │    ├── SaveButton → PlaylistModal
      │    └── MoreActions (Report, Transcript)
      │
      ├── ChannelInfo
      │    ├── ChannelAvatar
      │    ├── ChannelName + SubscriberCount
      │    └── SubscribeButton (toggle state)
      │
      ├── CommentsSection (lazy loaded)
      │    ├── CommentCount + SortToggle (Top / Newest)
      │    ├── CreateComment (avatar + input + submit)
      │    └── CommentList
      │         └── CommentItem[]
      │              ├── CommentAvatar
      │              ├── CommentBody (username + timestamp + text)
      │              ├── CommentActions (like, dislike, reply)
      │              └── RepliesThread (expandable, lazy loaded)
      │
      └── RecommendedVideos (sidebar on desktop, below on mobile)
           └── VideoCard[] (compact horizontal layout)
```

---

### 5.2 Video Player Architecture and Adaptive Streaming (Deep Dive)

The video player is the **core of the entire application** — the primary surface where users spend the vast majority of their time. Getting this right matters because:
*   Users expect **instant playback** — any delay in time-to-first-frame is immediately noticeable.
*   Network conditions vary wildly — from 5G to 2G to flaky WiFi. The player must **adapt in real-time** to avoid buffering stalls.
*   The player must handle a complex state machine — idle, loading, playing, paused, buffering, seeking, ended, error — with smooth transitions.
*   Custom controls must layer on top of the native `<video>` element without breaking accessibility or keyboard navigation.
*   DRM-protected content requires integration with browser-specific license servers.

---

#### 5.2.1 HLS and DASH Adaptive Bitrate Streaming

##### Why Not Just Serve a Single MP4 File?

| Approach | How it works | Problems |
|----------|-------------|----------|
| **Single MP4** | Server sends one fixed-quality video file | Cannot adapt to network changes; wastes bandwidth on slow connections; no quality options; huge download for HD/4K; no seeking without range requests; browser must download from start |
| **Adaptive streaming (HLS/DASH)** | Video is split into 2-10s segments at multiple quality levels; a manifest file describes all available qualities; the client requests segments one at a time, switching quality as needed | Adapts to bandwidth in real-time; fast startup (starts with low quality and upgrades); efficient seeking (jump to any segment); works with CDN caching per-segment |

##### How Adaptive Bitrate (ABR) Streaming Works

```
┌──── Backend (pre-processing) ────────────────────────────────────┐
│                                                                   │
│  Original Video (1080p, 2GB)                                     │
│       │                                                           │
│       ▼                                                           │
│  Transcoder (FFmpeg / MediaConvert)                              │
│       │                                                           │
│       ├── 240p   → segments (seg0.ts, seg1.ts, seg2.ts, ...)    │
│       ├── 360p   → segments (seg0.ts, seg1.ts, seg2.ts, ...)    │
│       ├── 480p   → segments (seg0.ts, seg1.ts, seg2.ts, ...)    │
│       ├── 720p   → segments (seg0.ts, seg1.ts, seg2.ts, ...)    │
│       ├── 1080p  → segments (seg0.ts, seg1.ts, seg2.ts, ...)    │
│       └── master.m3u8 (manifest listing all quality levels)      │
│                                                                   │
└──────────────── uploaded to CDN ─────────────────────────────────┘

┌──── Frontend (runtime) ──────────────────────────────────────────┐
│                                                                   │
│  1. Player fetches master.m3u8 manifest                          │
│  2. Parses available quality levels and their bandwidth targets  │
│  3. Estimates current network bandwidth                          │
│  4. Selects the highest quality that fits within bandwidth       │
│  5. Fetches segment 0 of that quality → starts playback          │
│  6. Continuously monitors download speed                         │
│  7. If bandwidth drops → switches DOWN to lower quality segment  │
│  8. If bandwidth improves → switches UP to higher quality        │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

##### HLS Master Manifest Example (master.m3u8)

```
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=400000,RESOLUTION=426x240
quality/240p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
quality/360p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1400000,RESOLUTION=854x480
quality/480p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1280x720
quality/720p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
quality/1080p/playlist.m3u8
```

##### HLS vs DASH Comparison

| Feature | HLS (HTTP Live Streaming) | DASH (Dynamic Adaptive Streaming over HTTP) |
|---------|---------------------------|---------------------------------------------|
| **Origin** | Apple | MPEG consortium (open standard) |
| **Manifest format** | .m3u8 (playlist text) | .mpd (XML) |
| **Segment format** | .ts (MPEG-TS) or .fmp4 | .m4s (fragmented MP4) |
| **Native browser support** | Safari, iOS, macOS | None natively (needs dash.js) |
| **JS library needed** | hls.js (for non-Safari) | dash.js or shaka-player |
| **DRM support** | FairPlay (Safari), Widevine via EME | Widevine, PlayReady via EME |
| **Industry adoption** | Dominant (YouTube, Netflix, Twitch all use HLS or hybrid) | Used by YouTube (alongside HLS), Netflix |

##### Implementation with hls.js

```tsx
import Hls from 'hls.js';
import { useEffect, useRef, useState } from 'react';

function useAdaptivePlayer(manifestUrl: string) {
  const videoRef = useRef<HTMLVideoElement>(null);
  const hlsRef = useRef<Hls | null>(null);
  const [qualityLevels, setQualityLevels] = useState<QualityLevel[]>([]);
  const [currentLevel, setCurrentLevel] = useState<number>(-1); // -1 = auto

  useEffect(() => {
    const video = videoRef.current;
    if (!video || !manifestUrl) return;

    // Safari supports HLS natively via <video src="...m3u8">
    if (video.canPlayType('application/vnd.apple.mpegurl')) {
      video.src = manifestUrl;
      return;
    }

    // All other browsers: use hls.js
    if (Hls.isSupported()) {
      const hls = new Hls({
        startLevel: -1,                    // auto-select initial quality
        capLevelToPlayerSize: true,        // don't fetch 4K for a 360px player
        maxBufferLength: 30,               // buffer up to 30s ahead
        maxMaxBufferLength: 60,            // hard cap at 60s
        lowLatencyMode: false,             // not live, no need for low-latency
      });

      hls.loadSource(manifestUrl);
      hls.attachMedia(video);

      hls.on(Hls.Events.MANIFEST_PARSED, (_event, data) => {
        setQualityLevels(
          data.levels.map((level, index) => ({
            index,
            height: level.height,
            bitrate: level.bitrate,
            label: `${level.height}p`,
          }))
        );
      });

      hls.on(Hls.Events.LEVEL_SWITCHED, (_event, data) => {
        setCurrentLevel(data.level);
      });

      hlsRef.current = hls;

      return () => {
        hls.destroy();
        hlsRef.current = null;
      };
    }
  }, [manifestUrl]);

  const setQuality = (levelIndex: number) => {
    if (hlsRef.current) {
      // -1 = auto, 0+ = fixed quality
      hlsRef.current.currentLevel = levelIndex;
    }
  };

  return { videoRef, qualityLevels, currentLevel, setQuality };
}
```

##### Why `capLevelToPlayerSize: true`?

If the video player element is 640px wide, there is no visual benefit in downloading 1080p or 4K video — the extra pixels are wasted. This option tells hls.js to cap the quality level to the player's physical dimensions, saving bandwidth significantly on smaller viewports (mobile, embedded players, mini-player mode).

---

#### 5.2.2 Video Player State Machine

The video player has a complex set of states that must be managed carefully to avoid glitchy behavior:

```
                    ┌──────────┐
                    │  IDLE    │  (no video loaded yet)
                    └────┬─────┘
                         │ loadSource()
                         ▼
                    ┌──────────┐
                    │ LOADING  │  (fetching manifest + first segments)
                    └────┬─────┘
                         │ canplay event
                         ▼
                    ┌──────────┐
            ┌──────│ READY    │  (first frame available, not playing yet)
            │      └────┬─────┘
            │           │ play()
            │           ▼
            │      ┌──────────┐     seek()    ┌──────────┐
            │      │ PLAYING  │──────────────→│ SEEKING  │
            │      └────┬─────┘              └────┬─────┘
            │           │ │                       │ seeked event
            │    pause()│ │ waiting event         │
            │           │ ▼                       ▼
            │      ┌──────────┐              ┌──────────┐
            │      │ PAUSED   │              │ PLAYING  │
            │      └──────────┘              └──────────┘
            │           │
            │    ended event
            │           ▼
            │      ┌──────────┐
            │      │ ENDED    │  (reached video end)
            │      └──────────┘
            │
            │      ┌──────────┐
            └─────→│ ERROR    │  (network failure, decode error, DRM error)
                   └──────────┘
```

##### State Machine Implementation

```tsx
type PlayerState = 'idle' | 'loading' | 'ready' | 'playing' | 'paused'
  | 'buffering' | 'seeking' | 'ended' | 'error';

interface PlayerStore {
  state: PlayerState;
  currentTime: number;
  duration: number;
  buffered: number;         // buffered ahead in seconds
  volume: number;
  isMuted: boolean;
  isFullscreen: boolean;
  playbackRate: number;
  qualityLevel: number;     // -1 = auto
  captionsEnabled: boolean;
  error: string | null;
}

function usePlayerState(videoRef: React.RefObject<HTMLVideoElement>) {
  const [store, setStore] = useState<PlayerStore>({
    state: 'idle',
    currentTime: 0,
    duration: 0,
    buffered: 0,
    volume: 1,
    isMuted: false,
    isFullscreen: false,
    playbackRate: 1,
    qualityLevel: -1,
    captionsEnabled: false,
    error: null,
  });

  useEffect(() => {
    const video = videoRef.current;
    if (!video) return;

    const handlers: Record<string, () => void> = {
      loadstart: () => setStore((s) => ({ ...s, state: 'loading' })),
      canplay: () => setStore((s) => ({
        ...s,
        state: s.state === 'loading' ? 'ready' : s.state,
        duration: video.duration,
      })),
      play: () => setStore((s) => ({ ...s, state: 'playing' })),
      pause: () => setStore((s) => ({
        ...s,
        state: video.ended ? 'ended' : 'paused',
      })),
      waiting: () => setStore((s) => ({ ...s, state: 'buffering' })),
      playing: () => setStore((s) => ({ ...s, state: 'playing' })),
      seeking: () => setStore((s) => ({ ...s, state: 'seeking' })),
      seeked: () => setStore((s) => ({
        ...s,
        state: video.paused ? 'paused' : 'playing',
      })),
      ended: () => setStore((s) => ({ ...s, state: 'ended' })),
      timeupdate: () => setStore((s) => ({
        ...s,
        currentTime: video.currentTime,
        buffered: video.buffered.length > 0
          ? video.buffered.end(video.buffered.length - 1)
          : 0,
      })),
      volumechange: () => setStore((s) => ({
        ...s,
        volume: video.volume,
        isMuted: video.muted,
      })),
      error: () => setStore((s) => ({
        ...s,
        state: 'error',
        error: video.error?.message || 'Playback error',
      })),
    };

    Object.entries(handlers).forEach(([event, handler]) => {
      video.addEventListener(event, handler);
    });

    return () => {
      Object.entries(handlers).forEach(([event, handler]) => {
        video.removeEventListener(event, handler);
      });
    };
  }, [videoRef]);

  return store;
}
```

---

#### 5.2.3 Custom Controls Overlay

YouTube (and all serious video platforms) replace the browser's default video controls with a fully custom UI. Reasons:
*   Default controls differ across browsers (Chrome vs Safari vs Firefox) — inconsistent UX.
*   No support for quality selection, playback speed, captions, PiP, theater mode, or seek preview thumbnails.
*   Limited styling capabilities — cannot match brand design.
*   No keyboard shortcut customization.

##### Controls Architecture

```tsx
function VideoPlayer({ manifestUrl, videoId }: Props) {
  const { videoRef, qualityLevels, currentLevel, setQuality } =
    useAdaptivePlayer(manifestUrl);
  const playerState = usePlayerState(videoRef);
  const [controlsVisible, setControlsVisible] = useState(true);
  const hideTimerRef = useRef<ReturnType<typeof setTimeout>>();

  // Auto-hide controls after 3s of inactivity
  const showControls = useCallback(() => {
    setControlsVisible(true);
    clearTimeout(hideTimerRef.current);
    hideTimerRef.current = setTimeout(() => {
      if (playerState.state === 'playing') {
        setControlsVisible(false);
      }
    }, 3000);
  }, [playerState.state]);

  // Keyboard shortcuts
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      const video = videoRef.current;
      if (!video) return;

      // Ignore if user is typing in an input field
      if (
        e.target instanceof HTMLInputElement ||
        e.target instanceof HTMLTextAreaElement
      ) return;

      switch (e.key) {
        case ' ':
        case 'k':
          e.preventDefault();
          video.paused ? video.play() : video.pause();
          break;
        case 'ArrowLeft':
          e.preventDefault();
          video.currentTime = Math.max(0, video.currentTime - 5);
          break;
        case 'ArrowRight':
          e.preventDefault();
          video.currentTime = Math.min(video.duration, video.currentTime + 5);
          break;
        case 'ArrowUp':
          e.preventDefault();
          video.volume = Math.min(1, video.volume + 0.1);
          break;
        case 'ArrowDown':
          e.preventDefault();
          video.volume = Math.max(0, video.volume - 0.1);
          break;
        case 'f':
          document.fullscreenElement
            ? document.exitFullscreen()
            : videoRef.current?.requestFullscreen();
          break;
        case 'm':
          video.muted = !video.muted;
          break;
        case 'c':
          // Toggle captions — handled by captions layer
          break;
        case 'j':
          video.currentTime = Math.max(0, video.currentTime - 10);
          break;
        case 'l':
          video.currentTime = Math.min(video.duration, video.currentTime + 10);
          break;
      }
      showControls();
    };

    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [videoRef, showControls]);

  return (
    <div
      className="video-player-container"
      onMouseMove={showControls}
      onTouchStart={showControls}
      style={{ cursor: controlsVisible ? 'default' : 'none' }}
    >
      <video
        ref={videoRef}
        className="video-element"
        playsInline
        // Remove default controls
      />

      {/* Captions overlay */}
      <CaptionsRenderer videoRef={videoRef} enabled={playerState.captionsEnabled} />

      {/* Buffering spinner */}
      {playerState.state === 'buffering' && <BufferingSpinner />}

      {/* Click to play/pause */}
      <div
        className="click-layer"
        onClick={() => {
          const video = videoRef.current;
          if (video) video.paused ? video.play() : video.pause();
        }}
      />

      {/* Controls bar */}
      <ControlsOverlay
        visible={controlsVisible}
        playerState={playerState}
        videoRef={videoRef}
        qualityLevels={qualityLevels}
        currentLevel={currentLevel}
        onQualityChange={setQuality}
      />
    </div>
  );
}
```

---

#### 5.2.4 Buffering and Preload Strategy

##### How Much to Buffer Ahead

| Strategy | Buffer ahead | Trade-off |
|----------|-------------|-----------|
| **Minimal (10s)** | 10 seconds of video | Low bandwidth usage; risk of stalls on variable networks |
| **Moderate (30s)** | 30 seconds | Good balance — enough buffer for network hiccups; reasonable bandwidth |
| **Aggressive (60s+)** | 60+ seconds | Almost stall-proof; wastes bandwidth if user leaves early or seeks past buffered region |
| **YouTube's approach** | ~30s playing, pauses buffering when paused (after ~1min buffer is built) | Saves bandwidth when user pauses; resumes buffering when play resumes |

##### Pause Buffer Strategy

When the user pauses the video, YouTube stops buffering after a small lead buffer is filled. This saves bandwidth because users frequently pause and never return. Implementation:

```tsx
function useSmartBuffering(videoRef: React.RefObject<HTMLVideoElement>, hlsRef: React.RefObject<Hls | null>) {
  useEffect(() => {
    const video = videoRef.current;
    const hls = hlsRef.current;
    if (!video || !hls) return;

    let pauseBufferTimer: ReturnType<typeof setTimeout>;

    const onPause = () => {
      // When paused, allow buffering for 10 more seconds of content,
      // then stop fetching new segments
      pauseBufferTimer = setTimeout(() => {
        // Reduce max buffer to what we already have
        hls.config.maxBufferLength = 10;
      }, 5000); // wait 5s after pause before throttling
    };

    const onPlay = () => {
      clearTimeout(pauseBufferTimer);
      // Restore normal buffer size on play
      hls.config.maxBufferLength = 30;
    };

    video.addEventListener('pause', onPause);
    video.addEventListener('play', onPlay);

    return () => {
      clearTimeout(pauseBufferTimer);
      video.removeEventListener('pause', onPause);
      video.removeEventListener('play', onPlay);
    };
  }, [videoRef, hlsRef]);
}
```

---

#### 5.2.5 Quality Selection and Auto Resolution

The quality selector menu shows all available resolutions with an "Auto" option as default:

```
┌──────────────────┐
│ Quality          │
│ ────────────────│
│   Auto (720p)  ✓│  ← currently selected; shows actual auto-picked level
│   2160p (4K)    │
│   1080p         │
│   720p          │
│   480p          │
│   360p          │
│   240p          │
└──────────────────┘
```

##### Auto Quality Logic

When "Auto" is selected (default), the ABR controller in hls.js/dash.js picks the quality level based on:

| Factor | How it is used |
|--------|---------------|
| **Measured bandwidth** | Download speed of the last N segments — primary signal |
| **Buffer health** | If buffer is low (< 5s), be conservative and pick lower quality to fill buffer faster |
| **Player dimensions** | If player is 640px wide, cap at 720p (no visual benefit from 1080p) |
| **Device capabilities** | Some devices cannot decode 4K; respect `MediaCapabilities` API |

When the user manually selects a quality (e.g., 1080p), the ABR controller is bypassed and all future segments are downloaded at that fixed quality regardless of bandwidth. The user can switch back to "Auto" at any time.

---

#### 5.2.6 Picture in Picture and Mini Player

Two flavors of "keep watching while browsing":

| Feature | Mechanism | When used |
|---------|-----------|-----------|
| **Browser PiP** | `video.requestPictureInPicture()` — browser-native floating window | User clicks PiP button; video floats above all windows |
| **Mini-player** | Custom fixed-position element at bottom-right of the page | User scrolls past the video on the watch page; mini-player appears |

##### Mini Player on Scroll

```tsx
function useMiniPlayer(
  playerContainerRef: React.RefObject<HTMLElement>,
  videoRef: React.RefObject<HTMLVideoElement>
) {
  const [isMiniPlayer, setIsMiniPlayer] = useState(false);

  useEffect(() => {
    const container = playerContainerRef.current;
    if (!container) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        const video = videoRef.current;
        // Show mini-player only if video is playing and player is out of view
        if (!entry.isIntersecting && video && !video.paused) {
          setIsMiniPlayer(true);
        } else {
          setIsMiniPlayer(false);
        }
      },
      { threshold: 0.5 } // trigger when less than 50% visible
    );

    observer.observe(container);
    return () => observer.disconnect();
  }, [playerContainerRef, videoRef]);

  return isMiniPlayer;
}
```

When `isMiniPlayer` is true, the player renders as a fixed-position element:

```css
.mini-player {
  position: fixed;
  bottom: 24px;
  right: 24px;
  width: 400px;
  aspect-ratio: 16 / 9;
  z-index: 1000;
  border-radius: 12px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.3);
  transition: all 0.3s ease;
}
```

---

#### 5.2.7 Keyboard Shortcuts and Accessibility

YouTube's keyboard shortcuts must be discoverable and accessible:

| Key | Action | Notes |
|-----|--------|-------|
| `Space` / `K` | Play / Pause | Space only when player is focused (not in input) |
| `J` | Rewind 10s | |
| `L` | Forward 10s | |
| `←` | Rewind 5s | |
| `→` | Forward 5s | |
| `↑` / `↓` | Volume up / down | 10% increments |
| `F` | Toggle fullscreen | |
| `M` | Toggle mute | |
| `C` | Toggle captions | |
| `0-9` | Seek to 0%-90% | Number keys as percentage of duration |
| `,` / `.` | Frame back / forward | When paused, step through frames |
| `>` / `<` | Increase / decrease speed | |
| `Escape` | Exit fullscreen | Browser default |

##### ARIA Requirements for the Player

```html
<div role="region" aria-label="Video player" tabindex="-1">
  <video aria-label="Video: How to Build a React App" />

  <div role="toolbar" aria-label="Player controls">
    <button aria-label="Play" aria-pressed="false">▶</button>

    <input
      type="range"
      role="slider"
      aria-label="Seek"
      aria-valuemin="0"
      aria-valuemax="756"
      aria-valuenow="134"
      aria-valuetext="2 minutes 14 seconds of 12 minutes 36 seconds"
    />

    <button aria-label="Mute" aria-pressed="false">🔊</button>

    <input
      type="range"
      role="slider"
      aria-label="Volume"
      aria-valuemin="0"
      aria-valuemax="100"
      aria-valuenow="80"
    />

    <button aria-label="Captions" aria-pressed="false">CC</button>
    <button aria-label="Fullscreen">⛶</button>
  </div>
</div>
```

---

#### 5.2.8 Decision Matrix

| Decision | Options | Chosen | Rationale |
|----------|---------|--------|-----------|
| **Streaming protocol** | HLS vs DASH vs Progressive MP4 | HLS (with hls.js for non-Safari) | Widest device support; Safari native; hls.js is mature and lightweight (~60KB gzipped) |
| **Player library** | Custom + hls.js vs shaka-player vs video.js | Custom + hls.js | Minimal bundle; full control over UI; shaka-player adds DRM but is larger |
| **Controls** | Native browser vs Custom overlay | Custom overlay | Consistent UX across browsers; YouTube-like features (quality selector, speed, PiP) not available natively |
| **Buffering strategy** | Aggressive vs Moderate vs Minimal | Moderate (30s) with pause-aware throttling | Balances stall prevention with bandwidth savings |
| **Quality default** | Auto (ABR) vs Fixed | Auto with `capLevelToPlayerSize` | Best experience for majority; manual override available |
| **Mini player** | Browser PiP only vs Custom mini-player | Both — custom mini-player on scroll + PiP button | Mini-player keeps user in the app; PiP works across tabs |
| **Captions** | Burned-in vs WebVTT overlay | WebVTT overlay | User can toggle; style customization; multi-language support |

---

### 5.3 Reusability Strategy

*   **VideoCard** component: Reused across home feed, search results, recommendations sidebar, channel page, and playlist views. Accepts `variant` prop for layout (grid, horizontal compact, list).
*   **VideoPlayer**: Standalone module that accepts a manifest URL and video metadata. Reusable for main watch page, embedded player, and mini-player.
*   **EngagementBar**: Reused for videos and community posts (like, dislike, share, save).
*   **CommentSection**: Generic threaded comment component reusable across videos and community posts.
*   **InfiniteScrollContainer**: Shared sentinel-based container used for home feed, search results, and comment lists.
*   **Design System**: Button, Menu, Modal, Toast, Avatar, Badge components from a shared library.

---

### 5.4 Module Organization

```
src/
 ├── pages/
 │    ├── HomePage/
 │    │    ├── HomePage.tsx
 │    │    ├── CategoryChips.tsx
 │    │    └── VideoGrid.tsx
 │    ├── WatchPage/
 │    │    ├── WatchPage.tsx
 │    │    ├── VideoMetadata.tsx
 │    │    ├── EngagementBar.tsx
 │    │    ├── ChannelInfo.tsx
 │    │    └── RecommendedVideos.tsx
 │    ├── SearchPage/
 │    └── ChannelPage/
 │
 ├── features/
 │    ├── player/
 │    │    ├── VideoPlayer.tsx
 │    │    ├── ControlsOverlay.tsx
 │    │    ├── ProgressBar.tsx
 │    │    ├── QualitySelector.tsx
 │    │    ├── CaptionsRenderer.tsx
 │    │    ├── MiniPlayer.tsx
 │    │    ├── useAdaptivePlayer.ts
 │    │    ├── usePlayerState.ts
 │    │    ├── useMiniPlayer.ts
 │    │    └── useSmartBuffering.ts
 │    ├── comments/
 │    │    ├── CommentsSection.tsx
 │    │    ├── CommentItem.tsx
 │    │    ├── CreateComment.tsx
 │    │    └── RepliesThread.tsx
 │    ├── search/
 │    │    ├── SearchBar.tsx
 │    │    └── SuggestionsDropdown.tsx
 │    └── feed/
 │         ├── VideoCard.tsx
 │         ├── ThumbnailPreview.tsx
 │         └── FeedSentinel.tsx
 │
 ├── shared/
 │    ├── components/   (Button, Modal, Menu, Avatar, Toast, Badge)
 │    ├── hooks/        (useIntersectionObserver, useDebounce, useMediaQuery)
 │    ├── utils/        (formatNumber, formatDuration, formatTimeAgo)
 │    └── api/          (apiClient, requestDeduplicator, retryLogic)
 │
 └── store/
      ├── playerStore.ts
      ├── feedStore.ts
      ├── commentsStore.ts
      ├── userStore.ts
      └── watchHistoryStore.ts
```

---

## 6. High Level Data Flow Explanation

### 6.1 Initial Load Flow

```
User navigates to /watch?v=abc123
        │
        ▼
┌─── Server (SSR) ──────────────────────────────────────┐
│  1. Fetch video metadata (title, description, channel) │
│  2. Render HTML shell with metadata (SEO + fast FCP)   │
│  3. Include <link rel="preconnect"> to CDN              │
│  4. Include <link rel="preload"> for manifest URL       │
│  5. Send HTML to browser                                │
└────────────────────────────────────────────────────────┘
        │
        ▼
┌─── Browser ────────────────────────────────────────────┐
│  1. Parse HTML → paint metadata (title, channel info)   │
│  2. Load JS bundle → hydrate React app                  │
│  3. VideoPlayer mounts:                                 │
│     a. Fetch HLS master manifest (master.m3u8)          │
│     b. Parse quality levels                             │
│     c. Select initial quality (based on bandwidth est.) │
│     d. Fetch first 2-3 video segments                   │
│     e. Decode and paint first frame → time-to-first-frame│
│     f. Start playback (autoplay if allowed by browser)  │
│  4. Lazy load comments section (below the fold)         │
│  5. Fetch recommended videos for sidebar                │
│  6. Report watch session start to analytics             │
└────────────────────────────────────────────────────────┘
```

**Why preload the manifest?**
The HLS manifest is a small text file (~1KB) but is on the critical path to playback. By including `<link rel="preload" href="manifest.m3u8" as="fetch">` in the SSR HTML, the browser starts downloading it immediately — even before JS is parsed and the player component mounts. This shaves 100-300ms off time-to-first-frame.

---

### 6.2 User Interaction Flow

| Action | Frontend Flow |
|--------|--------------|
| **Click play** | `video.play()` → state changes to `playing` → controls update → analytics event "play" |
| **Seek (drag progress bar)** | Update preview thumbnail on hover → on release: `video.currentTime = seekTarget` → state `seeking` → segment fetched → `seeked` event → state `playing` |
| **Change quality** | `hls.currentLevel = selectedLevel` → hls.js fetches next segment at new quality → fade transition between qualities → state updates |
| **Like video** | Optimistic: UI immediately shows liked state → POST /api/videos/:id/like → on success: confirmed; on failure: revert UI + show toast |
| **Post comment** | Optimistic: comment appears immediately at top → POST /api/videos/:id/comments → on success: update with server-assigned ID; on failure: show retry option |
| **Subscribe** | Optimistic: button changes to "Subscribed" → POST /api/channels/:id/subscribe → confirmed or reverted |
| **Toggle captions** | Enable WebVTT track → render caption text overlay → persist preference to localStorage |
| **Enter fullscreen** | `playerContainer.requestFullscreen()` → CSS adapts layout → controls reflow → `fullscreenchange` event updates state |

---

### 6.3 Error and Retry Flow

| Error Type | Detection | Recovery |
|------------|-----------|----------|
| **Manifest fetch failure** | hls.js `ERROR` event with type `NETWORK_ERROR` | Retry with exponential backoff (1s, 2s, 4s); after 3 retries show error UI with "Tap to retry" |
| **Segment fetch failure** | hls.js `FRAG_LOAD_ERROR` | hls.js auto-retries; if persistent, drop to lower quality; if all qualities fail, show error |
| **Decode error** | `video.error` with `MEDIA_ERR_DECODE` | Try lower quality; if all fail, suggest different browser |
| **DRM license failure** | EME `keystatuseschange` with `output-restricted` | Show "Content not available on this device" message |
| **API failure (like, comment)** | HTTP 4xx/5xx response | Revert optimistic update; show toast with retry action |
| **Network offline** | `navigator.onLine` + `offline` event | Pause playback; show "You are offline" banner; resume when back online |

```tsx
function usePlaybackErrorRecovery(
  hlsRef: React.RefObject<Hls | null>,
  videoRef: React.RefObject<HTMLVideoElement>
) {
  useEffect(() => {
    const hls = hlsRef.current;
    if (!hls) return;

    const handleError = (_event: string, data: Hls.errorData) => {
      if (!data.fatal) return; // non-fatal errors are auto-recovered by hls.js

      switch (data.type) {
        case Hls.ErrorTypes.NETWORK_ERROR:
          // Try to recover by restarting the load
          hls.startLoad();
          break;
        case Hls.ErrorTypes.MEDIA_ERROR:
          // Try to recover from media errors
          hls.recoverMediaError();
          break;
        default:
          // Unrecoverable error — destroy and show error UI
          hls.destroy();
          break;
      }
    };

    hls.on(Hls.Events.ERROR, handleError);
    return () => hls.off(Hls.Events.ERROR, handleError);
  }, [hlsRef, videoRef]);
}
```

---

## 7. Data Modelling (Frontend Perspective)

### 7.1 Core Data Entities

*   **Video**: The primary content entity — metadata, streaming info, engagement counts.
*   **Channel**: The creator/publisher of videos — name, avatar, subscriber count.
*   **Comment**: User-generated text attached to a video, with threading support.
*   **User**: The authenticated viewer — preferences, watch history, subscriptions.
*   **Playlist**: An ordered collection of videos created by a user or system-generated.
*   **Caption**: Subtitle/caption track associated with a video.

---

### 7.2 Data Shape

```ts
interface Video {
  id: string;
  title: string;
  description: string;
  thumbnailUrl: string;          // CDN URL for thumbnail
  previewUrl: string | null;     // short silent preview clip for hover
  manifestUrl: string;           // HLS/DASH manifest URL
  duration: number;              // total duration in seconds
  viewCount: number;
  likeCount: number;
  dislikeCount: number;
  commentCount: number;
  publishedAt: string;           // ISO 8601
  channel: ChannelSummary;
  tags: string[];
  category: string;
  isLive: boolean;
  captions: CaptionTrack[];
  watchProgress: number | null;  // seconds watched (for "continue watching")
  isLiked: boolean | null;       // null = not voted, true = liked, false = disliked
}

interface ChannelSummary {
  id: string;
  name: string;
  avatarUrl: string;
  subscriberCount: number;
  isSubscribed: boolean;
}

interface Channel extends ChannelSummary {
  bannerUrl: string;
  description: string;
  videoCount: number;
  joinedAt: string;
  links: { label: string; url: string }[];
}

interface Comment {
  id: string;
  videoId: string;
  author: {
    id: string;
    name: string;
    avatarUrl: string;
    isChannelOwner: boolean;    // highlighted if video creator replies
  };
  text: string;
  likeCount: number;
  isLiked: boolean;
  replyCount: number;
  createdAt: string;
  isEdited: boolean;
  parentId: string | null;      // null = top-level comment
}

interface CaptionTrack {
  language: string;             // "en", "es", "hi"
  label: string;                // "English", "Spanish"
  url: string;                  // WebVTT file URL
  isAutoGenerated: boolean;
}

interface VideoCard {
  id: string;
  title: string;
  thumbnailUrl: string;
  duration: number;
  viewCount: number;
  publishedAt: string;
  channel: {
    id: string;
    name: string;
    avatarUrl: string;
  };
  watchProgress: number | null;
}

interface Playlist {
  id: string;
  title: string;
  thumbnailUrl: string;
  videoCount: number;
  visibility: 'public' | 'unlisted' | 'private';
  updatedAt: string;
}
```

---

### 7.3 Entity Relationships

```
Channel  ──(1:many)──→  Video
Video    ──(1:many)──→  Comment
Comment  ──(1:many)──→  Comment (replies via parentId)
Video    ──(1:many)──→  CaptionTrack
User     ──(many:many)──→  Channel (subscriptions)
User     ──(many:many)──→  Video (watch history, likes)
Playlist ──(many:many)──→  Video (ordered list)
```

*   **Normalized storage**: Videos, channels, and comments are stored in separate normalized maps in the store. References use IDs.
*   **Denormalized in API responses**: API returns embedded `channel` data within `Video` to avoid N+1 fetches for the feed. Frontend can extract and normalize on receipt.

---

### 7.4 UI Specific Data Models

```ts
// Derived state for the video player
interface PlayerUIState {
  formattedCurrentTime: string;        // "2:34"
  formattedDuration: string;           // "12:56"
  progressPercent: number;             // 0-100 for progress bar width
  bufferedPercent: number;             // 0-100 for buffer indicator
  qualityLabel: string;               // "720p" or "Auto (720p)"
  isControlsVisible: boolean;
  isMiniPlayer: boolean;
}

// Derived state for video card in feed
interface VideoCardUIState {
  formattedViewCount: string;          // "1.2M views"
  formattedPublishedAt: string;        // "3 days ago"
  formattedDuration: string;           // "12:34"
  watchProgressPercent: number | null; // red bar on thumbnail
}

// Aggregated comment section state
interface CommentSectionUIState {
  formattedCommentCount: string;       // "1.2K comments"
  sortOrder: 'top' | 'newest';
  isLoadingMore: boolean;
  hasMoreComments: boolean;
}
```

---

## 8. State Management Strategy

### 8.1 State Classification

| State Type | What it contains | Tool |
|------------|-----------------|------|
| **Server state** | Video metadata, comments, recommendations, search results, channel data | React Query / TanStack Query (cache, refetch, optimistic updates) |
| **Player state** | Current time, duration, buffered, playing/paused, volume, quality level, fullscreen, captions | Local component state via `usePlayerState` hook (high-frequency updates) |
| **Global app state** | Current user session, auth token, theme preference, sidebar collapsed state | Zustand or React Context |
| **Feature state** | Comments sort order, search filters, category chip selection | Component-local or feature-level Zustand slice |
| **Persisted state** | Watch history, playback position per video, volume preference, caption preference, playback speed | localStorage synced to store |

---

### 8.2 State Ownership

| State | Owner | Why |
|-------|-------|-----|
| **Video metadata** | React Query cache (keyed by videoId) | Server-owned data; cache invalidated on refetch |
| **Player playback state** | `usePlayerState` hook (local to VideoPlayer) | High-frequency time updates should NOT propagate to global store — would cause unnecessary re-renders |
| **Like/Dislike state** | React Query mutation with optimistic update | Server-owned but UI needs instant feedback |
| **Comments list** | React Query infinite query (keyed by videoId + sort) | Paginated server data with cursor-based fetching |
| **Subscription state** | React Query + global user store | Affects multiple UI elements (subscribe button, sidebar) |
| **Mini-player visibility** | `useMiniPlayer` hook (local) | Derived from IntersectionObserver — no need in global store |
| **Watch progress** | localStorage + debounced API sync | Persisted locally for instant "continue watching"; synced to server periodically |

##### Why Player State is Kept Local

The video `timeupdate` event fires ~4 times per second. If player state lived in a global store (Redux, Zustand), every 250ms would trigger:
1.  Store update
2.  Selector recalculation for all subscribers
3.  Re-renders in any component consuming player state

By keeping player state in the `VideoPlayer` component tree, only the progress bar and time display re-render on `timeupdate`. The rest of the page is unaffected.

---

### 8.3 Persistence Strategy

| Data | Storage | Sync Strategy |
|------|---------|---------------|
| **Watch progress per video** | `localStorage: { [videoId]: seconds }` | Write on `timeupdate` (throttled to every 5s); sync to server every 30s |
| **Volume and mute** | `localStorage: volume` | Write on `volumechange` |
| **Playback speed** | `localStorage: playbackRate` | Write on change; applied to next video |
| **Caption preference** | `localStorage: captionsEnabled` and `preferredCaptionLang` | Write on toggle; auto-enable on next video |
| **Search history** | `localStorage: searchHistory[]` (capped at 20 entries) | Write on search submit; used for suggestions |
| **Theme preference** | `localStorage: theme` | Write on toggle; applied on app load before render (avoid flash) |

```tsx
// Throttled watch progress persistence
function useWatchProgressSync(videoId: string, videoRef: React.RefObject<HTMLVideoElement>) {
  const lastSyncRef = useRef<number>(0);

  useEffect(() => {
    const video = videoRef.current;
    if (!video) return;

    const handleTimeUpdate = () => {
      const now = Date.now();
      // Throttle localStorage writes to every 5 seconds
      if (now - lastSyncRef.current < 5000) return;
      lastSyncRef.current = now;

      const progress = Math.floor(video.currentTime);
      localStorage.setItem(`wp_${videoId}`, String(progress));
    };

    video.addEventListener('timeupdate', handleTimeUpdate);
    return () => video.removeEventListener('timeupdate', handleTimeUpdate);
  }, [videoId, videoRef]);

  // Restore on mount
  useEffect(() => {
    const video = videoRef.current;
    const saved = localStorage.getItem(`wp_${videoId}`);
    if (video && saved) {
      const seconds = parseInt(saved, 10);
      // Only resume if not near the end (within last 10s = treat as watched)
      if (seconds > 5 && seconds < video.duration - 10) {
        video.currentTime = seconds;
      }
    }
  }, [videoId, videoRef]);
}
```

---

## 9. High Level API Design (Frontend POV)

### 9.1 Required APIs

| API | Method | Purpose |
|-----|--------|---------|
| `GET /api/videos/:id` | GET | Fetch video metadata, manifest URL, channel info, caption tracks |
| `GET /api/videos/:id/recommendations` | GET | Fetch recommended videos for sidebar |
| `GET /api/videos/:id/comments` | GET | Fetch paginated comments (cursor-based) |
| `POST /api/videos/:id/comments` | POST | Post a new comment |
| `POST /api/videos/:id/like` | POST | Like a video |
| `POST /api/videos/:id/dislike` | POST | Dislike a video |
| `DELETE /api/videos/:id/reaction` | DELETE | Remove like/dislike |
| `GET /api/feed` | GET | Fetch personalized home feed (cursor-based) |
| `GET /api/search` | GET | Search videos with filters |
| `GET /api/search/suggestions` | GET | Typeahead suggestions |
| `GET /api/channels/:id` | GET | Fetch channel profile |
| `POST /api/channels/:id/subscribe` | POST | Subscribe to channel |
| `DELETE /api/channels/:id/subscribe` | DELETE | Unsubscribe |
| `GET /api/history` | GET | Watch history |
| `PUT /api/history/:videoId` | PUT | Update watch progress |

---

### 9.2 Adaptive Streaming Manifest Fetching Strategy (Deep Dive)

The video manifest URL is the most critical resource for playback start. How we fetch it determines time-to-first-frame.

##### Three Phase Strategy

```
Phase 1: SSR Preload (during server rendering)
────────────────────────────────────────────
  Server embeds: <link rel="preload" href="https://cdn.example.com/v/abc123/master.m3u8" as="fetch" crossorigin>
  → Browser starts downloading manifest immediately, in parallel with JS bundle

Phase 2: Player Mount (after JS hydration)
────────────────────────────────────────────
  VideoPlayer mounts → hls.loadSource(manifestUrl)
  → If preload completed, manifest is served from browser cache (0ms fetch)
  → If still loading, hls.js waits for the in-flight request

Phase 3: Segment Fetching (after manifest parsed)
────────────────────────────────────────────
  hls.js parses manifest → selects quality → fetches first 2 segments
  → First segment decodes → first frame painted → playback starts
```

##### Why Preloading Works

| Without preload | With preload |
|----------------|-------------|
| HTML loads → JS loads → React hydrates → Player mounts → **manifest starts downloading** → segments download → playback | HTML loads → **manifest starts downloading** (parallel) → JS loads → React hydrates → Player mounts → manifest already cached → segments download → playback |
| Manifest fetch blocked behind JS execution (~500ms-1s delay) | Manifest fetch overlaps with JS loading (saves 200-500ms) |

---

### 9.3 Request and Response Structure

##### Get Video Details

```json
// GET /api/videos/abc123

// Response:
{
  "data": {
    "id": "abc123",
    "title": "How to Build a React App in 2025",
    "description": "In this video we cover...",
    "thumbnailUrl": "https://cdn.example.com/thumbs/abc123.webp",
    "manifestUrl": "https://cdn.example.com/v/abc123/master.m3u8",
    "duration": 756,
    "viewCount": 1245000,
    "likeCount": 45200,
    "dislikeCount": 890,
    "commentCount": 3400,
    "publishedAt": "2025-01-15T10:30:00Z",
    "channel": {
      "id": "ch_xyz",
      "name": "React Mastery",
      "avatarUrl": "https://cdn.example.com/avatars/ch_xyz.webp",
      "subscriberCount": 850000,
      "isSubscribed": true
    },
    "tags": ["react", "tutorial", "frontend"],
    "category": "Education",
    "captions": [
      {
        "language": "en",
        "label": "English",
        "url": "https://cdn.example.com/captions/abc123_en.vtt",
        "isAutoGenerated": false
      },
      {
        "language": "hi",
        "label": "Hindi",
        "url": "https://cdn.example.com/captions/abc123_hi.vtt",
        "isAutoGenerated": true
      }
    ],
    "watchProgress": 245,
    "isLiked": true
  }
}
```

##### Get Home Feed (Cursor Based Pagination)

```json
// GET /api/feed?cursor=eyJ0IjoiMjAyNS0wMS0xNSJ9&limit=20&category=trending

// Response:
{
  "data": [
    {
      "id": "vid_001",
      "title": "10 CSS Tricks You Didn't Know",
      "thumbnailUrl": "https://cdn.example.com/thumbs/vid_001.webp",
      "duration": 480,
      "viewCount": 520000,
      "publishedAt": "2025-01-14T08:00:00Z",
      "channel": {
        "id": "ch_css",
        "name": "CSS Weekly",
        "avatarUrl": "https://cdn.example.com/avatars/ch_css.webp"
      },
      "watchProgress": null
    }
  ],
  "pagination": {
    "nextCursor": "eyJ0IjoiMjAyNS0wMS0xNCJ9",
    "hasMore": true
  }
}
```

##### Get Comments (Cursor Based)

```json
// GET /api/videos/abc123/comments?sort=top&cursor=cmnt_page2&limit=20

// Response:
{
  "data": [
    {
      "id": "cmnt_001",
      "author": {
        "id": "user_42",
        "name": "DevFan",
        "avatarUrl": "https://cdn.example.com/avatars/user_42.webp",
        "isChannelOwner": false
      },
      "text": "Great tutorial! The hooks explanation was really clear.",
      "likeCount": 234,
      "isLiked": false,
      "replyCount": 12,
      "createdAt": "2025-01-15T12:30:00Z",
      "isEdited": false,
      "parentId": null
    }
  ],
  "pagination": {
    "nextCursor": "cmnt_page3",
    "hasMore": true
  }
}
```

---

### 9.4 Error Handling and Status Codes

| Status | Meaning | Frontend Behavior |
|--------|---------|-------------------|
| `200` | Success | Render data normally |
| `400` | Bad request (invalid params) | Show inline validation error |
| `401` | Unauthorized | Redirect to login; for engagement actions show "Sign in to like/comment" prompt |
| `403` | Forbidden (age-restricted, geo-blocked) | Show "This video is not available" with reason |
| `404` | Video/channel not found | Show "This video is no longer available" page |
| `429` | Rate limited | Back off; show "Too many requests, try again later" toast |
| `500` | Server error | Show generic error with retry button |
| `503` | Service unavailable | Show cached data if available; retry with backoff |

---

## 10. Caching Strategy

### 10.1 What to Cache

| Data | Cache? | Reason |
|------|--------|--------|
| **Video metadata** | Yes (5min TTL via React Query) | Metadata rarely changes; stale data is acceptable briefly |
| **Home feed** | Yes (stale-while-revalidate) | Show cached feed instantly on revisit; refetch in background |
| **Recommendations** | Yes (per videoId, 5min TTL) | Unlikely to change during a single session |
| **Comments** | Yes (stale-while-revalidate) | Show cached; refetch for new comments |
| **Search results** | Yes (per query string, 2min TTL) | Allow instant back-navigation to search results |
| **Channel profile** | Yes (10min TTL) | Rarely changes |
| **Video segments (HLS)** | Yes (HTTP cache, long TTL) | Segments are immutable after transcoding |
| **Thumbnails** | Yes (HTTP cache, long TTL, CDN) | Static assets; cache-busted via URL when updated |
| **Watch progress** | localStorage (not HTTP cached) | Client-owned data; must persist across sessions |

### 10.2 Where to Cache

| Layer | What | How |
|-------|------|-----|
| **React Query in-memory cache** | API responses (metadata, feed, comments) | Automatic with `staleTime` and `gcTime` configuration |
| **Browser HTTP cache** | Video segments (.ts), thumbnails (.webp), caption files (.vtt) | `Cache-Control: public, max-age=31536000, immutable` for versioned assets |
| **Service Worker cache** | App shell, static assets (JS, CSS, fonts) | Precache on install; serve from cache with network fallback |
| **localStorage** | Watch progress, volume, playback speed, caption preference | Written on user interaction; read on mount |

### 10.3 Cache Invalidation

| Data | Invalidation Strategy |
|------|----------------------|
| **Video metadata** | Time-based (5min staleTime); manual refetch on like/dislike action |
| **Comments** | Refetch after posting a comment; staleTime 2min otherwise |
| **Feed** | Stale-while-revalidate; refetch on category change or pull-to-refresh |
| **Subscriptions** | Invalidate channel query and subscriber count on subscribe/unsubscribe |
| **Video segments** | Immutable — never invalidated. New transcode creates new segment URLs. |
| **Thumbnails** | URL-versioned (e.g., `thumb_v2.webp`); long HTTP cache; new URL on update |

---

## 11. CDN and Asset Optimization

### 11.1 Video Delivery

*   Video segments served from **multi-region CDN** (edge servers close to user for low latency).
*   Each segment is 2-10 seconds of video — small enough for HTTP cache and fast enough for adaptive quality switching.
*   Segments use `Cache-Control: public, max-age=31536000, immutable` — once transcoded, segment content never changes.
*   Manifest files (.m3u8) have **short TTL** (30-60s) for live streams but can be cached longer for VOD.

### 11.2 Thumbnail Optimization

| Strategy | Implementation |
|----------|----------------|
| **Format** | WebP (30% smaller than JPEG) with JPEG fallback via `<picture>` |
| **Responsive sizes** | Multiple thumbnail sizes: 120px (mini), 246px (sidebar), 360px (grid), 720px (hover preview) |
| **Lazy loading** | `loading="lazy"` for below-the-fold thumbnails; `loading="eager"` for first row |
| **Blur-up placeholder** | Inline 16px base64 LQIP in API response → blurred → full image loads |
| **CDN with auto-resize** | Use CDN image transformation (e.g., `thumbs/vid_001.webp?w=360&q=80`) |

```html
<picture>
  <source
    type="image/webp"
    srcset="thumb_360.webp 360w, thumb_720.webp 720w"
    sizes="(max-width: 600px) 360px, 720px"
  />
  <img
    src="thumb_360.jpg"
    alt="Video thumbnail: 10 CSS Tricks"
    width="360"
    height="202"
    loading="lazy"
  />
</picture>
```

### 11.3 Static Asset Optimization

*   **JS bundles**: Code-split per route; main bundle < 150KB gzipped; player chunk loaded on watch page; comments chunk lazy-loaded.
*   **CSS**: Critical CSS inlined in SSR; rest loaded async.
*   **Fonts**: `font-display: swap`; preload primary font; subset to reduce file size.
*   **Compression**: Brotli (preferred) or gzip for all text assets (HTML, JS, CSS, JSON).

---

## 12. Rendering Strategy

| Page | Strategy | Rationale |
|------|----------|-----------|
| **Watch page** | **SSR** (shell with metadata) + **CSR** (player, comments, recommendations) | SEO for shared links (title, thumbnail in OG tags); fast FCP with metadata; player requires client-side HLS/DASH |
| **Home feed** | **SSR** (first grid batch) + **CSR** (infinite scroll, hover previews) | Fast initial paint; SEO for crawlers; subsequent pages are client-fetched |
| **Search results** | **SSR** (first results) + **CSR** (filters, pagination) | SEO for search URLs; fast FCP |
| **Channel page** | **SSR** (channel info + first video tab) + **CSR** (tab switching, infinite scroll) | SEO for channel URLs |
| **Embed page** | **CSR only** | Embedded in iframes on third-party sites; no SEO needed; minimal JS bundle (player only) |

### Hydration Strategy

*   Server renders static HTML with video metadata, descriptions, and first-batch thumbnails.
*   Client hydrates React on the rendered HTML — interactive elements (player, buttons, comments) become functional.
*   **Progressive hydration**: Player hydrates first (critical); comments section hydrates on scroll (below fold); sidebar recommendations hydrate after player.
*   Use `<Suspense>` boundaries around lazy-loaded sections (comments, recommendations) to prevent blocking the player.

---

## 13. Cross Cutting Non Functional Concerns

### 13.1 Security

*   **Signed CDN URLs**: Video manifest and segment URLs include short-lived tokens (e.g., `?token=abc&expires=1710000000`). Prevents unauthorized direct access to video files.
*   **DRM**: For premium content, use Encrypted Media Extensions (EME) with Widevine (Chrome, Firefox, Android), FairPlay (Safari, iOS), or PlayReady (Edge). License keys are fetched from a license server and never exposed to JavaScript.
*   **XSS prevention**: All user-generated content (comments, descriptions, channel names) is rendered via React JSX (auto-escaped). HTML in descriptions is parsed and sanitized server-side; only safe tags (links, line breaks) are allowed.
*   **CSRF**: All mutation APIs (like, comment, subscribe) require CSRF tokens or use `SameSite=Strict` cookies.
*   **Content Security Policy**: Restrict sources for scripts, media, and frames via CSP headers. Only allow CDN domains for media.
*   **Token storage**: Auth tokens stored in HTTP-only, Secure, SameSite=Lax cookies — never in localStorage or JS-accessible cookies.

---

### 13.2 Accessibility

*   **Video player**:
    *   Full keyboard navigation: all controls focusable and operable via keyboard.
    *   ARIA roles: `role="slider"` for seek/volume with `aria-valuenow`, `aria-valuetext`.
    *   Screen reader announcements: quality change, captions toggled, error states via `aria-live` regions.
    *   Closed captions: WebVTT support; user can toggle; caption style customizable (size, background).
    *   Reduced motion: Disable auto-advancing animations; respect `prefers-reduced-motion`.
*   **Feed and navigation**:
    *   Semantic HTML: `<nav>`, `<main>`, `<article>` for proper document structure.
    *   Skip link: "Skip to main content" link that bypasses header and sidebar.
    *   Focus management: When navigating to a new page, focus moves to page heading.
    *   Video cards: `alt` text on thumbnails; duration and view count available to screen readers.
*   **Color contrast**: WCAG AA minimum (4.5:1) for all text; AAA (7:1) for critical controls.

---

### 13.3 Performance Optimization

*   **Code splitting**: Route-based splitting (home, watch, search, channel); player module loaded only on watch page; comments lazy-loaded on scroll.
*   **Bundle size budget**: Main bundle < 150KB gzipped; player chunk < 80KB; hls.js ~60KB gzipped imported only on watch page.
*   **Image lazy loading**: `loading="lazy"` on all below-fold thumbnails; eager for first visible row.
*   **Thumbnail hover preview**: Load preview video only on hover (after 500ms delay) to avoid wasted requests on accidental hovers.
*   **Debounced search**: Typeahead debounced at 300ms to limit API calls.
*   **Virtualization**: For large comment threads (1000+ comments), virtualize the comment list to keep DOM lean.
*   **Web Workers**: Offload heavy computations (e.g., comment text processing, analytics batching) to Web Workers.
*   **Media Session API**: Integrate with OS media controls (lock screen, notification bar) for playback control.
*   **Performance metrics**: Track FCP, LCP, TTI, CLS, time-to-first-frame, buffering ratio, quality switches.

```tsx
// Report time-to-first-frame metric
function useTimeToFirstFrame(videoRef: React.RefObject<HTMLVideoElement>) {
  useEffect(() => {
    const video = videoRef.current;
    if (!video) return;

    const startTime = performance.now();

    const handleFirstFrame = () => {
      const ttff = performance.now() - startTime;
      // Send to analytics
      analytics.track('time_to_first_frame', { ttff, videoId: video.dataset.videoId });
      video.removeEventListener('playing', handleFirstFrame);
    };

    video.addEventListener('playing', handleFirstFrame);
    return () => video.removeEventListener('playing', handleFirstFrame);
  }, [videoRef]);
}
```

---

### 13.4 Observability and Reliability

*   **Error boundaries**: Wrap player, comments, and recommendations in independent error boundaries. A crash in comments should not take down the player.
*   **Logging**: Capture player errors (manifest failures, segment errors, DRM issues), API errors, and client exceptions. Send to centralized logging (e.g., Sentry, Datadog).
*   **Video quality metrics**: Track per-session: average bitrate, quality switches count, buffering duration, stall count, time-to-first-frame.
*   **Feature flags**: Gate new features (new player UI, experimental ABR algorithm, new comment layout) behind flags for gradual rollout.
*   **Health checks**: Monitor CDN availability, manifest accessibility, and playback success rate.

```tsx
// Error boundary for player isolation
function WatchPage() {
  return (
    <div className="watch-page">
      <ErrorBoundary fallback={<PlayerErrorFallback />}>
        <VideoPlayer manifestUrl={manifestUrl} videoId={videoId} />
      </ErrorBoundary>

      <VideoMetadata videoId={videoId} />
      <EngagementBar videoId={videoId} />

      <ErrorBoundary fallback={<CommentsErrorFallback />}>
        <Suspense fallback={<CommentsSkeleton />}>
          <CommentsSection videoId={videoId} />
        </Suspense>
      </ErrorBoundary>

      <ErrorBoundary fallback={<RecommendationsErrorFallback />}>
        <Suspense fallback={<RecommendationsSkeleton />}>
          <RecommendedVideos videoId={videoId} />
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
| **Very long video (4+ hours)** | Progress bar precision: use seconds (not pixels) for accurate seeking; buffer only 30s ahead to save bandwidth; thumbnail sprites may be very large — paginate sprite sheets |
| **User seeks past buffered region** | Discard current buffer; fetch segments at new position; show brief buffering spinner; ABR may start at lower quality then upgrade |
| **Slow network (2G/3G)** | ABR drops to 240p automatically; show quality indicator; if buffer empties, player enters buffering state with spinner |
| **Network disconnects mid-playback** | Player continues from buffer; when buffer exhausts, show "You are offline" overlay; auto-resume when connectivity returns (`online` event) |
| **Browser autoplay policy blocks playback** | Detect `play()` promise rejection; show a "Click to play" overlay with muted autoplay as fallback (muted autoplay is always allowed) |
| **Video not available (deleted or private)** | API returns 404/403; show "This video is unavailable" page with recommended videos |
| **Age restricted content** | API response includes `ageRestricted: true`; show age verification gate before loading manifest |
| **Multiple tabs playing** | Use `document.visibilityState` — pause video when tab becomes hidden (optional, configurable) |
| **Comment spam / long text** | Truncate comments at 500 characters with "Read more"; sanitize all input server-side; rate limit comment submissions |
| **Concurrent like/subscribe clicks** | Debounce rapid clicks; ignore duplicate mutations; use request deduplication in API layer |
| **Embed in third party sites** | Minimal player bundle; no sidebar/comments; communicate with parent page via `postMessage` API |
| **RTL languages** | Mirror player controls; swap left/right seek zones; ensure progress bar fills from right-to-left |
| **Low-end devices** | Cap max quality to 480p; reduce buffer size; disable hover preview; simplify animations |

---

## 15. Summary and Future Improvements

### Key Architectural Decisions

| Decision | Rationale |
|----------|-----------|
| SPA with SSR for watch/feed pages | SEO for shared video links; fast FCP; rich interactivity |
| HLS with hls.js for adaptive streaming | Universal browser support; automatic quality adaptation; mature library |
| Custom video controls over native | Consistent UX; YouTube-like features (quality, speed, PiP, captions) |
| React Query for server state | Caching, deduplication, optimistic updates, stale-while-revalidate |
| Local state for player internals | High-frequency `timeupdate` events; avoid global store churn |
| Code-split player and comments | Player loads on watch page only; comments lazy loaded below fold |
| IntersectionObserver for infinite scroll and mini-player | Performant, off-main-thread intersection detection |
| Cursor-based pagination | Stable under concurrent inserts; efficient for infinite scroll |
| localStorage for watch progress | Instant resume; periodic server sync for cross-device |

### Major Tradeoffs

| Tradeoff | Chosen | Alternative | Why |
|----------|--------|-------------|-----|
| Bundle size vs features | Custom player + hls.js (~60KB) | Full-featured video.js (~200KB+) | Smaller bundle; full control over UI and behavior |
| SSR complexity vs SEO | SSR for metadata shell | Full CSR with prerendering | Critical for social sharing (OG tags) and search indexing |
| Buffer size vs bandwidth | 30s with pause-aware throttling | Aggressive 60s+ buffering | Saves bandwidth on paused/abandoned videos |
| Optimistic updates vs consistency | Optimistic for likes/comments | Wait for server confirmation | Instant feedback; rare conflicts in engagement actions |
| Virtualization for comments vs simplicity | Virtualize at 50+ comments | Render all comments | Prevents DOM bloat in popular videos with thousands of comments |

### Future Enhancements

*   **Offline viewing**: Service Worker caches video segments for later playback (premium feature).
*   **Watch party**: Synchronized playback across multiple users via WebSocket.
*   **AI-generated chapters**: Auto-segment videos with chapter markers using backend ML.
*   **Theater mode**: Widen player to full browser width while dimming sidebar.
*   **Ambient mode**: Extract dominant colors from video frames to create ambient glow behind player.
*   **Playback in background tab**: Continue audio playback when tab is not visible.
*   **Multi-language audio tracks**: Switch audio language without reloading video.
*   **Advanced analytics for creators**: Real-time view counts, audience retention graphs (separate Creator Studio deep-dive).

---

Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: [https://github.com/ZeeshanAli-0704/front-end-system-design](https://github.com/ZeeshanAli-0704/front-end-system-design)
