# Frontend System Design: Uber Ola Ride Hailing Application

- [Frontend System Design: Uber Ola Ride Hailing Application](#frontend-system-design-uber-ola-ride-hailing-application)
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
    - [5.2 Ride Booking Flow with Map and Real Time Tracking Strategy (Deep Dive)](#52-ride-booking-flow-with-map-and-real-time-tracking-strategy-deep-dive)
      - [5.2.1 Map Centered Booking Interface](#521-map-centered-booking-interface)
      - [5.2.2 Location Input with Place Autocomplete](#522-location-input-with-place-autocomplete)
      - [5.2.3 Ride Type Selection and Fare Estimation](#523-ride-type-selection-and-fare-estimation)
      - [5.2.4 Driver Matching State Machine](#524-driver-matching-state-machine)
      - [5.2.5 Real Time Ride Tracking with Live Map](#525-real-time-ride-tracking-with-live-map)
      - [5.2.6 Route Polyline Rendering on Map](#526-route-polyline-rendering-on-map)
      - [5.2.7 ETA Countdown and Dynamic Updates](#527-eta-countdown-and-dynamic-updates)
      - [5.2.8 Post Ride Flow (Rating, Payment Summary)](#528-post-ride-flow-rating-payment-summary)
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
    - [9.2 Real Time Ride Tracking Strategy (Deep Dive)](#92-real-time-ride-tracking-strategy-deep-dive)
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

*   A ride-hailing web application (like Uber / Ola) that allows users to book rides on demand by specifying a pickup and drop-off location, selecting a vehicle type, and tracking their driver in real time.
*   Target users: commuters, travelers, and anyone needing point-to-point transportation — from daily office commuters to airport travelers.
*   Primary use case: entering a destination, viewing fare estimates for different ride types, requesting a ride, tracking the driver's arrival, and monitoring the trip until drop-off.

---

### 1.2 Key User Personas

*   **Daily Commuter**: Books rides frequently (office, metro station). Uses saved places (Home, Work). Expects quick booking with minimal taps. Cares about ETA accuracy and consistent pricing.
*   **Occasional Rider**: Books rides for specific trips (airport, events). Compares ride types by price and ETA. May schedule rides in advance. Values transparent fare breakdown.
*   **Shared Ride User**: Opts for shared/pool rides to save cost. Tolerates longer routes and additional pickups. Price-sensitive; watches for promotions.

---

### 1.3 Core User Flows (High Level)

*   **Booking a Ride (Primary Flow)**:
    1.  User opens the app → map centers on current location (Geolocation API).
    2.  User enters destination in the search bar → autocomplete suggests places.
    3.  Pickup and drop-off locations are set → route is drawn on the map.
    4.  App shows available ride types (Mini, Sedan, SUV, Pool) with fare estimates and ETAs.
    5.  User selects a ride type → taps "Request Ride".
    6.  App enters matching phase → shows "Finding your driver..." with nearby driver dots on map.
    7.  Driver matched → driver details (name, photo, car, rating, license plate) appear.
    8.  User tracks driver arrival on the map → ETA countdown.
    9.  Driver arrives → "Your ride is here" notification.
    10. Trip starts → live map tracks route progress with updated ETA to destination.
    11. Trip ends → fare summary, payment processed, rating screen.

*   **Scheduling a Ride (Secondary Flow)**:
    1.  User enters pickup and drop-off → selects "Schedule" instead of "Request Now".
    2.  Picks a date and time → confirms scheduled ride.
    3.  Reminder notification before the scheduled time.
    4.  Driver is auto-matched closer to the scheduled time.

*   **Viewing Ride History (Secondary Flow)**:
    1.  User navigates to "Your Trips" → sees past rides listed chronologically.
    2.  Taps a ride → sees route map, fare breakdown, driver details, and receipt.
    3.  Can download or share the receipt.

---

## 2. Requirements

### 2.1 Functional Requirements

*   **Map Interface**:
    *   Display a full-screen interactive map centered on the user's current location.
    *   Show pickup pin (draggable) and drop-off pin once destination is set.
    *   Draw the route polyline between pickup and drop-off.
    *   Show nearby available drivers as animated car icons on the map during the booking phase.
    *   Smooth animated marker movement for driver tracking (interpolated GPS updates).
*   **Location Input**:
    *   Pickup: Auto-detect via Geolocation API; user can edit by dragging the map pin or typing.
    *   Drop-off: Search input with Google Places autocomplete suggestions.
    *   Saved places: Home, Work, and custom saved locations for quick selection.
    *   Recent searches: Show last 5 searched destinations for quick rebooking.
*   **Ride Type Selection**:
    *   Display available ride categories (Mini, Sedan, SUV, Pool/Share, Premium, Auto, Bike).
    *   Each option shows: ride name, estimated fare (range or fixed), ETA for nearest driver, capacity, description.
    *   Highlight surge pricing with a multiplier badge when applicable.
    *   Show fare breakdown on tap (base fare, distance charge, time charge, surge, taxes).
*   **Ride Request and Matching**:
    *   "Request Ride" button sends the booking request.
    *   Matching phase: animated loader + nearby drivers shown on map.
    *   Cancel option available during matching phase.
    *   Driver match: show driver details (name, photo, rating, vehicle info, license plate, OTP for verification).
*   **Real Time Tracking**:
    *   Live driver location on the map (updated every 3-5 seconds via WebSocket).
    *   ETA countdown that updates dynamically.
    *   Route progress: show completed portion of the route in a different color.
    *   Trip status transitions: Driver En Route → Arrived → Trip Started → Trip Ended.
*   **Post Ride**:
    *   Fare summary (base, distance, time, surge, taxes, tip, total).
    *   Star rating (1-5) + optional text feedback.
    *   Tip option (preset amounts or custom).
    *   Receipt (viewable, downloadable, emailable).
*   **Ride History**:
    *   List of past rides with date, route summary, fare, and driver name.
    *   Tap to see full details with map, receipt, and support options.
*   **Promotions and Payments**:
    *   Apply promo codes before requesting a ride.
    *   Select payment method: UPI, credit/debit card, wallet, cash.
    *   Saved payment methods with default selection.

---

### 2.2 Non Functional Requirements

*   **Performance**: FCP < 1.5s; TTI < 3s; map tiles must load within 2s; smooth 60fps map panning and zooming; driver marker animation must be jank-free; ETA updates must reflect in < 500ms.
*   **Scalability**: Support thousands of concurrent ride requests; handle hundreds of driver location updates per second on the tracking screen; map must work with 50+ driver markers during surge.
*   **Availability**: Graceful degradation — if map SDK fails to load, show a text-based booking fallback; if WebSocket drops, fall back to polling; cached ride history available offline.
*   **Security**: Rider phone number masked from driver (server-side); no raw GPS coordinates exposed in client-accessible API responses; payment tokens handled via gateway SDK; CSRF protection on all mutations.
*   **Accessibility**: Keyboard-navigable location inputs and ride type selection; screen reader announces ETA changes and ride status transitions; map has text-based alternative for screen readers; focus management on modal transitions.
*   **Device Support**: Mobile web (primary — 80%+ traffic), desktop web (responsive), low-end Android devices with limited GPU for map rendering.
*   **i18n**: RTL layout support; localized currency formatting; locale-aware distance units (km/miles); multi-language place names and UI labels.

---

## 3. Scope Clarification (Interview Scoping)

### 3.1 In Scope

*   Map-centered booking interface with pickup/drop-off pins and route polyline.
*   Location input with autocomplete (Google Places).
*   Ride type selection with fare estimation.
*   Ride request, driver matching, and real-time tracking via WebSocket.
*   Post-ride flow: fare summary and rating.
*   State management for ride lifecycle (idle → searching → matching → en route → in trip → completed).
*   Performance optimization for map rendering and real-time updates.
*   API design from the frontend perspective.

---

### 3.2 Out of Scope

*   Driver-side application (driver app is a separate product).
*   Backend matching algorithm (supply-demand matching, surge calculation).
*   Payment gateway integration internals (assume third-party SDK).
*   Push notifications (browser/mobile).
*   Ride scheduling backend logic.
*   Admin dashboard (dispatch, ops monitoring).
*   In-app chat or calling (assume deep link to phone dialer).

---

### 3.3 Assumptions

*   User is authenticated; auth token is available via HTTP-only cookie or Authorization header.
*   Fare estimates and surge multipliers are computed server-side; frontend displays the results.
*   Google Maps (or Mapbox) SDK is available for map rendering, geocoding, and route drawing.
*   Driver locations are streamed via WebSocket; the backend handles the fan-out.
*   Payment processing is handled by a third-party SDK (Stripe, Razorpay) with a drop-in UI.

---

## 4. High Level Frontend Architecture

### 4.1 Overall Approach

*   **SPA** (Single Page Application) with client-side routing.
*   **CSR** (Client-Side Rendering) for the entire app — ride-hailing is a highly interactive, map-heavy, auth-gated experience; SEO is not a concern for the booking flow.
*   **SSR** only for static/marketing pages (landing page, city pages, help center) — not for the core booking experience.
*   The map is the **primary UI surface** (occupies 60-70% of the viewport). All other UI panels (location input, ride selection, driver details, trip status) overlay or dock onto the map.
*   Code-split by major flows: Booking (primary chunk), Ride History (lazy), Ride Detail (lazy), Settings/Profile (lazy).

---

### 4.2 Major Architectural Layers

```
┌──────────────────────────────────────────────────────────────────┐
│  UI Layer                                                        │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ MapView (full-screen Google Maps / Mapbox)                 │  │
│  │  ┌──────────┐ ┌──────────┐ ┌─────────────────────────┐    │  │
│  │  │ Pickup   │ │ Dropoff  │ │ DriverMarker[] (animated)│    │  │
│  │  │ Pin      │ │ Pin      │ └─────────────────────────┘    │  │
│  │  └──────────┘ └──────────┘ ┌─────────────────────────┐    │  │
│  │                            │ RoutePolyline            │    │  │
│  │                            └─────────────────────────┘    │  │
│  └────────────────────────────────────────────────────────────┘  │
│  ┌────────────────┐  ┌──────────────────────────────────────┐    │
│  │ LocationInput  │  │ BottomSheet (dynamic panels)         │    │
│  │ Panel          │  │  ┌────────────────────────────────┐  │    │
│  │ (pickup/drop)  │  │  │ RideTypeSelector               │  │    │
│  └────────────────┘  │  │ DriverMatchingView             │  │    │
│                      │  │ DriverDetailsPanel             │  │    │
│                      │  │ TripProgressPanel              │  │    │
│                      │  │ RideSummaryPanel               │  │    │
│                      │  └────────────────────────────────┘  │    │
│                      └──────────────────────────────────────┘    │
├──────────────────────────────────────────────────────────────────┤
│  State Management Layer                                          │
│  (Ride State Machine, Location State, Map State, User Session,   │
│   Payment Methods, Saved Places, Active Ride)                    │
├──────────────────────────────────────────────────────────────────┤
│  API and Data Access Layer                                       │
│  (REST Client, WebSocket Manager for Ride Tracking,              │
│   Request Deduplication, Retry Logic, Fare Estimation Cache)     │
├──────────────────────────────────────────────────────────────────┤
│  Shared / Utility Layer                                          │
│  (Map helpers, Marker animation, Polyline decoder,               │
│   Currency formatting, Distance/ETA formatting,                  │
│   Geolocation wrapper, Debounce/Throttle, Analytics tracker)     │
└──────────────────────────────────────────────────────────────────┘
```

---

### 4.3 External Integrations

*   **Map SDK**: Google Maps JavaScript API / Mapbox GL JS for map rendering, geocoding, Places autocomplete, Directions API (route polylines), and marker management.
*   **CDN**: Serves static assets (JS, CSS, icons, vehicle type images).
*   **Payment Gateway SDK**: Stripe / Razorpay drop-in checkout for payment method management and in-ride payments.
*   **Analytics SDK**: Track booking funnel events (location set, ride type selected, ride requested, ride completed), map interaction metrics, and session data.
*   **Backend Services**: Fare estimation API, ride request API, ride status API, WebSocket endpoint for real-time tracking, ride history API.
*   **Geolocation API**: Browser native API for auto-detecting user location with fallback to IP-based.

---

## 5. Component Design and Modularization

### 5.1 Component Hierarchy

```
App
 ├── Header (minimal — logo, hamburger menu)
 │    ├── HamburgerMenu
 │    │    ├── Your Trips
 │    │    ├── Payment Methods
 │    │    ├── Promotions
 │    │    ├── Settings
 │    │    └── Help
 │    └── ProfileAvatar
 │
 ├── BookingPage (main page — map + overlays)
 │    ├── MapView (full-screen map container)
 │    │    ├── PickupMarker (draggable green pin)
 │    │    ├── DropoffMarker (red pin)
 │    │    ├── RoutePolyline (encoded path between pickup and drop)
 │    │    ├── NearbyDriverMarkers[] (car icons during booking phase)
 │    │    ├── DriverTrackingMarker (animated car during ride)
 │    │    ├── CurrentLocationButton (recenter map on user location)
 │    │    └── MapAttribution
 │    │
 │    ├── LocationInputPanel (top panel overlay)
 │    │    ├── PickupInput (auto-filled with current location)
 │    │    ├── DropoffInput (with Google Places autocomplete)
 │    │    ├── SavedPlacesList (Home, Work, custom)
 │    │    └── RecentSearchesList (last 5 destinations)
 │    │
 │    ├── BottomSheet (draggable panel — content changes by ride state)
 │    │    │
 │    │    ├── [STATE: IDLE] → DestinationPrompt
 │    │    │    └── "Where to?" input
 │    │    │
 │    │    ├── [STATE: DESTINATION_SET] → RideTypeSelector
 │    │    │    ├── RideTypeCard[] (repeated per ride type)
 │    │    │    │    ├── VehicleIcon
 │    │    │    │    ├── RideTypeName + Description
 │    │    │    │    ├── EstimatedFare (range or fixed)
 │    │    │    │    ├── ETA ("3 min away")
 │    │    │    │    ├── SurgeBadge (if active)
 │    │    │    │    └── CapacityLabel ("4 seats")
 │    │    │    ├── FareBreakdownAccordion
 │    │    │    ├── PromoCodeInput
 │    │    │    ├── PaymentMethodSelector
 │    │    │    └── RequestRideButton
 │    │    │
 │    │    ├── [STATE: MATCHING] → DriverMatchingView
 │    │    │    ├── AnimatedSearchingIndicator
 │    │    │    ├── NearbyDriversOnMap (pulsing dots)
 │    │    │    └── CancelButton
 │    │    │
 │    │    ├── [STATE: DRIVER_ASSIGNED] → DriverDetailsPanel
 │    │    │    ├── DriverPhoto + Name + Rating
 │    │    │    ├── VehicleInfo (make, model, color, license plate)
 │    │    │    ├── OTPDisplay (ride verification code)
 │    │    │    ├── ETAToPickup ("Arriving in 4 min")
 │    │    │    ├── ContactDriverButton (call / message)
 │    │    │    ├── ShareRideButton (share trip with contacts)
 │    │    │    └── CancelRideButton
 │    │    │
 │    │    ├── [STATE: DRIVER_ARRIVED] → DriverArrivedPanel
 │    │    │    ├── "Your ride is here" header
 │    │    │    ├── DriverPhoto + VehicleInfo
 │    │    │    ├── OTPDisplay
 │    │    │    └── CancelRideButton
 │    │    │
 │    │    ├── [STATE: IN_TRIP] → TripProgressPanel
 │    │    │    ├── ETAToDestination ("12 min to destination")
 │    │    │    ├── RouteProgressBar
 │    │    │    ├── DriverInfo (compact)
 │    │    │    ├── ShareRideButton
 │    │    │    └── EmergencySOSButton
 │    │    │
 │    │    └── [STATE: COMPLETED] → RideSummaryPanel
 │    │         ├── FareSummary (base, distance, time, surge, total)
 │    │         ├── PaymentStatus ("Paid via UPI")
 │    │         ├── StarRating (1-5)
 │    │         ├── FeedbackInput (optional text)
 │    │         ├── TipSelector (preset amounts + custom)
 │    │         └── DoneButton
 │    │
 │    └── SafetyToolbar (persistent during active ride)
 │         ├── ShareTripButton
 │         ├── EmergencySOSButton
 │         └── TrustedContactsAlert
 │
 ├── RideHistoryPage (lazy loaded)
 │    └── RideHistoryList
 │         └── RideHistoryCard[] (date, route, fare, driver)
 │
 ├── RideDetailPage (lazy loaded)
 │    ├── RideRouteMap (static map with route)
 │    ├── FareBreakdown
 │    ├── DriverDetails
 │    └── ReceiptActions (download, share, support)
 │
 └── SettingsPage (lazy loaded)
      ├── SavedPlaces
      ├── PaymentMethods
      ├── PromoCodes
      └── ProfileSettings
```

---

### 5.2 Ride Booking Flow with Map and Real Time Tracking Strategy (Deep Dive)

The ride booking interface is the **core of the entire application** — a full-screen map with contextual overlays that change based on the ride lifecycle. Getting this right matters because:
*   The map is the **primary UI** — users interact with it 100% of the time during a booking.
*   The bottom sheet must smoothly **transition between states** (idle → ride selection → matching → tracking → summary) without jarring reloads.
*   Driver location updates arrive every **3-5 seconds** and must animate smoothly on the map (no teleporting markers).
*   The app must handle **surge pricing** information, ETA changes, and route updates dynamically.
*   Map performance on **low-end mobile devices** is critical — GPU-intensive map rendering + frequent marker updates can cause jank.

---

#### 5.2.1 Map Centered Booking Interface

##### The Core Layout — Map as the Canvas

Unlike most web apps where the content scrolls, a ride-hailing app uses the **map as the primary canvas**. All other UI elements float on top of or dock to edges of the map.

```
┌──────────────────────────────────────┐
│  [≡]  Pickup: 123 Main Street    [x]│  ← Location input (top)
│  [●]  Drop: Enter destination       │
├──────────────────────────────────────┤
│                                      │
│          🚗   🚗                     │
│                    📍 (pickup pin)   │
│             🚗                       │  ← Full-screen map
│                         🚗           │
│                                      │
│                    🚗                │
│          [◎] (recenter button)       │
├──────────────────────────────────────┤
│  ┌──────────────────────────────────┐│
│  │  ──── (drag handle)              ││
│  │  UberGo    ₹120-150    3 min     ││  ← Bottom sheet
│  │  Sedan     ₹180-220    5 min     ││    (ride types)
│  │  SUV       ₹250-300    7 min     ││
│  │  Pool      ₹80-100     4 min     ││
│  │  ┌──────────────────────────┐    ││
│  │  │  Request UberGo  →      │    ││  ← CTA button
│  │  └──────────────────────────┘    ││
│  └──────────────────────────────────┘│
└──────────────────────────────────────┘
```

##### Map Container Implementation

```tsx
import { useRef, useEffect, useCallback } from 'react';

function MapView({
  pickupLocation,
  dropoffLocation,
  nearbyDrivers,
  trackingDriver,
  routePolyline,
  rideState,
}: MapViewProps) {
  const mapRef = useRef<HTMLDivElement>(null);
  const mapInstanceRef = useRef<google.maps.Map | null>(null);
  const pickupMarkerRef = useRef<google.maps.Marker | null>(null);
  const dropoffMarkerRef = useRef<google.maps.Marker | null>(null);
  const driverMarkersRef = useRef<Map<string, google.maps.Marker>>(new Map());
  const polylineRef = useRef<google.maps.Polyline | null>(null);

  // Initialize map
  useEffect(() => {
    if (!mapRef.current || mapInstanceRef.current) return;

    const map = new google.maps.Map(mapRef.current, {
      center: pickupLocation || { lat: 12.97, lng: 77.59 }, // default to Bangalore
      zoom: 15,
      disableDefaultUI: true,
      zoomControl: true,
      gestureHandling: 'greedy', // single-finger pan on mobile
      styles: CUSTOM_MAP_STYLES, // minimal, clean style
    });

    mapInstanceRef.current = map;
  }, []);

  // Update pickup marker
  useEffect(() => {
    const map = mapInstanceRef.current;
    if (!map || !pickupLocation) return;

    if (!pickupMarkerRef.current) {
      pickupMarkerRef.current = new google.maps.Marker({
        position: pickupLocation,
        map,
        icon: {
          url: '/icons/pickup-pin.svg',
          scaledSize: new google.maps.Size(40, 40),
          anchor: new google.maps.Point(20, 40),
        },
        draggable: rideState === 'IDLE' || rideState === 'DESTINATION_SET',
        title: 'Pickup location',
      });

      // When user drags the pickup pin, update the address
      pickupMarkerRef.current.addListener('dragend', () => {
        const pos = pickupMarkerRef.current!.getPosition()!;
        onPickupDragged({ lat: pos.lat(), lng: pos.lng() });
      });
    } else {
      pickupMarkerRef.current.setPosition(pickupLocation);
    }

    map.panTo(pickupLocation);
  }, [pickupLocation, rideState]);

  // Update dropoff marker
  useEffect(() => {
    const map = mapInstanceRef.current;
    if (!map) return;

    if (dropoffLocation) {
      if (!dropoffMarkerRef.current) {
        dropoffMarkerRef.current = new google.maps.Marker({
          position: dropoffLocation,
          map,
          icon: {
            url: '/icons/dropoff-pin.svg',
            scaledSize: new google.maps.Size(40, 40),
            anchor: new google.maps.Point(20, 40),
          },
          title: 'Drop-off location',
        });
      } else {
        dropoffMarkerRef.current.setPosition(dropoffLocation);
      }

      // Fit map to show both pickup and dropoff
      const bounds = new google.maps.LatLngBounds();
      bounds.extend(pickupLocation!);
      bounds.extend(dropoffLocation);
      map.fitBounds(bounds, { top: 120, bottom: 350, left: 40, right: 40 });
    } else {
      // Remove dropoff marker if destination cleared
      dropoffMarkerRef.current?.setMap(null);
      dropoffMarkerRef.current = null;
    }
  }, [dropoffLocation, pickupLocation]);

  // Draw route polyline
  useEffect(() => {
    const map = mapInstanceRef.current;
    if (!map) return;

    // Clear existing polyline
    polylineRef.current?.setMap(null);

    if (routePolyline) {
      const decodedPath = google.maps.geometry.encoding.decodePath(routePolyline);

      polylineRef.current = new google.maps.Polyline({
        path: decodedPath,
        strokeColor: '#276EF1', // Uber blue
        strokeWeight: 5,
        strokeOpacity: 0.9,
        map,
      });
    }
  }, [routePolyline]);

  // Update nearby driver markers (during booking phase)
  useEffect(() => {
    const map = mapInstanceRef.current;
    if (!map) return;

    if (rideState === 'IDLE' || rideState === 'DESTINATION_SET') {
      // Add/update driver markers
      const currentIds = new Set<string>();

      nearbyDrivers.forEach((driver) => {
        currentIds.add(driver.id);
        const existing = driverMarkersRef.current.get(driver.id);

        if (existing) {
          animateMarkerTo(existing, driver.location);
        } else {
          const marker = new google.maps.Marker({
            position: driver.location,
            map,
            icon: {
              url: '/icons/car-icon.svg',
              scaledSize: new google.maps.Size(28, 28),
              anchor: new google.maps.Point(14, 14),
            },
          });
          driverMarkersRef.current.set(driver.id, marker);
        }
      });

      // Remove drivers that are no longer nearby
      driverMarkersRef.current.forEach((marker, id) => {
        if (!currentIds.has(id)) {
          marker.setMap(null);
          driverMarkersRef.current.delete(id);
        }
      });
    } else {
      // Clear all nearby markers during ride
      driverMarkersRef.current.forEach((m) => m.setMap(null));
      driverMarkersRef.current.clear();
    }
  }, [nearbyDrivers, rideState]);

  return (
    <div className="map-container">
      <div ref={mapRef} className="map" style={{ width: '100%', height: '100%' }} />
      <button
        className="recenter-button"
        onClick={() => recenterMap(mapInstanceRef.current, pickupLocation)}
        aria-label="Recenter map on your location"
      >
        ◎
      </button>
    </div>
  );
}
```

##### Map Style Optimization for Ride Hailing

```ts
const CUSTOM_MAP_STYLES: google.maps.MapTypeStyle[] = [
  // Reduce visual noise — hide POI icons (shops, restaurants)
  { featureType: 'poi', elementType: 'labels.icon', stylers: [{ visibility: 'off' }] },
  // Keep road labels visible (users need to identify streets)
  { featureType: 'road', elementType: 'labels', stylers: [{ visibility: 'on' }] },
  // Simplify transit stations (only major ones)
  { featureType: 'transit.station', stylers: [{ visibility: 'simplified' }] },
  // Slightly desaturate the map to make pins and route stand out
  { featureType: 'all', elementType: 'geometry', stylers: [{ saturation: -20 }] },
];
```

**Why customize map styles?**
*   Default Google Maps is visually noisy — hundreds of POI icons (restaurants, shops, ATMs) that distract from the ride booking experience.
*   By hiding non-essential POIs and desaturating the base map, the pickup/dropoff pins and route polyline **pop visually**.
*   Road labels remain visible because users need to confirm their pickup/dropoff street.

---

#### 5.2.2 Location Input with Place Autocomplete

##### Autocomplete Search with Debounce

```tsx
function LocationInput({
  label,
  placeholder,
  value,
  onChange,
  onSelect,
  savedPlaces,
  recentSearches,
}: LocationInputProps) {
  const [query, setQuery] = useState(value || '');
  const [suggestions, setSuggestions] = useState<PlaceSuggestion[]>([]);
  const [isOpen, setIsOpen] = useState(false);
  const autocompleteService = useRef<google.maps.places.AutocompleteService | null>(null);
  const sessionToken = useRef<google.maps.places.AutocompleteSessionToken | null>(null);

  // Initialize Google Places Autocomplete Service
  useEffect(() => {
    autocompleteService.current = new google.maps.places.AutocompleteService();
    sessionToken.current = new google.maps.places.AutocompleteSessionToken();
  }, []);

  // Debounced autocomplete fetch
  const fetchSuggestions = useDebouncedCallback(async (input: string) => {
    if (!input || input.length < 2 || !autocompleteService.current) {
      setSuggestions([]);
      return;
    }

    autocompleteService.current.getPlacePredictions(
      {
        input,
        sessionToken: sessionToken.current!,
        // Bias results to the user's current location
        locationBias: { center: userLocation, radius: 50000 },
      },
      (predictions, status) => {
        if (status === google.maps.places.PlacesServiceStatus.OK && predictions) {
          setSuggestions(
            predictions.map((p) => ({
              placeId: p.place_id,
              mainText: p.structured_formatting.main_text,
              secondaryText: p.structured_formatting.secondary_text,
              fullText: p.description,
            }))
          );
        }
      }
    );
  }, 300);

  const handleInputChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const val = e.target.value;
    setQuery(val);
    onChange(val);
    fetchSuggestions(val);
    setIsOpen(true);
  };

  const handleSelect = async (suggestion: PlaceSuggestion) => {
    setQuery(suggestion.mainText);
    setIsOpen(false);

    // Geocode the selected place to get coordinates
    const geocoder = new google.maps.Geocoder();
    geocoder.geocode({ placeId: suggestion.placeId }, (results, status) => {
      if (status === 'OK' && results && results[0]) {
        const loc = results[0].geometry.location;
        onSelect({
          placeId: suggestion.placeId,
          address: suggestion.fullText,
          coordinates: { lat: loc.lat(), lng: loc.lng() },
        });
        // Reset session token after selection (billing optimization)
        sessionToken.current = new google.maps.places.AutocompleteSessionToken();
      }
    });
  };

  return (
    <div className="location-input" role="combobox" aria-expanded={isOpen} aria-haspopup="listbox">
      <label className="input-label">{label}</label>
      <input
        type="text"
        value={query}
        onChange={handleInputChange}
        onFocus={() => setIsOpen(true)}
        placeholder={placeholder}
        className="location-text-input"
        role="searchbox"
        aria-autocomplete="list"
        aria-controls="suggestions-list"
      />

      {isOpen && (
        <div id="suggestions-list" className="suggestions-dropdown" role="listbox">
          {/* Saved places */}
          {query.length === 0 && savedPlaces.length > 0 && (
            <div className="suggestions-section">
              <h4 className="section-label">Saved Places</h4>
              {savedPlaces.map((place) => (
                <button
                  key={place.id}
                  className="suggestion-item saved"
                  role="option"
                  onClick={() => onSelect(place)}
                >
                  <span className="icon">{place.label === 'Home' ? '🏠' : '🏢'}</span>
                  <div>
                    <span className="main-text">{place.label}</span>
                    <span className="secondary-text">{place.address}</span>
                  </div>
                </button>
              ))}
            </div>
          )}

          {/* Recent searches */}
          {query.length === 0 && recentSearches.length > 0 && (
            <div className="suggestions-section">
              <h4 className="section-label">Recent</h4>
              {recentSearches.map((search) => (
                <button
                  key={search.placeId}
                  className="suggestion-item recent"
                  role="option"
                  onClick={() => onSelect(search)}
                >
                  <span className="icon">🕐</span>
                  <div>
                    <span className="main-text">{search.address}</span>
                  </div>
                </button>
              ))}
            </div>
          )}

          {/* Autocomplete suggestions */}
          {query.length >= 2 && suggestions.map((suggestion) => (
            <button
              key={suggestion.placeId}
              className="suggestion-item"
              role="option"
              onClick={() => handleSelect(suggestion)}
            >
              <span className="icon">📍</span>
              <div>
                <span className="main-text">{suggestion.mainText}</span>
                <span className="secondary-text">{suggestion.secondaryText}</span>
              </div>
            </button>
          ))}

          {/* No results */}
          {query.length >= 2 && suggestions.length === 0 && (
            <div className="no-results">No places found for "{query}"</div>
          )}
        </div>
      )}
    </div>
  );
}
```

##### Why Google Places Session Tokens Matter

| Without Session Token | With Session Token |
|-----------------------|--------------------|
| Each keystroke triggers a separate API call, billed individually | All keystrokes + final selection count as one "session" |
| User types "air" → "airp" → "airpo" → "airport" = **4 billed requests** + 1 place details = **5 API calls** | Same typing = **1 billed session** (autocomplete + details together) |
| Cost adds up fast at scale | Significant cost savings (60-80% reduction) |

**The session token is created on focus** and **reset after selection** — this groups all the autocomplete queries + the final geocode into one billing session.

---

#### 5.2.3 Ride Type Selection and Fare Estimation

##### Ride Type Selector with Fare Estimates

Once the user has set both pickup and drop-off, the app fetches fare estimates for all ride types and displays them in a scrollable bottom sheet.

```
┌──────────────────────────────────────┐
│  ━━━━ (drag handle)                  │
│                                      │
│  Choose a ride                       │
│  ┌──────────────────────────────────┐│
│  │ 🚗 UberGo              ₹120-150 ││  ← selected (highlighted)
│  │    Affordable, everyday rides     ││
│  │    3 min away · 4 seats          ││
│  ├──────────────────────────────────┤│
│  │ 🚙 Sedan               ₹180-220 ││
│  │    Comfortable sedans             ││
│  │    5 min away · 4 seats          ││
│  ├──────────────────────────────────┤│
│  │ 🚐 SUV         ⚡1.5x  ₹350-420 ││  ← surge indicator
│  │    Extra space for groups         ││
│  │    7 min away · 6 seats          ││
│  ├──────────────────────────────────┤│
│  │ 👥 Pool                  ₹80-100 ││
│  │    Share your ride, save more     ││
│  │    4 min away · 2 seats          ││
│  └──────────────────────────────────┘│
│                                      │
│  💳 UPI ****1234        [Change]     │
│  🏷️ Apply promo                      │
│                                      │
│  ┌──────────────────────────────────┐│
│  │     Request UberGo →             ││  ← CTA
│  └──────────────────────────────────┘│
└──────────────────────────────────────┘
```

##### Implementation

```tsx
function RideTypeSelector({
  fareEstimates,
  selectedType,
  onSelectType,
  onRequestRide,
  paymentMethod,
  onChangePayment,
}: RideTypeSelectorProps) {
  return (
    <div className="ride-type-selector">
      <h2 className="panel-title">Choose a ride</h2>

      <div className="ride-type-list" role="radiogroup" aria-label="Available ride types">
        {fareEstimates.map((estimate) => (
          <button
            key={estimate.rideType}
            className={`ride-type-card ${selectedType === estimate.rideType ? 'selected' : ''}`}
            onClick={() => onSelectType(estimate.rideType)}
            role="radio"
            aria-checked={selectedType === estimate.rideType}
            disabled={!estimate.available}
          >
            <img
              src={estimate.vehicleIcon}
              alt={estimate.displayName}
              className="vehicle-icon"
              width="60"
              height="40"
            />

            <div className="ride-info">
              <div className="ride-header">
                <span className="ride-name">{estimate.displayName}</span>
                {estimate.surgeMultiplier > 1 && (
                  <span className="surge-badge" aria-label={`Surge pricing ${estimate.surgeMultiplier}x`}>
                    ⚡{estimate.surgeMultiplier}x
                  </span>
                )}
              </div>
              <span className="ride-description">{estimate.description}</span>
              <span className="ride-meta">
                {estimate.etaMinutes} min away · {estimate.capacity} seats
              </span>
            </div>

            <div className="fare-estimate">
              <span className="fare-range">
                {formatCurrency(estimate.fareRange.min)}–{formatCurrency(estimate.fareRange.max)}
              </span>
            </div>
          </button>
        ))}
      </div>

      {/* Payment and promo */}
      <div className="booking-options">
        <button className="payment-selector" onClick={onChangePayment}>
          <span className="payment-icon">💳</span>
          <span>{paymentMethod.label}</span>
          <span className="change-link">Change</span>
        </button>
        <PromoCodeInput />
      </div>

      {/* Request button */}
      <button
        className="request-ride-button"
        onClick={onRequestRide}
        disabled={!selectedType}
      >
        Request {fareEstimates.find((e) => e.rideType === selectedType)?.displayName || 'Ride'}
      </button>
    </div>
  );
}
```

##### Fare Breakdown Accordion

```tsx
function FareBreakdownAccordion({ estimate }: { estimate: FareEstimate }) {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div className="fare-breakdown">
      <button
        className="fare-breakdown-toggle"
        onClick={() => setIsOpen(!isOpen)}
        aria-expanded={isOpen}
      >
        Fare breakdown {isOpen ? '▲' : '▼'}
      </button>

      {isOpen && (
        <div className="breakdown-details" role="region" aria-label="Fare breakdown">
          <div className="breakdown-row">
            <span>Base fare</span>
            <span>{formatCurrency(estimate.breakdown.baseFare)}</span>
          </div>
          <div className="breakdown-row">
            <span>Distance ({estimate.breakdown.distanceKm} km)</span>
            <span>{formatCurrency(estimate.breakdown.distanceCharge)}</span>
          </div>
          <div className="breakdown-row">
            <span>Time ({estimate.breakdown.durationMin} min)</span>
            <span>{formatCurrency(estimate.breakdown.timeCharge)}</span>
          </div>
          {estimate.surgeMultiplier > 1 && (
            <div className="breakdown-row surge">
              <span>Surge ({estimate.surgeMultiplier}x)</span>
              <span>+{formatCurrency(estimate.breakdown.surgeCharge)}</span>
            </div>
          )}
          <div className="breakdown-row">
            <span>Taxes and fees</span>
            <span>{formatCurrency(estimate.breakdown.taxes)}</span>
          </div>
          <div className="breakdown-row total">
            <span>Total estimate</span>
            <span>{formatCurrency(estimate.fareRange.min)}–{formatCurrency(estimate.fareRange.max)}</span>
          </div>
        </div>
      )}
    </div>
  );
}
```

---

#### 5.2.4 Driver Matching State Machine

After the user taps "Request Ride", the app enters a **matching phase** — a loading state where the backend finds a suitable driver. This is one of the most critical UX moments: the user is waiting and uncertain.

##### Ride State Machine

```
                    ┌──────────┐
                    │   IDLE   │ ← user opens app; map centered on current location
                    └────┬─────┘
                         │ user enters destination
                         ▼
              ┌───────────────────┐
              │ DESTINATION_SET   │ ← route drawn; fare estimates shown
              └────┬──────────┬──┘
                   │          │
          Request  │          │ clear destination
          Ride     │          │
                   ▼          ▼
          ┌──────────────┐  (back to IDLE)
          │   MATCHING   │ ← "Finding your driver..."
          └──┬───────┬───┘
             │       │
      matched│       │ no driver / timeout / user cancel
             │       │
             ▼       ▼
    ┌─────────────┐  (back to DESTINATION_SET with message)
    │ DRIVER      │
    │ ASSIGNED    │ ← driver details shown; tracking starts
    └──────┬──────┘
           │ driver arrives at pickup
           ▼
    ┌──────────────┐
    │ DRIVER       │ ← "Your ride is here"
    │ ARRIVED      │
    └──────┬───────┘
           │ trip starts (OTP verified)
           ▼
    ┌──────────────┐
    │  IN_TRIP     │ ← live tracking to destination
    └──────┬───────┘
           │ trip ends
           ▼
    ┌──────────────┐
    │  COMPLETED   │ ← fare summary + rating
    └──────┬───────┘
           │ user dismisses
           ▼
        (back to IDLE)

At any point during MATCHING / DRIVER_ASSIGNED / DRIVER_ARRIVED:
  ──→ CANCELLED (by rider, driver, or system)
  ──→ back to DESTINATION_SET with appropriate message
```

##### State Machine Implementation (Zustand)

```tsx
import { create } from 'zustand';

type RideState =
  | 'IDLE'
  | 'DESTINATION_SET'
  | 'MATCHING'
  | 'DRIVER_ASSIGNED'
  | 'DRIVER_ARRIVED'
  | 'IN_TRIP'
  | 'COMPLETED'
  | 'CANCELLED';

interface RideStore {
  state: RideState;
  pickup: Location | null;
  dropoff: Location | null;
  selectedRideType: string | null;
  fareEstimates: FareEstimate[];
  routePolyline: string | null;
  driver: DriverInfo | null;
  driverLocation: LatLng | null;
  tripOTP: string | null;
  etaMinutes: number | null;
  rideSummary: RideSummary | null;
  cancellationReason: string | null;

  // Actions
  setPickup: (location: Location) => void;
  setDropoff: (location: Location) => void;
  setFareEstimates: (estimates: FareEstimate[], polyline: string) => void;
  selectRideType: (type: string) => void;
  requestRide: () => void;
  driverMatched: (driver: DriverInfo, otp: string) => void;
  driverArrived: () => void;
  tripStarted: () => void;
  tripCompleted: (summary: RideSummary) => void;
  rideCancelled: (reason: string) => void;
  updateDriverLocation: (location: LatLng) => void;
  updateETA: (minutes: number) => void;
  resetRide: () => void;
}

const useRideStore = create<RideStore>((set) => ({
  state: 'IDLE',
  pickup: null,
  dropoff: null,
  selectedRideType: null,
  fareEstimates: [],
  routePolyline: null,
  driver: null,
  driverLocation: null,
  tripOTP: null,
  etaMinutes: null,
  rideSummary: null,
  cancellationReason: null,

  setPickup: (location) =>
    set({ pickup: location }),

  setDropoff: (location) =>
    set({ dropoff: location, state: 'DESTINATION_SET' }),

  setFareEstimates: (estimates, polyline) =>
    set({ fareEstimates: estimates, routePolyline: polyline }),

  selectRideType: (type) =>
    set({ selectedRideType: type }),

  requestRide: () =>
    set({ state: 'MATCHING' }),

  driverMatched: (driver, otp) =>
    set({
      state: 'DRIVER_ASSIGNED',
      driver,
      tripOTP: otp,
      driverLocation: driver.location,
    }),

  driverArrived: () =>
    set({ state: 'DRIVER_ARRIVED' }),

  tripStarted: () =>
    set({ state: 'IN_TRIP' }),

  tripCompleted: (summary) =>
    set({ state: 'COMPLETED', rideSummary: summary }),

  rideCancelled: (reason) =>
    set({ state: 'CANCELLED', cancellationReason: reason }),

  updateDriverLocation: (location) =>
    set({ driverLocation: location }),

  updateETA: (minutes) =>
    set({ etaMinutes: minutes }),

  resetRide: () =>
    set({
      state: 'IDLE',
      dropoff: null,
      selectedRideType: null,
      fareEstimates: [],
      routePolyline: null,
      driver: null,
      driverLocation: null,
      tripOTP: null,
      etaMinutes: null,
      rideSummary: null,
      cancellationReason: null,
    }),
}));
```

##### Matching Phase UI

```tsx
function DriverMatchingView({ onCancel }: { onCancel: () => void }) {
  const [elapsedSeconds, setElapsedSeconds] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      setElapsedSeconds((prev) => prev + 1);
    }, 1000);
    return () => clearInterval(interval);
  }, []);

  return (
    <div className="matching-view" aria-live="polite">
      <div className="matching-animation">
        {/* Pulsing concentric circles animation */}
        <div className="pulse-ring" />
        <div className="pulse-ring delay-1" />
        <div className="pulse-ring delay-2" />
        <img src="/icons/car-searching.svg" alt="" className="searching-car" />
      </div>

      <h2 className="matching-title">Finding your driver...</h2>
      <p className="matching-subtitle">
        Looking for nearby drivers ({elapsedSeconds}s)
      </p>

      <button className="cancel-button" onClick={onCancel}>
        Cancel
      </button>
    </div>
  );
}
```

**Matching timeout handling:**

| Scenario | Duration | Action |
|----------|----------|--------|
| Normal match | 5-30 seconds | Driver found → transition to DRIVER_ASSIGNED |
| Slow match | 30-60 seconds | Show "Taking longer than usual. Keep waiting?" prompt |
| No match | > 60 seconds | Show "No drivers available nearby. Try again or change ride type." → back to DESTINATION_SET |
| Surge acceptance | Pre-request | If surge is active, show surge confirmation before entering MATCHING phase |

---

#### 5.2.5 Real Time Ride Tracking with Live Map

Once a driver is assigned, the app enters real-time tracking mode. The driver's location updates every 3-5 seconds via WebSocket, and the map marker must move smoothly.

##### WebSocket Based Ride Tracking

```tsx
function useRideTracking(rideId: string | null) {
  const {
    updateDriverLocation,
    updateETA,
    driverArrived,
    tripStarted,
    tripCompleted,
    rideCancelled,
  } = useRideStore();

  useEffect(() => {
    if (!rideId) return;

    const ws = new WebSocket(
      `wss://api.example.com/rides/${encodeURIComponent(rideId)}/track`
    );

    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);

      switch (message.type) {
        case 'DRIVER_LOCATION':
          updateDriverLocation({
            lat: message.lat,
            lng: message.lng,
            heading: message.heading,
          });
          break;

        case 'ETA_UPDATE':
          updateETA(message.etaMinutes);
          break;

        case 'DRIVER_ARRIVED':
          driverArrived();
          break;

        case 'TRIP_STARTED':
          tripStarted();
          break;

        case 'TRIP_COMPLETED':
          tripCompleted(message.summary);
          break;

        case 'RIDE_CANCELLED':
          rideCancelled(message.reason);
          break;

        case 'ROUTE_UPDATE':
          // Route recalculated (e.g., driver took a different path)
          useRideStore.getState().setFareEstimates(
            useRideStore.getState().fareEstimates,
            message.updatedPolyline
          );
          break;
      }
    };

    ws.onclose = () => {
      // Reconnect with exponential backoff
      reconnectWithBackoff(rideId);
    };

    return () => ws.close();
  }, [rideId]);
}
```

##### Smooth Marker Animation for Driver Tracking

Drivers send GPS updates every 3-5 seconds. Simply setting the marker position to each new coordinate causes the marker to "teleport" — visually jarring. Instead, we **interpolate** the marker position between the old and new coordinates over 1-2 seconds.

```tsx
function animateMarkerTo(
  marker: google.maps.Marker,
  newPosition: { lat: number; lng: number; heading?: number },
  duration: number = 1500
) {
  const startPosition = marker.getPosition()!;
  const endLat = newPosition.lat;
  const endLng = newPosition.lng;
  const startTime = performance.now();

  function step(currentTime: number) {
    const elapsed = currentTime - startTime;
    const progress = Math.min(elapsed / duration, 1);

    // Ease-out cubic for natural deceleration
    const eased = 1 - Math.pow(1 - progress, 3);

    const lat = startPosition.lat() + (endLat - startPosition.lat()) * eased;
    const lng = startPosition.lng() + (endLng - startPosition.lng()) * eased;

    marker.setPosition({ lat, lng });

    // Rotate marker icon to match heading (direction of travel)
    if (newPosition.heading !== undefined) {
      const icon = marker.getIcon() as google.maps.Icon;
      if (icon) {
        marker.setIcon({
          ...icon,
          rotation: newPosition.heading,
        });
      }
    }

    if (progress < 1) {
      requestAnimationFrame(step);
    }
  }

  requestAnimationFrame(step);
}
```

##### Why Interpolation Matters

| Approach | Visual Result | UX |
|----------|--------------|-----|
| **Direct setPosition** | Marker teleports to new location every 3-5 seconds | Jarring; looks broken; hard to follow |
| **Linear interpolation** | Marker slides smoothly at constant speed | Better, but stops abruptly before next update |
| **Ease-out interpolation** | Marker starts fast and decelerates — creates a natural "gliding" motion | Feels like a real car moving; the deceleration at the end smoothly transitions to the next animation cycle |

##### Map Camera Following Strategy

During tracking, the map camera should follow the driver but **not fight the user** if they pan away.

```tsx
function useMapCameraFollow(
  map: google.maps.Map | null,
  driverLocation: LatLng | null,
  isUserInteracting: boolean
) {
  const autoFollowRef = useRef(true);
  const timeoutRef = useRef<NodeJS.Timeout | null>(null);

  // When user manually pans the map, disable auto-follow temporarily
  useEffect(() => {
    if (!map) return;

    const listener = map.addListener('dragstart', () => {
      autoFollowRef.current = false;
      // Re-enable auto-follow after 10 seconds of no interaction
      if (timeoutRef.current) clearTimeout(timeoutRef.current);
      timeoutRef.current = setTimeout(() => {
        autoFollowRef.current = true;
      }, 10000);
    });

    return () => google.maps.event.removeListener(listener);
  }, [map]);

  // Follow driver when auto-follow is enabled
  useEffect(() => {
    if (map && driverLocation && autoFollowRef.current) {
      map.panTo(driverLocation);
    }
  }, [map, driverLocation]);
}
```

**Why this pattern?**
*   If the map always forcefully re-centers on the driver, the user can never look around (e.g., check where the restaurant is, zoom out to see the full route).
*   If the map never re-centers, the driver marker can move off-screen and the user loses track.
*   **Auto-follow with user override** is the sweet spot: follow by default, pause when user pans, resume after 10 seconds of inactivity.

---

#### 5.2.6 Route Polyline Rendering on Map

##### Drawing and Updating the Route

The backend returns the route as a Google Maps [encoded polyline](https://developers.google.com/maps/documentation/utilities/polylinealgorithm) string. The frontend decodes it and draws it on the map.

```tsx
function useRoutePolyline(
  map: google.maps.Map | null,
  polylineEncoded: string | null,
  driverLocation: LatLng | null,
  rideState: RideState
) {
  const remainingPolylineRef = useRef<google.maps.Polyline | null>(null);
  const completedPolylineRef = useRef<google.maps.Polyline | null>(null);

  useEffect(() => {
    if (!map || !polylineEncoded) return;

    // Clear previous polylines
    remainingPolylineRef.current?.setMap(null);
    completedPolylineRef.current?.setMap(null);

    const fullPath = google.maps.geometry.encoding.decodePath(polylineEncoded);

    if (rideState === 'IN_TRIP' && driverLocation) {
      // Split the path into completed and remaining portions
      const { completedPath, remainingPath } = splitPathAtPoint(fullPath, driverLocation);

      // Completed portion (greyed out)
      completedPolylineRef.current = new google.maps.Polyline({
        path: completedPath,
        strokeColor: '#AAAAAA',
        strokeWeight: 4,
        strokeOpacity: 0.6,
        map,
      });

      // Remaining portion (bold blue)
      remainingPolylineRef.current = new google.maps.Polyline({
        path: remainingPath,
        strokeColor: '#276EF1',
        strokeWeight: 5,
        strokeOpacity: 0.9,
        map,
      });
    } else {
      // Before trip starts, show the full route
      remainingPolylineRef.current = new google.maps.Polyline({
        path: fullPath,
        strokeColor: '#276EF1',
        strokeWeight: 5,
        strokeOpacity: 0.9,
        map,
      });
    }

    return () => {
      remainingPolylineRef.current?.setMap(null);
      completedPolylineRef.current?.setMap(null);
    };
  }, [map, polylineEncoded, driverLocation, rideState]);
}

/**
 * Splits the route path into completed and remaining segments
 * based on the driver's current location.
 */
function splitPathAtPoint(
  fullPath: google.maps.LatLng[],
  currentPoint: LatLng
): { completedPath: google.maps.LatLng[]; remainingPath: google.maps.LatLng[] } {
  let closestIndex = 0;
  let closestDistance = Infinity;

  // Find the closest point on the path to the driver's current location
  fullPath.forEach((point, index) => {
    const dist = google.maps.geometry.spherical.computeDistanceBetween(
      point,
      new google.maps.LatLng(currentPoint.lat, currentPoint.lng)
    );
    if (dist < closestDistance) {
      closestDistance = dist;
      closestIndex = index;
    }
  });

  return {
    completedPath: fullPath.slice(0, closestIndex + 1),
    remainingPath: fullPath.slice(closestIndex),
  };
}
```

##### Route Visualization by Ride State

| Ride State | Polyline Visualization |
|-----------|----------------------|
| **DESTINATION_SET** | Full route in blue — pickup to drop-off |
| **DRIVER_ASSIGNED** | Two polylines: gray dashed (driver to pickup) + blue solid (pickup to drop-off) |
| **DRIVER_ARRIVED** | Only blue solid (pickup to drop-off) |
| **IN_TRIP** | Gray (completed portion) + blue (remaining portion), split at driver's current position |
| **COMPLETED** | Full route in gray (historical view) |

---

#### 5.2.7 ETA Countdown and Dynamic Updates

##### ETA Display with Live Updates

```tsx
function ETADisplay({ etaMinutes, label }: { etaMinutes: number | null; label: string }) {
  const [displayEta, setDisplayEta] = useState(etaMinutes);
  const lastServerEta = useRef(etaMinutes);
  const countdownRef = useRef<NodeJS.Timeout | null>(null);

  // Update when server sends a new ETA
  useEffect(() => {
    if (etaMinutes !== null) {
      lastServerEta.current = etaMinutes;
      setDisplayEta(etaMinutes);
    }
  }, [etaMinutes]);

  // Client-side countdown between server updates
  useEffect(() => {
    countdownRef.current = setInterval(() => {
      setDisplayEta((prev) => {
        if (prev === null || prev <= 0) return prev;
        return Math.max(0, prev - 1 / 60); // decrease by 1 second (1/60 of a minute)
      });
    }, 1000);

    return () => {
      if (countdownRef.current) clearInterval(countdownRef.current);
    };
  }, []);

  if (displayEta === null) return null;

  const minutes = Math.ceil(displayEta);

  return (
    <div className="eta-display" aria-live="polite" aria-atomic="true">
      <span className="eta-value">{minutes}</span>
      <span className="eta-unit">min</span>
      <span className="eta-label">{label}</span>
    </div>
  );
}
```

##### Why Client Side Countdown Between Server Updates

The server sends ETA updates every 10-30 seconds. Between updates, the displayed ETA would stay frozen (e.g., "5 min" for 30 seconds, then suddenly jump to "4 min"). This feels laggy.

By running a client-side countdown (ticking 1 second at a time), the ETA **feels dynamic and accurate**, even between server updates. When a server update arrives, it corrects any drift (e.g., if traffic changed, the server ETA might increase instead of decrease).

| Approach | Behavior | UX |
|----------|----------|-----|
| Server ETA only | "5 min" → (frozen 30s) → "4 min" | Feels frozen; user wonders if app is working |
| Client countdown only | Decrements every second; no server correction | Drifts from reality; shows "1 min" when actual ETA is 3 min |
| **Client countdown + server correction** | Smooth countdown; server corrects drift every 10-30s | Best of both — feels live and stays accurate |

---

#### 5.2.8 Post Ride Flow (Rating, Payment Summary)

##### Ride Summary and Rating

```tsx
function RideSummaryPanel({ summary, onSubmitRating, onDone }: RideSummaryProps) {
  const [rating, setRating] = useState<number>(0);
  const [feedback, setFeedback] = useState('');
  const [tipAmount, setTipAmount] = useState<number>(0);

  const handleSubmit = () => {
    onSubmitRating({
      rating,
      feedback: feedback.trim() || undefined,
      tipAmount: tipAmount > 0 ? tipAmount : undefined,
    });
    onDone();
  };

  return (
    <div className="ride-summary">
      <h2>Trip Complete</h2>

      {/* Route summary */}
      <div className="route-summary">
        <div className="route-point">
          <span className="dot green" />
          <span>{summary.pickupAddress}</span>
        </div>
        <div className="route-point">
          <span className="dot red" />
          <span>{summary.dropoffAddress}</span>
        </div>
      </div>

      {/* Fare breakdown */}
      <div className="fare-summary">
        <div className="fare-row"><span>Base fare</span><span>{formatCurrency(summary.baseFare)}</span></div>
        <div className="fare-row"><span>Distance ({summary.distanceKm} km)</span><span>{formatCurrency(summary.distanceCharge)}</span></div>
        <div className="fare-row"><span>Time ({summary.durationMin} min)</span><span>{formatCurrency(summary.timeCharge)}</span></div>
        {summary.surgeCharge > 0 && (
          <div className="fare-row surge"><span>Surge</span><span>+{formatCurrency(summary.surgeCharge)}</span></div>
        )}
        <div className="fare-row"><span>Taxes and fees</span><span>{formatCurrency(summary.taxes)}</span></div>
        {summary.promoDiscount > 0 && (
          <div className="fare-row discount"><span>Promo discount</span><span>-{formatCurrency(summary.promoDiscount)}</span></div>
        )}
        <div className="fare-row total"><span>Total</span><span>{formatCurrency(summary.total)}</span></div>
        <div className="payment-method">Paid via {summary.paymentMethod}</div>
      </div>

      {/* Rating */}
      <div className="rating-section">
        <h3>Rate your ride with {summary.driverName}</h3>
        <div className="star-rating" role="radiogroup" aria-label="Rate your ride">
          {[1, 2, 3, 4, 5].map((star) => (
            <button
              key={star}
              className={`star ${star <= rating ? 'filled' : ''}`}
              onClick={() => setRating(star)}
              role="radio"
              aria-checked={star === rating}
              aria-label={`${star} star${star > 1 ? 's' : ''}`}
            >
              ★
            </button>
          ))}
        </div>
        <textarea
          className="feedback-input"
          placeholder="Any additional feedback? (optional)"
          value={feedback}
          onChange={(e) => setFeedback(e.target.value)}
          maxLength={500}
          rows={2}
        />
      </div>

      {/* Tip */}
      <div className="tip-section">
        <h3>Add a tip for {summary.driverName}</h3>
        <div className="tip-options">
          {[20, 30, 50].map((amount) => (
            <button
              key={amount}
              className={`tip-chip ${tipAmount === amount ? 'selected' : ''}`}
              onClick={() => setTipAmount(tipAmount === amount ? 0 : amount)}
            >
              {formatCurrency(amount)}
            </button>
          ))}
          <button
            className={`tip-chip ${![0, 20, 30, 50].includes(tipAmount) ? 'selected' : ''}`}
            onClick={() => {
              const custom = prompt('Enter tip amount');
              if (custom && !isNaN(Number(custom))) setTipAmount(Number(custom));
            }}
          >
            Custom
          </button>
        </div>
      </div>

      <button className="done-button" onClick={handleSubmit}>
        Done
      </button>
    </div>
  );
}
```

---

#### 5.2.9 Decision Matrix

| Scenario | Strategy | Tool / Approach | Notes |
|----------|----------|----------------|-------|
| **Map rendering** | Google Maps JS API or Mapbox GL JS | SDK loaded async; custom styles for minimal visual noise | Lazy-load map SDK; show skeleton during load |
| **Marker animation** | Ease-out cubic interpolation via requestAnimationFrame | Custom `animateMarkerTo` utility | 1.5s animation duration; match heading/rotation |
| **Real-time tracking** | WebSocket with exponential backoff; polling fallback | `useRideTracking` hook | Reconnect up to 3 times; then poll every 10s |
| **Location autocomplete** | Google Places API with session tokens and debounce | `useDebouncedCallback(300ms)` | Session tokens reduce billing; 300ms debounce limits API calls |
| **Ride state management** | Finite state machine in Zustand | Explicit state transitions; no invalid states | Single source of truth for entire booking flow |
| **Bottom sheet panels** | Dynamic panel based on ride state; draggable height | CSS transitions + touch gesture handler | Three snap points: collapsed, half, full |
| **ETA display** | Server ETA + client countdown | `setInterval(1s)` with server correction | Smooth countdown; server corrects drift |
| **Route rendering** | Encoded polyline decoded and drawn as Polyline | `google.maps.geometry.encoding.decodePath` | Split into completed/remaining during IN_TRIP |
| **Camera following** | Auto-follow with user pan override | 10s timeout to re-enable auto-follow | Respects user exploration; resumes tracking |
| **Low-end devices** | Reduce marker count; simplify map styles; disable animation | Feature-flag based; media query for GPU capability | Show only assigned driver marker; skip nearby drivers |

---

### 5.3 Reusability Strategy

*   **MapView**: Core map component reused across BookingPage, RideDetailPage, and RideHistoryDetail. Configurable for interactive (booking) vs static (history) modes.
*   **LocationInput**: Reused for both pickup and dropoff inputs; configurable with different placeholders and icons.
*   **BottomSheet**: Generic draggable bottom panel reused for ride type selection, driver details, trip progress, and ride summary. Content changes via children.
*   **ETADisplay**: Shows "X min" with countdown; reused for driver ETA to pickup and ETA to destination.
*   **StarRating**: Rating component reused in ride summary (interactive) and ride history (read-only).
*   **DriverInfoCard**: Compact driver card (photo, name, rating, vehicle) reused in DriverDetailsPanel, TripProgressPanel, and RideHistoryDetail.
*   **FareBreakdownAccordion**: Reused in ride type selection (estimate) and ride summary (actual).
*   Props-driven configuration: e.g., `<MapView mode="interactive" />` for booking vs `<MapView mode="static" route={historicalRoute} />` for ride detail.

---

### 5.4 Module Organization

```
src/
├── pages/
│   ├── BookingPage/
│   ├── RideHistoryPage/
│   ├── RideDetailPage/
│   └── SettingsPage/
├── features/
│   ├── map/
│   │   ├── MapView.tsx
│   │   ├── PickupMarker.tsx
│   │   ├── DropoffMarker.tsx
│   │   ├── DriverTrackingMarker.tsx
│   │   ├── NearbyDriverMarkers.tsx
│   │   ├── RoutePolyline.tsx
│   │   ├── mapStyles.ts
│   │   └── hooks/
│   │       ├── useMapCamera.ts
│   │       ├── useRoutePolyline.ts
│   │       └── useMarkerAnimation.ts
│   ├── booking/
│   │   ├── LocationInputPanel.tsx
│   │   ├── LocationInput.tsx
│   │   ├── RideTypeSelector.tsx
│   │   ├── RideTypeCard.tsx
│   │   ├── FareBreakdownAccordion.tsx
│   │   ├── PromoCodeInput.tsx
│   │   ├── PaymentMethodSelector.tsx
│   │   └── DriverMatchingView.tsx
│   ├── ride-tracking/
│   │   ├── DriverDetailsPanel.tsx
│   │   ├── DriverArrivedPanel.tsx
│   │   ├── TripProgressPanel.tsx
│   │   ├── ETADisplay.tsx
│   │   ├── SafetyToolbar.tsx
│   │   └── hooks/useRideTracking.ts
│   ├── ride-summary/
│   │   ├── RideSummaryPanel.tsx
│   │   ├── StarRating.tsx
│   │   ├── TipSelector.tsx
│   │   └── FareSummary.tsx
│   ├── ride-history/
│   │   ├── RideHistoryList.tsx
│   │   ├── RideHistoryCard.tsx
│   │   └── ReceiptView.tsx
│   ├── location/
│   │   ├── SavedPlacesList.tsx
│   │   ├── RecentSearches.tsx
│   │   └── hooks/useUserLocation.ts
│   └── ride-state/
│       └── rideStore.ts (Zustand state machine)
├── shared/
│   ├── components/ (Button, Modal, BottomSheet, Skeleton, Toast)
│   ├── hooks/ (useDebounce, useMediaQuery, useWebSocket)
│   └── utils/ (currency.ts, distance.ts, polyline.ts, analytics.ts)
├── api/
│   ├── fareApi.ts
│   ├── rideApi.ts
│   ├── locationApi.ts
│   ├── rideHistoryApi.ts
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
  ├── 1. SPA shell loads (CSR) → map container renders with skeleton
  │
  ├── 2. Map SDK loads async (Google Maps JS API)
  │       ├── Show loading spinner over map area until SDK ready
  │       └── Once loaded, initialize map centered on default location
  │
  ├── 3. Location hook initializes:
  │     ├── Reads saved address from localStorage
  │     ├── OR triggers browser Geolocation API
  │     └── Map re-centers on detected location
  │
  ├── 4. Fetch nearby drivers (for map decoration):
  │       GET /api/drivers/nearby?lat=12.97&lng=77.59
  │       → Show car icons on the map (creates sense of availability)
  │
  ├── 5. Fetch saved places + recent searches from API/localStorage
  │
  ├── 6. Check for active ride (user may have closed app mid-ride):
  │       GET /api/rides/active
  │       ├── Active ride found → Restore ride state; resume tracking
  │       └── No active ride → Show IDLE state ("Where to?")
  │
  └── 7. App is interactive (TTI) — user can enter destination
```

### 6.2 User Interaction Flow

```
User enters destination and requests a ride
  │
  ├── 1. User taps "Where to?" → LocationInputPanel expands
  │
  ├── 2. User types destination → Google Places autocomplete fires (debounced 300ms)
  │       → Suggestions appear in dropdown
  │
  ├── 3. User selects a suggestion → Geocode place → Drop-off coordinates set
  │       → State: IDLE → DESTINATION_SET
  │
  ├── 4. App fetches fare estimates + route:
  │       POST /api/fare/estimate { pickup, dropoff }
  │       → Returns estimates for all ride types + encoded route polyline
  │       → Route drawn on map; fare cards shown in bottom sheet
  │
  ├── 5. User selects ride type (e.g., UberGo) → RideTypeCard highlighted
  │
  ├── 6. User taps "Request UberGo"
  │       → State: DESTINATION_SET → MATCHING
  │       → POST /api/rides/request { pickup, dropoff, rideType, paymentMethod }
  │       → Returns rideId
  │       → WebSocket connects: wss://api/rides/{rideId}/track
  │
  ├── 7. WebSocket events:
  │       ├── DRIVER_MATCHED → State: MATCHING → DRIVER_ASSIGNED
  │       │    Show driver details + OTP + ETA to pickup
  │       ├── DRIVER_LOCATION → Update marker position (animated)
  │       ├── ETA_UPDATE → Update ETA countdown
  │       ├── DRIVER_ARRIVED → State: DRIVER_ASSIGNED → DRIVER_ARRIVED
  │       ├── TRIP_STARTED → State: DRIVER_ARRIVED → IN_TRIP
  │       │    Route tracking begins; polyline splits into completed/remaining
  │       └── TRIP_COMPLETED → State: IN_TRIP → COMPLETED
  │            Show fare summary + rating
  │
  ├── 8. User rates driver (1-5 stars) + optional feedback + tip
  │       POST /api/rides/{rideId}/rate { rating, feedback, tip }
  │
  └── 9. User taps "Done" → State: COMPLETED → IDLE (reset)
```

### 6.3 Error and Retry Flow

*   **Map SDK fails to load**:
    *   Show a fallback text-based location input (no map). Users can still enter pickup/dropoff addresses and request rides.
    *   Retry loading the SDK after 5 seconds.
    *   Log the failure to error monitoring.

*   **Geolocation denied or fails**:
    *   Fall back to IP-based location (city-level accuracy).
    *   If IP location also fails, prompt user to manually enter pickup address.
    *   Do NOT block the app — location input is always available as a manual fallback.

*   **Fare estimation fails**:
    *   Show error toast: "Unable to get fare estimates. Retry?"
    *   Offer retry button. On retry, re-submit with same pickup/dropoff.
    *   If persistent failure, show "Try again later" with option to change ride type.

*   **Ride request fails (POST /api/rides/request)**:
    *   Show error toast: "Unable to request ride. Please try again."
    *   Return to DESTINATION_SET state (not IDLE) so user can retry without re-entering destination.
    *   If 503 (server overloaded), show "High demand — please try again in a moment."

*   **WebSocket disconnection during tracking**:
    *   Reconnect with exponential backoff (3 attempts: 1s, 2s, 4s).
    *   While disconnected, fall back to polling: `GET /api/rides/{id}/status` every 5 seconds.
    *   Show "Reconnecting..." indicator over the map.
    *   When reconnected, resume WebSocket stream and stop polling.

*   **Driver cancels the ride**:
    *   WebSocket sends `RIDE_CANCELLED` with reason.
    *   Show toast: "Your driver cancelled. Finding another driver..."
    *   Auto-retry matching (send new request) OR return to DESTINATION_SET and let user re-request.

*   **Payment failure post-ride**:
    *   Show fare summary with "Payment failed" status.
    *   Allow user to retry with a different payment method.
    *   Do NOT block the rating flow — user can rate even if payment is pending.

---

## 7. Data Modelling (Frontend Perspective)

### 7.1 Core Data Entities

*   **Ride**: The central entity — represents a single ride from request to completion. Tracks pickup, dropoff, driver, fare, status, and tracking data.
*   **FareEstimate**: An estimate for a specific ride type (Mini, Sedan, SUV, Pool) — includes fare range, ETA, surge, and breakdown.
*   **Driver**: Assigned driver info — name, photo, rating, vehicle details, current location.
*   **Vehicle**: Driver's vehicle — make, model, color, license plate, capacity.
*   **Location**: A geographic point — coordinates + human-readable address.
*   **SavedPlace**: A user's saved location — label (Home, Work), address, coordinates.
*   **RideHistory**: A past ride record for the history screen.
*   **PaymentMethod**: Stored payment option — card, UPI, wallet.

---

### 7.2 Data Shape

```ts
interface Location {
  coordinates: { lat: number; lng: number };
  address: string;
  placeId?: string;
}

interface FareEstimate {
  rideType: string;              // "UberGo", "Sedan", "SUV", "Pool"
  displayName: string;
  description: string;
  vehicleIcon: string;           // URL to vehicle type image
  capacity: number;
  available: boolean;
  etaMinutes: number;            // ETA for nearest driver of this type
  fareRange: {
    min: number;
    max: number;
  };
  surgeMultiplier: number;       // 1.0 = no surge; 1.5 = 50% surge
  breakdown: FareBreakdown;
}

interface FareBreakdown {
  baseFare: number;
  distanceKm: number;
  distanceCharge: number;
  durationMin: number;
  timeCharge: number;
  surgeCharge: number;
  taxes: number;
}

interface DriverInfo {
  id: string;
  name: string;
  photoUrl: string;
  rating: number;
  totalTrips: number;
  phone: string;                 // masked phone (server-side masking)
  vehicle: VehicleInfo;
  location: { lat: number; lng: number };
}

interface VehicleInfo {
  make: string;                  // "Maruti"
  model: string;                 // "Swift Dzire"
  color: string;                 // "White"
  licensePlate: string;          // "KA 01 AB 1234"
  type: string;                  // "Sedan"
}

interface Ride {
  id: string;
  status: RideState;
  pickup: Location;
  dropoff: Location;
  rideType: string;
  fareEstimate: FareEstimate;
  driver: DriverInfo | null;
  otp: string | null;
  routePolyline: string | null;  // encoded polyline
  requestedAt: string;           // ISO timestamp
  estimatedArrival: string | null;
  estimatedDropoff: string | null;
  paymentMethod: PaymentMethod;
  promoCode: string | null;
}

interface RideSummary {
  rideId: string;
  pickupAddress: string;
  dropoffAddress: string;
  driverName: string;
  driverPhoto: string;
  baseFare: number;
  distanceKm: number;
  distanceCharge: number;
  durationMin: number;
  timeCharge: number;
  surgeCharge: number;
  taxes: number;
  promoDiscount: number;
  total: number;
  paymentMethod: string;
  paymentStatus: 'SUCCESS' | 'PENDING' | 'FAILED';
}

interface SavedPlace {
  id: string;
  label: string;                 // "Home", "Work", "Gym"
  address: string;
  coordinates: { lat: number; lng: number };
  placeId: string;
}

interface PaymentMethod {
  id: string;
  type: 'upi' | 'card' | 'wallet' | 'cash';
  label: string;                 // "UPI ****1234", "Visa ****5678"
  isDefault: boolean;
}

interface NearbyDriver {
  id: string;
  location: { lat: number; lng: number };
  heading: number;               // direction of travel in degrees
  rideType: string;              // what type of ride they serve
}

interface RideHistoryItem {
  rideId: string;
  date: string;                  // ISO date
  pickupAddress: string;
  dropoffAddress: string;
  fare: number;
  driverName: string;
  rideType: string;
  status: 'COMPLETED' | 'CANCELLED';
  rating: number | null;
}
```

---

### 7.3 Entity Relationships

```
User 1 ──────── * SavedPlace         (one user has many saved places)
User 1 ──────── * PaymentMethod      (one user has many payment methods)
User 1 ──────── * Ride               (one user has many ride records)

Ride 1 ──────── 1 DriverInfo         (each ride has one assigned driver)
Ride 1 ──────── 1 VehicleInfo        (through DriverInfo)
Ride 1 ──────── 1 Location (pickup)  (each ride has one pickup location)
Ride 1 ──────── 1 Location (dropoff) (each ride has one drop-off location)
Ride 1 ──────── 1 FareEstimate       (selected ride type estimate)
Ride 1 ──────── 1 RideSummary        (generated on trip completion)
Ride 1 ──────── 1 PaymentMethod      (selected payment for this ride)

FareEstimate * ── 1 RideRequest      (multiple estimates shown; one selected)
```

*   **Denormalized for display**: DriverInfo includes embedded VehicleInfo — avoids extra API calls.
*   **Ride is the root aggregate**: All ride-related data (driver, fare, route, status) is accessed through the Ride entity.
*   **FareEstimates are ephemeral**: Recalculated on every pickup/dropoff change; not persisted long-term.

---

### 7.4 UI Specific Data Models

*   **RideTypeCardViewModel**: Combines FareEstimate with display formatting — `fareLabel` ("Rs120-150"), `etaLabel` ("3 min away"), `surgeLabel` ("1.5x"), used by RideTypeCard.
*   **DriverDetailsViewModel**: Combines DriverInfo with computed fields — `nameInitials`, `ratingLabel` ("4.8 ★"), `vehicleDescription` ("White Maruti Swift Dzire · KA01AB1234").
*   **ETAViewModel**: Server ETA + client countdown state — `displayMinutes`, `lastServerUpdate`, `isCountingDown`.
*   **MapCamera**: Current map center, zoom level, bounds — derived from ride state (e.g., fit both pickup and dropoff, or follow driver).
*   **BottomSheetContent**: Which panel to show in the bottom sheet, derived from `rideState` — maps `IDLE` → DestinationPrompt, `DESTINATION_SET` → RideTypeSelector, etc.

---

## 8. State Management Strategy

### 8.1 State Classification

| State Type | Examples | Where Stored |
|-----------|----------|--------------|
| **Ride lifecycle state** | Current ride state machine (IDLE, MATCHING, IN_TRIP, etc.), driver info, fare, OTP, route | Zustand global store (`rideStore`) |
| **Map state** | Map center, zoom, markers, polylines | React refs + Google Maps SDK internal state |
| **Server state** | Fare estimates, ride history, saved places, payment methods | React Query cache |
| **Location state** | Current user location, selected pickup, selected dropoff | Zustand ride store (pickup/dropoff) + Geolocation API |
| **Real-time state** | Driver location, ETA, ride status updates | Zustand store (updated via WebSocket) |
| **Local UI state** | Bottom sheet height, autocomplete dropdown open, rating stars selected | React useState within components |
| **Derived state** | Bottom sheet panel to show (from ride state), formatted fare, ETA countdown | Computed from store (selectors / derived) |

---

### 8.2 State Ownership

*   **Ride Store (Zustand)** owns the entire ride lifecycle — ride state machine, pickup/dropoff, driver, fare, OTP, route, summary. All ride-related components read from this single store.
*   **React Query** owns all server/async data — fare estimates (fetched on demand), ride history, saved places, payment methods.
*   **WebSocket hook** pushes updates into the Zustand ride store — driver location, ETA, status changes are written to the store by the `useRideTracking` hook.
*   **Map SDK** owns its own rendering state — map center, zoom, tile loading. React components control the map via refs and imperative API calls (not declarative React rendering).
*   **Local component state** owns ephemeral UI state — bottom sheet drag position, autocomplete open/closed, rating selection.

---

### 8.3 Persistence Strategy

| Data | Storage | Reason |
|------|---------|--------|
| **Active ride** | Zustand (in-memory) + server | On app reload, fetch active ride from `GET /api/rides/active` to restore state |
| **Pickup location** | localStorage (last used) | Quick restore on next session |
| **Saved places** | Server (API) + React Query cache | Cross-device sync; small list |
| **Recent searches** | localStorage (last 5) | Quick access; no server sync needed |
| **Payment methods** | Server (API) + React Query cache | Sensitive data; must be server-authoritative |
| **Auth token** | HTTP-only cookie | Secure; sent automatically |
| **Map preferences** | localStorage (zoom level, map type) | User preference persistence |
| **Ride history** | Server (API) + React Query cache | Fetched on demand; no offline requirement |

---

## 9. High Level API Design (Frontend POV)

### 9.1 Required APIs

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/drivers/nearby` | GET | Fetch nearby driver locations for map decoration |
| `/api/fare/estimate` | POST | Get fare estimates for all ride types given pickup and dropoff |
| `/api/rides/request` | POST | Request a new ride |
| `/api/rides/active` | GET | Check if user has an active ride (for reload recovery) |
| `/api/rides/:id` | GET | Fetch ride details |
| `/api/rides/:id/cancel` | POST | Cancel a ride |
| `/api/rides/:id/rate` | POST | Submit rating, feedback, and tip |
| `/api/rides/:id/status` | GET | Fetch current ride status (polling fallback) |
| `wss://api/rides/:id/track` | WebSocket | Real-time ride tracking stream |
| `/api/places/saved` | GET | Fetch user's saved places |
| `/api/places/saved` | POST | Add a new saved place |
| `/api/places/recent` | GET | Fetch recent search locations |
| `/api/payments/methods` | GET | Fetch saved payment methods |
| `/api/promo/validate` | POST | Validate a promo code |
| `/api/rides/history` | GET | Fetch paginated ride history |
| `/api/rides/:id/receipt` | GET | Fetch ride receipt |

---

### 9.2 Real Time Ride Tracking Strategy (Deep Dive)

##### Why WebSocket over SSE or Polling

| Approach | How it Works | Latency | Complexity | Best For |
|----------|-------------|---------|------------|----------|
| **Polling** | Client sends GET every N seconds | High (3-10s delay) | Simple | Low-priority status; fallback only |
| **SSE** | Server pushes events over HTTP | Low (real-time) | Moderate | One-way updates only |
| **WebSocket** | Full-duplex persistent connection | Lowest (real-time) | Higher | Driver location (every 3s) + status + ETA; bidirectional for OTP verification, cancel |

**Recommendation: WebSocket** because:
1.  **Driver location updates** arrive every 3-5 seconds. Polling would be too frequent; SSE works but doesn't support client-to-server messages.
2.  Multiple event types over one connection: location, status, ETA, route changes.
3.  Client may need to send messages: OTP confirmation, cancellation request.

##### WebSocket Message Types

```ts
// Server → Client messages
type ServerMessage =
  | { type: 'DRIVER_LOCATION'; lat: number; lng: number; heading: number }
  | { type: 'ETA_UPDATE'; etaMinutes: number }
  | { type: 'DRIVER_MATCHED'; driver: DriverInfo; otp: string }
  | { type: 'DRIVER_ARRIVED' }
  | { type: 'TRIP_STARTED' }
  | { type: 'TRIP_COMPLETED'; summary: RideSummary }
  | { type: 'RIDE_CANCELLED'; reason: string; cancelledBy: 'driver' | 'system' }
  | { type: 'ROUTE_UPDATE'; updatedPolyline: string }
  | { type: 'SURGE_UPDATE'; newMultiplier: number };

// Client → Server messages
type ClientMessage =
  | { type: 'OTP_VERIFIED' }
  | { type: 'CANCEL_RIDE'; reason: string }
  | { type: 'PING' };  // keep-alive
```

##### Reconnection Strategy

```
WebSocket disconnects
  │
  ├── Wait 1s → attempt reconnect #1
  │    ├── Success → resume tracking; request state snapshot
  │    └── Fail → wait 2s
  │
  ├── Attempt reconnect #2
  │    ├── Success → resume tracking
  │    └── Fail → wait 4s
  │
  ├── Attempt reconnect #3
  │    └── Fail → Switch to polling fallback
  │
  └── Polling mode: GET /api/rides/:id/status every 5s
       └── When WebSocket reconnects, stop polling
```

```tsx
function useResilientRideTracking(rideId: string | null) {
  const [mode, setMode] = useState<'websocket' | 'polling'>('websocket');
  const wsRef = useRef<WebSocket | null>(null);
  const reconnectAttemptRef = useRef(0);
  const maxReconnectAttempts = 3;

  const connect = useCallback(() => {
    if (!rideId) return;

    const ws = new WebSocket(
      `wss://api.example.com/rides/${encodeURIComponent(rideId)}/track`
    );

    ws.onopen = () => {
      reconnectAttemptRef.current = 0;
      setMode('websocket');
    };

    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      handleRideUpdate(message);
    };

    ws.onclose = () => {
      if (reconnectAttemptRef.current < maxReconnectAttempts) {
        const delay = Math.pow(2, reconnectAttemptRef.current) * 1000;
        reconnectAttemptRef.current += 1;
        setTimeout(connect, delay);
      } else {
        setMode('polling');
      }
    };

    wsRef.current = ws;
  }, [rideId]);

  // Polling fallback
  useEffect(() => {
    if (mode !== 'polling' || !rideId) return;

    const interval = setInterval(async () => {
      const response = await fetch(`/api/rides/${encodeURIComponent(rideId)}/status`);
      const data = await response.json();
      handleRideUpdate(data);
    }, 5000);

    // Keep trying WebSocket in background
    const retryWs = setTimeout(connect, 30000);

    return () => {
      clearInterval(interval);
      clearTimeout(retryWs);
    };
  }, [mode, rideId, connect]);

  useEffect(() => {
    connect();
    return () => wsRef.current?.close();
  }, [connect]);
}
```

---

### 9.3 Request and Response Structure

##### Fare Estimation API

```json
// POST /api/fare/estimate

