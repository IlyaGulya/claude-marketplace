---
name: xstate-typescript
description: Covers TypeScript patterns for type-safe XState v5 machines. Use when setting up typed machines with setup(), typing context/events/input/output, using type-bound helpers (v5.22+), assertEvent(), or type helpers like ActorRefFrom and SnapshotFrom. Requires TypeScript 5.0+.
---

# XState v5 TypeScript

## Prerequisites

- TypeScript **5.0+** (latest recommended)
- `strictNullChecks: true` in `tsconfig.json` (strongly recommended)
- `skipLibCheck: true` in `tsconfig.json` (recommended)

```json
{
  "compilerOptions": {
    "strictNullChecks": true,
    "skipLibCheck": true
  }
}
```

## The setup() Pattern

The `setup()` function is the primary way to create type-safe machines:

```ts
import { setup } from 'xstate';

const machine = setup({
  types: {
    context: {} as {
      userId: string;
      data: User | null;
      error: string | null;
    },
    events: {} as
      | { type: 'FETCH'; userId: string }
      | { type: 'RETRY' }
      | { type: 'RESET' },
    input: {} as {
      initialUserId: string;
    },
    output: {} as {
      result: User;
    },
    children: {} as {
      fetcher: 'fetchUser';
    },
    tags: {} as 'loading' | 'error',
  },
  actions: { /* ... */ },
  guards: { /* ... */ },
  actors: { /* ... */ },
  delays: { /* ... */ },
}).createMachine({
  // Everything is now fully typed
});
```

The `{} as Type` pattern is a TypeScript idiom for providing type information without runtime values.

## Typing Context and Events

```ts
const machine = setup({
  types: {
    context: {} as {
      count: number;
      items: string[];
      user: { name: string; email: string } | null;
    },
    events: {} as
      | { type: 'increment'; value: number }
      | { type: 'item.add'; item: string }
      | { type: 'item.remove'; index: number }
      | { type: 'user.set'; user: { name: string; email: string } },
  },
}).createMachine({
  context: {
    count: 0,
    items: [],
    user: null,
  },
  on: {
    increment: {
      actions: assign({
        count: ({ context, event }) => context.count + event.value,
        // event.value is typed as number
      }),
    },
    'item.add': {
      actions: assign({
        items: ({ context, event }) => [...context.items, event.item],
        // event.item is typed as string
      }),
    },
  },
});
```

## Typing Input and Output

For reusable machines that accept configuration and produce results:

```ts
const searchMachine = setup({
  types: {
    context: {} as {
      query: string;
      results: SearchResult[];
    },
    input: {} as {
      initialQuery: string;
      maxResults: number;
    },
    output: {} as {
      results: SearchResult[];
      totalCount: number;
    },
  },
}).createMachine({
  context: ({ input }) => ({
    query: input.initialQuery,   // typed
    results: [],
  }),
  // ...
  states: {
    done: {
      type: 'final',
    },
  },
  output: ({ context }) => ({
    results: context.results,    // typed
    totalCount: context.results.length,
  }),
});

// Usage — input is required and typed
const actor = createActor(searchMachine, {
  input: { initialQuery: 'xstate', maxResults: 10 },
});
```

## Typing Actions and Guards

### Parameterized Actions

```ts
const machine = setup({
  actions: {
    notify: (_, params: { message: string; level: 'info' | 'error' }) => {
      showNotification(params.message, params.level);
    },
  },
  guards: {
    isAboveThreshold: (_, params: { value: number; threshold: number }) => {
      return params.value > params.threshold;
    },
  },
}).createMachine({
  on: {
    SUCCESS: {
      actions: {
        type: 'notify',
        params: { message: 'Done!', level: 'info' }, // fully typed
      },
    },
    CHECK: {
      guard: {
        type: 'isAboveThreshold',
        params: ({ context }) => ({
          value: context.count,  // fully typed
          threshold: 100,
        }),
      },
    },
  },
});
```

## Type-Bound Helpers (v5.22+)

Create actions in separate files while maintaining full type safety:

```ts
// machineSetup.ts
import { setup } from 'xstate';

export const machineSetup = setup({
  types: {
    context: {} as { count: number; items: string[] },
    events: {} as
      | { type: 'increment' }
      | { type: 'addItem'; item: string }
      | { type: 'reset' },
    emitted: {} as { type: 'COUNT_CHANGED'; count: number },
  },
});

// actions.ts — fully typed, separate file
import { machineSetup } from './machineSetup';

export const incrementCount = machineSetup.assign({
  count: ({ context }) => context.count + 1,
  // context is fully typed
});

export const addItem = machineSetup.assign({
  items: ({ context, event }) => [...context.items, event.item],
  // event.item is typed as string
});

export const raiseReset = machineSetup.raise({ type: 'reset' });

export const emitChange = machineSetup.emit(({ context }) => ({
  type: 'COUNT_CHANGED',
  count: context.count,
}));

export const logState = machineSetup.createAction(({ context, event }) => {
  console.log(`Count: ${context.count}, Event: ${event.type}`);
});

// machine.ts
import { machineSetup } from './machineSetup';
import { incrementCount, addItem, logState } from './actions';

export const machine = machineSetup.createMachine({
  context: { count: 0, items: [] },
  initial: 'active',
  states: {
    active: {
      entry: logState,
      on: {
        increment: { actions: incrementCount },
        addItem: { actions: addItem },
      },
    },
  },
});
```

