---
name: xstate-machine-modeling
description: Teaches how to design and model XState v5 state machines from scratch using a systematic process. Use when starting a new state machine, deciding what should be a state vs context, choosing between actions and actors, or structuring events and states for a feature.
---

# XState v5 Machine Modeling

## The Modeling Process

Follow these 5 steps to build a state machine from scratch:

### Step 1: List Events

List all events your machine cares about — things that happen from the outside:

```
- User clicks "Submit"
- User changes input value
- API returns data
- Timer expires
- WebSocket message received
```

Think in sequences: `User changes input` → `User submits form` → `API responds`.

### Step 2: List Tasks (Side Effects)

List everything your machine needs to *do*:

```
- Validate form fields
- Send data to API
- Show notification
- Subscribe to WebSocket
- Focus an input
```

### Step 3: Divide Tasks into Actions vs Actors

**Actions** — fire-and-forget. Use when you do NOT care about the result:
- Log analytics event
- Update context
- Focus an input
- Show a toast notification

**Actors** — long-running or result-dependent. Use when you need to:
- Wait for a response (Promise)
- Handle success AND failure
- Clean up on exit (subscriptions)
- Communicate bidirectionally

Decision rule: **"Do I need to react to the outcome?"** → Yes = Actor, No = Action.

### Step 4: Define the Initial State

Ask: "What is the machine *doing* before anything happens?" That's your initial state.

- A form → `editing`
- A data fetcher → `idle`
- An auth flow → `unauthenticated`

### Step 5: Build States Iteratively

For each event, ask: "In which state can this happen, and where does it lead?"

```
idle --FETCH--> loading
loading --onDone--> success
loading --onError--> failure
failure --RETRY--> loading
```

## Decision: State vs Context

Use **finite states** when:
- The value is mutually exclusive (loading OR success OR error — never two at once)
- The value changes which events are possible (can only RETRY from `failure`)
- The value changes *behavior* (entry actions, invoked actors differ per state)

Use **context** when:
- The value is quantitative (count, name, list items)
- The value doesn't gate which events are possible
- The value is continuous (any string, any number)

```ts
// GOOD: States for mutually exclusive modes
const machine = setup({}).createMachine({
  initial: 'idle',
  context: { data: null, error: null }, // quantitative data in context
  states: {
    idle: { on: { FETCH: 'loading' } },
    loading: {
      invoke: {
        src: 'fetchData',
        onDone: { target: 'success', actions: assign({ data: ({ event }) => event.output }) },
        onError: { target: 'failure', actions: assign({ error: ({ event }) => event.error }) },
      },
    },
    success: {},
    failure: { on: { RETRY: 'loading' } },
  },
});

// BAD: Boolean flags in context instead of states
const machine = createMachine({
  context: { isLoading: false, isError: false, data: null },
  // Now you must manually check flags everywhere
});
```

**Rule of thumb:** If you find yourself writing `if (context.someFlag)` to decide behavior, it should probably be a state.

## Decision: Action vs Actor

| Criteria | Action | Actor |
|----------|--------|-------|
| Needs result? | No | Yes |
| Needs error handling? | No | Yes (`onError`) |
| Needs cleanup? | No | Yes (stopped on exit) |
| Duration | Instant | Over time |
| Communication | One-way | Bidirectional |

```ts
// Action: fire-and-forget logging
const machine = setup({
  actions: {
    logAnalytics: (_, params: { event: string }) => {
      analytics.track(params.event);
    },
  },
}).createMachine({
  states: {
    active: {
      entry: { type: 'logAnalytics', params: { event: 'page_viewed' } },
    },
  },
});

// Actor: need result + error handling
const machine = setup({
  actors: {
    fetchUser: fromPromise(async ({ input }: { input: { id: string } }) => {
      const res = await fetch(`/api/users/${input.id}`);
      if (!res.ok) throw new Error('Failed');
      return res.json();
    }),
  },
}).createMachine({
  states: {
    loading: {
      invoke: {
        src: 'fetchUser',
        input: ({ context }) => ({ id: context.userId }),
        onDone: { target: 'success', actions: assign({ user: ({ event }) => event.output }) },
        onError: { target: 'error', actions: assign({ error: ({ event }) => event.error }) },
      },
    },
  },
});
```

