---
description: "Move complex React state logic into a pure reducer with useReducer, design action types, handle state transitions, and test reducers without rendering."
---

# 03 · useReducer

When a component's event handlers each contain their own slice of update logic, it
gets hard to see *all the ways state can change*. `useReducer` gathers that logic into
one pure function: given the current state and an action describing what happened, it
returns the next state.

## Anatomy

```jsx
import { useReducer } from 'react'

function reducer(state, action) {
  switch (action.type) {
    case 'incremented':
      return { ...state, count: state.count + state.step }
    case 'decremented':
      return { ...state, count: state.count - state.step }
    case 'step_changed':
      return { ...state, step: action.step }
    case 'reset':
      return { ...state, count: 0 }
    default:
      throw new Error(`Unknown action: ${action.type}`)
  }
}

export default function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0, step: 1 })

  return (
    <>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: 'decremented' })}>−</button>
      <button onClick={() => dispatch({ type: 'incremented' })}>+</button>
      <input type="number" value={state.step}
             onChange={e => dispatch({ type: 'step_changed', step: Number(e.target.value) })} />
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
    </>
  )
}
```

- `useReducer(reducer, initialState)` returns the current state and a `dispatch`
  function.
- Event handlers no longer say *how* to change state; they report *what happened*.
- The reducer is a plain function outside the component. It must be **pure**: no
  mutation, no API calls, no randomness, no reading the current time.

A third argument, `useReducer(reducer, arg, init)`, computes the initial state lazily as
`init(arg)` — handy for reading `localStorage` once.

## Naming actions

Name actions after **events**, not setters: `'item_added'`, `'filter_changed'`,
`'checkout_started'`, rather than `'setItems'`. This keeps the reducer a record of what
the user did, and lets one action update several fields consistently.

## When to choose `useReducer` over `useState`

| Signal | Lean towards |
|---|---|
| One or two independent values | `useState` |
| Several values that change together | `useReducer` |
| Next state depends on previous state in complex ways | `useReducer` |
| Many handlers with similar update code | `useReducer` |
| You want to unit test state transitions without rendering | `useReducer` |

They are interchangeable in power; `useState` is actually implemented as a reducer
internally. Choose for readability.

## Worked example: a cart with rules

```jsx
import { useReducer } from 'react'

const initialCart = { items: [], coupon: null, error: null }

const COUPONS = { SAVE10: 0.1, FESTIVE20: 0.2 }

export function cartReducer(state, action) {
  switch (action.type) {
    case 'item_added': {
      const existing = state.items.find(i => i.sku === action.product.sku)
      const items = existing
        ? state.items.map(i => (i.sku === existing.sku ? { ...i, qty: i.qty + 1 } : i))
        : [...state.items, { ...action.product, qty: 1 }]
      return { ...state, items, error: null }
    }
    case 'quantity_changed': {
      if (action.qty <= 0) {
        return { ...state, items: state.items.filter(i => i.sku !== action.sku) }
      }
      return {
        ...state,
        items: state.items.map(i => (i.sku === action.sku ? { ...i, qty: Math.min(action.qty, 10) } : i)),
      }
    }
    case 'coupon_applied': {
      const code = action.code.trim().toUpperCase()
      if (!(code in COUPONS)) return { ...state, error: `"${code}" is not a valid coupon.` }
      return { ...state, coupon: code, error: null }
    }
    case 'cart_cleared':
      return initialCart
    default:
      throw new Error(`Unhandled action ${action.type}`)
  }
}

export function cartTotals(state) {
  const subtotal = state.items.reduce((s, i) => s + i.price * i.qty, 0)
  const discount = state.coupon ? subtotal * COUPONS[state.coupon] : 0
  return { subtotal, discount, total: subtotal - discount }
}

const products = [
  { sku: 'MUG', name: 'Mug', price: 350 },
  { sku: 'TEE', name: 'T-shirt', price: 799 },
]

export default function Shop() {
  const [cart, dispatch] = useReducer(cartReducer, initialCart)
  const { subtotal, discount, total } = cartTotals(cart)

  function handleCoupon(e) {
    e.preventDefault()
    dispatch({ type: 'coupon_applied', code: new FormData(e.currentTarget).get('code') })
  }

  return (
    <div>
      {products.map(p => (
        <button key={p.sku} onClick={() => dispatch({ type: 'item_added', product: p })}>
          Add {p.name} (₹{p.price})
        </button>
      ))}

      <ul>
        {cart.items.map(i => (
          <li key={i.sku}>
            {i.name}
            <input type="number" min="0" max="10" value={i.qty}
                   onChange={e => dispatch({ type: 'quantity_changed', sku: i.sku, qty: Number(e.target.value) })} />
          </li>
        ))}
      </ul>

      <form onSubmit={handleCoupon}>
        <input name="code" placeholder="Coupon" />
        <button>Apply</button>
      </form>
      {cart.error && <p role="alert">{cart.error}</p>}

      <p>Subtotal ₹{subtotal} · Discount ₹{discount.toFixed(0)} · <strong>Total ₹{total.toFixed(0)}</strong></p>
      <button onClick={() => dispatch({ type: 'cart_cleared' })}>Clear cart</button>
    </div>
  )
}
```

