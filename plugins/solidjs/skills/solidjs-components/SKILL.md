---
name: solidjs-components
description: >-
  Creates SolidJS components with correct prop handling, TypeScript types, and
  lifecycle management. Use when building components, passing props, handling children,
  or managing component lifecycle with onMount/onCleanup.
---

# SolidJS Components

## Component Structure

A SolidJS component is a function that runs **once** and returns JSX. It sets up the reactive graph, then the returned JSX handles updates.

```tsx
import { createSignal } from "solid-js";

function Counter() {
  const [count, setCount] = createSignal(0);
  // Everything here runs once
  return <button onClick={() => setCount(c => c + 1)}>{count()}</button>;
}
```

## Props — Never Destructure

Props are a reactive proxy object. Destructuring reads values eagerly, breaking reactivity.

```tsx
// WRONG
function Greeting({ name }: { name: string }) {
  return <h1>Hello {name}</h1>; // name is captured once, never updates
}

// CORRECT
function Greeting(props: { name: string }) {
  return <h1>Hello {props.name}</h1>; // props.name tracked in JSX
}
```

Always access props with `props.x` inside JSX or other tracking scopes.

## Default Props with mergeProps

Use `mergeProps` to provide defaults without losing reactivity.

```tsx
import { mergeProps } from "solid-js";

function Button(props: { color?: string; size?: number }) {
  const merged = mergeProps({ color: "blue", size: 16 }, props);
  return <button style={{ color: merged.color, "font-size": `${merged.size}px` }}>Click</button>;
}
```

`mergeProps` resolves properties in reverse order via a proxy — if `props.color` is provided, it overrides the default.

## Splitting Props with splitProps

Use `splitProps` to consume some props and forward the rest. This is the reactive-safe alternative to destructuring.

```tsx
import { splitProps } from "solid-js";
import type { JSX } from "solid-js";

function Button(props: { variant?: "primary" | "secondary" } & JSX.ButtonHTMLAttributes<HTMLButtonElement>) {
  const [local, others] = splitProps(props, ["variant"]);

  return (
    <button
      class={local.variant === "primary" ? "btn-primary" : "btn-secondary"}
      {...others}
    />
  );
}
```

Split into multiple groups:

```tsx
const [styleProps, eventProps, rest] = splitProps(props, ["class", "style"], ["onClick", "onInput"]);
```

## children() Helper

When you need to interact with resolved children (e.g., count them, iterate, modify), use the `children()` helper.

```tsx
import { children } from "solid-js";

function Wrapper(props: { children: JSX.Element }) {
  const resolved = children(() => props.children);

  return (
    <div>
      <span>Child count: {resolved.toArray().length}</span>
      {resolved()}
    </div>
  );
}
```

**Important:** Only use `children()` when you need to programmatically interact with the resolved children. For simple pass-through, just use `props.children` directly.

## TypeScript Component Types

```tsx
import type { Component, ParentComponent, VoidComponent, FlowComponent, ParentProps } from "solid-js";
import type { JSX } from "solid-js";

// Component — generic, explicit children if needed
const MyComponent: Component<{ title: string }> = (props) => {
  return <h1>{props.title}</h1>;
};

// VoidComponent — no children allowed
const Icon: VoidComponent<{ name: string }> = (props) => {
  return <svg><use href={`#${props.name}`} /></svg>;
};

// ParentComponent — includes optional children: JSX.Element
const Card: ParentComponent<{ title: string }> = (props) => {
  return <div><h2>{props.title}</h2>{props.children}</div>;
};

// FlowComponent — includes required children: JSX.Element
const Layout: FlowComponent<{ sidebar: JSX.Element }> = (props) => {
  return <div>{props.sidebar}<main>{props.children}</main></div>;
};

// Manual ParentProps helper
function Container(props: ParentProps<{ class?: string }>) {
  return <div class={props.class}>{props.children}</div>;
}
```

## TypeScript Event Types

```tsx
// Input event
function handleInput(e: InputEvent & { currentTarget: HTMLInputElement }) {
  console.log(e.currentTarget.value);
}

// In JSX — types are inferred
<input onInput={(e) => console.log(e.currentTarget.value)} />

