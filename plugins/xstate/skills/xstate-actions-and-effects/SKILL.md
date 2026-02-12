---
name: xstate-actions-and-effects
description: Covers XState v5 action types and side-effect patterns. Use when implementing entry/exit actions, assign() for context updates, raise(), sendTo(), enqueueActions(), or deciding where to place effects. Includes type-bound action helpers (v5.22+).
---

# XState v5 Actions and Effects

## Fundamentals

Actions are **fire-and-forget** side effects. They run once and the machine does not wait for them.

### Placement

Actions can be placed in three locations:

```ts
states: {
  active: {
    // 1. Entry — runs when entering this state
    entry: [{ type: 'logEnter' }],

    // 2. Exit — runs when leaving this state
    exit: [{ type: 'logExit' }],

    on: {
      CLICK: {
        target: 'next',
        // 3. Transition — runs during the transition
        actions: [{ type: 'handleClick' }],
      },
    },
  },
}
```

### Execution Order

On a transition from state A to state B:
1. Transition actions
2. A's exit actions
3. B's entry actions

## Named Actions

Define in `setup()`, reference by string or object:

```ts
const machine = setup({
  actions: {
    logEvent: () => console.log('Event occurred'),
    track: (_, params: { category: string; action: string }) => {
      analytics.track(params.category, params.action);
    },
  },
}).createMachine({
  entry: [
    // String shorthand
    'logEvent',
    // Object form with params
    { type: 'track', params: { category: 'page', action: 'view' } },
  ],
});
```

### Dynamic Params

```ts
const machine = setup({
  actions: {
    track: (_, params: { value: number }) => {
      analytics.track('count', params.value);
    },
  },
}).createMachine({
  context: { count: 0 },
  entry: {
    type: 'track',
    params: ({ context }) => ({ value: context.count }),
  },
});
```

### Overriding with .provide()

```ts
const testMachine = machine.provide({
  actions: {
    track: () => { /* mock implementation */ },
  },
});
```

## Assign

Updates context immutably. **Do not call `assign()` inside inline functions** — it returns an action object, not imperative code.

### Property Assigners (preferred)

```ts
import { assign } from 'xstate';

on: {
  INCREMENT: {
    actions: assign({
      count: ({ context }) => context.count + 1,
    }),
  },
  'item.add': {
    actions: assign({
      items: ({ context, event }) => [...context.items, event.item],
    }),
  },
}
```

### Function Assigners

```ts
on: {
  RESET: {
    actions: assign(({ context }) => ({
      count: 0,
      items: [],
      error: null,
    })),
  },
}
```

### Critical: Never Mutate

```ts
// BAD — mutation, causes bugs
assign({ items: ({ context }) => { context.items.push(x); return context.items; } })

// GOOD — immutable
assign({ items: ({ context }) => [...context.items, x] })
```

## Raise

Sends an event **to the same machine** (internal event queue):

```ts
import { raise } from 'xstate';

// Static event
entry: raise({ type: 'INTERNAL_EVENT' }),

// Dynamic event
entry: raise(({ context }) => ({
  type: 'PROCESS',
  data: context.pendingData,
})),

// Delayed raise
entry: raise({ type: 'TIMEOUT' }, { delay: 5000 }),
```

Raised events are processed **after** the current transition completes, before external events.

## SendTo

Sends an event **to a specific actor**:

```ts
import { sendTo } from 'xstate';

on: {
  FORWARD: {
    // To actor by ID
    actions: sendTo('childActor', { type: 'PROCESS' }),
  },
  NOTIFY: {
    // Dynamic event
    actions: sendTo('logger', ({ context }) => ({
      type: 'LOG',
      message: `Count: ${context.count}`,
    })),
  },
  DELAYED: {
    // With delay and cancellation ID
    actions: sendTo('worker', { type: 'START' }, { delay: 1000, id: 'delayedStart' }),
  },
}
```

Dynamic target (actor ref from context):

```ts
actions: sendTo(
  ({ context }) => context.parentRef,
  { type: 'CHILD_DONE' },
),
```

**Prefer `sendTo()` over `sendParent()`** — pass the parent ref via input for loose coupling. See the `xstate-actors-and-invocation` skill for the pattern.

## EnqueueActions

Conditional action execution — replaces the deprecated `choose()`:

```ts
import { enqueueActions } from 'xstate';

entry: enqueueActions(({ context, event, enqueue, check }) => {
  // Always enqueue
  enqueue.assign({ lastEvent: event.type });

  // Conditional
  if (context.count > 10) {
    enqueue({ type: 'notifyHighCount' });
  }

  // Guard-based condition
  if (check({ type: 'isAdmin' })) {
    enqueue.sendTo('adminPanel', { type: 'REFRESH' });
  }

  // Built-in helpers
  enqueue.raise({ type: 'INTERNAL' });
  enqueue.assign({ processed: true });
  enqueue.sendTo('logger', { type: 'LOG' });
}),
```

