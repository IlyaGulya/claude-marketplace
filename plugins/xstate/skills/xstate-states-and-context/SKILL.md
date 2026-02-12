---
name: xstate-states-and-context
description: Covers XState v5 state types and context management. Use when implementing compound (parent/child) states, parallel states, history states, final states, or managing context with assign(). Includes hierarchy design, context initialization patterns, and state reading.
---

# XState v5 States and Context

## Machine Creation

```ts
import { setup, createActor } from 'xstate';

// Recommended: setup() for typed, reusable machines
const machine = setup({
  types: {
    context: {} as { count: number },
    events: {} as { type: 'increment' } | { type: 'reset' },
  },
  actions: { /* named implementations */ },
  guards: { /* named implementations */ },
  actors: { /* named implementations */ },
}).createMachine({
  id: 'counter',
  initial: 'active',
  context: { count: 0 },
  states: { active: {} },
});

// Create and start an actor
const actor = createActor(machine);
actor.subscribe((snapshot) => console.log(snapshot.value));
actor.start();
actor.send({ type: 'increment' });
```

## State Types

### Atomic States

Leaf states with no children — the simplest state type:

```ts
states: {
  idle: {},       // atomic
  loading: {},    // atomic
  success: {},    // atomic
}
```

### Compound (Parent) States

States containing child states. Must have an `initial` property:

```ts
states: {
  form: {
    initial: 'editing',
    states: {
      editing: {
        on: { VALIDATE: 'validating' },
      },
      validating: {
        on: {
          'validation.pass': 'valid',
          'validation.fail': 'invalid',
        },
      },
      valid: {},
      invalid: {
        on: { EDIT: 'editing' },
      },
    },
  },
}
```

### Parallel States

Independent regions that are all active simultaneously:

```ts
const machine = createMachine({
  type: 'parallel',
  states: {
    upload: {
      initial: 'idle',
      states: {
        idle: { on: { UPLOAD: 'uploading' } },
        uploading: { on: { COMPLETE: 'done' } },
        done: { type: 'final' },
      },
    },
    form: {
      initial: 'editing',
      states: {
        editing: { on: { SUBMIT: 'submitting' } },
        submitting: {},
        done: { type: 'final' },
      },
    },
  },
  // onDone fires when ALL parallel regions reach final
  onDone: 'allComplete',
});
```

### Final States

Terminal states that signal completion to the parent:

```ts
states: {
  processing: {
    initial: 'step1',
    states: {
      step1: { on: { NEXT: 'step2' } },
      step2: { on: { NEXT: 'complete' } },
      complete: { type: 'final' },
    },
    // Fires when 'complete' is reached
    onDone: { target: 'finished' },
  },
  finished: {},
}
```

Top-level `final` states cause the entire actor to complete, producing output:

```ts
const machine = createMachine({
  // ...
  states: {
    done: {
      type: 'final',
    },
  },
  output: ({ context }) => ({ result: context.data }),
});
```

### History States

Remember the last active child state:

```ts
states: {
  settings: {
    initial: 'general',
    states: {
      general: {},
      privacy: {},
      notifications: {},
      hist: { type: 'history', history: 'shallow' },
      // 'deep' remembers nested child states too
    },
    on: { BACK: '.hist' }, // Returns to last active child
  },
}
```

- `history: 'shallow'` — remembers the direct child state only
- `history: 'deep'` — remembers the full nested state hierarchy

## Parent/Child Patterns

### When to Nest States

Nest when child states:
- Share transitions (defined on parent, handled regardless of child)
- Share invoked actors (invoked on parent, active in all children)
- Have a natural lifecycle (enter parent → process → exit parent)

### Transition Selection (Deepest First)

Events are handled by the deepest matching state first, then bubble up:

```ts
const machine = createMachine({
  initial: 'parent',
  states: {
    parent: {
      initial: 'child',
      on: {
        EVENT: { actions: 'parentHandler' }, // Only if child doesn't handle it
      },
      states: {
        child: {
          on: {
            EVENT: { actions: 'childHandler' }, // Handles first
          },
        },
      },
    },
  },
});
```

### Parent onDone

The parent's `onDone` fires when a child reaches a `final` state:

