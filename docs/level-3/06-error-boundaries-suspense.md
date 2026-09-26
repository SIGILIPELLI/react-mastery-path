---
description: "Contain rendering failures with React error boundaries, declare loading states with Suspense, and understand what each can and cannot catch."
---

# 06 · Error Boundaries & Suspense

Two components let you handle "not ready" and "went wrong" **declaratively**, by
wrapping parts of the tree, instead of threading `isLoading`/`error` checks through
every component:

- An **error boundary** catches errors thrown while rendering its children and shows a
  fallback.
- **`<Suspense>`** shows a fallback while something inside it is still loading.

## Error boundaries

Without one, an error thrown during render unmounts the **entire** app, leaving a blank
page. An error boundary limits the damage to its subtree.

Error boundaries are the one feature that still requires a **class component** — there is
no hook equivalent. You write one once (or use a library) and then use it like any
component:

```jsx
import { Component } from 'react'

export class ErrorBoundary extends Component {
  state = { error: null }

  static getDerivedStateFromError(error) {
    return { error }                     // render fallback on next render
  }

  componentDidCatch(error, info) {
    // report to your monitoring service
    console.error(error, info.componentStack)
  }

  reset = () => this.setState({ error: null })

  render() {
    if (this.state.error) {
      return this.props.fallback
        ? this.props.fallback({ error: this.state.error, reset: this.reset })
        : <p role="alert">Something went wrong.</p>
    }
    return this.props.children
  }
}
```

```jsx
<ErrorBoundary fallback={({ error, reset }) => (
  <div role="alert">
    <p>Couldn't load the chart: {error.message}</p>
    <button onClick={reset}>Try again</button>
  </div>
)}>
  <RevenueChart />
</ErrorBoundary>
```

The widely used `react-error-boundary` package provides a ready-made `ErrorBoundary`
with `fallbackRender`, `onReset`, `resetKeys` and a `useErrorBoundary` hook — a sensible
choice instead of maintaining your own.

### What boundaries catch — and don't

| Caught | Not caught |
|---|---|
| errors thrown during rendering of children | errors in event handlers |
| errors in lifecycle methods / effects of children | errors in async code (`setTimeout`, promise callbacks) that isn't rethrown during render |
| errors in constructors of child class components | errors in the boundary itself |
| | server-side rendering errors (handled differently) |

For event handlers, use `try/catch` and set error state. To push an async error into the
nearest boundary, set state and throw during render:

```jsx
function useThrowAsyncError() {
  const [, setState] = useState()
  return error => setState(() => { throw error })
}
```

