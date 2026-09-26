---
description: "Build React forms with controlled inputs, checkboxes, selects and textareas, validate on submit, and know when an uncontrolled input with useRef fits."
---

# 07 · Forms & Controlled Inputs

Forms are where React's "UI is a function of state" idea is tested hardest. The same
input can be managed two ways: React owns the value (**controlled**) or the DOM owns it
(**uncontrolled**). Knowing both, and when to use which, saves a lot of pain.

## Controlled inputs

```jsx
import { useState } from 'react'

function NameField() {
  const [name, setName] = useState('')
  return (
    <label>
      Name
      <input value={name} onChange={e => setName(e.target.value)} />
      <small>{name.length}/40</small>
    </label>
  )
}
```

The input's `value` comes from state, and every keystroke updates state. React is the
single source of truth, so you can transform, validate or react to the value
instantly — character counters, formatting, disabling a button until valid.

If you give an input `value` without `onChange`, it becomes read-only and React warns.
Use `defaultValue` if you meant an uncontrolled input with a starting value.

## One state object for many fields

```jsx
function AddressForm() {
  const [form, setForm] = useState({ line1: '', city: '', pin: '' })

  function handleChange(e) {
    const { name, value } = e.target
    setForm(prev => ({ ...prev, [name]: value }))
  }

  return (
    <>
      <input name="line1" value={form.line1} onChange={handleChange} />
      <input name="city" value={form.city} onChange={handleChange} />
      <input name="pin" value={form.pin} onChange={handleChange} inputMode="numeric" />
    </>
  )
}
```

`[name]: value` is a computed property key, so one handler serves every field whose
`name` attribute matches a key in the state object.

## Other input types

| Element | Controlled via | Read in handler |
|---|---|---|
| text, email, number, textarea | `value` | `e.target.value` (a string) |
| checkbox | `checked` | `e.target.checked` |
| radio group | `checked={choice === 'x'}` | `e.target.value` |
| select | `value` on `<select>` | `e.target.value` |
| multi-select | `value={array}` + `multiple` | `[...e.target.selectedOptions].map(o => o.value)` |

```jsx
<textarea value={bio} onChange={e => setBio(e.target.value)} rows={4} />

<select value={plan} onChange={e => setPlan(e.target.value)}>
  <option value="basic">Basic</option>
  <option value="pro">Pro</option>
</select>

<label>
  <input type="checkbox" checked={agree} onChange={e => setAgree(e.target.checked)} />
  I agree to the terms
</label>

{['card', 'upi', 'cod'].map(m => (
  <label key={m}>
    <input type="radio" name="pay" value={m} checked={method === m}
           onChange={e => setMethod(e.target.value)} />
    {m.toUpperCase()}
  </label>
))}
```

In JSX, `<textarea>` uses a `value` prop rather than children, and `<select>` takes
`value` instead of `selected` on an option.

## Uncontrolled inputs and `useRef`

Sometimes you don't need the value on every keystroke — only at submit. Then let the
DOM hold it:

```jsx
function QuickNote({ onSave }) {
  function handleSubmit(e) {
    e.preventDefault()
    const data = new FormData(e.currentTarget)
    onSave(data.get('note'))
    e.currentTarget.reset()
  }
  return (
    <form onSubmit={handleSubmit}>
      <input name="note" defaultValue="" />
      <button>Save</button>
    </form>
  )
}
```

To reach a DOM node directly — to focus it, measure it, or read a file input — use a
**ref**:

```jsx
import { useRef } from 'react'

function SearchBar() {
  const inputRef = useRef(null)
  return (
    <>
      <input ref={inputRef} type="search" />
      <button type="button" onClick={() => inputRef.current.focus()}>Focus search</button>
    </>
  )
}
```

`useRef(initial)` returns an object `{ current: initial }` that survives re-renders.
After React creates the `<input>`, it sets `inputRef.current` to that DOM node.
Changing `ref.current` does **not** trigger a re-render, which is why refs are the
escape hatch for things React doesn't need to display.

File inputs are always uncontrolled because their value can't be set by code for
security reasons.

## Worked example: a registration form with validation

