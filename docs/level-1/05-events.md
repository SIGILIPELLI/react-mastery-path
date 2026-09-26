---
description: "Handle clicks, keys and form events in React, pass handlers to children, stop default behaviour and propagation, and learn how React delegates events."
---

# 05 · Handling Events

Interactivity in React is wiring: you pass a function to an `on*` prop, React calls it
when the event happens, and your function usually updates state.

## Attaching handlers

```jsx
export default function Buttons() {
  function handleSave() {
    console.log('saving...')
  }

  return (
    <>
      <button onClick={handleSave}>Save</button>
      <button onClick={() => console.log('cancelled')}>Cancel</button>
    </>
  )
}
```

Pass the function; don't call it. `onClick={handleSave}` hands React a reference.
`onClick={handleSave()}` runs `handleSave` during render and passes its return value
(`undefined`).

To pass arguments, wrap in an arrow function:

```jsx
{colors.map(c => (
  <button key={c} onClick={() => setColor(c)}>{c}</button>
))}
```

## The event object

Handlers receive a **SyntheticEvent**: a React wrapper with the same interface as the
browser event (`target`, `currentTarget`, `key`, `preventDefault()`,
`stopPropagation()`…) that behaves consistently across browsers. The original is
available as `e.nativeEvent` if you ever need it.

```jsx
function SearchBox() {
  function handleKeyDown(e) {
    if (e.key === 'Escape') e.currentTarget.value = ''
    if (e.key === 'Enter') console.log('search for', e.currentTarget.value)
  }
  return <input onKeyDown={handleKeyDown} placeholder="Search" />
}
```

Common events: `onClick`, `onChange`, `onInput`, `onSubmit`, `onKeyDown`, `onFocus`,
`onBlur`, `onMouseEnter`, `onPointerDown`, `onScroll`.

Note React's `onChange` on text inputs fires on **every keystroke** (like the DOM's
`input` event), not only when the field loses focus.

## Preventing default behaviour

```jsx
function SignupForm() {
  function handleSubmit(e) {
    e.preventDefault()          // stop the full-page form submission
    const data = new FormData(e.currentTarget)
    console.log(Object.fromEntries(data))
  }
  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" required />
      <button type="submit">Join</button>
    </form>
  )
}
```

In React you cannot `return false` to prevent defaults; call `e.preventDefault()`.

## Propagation

Events bubble from the target up through ancestors, and React handlers follow the same
path through the **component tree**.

```jsx
function Card({ onOpen, onDelete }) {
  return (
    <div className="card" onClick={onOpen}>
      <h3>Invoice #88</h3>
      <button
        onClick={e => {
          e.stopPropagation() // don't also open the card
          onDelete()
        }}
      >
        Delete
      </button>
    </div>
  )
}
```

Without `stopPropagation`, clicking Delete would also trigger `onOpen` on the parent
`div`. Use capture-phase variants (`onClickCapture`) when an ancestor must see an event
before children do — rare, but useful for analytics or closing menus.

## Passing handlers down: naming conventions

A child that shouldn't know *what* happens on click exposes an `on*` prop; the parent
supplies a `handle*` function:

```jsx
function Rating({ value, onChange }) {
  return (
    <div role="radiogroup" aria-label="Rating">
      {[1, 2, 3, 4, 5].map(n => (
        <button
          key={n}
          role="radio"
          aria-checked={n === value}
          onClick={() => onChange(n)}
        >
          {n <= value ? '★' : '☆'}
        </button>
      ))}
    </div>
  )
}

export default function Review() {
  const [stars, setStars] = useState(0)
  function handleRatingChange(n) {
    setStars(n)
  }
  return (
    <>
      <Rating value={stars} onChange={handleRatingChange} />
      <p>{stars ? `You rated ${stars}/5` : 'Not rated yet'}</p>
    </>
  )
}
```

