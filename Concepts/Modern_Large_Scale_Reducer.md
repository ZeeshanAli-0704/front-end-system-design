
# 🏗️ Modern Large-Scale Reducer Design

> How to structure state management for apps with **100–200+ screens** without losing your sanity.

---

<a id="top"></a>

## Table of Contents

- [The Problem](#the-problem)
- [The Office Building Analogy](#the-office-building-analogy)
- [Core Principles](#core-principles)
- [Step by Step Process](#step-by-step-process)
- [Folder Structure](#folder-structure)
- [Skeleton Code Store Setup](#skeleton-code-store-setup)
- [Skeleton Code Feature Slice](#skeleton-code-feature-slice)
- [Skeleton Code Multiple Slices per Feature](#skeleton-code-multiple-slices-per-feature)
- [Skeleton Code Final State Shape](#skeleton-code-final-state-shape)
- [What State Goes Where](#what-state-goes-where)
- [Normalize Your Data](#normalize-your-data)
- [Selectors How to Read State](#selectors-how-to-read-state)
- [How Features Should Communicate](#how-features-should-communicate)
- [When to Split a Slice](#when-to-split-a-slice)
- [Async Data Strategy](#async-data-strategy)
- [Enterprise Rules](#enterprise-rules)
- [Migrating Legacy State](#migrating-legacy-state)
- [Strategy Comparison](#strategy-comparison)
- [Checklist](#checklist)
- [Interview Takeaways](#interview-takeaways)
- [Further Reading](#further-reading)

[⬆ Back to Top](#top)

---

## The Problem

When your app is small, one reducer file works fine. When it grows to **200+ screens with multiple teams**, that same file becomes:

- 3,000+ lines nobody owns
- A merge conflict magnet
- Impossible to test or code-split
- A place where changing "cart" logic breaks "orders"

**Reducer architecture** solves this by giving every feature **its own state boundary**.

[⬆ Back to Top](#top)

---

## The Office Building Analogy

| Real World              | State Concept        | Purpose                            |
| ----------------------- | -------------------- | ---------------------------------- |
| 🏢 Building             | Your App             | The whole system                   |
| 🧑‍💼 Building Manager    | Store                | Single source of truth             |
| 🏬 Each Floor           | Feature Domain       | Users, Payments, Orders            |
| 🗄️ File Cabinets        | Slices               | State + actions for one concern    |
| 📝 Memos                | Actions              | Describe what happened             |
| 🔍 Lookup Index         | Selectors            | How to read state efficiently      |

> **Golden Rule:** Files from one floor never mix with another floor. Each feature manages its own state.

[⬆ Back to Top](#top)

---

## Core Principles

| #  | Principle                    | One-Liner                                                      |
| -- | ---------------------------- | -------------------------------------------------------------- |
| 1  | Feature Ownership            | Each feature owns its state — nobody else writes to it         |
| 2  | Single Responsibility        | One slice = one domain concept                                 |
| 3  | Flat State                   | Max 2-3 levels deep. No deeply nested objects                  |
| 4  | Derive, Don't Store          | Computed values belong in selectors, not in state              |
| 5  | Normalize Entities           | Store lists as `{ ids: [], entities: {} }` for O(1) lookups   |
| 6  | Minimal Shared State         | Only auth, theme, and UI chrome belong in shared/              |
| 7  | Namespace Actions            | `users/setList` not `SET_LIST`                                 |
| 8  | Public API per Feature       | Features export only what others need via an index file        |

[⬆ Back to Top](#top)

---

## Step by Step Process

| Step | What to Do                              | Key Question to Ask                                          |
| ---- | --------------------------------------- | ------------------------------------------------------------ |
| 1    | **Map your domains**                    | What are the major business areas? (Users, Orders, Cart...)  |
| 2    | **Classify each piece of state**        | Is it server data, client UI, form state, URL state, or derived? |
| 3    | **Design folder structure**             | One folder per domain with slices, selectors, API, types     |
| 4    | **Define state shape per feature**      | What's the minimum flat data this feature needs?             |
| 5    | **Normalize entity lists**              | Does this list need fast lookups or individual updates?      |
| 6    | **Create selectors**                    | What derived/computed data do components need?               |
| 7    | **Choose async strategy**               | CRUD → caching library. Complex → thunks. Reactive → listeners |
| 8    | **Plan cross-feature communication**    | Can I use shared events instead of direct imports?           |
| 9    | **Add middleware** for cross-cutting     | Analytics, error reporting, auth refresh                     |
| 10   | **Split slices** when they grow too big | Is this slice > 300 lines or serving multiple screens?       |
| 11   | **Code-split reducers** by route        | Does every page need every reducer loaded?                   |
| 12   | **Document rules** for the team         | What conventions should every dev follow?                    |

[⬆ Back to Top](#top)

---

## Folder Structure

```
/src
 ├── app/                     ← Store config, root reducer, typed hooks
 │
 ├── features/                ← One folder per business domain
 │    ├── users/
 │    │    ├── slices/        ← State + reducers + actions
 │    │    ├── selectors/     ← Memoized state derivations
 │    │    ├── api/           ← Async operations
 │    │    ├── components/    ← Feature-specific UI
 │    │    ├── pages/         ← Route-level components
 │    │    ├── types/         ← TypeScript interfaces
 │    │    └── index          ← Public API (exports for other features)
 │    │
 │    ├── payments/           ← Same structure
 │    ├── orders/
 │    └── ...
 │
 ├── shared/                  ← ONLY: auth, UI chrome, config
 │    ├── slices/
 │    ├── hooks/
 │    └── utils/
 │
 └── components/              ← Reusable UI primitives (Button, Modal)
```

**Why this works:** Adding a feature = adding a folder. Deleting a feature = deleting a folder. No other code touched.

[⬆ Back to Top](#top)

---

## Skeleton Code Store Setup

### `app/store.js`

```js
import { configureStore } from "@reduxjs/toolkit";
import rootReducer from "./rootReducer";

export const store = configureStore({
  reducer: rootReducer,
});
```

### `app/rootReducer.js`

```js
import { combineReducers } from "@reduxjs/toolkit";

import usersReducer from "../features/users";
import paymentReducer from "../features/payments";
import orderReducer from "../features/orders";

import authReducer from "../shared/slices/authSlice";
import uiReducer from "../shared/slices/uiSlice";

export default combineReducers({
  users: usersReducer,
  payments: paymentReducer,
  orders: orderReducer,
  auth: authReducer,
  ui: uiReducer,
});
```

[⬆ Back to Top](#top)

---

## Skeleton Code Feature Slice

### `features/users/slices/usersSlice.js`

```js
import { createSlice } from "@reduxjs/toolkit";

const initialState = {
  list: [],
  loading: false,
};

const usersSlice = createSlice({
  name: "users",
  initialState,
  reducers: {
    setUsers(state, action) {
      state.list = action.payload;
    },
    setLoading(state, action) {
      state.loading = action.payload;
    },
  },
});

export const { setUsers, setLoading } = usersSlice.actions;
export default usersSlice.reducer;
```

Each feature behaves like a **mini-application**:

```txt
/features/users
 ├── slices/
 ├── api/
 ├── pages/
 ├── components/
 └── index.js
```

[⬆ Back to Top](#top)

---

## Skeleton Code Multiple Slices per Feature

Large features **should** have multiple slices:

```txt
features/users/slices/
 ├── usersListSlice.js
 ├── userDetailsSlice.js
 ├── userPermissionsSlice.js
 └── userFiltersSlice.js
```

Combine them in `features/users/index.js`:

```js
import { combineReducers } from "@reduxjs/toolkit";

import list from "./slices/usersListSlice";
import details from "./slices/userDetailsSlice";
import permissions from "./slices/userPermissionsSlice";
import filters from "./slices/userFiltersSlice";

export default combineReducers({
  list,
  details,
  permissions,
  filters,
});
```

[⬆ Back to Top](#top)

---

## Skeleton Code Final State Shape

```js
{
  users: {
    list: {},
    details: {},
    permissions: {},
    filters: {}
  },
  payments: {},
  orders: {},
  auth: {},
  ui: {}
}
```

Clean. Predictable. Scalable.

[⬆ Back to Top](#top)

---

## What State Goes Where

| State Type         | Where It Belongs             | Examples                                   |
| ------------------ | ---------------------------- | ------------------------------------------ |
| **Server data**    | Caching library or store     | User list, product catalog, orders         |
| **Shared UI**      | Global store (`shared/`)     | Modals, toasts, sidebar state              |
| **Feature state**  | Feature's own slice          | Filters, pagination, selected item         |
| **Form state**     | Local component or form lib  | Input values, validation errors            |
| **URL state**      | Router / URL params          | Current page, search query                 |
| **Derived data**   | Selectors (never stored!)    | Filtered list, total count, active items   |
| **Local UI**       | Component's own local state  | Hover, dropdown open, animation progress   |

> 💡 **#1 Mistake:** Putting everything in the global store. If state doesn't need to be shared, keep it local.

[⬆ Back to Top](#top)

---

## Normalize Your Data

| Approach       | Find by ID | Update One | Good For                   |
| -------------- | ---------- | ---------- | -------------------------- |
| **Array**      | O(n) scan  | O(n) map   | Small, rarely-updated lists|
| **Normalized** | O(1)       | O(1)       | Large lists, frequent updates|

**Normalized shape:**

```
{
  ids:      ['1', '2', '3'],
  entities: {
    '1': { id: '1', name: 'Alice' },
    '2': { id: '2', name: 'Bob' },
  }
}
```

**Normalize when:** List has 50+ items, items are updated individually, or same entity appears in multiple views.

[⬆ Back to Top](#top)

---

## Selectors How to Read State

| Level                | What                                   | Memoized? |
| -------------------- | -------------------------------------- | --------- |
| **Simple access**    | Direct path: get loading flag          | No        |
| **Derived data**     | Filtered/sorted/computed results       | Yes       |
| **Cross-feature**    | Combine data from 2+ features          | Yes       |

**Rules:**
- Components never hardcode state paths — always use named selectors
- Memoize anything that filters, maps, or computes
- Never create selectors inside render functions (kills memoization)
- Minimize cross-feature selectors (breaks isolation)

[⬆ Back to Top](#top)

---

## How Features Should Communicate

| Pattern                    | How                                              | When                                |
| -------------------------- | ------------------------------------------------ | ----------------------------------- |
| ✅ **Shared events**       | Global action multiple features listen to         | Logout → all features reset         |
| ✅ **Read-only selectors** | Feature B reads Feature A's exported selector     | Orders page shows user names        |
| ✅ **Listener middleware** | Middleware reacts to action and dispatches another | Order completed → refresh dashboard |
| ❌ **Direct imports**      | Feature A imports Feature B's internal actions     | NEVER — breaks isolation            |

[⬆ Back to Top](#top)

---

## When to Split a Slice

| Signal                                   | Action                              |
| ---------------------------------------- | ----------------------------------- |
| Slice exceeds **300 lines**              | Split by sub-domain                 |
| Manages **multiple unrelated screens**   | One slice per screen's unique state |
| Mixes **UI state + domain data**         | Separate into two slices            |
| Different parts update at **different rates** | Separate to prevent re-renders |
| Different **teams** own different parts  | One slice per team                  |

The root reducer sees one entry per feature. It doesn't know about the internal splits.

[⬆ Back to Top](#top)

---

## Async Data Strategy

| Pattern                | Best For                           | Caching | Auto Refetch |
| ---------------------- | ---------------------------------- | ------- | ------------ |
| **Caching library**    | Standard CRUD, server-heavy apps   | ✅ Auto  | ✅ Auto       |
| **Async thunks**       | Complex multi-step workflows       | Manual  | Manual       |
| **Listener middleware**| Cross-feature reactions            | N/A     | N/A          |

**Key decisions:**
- Separate server state from client state
- Define loading/error per operation (not one global flag)
- Plan cache invalidation — when does data go stale?

[⬆ Back to Top](#top)

---

## Enterprise Rules

| Rule                                          | Why                                                |
| --------------------------------------------- | -------------------------------------------------- |
| One feature = one state domain                | Clear ownership, no conflicts                      |
| Use modern slice APIs (not manual switch/case)| Less boilerplate, built-in immutability            |
| Read state via named selectors only           | Components don't break if state shape changes      |
| No cross-feature internal imports             | Enforce with ESLint boundaries plugin              |
| Shared state ≤ 5 slices                       | auth, UI, config, theme, notifications — that's it|
| Max 300 lines per slice                       | Forces proper splitting                            |
| Features export a public API via index file   | Internals stay private                             |
| Normalize entity lists                        | O(1) lookups, no duplicate data                    |
| Never store derived values                    | Compute in selectors to avoid stale data           |

[⬆ Back to Top](#top)

---

## Migrating Legacy State

| Step | Action                              | Details                                          |
| ---- | ----------------------------------- | ------------------------------------------------ |
| 1    | Audit                               | Count lines per reducer, action types, coupling  |
| 2    | Set up modern tooling alongside     | New store config accepts both old and new reducers|
| 3    | New features → modern pattern       | Every new feature uses feature-sliced structure   |
| 4    | Migrate pain points first           | Bug-heavy or frequently-modified reducers (highest ROI) |
| 5    | One feature at a time               | Create folder → rewrite → update components → delete old |
| 6    | Remove dead state                   | Audit for state keys nothing reads anymore       |

> Never do a big-bang rewrite. Migrate incrementally.

[⬆ Back to Top](#top)

---

## Strategy Comparison

| Strategy               | Best For                       | Boilerplate | Learning Curve |
| ---------------------- | ------------------------------ | ----------- | -------------- |
| **Redux Toolkit**      | Large enterprise, multi-team   | Low         | Medium         |
| **RTK Query**          | Server-heavy CRUD apps         | Very Low    | Medium         |
| **useReducer+Context** | Medium apps, no deps           | Medium      | Low            |
| **Zustand**            | Medium-large, simple API       | Very Low    | Low            |
| **Jotai**              | Fine-grained reactivity        | Minimal     | Low            |
| **NgRx** (Angular)     | Enterprise Angular              | High        | High           |
| **Pinia** (Vue)        | Vue 3 apps                     | Low         | Low            |

[⬆ Back to Top](#top)

---

## Checklist

- [ ] State domains identified and mapped to feature folders
- [ ] State classified (server / client / form / URL / derived / local)
- [ ] Feature-sliced folder structure in place
- [ ] State is flat, normalized for entity lists
- [ ] All state reads go through named selectors
- [ ] Derived data computed in selectors, not stored
- [ ] Async strategy chosen (caching lib vs thunks)
- [ ] Cross-feature communication uses shared events, not internal imports
- [ ] Shared state limited to auth, UI, config
- [ ] Slices < 300 lines; split when needed
- [ ] Code-split reducers for lazy-loaded routes
- [ ] DevTools configured for development
- [ ] Architecture rules documented for the team

[⬆ Back to Top](#top)

---

## Interview Takeaways

| Topic                       | Key Point                                                                        |
| --------------------------- | -------------------------------------------------------------------------------- |
| Why architecture?           | Without it → God reducer, merge conflicts, untestable, can't code-split          |
| Feature-sliced design       | Each feature owns its slices, selectors, API. Self-contained and deletable.      |
| State classification        | Server → caching lib. Form → local. URL → router. Derived → selectors. Shared → store. |
| Normalization               | `{ ids, entities }` → O(1) lookups, no duplicates                               |
| Selectors                   | Memoized, named, never inline. Derive computed data, never store it.             |
| Cross-feature communication | Shared events + read-only selectors. Never import another feature's internals.   |
| When to split slices        | > 300 lines, multiple screens, mixed concerns, different update frequencies.     |
| Migration                   | Incremental. New features → modern. Bug-heavy old reducers → migrate first.      |
| #1 mistake                  | Putting everything in the global store. If it's not shared, keep it local.       |

[⬆ Back to Top](#top)

---

## Further Reading

| Resource                            | Link                                                              |
| ----------------------------------- | ----------------------------------------------------------------- |
| Redux Toolkit Docs                  | https://redux-toolkit.js.org/                                     |
| Redux Style Guide                   | https://redux.js.org/style-guide/                                 |
| Feature-Sliced Design               | https://feature-sliced.design/                                    |
| Normalizing State Shape             | https://redux.js.org/usage/structuring-reducers/normalizing-state-shape |
| NgRx (Angular)                      | https://ngrx.io/                                                  |
| Pinia (Vue)                         | https://pinia.vuejs.org/                                          |
| Zustand                             | https://github.com/pmndrs/zustand                                 |

---

> 🏁 **Large-scale reducer design = feature boundaries + flat normalized state + memoized selectors + minimal shared state.** The library doesn't matter — these principles do.


More Details:

Get all articles related to system design 
Hastag: SystemDesignWithZeeshanAli


[systemdesignwithzeeshanali](https://dev.to/t/systemdesignwithzeeshanali)

Git: https://github.com/ZeeshanAli-0704/front-end-system-design

[⬆ Back to Top](#top)
