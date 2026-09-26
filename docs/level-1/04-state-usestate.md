---
description: "Use useState to make components interactive, update objects and arrays immutably, and understand state as a per-render snapshot."
---

# 04 · State with useState

Props come from outside. **State** is data a component owns and can change. Changing
state is the only normal way to make React re-render and update the screen.

## A first counter

```jsx
import { useState } from 'react'

export default function Counter() {
  const [count, setCount] = useState(0)

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  )
}
```

`useState(0)` returns a pair: the current value and a setter. Calling `setCount`
schedules a re-render; on that next render `useState` returns the new value.

Why not a regular variable?

```jsx
function BrokenCounter() {
  let count = 0
  return <button onClick={() => { count++ }}>Clicked {count}</button>
}
```

Two reasons it fails: changing a local variable doesn't tell React to re-render, and
even if something else re-rendered it, `let count = 0` runs again and starts from zero.
State lives *outside* the function call, managed by React.

## State is a snapshot

This surprises everyone once:

```jsx
function Tripler() {
  const [n, setN] = useState(0)

  function handleClick() {
    setN(n + 1)
    setN(n + 1)
    setN(n + 1)
    console.log(n) // still 0
  }

  return <button onClick={handleClick}>{n}</button>
}
```

One click shows `1`, not `3`. Inside this render, `n` is the constant `0`. All three
calls say "set it to 0 + 1". The setter doesn't change the variable you already have;
it asks React for a new render in which `n` will be different.

### Updater functions

When the new value depends on the old one, pass a function:

```jsx
function handleClick() {
  setN(prev => prev + 1)
  setN(prev => prev + 1)
  setN(prev => prev + 1)
}
```

Now one click gives `3`. React queues the three functions and runs them in order,
feeding each the result of the previous one.

Rule of thumb: if the next state is computed from the previous state, use the updater
form. It is never wrong, and it avoids stale-value bugs in async code.

## Objects and arrays: replace, don't mutate

React decides whether state changed by comparing old and new with `Object.is`. If you
mutate an object and pass the *same* object back, React sees no change.

```jsx
const [profile, setProfile] = useState({ name: 'Meera', city: 'Kochi' })

// ❌ same object; React may skip the update
profile.city = 'Chennai'
setProfile(profile)

// ✅ new object with the change
setProfile({ ...profile, city: 'Chennai' })
```

Arrays follow the same idea. Prefer non-mutating operations:

| Goal | Avoid (mutates) | Use (returns new) |
|---|---|---|
| add | `push`, `unshift` | `[...arr, item]`, `[item, ...arr]` |
| remove | `splice` | `arr.filter(x => x.id !== id)` |
| replace | `arr[i] = x` | `arr.map(x => x.id === id ? {...x, done: true} : x)` |
| sort | `sort()` | `[...arr].sort()` or `arr.toSorted()` |

Nested objects need copying at every level you change:

```jsx
setOrder(prev => ({
  ...prev,
  shipping: { ...prev.shipping, city: 'Mysuru' },
}))
```

## Lazy initial state

```jsx
const [notes, setNotes] = useState(() => JSON.parse(localStorage.getItem('notes') ?? '[]'))
```

Passing a function means it only runs on the first render. Writing
`useState(JSON.parse(...))` would re-read and re-parse storage on *every* render and
then ignore the result.

## Worked example: a shopping list

```jsx
import { useState } from 'react'

let nextId = 3

export default function ShoppingList() {
  const [items, setItems] = useState([
    { id: 1, name: 'Rice', bought: false },
    { id: 2, name: 'Curd', bought: true },
  ])
  const [draft, setDraft] = useState('')

  function addItem() {
    const name = draft.trim()
    if (!name) return
    setItems(prev => [...prev, { id: nextId++, name, bought: false }])
    setDraft('')
  }

  function toggle(id) {
    setItems(prev => prev.map(i => (i.id === id ? { ...i, bought: !i.bought } : i)))
  }

  function remove(id) {
    setItems(prev => prev.filter(i => i.id !== id))
  }

  const remaining = items.filter(i => !i.bought).length

  return (
    <div>
      <h2>Shopping ({remaining} left)</h2>
      <input value={draft} onChange={e => setDraft(e.target.value)} placeholder="Add item" />
      <button onClick={addItem}>Add</button>
      <ul>
        {items.map(item => (
          <li key={item.id}>
            <label>
              <input type="checkbox" checked={item.bought} onChange={() => toggle(item.id)} />
              <span style={{ textDecoration: item.bought ? 'line-through' : 'none' }}>{item.name}</span>
            </label>
            <button onClick={() => remove(item.id)} aria-label={`Remove ${item.name}`}>×</button>
          </li>
        ))}
      </ul>
    </div>
  )
}
```

Notice that `remaining` is **not** state. It's computed from `items` on every render.
Storing it separately would create two sources of truth that can drift apart.

## How It Actually Works

React keeps, for each mounted component instance, an internal record (a "fiber"). Each
fiber has an ordered **list of hooks**. On the first render, each `useState` call
appends a slot holding the initial value. On later renders React walks the same list:
the first `useState` call gets slot 1, the second gets slot 2, and so on. There are no
names involved — **position is identity**. That's why hooks must be called in the same
order every render (never inside `if` or loops). The Rules of Hooks, covered with
custom hooks in Level 2, are a direct consequence of this design.

Calling a setter doesn't change anything immediately. It pushes an **update object**
onto that hook's queue and schedules a re-render of the component. When React re-renders,
it processes the queue in order: a plain value replaces the state; an updater function
is called with the current state. The final result is what `useState` returns this time.

Updates are **batched**: since React 18, all setter calls made during the same event —
and also inside timeouts, promises and native event handlers — are grouped into a
single re-render. That's why three `setN` calls cause one render, not three.

Before scheduling, React compares the new value to the current one with `Object.is`.
If they're identical, React can bail out without re-rendering children. Mutating an
object in place defeats this check, which is the mechanical reason immutability matters.

Finally, each render's event handlers are closures over that render's state values. The
`handleClick` created during the render where `n` is `0` will always see `0`. This is
the "snapshot" behaviour, and later the root of the "stale closure" bugs discussed with
effects.

## Common mistakes

- **Reading state right after setting it** and expecting the new value. Use the value
  you computed, or read it on the next render.
- **Mutating then setting the same reference** — the UI doesn't update or updates
  inconsistently.
- **Storing derived data** (counts, filtered lists, full names) in state. Compute it
  during render instead.
- **Calling a setter during render** (`setX(1)` at the top level of the component)
  causes an infinite render loop: "Too many re-renders".
- **Writing `onClick={setCount(count + 1)}`** — that *calls* the setter during render.
  It must be a function: `onClick={() => setCount(count + 1)}`.

## Exercise

Build a `SeatPicker` for a 4×5 cinema grid:

1. Store the selected seats as an array of seat ids like `"B3"` in state.
2. Clicking a seat toggles it; selected seats get a different background.
3. Limit selection to 6 seats — ignore clicks beyond that and show a message.
4. Show the total price (₹180 per seat) computed from state, not stored in state.
5. Add an "Add two adjacent seats" button that calls a setter twice in the same
   handler. Make it work correctly using updater functions, and write a one-line
   comment explaining why the non-updater version would fail.