// Request:
{
  "pickup": { "lat": 12.9716, "lng": 77.5946 },
  "dropoff": { "lat": 12.9352, "lng": 77.6245 },
  "promoCode": "FIRST50"
}

// Response:
{
  "routePolyline": "a~l~Fjk~uOwHJy@P...",
  "distanceKm": 6.2,
  "durationMin": 18,
  "estimates": [
    {
      "rideType": "ubergo",
      "displayName": "UberGo",
      "description": "Affordable, everyday rides",
      "vehicleIcon": "https://cdn.example.com/vehicles/ubergo.png",
      "capacity": 4,
      "available": true,
      "etaMinutes": 3,
      "fareRange": { "min": 120, "max": 150 },
      "surgeMultiplier": 1.0,
      "breakdown": {
        "baseFare": 40,
        "distanceKm": 6.2,
        "distanceCharge": 56,
        "durationMin": 18,
        "timeCharge": 18,
        "surgeCharge": 0,
        "taxes": 11
      }
    },
    {
      "rideType": "sedan",
      "displayName": "Sedan",
      "description": "Comfortable sedans",
      "vehicleIcon": "https://cdn.example.com/vehicles/sedan.png",
      "capacity": 4,
      "available": true,
      "etaMinutes": 5,
      "fareRange": { "min": 180, "max": 220 },
      "surgeMultiplier": 1.0,
      "breakdown": { "..." : "..." }
    },
    {
      "rideType": "suv",
      "displayName": "SUV",
      "description": "Extra space for groups",
      "vehicleIcon": "https://cdn.example.com/vehicles/suv.png",
      "capacity": 6,
      "available": true,
      "etaMinutes": 7,
      "fareRange": { "min": 350, "max": 420 },
      "surgeMultiplier": 1.5,
      "breakdown": { "..." : "..." }
    }
  ]
}
```

##### Ride Request API

```json
// POST /api/rides/request