Every business rule — merging duplicates, capping quantity at 10, removing at zero,
validating coupons — lives in `cartReducer`. The component is just wiring.

## Testing a reducer

Because it's a pure function, you test it with plain assertions (Vitest syntax — lesson 9
sets it up):

```js
import { describe, it, expect } from 'vitest'
import { cartReducer } from './Shop'

describe('cartReducer', () => {
  const mug = { sku: 'MUG', name: 'Mug', price: 350 }

  it('merges repeated adds into quantity', () => {
    let s = cartReducer({ items: [], coupon: null, error: null }, { type: 'item_added', product: mug })
    s = cartReducer(s, { type: 'item_added', product: mug })
    expect(s.items).toEqual([{ ...mug, qty: 2 }])
  })

  it('rejects unknown coupons without changing the cart', () => {
    const start = { items: [], coupon: null, error: null }
    const s = cartReducer(start, { type: 'coupon_applied', code: 'bogus' })
    expect(s.coupon).toBeNull()
    expect(s.error).toMatch(/not a valid coupon/)
  })
})
```

## How It Actually Works

`useReducer` occupies one hook slot, like `useState`. Calling `dispatch(action)` pushes
the action onto that hook's update queue and schedules a render. During the next render,
React takes the current state and folds every queued action through your reducer in
order: `state = reducer(state, action)`. The result is returned from `useReducer`.

Two consequences:

1. **The reducer runs during rendering**, not when you call `dispatch`. That's why it
   must be pure, and why in `StrictMode` React calls it twice to catch impurity.
   A reducer that pushes into an array or calls `fetch` would do so twice.
2. **`dispatch` is stable**: it's the same function for the lifetime of the component.
   You can pass it deep into the tree (often through context, next lesson) or list it as
   an effect dependency without causing re-runs.

`useState` is literally a `useReducer` with a built-in reducer:
`(state, action) => typeof action === 'function' ? action(state) : action`. That's where
the "updater function" behaviour comes from.

If the reducer returns the **same** state object (`Object.is` equal), React bails out
and won't re-render children. So returning `state` unchanged for an ignored action is a
real optimisation, and mutating `state` and returning it is a real bug — React will see
"no change".

## Common mistakes

- **Mutating state in the reducer** (`state.items.push(...)`; `return state`). React
  sees the same reference and may not re-render.
- **Side effects in the reducer** — fetching, `localStorage.setItem`, `Math.random()`,
  `crypto.randomUUID()`. Generate ids and timestamps in the event handler and put them
  in the action payload.
- **Reading `state` right after `dispatch`.** Like setters, dispatch doesn't change the
  variable you hold; the new state arrives on the next render.
- **Silently ignoring unknown actions.** Throwing (or logging) in `default` catches
  typos like `'item_aded'` immediately.
- **One reducer for the entire app.** Keep reducers close to the feature they serve.

## Exercise

Build a multi-step signup wizard with `useReducer`:

1. State: `{ step, data: { name, email, plan }, errors, status }` where `status` is
   `'editing' | 'submitting' | 'done'`.
2. Actions: `field_changed`, `next_clicked`, `back_clicked`, `submitted`,
   `submit_succeeded`. `next_clicked` validates the current step and either advances
   or records errors — all inside the reducer.
3. The async "submit" (simulate with `setTimeout`) lives in the event handler; it
   dispatches `submitted` before and `submit_succeeded` after.
4. Write at least four Vitest tests for the reducer alone, including one proving it
   never mutates the previous state (freeze the input with `Object.freeze`).
