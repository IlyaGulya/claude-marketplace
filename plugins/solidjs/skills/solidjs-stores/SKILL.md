---
name: solidjs-stores
description: >-
  Manages complex nested state in SolidJS using stores with fine-grained reactivity.
  Use when working with objects, arrays, nested data structures, or when integrating
  API responses into reactive state.
---

# SolidJS Stores

## When to Use Stores vs Signals

| Use Case | Primitive |
|---|---|
| Simple values (string, number, boolean) | `createSignal` |
| Objects/arrays where you update the whole value | `createSignal` |
| Nested objects/arrays with fine-grained updates | `createStore` |
| Complex state trees (app state, form state) | `createStore` |

## createStore

Creates a reactive store with fine-grained tracking on nested properties.

```tsx
import { createStore } from "solid-js/store";

const [store, setStore] = createStore({
  user: { name: "John", age: 30 },
  todos: [
    { id: 1, text: "Learn Solid", done: false },
    { id: 2, text: "Build app", done: false },
  ],
});
```

**How it works:**
- Store values are wrapped in proxies
- Signals are created **lazily** — only when a property is accessed in a tracking scope
- Reading `store.user.name` in JSX creates a subscription to just that property
- Updating `store.user.name` only triggers updates for subscribers of that exact property

```tsx
// Only the specific <span> updates, not the entire component
return (
  <div>
    <span>{store.user.name}</span>  {/* tracked */}
    <span>{store.user.age}</span>   {/* independently tracked */}
  </div>
);
```

## Setter Patterns

### Direct value (shallow merge for objects)

```tsx
// Shallow merge at top level
setStore({ user: { name: "Jane", age: 25 } });

// Shallow merge — keeps unmentioned properties
setStore({ user: { name: "Jane" } }); // age is preserved
```

### Function form

```tsx
setStore(prev => ({ ...prev, newProp: "value" }));
```

### Path Syntax

Navigate to nested properties with string keys:

```tsx
// Set a nested property
setStore("user", "name", "Jane");

// Set deep nested property
setStore("user", "address", "city", "New York");

// Function form at any level
setStore("user", "age", prev => prev + 1);
```

### Array operations with path syntax

```tsx
// Update item at index
setStore("todos", 0, "done", true);

// Append to array
setStore("todos", store.todos.length, {
  id: 3, text: "Deploy", done: false,
});

// Update multiple indices
setStore("todos", [0, 2], "done", true);

// Range of indices
setStore("todos", { from: 0, to: 2 }, "done", false);

// Filter-based update
setStore("todos", todo => todo.done, "text", prev => `[DONE] ${prev}`);

// Dynamic update with function
setStore("todos", 0, "done", done => !done);
```

## produce — Immer-Style Mutations

For complex updates, `produce` lets you mutate a draft object.

```tsx
import { produce } from "solid-js/store";

setStore(
  produce((state) => {
    state.user.name = "Jane";
    state.todos.push({ id: 3, text: "Deploy", done: false });
    state.todos[0].done = true;
  })
);
```

Combine with path syntax:

```tsx
setStore(
  "todos",
  0,
  produce((todo) => {
    todo.done = true;
    todo.text = `[DONE] ${todo.text}`;
  })
);
```

**Limitations:** `produce` only works with objects and arrays, not Sets or Maps.

**When to use `produce` vs path syntax:**
- Path syntax: targeted single-property updates, simple operations
- `produce`: multiple changes to the same subtree, array mutations (push, splice)

## reconcile — Efficient Data Diffing

Diffs new data against existing store data, applying only the changes. Essential for API responses.

```tsx
import { reconcile } from "solid-js/store";

// Replace store data with API response, only updating what changed
const newData = await fetchTodos();
setStore("todos", reconcile(newData));
```

### Options

```tsx
// key — property used to match items (default: "id")
setStore("todos", reconcile(newTodos, { key: "todoId" }));

// merge — push diffing to leaves instead of replacing objects
setStore("todos", reconcile(newTodos, { merge: true }));
```

| Option | Default | Description |
|---|---|---|
| `key` | `"id"` | Property to match items across old/new data |
| `merge` | `false` | When false: referential check, replace if different. When true: deep diff to leaves. |

**Common pattern — resource + reconcile:**

```tsx
const [todos] = createResource(fetchTodos);

createEffect(() => {
  const data = todos();
  if (data) setStore("todos", reconcile(data));
});
```

## unwrap — Extract Plain Objects

Converts a store proxy back to a plain JavaScript object. Useful for serialization or passing to non-Solid code.

```tsx
import { unwrap } from "solid-js/store";

const rawData = unwrap(store);
console.log(JSON.stringify(rawData));
```

## createMutable

Creates a mutable reactive proxy. State can be mutated directly.

```tsx
import { createMutable } from "solid-js/store";

const state = createMutable({
  count: 0,
  user: { name: "John" },
});

// Direct mutation — reactive
state.count++;
state.user.name = "Jane";
```

**When to use:** Rarely. Primarily for interop with libraries that expect mutable objects (e.g., MobX-style patterns). Prefer `createStore` for most cases — the explicit setter makes state changes easier to track and debug.

## Store Getters

Stores support JavaScript getters for derived values:

```tsx
const [store, setStore] = createStore({
  firstName: "John",
  lastName: "Smith",
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  },
});

// store.fullName reactively updates when firstName or lastName change
```

## Nested Stores

You can create a store from a nested property of an existing store:

```tsx
const [store, setStore] = createStore({
  users: [{ name: "John" }, { name: "Jane" }],
});

// Create a derived store for the nested array
const [users, setUsers] = createStore(store.users);

// Changes through setUsers update store.users
setUsers(users.length, { name: "Bob" });
```
