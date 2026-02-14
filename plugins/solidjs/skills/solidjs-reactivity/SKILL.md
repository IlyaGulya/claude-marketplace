---
name: solidjs-reactivity
description: >-
  Implements SolidJS reactive patterns using signals, effects, and memos.
  Use when writing or reviewing SolidJS components, state management, or reactive code.
  Covers the core mental model: components run once, signals track in reactive scopes,
  and props must not be destructured.
---

# SolidJS Reactivity

## Core Mental Model

SolidJS is NOT React. The fundamental differences:

1. **Components run once** — A component function executes a single time to set up the reactive graph. It never re-runs.
2. **Reactivity is automatic** — Signal reads inside tracking scopes create subscriptions. No dependency arrays.
3. **Fine-grained updates** — Only the specific DOM nodes or computations that depend on a signal update when it changes. No virtual DOM diffing.

```tsx
// The component function runs ONCE. Only the {count()} expression re-evaluates.
function Counter() {
  const [count, setCount] = createSignal(0);
  console.log("This logs once, not on every update");
  return <button onClick={() => setCount(c => c + 1)}>{count()}</button>;
}
```

## createSignal

Creates a reactive value with a getter (accessor) and setter.

```tsx
import { createSignal } from "solid-js";

const [count, setCount] = createSignal(0);

// Read: call the getter — signals are functions
count(); // 0

// Write: call the setter
setCount(5);
setCount(prev => prev + 1);
```

**Options:**

```tsx
// Custom equality — skip update if values are "equal"
const [data, setData] = createSignal(initialData, {
  equals: (prev, next) => prev.id === next.id,
});

// Always notify — useful for triggering effects on every set
const [trigger, setTrigger] = createSignal(undefined, { equals: false });
```

**Key rule:** Signal getters must be called (with `()`) to read the value. `count` is a function, `count()` is the value.

## Tracking Scopes

Signals are only tracked when read inside a **tracking scope**. These are:

1. **JSX expressions** — `<div>{count()}</div>`
2. **createEffect** — `createEffect(() => console.log(count()))`
3. **createMemo** — `createMemo(() => count() * 2)`

Reading a signal outside these scopes does NOT create a subscription:

```tsx
function MyComponent() {
  const [count, setCount] = createSignal(0);

  // NOT tracked — runs once during component setup
  console.log(count());

  // Tracked — re-runs when count changes
  createEffect(() => {
    console.log(count()); // tracked
  });

  // Tracked — JSX expressions are tracking scopes
  return <div>{count()}</div>;
}
```

## createEffect

Runs side effects when dependencies change. Tracks signals automatically.

```tsx
import { createSignal, createEffect, onCleanup } from "solid-js";

function Timer() {
  const [count, setCount] = createSignal(0);

  createEffect(() => {
    const id = setInterval(() => setCount(c => c + 1), 1000);
    onCleanup(() => clearInterval(id)); // cleanup on re-run or disposal
  });

  return <span>{count()}</span>;
}
```

**Execution timing:**
- Initial run: after synchronous component code, before browser paint
- Subsequent runs: after tracked dependencies change
- Effects run AFTER all pure computations (memos) in the same update cycle
- Effects never run during SSR

**Cleanup:** Use `onCleanup` inside an effect to clean up before re-execution or disposal.

## createMemo

Creates a cached derived value. Only recalculates when dependencies change.

```tsx
import { createSignal, createMemo } from "solid-js";

function FilteredList() {
  const [query, setQuery] = createSignal("");
  const [items] = createSignal(["apple", "banana", "cherry"]);

  // Recalculates only when query() or items() change
  const filtered = createMemo(() =>
    items().filter(item => item.includes(query()))
  );

  return <ul><For each={filtered()}>{item => <li>{item}</li>}</For></ul>;
}
```

**Memo vs derived signal (plain function):**

```tsx
// Derived signal — recalculates every time it's read
const doubled = () => count() * 2;

// Memo — caches result, recalculates only when count changes
const doubled = createMemo(() => count() * 2);
```

Use `createMemo` when:
- The computation is expensive
- The result is read in multiple places (prevents duplicate computation)
- You need to suppress downstream updates when the result hasn't changed

Use a plain derived signal when:
- The computation is cheap
- It's only read in one place

## batch

Groups multiple signal updates so downstream computations run once.

```tsx
import { batch } from "solid-js";

batch(() => {
  setFirstName("John");
  setLastName("Doe");
  setAge(30);
}); // Effects/memos that depend on these signals update once here
```

**Automatic batching:** Updates inside `createEffect`, `onMount`, and store setters are already batched. You rarely need explicit `batch()`.

**Important:** If `batch` callback is async, only updates before the first `await` are batched.

## untrack

Reads a signal without creating a tracking dependency.

```tsx
import { untrack } from "solid-js";

createEffect(() => {
  // count() is tracked, name() is NOT tracked
  console.log(count(), untrack(() => name()));
});
```

Use `untrack` when you need a signal's current value without subscribing to changes.

## Anti-Patterns — See PITFALLS.md

The most common mistakes when writing SolidJS code are documented in detail in the PITFALLS.md companion file. The top issues:

1. Destructuring props (breaks reactivity)
2. Conditional access outside JSX (breaks reactivity)
3. Setting signals inside derived computations
4. Forgetting `()` on signal getters
5. Accessing signals outside tracking scopes
6. Event handler reactivity assumptions
7. `stopPropagation` with delegated events
8. Async breaking tracking context
9. Creating signals/effects outside component scope
10. Assuming effect execution order
