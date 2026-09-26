---
description: "Animate React UIs with CSS transitions, enter and exit animations, the Motion library for layout and gestures, and reduced-motion preferences."
---

# 09 · Animation

Good animation explains change: where a panel came from, which item was removed, that a
button did something. Bad animation is slow, janky, or makes some users physically
unwell. This lesson goes from plain CSS (enough for most UI) to a library for the hard
cases, with performance and accessibility built in.

## 1. CSS transitions driven by state

React changes a class or style; CSS animates between the values.

```jsx
function Drawer({ open, children }) {
  return (
    <aside className={open ? 'drawer drawer--open' : 'drawer'} aria-hidden={!open} inert={!open}>
      {children}
    </aside>
  )
}
```

```css
.drawer {
  position: fixed; inset: 0 auto 0 0; width: 320px;
  transform: translateX(-100%);
  transition: transform 250ms ease;
}
.drawer--open { transform: translateX(0); }
```

The element stays mounted; only a class changes. This is the cheapest and most robust
technique. `inert` (a boolean attribute supported by React 19; in React 18 you'd set it
via a ref or pass `inert=""`) removes the hidden drawer's contents from tab order and the
accessibility tree.

### Animate cheap properties

Browsers can animate `transform` and `opacity` on the compositor without recalculating
layout. Animating `width`, `height`, `top` or `left` forces layout work on every frame
and is more likely to stutter. Prefer `transform: translate/scale` and `opacity`.

## 2. Keyframe animations on mount

A newly mounted element can animate in with a CSS animation:

```css
@keyframes fade-up {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: none; }
}
.toast { animation: fade-up 200ms ease-out; }
```

Every time a `.toast` mounts, it plays. Mount animations are easy.

## 3. The exit problem

**Exits** are hard: when state says "remove this", React removes the DOM node immediately
— there's nothing left to animate. You need to keep the element mounted until its exit
animation finishes:

```jsx
import { useEffect, useState } from 'react'

function useDelayedUnmount(show, ms) {
  const [render, setRender] = useState(show)
  useEffect(() => {
    if (show) {
      setRender(true)
      return
    }
    const id = setTimeout(() => setRender(false), ms)
    return () => clearTimeout(id)
  }, [show, ms])
  return render
}

function Toast({ show, message }) {
  const render = useDelayedUnmount(show, 200)
  if (!render) return null
  return <div className={show ? 'toast' : 'toast toast--leaving'} role="status">{message}</div>
}
```

```css
.toast--leaving { animation: fade-out 200ms ease-in forwards; }
@keyframes fade-out { to { opacity: 0; transform: translateY(8px); } }
```

This works, but timings are duplicated in JS and CSS, and lists of exiting items get
complicated fast. That's where a library helps.

## 4. Motion (formerly Framer Motion)

Motion is a widely used React animation library. Install with `npm install motion`; the
React API is imported from `motion/react`. (Older projects use the `framer-motion`
package with the same component API.)

```jsx
import { AnimatePresence, motion } from 'motion/react'

function Notifications({ items, onDismiss }) {
  return (
    <ul className="stack">
      <AnimatePresence initial={false}>
        {items.map(n => (
          <motion.li
            key={n.id}
            layout
            initial={{ opacity: 0, y: -8 }}
            animate={{ opacity: 1, y: 0 }}
            exit={{ opacity: 0, x: 40 }}
            transition={{ duration: 0.2 }}
          >
            {n.text}
            <button onClick={() => onDismiss(n.id)} aria-label="Dismiss">×</button>
          </motion.li>
        ))}
      </AnimatePresence>
    </ul>
  )
}
```

- `motion.li` is an `li` that accepts animation props.
- `initial` → `animate` runs on mount; `exit` runs on removal because
  **`AnimatePresence`** keeps removed children mounted until their exit completes.
- **Keys are essential**: `AnimatePresence` detects removal by keys disappearing.
- `layout` animates position changes: when an item is removed, siblings slide into place
  instead of jumping.

### Shared layout transitions

```jsx
{tabs.map(tab => (
  <button key={tab} onClick={() => setActive(tab)} className="tab">
    {tab}
    {active === tab && <motion.span layoutId="underline" className="underline" />}
  </button>
))}
```

Elements with the same `layoutId` are treated as one element moving between places: the
underline glides between tabs.

