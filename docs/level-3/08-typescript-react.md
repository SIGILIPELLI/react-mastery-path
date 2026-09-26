---
description: "Type React components with TypeScript: props and children, events, useState and useReducer, refs, context, generic components, and discriminated unions."
---

# 08 · TypeScript with React

TypeScript catches the bugs React can't: a misspelled prop, a missing case in a reducer,
passing a string where a number is expected, forgetting that data may be `undefined`.
On a team, typed props are also the most reliable documentation a component has.

Start a typed project with `npm create vite@latest my-app -- --template react-ts`. Files
containing JSX use the `.tsx` extension. The React type definitions come from
`@types/react` and `@types/react-dom`, which the template installs.

## Typing props

```tsx
type ButtonProps = {
  variant?: 'primary' | 'secondary' | 'danger'
  size?: 'sm' | 'md'
  loading?: boolean
  children: React.ReactNode
  onClick?: () => void
}

export function Button({ variant = 'primary', size = 'md', loading = false, children, onClick }: ButtonProps) {
  return (
    <button className={`btn btn--${variant} btn--${size}`} disabled={loading} onClick={onClick}>
      {loading ? 'Working…' : children}
    </button>
  )
}
```

- Use a `type` (or `interface`) for props and annotate the destructured parameter. The
  older `React.FC` style is no longer recommended as a default; plain functions are
  simpler.
- String-literal unions (`'primary' | 'secondary'`) give autocomplete and reject typos.
- `React.ReactNode` is "anything renderable": elements, strings, numbers, `null`,
  arrays of those.

`React.` works without an import because the types expose a global `React` namespace in
most setups; you can also `import type { ReactNode } from 'react'`.

## Extending native element props

A wrapper around `<button>` should accept every native button attribute:

```tsx
import type { ComponentPropsWithoutRef } from 'react'

type IconButtonProps = ComponentPropsWithoutRef<'button'> & {
  icon: React.ReactNode
  label: string
}

export function IconButton({ icon, label, ...rest }: IconButtonProps) {
  return (
    <button type="button" aria-label={label} {...rest}>
      {icon}
    </button>
  )
}
```

Now `<IconButton icon={<Trash />} label="Delete" disabled onClick={...} />` type-checks,
including `disabled`, `onClick`, `aria-*`, etc.

## Events

```tsx
function Search({ onSearch }: { onSearch: (q: string) => void }) {
  const [q, setQ] = useState('')

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    setQ(e.target.value)
  }

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    onSearch(q)
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={q} onChange={handleChange} />
    </form>
  )
}
```

Tip: when an inline handler is typed by context (`onChange={e => ...}`), hover `e` in
your editor to discover the right type, then use it for extracted handlers.

## Hooks

```tsx
const [count, setCount] = useState(0)                // inferred: number
const [user, setUser] = useState<User | null>(null)  // explicit: starts null
const [tags, setTags] = useState<string[]>([])       // empty array needs a type

const inputRef = useRef<HTMLInputElement>(null)      // DOM ref
inputRef.current?.focus()                            // may be null before mount

const timerRef = useRef<number | undefined>(undefined) // mutable value ref
```

`useMemo` and `useCallback` infer from their return values. Type callback parameters.

## Discriminated unions for state

The best TypeScript pattern for React state: make impossible states unrepresentable.

```tsx
type FetchState<T> =
  | { status: 'loading' }
  | { status: 'error'; error: Error }
  | { status: 'success'; data: T }

function UserCard({ state }: { state: FetchState<User> }) {
  switch (state.status) {
    case 'loading':
      return <p>Loading…</p>
    case 'error':
      return <p role="alert">{state.error.message}</p>
    case 'success':
      return <h2>{state.data.name}</h2>      // data is only accessible here
  }
}
```

Inside each `case`, TypeScript **narrows** the type: `state.data` is an error outside the
`'success'` branch. You can't accidentally render data while loading.

## Typed reducers

```tsx
type Todo = { id: string; text: string; done: boolean }

type Action =
  | { type: 'added'; id: string; text: string }
  | { type: 'toggled'; id: string }
  | { type: 'deleted'; id: string }

function todosReducer(state: Todo[], action: Action): Todo[] {
  switch (action.type) {
    case 'added':
      return [...state, { id: action.id, text: action.text, done: false }]
    case 'toggled':
      return state.map(t => (t.id === action.id ? { ...t, done: !t.done } : t))
    case 'deleted':
      return state.filter(t => t.id !== action.id)
    default: {
      const unreachable: never = action   // compile error if a case is missing
      return unreachable
    }
  }
}
```

Adding a new action type to the union without handling it produces a compile error at
the `never` line — the compiler finds every reducer you need to update.

## Typed context

