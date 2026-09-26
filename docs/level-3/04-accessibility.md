---
description: "Build accessible React UIs: semantic HTML first, labels and names, focus management for dialogs and route changes, correct ARIA, and automated a11y checks."
---

# 04 · Accessibility in React

Accessibility (a11y) means people can use your app with a keyboard, a screen reader,
voice control, zoom, or reduced motion. React doesn't make apps inaccessible, but it
makes it very easy to build clickable `div`s, move focus into nowhere, and swap page
content without telling assistive technology. This lesson covers the React-specific
parts of getting it right.

## 1. Semantic HTML first

The browser gives native elements keyboard support, focus, roles and names for free.

```jsx
// ❌ looks like a button, isn't one
<div className="btn" onClick={save}>Save</div>

// ✅ focusable, Enter/Space activate it, announced as "Save, button"
<button type="button" onClick={save}>Save</button>
```

Use `<a href>` for navigation, `<button>` for actions, `<label>` with inputs, lists for
lists, headings in order (`h1` → `h2` → `h3`), and landmarks (`<header>`, `<nav>`,
`<main>`, `<footer>`).

The first rule of ARIA is: don't use ARIA when a native element does the job.

## 2. Every control needs an accessible name

```jsx
<label htmlFor="email">Email</label>
<input id="email" type="email" />

<button type="button" aria-label="Close dialog" onClick={onClose}>
  <XIcon aria-hidden="true" />
</button>

<img src={chart} alt="Sales rose 12% from March to April" />
<img src={divider} alt="" />   {/* decorative: empty alt, skipped by screen readers */}
```

Hard-coded ids break when a component renders twice. Use `useId`:

```jsx
import { useId } from 'react'

function TextField({ label, hint, ...inputProps }) {
  const id = useId()
  const hintId = `${id}-hint`
  return (
    <div className="field">
      <label htmlFor={id}>{label}</label>
      <input id={id} aria-describedby={hint ? hintId : undefined} {...inputProps} />
      {hint && <p id={hintId} className="hint">{hint}</p>}
    </div>
  )
}
```

## 3. Keyboard support

Everything clickable must be reachable with Tab and operable with Enter/Space. Custom
widgets need the keyboard behaviour users expect from their role. The WAI-ARIA Authoring
Practices Guide (APG) documents the expected keys for tabs, menus, comboboxes, etc.

Visible focus matters: never remove outlines without replacing them.

```css
button:focus-visible { outline: 3px solid #2f6fed; outline-offset: 2px; }
```

`:focus-visible` shows the ring for keyboard users without showing it on mouse click.

## 4. Focus management

In a multi-page site, the browser resets focus on each page load. In a SPA you move
content around, so **you** must manage focus.

### Dialogs

The native `<dialog>` element with `showModal()` gives you a focus trap, `Escape` to
close, inert background content, and focus return — for free in modern browsers:

```jsx
import { useEffect, useRef } from 'react'

export function Modal({ open, onClose, title, children }) {
  const ref = useRef(null)

  useEffect(() => {
    const dialog = ref.current
    if (open && !dialog.open) dialog.showModal()
    if (!open && dialog.open) dialog.close()
  }, [open])

  return (
    <dialog ref={ref} onClose={onClose} aria-labelledby="modal-title">
      <h2 id="modal-title">{title}</h2>
      {children}
      <button type="button" onClick={onClose}>Close</button>
    </dialog>
  )
}
```

The `close` event fires when the user presses Escape, and `onClose` keeps React state in
sync. (For multiple modals on one page, generate the title id with `useId`.) If you build
a dialog from `div`s instead, you must implement trapping, Escape handling, `aria-modal`,
background inertness and focus return yourself — which is why a well-tested library
component or the native element is recommended.

### Route changes

After client-side navigation, move focus to the new page's main heading so screen reader
users know something changed:

```jsx
function PageHeading({ children }) {
  const ref = useRef(null)
  useEffect(() => {
    ref.current?.focus()
  }, [])
  return <h1 ref={ref} tabIndex={-1}>{children}</h1>
}
```

`tabIndex={-1}` makes the heading focusable by script without adding it to the Tab
order. Also update `document.title` per page (in React 19 you can render `<title>`
inside a component and React hoists it into `<head>`).

## 5. Announce dynamic changes

Content that changes without a focus move — "Saved", "3 results", validation errors —
should be announced through a **live region**:

