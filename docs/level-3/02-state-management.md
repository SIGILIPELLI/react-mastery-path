---
description: "Compare Zustand and Redux Toolkit for client state in React, use selectors to limit re-renders, and learn how useSyncExternalStore powers external stores."
---

# 02 · Client State: Zustand vs Redux Toolkit

Local state, lifting and context take you surprisingly far. You reach for a dedicated
store when shared client state gets large, is updated from many places, or when context
re-renders become a measured problem. This lesson compares two popular, very different
options honestly, and shows the mechanism both rely on.

First, a distinction that removes most of the need for a store: **server state** (data
that lives on a server — users, orders, search results) is not the same as **client
state** (UI state that exists only in the browser — the open panel, a draft, the
selected tool). Server state is best handled by a caching library (next lesson). What's
left for a store is usually small.

## Option 1: Zustand

A minimal store library: a hook created from a function that describes state and
actions.

```bash
npm install zustand
```

```js
// stores/useCartStore.js
import { create } from 'zustand'

export const useCartStore = create((set, get) => ({
  items: [],
  addItem: product =>
    set(state => {
      const existing = state.items.find(i => i.id === product.id)
      return {
        items: existing
          ? state.items.map(i => (i.id === product.id ? { ...i, qty: i.qty + 1 } : i))
          : [...state.items, { ...product, qty: 1 }],
      }
    }),
  removeItem: id => set(state => ({ items: state.items.filter(i => i.id !== id) })),
  clear: () => set({ items: [] }),
  total: () => get().items.reduce((sum, i) => sum + i.price * i.qty, 0),
}))
```

```jsx
function CartBadge() {
  const count = useCartStore(state => state.items.length)   // selector
  return <span className="badge">{count}</span>
}

function AddToCart({ product }) {
  const addItem = useCartStore(state => state.addItem)
  return <button onClick={() => addItem(product)}>Add</button>
}
```

- No provider needed; the store is a module-level singleton.
- `set` merges the returned partial state (one level deep).
- Each component passes a **selector** and re-renders only when the selected value
  changes. `AddToCart` selects a function that never changes, so it never re-renders
  because of cart updates.

Careful with selectors that build new objects:

```js
// ❌ new object every time → re-renders on every store change
const { items, clear } = useCartStore(s => ({ items: s.items, clear: s.clear }))
```

Select separately, or use Zustand's `useShallow` helper (import path varies by version —
check the docs) to compare the result shallowly.

## Option 2: Redux Toolkit (RTK)

Redux keeps all state in one store changed only by dispatched actions through reducers.
Redux Toolkit is the official, modern way to write Redux; it removes most of the old
boilerplate.

```bash
npm install @reduxjs/toolkit react-redux
```

```js
// features/cart/cartSlice.js
import { createSlice } from '@reduxjs/toolkit'

const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [] },
  reducers: {
    itemAdded(state, action) {
      const existing = state.items.find(i => i.id === action.payload.id)
      if (existing) existing.qty += 1                 // looks like mutation…
      else state.items.push({ ...action.payload, qty: 1 })
    },
    itemRemoved(state, action) {
      state.items = state.items.filter(i => i.id !== action.payload)
    },
    cleared(state) {
      state.items = []
    },
  },
})

export const { itemAdded, itemRemoved, cleared } = cartSlice.actions
export default cartSlice.reducer

export const selectCartCount = state => state.cart.items.length
export const selectCartTotal = state =>
  state.cart.items.reduce((sum, i) => sum + i.price * i.qty, 0)
```

```js
// store.js
import { configureStore } from '@reduxjs/toolkit'
import cartReducer from './features/cart/cartSlice'

export const store = configureStore({ reducer: { cart: cartReducer } })
```

```jsx
// main.jsx
import { Provider } from 'react-redux'
import { store } from './store'

<Provider store={store}><App /></Provider>

// components
import { useDispatch, useSelector } from 'react-redux'
function CartBadge() {
  const count = useSelector(selectCartCount)
  return <span className="badge">{count}</span>
}
function AddToCart({ product }) {
  const dispatch = useDispatch()
  return <button onClick={() => dispatch(itemAdded(product))}>Add</button>
}
```

The "mutating" reducer code is safe: RTK wraps reducers with **Immer**, which records
your changes on a draft and produces a new immutable state. (It's only safe inside
`createSlice`/`createReducer` — mutating in a plain reducer is still a bug.)

`useSelector` also re-renders only when its selected value changes (compared with
`===` by default). For derived data that returns new arrays/objects, RTK's
`createSelector` memoizes the result.