// Request:
{
  "pickup": {
    "coordinates": { "lat": 12.9716, "lng": 77.5946 },
    "address": "123 MG Road, Bangalore",
    "placeId": "ChIJ..."
  },
  "dropoff": {
    "coordinates": { "lat": 12.9352, "lng": 77.6245 },
    "address": "456 Koramangala, Bangalore",
    "placeId": "ChIJ..."
  },
  "rideType": "ubergo",
  "paymentMethodId": "pm_001",
  "promoCode": "FIRST50"
}

// Response:
{
  "rideId": "ride_abc123",
  "status": "MATCHING",
  "estimatedMatchTime": 30,
  "trackingUrl": "wss://api.example.com/rides/ride_abc123/track"
}
```

##### Ride Status API (Polling Fallback)

```json
// GET /api/rides/ride_abc123/status

// Response:
{
  "rideId": "ride_abc123",
  "status": "DRIVER_ASSIGNED",
  "driver": {
    "id": "drv_001",
    "name": "Rajesh Kumar",
    "photoUrl": "https://cdn.example.com/drivers/drv_001.jpg",
    "rating": 4.8,
    "totalTrips": 1523,
    "phone": "+91****1234",
    "vehicle": {
      "make": "Maruti",
      "model": "Swift Dzire",
      "color": "White",
      "licensePlate": "KA 01 AB 1234",
      "type": "Sedan"
    },
    "location": { "lat": 12.9700, "lng": 77.5960 }
  },
  "otp": "4782",
  "etaMinutes": 4,
  "routePolyline": "a~l~Fjk~uOwHJy@P..."
}
```

##### Rate Ride API

```json
// POST /api/rides/ride_abc123/rate

