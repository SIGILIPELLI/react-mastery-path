---
description: "Fetch data in React effects without race conditions, cancel requests with AbortController, and model loading, error, empty and success states explicitly."
---

# 06 · Data Fetching, Loading & Errors

Almost every app loads data from a server. Doing it inside `useEffect` looks simple,
but a naive version has real bugs: responses arriving out of order, state updates after
unmount, and UI that forgets to handle failure. This lesson builds a correct fetching
pattern from scratch, then explains why most production apps eventually hand this job
to a library or framework.

The examples use `https://jsonplaceholder.typicode.com`, a free public fake API that
needs no key. If it's unreachable from your network, any JSON endpoint works the same
way.

## The naive version

```jsx
function UserList() {
  const [users, setUsers] = useState([])
  useEffect(() => {
    fetch('https://jsonplaceholder.typicode.com/users')
      .then(res => res.json())
      .then(setUsers)
  }, [])
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>
}
```

Problems: no loading indicator, errors are swallowed, a 404/500 response is treated as
success (`fetch` only rejects on *network* failure), and nothing handles changing inputs.

## Model the states explicitly

```jsx
const [state, setState] = useState({ status: 'idle', data: null, error: null })
```

One `status` field (`'idle' | 'loading' | 'success' | 'error'`) instead of separate
`isLoading`/`isError` booleans that could contradict each other.

## A correct fetch effect