### Parameterized EnqueueActions

```ts
const machine = setup({
  actions: {
    processItems: enqueueActions(({ enqueue }, params: { batchSize: number }) => {
      for (let i = 0; i < params.batchSize; i++) {
        enqueue({ type: 'processItem' });
      }
    }),
  },
}).createMachine({
  entry: { type: 'processItems', params: { batchSize: 5 } },
});
```

**Warning:** Do NOT use `async` inside `enqueueActions`. Actions must be synchronous.

## Other Built-in Actions

### log()

```ts
import { log } from 'xstate';

entry: log(({ context }) => `Entered with count: ${context.count}`),
```

### cancel()

Cancels a delayed `sendTo()` or `raise()` by ID:

```ts
import { cancel } from 'xstate';

on: {
  ABORT: {
    actions: cancel('delayedStart'),  // Cancels the delayed action with this ID
  },
}
```

### stopChild()

```ts
import { stopChild } from 'xstate';

on: {
  STOP_WORKER: {
    actions: [
      stopChild('workerId'),
      assign({ workerRef: undefined }),  // Clean up reference
    ],
  },
}
```

### spawnChild()

```ts
import { spawnChild } from 'xstate';

on: {
  START_WORKER: {
    actions: spawnChild('workerLogic', {
      id: 'worker',
      input: ({ context }) => ({ data: context.data }),
    }),
  },
}
```

## Type-Bound Helpers (v5.22+)

Create fully typed actions outside the machine definition. Enables splitting large machines across files.

```ts
import { setup } from 'xstate';

const machineSetup = setup({
  types: {
    context: {} as { count: number; items: string[] },
    events: {} as { type: 'increment' } | { type: 'addItem'; item: string },
    emitted: {} as { type: 'COUNT_CHANGED'; count: number },
  },
});

// Type-bound assign — can be in a separate file
const incrementCount = machineSetup.assign({
  count: ({ context }) => context.count + 1,
});

const addItem = machineSetup.assign({
  items: ({ context, event }) => [...context.items, event.item],
});

// Type-bound raise
const raiseIncrement = machineSetup.raise({ type: 'increment' });

// Type-bound emit
const emitCountChanged = machineSetup.emit(({ context }) => ({
  type: 'COUNT_CHANGED',
  count: context.count,
}));

// Custom action with createAction
const logState = machineSetup.createAction(({ context, event }) => {
  console.log(`Count: ${context.count}, Event: ${event.type}`);
});

// Type-bound enqueueActions
const batchUpdate = machineSetup.enqueueActions(({ enqueue }) => {
  enqueue(incrementCount);
  enqueue(logState);
});

// Use in machine — all fully typed
const machine = machineSetup.createMachine({
  context: { count: 0, items: [] },
  initial: 'active',
  states: {
    active: {
      entry: [incrementCount, logState],
      on: {
        increment: { actions: [incrementCount, emitCountChanged] },
        addItem: { actions: addItem },
      },
    },
  },
});
```

Available type-bound helpers:
- `machineSetup.assign()`
- `machineSetup.raise()`
- `machineSetup.emit()`
- `machineSetup.sendTo()`
- `machineSetup.log()`
- `machineSetup.cancel()`
- `machineSetup.spawnChild()`
- `machineSetup.stopChild()`
- `machineSetup.enqueueActions()`
- `machineSetup.createAction()`

## Anti-Patterns

### Calling assign() Inside Inline Functions

```ts
// BAD — assign() returns an object, doesn't execute imperatively
entry: ({ context }) => {
  assign({ count: context.count + 1 }); // Does nothing!
},

// GOOD — use assign directly
entry: assign({ count: ({ context }) => context.count + 1 }),

// GOOD — or use enqueueActions for imperative style
entry: enqueueActions(({ context, enqueue }) => {
  enqueue.assign({ count: context.count + 1 });
}),
```

### Async in Actions

```ts
// BAD — async actions are NOT awaited
entry: async ({ context }) => {
  const data = await fetchData(); // Machine doesn't wait for this!
},

// GOOD — use invoke for async operations
loading: {
  invoke: {
    src: 'fetchData',
    onDone: { target: 'success', actions: assign({ data: ({ event }) => event.output }) },
    onError: { target: 'error' },
  },
}
```

### Missing Implementations

```ts
// BAD — string reference with no implementation
entry: 'doSomething', // Will be a no-op if not provided

// GOOD — implement in setup or provide later
const machine = setup({
  actions: {
    doSomething: () => { /* implementation */ },
  },
}).createMachine({
  entry: { type: 'doSomething' },
});
```
