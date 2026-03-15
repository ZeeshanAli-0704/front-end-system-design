# Pagination Patterns — Complete Frontend System Design Guide

> Offset-based, Cursor-based, Keyset, Page-number, Infinite Scroll, Virtual Scroll, Load More — frontend & backend implementation, trade-offs, and interview questions.

---

<a id="top"></a>

## Table of Contents

- [Why Pagination Matters](#why-pagination-matters)
- [Pagination Strategies Overview](#pagination-strategies-overview)
- [Offset Based Pagination](#offset-based-pagination)
- [Cursor Based Pagination](#cursor-based-pagination)
- [Keyset (Seek) Pagination](#keyset-seek-pagination)
- [Page Number Pagination](#page-number-pagination)
- [Frontend Patterns](#frontend-patterns)
  - Infinite Scroll
  - Load More Button
  - Virtual Windowed Scroll
  - Traditional Page Numbers
- [Comparison Table](#comparison-table)
- [Real World Architecture Examples](#real-world-architecture-examples)
- [Edge Cases and Gotchas](#edge-cases-and-gotchas)
- [Interview Questions and Answers](#interview-questions-and-answers)
[⬆ Back to Top](#top)

---

## Why Pagination Matters

```
Without Pagination:
┌────────┐   GET /posts    ┌────────┐   SELECT * FROM posts   ┌────┐
│ Client │───────────────► │ Server │────────────────────────►│ DB │
│        │◄─────────────── │        │◄────────────────────────│    │
└────────┘  1M records     └────────┘   Full table scan       └────┘
            (💀 OOM)                    (💀 slow query)
```

**Problems without pagination:**
- **Memory** — Browser crashes rendering 100K+ DOM nodes
- **Bandwidth** — Sending megabytes of JSON over the wire
- **Database** — Full table scans, no index usage, locks
- **UX** — User waits 10+ seconds staring at a spinner
- **SEO** — Search engines can't crawl infinite content

**Pagination solves this** by fetching data in small, manageable **pages** (or chunks).

[⬆ Back to Top](#top)

---

## Pagination Strategies Overview

```
                     Pagination Strategies
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
      Offset-Based    Cursor-Based     Keyset (Seek)
      (LIMIT/OFFSET)  (opaque token)   (WHERE + ORDER)
            │               │               │
            ▼               ▼               ▼
     ┌──────────┐    ┌──────────┐    ┌──────────┐
     │ Page 1   │    │ after:   │    │ WHERE    │
     │ Page 2   │    │  abc123  │    │ id > 100 │
     │ Page 3   │    │          │    │ LIMIT 20 │
     └──────────┘    └──────────┘    └──────────┘

                  Frontend Patterns
                        │
         ┌──────────────┼──────────────┐──────────────┐
         ▼              ▼              ▼              ▼
   Page Numbers    Infinite      Load More       Virtual
   [1][2][3]►      Scroll        [ Button ]      Scroll
                   (auto)        (manual)        (windowed)
```

> **Key Distinction:** Offset / Cursor / Keyset = **how you ask the backend for data**. Infinite Scroll / Load More / Page Numbers = **how you present it on the frontend**.

[⬆ Back to Top](#top)

---

## Offset Based Pagination

### 3.1 How It Works

The client specifies `offset` (skip N rows) and `limit` (take N rows).

```
Page 1: offset=0,  limit=20  →  rows 1–20
Page 2: offset=20, limit=20  →  rows 21–40
Page 3: offset=40, limit=20  →  rows 41–60
```

### 3.2 Backend Implementation

```sql
-- SQL Query
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;  -- Page 3 (0-indexed)
```

```js
// ─── Express API ───
app.get('/api/posts', async (req, res) => {
  const page = Math.max(1, parseInt(req.query.page) || 1);
  const limit = Math.min(100, parseInt(req.query.limit) || 20); // cap at 100
  const offset = (page - 1) * limit;

  const [posts, totalCount] = await Promise.all([
    db.query('SELECT * FROM posts ORDER BY created_at DESC LIMIT $1 OFFSET $2', [limit, offset]),
    db.query('SELECT COUNT(*) FROM posts')
  ]);

  const totalPages = Math.ceil(totalCount.rows[0].count / limit);

  res.json({
    data: posts.rows,
    pagination: {
      page,
      limit,
      totalCount: parseInt(totalCount.rows[0].count),
      totalPages,
      hasNextPage: page < totalPages,
      hasPrevPage: page > 1
    }
  });
});
```

**API Response:**
```json
{
  "data": [
    { "id": 41, "title": "Post 41", "created_at": "2026-03-10" },
    { "id": 42, "title": "Post 42", "created_at": "2026-03-09" }
  ],
  "pagination": {
    "page": 3,
    "limit": 20,
    "totalCount": 1500,
    "totalPages": 75,
    "hasNextPage": true,
    "hasPrevPage": true
  }
}
```

### 3.3 Frontend Implementation

```js
// ─── React: Offset Pagination with TanStack Query ───
import { useQuery, keepPreviousData } from '@tanstack/react-query';

function usePaginatedPosts(page, limit = 20) {
  return useQuery({
    queryKey: ['posts', page, limit],
    queryFn: () =>
      fetch(`/api/posts?page=${page}&limit=${limit}`).then(r => r.json()),
    placeholderData: keepPreviousData, // keep old data while fetching new page
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
}

function PostList() {
  const [page, setPage] = useState(1);
  const { data, isLoading, isPlaceholderData } = usePaginatedPosts(page);

  if (isLoading) return <Skeleton count={20} />;

  return (
    <div>
      {data.data.map(post => (
        <PostCard key={post.id} post={post} />
      ))}

      <div className="pagination-controls">
        <button
          onClick={() => setPage(p => Math.max(1, p - 1))}
          disabled={!data.pagination.hasPrevPage}
        >
          Previous
        </button>

        <span>
          Page {data.pagination.page} of {data.pagination.totalPages}
        </span>

        <button
          onClick={() => setPage(p => p + 1)}
          disabled={!data.pagination.hasNextPage || isPlaceholderData}
        >
          Next
        </button>
      </div>
    </div>
  );
}
```

### 3.4 The Offset Problem — Skipping & Duplication

```
Initial state: [A, B, C, D, E, F, G, H, I, J]  (page size = 3)

Page 1 (offset=0): [A, B, C] ✅

    ↓ User deletes item B while on Page 1

Database now:      [A, C, D, E, F, G, H, I, J]

Page 2 (offset=3): [E, F, G]  ❌ Item D got SKIPPED!

                    A  C  D  E  F  G  H  I  J
                    └──┘  └─ skipped!
                    offset=0..2    offset=3..5
```

```
Initial state: [A, B, C, D, E, F, G, H, I, J]  (page size = 3)

Page 1 (offset=0): [A, B, C] ✅

    ↓ New item Z inserted at the top

Database now:      [Z, A, B, C, D, E, F, G, H, I, J]

Page 2 (offset=3): [C, D, E]  ❌ Item C appears AGAIN (duplicate)!
```

**This is the fundamental flaw of offset-based pagination in real-time datasets.**

### 3.5 Performance Problem

```sql
-- Page 1: Fast ✅
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 0;
-- DB reads 20 rows

-- Page 500: SLOW ❌
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 9980;
-- DB reads and discards 9980 rows, then returns 20
-- Gets worse linearly: O(offset + limit)
```

### 3.6 Pros & Cons

| Pros | Cons |
|---|---|
| Simple to implement | Slow for deep pages (`OFFSET 100000`) |
| Easy to jump to any page | Items can be skipped or duplicated on writes |
| `totalCount` enables "Page X of Y" | `COUNT(*)` is expensive on large tables |
| Stateless | Not suitable for real-time/frequently changing data |
| Works with any database | |

### 3.7 When to Use Offset

- **Admin panels** — Paginated tables with "Go to page" feature
- **Search results** — Google-style page numbers (data is relatively static per query)
- **Small datasets** — Under 100K rows where OFFSET performance is acceptable
- **When users need random access** — "Jump to page 50"

[⬆ Back to Top](#top)

---

## Cursor Based Pagination

### 4.1 How It Works

Instead of saying "skip N rows," the client sends a **cursor** — an opaque string that points to a specific item. The server returns items after (or before) that cursor.

```
Request 1:  GET /posts?first=20
Response:   { edges: [...20 items], pageInfo: { endCursor: "abc123", hasNextPage: true } }

Request 2:  GET /posts?first=20&after=abc123
Response:   { edges: [...20 items], pageInfo: { endCursor: "def456", hasNextPage: true } }

Request 3:  GET /posts?first=20&after=def456
Response:   { edges: [...15 items], pageInfo: { endCursor: "ghi789", hasNextPage: false } }
```

The cursor is typically a **Base64-encoded** value (e.g., the row's ID or timestamp) that the client treats as opaque.

### 4.2 Backend Implementation

```js
// ─── Express: Cursor-Based Pagination ───
app.get('/api/posts', async (req, res) => {
  const limit = Math.min(100, parseInt(req.query.first) || 20);
  const afterCursor = req.query.after; // opaque cursor string

  let query = 'SELECT * FROM posts';
  const params = [];

  if (afterCursor) {
    // Decode cursor → { id, created_at }
    const decoded = JSON.parse(Buffer.from(afterCursor, 'base64').toString());
    query += ` WHERE (created_at, id) < ($1, $2)`;
    params.push(decoded.created_at, decoded.id);
  }

  query += ` ORDER BY created_at DESC, id DESC LIMIT $${params.length + 1}`;
  params.push(limit + 1); // fetch 1 extra to check hasNextPage

  const result = await db.query(query, params);
  const hasNextPage = result.rows.length > limit;
  const edges = result.rows.slice(0, limit); // remove the extra

  res.json({
    edges: edges.map(node => ({
      node,
      cursor: Buffer.from(JSON.stringify({
        id: node.id,
        created_at: node.created_at
      })).toString('base64')
    })),
    pageInfo: {
      hasNextPage,
      hasPreviousPage: !!afterCursor,
      startCursor: edges[0]
        ? Buffer.from(JSON.stringify({ id: edges[0].id, created_at: edges[0].created_at })).toString('base64')
        : null,
      endCursor: edges.length > 0
        ? Buffer.from(JSON.stringify({ id: edges[edges.length - 1].id, created_at: edges[edges.length - 1].created_at })).toString('base64')
        : null
    }
  });
});
```

**API Response (Relay-style):**
```json
{
  "edges": [
    {
      "node": { "id": 1042, "title": "Post A", "created_at": "2026-03-10T10:00:00Z" },
      "cursor": "eyJpZCI6MTA0MiwiY3JlYXRlZF9hdCI6IjIwMjYtMDMtMTBUMTA6MDA6MDBaIn0="
    },
    {
      "node": { "id": 1041, "title": "Post B", "created_at": "2026-03-10T09:30:00Z" },
      "cursor": "eyJpZCI6MTA0MSwiY3JlYXRlZF9hdCI6IjIwMjYtMDMtMTBUMDk6MzA6MDBaIn0="
    }
  ],
  "pageInfo": {
    "hasNextPage": true,
    "hasPreviousPage": true,
    "startCursor": "eyJpZCI6MTA0MiwiY3JlYXRlZF9hdCI6IjIwMjYtMDMtMTBUMTA6MDA6MDBaIn0=",
    "endCursor": "eyJpZCI6MTA0MSwiY3JlYXRlZF9hdCI6IjIwMjYtMDMtMTBUMDk6MzA6MDBaIn0="
  }
}
```

### 4.3 GraphQL Relay Connection Spec

The most widely adopted cursor pagination standard:

```graphql
# Schema
type Query {
  posts(first: Int, after: String, last: Int, before: String): PostConnection!
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int
}

type PostEdge {
  node: Post!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type Post {
  id: ID!
  title: String!
  createdAt: DateTime!
}
```

```graphql
# Query
query GetPosts($after: String) {
  posts(first: 20, after: $after) {
    edges {
      node {
        id
        title
        createdAt
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

### 4.4 Frontend Implementation

```js
// ─── React: Cursor Pagination with TanStack Query (Infinite) ───
import { useInfiniteQuery } from '@tanstack/react-query';

function useInfinitePosts() {
  return useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam }) =>
      fetch(`/api/posts?first=20${pageParam ? `&after=${pageParam}` : ''}`)
        .then(r => r.json()),
    initialPageParam: null,
    getNextPageParam: (lastPage) =>
      lastPage.pageInfo.hasNextPage ? lastPage.pageInfo.endCursor : undefined,
  });
}

function PostFeed() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
  } = useInfinitePosts();

  if (isLoading) return <Skeleton />;

  const allPosts = data.pages.flatMap(page => page.edges.map(e => e.node));

  return (
    <div>
      {allPosts.map(post => (
        <PostCard key={post.id} post={post} />
      ))}

      {hasNextPage && (
        <button
          onClick={() => fetchNextPage()}
          disabled={isFetchingNextPage}
        >
          {isFetchingNextPage ? 'Loading...' : 'Load More'}
        </button>
      )}
    </div>
  );
}
```

### 4.5 Why Cursors Fix the Offset Problem

```
Initial state: [A, B, C, D, E, F, G, H, I, J]  (page size = 3)

Page 1 (no cursor): [A, B, C] → cursor points to C

    ↓ User deletes item B

Database now:      [A, C, D, E, F, G, H, I, J]

Page 2 (after C):  [D, E, F] ✅  No item skipped!
```

The cursor says "give me everything AFTER item C" — regardless of what was inserted or deleted before C.

### 4.6 Pros & Cons

| Pros | Cons |
|---|---|
| Consistent results even with inserts/deletes | Cannot jump to arbitrary page |
| High performance (uses index seek) | No "Page X of Y" without extra `COUNT(*)` |
| Works perfectly for infinite scroll | Slightly more complex to implement |
| Scales to billions of rows | Client must store cursor |
| Standard spec (Relay Connection) | Bi-directional cursoring is more complex |

### 4.7 When to Use Cursor

- **Social media feeds** — Instagram, Twitter, Facebook
- **Infinite scroll UIs** — news feeds, product catalogs
- **Chat message history** — scroll up to load older messages
- **Real-time / frequently changing data** — where offset guarantees break
- **Large datasets** — millions to billions of rows

[⬆ Back to Top](#top)

---

## Keyset (Seek) Pagination

### 5.1 How It Works

Keyset pagination is the **database-level technique** behind cursor-based pagination. Instead of an opaque cursor, the client sends the actual column values to seek from.

```sql
-- Instead of OFFSET:
SELECT * FROM posts
WHERE (created_at, id) < ('2026-03-10 09:30:00', 1041)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

This is essentially cursor-based pagination but with **transparent** (non-opaque) parameters.

### 5.2 Backend Implementation

```js
// ─── Express: Keyset Pagination ───
app.get('/api/posts', async (req, res) => {
  const limit = Math.min(100, parseInt(req.query.limit) || 20);
  const lastCreatedAt = req.query.last_created_at;
  const lastId = req.query.last_id;

  let query, params;

  if (lastCreatedAt && lastId) {
    // Keyset condition: fetch rows "after" the last seen row
    query = `
      SELECT * FROM posts
      WHERE (created_at, id) < ($1, $2)
      ORDER BY created_at DESC, id DESC
      LIMIT $3
    `;
    params = [lastCreatedAt, lastId, limit + 1];
  } else {
    query = `
      SELECT * FROM posts
      ORDER BY created_at DESC, id DESC
      LIMIT $1
    `;
    params = [limit + 1];
  }

  const result = await db.query(query, params);
  const hasMore = result.rows.length > limit;
  const posts = result.rows.slice(0, limit);
  const lastPost = posts[posts.length - 1];

  res.json({
    data: posts,
    pagination: {
      hasMore,
      nextParams: hasMore
        ? { last_created_at: lastPost.created_at, last_id: lastPost.id }
        : null
    }
  });
});
```

**API Response:**
```json
{
  "data": [
    { "id": 1042, "title": "Post A", "created_at": "2026-03-10T10:00:00Z" },
    { "id": 1041, "title": "Post B", "created_at": "2026-03-10T09:30:00Z" }
  ],
  "pagination": {
    "hasMore": true,
    "nextParams": {
      "last_created_at": "2026-03-10T09:30:00Z",
      "last_id": 1041
    }
  }
}
```

### 5.3 Keyset vs Cursor vs Offset — The DB Perspective

```sql
-- OFFSET: Scans and DISCARDS rows → O(offset + limit)
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
-- ⚠️ DB reads 10,020 rows, returns 20

-- KEYSET: Seeks directly via index → O(limit)
SELECT * FROM posts
WHERE (created_at, id) < ('2026-01-01', 5000)
ORDER BY created_at DESC, id DESC LIMIT 20;
-- ✅ DB reads exactly 20 rows (index seek)
```

**Requires a composite index:**
```sql
CREATE INDEX idx_posts_created_id ON posts (created_at DESC, id DESC);
```

### 5.4 Pros & Cons

| Pros | Cons |
|---|---|
| Fastest for deep pagination (index seek) | Exposes sort columns to client |
| Consistent results | Cannot jump to arbitrary page |
| Simple to understand (no encoding) | Need unique sort key (tie-breaking column) |
| No opaque cursor overhead | Multi-column sort is complex |

[⬆ Back to Top](#top)

---

## Page Number Pagination

### 6.1 How It Works

A simplified version of offset-based — the client sends a `page` number, and the server computes the offset internally.

```
GET /api/posts?page=3&per_page=20
→ Server computes: offset = (3-1) * 20 = 40
→ SQL: LIMIT 20 OFFSET 40
```

### 6.2 Backend Implementation

```js
// ─── Express: Page-Number Pagination ───
app.get('/api/posts', async (req, res) => {
  const page = Math.max(1, parseInt(req.query.page) || 1);
  const perPage = Math.min(100, parseInt(req.query.per_page) || 20);
  const offset = (page - 1) * perPage;

  const [posts, countResult] = await Promise.all([
    db.query(
      'SELECT * FROM posts ORDER BY created_at DESC LIMIT $1 OFFSET $2',
      [perPage, offset]
    ),
    db.query('SELECT COUNT(*) as total FROM posts')
  ]);

  const total = parseInt(countResult.rows[0].total);
  const totalPages = Math.ceil(total / perPage);

  res.json({
    data: posts.rows,
    meta: {
      currentPage: page,
      perPage,
      total,
      totalPages,
      hasNextPage: page < totalPages,
      hasPrevPage: page > 1,
      // Generate page links (HATEOAS-style)
      links: {
        first: `/api/posts?page=1&per_page=${perPage}`,
        last: `/api/posts?page=${totalPages}&per_page=${perPage}`,
        prev: page > 1 ? `/api/posts?page=${page - 1}&per_page=${perPage}` : null,
        next: page < totalPages ? `/api/posts?page=${page + 1}&per_page=${perPage}` : null
      }
    }
  });
});
```

### 6.3 Frontend: Page Number Navigation Component

```jsx
// ─── React: Page Number Pagination ───
function Pagination({ currentPage, totalPages, onPageChange }) {
  // Generate visible page numbers with ellipsis
  const getPageNumbers = () => {
    const delta = 2; // pages to show around current
    const pages = [];

    for (
      let i = Math.max(2, currentPage - delta);
      i <= Math.min(totalPages - 1, currentPage + delta);
      i++
    ) {
      pages.push(i);
    }

    // Add first page
    if (pages[0] > 2) pages.unshift('...');
    pages.unshift(1);

    // Add last page
    if (pages[pages.length - 1] < totalPages - 1) pages.push('...');
    if (totalPages > 1) pages.push(totalPages);

    return pages;
  };

  return (
    <nav aria-label="Pagination">
      <button
        onClick={() => onPageChange(currentPage - 1)}
        disabled={currentPage === 1}
        aria-label="Previous page"
      >
        ← Previous
      </button>

      {getPageNumbers().map((page, idx) =>
        page === '...' ? (
          <span key={`ellipsis-${idx}`} className="ellipsis">…</span>
        ) : (
          <button
            key={page}
            onClick={() => onPageChange(page)}
            className={page === currentPage ? 'active' : ''}
            aria-current={page === currentPage ? 'page' : undefined}
            aria-label={`Page ${page}`}
          >
            {page}
          </button>
        )
      )}

      <button
        onClick={() => onPageChange(currentPage + 1)}
        disabled={currentPage === totalPages}
        aria-label="Next page"
      >
        Next →
      </button>
    </nav>
  );
}

// Renders: ← Previous [1] ... [4] [5] [6] ... [50] Next →
```

```jsx
// ─── URL-Synced Pagination (Next.js / React Router) ───
import { useSearchParams } from 'react-router-dom';

function PostListPage() {
  const [searchParams, setSearchParams] = useSearchParams();
  const page = parseInt(searchParams.get('page') || '1');

  const { data, isLoading } = useQuery({
    queryKey: ['posts', page],
    queryFn: () => fetch(`/api/posts?page=${page}`).then(r => r.json()),
  });

  const handlePageChange = (newPage) => {
    setSearchParams({ page: newPage.toString() });
    window.scrollTo({ top: 0, behavior: 'smooth' });
  };

  return (
    <div>
      <PostList posts={data?.data} loading={isLoading} />
      <Pagination
        currentPage={page}
        totalPages={data?.meta.totalPages}
        onPageChange={handlePageChange}
      />
    </div>
  );
}
```

### 6.4 When to Use

- **E-commerce product listings** — users expect "Page 3 of 50"
- **Admin tables** — sortable, filterable data grids
- **Search results** — Google-style page numbers
- **Documentation / article listings** — SEO-friendly URLs (`?page=5`)

[⬆ Back to Top](#top)

---

## Frontend Patterns

### 7.1 Infinite Scroll

Automatically loads the next page when the user scrolls near the bottom.

```
┌─────────────────────┐
│  Post 1             │
│  Post 2             │  ← Visible viewport
│  Post 3             │
│  Post 4             │
├─────────────────────┤ ← Intersection trigger (sentinel)
│  ░░░ Loading... ░░░ │
└─────────────────────┘
         │
         ▼ fetches next page
┌─────────────────────┐
│  Post 1             │
│  Post 2             │
│  Post 3             │
│  Post 4             │
│  Post 5  (new)      │  ← Appended
│  Post 6  (new)      │
│  Post 7  (new)      │
├─────────────────────┤ ← New sentinel position
│                     │
└─────────────────────┘
```

#### Implementation with Intersection Observer

```jsx
// ─── React: Infinite Scroll with Intersection Observer ───
function useIntersectionObserver(onIntersect, options = {}) {
  const ref = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) onIntersect();
      },
      { rootMargin: '200px', threshold: 0, ...options }
    );

    const el = ref.current;
    if (el) observer.observe(el);

    return () => {
      if (el) observer.unobserve(el);
    };
  }, [onIntersect, options]);

  return ref;
}

function InfinitePostFeed() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
  } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam }) =>
      fetch(`/api/posts?first=20${pageParam ? `&after=${pageParam}` : ''}`)
        .then(r => r.json()),
    initialPageParam: null,
    getNextPageParam: (lastPage) =>
      lastPage.pageInfo.hasNextPage ? lastPage.pageInfo.endCursor : undefined,
  });

  // Sentinel ref — triggers fetch when scrolled into view
  const sentinelRef = useIntersectionObserver(
    useCallback(() => {
      if (hasNextPage && !isFetchingNextPage) {
        fetchNextPage();
      }
    }, [hasNextPage, isFetchingNextPage, fetchNextPage])
  );

  if (isLoading) return <PostSkeleton count={6} />;

  const allPosts = data.pages.flatMap(p => p.edges.map(e => e.node));

  return (
    <div role="feed" aria-busy={isFetchingNextPage}>
      {allPosts.map((post, index) => (
        <article key={post.id} aria-setsize={-1} aria-posinset={index + 1}>
          <PostCard post={post} />
        </article>
      ))}

      {/* Sentinel element — loads next page when visible */}
      <div ref={sentinelRef} style={{ height: 1 }} />

      {isFetchingNextPage && <Spinner />}

      {!hasNextPage && <p>You've reached the end!</p>}
    </div>
  );
}
```

#### Scroll Position Restoration

```jsx
// ─── Restore scroll position when user navigates back ───
function useScrollRestoration(key) {
  useEffect(() => {
    // Restore
    const savedPos = sessionStorage.getItem(`scroll-${key}`);
    if (savedPos) {
      window.scrollTo(0, parseInt(savedPos));
    }

    // Save on unmount
    return () => {
      sessionStorage.setItem(`scroll-${key}`, window.scrollY.toString());
    };
  }, [key]);
}
```

#### Pros & Cons of Infinite Scroll

| Pros | Cons |
|---|---|
| Seamless UX (no clicks) | Hard to reach footer content |
| Great for content feeds | Memory grows unbounded (DOM nodes) |
| Mobile-friendly (natural gesture) | Poor accessibility (no page landmarks) |
| Higher engagement metrics | Can't bookmark a specific "page" |
| | No "Page X of Y" information |
| | Back/forward navigation loses position |

---

### 7.2 Load More Button

Manual variant of infinite scroll — user clicks a button to load more.

```jsx
function PostListWithLoadMore() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfinitePosts();

  const allPosts = data?.pages.flatMap(p => p.edges.map(e => e.node)) || [];

  return (
    <div>
      {allPosts.map(post => <PostCard key={post.id} post={post} />)}

      {hasNextPage && (
        <button
          onClick={() => fetchNextPage()}
          disabled={isFetchingNextPage}
          className="load-more-btn"
        >
          {isFetchingNextPage ? (
            <Spinner size="sm" />
          ) : (
            `Load More (${allPosts.length} of ${data.pages[0].totalCount || '?'})`
          )}
        </button>
      )}
    </div>
  );
}
```

#### When to Prefer Load More over Infinite Scroll

- User needs to **reach the footer** (contact info, links)
- Content is **valuable** — users want to decide when to load more (not forced)
- **Accessibility** — Screen readers handle a button better than auto-loading
- **Monetization** — Place ads between "Load More" batches

---

### 7.3 Virtual / Windowed Scroll

Renders only the items **visible in the viewport** (+ a small buffer). Perfect for very long lists.

```
Full List (10,000 items):
  [Item 0   ] ─── NOT rendered (above viewport)
  [Item 1   ]
  ...
  [Item 98  ]
  ─────────── ← Viewport top
  [Item 99  ] ─── RENDERED
  [Item 100 ] ─── RENDERED
  [Item 101 ] ─── RENDERED
  [Item 102 ] ─── RENDERED
  [Item 103 ] ─── RENDERED
  ─────────── ← Viewport bottom
  [Item 104 ]
  ...
  [Item 9999] ─── NOT rendered (below viewport)
```

Only 5-15 DOM nodes exist at any time, regardless of list size.

```jsx
// ─── React: Virtual Scroll with @tanstack/react-virtual ───
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualPostList({ posts }) {
  const parentRef = useRef(null);

  const virtualizer = useVirtualizer({
    count: posts.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 120, // estimated row height in px
    overscan: 5, // extra items above/below viewport
  });

  return (
    <div
      ref={parentRef}
      style={{ height: '100vh', overflow: 'auto' }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            <PostCard post={posts[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

```jsx
// ─── Infinite Scroll + Virtual Scroll (Best combo for huge lists) ───
function VirtualInfiniteList() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfinitePosts();
  const allPosts = data?.pages.flatMap(p => p.edges.map(e => e.node)) || [];
  const parentRef = useRef(null);

  const virtualizer = useVirtualizer({
    count: hasNextPage ? allPosts.length + 1 : allPosts.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 120,
    overscan: 5,
  });

  // Trigger next page fetch when scrolling near the end
  useEffect(() => {
    const lastItem = virtualizer.getVirtualItems().at(-1);
    if (!lastItem) return;

    if (
      lastItem.index >= allPosts.length - 1 &&
      hasNextPage &&
      !isFetchingNextPage
    ) {
      fetchNextPage();
    }
  }, [virtualizer.getVirtualItems(), hasNextPage, isFetchingNextPage, allPosts.length]);

  return (
    <div ref={parentRef} style={{ height: '100vh', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map(virtualRow => {
          const isLoaderRow = virtualRow.index >= allPosts.length;

          return (
            <div
              key={virtualRow.key}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                transform: `translateY(${virtualRow.start}px)`,
              }}
            >
              {isLoaderRow ? <Spinner /> : <PostCard post={allPosts[virtualRow.index]} />}
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

#### When to Use Virtual Scroll

- Very long lists (1K+ items already loaded)
- Tables with thousands of rows (admin dashboards)
- Combined with infinite scroll for **memory-efficient** infinite lists
- Chat message history (Discord, Slack)

---

### 7.4 Frontend Pattern Comparison

| Pattern | Trigger | UX | DOM Nodes | SEO | A11y |
|---|---|---|---|---|---|
| Page Numbers | Click page | Jump between pages | Small (1 page) | ✅ `/page/3` | ✅ Best |
| Infinite Scroll | Auto (scroll) | Seamless browsing | Grows unbounded | ❌ | ⚠️ Needs ARIA |
| Load More | Click button | User-controlled | Grows unbounded | ⚠️ | ✅ |
| Virtual Scroll | Scroll | Smooth, fast | Fixed (~10-20) | ❌ | ⚠️ Needs ARIA |
| Virtual + Infinite | Auto (scroll) | Best of both | Fixed (~10-20) | ❌ | ⚠️ Needs ARIA |

[⬆ Back to Top](#top)

---

## Comparison Table

### Backend Strategies

| Feature | Offset-Based | Cursor-Based | Keyset (Seek) | Page-Number |
|---|---|---|---|---|
| **API Params** | `offset`, `limit` | `after`, `first` | `last_id`, `limit` | `page`, `per_page` |
| **SQL Technique** | `LIMIT/OFFSET` | `WHERE + LIMIT` | `WHERE + LIMIT` | `LIMIT/OFFSET` |
| **Deep Page Performance** | ❌ O(offset+limit) | ✅ O(limit) | ✅ O(limit) | ❌ O(offset+limit) |
| **Data Consistency** | ❌ Skip/duplicate | ✅ Stable | ✅ Stable | ❌ Skip/duplicate |
| **Random Page Access** | ✅ Jump to page N | ❌ Sequential only | ❌ Sequential only | ✅ Jump to page N |
| **Total Count** | Optional (expensive) | Optional (expensive) | Optional (expensive) | Required |
| **Implementation** | Simple | Moderate | Moderate | Simple |
| **Sorting Flexibility** | ✅ Any column | ⚠️ Needs indexed sort | ⚠️ Needs indexed sort | ✅ Any column |
| **Best For** | Admin panels, search | Feeds, mobile apps | High-volume APIs | E-commerce, blogs |

### At a Glance

```
                     Random       Deep Page      Real-Time
                     Access?      Performance?   Consistency?
                     
Offset/Page-Number:   ✅            ❌              ❌
Cursor-Based:         ❌            ✅              ✅
Keyset:               ❌            ✅              ✅
```

[⬆ Back to Top](#top)

---

## Real World Architecture Examples

### 9.1 Instagram Feed (Cursor + Infinite Scroll)

```
┌─────────────┐  GET /feed?after=cursor  ┌──────────────┐   WHERE (score, id) < (x, y)
│  Mobile App  │──────────────────────────►│  Feed Service │──────────────────────────────►│ DB │
│  (React      │◄──────────────────────────│  (generates  │◄──────────────────────────────│    │
│   Native)    │  { edges, pageInfo }      │   cursors)   │   LIMIT 10 (keyset seek)     └────┘
└─────────────┘                           └──────────────┘
```

- **Backend:** Cursor-based (opaque cursor encoding `score + post_id`)
- **Frontend:** Infinite scroll with Intersection Observer
- **Why:** Feed is real-time (new posts constantly), deep pagination must be O(1), users never need "page 50"

### 9.2 Amazon Product Search (Offset + Page Numbers)

```
┌──────────┐  GET /search?q=laptop&page=3  ┌───────────────┐
│  Browser  │───────────────────────────────►│  Search API    │──►│ Elasticsearch │
│           │◄───────────────────────────────│  (from: 40,   │◄──│ (from/size)   │
└──────────┘  { results, total, page }      │   size: 20)   │   └───────────────┘
                                            └───────────────┘
```

- **Backend:** Offset-based (Elasticsearch `from`/`size` internally)
- **Frontend:** Page number navigation ("Page 3 of 150")
- **Why:** Users want to jump to specific pages; product catalog changes infrequently relative to reads; Elasticsearch limits `from` to 10,000 max

### 9.3 Slack Messages (Keyset + Virtual Scroll)

```
┌──────────┐  GET /messages?before=ts_123&limit=50  ┌───────────────┐
│  Desktop  │───────────────────────────────────────►│  Message API   │
│  App      │◄───────────────────────────────────────│  (keyset on    │
│           │  { messages, has_more }                 │   timestamp)   │
└──────────┘                                        └───────────────┘

Frontend: Virtual scroll (only renders visible messages)
- Scroll up → loads older messages (fetch before=oldest_ts)
- Scroll down → loads newer messages (fetch after=newest_ts)
- Anchored scroll position on prepend
```

### 9.4 GitHub Issues (Hybrid: Cursor + Page Numbers)

GitHub's API supports **both**:

```
REST API (offset-based):
GET /repos/facebook/react/issues?page=3&per_page=30

GraphQL API (cursor-based):
query {
  repository(owner: "facebook", name: "react") {
    issues(first: 30, after: "Y3Vyc29yOnYyOpHOBj3...") {
      edges { node { title } cursor }
      pageInfo { hasNextPage endCursor }
    }
  }
}
```

[⬆ Back to Top](#top)

---

## Edge Cases and Gotchas

### 10.1 The COUNT(*) Problem

```sql
-- On a table with 50M rows:
SELECT COUNT(*) FROM posts;  -- ⏱️ 2-5 seconds on PostgreSQL!
```

**Solutions:**
1. **Approximate count:** `SELECT reltuples FROM pg_class WHERE relname = 'posts';`
2. **Cached count:** Store total in a separate counter table/Redis, update on insert/delete
3. **Don't show total at all:** Just show "Next" / "Previous" (Twitter, Instagram approach)
4. **Cap it:** "About 10,000 results" (Google approach after page ~40)

### 10.2 Cursor Stability with Changing Sort Orders

```
Problem: User sorts by "popularity" — scores change constantly.
Post A (score: 100) is on page 1.
User fetches page 2 with cursor pointing after Post A.
Post A's score drops to 5 — it would now be on page 50.
But cursor already passed it → NO DUPLICATE. ✅

New problem: Post Z's score jumps from 1 to 999.
It should now be on page 1, but cursor is past it → MISSED. ❌
```

**Solution:** For volatile sort orders, use offset-based pagination or accept eventual consistency.

### 10.3 Infinite Scroll Memory Management

```jsx
// Problem: User scrolls through 5000 posts → DOM has 5000 nodes → 💀
// Solution 1: Virtual scroll (renders only ~20 nodes)
// Solution 2: Unload old pages:

const MAX_PAGES = 5; // only keep 5 pages in memory

const { data, fetchNextPage } = useInfiniteQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  // Remove old pages to limit memory
  maxPages: MAX_PAGES,
  getNextPageParam: (lastPage) => lastPage.pageInfo.endCursor,
});
```

### 10.4 SEO and Pagination

```html
<!-- For page-number pagination, use rel="next"/"prev" and canonical -->
<link rel="canonical" href="https://example.com/posts?page=3" />
<link rel="prev" href="https://example.com/posts?page=2" />
<link rel="next" href="https://example.com/posts?page=4" />
```

```jsx
// Next.js example
import Head from 'next/head';

function PostsPage({ page, totalPages }) {
  return (
    <Head>
      <link rel="canonical" href={`https://example.com/posts?page=${page}`} />
      {page > 1 && <link rel="prev" href={`https://example.com/posts?page=${page - 1}`} />}
      {page < totalPages && <link rel="next" href={`https://example.com/posts?page=${page + 1}`} />}
    </Head>
  );
}
```

**Infinite scroll is not SEO-friendly** — search engines can't trigger scroll events. Use SSR with page-number URLs as a fallback.

### 10.5 Race Conditions with Fast Scrolling

```jsx
// Problem: User scrolls fast → multiple concurrent fetches → results arrive out of order

// Solution: AbortController
function usePaginatedPosts(page) {
  return useQuery({
    queryKey: ['posts', page],
    queryFn: ({ signal }) =>
      fetch(`/api/posts?page=${page}`, { signal }).then(r => r.json()),
    // TanStack Query automatically aborts previous request
    // when queryKey changes
  });
}
```

[⬆ Back to Top](#top)

---

## Interview Questions and Answers

### Q1: What's the difference between offset-based and cursor-based pagination?

**Answer:**

| Aspect | Offset | Cursor |
|---|---|---|
| Mechanism | Skip N rows (`OFFSET 40`) | Seek after a specific row (`WHERE id < cursor`) |
| Deep page perf | ❌ O(offset + limit) | ✅ O(limit) |
| Data consistency | ❌ Items can be skipped/duplicated if data changes | ✅ Stable — always continues from last seen item |
| Random access | ✅ Can jump to any page | ❌ Must traverse sequentially |
| Best for | Admin panels, search results | Social feeds, infinite scroll |

**Key insight:** Offset = "give me rows 40–60." Cursor = "give me 20 rows after THIS item." Cursor is immune to inserts/deletes shifting positions.

---

### Q2: How would you implement infinite scroll?

**Answer:**

1. **API:** Use cursor-based pagination (`GET /posts?after=cursor&first=20`).
2. **Trigger:** Use `IntersectionObserver` on a sentinel element near the bottom.
3. **State Management:** TanStack Query's `useInfiniteQuery` manages page accumulation.
4. **Memory:** Use virtual scrolling (`@tanstack/react-virtual`) for large lists to keep DOM node count constant.
5. **Loading UX:** Show skeleton placeholders while fetching.
6. **End state:** When `hasNextPage` is false, show "You've reached the end."
7. **Error handling:** Show retry button on failure, don't remove existing content.
8. **Scroll restoration:** Save position in `sessionStorage` for back/forward navigation.

---

### Q3: When would you choose page numbers over infinite scroll?

**Answer:**

| Choose Page Numbers When... | Choose Infinite Scroll When... |
|---|---|
| User needs to reach the **footer** | Content is exploratory (feeds, social) |
| **SEO** is important (crawlable URLs) | Mobile app (natural scroll gesture) |
| User wants to **bookmark** a specific page | Maximizing **engagement** metrics |
| Content has a **clear total** ("142 results") | Content is **time-ordered** (newest first) |
| **Admin/data tables** with sorting + filtering | No need to jump to specific position |
| E-commerce product listings | |

---

### Q4: How does cursor-based pagination work under the hood?

**Answer:**

1. **Cursor is an opaque string** (usually Base64-encoded) containing sort key values.
2. Decoding `eyJpZCI6MTAwfQ==` → `{ "id": 100 }`
3. Server uses this to build a `WHERE` clause:

```sql
-- Cursor decoded to id=100
SELECT * FROM posts WHERE id < 100 ORDER BY id DESC LIMIT 20;
```

4. Server encodes the last item's values as the new `endCursor`.
5. This is a **keyset seek** — it uses an index, so performance is O(limit), not O(offset + limit).
6. The cursor must include ALL columns used in the sort order + a unique tie-breaker:

```sql
-- Sort by created_at DESC, with id as tie-breaker
WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT 20;
```

---

### Q5: How do you handle the COUNT(*) problem in pagination?

**Answer:**

`COUNT(*)` on large tables can take seconds. Strategies:
1. **Avoid it entirely** — Use cursor-based pagination with `hasNextPage` instead of "Page X of Y"
2. **Approximate count** — PostgreSQL: `SELECT reltuples FROM pg_class`; MySQL: `SHOW TABLE STATUS`
3. **Cached count** — Maintain a counter in Redis, increment/decrement on insert/delete
4. **Materialized view** — Pre-compute counts for filtered queries
5. **Cap display** — "About 10,000+ results" (stop counting after reaching a threshold)
6. **Count in background** — Return results immediately; send total count asynchronously

---

### Q6: What's the problem with OFFSET 1000000?

**Answer:**

```sql
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 1000000;
```

The database must:
1. Perform a full sort (or index scan) on `created_at`
2. **Read and discard** 1,000,000 rows
3. Return the next 20 rows

This is **O(offset + limit)** — linear time. On a table with 50M rows, OFFSET 1M scans ~1M index entries. It gets progressively slower as the user goes deeper.

**Solutions:**
- Switch to **keyset/cursor** pagination for O(limit) performance
- **Enforce a max page depth** (Google caps at ~page 40)
- Use a **covering index** so the DB doesn't need table lookups for the offset scan

---

### Q7: How would you build pagination for a real-time chat app?

**Answer:**

**Bidirectional cursor pagination** — user can scroll up (older messages) and down (newer messages).

```
API:
GET /messages?channel=general&before=cursor_A&limit=50  (older)
GET /messages?channel=general&after=cursor_B&limit=50   (newer)
```

**Frontend approach:**
1. Load initial batch (latest 50 messages)
2. **Scroll up** → fetch older messages with `before` cursor
3. **New messages** arrive via WebSocket → append at bottom
4. **Virtual scroll** to handle thousands of messages efficiently
5. **Anchor scroll position** — when prepending older messages, maintain the user's scroll position so they don't jump

```jsx
// Anchor scroll on prepend
function usePrependScrollAnchor(containerRef, data) {
  const prevScrollHeight = useRef(0);

  useLayoutEffect(() => {
    const el = containerRef.current;
    if (prevScrollHeight.current > 0) {
      // Adjust scroll position by the difference in height
      el.scrollTop += el.scrollHeight - prevScrollHeight.current;
    }
    prevScrollHeight.current = el.scrollHeight;
  }, [data]);
}
```

---

### Q8: How do you make infinite scroll accessible?

**Answer:**

1. **ARIA `role="feed"`** — Announces the feed to screen readers
2. **`aria-busy="true"`** while fetching
3. **`aria-setsize` and `aria-posinset`** on each article (or `-1` if total unknown)
4. **Focus management** — Don't steal focus on auto-load; let screen reader users navigate naturally
5. **Announce new content** — Use a live region: `<div aria-live="polite">20 more posts loaded</div>`
6. **Provide alternative** — Offer a "View All Pages" link for traditional pagination
7. **Keyboard navigation** — Ensure items are focusable and follow logical tab order

```jsx
<div role="feed" aria-busy={isFetching} aria-label="News feed">
  {posts.map((post, i) => (
    <article
      key={post.id}
      aria-setsize={-1}
      aria-posinset={i + 1}
      tabIndex={0}
    >
      <PostCard post={post} />
    </article>
  ))}

  {/* Screen reader announcement */}
  <div aria-live="polite" className="sr-only">
    {isFetching ? 'Loading more posts...' : ''}
  </div>
</div>
```

---

### Q9: Compare REST pagination vs GraphQL pagination.

**Answer:**

**REST — Offset style:**
```
GET /api/posts?page=2&per_page=20
```
```json
{
  "data": [...],
  "meta": { "page": 2, "totalPages": 50 }
}
```

**REST — Cursor style:**
```
GET /api/posts?after=abc123&limit=20
```
```json
{
  "data": [...],
  "next_cursor": "def456",
  "has_more": true
}
```

**GraphQL — Relay Connection Spec (cursor):**
```graphql
query {
  posts(first: 20, after: "abc123") {
    edges {
      node { id title }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}
```

| Feature | REST Offset | REST Cursor | GraphQL Relay |
|---|---|---|---|
| Standardization | No standard | No standard | Relay Connection Spec |
| Client flexibility | Fixed fields | Fixed fields | Client picks fields |
| Page metadata | Custom | Custom | Standardized `pageInfo` |
| Caching | Easy (URL-based) | Hard (cursor changes) | Normalized cache (Apollo) |

---

### Q10: Explain pagination in Elasticsearch / search engines.

**Answer:**

Elasticsearch supports multiple strategies:

1. **`from` / `size` (offset-based):** Limited to 10,000 hits by default.
   ```json
   { "from": 40, "size": 20, "query": { "match": { "title": "react" } } }
   ```

2. **`search_after` (keyset):** For deep pagination.
   ```json
   { "size": 20, "search_after": [1609459200, "post-123"], "sort": [{ "date": "desc" }, { "id": "desc" }] }
   ```

3. **Scroll API:** For processing all results (batch jobs, not user-facing).
   ```json
   { "scroll": "5m", "size": 1000, "query": { "match_all": {} } }
   ```

4. **Point-in-Time (PIT):** Consistent snapshot for paginating through changing data.

**Frontend rule:** Use `from/size` for pages 1–500. Switch to `search_after` beyond that. Never expose Scroll API to end users.

---

### Bonus: Quick Decision Reference

| Scenario | Backend Strategy | Frontend Pattern |
|---|---|---|
| Social media feed | Cursor | Infinite Scroll |
| E-commerce product grid | Offset / Page-number | Page Numbers |
| Admin data table | Offset / Keyset | Page Numbers + Sorting |
| Chat messages (load older) | Cursor (bidirectional) | Virtual + Load on scroll up |
| Search results | Offset (capped) + Keyset fallback | Page Numbers |
| Activity log / audit trail | Cursor | Load More |
| Image gallery | Cursor | Infinite + Virtual Grid |
| Blog / documentation | Page-number | Page Numbers (SEO) |
| Notification list | Cursor | Load More |
| Analytics dashboard table | Offset | Page Numbers + Export |

---

> **Interview Tip:** When asked "How would you paginate X?", structure your answer as:
> 1. **Identify the data pattern** — Static or real-time? How large? User access pattern?
> 2. **Pick the backend strategy** — Offset for random access, Cursor for streams
> 3. **Pick the frontend pattern** — Page numbers, infinite scroll, load more, virtual scroll
> 4. **Address edge cases** — COUNT(*) cost, consistency, SEO, accessibility, scroll position
> 5. **Mention performance** — Index design, caching, memory management

[⬆ Back to Top](#top)

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design

[⬆ Back to Top](#top)
