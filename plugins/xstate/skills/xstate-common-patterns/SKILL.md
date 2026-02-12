---
name: xstate-common-patterns
description: Ready-to-use XState v5 patterns for common scenarios. Use when implementing data fetching with loading/error/retry, form validation, authentication flows, debounce/throttle, retry with backoff, persistence, or CRUD with dynamic actors. Each pattern is a complete machine.
---

# XState v5 Common Patterns

## Data Fetching

The most common pattern: idle → loading → success/failure with retry.

```ts
import { setup, assign, fromPromise } from 'xstate';

const fetchMachine = setup({
  types: {
    context: {} as {
      data: unknown | null;
      error: unknown | null;
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
}).createMachine({
  id: 'fetch',
  initial: 'idle',
  context: { data: null, error: null },
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
          actions: assign({ error: ({ event }) => event.error }),
        },
      },
    },
    success: {
      on: { FETCH: 'loading', RESET: 'idle' },
    },
    failure: {
      on: {
        RETRY: 'loading',
        RESET: { target: 'idle', actions: assign({ error: null }) },
      },
    },
  },
});
```

## Retry with Exponential Backoff

```ts
const retryMachine = setup({
  types: {
    context: {} as {
      data: unknown | null;
      error: unknown | null;
      attempts: number;
      maxAttempts: number;
    },
  },
  actors: {
    fetchData: fromPromise(async ({ input }: { input: { url: string } }) => {
      const res = await fetch(input.url);
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      return res.json();
    }),
  },
  guards: {
    canRetry: ({ context }) => context.attempts < context.maxAttempts,
    maxRetriesReached: ({ context }) => context.attempts >= context.maxAttempts,
  },
  delays: {
    retryDelay: ({ context }) => Math.min(1000 * Math.pow(2, context.attempts), 30000),
  },
}).createMachine({
  id: 'retry',
  initial: 'fetching',
  context: { data: null, error: null, attempts: 0, maxAttempts: 3 },
  states: {
    fetching: {
      invoke: {
        src: 'fetchData',
        input: { url: '/api/data' },
        onDone: {
          target: 'success',
          actions: assign({ data: ({ event }) => event.output }),
        },
        onError: {
          target: 'retryCheck',
          actions: assign({
            error: ({ event }) => event.error,
            attempts: ({ context }) => context.attempts + 1,
          }),
        },
      },
    },
    retryCheck: {
      always: [
        { guard: 'canRetry', target: 'waiting' },
        { guard: 'maxRetriesReached', target: 'failure' },
      ],
    },
    waiting: {
      after: {
        retryDelay: { target: 'fetching' },
      },
    },
    success: { type: 'final' },
    failure: { type: 'final' },
  },
});
```

## Form Validation + Submission

```ts
const formMachine = setup({
  types: {
    context: {} as {
      fields: Record<string, string>;
      errors: Record<string, string>;
      submitError: string | null;
    },
    events: {} as
      | { type: 'field.change'; field: string; value: string }
      | { type: 'SUBMIT' }
      | { type: 'RESET' },
  },
  actors: {
    submitForm: fromPromise(async ({ input }: { input: { fields: Record<string, string> } }) => {
      const res = await fetch('/api/submit', {
        method: 'POST',
        body: JSON.stringify(input.fields),
        headers: { 'Content-Type': 'application/json' },
      });
      if (!res.ok) throw new Error('Submit failed');
      return res.json();
    }),
  },
  guards: {
    isValid: ({ context }) => Object.keys(context.errors).length === 0,
  },
  actions: {
    validate: assign(({ context }) => {
      const errors: Record<string, string> = {};
      if (!context.fields.name) errors.name = 'Name is required';
      if (!context.fields.email?.includes('@')) errors.email = 'Invalid email';
      return { errors };
    }),
  },
}).createMachine({
  id: 'form',
  initial: 'editing',
  context: { fields: {}, errors: {}, submitError: null },
  states: {
    editing: {
      on: {
        'field.change': {
          actions: [
            assign({
              fields: ({ context, event }) => ({
                ...context.fields,
                [event.field]: event.value,
              }),
            }),
            { type: 'validate' },
          ],
        },
        SUBMIT: 'validating',
      },
    },
    validating: {
      entry: { type: 'validate' },
      always: [
        { guard: 'isValid', target: 'submitting' },
        { target: 'editing' },
      ],
    },
    submitting: {
      invoke: {
        src: 'submitForm',
        input: ({ context }) => ({ fields: context.fields }),
        onDone: { target: 'success' },
        onError: {
          target: 'editing',
          actions: assign({ submitError: ({ event }) => event.error?.message ?? 'Unknown error' }),
        },
      },
    },
    success: {
      on: { RESET: { target: 'editing', actions: assign({ fields: {}, errors: {}, submitError: null }) } },
    },
  },
});
```

