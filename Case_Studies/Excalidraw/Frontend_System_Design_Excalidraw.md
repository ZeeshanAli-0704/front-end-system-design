# Frontend System Design: Excalidraw (Collaborative Whiteboard)

- [Frontend System Design: Excalidraw (Collaborative Whiteboard)](#frontend-system-design-excalidraw-collaborative-whiteboard)
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
    - [5.2 Canvas Rendering Engine (Deep Dive)](#52-canvas-rendering-engine-deep-dive)
      - [5.2.1 Why HTML5 Canvas Over SVG](#521-why-html5-canvas-over-svg)
      - [5.2.2 Rendering Pipeline](#522-rendering-pipeline)
      - [5.2.3 Dirty Rectangle Optimization](#523-dirty-rectangle-optimization)
      - [5.2.4 Multi Layer Canvas Architecture](#524-multi-layer-canvas-architecture)
      - [5.2.5 Hand Drawn (Rough) Style Rendering](#525-hand-drawn-rough-style-rendering)
      - [5.2.6 Hit Testing and Element Selection](#526-hit-testing-and-element-selection)
      - [5.2.7 Zoom and Pan with Infinite Canvas](#527-zoom-and-pan-with-infinite-canvas)
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
  - [9. IndexedDB Deep Dive (Local First Storage)](#9-indexeddb-deep-dive-local-first-storage)
    - [9.1 Why IndexedDB for a Drawing App](#91-why-indexeddb-for-a-drawing-app)
    - [9.2 IndexedDB Schema Design](#92-indexeddb-schema-design)
    - [9.3 CRUD Operations on Scenes](#93-crud-operations-on-scenes)
    - [9.4 Auto Save Strategy](#94-auto-save-strategy)
    - [9.5 Image and Asset Blob Storage](#95-image-and-asset-blob-storage)
    - [9.6 Versioning and Migration](#96-versioning-and-migration)
    - [9.7 Storage Quota and Cleanup](#97-storage-quota-and-cleanup)
    - [9.8 IndexedDB vs Alternatives Comparison](#98-indexeddb-vs-alternatives-comparison)
  - [10. Undo Redo System (Deep Dive)](#10-undo-redo-system-deep-dive)
    - [10.1 Command Pattern Architecture](#101-command-pattern-architecture)
    - [10.2 History Stack Implementation](#102-history-stack-implementation)
  - [11. Real Time Collaboration (Deep Dive)](#11-real-time-collaboration-deep-dive)
    - [11.1 Collaboration Architecture Overview](#111-collaboration-architecture-overview)
    - [11.2 Conflict Resolution with CRDTs](#112-conflict-resolution-with-crdts)
    - [11.3 Cursor and Presence Awareness](#113-cursor-and-presence-awareness)
    - [11.4 Room and Session Management](#114-room-and-session-management)
  - [12. High Level API Design (Frontend POV)](#12-high-level-api-design-frontend-pov)
    - [12.1 Required APIs](#121-required-apis)
    - [12.2 Request and Response Structure](#122-request-and-response-structure)
    - [12.3 Error Handling and Status Codes](#123-error-handling-and-status-codes)
  - [13. Caching Strategy](#13-caching-strategy)
  - [14. CDN and Asset Optimization](#14-cdn-and-asset-optimization)
  - [15. Rendering Strategy](#15-rendering-strategy)
  - [16. Cross Cutting Non Functional Concerns](#16-cross-cutting-non-functional-concerns)
    - [16.1 Security](#161-security)
    - [16.2 Accessibility](#162-accessibility)
    - [16.3 Performance Optimization](#163-performance-optimization)
    - [16.4 Observability and Reliability](#164-observability-and-reliability)
  - [17. Edge Cases and Tradeoffs](#17-edge-cases-and-tradeoffs)
  - [18. Summary and Future Improvements](#18-summary-and-future-improvements)

---

## 1. Concept, Idea, Product Overview

### 1.1 Product Description

*   Excalidraw is an open-source, browser-based collaborative whiteboard application with a hand-drawn, sketch-like aesthetic.
*   It is a **local-first** application — drawings are stored in the browser (IndexedDB) by default, with optional cloud sync and real-time collaboration.
*   Target users: engineers, designers, product managers, educators, and anyone who needs quick visual communication — architecture diagrams, wireframes, flowcharts, mind maps, or freeform sketches.
*   Primary use case: rapidly create and share visual diagrams that look informal and approachable, with optional real-time multi-user collaboration.

---

### 1.2 Key User Personas

*   **Solo Sketcher**: Creates quick diagrams locally without signing in. Expects zero-friction startup, offline support, and instant save (no "save" button — everything auto-persists to the browser).
*   **Collaborative Team Member**: Joins a shared room via link. Edits simultaneously with 2-10 collaborators. Expects real-time cursor visibility, conflict-free concurrent edits, and smooth sync.
*   **Presenter / Educator**: Uses the whiteboard during live presentations or teaching sessions. Needs pan/zoom, laser pointer mode, and the ability to export diagrams as PNG/SVG.

---

### 1.3 Core User Flows (High Level)

*   **Solo Drawing (Primary Flow)**:
    1.  User opens `excalidraw.com` → app loads instantly (no sign-in required).
    2.  Previous drawing is auto-restored from IndexedDB → user continues where they left off.
    3.  User selects a tool (rectangle, arrow, text) → draws on the infinite canvas.
    4.  Every change is auto-saved to IndexedDB within 300ms (debounced).
    5.  User exports as PNG/SVG or copies a shareable link.

*   **Real Time Collaboration (Secondary Flow)**:
    1.  User clicks "Live collaboration" → a unique room ID is generated.
    2.  User shares the room link with collaborators.
    3.  Collaborators join → their cursors appear in real-time.
    4.  All participants draw simultaneously → changes sync via WebSocket.
    5.  Conflicts are resolved automatically using CRDTs (last-writer-wins per element).

*   **File Management (Secondary Flow)**:
    1.  User can save scene as `.excalidraw` JSON file to disk.
    2.  User can open/import `.excalidraw`, `.png`, or `.svg` files.
    3.  User can load a library of reusable shapes and components.

---

## 2. Requirements

### 2.1 Functional Requirements

*   **Drawing Tools**:
    *   Primitive shapes: rectangle, ellipse, diamond, line, arrow, freehand draw.
    *   Text tool with inline editing directly on canvas.
    *   Image embedding (drag-and-drop or paste from clipboard).
    *   Sticky note / frame grouping.
*   **Canvas Interactions**:
    *   Infinite canvas with pan (hand tool or scroll) and zoom (pinch or Ctrl+scroll).
    *   Select, move, resize, rotate, group, and align elements.
    *   Snap-to-grid and smart alignment guides.
    *   Multi-select with drag box or Shift+click.
*   **Undo / Redo**: Full undo/redo stack with Ctrl+Z / Ctrl+Shift+Z. Supports complex operations (group move, paste, delete multiple).
*   **Styling**:
    *   Stroke color, background fill, stroke width, stroke style (solid, dashed, dotted).
    *   Font family (hand-drawn, normal, code), font size.
    *   Element opacity and layer ordering (bring to front, send to back).
*   **Persistence**:
    *   Auto-save to IndexedDB (local browser storage) — no explicit save action needed.
    *   Export to `.excalidraw` (JSON), PNG, SVG, clipboard.
    *   Import from `.excalidraw` file or paste image.
*   **Collaboration**:
    *   Real-time multi-user editing via shareable room link.
    *   Live cursor positions and user names visible.
    *   End-to-end encryption for shared sessions.
*   **Libraries**:
    *   Load reusable shape libraries (`.excalidrawlib` files).
    *   Drag shapes from the library panel onto the canvas.

---

### 2.2 Non Functional Requirements

*   **Performance**: Canvas must render at 60fps during pan/zoom/draw even with 1,000+ elements. Initial app load < 2s (no backend dependency for solo mode). Auto-save debounce < 300ms (user never loses more than 300ms of work).
*   **Scalability**: Support scenes with 5,000+ elements. Support collaboration rooms with up to 20 simultaneous users. IndexedDB storage for scenes up to 50MB (including embedded images).
*   **Availability**: Fully functional offline (local-first architecture). Collaboration requires network but gracefully degrades — user can continue drawing offline, and changes sync when reconnected.
*   **Security**: End-to-end encryption (E2EE) for collaborative sessions — the server never sees unencrypted scene data. No user data stored server-side in solo mode.
*   **Accessibility**: Keyboard shortcuts for all tools and actions. Screen reader announcements for tool changes and mode switches. High-contrast UI controls. Respects `prefers-reduced-motion` for canvas animations.
*   **Device Support**: Desktop web (primary), tablet web (with touch/stylus support), mobile web (basic viewing). Chrome, Firefox, Safari, Edge.
*   **i18n**: UI translated into 20+ languages. RTL layout support for Arabic/Hebrew UI panels (canvas itself is direction-agnostic).

---

## 3. Scope Clarification (Interview Scoping)

### 3.1 In Scope

*   Canvas rendering engine design (HTML5 Canvas 2D).
*   IndexedDB-based local persistence (deep dive).
*   Element data model and scene state management.
*   Undo/redo system with command pattern.
*   Real-time collaboration architecture (WebSocket + CRDTs).
*   Export/import pipeline (PNG, SVG, JSON).
*   Performance optimization for large canvases.

---

### 3.2 Out of Scope

*   Backend infrastructure (WebSocket server scaling, database).
*   Rich text editing within text elements (font ligatures, markdown rendering).
*   Mobile native app (React Native / Flutter).
*   Detailed encryption implementation (E2EE algorithm specifics).
*   Library marketplace / sharing platform.
*   User authentication and account management.

---

### 3.3 Assumptions

*   The application is a **SPA** that runs entirely in the browser for solo mode.
*   No user sign-in is required for local drawing (zero-friction onboarding).
*   The browser supports Canvas 2D API, IndexedDB, WebSocket, and Web Crypto API.
*   For collaboration, a lightweight WebSocket relay server exists (we focus on the frontend).
*   Image assets embedded in drawings are stored as base64 or Blob data in IndexedDB.

---

## 4. High Level Frontend Architecture

### 4.1 Overall Approach

*   **SPA** (Single Page Application) — the entire app runs client-side. No server rendering needed.
*   **CSR only** — the app shell is a static HTML page; all drawing logic is client-side JavaScript.
*   **Local-first architecture** — IndexedDB is the primary data store. Cloud sync (collaboration) is an optional overlay.
*   The app is a **monolith frontend** — no micro-frontends. The codebase is modular but ships as a single bundle (with code-splitting for collaboration and export features).

---

### 4.2 Major Architectural Layers

```
┌──────────────────────────────────────────────────────────────┐
│  UI Layer                                                    │
│  ┌───────────────────────┐  ┌─────────────────────────────┐  │
│  │ Toolbar               │  │ Canvas (HTML5 Canvas 2D)    │  │
│  │ (tools, colors, style)│  │  ┌─────────────────────┐    │  │
│  ├───────────────────────┤  │  │ Static Layer        │    │  │
│  │ Sidebar               │  │  │ (committed elements)│    │  │
│  │ (layers, libraries)   │  │  ├─────────────────────┤    │  │
│  ├───────────────────────┤  │  │ Interactive Layer   │    │  │
│  │ Properties Panel      │  │  │ (in-progress drawing│    │  │
│  │ (element settings)    │  │  │  selection handles, │    │  │
│  ├───────────────────────┤  │  │  cursors)           │    │  │
│  │ Context Menu          │  │  └─────────────────────┘    │  │
│  │ (right-click actions) │  └─────────────────────────────┘  │
│  └───────────────────────┘                                   │
├──────────────────────────────────────────────────────────────┤
│  State Management Layer                                      │
│  (Scene State, App State, History Stack,                     │
│   Collaboration State, UI Preferences)                       │
├──────────────────────────────────────────────────────────────┤
│  Persistence Layer                                           │
│  (IndexedDB Store, File Import/Export,                       │
│   Auto-save Manager, Blob Storage)                           │
├──────────────────────────────────────────────────────────────┤
│  Collaboration Layer                                         │
│  (WebSocket Client, CRDT Engine,                             │
│   Presence Manager, E2E Encryption)                          │
├──────────────────────────────────────────────────────────────┤
│  Rendering Engine                                            │
│  (Canvas Renderer, Hit Testing, Rough.js Integration,        │
│   Viewport Transform, Export Renderer)                       │
├──────────────────────────────────────────────────────────────┤
│  Shared / Utility Layer                                      │
│  (Math utils, Geometry helpers, Keyboard manager,            │
│   Clipboard handler, Analytics tracker)                      │
└──────────────────────────────────────────────────────────────┘
```

---

### 4.3 External Integrations

*   **Rough.js**: Library for rendering shapes with a hand-drawn/sketchy appearance on Canvas 2D.
*   **WebSocket Relay Server**: Lightweight server that relays scene diffs between collaborators in a room. Does not store data (E2EE means server only sees encrypted blobs).
*   **Web Crypto API**: Browser-native API for AES-GCM encryption/decryption of scene data during collaboration.
*   **CDN**: Serves static app assets (JS bundles, fonts, default libraries). Drawing data is never on the CDN.
*   **File System Access API** (optional): For native file save/open dialogs in supported browsers (Chrome).

---

## 5. Component Design and Modularization

### 5.1 Component Hierarchy

```
App
 ├── Toolbar
 │    ├── ToolSelector (selection, rectangle, ellipse, diamond, arrow, line, freehand, text, image, eraser)
 │    ├── ActionButtons (undo, redo, clear, export, collaboration)
 │    └── ZoomControls (zoom in, zoom out, fit to screen, reset)
 │
 ├── PropertiesPanel (context-sensitive)
 │    ├── StrokeColorPicker
 │    ├── BackgroundColorPicker
 │    ├── StrokeWidthSelector
 │    ├── StrokeStyleSelector (solid, dashed, dotted)
 │    ├── FontFamilySelector
 │    ├── FontSizeSelector
 │    ├── OpacitySlider
 │    └── LayerActions (bring front, send back, group, ungroup)
 │
 ├── CanvasWrapper
 │    ├── StaticCanvas (renders committed elements)
 │    ├── InteractiveCanvas (renders in-progress drawing, selection box, resize handles, collaborator cursors)
 │    └── TextEditor (floating <textarea> positioned over text element being edited)
 │
 ├── Sidebar
 │    ├── LibraryPanel (reusable shapes)
 │    └── LayersPanel (z-order management)
 │
 ├── CollaborationOverlay
 │    ├── UserAvatars (collaborator list)
 │    └── ShareLinkModal
 │
 ├── ContextMenu (right-click)
 │    ├── Copy / Paste / Duplicate
 │    ├── Delete / Select All
 │    ├── Bring to Front / Send to Back
 │    └── Add to Library
 │
 └── ExportDialog
      ├── FormatSelector (PNG, SVG, JSON, Clipboard)
      ├── ScaleSelector (1x, 2x, 3x)
      ├── BackgroundToggle (transparent / white)
      └── ExportPreview
```

---

### 5.2 Canvas Rendering Engine (Deep Dive)

The canvas is the **heart** of Excalidraw. Every shape, line, text element, and collaborator cursor is rendered onto an HTML5 Canvas 2D context. Getting the rendering pipeline right determines whether the app feels **fluid** or **sluggish**.

---

#### 5.2.1 Why HTML5 Canvas Over SVG

| Criterion | HTML5 Canvas 2D | SVG |
|-----------|-----------------|-----|
| **Rendering model** | Immediate mode — draw pixels, forget structure | Retained mode — DOM nodes for each element |
| **Performance at scale** | Excellent — rendering cost is O(visible elements) regardless of total | Degrades — each element is a DOM node; 1000+ elements causes jank |
| **Interactivity** | Manual hit-testing (math-based) | Built-in (click on SVG element) |
| **Styling** | Programmatic (via Rough.js for hand-drawn style) | CSS-based (easier for static styles) |
| **Export** | Can export to PNG natively; SVG requires re-rendering | Native SVG export is trivial |
| **Text rendering** | Limited (`fillText` — no wrapping, no rich text) | Full text layout support |
| **Memory** | Low — just pixels in a bitmap buffer | High — full DOM tree for each element |
| **Partial re-render** | Full or dirty-rect redraw | Browser handles DOM diffing per node |
| **Best for** | Large, dynamic, interactive drawings (Excalidraw, Figma) | Small, static, accessible diagrams |

**Excalidraw uses Canvas because:**
*   Scenes routinely contain hundreds to thousands of elements.
*   Constant re-renders during drag, resize, and pan require 60fps.
*   The hand-drawn style (via Rough.js) generates complex path data — rendering this as SVG DOM nodes would be extremely heavy.
*   Manual hit-testing is acceptable given the relatively simple shape primitives.

---

#### 5.2.2 Rendering Pipeline

Every time the scene changes (element drawn, moved, zoomed, panned), the canvas re-renders. The pipeline runs in a `requestAnimationFrame` loop:

```
User action (draw, pan, zoom, select)
     ↓
State update (scene elements, viewport transform)
     ↓
requestAnimationFrame scheduled (batches multiple updates per frame)
     ↓
┌─────────────────────────────────────────────────────┐
│  RENDER FRAME                                        │
│                                                      │
│  1. Clear canvas (or dirty region)                   │
│  2. Apply viewport transform (zoom + pan offset)     │
│  3. Determine visible elements (viewport culling)    │
│  4. Sort elements by z-index                         │
│  5. For each visible element:                        │
│     a. Apply element transform (position, rotation)  │
│     b. Render shape via Rough.js (hand-drawn paths)  │
│     c. Render text (if text element)                 │
│     d. Render selection handles (if selected)        │
│  6. Render grid (if enabled)                         │
│  7. Render collaborator cursors + names              │
│  8. Render selection box (if dragging to select)     │
└─────────────────────────────────────────────────────┘
     ↓
Browser composites and paints to screen
```

##### Implementation Sketch

```tsx
function renderScene(
  canvas: HTMLCanvasElement,
  elements: ExcalidrawElement[],
  appState: AppState,
  collaborators: Map<string, Collaborator>
) {
  const ctx = canvas.getContext('2d')!;
  const { zoom, scrollX, scrollY } = appState;

  // 1. Clear
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // 2. Apply viewport transform
  ctx.save();
  ctx.scale(zoom.value, zoom.value);
  ctx.translate(scrollX, scrollY);

  // 3. Viewport culling — only render elements within the visible area
  const viewportBounds = getViewportBounds(canvas, appState);
  const visibleElements = elements.filter((el) =>
    isElementInViewport(el, viewportBounds)
  );

  // 4. Sort by z-index (array order = z-order)
  // elements are already sorted — earlier in array = further back

  // 5. Render each element
  for (const element of visibleElements) {
    ctx.save();
    ctx.translate(element.x, element.y);
    if (element.angle) {
      ctx.rotate(element.angle);
    }

    renderElement(ctx, element); // delegates to Rough.js

    ctx.restore();
  }

  // 6. Render grid
  if (appState.gridSize) {
    renderGrid(ctx, appState);
  }

  // 7. Render collaborator cursors
  for (const [id, collab] of collaborators) {
    renderCursor(ctx, collab);
  }

  ctx.restore();

  // 8. Render selection handles (on interactive canvas layer)
  if (appState.selectedElementIds.length > 0) {
    renderSelectionHandles(ctx, appState, elements);
  }
}
```

##### Why `requestAnimationFrame` and Not Direct Rendering

```
BAD: Direct rendering on every state change
  mouseMove → render()    (60 events/sec during drag)
  mouseMove → render()    (browser may fire faster than display refresh)
  mouseMove → render()    ← wasted render, previous frame wasn't even painted

GOOD: requestAnimationFrame batching
  mouseMove → scheduleRender()  (sets a dirty flag)
  mouseMove → scheduleRender()  (flag already set — no-op)
  mouseMove → scheduleRender()  (flag already set — no-op)
  rAF fires → render()         (one render per display frame)
```

```tsx
let renderScheduled = false;

function scheduleRender() {
  if (renderScheduled) return;
  renderScheduled = true;
  requestAnimationFrame(() => {
    renderScene(canvas, elements, appState, collaborators);
    renderScheduled = false;
  });
}
```

This ensures:
*   At most one render per display frame (16.67ms at 60Hz).
*   No wasted renders between frames.
*   Mouse events can fire at 120Hz+ without causing 120 renders.

---

#### 5.2.3 Dirty Rectangle Optimization

For large scenes, clearing and re-rendering the **entire canvas** every frame is wasteful. If the user is dragging a single element, only the area around that element needs to be redrawn.

```
Full canvas: 1920 × 1080 = ~2M pixels cleared + redrawn

Dirty rect: 200 × 150 = 30K pixels cleared + redrawn
                         → 98.5% fewer pixels processed
```

##### How It Works

```
1. Track the BOUNDING BOX of every element that changed since last frame.
2. Union those bounding boxes into a single "dirty rectangle".
3. Expand by a small margin (for anti-aliasing and shadows).
4. Clear only the dirty rectangle.
5. Clip rendering to the dirty rectangle.
6. Re-render only elements that INTERSECT the dirty rectangle.
```

```tsx
function renderDirty(
  ctx: CanvasRenderingContext2D,
  dirtyRect: Rect,
  elements: ExcalidrawElement[]
) {
  // Expand for anti-aliasing margin
  const margin = 2;
  const expanded = {
    x: dirtyRect.x - margin,
    y: dirtyRect.y - margin,
    width: dirtyRect.width + margin * 2,
    height: dirtyRect.height + margin * 2,
  };

  // Clear only the dirty region
  ctx.clearRect(expanded.x, expanded.y, expanded.width, expanded.height);

  // Clip to dirty region — anything drawn outside is discarded
  ctx.save();
  ctx.beginPath();
  ctx.rect(expanded.x, expanded.y, expanded.width, expanded.height);
  ctx.clip();

  // Render only elements that overlap the dirty region
  const affectedElements = elements.filter((el) =>
    rectsOverlap(getElementBounds(el), expanded)
  );

  for (const element of affectedElements) {
    renderElement(ctx, element);
  }

  ctx.restore();
}
```

##### When to Use Dirty Rect vs Full Redraw

| Action | Strategy | Why |
|--------|----------|-----|
| Dragging single element | Dirty rect (old pos + new pos) | Only ~1% of canvas changes |
| Zoom / Pan | Full redraw | Everything moves — dirty rect = full canvas |
| Drawing freehand | Dirty rect (tip of the pen) | Very small region changes per frame |
| Adding text | Dirty rect (text bounding box) | Localized change |
| Deleting elements | Dirty rect (deleted element bounds) | Need to redraw background behind it |
| Collaboration (10 cursors moving) | Dirty rect (union of all cursor regions) | Multiple small regions |

---

#### 5.2.4 Multi Layer Canvas Architecture

##### Simple Analogy

Think of it like drawing on **two transparent sheets of plastic stacked on top of each other**, instead of one:

*   **Bottom sheet (Static Canvas)** — All your actual drawings live here (rectangles, arrows, text). This sheet only gets redrawn when you add, move, or delete a shape.
*   **Top sheet (Interactive Canvas)** — Temporary, fast-changing stuff lives here: your cursor, selection boxes (the blue dotted rectangle when you drag-select), resize handles (the little squares on corners), and other people's cursors in collaboration mode.

**Why bother with two layers?** Imagine you're just moving your mouse around (not drawing anything). Without two layers, the app would have to redraw **every single shape** on the canvas 60 times per second just to update a tiny cursor dot. With 1000 shapes, that's a lot of wasted work. With two layers, moving your mouse only redraws the **top sheet** (which has almost nothing on it — just a cursor). The bottom sheet with all 1000 shapes stays untouched — dramatically cheaper.

It's the same idea as **animation cels in old cartoons** — the background is painted once, and only the moving character is redrawn frame by frame on a transparent overlay.

##### Technical Details

Excalidraw uses **two stacked canvas elements** to avoid re-rendering the entire scene when only the interactive overlay changes:

```
┌─────────────────────────────────────┐
│  Interactive Canvas (top)           │  ← selection handles, resize handles,
│  (z-index: 2, pointer-events: none) │     collaborator cursors, drawing preview
├─────────────────────────────────────┤
│  Static Canvas (bottom)             │  ← committed elements (shapes, text, images)
│  (z-index: 1)                       │     re-renders only when elements change
└─────────────────────────────────────┘
```

**Why two layers?**

| Interaction | What changes | Without layers | With layers |
|-------------|-------------|---------------|-------------|
| Mouse hover (cursor position indicator) | Just the cursor overlay | Full scene redraw for a cursor dot | Redraw only interactive canvas |
| Dragging resize handle | Handle position | Full scene redraw | Static canvas untouched; interactive canvas redraws |
| Moving an element | Element position | Full scene redraw | Static canvas redraws (element moved); interactive canvas redraws (selection handles) |
| Collaborator cursor moves | Cursor position | Full scene redraw for remote cursor | Only interactive canvas |
| Drawing in progress | New shape preview | Full scene + new shape | Static untouched; interactive shows preview |

The static canvas only re-renders when the scene data changes (element added, moved, resized, deleted). The interactive canvas re-renders at mouse-move frequency (60fps) but only draws lightweight overlays.

```tsx
function CanvasWrapper() {
  const staticCanvasRef = useRef<HTMLCanvasElement>(null);
  const interactiveCanvasRef = useRef<HTMLCanvasElement>(null);

  // Static canvas — re-render when elements change
  useEffect(() => {
    renderStaticScene(staticCanvasRef.current!, elements, appState);
  }, [elements, appState.zoom, appState.scrollX, appState.scrollY]);

  // Interactive canvas — re-render on every pointer move
  useEffect(() => {
    const handlePointerMove = () => {
      renderInteractiveScene(
        interactiveCanvasRef.current!,
        appState,
        collaborators
      );
    };
    // Uses rAF internally
    window.addEventListener('pointermove', handlePointerMove);
    return () => window.removeEventListener('pointermove', handlePointerMove);
  }, [appState, collaborators]);

  return (
    <div className="canvas-wrapper" style={{ position: 'relative' }}>
      <canvas ref={staticCanvasRef} style={{ position: 'absolute' }} />
      <canvas
        ref={interactiveCanvasRef}
        style={{ position: 'absolute', pointerEvents: 'none' }}
      />
    </div>
  );
}
```

---

#### 5.2.5 Hand Drawn (Rough) Style Rendering

Excalidraw's signature look is the **hand-drawn, sketchy style**. This is achieved via **Rough.js**, a library that takes geometric primitives and renders them with randomized imperfections.

```
Standard Canvas:          Rough.js:
┌──────────┐              ┌~─~──────~─┐
│          │              │~         ~│
│          │              │           │
└──────────┘              └~──────~─~─┘
                           ↑ subtle wobbles, double strokes
```

##### How Rough.js Works

1.  You specify a shape: `roughCanvas.rectangle(10, 10, 200, 100)`.
2.  Rough.js generates **SVG path data** with randomized control points.
3.  It renders the path segments onto the Canvas 2D context using `ctx.stroke()` / `ctx.fill()`.

Each shape call generates a `Drawable` object — a set of path instructions. These are **cacheable**: once computed, the same Drawable can be re-rendered without recalculating the random paths.

---

#### 5.2.6 Hit Testing and Element Selection

Since Canvas is immediate-mode (no DOM nodes for shapes), we need **manual hit-testing** to determine which element the user clicked.

##### Hit Testing Algorithm

```
User clicks at (mouseX, mouseY)
     ↓
Transform to scene coordinates:
  sceneX = (mouseX / zoom) - scrollX
  sceneY = (mouseY / zoom) - scrollY
     ↓
Iterate elements in REVERSE z-order (top to bottom):
  For each element:
    1. Check if (sceneX, sceneY) is within the element's bounding box (fast reject)
    2. If yes, perform precise shape test:
       - Rectangle: point inside rect bounds
       - Ellipse: point satisfies (x/a)² + (y/b)² ≤ 1
       - Line/Arrow: distance from point to line segment < threshold
       - Freehand: distance to any segment of the path < threshold
       - Text: point inside measured text bounding box
    3. If hit → return this element (first match = topmost)
     ↓
No hit → deselect all
```

```tsx
function getElementAtPosition(
  elements: ExcalidrawElement[],
  sceneX: number,
  sceneY: number
): ExcalidrawElement | null {
  // Iterate in reverse (top-most elements first)
  for (let i = elements.length - 1; i >= 0; i--) {
    const element = elements[i];
    if (element.isDeleted) continue;

    // Fast bounding box check
    const bounds = getElementBounds(element);
    if (!isPointInRect(sceneX, sceneY, bounds)) continue;

    // Precise shape-specific hit test
    if (hitTestElement(element, sceneX, sceneY)) {
      return element;
    }
  }
  return null;
}

function hitTestElement(
  element: ExcalidrawElement,
  x: number,
  y: number
): boolean {
  // Translate to element-local coordinates
  const localX = x - element.x;
  const localY = y - element.y;

  switch (element.type) {
    case 'rectangle':
    case 'diamond':
    case 'image':
      return isPointInRect(localX, localY, { x: 0, y: 0, width: element.width, height: element.height });

    case 'ellipse':
      return isPointInEllipse(localX, localY, element.width / 2, element.height / 2);

    case 'line':
    case 'arrow':
      return isPointNearPolyline(localX, localY, element.points, HIT_THRESHOLD);

    case 'freedraw':
      return isPointNearPath(localX, localY, element.points, HIT_THRESHOLD);

    case 'text':
      return isPointInRect(localX, localY, getTextBounds(element));

    default:
      return false;
  }
}
```

##### Selection Visuals

When an element is selected, the **interactive canvas** renders:
*   A **bounding box** with dashed border around the element.
*   **Resize handles** at corners and edge midpoints (small squares).
*   A **rotation handle** above the top-center.
*   For multi-select, a unified bounding box around all selected elements.

---

#### 5.2.7 Zoom and Pan with Infinite Canvas

##### Viewport Transform

The canvas represents an **infinite 2D plane**. The visible area is a **viewport window** into that plane. The viewport is defined by:

*   `scrollX`, `scrollY` — the offset (how far the user has panned).
*   `zoom.value` — the scale factor.

All rendering is transformed by these values:

```
Screen coordinates → Scene coordinates:
  sceneX = (screenX / zoom) - scrollX
  sceneY = (screenY / zoom) - scrollY

Scene coordinates → Screen coordinates:
  screenX = (sceneX + scrollX) * zoom
  screenY = (sceneY + scrollY) * zoom
```

##### Pan Implementation

```tsx
// Pan via middle mouse button drag or hand tool
function handlePan(deltaX: number, deltaY: number) {
  setAppState((prev) => ({
    ...prev,
    scrollX: prev.scrollX + deltaX / prev.zoom.value,
    scrollY: prev.scrollY + deltaY / prev.zoom.value,
  }));
}

// Pan via scroll wheel (shift+scroll for horizontal)
function handleWheel(event: WheelEvent) {
  event.preventDefault();

  if (event.ctrlKey || event.metaKey) {
    // Pinch zoom (trackpad) or Ctrl+scroll
    handleZoom(event);
  } else {
    // Pan
    handlePan(-event.deltaX, -event.deltaY);
  }
}
```

##### Zoom to Cursor

When the user zooms (Ctrl+scroll or pinch), the zoom should center on the **cursor position**, not the canvas center. This is the "zoom to point" pattern:

```
Before zoom (zoom = 1.0):
  ┌─────────────────────────┐
  │          🖱️             │  ← cursor at screen (400, 300)
  │       [element]         │     = scene (400, 300)
  └─────────────────────────┘

After zoom IN to 2.0 (naive — zoom to center):
  ┌─────────────────────────┐
  │                         │  ← element moved away from cursor!
  │                         │     cursor still at (400, 300) on screen
  │             [element]   │     but element is now at different screen position
  └─────────────────────────┘

After zoom IN to 2.0 (correct — zoom to cursor):
  ┌─────────────────────────┐
  │          🖱️             │  ← cursor still points at the same scene position
  │       [element]         │     the scene expanded AROUND the cursor point
  └─────────────────────────┘
```

```tsx
function handleZoomAtPoint(
  newZoom: number,
  cursorScreenX: number,
  cursorScreenY: number
) {
  setAppState((prev) => {
    const oldZoom = prev.zoom.value;

    // Scene point under cursor (must stay fixed after zoom)
    const sceneX = cursorScreenX / oldZoom - prev.scrollX;
    const sceneY = cursorScreenY / oldZoom - prev.scrollY;

    // New scroll to keep the same scene point under the cursor
    const newScrollX = cursorScreenX / newZoom - sceneX;
    const newScrollY = cursorScreenY / newZoom - sceneY;

    return {
      ...prev,
      zoom: { value: clamp(newZoom, 0.1, 10) },
      scrollX: newScrollX,
      scrollY: newScrollY,
    };
  });
}
```

---

### 5.3 Reusability Strategy

*   **`CanvasRenderer`**: Core rendering engine used for both live canvas display and export rendering (PNG/SVG). The same render function draws to an on-screen canvas or an off-screen export canvas.
*   **`ElementRenderer`**: Delegates to shape-specific renderers (rectangle, ellipse, line, text, image). Each is a pure function: `(ctx, element) → void`.
*   **`HitTester`**: Reusable for click-to-select, drag-to-select (box intersection), and eraser tool (path intersection).
*   **`ViewportTransform`**: Shared coordinate transform utilities used by renderer, hit-tester, and export.
*   **`HistoryManager`**: Generic undo/redo engine (command pattern) reusable across any state-based application.
*   **`IndexedDBStore`**: Generic wrapper around IndexedDB operations, reusable for scenes, libraries, and user preferences.
*   Design system tokens for panel colors, button styles, icon sizes.

---

### 5.4 Module Organization

```
src/
 ├── renderer/         — Canvas rendering, Rough.js integration, drawable cache
 ├── scene/            — Element operations, hit testing, bounds, viewport
 ├── components/       — React UI (App, Canvas, Toolbar, Panels, Dialogs)
 ├── state/            — App state, undo/redo history, action manager
 ├── persistence/      — IndexedDB wrapper, auto-save, file import/export
 ├── collaboration/    — WebSocket client, CRDT engine, presence, encryption
 ├── tools/            — Per-tool logic (rectangle, ellipse, line, arrow, text, etc.)
 ├── shared/           — Math/geometry utils, clipboard, keyboard, analytics
 └── types/            — TypeScript type definitions
```

---

## 6. High Level Data Flow Explanation

### 6.1 Initial Load Flow

```
1. User opens excalidraw.com
     ↓
2. Static HTML shell loads (from CDN) → shows loading skeleton
     ↓
3. JS bundle loads and executes → App component mounts
     ↓
4. IndexedDB is opened → last saved scene is read
     ↓
5a. If scene found → restore elements + appState → render on canvas
5b. If no scene (first visit) → empty canvas with welcome hint
     ↓
6. Register event listeners (pointer, keyboard, wheel, resize)
     ↓
7. App is interactive — user can draw immediately
     ↓
8. (Optional) If collaboration link → connect WebSocket → join room → sync scene
```

---

### 6.2 User Interaction Flow

**Drawing a shape (e.g., rectangle):**

```
User selects Rectangle tool (click or press R)
     ↓
User presses mouse down on canvas
     ↓
1. Create a new element object:
   { id: nanoid(), type: 'rectangle', x, y, width: 0, height: 0, seed: randomInt(), ... }
     ↓
User drags mouse
     ↓
2. On each mousemove:
   - Update element.width = currentX - startX
   - Update element.height = currentY - startY
   - Re-render interactive canvas (shows rectangle preview)
     ↓
User releases mouse
     ↓
3. Commit element to scene state
4. Push to undo history
5. Debounced auto-save to IndexedDB (300ms)
6. If collaborating → broadcast element creation via WebSocket
```

**Moving an element:**

```
User clicks on an element → selected
     ↓
User drags the element
     ↓
1. On each mousemove:
   - Update element.x += deltaX, element.y += deltaY
   - Re-render both canvases
     ↓
User releases mouse
     ↓
2. Commit final position to scene state
3. Push position change to undo history (stores old + new position)
4. Debounced auto-save to IndexedDB
5. If collaborating → broadcast element update
```

---

### 6.3 Error and Retry Flow

*   **IndexedDB write fails** (storage quota exceeded): Show a warning toast "Storage is full — your drawing may not be saved. Export to file." Fall back to in-memory only.
*   **IndexedDB read fails** (corrupted database): Log the error, start with an empty canvas, offer to import from a `.excalidraw` file.
*   **WebSocket disconnects** (collaboration): Show "Reconnecting..." banner. Auto-reconnect with exponential backoff (1s → 2s → 4s → 8s, max 30s). Buffer local changes and sync on reconnect.
*   **Image paste/drop fails** (unsupported format, too large): Show toast "Could not add image — unsupported format or file too large (max 10MB)."
*   **Export fails** (canvas too large for PNG): Show error dialog, suggest exporting at lower scale or as SVG.
*   **Undo history corrupted**: Reset history stack, log error. User loses undo ability for past actions but can continue drawing.

---

## 7. Data Modelling (Frontend Perspective)

### 7.1 Core Data Entities

*   **ExcalidrawElement** — a single shape on the canvas (rectangle, ellipse, line, arrow, freehand, text, image).
*   **AppState** — the application state (selected tool, zoom level, scroll position, selected elements, active collaborators).
*   **Scene** — the complete drawing: all elements + app state.
*   **HistoryEntry** — a snapshot or delta for undo/redo.
*   **Library** — a collection of reusable element groups.
*   **Collaborator** — a remote user in a collaboration session (cursor position, name, color).

---

### 7.2 Data Shape

```ts
// Base element — all shapes extend this
type ExcalidrawElement = {
  id: string;                      // unique ID (nanoid)
  type: 'rectangle' | 'ellipse' | 'diamond' | 'line' | 'arrow' | 'freedraw' | 'text' | 'image' | 'frame';
  x: number;                      // top-left X in scene coordinates
  y: number;                      // top-left Y in scene coordinates
  width: number;
  height: number;
  angle: number;                   // rotation in radians
  strokeColor: string;             // e.g., "#1e1e1e"
  backgroundColor: string;        // e.g., "transparent" or "#a5d8ff"
  fillStyle: 'hachure' | 'cross-hatch' | 'solid';
  strokeWidth: 1 | 2 | 4;
  strokeStyle: 'solid' | 'dashed' | 'dotted';
  roughness: 0 | 1 | 2;           // 0 = clean, 1 = default sketch, 2 = very rough
  opacity: number;                 // 0-100
  seed: number;                    // deterministic random seed for Rough.js
  version: number;                 // incremented on every update (for CRDT)
  versionNonce: number;            // random nonce to break ties in concurrent edits
  isDeleted: boolean;              // soft-delete (for CRDT tombstoning)
  groupIds: string[];              // IDs of groups this element belongs to
  boundElements: BoundElement[];   // arrows/text bound to this element
  updated: number;                 // timestamp of last update
  link: string | null;             // hyperlink on the element
  locked: boolean;                 // prevent editing
};

// Line / Arrow specific
type ExcalidrawLinearElement = ExcalidrawElement & {
  type: 'line' | 'arrow';
  points: [number, number][];      // array of [x, y] offsets from element origin
  startBinding: PointBinding | null;
  endBinding: PointBinding | null;
  startArrowhead: 'arrow' | 'bar' | 'dot' | null;
  endArrowhead: 'arrow' | 'bar' | 'dot' | null;
};

type PointBinding = {
  elementId: string;               // ID of the shape this endpoint is bound to
  focus: number;                   // where on the element edge (-1 to 1)
  gap: number;                     // distance from the edge
};

// Text specific
type ExcalidrawTextElement = ExcalidrawElement & {
  type: 'text';
  text: string;
  fontSize: number;
  fontFamily: 1 | 2 | 3;          // 1=hand-drawn (Virgil), 2=normal (Helvetica), 3=code (Cascadia)
  textAlign: 'left' | 'center' | 'right';
  verticalAlign: 'top' | 'middle';
  containerId: string | null;      // if text is inside a shape (bound text)
  originalText: string;            // before word-wrapping
  lineHeight: number;
};

// Image specific
type ExcalidrawImageElement = ExcalidrawElement & {
  type: 'image';
  fileId: string;                  // reference to blob in IndexedDB
  status: 'pending' | 'saved' | 'error';
  scale: [number, number];
};

// Freehand draw
type ExcalidrawFreeDrawElement = ExcalidrawElement & {
  type: 'freedraw';
  points: [number, number, number][]; // [x, y, pressure] — supports stylus pressure
  simulatePressure: boolean;
};

// Application state
type AppState = {
  activeTool: ToolType;
  selectedElementIds: Record<string, boolean>;
  zoom: { value: number };
  scrollX: number;
  scrollY: number;
  cursorButton: 'up' | 'down';
  theme: 'light' | 'dark';
  gridSize: number | null;
  viewBackgroundColor: string;
  name: string;                    // scene/file name
  collaborators: Map<string, Collaborator>;
  isCollaborating: boolean;
  openDialog: string | null;
  lastSavedAt: number;
};

// Collaborator cursor
type Collaborator = {
  pointer: { x: number; y: number } | null;
  button: 'up' | 'down';
  selectedElementIds: Record<string, boolean>;
  username: string;
  color: { background: string; stroke: string };
};

// Library item (reusable shape group)
type LibraryItem = {
  id: string;
  status: 'published' | 'unpublished';
  elements: ExcalidrawElement[];
  name: string;
  created: number;
};

// Persisted scene (stored in IndexedDB)
type PersistedScene = {
  id: string;
  elements: ExcalidrawElement[];
  appState: Partial<AppState>;     // only persisted subset (not transient UI state)
  files: Record<string, BinaryFileData>;
  version: number;
  updatedAt: number;
  name: string;
};

// Binary file (embedded image)
type BinaryFileData = {
  mimeType: string;
  id: string;
  dataURL: string;                 // base64 data URL
  created: number;
  lastRetrieved: number;
};
```

---

### 7.3 Entity Relationships

*   **One-to-Many**: One `Scene` → many `ExcalidrawElement` items.
*   **One-to-One**: One `ExcalidrawTextElement` → one container element (`containerId`) if text is bound inside a shape.
*   **Many-to-Many**: Elements ↔ Groups (an element can be in multiple nested groups; a group contains multiple elements).
*   **Self-referential**: `ExcalidrawLinearElement.startBinding` / `endBinding` → another `ExcalidrawElement` (arrows connect to shapes).
*   **One-to-Many**: One `Scene` → many `BinaryFileData` (embedded images).
*   **One-to-Many**: One `LibraryItem` → many `ExcalidrawElement` (library groups).

**Data structure choice — Array, not Map:**
Elements are stored as an **ordered array**, not a map. The array order defines the **z-index** (earlier in array = further back). This makes z-order operations (bring to front, send to back) trivial — just move the element in the array.

```
elements = [
  { id: 'bg-rect', ... },    // z-index 0 (bottom)
  { id: 'circle-1', ... },   // z-index 1
  { id: 'arrow-1', ... },    // z-index 2
  { id: 'text-label', ... }, // z-index 3 (top)
]
```

For lookups by ID (selection, binding), a **derived Map** is computed lazily and cached:

```tsx
const elementMap = useMemo(
  () => new Map(elements.map((el) => [el.id, el])),
  [elements]
);
```

---

### 7.4 UI Specific Data Models

```ts
// Transient state during element creation (not persisted)
type DraggingElement = {
  element: ExcalidrawElement;
  startX: number;
  startY: number;
  originX: number;
  originY: number;
};

// Selection bounding box (computed, not stored)
type SelectionBounds = {
  x: number;
  y: number;
  width: number;
  height: number;
  angle: number;
  handles: ResizeHandle[];
};

// Derived: viewport bounds (what's visible on screen)
type ViewportBounds = {
  minX: number;
  minY: number;
  maxX: number;
  maxY: number;
};

// Export settings (transient dialog state)
type ExportConfig = {
  format: 'png' | 'svg' | 'json' | 'clipboard';
  scale: 1 | 2 | 3;
  withBackground: boolean;
  onlySelected: boolean;
};
```

---

## 8. State Management Strategy

### 8.1 State Classification

| State Type | Examples | Storage |
|---|---|---|
| **Scene State** | Elements array (all shapes on canvas) | In-memory + IndexedDB (auto-saved) |
| **App State** | Active tool, zoom, scroll, theme, grid | In-memory; subset persisted to IndexedDB |
| **History State** | Undo/redo stacks | In-memory only (not persisted) |
| **Collaboration State** | Connected users, their cursors, room ID | In-memory (ephemeral, from WebSocket) |
| **Component Local State** | Dialog open/closed, text input value, panel collapsed | `useState` / `useReducer` |
| **Derived / Computed State** | Element bounds, selection box, viewport visible elements | `useMemo` / computed on render |

---

### 8.2 State Ownership

*   **App (root)** owns the scene state (elements array) and app state. All state mutations go through a centralized `actionManager` that applies actions and records history.
*   **Canvas** reads elements and appState for rendering. It does not own state — it receives it via props or context.
*   **Toolbar / PropertiesPanel** reads appState (active tool, selected element properties) and dispatches actions to the actionManager.
*   **CollaborationOverlay** owns the WebSocket connection lifecycle. It receives remote changes and merges them into the scene state via the CRDT engine.
*   **HistoryManager** is a sibling to the scene state — it observes changes and maintains undo/redo stacks.

**State flow:**

```
User action (click tool, draw shape, change color)
     ↓
Action dispatched to ActionManager
     ↓
ActionManager:
  1. Applies the action to produce new (elements, appState)
  2. Records the change in HistoryManager (for undo)
  3. Triggers auto-save to IndexedDB (debounced)
  4. If collaborating: broadcasts the change via WebSocket
     ↓
New state flows to components → re-render
```

---

### 8.3 Persistence Strategy

| Data | Persistence | Reason |
|------|------------|--------|
| Scene elements | IndexedDB (auto-save, debounced 300ms) | Primary storage — local-first; survives browser refresh |
| App state (zoom, scroll, theme, grid) | IndexedDB (subset; auto-save with scene) | Restore viewport position on reload |
| Embedded images (blobs) | IndexedDB (separate object store) | Images can be large (MB); stored as blobs, not inline in scene |
| Undo/redo history | In-memory only | Too large to persist; acceptable to lose on page reload |
| Collaboration state | WebSocket (ephemeral) | Only exists while connected; no need to persist |
| Library items | IndexedDB (separate store) | Persist across sessions; user-curated |
| User preferences | localStorage | Small data; fast sync read on startup |

---

## 9. IndexedDB Deep Dive (Local First Storage)

### 9.1 Why IndexedDB for a Drawing App

Excalidraw is a **local-first** application — the primary copy of your data lives in **your browser**, not on a server. This requires a client-side storage solution that can handle:

| Requirement | localStorage | sessionStorage | IndexedDB |
|---|---|---|---|
| **Storage limit** | ~5-10MB | ~5MB | **50MB-unlimited** (browser prompts for more) |
| **Data types** | Strings only (must JSON.stringify) | Strings only | **Any structured data: objects, arrays, Blobs, ArrayBuffers, Files** |
| **Async / non-blocking** | Synchronous (blocks main thread) | Synchronous | **Asynchronous (non-blocking)** |
| **Transaction support** | None | None | **Full ACID transactions** |
| **Indexed queries** | Key-value only | Key-value only | **Indexes on any object property** |
| **Binary data (images)** | Must base64 encode (33% size overhead) | Must base64 encode | **Native Blob/ArrayBuffer support (no overhead)** |
| **Survives browser close** | Yes | No | **Yes** |
| **Web Worker support** | No | No | **Yes** |

**Why not localStorage?**

A typical Excalidraw scene with 200 elements is ~500KB of JSON. With embedded images, a scene can easily reach 5-20MB. localStorage's 5-10MB limit and sync blocking API make it unsuitable:

```
localStorage.setItem('scene', JSON.stringify(sceneData)); // ← BLOCKS main thread
// For a 5MB scene, this can block for 50-100ms — visible frame drops during auto-save
```

**Why IndexedDB?**
*   **Large storage**: Can store 50MB+ without issues. Browser may prompt for permission beyond quota.
*   **Async**: All operations return Promises / use callbacks — never blocks the main thread.
*   **Blob storage**: Embedded images can be stored as raw Blobs (no base64 encoding overhead).
*   **Transactions**: Batch multiple writes atomically — if the app crashes mid-save, data is either fully written or not at all.
*   **Structured cloning**: Objects are stored as-is (no JSON.stringify / parse overhead for complex nested structures).

---

### 9.2 IndexedDB Schema Design

```
Database: "excalidraw-store"
Version: 3

┌─────────────────────────────────────────────────────────────┐
│  Object Store: "scenes"                                      │
│  keyPath: "id"                                               │
│  Indexes:                                                    │
│    - "updatedAt" (for sorting by last modified)              │
│    - "name" (for search)                                     │
│                                                              │
│  Shape: {                                                    │
│    id: string,              // unique scene ID               │
│    elements: ExcalidrawElement[],                            │
│    appState: Partial<AppState>,                              │
│    name: string,                                             │
│    version: number,                                          │
│    createdAt: number,       // Date.now() timestamp          │
│    updatedAt: number,       // Date.now() timestamp          │
│  }                                                           │
├────────────────────────────────────────────────────────── ───┤
│  Object Store: "files"                                       │
│  keyPath: "id"                                               │
│  Indexes:                                                    │
│    - "sceneId" (for cleanup: delete orphaned files)          │
│    - "lastRetrieved" (for LRU eviction)                      │
│                                                              │
│  Shape: {                                                    │
│    id: string,              // file ID(referenced by element)│
│    sceneId: string,         // parent scene                  │
│    mimeType: string,        // "image/png", "image/jpeg"     │
│    data: Blob,              // raw binary image data         │
│    created: number,                                          │
│    lastRetrieved: number,   // updated on read (for LRU)     │
│    size: number,            // bytes                         │
│  }                                                           │
├─────────────────────────────────────────────────────────── ──┤
│  Object Store: "libraries"                                   │
│  keyPath: "id"                                               │
│                                                              │
│  Shape: {                                                    │
│    id: string,                                               │
│    name: string,                                             │
│    elements: ExcalidrawElement[][],  // array of item groups │
│    source: string,           // URL or "local"               │
│    addedAt: number,                                          │
│  }                                                           │
├─────────────────────────────────────────────────────────── ──┤
│  Object Store: "metadata"                                    │
│  keyPath: "key"                                              │
│                                                              │
│  Shape: {                                                    │
│    key: string,              // e.g., "lastActiveSceneId"    │
│    value: any,                                               │
│  }                                                           │
└─────────────────────────────────────────────────────────── ──┘
```

##### Database Initialization

```tsx
const DB_NAME = 'excalidraw-store';
const DB_VERSION = 3;

function openDatabase(): Promise<IDBDatabase> {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open(DB_NAME, DB_VERSION);

    request.onupgradeneeded = (event) => {
      const db = request.result;
      const oldVersion = event.oldVersion;

      // Version 1: initial schema
      if (oldVersion < 1) {
        const sceneStore = db.createObjectStore('scenes', { keyPath: 'id' });
        sceneStore.createIndex('updatedAt', 'updatedAt', { unique: false });
        sceneStore.createIndex('name', 'name', { unique: false });
      }

      // Version 2: added files store for image blobs
      if (oldVersion < 2) {
        const fileStore = db.createObjectStore('files', { keyPath: 'id' });
        fileStore.createIndex('sceneId', 'sceneId', { unique: false });
        fileStore.createIndex('lastRetrieved', 'lastRetrieved', { unique: false });
      }

      // Version 3: added libraries and metadata stores
      if (oldVersion < 3) {
        db.createObjectStore('libraries', { keyPath: 'id' });
        db.createObjectStore('metadata', { keyPath: 'key' });
      }
    };

    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}
```

##### Why `onupgradeneeded` with Version Checks

IndexedDB uses a **version number** to manage schema migrations. When you open a database with a higher version than what exists, `onupgradeneeded` fires. The version checks (`if (oldVersion < N)`) ensure that:
*   A fresh install runs all migrations (1 → 2 → 3).
*   An existing user on version 2 only runs migration 3.
*   Migrations are **additive and non-destructive** — existing data is never lost.

```
First visit:        oldVersion=0  → creates stores for v1, v2, v3
Returning user v1:  oldVersion=1  → adds files store (v2), libraries + metadata (v3)
Returning user v2:  oldVersion=2  → adds libraries + metadata (v3)
Returning user v3:  oldVersion=3  → no upgrade needed
```

---

### 9.3 CRUD Operations on Scenes

##### Save Scene (Create or Update)

```tsx
async function saveScene(scene: PersistedScene): Promise<void> {
  const db = await openDatabase();

  return new Promise((resolve, reject) => {
    const tx = db.transaction(['scenes'], 'readwrite');
    const store = tx.objectStore('scenes');

    // put() creates or updates (upsert)
    const request = store.put({
      ...scene,
      updatedAt: Date.now(),
    });

    tx.oncomplete = () => resolve();
    tx.onerror = () => reject(tx.error);
  });
}
```

##### Load Scene by ID

```tsx
async function loadScene(sceneId: string): Promise<PersistedScene | null> {
  const db = await openDatabase();

  return new Promise((resolve, reject) => {
    const tx = db.transaction(['scenes'], 'readonly');
    const store = tx.objectStore('scenes');
    const request = store.get(sceneId);

    request.onsuccess = () => resolve(request.result ?? null);
    request.onerror = () => reject(request.error);
  });
}
```

##### List All Scenes (Sorted by Last Modified)

```tsx
async function listScenes(): Promise<PersistedScene[]> {
  const db = await openDatabase();

  return new Promise((resolve, reject) => {
    const tx = db.transaction(['scenes'], 'readonly');
    const store = tx.objectStore('scenes');
    const index = store.index('updatedAt');

    // Open cursor in reverse order (newest first)
    const request = index.openCursor(null, 'prev');
    const results: PersistedScene[] = [];

    request.onsuccess = () => {
      const cursor = request.result;
      if (cursor) {
        results.push(cursor.value);
        cursor.continue();
      } else {
        resolve(results);
      }
    };

    request.onerror = () => reject(request.error);
  });
}
```

##### Delete Scene (with Cascade to Files)

```tsx
async function deleteScene(sceneId: string): Promise<void> {
  const db = await openDatabase();

  return new Promise((resolve, reject) => {
    // Use a single transaction across both stores for atomicity
    const tx = db.transaction(['scenes', 'files'], 'readwrite');

    // Delete the scene
    tx.objectStore('scenes').delete(sceneId);

    // Delete all associated image files
    const fileIndex = tx.objectStore('files').index('sceneId');
    const cursorReq = fileIndex.openKeyCursor(IDBKeyRange.only(sceneId));

    cursorReq.onsuccess = () => {
      const cursor = cursorReq.result;
      if (cursor) {
        tx.objectStore('files').delete(cursor.primaryKey);
        cursor.continue();
      }
    };

    tx.oncomplete = () => resolve();
    tx.onerror = () => reject(tx.error);
  });
}
```

**Why cascade delete in a single transaction?** If the scene is deleted but the file deletion fails (browser crash, tab close), we'd have orphaned blobs wasting storage. A single transaction guarantees both operations succeed or both fail.

---

### 9.4 Auto Save Strategy

Excalidraw has **no save button**. Every change is automatically persisted to IndexedDB. The challenge is balancing **data safety** (save often) with **performance** (don't write to IndexedDB on every pixel moved during a drag).

##### Debounced Auto Save

```tsx
const SAVE_DEBOUNCE_MS = 300;

class AutoSaveManager {
  private timer: ReturnType<typeof setTimeout> | null = null;
  private pendingScene: PersistedScene | null = null;
  private isSaving = false;

  // Called on every state change
  scheduleAutoSave(scene: PersistedScene) {
    this.pendingScene = scene;

    // Clear previous timer (debounce)
    if (this.timer) clearTimeout(this.timer);

    // Schedule new save
    this.timer = setTimeout(() => this.flush(), SAVE_DEBOUNCE_MS);
  }

  // Force immediate save (e.g., before page unload)
  async flush() {
    if (!this.pendingScene || this.isSaving) return;

    this.isSaving = true;
    const sceneToSave = this.pendingScene;
    this.pendingScene = null;

    try {
      await saveScene(sceneToSave);
      console.debug('[AutoSave] Scene saved at', new Date().toISOString());
    } catch (error) {
      console.error('[AutoSave] Failed to save:', error);
      // Re-queue the failed save
      this.pendingScene = sceneToSave;
      this.scheduleAutoSave(sceneToSave);
    } finally {
      this.isSaving = false;
    }
  }
}
```

##### Save on Page Unload

When the user closes the tab or navigates away, we must save immediately — the debounce timer hasn't fired yet:

```tsx
// Register before-unload handler
window.addEventListener('beforeunload', () => {
  autoSaveManager.flush();
});

// Also use navigator.sendBeacon or visibilitychange for mobile
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden') {
    autoSaveManager.flush();
  }
});
```

**Why `visibilitychange` in addition to `beforeunload`?**
On mobile browsers, `beforeunload` is unreliable — the OS may kill the tab without firing the event. `visibilitychange → hidden` fires reliably when the user switches tabs, switches apps, or locks their phone.

##### Auto Save Timing

| Event | Save Strategy |
|-------|--------------|
| Element drawn / moved / resized / deleted | Debounce 300ms, then save |
| Style change (color, stroke, font) | Debounce 300ms, then save |
| Zoom / pan | Do NOT save (too frequent; save on next element change) |
| Paste / import | Immediate save (significant data change) |
| Tab close / hide | Immediate flush |
| Collaboration sync received | Debounce 1000ms (remote changes arrive in bursts) |

---

### 9.5 Image and Asset Blob Storage

When a user embeds an image (paste, drag-drop, or upload), the image is stored as a **Blob** in the `files` object store. The scene element references it by `fileId`.

```
Scene Element (in "scenes" store):
  { type: 'image', fileId: 'img_abc123', ... }
                              │
                              ▼
File Blob (in "files" store):
  { id: 'img_abc123', data: Blob(45KB), mimeType: 'image/png', sceneId: 'scene_1' }
```

##### Saving an Image

```tsx
async function saveImageFile(
  sceneId: string,
  fileId: string,
  blob: Blob,
  mimeType: string
): Promise<void> {
  const db = await openDatabase();

  return new Promise((resolve, reject) => {
    const tx = db.transaction(['files'], 'readwrite');
    const store = tx.objectStore('files');

    store.put({
      id: fileId,
      sceneId,
      mimeType,
      data: blob,            // Raw Blob — no base64 encoding!
      created: Date.now(),
      lastRetrieved: Date.now(),
      size: blob.size,
    });

    tx.oncomplete = () => resolve();
    tx.onerror = () => reject(tx.error);
  });
}
```

##### Loading an Image (with Object URL)

```tsx
async function loadImageFile(fileId: string): Promise<string | null> {
  const db = await openDatabase();

  return new Promise((resolve, reject) => {
    const tx = db.transaction(['files'], 'readwrite');
    const store = tx.objectStore('files');
    const request = store.get(fileId);

    request.onsuccess = () => {
      const file = request.result;
      if (!file) {
        resolve(null);
        return;
      }

      // Update lastRetrieved for LRU tracking
      file.lastRetrieved = Date.now();
      store.put(file);

      // Create an Object URL from the Blob for rendering
      const objectURL = URL.createObjectURL(file.data);
      resolve(objectURL);
    };

    request.onerror = () => reject(request.error);
  });
}
```

**Why Blob instead of base64 data URL?**

| Storage method | 1MB image stored as | IndexedDB size | Read overhead |
|---|---|---|---|
| base64 data URL (string) | ~1.33MB (33% overhead from base64 encoding) | 1.33MB | Must decode base64 → binary on every render |
| **Blob** | 1MB (raw binary) | **1MB** | Zero — `URL.createObjectURL()` gives a direct memory reference |

For a scene with 10 embedded images, the difference is **3.3MB saved** and zero decode overhead.

##### Image Loading Pipeline

```
User pastes/drops an image
     ↓
1. Read as Blob from clipboard/file input
     ↓
2. Generate fileId (nanoid)
     ↓
3. Create image element: { type: 'image', fileId, ... }
     ↓
4. Store Blob in IndexedDB "files" store (async)
     ↓
5. Create URL.createObjectURL(blob) for immediate rendering
     ↓
6. When scene reloads later:
   a. Scene loads from IndexedDB → element has fileId
   b. Load Blob from "files" store by fileId
   c. URL.createObjectURL(blob) → render on canvas
```

##### Memory Management for Object URLs

`URL.createObjectURL()` creates a reference to a Blob in browser memory. If you never revoke it, it leaks memory.

```tsx
const objectURLCache = new Map<string, string>();

function getImageObjectURL(fileId: string, blob: Blob): string {
  const existing = objectURLCache.get(fileId);
  if (existing) return existing;

  const url = URL.createObjectURL(blob);
  objectURLCache.set(fileId, url);
  return url;
}

// Revoke when scene is closed or image is removed
function revokeImageObjectURL(fileId: string) {
  const url = objectURLCache.get(fileId);
  if (url) {
    URL.revokeObjectURL(url);
    objectURLCache.delete(fileId);
  }
}
```

---

### 9.6 Versioning and Migration

As the app evolves, the data shape stored in IndexedDB changes. Migrations are handled via IndexedDB's built-in versioning in `onupgradeneeded`:

##### Migration Strategy

```tsx
request.onupgradeneeded = (event) => {
  const db = request.result;
  const oldVersion = event.oldVersion;
  const tx = request.transaction!;

  // V1 → V2: Add files store
  if (oldVersion < 2) {
    const fileStore = db.createObjectStore('files', { keyPath: 'id' });
    fileStore.createIndex('sceneId', 'sceneId', { unique: false });
    fileStore.createIndex('lastRetrieved', 'lastRetrieved', { unique: false });
  }

  // V2 → V3: Add 'version' field to all existing scenes
  if (oldVersion < 3 && oldVersion >= 1) {
    const sceneStore = tx.objectStore('scenes');
    const cursorReq = sceneStore.openCursor();
    cursorReq.onsuccess = () => {
      const cursor = cursorReq.result;
      if (cursor) {
        const scene = cursor.value;
        if (!scene.version) {
          scene.version = 1; // backfill default version
          cursor.update(scene);
        }
        cursor.continue();
      }
    };
  }
};
```

##### Data Shape Migration (Within Scene Data)

Sometimes the element data shape changes (e.g., adding a new property, renaming a field). This happens **on read**, not during the IndexedDB upgrade:

```tsx
function migrateElement(element: any): ExcalidrawElement {
  // v1 → v2: "strokeType" renamed to "strokeStyle"
  if ('strokeType' in element && !('strokeStyle' in element)) {
    element.strokeStyle = element.strokeType;
    delete element.strokeType;
  }

  // v2 → v3: "groupIds" added (default to empty array)
  if (!element.groupIds) {
    element.groupIds = [];
  }

  // v3 → v4: "locked" added (default to false)
  if (element.locked === undefined) {
    element.locked = false;
  }

  return element as ExcalidrawElement;
}

// Applied when loading a scene
async function loadSceneWithMigration(sceneId: string): Promise<PersistedScene | null> {
  const scene = await loadScene(sceneId);
  if (!scene) return null;

  // Migrate each element
  scene.elements = scene.elements.map(migrateElement);

  return scene;
}
```

**Why migrate on read instead of in `onupgradeneeded`?**
*   `onupgradeneeded` runs synchronously and blocks the database — migrating thousands of elements would freeze the app.
*   On-read migration is lazy — only the currently loaded scene is migrated.
*   The migrated scene is saved back on the next auto-save, so the migration only happens once per scene.

---

### 9.7 Storage Quota and Cleanup

Browsers have storage limits. On Chrome, it is typically 60% of the disk space for all origins, with per-origin limits. When quota is approached, older data should be evicted.

##### Check Available Storage

```tsx
async function getStorageEstimate(): Promise<{ used: number; quota: number }> {
  if ('storage' in navigator && 'estimate' in navigator.storage) {
    const estimate = await navigator.storage.estimate();
    return {
      used: estimate.usage ?? 0,
      quota: estimate.quota ?? 0,
    };
  }
  return { used: 0, quota: Infinity };
}
```

##### LRU Eviction for Image Blobs

When storage is tight, evict the least recently used image blobs first:

```tsx
async function evictOldFiles(targetFreeBytes: number): Promise<number> {
  const db = await openDatabase();
  let freedBytes = 0;

  return new Promise((resolve, reject) => {
    const tx = db.transaction(['files'], 'readwrite');
    const store = tx.objectStore('files');
    const index = store.index('lastRetrieved');

    // Open cursor in ascending order (oldest retrieved first)
    const cursorReq = index.openCursor();

    cursorReq.onsuccess = () => {
      const cursor = cursorReq.result;
      if (!cursor || freedBytes >= targetFreeBytes) {
        resolve(freedBytes);
        return;
      }

      const file = cursor.value;
      freedBytes += file.size;
      cursor.delete();
      cursor.continue();
    };

    tx.onerror = () => reject(tx.error);
  });
}
```

##### Request Persistent Storage

By default, browser storage can be evicted under storage pressure (e.g., other sites using lots of storage). You can request **persistent storage** to prevent this:

```tsx
async function requestPersistentStorage(): Promise<boolean> {
  if (navigator.storage && navigator.storage.persist) {
    const granted = await navigator.storage.persist();
    console.log('Persistent storage:', granted ? 'granted' : 'denied');
    return granted;
  }
  return false;
}
```

**When to request:** On the first meaningful save (user has created content worth preserving). Don't request on first visit before the user has drawn anything.

---

### 9.8 IndexedDB vs Alternatives Comparison

| Criterion | IndexedDB | localStorage | Cache API | OPFS (Origin Private File System) |
|---|---|---|---|---|
| **Storage limit** | 50MB-unlimited | 5-10MB | Large (like IndexedDB) | Large |
| **Data types** | Any (objects, Blobs, arrays) | Strings only | Request/Response pairs | Files (binary data) |
| **API** | Async (callback/Promise) | Sync (blocking) | Async (Promise) | Async (Promise) |
| **Transactions** | Yes (ACID) | No | No | No |
| **Query/Index** | Yes (indexes, cursors, ranges) | Key-value only | URL-based lookup | File path-based |
| **Binary data** | Native Blob support | Base64 string (33% overhead) | Response body | Native file I/O |
| **Web Worker** | Yes | No | Yes | Yes |
| **Browser support** | All modern | All | All modern | Chrome 86+, Firefox 111+, Safari 15.2+ |
| **Best for** | Structured app data with queries | Small key-value config | HTTP response caching | High-performance file I/O |
| **Excalidraw use** | Primary storage (scenes, files, libraries) | User preferences only | Not used | Potential future (large file performance) |

**Why not Cache API for scene data?** Cache API is designed for HTTP request/response pairs. Storing arbitrary application state requires wrapping it in a fake Response object — an awkward misuse of the API. IndexedDB is purpose-built for application data.

**Why not OPFS?** OPFS (Origin Private File System) offers high-performance file I/O and is ideal for very large files (100MB+). However, browser support is still maturing, and the API is file-oriented (not record-oriented). It would be a good fit for a future optimization: storing large image blobs in OPFS while keeping scene metadata in IndexedDB.

---

## 10. Undo Redo System (Deep Dive)

### 10.1 Command Pattern Architecture

Every user action that modifies the scene is represented as a **command** with enough information to **undo** and **redo** it.

##### Two Approaches

| Approach | How it works | Pros | Cons |
|----------|-------------|------|------|
| **Snapshot-based** | Store the entire elements array on each change | Simple; trivial undo (just restore snapshot) | Memory-heavy: 500 elements × 100 history entries = 50,000 element copies |
| **Delta-based (Command Pattern)** | Store only what changed: `{ type: 'move', elementId, oldPos, newPos }` | Memory-efficient | Complex: must implement inverse for every action type |

**Excalidraw uses a hybrid approach:**
*   Store a **snapshot of only the changed elements** (not the entire scene).
*   On undo, replace the changed elements with their previous versions.

```tsx
type HistoryEntry = {
  elementsChange: Map<string, {
    before: ExcalidrawElement | null;  // null if element was created (undo = delete)
    after: ExcalidrawElement | null;   // null if element was deleted (redo = delete)
  }>;
  appStateChange: Partial<AppState> | null;
};
```

| Action | `before` | `after` |
|--------|---------|---------|
| Create element | `null` | `{ ...newElement }` |
| Delete element | `{ ...deletedElement }` | `null` |
| Move element | `{ x: 10, y: 20, ... }` | `{ x: 50, y: 80, ... }` |
| Change color | `{ strokeColor: '#000' }` | `{ strokeColor: '#f00' }` |
| Multi-element move | Multiple entries in the Map | Multiple entries |

---

### 10.2 History Stack Implementation

```tsx
class HistoryManager {
  private undoStack: HistoryEntry[] = [];
  private redoStack: HistoryEntry[] = [];
  private maxEntries = 100;

  // Record a change (called after every action)
  record(entry: HistoryEntry) {
    this.undoStack.push(entry);

    // Clear redo stack — after a new action, redo is no longer valid
    this.redoStack = [];

    // Cap history size
    if (this.undoStack.length > this.maxEntries) {
      this.undoStack.shift(); // remove oldest
    }
  }

  // Undo: pop from undo stack, apply "before" states, push to redo stack
  undo(currentElements: ExcalidrawElement[]): ExcalidrawElement[] | null {
    const entry = this.undoStack.pop();
    if (!entry) return null;

    // Push the inverse to redo stack
    this.redoStack.push(entry);

    // Apply "before" states
    return applyHistoryEntry(currentElements, entry, 'undo');
  }

  // Redo: pop from redo stack, apply "after" states, push to undo stack
  redo(currentElements: ExcalidrawElement[]): ExcalidrawElement[] | null {
    const entry = this.redoStack.pop();
    if (!entry) return null;

    this.undoStack.push(entry);

    return applyHistoryEntry(currentElements, entry, 'redo');
  }

  canUndo(): boolean { return this.undoStack.length > 0; }
  canRedo(): boolean { return this.redoStack.length > 0; }
}

function applyHistoryEntry(
  elements: ExcalidrawElement[],
  entry: HistoryEntry,
  direction: 'undo' | 'redo'
): ExcalidrawElement[] {
  let result = [...elements];

  for (const [elementId, change] of entry.elementsChange) {
    const target = direction === 'undo' ? change.before : change.after;

    if (target === null) {
      // Element should not exist → mark as deleted (soft delete for CRDT compatibility)
      result = result.map((el) =>
        el.id === elementId ? { ...el, isDeleted: true } : el
      );
    } else {
      // Element should exist with these properties
      const existingIndex = result.findIndex((el) => el.id === elementId);
      if (existingIndex >= 0) {
        result[existingIndex] = target;
      } else {
        result.push(target); // re-add deleted element
      }
    }
  }

  return result;
}
```

##### Why Soft Delete (`isDeleted`) Instead of Array Removal

In a collaborative environment, removing an element from the array causes **index-based conflicts**. If User A deletes element 5 and User B moves element 6, the array positions are inconsistent. By using `isDeleted: true`, the element stays in the array (maintaining positions) but is skipped during rendering and hit-testing. The CRDT can properly merge the delete with other concurrent operations.

---

## 11. Real Time Collaboration (Deep Dive)

### 11.1 Collaboration Architecture Overview

```
┌──────────────────┐         WebSocket          ┌──────────────────┐
│   User A         │◄───────────────────────────│   User B         │
│   (Browser)      │         (encrypted)        │   (Browser)      │
│                  ├───────────────────────────►│                  │
│  Scene State     │                            │  Scene State     │
│  CRDT Engine     │                            │  CRDT Engine     │
│  Encryption      │                            │  Encryption      │
└────────┬─────────┘                            └────────┬─────────┘
         │                                               │
         └──────────┐         ┌──────────────────────────┘
                    ▼         ▼
              ┌─────────────────────┐
              │  WebSocket Relay    │
              │  Server             │
              │  (No data storage)  │
              │  (Cannot decrypt)   │
              └─────────────────────┘
```

The server is a **dumb relay** — it forwards encrypted messages between clients in a room. It has **no knowledge** of the scene data (end-to-end encryption).

---

### 11.2 Conflict Resolution with CRDTs

When two users edit simultaneously, their changes may conflict. Excalidraw uses a **Last-Writer-Wins (LWW) per-element** strategy, which is a simple CRDT approach:

```
User A: Moves rectangle to (100, 200) → version=5, versionNonce=42
User B: Changes rectangle color to red → version=5, versionNonce=87

Both arrive at the relay server → both forwarded to both clients.

Resolution rule:
  Compare version numbers:
    - Higher version wins.
    - If tied, higher versionNonce wins (random tiebreaker).

Result:
  version=5, nonce=87 > nonce=42 → User B's change wins.
  BUT User A's position change is on a different property...
```

In practice, Excalidraw resolves at the **element level**, not the property level. This means if two users edit the **same element** simultaneously, one user's change is overwritten entirely. This is acceptable because:
*   Concurrent edit of the **same element** is rare (users typically work on different parts of the canvas).
*   Elements are small (a single shape) — the conflict is visible and easy to redo.
*   Per-property CRDT merge would be significantly more complex.

##### Implementation

```tsx
function reconcileElements(
  localElements: ExcalidrawElement[],
  remoteElements: ExcalidrawElement[]
): ExcalidrawElement[] {
  const localMap = new Map(localElements.map((el) => [el.id, el]));
  const reconciledMap = new Map(localMap);

  for (const remoteEl of remoteElements) {
    const localEl = localMap.get(remoteEl.id);

    if (!localEl) {
      // New element from remote — add it
      reconciledMap.set(remoteEl.id, remoteEl);
    } else if (
      remoteEl.version > localEl.version ||
      (remoteEl.version === localEl.version &&
        remoteEl.versionNonce > localEl.versionNonce)
    ) {
      // Remote is newer — replace local
      reconciledMap.set(remoteEl.id, remoteEl);
    }
    // else: local is newer or equal — keep local
  }

  // Maintain array order (z-index) — merge remote element positions
  return Array.from(reconciledMap.values());
}
```

---

### 11.3 Cursor and Presence Awareness

Each collaborator broadcasts their cursor position and selected elements at high frequency (throttled to ~30fps):

```tsx
// Broadcast local cursor position
function broadcastPresence(ws: WebSocket, appState: AppState) {
  const message = {
    type: 'presence',
    payload: {
      pointer: appState.cursorPosition,
      button: appState.cursorButton,
      selectedElementIds: appState.selectedElementIds,
      username: appState.username,
    },
  };

  ws.send(encrypt(JSON.stringify(message)));
}

// Throttle to 30fps to reduce WebSocket traffic
const throttledBroadcast = throttle(broadcastPresence, 33);
```

##### Rendering Remote Cursors

```tsx
function renderCursor(ctx: CanvasRenderingContext2D, collaborator: Collaborator) {
  if (!collaborator.pointer) return;

  const { x, y } = collaborator.pointer;

  // Draw cursor arrow
  ctx.save();
  ctx.fillStyle = collaborator.color.background;
  ctx.strokeStyle = collaborator.color.stroke;
  ctx.beginPath();
  ctx.moveTo(x, y);
  ctx.lineTo(x + 2, y + 14);
  ctx.lineTo(x + 8, y + 10);
  ctx.closePath();
  ctx.fill();
  ctx.stroke();

  // Draw username label
  ctx.font = '12px sans-serif';
  ctx.fillStyle = collaborator.color.background;
  ctx.fillRect(x + 10, y + 12, ctx.measureText(collaborator.username).width + 8, 20);
  ctx.fillStyle = '#fff';
  ctx.fillText(collaborator.username, x + 14, y + 26);
  ctx.restore();
}
```

---

### 11.4 Room and Session Management

```
Create Room:
  1. Generate random room ID + encryption key (client-side)
  2. Room link format: https://excalidraw.com/#room=<roomId>,<encryptionKey>
     (key is in the URL fragment — never sent to server)
  3. Connect to WebSocket: wss://collab.excalidraw.com/<roomId>
  4. Broadcast full scene to new joiners

Join Room:
  1. Open room link → extract roomId and encryptionKey from URL fragment
  2. Connect to WebSocket
  3. Receive initial scene from existing participants (encrypted)
  4. Decrypt and render
  5. Start sending/receiving incremental updates
```

##### End to End Encryption

The encryption key is the **URL fragment** (after `#`). URL fragments are **never sent to the server** by the browser — the server only sees the room ID. This means:
*   The relay server cannot decrypt any messages.
*   Only users who have the full link can read the content.
*   If you share the link via a secure channel, the content is fully private.

```tsx
async function encrypt(data: string, key: CryptoKey): Promise<ArrayBuffer> {
  const iv = crypto.getRandomValues(new Uint8Array(12)); // 96-bit IV for AES-GCM
  const encoded = new TextEncoder().encode(data);

  const encrypted = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv },
    key,
    encoded
  );

  // Prepend IV to encrypted data (receiver needs it to decrypt)
  const result = new Uint8Array(iv.length + encrypted.byteLength);
  result.set(iv);
  result.set(new Uint8Array(encrypted), iv.length);

  return result.buffer;
}

async function decrypt(data: ArrayBuffer, key: CryptoKey): Promise<string> {
  const array = new Uint8Array(data);
  const iv = array.slice(0, 12);
  const ciphertext = array.slice(12);

  const decrypted = await crypto.subtle.decrypt(
    { name: 'AES-GCM', iv },
    key,
    ciphertext
  );

  return new TextDecoder().decode(decrypted);
}
```

---

## 12. High Level API Design (Frontend POV)

### 12.1 Required APIs

| API | Method | Description |
|-----|--------|-------------|
| `wss://collab.excalidraw.com/:roomId` | **WebSocket** | Real-time scene sync and cursor presence |
| `/api/scenes` | **POST** | Save scene to cloud (optional, for logged-in users) |
| `/api/scenes/:id` | **GET** | Load a cloud-saved scene |
| `/api/scenes/:id` | **DELETE** | Delete a cloud-saved scene |
| `/api/share` | **POST** | Generate a shareable link for a static snapshot |
| `/api/share/:id` | **GET** | Load a shared scene snapshot (read-only) |
| `/api/libraries` | **GET** | Fetch public libraries or user's saved libraries |

**Note:** For solo mode (the primary use case), **no APIs are called**. Everything is local (IndexedDB). APIs are only used for cloud save and sharing features.

---

### 12.2 Request and Response Structure

**WebSocket Messages**

```json
// Scene update (broadcast to room)
{
  "type": "scene_update",
  "payload": {
    "elements": [ /* encrypted ExcalidrawElement[] diff */ ],
    "version": 42
  }
}

// Cursor/presence update
{
  "type": "presence",
  "payload": {
    "pointer": { "x": 150, "y": 320 },
    "button": "down",
    "selectedElementIds": { "elem_1": true },
    "username": "Zeeshan"
  }
}

// Initial scene sync (sent to new joiner)
{
  "type": "scene_init",
  "payload": {
    "elements": [ /* full encrypted scene */ ],
    "files": { /* encrypted file data */ }
  }
}
```

**POST /api/scenes (Cloud Save)**

```json
// Request
{
  "name": "Architecture Diagram",
  "elements": [ /* ExcalidrawElement[] */ ],
  "appState": { "viewBackgroundColor": "#ffffff", "gridSize": null },
  "files": { "img_abc": { "mimeType": "image/png", "dataURL": "data:image/png;base64,..." } }
}

// Response
{
  "id": "scene_xyz789",
  "shareUrl": "https://excalidraw.com/#json=scene_xyz789",
  "createdAt": "2026-03-17T10:00:00Z"
}
```

---

### 12.3 Error Handling and Status Codes

| Status | Scenario | Frontend Handling |
|--------|----------|-------------------|
| `200` | Success | Process response |
| `400` | Malformed scene data | Show error toast; scene too large or corrupted |
| `401` | Unauthorized | Redirect to sign-in (cloud features only) |
| `404` | Scene not found (deleted or expired link) | Show "This drawing no longer exists" page |
| `413` | Payload too large | Show "Scene too large for cloud storage. Export as file." |
| `429` | Rate limit | Back off auto-save; show "Saving paused" |
| `500` | Server error | Retry with backoff; continue local-only |
| WebSocket `close(1006)` | Abnormal disconnect | Auto-reconnect with exponential backoff |

---

## 13. Caching Strategy

### 13.1 What to Cache

*   **Scene data**: Primary storage is IndexedDB (not a cache — it's the source of truth for local-first).
*   **Image blobs**: Stored in IndexedDB; Object URLs cached in-memory for active session.
*   **Rough.js drawables**: Cached in-memory (WeakMap keyed by element reference). Regenerated only when element changes.
*   **Font files**: Cached by Service Worker / browser HTTP cache. Custom fonts (Virgil hand-drawn) are critical for rendering.
*   **Application bundle**: Cached by Service Worker for offline support.

### 13.2 Where to Cache

| Data | Cache Location | TTL |
|------|----------------|-----|
| Scene elements | IndexedDB (persistent) | Until user deletes |
| Image blobs | IndexedDB (persistent) + Object URL (in-memory) | Object URLs revoked on scene close; blobs LRU-evicted |
| Rough.js drawables | WeakMap (in-memory) | Automatically GC'd when element object changes |
| Font files (Virgil, etc.) | Service Worker cache + HTTP `immutable` | Forever (versioned filenames) |
| App shell (HTML, JS, CSS) | Service Worker cache | Until new version deployed |
| Collaboration scene (remote) | In-memory only | Ephemeral — only while connected |

### 13.3 Cache Invalidation

*   **Scene data**: No TTL — user explicitly deletes or data migrates on version upgrade.
*   **Image blobs**: LRU eviction when storage quota is approached. Orphaned files cleaned up when their parent scene is deleted.
*   **Rough.js cache**: Invalidated automatically via WeakMap when element is replaced (immutable state updates create new objects).
*   **Fonts**: Never invalidated (immutable filenames with content hash).
*   **App bundle**: Service Worker checks for updates on each load; stale-while-revalidate pattern.

---

## 14. CDN and Asset Optimization

*   **App delivery**: Static HTML + JS + CSS served from CDN. Very small payload (~500KB gzipped for the core app).
*   **Font delivery**: Custom hand-drawn font (Virgil) is ~100KB. Loaded with `font-display: swap` to avoid blocking render. Preloaded via `<link rel="preload" as="font">` since it is critical for canvas text rendering.
*   **No user content on CDN**: Drawings are stored locally (IndexedDB) or encrypted on the collaboration server. The CDN never serves user content.
*   **Image optimization**: Pasted images are stored as-is (no server-side processing). The app could optionally resize oversized images client-side before storing (cap at 4096px width, compress to JPEG 80% quality).
*   **Cache headers**:
    *   JS/CSS: `Cache-Control: public, max-age=31536000, immutable` (content-hashed filenames).
    *   HTML: `Cache-Control: no-cache` (always check for updates).
    *   Fonts: `Cache-Control: public, max-age=31536000, immutable`.
*   **Compression**: Brotli for all text assets.

---

## 15. Rendering Strategy

*   **Fully CSR** (Client-Side Rendered) — no SSR or SSG for the drawing canvas.
*   The HTML page is a **static shell** (minimal HTML with a `<canvas>` element and a script tag). It loads instantly from the CDN.
*   The JS bundle initializes the canvas renderer, opens IndexedDB, restores the scene, and renders.
*   **No hydration** — there is no server-rendered HTML to hydrate. The app boots entirely client-side.
*   **Code splitting**:
    *   **Core bundle** (~200KB gzipped): Canvas renderer, element operations, basic tools, IndexedDB persistence.
    *   **Collaboration chunk** (~80KB): WebSocket client, CRDT engine, encryption. Lazy-loaded only when user starts a collaboration session.
    *   **Export chunk** (~50KB): PNG/SVG export, canvas-to-blob conversion. Lazy-loaded when user opens export dialog.
    *   **Library panel chunk** (~30KB): Library browser and import UI. Lazy-loaded when user opens the library panel.
*   **Service Worker** for offline support: Caches the app shell and fonts, enabling full offline functionality. Scene data is already local (IndexedDB), so the app works without any network.

---

## 16. Cross Cutting Non Functional Concerns

### 16.1 Security

*   **End-to-end encryption (E2EE)**: All collaboration data is encrypted with AES-GCM using a key derived from the URL fragment. The server is a blind relay.
*   **No user data on server** (solo mode): IndexedDB stores data locally. No cookies, no accounts, no telemetry send.
*   **XSS prevention**: User-generated text is rendered on Canvas (not DOM), so XSS via injected HTML is not possible for canvas content. The properties panel and text editor sanitize inputs.
*   **Content Security Policy (CSP)**: Strict CSP: `script-src 'self'`; no inline scripts; `img-src 'self' blob: data:` (for embedded images).
*   **URL fragment privacy**: Encryption keys in URL fragments are never sent to the server (browser behavior). Warn users not to share links via unencrypted channels.
*   **Clipboard safety**: When pasting content, validate that it is a recognized image or Excalidraw JSON format before processing. Do not execute arbitrary pasted data.
*   **Library imports**: Libraries loaded from external URLs are validated against the expected schema before being stored in IndexedDB. No script execution from library data.

---

### 16.2 Accessibility

*   **Keyboard navigation**:
    *   Single-key shortcuts: `R` for rectangle, `E` for ellipse, `A` for arrow, `T` for text, `D` for diamond, `L` for line, `V` for selection.
    *   `Tab` cycles through elements on canvas. `Escape` deselects.
    *   `Ctrl+A` selects all, `Delete` removes selected.
    *   `Ctrl+Z` / `Ctrl+Shift+Z` for undo/redo.
    *   `Ctrl+D` duplicate, `Ctrl+G` group, `Ctrl+Shift+G` ungroup.
*   **Screen reader support**:
    *   Toolbar buttons have `aria-label` with descriptive names.
    *   Tool state communicates via `aria-pressed` / `aria-selected`.
    *   Canvas content is visual-only (not accessible to screen readers by nature). A textual summary of the scene could be provided via an `aria-live` region: "5 elements on canvas: 2 rectangles, 1 arrow, 2 text labels."
*   **Color contrast**: All toolbar icons and text meet WCAG AA contrast ratios. Dark mode respects system preference (`prefers-color-scheme`).
*   **Reduced motion**: `prefers-reduced-motion` disables any CSS transitions and animation effects in the UI panels.
*   **Touch and stylus**: Full support for touch drawing (mobile/tablet). Stylus pressure sensitivity is used for freehand drawing (`freeDrawElement.points` stores pressure values).

---

### 16.3 Performance Optimization

##### Canvas Performance

*   **Viewport culling**: Only render elements within the visible viewport bounds. A scene with 5000 elements but 20 visible only renders 20.
*   **Rough.js drawable caching**: WeakMap cache avoids regenerating hand-drawn path data every frame.
*   **Multi-layer canvas**: Static elements re-render only on data change. Interactive overlays (cursors, selection handles) re-render on pointer move.
*   **requestAnimationFrame batching**: Multiple state updates within a single frame produce only one render.
*   **`will-change: transform`** on the canvas container for GPU-accelerated pan/zoom.
*   **OffscreenCanvas** (Web Worker rendering): For very large scenes, render the static canvas in a Web Worker using `OffscreenCanvas` to free the main thread for event handling. (Progressive enhancement — not all browsers support it.)

##### Application Performance

*   **Code splitting**: Collaboration, export, and library modules are lazy-loaded.
*   **Font preloading**: Custom hand-drawn font preloaded to avoid Flash of Unstyled Text (FOUT) when rendering text elements.
*   **IndexedDB non-blocking**: All persistence is async and debounced. Never blocks rendering.
*   **Debounced auto-save**: 300ms debounce prevents excessive IndexedDB writes during rapid editing.
*   **Abort stale operations**: If user starts a new drawing action mid-export, abort the export rendering.
*   **Memory management**: Object URLs for images are revoked when no longer needed. Old history entries are dropped when stack reaches max size.

##### Performance Measurements

| Metric | Target | How |
|--------|--------|-----|
| App load (first paint) | < 1.5s | CDN-served static shell + minimal critical JS |
| Scene restore from IndexedDB | < 200ms for 500 elements | Async read + batch render |
| Canvas render frame | < 16ms (60fps) | Viewport culling + drawable cache |
| Auto-save write | < 50ms for 500 elements | Debounced IndexedDB transaction |
| Collaboration latency (cursor) | < 100ms perceived | Throttled WebSocket at 30fps |

---

### 16.4 Observability and Reliability

*   **Error Boundaries**: Wrap the canvas and toolbar in separate error boundaries. If the toolbar crashes, the canvas remains functional (and vice versa).
*   **IndexedDB error handling**: If IndexedDB is unavailable (private browsing in some browsers, storage quota exceeded), fall back to in-memory only with a warning banner.
*   **Logging**:
    *   Track scene size (element count), IndexedDB storage used, and collaboration room metrics.
    *   Log rendering performance (frame times > 16ms).
    *   Log auto-save failures and IndexedDB errors.
*   **Performance monitoring**: Track:
    *   Time from page load to first canvas render.
    *   Time from IndexedDB open to scene restore complete.
    *   Average frame render time during active drawing.
    *   Collaboration round-trip latency (send element → receive echo).
*   **Feature flags**: Gate new features:
    *   New tools, canvas rendering optimizations, OffscreenCanvas.
    *   A/B test auto-save intervals.
*   **Graceful degradation**:
    *   If WebSocket fails → show "Collaboration unavailable" but local drawing continues.
    *   If IndexedDB fails → in-memory mode with export-to-file as backup persistence.
    *   If Rough.js fails → fall back to clean/geometric rendering (no hand-drawn style).

---

## 17. Edge Cases and Tradeoffs

| Edge Case | Handling |
|-----------|---------|
| **Browser crashes mid-save** | IndexedDB transactions are atomic — either the full save completes or nothing is written. The last successful save is preserved. |
| **Very large scene (5000+ elements)** | Viewport culling ensures only visible elements render. Consider paginating IndexedDB writes (save in chunks of 500 elements). |
| **Private browsing (no IndexedDB)** | Some browsers restrict IndexedDB in private mode. Detect this on startup and warn the user: "Your drawing will not be saved. Use regular browsing or export to file." |
| **Multiple browser tabs editing same scene** | IndexedDB is shared across tabs. Use `BroadcastChannel` to sync changes between tabs, or use a `versionchange` event to warn the user. |
| **User pastes a 20MB image** | Validate size before storing. Cap at 10MB, compress if possible, or reject with a clear error message. |
| **Collaborator has slow connection** | Throttle scene sync to avoid overwhelming their connection. Buffer changes and send batches. Show "Syncing..." indicator. |
| **Clock skew between collaborators** | Use `version` + `versionNonce` (logical clock) instead of wall-clock timestamps for conflict resolution. |
| **User draws while offline and then goes online** | Queue all local changes. On reconnect, send full local state. CRDT merge resolves conflicts with any changes made by others during the offline period. |
| **IndexedDB storage quota exceeded** | Detect via `DOMException: QuotaExceededError`. Trigger LRU eviction of old image blobs. Warn user. Suggest exporting. |
| **Scene JSON import from older version** | Run element migration pipeline on import. Invalid elements are logged and skipped (not crash the entire import). |
| **Undo across collaboration boundary** | Undo only affects local user's actions. If User A undoes a move, it creates a new version of the element (not a revert of User B's concurrent changes). |
| **Rapid tool switching during draw** | Only the active tool's drawing is committed. Switching tools mid-draw cancels the in-progress shape without committing. |

### Key Tradeoffs

| Decision | Tradeoff |
|----------|----------|
| **Canvas 2D over SVG** | Excellent performance at scale, but no built-in DOM accessibility for shape content. Hit-testing is manual. |
| **IndexedDB over localStorage** | Supports large scenes and binary blobs, but API is complex (async, transactions, cursors). Wrapped in a simpler utility layer. |
| **Local-first over cloud-first** | Zero-friction onboarding (no sign-up), works offline, privacy-respecting. But no cross-device sync without explicit sharing. |
| **LWW CRDT over OT** | Simple to implement, deterministic conflict resolution. But concurrent edits on the same element cause one user's change to be lost. |
| **Element-level conflict resolution over property-level** | Much simpler merge logic. But if two users change different properties of the same element simultaneously, one change is lost. |
| **Snapshot-based undo over pure command pattern** | Simpler to implement, handles grouped operations naturally. But uses more memory (mitigated by storing only changed elements, not full scene). |
| **E2E encryption** | Full privacy — server cannot read data. But makes server-side features harder (search, thumbnails, indexing). |
| **Debounced auto-save (300ms)** | Good balance of safety and performance. But user can lose up to 300ms of work on a crash. Acceptable because the last committed action was < 300ms ago. |
| **Custom rendering engine over a library (Fabric.js, Konva)** | Full control over performance and rendering style. But higher maintenance burden — every feature (hit-testing, bounds calculation, export) must be implemented. |
| **Blob storage over base64 in IndexedDB** | 33% smaller, native IO, no encode/decode overhead. But requires Object URL management (create/revoke lifecycle). |

---

## 18. Summary and Future Improvements

### Key Architectural Decisions

1.  **HTML5 Canvas 2D with Rough.js** — immediate-mode rendering with hand-drawn style. Multi-layer architecture separates static scene from interactive overlays.
2.  **IndexedDB as primary storage** — local-first architecture with auto-save, Blob storage for images, versioned schema with migrations, and LRU eviction for storage management.
3.  **CRDT-based real-time collaboration** — last-writer-wins per element with deterministic tie-breaking. Encrypted WebSocket relay with E2E encryption.
4.  **Viewport culling + Rough.js caching** — renders only visible elements; caches hand-drawn path data to avoid recalculation.
5.  **Hybrid snapshot undo/redo** — stores only changed elements per history entry, balancing memory and simplicity.
6.  **Zero-friction local-first UX** — no sign-in, no save button, no server dependency for solo drawing. Cloud features are optional layers.

### Possible Future Enhancements

*   **OffscreenCanvas rendering**: Move static scene rendering to a Web Worker to keep the main thread free for interactions. Progressive enhancement for browsers that support it.
*   **OPFS for large files**: Use the Origin Private File System for high-performance storage of embedded images larger than 5MB, with IndexedDB for metadata.
*   **Operational Transform (OT) collaboration**: For property-level conflict resolution instead of element-level, enabling truly concurrent edits on the same element (e.g., one user moves while another recolors).
*   **Version history**: Store snapshots of the scene at intervals, allowing users to browse and restore past versions (like Google Docs version history).
*   **WebAssembly rendering**: Port the rendering engine to WASM (Rust + wgpu) for GPU-accelerated rendering of very large scenes (10,000+ elements).
*   **Plugin system**: Allow third-party extensions to add custom tools, shape types, and integrations via a sandboxed plugin API.
*   **Cross-device sync via CRDTs**: Combine local IndexedDB storage with a cloud CRDT store (like Automerge or Yjs) for seamless cross-device sync without a centralized server.
*   **AI-powered features**: Converting hand-drawn wireframes to structured UI components; auto layout and alignment suggestions via ML models.

---

### Endpoint Summary

| Endpoint | Method | Description |
|----------|--------|-------------|
| `wss://collab.excalidraw.com/:roomId` | WebSocket | Real-time collaboration relay |
| `/api/scenes` | POST | Save scene to cloud |
| `/api/scenes/:id` | GET | Load cloud-saved scene |
| `/api/scenes/:id` | DELETE | Delete cloud-saved scene |
| `/api/share` | POST | Create shareable snapshot link |
| `/api/share/:id` | GET | Load shared scene (read-only) |
| `/api/libraries` | GET | Fetch libraries |

---

### Complete Data Flow Summary

| Direction | Mechanism | Trigger | Target | Action |
|-----------|-----------|---------|--------|--------|
| Initial Load | IndexedDB read | App mounts | IndexedDB → State → Canvas | Restore last scene |
| Drawing | State update | User draws | State → Canvas + IndexedDB (debounced) | Render + persist |
| Undo/Redo | History stack | Ctrl+Z / Ctrl+Shift+Z | History → State → Canvas + IndexedDB | Restore previous state |
| Auto Save | IndexedDB write | Debounced (300ms) | State → IndexedDB | Persist current scene |
| Collaboration (send) | WebSocket | Local state change | State → Encrypt → WebSocket | Broadcast to room |
| Collaboration (receive) | WebSocket | Remote message | WebSocket → Decrypt → CRDT merge → State → Canvas | Merge remote changes |
| Export | Canvas render | User exports | State → OffscreenCanvas → Blob → Download | Generate PNG/SVG/JSON |
| Import | File read | User imports file | File → Parse → State → Canvas + IndexedDB | Load external scene |

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design
