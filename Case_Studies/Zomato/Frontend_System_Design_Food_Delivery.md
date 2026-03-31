# Frontend System Design: Zomato Swiggy Food Delivery Web Application

- [Frontend System Design: Zomato Swiggy Food Delivery Web Application](#frontend-system-design-zomato-swiggy-food-delivery-web-application)
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
    - [5.2 Restaurant Listing with Filters and Real Time Cart Strategy (Deep Dive)](#52-restaurant-listing-with-filters-and-real-time-cart-strategy-deep-dive)
      - [5.2.1 Restaurant Card Grid with Dynamic Loading](#521-restaurant-card-grid-with-dynamic-loading)
      - [5.2.2 Multi Faceted Filtering with URL Sync](#522-multi-faceted-filtering-with-url-sync)
      - [5.2.3 Horizontal Scrollable Category Carousels](#523-horizontal-scrollable-category-carousels)
      - [5.2.4 Location Aware Restaurant Discovery](#524-location-aware-restaurant-discovery)
      - [5.2.5 Menu Page with Sticky Cart and Category Navigation](#525-menu-page-with-sticky-cart-and-category-navigation)
      - [5.2.6 Cart Management Across Restaurant Contexts](#526-cart-management-across-restaurant-contexts)
      - [5.2.7 Real Time Order Tracking with Map Integration](#527-real-time-order-tracking-with-map-integration)
      - [5.2.8 Scroll Performance and Lazy Loading Strategy](#528-scroll-performance-and-lazy-loading-strategy)
      - [5.2.9 Decision Matrix](#529-decision-matrix)
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
  - [9. High Level API Design (Frontend POV)](#9-high-level-api-design-frontend-pov)
    - [9.1 Required APIs](#91-required-apis)
    - [9.2 Real Time Order Tracking Strategy (Deep Dive)](#92-real-time-order-tracking-strategy-deep-dive)
    - [9.3 Request and Response Structure](#93-request-and-response-structure)
    - [9.4 Error Handling and Status Codes](#94-error-handling-and-status-codes)
  - [10. Caching Strategy](#10-caching-strategy)
  - [11. CDN and Asset Optimization](#11-cdn-and-asset-optimization)
  - [12. Rendering Strategy](#12-rendering-strategy)
  - [13. Cross Cutting Non Functional Concerns](#13-cross-cutting-non-functional-concerns)
    - [13.1 Security](#131-security)
    - [13.2 Accessibility](#132-accessibility)
    - [13.3 Performance Optimization](#133-performance-optimization)
    - [13.4 Observability and Reliability](#134-observability-and-reliability)
  - [14. Edge Cases and Tradeoffs](#14-edge-cases-and-tradeoffs)
  - [15. Summary and Future Improvements](#15-summary-and-future-improvements)

---

## 1. Concept, Idea, Product Overview

### 1.1 Product Description

*   A food delivery web application (like Zomato / Swiggy) that allows users to discover nearby restaurants, browse menus, place orders, and track deliveries in real time.
*   Target users: hungry consumers looking to order food online, ranging from daily office lunch orderers to weekend family meal planners.
*   Primary use case: browsing restaurant listings based on location, adding food items to a cart, placing an order with online payment, and tracking the order until delivery.

---

### 1.2 Key User Personas

*   **Hungry Consumer**: Browses restaurants near their location, applies filters (cuisine, rating, delivery time, price), selects a restaurant, browses the menu, adds items to cart, places the order, and tracks delivery in real time.
*   **Repeat Orderer**: Relies on order history, reorders previous meals, uses saved addresses, and expects personalized recommendations based on past orders.
*   **Deal Seeker**: Primarily browses via offers and coupons, filters by discounts, and looks for the best deal before placing an order.

---

### 1.3 Core User Flows (High Level)

*   **Ordering Food (Primary Flow)**:
    1.  User opens the app → location is auto-detected or manually entered.
    2.  Home page loads with restaurant listings, banners, cuisine carousels, and top offers.
    3.  User applies filters (cuisine, rating, delivery time, price range) → restaurant list updates.
    4.  User taps a restaurant → restaurant menu page loads with categorized items.
    5.  User adds items to cart → sticky cart bar appears at the bottom with item count and total.
    6.  User taps "View Cart" → cart page shows item breakdown, customizations, delivery fee, taxes.
    7.  User selects address, applies coupon, selects payment method → places order.
    8.  Order confirmation page loads → real-time tracking begins (order accepted → preparing → out for delivery → delivered).

*   **Searching for a Specific Dish (Secondary Flow)**:
    1.  User types in the search bar → autocomplete suggests dishes, restaurants, and cuisines.
    2.  User selects a dish → search results show restaurants that serve that dish, sorted by relevance and delivery time.
    3.  User picks a restaurant → jumps to that dish within the restaurant menu.

*   **Reordering a Previous Meal (Secondary Flow)**:
    1.  User navigates to "My Orders" → sees past order history.
    2.  Taps "Reorder" → cart is populated with the same items.
    3.  If any item is unavailable, a prompt suggests alternatives.
    4.  User proceeds to checkout.

---

## 2. Requirements

### 2.1 Functional Requirements

*   **Restaurant Listing**:
    *   Display a list/grid of nearby restaurants based on the user's delivery location.
    *   Each restaurant card shows: name, image/thumbnail, cuisines, rating, delivery time estimate, price for two, offer tag (if any), promoted badge.
    *   Support multiple layout modes: card grid on desktop, vertical list on mobile.
*   **Filtering and Sorting**:
    *   Filter by: cuisine type, rating (4+), delivery time (under 30 min), price range, veg/non-veg, offers available.
    *   Sort by: relevance (default), delivery time, rating, cost low to high, cost high to low.
    *   Filters are reflected in the URL for shareability and back/forward navigation.
*   **Restaurant Menu Page**:
    *   Display menu items grouped by category (Starters, Main Course, Desserts, Beverages).
    *   Sticky category navigation bar that highlights the active section on scroll.
    *   Each menu item shows: name, description, price, veg/non-veg indicator, image (if available), customization options.
    *   "Add to Cart" button per item, with quantity stepper (+ / −) after first add.
*   **Cart Management**:
    *   Persistent sticky cart bar at the bottom of the menu page showing item count and total price.
    *   Full cart page with item breakdown, customization details, delivery fee, taxes, and tip option.
    *   If user tries to add items from a different restaurant, show a confirmation dialog to clear the existing cart.
    *   Apply coupon codes with real-time validation.
*   **Order Placement and Tracking**:
    *   Address selection (saved addresses + add new via map picker).
    *   Payment method selection (UPI, card, wallet, cash on delivery).
    *   Real-time order status tracking with a map showing delivery partner's live location.
    *   Status progression: Order Placed → Accepted → Preparing → Out for Delivery → Delivered.
*   **Search**:
    *   Search bar with autocomplete for dishes, restaurants, and cuisines.
    *   Search results page with restaurant cards that serve the searched dish.
*   **Location Management**:
    *   Auto-detect location via Geolocation API.
    *   Manual address entry with autocomplete (Google Places).
    *   Saved addresses with labels (Home, Work, Other).

---

### 2.2 Non Functional Requirements

*   **Performance**: FCP < 1.5s; TTI < 3s; smooth 60fps scrolling through restaurant lists; menu page category scroll must be seamless; cart interactions must feel instant (< 100ms perceived).
*   **Scalability**: Support listing hundreds of restaurants per area; menus with 200+ items; thousands of concurrent users during peak meal hours.
*   **Availability**: Graceful degradation — show cached restaurant listings if network is slow; skeleton placeholders during loading; offline cart persistence.
*   **Security**: XSS prevention on user-generated content (reviews, addresses); CSRF tokens on all state-changing requests; PCI compliance for payment data (handled via payment gateway SDK); authenticated API calls.
*   **Accessibility**: Full keyboard navigation through restaurant list and menu items; screen reader support with semantic HTML; focus management on modals (cart, address picker); color contrast compliance for veg/non-veg indicators.
*   **Device Support**: Mobile web (primary — 70%+ traffic), desktop web (responsive), low-end Android devices with limited memory.
*   **i18n**: RTL layout support; localized currency formatting; locale-aware distance and time display; multi-language menu support.

---

## 3. Scope Clarification (Interview Scoping)

### 3.1 In Scope

*   Restaurant listing page with card grid, filters, sorting, and infinite scroll.
*   Restaurant menu page with categorized items, sticky navigation, and add-to-cart interactions.
*   Cart management with sticky cart bar, full cart page, and coupon application.
*   Real-time order tracking with WebSocket and map integration.
*   Location management (auto-detect + manual entry).
*   Search with autocomplete.
*   State management for cart, filters, location, and order tracking.
*   Performance optimization for image-heavy restaurant listings.
*   API design from the frontend perspective.

---

### 3.2 Out of Scope

*   Backend recommendation/ranking algorithm for restaurant ordering.
*   Payment gateway integration internals (assume third-party SDK handles it).
*   Restaurant admin panel (menu management, order acceptance flow).
*   Delivery partner app and assignment logic.
*   Push notifications.
*   Review and rating submission flow.
*   Loyalty/rewards program.

---

### 3.3 Assumptions

*   User is authenticated; auth token is available via HTTP-only cookie or Authorization header.
*   APIs return restaurants pre-ranked by the backend based on location, relevance, and promotions.
*   Restaurant images and food item images are served from a CDN with multiple size variants (thumbnail, medium, original).
*   Location coordinates are available either via Geolocation API or manual address selection before restaurant listing is shown.
*   Payment processing is handled by a third-party SDK (Razorpay, Stripe) that provides a drop-in iframe/modal.

---

## 4. High Level Frontend Architecture

### 4.1 Overall Approach

*   **SPA** (Single Page Application) with client-side routing.
*   **SSR** for the home page and restaurant listing — server renders the initial restaurant cards for fast FCP, SEO (restaurant pages should be indexable with structured data), and social preview (OG metadata for shared restaurant links).
*   **CSR** for all subsequent interactions — menu browsing, cart management, order tracking, filter changes.
*   Home page and restaurant listing are the **primary chunks**; menu page, cart, checkout, and order tracking are **code-split** and lazy loaded on navigation.

---

### 4.2 Major Architectural Layers

```
┌──────────────────────────────────────────────────────────────────┐
│  UI Layer                                                        │
│  ┌────────────────┐  ┌───────────────────────────────────────┐   │
│  │ LocationBar    │  │ Restaurant List (Scroll Container)    │   │
│  │ (detect/pick)  │  │  ┌──────────────┐ ┌──────────────┐    │   │
│  └────────────────┘  │  │ RestaurantCard│ │ RestaurantCard│    │   │
│  ┌────────────────┐  │  └──────────────┘ └──────────────┘    │   │
│  │ FilterBar      │  │       ┌───────────────────┐           │   │
│  │ (cuisine,      │  │       │ ListSentinel      │           │   │
│  │  sort, veg)    │  │       │ (infinite scroll)  │           │   │
│  └────────────────┘  │       └───────────────────┘           │   │
│  ┌────────────────┐  └───────────────────────────────────────┘   │
│  │ SearchBar      │  ┌───────────────────────────────────────┐   │
│  └────────────────┘  │ MenuPage  │ CartPage  │ OrderTracking │   │
│  ┌────────────────┐  └───────────────────────────────────────┘   │
│  │ StickyCartBar  │                                              │
│  └────────────────┘                                              │
├──────────────────────────────────────────────────────────────────┤
│  State Management Layer                                          │
│  (Cart Store, Filter State, Location State, Order Tracking,      │
│   User Session, Restaurant Cache)                                │
├──────────────────────────────────────────────────────────────────┤
│  API and Data Access Layer                                       │
│  (REST Client, WebSocket Manager for Order Tracking,             │
│   Request Deduplication, Retry Logic, Optimistic Cart Updates)   │
├──────────────────────────────────────────────────────────────────┤
│  Shared / Utility Layer                                          │
│  (IntersectionObserver helpers, Debounce/Throttle,               │
│   Currency formatting, Distance calculator, Analytics tracker,   │
│   Geolocation wrapper, Map SDK loader)                           │
└──────────────────────────────────────────────────────────────────┘
```

---

### 4.3 External Integrations

*   **CDN**: Serves restaurant images, food item images, banner assets (Cloudfront / Akamai / Fastly).
*   **Map SDK**: Google Maps / Mapbox for delivery tracking map, address picker with autocomplete, and location pin.
*   **Payment Gateway SDK**: Razorpay / Stripe drop-in checkout for payment processing (iframed).
*   **Analytics SDK**: Track impressions (restaurant viewed), engagement events (menu opened, item added, order placed), funnel metrics, and session data.
*   **Backend Services**: Restaurant listing API, menu API, cart API, order API, WebSocket endpoint for real-time tracking.
*   **Geolocation API**: Browser native API for auto-detecting user location with fallback to IP-based geolocation.

---

## 5. Component Design and Modularization

### 5.1 Component Hierarchy

```
App
 ├── LocationBar
 │    ├── CurrentLocationLabel ("Delivering to: Koramangala")
 │    └── LocationPickerModal
 │         ├── GeolocationDetectButton
 │         ├── AddressSearchInput (Google Places autocomplete)
 │         ├── SavedAddressList
 │         └── MapPinPicker
 │
 ├── Header
 │    ├── Logo
 │    ├── SearchBar (with autocomplete dropdown)
 │    ├── CartIcon (with badge count)
 │    └── UserMenu (profile, orders, logout)
 │
 ├── HomePage
 │    ├── PromoBannerCarousel (hero banners with offers)
 │    ├── CuisineCategoryScroller (horizontal chip/icon scroll)
 │    ├── FilterBar
 │    │    ├── FilterChip[] (Veg, Rating 4+, Under 30 min, Offers, etc.)
 │    │    └── SortDropdown
 │    ├── RestaurantList (scrollable grid container)
 │    │    └── RestaurantCard[] (repeated per restaurant)
 │    │         ├── RestaurantImage (lazy loaded with blur placeholder)
 │    │         ├── OfferTag ("50% OFF up to 100")
 │    │         ├── RestaurantInfo (name, cuisines, rating, delivery time, price)
 │    │         └── PromotedBadge (if sponsored)
 │    └── ListSentinel (IntersectionObserver — triggers next page load)
 │
 ├── RestaurantMenuPage
 │    ├── RestaurantHeader (banner, name, rating, delivery info)
 │    ├── MenuSearchInput ("Search within menu")
 │    ├── CategoryNav (sticky horizontal nav — scrolls to categories)
 │    ├── MenuCategoryList
 │    │    └── MenuCategory[] (repeated per category)
 │    │         ├── CategoryTitle ("Starters", "Main Course")
 │    │         └── MenuItem[] (repeated per item)
 │    │              ├── VegNonVegIcon
 │    │              ├── ItemName + Description
 │    │              ├── ItemPrice
 │    │              ├── ItemImage (thumbnail, optional)
 │    │              ├── AddToCartButton / QuantityStepper
 │    │              └── CustomizationModal (size, toppings, extras)
 │    └── StickyCartBar ("2 items | Rs 450 — View Cart")
 │
 ├── CartPage
 │    ├── CartItemList
 │    │    └── CartItem[] (name, qty stepper, price, customizations)
 │    ├── CouponInput (apply/remove coupon)
 │    ├── BillDetails (item total, delivery fee, taxes, tip, grand total)
 │    ├── DeliveryAddressSelector
 │    └── PlaceOrderButton
 │
 ├── OrderTrackingPage
 │    ├── OrderStatusStepper (Placed → Accepted → Preparing → Out → Delivered)
 │    ├── DeliveryMap (live location of delivery partner)
 │    ├── EstimatedTimeDisplay
 │    ├── DeliveryPartnerInfo (name, photo, phone)
 │    └── OrderDetails (items ordered, bill summary)
 │
 └── Footer (links, app download, social links)
```

---

### 5.2 Restaurant Listing with Filters and Real Time Cart Strategy (Deep Dive)

The restaurant listing and cart management are the **core of the entire application** — a location-aware, filterable, infinitely scrolling list of restaurant cards combined with a persistent cart that follows the user across pages. Getting this right matters because:
*   The listing page is the **primary discovery surface** — users spend significant time browsing here during peak hours.
*   Filters and sorting must update **instantly** without full page reloads, and state must sync with the URL.
*   Restaurant cards are **image-heavy** — each card has a restaurant photo, offer badge, and rating; loading performance is critical.
*   The cart must **persist** across restaurant and menu page transitions and handle the single-restaurant constraint elegantly.
*   Real-time order tracking requires **WebSocket** integration with a live map — a complex UI state machine.

---

#### 5.2.1 Restaurant Card Grid with Dynamic Loading

##### The Core Pattern — Card Grid with IntersectionObserver

Like social feeds, the restaurant listing uses **IntersectionObserver** for infinite scroll. However, unlike a simple feed, the restaurant list must also handle **filter changes** that completely replace the list content.

```
┌──────────────────────────────────────┐
│  FilterBar (sticky at top)           │
│  [Veg] [Rating 4+] [Under 30 min]   │
│  Sort: Relevance ▾                   │
├──────────────────────────────────────┤
│  ┌────────────┐  ┌────────────┐      │
│  │ Restaurant │  │ Restaurant │      │  ← visible cards
│  │    Card 1  │  │    Card 2  │      │
│  └────────────┘  └────────────┘      │
│  ┌────────────┐  ┌────────────┐      │
│  │ Restaurant │  │ Restaurant │      │  ← visible cards
│  │    Card 3  │  │    Card 4  │      │
│  └────────────┘  └────────────┘      │
├──────────────────────────────────────┤
│  ┌────────────┐  ┌────────────┐      │
│  │    Card 5  │  │    Card 6  │      │  ← below viewport
│  └────────────┘  └────────────┘      │
│  ┌──────────────────────────────┐    │
│  │  SENTINEL (0px height)       │    │  ← triggers next page
│  └──────────────────────────────┘    │
└──────────────────────────────────────┘
```

##### Implementation — Restaurant List with Filter Reset

When filters change, the entire list must be replaced (not appended). This requires resetting the cursor and clearing existing data.

```tsx
import { useEffect, useRef, useCallback, useState } from 'react';
import { useSearchParams } from 'react-router-dom';

interface Filters {
  cuisine?: string;
  rating?: number;
  maxDeliveryTime?: number;
  vegOnly?: boolean;
  hasOffers?: boolean;
  sortBy?: 'relevance' | 'deliveryTime' | 'rating' | 'costLowToHigh' | 'costHighToLow';
}

function RestaurantList() {
  const [searchParams, setSearchParams] = useSearchParams();
  const sentinelRef = useRef<HTMLDivElement>(null);
  const [restaurants, setRestaurants] = useState<Restaurant[]>([]);
  const [cursor, setCursor] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);

  // Derive filters from URL search params
  const filters: Filters = {
    cuisine: searchParams.get('cuisine') || undefined,
    rating: searchParams.get('rating') ? Number(searchParams.get('rating')) : undefined,
    maxDeliveryTime: searchParams.get('maxTime') ? Number(searchParams.get('maxTime')) : undefined,
    vegOnly: searchParams.get('veg') === 'true',
    hasOffers: searchParams.get('offers') === 'true',
    sortBy: (searchParams.get('sort') as Filters['sortBy']) || 'relevance',
  };

  // Fetch restaurants with current filters and cursor
  const fetchRestaurants = useCallback(async (
    currentFilters: Filters,
    currentCursor: string | null,
    append: boolean
  ) => {
    if (isLoading) return;
    setIsLoading(true);

    try {
      const params = new URLSearchParams();
      if (currentCursor) params.set('cursor', currentCursor);
      params.set('limit', '20');
      if (currentFilters.cuisine) params.set('cuisine', currentFilters.cuisine);
      if (currentFilters.rating) params.set('rating', String(currentFilters.rating));
      if (currentFilters.maxDeliveryTime) params.set('maxTime', String(currentFilters.maxDeliveryTime));
      if (currentFilters.vegOnly) params.set('veg', 'true');
      if (currentFilters.hasOffers) params.set('offers', 'true');
      if (currentFilters.sortBy) params.set('sort', currentFilters.sortBy);

      const response = await fetch(`/api/restaurants?${params}`);
      const data = await response.json();

      if (append) {
        setRestaurants((prev) => [...prev, ...data.restaurants]);
      } else {
        // Filter change → replace the entire list
        setRestaurants(data.restaurants);
      }
      setCursor(data.nextCursor);
      setHasMore(data.hasMore);
    } catch (error) {
      // Error handling in section 6.3
    } finally {
      setIsLoading(false);
    }
  }, [isLoading]);

  // When filters change (URL params change), reset and reload
  useEffect(() => {
    setCursor(null);
    setHasMore(true);
    fetchRestaurants(filters, null, false);
    // Scroll to top on filter change
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }, [searchParams]); // Triggered by URL changes

  // IntersectionObserver for infinite scroll
  useEffect(() => {
    const sentinel = sentinelRef.current;
    if (!sentinel) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && hasMore && !isLoading) {
          fetchRestaurants(filters, cursor, true);
        }
      },
      { root: null, rootMargin: '0px 0px 400px 0px', threshold: 0 }
    );

    observer.observe(sentinel);
    return () => observer.disconnect();
  }, [cursor, hasMore, isLoading, filters, fetchRestaurants]);

  // Update URL when user changes a filter
  const updateFilter = (key: string, value: string | null) => {
    const newParams = new URLSearchParams(searchParams);
    if (value === null) {
      newParams.delete(key);
    } else {
      newParams.set(key, value);
    }
    setSearchParams(newParams);
  };

  return (
    <div className="restaurant-list-page">
      <FilterBar filters={filters} onFilterChange={updateFilter} />

      <div className="restaurant-grid">
        {restaurants.map((restaurant) => (
          <RestaurantCard key={restaurant.id} restaurant={restaurant} />
        ))}
      </div>

      {hasMore && (
        <div ref={sentinelRef} style={{ height: 0 }} aria-hidden="true" />
      )}
      {isLoading && <RestaurantListSkeleton count={4} />}
      {!hasMore && restaurants.length > 0 && <EndOfListMessage />}
      {!isLoading && restaurants.length === 0 && <NoRestaurantsFound filters={filters} />}
    </div>
  );
}
```

##### Why URL-Synced Filters Matter

| Benefit | Explanation |
|---------|-------------|
| **Shareability** | User can share a URL like `/restaurants?cuisine=pizza&rating=4&sort=deliveryTime` and the recipient sees the same filtered view. |
| **Back/Forward Navigation** | Browser back button restores the previous filter state naturally using URL history. |
| **Deep Linking** | Marketing campaigns can link directly to filtered views (e.g., "Best rated restaurants near you"). |
| **SSR Compatibility** | Server can read query params and pre-render the filtered restaurant list for faster FCP. |
| **Analytics** | Filter usage patterns are automatically captured in URL-based analytics without extra instrumentation. |

---

#### 5.2.2 Multi Faceted Filtering with URL Sync

Food delivery filters are **multi-faceted** — users frequently combine multiple filters (e.g., "Veg + Rating 4+ + Under 30 min"). The challenge is managing multiple active filter states simultaneously.

##### Filter Chip Pattern

```tsx
function FilterBar({ filters, onFilterChange }: FilterBarProps) {
  const filterChips = [
    {
      key: 'veg',
      label: 'Pure Veg',
      active: filters.vegOnly,
      toggle: () => onFilterChange('veg', filters.vegOnly ? null : 'true'),
    },
    {
      key: 'rating',
      label: 'Rating 4.0+',
      active: !!filters.rating,
      toggle: () => onFilterChange('rating', filters.rating ? null : '4'),
    },
    {
      key: 'maxTime',
      label: 'Under 30 min',
      active: !!filters.maxDeliveryTime,
      toggle: () => onFilterChange('maxTime', filters.maxDeliveryTime ? null : '30'),
    },
    {
      key: 'offers',
      label: 'Offers',
      active: filters.hasOffers,
      toggle: () => onFilterChange('offers', filters.hasOffers ? null : 'true'),
    },
  ];

  return (
    <div className="filter-bar" role="toolbar" aria-label="Restaurant filters">
      {filterChips.map((chip) => (
        <button
          key={chip.key}
          className={`filter-chip ${chip.active ? 'active' : ''}`}
          onClick={chip.toggle}
          aria-pressed={chip.active}
          role="switch"
        >
          {chip.label}
          {chip.active && <span className="chip-close" aria-hidden="true">✕</span>}
        </button>
      ))}

      <SortDropdown
        value={filters.sortBy || 'relevance'}
        onChange={(val) => onFilterChange('sort', val === 'relevance' ? null : val)}
      />
    </div>
  );
}
```

##### Filter Request Flow

```
User taps "Rating 4+" chip
  │
  ├── 1. Update URL: /restaurants?rating=4&veg=true  (if veg was already active)
  │
  ├── 2. useEffect detects searchParams change
  │
  ├── 3. Reset cursor to null, clear restaurant list
  │
  ├── 4. Fetch: GET /api/restaurants?rating=4&veg=true&limit=20
  │
  ├── 5. API returns filtered results
  │
  └── 6. Render new restaurant grid + scroll to top
```

---

#### 5.2.3 Horizontal Scrollable Category Carousels

The home page features **horizontal scrollable carousels** for cuisine categories (Pizza, Biryani, Burgers, Chinese, etc.) and promotional banners. These need to work seamlessly on both desktop (with arrow buttons) and mobile (with native touch scroll).

##### CSS Based Horizontal Scroll

```tsx
function CuisineCategoryScroller() {
  const scrollRef = useRef<HTMLDivElement>(null);

  const scroll = (direction: 'left' | 'right') => {
    const container = scrollRef.current;
    if (!container) return;
    const scrollAmount = container.clientWidth * 0.8;
    container.scrollBy({
      left: direction === 'left' ? -scrollAmount : scrollAmount,
      behavior: 'smooth',
    });
  };

  return (
    <div className="category-scroller-wrapper">
      <button
        className="scroll-arrow left"
        onClick={() => scroll('left')}
        aria-label="Scroll cuisines left"
      >
        ‹
      </button>

      <div
        ref={scrollRef}
        className="category-scroller"
        role="list"
        aria-label="Cuisine categories"
      >
        {cuisines.map((cuisine) => (
          <a
            key={cuisine.id}
            href={`/restaurants?cuisine=${cuisine.slug}`}
            className="cuisine-item"
            role="listitem"
          >
            <img
              src={cuisine.iconUrl}
              alt={cuisine.name}
              loading="lazy"
              width="80"
              height="80"
            />
            <span>{cuisine.name}</span>
          </a>
        ))}
      </div>

      <button
        className="scroll-arrow right"
        onClick={() => scroll('right')}
        aria-label="Scroll cuisines right"
      >
        ›
      </button>
    </div>
  );
}
```

```css
.category-scroller {
  display: flex;
  gap: 16px;
  overflow-x: auto;
  scroll-snap-type: x mandatory;
  -webkit-overflow-scrolling: touch;
  scrollbar-width: none; /* Firefox */
  padding: 8px 0;
}

.category-scroller::-webkit-scrollbar {
  display: none; /* Chrome/Safari */
}

.cuisine-item {
  scroll-snap-align: start;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-decoration: none;
  width: 96px;
}

.scroll-arrow {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  z-index: 2;
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 50%;
  width: 36px;
  height: 36px;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

/* Hide arrows on mobile (native touch scroll is better) */
@media (max-width: 768px) {
  .scroll-arrow { display: none; }
}
```

---

#### 5.2.4 Location Aware Restaurant Discovery

Location is the **foundation** of the entire listing — every restaurant query requires a delivery coordinate. The location flow has multiple layers of fallback:

```
App loads
  │
  ├── 1. Check localStorage for last saved delivery address
  │     ├── Found → Use it, show "Delivering to: Koramangala"
  │     └── Not found → Continue to step 2
  │
  ├── 2. Request browser Geolocation (navigator.geolocation)
  │     ├── Permission granted → Reverse geocode → Show address
  │     ├── Permission denied → Continue to step 3
  │     └── Timeout / Error → Continue to step 3
  │
  ├── 3. Fallback: IP-based geolocation (server-side header or API)
  │     ├── Approximate city detected → Show "Delivering to: Bangalore"
  │     └── Failed → Continue to step 4
  │
  └── 4. Show LocationPickerModal → User must manually enter address
        └── Google Places autocomplete → Select → Save to localStorage
```

##### Geolocation Hook

```tsx
function useUserLocation() {
  const [location, setLocation] = useState<LocationState>({
    coordinates: null,
    address: null,
    source: 'none',
    isLoading: true,
  });

  useEffect(() => {
    // Step 1: Check localStorage
    const saved = localStorage.getItem('deliveryAddress');
    if (saved) {
      const parsed = JSON.parse(saved);
      setLocation({
        coordinates: parsed.coordinates,
        address: parsed.address,
        source: 'saved',
        isLoading: false,
      });
      return;
    }

    // Step 2: Browser Geolocation
    if ('geolocation' in navigator) {
      navigator.geolocation.getCurrentPosition(
        async (position) => {
          const { latitude, longitude } = position.coords;
          const address = await reverseGeocode(latitude, longitude);
          const loc = {
            coordinates: { lat: latitude, lng: longitude },
            address,
            source: 'geolocation' as const,
            isLoading: false,
          };
          setLocation(loc);
          localStorage.setItem('deliveryAddress', JSON.stringify(loc));
        },
        () => {
          // Step 3: Fallback to IP-based (handled by backend header)
          fetchIPLocation().then((ipLoc) => {
            setLocation({
              coordinates: ipLoc.coordinates,
              address: ipLoc.city,
              source: 'ip',
              isLoading: false,
            });
          }).catch(() => {
            // Step 4: Manual entry required
            setLocation((prev) => ({ ...prev, isLoading: false, source: 'none' }));
          });
        },
        { timeout: 5000, maximumAge: 300000 } // 5s timeout, 5min cache
      );
    }
  }, []);

  return location;
}
```

---

#### 5.2.5 Menu Page with Sticky Cart and Category Navigation

The restaurant menu page is where the user **selects food items** — it needs two key patterns: a **sticky category navigation** bar that highlights the active section as the user scrolls, and a **sticky cart bar** at the bottom.

##### Sticky Category Navigation with Scroll Spy

```
┌──────────────────────────────────────┐
│  Restaurant Header (banner, info)    │
├──────────────────────────────────────┤
│  [Recommended] [Starters] [Main]... │  ← sticky category nav
├──────────────────────────────────────┤
│  ★ Recommended                       │
│  ┌──────────────────────────────────┐│
│  │ Paneer Tikka      ₹249  [ADD]   ││
│  │ Chicken Wings     ₹299  [ADD]   ││
│  └──────────────────────────────────┘│
│  Starters                            │
│  ┌──────────────────────────────────┐│
│  │ Spring Roll       ₹149  [ADD]   ││
│  │ Soup of the Day   ₹129  [2 −+] ││  ← already in cart
│  └──────────────────────────────────┘│
│  Main Course                         │
│  ┌──────────────────────────────────┐│
│  │ Dal Makhani       ₹229  [ADD]   ││
│  └──────────────────────────────────┘│
├──────────────────────────────────────┤
│  ┌──────────────────────────────────┐│
│  │  2 items | ₹378 — VIEW CART →   ││  ← sticky cart bar
│  └──────────────────────────────────┘│
└──────────────────────────────────────┘
```

##### Scroll Spy Implementation

The category navigation bar highlights the active category based on which section is currently visible. This uses IntersectionObserver to track which category sections are in the viewport.

```tsx
function MenuPage({ restaurantId }: { restaurantId: string }) {
  const [activeCategory, setActiveCategory] = useState<string>('');
  const categoryRefs = useRef<Map<string, HTMLElement>>(new Map());

  const { data: menu } = useQuery({
    queryKey: ['menu', restaurantId],
    queryFn: () => fetchMenu(restaurantId),
  });

  // Scroll spy — observe each category section
  useEffect(() => {
    if (!menu) return;

    const observers: IntersectionObserver[] = [];

    menu.categories.forEach((category) => {
      const element = categoryRefs.current.get(category.id);
      if (!element) return;

      const observer = new IntersectionObserver(
        ([entry]) => {
          if (entry.isIntersecting) {
            setActiveCategory(category.id);
          }
        },
        {
          root: null,
          rootMargin: '-80px 0px -70% 0px', // top offset for sticky nav
          threshold: 0,
        }
      );

      observer.observe(element);
      observers.push(observer);
    });

    return () => observers.forEach((obs) => obs.disconnect());
  }, [menu]);

  // Scroll to category when nav item is clicked
  const scrollToCategory = (categoryId: string) => {
    const element = categoryRefs.current.get(categoryId);
    if (element) {
      const offset = 80; // height of sticky nav
      const top = element.getBoundingClientRect().top + window.scrollY - offset;
      window.scrollTo({ top, behavior: 'smooth' });
    }
  };

  if (!menu) return <MenuPageSkeleton />;

  return (
    <div className="menu-page">
      <RestaurantHeader restaurant={menu.restaurant} />

      {/* Sticky category nav */}
      <nav className="category-nav" role="tablist" aria-label="Menu categories">
        {menu.categories.map((category) => (
          <button
            key={category.id}
            role="tab"
            aria-selected={activeCategory === category.id}
            className={`category-tab ${activeCategory === category.id ? 'active' : ''}`}
            onClick={() => scrollToCategory(category.id)}
          >
            {category.name} ({category.items.length})
          </button>
        ))}
      </nav>

      {/* Menu items by category */}
      <div className="menu-content">
        {menu.categories.map((category) => (
          <section
            key={category.id}
            ref={(el) => { if (el) categoryRefs.current.set(category.id, el); }}
            id={`category-${category.id}`}
            aria-labelledby={`heading-${category.id}`}
          >
            <h2 id={`heading-${category.id}`}>{category.name}</h2>
            {category.items.map((item) => (
              <MenuItemCard key={item.id} item={item} />
            ))}
          </section>
        ))}
      </div>

      <StickyCartBar restaurantId={restaurantId} />
    </div>
  );
}
```

```css
.category-nav {
  position: sticky;
  top: 0;
  z-index: 10;
  background: white;
  display: flex;
  overflow-x: auto;
  scrollbar-width: none;
  border-bottom: 1px solid #e0e0e0;
  padding: 0 16px;
}

.category-tab {
  flex-shrink: 0;
  padding: 12px 16px;
  border: none;
  background: none;
  font-size: 14px;
  color: #666;
  cursor: pointer;
  white-space: nowrap;
  border-bottom: 2px solid transparent;
  transition: color 0.2s, border-color 0.2s;
}

.category-tab.active {
  color: #e23744; /* Zomato red */
  border-bottom-color: #e23744;
  font-weight: 600;
}
```

---

#### 5.2.6 Cart Management Across Restaurant Contexts

The cart in a food delivery app has a unique constraint: **items can only be from one restaurant at a time**. If a user has items from Restaurant A and tries to add from Restaurant B, the app must prompt them.

##### Cart State Machine

```
                    ┌─────────┐
                    │  EMPTY  │
                    └────┬────┘
                         │ add item
                         ▼
              ┌──────────────────┐
              │  ACTIVE          │
              │  (restaurantId)  │◄──── add item (same restaurant)
              └────┬────────┬───┘
                   │        │
   add item        │        │ place order
   (different      │        │
    restaurant)    │        ▼
                   │   ┌──────────┐
                   │   │  PLACED  │ → order tracking
                   │   └──────────┘
                   ▼
          ┌────────────────┐
          │  CONFLICT      │
          │  "Replace cart  │
          │   with items    │
          │   from [new]?"  │
          └───┬────────┬───┘
              │        │
        Yes   │        │ No
              ▼        ▼
         Clear cart   Keep cart
         + add new    (go back)
```

##### Cart Store Implementation

```tsx
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface CartItem {
  id: string;
  menuItemId: string;
  name: string;
  price: number;
  quantity: number;
  customizations?: { name: string; price: number }[];
}

interface CartState {
  restaurantId: string | null;
  restaurantName: string | null;
  items: CartItem[];
  couponCode: string | null;
  couponDiscount: number;

  addItem: (restaurantId: string, restaurantName: string, item: CartItem) => 'added' | 'conflict';
  removeItem: (itemId: string) => void;
  updateQuantity: (itemId: string, quantity: number) => void;
  clearCart: () => void;
  forceReplaceCart: (restaurantId: string, restaurantName: string, item: CartItem) => void;
  applyCoupon: (code: string, discount: number) => void;
  removeCoupon: () => void;

  // Derived
  totalItems: () => number;
  subtotal: () => number;
}

const useCartStore = create<CartState>()(
  persist(
    (set, get) => ({
      restaurantId: null,
      restaurantName: null,
      items: [],
      couponCode: null,
      couponDiscount: 0,

      addItem: (restaurantId, restaurantName, item) => {
        const state = get();
        // If cart has items from a different restaurant, signal conflict
        if (state.restaurantId && state.restaurantId !== restaurantId && state.items.length > 0) {
          return 'conflict';
        }

        set((prev) => {
          const existing = prev.items.find((i) => i.menuItemId === item.menuItemId);
          if (existing) {
            return {
              ...prev,
              items: prev.items.map((i) =>
                i.menuItemId === item.menuItemId
                  ? { ...i, quantity: i.quantity + 1 }
                  : i
              ),
            };
          }
          return {
            ...prev,
            restaurantId,
            restaurantName,
            items: [...prev.items, { ...item, quantity: 1 }],
          };
        });
        return 'added';
      },

      removeItem: (itemId) =>
        set((prev) => {
          const newItems = prev.items.filter((i) => i.id !== itemId);
          return {
            items: newItems,
            restaurantId: newItems.length === 0 ? null : prev.restaurantId,
            restaurantName: newItems.length === 0 ? null : prev.restaurantName,
          };
        }),

      updateQuantity: (itemId, quantity) =>
        set((prev) => ({
          items: quantity <= 0
            ? prev.items.filter((i) => i.id !== itemId)
            : prev.items.map((i) => (i.id === itemId ? { ...i, quantity } : i)),
        })),

      clearCart: () =>
        set({ items: [], restaurantId: null, restaurantName: null, couponCode: null, couponDiscount: 0 }),

      forceReplaceCart: (restaurantId, restaurantName, item) =>
        set({
          restaurantId,
          restaurantName,
          items: [{ ...item, quantity: 1 }],
          couponCode: null,
          couponDiscount: 0,
        }),

      applyCoupon: (code, discount) => set({ couponCode: code, couponDiscount: discount }),
      removeCoupon: () => set({ couponCode: null, couponDiscount: 0 }),

      totalItems: () => get().items.reduce((sum, i) => sum + i.quantity, 0),
      subtotal: () => get().items.reduce((sum, i) => sum + i.price * i.quantity, 0),
    }),
    {
      name: 'food-delivery-cart',
      // Persist cart to localStorage so it survives page refreshes
    }
  )
);
```

##### Restaurant Conflict Dialog

```tsx
function AddToCartButton({ restaurantId, restaurantName, item }: AddToCartProps) {
  const [showConflict, setShowConflict] = useState(false);
  const { addItem, forceReplaceCart } = useCartStore();

  const handleAdd = () => {
    const result = addItem(restaurantId, restaurantName, item);
    if (result === 'conflict') {
      setShowConflict(true);
    }
  };

  return (
    <>
      <button onClick={handleAdd} className="add-to-cart-btn">ADD</button>

      {showConflict && (
        <ConfirmDialog
          title="Replace cart items?"
          message={`Your cart contains items from ${useCartStore.getState().restaurantName}. 
                     Do you want to clear the cart and add items from ${restaurantName}?`}
          confirmText="Yes, start fresh"
          cancelText="No"
          onConfirm={() => {
            forceReplaceCart(restaurantId, restaurantName, item);
            setShowConflict(false);
          }}
          onCancel={() => setShowConflict(false)}
        />
      )}
    </>
  );
}
```

---

#### 5.2.7 Real Time Order Tracking with Map Integration

After order placement, the user enters the **order tracking** phase — a live dashboard showing order status progression and the delivery partner's real-time location on a map.

##### Order Status State Machine

```
PLACED ──→ ACCEPTED ──→ PREPARING ──→ PICKED_UP ──→ OUT_FOR_DELIVERY ──→ DELIVERED
  │            │             │            │                │
  │            │             │            │                └── ETA updates via WebSocket
  │            │             │            └── Map shows live rider location
  │            │             └── Kitchen preparing; show estimated time
  │            └── Restaurant acknowledged order
  └── Order submitted; awaiting restaurant confirmation
  
At any point: ──→ CANCELLED (by user, restaurant, or system)
```

##### WebSocket Based Tracking

```tsx
function useOrderTracking(orderId: string) {
  const [orderStatus, setOrderStatus] = useState<OrderTrackingState>({
    status: 'PLACED',
    estimatedDeliveryTime: null,
    rider: null,
    riderLocation: null,
    statusHistory: [],
  });

  useEffect(() => {
    const ws = new WebSocket(
      `wss://api.example.com/orders/${encodeURIComponent(orderId)}/track`
    );

    ws.onmessage = (event) => {
      const update = JSON.parse(event.data);

      switch (update.type) {
        case 'STATUS_CHANGE':
          setOrderStatus((prev) => ({
            ...prev,
            status: update.status,
            estimatedDeliveryTime: update.eta,
            statusHistory: [
              ...prev.statusHistory,
              { status: update.status, timestamp: update.timestamp },
            ],
          }));
          break;

        case 'RIDER_ASSIGNED':
          setOrderStatus((prev) => ({
            ...prev,
            rider: {
              name: update.rider.name,
              phone: update.rider.phone,
              photo: update.rider.photo,
            },
          }));
          break;

        case 'RIDER_LOCATION':
          setOrderStatus((prev) => ({
            ...prev,
            riderLocation: {
              lat: update.lat,
              lng: update.lng,
            },
          }));
          break;

        case 'ETA_UPDATE':
          setOrderStatus((prev) => ({
            ...prev,
            estimatedDeliveryTime: update.eta,
          }));
          break;
      }
    };

    ws.onclose = () => {
      // Reconnect with exponential backoff
      setTimeout(() => {
        // Reconnection logic
      }, 3000);
    };

    return () => ws.close();
  }, [orderId]);

  return orderStatus;
}
```

##### Order Tracking UI Component

```tsx
function OrderTrackingPage({ orderId }: { orderId: string }) {
  const tracking = useOrderTracking(orderId);

  const statusSteps = [
    { key: 'PLACED', label: 'Order Placed' },
    { key: 'ACCEPTED', label: 'Restaurant Accepted' },
    { key: 'PREPARING', label: 'Preparing Your Food' },
    { key: 'PICKED_UP', label: 'Rider Picked Up' },
    { key: 'OUT_FOR_DELIVERY', label: 'On the Way' },
    { key: 'DELIVERED', label: 'Delivered' },
  ];

  const currentIndex = statusSteps.findIndex((s) => s.key === tracking.status);

  return (
    <div className="order-tracking">
      {/* Status stepper */}
      <div className="status-stepper" role="progressbar" aria-valuenow={currentIndex + 1} aria-valuemax={statusSteps.length}>
        {statusSteps.map((step, index) => (
          <div
            key={step.key}
            className={`step ${index <= currentIndex ? 'completed' : ''} ${index === currentIndex ? 'current' : ''}`}
          >
            <div className="step-dot" />
            <span className="step-label">{step.label}</span>
            {tracking.statusHistory.find((h) => h.status === step.key) && (
              <span className="step-time">
                {formatTime(tracking.statusHistory.find((h) => h.status === step.key)!.timestamp)}
              </span>
            )}
          </div>
        ))}
      </div>

      {/* ETA display */}
      {tracking.estimatedDeliveryTime && (
        <div className="eta-display">
          <span className="eta-label">Estimated Delivery</span>
          <span className="eta-time">{formatETA(tracking.estimatedDeliveryTime)}</span>
        </div>
      )}

      {/* Live map (only when rider is assigned and out for delivery) */}
      {tracking.riderLocation && (
        <DeliveryMap
          riderLocation={tracking.riderLocation}
          deliveryLocation={tracking.deliveryAddress}
          restaurantLocation={tracking.restaurantLocation}
        />
      )}

      {/* Delivery partner info */}
      {tracking.rider && (
        <div className="rider-info">
          <img src={tracking.rider.photo} alt={tracking.rider.name} className="rider-photo" />
          <div>
            <p className="rider-name">{tracking.rider.name}</p>
            <p className="rider-role">Delivery Partner</p>
          </div>
          <a href={`tel:${tracking.rider.phone}`} className="call-button" aria-label={`Call ${tracking.rider.name}`}>
            📞
          </a>
        </div>
      )}
    </div>
  );
}
```

##### Delivery Map Integration

```tsx
function DeliveryMap({ riderLocation, deliveryLocation, restaurantLocation }: DeliveryMapProps) {
  const mapRef = useRef<HTMLDivElement>(null);
  const mapInstanceRef = useRef<google.maps.Map | null>(null);
  const riderMarkerRef = useRef<google.maps.Marker | null>(null);

  // Initialize map
  useEffect(() => {
    if (!mapRef.current) return;

    const map = new google.maps.Map(mapRef.current, {
      center: riderLocation,
      zoom: 15,
      disableDefaultUI: true,
      zoomControl: true,
    });
    mapInstanceRef.current = map;

    // Restaurant marker (static)
    new google.maps.Marker({
      position: restaurantLocation,
      map,
      icon: { url: '/icons/restaurant-pin.svg', scaledSize: new google.maps.Size(32, 32) },
      title: 'Restaurant',
    });

    // Delivery address marker (static)
    new google.maps.Marker({
      position: deliveryLocation,
      map,
      icon: { url: '/icons/home-pin.svg', scaledSize: new google.maps.Size(32, 32) },
      title: 'Delivery Address',
    });

    // Rider marker (will be updated)
    riderMarkerRef.current = new google.maps.Marker({
      position: riderLocation,
      map,
      icon: { url: '/icons/rider-pin.svg', scaledSize: new google.maps.Size(40, 40) },
      title: 'Delivery Partner',
    });

    return () => { map.unbindAll?.(); };
  }, []);

  // Update rider position smoothly
  useEffect(() => {
    if (riderMarkerRef.current && riderLocation) {
      // Animate marker movement for smooth tracking
      animateMarker(riderMarkerRef.current, riderLocation);
      mapInstanceRef.current?.panTo(riderLocation);
    }
  }, [riderLocation]);

  return (
    <div
      ref={mapRef}
      className="delivery-map"
      style={{ width: '100%', height: '300px', borderRadius: '12px' }}
      role="img"
      aria-label="Live delivery tracking map"
    />
  );
}

function animateMarker(marker: google.maps.Marker, newPosition: google.maps.LatLngLiteral) {
  const start = marker.getPosition()!;
  const end = new google.maps.LatLng(newPosition.lat, newPosition.lng);
  const duration = 1000; // 1 second animation
  const startTime = Date.now();

  function step() {
    const elapsed = Date.now() - startTime;
    const progress = Math.min(elapsed / duration, 1);
    // Ease-out interpolation
    const eased = 1 - Math.pow(1 - progress, 3);

    const lat = start.lat() + (end.lat() - start.lat()) * eased;
    const lng = start.lng() + (end.lng() - start.lng()) * eased;
    marker.setPosition({ lat, lng });

    if (progress < 1) {
      requestAnimationFrame(step);
    }
  }

  requestAnimationFrame(step);
}
```

---

#### 5.2.8 Scroll Performance and Lazy Loading Strategy

Restaurant listings are **image-heavy** — each card has a large restaurant photo, and pages can have 50+ cards loaded during a scroll session.

##### Image Loading Strategy for Restaurant Cards

| Technique | How It Works | When to Use |
|-----------|-------------|-------------|
| **Native lazy loading** | `loading="lazy"` on `<img>` | Default for all restaurant images below the fold |
| **Blur-up placeholder** | Show a tiny (20px) blurred version, fade in full image on load | Restaurant card thumbnails — prevents layout shift and looks polished |
| **LQIP (Low Quality Image Placeholder)** | Inline a base64 4x4 pixel image as placeholder | Critical for LCP image (first restaurant card above the fold) |
| **Responsive images** | `srcSet` with 200w, 400w, 800w variants + `sizes` attribute | Serve optimal resolution per device (mobile gets 200w, desktop gets 800w) |
| **Content-visibility** | `content-visibility: auto` on cards below the fold | Reduce initial rendering cost for off-screen cards |

##### Restaurant Card with Optimized Image Loading

```tsx
function RestaurantCard({ restaurant }: { restaurant: Restaurant }) {
  return (
    <article className="restaurant-card" style={{ contentVisibility: 'auto', containIntrinsicSize: '0 320px' }}>
      <a href={`/restaurant/${restaurant.slug}`} className="card-link">
        <div className="card-image-wrapper">
          <img
            src={restaurant.imageUrl}
            srcSet={`
              ${restaurant.imageUrl}?w=200 200w,
              ${restaurant.imageUrl}?w=400 400w,
              ${restaurant.imageUrl}?w=800 800w
            `}
            sizes="(max-width: 600px) 100vw, (max-width: 1024px) 50vw, 33vw"
            alt={`${restaurant.name} restaurant`}
            loading="lazy"
            decoding="async"
            width="400"
            height="250"
            className="card-image"
            style={{ backgroundImage: `url(${restaurant.blurHash})`, backgroundSize: 'cover' }}
          />
          {restaurant.offer && (
            <div className="offer-tag" aria-label={`Offer: ${restaurant.offer}`}>
              {restaurant.offer}
            </div>
          )}
          {restaurant.promoted && (
            <span className="promoted-badge">Promoted</span>
          )}
        </div>

        <div className="card-info">
          <div className="card-header">
            <h3 className="restaurant-name">{restaurant.name}</h3>
            <div className={`rating-badge ${restaurant.rating >= 4 ? 'high' : ''}`}>
              ★ {restaurant.rating}
            </div>
          </div>
          <p className="cuisines">{restaurant.cuisines.join(', ')}</p>
          <div className="card-meta">
            <span className="delivery-time">🕐 {restaurant.deliveryTime} min</span>
            <span className="price-for-two">₹{restaurant.priceForTwo} for two</span>
          </div>
        </div>
      </a>
    </article>
  );
}
```

---

#### 5.2.9 Decision Matrix

##### Rendering Strategy for Restaurant List

| Approach | Pros | Cons | Best For |
|----------|------|------|----------|
| **Simple list with lazy images** | Minimal complexity; native browser lazy loading | All DOM nodes present; memory grows with scroll depth | Small lists (< 50 restaurants) |
| **IntersectionObserver infinite scroll** | Smooth UX; preloads before user reaches bottom; easy to implement | DOM grows over time (all loaded cards remain) | Most food delivery apps; lists up to 200 items per session |
| **Virtualized list** | Constant DOM size; handles massive lists | Adds complexity; requires fixed or measured heights; scroll restoration harder | Extremely long lists (1000+ items); low-end devices |
| **content-visibility: auto** | CSS-only; browser skips rendering off-screen elements | Less precise than virtualization; scrollbar may jump; browser support varies | Progressive enhancement on top of infinite scroll |

**Recommendation for Zomato/Swiggy**: **IntersectionObserver infinite scroll with content-visibility: auto** as an enhancement. Food delivery listings rarely exceed 200 restaurants per session (due to location-based filtering), so full virtualization is overkill. The combination of native lazy image loading + content-visibility + sentinel-based pagination is the sweet spot.

##### Cart Persistence Strategy

| Approach | Pros | Cons | Best For |
|----------|------|------|----------|
| **In-memory only (Zustand store)** | Simplest; no serialization overhead | Lost on page refresh | Prototyping; low-stakes apps |
| **localStorage (Zustand persist)** | Survives refresh; fast read/write | 5MB limit; synchronous; no expiry | Most food delivery carts; simple data |
| **Server-side cart** | Cross-device sync; no data loss | API calls for every cart change; latency | Multi-device experiences; high-value carts |
| **localStorage + server sync** | Best of both — instant local updates + server backup | Conflict resolution needed; more complex | Production food delivery apps at scale |

**Recommendation**: **Zustand with localStorage persistence** for the primary cart, with an optional server sync on order placement. Cart data is small (typically < 20 items), and the user expects it to survive page refreshes but not necessarily sync across devices.

---

### 5.3 Reusability Strategy

*   **Design system components**: Button, Input, Modal, Chip, Badge, Skeleton, Toast, BottomSheet — shared across all pages.
*   **RestaurantCard** is reused in: home listing, search results, cuisine filtered view, and "similar restaurants" recommendations.
*   **MenuItemCard** is reused in: restaurant menu page, order history (reorder view), and cart page (simplified variant).
*   **RatingBadge** composes rating value + color logic (+star icon) and is used in RestaurantCard, RestaurantHeader, and order history.
*   **StickyBottomBar** pattern is reused for: cart bar (menu page), place order button (cart page), and delivery tracking CTA.
*   **QuantityStepper** (−/+/count) is shared between MenuItemCard and CartItem.
*   Props-driven configuration: e.g., `<RestaurantCard variant="compact" />` for search results vs `<RestaurantCard variant="full" />` for home listing.

---

### 5.4 Module Organization

```
src/
├── pages/
│   ├── HomePage/
│   ├── RestaurantMenuPage/
│   ├── CartPage/
│   ├── CheckoutPage/
│   ├── OrderTrackingPage/
│   ├── SearchPage/
│   └── OrderHistoryPage/
├── features/
│   ├── restaurant-listing/
│   │   ├── RestaurantList.tsx
│   │   ├── RestaurantCard.tsx
│   │   ├── FilterBar.tsx
│   │   ├── SortDropdown.tsx
│   │   └── hooks/useRestaurantList.ts
│   ├── menu/
│   │   ├── MenuCategoryList.tsx
│   │   ├── MenuItemCard.tsx
│   │   ├── CategoryNav.tsx
│   │   ├── CustomizationModal.tsx
│   │   └── hooks/useMenuScrollSpy.ts
│   ├── cart/
│   │   ├── StickyCartBar.tsx
│   │   ├── CartItemList.tsx
│   │   ├── CartItem.tsx
│   │   ├── CouponInput.tsx
│   │   ├── BillDetails.tsx
│   │   └── store/cartStore.ts
│   ├── order-tracking/
│   │   ├── OrderStatusStepper.tsx
│   │   ├── DeliveryMap.tsx
│   │   ├── RiderInfo.tsx
│   │   └── hooks/useOrderTracking.ts
│   ├── location/
│   │   ├── LocationBar.tsx
│   │   ├── LocationPickerModal.tsx
│   │   └── hooks/useUserLocation.ts
│   └── search/
│       ├── SearchBar.tsx
│       ├── SearchAutocomplete.tsx
│       └── SearchResults.tsx
├── shared/
│   ├── components/ (Button, Modal, Skeleton, Toast, BottomSheet, etc.)
│   ├── hooks/ (useIntersectionObserver, useDebounce, useMediaQuery)
│   └── utils/ (currency.ts, date.ts, distance.ts, analytics.ts)
├── api/
│   ├── restaurantApi.ts
│   ├── menuApi.ts
│   ├── cartApi.ts
│   ├── orderApi.ts
│   └── httpClient.ts
└── styles/
    ├── tokens.css (colors, spacing, typography)
    ├── global.css
    └── components/ (per-component CSS modules)
```

---

## 6. High Level Data Flow Explanation

### 6.1 Initial Load Flow

```
User opens app
  │
  ├── 1. SSR: Server reads location from cookie/IP → fetches first batch of restaurants
  │        → Renders HTML shell with restaurant cards + hydration markers
  │
  ├── 2. Browser receives HTML → FCP fires (user sees restaurant cards)
  │
  ├── 3. JS bundle loads → React hydrates the server-rendered content
  │
  ├── 4. Location hook initializes:
  │     ├── Reads saved address from localStorage
  │     ├── OR triggers Geolocation API
  │     └── If location differs from SSR, re-fetches restaurant list
  │
  ├── 5. Cart store rehydrates from localStorage (if user had items from previous session)
  │
  └── 6. App is interactive (TTI) — user can scroll, filter, tap restaurants
```

### 6.2 User Interaction Flow

```
User adds item to cart (on menu page)
  │
  ├── 1. Click "ADD" button on MenuItemCard
  │
  ├── 2. Cart store checks: same restaurant as existing cart items?
  │     ├── YES → Item added to cart array; quantity = 1
  │     ├── NO (cart has items from different restaurant) → Show conflict dialog
  │     │    ├── User confirms → clearCart() + addItem()
  │     │    └── User cancels → no change
  │     └── EMPTY cart → Item added; restaurantId set
  │
  ├── 3. UI updates instantly:
  │     ├── "ADD" button transforms into QuantityStepper (− 1 +)
  │     ├── StickyCartBar at bottom updates: "2 items | ₹378 — VIEW CART"
  │     └── CartIcon in header badge count updates
  │
  ├── 4. Cart persisted to localStorage (Zustand persist middleware)
  │
  └── 5. (Optional) Cart synced to server for cross-device continuity
         POST /api/cart/sync { restaurantId, items[] }
```

### 6.3 Error and Retry Flow

*   **Restaurant list fetch fails**:
    *   Show error banner above the list with a "Retry" button.
    *   If cached data exists (React Query stale data), continue showing stale results with an "Update failed" indicator.
    *   Automatic retry with exponential backoff (1s, 2s, 4s) — max 3 retries.

*   **Menu fetch fails**:
    *   Show full-page error state with "Try again" button on the menu page.
    *   If the menu was previously cached, show stale data with a "Menu may be outdated" notice.

*   **Add to cart fails (server-synced cart)**:
    *   Since cart is primarily local (Zustand + localStorage), this only fails if server sync is enabled.
    *   Optimistic add to local cart; if server sync fails, queue for retry. User experience is unaffected.

*   **Order placement fails**:
    *   Show error toast: "Order failed. Please try again."
    *   Do NOT clear the cart — user can retry without re-adding items.
    *   If payment was deducted but order API failed, show "Payment received, but order could not be placed. Contact support." with support link.

*   **WebSocket disconnection (order tracking)**:
    *   Reconnect with exponential backoff.
    *   While disconnected, fall back to polling: `GET /api/orders/{id}/status` every 10 seconds.
    *   Show a "Reconnecting..." indicator on the tracking page.

---

## 7. Data Modelling (Frontend Perspective)

### 7.1 Core Data Entities

*   **Restaurant**: The listing entity shown in card grids — name, image, rating, cuisines, delivery info, offers.
*   **MenuItem**: An item within a restaurant's menu — name, price, description, category, customizations.
*   **CartItem**: A menu item added to the cart — references a MenuItem with quantity and selected customizations.
*   **Order**: A placed order — items, restaurant, delivery address, payment, status, tracking info.
*   **User Address**: Saved delivery locations — coordinates, formatted address, label.
*   **Category**: A cuisine or menu category — used for filtering and grouping.

---

### 7.2 Data Shape

```ts
interface Restaurant {
  id: string;
  name: string;
  slug: string;
  imageUrl: string;
  blurHash: string; // tiny placeholder for blur-up loading
  cuisines: string[];
  rating: number;
  totalRatings: number;
  deliveryTime: number; // in minutes
  priceForTwo: number;
  offer?: string; // e.g., "50% OFF up to ₹100"
  promoted: boolean;
  isOpen: boolean;
  distance: number; // km from user
  location: {
    lat: number;
    lng: number;
  };
}

interface MenuCategory {
  id: string;
  name: string;
  items: MenuItem[];
}

interface MenuItem {
  id: string;
  name: string;
  description: string;
  price: number;
  imageUrl?: string;
  isVeg: boolean;
  isAvailable: boolean;
  isBestseller: boolean;
  customizationGroups?: CustomizationGroup[];
}

interface CustomizationGroup {
  id: string;
  title: string;  // e.g., "Choose Size", "Add Toppings"
  required: boolean;
  maxSelections: number;
  options: CustomizationOption[];
}

interface CustomizationOption {
  id: string;
  name: string;
  price: number; // additional price
}

interface CartItem {
  id: string; // unique cart item id
  menuItemId: string;
  name: string;
  price: number;
  quantity: number;
  customizations: SelectedCustomization[];
}

interface SelectedCustomization {
  groupId: string;
  groupTitle: string;
  optionId: string;
  optionName: string;
  price: number;
}

interface Order {
  id: string;
  restaurantId: string;
  restaurantName: string;
  items: CartItem[];
  subtotal: number;
  deliveryFee: number;
  taxes: number;
  tip: number;
  total: number;
  couponCode?: string;
  couponDiscount: number;
  status: OrderStatus;
  placedAt: string; // ISO timestamp
  estimatedDeliveryTime: string;
  deliveryAddress: UserAddress;
  paymentMethod: string;
}

type OrderStatus =
  | 'PLACED'
  | 'ACCEPTED'
  | 'PREPARING'
  | 'PICKED_UP'
  | 'OUT_FOR_DELIVERY'
  | 'DELIVERED'
  | 'CANCELLED';

interface UserAddress {
  id: string;
  label: string; // "Home", "Work", "Other"
  formattedAddress: string;
  coordinates: { lat: number; lng: number };
  landmark?: string;
}

interface SearchSuggestion {
  type: 'dish' | 'restaurant' | 'cuisine';
  id: string;
  name: string;
  metadata?: string; // e.g., "available at 15 restaurants"
}
```

---

### 7.3 Entity Relationships

```
User 1 ──────── * UserAddress       (one user has many saved addresses)
User 1 ──────── * Order             (one user has many orders)

Restaurant 1 ── * MenuCategory      (one restaurant has many categories)
MenuCategory 1─ * MenuItem          (one category has many menu items)

Order * ──────── 1 Restaurant       (each order is from one restaurant)
Order * ──────── 1 UserAddress      (each order has one delivery address)
Order 1 ──────── * CartItem         (one order has many cart items)
CartItem * ───── 1 MenuItem         (cart item references a menu item)

Cart 1 ───────── 1 Restaurant       (cart is locked to one restaurant)
Cart 1 ───────── * CartItem         (cart has many items)
```

*   **Denormalized by default**: Restaurant cards carry all needed display data (no need to fetch user info separately).
*   **Menu is fetched fresh** per restaurant visit (menus change frequently — items may become unavailable during peak hours).
*   **Cart is self-contained**: Each CartItem stores name, price, and customizations locally — not just a reference to a MenuItem. This prevents issues when menu prices change while the user has an active cart.

---

### 7.4 UI Specific Data Models

*   **RestaurantCardViewModel**: Combines Restaurant data with computed fields like `distanceLabel` ("1.2 km"), `deliveryTimeLabel` ("25-30 min"), `priceLabel` ("₹300 for two"), `offerLabel` ("50% OFF up to ₹100").
*   **CartSummary**: Derived from cart store — `totalItems`, `subtotal`, `restaurantName`, used by StickyCartBar and CartIcon.
*   **BillBreakdown**: Computed from cart items + delivery fee + taxes + tip — used by BillDetails component in CartPage.
*   **FilterState**: Current active filters derived from URL search params — used by FilterBar and passed to API calls.
*   **OrderTrackingState**: Combined order status + rider location + ETA — fed by WebSocket updates.

---

## 8. State Management Strategy

### 8.1 State Classification

| State Type | Examples | Where Stored |
|-----------|----------|--------------|
| **Server state** | Restaurant list, menu items, order history, search results | React Query cache (auto-cached, background refetch, stale-while-revalidate) |
| **Global client state** | Current delivery location, user auth info | React Context or Zustand (global store) |
| **Feature state (Cart)** | Cart items, restaurant lock, coupon | Zustand with localStorage persistence |
| **Feature state (Filters)** | Active cuisine, sort order, rating filter | URL search params (derived via useSearchParams) |
| **Feature state (Order Tracking)** | Order status, rider location, ETA | Component-level state (fed by WebSocket) |
| **Local UI state** | Modal open/close, quantity stepper value, dropdown expanded | React useState within the component |
| **Derived state** | Cart total, cart item count, bill breakdown, active category in menu | Computed from store (Zustand selectors / React Query select) |

---

### 8.2 State Ownership

*   **Cart Store (Zustand)** owns cart state globally — accessible from MenuPage (add item), CartPage (view/edit), Header (badge count), and CheckoutPage (place order).
*   **Location State (Context)** owns the current delivery address — consumed by restaurant listing API calls, address display in header, and checkout.
*   **Filter State (URL)** is owned by the URL — FilterBar reads from and writes to searchParams. This means the browser's back/forward buttons naturally handle filter history.
*   **React Query** owns all server/async state — restaurant lists, menus, order history. Components subscribe to query results; cache handles deduplication and freshness.
*   **Order Tracking (Local)** — the WebSocket hook manages tracking state locally within OrderTrackingPage. It doesn't need to be global because tracking is only viewed on one page at a time.

---

### 8.3 Persistence Strategy

| Data | Storage | Reason |
|------|---------|--------|
| **Cart** | Zustand → localStorage | Must survive page refresh; small data; instant read |
| **Last delivery address** | localStorage | Quick restore on next visit; avoid re-prompting Geolocation |
| **Auth token** | HTTP-only cookie | Secure; sent automatically with API requests |
| **Restaurant list cache** | React Query in-memory | Fresh data needed per session; stale after location change |
| **Menu cache** | React Query in-memory (short TTL) | Menus change during peak hours; stale data = bad UX (unavailable items) |
| **Order history** | React Query in-memory | Fetched on demand; no offline requirement |
| **Filter preferences** | URL search params | Shareable; survives page refresh via URL; no storage needed |
| **Search history** | localStorage (last 5 searches) | Convenience for repeat searches; small data |

---

## 9. High Level API Design (Frontend POV)

### 9.1 Required APIs

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/restaurants` | GET | Fetch paginated restaurant list with filters (location, cuisine, rating, sort) |
| `/api/restaurants/:slug` | GET | Fetch restaurant details + full menu |
| `/api/restaurants/:id/menu` | GET | Fetch menu items grouped by category |
| `/api/search/suggest` | GET | Autocomplete suggestions for dishes, restaurants, cuisines |
| `/api/search/results` | GET | Full search results with restaurant cards |
| `/api/cart/sync` | POST | Sync local cart to server (optional) |
| `/api/coupons/validate` | POST | Validate a coupon code and return discount |
| `/api/orders` | POST | Place a new order |
| `/api/orders/:id` | GET | Fetch order details |
| `/api/orders/:id/status` | GET | Fetch current order status (polling fallback) |
| `wss://api/orders/:id/track` | WebSocket | Real-time order tracking (status + rider location) |
| `/api/addresses` | GET | Fetch user's saved addresses |
| `/api/addresses` | POST | Add a new delivery address |
| `/api/geocode/reverse` | GET | Reverse geocode coordinates to address string |

---

### 9.2 Real Time Order Tracking Strategy (Deep Dive)

##### Why WebSocket over SSE or Polling

| Approach | How it Works | Latency | Complexity | Best For |
|----------|-------------|---------|------------|----------|
| **Polling** | Client sends GET every N seconds | High (2-10s delay) | Simple | Low-priority status checks; fallback |
| **SSE** | Server pushes events over HTTP | Low (real-time) | Moderate | One-way updates (status changes only) |
| **WebSocket** | Full-duplex persistent connection | Lowest (real-time) | Higher | Two-way: status updates + rider location + ETA; allows client to send acknowledgments |

**Recommendation: WebSocket** for order tracking because:
1. **Rider location updates** arrive every 3-5 seconds — polling would generate too many HTTP requests.
2. We need **multiple event types** over one connection: status changes, rider location, ETA updates, rider assignment.
3. The client may need to **send acknowledgments** (e.g., "delivery received" confirmation).

##### WebSocket Message Types

```ts
// Server → Client messages
type ServerMessage =
  | { type: 'STATUS_CHANGE'; status: OrderStatus; eta: string; timestamp: string }
  | { type: 'RIDER_ASSIGNED'; rider: { name: string; phone: string; photo: string } }
  | { type: 'RIDER_LOCATION'; lat: number; lng: number; heading: number }
  | { type: 'ETA_UPDATE'; eta: string; reason?: string }
  | { type: 'ORDER_CANCELLED'; reason: string };

// Client → Server messages (optional)
type ClientMessage =
  | { type: 'DELIVERY_CONFIRMED' }
  | { type: 'PING' }; // keep-alive
```

##### Reconnection Strategy

```
WebSocket disconnects
  │
  ├── Wait 1 second → attempt reconnect #1
  │    ├── Success → resume tracking
  │    └── Fail → wait 2 seconds
  │
  ├── Attempt reconnect #2
  │    ├── Success → resume tracking
  │    └── Fail → wait 4 seconds
  │
  ├── Attempt reconnect #3
  │    └── Fail → Switch to polling fallback
  │
  └── Polling mode: GET /api/orders/:id/status every 10 seconds
       └── When WebSocket reconnects, stop polling and resume WebSocket
```

```tsx
function useResilientOrderTracking(orderId: string) {
  const [mode, setMode] = useState<'websocket' | 'polling'>('websocket');
  const wsRef = useRef<WebSocket | null>(null);
  const reconnectAttemptRef = useRef(0);
  const maxReconnectAttempts = 3;

  const connect = useCallback(() => {
    const ws = new WebSocket(
      `wss://api.example.com/orders/${encodeURIComponent(orderId)}/track`
    );

    ws.onopen = () => {
      reconnectAttemptRef.current = 0;
      setMode('websocket');
    };

    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      handleTrackingUpdate(message);
    };

    ws.onclose = () => {
      if (reconnectAttemptRef.current < maxReconnectAttempts) {
        const delay = Math.pow(2, reconnectAttemptRef.current) * 1000;
        reconnectAttemptRef.current += 1;
        setTimeout(connect, delay);
      } else {
        // Fallback to polling
        setMode('polling');
      }
    };

    wsRef.current = ws;
  }, [orderId]);

  // Polling fallback
  useEffect(() => {
    if (mode !== 'polling') return;

    const interval = setInterval(async () => {
      const response = await fetch(`/api/orders/${encodeURIComponent(orderId)}/status`);
      const data = await response.json();
      handleTrackingUpdate({ type: 'STATUS_CHANGE', ...data });
    }, 10000);

    // Keep trying WebSocket in background
    const retryWs = setTimeout(connect, 30000);

    return () => {
      clearInterval(interval);
      clearTimeout(retryWs);
    };
  }, [mode, orderId, connect]);

  useEffect(() => {
    connect();
    return () => wsRef.current?.close();
  }, [connect]);
}
```

---

### 9.3 Request and Response Structure

##### Restaurant List API

```json
// GET /api/restaurants?lat=12.97&lng=77.59&cuisine=pizza&rating=4&sort=deliveryTime&cursor=abc123&limit=20

// Response:
{
  "restaurants": [
    {
      "id": "r_001",
      "name": "Pizza Palace",
      "slug": "pizza-palace-koramangala",
      "imageUrl": "https://cdn.example.com/restaurants/r_001.jpg",
      "blurHash": "data:image/jpeg;base64,/9j/4AAQ...",
      "cuisines": ["Pizza", "Italian", "Pasta"],
      "rating": 4.3,
      "totalRatings": 1250,
      "deliveryTime": 25,
      "priceForTwo": 500,
      "offer": "50% OFF up to ₹100",
      "promoted": false,
      "isOpen": true,
      "distance": 1.2
    }
  ],
  "nextCursor": "def456",
  "hasMore": true,
  "totalCount": 145
}
```

##### Menu API

```json
// GET /api/restaurants/r_001/menu

// Response:
{
  "restaurant": {
    "id": "r_001",
    "name": "Pizza Palace",
    "rating": 4.3,
    "deliveryTime": 25,
    "deliveryFee": 30,
    "bannerUrl": "https://cdn.example.com/restaurants/r_001_banner.jpg"
  },
  "categories": [
    {
      "id": "cat_1",
      "name": "Recommended",
      "items": [
        {
          "id": "item_001",
          "name": "Margherita Pizza",
          "description": "Classic delight with mozzarella cheese and fresh basil",
          "price": 249,
          "imageUrl": "https://cdn.example.com/items/item_001.jpg",
          "isVeg": true,
          "isAvailable": true,
          "isBestseller": true,
          "customizationGroups": [
            {
              "id": "cg_1",
              "title": "Choose Size",
              "required": true,
              "maxSelections": 1,
              "options": [
                { "id": "opt_1", "name": "Regular", "price": 0 },
                { "id": "opt_2", "name": "Medium", "price": 100 },
                { "id": "opt_3", "name": "Large", "price": 200 }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

##### Place Order API

```json
// POST /api/orders

// Request:
{
  "restaurantId": "r_001",
  "items": [
    {
      "menuItemId": "item_001",
      "quantity": 2,
      "customizations": [
        { "groupId": "cg_1", "optionId": "opt_2" }
      ]
    }
  ],
  "addressId": "addr_001",
  "couponCode": "FIRST50",
  "paymentMethod": "upi",
  "tip": 20
}

// Response:
{
  "orderId": "ord_12345",
  "status": "PLACED",
  "estimatedDeliveryTime": "2026-03-17T13:45:00Z",
  "total": 578,
  "paymentStatus": "SUCCESS",
  "trackingUrl": "/orders/ord_12345/track"
}
```

---

### 9.4 Error Handling and Status Codes

| Status Code | Scenario | Frontend Action |
|-------------|----------|-----------------|
| **200** | Success | Process response normally |
| **400** | Invalid request (bad filters, invalid coupon) | Show inline validation error |
| **401** | Authentication expired | Redirect to login; preserve cart in localStorage |
| **403** | Forbidden (e.g., accessing another user's order) | Show "Access denied" page |
| **404** | Restaurant not found / Order not found | Show "Not found" page with link to home |
| **409** | Conflict (e.g., menu item unavailable mid-order) | Show "Some items are no longer available" + prompt to update cart |
| **422** | Validation error (minimum order amount not met) | Show specific validation message near the relevant field |
| **429** | Rate limited | Show "Too many requests, please wait" toast; apply local throttle |
| **500** | Server error | Show generic error with retry button; log to error monitoring |
| **503** | Service unavailable (high traffic during meal rush) | Show "We are experiencing high demand" page with auto-retry |

---

## 10. Caching Strategy

### What to Cache

| Data | Cache Duration | Invalidation |
|------|---------------|-------------|
| **Restaurant list** | 5 min (staleTime) | On location change; on filter change (refetch); background refetch on window focus |
| **Menu items** | 2 min (staleTime) | On restaurant page revisit; items may become unavailable during peak hours |
| **Search suggestions** | 30 seconds | New query replaces old cache |
| **Restaurant images** | Long-lived (CDN) | URL-based versioning (image URL changes on update) |
| **Food item images** | Long-lived (CDN) | URL-based versioning |
| **User addresses** | 10 min | On address add/edit/delete |
| **Order history** | 1 min | On new order placement |

### Where to Cache

| Layer | What | Strategy |
|-------|------|----------|
| **React Query** | All API responses (restaurants, menus, orders) | Stale-while-revalidate; configurable staleTime per query; automatic garbage collection |
| **Browser HTTP cache** | CDN assets (images, fonts, JS/CSS bundles) | `Cache-Control: public, max-age=31536000, immutable` for hashed assets |
| **Service Worker** | App shell (HTML, critical CSS, core JS) | Cache-first for shell; network-first for API calls |
| **localStorage** | Cart, last address, search history, user preferences | Manual read/write; no TTL (app manages cleanup) |
| **CDN edge** | Restaurant images, food images, banners | Edge-cached with long TTL; purged on image update |

### Cache Invalidation

*   **Location change** → Invalidate all restaurant list queries (`queryClient.invalidateQueries(['restaurants'])`).
*   **Cart restaurant conflict** → No cache to invalidate; handled in UI state.
*   **Menu item unavailability** → Server returns `409` on order placement → show prompt to remove unavailable items → re-fetch menu.
*   **Order status** → No caching; always fresh via WebSocket / polling.

---

## 11. CDN and Asset Optimization

*   **Restaurant images**: Served via CDN with multiple size variants.
    *   Card thumbnail: 400x250 (JPEG, ~30KB).
    *   Banner: 800x400 (JPEG, ~80KB).
    *   WebP format with JPEG fallback using `<picture>` or `Accept` header negotiation.
*   **Food item images**: Smaller thumbnails (200x200, ~15KB); many menu items have no image (text-only is fine).
*   **Lazy loading**: All images below the fold use `loading="lazy"`. First 4 restaurant cards use eager loading for LCP.
*   **Blur-up placeholders**: Use `blurHash` (tiny base64 image inline in API response) as CSS `background-image` while full image loads.
*   **Font optimization**: Use `font-display: swap`; preload primary font weight. System font stack as fallback.
*   **JS/CSS bundles**: Content-hashed filenames; `Cache-Control: immutable`; code-split per page route.
*   **Compression**: Brotli (preferred) or gzip on all text assets (HTML, CSS, JS, JSON).
*   **Preconnect**: `<link rel="preconnect">` to CDN domain and Map SDK domain for early connection.

---

## 12. Rendering Strategy

| Page | Strategy | Reason |
|------|----------|--------|
| **Home / Restaurant Listing** | SSR (first batch) + CSR (infinite scroll) | Fast FCP with pre-rendered restaurant cards; SEO for restaurant pages; CSR for dynamic filter changes |
| **Restaurant Menu Page** | SSR (initial menu) + CSR (cart interactions) | SEO for restaurant + menu pages (indexed by Google); fast first paint; CSR for add-to-cart, customization modals |
| **Cart Page** | CSR only | No SEO value; cart is client state; dynamic pricing calculations |
| **Checkout Page** | CSR only | Auth-required; no SEO value; payment SDK loaded client-side |
| **Order Tracking Page** | CSR only | Auth-required; real-time WebSocket data; no SEO value |
| **Search Results** | CSR (no SSR) | Search is highly dynamic; results depend on query + location; no SEO benefit |
| **Static Pages** (About, T&C, Help) | SSG (Static Generation) | Content rarely changes; maximum performance; fully cacheable |

### Hydration Strategy

*   SSR pages send **serialized initial data** (restaurant list, menu) in a `<script>` tag (`window.__INITIAL_DATA__`).
*   React Query is initialized with this data on the client, avoiding a duplicate fetch.
*   Interactive elements (filter chips, add-to-cart buttons) become functional after hydration completes.
*   **Selective hydration** (React 18): Cart-related components and modals hydrate with lower priority; restaurant cards hydrate first.

---

## 13. Cross Cutting Non Functional Concerns

### 13.1 Security

*   **Authentication**: JWT stored in HTTP-only, Secure, SameSite cookie. Refresh token rotation on expiry.
*   **CSRF**: CSRF token sent via a meta tag (server-rendered) and included in all state-changing API requests.
*   **XSS Prevention**: All user-generated content (restaurant names, reviews, addresses) rendered using React's default escaping. No `dangerouslySetInnerHTML` for user data.
*   **Input Validation**: Client-side validation for addresses, phone numbers, coupon codes — but never trusted; server re-validates everything.
*   **Payment Security**: Payment data never touches our frontend. Razorpay/Stripe SDK handles card input via an iframed element. PCI DSS compliance is the payment gateway's responsibility.
*   **Content Security Policy**: Strict CSP headers limiting script sources to own domain + payment SDK + map SDK + analytics.
*   **Rate Limiting**: Frontend applies local debounce on search input (300ms) and filter changes (200ms) to reduce API load.

---

### 13.2 Accessibility

*   **Semantic HTML**: Restaurant cards use `<article>`; menu categories use `<section>` + `<h2>`; filter chips use `role="switch"` + `aria-pressed`; category nav uses `role="tablist"`.
*   **Keyboard Navigation**: Tab through restaurant cards; Enter to open; arrow keys in filter chips; Escape to close modals. Cart quantity stepper is keyboard-accessible.
*   **Screen Reader Support**: 
    *   Restaurant card: "Pizza Palace, 4.3 stars, Pizza Italian Pasta, 25 minutes delivery, 500 rupees for two".
    *   Veg/non-veg indicators: `aria-label="Vegetarian"` / `aria-label="Non-vegetarian"` (not just color).
    *   Cart updates announced via `aria-live="polite"` region.
    *   Order status changes announced via `aria-live="assertive"`.
*   **Focus Management**: When filter changes and list reloads, focus returns to the first restaurant card. When modal opens (customization, address picker), focus is trapped inside the modal. On modal close, focus returns to the trigger element.
*   **Color Contrast**: Veg/non-veg indicators use icons (green dot / red dot) PLUS text labels, not color alone. Rating badge meets WCAG AA contrast ratios. Offer tags have sufficient contrast on image overlays.
*   **Reduced Motion**: `prefers-reduced-motion` disables carousel animations, marker animation on map, and smooth scroll behavior.

---

### 13.3 Performance Optimization

*   **Code Splitting**: Each page route is a lazy chunk. Heavy dependencies (Map SDK, Payment SDK) are dynamically imported only on the pages that need them.
    ```tsx
    const OrderTrackingPage = lazy(() => import('./pages/OrderTrackingPage'));
    const CheckoutPage = lazy(() => import('./pages/CheckoutPage'));
    ```
*   **Bundle Optimization**: Tree-shake map SDK (only import needed modules); dead-code-eliminate unused filter components; analyze bundle size with webpack-bundle-analyzer.
*   **Image Optimization**: WebP with JPEG fallback; responsive `srcSet`; `content-visibility: auto` on restaurant cards; blur-up placeholders.
*   **Network Optimization**: 
    *   `<link rel="preconnect" href="https://cdn.example.com" />` — CDN.
    *   `<link rel="preconnect" href="https://maps.googleapis.com" />` — Map SDK (only on tracking page).
    *   Prefetch menu data on restaurant card hover: `queryClient.prefetchQuery(['menu', id])`.
*   **Performance Metrics**:
    *   FCP < 1.5s (SSR restaurant cards).
    *   LCP < 2.5s (first restaurant card image).
    *   CLS < 0.1 (image dimensions set; skeleton heights match content).
    *   INP < 200ms (filter taps, add-to-cart clicks respond instantly).

---

### 13.4 Observability and Reliability

*   **Error Boundaries**: Wrap each major section (restaurant list, menu, cart, order tracking) in an error boundary. A broken restaurant card doesn't crash the entire listing.
*   **Error Logging**: Unhandled errors and API failures logged to Sentry/Datadog with context (page, user action, request details).
*   **Performance Monitoring**: Web Vitals (FCP, LCP, CLS, INP) reported to analytics. Custom metrics: time-to-first-restaurant-card, menu-load-time, order-placement-duration.
*   **Feature Flags**: Roll out new UI components (redesigned restaurant card, new filter chip) behind feature flags. A/B test cart flow variations.
*   **Analytics Events**:
    *   `restaurant_impression` — restaurant card visible in viewport.
    *   `restaurant_click` — user taps a restaurant card.
    *   `menu_item_added` — item added to cart (with item name, price, restaurant).
    *   `order_placed` — order submitted (with total, item count, coupon used).
    *   `filter_applied` — filter chip toggled (with filter type and value).
    *   `search_performed` — search query submitted.
    *   `order_tracking_opened` — tracking page viewed.
*   **Health Checks**: Frontend pings `/api/health` on app load. If unhealthy, show a "Service temporarily unavailable" banner instead of broken UI.

---

## 14. Edge Cases and Tradeoffs

### Edge Cases

| Scenario | How to Handle |
|---------|---------------|
| **Restaurant closes mid-browse** | API returns `isOpen: false`; show a "Currently Closed" overlay on the card; disable "View Menu". If user is on menu, show toast: "Restaurant is now closed" and disable add-to-cart. |
| **Menu item becomes unavailable after adding to cart** | On cart sync or order placement, API returns `409` with unavailable item IDs. Show prompt: "Butter Chicken is no longer available — remove it?" |
| **User changes delivery address to a different zone** | Restaurant list changes completely. If cart has items from a restaurant out of range, show: "Your cart restaurant doesn't deliver to this address. Clear cart?" |
| **Network failure during order placement** | Do NOT clear the cart. Show error with retry. If payment was charged (gateway callback received), show "Payment received — retrying order..." with automatic retry. |
| **Extremely long menu (200+ items)** | Category navigation helps users jump to sections. Virtual scroll within categories if a single category has 50+ items. Menu search filters items across all categories. |
| **Peak hours (lunchtime surge)** | Delivery times may spike. Show dynamic messaging: "High demand — delivery may take longer." Rate limit API calls with local debounce. |
| **User adds same item with different customizations** | Treat as separate cart items. "Margherita (Medium)" and "Margherita (Large)" are distinct line items in the cart. |
| **Coupon no longer valid at checkout** | Validate coupon on every cart page visit (not just on "Apply"). If invalid, auto-remove with toast: "Coupon FIRST50 has expired." |
| **WebSocket drops during order tracking** | Exponential backoff reconnect (3 attempts). Fall back to polling every 10s. Show "Reconnecting..." indicator. |
| **Rider location jumps erratically (GPS noise)** | Smooth marker movement with animation interpolation (1s ease-out). Ignore coordinates that jump more than 500m in 3 seconds (likely GPS spike). |
| **User refreshes during checkout** | Cart survives via localStorage. Order is not placed until API confirms. Payment gateway handles refresh via callback/redirect. |
| **Zero restaurants found for location** | Show friendly empty state: "No restaurants deliver to this area yet. Try a different address." with address picker CTA. |
| **Slow 3G connection** | Skeleton loading states for every data-dependent section. Compressed image variants. Minimal JS bundle for first paint (SSR handles initial render). |

### Key Tradeoffs

| Decision | Tradeoff |
|----------|----------|
| **SSR for restaurant listing** | Better FCP and SEO, but adds server load and infra complexity. CSR-only would be simpler but slower perceived load on first visit. |
| **URL-synced filters** | Shareable, back-button friendly, SEO-crawlable. But adds complexity vs simple in-memory filter state. |
| **Zustand over Redux** | Smaller bundle, simpler API for cart state. But less ecosystem tooling (DevTools, middleware) compared to Redux Toolkit. |
| **localStorage cart over server cart** | Instant reads, works offline, no API calls. But no cross-device sync; data can be lost if browser storage is cleared. |
| **WebSocket over SSE for tracking** | Bidirectional, supports rider location + status + ETA over one connection. But more complex than SSE; requires sticky sessions or pub/sub backend. |
| **Google Maps over Mapbox** | More familiar to users, better geocoding in India. But more expensive at scale; Mapbox offers better customization and pricing. |
| **Lazy-loaded images over Blur-hash** | Simpler; browser-native `loading="lazy"`. But blur-hash gives a polished loading experience. Worth the extra API payload (20 bytes per item). |
| **Menu fetched fresh vs cached** | Fresh ensures availability accuracy. But adds latency on every restaurant visit. Compromise: 2-min staleTime with background refetch. |

---

## 15. Summary and Future Improvements

### Key Architectural Decisions

1.  **Location-first architecture** — Everything starts with the delivery address. Restaurant listing, menu availability, delivery times, and pricing all depend on location. Location state is established before any data fetch.
2.  **URL-synced filters with IntersectionObserver infinite scroll** — Filters live in URL params for shareability and SEO; sentinel-based scroll loads more restaurants without scroll event overhead.
3.  **Zustand with localStorage persistence for cart** — Lightweight global store that survives page refreshes; single-restaurant constraint enforced in the store logic.
4.  **Scroll-spy category navigation on menu page** — IntersectionObserver tracks which menu category is visible; sticky nav highlights the active category for easy navigation.
5.  **WebSocket for real-time order tracking** — Rider location, order status, and ETA updates stream over a single persistent connection with polling fallback.
6.  **SSR for listing and menu pages** — Fast FCP with server-rendered restaurant cards and menu items; SSR data seeded into React Query to avoid duplicate fetches.
7.  **React Query for server state** — Automatic caching with stale-while-revalidate, background refetch on window focus, and configurable stale times per data type.

### Possible Future Enhancements

*   **Offline cart and browsing**: Cache last-viewed restaurant menus in IndexedDB via Service Worker. Allow cart management offline; place order when network returns.
*   **Predictive prefetching**: Prefetch menu data when user hovers over a restaurant card (desktop) or when a card enters the viewport (mobile). Reduce perceived latency for the menu page.
*   **Web Workers for search**: Offload fuzzy menu search (within a 200+ item menu) to a Web Worker to keep main thread free during scroll-spy and cart animations.
*   **Shared Element Transitions**: Use the View Transitions API to animate restaurant card → menu page transitions (card image expands into banner).
*   **Live order ETA on home page**: If the user has an active order, show a mini tracking bar on the home page (like Uber Eats) without navigating to the tracking page.
*   **Voice ordering**: Use the Web Speech API for accessibility — allow users to add items to cart via voice commands.
*   **Group ordering**: Multiple users can add items to the same cart via unique share links — requires real-time cart sync via WebSocket.
*   **AR menu preview**: Use WebXR to show 3D food previews when users point their camera at a menu item (experimental; long-term vision).

---

### Endpoint Summary

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/restaurants` | GET | Fetch paginated restaurant list with location and filters |
| `/api/restaurants/:slug` | GET | Fetch restaurant details |
| `/api/restaurants/:id/menu` | GET | Fetch menu items grouped by category |
| `/api/search/suggest` | GET | Autocomplete suggestions for search |
| `/api/search/results` | GET | Full search results with restaurant cards |
| `/api/cart/sync` | POST | Sync local cart state to server |
| `/api/coupons/validate` | POST | Validate and apply a coupon code |
| `/api/orders` | POST | Place a new order |
| `/api/orders/:id` | GET | Fetch order details |
| `/api/orders/:id/status` | GET | Fetch order status (polling fallback) |
| `wss://api/orders/:id/track` | WebSocket | Real time order tracking stream |
| `/api/addresses` | GET | Fetch saved delivery addresses |
| `/api/addresses` | POST | Add new delivery address |
| `/api/geocode/reverse` | GET | Reverse geocode coordinates to address |

---

### Complete Order Flow

| Direction | Mechanism | Trigger | Endpoint | Action |
|-----------|-----------|---------|----------|--------|
| Initial Load | REST (SSR + CSR) | On mount | `GET /api/restaurants?lat=...&lng=...&limit=20` | Render first 20 restaurant cards |
| Infinite Scroll | REST | Sentinel intersects viewport | `GET /api/restaurants?cursor=<token>` | Append next batch of restaurants |
| Filter Change | REST | User toggles filter chip | `GET /api/restaurants?cuisine=pizza&rating=4` | Replace list with filtered results |
| Open Menu | REST | User taps restaurant card | `GET /api/restaurants/:id/menu` | Render categorized menu items |
| Add to Cart | Local | User taps ADD button | (Zustand store + localStorage) | Update cart state; show sticky bar |
| Place Order | REST | User confirms checkout | `POST /api/orders` | Submit order; receive orderId |
| Track Order | WebSocket | Order placed | `wss://api/orders/:id/track` | Stream status + rider location |
| Track Fallback | REST (Polling) | WebSocket disconnected | `GET /api/orders/:id/status` | Poll every 10s until reconnect |
| Search | REST | User types in search bar | `GET /api/search/suggest?q=...` | Show autocomplete suggestions |

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design
