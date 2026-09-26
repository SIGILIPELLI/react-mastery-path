---
description: "Learn what triggers React re-renders, when memo, useMemo and useCallback actually help, when they are wasted effort, and cheaper structural fixes."
---

# 08 · Performance Basics: memo, useMemo, useCallback

React is fast by default for most UIs. Performance problems usually come from a few
specific causes, and the memoization APIs are tools for those causes — not seasoning to
sprinkle on every component. This lesson teaches you to **measure first** and to reach
for structural fixes before memoization.

## What causes a re-render

A component re-renders when:

1. Its own **state** changes (via a setter or dispatch).
2. Its **parent re-renders** (by default, all children re-render too).
3. A **context** it reads changes.

Props changing is *not* a separate trigger: props only change because the parent
re-rendered. And a re-render is not a DOM update — React re-runs your function, diffs
the result, and touches the DOM only where output differs. Most re-renders are cheap.

## Measure before optimizing

- **React Developer Tools → Profiler**: record an interaction, see which components
  rendered, how long each took, and (with the setting enabled) *why* each rendered.
- **"Highlight updates when components render"** in the React DevTools settings flashes
  re-rendering components on screen.
- Test with a production build (`npm run build && npm run preview`); development mode
  is significantly slower and double-renders in `StrictMode`.

If an interaction feels instant and the profiler shows renders of a few milliseconds,
stop. There's nothing to fix.

## Structural fixes first

### Move state down

```jsx
// ❌ typing re-renders the expensive chart
function Dashboard() {
  const [query, setQuery] = useState('')
  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <SearchResults query={query} />
      <ExpensiveChart />
    </>
  )
}

// ✅ state lives where it's used
function Search() {
  const [query, setQuery] = useState('')
  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <SearchResults query={query} />
    </>
  )
}
function Dashboard() {
  return (
    <>
      <Search />
      <ExpensiveChart />
    </>
  )
}
```

### Lift content up (pass as children)

```jsx
function ScrollTracker({ children }) {
  const [y, setY] = useState(0)
  useEffect(() => {
    const onScroll = () => setY(window.scrollY)
    window.addEventListener('scroll', onScroll, { passive: true })
    return () => window.removeEventListener('scroll', onScroll)
  }, [])
  return (
    <div>
      <ProgressBar value={y} />
      {children}
    </div>
  )
}

<ScrollTracker>
  <Article />   {/* not re-rendered on scroll: the element came from the parent */}
</ScrollTracker>
```

These two moves solve a large share of real re-render problems with zero memoization.

## `memo`: skip re-rendering when props are equal

```jsx
import { memo } from 'react'

const ProductRow = memo(function ProductRow({ product, onAdd }) {
  return (
    <li>
      {product.name} <button onClick={() => onAdd(product.id)}>Add</button>
    </li>
  )
})
```

`memo` makes React compare the new props with the previous ones (shallowly, each prop
with `Object.is`). If all are equal, it reuses the last result and skips the function.

But `memo` is easily defeated:

```jsx
function ProductList({ products }) {
  const [cart, setCart] = useState([])
  const onAdd = id => setCart(c => [...c, id])      // new function every render
  return products.map(p => <ProductRow key={p.id} product={p} onAdd={onAdd} />)
}
```

`onAdd` is a new function each render, so `memo`'s comparison fails every time.

## `useCallback` and `useMemo`: keep references stable

```jsx
const onAdd = useCallback(id => setCart(c => [...c, id]), [])
```

`useCallback(fn, deps)` returns the same function until a dependency changes.
`useMemo(() => value, deps)` does the same for any computed value:

```jsx
const sorted = useMemo(
  () => products.toSorted((a, b) => a.price - b.price),
  [products],
)
```

`useCallback(fn, deps)` is exactly `useMemo(() => fn, deps)`.

## When `useMemo` is worth it

1. **An expensive calculation** — sorting/filtering thousands of items, heavy parsing —
   measured to cost noticeable time (say, over a millisecond or two per render). You can
   check with `console.time('sort')` / `console.timeEnd('sort')`.
2. **Referential stability** — the value is passed to a `memo` child, used as an effect
   dependency, or as a context value.

