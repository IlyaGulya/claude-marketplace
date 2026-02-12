---
name: xstate-actors-and-invocation
description: Covers XState v5 actor model, actor types, invocation, spawning, and communication. Use when choosing between promise/callback/observable/state-machine actors, implementing invoke vs spawn, designing parent-child communication, or managing actor lifecycle with input/output.
---

# XState v5 Actors and Invocation

## Actor Model

In XState, actors are independent entities that:
- Have their own encapsulated internal state
- Communicate via asynchronous message passing (events)
- Process one message at a time (internal "mailbox" queue)
- Cannot share or directly access another actor's state
- Can create (spawn/invoke) new actors

## Actor Logic Types

| Type | Receive Events | Send Events | Spawn Actors | Input | Output |
|------|:---:|:---:|:---:|:---:|:---:|
| `createMachine()` | Yes | Yes | Yes | Yes | Yes |
| `fromPromise()` | No | Yes | No | Yes | Yes |
| `fromCallback()` | Yes | Yes | No | Yes | No |
| `fromObservable()` | No | Yes | No | Yes | No |
| `fromEventObservable()` | No | Yes | No | Yes | No |
| `fromTransition()` | Yes | Yes | No | Yes | No |

### Promise Actors

For async operations that resolve or reject:

```ts
import { fromPromise } from 'xstate';

const fetchUser = fromPromise(async ({ input }: { input: { userId: string } }) => {
  const res = await fetch(`/api/users/${input.userId}`);
  if (!res.ok) throw new Error('Failed');
  return res.json(); // This becomes event.output in onDone
});
```

### Callback Actors

For bidirectional communication, event listeners, intervals:

```ts
import { fromCallback } from 'xstate';

const keyListener = fromCallback(({ sendBack, receive, input }) => {
  const handler = (e: KeyboardEvent) => {
    sendBack({ type: 'KEY_PRESS', key: e.key });
  };

  document.addEventListener('keydown', handler);

  // Receive events from parent
  receive((event) => {
    if (event.type === 'PAUSE') { /* ... */ }
  });

  // Cleanup function — called when actor is stopped
  return () => document.removeEventListener('keydown', handler);
});
```

### Observable Actors

For streams of values (requires RxJS or compatible):

```ts
import { fromObservable, fromEventObservable } from 'xstate';
import { interval, fromEvent } from 'rxjs';

// Value observable — emits snapshots
const ticker = fromObservable(() => interval(1000));

// Event observable — emits events directly to parent
const clicks = fromEventObservable(
  () => fromEvent(document, 'click') as any,
);
```

### Transition Actors

Reducer-style logic:

```ts
import { fromTransition } from 'xstate';

const counter = fromTransition(
  (state, event) => {
    if (event.type === 'INCREMENT') return { count: state.count + 1 };
    if (event.type === 'DECREMENT') return { count: state.count - 1 };
    return state;
  },
  { count: 0 }, // initial state
);
```

## Invoke vs Spawn

### Invoke — State-bound lifecycle

Use `invoke` when the actor's lifecycle is tied to a specific state:

```ts
states: {
  loading: {
    // Actor starts when entering 'loading', stops when exiting
    invoke: {
      src: 'fetchData',
      input: ({ context }) => ({ url: context.url }),
      onDone: { target: 'success', actions: assign({ data: ({ event }) => event.output }) },
      onError: { target: 'error', actions: assign({ error: ({ event }) => event.error }) },
    },
  },
}
```

**Use invoke for:** API calls, data loading, single-purpose async tasks, state-scoped subscriptions.

### Spawn — Action-based lifecycle

Use `spawn`/`spawnChild` when actors need to:
- Survive across multiple states
- Be created dynamically (unknown number)
- Be stopped manually

```ts
import { spawnChild, stopChild, assign } from 'xstate';

on: {
  'todo.add': {
    actions: spawnChild('todoMachine', {
      id: ({ event }) => `todo-${event.id}`,
      input: ({ event }) => ({ text: event.text }),
    }),
  },
  'todo.remove': {
    actions: stopChild(({ event }) => `todo-${event.id}`),
  },
}
```

Or with `spawn` in `assign` to keep a reference:

```ts
on: {
  'worker.start': {
    actions: assign({
      workerRef: ({ spawn }) => spawn('workerLogic', { id: 'worker' }),
    }),
  },
  'worker.stop': {
    actions: [
      stopChild('worker'),
      assign({ workerRef: undefined }), // Clean up!
    ],
  },
}
```

### Decision Framework

| Criteria | Invoke | Spawn |
|----------|--------|-------|
| Known number of actors | Yes | Either |
| Dynamic number of actors | No | Yes |
| Lifecycle tied to a state | Yes | No |
| Need to survive state changes | No | Yes |
| Has onDone/onError | Yes | No |
| Automatic cleanup | Yes | Manual |

## Invoking

### Full API

```ts
states: {
  loading: {
    invoke: {
      src: 'fetchUser',        // Actor logic name (from setup) or inline logic
      id: 'userFetcher',       // Unique ID within parent
      input: ({ context }) => ({ userId: context.userId }),  // Input data
      onDone: {                // When actor completes successfully
        target: 'success',
        actions: assign({ user: ({ event }) => event.output }),
      },
      onError: {               // When actor throws/rejects
        target: 'failure',
        actions: assign({ error: ({ event }) => event.error }),
      },
      onSnapshot: {            // When actor emits a new snapshot
        actions: ({ event }) => console.log(event.snapshot),
      },
    },
  },
}
```

### Setup Actors