## Modular State Configs (v5.21+)

Split large machines across files with `createStateConfig()`:

```ts
// setup.ts
export const appSetup = setup({
  types: {
    context: {} as AppContext,
    events: {} as AppEvent,
  },
  actions: { /* ... */ },
  actors: { /* ... */ },
});

// states/editing.ts
import { appSetup } from '../setup';

export const editingState = appSetup.createStateConfig({
  entry: { type: 'loadDraft' },
  on: {
    SAVE: { target: 'saving', actions: { type: 'saveDraft' } },
    VALIDATE: { target: 'validating' },
  },
});

// machine.ts
import { appSetup } from './setup';
import { editingState } from './states/editing';

export const appMachine = appSetup.createMachine({
  initial: 'editing',
  states: {
    editing: editingState,
    validating: { /* ... */ },
    saving: { /* ... */ },
  },
});
```

## Type Helpers

### ActorRefFrom

Get a typed actor reference from actor logic:

```ts
import { type ActorRefFrom } from 'xstate';

type MyActorRef = ActorRefFrom<typeof myMachine>;

// Useful for typing props or context
interface Props {
  actorRef: ActorRefFrom<typeof formMachine>;
}
```

### SnapshotFrom

Get a typed snapshot from actor logic or actor ref:

```ts
import { type SnapshotFrom } from 'xstate';

type MySnapshot = SnapshotFrom<typeof myMachine>;

function renderState(snapshot: SnapshotFrom<typeof myMachine>) {
  snapshot.context; // fully typed
  snapshot.value;   // fully typed
}
```

### EventFromLogic

Get all event types from actor logic:

```ts
import { type EventFromLogic } from 'xstate';

type MyEvent = EventFromLogic<typeof myMachine>;
// Union of all event types
```

### OutputFrom

Get the output type from actor logic:

```ts
import { type OutputFrom } from 'xstate';

type MyOutput = OutputFrom<typeof myMachine>;
```

## assertEvent()

Narrow event types in actions/guards where the event type is a union:

```ts
import { assertEvent } from 'xstate';

const machine = setup({
  types: {
    events: {} as
      | { type: 'greet'; name: string }
      | { type: 'submit'; data: FormData }
      | { type: 'cancel' },
  },
}).createMachine({
  states: {
    greeting: {
      entry: ({ event }) => {
        // event.type is 'greet' | 'submit' | 'cancel'
        assertEvent(event, 'greet');
        // Now event is narrowed: { type: 'greet'; name: string }
        console.log(event.name.toUpperCase());
      },
    },
    processing: {
      exit: ({ event }) => {
        // Assert multiple types
        assertEvent(event, ['greet', 'submit']);
        // event is { type: 'greet'; name: string } | { type: 'submit'; data: FormData }
      },
    },
  },
});
```

**Prefer dynamic params over assertEvent** — params are more reusable:

```ts
// BETTER — action doesn't depend on machine's event types
actions: {
  greetUser: (_, params: { name: string }) => {
    console.log(`Hello, ${params.name}!`);
  },
}

// Use assertEvent only when params are not feasible
```

## Common Patterns

### Typing fromPromise

```ts
import { fromPromise } from 'xstate';

interface User { id: string; name: string }

// fromPromise<OutputType, InputType>
const fetchUser = fromPromise<User, { userId: string }>(async ({ input }) => {
  const res = await fetch(`/api/users/${input.userId}`);
  return res.json() as Promise<User>;
});

// Output and input are now fully typed in invoke
invoke: {
  src: 'fetchUser',
  input: ({ context }) => ({ userId: context.userId }), // typed
  onDone: {
    actions: assign({
      user: ({ event }) => event.output, // typed as User
    }),
  },
}
```

### Typing .provide()

```ts
// Original machine type is preserved
const testMachine = machine.provide({
  actions: {
    // Must match the original action's signature
    notify: (_, params: { message: string; level: 'info' | 'error' }) => {
      console.log(params.message);
    },
  },
  actors: {
    fetchUser: fromPromise<User, { userId: string }>(async ({ input }) => {
      return mockUser;
    }),
  },
});
```

### Context Factory with Input

```ts
const machine = setup({
  types: {
    context: {} as {
      userId: string;
      preferences: UserPrefs;
      isReady: boolean;
    },
    input: {} as {
      userId: string;
      preferences?: Partial<UserPrefs>;
    },
  },
}).createMachine({
  context: ({ input }) => ({
    userId: input.userId,
    preferences: {
      theme: 'light',
      language: 'en',
      ...input.preferences,
    },
    isReady: false,
  }),
});
```
