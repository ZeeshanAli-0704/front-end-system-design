# System Design Interview: Autocomplete / Type‑ahead System for a Search Box

Autocomplete (also called *type‑ahead*) is one of the most visible and performance‑critical components of any search system. As a user types characters into a search box, the system continuously predicts and displays the most relevant full queries. These suggestions are not random strings — they are derived from **historical search data**, **user behavior**, and **recent trends**.

At small scale, autocomplete may look simple. At Google‑scale, however, it becomes a complex distributed system problem involving **ultra‑low latency**, **massive read traffic**, **frequent data updates**, and **global availability**.

This article presents a **complete, interview‑ready design** of a large‑scale autocomplete system. The focus is on *clear explanation*, *approach*, *data flow*, and *practical trade‑offs*, supported by examples at every step.

---

## 1. Understanding the Problem

When a user types a query like:

```
"sys"
```

the system should instantly suggest:

```
"system design"
"system design interview"
"system configuration"
```

These suggestions should:

* Start with the typed prefix
* Be ranked by popularity and freshness
* Update as user search behavior changes
* Appear almost instantly (within a few milliseconds)

The challenge is not just finding matching strings, but doing it **fast**, **reliably**, and **at massive scale**.

---

## 2. Functional Requirements

The autocomplete system must:

1. Return the **top K suggestions** for a given prefix (commonly K = 3 or 10).
2. Rank suggestions based on:

   * **Frequency**: how often the query is searched
   * **Recency**: newer searches should matter more than very old ones
3. Continuously evolve as new queries are searched.

### Public APIs

```text
getSuggestions(prefix) → List<String>
addToDB(query)
```

* `getSuggestions(prefix)` is a **read‑heavy API** that must be extremely fast.
* `addToDB(query)` records searches and contributes to future suggestions, but does not update the live index synchronously.

---

## 3. Non‑Functional Requirements

Autocomplete systems are dominated by non‑functional constraints.

### Low Latency

Users expect suggestions to appear instantly while typing. Any noticeable delay degrades user experience. In practice, this means:

* Responses typically under **50 ms**
* Most requests served from memory, not disk

### High Scalability

A global search engine serves **hundreds of thousands of requests per second**. The system must scale horizontally by adding more machines.

### High Availability

Autocomplete should almost never be down. Failures must be masked using replication and redundancy.

### Global Reach

Users across the world should receive equally fast responses. Data must be geographically distributed.

### Security and Privacy

Search queries can contain sensitive information. Logs and stored data must comply with privacy regulations and retention policies.

---

## 4. Capacity Estimation

Capacity estimation demonstrates whether the design is realistic.

### Traffic Assumptions

* Total searches per day: **3 billion**
* Average searches per user session: **3**
* Seconds per day: **86,400**

### Queries Per Second

```
QPS = (3 × 10^9 × 3) / 86,400 ≈ 104,000 QPS
```

This is the **average QPS**. Peak traffic can easily be 2–3× higher.

### Storage Estimation

* Average query length: 15 characters
* Storage per character: 2 bytes

```
Per query storage ≈ 30 bytes
1.5B unique queries/day ≈ 45 GB/day
```

This volume clearly requires distributed storage and aggressive pruning of old or unpopular queries.

---

## 5. Choosing the Core Data Structure

Autocomplete is fundamentally a **prefix‑matching problem**. Given a prefix, the system must efficiently retrieve all strings that start with it.

### Why a Trie Works Well

A **Trie (prefix tree)** stores strings character by character. Each node represents a prefix, and all descendants share that prefix.

Key advantages:

* Prefix lookup time depends only on prefix length
* No scanning of unrelated strings
* Naturally groups similar queries together

### Example

Stored queries:

```
be, bee, bell, bent, belt
```

Trie traversal for prefix `"be"` leads to a subtree containing:

```
bee, bell, bent, belt
```

This is exactly what autocomplete needs.

---

## 6. Optimizing the Trie Structure

A naive trie can become extremely large and deep when storing billions of queries.

### Compressed Trie (Radix Tree)

In many cases, a node has only one child. These chains can be merged into a single edge storing multiple characters.

