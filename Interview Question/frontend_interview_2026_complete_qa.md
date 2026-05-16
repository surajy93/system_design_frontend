# Frontend Developer Interview 2026 — Complete Q&A Guide

> **40+ questions** with in-depth answers, real examples, and code snippets.  
> Organized by category. Use this to prepare, study, or evaluate candidates.

---

## Table of Contents

1. [JavaScript](#javascript)
2. [TypeScript](#typescript)
3. [CSS](#css)
4. [React](#react)
5. [Angular](#angular)
6. [Performance](#performance)
7. [System Design](#system-design)
8. [AI & Dev Tools](#ai--dev-tools)
9. [Web APIs & Accessibility](#web-apis--accessibility)

---

## JavaScript

---

### Q1. Walk me through the JavaScript event loop — what happens when an async function and a setTimeout(0) are queued together?

**Difficulty:** Mid

#### Answer

JavaScript is single-threaded. The event loop is the mechanism that allows it to handle asynchronous operations without blocking the main thread.

**The key mental model:**

```
Call Stack → Microtask Queue → Macrotask Queue (Task Queue)
```

- **Call Stack** — where synchronous code executes, one frame at a time.
- **Microtask Queue** — holds Promise callbacks, `queueMicrotask()`, `MutationObserver` callbacks.
- **Macrotask Queue** — holds `setTimeout`, `setInterval`, `setImmediate`, I/O callbacks, UI render tasks.

**The Rule:** After every macrotask completes, the engine drains the **entire** microtask queue before moving to the next macrotask (or rendering).

#### Example

```js
console.log("1 - sync start");

setTimeout(() => console.log("4 - setTimeout(0)"), 0);

Promise.resolve().then(() => console.log("3 - microtask"));

console.log("2 - sync end");

// Output order:
// 1 - sync start
// 2 - sync end
// 3 - microtask   ← microtask queue drains before setTimeout
// 4 - setTimeout(0)
```

#### With async/await

```js
async function fetchData() {
  console.log("A - async fn starts (sync)");
  await Promise.resolve();            // suspends here, schedules microtask
  console.log("C - after await");     // microtask callback
}

console.log("1 - before call");
fetchData();
setTimeout(() => console.log("D - setTimeout"), 0);
console.log("B - after call (sync)");

// Output:
// 1 - before call
// A - async fn starts (sync)
// B - after call (sync)
// C - after await      ← microtask runs before setTimeout
// D - setTimeout
```

**Key insight:** `async/await` is syntactic sugar over Promises. Every `await` suspends the function and schedules the remainder as a microtask — meaning it always runs before any `setTimeout(0)`.

#### Real-world implication

In a React app, if you call `setState` inside a `setTimeout` vs inside a microtask (Promise), the batching behavior differs — React 18 unified batching across both, but understanding the order helps debug subtle state update timing bugs.

---

### Q2. How does garbage collection work in V8, and what patterns cause memory leaks in SPAs?

**Difficulty:** Hard

#### Answer

V8 uses a **generational garbage collector**:

- **Young Generation (Scavenger)** — short-lived objects. GC runs frequently (minor GC). Survives 2 cycles → promoted.
- **Old Generation (Mark-Sweep + Mark-Compact)** — long-lived objects. Major GC runs less often but is more expensive.

V8 uses **tri-color mark-and-sweep**: marks objects as white (unvisited), grey (reachable but children not checked), black (reachable + children checked). Unreachable (white) objects are swept.

#### Common SPA Memory Leak Patterns

**1. Event listeners not removed**

```js
// ❌ Leak — listener added on mount, never removed
function setupComponent(el) {
  window.addEventListener("resize", () => {
    el.style.width = window.innerWidth + "px";
  });
}

// ✅ Fix
function setupComponent(el) {
  const handler = () => { el.style.width = window.innerWidth + "px"; };
  window.addEventListener("resize", handler);
  return () => window.removeEventListener("resize", handler); // cleanup
}
```

**2. Closures holding DOM references**

```js
// ❌ bigData is kept alive because the closure references it
function createLeak() {
  const bigData = new Array(1_000_000).fill("x");
  const btn = document.getElementById("btn");
  btn.addEventListener("click", () => {
    console.log(bigData.length); // bigData can't be GC'd
  });
}
```

**3. Detached DOM nodes**

```js
// ❌ Node removed from DOM but still referenced in JS
const cache = {};
function cacheNode() {
  const node = document.createElement("div");
  document.body.appendChild(node);
  cache.myNode = node;            // strong reference
  document.body.removeChild(node); // removed from DOM but NOT GC'd
}
```

**4. Timers not cleared**

```js
// ❌ Interval keeps running even after component unmounts
setInterval(() => updateDashboard(), 1000);

// ✅ Fix in React
useEffect(() => {
  const id = setInterval(() => updateDashboard(), 1000);
  return () => clearInterval(id);
}, []);
```

**5. Global state accumulation (Redux, Zustand)**

In long-running apps, pushing to global arrays/maps without pruning causes old generation GC pressure.

#### Detection Tools

- Chrome DevTools → Memory → Heap Snapshot (compare before/after navigation)
- `performance.measureUserAgentSpecificMemory()` (Chrome 89+)
- Search for "Detached HTMLElement" in heap snapshots

---

### Q3. Difference between structuredClone(), JSON parse/stringify, and spread — when does each break?

**Difficulty:** Mid

#### Answer

| Method | Deep? | Circular refs | Date | Map/Set | undefined | Functions |
|--------|-------|--------------|------|---------|-----------|-----------|
| Spread `{...obj}` | ❌ Shallow | ✅ ok | ✅ ok | ✅ ok | ✅ ok | ✅ ok |
| `JSON.parse(JSON.stringify())` | ✅ Deep | ❌ throws | ❌ becomes string | ❌ becomes `{}` | ❌ dropped | ❌ dropped |
| `structuredClone()` | ✅ Deep | ✅ ok | ✅ ok | ✅ ok | ✅ ok | ❌ throws |

#### Examples

```js
// --- Spread (shallow) ---
const obj = { a: 1, nested: { b: 2 } };
const copy = { ...obj };
copy.nested.b = 99;
console.log(obj.nested.b); // 99 — same reference!

// --- JSON round-trip ---
const data = {
  name: "Alice",
  created: new Date(),      // ❌ becomes string "2024-..."
  tags: new Set([1,2,3]),   // ❌ becomes {}
  greet: () => "hello",     // ❌ dropped silently
  value: undefined          // ❌ dropped silently
};
const clone = JSON.parse(JSON.stringify(data));
console.log(clone.created); // string, not Date object
console.log(clone.tags);    // {}

// --- structuredClone (best for data) ---
const original = {
  date: new Date(),
  map: new Map([["key", "val"]]),
  set: new Set([1, 2, 3]),
  nested: { arr: [1, 2, 3] }
};
const deep = structuredClone(original);
deep.nested.arr.push(4);
console.log(original.nested.arr); // [1, 2, 3] — truly independent

// structuredClone with circular ref — works!
const circular = { self: null };
circular.self = circular;
const cloned = structuredClone(circular); // ✅ no error

// structuredClone with function — throws!
structuredClone({ fn: () => {} }); // ❌ DataCloneError
```

#### Real-world guidance

- **State updates in React/Redux** → spread is fine for shallow, `structuredClone` for deep nested objects.
- **API payloads** → JSON round-trip is acceptable if you know the shape contains only serializable primitives.
- **Web Worker messages** → `structuredClone` is used internally for `postMessage`.
- **Never use JSON round-trip for Dates** in forms — you'll lose the Date object.

---

### Q4. What is a closure — give a real-world component example?

**Difficulty:** Easy

#### Answer

A closure is a function that **remembers the variables from the scope in which it was defined**, even after that outer scope has exited.

#### Real-world example: Debounce

```js
function debounce(fn, delay) {
  let timerId;           // this variable is "closed over"
  return function(...args) {
    clearTimeout(timerId);
    timerId = setTimeout(() => fn.apply(this, args), delay);
  };
}

// Usage in a search input
const handleSearch = debounce((query) => {
  fetchResults(query);
}, 300);

input.addEventListener("input", (e) => handleSearch(e.target.value));
// timerId persists between calls because of closure
```

#### Another example: Memoization

```js
function memoize(fn) {
  const cache = new Map(); // closed over
  return function(key) {
    if (cache.has(key)) return cache.get(key);
    const result = fn(key);
    cache.set(key, result);
    return result;
  };
}

const expensiveCalc = memoize((n) => n * n);
expensiveCalc(5); // computes
expensiveCalc(5); // returns from cache
```

#### The classic loop gotcha

```js
// ❌ All timeouts print 3
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}

// ✅ Fix with let (block-scoped, creates new binding per iteration)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100); // 0, 1, 2
}
```

---

### Q5. WeakRef and FinalizationRegistry — frontend use cases?

**Difficulty:** Hard

#### Answer

`WeakRef` holds a **weak reference** to an object — it doesn't prevent GC from collecting that object.  
`FinalizationRegistry` lets you register a callback to be called **after** an object is garbage collected.

```js
const ref = new WeakRef(someObject);
const target = ref.deref(); // returns object or undefined if GC'd
```

#### Frontend use case: Cache that doesn't cause memory leaks

```js
class WeakCache {
  #cache = new Map();
  #registry = new FinalizationRegistry((key) => {
    this.#cache.delete(key); // auto-cleanup when value is GC'd
    console.log(`Cache entry "${key}" was garbage collected`);
  });

  set(key, value) {
    this.#registry.register(value, key);
    this.#cache.set(key, new WeakRef(value));
  }

  get(key) {
    const ref = this.#cache.get(key);
    return ref?.deref(); // undefined if GC'd
  }
}

const cache = new WeakCache();

let largeComponent = { data: new Array(100_000).fill("x") };
cache.set("dashboard", largeComponent);

cache.get("dashboard"); // returns object

largeComponent = null; // now eligible for GC
// Eventually: "Cache entry 'dashboard' was garbage collected"
```

#### Use case: DOM element tracking without memory pressure

```js
const elementRefs = new Map();

function trackElement(id, el) {
  elementRefs.set(id, new WeakRef(el));
}

function getElement(id) {
  return elementRefs.get(id)?.deref() ?? null; // null if DOM node was removed
}
```

**Important caveat:** GC timing is non-deterministic — never rely on FinalizationRegistry for business-critical logic, only for cleanup/optimization.

---

### Q6. Implement debounce with leading-edge support

**Difficulty:** Mid

#### Answer

```ts
function debounce<T extends (...args: unknown[]) => unknown>(
  fn: T,
  delay: number,
  options: { leading?: boolean; trailing?: boolean } = { trailing: true }
): (...args: Parameters<T>) => void {
  let timerId: ReturnType<typeof setTimeout> | null = null;
  let leadingCalled = false;

  return function (this: unknown, ...args: Parameters<T>) {
    const isLeading = options.leading && !timerId;

    if (timerId) clearTimeout(timerId);

    timerId = setTimeout(() => {
      timerId = null;
      leadingCalled = false;
      if (options.trailing && !isLeading) {
        fn.apply(this, args);
      }
    }, delay);

    if (isLeading) {
      leadingCalled = true;
      fn.apply(this, args);
    }
  };
}

// Usage:
// Trailing only (default) — fires after user stops typing
const handleSearch = debounce(fetchResults, 300);

// Leading — fires immediately on first call, then waits
const handleClick = debounce(submitForm, 500, { leading: true, trailing: false });

// Both — fires immediately AND at end of burst
const handleScroll = debounce(logScroll, 200, { leading: true, trailing: true });
```

**Edge cases covered:**
- `this` context preserved via `.apply(this, args)`
- TypeScript generics preserve argument types
- Leading fires on first call in a burst
- Trailing fires after silence
- Both can coexist

---

## TypeScript

---

### Q7. `interface` vs `type` in TypeScript 5.x — when to prefer which?

**Difficulty:** Mid

#### Answer

| Feature | `interface` | `type` |
|---------|------------|--------|
| Declaration merging | ✅ Yes | ❌ No |
| Extends | ✅ `extends` | ✅ `&` intersection |
| Union types | ❌ No | ✅ Yes |
| Computed properties | ❌ Limited | ✅ Full |
| Mapped types | ❌ No | ✅ Yes |
| Recursive types | ✅ Yes | ✅ Yes |
| Error messages | Clearer for objects | Can be verbose |

#### When to use `interface`

```ts
// ✅ Use interface for object shapes you expect to extend
interface User {
  id: string;
  name: string;
}

interface Admin extends User {
  role: "admin";
  permissions: string[];
}

// ✅ Declaration merging — crucial for augmenting third-party types
interface Window {
  myAnalytics: Analytics; // adds to existing Window interface
}
```

#### When to use `type`

```ts
// ✅ Unions — interface can't do this
type Status = "idle" | "loading" | "success" | "error";

// ✅ Computed / mapped types
type ReadOnly<T> = { readonly [K in keyof T]: T[K] };

// ✅ Utility composition
type ApiResponse<T> = { data: T; status: number; message: string };
type UserResponse = ApiResponse<User>;

// ✅ Conditional types
type NonNullable<T> = T extends null | undefined ? never : T;

// ✅ Template literal types
type EventName = "click" | "focus" | "blur";
type Handler = `on${Capitalize<EventName>}`; // "onClick" | "onFocus" | "onBlur"
```

**Rule of thumb (2026):**
- Public library APIs → `interface` (consumers can extend via merging)
- Internal app types, utilities, unions → `type`

---

### Q8. DeepPartial utility type — recursive generics

**Difficulty:** Hard

#### Answer

```ts
// Built-in Partial only goes one level deep
type Partial<T> = { [K in keyof T]?: T[K] };

// DeepPartial — recursively makes all nested properties optional
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

// Usage
interface UserSettings {
  theme: {
    color: string;
    fontSize: number;
    dark: boolean;
  };
  notifications: {
    email: boolean;
    push: { enabled: boolean; frequency: "daily" | "weekly" };
  };
}

// With built-in Partial — only top level optional
type ShallowPartial = Partial<UserSettings>;
// theme, notifications optional — but if theme provided, ALL sub-fields required

// With DeepPartial — every level optional
type DeepUserSettings = DeepPartial<UserSettings>;
// Can do: { theme: { color: "blue" } } — omitting fontSize, dark, notifications

function updateSettings(patch: DeepPartial<UserSettings>) {
  // merge patch into current settings
}

updateSettings({ theme: { color: "blue" } }); // ✅ valid
updateSettings({ notifications: { push: { enabled: true } } }); // ✅ valid
```

#### Extended version handling arrays

```ts
type DeepPartial<T> =
  T extends Array<infer U>
    ? Array<DeepPartial<U>>
    : T extends object
      ? { [K in keyof T]?: DeepPartial<T[K]> }
      : T;
```

**TypeScript 4.1+ note:** Recursive conditional types are now officially supported with lazy evaluation, so this pattern is safe.

---

### Q9. Template literal types — practical use cases

**Difficulty:** Mid

#### Answer

```ts
// Basic syntax
type Greeting = `Hello, ${string}`;

// ---- Use case 1: Type-safe event system ----
type EventName = "click" | "focus" | "blur" | "change";
type Handler = `on${Capitalize<EventName>}`;
// = "onClick" | "onFocus" | "onBlur" | "onChange"

type EventMap = {
  [K in Handler]: (event: Event) => void;
};

// ---- Use case 2: CSS custom property names ----
type ColorToken = "primary" | "secondary" | "danger" | "success";
type CSSVar = `--color-${ColorToken}`;
// = "--color-primary" | "--color-secondary" | ...

function setCSSVar(name: CSSVar, value: string) {
  document.documentElement.style.setProperty(name, value);
}

setCSSVar("--color-primary", "#7F77DD"); // ✅
setCSSVar("--color-unknown", "#000");    // ❌ TypeScript error

// ---- Use case 3: API route typing ----
type HttpMethod = "get" | "post" | "put" | "delete";
type Resource = "users" | "posts" | "comments";
type Route = `/${Resource}` | `/${Resource}/${string}`;

function apiCall(method: HttpMethod, route: Route) {
  return fetch(route, { method: method.toUpperCase() });
}

apiCall("get", "/users");      // ✅
apiCall("post", "/users/123"); // ✅
apiCall("get", "/unknown");    // ❌ TypeScript error

// ---- Use case 4: Deep key paths (simplified) ----
type Paths<T> = T extends object
  ? { [K in keyof T]: K extends string
      ? T[K] extends object
        ? K | `${K}.${Paths<T[K]>}`
        : K
      : never
    }[keyof T]
  : never;

interface Config { server: { host: string; port: number }; debug: boolean }
type ConfigPaths = Paths<Config>;
// = "server" | "server.host" | "server.port" | "debug"
```

---

### Q10. TypeScript `satisfies` operator vs type assertion

**Difficulty:** Easy

#### Answer

```ts
// --- Type assertion (as) ---
// Widens or forces the type, loses literal narrowing
const palette = {
  red: [255, 0, 0],
  green: "#00ff00",
} as Record<string, string | number[]>;

palette.red; // type: string | number[]
// Now you've lost the fact that red is number[] — TypeScript thinks it could be a string

// --- satisfies operator (TypeScript 4.9+) ---
// Validates against a type BUT preserves the actual inferred literal type
const palette2 = {
  red: [255, 0, 0],
  green: "#00ff00",
} satisfies Record<string, string | number[]>;

palette2.red;   // type: number[] ✅ — literal type preserved
palette2.green; // type: string ✅ — literal type preserved

// So you can safely call array methods on red:
palette2.red.at(0); // ✅ works — TypeScript knows it's number[]

// --- Real-world: config objects ---
type Config = {
  env: "development" | "production" | "test";
  port: number;
  features: Record<string, boolean>;
};

// ❌ assertion loses literal type
const config1 = {
  env: "development",
  port: 3000,
  features: { darkMode: true }
} as Config;

// ✅ satisfies validates shape AND preserves literals
const config2 = {
  env: "development", // type: "development" (not string)
  port: 3000,
  features: { darkMode: true }
} satisfies Config;

// config2.env narrows to "development" specifically — useful for exhaustive checks
```

**When to use `satisfies`:** Config objects, style tokens, route maps — anywhere you want type validation without losing narrowed literal types.

---

## CSS

---

### Q11. CSS containment (`contain` property) and rendering performance

**Difficulty:** Mid

#### Answer

CSS `contain` tells the browser: "changes inside this element won't affect anything outside it." This enables the browser to skip layout/paint calculations for the rest of the page.

```css
/* Values */
.box {
  contain: none;         /* default */
  contain: layout;       /* internal layout doesn't affect outside */
  contain: paint;        /* children clipped to border-box, not painted outside */
  contain: size;         /* element's size doesn't depend on children */
  contain: style;        /* counters/quotes scoped inside */
  contain: strict;       /* layout + paint + size */
  contain: content;      /* layout + paint (most useful) */
}
```

#### Real-world example: Card grid

```css
/* Without containment — changing one card triggers full-page layout recalc */
.card { background: white; padding: 1rem; }

/* With containment — browser can isolate each card */
.card {
  background: white;
  padding: 1rem;
  contain: content; /* layout + paint */
}
```

#### With container queries (2025+ baseline)

```css
.card-wrapper {
  container-type: inline-size; /* implicitly applies layout containment */
  container-name: card;
}

@container card (min-width: 400px) {
  .card-title { font-size: 1.5rem; }
}
```

`container-type: inline-size` automatically implies `contain: layout style inline-size` — so container queries have containment built in.

#### Performance impact

In a list of 500 items, without containment: updating item #3's text could trigger layout recalc for all 500. With `contain: content`: only item #3 is recalculated.

---

### Q12. CSS Grid vs Flexbox — practical decision guide

**Difficulty:** Easy

#### Answer

**Flexbox** = 1D layout (row OR column)  
**Grid** = 2D layout (rows AND columns simultaneously)

```css
/* ---- Flexbox: navigation bar ---- */
nav {
  display: flex;
  align-items: center;
  gap: 16px;
}
/* Items flow in a single row — classic flexbox use case */

/* ---- Flexbox nightmare: card grid ---- */
/* ❌ Trying to make 3-column grid with flex — last row breaks */
.card-grid { display: flex; flex-wrap: wrap; }
.card { width: calc(33.33% - 16px); margin: 8px; }
/* Last row has 1-2 cards that stretch oddly */

/* ---- Grid saves you ---- */
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 16px;
}
/* Columns auto-adjust. Last row items keep their size. No math needed. */
```

#### Subgrid — 2026 baseline

```css
/* ---- Subgrid: align card internals across rows ---- */
.card-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: auto;
}

.card {
  display: grid;
  grid-row: span 3; /* each card spans 3 rows: header, body, footer */
  grid-template-rows: subgrid; /* aligns to PARENT grid rows */
}

/* Now all card headers are at exactly the same vertical position
   regardless of content length — previously impossible with flex */
```

**Decision rule:**
- Axis-agnostic alignment of equal items? → **Grid**
- Single-direction flow with flexible item sizes? → **Flexbox**
- Complex 2D page layout? → **Grid**
- Centering a single element? → Either (Flexbox is simpler)

---

### Q13. CSS custom properties vs Sass variables — dynamic theming system

**Difficulty:** Hard

#### Answer

| Feature | CSS Custom Properties | Sass Variables |
|---------|----------------------|----------------|
| Runtime | ✅ Live, changeable | ❌ Compiled away |
| Cascade/Inherit | ✅ Yes | ❌ No |
| JS access | ✅ `getPropertyValue`, `setProperty` | ❌ No |
| Browser support | ✅ All modern | ✅ (compiled) |

#### Building a native CSS theming system

```css
/* 1. Define tokens on :root (light theme default) */
:root {
  --color-bg: #ffffff;
  --color-text: #1a1a1a;
  --color-primary: #7F77DD;
  --color-surface: #f5f5f5;
  --spacing-md: 1rem;
  --radius-md: 8px;
}

/* 2. Dark theme override via class */
[data-theme="dark"] {
  --color-bg: #1a1a1a;
  --color-text: #f0f0f0;
  --color-primary: #AFA9EC;
  --color-surface: #2c2c2c;
}

/* 3. Components use variables — no theme-specific rules needed */
.card {
  background: var(--color-surface);
  color: var(--color-text);
  border-radius: var(--radius-md);
  padding: var(--spacing-md);
}

.btn-primary {
  background: var(--color-primary);
  color: white;
}
```

```js
// 4. JavaScript theme switching — instant, no re-render
function setTheme(theme) {
  document.documentElement.setAttribute("data-theme", theme);
  localStorage.setItem("theme", theme);
}

// 5. Runtime token override (e.g., brand customization)
function setBrandColor(hex) {
  document.documentElement.style.setProperty("--color-primary", hex);
}

// 6. Read current value
const primary = getComputedStyle(document.documentElement)
  .getPropertyValue("--color-primary").trim();
```

```css
/* 7. Component-scoped overrides — cascade works! */
.dark-card {
  --color-surface: #2c2c2c; /* overrides only within this component */
  --color-text: #f0f0f0;
  background: var(--color-surface);
  color: var(--color-text);
}

/* 8. Using @media prefers-color-scheme as fallback */
@media (prefers-color-scheme: dark) {
  :root:not([data-theme]) {
    --color-bg: #1a1a1a;
    --color-text: #f0f0f0;
  }
}
```

---

### Q14. CSS Container Queries — how they change component architecture

**Difficulty:** Mid

#### Answer

Media queries respond to the **viewport size**. Container queries respond to the **container's size** — enabling truly component-driven responsive design.

```css
/* Old way — component layout tied to viewport */
@media (max-width: 768px) {
  .product-card { flex-direction: column; } /* assumes card is in full-width viewport */
}

/* New way — component responds to its own container */
.card-wrapper {
  container-type: inline-size;
  container-name: product-card;
}

.product-card { display: flex; flex-direction: row; }

@container product-card (max-width: 400px) {
  .product-card { flex-direction: column; } /* responds to wrapper width, not viewport */
}
```

#### Real power: same component, different contexts

```html
<!-- Sidebar (narrow container) -->
<aside class="sidebar">           <!-- container-type: inline-size -->
  <ProductCard />                 <!-- renders as stacked/vertical -->
</aside>

<!-- Main content (wide container) -->
<main class="content">           <!-- container-type: inline-size -->
  <ProductCard />                 <!-- same component, renders as horizontal -->
</main>
```

```css
/* One set of rules handles both layouts automatically */
@container (min-width: 500px) {
  .product-card { flex-direction: row; }
  .product-card img { width: 200px; }
}

@container (max-width: 499px) {
  .product-card { flex-direction: column; }
  .product-card img { width: 100%; }
}
```

#### Container query units

```css
/* cqi = 1% of container's inline size */
.product-card { font-size: clamp(0.875rem, 3cqi, 1.25rem); }
```

**Architecture impact:** Components are now truly self-contained responsive units. No more tracking viewport breakpoints per-component. This pairs perfectly with design system component libraries.

---

### Q15. CSS Houdini — APIs and performance use cases

**Difficulty:** Hard

#### Answer

CSS Houdini exposes browser rendering engine hooks to JavaScript, allowing custom CSS properties, layouts, and animations that run **off the main thread**.

#### Key APIs

**1. CSS Paint API (Houdini Paint Worklet)**
```js
// paint-worklet.js — runs in a separate thread
registerPaint("fancy-border", class {
  static get inputProperties() { return ["--border-color", "--border-width"]; }

  paint(ctx, geom, props) {
    const color = props.get("--border-color").toString();
    const width = parseInt(props.get("--border-width"));

    ctx.strokeStyle = color;
    ctx.lineWidth = width;

    // Draw a custom dashed animated border
    ctx.setLineDash([10, 5]);
    ctx.strokeRect(0, 0, geom.width, geom.height);
  }
});
```

```js
// Register worklet
CSS.paintWorklet.addModule("paint-worklet.js");
```

```css
/* Use as a CSS value */
.fancy-box {
  --border-color: #7F77DD;
  --border-width: 3;
  border: 3px solid transparent;
  background-image: paint(fancy-border);
}

/* Animate using CSS custom property — no JS animation loop! */
@keyframes shift-dash {
  from { --dash-offset: 0; }
  to   { --dash-offset: 30; }
}
.fancy-box { animation: shift-dash 1s linear infinite; }
```

**2. CSS Properties and Values API**
```js
// Teach the browser what type a custom property is
CSS.registerProperty({
  name: "--gradient-angle",
  syntax: "<angle>",  // browser understands it's an angle — enables interpolation
  inherits: false,
  initialValue: "0deg"
});
```

```css
/* Now this transition actually animates (without registerProperty, it snaps) */
.box {
  background: conic-gradient(from var(--gradient-angle), #7F77DD, #5DCAA5);
  transition: --gradient-angle 0.5s ease;
}
.box:hover { --gradient-angle: 360deg; }
```

**Performance win:** Paint Worklets run on the compositor thread, not the main thread. Complex custom drawing (patterns, sparklines, waveforms) that would require Canvas on the main thread can be offloaded to Houdini Paint, keeping the main thread free for JS execution.

---

## React

---

### Q16. React 19 concurrent rendering and useTransition

**Difficulty:** Mid

#### Answer

React's concurrent mode makes rendering **interruptible**. React can start rendering, pause to handle urgent updates (like user input), and resume.

**The problem without concurrency:**
```js
// Heavy render blocks input — typing feels laggy
function FilteredList({ items }) {
  const [query, setQuery] = useState("");
  const filtered = items.filter(i => i.name.includes(query)); // 10,000 items

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <List items={filtered} /> {/* expensive — blocks input */}
    </>
  );
}
```

**Fix with useTransition:**
```js
import { useState, useTransition, useDeferredValue } from "react";

function FilteredList({ items }) {
  const [query, setQuery] = useState("");
  const [isPending, startTransition] = useTransition();
  const [deferredQuery, setDeferredQuery] = useState("");

  function handleChange(e) {
    const value = e.target.value;
    setQuery(value); // urgent — updates input immediately

    startTransition(() => {
      setDeferredQuery(value); // non-urgent — can be interrupted
    });
  }

  const filtered = items.filter(i => i.name.includes(deferredQuery));

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <List items={filtered} style={{ opacity: isPending ? 0.6 : 1 }} />
    </>
  );
}
```

**How it works under the hood:**
- React maintains a **priority lane system**. User input = SyncLane (highest). `startTransition` = TransitionLane (lower priority).
- When a high-priority update arrives, React **interrupts** the low-priority render, handles the urgent update, then resumes the transition render.
- The UI never freezes because React yields to the browser between work chunks (using `MessageChannel` / `requestIdleCallback`-like scheduling).

#### useDeferredValue vs useTransition

```js
// useDeferredValue — defer a value without controlling when it changes
const deferredQuery = useDeferredValue(query);
// React keeps showing old results while new ones render in background

// useTransition — wrap a state setter to mark it as non-urgent
const [isPending, startTransition] = useTransition();
startTransition(() => setFilteredData(newData));
// You get isPending flag — useful for loading UI
```

---

### Q17. React Server Components vs Client Components

**Difficulty:** Hard

#### Answer

```
┌─────────────────────────────────────────────────────┐
│                    SERVER COMPONENTS                │
│  - Render on server (Node/Edge runtime)             │
│  - Zero JS shipped to client                        │
│  - Direct DB/filesystem access                      │
│  - No useState, useEffect, event handlers           │
│  - Can fetch data directly (async/await in render)  │
└─────────────────────────────────────────────────────┘
                        │ passes serialized props
┌─────────────────────────────────────────────────────┐
│                    CLIENT COMPONENTS                │
│  'use client' directive at top of file              │
│  - Render on server (for HTML) + hydrate on client  │
│  - useState, useEffect, event handlers ✅           │
│  - Adds JS to the bundle                            │
└─────────────────────────────────────────────────────┘
```

#### Real example: Product page

```jsx
// app/products/[id]/page.tsx — Server Component (default)
// No "use client" = server component
import { ProductDetails } from "./ProductDetails"; // server component
import { AddToCartButton } from "./AddToCartButton"; // client component

async function ProductPage({ params }) {
  // Direct DB call — zero API layer needed
  const product = await db.product.findUnique({ where: { id: params.id } });

  return (
    <div>
      <ProductDetails product={product} /> {/* server — no JS bundle */}
      <AddToCartButton productId={product.id} /> {/* client — interactive */}
    </div>
  );
}
```

```jsx
// ProductDetails.tsx — Server Component
// Renders on server, ships HTML, no JS
function ProductDetails({ product }) {
  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <span>${product.price}</span>
    </div>
  );
}
```

```jsx
// AddToCartButton.tsx — Client Component
"use client"; // 👈 this directive marks it as client

import { useState } from "react";

function AddToCartButton({ productId }) {
  const [loading, setLoading] = useState(false);

  async function handleClick() {
    setLoading(true);
    await addToCart(productId);
    setLoading(false);
  }

  return (
    <button onClick={handleClick} disabled={loading}>
      {loading ? "Adding..." : "Add to Cart"}
    </button>
  );
}
```

**Decision framework:**
- Needs `useState`, `useEffect`, event handlers, browser APIs → **Client Component**
- Data fetching, static rendering, no interactivity, sensitive data (tokens, DB) → **Server Component**
- **Default to Server Components** and opt into Client only when needed

---

### Q18. useMemo and useCallback — when they help vs hurt

**Difficulty:** Mid

#### Answer

```jsx
// ---- When useMemo HELPS ----
// Expensive calculation on large dataset
function DataGrid({ rows, filters }) {
  // Without memo: recalculates on EVERY render, even unrelated state changes
  const filteredRows = useMemo(
    () => rows.filter(row => filters.every(f => f.test(row))),
    [rows, filters] // ✅ only recalculates when rows or filters change
  );

  return <Table data={filteredRows} />;
}

// ---- When useMemo HURTS (pessimization) ----
// ❌ Memoizing trivial computation — overhead > benefit
function UserGreeting({ user }) {
  // This memo costs more than the computation it "saves"
  const greeting = useMemo(() => `Hello, ${user.name}!`, [user.name]);
  return <h1>{greeting}</h1>;
}

// ✅ Just compute it — string concatenation is microseconds
function UserGreeting({ user }) {
  const greeting = `Hello, ${user.name}!`;
  return <h1>{greeting}</h1>;
}

// ---- useCallback HELPS: stable reference for child component ----
function Parent() {
  const [count, setCount] = useState(0);

  // Without useCallback: new function reference every render → child always re-renders
  const handleSubmit = useCallback((data) => {
    submitForm(data);
  }, []); // stable reference

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>+{count}</button>
      <MemoizedForm onSubmit={handleSubmit} /> {/* only re-renders if handleSubmit changes */}
    </>
  );
}

const MemoizedForm = React.memo(function Form({ onSubmit }) {
  return <form onSubmit={onSubmit}>...</form>;
});

// ---- useCallback HURTS: child isn't memoized anyway ----
function Parent() {
  // ❌ Pointless — Form re-renders regardless (not wrapped in React.memo)
  const handleSubmit = useCallback(() => submitForm(), []);
  return <Form onSubmit={handleSubmit} />;
}
```

**Rule of thumb:**
- Profile first (React DevTools Profiler). Don't prematurely memoize.
- `useMemo` → computations that take >1ms or complex object creation passed as props.
- `useCallback` → functions passed to `React.memo`-wrapped children or used as `useEffect` dependencies.

---

### Q19. Virtualized infinite scroll list in React

**Difficulty:** Hard

#### Answer

```jsx
import { useVirtualizer } from "@tanstack/react-virtual";
import { useRef, useEffect, useCallback } from "react";
import { useInfiniteQuery } from "@tanstack/react-query";

function VirtualizedList() {
  const parentRef = useRef(null);

  // Fetch paginated data
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage
  } = useInfiniteQuery({
    queryKey: ["items"],
    queryFn: ({ pageParam = 0 }) => fetchItems({ page: pageParam, limit: 50 }),
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });

  const allItems = data?.pages.flatMap(page => page.items) ?? [];

  // Virtualizer — only renders visible items + overscan buffer
  const rowVirtualizer = useVirtualizer({
    count: hasNextPage ? allItems.length + 1 : allItems.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 72,       // estimated row height
    overscan: 10,                 // render 10 extra rows above/below viewport
  });

  const virtualItems = rowVirtualizer.getVirtualItems();

  // Infinite scroll trigger — load more when last virtual item is visible
  useEffect(() => {
    const lastItem = virtualItems[virtualItems.length - 1];
    if (!lastItem) return;

    if (
      lastItem.index >= allItems.length - 1 &&
      hasNextPage &&
      !isFetchingNextPage
    ) {
      fetchNextPage();
    }
  }, [virtualItems, allItems.length, hasNextPage, isFetchingNextPage, fetchNextPage]);

  return (
    <div
      ref={parentRef}
      style={{ height: "600px", overflow: "auto" }}
    >
      {/* Total height div — creates scrollbar for full list */}
      <div style={{ height: rowVirtualizer.getTotalSize(), position: "relative" }}>
        {virtualItems.map((virtualRow) => {
          const item = allItems[virtualRow.index];
          const isLoaderRow = virtualRow.index > allItems.length - 1;

          return (
            <div
              key={virtualRow.index}
              style={{
                position: "absolute",
                top: 0,
                left: 0,
                width: "100%",
                height: `${virtualRow.size}px`,
                transform: `translateY(${virtualRow.start}px)`,
              }}
              ref={rowVirtualizer.measureElement} // dynamic height measurement
            >
              {isLoaderRow
                ? (hasNextPage ? <Skeleton /> : <EndOfList />)
                : <ListItem item={item} />
              }
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

**Key concepts:**
- Only DOM nodes for **visible rows** + overscan buffer exist — could have 100k items, only ~30 in DOM.
- `measureElement` allows **dynamic heights** (rows can be different sizes).
- `translateY` positions rows absolutely — no reflow as you scroll.
- `useInfiniteQuery` handles pagination, caching, background refetching.

---

### Q20. Controlled vs uncontrolled components — when to use uncontrolled in 2026

**Difficulty:** Easy

#### Answer

```jsx
// ---- Controlled: React owns the value ----
function ControlledInput() {
  const [value, setValue] = useState("");

  return (
    <input
      value={value}                          // React controls value
      onChange={e => setValue(e.target.value)} // must update state on every keystroke
    />
  );
}

// ---- Uncontrolled: DOM owns the value ----
function UncontrolledInput() {
  const ref = useRef(null);

  function handleSubmit() {
    console.log(ref.current.value); // read on demand
  }

  return (
    <>
      <input ref={ref} defaultValue="initial" /> {/* DOM manages value */}
      <button onClick={handleSubmit}>Submit</button>
    </>
  );
}
```

**When uncontrolled makes sense in 2026:**

1. **File inputs** — always uncontrolled (browser restriction)
```jsx
<input type="file" ref={fileRef} />
const file = fileRef.current.files[0];
```

2. **Large forms with 50+ fields** — controlled forms cause re-render per keystroke; uncontrolled + `react-hook-form` is dramatically faster:
```jsx
// react-hook-form uses uncontrolled inputs internally for performance
const { register, handleSubmit } = useForm();
<input {...register("email")} />
```

3. **Rich text editors** — ContentEditable elements are inherently uncontrolled.

4. **Third-party DOM libraries** (Quill, CodeMirror) — they manage their own DOM, so uncontrolled + ref is the only option.

---

## Angular

---

### Q21. Angular Signals — change detection model vs Zone.js

**Difficulty:** Hard

#### Answer

**Zone.js (old model):**
- Monkey-patches async APIs (setTimeout, Promises, XHR, etc.)
- Notifies Angular after ANY async operation completes
- Angular runs `checkAllComponentViews()` — walks entire component tree
- Problem: unnecessary checks on unrelated components

**Signals (2026 standard — zoneless Angular):**
- Push-based, fine-grained reactivity
- Only components that **read** a changed Signal re-render
- No Zone.js needed → smaller bundle, faster startup

```ts
// ---- Old Zone.js approach ----
@Component({
  template: `{{ count }}`,
  changeDetection: ChangeDetectionStrategy.Default // checks on every async event
})
export class CounterComponent {
  count = 0;
  increment() { this.count++; } // triggers change detection via Zone
}

// ---- Signals approach (Angular 17+) ----
import { signal, computed, effect } from "@angular/core";

@Component({
  template: `
    <p>Count: {{ count() }}</p>
    <p>Double: {{ double() }}</p>
    <button (click)="increment()">+</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush // or fully zoneless
})
export class CounterComponent {
  count = signal(0);                        // reactive primitive
  double = computed(() => this.count() * 2); // derived signal — auto-updates

  constructor() {
    effect(() => {
      console.log("Count changed:", this.count()); // reactive side effect
    });
  }

  increment() {
    this.count.update(c => c + 1); // or this.count.set(newValue)
  }
}
```

#### rxResource and resource (Angular 19)

```ts
import { resource, rxResource } from "@angular/core";
import { signal } from "@angular/core";

@Component({...})
export class ProductComponent {
  productId = signal(1);

  // resource — Promise-based async resource tied to signals
  productResource = resource({
    request: () => ({ id: this.productId() }), // re-fetches when productId changes
    loader: ({ request }) => fetch(`/api/products/${request.id}`).then(r => r.json())
  });

  // rxResource — RxJS Observable-based
  productResource2 = rxResource({
    request: () => this.productId(),
    loader: ({ request }) => this.productService.getById(request) // returns Observable
  });

  // Access in template:
  // productResource.value() — the data
  // productResource.isLoading() — loading state
  // productResource.error() — error state
}
```

#### Migration path

```ts
// Step 1: Add OnPush everywhere (reduces Zone-triggered checks)
// Step 2: Replace @Input() primitives with Signals
// Step 3: Replace RxJS in components with Signals / rxResource
// Step 4: Enable zoneless in app.config.ts
import { provideExperimentalZonelessChangeDetection } from "@angular/core";

export const appConfig: ApplicationConfig = {
  providers: [
    provideExperimentalZonelessChangeDetection(), // Angular 18+
  ]
};
// Step 5: Remove zone.js from polyfills.ts
```

---

### Q22. Angular DI hierarchy — `providedIn: 'root'` vs component-level `providers`

**Difficulty:** Mid

#### Answer

```
Angular Injector Tree:
┌─────────────────────────────┐
│  Root Injector              │  ← providedIn: 'root' → singleton for entire app
├─────────────────────────────┤
│  Platform Injector          │  ← Platform-level services
├─────────────────────────────┤
│  Module Injector            │  ← providers in @NgModule (legacy)
├─────────────────────────────┤
│  Component Injector (A)     │  ← providers: [MyService] in @Component
│  ├── Component Injector (B) │  ← child gets scoped instance
│  └── Component Injector (C) │  ← separate scoped instance
└─────────────────────────────┘
```

```ts
// ---- providedIn: 'root' — app-wide singleton ----
@Injectable({ providedIn: "root" })
export class AuthService {
  private user = signal<User | null>(null);
  // Same instance across entire app — state shared globally
}

// ---- Component-level providers — scoped instance ----
@Component({
  selector: "app-wizard",
  providers: [WizardStateService], // new instance per wizard component
  template: `<app-wizard-step-1 /><app-wizard-step-2 />`
})
export class WizardComponent {}

@Component({ selector: "app-wizard-step-1" })
export class WizardStep1 {
  // Gets the SAME WizardStateService as WizardComponent (walks up injector tree)
  constructor(private wizard: WizardStateService) {}
}
```

**Real-world issue: lazy loading**

```ts
// ❌ Problem: Service provided in root but used only in lazy module
// It's eagerly loaded in the main bundle

// ✅ Solution: providedIn lazy-loaded module (or 'any' for per-lazy-module instance)
@Injectable({ providedIn: "any" })
export class FeatureService {}
// Creates a new instance per lazy-loaded module that imports it
```

**Destroy lifecycle:**
- Root-provided services → destroyed when app destroyed
- Component-provided services → destroyed when component is destroyed (perfect for cleanup)

---

### Q23. Angular compiler pipeline — template to JavaScript

**Difficulty:** Hard

#### Answer

```
Source Code (TypeScript + HTML Templates)
         │
         ▼
  ┌─────────────┐
  │    ngtsc    │  Angular's TypeScript compiler wrapper
  └──────┬──────┘
         │
         ├─► Type-checking: template expressions validated against component types
         │   (catches {{ user.naem }} typos at compile time!)
         │
         ├─► Template compilation:
         │   HTML template → Angular template AST
         │   → Compiled template function (JavaScript)
         │
         └─► Output: compiled JS + type definitions
```

#### What a compiled component looks like

```ts
// Source
@Component({
  template: `<h1>{{ title }}</h1><button (click)="greet()">Hi</button>`
})
class AppComponent {
  title = "Hello";
  greet() { alert("Hi!"); }
}

// Compiled output (simplified Ivy output)
AppComponent.ɵcmp = defineComponent({
  type: AppComponent,
  selectors: [["app-root"]],
  template: function AppComponent_Template(rf, ctx) {
    if (rf & 1) { // CREATE phase
      ɵɵelementStart(0, "h1");
      ɵɵtext(1);
      ɵɵelementEnd();
      ɵɵelementStart(2, "button");
      ɵɵlistener("click", function() { return ctx.greet(); });
      ɵɵtext(3, "Hi");
      ɵɵelementEnd();
    }
    if (rf & 2) { // UPDATE phase
      ɵɵadvance(1);
      ɵɵtextInterpolate(ctx.title); // only runs when title changes
    }
  }
});
```

**Key Ivy improvements over ViewEngine:**
- Tree-shakeable: unused Angular features not bundled
- Locality principle: each component compiled independently (faster incremental builds)
- Better error messages: template type errors point to exact template location
- Runtime smaller: no NgFactory files, component self-describes

---

### Q24. `rxResource` vs `resource` in Angular 19+

**Difficulty:** Mid

*(See Q21 for code examples — additional depth here)*

```ts
// resource — for Promise-based async
// Best for: fetch API, one-shot async operations
const userResource = resource({
  request: () => ({ id: this.userId() }),
  loader: async ({ request, abortSignal }) => {
    const res = await fetch(`/api/users/${request.id}`, { signal: abortSignal });
    if (!res.ok) throw new Error("Fetch failed");
    return res.json();
  }
});

// rxResource — for RxJS Observable-based streams
// Best for: services already returning Observables, complex operators needed
const userResource2 = rxResource({
  request: () => this.userId(),
  loader: ({ request }) => this.userService.getUser(request).pipe(
    retry(3),
    catchError(err => of(null))
  )
});

// Both expose identical Signal-based API:
// .value()      — current data (undefined if loading)
// .isLoading()  — boolean Signal
// .error()      — error Signal
// .status()     — 'idle' | 'loading' | 'resolved' | 'error'
// .reload()     — manually trigger refetch
// .update()     — optimistic update
```

**When to use which:**
- Fresh projects using `fetch` → `resource`
- Existing services returning `Observable<T>` → `rxResource`
- Need RxJS operators (retry, debounce, switchMap) → `rxResource`

---

## Performance

---

### Q25. Core Web Vitals — LCP, INP, CLS, TTFB

**Difficulty:** Mid

#### Answer

| Metric | Measures | Good | Poor |
|--------|----------|------|------|
| **LCP** (Largest Contentful Paint) | When main content is visible | < 2.5s | > 4s |
| **INP** (Interaction to Next Paint) | Responsiveness to user input | < 200ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | Visual stability | < 0.1 | > 0.25 |
| **TTFB** (Time to First Byte) | Server response speed | < 800ms | > 1.8s |

**INP replaced FID (First Input Delay) in March 2024.** INP measures the worst interaction (click, tap, keyboard) across the entire page session.

#### Diagnosing and fixing each

**LCP — fix strategy:**
```html
<!-- ❌ LCP image lazy-loaded (wrong!) -->
<img src="hero.jpg" loading="lazy" />

<!-- ✅ LCP image preloaded and eager -->
<link rel="preload" as="image" href="hero.jpg" />
<img src="hero.jpg" loading="eager" fetchpriority="high" />
```

**INP — fix strategy:**
```js
// ❌ Long task blocks interaction response
button.addEventListener("click", () => {
  const result = heavyComputation(data); // blocks for 500ms
  updateUI(result);
});

// ✅ Break into smaller tasks using scheduler
button.addEventListener("click", async () => {
  updateUI({ loading: true });
  await scheduler.yield(); // yield to browser — allows paint
  const result = heavyComputation(data);
  updateUI({ loading: false, data: result });
});
```

**CLS — fix strategy:**
```css
/* ❌ Image without dimensions — shifts layout when loaded */
img { max-width: 100%; }

/* ✅ Reserve space before image loads */
img {
  width: 800px;
  height: 450px;
  max-width: 100%;
  aspect-ratio: 16/9; /* modern approach */
}
```

**Priority order for a content-heavy React app:**
1. **TTFB** — if >1s, no amount of frontend optimization helps.
2. **LCP** — preload the hero, eliminate render-blocking resources.
3. **CLS** — set dimensions on images/ads, avoid injecting content above fold.
4. **INP** — profile long tasks, use `scheduler.yield()`, move work to Web Workers.

---

### Q26. React dashboard with 200+ re-renders per minute — debugging approach

**Difficulty:** Hard

#### Answer

**Step 1: Profile to find the cause**
```js
// Enable React DevTools Profiler recording
// Look for: components with high "Render duration" and "Render count"
// Identify: what caused each render (prop changes, state changes, parent renders)
```

**Step 2: Identify unnecessary re-renders**
```jsx
// Add why-did-you-render (dev tool)
import whyDidYouRender from "@welldone-software/why-did-you-render";
whyDidYouRender(React, {
  trackAllPureComponents: true,
  logOnDifferentValues: true
});
```

**Step 3: Common fixes**

```jsx
// ---- Fix 1: Memoize expensive components ----
const PriceWidget = React.memo(({ price, ticker }) => {
  return <div>{ticker}: {price}</div>;
}, (prev, next) => prev.price === next.price && prev.ticker === next.ticker);

// ---- Fix 2: Normalize state — avoid object recreation ----
// ❌ Bad — new object every time, all subscribers re-render
const [data, setData] = useState({ prices: {}, volumes: {} });
setData({ ...data, prices: { ...data.prices, BTC: 50000 } }); // full state copy

// ✅ Better — split into granular atoms (Zustand/Jotai)
const usePriceStore = create((set) => ({
  prices: {},
  updatePrice: (ticker, price) => set(state => ({
    prices: { ...state.prices, [ticker]: price }
  }))
}));

// Only components subscribing to BTC price re-render when BTC changes
const BtcPrice = () => {
  const price = usePriceStore(state => state.prices.BTC); // selector
  return <div>BTC: {price}</div>;
};

// ---- Fix 3: Throttle high-frequency updates ----
// ❌ WebSocket pushing 60fps price updates directly to state
ws.onmessage = (e) => setData(JSON.parse(e.data));

// ✅ Batch updates with throttle
const throttledUpdate = useCallback(
  throttle((newData) => {
    startTransition(() => setData(newData)); // non-urgent
  }, 100), // max 10 updates/second
  []
);

// ---- Fix 4: Move computation to Web Worker ----
const worker = new Worker(new URL("./dataProcessor.worker.js", import.meta.url));

ws.onmessage = (e) => {
  worker.postMessage(e.data); // offload processing
};

worker.onmessage = (e) => {
  startTransition(() => setProcessedData(e.data));
};

// ---- Fix 5: Virtualize large lists ----
// 1000 rows visible → only render 20 in DOM (react-virtual)
```

---

### Q27. Code splitting — route-level and component-level

**Difficulty:** Mid

#### Answer

```jsx
// ---- React: Route-level splitting ----
import { lazy, Suspense } from "react";
import { createBrowserRouter } from "react-router-dom";

// Each lazy() creates a separate chunk — only loaded when route is visited
const Dashboard = lazy(() => import("./pages/Dashboard"));
const Analytics = lazy(() => import("./pages/Analytics"));
const Settings = lazy(() => import("./pages/Settings"));

const router = createBrowserRouter([
  {
    path: "/dashboard",
    element: (
      <Suspense fallback={<PageSkeleton />}>
        <Dashboard />
      </Suspense>
    )
  },
  { path: "/analytics", element: <Suspense fallback={<PageSkeleton />}><Analytics /></Suspense> },
]);

// ---- Component-level splitting — heavy components ----
const RichTextEditor = lazy(() => import("./components/RichTextEditor")); // heavy dep
const DataChart = lazy(() => import("./components/DataChart"));

function PostEditor({ post }) {
  const [editMode, setEditMode] = useState(false);

  return (
    <div>
      <button onClick={() => setEditMode(true)}>Edit</button>
      {editMode && (
        <Suspense fallback={<EditorSkeleton />}>
          <RichTextEditor content={post.body} /> {/* loaded only when user clicks Edit */}
        </Suspense>
      )}
    </div>
  );
}

// ---- Angular: Lazy routes ----
// app.routes.ts
export const routes: Routes = [
  {
    path: "dashboard",
    loadComponent: () => import("./dashboard/dashboard.component")
      .then(m => m.DashboardComponent)
  },
  {
    path: "admin",
    loadChildren: () => import("./admin/admin.routes")
      .then(m => m.ADMIN_ROUTES)
  }
];
```

**Trade-offs:**

| | Benefit | Risk |
|--|---------|------|
| Route splitting | Faster initial load | Loading spinner on navigation |
| Component splitting | Defer heavy deps | Loading state inside page |
| Too many chunks | Parallel downloads | HTTP/2 multiplexing overhead |

**Preloading strategy** — load chunks before user needs them:
```js
// React Router — preload on hover
<Link
  to="/analytics"
  onMouseEnter={() => import("./pages/Analytics")} // preload on hover
>Analytics</Link>

// Angular — preload all lazy routes after initial load
provideRouter(routes, withPreloading(PreloadAllModules))
// Or custom strategy: preload based on network speed
```

---

### Q28. Browser rendering pipeline — where JS and CSS cause bottlenecks

**Difficulty:** Hard

#### Answer

```
Parse HTML → Build DOM
Parse CSS  → Build CSSOM
              │
              ▼
         Render Tree (DOM + CSSOM merged — only visible nodes)
              │
              ▼
    Layout (Reflow) — calculate size and position of every element
              │
              ▼
    Paint — draw pixels for each layer (text, borders, colors, shadows)
              │
              ▼
    Composite — GPU combines layers, sends to screen
```

#### What triggers expensive recalculations

**Layout (Reflow) triggers — most expensive:**
```js
// ❌ Reading these properties forces layout sync (layout thrashing)
const height = element.offsetHeight;   // forces layout
const width = element.getBoundingClientRect().width;
const scroll = element.scrollTop;

// ❌ Layout thrashing — read/write alternating in a loop
for (const el of elements) {
  const h = el.offsetHeight;          // READ — forces layout
  el.style.height = h * 2 + "px";    // WRITE — invalidates layout
  // browser must recalculate layout EVERY iteration
}

// ✅ Batch reads, then batch writes (FastDOM pattern)
const heights = elements.map(el => el.offsetHeight); // all reads first
elements.forEach((el, i) => { el.style.height = heights[i] * 2 + "px"; }); // then writes

// ✅ Modern: use ResizeObserver instead of reading layout in loops
```

**Paint triggers:**
```css
/* Expensive — triggers paint */
.box:hover { box-shadow: 0 4px 20px rgba(0,0,0,0.3); }
.box:hover { background-color: red; }
.box:hover { border-radius: 50%; }

/* Compositor-only — no paint, runs on GPU thread ✅ */
.box:hover { transform: scale(1.1); }  /* compositor only */
.box:hover { opacity: 0.5; }           /* compositor only */
```

**CSS properties that create new compositor layers:**
```css
.animated {
  will-change: transform; /* promotes to own layer — expensive to promote, cheap to animate */
  transform: translateZ(0); /* old hack for layer promotion */
}
/* Use sparingly — too many layers = memory pressure */
```

**Key insight:** Animations using only `transform` and `opacity` run entirely on the GPU compositor thread, never touching the main thread — they can't be blocked by JavaScript execution.

---

### Q29. Web Workers — constraints and use cases

**Difficulty:** Mid

#### Answer

```js
// ---- Creating a worker ----
// main.js
const worker = new Worker(new URL("./my.worker.js", import.meta.url), {
  type: "module" // supports ES modules
});

// Send data to worker (structured clone algorithm)
worker.postMessage({ type: "PROCESS", payload: largeArray });

// Receive results
worker.onmessage = (e) => {
  console.log("Worker result:", e.data);
};

worker.onerror = (e) => {
  console.error("Worker error:", e.message);
};
```

```js
// my.worker.js
self.onmessage = (e) => {
  const { type, payload } = e.data;

  if (type === "PROCESS") {
    // Heavy computation — runs off main thread
    const result = payload.map(item => expensiveTransform(item));
    self.postMessage({ type: "RESULT", data: result });
  }
};
```

**What you CAN do in a Worker:**
- Heavy computation (ML inference, crypto, data transforms)
- `fetch()` API calls
- WebSockets, IndexedDB
- Canvas (OffscreenCanvas)
- Wasm execution

**What you CANNOT do:**
- Access DOM (`document`, `window`, `localStorage`)
- Access `document.cookie`
- Synchronous XHR

**Structured Clone constraints** (what can be posted):
```js
// ✅ Can transfer
worker.postMessage({ number: 42, string: "hi", array: [1,2,3], date: new Date() });

// ❌ Cannot transfer (throws DataCloneError)
worker.postMessage({ fn: () => {}, symbol: Symbol("x") });

// ✅ Transferable objects — zero-copy transfer (faster!)
const buffer = new ArrayBuffer(1024 * 1024); // 1MB
worker.postMessage(buffer, [buffer]); // buffer is MOVED, not copied — main loses access
```

**Real example: CSV parsing off main thread**
```js
// Parse a 50MB CSV without freezing the UI
const csvWorker = new Worker(new URL("./csv.worker.js", import.meta.url));

function parseCSV(file) {
  return new Promise((resolve) => {
    file.arrayBuffer().then(buffer => {
      csvWorker.postMessage(buffer, [buffer]); // transfer, not copy
      csvWorker.onmessage = (e) => resolve(e.data);
    });
  });
}
```

---

## System Design

---

### Q30. Design a scalable design system

**Difficulty:** Hard

#### Answer

```
Architecture:
┌─────────────────────────────────────────────────┐
│           Design Tokens (source of truth)       │
│   JSON/Style Dictionary → CSS vars, JS, iOS,    │
│   Android, Figma tokens                         │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│           Base/Primitive Components             │
│   Button, Input, Icon, Text, Stack, Grid        │
│   Zero styling opinions — only accessibility    │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│           Composite Components                  │
│   Modal, Dropdown, DataTable, DatePicker        │
│   Built from primitives + design tokens         │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│           Application Patterns                  │
│   PageLayout, AuthForm, NavBar                  │
│   Business-specific compositions                │
└─────────────────────────────────────────────────┘
```

**Token system:**
```json
// tokens.json (Style Dictionary source)
{
  "color": {
    "brand": { "primary": { "value": "#7F77DD", "type": "color" } },
    "semantic": {
      "danger": { "value": "{color.red.600}", "type": "color" }
    }
  },
  "spacing": {
    "sm": { "value": "8px" },
    "md": { "value": "16px" },
    "lg": { "value": "24px" }
  }
}
```

```css
/* Generated CSS (web) */
:root {
  --color-brand-primary: #7F77DD;
  --color-semantic-danger: #A32D2D;
  --spacing-sm: 8px;
}
```

**Cross-framework support — Web Components as base:**
```js
// Define once, use everywhere
class DSButton extends HTMLElement {
  static get observedAttributes() { return ["variant", "size", "disabled"]; }

  connectedCallback() {
    this.render();
  }

  render() {
    this.innerHTML = `
      <button class="ds-btn ds-btn--${this.getAttribute("variant") || "primary"}">
        <slot></slot>
      </button>`;
  }
}

customElements.define("ds-button", DSButton);
```

```jsx
// React wrapper (thin layer)
export const Button = ({ variant, children, ...props }) => (
  <ds-button variant={variant} {...props}>{children}</ds-button>
);

// Angular wrapper
@Component({ template: `<ds-button [attr.variant]="variant"><ng-content /></ds-button>` })
export class ButtonComponent { @Input() variant = "primary"; }
```

**Versioning strategy:**
- Semantic versioning (MAJOR.MINOR.PATCH)
- Breaking changes only in major versions
- Changelog automation via conventional commits
- Changelogs per package in monorepo (Lerna/Turborepo)
- Visual regression testing with Chromatic + Storybook

---

### Q31. Design a real-time collaborative document editor frontend

**Difficulty:** Hard

#### Answer

```
Architecture overview:

Client A                    Server                    Client B
   │                           │                          │
   │──── WebSocket connect ────▶│◀──── WebSocket connect──│
   │                           │                          │
   │  User types "Hello"        │                          │
   │──── Op: {insert,0,"H"} ──▶│──── broadcast ──────────▶│
   │                           │                          │ applies op
   │  Sync confirmed            │                          │
   │◀─── acknowledge ──────────│                          │
```

**Conflict resolution — Operational Transformation (simplified):**
```ts
type Operation =
  | { type: "insert"; position: number; text: string }
  | { type: "delete"; position: number; length: number };

// Transform op A against concurrent op B
function transform(opA: Operation, opB: Operation): Operation {
  if (opA.type === "insert" && opB.type === "insert") {
    if (opB.position <= opA.position) {
      return { ...opA, position: opA.position + opB.text.length };
    }
    return opA;
  }
  // ... other cases
  return opA;
}
```

**State management:**
```ts
// Zustand store for document state
const useDocStore = create<DocState>((set, get) => ({
  content: "",
  version: 0,
  pendingOps: [],

  applyLocal: (op: Operation) => {
    set(state => ({
      content: applyOp(state.content, op),
      pendingOps: [...state.pendingOps, op]
    }));
    ws.send(JSON.stringify({ op, version: get().version }));
  },

  applyRemote: (op: Operation, remoteVersion: number) => {
    set(state => {
      // Transform remote op against all pending local ops
      const transformedOp = state.pendingOps.reduce(
        (acc, pendingOp) => transform(acc, pendingOp),
        op
      );
      return {
        content: applyOp(state.content, transformedOp),
        version: remoteVersion
      };
    });
  }
}));
```

**Offline support:**
```ts
// IndexedDB for offline persistence
const db = await openDB("collab-docs", 1, {
  upgrade(db) {
    db.createObjectStore("documents", { keyPath: "id" });
    db.createObjectStore("pending-ops", { autoIncrement: true });
  }
});

// Service Worker caches the app shell
// When offline: ops queued in IndexedDB
// On reconnect: replay queued ops against server state
```

---

### Q32. Design a search autocomplete with full production quality

**Difficulty:** Mid

#### Answer

```tsx
// Full implementation with debounce, cache, abort, accessibility

function SearchAutocomplete({ onSelect }) {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const [isOpen, setIsOpen] = useState(false);
  const [activeIndex, setActiveIndex] = useState(-1);
  const [isLoading, setIsLoading] = useState(false);

  const cache = useRef(new Map<string, SearchResult[]>());
  const abortRef = useRef<AbortController | null>(null);
  const inputRef = useRef<HTMLInputElement>(null);
  const listRef = useRef<HTMLUListElement>(null);

  const search = useMemo(() =>
    debounce(async (q: string) => {
      if (!q.trim()) { setResults([]); setIsOpen(false); return; }

      // Cache hit
      if (cache.current.has(q)) {
        setResults(cache.current.get(q)!);
        setIsOpen(true);
        return;
      }

      // Abort previous request
      abortRef.current?.abort();
      abortRef.current = new AbortController();

      setIsLoading(true);
      try {
        const res = await fetch(`/api/search?q=${encodeURIComponent(q)}`, {
          signal: abortRef.current.signal
        });
        const data = await res.json();
        cache.current.set(q, data.results);
        setResults(data.results);
        setIsOpen(true);
        setActiveIndex(-1);
      } catch (e) {
        if ((e as Error).name !== "AbortError") setResults([]);
      } finally {
        setIsLoading(false);
      }
    }, 300),
    []
  );

  // Keyboard navigation — ARIA combobox pattern
  function handleKeyDown(e: React.KeyboardEvent) {
    switch (e.key) {
      case "ArrowDown":
        e.preventDefault();
        setActiveIndex(i => Math.min(i + 1, results.length - 1));
        break;
      case "ArrowUp":
        e.preventDefault();
        setActiveIndex(i => Math.max(i - 1, -1));
        break;
      case "Enter":
        if (activeIndex >= 0) {
          onSelect(results[activeIndex]);
          setIsOpen(false);
        }
        break;
      case "Escape":
        setIsOpen(false);
        setActiveIndex(-1);
        inputRef.current?.focus();
        break;
      case "Tab":
        setIsOpen(false);
        break;
    }
  }

  const listboxId = useId();
  const activeId = activeIndex >= 0 ? `option-${activeIndex}` : undefined;

  return (
    <div role="combobox" aria-expanded={isOpen} aria-haspopup="listbox">
      <input
        ref={inputRef}
        type="text"
        value={query}
        aria-label="Search"
        aria-autocomplete="list"
        aria-controls={listboxId}
        aria-activedescendant={activeId}
        onChange={e => { setQuery(e.target.value); search(e.target.value); }}
        onKeyDown={handleKeyDown}
        onBlur={() => setTimeout(() => setIsOpen(false), 150)}
      />
      {isLoading && <Spinner aria-label="Loading results" />}
      {isOpen && results.length > 0 && (
        <ul
          ref={listRef}
          id={listboxId}
          role="listbox"
          aria-label="Search results"
        >
          {results.map((result, i) => (
            <li
              key={result.id}
              id={`option-${i}`}
              role="option"
              aria-selected={i === activeIndex}
              onClick={() => { onSelect(result); setIsOpen(false); }}
            >
              <HighlightMatch text={result.label} query={query} />
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

### Q33. Micro-frontend architecture for 3 teams

**Difficulty:** Hard

#### Answer

```
┌─────────────────────────────────────────────┐
│              Shell Application              │
│  - Routing (which MFE to load)             │
│  - Shared auth state                       │
│  - Global navigation, header, footer       │
│  - Shared dependencies (React, design sys) │
└──────┬──────────────┬──────────────┬───────┘
       │              │              │
  ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
  │  Team A │   │  Team B │   │  Team C │
  │ MFE:    │   │ MFE:    │   │ MFE:    │
  │/dashboard│  │/orders  │   │/profile │
  │ (React) │   │(Angular)│   │ (React) │
  └─────────┘   └─────────┘   └─────────┘
```

**Webpack Module Federation (webpack.config.js):**
```js
// Shell's webpack config
new ModuleFederationPlugin({
  name: "shell",
  remotes: {
    dashboard: "dashboard@http://team-a.internal/remoteEntry.js",
    orders: "orders@http://team-b.internal/remoteEntry.js",
  },
  shared: {
    react: { singleton: true, requiredVersion: "^19.0.0" },
    "react-dom": { singleton: true, requiredVersion: "^19.0.0" },
    "@design-system/core": { singleton: true }
  }
});

// Team A's webpack config (exposes their app)
new ModuleFederationPlugin({
  name: "dashboard",
  filename: "remoteEntry.js",
  exposes: {
    "./App": "./src/App",
    "./DashboardWidget": "./src/components/DashboardWidget"
  },
  shared: { react: { singleton: true } }
});
```

**Shell routing:**
```jsx
const Dashboard = lazy(() => import("dashboard/App")); // loaded from Team A's server
const Orders = lazy(() => import("orders/App"));

function Shell() {
  return (
    <Router>
      <SharedNav />
      <Routes>
        <Route path="/dashboard/*" element={<Suspense fallback={<Loader />}><Dashboard /></Suspense>} />
        <Route path="/orders/*" element={<Suspense fallback={<Loader />}><Orders /></Suspense>} />
      </Routes>
    </Router>
  );
}
```

**Cross-MFE communication:**
```ts
// Custom events — decoupled, no shared state library needed
// Team B fires event when order is placed
window.dispatchEvent(new CustomEvent("mfe:order-placed", {
  detail: { orderId: "123", total: 99.99 }
}));

// Shell or Team C listens
window.addEventListener("mfe:order-placed", (e: CustomEvent) => {
  updateNotificationCount();
});

// Shared auth — via cookie or broadcast channel
const authChannel = new BroadcastChannel("auth");
authChannel.postMessage({ type: "LOGOUT" }); // all MFEs receive this
```

---

### Q34. Frontend observability strategy

**Difficulty:** Mid

#### Answer

**Four pillars:**

```ts
// 1. Error tracking (Sentry)
Sentry.init({
  dsn: "...",
  tracesSampleRate: 0.1,      // 10% of transactions for performance
  replaysSessionSampleRate: 0.05, // 5% session replay
  integrations: [
    Sentry.replayIntegration({ maskAllText: true }), // GDPR-safe replay
  ]
});

// Custom error boundary
class ErrorBoundary extends React.Component {
  componentDidCatch(error, info) {
    Sentry.captureException(error, { extra: { componentStack: info.componentStack } });
  }
}

// 2. Performance monitoring (Web Vitals → RUM)
import { onLCP, onINP, onCLS, onTTFB } from "web-vitals";

function sendToAnalytics({ name, value, rating, id }) {
  // Sample: only send 20% of metrics to avoid hitting limits
  if (Math.random() > 0.2) return;

  fetch("/analytics/vitals", {
    method: "POST",
    body: JSON.stringify({ metric: name, value, rating, page: location.pathname }),
    keepalive: true // fires even when page is unloading
  });
}

onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
onTTFB(sendToAnalytics);

// 3. User interaction tracking (privacy-first)
// Track actions, not content
analytics.track("button_clicked", {
  component: "AddToCartButton",
  productId: product.id,
  // ❌ Never: userName, email, raw search queries
});

// 4. Logging with OpenTelemetry
import { trace, context } from "@opentelemetry/api";

const tracer = trace.getTracer("frontend");
const span = tracer.startSpan("checkout_flow");
// ... user completes checkout ...
span.setStatus({ code: SpanStatusCode.OK });
span.end();
```

---

## AI & Dev Tools

---

### Q35. How are you using AI tools in your frontend workflow?

**Difficulty:** Mid

#### Answer (model answer — what a senior engineer should say)

**Where AI genuinely helps:**
- Boilerplate generation: typing out repetitive component shells, form validation schemas, test setup
- Code review assistance: "explain why this might cause memory leaks"
- Debugging: pasting error + stack trace for quick diagnosis hypothesis
- Documentation: generating JSDoc from function signatures
- Regex / complex string manipulation: "write a regex for email that handles subdomains"
- Test case generation: "what edge cases am I missing for this input validator?"
- Migration assistance: "convert this class component to hooks"

**Where AI is NOT trusted:**
- Architecture decisions — AI suggests patterns without knowing your constraints
- Security-sensitive code — always manually review auth, input sanitization
- Complex state logic — often generates subtly wrong reducer logic
- Version-specific APIs — hallucinations in newer framework APIs (Angular Signals, React 19 specifics)
- Final production code — always review, understand, and test output

**Concrete workflow:**
```
Write component signature + JSDoc comment
→ Copilot generates initial implementation
→ Review every line (don't blindly accept)
→ Run tests — fix AI mistakes (there always are some)
→ Refactor to match codebase conventions
```

---

### Q36. Build a frontend for streaming LLM response

**Difficulty:** Hard

#### Answer

```tsx
function LLMChat() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [streamingContent, setStreamingContent] = useState("");
  const [isStreaming, setIsStreaming] = useState(false);
  const abortRef = useRef<AbortController | null>(null);

  async function sendMessage(userMessage: string) {
    setMessages(prev => [...prev, { role: "user", content: userMessage }]);
    setIsStreaming(true);
    setStreamingContent("");

    abortRef.current = new AbortController();

    try {
      const response = await fetch("/api/chat", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ messages: [...messages, { role: "user", content: userMessage }] }),
        signal: abortRef.current.signal
      });

      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      if (!response.body) throw new Error("No response body");

      // ReadableStream processing
      const reader = response.body.getReader();
      const decoder = new TextDecoder();
      let fullContent = "";

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const chunk = decoder.decode(value, { stream: true });

        // Parse SSE format: "data: {...}\n\n"
        const lines = chunk.split("\n").filter(l => l.startsWith("data: "));
        for (const line of lines) {
          const data = line.slice(6); // remove "data: "
          if (data === "[DONE]") break;

          try {
            const parsed = JSON.parse(data);
            const delta = parsed.choices?.[0]?.delta?.content ?? "";
            fullContent += delta;
            setStreamingContent(fullContent); // update on each chunk
          } catch { continue; }
        }
      }

      // Finalize: add complete message
      setMessages(prev => [...prev, { role: "assistant", content: fullContent }]);
    } catch (e) {
      if ((e as Error).name !== "AbortError") {
        setMessages(prev => [...prev, { role: "error", content: "Something went wrong." }]);
      }
    } finally {
      setIsStreaming(false);
      setStreamingContent("");
    }
  }

  return (
    <div>
      <MessageList messages={messages} />

      {/* Streaming in progress — progressive markdown render */}
      {isStreaming && (
        <div aria-live="polite" aria-label="Assistant response">
          <MarkdownRenderer content={streamingContent} />
          <BlinkingCursor />
          <button onClick={() => abortRef.current?.abort()}>Stop</button>
        </div>
      )}

      <MessageInput onSend={sendMessage} disabled={isStreaming} />
    </div>
  );
}

// Markdown rendered progressively — use a streaming-aware parser
function MarkdownRenderer({ content }: { content: string }) {
  // marked.js or micromark — parse incrementally
  const html = useMemo(() => marked.parse(content), [content]);
  return <div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(html) }} />;
}
```

---

## Web APIs & Accessibility

---

### Q37. Intersection Observer — lazy loading and analytics

**Difficulty:** Mid

#### Answer

```js
// ---- Lazy loading images ----
const lazyImages = document.querySelectorAll("img[data-src]");

const imageObserver = new IntersectionObserver((entries, observer) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src;           // swap placeholder for real src
      img.removeAttribute("data-src");
      observer.unobserve(img);             // stop observing once loaded
    }
  });
}, {
  root: null,        // viewport
  rootMargin: "200px", // start loading 200px before entering viewport
  threshold: 0       // trigger as soon as any pixel is visible
});

lazyImages.forEach(img => imageObserver.observe(img));

// ---- Analytics: track which sections users actually see ----
const sectionObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting && entry.intersectionRatio >= 0.5) {
      analytics.track("section_viewed", {
        sectionId: entry.target.dataset.section,
        timeVisible: Date.now()
      });
    }
  });
}, { threshold: 0.5 }); // only fire when 50% visible

