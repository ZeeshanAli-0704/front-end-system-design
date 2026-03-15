# The Evolution of the Web: Comparing HTTP/1.1, HTTP/2, and HTTP/3

If you've ever wondered why the modern web feels so much snappier than it did a decade ago, the answer lies in the "plumbing" of the internet. The Hypertext Transfer Protocol (HTTP) has undergone a massive transformation to keep up with our data-heavy world.

Here is a breakdown of how the web evolved from the "one-at-a-time" days of HTTP/1.1 to the lightning-fast, mobile-first world of HTTP/3.

---

<a id="top"></a>

## Table of Contents

- [The Bottleneck HTTP 1.1](#the-bottleneck-http-11)
- [The Efficiency Upgrade HTTP 2](#the-efficiency-upgrade-http-2)
- [The Future is Here HTTP 3 (QUIC)](#the-future-is-here-http-3-quic)
- [Technical Comparison Table](#technical-comparison-table)
- [Summary Which Should You Use](#summary-which-should-you-use)

[⬆ Back to Top](#top)

---

## The Bottleneck HTTP 1.1

In the early days, HTTP/1.1 was the gold standard. However, it had a major flaw: **Head-of-Line (HOL) Blocking**.

Imagine a grocery store with only one checkout lane. Even if you only have a candy bar, you have to wait for the person with a full cart to finish. In HTTP/1.1, the browser could only handle **one request at a time per connection**. To get around this, browsers typically allow a **maximum of 6 concurrent TCP connections** to a single domain. If your website has 100 images, the browser has to queue them up in batches of 6, creating a significant delay.

[⬆ Back to Top](#top)

---

## The Efficiency Upgrade HTTP 2

Released in 2015, HTTP/2 introduced **True Multiplexing**. Instead of opening multiple TCP connections, it uses a single connection and splits data into "streams."

* **HPACK Compression:** It shrinks headers (metadata about the request), saving precious bytes.
* **The Catch:** While it solved application-level blocking, it still relies on **TCP**. If a single packet is lost in transit, TCP pauses *everything* to wait for that packet to be resent. This is known as TCP-level Head-of-Line blocking.

[⬆ Back to Top](#top)

---

## The Future is Here HTTP 3 (QUIC)

HTTP/3 is a radical departure because it ditches TCP entirely in favor of **QUIC (built over UDP)**. This change addresses the final frontier of web performance: unreliable networks.

* **No More Waiting:** If a packet for "Image A" is lost, "Image B" continues to load without interruption.
* **Zero Handshake (0-RTT):** It combines the connection and security handshake into one, making the initial "hello" between your phone and the server nearly instant.
* **Connection Migration:** Have you ever walked out of your house, switched from Wi-Fi to LTE, and had your music stream or download fail? HTTP/3 uses a unique Connection ID, allowing your session to survive IP changes seamlessly.

[⬆ Back to Top](#top)

---

## Technical Comparison Table

| Aspect | HTTP/1.1 | HTTP/2 | HTTP/3 |
| --- | --- | --- | --- |
| **Transport** | TCP | TCP | **QUIC over UDP** |
| **Concurrency** | Max ~6 requests/conn | Multiplexed Streams | Multiplexed (No HOL) |
| **HOL Blocking** | Connection-level | TCP-level (on packet loss) | **None** |
| **Compression** | None (Plaintext) | HPACK | QPACK |
| **TLS** | Optional | Effectively Mandatory | **Mandatory (TLS 1.3)** |
| **Handshake** | Multi-step | Faster (ALPN) | **Instant (0-RTT/1-RTT)** |
| **Connection Migration** | No | No | **Yes (Wi-Fi to LTE)** |

[⬆ Back to Top](#top)

---

## Summary Which Should You Use

* **HTTP/1.1:** Best left for legacy internal systems.
* **HTTP/2:** The current industry standard for general web traffic.
* **HTTP/3:** Essential for mobile apps, high-latency regions, and performance-critical platforms (like Google, Facebook, and Netflix).

The shift to HTTP/3 represents a "mobile-first" internet where connections are no longer assumed to be stable, but are optimized to be resilient.

[⬆ Back to Top](#top)

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design

[⬆ Back to Top](#top)