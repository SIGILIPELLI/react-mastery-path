---
description: "Extract reusable stateful logic into custom React hooks, follow the Rules of Hooks, and understand why hook call order is how React tracks state."
---

# 05 · Custom Hooks

When two components need the same *stateful behaviour* — tracking online status,
syncing with `localStorage`, debouncing input — you don't share the component; you share
the logic. A **custom hook** is a function whose name starts with `use` and which calls
other hooks.

## Extracting a hook

Two components both track whether the browser is online:

```jsx
function StatusBar() {
  const [online, setOnline] = useState(navigator.onLine)
  useEffect(() => {
    const on = () => setOnline(true)
    const off = () => setOnline(false)
    window.addEventListener('online', on)
    window.addEventListener('offline', off)
    return () => {
      window.removeEventListener('online', on)
      window.removeEventListener('offline', off)
    }
  }, [])
  return <p>{online ? '✅ Online' : '❌ Offline'}</p>
}
```

Pull the logic out:

```jsx
export function useOnlineStatus() {
  const [online, setOnline] = useState(navigator.onLine)
  useEffect(() => {
    const on = () => setOnline(true)
    const off = () => setOnline(false)
    window.addEventListener('online', on)
    window.addEventListener('offline', off)
    return () => {
      window.removeEventListener('online', on)
      window.removeEventListener('offline', off)
    }
  }, [])
  return online
}

function StatusBar() {
  const online = useOnlineStatus()
  return <p>{online ? '✅ Online' : '❌ Offline'}</p>
}

function SaveButton({ onSave }) {
  const online = useOnlineStatus()
  return <button disabled={!online} onClick={onSave}>{online ? 'Save' : 'Reconnecting…'}</button>
}
```

Each component that calls `useOnlineStatus` gets **its own independent state**. Custom
hooks share *logic*, never state. (To share state, lift it or use context.)

(For subscribing to external sources like this, React also has `useSyncExternalStore`,
which handles some concurrent-rendering edge cases. The effect version is fine for
learning; Level 3 shows the store version.)

## Rules of Hooks

1. **Only call hooks at the top level** of a component or custom hook — never inside
   conditions, loops, nested functions, or after an early `return`.
2. **Only call hooks from React functions** — components or other custom hooks, not
   ordinary utility functions or event handlers.

```jsx
function Profile({ userId }) {
  if (!userId) return null          // ❌ early return before a hook
  const [tab, setTab] = useState('posts')
  ...
}

function Profile({ userId }) {
  const [tab, setTab] = useState('posts') // ✅ hooks first
  if (!userId) return null
  ...
}
```

The `use` prefix is what lets the linter (`eslint-plugin-react-hooks`) find hook calls
and enforce these rules. A function that doesn't call hooks shouldn't be named `use…`.

## A toolbox of useful hooks

### `useLocalStorage`

```jsx
export function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      const raw = localStorage.getItem(key)
      return raw !== null ? JSON.parse(raw) : initialValue
    } catch {
      return initialValue
    }
  })

  useEffect(() => {
    try {
      localStorage.setItem(key, JSON.stringify(value))
    } catch {
      // storage full or blocked (e.g. private mode) — keep working in memory
    }
  }, [key, value])

  return [value, setValue]
}

// usage — same shape as useState
const [theme, setTheme] = useLocalStorage('theme', 'light')
```

### `useDebouncedValue`

```jsx
export function useDebouncedValue(value, delay = 300) {
  const [debounced, setDebounced] = useState(value)
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay)
    return () => clearTimeout(id)
  }, [value, delay])
  return debounced
}

function Search() {
  const [query, setQuery] = useState('')
  const debouncedQuery = useDebouncedValue(query, 400)
  // fetch using debouncedQuery (lesson 6), not query
}
```

### `useToggle`

```jsx
export function useToggle(initial = false) {
  const [on, setOn] = useState(initial)
  const toggle = useCallback(() => setOn(v => !v), [])
  return [on, toggle, setOn]
}
```

`useCallback` keeps `toggle` the same function across renders, which matters if callers
pass it to memoized children or effect dependencies (lesson 8).

## Worked example: `useCountdown`

