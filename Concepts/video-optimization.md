# Video Optimization – Frontend Performance

Video content is the heaviest asset type on the web. A single unoptimized video can be larger than all other page assets combined. Optimization directly impacts load time, bandwidth costs, and mobile data usage.

## Table of Contents

- [1. Progressive Enhancement for Video](#1-progressive-enhancement-for-video)
- [2. Video Formats (WebM, MP4)](#2-video-formats-webm-mp4)
- [3. Progressive Poster Images](#3-progressive-poster-images)
- [4. Remove Audio from Videos](#4-remove-audio-from-videos)
- [5. Streaming Algorithms and Techniques](#5-streaming-algorithms-and-techniques)
- [6. Platform-Based Video Dimensions](#6-platform-based-video-dimensions)
- [7. Video Preloading Strategy](#7-video-preloading-strategy)
- [Key Takeaways](#key-takeaways)

---

## 1. Progressive Enhancement for Video

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

---

## 2. Video Formats (WebM, MP4)

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

---

## 3. Progressive Poster Images

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

---

## 4. Remove Audio from Videos

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

---

## 5. Streaming Algorithms and Techniques

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

---

## 6. Platform-Based Video Dimensions

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

---

## 7. Video Preloading Strategy

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
