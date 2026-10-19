# What is useState?

`useState` is a React hook that lets a function component store and update local state. When the state changes, React re-renders the component so the screen shows the new value.

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0); // initial value is 0

  return (
    <button onClick={() => setCount(count + 1)}>Clicked {count} times</button>
  );
}
```

`useState` returns two things:

- **The current value** (`count`)
- **A setter function** (`setCount`) that queues an update and schedules a re-render

## Why not use a normal variable?

You might try this:

```jsx
function Counter() {
  let count = 0;

  return (
    <button
      onClick={() => {
        count = count + 1;
      }}
    >
      Clicked {count} times
    </button>
  );
}
```

This fails for two reasons:

1. **Changing a variable does not re-render the component.** The button click changes `count`, but React has no reason to update the screen, so the text still shows `0`.
2. **Local variables reset on every render.** Each time the component runs, `let count = 0` runs again, so even if something triggered a re-render, the count would start from `0`.

`useState` solves both problems. React stores the value outside the function, keeps it between renders, and re-renders the component when you call the setter.

## How a state update happens

When you click the button, this is the sequence:

1. **The click runs the handler.** `handleClick` runs with the `count` value from the current render, which is `0`.
2. **The setter is queued.** `setCount(count + 1)` does not change `count`. It adds an update to React's queue that says "set count to 1."
3. **React schedules a re-render.** The component is marked for an update, and React processes the queue.
4. **React re-runs the component.** When the component function runs again, `useState` applies the queued update and returns the new value, `1`.
5. **The screen updates.** React compares the new output with the old one and updates the DOM.

```jsx
function Counter() {
  const [count, setCount] = useState(0); // returns 0 in render 1, 1 in render 2

  const handleClick = () => {
    setCount(count + 1); // queues an update, does not change count now
    console.log(count); // still 0: this function belongs to render 1
  };

  return <button onClick={handleClick}>Clicked {count} times</button>;
}
```

### Why the console shows the old value

Each render creates its own copy of `count` and its own `handleClick`. The handler from render 1 holds `count = 0`, even after React has moved on to render 2.

| Render | `count` value | `handleClick` sees |
| ------ | ------------- | ------------------ |
| 1      | `0`           | `count = 0`        |
| 2      | `1`           | `count = 1`        |

So the component shows the old value until React runs it again with the new one.

### Why two calls add only 1

```jsx
const handleClick = () => {
  setCount(count + 1); // queued: set to 0 + 1 = 1
  setCount(count + 1); // queued: set to 0 + 1 = 1
};
```

Both calls read `count` from the same render, which is `0`. React queues two updates, both setting the value to `1`. It processes them in order and re-renders once, so the result is `1`.

To make each update build on the previous queued one, use the functional form:

```jsx
const handleClick = () => {
  setCount((prev) => prev + 1); // queued: take the pending value, add 1
  setCount((prev) => prev + 1); // queued: take the result above, add 1
};
```

Now React processes the queue as `0 → 1 → 2`, and the result is `2`.

## Batching

When you call the setter several times in the same event, React combines the updates into one re-render. This is called **batching**. Since React 18, batching happens automatically, even in promises and timeouts.

## Each component instance has its own state

If you render `<Counter />` twice, each one keeps its own `count`. Changing one does not change the other.

## Updating objects and arrays

Don't change the existing object or array directly. Create a new one, so React can detect the change:

```jsx
const [user, setUser] = useState({ name: "Asha", age: 25 });

// Wrong: mutates the existing object, so React may not re-render
user.age = 26;

// Correct: creates a new object
setUser({ ...user, age: 26 });
```

## Summary

- `useState` stores values that persist between renders and re-renders the component when they change.
- A plain variable does not trigger re-renders and resets on every render.
- Clicking queues an update. The new value appears only on the next render, so the handler still sees the old value.
- Multiple setter calls in one event are batched into one re-render.
- Use the functional form `setCount(prev => prev + 1)` when the new value depends on the old one.
