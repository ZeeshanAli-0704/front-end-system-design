# Video Optimization – Frontend Performance

Video content is the heaviest asset type on the web. A single unoptimized video can be larger than all other page assets combined. Optimization directly impacts load time, bandwidth costs, and mobile data usage.

<a id="top"></a>

## Table of Contents

- [Progressive Enhancement for Video](#progressive-enhancement-for-video)
- [Video Formats (WebM, MP4)](#video-formats-webm-mp4)
- [Progressive Poster Images](#progressive-poster-images)
- [Remove Audio from Videos](#remove-audio-from-videos)
- [Streaming Algorithms and Techniques](#streaming-algorithms-and-techniques)
- [Platform Based Video Dimensions](#platform-based-video-dimensions)
- [Video Preloading Strategy](#video-preloading-strategy)
- [Lazy Loading Video with Intersection Observer](#lazy-loading-video-with-intersection-observer)
- [Video Accessibility](#video-accessibility)
- [Video Compression Comparison](#video-compression-comparison)
- [Key Takeaways](#key-takeaways)


[⬆ Back to Top](#top)

---

## Progressive Enhancement for Video

**Concept:**
Provide multiple video formats and let the browser select the best one it supports. This ensures compatibility across browsers while using the most efficient codec when available.

**How it works:**
- List sources from most efficient to most compatible
- Browser picks the first format it can play
- If no source is supported, fallback text is shown

**Example:**

```html
<video controls width="720" height="405">
  <source src="video.av1.mp4" type="video/mp4; codecs=av01.0.05M.08">
  <source src="video.webm" type="video/webm">
  <source src="video.mp4" type="video/mp4">
  <p>Your browser does not support HTML5 video.</p>
</video>
```

**Why HTML5 video over animated GIFs:**

| Feature | Animated GIF | HTML5 Video |
|---------|-------------|-------------|
| File size (30s clip) | 10-50 MB | 1-5 MB |
| Color depth | 256 colors | Millions |
| Frame rate | Choppy | Smooth 30-60fps |
| CPU/GPU decoding | CPU only | GPU-accelerated |
| Controls | None | Play/pause/seek |
| Accessibility | None | Captions, audio desc |

**Replace GIFs with auto-playing muted video:**

```html
<video autoplay loop muted playsinline>
  <source src="animation.webm" type="video/webm">
  <source src="animation.mp4" type="video/mp4">
</video>
```

- `autoplay` → Starts automatically (requires `muted`)
- `loop` → Repeats continuously
- `muted` → No audio (required for autoplay)
- `playsinline` → Prevents fullscreen on iOS

[⬆ Back to Top](#top)

---

## Video Formats (WebM, MP4)

**Concept:**
Different video containers and codecs offer different compression efficiency and browser support trade-offs.

**Format comparison:**

| Format | Codec | Compression | Support | Use Case |
|--------|-------|-------------|---------|----------|
| MP4 (H.264) | AVC | Good | Universal | Default fallback |
| MP4 (H.265) | HEVC | Better (~40% smaller) | Safari, some browsers | Apple ecosystem |
| WebM (VP9) | VP9 | Better (~30-40% smaller) | Chrome, Firefox, Edge | Modern web |
| MP4 (AV1) | AV1 | Best (~50% smaller) | Chrome, Firefox | Next-gen |

**Encoding for web (FFmpeg):**

```bash
# MP4 (H.264) – universal fallback
ffmpeg -i input.mov -c:v libx264 -crf 23 -preset slow -c:a aac -b:a 128k output.mp4

# WebM (VP9) – better compression
ffmpeg -i input.mov -c:v libvpx-vp9 -crf 30 -b:v 0 -c:a libopus -b:a 128k output.webm

# MP4 (AV1) – best compression
ffmpeg -i input.mov -c:v libaom-av1 -crf 30 -c:a libopus output.av1.mp4
```

- `crf` = Constant Rate Factor (lower = better quality, bigger file)
- H.264 sweet spot: CRF 18-28
- VP9 sweet spot: CRF 25-35
- AV1 sweet spot: CRF 25-35

[⬆ Back to Top](#top)

---

## Progressive Poster Images

**Concept:**
A poster image is a still frame displayed before the video starts playing. It provides immediate visual context while the video data loads.

**How it works:**
1. Browser shows the poster image immediately
2. Video data downloads in the background
3. Once user clicks play (or autoplay starts), poster is replaced by video

**Example:**

```html
<video
  poster="video-poster.jpg"
  controls
  width="720"
  height="405"
  preload="metadata"
>
  <source src="video.webm" type="video/webm">
  <source src="video.mp4" type="video/mp4">
</video>
```

**Best practices for poster images:**
- Use the same aspect ratio as the video to avoid layout shift
- Compress the poster image (WebP format, quality 75-80)
- Use responsive poster images for different devices
- Choose a representative frame (not a black frame)

**Responsive poster with JavaScript:**

```javascript
const video = document.querySelector('video');
const isMobile = window.innerWidth < 768;

video.poster = isMobile
  ? 'poster-mobile-400.jpg'
  : 'poster-desktop-1200.jpg';
```

[⬆ Back to Top](#top)

---

## Remove Audio from Videos

**Concept:**
When a video has no meaningful sound (background videos, decorative loops, animations), removing the audio track reduces file size by 10-30%.

**Common use cases:**
- Hero background videos
- Product animation loops
- Loading/transition animations
- Decorative ambient videos

**How to remove audio:**

```bash
# FFmpeg – remove audio track entirely
ffmpeg -i input.mp4 -an -c:v copy output-no-audio.mp4

# -an = no audio
# -c:v copy = keep video codec as-is (fast, no re-encoding)
```

**In HTML – mute at playback level:**

```html
<video autoplay loop muted playsinline>
  <source src="background.webm" type="video/webm">
  <source src="background.mp4" type="video/mp4">
</video>
```

**Important distinction:**
- `muted` attribute → Audio track still exists in file, just silenced (still wastes bytes)
- Removing audio at encoding → Track physically removed, smaller file size
- Always do both: remove at encoding AND set `muted` in HTML

[⬆ Back to Top](#top)

---

## Streaming Algorithms and Techniques

**Concept:**
Instead of downloading the entire video file before playback, streaming delivers video in small chunks progressively. The user can start watching almost immediately.

**Types of video delivery:**

| Method | Behavior | Use Case |
|--------|----------|----------|
| Full download | Download entire file, then play | Small files only |
| Progressive download | Play while downloading sequentially | Simple video hosting |
| Adaptive streaming (HLS/DASH) | Adjusts quality based on bandwidth | Production video platforms |

**HLS (HTTP Live Streaming):**

```
How it works:
1. Video is split into small segments (2-10 seconds each)
2. Multiple quality levels are encoded (360p, 720p, 1080p)
3. A manifest file (.m3u8) lists all segments and qualities
4. Player monitors bandwidth and switches quality dynamically
```

**Simplified HLS flow:**

```
User clicks play
  → Player downloads manifest (.m3u8)
  → Player checks network speed
  → Downloads first segment at estimated quality
  → Measures actual download speed
  → Adjusts quality up/down for next segments
  → Continues seamlessly
```

**Example – Using HLS.js player:**

```html
<video id="video" controls></video>

<script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
<script>
  const video = document.getElementById('video');
  const videoSrc = 'https://example.com/video/playlist.m3u8';

  if (Hls.isSupported()) {
    const hls = new Hls();
    hls.loadSource(videoSrc);
    hls.attachMedia(video);
  } else if (video.canPlayType('application/vnd.apple.mpegurl')) {
    // Safari has native HLS support
    video.src = videoSrc;
  }
</script>
```

**Benefits of adaptive streaming:**
- Playback starts in seconds, not minutes
- Smoothly adjusts quality on fluctuating networks
- Reduces buffering events by 80-90%
- Users on fast networks get HD, slow networks get SD automatically

### HLS vs DASH

There are two main adaptive streaming protocols: **HLS** (HTTP Live Streaming) by Apple and **DASH** (Dynamic Adaptive Streaming over HTTP), an open international standard.

| Feature | HLS | DASH |
|---------|-----|------|
| **Creator** | Apple | MPEG (open standard) |
| **Manifest format** | `.m3u8` (text-based playlist) | `.mpd` (XML-based) |
| **Segment format** | `.ts` (MPEG-TS) or `.fmp4` | `.mp4` (fMP4) or `.webm` |
| **Native browser support** | Safari, iOS, most mobile | None (requires JS player) |
| **Codec support** | H.264, H.265, fMP4(AV1) | Any codec (H.264, VP9, AV1, etc.) |
| **DRM support** | FairPlay | Widevine, PlayReady |
| **Latency** | Higher (6-30s default, low-latency HLS reduces to ~2s) | Lower (2-5s typical) |
| **Adoption** | Dominant for live + VOD (Apple ecosystem) | Common for VOD, YouTube uses DASH |
| **JS library needed** | hls.js (for non-Safari) | dash.js (for all browsers) |

**When to use which:**
- **HLS** — Best default for most projects. Native iOS/Safari support means fewer edge cases. hls.js handles all other browsers.
- **DASH** — Better when you need broader codec support (VP9, AV1), lower latency, or DRM with Widevine. YouTube and Netflix use DASH.
- **Both** — Large platforms (Netflix, Disney+) encode in both and select per device.

**DASH example with dash.js:**

```html
<video id="video" controls></video>

<script src="https://cdn.dashjs.org/latest/dash.all.min.js"></script>
<script>
  const player = dashjs.MediaPlayer().create();
  player.initialize(
    document.getElementById('video'),
    'https://example.com/video/manifest.mpd',
    true  // autoplay
  );
</script>
```

[⬆ Back to Top](#top)

---

## Platform Based Video Dimensions

**Concept:**
Serve different video resolutions based on the device and screen size. A mobile phone does not need a 4K video. Serving appropriately sized video saves bandwidth and decoding cost.

**Dimension recommendations:**

| Platform | Max Resolution | Bitrate Range |
|----------|---------------|---------------|
| Mobile (portrait) | 480p (480x854) | 500kbps - 1.5Mbps |
| Mobile (landscape) | 720p (720x1280) | 1-2.5Mbps |
| Tablet | 720p-1080p | 2-5Mbps |
| Desktop | 1080p (1920x1080) | 3-8Mbps |
| Large display | 4K (3840x2160) | 10-25Mbps |

**Example – JavaScript-based source selection:**

```javascript
function getVideoSource() {
  const width = window.innerWidth;
  const connection = navigator.connection;

  // Check network speed if available
  const isSlow = connection && connection.effectiveType === '2g' || connection?.effectiveType === '3g';

  if (isSlow || width < 600) {
    return 'video-480p.mp4';    // ~1MB/min
  } else if (width < 1200) {
    return 'video-720p.mp4';    // ~3MB/min
  } else {
    return 'video-1080p.mp4';   // ~6MB/min
  }
}

const video = document.querySelector('video');
video.src = getVideoSource();
```

**Benefits:**
- Mobile users save 60-80% bandwidth vs receiving 1080p
- Faster playback start on slow networks
- Reduced CPU/GPU decoding cost
- Better battery life on mobile devices

[⬆ Back to Top](#top)

---

## Video Preloading Strategy

**Concept:**
The `preload` attribute controls how much video data the browser downloads before the user interacts with it.

**Three preload options:**

| Value | Behavior | Downloaded Data | Use Case |
|-------|----------|----------------|----------|
| `none` | Downloads nothing | 0 bytes | Multiple videos on page |
| `metadata` | Downloads dimensions, duration, first frame | ~3-5% of file | Default recommendation |
| `auto` | Downloads entire video | 100% of file | Single critical video |

**Example:**

```html
<!-- Best default: only metadata -->
<video preload="metadata" controls poster="thumb.jpg">
  <source src="video.mp4" type="video/mp4">
</video>

<!-- Page with many videos: load nothing -->
<video preload="none" controls poster="thumb.jpg">
  <source src="video.mp4" type="video/mp4">
</video>

<!-- Single hero video that must play immediately -->
<video preload="auto" autoplay muted playsinline>
  <source src="hero.mp4" type="video/mp4">
</video>
```

**Preload based on user intent:**

```javascript
// Preload video only when user hovers over thumbnail
const thumbnail = document.querySelector('.video-thumbnail');
const video = document.querySelector('video');

video.preload = 'none';

thumbnail.addEventListener('mouseenter', () => {
  video.preload = 'auto';  // Start downloading
}, { once: true });

thumbnail.addEventListener('click', () => {
  video.play();
});
```

**Best practices:**
- Default to `preload="metadata"` for most videos
- Use `preload="none"` when page has multiple videos
- Use `preload="auto"` only for auto-playing hero videos
- Combine with poster images for best UX

[⬆ Back to Top](#top)

---

## Lazy Loading Video with Intersection Observer

**Concept:**
Just like images, videos below the fold should not load until the user scrolls near them. This is especially important because videos are the **heaviest assets** on a page — even `preload="metadata"` downloads several hundred KB.

**The problem without lazy loading:**
- A page with 5 videos downloads metadata for all 5 immediately
- Background videos in carousels waste bandwidth even if never viewed
- Mobile users on data plans pay for video data they never watch

**Intersection Observer approach:**

```html
<!-- Video with no src initially — data attributes hold the real sources -->
<video
  class="lazy-video"
  poster="thumbnail.jpg"
  controls
  preload="none"
  width="720"
  height="405"
  data-src="video.webm"
  data-src-fallback="video.mp4"
>
</video>
```

```javascript
// Load video sources only when the user scrolls near them
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const video = entry.target;

      // Create source elements and add them to the video
      const webm = document.createElement('source');
      webm.src = video.dataset.src;
      webm.type = 'video/webm';

      const mp4 = document.createElement('source');
      mp4.src = video.dataset.srcFallback;
      mp4.type = 'video/mp4';

      video.appendChild(webm);
      video.appendChild(mp4);
      video.load();  // Tell browser to start loading

      observer.unobserve(video);  // Stop observing once loaded
    }
  });
}, {
  rootMargin: '200px'  // Start loading 200px before the video enters viewport
});

// Observe all lazy videos
document.querySelectorAll('.lazy-video').forEach(video => {
  observer.observe(video);
});
```

**Auto-play background videos on scroll:**

```javascript
// Play/pause background videos based on visibility
const bgObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    const video = entry.target;
    if (entry.isIntersecting) {
      video.play().catch(() => {}); // Play when visible
    } else {
      video.pause(); // Pause when not visible (saves CPU + bandwidth)
    }
  });
}, { threshold: 0.5 });  // 50% visible before playing

document.querySelectorAll('.bg-video').forEach(v => bgObserver.observe(v));
```

**Benefits:**
- Reduces initial page weight by hundreds of KB to MBs
- Improves LCP by freeing bandwidth for critical resources
- Saves mobile data costs for users
- Pausing off-screen videos reduces CPU/GPU usage and extends battery life

[⬆ Back to Top](#top)

---

## Video Accessibility

**Concept:**
Video content must be accessible to users who are deaf/hard of hearing, blind/low vision, or unable to use a mouse. Accessibility is also a legal requirement under ADA, EAA, and Section 508.

### Captions and Subtitles (`<track>` element)

The `<track>` element adds text tracks (captions, subtitles, descriptions) to HTML5 video:

```html
<video controls width="720" height="405">
  <source src="video.webm" type="video/webm">
  <source src="video.mp4" type="video/mp4">

  <!-- Captions (same language) — includes sound effects, speaker identification -->
  <track
    kind="captions"
    src="captions-en.vtt"
    srclang="en"
    label="English"
    default
  >

  <!-- Subtitles (different language) — translation of dialogue only -->
  <track
    kind="subtitles"
    src="subtitles-es.vtt"
    srclang="es"
    label="Español"
  >

  <!-- Audio descriptions — for visually impaired users -->
  <track
    kind="descriptions"
    src="descriptions-en.vtt"
    srclang="en"
    label="Audio Descriptions"
  >
</video>
```

**WebVTT caption format:**

```
WEBVTT

00:00:01.000 --> 00:00:04.000
Welcome to our product demo.

00:00:05.000 --> 00:00:08.000
[upbeat music playing]

00:00:09.000 --> 00:00:12.000
<v Speaker 1>Let me show you the new features.</v>
```

**`kind` values:**

| Kind | Purpose | Who It's For |
|------|---------|-------------|
| `captions` | Dialogue + sound effects + speaker ID | Deaf/hard of hearing |
| `subtitles` | Translated dialogue | Non-native speakers |
| `descriptions` | Narration of visual content | Blind/low vision |
| `chapters` | Navigation markers | All users (video scrubbing) |
| `metadata` | Machine-readable data | Script/application use |

### Keyboard Controls

Native `<video controls>` provides keyboard support automatically:

| Key | Action |
|-----|--------|
| `Space` / `Enter` | Play / Pause |
| `←` / `→` | Seek backward / forward |
| `↑` / `↓` | Volume up / down |
| `M` | Mute / Unmute |
| `F` | Fullscreen toggle |
| `C` | Toggle captions (in some browsers) |

> **Important:** If building a custom video player, you must implement all keyboard controls manually. Never remove `controls` without providing an accessible custom alternative.

### ARIA Labels and Descriptions

```html
<!-- Provide context for screen readers -->
<video
  controls
  aria-label="Product demo video showing new dashboard features"
  aria-describedby="video-description"
>
  <source src="demo.mp4" type="video/mp4">
</video>
<p id="video-description" class="sr-only">
  A 3-minute walkthrough of the new analytics dashboard,
  demonstrating real-time charts, filtering, and export features.
</p>
```

### Accessibility Checklist for Video

- ✅ Provide **captions** for all video with dialogue or meaningful audio
- ✅ Provide **audio descriptions** for video with important visual content not described in dialogue
- ✅ Use the native `controls` attribute (or build a fully keyboard-accessible custom player)
- ✅ Add `aria-label` describing the video content
- ✅ Ensure sufficient **contrast** for custom player controls
- ✅ Never autoplay video **with sound** (violates WCAG 1.4.2)
- ✅ Provide a **transcript** as a text alternative for users who can't watch video at all

[⬆ Back to Top](#top)

---

## Video Compression Comparison

**Concept:**
Different codecs achieve dramatically different file sizes at equivalent visual quality. Choosing the right codec directly impacts bandwidth, load time, and mobile data costs.

**Comparison at equivalent visual quality (30s clip, 1080p, 30fps):**

| Codec | Container | File Size | Savings vs H.264 | Encoding Speed | Browser Support |
|-------|-----------|-----------|-------------------|----------------|----------------|
| **H.264 (AVC)** | MP4 | ~15 MB (baseline) | — | Fast | Universal |
| **H.265 (HEVC)** | MP4 | ~9 MB | ~40% smaller | Medium | Safari, some browsers |
| **VP9** | WebM | ~9-10 MB | ~35% smaller | Slow | Chrome, Firefox, Edge |
| **AV1** | MP4/WebM | ~7 MB | ~50% smaller | Very slow | Chrome, Firefox, Edge |

**The encoding speed vs file size trade-off:**

```
Encoding Speed:  H.264 >>>> H.265 >> VP9 >> AV1
File Size:       H.264 <<<< H.265  < VP9  < AV1 (smallest)
Quality/size:    AV1 > VP9 ≈ H.265 > H.264
```

**Practical encoding recommendations:**

| Use Case | Primary Codec | Fallback | Why |
|----------|---------------|----------|-----|
| Short clips (< 2 min) | AV1 | WebM (VP9) → MP4 (H.264) | Encoding time acceptable, maximum savings |
| Long-form video (> 10 min) | VP9 or H.265 | MP4 (H.264) | AV1 encoding too slow for long content |
| Live streaming | H.264 | — | Real-time encoding requires speed |
| User-generated content | H.264 | — | Fast server-side transcoding needed |
| Archive / reusable assets | AV1 | — | Encode once, serve forever — slow encoding is fine |

**Quality at different file sizes (visual comparison):**

```
Target: 2 MB file for a 30s 720p clip

H.264 at 2MB:  Visible compression artifacts, blocky motion
VP9 at 2MB:    Good quality, minor artifacts in fast motion
AV1 at 2MB:    Near-transparent quality, smooth motion

→ Same file size, dramatically different quality
→ OR: Same quality, dramatically different file size
```

> **Best practice for most sites:** Encode in **AV1 > WebP/VP9 > H.264** using `<source>` tags with progressive enhancement. The browser automatically selects the best format it supports. Pre-encode at build time (encoding speed doesn't matter for static assets).

[⬆ Back to Top](#top)

---

## Key Takeaways

- Provide multiple formats (AV1 > WebM > MP4) for progressive enhancement
- Replace animated GIFs with muted auto-playing video (10x smaller)
- Use poster images for instant visual context before playback
- Remove audio track from silent videos at encoding level (10-30% savings)
- Use adaptive streaming (HLS/DASH) for long-form video content
- Serve platform-appropriate resolutions based on device and network
- Default to `preload="metadata"` to balance UX and bandwidth

### Performance Metrics Impact

| Metric | Impact |
|--------|--------|
| LCP (Largest Contentful Paint) | ++ Moderate – video posters and preloading affect LCP |
| CLS (Cumulative Layout Shift) | + Minor – proper dimensions and posters prevent shifts |
| TTFB (Time to First Byte) | + Minor – streaming reduces perceived wait time |

[⬆ Back to Top](#top)

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design

[⬆ Back to Top](#top)
