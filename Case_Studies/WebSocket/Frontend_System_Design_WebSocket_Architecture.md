# Frontend System Design: WebSocket Architecture (Simple)

- [Frontend System Design: WebSocket Architecture (Simple)](#frontend-system-design-websocket-architecture-simple)
    - [1. What Is This](#1-what-is-this)
    - [2. Simple Slack Example](#2-simple-slack-example)
    - [3. Functional Requirements (Simple)](#3-functional-requirements-simple)
        - [3.1 Connect Once](#31-connect-once)
        - [3.2 Subscribe to Needed Channels Only](#32-subscribe-to-needed-channels-only)
        - [3.3 Receive and Process Events](#33-receive-and-process-events)
        - [3.4 Send User Actions](#34-send-user-actions)
        - [3.5 Recover Automatically](#35-recover-automatically)
        - [3.6 Multi Tab Behavior](#36-multi-tab-behavior)
    - [4. Non Functional Requirements (Simple)](#4-non-functional-requirements-simple)
    - [5. High Level Architecture](#5-high-level-architecture)
    - [6. Simple Data Flow](#6-simple-data-flow)
        - [6.1 Initial Load](#61-initial-load)
        - [6.2 Sending a Message](#62-sending-a-message)
        - [6.3 On Network Drop](#63-on-network-drop)
    - [7. Data Model (Simple)](#7-data-model-simple)
        - [7.1 Event Schema Versioning and Compatibility](#71-event-schema-versioning-and-compatibility)
    - [8. State Management (Simple)](#8-state-management-simple)
    - [9. API Design (Frontend POV)](#9-api-design-frontend-pov)
        - [9.1 Delivery Guarantees and Failure Semantics](#91-delivery-guarantees-and-failure-semantics)
    - [10. Caching, Rendering, and Performance](#10-caching-rendering-and-performance)
        - [10.1 Connection Management Defaults (Practical Numbers)](#101-connection-management-defaults-practical-numbers)
        - [10.2 Client Backpressure and Memory Policy](#102-client-backpressure-and-memory-policy)
    - [11. Security and Reliability](#11-security-and-reliability)
        - [11.1 Auth Lifecycle After Connect (Revocation and Role Changes)](#111-auth-lifecycle-after-connect-revocation-and-role-changes)
    - [12. Edge Cases](#12-edge-cases)
        - [12.1 Protocol Fallback Strategy (When WebSocket Is Blocked)](#121-protocol-fallback-strategy-when-websocket-is-blocked)
    - [13. WSS Token and Handshake (ASCII Diagrams)](#13-wss-token-and-handshake-ascii-diagrams)
        - [13.1 Proper End-to-End ASCII Sequence](#131-proper-end-to-end-ascii-sequence)
        - [13.2 API Count + Call Pattern (ASCII)](#132-api-count--call-pattern-ascii)
        - [13.3 Token Fetch + Connect Flow](#133-token-fetch--connect-flow)
        - [13.4 Handshake Headers (What Actually Happens)](#134-handshake-headers-what-actually-happens)
        - [13.5 First Messages After Handshake](#135-first-messages-after-handshake)
        - [13.6 Failure Cases](#136-failure-cases)
        - [13.7 Token Mechanism in Simple Terms](#137-token-mechanism-in-simple-terms)
    - [14. WebSocket Internals and Scaling](#14-websocket-internals-and-scaling)
        - [14.1 How WebSocket Works Internally (Simple)](#141-how-websocket-works-internally-simple)
        - [14.2 Internal Components (Backend Side)](#142-internal-components-backend-side)
        - [14.3 Scaling Stages (1 -> 100s -> Millions)](#143-scaling-stages-1---100s---millions)
            - [Stage A: 1 to 100 users (single node)](#stage-a-1-to-100-users-single-node)
            - [Stage B: 100s to 10k users (multi-gateway)](#stage-b-100s-to-10k-users-multi-gateway)
            - [Stage C: 10k to millions (distributed realtime fabric)](#stage-c-10k-to-millions-distributed-realtime-fabric)
            - [Hot connections and hot channels (important)](#hot-connections-and-hot-channels-important)
            - [Step-by-step migration checklist](#step-by-step-migration-checklist)
        - [14.4 How Traffic Is Handled Safely](#144-how-traffic-is-handled-safely)
        - [14.5 Practical Capacity Metrics to Track](#145-practical-capacity-metrics-to-track)
        - [14.6 Deep Theory: Kafka, Routing Table, and Consumers](#146-deep-theory-kafka-routing-table-and-consumers)
            - [A) End-to-end cross-node flow (theory)](#a-end-to-end-cross-node-flow-theory)
            - [B) Why Kafka is used](#b-why-kafka-is-used)
            - [C) Routing table: what it is, who updates it, who reads it](#c-routing-table-what-it-is-who-updates-it-who-reads-it)
            - [D) Consumer configuration patterns](#d-consumer-configuration-patterns)
            - [E) Can Node A also be subscribed/consumer?](#e-can-node-a-also-be-subscribedconsumer)
            - [F) Where load balancer fits](#f-where-load-balancer-fits)
            - [G) Failure and correctness notes](#g-failure-and-correctness-notes)
        - [14.7 Ordering Policy (What Is Guaranteed and Where)](#147-ordering-policy-what-is-guaranteed-and-where)
        - [14.8 Observability, SLOs, and Alerts](#148-observability-slos-and-alerts)
        - [14.9 Testing Strategy and Validation Gates](#149-testing-strategy-and-validation-gates)
    - [15. Final Summary](#15-final-summary)

---

## 1. What Is This

WebSocket in simple words:
- It is one live connection between browser and server.
- You keep it open and receive updates instantly.
- You do not need to refresh page again and again.

Reference system in this document:
- Slack-like chat app with channels, DMs, typing, and presence.

---

## 2. Simple Slack Example

Think like this:
1. You open Slack web.
2. App logs in and opens one live socket.
3. App subscribes to channels you care about.
4. New message appears instantly.
5. You send message, it shows `sending`, then `sent`.
6. If internet drops, app reconnects and catches up missed messages.

---

## 3. Functional Requirements (Simple)

### 3.1 Connect Once

- After login, open one WebSocket connection.
- Show clear status in UI: connecting, connected, reconnecting, offline.

### 3.2 Subscribe to Needed Channels Only

Client should subscribe only to relevant channels.

Where channel list comes from:
- Bootstrap API at startup.
- User membership data (joined channels, DMs).
- Current screen (open channel/thread).

Simple rule:
- Subscribed channels = allowed channels intersect currently needed channels.

### 3.3 Receive and Process Events

- `message.created` -> update thread and sidebar preview.
- `presence.updated` -> update online dot.
- Ignore duplicate events using `eventId`.
- Keep ordering using `sequence`.

### 3.4 Send User Actions

- User sends message.
- UI adds optimistic bubble with `sending`.
- Server ack confirms `sent`.
- On error, show retry.

### 3.5 Recover Automatically

- Detect disconnect.
- Reconnect with backoff.
- Resubscribe channels.
- Replay missed events from last sequence.

### 3.6 Multi Tab Behavior

- Prefer one leader tab to own socket.
- Other tabs receive same updates via BroadcastChannel/SharedWorker.
- If leader closes, another tab becomes leader.

```ascii
    Tab A (leader)            BroadcastChannel             Tab B (follower)
        |                           |                            |
        |==== owns WebSocket ======>|                            |
        |<=== receives live events =|                            |
        |--- publish updates ------>|--- deliver updates ------->|
        |                           |                            |
        |---- closes/crashes ----X  |                            |
        |                           |<-- election heartbeat -----|
        |                           |--- Tab B becomes leader -->|
        |                           |==== Tab B opens socket ===>|
```

---

## 4. Non Functional Requirements (Simple)

- Fast: chat feels instant.
- Reliable: reconnect and recover automatically.
- Scalable: subscribe only to needed data.
- Secure: authenticated socket and channel authorization.
- Usable: user always sees connection status.

---

## 5. High Level Architecture

Simple layers:
- UI layer (channel list, message thread, composer)
- State layer (messages, unread, presence, connection)
- Realtime layer (socket, subscribe, reconnect, replay)
- API layer (bootstrap, snapshot, replay endpoints)

```text
UI -> State -> Realtime -> API
```

---

## 6. Simple Data Flow

### 6.1 Initial Load

1. User logs in.
2. App calls bootstrap API.
3. App opens WebSocket and authenticates.
4. App subscribes to required channels.
5. App requests missed events from replay cursor.

```ascii
    Browser                 App API                  WSS Gateway            Replay API
        |                       |                           |                     |
        |--- login/session ---->|                           |                     |
        |--- GET /bootstrap --->|                           |                     |
        |<-- wsUrl/channels ----|                           |                     |
        |--- POST /token ------>|                           |                     |
        |<-- wsToken -----------|                           |                     |
        |==================== open wss://...token ==============================>|
        |-------------------- subscribe(channel, fromSeq) ----------------------->|
        |--- GET /replay?fromSeq=cursor ----------------------------------------->|
        |<-- missed events -------------------------------------------------------|
```

### 6.2 Sending a Message

1. User types and sends message.
2. Optimistic message appears immediately.
3. Command goes over socket.
4. Ack arrives.
5. UI becomes final state.

```ascii
    User/UI                   Browser Socket              WSS Gateway            Storage
        |                             |                         |                     |
        |-- click send -------------->|                         |                     |
        |  add optimistic bubble      |                         |                     |
        |                             |--- message.send ------->|                     |
        |                             |                         |--- persist --------->|
        |                             |                         |<-- saved id --------|
        |                             |<-- message.ack ---------|                     |
        |<-- optimistic -> sent ------|                         |                     |
```

### 6.3 On Network Drop

1. Socket disconnects.
2. Reconnect starts with backoff.
3. After reconnect, resubscribe.
4. Replay missed events.

```ascii
    Browser Client              Network              WSS Gateway             Replay API
        |                           |                      |                      |
        |<====== live stream ======>|                      |                      |
        |-------- X disconnect -----|                      |                      |
        | wait 1s, 2s, 4s           |                      |                      |
        |================ reconnect wss ===================>|                      |
        |---------------- resubscribe(fromSeq) ----------->|                      |
        |---------------- GET /replay?fromSeq=last ------->|--------------------->|
        |<--------------- missed events ------------------------------------------|
        | UI catches up and continues live                 |                      |
```

---

## 7. Data Model (Simple)

Core entities:
- ConnectionState
- ChannelSubscription
- EventEnvelope
- ReplayCursor

```ts
type EventEnvelope<T> = {
eventId: string;
sequence: number;
channel: string;
type: string;
payload: T;
};
```

### 7.1 Event Schema Versioning and Compatibility

Use schema versioning from day 1 so old clients do not break when payload evolves.

Simple rules:
- Add `schemaVersion` in every event envelope.
- Prefer additive changes (new optional fields) over breaking changes.
- Never remove/rename required fields without version bump.
- Support at least `N-1` client version in production rollout.

Example envelope:

```json
{
"eventId": "evt_123",
"schemaVersion": 2,
"channel": "channel:engineering",
"type": "message.created",
"payload": {
    "id": "m_1",
    "text": "hello",
    "mentions": []
}
}
```

Deprecation flow:
1. Introduce new optional fields.
2. Ship clients that can read both old and new shape.
3. Mark old fields deprecated in API contract.
4. Remove old fields only after upgrade window closes.

---

## 8. State Management (Simple)

Global state:
- socket status
- subscribed channels
- replay cursor

Feature state:
- messages
- unread counts
- presence

Local state:
- draft input
- panel open/close

---

## 9. API Design (Frontend POV)

Useful APIs:
- `GET /realtime/bootstrap`
- `GET /snapshot/{channel}`
- `GET /replay/{channel}?fromSeq=...`
- `POST /realtime/token`

How many APIs are used?
- Total core APIs: `4`
- Always called on startup: `2`
- `GET /realtime/bootstrap`
- `POST /realtime/token`
- Called only when needed: `2`
- `GET /snapshot/{channel}`
- `GET /replay/{channel}?fromSeq=...`

Which API is called when?
1. `GET /realtime/bootstrap`
- Called right after login/page load.
- Gives websocket URL, allowed channels, and cursor hints.
2. `POST /realtime/token`
- Called before opening `wss://`.
- Gives short-lived websocket token.
3. `GET /snapshot/{channel}`
- Called when channel data is missing in cache.
4. `GET /replay/{channel}?fromSeq=...`
- Called after reconnect or sequence gap.
- Gives only missed events.

Example subscribe message:

```json
{
"subscribe": {
    "channel": "channel:engineering",
    "fromSequence": 1234
}
}
```

### 9.1 Delivery Guarantees and Failure Semantics

For realtime chat, this practical model is recommended:
- Transport delivery: at-least-once (duplicates are possible).
- UI behavior: effectively-once using `eventId` dedupe.
- Ordering: guaranteed per channel partition key, not globally across all channels.

What each means:
1. At-most-once
- No duplicates, but messages can be lost on failure.

2. At-least-once
- Retries can duplicate events, but loss risk is lower.

3. Effectively-once at UI
- Client dedupes with `eventId` and sequence checks.
- User sees one final message even if network retries happened.

Failure semantics to document in API contract:
- Publisher timeout: client shows retry.
- Ack lost but write succeeded: retry may duplicate, dedupe must handle.
- Replay overlap: replay can include already seen events, client dedupe handles.

---

## 10. Caching, Rendering, and Performance

- Keep current chat state in memory.
- Store replay cursor in IndexedDB/local storage.
- Render only changed UI parts.
- Use virtualization for large message lists.
- Batch frequent events to avoid too many rerenders.

### 10.1 Connection Management Defaults (Practical Numbers)

Use concrete defaults so behavior is predictable:
- Heartbeat interval: 25s
- Heartbeat timeout: 10s
- Reconnect backoff: 1s, 2s, 4s, 8s, 16s (cap 30s)
- Reconnect jitter: plus/minus 20%
- Max reconnect attempts before offline banner: 8
- Max channels per socket (soft limit): 200
- Max inbound frame size: 64 KB
- Max outbound buffer per socket: 1 MB

These are starting points. Tune from production telemetry.

### 10.2 Client Backpressure and Memory Policy

Client must avoid unbounded queues.

Recommended policy:
1. Keep only bounded in-memory event queue (for example 5,000 events).
2. Prioritize visible channel updates over background channels.
3. Coalesce high-frequency event types (`presence.updated`, typing).
4. If queue is near full:
- Drop non-critical transient events.
- Keep critical durable events (`message.created`, moderation actions).
5. If queue overflows:
- Trigger replay resync from last stable sequence.
- Show subtle "syncing" UI state.

---

## 11. Security and Reliability

Security:
- Use `wss://` only.
- Validate auth token.
- Validate channel permission on server.

Reliability:
- Heartbeat ping/pong.
- Reconnect with backoff.
- Replay missed events.
- Dedupe by event id.

### 11.1 Auth Lifecycle After Connect (Revocation and Role Changes)

Authentication is not only a connect-time check.

Handle these runtime cases:
1. Session revoked (logout from another device)
- Server sends `auth.revoked` control event.
- Client clears local auth and redirects to login.

2. Permission changed while connected
- Example: user removed from channel.
- Next subscribe/publish must fail authorization.
- Server can also push `channel.access.revoked` event.

3. Token nearing expiry
- Client fetches fresh websocket token before reconnect.
- Avoid silent long-lived stale tokens.

```ascii
Auth Service           Gateway                Client
    |                     |                     |
    |-- revoke user ----->|                     |
    |                     |-- auth.revoked ---->|
    |                     |<-- close/ack -------|
    |                     |                     |
    |                     |  client clears state |
```

---

## 12. Edge Cases

- Tab sleeps: reconnect and replay on resume.
- Duplicate delivery: ignore by `eventId`.
- Out-of-order delivery: fix using `sequence`.
- Multiple tabs: leader tab pattern.

### 12.1 Protocol Fallback Strategy (When WebSocket Is Blocked)

Some corporate networks/proxies block WebSocket upgrade.

Fallback order:
1. Try WebSocket (`wss://`) first.
2. If upgrade repeatedly fails, switch to SSE.
3. If SSE also fails, switch to long-polling.

Feature degradation plan:
- WebSocket: full duplex (send/receive realtime).
- SSE: realtime receive is good; sends still go through HTTP.
- Long-poll: highest latency and cost; keep only essential updates.

Client rule:
- Persist current transport mode in memory only.
- Retry WebSocket periodically (for example every 5-10 minutes).

---

## 13. WSS Token and Handshake (ASCII Diagrams)

Use this simple, secure pattern:
1. Client gets short-lived websocket token using normal HTTPS API.
2. Client opens `wss://` connection with that token.
3. Server validates token and upgrades HTTP -> WebSocket.
4. After connect, client sends subscribe frames.

Important:
- Keep token short-lived (for example 1-5 minutes).
- Prefer ephemeral websocket token, not long-lived access token.
- Never hardcode real production token in frontend code.

### 13.1 Proper End-to-End ASCII Sequence

```ascii
+---------+        +-------------------+        +-------------------+        +----------------+
| Browser |        | App API           |        | Token API         |        | WSS Gateway    |
+---------+        +-------------------+        +-------------------+        +----------------+
|                        |                           |                            |
| (A) user already logged in                         |                            |
|----------------------->|                           |                            |
|                        |                           |                            |
| (1) GET /realtime/bootstrap                        |                            |
|----------------------->|                           |                            |
|<-----------------------| 200 { wsUrl, allowedChannels, cursorHints }            |
|                        |                           |                            |
| (2) POST /realtime/token                           |                            |
|----------------------------------->|               |                            |
|<-----------------------------------| 200 { wsToken, exp }                        |
|                        |                           |                            |
| (3) GET /snapshot/{channel} (optional, cache miss)                             |
|----------------------->|                           |                            |
|<-----------------------| 200 { initial channel state }                          |
|                        |                           |                            |
| (4) Open WSS: wss://rt.example.com/ws?token=...                                |
|--------------------------------------------------------------------------->     |
|<---------------------------------------------------------------------------     |
|                         101 Switching Protocols (Upgrade success)               |
|                                                                                    
| (5) WS frame: subscribe(channel:engineering, fromSeq:1234)                     |
|--------------------------------------------------------------------------->     |
| (6) WS frame: connection.ack(conn_7f2a)                                         |
|<---------------------------------------------------------------------------     |
|                                                                                    
| (7) On reconnect/gap -> GET /replay/{channel}?fromSeq=1234                      |
|----------------------->|                                                          |
|<-----------------------| 200 { missedEvents:[...] }                              |
```

### 13.2 API Count + Call Pattern (ASCII)

```ascii
Core APIs = 4

[Always]
1) GET  /realtime/bootstrap
2) POST /realtime/token

[Conditional]
3) GET  /snapshot/{channel}
4) GET  /replay/{channel}?fromSeq=...
```

### 13.3 Token Fetch + Connect Flow

```ascii
Browser Client                               API/Gateway
--------------                               -----------
1) HTTPS login already done

2) GET /realtime/token  ------------------->  validate session
(cookie/JWT)

3) <-------------------------- 200 { wsToken: "eyJ...abc", exp: 60s }

4) new WebSocket(
    "wss://rt.example.com/ws?token=eyJ...abc"
)

5) HTTP Upgrade request ------------------->  verify token + claims
                                            check exp, userId, tenantId

6) <-------------------------- 101 Switching Protocols (Upgrade success)

7) send subscribe frame ------------------->  start streaming events
```

### 13.4 Handshake Headers (What Actually Happens)

```ascii
GET /ws?token=eyJ...abc HTTP/1.1
Host: rt.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Version: 13
Origin: https://app.example.com
```

```ascii
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
```

### 13.5 First Messages After Handshake

```json
{
"type": "subscribe",
"channel": "channel:engineering",
"fromSequence": 1234
}
```

```json
{
"type": "connection.ack",
"connectionId": "conn_7f2a",
"userId": "u_42"
}
```

### 13.6 Failure Cases

```ascii
token expired -> server rejects upgrade or closes with auth code -> client fetches new wsToken -> reconnect
invalid channel subscribe -> server sends error frame -> client does not retry that channel
network break -> reconnect with backoff -> resubscribe -> replay missed events
```

```ascii
Case 1: token expired
Client ---- open wss(token_old) ----> Gateway
Client <--- auth close / reject ----- Gateway
Client ---- POST /realtime/token ---> API
Client <--- new wsToken ------------- API
Client ---- open wss(token_new) ----> Gateway

Case 2: invalid subscribe
Client ---- subscribe(private:admin) -> Gateway
Client <--- error.not_authorized ----- Gateway
Client ---- stop retry for this chan -> (client rule)

Case 3: network break
Client ----X socket lost X----------- Gateway
Client ---- reconnect + subscribe ---> Gateway
Client ---- GET /replay?fromSeq ----> API
Client <--- missed events ----------- API
```

### 13.7 Token Mechanism in Simple Terms

Use this easy model:

1. User authentication (normal login)
- User logs in via password/SSO.
- Server creates authenticated session (cookie or access token).

2. WebSocket token exchange
- Client calls `POST /realtime/token` using logged-in session.
- Server returns short-lived `wsToken`.

3. WSS handshake auth
- Client opens `wss://.../ws?token=wsToken`.
- Gateway validates token and returns `101 Switching Protocols`.
- Connection is now bound to that user identity.

4. Authorization after connect
- Auth says who the user is.
- Authorization says which channels the user can subscribe to.
- Server checks permissions on every subscribe request.

5. Reconnect + token refresh
- If socket drops and token expired, fetch new `wsToken`.
- Reconnect, resubscribe, replay missed events.

Simple security checklist:
- Use only `wss://`.
- Keep websocket token very short-lived (for example 1-5 min).
- Do not expose long-lived auth token in socket URL.
- Do not log full tokens in frontend/backend logs.

---

## 14. WebSocket Internals and Scaling

This section explains how WebSocket works internally and how systems scale from a few users to millions.

### 14.1 How WebSocket Works Internally (Simple)

No contradiction note:
- In code, client starts with `wss://...` URL.
- Under the hood, browser sends an HTTPS HTTP-Upgrade request (`Upgrade: websocket`) and gets `101 Switching Protocols`.
- After `101`, connection switches from HTTP semantics to WebSocket frames.

1. HTTP starts the connection
- Client sends an HTTP request with `Upgrade: websocket`.

2. Server upgrades protocol
- Server returns `101 Switching Protocols`.
- Now it is a persistent full-duplex connection.

3. Frames are exchanged
- Data is sent as WebSocket frames (small packets), not full HTTP responses.
- Client and server can both send anytime.

4. Keep-alive and health
- Ping/Pong frames check if connection is alive.
- If heartbeat fails, client reconnects.

5. Connection lifecycle
- Connect -> authenticate -> subscribe -> stream events -> reconnect if broken.

### 14.2 Internal Components (Backend Side)

```ascii
+----------------+      +----------------+      +-------------------+
|  Clients (Web) | ---> |  WSS Gateway   | ---> |  Event Router      |
+----------------+      +----------------+      +-------------------+
        |                         |
        v                         v
        +----------------+       +-------------------+
        | Presence Store |       | Message/Event Bus |
        +----------------+       +-------------------+
        |                         |
        +-----------+-------------+
            |
            v
            +---------------------+
            | Channel Subscribers |
            +---------------------+
```

What each does:
- WSS Gateway: accepts socket connections and does auth.
- Event Router: routes each event to correct channel subscribers.
- Presence Store: tracks online users.
- Event Bus: distributes events across multiple backend nodes.

### 14.3 Scaling Stages (1 -> 100s -> Millions)

Below is the practical journey most realtime systems follow.

Step 0: understand the core scaling problem
- HTTP requests are short-lived; WebSocket connections are long-lived.
- Each connected user consumes memory/file descriptors/CPU on one gateway.
- At high scale, fan-out (one message to many subscribers) becomes the biggest cost.

#### Stage A: 1 to 100 users (single node)

What architecture looks like:
- One gateway instance handles all sockets.
- Subscriptions are kept in memory map:
- `channelId -> [connectionIds]`

Why this works:
- Small connection count.
- Low fan-out pressure.
- Simple debugging and fast iteration.

Limits you will hit:
- Single point of failure.
- Restart drops all connections.
- No horizontal scaling.

#### Stage B: 100s to 10k users (multi-gateway)

What changes first:
- Run multiple gateway instances behind a load balancer.

Why multiple gateways are needed:
- One server cannot hold all sockets safely forever.
- You need rolling deploys without dropping everyone.
- You need failover if one node dies.

Why sticky sessions are used:
- WebSocket is a persistent TCP connection.
- After upgrade, connection stays bound to one gateway.
- Sticky routing keeps reconnects from bouncing unpredictably between nodes.
- This reduces repeated rehydration work (presence, channel maps, cache warmup).

Sticky session options:
- LB cookie affinity.
- Source-IP hash (less accurate behind NAT/mobile).
- Consistent hash on `userId`/`tenantId` when possible.

What must move out of memory now:
- Presence state (online/offline, last seen).
- Subscription metadata needed across nodes.
- Use shared store like Redis for cross-node coordination.

#### Stage C: 10k to millions (distributed realtime fabric)

What architecture becomes:
- Many gateway pools (often per region).
- Event bus/pub-sub backbone for cross-node fan-out.
- Channel partitioning so work is spread evenly.

Core patterns:
1. Partitioning
- Route channel events by partition key (workspaceId/channelId).
- Keeps ordering per channel while distributing load.

2. Fan-out pipeline
- Producer publishes event once.
- Bus/stream distributes to partitions.
- Gateway workers push only to local subscribed sockets.

3. Regionalization
- Users connect to nearest region for lower latency.
- Cross-region replication handles global rooms if needed.

4. Autoscaling
- Scale by connection count and outbound events/sec.
- Also watch CPU, memory, and queue lag.

#### Hot connections and hot channels (important)

Hot connection means:
- A single client sends/receives too much traffic (bot, abuse, power stream).

Hot channel means:
- One channel/room has massive fan-out (for example all-hands channel).

How to handle hot traffic:
1. Per-connection limits
- Rate-limit publish and subscribe actions per user.
- Apply token bucket/leaky bucket policies.

2. Per-channel protection
- Cap burst fan-out per time window.
- Batch/coalesce updates for high-frequency channels.

3. Shard heavy channels
- Move very large channels to dedicated partition/gateway pool.
- Isolate them from normal traffic.

4. Priority lanes
- Prioritize user-visible critical events over low-priority events.
- Defer/drop non-critical updates under pressure.

5. Slow consumer strategy
- Bound per-socket buffer size.
- If buffer exceeds threshold: drop low-priority events or force replay resync.

#### Step-by-step migration checklist

1. Start single gateway and measure baseline.
2. Add second gateway + load balancer.
3. Introduce shared presence/subscription store.
4. Add event bus for inter-node fan-out.
5. Partition channels by stable key.
6. Add autoscaling rules from real traffic metrics.
7. Add hot-channel isolation and rate-limits.
8. Run chaos/failover tests (node kill, region outage, reconnect storm).

Simple mental model:
- 1 node: easy and cheap.
- 10 nodes: shared state + sticky routing.
- 100+ nodes: partitioning + event bus + regional strategy.

### 14.4 How Traffic Is Handled Safely

1. Backpressure
- Do not render every event instantly on client.
- Batch/coalesce frequent updates.

2. Rate limiting
- Limit publish/subscribe rate per user and per tenant.

3. Message fan-out control
- Popular channels can create burst traffic.
- Use queue + worker fan-out and partitioning.

4. Slow consumer handling
- If a client cannot keep up, buffer with limits.
- Drop old non-critical events or force resync via replay.

5. Replay and recovery
- Keep sequence numbers.
- On reconnect, request only missed events (`fromSeq`).

### 14.5 Practical Capacity Metrics to Track

- Concurrent socket connections
- Messages in/out per second
- P95/P99 delivery latency
- Reconnect rate
- Replay request rate
- Dropped/failed message count

Simple rule:
- If reconnect rate and replay rate spike together, your system is under stress.

### 14.6 Deep Theory: Kafka, Routing Table, and Consumers

This subsection answers the exact question: when sender hits Node A, what happens if receiver is on Node B?

#### A) End-to-end cross-node flow (theory)

1. Sender is connected to Gateway Node A.
2. Receiver is connected to Gateway Node B.
3. Sender sends frame to Node A.
4. Node A validates auth + publish permission.
5. Node A persists message (DB/log) and publishes one event to Kafka (key usually = channelId).
6. A fanout/routing component reads that event and checks routing table.
7. Routing table returns target gateway nodes for this channel (Node B, maybe Node A too).
8. Delivery tasks are sent to those target nodes.
9. Node B pushes frame to receiver socket(s).
10. Receiver UI renders.

So yes: event bus (Kafka) is the bridge across nodes.

#### B) Why Kafka is used

Kafka is used for backend event distribution and durability, not for direct socket delivery.

Kafka gives:
- Durable event log (survives node restarts).
- Decoupling (Node A can publish quickly; delivery can happen asynchronously).
- Backpressure buffer (consumer lag can recover later).
- Partitioned scale (parallel processing).
- Replay by offsets.

Kafka does NOT know live sockets by itself.
- Socket-to-node mapping still comes from routing/subscription metadata.

#### C) Routing table: what it is, who updates it, who reads it

Typical routing table model:
- `channelId -> set(gatewayNodeIds)`
- `gatewayNodeId -> set(connectionIds)`
- `connectionId -> userId/session/subscriptions`

Who updates it:
1. Gateway node updates local memory on connect/subscribe/unsubscribe/disconnect.
2. Gateway node updates shared store (Redis/etc.) for cross-node visibility.

Who reads it:
- Fanout router/service reads it when a new message event is consumed.
- In smaller systems, Node A may read it directly.

#### D) Consumer configuration patterns

Pattern 1: all gateways consume same topic (simple, wasteful)
- Every node reads many events and drops most if no local subscribers.
- Easy start, poor efficiency at scale.

Pattern 2: targeted fanout (recommended)
1. Producer publishes once to channel-event topic.
2. Fanout service consumer group reads events.
3. Fanout resolves target nodes via routing table.
4. Fanout writes to node-targeted streams/topics/queues.
5. Each gateway consumes only its own delivery stream.

```ascii
Sender on Node A
    |
    | 1) ws frame: message.send
    v
+-----------+      2) publish(channelId key)      +----------------+
| Gateway A | -----------------------------------> | Kafka Topic(s) |
+-----------+                                      +----------------+
                                                      |
                                                      | 3) consume
                                                      v
                                               +---------------+
                                               | Fanout Router |
                                               +---------------+
                                                      |
                                   4) read routing table: channel -> nodes
                                                      |
                       +------------------------------+-----------------------------+
                       |                                                            |
                       v                                                            v
              +-------------------+                                        +-------------------+
              | Node B delivery Q |                                        | Node A delivery Q |
              +-------------------+                                        +-------------------+
                       |                                                            |
                       | 5) consume only own queue                                 | (optional)
                       v                                                            v
                  +-----------+                                                +-----------+
                  | Gateway B |                                                | Gateway A |
                  +-----------+                                                +-----------+
                       |                                                            |
                       | 6) push to local sockets                                  | local subscriber(s)
                       v                                                            v
                   Receiver(s)                                                  Local receiver(s)
```

This reduces unnecessary cross-node work significantly.

#### E) Can Node A also be subscribed/consumer?

Yes, possible and common.
- If Node A also has local subscribers for that channel, it is also a valid target.
- It should receive delivery work only when it has matching subscribers.

Important distinction:
- "Node A can be a target" is good.
- "Every node consumes every event" is expensive at high scale.

#### F) Where load balancer fits

LB handles:
- Connection admission (TLS + handshake routing to a gateway).
- Reconnect placement policy (sticky/cookie/hash).

LB does not perform cross-node fanout for messages.
- Cross-node fanout is handled by Kafka + fanout service + routing table.

#### G) Failure and correctness notes

- If Node B is down, routing table must evict stale node entries (TTL/heartbeat).
- If consumer lags, Kafka offsets allow catch-up.
- If receiver is offline, message remains durable and is delivered via replay later.
- Ordering is guaranteed per partition key (so keying strategy matters).

### 14.7 Ordering Policy (What Is Guaranteed and Where)

Ordering rules should be explicit:
1. Per-channel ordering: guaranteed when all events for a channel share one partition key.
2. Cross-channel ordering: not guaranteed.
3. Cross-region ordering: eventual, may be delayed/reordered.
4. Client rendering rule: sort by `sequence` inside a channel.
5. Gap rule: if `nextSequence != lastSequence + 1`, trigger replay.

Tie-break policy:
- Primary: `sequence`
- Secondary: server timestamp
- Final fallback: lexical `eventId`

### 14.8 Observability, SLOs, and Alerts

Track and alert with clear thresholds.

Suggested SLOs:
- Realtime delivery latency P99 < 2s
- Socket connect success rate > 99.9%
- Reconnect recovery under 30s for 99% sessions
- Replay success rate > 99.5%

Suggested alerts:
1. Reconnect spike
- Trigger: reconnect rate > 3x baseline for 5 min.

2. Replay spike
- Trigger: replay requests > 2x baseline for 10 min.

3. Consumer lag
- Trigger: lag exceeds threshold for 5 min.

4. Error burst
- Trigger: auth/subscribe errors exceed normal percentile.

Minimal runbook:
1. Check regional gateway health.
2. Check broker lag and partition hot spots.
3. Check auth/token service latency.
4. If needed, enable protective throttling and coalescing.

### 14.9 Testing Strategy and Validation Gates

Add repeatable tests before production rollout.

Core test suites:
1. Functional
- Connect, subscribe, publish, ack, replay, dedupe.

2. Resilience
- Node restart, broker restart, network partition, reconnect storm.

3. Scale
- Concurrent sockets, fan-out stress, hot-channel stress.

4. Security
- Expired token, revoked token, unauthorized subscribe/publish.

Pass criteria examples:
- No message loss in replay scenarios.
- Duplicate rate remains within accepted threshold and UI dedupe hides duplicates.
- P99 latency stays under SLO at target load.
- Recovery after single-node failure meets reconnect SLO.

---

## 15. Final Summary

If you explain WebSocket design in these 4 lines, it is enough for most interviews:
1. Open one live socket after login.
2. Subscribe only to relevant channels.
3. Update UI instantly with optimistic + ack flow.
4. Reconnect and replay missed events after disconnect.


More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design
