---
description: "How React renders and commits: fibers, reconciliation heuristics for types and keys, batching, bail-outs, and why state resets when identity changes."
---

# 01 · The Render & Reconciliation Model

Most confusing React behaviour — state that resets unexpectedly, state that *doesn't*
reset when you expected it to, effects running "too often", a component re-rendering
when nothing changed — becomes predictable once you know what React does between a
state update and a DOM change. This lesson is that model, with enough detail to reason
about real bugs, and without pretending to document every internal.

## Three phases

1. **Trigger** — something schedules an update: the initial `root.render`, a state
   setter, a `dispatch`, or a context value change.
2. **Render** — React calls component functions to compute what the UI *should* be, and
   compares it with what it was (reconciliation). No DOM is touched. This phase can be
   interrupted, restarted or thrown away in concurrent rendering (Level 4).
3. **Commit** — React applies the computed changes to the DOM, updates refs, runs
   layout effects synchronously, then schedules passive effects (`useEffect`). This
   phase is never interrupted: the user never sees half a commit.

Then the browser paints. Keep this split in mind: "rendering" in React means *calling
your functions*, not *painting pixels*.

## Fibers

React keeps an internal tree of **fiber** objects, one per component instance or host
element (`div`, `p`). A fiber holds the component type, key, props, the hook list (state,
effects, memos), a pointer to the DOM node for host elements, and links to parent,
first child and next sibling.

