---
name: xstate-event-emitter
description: |
  Covers XState v5 event emitter pattern for outward-facing events.
  Use when emitting events to external handlers via emit() action, subscribing to emitted events with actor.on(), or typing emitted events. Available since XState 5.9.0.
---

# XState v5 Event Emitter

Emitted events go **outward** from the actor to external listeners — the opposite of `actor.send()` which sends events **inward**.

## emit() Action Creator

Emit events from state machine transitions:

```ts
import { setup, emit, createActor } from 'xstate';

const machine = setup({
  types: {
    emitted: {} as
      | { type: 'notification'; message: string }
      | { type: 'error'; code: number },
  },
}).createMachine({
  on: {
    submit: {
      actions: emit({ type: 'notification', message: 'Submitted!' }),
    },
  },
});

const actor = createActor(machine);

// Subscribe to emitted events
actor.on('notification', (event) => {
  console.log(event.message); // 'Submitted!'
});

actor.start();
actor.send({ type: 'submit' });
```

### Static vs Dynamic emit

```ts
setup({
  actions: {
    // Static — fixed event
    emitStatic: emit({
      type: 'notification',
      message: 'Hello',
    }),

    // Dynamic — based on context/event
    emitDynamic: emit(({ context }) => ({
      type: 'notification',
      message: 'Count is ' + context.count,
    })),
  },
}).createMachine({ /* ... */ });
```

## Listening for Emitted Events

### actor.on(eventType, handler)

Returns a subscription object:

```ts
const actor = createActor(machine);

const sub = actor.on('notification', (event) => {
  console.log(event.message);
});

actor.start();

// Later: stop listening
sub.unsubscribe();
```

### Wildcard Listener

Listen to all emitted events with `'*'`:

```ts
actor.on('*', (emitted) => {
  console.log(emitted); // Union of all emitted event types
});
```

## Emitting from Actor Logic Types

All actor logic creators support `emit`:

### Promise Actors

```ts
const logic = fromPromise(async ({ emit }) => {
  emit({ type: 'progress', percent: 50 });
  const result = await doWork();
  emit({ type: 'progress', percent: 100 });
  return result;
});
```

### Callback Actors

```ts
const logic = fromCallback(({ emit }) => {
  const interval = setInterval(() => {
    emit({ type: 'tick' });
  }, 1000);

  return () => clearInterval(interval);
});
```

### Observable Actors

```ts
const logic = fromObservable(({ emit }) => {
  emit({ type: 'started' });
  return interval(1000);
});
```

### Transition Actors

```ts
const logic = fromTransition((state, event, { emit }) => {
  if (event.type === 'INCREMENT') {
    emit({ type: 'changed', value: state.count + 1 });
    return { count: state.count + 1 };
  }
  return state;
}, { count: 0 });
```

## TypeScript

Strongly type emitted events in `setup()`:

```ts
const machine = setup({
  types: {
    emitted: {} as
      | { type: 'notification'; message: string }
      | { type: 'error'; error: Error },
  },
}).createMachine({ /* ... */ });

const actor = createActor(machine);

// Fully typed — event.message is string
actor.on('notification', (event) => {
  console.log(event.message);
});

// Type error: 'unknown' is not a valid emitted event
// actor.on('unknown', (event) => {});
```

## Use Cases

### Decoupling UI Effects from Machine Logic

```ts
const formMachine = setup({
  types: {
    emitted: {} as
      | { type: 'toast'; message: string; variant: 'success' | 'error' }
      | { type: 'analytics'; event: string },
  },
}).createMachine({
  states: {
    submitting: {
      invoke: {
        src: 'submitForm',
        onDone: {
          target: 'success',
          actions: [
            emit({ type: 'toast', message: 'Saved!', variant: 'success' }),
            emit(({ context }) => ({
              type: 'analytics',
              event: 'form_submitted',
            })),
          ],
        },
        onError: {
          target: 'error',
          actions: emit({ type: 'toast', message: 'Failed', variant: 'error' }),
        },
      },
    },
    success: {},
    error: {},
  },
});

// In the UI layer — connect to whatever toast/analytics system you use
const actor = createActor(formMachine);
actor.on('toast', ({ message, variant }) => showToast(message, variant));
actor.on('analytics', ({ event }) => analytics.track(event));
```

### vs sendParent / vs context

| Pattern | When to Use |
|---------|-------------|
| `emit()` | External side effects (toasts, analytics, logging) |
| `sendTo(parentRef)` | Parent-child actor communication |
| `assign()` | Data that the machine needs to track internally |

`emit()` is best when the machine shouldn't know or care about what happens with the event.