```tsx
import { createContext, useContext } from 'react'

type Theme = 'light' | 'dark'
type ThemeContextValue = { theme: Theme; toggle: () => void }

const ThemeContext = createContext<ThemeContextValue | null>(null)

export function useTheme(): ThemeContextValue {
  const ctx = useContext(ThemeContext)
  if (!ctx) throw new Error('useTheme must be used within ThemeProvider')
  return ctx                  // narrowed to ThemeContextValue
}
```

## Generic components

```tsx
type ListProps<T> = {
  items: T[]
  getKey: (item: T) => string | number
  renderItem: (item: T) => React.ReactNode
  empty?: React.ReactNode
}

export function List<T>({ items, getKey, renderItem, empty = <p>Nothing here.</p> }: ListProps<T>) {
  if (items.length === 0) return <>{empty}</>
  return <ul>{items.map(item => <li key={getKey(item)}>{renderItem(item)}</li>)}</ul>
}

// T is inferred as Product; renderItem's parameter is typed automatically
<List items={products} getKey={p => p.id} renderItem={p => `${p.name} — ₹${p.price}`} />
```

## Worked example: a typed data table

```tsx
type Column<T> = {
  key: keyof T & string
  header: string
  render?: (value: T[keyof T], row: T) => React.ReactNode
  align?: 'left' | 'right'
}

type TableProps<T extends { id: string | number }> = {
  rows: T[]
  columns: Column<T>[]
  onRowClick?: (row: T) => void
}

export function Table<T extends { id: string | number }>({ rows, columns, onRowClick }: TableProps<T>) {
  return (
    <table>
      <thead>
        <tr>{columns.map(c => <th key={c.key} style={{ textAlign: c.align }}>{c.header}</th>)}</tr>
      </thead>
      <tbody>
        {rows.map(row => (
          <tr key={row.id} onClick={onRowClick ? () => onRowClick(row) : undefined}>
            {columns.map(c => (
              <td key={c.key} style={{ textAlign: c.align }}>
                {c.render ? c.render(row[c.key], row) : String(row[c.key])}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  )
}

type Invoice = { id: string; customer: string; amount: number; paid: boolean }

<Table<Invoice>
  rows={invoices}
  columns={[
    { key: 'customer', header: 'Customer' },
    { key: 'amount', header: 'Amount', align: 'right', render: v => `₹${v}` },
    { key: 'paid', header: 'Status', render: v => (v ? 'Paid' : 'Due') },
    // { key: 'amnt', header: 'x' }  ← compile error: not a key of Invoice
  ]}
/>
```

`keyof T & string` restricts `key` to real property names of the row type, so a typo
in a column definition fails at compile time instead of rendering `undefined`. (A fully
precise `render` value type per column needs a mapped type; this simpler version keeps
`value` as the union of all property types.)

## How It Actually Works

TypeScript exists only at compile time. Vite strips types with a fast transpiler
(esbuild or SWC) that **doesn't type-check** — it just deletes annotations. That's why
the dev server starts instantly and why type errors don't stop it. Type checking happens
separately: in your editor via the TypeScript language server, and via `tsc` (the
template's `build` script runs `tsc -b` before `vite build`, so type errors fail
production builds).

JSX type checking works through the `JSX` namespace in `@types/react`. For an element
like `<Button variant="x" />`, TypeScript looks up the type of `Button`, takes its first
parameter's type as the allowed props, and checks the attributes against it. For
lower-case tags, it looks up `JSX.IntrinsicElements['button']`, which maps each HTML tag
to its allowed attributes — the same source `ComponentPropsWithoutRef<'button'>` reads
from.

Narrowing on discriminated unions is control-flow analysis: after `case 'success':`
TypeScript knows `state.status` is the literal `'success'`, filters the union down to the
members with that literal, and permits only their properties. The `never` trick works
because once every member has been handled, nothing remains in the union.

## Common mistakes

- **`any` everywhere** to silence errors — you get TypeScript's cost with none of the
  benefit. Prefer `unknown` and narrow.
- **`useState([])` without a type** → `never[]`, and nothing can be added.
- **Non-null assertions (`ref.current!`) by habit** instead of handling `null`.
- **Typing `children` as `JSX.Element`** — too narrow (rejects strings, arrays, null).
  Use `React.ReactNode`.
- **Casting API responses** (`as User`) and trusting them. Types don't validate runtime
  data; parse unknown data with a schema (Zod) at the boundary.

## Exercise

Convert the Level 2 recipe finder (or another app) to TypeScript:

1. Rename files to `.tsx`/`.ts`, enable `"strict": true`, and fix every error without
   using `any`.
2. Model fetch state as a discriminated union and make `useFetch<T>` generic.
3. Type the favourites reducer with an `Action` union and an exhaustive `never` check;
   add a new action and watch the compiler point you to the reducer.
4. Write a generic `Select<T>` component taking `options: T[]`, `getLabel`, `getValue`
   and `onChange(option: T)`.
5. Validate the API response with a Zod schema and derive the TypeScript type from it
   with `z.infer`.
