# SolidJS Reactivity Pitfalls

Detailed anti-patterns with incorrect and correct examples. These are the most common mistakes, often from React mental models.

## 1. Destructuring Props

Props are reactive proxies. Destructuring reads the values once during component setup, losing reactivity.

```tsx
// WRONG — name is read once, never updates
function Greeting({ name }: { name: string }) {
  return <h1>Hello {name}</h1>;
}

// CORRECT — access props.name in JSX (tracking scope)
function Greeting(props: { name: string }) {
  return <h1>Hello {props.name}</h1>;
}

// CORRECT — use splitProps when you need to separate props
function Greeting(props: { name: string; class?: string }) {
  const [local, others] = splitProps(props, ["name"]);
  return <h1 {...others}>Hello {local.name}</h1>;
}
```

## 2. Conditional Logic Outside JSX

Conditions evaluated during component setup run once. The branch is fixed.

```tsx
// WRONG — condition evaluated once, never re-evaluates
function Status(props: { isOnline: boolean }) {
  if (props.isOnline) {
    return <span>Online</span>;
  }
  return <span>Offline</span>;
}

// CORRECT — use Show or ternary inside JSX
function Status(props: { isOnline: boolean }) {
  return (
    <Show when={props.isOnline} fallback={<span>Offline</span>}>
      <span>Online</span>
    </Show>
  );
}

// CORRECT — ternary in JSX expression (tracking scope)
function Status(props: { isOnline: boolean }) {
  return <span>{props.isOnline ? "Online" : "Offline"}</span>;
}
```

## 3. Deriving State with Effects Instead of Memos

Using createEffect + createSignal to compute derived state is wasteful and error-prone.

```tsx
// WRONG — unnecessary signal + effect, causes extra updates
function DoubleCount() {
  const [count, setCount] = createSignal(0);
  const [doubled, setDoubled] = createSignal(0);

  createEffect(() => {
    setDoubled(count() * 2); // setting signal in effect
  });

  return <span>{doubled()}</span>;
}

// CORRECT — use createMemo for derived values
function DoubleCount() {
  const [count, setCount] = createSignal(0);
  const doubled = createMemo(() => count() * 2);
  return <span>{doubled()}</span>;
}

// CORRECT — or just a derived signal for simple cases
function DoubleCount() {
  const [count, setCount] = createSignal(0);
  const doubled = () => count() * 2;
  return <span>{doubled()}</span>;
}
```

## 4. Forgetting () on Signal Getters

Signals are functions. Using `count` instead of `count()` passes the function itself.

```tsx
// WRONG — passes the getter function, not the value
<span>{count}</span>       // displays the function reference
<div class={isActive}>     // sets class to function toString

// CORRECT — call the getter
<span>{count()}</span>
<div class={isActive() ? "active" : ""}>
```

## 5. Accessing Signals Outside Tracking Scopes

Code in the component body runs once. Signal reads there are not tracked.

```tsx
// WRONG — this log runs once, not on every change
function MyComponent() {
  const [count, setCount] = createSignal(0);
  console.log("count is", count()); // runs once

  return <button onClick={() => setCount(c => c + 1)}>{count()}</button>;
}

// CORRECT — put reactive logic in an effect
function MyComponent() {
  const [count, setCount] = createSignal(0);

  createEffect(() => {
    console.log("count is", count()); // runs on every change
  });

  return <button onClick={() => setCount(c => c + 1)}>{count()}</button>;
}
```

## 6. Event Handler Reactivity Assumptions

Event handlers are NOT tracking scopes. They also don't dynamically rebind.

```tsx
// The handler is bound once. This is fine — signals work in handlers
// because you're imperatively reading values, not creating subscriptions.
<button onClick={() => {
  console.log(count()); // reads current value (not tracked, but that's OK)
  setCount(c => c + 1);
}}>

// WRONG assumption — handler prop won't "rebind" dynamically
// But the handler itself can call reactive sources:
<div onClick={() => props.handleClick?.()} />
```

## 7. stopPropagation with Delegated Events

Solid uses event delegation for common events (onClick, onInput, etc). `stopPropagation` may not work as expected because delegated events are handled at the document level.

```tsx
// WRONG — stopPropagation won't prevent native listeners on ancestors
<button onClick={(e) => {
  e.stopPropagation(); // may not stop native event listeners
  doSomething();
}}>

// CORRECT — use native event binding when you need stopPropagation
<button on:click={(e) => {
  e.stopPropagation(); // works correctly with native events
  doSomething();
}}>
```

## 8. Async Breaking Tracking Context

`await` inside a tracking scope breaks the synchronous tracking context. Signal reads after `await` are not tracked.

```tsx
// WRONG — signal reads after await are not tracked
createEffect(async () => {
  const id = userId(); // tracked
  const data = await fetchUser(id);
  setName(data.name); // this part runs fine
  console.log(count()); // NOT tracked — after await
});

// CORRECT — read all signals before await
createEffect(() => {
  const id = userId(); // tracked
  const currentCount = count(); // tracked

  (async () => {
    const data = await fetchUser(id);
    setName(data.name);
    console.log(currentCount);
  })();
});

// CORRECT — use createResource for async data fetching
const [user] = createResource(userId, fetchUser);
```

## 9. Creating Signals/Effects Outside Reactive Scope

Signals and effects created outside a component or `createRoot` won't be properly disposed.

```tsx
// WRONG — effect created in event handler, no owner for cleanup
function App() {
  return (
    <button onClick={() => {
      const [count, setCount] = createSignal(0);
      createEffect(() => console.log(count())); // memory leak!
    }}>
  );
}

// CORRECT — create reactive primitives inside the component body
function App() {
  const [count, setCount] = createSignal(0);

  createEffect(() => console.log(count()));

  return <button onClick={() => setCount(c => c + 1)}>Click</button>;
}
```

## 10. Assuming Effect Execution Order

The order of execution among multiple effects is NOT guaranteed.

```tsx
// WRONG — assuming effectB runs after effectA
createEffect(() => { /* effectA */ setX(y()); });
createEffect(() => { /* effectB */ console.log(x()); });

// CORRECT — if you need ordering, use a single effect or derive with memo
const x = createMemo(() => y() * 2);
createEffect(() => console.log(x()));
```

## 11. Early Returns in Components

Since components run once, an early return creates a static result.

```tsx
// WRONG — early return is evaluated once
function UserProfile(props: { userId?: string }) {
  if (!props.userId) return <div>No user</div>; // fixed forever

  const [user] = createResource(() => props.userId, fetchUser);
  return <div>{user()?.name}</div>;
}

// CORRECT — use Show for conditional rendering
function UserProfile(props: { userId?: string }) {
  return (
    <Show when={props.userId} fallback={<div>No user</div>}>
      {(id) => {
        const [user] = createResource(id, fetchUser);
        return <div>{user()?.name}</div>;
      }}
    </Show>
  );
}
```