Benefits:

* Reduced memory usage
* Fewer pointer traversals
* Faster lookups

### Storing Popularity Information

> *How popular is a complete search query?*

Each complete query ends at a **terminal node**. At this node, we store:

* Search frequency (or a weighted score combining frequency and recency)

#### Example

Queries entered by users:

```
"apple" → searched 100 times
"application" → searched 60 times
"app store" → searched 200 times
```

In the trie:

```
app
 ├── apple (freq=100)
 ├── application (freq=60)
 └── app store (freq=200)
```

At this stage:

* We **only know popularity of full words**
* We **cannot yet answer fast prefix queries**

This section answers:

> *“How popular is each query?”*

---

## 7. How Top‑K Suggestions Are Retrieved

> *Given a prefix, how do we return the best suggestions instantly?*

### Naive Approach

For a prefix like `"be"`:

1. Traverse to the prefix node
2. Traverse the entire subtree
3. Collect all terminal queries
4. Sort by frequency

This approach is far too slow and unpredictable for real‑time systems.

### Pre‑Computed Top‑K Optimization

To guarantee fast responses, each trie node stores the **top K suggestions** for that prefix.

Instead of searching the subtree at query time, the system simply:

```
Traverses to prefix node → returns stored top K
```

To reduce storage overhead:

* Only references to terminal nodes are stored
* Frequencies are stored alongside references

This trades additional memory for extremely fast reads, which is acceptable because autocomplete is read‑heavy.

---

## 8. Building and Maintaining Top‑K Lists

Top‑K lists are built **bottom‑up**:

1. Leaf nodes know their own frequency
2. Parent nodes merge children’s top‑K lists
3. Only the top K entries are retained
4. This process propagates upward to the root

- This ensures that every prefix node always has correct and pre‑sorted suggestions.

### Think of this as two layers:

- *Layer 1: Popularity Tracking (Data Layer)*

    - Tracks how often full queries are searched

    - Stored at terminal nodes

    - Updated offline via logs and aggregation

- *Layer 2: Fast Retrieval (Serving Layer)*

    - Uses popularity data to compute rankings

    - Stores only top-K per prefix

    - Optimized for read latency

---


## 9. High-Level System Architecture

The autocomplete system is designed as a **read-heavy, latency-critical pipeline**. The entire architecture is optimized so that the most common prefixes are resolved as early as possible, using the fastest storage available, while protecting deeper layers from unnecessary load.

---

#### Overall Architectural Idea

At a high level, the system follows this principle:

> **Serve from memory first → fall back to structured storage → never compute on the request path**

Autocomplete is not a compute problem; it is a **data access problem**. The architecture reflects that.

---

#### Logical Components

```
User
 ↓
Browser / Mobile Client
 ↓
Global Load Balancer
 ↓
Stateless Application Servers
 ↓
In-Memory Cache (Redis)
 ↓
Distributed Trie Storage
```

Each layer has a **single responsibility**, which keeps the system scalable and easy to reason about.

---

#### Step-by-Step Request Flow (Detailed)

###### 1. Client Layer (Browser / Mobile)

* The client sends the **current prefix**, not the full query.
* Requests are **debounced** (e.g., 150 ms) to avoid flooding the backend.
* Example user typing:

  ```
  h → ho → how
  ```

  Only meaningful transitions trigger network calls.

Payload example:

```json
{
  "prefix": "how",
  "k": 5,
  "locale": "en-IN"
}
```

---

###### 2. Load Balancer

The load balancer is responsible for:

* Routing traffic to the **nearest region**
* Distributing requests evenly across application servers
* Removing unhealthy servers from rotation

In large systems:

* Global LB (GeoDNS / Anycast)
* Regional LB inside each data center

This keeps **network latency minimal**.

---

###### 3. Application Servers (Stateless)

Application servers act as **coordinators**, not data holders.

They handle:

* Input normalization
  (`"How "` → `"how"`)
* Validation (empty prefix, max length)
* Cache lookup orchestration
* Fallback to trie storage if needed

Because they are stateless:

* Scaling = add more servers
* No data migration required

---

