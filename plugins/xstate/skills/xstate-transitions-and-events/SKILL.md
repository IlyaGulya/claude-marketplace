---
name: xstate-transitions-and-events
description: Covers XState v5 event handling, transition types, and guard patterns. Use when implementing guarded transitions, delayed (after) transitions, eventless (always) transitions, self-transitions, wildcard events, or composable guard logic with and/or/not.
---

# XState v5 Transitions and Events

## Events

Events are objects with a `type` string and optional payload:

```ts
// Sending events to an actor
actor.send({ type: 'feedback.submit', rating: 5, comment: 'Great!' });

// Typing events in setup
setup({
  types: {
    events: {} as
      | { type: 'feedback.submit'; rating: number; comment: string }
      | { type: 'feedback.cancel' }
      | { type: 'form.field.change'; field: string; value: string },
  },
});
```

Use **dot-notation** to namespace events. This enables partial wildcard matching (`'feedback.*'`).

## Basic Transitions

```ts
states: {
  idle: {
    on: {
      // Full object form (recommended)
      FETCH: { target: 'loading' },

      // With actions
      LOG: {
        target: 'active',
        actions: [{ type: 'logEvent' }],
      },

      // Shorthand (target only)
      ACTIVATE: 'active',
    },
  },
  loading: {},
  active: {},
}
```

## Self-Transitions

Transitions that stay in the same state — useful for updating context without changing state:

```ts
states: {
  editing: {
    on: {
      // No target = self-transition (does NOT re-enter)
      'field.change': {
        actions: assign({
          formData: ({ context, event }) => ({
            ...context.formData,
            [event.field]: event.value,
          }),
        }),
      },
      // With reenter: true = exit + re-enter (re-runs entry actions, restarts invocations)
      RESET: {
        target: 'editing',
        reenter: true,
      },
    },
  },
}
```

Default behavior (no `reenter`): does NOT exit/re-enter the state, does NOT restart invoked actors.

With `reenter: true`: exits the state, runs exit actions, stops invoked actors, re-enters, runs entry actions, starts new invocations.

## Target Resolution

```ts
states: {
  a: {
    on: {
      // Sibling target
      GO_B: { target: 'b' },

      // Sibling's child target
      GO_B_CHILD: { target: 'b.nested' },
    },
  },
  b: {
    initial: 'nested',
    states: { nested: {} },
  },
  c: { id: 'myC' },
}

// From parent to child (prefix with .)
on: {
  RESET: { target: '.a' },
}

// ID-based target (prefix with #) — works from anywhere
on: {
  GO_C: { target: '#myC' },
}
```

## Guards

Guards are pure synchronous functions that return `true` or `false`. A guarded transition is only taken if its guard passes.

### Named Guards (recommended)

```ts
const machine = setup({
  guards: {
    isValid: ({ context }) => context.feedback.length > 0,
    isUnderLimit: (_, params: { max: number }) => params.count < params.max,
  },
}).createMachine({
  on: {
    SUBMIT: {
      guard: 'isValid',
      target: 'submitting',
    },
    ADD: {
      guard: {
        type: 'isUnderLimit',
        params: ({ context }) => ({ max: 10, count: context.items.length }),
      },
      actions: 'addItem',
    },
  },
});
```

### Inline Guards

```ts
on: {
  SUBMIT: {
    guard: ({ context }) => context.feedback.length > 0,
    target: 'submitting',
  },
}
```

### Higher-Level Guards (and/or/not)

```ts
import { and, or, not } from 'xstate';

on: {
  SUBMIT: {
    guard: and(['isValid', 'isAuthenticated']),
    target: 'submitting',
  },
  DELETE: {
    guard: and(['isOwner', or(['isAdmin', not('isReadOnly')])]),
    target: 'deleting',
  },
}
```

### In-State Guards

Check if the machine is in a specific state (useful for parallel states):

