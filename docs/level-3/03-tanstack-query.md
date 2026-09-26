---
description: "Manage server state with TanStack Query: query keys, caching and staleness, background refetching, mutations, invalidation, and optimistic updates."
---

# 03 · Server State with TanStack Query

In Level 2 you wrote `useFetch` and listed what it lacked: caching, deduplication,
background refresh, and mutation handling. TanStack Query (formerly React Query) is a
library that provides exactly those things. It treats server data as a **cache** that
can go stale, not as state you own.

Examples here follow TanStack Query v5's API (object-argument syntax). Older articles
use v3/v4 syntax with positional arguments; the concepts carry over.

## Setup

```bash
npm install @tanstack/react-query
```

```jsx
// main.jsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'

const queryClient = new QueryClient()

createRoot(document.getElementById('root')).render(
  <QueryClientProvider client={queryClient}>
    <App />
  </QueryClientProvider>,
)
```

A separate `@tanstack/react-query-devtools` package adds an in-app panel showing every
cache entry and its state — very useful while learning.

## Your first query

```jsx
import { useQuery } from '@tanstack/react-query'

async function fetchTodos() {
  const res = await fetch('https://jsonplaceholder.typicode.com/todos?_limit=10')
  if (!res.ok) throw new Error(`HTTP ${res.status}`)
  return res.json()
}

function Todos() {
  const { data, isPending, isError, error, isFetching } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  })

  if (isPending) return <p>Loading…</p>
  if (isError) return <p role="alert">{error.message}</p>

  return (
    <>
      {isFetching && <small>Refreshing…</small>}
      <ul>{data.map(t => <li key={t.id}>{t.title}</li>)}</ul>
    </>
  )
}
```

- `queryKey` identifies the data in the cache. Any component using `['todos']` shares one
  cache entry and one request.
- `queryFn` returns a promise; it must **throw** on failure (hence the `res.ok` check).
- `isPending` means "no data yet"; `isFetching` means "a request is in flight", including
  background refetches while old data is shown.

## Keys with parameters

```jsx
function useTodo(id) {
  return useQuery({
    queryKey: ['todos', id],
    queryFn: async ({ signal }) => {
      const res = await fetch(`https://jsonplaceholder.typicode.com/todos/${id}`, { signal })
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      return res.json()
    },
    enabled: id != null,
  })
}
```

The key includes every variable the query depends on — think of it as the dependency
array. Changing `id` switches to a different cache entry; switching back is instant if
that entry is still cached. The `signal` lets the library cancel requests that are no
longer needed. `enabled: false` pauses a query until its inputs exist.

## Staleness and caching: the two timers

- **`staleTime`** (default `0`): how long fetched data is considered fresh. Fresh data is
  served from cache without refetching. Stale data is still shown, but a background
  refetch is triggered on certain events: a new component mounting with that key, the
  window regaining focus, or the network reconnecting.
- **`gcTime`** (default 5 minutes): how long an **unused** cache entry (no components
  subscribed) is kept before being garbage-collected.

```jsx
useQuery({ queryKey: ['countries'], queryFn: fetchCountries, staleTime: 1000 * 60 * 60 })
```

For data that rarely changes, a long `staleTime` avoids pointless refetches. For
fast-changing data, the default is sensible. Choosing `staleTime` per query is the main
tuning decision you make.

## Mutations and invalidation

```jsx
import { useMutation, useQueryClient } from '@tanstack/react-query'

