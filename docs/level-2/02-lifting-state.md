---
description: "Decide which component owns each piece of React state, lift it to the nearest common parent, avoid duplicate or derived state, and reset state with keys."
---

# 02 · Lifting State & Data Flow

React data flows one way: down through props. When two components need the same
changing data, neither can "reach across" to the other. The fix is to move the state
**up** to their closest common parent and pass it down to both. This lesson is about
deciding where state belongs — the single most important structural decision in a React
app.

## The problem

Two panels that should behave as an accordion, where only one can be open:

```jsx
function Panel({ title, children }) {
  const [open, setOpen] = useState(false)
  return (
    <section>
      <h3>{title} <button onClick={() => setOpen(!open)}>{open ? 'Hide' : 'Show'}</button></h3>
      {open && children}
    </section>
  )
}
```

Each `Panel` owns its own `open`, so they can't coordinate. Opening one knows nothing
about the other.

## Lifting it up

1. **Remove** the state from the children.
2. **Pass** the values down as props.
3. **Add** the state to the common parent, and pass callbacks for changes.

```jsx
function Panel({ title, isOpen, onToggle, children }) {
  return (
    <section>
      <h3>
        {title} <button onClick={onToggle}>{isOpen ? 'Hide' : 'Show'}</button>
      </h3>
      {isOpen && children}
    </section>
  )
}

export default function Faq() {
  const [openId, setOpenId] = useState(null)
  const toggle = id => setOpenId(current => (current === id ? null : id))

  return (
    <>
      <Panel title="Shipping" isOpen={openId === 'ship'} onToggle={() => toggle('ship')}>
        Orders ship within two working days.
      </Panel>
      <Panel title="Returns" isOpen={openId === 'ret'} onToggle={() => toggle('ret')}>
        Return unused items within 30 days.
      </Panel>
    </>
  )
}
```

`Panel` is now **controlled**: its important state is driven by props. The parent
decides, which makes "only one open" a one-line rule.

## Finding the right owner

For each piece of state:

1. List every component that **reads** it or **changes** it.
2. Find their closest common ancestor.
3. Put the state there (or higher, if something above also needs it).

Put state as **low** as possible while still reaching everyone who needs it. State
too high makes large parts of the tree re-render on every change and makes components
harder to reuse; state too low forces awkward syncing.

## Single source of truth: avoid duplicate and derived state

The most common structural bug is keeping two pieces of state that describe the same
fact.

```jsx
// ❌ selected item duplicated: editing items won't update selectedItem
const [items, setItems] = useState(initialItems)
const [selectedItem, setSelectedItem] = useState(items[0])

// ✅ store the id, derive the object
const [items, setItems] = useState(initialItems)
const [selectedId, setSelectedId] = useState(initialItems[0].id)
const selectedItem = items.find(i => i.id === selectedId)
```

Questions to ask about each `useState`:

- Can I compute it from props or other state? → Don't store it.
- Does it mirror a prop without ever diverging? → Use the prop.
- Are two variables always updated together? → Maybe merge them into one object.
- Could two booleans contradict each other (`isLoading` and `isError` both true)? →
  Replace with one `status` value: `'idle' | 'loading' | 'error' | 'success'`.

## Resetting state with a key

A classic bug: a chat input keeps its draft when you switch contacts.

```jsx
<Chat contact={selectedContact} />
```

`Chat` stays mounted in the same position with the same type, so React keeps its state.
If a fresh `Chat` per contact is what you want, say so with a key:

```jsx
<Chat key={selectedContact.id} contact={selectedContact} />
```

Changing the key tells React "this is a different component instance" — the old one
unmounts and a new one mounts with fresh state.

## Worked example: a temperature converter

Two inputs that must stay in sync — the textbook lifting case:

```jsx
import { useState } from 'react'

const toC = f => ((f - 32) * 5) / 9
const toF = c => (c * 9) / 5 + 32

function tryConvert(value, convert) {
  const n = parseFloat(value)
  if (Number.isNaN(n)) return ''
  return String(Math.round(convert(n) * 100) / 100)
}

function TemperatureInput({ scale, value, onChange }) {
  const label = scale === 'c' ? 'Celsius' : 'Fahrenheit'
  return (
    <label>
      {label}
      <input value={value} onChange={e => onChange(e.target.value)} inputMode="decimal" />
    </label>
  )
}

export default function Converter() {
  // Store only what the user typed, and in which scale.
  const [input, setInput] = useState({ scale: 'c', value: '' })

  const celsius = input.scale === 'c' ? input.value : tryConvert(input.value, toC)
  const fahrenheit = input.scale === 'f' ? input.value : tryConvert(input.value, toF)
  const c = parseFloat(celsius)

  return (
    <fieldset>
      <legend>Temperature</legend>
      <TemperatureInput scale="c" value={celsius} onChange={v => setInput({ scale: 'c', value: v })} />
      <TemperatureInput scale="f" value={fahrenheit} onChange={v => setInput({ scale: 'f', value: v })} />
      {!Number.isNaN(c) && <p>{c >= 100 ? 'Water would boil.' : 'Water would not boil.'}</p>}
    </fieldset>
  )
}
```

Only one fact is stored: the value the user last typed and its scale. The other field
and the boiling message are derived. Storing both temperatures would invite rounding
drift where each field rewrites the other.

## How It Actually Works

When state lives in a component, React stores it on that component's fiber. A setter
call schedules a re-render **starting at that fiber**. React re-renders that component
and, by default, every component below it, because a parent's new output may contain
new props for any child. Components above and beside it are not touched.

This is why placement matters mechanically, not just stylistically:

- State in `App` → any change re-renders the whole app (cheap for small apps, noticeable
  in big ones).
- State in a leaf → changes only re-render that leaf.

"Controlled" and "uncontrolled" components are the same idea you saw with inputs: a
component is uncontrolled when it keeps important info in its own state and controlled
when its parent drives it via props. Lifting state converts uncontrolled components to
controlled ones.

State identity is tied to **position in the tree** plus **type** plus **key**, not to
the variable or JSX that produced it. That's why `<Chat contact={a} />` and
`<Chat contact={b} />` in the same position share state, and why adding a key breaks
that sharing. It's also why rendering `{showA ? <Counter /> : <Counter />}` does *not*
reset the counter, while `{showA ? <Counter key="a" /> : <Counter key="b" />}` does.

## Common mistakes

- **Syncing state between siblings with effects** (`useEffect(() => setB(a), [a])`).
  Lift the state instead; effects for syncing cause extra renders and brief
  inconsistency.
- **Initialising state from props and expecting it to follow the prop.** Either use the
  prop directly, or reset with a `key`.
- **Lifting everything to the top "just in case".** Keep state as local as it can be.
- **Contradictory booleans** instead of a single status value.

## Exercise

Build a two-pane email client (mock data, no server):

1. Left pane: a list of folders (Inbox, Sent, Archive). Middle: messages in the folder.
   Right: the selected message.
2. Decide and write down, in a comment at the top of `App`, which component owns
   `selectedFolder`, `selectedMessageId`, and the messages array — and why.
3. Store `selectedMessageId`, never the message object. Archiving a message moves it
   to the Archive folder; the reading pane must update or clear correctly.
4. Add a reply draft `<textarea>` in the reading pane that resets when you open a
   different message, using a key — not an effect.
5. Replace any `isX`/`isY` boolean pairs with a single status value.
