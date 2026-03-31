# Frontend System Design: Typeahead Autocomplete

- [Frontend System Design: Typeahead Autocomplete](#frontend-system-design-typeahead-autocomplete)
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
    - [5.2 Typeahead Input and Dropdown Rendering Strategy (Deep Dive)](#52-typeahead-input-and-dropdown-rendering-strategy-deep-dive)
      - [5.2.1 Debouncing Strategy and Why It Is Critical](#521-debouncing-strategy-and-why-it-is-critical)
      - [5.2.2 Handling Out of Order Responses (Race Conditions)](#522-handling-out-of-order-responses-race-conditions)
      - [5.2.3 Request Cancellation with AbortController](#523-request-cancellation-with-abortcontroller)
      - [5.2.4 Client Side Suggestion Cache](#524-client-side-suggestion-cache)
      - [5.2.5 Prefix Highlighting in Suggestions](#525-prefix-highlighting-in-suggestions)
      - [5.2.6 Keyboard Navigation and Selection](#526-keyboard-navigation-and-selection)
      - [5.2.7 Dropdown Positioning and Portal Rendering](#527-dropdown-positioning-and-portal-rendering)
      - [5.2.8 Mobile Specific Considerations](#528-mobile-specific-considerations)
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
    - [9.2 Request Lifecycle and Optimization (Deep Dive)](#92-request-lifecycle-and-optimization-deep-dive)
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

*   Typeahead (autocomplete) is a search input component that predicts and displays query suggestions in real time as the user types characters into a search box.
*   Target users: all users of a search-enabled platform — Google Search, YouTube, Amazon, GitHub, Slack, or any app with a search bar.
*   Primary use case: reducing user effort by suggesting complete queries after only a few keystrokes, improving search speed, accuracy, and discoverability.

---

### 1.2 Key User Personas

*   **Casual Searcher**: Types a few characters, picks a suggestion from the dropdown, and expects instant results. Represents the majority of users.
*   **Power User**: Types rapidly, uses keyboard shortcuts (arrow keys, Enter), and expects the UI to keep up without lag or stale results.
*   **Mobile User**: Taps on a small keyboard, expects fewer but highly relevant suggestions, and needs touch-friendly dropdown items with minimal layout disruption.

---

### 1.3 Core User Flows (High Level)

*   **Typing and Selecting a Suggestion (Primary Flow)**:
    1.  User focuses the search input.
    2.  User types characters (e.g., `s` → `sy` → `sys`).
    3.  After a debounce pause, the frontend sends the current prefix to the backend.
    4.  Backend returns ranked suggestions (e.g., "system design", "system design interview", "system configuration").
    5.  Dropdown appears below the input with highlighted matching prefix.
    6.  User selects a suggestion via click or keyboard (↓ + Enter).
    7.  Input is populated with the selected suggestion → search executes.

*   **Typing and Submitting a Custom Query (Secondary Flow)**:
    1.  User types a full query without selecting any suggestion.
    2.  User presses Enter or taps the Search button.
    3.  Search executes with the raw input value.
    4.  Dropdown closes.

*   **Clearing and Re-Searching (Secondary Flow)**:
    1.  User clears the input (backspace or clear button).
    2.  Dropdown closes when input is empty.
    3.  User types a new query → flow restarts.

---

## 2. Requirements

### 2.1 Functional Requirements

*   **Suggestion Display**:
    *   Show a dropdown of top K suggestions (typically K = 5–10) that match the typed prefix.
    *   Each suggestion shows the full query text with the typed prefix highlighted/bolded.
    *   Optionally show suggestion category icons (e.g., recent search, trending, product, person).
*   **Input Behavior**:
    *   Debounce keystrokes to avoid flooding the backend with requests on every character.
    *   Cancel stale in-flight requests when the user continues typing (abort previous fetch).
    *   Clear suggestions when input is emptied.
*   **Selection**:
    *   Click on a suggestion to populate the input and execute search.
    *   Keyboard navigation: ↑/↓ to navigate suggestions, Enter to select, Escape to close.
    *   On selection, the dropdown closes and the selected suggestion replaces the input value.
*   **Recent Searches**:
    *   Show recent search history when the input is focused but empty.
    *   Allow clearing individual recent items or all history.
*   **Trending Suggestions**:
    *   Show trending/popular queries when no prefix is entered (optional).

---

### 2.2 Non Functional Requirements

*   **Performance**: Perceived suggestion latency < 100ms from keystroke to dropdown render. Debounce interval: 150ms (desktop), 250ms (mobile). API response target: < 50ms.
*   **Scalability**: Handle high-frequency keystrokes from millions of concurrent users; backend queries per second (QPS) can reach 100,000+.
*   **Availability**: Autocomplete is an enhancement, not a hard dependency. If suggestions fail, the user can still type and submit a search. Graceful degradation: hide dropdown on error, never block typing.
*   **Security**: Sanitize user input before rendering suggestions (XSS prevention). Do not log sensitive prefixes client-side. Respect user privacy settings.
*   **Accessibility**: Full keyboard navigation (arrow keys, Enter, Escape). ARIA `combobox` pattern with `listbox` and `option` roles. Screen reader announces suggestion count and active option.
*   **Device Support**: Desktop (primary), mobile web (responsive), tablet. Touch-friendly dropdown items on mobile.
*   **i18n**: RTL language support for input and dropdown. Locale parameter in API requests for localized suggestions.

---

## 3. Scope Clarification (Interview Scoping)

### 3.1 In Scope

*   Typeahead input component with debounced API calls.
*   Suggestion dropdown rendering with prefix highlighting.
*   Keyboard navigation and selection.
*   Race condition handling (out-of-order responses, request cancellation).
*   Client-side suggestion caching.
*   Accessibility (ARIA combobox pattern).
*   State management for suggestions, loading, and error states.
*   API design from the frontend perspective.
*   Performance optimization for low-latency interaction.

---

### 3.2 Out of Scope

*   Backend trie data structure, ranking algorithm, and distributed storage (covered in the companion article).
*   Full search results page and result rendering.
*   Visual search or image-based autocomplete.
*   Voice input integration.
*   Rich suggestion cards (product previews, person cards) — these are extensions of the base pattern.

---

### 3.3 Assumptions

*   Backend autocomplete API is available and returns ranked suggestions for a given prefix within < 50ms.
*   Backend handles all ranking, frequency weighting, and personalization — frontend only sends the prefix and renders results.
*   User is not necessarily authenticated (autocomplete works for anonymous users too).
*   The typeahead component is embedded within a larger application (e.g., a search page, nav bar, or command palette).

---

## 4. High Level Frontend Architecture

### 4.1 Overall Approach

*   The typeahead is a **self-contained, reusable component** that can be dropped into any page or context.
*   **CSR** (Client-Side Rendering) — the typeahead is inherently interactive; SSR provides no benefit for a component that activates on user focus/input.
*   The component is **code-split** and lazy loaded if it lives behind a user action (e.g., clicking a search icon in the nav bar expands the search input).
*   No global state — the typeahead manages its own state (input value, suggestions, active index, loading, error) internally via `useReducer` or a local store.

---

### 4.2 Major Architectural Layers

```
┌──────────────────────────────────────────────────────────────┐
│  UI Layer                                                    │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  TypeaheadInput                                         │ │
│  │  ┌────────────────────────────┐                         │ │
│  │  │ SearchInput (text input)   │                         │ │
│  │  └────────────────────────────┘                         │ │
│  │  ┌────────────────────────────────────────────────────┐ │ │
│  │  │ SuggestionDropdown                                 │ │ │
│  │  │  ┌─────────────────┐  ┌─────────────────┐         │ │ │
│  │  │  │ SuggestionItem  │  │ SuggestionItem  │ ...     │ │ │
│  │  │  └─────────────────┘  └─────────────────┘         │ │ │
│  │  └────────────────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────┤
│  State Management Layer                                      │
│  (Input Value, Suggestions List, Active Index, Loading,      │
│   Error, Dropdown Visibility, Recent Searches)               │
├──────────────────────────────────────────────────────────────┤
│  API and Data Access Layer                                   │
│  (Fetch with AbortController, Debounce Logic, Client Cache,  │
│   Request Versioning, Retry Logic)                           │
├──────────────────────────────────────────────────────────────┤
│  Shared / Utility Layer                                      │
│  (Debounce/Throttle, String highlighting, Keyboard handler,  │
│   ARIA helpers, Analytics tracker)                           │
└──────────────────────────────────────────────────────────────┘
```

---

### 4.3 External Integrations

*   **Autocomplete API**: `GET /api/autocomplete?prefix=<prefix>&k=<count>&locale=<locale>`. Returns ranked suggestions for the given prefix. Served from Redis (hot prefixes) or distributed trie (cold prefixes).
*   **Search API**: On suggestion selection or Enter press, the full search query is sent to the search endpoint.
*   **Analytics SDK**: Track suggestion impressions (shown), suggestion taps (selected), manual search submits (typed), and query abandonment (typed but neither selected nor searched).
*   **Recent Searches Storage**: localStorage or backend API for persisting recent search history per user.

---

## 5. Component Design and Modularization

### 5.1 Component Hierarchy

```
TypeaheadContainer
 ├── SearchInput
 │    ├── SearchIcon
 │    ├── TextInput (controlled, with aria-combobox)
 │    └── ClearButton (visible when input has value)
 │
 └── SuggestionDropdown (conditional — visible when suggestions exist)
      ├── RecentSearchesSection (when input is empty, on focus)
      │    └── RecentSearchItem[] (with delete button)
      │
      ├── TrendingSuggestionsSection (when input is empty, optional)
      │    └── TrendingItem[]
      │
      └── SuggestionList (when input has value and API returned results)
           └── SuggestionItem[] (role="option")
                ├── SuggestionIcon (search icon, clock for recent, trending arrow)
                ├── HighlightedText (prefix bolded, rest normal)
                └── ActionIcon (optional — "fill input" arrow, remove from recents)
```

---

### 5.2 Typeahead Input and Dropdown Rendering Strategy (Deep Dive)

The typeahead component is one of the most **deceptively simple** UI components in frontend system design. It looks like "just an input with a dropdown" — but getting it production-ready requires solving:
*   **Timing problems** (debounce, race conditions, stale responses).
*   **Performance problems** (request flooding, unnecessary re-renders).
*   **Accessibility problems** (keyboard navigation, screen reader announcements).
*   **Edge cases** (rapid typing, slow networks, empty states, input clearing).

---

#### 5.2.1 Debouncing Strategy and Why It Is Critical

##### The Problem — Every Keystroke Triggers a Request

Without debouncing, a user typing "system design" generates **13 network requests**:

```
Keystroke:  s   y   s   t   e   m   (space)   d   e   s   i   g   n
Request:   [1] [2] [3] [4] [5] [6]    [7]    [8] [9] [10][11][12][13]
```

At 100,000 concurrent users, this means **1.3 million requests** for a single common query. Most of these requests are **wasted** — by the time the response for "s" arrives, the user has already typed "sys" and needs a different set of suggestions.

##### How Debouncing Works

Debouncing delays the API call until the user **stops typing** for a specified duration. If the user types another character within that window, the timer resets.

```
User types: s → y → s → t → e → m
             |   |   |   |   |   |
             150ms────────────────┤
                                  └─ FIRE: fetch("system")
                                     Only 1 request!
```

##### Implementation

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer); // reset timer on new value
  }, [value, delay]);

  return debouncedValue;
}

// Usage in TypeaheadContainer
function TypeaheadContainer() {
  const [inputValue, setInputValue] = useState('');
  const debouncedPrefix = useDebounce(inputValue, 150);

  // Fetch triggers when debouncedPrefix changes (not on every keystroke)
  useEffect(() => {
    if (debouncedPrefix.length === 0) {
      clearSuggestions();
      return;
    }
    fetchSuggestions(debouncedPrefix);
  }, [debouncedPrefix]);

  // ...
}
```

##### Choosing the Right Debounce Interval

| Device | Recommended Delay | Reasoning |
|--------|------------------|-----------|
| **Desktop (keyboard)** | 100–150ms | Fast typists average 50-80ms between keystrokes. 150ms catches most natural pauses. |
| **Mobile (touch keyboard)** | 200–300ms | Touch typing is slower; larger delay reduces unnecessary requests on imprecise input. |
| **Slow network (detected)** | 250–400ms | On slow connections, requests take longer; sending fewer but more complete prefixes is more efficient. |

**Why not 0ms (no debounce)?** — Backend is flooded with requests. Each request competes for bandwidth. Earlier responses are discarded anyway.

**Why not 500ms+?** — User perceives noticeable lag. "I typed 'system' and nothing happened for half a second." The UI feels unresponsive.

**Why 150ms is the sweet spot for desktop:**
*   Average inter-key delay for fluent typing: ~80ms.
*   After 150ms of no typing, the user has likely paused (thinking, reading suggestions, or finished a word).
*   API response (< 50ms) + render (< 10ms) ≈ 60ms. Total: 150ms + 60ms = **210ms from last keystroke to visible suggestions**. This feels instant.

##### Adaptive Debouncing (Advanced)

Instead of a fixed delay, adjust the debounce based on context:

```tsx
function getAdaptiveDelay(prefix: string, networkType?: string): number {
  // Short prefixes generate huge result sets — debounce more aggressively
  if (prefix.length <= 2) return 300;

  // On slow networks, reduce request frequency
  if (networkType === '2g' || networkType === 'slow-2g') return 400;
  if (networkType === '3g') return 250;

  // Default desktop delay
  return 150;
}
```

**Why short prefixes get a longer delay:**
*   The prefix "a" matches millions of queries — the API is doing the most work for the least specific input.
*   Users almost never stop at one character — they are still typing. Waiting 300ms is safe.
*   The prefix "system des" is very specific — 150ms is appropriate because the user is close to their target and the result set is small.

---

#### 5.2.2 Handling Out of Order Responses (Race Conditions)

##### The Problem

Network responses do **not** arrive in the order they were sent. If the user types "ho" → "how", the API call for "ho" might return **after** the API call for "how":

```
Time →

User types: h → o → w
                |       |
  Request "ho" ─┘       └─ Request "how"
                |             |
  Response "how" arrives      |
  (fast: 40ms)                |
                              Response "ho" arrives
                              (slow: 120ms — network jitter)

Without protection:
  Dropdown shows results for "how" ✅
  Then OVERWRITES with results for "ho" ❌ (stale!)
```

The user is looking at results for "ho" even though they have already typed "how". This is a **classic race condition** in frontend systems.

##### Solution 1: Request Versioning (Simple and Effective)

Track a monotonically increasing request ID. Only render the response if its ID matches the latest request.

```tsx
function useTypeahead() {
  const [suggestions, setSuggestions] = useState<string[]>([]);
  const requestIdRef = useRef(0);

  const fetchSuggestions = useCallback(async (prefix: string) => {
    const currentId = ++requestIdRef.current;

    try {
      const response = await fetch(`/api/autocomplete?prefix=${encodeURIComponent(prefix)}&k=10`);
      const data = await response.json();

      // Only update state if this is still the latest request
      if (currentId !== requestIdRef.current) {
        return; // stale response — discard silently
      }

      setSuggestions(data.suggestions);
    } catch (error) {
      if (currentId === requestIdRef.current) {
        setSuggestions([]);
      }
    }
  }, []);

  return { suggestions, fetchSuggestions };
}
```

**How it works:**

```
Request "ho"  → requestId = 1
Request "how" → requestId = 2

Response for "how" arrives (requestId check: 2 === 2 ✅) → update state
Response for "ho" arrives  (requestId check: 1 !== 2 ❌) → discarded
```

##### Solution 2: AbortController (Cancel the Request Entirely)

Instead of discarding the response after it arrives, **cancel the in-flight request** so the browser never even processes the response. This saves bandwidth and CPU.

See section 5.2.3 for the full implementation.

##### Comparison

| Approach | When stale response arrives | Network cost | Complexity |
|----------|---------------------------|-------------|------------|
| **Request versioning** | Response arrives but is discarded | Full response downloaded (wasted bandwidth) | Simple — one `ref` |
| **AbortController** | Request is cancelled; no response data transferred | Minimal (request aborted mid-flight) | Moderate — must handle `AbortError` |
| **Both combined** | Belt and suspenders — abort + version check as fallback | Minimal | Slightly more code but most robust |

**Recommendation:** Use **AbortController as primary** (saves bandwidth) with **request versioning as a safety net** (catches edge cases where abort doesn't fire in time).

---

#### 5.2.3 Request Cancellation with AbortController

##### Why Cancellation Matters

When a user types fast, multiple fetch requests may be in flight simultaneously:

```
Active requests at time T when user has typed "syst":
  fetch("s")    → in flight (200ms old, still waiting)
  fetch("sy")   → in flight (150ms old)
  fetch("sys")  → in flight (80ms old)
  fetch("syst") → just sent
```

Without cancellation, the browser:
*   Has 4 concurrent connections to the same endpoint.
*   May hit the browser's per-origin connection limit (6 concurrent in HTTP/1.1).
*   Downloads and parses 3 responses that will be immediately discarded.

With `AbortController`, we cancel all previous requests when a new one is sent:

```
User types "syst":
  fetch("s")    → ABORTED ✗ (freed connection)
  fetch("sy")   → ABORTED ✗ (freed connection)
  fetch("sys")  → ABORTED ✗ (freed connection)
  fetch("syst") → ACTIVE ✓ (only this one is in flight)
```

##### Implementation

```tsx
function useTypeaheadWithAbort() {
  const [suggestions, setSuggestions] = useState<string[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const abortControllerRef = useRef<AbortController | null>(null);
  const requestIdRef = useRef(0);

  const fetchSuggestions = useCallback(async (prefix: string) => {
    // 1. Abort any previous in-flight request
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }

    // 2. Create a new AbortController for this request
    const controller = new AbortController();
    abortControllerRef.current = controller;
    const currentId = ++requestIdRef.current;

    setIsLoading(true);

    try {
      const response = await fetch(
        `/api/autocomplete?prefix=${encodeURIComponent(prefix)}&k=10`,
        { signal: controller.signal }
      );
      const data = await response.json();

      // 3. Safety check — ensure this is still the latest request
      if (currentId !== requestIdRef.current) return;

      setSuggestions(data.suggestions);
    } catch (error) {
      // 4. AbortError is expected — don't treat it as a real error
      if (error instanceof DOMException && error.name === 'AbortError') {
        return; // silently ignore — request was intentionally cancelled
      }

      // 5. Real errors (network failure, server error)
      if (currentId === requestIdRef.current) {
        setSuggestions([]);
      }
    } finally {
      if (currentId === requestIdRef.current) {
        setIsLoading(false);
      }
    }
  }, []);

  // 6. Cleanup on unmount — abort any in-flight request
  useEffect(() => {
    return () => {
      abortControllerRef.current?.abort();
    };
  }, []);

  return { suggestions, isLoading, fetchSuggestions };
}
```

##### Why Catching `AbortError` Is Critical

When a request is aborted, `fetch()` throws a `DOMException` with `name: 'AbortError'`. If we don't catch it specifically:
*   The catch block treats it as a real error.
*   The UI might flash an error state for a perfectly normal cancellation.
*   Error monitoring systems log false positives.

```tsx
// ✅ Correct: abort is expected, not an error
if (error.name === 'AbortError') return;

// ❌ Wrong: treating abort as a failure
catch (error) {
  setError('Failed to fetch suggestions'); // shows error for normal typing!
}
```

---

#### 5.2.4 Client Side Suggestion Cache

##### The Problem — Repeated Prefixes

Users often type, backspace, and retype. Or they search for similar queries across a session:

```
Session:
  User types "how" → gets suggestions → selects "how to cook rice"
  User clears input
  User types "how" again → same suggestions needed
  Without cache: another API request
  With cache: instant results from memory
```

##### Implementation — LRU Cache with TTL

```tsx
class SuggestionCache {
  private cache: Map<string, { suggestions: string[]; timestamp: number }>;
  private maxSize: number;
  private ttlMs: number;

  constructor(maxSize = 50, ttlMs = 5 * 60 * 1000) { // 50 entries, 5 min TTL
    this.cache = new Map();
    this.maxSize = maxSize;
    this.ttlMs = ttlMs;
  }

  get(prefix: string): string[] | null {
    const entry = this.cache.get(prefix);
    if (!entry) return null;

    // Check TTL
    if (Date.now() - entry.timestamp > this.ttlMs) {
      this.cache.delete(prefix);
      return null;
    }

    // Move to end (most recently used) — LRU eviction support
    this.cache.delete(prefix);
    this.cache.set(prefix, entry);

    return entry.suggestions;
  }

  set(prefix: string, suggestions: string[]): void {
    // Evict oldest entry if at capacity
    if (this.cache.size >= this.maxSize) {
      const oldestKey = this.cache.keys().next().value;
      this.cache.delete(oldestKey);
    }

    this.cache.set(prefix, { suggestions, timestamp: Date.now() });
  }

  clear(): void {
    this.cache.clear();
  }
}

// Usage
const suggestionCache = new SuggestionCache();

async function fetchSuggestions(prefix: string) {
  // 1. Check cache first
  const cached = suggestionCache.get(prefix);
  if (cached) {
    setSuggestions(cached);
    return; // instant — no network request
  }

  // 2. Cache miss — fetch from API
  const data = await fetchFromAPI(prefix);
  setSuggestions(data.suggestions);

  // 3. Store in cache for future lookups
  suggestionCache.set(prefix, data.suggestions);
}
```

##### Why Limit Cache Size to 50 Entries?

*   Each entry is tiny: a prefix string (5-20 bytes) + an array of 5-10 suggestion strings (100-500 bytes total) ≈ **~500 bytes per entry**.
*   50 entries ≈ **~25KB** — negligible memory cost.
*   100+ entries rarely provide additional cache hits (users don't typically search > 100 distinct prefixes in a session).
*   The LRU eviction ensures the cache always contains the **most recently searched** prefixes, which have the highest reuse probability.

##### Prefix Hierarchy Optimization (Advanced)

If the user types "sys", we send an API request. When they continue to "syst", instead of fetching again, we can **filter the existing "sys" results client-side**:

```tsx
function getOptimisticSuggestions(prefix: string, cache: SuggestionCache): string[] | null {
  // Check for exact cache hit first
  const exact = cache.get(prefix);
  if (exact) return exact;

  // Check if a shorter prefix has cached results we can filter
  for (let i = prefix.length - 1; i >= 1; i--) {
    const shorterPrefix = prefix.substring(0, i);
    const cached = cache.get(shorterPrefix);
    if (cached) {
      // Filter cached results that still match the longer prefix
      const filtered = cached.filter((s) =>
        s.toLowerCase().startsWith(prefix.toLowerCase())
      );
      if (filtered.length > 0) return filtered;
    }
  }

  return null;
}
```

**When this works well:**
*   "sys" → API returns ["system design", "system config", "sysadmin"]. User types "syst" → instantly filter to ["system design", "system config"]. No API call needed.

**When this breaks:**
*   If the backend's top-K for "syst" includes results that weren't in the top-K for "sys" (e.g., "systemic risk" was #11 for "sys" but #3 for "syst"). The filtered client-side results would miss it.
*   **Mitigation:** Use this as an **optimistic fast path** — show filtered results immediately, then fire the API call in the background and update if the server returns different results.

---

#### 5.2.5 Prefix Highlighting in Suggestions

##### The Concept

When the user types "how", suggestion items should visually distinguish the matching prefix from the rest:

```
Input: "how"

Dropdown:
  [how] to cook rice            ← "how" is bold, rest is normal
  [how] to lose weight
  [how] to make tea
```

This helps users:
*   Confirm the system understood their input.
*   Scan results faster by reading only the non-bold (differentiating) part.
*   Identify exact matches vs partial matches.

##### Implementation

```tsx
function HighlightedSuggestion({
  suggestion,
  prefix,
}: {
  suggestion: string;
  prefix: string;
}) {
  const lowerSuggestion = suggestion.toLowerCase();
  const lowerPrefix = prefix.toLowerCase();

  // Find where the prefix matches in the suggestion
  const matchIndex = lowerSuggestion.indexOf(lowerPrefix);

  if (matchIndex === -1) {
    // No match — render as-is (shouldn't happen for prefix-matched results)
    return <span>{suggestion}</span>;
  }

  const before = suggestion.slice(0, matchIndex);
  const match = suggestion.slice(matchIndex, matchIndex + prefix.length);
  const after = suggestion.slice(matchIndex + prefix.length);

  return (
    <span>
      {before}
      <strong>{match}</strong>
      {after}
    </span>
  );
}
```

##### Why `indexOf` Instead of Just `startsWith`?

Most autocomplete systems return prefix-matched results (suggestions **start with** the prefix). But some systems also return:
*   **Infix matches**: "how" matches "learn **how** to code" (match is mid-string).
*   **Fuzzy matches**: "systm" matches "system" (typo tolerance).

Using `indexOf` handles both prefix and infix matches. For fuzzy matches, a more sophisticated highlighting algorithm is needed (e.g., highlight each matching character individually).

---

#### 5.2.6 Keyboard Navigation and Selection

##### Requirements

The typeahead must be fully operable via keyboard:

| Key | Action |
|-----|--------|
| ↓ (ArrowDown) | Move highlight to next suggestion |
| ↑ (ArrowUp) | Move highlight to previous suggestion |
| Enter | Select the highlighted suggestion (or submit raw input) |
| Escape | Close the dropdown |
| Tab | Close dropdown + move focus to next element |

##### Implementation with useReducer

```tsx
type TypeaheadState = {
  inputValue: string;
  suggestions: string[];
  activeIndex: number;        // -1 = no selection; 0+ = highlighted item
  isOpen: boolean;
  isLoading: boolean;
};

type TypeaheadAction =
  | { type: 'SET_INPUT'; value: string }
  | { type: 'SET_SUGGESTIONS'; suggestions: string[] }
  | { type: 'ARROW_DOWN' }
  | { type: 'ARROW_UP' }
  | { type: 'SELECT' }
  | { type: 'ESCAPE' }
  | { type: 'SET_LOADING'; isLoading: boolean };

function typeaheadReducer(state: TypeaheadState, action: TypeaheadAction): TypeaheadState {
  switch (action.type) {
    case 'SET_INPUT':
      return {
        ...state,
        inputValue: action.value,
        activeIndex: -1,             // reset highlight on new input
        isOpen: action.value.length > 0,
      };

    case 'SET_SUGGESTIONS':
      return {
        ...state,
        suggestions: action.suggestions,
        activeIndex: -1,
        isOpen: action.suggestions.length > 0,
        isLoading: false,
      };

    case 'ARROW_DOWN':
      if (!state.isOpen || state.suggestions.length === 0) return state;
      return {
        ...state,
        activeIndex: (state.activeIndex + 1) % state.suggestions.length,
      };

    case 'ARROW_UP':
      if (!state.isOpen || state.suggestions.length === 0) return state;
      return {
        ...state,
        activeIndex:
          state.activeIndex <= 0
            ? state.suggestions.length - 1
            : state.activeIndex - 1,
      };

    case 'SELECT': {
      if (state.activeIndex < 0) return { ...state, isOpen: false }; // submit raw input
      const selected = state.suggestions[state.activeIndex];
      return {
        ...state,
        inputValue: selected,
        isOpen: false,
        activeIndex: -1,
      };
    }

    case 'ESCAPE':
      return { ...state, isOpen: false, activeIndex: -1 };

    case 'SET_LOADING':
      return { ...state, isLoading: action.isLoading };

    default:
      return state;
  }
}
```

##### Key Handler

```tsx
function handleKeyDown(event: React.KeyboardEvent) {
  switch (event.key) {
    case 'ArrowDown':
      event.preventDefault();  // prevent cursor from moving to end of input
      dispatch({ type: 'ARROW_DOWN' });
      break;

    case 'ArrowUp':
      event.preventDefault();
      dispatch({ type: 'ARROW_UP' });
      break;

    case 'Enter':
      event.preventDefault();
      dispatch({ type: 'SELECT' });
      onSearch(
        state.activeIndex >= 0
          ? state.suggestions[state.activeIndex]
          : state.inputValue
      );
      break;

    case 'Escape':
      dispatch({ type: 'ESCAPE' });
      break;
  }
}
```

##### Why `event.preventDefault()` on Arrow Keys?

Without it, pressing ↓ in a text input moves the cursor to the **end** of the text, and ↑ moves it to the **start**. This conflicts with our intended behavior (navigating suggestions). `preventDefault()` stops the browser's default cursor movement.

##### Visual Feedback for Active Index

```tsx
<li
  role="option"
  aria-selected={index === state.activeIndex}
  className={index === state.activeIndex ? 'suggestion-active' : ''}
  onMouseEnter={() => dispatch({ type: 'SET_ACTIVE', index })}
>
  <HighlightedSuggestion suggestion={item} prefix={state.inputValue} />
</li>
```

**Why `onMouseEnter` updates `activeIndex`:**
*   When the user switches from keyboard to mouse mid-interaction, the highlight should follow the mouse (not stay stuck on the keyboard-selected item).
*   This is the behavior users expect from native `<select>` and all modern autocomplete implementations (VS Code, Google Search, Slack).

---

#### 5.2.7 Dropdown Positioning and Portal Rendering

##### The Problem — Dropdown Clipping

If the typeahead lives inside a container with `overflow: hidden` or a modal with `overflow: auto`, the dropdown gets clipped:

```
┌── Modal ─────────────────────┐
│  ┌── SearchInput ──────────┐ │
│  │ [how_______________]    │ │
│  │ ┌────────────────────┐  │ │
│  │ │ how to cook rice   │  │ │  ← dropdown is INSIDE the modal
│  │ │ how to lose weight │  │ │     if modal has overflow: hidden,
│  │ └───────────────── ──┘  │ │     dropdown is clipped
│  └─────────────────────────┘ │
└──────────────────────────────┘
```

##### Solution — Portal Rendering

Render the dropdown in a React **Portal** attached to `document.body`. This takes it out of the parent's overflow context:

```tsx
function SuggestionDropdown({ anchorRef, children, isOpen }: Props) {
  if (!isOpen) return null;

  const anchorRect = anchorRef.current?.getBoundingClientRect();
  if (!anchorRect) return null;

  return ReactDOM.createPortal(
    <ul
      role="listbox"
      style={{
        position: 'fixed',
        top: anchorRect.bottom + 4,       // 4px gap below input
        left: anchorRect.left,
        width: anchorRect.width,          // match input width
        maxHeight: '300px',
        overflowY: 'auto',
        zIndex: 9999,
      }}
    >
      {children}
    </ul>,
    document.body
  );
}
```

##### When Portal Is Needed vs Not

| Scenario | Portal needed? | Why |
|----------|---------------|-----|
| Typeahead in the main page (no overflow containers) | **No** | Dropdown flows naturally below input |
| Typeahead inside a modal or dialog | **Yes** | Modal's `overflow: hidden` would clip dropdown |
| Typeahead in a fixed navbar | **Yes** | Navbar may have `overflow: hidden` or limited height |
| Typeahead in a table cell | **Yes** | Table cells clip overflow content |

##### Advanced Positioning with Floating UI

For production-grade positioning (flip when near viewport edge, scroll tracking), use a library like **Floating UI** (successor to Popper.js):

```tsx
import { useFloating, offset, flip, shift } from '@floating-ui/react';

function TypeaheadWithFloating() {
  const { refs, floatingStyles } = useFloating({
    placement: 'bottom-start',
    middleware: [
      offset(4),             // 4px gap
      flip(),                // flip to top if no space below
      shift({ padding: 8 }), // keep within viewport with 8px padding
    ],
  });

  return (
    <>
      <input ref={refs.setReference} />
      {isOpen && (
        <ul ref={refs.setFloating} style={floatingStyles} role="listbox">
          {/* suggestions */}
        </ul>
      )}
    </>
  );
}
```

---

#### 5.2.8 Mobile Specific Considerations

| Concern | Desktop Behavior | Mobile Adaptation |
|---------|-----------------|-------------------|
| **Debounce delay** | 100–150ms | 200–300ms (touch typing is slower) |
| **Suggestion count** | 8–10 items | 3–5 items (limited vertical space) |
| **Dropdown height** | 300-400px max | 50% of viewport height max |
| **Touch targets** | N/A | Min 44px height per suggestion item (Apple HIG) |
| **Keyboard interaction** | Arrow keys + Enter | Tap to select; no arrow key navigation |
| **Layout shift** | Dropdown overlays content | Dropdown pushes content down (avoids covering the keyboard) |
| **Focus behavior** | Input auto-focused on click | Input focus opens virtual keyboard — be cautious about when to auto-focus |

##### Mobile Dropdown Strategy

On mobile, the suggestion dropdown should **not** use `position: fixed` (it conflicts with the virtual keyboard). Instead, render it inline below the input:

```tsx
const isMobile = window.matchMedia('(max-width: 768px)').matches;

return (
  <div className="typeahead-container">
    <input /* ... */ />
    {isOpen && (
      <ul
        role="listbox"
        style={{
          // On mobile: inline position (flows with document)
          // On desktop: absolute/fixed overlay
          position: isMobile ? 'relative' : 'absolute',
          maxHeight: isMobile ? '40vh' : '300px',
        }}
      >
        {suggestions.map(/* ... */)}
      </ul>
    )}
  </div>
);
```

---

#### 5.2.9 Decision Matrix

| Scenario | Strategy | Implementation | Notes |
|----------|----------|---------------|-------|
| **Prevent request flooding** | Debounce (150ms desktop, 250ms mobile) | `useDebounce` hook | Adaptive delay based on prefix length |
| **Stale response handling** | AbortController + request versioning | `useRef` for controller + requestId | AbortController saves bandwidth; versioning as safety net |
| **Repeated prefix lookup** | Client-side LRU cache (50 entries, 5min TTL) | `SuggestionCache` class | Prefix hierarchy filtering for instant results |
| **Keyboard navigation** | useReducer with activeIndex | Arrow keys, Enter, Escape | `event.preventDefault()` on arrows |
| **Dropdown clipping** | Portal rendering (in modals/navbars) | `ReactDOM.createPortal` or Floating UI | Not needed for simple page-level typeahead |
| **Prefix highlighting** | Split string at match index + bold | `HighlightedSuggestion` component | Handles prefix and infix matches |
| **Mobile adaptation** | Larger debounce, fewer items, bigger targets | CSS media queries + responsive props | 44px min touch target height |
| **Empty input (focus)** | Show recent searches or trending | Conditional section in dropdown | Stored in localStorage |

---

### 5.3 Reusability Strategy

*   **`TypeaheadInput`**: The complete typeahead component — configurable via props: `onSearch`, `fetchSuggestions`, `debounceMs`, `maxSuggestions`, `placeholder`, `renderSuggestion`.
*   **`useDebounce`**: Generic debounce hook, reusable for any delayed value (search inputs, filter inputs, window resize handlers).
*   **`useTypeaheadFetch`**: Encapsulates AbortController + request versioning + caching logic. Reusable for any fetch-on-type pattern.
*   **`SuggestionDropdown`**: Generic dropdown with keyboard navigation, ARIA roles, and portal support. Reusable for command palettes, mention pickers, tag selectors.
*   **`HighlightedText`**: String highlighting utility, reusable for search result highlighting, text filtering UIs.
*   Design system tokens for input styling, dropdown shadows, active item highlight colors.

---

### 5.4 Module Organization

```
src/
 ├── features/
 │    └── typeahead/
 │         ├── components/
 │         │    ├── TypeaheadContainer.tsx
 │         │    ├── SearchInput.tsx
 │         │    ├── SuggestionDropdown.tsx
 │         │    ├── SuggestionItem.tsx
 │         │    ├── HighlightedSuggestion.tsx
 │         │    ├── RecentSearches.tsx
 │         │    └── TypeaheadSkeleton.tsx
 │         ├── hooks/
 │         │    ├── useTypeahead.ts
 │         │    ├── useTypeaheadFetch.ts
 │         │    ├── useDebounce.ts
 │         │    └── useKeyboardNavigation.ts
 │         ├── utils/
 │         │    ├── suggestionCache.ts
 │         │    ├── highlightMatch.ts
 │         │    └── typeaheadHelpers.ts
 │         ├── store/
 │         │    └── typeaheadReducer.ts
 │         └── types/
 │              └── typeahead.types.ts
 ├── shared/
 │    ├── components/ (Portal, Dropdown, Input, Button)
 │    ├── hooks/ (useDebounce, useClickOutside, useKeyHandler)
 │    └── utils/ (stringHighlight, localStorageHelper)
```

---

## 6. High Level Data Flow Explanation

### 6.1 Initial Load Flow

```
1. Page loads → TypeaheadContainer mounts
     ↓
2. Input renders (empty, unfocused)
     ↓
3. No API call on mount — typeahead is passive until user interaction
     ↓
4. (Optional) Prefetch trending suggestions in background
     for display on first focus
```

---

### 6.2 User Interaction Flow

```
User focuses input
     ↓
1. If input is empty → show recent searches (from localStorage)
     ↓
2. User types "how"
     ↓
3. Debounce timer starts (150ms)
     ↓
4. If user types another character within 150ms → timer resets
     ↓
5. Timer fires → check client cache
     ↓
6a. Cache HIT → render suggestions instantly (no API call)
     ↓
6b. Cache MISS → abort previous request → fire: GET /api/autocomplete?prefix=how
     ↓
7. Loading state active (optional skeleton in dropdown)
     ↓
8. API responds with suggestions
     ↓
9. Request versioning check: is this still the latest request?
     ↓
10a. YES → update state, cache results, render dropdown
10b. NO → discard (stale response)
     ↓
11. User navigates with ↓ key → activeIndex updates → visual highlight moves
     ↓
12. User presses Enter → selected suggestion populates input → onSearch fires
     ↓
13. Dropdown closes
```

---

### 6.3 Error and Retry Flow

*   **API request fails (network error)**: Hide dropdown. Do **not** show an error message for autocomplete — it is an enhancement, not a core feature. Let the user type and submit manually.
*   **API returns empty results**: Show "No suggestions found" text in dropdown (or hide dropdown entirely — team preference).
*   **Request times out (> 3s)**: Abort request. Hide loading state. User can still type and search.
*   **Multiple rapid errors**: After 3 consecutive failures, disable autocomplete for the session (prevent retry storms). Re-enable on next successful request.
*   **Offline**: Detect via `navigator.onLine`. Serve suggestions from client cache only. Show subtle "offline" indicator.

---

## 7. Data Modelling (Frontend Perspective)

### 7.1 Core Data Entities

*   **Suggestion** — a single autocomplete suggestion string with optional metadata.
*   **RecentSearch** — a query the user previously searched (stored locally).
*   **TrendingQuery** — a currently trending search query (fetched from backend).

---

### 7.2 Data Shape

```ts
type Suggestion = {
  text: string;                    // the full suggestion text
  type?: 'query' | 'recent' | 'trending' | 'product' | 'person';
  icon?: string;                   // optional icon URL or icon name
  metadata?: Record<string, unknown>; // extension point for rich suggestions
};

type AutocompleteResponse = {
  prefix: string;                  // echoed back from request
  suggestions: Suggestion[];
  source?: 'cache' | 'trie' | 'redis';  // backend cache hit info (optional)
};

type RecentSearch = {
  query: string;
  timestamp: number;               // for ordering by recency
};
```

---

### 7.3 Entity Relationships

*   **One-to-Many**: One prefix → many `Suggestion` items (the API returns top K).
*   **One-to-One**: One `RecentSearch` item → one query string (stored in localStorage).
*   No complex relationships — the typeahead data model is intentionally flat and simple.

---

### 7.4 UI Specific Data Models

```ts
type TypeaheadState = {
  inputValue: string;              // current text in the input
  suggestions: Suggestion[];       // current suggestion list
  activeIndex: number;             // -1 = none; 0+ = highlighted suggestion
  isOpen: boolean;                 // dropdown visibility
  isLoading: boolean;              // fetch in progress
  error: string | null;            // last error (rarely shown in UI)
  recentSearches: RecentSearch[];  // from localStorage
};

// Derived state
type DerivedState = {
  activeSuggestion: Suggestion | null;     // suggestions[activeIndex]
  hasResults: boolean;                     // suggestions.length > 0
  showRecents: boolean;                    // input empty + focused
  displayValue: string;                    // activeIndex >= 0 ? activeSuggestion.text : inputValue
};
```

---

## 8. State Management Strategy

### 8.1 State Classification

| State Type | Examples | Storage |
|---|---|---|
| **Component Local State** | Input value, suggestions, activeIndex, isOpen, isLoading | `useReducer` inside TypeaheadContainer |
| **Ephemeral / Request State** | Current AbortController, requestId ref, debounce timer | `useRef` (not in React state — avoids re-renders) |
| **Persistent User State** | Recent searches, user preferences | localStorage |
| **Derived / Computed State** | Active suggestion text, show recents flag, display value | Computed inline or via `useMemo` |

---

### 8.2 State Ownership

*   **TypeaheadContainer** owns all typeahead state via `useReducer`. It is the single source of truth for the component.
*   **SearchInput** is a controlled component — receives `value` and `onChange` from the container.
*   **SuggestionDropdown** receives `suggestions`, `activeIndex`, and `onSelect` as props — it is a pure rendering component.
*   **No global state** — the typeahead is self-contained. It communicates with the parent via callbacks (`onSearch`, `onSuggestionSelect`).

**Why `useReducer` over `useState`?**
*   The typeahead has **interrelated state transitions** — e.g., ARROW_DOWN must check `suggestions.length`, SELECT must read `activeIndex`, etc.
*   `useReducer` keeps all transition logic in one place (the reducer), making it testable and predictable.
*   `useState` would require multiple `set` calls per action, risking inconsistent intermediate states.

---

### 8.3 Persistence Strategy

| Data | Persistence | Reason |
|------|------------|--------|
| Suggestions | In-memory (SuggestionCache) | Ephemeral; cleared on page refresh; 5-min TTL |
| Input value | Component state (not persisted) | Cleared on navigation |
| Recent searches | localStorage | Survive sessions; user expects history to persist |
| Trending queries | In-memory (fetched once per session) | Refreshed on each session start |
| Active index | Component state | UI-only; never persisted |
| AbortController / requestId | useRef | Not React state (avoids re-renders) |

---

## 9. High Level API Design (Frontend POV)

### 9.1 Required APIs

| API | Method | Description |
|-----|--------|-------------|
| `/api/autocomplete` | **GET** | Fetch ranked suggestions for a prefix |
| `/api/search` | **GET** | Execute full search with a query (called on submit/select) |
| `/api/trending` | **GET** | Fetch trending/popular queries (for empty state) |

---

### 9.2 Request Lifecycle and Optimization (Deep Dive)

#### The Complete Request Timeline

```
User types 's' → 'sy' → 'sys' → 'syst' → 'syste' → 'system'

Time  0ms    80ms   160ms   240ms   320ms   400ms
Key:  [s]    [y]    [s]     [t]     [e]     [m]

                ┌──debounce──┐
Debounce timer: |reset|reset|reset|reset|reset|────┤ FIRE at 550ms
                                                   │
                                          Fetch: "system"

Without debounce: 6 API calls
With 150ms debounce: 1 API call ✅
```

**Savings calculation at scale:**
*   Without debounce: 6 requests × 100,000 concurrent users = **600,000 QPS**
*   With debounce: 1 request × 100,000 concurrent users = **100,000 QPS**
*   **83% reduction in backend load** from one simple optimization.

#### When to Skip the API Entirely

| Condition | Action |
|-----------|--------|
| Input is empty | Clear suggestions; show recents (no API call) |
| Input length is 1 character | Optionally skip (single-char results are too broad). Or use a longer debounce (300ms). |
| Same prefix already cached | Use cache immediately; skip API |
| Longer prefix of a cached shorter prefix | Filter cached results client-side; optionally background-fetch fresh results |
| Component is unmounting | Abort all in-flight requests |

#### Request Deduplication

If the debounce fires for prefix "sys" while a request for "sys" is already in flight (e.g., from a background refresh), deduplicate:

```tsx
const inFlightRequests = new Map<string, Promise<Suggestion[]>>();

async function fetchSuggestions(prefix: string): Promise<Suggestion[]> {
  // Return existing promise if a request for the same prefix is in flight
  if (inFlightRequests.has(prefix)) {
    return inFlightRequests.get(prefix)!;
  }

  const promise = fetch(`/api/autocomplete?prefix=${encodeURIComponent(prefix)}&k=10`)
    .then((res) => res.json())
    .then((data) => data.suggestions)
    .finally(() => inFlightRequests.delete(prefix));

  inFlightRequests.set(prefix, promise);
  return promise;
}
```

---

### 9.3 Request and Response Structure

**GET /api/autocomplete**

```json
// Request
// GET /api/autocomplete?prefix=how&k=5&locale=en-IN

// Response
{
  "prefix": "how",
  "suggestions": [
    { "text": "how to cook rice", "type": "query" },
    { "text": "how to lose weight", "type": "query" },
    { "text": "how to make tea", "type": "query" },
    { "text": "how are you", "type": "trending" },
    { "text": "howard stern show", "type": "query" }
  ],
  "source": "redis"
}
```

**GET /api/trending**

```json
// Response
{
  "trending": [
    { "text": "IPL 2026 schedule", "type": "trending" },
    { "text": "budget 2026", "type": "trending" },
    { "text": "weather today", "type": "trending" }
  ]
}
```

---

### 9.4 Error Handling and Status Codes

| Status | Scenario | Frontend Handling |
|--------|----------|-------------------|
| `200` | Success | Render suggestions |
| `204` | No suggestions found | Show empty state or hide dropdown |
| `304` | Not Modified (HTTP cache) | Use cached response |
| `400` | Bad request (invalid prefix) | Silently ignore; hide dropdown |
| `401` | Unauthorized | Ignore for autocomplete (it works for anon users too) |
| `429` | Rate limited | Back off; disable autocomplete temporarily; increase debounce |
| `500` | Server error | Hide dropdown; don't show error — autocomplete is non-critical |
| `503` | Service unavailable | Serve from client cache only; retry with exponential backoff |

---

## 10. Caching Strategy

### 10.1 What to Cache

*   **Autocomplete responses**: Cached per prefix (key = normalized prefix string).
*   **Trending queries**: Cached once per session or with 15-min TTL.
*   **Recent searches**: Stored in localStorage (persistent across sessions).
*   **Static assets**: JS bundle for the typeahead component (immutable, content-hashed).

### 10.2 Where to Cache

| Data | Cache Location | TTL |
|------|----------------|-----|
| Suggestion responses | Client-side LRU cache (in-memory) | 5 min |
| Suggestion responses (HTTP) | Browser HTTP cache via `Cache-Control` headers | Varies (backend-controlled; typically 1-10 min) |
| Trending queries | In-memory (fetched once) | 15 min |
| Recent searches | localStorage | No expiry (user-managed) |
| Typeahead JS bundle | Browser HTTP cache + CDN | 1 year (immutable, content-hashed) |

### 10.3 Cache Invalidation

*   **Client-side LRU cache**: Entries expire after 5 min (TTL-based). Entire cache cleared on session end.
*   **HTTP cache**: Backend sets `Cache-Control: public, max-age=60, stale-while-revalidate=300` — browser serves cached response for 1 min, then revalidates in background for up to 5 min.
*   **Recent searches**: User-managed (explicit delete). Or automatically prune entries older than 30 days.
*   **Trending queries**: Refetched on session start and every 15 min via `setTimeout`.

---

## 11. CDN and Asset Optimization

*   **Typeahead component bundle**: Code-split and served via CDN with immutable caching. ~15-30KB (including debounce, cache, keyboard handler).
*   **No image assets**: Typeahead is purely text-based; no images to optimize (unless suggestion items include thumbnails — out of scope).
*   **API proximity**: Autocomplete API should be served from edge/regional servers (not a single origin). Redis caching at the edge reduces round-trip latency.
*   **Preconnect**: `<link rel="preconnect" href="https://api.example.com">` to pre-establish the connection before the first typeahead request.
*   **Compression**: API responses are small (< 1KB JSON for 10 suggestions). Brotli/Gzip compression still helps — reduces to ~300-500 bytes.

---

## 12. Rendering Strategy

*   **Typeahead component**: Fully **CSR** — no SSR benefit for an interactive input component.
*   **Initial render**: The `<input>` element is part of the SSR'd page shell (for fast FCP). The typeahead logic (debounce, fetch, dropdown) hydrates on client.
*   **Dropdown rendering**: Rendered conditionally — only when `isOpen` is true. Uses React Portal when inside a modal/overlay.
*   **Lazy loading**: If the typeahead is behind a user action (e.g., clicking a search icon), the entire typeahead module can be lazy loaded: `React.lazy(() => import('./TypeaheadContainer'))`.
*   **Suggestion items**: `React.memo` on `SuggestionItem` — prevents re-renders when `activeIndex` changes for a non-active item (only the previously-active and newly-active items re-render).

---

## 13. Cross Cutting Non Functional Concerns

### 13.1 Security

*   **XSS mitigation**: All suggestion text is rendered as **text content**, never as raw HTML. Using React's JSX (`{suggestion.text}`) auto-escapes HTML entities. If using `dangerouslySetInnerHTML` for highlighting, wrap with DOMPurify.
*   **Input sanitization**: Normalize input before sending to API (trim whitespace, remove control characters). The backend should also sanitize.
*   **No sensitive data in URL**: The prefix is sent as a query parameter (`/api/autocomplete?prefix=how`). If prefixes could contain sensitive data, use POST with a request body instead.
*   **CSRF**: Autocomplete is a GET request (read-only) — CSRF is typically not a concern. If using POST for sensitive contexts, include a CSRF token.
*   **Rate limiting**: Client-side debounce is the first line of defense. Backend enforces authoritative rate limits (429 response).

---

### 13.2 Accessibility

*   **ARIA Combobox Pattern**:
    *   Input: `role="combobox"`, `aria-expanded="true/false"`, `aria-controls="suggestion-listbox"`, `aria-activedescendant="suggestion-{activeIndex}"`.
    *   Dropdown: `role="listbox"`, `id="suggestion-listbox"`.
    *   Each suggestion item: `role="option"`, `id="suggestion-{index}"`, `aria-selected="true/false"`.

```tsx
<input
  role="combobox"
  aria-expanded={isOpen}
  aria-controls="suggestion-listbox"
  aria-activedescendant={activeIndex >= 0 ? `suggestion-${activeIndex}` : undefined}
  aria-autocomplete="list"
  aria-haspopup="listbox"
/>

<ul role="listbox" id="suggestion-listbox">
  {suggestions.map((item, index) => (
    <li
      key={index}
      role="option"
      id={`suggestion-${index}`}
      aria-selected={index === activeIndex}
    >
      {item.text}
    </li>
  ))}
</ul>
```

*   **Screen reader announcements**:
    *   When suggestions load: announce "{count} suggestions available" via an `aria-live="polite"` region.
    *   On arrow key navigation: `aria-activedescendant` updates tell the screen reader which option is active.
    *   On selection: announce "Selected: {suggestion text}".

```tsx
<div aria-live="polite" className="sr-only">
  {suggestions.length > 0
    ? `${suggestions.length} suggestions available`
    : isLoading
      ? 'Loading suggestions...'
      : 'No suggestions'}
</div>
```

*   **Keyboard support**: ↑/↓ navigate, Enter selects, Escape closes, Tab moves to next focusable element.
*   **Focus management**: Dropdown opens on input focus; closes on blur (with a small delay to allow click-on-suggestion to register first).
*   **High contrast**: Ensure active suggestion highlight meets WCAG AA contrast ratios (4.5:1 for text).

---

### 13.3 Performance Optimization

#### Suggestion Latency Budget

```
Target: < 200ms from keystroke to visible suggestions

  Keystroke                  → 0ms
  Debounce wait              → 150ms
  API call + response        → 30-50ms
  React render + DOM update  → 5-10ms
  ─────────────────────────────────
  Total:                       185-210ms ✅
```

*   **Debounce** eliminates 80-90% of API calls.
*   **AbortController** frees browser connections and prevents wasted work.
*   **Client cache** eliminates API calls for repeated prefixes (instant: < 5ms).
*   **Prefix hierarchy filtering** provides instant suggestions while the API is in flight.

#### Render Optimization

*   **`React.memo` on SuggestionItem**: Only 2 items re-render on arrow key navigation (the previously active item and the newly active item), not all items.
*   **`useMemo` on highlighted text computation**: Avoid recalculating string splits on every render.
*   **Conditional dropdown mount**: Dropdown is not in the DOM when `isOpen` is false. No hidden rendering.
*   **Virtualize long suggestion lists**: If suggestions exceed ~20 items (unusual for typeahead but possible for command palettes), use a virtual list.

#### Network Optimization

*   **HTTP/2 multiplexing**: Multiple typeahead requests share a single connection (in case abort doesn't fully cancel within the browser pipeline).
*   **`Cache-Control: public, max-age=60`**: Popular prefixes like "a", "th", "how" can be cached at the CDN/browser level.
*   **Connection prewarming**: `<link rel="preconnect">` to the autocomplete API domain reduces first-request latency by ~100ms (DNS + TCP + TLS).

---

### 13.4 Observability and Reliability

*   **Analytics events**:
    *   **Suggestion Impression**: Track when a suggestion dropdown is shown (with the prefix and suggestion list).
    *   **Suggestion Selection**: Track when a user selects a suggestion (which position/index, the prefix, the selected text).
    *   **Manual Search**: Track when a user submits without selecting a suggestion (their raw query).
    *   **Abandonment**: Track when a user types, sees suggestions, but neither selects nor searches (potential UX issue).
    *   **Latency**: Track end-to-end latency (keystroke → suggestions visible) at p50, p90, p99.
*   **Error monitoring**:
    *   Log API failures (non-AbortError) with prefix, status code, and response time.
    *   Track client cache hit rate (ideally > 30% for active sessions).
*   **Feature flags**: Gate new features behind flags:
    *   Trending suggestions, rich suggestion cards, new highlighting algorithm.
    *   A/B test debounce intervals, suggestion count, and cache TTL.
*   **Graceful degradation**:
    *   If autocomplete API is down → disable dropdown, allow manual search.
    *   If JavaScript fails → input still works as a plain HTML form input.
    *   If client cache is corrupted → clear cache, fetch fresh.

---

## 14. Edge Cases and Tradeoffs

| Edge Case | Handling |
|-----------|---------|
| **User types extremely fast (< 50ms between keys)** | Debounce handles this — only the final prefix triggers a request. AbortController cancels any intermediate requests that slipped through. |
| **User pastes a long query** | Treat as a single input change → debounce fires once with the full pasted text. |
| **User presses Enter before suggestions load** | Submit the raw input value as the search query. Don't wait for suggestions. |
| **Network is very slow (> 2s response time)** | Show a subtle loading indicator in the dropdown. If timeout reached (3s), hide dropdown silently. |
| **Backend returns stale suggestions (ranked based on old data)** | Not a frontend concern — but the client cache TTL (5 min) limits how long stale data is shown. |
| **Input contains special characters (quotes, ampersands, unicode)** | URL-encode the prefix with `encodeURIComponent`. Backend handles decoding. |
| **User navigates away mid-request** | `useEffect` cleanup aborts in-flight requests. No memory leaks, no state updates on unmounted components. |
| **Dropdown is open + user clicks outside** | Close dropdown on `mousedown` outside the component. Use `useClickOutside` hook. |
| **User clicks a suggestion while keyboard has a different item active** | Mouse `onClick` takes priority. `onMouseEnter` updates `activeIndex` in real time, so the clicked item is always the visually active one. |
| **RTL language input** | Input `dir="auto"` detects text direction. Dropdown items inherit the direction. |
| **Empty API response after valid prefix** | Show "No results for '{prefix}'" in dropdown, or hide dropdown entirely. |

### Key Tradeoffs

| Decision | Tradeoff |
|----------|----------|
| **Debounce over throttle** | Debounce waits for typing to stop before firing. Throttle fires at intervals during typing. Debounce is better for autocomplete because we want the **most complete prefix**, not periodic intermediate prefixes. But it means the very first keystroke has a forced delay. |
| **AbortController over ignore-stale** | AbortController saves bandwidth (cancelled requests don't transfer data). But it adds code complexity and requires careful `AbortError` handling. |
| **Client-side cache** | Reduces API calls for repeated/similar prefixes. But cached results may be slightly stale (mitigated by 5-min TTL). |
| **useReducer over useState** | Centralized, testable state transitions. But more boilerplate than multiple `useState` calls. |
| **Portal rendering for dropdown** | Avoids clipping issues in modals/overlays. But requires manual positioning and scroll-tracking logic. |
| **ARIA combobox pattern** | Full screen reader support. But adds many ARIA attributes and requires careful ID management. |
| **Inline dropdown on mobile** | Avoids virtual keyboard conflicts. But may push page content down and cause layout shift. |
| **Prefix hierarchy filtering (optimistic)** | Instant results from cached parent prefix. But may miss new top-K results that the server would return. Mitigated by background fetch. |

---

## 15. Summary and Future Improvements

### Key Architectural Decisions

1.  **Debounced requests (150ms desktop, 250ms mobile)** — eliminates 80-90% of API calls without perceptible delay.
2.  **AbortController + request versioning** — prevents race conditions and saves bandwidth by cancelling stale in-flight requests.
3.  **Client-side LRU cache with TTL** — instant suggestions for repeated prefixes; prefix hierarchy filtering for progressive refinement.
4.  **useReducer for state management** — centralized, predictable state transitions for input, suggestions, active index, and dropdown visibility.
5.  **ARIA combobox pattern** — full keyboard and screen reader accessibility.
6.  **Graceful degradation** — autocomplete is an enhancement; failures never block core search functionality.

### Possible Future Enhancements

*   **Rich suggestion cards**: Show product images, person avatars, or category icons alongside text suggestions. Requires extending the `Suggestion` type and rendering logic.
*   **Fuzzy matching with typo tolerance**: Highlight matching characters individually (not just prefix). Requires a more sophisticated highlight algorithm.
*   **Voice input integration**: Allow microphone input that feeds into the typeahead. Show interim transcription results as the "prefix".
*   **Multi-category suggestions**: Group suggestions by type (queries, products, people, pages) with section headers in the dropdown.
*   **Offline autocomplete**: Use a small client-side trie (Web Worker) for basic prefix matching when offline. Sync with server on reconnect.
*   **Personalized suggestions**: Include user-specific signals (recent searches, purchase history) in the API request for personalized ranking.
*   **Visual search integration**: "Search by image" button in the typeahead that opens a camera/upload flow.
*   **Web Worker for cache operations**: Offload LRU cache management and prefix filtering to a Web Worker to keep the main thread free.

---

### Endpoint Summary

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/autocomplete` | GET | Fetch ranked suggestions for a prefix |
| `/api/search` | GET | Execute full search with a query |
| `/api/trending` | GET | Fetch trending queries (for empty state) |

---

### Complete Typeahead Flow

| Direction | Mechanism | Trigger | Endpoint | Action |
|-----------|-----------|---------|----------|--------|
| Initial Load | CSR | Page mount | — | Render empty input; no API call |
| Focus (empty input) | localStorage | Input focus | — | Show recent searches (local) |
| Trending (optional) | REST | First focus | `GET /api/trending` | Show trending suggestions |
| Typing | Debounce | Keystroke + 150ms pause | — | Debounce timer starts/resets |
| Fetch Suggestions | REST + Abort | Debounce fires | `GET /api/autocomplete?prefix=X` | Cancel previous; fetch new |
| Cache Hit | Client Cache | Debounce fires | — | Serve from LRU cache (no API) |
| Arrow Key Navigation | Client State | ArrowDown / ArrowUp | — | Update activeIndex in reducer |
| Selection | Callback | Enter or Click | `GET /api/search?q=X` | Populate input; execute search |
| Escape / Blur | Client State | Escape key or blur | — | Close dropdown |
| Clear Input | Client State | Backspace or clear button | — | Clear suggestions; show recents |

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design