```jsx
import { useEffect, useRef, useState } from 'react'

export function useCountdown(seconds, { onDone } = {}) {
  const [remaining, setRemaining] = useState(seconds)
  const [running, setRunning] = useState(false)

  // Keep the latest onDone without restarting the interval when it changes.
  const onDoneRef = useRef(onDone)
  useEffect(() => {
    onDoneRef.current = onDone
  })

  useEffect(() => {
    if (!running) return
    const id = setInterval(() => {
      setRemaining(r => Math.max(r - 1, 0)) // pure updater: no other side effects in here
    }, 1000)
    return () => clearInterval(id)
  }, [running])

  useEffect(() => {
    if (running && remaining === 0) {
      setRunning(false)
      onDoneRef.current?.()
    }
  }, [running, remaining])

  return {
    remaining,
    running,
    start: () => setRunning(true),
    pause: () => setRunning(false),
    reset: () => {
      setRunning(false)
      setRemaining(seconds)
    },
  }
}

function EggTimer() {
  const { remaining, running, start, pause, reset } = useCountdown(180, {
    onDone: () => alert('Eggs are ready!'),
  })
  const mm = String(Math.floor(remaining / 60)).padStart(2, '0')
  const ss = String(remaining % 60).padStart(2, '0')
  return (
    <div>
      <p style={{ fontSize: '2rem' }}>{mm}:{ss}</p>
      {running ? <button onClick={pause}>Pause</button> : <button onClick={start}>Start</button>}
      <button onClick={reset}>Reset</button>
    </div>
  )
}
```

The "latest ref" trick (`onDoneRef`) solves a common problem: callers usually pass an
inline `onDone` arrow function, which is a new function every render. Putting it in the
interval effect's dependencies would restart the timer every render. Storing it in a
ref lets the effect always call the newest callback without depending on it.

Keep updater functions pure, too: React may call them twice in `StrictMode`, so the
"stop when we hit zero" logic lives in an effect rather than inside `setRemaining`.

(React 19.2 added `useEffectEvent`, aimed at exactly this "read the latest callback
from an effect" pattern. On earlier versions, the ref technique above is the standard
approach.)

## How It Actually Works

There is no special runtime support for custom hooks. React doesn't even know they
exist. When `EggTimer` calls `useCountdown`, which calls `useState` twice, `useRef`
once and `useEffect` three times, React simply sees `EggTimer` making six hook calls in
a row. The custom hook is inlined into the component as far as React is concerned.

React identifies each hook by its **position in the call sequence** for that component.
Internally, the fiber has a linked list of hook objects; each hook call during render
moves a cursor one step along the list and reads or writes that node. On the first
render the list is built; on every re-render it's walked in the same order.

Now imagine:

```jsx
if (isAdmin) useEffect(...)   // hook #2 only sometimes
const [name] = useState('')   // hook #2 or #3?
```

If `isAdmin` flips from `true` to `false`, the `useState` call becomes the second call.
React hands it the node that was the effect's slot — wrong type, wrong data. React
detects many of these mismatches and throws ("Rendered fewer hooks than expected"), but
the root cause is always the same: **position is the only identity a hook has**. That's
why the Rules of Hooks exist, and why a naming convention (`use…`) is enough for a
linter to enforce them.

It's also why custom hooks don't share state: each call site creates its own nodes in
its own component's list.

## Common mistakes

- **Expecting shared state between two components using the same hook.** Each call is
  independent.
- **Conditional hook calls**, including after an early return.
- **Returning new objects/functions every render** from a hook, then callers putting
  them in effect deps and getting infinite re-runs. Memoize what callers are likely to
  depend on.
- **Naming non-hooks `useSomething`**, or hooks without `use` — both confuse the linter.
- **Over-extracting.** A hook used once that just wraps one `useState` adds indirection
  without value.

## Exercise

Write and use these hooks:

1. `useMediaQuery(query)` → boolean, using `window.matchMedia` and its `change` event,
   with cleanup. Use it to switch a layout between list and grid at 700px.
2. `usePrevious(value)` → the value from the previous render (hint: a ref updated in an
   effect). Use it to show "▲" or "▼" when a stock price changes.
3. `useClickOutside(ref, handler)` that calls `handler` when a `pointerdown` happens
   outside the element. Use it to close a dropdown menu.
4. Deliberately break the Rules of Hooks once (put `usePrevious` in an `if`), observe
   the lint error and runtime behaviour, then fix it and explain the cause in one
   sentence.
