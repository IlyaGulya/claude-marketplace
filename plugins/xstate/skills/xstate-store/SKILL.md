---
name: xstate-store
description: |
  Covers @xstate/store v3 — lightweight event-driven state management.
  Use when implementing simple stores with createStore(), atoms, effects, emitting events, selectors, undo/redo, or bridging stores with XState actors via fromStore().
---

# @xstate/store v3

Lightweight state management — simpler alternative to full XState machines. Comparable to Zustand/Redux but event-driven. Requires TypeScript 5.4+.

## Creating a Store

```ts
import { createStore } from '@xstate/store';

const store = createStore({
  context: { count: 0, name: 'David' },
  on: {
    inc: (context) => ({
      ...context,
      count: context.count + 1,
    }),
    add: (context, event: { num: number }) => ({
      ...context,
      count: context.count + event.num,
    }),
    changeName: (context, event: { newName: string }) => ({
      ...context,
      name: event.newName,
    }),
  },
});

// Read snapshot
store.getSnapshot(); // { status: 'active', context: { count: 0, name: 'David' } }

// Subscribe to changes
store.subscribe((snapshot) => console.log(snapshot.context));

// Send events
store.send({ type: 'inc' });

// Fluent trigger API
store.trigger.add({ num: 10 });
store.trigger.changeName({ newName: 'Jenny' });
```

**Important:** When using function transitions, return the **complete** context object (spread `...context` to preserve other properties).

## Effects

Enqueue side effects in transitions via the 3rd argument:

```ts
const store = createStore({
  context: { count: 0 },
  on: {
    incrementDelayed: (context, event, enqueue) => {
      enqueue.effect(() => {
        setTimeout(() => {
          store.send({ type: 'increment' });
        }, 1000);
      });
      return context; // return unchanged context
    },
    increment: (context) => ({
      ...context,
      count: context.count + 1,
    }),
  },
});
```

**Critical:** `enqueue.effect()` and `enqueue.emit()` must be called **synchronously** within the transition. For async operations, fire an effect that sends an event back:

```ts
on: {
  fetchData: (context, event, enqueue) => {
    enqueue.effect(async () => {
      const data = await fetch('/api/data').then((r) => r.json());
      store.trigger.dataLoaded({ data });
    });
    return context;
  },
  dataLoaded: (context, event: { data: string }) => ({
    ...context,
    data: event.data,
  }),
}
```

## Emitting Events

Notify external listeners from transitions:

```ts
const store = createStore({
  context: { count: 0 },
  emits: {
    increased: (payload: { by: number }) => {},
  },
  on: {
    inc: (context, event: { by: number }, enqueue) => {
      enqueue.emit.increased({ by: event.by });
      return { ...context, count: context.count + event.by };
    },
  },
});

// Listen for emitted events
store.on('increased', (event) => {
  console.log('Count increased by ' + event.by);
});
```

## Selectors

Efficient subscriptions to specific parts of state:

```ts
const position = store.select((context) => context.position);

position.get();  // { x: 0, y: 0 }
position.subscribe((pos) => console.log('Moved:', pos));

// With custom equality
const position = store.select(
  (state) => state.context.position,
  (prev, next) => prev.x === next.x,
);
```

Built-in `shallowEqual`:

```ts
import { shallowEqual } from '@xstate/store';
const pos = store.select((s) => s.context.position, shallowEqual);
```

## Pure Transitions (for testing)

Compute next state without mutating the store:

```ts
const snapshot = store.getSnapshot();
const [nextState, effects] = store.transition(snapshot, { type: 'inc', by: 1 });

nextState.context; // { count: 1 }
effects;           // [{ type: 'incremented', by: 1 }, Function]

store.getSnapshot().context; // { count: 0 } — unchanged
```

## Atoms

Lightweight reactive state pieces:

```ts
import { createAtom } from '@xstate/store';

const countAtom = createAtom(0);

countAtom.get();           // 0
countAtom.set(1);          // 1
countAtom.set((p) => p+1); // 2

countAtom.subscribe((val) => console.log(val));
```

### Computed Atoms

Derived from other atoms/stores — read-only, auto-updating:

```ts
const nameAtom = createAtom('David');
const ageAtom = createAtom(30);

const userAtom = createAtom(() => ({
  name: nameAtom.get(),
  age: ageAtom.get(),
}));

userAtom.get(); // { name: 'David', age: 30 }
nameAtom.set('John');
userAtom.get(); // { name: 'John', age: 30 }
```

With previous value access:

```ts
const totalAtom = createAtom<number>((_, prev) => countAtom.get() + (prev ?? 0));
```