(`react-error-boundary`'s `showBoundary` from `useErrorBoundary` does the same thing.)

### Where to place boundaries

- One near the root, so users see a friendly page instead of a blank screen.
- Around independent widgets (charts, feeds, third-party embeds) so one failure doesn't
  take out the page.
- Around each route's content, so navigation still works when one page breaks.

## Suspense

`<Suspense fallback={...}>` shows the fallback while any component inside it
**suspends** — that is, while it's waiting for something that isn't ready yet.

Things that suspend:

- **Lazy components** (`React.lazy`, next lesson) while their code downloads.
- **Suspense-enabled data fetching**: TanStack Query's `useSuspenseQuery`, framework
  loaders, and React 19's `use(promise)` with a cached promise.
- Not: data fetched in a `useEffect`. Effects run after render; Suspense can't see them.

```jsx
import { Suspense } from 'react'
import { useSuspenseQuery } from '@tanstack/react-query'

function Profile({ id }) {
  const { data: user } = useSuspenseQuery({ queryKey: ['user', id], queryFn: () => fetchUser(id) })
  return <h2>{user.name}</h2>          // data is always defined here
}

function ProfilePage({ id }) {
  return (
    <ErrorBoundary fallback={({ error }) => <p role="alert">{error.message}</p>}>
      <Suspense fallback={<p>Loading profile…</p>}>
        <Profile id={id} />
      </Suspense>
    </ErrorBoundary>
  )
}
```

`Profile` contains no loading or error branches: loading is declared by the nearest
`Suspense`, errors by the nearest error boundary.

### `use(promise)` in React 19

```jsx
import { use, Suspense } from 'react'

function Comments({ commentsPromise }) {
  const comments = use(commentsPromise)   // suspends until resolved
  return comments.map(c => <p key={c.id}>{c.text}</p>)
}
```

The promise must be **created outside** the suspending component and be stable across
renders (created by a parent, a cache, a framework loader, or passed from a Server
Component). Creating it inside the component (`use(fetch(...))`) makes a new promise
each render, so it never settles from React's point of view. `use` can also read context
and, unlike other hooks, can be called conditionally.

## Designing loading states

- **Nesting** controls granularity: one `Suspense` around the page shows one spinner;
  one per panel lets panels appear independently.
- Everything inside one boundary **reveals together**, which avoids layout jumping around
  as small pieces pop in.
- Skeletons that match the final layout feel faster than spinners.

```jsx
<Suspense fallback={<PageSkeleton />}>
  <Header />
  <Suspense fallback={<ChartSkeleton />}>
    <SalesChart />
  </Suspense>
  <Suspense fallback={<TableSkeleton />}>
    <OrdersTable />
  </Suspense>
</Suspense>
```

## Worked example: a resilient dashboard widget

```jsx
import { Suspense, useId } from 'react'
import { ErrorBoundary } from 'react-error-boundary'
import { useQueryErrorResetBoundary } from '@tanstack/react-query'

function Widget({ title, children }) {
  const { reset } = useQueryErrorResetBoundary()
  const headingId = useId()
  return (
    <section className="widget" aria-labelledby={headingId}>
      <h2 id={headingId}>{title}</h2>
      <ErrorBoundary
        onReset={reset}
        fallbackRender={({ error, resetErrorBoundary }) => (
          <div role="alert">
            <p>{error.message}</p>
            <button onClick={resetErrorBoundary}>Retry</button>
          </div>
        )}
      >
        <Suspense fallback={<div className="skeleton" aria-busy="true" />}>{children}</Suspense>
      </ErrorBoundary>
    </section>
  )
}

<Widget title="Signups"><SignupsChart /></Widget>
<Widget title="Revenue"><RevenueTable /></Widget>
```

Each widget loads, fails and retries independently. `useQueryErrorResetBoundary` tells
TanStack Query to retry the failed query when the boundary resets, rather than
immediately re-throwing the cached error.

## How It Actually Works

**Error boundaries.** When a component throws during render, React unwinds the fiber
tree upward looking for the nearest class component with `getDerivedStateFromError`.
It discards the partially rendered work below it, calls `getDerivedStateFromError` to
compute new state, and re-renders the boundary, which now returns its fallback.
`componentDidCatch` is called during commit for side effects like logging. If no
boundary exists, React unmounts the whole root. Event handlers aren't covered because
they run outside rendering — by the time a click handler throws, React isn't in the
middle of building a tree it could replace.

**Suspense.** A component suspends by **throwing a promise** (or, with `use`, by React
internally tracking a pending thenable). React catches it like an error, but looks for
the nearest `<Suspense>` boundary instead of an error boundary. It renders that
boundary's fallback in place of its children, and attaches a callback to the promise.
When the promise settles, React retries rendering the children. If the promise
rejected, the retry throws the error, which then goes to the nearest error boundary.

On retry, React **re-renders the suspended component from scratch** — state inside a
component that never finished its first mount is not preserved. That's why the data must
come from a cache that outlives the render (the TanStack cache, a framework, a promise
created higher up).

If content that is **already visible** suspends again because of an update wrapped in
`startTransition`, React keeps showing the old UI instead of flashing back to the
fallback. Level 4's concurrency lesson builds on this.

## Common mistakes

- **Expecting boundaries to catch event handler or async errors.**
- **One boundary around the whole app only** → any widget failure blanks every page.
- **Creating a promise inside the component** passed to `use` → infinite suspension.
- **Wrapping effect-based fetching in Suspense** and wondering why the fallback never
  shows.
- **No way to recover** — fallbacks should offer retry or navigation.

## Exercise

1. Build a dashboard with three widgets using TanStack Query's `useSuspenseQuery`, each
   wrapped in its own `Suspense` + error boundary.
2. Make one widget's fetch fail 50% of the time (`Math.random()` in the `queryFn`) and
   show a working Retry.
3. Add an app-level error boundary with a "Reload page" button, and throw from an event
   handler to confirm it is *not* caught. Then route that error into the boundary with
   `showBoundary`.
4. Rearrange the `Suspense` boundaries so the whole dashboard reveals at once, then so
   each widget reveals independently. Describe which you'd ship and why.
