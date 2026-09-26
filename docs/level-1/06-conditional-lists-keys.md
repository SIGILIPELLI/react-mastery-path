---
description: "Render UI conditionally with if, ternaries and &&, turn arrays into elements with map, and choose keys that keep list state correct."
---

# 06 · Conditional Rendering, Lists & Keys

Real interfaces constantly show different things depending on data: a spinner or a
table, an empty state or a list, a badge only for admins. In React, all of this is just
JavaScript deciding what to return.

## Three ways to branch

### 1. Early return with `if`

Best when whole outputs differ:

```jsx
function OrderStatus({ order }) {
  if (!order) return <p>Select an order to see its status.</p>
  if (order.cancelled) return <p className="error">This order was cancelled.</p>

  return <p>Order {order.id} ships on {order.shipDate}.</p>
}
```

### 2. Ternary `? :` inside JSX

Best for choosing between two small pieces inline:

```jsx
<button>{isFollowing ? 'Unfollow' : 'Follow'}</button>
```

### 3. Logical `&&`

Best for "show this or nothing":

```jsx
{unread > 0 && <span className="badge">{unread}</span>}
```

### The `&&` zero trap

```jsx
{messages.length && <MessageList messages={messages} />}
```

When `messages.length` is `0`, the expression evaluates to `0`, and React renders
numbers — so a literal "0" appears on the page. Make the left side a real boolean:

```jsx
{messages.length > 0 && <MessageList messages={messages} />}
```

### Returning `null`

A component can render nothing by returning `null`. The component still exists (its
state is kept); it just produces no DOM.

```jsx
function Toast({ message }) {
  if (!message) return null
  return <div role="status" className="toast">{message}</div>
}
```

## Rendering lists with `map`

```jsx
const tasks = [
  { id: 't1', title: 'Write report', priority: 'high' },
  { id: 't2', title: 'Book tickets', priority: 'low' },
  { id: 't3', title: 'Call plumber', priority: 'high' },
]

function TaskList() {
  const urgent = tasks.filter(t => t.priority === 'high')
  return (
    <ul>
      {urgent.map(task => (
        <li key={task.id}>{task.title}</li>
      ))}
    </ul>
  )
}
```

`map` turns data into an array of elements, and React renders arrays in order.
`filter`, `toSorted` and `slice` shape the data first.

## Keys

Every element in an array needs a `key` prop that is:

- **Unique among its siblings** (not globally).
- **Stable** — the same item gets the same key on every render.

Good keys come from your data: database ids, UUIDs, slugs. If you create items on the
client, generate an id when the item is created (`crypto.randomUUID()`), not during
render.

### Why array index keys break things

```jsx
{todos.map((todo, index) => (
  <li key={index}>
    <input type="checkbox" /> {todo.title}
  </li>
))}
```

The checkbox here is uncontrolled — the DOM holds its checked state. Check the first
item, then delete it. The item that was second is now at index 0, so React reuses the
DOM node for key `0` — **including the checked checkbox** — for a different todo. The
UI now claims the wrong todo is checked.

Index keys are acceptable only when the list is static: never reordered, filtered,
inserted into or deleted from, and items have no state.

### Key placement

The key goes on the outermost element returned by the `map` callback:

```jsx
// ❌ key is inside the component, React can't see it at the array level
function Row({ item }) { return <li key={item.id}>{item.name}</li> }
items.map(item => <Row item={item} />)

// ✅
items.map(item => <Row key={item.id} item={item} />)
```

When one item produces multiple elements, use a keyed `Fragment`:

```jsx
import { Fragment } from 'react'

<dl>
  {terms.map(t => (
    <Fragment key={t.id}>
      <dt>{t.word}</dt>
      <dd>{t.meaning}</dd>
    </Fragment>
  ))}
</dl>
```

## Worked example: filterable, sortable contacts

