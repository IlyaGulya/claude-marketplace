---
name: solidjs-async
description: >-
  Handles asynchronous data fetching in SolidJS using createResource with Suspense
  integration. Use when fetching data from APIs, managing loading/error states, or
  implementing optimistic updates in SolidJS.
---

# SolidJS Async Patterns

## createResource

The primary primitive for async data fetching. Integrates with Suspense and ErrorBoundary.

### Without source (fetch once)

```tsx
import { createResource } from "solid-js";

const [data, { mutate, refetch }] = createResource(async () => {
  const res = await fetch("/api/data");
  return res.json();
});
```

### With source signal (auto-refetch)

```tsx
const [userId, setUserId] = createSignal(1);

const [user] = createResource(userId, async (id) => {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
});

// Automatically refetches when userId changes
setUserId(2);
```

**Source signal behavior:**
- When source returns `false`, `null`, or `undefined` — fetcher is NOT called
- When source changes to a truthy value — fetcher is called with that value
- This enables conditional fetching:

```tsx
const [searchQuery, setSearchQuery] = createSignal("");

// Only fetches when query is non-empty
const [results] = createResource(
  () => searchQuery() || false, // falsy = skip fetch
  async (query) => {
    const res = await fetch(`/api/search?q=${query}`);
    return res.json();
  }
);
```

## Resource States

A resource has 5 states:

| State | `resource()` | `.loading` | `.error` | `.latest` | Description |
|---|---|---|---|---|---|
| `unresolved` | `undefined` | `false` | `undefined` | `undefined` | Initial, fetcher not yet called |
| `pending` | `undefined` | `true` | `undefined` | `undefined` | Fetching in progress |
| `ready` | `T` | `false` | `undefined` | `T` | Successfully fetched |
| `refreshing` | `T` | `true` | `undefined` | `T` | Re-fetching, previous value available |
| `errored` | `undefined` | `false` | `any` | `undefined` | Fetch failed |

### Accessing resource data

```tsx
// Current value (undefined while loading)
const value = data();

// Loading state
const isLoading = data.loading;

// Error state
const error = data.error;

// Latest successful value (persists during refresh)
const latest = data.latest;

// Exact state
const state = data.state; // "unresolved" | "pending" | "ready" | "refreshing" | "errored"
```

### resource.latest

`latest` keeps the previous successful value while refreshing. Useful for showing stale data during reload:

```tsx
function UserList() {
  const [users, { refetch }] = createResource(fetchUsers);

  return (
    <div>
      <button onClick={refetch} disabled={users.loading}>
        {users.loading ? "Refreshing..." : "Refresh"}
      </button>
      {/* Show previous data while refreshing instead of spinner */}
      <For each={users.latest ?? []}>
        {(user) => <div>{user.name}</div>}
      </For>
    </div>
  );
}
```

## mutate and refetch

```tsx
const [todos, { mutate, refetch }] = createResource(fetchTodos);

// Optimistic update — immediately update local state without fetching
mutate(prev => [...(prev ?? []), newTodo]);

// Manual refresh — re-runs the fetcher
await refetch();

// refetch with info — passed to fetcher's info.refetching
await refetch("forced");
// In fetcher: async (source, { refetching }) => { if (refetching === "forced") ... }
```

## Suspense Integration

`createResource` automatically triggers the nearest `<Suspense>` boundary when read.

```tsx
import { createResource, Suspense } from "solid-js";

function UserProfile() {
  const [user] = createResource(fetchCurrentUser);

  return (
    <Suspense fallback={<LoadingSpinner />}>
      <div>
        <h1>{user()?.name}</h1>
        <p>{user()?.email}</p>
      </div>
    </Suspense>
  );
}
```

**Best practice:** Use optional chaining (`user()?.name`) inside Suspense — the resource starts as `undefined` and the DOM is pre-created before the resource resolves.

### Nested Suspense

Each resource triggers only its closest Suspense boundary:

```tsx
<Suspense fallback={<PageLoader />}>
  <h1>{pageTitle()}</h1>
  <Suspense fallback={<SidebarLoader />}>
    <Sidebar data={sidebarData()} />
  </Suspense>
</Suspense>
```

## Error Handling

Use `<ErrorBoundary>` to catch resource errors:

```tsx
import { ErrorBoundary, Suspense } from "solid-js";

<ErrorBoundary
  fallback={(err, reset) => (
    <div>
      <p>Failed to load: {err.message}</p>
      <button onClick={reset}>Retry</button>
    </div>
  )}
>
  <Suspense fallback={<Loading />}>
    <DataComponent />
  </Suspense>
</ErrorBoundary>
```

You can also check `resource.error` directly:

```tsx
const [data] = createResource(fetcher);

return (
  <Show when={!data.error} fallback={<div>Error: {data.error?.message}</div>}>
    <div>{data()?.value}</div>
  </Show>
);
```

## Options

```tsx
const [data] = createResource(fetcher, {
  // Initial value — resource starts in "ready" state, never undefined
  initialValue: [],

  // SSR: where to load data from during hydration
  ssrLoadFrom: "server", // default — use server-fetched value
  // ssrLoadFrom: "initial", // re-fetch on client after hydration

  // Defer streaming until resource resolves
  deferStream: false,

  // Custom storage (e.g., for persistence)
  storage: createSignal,
});
```

### initialValue

When provided, `resource()` is never `undefined` and the type reflects this:

```tsx
const [todos] = createResource(fetchTodos, { initialValue: [] });

// todos() is Todo[], not Todo[] | undefined
return <For each={todos()}>{(todo) => <div>{todo.title}</div>}</For>;
```

## createResource vs createSignal + onMount

```tsx
// createResource — integrated with Suspense, loading/error states built in
const [data] = createResource(fetchData);

// Manual approach — no Suspense integration, manual loading/error tracking
const [data, setData] = createSignal<Data>();
const [loading, setLoading] = createSignal(false);
const [error, setError] = createSignal<Error>();

onMount(async () => {
  setLoading(true);
  try {
    setData(await fetchData());
  } catch (e) {
    setError(e as Error);
  } finally {
    setLoading(false);
  }
});
```

**Use `createResource` when:**
- You want Suspense integration
- You need loading/error state tracking
- You want auto-refetch on source signal changes
- You want optimistic updates via `mutate`

**Use manual fetch when:**
- You explicitly don't want Suspense to trigger
- You need full control over the fetch lifecycle
- The fetch is a one-time side effect, not data loading
