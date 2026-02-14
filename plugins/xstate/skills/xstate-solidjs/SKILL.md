---
name: xstate-solidjs
description: |
  Covers XState v5 integration with SolidJS via @xstate/solid and @xstate/store-solid.
  Use when connecting XState machines or stores to Solid components, using useActor/useActorRef/fromActorRef hooks, or subscribing to store state as Solid signals.
---

# XState v5 + SolidJS Integration

## Packages

| Package | Purpose | Install |
|---------|---------|---------|
| `@xstate/solid` | XState machines in Solid components | `npm i xstate @xstate/solid` |
| `@xstate/store-solid` | @xstate/store with Solid signals | `npm i @xstate/store-solid` |

## @xstate/solid — Machines in Solid

### useActor(logic, options?)

Creates and starts an actor for the component's lifetime. Returns `[snapshot, send, actorRef]`:

```tsx
import { createMachine } from 'xstate';
import { useActor } from '@xstate/solid';

const toggleMachine = createMachine({
  id: 'toggle',
  initial: 'inactive',
  states: {
    inactive: { on: { TOGGLE: 'active' } },
    active: { on: { TOGGLE: 'inactive' } },
  },
});

function Toggler() {
  const [snapshot, send] = useActor(toggleMachine);

  return (
    <button onclick={() => send({ type: 'TOGGLE' })}>
      {snapshot.value === 'inactive' ? 'Activate' : 'Deactivate'}
    </button>
  );
}
```

With input and provided implementations:

```tsx
function UserProfile(props: { userId: string }) {
  const [snapshot, send] = useActor(
    userMachine.provide({
      actions: {
        notifyParent: () => props.onComplete?.(),
      },
    }),
    { input: { userId: props.userId } },
  );

  return <div>{snapshot.context.name}</div>;
}
```

### useActorRef(logic, options?)

Returns only the actor ref without triggering reactivity on snapshot changes. Use when you need the ref but not re-renders:

```tsx
import { useActorRef } from '@xstate/solid';

function Logger() {
  const actorRef = useActorRef(logMachine, {
    input: { level: 'debug' },
  });

  // No re-renders on state changes
  return <button onclick={() => actorRef.send({ type: 'LOG' })}>Log</button>;
}
```

Combine with `useSelector` from `@xstate/solid` for selective reactivity:

```tsx
import { useActorRef, useSelector } from '@xstate/solid';

function Counter() {
  const actorRef = useActorRef(counterMachine);
  const count = useSelector(actorRef, (s) => s.context.count);

  return <span>{count()}</span>;
}
```

### fromActorRef(actorRef)

Subscribes to an existing actor ref's snapshot. Accepts a static ref or a Signal:

```tsx
import { fromActorRef } from '@xstate/solid';

function ChildView(props: { actorRef: AnyActorRef }) {
  const snapshot = fromActorRef(() => props.actorRef);

  return <div>Status: {snapshot().status}</div>;
}
```

## State Matching with Solid Control Flow

Use `snapshot.matches()` with Solid's `<Switch>`/`<Match>`:

```tsx
import { Switch, Match } from 'solid-js';

function LoaderView() {
  const [snapshot, send] = useActor(loaderMachine);

  return (
    <Switch fallback={<p>Unknown state</p>}>
      <Match when={snapshot.matches('idle')}>
        <button onclick={() => send({ type: 'FETCH' })}>Load</button>
      </Match>
      <Match when={snapshot.matches('loading')}>
        <p>Loading...</p>
      </Match>
      <Match when={snapshot.matches('success')}>
        <p>Data: {JSON.stringify(snapshot.context.data)}</p>
      </Match>
      <Match when={snapshot.matches('error')}>
        <p>Error! <button onclick={() => send({ type: 'RETRY' })}>Retry</button></p>
      </Match>
    </Switch>
  );
}
```

For hierarchical states:

```tsx
<Match when={snapshot.matches({ loading: 'user' })}>
  <UserSkeleton />
</Match>
```