React maintains two versions of this tree: the **current** tree (what's on screen) and a
**work-in-progress** tree being built during render. At commit, the work-in-progress tree
becomes current. This double buffering is what allows React to prepare an update without
disturbing the screen, and to discard it if something more urgent arrives.

## Reconciliation: comparing old and new

A general algorithm for the minimal difference between two trees is far too slow for UI
(the classic results are around O(n³)). React uses heuristics that make it linear, based
on two assumptions:

1. **Elements of different types produce different trees.**
2. **Keys identify which children are stable across renders.**

### Rule 1: same position, different type → replace

```jsx
{isEditing ? <input value={text} onChange={...} /> : <span>{text}</span>}
```

Switching from `span` to `input` destroys the `span` (and everything below it) and
creates a new `input`. The same applies to components: `<LoginForm />` replaced by
`<SignupForm />` unmounts one and mounts the other, even if they render similar markup.

A subtle consequence:

```jsx
function Parent() {
  function Child() { ... }        // ❌ a new function (a new type) every render
  return <Child />
}
```

Every render creates a **different** `Child` function, so React sees a type change and
remounts it each time — losing state and DOM, and re-running effects. Define components
at module level.

### Rule 2: same position, same type → update

React keeps the fiber (and its state), and:

- for host elements, compares props and updates only changed DOM attributes;
- for components, calls the function again with new props, then recurses into its output.

### Rule 3: lists use keys

For arrays of children, React matches by `key` (Level 1 lesson 6). Without keys it falls
back to index matching.

### What "position" means

Position is the path of child slots from the root. This matters in conditional JSX:

```jsx
<div>
  {showBanner && <Banner />}
  <Counter />
</div>
```

`{showBanner && <Banner />}` always occupies slot 0 (it's `false` when hidden), so
`Counter` always stays in slot 1 and keeps its state when the banner toggles. If you
instead returned two completely different JSX structures via early return, `Counter`
might move to a different slot and remount.

## Worked example: predicting state resets

```jsx
function Scoreboard() {
  const [isPlayerA, setIsPlayerA] = useState(true)
  return (
    <div>
      {isPlayerA ? <Counter person="Taylor" /> : <Counter person="Sarah" />}
      <button onClick={() => setIsPlayerA(!isPlayerA)}>Next player</button>
    </div>
  )
}
```

Clicking "Next player" **keeps** the score: same type, same slot, so React treats it as
the same `Counter` with a new `person` prop. Three ways to make each player independent:

```jsx
// 1. Different keys → different identities
{isPlayerA ? <Counter key="Taylor" person="Taylor" /> : <Counter key="Sarah" person="Sarah" />}

// 2. Different slots
{isPlayerA && <Counter person="Taylor" />}
{!isPlayerA && <Counter person="Sarah" />}

// 3. Keep state above, if both scores must survive switching
```

Options 1 and 2 reset the score on switch (the hidden player's component unmounts). If
both scores must be remembered, the state has to live in the parent.

## Batching

```jsx
function handleClick() {
  setCount(c => c + 1)
  setFlag(f => !f)
  setItems(prev => [...prev, 'x'])
}
```

Three updates, one render. Since React 18 ("automatic batching"), updates are batched
everywhere — inside event handlers, promises, `setTimeout` and native listeners. React
waits until your code yields, then processes all queued updates in one render pass.

Occasionally you need the DOM updated *immediately* (e.g. to scroll to a just-added
item). `flushSync` from `react-dom` forces a synchronous render and commit:

```jsx
import { flushSync } from 'react-dom'

function addMessage(msg) {
  flushSync(() => {
    setMessages(prev => [...prev, msg])
  })
  listRef.current.lastElementChild.scrollIntoView()
}
```

Use it rarely; it defeats batching and can hurt performance.

## Bail-outs: when React skips work

- **Same state**: a setter called with a value `Object.is`-equal to the current state
  usually skips re-rendering. (React may still call the component once before deciding to
  bail out; it won't commit or render children.)
- **Same element**: if a child element is the identical object as last render (e.g.
  `children` from further up), React skips it.
- **`memo`**: props shallowly equal → skip.
- Context changes force consumers to re-render even inside skipped subtrees.

## Keys to reset state, deliberately

```jsx
<ProfileEditor key={userId} userId={userId} />
```

A new `userId` means a new key, so React unmounts the old editor and mounts a fresh one:
all local state, refs and effects start over. This is usually far better than an effect
that tries to "reset state when the prop changes", which renders once with stale state
before correcting itself.

## How It Actually Works

Rendering is a loop over units of work. React starts at the fiber where the update was
scheduled, calls `beginWork` on it (run the component, reconcile its returned children
into child fibers), moves to the first child, and repeats depth-first. When a fiber has
no more children, `completeWork` runs on it (for host elements: prepare the DOM node or
compute the list of changed attributes) and React moves to its sibling or back up to the
parent. Fibers that need DOM changes are flagged with **effect flags** (placement,
update, deletion).

Because the loop is expressed as small units rather than a recursive call stack, React
can pause between units, check whether the browser needs the main thread back
(`shouldYield`), and continue later. That's the foundation of concurrent rendering.

Updates carry a **lane** — a priority bit. A click's update gets a high-priority
synchronous lane; an update wrapped in `startTransition` gets a transition lane; data
arriving later can get lower lanes. React processes the highest-priority lanes first
and can render the same tree more than once for different lanes. This is why render must
be pure: a render may happen, be interrupted and be discarded without committing.

At commit, React walks the flagged fibers and performs DOM mutations, then swaps the
current/work-in-progress pointers, attaches refs, runs `useLayoutEffect` callbacks
synchronously, and schedules `useEffect` callbacks to run after paint.

## Common mistakes

- **Declaring components inside components** → remount every render.
- **Expecting a prop change to reset state.** It doesn't; use a key.
- **Changing a wrapper element conditionally** (`isCompact ? <section>{content}</section> : <div>{content}</div>`)
  → everything inside remounts when it flips.
- **Side effects during render** (logging to analytics, mutating module variables);
  renders can be repeated or discarded.
- **Sprinkling `flushSync`** to "fix" timing issues that are really about reading state
  too early.

## Exercise

1. Build the `Scoreboard` above and confirm the score survives switching. Then implement
   both reset strategies and one "remember both scores" strategy.
2. Build a form whose layout switches between `<div className="stack">` and
   `<div className="grid">` via a toggle. Type in a field and toggle — does the text
   survive? Now switch the wrapper to `<section>` vs `<div>` and repeat. Explain the
   difference using the reconciliation rules.
3. In a component with three state setters called from one click handler, add a
   `console.count('render')` and prove there's one render per click — including when the
   setters run inside a `setTimeout`.
4. Use React DevTools' Profiler with "Record why each component rendered" enabled to
   confirm your predictions.