```ts
const machine = setup({
  actors: {
    fetchUser: fromPromise(async ({ input }: { input: { userId: string } }) => {
      return fetch(`/api/users/${input.userId}`).then(r => r.json());
    }),
    childMachine: childMachine,  // State machine actor
    listener: fromCallback(({ sendBack }) => { /* ... */ }),
  },
}).createMachine({
  states: {
    loading: {
      invoke: { src: 'fetchUser', /* ... */ },
    },
  },
});
```

### Multiple Invocations

```ts
states: {
  checking: {
    invoke: [
      { src: 'checkAuth', id: 'auth', onDone: '.authDone' },
      { src: 'loadConfig', id: 'config', onDone: '.configDone' },
    ],
  },
}
```

### Root-Level Invoke

Active for the entire machine lifetime:

```ts
const machine = createMachine({
  invoke: {
    src: fromEventObservable(() => fromEvent(document, 'click') as any),
  },
  on: {
    click: { actions: 'handleClick' },
  },
});
```

## Spawning

### spawnChild (preferred — no context reference)

```ts
import { spawnChild } from 'xstate';

entry: spawnChild('workerLogic', {
  id: 'worker-1',
  input: { batchSize: 100 },
}),

// Multiple
entry: [
  spawnChild('workerLogic', { id: 'worker-1' }),
  spawnChild('workerLogic', { id: 'worker-2' }),
],
```

### spawn in assign (when you need the reference)

```ts
actions: assign({
  workerRef: ({ spawn }) => spawn('workerLogic', { id: 'worker' }),
}),
```

**Important:** When using `spawn` in assign, always clean up when stopping:

```ts
actions: [stopChild('worker'), assign({ workerRef: undefined })],
```

## Input and Output

### Input (replaces factory functions)

```ts
// OLD way (v4 pattern — avoid)
const createMachine = (userId) => createMachine({ context: { userId } });

// NEW way — use input
const machine = setup({
  types: {
    input: {} as { userId: string },
    context: {} as { userId: string; data: null | object },
  },
}).createMachine({
  context: ({ input }) => ({
    userId: input.userId,
    data: null,
  }),
});

const actor = createActor(machine, { input: { userId: '42' } });
```

### Output (from final states)

```ts
const machine = createMachine({
  // ...
  states: {
    done: { type: 'final' },
  },
  output: ({ context }) => ({ result: context.processedData }),
});

// In parent — access via onDone
invoke: {
  src: 'childMachine',
  onDone: {
    actions: ({ event }) => console.log(event.output), // { result: ... }
  },
}
```

## Communication Patterns

### Parent → Child (via sendTo)

```ts
import { sendTo } from 'xstate';

on: {
  UPDATE_CHILD: {
    actions: sendTo('childActorId', ({ event }) => ({
      type: 'UPDATE',
      data: event.data,
    })),
  },
}
```

### Child → Parent (via input ref — preferred over sendParent)

```ts
// Child machine — receives parent ref via input
const childMachine = setup({
  types: {
    context: {} as { parentRef: AnyActorRef },
    input: {} as { parentRef: AnyActorRef },
  },
}).createMachine({
  context: ({ input }) => ({ parentRef: input.parentRef }),
  on: {
    DONE: {
      actions: sendTo(
        ({ context }) => context.parentRef,
        { type: 'CHILD_COMPLETED' },
      ),
    },
  },
});

// Parent machine — passes self via input
const parentMachine = setup({
  actors: { child: childMachine },
}).createMachine({
  invoke: {
    src: 'child',
    input: ({ self }) => ({ parentRef: self }),
  },
  on: {
    CHILD_COMPLETED: { /* handle */ },
  },
});
```

### System-Level (systemId)

For actors that need to be globally addressable:

```ts
invoke: {
  src: 'logger',
  systemId: 'logger', // Unique across the entire actor system
}

// Any actor in the system can address it
actions: sendTo(({ system }) => system.get('logger'), { type: 'LOG' }),
```

## Lifecycle

- **Invoked actors** start on state entry, stop on state exit
- **Spawned actors** start when spawned, survive state changes, stop when parent stops or `stopChild()` is called
- If a state is entered and immediately exited (via `always`), invoked actors are NOT started

### toPromise

Convert any actor to a Promise:

```ts
import { toPromise } from 'xstate';

const actor = createActor(machine).start();
const output = await toPromise(actor);
// Resolves with actor's output when done, rejects on error
```

## Anti-Patterns

### Orphaned Spawned Refs

```ts
// BAD — spawned ref in context not cleaned up
actions: stopChild('worker'),
// workerRef still in context, pointing to stopped actor!

// GOOD — always clean up
actions: [stopChild('worker'), assign({ workerRef: undefined })],
```

### Using sendParent() (tight coupling)

```ts
// BAD — child is tightly coupled to parent's event types
import { sendParent } from 'xstate';
actions: sendParent({ type: 'DONE' }),

// GOOD — pass parent ref via input
context: ({ input }) => ({ parentRef: input.parentRef }),
actions: sendTo(({ context }) => context.parentRef, { type: 'DONE' }),
```

### Async in Actions

```ts
// BAD — actions are NOT awaited
entry: async () => { await fetch('/api') },

// GOOD — use invoke for async
invoke: {
  src: fromPromise(() => fetch('/api')),
  onDone: { /* ... */ },
  onError: { /* ... */ },
}
```

### Missing onError

```ts
// BAD — unhandled rejection will throw
invoke: { src: 'fetchData', onDone: 'success' },

// GOOD — always handle errors
invoke: {
  src: 'fetchData',
  onDone: { target: 'success' },
  onError: { target: 'error' },
},
```
