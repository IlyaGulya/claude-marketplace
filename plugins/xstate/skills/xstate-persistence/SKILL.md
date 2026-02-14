---
name: xstate-persistence
description: |
  Covers XState v5 state persistence and restoration.
  Use when saving/restoring actor state to localStorage, databases, or other storage, implementing deep persistence of actor hierarchies, or using event sourcing for state replay.
---

# XState v5 Persistence

## Persisting State

Get the internal state to persist via `actor.getPersistedSnapshot()`:

```ts
const feedbackActor = createActor(feedbackMachine).start();

// Get state to persist
const persistedState = feedbackActor.getPersistedSnapshot();

// Save to storage
localStorage.setItem('feedback', JSON.stringify(persistedState));
```

**Important:** `getPersistedSnapshot()` is NOT the same as `getSnapshot()`:
- `getPersistedSnapshot()` — internal state for persistence/restoration
- `getSnapshot()` — last emitted value (what subscribers see)

## Restoring State

Pass persisted state as `snapshot` option to `createActor`:

```ts
const restoredState = JSON.parse(localStorage.getItem('feedback'));

const restoredActor = createActor(feedbackMachine, {
  snapshot: restoredState,
});

restoredActor.start();
```

**Key behaviors on restoration:**
- Entry actions are **NOT** re-executed (assumed already executed)
- Invoked actors **ARE** restarted
- Spawned actors **ARE** restored recursively

## Deep Persistence

Persisting a parent actor automatically includes all invoked and spawned children:

```ts
const feedbackMachine = createMachine({
  states: {
    form: {
      invoke: {
        id: 'form',
        src: formMachine,
      },
    },
  },
});

const actor = createActor(feedbackMachine).start();

// Persists both feedbackActor AND the invoked formMachine
const persisted = actor.getPersistedSnapshot();
localStorage.setItem('feedback', JSON.stringify(persisted));

// Restores both actors at their persisted states
const restored = createActor(feedbackMachine, {
  snapshot: JSON.parse(localStorage.getItem('feedback')),
}).start();
```

## Partial Restoration (State Value Only)

Restore only the finite state value (and optionally context) using `machine.resolveState()`:

```ts
const savedValue = localStorage.getItem('someState'); // e.g. "pending"

const resolvedState = someMachine.resolveState({
  value: savedValue,
  // context: { ... } // optionally provide context
});

const restoredActor = createActor(someMachine, {
  snapshot: resolvedState,
}).start();
```

Useful when you only need to restore "where" the machine was, not the full internal state.

## Event Sourcing

Alternative to persisting state — replay events to reconstruct state:

```ts
const events: any[] = [];

const actor = createActor(someMachine, {
  inspect: (inspEvent) => {
    if (inspEvent.type === '@xstate.event') {
      // Only capture events for the root actor
      if (inspEvent.actorRef === actor) {
        events.push(inspEvent.event);
      }
    }
  },
});

actor.start();

// ... actor processes events ...

// Persist events
localStorage.setItem('events', JSON.stringify(events));

// Restore by replaying
const restoredActor = createActor(someMachine);
restoredActor.start();

for (const event of JSON.parse(localStorage.getItem('events'))) {
  restoredActor.send(event);
}
```

**Event sourcing advantages:**
- Less prone to incompatible state after machine logic changes
- Replays actions (persistence skips re-executing actions)
- Auditable event log

## Complete Pattern: Auto-Persist

```ts
import { createActor, type AnyActorLogic } from 'xstate';

function createPersistedActor<T extends AnyActorLogic>(
  logic: T,
  storageKey: string,
) {
  const saved = localStorage.getItem(storageKey);
  const snapshot = saved ? JSON.parse(saved) : undefined;

  const actor = createActor(logic, snapshot ? { snapshot } : {});

  actor.subscribe(() => {
    localStorage.setItem(
      storageKey,
      JSON.stringify(actor.getPersistedSnapshot()),
    );
  });

  return actor;
}

// Usage
const actor = createPersistedActor(myMachine, 'my-machine-state');
actor.start();
```

## Caveats

### Incompatible State

If machine logic changes between persist and restore, the restored state may be invalid:
- States that no longer exist
- Context shape changes
- New required states

**Mitigation:** Version your persisted state or use event sourcing.

### Serialization

Persisted state must be JSON-serializable. Cannot persist:
- Functions
- Class instances
- Circular references
- Symbols

### Actions Not Replayed

Entry/exit actions from the restored state are **not** re-executed. If you need side effects to re-run on restoration, use event sourcing or handle it in `actor.subscribe()` after start.
