# React.js — Deep-Dive Interview Handbook

> Structured as: **Concept → Internals → Code → Real-world Example → Best Practices → Common Mistakes → Follow-up Questions**. Tricky questions flagged 🎯.

---

## Table of Contents
1. Core Fundamentals (JSX, Virtual DOM, Fiber, Reconciliation)
2. Components, Props, State
3. Hooks Deep Dive (useState, useEffect, useMemo, useCallback, useRef, useContext, useReducer, custom hooks)
4. Rendering Behavior & Keys
5. Performance Optimization
6. State Management (Context API, Redux, Zustand)
7. React Router
8. Forms (Controlled vs Uncontrolled)
9. Class Component Lifecycle → Hooks Mapping
10. Error Boundaries
11. Testing (Jest, React Testing Library)
12. Server-Side Rendering & Next.js Basics
13. Design Patterns (HOC, Render Props, Compound Components)
14. React 18/19 Features (Concurrent Rendering, Suspense, Transitions)
15. Rapid-Fire Tricky Q&A

---

## 1. Core Fundamentals

### Q1.1 — What is the Virtual DOM, and how does reconciliation actually work internally?

**Answer:** The Virtual DOM (VDOM) is a lightweight, in-memory JavaScript representation of the actual DOM. Instead of mutating the real DOM directly on every state change (expensive — triggers layout/reflow/repaint), React:
1. Builds a new VDOM tree representing the desired UI state.
2. **Diffs** it against the previous VDOM tree (the "reconciliation" algorithm).
3. Computes the minimal set of actual DOM mutations needed.
4. Applies ("commits") only those changes to the real DOM in a single batch.