function AddTodo() {
  const queryClient = useQueryClient()
  const mutation = useMutation({
    mutationFn: async title => {
      const res = await fetch('https://jsonplaceholder.typicode.com/todos', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ title, completed: false, userId: 1 }),
      })
      if (!res.ok) throw new Error('Could not save')
      return res.json()
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] })
    },
  })

  return (
    <form onSubmit={e => {
      e.preventDefault()
      mutation.mutate(new FormData(e.currentTarget).get('title'))
      e.currentTarget.reset()
    }}>
      <input name="title" required />
      <button disabled={mutation.isPending}>{mutation.isPending ? 'Saving…' : 'Add'}</button>
      {mutation.isError && <p role="alert">{mutation.error.message}</p>}
    </form>
  )
}
```

`invalidateQueries({ queryKey: ['todos'] })` marks every query whose key **starts with**
`['todos']` as stale and refetches the ones currently on screen. You don't manually edit
the list; you tell the cache it's out of date.

Note: JSONPlaceholder accepts POSTs and returns a fake result but **doesn't persist
anything**, so after invalidation the refetched list won't contain your item. Use it to
see the request flow; use a real or local backend (e.g. `json-server`) to see the data
change.

## Worked example: optimistic toggle with rollback

```jsx
function useToggleTodo() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: async todo => {
      const res = await fetch(`/api/todos/${todo.id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ completed: !todo.completed }),
      })
      if (!res.ok) throw new Error('Update failed')
      return res.json()
    },
    onMutate: async todo => {
      await queryClient.cancelQueries({ queryKey: ['todos'] })
      const previous = queryClient.getQueryData(['todos'])
      queryClient.setQueryData(['todos'], old =>
        old.map(t => (t.id === todo.id ? { ...t, completed: !t.completed } : t)),
      )
      return { previous }
    },
    onError: (_err, _todo, context) => {
      queryClient.setQueryData(['todos'], context.previous)
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] })
    },
  })
}
```

The checkbox flips instantly (`onMutate` writes to the cache), in-flight refetches are
cancelled so they can't overwrite the optimistic value, a failure restores the snapshot,
and either way the list is re-synced with the server afterwards. `/api/todos` stands in
for your own backend.

## How It Actually Works

The `QueryClient` holds a **QueryCache**: a map from a *hashed* query key (keys are
serialised deterministically, so `['todos', { page: 1, sort: 'asc' }]` equals the same
object with its properties in a different order) to a `Query` object. Each `Query`
stores the data, error, status, timestamps and a list of **observers**.

Each `useQuery` call creates a `QueryObserver` that subscribes to its `Query` and plugs
into React via `useSyncExternalStore` (the same mechanism as the stores in lesson 2).
When a query's state changes, observers compute their result and notify their
components. Observers track which result fields your component actually read
(`data`, `isFetching`, …) and only trigger re-renders when those change — so a component
that never reads `isFetching` isn't re-rendered by background refetches starting.

When the first observer mounts, the `Query` checks its data's age against `staleTime`.
Fresh → do nothing. Stale or empty → call `queryFn`. If a fetch for that key is already
in flight, new observers attach to the same promise instead of starting another —
that's deduplication. Window focus and reconnect events are wired globally by a focus
manager and online manager, which tell stale active queries to refetch.

When the last observer unsubscribes, a timer of `gcTime` starts. If nothing subscribes
again before it fires, the query is removed from the cache.

Failed queries retry (by default 3 times with exponential backoff) before reaching the
error state — which is why an error UI appears only after a few seconds. Set
`retry: false` in tests.

## Common mistakes

- **Leaving variables out of the key** → different inputs share one cache entry.
- **`queryFn` that doesn't throw on HTTP errors** → error responses cached as data.
- **Copying query data into `useState`** → your copy goes stale when the cache updates.
  Read from the query directly.
- **Creating the `QueryClient` inside a component** → a new, empty cache on every render.
  Create it once, outside (or in a `useState` initialiser).
- **Tests failing slowly** because of default retries. Use a fresh `QueryClient` per test
  with `retry: false`.

## Exercise

Using `json-server` (`npx json-server db.json`) with a `db.json` containing `projects`
and `tasks`:

1. Show a project list and a selected project's tasks (`['projects']`,
   `['projects', id, 'tasks']`).
2. Add a task (mutation + invalidation of just that project's tasks).
3. Toggle a task optimistically with rollback; stop the server mid-demo to watch the
   rollback happen.
4. Set `staleTime` to 30 seconds for projects and explain, with DevTools open, what
   happens when you switch browser tabs and come back before and after 30 seconds.
5. Prefetch a project's tasks on hover of its link with `queryClient.prefetchQuery`.