// Request:
{
  "rating": 5,
  "feedback": "Great ride, very smooth driving!",
  "tipAmount": 30
}

// Response:
{
  "success": true,
  "message": "Thank you for your rating!"
}
```

---

### 9.4 Error Handling and Status Codes

| Status Code | Scenario | Frontend Action |
|-------------|----------|-----------------|
| **200** | Success | Process response normally |
| **400** | Invalid request (bad coordinates, invalid ride type) | Show inline validation error |
| **401** | Authentication expired | Redirect to login; preserve location inputs |
| **403** | Forbidden (accessing another user's ride) | Show "Access denied" page |
| **404** | Ride not found | Show "Ride not found" with link to history |
| **409** | Conflict (e.g., user already has an active ride) | Show "You already have an active ride" + link to tracking |
| **422** | Validation (destination too close, same pickup and dropoff) | Show specific message: "Pickup and drop-off are too close" |
| **429** | Rate limited | Show "Too many requests" toast; local throttle |
| **500** | Server error | Show error toast with retry button |
| **503** | Service unavailable (peak demand) | Show "High demand — please try again shortly" |

---

## 10. Caching Strategy

### What to Cache

| Data | Cache Duration | Invalidation |
|------|---------------|-------------|
| **Nearby drivers** | 30 seconds | Auto-refresh on map pan/zoom; replaced on each fetch |
| **Fare estimates** | None (always fresh) | Recalculated on every pickup/dropoff change |
| **Saved places** | 10 min | On place add/edit/delete |
| **Payment methods** | 10 min | On payment method change |
| **Ride history** | 5 min | On new ride completion |
| **Vehicle type images** | Long-lived (CDN) | URL-versioned |
| **Map tiles** | Browser HTTP cache | Managed by Maps SDK |

### Where to Cache

| Layer | What | Strategy |
|-------|------|----------|
| **React Query** | Saved places, payment methods, ride history | Stale-while-revalidate; configurable staleTime |
| **Browser HTTP cache** | Map tiles, CDN assets, vehicle icons | `Cache-Control: public, max-age=31536000, immutable` for hashed assets |
| **Map SDK internal cache** | Geocode results, places predictions | SDK manages internally (session-scoped) |
| **localStorage** | Last pickup location, recent searches | Manual read/write |
| **In-memory (Zustand)** | Active ride state, driver location | Ephemeral; lost on page close |

### Cache Invalidation

*   **Location change** → Nearby drivers refetched; fare estimates recalculated.
*   **Ride completion** → Invalidate ride history query.
*   **Payment method change** → Invalidate payment methods query.
*   **Active ride data** → Never cached; always live via WebSocket.

---

## 11. CDN and Asset Optimization

*   **Vehicle type images**: Small PNG/SVG icons (~5KB each) served via CDN. Pre-loaded during app shell load.
*   **Map SDK**: Loaded asynchronously via `<script async>` with `loading=async` parameter. Not bundled — loaded from Google CDN.
*   **Driver photos**: Small thumbnails (80x80, ~10KB) served via CDN with signed URLs.
*   **JS/CSS bundles**: Content-hashed filenames; `Cache-Control: immutable`; code-split by route.
*   **Compression**: Brotli on all text assets (HTML, CSS, JS, JSON).
*   **Preconnect**: `<link rel="preconnect" href="https://maps.googleapis.com">` and `<link rel="preconnect" href="https://cdn.example.com">`.
*   **Font optimization**: `font-display: swap`; preload primary font. System font stack as fallback.

---

## 12. Rendering Strategy

| Page | Strategy | Reason |
|------|----------|--------|
| **Booking Page** | CSR only | Highly interactive; map-heavy; auth-gated; no SEO value |
| **Ride History** | CSR only | Auth-gated; dynamic per user; no SEO value |
| **Ride Detail** | CSR only | Auth-gated; fetched on demand |
| **Settings/Profile** | CSR only | Auth-gated |
| **Landing Page** | SSG (Static Generation) | Marketing page; needs SEO; rarely changes |
| **City Pages** | SSR/SSG | SEO for "Uber in Bangalore"; structured data |
| **Help Center** | SSG | Content rarely changes; fully cacheable |

### Hydration Strategy

*   The core booking experience is **fully CSR** — no SSR hydration needed. The SPA shell loads first, then the Map SDK loads, then data fetches fire.
*   The landing/marketing pages use SSG with minimal hydration (only interactive elements like "Sign Up" buttons).
*   Map is initialized imperatively (not React-rendered), so there's no hydration mismatch risk.

---

## 13. Cross Cutting Non Functional Concerns

### 13.1 Security

*   **Authentication**: JWT stored in HTTP-only, Secure, SameSite cookie. Refresh token rotation.
*   **CSRF**: CSRF token in meta tag; included in all state-changing requests (ride request, cancel, rate).
*   **Phone Masking**: Driver and rider phone numbers are masked server-side. The frontend receives masked strings ("+91****1234") — never raw phone numbers.
*   **OTP Security**: Ride verification OTP is displayed only to the rider and must be verbally told to the driver. Not transmitted via frontend logging or analytics.
*   **Payment Security**: Payment tokens handled via gateway SDK (Stripe/Razorpay). No card numbers or CVVs ever touch the frontend.
*   **Content Security Policy**: Strict CSP allowing only Maps SDK, payment SDK, analytics, and own domain.
*   **WebSocket Security**: WSS (encrypted); authenticated via token in the connection handshake.

---

### 13.2 Accessibility

*   **Semantic HTML**: Location inputs use `role="combobox"` + `aria-autocomplete`; ride types use `role="radiogroup"`; rating uses `role="radiogroup"`.
*   **Keyboard Navigation**: Tab through location inputs, ride type options, request button. Arrow keys within ride type group. Escape closes autocomplete dropdown.
*   **Screen Reader Support**:
    *   Ride status changes announced via `aria-live="assertive"`: "Driver assigned. Rajesh Kumar arriving in 4 minutes."
    *   ETA changes announced via `aria-live="polite"`.
    *   Map has a text-based alternative: "Pickup at 123 MG Road. Drop-off at 456 Koramangala. Route distance: 6.2 km."
*   **Focus Management**: When ride state changes (matching → driver assigned), focus moves to the new panel's heading. When autocomplete opens, focus moves to the suggestion list.
*   **Color Contrast**: Surge badge uses both color (orange) and text ("1.5x") — not color alone. Pin icons have distinct shapes for pickup (circle) and drop-off (flag) — not just different colors.
*   **Reduced Motion**: `prefers-reduced-motion` disables marker animation, pulse animations, and smooth map pan.

---

### 13.3 Performance Optimization

*   **Map SDK Lazy Loading**: Load Google Maps SDK only when needed (on app mount, not in the initial HTML). Show skeleton map during load.
    ```html
    <script async defer src="https://maps.googleapis.com/maps/api/js?key=API_KEY&libraries=places,geometry&callback=initMap"></script>
    ```
*   **Code Splitting**: Ride history, settings, ride detail pages are lazy-loaded. Map-heavy code is in the primary chunk (since it's always needed on the booking page).
    ```tsx
    const RideHistoryPage = lazy(() => import('./pages/RideHistoryPage'));
    const RideDetailPage = lazy(() => import('./pages/RideDetailPage'));
    const SettingsPage = lazy(() => import('./pages/SettingsPage'));
    ```
*   **Marker Optimization on Map**: Limit visible markers to 20 nearest drivers. Use OverlayView with custom DOM elements for markers (lighter than default Google Marker for bulk operations). Cluster markers if > 20 in viewport.
*   **Throttle Driver Location Renders**: Even though WebSocket sends updates every 3s, batch marker position updates using `requestAnimationFrame` to avoid layout thrashing.
*   **Network Optimization**:
    *   `<link rel="preconnect" href="https://maps.googleapis.com" />` — Map SDK.
    *   `<link rel="preconnect" href="https://cdn.example.com" />` — CDN for assets.
    *   Debounce autocomplete by 300ms to limit Places API calls.
*   **Performance Metrics**:
    *   FCP < 1.5s (app shell renders with map skeleton).
    *   Map interactive < 3s (tiles load + first marker visible).
    *   LCP < 2.5s (map with nearby drivers rendered).
    *   INP < 200ms (ride type selection, request button respond instantly).

---

### 13.4 Observability and Reliability

*   **Error Boundaries**: Wrap map, location input, bottomsheet, and ride tracking in separate error boundaries. A map crash doesn't break the entire app — fallback to text-based location input.
*   **Error Logging**: Unhandled errors and API failures logged to Sentry/Datadog with context (ride state, ride ID, user action).
*   **Performance Monitoring**: Web Vitals (FCP, LCP, CLS, INP) reported to analytics. Custom metrics: map-load-time, time-to-fare-estimate, matching-duration.
*   **Feature Flags**: Roll out new map features (dark mode map, new ride types, redesigned bottom sheet) behind feature flags.
*   **Analytics Events**:
    *   `location_set` — pickup or dropoff set (with source: autocomplete, saved, current location).
    *   `fare_viewed` — fare estimates viewed (with ride types shown, surge status).
    *   `ride_requested` — ride request submitted (with ride type, surge multiplier, fare estimate).
    *   `ride_completed` — ride finished (with actual fare, duration, distance, rating).
    *   `ride_cancelled` — cancellation (with cancelled_by, reason, time_in_state).
    *   `map_interaction` — zoom, pan, recenter (for UX analysis).
    *   `ws_disconnection` — WebSocket disconnect (with ride state, reconnect attempts).
*   **Health Checks**: Frontend pings `/api/health` on app load. If unhealthy, show banner: "Some features may be unavailable."

---

## 14. Edge Cases and Tradeoffs

### Edge Cases

| Scenario | How to Handle |
|---------|---------------|
| **GPS inaccurate (urban canyons, indoor)** | Show "Adjust your pickup" prompt with draggable pin. Accept a tolerance radius (~50m). Reverse geocode on pin drag end. |
| **User closes app during ride** | On reopen, `GET /api/rides/active` returns the in-progress ride. Restore state machine to the correct phase; reconnect WebSocket. |
| **Driver takes a different route than shown** | Server sends `ROUTE_UPDATE` via WebSocket with a new polyline. Frontend redraws the route. Show "Route updated" toast. |
| **Surge pricing changes after fare shown** | Fare estimate has a TTL (e.g., 5 minutes). If expired, re-fetch before allowing request. If surge increased, show updated fare confirmation. |
| **No drivers available** | After 60s matching timeout, show "No drivers available nearby" with options: try a different ride type, schedule for later, or try again. |
| **Multiple ride requests (double tap)** | Disable "Request" button immediately on tap; re-enable on error. Server also enforces one active ride per user (409 response). |
| **Very long routes (intercity)** | Show a warning: "This is a long-distance ride (~150 km). Estimated fare: Rs 2500-3000." Require explicit confirmation. |
| **User enters same pickup and dropoff** | Validate client-side and show: "Pickup and drop-off are the same location." Don't send API request. |
| **Network drops during matching** | Queue the request; show "Reconnecting..." When reconnected, check ride status — driver may have been matched while offline. |
| **Driver is very far (> 15 min ETA)** | Show a notice: "Nearest driver is 15+ minutes away. Continue waiting?" with option to cancel. |
| **Payment method declined post-ride** | Show "Payment failed" on ride summary with "Retry" and "Change payment method" options. Block new ride requests until resolved. |
| **Map SDK fails to load (CDN down, ad-blocker)** | Show text-based fallback: location inputs still work, fare estimates still show, but no map visualization. Track this in analytics. |
| **Low battery / data saver mode** | Reduce map interaction (disable tilt, simplify styles); increase WebSocket polling interval to 10s; disable marker animation. |
| **Scheduled ride reminder** | Store scheduled ride in localStorage + server. Show notification banner at T-15 minutes. Auto-trigger matching at scheduled time. |

### Key Tradeoffs

| Decision | Tradeoff |
|----------|----------|
| **Full CSR (no SSR for booking)** | Faster interactivity after load; simpler infra. But slightly slower FCP than SSR; no SEO for the booking flow (acceptable — it is auth-gated). |
| **Google Maps over Mapbox** | Better geocoding in India; familiar to users; rich Places API. But more expensive at scale; less stylistic control. |
| **WebSocket over SSE for tracking** | Bidirectional; supports cancel and OTP verification. But more complex; requires sticky sessions or pub/sub. |
| **Zustand state machine over XState** | Simpler; smaller bundle; team familiarity. But less formal — no visual state chart; invalid transitions possible if not careful. |
| **Client-side ETA countdown** | Smooth UX; feels real-time. But can drift from server; requires correction handling. |
| **Encoded polyline over GeoJSON** | Much smaller payload (~90% smaller); fast decode. But less human-readable; requires geometry library for decoding. |
| **Session tokens for Places API** | Significant billing reduction. But adds complexity (token lifecycle management); must reset on selection. |
| **Map style customization** | Cleaner, focused UX. But deviates from user's familiar Google Maps look; may need updates when SDK changes. |
| **Driver marker animation** | Premium, smooth experience. But CPU-intensive on low-end devices; must be disabled for performance on weak hardware. |

---

## 15. Summary and Future Improvements

### Key Architectural Decisions

1.  **Map-first, CSR architecture** — The map is the primary UI canvas. All other panels overlay or dock to the map. Fully client-side rendered (no SSR for the booking flow) because the app is auth-gated and highly interactive.
2.  **Finite state machine for ride lifecycle** — A Zustand store with explicit states (IDLE → DESTINATION_SET → MATCHING → DRIVER_ASSIGNED → IN_TRIP → COMPLETED) ensures predictable UI transitions and prevents invalid states.
3.  **WebSocket for real-time tracking with polling fallback** — Driver location, ETA, and ride status stream over a single persistent connection. On disconnect, exponential backoff reconnect with REST polling as a last resort.
4.  **Ease-out interpolated marker animation** — Driver markers glide smoothly between GPS updates using `requestAnimationFrame` with cubic ease-out — prevents the "teleporting marker" problem.
5.  **Google Places autocomplete with session tokens** — Optimized billing by grouping all autocomplete queries + geocode into one billing session per location selection.
6.  **Dynamic bottom sheet panels** — A single draggable BottomSheet component renders different content based on the ride state (ride types, driver details, trip progress, summary).
7.  **Auto-follow map camera with user override** — Map follows the driver by default but pauses auto-follow when the user pans. Re-enables after 10s of inactivity.

### Possible Future Enhancements

*   **Offline ride recovery**: Cache active ride state in IndexedDB. If user goes offline temporarily, restore the last known state and show "Offline — will reconnect when network is available."
*   **Web Workers for map calculations**: Offload polyline splitting, distance calculations, and marker clustering to a Web Worker to keep the main thread free for smooth map rendering.
*   **Predictive destination**: Based on time of day and ride history, auto-suggest the most likely destination (e.g., "Home" in the evening, "Office" in the morning).
*   **Multi-stop rides**: Allow users to add intermediate stops. Requires polyline re-rendering and fare recalculation for each added stop.
*   **AR navigation for pickup**: Use WebXR to show an AR arrow directing the rider to the exact driver location in a crowded parking lot or airport pickup zone.
*   **Dark mode map**: Switch to a dark map style during nighttime (based on local time) — easier on the eyes and saves battery on OLED screens.
*   **Group ride cost-splitting**: After ride completion, allow the rider to split the fare with other passengers via payment links.
*   **Carbon footprint tracker**: Show CO2 saved when users choose shared/pool rides versus solo rides.

---

### Endpoint Summary

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/drivers/nearby` | GET | Fetch nearby driver locations |
| `/api/fare/estimate` | POST | Get fare estimates for ride types |
| `/api/rides/request` | POST | Request a new ride |
| `/api/rides/active` | GET | Check for active ride (reload recovery) |
| `/api/rides/:id` | GET | Fetch ride details |
| `/api/rides/:id/cancel` | POST | Cancel a ride |
| `/api/rides/:id/rate` | POST | Submit rating and tip |
| `/api/rides/:id/status` | GET | Fetch ride status (polling fallback) |
| `wss://api/rides/:id/track` | WebSocket | Real time ride tracking stream |
| `/api/places/saved` | GET | Fetch saved places |
| `/api/places/saved` | POST | Add saved place |
| `/api/places/recent` | GET | Fetch recent searches |
| `/api/payments/methods` | GET | Fetch saved payment methods |
| `/api/promo/validate` | POST | Validate promo code |
| `/api/rides/history` | GET | Fetch paginated ride history |
| `/api/rides/:id/receipt` | GET | Fetch ride receipt |

---

### Complete Ride Flow

| Step | Direction | Mechanism | Endpoint | Action |
|------|-----------|-----------|----------|--------|
| 1 | Initial | REST | `GET /api/drivers/nearby?lat=...&lng=...` | Show nearby driver dots on map |
| 2 | User Action | Google Places API | Places Autocomplete | User enters destination |
| 3 | Fare | REST | `POST /api/fare/estimate` | Show ride types with fare estimates |
| 4 | Request | REST | `POST /api/rides/request` | Initiate ride; get rideId |
| 5 | Tracking | WebSocket | `wss://api/rides/:id/track` | Stream driver location, status, ETA |
| 6 | Fallback | REST (Polling) | `GET /api/rides/:id/status` | Poll every 5s if WebSocket disconnected |
| 7 | Complete | WebSocket | `TRIP_COMPLETED` message | Show fare summary and rating |
| 8 | Rating | REST | `POST /api/rides/:id/rate` | Submit rating, feedback, tip |
| 9 | History | REST | `GET /api/rides/history` | View past rides |

---

More Details:

Get all articles related to system design
Hashtag: SystemDesignWithZeeshanAli

[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design
