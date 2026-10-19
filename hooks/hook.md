# What is a Hook in React?

In React 16.8, React introduced Hooks, a new pattern for using state and side effects in function components. Hooks are simple JavaScript functions that isolate reusable logic from a component. They let you share stateful logic across components without changing your component hierarchy.

## Why hooks exist

Before hooks, you needed class components to use state and lifecycle features. Hooks let you do the same things in regular functions, and they make it easier to reuse logic.

## Common built-in hooks

| Hook         | Purpose                                       | Example                                                         |
| ------------ | --------------------------------------------- | --------------------------------------------------------------- |
| `useState`   | Manages data that changes over time           | Tracking a counter, form input, or whether a menu is open       |
| `useEffect`  | Runs code after render to handle side effects | Fetching data from an API or listening to a window resize event |
| `useContext` | Reads shared values without passing props     | Sharing the logged-in user or theme across the whole app        |

### Example: `useState`

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>Clicked {count} times</button>
  );
}
```

## Custom hooks

A custom hook is a function whose name starts with `use` and that calls other hooks. Examples include `useSearchDebounce` and `useClickOutside`, which are useful when building a dashboard.

### Example: `useWindowWidth`

```jsx
import { useState, useEffect } from "react";

function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  return width;
}

function Dashboard() {
  const width = useWindowWidth();

  return <p>Window width: {width}px</p>;
}
```

## Do we need the `use` prefix?

No. The code still runs without it, because React identifies hooks by call order, not by name. The `use` prefix is a convention with two practical benefits:

1. **Linting:** ESLint's `react-hooks` rules check functions named `use...` for mistakes like calling hooks inside conditions or loops.
2. **Readability:** Other developers can tell at a glance that the function contains hooks and must follow the rules of hooks.

## The two rules of hooks

### 1. Only call hooks at the top level

- Do not call hooks inside loops, conditions, nested functions, or after early returns.
- Always call them at the top level of your React function body.
- **Why:** React relies on the same call order on every render to match state to each hook. Breaking this order causes state and effects to get out of sync.

```jsx
// Wrong: hook inside a condition
function Profile({ isLoggedIn }) {
  if (isLoggedIn) {
    const [name, setName] = useState("");
  }
}

// Correct: hook at the top level
function Profile({ isLoggedIn }) {
  const [name, setName] = useState("");
}
```

### 2. Only call hooks from React functions

- Call hooks only from React function components or custom hooks.
- Do not call them from regular JavaScript functions.
- **Why:** This keeps stateful logic tied to the React component lifecycle and easy to find.

## Summary

Hooks let function components use state and side effects. Follow the two rules, and use the `use` prefix so tools and other developers can recognize them.