## Authentication Flow

```ts
const authMachine = setup({
  types: {
    context: {} as {
      token: string | null;
      user: { id: string; name: string } | null;
      error: string | null;
    },
    events: {} as
      | { type: 'LOGIN'; email: string; password: string }
      | { type: 'LOGOUT' }
      | { type: 'TOKEN_EXPIRED' },
  },
  actors: {
    authenticate: fromPromise(async ({ input }: { input: { email: string; password: string } }) => {
      const res = await fetch('/api/auth/login', {
        method: 'POST',
        body: JSON.stringify(input),
        headers: { 'Content-Type': 'application/json' },
      });
      if (!res.ok) throw new Error('Invalid credentials');
      return res.json() as Promise<{ token: string; user: { id: string; name: string } }>;
    }),
    refreshToken: fromPromise(async ({ input }: { input: { token: string } }) => {
      const res = await fetch('/api/auth/refresh', {
        method: 'POST',
        headers: { Authorization: `Bearer ${input.token}` },
      });
      if (!res.ok) throw new Error('Refresh failed');
      return res.json() as Promise<{ token: string }>;
    }),
  },
}).createMachine({
  id: 'auth',
  initial: 'unauthenticated',
  context: { token: null, user: null, error: null },
  states: {
    unauthenticated: {
      entry: assign({ token: null, user: null }),
      on: { LOGIN: 'authenticating' },
    },
    authenticating: {
      invoke: {
        src: 'authenticate',
        input: ({ event }) => ({ email: event.email, password: event.password }),
        onDone: {
          target: 'authenticated',
          actions: assign({
            token: ({ event }) => event.output.token,
            user: ({ event }) => event.output.user,
            error: null,
          }),
        },
        onError: {
          target: 'unauthenticated',
          actions: assign({
            error: ({ event }) => event.error?.message ?? 'Login failed',
          }),
        },
      },
    },
    authenticated: {
      on: {
        LOGOUT: 'unauthenticated',
        TOKEN_EXPIRED: 'refreshing',
      },
    },
    refreshing: {
      invoke: {
        src: 'refreshToken',
        input: ({ context }) => ({ token: context.token! }),
        onDone: {
          target: 'authenticated',
          actions: assign({ token: ({ event }) => event.output.token }),
        },
        onError: 'unauthenticated',
      },
    },
  },
});
```

## Debounce

Self-transition resets the delay timer:

```ts
const debounceMachine = setup({
  types: {
    context: {} as { query: string },
    events: {} as { type: 'INPUT'; value: string },
  },
  delays: {
    debounceDelay: 300,
  },
  actors: {
    search: fromPromise(async ({ input }: { input: { query: string } }) => {
      return fetch(`/api/search?q=${input.query}`).then(r => r.json());
    }),
  },
}).createMachine({
  id: 'debounce',
  initial: 'idle',
  context: { query: '' },
  states: {
    idle: {
      on: {
        INPUT: {
          target: 'debouncing',
          actions: assign({ query: ({ event }) => event.value }),
        },
      },
    },
    debouncing: {
      on: {
        INPUT: {
          // Self-transition resets the after timer
          target: 'debouncing',
          reenter: true,
          actions: assign({ query: ({ event }) => event.value }),
        },
      },
      after: {
        debounceDelay: 'searching',
      },
    },
    searching: {
      invoke: {
        src: 'search',
        input: ({ context }) => ({ query: context.query }),
        onDone: { target: 'idle' },
        onError: { target: 'idle' },
      },
      on: {
        INPUT: {
          target: 'debouncing',
          actions: assign({ query: ({ event }) => event.value }),
        },
      },
    },
  },
});
```

