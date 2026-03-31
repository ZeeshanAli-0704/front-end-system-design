# Frontend System Design: Google Docs (Collaborative Document Editor)

- [Frontend System Design: Google Docs (Collaborative Document Editor)](#frontend-system-design-google-docs-collaborative-document-editor)
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
    - [5.2 Rich Text Editor Engine (Deep Dive)](#52-rich-text-editor-engine-deep-dive)
      - [5.2.1 Why ContentEditable vs Custom Rendering](#521-why-contenteditable-vs-custom-rendering)
      - [5.2.2 Document Model (Structured Data, Not HTML)](#522-document-model-structured-data-not-html)
      - [5.2.3 Selection and Cursor Model](#523-selection-and-cursor-model)
      - [5.2.4 Input Handling Pipeline](#524-input-handling-pipeline)
      - [5.2.5 Rendering Pipeline (Model to View)](#525-rendering-pipeline-model-to-view)
      - [5.2.6 Block and Inline Formatting](#526-block-and-inline-formatting)
      - [5.2.7 Embedded Content (Images, Tables, Embeds)](#527-embedded-content-images-tables-embeds)
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
  - [9. Real Time Collaboration Deep Dive](#9-real-time-collaboration-deep-dive)
    - [9.1 The Fundamental Problem of Concurrent Editing](#91-the-fundamental-problem-of-concurrent-editing)
    - [9.2 Operational Transformation (OT)](#92-operational-transformation-ot)
      - [9.2.1 What is an Operation](#921-what-is-an-operation)
      - [9.2.2 The Transform Function](#922-the-transform-function)
      - [9.2.3 OT with a Central Server](#923-ot-with-a-central-server)
      - [9.2.4 Complete OT Client Implementation](#924-complete-ot-client-implementation)
      - [9.2.5 Server Side OT Orchestration](#925-server-side-ot-orchestration)
    - [9.3 CRDTs (Conflict Free Replicated Data Types)](#93-crdts-conflict-free-replicated-data-types)
      - [9.3.1 How CRDTs Differ from OT](#931-how-crdts-differ-from-ot)
      - [9.3.2 Sequence CRDTs for Text (Yjs Example)](#932-sequence-crdts-for-text-yjs-example)
      - [9.3.3 CRDT Integration with Editor](#933-crdt-integration-with-editor)
    - [9.4 OT vs CRDT Comparison](#94-ot-vs-crdt-comparison)
    - [9.5 Cursor and Selection Presence](#95-cursor-and-selection-presence)
    - [9.6 WebSocket Transport Layer](#96-websocket-transport-layer)
    - [9.7 Offline Editing and Reconnection](#97-offline-editing-and-reconnection)
  - [10. Undo Redo in Collaborative Context](#10-undo-redo-in-collaborative-context)
  - [11. High Level API Design (Frontend POV)](#11-high-level-api-design-frontend-pov)
    - [11.1 Required APIs](#111-required-apis)
    - [11.2 Request and Response Structure](#112-request-and-response-structure)
    - [11.3 Error Handling and Status Codes](#113-error-handling-and-status-codes)
  - [12. Caching Strategy](#12-caching-strategy)
  - [13. CDN and Asset Optimization](#13-cdn-and-asset-optimization)
  - [14. Rendering Strategy](#14-rendering-strategy)
  - [15. Cross Cutting Non Functional Concerns](#15-cross-cutting-non-functional-concerns)
    - [15.1 Security](#151-security)
    - [15.2 Accessibility](#152-accessibility)
    - [15.3 Performance Optimization](#153-performance-optimization)
    - [15.4 Observability and Reliability](#154-observability-and-reliability)
  - [16. Edge Cases and Tradeoffs](#16-edge-cases-and-tradeoffs)
  - [17. Summary and Future Improvements](#17-summary-and-future-improvements)

---

## 1. Concept, Idea, Product Overview

### 1.1 Product Description

*   Google Docs is a **web-based collaborative document editor** that allows multiple users to create, edit, and format rich-text documents simultaneously in real time.
*   It is the defining example of **real-time collaboration** on the web — every keystroke from every user is reflected in all other users' views within milliseconds.
*   Target users: anyone who writes — students, professionals, writers, teams, enterprises. Over 1 billion users.
*   Primary use case: authoring and co-editing rich-text documents (reports, articles, meeting notes, proposals) with real-time multi-user collaboration, commenting, and version history.

---

### 1.2 Key User Personas

*   **Solo Writer**: Creates and edits documents alone. Expects fast, responsive typing, rich formatting (headings, lists, tables, images), and auto-save. May work across devices.
*   **Collaborative Team Member**: Edits documents simultaneously with 2-20 team members. Expects real-time cursor visibility, conflict-free concurrent editing, commenting/suggesting workflow, and version history.
*   **Reviewer / Commenter**: Opens shared documents in "Suggesting" or "Viewing" mode. Reads content, leaves comments, suggests edits (tracked changes), and resolves comment threads.

---

### 1.3 Core User Flows (High Level)

*   **Editing a Document (Primary Flow)**:
    1.  User opens a document URL → document loads with the latest content.
    2.  User types text → characters appear instantly at the cursor position.
    3.  Every edit is auto-saved to the server (no save button).
    4.  If other users are present, their cursors are visible in real time with names and colors.
    5.  User applies formatting (bold, heading, list) → formatting is applied immediately.
    6.  All changes from all users are merged in real time without conflicts.

*   **Commenting and Suggesting (Secondary Flow)**:
    1.  User selects text → clicks "Add comment" or switches to "Suggesting" mode.
    2.  Comment appears in the right margin, linked to the highlighted text.
    3.  Other users see the comment/suggestion in real time and can reply or resolve it.

*   **Version History (Secondary Flow)**:
    1.  User opens "Version history" panel.
    2.  Sees a timeline of all changes grouped by author and time.
    3.  Can preview any past version and restore it.

---

## 2. Requirements

### 2.1 Functional Requirements

*   **Rich Text Editing**:
    *   Type, delete, copy, paste, cut text with full Unicode support.
    *   Inline formatting: bold, italic, underline, strikethrough, code, links, text color, highlight.
    *   Block formatting: headings (H1–H6), paragraphs, ordered/unordered lists, blockquotes, code blocks.
    *   Embedded content: images, tables, horizontal rules, page breaks.
*   **Real Time Collaboration**:
    *   Multiple users editing the same document simultaneously.
    *   Each user sees all others' changes in real time (sub-second latency).
    *   Cursor/selection presence: see where each collaborator's cursor is and what they've selected, with a color-coded name label.
    *   Conflict-free concurrent edits: two users typing at the same position must not lose either's work.
*   **Comments and Suggestions**:
    *   Add comments on selected text ranges.
    *   Threaded comment replies.
    *   "Suggesting" mode: edits appear as tracked changes that can be accepted or rejected.
*   **Auto Save and Version History**:
    *   All changes are automatically saved — no save button.
    *   Full version history with timestamps and author attribution.
    *   Ability to name versions and restore any previous state.
*   **Sharing and Permissions**:
    *   Share via link with permission levels: Viewer, Commenter, Editor.
    *   Real-time permission enforcement (viewer cannot edit).
*   **Export and Import**:
    *   Export as PDF, DOCX, plain text, HTML.
    *   Import from DOCX, plain text, HTML.

---

### 2.2 Non Functional Requirements

*   **Performance**: Typing must feel instantaneous (< 16ms input-to-screen latency for local edits). Document load < 2s for a 50-page document. Support documents with 100+ pages without sluggish scrolling.
*   **Scalability**: Support documents with 100+ pages, 50+ simultaneous editors, and 10,000+ revision history entries. Handle paste operations of 10,000+ characters seamlessly.
*   **Availability**: Offline editing support with sync on reconnect. Graceful degradation when the collaboration server is down — user can continue editing locally.
*   **Security**: Access control enforced at API level (viewer cannot send edit operations). All data in transit is TLS encrypted. Content is not stored in the frontend beyond the active session.
*   **Accessibility**: Full keyboard navigation. Screen reader support for document content (semantic DOM). WCAG AA contrast. Focus management in toolbars and dialogs.
*   **Device Support**: Desktop web (primary), tablet web, mobile web (basic editing). Chrome, Firefox, Safari, Edge.
*   **i18n**: Full RTL support (Arabic, Hebrew) including bidirectional text within documents. IME (Input Method Editor) support for CJK languages. Localized UI in 100+ languages.

---

## 3. Scope Clarification (Interview Scoping)

### 3.1 In Scope

*   Rich text editor architecture (document model, rendering, input handling).
*   **Real-time collaboration deep dive** (Operational Transformation, CRDTs, conflict resolution).
*   Cursor and selection presence across collaborators.
*   Auto-save and sync pipeline.
*   Document data model and state management.
*   Performance optimization for large documents.
*   Undo/redo in a multi-user context.

---

### 3.2 Out of Scope

*   Backend database and storage (we assume APIs exist).
*   Detailed permission system / sharing dialog.
*   Rich inline features (smart chips, @mentions autocomplete, spell check).
*   Add-ons and extensions marketplace.
*   Native mobile app.
*   Print/PDF rendering pipeline.

---

### 3.3 Assumptions

*   User is authenticated; auth token is available.
*   A WebSocket-based collaboration server exists (we design the frontend interaction).
*   Server provides an OT/CRDT-compatible API for document operations.
*   Documents are fetched as structured JSON (not raw HTML).
*   Media (images) are uploaded to a separate service and referenced by URL.

---

## 4. High Level Frontend Architecture

### 4.1 Overall Approach

*   **SPA** (Single Page Application) with client-side routing.
*   **SSR** for the document shell and initial content — server renders the page frame + first viewport of content for fast FCP. Subsequent editing is fully CSR.
*   The editor is a **custom rendering engine** — does NOT rely on browser's native `contentEditable` directly for document model management (only for input capture).
*   **Monolith frontend** — the editor is highly integrated; micro-frontends add unnecessary complexity for a deeply coupled editing experience.

---

### 4.2 Major Architectural Layers

```
┌──────────────────────────────────────────────────────────────┐
│  UI Layer                                                    │
│  ┌───────────────────────┐  ┌─────────────────────────────┐  │
│  │ Toolbar / Menu Bar    │  │ Editor Surface              │  │
│  │ (formatting, actions) │  │  ┌──────────────────────┐   │  │
│  ├───────────────────────┤  │  │ Document Content     │   │  │
│  │ Comment Sidebar       │  │  │ (rendered from model)│   │  │
│  │ (threads, replies)    │  │  ├──────────────────────┤   │  │
│  ├───────────────────────┤  │  │ Cursor Overlays      │   │  │
│  │ Version History Panel │  │  │ (remote cursors)     │   │  │
│  │ (timeline, preview)   │  │  ├──────────────────────┤   │  │
│  ├───────────────────────┤  │  │ Selection Highlights │   │  │
│  │ Permissions Banner    │  │  │ (remote selections)  │   │  │
│  │ (viewer / editor)     │  │  └──────────────────────┘   │  │
│  └───────────────────────┘  └─────────────────────────────┘  │
├──────────────────────────────────────────────────────────────┤
│  Document Model Layer                                        │
│  (Tree / flat structure of nodes: paragraphs, text runs,     │
│   inline marks, block types, embedded objects)               │
├──────────────────────────────────────────────────────────────┤
│  Editor Core (Command / Transaction Layer)                   │
│  (Operations: insert, delete, format, split, merge.          │
│   Selection model, Schema validation, Input handling)        │
├──────────────────────────────────────────────────────────────┤
│  Collaboration Layer                                         │
│  (OT / CRDT engine, WebSocket transport,                     │
│   Presence manager, Conflict resolution)                     │
├──────────────────────────────────────────────────────────────┤
│  Persistence Layer                                           │
│  (Auto-save manager, Version history client,                 │
│   Offline buffer, Export/Import)                             │
├──────────────────────────────────────────────────────────────┤
│  Shared / Utility Layer                                      │
│  (Keyboard manager, Clipboard handler, IME handler,          │
│   Selection utilities, Analytics tracker)                    │
└──────────────────────────────────────────────────────────────┘
```

---

### 4.3 External Integrations

*   **Collaboration Server**: WebSocket-based server that receives operations from all clients, transforms them (OT), and broadcasts the transformed operations to all other clients.
*   **Document Storage API**: REST API for loading/saving document snapshots, version history, and metadata.
*   **Media Upload Service**: Separate endpoint for image/file uploads; returns a URL for embedding.
*   **Comment Service**: REST + real-time API for comments and suggestion threads.
*   **Analytics SDK**: Track editing session duration, collaboration metrics, feature usage.
*   **Spell Check / Grammar Service**: Server-side NLP service called asynchronously with document text.

---

## 5. Component Design and Modularization

### 5.1 Component Hierarchy

```
DocumentPage
 ├── MenuBar
 │    ├── FileMenu (new, open, import, export, share)
 │    ├── EditMenu (undo, redo, cut, copy, paste, find/replace)
 │    ├── ViewMenu (zoom, page break, ruler)
 │    └── InsertMenu (image, table, link, heading, page break)
 │
 ├── Toolbar
 │    ├── UndoRedoButtons
 │    ├── FontSelector
 │    ├── FontSizeSelector
 │    ├── InlineFormatButtons (bold, italic, underline, strikethrough, code)
 │    ├── TextColorPicker
 │    ├── HighlightColorPicker
 │    ├── AlignmentButtons (left, center, right, justify)
 │    ├── ListButtons (ordered, unordered, checklist)
 │    ├── IndentButtons (indent, outdent)
 │    └── InsertButtons (link, image, table, comment)
 │
 ├── EditorSurface (main editing area)
 │    ├── PageContainer (simulates paper page)
 │    │    ├── BlockNode[] (rendered from document model)
 │    │    │    ├── ParagraphBlock
 │    │    │    │    └── TextRun[] (spans with inline marks)
 │    │    │    ├── HeadingBlock
 │    │    │    ├── ListBlock
 │    │    │    │    └── ListItem[]
 │    │    │    ├── CodeBlock
 │    │    │    ├── BlockquoteBlock
 │    │    │    ├── TableBlock
 │    │    │    │    └── TableRow[] → TableCell[]
 │    │    │    └── ImageBlock
 │    │    │
 │    │    ├── CursorOverlay (local cursor + remote cursors with name labels)
 │    │    └── SelectionOverlay (colored highlights for remote selections)
 │    │
 │    └── FloatingToolbar (appears on text selection — quick format options)
 │
 ├── CommentSidebar
 │    └── CommentThread[]
 │         ├── CommentBody
 │         ├── CommentReplies
 │         └── ResolveButton
 │
 ├── VersionHistoryPanel (slide-in from right)
 │    ├── VersionTimeline
 │    └── VersionPreview
 │
 └── CollaboratorAvatars (top-right — shows who's online)
```

---

### 5.2 Rich Text Editor Engine (Deep Dive)

Building a collaborative rich text editor is one of the most **complex frontend engineering challenges**. The editor must:
*   Capture user input reliably across all browsers, IME systems, and operating systems.
*   Maintain a structured document model that is independent of the DOM.
*   Render the document model to DOM efficiently.
*   Support concurrent editing from multiple users without conflicts.
*   Handle undo/redo correctly in a multi-user context.

---

#### 5.2.1 Why ContentEditable vs Custom Rendering

Every rich text editor on the web must decide how to capture user input and render content. There are three fundamental approaches:

| Approach | How it works | Used by | Pros | Cons |
|----------|-------------|---------|------|------|
| **Full `contentEditable`** | Set `contentEditable="true"` on a div. Let the browser handle all input, rendering, selection, and formatting. | Simple WYSIWYG editors (Quill v1, Medium's first editor) | Zero effort for basic editing; browser handles IME, spell check, autocomplete natively | Wildly inconsistent across browsers; DOM is the model (fragile); very hard to add custom behavior; nightmare for collaboration |
| **ContentEditable for input, custom model for data** | Use `contentEditable` to capture keystrokes and IME input. But maintain a separate structured data model. On input, translate the DOM mutation into a model operation, then re-render the model back to DOM. | Google Docs, Slate.js, ProseMirror, TipTap | Input capture is reliable (leverages browser's IME/spell check); model is clean and structured; collaboration-friendly | Must keep DOM and model in sync; complex input pipeline; occasional browser quirks in contentEditable |
| **Custom input (Canvas / hidden textarea)** | Render the document on a `<canvas>` element. Capture input via a hidden `<textarea>` or custom keyboard handler. Bypass `contentEditable` entirely. | Google Docs (2021+ Canvas-based renderer), Figma (for text editing) | Full control over rendering and layout; pixel-perfect consistency; no browser DOM quirks | Must implement ALL text rendering (line breaking, BiDi, ligatures, caret blinking, selection rectangles); loses native spell check, screen readers need extra work; massive engineering effort |

**Strategy for this design: ContentEditable for input + custom document model.**
This is the approach used by ProseMirror, Slate.js, and TipTap — and the original Google Docs (before the 2021 Canvas switch). It offers the best balance of reliability, collaboration support, and engineering effort.

**Notable: Google Docs moved to Canvas-based rendering in 2021.** This allowed them to own the entire rendering pipeline (consistent across all browsers, better performance for large documents, custom features like paginated layout). However, this is a massive engineering investment and is out of scope for our design. We'll use the DOM-based approach.

---

#### 5.2.2 Document Model (Structured Data, Not HTML)

The editor does NOT use HTML as its data model. Instead, it maintains a **structured tree of typed nodes** that represents the document content independently of how it's displayed.

**Why not use the DOM as the model?**

| Aspect | DOM as model | Custom structured model |
|--------|-------------|------------------------|
| Data shape | Unpredictable HTML (browser adds random `<span>`, `<font>`, `<br>` tags) | Clean, typed, predictable JSON tree |
| Serialization | Must parse/sanitize HTML | `JSON.stringify()` — trivial |
| Collaboration | How do you diff arbitrary HTML? Very hard | Operations on typed nodes are well-defined |
| Undo/redo | Browser's built-in undo is unreliable across browsers | Custom command stack with full control |
| Validation | Can the DOM contain a heading inside a list item? No easy way to enforce | Schema defines allowed structures; invalid changes are rejected |
| Cross-platform | Different browsers produce different HTML for the same user action | Model is browser-agnostic |

##### Document Tree Structure

```
Document
 └── Block[]
      ├── Paragraph
      │    └── TextRun[]
      │         ├── { text: "Hello ", marks: [] }
      │         ├── { text: "world", marks: ["bold"] }
      │         └── { text: "!", marks: [] }
      │
      ├── Heading (level: 2)
      │    └── TextRun[]
      │         └── { text: "Introduction", marks: [] }
      │
      ├── BulletList
      │    └── ListItem[]
      │         ├── ListItem
      │         │    └── Paragraph → TextRun[]
      │         └── ListItem
      │              └── Paragraph → TextRun[]
      │
      ├── CodeBlock
      │    └── { text: "const x = 1;", language: "javascript" }
      │
      ├── Table
      │    └── TableRow[]
      │         └── TableCell[]
      │              └── Block[] (paragraphs inside cells)
      │
      └── Image
           └── { src: "...", alt: "...", width: 400, height: 300 }
```

##### JSON Representation

```json
{
  "type": "doc",
  "content": [
    {
      "type": "paragraph",
      "content": [
        { "type": "text", "text": "Hello ", "marks": [] },
        { "type": "text", "text": "world", "marks": [{ "type": "bold" }] },
        { "type": "text", "text": "!", "marks": [] }
      ]
    },
    {
      "type": "heading",
      "attrs": { "level": 2 },
      "content": [
        { "type": "text", "text": "Introduction", "marks": [] }
      ]
    },
    {
      "type": "bulletList",
      "content": [
        {
          "type": "listItem",
          "content": [
            {
              "type": "paragraph",
              "content": [
                { "type": "text", "text": "First item", "marks": [] }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

This model is:
*   **Framework-agnostic** — just data; can be rendered by React, vanilla JS, or even Canvas.
*   **Serializable** — `JSON.stringify()` for saving; `JSON.parse()` for loading.
*   **Diffable** — operations transform this model, and diffs can be computed for collaboration.
*   **Validatable** — a schema defines what node types exist and where they can appear.

---

#### 5.2.3 Selection and Cursor Model

The browser's native `Selection` API is **position-based** (anchored to DOM nodes + offsets). Our editor needs a model-based selection — positions that reference the **document model**, not the DOM.

##### Model Position

A position in the document model is a **numerical offset** counting from the start of the document. Every "step" into or out of a node counts as one position:

```
Content:  "Hello world"
Position: 0     1     2     3     4     5     6     7     8     9     10    11

If the content is: <p>Hello</p><p>World</p>

Model positions:
  0: before <p>
  1: before 'H'
  2: before 'e'
  3: before 'l'
  4: before 'l'
  5: before 'o'
  6: after 'o' / after </p>
  7: before <p>
  8: before 'W'
  ...

A selection is: { anchor: 2, head: 5 }  →  selects "ello"
A cursor is:    { anchor: 5, head: 5 }  →  cursor after "Hello"
```

This model-based selection is:
*   **Independent of the DOM** — if the DOM re-renders, the model position is still valid.
*   **Shareable** — we can send `{ anchor: 5, head: 5 }` to other collaborators to show our cursor.
*   **Transformable** — when a remote user inserts text before position 5, we can adjust our position to 5 + insertLength.

```tsx
type EditorSelection = {
  anchor: number;  // where the selection started (user pressed mouse down)
  head: number;    // where the selection ended (user released mouse)
};

// Cursor is a collapsed selection
type Cursor = EditorSelection; // where anchor === head
```

---

#### 5.2.4 Input Handling Pipeline

The input pipeline converts raw browser events into document model operations. This is the most challenging part of editor development because of IME (Input Method Editor), autocorrect, speech-to-text, and browser inconsistencies.

```
Browser event (keydown, input, compositionstart/end, paste, drop)
     ↓
Input Handler (intercepts and classifies)
     ↓
┌─────────────────────────────────────────────────────────┐
│  Classification:                                         │
│  - Regular character input → create InsertText op        │
│  - Backspace/Delete → create DeleteText op               │
│  - Enter → create SplitBlock op                          │
│  - Tab in list → create IndentListItem op                │
│  - Ctrl+B → create ToggleMark("bold") op                 │
│  - Paste → parse clipboard → create InsertSlice op       │
│  - IME composition → buffer until compositionend         │
│  - Drop → parse drag data → create InsertSlice op        │
└─────────────────────────────────────────────────────────┘
     ↓
Create Transaction (one or more operations)
     ↓
Apply Transaction to Document Model
     ↓
Emit to Collaboration Layer (send operation to server)
     ↓
Re-render affected DOM nodes (view update)
     ↓
Update browser selection to match model selection
```

##### Why `beforeinput` Event is Critical

Modern editors rely on the `beforeinput` event (InputEvent Level 2) instead of `keydown`:

| Event | What it tells you | Why it matters |
|-------|-------------------|---------------|
| `keydown` | Which physical key was pressed | Doesn't tell you what character will be produced (IME, dead keys, compose) |
| `input` | The DOM was already mutated | Too late — the DOM changed before you could intercept |
| **`beforeinput`** | What the browser **intends to do** AND you can **prevent** it | You can intercept, create your own model operation, and cancel the default DOM mutation |

```tsx
editorElement.addEventListener('beforeinput', (event: InputEvent) => {
  // Prevent the browser from mutating the DOM directly
  event.preventDefault();

  switch (event.inputType) {
    case 'insertText':
      // User typed a character
      applyOperation(insertText(selection, event.data!));
      break;

    case 'insertParagraph':
      // User pressed Enter
      applyOperation(splitBlock(selection));
      break;

    case 'deleteContentBackward':
      // User pressed Backspace
      applyOperation(deleteBackward(selection));
      break;

    case 'deleteContentForward':
      // User pressed Delete
      applyOperation(deleteForward(selection));
      break;

    case 'insertFromPaste':
      // User pasted content
      const html = event.dataTransfer?.getData('text/html');
      const text = event.dataTransfer?.getData('text/plain');
      applyOperation(insertSlice(selection, parseClipboard(html, text)));
      break;

    case 'formatBold':
      applyOperation(toggleMark('bold', selection));
      break;

    // ... other inputTypes
  }
});
```

##### IME (Input Method Editor) Handling

For CJK (Chinese, Japanese, Korean) languages, users type in a **composition mode** where multiple keystrokes produce a single character. During composition:

```
User types "ni" (Japanese IME for "に")
  compositionstart → browser shows inline composition UI
  compositionupdate → "n" appears, then "ni", then "に"
  compositionend → final character "に" is committed

During composition, we must NOT process individual keystrokes.
We buffer the composition and only apply the final result.
```

```tsx
let isComposing = false;

editorElement.addEventListener('compositionstart', () => {
  isComposing = true;
  // Let the browser handle composition rendering natively
});

editorElement.addEventListener('compositionend', (event) => {
  isComposing = false;
  // Now apply the final composed text as a single operation
  applyOperation(insertText(selection, event.data));
});

// In beforeinput handler, skip input during composition
editorElement.addEventListener('beforeinput', (event) => {
  if (isComposing) return; // Let browser handle it
  event.preventDefault();
  // ... handle normally
});
```

---

#### 5.2.5 Rendering Pipeline (Model to View)

After every edit, the document model changes and the DOM must be updated to reflect it. This is NOT a full re-render — it's a **surgical DOM update** that patches only the changed nodes.

```
Document Model (after edit)
     ↓
Diff with previous model (which nodes changed?)
     ↓
For each changed node:
  1. Find the corresponding DOM node
  2. Update its content / attributes
  3. If structure changed (node added/removed), create/remove DOM nodes
     ↓
Update browser selection to match model selection
```

##### Why Not Use React for the Editor Content?

| Approach | How it works | Problem |
|----------|-------------|---------|
| React-rendered content | Each paragraph / text run is a React component | React's reconciliation batches updates and defers them. Typing feels laggy if the DOM update waits for React's scheduler. Also, React manages the DOM tree — but `contentEditable` expects to manage it too. Conflicts arise (React removes a node that the browser selection is pointing to → crash). |
| **Custom DOM updates** | Editor kernel directly manipulates DOM via `document.createElement`, `textNode.data = ...` | Immediate, synchronous DOM updates = zero input latency. Full control over what changes and when. |

**ProseMirror, Slate.js, and Google Docs all use custom DOM updating, not a framework's virtual DOM.**

The rendering pipeline for a paragraph:

```tsx
function renderParagraph(block: ParagraphNode, existingDom?: HTMLElement): HTMLElement {
  const p = existingDom || document.createElement('p');

  // Clear and rebuild text runs (optimized: only update changed runs)
  while (p.firstChild) p.removeChild(p.firstChild);

  for (const run of block.content) {
    const span = document.createElement('span');
    span.textContent = run.text;

    // Apply inline marks as CSS classes or inline styles
    for (const mark of run.marks) {
      switch (mark.type) {
        case 'bold':
          span.style.fontWeight = 'bold';
          break;
        case 'italic':
          span.style.fontStyle = 'italic';
          break;
        case 'link':
          const a = document.createElement('a');
          a.href = mark.attrs.href;
          a.textContent = run.text;
          // Replace span with anchor
          p.appendChild(a);
          continue;
        // ...
      }
    }
    p.appendChild(span);
  }

  return p;
}
```

In practice, a diffing algorithm identifies which text runs changed and only updates those DOM nodes, avoiding full paragraph re-creation on every keystroke.

---

#### 5.2.6 Block and Inline Formatting

##### Inline Marks (Applied to Text Runs)

Marks are formatting attributes attached to text ranges. A single character can have multiple marks (bold + italic + link).

```
Text: "Hello bold world"
Marks: [
  { type: "text", text: "Hello ", marks: [] },
  { type: "text", text: "bold", marks: [{ type: "bold" }] },
  { type: "text", text: " world", marks: [] },
]
```

When the user selects "bold" and presses Ctrl+I (italic):

```
Before: { text: "bold", marks: [{ type: "bold" }] }
After:  { text: "bold", marks: [{ type: "bold" }, { type: "italic" }] }
```

##### Block Types (Applied to Structural Nodes)

Block operations change the type of a node in the document tree:

```
Before: { type: "paragraph", content: [{ text: "My Title" }] }
  ↓ User applies "Heading 1"
After:  { type: "heading", attrs: { level: 1 }, content: [{ text: "My Title" }] }
```

##### Formatting as Operations

Every formatting change is an **operation** in the collaboration sense:

```tsx
type FormatOperation = {
  type: 'addMark' | 'removeMark' | 'setBlockType';
  from: number;      // start position in document
  to: number;        // end position
  mark?: Mark;       // for inline marks
  blockType?: string; // for block type changes
  attrs?: Record<string, any>;
};
```

This operation can be:
*   Applied locally and rendered immediately.
*   Sent to the collaboration server for transformation and broadcast.
*   Stored in the undo/redo history.

---

#### 5.2.7 Embedded Content (Images, Tables, Embeds)

##### Images

Images are "atom" nodes — they are leaf nodes in the document tree that represent a single object, not a container of text.

```ts
type ImageNode = {
  type: 'image';
  attrs: {
    src: string;
    alt: string;
    width: number;
    height: number;
    alignment: 'left' | 'center' | 'right' | 'inline';
  };
};
```

**Image upload flow:**
1.  User pastes/drops an image.
2.  Generate a temporary local blob URL → insert an `ImageNode` with `src = blob:...`.
3.  Upload the image to the media service in the background.
4.  On upload success, replace `src` with the permanent CDN URL.
5.  If upload fails, show an error overlay on the image with a retry button.

##### Tables

Tables are complex nested structures:

```ts
type TableNode = {
  type: 'table';
  content: TableRowNode[];
};

type TableRowNode = {
  type: 'tableRow';
  content: TableCellNode[];
};

type TableCellNode = {
  type: 'tableCell';
  attrs: {
    colspan: number;
    rowspan: number;
    backgroundColor?: string;
  };
  content: BlockNode[];  // Each cell contains paragraphs
};
```

Tables are particularly challenging for collaboration because operations can affect multiple cells (merge cells, add row, resize columns) and must be correctly transformed.

---

### 5.3 Reusability Strategy

*   **`EditorCore`**: Headless editor engine (document model, operations, schema) — reusable across different UI surfaces (full editor, comment editor, inline text fields).
*   **`FormatButton`**: Generic toolbar button component that reads the current selection's state and dispatches format operations.
*   **`BlockRenderer`**: Polymorphic renderer that delegates to type-specific components based on the block's `type` property.
*   **`CollaborationProvider`**: React context that provides the WebSocket connection and OT/CRDT engine to any component that needs it.
*   **`PresenceCursor`**: Renders a colored cursor with a name label; reused for each remote collaborator.
*   Design tokens for editor chrome (toolbar colors, spacing, font stacks).

---

### 5.4 Module Organization

```
src/
 ├── editor/
 │    ├── model/
 │    │    ├── documentModel.ts    (tree structure, node types)
 │    │    ├── schema.ts           (allowed node/mark types and relationships)
 │    │    ├── selection.ts        (model-based selection)
 │    │    └── position.ts         (position mapping, resolve)
 │    ├── operations/
 │    │    ├── insertText.ts
 │    │    ├── deleteText.ts
 │    │    ├── splitBlock.ts
 │    │    ├── joinBlocks.ts
 │    │    ├── setBlockType.ts
 │    │    ├── toggleMark.ts
 │    │    ├── insertSlice.ts      (paste/drop)
 │    │    └── tableOps.ts         (add row, merge cell, etc.)
 │    ├── view/
 │    │    ├── editorView.ts       (DOM rendering, event binding)
 │    │    ├── blockRenderer.ts    (paragraph, heading, list, etc.)
 │    │    ├── markRenderer.ts     (bold, italic, link, etc.)
 │    │    ├── selectionView.ts    (render browser selection from model)
 │    │    └── decorations.ts      (comment highlights, find/replace highlights)
 │    ├── input/
 │    │    ├── inputHandler.ts     (beforeinput, keydown)
 │    │    ├── imeHandler.ts       (composition events)
 │    │    ├── clipboardHandler.ts (paste, copy, cut)
 │    │    └── dropHandler.ts      (drag and drop)
 │    └── history/
 │         ├── undoManager.ts
 │         └── historyEntry.ts
 ├── collaboration/
 │    ├── otClient.ts              (OT client state machine)
 │    ├── otTransform.ts           (transform functions)
 │    ├── wsTransport.ts           (WebSocket connection)
 │    ├── presenceManager.ts       (cursor/selection sharing)
 │    └── offlineBuffer.ts         (queue ops when offline)
 ├── components/
 │    ├── DocumentPage.tsx
 │    ├── EditorSurface.tsx
 │    ├── Toolbar.tsx
 │    ├── FormatButton.tsx
 │    ├── CommentSidebar.tsx
 │    ├── VersionHistoryPanel.tsx
 │    ├── CollaboratorAvatars.tsx
 │    ├── CursorOverlay.tsx
 │    └── ImageUploadOverlay.tsx
 ├── api/
 │    ├── documentApi.ts
 │    ├── commentApi.ts
 │    └── mediaApi.ts
 ├── shared/
 │    ├── keyboard.ts
 │    ├── clipboard.ts
 │    ├── debounce.ts
 │    └── analytics.ts
 └── types/
      └── types.ts
```

---

## 6. High Level Data Flow Explanation

### 6.1 Initial Load Flow

```
1. User navigates to /doc/:docId
     ↓
2. Server renders document shell (SSR) — toolbar, sidebar layout, first viewport of content
     ↓
3. Client hydrates → DocumentPage mounts
     ↓
4. Fetch full document: GET /api/docs/:docId
     ↓
5. Initialize document model from response JSON
     ↓
6. Render document model to DOM (editor surface)
     ↓
7. Establish WebSocket connection for collaboration: wss://collab.example.com/:docId
     ↓
8. Receive any pending operations from other users since the snapshot was taken
     ↓
9. Apply pending operations → document is now in sync
     ↓
10. Editor is ready — user can type
```

---

### 6.2 User Interaction Flow

**Typing a character:**

```
User types "a"
     ↓
1. beforeinput event fires (inputType: "insertText", data: "a")
     ↓
2. event.preventDefault() — prevent browser DOM mutation
     ↓
3. Create operation: insert("a", position=42)
     ↓
4. Apply operation to document model → model updated
     ↓
5. Update DOM (insert "a" into the correct text node) → user sees "a" instantly
     ↓
6. Update model selection (move cursor right by 1)
     ↓
7. Record in undo history
     ↓
8. Send operation to collaboration server via WebSocket
     ↓
9. Server transforms + broadcasts to other clients
     ↓
10. Other clients apply the transformed operation → they see "a" appear
```

**Applying formatting:**

```
User selects "world" → presses Ctrl+B
     ↓
1. Create operation: addMark("bold", from=6, to=11)
     ↓
2. Apply to model → text run splits:
   ["Hello ", "world" (bold), "!"]
     ↓
3. Update DOM → the "world" span gets font-weight: bold
     ↓
4. Toolbar "Bold" button updates to show active/pressed state
     ↓
5. Send operation to server → broadcast to collaborators
```

---

### 6.3 Error and Retry Flow

*   **WebSocket disconnects**: Show "Reconnecting..." banner. Buffer local operations in an offline queue. On reconnect, resync with server (fetch missed operations, rebase local operations on top).
*   **Operation rejected by server**: If the server rejects an operation (schema violation, permission issue), rollback the local model to the last confirmed state. Show error toast.
*   **Document save fails**: Show "Unable to save — retrying..." banner. Auto-retry with exponential backoff. Keep buffer of unsaved operations.
*   **Image upload fails**: Show error overlay on the image with retry button. Don't block editing — user can continue typing.
*   **Conflict during reconnect**: If the user edited extensively offline and reconnects, the OT/CRDT engine rebases all local operations against the server's current state. If this produces unexpected content, the user can use undo or version history to recover.

---

## 7. Data Modelling (Frontend Perspective)

### 7.1 Core Data Entities

*   **Document** — the top-level entity: metadata + content tree.
*   **Node** — a single element in the document tree (paragraph, heading, text, image, table, etc.).
*   **Mark** — an inline formatting annotation on a text node (bold, italic, link, etc.).
*   **Operation** — a single atomic change to the document (insert, delete, format).
*   **Comment** — a comment anchored to a text range, with replies.
*   **Collaborator** — a remote user with cursor position, selection, and presence.

---

### 7.2 Data Shape

```ts
// Document metadata
type DocumentMeta = {
  id: string;
  title: string;
  ownerId: string;
  createdAt: string;
  updatedAt: string;
  permission: 'viewer' | 'commenter' | 'editor';
  collaborators: CollaboratorInfo[];
};

// Document content (the tree model)
type DocumentContent = {
  type: 'doc';
  content: BlockNode[];
};

// Block node types
type BlockNode =
  | ParagraphNode
  | HeadingNode
  | BulletListNode
  | OrderedListNode
  | ListItemNode
  | CodeBlockNode
  | BlockquoteNode
  | TableNode
  | ImageNode
  | HorizontalRuleNode;

type ParagraphNode = {
  type: 'paragraph';
  content: InlineNode[];
};

type HeadingNode = {
  type: 'heading';
  attrs: { level: 1 | 2 | 3 | 4 | 5 | 6 };
  content: InlineNode[];
};

// Inline node types
type InlineNode = TextNode | HardBreakNode;

type TextNode = {
  type: 'text';
  text: string;
  marks: Mark[];
};

// Marks (inline formatting)
type Mark =
  | { type: 'bold' }
  | { type: 'italic' }
  | { type: 'underline' }
  | { type: 'strikethrough' }
  | { type: 'code' }
  | { type: 'link'; attrs: { href: string; title?: string } }
  | { type: 'textColor'; attrs: { color: string } }
  | { type: 'highlight'; attrs: { color: string } }
  | { type: 'comment'; attrs: { commentId: string } };

// Operations (for OT)
type Operation =
  | { type: 'retain'; count: number }
  | { type: 'insert'; text: string; marks?: Mark[] }
  | { type: 'delete'; count: number }
  | { type: 'insertNode'; node: BlockNode }
  | { type: 'deleteNode'; count: number }
  | { type: 'addMark'; from: number; to: number; mark: Mark }
  | { type: 'removeMark'; from: number; to: number; mark: Mark }
  | { type: 'setBlockType'; pos: number; type: string; attrs?: Record<string, any> };

// Collaborator presence
type CollaboratorPresence = {
  userId: string;
  name: string;
  color: string;         // unique color per user
  cursor: number | null; // model position
  selection: { anchor: number; head: number } | null;
  lastActive: number;    // timestamp
};

// Comment
type Comment = {
  id: string;
  author: { id: string; name: string; avatar: string };
  text: string;
  anchor: { from: number; to: number }; // position in document model
  createdAt: string;
  resolved: boolean;
  replies: CommentReply[];
};

type CommentReply = {
  id: string;
  author: { id: string; name: string; avatar: string };
  text: string;
  createdAt: string;
};
```

---

### 7.3 Entity Relationships

*   **One-to-Many**: One `Document` → many `BlockNode` items (content tree).
*   **One-to-Many**: One `TextNode` → many `Mark` items (inline formatting).
*   **One-to-Many**: One `Document` → many `Comment` items.
*   **One-to-Many**: One `Comment` → many `CommentReply` items.
*   **Many-to-Many**: `Comment.anchor` references a range of document positions — as the document changes, these anchors must be remapped.
*   **Nested**: `Table` → `TableRow` → `TableCell` → `Block[]` (cells contain sub-documents).

**Normalized vs Denormalized:**
The document content is a **denormalized tree** (each node contains its children inline). This is necessary because the tree structure IS the content — you need the full tree to render the document. There's no benefit to normalizing it.

Collaborator presence is stored in a **flat Map** keyed by userId — it's ephemeral and frequently updated.

---

### 7.4 UI Specific Data Models

```ts
// Toolbar state (derived from current selection)
type ToolbarState = {
  isBold: boolean;
  isItalic: boolean;
  isUnderline: boolean;
  isStrikethrough: boolean;
  isCode: boolean;
  currentBlockType: string;     // "paragraph", "heading", etc.
  headingLevel: number | null;
  textAlign: string;
  fontFamily: string;
  fontSize: number;
  textColor: string;
  highlightColor: string;
  isLink: boolean;
  linkHref: string | null;
  canUndo: boolean;
  canRedo: boolean;
  listType: 'bullet' | 'ordered' | 'checklist' | null;
};

// Derived from the document model + current selection, recomputed on every selection change.

// Editor viewport state (for large document optimization)
type ViewportState = {
  scrollTop: number;
  visibleBlockRange: { start: number; end: number }; // which blocks are in the viewport
};
```

---

## 8. State Management Strategy

### 8.1 State Classification

| State Type | Examples | Storage |
|---|---|---|
| **Document State** | Content tree (all nodes, marks) | Editor core (in-memory); synced via OT/CRDT |
| **Collaboration State** | Connected users, their cursors/selections | In-memory Map; updated via WebSocket |
| **Editor UI State** | Active tool, toolbar state, open dialog | Derived from selection + local `useState` |
| **Comment State** | Comment threads linked to document ranges | Server-fetched; React Query cache |
| **Component Local State** | Dropdown open, search input value | `useState` / `useReducer` |
| **Derived State** | Toolbar active states, word count, outline | Computed from document model + selection |

---

### 8.2 State Ownership

*   **EditorCore** (custom class, not React) owns the document model, selection, and undo history. It is the single source of truth for document content. React components **read** from it but never directly mutate it.
*   **CollaborationProvider** (React context) owns the WebSocket connection, OT client, and presence state. It listens for remote operations and feeds them into the EditorCore.
*   **Toolbar** derives its state from the EditorCore's current selection. When the user clicks a format button, Toolbar dispatches a formatting operation to the EditorCore.
*   **CommentSidebar** manages comment data via React Query (server-fetched). Comment anchors are stored as document positions and remapped when the document changes.

**Data flow:**
```
Input event → EditorCore.applyOperation() → Model updated
     ↓
EditorCore emits "change" event
     ↓
React components re-render via subscription (useEditorState hook)
     ↓
CollaborationProvider sends operation to server
     ↓
Server transforms + broadcasts
     ↓
CollaborationProvider receives remote op → EditorCore.applyRemoteOperation()
     ↓
Model updated → React re-renders → DOM updated
```

---

### 8.3 Persistence Strategy

| Data | Persistence | Reason |
|------|------------|--------|
| Document content | Server-synced via OT/CRDT + periodic snapshots | Authoritative server state; collaboration requires it |
| Pending operations (offline) | In-memory buffer; IndexedDB for crash recovery | Preserve user work if tab closes before sync |
| User cursor/selection | WebSocket (ephemeral) | Only relevant while connected |
| Comments | Server via REST API | Persistent, shared across sessions |
| Document metadata | Server via REST API | Title, permissions, last modified |
| Editor preferences (font zoom, spell check) | localStorage | Per-user, per-device |
| Version history | Server | Complete change history |

---

## 9. Real Time Collaboration Deep Dive

This is the **most technically distinctive** aspect of Google Docs. The problem: how do N users edit the same document simultaneously without losing anyone's work, without locking any region, and with sub-second propagation of every edit?

---

### 9.1 The Fundamental Problem of Concurrent Editing

```
Initial document: "ABCD"

User A (in New York) inserts "X" at position 1:  → "AXBCD"
User B (in London)  deletes char at position 3:  → "ABD"

Both happen at the same time (neither has seen the other's edit yet).

If we apply them naively:
  Start:   "ABCD"
  Apply A: "AXBCD"  (insert X at 1)
  Apply B: delete at position 3 → deletes "C" → "AXBD"  ✓

BUT if the order is reversed:
  Start:   "ABCD"
  Apply B: "ABD"    (delete at 3 removes D)
  Apply A: insert X at 1 → "AXBD"  ✓

Here it works. But consider:
  User A inserts at position 2
  User B inserts at position 2

  Both applied → one character is placed BEFORE the other. But which one?
  Without a deterministic rule, different clients might produce different results.
  → DIVERGENCE. The documents are no longer the same.
```

**The fundamental requirement: all clients must converge to the same document state, regardless of the order operations are received.**

Two algorithms solve this: **Operational Transformation (OT)** and **CRDTs (Conflict-free Replicated Data Types)**.

---

### 9.2 Operational Transformation (OT)

OT is the algorithm used by the original Google Docs (and Google Wave before it). The core idea: **when you receive a remote operation, transform it against any local operations that the remote user hasn't seen yet, so that the transformed operation produces the correct result in your local context.**

---

#### 9.2.1 What is an Operation

An operation is a sequence of **components** that walk through the document from start to end:

```ts
type OTOperation = OTComponent[];

type OTComponent =
  | { type: 'retain'; count: number }    // skip N characters (no change)
  | { type: 'insert'; text: string }     // insert text at current position
  | { type: 'delete'; count: number };   // delete N characters at current position
```

**Every operation must exactly span the entire document length.**

Example: Document is `"HELLO"` (length 5). User inserts `"X"` at position 2:

```
Operation: [
  { type: 'retain', count: 2 },   // skip "HE"
  { type: 'insert', text: 'X' },  // insert "X" → now "HEXLLO"
  { type: 'retain', count: 3 },   // skip "LLO"
]
// Input length: 5 (2 + 3 retained)
// Output length: 6 (2 retained + 1 inserted + 3 retained)
```

Example: Delete character at position 3 from `"HELLO"`:

```
Operation: [
  { type: 'retain', count: 3 },   // skip "HEL"
  { type: 'delete', count: 1 },   // delete "L"
  { type: 'retain', count: 1 },   // skip "O"
]
// Input length: 5 (3 + 1 + 1)
// Output length: 4 (3 retained + 1 retained, 1 deleted)
```

---

#### 9.2.2 The Transform Function

The transform function is the **heart of OT**. Given two concurrent operations A and B (both based on the same document state), transform produces A' and B' such that:

```
apply(apply(doc, A), B') === apply(apply(doc, B), A')
```

This is the **convergence property** — no matter what order the operations are applied, the result is the same.

```
        doc
       /   \
      A     B
     /       \
  docA      docB
     \       /
      B'   A'
       \   /
       docAB  ===  docBA
```

##### Transform(Insert, Insert)

```
Document: "AB"
User A: insert "X" at position 1 → [retain(1), insert("X"), retain(1)]
User B: insert "Y" at position 1 → [retain(1), insert("Y"), retain(1)]

Both are concurrent (based on "AB").

Transform result:
  A' = [retain(1), insert("X"), retain(2)]  // retain 2 because Y was inserted
  B' = [retain(2), insert("Y"), retain(1)]  // retain 2 because X was inserted

Apply A then B':
  "AB" → "AXAB" ... wait, that's wrong.

Let's trace more carefully:
  "AB" → apply A → "AXB" (insert X at 1)
  "AXB" → apply B' → B' needs to account for X being inserted before Y's position
  B' = [retain(2), insert("Y"), retain(1)]
  "AXB" → retain 2 ("AX") → insert "Y" → retain 1 ("B") → "AXYB"

Apply B then A':
  "AB" → apply B → "AYB" (insert Y at 1)
  "AYB" → apply A' → A' = [retain(1), insert("X"), retain(2)]
  "AYB" → retain 1 ("A") → insert "X" → retain 2 ("YB") → "AXYB"

Both paths → "AXYB" ✅ CONVERGED!
```

**Tie-breaking rule: when two inserts happen at the same position, the server decides who goes first (usually by user ID or timestamp). This ensures deterministic ordering.**

##### Transform(Insert, Delete)

```
Document: "ABCD"
User A: insert "X" at position 1 → [retain(1), insert("X"), retain(3)]
User B: delete at position 2       → [retain(2), delete(1), retain(1)]

Transform:
  A' against B: B deleted at position 2. A inserts at position 1 (before the delete).
    A's position is unaffected. But the document is now shorter (3 chars).
    A' = [retain(1), insert("X"), retain(2)]  // retain 2 instead of 3 (one char deleted)

  B' against A: A inserted at position 1. B deletes at position 2, but since A's insert
    shifted everything after position 1 to the right by 1, B's delete position becomes 3.
    B' = [retain(3), delete(1), retain(1)]

Apply A then B':
  "ABCD" → A → "AXBCD" → B' → retain 3 ("AXB"), delete 1 ("C"), retain 1 ("D") → "AXBD"

Apply B then A':
  "ABCD" → B → "ABD" → A' → retain 1 ("A"), insert "X", retain 2 ("BD") → "AXBD"

Both → "AXBD" ✅ CONVERGED!
```

##### Transform Implementation

```tsx
function transform(
  opA: OTComponent[],
  opB: OTComponent[],
  priority: 'left' | 'right' = 'left' // tie-breaker for same-position inserts
): [OTComponent[], OTComponent[]] {
  const aPrime: OTComponent[] = [];
  const bPrime: OTComponent[] = [];

  let indexA = 0;
  let indexB = 0;
  let compA = opA[indexA];
  let compB = opB[indexB];

  while (indexA < opA.length || indexB < opB.length) {
    // Case 1: A is inserting (inserts don't consume input, so B must account for them)
    if (compA && compA.type === 'insert') {
      aPrime.push(compA);
      bPrime.push({ type: 'retain', count: compA.text.length });
      compA = opA[++indexA];
      continue;
    }

    // Case 2: B is inserting
    if (compB && compB.type === 'insert') {
      bPrime.push(compB);
      aPrime.push({ type: 'retain', count: compB.text.length });
      compB = opB[++indexB];
      continue;
    }

    // Case 3: Both retain
    if (compA?.type === 'retain' && compB?.type === 'retain') {
      const len = Math.min(compA.count, compB.count);
      aPrime.push({ type: 'retain', count: len });
      bPrime.push({ type: 'retain', count: len });
      // Consume min length from both
      compA = consumeComponent(compA, len, opA, indexA);
      compB = consumeComponent(compB, len, opB, indexB);
      if (compA?.count === 0) compA = opA[++indexA];
      if (compB?.count === 0) compB = opB[++indexB];
      continue;
    }

    // Case 4: A retains, B deletes
    if (compA?.type === 'retain' && compB?.type === 'delete') {
      const len = Math.min(compA.count, compB.count);
      bPrime.push({ type: 'delete', count: len });
      // A's retain is consumed by B's delete — nothing added to aPrime for this range
      compA = { ...compA, count: compA.count - len };
      compB = { ...compB, count: compB.count - len };
      if (compA.count === 0) compA = opA[++indexA];
      if (compB.count === 0) compB = opB[++indexB];
      continue;
    }

    // Case 5: A deletes, B retains (symmetric to case 4)
    if (compA?.type === 'delete' && compB?.type === 'retain') {
      const len = Math.min(compA.count, compB.count);
      aPrime.push({ type: 'delete', count: len });
      compA = { ...compA, count: compA.count - len };
      compB = { ...compB, count: compB.count - len };
      if (compA.count === 0) compA = opA[++indexA];
      if (compB.count === 0) compB = opB[++indexB];
      continue;
    }

    // Case 6: Both delete at the same position — they cancel each other
    if (compA?.type === 'delete' && compB?.type === 'delete') {
      const len = Math.min(compA.count, compB.count);
      compA = { ...compA, count: compA.count - len };
      compB = { ...compB, count: compB.count - len };
      if (compA.count === 0) compA = opA[++indexA];
      if (compB.count === 0) compB = opB[++indexB];
      continue;
    }

    break; // shouldn't reach here if operations are valid
  }

  return [aPrime, bPrime];
}
```

---

#### 9.2.3 OT with a Central Server

Google Docs uses a **central server** model for OT. The server is the single source of truth and assigns a **linear order** to all operations (a revision number):

```
┌────────────┐                ┌────────────────┐                ┌────────────┐
│  Client A   │                │  Server         │                │  Client B   │
│  (rev 5)    │                │  (rev 5)        │                │  (rev 5)    │
└─────┬──────┘                └───────┬────────┘                └─────┬──────┘
      │                                │                                │
      │  op_A (based on rev 5)         │                                │
      │─────────────────────────────►│                                │
      │                                │  op_B (based on rev 5)         │
      │                                │◄──────────────────────────────│
      │                                │                                │
      │                    Server receives op_A first:                  │
      │                    1. Apply op_A → rev 6                        │
      │                    2. Broadcast op_A to Client B                │
      │                                │                                │
      │                    Server receives op_B (based on rev 5):       │
      │                    3. Transform op_B against op_A               │
      │                       → op_B' (now based on rev 6)             │
      │                    4. Apply op_B' → rev 7                       │
      │                    5. Broadcast op_B' to Client A               │
      │                                │                                │
      │  ack(rev 6) ← confirm op_A    │                                │
      │◄─────────────────────────────│                                │
      │                                │  op_A (broadcast to B)         │
      │                                │──────────────────────────────►│
      │  op_B' (transformed)           │                                │
      │◄─────────────────────────────│                                │
      │                                │  ack(rev 7) ← confirm op_B'   │
      │                                │──────────────────────────────►│
```

**Key properties of server-based OT:**
*   The server always has the **authoritative** document state.
*   Each operation is assigned a monotonically increasing **revision number**.
*   Operations from clients carry the revision they're based on. If it's not the latest, the server transforms them.
*   Clients receive either **acknowledgments** (for their own ops) or **remote operations** (from others).

---

#### 9.2.4 Complete OT Client Implementation

The OT client is a **state machine** with three states:

```
┌──────────────────────┐
│   SYNCHRONIZED       │  Client is in sync with server.
│   (no pending ops)   │  Local revision === server revision.
└──────────┬───────────┘
           │ User edits locally
           ▼
┌──────────────────────┐
│   AWAITING_CONFIRM   │  Client has sent an operation to server.
│   (op in flight)     │  Waiting for acknowledgment.
└──────────┬───────────┘
           │ User edits again before ack arrives
           ▼
┌──────────────────────┐
│   AWAITING_WITH_     │  Client has sent one op AND has a
│   BUFFER             │  second op buffered locally.
│   (op in flight +    │  When ack arrives, buffer is sent.
│    buffered op)      │
└──────────────────────┘
```

```tsx
type ClientState =
  | { type: 'synchronized' }
  | { type: 'awaitingConfirm'; outstanding: OTOperation }
  | { type: 'awaitingWithBuffer'; outstanding: OTOperation; buffer: OTOperation };

class OTClient {
  private state: ClientState = { type: 'synchronized' };
  private revision: number;
  private doc: string; // simplified — real system uses document model

  constructor(
    private transport: WebSocketTransport,
    initialDoc: string,
    initialRevision: number
  ) {
    this.doc = initialDoc;
    this.revision = initialRevision;

    // Listen for server messages
    transport.on('ack', () => this.handleAck());
    transport.on('operation', (op: OTOperation) => this.handleRemoteOp(op));
  }

  // User made a local edit — apply it and send/buffer
  applyLocal(operation: OTOperation) {
    // Apply to local document immediately (instant feedback)
    this.doc = applyOperation(this.doc, operation);

    switch (this.state.type) {
      case 'synchronized':
        // No pending ops — send directly
        this.transport.send({ op: operation, revision: this.revision });
        this.state = { type: 'awaitingConfirm', outstanding: operation };
        break;

      case 'awaitingConfirm':
        // Already have one op in flight — buffer this one
        this.state = {
          type: 'awaitingWithBuffer',
          outstanding: this.state.outstanding,
          buffer: operation,
        };
        break;

      case 'awaitingWithBuffer':
        // Already buffering — compose the new op with the buffer
        this.state = {
          ...this.state,
          buffer: compose(this.state.buffer, operation),
        };
        break;
    }
  }

  // Server acknowledged our operation
  private handleAck() {
    this.revision++;

    switch (this.state.type) {
      case 'awaitingConfirm':
        // Our op is confirmed — back to synchronized
        this.state = { type: 'synchronized' };
        break;

      case 'awaitingWithBuffer':
        // Our outstanding op is confirmed — send the buffer
        this.transport.send({ op: this.state.buffer, revision: this.revision });
        this.state = {
          type: 'awaitingConfirm',
          outstanding: this.state.buffer,
        };
        break;

      default:
        throw new Error('Received ack in unexpected state');
    }
  }

  // Received a remote operation from another user
  private handleRemoteOp(serverOp: OTOperation) {
    this.revision++;

    switch (this.state.type) {
      case 'synchronized':
        // No local pending ops — apply server op directly
        this.doc = applyOperation(this.doc, serverOp);
        this.emitChange();
        break;

      case 'awaitingConfirm': {
        // Transform server op against our outstanding op
        const [outstandingPrime, serverOpPrime] = transform(
          this.state.outstanding,
          serverOp
        );
        this.state = { type: 'awaitingConfirm', outstanding: outstandingPrime };
        this.doc = applyOperation(this.doc, serverOpPrime);
        this.emitChange();
        break;
      }

      case 'awaitingWithBuffer': {
        // Transform against outstanding, then against buffer
        const [outstandingPrime, serverOp1] = transform(
          this.state.outstanding,
          serverOp
        );
        const [bufferPrime, serverOp2] = transform(this.state.buffer, serverOp1);

        this.state = {
          type: 'awaitingWithBuffer',
          outstanding: outstandingPrime,
          buffer: bufferPrime,
        };
        this.doc = applyOperation(this.doc, serverOp2);
        this.emitChange();
        break;
      }
    }
  }
}
```

##### Why Three States?

```
Q: Why not just send every operation immediately?

A: If we send op1 and then immediately send op2 (before op1 is acknowledged),
   the server might process op2 before op1 (network reordering), or transform
   op2 against a different base than expected. The three-state design ensures:

   1. Only ONE operation is in flight at a time.
   2. Additional edits are buffered and composed.
   3. After ack, the buffer is sent as the next operation.

   This guarantees correct ordering and transformation.
```

##### State Machine Visualization

```
                    ┌──────────────┐
               ┌───►│ SYNCHRONIZED │◄──────┐
               │    └──────┬───────┘       │
               │           │ local edit    │ ack (no buffer)
               │           ▼               │
               │    ┌──────────────────┐   │
               │    │ AWAITING_CONFIRM │───┘
               │    └──────┬───────────┘
               │           │ local edit
               │           ▼
               │    ┌──────────────────────┐
               │    │ AWAITING_WITH_BUFFER │◄─┐
               │    └──────┬───────────────┘  │
               │           │ ack              │ local edit
               │           │ (send buffer)    │ (compose into buffer)
               │           ▼                  │
               │    ┌──────────────────┐      │
               └────│ AWAITING_CONFIRM │──────┘
                    └──────────────────┘
```

---

#### 9.2.5 Server Side OT Orchestration

The server maintains the **canonical operation log** — an ordered list of all operations, each with a revision number:

```tsx
// Simplified server-side OT handler
class OTServer {
  private operations: OTOperation[] = [];  // canonical operation log
  private document: string;                // current document state

  handleClientOperation(clientOp: OTOperation, clientRevision: number, clientId: string) {
    // Client's operation is based on revision `clientRevision`.
    // Server's current revision is `this.operations.length`.
    // Transform the client's op against all ops it has missed.

    let transformedOp = clientOp;

    for (let i = clientRevision; i < this.operations.length; i++) {
      const [, serverOpTransformed] = transform(this.operations[i], transformedOp);
      transformedOp = serverOpTransformed;
      // Note: we need the second output (transformedOp adjusted for the server op)
    }

    // Apply the fully transformed operation
    this.document = applyOperation(this.document, transformedOp);
    this.operations.push(transformedOp);

    // Send ack to the author
    this.send(clientId, { type: 'ack' });

    // Broadcast to all other clients
    this.broadcast(clientId, { type: 'operation', op: transformedOp });
  }
}
```

---

### 9.3 CRDTs (Conflict Free Replicated Data Types)

CRDTs are an alternative to OT that is gaining popularity for real-time collaboration. Unlike OT (which transforms operations), CRDTs design the **data structure itself** so that concurrent edits automatically converge without transformation.

---

#### 9.3.1 How CRDTs Differ from OT

| Aspect | OT | CRDT |
|--------|---|----|
| **Core idea** | Transform operations against each other to account for concurrency | Design data types that mathematically guarantee convergence |
| **Server requirement** | Central server is needed to order operations | No central server needed (peer-to-peer works) |
| **Operation complexity** | Operations are simple (insert, delete). Transform function is complex. | Operations are simple. Data structure carries unique IDs and metadata. |
| **Metadata overhead** | Minimal — operations are lightweight | Each character may carry a unique ID (increased memory) |
| **Deletion** | Physical deletion (character is removed) | Tombstoning (deleted characters are marked, not removed) |
| **Used by** | Google Docs, Google Wave | Yjs, Automerge, Apple Notes, Figma (for some features) |
| **Undo complexity** | Complex (must invert and transform against concurrent ops) | Complex (must track causal history) |
| **Offline support** | Limited (requires server for transformation) | Excellent (edits can be merged peer-to-peer on reconnect) |

---

#### 9.3.2 Sequence CRDTs for Text (Yjs Example)

For text editing, we need a **sequence CRDT** — a data structure where elements (characters) can be inserted and deleted at any position, and concurrent operations always produce the same result on all replicas.

The key insight: instead of using **index positions** (which shift when characters are inserted/deleted by others), each character gets a **globally unique ID** that never changes.

##### How Yjs Represents Text

```
Traditional text: "HELLO"
  index:           0 1 2 3 4
  Problem: if User B inserts at index 2, all indices after 2 change.

CRDT text (Yjs): "HELLO"
  Each character has a unique, stable identifier:
    'H' → { id: (clientA, clock=0) }
    'E' → { id: (clientA, clock=1) }
    'L' → { id: (clientA, clock=2) }
    'L' → { id: (clientA, clock=3) }
    'O' → { id: (clientA, clock=4) }

  Insert "X" between "E" and "L":
    New item: { id: (clientB, clock=0), content: 'X', origin: (clientA, clock=1) }
    Result: H E X L L O
    No index shifting — X is positioned by its relationship to "E" (its origin).
```

##### Concurrent Insert Resolution

```
Document: "AB"
User A inserts "X" between A and B:
  { id: (A, 0), content: 'X', originLeft: idOfA, originRight: idOfB }

User B inserts "Y" between A and B:
  { id: (B, 0), content: 'Y', originLeft: idOfA, originRight: idOfB }

Both X and Y are between A and B. Which goes first?

CRDT resolution rule: compare client IDs.
  If clientA < clientB → X comes before Y: "AXYB"
  This is deterministic — all clients produce the same result.

No transformation needed. Just insert both items and let the CRDT's ordering rules sort them.
```

---

#### 9.3.3 CRDT Integration with Editor

Using Yjs with a ProseMirror-based editor:

```tsx
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';
import { ySyncPlugin, yCursorPlugin, yUndoPlugin } from 'y-prosemirror';

function setupCollaborativeEditor(docId: string) {
  // Create a Yjs document
  const ydoc = new Y.Doc();

  // The document content is a Yjs XML Fragment (maps to ProseMirror's document model)
  const yxml = ydoc.getXmlFragment('prosemirror');

  // WebSocket provider for real-time sync
  const provider = new WebsocketProvider(
    'wss://collab.example.com',
    docId,
    ydoc
  );

  // Set user awareness (cursor, selection, name, color)
  provider.awareness.setLocalStateField('user', {
    name: currentUser.name,
    color: currentUser.color,
    cursor: null,
  });

  // Create ProseMirror editor with Yjs plugins
  const editor = new EditorView(document.getElementById('editor'), {
    state: EditorState.create({
      schema: editorSchema,
      plugins: [
        // Syncs ProseMirror's document model with the Yjs CRDT
        ySyncPlugin(yxml),
        // Shows remote cursors and selections
        yCursorPlugin(provider.awareness),
        // Wires undo/redo to Yjs's undo manager (not ProseMirror's default)
        yUndoPlugin(),
        // ... other plugins (keymap, input rules, etc.)
      ],
    }),
  });

  return { editor, ydoc, provider };
}
```

**How the sync plugin works:**
1.  User types in ProseMirror → ProseMirror creates a transaction.
2.  `ySyncPlugin` intercepts the transaction and converts it to Yjs operations on `yxml`.
3.  Yjs propagates the changes to the `WebsocketProvider`.
4.  Provider sends the changes to the server → server relays to other clients.
5.  Other clients' Yjs instances receive changes → `ySyncPlugin` converts them to ProseMirror transactions → editor updates.

---

### 9.4 OT vs CRDT Comparison

| Criterion | OT (Google Docs approach) | CRDT (Yjs / Automerge approach) |
|-----------|--------------------------|--------------------------------|
| **Convergence guarantee** | Requires correct transform function (hard to prove, easy to have bugs) | Mathematically guaranteed by the data type design |
| **Central server** | Required (assigns operation order) | Optional (works peer-to-peer) |
| **Offline support** | Weak — must sync through server | Strong — merge locally, sync when online |
| **Memory overhead** | Low — operations are lightweight | Higher — each character carries metadata (unique ID, tombstones) |
| **Large documents** | Efficient (operations on a flat string) | Can be slower (tree traversal for large CRDT structures) |
| **Undo/redo** | Complex but well-understood | Very complex (must track causal dependencies) |
| **Maturity** | 20+ years (Google Wave in 2009) | 10+ years (academic); Yjs mature since ~2019 |
| **Implementation effort** | High (correct transform functions are subtle) | Moderate (use a library like Yjs) |
| **Best for** | Centralized apps (Google Docs, Notion) | Decentralized/local-first apps (Excalidraw, linear, some Figma features) |

**For Google Docs specifically, OT is the right choice:**
*   Google already has robust server infrastructure.
*   OT has lower memory overhead (important for 100-page documents).
*   Server-authoritative model enables features like version history, permission enforcement, and server-side spell check.

**For a startup building a collaborative editor, CRDT (via Yjs) is often the practical choice:**
*   Much easier to implement correctly (just use the library).
*   Better offline support out of the box.
*   No need for a complex OT server.

---

### 9.5 Cursor and Selection Presence

Each collaborator's cursor position and selection must be visible to all other users in real time.

##### How Cursor Positions Are Shared

```
User A types → cursor at model position 42
     ↓
Client A sends presence update via WebSocket:
  { userId: "A", cursor: 42, selection: null, name: "Alice", color: "#e06666" }
     ↓
Server relays to all other clients
     ↓
Client B receives → renders Alice's cursor at position 42 with a colored flag
```

##### How Cursor Positions Survive Edits

When a remote operation changes the document, all cursor positions might shift. We must **map** cursor positions through the operation:

```
User B's cursor is at position 10.
User A inserts 5 characters at position 3.

User B's cursor should now be at position 15 (shifted right by 5).

Mapping function:
  newPosition = mapPositionThroughOperation(oldPosition, operation)
```

```tsx
function mapPosition(pos: number, operation: OTComponent[]): number {
  let currentPos = 0;
  let newPos = pos;

  for (const comp of operation) {
    if (currentPos >= pos) break;

    switch (comp.type) {
      case 'retain':
        currentPos += comp.count;
        break;

      case 'insert':
        // Insert happened before our position — shift right
        if (currentPos <= pos) {
          newPos += comp.text.length;
        }
        break;

      case 'delete':
        // Delete happened before our position — shift left
        const deleteEnd = currentPos + comp.count;
        if (deleteEnd <= pos) {
          newPos -= comp.count;
        } else if (currentPos <= pos && deleteEnd > pos) {
          // Our cursor was inside the deleted range — move to start of deletion
          newPos = currentPos;
        }
        currentPos += comp.count;
        break;
    }
  }

  return Math.max(0, newPos);
}
```

##### Rendering Remote Cursors

```tsx
function CursorOverlay({ collaborators }: { collaborators: CollaboratorPresence[] }) {
  return (
    <>
      {collaborators.map((collab) => {
        if (!collab.cursor) return null;

        // Convert model position to screen coordinates
        const coords = positionToScreenCoords(collab.cursor);
        if (!coords) return null;

        return (
          <div key={collab.userId} className="remote-cursor" style={{
            position: 'absolute',
            left: coords.left,
            top: coords.top,
            pointerEvents: 'none',
          }}>
            {/* Blinking cursor line */}
            <div style={{
              width: 2,
              height: coords.lineHeight,
              backgroundColor: collab.color,
            }} />

            {/* Name label */}
            <div style={{
              backgroundColor: collab.color,
              color: 'white',
              fontSize: 11,
              padding: '1px 4px',
              borderRadius: 3,
              whiteSpace: 'nowrap',
              transform: 'translateY(-100%)',
            }}>
              {collab.name}
            </div>
          </div>
        );
      })}
    </>
  );
}
```

##### Remote Selection Highlighting

When a collaborator selects a range of text, the selection is rendered as a **semi-transparent colored highlight** behind the text:

```tsx
// Convert a model range (anchor, head) to a set of DOM rectangles
function getSelectionRects(from: number, to: number): DOMRect[] {
  // Map model positions to DOM text node + offset
  const startDOM = positionToDOM(from);
  const endDOM = positionToDOM(to);

  // Use the Range API to get visual rectangles
  const range = document.createRange();
  range.setStart(startDOM.node, startDOM.offset);
  range.setEnd(endDOM.node, endDOM.offset);

  return Array.from(range.getClientRects());
}
```

---

### 9.6 WebSocket Transport Layer

```tsx
class CollaborationTransport {
  private ws: WebSocket;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;
  private offlineBuffer: OTOperation[] = [];

  connect(docId: string) {
    this.ws = new WebSocket(`wss://collab.example.com/doc/${docId}`);

    this.ws.onopen = () => {
      this.reconnectAttempts = 0;
      // Flush offline buffer
      this.flushOfflineBuffer();
    };

    this.ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      switch (message.type) {
        case 'ack':
          this.emit('ack');
          break;
        case 'operation':
          this.emit('operation', message.op);
          break;
        case 'presence':
          this.emit('presence', message.data);
          break;
        case 'error':
          this.handleServerError(message);
          break;
      }
    };

    this.ws.onclose = (event) => {
      if (!event.wasClean) {
        this.scheduleReconnect();
      }
    };
  }

  send(operation: OTOperation, revision: number) {
    if (this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify({
        type: 'operation',
        op: operation,
        revision,
      }));
    } else {
      // Offline — buffer the operation
      this.offlineBuffer.push(operation);
    }
  }

  private scheduleReconnect() {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      this.emit('disconnected');
      return;
    }

    const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
    this.reconnectAttempts++;

    setTimeout(() => {
      this.connect(this.docId);
    }, delay);
  }

  private flushOfflineBuffer() {
    // On reconnect, we need to resync with the server first,
    // then rebase our buffered operations on the latest server state.
    // This is handled by the OT client's state machine.
    for (const op of this.offlineBuffer) {
      this.emit('localBuffered', op);
    }
    this.offlineBuffer = [];
  }
}
```

---

### 9.7 Offline Editing and Reconnection

When the network drops, the user should continue editing seamlessly:

```
Network drops
     ↓
1. OT client switches to "offline" mode
2. User continues typing — all operations are applied locally and buffered
3. UI shows "Offline — changes will be saved when you reconnect"
     ↓
Network returns
     ↓
4. WebSocket reconnects
5. Client sends: "I'm at revision 42. What have I missed?"
6. Server responds with operations 43, 44, 45, ... (all ops since rev 42)
7. Client transforms its buffered operations against the missed server ops
8. Client sends its rebased operations to the server
9. Server acknowledges → client is back in sync
10. UI shows "All changes saved"
```

##### Offline Operation Buffer with IndexedDB

For crash safety, buffered operations are periodically written to IndexedDB:

```tsx
async function persistOfflineBuffer(buffer: OTOperation[]) {
  const db = await openDB('doc-offline', 1, {
    upgrade(db) {
      db.createObjectStore('operations', { keyPath: 'id', autoIncrement: true });
    }
  });

  const tx = db.transaction('operations', 'readwrite');
  for (const op of buffer) {
    tx.store.put({ op, timestamp: Date.now() });
  }
  await tx.done;
}
```

If the user closes the tab and reopens later, the buffered operations are loaded from IndexedDB and resynced with the server.

---

## 10. Undo Redo in Collaborative Context

Undo/redo in a collaborative editor is **much harder** than in a single-user editor. The key question: **what does "undo" mean when other users have edited the document since your last action?**

##### The Problem

```
User A types "Hello" at the beginning.
User B types "World" at the end.
User A presses Ctrl+Z (undo).

What should happen?
  Option 1: Undo A's "Hello" → "World" remains. ✅ This is correct.
  Option 2: Undo the LAST operation (B's "World") → wrong! That's B's work.
```

**Undo should only undo YOUR OWN operations, even if other users have since edited the document.**

##### Implementation

```tsx
class CollaborativeUndoManager {
  private undoStack: OTOperation[] = [];  // inverse of user's own operations
  private redoStack: OTOperation[] = [];

  // When the local user performs an edit
  record(operation: OTOperation) {
    this.undoStack.push(invertOperation(operation));
    this.redoStack = []; // clear redo on new edit
  }

  // When a remote operation arrives, transform undo/redo stacks
  transformStacks(remoteOp: OTOperation) {
    // Transform every entry in the undo stack against the remote op
    this.undoStack = this.undoStack.map((undoOp) => {
      const [transformedUndo] = transform(undoOp, remoteOp);
      return transformedUndo;
    });

    this.redoStack = this.redoStack.map((redoOp) => {
      const [transformedRedo] = transform(redoOp, remoteOp);
      return transformedRedo;
    });
  }

  undo(): OTOperation | null {
    const undoOp = this.undoStack.pop();
    if (!undoOp) return null;

    // Push the inverse (redo version) onto the redo stack
    this.redoStack.push(invertOperation(undoOp));

    // Return the operation to be applied and sent to the server
    return undoOp;
  }

  redo(): OTOperation | null {
    const redoOp = this.redoStack.pop();
    if (!redoOp) return null;

    this.undoStack.push(invertOperation(redoOp));
    return redoOp;
  }
}
```

**Why transform the undo stack?**
When User B inserts 5 characters at position 3, all positions in User A's undo stack that are >= 3 must shift right by 5. Without transforming, undoing would delete the wrong characters.

---

## 11. High Level API Design (Frontend POV)

### 11.1 Required APIs

| API | Method | Description |
|-----|--------|-------------|
| `/api/docs/:id` | **GET** | Fetch document content + metadata |
| `/api/docs/:id` | **PUT** | Save document snapshot (periodic, not per-keystroke) |
| `/api/docs` | **POST** | Create new document |
| `/api/docs/:id` | **DELETE** | Delete document |
| `wss://collab.example.com/doc/:id` | **WebSocket** | Real-time operation sync and presence |
| `/api/docs/:id/comments` | **GET** | Fetch all comments for a document |
| `/api/docs/:id/comments` | **POST** | Add a new comment |
| `/api/docs/:id/comments/:cid` | **PUT** | Edit or resolve a comment |
| `/api/docs/:id/comments/:cid/replies` | **POST** | Add a reply to a comment thread |
| `/api/docs/:id/versions` | **GET** | Fetch version history list |
| `/api/docs/:id/versions/:vid` | **GET** | Fetch a specific version snapshot |
| `/api/docs/:id/media` | **POST** | Upload an image or file |
| `/api/docs/:id/export` | **POST** | Export to PDF, DOCX, HTML |

---

### 11.2 Request and Response Structure

**GET /api/docs/:id**

```json
// Response
{
  "id": "doc_abc123",
  "title": "Q4 Project Plan",
  "content": {
    "type": "doc",
    "content": [
      {
        "type": "heading",
        "attrs": { "level": 1 },
        "content": [{ "type": "text", "text": "Q4 Project Plan" }]
      },
      {
        "type": "paragraph",
        "content": [
          { "type": "text", "text": "This document outlines..." }
        ]
      }
    ]
  },
  "revision": 42,
  "collaborators": [
    { "userId": "u_1", "name": "Zeeshan Ali", "avatar": "...", "color": "#e06666" }
  ],
  "permission": "editor",
  "updatedAt": "2026-03-17T08:00:00Z"
}
```

**WebSocket Messages**

```json
// Client → Server: Send operation
{
  "type": "operation",
  "op": [
    { "type": "retain", "count": 42 },
    { "type": "insert", "text": "Hello" },
    { "type": "retain", "count": 100 }
  ],
  "revision": 42
}

// Server → Client: Acknowledge
{
  "type": "ack"
}

// Server → Client: Remote operation
{
  "type": "operation",
  "op": [
    { "type": "retain", "count": 10 },
    { "type": "delete", "count": 5 },
    { "type": "retain", "count": 137 }
  ],
  "userId": "u_2",
  "revision": 43
}

// Client → Server: Presence update
{
  "type": "presence",
  "cursor": 42,
  "selection": { "anchor": 42, "head": 50 }
}

// Server → Client: Remote presence
{
  "type": "presence",
  "userId": "u_2",
  "name": "Ali",
  "color": "#6aa84f",
  "cursor": 78,
  "selection": null
}
```

---

### 11.3 Error Handling and Status Codes

| Status | Scenario | Frontend Handling |
|--------|----------|-------------------|
| `200` | Success | Render document |
| `304` | Not Modified | Use cached version |
| `400` | Malformed operation | Log error; this indicates a client bug |
| `401` | Unauthorized | Redirect to sign-in |
| `403` | No permission (viewer trying to edit) | Show "View only" banner; disable editing |
| `404` | Document not found | Show "Document not found" error page |
| `409` | Revision conflict (rare in OT) | Refetch document and resync |
| `413` | Document/media too large | Show "File too large" error |
| `429` | Rate limit | Throttle operations; show "Saving paused" |
| `500` | Server error | Show "Unable to save — retrying" banner |
| WS `1006` | Abnormal WebSocket close | Auto-reconnect with exponential backoff |

---

## 12. Caching Strategy

### 12.1 What to Cache

*   **Document content**: The source of truth is the collaboration server. Local state is the "cache" — always in sync via OT/CRDT.
*   **Document metadata** (title, permissions): React Query cache, short TTL (1 min).
*   **Comments**: React Query cache keyed by document ID. Refetch on focus.
*   **User profiles / avatars**: Browser HTTP cache + CDN (long TTL).
*   **Fonts**: Service Worker cache (critical for consistent text rendering).
*   **Application bundle**: Service Worker cache for offline access.

### 12.2 Where to Cache

| Data | Cache Location | TTL |
|------|----------------|-----|
| Document content (active) | In-memory (editor model) | Ephemeral — synced in real time |
| Offline operation buffer | IndexedDB | Until synced |
| Comments | React Query in-memory | 1 min; refetch on focus |
| User avatars | Browser HTTP cache + CDN | 24h |
| Fonts | Service Worker cache | Indefinite (immutable filenames) |
| App shell | Service Worker | Until new version |
| Editor preferences | localStorage | Indefinite |

### 12.3 Cache Invalidation

*   **Document content**: Not cached traditionally — the OT/CRDT system keeps it in sync. On reconnect, the client fetches missed operations.
*   **Comments**: Invalidated when a new comment is added (optimistic update + background refetch).
*   **User avatars**: Immutable with content-hashed URLs.
*   **App bundle**: Service Worker checks for updates; stale-while-revalidate.

---

## 13. CDN and Asset Optimization

*   **App delivery**: Static HTML + JS + CSS served from CDN. The editor is a heavy bundle (~500KB gzipped), so code splitting is critical.
*   **Font delivery**: Custom editor fonts (serif, sans-serif, monospace options) served from CDN with `font-display: swap`. Preloaded in `<head>` since they're critical for text rendering.
*   **Image delivery**: Uploaded images are stored in a blob service and served via CDN with signed URLs. Responsive sizes via `srcset`.
*   **Cache headers**:
    *   JS/CSS: `Cache-Control: public, max-age=31536000, immutable` (content-hashed).
    *   HTML: `Cache-Control: no-cache`.
    *   Fonts: `Cache-Control: public, max-age=31536000, immutable`.
    *   API responses: `Cache-Control: private, no-cache`.
*   **Compression**: Brotli for all text-based assets. Document content over WebSocket is also compressed (permessage-deflate).

---

## 14. Rendering Strategy

*   **SSR** for the document page shell (toolbar, sidebar, page layout). Provides fast FCP and allows the page to be interactive sooner — the user sees the editor chrome while the document loads.
*   **CSR** for the document content. The editor model is initialized client-side from the fetched JSON. There's no benefit to server-rendering the document content because:
    1.  The content changes constantly (collaboration invalidates any SSR'd content instantly).
    2.  The custom rendering engine owns the DOM — React/SSR would conflict.
*   **Code splitting**:
    *   Core editor + toolbar: main chunk.
    *   Collaboration module: lazy-loaded on WebSocket connect.
    *   Comments sidebar: lazy-loaded when user opens comments.
    *   Export module: lazy-loaded when user opens File → Export.
    *   Version history: lazy-loaded when user opens the panel.
*   **Virtualization for large documents**: For 100+ page documents, only render the blocks visible in the viewport (+ a buffer above and below). Off-screen blocks are replaced with placeholder divs of the correct height. This keeps DOM size manageable.

---

## 15. Cross Cutting Non Functional Concerns

### 15.1 Security

*   **Access control**: Permission level (viewer/commenter/editor) is enforced on both client and server. The editor UI disables editing for viewers. The server rejects operations from non-editors.
*   **XSS prevention**: User-generated content is rendered from the typed document model (not arbitrary HTML). Pasted HTML is sanitized and converted to the model's schema — any script tags, event handlers, or dangerous elements are stripped.
*   **CSRF**: All API calls include a CSRF token. WebSocket connection is authenticated via the same session.
*   **Token storage**: Auth tokens in HTTP-only cookies. No storage of sensitive data in localStorage.
*   **Content isolation**: Documents from different users/orgs are isolated at the API level. The frontend has no access to data outside the current document.
*   **Clipboard sanitization**: When pasting from external sources, parse HTML through a whitelist of allowed tags and attributes. Disallow `<script>`, `<iframe>`, `<form>`, `on*` attributes.

---

### 15.2 Accessibility

*   **Semantic DOM structure**:
    *   Headings rendered as `<h1>` through `<h6>`.
    *   Lists as `<ul>`, `<ol>`, `<li>`.
    *   Links as `<a>`.
    *   Tables as `<table>`, `<tr>`, `<td>`.
    *   Document content inside `<main role="document">`.
*   **Keyboard navigation**:
    *   Standard keyboard shortcuts: Ctrl+B (bold), Ctrl+I (italic), Ctrl+Z (undo), etc.
    *   Tab/Shift+Tab in lists for indent/outdent.
    *   F7 to toggle caret browsing.
    *   Toolbar navigation via Tab + arrow keys (roving tabindex).
*   **Screen reader support**:
    *   Live announcements for collaborator actions: "Alice started editing" (via `aria-live="polite"`).
    *   Format state announced when cursor moves: "Bold, Heading 2."
    *   Comment markers indicated to screen readers.
*   **Focus management**: When dialogs open (Insert Image, Share), focus is trapped. On close, focus returns to the editor.
*   **Color contrast**: Toolbar icons, comment text, and status messages meet WCAG AA. Collaborator colors are chosen from an accessible palette.

---

### 15.3 Performance Optimization

##### Typing Latency

The most critical performance metric for an editor is **input latency** — the time from keypress to the character appearing on screen. Target: < 16ms (one frame).

```
Keypress → beforeinput event (~0ms)
     ↓
Create operation (~0.1ms)
     ↓
Apply to model (~0.1ms for a single character insert)
     ↓
Update DOM (~0.5ms — modify one text node)
     ↓
Update selection (~0.1ms)
     ↓
Browser paints (~1-2ms)
     ↓
Total: ~3ms — WELL within 16ms budget ✅
```

The collaboration layer (send to server, transform, receive) does NOT block the input pipeline. The operation is sent asynchronously after the local update completes.

##### Large Document Optimization

| Technique | How it helps |
|-----------|-------------|
| **Block-level viewport virtualization** | Only render blocks visible on screen + buffer. Replace off-screen blocks with height-matching placeholders. |
| **Deferred formatting updates** | When applying a format change to 100 paragraphs (select all → bold), batch the DOM updates using `requestAnimationFrame`. |
| **Incremental rendering** | After a paste of 10,000 characters, render the first viewport immediately, then render remaining paragraphs in subsequent frames. |
| **Debounced toolbar updates** | Toolbar state (bold active, heading level) is derived from selection. Debounce derivation to 50ms — fast enough to feel instant, but avoids recomputing on every arrow key press in rapid navigation. |
| **WeakMap caches for computed data** | Cache computed block heights, parsed link previews, and heading outline for the sidebar. Invalidate per-block when that block changes. |
| **Object pooling for DOM nodes** | When virtualizing, recycle DOM elements instead of creating new ones (reduces GC pressure). |

##### Code Splitting Impact

| Chunk | Size (Gzipped) | Load trigger |
|-------|----------------|-------------|
| Core editor + toolbar | ~250KB | Initial page load |
| Collaboration module | ~80KB | On WebSocket connect |
| Export (PDF/DOCX) | ~120KB | File → Export click |
| Comments sidebar | ~40KB | Open comments panel |
| Version history | ~50KB | Open version history |
| Image upload + resize | ~30KB | Insert → Image click |

---

### 15.4 Observability and Reliability

*   **Error Boundaries**: Wrap the editor surface, toolbar, and comment sidebar in separate error boundaries. If comments crash, editing continues.
*   **Logging**:
    *   Track keypress-to-screen latency for P95 monitoring.
    *   Track OT/CRDT operation round-trip time.
    *   Log WebSocket disconnects, reconnects, and their durations.
    *   Track "operation rejected" events (indicates client bugs).
*   **Performance monitoring**:
    *   FCP, TTI, and LCP for document page.
    *   Editor first-interactive time (time from page load to first keystroke accepted).
    *   Frame rate during continuous typing (should be 60fps).
*   **Feature flags**: Gate new editor features:
    *   New block types (toggleable sections, callout boxes).
    *   Alternative collaboration engines (OT vs CRDT A/B test).
*   **Graceful degradation**:
    *   If WebSocket fails → switch to periodic HTTP polling (long-poll) for operations.
    *   If collaboration fails entirely → single-user mode with auto-save via REST.
    *   If custom rendering fails → fall back to a simplified contentEditable mode.

---

## 16. Edge Cases and Tradeoffs

| Edge Case | Handling |
|-----------|---------|
| **Two users type at the exact same position** | OT transform function deterministically orders inserts by user ID. Both characters appear, but in a consistent order across all clients. |
| **User pastes 50,000 characters** | Parse and insert as a single operation. Render in batches (first viewport immediately, rest deferred). Large pastes may cause a brief freeze — show a progress indicator for pastes > 5,000 chars. |
| **User edits offline for 2 hours** | All operations buffered in memory + IndexedDB. On reconnect, rebase buffered ops against server's current state. If the document has diverged significantly, warn the user and let them review the merge. |
| **100+ page document scrolling** | Virtualize editor blocks — only render visible blocks. Maintain placeholder divs for height stability. Render off-screen blocks lazily on scroll. |
| **Collaborator's cursor is in a deleted paragraph** | When User A deletes a paragraph that User B's cursor is in, map B's cursor to the start of the nearest surviving block. |
| **Comment anchor on deleted text** | Comments anchored to deleted text become "orphaned." Show them in the sidebar with a note: "The text this comment refers to was deleted." Allow the comment to be resolved or re-anchored. |
| **Browser crash during editing** | The last synced revision is on the server. Unsaved local operations (up to debounce interval) may be lost. IndexedDB buffer mitigates this for offline edits. |
| **IME input during collaboration** | While composing (e.g., Japanese IME), do not send intermediate states to the server. Only send the final committed text. Remote operations received during composition are queued and applied after composition ends. |
| **Table with 1000s of cells** | Virtualize table rendering (only cells in the viewport). Prohibit operations that would create tables larger than a limit (e.g., 500 cells). |
| **Very large number of collaborators (50+)** | Throttle presence broadcasts (every 100ms instead of every mouse move). Group cursors that are close together. Cap cursor rendering at 20 visible cursors. |
| **Concurrent undo from different users** | Each user's undo stack is independent. User A's undo only reverses A's operations. Remote operations transform the undo stack so it remains valid. |

### Key Tradeoffs

| Decision | Tradeoff |
|----------|----------|
| **OT over CRDT** | Lower memory overhead, server-authoritative (good for permissions/versioning). But requires a central server and complex transform functions. CRDTs would be better for offline-first scenarios. |
| **Custom document model over raw HTML** | Clean data, collaboration-friendly, validatable. But requires building a full rendering pipeline and input handler from scratch. |
| **ContentEditable for input over Canvas** | Gets native IME, spell check, autocomplete for free. But must deal with browser inconsistencies in contentEditable behavior. Canvas-based (Google's 2021 approach) would give full control but requires reimplementing all text rendering. |
| **SSR for page shell, CSR for content** | Fast perceived load (toolbar visible early). But document content isn't available for SEO (acceptable — docs are gated behind auth). |
| **Block-level virtualization** | Handles 100+ page documents with stable performance. But adds complexity: scroll position management, height estimation for unmeasured blocks, layout shifts. |
| **WebSocket for collaboration** | Low-latency bidirectional communication. But requires persistent connections (server resource cost) and reconnection handling. SSE would be simpler but only server→client. |
| **Per-user undo** | Correct behavior in collaborative context (undo YOUR changes, not others'). But complex to implement — undo stack must be transformed against every remote operation. |
| **Debounced auto-save (server snapshots)** | Reduces server write load. But periodic snapshots mean version history is coarser than operation-level history. Actual operation log is the true history; snapshots are for fast loading. |

---

## 17. Summary and Future Improvements

### Key Architectural Decisions

1.  **Custom document model** — a typed tree of nodes with inline marks, independent of the DOM. Enables clean serialization, validation, collaboration, and cross-platform rendering.
2.  **Operational Transformation (OT)** for real-time collaboration — a central server assigns operation order; clients transform concurrent operations. Three-state client machine (synchronized, awaiting confirm, awaiting with buffer) ensures correct ordering.
3.  **ContentEditable for input capture + custom DOM rendering** — leverages browser's native input handling (IME, spell check) while maintaining full control over the document model and rendering.
4.  **Block-level virtualization** for large documents — only visible blocks are rendered to DOM; off-screen blocks use height-matched placeholders.
5.  **Per-user undo with stack transformation** — each user's undo stack is independent and transformed against remote operations.
6.  **Auto-save via operation stream** — every operation is synced in real time via WebSocket; periodic snapshots provide fast document loading.

### Possible Future Enhancements

*   **Canvas-based rendering** (Google Docs 2021 approach): Render the entire document on `<canvas>` for pixel-perfect consistency across browsers, custom pagination, and better performance for very large documents.
*   **CRDT migration**: Migrate from OT to a CRDT-based system (like Yjs) for better offline support and peer-to-peer collaboration without a central server.
*   **AI-powered features**: Real-time grammar suggestions, auto-complete sentences, summarize document, generate content from prompts.
*   **Branching and merging**: Git-like document branches where users can make changes in a "branch" and merge them back into the main document with conflict resolution.
*   **Real-time voice/video**: Embedded video chat within the document editing session for tighter collaboration.
*   **Plugin / extension system**: Allow third-party developers to add custom block types, toolbar actions, and integrations (e.g., embed Figma frames, Jira issues).
*   **Cross-document linking**: Reference and embed content from other documents with live updates.
*   **Advanced table operations**: Spreadsheet-like formulas within tables, charts generated from table data.
*   **Web Worker for OT computation**: Offload operation transformation and document model computation to a Web Worker, keeping the main thread free for rendering.

---

### Endpoint Summary

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/docs/:id` | GET | Fetch document content and metadata |
| `/api/docs/:id` | PUT | Save document snapshot |
| `/api/docs` | POST | Create new document |
| `/api/docs/:id` | DELETE | Delete document |
| `wss://collab.example.com/doc/:id` | WebSocket | Real-time sync and presence |
| `/api/docs/:id/comments` | GET / POST | Fetch or add comments |
| `/api/docs/:id/comments/:cid` | PUT | Edit or resolve comment |
| `/api/docs/:id/comments/:cid/replies` | POST | Add reply to comment thread |
| `/api/docs/:id/versions` | GET | List version history |
| `/api/docs/:id/versions/:vid` | GET | Fetch specific version |
| `/api/docs/:id/media` | POST | Upload media |
| `/api/docs/:id/export` | POST | Export to PDF/DOCX/HTML |

---

### Complete Data Flow Summary

| Direction | Mechanism | Trigger | Target | Action |
|-----------|-----------|---------|--------|--------|
| Initial Load | REST (SSR + CSR) | Page open | `GET /api/docs/:id` → Model → Render | Load and display document |
| User Types | Local (synchronous) | Keystroke | BeforeInput → Model → DOM | Instant local update |
| Sync to Server | WebSocket (async) | After local apply | OT Client → WebSocket → Server | Send operation for transformation |
| Receive Remote Edit | WebSocket (push) | Server broadcast | WebSocket → OT Transform → Model → DOM | Apply remote changes |
| Cursor Presence | WebSocket (throttled) | Cursor move | Presence → WebSocket → All Clients | Show remote cursors |
| Format Text | Local + Sync | Toolbar click or shortcut | Model operation → DOM → WebSocket | Apply and broadcast formatting |
| Add Comment | REST + RT | User comments | `POST /api/docs/:id/comments` + WS broadcast | Comment appears for all users |
| Auto Save | Implicit (OT stream) | Every operation | OT ops → Server → persist | Every edit is saved server-side |
| Version Restore | REST | User selects version | `GET /api/docs/:id/versions/:vid` → Model → Render | Reload document at version |
| Offline Edit | Local buffer | Network drops | Model → IndexedDB buffer → Sync on reconnect | Preserve and sync later |

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design
