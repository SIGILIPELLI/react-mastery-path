---
description: "Level 1 project: build a complete expense tracker in React with components, controlled forms, derived totals, filtering, and localStorage persistence."
---

# 10 · Project — Expense Tracker

Time to combine everything from Level 1 into one app you could actually use. You'll
build an expense tracker where you can add expenses, categorize them, filter by month
and category, see totals, delete entries, and keep everything after a page refresh.

## What you'll build

- A form to add an expense: description, amount, category, date.
- A list of expenses, newest first, with a delete button on each.
- Filters by category and by month.
- A summary showing the filtered total and a per-category breakdown bar.
- Persistence in `localStorage`.

## Plan the components and state first

Before writing code, decide **where state lives**. Ask for each piece of data: who
needs it, and can it be computed from something else?

| Data | Kind | Lives in |
|---|---|---|
| list of expenses | state | `App` (list, summary and form all need it) |
| category filter, month filter | state | `App` (list and summary both use them) |
| form field values | state | `ExpenseForm` (nobody else cares until submit) |
| filtered list, totals, breakdown | **derived** | computed in `App` during render |

Component tree:

```text
App
├── ExpenseForm        (onAdd)
├── Filters            (category, month, onChange…)
├── Summary            (expenses)
└── ExpenseList        (expenses, onDelete)
    └── ExpenseRow
```

## Step 1 — project setup

```bash
npm create vite@latest expense-tracker -- --template react
cd expense-tracker
npm install
npm run dev
```

Create `src/components/` for the pieces below, and clear `App.css`.

## Step 2 — shared constants and helpers

```js
// src/lib.js
export const CATEGORIES = ['Food', 'Transport', 'Bills', 'Shopping', 'Health', 'Other']

export const formatINR = amount =>
  new Intl.NumberFormat('en-IN', { style: 'currency', currency: 'INR' }).format(amount)

export const monthKey = isoDate => isoDate.slice(0, 7) // "2026-09-14" -> "2026-09"

export const today = () => new Date().toISOString().slice(0, 10)
```

`Intl.NumberFormat` handles currency symbols and Indian digit grouping for you. Change
the locale and currency if you prefer another.

Note: `toISOString()` is in UTC, so near midnight "today" may be off by a day in some
time zones. For this project that's acceptable; a production app would format the local
date instead.

## Step 3 — the form

```jsx
// src/components/ExpenseForm.jsx
import { useRef, useState } from 'react'
import { CATEGORIES, today } from '../lib'

const blank = () => ({ description: '', amount: '', category: 'Food', date: today() })

export default function ExpenseForm({ onAdd }) {
  const [values, setValues] = useState(blank)
  const [error, setError] = useState('')
  const descRef = useRef(null)

  function handleChange(e) {
    const { name, value } = e.target
    setValues(prev => ({ ...prev, [name]: value }))
  }

  function handleSubmit(e) {
    e.preventDefault()
    const amount = Number(values.amount)
    if (!values.description.trim()) return setError('Add a short description.')
    if (!Number.isFinite(amount) || amount <= 0) return setError('Amount must be greater than 0.')

    onAdd({
      id: crypto.randomUUID(),
      description: values.description.trim(),
      amount: Math.round(amount * 100) / 100,
      category: values.category,
      date: values.date,
    })
    setValues(blank())
    setError('')
    descRef.current.focus()
  }

  return (
    <form className="card form" onSubmit={handleSubmit}>
      <h2>Add expense</h2>
      <input ref={descRef} name="description" placeholder="What was it?"
             value={values.description} onChange={handleChange} />
      <input name="amount" type="number" min="0" step="0.01" placeholder="Amount"
             value={values.amount} onChange={handleChange} />
      <select name="category" value={values.category} onChange={handleChange}>
        {CATEGORIES.map(c => <option key={c}>{c}</option>)}
      </select>
      <input name="date" type="date" value={values.date} onChange={handleChange} />
      {error && <p className="error" role="alert">{error}</p>}
      <button type="submit">Add</button>
    </form>
  )
}
```

