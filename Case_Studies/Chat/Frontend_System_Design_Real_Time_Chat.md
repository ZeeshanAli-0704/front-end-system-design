# Frontend System Design: Real Time Chat Application

- [Frontend System Design: Real Time Chat Application](#frontend-system-design-real-time-chat-application)
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
    - [5.2 Message List Rendering and Real Time Sync Strategy (Deep Dive)](#52-message-list-rendering-and-real-time-sync-strategy-deep-dive)
      - [5.2.1 WebSocket Connection and Message Transport](#521-websocket-connection-and-message-transport)
      - [5.2.2 Optimistic Message Sending](#522-optimistic-message-sending)
      - [5.2.3 Virtualized Message List for Large Conversations](#523-virtualized-message-list-for-large-conversations)
      - [5.2.4 Reverse Infinite Scroll for Message History](#524-reverse-infinite-scroll-for-message-history)
      - [5.2.5 Scroll Position Management and Anchoring](#525-scroll-position-management-and-anchoring)
      - [5.2.6 Typing Indicators](#526-typing-indicators)
      - [5.2.7 Message Status Indicators (Sent, Delivered, Read)](#527-message-status-indicators-sent-delivered-read)
      - [5.2.8 Rich Media Messages (Images, Files, Links)](#528-rich-media-messages-images-files-links)
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
    - [9.2 Message Delivery Guarantees and Ordering (Deep Dive)](#92-message-delivery-guarantees-and-ordering-deep-dive)
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

*   A real-time chat application is a **core communication feature** found in platforms like WhatsApp Web, Slack, Discord, Facebook Messenger, and Microsoft Teams.
*   It enables users to exchange text messages, media (images, files), and reactions in real time within one-on-one or group conversations.
*   Target users: all platform users — individuals communicating privately, teams collaborating in group channels, and customer support agents handling service chats.
*   Primary use case: sending and receiving messages in real time with sub-second delivery latency, presence awareness, and message status tracking (sent, delivered, read).

---

### 1.2 Key User Personas

*   **Casual User**: Sends 1-on-1 messages, shares photos/links, expects instant delivery and read receipts. Uses mobile web primarily.
*   **Power User / Team Collaborator**: Participates in multiple group chats simultaneously, types rapidly, uses keyboard shortcuts, expects message search and thread replies.
*   **Support Agent**: Handles multiple concurrent conversations, needs quick context switching, canned responses, and conversation assignment indicators.

---

### 1.3 Core User Flows (High Level)

*   **Sending and Receiving Messages (Primary Flow)**:
    1.  User opens a conversation from the conversation list.
    2.  Previous messages load (most recent at the bottom).
    3.  User types a message in the composer → presses Enter or taps Send.
    4.  Message appears instantly at the bottom of the message list (optimistic update) with a "sending" indicator.
    5.  Server acknowledges → status changes to "sent" ✓, then "delivered" ✓✓ when recipient's client receives it, then "read" (blue ✓✓) when recipient opens the conversation.
    6.  Recipient sees the message appear in real time at the bottom of their message list.

*   **Browsing Conversation List (Secondary Flow)**:
    1.  User opens the chat app → conversation list loads, sorted by most recent activity.
    2.  Each conversation shows: avatar, name, last message preview, timestamp, unread count badge.
    3.  A new incoming message bumps that conversation to the top of the list and increments the unread badge.
    4.  User taps a conversation → chat view opens.

*   **Sharing Media (Secondary Flow)**:
    1.  User taps the attachment icon → file picker opens.
    2.  Selects an image/file → preview appears in the composer.
    3.  User taps Send → media uploads in the background with a progress indicator.
    4.  Upload completes → message with media thumbnail appears in the chat.
    5.  Recipient can tap the thumbnail to view the full-resolution image or download the file.

---

## 2. Requirements

### 2.1 Functional Requirements

*   **Conversation List**:
    *   Display a vertical list of conversations sorted by most-recent activity.
    *   Show avatar, name (or group name), last message preview, timestamp, and unread badge count.
    *   Search/filter conversations by name or content.
    *   Create new 1-on-1 or group conversations.
*   **Message View**:
    *   Display messages in chronological order (oldest at top, newest at bottom).
    *   Distinguish between sent and received messages (left/right alignment).
    *   Support text messages, images, files, links with OG previews, and emoji/reactions.
    *   Show message status indicators: sending → sent → delivered → read.
    *   Show date separators between messages from different days.
    *   Load older messages on scroll-up (reverse infinite scroll).
*   **Message Composer**:
    *   Text input with multiline support (Shift+Enter for newline, Enter to send).
    *   Attachment button for images and files.
    *   Emoji picker.
    *   Typing indicator broadcast (show "Alice is typing..." to other participants).
*   **Real Time Features**:
    *   Instant message delivery via WebSocket.
    *   Typing indicators (visible to all conversation participants).
    *   Online/offline presence status (green dot).
    *   Read receipts (who has read the message and when).
*   **Group Chat**:
    *   Multiple participants in a single conversation.
    *   Group name and avatar.
    *   Member list with online/offline status.
    *   Admin controls (add/remove members).

---

### 2.2 Non Functional Requirements

*   **Performance**: Message delivery latency < 200ms (WebSocket round trip); message list renders at 60fps during scroll; conversation list opens in < 300ms; composer input has zero perceived lag.
*   **Scalability**: Support conversations with 10,000+ messages (history); conversation list with 500+ conversations; group chats with 100+ participants.
*   **Availability**: Graceful degradation — queue outgoing messages when offline, deliver when reconnected; show cached conversations if network is down.
*   **Security**: End-to-end encryption (optional, client-side); XSS prevention on message content; CSRF protection on all POST requests; authenticated WebSocket connections; media served via signed CDN URLs.
*   **Accessibility**: Full keyboard navigation in conversation list and message view; screen reader support with ARIA live regions for new messages; focus management on conversation switch; sufficient color contrast.
*   **Device Support**: Mobile web (primary), desktop web (responsive), low-end devices with limited memory.
*   **i18n**: RTL layout support; localized timestamps ("2 hours ago"); locale-aware date formatting; translated UI strings.

---

## 3. Scope Clarification (Interview Scoping)

### 3.1 In Scope

*   Conversation list component with real-time updates.
*   Message view with real-time message delivery and status tracking.
*   Message composer with text, media attachments, and emoji.
*   WebSocket-based real-time transport with reconnection handling.
*   Typing indicators and presence (online/offline) status.
*   Optimistic message sending with retry on failure.
*   Reverse infinite scroll for message history.
*   State management for conversations and messages.
*   API design from the frontend perspective.
*   Performance optimization for message-heavy views.

---

### 3.2 Out of Scope

*   Backend message routing, queueing, and fan-out architecture.
*   End-to-end encryption implementation (Signal Protocol / Double Ratchet).
*   Voice and video calling.
*   Message search across all conversations (full-text search).
*   Rich text formatting in messages (Markdown, code blocks).
*   Bot and webhook integrations.
*   Admin moderation panel.

---

### 3.3 Assumptions

*   User is authenticated; auth token is available via HTTP-only cookie or Authorization header.
*   Backend provides a WebSocket endpoint for real-time message delivery and presence.
*   Messages are stored server-side; frontend fetches history via REST and receives new messages via WebSocket.
*   Media (images, files) are uploaded to a backend service that returns a CDN URL.
*   The chat feature may be a standalone app or embedded within a larger platform (e.g., a chat panel in a SaaS app).

---

## 4. High Level Frontend Architecture

### 4.1 Overall Approach

*   **SPA** (Single Page Application) with client-side routing between conversation list and chat views.
*   **CSR** for all chat UI — messages, composer, presence are all real-time, interactive components that don't benefit from SSR.
*   **SSR** optionally for the app shell (header, sidebar layout) for fast FCP.
*   WebSocket connection is **established on app boot** and shared across all conversations.
*   Message view and conversation list are **always resident in memory** (not unmounted on navigation) to preserve scroll position and avoid refetching.

---

### 4.2 Major Architectural Layers

```
┌──────────────────────────────────────────────────────────────┐
│  UI Layer                                                    │
│  ┌───────────────────┐   ┌────────────────────────────────┐  │
│  │ ConversationList  │   │ ChatView                       │  │
│  │ ┌───────────────┐ │   │  ┌──────────────────────────┐  │  │
│  │ │ ConvoItem[]   │ │   │  │ MessageList (virtual)    │  │  │
│  │ │ ┌───────────┐ │ │   │  │  ┌────────────────────┐  │  │  │
│  │ │ │ Avatar    │ │ │   │  │  │ MessageBubble[]    │  │  │  │
│  │ │ │ Preview   │ │ │   │  │  │ DateSeparator[]    │  │  │  │
│  │ │ │ Badge     │ │ │   │  │  └────────────────────┘  │  │  │
│  │ │ └───────────┘ │ │   │  └──────────────────────────┘  │  │
│  │ └───────────────┘ │   │  ┌──────────────────────────┐  │  │
│  │                   │   │  │ MessageComposer          │  │  │
│  │ ┌───────────────┐ │   │  │ (Input + Attach + Emoji) │  │  │
│  │ │ SearchBar     │ │   │  └──────────────────────────┘  │  │
│  │ └───────────────┘ │   │  ┌──────────────────────────┐  │  │
│  └───────────────────┘   │  │ TypingIndicator          │  │  │
│                          │  └──────────────────────────┘  │  │
│                          └────────────────────────────────┘  │
├──────────────────────────────────────────────────────────────┤
│  State Management Layer                                      │
│  (Conversation Store, Message Store, Presence Store,         │
│   Typing State, Unread Counts, Draft Messages)               │
├──────────────────────────────────────────────────────────────┤
│  Real Time Transport Layer                                   │
│  (WebSocket Manager, Reconnection Logic,                     │
│   Message Queue, Event Routing, Presence Channel)            │
├──────────────────────────────────────────────────────────────┤
│  API and Data Access Layer                                   │
│  (REST Client, File Upload Service,                          │
│   Retry Logic, Request Deduplication)                        │
├──────────────────────────────────────────────────────────────┤
│  Shared / Utility Layer                                      │
│  (Date formatting, Linkify, Emoji parser,                    │
│   Sound player, Encryption utils, Analytics)                 │
└──────────────────────────────────────────────────────────────┘
```

---

### 4.3 External Integrations

*   **WebSocket Server**: Real-time message delivery, typing indicators, presence updates (`wss://api.example.com/chat/ws`).
*   **CDN**: Serves user avatars, shared images, file attachments, and link preview thumbnails.
*   **Media Upload Service**: Handles image/file uploads, generates thumbnails, returns CDN URLs.
*   **Analytics SDK**: Track message sent/received events, conversation open events, and engagement metrics.
*   **Backend Services**: Conversation list API, message history API, user profile/presence API, contact search API.
*   **Link Preview Service**: Backend-generated OG metadata (title, description, thumbnail) for shared URLs.
*   **Push Service**: Web Push API for notifications when the tab is not active (Service Worker).

---

## 5. Component Design and Modularization

### 5.1 Component Hierarchy

```
ChatApp
 ├── Sidebar
 │    ├── SidebarHeader (user avatar, settings, new chat button)
 │    ├── ConversationSearch (filter conversations)
 │    └── ConversationList (scrollable)
 │         └── ConversationItem[] (repeated)
 │              ├── Avatar (with online/offline dot)
 │              ├── ConvoPreview (name + last message)
 │              ├── Timestamp ("2h ago")
 │              └── UnreadBadge (count pill)
 │
 ├── ChatView (main panel)
 │    ├── ChatHeader
 │    │    ├── ConvoAvatar + ConvoName
 │    │    ├── PresenceStatus ("Online" / "Last seen 2h ago")
 │    │    └── ActionButtons (call, search, info)
 │    ├── MessageList (scrollable, virtualized)
 │    │    ├── DateSeparator ("Today", "March 16, 2026")
 │    │    └── MessageBubble[] (repeated)
 │    │         ├── MessageAvatar (group chat only — sender)
 │    │         ├── MessageContent
 │    │         │    ├── TextMessage (with link detection)
 │    │         │    ├── ImageMessage (thumbnail + lightbox)
 │    │         │    ├── FileMessage (icon + filename + size)
 │    │         │    └── LinkPreviewCard (OG metadata)
 │    │         ├── MessageTimestamp ("10:30 AM")
 │    │         ├── MessageStatus (✓ sent, ✓✓ delivered, blue ✓✓ read)
 │    │         └── MessageReactions (emoji reactions bar)
 │    ├── TypingIndicator ("Alice is typing...")
 │    └── MessageComposer
 │         ├── AttachmentButton (file/image picker)
 │         ├── TextInput (auto-grow textarea)
 │         ├── EmojiPicker (lazy-loaded popover)
 │         └── SendButton
 │
 └── InfoPanel (optional — toggled via header)
      ├── ConvoDetails (name, created date)
      ├── MemberList (group chats)
      │    └── MemberItem[] (avatar + name + role)
      └── SharedMedia (images, files shared in conversation)
```

---

### 5.2 Message List Rendering and Real Time Sync Strategy (Deep Dive)

The message list is the **core of the entire chat application** — a vertically scrolling, reverse-chronologically loaded list of messages with real-time additions. Getting this right matters because:
*   Users spend **90%+ of their time** looking at the message list.
*   Messages arrive in **real time** and must appear instantly at the bottom without disrupting scroll position.
*   Conversation history can contain **thousands of messages** — loading all at once is not feasible.
*   The list scrolls in **reverse** — older messages load on scroll-up, unlike a typical feed where older content loads on scroll-down.
*   Message heights are **variable** — text-only messages are short, images are tall, file attachments vary.
*   Every scroll frame must render at **60fps** — jank destroys the "instant messaging" feel.

---

#### 5.2.1 WebSocket Connection and Message Transport

##### Why WebSocket, Not SSE or Polling

Chat is a **bidirectional, high-frequency, low-latency** use case — the definitive scenario for WebSocket:
*   **Client sends messages** to the server (not just receives).
*   **Typing indicators** flow both ways every 1-3 seconds.
*   **Presence updates** (online/offline/typing) are frequent, small payloads.
*   **Message delivery is latency-critical** — users expect sub-200ms round trip.

| Feature | WebSocket | SSE | Polling |
|---------|-----------|-----|---------|
| **Direction** | Bidirectional | Server → Client only | Client → Server |
| **Latency** | ~50-100ms (frame-level) | ~100-200ms (HTTP chunk) | N × poll interval (seconds) |
| **Typing indicators** | Native (send via same connection) | Need separate POST per keystroke | Impractical (too frequent) |
| **Presence** | Real-time heartbeat | Possible but one-way | Polling presence API separately |
| **Message sending** | Send via same socket | Need separate REST POST | REST POST per message |
| **Connection overhead** | 1 persistent TCP connection | 1 persistent HTTP connection | New HTTP request every poll |

**WebSocket is the only viable primary transport for chat.** SSE could work as a fallback for receive-only, but you'd still need REST for sending — adding complexity and latency.

##### WebSocket Message Protocol

All chat events flow through a single multiplexed WebSocket connection:

```tsx
// Outgoing (client → server)
type ClientMessage =
  | { type: 'message.send'; payload: { conversationId: string; content: string; clientId: string; attachments?: Attachment[] } }
  | { type: 'message.read'; payload: { conversationId: string; messageId: string } }
  | { type: 'typing.start'; payload: { conversationId: string } }
  | { type: 'typing.stop'; payload: { conversationId: string } }
  | { type: 'presence.heartbeat' }
  | { type: 'ping' };

// Incoming (server → client)
type ServerMessage =
  | { type: 'message.new'; payload: Message }
  | { type: 'message.status'; payload: { messageId: string; status: 'delivered' | 'read'; readBy?: string } }
  | { type: 'typing.update'; payload: { conversationId: string; userId: string; isTyping: boolean } }
  | { type: 'presence.update'; payload: { userId: string; status: 'online' | 'offline'; lastSeen?: string } }
  | { type: 'conversation.updated'; payload: Partial<Conversation> }
  | { type: 'pong' };
```

**Why a single multiplexed connection?**
*   Browsers limit concurrent connections per domain (~6 for HTTP/1.1). A single WebSocket leaves room for other HTTP requests (file uploads, API calls).
*   One connection = one set of reconnection logic, one auth handshake, one heartbeat timer.
*   The `type` field acts as a **routing key** — the client routes each message type to the appropriate handler/store.

##### Connection Manager Implementation

```tsx
class ChatSocket {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 15;
  private baseDelay = 1000;
  private maxDelay = 30000;
  private messageQueue: ClientMessage[] = [];  // queue messages while disconnected
  private handlers = new Map<string, Set<(payload: any) => void>>();

  connect(authToken: string) {
    const url = `wss://api.example.com/chat/ws?token=${authToken}`;
    this.ws = new WebSocket(url);

    this.ws.onopen = () => {
      this.reconnectAttempts = 0;
      // Flush queued messages
      while (this.messageQueue.length > 0) {
        const msg = this.messageQueue.shift()!;
        this.ws!.send(JSON.stringify(msg));
      }
      this.startHeartbeat();
    };

    this.ws.onmessage = (event) => {
      const data: ServerMessage = JSON.parse(event.data);
      // Route to registered handlers
      const typeHandlers = this.handlers.get(data.type);
      if (typeHandlers) {
        typeHandlers.forEach((handler) => handler(data.payload));
      }
    };

    this.ws.onclose = (event) => {
      this.stopHeartbeat();
      if (event.code !== 1000) {
        this.scheduleReconnect();
      }
    };

    this.ws.onerror = () => this.ws?.close();
  }

  send(message: ClientMessage) {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(message));
    } else {
      // Queue for delivery when connection is restored
      this.messageQueue.push(message);
    }
  }

  on(type: string, handler: (payload: any) => void) {
    if (!this.handlers.has(type)) {
      this.handlers.set(type, new Set());
    }
    this.handlers.get(type)!.add(handler);
    return () => this.handlers.get(type)?.delete(handler);  // unsubscribe
  }

  private scheduleReconnect() {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) return;

    const delay = Math.min(
      this.baseDelay * Math.pow(2, this.reconnectAttempts) + Math.random() * 1000,
      this.maxDelay
    );

    setTimeout(() => {
      this.reconnectAttempts++;
      this.connect(this.authToken);
    }, delay);
  }

  private startHeartbeat() {
    this.heartbeatTimer = setInterval(() => {
      this.send({ type: 'ping' });
    }, 30_000);
  }
}
```

**Key design decisions:**

| Decision | Rationale |
|----------|-----------|
| **Message queue during disconnect** | Messages typed while offline are queued in memory and flushed on reconnect. User doesn't lose their messages. |
| **Event-based handler routing** | `on('message.new', handler)` pattern decouples the socket from stores. Stores subscribe to events they care about. |
| **Single auth token in URL** | WebSocket doesn't support custom headers in browsers. A short-lived token in the query string is the standard workaround. Token is rotated on reconnect. |
| **Exponential backoff with jitter** | Prevents thundering herd on server restart. Random jitter (0-1s) spreads reconnections. |

---

#### 5.2.2 Optimistic Message Sending

##### The Problem — Network Latency Makes Chat Feel Slow

When the user hits Send, the message must travel:
```
User types → Client → Internet → Server → Database → Server → Internet → Client (confirm)
                                 ↓                    ↓
                          also: Server → Internet → Recipient
```

This round trip (client → server → client ACK) takes **100-500ms** depending on network conditions. If the message only appears after the server confirms, the user experiences a **perceptible delay** — the chat feels sluggish, unlike native messaging apps.

##### The Solution — Show It Immediately

**Optimistic sending** means: **render the message in the UI instantly, before the server confirms it.** The server acknowledgment arrives later and updates the status indicator.

```
Timeline:
  t=0ms:   User hits Send
  t=1ms:   Message appears in the list with status "sending" (grey clock icon ⏳)
  t=150ms: Server acknowledges → status updates to "sent" (single check ✓)
  t=300ms: Recipient receives → status updates to "delivered" (double check ✓✓)
  t=∞:     Recipient opens conversation → status updates to "read" (blue ✓✓)
```

**The user sees zero delay** — the message appears at t=1ms.

##### Implementation — Optimistic Send Flow

```tsx
function useSendMessage(conversationId: string) {
  const addMessage = useMessageStore((s) => s.addMessage);
  const updateMessageStatus = useMessageStore((s) => s.updateMessageStatus);
  const chatSocket = useChatSocket();

  const sendMessage = useCallback(
    (content: string, attachments?: Attachment[]) => {
      // Generate a client-side ID for correlation
      const clientId = crypto.randomUUID();

      // 1. Create optimistic message — appears instantly in the UI
      const optimisticMessage: Message = {
        id: clientId,            // temporary ID — server will assign real ID
        conversationId,
        senderId: currentUserId,
        content,
        attachments,
        status: 'sending',       // ⏳ grey clock
        createdAt: new Date().toISOString(),
        isOptimistic: true,
      };

      // 2. Add to local store immediately — triggers UI re-render
      addMessage(conversationId, optimisticMessage);

      // 3. Send via WebSocket (or queue if disconnected)
      chatSocket.send({
        type: 'message.send',
        payload: { conversationId, content, clientId, attachments },
      });
    },
    [conversationId, addMessage, chatSocket]
  );

  // 4. Listen for server acknowledgment
  useEffect(() => {
    return chatSocket.on('message.new', (serverMessage: Message) => {
      if (serverMessage.clientId) {
        // Server confirmed our optimistic message — replace temp ID with real ID
        updateMessageStatus(conversationId, serverMessage.clientId, {
          id: serverMessage.id,        // real server ID
          status: 'sent',              // ✓
          isOptimistic: false,
        });
      }
    });
  }, [chatSocket, conversationId, updateMessageStatus]);

  return sendMessage;
}
```

##### Handling Send Failures

If the server doesn't acknowledge within a timeout (e.g., 10 seconds), or the WebSocket is disconnected:

```tsx
// In the message store
retryMessage: (conversationId, clientId) => {
  set((state) => {
    const messages = state.messages[conversationId];
    const idx = messages.findIndex((m) => m.id === clientId);
    if (idx !== -1) {
      messages[idx] = { ...messages[idx], status: 'sending' };
    }
    return { messages: { ...state.messages } };
  });
  // Re-send via WebSocket
  chatSocket.send({
    type: 'message.send',
    payload: { conversationId, content, clientId },
  });
};

failMessage: (conversationId, clientId) => {
  set((state) => {
    const messages = state.messages[conversationId];
    const idx = messages.findIndex((m) => m.id === clientId);
    if (idx !== -1) {
      messages[idx] = { ...messages[idx], status: 'failed' };
    }
    return { messages: { ...state.messages } };
  });
};
```

**UI for failed messages:**

```
┌────────────────────────────────────┐
│  "Hey, are you free tonight?"  ⚠️   │
│                         Failed to  │
│                         send. Tap  │
│                         to retry.  │
└────────────────────────────────────┘
```

The user taps the message → `retryMessage()` fires → message re-sends.

| Status | Icon | Color | User Action |
|--------|------|-------|-------------|
| `sending` | ⏳ (clock) | Grey | Wait |
| `sent` | ✓ | Grey | None |
| `delivered` | ✓✓ | Grey | None |
| `read` | ✓✓ | Blue | None |
| `failed` | ⚠️ | Red | Tap to retry |

---

#### 5.2.3 Virtualized Message List for Large Conversations

##### The Problem — Chat History Can Be Enormous

A conversation between two active users can accumulate **10,000+ messages** over months. A group chat in a workplace (like Slack) can have **100,000+ messages**. Rendering all of them is not feasible:

*   10,000 messages × ~5 DOM nodes per message = **50,000+ DOM nodes**.
*   Browser slows to a crawl, memory usage spikes, scrolling stutters.
*   Even loading the data (10K message objects in memory) is problematic on low-end devices.

##### The Approach — Only Render What's Visible

Virtualization renders only the messages visible in the viewport, plus a buffer (overscan) above and below:

```
┌─ Message List Container ──────────────────────────────┐
│                                                       │
│  [Messages 1-4980: NOT in DOM — above viewport]       │
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │ Message 4981  (overscan - above viewport)       │  │
│  │ Message 4982  (overscan - above viewport)       │  │
│  │ ──────────── viewport top ───────────────────── │  │
│  │ Message 4983  (visible)                         │  │
│  │ Message 4984  (visible)                         │  │
│  │ Message 4985  (visible)                         │  │
│  │ Message 4986  (visible)                         │  │
│  │ Message 4987  (visible)                         │  │
│  │ ──────────── viewport bottom ────────────────── │  │
│  │ Message 4988  (overscan - below viewport)       │  │
│  │ Message 4989  (overscan - below viewport)       │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  [Messages 4990-5000: NOT in DOM — below viewport]    │
│                                                       │
└───────────────────────────────────────────────────────┘
         Only ~9 DOM items rendered out of 5000
```

##### Why Chat Virtualization Is Harder Than Feed Virtualization

| Challenge | Feed (e.g., Facebook) | Chat |
|-----------|----------------------|------|
| **Scroll direction** | Top-to-bottom (new content at bottom) | **Bottom-anchored** — newest messages at bottom, older on scroll-up |
| **New items arrive** | Prepended at top (banner pattern) | **Appended at bottom** while user is reading |
| **Item heights** | Variable but measured once | Variable **and can change** (image loads, link preview expands) |
| **Initial scroll position** | Top (start of feed) | **Bottom** (most recent message) |
| **History loading** | Scroll down → load next page | **Scroll up** → load older messages (reverse pagination) |

##### Implementation with Bottom-Anchoring

```tsx
import { useRef, useEffect } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualizedMessageList({
  messages,
  onLoadMore
}: {
  messages: Message[];
  onLoadMore: () => void;
}) {
  const scrollRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: messages.length,
    getScrollElement: () => scrollRef.current,
    estimateSize: () => 60,   // average message height ~60px
    overscan: 10,             // extra buffer for smooth scrolling
    // Start at the bottom (most recent message)
    initialOffset: Infinity,
  });

  // Auto-scroll to bottom when new message arrives (if already at bottom)
  const isAtBottom = useRef(true);

  useEffect(() => {
    if (isAtBottom.current) {
      virtualizer.scrollToIndex(messages.length - 1, { align: 'end' });
    }
  }, [messages.length]);

  // Track whether user is at the bottom
  const handleScroll = () => {
    const el = scrollRef.current;
    if (!el) return;
    const threshold = 100; // within 100px of bottom = "at bottom"
    isAtBottom.current =
      el.scrollHeight - el.scrollTop - el.clientHeight < threshold;
  };

  return (
    <div
      ref={scrollRef}
      className="message-list"
      onScroll={handleScroll}
      style={{ height: '100%', overflow: 'auto' }}
      role="log"
      aria-label="Messages"
      aria-live="polite"
    >
      <div
        style={{
          height: virtualizer.getTotalSize(),
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualItem) => {
          const message = messages[virtualItem.index];
          return (
            <div
              key={message.id}
              ref={virtualizer.measureElement}
              data-index={virtualItem.index}
              style={{
                position: 'absolute',
                top: virtualItem.start,
                width: '100%',
              }}
            >
              <MessageBubble message={message} />
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

| Option | Value | Rationale |
|--------|-------|-----------|
| `estimateSize: 60` | Average message height | Text-only messages ~50px, images ~300px; 60px is a reasonable average for mostly-text chats |
| `overscan: 10` | 10 items above and below | Chat users scroll fast through history; 10 items buffer prevents blank flashes |
| `initialOffset: Infinity` | Start at bottom | Chat opens with the latest message visible, not the oldest |

---

#### 5.2.4 Reverse Infinite Scroll for Message History

##### The Pattern — Load Older Messages on Scroll Up

Unlike a feed (where you scroll down to load more), chat loads **older messages when the user scrolls up** toward the top of the message list:

```
User scrolls up (toward older messages)
       ↓
1. Sentinel element near the top of the list enters viewport
       ↓
2. IntersectionObserver fires → fetch older messages
       ↓
3. GET /api/conversations/:id/messages?before=<oldest_message_id>&limit=30
       ↓
4. Prepend older messages to the TOP of the message list
       ↓
5. Maintain scroll position (don't jump to the newly added messages!)
```

##### The Critical Problem — Scroll Position Jumping

When older messages are **prepended** at the top of the list, the scroll container's `scrollHeight` increases. Without correction, the viewport jumps:

```
BEFORE prepend:
  ┌──────────────────────┐
  │ Message 51           │ ← scrollTop = 0 (at top)
  │ Message 52           │
  │ Message 53           │
  └──────────────────────┘

AFTER prepend (without correction):
  ┌──────────────────────┐
  │ Message 21 (new!)    │ ← scrollTop = 0 → user sees THIS instead of 51
  │ Message 22 (new!)    │
  │ ...                  │
  │ Message 50 (new!)    │
  │ Message 51           │ ← user was looking at this — now pushed down!
  │ Message 52           │
  └──────────────────────┘

User's viewport jumped from Message 51 to Message 21 — jarring!
```

**The fix — adjust scrollTop by the height of prepended content:**

```tsx
function useReverseScroll(scrollRef: RefObject<HTMLDivElement>) {
  const prevScrollHeight = useRef<number>(0);

  // Call this BEFORE prepending messages
  const saveScrollPosition = () => {
    if (scrollRef.current) {
      prevScrollHeight.current = scrollRef.current.scrollHeight;
    }
  };

  // Call this AFTER prepending messages (in useLayoutEffect to run before paint)
  const restoreScrollPosition = () => {
    if (scrollRef.current) {
      const newScrollHeight = scrollRef.current.scrollHeight;
      const addedHeight = newScrollHeight - prevScrollHeight.current;
      scrollRef.current.scrollTop += addedHeight;
    }
  };

  return { saveScrollPosition, restoreScrollPosition };
}
```

##### Implementation

```tsx
function useLoadOlderMessages(conversationId: string, scrollRef: RefObject<HTMLDivElement>) {
  const sentinelRef = useRef<HTMLDivElement>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  const { saveScrollPosition, restoreScrollPosition } = useReverseScroll(scrollRef);
  const prependMessages = useMessageStore((s) => s.prependMessages);

  useEffect(() => {
    const sentinel = sentinelRef.current;
    if (!sentinel || !hasMore) return;

    const observer = new IntersectionObserver(
      async ([entry]) => {
        if (entry.isIntersecting && !isLoading) {
          setIsLoading(true);
          saveScrollPosition();

          const oldestMessage = getOldestMessage(conversationId);
          const response = await fetch(
            `/api/conversations/${conversationId}/messages?before=${oldestMessage.id}&limit=30`
          );
          const data = await response.json();

          prependMessages(conversationId, data.messages);
          setHasMore(data.hasMore);
          setIsLoading(false);

          // Restore scroll position in the next layout frame
          requestAnimationFrame(() => restoreScrollPosition());
        }
      },
      { root: scrollRef.current, rootMargin: '200px 0px 0px 0px', threshold: 0 }
    );

    observer.observe(sentinel);
    return () => observer.disconnect();
  }, [conversationId, isLoading, hasMore]);

  return { sentinelRef, isLoading, hasMore };
}
```

**Key differences from forward infinite scroll:**

| Aspect | Forward (Feed) | Reverse (Chat) |
|--------|----------------|-----------------|
| **Sentinel position** | Bottom of list | **Top** of list |
| **rootMargin for early trigger** | `0px 0px 500px 0px` (bottom) | `200px 0px 0px 0px` (**top**) |
| **New items added** | Appended at bottom | **Prepended at top** |
| **Scroll correction needed** | No | **Yes** — must offset scrollTop |
| **Pagination parameter** | `after=<cursor>` | `before=<cursor>` |

---

#### 5.2.5 Scroll Position Management and Anchoring

##### Three Scenarios That Require Scroll Management

**1. New message arrives while user is at the bottom → auto-scroll to show it**

```tsx
useEffect(() => {
  // Only auto-scroll if user is already at the bottom
  if (isAtBottom.current && messages.length > 0) {
    scrollToBottom('smooth');
  }
}, [messages.length]);

function scrollToBottom(behavior: ScrollBehavior = 'smooth') {
  scrollRef.current?.scrollTo({
    top: scrollRef.current.scrollHeight,
    behavior,
  });
}
```

**2. New message arrives while user is scrolled up reading history → show "New messages" pill**

```tsx
function NewMessagesPill({ count, onClick }: { count: number; onClick: () => void }) {
  if (count === 0) return null;

  return (
    <button
      className="new-messages-pill"
      onClick={onClick}
      aria-label={`${count} new messages. Click to scroll to bottom.`}
    >
      ↓ {count} new message{count > 1 ? 's' : ''}
    </button>
  );
}
```

```css
.new-messages-pill {
  position: sticky;
  bottom: 80px;          /* above the composer */
  left: 50%;
  transform: translateX(-50%);
  z-index: 10;
  padding: 6px 16px;
  border-radius: 20px;
  background: #3b82f6;
  color: #ffffff;
  font-size: 13px;
  font-weight: 600;
  box-shadow: 0 2px 8px rgba(0,0,0,0.15);
  cursor: pointer;
  border: none;
  animation: pillSlideUp 0.2s ease;
}
```

**3. Switching between conversations → restore previous scroll position**

When the user switches from Conversation A to Conversation B and back, they expect to be at the same scroll position they left Conversation A at.

```tsx
const scrollPositions = useRef(new Map<string, number>());

// Save position when leaving a conversation
function saveScrollPosition(conversationId: string) {
  if (scrollRef.current) {
    scrollPositions.current.set(conversationId, scrollRef.current.scrollTop);
  }
}

// Restore position when re-entering a conversation
function restoreScrollPosition(conversationId: string) {
  const savedTop = scrollPositions.current.get(conversationId);
  if (savedTop !== undefined && scrollRef.current) {
    scrollRef.current.scrollTop = savedTop;
  } else {
    scrollToBottom('instant'); // new conversation — start at bottom
  }
}
```

---

#### 5.2.6 Typing Indicators

##### How Typing Indicators Work

Typing indicators show "Alice is typing..." when another participant is actively composing a message. The flow:

```
Alice types a character
     ↓
1. Client sends: { type: 'typing.start', payload: { conversationId } }
     ↓
2. Server broadcasts to all other participants in the conversation
     ↓
3. Bob's client receives: { type: 'typing.update', payload: { conversationId, userId: 'alice', isTyping: true } }
     ↓
4. Bob sees: "Alice is typing..." (animated dots)
     ↓
5. Alice stops typing for 3 seconds → client sends typing.stop
     ↓
6. Bob's client hides the indicator
```

##### Throttling is Critical

Without throttling, every keystroke sends a WebSocket message. A fast typist at 80 WPM = ~6-7 keystrokes per second = 6-7 WebSocket messages per second, per user, per conversation. In a group chat with 10 active typers, that's 70 messages/second just for typing indicators.

**Solution: Throttle typing events to once every 2-3 seconds.**

```tsx
function useTypingIndicator(conversationId: string) {
  const chatSocket = useChatSocket();
  const typingTimer = useRef<number | null>(null);
  const isTyping = useRef(false);

  const sendTypingStart = useCallback(() => {
    if (!isTyping.current) {
      isTyping.current = true;
      chatSocket.send({
        type: 'typing.start',
        payload: { conversationId },
      });
    }

    // Reset the "stop" timer on each keystroke
    if (typingTimer.current) clearTimeout(typingTimer.current);
    typingTimer.current = window.setTimeout(() => {
      isTyping.current = false;
      chatSocket.send({
        type: 'typing.stop',
        payload: { conversationId },
      });
    }, 3000); // 3 seconds of inactivity → stop typing
  }, [conversationId, chatSocket]);

  return { onInputChange: sendTypingStart };
}
```

**Key decisions:**

| Decision | Rationale |
|----------|-----------|
| **Send `typing.start` only once** until stopped | Avoids flooding WebSocket with duplicate start events |
| **3-second inactivity timeout** | If user pauses typing for 3s, assume they stopped. Too short (1s) = indicator flickers. Too long (10s) = stale indicator. |
| **`typing.stop` explicitly sent** | Don't rely solely on timeout — explicit stop is faster feedback for recipients |
| **Throttle, not debounce** | Debounce would delay the *first* event; throttle sends the first immediately and drops subsequent until the interval passes |

##### Rendering the Indicator

```tsx
function TypingIndicator({ conversationId }: { conversationId: string }) {
  const typingUsers = usePresenceStore(
    (s) => s.typingUsers[conversationId] || []
  );

  if (typingUsers.length === 0) return null;

  const text =
    typingUsers.length === 1
      ? `${typingUsers[0].name} is typing`
      : typingUsers.length === 2
        ? `${typingUsers[0].name} and ${typingUsers[1].name} are typing`
        : `${typingUsers[0].name} and ${typingUsers.length - 1} others are typing`;

  return (
    <div className="typing-indicator" aria-live="polite" role="status">
      <span className="typing-dots">
        <span className="dot" />
        <span className="dot" />
        <span className="dot" />
      </span>
      <span className="typing-text">{text}</span>
    </div>
  );
}
```

```css
.typing-dots .dot {
  display: inline-block;
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #94a3b8;
  margin: 0 2px;
  animation: dotBounce 1.4s ease-in-out infinite;
}
.typing-dots .dot:nth-child(2) { animation-delay: 0.2s; }
.typing-dots .dot:nth-child(3) { animation-delay: 0.4s; }

@keyframes dotBounce {
  0%, 80%, 100% { transform: translateY(0); opacity: 0.4; }
  40% { transform: translateY(-6px); opacity: 1; }
}
```

---

#### 5.2.7 Message Status Indicators (Sent, Delivered, Read)

##### The Four States of a Message

```
┌──────────────────────────────────────────────────────────────┐
│  SENDING    → message left client, waiting for server ACK    │
│  (⏳ clock)   Shown immediately via optimistic update        │
│                                                              │
│  SENT       → server received and persisted the message      │
│  (✓ grey)    Server sends ACK with real message ID           │
│                                                              │
│  DELIVERED  → recipient's client received the message        │
│  (✓✓ grey)   Recipient's client sends delivery receipt      │
│                                                              │
│  READ       → recipient opened the conversation and saw it   │
│  (✓✓ blue)   Recipient's client sends read receipt          │
└──────────────────────────────────────────────────────────────┘
```

##### How Read Receipts Flow

```
Alice sends message to Bob
  ├─ Alice: status = "sending" → "sent" (server ACK)
  │
  ├─ Bob receives message via WebSocket
  │    └─ Bob's client sends: { type: 'message.delivered', messageId }
  │        └─ Server relays to Alice: { type: 'message.status', status: 'delivered' }
  │            └─ Alice: status = "delivered" ✓✓
  │
  └─ Bob opens the conversation (or already has it open)
       └─ Bob's client sends: { type: 'message.read', messageId }
           └─ Server relays to Alice: { type: 'message.status', status: 'read' }
               └─ Alice: status = "read" (blue ✓✓)
```

##### Implementation

```tsx
// Auto-send read receipts when messages become visible
function useReadReceipts(conversationId: string, messages: Message[]) {
  const chatSocket = useChatSocket();
  const lastReadId = useRef<string | null>(null);

  useEffect(() => {
    // Find the latest unread message from other users
    const unreadFromOthers = messages
      .filter((m) => m.senderId !== currentUserId && m.status !== 'read')
      .pop();

    if (unreadFromOthers && unreadFromOthers.id !== lastReadId.current) {
      lastReadId.current = unreadFromOthers.id;

      // Debounce to avoid sending receipt for every message during fast scroll
      const timer = setTimeout(() => {
        chatSocket.send({
          type: 'message.read',
          payload: { conversationId, messageId: unreadFromOthers.id },
        });
      }, 500);

      return () => clearTimeout(timer);
    }
  }, [messages, conversationId, chatSocket]);
}
```

##### Group Chat Read Receipts

In group chats, read receipts are more complex — multiple participants read at different times:

```tsx
type GroupReadReceipt = {
  messageId: string;
  readBy: Array<{
    userId: string;
    username: string;
    readAt: string;
  }>;
};

// Display: "Read by Alice, Bob, and 3 others"
function ReadReceiptSummary({ receipt }: { receipt: GroupReadReceipt }) {
  const readers = receipt.readBy;
  if (readers.length === 0) return null;

  const text =
    readers.length <= 2
      ? `Read by ${readers.map((r) => r.username).join(' and ')}`
      : `Read by ${readers[0].username} and ${readers.length - 1} others`;

  return <span className="read-receipt-summary">{text}</span>;
}
```

---

#### 5.2.8 Rich Media Messages (Images, Files, Links)

##### Image Messages — Upload with Preview

When a user attaches an image, the flow is:

```
User selects image from picker
  ↓
1. Generate local preview (URL.createObjectURL or FileReader)
  ↓
2. Show optimistic message with local preview + progress bar
  ↓
3. Upload to server: POST /api/upload (multipart form data)
  ↓
4. Server returns CDN URL for the uploaded image
  ↓
5. Send message via WebSocket with CDN URL
  ↓
6. Replace local preview with CDN URL
```

```tsx
async function sendImageMessage(
  conversationId: string,
  file: File
) {
  const clientId = crypto.randomUUID();

  // 1. Create local preview
  const localPreviewUrl = URL.createObjectURL(file);

  // 2. Optimistic message with local preview
  addMessage(conversationId, {
    id: clientId,
    type: 'image',
    content: '',
    imageUrl: localPreviewUrl,
    uploadProgress: 0,
    status: 'sending',
    isOptimistic: true,
  });

  // 3. Upload with progress tracking
  const cdnUrl = await uploadFile(file, (progress) => {
    updateMessage(conversationId, clientId, { uploadProgress: progress });
  });

  // 4. Send message with CDN URL via WebSocket
  chatSocket.send({
    type: 'message.send',
    payload: {
      conversationId,
      clientId,
      type: 'image',
      attachments: [{ url: cdnUrl, type: 'image', name: file.name }],
    },
  });

  // 5. Clean up local preview
  URL.revokeObjectURL(localPreviewUrl);
}

// Upload with progress using XMLHttpRequest (fetch doesn't support upload progress)
function uploadFile(
  file: File,
  onProgress: (percent: number) => void
): Promise<string> {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.open('POST', '/api/upload');

    xhr.upload.onprogress = (event) => {
      if (event.lengthComputable) {
        onProgress(Math.round((event.loaded / event.total) * 100));
      }
    };

    xhr.onload = () => {
      if (xhr.status === 200) {
        const { url } = JSON.parse(xhr.responseText);
        resolve(url);
      } else {
        reject(new Error('Upload failed'));
      }
    };

    xhr.onerror = () => reject(new Error('Network error'));

    const formData = new FormData();
    formData.append('file', file);
    xhr.send(formData);
  });
}
```

##### Link Preview Cards

When a message contains a URL, detect it and fetch OG metadata:

```tsx
function MessageContent({ message }: { message: Message }) {
  const urls = extractUrls(message.content);

  return (
    <div className="message-content">
      <p>{linkifyText(message.content)}</p>
      {urls.length > 0 && (
        <LinkPreviewCard url={urls[0]} />
      )}
    </div>
  );
}

function LinkPreviewCard({ url }: { url: string }) {
  const { data: preview, isLoading } = useQuery({
    queryKey: ['linkPreview', url],
    queryFn: () => fetch(`/api/link-preview?url=${encodeURIComponent(url)}`).then((r) => r.json()),
    staleTime: 24 * 60 * 60 * 1000, // OG metadata rarely changes — cache for 24h
  });

  if (isLoading) return <LinkPreviewSkeleton />;
  if (!preview) return null;

  return (
    <a href={url} target="_blank" rel="noopener noreferrer" className="link-preview">
      {preview.image && <img src={preview.image} alt="" className="link-preview__image" />}
      <div className="link-preview__meta">
        <span className="link-preview__title">{preview.title}</span>
        <span className="link-preview__description">{preview.description}</span>
        <span className="link-preview__domain">{new URL(url).hostname}</span>
      </div>
    </a>
  );
}
```

---

#### 5.2.9 Decision Matrix

| Scenario | Strategy | Library / Tool | Notes |
|----------|----------|---------------|-------|
| **Real-time transport** | WebSocket (primary) with offline message queue | Native `WebSocket` | Single multiplexed connection for messages, typing, presence |
| **Message sending** | Optimistic update with client-generated ID | Custom hook (`useSendMessage`) | Show instantly, confirm/fail asynchronously |
| **Large message lists** | Virtualization with dynamic row heights | `@tanstack/react-virtual` | Only 15-25 DOM nodes regardless of conversation size |
| **Loading older messages** | Reverse infinite scroll with scroll position correction | IntersectionObserver + `scrollTop` adjustment | Sentinel at top of list; preserve viewport after prepend |
| **Auto-scroll on new messages** | Conditional — only if user is at bottom | `isAtBottom` ref + `scrollTo` | Show "New messages" pill if scrolled up |
| **Typing indicators** | Throttled WebSocket events (1 per 3s) | Custom hook with `setTimeout` | Explicit start/stop; 3s inactivity timeout |
| **Message status** | Event-driven via WebSocket (sent → delivered → read) | Store updates on status events | Optimistic for sent; server for delivered/read |
| **Image messages** | Local preview + progress upload + CDN URL swap | `URL.createObjectURL` + `XMLHttpRequest` | XHR for upload progress (fetch lacks it) |
| **Link previews** | Server-side OG fetch + React Query cache | `useQuery` with 24h staleTime | Fetch OG metadata from your backend (avoid CORS) |
| **Scroll restoration** | Per-conversation scroll position map | `useRef(Map)` | Save/restore scrollTop on conversation switch |

---

### 5.3 Reusability Strategy

*   **`Avatar`**: Shared across conversation list, message view, member list. Props: `src`, `size`, `online` (green dot), `fallback` (initials).
*   **`Badge`**: Generic count badge reused for unread messages, notification count. Props: `count`, `max`, `color`.
*   **`RelativeTime`**: Shared utility ("2h ago", "Yesterday", "Mar 16") used in conversations, messages, and presence.
*   **`MessageBubble`**: Configurable for text, image, file, and link types via composition (slot pattern).
*   **`VirtualList`**: Wrapper around `@tanstack/react-virtual` used for both conversation list and message list with different configurations.
*   Design-system tokens for colors, spacing, typography.

---

### 5.4 Module Organization

```
src/
 ├── features/
 │    └── chat/
 │         ├── components/
 │         │    ├── ConversationList.tsx
 │         │    ├── ConversationItem.tsx
 │         │    ├── ChatView.tsx
 │         │    ├── MessageList.tsx
 │         │    ├── MessageBubble.tsx
 │         │    ├── MessageComposer.tsx
 │         │    ├── TypingIndicator.tsx
 │         │    ├── MessageStatus.tsx
 │         │    ├── LinkPreviewCard.tsx
 │         │    ├── EmojiPicker.tsx
 │         │    └── InfoPanel.tsx
 │         ├── transport/
 │         │    ├── ChatSocket.ts
 │         │    ├── ConnectionManager.ts
 │         │    └── MessageQueue.ts
 │         ├── hooks/
 │         │    ├── useSendMessage.ts
 │         │    ├── useTypingIndicator.ts
 │         │    ├── useReadReceipts.ts
 │         │    ├── useReverseScroll.ts
 │         │    ├── usePresence.ts
 │         │    └── useMessageSearch.ts
 │         ├── store/
 │         │    ├── conversationStore.ts
 │         │    ├── messageStore.ts
 │         │    └── presenceStore.ts
 │         ├── api/
 │         │    ├── chatApi.ts
 │         │    └── uploadApi.ts
 │         ├── utils/
 │         │    ├── linkify.ts
 │         │    ├── messageGrouping.ts
 │         │    ├── emojiParser.ts
 │         │    └── soundPlayer.ts
 │         └── types/
 │              └── chat.types.ts
 ├── shared/
 │    ├── components/ (Avatar, Badge, RelativeTime, VirtualList)
 │    ├── hooks/ (useIntersection, useVisibilityChange)
 │    └── utils/ (debounce, throttle, formatDate)
```

---

## 6. High Level Data Flow Explanation

### 6.1 Initial Load Flow

```
1. App loads → Shell renders (Sidebar + empty ChatView)
     ↓
2. Fetch conversation list: GET /api/conversations?limit=30
     ↓
3. Render conversation list with last message previews
     ↓
4. Establish WebSocket connection: wss://api.example.com/chat/ws
     ↓
5. User clicks a conversation
     ↓
6. Fetch recent messages: GET /api/conversations/:id/messages?limit=50
     ↓
7. Render messages, scroll to bottom
     ↓
8. Send read receipt for last message in the conversation
     ↓
9. WebSocket receives real-time messages from this point on
```

---

### 6.2 User Interaction Flow

```
User types in the composer
     ↓
1. Typing indicator sent (throttled, once per 3s)
     ↓
2. User presses Enter / taps Send
     ↓
3. Optimistic message added to local store (status: "sending")
     ↓
4. Message rendered instantly at bottom of list
     ↓
5. Auto-scroll to bottom (if user was already at bottom)
     ↓
6. Message sent via WebSocket
     ↓
7. Server ACK received → status updated to "sent" ✓
     ↓
8. Recipient receives message → server sends "delivered" status → ✓✓
     ↓
9. Recipient reads message → server sends "read" status → blue ✓✓
     ↓
10. Conversation list updated (last message preview, timestamp, sort order)
```

**Incoming message flow:**

```
Server pushes message via WebSocket
     ↓
1. Parse and deduplicate (by message ID)
     ↓
2. Add to message store for the correct conversation
     ↓
3. If conversation is currently active:
     ├─ User at bottom → auto-scroll and show new message
     └─ User scrolled up → show "New messages" pill
     ↓
4. Update conversation list (bump to top, update preview/timestamp)
     ↓
5. Increment unread count if conversation is not active
     ↓
6. Send delivery receipt: { type: 'message.delivered', messageId }
     ↓
7. Play notification sound (if preference allows)
```

---

### 6.3 Error and Retry Flow

*   **WebSocket disconnects**: Trigger exponential backoff reconnection (1s → 2s → 4s → 30s max). Queue outgoing messages in memory. On reconnect, flush queue and fetch missed messages via REST.
*   **Message send fails**: Mark message as `failed`; show retry icon. User taps to retry → re-send via WebSocket.
*   **Media upload fails**: Show progress bar in error state with retry button. On retry, re-upload the file.
*   **Conversation list fetch fails**: Show cached conversations from last successful fetch + error banner with retry.
*   **Message history fetch fails**: Show error state within the conversation view + retry button.
*   **Network loss detected (offline)**: Show "Connecting..." banner at the top of the chat. Queue all outgoing messages. On reconnection, flush queue and fetch missed messages via `GET /api/conversations/:id/messages?after=<lastMessageId>`.

---

## 7. Data Modelling (Frontend Perspective)

### 7.1 Core Data Entities

*   **Conversation** — a chat thread (1-on-1 or group)
*   **Message** — an individual message within a conversation
*   **Participant** — a user in a conversation with role and status
*   **Attachment** — media or file associated with a message
*   **Presence** — user's online/offline/typing state

---

### 7.2 Data Shape

```ts
type Conversation = {
  id: string;
  type: 'direct' | 'group';
  name?: string;                     // group name (null for direct chats)
  avatarUrl?: string;                // group avatar (null for direct chats)
  participants: Participant[];
  lastMessage: MessagePreview | null;
  unreadCount: number;
  updatedAt: string;                 // ISO 8601 — for sort order
  isPinned: boolean;
  isMuted: boolean;
};

type Participant = {
  userId: string;
  username: string;
  avatarUrl: string;
  role: 'admin' | 'member';
  joinedAt: string;
};

type Message = {
  id: string;
  conversationId: string;
  senderId: string;
  senderName: string;
  senderAvatar: string;
  content: string;
  type: 'text' | 'image' | 'file' | 'system';   // system = "Alice added Bob"
  attachments: Attachment[];
  status: 'sending' | 'sent' | 'delivered' | 'read' | 'failed';
  readBy?: ReadReceipt[];              // for group chats
  createdAt: string;
  editedAt?: string;
  replyTo?: MessagePreview;            // quoted reply
  reactions?: Reaction[];
  clientId?: string;                   // for optimistic correlation
  isOptimistic?: boolean;
};

type Attachment = {
  id: string;
  type: 'image' | 'file' | 'video' | 'audio';
  url: string;                          // CDN URL
  thumbnailUrl?: string;
  name: string;
  size: number;                          // bytes
  mimeType: string;
};

type MessagePreview = {
  id: string;
  senderId: string;
  senderName: string;
  content: string;
  type: string;
  createdAt: string;
};

type Reaction = {
  emoji: string;
  userId: string;
  username: string;
};

type ReadReceipt = {
  userId: string;
  username: string;
  readAt: string;
};

type PresenceState = {
  userId: string;
  status: 'online' | 'offline' | 'away';
  lastSeen?: string;
};
```

---

### 7.3 Entity Relationships

*   **One-to-Many**: One Conversation → many Messages.
*   **Many-to-Many**: Participants ↔ Conversations (a user can be in many conversations; a conversation has many participants).
*   **One-to-Many**: One Message → many Attachments.
*   **Normalized storage**: Messages stored by conversation ID in a map for O(1) lookup; conversations stored in an ordered array for the list.

```tsx
type MessageStore = {
  // Messages indexed by conversation ID → array of messages sorted by createdAt
  messages: Record<string, Message[]>;
  // Active conversation ID
  activeConversationId: string | null;
};

type ConversationStore = {
  // Ordered list of conversations (sorted by updatedAt descending)
  conversations: Conversation[];
  // Quick lookup by ID
  conversationMap: Map<string, Conversation>;
};
```

---

### 7.4 UI Specific Data Models

```ts
// Derived view model for the composition area
type ComposerState = {
  text: string;
  attachments: PendingAttachment[];    // files selected but not yet sent
  replyTo: MessagePreview | null;      // quoted reply
  isEmojiPickerOpen: boolean;
};

type PendingAttachment = {
  file: File;
  localPreviewUrl: string;
  uploadProgress: number;              // 0-100
  status: 'pending' | 'uploading' | 'uploaded' | 'failed';
};

// Derived view model for the message list
type MessageGroup = {
  senderId: string;
  messages: Message[];                 // consecutive messages from same sender
};

// Conversation list with computed fields
type ConversationListItem = Conversation & {
  displayName: string;                 // computed: other user's name for direct, group name for group
  displayAvatar: string;               // computed: other user's avatar for direct, group avatar for group
  formattedTime: string;               // "2h ago", "Yesterday"
  isOnline: boolean;                   // computed from presence store (direct chats only)
  isTyping: boolean;                   // computed from typing store
};
```

---

## 8. State Management Strategy

### 8.1 State Classification

| State Type | Examples | Storage |
|---|---|---|
| **Global App State** | Current user, auth token, app-level preferences | Zustand global store |
| **Feature State** | Conversation list, message history per conversation, draft messages | Feature-level stores (Zustand slices) |
| **Transport State** | WebSocket connection status, reconnect attempts, message queue | Transport manager (class instance) |
| **Presence State** | Online/offline per user, typing per conversation | Dedicated presence store |
| **Component Local State** | Emoji picker open, composer text, scroll position | useState / useRef |
| **Derived State** | Unread total, conversation display name, message groups | Computed selectors |

---

### 8.2 State Ownership

*   **ConversationStore** (Zustand) owns the conversation list, unread counts, and active conversation ID.
*   **MessageStore** (Zustand) owns messages per conversation, message status updates, and optimistic messages.
*   **PresenceStore** (Zustand) owns online/offline/typing state per user. Updated directly by WebSocket events.
*   **ChatSocket** (class instance) owns transport state (connection status, message queue, reconnect timer). Exposes state to stores via events.
*   **MessageComposer** owns draft text, pending attachments, and reply-to state locally.
*   **Prop drilling is minimal**: Components read from stores via selectors; WebSocket writes to stores via handlers.

---

### 8.3 Persistence Strategy

*   **In-memory**: Messages (ephemeral, loaded from server on conversation open), typing state, presence state.
*   **React Query / SWR cache**: Conversation list with `staleTime: 1min`; message pages for recently viewed conversations.
*   **localStorage**: Draft messages per conversation (saved on every keystroke, restored on app reload); mute/pin preferences; last active conversation ID.
*   **IndexedDB (for offline mode)**: Recent messages for top N conversations; conversation list snapshot — enables the app to boot and show content even without network.
*   **Service Worker cache**: Static assets, app shell for offline PWA mode.

---

## 9. High Level API Design (Frontend POV)

### 9.1 Required APIs

| API | Method | Description |
|-----|--------|-------------|
| `/api/conversations` | **GET** | Fetch paginated conversation list |
| `/api/conversations/:id/messages` | **GET** | Fetch paginated messages for a conversation |
| `/api/conversations` | **POST** | Create a new conversation (1-on-1 or group) |
| `/api/conversations/:id/members` | **POST/DELETE** | Add or remove members (group chat) |
| `/api/upload` | **POST** | Upload media/files, returns CDN URL |
| `/api/link-preview` | **GET** | Fetch OG metadata for a URL |
| `/api/contacts/search` | **GET** | Search users to start a new conversation |
| `/api/conversations/:id/read` | **POST** | Mark conversation as read up to a message ID |
| `wss://api.example.com/chat/ws` | **WebSocket** | Real-time message delivery, typing, presence |

---

### 9.2 Message Delivery Guarantees and Ordering (Deep Dive)

#### The Challenges of Real Time Message Delivery

Chat messages must arrive **in order** and **exactly once**. Sounds simple, but network conditions make it hard:

*   **Out-of-order delivery**: If two messages are sent 100ms apart, network jitter can cause the second to arrive before the first.
*   **Duplicate delivery**: On WebSocket reconnect, the server may replay events that were already received.
*   **Lost messages**: Messages sent during a brief disconnect may never be confirmed by the server.

#### Client-Side Ordering

The server assigns each message a **monotonically increasing sequence number** or a **server timestamp**. The frontend sorts by this value, not by local arrival time:

```tsx
function insertMessageInOrder(
  messages: Message[],
  newMessage: Message
): Message[] {
  // Binary search to find correct insertion point (sorted by createdAt)
  let low = 0;
  let high = messages.length;
  const newTime = new Date(newMessage.createdAt).getTime();

  while (low < high) {
    const mid = Math.floor((low + high) / 2);
    const midTime = new Date(messages[mid].createdAt).getTime();
    if (midTime < newTime) {
      low = mid + 1;
    } else {
      high = mid;
    }
  }

  const result = [...messages];
  result.splice(low, 0, newMessage);
  return result;
}
```

#### Deduplication

Prevent duplicate processing using a seen-message set:

```tsx
const processedMessageIds = new Set<string>();

function handleIncomingMessage(message: Message) {
  if (processedMessageIds.has(message.id)) return;
  processedMessageIds.add(message.id);

  // Prevent unbounded growth
  if (processedMessageIds.size > 5000) {
    const ids = Array.from(processedMessageIds);
    ids.slice(0, 2500).forEach((id) => processedMessageIds.delete(id));
  }

  messageStore.addMessage(message.conversationId, message);
}
```

#### Gap Detection and Backfill

When the client reconnects, there may be a gap between the last received message and new messages:

```tsx
// On WebSocket reconnect
async function backfillMissedMessages(conversationId: string) {
  const lastMessage = getLatestMessage(conversationId);
  if (!lastMessage) return;

  const response = await fetch(
    `/api/conversations/${conversationId}/messages?after=${lastMessage.id}&limit=100`
  );
  const data = await response.json();

  data.messages.forEach((msg: Message) => {
    handleIncomingMessage(msg); // deduplication built in
  });
}
```

---

### 9.3 Request and Response Structure

**GET /api/conversations?limit=30&cursor=abc123**

```json
{
  "conversations": [
    {
      "id": "conv_001",
      "type": "direct",
      "participants": [
        {
          "userId": "u_001",
          "username": "alice",
          "avatarUrl": "https://cdn.example.com/avatars/u_001.jpg",
          "role": "member"
        },
        {
          "userId": "u_002",
          "username": "bob",
          "avatarUrl": "https://cdn.example.com/avatars/u_002.jpg",
          "role": "member"
        }
      ],
      "lastMessage": {
        "id": "msg_500",
        "senderId": "u_001",
        "senderName": "alice",
        "content": "See you at 5!",
        "type": "text",
        "createdAt": "2026-03-17T10:30:00Z"
      },
      "unreadCount": 2,
      "updatedAt": "2026-03-17T10:30:00Z",
      "isPinned": false,
      "isMuted": false
    }
  ],
  "pagination": {
    "nextCursor": "c_def456",
    "hasMore": true
  }
}
```

**GET /api/conversations/:id/messages?limit=50&before=msg_500**

```json
{
  "messages": [
    {
      "id": "msg_451",
      "conversationId": "conv_001",
      "senderId": "u_002",
      "senderName": "bob",
      "senderAvatar": "https://cdn.example.com/avatars/u_002.jpg",
      "content": "Hey, want to grab dinner?",
      "type": "text",
      "attachments": [],
      "status": "read",
      "readBy": [{ "userId": "u_001", "username": "alice", "readAt": "2026-03-17T10:25:00Z" }],
      "createdAt": "2026-03-17T10:20:00Z"
    }
  ],
  "pagination": {
    "nextCursor": "msg_401",
    "hasMore": true
  }
}
```

**WebSocket Message (server → client): New message**

```json
{
  "type": "message.new",
  "payload": {
    "id": "msg_501",
    "conversationId": "conv_001",
    "senderId": "u_002",
    "senderName": "bob",
    "senderAvatar": "https://cdn.example.com/avatars/u_002.jpg",
    "content": "On my way!",
    "type": "text",
    "attachments": [],
    "status": "sent",
    "createdAt": "2026-03-17T10:35:00Z",
    "clientId": null
  }
}
```

---

### 9.4 Error Handling and Status Codes

| Status | Scenario | Frontend Handling |
|--------|----------|-------------------|
| `200` | Success | Render data |
| `304` | Not Modified | Use cached response |
| `400` | Bad request (invalid message content) | Show validation error in composer |
| `401` | Unauthorized | Redirect to login; close WebSocket |
| `403` | Not a member of conversation | Show "You can't send messages to this conversation" |
| `404` | Conversation deleted | Remove from conversation list; show "This conversation no longer exists" |
| `413` | File too large | Show "File exceeds maximum size (25MB)" |
| `429` | Rate limited (sending too fast) | Queue locally; retry after backoff |
| `500` | Server error | Show error toast; retry with exponential backoff |

---

## 10. Caching Strategy

### 10.1 What to Cache

*   **Conversation list**: Cached for instant render on app open; background refetch for freshness.
*   **Recent messages**: Last page of messages for each recently opened conversation.
*   **User profiles**: Participant info (name, avatar) cached to avoid redundant fetches.
*   **Link previews**: OG metadata cached for 24h — rarely changes.
*   **Draft messages**: Saved per conversation in localStorage.

### 10.2 Where to Cache

| Data | Cache Location |
|------|----------------|
| Conversation list | React Query in-memory + IndexedDB (offline) |
| Messages (recent per conversation) | React Query in-memory |
| Presence/typing state | Zustand store (ephemeral, no persistence) |
| Draft messages | localStorage (keyed by conversation ID) |
| User avatars | Browser HTTP cache + CDN |
| Link preview metadata | React Query cache (24h staleTime) |
| Media attachments | Browser HTTP cache + CDN |
| Notification sound | Browser HTTP cache (immutable) |

### 10.3 Cache Invalidation

*   **Conversation list**: Invalidated on every WebSocket event that changes a conversation (new message, member change). Background refetch on app focus.
*   **Messages**: Invalidated when new messages arrive via WebSocket (appended to cache, not refetched). Older pages evicted on memory pressure.
*   **Presence**: No caching — purely ephemeral state from WebSocket events.
*   **Draft messages**: Cleared when the message is successfully sent.

---

## 11. CDN and Asset Optimization

*   **Media delivery**: All shared images, videos, and files served via CDN with edge caching.
*   **Image optimization**:
    *   Uploaded images auto-generate thumbnails (200px width for chat bubble, full-res for lightbox).
    *   Serve WebP/AVIF with JPEG fallback via `<picture>` element or `srcset`.
    *   Avatar images served at 48x48px (conversation list) and 32x32px (message sender avatar in group chats).
*   **File downloads**: CDN-served with `Content-Disposition: attachment` header.
*   **Signed URLs**: Media URLs include short-lived tokens (1h expiry) to prevent unauthorized access.
*   **Cache headers**: `Cache-Control: public, max-age=31536000, immutable` for uploaded media (URL contains content hash). Short-lived for avatars (`max-age=3600`) since users change profile photos.

---

## 12. Rendering Strategy

*   **App Shell**: SSR for the layout skeleton (sidebar frame, header) → fast FCP. Hydrate on client.
*   **Conversation List**: **CSR** — real-time data from WebSocket; no SEO value.
*   **Chat View (Messages)**: Fully **CSR** — interactive, real-time, media-heavy. Not SSR-able.
*   **Emoji Picker**: **Lazy-loaded** chunk on first open (`React.lazy` + `Suspense`). ~50KB of emoji data.
*   **Info Panel**: **Lazy-loaded** on toggle.
*   **No SSR for message content**: Chat messages are private, real-time, interactive — SSR adds no value.
*   **Portal rendering**: Emoji picker, file preview lightbox, and context menus rendered via React Portal to escape parent overflow/z-index stacking.

---

## 13. Cross Cutting Non Functional Concerns

### 13.1 Security

*   **Authenticated WebSocket**: Short-lived JWT token passed via query parameter on connection. Server validates; token is rotated on each reconnect.
*   **No cross-user message leakage**: Server enforces that WebSocket only delivers messages for conversations the user is a member of.
*   **XSS prevention**: All message content rendered as text nodes (not `innerHTML`). If rich HTML is needed, use DOMPurify. Link URLs are validated against an allowlist of schemes (`http`, `https`).
*   **CSRF**: All REST POST requests include CSRF tokens.
*   **Media access control**: Uploaded files served via signed CDN URLs tied to the user's session. URLs expire after 1 hour.
*   **Input validation**: Message content length limited client-side (e.g., 4000 chars). File size limited (25MB). File type validated (no executable uploads).
*   **Rate limiting**: Client-side throttle on message sending (max 1 message per 100ms) to prevent accidental rapid-fire from held Enter key.

---

### 13.2 Accessibility

*   **Conversation List**:
    *   `role="listbox"` with `role="option"` for each conversation.
    *   `aria-selected` for the active conversation.
    *   Arrow key navigation between conversations.
*   **Message List**:
    *   `role="log"` container — screen readers announce new messages automatically.
    *   `aria-live="polite"` for incoming messages.
    *   Each message is a `<div>` with sender name and timestamp accessible to screen readers.
*   **Composer**:
    *   `aria-label="Type a message"` on the textarea.
    *   `aria-keyshortcuts="Enter"` to indicate Send shortcut.
    *   Announce "Message sent" via `aria-live` region on successful send.
*   **Typing Indicator**:
    *   `role="status"` with `aria-live="polite"` — screen reader announces "Alice is typing".
*   **Emoji Picker**:
    *   `role="dialog"` with keyboard grid navigation.
    *   Search input with live results announced.
*   **Focus Management**:
    *   On conversation switch, focus moves to the composer input.
    *   `Escape` closes emoji picker, info panel, and lightbox.
*   **Reduced Motion**: Disable typing indicator animation and message slide-in when `prefers-reduced-motion` is set.
*   **Color Contrast**: Message status icons, unread badges, and presence dots meet WCAG AA 4.5:1 ratio.

---

### 13.3 Performance Optimization

*   **Code splitting**: Chat view loaded as a primary chunk; emoji picker (~50KB), info panel, and file preview lightbox lazy-loaded.
*   **Virtualized lists**: Both conversation list and message list use virtualization — ~20 DOM nodes regardless of data size.
*   **Memoization**: `React.memo` on `MessageBubble` keyed by `message.id + message.status` — prevents re-renders when other messages in the list update.
*   **Efficient store selectors**: Zustand selectors return only the slice of state the component needs. Conversation list only subscribes to `conversations[]`, not individual messages.
*   **Batched WebSocket events**: Server can batch multiple events (e.g., 5 typing updates) into a single WebSocket frame. Client processes the batch in a single store update.
*   **Lazy avatar loading**: `loading="lazy"` for avatars in conversation list and message history (except visible viewport).
*   **RAF-based scroll**: Scroll position checks and auto-scroll use `requestAnimationFrame` for 60fps.
*   **Debounced draft save**: Save draft to localStorage at most once per second, not on every keystroke.

---

### 13.4 Observability and Reliability

*   **Error Boundaries**: Wrap `ChatView`, `ConversationList`, and `MessageComposer` in error boundaries. A crash in one doesn't break the others.
*   **Logging**: Track:
    *   Message sent, delivered, read (funnel analysis).
    *   WebSocket connect, disconnect, reconnect events (reliability).
    *   Message send failures and retry attempts.
    *   Media upload success/failure rates and upload duration.
    *   Time-to-first-message (open conversation → first message rendered).
*   **Performance monitoring**:
    *   Message round-trip time (send → server ACK).
    *   WebSocket reconnection frequency and duration.
    *   Conversation list load time, message history load time.
    *   Core Web Vitals for the chat page.
*   **Feature flags**: Gate new features (e.g., reactions, threads, link previews) behind flags for gradual rollout.
*   **Health check**: Client pings the WebSocket every 30s; if no pong within 5s, trigger reconnection.

---

## 14. Edge Cases and Tradeoffs

| Edge Case | Handling |
|-----------|----------|
| **User sends message while offline** | Queue in memory; show as "sending". On reconnect, flush queue via WebSocket. If flush fails, mark as "failed" with retry option. |
| **Rapid message sending (held Enter)** | Client-side throttle (max 1 per 100ms). Queue excess messages locally and send sequentially. |
| **Conversation with 50,000+ messages** | Virtualized list renders only visible. Older messages fetched on demand via reverse scroll. Messages evicted from memory on conversation switch. |
| **User switches conversations rapidly** | Cancel in-flight message history fetch (`AbortController`). Load new conversation immediately. Previous fetch result is discarded. |
| **Image upload fails mid-way** | Show progress bar in error state. Offer retry button. Don't send the message until upload succeeds (or user removes the attachment). |
| **Two tabs open same conversation** | Both receive WebSocket messages. Use `BroadcastChannel` to sync read receipts and typing state across tabs. |
| **Message received for hidden conversation** | Update conversation list (bump to top, increment unread). Don't load full message into message store until user opens the conversation. |
| **Network reconnect after long gap (hours)** | Backfill missed messages via REST (`GET /messages?after=lastId`). Update conversation list (some conversations may have been deleted or members changed). |
| **Group chat: 100+ participants typing** | Cap typing indicator to "Several people are typing..." beyond 5 users. Don't send individual typing notifications to avoid noise. |
| **Message contains malicious script** | Render as text node, never `innerHTML`. DOMPurify as defense-in-depth. CSP headers prevent inline script execution. |

### Key Tradeoffs

| Decision | Tradeoff |
|----------|----------|
| **WebSocket for all real-time** | Lowest latency and bidirectional, but requires persistent connection infrastructure. Mobile browsers may throttle background connections. |
| **Optimistic message sending** | Instant UI feedback, but requires rollback mechanism for failures. Adds complexity for status tracking. |
| **Virtualized message list** | Handles 50K+ messages with constant DOM node count, but adds complexity for scroll position management, dynamic heights, and bottom-anchoring. |
| **In-memory message storage** | Fast access, no persistence overhead, but messages are lost on tab close. Mitigated by IndexedDB for offline mode. |
| **Per-conversation message loading** | Only loads messages for active conversation, saving memory and bandwidth. But conversation switch has a loading delay (mitigated by caching recent pages). |
| **LocalStorage for drafts** | Survives page reload, but limited to 5-10MB. For large attachments, only store text draft + attachment metadata. |
| **Client-side link preview fetch** | Reduces server load, but blocked by CORS for most sites. Better to proxy through your backend. |
| **Single multiplexed WebSocket** | Simpler connection management, but all event types share bandwidth. A burst of typing indicators could theoretically delay message delivery (mitigated by server-side prioritization). |

---

## 15. Summary and Future Improvements

### Key Architectural Decisions

1.  **Single multiplexed WebSocket** — one connection carries messages, typing, presence, and status events. Simplifies lifecycle management and reduces server connections.
2.  **Optimistic message sending with client ID correlation** — messages appear instantly; server ACK maps `clientId` to real `messageId` for subsequent status tracking.
3.  **Virtualized message list with bottom-anchoring** — handles conversations of any size at constant DOM cost; auto-scrolls on new messages when user is at bottom.
4.  **Reverse infinite scroll with scroll position correction** — older messages load on scroll-up without viewport jumping; `scrollTop` adjusted by prepended content height.
5.  **Per-conversation message store with lazy loading** — only active conversation's messages are in memory; others loaded on demand. React Query caches recent pages.
6.  **Throttled typing indicators with explicit start/stop** — send once per 3s to avoid WebSocket flooding; 3s inactivity timeout auto-stops.

### Possible Future Enhancements

*   **Message threads and replies**: Threaded conversations within a message (like Slack threads) — requires a nested message list component.
*   **End-to-end encryption**: Implement Signal Protocol on the client for encrypted messaging. Key exchange and storage via the Web Crypto API.
*   **Voice messages**: Record audio via MediaRecorder API, upload, and play back inline with a waveform visualization.
*   **Message editing and deletion**: Edit within a time window (15 min); delete for everyone with "This message was deleted" placeholder.
*   **Offline-first with IndexedDB**: Store recent messages and conversation list in IndexedDB for instant app boot without network.
*   **Shared Worker for multi-tab**: Use a SharedWorker to maintain a single WebSocket connection shared across all tabs, reducing server connections.
*   **Web Workers for message processing**: Offload message parsing, grouping, link detection, and encryption to a Web Worker to keep the main thread free.
*   **View Transitions API**: Animate conversation switches (conversation list → chat view) using the View Transitions API for smoother navigation.

---

### Endpoint Summary

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/conversations` | GET | Fetch conversation list |
| `/api/conversations` | POST | Create new conversation |
| `/api/conversations/:id/messages` | GET | Fetch message history (paginated) |
| `/api/conversations/:id/members` | POST/DELETE | Add or remove members |
| `/api/conversations/:id/read` | POST | Mark conversation as read |
| `/api/upload` | POST | Upload media/file |
| `/api/link-preview` | GET | Fetch OG metadata for URL |
| `/api/contacts/search` | GET | Search users for new conversation |
| `wss://api.example.com/chat/ws` | WebSocket | Real-time messaging channel |

---

### Complete Chat Flow

| Direction | Mechanism | Trigger | Endpoint | Action |
|-----------|-----------|---------|----------|--------|
| Initial Load | REST | On mount | `/api/conversations` | Show conversation list |
| Open Chat | REST | Tap conversation | `/api/conversations/:id/messages` | Load message history |
| Send Message | WebSocket | User hits Send | `message.send` (WS) | Optimistic render + send |
| Receive Message | WebSocket | Server push | `message.new` (WS) | Append to message list |
| Typing Start | WebSocket | User types (throttled) | `typing.start` (WS) | Show "is typing..." |
| Typing Stop | WebSocket | 3s inactivity | `typing.stop` (WS) | Hide indicator |
| Read Receipt | WebSocket | Open conversation | `message.read` (WS) | Mark messages as read |
| Delivery Receipt | WebSocket | Client receives message | `message.delivered` (WS) | Update sender's status |
| Presence Update | WebSocket | User goes online/offline | `presence.update` (WS) | Green/grey dot |
| Load History | REST | Scroll up | `/api/conversations/:id/messages?before=` | Prepend older messages |
| Upload Media | REST | Attach file | `/api/upload` | Upload + get CDN URL |
| Tab Refocus | REST | `visibilitychange` | `/api/conversations` | Sync conversation list |
| Reconnect | WebSocket + REST | Connection restored | WS reconnect + `/messages?after=` | Flush queue + backfill |

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design
