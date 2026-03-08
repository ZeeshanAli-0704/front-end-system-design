# Image Optimization – Frontend Performance

Images are typically the largest assets on a web page, often accounting for 50-70% of total page weight. Optimizing images directly impacts LCP (Largest Contentful Paint), bandwidth usage, and overall user experience.

## Table of Contents

- [1. Image Formats (JPEG, PNG, WebP, AVIF, SVG)](#1-image-formats-jpeg-png-webp-avif-svg)
- [2. Responsive Images (srcset, sizes)](#2-responsive-images-srcset-sizes)
- [3. Lazy Loading Images](#3-lazy-loading-images)
- [4. Progressive and Interlaced Images](#4-progressive-and-interlaced-images)
- [5. Image Compression (Lossy vs Lossless)](#5-image-compression-lossy-vs-lossless)
- [6. Image CDN and Transformation](#6-image-cdn-and-transformation)
- [7. Art Direction (Picture Element)](#7-art-direction-picture-element)
- [8. Placeholder Strategies (LQIP, BlurHash, Dominant Color)](#8-placeholder-strategies-lqip-blurhash-dominant-color)
- [Key Takeaways](#key-takeaways)

---

## 1. Image Formats (JPEG, PNG, WebP, AVIF, SVG)

**Concept:**
Choosing the right image format is the first and most impactful optimization. Each format has specific strengths depending on the content type, transparency needs, and compression quality.

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

---

## 2. Responsive Images (srcset, sizes)

**Concept:**
Instead of serving one large image to all devices, serve different image resolutions based on the device's screen size and pixel density. A 4K hero image on a 320px mobile screen wastes bandwidth.

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

---

## 3. Lazy Loading Images

**Concept:**
Do not load images that are below the fold (not visible in the viewport). Load them only when the user scrolls near them. This reduces initial page weight and speeds up first render.

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

---

## 4. Progressive and Interlaced Images

**Concept:**
Normal images load top-to-bottom, line by line. Progressive images load as a blurry full image first, then sharpen progressively. This gives users a faster perceived experience.

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

---

## 5. Image Compression (Lossy vs Lossless)

**Concept:**
Compression reduces image file size. The key decision is how much quality loss is acceptable.

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

---

## 6. Image CDN and Transformation

**Concept:**
An Image CDN serves images from edge servers closest to the user and can dynamically resize, convert, and optimize images on-the-fly via URL parameters. No need to manually create multiple sizes.

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

## 7. Art Direction (Picture Element)

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

## 8. Placeholder Strategies (LQIP, BlurHash, Dominant Color)

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

## Key Takeaways

- Use modern formats (WebP, AVIF) with JPEG/PNG fallback
- Serve responsive images with `srcset` and `sizes`
- Lazy load below-the-fold images, prioritize above-the-fold
- Use progressive JPEG for perceived performance
- Compress at quality 75-85% (invisible loss, major size reduction)
- Use Image CDN for on-the-fly transformation
- Use art direction for different crops on different devices
- Show placeholders (LQIP, BlurHash) while loading

### Performance Metrics Impact

| Metric | Impact |
|--------|--------|
| LCP (Largest Contentful Paint) | +++ Major – images are often the LCP element |
| CLS (Cumulative Layout Shift) | ++ Moderate – proper dimensions and placeholders prevent shifts |
| FCP (First Contentful Paint) | + Minor – faster image starts mean earlier paints |