(Import `useState` from `react` at the top.) `Rating` owns no state; it reports clicks
upward. This pattern — state in the parent, events flowing up through callbacks — is
the core of React data flow.

## Worked example: keyboard-driven counter

```jsx
import { useState } from 'react'

export default function StepCounter() {
  const [value, setValue] = useState(0)
  const [step, setStep] = useState(1)

  function handleKeyDown(e) {
    if (e.key === 'ArrowUp') {
      e.preventDefault() // stop the page scrolling
      setValue(v => v + step)
    } else if (e.key === 'ArrowDown') {
      e.preventDefault()
      setValue(v => v - step)
    }
  }

  return (
    <div>
      <div
        tabIndex={0}
        onKeyDown={handleKeyDown}
        className="counter-box"
        aria-label={`Value ${value}. Use arrow keys to change.`}
      >
        {value}
      </div>
      <label>
        Step:
        <select value={step} onChange={e => setStep(Number(e.target.value))}>
          <option value={1}>1</option>
          <option value={5}>5</option>
          <option value={10}>10</option>
        </select>
      </label>
    </div>
  )
}
```

`tabIndex={0}` makes the `div` focusable so it can receive keyboard events.
`Number(e.target.value)` matters: form values are always strings.

## How It Actually Works

React does **not** attach a listener to every element with an `onClick`. Since React 17,
it installs one listener per event type on the **root container** (the `#root` div you
passed to `createRoot`) — a technique called event delegation. When you click a
button, the native event bubbles up to the root. React's listener then:

1. Finds the fiber (internal component node) that corresponds to the DOM target.
2. Walks up the fiber tree from there, collecting every `onClick` (for the bubble
   phase) or `onClickCapture` (for the capture phase) prop it finds.
3. Creates one SyntheticEvent and calls those handlers in order, stopping early if one
   calls `stopPropagation()`.

Because the walk follows the **React tree**, not the DOM tree, events from inside a
portal (a component rendered into a different DOM node, e.g. a modal attached to
`document.body`) still bubble to their React parent. That is intentional and often
useful, but can surprise you.

React also assigns each event a **priority**. Discrete events such as clicks and key
presses are handled urgently: state updates inside them are batched and flushed
synchronously at the end of the handler, so the UI never shows an intermediate state
between two keystrokes. Continuous events like `mousemove` or `scroll` get lower
priority. This ties into concurrent rendering in Level 4.

Finally, since your handler is a closure created during a particular render, it sees
that render's props and state. If you need the latest state inside a handler that
updates it, use the updater form of the setter.

## Common mistakes

- **Calling the handler in JSX:** `onClick={remove(id)}` runs during render. Use
  `onClick={() => remove(id)}`.
- **Forgetting `type="button"`** on non-submit buttons inside a `<form>`; the default
  type is `submit`, so they submit the form.
- **Treating `e.target.value` as a number.** It's always a string for inputs and
  selects.
- **Reading `e.target` when you mean `e.currentTarget`.** `target` is the innermost
  element clicked (maybe an icon inside the button); `currentTarget` is the element the
  handler is attached to.
- **Putting `onClick` on a `div` instead of a `button`.** You lose keyboard access and
  screen-reader semantics. Use real buttons for actions.

## Exercise

Build a `ColorMixer`:

1. Three `<input type="range">` sliders (0–255) for red, green and blue, each updating
   its own state with `onChange`.
2. A swatch `div` showing `rgb(r, g, b)` as its background, with the hex code in text.
3. A "Random" button that sets all three at once, and a "Copy hex" button that calls
   `navigator.clipboard.writeText(hex)`.
4. Wrap everything in a card whose `onClick` logs "card clicked". Make sure pressing
   either button does **not** log that message, and explain in a comment which method
   you used and why.
5. Pressing the `r`, `g` or `b` key while the card is focused should focus the
   matching slider (hint: `tabIndex` on the card, and `document.getElementById` is fine
   for now — lesson 7 introduces refs, after which you can come back and replace it).
