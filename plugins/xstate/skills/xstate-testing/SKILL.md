---
name: xstate-testing
description: Covers testing strategies for XState v5 state machines. Use when writing unit tests for machines, testing transitions, mocking with provide(), using the pure transition() function, or testing actors with async operations. Includes patterns for Vitest/Jest.
---

# XState v5 Testing

## Philosophy

Test **behavior** (transitions and context changes), not config structure. Tests should answer: "When the machine is in state X and receives event Y, does it go to state Z with the correct context?"

Follow the **Arrange, Act, Assert** pattern.

## Actor Testing

Create an actor, send events, assert the snapshot:

```ts
import { createActor } from 'xstate';
import { describe, test, expect } from 'vitest';
import { toggleMachine } from './toggleMachine';

describe('toggleMachine', () => {
  test('toggles between inactive and active', () => {
    // Arrange
    const actor = createActor(toggleMachine);
    actor.start();

    // Assert initial state
    expect(actor.getSnapshot().value).toBe('inactive');

    // Act
    actor.send({ type: 'TOGGLE' });

    // Assert
    expect(actor.getSnapshot().value).toBe('active');

    // Act again
    actor.send({ type: 'TOGGLE' });
    expect(actor.getSnapshot().value).toBe('inactive');
  });

  test('updates context on increment', () => {
    const actor = createActor(counterMachine);
    actor.start();

    actor.send({ type: 'increment', value: 5 });
    actor.send({ type: 'increment', value: 3 });

    expect(actor.getSnapshot().context.count).toBe(8);
  });
});
```

## Pure Transition Testing (v5.19+)

Test state transitions **without creating an actor** using the pure `transition()` and `initialTransition()` functions. This is the fastest way to test:

```ts
import { transition, initialTransition } from 'xstate';
import { describe, test, expect } from 'vitest';
import { fetchMachine } from './fetchMachine';

describe('fetchMachine transitions', () => {
  test('initial state is idle', () => {
    const [state, actions] = initialTransition(fetchMachine);

    expect(state.value).toBe('idle');
    expect(actions).toEqual([]);
  });

  test('FETCH transitions from idle to loading', () => {
    const [initialState] = initialTransition(fetchMachine);
    const [nextState, actions] = transition(
      fetchMachine,
      initialState,
      { type: 'FETCH', url: '/api/data' },
    );

    expect(nextState.value).toBe('loading');
  });

  test('returns action objects for named actions', () => {
    const [initialState] = initialTransition(fetchMachine);
    const [nextState, actions] = transition(
      fetchMachine,
      initialState,
      { type: 'FETCH', url: '/api/data' },
    );

    // Actions are returned as objects, not executed
    expect(actions).toContainEqual(
      expect.objectContaining({ type: 'logFetch' }),
    );
  });

  test('does not transition on unknown events', () => {
    const [initialState] = initialTransition(fetchMachine);
    const [nextState] = transition(
      fetchMachine,
      initialState,
      { type: 'UNKNOWN' },
    );

    expect(nextState.value).toBe('idle');
  });
});
```

Benefits of pure transition testing:
- No actor creation overhead
- Synchronous — no need for `await`
- Returns action objects for inspection
- Tests the machine logic in isolation

## Mocking with .provide()

Override actions, actors, and guards for testing:

```ts
import { createActor } from 'xstate';
import { vi, test, expect } from 'vitest';
import { notificationMachine } from './notificationMachine';

test('calls notify action on success', () => {
  const mockNotify = vi.fn();

  // Provide mock implementations
  const testMachine = notificationMachine.provide({
    actions: {
      notify: mockNotify,
    },
    actors: {
      fetchData: fromPromise(async () => ({ result: 'mocked' })),
    },
    guards: {
      isValid: () => true, // Force guard to pass
    },
  });

  const actor = createActor(testMachine);
  actor.start();

  actor.send({ type: 'SUBMIT' });

  expect(mockNotify).toHaveBeenCalled();
});
```

## Async Testing

### waitFor

Wait for an actor to reach a specific state:

```ts
import { createActor, waitFor } from 'xstate';
import { test, expect } from 'vitest';

test('eventually reaches success state', async () => {
  const testMachine = fetchMachine.provide({
    actors: {
      fetchData: fromPromise(async () => ({ data: 'test' })),
    },
  });

  const actor = createActor(testMachine);
  actor.start();

  actor.send({ type: 'FETCH' });

  const snapshot = await waitFor(
    actor,
    (snap) => snap.matches('success'),
    { timeout: 5000 },
  );

  expect(snapshot.context.data).toEqual({ data: 'test' });
});
```

### toPromise

Wait for an actor to complete (reach final state):

```ts
import { createActor, toPromise } from 'xstate';

test('produces correct output', async () => {
  const testMachine = processMachine.provide({
    actors: {
      processData: fromPromise(async () => 42),
    },
  });

  const actor = createActor(testMachine);
  actor.start();

  actor.send({ type: 'START' });

  const output = await toPromise(actor);
  expect(output).toEqual({ result: 42 });
});
```

### Promise Resolution

For promise actors, await microtask resolution:

```ts
test('resolves promise actor', async () => {
  const mockFetch = vi.fn().mockResolvedValue({ data: 'test' });

  const testMachine = machine.provide({
    actors: {
      fetchData: fromPromise(mockFetch),
    },
  });

  const actor = createActor(testMachine);
  actor.start();

  actor.send({ type: 'FETCH' });

  // Wait for promise to resolve
  await waitFor(actor, (snap) => snap.matches('success'));

  expect(actor.getSnapshot().context.data).toEqual({ data: 'test' });
  expect(mockFetch).toHaveBeenCalledOnce();
});
```

## Testing Patterns

### Guard Testing

```ts
test('guarded transition is blocked when guard fails', () => {
  const testMachine = machine.provide({
    guards: {
      isValid: () => false, // Force guard to fail
    },
  });

  const actor = createActor(testMachine);
  actor.start();

  actor.send({ type: 'SUBMIT' });

  // Should NOT transition because guard failed
  expect(actor.getSnapshot().value).toBe('editing');
});

test('guarded transition succeeds when guard passes', () => {
  const testMachine = machine.provide({
    guards: {
      isValid: () => true,
    },
  });

  const actor = createActor(testMachine);
  actor.start();

  actor.send({ type: 'SUBMIT' });

  expect(actor.getSnapshot().value).toBe('submitting');
});
```

### Entry/Exit Verification

```ts
test('runs entry action when entering state', () => {
  const entryFn = vi.fn();

  const testMachine = machine.provide({
    actions: {
      onEnterActive: entryFn,
    },
  });

  const actor = createActor(testMachine);
  actor.start();

  actor.send({ type: 'ACTIVATE' });

  expect(entryFn).toHaveBeenCalledOnce();
});
```

### Error Path Testing

```ts
test('handles fetch error', async () => {
  const testMachine = fetchMachine.provide({
    actors: {
      fetchData: fromPromise(async () => {
        throw new Error('Network error');
      }),
    },
  });

  const actor = createActor(testMachine);
  actor.start();

  actor.send({ type: 'FETCH' });

  await waitFor(actor, (snap) => snap.matches('error'));

  expect(actor.getSnapshot().context.error).toBeInstanceOf(Error);
  expect(actor.getSnapshot().context.error.message).toBe('Network error');
});
```

### Delayed Transition Testing

For testing `after` transitions, mock the delay:

```ts
import { vi, test, expect, beforeEach, afterEach } from 'vitest';

beforeEach(() => {
  vi.useFakeTimers();
});

afterEach(() => {
  vi.useRealTimers();
});

test('times out after delay', () => {
  const actor = createActor(timeoutMachine);
  actor.start();

  expect(actor.getSnapshot().value).toBe('waiting');

  vi.advanceTimersByTime(5000);

  expect(actor.getSnapshot().value).toBe('timedOut');
});
```

### Testing with state.can()

```ts
test('can only submit from editing state with valid data', () => {
  const actor = createActor(formMachine);
  actor.start();

  // Can't submit in idle
  expect(actor.getSnapshot().can({ type: 'SUBMIT' })).toBe(false);

  actor.send({ type: 'EDIT' });

  // Still can't submit without valid data (guard fails)
  expect(actor.getSnapshot().can({ type: 'SUBMIT' })).toBe(false);

  actor.send({ type: 'field.change', field: 'name', value: 'John' });

  // Now can submit
  expect(actor.getSnapshot().can({ type: 'SUBMIT' })).toBe(true);
});
```

## Framework Testing

### React Testing Library + useMachine

```tsx
import { render, screen, act } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { test, expect } from 'vitest';

function Counter() {
  const [snapshot, send] = useMachine(counterMachine);
  return (
    <div>
      <span data-testid="count">{snapshot.context.count}</span>
      <button onClick={() => send({ type: 'increment' })}>+</button>
    </div>
  );
}

test('increments count on button click', async () => {
  render(<Counter />);

  expect(screen.getByTestId('count')).toHaveTextContent('0');

  await userEvent.click(screen.getByText('+'));

  expect(screen.getByTestId('count')).toHaveTextContent('1');
});
```

## Anti-Patterns

### Testing Config Instead of Behavior

```ts
// BAD — brittle, breaks on refactoring
test('has loading state', () => {
  expect(machine.config.states).toHaveProperty('loading');
});

// GOOD — test behavior
test('transitions to loading on FETCH', () => {
  const actor = createActor(machine).start();
  actor.send({ type: 'FETCH' });
  expect(actor.getSnapshot().value).toBe('loading');
});
```

### Ignoring Error Paths

```ts
// BAD — only testing happy path
test('fetches data', async () => {
  // ... only tests success case

// GOOD — test both paths
test('handles fetch success', async () => { /* ... */ });
test('handles fetch error', async () => { /* ... */ });
test('allows retry after error', async () => { /* ... */ });
```

### Coupling to State Names

```ts
// BAD — breaks if state is renamed
expect(snapshot.value).toBe('loadingUserData');

// BETTER — use tags for resilient assertions
expect(snapshot.hasTag('loading')).toBe(true);

// BEST — test observable behavior
expect(snapshot.context.data).toBeDefined();
```
