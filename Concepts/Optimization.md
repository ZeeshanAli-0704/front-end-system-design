# Asset Optimization – Frontend Performance

> This is the index page for all asset optimization articles. Each topic is covered in a dedicated article with concepts, syntax, and examples.

---

## Articles

### 1. [Image Optimization](Image_Optimization.md)
- Image Formats (JPEG, PNG, WebP, AVIF, SVG)
- Device Pixel Ratio (DPR)
- Responsive Images (srcset, sizes)
- Lazy Loading Images
- Progressive and Interlaced Images
- Image Compression (Lossy vs Lossless)
- Image CDN and Transformation
- Art Direction (Picture Element)
- Placeholder Strategies (LQIP, BlurHash, Dominant Color)
- CSS Sprite Images
- Adaptive Media Loading

### 2. [Video Optimization](Video_Optimization.md)
- Progressive Enhancement for Video
- Video Formats (WebM, MP4)
- Progressive Poster Images
- Remove Audio from Videos
- Streaming Algorithms and Techniques
- Platform-Based Video Dimensions
- Video Preloading Strategy

### 3. [Font Optimization](Font_Optimization.md)
- Common Font Loading Issues (FOUT, FOIT)
- Font Display Strategy
- Font Format Optimization (WOFF2, WOFF)
- Font Preloading
- Font Face Observer

### 4. [CSS, JavaScript & UI Optimization](CSS_JS_UI_Optimization.md)
- Critical CSS Rendering
- Non-Critical CSS (Async Loading)
- Lazy Loading CSS (Media-Based)
- JS-Based CSS Loading
- Async and Defer Strategy
- Module Loading and Tree Shaking
- Configurable UI (Conditional Rendering)
- Streaming UI Rendering

### 5. [Network Optimization](Network_Optimization.md)
- HTTP/2 and HTTP/3
- Resource Hints (preload, prefetch, preconnect, dns-prefetch)
- Caching Strategies
- Compression (Gzip, Brotli)
- CDN (Content Delivery Network)
- Bundle Splitting and Code Splitting
- Service Workers and Offline Caching

---

## Quick Reference – Performance Metrics Impact

| Optimization | LCP | FCP | CLS | TTI | TTFB |
|-------------|-----|-----|-----|-----|------|
| Image optimization | +++ | + | ++ | | |
| Video optimization | ++ | | + | | + |
| Font optimization | + | ++ | ++ | | |
| Critical CSS | | +++ | + | | |
| JS async/defer | | + | | +++ | |
| Code splitting | | | | +++ | |
| Streaming SSR | | ++ | | ++ | +++ |
| Compression | + | + | | + | ++ |
| CDN | ++ | ++ | | + | +++ |
| Caching | +++ | +++ | | +++ | +++ |

`+++` = Major impact, `++` = Moderate impact, `+` = Minor impact
