# Logging, Analytics & Feature Flags — A Complete Frontend Guide

> *"If you can't observe it, you can't improve it. Logging, analytics, and feature flags form the operational backbone of every production frontend — they tell you what users do, when things break, and how to ship safely."*

This guide covers **analytics architecture** (event tracking, funnel tracking), **A/B testing infrastructure**, **feature flag systems** (LaunchDarkly, Unleash), **session replay & heatmaps**, **frontend error tracking & debugging**, and **where / how to store and ship logs** at scale.

---

<a id="top"></a>

## Table of Contents

- [Analytics Architecture](#analytics-architecture)
- [A B Testing Infrastructure](#a-b-testing-infrastructure)
- [Feature Flag Systems](#feature-flag-systems)
- [Session Replay and Heatmaps](#session-replay-and-heatmaps)
- [Frontend Error Tracking and Debugging](#frontend-error-tracking-and-debugging)
- [Frontend Logging Where to Hold Logs and How](#frontend-logging-where-to-hold-logs-and-how)
- [Putting It All Together Unified Observability](#putting-it-all-together-unified-observability)
- [Decision Matrix and Quick Reference](#decision-matrix-and-quick-reference)
- [Key Interview Takeaways](#key-interview-takeaways)
- [Further Reading and Resources](#further-reading-and-resources)


[⬆ Back to Top](#top)

---

## Analytics Architecture

### 1.1 Why Frontend Analytics Matter

Analytics answer **three critical questions** in system design:

| Question | What It Drives |
|---|---|
| **What are users doing?** | Product decisions, UX improvements |
| **Where are users dropping off?** | Funnel optimization, revenue impact |
| **Is the new feature working?** | A/B test evaluation, rollout decisions |

Without analytics, every product decision is a guess. In a frontend system design interview, analytics is the **feedback loop** that validates your design choices.

---

### 1.2 Event Tracking — Design & Implementation

#### Event Taxonomy

A well-designed event taxonomy is the foundation of good analytics. Every event should answer: **Who** did **what**, **where**, and **when**?

```
┌──────────────────────────────────────────────────────┐
│                Event Schema                           │
│                                                       │
│  {                                                    │
│    "event":      "product_added_to_cart",             │
│    "timestamp":  "2026-03-13T10:30:00.000Z",          │
│    "userId":     "u_abc123",                          │
│    "sessionId":  "s_xyz789",                          │
│    "properties": {                                    │
│      "productId":    "SKU-1234",                      │
│      "productName":  "Wireless Headphones",           │
│      "price":        79.99,                           │
│      "currency":     "USD",                           │
│      "category":     "Electronics",                   │
│      "quantity":     1                                │
│    },                                                 │
│    "context": {                                       │
│      "page":         "/product/SKU-1234",             │
│      "referrer":     "/search?q=headphones",          │
│      "device":       "mobile",                        │
│      "browser":      "Chrome 120",                    │
│      "os":           "Android 14",                    │
│      "viewport":     "390x844",                       │
│      "network":      "4g",                            │
│      "locale":       "en-US"                          │
│    }                                                  │
│  }                                                    │
└──────────────────────────────────────────────────────┘
```

#### Naming Convention

Use a consistent **object_action** pattern:

```
✅ Good (consistent, greppable):
  page_viewed
  product_added_to_cart
  checkout_started
  payment_completed
  search_performed
  filter_applied

❌ Bad (inconsistent, hard to query):
  viewPage
  AddToCart
  user clicked checkout
  pay_done
```

#### Analytics Service Implementation

```ts
// src/analytics/analyticsService.ts

interface EventProperties {
  [key: string]: string | number | boolean | null;
}

interface EventContext {
  page: string;
  referrer: string;
  device: 'mobile' | 'tablet' | 'desktop';
  sessionId: string;
  userId?: string;
}

interface AnalyticsEvent {
  event: string;
  timestamp: string;
  properties: EventProperties;
  context: EventContext;
}

class AnalyticsService {
  private queue: AnalyticsEvent[] = [];
  private flushInterval: number;
  private maxQueueSize: number;
  private endpoint: string;

  constructor(config: {
    endpoint: string;
    flushIntervalMs?: number;
    maxQueueSize?: number;
  }) {
    this.endpoint = config.endpoint;
    this.flushInterval = config.flushIntervalMs ?? 5000;  // Flush every 5s
    this.maxQueueSize = config.maxQueueSize ?? 20;

    // Periodic flush
    setInterval(() => this.flush(), this.flushInterval);

    // Flush on page unload
    window.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'hidden') {
        this.flush();
      }
    });
  }

  track(event: string, properties: EventProperties = {}): void {
    const analyticsEvent: AnalyticsEvent = {
      event,
      timestamp: new Date().toISOString(),
      properties,
      context: this.getContext(),
    };

    this.queue.push(analyticsEvent);

    // Auto-flush if queue is full
    if (this.queue.length >= this.maxQueueSize) {
      this.flush();
    }
  }

  private async flush(): Promise<void> {
    if (this.queue.length === 0) return;

    const batch = [...this.queue];
    this.queue = [];

    try {
      // Use sendBeacon for reliability (survives page unload)
      const blob = new Blob([JSON.stringify(batch)], { type: 'application/json' });
      const sent = navigator.sendBeacon(this.endpoint, blob);

      if (!sent) {
        // Fallback to fetch with keepalive
        await fetch(this.endpoint, {
          method: 'POST',
          body: JSON.stringify(batch),
          headers: { 'Content-Type': 'application/json' },
          keepalive: true,
        });
      }
    } catch (error) {
      // Put events back in queue (with limit to prevent memory leak)
      this.queue = [...batch.slice(-this.maxQueueSize), ...this.queue];
      console.warn('Analytics flush failed, events re-queued', error);
    }
  }

  private getContext(): EventContext {
    return {
      page: window.location.pathname,
      referrer: document.referrer,
      device: this.getDeviceType(),
      sessionId: this.getSessionId(),
      userId: this.getUserId(),
    };
  }

  private getDeviceType(): 'mobile' | 'tablet' | 'desktop' {
    const w = window.innerWidth;
    if (w < 768) return 'mobile';
    if (w < 1024) return 'tablet';
    return 'desktop';
  }

  private getSessionId(): string {
    let sid = sessionStorage.getItem('analytics_sid');
    if (!sid) {
      sid = crypto.randomUUID();
      sessionStorage.setItem('analytics_sid', sid);
    }
    return sid;
  }

  private getUserId(): string | undefined {
    // Pull from your auth layer
    return (window as any).__AUTH_USER__?.id;
  }
}

// Singleton
export const analytics = new AnalyticsService({
  endpoint: '/api/analytics/events',
  flushIntervalMs: 5000,
  maxQueueSize: 20,
});
```

#### React Integration — Custom Hook

```tsx
// src/analytics/useTrack.ts
import { useEffect, useCallback } from 'react';
import { analytics } from './analyticsService';

// Automatic page-view tracking
export function usePageView(pageName: string) {
  useEffect(() => {
    analytics.track('page_viewed', { page_name: pageName });
  }, [pageName]);
}

// Manual event tracking
export function useTrack() {
  return useCallback(
    (event: string, properties?: Record<string, any>) => {
      analytics.track(event, properties ?? {});
    },
    []
  );
}

// Usage in a component
function ProductPage({ product }) {
  usePageView('product_detail');
  const track = useTrack();

  const handleAddToCart = () => {
    track('product_added_to_cart', {
      productId: product.id,
      price: product.price,
    });
    addToCart(product);
  };

  return <button onClick={handleAddToCart}>Add to Cart</button>;
}
```

#### Declarative Tracking with Data Attributes

For large apps, **declarative tracking** reduces boilerplate:

```tsx
// Attach tracking metadata to any element
<button
  data-track="product_added_to_cart"
  data-track-product-id={product.id}
  data-track-price={product.price}
  onClick={addToCart}
>
  Add to Cart
</button>

// Global click listener (set up once)
document.addEventListener('click', (e) => {
  const target = (e.target as HTMLElement).closest('[data-track]');
  if (!target) return;

  const event = target.getAttribute('data-track')!;
  const properties: Record<string, string> = {};

  // Collect all data-track-* attributes
  for (const attr of target.attributes) {
    if (attr.name.startsWith('data-track-') && attr.name !== 'data-track') {
      const key = attr.name.replace('data-track-', '').replace(/-/g, '_');
      properties[key] = attr.value;
    }
  }

  analytics.track(event, properties);
});
```

---

### 1.3 Funnel Tracking

A **funnel** tracks a sequence of user steps toward a goal (e.g., purchase). Drop-off at any step represents lost revenue or engagement.

#### E-Commerce Funnel Example

```
Step 1: page_viewed        (page: /products)         100% ████████████████████
Step 2: product_viewed     (productId: SKU-123)       60% ████████████
Step 3: product_added_to_cart                          25% █████
Step 4: checkout_started                               15% ███
Step 5: payment_info_entered                           12% ██▌
Step 6: order_completed                                 8% █▋

Drop-off analysis:
  Step 1→2: 40% lose interest (improve recommendations)
  Step 2→3: 58% don't add to cart (improve product page, pricing)
  Step 3→4: 40% abandon cart (add urgency, simplify UX)
  Step 4→5: 20% hesitate at payment (add trust signals, payment options)
  Step 5→6: 33% fail to complete (optimize payment flow, error handling)
```

#### Funnel Tracking Implementation

```ts
// src/analytics/funnel.ts

interface FunnelStep {
  name: string;
  event: string;
  timestamp?: number;
}

class FunnelTracker {
  private funnels: Map<string, FunnelStep[]> = new Map();

  /**
   * Define a funnel with named steps
   */
  defineFunnel(funnelName: string, steps: string[]) {
    this.funnels.set(
      funnelName,
      steps.map((event) => ({ name: event, event }))
    );
  }

  /**
   * Record a funnel step completion
   */
  recordStep(funnelName: string, eventName: string, properties: Record<string, any> = {}) {
    const funnel = this.funnels.get(funnelName);
    if (!funnel) return;

    const stepIndex = funnel.findIndex((s) => s.event === eventName);
    if (stepIndex === -1) return;

    analytics.track('funnel_step_completed', {
      funnel_name: funnelName,
      step_name: eventName,
      step_index: stepIndex,
      total_steps: funnel.length,
      ...properties,
    });
  }
}

// Usage
const funnelTracker = new FunnelTracker();

funnelTracker.defineFunnel('checkout', [
  'cart_viewed',
  'checkout_started',
  'shipping_entered',
  'payment_entered',
  'order_completed',
]);

// In your component:
funnelTracker.recordStep('checkout', 'checkout_started', { cartTotal: 149.99 });
```

---

### 1.4 Analytics Pipeline Architecture

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Browser     │    │   Ingestion   │    │   Stream      │    │   Storage     │
│               │    │   Layer       │    │   Processing  │    │   & Query     │
│ analytics.js  │───>│              │───>│              │───>│              │
│               │    │  API Gateway  │    │  Kafka /     │    │  ClickHouse  │
│ sendBeacon()  │    │  or Collector │    │  Kinesis     │    │  BigQuery    │
│ fetch()       │    │  Endpoint     │    │              │    │  Snowflake   │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                                              │
                                              ▼
                                     ┌──────────────┐    ┌──────────────┐
                                     │  Real-time    │    │  Dashboard    │
                                     │  Aggregation  │───>│  & Alerts     │
                                     │  (Flink/      │    │  (Grafana,    │
                                     │   Spark)      │    │   Amplitude)  │
                                     └──────────────┘    └──────────────┘
```

**Key design decisions:**

| Decision | Recommendation | Why |
|---|---|---|
| **Transport** | `navigator.sendBeacon` + `fetch` fallback | Survives page unload |
| **Batching** | Batch events, flush every 5s or 20 events | Reduces network requests |
| **Schema** | Strict typed schema with validation | Prevents garbage data |
| **Sampling** | 100% for critical events, 10–50% for high-volume | Cost management |
| **Buffering** | Queue in memory; IndexedDB for offline | No data loss |
| **Backend** | Kafka/Kinesis → ClickHouse/BigQuery | Handles billions of events |

---

### 1.5 Popular Analytics Platforms Compared

| Platform | Type | Strengths | Weaknesses | Cost Model |
|---|---|---|---|---|
| **Google Analytics 4** | Full-stack | Free, deep Google integration | Complex setup, data sampling | Free / GA360 paid |
| **Amplitude** | Product analytics | Best funnel & cohort analysis | Expensive at scale | Event volume |
| **Mixpanel** | Product analytics | Great UX, real-time | Limited raw data access | Event volume |
| **Segment** | Customer Data Platform | Unified tracking, 300+ integrations | Expensive, adds latency | Monthly tracked users |
| **PostHog** | All-in-one (open-source) | Self-hostable, feature flags + analytics | Younger ecosystem | Events or self-host |
| **Plausible** | Privacy-first | No cookies, GDPR-compliant, simple | Limited features | Pageviews |
| **Custom (ClickHouse)** | Build-your-own | Full control, no vendor lock-in | Engineering cost | Infrastructure |

[⬆ Back to Top](#top)

---

## A B Testing Infrastructure

### 2.1 How A/B Tests Work on the Frontend

```
User visits page
      │
      ▼
┌─────────────────────┐
│ Experiment SDK       │
│ evaluates user into  │
│ variant              │
│                      │
│ hash(userId +        │
│   experimentId)      │
│   % bucketCount      │──── Deterministic: same user always
│                      │     gets same variant
│ → Variant A (50%)    │
│ → Variant B (50%)    │
└──────┬──────────────┘
       │
       ▼
┌──────────────────┐        ┌──────────────────┐
│ Variant A        │        │ Variant B        │
│ (Control)        │        │ (Treatment)      │
│                  │        │                  │
│ Blue "Buy" btn   │        │ Green "Buy" btn  │
│                  │        │                  │
│ Track: exposure  │        │ Track: exposure  │
│ Track: clicks    │        │ Track: clicks    │
│ Track: purchase  │        │ Track: purchase  │
└──────────────────┘        └──────────────────┘
       │                           │
       └─────────┬─────────────────┘
                 ▼
      ┌──────────────────┐
      │ Analytics         │
      │                   │
      │ Compare metrics   │
      │ per variant with  │
      │ statistical       │
      │ significance      │
      └──────────────────┘
```

### 2.2 Client-Side vs Server-Side A/B Testing

| Aspect | Client-Side | Server-Side |
|---|---|---|
| **Where it runs** | Browser (JS SDK) | Server / Edge / CDN |
| **Flicker risk** | Yes — content shifts after JS loads | No — correct variant served |
| **Performance** | Extra JS + SDK overhead | Zero client-side cost |
| **Personalization** | Limited (must wait for JS) | Full (server has all user data) |
| **Ease of setup** | Easy (drop-in SDK) | Requires backend integration |
| **SEO impact** | Can cause CLS issues | None |
| **Examples** | Google Optimize, Optimizely (client) | LaunchDarkly, Unleash, Statsig |

**Modern Trend:** Server-side / Edge-side evaluation with a lightweight client SDK that receives the decision — no flicker, no performance cost.

---

### 2.3 Implementation Patterns

#### Pattern 1: Feature-Flag-Driven A/B Tests

The cleanest approach — A/B tests are just **feature flags with analytics**:

```tsx
// Uses the same feature flag system (Section 3)
import { useFeatureFlag } from '@org/feature-flags';
import { analytics } from '@org/analytics';
import { useEffect } from 'react';

function CheckoutButton() {
  const variant = useFeatureFlag('checkout_button_experiment');
  // Returns: 'control' | 'green_button' | 'urgency_copy'

  // Track exposure (user saw the experiment)
  useEffect(() => {
    analytics.track('experiment_exposure', {
      experiment: 'checkout_button_experiment',
      variant,
    });
  }, [variant]);

  // Track conversion
  const handleClick = () => {
    analytics.track('experiment_conversion', {
      experiment: 'checkout_button_experiment',
      variant,
      action: 'checkout_clicked',
    });
    proceedToCheckout();
  };

  switch (variant) {
    case 'green_button':
      return <button className="btn-green" onClick={handleClick}>Buy Now</button>;
    case 'urgency_copy':
      return <button className="btn-blue" onClick={handleClick}>Buy Now — Only 3 Left!</button>;
    default:
      return <button className="btn-blue" onClick={handleClick}>Buy Now</button>;
  }
}
```

#### Pattern 2: Component-Level Experiments

```tsx
// src/experiments/Experiment.tsx
interface ExperimentProps {
  name: string;
  children: Record<string, React.ReactNode>;  // variant → component
}

function Experiment({ name, children }: ExperimentProps) {
  const variant = useFeatureFlag(name);

  useEffect(() => {
    analytics.track('experiment_exposure', { experiment: name, variant });
  }, [name, variant]);

  return <>{children[variant] ?? children['control']}</>;
}

// Usage
<Experiment name="hero_redesign">
  {{
    control: <HeroV1 />,
    variant_a: <HeroV2WithVideo />,
    variant_b: <HeroV3Minimal />,
  }}
</Experiment>
```

---

### 2.4 Avoiding Flicker (FOOC)

**Flash of Original Content (FOOC):** User briefly sees the control variant before the experiment SDK assigns them to the treatment.

```
Timeline WITHOUT anti-flicker:
  0ms: HTML loads → shows control
  200ms: Experiment SDK loads
  300ms: SDK assigns variant B
  301ms: Page re-renders with variant B  ← USER SEES FLICKER ⚠️

Timeline WITH anti-flicker:
  0ms: HTML loads → body is hidden (opacity: 0)
  200ms: Experiment SDK loads
  300ms: SDK assigns variant, reveals body
  301ms: User sees correct variant from the start  ✅
```

**Anti-flicker snippet (in `<head>`):**

```html
<script>
  // Hide body until experiment SDK resolves (max 2 seconds)
  document.documentElement.style.opacity = '0';
  setTimeout(() => {
    document.documentElement.style.opacity = '1';
  }, 2000); // Safety timeout — never block rendering forever
</script>
```

> ⚠️ **Better approach:** Use **server-side** or **edge-side** evaluation so the correct variant is served from the start — zero flicker, zero CLS impact.

---

### 2.5 Statistical Considerations

| Concept | What It Means | Implication |
|---|---|---|
| **Sample size** | Enough users to detect a meaningful difference | Don't call experiments early |
| **Statistical significance** | p-value < 0.05 (95% confidence) | Avoid false positives |
| **Minimum Detectable Effect (MDE)** | Smallest improvement worth detecting | Drives sample size calculation |
| **Duration** | Run for at least 1–2 full business cycles (weeks) | Captures weekday/weekend variance |
| **Multiple comparisons** | Testing 5 variants inflates false positive rate | Use Bonferroni correction |
| **Novelty effect** | Users interact more with "new" things | Wait for effect to stabilize |

[⬆ Back to Top](#top)

---

## Feature Flag Systems

### 3.1 What Are Feature Flags?

Feature flags (also called feature toggles) are **runtime switches** that control which features are visible/enabled without deploying new code.

```
Traditional deploy:
  Code change → Build → Deploy → Feature is live for ALL users

With feature flags:
  Code change → Build → Deploy → Feature is OFF by default
                                       │
                                       ▼
                              Enable for 5% of users (canary)
                              Enable for internal team (dogfood)
                              Enable for beta users
                              Enable for 100% (full rollout)
                              Instantly disable if buggy (kill switch)
```

---

### 3.2 Types of Feature Flags

| Type | Lifetime | Purpose | Example |
|---|---|---|---|
| **Release flag** | Days–Weeks | Gate unfinished features | `new_checkout_flow` |
| **Experiment flag** | Weeks | A/B tests | `hero_redesign_experiment` |
| **Ops flag** | Permanent | Kill switches, load shedding | `disable_recommendations` |
| **Permission flag** | Permanent | Entitlement/plan gating | `premium_feature_export` |

---

### 3.3 Feature Flag Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                    Feature Flag System                             │
│                                                                    │
│  ┌──────────────┐    ┌──────────────────┐    ┌─────────────────┐  │
│  │  Admin UI /   │    │  Flag Evaluation  │    │  Client SDKs    │  │
│  │  Dashboard    │───>│  Service (API)    │───>│  (JS, React,    │  │
│  │              │    │                   │    │   Node, etc.)   │  │
│  │ Create flags  │    │ Rules engine:     │    │                 │  │
│  │ Set targeting │    │ - User segments  │    │ Evaluates flags  │  │
│  │ Define rules  │    │ - % rollout      │    │ locally or via   │  │
│  │ Kill switch   │    │ - Env (prod/stg) │    │ SSE stream       │  │
│  └──────────────┘    │ - Custom attrs   │    └─────────────────┘  │
│                       └──────────────────┘                         │
│                              │                                     │
│                              ▼                                     │
│                    ┌──────────────────┐                            │
│                    │  Flag Store       │                            │
│                    │  (Redis / DB)     │                            │
│                    │                   │                            │
│                    │  Caches evaluated │                            │
│                    │  flag values for  │                            │
│                    │  fast reads       │                            │
│                    └──────────────────┘                            │
└───────────────────────────────────────────────────────────────────┘
```

**Evaluation Flow:**

```
1. App starts → SDK fetches all flag definitions (bootstrap)
2. SDK caches flags in memory
3. SDK opens SSE / WebSocket connection for real-time updates
4. Component calls useFeatureFlag('new_checkout')
5. SDK evaluates locally:
   - Is user in target segment? (e.g., country === 'US')
   - Does user fall within rollout percentage?
   - Is the flag enabled for this environment?
6. Returns boolean / string variant
7. On flag change → SSE pushes update → SDK re-evaluates → component re-renders
```

---

### 3.4 LaunchDarkly — Deep Dive

LaunchDarkly is the market-leading feature flag platform. Here's how it works:

#### Architecture

```
┌───────────────┐     ┌──────────────────────┐     ┌────────────────┐
│  LaunchDarkly  │     │  Streaming edge       │     │  Your App       │
│  Dashboard     │────>│  (LD Relay / CDN)     │────>│  (LD SDK)       │
│                │     │                       │     │                 │
│  Flag config   │     │  SSE stream delivers  │     │  Local eval,    │
│  is pushed to  │     │  flag changes in      │     │  no latency     │
│  edge in ~200ms│     │  real-time             │     │  per flag check │
└───────────────┘     └──────────────────────┘     └────────────────┘
```

#### React SDK Integration

```tsx
// src/index.tsx
import { LDProvider } from 'launchdarkly-react-client-sdk';

const ldConfig = {
  clientSideID: 'YOUR_CLIENT_SIDE_ID',
  context: {
    kind: 'user',
    key: 'user-123',
    name: 'Jane Doe',
    email: 'jane@example.com',
    custom: {
      plan: 'premium',
      company: 'Acme Inc',
      country: 'US',
    },
  },
};

function Root() {
  return (
    <LDProvider {...ldConfig}>
      <App />
    </LDProvider>
  );
}
```

```tsx
// src/components/Feature.tsx
import { useFlags, useLDClient } from 'launchdarkly-react-client-sdk';

function DashboardPage() {
  const { newDashboardLayout, enableExport, darkModeExperiment } = useFlags();
  const ldClient = useLDClient();

  // Track custom event for experiments
  const handleExport = () => {
    ldClient?.track('export_clicked', { format: 'csv' });
    exportData();
  };

  return (
    <div className={darkModeExperiment === 'dark' ? 'dark-theme' : 'light-theme'}>
      {newDashboardLayout ? <DashboardV2 /> : <DashboardV1 />}
      {enableExport && <button onClick={handleExport}>Export</button>}
    </div>
  );
}
```

#### Targeting Rules Examples

```yaml
Flag: new_checkout_flow
  Default: false

  Rules (evaluated top-to-bottom):
    1. IF user.email ENDS WITH "@mycompany.com"  →  true   (internal dogfood)
    2. IF user.plan == "beta"                     →  true   (beta testers)
    3. IF user.country IN ["US", "CA"]            →  true (25% rollout)
    4. ELSE                                       →  false  (everyone else)
```

---

### 3.5 Unleash — Open-Source Alternative

**Unleash** is the leading **open-source** feature flag system. Self-hostable, API-compatible, and great for teams that want full control.

#### Setup

```bash
# Docker Compose for self-hosting
docker compose up -d  # Runs Unleash server + PostgreSQL

# Or use hosted: Unleash Cloud
```

#### React Integration

```tsx
// src/index.tsx
import { FlagProvider } from '@unleash/proxy-client-react';

const config = {
  url: 'https://your-unleash-proxy.com/api/frontend',
  clientKey: 'your-proxy-client-key',
  refreshInterval: 15,  // seconds
  appName: 'my-web-app',
  context: {
    userId: 'user-123',
    properties: { plan: 'premium', country: 'US' },
  },
};

function Root() {
  return (
    <FlagProvider config={config}>
      <App />
    </FlagProvider>
  );
}
```

```tsx
// src/components/Feature.tsx
import { useFlag, useVariant } from '@unleash/proxy-client-react';

function ProductPage() {
  const isNewLayout = useFlag('new_product_layout');
  const priceVariant = useVariant('price_experiment');

  return (
    <div>
      {isNewLayout ? <ProductLayoutV2 /> : <ProductLayoutV1 />}
      {priceVariant.name === 'show_discount' && <DiscountBadge />}
    </div>
  );
}
```

---

### 3.6 LaunchDarkly vs Unleash vs Custom

| Aspect | LaunchDarkly | Unleash | Custom (DIY) |
|---|---|---|---|
| **Hosting** | SaaS only | Self-host or Cloud | Self-host |
| **Cost** | $$$$ (per seat + MAU) | Free (OSS) / $$ (Cloud) | Engineering time |
| **Real-time updates** | SSE (< 200ms) | Polling (15s default) | Build it |
| **Targeting rules** | Rich UI, segments, % rollout | Good, extensible | Build it |
| **A/B experiments** | Built-in (Experimentation add-on) | Via Unleash Analytics | Build it |
| **SDKs** | 25+ (best-in-class) | 15+ (good quality) | Build it |
| **Audit log** | Full | Full | Build it |
| **TypeScript types** | Auto-generated | Manual | Manual |
| **Best for** | Enterprise, large teams | Mid-size, privacy-conscious | Simple needs |

---

### 3.7 Best Practices & Flag Hygiene

```
┌──────────────────────────────────────────────────────────┐
│                  Feature Flag Lifecycle                    │
│                                                           │
│  CREATE   →   DEVELOP   →   ROLLOUT   →   CLEANUP        │
│  Name it      Gate code     Canary (5%)   Remove flag     │
│  Document     Write tests   Expand (25%)  Remove dead     │
│  Set owner    Ship behind   Full (100%)   code paths      │
│               flag          Monitor       Archive in DB   │
└──────────────────────────────────────────────────────────┘
```

**Rules:**

| Rule | Why |
|---|---|
| **Every flag has an owner** | Someone is responsible for cleanup |
| **Set expiry dates** | Flags lingering for months become tech debt |
| **Limit active flags** | > 50 active flags = cognitive overload |
| **Test both paths** | Flag on AND flag off must work |
| **No nested flags** | `if (flagA && flagB && !flagC)` → unmaintainable |
| **Monitor flag usage** | Alert on stale flags (not evaluated in 30 days) |
| **Use naming conventions** | `release_*`, `experiment_*`, `ops_*`, `perm_*` |

```ts
// Bad: flag cleanup debt
if (useFeatureFlag('new_checkout_v2')) {
  if (useFeatureFlag('checkout_experiment_q3')) {
    if (!useFeatureFlag('disable_stripe_v4')) {
      return <CheckoutV2WithStripeV5 />;
    }
  }
}

// Good: single flag, clean code
const checkoutVersion = useFeatureFlag('checkout_version'); // 'v1' | 'v2' | 'v3'
return <Checkout version={checkoutVersion} />;
```

[⬆ Back to Top](#top)

---

## Session Replay and Heatmaps

### 4.1 How Session Replay Works

Session replay tools **record DOM mutations, mouse movements, scroll events, and network requests** to recreate a user's session as a video — without actually recording a video.

```
Browser (User's session)
  │
  ├── MutationObserver → captures DOM changes
  ├── mousemove / click / scroll → captures interactions
  ├── Performance Observer → captures resource timing
  ├── Console proxy → captures console.log / console.error
  ├── Network proxy → captures XHR / fetch requests & responses
  │
  ▼
┌──────────────────────────┐
│  Serialized Event Stream  │
│                           │
│  [                        │
│    { type: 'snapshot',    │   // Initial full DOM snapshot
│      data: <DOM tree> },  │
│    { type: 'mutation',    │   // Incremental DOM change
│      data: <diff> },      │
│    { type: 'mouse',       │   // Mouse position
│      data: {x, y} },      │
│    { type: 'input',       │   // User typed (masked)
│      data: '****' },      │
│    ...                    │
│  ]                        │
└──────────┬───────────────┘
           │
           ▼  Compressed & batched
┌──────────────────────┐
│  Replay Backend       │
│  (stores as events,   │
│   replays by          │
│   re-applying events  │
│   to a virtual DOM)   │
└──────────────────────┘
```

**Key Library:** [rrweb](https://github.com/rrweb-io/rrweb) — the open-source engine behind most session replay tools.

```ts
// Using rrweb directly
import { record } from 'rrweb';

const events: any[] = [];

const stopRecording = record({
  emit(event) {
    events.push(event);

    // Batch and send every 50 events
    if (events.length >= 50) {
      sendToBackend(events.splice(0));
    }
  },
  maskAllInputs: true,            // Mask password, email, etc.
  blockSelector: '.pii-block',    // Don't capture these elements
  maskTextSelector: '.pii-mask',  // Replace text with ***
});
```

---

### 4.2 Heatmaps — Click, Scroll, Move

| Type | What It Shows | Use Case |
|---|---|---|
| **Click heatmap** | Where users click (hot = frequent) | Find dead clicks, misclicked areas |
| **Scroll heatmap** | How far users scroll (% visible) | Determine fold line, content priority |
| **Move heatmap** | Where mouse hovers (≈ attention) | Eye-tracking proxy |
| **Rage click map** | Repeated rapid clicks on same element | Find broken UI, frustration |

```
Click Heatmap Visualization:

  ┌────────────────────────────────┐
  │  HEADER / NAV BAR              │  🔴🔴🔴 (nav links heavily clicked)
  ├────────────────────────────────┤
  │                                │
  │  Hero Banner                   │  🟡 (moderate clicks)
  │  [CTA Button]                  │  🔴🔴🔴🔴 (highest clicks)
  │                                │
  ├────────────────────────────────┤
  │                                │
  │  Product Grid                  │  🟡🟡 (moderate)
  │  Card 1  Card 2  Card 3       │
  │                                │
  ├────────────────────────────────┤
  │                                │
  │  Footer                        │  🔵 (few clicks — most users
  │                                │       don't scroll this far)
  └────────────────────────────────┘

  Scroll depth:  ███████████████  100% (top)
                 ████████████     80%
                 █████████        60%
                 ██████           40%
                 ███              20%  (only 20% reach footer)
```

---

### 4.3 Privacy & PII Concerns

| Risk | Mitigation |
|---|---|
| **Passwords / CC numbers** | `maskAllInputs: true` — replaces with `***` |
| **Sensitive text** | Apply `.pii-mask` CSS class or selector-based masking |
| **PII in URLs** | Strip query params before recording |
| **GDPR / CCPA** | Get user consent before recording; honor Do Not Track |
| **Data retention** | Auto-delete replays after 30–90 days |
| **Employee data** | Exclude internal users from recording |

---

### 4.4 Tools Compared

| Tool | Session Replay | Heatmaps | Analytics | Error Tracking | Pricing |
|---|---|---|---|---|---|
| **Hotjar** | ✅ | ✅ Click, Scroll, Move | Basic | ❌ | Free tier / $$ |
| **FullStory** | ✅ (best) | ✅ | ✅ | ✅ | $$$$ |
| **LogRocket** | ✅ | ❌ | ✅ | ✅ (Redux, network) | $$$ |
| **PostHog** | ✅ | ✅ | ✅ | ❌ | Free (self-host) / $$ |
| **Microsoft Clarity** | ✅ | ✅ Click, Scroll | Basic | ❌ | **Free** |
| **Mouseflow** | ✅ | ✅ Full suite | ✅ Funnels | ❌ | $$ |
| **Heap** | ✅ | ❌ | ✅ (auto-capture) | ❌ | $$$ |

> 💡 **Microsoft Clarity** is a great **free** option for session replay + heatmaps. It uses rrweb under the hood and integrates with Google Analytics.

[⬆ Back to Top](#top)

---

## Frontend Error Tracking and Debugging

### 5.1 Types of Frontend Errors

```
┌──────────────────────────────────────────────────────────────┐
│                  Frontend Error Taxonomy                      │
│                                                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │ JavaScript       │  │ Network          │  │ Resource     │ │
│  │ Runtime Errors   │  │ Errors           │  │ Errors       │ │
│  │                  │  │                  │  │              │ │
│  │ TypeError        │  │ API 4xx/5xx      │  │ Image 404    │ │
│  │ ReferenceError   │  │ Timeout          │  │ Script fail  │ │
│  │ RangeError       │  │ Network offline  │  │ CSS fail     │ │
│  │ SyntaxError      │  │ CORS blocked     │  │ Font fail    │ │
│  │ Unhandled        │  │ SSL errors       │  │ Chunk load   │ │
│  │ Promise reject   │  │                  │  │ error        │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
│                                                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │ Framework        │  │ Performance      │  │ User         │ │
│  │ Errors           │  │ Errors           │  │ Errors       │ │
│  │                  │  │                  │  │              │ │
│  │ React render     │  │ Memory leak      │  │ Rage clicks  │ │
│  │ error            │  │ Long task (>50ms)│  │ Dead clicks  │ │
│  │ Hydration        │  │ Layout thrashing │  │ Form errors  │ │
│  │ mismatch         │  │ Jank (dropped    │  │ Navigation   │ │
│  │ State mismatch   │  │ frames)          │  │ frustration │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

### 5.2 Capturing Errors — The Complete Picture

#### Global Error Handlers

```ts
// src/errorTracking/globalHandlers.ts

// 1. Uncaught JS errors
window.addEventListener('error', (event) => {
  // event.error    → Error object
  // event.message  → Error message
  // event.filename → Script URL
  // event.lineno   → Line number
  // event.colno    → Column number

  reportError({
    type: 'uncaught_error',
    message: event.message,
    stack: event.error?.stack,
    source: event.filename,
    line: event.lineno,
    column: event.colno,
  });
});

// 2. Unhandled Promise rejections
window.addEventListener('unhandledrejection', (event) => {
  reportError({
    type: 'unhandled_promise_rejection',
    message: event.reason?.message || String(event.reason),
    stack: event.reason?.stack,
  });
});

// 3. Resource loading errors (images, scripts, stylesheets)
window.addEventListener('error', (event) => {
  const target = event.target as HTMLElement;
  if (target.tagName) {
    reportError({
      type: 'resource_error',
      tag: target.tagName.toLowerCase(),
      src: (target as HTMLImageElement).src || (target as HTMLScriptElement).src,
    });
  }
}, true);  // ← Must use capture phase for resource errors

// 4. Console error proxy
const originalConsoleError = console.error;
console.error = (...args) => {
  reportError({
    type: 'console_error',
    message: args.map(String).join(' '),
  });
  originalConsoleError.apply(console, args);
};
```

#### Network Error Tracking

```ts
// Intercept fetch to track API errors
const originalFetch = window.fetch;

window.fetch = async (...args) => {
  const startTime = performance.now();
  const [url, options] = args;

  try {
    const response = await originalFetch(...args);
    const duration = performance.now() - startTime;

    if (!response.ok) {
      reportError({
        type: 'api_error',
        url: typeof url === 'string' ? url : url.toString(),
        method: options?.method || 'GET',
        status: response.status,
        statusText: response.statusText,
        duration,
      });
    }

    // Track slow APIs
    if (duration > 3000) {
      reportError({
        type: 'slow_api',
        url: typeof url === 'string' ? url : url.toString(),
        duration,
      });
    }

    return response;
  } catch (error) {
    reportError({
      type: 'network_error',
      url: typeof url === 'string' ? url : url.toString(),
      message: (error as Error).message,
    });
    throw error;
  }
};
```

---

### 5.3 Error Tracking with Sentry

**Sentry** is the most popular frontend error tracking platform.

#### Setup

```ts
// src/index.ts
import * as Sentry from '@sentry/react';

Sentry.init({
  dsn: 'https://examplePublicKey@o0.ingest.sentry.io/0',

  // Release & environment
  release: 'my-app@2.3.1',
  environment: process.env.NODE_ENV,

  // Performance monitoring
  integrations: [
    Sentry.browserTracingIntegration(),
    Sentry.replayIntegration({
      maskAllText: false,
      blockAllMedia: false,
    }),
  ],

  // Sample rates
  tracesSampleRate: 0.1,        // 10% of transactions for perf monitoring
  replaysSessionSampleRate: 0.1, // 10% of sessions get replay
  replaysOnErrorSampleRate: 1.0, // 100% of error sessions get replay

  // Filter noise
  ignoreErrors: [
    'ResizeObserver loop limit exceeded',
    'Non-Error promise rejection captured',
    /^Loading chunk \d+ failed/,
  ],

  // PII protection
  beforeSend(event) {
    // Strip PII from error events
    if (event.request?.url) {
      event.request.url = stripSensitiveParams(event.request.url);
    }
    return event;
  },

  // Breadcrumbs — trail of events leading to an error
  beforeBreadcrumb(breadcrumb) {
    // Don't track breadcrumbs from analytics scripts
    if (breadcrumb.category === 'xhr' &&
        breadcrumb.data?.url?.includes('analytics')) {
      return null;
    }
    return breadcrumb;
  },
});
```

#### React Error Boundary Integration

```tsx
import * as Sentry from '@sentry/react';

// Sentry-powered Error Boundary
const SentryErrorBoundary = Sentry.withErrorBoundary(App, {
  fallback: ({ error, resetError }) => (
    <div className="error-page">
      <h1>Something went wrong</h1>
      <p>Our team has been notified. Please try again.</p>
      <button onClick={resetError}>Try Again</button>
    </div>
  ),
  showDialog: true,  // Show Sentry feedback dialog
});

// Or wrap specific components
function ProductPage() {
  return (
    <Sentry.ErrorBoundary fallback={<ProductErrorFallback />}>
      <ProductDetail />
    </Sentry.ErrorBoundary>
  );
}
```

#### Manual Error Capture with Context

```ts
// Capture errors with rich context
try {
  await processPayment(order);
} catch (error) {
  Sentry.withScope((scope) => {
    // Add structured context
    scope.setTag('payment_provider', 'stripe');
    scope.setTag('order_type', 'subscription');
    scope.setLevel('critical');

    scope.setContext('order', {
      orderId: order.id,
      amount: order.total,
      currency: order.currency,
      items: order.items.length,
    });

    scope.setContext('user_journey', {
      steps_completed: ['cart', 'shipping', 'payment'],
      time_in_checkout: '3m 22s',
    });

    // Set user (without PII)
    scope.setUser({
      id: user.id,
      segment: user.plan,  // Not email — PII
    });

    Sentry.captureException(error);
  });
}
```

#### Sentry — What Gets Captured Automatically

| Data | Source | How |
|---|---|---|
| **Stack trace** | `window.onerror` | Auto (with source maps) |
| **Breadcrumbs** | Console, clicks, navigation, XHR | Auto |
| **User interactions** | Clicks, inputs, navigation | Auto (breadcrumb trail) |
| **HTTP requests** | Fetch/XHR | Auto (BrowserTracing) |
| **Performance spans** | Page load, route change | Auto (BrowserTracing) |
| **Session replay** | DOM mutations, mouse, scroll | Auto (Replay integration) |
| **Device info** | User-Agent, viewport, OS | Auto |
| **Release info** | `release` config | Manual (set in CI) |

---

### 5.4 Source Maps for Production Debugging

Minified production code produces **useless stack traces**:

```
// Minified error (useless):
TypeError: Cannot read property 'a' of undefined
  at e.value (main.abc123.js:1:24589)

// With source maps (useful):
TypeError: Cannot read property 'name' of undefined
  at ProductCard.render (src/components/ProductCard.tsx:42:18)
```

#### Source Map Strategy

```
┌──────────────────────────────────────────────────────────┐
│                Source Map Strategies                       │
│                                                           │
│  Option 1: Upload to Sentry (RECOMMENDED)                 │
│  ─────────────────────────────────────────                │
│  • Source maps uploaded during CI/CD build                 │
│  • NOT served to browsers (private)                       │
│  • Sentry un-minifies stack traces server-side            │
│  • No performance or security impact                      │
│                                                           │
│  Option 2: Hidden source maps                             │
│  ─────────────────────────────────────────                │
│  • devtool: 'hidden-source-map' in Webpack                │
│  • .map files generated but no //# sourceMappingURL       │
│  • Deployed to restricted storage (S3 with IAM)           │
│  • Error tracking service fetches them server-side        │
│                                                           │
│  Option 3: Source maps behind auth (NOT RECOMMENDED)      │
│  ─────────────────────────────────────────                │
│  • Anyone with Chrome DevTools can see your source code   │
│                                                           │
│  ❌ Option 4: No source maps in prod                      │
│  • Debugging is impossible                                │
│  • NEVER do this                                          │
└──────────────────────────────────────────────────────────┘
```

#### Uploading Source Maps to Sentry (CI/CD)

```yaml
# .github/workflows/deploy.yml
- name: Build
  run: npm run build
  env:
    GENERATE_SOURCEMAP: true

- name: Upload Source Maps to Sentry
  run: |
    npx @sentry/cli releases new ${{ github.sha }}
    npx @sentry/cli releases files ${{ github.sha }} \
      upload-sourcemaps ./build/static/js \
      --url-prefix '~/static/js'
    npx @sentry/cli releases finalize ${{ github.sha }}

- name: Delete source maps from deploy bundle
  run: find ./build -name '*.map' -delete

- name: Deploy (without source maps)
  run: aws s3 sync ./build s3://my-app-bucket
```

---

### 5.5 Error Boundaries (React)

Error boundaries catch **render-time** errors and prevent the entire app from crashing.

```tsx
// src/components/ErrorBoundary.tsx
import React, { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: ReactNode | ((props: { error: Error; reset: () => void }) => ReactNode);
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
  level?: 'page' | 'section' | 'widget';
}

interface State {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    // Report to error tracking
    this.props.onError?.(error, errorInfo);

    console.error(`[ErrorBoundary:${this.props.level}]`, error, errorInfo);
  }

  reset = () => {
    this.setState({ hasError: false, error: null });
  };

  render() {
    if (this.state.hasError) {
      if (typeof this.props.fallback === 'function') {
        return this.props.fallback({
          error: this.state.error!,
          reset: this.reset,
        });
      }
      return this.props.fallback ?? <DefaultErrorFallback onRetry={this.reset} />;
    }
    return this.props.children;
  }
}

// Granular boundaries at different levels
function App() {
  return (
    <ErrorBoundary level="page" fallback={<FullPageError />}>
      <Header />
      <main>
        <ErrorBoundary level="section" fallback={<SectionError />}>
          <ProductGrid />
        </ErrorBoundary>
        <ErrorBoundary level="widget" fallback={<WidgetError />}>
          <Recommendations />
        </ErrorBoundary>
      </main>
    </ErrorBoundary>
  );
}
```

**Error Boundary Placement Strategy:**

```
┌────────────────────────────────────────┐
│  Page-Level Boundary                    │  ← Catches catastrophic failures
│                                         │     Shows "Something went wrong" page
│  ┌──────────────────────────────────┐  │
│  │  Section-Level Boundary           │  │  ← Catches section failures
│  │                                   │  │     Shows fallback, rest of page works
│  │  ┌──────────┐  ┌──────────┐      │  │
│  │  │ Widget   │  │ Widget   │      │  │  ← Catches widget failures
│  │  │ Boundary │  │ Boundary │      │  │     Shows placeholder, section works
│  │  └──────────┘  └──────────┘      │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
```

---

### 5.6 Structured Error Context

Always enrich errors with context so debugging is fast:

```ts
// src/errorTracking/errorReporter.ts

interface ErrorReport {
  // Error details
  type: string;
  message: string;
  stack?: string;

  // Context
  userId?: string;
  sessionId: string;
  page: string;
  route: string;
  component?: string;

  // Environment
  release: string;
  environment: string;
  browser: string;
  os: string;
  viewport: string;
  network: string;

  // Timing
  timestamp: string;
  pageLoadTime?: number;
  sessionDuration?: number;

  // Breadcrumbs (last N actions before error)
  breadcrumbs: Array<{
    type: 'click' | 'navigation' | 'api' | 'console' | 'state';
    message: string;
    timestamp: string;
    data?: Record<string, any>;
  }>;

  // App state snapshot (sanitized)
  state?: {
    cart?: { itemCount: number };
    auth?: { isLoggedIn: boolean; plan: string };
    flags?: Record<string, boolean>;
  };
}
```

[⬆ Back to Top](#top)

---

## Frontend Logging Where to Hold Logs and How

### 6.1 Client-Side Log Collection

Frontend logs are fundamentally different from backend logs — they originate on the **user's device**, not your server. Getting them off the client **reliably** is the core challenge.

#### Log Levels

```ts
// src/logging/logger.ts

enum LogLevel {
  DEBUG = 0,   // Development only — verbose debugging info
  INFO = 1,    // Normal operations — page views, user actions
  WARN = 2,    // Potential issues — deprecation, slow API, retry
  ERROR = 3,   // Errors — caught exceptions, API failures
  FATAL = 4,   // Critical — app crash, payment failure
}

interface LogEntry {
  level: LogLevel;
  message: string;
  data?: Record<string, any>;
  timestamp: string;
  sessionId: string;
  userId?: string;
  page: string;
  release: string;
}

class Logger {
  private buffer: LogEntry[] = [];
  private minLevel: LogLevel;
  private maxBufferSize: number;
  private flushInterval: ReturnType<typeof setInterval>;

  constructor(config: { minLevel: LogLevel; maxBufferSize?: number; flushIntervalMs?: number }) {
    this.minLevel = config.minLevel;
    this.maxBufferSize = config.maxBufferSize ?? 50;
    this.flushInterval = setInterval(() => this.flush(), config.flushIntervalMs ?? 10000);

    // Flush on page unload
    document.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'hidden') this.flush();
    });
  }

  debug(message: string, data?: Record<string, any>) { this.log(LogLevel.DEBUG, message, data); }
  info(message: string, data?: Record<string, any>)  { this.log(LogLevel.INFO, message, data); }
  warn(message: string, data?: Record<string, any>)  { this.log(LogLevel.WARN, message, data); }
  error(message: string, data?: Record<string, any>) { this.log(LogLevel.ERROR, message, data); }
  fatal(message: string, data?: Record<string, any>) { this.log(LogLevel.FATAL, message, data); }

  private log(level: LogLevel, message: string, data?: Record<string, any>) {
    if (level < this.minLevel) return;

    const entry: LogEntry = {
      level,
      message,
      data,
      timestamp: new Date().toISOString(),
      sessionId: getSessionId(),
      userId: getCurrentUserId(),
      page: window.location.pathname,
      release: __APP_VERSION__,
    };

    this.buffer.push(entry);

    // Immediate flush for FATAL
    if (level === LogLevel.FATAL) {
      this.flush();
      return;
    }

    if (this.buffer.length >= this.maxBufferSize) {
      this.flush();
    }
  }

  private async flush() {
    if (this.buffer.length === 0) return;

    const batch = this.buffer.splice(0);

    try {
      const blob = new Blob([JSON.stringify(batch)], { type: 'application/json' });
      const sent = navigator.sendBeacon('/api/logs', blob);

      if (!sent) {
        await fetch('/api/logs', {
          method: 'POST',
          body: JSON.stringify(batch),
          headers: { 'Content-Type': 'application/json' },
          keepalive: true,
        });
      }
    } catch {
      // Save to IndexedDB for retry
      await this.saveToIndexedDB(batch);
    }
  }

  private async saveToIndexedDB(entries: LogEntry[]) {
    try {
      const db = await openLogDB();
      const tx = db.transaction('logs', 'readwrite');
      for (const entry of entries) {
        tx.objectStore('logs').add(entry);
      }
      await tx.done;
    } catch {
      // Last resort — logs are lost. Acceptable for DEBUG/INFO.
    }
  }
}

// Production: only WARN and above
// Development: everything
export const logger = new Logger({
  minLevel: process.env.NODE_ENV === 'production' ? LogLevel.WARN : LogLevel.DEBUG,
  maxBufferSize: 50,
  flushIntervalMs: 10_000,
});
```

---

### 6.2 Log Transport — Getting Logs Off the Client

| Transport Method | Reliability | Survives Page Unload | Size Limit | Use Case |
|---|---|---|---|---|
| **`navigator.sendBeacon()`** | High | ✅ Yes | ~64 KB | Primary method (analytics, logs) |
| **`fetch()` with `keepalive`** | High | ✅ Yes | ~64 KB | Fallback for sendBeacon |
| **`fetch()` (normal)** | Medium | ❌ No | Unlimited | Batch uploads |
| **`XMLHttpRequest`** | Medium | ❌ No | Unlimited | Legacy fallback |
| **`Image` pixel** | High | ⚠️ Partial | ~2 KB (URL) | Simple event pings |
| **WebSocket** | High (persistent) | ❌ No | Unlimited | Real-time log streaming |
| **IndexedDB (offline buffer)** | High | ✅ Persists | ~50 MB+ | Offline-first, retry queue |

#### Transport Priority Chain

```js
async function sendLogs(payload) {
  const body = JSON.stringify(payload);
  const blob = new Blob([body], { type: 'application/json' });

  // 1. Try sendBeacon (most reliable for page unload)
  if (navigator.sendBeacon?.('/api/logs', blob)) {
    return;
  }

  // 2. Fallback to fetch with keepalive
  try {
    await fetch('/api/logs', {
      method: 'POST',
      body,
      headers: { 'Content-Type': 'application/json' },
      keepalive: true,
    });
    return;
  } catch {}

  // 3. Last resort: save to IndexedDB for later retry
  await saveToIndexedDB(payload);
}
```

---

### 6.3 Log Storage & Querying Infrastructure

Where do logs go after they leave the browser? Here's the full pipeline:

```
┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌───────────────┐
│  Browser  │    │  Ingestion    │    │  Processing   │    │  Storage &     │
│           │    │  Layer        │    │  Layer        │    │  Querying      │
│ sendBeacon│───>│              │───>│              │───>│               │
│ fetch()   │    │  Options:     │    │  Options:     │    │  Options:      │
│           │    │  • API Gateway│    │  • Kafka      │    │  • ELK Stack   │
│           │    │  • Nginx      │    │  • Kinesis    │    │  • Loki+Grafana│
│           │    │  • Fluentd    │    │  • Datadog    │    │  • Datadog     │
│           │    │  • Vector     │    │    Pipeline   │    │  • CloudWatch  │
│           │    │  • CloudFlare │    │  • Logstash   │    │  • Splunk      │
│           │    │    Workers    │    │  • Vector     │    │  • ClickHouse  │
└──────────┘    └──────────────┘    └──────────────┘    └───────────────┘
```

#### Option 1: ELK Stack (Elasticsearch + Logstash + Kibana)

```
Browser → API Endpoint → Logstash → Elasticsearch → Kibana Dashboard

Pros: Powerful full-text search, great for debugging
Cons: Resource-heavy, expensive at scale

Best for: Large orgs with dedicated infra teams
```

#### Option 2: Loki + Grafana (Lightweight)

```
Browser → API Endpoint → Promtail/Vector → Loki → Grafana

Pros: Lightweight, cost-effective (only indexes labels, not content)
Cons: Weaker full-text search than ELK

Best for: Teams already using Grafana for metrics
```

#### Option 3: Datadog

```
Browser → Datadog Browser SDK → Datadog Logs → Datadog Dashboard

Pros: SaaS, zero infra, correlates logs + traces + metrics
Cons: Expensive at volume

Best for: Teams that want a single observability platform
```

#### Option 4: AWS CloudWatch Logs

```
Browser → API Gateway → Lambda → CloudWatch Logs → CloudWatch Insights

Pros: Serverless, auto-scales, native AWS integration
Cons: Query language is limited, UI is basic

Best for: AWS-native teams
```

#### Option 5: Sentry (Errors + Logs combined)

```
Browser → Sentry SDK → Sentry → Sentry Dashboard

Pros: Errors + breadcrumbs + replay in one place
Cons: Not designed for high-volume logging (use for errors/warnings only)

Best for: Error-focused logging (not general analytics logs)
```

#### Comparison Matrix

| Solution | Self-Host | Cost | Query Power | Setup | Best For |
|---|---|---|---|---|---|
| **ELK Stack** | ✅ | $$$–$$$$ (infra) | ⭐⭐⭐⭐⭐ | Complex | Large scale, full-text search |
| **Loki + Grafana** | ✅ | $–$$ | ⭐⭐⭐ | Medium | Cost-conscious, Grafana users |
| **Datadog** | ❌ SaaS | $$$$ | ⭐⭐⭐⭐ | Easy | All-in-one observability |
| **CloudWatch** | ❌ AWS | $$ | ⭐⭐⭐ | Easy | AWS-native teams |
| **Sentry** | ❌ SaaS (or self-host) | $$–$$$ | ⭐⭐⭐ | Easy | Error-focused |
| **ClickHouse** | ✅ | $–$$ | ⭐⭐⭐⭐ | Medium | Custom analytics + logs |

---

### 6.4 Structured Logging Best Practices

```ts
// ❌ BAD: Unstructured logs — impossible to query
console.log('User clicked button');
console.error('Payment failed for user john');

// ✅ GOOD: Structured logs — queryable, filterable
logger.info('button_clicked', {
  button_id: 'add_to_cart',
  product_id: 'SKU-123',
  page: '/product/SKU-123',
});

logger.error('payment_failed', {
  provider: 'stripe',
  error_code: 'card_declined',
  order_id: 'ORD-456',
  amount: 99.99,
  currency: 'USD',
  // ❌ NEVER log: user.email, user.name, card number, password
});
```

**Structured log fields:**

| Field | Always Include? | Purpose |
|---|---|---|
| `timestamp` | ✅ | When it happened |
| `level` | ✅ | Severity (debug/info/warn/error/fatal) |
| `message` | ✅ | What happened (snake_case event name) |
| `sessionId` | ✅ | Group logs by session |
| `userId` | If available | Group logs by user |
| `page` / `route` | ✅ | Where it happened |
| `release` | ✅ | Which app version |
| `data.*` | Situational | Additional structured context |
| `traceId` | For APIs | Correlate frontend + backend logs |

---

### 6.5 Log Levels & Sampling

Not all logs are equal. Sending every `debug` log from every user would be overwhelming and expensive.

```
Production Sampling Strategy:

  FATAL  → 100% sent (every fatal is critical)
  ERROR  → 100% sent (every error matters)
  WARN   → 50% sampled (reduce noise, still catch patterns)
  INFO   → 10% sampled (spot-check user flows)
  DEBUG  → 0% in production (development only)

  Exceptions:
  - 100% for users with active support tickets
  - 100% for internal/beta users
  - 100% for users in active experiments
```

```ts
// Sampling implementation
function shouldSample(level: LogLevel): boolean {
  const rates: Record<LogLevel, number> = {
    [LogLevel.FATAL]: 1.0,
    [LogLevel.ERROR]: 1.0,
    [LogLevel.WARN]:  0.5,
    [LogLevel.INFO]:  0.1,
    [LogLevel.DEBUG]: 0.0,
  };

  // Always log for internal users
  if (isInternalUser()) return true;

  return Math.random() < (rates[level] ?? 0);
}
```

---

### 6.6 Complete Logging Pipeline Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                   Frontend Logging Pipeline                          │
│                                                                      │
│   COLLECTION          TRANSPORT         PROCESSING       STORAGE     │
│                                                                      │
│  ┌─────────────┐    ┌────────────┐    ┌────────────┐  ┌──────────┐  │
│  │ Logger       │    │ sendBeacon │    │ API Gateway│  │ Kafka /  │  │
│  │ (in-memory   │───>│ (primary)  │───>│ / Nginx    │─>│ Kinesis  │  │
│  │  buffer)     │    │            │    │            │  │          │  │
│  │              │    │ fetch()    │    └────────────┘  └────┬─────┘  │
│  │ Batch (20)   │    │ (fallback) │                         │        │
│  │ Flush (10s)  │    │            │                         ▼        │
│  │ Flush on     │    └────────────┘                   ┌──────────┐  │
│  │  unload      │                                     │ Stream    │  │
│  └──────┬──────┘                                     │ processor │  │
│         │ offline?                                    │ (enrich,  │  │
│         ▼                                            │  filter,  │  │
│  ┌─────────────┐    On reconnect                     │  route)   │  │
│  │ IndexedDB    │────── retry ──────────>             └─────┬────┘  │
│  │ (offline     │                                          │        │
│  │  buffer)     │                                          ▼        │
│  └─────────────┘                                    ┌──────────┐   │
│                                                      │ Storage   │   │
│                                                      │           │   │
│                                        Errors ──────>│ Sentry    │   │
│                                        Logs ────────>│ ELK/Loki  │   │
│                                        Analytics ───>│ ClickHouse│   │
│                                        Replays ─────>│ FullStory │   │
│                                                      └──────────┘   │
│                                                           │         │
│                                                           ▼         │
│                                                      ┌──────────┐  │
│                                                      │ Dashboards│  │
│                                                      │ & Alerts  │  │
│                                                      │ (Grafana, │  │
│                                                      │  PagerDuty│  │
│                                                      │  Slack)   │  │
│                                                      └──────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

[⬆ Back to Top](#top)

---

## Putting It All Together Unified Observability

In production, these systems are interconnected:

```
User Action
    │
    ├──→ Analytics:    track('product_added_to_cart', { ... })
    │
    ├──→ Feature Flag: useFeatureFlag('new_cart_animation')
    │
    ├──→ A/B Test:     experiment exposure tracked
    │
    ├──→ Logger:       logger.info('cart_updated', { itemCount: 3 })
    │
    └──→ Error (if any): Sentry captures with breadcrumbs + replay

All connected by:
  • sessionId  — groups all events from one visit
  • userId     — groups all events from one user
  • traceId    — connects frontend request to backend span
  • release    — associates events with a specific deploy
```

### Unified SDK Example

```ts
// src/observability/index.ts
import { analytics } from './analytics';
import { logger } from './logger';
import { featureFlags } from './featureFlags';
import * as Sentry from '@sentry/react';

export const observability = {
  // Analytics
  track: analytics.track.bind(analytics),

  // Logging
  log: logger,

  // Feature flags
  flag: featureFlags.evaluate.bind(featureFlags),

  // Error tracking
  captureError: (error: Error, context?: Record<string, any>) => {
    Sentry.captureException(error, { extra: context });
    logger.error(error.message, context);
  },

  // Identify user across all systems
  identify(user: { id: string; plan: string; [key: string]: any }) {
    analytics.identify(user);
    Sentry.setUser({ id: user.id });
    featureFlags.updateContext({ userId: user.id, plan: user.plan });
    logger.info('user_identified', { userId: user.id });
  },
};
```

[⬆ Back to Top](#top)

---

## Decision Matrix and Quick Reference

### What Tool for What Job?

| Need | Tool Category | Recommendations |
|---|---|---|
| **"What are users doing?"** | Product Analytics | Amplitude, Mixpanel, PostHog |
| **"Why is this page slow?"** | Performance Monitoring | Web Vitals + Sentry Perf / Datadog RUM |
| **"What happened before the crash?"** | Error Tracking + Replay | Sentry (errors) + LogRocket/FullStory (replay) |
| **"Where are users clicking?"** | Heatmaps | Hotjar, Microsoft Clarity (free) |
| **"Is the new design better?"** | A/B Testing | LaunchDarkly, Statsig, PostHog |
| **"Can I ship this safely?"** | Feature Flags | LaunchDarkly, Unleash, Flagsmith |
| **"What's in the logs?"** | Log Management | ELK, Loki+Grafana, Datadog Logs |
| **"User reported a bug"** | Session Replay | FullStory, LogRocket, Sentry Replay |

### Startup vs Enterprise Stack

| Stage | Recommended Stack | Monthly Cost |
|---|---|---|
| **Early startup** | PostHog (all-in-one, free self-host) + Sentry (free tier) | $0–$50 |
| **Growing startup** | Amplitude (analytics) + Sentry (errors) + LaunchDarkly (flags) + Clarity (replay) | $200–$1,000 |
| **Mid-size** | Segment (CDP) + Amplitude + Sentry + LaunchDarkly + FullStory | $2,000–$10,000 |
| **Enterprise** | Segment + custom analytics (ClickHouse) + Datadog (full stack) + LaunchDarkly | $10,000+ |

[⬆ Back to Top](#top)

---

## Key Interview Takeaways

| Topic | Key Points to Mention |
|---|---|
| **Analytics** | Event taxonomy (object_action), batching with sendBeacon, sampling for cost, funnel tracking for business metrics |
| **A/B Testing** | Server-side evaluation to avoid flicker, deterministic hashing for assignment, statistical significance before calling results |
| **Feature Flags** | Decouple deploy from release, singleton evaluation (LD/Unleash), flag hygiene (expiry, owners, cleanup) |
| **Session Replay** | DOM diffing (rrweb / MutationObserver), privacy masking, storage cost vs value |
| **Error Tracking** | Global handlers (`error`, `unhandledrejection`), source maps uploaded to Sentry (not served), breadcrumbs for context, Error Boundaries at page/section/widget levels |
| **Logging** | Structured logs (not console.log), sendBeacon + IndexedDB for reliability, log levels + sampling for cost, ELK/Loki/Datadog for storage |
| **Where to store logs** | Sentry for errors, ELK/Loki/Datadog for general logs, ClickHouse for analytics, IndexedDB for offline buffer — never only in the browser |

[⬆ Back to Top](#top)

---

## Further Reading and Resources

- [Sentry Documentation](https://docs.sentry.io/)
- [LaunchDarkly Docs](https://docs.launchdarkly.com/)
- [Unleash Documentation](https://docs.getunleash.io/)
- [rrweb — Open Source Session Replay](https://github.com/rrweb-io/rrweb)
- [PostHog — Open Source Product Analytics](https://posthog.com/)
- [Microsoft Clarity — Free Heatmaps & Replay](https://clarity.microsoft.com/)
- [Segment Analytics.js](https://segment.com/docs/connections/sources/catalog/libraries/website/javascript/)
- [Web Vitals Library](https://github.com/GoogleChrome/web-vitals)
- [Grafana Loki Documentation](https://grafana.com/docs/loki/latest/)

[⬆ Back to Top](#top)

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design

[⬆ Back to Top](#top)
