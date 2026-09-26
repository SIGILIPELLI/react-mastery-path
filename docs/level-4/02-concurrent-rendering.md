---
description: "Keep React UIs responsive with useTransition and useDeferredValue, understand interruptible rendering and lanes, and avoid unwanted Suspense fallbacks."
---

# 02 · Concurrent Rendering

Before React 18, once React started rendering an update it ran to completion. A slow
render — filtering 20,000 rows, drawing a big chart — froze the page, including the text
box the user was typing in. Concurrent rendering lets React **prepare an update in the
background, interrupt it for something more urgent, and throw it away if it's
outdated**. You opt in per update with two hooks.

## Urgent vs non-urgent updates

- **Urgent**: direct feedback to input — typing, clicking, pressing. Users expect these
  immediately.
- **Non-urgent (transitions)**: the *consequences* — the filtered list, the next tab's
  content, the chart. A short delay is fine, as long as the input stays responsive.

## `useTransition`

```jsx
import { useState, useTransition } from 'react'

export default function ProductSearch({ products }) {
  const [query, setQuery] = useState('')
  const [filter, setFilter] = useState('')
  const [isPending, startTransition] = useTransition()

  function handleChange(e) {
    setQuery(e.target.value)            // urgent: input shows the character now
    startTransition(() => {
      setFilter(e.target.value)         // non-urgent: list can lag behind
    })
  }

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <small>Updating…</small>}
      <SlowList products={products} filter={filter} />
    </>
  )
}
```

- Updates inside `startTransition` are marked low priority.
- If the user types again before the list finishes rendering, React **abandons** the
  in-progress render and starts over with the latest value.
- `isPending` is `true` while a transition is in progress — use it to dim stale content
  rather than blanking it.

`SlowList` must be meaningfully expensive for this to matter. For a demo, add an
artificial slowdown in each row:

```jsx
function SlowRow({ product }) {
  const start = performance.now()
  while (performance.now() - start < 0.5) {} // ~0.5ms per row, demo only!
  return <li>{product.name}</li>
}
```

With 1,000 rows that's ~500 ms per render: typing freezes without the transition, and
stays smooth with it.

`startTransition` can also be imported directly from `react` when you don't need
`isPending`. In React 19, the function passed to `startTransition` may be `async`
(an "Action"); `isPending` stays true until it finishes. State updates made after an
`await` inside it need their own `startTransition` wrapper in current React versions —
check the docs for your version.

## `useDeferredValue`

Sometimes you don't control the state update — the value comes in as a prop. Defer the
value instead:

```jsx
import { memo, useDeferredValue, useState } from 'react'

export default function Search({ products }) {
  const [query, setQuery] = useState('')
  const deferredQuery = useDeferredValue(query)
  const isStale = query !== deferredQuery

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <div style={{ opacity: isStale ? 0.6 : 1 }}>
        <Results products={products} query={deferredQuery} />
      </div>
    </>
  )
}

const Results = memo(function Results({ products, query }) {
  const q = query.toLowerCase()
  return (
    <ul>
      {products.filter(p => p.name.toLowerCase().includes(q)).map(p => <SlowRow key={p.id} product={p} />)}
    </ul>
  )
})
```

On each keystroke React first renders with the **old** `deferredQuery` (so `Results`
gets the same props and `memo` skips it — that render is cheap), commits the input, then
renders again in the background with the new value. **`memo` is essential here**:
without it, `Results` would re-render during the urgent pass anyway, defeating the point.

## Transitions and Suspense

When content that's already on screen suspends (lazy route, `use(promise)`, suspense
query), React normally shows the nearest fallback — replacing the page with a spinner.
If the update that caused it is a **transition**, React instead keeps showing the old UI
until the new one is ready:

```jsx
function Tabs() {
  const [tab, setTab] = useState('about')
  const [isPending, startTransition] = useTransition()

  return (
    <>
      <nav style={{ opacity: isPending ? 0.7 : 1 }}>
        {['about', 'posts', 'contact'].map(t => (
          <button key={t} onClick={() => startTransition(() => setTab(t))}>{t}</button>
        ))}
      </nav>
      <Suspense fallback={<Spinner />}>
        {tab === 'about' && <About />}
        {tab === 'posts' && <Posts />}          {/* suspends while loading */}
        {tab === 'contact' && <Contact />}
      </Suspense>
    </>
  )
}
```

Clicking "posts" keeps "about" visible (dimmed) until posts are ready — no flash of
spinner. Routers use this for navigation for the same reason. (New Suspense boundaries
that appear for the first time still show their fallbacks; transitions only avoid hiding
content that's already revealed.)

## What concurrency does *not* do

- It doesn't make your rendering code faster; it schedules it better.
- It can't interrupt a **single** long-running JavaScript function. React yields between
  components, so one component that loops for 300 ms still blocks for 300 ms.
- It doesn't apply to controlled input values: the input's own state must be urgent
  (never wrap the input's `setState` in a transition).
- Heavy non-rendering work (parsing a big file) belongs in a Web Worker.

## Worked example: choosing between them

| Situation | Tool |
|---|---|
| You own the `setState` that triggers the slow render | `useTransition` |
| The slow part receives a value from a prop or a hook you don't control | `useDeferredValue` |
| You want a pending indicator for navigation or a tab switch | `useTransition` (`isPending`) |
| You want to show stale results dimmed while new ones render | either; compare value vs deferred value, or use `isPending` |
| The work is one huge synchronous computation | neither — optimise it, memoize it, or move it to a worker |

## How It Actually Works

Every update is assigned a **lane**, a bit in a bitmask representing priority. Updates
from discrete events (click, keydown) get the sync lane; updates inside
`startTransition` get one of several transition lanes; `useDeferredValue`'s background
re-render also uses a lower-priority lane.

React's **work loop** renders one fiber at a time. In concurrent mode (used for
transition lanes), after each unit of work it calls `shouldYield()`, which checks
whether it has used its time slice (on the order of a few milliseconds). If so, it
yields control back to the browser so input events and painting can happen, then
resumes via its scheduler (which uses a `MessageChannel` to schedule a new macrotask).
Sync-lane work runs without yielding.

If an urgent update arrives while a transition render is in progress, React notices a
higher-priority lane is pending, **pauses the transition's work-in-progress tree**,
renders and commits the urgent update, then restarts the transition from the new state
(the old partial work is discarded because its inputs are outdated). Because the
abandoned tree was never committed, no DOM changes, effects or refs from it are ever
visible — which is exactly why render functions must be pure: they may run for trees
that never appear.

`useDeferredValue(value)` works by returning the old value during urgent renders and
scheduling a transition-priority render where it returns the new value. `isPending` for
`useTransition` is itself implemented as an urgent state update (`true`) plus the
transition (which sets it back to `false` when it commits).

## Common mistakes

- **Wrapping the input's own value update in a transition** → laggy or jumping typing.
- **`useDeferredValue` without `memo`** on the expensive child.
- **Expecting transitions to fix a single slow function.**
- **Showing a full spinner during transitions** instead of keeping and dimming old
  content.
- **Using transitions for everything**; urgent feedback should stay urgent.

## Exercise

1. Build a search over 5,000 generated items with the artificial `SlowRow` above and
   confirm typing is janky.
2. Fix it with `useTransition`; add a pending indicator.
3. Fix it instead with `useDeferredValue` + `memo`; then remove `memo` and explain what
   you observe.
4. Record both versions in the Performance panel and compare the longest task during
   typing.
5. Build the tab example with a lazy-loaded tab and show that the old tab stays visible
   when switching inside a transition but a spinner appears without it.