###### 4. Redis Cache (Hot Prefix Layer)

Redis is the **first data stop**.

######## Cache key structure

```
autocomplete:{locale}:{prefix}
```

Example:

```
autocomplete:en:how
```

Value:

```json
[
  "how to cook rice",
  "how to lose weight",
  "how to make tea"
]
```

Why Redis works well here:

* A **small set of prefixes dominates traffic**
* Memory access is extremely fast (sub-millisecond)
* TTL ensures stale suggestions fade out naturally

> Prefixes like `"a"`, `"th"`, `"how"` may be served **99% from cache**.

---

###### 5. Cache Miss Path (Trie Lookup)

If Redis does not contain the prefix:

* The request is forwarded to **distributed trie storage**
* The trie node already stores **pre-computed top-K**
* No traversal or sorting is done at request time

Important detail:

* Trie servers are **read-only** from the serving path
* Writes and rebuilds happen offline

This keeps latency predictable even under load.

---

###### 6. Response & Cache Population

Once suggestions are retrieved:

* Results are returned to the client
* The same result is written back to Redis
* A TTL (e.g., 10–30 minutes) is applied

This turns a **cold prefix into a hot one** automatically.

---

#### Failure Handling at HLD Level

###### If Redis goes down

* Application servers bypass cache
* Read directly from trie storage
* Latency increases slightly but system remains functional

###### If a Trie shard goes down

* Traffic is routed to replica
* Load balancer avoids unhealthy nodes

###### If an App Server goes down

* Load balancer removes it
* No data loss (stateless)

---

#### Regional Architecture (Global Scale)

```
User (India)
   ↓
India Load Balancer
   ↓
India App Servers
   ↓
India Redis
   ↓
India Trie Replica
```

Each region:

* Serves local users
* Has its own cache
* Shares replicated trie data

This avoids cross-continent latency.

### Logical Architecture Diagram (Conceptual)

```
+---------+      +--------------+      +------------------+
|  Client | ---> | Load Balancer| ---> | Application Server|
+---------+      +--------------+      +------------------+
                                              |
                         Cache Hit            | Cache Miss
                        +-----------+         v
                        |  Redis    |     +-----------------+
                        +-----------+     | Distributed Trie|
                                           |    Storage     |
                                           +-----------------+
```

---



## 10. Storage, Replication, and Availability

Storing the entire trie on a single machine is not feasible.

### Distributed Trie Storage
At large scale, the autocomplete trie cannot live on a single machine. The number of prefixes, the memory required to store pre-computed top-K suggestions, and the sheer read traffic make a single-node design both fragile and non-scalable. To address this, the trie is logically one structure but physically distributed across many servers.

* Partitioning the Trie
The trie is split into independent partitions, where each partition is responsible for a subset of prefixes. A common approach is to partition by prefix ranges. For example:

- One shard handles prefixes starting with a–c

- Another handles d–f

- More granular splits can be done for hot prefixes like a, s, or th

* Replication Strategy
Every trie partition is replicated across multiple servers.

Replication serves several purposes:

- High Availability
If one node crashes, traffic is instantly routed to another replica without rebuilding or reloading the trie.

- Fault Tolerance
Hardware failures, network issues, or rolling deployments do not interrupt autocomplete availability.

- Read Scalability
Since autocomplete is read-heavy, replicas can share traffic and handle large QPS spikes.


Possible storage technologies:

* Cassandra (large‑scale NoSQL storage)

    - Cassandra is a good fit when:

        - Trie partitions are large

        - High read throughput is required

        - Data needs to be replicated across regions

    - Usage pattern:

        - Each partition is stored as serialized trie blocks

        - Data is loaded into memory on startup

        - Cassandra acts as durable backing storage and replication layer

    - Strengths:

    - Horizontal scalability

    - High availability

    - Tunable consistency for reads

* ZooKeeper (coordination and metadata)
ZooKeeper is not used to store the full trie.

    - Instead, it is used for:

    - Metadata about partitions

    - Prefix-to-server mappings

    - Leader election (if needed)

    - Atomic version switching during trie updates