document.querySelectorAll("[data-section]").forEach(s => sectionObserver.observe(s));

// ---- React hook ----
function useIntersectionObserver(options = {}) {
  const [isVisible, setIsVisible] = useState(false);
  const ref = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      setIsVisible(entry.isIntersecting);
    }, options);

    if (ref.current) observer.observe(ref.current);
    return () => observer.disconnect(); // cleanup!
  }, []);

  return [ref, isVisible];
}

// Usage
function LazySection() {
  const [ref, isVisible] = useIntersectionObserver({ threshold: 0.1 });
  return (
    <section ref={ref}>
      {isVisible ? <HeavyComponent /> : <Placeholder />}
    </section>
  );
}
```

**Why better than scroll events:**
- Runs off main thread (browser-native)
- No throttling needed
- No layout thrashing from reading `scrollTop`/`getBoundingClientRect()` repeatedly
- Automatically handles nested scroll containers, iframes

---

### Q38. Storage architecture — localStorage, IndexedDB, Cache API

**Difficulty:** Hard

#### Answer

| Storage | Sync? | Size | Expiry | Access | Best for |
|---------|-------|------|--------|--------|----------|
| `localStorage` | Sync | ~5MB | Never | Main thread | Small UI prefs, non-sensitive settings |
| `sessionStorage` | Sync | ~5MB | Tab close | Main thread | Temp tab state |
| `IndexedDB` | Async | Hundreds MB | Never | Main + Worker | Large datasets, offline data |
| `Cache API` | Async | Hundreds MB | Manual | Main + SW | Network responses, app shell |
| Cookies | Sync | ~4KB | Configurable | Server + client | Auth tokens (HttpOnly) |

```ts
// ---- Design: Tiered caching strategy ----

