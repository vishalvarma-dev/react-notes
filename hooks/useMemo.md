# What is useMemo?

`useMemo` is a hook that caches a computed value. It recalculates the value only when its dependencies change, and returns the cached value on every other render.

## When to use it

- When a calculation is expensive and you don't want to repeat it on every render.
- When you need to derive data from state, such as a filtered list, and pass it to another component. You can compute it directly instead of storing it in extra state.

## Without useMemo

```jsx
import { useState, useEffect } from "react";

function ProductList({ products }) {
  const [search, setSearch] = useState("");
  const [filtered, setFiltered] = useState([]);

  // Runs after render, then sets state, which causes another render
  useEffect(() => {
    setFiltered(
      products.filter((p) =>
        p.name.toLowerCase().includes(search.toLowerCase()),
      ),
    );
  }, [search, products]);

  return (
    <div>
      <input value={search} onChange={(e) => setSearch(e.target.value)} />
      {filtered.map((p) => (
        <div key={p.id}>{p.name}</div>
      ))}
    </div>
  );
}
```

This works, but it adds an extra state variable and an extra render every time `search` changes.

## With useMemo

```jsx
import { useState, useMemo } from "react";

function ProductList({ products }) {
  const [search, setSearch] = useState("");
  const [category, setCategory] = useState("all");

  // Recalculated only when products, search, or category changes
  const filtered = useMemo(() => {
    console.log("Filtering products...");
    return products
      .filter((p) => category === "all" || p.category === category)
      .filter((p) => p.name.toLowerCase().includes(search.toLowerCase()))
      .sort((a, b) => a.name.localeCompare(b.name));
  }, [products, search, category]);

  return (
    <div>
      <input
        placeholder="Search products"
        value={search}
        onChange={(e) => setSearch(e.target.value)}
      />
      <select value={category} onChange={(e) => setCategory(e.target.value)}>
        <option value="all">All</option>
        <option value="electronics">Electronics</option>
        <option value="clothing">Clothing</option>
      </select>

      {filtered.map((p) => (
        <div key={p.id}>
          {p.name} - ${p.price}
        </div>
      ))}
    </div>
  );
}
```

## Full working example

This version runs on its own. It also shows how a value that doesn't depend on the inputs stays cached.

```jsx
import { useState, useMemo } from "react";

const products = [
  { id: 1, name: "Laptop", category: "electronics", price: 999 },
  { id: 2, name: "T-Shirt", category: "clothing", price: 20 },
  { id: 3, name: "Headphones", category: "electronics", price: 150 },
  { id: 4, name: "Jacket", category: "clothing", price: 80 },
];

export default function App() {
  const [search, setSearch] = useState("");
  const [category, setCategory] = useState("all");
  const [theme, setTheme] = useState("light");

  // Recalculated only when search or category changes
  const filtered = useMemo(() => {
    console.log("Filtering products...");
    return products
      .filter((p) => category === "all" || p.category === category)
      .filter((p) => p.name.toLowerCase().includes(search.toLowerCase()));
  }, [search, category]);

  return (
    <div
      style={{
        background: theme === "light" ? "#fff" : "#222",
        color: theme === "light" ? "#000" : "#fff",
      }}
    >
      <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>
        Toggle theme
      </button>

      <input
        placeholder="Search products"
        value={search}
        onChange={(e) => setSearch(e.target.value)}
      />
      <select value={category} onChange={(e) => setCategory(e.target.value)}>
        <option value="all">All</option>
        <option value="electronics">Electronics</option>
        <option value="clothing">Clothing</option>
      </select>

      {filtered.map((p) => (
        <div key={p.id}>
          {p.name} - ${p.price}
        </div>
      ))}
    </div>
  );
}
```

**What to notice:** Clicking "Toggle theme" re-renders the component, but `Filtering products...` does not appear in the console. `useMemo` returns the cached result because `search` and `category` didn't change. Typing in the search box or changing the category logs the message, because those dependencies changed.

## Summary

`useMemo` caches a computed value and recalculates it only when its dependencies change. Use it for expensive calculations or derived data, and don't use it for simple work that is cheap to repeat.