* Replication ensures:

    * High availability
    * Fault tolerance
    * Read scalability

---

## 11. Splitting the Trie Across Servers 

Once range-based partitioning is chosen, the trie must be physically split in a way that keeps routing simple and lookup fast.

---

### Alphabet-Based Sharding

The most straightforward approach is alphabet-based sharding.

**Example**

```
Shard 1 → a–k
Shard 2 → l–z
```

Each shard:

* Contains its own trie fragment
* Has multiple replicas
* Can independently serve autocomplete queries

This approach ensures:

* Deterministic routing
* Single-shard reads
* Predictable latency

---

### Finer-Grained Sharding

As data and traffic grow, shards can be split further.

**Examples**

```
Shard 1 → a
Shard 2 → b
Shard 3 → c–d
```

or even:

```
Shard A1 → ap
Shard A2 → am
Shard A3 → an
```

This incremental splitting allows the system to scale **without changing its core design**.

Routing remains prefix-based, fast, and easy to reason about.

---

### Why This Works Well

* Autocomplete requests hit **exactly one shard**
* No result aggregation is needed
* Latency remains low even at global scale

---

### Alphabet‑Based Sharding

Queries are partitioned based on starting characters:

* `a–k` → Shard‑1
* `l–z` → Shard‑2

Each shard has replicas.

### Finer‑Grained Sharding

As data grows:

* Individual letters or letter ranges are assigned separate shards
* Shards can be split further when they become too large

This approach keeps routing simple and predictable.

---

## 12. Updating the Trie Efficiently

Updating the trie for every search is not practical at scale.

### Logging and Aggregation

1. Search queries are logged asynchronously
2. Frequencies are stored in a temporary store (e.g., Cassandra)
3. Periodic batch jobs aggregate counts

### Pruning

During aggregation:

* Very old queries are discarded
* Queries below a frequency threshold are ignored

### Applying Updates

* A new trie is built offline using aggregated data
* Once ready, it atomically replaces the old trie

This ensures read traffic is never blocked.

---

## 13. Deployment Strategies

### Blue‑Green Deployment

* New trie built in parallel
* Traffic switches after validation
* Old trie is discarded

### Master‑Slave Strategy

* Master serves queries
* Slave is updated
* Roles are swapped

Both approaches ensure zero downtime.

---

## 14. Persisting the Trie

To support fast recovery:

* Trie snapshots are periodically written to disk
* Nodes are serialized level‑by‑level

Example serialization:

```
C2,A2,R1,T,P,O1,D
```

On restart, the trie is reconstructed and top‑K lists are recomputed.

---

## 15. Data Partitioning Strategies

When the autocomplete dataset grows beyond the capacity of a single machine, the system must decide **how to split data across multiple servers**. This decision directly impacts latency, scalability, and operational complexity. Different partitioning strategies exist, each with trade-offs. Understanding them helps justify why a particular approach is chosen later.

---

### Range-Based Partitioning

In range-based partitioning, data is split using **natural key ranges**, usually based on prefixes.

**Example**
If queries are English words:

```
Shard 1 → a–c
Shard 2 → d–f
Shard 3 → g–z
```

All queries starting with `"ap"` or `"car"` live in Shard 1, while `"dog"` lives in Shard 2.

**Why it works well for autocomplete**

* Prefix routing is straightforward
* Only **one shard** needs to be queried for a prefix
* Trie traversal remains efficient and localized

**Limitation**

* Popular prefixes (`a`, `s`, `th`) can receive disproportionately high traffic
* This can create **hot shards**, requiring further splitting

Range-based partitioning is simple and efficient but must be refined as data grows.

---

### Capacity-Based Partitioning

In capacity-based partitioning, servers are filled **until they reach memory or CPU limits**, after which new data spills into the next server.

**Example**

```
Server 1 stores prefixes: a → apple → apply → amazon (memory full)
Server 2 stores prefixes: aws → banana → ball
```

**Advantages**

* Maximizes hardware utilization
* Prevents a single server from becoming too large

**Challenges**

* Prefix-to-server mapping becomes dynamic
* Routing logic must consult metadata to know where a prefix lives
* Harder to reason about during failures or rebalancing

