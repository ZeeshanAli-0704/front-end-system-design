## 🧠 System Design: Facebook News Feed (Frontend)


- [System Design: Facebook News Feed (Frontend)](#system-design-facebook-news-feed-frontend)
  - [1. General Requirements](#1-general-requirements)
  - [2. Component Architecture](#2-component-architecture)
      - [Top Level Layout](#top-level-layout)
      - [Story Component StoryCard](#story-component-storycard)
      - [Comment Component CommentList](#comment-component-commentlist)
      - [Create Comment Component CreateComment](#create-comment-component-createcomment)
      - [Component Hierarchy](#component-hierarchy)
  - [3. Data Entities](#3-data-entities)
  - [4. Data APIs](#4-data-apis)
  - [5. Data Store (Frontend State Management)](#5-data-store-frontend-state-management)
  - [6. Infinite Scrolling (Old Feeds)](#6-infinite-scrolling-old-feeds)
  - [7. Real Time New Feeds via SSE](#7-real-time-new-feeds-via-sse)
  - [8. Complete Feed Flow](#8-complete-feed-flow)
  - [9. Optimization Strategies](#9-optimization-strategies)
      - [A. Network Performance](#a-network-performance)
      - [B. Rendering Performance](#b-rendering-performance)
      - [C. JS Performance](#c-js-performance)
  - [10. Accessibility (A11y)](#10-accessibility-a11y)
  - [11. Example Flow: Combined Infinite Scroll SSE](#11-example-flow-combined-infinite-scroll-sse)
  - [12. Endpoint Summary](#12-endpoint-summary)
  - [Final Summary](#final-summary)

---

### 1. 🎯 General Requirements

* Display **feed/stories** from people the user follows.
* User can **create posts** (text, image, video).
* Users can **Like**, **Share**, and **Comment**.
* Should support **Infinite Scrolling**:

  * Fetch **older feeds** as you scroll down.
  * Show **new feeds** in real-time (without refresh).
* Real time updates when:

  * New post is created.
  * Someone likes/comments on an existing post.

---

### 2. 🧩 Component Architecture

We’ll design a modular, maintainable component system.

#### **Top Level Layout**

* `FeedPage`

  * Manages **initial data load**, **scrolling**, and **SSE subscription**.
  * Renders a list of `StoryCard` components.

---

#### **Story Component StoryCard**

**Responsibilities:**

* Show user avatar, name, timestamp.
* Render post content (text/media).
* Include controls:

  * ❤️ Like
  * 💬 Comment
  * 🔁 Share

**Sub-components:**

* `StoryHeader` – avatar, name, timestamp
* `StoryMedia` – renders image/video
* `StoryActions` – like/share/comment buttons
* `CommentList` – render list of comments
* `CreateComment` – textbox + post button

---

#### **Comment Component CommentList**

* Displays all comments (virtualized for performance).
* Each comment has:

  * Avatar
  * Username
  * Comment text

---

#### **Create Comment Component CreateComment**

* Textarea + Post button.
* Emits comment creation event to parent (`StoryCard`).

---

#### 🧠 Component Hierarchy

```
FeedPage
 ├── StoryCard
 │    ├── StoryHeader
 │    ├── StoryMedia
 │    ├── StoryActions
 │    ├── CommentList
 │    └── CreateComment
 └── InfiniteScrollLoader (sentinel element)
```

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/36z8o2p3kb65w1pfd7jx.png)
---

### 3. 🧱 Data Entities

```ts
type Media = {
  id: string;
  type: 'image' | 'video';
  url: string;
  thumbnail?: string;
};

type Comment = {
  id: string;
  user: { id: string; name: string; avatarUrl: string };
  text: string;
  createdAt: string;
};

type Story = {
  id: string;
  author: { id: string; name: string; avatarUrl: string };
  content: string;
  media?: Media[];
  likes: number;
  shares: number;
  comments: Comment[];
  createdAt: string;
};
```

---

### 4. 🔌 Data APIs

| API                       | Method        | Description                                   |
| ------------------------- | ------------- | --------------------------------------------- |
| `/api/feed?cursor=<id>`   | **GET**       | Fetch paginated stories (for infinite scroll) |
| `/api/story`              | **POST**      | Create a new story                            |
| `/api/story/:id/like`     | **POST**      | Like/unlike story                             |
| `/api/story/:id/comment`  | **POST**      | Add comment                                   |
| `/api/story/:id/comments` | **GET**       | Fetch comments                                |
| `/api/feeds/stream`       | **GET (SSE)** | Receive new stories in real-time              |

---

### 5. 🗂️ Data Store (Frontend State Management)

Use **React Query**, **Zustand**, or **Redux Toolkit** for reactive state.

**State Structure:**

```ts
{
  feed: {
    stories: Story[];
    cursor: string; // pagination cursor for older feeds
    loading: boolean;
  },
  newStoriesBuffer: Story[]; // for stories from SSE
  comments: { [storyId: string]: Comment[] },
  user: { id: string; name: string }
}
```

**Behaviors:**

* **Optimistic Updates:**
  Apply changes (likes/comments) instantly, rollback on failure.
* **Normalized Storage:**
  Map comments and stories by IDs.
* **Buffer for SSE stories:**
  Temporarily store new stories and show a “New Posts Available” banner.

---

### 6. 🔄 Infinite Scrolling (Old Feeds)

Use **IntersectionObserver** for older feed pagination.

**Frontend Flow:**

```js
const sentinel = document.querySelector('#load-more');

const observer = new IntersectionObserver(async entries => {
  if (entries[0].isIntersecting) {
    await fetchOlderFeeds(); // REST API call
  }
});

observer.observe(sentinel);
```

**API Example:**

```
GET /api/feed?before=<oldest_story_id>&limit=10
```

**Backend returns:**

```json
[
  { "id": 201, "author": "Zeeshan", "content": "Older post...", "createdAt": "..." }
]
```

---

### 7. ⚡ Real Time New Feeds via SSE

To get **new stories without sockets**, use **Server-Sent Events (SSE)**.

**Frontend:**

```js
// Connect to SSE endpoint
const eventSource = new EventSource('/api/feeds/stream');

eventSource.onmessage = (event) => {
  const newStory = JSON.parse(event.data);
  addToNewStoryBuffer(newStory); // Show "New posts available" banner
};
```

When user clicks “Show new posts” → prepend buffer to feed.

**Backend SSE Endpoint:**

```js
app.get('/api/feeds/stream', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.flushHeaders();

  const sendStory = (story) => {
    res.write(`data: ${JSON.stringify(story)}\n\n`);
  };

  feedEmitter.on('newStory', sendStory);

  req.on('close', () => feedEmitter.off('newStory', sendStory));
});
```

**Example Event Pushed:**

```
data: {"id":302,"author":"Ali","content":"New post here!"}
```

---

### 8. 🧭 Complete Feed Flow

| Direction                   | Mechanism | Trigger       | Endpoint                | Action                            |
| --------------------------- | --------- | ------------- | ----------------------- | --------------------------------- |
| Initial Load                | REST      | On mount      | `/api/feed?limit=10`    | Show first 10 stories             |
| Infinite Scroll (Old Feeds) | REST      | Scroll bottom | `/api/feed?before=<id>` | Append older stories              |
| New Feed Updates            | SSE       | Server push   | `/api/feeds/stream`     | Prepend new stories / show banner |

---

### 9. 🚀 Optimization Strategies

#### **A. Network Performance**

* Gzip/Brotli compression.
* WebP/AVIF images.
* Serve assets via CDN.
* Enable HTTP/2.
* Lazy load heavy media via IntersectionObserver.
* Bundle splitting: vendor vs. app bundles.

#### **B. Rendering Performance**

* Use SSR or hydration.
* Virtualized list rendering (React Window / VirtualScroller).
* Use `defer`/`async` for scripts.
* Memoize components and results.

#### **C. JS Performance**

* Debounce scroll handlers.
* Code-splitting by route.
* Offload heavy tasks to Web Workers.


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/m6xjpd9ycj43ra1tp5jc.png)
---

### 10. 🧩 Accessibility (A11y)

* Semantic HTML: `<article>`, `<header>`, `<section>`, `<button>`.
* ARIA roles for live feed updates:
  `aria-live="polite"`.
* Keyboard navigation.
* Alt text for media.
* High contrast colors.

---

### 11. 🧮 Example Flow: Combined Infinite Scroll + SSE

**Scenario:**

1. User opens app →
   `GET /api/feed?limit=10`
2. Scrolls to bottom →
   `GET /api/feed?before=201&limit=10`
3. Another user posts →
   Server sends SSE →
   `data: {"id":305,"author":"Sara","content":"New story!"}`
4. Frontend receives event →
   Shows “New posts available” banner.
5. User clicks →
   New story appears at top of feed.

---

### 12. 🧾 Endpoint Summary

| Endpoint                 | Method    | Description                   |
| ------------------------ | --------- | ----------------------------- |
| `/api/feed?limit=10`     | GET       | Initial feed load             |
| `/api/feed?before=<id>`  | GET       | Fetch older feeds (scroll)    |
| `/api/feeds/stream`      | GET (SSE) | Stream new feeds in real-time |
| `/api/story`             | POST      | Create new story              |
| `/api/story/:id/like`    | POST      | Like story                    |
| `/api/story/:id/comment` | POST      | Add comment                   |

---

### ✅ Final Summary

| Feature                | Mechanism             | Endpoint                | Direction     |
| ---------------------- | --------------------- | ----------------------- | ------------- |
| **Old Feeds (Scroll)** | REST API              | `/api/feed?before=<id>` | ⬇️ Downward   |
| **New Feeds (Live)**   | SSE                   | `/api/feeds/stream`     | ⬆️ Upward     |
| **Feed Rendering**     | React Components      | —                       | Client-side   |
| **State Sync**         | Store (Redux/Zustand) | —                       | Bidirectional |


---

More Details:

Get all articles related to system design 
Hastag: SystemDesignWithZeeshanAli


[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design
