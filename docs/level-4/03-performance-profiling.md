---
description: "Profile React apps methodically with Core Web Vitals, the DevTools Profiler, the browser Performance panel, the Profiler API, and a measure-fix loop."
---

# 03 · Performance Profiling

Level 2 introduced memoization with a warning: measure first. This lesson is about the
measuring — the tools, what their numbers mean, and a method that turns "the app feels
slow" into a specific cause and a verified fix.

## What "fast" means to users

Google's **Core Web Vitals** are a widely used, user-centred set of metrics:

- **LCP — Largest Contentful Paint**: when the main content becomes visible (loading).
- **INP — Interaction to Next Paint**: how long the page takes to visually respond to
  clicks, taps and key presses, across the visit (responsiveness).
- **CLS — Cumulative Layout Shift**: how much content jumps around unexpectedly
  (visual stability).

Google publishes "good" thresholds for each on web.dev; check the current values there
rather than relying on remembered numbers. For React apps, INP is where rendering
performance shows up most directly: a click that triggers a 300 ms render is a 300 ms+
interaction.

**Lab vs field**: Lighthouse and dev tools measure on *your* machine (lab data). Real
users on mid-range phones and slow networks are the ones who matter; the `web-vitals`
library can report their metrics (field data) to your analytics.

```js
// main.jsx — report field metrics
import { onCLS, onINP, onLCP } from 'web-vitals'

function send(metric) {
  navigator.sendBeacon('/analytics', JSON.stringify({ name: metric.name, value: metric.value, id: metric.id }))
}
onCLS(send)
onINP(send)
onLCP(send)
```

`/analytics` stands in for your own collection endpoint.

## Tool 1: React DevTools Profiler

1. Open React DevTools → **Profiler** tab. In settings, enable **"Record why each
   component rendered while profiling"**.
2. Press record, perform *one* slow interaction, stop.
3. Read the results:
   - **Commits** bar (top right): each bar is one commit; height/colour show duration.
   - **Flamegraph**: components in the commit; width ≈ render time including children,
     colour = own render time; grey = didn't render.
   - **Ranked**: components sorted by own render time — usually the fastest way to find
     the culprit.
   - Selecting a component shows **why it rendered**: "Props changed: (onSelect)",
     "Hook 2 changed", "The parent component rendered".

"Props changed: onSelect" on a `memo` component is a classic signal: an unstable callback
is defeating memoization.

## Tool 2: Browser Performance panel

The React Profiler only sees React. The browser's Performance panel sees everything:
JavaScript, style recalculation, layout, paint, garbage collection, network.

1. Use a **production build** (`npm run build && npm run preview`) — dev mode adds heavy
   checks and double renders.
2. Enable **CPU throttling** (4× or 6× slowdown) to approximate a mid-range phone.
3. Record the interaction and look for **long tasks** (red-flagged tasks over 50 ms) in
   the Main track. Expand them to see which functions consumed the time.
4. The **Interactions** track shows each interaction's duration, useful for INP.

Recent React versions also integrate with Chrome's performance tooling to show React
activity (scheduler lanes, component render timings) directly in the Performance panel
in some builds; check the React docs for what your version provides.

## Tool 3: the `<Profiler>` API

Measure specific subtrees in code, including in production builds configured for
profiling:

```jsx
import { Profiler } from 'react'

function onRender(id, phase, actualDuration, baseDuration, startTime, commitTime) {
  if (actualDuration > 16) {
    console.warn(`[perf] ${id} ${phase} took ${actualDuration.toFixed(1)}ms (base ${baseDuration.toFixed(1)}ms)`)
  }
}

<Profiler id="OrdersTable" onRender={onRender}>
  <OrdersTable />
</Profiler>
```

- `actualDuration`: time spent rendering this commit (low when memoization works).
- `baseDuration`: estimated time to render the whole subtree without memoization — the
  worst case.

