---
name: xstate-inspection
description: |
  Covers XState v5 Inspect API and developer tools.
  Use when debugging actor systems, inspecting state transitions, integrating Stately Inspector, or observing actor lifecycle events and microsteps.
---

# XState v5 Inspection and DevTools

## Inspect API

Attach an inspector to observe everything in an actor system:

```ts
import { createActor } from 'xstate';

const actor = createActor(machine, {
  inspect: (inspectionEvent) => {
    console.log(inspectionEvent);
  },
});

actor.start();
```

The inspector receives events for **every** actor in the system (root + all invoked/spawned children).

## Inspection Event Types

### @xstate.actor — Actor Created

```ts
{
  type: '@xstate.actor',
  actorRef: /* actor reference */,
  rootId: 'x:0',
}
```

Emitted when any actor in the system is created.

### @xstate.event — Event Sent

```ts
{
  type: '@xstate.event',
  actorRef: /* target actor */,
  rootId: 'x:0',
  event: { type: 'someEvent', data: 'hello' },
  sourceRef: /* sending actor (or undefined if external) */,
}
```

Emitted when an event is sent to any actor. `sourceRef` is `undefined` for externally sent events.

### @xstate.snapshot — State Updated

```ts
{
  type: '@xstate.snapshot',
  actorRef: /* actor reference */,
  rootId: 'x:0',
  snapshot: {
    status: 'active',
    context: { count: 31 },
    // ...
  },
  event: { type: 'increment' },
}
```

Emitted when any actor's snapshot changes.

### @xstate.microstep — Transition Details

```ts
{
  type: '@xstate.microstep',
  value: 'c',
  event: { type: 'EV' },
  transitions: [
    { eventType: 'EV', target: ['(machine).b'] },
    { eventType: '', target: ['(machine).c'] },  // eventless transition
  ],
}
```

Shows each individual state transition including intermediate steps through `always` transitions. Empty `eventType` indicates eventless transitions.

## Stately Inspector

Visual debugging tool for XState actors:

```ts
import { createBrowserInspector } from '@statelyai/inspect';
import { createActor } from 'xstate';

const inspector = createBrowserInspector({
  // Opens inspector in a new window
});

const actor = createActor(machine, {
  inspect: inspector,
});

actor.start();
```

Install: `npm i @statelyai/inspect`

## Inspecting Stores

`@xstate/store` also supports the Inspect API:

```ts
import { createStore } from '@xstate/store';

const store = createStore({ /* ... */ });

store.inspect((inspectionEvent) => {
  console.log(inspectionEvent);
  // @xstate.snapshot or @xstate.event
});

// With Stately Inspector
store.inspect(inspector);
```

The store inspector receives the initial snapshot immediately (store is auto-started).

## Practical Patterns

### Debug Logger

```ts
const actor = createActor(machine, {
  inspect: (event) => {
    switch (event.type) {
      case '@xstate.event':
        console.log(
          '[EVENT]',
          event.event.type,
          event.sourceRef ? 'from ' + event.sourceRef.id : '(external)',
        );
        break;
      case '@xstate.snapshot':
        console.log('[STATE]', event.snapshot.value);
        break;
      case '@xstate.actor':
        console.log('[ACTOR]', event.actorRef.id, 'created');
        break;
    }
  },
});
```

### Event Recorder (for Event Sourcing)

```ts
const events: any[] = [];

const actor = createActor(machine, {
  inspect: (inspEvent) => {
    if (inspEvent.type === '@xstate.event' && inspEvent.actorRef === actor) {
      events.push(inspEvent.event);
    }
  },
});

actor.start();
// events[] now contains all events sent to the root actor
```

### Filter by Actor

```ts
const actor = createActor(machine, {
  inspect: (event) => {
    if (event.type === '@xstate.snapshot') {
      // Only log the root actor's state changes
      if (event.actorRef === actor) {
        console.log('Root state:', event.snapshot.value);
      }
    }
  },
});
```

### Conditional Inspector (dev only)

```ts
const actor = createActor(machine, {
  inspect: import.meta.env.DEV
    ? (event) => console.log(event)
    : undefined,
});
```