## CRUD with Dynamic Actors

Use spawned actors for each entity:

```ts
import { setup, assign, spawnChild, stopChild, sendTo, fromPromise } from 'xstate';

const todoMachine = setup({
  types: {
    context: {} as { id: string; text: string; completed: boolean },
    input: {} as { id: string; text: string },
    events: {} as { type: 'TOGGLE' } | { type: 'UPDATE'; text: string },
  },
}).createMachine({
  context: ({ input }) => ({ id: input.id, text: input.text, completed: false }),
  on: {
    TOGGLE: { actions: assign({ completed: ({ context }) => !context.completed }) },
    UPDATE: { actions: assign({ text: ({ event }) => event.text }) },
  },
});

const todoListMachine = setup({
  types: {
    events: {} as
      | { type: 'todo.add'; id: string; text: string }
      | { type: 'todo.remove'; id: string }
      | { type: 'todo.toggle'; id: string },
  },
  actors: { todo: todoMachine },
}).createMachine({
  on: {
    'todo.add': {
      actions: spawnChild('todo', {
        id: ({ event }) => `todo-${event.id}`,
        input: ({ event }) => ({ id: event.id, text: event.text }),
      }),
    },
    'todo.remove': {
      actions: stopChild(({ event }) => `todo-${event.id}`),
    },
    'todo.toggle': {
      actions: sendTo(
        ({ event }) => `todo-${event.id}`,
        { type: 'TOGGLE' },
      ),
    },
  },
});
```

## Persistence

Save and restore machine state across sessions:

```ts
import { createActor } from 'xstate';

// Save state
const actor = createActor(myMachine).start();

actor.subscribe(() => {
  const persistedState = actor.getPersistedSnapshot();
  localStorage.setItem('app-state', JSON.stringify(persistedState));
});

// Restore state
const savedState = JSON.parse(localStorage.getItem('app-state') ?? 'null');

const restoredActor = createActor(myMachine, {
  snapshot: savedState ?? undefined,
}).start();
// Actor resumes from saved state. Actions are NOT re-executed.
// Invoked actors ARE restarted. Spawned actors are restored recursively.
```

### Restore from State Value Only

```ts
// If you only saved the state value (not full snapshot)
const savedValue = localStorage.getItem('state-value'); // e.g., "editing"

const resolvedState = myMachine.resolveState({
  value: savedValue,
  // context: { ... } // optionally restore context too
});

const actor = createActor(myMachine, {
  snapshot: resolvedState,
}).start();
```

## React Integration

### useActor / useMachine

```tsx
import { useMachine } from '@xstate/react';

function FetchComponent() {
  const [snapshot, send] = useMachine(fetchMachine);

  if (snapshot.matches('idle')) {
    return <button onClick={() => send({ type: 'FETCH', url: '/api/data' })}>Fetch</button>;
  }
  if (snapshot.matches('loading')) {
    return <div>Loading...</div>;
  }
  if (snapshot.matches('success')) {
    return <div>{JSON.stringify(snapshot.context.data)}</div>;
  }
  if (snapshot.matches('failure')) {
    return (
      <div>
        <p>Error: {String(snapshot.context.error)}</p>
        <button onClick={() => send({ type: 'RETRY' })}>Retry</button>
      </div>
    );
  }
  return null;
}
```

### useSelector (optimized re-renders)

```tsx
import { useActorRef, useSelector } from '@xstate/react';

const selectCount = (snapshot: SnapshotFrom<typeof counterMachine>) => snapshot.context.count;

function Counter() {
  const actorRef = useActorRef(counterMachine);
  const count = useSelector(actorRef, selectCount);

  return <button onClick={() => actorRef.send({ type: 'increment' })}>Count: {count}</button>;
}
```

### createActorContext (global state)

```tsx
import { createActorContext } from '@xstate/react';

const AppContext = createActorContext(appMachine);

// Provider wraps the app
function App() {
  return (
    <AppContext.Provider>
      <Child />
    </AppContext.Provider>
  );
}

// Consume anywhere in the tree
function Child() {
  const user = AppContext.useSelector((s) => s.context.user);
  const actorRef = AppContext.useActorRef();
  return <button onClick={() => actorRef.send({ type: 'LOGOUT' })}>{user?.name}</button>;
}
```
