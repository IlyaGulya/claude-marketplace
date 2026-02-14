# SolidJS Props Reference

## mergeProps

Reactively merges multiple props objects. Properties resolve in reverse order (last source wins).

```ts
import { mergeProps } from "solid-js";

function mergeProps(...sources: any): any;
```

### Patterns

```tsx
// Default props
function Button(props: { color?: string; disabled?: boolean }) {
  const merged = mergeProps({ color: "blue", disabled: false }, props);
  return <button style={{ color: merged.color }} disabled={merged.disabled} />;
}

// Clone props (creates new reactive proxy)
const cloned = mergeProps(props);

// Merge multiple sources
const final = mergeProps(defaults, userProps, overrides);
```

## splitProps

Splits a reactive props object into groups by key. Returns reactive proxy objects.

```ts
import { splitProps } from "solid-js";

function splitProps<T>(props: T, ...keys: Array<(keyof T)[]>): [...parts: Partial<T>];
```

### Patterns

```tsx
// Two-way split: local props + rest to forward
function Input(props: { label: string } & JSX.InputHTMLAttributes<HTMLInputElement>) {
  const [local, inputProps] = splitProps(props, ["label"]);
  return (
    <label>
      {local.label}
      <input {...inputProps} />
    </label>
  );
}

// Multi-way split
function Widget(props: { title: string; class?: string; onClick?: () => void; data: any[] }) {
  const [content, style, events, rest] = splitProps(
    props,
    ["title", "data"],
    ["class"],
    ["onClick"]
  );
  // content = { title, data }
  // style = { class }
  // events = { onClick }
  // rest = {} (leftover)
}
```

## children()

Resolves and normalizes a component's `children` prop into a stable accessor.

```ts
import { children } from "solid-js";

function children(fn: () => JSX.Element): ChildrenReturn;

type ChildrenReturn = Accessor<ResolvedChildren> & {
  toArray: () => ResolvedChildren[];
};
```

### Patterns

```tsx
// Basic usage — resolve children for programmatic access
function Wrapper(props: { children: JSX.Element }) {
  const resolved = children(() => props.children);
  return <div>{resolved()}</div>;
}

// Count and iterate children
function List(props: { children: JSX.Element }) {
  const resolved = children(() => props.children);
  return (
    <ul>
      {resolved.toArray().map((child) => <li>{child}</li>)}
    </ul>
  );
}
```

**When to use `children()`:**
- You need to count, iterate, filter, or transform children
- You're building a layout component that reads child metadata

**When NOT to use `children()`:**
- Simple pass-through: just use `{props.children}` directly
- Using `<Show>`, `<For>`, or other control flow around children

## TypeScript Module Augmentation

### Custom Events

```ts
declare module "solid-js" {
  namespace JSX {
    interface CustomEvents {
      myEvent: CustomEvent<{ detail: string }>;
    }
  }
}

// Usage: <div on:myEvent={(e) => console.log(e.detail)} />
```

### Custom Directives

```ts
declare module "solid-js" {
  namespace JSX {
    interface DirectiveFunctions {
      myDirective: typeof myDirectiveFunction;
    }
  }
}

// Usage: <div use:myDirective={value} />
```
