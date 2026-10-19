# What is useEffect?

`useEffect` is a hook that lets you run side effects in a component. A side effect is anything that happens outside of rendering the UI, such as calling an API, opening a WebSocket, or setting up a timer.

## Why not call the API directly?

A component function runs on every render. If you call an API directly in the component body, this loop happens:

1. The component renders.
2. `fetch` runs and starts a request.
3. When the response arrives, `setUsers` stores a new array, which causes a re-render.
4. The re-render runs the component body again, so `fetch` runs again.
5. This repeats forever, sending requests without stopping.

```jsx
let renderCount = 0; // for demonstration only

function Users() {
  const [users, setUsers] = useState([]);
  renderCount++;
  console.log("Render:", renderCount);

  // ❌ Wrong: runs on every render, which creates an infinite loop
  fetch("/api/users")
    .then((res) => res.json())
    .then((data) => setUsers(data));

  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}
```

Open the browser console and you'll see `Render: 1`, `Render: 2`, `Render: 3`, and so on, increasing without stopping, along with a stream of network requests.

React expects components to be predictable. Its job is to calculate the UI from props and state. Running API calls during rendering breaks that expectation, and React may render a component more than once.

> **Note on React Strict Mode:** In development, React deliberately mounts and remounts components an extra time to help detect missing cleanups. Seeing effects run twice initially is intentional development behavior, not an infinite loop.

---

## The Correct Way with useEffect

`useEffect` runs after React has updated the screen. Put side effects inside it:

```jsx
import { useState, useEffect } from "react";

function Users() {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    fetch("/api/users")
      .then((res) => res.json())
      .then((data) => setUsers(data));
  }, []); // empty array: runs once after the first render

  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}
```

---

## The Dependency Array

The second argument to `useEffect` is the dependency array. It tells React when to run the effect again.

| Dependency array | When the effect runs                                        |
| :--------------- | :---------------------------------------------------------- |
| **No array**     | After every render                                          |
| `[]` (empty)     | Only once, after the first render                           |
| `[userId]`       | After the first render, and again whenever `userId` changes |

### Empty dependency array `[]`

With `[]`, React runs the effect **once, after the first render**, and does not run it again while the component stays on screen.

Think of `[]` as saying: _"None of my values change, so I don't need to run again."_

1. The component renders with an empty `users` list.
2. React runs the effect, which calls the API.
3. The data arrives and `setUsers` updates the state, so the component renders again.
4. React sees `[]` again, so it does **not** call the API a second time.

### Dependency array with values

Adding a value to the array tells React to run the effect again whenever that value changes:

```jsx
import { useState, useEffect } from "react";

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then((data) => setUser(data));
  }, [userId]); // runs again when userId changes

  return <p>{user ? user.name : "Loading..."}</p>;
}
```

**Rule of thumb:** list every reactive value from props or state that the effect reads. If you leave one out, the effect can use an old value (a **stale closure**).

---

## Stale Closures (Stale Data)

A **stale closure** happens when an effect captures a value from an earlier render because that value was omitted from the dependency array.

```jsx
import { useState, useEffect } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log("Count is:", count); // always logs 0
    }, 1000);

    return () => clearInterval(id);
  }, []); // ❌ count is missing from the dependency array

  return (
    <button onClick={() => setCount(count + 1)}>Clicked {count} times</button>
  );
}
```

**Why it happens:** The effect runs once with `[]`, capturing the value of `count` from that initial render (`0`). Even when the user clicks the button and updates the state, the interval closure still points to `0`.

### Fix 1: Add the value to the dependency array

The effect tears down and restarts with the latest value whenever it changes:

```jsx
useEffect(() => {
  const id = setInterval(() => {
    console.log("Count is:", count);
  }, 1000);

  return () => clearInterval(id);
}, [count]); // restarts whenever count updates
```

### Fix 2: Use the functional updater

When you only need the previous value to compute the next state, pass an updater function instead:

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount((prev) => prev + 1); // always receives the latest state
  }, 1000);

  return () => clearInterval(id);
}, []); // count does not need to be in the array
```

---

## Cleanup Functions

If an effect creates a persistent connection, subscribes to an event, or starts a timer, return a cleanup function. React executes this cleanup:

1. **Before** running the effect again on a re-render.
2. When the component **unmounts**.

```jsx
useEffect(() => {
  const timer = setInterval(() => console.log("tick"), 1000);

  // Cleanup: prevents memory leaks when the component unmounts
  return () => clearInterval(timer);
}, []);
```

Without a cleanup function, the timer or subscription keeps running in the background after the component is removed from the screen, causing a **memory leak**.

### Handling Network Race Conditions

When fetching data inside `useEffect`, fast-changing dependencies can cause old responses to overwrite newer ones. Use a cleanup flag or `AbortController`:

```jsx
useEffect(() => {
  const controller = new AbortController();

  fetch(`/api/users/${userId}`, { signal: controller.signal })
    .then((res) => res.json())
    .then((data) => setUser(data))
    .catch((err) => {
      if (err.name !== "AbortError") console.error(err);
    });

  return () => controller.abort(); // cancels the in-flight request if userId changes
}, [userId]);
```

---

## Summary

- Components should focus purely on calculating UI.
- Use `useEffect` to synchronize your component with external systems (APIs, timers, subscriptions) after rendering.
- Always list every reactive prop and state variable read by the effect in its dependency array.
- Return a cleanup function for any continuous process to avoid memory leaks and stale operations.
