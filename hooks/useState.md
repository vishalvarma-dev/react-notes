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

```jsx
function Counter() {
  // Step 1 — React gives us the current count (0 on first render)
  const [count, setCount] = useState(0);

  const handleClick = () => {
    // Step 2 — you call setCount with the next value
    // count is still 0 here — React hasn't changed it yet for this render
    // Step 3 — React queues the update: count = 1 (doesn't apply it yet)
    // Step 4 — React batches this with any other setState calls in the same event
    setCount(count + 1);

    console.log(count); // still 0 — the value is fixed for this render
  };

  // Step 5 — after re-render, count is 1, DOM updates: "Clicked 1 times"
  return <button onClick={handleClick}>Clicked {count} times</button>;
}
```


```
  handleClick()
  setCount(count + 1)        ← you call this
         │
         ▼
  📦 Batching
  React collects all setState calls
  in the same event into one batch
         │
         ▼
  🗂️ Update Queue
  count = 1  ← sits in React's internal queue
         │
         ▼
  🔄 Counter re-renders
  useState() returns new count → 1
  UI updates: "Clicked 1 times"
```



When you click the button, this is the sequence:

1. **The click runs the handler.** `handleClick` runs with the `count` value from the current render, which is `0`.
2. **The setter is queued.** `setCount(count + 1)` does not change `count`. It adds an update to React's queue that says "set count to 1."
3. **React schedules a re-render.** The component is marked for an update, and React processes the queue.
4. **React re-runs the component.** When the component function runs again, `useState` applies the queued update and returns the new value, `1`.
5. **The screen updates.** React compares the new output with the old one and updates the DOM.

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

When you update state, React doesn't immediately stop everything and re-render the screen on every single line. Instead, it groups multiple state updates together into **one single re-render**. This is called **batching**.

### Why does React batch updates?

1. **Performance (Prevents unnecessary re-renders)**
   Re-rendering components and updating the browser DOM is expensive. If you update 3 state variables in a row, re-rendering 3 times would cause lag:
   
   ```
   Without batching:  setState() → Render 1 → setState() → Render 2 → setState() → Render 3 🐢
   With batching:     setState() + setState() + setState() ──▶ Single Render 🚀
   ```

2. **Prevents half-finished UI states (Visual Glitches)**
   Imagine you update a user's name and age at the same time:
   ```jsx
   setFirstName("John");
   setLastName("Doe");
   ```
   Without batching, the component would re-render halfway through — showing the new first name with the old last name for a split second before updating again. Batching ensures the UI updates only when all related changes are ready.

> **Analogy:** Think of batching like a waiter at a restaurant. They don't run to the kitchen after you order your drink, run back to take your appetizer, and run back again for your main course. They write down your entire order first, then visit the kitchen once.

Since React 18, batching happens **automatically everywhere** — inside event handlers, `setTimeout`, `fetch` callbacks, and promises.

## Each component instance has its own state

If you render `<Counter />` twice, each one keeps its own `count`. Changing one does not change the other.

## Updating objects and arrays

In JavaScript, objects and arrays are stored **by reference**, not by value. This means a variable doesn't hold the actual data — it holds a pointer to where the data lives in memory.

```js
const user = { name: "Asha", age: 25 };
const same = user;

same.age = 26;

console.log(user === same); // true — both point to the same object in memory
```

React detects state changes by comparing the old reference with the new one:

```
old reference ──▶ { name: "Asha", age: 25 }
new reference ──▶ { name: "Asha", age: 25 }   ← same object, same address

React sees: old === new → no change → skips re-render ❌
```

So when you do `user.age = 26` directly, you mutate the same object. React compares the old and new references, sees they are identical, and does nothing.

Instead, every update must produce a **new object or array**. React then sees a different reference and knows something changed:

```
old reference ──▶ { name: "Asha", age: 25 }
new reference ──▶ { name: "Asha", age: 26 }   ← brand new object

React sees: old !== new → re-render ✅
```

```jsx
const [user, setUser] = useState({ name: "Asha", age: 25 });

// ❌ Wrong — mutates the existing object, reference doesn't change
user.age = 26;
setUser(user); // React sees the same reference, skips re-render

// ✅ Correct — spread creates a brand new object with a new reference
setUser({ ...user, age: 26 });
```

The same rule applies to arrays — never use `.push()`, `.pop()`, or direct index assignment. Use methods that return a new array instead:

```jsx
const [items, setItems] = useState(["a", "b", "c"]);

// ❌ Wrong — mutates the existing array
items.push("d");
setItems(items);

// ✅ Correct — spread creates a new array
setItems([...items, "d"]);
```

## Summary

- `useState` stores values that persist between renders and re-renders the component when they change.
- A plain variable does not trigger re-renders and resets on every render.
- Clicking queues an update. The new value appears only on the next render, so the handler still sees the old value.
- Multiple setter calls in one event are batched into one re-render.
- Use the functional form `setCount(prev => prev + 1)` when the new value depends on the old one.