```jsx
import { useState } from 'react'

const empty = { email: '', password: '', confirm: '', role: 'student', terms: false }

function validate(v) {
  const errors = {}
  if (!/^\S+@\S+\.\S+$/.test(v.email)) errors.email = 'Enter a valid email address.'
  if (v.password.length < 8) errors.password = 'Use at least 8 characters.'
  if (v.confirm !== v.password) errors.confirm = 'Passwords do not match.'
  if (!v.terms) errors.terms = 'You must accept the terms.'
  return errors
}

export default function Register() {
  const [values, setValues] = useState(empty)
  const [touched, setTouched] = useState({})
  const [submitted, setSubmitted] = useState(false)

  const errors = validate(values)            // derived, not stored
  const isValid = Object.keys(errors).length === 0
  const show = field => (touched[field] || submitted) && errors[field]

  function handleChange(e) {
    const { name, type, value, checked } = e.target
    setValues(prev => ({ ...prev, [name]: type === 'checkbox' ? checked : value }))
  }

  function handleBlur(e) {
    setTouched(prev => ({ ...prev, [e.target.name]: true }))
  }

  function handleSubmit(e) {
    e.preventDefault()
    setSubmitted(true)
    if (!isValid) return
    alert(`Registered ${values.email} as ${values.role}`)
    setValues(empty)
    setTouched({})
    setSubmitted(false)
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      <label>
        Email
        <input name="email" type="email" value={values.email}
               onChange={handleChange} onBlur={handleBlur}
               aria-invalid={Boolean(show('email'))} aria-describedby="email-err" />
      </label>
      {show('email') && <p id="email-err" className="error">{errors.email}</p>}

      <label>
        Password
        <input name="password" type="password" value={values.password}
               onChange={handleChange} onBlur={handleBlur} />
      </label>
      {show('password') && <p className="error">{errors.password}</p>}

      <label>
        Confirm password
        <input name="confirm" type="password" value={values.confirm}
               onChange={handleChange} onBlur={handleBlur} />
      </label>
      {show('confirm') && <p className="error">{errors.confirm}</p>}

      <label>
        Role
        <select name="role" value={values.role} onChange={handleChange}>
          <option value="student">Student</option>
          <option value="mentor">Mentor</option>
        </select>
      </label>

      <label>
        <input name="terms" type="checkbox" checked={values.terms}
               onChange={handleChange} onBlur={handleBlur} />
        I accept the terms
      </label>
      {show('terms') && <p className="error">{errors.terms}</p>}

      <button type="submit">Create account</button>
    </form>
  )
}
```

Design choices worth copying:

- **Errors are derived** from values on every render rather than stored, so they can
  never be out of date.
- Errors appear only after a field is **touched** (blurred) or the form is submitted —
  nobody wants "invalid email" after typing one letter.
- `noValidate` turns off the browser's built-in bubbles so the custom messages are the
  only ones shown; `aria-invalid` and `aria-describedby` keep them accessible.

This example stops at an `alert` because it has no backend. Never treat client-side
validation as security: the server must validate again.

## How It Actually Works

A controlled input is a small loop: the DOM fires an `input` event → React's delegated
listener calls your `onChange` → you set state → React re-renders → React sets the DOM
node's `value` property to your state.

But the browser has *already* changed the input's value before your handler runs. So
what stops the user typing anything when you *refuse* to update state (say, rejecting
non-digits)? After the event handler finishes, React checks every controlled input
involved in the event and **restores** its DOM value to match the current prop value if
they differ. That "controlled value restoration" is why a controlled input genuinely
cannot hold a value that isn't in state.

One subtle consequence: state updates from a text input are processed synchronously
at high priority. If you updated input state inside a `setTimeout` or after an `await`,
React would restore the old value first and the caret would jump to the end. Keep the
`setState` for the input's value directly in the `onChange` handler.

Switching an input between controlled and uncontrolled (`value={undefined}` on one render
and a string on the next) confuses this mechanism, which is why React warns "A
component is changing an uncontrolled input to be controlled". Always initialise
controlled values to `''`, not `undefined`.

Refs work because React fills in `ref.current` during the **commit** phase, after the
DOM node exists and before effects run. During rendering, `ref.current` for a DOM ref
is still `null` on the first render, so read refs in event handlers or effects, not in
the render body.

## Common mistakes

- **`value` without `onChange`** → a frozen, read-only input.
- **Initial state `undefined`** → the controlled/uncontrolled warning. Use `''` or
  `false`.
- **Storing `isValid`/`errors` in state** alongside values, then forgetting to update
  them. Derive them.
- **Number inputs:** `e.target.value` is still a string (possibly `''`). Convert with
  `Number()` when computing, but keep the raw string in state so the user can clear the
  field.
- **Missing `preventDefault()`** in `onSubmit` → the page reloads and your state vanishes.

## Exercise

Build a `LoanCalculator` form:

1. Controlled fields: principal (₹), annual interest rate (%), tenure in years
   (a `<select>` of 1–30) and a "Show amortization" checkbox.
2. Compute the EMI on every render using
   `EMI = P × r × (1+r)^n / ((1+r)^n − 1)` where `r` is the monthly rate and `n` the
   number of months. Guard against a 0% rate.
3. Show field errors (principal must be > 0, rate between 0 and 30) only after blur.
4. Add a "Focus principal" button that uses `useRef` to focus the first field.
5. When the checkbox is on, render a table of the first 12 months: interest, principal
   and remaining balance.