```ts
import { stateIn } from 'xstate';

on: {
  SUBMIT: {
    guard: stateIn({ form: 'valid' }),
    target: 'submitting',
  },
}
```

## Multiple Guarded Transitions

Array form — first matching guard wins:

```ts
on: {
  SUBMIT: [
    // First: check if premium
    {
      guard: 'isPremiumUser',
      target: 'premiumSubmit',
    },
    // Second: check if valid
    {
      guard: 'isValid',
      target: 'standardSubmit',
    },
    // Default fallback (no guard)
    {
      target: 'validationError',
    },
  ],
}
```

## Delayed Transitions

Transitions triggered after a timeout. Timer auto-cancels if the state is exited.

### Inlined Delays

```ts
states: {
  waiting: {
    after: {
      5000: { target: 'timedOut' },  // 5 seconds
    },
    on: {
      RESPOND: { target: 'received' },
    },
  },
}
```

### Named Delays

```ts
const machine = setup({
  delays: {
    timeout: 5000,
  },
}).createMachine({
  states: {
    waiting: {
      after: {
        timeout: { target: 'timedOut' },
      },
    },
  },
});
```

### Dynamic Delays

```ts
const machine = setup({
  types: {
    context: {} as { attempts: number },
  },
  delays: {
    retryDelay: ({ context }) => context.attempts * 1000,  // Exponential backoff
  },
}).createMachine({
  states: {
    retrying: {
      after: {
        retryDelay: { target: 'fetching' },
      },
    },
  },
});
```

Delayed transition timers are **automatically cancelled** when the state is exited.

## Eventless (Always) Transitions

Transitions that are checked immediately after every transition. Must have a `guard` and/or `target` to avoid infinite loops.

```ts
states: {
  checking: {
    // Immediately routes based on context
    always: [
      { guard: 'hasItems', target: 'showItems' },
      { target: 'empty' },  // Default
    ],
  },
  showItems: {},
  empty: {},
}
```

Use `always` for conditional routing — when the next state depends on data, not an event:

```ts
states: {
  processing: {
    always: {
      guard: ({ context }) => context.temperature > 100,
      target: 'overheated',
    },
  },
}
```

**Warning:** Avoid unguarded `always` without a target — this causes infinite loops.

## Wildcard Transitions

### Full Wildcard

Matches any event not handled by a more specific transition:

```ts
states: {
  sleeping: {
    on: {
      '*': { target: 'awake' },  // Any event wakes up
    },
  },
}
```

### Partial Wildcard

Matches events with a specific prefix:

```ts
states: {
  prompt: {
    on: {
      'feedback.*': { target: 'form' },  // feedback, feedback.good, feedback.bad
    },
  },
}
```

Valid: `mouse.*`, `mouse.click.*`
Invalid: `mouse*`, `*.click`, `mouse.*.click`

## Forbidden Transitions

Prevent event handling to stop event bubbling to parent:

```ts
states: {
  locked: {
    on: {
      EDIT: {},  // Empty = forbidden. Stops bubbling to parent.
    },
  },
}
```

## Transition Selection Algorithm

1. Start at the **deepest active (atomic) state**
2. Check if it has an enabled transition for the event
3. If not, check the **parent state**, then grandparent, etc.
4. If no transition found anywhere, the event is ignored

```ts
const machine = createMachine({
  initial: 'parent',
  on: {
    GLOBAL: { actions: 'handleGlobal' },  // Checked last
  },
  states: {
    parent: {
      initial: 'child',
      on: {
        SHARED: { actions: 'parentHandles' },  // Checked second
      },
      states: {
        child: {
          on: {
            SHARED: { actions: 'childHandles' },  // Checked first — wins
          },
        },
      },
    },
  },
});
```

## Multiple Targets (Parallel States)

```ts
on: {
  'set.dark.custom': {
    target: ['.mode.dark', '.theme.custom'],  // Transition both regions
  },
}
```
