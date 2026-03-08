# Font Optimization – Frontend Performance

Custom fonts enhance brand identity but can severely hurt performance. An unoptimized font setup can delay text rendering by 1-3 seconds, causing Flash of Invisible Text (FOIT) or Flash of Unstyled Text (FOUT).

## Table of Contents

- [1. Common Font Loading Issues (FOUT, FOIT)](#1-common-font-loading-issues-fout-foit)
- [2. Font Display Strategy](#2-font-display-strategy)
- [3. Font Format Optimization (WOFF2, WOFF)](#3-font-format-optimization-woff2-woff)
- [4. Font Preloading](#4-font-preloading)
- [5. Font Face Observer](#5-font-face-observer)
- [Key Takeaways](#key-takeaways)

---

## 1. Common Font Loading Issues (FOUT, FOIT)

**Concept:**
When a browser encounters custom fonts, it must decide what to show while the font downloads. This creates two well-known problems:

**FOUT (Flash of Unstyled Text):**

```
Timeline:
[0ms]    HTML parsed, text visible in fallback font (e.g., Arial)
[800ms]  Custom font downloads
[800ms]  Text re-renders with custom font → visible style jump
```

- Text is always visible (good for readability)
- Causes a visible style change (bad for visual polish)
- Layout may shift if fallback and custom font have different metrics

**FOIT (Flash of Invisible Text):**

```
Timeline:
[0ms]     HTML parsed, text is HIDDEN (invisible)
[800ms]   Custom font downloads
[800ms]   Text appears with custom font
[3000ms]  If font fails → fallback shows after timeout (browser-dependent)
```

- Text is hidden until font loads (bad for readability)
- No style flash (good for visual consistency)
- Can block reading for seconds on slow networks

**Which is worse?**
- FOIT is generally worse – users see blank space and may think content is missing
- FOUT is acceptable – content is readable from the start
- Best approach: Control the behavior explicitly with `font-display`

---

## 2. Font Display Strategy

**Concept:**
The `font-display` CSS property controls how a font face is displayed based on whether and when it is downloaded and ready to use.

**font-display values:**

| Value | Block Period | Swap Period | Behavior |
|-------|------------|------------|----------|
| `auto` | Browser decides | Browser decides | Default, unpredictable |
| `block` | Short (3s) | Infinite | FOIT – hides text, then swaps |
| `swap` | None | Infinite | FOUT – shows fallback, then swaps |
| `fallback` | Very short (100ms) | Short (3s) | Brief FOIT, then fallback permanently |
| `optional` | Very short (100ms) | None | Uses cached font or fallback, no swap |

**Example:**

```css
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2'),
       url('/fonts/custom.woff') format('woff');
  font-display: swap;   /* Show fallback immediately, swap when ready */
  font-weight: 400;
  font-style: normal;
}
```

**When to use each value:**
- `swap` → Body text, headings – content must be readable immediately
- `optional` → Non-critical decorative fonts – use if cached, skip if not
- `fallback` → Important text where brief invisible flash is acceptable
- `block` → Icon fonts where fallback would show wrong characters

**Recommended approach for most sites:**

```css
/* Body font → must be readable immediately */
@font-face {
  font-family: 'BodyFont';
  src: url('body.woff2') format('woff2');
  font-display: swap;
}

/* Decorative font → optional, use if cached */
@font-face {
  font-family: 'FancyHeading';
  src: url('fancy.woff2') format('woff2');
  font-display: optional;
}
```

---

## 3. Font Format Optimization (WOFF2, WOFF)

**Concept:**
Font file format directly impacts download size. WOFF2 uses Brotli compression and is 30% smaller than WOFF, which uses gzip compression.

**Format comparison:**

| Format | Compression | Size (typical) | Browser Support |
|--------|-------------|---------------|-----------------|
| TTF/OTF | None | 100% (baseline) | Universal |
| WOFF | gzip | ~60% of TTF | 98%+ browsers |
| WOFF2 | Brotli | ~40% of TTF | 96%+ browsers |
| EOT | Proprietary | ~70% of TTF | IE only (dead) |

**Example – Modern @font-face with format priority:**

```css
@font-face {
  font-family: 'CustomFont';
  src: url('font.woff2') format('woff2'),   /* Best: try first */
       url('font.woff') format('woff');      /* Fallback */
  font-display: swap;
}
```

**Additional font file optimizations:**

**Subsetting – Remove unused characters:**

```bash
# Using pyftsubset (fonttools)
pyftsubset font.ttf \
  --output-file=font-subset.woff2 \
  --flavor=woff2 \
  --text="ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789"

# Result: 80-90% smaller for Latin-only sites
```

**Variable fonts – One file for all weights:**

```css
/* Instead of loading 4 separate font files for 4 weights... */
@font-face {
  font-family: 'Inter';
  src: url('Inter-Variable.woff2') format('woff2-variations');
  font-weight: 100 900;  /* Supports entire weight range */
  font-display: swap;
}

/* Use any weight */
h1 { font-weight: 700; }
p  { font-weight: 400; }
```

- One variable font file replaces 4-8 individual files
- Total size is typically smaller than 2-3 static files combined

---

## 4. Font Preloading

**Concept:**
Fonts are discovered late in the rendering process (CSS must be parsed first). Preloading tells the browser to start downloading the font file early, before CSS parsing discovers it.

**Normal font loading timeline:**

```
HTML downloaded → CSS downloaded → CSS parsed → Font URL found → Font download starts
                                                                  ^^^^ Late discovery!
```

**With preloading:**

```
HTML downloaded → Font download starts immediately (parallel with CSS)
```

**Example:**

```html
<head>
  <!-- Preload critical fonts -->
  <link rel="preload" href="/fonts/body-font.woff2" as="font" type="font/woff2" crossorigin>
  <link rel="preload" href="/fonts/heading-font.woff2" as="font" type="font/woff2" crossorigin>

  <!-- Regular CSS that uses these fonts -->
  <link rel="stylesheet" href="/styles/main.css">
</head>
```

**Important rules:**
- `crossorigin` attribute is required even for same-origin fonts (font spec requirement)
- `as="font"` tells the browser it is a font resource (correct priority)
- `type="font/woff2"` lets the browser skip if it doesn't support WOFF2
- Only preload fonts actually used on the current page
- Preloading too many fonts wastes bandwidth and hurts other resources

**What to preload (and what not to):**

| Preload | Don't Preload |
|---------|--------------|
| Body text font (used on every page) | Fonts for rarely visited pages |
| Primary heading font | Icon fonts loaded later |
| Above-the-fold fonts | Secondary decorative fonts |

---

## 5. Font Face Observer

**Concept:**
A JavaScript library that detects when a specific font has fully loaded and is ready to use. This gives you programmatic control over font-dependent styling and transitions.

**Why use it:**
- CSS `font-display` gives basic control, but no JavaScript hooks
- Font Face Observer lets you run code exactly when a font loads
- Useful for adding/removing CSS classes, triggering animations, removing placeholders

**Example:**

```javascript
import FontFaceObserver from 'fontfaceobserver';

// Create observers for each font
const bodyFont = new FontFaceObserver('CustomBody');
const headingFont = new FontFaceObserver('CustomHeading', { weight: 700 });

// Wait for fonts to load
Promise.all([
  bodyFont.load(null, 5000),     // 5s timeout
  headingFont.load(null, 5000)
]).then(() => {
  // Fonts loaded – apply custom font class
  document.documentElement.classList.add('fonts-loaded');
}).catch(() => {
  // Fonts failed – keep fallback, no flash
  document.documentElement.classList.add('fonts-failed');
});
```

```css
/* Default: fallback font */
body {
  font-family: Arial, sans-serif;
}

/* Applied only after fonts load */
.fonts-loaded body {
  font-family: 'CustomBody', Arial, sans-serif;
}

.fonts-loaded h1, .fonts-loaded h2 {
  font-family: 'CustomHeading', Arial, sans-serif;
}
```

**Benefits of this approach:**
- Zero FOIT (text is always visible)
- Controlled font swap with smooth transition
- Can add CSS transition for opacity/color during swap
- Graceful degradation if font fails to load
- Can combine with localStorage to remember if fonts are cached

**Advanced – Cache-aware font loading:**

```javascript
// Check if fonts are already cached
if (sessionStorage.getItem('fonts-loaded')) {
  document.documentElement.classList.add('fonts-loaded');
} else {
  const font = new FontFaceObserver('CustomBody');
  font.load().then(() => {
    document.documentElement.classList.add('fonts-loaded');
    sessionStorage.setItem('fonts-loaded', 'true');
  });
}
```

---

## Key Takeaways

- Use `font-display: swap` for body text to avoid FOIT
- Prefer WOFF2 format (30% smaller than WOFF, 60% smaller than TTF)
- Preload critical fonts in `<head>` with `crossorigin` attribute
- Subset fonts to remove unused characters (80-90% smaller)
- Use variable fonts to replace multiple font weight files
- Use Font Face Observer for precise JavaScript-based font loading control
- Cache-aware loading prevents redundant font downloads on repeat visits

### Performance Metrics Impact

| Metric | Impact |
|--------|--------|
| FCP (First Contentful Paint) | ++ Moderate – font-display and preloading speed up text rendering |
| CLS (Cumulative Layout Shift) | ++ Moderate – font swapping causes layout shifts if metrics differ |
| LCP (Largest Contentful Paint) | + Minor – text elements can be LCP if they are the largest visible element |
