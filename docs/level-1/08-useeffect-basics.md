---
description: "Use useEffect to synchronize components with timers, the document, storage and subscriptions, with correct dependencies and cleanup, and learn when not to."
---

# 08 · useEffect Basics

Rendering must be pure: take props and state, return JSX, touch nothing else. But apps
need to do things outside React — set the page title, start a timer, subscribe to a
WebSocket, save to `localStorage`. `useEffect` is where that code goes. Think of an
effect as **synchronizing** some external system with your current props and state.

## Shape of an effect

```jsx
import { useEffect, useState } from 'react'

function PageTitle() {
  const [unread, setUnread] = useState(3)

  useEffect(() => {
    document.title = unread > 0 ? `(${unread}) Inbox` : 'Inbox'
  }, [unread])

  return <button onClick={() => setUnread(0)}>Mark all read</button>
}
```

- The first argument is the **setup** function. React runs it *after* the render has
  been committed to the screen.
- The second is the **dependency array**: the reactive values the effect reads. React
  re-runs the effect only when one of them changed since last time (compared with
  `Object.is`).

| Dependencies | Effect runs |
|---|---|
| `[a, b]` | after mount, and after any render where `a` or `b` changed |
| `[]` | after mount only (and again on remount) |
| omitted | after every render |

## Cleanup

If setup starts something, return a function that stops it:

```jsx
function Clock() {
  const [now, setNow] = useState(() => new Date())

  useEffect(() => {
    const id = setInterval(() => setNow(new Date()), 1000)
    return () => clearInterval(id)
  }, [])

  return <time>{now.toLocaleTimeString()}</time>
}
```

React calls cleanup **before re-running the effect** with new dependencies, and when
the component **unmounts**. Without it, every remount would leave another interval
running.

In development, `StrictMode` deliberately mounts, unmounts and remounts each component
once. If you see your effect run twice, that's the test: with correct cleanup, the
visible behaviour is identical. If it breaks, the bug would have hit real users later
(for example when navigating away and back).

## Dependencies are not optional

Every prop, state variable, or value computed from them that the effect reads must be in
the array. The `react-hooks/exhaustive-deps` lint rule (enabled in the Vite template)
checks this for you. Don't silence it; fix the cause.

```jsx
function Countdown({ from }) {
  const [left, setLeft] = useState(from)

  useEffect(() => {
    if (left <= 0) return
    const id = setTimeout(() => setLeft(left - 1), 1000)
    return () => clearTimeout(id)
  }, [left])

  return <p>{left > 0 ? `${left}s` : 'Done!'}</p>
}
```

## Worked example: persist to localStorage

```jsx
import { useEffect, useState } from 'react'

export default function Notepad() {
  const [text, setText] = useState(() => localStorage.getItem('notepad') ?? '')
  const [savedAt, setSavedAt] = useState(null)

  useEffect(() => {
    const id = setTimeout(() => {
      localStorage.setItem('notepad', text)
      setSavedAt(new Date())
    }, 500) // debounce: save 500ms after typing stops
    return () => clearTimeout(id)
  }, [text])

  return (
    <div>
      <textarea value={text} onChange={e => setText(e.target.value)} rows={8} cols={50} />
      <p>{savedAt ? `Saved at ${savedAt.toLocaleTimeString()}` : 'Not saved yet'}</p>
    </div>
  )
}
```

Each keystroke changes `text`, so React cleans up the pending timeout and sets a new
one. Only when typing pauses for 500 ms does a save actually happen. The cleanup is what
turns a naive "save on every change" into a debounce.

## Worked example: subscribe to a browser event

```jsx
function WindowWidth() {
  const [width, setWidth] = useState(window.innerWidth)

  useEffect(() => {
    function onResize() {
      setWidth(window.innerWidth)
    }
    window.addEventListener('resize', onResize)
    return () => window.removeEventListener('resize', onResize)
  }, [])

  return <p>Window is {width}px wide</p>
}
```

The function passed to `removeEventListener` must be the *same* function that was
added, which is why it's defined inside the effect.

## You might not need an effect

Effects are for synchronizing with things **outside** React. Many beginner effects
should just be calculations:

```jsx
// ❌ extra state + effect + an extra render with stale data
const [fullName, setFullName] = useState('')
useEffect(() => {
  setFullName(`${first} ${last}`)
}, [first, last])

// ✅ derive during render
const fullName = `${first} ${last}`
```

Likewise, code that responds to a user action ("when the form is submitted, send a
request") belongs in the **event handler**, not in an effect watching some flag.
Ask: *is this caused by the component being on screen, or by a specific event?* Only
the former is an effect.

Data fetching with effects is covered properly in Level 2, including race conditions.

## How It Actually Works

During rendering, `useEffect` doesn't run anything. It records the setup function and
dependency array in the component's hook slot (by call order, like `useState`) and marks
the fiber as having an effect to process.

After React commits the DOM changes, it runs effects in a later pass — usually after the
browser has painted, so effects don't delay the first visual frame. For each effect it
compares the new dependency array with the one stored from the last commit, item by
item with `Object.is`. If any item differs (or there's no array), it runs the previous
cleanup (if any) and then the new setup, storing the returned cleanup for next time.

The key insight is that **each render has its own effect**. The setup function is a
closure created during a particular render, so it sees *that* render's props and state.
If you leave a value out of the dependency array, React keeps the old effect — along
with the old closure — which then keeps reading old values. That's a **stale closure**:

```jsx
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000) // captures count = 0 forever
  return () => clearInterval(id)
}, []) // ❌ count missing
```

The interval callback always sees the `count` from the first render, so the counter
goes to 1 and stays there. The fixes: add `count` to deps (the interval is recreated
each second), or better, remove the dependency by using the updater form
`setCount(c => c + 1)` which doesn't read `count` from the closure.

`useLayoutEffect` has the same API but runs synchronously after DOM mutation and
**before** the browser paints. It exists for measuring layout (e.g. positioning a
tooltip) without a visible flicker; it blocks painting, so use `useEffect` by default.

## Common mistakes

- **Missing dependencies** → stale values. Trust the linter.
- **Objects or functions created during render as dependencies** — they're new every
  render, so the effect runs every time. Move them inside the effect or compute from
  primitives.
- **Setting state unconditionally in an effect that depends on that state** → infinite
  loop.
- **Forgetting cleanup** for intervals, listeners and subscriptions → leaks and
  duplicate handlers.
- **Using an effect to derive data or to react to a button click.** Compute during
  render; handle clicks in handlers.
- **`async` setup function.** `useEffect(async () => ...)` returns a promise, not a
  cleanup. Define an async function inside and call it.

## Exercise

Build a `FocusTimer` (Pomodoro-style):

1. State: remaining seconds (start at 25 × 60), `running` (boolean), and completed
   sessions count.
2. An effect that ticks once per second only while `running` is true, with proper
   cleanup. Use the updater form so the interval doesn't need `remaining` as a dependency.
3. When remaining hits 0, stop, increment sessions and reset to 25:00 — do this in the
   updater/tick logic, not in a second effect that watches `remaining`.
4. Keep `document.title` in sync with the remaining time (e.g. `12:04 · Focus`).
5. Persist the completed-sessions count to `localStorage` and restore it on load.
6. Verify in dev with `StrictMode` that the timer doesn't run at double speed.