## An honest comparison

| | Zustand | Redux Toolkit |
|---|---|---|
| Setup | one `create` call | store + slices + `Provider` |
| Structure | free-form; you decide conventions | opinionated: slices, actions, reducers |
| Update style | call actions directly | dispatch action objects |
| DevTools / time travel | via middleware | excellent, built in |
| Server data | pair with TanStack Query | RTK Query included |
| Best fit | small–medium apps, quick shared UI state | large teams wanting strict conventions and traceability |
| Cost | easy to create inconsistent patterns at scale | more concepts and ceremony up front |

Neither is "better". Many apps need **neither**: context for low-frequency values plus
TanStack Query for server data covers a lot. Choose a store when you can name the
problem it solves (frequent shared updates, cross-cutting actions, debugging needs).

Other credible options exist (Jotai's atomic model, MobX's observables, XState for
explicit state machines). The selector concept below applies to most of them.

## Worked example: a selector that avoids re-renders

```jsx
const useEditorStore = create(set => ({
  tool: 'pen',
  color: '#222',
  strokes: [],
  setTool: tool => set({ tool }),
  addStroke: stroke => set(s => ({ strokes: [...s.strokes, stroke] })),
}))

function Toolbar() {
  const tool = useEditorStore(s => s.tool)
  const setTool = useEditorStore(s => s.setTool)
  console.count('Toolbar render')
  return ['pen', 'eraser'].map(t => (
    <button key={t} aria-pressed={tool === t} onClick={() => setTool(t)}>{t}</button>
  ))
}

function Canvas() {
  const strokes = useEditorStore(s => s.strokes)
  // draws strokes...
}
```

Drawing adds strokes rapidly. `Toolbar` doesn't select `strokes`, so it never
re-renders while you draw. With a single context holding the whole editor state, it
would re-render on every stroke.

## How It Actually Works

Both libraries keep state **outside React** in a plain JavaScript object with a
subscribe mechanism: `subscribe(listener)`, `getState()`, and a way to replace state
and notify listeners. React components connect through the built-in hook
**`useSyncExternalStore(subscribe, getSnapshot)`**. You can build a tiny store yourself:

```jsx
import { useSyncExternalStore } from 'react'

function createStore(initial) {
  let state = initial
  const listeners = new Set()
  return {
    getState: () => state,
    setState: fn => {
      state = fn(state)
      listeners.forEach(l => l())
    },
    subscribe: l => {
      listeners.add(l)
      return () => listeners.delete(l)
    },
  }
}

const counterStore = createStore({ count: 0 })

function useStore(store, selector) {
  return useSyncExternalStore(store.subscribe, () => selector(store.getState()))
}

function Count() {
  const count = useStore(counterStore, s => s.count)
  return <button onClick={() => counterStore.setState(s => ({ count: s.count + 1 }))}>{count}</button>
}
```

On each store change, React calls every subscribed component's `getSnapshot` (here, the
selector) and compares the result with the previous snapshot using `Object.is`. Only
components whose snapshot changed re-render. That's the entire "selector" optimisation,
and it's why a selector that returns a new object every call causes a re-render every
time (and React may warn that the result of `getSnapshot` should be cached).

`useSyncExternalStore` also solves **tearing**: during concurrent rendering, React could
otherwise render part of the tree with one store value and part with another if the
store changes mid-render. The hook detects this and forces a consistent synchronous
re-render. That's why stores should use it rather than `useState` + `useEffect`
subscriptions.

## Common mistakes

- **Putting server data in a client store** and re-implementing caching, loading flags
  and refetching by hand.
- **Selecting the whole store** (`useCartStore()` with no selector, `useSelector(s => s)`)
  → re-render on every change.
- **Selectors returning fresh objects/arrays** without shallow comparison or memoization.
- **Mutating state in Zustand's `set` or a plain Redux reducer** — only RTK's Immer-wrapped
  reducers allow the mutation syntax.
- **Global store for everything**, including form drafts and hover state that should be
  local.

## Exercise

1. Implement the same "notes" feature (list, add, delete, toggle pin, filter pinned) three
   times: with context + `useReducer`, with Zustand, and with Redux Toolkit.
2. Add a `console.count` in a component that only displays the number of pinned notes.
   Compare how often it renders in each version when you edit note text.
3. Build the `createStore` + `useSyncExternalStore` version from this lesson and make it
   pass the same behaviour.
4. Write a short recommendation (5–8 sentences) for which you'd pick for a 3-person team
   building an internal admin app, and why.
