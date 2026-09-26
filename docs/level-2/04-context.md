---
description: "Share values across a React tree with createContext and useContext, combine context with reducers, and avoid unnecessary re-renders from context updates."
---

# 04 · Context

Passing a prop through five components that don't use it, just so the sixth can, is
called **prop drilling**. Context lets a component provide a value that any descendant
can read directly, however deep.

## The three steps

```jsx
import { createContext, useContext, useState } from 'react'

// 1. Create
const ThemeContext = createContext('light')

// 2. Provide
export default function App() {
  const [theme, setTheme] = useState('light')
  return (
    <ThemeContext.Provider value={theme}>
      <button onClick={() => setTheme(t => (t === 'light' ? 'dark' : 'light'))}>Toggle</button>
      <Page />
    </ThemeContext.Provider>
  )
}

// 3. Consume — anywhere below the provider
function Page() {
  return <section><Toolbar /></section>
}

function Toolbar() {
  const theme = useContext(ThemeContext)
  return <div className={`toolbar toolbar--${theme}`}>Current theme: {theme}</div>
}
```

`Page` never mentions theme. `Toolbar` reads it directly.

The argument to `createContext` is the **default value**, used only when there is no
provider above the consumer. In React 19 you can also render the context itself as the
provider (`<ThemeContext value={theme}>`); `<ThemeContext.Provider>` works in both 18
and 19, so this course uses it.

## A context module pattern

In real code, wrap the context in a provider component and a custom hook, and export
only those:

```jsx
// auth-context.jsx
import { createContext, useContext, useMemo, useState } from 'react'

const AuthContext = createContext(null)

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null)

  const value = useMemo(
    () => ({
      user,
      login: name => setUser({ name }),
      logout: () => setUser(null),
    }),
    [user],
  )

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>
}

export function useAuth() {
  const ctx = useContext(AuthContext)
  if (!ctx) throw new Error('useAuth must be used inside <AuthProvider>')
  return ctx
}
```

```jsx
// main.jsx
<AuthProvider>
  <App />
</AuthProvider>

// anywhere
function UserMenu() {
  const { user, logout } = useAuth()
  if (!user) return <a href="/login">Sign in</a>
  return <button onClick={logout}>Sign out {user.name}</button>
}
```

The `null` default plus a throwing hook turns "forgot the provider" into a clear error
instead of a confusing `undefined`. This is a demo of the pattern only: real
authentication needs a server, and client state alone never secures anything.

## Context + reducer: app-level state without a library

```jsx
import { createContext, useContext, useReducer } from 'react'

const TodosContext = createContext(null)
const TodosDispatchContext = createContext(null)

function todosReducer(todos, action) {
  switch (action.type) {
    case 'added':
      return [...todos, { id: action.id, text: action.text, done: false }]
    case 'toggled':
      return todos.map(t => (t.id === action.id ? { ...t, done: !t.done } : t))
    case 'deleted':
      return todos.filter(t => t.id !== action.id)
    default:
      throw new Error(action.type)
  }
}

export function TodosProvider({ children }) {
  const [todos, dispatch] = useReducer(todosReducer, [])
  return (
    <TodosContext.Provider value={todos}>
      <TodosDispatchContext.Provider value={dispatch}>{children}</TodosDispatchContext.Provider>
    </TodosContext.Provider>
  )
}

export const useTodos = () => useContext(TodosContext)
export const useTodosDispatch = () => useContext(TodosDispatchContext)
```

Why two contexts? `dispatch` never changes, so components that only *dispatch* (an
"Add todo" form) subscribe to a context whose value never changes and won't re-render
when the list changes.

```jsx
function AddTodo() {
  const dispatch = useTodosDispatch()
  const [text, setText] = useState('')
  return (
    <form onSubmit={e => {
      e.preventDefault()
      dispatch({ type: 'added', id: crypto.randomUUID(), text })
      setText('')
    }}>
      <input value={text} onChange={e => setText(e.target.value)} />
    </form>
  )
}
```

## Worked example: overriding context in a subtree

Providers can be nested; the nearest one wins. That makes context great for "local
defaults":

```jsx
const DensityContext = createContext('comfortable')

function Table({ rows }) {
  const density = useContext(DensityContext)
  return <table className={`table table--${density}`}>{/* ... */}</table>
}

function ReportPage() {
  return (
    <>
      <Table rows={summary} />                   {/* comfortable */}
      <DensityContext.Provider value="compact">
        <Table rows={transactions} />            {/* compact */}
      </DensityContext.Provider>
    </>
  )
}
```

## What context is good (and bad) for

Good fits: theme, locale, current user, feature flags, a design-system's configuration,
compound components (lesson 1), and dispatch functions.

Poor fits: rapidly changing values read by many components (mouse position, text being
typed), or large stores where each consumer needs a small slice. Every consumer
re-renders on every change, even if it only uses one field. That's where external stores
(Zustand, Redux Toolkit — Level 3) or splitting contexts come in.

Also consider whether you need context at all: often passing `children` (composition)
removes the drilling. If `Layout` passes `user` to `Header` only so `Header` can pass
it to `Avatar`, let the top level render `<Layout header={<Header avatar={<Avatar user={user} />} />}>`
instead.

## How It Actually Works

A provider stores its `value` on its fiber. When a component calls
`useContext(SomeContext)`, React walks up the fiber tree at render time to the nearest
matching provider and reads its current value. It also records a **dependency**: this
fiber reads that context.

When a provider re-renders with a new `value`, React compares it with the previous one
using `Object.is`. If it changed, React scans the provider's subtree for fibers that
recorded a dependency on this context and schedules them to re-render — **even if a
component in between is wrapped in `memo`** or otherwise skipped. Context "tunnels"
past memoized parents.

This explains two important rules:

1. **Stabilise object values.** `value={{ user, login }}` creates a new object every
   time the provider's component renders, so every consumer re-renders even when `user`
   didn't change. `useMemo` (as in `AuthProvider` above) keeps the same object until its
   dependencies change.
2. **Context has no selectors.** A consumer that only uses `login` still re-renders when
   `user` changes, because the value object as a whole changed. Split contexts by how
   often things change, or use an external store with selectors.

## Common mistakes

- **Consuming outside a provider** and silently getting the default value. Use a
  `null` default and a throwing custom hook.
- **New object/array `value` every render** → all consumers re-render needlessly.
- **One giant `AppContext`** holding everything; any change re-renders everything that
  reads any part of it.
- **Using context to avoid passing props one or two levels.** Props are explicit and
  easy to trace; reach for context when drilling genuinely hurts.
- **Putting the provider inside the component that uses it.** `useContext` looks
  *upward*; a provider in the same component's JSX doesn't affect that component's own
  `useContext` call.

## Exercise

Build a `NotificationsProvider`:

1. It exposes `notify(message, { tone, timeout })` and renders a stack of toasts in the
   corner (render them inside the provider, after `children`).
2. Use a reducer for the toast list; each toast auto-dismisses after its timeout via an
   effect in a `Toast` component with cleanup.
3. Split into two contexts so components that only call `notify` never re-render when
   toasts appear or disappear. Verify this by adding a `console.count` in such a
   component.
4. Provide a `useNotify()` hook that throws outside the provider.
5. Trigger notifications from three unrelated components at different depths.