Profiling adds overhead, so it's disabled in standard production builds; production
profiling requires a special build of `react-dom` (see the React docs for the profiling
build alias for your bundler).

## A repeatable method

1. **Reproduce** a specific slow interaction ("typing in the filter box on the orders
   page with 2,000 orders").
2. **Measure** it in a production build with throttling. Write the number down.
3. **Locate** the cost: React Profiler (which components, why) + Performance panel (is it
   even React? could be layout, a third-party script, a huge JSON parse).
4. **Hypothesise and change one thing.**
5. **Re-measure** the same interaction. Keep the change only if the number moved.
6. **Guard** against regression: a performance budget in CI (bundle size limits,
   Lighthouse CI) or a note in the PR.

## Worked example: a slow table

Symptom: selecting a row in a 2,000-row orders table takes ~400 ms at 4× throttling.

- **Profiler, ranked view**: every `OrderRow` rendered; "why": *parent rendered*.
  `OrderRow` isn't memoized.
- Fix 1: `memo(OrderRow)`. Re-measure: still ~380 ms. Profiler "why": *props changed:
  `onSelect`, `style`*.
- `onSelect` is an inline arrow; `style={{ height: 40 }}` is a new object per render.
  Fix 2: `useCallback` for `onSelect`, move the style to CSS. Re-measure: ~60 ms. Only
  two rows render per selection.
- Performance panel: the remaining 60 ms includes ~35 ms of **layout** — 2,000 rows in
  the DOM. Fix 3: virtualize the table (e.g. TanStack Virtual) to render only visible
  rows. Re-measure: well under one frame.

The numbers above are an illustration of the method, not a benchmark — your figures will
differ by device, data and browser. The point is the loop: each fix was chosen because a
measurement pointed at it, and each was verified.

## Other common findings

- **Context value recreated every render** → every consumer re-renders (stabilise with
  `useMemo`, split contexts).
- **Store selectors returning new objects** → re-render on every store change.
- **Large lists** → virtualization.
- **Expensive derived data** → `useMemo`, or compute on the server.
- **Layout thrashing** (reading `offsetHeight` then writing styles in a loop) → batch
  reads before writes.
- **Too much JavaScript at startup** → code-splitting (Level 3), lighter dependencies.
- **Images** without dimensions (CLS) or not lazy-loaded; oversized images hurting LCP.

## How It Actually Works

When profiling is enabled, React records timestamps around each fiber's render work
(`beginWork`/`completeWork`) and accumulates `actualDuration` per fiber (its own time plus
its children's time in this commit). `baseDuration` is maintained as the most recent
render time of each fiber summed over its subtree, so it estimates the cost of
re-rendering everything even when parts were skipped. DevTools reads these values from
the fibers at each commit, plus the reasons it can infer by comparing previous and
current props, state and hooks.

Browsers measure INP from event timing entries: for each interaction, the time from the
input event to the next frame being presented. It includes **input delay** (waiting for
the main thread to be free — e.g. a long render already in progress), **processing time**
(your event handlers, and synchronous React renders they trigger) and **presentation
delay** (style, layout, paint). React work can affect all three, which is why moving
non-urgent renders into transitions (lesson 2) can improve INP: they yield, reducing
input delay for the next interaction.

## Common mistakes

- **Profiling in development mode** and "fixing" dev-only overhead.
- **Changing several things at once** — you won't know which helped.
- **Optimising what doesn't matter** — a 2 ms component that renders rarely.
- **Only testing on a fast laptop**; always throttle.
- **Trusting lab data alone** for decisions about real users.

## Exercise

1. Take one of your apps (the Kanban project is a good candidate) and seed it with 2,000
   cards.
2. Pick the slowest interaction you can find; record baseline numbers in a production
   build with 4× CPU throttling.
3. Use the Profiler (with "why did this render") and the Performance panel to identify at
   least two causes.
4. Apply one fix at a time and record the numbers in a table.
5. Add `web-vitals` logging to the console and note INP for your interaction before and
   after.