This approach is often combined with metadata services (like ZooKeeper) but adds operational complexity.

---

### Hash-Based Partitioning

Hash-based partitioning distributes data using a hash function, typically with **consistent hashing**.

**Example**

```
hash("apple")  → Server 2
hash("apply")  → Server 7
hash("app")    → Server 4
```

**Strength**

* Excellent load balancing
* Minimizes hotspots

**Why it fails for autocomplete**

Autocomplete depends on **prefix grouping**. With hashing:

* Words sharing the same prefix end up on different servers
* A single autocomplete request must query **multiple shards**
* Results must be aggregated and sorted

This adds network hops and latency, which is unacceptable for real-time suggestions.

As a result, hash-based partitioning is generally **avoided** for prefix-search systems.

---

### Summary of Strategy Selection

While multiple partitioning strategies exist, **range-based partitioning aligns best with prefix-based queries**, making it the natural foundation for autocomplete systems. The following sections build upon this choice.

---

---

### Distributed Trie Storage (Expanded)

At large scale, an autocomplete trie cannot live on a single machine. The number of prefixes, the memory required to store pre-computed top-K suggestions, and the sheer read traffic make a single-node design both fragile and non-scalable.

To solve this, the trie is treated as:

* **Logically one data structure**
* **Physically distributed across many machines**

Each machine stores only a portion of the trie but behaves as if it is part of a single system.

---

### Partitioning the Trie

The trie is divided into **independent partitions**, where each partition owns a subset of prefixes.

**Example partitioning**

```
Shard 1 → prefixes a–c
Shard 2 → prefixes d–f
Shard 3 → prefixes g–z
```

When a user types `"car"`, the system directly routes the request to **Shard 1**, avoiding unnecessary fan-out.

For high-traffic prefixes, finer splits are introduced:

```
Shard A1 → a
Shard A2 → b
Shard A3 → c
```

This keeps shard sizes manageable and reduces hotspots.

---

### Replication Strategy

Each trie partition is replicated across multiple servers.

**Example**

```
Shard 1 → Replica 1, Replica 2, Replica 3
```

Replication serves multiple purposes:

#### High Availability

If one replica goes down, traffic is immediately routed to another replica without interrupting autocomplete suggestions.

#### Fault Tolerance

Hardware failures, deployments, or network issues do not impact system correctness.

#### Read Scalability

Autocomplete is heavily read-oriented. Replicas allow traffic to be distributed, enabling the system to handle massive QPS spikes.

---

### Storage Technologies

#### Cassandra (Durable Storage Layer)

Cassandra is well-suited when:

* Trie partitions are large
* Read throughput is extremely high
* Data must be replicated across data centers

**Typical usage pattern**

* Trie partitions are stored as serialized blocks
* On startup, data is loaded into memory
* Cassandra acts as the source of truth and replication backbone

**Strengths**

* Horizontal scalability
* Built-in replication
* Tunable read consistency

---

#### ZooKeeper (Coordination & Metadata)

ZooKeeper is **not used to store the trie itself**.

Instead, it manages:

* Prefix-to-shard mappings
* Replica health information
* Leader election (if needed)
* Atomic switching between trie versions during updates

ZooKeeper ensures that **routing decisions are consistent and correct** across the system.

---

---

## 16. Trade‑Off Summary

| Aspect         | Decision           | Explanation                  |
| -------------- | ------------------ | ---------------------------- |
| Data Structure | Trie               | Efficient prefix lookup      |
| Ranking        | Pre‑computed Top‑K | Predictable low latency      |
| Cache          | Redis              | Absorb hot prefixes          |
| Updates        | Offline batching   | Protect read path            |
| Storage        | Distributed shards | Scalability and availability |

---

## 17. Final Thoughts

Autocomplete is a **read‑heavy, latency‑sensitive** distributed system. The core idea is simple — prefix matching — but scaling it requires careful engineering.

By combining:

* Tries for structure
* Pre‑computation for speed
* Caching for hot paths
* Batch updates for scalability

we can build an autocomplete system that scales from a small application to a global search engine.

---


