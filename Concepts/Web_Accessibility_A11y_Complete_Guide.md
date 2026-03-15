
# ♿ Web Accessibility (A11y) — A Complete Guide to Building Inclusive Web Applications

> *"The power of the Web is in its universality. Access by everyone regardless of disability is an essential aspect."* — **Tim Berners-Lee**

Accessibility (often abbreviated as **a11y** — because there are 11 letters between the "a" and "y") is the practice of making your web applications usable by **as many people as possible**. This includes people with visual, auditory, motor, cognitive, and neurological disabilities — but also benefits users on slow networks, mobile devices, aging hardware, or those with temporary impairments (like a broken arm).

If you've ever built a beautiful UI that a screen reader can't navigate, or a form that can't be submitted with the keyboard — you know the gap between looking good and being truly usable. 😅

This guide covers **why accessibility matters**, **how to audit and check it**, **how to fix common issues**, and the **best practices** every frontend engineer should follow.

---

<a id="top"></a>

## Table of Contents

- [Why Accessibility Matters](#why-accessibility-matters)
- [Understanding WCAG Standards](#understanding-wcag-standards)
- [The Four Principles POUR](#the-four-principles-pour)
- [Common Accessibility Issues in Web Apps](#common-accessibility-issues-in-web-apps)
- [Semantic HTML The Foundation of A11y](#semantic-html-the-foundation-of-a11y)
- [ARIA Roles States and Properties](#aria-roles-states-and-properties)
- [Keyboard Accessibility](#keyboard-accessibility)
- [Focus Management](#focus-management)
- [Color Contrast and Visual Design](#color-contrast-and-visual-design)
- [Accessible Forms](#accessible-forms)
- [Accessible Images and Media](#accessible-images-and-media)
- [Accessible Navigation and Routing (SPAs)](#accessible-navigation-and-routing-spas)
- [Accessible Modals Dialogs and Popups](#accessible-modals-dialogs-and-popups)
- [Responsive and Zoom Friendly Design](#responsive-and-zoom-friendly-design)
- [How to Check and Audit Accessibility](#how-to-check-and-audit-accessibility)
- [How to Fix Accessibility Issues Step by Step](#how-to-fix-accessibility-issues-step-by-step)
- [Accessibility in React Angular and Vue](#accessibility-in-react-angular-and-vue)
- [Accessibility Testing Checklist](#accessibility-testing-checklist)
- [Key Interview Takeaways](#key-interview-takeaways)
- [Further Reading and Resources](#further-reading-and-resources)

[⬆ Back to Top](#top)

---

## Why Accessibility Matters

### Real-World Impact

Over **1 billion people worldwide** live with some form of disability. That's roughly **15% of the global population**. Beyond permanent disabilities, consider:

- A user with a **broken wrist** who can only use a keyboard
- A person in **bright sunlight** who needs high contrast
- A developer using the app **without a mouse** (power user)
- An elderly user with **declining vision**

### Business & Legal Case

| Reason                     | Details                                                                                         |
| -------------------------- | ----------------------------------------------------------------------------------------------- |
| **Legal Compliance**       | ADA (US), EAA (EU), Section 508, AODA (Canada) — lawsuits are increasing year over year         |
| **Larger Audience**        | ~15% of users are excluded if your app is inaccessible                                          |
| **SEO Benefits**           | Semantic HTML, alt text, proper headings all boost search rankings                              |
| **Better UX for Everyone** | Accessibility improvements (keyboard nav, focus states, clear labels) help ALL users             |
| **Brand Reputation**       | Inclusive design signals maturity and user respect                                               |

> 💡 In 2025, over **4,000+ accessibility-related lawsuits** were filed in the US alone. Companies like Domino's, Nike, and Beyoncé's website have all faced legal action.

[⬆ Back to Top](#top)

---

## Understanding WCAG Standards

**WCAG (Web Content Accessibility Guidelines)** is the international standard maintained by the **W3C (World Wide Web Consortium)**.

### WCAG Versions

| Version        | Year | Key Changes                                             |
| -------------- | ---- | ------------------------------------------------------- |
| WCAG 2.0       | 2008 | Established the 4 core principles (POUR)                |
| WCAG 2.1       | 2018 | Added mobile, cognitive, and low-vision criteria        |
| WCAG 2.2       | 2023 | Focus appearance, dragging alternatives, target size    |
| WCAG 3.0       | TBD  | Major overhaul — new scoring model (Bronze/Silver/Gold) |

### Conformance Levels

| Level  | Description                                        | Example                                         |
| ------ | -------------------------------------------------- | ------------------------------------------------ |
| **A**  | Minimum — removes biggest barriers                 | All images have alt text                         |
| **AA** | Mid-range — standard for most legal requirements   | Color contrast ratio ≥ 4.5:1                     |
| **AAA**| Highest — gold standard (not always feasible)      | Color contrast ratio ≥ 7:1, sign language videos |

> 🎯 **Most organizations aim for WCAG 2.1 Level AA** — this is the most commonly required level by law.

[⬆ Back to Top](#top)

---

## The Four Principles POUR

WCAG is built on **four foundational principles** known as **POUR**:

### 1. **Perceivable** — Users must be able to perceive the content

- Provide **text alternatives** for non-text content (images, icons, charts)
- Provide **captions and transcripts** for audio/video
- Ensure sufficient **color contrast**
- Content should be **adaptable** — presentable in different ways without losing meaning

### 2. **Operable** — Users must be able to operate the interface

- All functionality must be available via **keyboard**
- Provide users enough **time** to read and use content
- Don't design content that causes **seizures** (no rapid flashing)
- Provide ways to **navigate, find content, and determine location**

### 3. **Understandable** — Content and UI must be understandable

- Text should be **readable and predictable**
- Pages should **operate in predictable ways**
- Help users **avoid and correct mistakes** (form validation, error messages)

### 4. **Robust** — Content must be robust enough for assistive technologies

- Use **valid, semantic HTML**
- Ensure compatibility with **current and future** assistive technologies
- Use **ARIA** when native semantics are insufficient

[⬆ Back to Top](#top)

---

## Common Accessibility Issues in Web Apps

Here are the **most frequently found** accessibility violations (based on WebAIM's annual Million Report):

| Rank | Issue                             | Prevalence | Impact                                       |
| ---- | --------------------------------- | ---------- | -------------------------------------------- |
| 1    | Low contrast text                 | ~81%       | Unreadable text for low-vision users         |
| 2    | Missing alt text on images        | ~54%       | Screen readers announce "image" with no info |
| 3    | Missing form input labels         | ~48%       | Users don't know what to type or select      |
| 4    | Empty links / buttons             | ~44%       | "Link" announced with no context             |
| 5    | Missing document language         | ~28%       | Screen readers mispronounce content          |
| 6    | Empty table headers               | ~24%       | Data tables become impossible to navigate    |

> ⚠️ These 6 issues alone account for **96%+ of all detected errors** across the top 1 million home pages.

[⬆ Back to Top](#top)

---

## Semantic HTML The Foundation of A11y

The single most impactful thing you can do for accessibility is write **proper, semantic HTML**. Before reaching for ARIA, use the right HTML elements.

### ❌ Non-Semantic (Div Soup)

```html
<div class="header">
  <div class="nav">
    <div class="nav-item" onclick="navigate('/home')">Home</div>
    <div class="nav-item" onclick="navigate('/about')">About</div>
  </div>
</div>
<div class="main-content">
  <div class="title">Welcome to Our Site</div>
  <div class="text">This is a paragraph of text.</div>
</div>
<div class="footer">© 2026</div>
```

**Problems**: No landmarks, no keyboard support, no structure for screen readers.

### ✅ Semantic HTML

```html
<header>
  <nav aria-label="Main navigation">
    <ul>
      <li><a href="/home">Home</a></li>
      <li><a href="/about">About</a></li>
    </ul>
  </nav>
</header>
<main>
  <h1>Welcome to Our Site</h1>
  <p>This is a paragraph of text.</p>
</main>
<footer>© 2026</footer>
```

**Benefits**: Landmarks (`<header>`, `<nav>`, `<main>`, `<footer>`), heading hierarchy, keyboard-navigable links.

### Semantic Elements Cheatsheet

| Instead of...            | Use...                         | Why                                              |
| ------------------------ | ------------------------------ | ------------------------------------------------ |
| `<div onclick="...">`    | `<button>`                     | Keyboard support, role, focus built-in           |
| `<div class="header">`   | `<header>`                     | Landmark role for screen readers                 |
| `<div class="nav">`      | `<nav>`                        | Navigation landmark                              |
| `<span class="link">`    | `<a href="...">`               | Focusable, keyboard operable, announced as link  |
| `<div class="list">`     | `<ul>` / `<ol>` + `<li>`      | Screen readers announce list and item count      |
| `<div class="table">`    | `<table>` + `<th>` + `<td>`   | Navigable, relationships announced               |
| `<b>` / `<i>`            | `<strong>` / `<em>`            | Conveys semantic emphasis, not just visual        |

> 💡 **Rule of Thumb**: If a native HTML element can do the job, prefer it over a `<div>` + ARIA. The first rule of ARIA is: **Don't use ARIA if you can use native HTML.**

[⬆ Back to Top](#top)

---

## ARIA Roles States and Properties

**ARIA (Accessible Rich Internet Applications)** extends HTML semantics when native elements aren't sufficient — particularly for custom widgets, SPAs, and dynamic content.

### When to Use ARIA

| Scenario                               | Use ARIA?  |
| -------------------------------------- | ---------- |
| Standard button                        | ❌ Use `<button>` |
| Custom dropdown built with `<div>`s    | ✅ Yes      |
| Tab interface with no native equivalent| ✅ Yes      |
| Image that is decorative               | ❌ Use `alt=""` |
| Live updating notification area        | ✅ Yes — `aria-live` |

### Key ARIA Attributes

```html
<!-- Roles: What is this element? -->
<div role="alert">Session expired!</div>
<div role="tablist">
  <div role="tab" aria-selected="true">Tab 1</div>
  <div role="tab" aria-selected="false">Tab 2</div>
</div>

<!-- States: What is the current state? -->
<button aria-expanded="false" aria-controls="menu1">Menu</button>
<div id="menu1" hidden>...</div>

<!-- Properties: Relationships and descriptions -->
<input aria-labelledby="label1" aria-describedby="hint1" />
<span id="label1">Email Address</span>
<span id="hint1">We'll never share your email.</span>

<!-- Live regions: Dynamic content updates -->
<div aria-live="polite">3 new messages</div>   <!-- Announced when idle -->
<div aria-live="assertive">Error: Payment failed</div>  <!-- Announced immediately -->
```

### Commonly Used ARIA Attributes

| Attribute              | Purpose                                             | Example                                          |
| ---------------------- | --------------------------------------------------- | ------------------------------------------------ |
| `role`                 | Defines what the element is                         | `role="dialog"`, `role="alert"`                  |
| `aria-label`           | Provides an accessible name                         | `aria-label="Close sidebar"`                     |
| `aria-labelledby`      | Points to another element for its name              | `aria-labelledby="section-heading"`              |
| `aria-describedby`     | Points to descriptive text                          | `aria-describedby="password-hint"`               |
| `aria-hidden`          | Hides element from assistive tech                   | `aria-hidden="true"` on decorative icons         |
| `aria-expanded`        | Indicates expanded/collapsed state                  | Accordion, dropdown                              |
| `aria-live`            | Announces dynamic content changes                   | Toast notifications, chat messages               |
| `aria-required`        | Marks a form field as required                      | `<input aria-required="true">`                   |
| `aria-invalid`         | Marks a field with validation error                 | `<input aria-invalid="true">`                    |
| `aria-current`         | Indicates current item in a set                     | `aria-current="page"` in navigation              |

> ⚠️ **Warning**: Incorrect ARIA is worse than no ARIA at all. A `<div role="button">` without keyboard handling misleads assistive tech users into thinking it's interactive when it's not.

[⬆ Back to Top](#top)

---

## Keyboard Accessibility

**All interactive elements must be operable via keyboard alone.** Many users — including power users, screen reader users, and motor-impaired users — navigate exclusively with the keyboard.

### Essential Keyboard Interactions

| Key              | Expected Behavior                                           |
| ---------------- | ----------------------------------------------------------- |
| `Tab`            | Move focus to next interactive element                      |
| `Shift + Tab`    | Move focus to previous interactive element                  |
| `Enter`          | Activate buttons, links, submit forms                       |
| `Space`          | Activate buttons, toggle checkboxes                         |
| `Escape`         | Close modal, dropdown, popover                              |
| `Arrow Keys`     | Navigate within widgets (tabs, menus, radio groups, sliders)|

### Making Custom Elements Keyboard Accessible

```html
<!-- ❌ Not keyboard accessible -->
<div class="button" onclick="submitForm()">Submit</div>

<!-- ✅ Keyboard accessible with native button -->
<button onclick="submitForm()">Submit</button>

<!-- ✅ If you MUST use a div (rare), add these: -->
<div
  role="button"
  tabindex="0"
  onclick="submitForm()"
  onkeydown="if(event.key === 'Enter' || event.key === ' ') submitForm()"
>
  Submit
</div>
```

### Tab Order Best Practices

```html
<!-- tabindex values explained -->
<button tabindex="0">Normal tab order (follows DOM)</button>
<div tabindex="0">Added to tab order</div>
<div tabindex="-1">Programmatically focusable, but NOT in tab order</div>

<!-- ❌ NEVER do this — disrupts natural tab order -->
<button tabindex="5">Don't use positive tabindex!</button>
```

| tabindex Value | Behavior                                               | Use Case                             |
| -------------- | ------------------------------------------------------ | ------------------------------------ |
| Not set        | Only focusable if natively interactive                 | Default for `<button>`, `<a>`, etc.  |
| `0`            | Follows natural DOM order                              | Custom interactive widgets           |
| `-1`           | Focusable via JS, not via Tab key                      | Focus management (modals, errors)    |
| `1+`           | ❌ Forces order — unpredictable and fragile             | **Never use**                        |

[⬆ Back to Top](#top)

---

## Focus Management

Focus management ensures users always know **where they are** and that focus moves logically through the interface.

### Visible Focus Indicators

```css
/* ❌ NEVER do this without a replacement */
*:focus {
  outline: none;
}

/* ✅ Custom focus styles (visible and clear) */
:focus-visible {
  outline: 3px solid #4A90D9;
  outline-offset: 2px;
  border-radius: 4px;
}

/* ✅ Differentiate mouse and keyboard focus */
button:focus:not(:focus-visible) {
  outline: none; /* Hide for mouse clicks */
}

button:focus-visible {
  outline: 3px solid #4A90D9; /* Show for keyboard */
}
```

### Skip Navigation Link

The **skip link** allows keyboard users to bypass repetitive navigation and jump directly to main content:

```html
<body>
  <a href="#main-content" class="skip-link">Skip to main content</a>
  <header>
    <nav><!-- Long navigation --></nav>
  </header>
  <main id="main-content" tabindex="-1">
    <!-- Page content -->
  </main>
</body>
```

```css
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  background: #000;
  color: #fff;
  padding: 8px 16px;
  z-index: 100;
  transition: top 0.2s;
}

.skip-link:focus {
  top: 0;
}
```

### Focus Trapping in Modals

When a modal/dialog is open, focus must be **trapped inside** it — Tab should cycle through the modal, not escape to content behind it.

```javascript
function trapFocus(modalElement) {
  const focusableElements = modalElement.querySelectorAll(
    'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
  );
  const firstFocusable = focusableElements[0];
  const lastFocusable = focusableElements[focusableElements.length - 1];

  modalElement.addEventListener('keydown', (e) => {
    if (e.key !== 'Tab') return;

    if (e.shiftKey) {
      // Shift + Tab: if on first element, wrap to last
      if (document.activeElement === firstFocusable) {
        e.preventDefault();
        lastFocusable.focus();
      }
    } else {
      // Tab: if on last element, wrap to first
      if (document.activeElement === lastFocusable) {
        e.preventDefault();
        firstFocusable.focus();
      }
    }
  });

  // Focus the first element on open
  firstFocusable.focus();
}
```

[⬆ Back to Top](#top)

---

## Color Contrast and Visual Design

### Contrast Requirements

| Level  | Normal Text (< 18px) | Large Text (≥ 18px or 14px bold) | UI Components |
| ------ | --------------------- | --------------------------------- | ------------- |
| **AA** | 4.5:1                 | 3:1                               | 3:1           |
| **AAA**| 7:1                   | 4.5:1                             | —             |

### Don't Rely on Color Alone

```html
<!-- ❌ Error communicated ONLY by color -->
<input style="border-color: red;" />

<!-- ✅ Error communicated by color + icon + text -->
<div class="field-error">
  <input style="border-color: red;" aria-invalid="true" aria-describedby="email-error" />
  <span id="email-error" role="alert">
    ⚠️ Please enter a valid email address.
  </span>
</div>
```

### Tools for Checking Contrast

| Tool                     | Type          | URL / Usage                                        |
| ------------------------ | ------------- | -------------------------------------------------- |
| **WebAIM Contrast Checker** | Web        | https://webaim.org/resources/contrastchecker/      |
| **Colour Contrast Analyser** | Desktop   | Free app by TPGi                                   |
| **Chrome DevTools**      | Browser       | Inspect element → Color picker shows ratio          |
| **Stark (Figma Plugin)** | Design Tool   | Check contrast during the design phase              |

[⬆ Back to Top](#top)

---

## Accessible Forms

Forms are the **most common source of accessibility failures**. Every input must have a programmatic label, clear instructions, and proper error handling.

### Labeling Inputs

```html
<!-- ✅ Method 1: Explicit label (BEST — highest support) -->
<label for="email">Email Address</label>
<input type="email" id="email" name="email" />

<!-- ✅ Method 2: Wrapping label -->
<label>
  Email Address
  <input type="email" name="email" />
</label>

<!-- ✅ Method 3: aria-label (use when no visible label exists) -->
<input type="search" aria-label="Search products" />

<!-- ✅ Method 4: aria-labelledby (pointing to existing text) -->
<h2 id="billing-heading">Billing Information</h2>
<input aria-labelledby="billing-heading" />

<!-- ❌ NEVER: Placeholder is NOT a label -->
<input type="email" placeholder="Enter your email" />
<!-- Placeholder disappears on type, not announced reliably -->
```

### Error Handling and Validation

```html
<form novalidate>
  <label for="email">Email Address <span aria-hidden="true">*</span></label>
  <input
    type="email"
    id="email"
    name="email"
    required
    aria-required="true"
    aria-invalid="false"
    aria-describedby="email-hint email-error"
  />
  <span id="email-hint" class="hint">e.g., name@example.com</span>
  <span id="email-error" class="error" role="alert" hidden>
    Please enter a valid email address.
  </span>
</form>
```

```javascript
// On form submission or blur
function validateEmail(input) {
  const errorEl = document.getElementById('email-error');
  const isValid = input.validity.valid;

  input.setAttribute('aria-invalid', !isValid);
  errorEl.hidden = isValid;

  if (!isValid) {
    input.focus(); // Move focus to the first invalid field
  }
}
```

### Form Accessibility Checklist

| Requirement                        | How to Implement                                    |
| ---------------------------------- | --------------------------------------------------- |
| Every input has a label            | `<label for="...">` or `aria-label`                 |
| Required fields are indicated      | `aria-required="true"` + visual indicator            |
| Error messages are associated      | `aria-describedby` pointing to error `<span>`        |
| Errors are announced               | `role="alert"` or `aria-live="assertive"` on error   |
| Group related fields               | `<fieldset>` + `<legend>`                            |
| Autocomplete hints are provided    | `autocomplete="email"`, `autocomplete="given-name"`  |

[⬆ Back to Top](#top)

---

## Accessible Images and Media

### Images

```html
<!-- ✅ Informative image — describe the content -->
<img src="chart.png" alt="Bar chart showing 40% increase in revenue in Q3 2025" />

<!-- ✅ Decorative image — empty alt to hide from screen readers -->
<img src="decorative-swirl.png" alt="" />

<!-- ✅ Complex image — provide a longer description -->
<figure>
  <img src="architecture-diagram.png" alt="System architecture diagram" aria-describedby="arch-desc" />
  <figcaption id="arch-desc">
    The system uses a React frontend, Node.js API gateway, and PostgreSQL database,
    connected through a Redis cache and WebSocket layer for real-time updates.
  </figcaption>
</figure>

<!-- ✅ Icon buttons — use aria-label -->
<button aria-label="Close dialog">
  <svg aria-hidden="true"><!-- X icon --></svg>
</button>

<!-- ❌ DON'T: Redundant alt text -->
<img src="logo.png" alt="Image of company logo image" />
<!-- ✅ DO: -->
<img src="logo.png" alt="Acme Corp logo" />
```

### Video and Audio

```html
<!-- ✅ Captions for video -->
<video controls>
  <source src="intro.mp4" type="video/mp4" />
  <track kind="captions" src="captions-en.vtt" srclang="en" label="English" default />
  <track kind="captions" src="captions-es.vtt" srclang="es" label="Spanish" />
</video>

<!-- ✅ Audio with transcript link -->
<audio controls>
  <source src="podcast.mp3" type="audio/mpeg" />
</audio>
<a href="/transcript/episode-42">Read the transcript</a>
```

[⬆ Back to Top](#top)

---

## Accessible Navigation and Routing (SPAs)

Single Page Applications (React, Angular, Vue) introduce unique challenges because **page transitions don't trigger a browser page load** — screen readers aren't notified of content changes.

### Announce Route Changes

```javascript
// React example using a live region
function RouteAnnouncer() {
  const location = useLocation();
  const [announcement, setAnnouncement] = useState('');

  useEffect(() => {
    // Get the page title or heading after navigation
    const pageTitle = document.title || 'Page loaded';
    setAnnouncement(`Navigated to ${pageTitle}`);
  }, [location]);

  return (
    <div
      role="status"
      aria-live="polite"
      aria-atomic="true"
      className="sr-only" // Visually hidden
    >
      {announcement}
    </div>
  );
}
```

### Manage Focus on Navigation

```javascript
// After route change, move focus to the main heading
useEffect(() => {
  const heading = document.querySelector('h1');
  if (heading) {
    heading.setAttribute('tabindex', '-1');
    heading.focus();
  }
}, [location]);
```

### Visually Hidden Utility Class (Screen Reader Only)

```css
/* Content is hidden visually but accessible to screen readers */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

[⬆ Back to Top](#top)

---

## Accessible Modals Dialogs and Popups

Modals are one of the **most commonly inaccessible components** on the web. Here's how to build them right.

### Using Native `<dialog>` (Modern Approach)

```html
<dialog id="confirm-dialog" aria-labelledby="dialog-title" aria-describedby="dialog-desc">
  <h2 id="dialog-title">Confirm Deletion</h2>
  <p id="dialog-desc">Are you sure you want to delete this item? This action cannot be undone.</p>
  <div class="dialog-actions">
    <button onclick="document.getElementById('confirm-dialog').close()">Cancel</button>
    <button onclick="deleteItem()">Delete</button>
  </div>
</dialog>

<button onclick="document.getElementById('confirm-dialog').showModal()">
  Delete Item
</button>
```

The native `<dialog>` element provides:
- ✅ Focus trapping automatically
- ✅ Escape to close
- ✅ `::backdrop` for overlay styling
- ✅ Proper `role="dialog"` built-in
- ✅ Restores focus to trigger element on close

### Custom Modal Checklist

If building custom modals, ensure you handle:

| Requirement                          | Implementation                                             |
| ------------------------------------ | ---------------------------------------------------------- |
| Focus is moved into modal on open    | Focus the first interactive element or the modal itself    |
| Focus is trapped inside modal        | Tab/Shift+Tab cycle within the modal                       |
| Escape key closes the modal          | `keydown` listener for `Escape`                            |
| Background content is inert          | `aria-hidden="true"` on content behind modal or `inert` attribute |
| Focus returns on close               | Save trigger reference, restore focus after closing         |
| Announced as a dialog                | `role="dialog"` + `aria-labelledby`                        |

[⬆ Back to Top](#top)

---

## Responsive and Zoom Friendly Design

### Zoom and Text Resize

WCAG requires that content remains usable when **zoomed to 200%** and text can be resized up to **200%** without loss of content or functionality.

```css
/* ✅ Use relative units */
body {
  font-size: 1rem;        /* Not px — respects user preferences */
}

h1 {
  font-size: 2.5rem;      /* Scales with user settings */
}

.container {
  max-width: 75rem;        /* 1200px equivalent, but scalable */
  padding: 1.5rem;
}

/* ❌ Avoid fixed heights on text containers */
.card {
  /* height: 200px;  — text will overflow on zoom */
  min-height: 200px;       /* ✅ Grows with content */
}
```

### Responsive Touch Targets

```css
/* WCAG 2.2 requires minimum 24x24px target size (Level AA) */
/* Best practice: 44x44px for touch targets */
button, a, input[type="checkbox"], input[type="radio"] {
  min-width: 44px;
  min-height: 44px;
}
```

### Orientation and Reflow

```css
/* Don't lock orientation */
/* ❌ */ @media (orientation: portrait) { /* Don't force portrait only */ }

/* ✅ Content reflows at 320px width (no horizontal scrolling) */
@media (max-width: 320px) {
  .grid {
    grid-template-columns: 1fr; /* Stack content vertically */
  }
}
```

[⬆ Back to Top](#top)

---

## How to Check and Audit Accessibility

### 1. Automated Testing Tools

Automated tools catch about **30–40% of accessibility issues** — they're fast but can't replace manual testing.

| Tool                          | Type                | What It Catches                                        |
| ----------------------------- | ------------------- | ------------------------------------------------------ |
| **axe DevTools**              | Browser Extension   | WCAG violations, best practices, color contrast        |
| **Lighthouse (Chrome)**       | Built-in DevTools   | Accessibility score + specific issues                  |
| **WAVE**                      | Browser Extension   | Visual overlay of errors, alerts, and structural info  |
| **eslint-plugin-jsx-a11y**    | Linter Plugin       | Catches issues at code-writing time (React)            |
| **axe-core**                  | CI/CD Integration   | Automated checks in your test pipeline                 |
| **Pa11y**                     | CLI / CI            | Command-line accessibility testing for CI pipelines    |

#### Running axe in Your Test Suite

```javascript
// Using @axe-core/react in development
import React from 'react';
import ReactDOM from 'react-dom';

if (process.env.NODE_ENV !== 'production') {
  const axe = require('@axe-core/react');
  axe(React, ReactDOM, 1000); // Logs violations in console
}
```

```javascript
// Using axe with Cypress for E2E testing
describe('Homepage Accessibility', () => {
  it('should have no a11y violations', () => {
    cy.visit('/');
    cy.injectAxe();
    cy.checkA11y();  // Fails test if violations found
  });

  it('should have no a11y violations on the modal', () => {
    cy.visit('/');
    cy.injectAxe();
    cy.get('[data-testid="open-modal"]').click();
    cy.checkA11y('#modal');  // Check specific container
  });
});
```

```javascript
// Using axe with Jest + Testing Library
import { axe, toHaveNoViolations } from 'jest-axe';
import { render } from '@testing-library/react';

expect.extend(toHaveNoViolations);

test('LoginForm should be accessible', async () => {
  const { container } = render(<LoginForm />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

### 2. Manual Testing (Keyboard)

This catches issues that automated tools miss entirely.

| Test                         | How                                                        | What to Check                        |
| ---------------------------- | ---------------------------------------------------------- | ------------------------------------ |
| **Tab through the page**     | Press `Tab` repeatedly                                     | All interactive elements are reached |
| **Reverse tab**              | Press `Shift + Tab`                                        | Focus moves backward logically       |
| **Activate elements**        | Press `Enter` / `Space`                                    | Buttons and links work               |
| **Navigate dropdowns**       | Use `Arrow` keys                                           | Options are selectable               |
| **Close overlays**           | Press `Escape`                                             | Modals and menus close               |
| **Skip link**                | Tab once on page load                                      | "Skip to content" link appears       |
| **Focus visibility**         | Tab through page                                           | Focus indicator always visible       |

### 3. Screen Reader Testing

| Screen Reader      | Platform     | Browser       | Cost     |
| ------------------ | ------------ | ------------- | -------- |
| **NVDA**           | Windows      | Firefox       | Free     |
| **JAWS**           | Windows      | Chrome, Edge  | Paid     |
| **VoiceOver**      | macOS / iOS  | Safari        | Built-in |
| **TalkBack**       | Android      | Chrome        | Built-in |
| **Narrator**       | Windows      | Edge          | Built-in |

> 💡 **Recommended minimum testing**: NVDA + Firefox on Windows and VoiceOver + Safari on macOS.

### 4. Lighthouse Accessibility Audit

```bash
# Run from CLI
npx lighthouse https://yoursite.com --only-categories=accessibility --output=html --output-path=./report.html

# Or in Chrome: DevTools → Lighthouse tab → Check "Accessibility" → Generate Report
```

### 5. CI/CD Integration

```yaml
# GitHub Actions example with Pa11y
name: Accessibility Check
on: [push, pull_request]

jobs:
  a11y:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - run: npm ci
      - run: npm run build
      - run: npm run start &

      - name: Run Pa11y tests
        run: npx pa11y-ci --config .pa11yci.json
```

```json
// .pa11yci.json
{
  "defaults": {
    "standard": "WCAG2AA",
    "timeout": 10000
  },
  "urls": [
    "http://localhost:3000/",
    "http://localhost:3000/login",
    "http://localhost:3000/dashboard"
  ]
}
```

[⬆ Back to Top](#top)

---

## How to Fix Accessibility Issues Step by Step

Follow this **prioritized approach** when fixing accessibility issues in an existing application:

### Step 1: Run an Automated Audit

```bash
# Option A: Browser — Open Chrome DevTools > Lighthouse > Accessibility
# Option B: axe DevTools extension — click "Scan" on any page
# Option C: CLI
npx pa11y https://yoursite.com
```

### Step 2: Fix Critical Issues First (Impact: Critical / Serious)

| Priority | Issue                          | Fix                                                              |
| -------- | ------------------------------ | ---------------------------------------------------------------- |
| 🔴 P0    | Missing page `lang`            | Add `<html lang="en">` to every page                            |
| 🔴 P0    | Images without `alt`           | Add descriptive `alt` or `alt=""` for decorative                 |
| 🔴 P0    | Form inputs without labels     | Add `<label for="...">` to every input                          |
| 🟠 P1    | Low color contrast             | Adjust colors to meet 4.5:1 ratio                               |
| 🟠 P1    | Non-keyboard-accessible controls | Replace `<div onClick>` with `<button>` or add keyboard handlers |
| 🟡 P2    | Missing heading hierarchy      | Ensure one `<h1>` per page, headings don't skip levels           |
| 🟡 P2    | Missing landmark regions       | Wrap content in `<header>`, `<main>`, `<nav>`, `<footer>`        |

### Step 3: Add Keyboard Support

- Test every interactive element with keyboard
- Add `tabindex="0"` to custom widgets
- Add keyboard event handlers (`Enter`, `Space`, `Escape`, arrow keys)
- Implement focus trapping in modals

### Step 4: Test with a Screen Reader

- Navigate your entire user journey with NVDA/VoiceOver
- Verify announcements make sense
- Check `aria-live` regions for dynamic updates
- Ensure form errors are announced

### Step 5: Set Up Ongoing Monitoring

- Add `eslint-plugin-jsx-a11y` to your linter
- Add `jest-axe` tests for every component
- Add Pa11y or axe-core to your CI pipeline
- Schedule quarterly manual audits

[⬆ Back to Top](#top)

---

## Accessibility in React Angular and Vue

### React

```jsx
// ✅ Fragment instead of extra div
return (
  <>
    <h1>Dashboard</h1>
    <DashboardContent />
  </>
);

// ✅ Use htmlFor instead of for
<label htmlFor="username">Username</label>
<input id="username" />

// ✅ Linting: eslint-plugin-jsx-a11y
// .eslintrc.json
{
  "extends": ["plugin:jsx-a11y/recommended"]
}

// ✅ Accessible component pattern
function Alert({ message, type = 'info' }) {
  return (
    <div role="alert" aria-live="assertive" className={`alert alert-${type}`}>
      {message}
    </div>
  );
}

// ✅ useId for unique accessible IDs (React 18+)
function FormField({ label }) {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}
```

### Angular

```typescript
// ✅ Use Angular CDK's A11yModule
import { A11yModule } from '@angular/cdk/a11y';

// ✅ Live announcer for dynamic updates
import { LiveAnnouncer } from '@angular/cdk/a11y';

@Component({ /* ... */ })
export class CartComponent {
  constructor(private liveAnnouncer: LiveAnnouncer) {}

  addToCart(item: string) {
    this.liveAnnouncer.announce(`${item} added to cart`);
  }
}

// ✅ Focus trap for modals
import { CdkTrapFocus } from '@angular/cdk/a11y';

// In template:
// <div cdkTrapFocus>
//   <h2>Modal Content</h2>
//   <button>Close</button>
// </div>
```

### Vue

```vue
<!-- ✅ vue-announcer for route changes -->
<template>
  <div>
    <VueAnnouncer />
    <router-view />
  </div>
</template>

<!-- ✅ Accessible component -->
<template>
  <div role="alert" aria-live="polite" v-if="error" class="error-banner">
    {{ error }}
  </div>
</template>

<!-- ✅ Focus management -->
<script setup>
import { ref, nextTick } from 'vue';

const headingRef = ref(null);

async function onNavigate() {
  await nextTick();
  headingRef.value?.focus();
}
</script>

<!-- ✅ eslint-plugin-vuejs-accessibility -->
<!-- .eslintrc.json -->
<!-- { "extends": ["plugin:vuejs-accessibility/recommended"] } -->
```

[⬆ Back to Top](#top)

---

## Accessibility Testing Checklist

Use this checklist for every component and page before release:

### Structure & Semantics
- [ ] Page has a `<html lang="...">` attribute
- [ ] One `<h1>` per page, heading levels don't skip
- [ ] Landmark regions are used (`<header>`, `<main>`, `<nav>`, `<footer>`)
- [ ] Lists use `<ul>/<ol>` + `<li>`
- [ ] Tables have `<th>` with `scope` attributes

### Images & Media
- [ ] All meaningful images have descriptive `alt` text
- [ ] Decorative images have `alt=""`
- [ ] Videos have captions/subtitles
- [ ] Audio has transcripts

### Forms
- [ ] Every input has a programmatic label
- [ ] Required fields are indicated (visually + `aria-required`)
- [ ] Error messages are associated via `aria-describedby`
- [ ] Errors are announced (`role="alert"`)
- [ ] `autocomplete` attributes are used where appropriate

### Keyboard & Focus
- [ ] All interactive elements are reachable via `Tab`
- [ ] Focus order is logical
- [ ] Focus indicator is always visible
- [ ] `Escape` closes modals/overlays
- [ ] Skip link is present and functional
- [ ] No keyboard traps (except intentional as in modals)

### Color & Visual
- [ ] Text contrast meets 4.5:1 (AA)
- [ ] UI component contrast meets 3:1
- [ ] Information is not conveyed by color alone
- [ ] Content is usable at 200% zoom
- [ ] No content requires horizontal scrolling at 320px

### Dynamic Content
- [ ] Route changes are announced in SPAs
- [ ] Toast/notifications use `aria-live`
- [ ] Loading states are communicated
- [ ] Animations can be paused (`prefers-reduced-motion`)

```css
/* Respect user's motion preferences */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

[⬆ Back to Top](#top)

---

## Key Interview Takeaways

| Topic                     | Key Point                                                                                    |
| ------------------------- | -------------------------------------------------------------------------------------------- |
| **What is A11y?**         | Making web apps usable by everyone, including people with disabilities                       |
| **Standard**              | WCAG 2.1 AA is the most common target                                                        |
| **Principles**            | POUR — Perceivable, Operable, Understandable, Robust                                         |
| **First Rule of ARIA**    | Don't use ARIA if native HTML can do the job                                                 |
| **Keyboard**              | All interactive elements must work with keyboard alone                                       |
| **Focus**                 | Always visible, logically ordered, trapped in modals                                         |
| **Forms**                 | Every input needs a label; errors must be announced                                          |
| **Color**                 | 4.5:1 contrast ratio for text; never use color as the only indicator                         |
| **Testing**               | Automated (axe, Lighthouse) catches ~30-40%; manual + screen reader testing is essential     |
| **SPAs**                  | Announce route changes, manage focus on navigation                                           |
| **CI/CD**                 | Integrate axe-core or Pa11y into your pipeline to prevent regressions                        |

[⬆ Back to Top](#top)

---

## Further Reading and Resources

| Resource                                | Link                                                      |
| --------------------------------------- | --------------------------------------------------------- |
| WCAG 2.2 Quick Reference                | https://www.w3.org/WAI/WCAG22/quickref/                   |
| WebAIM — Web Accessibility In Mind       | https://webaim.org/                                       |
| MDN — Accessibility                      | https://developer.mozilla.org/en-US/docs/Web/Accessibility |
| A11Y Project Checklist                   | https://www.a11yproject.com/checklist/                    |
| axe DevTools                             | https://www.deque.com/axe/devtools/                       |
| Inclusive Components (Heydon Pickering)  | https://inclusive-components.design/                       |
| WAI-ARIA Authoring Practices             | https://www.w3.org/WAI/ARIA/apg/                          |
| The WebAIM Million Report                | https://webaim.org/projects/million/                      |
| pa11y                                    | https://pa11y.org/                                        |
| eslint-plugin-jsx-a11y                   | https://github.com/jsx-eslint/eslint-plugin-jsx-a11y     |

---

> 🏁 **Accessibility is not a feature — it's a requirement.** Starting with semantic HTML, testing with keyboards and screen readers, and integrating automated checks into CI/CD will cover the vast majority of issues. Make it part of your definition of done, not an afterthought.



More Details:

Get all articles related to system design 
Hastag: SystemDesignWithZeeshanAli


[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design

[⬆ Back to Top](#top)
