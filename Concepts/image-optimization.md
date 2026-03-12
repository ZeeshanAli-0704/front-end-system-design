# Image Optimization – Frontend Performance

Images are typically the largest assets on a web page, often accounting for 50-70% of total page weight. Optimizing images directly impacts LCP (Largest Contentful Paint), bandwidth usage, and overall user experience.

## Table of Contents

- [1. Image Formats (JPEG, PNG, WebP, AVIF, SVG)](#1-image-formats-jpeg-png-webp-avif-svg)
- [2. Device Pixel Ratio (DPR)](#2-device-pixel-ratio-dpr)
- [3. Responsive Images (srcset, sizes)](#3-responsive-images-srcset-sizes)
- [4. Lazy Loading Images](#4-lazy-loading-images)
- [5. Progressive and Interlaced Images](#5-progressive-and-interlaced-images)
- [6. Image Compression (Lossy vs Lossless)](#6-image-compression-lossy-vs-lossless)
- [7. Image CDN and Transformation](#7-image-cdn-and-transformation)
- [8. Art Direction (Picture Element)](#8-art-direction-picture-element)
- [9. Placeholder Strategies (LQIP, BlurHash, Dominant Color)](#9-placeholder-strategies-lqip-blurhash-dominant-color)
- [10. CSS Sprite Images](#10-css-sprite-images)
- [11. Adaptive Media Loading](#11-adaptive-media-loading)
- [Key Takeaways](#key-takeaways)

---

## 1. Image Formats (JPEG, PNG, WebP, AVIF, SVG)

**Concept:**
Choosing the right image format is the first and most impactful optimization. Each format has specific strengths depending on the content type, transparency needs, and compression quality.

At a high level, image formats fall into two categories:
- **Raster formats** (JPEG, PNG, WebP, AVIF) – Made of pixels. They have a fixed resolution and become blurry when scaled beyond their native size. Best for photographs and complex visual content.
- **Vector formats** (SVG) – Made of mathematical paths and shapes. They scale infinitely without quality loss. Best for icons, logos, and illustrations.

The choice of format can affect file size by **2-10x** for the same visual quality, so it's worth getting right.

**Format Comparison:**

| Format | Best For | Transparency | Animation | Compression | Browser Support |
|--------|----------|-------------|-----------|-------------|-----------------|
| JPEG | Photographs, complex colors | No | No | Lossy | Universal |
| PNG | Icons, logos, transparency needed | Yes | No | Lossless | Universal |
| WebP | General purpose (photos + graphics) | Yes | Yes | Both lossy & lossless | 95%+ browsers |
| AVIF | High-quality photos at smallest size | Yes | Yes | Both lossy & lossless | 85%+ browsers |
| SVG | Icons, logos, illustrations | Yes | Yes (CSS/JS) | Vector (infinite scale) | Universal |

**When to use what:**
- JPEG → Product photos, hero banners, thumbnails
- PNG → Logos with transparency, screenshots, UI icons
- WebP → Default modern replacement for JPEG and PNG
- AVIF → When you need smallest file size and can provide fallback
- SVG → Icons, logos, illustrations that need to scale

**Example – Serving modern formats with fallback:**

```html
<picture>
  <source srcset="hero.avif" type="image/avif">
  <source srcset="hero.webp" type="image/webp">
  <img src="hero.jpg" alt="Hero banner">
</picture>
```

**How it works:**
- Browser checks `source` elements from top to bottom
- Picks the first format it supports
- Falls back to `img src` if no source is supported
- AVIF is ~50% smaller than JPEG, WebP is ~30% smaller than JPEG

**Format decision flowchart:**
```
Is it an icon/logo/illustration?
  ├── YES → Use SVG
  └── NO → Is it a photograph or complex image?
        ├── YES → Does the browser support AVIF?
        │     ├── YES → Use AVIF
        │     └── NO → Does the browser support WebP?
        │           ├── YES → Use WebP
        │           └── NO → Use JPEG
        └── NO → Does it need transparency?
              ├── YES → Use WebP (with PNG fallback)
              └── NO → Use WebP (with JPEG fallback)
```

---

## 2. Device Pixel Ratio (DPR)

**Concept:**
Device Pixel Ratio (DPR) is the ratio between **physical pixels** on the screen and **CSS (logical) pixels** used in layout. It tells you how many hardware pixels represent one CSS pixel.

```
DPR = Physical Pixels / CSS Pixels
```

**Why it matters:**
A device with DPR 2 (like Retina displays) has **4 times the pixels** (2x width × 2x height) compared to a DPR 1 screen for the same CSS area. If you serve a 400px-wide image to a 400px CSS container on a 2x screen, the image gets stretched across 800 physical pixels — resulting in a **blurry image**.

**Common DPR values:**

| Device | DPR | Physical Pixels for 400px CSS |
|--------|-----|-------------------------------|
| Standard desktop monitor | 1x | 400 × 400 |
| MacBook Retina / iPhone SE | 2x | 800 × 800 |
| iPhone Pro / Samsung Galaxy S | 3x | 1200 × 1200 |
| Some Android devices | 1.5x, 2.75x | Varies |

**Detecting DPR in JavaScript:**

```javascript
// Get current device pixel ratio
const dpr = window.devicePixelRatio;  // e.g., 2

console.log(`This device has a DPR of ${dpr}`);
// On a Retina MacBook → "This device has a DPR of 2"

// Listen for DPR changes (e.g., dragging window between monitors)
const mqList = window.matchMedia(`(resolution: ${dpr}dppx)`);
mqList.addEventListener('change', () => {
  console.log('DPR changed to:', window.devicePixelRatio);
});
```

**Detecting DPR in CSS:**

```css
/* Target high-DPR screens */
@media (-webkit-min-device-pixel-ratio: 2), (min-resolution: 192dpi) {
  .hero {
    background-image: url('hero@2x.jpg');
  }
}

@media (-webkit-min-device-pixel-ratio: 3), (min-resolution: 288dpi) {
  .hero {
    background-image: url('hero@3x.jpg');
  }
}
```

**Serving the right image based on DPR:**

The browser uses DPR internally when processing `srcset`. Here's the math:

```
Required image width = CSS display width × DPR

Example:
  CSS container = 400px wide
  Device DPR = 2
  Required image = 400 × 2 = 800px wide
```

```html
<!-- Density-based: explicitly tell browser which image is for which DPR -->
<img
  srcset="product.jpg 1x,
          product@2x.jpg 2x,
          product@3x.jpg 3x"
  src="product.jpg"
  alt="Product photo"
>

<!-- Width-based: browser calculates DPR automatically -->
<img
  srcset="product-400.jpg 400w,
          product-800.jpg 800w,
          product-1200.jpg 1200w"
  sizes="400px"
  src="product-400.jpg"
  alt="Product photo"
>
<!-- On a 2x device, browser picks product-800.jpg (400 × 2 = 800) -->
<!-- On a 3x device, browser picks product-1200.jpg (400 × 3 = 1200) -->
```

**Canvas rendering for high-DPR:**

Canvas elements also need DPR-aware sizing to avoid blurry rendering:

```javascript
const canvas = document.getElementById('myCanvas');
const ctx = canvas.getContext('2d');
const dpr = window.devicePixelRatio || 1;

// Set canvas size in physical pixels
canvas.width = 400 * dpr;
canvas.height = 300 * dpr;

// Scale CSS size back to logical pixels
canvas.style.width = '400px';
canvas.style.height = '300px';

// Scale all drawing operations
ctx.scale(dpr, dpr);

// Now draw at logical coordinates — output is crisp on Retina
ctx.fillRect(10, 10, 100, 100);
```

**Key rule:** Always serve images at **display size × DPR**. But cap at 2x — the visual difference between 2x and 3x is negligible for most images, while 3x files are significantly larger.

---

## 3. Responsive Images (srcset, sizes)

**Concept:**
Instead of serving one large image to all devices, serve different image resolutions based on the device's screen size and pixel density. A 4K hero image on a 320px mobile screen wastes bandwidth.

**The problem without responsive images:**
- A 1600px hero image is ~300KB as JPEG
- On a 320px mobile screen (even at 2x DPR), you only need 640px which is ~50KB
- You waste ~250KB per image × multiple images = MBs wasted on mobile
- This directly hurts LCP, increases data costs, and drains battery

**How srcset works:**
- `srcset` provides a list of image files with their widths or pixel densities
- Browser selects the most appropriate image based on viewport and DPR (Device Pixel Ratio)

**Example – Width-based srcset:**

```html
<img
  srcset="photo-400.jpg 400w,
          photo-800.jpg 800w,
          photo-1200.jpg 1200w,
          photo-1600.jpg 1600w"
  sizes="(max-width: 600px) 400px,
         (max-width: 1024px) 800px,
         1200px"
  src="photo-800.jpg"
  alt="Responsive photo"
>
```

**Breakdown:**
- `400w` means the image file is 400 pixels wide
- `sizes` tells the browser how wide the image will be displayed at different breakpoints
- Browser calculates: display width x DPR = picks the closest match
- On a 2x Retina phone at 400px display → picks `photo-800.jpg` (400 x 2 = 800)

**Example – Pixel density-based srcset:**

```html
<img
  srcset="logo.png 1x,
          logo@2x.png 2x,
          logo@3x.png 3x"
  src="logo.png"
  alt="Company logo"
>
```

**When to use which approach:**
- Width descriptors (`w`) → For fluid/responsive layout images
- Density descriptors (`x`) → For fixed-size images like logos

**How the browser selection algorithm works (step-by-step):**
1. Parse `sizes` attribute to determine **display width** at current viewport
2. Multiply display width by **DPR** → target pixel width
3. Look through `srcset` entries to find the **closest match** ≥ target
4. Download that single image

```
Example: Viewport = 500px, DPR = 2
  sizes says: (max-width: 600px) 400px → display width = 400px
  Target = 400 × 2 = 800px
  srcset: 400w, 800w, 1200w → browser picks 800w ✓
```

---

## 4. Lazy Loading Images

**Concept:**
Do not load images that are below the fold (not visible in the viewport). Load them only when the user scrolls near them. This reduces initial page weight and speeds up first render.

**Why it matters:**
- An average web page has 30-50 images, but only 3-5 are visible on initial load
- Without lazy loading, the browser downloads ALL images at page load
- This blocks bandwidth for critical resources (CSS, JS, above-the-fold images)
- Lazy loading can reduce initial page weight by **50-70%**

**Native lazy loading:**

```html
<img src="photo.jpg" loading="lazy" alt="A photo">
```

**How it works:**
- Browser defers loading until the image approaches the viewport
- Built-in to modern browsers (Chrome, Firefox, Edge, Safari)
- No JavaScript needed
- Browser uses internal threshold (typically ~1250px before viewport)

**Important rules:**
- NEVER lazy load above-the-fold images (hero, logo, first visible image)
- Above-the-fold images should use `loading="eager"` (default)
- Lazy loading above-the-fold images hurts LCP

**JavaScript-based lazy loading (for more control):**

```javascript
// Using Intersection Observer API
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src;        // Move data-src to src
      img.srcset = img.dataset.srcset;  // Move data-srcset to srcset
      observer.unobserve(img);           // Stop observing once loaded
    }
  });
}, {
  rootMargin: '200px'  // Start loading 200px before viewport
});

document.querySelectorAll('img[data-src]').forEach(img => {
  observer.observe(img);
});
```

```html
<img data-src="photo.jpg" data-srcset="photo-400.jpg 400w, photo-800.jpg 800w" alt="Lazy photo">
```

**When to use JS-based over native:**
- Need custom threshold distances
- Need loading animations or transitions
- Need to support older browsers
- Need callback when image loads

**fetchpriority for above-the-fold images:**

```html
<!-- Mark the LCP image as high priority -->
<img src="hero.jpg" fetchpriority="high" alt="Hero banner">

<!-- Explicitly lower priority for non-critical visible images -->
<img src="sidebar-ad.jpg" fetchpriority="low" alt="Ad">
```

`fetchpriority="high"` tells the browser to prioritize this image in the network queue, which directly improves LCP.

---

## 5. Progressive and Interlaced Images

**Concept:**
Normal images load top-to-bottom, line by line. Progressive images load as a blurry full image first, then sharpen progressively. This gives users a faster perceived experience.

This is fundamentally about **perceived performance** — the image isn't loading faster, but the user *perceives* it as faster because they see meaningful content sooner.

**How they differ:**

| Type | Loading Behavior | User Perception |
|------|-----------------|-----------------|
| Baseline JPEG | Top to bottom, row by row | Incomplete image until done |
| Progressive JPEG | Full blurry image → sharpens | See full image immediately |
| Interlaced PNG | Similar to progressive JPEG | Full image appears early |

**How progressive JPEG works internally:**
1. First pass → Very low quality full image (small data)
2. Second pass → Medium quality refinement
3. Third pass → Full quality final image

**Creating progressive images:**

```bash
# Using ImageMagick
convert input.jpg -interlace Plane output-progressive.jpg

# Using Sharp (Node.js)
sharp('input.jpg')
  .jpeg({ progressive: true, quality: 80 })
  .toFile('output.jpg');

# Using cjpeg
cjpeg -progressive input.bmp > output-progressive.jpg
```

**Benefits:**
- Perceived load time improves by 30-50%
- Users see content faster even on slow connections
- Progressive JPEGs are often slightly smaller than baseline
- Ideal for large hero images and product photos

**Trade-off awareness:**
- Progressive JPEGs require more **CPU to decode** (multiple passes)
- On very fast connections, baseline JPEG may actually appear faster
- On slow connections (3G/4G), progressive JPEG is significantly better
- Rule: Use progressive for images > 10KB

---

## 6. Image Compression (Lossy vs Lossless)

**Concept:**
Compression reduces image file size. The key decision is how much quality loss is acceptable.

Every image contains two types of data:
- **Visual data** — what humans actually see
- **Redundant data** — metadata (EXIF, color profiles), repeated patterns, imperceptible details

Lossy compression removes both redundant AND some visual data. Lossless removes only redundant data.

**Lossy compression:**
- Removes some image data permanently
- Significant file size reduction (60-80% smaller)
- Quality loss is often invisible at moderate settings
- Best for: photographs, hero images, thumbnails

**Lossless compression:**
- Removes redundant data without quality loss
- Moderate file size reduction (10-30% smaller)
- Pixel-perfect output
- Best for: logos, icons, screenshots, medical images

**Quality sweet spots:**

| Format | Recommended Quality | Typical Saving |
|--------|-------------------|----------------|
| JPEG | 75-85% | 60-70% smaller |
| WebP | 75-80% | 70-80% smaller vs JPEG |
| AVIF | 60-70% | 80-90% smaller vs JPEG |
| PNG | Lossless only | 10-30% with tools |

**Example – Compression with Sharp (Node.js):**

```javascript
const sharp = require('sharp');

// Lossy JPEG compression
sharp('input.jpg')
  .jpeg({ quality: 80 })
  .toFile('output.jpg');

// Lossy WebP compression
sharp('input.jpg')
  .webp({ quality: 75 })
  .toFile('output.webp');

// Lossy AVIF compression
sharp('input.jpg')
  .avif({ quality: 65 })
  .toFile('output.avif');

// Lossless PNG compression
sharp('input.png')
  .png({ compressionLevel: 9 })
  .toFile('output.png');
```

**Build-time compression (Webpack):**

```javascript
// webpack.config.js
const ImageMinimizerPlugin = require('image-minimizer-webpack-plugin');

module.exports = {
  optimization: {
    minimizer: [
      new ImageMinimizerPlugin({
        minimizer: {
          implementation: ImageMinimizerPlugin.sharpMinify,
          options: {
            encodeOptions: {
              jpeg: { quality: 80 },
              webp: { quality: 75 },
              avif: { quality: 65 },
            }
          }
        }
      })
    ]
  }
};
```

**Automation tip:** Never manually compress images. Integrate compression into your CI/CD pipeline or use an Image CDN that handles it automatically. Manual compression doesn't scale and is error-prone.

---

## 7. Image CDN and Transformation

**Concept:**
An Image CDN serves images from edge servers closest to the user and can dynamically resize, convert, and optimize images on-the-fly via URL parameters. No need to manually create multiple sizes.

**Why use an Image CDN instead of static optimization:**
- Static: You pre-generate 5 sizes × 3 formats = 15 files per image. For 1000 images = 15,000 files to manage
- CDN: Upload 1 original. CDN generates any variant on-demand via URL params
- CDN auto-detects browser support (serves AVIF to Chrome, WebP to Safari, JPEG to old browsers)
- CDN handles DPR-aware serving via `Accept` headers and Client Hints

**How it works:**
1. Upload original high-resolution image once
2. CDN generates optimized variants on request
3. Results are cached at edge locations worldwide
4. Browser gets the fastest, most optimized version

**Popular Image CDNs:**
- Cloudinary
- Imgix
- Cloudflare Images
- Akamai Image Manager
- Vercel Image Optimization (Next.js)

**Example – Cloudinary URL transformation:**

```
Original:
https://res.cloudinary.com/demo/image/upload/sample.jpg

Resized to 400px width:
https://res.cloudinary.com/demo/image/upload/w_400/sample.jpg

Resized + WebP format + quality 80:
https://res.cloudinary.com/demo/image/upload/w_400,f_webp,q_80/sample.jpg

Auto format + auto quality (browser-aware):
https://res.cloudinary.com/demo/image/upload/f_auto,q_auto/sample.jpg
```

**Example – Next.js built-in Image Optimization:**

```jsx
import Image from 'next/image';

function HeroSection() {
  return (
    <Image
      src="/hero.jpg"
      alt="Hero banner"
      width={1200}
      height={600}
      priority              // Above-the-fold, no lazy loading
      sizes="(max-width: 768px) 100vw, 1200px"
      quality={80}
    />
  );
}
```

**What Next.js Image does automatically:**
- Generates multiple sizes (srcset)
- Converts to WebP/AVIF when supported
- Lazy loads by default (unless `priority` is set)
- Prevents layout shift with width/height
- Serves from optimized CDN endpoint

---

## 8. Art Direction (Picture Element)

**Concept:**
Art direction is about showing a completely different image (not just a resized version) at different breakpoints. A wide landscape photo on desktop may need a tightly cropped portrait version on mobile.

**Difference from srcset:**
- `srcset` → Same image, different resolutions
- `picture` + art direction → Different images for different contexts

**Example:**

```html
<picture>
  <!-- Mobile: tightly cropped portrait -->
  <source media="(max-width: 600px)" srcset="hero-mobile.jpg">

  <!-- Tablet: medium crop -->
  <source media="(max-width: 1024px)" srcset="hero-tablet.jpg">

  <!-- Desktop: full wide landscape -->
  <img src="hero-desktop.jpg" alt="Product showcase">
</picture>
```

**Real-world example (e-commerce product):**
- Desktop → Full product with lifestyle background
- Tablet → Product centered, less background
- Mobile → Product close-up, no background

**Combining art direction with format switching:**

```html
<picture>
  <!-- Mobile + AVIF -->
  <source media="(max-width: 600px)" srcset="hero-mobile.avif" type="image/avif">
  <!-- Mobile + WebP -->
  <source media="(max-width: 600px)" srcset="hero-mobile.webp" type="image/webp">
  <!-- Mobile + JPEG fallback -->
  <source media="(max-width: 600px)" srcset="hero-mobile.jpg">

  <!-- Desktop + AVIF -->
  <source srcset="hero-desktop.avif" type="image/avif">
  <!-- Desktop + WebP -->
  <source srcset="hero-desktop.webp" type="image/webp">

  <!-- Ultimate fallback -->
  <img src="hero-desktop.jpg" alt="Product showcase">
</picture>
```

---

## 9. Placeholder Strategies (LQIP, BlurHash, Dominant Color)

**Concept:**
While the real image loads, show a lightweight placeholder to avoid blank space and layout shifts. This dramatically improves perceived performance.

**Three common strategies:**

**1. LQIP (Low Quality Image Placeholder):**
- Generate a tiny (20-40px wide) version of the image
- Display it blurred and scaled up
- Replace with full image once loaded
- Size: ~200-500 bytes inline as base64

```html
<!-- Inline tiny base64 LQIP -->
<img
  src="data:image/jpeg;base64,/9j/4AAQSkZJRg..."
  data-src="full-image.jpg"
  style="filter: blur(20px); transition: filter 0.3s;"
  alt="Product"
>
```

```javascript
// On full image load, swap and remove blur
const img = document.querySelector('img[data-src]');
const fullImage = new Image();
fullImage.onload = () => {
  img.src = img.dataset.src;
  img.style.filter = 'none';
};
fullImage.src = img.dataset.src;
```

**2. BlurHash:**
- Encode image into a short hash string (20-30 characters)
- Decode hash into a blurry color gradient on the client
- Very small payload, visually appealing
- Popular in mobile apps (Instagram, Wolt, Unsplash)

```
BlurHash string example: "LEHV6nWB2yk8pyo0adR*.7kCMdnj"
```

```javascript
// Using blurhash library
import { decode } from 'blurhash';

const pixels = decode('LEHV6nWB2yk8pyo0adR*.7kCMdnj', 32, 32);
// Returns Uint8ClampedArray of RGBA pixel data
// Render to canvas as placeholder
const canvas = document.createElement('canvas');
canvas.width = 32;
canvas.height = 32;
const ctx = canvas.getContext('2d');
const imageData = ctx.createImageData(32, 32);
imageData.data.set(pixels);
ctx.putImageData(imageData, 0, 0);
```

**3. Dominant Color:**
- Extract the main color of the image
- Show a solid color background as placeholder
- Smallest possible placeholder (just a hex color)
- Simple, lightweight, no decoding needed

```html
<div style="background-color: #2a6496; aspect-ratio: 16/9;">
  <img
    src="ocean.jpg"
    loading="lazy"
    alt="Ocean view"
    style="opacity: 0; transition: opacity 0.3s;"
    onload="this.style.opacity = 1"
  >
</div>
```

**Comparison:**

| Strategy | Payload Size | Visual Quality | Implementation |
|----------|-------------|---------------|----------------|
| LQIP | ~200-500 bytes | Good (blurred preview) | Moderate |
| BlurHash | ~20-30 chars | Good (color gradient) | Requires library |
| Dominant Color | ~7 chars (hex) | Basic (solid color) | Simplest |

---

## 10. CSS Sprite Images

**Concept:**
A CSS sprite is a single image file that contains multiple smaller images (icons, buttons, UI elements) arranged in a grid. Instead of making separate HTTP requests for each small image, you load one sprite sheet and use CSS `background-position` to display only the portion you need.

**The problem sprites solve:**
- Each image = 1 HTTP request
- A page with 30 small icons = 30 HTTP requests
- Even with HTTP/2 multiplexing, many small requests have overhead (headers, connection management)
- A single sprite sheet = 1 HTTP request for all 30 icons

**How it works:**
1. Combine many small images into one large image (the "sprite sheet")
2. Each element uses the sprite sheet as `background-image`
3. Use `background-position` to shift the visible window to the correct icon
4. Set `width` and `height` to match the individual icon size

**Visual explanation:**
```
Sprite Sheet (sprite.png):
┌──────┬──────┬──────┬──────┐
│ Home │ User │ Cart │ Star │   ← Row 0 (y: 0)
│ 32px │ 32px │ 32px │ 32px │
├──────┼──────┼──────┼──────┤
│ Mail │ Bell │ Gear │ Lock │   ← Row 1 (y: -32px)
│ 32px │ 32px │ 32px │ 32px │
└──────┴──────┴──────┴──────┘
  x:0   x:-32  x:-64  x:-96

To show "Cart": background-position: -64px 0;
To show "Bell": background-position: -32px -32px;
```

**CSS implementation:**

```css
/* Base sprite class */
.icon {
  display: inline-block;
  background-image: url('sprite.png');
  background-repeat: no-repeat;
  width: 32px;
  height: 32px;
}

/* Individual icon positions */
.icon-home  { background-position: 0 0; }
.icon-user  { background-position: -32px 0; }
.icon-cart  { background-position: -64px 0; }
.icon-star  { background-position: -96px 0; }
.icon-mail  { background-position: 0 -32px; }
.icon-bell  { background-position: -32px -32px; }
.icon-gear  { background-position: -64px -32px; }
.icon-lock  { background-position: -96px -32px; }
```

```html
<span class="icon icon-home"></span>
<span class="icon icon-cart"></span>
<span class="icon icon-bell"></span>
```

**Retina/High-DPR sprites:**

```css
/* Create a 2x sprite sheet (double the resolution) */
.icon {
  background-image: url('sprite.png');
  background-size: 128px 64px;  /* Half of actual sprite dimensions */
  width: 32px;
  height: 32px;
}

/* On high-DPR screens, use 2x sprite */
@media (-webkit-min-device-pixel-ratio: 2), (min-resolution: 192dpi) {
  .icon {
    background-image: url('sprite@2x.png');
    background-size: 128px 64px;  /* Scale down to logical size */
  }
}
```

**Generating sprites automatically:**

```javascript
// Using webpack-spritesmith
const SpritesmithPlugin = require('webpack-spritesmith');

module.exports = {
  plugins: [
    new SpritesmithPlugin({
      src: {
        cwd: path.resolve(__dirname, 'src/icons'),  // Folder with individual icons
        glob: '*.png'
      },
      target: {
        image: path.resolve(__dirname, 'src/assets/sprite.png'),
        css: path.resolve(__dirname, 'src/styles/sprite.css')
      },
      apiOptions: {
        cssImageRef: '../assets/sprite.png'
      }
    })
  ]
};
```

**When to use sprites vs. other techniques:**

| Technique | Best For | HTTP Requests | Scalability | Flexibility |
|-----------|----------|---------------|-------------|-------------|
| CSS Sprites | Small UI icons, decorative elements | 1 (excellent) | Fixed resolution | Limited |
| SVG Icons | Scalable icons, colored icons | 1 per icon (or inline) | Infinite | High |
| Icon Fonts | Monochrome icon sets | 1 font file | Infinite | Moderate |
| Inline SVG Sprite | Component-based apps (React, Vue) | 0 (bundled) | Infinite | Highest |

**Modern alternative – SVG sprite sheet:**

```html
<!-- Define SVG sprite (hidden, loaded once) -->
<svg xmlns="http://www.w3.org/2000/svg" style="display: none;">
  <symbol id="icon-home" viewBox="0 0 24 24">
    <path d="M12 3L2 12h3v8h6v-6h2v6h6v-8h3L12 3z"/>
  </symbol>
  <symbol id="icon-cart" viewBox="0 0 24 24">
    <path d="M7 18c-1.1 0-2 .9-2 2s.9 2 2 2 2-.9 2-2-.9-2-2-2zm10 0c-1.1 0-2 .9-2 2s.9 2 2 2 2-.9 2-2-.9-2-2-2z"/>
  </symbol>
</svg>

<!-- Use icons anywhere -->
<svg class="icon"><use href="#icon-home"/></svg>
<svg class="icon"><use href="#icon-cart"/></svg>
```

**Bottom line:** CSS sprites are still relevant for raster icon sets, but for new projects, prefer **SVG sprites** or **inline SVGs** for scalability and DPR independence. Use CSS sprites when working with photographic/raster UI elements that can't be vectorized.

---

## 11. Adaptive Media Loading

**Concept:**
Adaptive media loading is the practice of serving **different quality, size, or even type of media** based on the user's **device capabilities, network conditions, and preferences**. Instead of one-size-fits-all, you adapt the experience to each user's context.

**The three signals for adaptation:**

```
┌─────────────────────────────────────────────┐
│           Adaptive Media Loading             │
├──────────────┬──────────────┬───────────────┤
│   Network    │    Device    │     User      │
│  Conditions  │ Capabilities │  Preferences  │
├──────────────┼──────────────┼───────────────┤
│ • Connection │ • DPR        │ • Data Saver  │
│   type (4G,  │ • CPU cores  │   mode        │
│   3G, WiFi)  │ • Memory     │ • Reduced     │
│ • Bandwidth  │ • GPU        │   motion      │
│ • RTT        │ • Screen     │ • Prefers     │
│              │   size       │   contrast    │
└──────────────┴──────────────┴───────────────┘
```

### 11.1 Network-Based Adaptation

**Using the Network Information API:**

```javascript
const connection = navigator.connection || navigator.mozConnection || navigator.webkitConnection;

function getImageQuality() {
  if (!connection) return 'high';  // Default if API unavailable

  const { effectiveType, saveData } = connection;

  // User has Data Saver enabled
  if (saveData) return 'low';

  // Adapt based on connection speed
  switch (effectiveType) {
    case '4g':  return 'high';    // Full quality images
    case '3g':  return 'medium';  // Compressed images
    case '2g':  return 'low';     // Thumbnails or placeholders only
    case 'slow-2g': return 'none'; // Skip images entirely
    default:    return 'high';
  }
}

// Use it
const quality = getImageQuality();
const imageUrl = `https://cdn.example.com/photo.jpg?q=${quality === 'high' ? 80 : quality === 'medium' ? 50 : 20}`;
```

**Connection properties available:**

| Property | Description | Example Values |
|----------|-------------|----------------|
| `effectiveType` | Estimated connection type | `slow-2g`, `2g`, `3g`, `4g` |
| `downlink` | Estimated bandwidth (Mbps) | `1.5`, `10`, `50` |
| `rtt` | Estimated round-trip time (ms) | `50`, `200`, `800` |
| `saveData` | User enabled Data Saver | `true` / `false` |

**Listening for network changes:**

```javascript
navigator.connection.addEventListener('change', () => {
  const { effectiveType, downlink } = navigator.connection;
  console.log(`Network changed: ${effectiveType}, ${downlink}Mbps`);

  // Dynamically adjust image loading strategy
  updateImageStrategy(effectiveType);
});
```

### 11.2 Device Memory & Hardware Adaptation

```javascript
// Device Memory API (in GB)
const memory = navigator.deviceMemory || 4;  // Default 4GB if unsupported

// Hardware Concurrency (CPU cores)
const cores = navigator.hardwareConcurrency || 4;

function getMediaStrategy() {
  // Low-end device: < 2GB RAM or ≤ 2 cores
  if (memory <= 2 || cores <= 2) {
    return {
      imageQuality: 'low',
      lazyLoadMargin: '500px',    // Load earlier (less decoding at once)
      maxImagesPerPage: 10,        // Limit total images
      useBlurHash: false,          // Skip decode-heavy placeholders
      autoplayVideo: false         // Don't autoplay
    };
  }

  // Mid-range device
  if (memory <= 4 || cores <= 4) {
    return {
      imageQuality: 'medium',
      lazyLoadMargin: '300px',
      maxImagesPerPage: 30,
      useBlurHash: true,
      autoplayVideo: true
    };
  }

  // High-end device
  return {
    imageQuality: 'high',
    lazyLoadMargin: '200px',
    maxImagesPerPage: Infinity,
    useBlurHash: true,
    autoplayVideo: true
  };
}
```

### 11.3 Client Hints (Server-Side Adaptation)

Client Hints allow the **server** to adapt images without any JavaScript. The browser sends device info as HTTP headers, and the server responds with the optimal image.

**Opting in to Client Hints:**

```html
<!-- Tell the browser to send these hints -->
<meta http-equiv="Accept-CH" content="DPR, Width, Viewport-Width, Save-Data, ECT, Device-Memory">
```

**What the browser sends:**

```http
GET /images/hero.jpg HTTP/2
Accept: image/avif, image/webp, image/jpeg
DPR: 2
Width: 800
Viewport-Width: 1440
Save-Data: on
ECT: 4g
Device-Memory: 4
```

**Server can then:**
- See `DPR: 2` → serve 2x image
- See `Width: 800` → resize to 800px
- See `Save-Data: on` → serve heavily compressed version
- See `ECT: 2g` → serve tiny thumbnail
- See `Accept: image/avif` → serve AVIF format

### 11.4 CSS-Based Adaptation

```css
/* Prefer reduced data — user has Data Saver enabled */
@media (prefers-reduced-data: reduce) {
  .hero {
    background-image: url('hero-low.jpg');  /* Smaller image */
  }
  .decorative-bg {
    background-image: none;  /* Skip decorative images entirely */
  }
}

/* Prefer reduced motion — skip animated images */
@media (prefers-reduced-motion: reduce) {
  .animated-hero {
    animation: none;
  }
  img[src$=".gif"] {
    display: none;  /* Hide GIFs, show static alternative */
  }
}

/* Adapt to screen resolution/DPR */
@media (min-resolution: 2dppx) {
  .logo { background-image: url('logo@2x.png'); }
}
@media (min-resolution: 3dppx) {
  .logo { background-image: url('logo@3x.png'); }
}
```

### 11.5 React Implementation Example

```jsx
import { useState, useEffect } from 'react';

// Custom hook for adaptive loading
function useAdaptiveLoading() {
  const [config, setConfig] = useState({
    quality: 'high',
    loadImages: true,
    autoplay: true
  });

  useEffect(() => {
    const connection = navigator.connection;
    const memory = navigator.deviceMemory || 4;

    function updateConfig() {
      const saveData = connection?.saveData;
      const ect = connection?.effectiveType;

      setConfig({
        quality: saveData ? 'low' : ect === '3g' ? 'medium' : 'high',
        loadImages: ect !== 'slow-2g',       // Skip images on very slow
        autoplay: memory > 2 && ect === '4g' // Only autoplay on capable devices
      });
    }

    updateConfig();
    connection?.addEventListener('change', updateConfig);
    return () => connection?.removeEventListener('change', updateConfig);
  }, []);

  return config;
}

// Usage in component
function ProductImage({ src, alt }) {
  const { quality, loadImages } = useAdaptiveLoading();

  if (!loadImages) {
    return <div className="image-placeholder">{alt}</div>;
  }

  const qualityMap = { high: 80, medium: 50, low: 20 };
  const optimizedSrc = `${src}?q=${qualityMap[quality]}&f=auto`;

  return <img src={optimizedSrc} alt={alt} loading="lazy" />;
}
```

**Real-world usage:**
- **Instagram** → Reduces image quality on slow connections
- **YouTube** → Adjusts video resolution based on bandwidth (adaptive bitrate)
- **Twitter/X** → Shows "Load images" button on Data Saver mode
- **Google Search** → Serves lighter pages on 2G connections

**Key principle:** Adaptive loading is not about degrading the experience — it's about delivering **the best possible experience for each user's context**.

---

## Key Takeaways

- Use modern formats (WebP, AVIF) with JPEG/PNG fallback
- Understand DPR — serve images at **display size × DPR** (cap at 2x)
- Serve responsive images with `srcset` and `sizes`
- Lazy load below-the-fold images, use `fetchpriority="high"` for LCP images
- Use progressive JPEG for perceived performance on slow networks
- Compress at quality 75-85% (invisible loss, major size reduction)
- Use Image CDN for on-the-fly transformation and format negotiation
- Use art direction for different crops on different devices
- Show placeholders (LQIP, BlurHash) while loading
- Use CSS sprites for small raster icon sets; prefer SVG sprites for new projects
- Implement adaptive media loading — adapt quality based on network, device, and user preferences
- Use Client Hints to let the server optimize without client-side JavaScript

### Performance Metrics Impact

| Metric | Impact |
|--------|--------|
| LCP (Largest Contentful Paint) | +++ Major – images are often the LCP element |
| CLS (Cumulative Layout Shift) | ++ Moderate – proper dimensions and placeholders prevent shifts |
| FCP (First Contentful Paint) | + Minor – faster image starts mean earlier paints |