// Layer 1: Memory cache (fastest, lost on page refresh)
const memoryCache = new Map<string, { data: unknown; expires: number }>();

// Layer 2: localStorage (fast, survives refresh, ~5MB limit)
const localCache = {
  get: (key: string) => {
    const item = localStorage.getItem(key);
    if (!item) return null;
    const { data, expires } = JSON.parse(item);
    if (expires < Date.now()) { localStorage.removeItem(key); return null; }
    return data;
  },
  set: (key: string, data: unknown, ttlMs: number) => {
    localStorage.setItem(key, JSON.stringify({ data, expires: Date.now() + ttlMs }));
  }
};

// Layer 3: IndexedDB (large data, complex queries)
import { openDB } from "idb";

const db = await openDB("app-cache", 1, {
  upgrade(db) {
    db.createObjectStore("responses", { keyPath: "url" });
    db.createObjectStore("user-data", { keyPath: "id" });
  }
});

async function getCachedResponse(url: string) {
  // Check memory first
  const mem = memoryCache.get(url);
  if (mem && mem.expires > Date.now()) return mem.data;

  // Check localStorage
  const local = localCache.get(url);
  if (local) return local;

  // Check IndexedDB
  const idbData = await db.get("responses", url);
  if (idbData) return idbData.body;

  return null;
}