The form owns its field state and only tells the parent about a **complete, valid**
expense through `onAdd`. The id is generated once, when the expense is created —
never during render.

## Step 4 — filters

```jsx
// src/components/Filters.jsx
import { CATEGORIES } from '../lib'

export default function Filters({ category, month, months, onCategoryChange, onMonthChange }) {
  return (
    <div className="card filters">
      <label>
        Category
        <select value={category} onChange={e => onCategoryChange(e.target.value)}>
          <option value="All">All</option>
          {CATEGORIES.map(c => <option key={c}>{c}</option>)}
        </select>
      </label>
      <label>
        Month
        <select value={month} onChange={e => onMonthChange(e.target.value)}>
          <option value="All">All</option>
          {months.map(m => <option key={m} value={m}>{m}</option>)}
        </select>
      </label>
    </div>
  )
}
```

`Filters` is fully **controlled by its parent**: it shows values it's given and reports
changes. It has no state of its own.

## Step 5 — summary with a breakdown

```jsx
// src/components/Summary.jsx
import { formatINR } from '../lib'

export default function Summary({ expenses }) {
  const total = expenses.reduce((sum, e) => sum + e.amount, 0)
  const byCategory = {}
  for (const e of expenses) byCategory[e.category] = (byCategory[e.category] ?? 0) + e.amount
  const rows = Object.entries(byCategory).sort((a, b) => b[1] - a[1])

  return (
    <section className="card">
      <h2>Total: {formatINR(total)}</h2>
      {rows.length === 0 ? (
        <p className="muted">Nothing to summarise yet.</p>
      ) : (
        <ul className="breakdown">
          {rows.map(([cat, amt]) => (
            <li key={cat}>
              <span>{cat}</span>
              <div className="bar" style={{ '--pct': `${(amt / total) * 100}%` }} />
              <span>{formatINR(amt)}</span>
            </li>
          ))}
        </ul>
      )}
    </section>
  )
}
```

`Object.entries(...)` creates a new array here, so calling `.sort` on it is safe — it
doesn't touch props.

## Step 6 — the list

```jsx
// src/components/ExpenseList.jsx
import { formatINR } from '../lib'

function ExpenseRow({ expense, onDelete }) {
  return (
    <li className="row">
      <div>
        <strong>{expense.description}</strong>
        <small>{expense.category} · {expense.date}</small>
      </div>
      <span>{formatINR(expense.amount)}</span>
      <button type="button" onClick={() => onDelete(expense.id)}
              aria-label={`Delete ${expense.description}`}>
        ✕
      </button>
    </li>
  )
}

export default function ExpenseList({ expenses, onDelete }) {
  if (expenses.length === 0) return <p className="card muted">No expenses match these filters.</p>
  return (
    <ul className="card list">
      {expenses.map(e => <ExpenseRow key={e.id} expense={e} onDelete={onDelete} />)}
    </ul>
  )
}
```

## Step 7 — wire it together in `App`

```jsx
// src/App.jsx
import { useEffect, useState } from 'react'
import ExpenseForm from './components/ExpenseForm'
import Filters from './components/Filters'
import Summary from './components/Summary'
import ExpenseList from './components/ExpenseList'
import { monthKey } from './lib'
import './App.css'

const STORAGE_KEY = 'expense-tracker:v1'

function loadExpenses() {
  try {
    const parsed = JSON.parse(localStorage.getItem(STORAGE_KEY))
    return Array.isArray(parsed) ? parsed : []
  } catch {
    return []
  }
}

export default function App() {
  const [expenses, setExpenses] = useState(loadExpenses)
  const [category, setCategory] = useState('All')
  const [month, setMonth] = useState('All')

  useEffect(() => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(expenses))
  }, [expenses])

  function handleAdd(expense) {
    setExpenses(prev => [expense, ...prev])
  }

  function handleDelete(id) {
    setExpenses(prev => prev.filter(e => e.id !== id))
  }

  // Derived data — recomputed each render, never stored.
  const months = [...new Set(expenses.map(e => monthKey(e.date)))].sort().reverse()
  const visible = expenses
    .filter(e => category === 'All' || e.category === category)
    .filter(e => month === 'All' || monthKey(e.date) === month)
    .toSorted((a, b) => b.date.localeCompare(a.date))

  return (
    <main className="layout">
      <h1>Expense Tracker</h1>
      <ExpenseForm onAdd={handleAdd} />
      <Filters category={category} month={month} months={months}
               onCategoryChange={setCategory} onMonthChange={setMonth} />
      <Summary expenses={visible} />
      <ExpenseList expenses={visible} onDelete={handleDelete} />
    </main>
  )
}
```

