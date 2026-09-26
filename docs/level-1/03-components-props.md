---
description: "Split UI into reusable function components, pass data with props, set defaults, and understand why props are read-only snapshots."
---

# 03 · Components & Props

A component is a function. **Props** are its arguments. That is nearly the whole model:
parents decide what data a child gets, and the child turns that data into UI.

## Passing and reading props

```jsx
function ProductCard(props) {
  return (
    <div className="card">
      <h3>{props.name}</h3>
      <p>₹{props.price}</p>
    </div>
  )
}

export default function App() {
  return (
    <>
      <ProductCard name="Desk lamp" price={1499} />
      <ProductCard name="Monitor arm" price={2899} />
    </>
  )
}
```

React collects all attributes into one object and passes it as the first argument. The
usual style destructures it right in the signature:

```jsx
function ProductCard({ name, price, inStock = true }) {
  return (
    <div className="card">
      <h3>{name}</h3>
      <p>₹{price}</p>
      {!inStock && <p className="muted">Out of stock</p>}
    </div>
  )
}
```

`inStock = true` is an ordinary JavaScript default parameter. It applies when the prop
is missing or explicitly `undefined` — but not when it is `null` or `false`.

## Any value can be a prop

Strings, numbers, booleans, arrays, objects, functions and even JSX:

```jsx
<UserList
  users={[{ id: 1, name: 'Ravi' }]}
  onSelect={id => console.log('picked', id)}
  emptyState={<p>No users yet.</p>}
/>
```

Passing functions is how children talk back to parents (lesson 5). Passing JSX is the
basis of composition (Level 2 lesson 1).

## The `children` prop

Whatever you put between the opening and closing tag arrives as `children`:

```jsx
function Panel({ title, children }) {
  return (
    <section className="panel">
      <h2>{title}</h2>
      <div className="panel-body">{children}</div>
    </section>
  )
}

<Panel title="Shipping">
  <p>Free over ₹999.</p>
  <p>Arrives in 3–5 days.</p>
</Panel>
```

`Panel` doesn't need to know what it wraps. That makes it reusable.

## Spreading props

```jsx
function Avatar({ size = 48, ...imgProps }) {
  return <img width={size} height={size} className="avatar" {...imgProps} />
}

<Avatar src="/me.png" alt="Profile picture" size={64} />
```

`...imgProps` gathers everything not named; `{...imgProps}` spreads it back onto the
`<img>`. Useful for thin wrappers, but don't spread blindly everywhere — it hides which
props a component actually uses.

## Props are read-only

A component must never modify its props:

```jsx
function Bad({ user }) {
  user.name = user.name.trim()   // ❌ mutates the parent's object
  return <p>{user.name}</p>
}

function Good({ user }) {
  const name = user.name.trim()  // ✅ derive a new value
  return <p>{name}</p>
}
```

When a component needs a value to *change over time*, that's state (next lesson), not
props.

## Worked example: a pricing table from data

```jsx
const plans = [
  { id: 'free', name: 'Free', price: 0, features: ['1 project', 'Community support'] },
  { id: 'pro', name: 'Pro', price: 499, features: ['Unlimited projects', 'Email support'], highlighted: true },
  { id: 'team', name: 'Team', price: 1999, features: ['SSO', 'Priority support', 'Audit log'] },
]

function Price({ amount }) {
  return <p className="price">{amount === 0 ? 'Free' : `₹${amount}/mo`}</p>
}

function FeatureList({ features }) {
  return (
    <ul>
      {features.map(f => <li key={f}>{f}</li>)}
    </ul>
  )
}

function PlanCard({ name, price, features, highlighted = false }) {
  return (
    <div className={highlighted ? 'plan plan--highlighted' : 'plan'}>
      <h3>{name}</h3>
      <Price amount={price} />
      <FeatureList features={features} />
    </div>
  )
}

export default function Pricing() {
  return (
    <div className="plans">
      {plans.map(plan => <PlanCard key={plan.id} {...plan} />)}
    </div>
  )
}
```

Three levels of components, each with one job. Data flows strictly downward:
`Pricing → PlanCard → Price / FeatureList`. The `key` prop is consumed by React itself
and is **not** passed to `PlanCard` (lesson 6 explains keys).

## When to split a component

Good signals that a piece of JSX deserves its own component:

- It repeats (cards, rows, list items).
- It has a name in the product vocabulary ("price", "plan", "avatar").
- The parent's JSX is getting long enough that you scroll to understand it.
- It will need its own state later.

Don't split just to make files tiny; a 60-line component that reads top to bottom is
fine.

## How It Actually Works

When React encounters an element `{ type: PlanCard, props: {...} }`, it calls
`PlanCard(props)` and uses the return value as that component's output. Props are just
the object built from the JSX attributes, with `children` added and `key`/`ref`
handled specially. In development, React **freezes** the props object
(`Object.freeze`), so assigning `props.name = ...` throws in strict mode — the
read-only rule is enforced, not just a convention.

Why does React care so much? Because it assumes components are **pure**: same props (and
state) in → same JSX out, with no side effects during rendering. That assumption lets
React call your component whenever it likes, as many times as it likes — render twice
in `StrictMode`, render and throw away the result during concurrent rendering (Level 4),
or skip calling it entirely when inputs haven't changed (`memo`, Level 2). If a
component mutated its props, the parent's data would silently change underneath it,
and any of those optimizations would produce inconsistent UI.

Note that props are a **snapshot for this render**. Each time the parent re-renders,
the child gets a brand-new props object. Nothing "subscribes" to props; the child simply
gets called again with the new values.

## Common mistakes

- **Mutating a prop object or array** (`items.push(...)`, `user.name = ...`). Derive new
  values instead.
- **Copying props into state** (`const [name, setName] = useState(props.name)`) and
  expecting it to update when the parent changes. `useState`'s argument is only used on
  the first render. Usually you should just use the prop directly.
- **Passing numbers as strings.** `price="499"` is the string `"499"`; `price={499}` is
  the number. `"499" + 1` is `"4991"`.
- **Defining components inside other components.** A function declared inside
  `Pricing`'s body is a new component type on every render, so React unmounts and
  remounts it each time, losing its state. Define components at the top level.

## Exercise

Build a `ContactList` page from an array of at least four contacts
(`{ id, name, email, phone, favorite }`):

1. A `ContactCard` component with props `name`, `email`, `phone` and `favorite`
   (default `false`). Show a ★ next to favorites.
2. A reusable `Section` component that takes a `title` and `children`.
3. Render two `Section`s: "Favorites" and "Everyone else", each containing the right
   `ContactCard`s (filter the array before mapping).
4. Hide the phone number entirely when it's missing, without rendering an empty `<p>`.
