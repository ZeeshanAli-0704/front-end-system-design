# Frontend System Design: Notification System

- [Frontend System Design: Notification System](#frontend-system-design-notification-system)
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
    - [5.2 Real Time Notification Delivery and Rendering Strategy (Deep Dive)](#52-real-time-notification-delivery-and-rendering-strategy-deep-dive)
      - [5.2.1 WebSocket vs SSE vs Polling](#521-websocket-vs-sse-vs-polling)
      - [5.2.2 Connection Lifecycle Management](#522-connection-lifecycle-management)
      - [5.2.3 Toast and Snackbar Queue System](#523-toast-and-snackbar-queue-system)
      - [5.2.4 Notification Bell Badge with Unread Count](#524-notification-bell-badge-with-unread-count)
      - [5.2.5 Notification Grouping and Stacking](#525-notification-grouping-and-stacking)
      - [5.2.6 Virtualized Notification List](#526-virtualized-notification-list)
      - [5.2.7 Web Push Notifications Integration](#527-web-push-notifications-integration)
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
    - [9.2 Real Time Delivery Channel Design (Deep Dive)](#92-real-time-delivery-channel-design-deep-dive)
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

*   A notification system is the **primary communication channel** between a platform and its users — delivering real-time alerts, updates, and action prompts across the application.
*   Target users: all platform users — consumers receiving activity alerts, content creators tracking engagement, and admins monitoring system events.
*   Primary use case: delivering timely, relevant notifications via an in-app notification center (bell icon + dropdown/panel), real-time toast banners, and optional browser push notifications.

---

### 1.2 Key User Personas

*   **Consumer**: Receives notifications about social interactions (likes, comments, follows), order updates, reminders, and system alerts. Manages notification preferences.
*   **Content Creator / Seller**: Tracks engagement notifications (new followers, post reactions), sales alerts, and time-sensitive updates.
*   **Admin / Internal User**: Receives system health alerts, moderation flags, and operational notifications.

---

### 1.3 Core User Flows (High Level)

*   **Receiving and Viewing Notifications (Primary Flow)**:
    1.  An event occurs on the platform (someone likes the user's post, an order ships, etc.).
    2.  The server pushes a notification via WebSocket/SSE → a toast appears briefly at the top/bottom of the screen.
    3.  The notification bell icon updates its **unread badge count** (e.g., "3").
    4.  User clicks the bell → a notification panel/dropdown opens showing a reverse-chronological list of notifications.
    5.  User clicks a specific notification → navigates to the relevant page (post, order, profile).
    6.  Notification is marked as read → badge count decrements.

*   **Managing Notifications (Secondary Flow)**:
    1.  User opens notification settings.
    2.  Toggles notification categories on/off (e.g., disable marketing, keep social).
    3.  Enables/disables browser push notifications.
    4.  Adjusts sound/vibration preferences.

*   **Bulk Actions (Secondary Flow)**:
    1.  User opens notification panel → taps "Mark all as read".
    2.  All visible notifications transition to read state → badge resets to 0.
    3.  User deletes/archives specific notifications.

---

## 2. Requirements

### 2.1 Functional Requirements

*   **Notification Bell**:
    *   Display a persistent bell icon in the app header with an unread count badge.
    *   Badge shows exact count for small numbers (1-9), "9+" for larger counts.
    *   Clicking the bell opens a notification panel/dropdown.
*   **Notification Panel**:
    *   Display a reverse-chronological list of notifications.
    *   Support multiple notification types rendered differently (social, transactional, system, promotional).
    *   Visual distinction between read and unread notifications (background color, bold text, dot indicator).
    *   Infinite scroll or paginated "Load More" for older notifications.
    *   "Mark all as read" action.
    *   Empty state when no notifications exist.
*   **Real Time Delivery**:
    *   New notifications appear instantly without page refresh.
    *   Toast/snackbar popup for high-priority notifications with auto-dismiss after timeout.
    *   Toast queue — multiple notifications arriving simultaneously are stacked or shown sequentially.
*   **Notification Interactions**:
    *   Click notification → navigate to the relevant resource (deep link).
    *   Mark individual notifications as read/unread.
    *   Delete or dismiss notifications.
    *   Action buttons within notifications (e.g., "Accept", "Decline" for friend requests).
*   **Notification Grouping**:
    *   Group related notifications (e.g., "Alice, Bob, and 3 others liked your post").
    *   Expand grouped notifications to see individual items.
*   **Push Notifications**:
    *   Browser push notifications for important events when the tab is not focused or the browser is closed.
    *   Permission prompt handling (ask, granted, denied states).
*   **Preference Management**:
    *   Category-level toggles (social, transactional, marketing).
    *   Channel toggles (in-app, push, email, SMS).

---

### 2.2 Non Functional Requirements

*   **Performance**: Notification bell renders within FCP < 1.5s; toast appears within 200ms of server event; notification panel opens in < 300ms; smooth 60fps scroll in notification list.
*   **Scalability**: Support users with thousands of accumulated notifications; real-time delivery at high throughput (100+ events/second across all users).
*   **Availability**: Graceful degradation — show cached notifications if the real-time channel drops; fall back from WebSocket to SSE to polling.
*   **Security**: Authenticated real-time channels; no cross-user notification leakage; XSS prevention on notification content; signed push subscription tokens.
*   **Accessibility**: Full keyboard navigation in notification panel; screen reader announces new notifications via `aria-live`; focus management on panel open/close; sufficient color contrast for read/unread states.
*   **Device Support**: Mobile web (primary), desktop web (responsive), low-end devices with limited memory.
*   **i18n**: RTL layout support; localized timestamps ("2 hours ago"); locale-aware number formatting; translated notification templates.

---

## 3. Scope Clarification (Interview Scoping)

### 3.1 In Scope

*   Notification bell with real-time unread badge count.
*   Notification panel/dropdown with notification list rendering.
*   Real-time notification delivery via WebSocket/SSE.
*   Toast/snackbar system for incoming notifications.
*   Notification grouping and stacking.
*   State management for notifications (read/unread, badge count).
*   API design from the frontend perspective.
*   Web Push Notifications integration.
*   Performance optimization for large notification lists.

---

### 3.2 Out of Scope

*   Backend notification generation and routing engine.
*   Email and SMS notification channels (backend concern).
*   Rich notification composer (admin tool for creating notification templates).
*   Analytics dashboard for notification delivery metrics.
*   A/B testing of notification content.
*   Native mobile push notifications (iOS/Android — handled by native SDKs).

---

### 3.3 Assumptions

*   User is authenticated; auth token is available via HTTP-only cookie or Authorization header.
*   Backend provides a WebSocket or SSE endpoint for real-time notification delivery.
*   Notifications are pre-formatted by the backend (frontend does not construct notification text from raw event data).
*   The notification system is a global feature embedded in the app shell (header), not a standalone page.

---

## 4. High Level Frontend Architecture

### 4.1 Overall Approach

*   **SPA** (Single Page Application) with the notification system as a **global module** embedded in the app shell.
*   **CSR** for all notification UI — the bell, panel, and toasts are interactive, real-time components that don't benefit from SSR.
*   The notification module is **always active** (WebSocket connection established on app boot) but the **panel UI is lazy-loaded** on first bell click.
*   The toast system is loaded in the **initial bundle** since it must be ready to display at any time.

---

### 4.2 Major Architectural Layers

```
┌──────────────────────────────────────────────────────────────┐
│  UI Layer                                                    │
│  ┌───────────────┐   ┌────────────────────────────────────┐  │
│  │ NotifBell     │   │ NotifPanel (Dropdown/Sidebar)      │  │
│  │ (Badge Count) │   │  ┌──────────────┐ ┌─────────────┐  │  │
│  └───────────────┘   │  │ NotifItem[]  │ │ FilterTabs  │  │  │
│                      │  │ ┌──────────┐ │ │ (All/Unread)│  │  │
│  ┌───────────────┐   │  │ │ Avatar   │ │ └─────────────┘  │  │
│  │ ToastStack    │   │  │ │ Content  │ │                   │  │
│  │ (top-right)   │   │  │ │ Actions  │ │ ┌─────────────┐  │  │
│  │ ┌───────────┐ │   │  │ │ Time     │ │ │ MarkAllRead │  │  │
│  │ │ Toast 1   │ │   │  │ └──────────┘ │ └─────────────┘  │  │
│  │ │ Toast 2   │ │   │  └──────────────┘                   │  │
│  │ └───────────┘ │   │        ┌─────────────────┐          │  │
│  └───────────────┘   │        │ LoadMore/Scroll │          │  │
│                      │        │ Sentinel        │          │  │
│                      │        └─────────────────┘          │  │
│                      └────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────────┤
│  State Management Layer                                      │
│  (Notification Store, Unread Count, Toast Queue,             │
│   Preferences, Connection Status)                            │
├──────────────────────────────────────────────────────────────┤
│  Real Time Transport Layer                                   │
│  (WebSocket Manager, SSE Fallback, Reconnection Logic,       │
│   Push Subscription, Message Deduplication)                  │
├──────────────────────────────────────────────────────────────┤
│  API and Data Access Layer                                   │
│  (REST Client, Retry Logic, Optimistic Updates,              │
│   Request Batching, AbortController)                         │
├──────────────────────────────────────────────────────────────┤
│  Shared / Utility Layer                                      │
│  (Relative time formatter, Sound player,                     │
│   Debounce/Throttle, Analytics tracker)                      │
└──────────────────────────────────────────────────────────────┘
```

---

### 4.3 External Integrations

*   **WebSocket/SSE Server**: Real-time notification delivery endpoint (e.g., `wss://api.example.com/notifications/stream`).
*   **Push Service**: Web Push API via Service Worker — subscribes to browser push notifications (FCM/VAPID).
*   **CDN**: Serves notification-related media (avatars, thumbnail images in notification cards).
*   **Analytics SDK**: Track notification impressions, click-through rates, dismiss rates, and permission prompts.
*   **Backend Services**: Notification list API, mark-read API, preferences API, push subscription API.
*   **Sound System**: Play notification sounds using the Web Audio API or `<audio>` element.

---

## 5. Component Design and Modularization

### 5.1 Component Hierarchy

```
AppShell (global layout — header + content)
 ├── Header
 │    └── NotificationBell
 │         ├── BellIcon (SVG)
 │         ├── UnreadBadge (count pill)
 │         └── NotificationPanel (dropdown — rendered on click)
 │              ├── PanelHeader
 │              │    ├── Title ("Notifications")
 │              │    ├── FilterTabs (All | Unread)
 │              │    └── MarkAllReadButton
 │              ├── NotificationList (scrollable container)
 │              │    └── NotificationItem[] (repeated)
 │              │         ├── NotificationAvatar (user photo or icon)
 │              │         ├── NotificationContent
 │              │         │    ├── NotificationText (rich text with actor names)
 │              │         │    ├── NotificationTimestamp ("2h ago")
 │              │         │    └── NotificationActions (Accept/Decline buttons)
 │              │         ├── UnreadDot (blue dot for unread)
 │              │         └── NotificationMenu (mark read, delete, mute)
 │              ├── NotificationGroupItem (expandable)
 │              │    ├── GroupSummary ("Alice, Bob, +3 liked your post")
 │              │    └── GroupExpandedList
 │              │         └── NotificationItem[]
 │              ├── LoadMoreSentinel (infinite scroll trigger)
 │              └── EmptyState ("No notifications yet")
 │
 └── ToastContainer (fixed position — top-right or bottom-right)
      └── ToastItem[] (animated stack)
           ├── ToastIcon (notification type icon)
           ├── ToastMessage (short text preview)
           ├── ToastAction (optional CTA button)
           ├── ToastDismiss (close button)
           └── ToastProgressBar (auto-dismiss timer visual)
```

---

### 5.2 Real Time Notification Delivery and Rendering Strategy (Deep Dive)

The notification system is fundamentally a **real-time communication pipeline** between server and client. Getting real-time delivery right matters because:
*   Notifications are **time-sensitive** — a delayed notification loses its value (e.g., "Someone is typing" shown 30 seconds late).
*   The system must handle **high-frequency bursts** (e.g., a viral post generating hundreds of likes in seconds) without overwhelming the UI.
*   The real-time channel must be **resilient** — reconnect automatically on network drops, handle tab backgrounding, and fall back gracefully.
*   The toast system must **queue and rate-limit** incoming notifications to avoid flooding the user's screen.
*   The bell badge must update **atomically and accurately** — showing a stale count destroys trust.

---

#### 5.2.1 WebSocket vs SSE vs Polling

##### The Core Question — How Does the Server Push Notifications to the Client?

There are three approaches for real-time server-to-client communication. Each has distinct trade-offs:

| Feature | WebSocket | Server-Sent Events (SSE) | Long Polling |
|---------|-----------|--------------------------|-------------|
| **Direction** | Full duplex (bidirectional) | Server → Client only | Client → Server (repeated requests) |
| **Protocol** | `ws://` / `wss://` (upgrade from HTTP) | Standard HTTP with `text/event-stream` | Standard HTTP (repeated GET) |
| **Connection** | Single persistent TCP connection | Single persistent HTTP connection | New HTTP request every N seconds |
| **Reconnection** | Manual (must implement) | Built-in (`EventSource` auto-reconnects) | Built-in (just make next request) |
| **Binary data** | Yes (ArrayBuffer, Blob) | No (text only, UTF-8) | Yes (any HTTP response) |
| **Browser support** | All modern browsers | All modern browsers | Universal |
| **HTTP/2 multiplexing** | No (separate TCP connection) | Yes (shares HTTP/2 connection) | Yes (standard HTTP) |
| **Proxy / CDN friendly** | Can be blocked by corporate proxies | Works through all HTTP proxies | Works everywhere |
| **Max connections per domain** | Separate from HTTP limit | Counts toward HTTP/2 stream limit | Counts toward connection limit |
| **Server complexity** | Higher (WebSocket server needed) | Lower (standard HTTP response with streaming) | Lowest (standard request/response) |

##### Why WebSocket Is the Primary Choice for Notifications

For a notification system, **WebSocket** is the best primary channel because:

1.  **Bidirectional communication**: The client needs to send acknowledgments (mark-as-read, dismiss) back through the same channel. With SSE, you'd need a separate REST endpoint for client-to-server messages.

2.  **Lower latency**: WebSocket frames have minimal overhead (2-14 bytes header) vs SSE (HTTP chunked encoding overhead) vs polling (full HTTP request/response per poll).

3.  **Efficient for high-frequency updates**: A viral post generating 50 likes per second sends 50 small WebSocket frames. With polling at 5-second intervals, you'd batch 250 events per poll — higher latency and burstier UI updates.

4.  **Single connection for all real-time features**: The same WebSocket can carry notifications, typing indicators, presence status, and live activity updates. SSE would require separate EventSource connections per feature (or a multiplexed stream).

##### Why SSE Is the Best Fallback

If WebSocket fails (corporate proxy blocks `wss://`, or server doesn't support it), SSE is the ideal fallback because:

*   **Auto-reconnection**: `EventSource` reconnects automatically with exponential backoff. WebSocket requires manual reconnection logic.
*   **HTTP/2 multiplexing**: SSE connections share the HTTP/2 multiplexed connection, avoiding the "one extra TCP connection" cost of WebSocket.
*   **Simpler server implementation**: The server just writes to an HTTP response stream — no WebSocket upgrade handshake.
*   **Works through all proxies**: SSE is standard HTTP, so corporate proxies and CDNs handle it correctly.

##### Implementation — Fallback Chain

```tsx
type TransportType = 'websocket' | 'sse' | 'polling';

function createNotificationTransport(): TransportType {
  // Try WebSocket first
  if ('WebSocket' in window) {
    try {
      const ws = new WebSocket('wss://api.example.com/notifications/stream');
      ws.onopen = () => { /* connected via WebSocket */ };
      ws.onerror = () => {
        // WebSocket failed — fall back to SSE
        ws.close();
        return createSSETransport();
      };
      return 'websocket';
    } catch {
      return createSSETransport();
    }
  }
  return createSSETransport();
}

function createSSETransport(): TransportType {
  if ('EventSource' in window) {
    const sse = new EventSource('/api/notifications/stream');
    sse.onerror = () => {
      // SSE failed — fall back to polling
      sse.close();
      return startPolling();
    };
    return 'sse';
  }
  return startPolling();
}

function startPolling(): TransportType {
  // Ultimate fallback — poll every 30 seconds
  setInterval(async () => {
    const response = await fetch('/api/notifications?since=' + lastTimestamp);
    const data = await response.json();
    processNotifications(data.notifications);
  }, 30_000);
  return 'polling';
}
```

##### When to Use Which

| Scenario | Recommended Transport | Why |
|----------|----------------------|-----|
| **Modern web app, no proxy restrictions** | WebSocket (primary) | Lowest latency, bidirectional |
| **Corporate environment with strict proxies** | SSE (auto-fallback) | Works through HTTP proxies |
| **Very old browsers or extreme restrictions** | Long polling (last resort) | Universal compatibility |
| **Read-only notifications (no client acks)** | SSE (primary) | Simpler, auto-reconnect, no need for bidirectional |
| **Multiple real-time features (notif + chat + presence)** | WebSocket (single connection) | Multiplexes multiple event types |

---

#### 5.2.2 Connection Lifecycle Management

##### The Problem — Real-Time Connections Are Fragile

A WebSocket or SSE connection must survive:
*   **Network drops** (Wi-Fi switch, mobile data loss, tunnel/subway).
*   **Tab backgrounding** (browser throttles timers and may close connections after 5 minutes).
*   **Server restarts / deployments** (connection drops during rolling deploys).
*   **Token expiry** (auth token expires while connection is open).
*   **Device sleep** (laptop lid close, phone screen off).

Without proper lifecycle management, the user's notification bell silently goes stale — they stop receiving real-time updates and don't know it.

##### Connection State Machine

```
┌────────────┐    connect()     ┌──────────────┐
│            │ ───────────────→ │              │
│ DISCONNECTED│                 │  CONNECTING  │
│            │ ←─────────────── │              │
└────────────┘   max retries    └──────┬───────┘
      ↑          exceeded              │
      │                                │ onopen
      │                                ▼
      │                        ┌──────────────┐
      │          onclose/      │              │
      │          onerror       │  CONNECTED   │
      └─────────────────────── │              │
               │               └──────────────┘
               │                       │
               ▼                       │ visibilitychange
        ┌──────────────┐               │ (tab hidden)
        │              │               ▼
        │ RECONNECTING │        ┌──────────────┐
        │ (exp backoff)│        │              │
        │              │        │   PAUSED     │
        └──────────────┘        │ (tab hidden) │
               │                └──────────────┘
               │ onopen                │
               ▼                       │ visibilitychange
        ┌──────────────┐               │ (tab visible)
        │  CONNECTED   │ ←────────────┘
        └──────────────┘   reconnect + fetch missed
```

##### Implementation — Resilient WebSocket Manager

```tsx
class NotificationSocket {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;
  private baseDelay = 1000;      // 1 second
  private maxDelay = 30000;      // 30 seconds
  private reconnectTimer: number | null = null;
  private lastEventId: string | null = null;
  private onNotification: (notif: Notification) => void;

  constructor(onNotification: (notif: Notification) => void) {
    this.onNotification = onNotification;
    this.setupVisibilityHandler();
  }

  connect() {
    const url = new URL('wss://api.example.com/notifications/stream');
    // Send last event ID so server can replay missed notifications
    if (this.lastEventId) {
      url.searchParams.set('lastEventId', this.lastEventId);
    }

    this.ws = new WebSocket(url.toString());

    this.ws.onopen = () => {
      this.reconnectAttempts = 0; // reset on successful connect
    };

    this.ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.lastEventId = data.eventId;
      this.onNotification(data.notification);
    };

    this.ws.onclose = (event) => {
      if (event.code !== 1000) {
        // Abnormal close — reconnect
        this.scheduleReconnect();
      }
    };

    this.ws.onerror = () => {
      this.ws?.close();
    };
  }

  private scheduleReconnect() {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      // Fall back to SSE or polling
      this.fallbackToSSE();
      return;
    }

    // Exponential backoff with jitter
    const delay = Math.min(
      this.baseDelay * Math.pow(2, this.reconnectAttempts) + Math.random() * 1000,
      this.maxDelay
    );

    this.reconnectTimer = window.setTimeout(() => {
      this.reconnectAttempts++;
      this.connect();
    }, delay);
  }

  private setupVisibilityHandler() {
    document.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'hidden') {
        // Tab is hidden — connection may be throttled by browser
        // Keep connection alive but note the timestamp
        this.pausedAt = Date.now();
      } else {
        // Tab is visible again
        if (this.ws?.readyState !== WebSocket.OPEN) {
          // Connection was dropped while tab was hidden
          this.connect(); // reconnect with lastEventId to get missed notifications
        } else if (this.pausedAt && Date.now() - this.pausedAt > 60_000) {
          // Connected but was hidden for > 1 minute — fetch missed notifications via REST
          this.fetchMissedNotifications();
        }
        this.pausedAt = null;
      }
    });
  }

  private async fetchMissedNotifications() {
    // Supplement real-time channel with REST fetch for any missed during background
    const response = await fetch(
      `/api/notifications?after=${this.lastEventId}`
    );
    const data = await response.json();
    data.notifications.forEach((notif: Notification) => {
      this.onNotification(notif);
    });
  }

  disconnect() {
    if (this.reconnectTimer) clearTimeout(this.reconnectTimer);
    this.ws?.close(1000, 'User navigated away');
  }
}
```

##### Key Design Decisions Explained

| Decision | Rationale |
|----------|-----------|
| **Exponential backoff with jitter** | Prevents thundering herd — if the server restarts, 10K clients don't all reconnect at the same millisecond. Jitter (random 0-1s) spreads reconnections. |
| **`lastEventId` on reconnect** | Server replays events missed during the disconnect gap. Without this, the user misses notifications. |
| **Visibility change handler** | Browsers throttle background tabs — timers slow to 1/min, WebSocket may be closed after 5 min. On tab re-focus, check connection health and fetch missed events. |
| **REST backfill after long background** | Even if WebSocket stayed connected during background, messages may have been dropped by the browser's throttling. A REST fetch ensures nothing was missed. |
| **Max reconnect attempts** | After 10 failed attempts (~5 minutes of exponential backoff), fall back to a simpler transport instead of retrying forever. |

---

#### 5.2.3 Toast and Snackbar Queue System

##### The Problem — Notifications Arrive Unpredictably

Real-time notifications can arrive:
*   **One at a time** (normal case — "Alice liked your post").
*   **In rapid bursts** (viral post — 50 likes in 10 seconds).
*   **While a toast is already showing** (new notification arrives before the previous one dismisses).
*   **While the user is interacting** (user is typing, filling a form — toast shouldn't be distracting).

Without a queue system, rapid arrivals either:
*   **Stack infinitely** — screen fills with 20 toasts covering the content.
*   **Replace each other instantly** — user can't read any of them.
*   **Get lost** — later toasts silently fail to display.

##### The Queue Model

```
Incoming notifications (from WebSocket)
       ↓
┌──────────────────────────────────────────────┐
│  TOAST QUEUE (FIFO with priority)            │
│  [High] [High] [Normal] [Normal] [Low]       │
│                                              │
│  Rules:                                      │
│   • Max 3 visible toasts at once             │
│   • High priority: show immediately          │
│   • Normal priority: queue if 3 visible      │
│   • Low priority: skip toast, badge only     │
│   • Auto-dismiss: 5s (normal), 8s (action)   │
│   • User hover: pause auto-dismiss timer     │
└──────────────────────────────────────────────┘
       ↓
┌──────────────────────────────────────────────┐
│  VISIBLE TOAST STACK (max 3)                 │
│                                              │
│  ┌─────────────────────────────────────────┐ │
│  │ Toast 1: "Alice liked your photo" [5s]  │ │
│  └─────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────┐ │
│  │ Toast 2: "Order #123 shipped" [8s]      │ │
│  └─────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────┐ │
│  │ Toast 3: "Bob started following you"    │ │
│  └─────────────────────────────────────────┘ │
│                                              │
│  Toast 4, 5, 6... waiting in queue           │
└──────────────────────────────────────────────┘
```

##### Burst Throttling — Handling Viral Moments

When a post goes viral, the user might receive 100 "X liked your post" notifications in a minute. Showing each as a separate toast is overwhelming.

**Solution: Aggregate within a time window.**

```
Incoming:
  t=0s: "Alice liked your post"   → Show toast immediately
  t=1s: "Bob liked your post"     → Aggregate: "Alice, Bob liked your post"
  t=2s: "Carol liked your post"   → Aggregate: "Alice, Bob, Carol liked your post"
  t=3s: "Dave liked your post"    → Aggregate: "Alice and 3 others liked your post"
  ...
  t=5s: First toast auto-dismisses → Show next queued toast (if any)
```

**The aggregation window** (3-5 seconds) collects same-type notifications and merges them into a single grouped toast.

##### Implementation — Toast Queue Manager

```tsx
import { useCallback, useEffect, useReducer, useRef } from 'react';

type ToastPriority = 'high' | 'normal' | 'low';

type ToastItem = {
  id: string;
  message: string;
  type: 'social' | 'transactional' | 'system';
  priority: ToastPriority;
  action?: { label: string; onClick: () => void };
  duration: number;     // ms — auto-dismiss timeout
  groupKey?: string;    // for aggregation (e.g., "like:post_123")
};

type ToastState = {
  queue: ToastItem[];
  visible: ToastItem[];
  maxVisible: number;
};

type ToastAction =
  | { type: 'ENQUEUE'; toast: ToastItem }
  | { type: 'SHOW_NEXT' }
  | { type: 'DISMISS'; id: string }
  | { type: 'AGGREGATE'; groupKey: string; toast: ToastItem };

function toastReducer(state: ToastState, action: ToastAction): ToastState {
  switch (action.type) {
    case 'ENQUEUE': {
      // Check if we can aggregate with an existing visible/queued toast
      if (action.toast.groupKey) {
        const existingIndex = state.visible.findIndex(
          (t) => t.groupKey === action.toast.groupKey
        );
        if (existingIndex !== -1) {
          // Update existing toast with aggregated message
          const updated = [...state.visible];
          updated[existingIndex] = {
            ...updated[existingIndex],
            message: action.toast.message, // pre-aggregated by caller
          };
          return { ...state, visible: updated };
        }
      }

      // If under max visible, show immediately
      if (state.visible.length < state.maxVisible) {
        return {
          ...state,
          visible: [...state.visible, action.toast],
        };
      }

      // Otherwise, queue it
      // High priority: front of queue. Normal: back.
      const queue =
        action.toast.priority === 'high'
          ? [action.toast, ...state.queue]
          : [...state.queue, action.toast];

      return { ...state, queue };
    }

    case 'DISMISS': {
      const visible = state.visible.filter((t) => t.id !== action.id);
      // Promote next from queue
      if (state.queue.length > 0 && visible.length < state.maxVisible) {
        const [next, ...rest] = state.queue;
        return { ...state, visible: [...visible, next], queue: rest };
      }
      return { ...state, visible };
    }

    case 'SHOW_NEXT': {
      if (state.queue.length === 0 || state.visible.length >= state.maxVisible) {
        return state;
      }
      const [next, ...rest] = state.queue;
      return {
        ...state,
        visible: [...state.visible, next],
        queue: rest,
      };
    }

    default:
      return state;
  }
}

function useToastQueue() {
  const [state, dispatch] = useReducer(toastReducer, {
    queue: [],
    visible: [],
    maxVisible: 3,
  });

  const timers = useRef<Map<string, number>>(new Map());

  const enqueue = useCallback((toast: ToastItem) => {
    if (toast.priority === 'low') {
      // Low priority: skip toast entirely — only update badge
      return;
    }
    dispatch({ type: 'ENQUEUE', toast });
  }, []);

  const dismiss = useCallback((id: string) => {
    const timer = timers.current.get(id);
    if (timer) clearTimeout(timer);
    timers.current.delete(id);
    dispatch({ type: 'DISMISS', id });
  }, []);

  // Auto-dismiss timers for visible toasts
  useEffect(() => {
    state.visible.forEach((toast) => {
      if (!timers.current.has(toast.id)) {
        const timer = window.setTimeout(() => {
          dismiss(toast.id);
        }, toast.duration);
        timers.current.set(toast.id, timer);
      }
    });
  }, [state.visible, dismiss]);

  return { visible: state.visible, enqueue, dismiss };
}
```

##### Toast Component with Animations

```tsx
function ToastContainer() {
  const { visible, dismiss } = useToastQueue();

  return (
    <div
      className="toast-container"
      role="status"
      aria-live="polite"
      aria-relevant="additions removals"
    >
      {visible.map((toast, index) => (
        <div
          key={toast.id}
          className={`toast toast--${toast.type}`}
          style={{
            transform: `translateY(${index * 80}px)`,
            transition: 'transform 0.3s ease, opacity 0.3s ease',
          }}
          onMouseEnter={() => pauseTimer(toast.id)}
          onMouseLeave={() => resumeTimer(toast.id)}
          role="alert"
        >
          <span className="toast__message">{toast.message}</span>
          {toast.action && (
            <button
              className="toast__action"
              onClick={toast.action.onClick}
            >
              {toast.action.label}
            </button>
          )}
          <button
            className="toast__dismiss"
            onClick={() => dismiss(toast.id)}
            aria-label="Dismiss notification"
          >
            ✕
          </button>
        </div>
      ))}
    </div>
  );
}
```

```css
.toast-container {
  position: fixed;
  top: 16px;
  right: 16px;
  z-index: 9999;
  display: flex;
  flex-direction: column;
  gap: 8px;
  pointer-events: none;        /* allow clicks to pass through empty space */
}

.toast {
  pointer-events: auto;         /* but toasts themselves are clickable */
  min-width: 320px;
  max-width: 420px;
  padding: 12px 16px;
  border-radius: 8px;
  background: #1a1a2e;
  color: #ffffff;
  box-shadow: 0 4px 16px rgba(0,0,0,0.2);
  display: flex;
  align-items: center;
  gap: 12px;
  animation: slideIn 0.3s ease forwards;
}

@keyframes slideIn {
  from { transform: translateX(120%); opacity: 0; }
  to   { transform: translateX(0);    opacity: 1; }
}

.toast--social      { border-left: 4px solid #3b82f6; }
.toast--transactional { border-left: 4px solid #10b981; }
.toast--system      { border-left: 4px solid #f59e0b; }
```

##### Pause on Hover — Don't Steal the User's Toast

When the user hovers over a toast (to read it or click an action), the auto-dismiss timer must **pause**. Otherwise, the toast disappears while they're reading it.

```tsx
function pauseTimer(toastId: string) {
  const timer = timers.current.get(toastId);
  if (timer) {
    clearTimeout(timer);
    timers.current.delete(toastId);
    // Store remaining time for resume
  }
}

function resumeTimer(toastId: string) {
  // Restart the timer with remaining duration
  const remaining = getRemainingTime(toastId);
  const timer = window.setTimeout(() => dismiss(toastId), remaining);
  timers.current.set(toastId, timer);
}
```

---

#### 5.2.4 Notification Bell Badge with Unread Count

##### The Badge Must Be Instantly Accurate

The unread badge is the user's **primary trust signal** for notifications. If it shows "3" but there are actually 5 unread notifications, or shows "3" after the user has read them all, trust erodes. The badge must:

*   **Increment** immediately when a new notification arrives (via WebSocket).
*   **Decrement** immediately when a notification is read (optimistic update).
*   **Reset to 0** when "Mark all as read" is clicked (optimistic update).
*   **Sync with server** on reconnection or app focus (handle drift from missed events).

##### Implementation

```tsx
function NotificationBell() {
  const unreadCount = useNotificationStore((s) => s.unreadCount);
  const [isPanelOpen, setIsPanelOpen] = useState(false);
  const bellRef = useRef<HTMLButtonElement>(null);

  // Format display: show exact count for 1-9, "9+" for larger
  const displayCount = unreadCount > 9 ? '9+' : unreadCount > 0 ? String(unreadCount) : null;

  return (
    <div className="notification-bell-wrapper">
      <button
        ref={bellRef}
        className="notification-bell"
        onClick={() => setIsPanelOpen((prev) => !prev)}
        aria-label={`Notifications${unreadCount > 0 ? `, ${unreadCount} unread` : ''}`}
        aria-expanded={isPanelOpen}
        aria-haspopup="true"
      >
        <BellIcon />
        {displayCount && (
          <span className="unread-badge" aria-hidden="true">
            {displayCount}
          </span>
        )}
      </button>

      {isPanelOpen && (
        <NotificationPanel
          onClose={() => {
            setIsPanelOpen(false);
            bellRef.current?.focus(); // return focus to bell on close
          }}
        />
      )}
    </div>
  );
}
```

```css
.notification-bell-wrapper {
  position: relative;      /* anchor for the dropdown panel */
}

.notification-bell {
  position: relative;
  background: none;
  border: none;
  cursor: pointer;
  padding: 8px;
}

.unread-badge {
  position: absolute;
  top: 2px;
  right: 2px;
  min-width: 18px;
  height: 18px;
  padding: 0 5px;
  border-radius: 9px;
  background: #ef4444;
  color: #ffffff;
  font-size: 11px;
  font-weight: 700;
  line-height: 18px;
  text-align: center;
  /* Animate badge appearance */
  animation: badgePop 0.3s ease;
}

@keyframes badgePop {
  0% { transform: scale(0); }
  70% { transform: scale(1.2); }
  100% { transform: scale(1); }
}
```

##### Count Synchronization Strategy

```
┌─────────────────────────────────────────────────────────────┐
│  Unread Count Sources (must agree)                          │
│                                                             │
│  1. Server (source of truth):  GET /api/notifications/count │
│     → Response: { unreadCount: 5 }                          │
│                                                             │
│  2. WebSocket increment:  +1 on each new notification       │
│                                                             │
│  3. Local decrement:  -1 on each mark-as-read              │
│                                                             │
│  4. Reconciliation:  On reconnect or focus,                 │
│     re-fetch count from server to correct any drift         │
└─────────────────────────────────────────────────────────────┘
```

```tsx
// In the notification store (Zustand example)
const useNotificationStore = create<NotificationStore>((set, get) => ({
  unreadCount: 0,
  notifications: [],

  // Called when WebSocket delivers a new notification
  addNotification: (notif) => {
    set((state) => ({
      notifications: [notif, ...state.notifications],
      unreadCount: state.unreadCount + 1,
    }));
  },

  // Called when user clicks a notification
  markAsRead: (notifId) => {
    set((state) => ({
      notifications: state.notifications.map((n) =>
        n.id === notifId ? { ...n, isRead: true } : n
      ),
      unreadCount: Math.max(0, state.unreadCount - 1),
    }));
    // Fire-and-forget API call
    fetch(`/api/notifications/${notifId}/read`, { method: 'POST' });
  },

  // Called on "Mark all as read"
  markAllAsRead: () => {
    set((state) => ({
      notifications: state.notifications.map((n) => ({ ...n, isRead: true })),
      unreadCount: 0,
    }));
    fetch('/api/notifications/read-all', { method: 'POST' });
  },

  // Called on reconnect / tab focus to correct drift
  syncUnreadCount: async () => {
    const response = await fetch('/api/notifications/count');
    const { unreadCount } = await response.json();
    set({ unreadCount });
  },
}));
```

##### Document Title Badge — Unread Count in the Browser Tab

When the tab is not focused, show the unread count in the page title so users see it in their taskbar/tab bar:

```tsx
useEffect(() => {
  const originalTitle = document.title.replace(/^\(\d+\+?\)\s/, '');
  if (unreadCount > 0) {
    document.title = `(${unreadCount > 9 ? '9+' : unreadCount}) ${originalTitle}`;
  } else {
    document.title = originalTitle;
  }
}, [unreadCount]);
```

This produces `(3) MyApp` in the browser tab — a universal web pattern used by Gmail, Slack, Facebook, etc.

---

#### 5.2.5 Notification Grouping and Stacking

##### The Problem — Notification Overload

A single event can generate many notifications:
*   **Post goes viral**: "Alice liked your post", "Bob liked your post", "Carol liked..." × 200.
*   **Group chat**: 50 messages in a conversation → 50 separate notifications.
*   **Follower spike**: "Alice followed you", "Bob followed you" × 30.

Showing each as a separate notification item:
*   **Overwhelms the list** — 200 identical "X liked your post" items push out all other notifications.
*   **Provides no value** — the user doesn't need to see each individual like; they care about the aggregate ("200 people liked your post").
*   **Wastes screen space** — grouped view is far more scannable.

##### Grouping Strategy

```
BEFORE Grouping:
  ┌─────────────────────────────────────────────┐
  │ 🔔 Alice liked your post "Hello world"      │
  │ 🔔 Bob liked your post "Hello world"        │
  │ 🔔 Carol liked your post "Hello world"      │
  │ 🔔 Dave liked your post "Hello world"       │
  │ 🔔 Eve liked your post "Hello world"        │
  │ 🔔 Frank commented on your post             │
  │ 🔔 Grace started following you              │
  │ 🔔 Helen started following you              │
  └─────────────────────────────────────────────┘

AFTER Grouping:
  ┌─────────────────────────────────────────────┐
  │ 🔔 Alice, Bob, and 3 others liked your      │
  │    post "Hello world"                [5]     │
  │ 🔔 Frank commented on your post              │
  │ 🔔 Grace and Helen started following you [2] │
  └─────────────────────────────────────────────┘
```

##### Group Key Generation

Notifications are grouped by a **composite key** that identifies "same type + same target":

```tsx
function getGroupKey(notification: Notification): string {
  // Group by: action type + target resource
  switch (notification.type) {
    case 'like':
      return `like:${notification.targetId}`;      // all likes on the same post
    case 'comment':
      return `comment:${notification.targetId}`;   // all comments on the same post
    case 'follow':
      return 'follow';                             // all follows grouped together
    case 'mention':
      return `mention:${notification.targetId}`;   // all mentions in the same post
    default:
      return notification.id;                      // unique — no grouping
  }
}
```

##### Implementation — Building Grouped Notifications

```tsx
type NotificationGroup = {
  groupKey: string;
  type: string;
  notifications: Notification[];
  latestTimestamp: string;
  actors: Array<{ userId: string; username: string; avatarUrl: string }>;
  targetId: string;
  targetTitle: string;
  isRead: boolean;        // true only if ALL notifications in the group are read
};

function groupNotifications(notifications: Notification[]): NotificationGroup[] {
  const groupMap = new Map<string, Notification[]>();

  // Group by key
  notifications.forEach((notif) => {
    const key = getGroupKey(notif);
    const group = groupMap.get(key) || [];
    group.push(notif);
    groupMap.set(key, group);
  });

  // Convert to group objects
  const groups: NotificationGroup[] = [];
  groupMap.forEach((notifs, key) => {
    const latest = notifs[0]; // notifications are already sorted by time
    groups.push({
      groupKey: key,
      type: latest.type,
      notifications: notifs,
      latestTimestamp: latest.createdAt,
      actors: notifs
        .map((n) => ({ userId: n.actorId, username: n.actorName, avatarUrl: n.actorAvatar }))
        .filter((actor, i, arr) => arr.findIndex((a) => a.userId === actor.userId) === i),
      targetId: latest.targetId,
      targetTitle: latest.targetTitle,
      isRead: notifs.every((n) => n.isRead),
    });
  });

  // Sort groups by latest timestamp
  return groups.sort(
    (a, b) => new Date(b.latestTimestamp).getTime() - new Date(a.latestTimestamp).getTime()
  );
}
```

##### Rendering Grouped Summary Text

```tsx
function getGroupSummaryText(group: NotificationGroup): string {
  const actors = group.actors;
  const count = actors.length;

  if (count === 1) {
    return `${actors[0].username} ${getActionVerb(group.type)} ${group.targetTitle}`;
  }
  if (count === 2) {
    return `${actors[0].username} and ${actors[1].username} ${getActionVerb(group.type)} ${group.targetTitle}`;
  }
  return `${actors[0].username}, ${actors[1].username}, and ${count - 2} others ${getActionVerb(group.type)} ${group.targetTitle}`;
}

function getActionVerb(type: string): string {
  switch (type) {
    case 'like': return 'liked your post';
    case 'comment': return 'commented on your post';
    case 'follow': return 'started following you';
    case 'mention': return 'mentioned you in';
    default: return '';
  }
}
```

##### Expand/Collapse Group

```tsx
function NotificationGroupItem({ group }: { group: NotificationGroup }) {
  const [isExpanded, setIsExpanded] = useState(false);

  return (
    <div className={`notif-group ${group.isRead ? '' : 'notif-group--unread'}`}>
      <button
        className="notif-group__summary"
        onClick={() => group.notifications.length > 1 && setIsExpanded(!isExpanded)}
        aria-expanded={isExpanded}
      >
        <AvatarStack avatars={group.actors.slice(0, 3)} />
        <span>{getGroupSummaryText(group)}</span>
        <RelativeTime timestamp={group.latestTimestamp} />
        {group.notifications.length > 1 && (
          <span className="notif-group__count">{group.notifications.length}</span>
        )}
      </button>

      {isExpanded && (
        <div className="notif-group__expanded">
          {group.notifications.map((notif) => (
            <NotificationItem key={notif.id} notification={notif} />
          ))}
        </div>
      )}
    </div>
  );
}
```

---

#### 5.2.6 Virtualized Notification List

##### When to Virtualize

Power users can accumulate **thousands of notifications** over time. The notification panel typically shows a scrollable list that loads more items on scroll. Without virtualization:

*   **500 notification items** = ~2500+ DOM nodes (avatar + text + timestamp + actions per item).
*   **Slow panel open**: Rendering 500 items on first open causes 200-500ms jank.
*   **Memory pressure**: All 500 images loaded, all text nodes in DOM.

With virtualization:
*   Only **15-20 items** are in the DOM at any time (viewport + overscan).
*   Panel opens instantly regardless of total notification count.
*   Memory usage is bounded.

##### Implementation

```tsx
import { useRef } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

function NotificationList({ groups }: { groups: NotificationGroup[] }) {
  const scrollRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: groups.length,
    getScrollElement: () => scrollRef.current,
    estimateSize: () => 80,     // avg notification item height ~80px
    overscan: 5,
  });

  return (
    <div
      ref={scrollRef}
      className="notification-list"
      style={{ height: '400px', overflow: 'auto' }}
      role="list"
      aria-label="Notifications"
    >
      <div
        style={{
          height: virtualizer.getTotalSize(),
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualItem) => {
          const group = groups[virtualItem.index];
          return (
            <div
              key={group.groupKey}
              ref={virtualizer.measureElement}
              data-index={virtualItem.index}
              role="listitem"
              style={{
                position: 'absolute',
                top: virtualItem.start,
                width: '100%',
              }}
            >
              <NotificationGroupItem group={group} />
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

| Metric | No virtualization (500 items) | Virtualized (overscan=5) |
|--------|-------------------------------|--------------------------|
| DOM nodes | ~2500+ | ~100 (20 items x 5 nodes) |
| Panel open time | 200-500ms | 10-30ms |
| Memory usage | All images loaded | Only ~20 images loaded |
| Scroll smoothness | Can jank on low-end | Consistent 60fps |
| Bundle size added | 0 | ~5-8KB (`@tanstack/react-virtual`) |

---

#### 5.2.7 Web Push Notifications Integration

##### Why Push Notifications Matter

In-app notifications (bell + toast) only work when the user **has the tab open**. Push notifications bridge the gap:
*   **Tab is closed**: User is on a different site or using another app.
*   **Browser is in background**: User is working in another application.
*   **Browser is closed** (on supported platforms): Service Worker receives the push.

##### The Web Push Architecture

```
┌─────────────────┐    1. Subscribe     ┌────────────────────┐
│  Frontend App   │ ──────────────────→ │  Push Service      │
│  (Service Worker│                     │  (FCM / Mozilla    │
│   + Push API)   │ ←────────────────── │   Push / APNs)     │
│                 │    4. Push Message   │                    │
└────────┬────────┘                     └────────┬───────────┘
         │                                       ↑
         │  2. Send subscription                 │ 3. Server sends
         │     to your backend                   │    push via API
         ▼                                       │
┌─────────────────┐                     ┌────────┴───────────┐
│  Your Backend   │ ──────────────────→ │  Push Service      │
│  (stores sub,   │   3. POST push msg  │  (delivers to      │
│   sends push)   │                     │   user's browser)  │
└─────────────────┘                     └────────────────────┘
```

**Step-by-step flow:**

1.  **User grants permission**: Frontend calls `Notification.requestPermission()` → user clicks "Allow".
2.  **Subscribe to push**: Service Worker calls `pushManager.subscribe()` with VAPID public key → returns a `PushSubscription` object containing the browser's push endpoint URL.
3.  **Send subscription to backend**: Frontend POSTs the subscription to your server. Server stores it.
4.  **Server sends push**: When a notification event occurs, server sends the payload to the push service endpoint (FCM/Mozilla/APNs).
5.  **Push service delivers**: The push service delivers the message to the user's browser.
6.  **Service Worker receives**: The `push` event fires in the Service Worker → displays a native notification via `self.registration.showNotification()`.
7.  **User clicks notification**: `notificationclick` event fires → Service Worker opens the app and navigates to the relevant page.

##### Implementation — Permission and Subscription

```tsx
async function requestPushPermission(): Promise<boolean> {
  // Check if push is supported
  if (!('serviceWorker' in navigator) || !('PushManager' in window)) {
    return false;
  }

  // Check current permission state
  const permission = await Notification.requestPermission();
  if (permission !== 'granted') {
    return false;
  }

  // Get the Service Worker registration
  const registration = await navigator.serviceWorker.ready;

  // Subscribe to push
  const subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true,       // required: must show a notification for each push
    applicationServerKey: urlBase64ToUint8Array(VAPID_PUBLIC_KEY),
  });

  // Send subscription to your backend
  await fetch('/api/push/subscribe', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(subscription.toJSON()),
  });

  return true;
}
```

##### Service Worker — Handling Push Events

```js
// sw.js (Service Worker)

self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? {};

  const options = {
    body: data.body || 'You have a new notification',
    icon: data.icon || '/icons/notification-192.png',
    badge: '/icons/badge-72.png',
    tag: data.tag || 'default',            // group by tag — replace existing with same tag
    renotify: true,                        // vibrate even if replacing same tag
    data: { url: data.url || '/' },        // pass deep link URL
    actions: data.actions || [],           // e.g., [{ action: 'view', title: 'View' }]
    timestamp: data.timestamp || Date.now(),
  };

  event.waitUntil(
    self.registration.showNotification(data.title || 'Notification', options)
  );
});

self.addEventListener('notificationclick', (event) => {
  event.notification.close();

  const url = event.notification.data?.url || '/';

  event.waitUntil(
    clients.matchAll({ type: 'window', includeUncontrolled: true }).then((clientList) => {
      // If app is already open in a tab, focus it and navigate
      for (const client of clientList) {
        if (client.url.includes(self.location.origin) && 'focus' in client) {
          client.focus();
          client.postMessage({ type: 'NAVIGATE', url });
          return;
        }
      }
      // Otherwise, open a new tab
      return clients.openWindow(url);
    })
  );
});
```

##### Permission Prompt UX — Don't Ask Immediately

**Bad pattern**: Immediately showing the browser's permission dialog on page load.
*   User hasn't seen the app's value yet → high denial rate.
*   Once denied, the user must manually re-enable permissions in browser settings → friction.

**Good pattern**: Custom pre-prompt that explains the value, then triggers the real prompt on opt-in.

```tsx
function PushPermissionPrompt() {
  const [showPrompt, setShowPrompt] = useState(false);

  useEffect(() => {
    // Show our custom prompt after the user has engaged with notifications
    // e.g., after they've received their first in-app notification
    if (Notification.permission === 'default' && hasReceivedFirstNotification()) {
      setShowPrompt(true);
    }
  }, []);

  if (!showPrompt) return null;

  return (
    <div className="push-prompt" role="alertdialog" aria-label="Enable push notifications">
      <p>Get notified when someone interacts with your posts, even when you're away.</p>
      <button onClick={async () => {
        await requestPushPermission();
        setShowPrompt(false);
      }}>
        Enable Notifications
      </button>
      <button onClick={() => setShowPrompt(false)}>Not Now</button>
    </div>
  );
}
```

##### When to Use Push vs In-App

| Scenario | In-App (Bell + Toast) | Push Notification | Both |
|----------|----------------------|-------------------|------|
| User has tab open and focused | ✅ Primary | ❌ Unnecessary | — |
| User has tab open but unfocused | ✅ Works (toast visible) | ✅ Gets attention | Use both |
| User has tab closed | ❌ Not possible | ✅ Only option | Push only |
| Low-priority notifications | ✅ Badge only | ❌ Too intrusive | Badge only |
| High-priority alerts (security) | ✅ Toast + sound | ✅ Always send | Both |

---

#### 5.2.8 Decision Matrix

| Scenario | Strategy | Library / Tool | Notes |
|----------|----------|---------------|-------|
| **Real-time delivery** | WebSocket (primary) → SSE fallback → Polling fallback | Native `WebSocket` / `EventSource` | Exponential backoff reconnection with jitter |
| **Toast management** | Queue with max visible + priority + aggregation | Custom `useReducer` + `useToastQueue` | Max 3 visible; burst aggregation within 3-5s window |
| **Badge count** | Zustand store + optimistic increment/decrement | Zustand / Redux | Sync with server on reconnect and tab focus |
| **Notification grouping** | Client-side grouping by composite key (type + targetId) | Custom `groupNotifications()` | Server may pre-group for efficiency |
| **Large notification lists** | Virtualized list with dynamic row heights | `@tanstack/react-virtual` | Only 15-20 DOM nodes regardless of total count |
| **Push notifications** | Web Push API via Service Worker + VAPID | Native `PushManager` + Service Worker | Custom pre-prompt before browser permission dialog |
| **Offline badge** | Persist unread count in localStorage | Native `localStorage` | Show last-known count on app load before API responds |
| **Tab-hidden handling** | `visibilitychange` → REST backfill on re-focus | Native Page Visibility API | Fetch missed notifications via REST on tab re-focus |
| **Sound notifications** | Web Audio API with user preference toggle | Native `AudioContext` | Respect browser autoplay policy; require user gesture |

---

### 5.3 Reusability Strategy

*   **`ToastContainer`**: Reused across the app for any ephemeral messages (success, error, info), not just notifications. Configurable via props: `position`, `maxVisible`, `theme`.
*   **`Badge`**: Generic badge component reused for notification count, cart count, message count. Props: `count`, `max`, `color`.
*   **`RelativeTime`**: Shared utility component ("2h ago", "Just now") used in notifications, feed posts, comments. Configurable locale.
*   **`AvatarStack`**: Overlapping avatar circles used in notification groups, team views, collaboration features.
*   Design-system tokens for colors, spacing, typography.

---

### 5.4 Module Organization

```
src/
 ├── features/
 │    └── notifications/
 │         ├── components/
 │         │    ├── NotificationBell.tsx
 │         │    ├── NotificationPanel.tsx
 │         │    ├── NotificationList.tsx
 │         │    ├── NotificationItem.tsx
 │         │    ├── NotificationGroupItem.tsx
 │         │    ├── FilterTabs.tsx
 │         │    └── PushPermissionPrompt.tsx
 │         ├── toast/
 │         │    ├── ToastContainer.tsx
 │         │    ├── ToastItem.tsx
 │         │    └── useToastQueue.ts
 │         ├── transport/
 │         │    ├── NotificationSocket.ts
 │         │    ├── SSEFallback.ts
 │         │    └── PollingFallback.ts
 │         ├── hooks/
 │         │    ├── useNotificationSocket.ts
 │         │    ├── useUnreadCount.ts
 │         │    └── usePushPermission.ts
 │         ├── store/
 │         │    └── notificationStore.ts
 │         ├── api/
 │         │    └── notificationApi.ts
 │         ├── utils/
 │         │    ├── groupNotifications.ts
 │         │    ├── notificationSound.ts
 │         │    └── formatNotification.ts
 │         └── types/
 │              └── notification.types.ts
 ├── shared/
 │    ├── components/ (Badge, AvatarStack, RelativeTime)
 │    ├── hooks/ (useIntersection, useVisibilityChange)
 │    └── utils/ (debounce, localStorage helpers)
 ├── service-worker/
 │    └── sw.js (push event handler, notification click handler)
```

---

## 6. High Level Data Flow Explanation

### 6.1 Initial Load Flow

```
1. App loads → Shell renders with NotificationBell in header
     ↓
2. Bell reads cached unreadCount from localStorage (instant badge render)
     ↓
3. Fetch actual unread count: GET /api/notifications/count
     ↓
4. Update badge to server value (reconcile with cached)
     ↓
5. Establish WebSocket connection: wss://api.example.com/notifications/stream
     ↓
6. WebSocket connected → ready to receive real-time notifications
     ↓
7. (Deferred) NotificationPanel chunk is NOT loaded yet
   → loaded lazily on first bell click
```

---

### 6.2 User Interaction Flow

```
New notification arrives via WebSocket
     ↓
1. Parse notification → add to store (prepend to list)
     ↓
2. Increment unreadCount (store update → badge re-renders)
     ↓
3. Show toast (if priority is normal or high)
     ↓
4. Play notification sound (if user preference allows)
     ↓
5. Update document title: "(3) MyApp"
     ↓

User clicks bell → opens NotificationPanel
     ↓
6. Fetch notification list: GET /api/notifications?limit=20
     ↓
7. Render notification list (grouped, sorted by time)
     ↓
8. User clicks a notification item
     ↓
9. Mark as read (optimistic) → decrement badge
     ↓
10. Navigate to target resource (deep link)
     ↓
11. POST /api/notifications/:id/read (fire-and-forget)
```

---

### 6.3 Error and Retry Flow

*   **WebSocket disconnects**: Trigger reconnection with exponential backoff (1s, 2s, 4s, 8s... up to 30s). After 10 attempts, fall back to SSE, then polling.
*   **Notification list API fails**: Show cached list from last successful fetch + error banner with retry button.
*   **Mark-as-read API fails**: Keep optimistic state; queue failed mark-as-read requests and retry on next successful API call.
*   **Toast rendering fails**: Silently skip the toast — badge count is still correct, panel still works.
*   **Push subscription fails**: Log error; fall back to in-app only (no push). Retry subscription on next app load.

---

## 7. Data Modelling (Frontend Perspective)

### 7.1 Core Data Entities

*   **Notification** — an individual notification event (like, comment, follow, system alert)
*   **NotificationGroup** — a collection of related notifications displayed as one item
*   **NotificationPreference** — user's per-category and per-channel notification settings
*   **PushSubscription** — browser push subscription details

---

### 7.2 Data Shape

```ts
type NotificationType = 'like' | 'comment' | 'follow' | 'mention' | 'order_update'
  | 'system_alert' | 'friend_request' | 'message';

type NotificationPriority = 'high' | 'normal' | 'low';

type Notification = {
  id: string;
  type: NotificationType;
  priority: NotificationPriority;
  actorId: string;
  actorName: string;
  actorAvatar: string;
  targetId: string;                // the resource this notification is about
  targetType: string;              // 'post' | 'comment' | 'order' | 'profile'
  targetTitle: string;             // preview text (e.g., post excerpt)
  deepLink: string;                // URL to navigate to on click
  message: string;                 // pre-rendered display text
  isRead: boolean;
  createdAt: string;               // ISO 8601
  actions?: NotificationAction[];  // inline actions (Accept/Decline)
  imageUrl?: string;               // optional thumbnail (e.g., liked photo)
  groupKey?: string;               // server-provided grouping key
};

type NotificationAction = {
  id: string;
  label: string;                   // "Accept" | "Decline"
  actionType: 'accept' | 'decline' | 'view' | 'dismiss';
  endpoint: string;                // API endpoint to call
};

type NotificationPreference = {
  category: string;                // 'social' | 'transactional' | 'marketing'
  channels: {
    inApp: boolean;
    push: boolean;
    email: boolean;
    sms: boolean;
  };
};
```

---

### 7.3 Entity Relationships

*   **One-to-Many**: One user → many Notifications.
*   **Many-to-One**: Many Notifications → one NotificationGroup (when grouped by composite key).
*   **One-to-Many**: One user → many NotificationPreferences (one per category).
*   **Normalized storage**: Notifications stored in an array sorted by `createdAt`; groups computed as derived state.

---

### 7.4 UI Specific Data Models

```ts
// Derived view model for the notification panel
type NotificationPanelState = {
  isOpen: boolean;
  activeFilter: 'all' | 'unread';
  groups: NotificationGroup[];
  isLoading: boolean;
  hasMore: boolean;
  cursor: string | null;
};

// Derived view model for toast
type ToastViewModel = {
  id: string;
  message: string;
  type: NotificationType;
  priority: NotificationPriority;
  duration: number;
  groupKey?: string;
  action?: { label: string; onClick: () => void };
};

// Connection state
type ConnectionState = {
  transport: 'websocket' | 'sse' | 'polling' | 'disconnected';
  status: 'connected' | 'connecting' | 'reconnecting' | 'disconnected';
  reconnectAttempts: number;
  lastEventId: string | null;
};
```

---

## 8. State Management Strategy

### 8.1 State Classification

| State Type | Examples | Storage |
|---|---|---|
| **Global App State** | Unread count, connection status, user preferences | Zustand global store |
| **Feature State** | Notification list, grouped notifications, toast queue | Feature-level store (Zustand slice / React Query cache) |
| **Panel UI State** | Panel open/close, active filter tab, scroll position | Component-local state (useState) |
| **Derived State** | Grouped notifications, filtered list, display count | Computed selectors (Zustand selectors / useMemo) |
| **Transport State** | WebSocket connection status, last event ID, reconnect count | Transport manager (class instance) |

---

### 8.2 State Ownership

*   **NotificationStore** (Zustand) owns the notification list, unread count, and preferences. Shared between bell, panel, and toast.
*   **ToastQueue** (useReducer) owns the toast queue and visible toasts. Local to ToastContainer.
*   **NotificationPanel** owns filter state and scroll position locally.
*   **NotificationSocket** (class) owns transport state internally; exposes connection status to the store.
*   Prop drilling is minimal: Bell reads count from store → Panel reads list from store → Socket writes to store.

---

### 8.3 Persistence Strategy

*   **In-memory**: Notification list, toast queue, connection state (ephemeral).
*   **React Query / SWR cache**: Notification list with `staleTime: 2min`; refetch on panel open and window focus.
*   **localStorage**: Unread count (for instant badge on app load), notification preferences, push permission state, last seen timestamp.
*   **Service Worker**: Push subscription stored in the browser's push subscription manager.
*   **IndexedDB (optional)**: For offline notification storage in PWA mode.

---

## 9. High Level API Design (Frontend POV)

### 9.1 Required APIs

| API | Method | Description |
|-----|--------|-------------|
| `/api/notifications` | **GET** | Fetch paginated notification list |
| `/api/notifications/count` | **GET** | Fetch unread notification count |
| `/api/notifications/:id/read` | **POST** | Mark a single notification as read |
| `/api/notifications/read-all` | **POST** | Mark all notifications as read |
| `/api/notifications/:id/dismiss` | **DELETE** | Dismiss/delete a notification |
| `/api/notifications/:id/action` | **POST** | Execute an inline action (accept/decline) |
| `/api/notifications/preferences` | **GET** | Fetch notification preferences |
| `/api/notifications/preferences` | **PUT** | Update notification preferences |
| `/api/push/subscribe` | **POST** | Register push subscription |
| `/api/push/unsubscribe` | **POST** | Remove push subscription |
| `/api/notifications/stream` | **WebSocket** | Real-time notification delivery channel |

---

### 9.2 Real Time Delivery Channel Design (Deep Dive)

#### WebSocket Message Protocol

The WebSocket channel carries multiple message types, not just notification payloads. A structured message protocol is essential:

```tsx
type WSMessage =
  | { type: 'notification'; payload: Notification }
  | { type: 'count_update'; payload: { unreadCount: number } }
  | { type: 'read_receipt'; payload: { notificationId: string } }
  | { type: 'ping' }
  | { type: 'pong' };
```

**Why `count_update` as a separate message type?**
*   When notifications are marked as read from another device/tab, the server broadcasts a `count_update` to all connected clients for that user.
*   This keeps badge counts in sync across multiple tabs/devices without the client having to calculate.

**Keepalive (ping/pong):**
*   The client sends a `ping` every 30 seconds to keep the connection alive through proxies and load balancers.
*   If no `pong` is received within 5 seconds, assume the connection is dead and trigger reconnection.

```tsx
// Keepalive in NotificationSocket
private startKeepalive() {
  this.keepaliveTimer = setInterval(() => {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify({ type: 'ping' }));
      this.pongTimer = setTimeout(() => {
        // No pong received — connection is dead
        this.ws?.close();
      }, 5000);
    }
  }, 30_000);
}
```

#### Message Deduplication

Messages can be duplicated in several scenarios:
*   Reconnection with `lastEventId` — server replays from the last known event, but some may overlap with already-received events.
*   Multiple tabs — each tab has its own connection; the user might see duplicates across tabs.

**Solution: Deduplicate by notification ID on the client.**

```tsx
const seenEventIds = new Set<string>();

function processIncomingNotification(notif: Notification) {
  if (seenEventIds.has(notif.id)) {
    return; // already processed — skip
  }
  seenEventIds.add(notif.id);

  // Limit set size to prevent memory leak
  if (seenEventIds.size > 1000) {
    const entries = Array.from(seenEventIds);
    entries.slice(0, 500).forEach((id) => seenEventIds.delete(id));
  }

  // Process the notification
  notificationStore.addNotification(notif);
  toastQueue.enqueue(notifToToast(notif));
}
```

---

### 9.3 Request and Response Structure

**GET /api/notifications?limit=20&cursor=abc123&filter=unread**

```json
// Response
{
  "notifications": [
    {
      "id": "n_001",
      "type": "like",
      "priority": "normal",
      "actorId": "u_456",
      "actorName": "Alice",
      "actorAvatar": "https://cdn.example.com/avatars/u_456.jpg",
      "targetId": "post_789",
      "targetType": "post",
      "targetTitle": "My thoughts on React Server Components",
      "deepLink": "/posts/post_789",
      "message": "Alice liked your post",
      "isRead": false,
      "createdAt": "2026-03-17T10:30:00Z",
      "groupKey": "like:post_789",
      "imageUrl": "https://cdn.example.com/posts/post_789_thumb.jpg"
    },
    {
      "id": "n_002",
      "type": "friend_request",
      "priority": "high",
      "actorId": "u_789",
      "actorName": "Bob",
      "actorAvatar": "https://cdn.example.com/avatars/u_789.jpg",
      "targetId": "u_789",
      "targetType": "profile",
      "targetTitle": "",
      "deepLink": "/profile/u_789",
      "message": "Bob sent you a friend request",
      "isRead": false,
      "createdAt": "2026-03-17T10:25:00Z",
      "actions": [
        { "id": "a_001", "label": "Accept", "actionType": "accept", "endpoint": "/api/friends/u_789/accept" },
        { "id": "a_002", "label": "Decline", "actionType": "decline", "endpoint": "/api/friends/u_789/decline" }
      ]
    }
  ],
  "pagination": {
    "nextCursor": "c_def456",
    "hasMore": true
  },
  "unreadCount": 7
}
```

**GET /api/notifications/count**

```json
// Response
{
  "unreadCount": 7
}
```

**POST /api/notifications/read-all**

```json
// Request
{}

// Response
{
  "success": true,
  "updatedCount": 7
}
```

**WebSocket Message (server → client)**

```json
{
  "type": "notification",
  "eventId": "evt_abc123",
  "payload": {
    "id": "n_003",
    "type": "comment",
    "priority": "normal",
    "actorId": "u_111",
    "actorName": "Carol",
    "actorAvatar": "https://cdn.example.com/avatars/u_111.jpg",
    "targetId": "post_789",
    "targetType": "post",
    "targetTitle": "My thoughts on React Server Components",
    "deepLink": "/posts/post_789#comment_222",
    "message": "Carol commented on your post",
    "isRead": false,
    "createdAt": "2026-03-17T11:00:00Z",
    "groupKey": "comment:post_789"
  }
}
```

---

### 9.4 Error Handling and Status Codes

| Status | Scenario | Frontend Handling |
|--------|----------|-------------------|
| `200` | Success | Render data |
| `304` | Not Modified | Use cached version |
| `401` | Unauthorized | Redirect to login; close WebSocket |
| `404` | Notification deleted / not found | Remove from local list silently |
| `429` | Rate limit (too many mark-as-read calls) | Backoff + batch pending requests |
| `500` | Server error | Show error toast; retry with exponential backoff |
| `503` | Service unavailable | Fall back to cached data; show stale indicator |

---

## 10. Caching Strategy

### 10.1 What to Cache

*   **Unread count**: Cached in localStorage for instant badge on app load; reconciled with server on connect.
*   **Notification list**: First page cached in React Query for instant panel open; background refetch for freshness.
*   **User preferences**: Cached in localStorage; rarely changes.
*   **Push permission state**: Cached to avoid unnecessary permission checks.

### 10.2 Where to Cache

| Data | Cache Location |
|------|----------------|
| Unread count | localStorage + Zustand store |
| Notification list (first page) | React Query in-memory cache |
| Notification list (older pages) | React Query cache (evicted on memory pressure) |
| User preferences | localStorage |
| Avatar images | Browser HTTP cache + CDN |
| Notification sounds | Browser HTTP cache (long-lived) |

### 10.3 Cache Invalidation

*   **Unread count**: Invalidated on every WebSocket notification, mark-as-read, and mark-all-as-read. Reconciled on tab focus via REST.
*   **Notification list**: Invalidated when panel opens (background refetch with `staleTime: 2min`). New real-time notifications prepended to cache.
*   **Preferences**: Invalidated on explicit save. Rarely changes.
*   **Sounds/Assets**: Long `Cache-Control: max-age=31536000` with content hash in URL for versioning.

---

## 11. CDN and Asset Optimization

*   **Avatar delivery**: User avatars served via CDN at small resolution (48x48px for notification items). Use WebP with JPEG fallback.
*   **Notification sounds**: Pre-cached audio files served from CDN. Small MP3/WAV files (< 50KB).
*   **Icon assets**: Notification type icons (like, comment, follow, system) as inline SVG or shared icon sprite.
*   **Thumbnail images**: Post thumbnails in notifications (e.g., "Alice liked your photo") served at 64x64px from CDN.
*   **Cache headers**: `Cache-Control: public, max-age=86400` for avatars (users rarely change photos daily). `max-age=31536000, immutable` for sound files and icons (versioned by hash).

---

## 12. Rendering Strategy

*   **App Shell (Header with Bell)**: SSR for the shell → badge shows cached count immediately on page load. Fast FCP.
*   **Notification Panel**: Fully **CSR** — loaded as a **lazy-loaded chunk** on first bell click.
    *   `React.lazy(() => import('./NotificationPanel'))` with `Suspense` fallback (skeleton dropdown).
*   **ToastContainer**: Loaded in the **initial bundle** — must be ready to display toasts at any time.
*   **Service Worker**: Registered on app boot for push notification handling. Runs independently of the main page lifecycle.
*   **No SSR for panel/toasts**: These are interactive-only, real-time components with no SEO value.
*   **Portal rendering**: NotificationPanel and ToastContainer rendered in React Portals to escape parent overflow/z-index stacking contexts.

---

## 13. Cross Cutting Non Functional Concerns

### 13.1 Security

*   **Authenticated WebSocket**: Connection includes auth token (via query param with short-lived token or cookie). Server validates on connection and on each message.
*   **No cross-user leakage**: Server ensures notifications are delivered only to the intended recipient. Client should never receive another user's notification.
*   **XSS mitigation**: All notification text (especially user-generated actor names, post titles) is sanitized via DOMPurify before rendering.
*   **CSRF**: All POST requests (mark-as-read, preferences) include CSRF tokens.
*   **Push security**: Push subscription uses VAPID keys (not plain text). Push payloads are encrypted end-to-end by the browser's push service.
*   **Token rotation**: If WebSocket auth token expires, server sends a `token_expired` message → client refreshes the token and reconnects.

---

### 13.2 Accessibility

*   **Notification Bell**:
    *   `aria-label="Notifications, 3 unread"` (dynamic label with count).
    *   `aria-expanded` reflects panel state.
    *   `aria-haspopup="true"` indicates dropdown behavior.
    *   Keyboard: `Enter` / `Space` toggles panel; `Escape` closes it.
*   **Notification Panel**:
    *   `role="region"` or `role="dialog"` with `aria-label="Notification panel"`.
    *   `role="list"` with `role="listitem"` for each notification.
    *   Arrow key navigation between notifications.
    *   `Tab` moves to action buttons within a notification.
    *   Focus trapped inside panel while open.
*   **Toast Notifications**:
    *   `role="status"` container with `aria-live="polite"` for normal priority.
    *   `role="alert"` with `aria-live="assertive"` for high priority.
    *   Screen reader announces toast text automatically.
    *   `aria-relevant="additions removals"` to announce when toasts appear/disappear.
*   **Reduced motion**: Respect `prefers-reduced-motion` — disable toast slide animations, use instant show/hide.
*   **Color contrast**: Unread dot and badge use sufficient contrast (WCAG AA 4.5:1 minimum).

---

### 13.3 Performance Optimization

*   **Code splitting**: Notification panel chunk loaded only on first bell click (~30-50KB). Toast system in initial bundle (~5KB).
*   **Virtualized notification list**: Only 15-20 DOM nodes regardless of total notification count.
*   **Debounced mark-as-read**: Batch mark-as-read API calls when user scrolls through notifications (debounce 1s).
*   **WebSocket message batching**: Server can batch multiple notifications into a single WebSocket frame to reduce frame overhead.
*   **Efficient grouping**: `groupNotifications()` runs as a memoized selector — only recalculates when the notification array reference changes.
*   **Abort stale fetches**: If the user closes the panel mid-fetch, cancel the in-flight request via `AbortController`.
*   **Lazy avatar loading**: Notification item avatars use `loading="lazy"` — only load when scrolled into view.

---

### 13.4 Observability and Reliability

*   **Error Boundaries**: Wrap `NotificationPanel` and `ToastContainer` in error boundaries — a crash in notifications shouldn't break the entire app.
*   **Logging**: Track key events:
    *   Notification received (via WebSocket), notification displayed (toast shown), notification clicked, notification dismissed.
    *   WebSocket connection events (connected, disconnected, reconnected, fallback triggered).
    *   Push permission prompt shown, granted, denied.
    *   Push notification shown, clicked, dismissed.
*   **Performance monitoring**:
    *   Track notification delivery latency (server event time → toast render time).
    *   Track panel open latency (bell click → list rendered).
    *   Track WebSocket reconnection frequency and duration.
*   **Feature flags**: Gate new features (e.g., notification grouping, push notifications, rich notification cards) behind flags.
*   **Alerting**: Monitor WebSocket disconnect rates; alert if fallback rate exceeds threshold.

---

## 14. Edge Cases and Tradeoffs

| Edge Case | Handling |
|-----------|----------|
| **Notification burst (100+ in seconds)** | Aggregate same-type notifications in toast queue; group in notification list; throttle badge animation to avoid visual noise. |
| **Multiple tabs open** | Each tab has its own WebSocket connection. Use `BroadcastChannel` API to sync mark-as-read and badge count across tabs to avoid drift. |
| **Tab hidden for hours** | On re-focus: verify WebSocket connection → reconnect if needed → REST backfill missed notifications → reconcile badge count with server. |
| **User has 10,000+ notifications** | Virtualized list renders only visible items. Older notifications evicted from memory; fetched via cursor pagination on scroll. |
| **Network loss during mark-as-read** | Queue failed requests in memory; retry on reconnection. Optimistic UI remains (item shown as read). |
| **Push notification clicked while app is open** | Service Worker `notificationclick` sends message to open tab via `client.postMessage()` → tab navigates without opening a duplicate. |
| **Notification for deleted resource** | Deep link returns 404 → show "This content is no longer available" page. Remove notification from local list. |
| **User denies push permission** | Fall back to in-app only. Show a subtle "Enable push for alerts when you're away" prompt in settings. |
| **Stale notification count** | On every tab focus (`visibilitychange`), re-fetch `/api/notifications/count` to correct any drift. |
| **Accessibility with screen readers** | `aria-live` region announces new toasts. Panel uses `role="list"` with keyboard navigation. Count is part of bell's `aria-label`. |

### Key Tradeoffs

| Decision | Tradeoff |
|----------|----------|
| **WebSocket as primary transport** | Lowest latency and bidirectional, but requires WebSocket server infrastructure and handling reconnection/fallback. SSE would be simpler for unidirectional. |
| **Client-side notification grouping** | Flexible and responsive to real-time updates, but uses client CPU. Server-side grouping would offload work but adds API complexity. |
| **Toast queue with max 3 visible** | Prevents screen flooding, but some notifications may be delayed in the queue during bursts. Badge count is always correct regardless. |
| **Lazy-load notification panel** | Smaller initial bundle, but ~100-200ms delay on first bell click. Mitigated with preload on bell hover. |
| **localStorage for unread count** | Instant badge on app load, but can be stale across devices. Always reconciled with server within seconds. |
| **Push notifications** | Essential for re-engagement when tab is closed, but requires Service Worker + VAPID setup. Users may deny permission. |
| **`BroadcastChannel` for cross-tab sync** | Keeps tabs consistent, but adds complexity. Without it, tabs diverge until next server sync. |

---

## 15. Summary and Future Improvements

### Key Architectural Decisions

1.  **WebSocket with SSE and Polling fallback chain** — ensures real-time delivery across all network conditions.
2.  **Prioritized toast queue with burst aggregation** — prevents UI flooding while ensuring important notifications are seen.
3.  **Optimistic badge updates with server reconciliation** — badge feels instant, server is source of truth.
4.  **Client-side notification grouping** — flexible, responsive grouping without backend dependency.
5.  **Lazy-loaded panel with virtualized list** — fast initial load, smooth scrolling regardless of notification volume.
6.  **Web Push via Service Worker** — notifications reach users even when the app is closed.

### Possible Future Enhancements

*   **Rich notification cards**: Inline media previews, interactive polls, and embedded action buttons within notifications.
*   **Notification center as a full page**: For power users with heavy notification volume, a dedicated `/notifications` page with filtering, search, and bulk actions.
*   **Smart notification batching**: ML-driven batching that groups less-urgent notifications into a digest, reducing interruption frequency.
*   **Cross-device read sync**: Real-time sync of read state across mobile app, desktop, and web via shared WebSocket events.
*   **Notification scheduling**: "Do not disturb" mode that queues notifications and delivers them at the user's preferred time.
*   **Web Workers for grouping**: Offload `groupNotifications()` to a Web Worker for users with 10K+ notifications.
*   **SharedWorker for multi-tab**: Use a `SharedWorker` to maintain a single WebSocket connection shared across all tabs, reducing server connections.

---

### Endpoint Summary

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/notifications` | GET | Fetch paginated notification list |
| `/api/notifications/count` | GET | Fetch unread count |
| `/api/notifications/:id/read` | POST | Mark notification as read |
| `/api/notifications/read-all` | POST | Mark all as read |
| `/api/notifications/:id/dismiss` | DELETE | Dismiss notification |
| `/api/notifications/:id/action` | POST | Execute inline action |
| `/api/notifications/preferences` | GET/PUT | Get or update preferences |
| `/api/push/subscribe` | POST | Register push subscription |
| `/api/push/unsubscribe` | POST | Remove push subscription |
| `/api/notifications/stream` | WebSocket | Real-time delivery channel |

---

### Complete Notification Flow

| Direction | Mechanism | Trigger | Endpoint | Action |
|-----------|-----------|---------|----------|--------|
| Initial Load | REST | On mount | `/api/notifications/count` | Show badge count |
| Real Time | WebSocket | Server push | `wss://.../notifications/stream` | Toast + badge increment |
| Open Panel | REST | Bell click | `/api/notifications?limit=20` | Render notification list |
| Mark Read | REST (optimistic) | Click notification | `/api/notifications/:id/read` | Decrement badge, update item |
| Mark All Read | REST (optimistic) | Click "Mark all" | `/api/notifications/read-all` | Reset badge to 0 |
| Push Subscribe | REST | User grants permission | `/api/push/subscribe` | Register push endpoint |
| Push Delivery | Web Push | Tab closed | Push Service → Service Worker | Native browser notification |
| Tab Refocus | REST | `visibilitychange` | `/api/notifications/count` | Reconcile badge count |
| Fallback | SSE / Polling | WebSocket failure | `/api/notifications/stream` (SSE) or `/api/notifications` (poll) | Maintain real-time delivery |

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design