Note `useState(loadExpenses)` — passing the function (not calling it) so storage is read
only once.

## Step 8 — minimal CSS

```css
/* src/App.css */
.layout { max-width: 720px; margin: 2rem auto; padding: 0 1rem; display: grid; gap: 1rem; font-family: system-ui, sans-serif; }
.card { border: 1px solid #ddd; border-radius: 10px; padding: 1rem; list-style: none; }
.form { display: grid; gap: 0.5rem; }
.filters { display: flex; gap: 1rem; }
.row { display: grid; grid-template-columns: 1fr auto auto; gap: 1rem; align-items: center; padding: 0.4rem 0; border-bottom: 1px solid #eee; }
.row small { display: block; color: #777; }
.breakdown li { display: grid; grid-template-columns: 90px 1fr 110px; gap: 0.5rem; align-items: center; }
.bar { height: 10px; border-radius: 5px; background: linear-gradient(90deg, #2f6fed var(--pct), #eee var(--pct)); }
.error { color: #b42318; }
.muted { color: #777; }
```

## How It Actually Works

Trace what happens when you submit the form:

1. `ExpenseForm`'s `handleSubmit` calls `onAdd(expense)`, which is `App`'s
   `handleAdd`. That calls `setExpenses` with an updater. Then the form resets its own
   fields — three more state updates in the same event.
2. React **batches** all of these into a single re-render pass.
3. React re-renders `App` (its state changed) and, by default, all of its children.
   `App` recomputes `months` and `visible` from the new `expenses` array. `Summary` and
   `ExpenseList` receive new props and re-render.
4. In `ExpenseList`, React reconciles the keyed rows. Existing expense ids match
   existing rows, so those DOM nodes are reused; only the new row is created and
   inserted.
5. After commit, the persistence effect runs because `expenses` is a new array
   (`Object.is` sees a different reference), and writes to `localStorage`.

Changing a filter updates `category` in `App`. `expenses` hasn't changed identity, so
the storage effect is skipped — the dependency array does exactly the filtering you
want.

The single-source-of-truth design is what makes this robust: there is one array of
expenses, and every number on screen is computed from it. There's no way for the total
to disagree with the list.

## Common pitfalls in this project

- Storing `total` or `visible` in state and forgetting to update one of them.
- Parsing `localStorage` without a `try/catch`. Corrupted or old-format data would
  crash the app on load.
- Using the array index as the key — deleting an expense would shuffle row identities.
- Keeping `amount` as a string; `reduce` would concatenate strings instead of adding.

## Exercise — extend the project

Pick at least three:

1. **Edit in place**: clicking an expense loads it into the form; saving replaces the
   entry by id instead of adding a new one.
2. **Budget per category**: let the user set a monthly budget per category and show a
   warning style on bars above 100%.
3. **CSV export**: a button that builds a CSV string from the visible expenses and
   downloads it via a `Blob` and a temporary `<a download>` link.
4. **Undo delete**: after deleting, show a toast with "Undo" for 5 seconds (an effect
   with a timeout and cleanup).
5. **Sort control**: sort by date, amount or description, ascending/descending, without
   ever mutating state.