```ts
states: {
  wizard: {
    initial: 'step1',
    states: {
      step1: { on: { NEXT: 'step2' } },
      step2: { on: { NEXT: 'done' } },
      done: { type: 'final' },
    },
    onDone: 'complete', // Transitions parent when child is final
  },
  complete: {},
}
```

### Modular State Configs (v5.21+)

Break large machines into separate files:

```ts
const machineSetup = setup({ /* types, actions, guards, actors */ });

// Can be in separate files
const editingState = machineSetup.createStateConfig({
  on: {
    VALIDATE: 'validating',
    SAVE: { actions: 'saveDraft' },
  },
});

const validatingState = machineSetup.createStateConfig({
  invoke: {
    src: 'validateForm',
    onDone: 'valid',
    onError: 'invalid',
  },
});

const machine = machineSetup.createMachine({
  initial: 'editing',
  states: { editing: editingState, validating: validatingState, valid: {}, invalid: {} },
});
```

## Context Management

### Static Initial Context

```ts
createMachine({
  context: {
    count: 0,
    items: [],
    user: null,
  },
});
```

### Lazy Initial Context

Evaluated per actor instance — good for timestamps, random IDs:

```ts
createMachine({
  context: () => ({
    id: crypto.randomUUID(),
    createdAt: Date.now(),
    items: [],
  }),
});
```

### Input-Based Context

Use `input` to parameterize machines (replaces factory functions):

```ts
const machine = setup({
  types: {
    context: {} as { userId: string; rating: number },
    input: {} as { userId: string; defaultRating: number },
  },
}).createMachine({
  context: ({ input }) => ({
    userId: input.userId,
    rating: input.defaultRating,
  }),
});

const actor = createActor(machine, {
  input: { userId: '123', defaultRating: 5 },
});
```

### Updating Context with assign()

Property assigners (preferred — partial update):

```ts
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

Function assigners (for dynamic/full updates):

```ts
on: {
  RESET: {
    actions: assign(({ context }) => ({
      ...context,
      count: 0,
      error: null,
    })),
  },
}
```

**Critical:** Never mutate context directly. Always return new values:

```ts
// BAD — mutation
actions: assign({
  items: ({ context, event }) => {
    context.items.push(event.item); // WRONG: mutates!
    return context.items;
  },
})

// GOOD — immutable
actions: assign({
  items: ({ context, event }) => [...context.items, event.item],
})
```

## Reading State

```ts
const actor = createActor(machine).start();
const snapshot = actor.getSnapshot();

// State value
snapshot.value;                    // 'idle' or { form: 'editing' }

// Context
snapshot.context;                  // { count: 0, ... }

// Matching states
snapshot.matches('loading');       // true/false
snapshot.matches({ form: 'editing' }); // nested match

// Tags (resilient to state name changes)
snapshot.hasTag('loading');        // true if any active state has tag

// Can an event cause a transition?
snapshot.can({ type: 'SUBMIT' }); // true/false (evaluates guards)

// Status
snapshot.status;                   // 'active' | 'done' | 'error' | 'stopped'

// Output (only when status === 'done')
snapshot.output;

// Child actors
snapshot.children;                 // { actorId: ActorRef }
```

Prefer `hasTag()` over `matches()` — tags survive refactoring:

```ts
states: {
  loading: { tags: ['busy'] },
  submitting: { tags: ['busy'] },
  refreshing: { tags: ['busy'] },
}
// snapshot.hasTag('busy') works for all three
```

## Anti-Patterns

### Booleans Instead of States

```ts
// BAD: impossible states possible (isLoading AND isError)
context: { isLoading: false, isError: false, data: null }

// GOOD: mutually exclusive by design
states: { idle: {}, loading: {}, success: {}, error: {} }
```

### Unnecessary Nesting

```ts
// BAD: single-child compound state adds no value
states: {
  wrapper: {
    initial: 'only',
    states: { only: {} },
  },
}

// GOOD: just use an atomic state
states: { only: {} }
```

### Context Mutation

```ts
// BAD: direct mutation creates shared reference bugs
assign({ items: ({ context }) => { context.items.push(x); return context.items; } })

// GOOD: always return new reference
assign({ items: ({ context }) => [...context.items, x] })
```