### Async Atoms

```ts
import { createAsyncAtom } from '@xstate/store';

const userAtom = createAsyncAtom(async () => {
  const res = await fetch('/api/user');
  return res.json();
});

userAtom.subscribe((snapshot) => {
  if (snapshot.status === 'pending') { /* loading */ }
  if (snapshot.status === 'done') { console.log(snapshot.data); }
  if (snapshot.status === 'error') { console.error(snapshot.error); }
});
```

### Atoms with Stores

```ts
const countSelector = store.select((s) => s.context.count);
const doubleCount = createAtom(() => 2 * countSelector.get());
```

## Undo/Redo

Built-in via `undoRedo` extension:

```ts
import { createStore } from '@xstate/store';
import { undoRedo } from '@xstate/store/undo';

const store = createStore({
  context: { count: 0 },
  on: {
    inc: (context) => ({ count: context.count + 1 }),
  },
}).with(undoRedo());

store.trigger.inc(); // count = 1
store.trigger.inc(); // count = 2
store.trigger.undo(); // count = 1
store.trigger.redo(); // count = 2
```

### Transactions (group events)

```ts
.with(undoRedo({
  getTransactionId: (event) => event.type, // group by type
}));

store.trigger.inc(); // batch 1
store.trigger.inc(); // batch 1
store.trigger.dec(); // batch 2
store.trigger.undo(); // undoes both decs
```

### Skip Events

```ts
.with(undoRedo({
  skipEvent: (event) => event.type === 'log',
}));
```

### Snapshot Strategy

```ts
.with(undoRedo({
  strategy: 'snapshot',
  historyLimit: 50,
  compare: (past, current) => past.context.count === current.context.count,
}));
```

Default is `'event-sourced'` (replays events). `'snapshot'` stores full state copies.

## Inspection

```ts
store.inspect((inspectionEvent) => {
  // type: '@xstate.snapshot' or '@xstate.event'
  console.log(inspectionEvent);
});
```

Works with Stately Inspector:

```ts
import { createBrowserInspector } from '@statelyai/inspect';
const inspector = createBrowserInspector();
store.inspect(inspector);
```

## Using Immer

```ts
import { produce } from 'immer';

const store = createStore({
  context: { count: 0, todos: [] as string[] },
  on: {
    inc: (context) => produce(context, (draft) => { draft.count++; }),
    addTodo: (context, event: { todo: string }) =>
      produce(context, (draft) => { draft.todos.push(event.todo); }),
  },
});
```

## Bridge to XState: fromStore()

Convert store logic to XState actor logic:

```ts
import { fromStore } from '@xstate/store';
import { createActor } from 'xstate';

const storeLogic = fromStore({
  context: { count: 0 },
  on: {
    inc: (context) => ({ ...context, count: context.count + 1 }),
  },
});

const actor = createActor(storeLogic);
actor.subscribe((snapshot) => console.log(snapshot));
actor.start();
actor.send({ type: 'inc' });
```

With input:

```ts
const storeLogic = fromStore({
  context: (initialCount: number) => ({ count: initialCount }),
  on: { /* ... */ },
});

const actor = createActor(storeLogic, { input: 42 });
```

## Converting Store to State Machine

1. Replace `createStore` with `createMachine`
2. Wrap assignments in `assign()`
3. Destructure `{ context, event }` from first arg

```ts
// Store
const store = createStore({
  context: { count: 0 },
  on: {
    inc: (context, event: { by: number }) => ({
      ...context,
      count: context.count + event.by,
    }),
  },
});

// Equivalent machine
const machine = createMachine({
  context: { count: 0 },
  on: {
    inc: {
      actions: assign({
        count: ({ context, event }) => context.count + event.by,
      }),
    },
  },
});
```

## Framework Integrations

| Framework | Package |
|-----------|---------|
| React | `@xstate/store-react` |
| Solid | `@xstate/store-solid` |
| Vue | `@xstate/store-vue` |
| Svelte | `@xstate/store-svelte` |
| Angular | `@xstate/store-angular` |

Deprecated: importing from `@xstate/store/react`, `@xstate/store/solid`, etc. Use dedicated packages.

## When to Use Store vs Machine

| Criteria | @xstate/store | XState machine |
|----------|---------------|----------------|
| Simple state updates | Yes | Overkill |
| Multiple states with different behaviors | No | Yes |
| Async with error handling | Manual | Built-in (invoke) |
| Complex event sequences | No | Yes |
| Guards/conditions | No | Yes |
| Nested/parallel states | No | Yes |
| Quick Zustand/Redux replacement | Yes | No |
