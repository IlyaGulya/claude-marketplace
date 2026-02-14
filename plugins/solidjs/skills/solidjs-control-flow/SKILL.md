---
name: solidjs-control-flow
description: >-
  Uses SolidJS control flow components for conditional rendering, list rendering,
  error handling, and suspense. Use when rendering lists, showing/hiding content
  conditionally, handling errors, or working with async boundaries in SolidJS.
---

# SolidJS Control Flow

## Why Control Flow Components

Since SolidJS components run once, JavaScript control flow (if/else, ternary, .map) in the component body executes once and produces static results. Use Solid's control flow components inside JSX to create reactive conditional and iterative rendering.

```tsx
// WRONG — if/else in component body runs once
function MyComponent(props: { show: boolean }) {
  if (props.show) return <div>Visible</div>;  // fixed forever
  return <div>Hidden</div>;
}

// CORRECT — use <Show> in JSX
function MyComponent(props: { show: boolean }) {
  return (
    <Show when={props.show} fallback={<div>Hidden</div>}>
      <div>Visible</div>
    </Show>
  );
}
```

**Note:** Ternaries and boolean expressions directly inside JSX expressions ARE reactive because JSX expressions are tracking scopes:

```tsx
// This works fine — JSX expression is a tracking scope
return <div>{props.show ? "Visible" : "Hidden"}</div>;
```

## `<Show>`

Conditional rendering. Renders children when `when` is truthy, otherwise renders `fallback`.

```tsx
import { Show } from "solid-js";

<Show when={user()} fallback={<LoginForm />}>
  <Dashboard />
</Show>
```

### Render function form (type narrowing)

Pass a function as children to receive the truthy `when` value as an accessor. Useful for non-null narrowing.

```tsx
<Show when={user()}>
  {(user) => <div>Welcome, {user().name}</div>}
</Show>
```

The callback argument `user` is an **accessor** (function). Call it with `user()` to get the value.

**Important:** The render function is internally wrapped with `untrack`. Signal reads directly in the function body are NOT tracked. Wrap in JSX elements for tracking:

```tsx
// NOT tracked — count() read directly in callback scope
<Show when={visible()}>{() => count()}</Show>

// Tracked — count() read inside JSX element
<Show when={visible()}>{() => <span>{count()}</span>}</Show>
```

### Keyed rendering

When `keyed` is set, changes to the `when` reference recreate children entirely:

```tsx
<Show when={user()} keyed>
  <UserCard user={user()} />
</Show>
```

## `<For>`

Referentially keyed list rendering. Each item is tracked by reference — efficient for arrays of objects.

```tsx
import { For } from "solid-js";

<For each={todos()} fallback={<div>No todos</div>}>
  {(todo, index) => (
    <div>
      #{index()} {todo.title}
    </div>
  )}
</For>
```

**Key points:**
- `item` is the **value** (not a signal)
- `index` is a **signal** (call with `index()`)
- Items are keyed by reference — reordering moves DOM nodes, doesn't recreate
- The callback runs once per item; only `index()` updates when position changes

## `<Index>`

Index-keyed list rendering. Each position is fixed — efficient for arrays of primitives.

```tsx
import { Index } from "solid-js";

<Index each={names()}>
  {(name, index) => (
    <div>
      #{index} {name()}
    </div>
  )}
</Index>
```

**Key points:**
- `item` is a **signal** (call with `name()`)
- `index` is a **number** (not a signal, fixed)
- Positions are fixed; when the value at an index changes, only that callback's signal updates

## `<For>` vs `<Index>` Decision Guide

| Scenario | Use |
|---|---|
| Array of objects with identity | `<For>` |
| Array of primitives (strings, numbers) | `<Index>` |
| Items may be reordered | `<For>` |
| Items only change in place | `<Index>` |
| Need stable DOM per item (animations) | `<For>` |

## `<Switch>` / `<Match>`

Multi-way conditional rendering. Like if/else-if chains.

```tsx
import { Switch, Match } from "solid-js";

<Switch fallback={<div>Not found</div>}>
  <Match when={route() === "home"}>
    <Home />
  </Match>
  <Match when={route() === "about"}>
    <About />
  </Match>
  <Match when={route() === "settings"}>
    <Settings />
  </Match>
</Switch>
```

`<Match>` also supports render function children for type narrowing:

```tsx
<Switch>
  <Match when={data()}>
    {(data) => <DataView data={data()} />}
  </Match>
</Switch>
```

## `<ErrorBoundary>`

Catches errors in child rendering, effects, memos, and resources. Does NOT catch errors in event handlers or setTimeout callbacks.

```tsx
import { ErrorBoundary } from "solid-js";

<ErrorBoundary
  fallback={(error, reset) => (
    <div>
      <p>Error: {error.message}</p>
      <button onClick={reset}>Try Again</button>
    </div>
  )}
>
  <RiskyComponent />
</ErrorBoundary>
```

**Catches:**
- Errors during JSX rendering
- Errors in `createEffect`, `createMemo`
- Errors in `createResource` (async)

**Does NOT catch:**
- Errors in event handlers
- Errors in `setTimeout` / `setInterval` callbacks

## `<Suspense>`

Shows a fallback while async resources inside are loading. Non-blocking — both branches exist simultaneously.

```tsx
import { Suspense } from "solid-js";

<Suspense fallback={<LoadingSpinner />}>
  <UserProfile />  {/* reads a createResource inside */}
</Suspense>
```

**Key behavior:**
- Triggered when a `createResource` is read inside the boundary
- Children DOM nodes are created immediately but not attached until resources resolve
- `onMount` and `createEffect` inside only run after resources resolve
- Nested `<Suspense>` boundaries scope which resources trigger which fallback

```tsx
// Nested suspense — each resource triggers its closest boundary
<Suspense fallback={<PageLoader />}>
  <h1>{title()}</h1>
  <Suspense fallback={<TableLoader />}>
    <DataTable data={tableData()} />
  </Suspense>
</Suspense>
```

**Suspense vs Show for loading states:**

```tsx
// Suspense — DOM pre-created, shown when ready (better UX, less work after resolve)
<Suspense fallback={<Loading />}>
  <div>{profile()?.name}</div>
</Suspense>

// Show — DOM created only after condition is true (more work after resolve)
<Show when={profile()} fallback={<Loading />}>
  <div>{profile()!.name}</div>
</Show>
```

## `<Portal>`

Renders children into a DOM node outside the component tree. Events still propagate through the component hierarchy.

```tsx
import { Portal } from "solid-js/web";

function Modal(props: { children: JSX.Element }) {
  return (
    <Portal mount={document.getElementById("modal-root")!}>
      <div class="modal-backdrop">
        {props.children}
      </div>
    </Portal>
  );
}
```

**Props:**
- `mount` — target DOM node (default: `document.body`)
- `useShadow` — use Shadow DOM for style isolation
- `isSVG` — required when mounting into SVG elements

Portal is client-only; hydration is disabled.

## `<Dynamic>`

Renders a component or HTML element dynamically based on a prop.

```tsx
import { Dynamic } from "solid-js/web";

// Dynamic component
<Dynamic component={selectedComponent()} data={data()} />

// Dynamic HTML element
<Dynamic component="div" class="container">Content</Dynamic>

// Common pattern: component map
const views = { home: Home, about: About, settings: Settings };
<Dynamic component={views[route()]} />
```

All extra props are forwarded to the rendered component/element.