```jsx
import { useEffect, useState } from 'react'

function UserPosts({ userId }) {
  const [state, setState] = useState({ status: 'loading', data: null, error: null })

  useEffect(() => {
    const controller = new AbortController()
    setState({ status: 'loading', data: null, error: null })

    async function load() {
      try {
        const res = await fetch(
          `https://jsonplaceholder.typicode.com/posts?userId=${userId}`,
          { signal: controller.signal },
        )
        if (!res.ok) throw new Error(`Request failed: ${res.status}`)
        const data = await res.json()
        setState({ status: 'success', data, error: null })
      } catch (err) {
        if (err.name === 'AbortError') return // we cancelled it; not a real error
        setState({ status: 'error', data: null, error: err })
      }
    }

    load()
    return () => controller.abort()
  }, [userId])

  if (state.status === 'loading') return <p aria-busy="true">Loading posts…</p>
  if (state.status === 'error') return <p role="alert">Couldn't load posts: {state.error.message}</p>
  if (state.data.length === 0) return <p>This user hasn't posted yet.</p>

  return (
    <ul>
      {state.data.map(p => <li key={p.id}>{p.title}</li>)}
    </ul>
  )
}
```

What each part fixes:

- **`res.ok` check** turns HTTP errors into real errors.
- **`AbortController` + cleanup** cancels the in-flight request when `userId` changes or
  the component unmounts.
- **Ignoring `AbortError`** stops a cancelled request from flashing an error.
- **Four UI states** (loading, error, empty, data) are all handled. Empty is not an error.

## The race condition, concretely

Without the abort, pick user 1, then quickly user 2. Two requests are in flight. If
user 1's response is slower, the order is: user 2 arrives → UI shows user 2 → user 1
arrives → UI shows **user 1's posts while user 2 is selected**. Aborting in cleanup
guarantees only the latest request can update state. If an API client can't be
aborted, use an `ignore` flag instead:

```jsx
useEffect(() => {
  let ignore = false
  getPosts(userId).then(data => {
    if (!ignore) setPosts(data)
  })
  return () => {
    ignore = true
  }
}, [userId])
```

## Extracting `useFetch`

```jsx
export function useFetch(url) {
  const [state, setState] = useState({ status: 'loading', data: null, error: null })
  const [reloadToken, setReloadToken] = useState(0)

  useEffect(() => {
    if (!url) return
    const controller = new AbortController()
    setState(s => ({ ...s, status: 'loading', error: null }))

    fetch(url, { signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`)
        return res.json()
      })
      .then(data => setState({ status: 'success', data, error: null }))
      .catch(error => {
        if (error.name !== 'AbortError') setState({ status: 'error', data: null, error })
      })

    return () => controller.abort()
  }, [url, reloadToken])

  return { ...state, reload: () => setReloadToken(t => t + 1) }
}
```

Keeping `data` while status is `'loading'` (`...s`) lets you show stale results dimmed
during a refetch instead of a blank spinner. The `reloadToken` gives a "Try again"
button something to change.

## Worked example: search with debounce

```jsx
import { useState } from 'react'
import { useDebouncedValue } from './useDebouncedValue' // from lesson 5
import { useFetch } from './useFetch'

export default function CountrySearch() {
  const [query, setQuery] = useState('')
  const q = useDebouncedValue(query.trim(), 400)
  const url = q ? `https://restcountries.com/v3.1/name/${encodeURIComponent(q)}?fields=name,capital,flag` : null
  const { status, data, error, reload } = useFetch(url)

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} placeholder="Search countries" />
      {!q && <p>Type a country name.</p>}
      {q && status === 'loading' && <p>Searching…</p>}
      {q && status === 'error' && (
        <p role="alert">
          {error.message === 'HTTP 404' ? 'No matches.' : 'Something went wrong.'}{' '}
          <button onClick={reload}>Try again</button>
        </p>
      )}
      {q && status === 'success' && (
        <ul>
          {data.map(c => (
            <li key={c.name.common}>{c.flag} {c.name.common} — {c.capital?.[0] ?? 'no capital'}</li>
          ))}
        </ul>
      )}
    </div>
  )
}
```

The REST Countries API is public and free at the time of writing, but third-party APIs
change; if its response shape differs, adjust the fields. Note `encodeURIComponent` —
never paste raw user input into URLs.

## Why libraries exist

Even the correct hook above lacks things real apps need:

- **Caching**: navigating away and back refetches from scratch.
- **Deduplication**: three components asking for the same user make three requests.
- **Background refetch** when the window regains focus or data goes stale.
- **Mutations** and keeping lists in sync after them.
- **Waterfalls**: effects only start after render, so nested components fetch one after
  another.

TanStack Query (Level 3 lesson 3) solves the first four; frameworks with route loaders
and Server Components (Level 4) attack the waterfall problem. Writing it by hand once is
still the best way to understand what those tools do for you.

## How It Actually Works

The effect runs **after** the component has rendered and been painted, so the first
paint always shows the initial state (here, `'loading'`). That's why effect-based
fetching can't avoid a loading flash, and why data for a child can't start loading until
its parent has finished loading and rendered it — the "request waterfall".

Each effect execution is a closure over the render that created it, so its `userId` is
fixed. When `userId` changes, React runs the previous effect's cleanup (aborting *that*
request) before running the new effect. `controller.abort()` makes the pending `fetch`
promise reject with a `DOMException` named `AbortError` and tells the browser to drop
the connection if the response hasn't arrived.

Under `StrictMode` in development, the mount → unmount → mount cycle means you'll see two
requests in the Network tab, the first one cancelled. That's the cleanup working, not a
bug; in production there's one.

Since React 18, calling `setState` on an unmounted component is silently ignored (the old
"can't perform a React state update on an unmounted component" warning was removed),
but ignoring *responses from outdated requests* is still your job — React can't know
which response is stale.

## Common mistakes

- **Forgetting `res.ok`** — a 500 page becomes "data".
- **Making the effect function `async`** directly. Define an inner async function.
- **No cleanup** → race conditions when inputs change quickly.
- **Objects in the dependency array** (`useFetch(url, { headers })` with an inline
  object) → refetch every render.
- **Fetching in response to a click inside an effect** (set a flag, watch it). Fetch in
  the event handler instead; effects are for "data this screen needs to show".
- **Treating empty results as errors**, or not handling them at all.

## Exercise

Build a "GitHub-style" user browser using `https://jsonplaceholder.typicode.com`:

1. A list of users (`/users`) on the left; clicking one loads their posts (`/posts?userId=`)
   and albums (`/albums?userId=`) on the right in **parallel** (hint: `Promise.all`
   inside one effect, sharing one `AbortController`).
2. Show a skeleton while loading, an error with a retry button, and an empty state.
3. Throttle your network to "Slow 3G" in dev tools and click users rapidly; prove the
   right user's data always wins.
4. Add a tiny in-memory cache (a `Map` outside the component keyed by URL) so revisiting
   a user shows data instantly while refetching in the background. Write down two
   problems this cache has that a real library would solve.