// Typing custom events with on: syntax requires module augmentation
declare module "solid-js" {
  namespace JSX {
    interface CustomEvents {
      myCustomEvent: CustomEvent<string>;
    }
  }
}
```

## Lifecycle: onMount

Runs once after the component's DOM elements are created and refs are set, but before browser paint.

```tsx
import { onMount } from "solid-js";

function Chart(props: { data: number[] }) {
  let canvas!: HTMLCanvasElement;

  onMount(() => {
    // refs are available here
    const ctx = canvas.getContext("2d")!;
    drawChart(ctx, props.data);
  });

  return <canvas ref={canvas} />;
}
```

`onMount` is equivalent to a non-tracking `createEffect` — it runs once and does not re-run on signal changes.

## Lifecycle: onCleanup

Registers cleanup that runs when the owning scope is disposed (component unmount, effect re-run, etc).

```tsx
import { createSignal, createEffect, onCleanup } from "solid-js";

function MouseTracker() {
  const [pos, setPos] = createSignal({ x: 0, y: 0 });

  const handler = (e: MouseEvent) => setPos({ x: e.clientX, y: e.clientY });
  document.addEventListener("mousemove", handler);
  onCleanup(() => document.removeEventListener("mousemove", handler));

  return <div>Mouse: {pos().x}, {pos().y}</div>;
}
```

`onCleanup` inside an effect re-runs its cleanup before each re-execution:

```tsx
createEffect(() => {
  const id = setInterval(() => tick(), delay());
  onCleanup(() => clearInterval(id)); // cleans up on each re-run
});
```

## Refs

Refs are assigned at render time, before DOM is connected. Two flavors:

```tsx
// Variable assignment — use with onMount to ensure DOM is ready
function MyComponent() {
  let divRef!: HTMLDivElement; // definite assignment assertion

  onMount(() => {
    console.log(divRef.clientWidth); // DOM is ready
  });

  return <div ref={divRef}>Hello</div>;
}

// Callback form — called before DOM connection
<div ref={(el) => {
  // el is the DOM element
  // Can create effects here, but el may not be in document yet
}} />
```

Forward refs to children:

```tsx
function MyComp(props: { ref?: HTMLDivElement | ((el: HTMLDivElement) => void) }) {
  return <div ref={props.ref}>Content</div>;
}

function Parent() {
  let ref!: HTMLDivElement;
  onMount(() => console.log(ref.clientWidth));
  return <MyComp ref={ref} />;
}
```

## Custom Directives (use:)

Directives attach reusable behavior to DOM elements.

```tsx
import type { Accessor } from "solid-js";

function clickOutside(el: HTMLElement, accessor: Accessor<() => void>) {
  const handler = (e: MouseEvent) => {
    if (!el.contains(e.target as Node)) accessor()();
  };
  document.addEventListener("click", handler);
  onCleanup(() => document.removeEventListener("click", handler));
}

// Usage
<div use:clickOutside={() => setOpen(false)}>Dropdown</div>

// TypeScript: augment JSX namespace
declare module "solid-js" {
  namespace JSX {
    interface DirectiveFunctions {
      clickOutside: typeof clickOutside;
    }
  }
}
```

Directives only work on native HTML elements, not on components.

## Event Handlers

Solid provides two event binding mechanisms:

```tsx
// Delegated events (onClick, onInput, etc.) — attached to document, efficient
<button onClick={(e) => handleClick(e)}>Click</button>

// Native events (on:event) — attached directly to element
<div on:scroll={(e) => handleScroll(e)}>Content</div>

// Binding syntax — avoids closures, first array item is handler, second is data
const handler = (data: string, e: MouseEvent) => console.log(data, e);
<button onClick={[handler, "hello"]}>Click</button>
```

**Delegated events** are not case-sensitive (`onClick` = `onclick`). They use event delegation on the document.

**Native events** (`on:event`) are case-sensitive, attached directly. Use when:
- You need `stopPropagation` to work correctly
- The event is uncommon/custom
- You need exact event timing control

See PROPS-REFERENCE.md for detailed mergeProps, splitProps, and children() API references.