**Reconciliation heuristics (React's diffing algorithm is O(n), not the theoretical O(n³) tree-diff — achieved via two key assumptions):**
1. **Different component types produce different trees** — if an element's type changes (`<div>` → `<span>`, or `<ComponentA>` → `<ComponentB>`), React tears down the old subtree entirely and builds a new one from scratch (including unmounting all children and losing their state) rather than trying to diff their internals.
2. **Keys hint stable identity** — for lists, `key` props let React match elements across renders even if their order changes, avoiding unnecessary unmount/remount and preserving component state correctly.

**Fiber Architecture (React 16+) — the internal reconciliation engine:**
- Before Fiber (React 15 "Stack Reconciler"), reconciliation was synchronous and **recursive** — once started, it couldn't be paused, meaning a large tree update could block the main thread for one long stretch, causing jank/dropped frames on animations/input.
- **Fiber** reimplemented reconciliation as an **incremental, interruptible, unit-of-work-based** linked-list/tree structure. Each "fiber" node represents a unit of work (roughly, one component instance). React can process fibers in **small chunks**, yield back to the browser to handle higher-priority work (user input, animations), and resume later — enabling **concurrent rendering** (see Section 14).
- Fiber has **two phases**:
  - **Render/Reconciliation phase** (can be paused/interrupted/discarded — "may be async"): builds the new "work-in-progress" tree, computes diffs.
  - **Commit phase** (always synchronous, cannot be interrupted): applies the computed DOM mutations, runs lifecycle methods/effects (`componentDidMount`, `useLayoutEffect`).

**Follow-up 🎯:** *"Why can't the render phase have side effects (like API calls or DOM mutations)?"* → Because the render phase can be paused, discarded, or **re-run multiple times** by React (e.g., during concurrent rendering, or when React throws away in-progress work due to a higher-priority update) before committing — any side effect there could run multiple times or run for a state that's later discarded, causing bugs. Side effects belong in the commit phase / `useEffect` (which specifically runs *after* commit).

---

### Q1.2 — What is JSX, and what does it actually compile to?

**Answer:** JSX is syntactic sugar over `React.createElement()` calls, transpiled by Babel (or the TypeScript compiler) at build time — it's **not** understood by browsers natively.

```jsx
const element = <h1 className="greeting">Hello, {name}</h1>;
```
compiles to (classic runtime):
```js
const element = React.createElement('h1', { className: 'greeting' }, 'Hello, ', name);
```
Since React 17's **new JSX transform**, it instead compiles to calls into `react/jsx-runtime` (`jsx()`/`jsxs()`), which no longer requires `React` to be in scope in every file (eliminating the old mandatory `import React from 'react'` boilerplate) and is slightly more optimized.

`React.createElement()` returns a plain JavaScript object (the "React element") describing the desired UI — **not** an actual DOM node:
```js
{ type: 'h1', props: { className: 'greeting', children: ['Hello, ', name] }, key: null, ref: null }
```

**Follow-up:** *"Is a React Element the same as a Component instance?"* → No — an **Element** is a plain, immutable, lightweight description object (cheap to create). A **Component instance** is the actual stateful object React creates and manages internally when it processes that element (for class components) or the function invocation + its Fiber node (for function components with hooks). You create many elements on every render; React manages the instances/fibers across renders.

---

## 2. Components, Props, State

### Q2.1 — Functional Components vs Class Components — why did React shift toward functional + hooks?

**Answer:**

| | Class Components | Functional Components + Hooks |
|---|---|---|
| State | `this.state` + `this.setState()` | `useState`/`useReducer` |
| Lifecycle | Explicit methods (`componentDidMount`, etc.) | `useEffect` (unified mental model) |
| `this` binding | Common source of bugs (must bind handlers in constructor or use arrow-function class fields) | No `this` — no binding issues |
| Code reuse across components | HOCs / Render Props (verbose, "wrapper hell") | Custom Hooks — clean, composable logic reuse |
| Boilerplate | More verbose | Concise |
| Logic organization | Related logic scattered across different lifecycle methods (e.g., subscribe in `componentDidMount`, unsubscribe in `componentWillUnmount` — split across the file) | Related logic can live together in one `useEffect` (setup + cleanup co-located) |

**Why the shift:** Hooks (introduced React 16.8) solved three specific, long-standing class-component pain points:
1. **Reusing stateful logic** was awkward — Higher-Order Components and Render Props both work but introduce extra component nesting ("wrapper hell" in dev tools) and indirection. Custom Hooks let you extract/reuse stateful logic as a plain function with no extra component tree nesting.
2. **Complex components became hard to understand** — lifecycle methods forced you to split logically-related code across `componentDidMount`/`componentDidUpdate`/`componentWillUnmount`, while unrelated logic (e.g., a data subscription vs an event listener) got merged together in the same lifecycle method.
3. **`this` confusion** — a very common source of bugs for both beginners and experienced JS developers, especially with callback handlers.

**Follow-up:** *"Are class components deprecated?"* → Not formally deprecated/removed, and Error Boundaries currently still require a class component (no Hook equivalent exists as of React 18/19) — but the React team's official guidance and virtually all new documentation/ecosystem tooling is Hooks-first; new code should default to functional components.

---

### Q2.2 — Props vs State — and why is "lifting state up" a core React pattern?

**Answer:**
- **Props:** read-only data passed **from parent to child** — a component must never mutate its own props (violates React's one-way data flow / makes the parent's data untrustworthy).
- **State:** data owned and managed **within** a component, which can change over time and triggers a re-render when updated via `setState`/the state setter function.

**Lifting State Up:** When two or more sibling components need to share/stay in sync on some piece of state, move that state to their closest common ancestor, and pass it down as props (with callback props for children to request changes) — since React has **one-way data flow** (data flows down via props, changes flow up via callback functions), there's no built-in two-way binding between arbitrary components, so shared state must live at a common ancestor.

```jsx
function Parent() {
  const [temperature, setTemperature] = useState(20);
  return (
    <>
      <CelsiusInput value={temperature} onChange={setTemperature} />
      <FahrenheitDisplay celsius={temperature} />
    </>
  );
}
```

**Follow-up 🎯:** *"When does lifting state up become a problem, and what's the alternative?"* → When state needs to be shared across many components far apart in the tree, lifting it all the way to a common top-level ancestor causes **prop drilling** (passing props through many intermediate components that don't themselves need that data, just to get it to a deeply nested child) — solved via **Context API** (for moderate cases) or a dedicated state management library like **Redux/Zustand** (for complex, frequently-updated, widely-shared state) — see Section 6.

---

## 3. Hooks Deep Dive

### Q3.1 — `useState` internals: how does React know which state belongs to which `useState` call across re-renders?

**Answer:** React maintains state for each component instance as **a linked list of "hook" objects attached to that component's Fiber node**. On each render, React walks this list **in the exact order the hooks were called** in the function body, matching the Nth `useState`/`useEffect`/etc. call in this render to the Nth hook object in the list from the previous render.

**This is exactly why the Rules of Hooks exist:**
1. **Only call hooks at the top level** (never inside loops, conditions, or nested functions) — because the call order must be **identical on every render** for React's positional matching to correctly associate state with the right hook slot. If you conditionally skip a hook call one render, all subsequent hooks in the list shift position and get mismatched with the wrong stored state.
2. **Only call hooks from React functions** (function components or custom hooks) — not regular JS functions.

```jsx
function Counter() {
  const [count, setCount] = useState(0);       // hook slot 0
  const [name, setName] = useState('');         // hook slot 1
  useEffect(() => { document.title = `${count}`; }); // hook slot 2
  // If you wrapped the second useState in `if (count > 0) { ... }`,
  // on renders where count === 0, slot 1 would never be created,
  // and slot 2 (useEffect) would shift into slot 1's position — corruption.
}
```

**Functional updates (`setCount(prev => prev + 1)`) vs direct updates (`setCount(count + 1)`):** Since state updates are **asynchronous/batched**, using the current `count` variable (captured via closure from the render that scheduled the update) can lead to stale values if multiple updates are queued before a re-render happens. The functional form receives the **latest** pending state as its argument, guaranteeing correctness regardless of batching:
```jsx
// BUG: if called twice in the same event handler, both use the same stale `count` closure — only increments by 1, not 2
setCount(count + 1);
setCount(count + 1);

// CORRECT: each functional update receives the actual latest state
setCount(prev => prev + 1);
setCount(prev => prev + 1);
```

**Follow-up:** *"Why does calling `setState` with the same value it already has still sometimes cause issues, or NOT cause a re-render?"* → React uses `Object.is` comparison (bailout mechanism) to skip re-rendering if the new state value is referentially/primitively identical to the current one — this is why mutating an object/array in place and calling `setState(sameReference)` **does NOT trigger a re-render** (a very common bug — always create a new object/array reference when updating object/array state: `setItems([...items, newItem])`, never `items.push(newItem); setItems(items)`).

---

### Q3.2 🎯 — `useEffect` deep dive: dependency array behavior, cleanup functions, and the stale closure problem.

**Answer:**

```jsx
useEffect(() => {
  const subscription = subscribeToData(id, data => setData(data));
  return () => subscription.unsubscribe(); // cleanup function
}, [id]); // dependency array
```

**Dependency array semantics:**
- `[]` (empty array) — effect runs once after the initial mount only (cleanup runs once on unmount).
- No array at all — effect runs after **every** render (rarely what you want — usually a bug).
- `[id]` — effect re-runs whenever `id` changes between renders (compared via `Object.is`, shallow comparison — this is why passing a new object/array/function literal as a dependency on every render, even if "logically" the same, causes the effect to re-run every time — a very common performance bug, fixed with `useMemo`/`useCallback`).

**Cleanup function timing:** Runs **before** the effect re-runs (for the next execution) AND when the component unmounts — critical for avoiding memory leaks (unsubscribing listeners, cancelling in-flight requests/timers) and avoiding "setting state on an unmounted component" warnings.

**The Stale Closure Problem:** A function/effect captures variables from the render in which it was created ("closes over" them). If you don't include a changing variable in the dependency array, the effect keeps referencing the **old** value from whichever render it was created in — a classic, extremely common React bug.

```jsx
// BUG: interval always logs the INITIAL count (0), never updates, because the effect
// only runs once ([] dependency) and its closure captured `count` from that first render
useEffect(() => {
  const interval = setInterval(() => { console.log(count); }, 1000);
  return () => clearInterval(interval);
}, []); // missing `count` dependency

// FIX 1: include the dependency (re-creates the interval each time count changes — sometimes undesirable)
useEffect(() => { ... }, [count]);

// FIX 2: use functional state update to avoid needing `count` in the closure at all
useEffect(() => {
  const interval = setInterval(() => { setCount(prev => prev + 1); }, 1000);
  return () => clearInterval(interval);
}, []); // no dependency on count needed since we never read it directly
```

**`useEffect` vs `useLayoutEffect`:** `useEffect` runs **asynchronously after the browser paints** the screen (doesn't block visual updates — better for most side effects like data fetching, subscriptions, logging). `useLayoutEffect` runs **synchronously after DOM mutations but before the browser paints** — used when you need to read layout (`getBoundingClientRect`) and synchronously adjust the DOM/state **before** the user sees a visual flicker (e.g., measuring an element's size to position a tooltip correctly on first render).

**Best Practices:**
- Always specify the dependency array explicitly and completely — use the `eslint-plugin-react-hooks` `exhaustive-deps` rule; don't silence it without a very good, documented reason.
- Split unrelated concerns into **separate** `useEffect` calls (one effect per logical concern), don't combine everything into one giant effect.
- Always clean up subscriptions/timers/listeners to avoid memory leaks and "can't perform state update on unmounted component" warnings.

**Common Mistakes:**
- Missing dependencies → stale closures (as above).
- Object/array/function literals as dependencies without memoization → effect re-runs every render, defeating the dependency array's purpose entirely.
- Fetching data in `useEffect` without handling race conditions (e.g., user rapidly changes a search query — an older, slower request can resolve *after* a newer one, overwriting fresher data with stale results) — fix with an `ignore`/`AbortController` flag in the cleanup function:
```jsx
useEffect(() => {
  let ignore = false;
  fetchData(query).then(result => { if (!ignore) setData(result); });
  return () => { ignore = true; };
}, [query]);
```

**Follow-up:** *"How does `useEffect` differ from `componentDidMount`/`componentDidUpdate`/`componentWillUnmount` conceptually?"* → Class lifecycle methods are organized around **when** code runs (mount/update/unmount phases), forcing you to split related logic (setup and its corresponding teardown) across different methods. `useEffect` is organized around **what** the code is synchronizing (a single concern's setup + cleanup co-located in one function) — a fundamentally different, generally cleaner mental model ("synchronizing with an external system" rather than "responding to a lifecycle event").

---

### Q3.3 — `useMemo` vs `useCallback` vs `React.memo` — when do you actually need them (and when do they hurt)?

**Answer:**

| | Purpose | Returns |
|---|---|---|
| `useMemo(fn, deps)` | Memoize an **expensive computed value** — recompute only when deps change | The computed value itself |
| `useCallback(fn, deps)` | Memoize a **function reference** — return the same function identity across renders unless deps change | The function itself (unchanged reference) |
| `React.memo(Component)` | Memoize a **component** — skip re-rendering if props are shallowly equal to the previous render | A wrapped component |

```jsx
const expensiveResult = useMemo(() => computeExpensiveValue(a, b), [a, b]);

const handleClick = useCallback(() => { doSomething(id); }, [id]);

const ExpensiveChild = React.memo(function ExpensiveChild({ onClick, data }) {
  // only re-renders if `onClick` or `data` reference changes
});
```

**Why `useCallback` matters specifically with `React.memo`:** If a parent re-renders and passes a **new function reference** to a memoized child on every render (`onClick={() => doSomething()}` inline, a new function created every render), `React.memo`'s shallow prop comparison sees a "changed" prop every time, defeating the memoization entirely — `useCallback` keeps the reference stable so `React.memo` can actually skip the re-render.

**🎯 The critical nuance interviewers probe: memoization is NOT free.** `useMemo`/`useCallback` themselves have a cost (storing the previous deps + value, comparing deps every render) — for cheap computations/simple components, this overhead can exceed the cost of just recomputing/re-rendering normally. **Rule of thumb:**
- Use `useMemo` for genuinely expensive computations (heavy loops, large data transformations) — not for trivial arithmetic.
- Use `useCallback` primarily when passing callbacks to memoized children (`React.memo`) or as a dependency of another hook (`useEffect`) where referential stability actually matters — not reflexively on every function in a component.
- Don't wrap every component in `React.memo` "just in case" — for components that re-render with genuinely different props every time anyway (common case), the shallow-comparison overhead is pure waste with zero benefit.

**Best Practice:** Profile first (React DevTools Profiler) — only reach for memoization on **measured, real** performance bottlenecks, not preemptively everywhere ("premature optimization" applies strongly here, and over-memoized code is harder to read/maintain for no real gain in many cases).

**Follow-up:** *"What comparison does React.memo use by default, and can you customize it?"* → Shallow comparison of each prop (`Object.is` per prop, one level deep — doesn't deeply compare nested object contents). You can pass a custom comparison function as `React.memo(Component, areEqualFn)` for cases needing deeper/custom equality logic — though a custom deep-equality check itself has a cost, so use judiciously.

---

### Q3.4 — `useRef` — what is it really for, and how is it different from state?

**Answer:** `useRef` returns a mutable object (`{ current: value }`) that **persists across renders** without causing a re-render when mutated — the opposite tradeoff from `useState` (which persists across renders AND triggers a re-render on change).

**Two primary use cases:**
1. **Accessing/holding a reference to a DOM node** directly (imperative DOM access — e.g., focusing an input, measuring an element, integrating a non-React library like a chart or map).
```jsx
function TextInput() {
  const inputRef = useRef(null);
  useEffect(() => { inputRef.current.focus(); }, []);
  return <input ref={inputRef} />;
}
```
2. **Storing any mutable value that shouldn't trigger a re-render when changed** — e.g., a timer ID, a previous value for comparison, an instance variable equivalent to what would be a class field in a class component (a counter that tracks something across renders without needing the UI to update on every change).

```jsx
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => { ref.current = value; });
  return ref.current; // returns value from the PREVIOUS render (a classic custom hook pattern)
}
```

**Follow-up 🎯:** *"Why doesn't mutating `ref.current` trigger a re-render, and when could that actually be a bug?"* → By design — refs are explicitly for values that live "outside" React's render/state cycle. The bug case: if you need the UI to reflect a value's change, using a ref instead of state means the screen simply won't update even though the underlying value did change — a common beginner mistake is reaching for `useRef` to "avoid re-renders" for something that actually needs to be visually reflected, which requires `useState` instead.

---

### Q3.5 — `useContext` — how does it work, and what's its main performance pitfall?

**Answer:** Context provides a way to pass data through the component tree without manually threading props through every intermediate level (solves prop drilling — see Q2.2).

```jsx
const ThemeContext = createContext('light');

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Toolbar />
    </ThemeContext.Provider>
  );
}
function Toolbar() { return <ThemedButton />; } // doesn't need to know about theme at all
function ThemedButton() {
  const theme = useContext(ThemeContext); // reads directly, no prop drilling through Toolbar
  return <button className={theme}>Click</button>;
}
```

**🎯 The major performance pitfall:** **Every component consuming a Context re-renders whenever the Provider's `value` changes — regardless of whether that specific consumer actually cares about the part of the value that changed.** If you put a large object with many unrelated fields into one Context, updating any single field re-renders **every** consumer of that Context, even ones only using unrelated fields.

```jsx
// ANTI-PATTERN: one big context object for everything
<AppContext.Provider value={{ user, theme, cart, notifications }}>
// updating `notifications` re-renders every component using useContext(AppContext),
// including ones that only care about `theme` — massive unnecessary re-render cascade
```

**Fixes:**
- **Split into multiple, more granular contexts** (`UserContext`, `ThemeContext`, `CartContext`) so consumers only subscribe to (and re-render for) the specific slice of state they actually use.
- **Memoize the context value** with `useMemo` — if the Provider component itself re-renders for unrelated reasons but the actual value content hasn't changed, memoizing prevents creating a new object reference (which would otherwise still trigger consumer re-renders even with logically identical content, since Context comparison is reference-based, not deep-equality-based).
```jsx
const value = useMemo(() => ({ user, theme }), [user, theme]);
<AppContext.Provider value={value}>
```
- For high-frequency updates (e.g., mouse position, real-time data), Context is often the **wrong tool entirely** — a dedicated state management library with fine-grained subscriptions (Zustand, Jotai, Redux with selectors) avoids the "any change re-renders all consumers" problem structurally, since they support subscribing to a narrow slice of state rather than the whole context value.

**Follow-up:** *"Is Context a state management solution, or just a dependency-injection mechanism?"* → Fundamentally the latter — Context itself doesn't provide state update logic, subscriptions to partial slices, or performance optimizations for frequent updates; it's typically paired with `useState`/`useReducer` to actually manage the state, with Context just being the delivery mechanism to avoid prop drilling. For genuinely complex/frequently-changing global state, purpose-built state libraries usually outperform raw Context.

---

### Q3.6 — `useReducer` vs `useState` — when do you reach for `useReducer`?

**Answer:** `useReducer` is preferable when:
- State transitions are **complex** (multiple sub-values that update together, or the next state depends on the previous state in non-trivial ways).
- You want to **centralize and test update logic** separately from the component (a pure `reducer(state, action)` function is trivially unit-testable in isolation).
- Many event handlers update the same piece of state in different ways — a reducer with named `action.type`s keeps this organized and readable compared to many scattered `setState` calls.
- You need to pass a **dispatch function** down to deeply nested children (dispatch is stable across renders, unlike passing many individual setter callbacks — pairs very well with Context to avoid prop drilling of update logic).

```jsx
function cartReducer(state, action) {
  switch (action.type) {
    case 'ADD_ITEM': return { ...state, items: [...state.items, action.payload] };
    case 'REMOVE_ITEM': return { ...state, items: state.items.filter(i => i.id !== action.payload) };
    case 'CLEAR': return { ...state, items: [] };
    default: throw new Error(`Unknown action: ${action.type}`);
  }
}
function Cart() {
  const [state, dispatch] = useReducer(cartReducer, { items: [] });
  return <button onClick={() => dispatch({ type: 'ADD_ITEM', payload: newItem })}>Add</button>;
}
```

**Follow-up:** *"Is `useReducer` essentially a mini-Redux inside one component?"* → Conceptually yes — same core pattern (pure reducer function, action-based dispatch, unidirectional data flow) that Redux applies at the whole-app level. `useReducer` + Context is a legitimate lightweight alternative to Redux for moderate app-wide state needs, without pulling in an external library — though it lacks Redux's ecosystem (devtools time-travel debugging, middleware, established patterns for large teams).

---

## 4. Rendering Behavior & Keys

### Q4.1 🎯 — Why does React need `key` props for lists, and why is using the array `index` as a key often a bug?

**Answer:** `key` gives React a stable identity for each element in a list across renders, letting it correctly match old elements to new ones during reconciliation (Q1.1) — determining which elements were added, removed, reordered, or updated, rather than assuming positional correspondence.

**Why `index` as key is problematic:** If the list can be **reordered, filtered, or have items inserted/removed from the middle** (not just appended at the end), using the array index as the key means React associates each index-based key with whatever *position* it's in, not the actual logical item — causing:
1. **Incorrect state association** — if list items are stateful components (e.g., each row has its own `useState` for an "expanded" toggle, or contains an uncontrolled `<input>`), removing an item from the middle can cause the WRONG remaining item to inherit the removed item's DOM node/state, since React matches by (index-based) key, not by actual data identity.
2. **Unnecessary re-renders/DOM mutations** — even when the underlying data for an item didn't change, if its position did, React sees a "different" key-matched item at that index and may re-render/recreate DOM more than necessary.

```jsx
// BUG-PRONE: if items can be reordered/filtered/inserted mid-list
{items.map((item, index) => <ListItem key={index} data={item} />)}

// CORRECT: use a stable, unique identifier from the actual data
{items.map(item => <ListItem key={item.id} data={item} />)}
```

**When is `index` as key actually fine?** Only when the list is **static** (never reordered/filtered/has items inserted/removed from anywhere but the end) — e.g., a hardcoded, unchanging list of menu items rendered once.

**Follow-up:** *"What happens if you don't provide a key at all?"* → React uses the array index implicitly (same problem as above) AND logs a console warning — always provide explicit, stable keys for dynamic lists.

---

### Q4.2 — Explain React's batching behavior — how does it differ between React 17 and React 18?

**Answer:** **Batching** means React groups multiple `setState` calls that happen within the same event/execution context into a **single** re-render, instead of re-rendering after each individual state update — a significant performance optimization.

```jsx
function handleClick() {
  setCount(c => c + 1);
  setFlag(f => !f);
  // Pre-React 18: batched automatically ONLY inside React event handlers (onClick, etc.)
  // → results in exactly ONE re-render for both updates combined
}

function handleFetch() {
  fetchData().then(() => {
    setCount(c => c + 1); // Pre-React 18: NOT batched — this is inside a Promise callback,
    setFlag(f => !f);      // outside React's own event handling — causes TWO separate re-renders
  });
}
```

**React 18's "Automatic Batching":** Extends batching to **all** state updates, regardless of where they originate — Promises, `setTimeout`, native event handlers, or anywhere else — not just inside React's synthetic event handlers. This is a genuinely valuable, "free" performance improvement upon upgrading to React 18 with the new root API (`createRoot`).

**Opting out (rare, for cases where you deliberately need synchronous, un-batched updates):**
```jsx
import { flushSync } from 'react-dom';
function handleClick() {
  flushSync(() => { setCount(c => c + 1); }); // forces this update to apply + re-render immediately
  flushSync(() => { setFlag(f => !f); });      // ...separately, before continuing
}
```

**Follow-up:** *"Why does React 18 require using `createRoot` instead of the legacy `ReactDOM.render` to get automatic batching?"* → Automatic batching (and concurrent rendering features generally) are part of React 18's new "Concurrent Renderer" opt-in root API — using the legacy `ReactDOM.render` API keeps the old React 17 rendering behavior for backward compatibility, even on React 18's library version, so upgrading the npm package alone isn't sufficient; you must also switch to `createRoot(container).render(<App />)` to actually get the new behaviors.

---

## 5. Performance Optimization

### Q5.1 — What are the main levers for optimizing a slow-rendering React app, in order of typical impact?

**Answer:**

1. **Fix unnecessary re-renders first (usually the biggest win):**
   - Identify with **React DevTools Profiler** (record a session, look for components re-rendering without a corresponding prop/state change — the "why did this render" highlighting feature is invaluable here).
   - Apply `React.memo`/`useMemo`/`useCallback` **surgically** where profiling shows real cost (Q3.3) — not everywhere.
   - Split large components into smaller ones so unrelated state changes don't force large subtrees to re-render.
   - Fix Context misuse causing broad re-render cascades (Q3.5).

2. **Code splitting / lazy loading** — reduce initial bundle size so the app becomes interactive faster:
```jsx
const AdminPanel = React.lazy(() => import('./AdminPanel'));
function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <AdminPanel />
    </Suspense>
  );
}
```
Route-based code splitting (loading each page's JS bundle only when that route is visited) is the most common, highest-value application of this.

3. **Virtualize long lists** — rendering thousands of DOM nodes for a long list is expensive even with perfect memoization; libraries like `react-window`/`react-virtualized` render only the currently-visible subset of rows (plus a small buffer), dramatically reducing DOM node count.

4. **Debounce/throttle high-frequency updates** — e.g., search-as-you-type inputs shouldn't trigger a re-render/API call on every keystroke; debounce the state update or the downstream effect.

5. **Avoid unnecessary work in the render path** — expensive computations inline in the render function run on every re-render; move genuinely expensive ones behind `useMemo`, or better, out of the render path entirely if possible (e.g., precompute once, or move to a Web Worker for truly heavy CPU work that would otherwise block the main thread).

6. **Use React 18's `useTransition`/`useDeferredValue`** for deprioritizing non-urgent updates (Section 14) — lets urgent updates (typing responsiveness) stay snappy while expensive downstream rendering (e.g., filtering a huge list based on that typed input) happens at lower priority without blocking the UI.

**Best Practice:** Always measure (Profiler, Lighthouse, Web Vitals) before and after optimizing — intuition about "what's slow" in React is frequently wrong, and unnecessary memoization adds real code complexity/maintenance cost for zero or even negative benefit in many cases.

**Follow-up:** *"What's the difference between optimizing render performance vs optimizing bundle/load performance — do they require different tools?"* → Yes — render performance (re-render frequency/cost) is diagnosed with the React DevTools Profiler; load/bundle performance (initial page load speed, Time to Interactive) is diagnosed with browser Network/Performance tabs, Lighthouse, and bundle analyzers (`webpack-bundle-analyzer`) — different problems, different tools, and optimizing one doesn't automatically fix the other.

---

## 6. State Management

### Q6.1 — Context API vs Redux vs Zustand — how do you choose?

**Answer:**

| | Context API + useReducer | Redux (Toolkit) | Zustand |
|---|---|---|---|
| Boilerplate | Low-moderate | Historically high; **Redux Toolkit** (RTK) significantly reduced this (built-in `createSlice`, Immer for "mutable-looking" immutable updates, `configureStore` defaults) | Very low |
| Re-render granularity | Coarse — any Provider value change re-renders all consumers unless carefully split (Q3.5) | Fine-grained via `useSelector` — components only re-render when their **selected slice** changes | Fine-grained via selector functions in the hook itself |
| DevTools | None built-in | Excellent — Redux DevTools with time-travel debugging, action history | Basic (can integrate with Redux DevTools) |
| Middleware/side-effect handling | Manual | Rich ecosystem (`redux-thunk`, `redux-saga`, RTK Query for data fetching) | Simpler built-in async support, less ecosystem tooling |
| Learning curve | Low (built into React) | Moderate (even with RTK — actions, reducers, store, selectors as concepts) | Very low — minimal API surface |
| Best for | Small-to-moderate apps, infrequent updates, avoiding an extra dependency | Large apps, complex state interactions, teams wanting strict conventions/strong tooling/time-travel debugging | Small-to-large apps wanting simplicity without Redux's ceremony |

```js
// Zustand — minimal boilerplate example
const useCartStore = create((set) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
  clear: () => set({ items: [] }),
}));

function Cart() {
  const items = useCartStore((state) => state.items); // fine-grained subscription — only re-renders if `items` changes
  const addItem = useCartStore((state) => state.addItem);
}
```

**Best Practice / senior-level answer:** *"Don't reach for a global state library by default — most state is genuinely local (`useState` in the component that needs it) or shared among a small, nearby group of components (lift state up, or Context for that specific subtree). Reach for Redux/Zustand specifically when state is truly global, updated frequently from many places, and Context's coarse re-render behavior becomes a measured problem — not preemptively on day one of a new project."*

**Follow-up:** *"What is 'server state' and why do libraries like React Query / TanStack Query exist separately from Redux/Zustand?"* → **Server state** (data fetched from an API — inherently asynchronous, can go stale, may be cached/shared across components, needs refetching/invalidation logic) is fundamentally different from **client state** (UI-only state — form inputs, modal open/closed, theme). React Query/SWR specialize in server state concerns (caching, background refetching, deduplication of in-flight requests, stale-while-revalidate patterns) that general-purpose state libraries like Redux weren't originally designed for — many modern apps use React Query for server state and a lighter tool (or just `useState`/Context) for genuine client-only state, rather than stuffing fetched API data into Redux.

---

## 7. React Router

### Q7.1 — Explain client-side routing concepts and nested routes in React Router (v6+).

**Answer:** Client-side routing intercepts navigation (link clicks, back/forward browser buttons) using the **History API** (`pushState`/`popState`) instead of triggering a full page reload — the SPA swaps which components render based on the current URL, without a round-trip to the server for a new HTML document.

```jsx
function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Layout />}>
          <Route index element={<Home />} />
          <Route path="orders" element={<Orders />}>
            <Route path=":orderId" element={<OrderDetail />} /> {/* nested + dynamic param */}
          </Route>
          <Route path="*" element={<NotFound />} /> {/* catch-all */}
        </Route>
      </Routes>
    </BrowserRouter>
  );
}
function Layout() {
  return (
    <div>
      <Nav />
      <Outlet /> {/* nested route's matched element renders here */}
    </div>
  );
}
function OrderDetail() {
  const { orderId } = useParams(); // access dynamic URL segment
  const navigate = useNavigate();  // programmatic navigation
}
```

**Key concepts:** `<Outlet />` is the placeholder where a matched child route renders within a parent layout — enables shared layouts (nav bars, sidebars) without re-rendering them on every child-route navigation. **Route-based code splitting** pairs naturally with routing (Q5.1) — each route's component lazy-loaded only when visited.

**Follow-up:** *"How would you protect a route so only authenticated users can access it?"* → A wrapper component checking auth state, rendering `<Outlet />`/children if authenticated, or `<Navigate to="/login" />` if not:
```jsx
function ProtectedRoute() {
  const { isAuthenticated } = useAuth();
  return isAuthenticated ? <Outlet /> : <Navigate to="/login" replace />;
}
// usage: <Route element={<ProtectedRoute />}><Route path="/dashboard" element={<Dashboard />} /></Route>
```

---

## 8. Forms

### Q8.1 — Controlled vs Uncontrolled Components — tradeoffs and when to use each.

**Answer:**

| | Controlled | Uncontrolled |
|---|---|---|
| Source of truth | React state (`value` + `onChange`) | The DOM itself (accessed via `ref` when needed) |
| Re-renders | On every keystroke (state update triggers re-render) | None from typing — DOM manages its own value internally |
| Validation/formatting-as-you-type | Easy (intercept `onChange`) | Harder (must read DOM value explicitly) |
| Performance for very large/complex forms | Can degrade with many controlled fields re-rendering the whole form on every keystroke (mitigated by splitting into sub-components, or a form library) | Better raw performance (no React re-render per keystroke) |

```jsx
// Controlled
function ControlledInput() {
  const [value, setValue] = useState('');
  return <input value={value} onChange={e => setValue(e.target.value)} />;
}

// Uncontrolled
function UncontrolledInput() {
  const ref = useRef();
  const handleSubmit = () => console.log(ref.current.value); // read on demand, not on every keystroke
  return <input ref={ref} defaultValue="" />;
}
```

**Best Practice:** Use controlled components for most cases (simpler mental model, easier validation, React is source of truth) — reach for uncontrolled + a form library (**React Hook Form**, which is built around uncontrolled inputs + refs internally for performance) for large/complex forms where per-keystroke re-render overhead across many fields becomes a measured problem.

**Follow-up:** *"Why does React Hook Form generally outperform a fully-controlled form implementation for large forms?"* → It keeps most fields **uncontrolled** internally (reading values via refs on submit/validation trigger, using the DOM as the source of truth) rather than driving every field through React state + re-render on every keystroke — drastically reducing re-render count for forms with many fields, while still providing a declarative validation API on top.

---

## 9. Class Component Lifecycle → Hooks Mapping

**Answer (a very common "translate this" question):**

| Class Lifecycle Method | Hooks Equivalent |
|---|---|
| `constructor` | `useState` initial value (or a `useState(() => computeInitial())` lazy initializer for expensive initial state) |
| `componentDidMount` | `useEffect(() => { ... }, [])` |
| `componentDidUpdate` | `useEffect(() => { ... }, [dep1, dep2])` (runs after updates when deps change — also runs after mount, unlike the class method, which is a subtle difference to know) |
| `componentWillUnmount` | The cleanup function returned from `useEffect` |
| `shouldComponentUpdate` | `React.memo` (for skipping a component's re-render) or careful dependency/memoization design |
| `getDerivedStateFromProps` | Calculating a value directly during render (no hook needed — just compute inline) |
| `componentDidCatch` / `getDerivedStateFromError` | **No Hook equivalent — Error Boundaries still require a class component** (Section 10) |

**Follow-up 🎯:** *"Is there a Hooks equivalent of `componentDidCatch`? If not, why not?"* → No — as of React 18/19, Error Boundaries must be class components. The React team hasn't shipped a Hook-based error boundary API; the common workaround is to use a small, one-time class component wrapper (or a well-maintained community library like `react-error-boundary`) specifically for this purpose while keeping the rest of the app fully functional-component-based.

---

## 10. Error Boundaries

### Q10.1 — What are Error Boundaries, what do they NOT catch, and how do you implement one?

**Answer:** An Error Boundary is a class component implementing `static getDerivedStateFromError()` (to render a fallback UI) and/or `componentDidCatch()` (to log the error) — it catches JavaScript errors thrown during rendering, in lifecycle methods, and in constructors of its child component tree, preventing the **entire app** from unmounting/crashing due to one broken subtree.

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError(error) { return { hasError: true }; }
  componentDidCatch(error, info) { logErrorToService(error, info); }
  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}
// usage: <ErrorBoundary fallback={<ErrorPage />}><Dashboard /></ErrorBoundary>
```

**🎯 What Error Boundaries do NOT catch (a favorite trick question):**
1. Errors in **event handlers** (`onClick`, etc.) — these are just regular JS execution, not part of React's render tree; use a normal `try/catch` inside the handler instead.
2. Errors in **asynchronous code** (`setTimeout`, Promise callbacks, `fetch` calls) — same reason, not part of the synchronous render call stack React is monitoring.
3. Errors during **server-side rendering**.
4. Errors thrown **within the Error Boundary component itself** (an Error Boundary can't catch its own errors — a parent boundary further up the tree would be needed).

**Best Practice:** Place multiple, granular Error Boundaries around independent sections of the UI (e.g., one around a sidebar widget, a separate one around the main content area) rather than one single boundary around the entire app — so a crash in one isolated feature doesn't take down the whole page, just that section (graceful degradation).

**Follow-up:** *"How would you handle an error thrown inside an `onClick` handler, given Error Boundaries don't catch it?"* → Wrap the handler logic in a standard `try/catch`, and manually set state to trigger a fallback UI if desired (`setState` inside the catch block to show an error message), or re-throw it in a way that surfaces during the next render (some libraries provide a hook to manually trigger the nearest Error Boundary for exactly this case).

---

## 11. Testing (Jest & React Testing Library)

### Q11.1 — What's the philosophy behind React Testing Library (RTL), and how does it differ from Enzyme's older approach?

**Answer:** RTL's guiding principle: **"The more your tests resemble the way your software is used, the more confidence they can give you."** It deliberately avoids testing component internals (state values, instance methods, prop values directly) and instead queries the rendered DOM the way a real user would — by visible text, accessible roles/labels, placeholder text — and simulates real user interactions.

```jsx
import { render, screen, fireEvent } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

test('increments counter when button is clicked', async () => {
  render(<Counter />);
  const button = screen.getByRole('button', { name: /increment/i });
  await userEvent.click(button);
  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});

test('shows validation error for invalid email', async () => {
  render(<SignupForm />);
  await userEvent.type(screen.getByLabelText(/email/i), 'not-an-email');
  await userEvent.click(screen.getByRole('button', { name: /submit/i }));
  expect(await screen.findByText(/invalid email/i)).toBeInTheDocument(); // findBy* waits for async updates
});
```

**RTL vs Enzyme (older library, largely superseded):** Enzyme's `shallow()` rendering and APIs like `.state()`/`.instance()` encouraged testing **implementation details** (exact internal state shape, which specific method got called) — tests that break on harmless refactors (e.g., converting a class component to a function component, or renaming an internal state variable) even though the actual user-facing behavior didn't change. RTL's DOM-and-behavior-focused queries make tests robust to internal refactors, since they only break when actual user-observable behavior changes — a much better signal-to-noise ratio for a test suite's value.

**Query priority (RTL's own recommended order, favoring accessibility-friendly queries):** `getByRole` > `getByLabelText` > `getByPlaceholderText` > `getByText` > `getByTestId` (last resort — `data-testid` doesn't reflect anything a real user or assistive technology perceives, so it's the least preferred, used only when no more semantic query is feasible).

**Best Practices:**
- Test behavior/outcomes ("when I click X, Y appears"), not implementation ("state.count equals 1").
- Prefer `userEvent` over `fireEvent` — `userEvent` more accurately simulates full real browser interaction sequences (e.g., typing character-by-character with proper focus/blur events) rather than firing a single synthetic event.
- Use `findBy*` queries (which return a Promise, auto-retrying) for elements that appear after an async operation (data fetch, state update after a timeout) — avoid arbitrary `setTimeout`-based waits in tests.

**Common Mistakes:**
- Testing internal state/props directly instead of rendered output — brittle, low-value tests.
- Overusing `data-testid` when a semantic query (`getByRole`, `getByLabelText`) would work and also double as an accessibility check (if `getByRole` can't find your button, real screen-reader users likely can't either — a valuable side benefit).
- Not awaiting async queries/interactions, leading to flaky tests or "act() warning" console noise.

**Follow-up:** *"What is the `act()` warning, and why does it happen?"* → React batches and processes state updates within an `act()` wrapper during tests to ensure all resulting DOM updates/effects have flushed before your assertions run — RTL's `render`/`fireEvent`/`userEvent` helpers wrap calls in `act()` automatically in most cases, but a warning appears when a state update happens **outside** of an act()-wrapped call (commonly, an async operation resolving after the test function has already finished) — signaling your test isn't properly waiting for all pending updates before making assertions, which can mask real bugs or cause flakiness.

---

## 12. Server-Side Rendering & Next.js Basics

### Q12.1 — Why use SSR, and what are the tradeoffs vs Client-Side Rendering (CSR)?

**Answer:**

| | CSR (traditional SPA) | SSR | SSG (Static Generation) |
|---|---|---|---|
| Initial HTML | Near-empty shell; JS renders content client-side | Fully-rendered HTML sent from server on each request | Fully-rendered HTML pre-built at build time, served from CDN |
| Time to First Contentful Paint | Slower (must download+parse+execute JS bundle before anything visible) | Faster (HTML visible immediately, JS hydrates after) | Fastest (served instantly from CDN, no per-request server work) |
| SEO | Historically poor (crawlers may not execute JS, though modern Googlebot does render JS with caveats) | Good — search engines get full content immediately | Excellent |
| Server load | Low (static file serving + API calls) | Higher (server does rendering work per request) | Very low (pre-built, cached) |
| Data freshness | Always fresh (fetched client-side per visit) | Fresh (rendered per request) | Can go stale until next build/revalidation |
| Best for | Highly interactive, behind-login apps (dashboards) where SEO doesn't matter | Content that needs to be fresh AND SEO-friendly (e.g., e-commerce product pages with live pricing/stock) | Content that changes infrequently (blogs, marketing pages, docs) |

**Hydration:** After SSR sends fully-rendered HTML, React "hydrates" it on the client — attaching event listeners and reconciling its Virtual DOM with the existing server-rendered markup **without re-creating the DOM from scratch** — the browser shows content immediately (fast perceived load), while the page becomes interactive once JS loads and hydration completes (there's a brief window where content is visible but not yet interactive).

**Next.js (the dominant React SSR/SSG framework)** provides file-based routing, and per-page rendering strategy choice:
- `getServerSideProps` (or the App Router's default Server Components) — SSR, fetched fresh per request.
- `getStaticProps` with optional `revalidate` (Incremental Static Regeneration) — SSG with periodic background regeneration, combining static speed with eventual freshness.
- App Router's **React Server Components (RSC)** — components that render **only on the server**, sending zero JS to the client for that component, and can directly access backend resources (DB, filesystem) without an API layer — a significant architectural shift from purely client-rendered React, reducing client bundle size substantially for content that doesn't need client-side interactivity.

**Follow-up 🎯:** *"What's the difference between SSR and React Server Components — isn't SSR already 'server-rendered'?"* → Traditional SSR still ships the **full component code** to the client as JS (for hydration — the client needs the component logic to "take over" interactivity) — it renders HTML on the server but the JS bundle cost is still paid. **Server Components** (Next.js App Router / React 18+) go further: certain components render on the server **and their code is never sent to the client at all** — no hydration needed for them, zero client JS bundle cost — reserved specifically for non-interactive parts of the UI, while genuinely interactive parts remain Client Components (marked `'use client'`) that do hydrate normally.

---

## 13. Design Patterns

### Q13.1 — HOC vs Render Props vs Custom Hooks for sharing logic — with code for each, and why Hooks won out.

**Answer:**

**Higher-Order Component (HOC):** A function that takes a component and returns a new, enhanced component.
```jsx
function withAuth(Component) {
  return function AuthenticatedComponent(props) {
    const { user } = useAuth();
    if (!user) return <Login />;
    return <Component {...props} user={user} />;
  };
}
const ProtectedDashboard = withAuth(Dashboard);
```
Downsides: prop naming collisions if multiple HOCs inject similarly-named props; "wrapper hell" in React DevTools (deeply nested wrapper components); unclear where a given prop actually originates from when reading the consuming component's code.

**Render Props:** A component takes a function as a prop (or `children` as a function) that it calls with some data/state, letting the consumer decide what to render.
```jsx
function DataFetcher({ url, render }) {
  const [data, setData] = useState(null);
  useEffect(() => { fetchData(url).then(setData); }, [url]);
  return render(data);
}
// usage: <DataFetcher url="/api/users" render={data => <UserList users={data} />} />
```
Downsides: can lead to deeply nested JSX ("callback hell" visually in the render tree) when combining multiple render-props components.

**Custom Hooks (the modern preferred approach):** Extract stateful logic into a plain function starting with `use`, following the Rules of Hooks — no extra component wrapping, no nested render trees, just a function call.
```jsx
function useAuth() {
  const [user, setUser] = useState(null);
  useEffect(() => { /* subscribe to auth state */ }, []);
  return { user, login, logout };
}
function Dashboard() {
  const { user } = useAuth(); // clean, direct, composable with other hooks freely
  if (!user) return <Login />;
  return <div>Welcome {user.name}</div>;
}
```

**Why Hooks won:** No extra component tree nesting (cleaner DevTools, no wrapper hell), no naming-collision risk (destructure exactly what you need under any name), trivially composable (call multiple custom hooks in one component, combine their results directly in plain JS), and much easier to read (the logic a component depends on is explicit at the top of the function, not hidden in a wrapping HOC or a render-prop's parent).

**Follow-up:** *"Are HOCs and Render Props now considered deprecated/wrong to use?"* → Not deprecated — they still work and some libraries/older codebases use them (e.g., some React Router v5 patterns, certain older library integrations) — but for **new code**, custom Hooks are essentially always the preferred pattern for logic reuse; HOCs still see legitimate use for certain cross-cutting concerns applied declaratively at the JSX level (e.g., some higher-level component-composition libraries), but that's now a minority use case.

---

## 14. React 18/19 Features

### Q14.1 — Concurrent Rendering, `useTransition`, `useDeferredValue`, and Suspense for Data Fetching.

**Answer:** React 18 introduced **Concurrent Rendering** as an opt-in capability (enabled by using `createRoot`) — the ability for React to prepare multiple versions of the UI at the same time, **interrupt** an in-progress render if a higher-priority update comes in (e.g., user typing), and discard/resume work as needed — built on the Fiber architecture's incremental rendering foundation (Q1.1).

**`useTransition` — mark updates as non-urgent ("low priority"):**
```jsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  const [results, setResults] = useState([]);

  function handleChange(e) {
    setQuery(e.target.value); // URGENT — keep the input responsive, updates immediately
    startTransition(() => {
      setResults(filterHugeList(e.target.value)); // NON-URGENT — can be interrupted/delayed if needed
    });
  }
  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <ResultsList results={results} />
    </>
  );
}
```
Without `useTransition`, typing into the search box would feel laggy if `filterHugeList` is expensive — every keystroke would force React to synchronously finish the expensive re-render of `ResultsList` before the input's own visual update is even committed. With `useTransition`, React keeps the input snappy (urgent update processed immediately) while the expensive list re-render happens at lower priority, and can even be **interrupted** by the next keystroke rather than finishing a now-stale computation first.

**`useDeferredValue` — a related, sometimes simpler alternative:** defers re-rendering a specific **value** (rather than wrapping a whole state-setting function in a transition) until higher-priority updates have finished:
```jsx
function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query); // "lags behind" query during rapid typing, catches up when idle
  const results = useMemo(() => filterHugeList(deferredQuery), [deferredQuery]);
  return <ResultsList results={results} />;
}
```

**Suspense for Data Fetching (paired with frameworks like Next.js App Router / Relay, or libraries with Suspense-compatible data fetching):** Lets a component "suspend" rendering while waiting for data, showing a fallback UI declaratively, similar to how `React.lazy` + Suspense works for code — but for **data**, not just code:
```jsx
<Suspense fallback={<Spinner />}>
  <UserProfile userId={id} /> {/* can "throw" a promise while its data is loading; Suspense catches it */}
</Suspense>
```

**Best Practices:**
- Use `useTransition`/`startTransition` for state updates that trigger expensive re-renders but aren't the direct, immediate visual feedback the user is interacting with (e.g., filtering/searching a large dataset based on typed input).
- Don't wrap genuinely urgent updates (the input's own displayed value) in a transition — only the expensive downstream consequence of that update.

**Follow-up 🎯:** *"Does concurrent rendering mean React runs your components on multiple threads/in parallel?"* → No — JavaScript (in the browser main thread, without Web Workers) is still single-threaded; "concurrent" here means React can **interleave** and **interrupt** rendering work (pause a low-priority render mid-way, handle an urgent update, then resume or discard the paused work) — it's cooperative time-slicing of work on the single main thread, not true multi-threaded parallelism. This is a very common and important misconception to correct in an interview.

---

## 15. Rapid-Fire Tricky Q&A

**Q: Why does calling `setState` inside a `useEffect` with no dependency array cause an infinite loop, and what if the dependency array includes the very state being set?**
A: No dependency array → effect runs after every render; if it calls a state setter, that triggers another render, which runs the effect again → infinite loop. If the dependency array includes the state being set (`useEffect(() => setCount(count + 1), [count])`), same problem — every state update makes the dependency change, re-triggering the effect, updating state again, forever. Fix: only set state in an effect when responding to something genuinely external (an event, a prop change, data arriving) — not deriving state from state changes triggered by the same effect.

**Q: What's the difference between `React.Fragment` (`<>...</>`) and returning an array from a component?**
A: `React.Fragment` lets you group multiple elements without adding an extra DOM node (useful since a component must return a single root element/fragment) and supports a `key` prop when needed in a list context (`<React.Fragment key={id}>`, the shorthand `<>` syntax doesn't support keys). Returning a plain array requires each element to have an explicit `key` prop directly, and is less common/idiomatic than Fragments in modern code.

**Q: Why shouldn't you use array `.map()` with inline anonymous arrow functions as event handlers for every list item, from a performance standpoint — and does it actually matter in practice?**
A: A new function reference is created on every render for each item (`onClick={() => handleClick(item.id)}`), which would defeat `React.memo` on the child if the child is memoized (Q3.3) — but if the child ISN'T memoized anyway (common case, especially for simple list items), this has essentially zero measurable performance impact, since the child re-renders regardless of the callback's referential identity. This is a case where "textbook advice" (memoize everything) needs to be weighed against actual profiling — premature optimization here is common and often pointless.

**Q: Can two components share the same key in a list rendered by React?**
A: No — keys must be unique **among siblings** (not globally unique across the whole app) — React will log a warning and behavior becomes undefined/unpredictable (typically only the last element with a duplicate key survives in that position) if duplicates exist within the same list.

**Q: Why does directly mutating an array/object in state (e.g., `state.items.push(x)`) sometimes appear to "work" visually but is still wrong?**
A: If the mutation happens to be followed by an unrelated state update that triggers a re-render anyway (or you're in React's development StrictMode double-render, which can mask certain issues), the mutated data might coincidentally show up correctly once — but you've broken React's ability to detect the change via reference comparison (`Object.is`), which will cause subtle bugs later: `React.memo` won't detect the "change" (Q3.1), time-travel debugging tools break, and concurrent rendering features that rely on comparing snapshots of state across paused/resumed work can behave unpredictably. Always treat state as immutable — create new objects/arrays on every update.

**Q: What's the actual difference between `useMemo(() => fn, deps)` and `useCallback(fn, deps)`?**
A: They're equivalent for the case of memoizing a function — `useCallback(fn, deps)` is literally implemented as sugar for `useMemo(() => fn, deps)` internally. The distinction is purely about intent/readability: use `useCallback` when the thing you want to memoize IS a function itself; use `useMemo` when memoizing a computed value (which may or may not be a function).

---

## Quick Self-Check Before Your Interview

- [ ] Can you explain reconciliation and Fiber's incremental rendering without notes?
- [ ] Can you explain the stale closure bug in `useEffect` and demonstrate 2 different fixes?
- [ ] Can you explain exactly why array index as `key` breaks for reorderable/filterable lists?
- [ ] Can you explain when `useMemo`/`useCallback`/`React.memo` actually help vs when they're pure overhead?
- [ ] Can you explain the Context re-render pitfall and at least two ways to fix it?
- [ ] Can you explain what Error Boundaries do NOT catch, and why?
- [ ] Can you explain the real difference between concurrent rendering and true multi-threading?
- [ ] Do you have a real project story about a re-render/performance bug you diagnosed and fixed?