```jsx
import { useState } from 'react'

const people = [
  { id: 'p1', name: 'Kiran', team: 'Design', online: true },
  { id: 'p2', name: 'Aditi', team: 'Engineering', online: false },
  { id: 'p3', name: 'Farhan', team: 'Engineering', online: true },
  { id: 'p4', name: 'Leela', team: 'Sales', online: false },
]

export default function Directory() {
  const [team, setTeam] = useState('All')
  const [onlineOnly, setOnlineOnly] = useState(false)

  const teams = ['All', ...new Set(people.map(p => p.team))]
  const visible = people
    .filter(p => team === 'All' || p.team === team)
    .filter(p => !onlineOnly || p.online)
    .toSorted((a, b) => a.name.localeCompare(b.name))

  return (
    <section>
      <div className="filters">
        {teams.map(t => (
          <button key={t} aria-pressed={t === team} onClick={() => setTeam(t)}>
            {t}
          </button>
        ))}
        <label>
          <input type="checkbox" checked={onlineOnly} onChange={e => setOnlineOnly(e.target.checked)} />
          Online only
        </label>
      </div>

      {visible.length === 0 ? (
        <p>No one matches these filters.</p>
      ) : (
        <ul>
          {visible.map(p => (
            <li key={p.id}>
              {p.name} <small>({p.team})</small>
              {p.online && <span className="dot" aria-label="online" />}
            </li>
          ))}
        </ul>
      )}
    </section>
  )
}
```

`toSorted` is a newer array method that returns a sorted copy. If you must support
older browsers, use `[...arr].sort(...)` instead. Never call `.sort()` directly on props
or state — it sorts in place.

## How It Actually Works

When a parent re-renders and returns a list, React has to match each new child element
with an existing child from the last render. That matching is how it decides whether
to **update** an existing component (keep its state and DOM node), **create** a new one,
or **destroy** an old one.

For a single child at a fixed position, matching is by position and type: if the
previous render had a `<ProductCard>` in slot 2 and so does this one, it's the same
instance. For arrays, position is unreliable because items move. So React builds a map
from **key → previous fiber**, then for each new element looks up its key:

- Key found and same `type` → reuse that fiber (state preserved), update its props,
  and move its DOM node if the order changed.
- Key not found → mount a new component.
- Any old keys left unused → unmount them (their state is destroyed, effects cleaned up).

With index keys, "key 0" always means "whatever is first", so deletion at the top makes
every remaining item update in place with a neighbour's data, and any state held in
those components or in the DOM (input text, checkbox state, focus, scroll position)
stays attached to the wrong item.

Keys also let you deliberately **reset** a component: `<Editor key={documentId} />`
throws away the editor's state whenever `documentId` changes, because React sees a
different key and treats it as a different component. Level 3's reconciliation lesson
revisits this trick.

Conditional rendering follows the same identity rules. `{isAdmin ? <Panel /> : <Panel />}`
keeps state when `isAdmin` flips (same type in the same position), while switching
between two *different* component types in that position unmounts one and mounts the
other.

## Common mistakes

- **Rendering `0`** from `count && ...`. Compare explicitly: `count > 0 &&`.
- **`Math.random()` or `crypto.randomUUID()` as a key during render** — a new key every
  render means every item remounts every time, which is slow and wipes state.
- **Duplicate keys** (e.g. using a name that isn't unique). React warns and may drop or
  duplicate items.
- **Sorting state in place** with `.sort()`, mutating it behind React's back.
- **Deeply nested ternaries.** Past two levels, move the logic into a variable or an
  early-return helper component.

## Exercise

Build a `Leaderboard`:

1. Start with an array of at least 6 players `{ id, name, score, country }`.
2. Show a table sorted by score (highest first); the top three rows get 🥇🥈🥉.
3. Add a country filter (buttons generated from the data) and a "Show top 3 only"
   checkbox.
4. Each row has an uncontrolled `<input placeholder="note">`. Type a note in the top
   row, then change a filter. Verify the note stays with the correct player. Then switch
   the keys to array indexes, repeat, and describe what goes wrong.
5. When no players match, show a friendly empty state instead of an empty table.
