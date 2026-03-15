# Communication Protocols & Real-Time Data — Frontend Interview Guide

> Flow diagrams, working explanations, and trade-offs for HTTP, Polling, Long Polling, WebSockets, SSE, WebRTC, gRPC-Web, and GraphQL Subscriptions.

---

<a id="top"></a>

## Table of Contents

- [Overview and Mental Model](#overview-and-mental-model)
- [HTTP](#http)
- [Short Polling](#short-polling)
- [Long Polling](#long-polling)
- [Server Sent Events (SSE)](#server-sent-events-sse)
- [WebSockets](#websockets)
- [WebRTC](#webrtc)
- [gRPC Web](#grpc-web)
- [GraphQL Subscriptions](#graphql-subscriptions)
- [Microservice Communication Patterns](#microservice-communication-patterns)
- [Comparison Table](#comparison-table)
- [Decision Flowchart Which Protocol to Pick](#decision-flowchart-which-protocol-to-pick)
- [Real World Architecture Examples](#real-world-architecture-examples)
- [Interview Questions and Answers](#interview-questions-and-answers)

[⬆ Back to Top](#top)

---

## Overview and Mental Model

```
Client ◄──────────────────────────────────► Server

 HTTP          Request → Response (one-shot)
 Short Poll    Request → Response → wait → repeat
 Long Poll     Request → ...hangs... → Response → repeat
 SSE           Request → Stream ←←←←← (server pushes)
 WebSocket     Handshake → ◄══ Full-duplex ══►
 WebRTC        Signaling → ◄══ Peer-to-Peer ══►
```

### The Core Question

| Need | Best Fit |
|---|---|
| Fetch data once | HTTP REST / GraphQL |
| Near-real-time updates (seconds OK) | Short Polling |
| Real-time, server → client only | SSE |
| Real-time, bidirectional | WebSocket |
| Ultra-low latency media/P2P | WebRTC |
| Strongly-typed RPCs from browser | gRPC-Web |

[⬆ Back to Top](#top)

---

## HTTP

### 2.1 What Is It?

HTTP (HyperText Transfer Protocol) is a **request-response** protocol. The client sends a request; the server sends back exactly one response. The connection is stateless by default.

### 2.2 How It Works

1. **Client initiates** — The browser (or any HTTP client) opens a TCP connection to the server and sends an HTTP request containing a method (`GET`, `POST`, etc.), headers, and optionally a body.
2. **Server processes** — The server receives the request, performs the appropriate logic (read from DB, compute, etc.), and prepares a response.
3. **Server responds** — The server sends back a status code (e.g., `200 OK`, `404 Not Found`), response headers, and a body (JSON, HTML, etc.).
4. **Connection closes or reuses** — In HTTP/1.0 the connection closes immediately. In HTTP/1.1+ with `keep-alive`, the same TCP connection can be reused for subsequent requests, reducing handshake overhead.
5. **Stateless** — Each request is independent. The server does not remember previous requests unless state is stored externally (cookies, sessions, tokens).

### 2.3 Evolution

| Version | Year | Key Feature |
|---|---|---|
| HTTP/1.0 | 1996 | One request per TCP connection |
| HTTP/1.1 | 1997 | Keep-alive, pipelining, chunked transfer |
| HTTP/2 | 2015 | Multiplexing, header compression (HPACK), server push |
| HTTP/3 | 2022 | QUIC (UDP-based), zero-RTT handshake, no head-of-line blocking |

### 2.4 HTTP/1.1 vs HTTP/2 vs HTTP/3

```
HTTP/1.1          HTTP/2               HTTP/3
┌──────┐         ┌──────┐            ┌──────┐
│ Req1 │──┐      │Req1  │──┐        │Req1  │──┐
│ Res1 │◄─┘      │Req2  │  │ MUX    │Req2  │  │ MUX over
│ Req2 │──┐      │Req3  │  │ over   │Req3  │  │ QUIC (UDP)
│ Res2 │◄─┘      │Res2  │◄─┘ TCP    │Res1  │◄─┘
│ Req3 │──┐      │Res1  │           │Res3  │
│ Res3 │◄─┘      │Res3  │           │Res2  │
└──────┘         └──────┘            └──────┘
 Sequential       Multiplexed         No HOL blocking
```

- **HTTP/1.1** — Requests are sequential on a single connection (head-of-line blocking). Browsers open up to 6 parallel TCP connections per domain to work around this.
- **HTTP/2** — Multiplexes many requests/responses over a single TCP connection using binary frames and streams. Header compression (HPACK) reduces overhead. However, a single TCP packet loss stalls all streams (TCP-level HOL blocking).
- **HTTP/3** — Replaces TCP with QUIC (built on UDP). Each stream is independent, so a lost packet only blocks its own stream. Supports 0-RTT handshakes for faster connection setup.

### 2.5 Basic Syntax

```js
// GET request
const res = await fetch('/api/posts');
const data = await res.json();

// POST request
await fetch('/api/posts', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ title: 'Hello' })
});
```

### 2.6 When to Use

- Standard CRUD operations (REST APIs)
- Page loads, form submissions
- Static asset fetching
- Any request-response interaction that doesn't need real-time

### 2.7 Pros & Cons

| Pros | Cons |
|---|---|
| Simple, well-understood | No server push (HTTP/1.1) |
| Cacheable (ETags, Cache-Control) | New connection overhead (HTTP/1.1) |
| Stateless → easy to scale | Not suitable for real-time |
| Wide tooling support | Head-of-line blocking (HTTP/1.1, partially HTTP/2) |

[⬆ Back to Top](#top)

---

## Short Polling

### 3.1 What Is It?

The client sends **repeated HTTP requests** at a fixed interval to check for new data. The server responds immediately — whether or not there's new data.

### 3.2 How It Works

```
Client                          Server
  │── GET /updates ──────────►│
  │◄── 200 { data: [] } ──────│  (no new data)
  │                            │
  │  ... wait 5 seconds ...    │
  │                            │
  │── GET /updates ──────────►│
  │◄── 200 { data: [msg1] } ──│  (new data!)
  │                            │
  │  ... wait 5 seconds ...    │
  │                            │
  │── GET /updates ──────────►│
  │◄── 200 { data: [] } ──────│  (no new data)
```

1. **Client sets a timer** — A `setInterval` (or similar) triggers a `fetch` request every N seconds (e.g., every 5s).
2. **Server responds immediately** — The server checks for new data and returns it right away. If nothing is new, it returns an empty response (e.g., `{ data: [] }`).
3. **Client processes response** — If data is present, the UI updates. If empty, the client simply waits for the next interval.
4. **Repeat** — The cycle continues indefinitely until the client stops the timer.
5. **No persistent connection** — Each request is a completely independent HTTP round-trip. There's no state between requests on the server side.

**Trade-off:** The maximum delay before the client sees new data equals the polling interval. Shorter intervals = more responsive but more server load. Longer intervals = less load but stale data.

### 3.3 Basic Syntax

```js
function startPolling(url, interval = 5000) {
  const poll = async () => {
    const res = await fetch(url);
    const data = await res.json();
    if (data.length > 0) handleNewData(data);
  };

  poll();
  return setInterval(poll, interval);
}

// Start & stop
const timerId = startPolling('/api/notifications', 3000);
clearInterval(timerId); // stop
```

### 3.4 When to Use

- Dashboard refresh (analytics, stock tickers with 5-10s tolerance)
- Checking job/task status (file upload progress, CI build status)
- Simple notification badge counts
- When infrastructure doesn't support WebSocket/SSE

### 3.5 Pros & Cons

| Pros | Cons |
|---|---|
| Simplest to implement | Wastes bandwidth (empty responses) |
| Works everywhere (plain HTTP) | Latency = up to `interval` delay |
| Easy to debug | Hammers the server at scale |
| Stateless | Not truly real-time |

[⬆ Back to Top](#top)

---

## Long Polling

### 4.1 What Is It?

The client sends a request, and the server **holds the connection open** until new data is available (or a timeout occurs). Once the client gets a response, it immediately sends a new request.

### 4.2 How It Works

```
Client                              Server
  │── GET /updates ───────────────►│
  │                                │  (server holds connection)
  │          ... waiting ...       │
  │                                │  ← new data arrives!
  │◄── 200 { data: [msg1] } ──────│
  │                                │
  │── GET /updates ───────────────►│  (immediately reconnect)
  │                                │  (server holds again...)
  │          ... waiting ...       │
  │         (timeout 30s)          │
  │◄── 204 No Content ────────────│
  │                                │
  │── GET /updates ───────────────►│  (reconnect)
```

1. **Client sends request** — The client makes a standard HTTP `GET` request, often including a `Last-Event-ID` or timestamp so the server knows what the client has already seen.
2. **Server holds the connection** — Instead of responding immediately, the server keeps the request open. It parks the response object and waits for something to happen (new message, event from a queue, etc.).
3. **New data arrives → server responds** — As soon as relevant data is available, the server writes it to the held-open response and closes it (HTTP 200 with the payload).
4. **Timeout handling** — If no data arrives within a timeout window (e.g., 30s), the server responds with `204 No Content` (or an empty body) so the connection doesn't hang forever. Proxies and load balancers also have timeouts that must be respected.
5. **Client immediately reconnects** — The moment the client receives any response (data or timeout), it fires a new request to the server. This creates a continuous loop that feels near-real-time.
6. **Server resource cost** — Each waiting client occupies a held-open connection on the server. This can be expensive at scale since the server needs to manage many parked response objects.

**Key difference from Short Polling:** instead of the client repeatedly asking "anything new?", the server answers only when there IS something new — drastically reducing empty responses.

### 4.3 Basic Syntax

```js
async function longPoll(url) {
  while (true) {
    try {
      const res = await fetch(url);
      if (res.status === 200) {
        const data = await res.json();
        handleNewData(data);
      }
      // 204 = timeout, no data → just reconnect
    } catch (err) {
      await new Promise(r => setTimeout(r, 3000)); // backoff on error
    }
  }
}
```

### 4.4 When to Use

- Chat applications (before WebSocket adoption — Facebook used this)
- Real-time notifications where SSE/WS aren't available
- Environments behind strict firewalls/proxies that kill WS connections
- When you need near-instant updates but can't use WebSocket

### 4.5 Pros & Cons

| Pros | Cons |
|---|---|
| Near real-time delivery | Server holds open connections (resource cost) |
| Works through most proxies/firewalls | More complex than short polling |
| Fewer empty responses than short polling | Timeouts need careful handling |
| Universal browser support | Ordering/duplicate issues possible |

[⬆ Back to Top](#top)

---

## Server Sent Events (SSE)

### 5.1 What Is It?

SSE is a **unidirectional** protocol where the server pushes data to the client over a single, long-lived HTTP connection. Built on top of HTTP — uses `text/event-stream` content type. The browser provides a native `EventSource` API.

### 5.2 How It Works

```
Client                                  Server
  │── GET /events ────────────────────►│
  │◄── HTTP 200                        │
  │◄── Content-Type: text/event-stream │
  │                                    │
  │◄── data: {"msg": "hello"}         │  ← push
  │                                    │
  │◄── data: {"msg": "world"}         │  ← push
  │                                    │
  │◄── event: alert                    │  ← named event
  │◄── data: {"level": "critical"}    │
  │                                    │
  │    (connection stays open...)      │
```

1. **Client opens connection** — The client creates an `EventSource` object pointing to a URL. The browser sends a standard HTTP `GET` request to that endpoint.
2. **Server responds with event stream** — The server responds with `Content-Type: text/event-stream` and keeps the connection open. It does NOT close the response — instead it writes data incrementally.
3. **Server pushes events** — Whenever the server has new data, it writes a text block in SSE format (`data: ...\n\n`) to the open response. Each double newline (`\n\n`) marks the end of one event.
4. **Client receives events automatically** — The `EventSource` API fires `onmessage` for default events, or custom event listeners for named events (e.g., `event: notification`).
5. **Auto-reconnection** — If the connection drops (network issue, server restart), the browser automatically reconnects after a configurable delay (`retry:` field). On reconnection, the browser sends a `Last-Event-ID` header with the last received `id:` value so the server can resume from where the client left off.
6. **Connection stays open** — Unlike regular HTTP, the response never "finishes". The server keeps the TCP connection alive and writes new events as they occur. The client can close it explicitly with `source.close()`.

**Event Stream Format:**

```
data: Simple text message\n\n

event: notification
id: 42
data: {"type": "friend_request"}\n\n

retry: 5000
```

- `data:` — The payload (required)
- `event:` — Custom event name (default is `"message"`)
- `id:` — Event ID (used for auto-reconnection with `Last-Event-ID` header)
- `retry:` — Reconnection interval in ms

### 5.3 Basic Syntax

```js
const source = new EventSource('/api/events');

source.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('New message:', data);
};

source.addEventListener('notification', (event) => {
  showNotification(JSON.parse(event.data));
});

source.onerror = () => console.error('SSE error');
// EventSource auto-reconnects — no manual retry needed

source.close(); // close when done
```

### 5.4 When to Use

- Live feeds (news, social media timeline updates)
- Stock tickers, sports scores
- Real-time notifications / alerts
- Log streaming / build output
- AI/LLM streaming responses (ChatGPT-style token streaming)
- Any scenario where server → client is the primary data flow

### 5.5 Pros & Cons

| Pros | Cons |
|---|---|
| Native browser API (`EventSource`) | Unidirectional only (server → client) |
| Auto-reconnection built-in | Max 6 connections per domain on HTTP/1.1 (limit removed on HTTP/2 via multiplexing) |
| Works over standard HTTP | No binary data (text only) |
| Lightweight, simple protocol | Less browser support than WebSocket for older browsers |
| Event IDs for reliable delivery | |
| Works with HTTP/2 multiplexing | |

[⬆ Back to Top](#top)

---

## WebSockets

### 6.1 What Is It?

WebSocket is a **full-duplex, bidirectional** communication protocol. It starts as an HTTP request (upgrade handshake), then switches to a persistent TCP connection where both client and server can send messages at any time.

### 6.2 How It Works

```
Client                                    Server
  │── GET /chat HTTP/1.1 ──────────────►│
  │   Upgrade: websocket                │
  │   Connection: Upgrade               │
  │   Sec-WebSocket-Key: dGhlIHNh...    │
  │                                      │
  │◄── HTTP 101 Switching Protocols ────│
  │    Upgrade: websocket               │
  │    Sec-WebSocket-Accept: s3pPLM...  │
  │                                      │
  │◄════════ Full Duplex ═══════════════►│
  │                                      │
  │──► {"type":"msg","text":"Hi"}       │
  │◄── {"type":"msg","text":"Hello!"}   │
  │◄── {"type":"typing","user":"Bob"}   │
  │──► {"type":"msg","text":"How?"}     │
  │                                      │
  │──► Close Frame ─────────────────────│
  │◄── Close Frame ─────────────────────│
```

1. **HTTP Upgrade Handshake** — The client sends a standard HTTP `GET` request with special headers: `Upgrade: websocket`, `Connection: Upgrade`, and a random `Sec-WebSocket-Key`. This goes over the same port as HTTP (80/443).
2. **Server accepts the upgrade** — If the server supports WebSocket, it responds with `HTTP 101 Switching Protocols` and a `Sec-WebSocket-Accept` header (computed from the client's key + a magic GUID). This proves the server understands the WebSocket protocol.
3. **Protocol switches** — After the 101 response, the connection is no longer HTTP. It becomes a persistent, raw TCP connection using the WebSocket frame-based binary protocol. Both sides can now send data at any time.
4. **Full-duplex communication** — Either the client or the server can send messages independently. Messages are wrapped in small WebSocket frames (2-14 bytes overhead vs ~200-800 bytes for HTTP headers). Both text and binary data are supported.
5. **Heartbeat / keep-alive** — To detect stale connections, implementations use ping/pong frames. The server sends a ping; the client automatically responds with a pong. If no pong is received, the server terminates the connection.
6. **No auto-reconnect** — Unlike SSE, WebSocket has no built-in reconnection. If the connection drops, the `onclose` event fires and the client must manually reconnect (ideally with exponential backoff + jitter to avoid thundering herd).
7. **Closing** — Either side can initiate a close by sending a close frame. The other side responds with a close frame, and the TCP connection is torn down. Clean closes use code `1000`.

### 6.3 Basic Syntax

```js
const ws = new WebSocket('wss://api.example.com/ws');

ws.onopen = () => ws.send(JSON.stringify({ type: 'join', room: 'general' }));

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  // handle data.type: 'message', 'typing', 'presence', etc.
};

ws.onclose = (event) => {
  if (!event.wasClean) setTimeout(() => reconnect(), 3000);
};

ws.close(1000, 'Done'); // clean close
```

### 6.4 Reconnection Strategy

WebSocket does **not** auto-reconnect. Production apps must implement:

```
Connection drops → onclose fires
  │
  ▼
Start reconnection loop:
  Attempt 1 → wait 1s + random jitter
  Attempt 2 → wait 2s + random jitter
  Attempt 3 → wait 4s + random jitter   (exponential backoff)
  ...
  Attempt N → wait min(2^N seconds, 30s max cap)
  │
  ▼
On successful reconnect:
  • Reset retry counter
  • Flush any queued messages (buffered while offline)
  • Re-subscribe to rooms/channels
  │
  ▼
After max retries exceeded:
  • Show error UI to user
  • Optionally fall back to polling
```

### 6.5 Scaling WebSocket Connections

```
┌──────────┐                 ┌──────────────┐
│  Client   │◄═══ WS ═══════►│  WS Server 1 │──┐
└──────────┘                 └──────────────┘  │   ┌─────────┐
                                                ├──►│  Redis   │
┌──────────┐                 ┌──────────────┐  │   │  Pub/Sub │
│  Client   │◄═══ WS ═══════►│  WS Server 2 │──┘   └─────────┘
└──────────┘                 └──────────────┘
       ▲                           ▲
       └── Sticky Sessions ────────┘
           (IP hash / cookie)
```

- **Sticky sessions** ensure a client always reaches the same server (needed because WS is stateful).
- **Pub/Sub layer** (Redis, Kafka, NATS) enables cross-server broadcasting — a message sent to Server 1 reaches clients on Server 2.
- A single server can handle ~10K-100K concurrent WS connections (tune OS file descriptors and memory).

### 6.6 When to Use

- Chat applications (Slack, WhatsApp Web, Discord)
- Multiplayer games
- Collaborative editing (Google Docs, Figma)
- Live trading platforms
- Real-time dashboards that also accept user input
- Any scenario with **frequent bidirectional** messages

### 6.7 Pros & Cons

| Pros | Cons |
|---|---|
| True bidirectional, full-duplex | More complex server infrastructure |
| Very low latency | Stateful → harder to scale horizontally |
| Binary + text data support | No auto-reconnect (must implement) |
| Single TCP connection | Proxy/firewall issues (some block WS) |
| Efficient for high-frequency messages | No built-in request-response semantics |
| Wide browser support | Memory cost per connection on server |

[⬆ Back to Top](#top)

---

## WebRTC

### 7.1 What Is It?

WebRTC (Web Real-Time Communication) enables **peer-to-peer** audio, video, and arbitrary data transfer directly between browsers, with minimal server involvement (signaling only).

### 7.2 How It Works

```
Browser A                  Signaling Server              Browser B
    │                           │                            │
    │── Offer (SDP) ──────────►│                            │
    │                           │── Offer (SDP) ───────────►│
    │                           │                            │
    │                           │◄── Answer (SDP) ──────────│
    │◄── Answer (SDP) ─────────│                            │
    │                           │                            │
    │── ICE Candidates ───────►│── ICE Candidates ─────────►│
    │◄── ICE Candidates ───────│◄── ICE Candidates ─────────│
    │                           │                            │
    │◄═══════════ Direct P2P Connection ══════════════════►  │
    │     (audio / video / data)                             │
```

1. **Signaling (via a server)** — Before peers can talk directly, they need to exchange connection metadata. This is done through a **signaling server** (using WebSocket, HTTP, or any transport). The signaling server does NOT relay media — it only relays setup messages.

2. **SDP Offer/Answer** — Peer A creates an **SDP (Session Description Protocol) offer** describing its capabilities: supported codecs, media types, encryption parameters. This offer is sent to Peer B via the signaling server. Peer B responds with an **SDP answer** confirming which capabilities it supports.

3. **ICE Candidate Gathering** — Both peers simultaneously discover their own network addresses using **ICE (Interactive Connectivity Establishment)**:
   - **Host candidates** — local IP addresses
   - **Server-reflexive candidates** — public IP discovered via a **STUN server** (helps peers behind NAT)
   - **Relay candidates** — fallback through a **TURN server** when direct connection is impossible (strict firewalls)
   
   Each discovered candidate is sent to the other peer via the signaling server.

4. **Connectivity checks** — ICE performs connectivity checks on all candidate pairs (Peer A's candidates × Peer B's candidates) to find the best working path. It prioritizes direct connections over relayed ones.

5. **Direct P2P connection established** — Once a working candidate pair is found, a direct connection is established between the two browsers. All subsequent data flows peer-to-peer, **bypassing the server entirely**.

6. **Secure media/data transfer** — Audio and video are encrypted with **SRTP**, data channels use **DTLS**. Encryption is mandatory in WebRTC — there is no unencrypted mode.

7. **For large groups** — Pure P2P doesn't scale (N users = N×(N-1)/2 connections). For group calls, architectures use:
   - **SFU (Selective Forwarding Unit)** — a server that receives streams and selectively forwards them (used by Zoom, Meet)
   - **MCU (Multipoint Control Unit)** — a server that mixes all streams into one (higher server CPU cost)

### 7.3 Basic Syntax

```js
// Create peer connection
const pc = new RTCPeerConnection({
  iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
});

// Create data channel (or add media tracks)
const channel = pc.createDataChannel('chat');
channel.onmessage = (e) => console.log(e.data);

// Create and send offer
const offer = await pc.createOffer();
await pc.setLocalDescription(offer);
// → send offer to remote peer via signaling server

// Receive answer from remote peer
await pc.setRemoteDescription(remoteAnswer);

// Exchange ICE candidates
pc.onicecandidate = (e) => {
  if (e.candidate) sendToRemotePeer(e.candidate);
};
```

### 7.4 When to Use

- Video/audio calls (Zoom, Google Meet, Teams)
- Screen sharing
- Peer-to-peer file transfer
- Low-latency multiplayer gaming
- IoT device communication

### 7.5 Pros & Cons

| Pros | Cons |
|---|---|
| Peer-to-peer (low latency) | Complex setup (ICE, STUN, TURN) |
| Supports audio, video, data | Firewall/NAT traversal issues |
| Encrypted by default (DTLS/SRTP) | Higher battery usage on mobile |
| Reduces server bandwidth costs | Doesn't scale for large groups (need SFU/MCU) |

### 7.5 STUN, TURN & ICE — NAT Traversal Explained

Most devices sit behind NATs (Network Address Translators) or firewalls. WebRTC needs to discover a path between two peers — that's where **ICE** (Interactive Connectivity Establishment) comes in.

```
┌─────────────────────────────────────────────────────────────────┐
│                    ICE Candidate Gathering                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Host Candidates        — Local IP addresses (LAN)           │
│  2. Server Reflexive (srflx) — Public IP via STUN               │
│  3. Relay Candidates       — Relayed via TURN (fallback)        │
│                                                                 │
│  ICE tries candidates in order: Host → srflx → Relay            │
│  (Fastest to slowest, cheapest to most expensive)                │
└─────────────────────────────────────────────────────────────────┘
```

#### STUN (Session Traversal Utilities for NAT)

```
┌────────┐    "What's my public IP?"    ┌────────────┐
│ Peer A  │────────────────────────────►│ STUN Server │
│ (NAT)   │◄────────────────────────────│ (Public)    │
└────────┘    "Your IP is 203.0.113.5   └────────────┘
               port 54321"
```

- **Purpose:** Discover the peer's public IP and port.
- **Lightweight:** Only used during connection setup, not for media relay.
- **Free:** Google provides free STUN servers (`stun:stun.l.google.com:19302`).
- **Limitation:** Fails if both peers are behind **symmetric NATs** (port mapping changes per destination).

#### TURN (Traversal Using Relays around NAT)

```
┌────────┐                   ┌─────────────┐                   ┌────────┐
│ Peer A  │═══ Encrypted ═══►│ TURN Server  │═══ Encrypted ═══►│ Peer B  │
│ (NAT)   │◄═════════════════│  (Relay)     │◄═════════════════│ (NAT)  │
└────────┘                   └─────────────┘                   └────────┘
```

- **Purpose:** Relay media/data when direct P2P fails (symmetric NAT, strict firewalls).
- **Always works:** Acts as a middleman — both peers connect to TURN, which forwards data.
- **Expensive:** TURN consumes bandwidth on the relay server (you pay for it).
- **Last resort:** ICE tries STUN/direct first; TURN is the fallback.
- **Allocation:** Peer requests a TURN allocation (reserved port), which is time-limited and authenticated.

#### ICE (Interactive Connectivity Establishment)

ICE orchestrates the entire process:

```
1. Gather all candidates (host, srflx via STUN, relay via TURN)
2. Exchange candidates with remote peer via signaling server
3. Pair local candidates with remote candidates
4. Run connectivity checks on each pair (STUN Binding Requests)
5. Select the best working pair (lowest latency, direct > relay)
6. Begin media/data transfer on the selected path
```

**ICE States:**

| State | Meaning |
|---|---|
| `new` | ICE agent created, no candidates gathered yet |
| `gathering` | Collecting local candidates (host, srflx, relay) |
| `checking` | Running connectivity checks on candidate pairs |
| `connected` | At least one working pair found |
| `completed` | All checks done, best pair selected |
| `failed` | No working pair found (connection impossible) |
| `disconnected` | Connectivity lost (may recover) |
| `closed` | ICE agent shut down |

```js
peerConnection.oniceconnectionstatechange = () => {
  console.log('ICE state:', peerConnection.iceConnectionState);
  // 'checking' → 'connected' → 'completed' (happy path)
  // 'checking' → 'failed' (no path found)
  // 'connected' → 'disconnected' → 'connected' (network hiccup)
};
```

**Trickle ICE vs Full ICE:**

| Approach | Description | Speed |
|---|---|---|
| **Full ICE** | Gather ALL candidates first, then send offer/answer | Slower (waits for TURN) |
| **Trickle ICE** | Send candidates as they're discovered, incrementally | Faster (connection starts sooner) |

Trickle ICE is the modern default — candidates are sent via signaling as they appear.

### 7.6 SDP (Session Description Protocol)

SDP is the **metadata format** exchanged during WebRTC signaling. It describes the media capabilities of each peer.

```
v=0
o=- 4611731400430051location 2 IN IP4 127.0.0.1
s=-
t=0 0
m=audio 49170 RTP/SAVPF 111 103 104
c=IN IP4 203.0.113.5
a=rtpmap:111 opus/48000/2         ← Opus audio codec
a=fmtp:111 minptime=10;useinbandfec=1
a=candidate:0 1 UDP 2113667327 192.168.1.5 54321 typ host
a=candidate:1 1 UDP 1694498815 203.0.113.5 54321 typ srflx raddr 192.168.1.5 rport 54321
m=video 51372 RTP/SAVPF 96 97
a=rtpmap:96 VP8/90000               ← VP8 video codec
a=rtpmap:97 H264/90000             ← H264 video codec
```

**Key SDP Fields:**

| Field | Meaning |
|---|---|
| `m=` | Media line (audio, video, or application for data channels) |
| `a=rtpmap:` | Codec mapping (which codecs the peer supports) |
| `a=candidate:` | ICE candidate (IP, port, type) |
| `a=fingerprint:` | DTLS certificate fingerprint (for encryption verification) |
| `a=ice-ufrag` / `a=ice-pwd` | ICE credentials for connectivity checks |

**Offer/Answer Model:**

```
Peer A creates Offer SDP  →  "I can do Opus audio + VP8/H264 video"
                               Sends via signaling server
Peer B receives Offer     →  "I can do Opus audio + VP8 video (no H264)"
Peer B creates Answer SDP →  "Let's use Opus + VP8"
                               Sends via signaling server
Peer A receives Answer    →  Negotiation complete, start media
```

### 7.7 WebRTC Media — Tracks, Streams & Transceivers

```js
// ─── Getting User Media ───
const stream = await navigator.mediaDevices.getUserMedia({
  video: { width: 1280, height: 720, frameRate: 30 },
  audio: { echoCancellation: true, noiseSuppression: true }
});

// ─── Screen Sharing ───
const screenStream = await navigator.mediaDevices.getDisplayMedia({
  video: { cursor: 'always' },
  audio: true // system audio (if supported)
});

// ─── Adding Tracks to Peer Connection ───
stream.getTracks().forEach(track => {
  peerConnection.addTrack(track, stream);
});

// ─── Receiving Remote Tracks ───
peerConnection.ontrack = (event) => {
  const [remoteStream] = event.streams;
  remoteVideoElement.srcObject = remoteStream;
};

// ─── Transceivers (fine-grained control) ───
const transceiver = peerConnection.addTransceiver('video', {
  direction: 'sendrecv',      // 'sendonly', 'recvonly', 'inactive'
  sendEncodings: [
    { rid: 'high', maxBitrate: 2500000 },  // Simulcast layers
    { rid: 'mid', maxBitrate: 500000, scaleResolutionDownBy: 2 },
    { rid: 'low', maxBitrate: 150000, scaleResolutionDownBy: 4 }
  ]
});
```

### 7.8 SFU vs MCU vs Mesh — Scaling WebRTC

WebRTC is P2P by design, but group calls need different topologies:

```
┌─────────────────────────── MESH ──────────────────────────────┐
│                                                                │
│     A ◄══════► B          Each peer connects to EVERY other    │
│     ▲ ╲      ╱ ▲          peer. N peers = N×(N-1)/2 connections│
│     ║   ╲  ╱   ║          Good for: 2-4 participants           │
│     ║    ╲╱    ║          Bad for: 5+ (CPU/bandwidth explodes) │
│     ▼   ╱ ╲   ▼                                                │
│     C ◄══════► D                                               │
└────────────────────────────────────────────────────────────────┘

┌─────────────────────────── SFU ───────────────────────────────┐
│              (Selective Forwarding Unit)                        │
│                                                                │
│     A ──send──► ┌─────┐ ──forward──► B                        │
│     B ──send──► │ SFU │ ──forward──► A                        │
│     C ──send──► │     │ ──forward──► A, B, D                  │
│     D ──send──► └─────┘ ──forward──► A, B, C                  │
│                                                                │
│  Each peer sends ONE stream to SFU.                            │
│  SFU selectively forwards to each recipient.                   │
│  No transcoding — just routing. Low server CPU.                │
│  Used by: Zoom, Google Meet, Twilio, Jitsi                     │
└────────────────────────────────────────────────────────────────┘

┌─────────────────────────── MCU ───────────────────────────────┐
│            (Multipoint Conferencing Unit)                       │
│                                                                │
│     A ──send──► ┌─────┐                                       │
│     B ──send──► │ MCU │ ──mixed stream──► A, B, C, D          │
│     C ──send──► │     │                                       │
│     D ──send──► └─────┘                                       │
│                                                                │
│  MCU decodes ALL streams, mixes them into ONE composite        │
│  stream, re-encodes, and sends to each peer.                   │
│  Very CPU-intensive server. Low client bandwidth.              │
│  Used by: Legacy video conferencing systems                    │
└────────────────────────────────────────────────────────────────┘
```

| Topology | Upload | Download | Server CPU | Best For |
|---|---|---|---|---|
| **Mesh** | N-1 streams | N-1 streams | None | 2-4 people, simple |
| **SFU** | 1 stream | N-1 streams | Low (routing) | 5-50+ people |
| **MCU** | 1 stream | 1 stream | Very High (transcoding) | Low-bandwidth clients |

### 7.9 WebRTC Security Model

```
┌────────────────────────────────────────────────────────────────┐
│                    WebRTC Security Stack                        │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────────┐   ┌─────────────────────────────────────┐  │
│  │  Data Channels  │   │  Media (Audio/Video)                │  │
│  │  ───────────── │   │  ─────────────────────              │  │
│  │  SCTP over DTLS │   │  SRTP (Secure RTP)                  │  │
│  │                │   │  Keys exchanged via DTLS             │  │
│  └────────┬───────┘   └──────────────┬──────────────────────┘  │
│           │                          │                         │
│           ▼                          ▼                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  DTLS (Datagram Transport Layer Security)               │   │
│  │  • TLS-like encryption for UDP                          │   │
│  │  • Certificate fingerprints verified via SDP            │   │
│  │  • Mutual authentication (both peers verify)            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                     │
│                          ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  ICE + UDP/TCP Transport                                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
│  Key Points:                                                   │
│  • ALL WebRTC communication is encrypted by default            │
│  • Cannot be disabled — encryption is mandatory                │
│  • SRTP keys derived from DTLS handshake (no external KMS)    │
│  • getUserMedia requires HTTPS (secure context) or localhost   │
│  • User must grant explicit permission for camera/mic          │
└────────────────────────────────────────────────────────────────┘
```

### 7.10 WebRTC Complete Connection Flow (Putting It All Together)

```
┌──────────┐         ┌──────────────┐          ┌──────────┐
│  Peer A   │         │  Signaling    │          │  Peer B   │
│           │         │  Server       │          │           │
│  1. Create│         │  (WebSocket/  │          │           │
│     PeerConn        │   HTTP)       │          │           │
│           │         │               │          │           │
│  2. getUserMedia()  │               │          │           │
│     (camera+mic)    │               │          │           │
│           │         │               │          │           │
│  3. addTrack()      │               │          │           │
│           │         │               │          │           │
│  4. createOffer()   │               │          │           │
│  5. setLocalDesc()  │               │          │           │
│           │         │               │          │           │
│  6. ──Offer SDP────►│──Offer SDP───►│          │           │
│           │         │               │  7. setRemoteDesc()  │
│           │         │               │  8. createAnswer()   │
│           │         │               │  9. setLocalDesc()   │
│           │         │               │          │           │
│           │◄─Answer SDP─│◄─Answer SDP──│          │
│ 10. setRemoteDesc() │               │          │           │
│           │         │               │          │           │
│ 11. ICE candidates ◄═══════════════► ICE candidates       │
│     (trickled via signaling)                    │           │
│           │         │               │          │           │
│ 12. DTLS Handshake ◄═════ P2P ═════► DTLS Handshake       │
│           │         │               │          │           │
│ 13. ◄════ SRTP Media + SCTP Data ════►         │           │
│           │         │               │          │           │
│     🎉 Connected!   │               │  🎉 Connected!       │
└──────────┘         └──────────────┘          └──────────┘
```

[⬆ Back to Top](#top)

---

## gRPC Web

### 8.1 What Is It?

gRPC-Web allows browsers to call **gRPC services** using Protocol Buffers. It provides strongly-typed, efficient RPC communication but requires a proxy (like Envoy) since browsers can't do native HTTP/2 gRPC.

### 8.2 How It Works

```
Browser ──► gRPC-Web ──► Envoy Proxy ──► gRPC Server
           (HTTP/1.1      (translates     (HTTP/2
            or HTTP/2)     to gRPC)        native)
```

1. **Define service contracts** — Services are defined in `.proto` files using Protocol Buffers (protobuf). These define the RPC methods, request types, and response types in a language-neutral schema.

2. **Code generation** — The `.proto` file is compiled into client-side JavaScript/TypeScript stubs using `protoc` with the gRPC-Web plugin. This generates typed request/response classes and service client methods — no manual HTTP calls needed.

3. **Client makes RPC call** — The browser calls the generated client method (e.g., `client.getUser(request, ...)`). Under the hood, gRPC-Web serializes the request into a compact binary protobuf format and sends it as an HTTP/1.1 or HTTP/2 request with `Content-Type: application/grpc-web`.

4. **Proxy translates** — Browsers cannot speak native gRPC (which requires HTTP/2 trailers and full-duplex streaming). An **Envoy proxy** (or similar) sits between the browser and the gRPC server, translating the gRPC-Web request into a native gRPC request.

5. **Server processes** — The backend gRPC server processes the request and responds in native gRPC format. The proxy translates it back to gRPC-Web format for the browser.

6. **Streaming support** — gRPC-Web supports **server streaming** (server sends a stream of messages in response to a single request). However, **client streaming** and **bidirectional streaming** are NOT supported in the browser due to HTTP limitations.

### 8.3 Basic Syntax

```protobuf
// user.proto — Service definition
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);
}
```

```js
// Client call (using generated stubs)
const request = new GetUserRequest();
request.setId('user-123');

client.getUser(request, {}, (err, response) => {
  console.log(response.getName());
});
```

### 8.4 When to Use

- Microservices architecture where backend already uses gRPC
- Need strongly-typed contracts between frontend and backend
- High-performance, low-bandwidth requirements
- Internal tools / dashboards

### 8.5 Pros & Cons

| Pros | Cons |
|---|---|
| Strongly typed (protobuf) | Requires proxy (Envoy) |
| Compact binary format | Not human-readable (debugging harder) |
| Code generation | Limited browser support (no bidi streaming) |
| Server streaming support | Steeper learning curve |

### 8.5 Protocol Buffers (Protobuf) — Deep Dive

Protobuf is the **serialization format** used by gRPC. It's a binary format that is ~5-10x smaller and ~20-100x faster to parse than JSON.

```
┌──────────────────────── JSON vs Protobuf ────────────────────┐
│                                                               │
│  JSON (text, 82 bytes):                                       │
│  {"id":"user-123","name":"Alice","email":"a@b.com",│
│   "age":30,"active":true}                                   │
│                                                               │
│  Protobuf (binary, ~28 bytes):                                │
│  0a 08 75 73 65 72 2d 31 32 33 12 05 41 6c 69 63 65 ...     │
│                                                               │
│  Savings: ~66% smaller payload                                │
└───────────────────────────────────────────────────────────────┘
```

**How Protobuf Encoding Works:**

```protobuf
message User {
  string id = 1;      // Field number 1 → tag = (1 << 3) | 2 = 0x0a
  string name = 2;    // Field number 2 → tag = (2 << 3) | 2 = 0x12
  string email = 3;   // Field number 3
  int32 age = 4;      // Varint encoding (30 = 0x1e, just 1 byte!)
  bool active = 5;    // 1 byte (0 or 1)
}
```

**Key Protobuf Rules:**
- **Field numbers are forever** — Once assigned, never change them (backward compat).
- **Adding fields is safe** — Old clients ignore unknown fields.
- **Removing fields** — Mark as `reserved` to prevent reuse.
- **Default values** — 0 for numbers, `""` for strings, `false` for bools (not serialized → saves space).

### 8.6 gRPC Streaming Types

gRPC supports 4 communication patterns:

```
┌──────────────────────────────────────────────────────────────┐
│                    gRPC Communication Patterns                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. UNARY (Request-Response)                                 │
│     Client ──Request──► Server                               │
│     Client ◄──Response── Server                              │
│     Like a regular REST call.                                │
│                                                              │
│  2. SERVER STREAMING                                         │
│     Client ──Request──► Server                               │
│     Client ◄──Stream──── Server  (multiple responses)        │
│     Example: Stock price feed, log tailing.                  │
│                                                              │
│  3. CLIENT STREAMING                                         │
│     Client ──Stream──► Server  (multiple requests)           │
│     Client ◄──Response── Server                              │
│     Example: File upload, sensor data ingestion.             │
│                                                              │
│  4. BIDIRECTIONAL STREAMING   ⚠️ NOT supported in gRPC-Web  │
│     Client ◄══Stream══► Server  (both sides stream)          │
│     Example: Chat, real-time collaboration.                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

```protobuf
// All 4 patterns in proto definition
service ChatService {
  rpc GetMessage (GetMessageRequest) returns (Message);                    // Unary
  rpc ListMessages (ListRequest) returns (stream Message);                 // Server streaming
  rpc UploadMessages (stream Message) returns (UploadResponse);           // Client streaming
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);             // Bidirectional
}
```

```js
// ─── Server Streaming Example (gRPC-Web supported) ───
const stream = client.listMessages(new ListRequest());

stream.on('data', (message) => {
  console.log('Received:', message.getText());
  appendToUI(message);
});

stream.on('status', (status) => {
  console.log('Stream status:', status.code, status.details);
});

stream.on('end', () => {
  console.log('Stream completed');
});
```

### 8.7 gRPC-Web vs Native gRPC

| Feature | Native gRPC | gRPC-Web |
|---|---|---|
| Transport | HTTP/2 (native) | HTTP/1.1 or HTTP/2 |
| Requires proxy | No | Yes (Envoy, grpc-web-proxy) |
| Unary | ✅ | ✅ |
| Server streaming | ✅ | ✅ |
| Client streaming | ✅ | ❌ |
| Bidi streaming | ✅ | ❌ |
| Browser support | ❌ (cannot do raw HTTP/2) | ✅ |
| Binary format | Protobuf | Protobuf or base64 text |
| Used in | Backend-to-backend | Browser-to-backend |

**Why browsers can't do native gRPC:**
- Browsers don't expose raw HTTP/2 frames.
- `fetch()` and `XMLHttpRequest` abstract away the transport layer.
- gRPC-Web is a modified protocol that works within browser constraints.

### 8.8 gRPC Error Handling & Status Codes

```js
// ─── gRPC Status Codes (subset relevant to frontend) ───
const grpcStatusCodes = {
  0:  'OK',                  // Success
  1:  'CANCELLED',           // Client cancelled the request
  2:  'UNKNOWN',             // Unknown error
  3:  'INVALID_ARGUMENT',    // Bad request (like HTTP 400)
  4:  'DEADLINE_EXCEEDED',   // Timeout (like HTTP 408)
  5:  'NOT_FOUND',           // Resource not found (like HTTP 404)
  7:  'PERMISSION_DENIED',   // Forbidden (like HTTP 403)
  13: 'INTERNAL',            // Server error (like HTTP 500)
  14: 'UNAVAILABLE',         // Service unavailable (like HTTP 503)
  16: 'UNAUTHENTICATED',     // Not authenticated (like HTTP 401)
};

// ─── Error Handling in gRPC-Web ───
client.getUser(request, metadata, (err, response) => {
  if (err) {
    switch (err.code) {
      case 3:  // INVALID_ARGUMENT
        showValidationError(err.message);
        break;
      case 14: // UNAVAILABLE
        retryWithBackoff(() => client.getUser(request, metadata, callback));
        break;
      case 16: // UNAUTHENTICATED
        redirectToLogin();
        break;
      default:
        showGenericError(err.message);
    }
    return;
  }
  renderUser(response);
});
```

### 8.9 gRPC Interceptors (Middleware)

```js
// ─── Client-side Interceptor for Auth + Logging ───
class AuthInterceptor {
  intercept(request, invoker) {
    // Add auth metadata to every request
    const metadata = request.getMetadata();
    metadata['authorization'] = `Bearer ${getToken()}`;
    metadata['x-request-id'] = crypto.randomUUID();

    const start = performance.now();

    return invoker(request).then((response) => {
      const duration = performance.now() - start;
      console.log(`gRPC ${request.getMethodDescriptor().name}: ${duration}ms`);
      return response;
    }).catch((err) => {
      if (err.code === 16) { // UNAUTHENTICATED
        return refreshToken().then(() => invoker(request)); // retry
      }
      throw err;
    });
  }
}

// Apply interceptor
const client = new UserServiceClient('http://localhost:8080', null, {
  unaryInterceptors: [new AuthInterceptor()],
  streamInterceptors: [new AuthInterceptor()]
});
```

### 8.10 gRPC Load Balancing Strategies

```
┌────────────────────── Proxy-Based (L7) ──────────────────────┐
│                                                               │
│  Browser ──► Envoy Proxy ──► gRPC Server 1                    │
│                          ──► gRPC Server 2                    │
│                          ──► gRPC Server 3                    │
│                                                               │
│  Envoy understands gRPC protocol, can do:                     │
│  • Round-robin / least-connections                             │
│  • gRPC health checking                                       │
│  • Per-RPC load balancing (not per-connection!)                │
│  • Retries with status code awareness                          │
│  • Circuit breaking                                           │
│                                                               │
│  ⚠️ Plain TCP load balancers (L4) fail with gRPC because      │
│  HTTP/2 multiplexes all RPCs over one TCP connection.          │
│  L4 LB sends ALL traffic to one backend!                      │
└───────────────────────────────────────────────────────────────┘
```

[⬆ Back to Top](#top)

---

## GraphQL Subscriptions

### 9.1 What Is It?

GraphQL Subscriptions enable **real-time data** via GraphQL. Under the hood, they typically use WebSockets (via `graphql-ws` protocol) to push updates when subscribed data changes.

### 9.2 How It Works

```
Client                                Server
  │── Subscribe: subscription {       │
  │     messageAdded(room: "general") │
  │       { id text author }          │
  │   } ─────────────────────────────►│
  │                                    │
  │◄── { messageAdded: {              │  ← push
  │       id: 1, text: "Hi",          │
  │       author: "Alice"             │
  │     }}                             │
  │                                    │
  │◄── { messageAdded: {              │  ← push
  │       id: 2, text: "Hello!",      │
  │       author: "Bob"               │
  │     }}                             │
```

1. **Transport setup** — The client establishes a WebSocket connection to the GraphQL server (using the `graphql-ws` or older `subscriptions-transport-ws` protocol). Regular queries/mutations continue over HTTP; only subscriptions use the WebSocket link.

2. **Client subscribes** — The client sends a `subscription` operation (just like a query, but with the `subscription` keyword). It specifies exactly which fields it wants — GraphQL's power of client-driven data shape applies here too.

3. **Server registers the subscription** — The server parses the subscription, validates it against the schema, and registers a listener. Internally, this often hooks into a Pub/Sub system (Redis, Kafka, or in-memory) that watches for relevant events.

4. **Event triggers push** — When the subscribed data changes (e.g., a new message is created via a mutation), the server's Pub/Sub system fires an event. The subscription resolver runs, resolves the data into the exact shape the client requested, and pushes it over the WebSocket.

5. **Client receives typed data** — The pushed data arrives in the same shape as a normal GraphQL response. It integrates seamlessly with the client's GraphQL cache (e.g., Apollo's `InMemoryCache`), so the UI updates automatically.

6. **Unsubscribe** — The client can stop listening by closing the subscription (or the entire WebSocket connection). The server de-registers the listener and frees resources.

### 9.3 Basic Syntax

```graphql
# Subscription definition
subscription OnMessageAdded($room: String!) {
  messageAdded(room: $room) {
    id
    text
    author
    createdAt
  }
}
```

```js
// Client usage (Apollo)
const { data, loading } = useSubscription(MESSAGE_SUBSCRIPTION, {
  variables: { room: 'general' }
});
```

### 9.4 When to Use

- When your API is already GraphQL
- Real-time features in a GraphQL application
- Want subscription data to integrate with GraphQL cache
- Type-safe real-time updates

### 9.5 Pros & Cons

| Pros | Cons |
|---|---|
| Unified API (queries + mutations + subscriptions) | Adds WebSocket complexity |
| Client specifies exact data shape | Scalability challenges |
| Integrates with GraphQL cache | Overhead if only subscriptions are needed |
| Type-safe with codegen | |

[⬆ Back to Top](#top)

---

## Microservice Communication Patterns

> How do microservices talk to each other — and how does the frontend fit into the picture?

### 10.1 Overview: Synchronous vs Asynchronous

```
┌────────────────────────────────────────────────────────────────────┐
│           Microservice Communication Spectrum                      │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  SYNCHRONOUS (Request-Response)      ASYNCHRONOUS (Event-Driven)  │
│  ─────────────────────────────      ────────────────────────────   │
│  • REST (HTTP/JSON)                 • Message Queues (RabbitMQ)    │
│  • gRPC (HTTP/2 + Protobuf)         • Event Streams (Kafka)        │
│  • GraphQL                          • Pub/Sub (Redis, NATS)        │
│  • WebSocket (real-time sync)       • Webhooks                     │
│                                                                    │
│  Caller WAITS for response          Caller sends & moves on        │
│  Tight coupling in time             Loose coupling                 │
│  Simpler to reason about            Better fault isolation         │
│  Cascading failures possible        Eventual consistency            │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 10.2 REST vs gRPC vs GraphQL — Inter-Service Communication

| Aspect | REST | gRPC | GraphQL |
|---|---|---|---|
| **Format** | JSON (text) | Protobuf (binary) | JSON (text) |
| **Transport** | HTTP/1.1 or HTTP/2 | HTTP/2 (native) | HTTP (any version) |
| **Contract** | OpenAPI/Swagger | `.proto` files | Schema (SDL) |
| **Code generation** | Optional (OpenAPI codegen) | Built-in (protoc) | Built-in (codegen) |
| **Streaming** | No (use SSE/WS) | 4 types (unary, server, client, bidi) | Subscriptions (WS) |
| **Payload size** | Large (verbose keys) | Small (binary, no keys) | Medium (query-shaped) |
| **Browser support** | ✅ Native | ❌ Needs proxy (gRPC-Web) | ✅ Native |
| **Best for** | Public APIs, CRUD | Internal microservices | Frontend-facing APIs |
| **Latency** | Medium | Low | Medium |
| **Learning curve** | Low | Medium-High | Medium |

```
┌──────────────────── Typical Architecture ───────────────────┐
│                                                              │
│   Browser/App                                                │
│      │                                                       │
│      │ REST / GraphQL / gRPC-Web                             │
│      ▼                                                       │
│   ┌──────────────┐                                           │
│   │  API Gateway  │  (or BFF — Backend for Frontend)         │
│   │  / BFF        │                                          │
│   └──────┬───────┘                                           │
│          │                                                   │
│     ┌────┼──────────────┐                                    │
│     │    │              │                                    │
│     ▼    ▼              ▼                                    │
│   ┌────┐ ┌────┐  ┌──────────┐                                │
│   │Svc │ │Svc │  │  Svc C   │     Between services:          │
│   │ A  │ │ B  │  │          │     • gRPC (fast, typed)       │
│   └────┘ └────┘  └──────────┘     • REST (simple, universal) │
│     │       │          │          • Events (async, decoupled) │
│     └───────┼──────────┘                                     │
│             ▼                                                │
│        Message Broker                                        │
│     (Kafka / RabbitMQ / NATS)                                │
└──────────────────────────────────────────────────────────────┘
```

### 10.3 Message Brokers — Async Communication

#### RabbitMQ (Message Queue)

```
Producer ──► Exchange ──► Queue ──► Consumer

┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│ Order    │────►│ Exchange │────►│  Queue   │────►│ Payment  │
│ Service  │     │ (routing)│     │          │     │ Service  │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
```

- **Point-to-point**: One message → one consumer.
- **Pub/Sub**: One message → multiple queues → multiple consumers.
- **Message acknowledgment**: Consumer confirms processing; unacked messages are redelivered.
- **Dead letter queue (DLQ)**: Failed messages go to a separate queue for inspection.
- **Best for**: Task queues, order processing, email sending, reliable delivery.

#### Apache Kafka (Event Streaming)

```
Producers ──► Topic (Partitions) ──► Consumer Groups

┌──────────┐      ┌─────────────────────┐      ┌──────────────┐
│ Order    │─────►│ Topic: orders       │─────►│ Payment Svc  │
│ Service  │      │ ┌─P0─┐ ┌─P1─┐ ┌─P2─┐│     │ (Group A)    │
└──────────┘      │ │msg1│ │msg2│ │msg3││      └──────────────┘
┌──────────┐      │ │msg4│ │msg5│ │msg6││      ┌──────────────┐
│ Inventory│─────►│ └────┘ └────┘ └────┘│─────►│ Analytics    │
│ Service  │      └─────────────────────┘      │ (Group B)    │
└──────────┘                                   └──────────────┘
```

- **Log-based**: Messages are appended to an immutable log, retained for days/weeks.
- **Consumer groups**: Multiple services can read the same topic independently.
- **Partitions**: Enable parallel consumption; ordering guaranteed per partition.
- **Replay**: Consumers can re-read old messages (great for debugging, event sourcing).
- **Best for**: Event sourcing, activity tracking, real-time analytics, high-throughput pipelines.

#### NATS (Lightweight Pub/Sub)

```
┌──────────┐    publish     ┌──────┐    subscribe    ┌──────────┐
│ Service A │──────────────►│ NATS │───────────────►│ Service B │
└──────────┘               │      │───────────────►│ Service C │
                            └──────┘                └──────────┘
```

- **Ultra-fast**: In-memory, no persistence by default (NATS JetStream adds persistence).
- **At-most-once delivery** (core NATS) or **at-least-once** (JetStream).
- **Request-Reply**: Built-in pattern for synchronous-style calls over async transport.
- **Best for**: IoT, edge computing, lightweight microservices, real-time signaling.

#### Comparison

| Feature | RabbitMQ | Kafka | NATS |
|---|---|---|---|
| **Model** | Message queue | Event log/stream | Pub/sub |
| **Delivery** | At-least-once | At-least-once | At-most-once (core) |
| **Ordering** | Per queue | Per partition | No guarantee |
| **Persistence** | Until consumed | Configurable retention | Optional (JetStream) |
| **Throughput** | ~50K msg/s | ~1M+ msg/s | ~10M+ msg/s |
| **Replay** | ❌ | ✅ | ✅ (JetStream) |
| **Best for** | Task queues | Event streaming | Real-time signaling |

### 10.4 API Gateway Pattern

```
┌────────────────────────────────────────────────────────────────┐
│                        API Gateway                             │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Browser ──HTTP──► ┌──────────────┐ ──► User Service (gRPC)   │
│                    │  API Gateway  │ ──► Order Service (gRPC)  │
│  Mobile ──HTTP──►  │              │ ──► Payment Service (REST) │
│                    │  (Kong /     │ ──► Notification (event)   │
│  3rd Party ──────► │   Nginx /    │                            │
│                    │   AWS ALB)   │                            │
│                    └──────────────┘                            │
│                                                                │
│  Responsibilities:                                             │
│  ✅ Request routing (path-based, header-based)                 │
│  ✅ Authentication & authorization (JWT validation)            │
│  ✅ Rate limiting & throttling                                  │
│  ✅ Request/response transformation                             │
│  ✅ Load balancing                                              │
│  ✅ Circuit breaking                                            │
│  ✅ Caching                                                     │
│  ✅ Logging, metrics, tracing                                   │
│  ✅ CORS handling                                               │
│  ✅ SSL termination                                             │
│  ✅ Protocol translation (REST ↔ gRPC)                          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 10.5 BFF (Backend for Frontend) Pattern

```
┌───────────┐     ┌─────────────┐
│  Web App   │────►│ Web BFF     │──┐
│  (React)   │     │ (GraphQL)   │  │
└───────────┘     └─────────────┘  │
                                    │     ┌────────────┐
┌───────────┐     ┌─────────────┐  ├────►│ User Svc   │
│ Mobile App │────►│ Mobile BFF  │──┤     └────────────┘
│ (iOS/And)  │     │ (REST, slim)│  │     ┌────────────┐
└───────────┘     └─────────────┘  ├────►│ Order Svc  │
                                    │     └────────────┘
┌───────────┐     ┌─────────────┐  │     ┌────────────┐
│  TV App    │────►│ TV BFF      │──┘     │ Product Svc│
│            │     │ (minimal)   │───────►└────────────┘
└───────────┘     └─────────────┘
```

**Why BFF?**
- **Different clients need different data shapes** — Web needs rich data; mobile needs slim payloads.
- **Prevents over-fetching** — Each BFF aggregates exactly what its client needs.
- **Team ownership** — The frontend team owns their BFF; they don't depend on a shared API.
- **Aggregation** — BFF calls multiple downstream services and combines responses.

### 10.6 Service Mesh (Istio / Linkerd)

```
┌──────────────────────────────────────────────────────────────┐
│                        Service Mesh                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────┐        ┌──────────────────┐            │
│  │  Service A        │        │  Service B        │           │
│  │  ┌─────────────┐ │  mTLS  │  ┌─────────────┐ │           │
│  │  │   App Code   │ │◄═════►│  │   App Code   │ │           │
│  │  └──────┬──────┘ │        │  └──────┬──────┘ │           │
│  │         │        │        │         │        │           │
│  │  ┌──────▼──────┐ │        │  ┌──────▼──────┐ │           │
│  │  │ Sidecar     │ │◄══════►│  │ Sidecar     │ │           │
│  │  │ Proxy       │ │        │  │ Proxy       │ │           │
│  │  │ (Envoy)     │ │        │  │ (Envoy)     │ │           │
│  │  └─────────────┘ │        │  └─────────────┘ │           │
│  └──────────────────┘        └──────────────────┘            │
│                                                              │
│  The app doesn't know about the mesh.                        │
│  Sidecar proxies handle ALL network concerns:                │
│                                                              │
│  • mTLS (mutual TLS) — automatic encryption between services │
│  • Load balancing — intelligent routing                       │
│  • Circuit breaking — stop calling failing services           │
│  • Retries & timeouts — configurable per service              │
│  • Observability — distributed tracing, metrics, access logs  │
│  • Traffic shifting — canary deployments, A/B testing         │
│  • Rate limiting — per-service or global                      │
│                                                              │
│  Control Plane (Istiod):                                      │
│  Manages configuration, pushes policies to all sidecars.      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Why it matters for frontend:**
- The frontend developer doesn't interact with the service mesh directly.
- But knowing it exists explains **why retry/timeout/auth logic** is sometimes handled at the infrastructure level rather than in application code.
- Service mesh makes gRPC inter-service communication trivial (handles mTLS, discovery, load balancing).

### 10.7 Event-Driven Architecture (EDA)

```
┌──────────────────────────────────────────────────────────────────────┐
│                  Event-Driven Architecture                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  User places order on Web UI                                         │
│         │                                                            │
│         ▼                                                            │
│  ┌──────────────┐   OrderCreated   ┌──────────────────────────────┐  │
│  │ Order Service │ ───event──────►  │        Event Bus             │  │
│  └──────────────┘                  │  (Kafka / RabbitMQ / NATS)   │  │
│                                     └──────────┬──┬──┬────────────┘  │
│                                                │  │  │               │
│                         ┌──────────────────────┘  │  └───────────┐   │
│                         ▼                         ▼              ▼   │
│                  ┌──────────────┐  ┌──────────────┐  ┌──────────┐   │
│                  │ Payment Svc  │  │ Inventory Svc│  │ Email Svc│   │
│                  │ (charge card)│  │ (reserve qty)│  │ (confirm)│   │
│                  └──────┬───────┘  └──────────────┘  └──────────┘   │
│                         │                                            │
│                         ▼                                            │
│                  PaymentCompleted event ──► more downstream actions   │
│                                                                      │
│  Benefits:                                                           │
│  • Services are DECOUPLED (Order doesn't know about Payment)         │
│  • Easy to ADD new consumers (e.g., add Analytics service)           │
│  • FAULT TOLERANT (if Email is down, message stays in queue)         │
│  • SCALABLE (each service scales independently)                      │
│                                                                      │
│  Challenges:                                                         │
│  • Eventual consistency (not immediate)                              │
│  • Debugging distributed flows is harder                             │
│  • Message ordering can be tricky                                    │
│  • Need idempotent consumers (duplicate messages possible)           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 10.8 Saga Pattern — Distributed Transactions

In a monolith, a single DB transaction covers everything. In microservices, each service has its own DB. The **Saga pattern** coordinates multi-service transactions.

```
┌──────────────── Choreography Saga ─────────────────────────┐
│                                                             │
│  Order Svc ──OrderCreated──► Payment Svc                   │
│                               │                             │
│                          PaymentDone──► Inventory Svc       │
│                                          │                  │
│                                    ItemReserved──► Shipping │
│                                                      │      │
│                                               ShipmentCreated│
│                                                             │
│  If Payment FAILS:                                          │
│  Payment Svc ──PaymentFailed──► Order Svc (cancel order)   │
│                                                             │
│  Each service reacts to events and publishes new events.    │
│  No central coordinator.                                    │
└─────────────────────────────────────────────────────────────┘

┌──────────────── Orchestration Saga ───────────────────────┐
│                                                            │
│  ┌─────────────────┐                                       │
│  │ Saga Orchestrator│ (central coordinator)                │
│  │ (Order Saga)     │                                      │
│  └────────┬────────┘                                       │
│           │                                                │
│           ├──► Step 1: Payment Svc.charge()                │
│           │       ✅ success                                │
│           ├──► Step 2: Inventory Svc.reserve()             │
│           │       ❌ failure                                │
│           ├──► Compensate: Payment Svc.refund()            │
│           └──► Mark saga as FAILED                          │
│                                                            │
│  The orchestrator knows the full workflow and handles       │
│  compensation (rollback) for each failed step.             │
└────────────────────────────────────────────────────────────┘
```

| Approach | Pros | Cons |
|---|---|---|
| **Choreography** | Decoupled, no single point of failure | Hard to track, complex for many steps |
| **Orchestration** | Clear workflow, easier to debug | Central coordinator = potential bottleneck |

### 10.9 Circuit Breaker Pattern

```
┌──────────────── Circuit Breaker States ─────────────────────┐
│                                                              │
│   CLOSED (normal)                                            │
│   ┌────────────────────┐                                     │
│   │ Requests flow       │                                    │
│   │ through normally    │──failure threshold exceeded──┐     │
│   └────────────────────┘                               │     │
│          ▲                                             ▼     │
│          │                                    OPEN (blocked) │
│          │                               ┌─────────────────┐ │
│   success in half-open                   │ ALL requests     │ │
│          │                               │ instantly fail   │ │
│          │                               │ (fail-fast)      │ │
│          │                               └────────┬────────┘ │
│          │                                        │          │
│          │                              timeout elapsed      │
│          │                                        │          │
│          │                                        ▼          │
│   ┌──────┴──────────────┐                 HALF-OPEN          │
│   │ HALF-OPEN            │          ┌─────────────────┐      │
│   │ Allow ONE test req   │◄─────────│ Let a few test  │      │
│   │ If success → CLOSED  │          │ requests through│      │
│   │ If failure → OPEN    │          └─────────────────┘      │
│   └─────────────────────┘                                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

```js
// ─── Simple Circuit Breaker Implementation ───
class CircuitBreaker {
  constructor(fn, options = {}) {
    this.fn = fn;
    this.failureThreshold = options.failureThreshold ?? 5;
    this.resetTimeout = options.resetTimeout ?? 30000;
    this.state = 'CLOSED';   // CLOSED | OPEN | HALF_OPEN
    this.failureCount = 0;
    this.lastFailureTime = null;
  }

  async call(...args) {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime >= this.resetTimeout) {
        this.state = 'HALF_OPEN'; // try one request
      } else {
        throw new Error('Circuit breaker is OPEN — request blocked');
      }
    }

    try {
      const result = await this.fn(...args);
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
      console.warn('Circuit breaker OPENED — too many failures');
    }
  }
}

// Usage
const fetchUsers = new CircuitBreaker(
  () => fetch('/api/users').then(r => r.json()),
  { failureThreshold: 3, resetTimeout: 10000 }
);

try {
  const users = await fetchUsers.call();
} catch (err) {
  showFallbackUI(); // graceful degradation
}
```

### 10.10 How the Frontend Connects to Microservices

```
┌──────────────────────────────────────────────────────────────────┐
│              Frontend ↔ Microservice Communication               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PATTERN 1: API Gateway (most common)                            │
│  ───────────────────────────────────                             │
│  Frontend ──REST/GraphQL──► API Gateway ──gRPC──► Services       │
│  • Single entry point for all API calls                          │
│  • Gateway handles auth, rate limiting, routing                  │
│  • Frontend doesn't know about individual services               │
│                                                                  │
│  PATTERN 2: BFF (Backend for Frontend)                           │
│  ─────────────────────────────────────                           │
│  Frontend ──GraphQL──► BFF ──gRPC/REST──► Services               │
│  • BFF aggregates data from multiple services                    │
│  • Tailored to frontend needs (no over/under-fetching)           │
│  • Frontend team owns the BFF                                    │
│                                                                  │
│  PATTERN 3: Direct Service Calls (rare, not recommended)         │
│  ────────────────────────────────────────────────────             │
│  Frontend ──REST──► Service A                                    │
│  Frontend ──REST──► Service B                                    │
│  • Tight coupling, CORS issues, no centralized auth              │
│  • Only for very simple architectures                            │
│                                                                  │
│  PATTERN 4: Real-Time Layer                                      │
│  ──────────────────────────                                      │
│  Frontend ──WebSocket──► WS Gateway ──Pub/Sub──► Services        │
│  Frontend ──SSE──► Event Gateway ──Kafka──► Services             │
│  • Separate real-time channel from request-response              │
│  • Gateway subscribes to event bus, pushes to connected clients  │
│                                                                  │
│  PATTERN 5: GraphQL Federation                                   │
│  ─────────────────────────────                                   │
│  Frontend ──GraphQL──► Apollo Gateway ──► Subgraph A (Users)     │
│                                       ──► Subgraph B (Products)  │
│                                       ──► Subgraph C (Orders)    │
│  • Each microservice exposes a GraphQL subgraph                  │
│  • Apollo Router/Gateway composes them into a unified schema     │
│  • Frontend queries one schema, gateway resolves across services │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 10.11 Webhooks — Server-to-Server Push

```
┌───────────────┐    Event occurs     ┌────────────────────┐
│  Stripe       │ ──POST /webhook───► │  Your Server       │
│  (3rd party)  │    (HTTP callback)  │  (listener endpoint)│
└───────────────┘                     └────────────────────┘
```

```js
// ─── Webhook Receiver (Express) ───
app.post('/webhooks/stripe', express.raw({ type: 'application/json' }), (req, res) => {
  const sig = req.headers['stripe-signature'];

  try {
    // Verify signature (prevent forgery)
    const event = stripe.webhooks.constructEvent(req.body, sig, WEBHOOK_SECRET);

    switch (event.type) {
      case 'payment_intent.succeeded':
        handlePaymentSuccess(event.data.object);
        break;
      case 'invoice.payment_failed':
        handlePaymentFailure(event.data.object);
        break;
    }

    res.status(200).json({ received: true }); // ACK quickly
  } catch (err) {
    res.status(400).send(`Webhook Error: ${err.message}`);
  }
});
```

**Webhook Best Practices:**
- **Verify signatures** — Always validate webhook payloads to prevent spoofing.
- **Respond fast (< 5s)** — ACK immediately, process asynchronously.
- **Idempotency** — Handle duplicate deliveries (use event ID for dedup).
- **Retry handling** — Webhooks typically retry on 4xx/5xx (exponential backoff).
- **Dead letter queue** — Store failed webhooks for manual inspection.

### 10.12 Communication Pattern Decision Matrix

| Scenario | Recommended Pattern | Why |
|---|---|---|
| Frontend fetching user profile | API Gateway + REST/GraphQL | Simple CRUD, request-response |
| Real-time chat in browser | WebSocket via WS Gateway | Bidirectional, low-latency |
| Order processing pipeline | Event-driven (Kafka/RabbitMQ) | Async, multi-step, decoupled |
| Service-to-service data fetch | gRPC | Fast, typed, streaming support |
| 3rd-party payment notification | Webhook | Server-to-server push callback |
| Live dashboard updates | SSE via Event Gateway | Server-to-client, auto-reconnect |
| Video call feature | WebRTC (+ signaling server) | P2P media, ultra-low latency |
| Multi-service data aggregation | BFF or GraphQL Federation | Reduce round trips, tailor responses |
| Distributed transaction | Saga (choreography/orchestration) | No distributed DB transactions |
| Prevent cascading failures | Circuit Breaker | Fail-fast, graceful degradation |

[⬆ Back to Top](#top)

---

## Comparison Table

| Feature | HTTP | Short Poll | Long Poll | SSE | WebSocket | WebRTC |
|---|---|---|---|---|---|---|
| **Direction** | Client → Server | Client → Server | Client → Server | Server → Client | Bidirectional | Bidirectional (P2P) |
| **Latency** | Per request | Up to interval | Near real-time | Real-time | Real-time | Ultra-low |
| **Connection** | Short-lived | Short-lived | Medium-lived | Long-lived | Long-lived | Long-lived (P2P) |
| **Protocol** | HTTP | HTTP | HTTP | HTTP | WS (TCP) | UDP/TCP |
| **Binary Data** | ✅ | ✅ | ✅ | ❌ (text only) | ✅ | ✅ |
| **Auto-reconnect** | N/A | N/A | Manual | ✅ Built-in | ❌ Manual | ❌ Manual |
| **Browser Support** | Universal | Universal | Universal | Modern (IE❌) | Modern | Modern |
| **Max Connections** | 6/domain (H1) | 6/domain (H1) | 6/domain (H1) | 6/domain (H1) | No HTTP limit | No HTTP limit |
| **Scalability** | ★★★★★ | ★★★☆☆ | ★★★☆☆ | ★★★★☆ | ★★★☆☆ | ★★★★★ (P2P) |
| **Complexity** | ★☆☆☆☆ | ★☆☆☆☆ | ★★☆☆☆ | ★★☆☆☆ | ★★★☆☆ | ★★★★★ |
| **Server Cost** | Low | Medium-High | High | Medium | High | Low (P2P) |

[⬆ Back to Top](#top)

---

## Decision Flowchart Which Protocol to Pick

```
                    Start
                      │
                      ▼
             Need real-time data?
            /                    \
          No                     Yes
          │                       │
          ▼                       ▼
     Use HTTP/REST          Need bidirectional?
     or GraphQL             /                \
                          No                  Yes
                          │                    │
                          ▼                    ▼
                   Server → Client        Need media/P2P?
                   only?                  /              \
                  /        \            No               Yes
                Yes         No          │                 │
                │           │           ▼                 ▼
                ▼           ▼       WebSocket           WebRTC
          Can use SSE?    Use Long
         /           \    Polling
       Yes            No
        │              │
        ▼              ▼
       SSE         Long Polling

  ┌─────────────────────────────────────────────┐
  │  Quick Rules:                                │
  │                                              │
  │  • CRUD / one-time data → HTTP               │
  │  • Dashboard refresh → Short Polling         │
  │  • Notifications, feeds → SSE                │
  │  • Chat, collaboration → WebSocket           │
  │  • Video call → WebRTC                       │
  │  • Already on GraphQL → GraphQL Subscriptions│
  └─────────────────────────────────────────────┘
```

[⬆ Back to Top](#top)

---

## Real World Architecture Examples

### 12.1 Chat App (WhatsApp Web / Slack)

```
┌──────────┐    WebSocket     ┌──────────────┐     Pub/Sub      ┌───────┐
│  Browser  │◄══════════════►│  WS Gateway   │◄═══════════════►│ Redis  │
│  (React)  │                │  (Load Bal.)  │                  │ Pub/Sub│
└──────────┘                 └──────────────┘                  └───────┘
                                    │                               │
                              ┌─────▼─────┐                  ┌─────▼─────┐
                              │  Message   │                  │ Presence  │
                              │  Service   │                  │  Service  │
                              └───────────┘                  └───────────┘
```

**Why WebSocket?** Bidirectional: users send AND receive messages. Typing indicators, read receipts all need server push + client push.

### 12.2 Live Score Dashboard

```
┌──────────┐      SSE        ┌──────────────┐
│  Browser  │◄════════════════│  Score API    │◄── Score Feed
│  (React)  │                │  Server       │
└──────────┘                 └──────────────┘
```

**Why SSE?** Unidirectional: server pushes scores. Client only reads. Auto-reconnection is a bonus.

### 12.3 Google Docs (Collaborative Editing)

```
┌────────┐  WebSocket  ┌────────────┐  CRDT/OT   ┌──────────┐
│ User A  │◄══════════►│  Collab     │◄══════════►│    DB    │
│ Browser │            │  Server     │            └──────────┘
└────────┘            └────────────┘
┌────────┐  WebSocket       ▲
│ User B  │◄════════════════┘
│ Browser │
└────────┘
```

**Why WebSocket?** Both users send edits AND receive others' edits in real-time. OT/CRDT operations need bidirectional, low-latency channel.

### 12.4 Uber / Ride Tracking

```
                           ┌──────────────┐
  Driver App ──HTTP POST──►│  Location    │
  (GPS updates)            │  Ingestion   │
                           │  Service     │
                           └──────┬───────┘
                                  │
                           ┌──────▼───────┐
  Rider App ◄─── SSE/WS ──│  Tracking    │
  (map updates)            │  Service     │
                           └──────────────┘
```

**Why SSE for rider?** Rider only receives location updates (unidirectional). Driver sends via HTTP POST (infrequent, bursty).

[⬆ Back to Top](#top)

---

## Interview Questions and Answers

### Q1: What's the difference between WebSocket and SSE?

| Aspect | SSE | WebSocket |
|---|---|---|
| Direction | Server → Client only | Full-duplex (both ways) |
| Protocol | HTTP | Separate WS protocol over TCP |
| Data format | Text only | Text + Binary |
| Reconnection | Automatic (built-in) | Manual implementation needed |
| Browser API | `EventSource` | `WebSocket` |
| Use case | Notifications, live feeds | Chat, gaming, collaboration |

**Key insight:** Use SSE when you only need server-to-client push (simpler). Use WebSocket when you need bidirectional communication.

---

### Q2: How would you scale WebSocket connections?

1. **Sticky sessions** — Use load balancer (IP hash / cookie) to route a client to the same server.
2. **Pub/Sub layer** — Use Redis Pub/Sub, Kafka, or NATS so that any server can broadcast to any client.
3. **Horizontal scaling** — Each WS server handles N connections. Add more servers behind the load balancer.
4. **Connection limits** — A single server can handle ~10K-100K WebSocket connections (tune OS: file descriptors, memory).
5. **Dedicated WS gateway** — Separate WebSocket handling from business logic.

---

### Q3: What happens if a WebSocket connection drops?

- The `onclose` event fires on the client.
- You must implement **manual reconnection** with:
  - **Exponential backoff** — `1s → 2s → 4s → 8s → ...` to avoid thundering herd.
  - **Jitter** — Add random delay to prevent all clients reconnecting simultaneously.
  - **Message queue** — Buffer unsent messages during disconnection.
  - **Last event ID** — Track the last received message to resume from where you left off.
  - **Max retries** — Give up after N attempts and show an error UI.

---

### Q4: Explain the WebSocket handshake.

1. Client sends an HTTP `GET` with `Upgrade: websocket` header and a random `Sec-WebSocket-Key`.
2. Server responds with `101 Switching Protocols` and a `Sec-WebSocket-Accept` (computed from the client's key + a magic GUID).
3. This proves the server understands WebSocket.
4. After this, the connection switches from HTTP to the WebSocket frame-based protocol.

---

### Q5: How does SSE handle reconnection?

- `EventSource` has **built-in auto-reconnection**.
- When the connection drops, the browser waits (default 3s, configurable via `retry:` field) and reconnects.
- On reconnection, the browser sends the `Last-Event-ID` header with the last received `id:` value.
- The server can use this ID to **resume from where the client left off**, avoiding duplicate or lost events.
- This makes SSE more reliable out-of-the-box than WebSocket for server-push scenarios.

---

### Q6: When would you choose Long Polling over WebSocket?

- **Proxy/firewall restrictions** — Some corporate networks block WebSocket upgrades. Long Polling works on plain HTTP.
- **Simplicity** — No special server infrastructure needed; any HTTP server works.
- **Low-frequency updates** — If updates are infrequent (once every few seconds), the overhead of maintaining a WebSocket connection isn't justified.
- **Legacy compatibility** — Older systems that can't support WS.
- **Fallback** — Libraries like Socket.IO use Long Polling as a fallback when WebSocket fails.

---

### Q7: What is the "thundering herd" problem and how do you avoid it?

When a server goes down, all connected clients (thousands) detect the disconnection simultaneously and try to reconnect at the same time → overwhelming the server.

**Solutions:**
1. **Exponential backoff** — Each retry waits longer: `delay = baseDelay * 2^attempt`
2. **Jitter** — Add random delay: `delay = baseDelay * 2^attempt + random(0, 1000)`
3. **Max backoff cap** — Don't exceed a maximum (e.g., 30s)
4. **Connection admission control** — Server rejects excess connections with `503 Retry-After`

---

### Q8: How would you implement real-time notifications in a large-scale app?

```
┌────────┐   SSE/WS    ┌────────────┐    Subscribe    ┌───────┐
│ Client  │◄════════════│ Notif.     │◄════════════════│ Redis  │
│ Browser │             │ Gateway    │                  │ Pub/Sub│
└────────┘             └────────────┘                 └───────┘
                                                          ▲
                        ┌────────────┐    Publish          │
                        │ Any Backend │═══════════════════►│
                        │ Service     │
                        └────────────┘
```

**Key decisions:**
1. **SSE** if notifications are server → client only (simpler)
2. **WebSocket** if user can mark-as-read or interact in real-time
3. Use **Redis Pub/Sub** for cross-server broadcasting
4. **Fallback to Short Polling** for environments where SSE/WS fail
5. Store notifications in DB for **offline users** (pull on reconnect)
6. Use **Service Workers** for push notifications when app is closed

---

### Q9: Compare HTTP/2 Server Push vs SSE vs WebSocket.

| Feature | HTTP/2 Server Push | SSE | WebSocket |
|---|---|---|---|
| Purpose | Push assets (CSS, JS) | Push events/data | Bidirectional messaging |
| Initiated by | Server (with request) | Server (after subscribe) | Either side |
| Use case | Asset preloading | Live data feeds | Chat, games |
| Client control | No (server decides) | Yes (EventSource) | Yes (send/receive) |
| Status | Deprecated in Chrome | Active & supported | Active & supported |

**Note:** HTTP/2 Server Push was designed for assets, not application data. It has been removed from Chrome (2022) due to low real-world benefit. Don't confuse it with SSE.

---

### Q10: How do you handle authentication with WebSocket?

| Option | Approach | Trade-off |
|---|---|---|
| **Token in URL** | `new WebSocket('wss://...?token=jwt')` | Simple but token leaks in logs/history |
| **Cookie-based** | Cookies sent automatically during handshake | Works with existing session auth |
| **Auth after connect** | First message = `{ type: 'auth', token }` | Recommended; server closes if invalid |
| **Ticket-based** | Get one-time ticket via HTTP, connect with ticket | Most secure; short TTL, single use |

---

### Q11: What is the difference between STUN and TURN in WebRTC?

**Answer:**
- **STUN** helps a peer discover its own public IP and port by asking a STUN server. It's lightweight and free. Once the public address is known, the two peers attempt a direct P2P connection.
- **TURN** acts as a relay server. If direct connection fails (symmetric NAT, strict firewall), both peers send data to the TURN server, which forwards it to the other peer.
- **ICE** orchestrates the process: it gathers candidates from host, STUN (srflx), and TURN (relay), then tests pairs to find the best working path.
- **Key insight:** ~85% of connections succeed with STUN alone. TURN is the expensive fallback that always works.

---

### Q12: How do microservices communicate with each other?

**Answer:**

Microservices use a combination of synchronous and asynchronous communication:

1. **Synchronous (request-response):**
   - **gRPC** — Preferred for internal communication (binary, typed, fast, streaming).
   - **REST** — Universal, simple, good for public-facing APIs.
   - **GraphQL** — Good when frontend needs flexible queries across services.

2. **Asynchronous (event-driven):**
   - **Message queues** (RabbitMQ) — Point-to-point task processing.
   - **Event streams** (Kafka) — Pub/sub with replay capability.
   - **Pub/Sub** (NATS, Redis) — Lightweight real-time event distribution.

3. **Patterns:**
   - **API Gateway** — Single entry point routing requests to services.
   - **BFF** — Per-client backend that aggregates microservice data.
   - **Service Mesh** (Istio) — Infrastructure-level networking (mTLS, retries, observability).
   - **Saga** — Distributed transaction coordination.
   - **Circuit Breaker** — Prevent cascading failures.

---

### Q13: When would you use gRPC over REST for microservices?

**Answer:**

| Use gRPC when... | Use REST when... |
|---|---|
| Internal service-to-service calls | Public-facing APIs |
| High throughput / low latency needed | Human-readable debugging needed |
| Strong typing and contracts matter | Simplicity is prioritized |
| Streaming (server, client, bidi) needed | Browser clients (without proxy) |
| Polyglot microservices (codegen for any lang) | Wide ecosystem tooling needed |

**Key insight:** Many architectures use BOTH — REST/GraphQL for external (browser-facing) traffic and gRPC for internal inter-service calls.

---

### Q14: Explain the Saga pattern for distributed transactions.

**Answer:**

In microservices, you can't use a single DB transaction across services. The Saga pattern breaks a transaction into a sequence of local transactions, each with a **compensating action** (rollback).

**Example: E-commerce order flow:**
1. Order Service → Create order (compensate: cancel order)
2. Payment Service → Charge card (compensate: refund)
3. Inventory Service → Reserve stock (compensate: release stock)
4. Shipping Service → Create shipment (compensate: cancel shipment)

If step 3 fails, the saga runs compensations in reverse: refund card → cancel order.

**Two approaches:**
- **Choreography:** Services react to events (decoupled, but hard to track).
- **Orchestration:** A central saga coordinator drives the flow (easier to debug).

---

### Q15: What is a Service Mesh, and why does it matter for frontend engineers?

**Answer:**

A service mesh (e.g., Istio, Linkerd) adds a **sidecar proxy** (like Envoy) alongside each microservice to handle networking concerns transparently:
- **mTLS** — Automatic encryption between services.
- **Load balancing** — Intelligent, per-request routing.
- **Retries & circuit breaking** — Resilience without code changes.
- **Observability** — Distributed tracing, metrics.

**Frontend relevance:** When you wonder why your API calls have retries, timeouts, or auth "built in" without explicit code — a service mesh may be handling it at the infrastructure level. Understanding this helps in debugging latency and failure scenarios.

---

### Bonus: Quick One-Liners for Interviews

| Question | Answer |
|---|---|
| WebSocket port? | Same as HTTP: 80 (ws://) and 443 (wss://) |
| SSE max connections? | 6 per domain on HTTP/1.1, unlimited on HTTP/2 |
| WebSocket frame overhead? | 2-14 bytes (vs HTTP headers ~200-800 bytes) |
| Can SSE send binary? | No, text only. Use base64 encoding as workaround. |
| Socket.IO = WebSocket? | No. Socket.IO is a library that CAN use WebSocket but also falls back to Long Polling. It adds rooms, acknowledgements, broadcasting. |
| Is WebSocket RESTful? | No. WS is stateful and doesn't follow REST principles. |
| gRPC vs REST? | gRPC: binary (protobuf), typed, streaming. REST: text (JSON), flexible, simpler. |
| WebRTC need server? | Yes, for signaling. Data transfer is P2P after connection. |
| STUN vs TURN? | STUN discovers public IP (lightweight). TURN relays data (expensive fallback). |
| SFU vs MCU? | SFU forwards streams (low CPU). MCU mixes streams into one (high CPU). |
| Kafka vs RabbitMQ? | Kafka: event log with replay. RabbitMQ: traditional message queue. |
| What is a BFF? | Backend for Frontend — a per-client API layer that aggregates microservice data. |
| Circuit breaker? | Stops calling a failing service after N failures. Retries after a timeout. |

---

> **Interview Tip:** When asked "How would you build X in real-time?", structure your answer as:
> 1. **Identify the data flow** — unidirectional or bidirectional?
> 2. **Pick the protocol** — SSE for server push, WS for bidirectional, Short Poll as fallback
> 3. **Address scaling** — Pub/Sub layer, sticky sessions, horizontal scaling
> 4. **Handle failures** — Reconnection strategy, message buffering, offline support
> 5. **Security** — Auth mechanism, rate limiting, origin validation

[⬆ Back to Top](#top)

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design

[⬆ Back to Top](#top)