## When NOT to memoize

- The component is cheap to render (most are).
- The `memo` child receives `children` or inline objects that change every render
  anyway — the comparison runs and fails, costing more than it saves.
- To "fix" an effect running too often. Usually the real fix is to move the object
  creation inside the effect or depend on primitives.
- Everywhere, "just in case". Every memo adds dependency arrays you must keep correct,
  and stale dependencies create real bugs.

## Worked example: a filtered list done right

```jsx
import { memo, useCallback, useMemo, useState } from 'react'

const Row = memo(function Row({ item, selected, onSelect }) {
  return (
    <li className={selected ? 'selected' : undefined} onClick={() => onSelect(item.id)}>
      {item.name} — ₹{item.price}
    </li>
  )
})

export default function Catalog({ items }) {       // imagine 5,000 items
  const [query, setQuery] = useState('')
  const [selectedId, setSelectedId] = useState(null)

  const visible = useMemo(() => {
    const q = query.toLowerCase()
    return items.filter(i => i.name.toLowerCase().includes(q))
  }, [items, query])

  const handleSelect = useCallback(id => setSelectedId(id), [])

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <p>{visible.length} results</p>
      <ul>
        {visible.map(item => (
          <Row key={item.id} item={item} selected={item.id === selectedId} onSelect={handleSelect} />
        ))}
      </ul>
    </>
  )
}
```

Selecting a row changes `selectedId`: `Catalog` re-renders, `visible` is reused (its
dependencies didn't change), and of 5,000 memoized rows only the two whose `selected`
prop flipped actually re-render. Each piece earns its place. For lists this long, also
consider **virtualization** (rendering only visible rows with a library such as
TanStack Virtual), which usually matters more than memoization.

## The React Compiler

React Compiler is a build-time tool from the React team that automatically inserts
memoization equivalent to `memo`/`useMemo`/`useCallback` where it's safe. It requires
your components to follow the Rules of React (pure rendering, no mutation). If your
project uses it, much of the manual memoization in this lesson becomes unnecessary —
but the mental model of *why* things re-render still applies. Check the official docs
for its current status and setup for your toolchain.

## How It Actually Works

When a parent re-renders, React compares each child element with the one in the same
position last time. Without `memo`, if the type matches it simply calls the child
function again. With `memo`, the element type is a special wrapper; React first
compares the old and new props object key by key with `Object.is` and, if all match
(and the component has no pending state or context updates of its own), it **bails
out**: reuses the previous fiber subtree without calling the function or descending
further.

One more built-in bail-out exists without `memo`: if a parent re-renders but passes the
**exact same element object** as last time (which happens with `children` passed from
further up), React skips that child. That's the mechanism behind "lift content up".

`useMemo` and `useCallback` store `[value, deps]` in their hook slot. On each render,
React compares the new deps array to the stored one with `Object.is` per item; if all
equal, it returns the stored value, otherwise it recomputes and stores the new pair.
React treats this as a performance hint, not a guarantee: it may discard cached values
(for example for components that suspend while mounting), so code must still be correct
if the memoized function runs again.

## Common mistakes

- **Memoizing without measuring**, especially wrapping every component in `memo`.
- **Passing inline objects/arrays/functions to `memo` children** and wondering why it
  "doesn't work".
- **Wrong dependencies** in `useMemo`/`useCallback` → stale values captured forever.
- **Profiling in development mode** and optimizing numbers that don't exist in
  production.
- **Using `useMemo` for side effects.** It runs during render; side effects belong in
  handlers or effects.

## Exercise

1. Create a list of 10,000 generated items (`Array.from({ length: 10000 }, (_, i) => ...)`)
   with a search box and a "selected" row, deliberately without any memoization.
2. Record an interaction in the React Profiler (production build if possible) and note
   the commit duration for typing and for selecting.
3. Apply fixes one at a time — moving state down, `useMemo` for filtering, `memo` +
   `useCallback` for rows — and record the profile after each. Keep a small table of
   your measurements.
4. Remove any optimisation that didn't measurably help, and write one sentence per
   removal explaining why it didn't matter.