## Persisted/Rehydrated State

```tsx
function App() {
  const persisted = JSON.parse(localStorage.getItem('state') ?? 'null');

  const [snapshot, send] = useActor(someMachine, {
    snapshot: persisted ?? undefined,
  });

  // Persist on changes
  createEffect(() => {
    localStorage.setItem('state', JSON.stringify(snapshot));
  });

  return <div>...</div>;
}
```

## Actor Subscriptions with Solid Lifecycle

```tsx
import { createEffect, onCleanup } from 'solid-js';

function ActorLogger() {
  const [snapshot, send, actorRef] = useActor(someMachine);

  createEffect(() => {
    const sub = actorRef.subscribe((s) => {
      console.log('State:', s.value);
    });
    onCleanup(() => sub.unsubscribe());
  });

  return <div>...</div>;
}
```

## @xstate/store-solid — Stores in Solid

### useSelector(store, selector?, compare?)

Subscribes to an `@xstate/store` store, returning a Solid signal:

```tsx
import { createStore, useSelector } from '@xstate/store-solid';

const store = createStore({
  context: { count: 0 },
  on: {
    inc: (context, event: { by?: number }) => ({
      ...context,
      count: context.count + (event.by ?? 1),
    }),
  },
});

function Counter() {
  // Returns a Solid signal — call as function to read
  const count = useSelector(store, (state) => state.context.count);

  return (
    <div>
      <p>Count: {count()}</p>
      <button onclick={() => store.trigger.inc()}>+1</button>
      <button onclick={() => store.trigger.inc({ by: 5 })}>+5</button>
    </div>
  );
}
```

With custom comparison:

```tsx
const user = useSelector(
  store,
  (state) => state.context.user,
  (prev, next) => prev.id === next.id,
);
```

**Key:** Always call the signal as a function (`count()`, not `count`) — this is how Solid tracks dependencies.

## Solid-Specific Patterns

### Context Provider with XState

```tsx
import { createContext, useContext } from 'solid-js';
import { useActorRef } from '@xstate/solid';

const MachineContext = createContext<ReturnType<typeof useActorRef>>();

function MachineProvider(props: { children: any }) {
  const actorRef = useActorRef(appMachine);
  return (
    <MachineContext.Provider value={actorRef}>
      {props.children}
    </MachineContext.Provider>
  );
}

function useAppMachine() {
  const ref = useContext(MachineContext);
  if (!ref) throw new Error('Missing MachineProvider');
  return ref;
}

// In a child component
function StatusBar() {
  const actorRef = useAppMachine();
  const status = useSelector(actorRef, (s) => s.value);
  return <span>{status()}</span>;
}
```

### Spawned Actor Lists with `<For>`

```tsx
import { For } from 'solid-js';
import { fromActorRef } from '@xstate/solid';

function TodoList() {
  const [snapshot, send] = useActor(todosMachine);

  return (
    <For each={Object.values(snapshot.children)}>
      {(todoRef) => <TodoItem actorRef={todoRef} />}
    </For>
  );
}

function TodoItem(props: { actorRef: AnyActorRef }) {
  const todo = fromActorRef(() => props.actorRef);
  return <li>{todo().context.text}</li>;
}
```

## Anti-Patterns

### Forgetting to Call Signals

```tsx
// BAD — reads the signal accessor, not the value
<p>Count: {count}</p>   // Shows "[Function]"

// GOOD — call the signal
<p>Count: {count()}</p>
```

### Creating Actors Outside Components

```tsx
// BAD — actor not tied to component lifecycle, leaks
const [snapshot, send] = useActor(machine); // at module level

// GOOD — inside component or createRoot
function App() {
  const [snapshot, send] = useActor(machine);
  // ...
}
```

### Using React Patterns in Solid

```tsx
// BAD — useEffect/useState are React patterns
useEffect(() => { /* ... */ }, [dep]);

// GOOD — Solid uses createEffect/createSignal
createEffect(() => { /* tracked automatically */ });
```
