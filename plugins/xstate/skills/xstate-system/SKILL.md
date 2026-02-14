---
name: xstate-system
description: |
  Covers XState v5 actor systems for cross-actor communication.
  Use when registering actors with systemId, addressing actors globally via system.get(), designing system-wide communication, or managing actor hierarchies beyond parent-child.
---

# XState v5 Actor Systems

An actor system is the collection of all actors created from a root actor. It enables cross-actor communication without direct parent-child references.

## System Basics

Every actor has access to its system:

```ts
import { createMachine, createActor } from 'xstate';

const machine = createMachine({
  entry: ({ system }) => {
    // Access the actor system
    console.log(system);
  },
});

const actor = createActor(machine).start();
actor.system; // The actor system
```

### Root Actor systemId

Optionally name the root actor:

```ts
const actor = createActor(machine, {
  systemId: 'root',
});
actor.start();
```

## Registering Actors

### Invoked Actors

Register with `systemId` in the `invoke` config:

```ts
const notifierMachine = createMachine({
  on: {
    notify: {
      actions: ({ event }) => console.log('Notification:', event.message),
    },
  },
});

const appMachine = createMachine({
  invoke: {
    src: notifierMachine,
    systemId: 'notifier', // Globally addressable
  },
  // ...
});
```

### Spawned Actors

Register with `systemId` in the `spawn` options:

```ts
const todosMachine = createMachine({
  on: {
    'todo.add': {
      actions: assign({
        todos: ({ context, spawn }) => {
          const newTodo = spawn(todoMachine, {
            systemId: 'todo-' + context.todos.length,
          });
          return [...context.todos, newTodo];
        },
      }),
    },
  },
});
```

## Cross-Actor Communication

### system.get(systemId)

Any actor can send events to any registered actor:

```ts
import { sendTo } from 'xstate';

const formMachine = createMachine({
  on: {
    submit: {
      actions: sendTo(
        ({ system }) => system.get('notifier'),
        { type: 'notify', message: 'Form submitted!' },
      ),
    },
  },
});
```

### Pattern: Global Logger

```ts
// Logger machine — registered with systemId
const loggerMachine = createMachine({
  context: { logs: [] as string[] },
  on: {
    LOG: {
      actions: assign({
        logs: ({ context, event }) => [
          ...context.logs,
          event.message,
        ],
      }),
    },
  },
});

// Root machine — registers the logger
const rootMachine = createMachine({
  invoke: {
    src: loggerMachine,
    systemId: 'logger',
  },
  // ...
});

// Any child/grandchild can log
const childMachine = createMachine({
  entry: sendTo(
    ({ system }) => system.get('logger'),
    { type: 'LOG', message: 'Child started' },
  ),
});
```

### Pattern: Event Bus

```ts
const eventBusMachine = createMachine({
  on: {
    'bus.publish': {
      actions: ({ event, system }) => {
        // Relay to all registered listeners
        for (const id of event.targets) {
          const ref = system.get(id);
          if (ref) {
            ref.send({ type: event.eventType, data: event.data });
          }
        }
      },
    },
  },
});

const appMachine = createMachine({
  invoke: {
    src: eventBusMachine,
    systemId: 'eventBus',
  },
});
```

## System Lifecycle

- The system is implicitly created from the root actor
- Stopping the root actor (`actor.stop()`) stops the entire system
- Descendant actors **cannot** stop the system — a warning is logged if attempted
- Registered actors are available via `system.get()` as long as they're running

## When to Use Systems vs Direct Communication

| Pattern | Use When |
|---------|----------|
| `sendTo(childId)` | Parent → direct child |
| `sendTo(parentRef)` | Child → parent (via input ref) |
| `system.get(id)` | Cross-branch communication (sibling actors, distant relatives) |
| `system.get(id)` | Global services (logger, auth, notifications) |

**Rule of thumb:** Use `systemId` when actors need to communicate but don't have a direct parent-child relationship.

## Anti-Patterns

### Missing Actor Check

```ts
// BAD — system.get() may return undefined if actor hasn't started yet
actions: sendTo(
  ({ system }) => system.get('notifier'),
  { type: 'NOTIFY' },
),
// Throws if 'notifier' isn't registered

// GOOD — guard against missing actors
actions: enqueueActions(({ system, enqueue }) => {
  const notifier = system.get('notifier');
  if (notifier) {
    enqueue(sendTo(notifier, { type: 'NOTIFY' }));
  }
}),
```

### Overusing Systems

```ts
// BAD — using system for simple parent-child
// Just use invoke with onDone/onError instead
invoke: { src: childMachine, systemId: 'child' }
// Then: system.get('child')

// GOOD — systems are for non-hierarchical communication
// For parent-child, use invoke/spawn + sendTo directly
```