// Layer 4: Cache API (via Service Worker — for offline)
// service-worker.js
self.addEventListener("fetch", (event) => {
  event.respondWith(
    caches.match(event.request).then(cached => {
      if (cached) return cached;

      return fetch(event.request).then(response => {
        // Cache API responses (JS, CSS, images)
        if (event.request.url.match(/\.(js|css|png|woff2)$/)) {
          const cache = caches.open("static-v2");
          cache.then(c => c.put(event.request, response.clone()));
        }
        return response;
      });
    })
  );
});
```

---

### Q39. Keyboard accessibility for a custom dropdown

**Difficulty:** Mid

#### Answer

```tsx
// ARIA combobox pattern — W3C specified
function AccessibleDropdown({ options, label, onChange }) {
  const [isOpen, setIsOpen] = useState(false);
  const [activeIndex, setActiveIndex] = useState(0);
  const [selected, setSelected] = useState(options[0]);

  const buttonRef = useRef<HTMLButtonElement>(null);
  const listRef = useRef<HTMLUListElement>(null);
  const id = useId();
  const listboxId = `${id}-listbox`;

  function selectOption(option, index) {
    setSelected(option);
    setActiveIndex(index);
    setIsOpen(false);
    onChange(option);
    buttonRef.current?.focus(); // return focus to trigger
  }

  function handleButtonKeyDown(e: React.KeyboardEvent) {
    switch (e.key) {
      case "Enter":
      case " ":
      case "ArrowDown":
        e.preventDefault();
        setIsOpen(true);
        // Focus first/active item after open
        setTimeout(() => {
          listRef.current?.children[activeIndex]?.focus();
        }, 0);
        break;
      case "ArrowUp":
        e.preventDefault();
        setIsOpen(true);
        break;
    }
  }

  function handleOptionKeyDown(e: React.KeyboardEvent, index: number) {
    switch (e.key) {
      case "ArrowDown":
        e.preventDefault();
        const next = Math.min(index + 1, options.length - 1);
        (listRef.current?.children[next] as HTMLElement)?.focus();
        setActiveIndex(next);
        break;
      case "ArrowUp":
        e.preventDefault();
        if (index === 0) { setIsOpen(false); buttonRef.current?.focus(); return; }
        const prev = index - 1;
        (listRef.current?.children[prev] as HTMLElement)?.focus();
        setActiveIndex(prev);
        break;
      case "Enter":
      case " ":
        e.preventDefault();
        selectOption(options[index], index);
        break;
      case "Escape":
        setIsOpen(false);
        buttonRef.current?.focus();
        break;
      case "Tab":
        setIsOpen(false);
        break;
      case "Home":
        e.preventDefault();
        (listRef.current?.children[0] as HTMLElement)?.focus();
        break;
      case "End":
        e.preventDefault();
        const last = options.length - 1;
        (listRef.current?.children[last] as HTMLElement)?.focus();
        break;
    }
  }

  // Close on outside click
  useEffect(() => {
    if (!isOpen) return;
    const handler = (e: MouseEvent) => {
      if (!listRef.current?.contains(e.target as Node) &&
          !buttonRef.current?.contains(e.target as Node)) {
        setIsOpen(false);
      }
    };
    document.addEventListener("mousedown", handler);
    return () => document.removeEventListener("mousedown", handler);
  }, [isOpen]);

  return (
    <div>
      <label id={`${id}-label`}>{label}</label>

      <button
        ref={buttonRef}
        aria-haspopup="listbox"
        aria-expanded={isOpen}
        aria-labelledby={`${id}-label`}
        aria-controls={listboxId}
        onClick={() => setIsOpen(o => !o)}
        onKeyDown={handleButtonKeyDown}
      >
        {selected.label}
        <i aria-hidden="true">▼</i>
      </button>

      {isOpen && (
        <ul
          ref={listRef}
          id={listboxId}
          role="listbox"
          aria-labelledby={`${id}-label`}
          aria-activedescendant={`${id}-option-${activeIndex}`}
        >
          {options.map((option, i) => (
            <li
              key={option.value}
              id={`${id}-option-${i}`}
              role="option"
              aria-selected={i === activeIndex}
              tabIndex={-1}                    // focusable programmatically
              onClick={() => selectOption(option, i)}
              onKeyDown={(e) => handleOptionKeyDown(e, i)}
            >
              {option.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

### Q40. Text Fragment API — how it works and SPA limitations

**Difficulty:** Mid

#### Answer

The Text Fragment API allows deep-linking to specific text on a page via URL:

```
https://example.com/docs#:~:text=getting%20started
https://example.com/docs#:~:text=start-,Getting%20Started,-with%20React
                                  prefix  highlighted text  suffix
```

**How the browser processes it:**
1. Page loads
2. Browser parses `#:~:text=...` fragment
3. Scrolls to and highlights the matched text (yellow highlight by default)
4. `:target-text` pseudo-element lets you style the highlight

```css
::target-text {
  background: yellow;
  color: black;
}
```

**The SPA problem — `history.replaceState` strips the fragment:**

```js
// Angular router, React Router, Vue Router all do this internally:
history.replaceState(null, "", "/current-path");
// ❌ This strips #:~:text=... from the URL
// User shares the link — recipient opens page without the highlight

// The bug flow:
// 1. User navigates to /docs#:~:text=authentication
// 2. Angular router processes the route
// 3. Router calls history.replaceState(state, "", "/docs") — strips fragment
// 4. Page loads. No text highlighted. User confused.
```

**Workaround:**
```ts
// In Angular router config — preserve fragment
RouterModule.forRoot(routes, {
  scrollPositionRestoration: "enabled",
  anchorScrolling: "enabled",
  // Don't use withInMemoryScrolling if you need text fragments — it triggers replaceState
})

// Manual workaround: re-apply text fragment after navigation
router.events.pipe(
  filter(e => e instanceof NavigationEnd)
).subscribe(() => {
  const textFragment = window.location.hash.includes(":~:text=");
  if (textFragment) {
    // Fragments already in URL — nothing to restore
    // If stripped, would need to save it before navigation and restore
  }
});
```

**Detection:**
```js
// Check if browser supports Text Fragments
const supportsTextFragments = "fragmentDirective" in document;
// Chrome 80+, Edge 83+, Opera 67+ — Firefox does NOT support as of 2026
```

---

## Evaluation Scoring Guide

| Category | 1 (Poor) | 3 (Average) | 5 (Excellent) |
|----------|----------|-------------|---------------|
| JS Fundamentals | Knows syntax only | Understands async, closures | Explains V8, event loop, memory |
| Framework Expertise | Uses framework | Knows lifecycle, patterns | Internals, compiler, signals |
| Problem Solving | Stuck without hints | Solves with guidance | Edge cases, trade-offs upfront |
| Code Clarity | Works but messy | Readable, some structure | Clean, typed, reusable |
| System Design | Component-level thinking | Considers state/API | Covers perf, a11y, scale |
| Communication | Vague, needs prompting | Clear with examples | Explains why, trade-offs |

---

## Final Verdict Framework

| Score Range | Verdict |
|-------------|---------|
| 27–30 | ✅ Strong Hire |
| 22–26 | ✅ Hire |
| 16–21 | 🟡 Lean Hire |
| <16 | ❌ No Hire |

---

*Last updated: 2026 — covers React 19, Angular 19, TypeScript 5.x, Core Web Vitals (INP era)*
