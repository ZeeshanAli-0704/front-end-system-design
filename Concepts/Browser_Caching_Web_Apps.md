# Browser Caching for Web Apps

> *"When your app feels snappy, users engage more and your infrastructure costs less. Browser caching is one of the simplest, highest-impact optimizations you can make."*

The trick is balancing speed with freshness: you want returning users to load instantly, but you also need them to get new code right after you deploy. This guide explains a proven, low-maintenance caching strategy you can adopt for any modern web app.

---

<a id="top"></a>

## Table of Contents

- [What Browser Caching Is](#what-browser-caching-is)
- [Where Caching Happens](#where-caching-happens)
- [Why Caching Matters](#why-caching-matters)
- [The Main Challenge Staying Fresh After a Deployment](#the-main-challenge-staying-fresh-after-a-deployment)
- [The Golden Pattern](#the-golden-pattern)
- [How to Implement It](#how-to-implement-it)
- [When to Use Caching](#when-to-use-caching)
- [What No Cache Actually Means](#what-no-cache-actually-means)
- [Example User Flow After a Deploy](#example-user-flow-after-a-deploy)
- [Configuration Examples](#configuration-examples)
- [Framework Tips](#framework-tips)
- [CDN Best Practices](#cdn-best-practices)
- [APIs and Data Freshness](#apis-and-data-freshness)
- [Service Workers (Optional Advanced)](#service-workers-optional-advanced)
- [How to Force Revalidation](#how-to-force-revalidation)
- [How to Verify Your Setup](#how-to-verify-your-setup)
- [Common Pitfalls](#common-pitfalls)
- [Security and Privacy Notes](#security-and-privacy-notes)
- [Deployment Checklist](#deployment-checklist)
- [FAQ](#faq)

[⬆ Back to Top](#top)

---

## What Browser Caching Is

A browser stores copies of files it downloads (HTML, JavaScript, CSS, images, fonts) on the user's device. On the next visit, it can reuse those files instead of downloading them again. The result is faster pages, lower bandwidth, fewer servers needed, and a better user experience.

[⬆ Back to Top](#top)

---

## Where Caching Happens

| Layer | Description |
| --- | --- |
| **Browser cache** | Local to the user's device (fastest path for repeat visits) |
| **CDN or proxy cache** | Copies at edge servers closer to users (reduces origin load and latency) |
| **Service worker cache** | App-controlled caching logic for offline and advanced update strategies (optional) |

[⬆ Back to Top](#top)

---

## Why Caching Matters

| Benefit | Impact |
| --- | --- |
| **Performance** | Returning visitors see dramatic speed gains |
| **Cost** | Fewer bytes leave your origin and APIs |
| **Reliability** | Less load on your servers during traffic spikes and incidents |

[⬆ Back to Top](#top)

---

## The Main Challenge Staying Fresh After a Deployment

If you cache aggressively, users can get stuck on old files. The goal is to ensure that when you deploy a new build, users automatically receive the correct new files without you purging all caches or asking users to hard refresh.

[⬆ Back to Top](#top)

---

## The Golden Pattern

A simple and reliable approach:

| Resource | Strategy |
| --- | --- |
| **HTML** | Revalidates on every navigation |
| **Static assets** (JS, CSS, images) | Long cache lifetimes, but filenames change when content changes |
| **APIs** | Revalidate frequently using `ETag` or `Last-Modified` |

[⬆ Back to Top](#top)

---

## How to Implement It

### 1. Content Hashed Filenames for Static Assets

Use build tools that produce filenames based on the content:

```
main.4f3c1.js
styles.a9b2.css
logo.8d12.png
```

The hash changes only when the file's content changes. After a new build, browsers fetch only the changed files (new names) and reuse unchanged ones.

### 2. Cache Control Headers

| Resource | Header | Why |
| --- | --- | --- |
| **HTML** | `Cache-Control: no-cache, must-revalidate` (or `max-age=0, must-revalidate`) | Tells the browser and CDN to check with the server before using a cached copy. With `ETag` or `Last-Modified`, this check is cheap and fast |
| **Hashed assets** (JS/CSS/images/fonts) | `Cache-Control: public, max-age=31536000, immutable` | Long-lived because the URL is unique to the content |
| **APIs** | `Cache-Control: no-cache` (or short `max-age` with `must-revalidate`) + `ETag` or `Last-Modified` | Clients receive quick `304 Not Modified` responses when data hasn't changed |

### 3. Deployment Order

1. Upload new hashed assets first
2. Then publish the updated HTML that references those new asset filenames
3. Optionally invalidate CDN cache for HTML routes so the updated entry point propagates quickly

[⬆ Back to Top](#top)

---

## When to Use Caching

| Scenario | Recommendation |
| --- | --- |
| **Production** | Always. It's a foundational performance practice |
| **Development** | Keep caching minimal to avoid confusion (e.g., disable cache in DevTools or use short `max-age`) |
| **Private or sensitive content** | Use stricter headers such as `Cache-Control: no-store` for confidential pages or data |

[⬆ Back to Top](#top)

---

## What No Cache Actually Means

`no-cache` does **not** mean "never store." It means **"revalidate before using the cached copy."**

With `ETag` or `Last-Modified`, revalidation usually returns a small `304 Not Modified` and the browser uses its local copy, which is fast and efficient.

[⬆ Back to Top](#top)

---

## Example User Flow After a Deploy

You deploy a build:

| File | Change | New Filename |
| --- | --- | --- |
| `main.js` | Changed | `main.newhash.js` |
| `styles.css` | Unchanged | `styles.samehash.css` |
| `index.html` | Updated references | References new filenames |

What happens when a user visits:

1. Browser revalidates `index.html` and gets the updated HTML
2. It downloads `main.newhash.js` (new URL)
3. It reuses cached `styles.samehash.css` (same URL)
4. **Result**: Only changed files are fetched; unchanged files load instantly

[⬆ Back to Top](#top)

---

## Configuration Examples

### Nginx

```nginx
location = /index.html {
    add_header Cache-Control "no-cache, must-revalidate";
}

location ~* \.(js|css|png|jpg|jpeg|gif|svg|woff2)$ {
    add_header Cache-Control "public, max-age=31536000, immutable";
}
```

### Apache (.htaccess)

```apache
<Files "index.html">
    Header set Cache-Control "no-cache, must-revalidate"
</Files>

<FilesMatch "\.(js|css|png|jpg|jpeg|gif|svg|woff2)$">
    Header set Cache-Control "public, max-age=31536000, immutable"
</FilesMatch>
```

### Node / Express

```js
app.use(express.static("dist", {
  setHeaders: (res, path) => {
    if (path.endsWith(".html")) {
      res.setHeader("Cache-Control", "no-cache, must-revalidate");
    } else {
      res.setHeader("Cache-Control", "public, max-age=31536000, immutable");
    }
  }
}));
```

[⬆ Back to Top](#top)

---

## Framework Tips

| Framework | Tip |
| --- | --- |
| **React (Vite/CRA), Angular, Vue CLI** | Production builds typically create content-hashed filenames automatically. Verify in your `dist/build` output that JS/CSS include hashes. Serve these assets with long-lived immutable caching |
| **Next.js, Nuxt (SSR)** | Let the framework manage asset hashing. Ensure HTML responses (SSR) have revalidation headers or a short CDN TTL with `must-revalidate`. Dynamic pages should emit `ETag` or `Last-Modified` if feasible |
| **Single Page Apps** | Always revalidate `index.html`. For deep links, serve `index.html` for app routes with the same headers |

[⬆ Back to Top](#top)

---

## CDN Best Practices

- Let your origin send the headers above; most CDNs respect them
- **Invalidate or purge HTML routes** after deployment so the updated entry point becomes visible quickly
- With hashed assets, you **rarely need to purge** JS/CSS because new builds use new filenames
- Consider a **short CDN TTL for HTML** (for example, 60-300 seconds) as a safety net if purges are missed
- **Keep older hashed assets** on the CDN for a while (and at origin) to avoid 404s for users who still reference previous build assets and to support rollbacks

[⬆ Back to Top](#top)

---

## APIs and Data Freshness

- Use `ETag` or `Last-Modified` with `Cache-Control: no-cache` or a short `max-age` and `must-revalidate`. This avoids stale data and keeps bandwidth low via `304`s
- For highly dynamic or sensitive responses that must never be reused, use `Cache-Control: no-store`

[⬆ Back to Top](#top)

---

## Service Workers (Optional Advanced)

- Service workers allow you to script caching and offline behavior
- **Version your caches** (for example, `app-cache-v42`) and precache assets on install for predictable offline behavior
- On each deploy, publish a new service worker. Decide your update UX:
  - **Prompt users** to refresh when an update is available (good control and clarity)
  - Or **auto-activate** with `self.skipWaiting()` and `clients.claim()` (faster, but consider UX tradeoffs)
- Do not let the service worker serve stale HTML forever. Use **network-first** or **stale-while-revalidate** for HTML so updates are discovered promptly

[⬆ Back to Top](#top)

---

## How to Force Revalidation

**For yourself:**
- Hard refresh (`Ctrl/Cmd+Shift+R`) or enable "Disable cache" in DevTools

**For all users:**
- Keep HTML as `no-cache, must-revalidate` and return `ETag` or `Last-Modified`
- Invalidate CDN cache for HTML routes immediately after deploy
- In emergencies, temporarily set `Cache-Control: no-store` on HTML to force a refresh, then revert

[⬆ Back to Top](#top)

---

## How to Verify Your Setup

**Browser DevTools (Network tab):**
- `index.html` should show `200` or `304` after reload (not "from cache"), indicating revalidation
- Hashed JS/CSS should typically show "from disk cache" or "from memory cache" between deployments

**curl checks:**

```bash
# Check headers
curl -I https://your.site/index.html

# Note the ETag, then test revalidation
curl -H "If-None-Match: <etag>" -I https://your.site/index.html
# Expect 304 if unchanged
```

[⬆ Back to Top](#top)

---

## Common Pitfalls

| Pitfall | Why It's a Problem |
| --- | --- |
| **Skipping content hashing** | Leads to stale files and complex purges. Always use hashed filenames |
| **Query-string cache busting** (`file.js?v=123`) | Some caches ignore query params. Prefer hashed filenames |
| **Long-lived caching for HTML** | Users won't see new builds. Keep HTML revalidated |
| **Removing old assets immediately** | Users with older HTML may still request old hashed files. Keep previous builds' assets available for a safe window |
| **Service worker traps** | A service worker that serves stale HTML indefinitely breaks updates. Ensure HTML revalidates and have a clear update strategy |

[⬆ Back to Top](#top)

---

## Security and Privacy Notes

- **Do not cache sensitive or private data.** Use `Cache-Control: no-store` on such responses
- If you adopt third-party CDNs, plugins, or service worker libraries, ensure they meet your organization's security and compliance requirements

[⬆ Back to Top](#top)

---

## Deployment Checklist

- [ ] Build outputs include **content-hashed filenames** for all static assets
- [ ] HTML responses include `Cache-Control: no-cache, must-revalidate` and `ETag` or `Last-Modified`
- [ ] Static assets use `Cache-Control: public, max-age=31536000, immutable`
- [ ] Upload new hashed assets first, then publish the updated HTML
- [ ] Invalidate CDN cache for HTML routes after deployment (recommended)
- [ ] Keep the last few builds' assets available for rollbacks and late-returning users
- [ ] If using a service worker, bump the version and implement an update prompt or auto-activation strategy

[⬆ Back to Top](#top)

---

## FAQ

| Question | Answer |
| --- | --- |
| **Will users always get the latest build?** | Yes. HTML revalidates and points to new hashed assets; changed assets download, unchanged assets are reused from cache |
| **What if only JS changed?** | The HTML changes the script `src` to the new hash and revalidation picks it up automatically |
| **Do I need users to hard refresh?** | Not with hashing plus proper headers |
| **Is no-cache slow?** | No. With `ETag` or `Last-Modified`, the browser typically gets a quick `304` and reuses its local copy |
| **Can I skip CDN purges?** | Usually yes for assets due to hashing. Purge HTML for immediate propagation |

[⬆ Back to Top](#top)

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design

[⬆ Back to Top](#top)