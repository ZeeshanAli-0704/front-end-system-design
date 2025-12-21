### 🧠 System Design: Pinterest Frontend

In this blog post, we'll break down how we can design a Pinterest-like frontend with a focus on the Masonry Layout, its component architecture, and the strategies behind optimizing network, rendering, and JS performance.

---

#### Table of Contents

* [1. General Requirements](#1-general-requirements)
* [2. Component Architecture](#2-component-architecture)

  * [Top-Level Layout](#top-level-layout)
  * [Masonry Layout & Component Design](#masonry-layout--component-design)
  * [InfiniteScroll Component Design](#infinite-scroll-component-design)
  * [Component Hierarchy](#component-hierarchy)
* [3. Data Entities](#3-data-entities)
* [4. Data APIs](#4-data-apis)
* [5. Data Store (Frontend State Management)](#5-data-store-frontend-state-management)
* [6. Infinite Scrolling Working](#6-infinite-scrolling-working)
* [7. Optimization Strategies](#7-optimization-strategies)

  * [A. Network Performance](#a-network-performance)
  * [B. Rendering Performance](#b-rendering-performance)
  * [C. JS Performance](#c-js-performance)
* [8. Accessibility (A11y)](#8-accessibility-a11y)
* [9. Endpoint Summary](#9-endpoint-summary)
* [Final Summary](#final-summary)

---

### 1. 🎯 General Requirements

* **Feed Display**: Show images, videos, and cards in a Pinterest-style grid.
* **Infinite Scrolling**: Load more items as the user scrolls down.
* **Responsiveness**: Adapt the layout based on device width, ensuring smooth experience on mobile, tablet, and desktop.
* **User Interaction**: Like, save, and share images, and click on a card to view details.

---

### 2. 🧩 Component Architecture

We'll structure the frontend into modular, reusable components, with a particular focus on the **Masonry Layout**.

#### **Top-Level Layout**

The top-level layout will consist of:

* **Header**: Includes the Pinterest logo, search bar, and user options (e.g., profile, settings).
* **Grid Layout**: Displays images in a **Masonry Grid** layout that arranges cards dynamically.
* **Footer**: Contains links to privacy policies, terms, etc.

#### **Masonry Layout & Component Design**

The **Masonry Layout** is crucial for a Pinterest-like design as it ensures images of varying heights are arranged in a visually appealing staggered manner.

In the **MasonryGrid** component, we:

* **Dynamically Calculate Columns**: We calculate how many columns to display based on the available screen width.
* **Position Cards Absolutely**: Cards are positioned based on the calculated column height, creating the staggered grid effect.
* **Handle Image Loading**: Images are loaded before their position is calculated to avoid layout shifts during the page load.

Here’s the core **Masonry Layout** code:

```jsx
import React, { useEffect, useRef, useState } from "react";

export default function MasonryGrid({ items = [], columns = 4, gap = 16 }) {
  const containerRef = useRef(null);
  const [loading, setLoading] = useState(true); // Track loader visibility

  useEffect(() => {
    if (items.length === 0) {
      setLoading(false);
      return;
    }

    const container = containerRef.current;
    const images = container.querySelectorAll("img");
    let loadedCount = 0;

    const checkAllLoaded = () => {
      loadedCount++;
      if (loadedCount === images.length) {
        arrangeMasonry();
        setLoading(false);
      }
    };

    // Attach load handlers
    images.forEach((img) => {
      if (img.complete) checkAllLoaded();
      else img.onload = checkAllLoaded;
    });

    // Handle resize
    const handleResize = debounce(arrangeMasonry, 200);
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
    // eslint-disable-next-line
  }, [items]);

  function arrangeMasonry() {
    const container = containerRef.current;
    if (!container) return;

    const cards = Array.from(container.children);
    const containerWidth = container.clientWidth;
    const cardWidth = (containerWidth - (columns - 1) * gap) / columns;
    const columnHeights = Array(columns).fill(0);

    cards.forEach((card) => {
      card.style.width = `${cardWidth}px`;
      const minColIndex = columnHeights.indexOf(Math.min(...columnHeights));
      const x = (cardWidth + gap) * minColIndex;
      const y = columnHeights[minColIndex];
      card.style.transform = `translate(${x}px, ${y}px)`;

      const img = card.querySelector("img");
      const updateHeight = () => {
        columnHeights[minColIndex] = y + card.offsetHeight + gap;
        container.style.height = Math.max(...columnHeights) + "px";
      };
      if (img.complete) updateHeight();
      else img.onload = updateHeight;
    });
  }

  function debounce(fn, delay) {
    let timeout;
    return (...args) => {
      clearTimeout(timeout);
      timeout = setTimeout(() => fn.apply(this, args), delay);
    };
  }

  return (
    <div style={{ position: "relative", width: "100%", margin: "auto" }}>
      {loading && (
        <div
          className="loader"
          style={{
            position: "absolute",
            top: "50%",
            left: "50%",
            transform: "translate(-50%, -50%)",
            display: "flex",
            alignItems: "center",
            justifyContent: "center",
            flexDirection: "column",
            zIndex: 10,
          }}
        >
          <div className="spinner" />
          <p style={{ marginTop: 8, color: "#555" }}>Loading...</p>
        </div>
      )}

      <div
        ref={containerRef}
        className="masonry-container"
        style={{
          position: "relative",
          opacity: loading ? 0 : 1,
          transition: "opacity 0.3s ease-in-out",
        }}
      >
        {items.map((item) => (
          <div
            key={item.id}
            className="masonry-card"
            style={{
              position: "absolute",
              transition: "transform 0.3s ease",
            }}
          >
            <img
              src={item.src}
              alt={item.title}
              style={{ width: "100%", borderRadius: 8 }}
            />
            <div className="desc" style={{ padding: "8px" }}>
              {item.title}
            </div>
          </div>
        ))}
      </div>

      {/* Spinner CSS */}
      <style>
        {`
          .spinner {
            width: 40px;
            height: 40px;
            border: 4px solid #ccc;
            border-top-color: #007bff;
            border-radius: 50%;
            animation: spin 1s linear infinite;
          }

          @keyframes spin {
            to { transform: rotate(360deg); }
          }
        `}
      </style>
    </div>
  );
}
```

#### **Infinite Scroll Component Design**

Infinite scrolling is essential for ensuring that users can continuously browse the feed without needing to click "Load More."

* **Sentinel Element**: We use the **IntersectionObserver** API to detect when the user reaches the end of the feed, triggering the loading of new content.
* **Fetching More Data**: The frontend sends a request to fetch more items from the backend as the user scrolls down.

Example of infinite scroll using **IntersectionObserver**:

```js
const sentinel = document.querySelector('#load-more');

const observer = new IntersectionObserver(async entries => {
  if (entries[0].isIntersecting) {
    await fetchOlderFeeds(); // REST API call to fetch more items
  }
});

observer.observe(sentinel);
```

#### 🧠 Component Hierarchy

```
App
 ├── Header
 ├── MasonryGrid
 │    ├── MasonryCard (for each item)
 └── Footer
```

---

### 3. 🧱 Data Entities

In the Pinterest frontend design, we need to define data entities that will represent the main elements of the app—such as pins, media, and users. These data entities are essential for the system to function efficiently, and they are used to structure the data that the app will handle.

### 3. 🧱 Data Entities

In the Pinterest frontend design, we need to define data entities that will represent the main elements of the app—such as **pins**, **media**, and **users**. These data entities are essential for the system to function efficiently, and they are used to structure the data that the app will handle.

Here’s a breakdown of the **Data Entities** with the code snippet provided:

---

#### **Media Entity**

The **Media** entity represents the type of content attached to a **Pin**, which could either be an image or a video. This entity includes details about the media being displayed in the feed.

```ts
type Media = {
  id: string;              // Unique identifier for each media item
  type: 'image' | 'video'; // Type of media (either image or video)
  url: string;             // URL of the media file (image or video source)
  thumbnail?: string;      // Optional: URL for the thumbnail image, if applicable
};
```

**Explanation of each field:**

1. **`id`**: A unique identifier for each media file. This helps identify the specific media item, especially when there are multiple media items per pin.

2. **`type`**: This field tells whether the media is an image or a video. The type can be either `'image'` or `'video'`. This distinction helps the frontend decide how to render the media (for instance, if it's a video, it will need a player, while an image just needs to be displayed).

3. **`url`**: This is the URL pointing to the media file itself. It’s the primary link to where the actual image or video is hosted.

4. **`thumbnail`**: An optional field, providing a URL to a smaller version of the media (like a preview image for a video). This can be used to load a lighter-weight version of the media before the user clicks to view the full content.

---

#### **Pin Entity**

The **Pin** entity represents a Pinterest-like post or card that holds media and metadata such as the author’s information, title, likes, and comments. A pin could contain multiple media items, making it flexible for handling posts with different types of content.

```ts
type Pin = {
  id: string;                       // Unique identifier for each pin
  author: {                         // Details about the user who created the pin
    id: string;                     // Unique identifier for the author
    name: string;                   // Name of the author
    avatarUrl: string;              // URL to the author's avatar image
  };
  title: string;                    // The title or description of the pin
  media: Media[];                   // List of media associated with the pin
  likes: number;                    // Number of likes the pin has received
  comments: number;                 // Number of comments on the pin
  createdAt: string;                // Timestamp of when the pin was created
};
```

**Explanation of each field:**

1. **`id`**: This is the unique identifier for each pin. It helps distinguish one pin from another and is useful when interacting with the backend or performing CRUD operations on a pin.

2. **`author`**: The `author` object contains information about the user who created the pin. This is crucial because each pin is attributed to a user. The `author` object contains:

   * **`id`**: The unique identifier for the user who created the pin.
   * **`name`**: The name of the user.
   * **`avatarUrl`**: The URL of the user's avatar image, which is used to display the user's profile picture next to the pin.

3. **`title`**: The title or description of the pin. This field allows the author to describe the pin’s content (e.g., "Beautiful Sunset in Hawaii"). It’s usually displayed under or next to the image or video in the pin.

4. **`media`**: An array of `Media` objects, which allows a pin to have multiple media items (e.g., an image and a video). The `media` array can store various types of content related to the pin. This is useful when pins can contain different types of media (e.g., images, videos, etc.).

5. **`likes`**: The number of likes the pin has received. This metric shows how popular the pin is and may be shown as part of the pin's metadata. It can also be used to trigger social actions like showing who liked the pin.

6. **`comments`**: The number of comments the pin has received. This field helps track engagement and is displayed alongside other actions (like and share buttons) on the pin.

7. **`createdAt`**: A timestamp of when the pin was created. This is essential for sorting pins in chronological order or showing the age of the pin (e.g., "Created 3 hours ago").

---

#### **How These Entities Work Together**

* A **Pin** can have multiple **Media** items (e.g., an image and a video). This relationship between **Pin** and **Media** is critical because it allows for flexible content representation.

* Each **Pin** belongs to an **author**, who is a user of the platform. The **author**'s information (like name and avatar) is included in the pin so that users can see who created the pin.

* **Likes** and **comments** are part of the **Pin** entity and represent user interactions with the content. These fields help track the popularity and engagement of the pin.

---

#### **Example JSON Representation**

Here's an example of how the data would look like in practice when returned from an API:

```json
{
  "id": "pin123",
  "author": {
    "id": "user456",
    "name": "John Doe",
    "avatarUrl": "https://example.com/avatars/johndoe.jpg"
  },
  "title": "Beautiful Sunset",
  "media": [
    {
      "id": "media1",
      "type": "image",
      "url": "https://example.com/images/sunset.jpg",
      "thumbnail": "https://example.com/images/sunset_thumb.jpg"
    },
    {
      "id": "media2",
      "type": "video",
      "url": "https://example.com/videos/sunset.mp4"
    }
  ],
  "likes": 1023,
  "comments": 120,
  "createdAt": "2025-11-05T10:00:00Z"
}
```

In this example, the pin has two types of media: an image and a video. It shows the author's information, the number of likes and comments, and the timestamp of when the pin was created.

---

By defining these **Data Entities**, we have a clear structure for handling **pins**, **media**, and **user interactions**. This is crucial for the backend to know how to store and manage the data, and it also provides a solid foundation for rendering and managing the UI on the frontend.


---

### 4. 🔌 Data APIs

| API                     | Method   | Description                                |
| ----------------------- | -------- | ------------------------------------------ |
| `/api/pins?cursor=<id>` | **GET**  | Fetch paginated pins (for infinite scroll) |
| `/api/pins`             | **POST** | Create a new pin                           |
| `/api/pins/:id/like`    | **POST** | Like/unlike a pin                          |
| `/api/pins/:id/comment` | **POST** | Add a comment                              |
| `/api/pins/:id/comments`| **GET**  | Fetch comments for a pin                   |


---

### 5. 🗂️ Data Store (Frontend State Management)

In a modern frontend application like Pinterest, managing the application state is crucial for ensuring smooth user interactions and keeping the UI in sync with the backend. For managing this state, we can either rely on React's built-in state management (via hooks like `useState` and `useEffect`) or opt for a global state management tool like **Redux**, **Context API**, or **Zustand**.

We'll dive into how **state management** can be handled in this Pinterest frontend design.

---

#### **State Structure**

The frontend state represents the data that is used by the application. For a Pinterest-like app, the state will typically consist of the following:

```ts
{
  pins: Pin[];       // Array of pins displayed in the feed
  cursor: string;    // Pagination cursor for fetching older feed items
  loading: boolean;  // Whether data is being fetched (loading state)
}
```

Each of these fields plays a specific role in managing the app's behavior:

1. **`pins`**:

   * This is an array of `Pin` objects, which contain all the data about the pins being displayed in the feed. This can include the pin's content (images, videos), metadata (likes, comments), and the author information.
   * The `pins` array will be populated from the backend via an API call and then displayed in the UI, usually in a **Masonry Grid** or similar layout.

2. **`cursor`**:

   * The `cursor` is used for **pagination**. It holds a string value that helps in fetching older feed items when the user scrolls to the bottom of the page.
   * Typically, pagination is done in a way where the server returns a "cursor" or an "ID" of the last fetched item, and this value is used in the next API request to fetch additional items after the last item.
   * This is important for **infinite scrolling**—a feature in Pinterest where new items are loaded dynamically as the user scrolls down.

3. **`loading`**:

   * This boolean flag is used to indicate whether the app is currently fetching new data from the backend.
   * It can be used to show loading indicators (e.g., spinners) when new pins are being loaded.
   * This state prevents multiple requests from being sent simultaneously and provides a smoother user experience by signaling when data is still being fetched.

---

#### **Optimistic Updates**

**Optimistic updates** is a pattern where the UI is immediately updated as if an action (like liking a pin or posting a comment) has already been successfully completed, before the actual backend request is made. This makes the app feel faster and more responsive, as the user sees the result immediately.

In the case of a **Pinterest frontend**:

* **Likes and Comments**:

  * When the user likes or comments on a pin, instead of waiting for the API call to succeed, the frontend **optimistically updates** the state to reflect the user's action.
  * For example, if a user clicks on the "like" button, the number of likes for the pin will immediately increment on the screen.
  * If the API call succeeds, the frontend state is not changed because it already reflected the user’s action. If the API call fails, the update is rolled back, and the UI is restored to its previous state.

**Example of Optimistic Update for "Liking" a Pin:**

```ts
const handleLike = (pinId: string) => {
  // Optimistically update the UI
  setPins(prevPins => prevPins.map(pin => 
    pin.id === pinId 
    ? { ...pin, likes: pin.likes + 1 }  // Increment the like count
    : pin
  ));

  // Send the API request to like the pin
  api.likePin(pinId)
    .then(response => {
      // If the API request was successful, do nothing (state is already updated)
    })
    .catch(error => {
      // If the request fails, roll back the update
      setPins(prevPins => prevPins.map(pin => 
        pin.id === pinId 
        ? { ...pin, likes: pin.likes - 1 }  // Decrement the like count
        : pin
      ));
      console.error("Failed to like the pin:", error);
    });
};
```

#### How It Works:

1. **Immediate UI Update**: As soon as the "like" button is clicked, the `handleLike` function optimistically updates the state by increasing the `likes` count for the pin.

2. **API Call**: The backend request is made via `api.likePin(pinId)`, but the user doesn't need to wait for the backend to respond because the UI has already reflected the expected result.

3. **Roll Back on Failure**: If the API request fails (e.g., due to a network issue), the state is updated again to roll back the optimistic change (i.e., decrement the like count). This ensures that the UI is consistent with the server's state.

---

#### **State Management with React or Redux**

* **React's Local State (`useState` & `useEffect`)**:

  * For smaller applications or local state management, you can manage this state within individual components using **React hooks** like `useState` and `useEffect`. For example, the `pins`, `cursor`, and `loading` state would typically be managed using `useState`.
* **Global State (Redux/Context API)**:

  * For larger applications where state needs to be shared across multiple components (e.g., across pages or deep components), a **global state manager** like **Redux** or **Context API** is used.
  * With **Redux**, you would define actions to fetch pins, handle pagination, and update the like count. The global state would hold the `pins`, `cursor`, and `loading` flags.

#### Example Redux Store:

```ts
const initialState = {
  pins: [],
  cursor: '',
  loading: false
};

const pinReducer = (state = initialState, action) => {
  switch (action.type) {
    case 'FETCH_PINS_REQUEST':
      return { ...state, loading: true };
    case 'FETCH_PINS_SUCCESS':
      return { 
        ...state, 
        loading: false, 
        pins: [...state.pins, ...action.payload.pins],
        cursor: action.payload.cursor
      };
    case 'LIKE_PIN_SUCCESS':
      return { 
        ...state, 
        pins: state.pins.map(pin => 
          pin.id === action.payload.pinId ? { ...pin, likes: pin.likes + 1 } : pin
        )
      };
    default:
      return state;
  }
};
```

In this example, the Redux reducer handles the logic of updating the state when new pins are fetched or when a pin is liked.

---

#### **Handling Data Fetching and Pagination**

For **infinite scrolling** or **pagination**, the `cursor` field plays a key role. Here's how it works:

* When the user scrolls to the bottom of the feed, the `cursor` is passed to the API to fetch more pins.
* After the pins are fetched, the `pins` array is updated with new data, and the `cursor` is updated to the latest cursor returned from the backend.

```ts
const loadMorePins = () => {
  if (loading) return; // Avoid multiple requests

  setLoading(true);

  api.fetchPins(cursor)
    .then(response => {
      setPins(prevPins => [...prevPins, ...response.pins]);
      setCursor(response.nextCursor); // Set the new cursor for the next batch
    })
    .finally(() => {
      setLoading(false);
    });
};
```

---

#### **overview**

By managing the frontend state effectively, we can ensure a seamless and responsive experience for users. The state management structure:

* **Stores the feed data (`pins`)**, the **pagination cursor (`cursor`)**, and the **loading state (`loading`)**.
* Implements **optimistic updates** to make interactions like liking or commenting feel instantaneous, even before the backend confirms the action.
* Utilizes React's local state or a global state management tool like Redux to synchronize the UI with the data from the backend, ensuring that the app responds promptly and consistently to user actions.


### 6. 🔄 Infinite Scrolling Working

Infinite scrolling works by detecting when the user has scrolled near the bottom of the feed and triggering a fetch for older pins. This is handled with an **IntersectionObserver** that watches the sentinel element.

---

### 7. 🚀 Optimization Strategies

#### **A. Network Performance**

* Use **lazy loading** for images.
* Implement **Gzip/Brotli** compression.
* Use **CDN** for media files.
* Minimize API calls by using **pagination** and **caching**.

#### **B. Rendering Performance**

* Use **CSS Grid** or **Flexbox** for layout (avoiding JS-heavy layout recalculation).
* **Virtualize the list** using tools like **react-window** for better performance on large feeds.

#### **C. JS Performance**

* **Debounce** window resizing and scroll events to prevent excessive function calls.
* **Code-split** large JavaScript bundles.
* Minimize **re-renders** with React's `memo` and `useMemo`.

---

### 8. 🧩 Accessibility (A11y)

* **ARIA roles**: Use appropriate roles and attributes for accessibility.
* **Alt text for images**: Ensure each image has a descriptive `alt` tag.
* **Keyboard Navigation**: Allow users to navigate the grid using keyboard shortcuts.
* **High Contrast**: Ensure high contrast for better visibility.

---

### 9. 🧾 Endpoint Summary

| Endpoint                 | Method        | Description                  |
| ------------------------ | ------------- | ---------------------------- |
| `/api/pins?cursor=<id>`  | **GET**       | Fetch paginated pins         |
| `/api/pins`              | **POST**      | Create a new pin             |
| `/api/pins/:id/like`     | **POST**      | Like/unlike a pin            |
| `/api/pins/:id/comments` | **GET**       | Fetch comments for a pin     |

---

| Feature               | Mechanism           | Endpoint                | Direction     |
| --------------------- | ------------------- | ----------------------- | ------------- |
| **Old Pins (Scroll)** | REST API            | `/api/pins?cursor=<id>` | ⬇️ Downward   |
| **New Pins (Live)**   | SSE                 | `/api/pins/stream`      | ⬆️ Upward     |
| **Feed Rendering**    | React Components    | —                       | Client-side   |
| **State Management**  | Redux / Context API | —                       | Bidirectional |


### ✅ Final Summary

This Pinterest-like frontend design focuses on efficient state management, ensuring a smooth user experience. We manage data with React hooks or Redux, handling dynamic feed content, pagination, and UI states like loading indicators. The design incorporates **infinite scrolling** for fetching older pins and new pins. **Optimistic updates** are used for actions like liking and commenting, making the UI feel instantaneous. Performance optimizations are applied across **network**, **rendering**, and **JS** to ensure fast and smooth interactions. The architecture is built for scalability, ensuring it can handle large amounts of content and user engagement efficiently.


These libraries provide various solutions from lightweight CSS approaches to full-featured JavaScript libraries for dynamic layouts, filtering, and reordering content

1. **Masonry.js**
   [https://masonry.desandro.com/](https://masonry.desandro.com/)

2. **React Masonry Component**
   [https://www.npmjs.com/package/react-masonry-component](https://www.npmjs.com/package/react-masonry-component)

3. **React Grid Layout**
   [https://react-grid-layout.github.io/react-grid-layout/](https://react-grid-layout.github.io/react-grid-layout/)

4. **Isotope.js**
   [https://isotope.metafizzy.co/](https://isotope.metafizzy.co/)

5. **PhotoSwipe**
   [https://photoswipe.com/](https://photoswipe.com/)

6. **Flickity**
   [https://flickity.metafizzy.co/](https://flickity.metafizzy.co/)

.


More Details:

Get all articles related to system design 
Hastag: SystemDesignWithZeeshanAli


[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design