## Event Design

Use **dot-notation namespacing** to group related events:

```ts
setup({
  types: {
    events: {} as
      | { type: 'form.submit' }
      | { type: 'form.reset' }
      | { type: 'form.field.change'; field: string; value: string }
      | { type: 'auth.login'; credentials: { email: string; password: string } }
      | { type: 'auth.logout' },
  },
});
```

Benefits:
- Wildcard transitions: `'form.*'` matches all form events
- Self-documenting event hierarchy
- Easy to filter in devtools

Keep payloads minimal — include only data needed to process the event.

## State Naming

Name states by what the machine is **doing**, not what happened:

```ts
// GOOD: describes current activity
states: {
  idle: {},
  loading: {},
  editing: {},
  submitting: {},
  validating: {},
}

// BAD: describes past event
states: {
  submitted: {},  // What is the machine doing NOW?
  loaded: {},     // Is it showing data? Waiting?
}
```

Exception: `success` and `failure` are acceptable terminal state names.

## Anti-Patterns

### Boolean Flags Instead of States

```ts
// BAD
context: { isLoading: false, isError: false, isSuccess: false }
// Can be isLoading AND isError — impossible states are possible!

// GOOD
states: { idle: {}, loading: {}, success: {}, error: {} }
// Only one at a time, guaranteed.
```

### Missing Error States

```ts
// BAD: no error handling
loading: {
  invoke: { src: 'fetchData', onDone: 'success' },
  // What happens on error? Machine gets stuck.
}

// GOOD: always handle errors
loading: {
  invoke: {
    src: 'fetchData',
    onDone: { target: 'success' },
    onError: { target: 'error' },
  },
}
```

### Over-Modeling

Not everything needs a state machine. Simple derived values or one-off toggles don't benefit from a full machine. Use XState when you have:
- Multiple states with different behaviors
- Complex event sequences
- Async operations with error handling
- State that needs to be predictable and testable

## Complete Example: Data Fetcher

Modeled step-by-step following the process above:

```ts
import { setup, assign, fromPromise, createActor } from 'xstate';

// Step 1: Events — FETCH, RETRY, RESET
// Step 2: Tasks — fetch data from API, show error
// Step 3: fetch = actor (need result), show error = action
// Step 4: Initial state = idle
// Step 5: idle->loading->success/failure, failure->loading (retry)

const fetchMachine = setup({
  types: {
    context: {} as {
      data: unknown | null;
      error: unknown | null;
      retries: number;
    },
    events: {} as
      | { type: 'FETCH'; url: string }
      | { type: 'RETRY' }
      | { type: 'RESET' },
  },
  actors: {
    fetchData: fromPromise(async ({ input }: { input: { url: string } }) => {
      const res = await fetch(input.url);
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      return res.json();
    }),
  },
  actions: {
    logError: (_, params: { error: unknown }) => {
      console.error('Fetch failed:', params.error);
    },
  },
  guards: {
    canRetry: ({ context }) => context.retries < 3,
  },
}).createMachine({
  id: 'fetcher',
  initial: 'idle',
  context: { data: null, error: null, retries: 0 },
  states: {
    idle: {
      on: { FETCH: 'loading' },
    },
    loading: {
      invoke: {
        src: 'fetchData',
        input: ({ event }) => ({ url: event.url }),
        onDone: {
          target: 'success',
          actions: assign({ data: ({ event }) => event.output, error: null }),
        },
        onError: {
          target: 'failure',
          actions: [
            assign({
              error: ({ event }) => event.error,
              retries: ({ context }) => context.retries + 1,
            }),
            {
              type: 'logError',
              params: ({ event }) => ({ error: event.error }),
            },
          ],
        },
      },
    },
    success: {
      on: { RESET: 'idle' },
    },
    failure: {
      on: {
        RETRY: { guard: 'canRetry', target: 'loading' },
        RESET: {
          target: 'idle',
          actions: assign({ retries: 0, error: null }),
        },
      },
    },
  },
});
```