```jsx
<p role="status">{saving ? 'Saving…' : savedAt ? 'All changes saved' : ''}</p>
<p role="alert">{error}</p>
```

`role="status"` is polite (announced when the user is idle); `role="alert"` interrupts.
The region must **already be in the DOM** before its text changes; a live region that
mounts together with its message is often not announced. Render the empty container
up front.

## 6. ARIA for custom widgets — carefully

When no native element fits, ARIA describes roles and states. They must stay in sync
with React state:

```jsx
<button
  aria-expanded={open}
  aria-controls={menuId}
  onClick={() => setOpen(o => !o)}
>
  Options
</button>
<ul id={menuId} role="menu" hidden={!open}>...</ul>
```

Wrong ARIA is worse than none: `role="button"` on a `div` promises keyboard behaviour
the `div` doesn't have.

## Worked example: an accessible toggle switch

```jsx
import { useId } from 'react'

export function Switch({ label, checked, onChange, description }) {
  const id = useId()
  return (
    <div className="switch-row">
      <button
        id={id}
        type="button"
        role="switch"
        aria-checked={checked}
        aria-describedby={description ? `${id}-desc` : undefined}
        onClick={() => onChange(!checked)}
        className="switch"
      >
        <span className="switch__thumb" aria-hidden="true" />
      </button>
      <label htmlFor={id}>{label}</label>
      {description && <p id={`${id}-desc`}>{description}</p>}
    </div>
  )
}
```

A `<button>` already handles focus, Enter and Space; `role="switch"` +
`aria-checked` tells assistive tech it's an on/off control; the `<label htmlFor>` gives
it a name and makes the label clickable. Screen readers announce something like
"Email notifications, switch, on".

## Testing accessibility

- **Manual keyboard pass**: unplug the mouse. Can you reach and operate everything? Is
  focus always visible? Does it go somewhere sensible after dialogs close?
- **Screen reader smoke test**: VoiceOver (macOS/iOS), NVDA (Windows, free), TalkBack
  (Android).
- **Linting**: `eslint-plugin-jsx-a11y` catches missing alt text, click handlers on
  non-interactive elements, and more.
- **Automated checks**: axe-core (via browser extensions or `vitest-axe`/`jest-axe` in
  tests) finds a meaningful subset of issues. Automated tools catch only part of the
  problems; they don't replace keyboard and screen reader testing.
- **Testing Library queries by role/label** fail when names are missing — a free check in
  every test.

## How It Actually Works

Browsers build an **accessibility tree** parallel to the DOM: each node has a role,
an accessible name, states (checked, expanded, disabled) and relationships. Screen
readers and other assistive tech read this tree through platform APIs, not your pixels.
Native elements come with correct entries; ARIA attributes **override or add** entries
but add no behaviour.

The accessible name is computed by a defined algorithm: `aria-labelledby` wins, then
`aria-label`, then native labelling (`<label>`, `alt`, a button's text content), then
`title`. That's why an icon-only button with no `aria-label` is announced just as
"button".

React's part is mechanical: it sets ARIA attributes as ordinary DOM attributes
(booleans like `aria-checked={true}` are stringified to `"true"`), and it doesn't touch
focus except for `autoFocus`. So any time React **removes** the focused element (e.g.
closing a menu whose item had focus), focus falls back to `<body>`, and a keyboard user
is thrown to the top of the page. That's why focus must be moved deliberately before or
after such updates, typically in an effect or right after `flushSync`.

## Common mistakes

- **Clickable `div`/`span`** instead of `button`/`a`.
- **Placeholder as label** — it disappears on typing and often has poor contrast.
- **`outline: none`** with no replacement focus style.
- **Live regions mounted with the message** instead of pre-rendered empty.
- **Hard-coded ids** in reusable components; use `useId`.
- **`aria-hidden="true"` on focusable content**, leaving focusable elements invisible to
  screen readers.

## Exercise

Audit and fix a component of yours (or the Level 2 recipe app):

1. Do a keyboard-only pass and list every problem.
2. Add `eslint-plugin-jsx-a11y` and fix all findings.
3. Replace any custom modal with one based on native `<dialog>`, and ensure focus returns
   to the triggering button after closing.
4. Add a polite live region that announces the number of search results after each
   search.
5. Add an axe check to one component test and make it pass.
6. Build a `Tabs` component following the APG tabs pattern: arrow keys move between
   tabs, Home/End jump to first/last, and only the active tab is in the Tab order.