## 5. Respecting reduced motion

Some users set "reduce motion" in their OS because animation can trigger dizziness or
nausea. Honour it.

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

In Motion, `useReducedMotion()` returns `true` when the preference is set, and a
`MotionConfig reducedMotion="user"` wrapper disables transform/layout animations for
those users automatically. A good rule: under reduced motion, replace movement with
simple fades, or no animation.

## Worked example: an animated, accessible accordion

```jsx
import { useId, useState } from 'react'
import { AnimatePresence, motion, useReducedMotion } from 'motion/react'

export function AccordionItem({ title, children }) {
  const [open, setOpen] = useState(false)
  const id = useId()
  const reduce = useReducedMotion()

  return (
    <div className="acc-item">
      <h3>
        <button aria-expanded={open} aria-controls={id} onClick={() => setOpen(o => !o)}>
          {title}
          <motion.span aria-hidden="true" animate={{ rotate: open ? 180 : 0 }} style={{ display: 'inline-block' }}>
            ▾
          </motion.span>
        </button>
      </h3>
      <AnimatePresence initial={false}>
        {open && (
          <motion.div
            id={id}
            key="panel"
            initial={{ height: 0, opacity: 0 }}
            animate={{ height: 'auto', opacity: 1 }}
            exit={{ height: 0, opacity: 0 }}
            transition={{ duration: reduce ? 0 : 0.25 }}
            style={{ overflow: 'hidden' }}
          >
            <div className="acc-body">{children}</div>
          </motion.div>
        )}
      </AnimatePresence>
    </div>
  )
}
```

Animating `height` to `'auto'` is something CSS transitions historically couldn't do;
Motion measures the content to make it work. Height animation does cost layout on each
frame — acceptable for a short, occasional accordion, not for a list of 200 items.

## A note on the View Transitions API

Browsers are adding a native **View Transitions API** (`document.startViewTransition`)
that snapshots the page before and after a DOM update and animates between them. React
has been developing built-in integration (a `<ViewTransition>` component) that was
experimental at the time of writing. Check the React docs for its current status before
depending on it.

## How It Actually Works

A browser frame roughly goes: run JavaScript → recalculate styles → **layout** (compute
geometry) → **paint** (fill pixels into layers) → **composite** (combine layers on the
GPU). At 60 Hz, all of that has about 16 ms. Changing `transform` or `opacity` on an
element promoted to its own layer only requires compositing, which the browser can do
on a separate compositor thread even while the main thread is busy running React. That's
why those two properties stay smooth when others stutter.

Motion's `animate` props are not re-rendered through React on every frame. Motion reads
the target values when the component renders, then drives the animation itself with
`requestAnimationFrame` (or the Web Animations API where possible), writing styles
directly to the DOM node. React only re-renders when *targets* change, not 60 times a
second.

`layout` animations use the **FLIP** technique: **F**irst (measure the element's box
before the change), **L**ast (let React commit, measure again), **I**nvert (apply a
transform that makes it *look* like it's still in the first position), **P**lay (animate
the transform to zero). The layout changes instantly; only a cheap transform is animated.

`AnimatePresence` works by keeping its own copy of the previous children. When a keyed
child disappears from props, it keeps rendering the old element, marks it as exiting,
and removes it from its internal list after the `exit` animation's promise resolves.

## Common mistakes

- **Animating layout properties** (`width`, `top`) for large or frequent animations.
- **Missing keys** on children of `AnimatePresence` — exits never run.
- **Conditionally rendering `AnimatePresence` itself** — it must stay mounted to see its
  children leave.
- **Long durations** — most UI transitions should be roughly 150–300 ms.
- **Ignoring `prefers-reduced-motion`.**
- **Leaving hidden content focusable** in off-screen drawers; use `inert` or unmount it.

## Exercise

1. Build a to-do list where added items slide in, deleted items fade and collapse, and
   remaining items glide into place (Motion with `AnimatePresence` + `layout`).
2. Build the same enter/exit effect for a single toast using only CSS and the
   `useDelayedUnmount` hook, and compare the amount of code.
3. Add a tab bar with a sliding underline using `layoutId`.
4. Turn on reduced motion in your OS and make sure every animation becomes a fade or
   disappears.
5. Record a Performance trace in dev tools while animating `left` vs `transform` on 50
   elements, and describe the difference you see in the frames.
