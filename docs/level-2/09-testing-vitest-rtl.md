---
description: "Set up Vitest with React Testing Library, query elements the way users find them, simulate interactions with user-event, and test async and mocked fetches."
---

# 09 · Testing with Vitest & Testing Library

Tests let you change code without fear. For React components, the most useful tests
render a component, interact with it like a user would, and check what the user would
see. **Vitest** runs the tests (it shares Vite's config and transforms, so JSX just
works), and **React Testing Library (RTL)** renders components and finds elements.

## Setup

```bash
npm install -D vitest jsdom @testing-library/react @testing-library/dom @testing-library/user-event @testing-library/jest-dom
```

```js
// vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: './src/test/setup.js',
    globals: true,
  },
})
```

```js
// src/test/setup.js
import '@testing-library/jest-dom/vitest'
```

```json
// package.json scripts
"test": "vitest"
```

`jsdom` simulates a browser DOM in Node. `jest-dom` adds matchers such as
`toBeInTheDocument()` and `toHaveValue()`. With `globals: true`, RTL automatically
cleans up the rendered DOM after each test.

## A first test

```jsx
// Counter.jsx
import { useState } from 'react'
export default function Counter({ initial = 0 }) {
  const [n, setN] = useState(initial)
  return (
    <>
      <p>Count: {n}</p>
      <button onClick={() => setN(c => c + 1)}>Increment</button>
    </>
  )
}
```

```jsx
// Counter.test.jsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import Counter from './Counter'

test('increments when the button is clicked', async () => {
  const user = userEvent.setup()
  render(<Counter initial={2} />)

  await user.click(screen.getByRole('button', { name: /increment/i }))

  expect(screen.getByText('Count: 3')).toBeInTheDocument()
})
```

Run `npm test`. Vitest watches files and re-runs affected tests on save.

## Querying like a user

RTL encourages queries that reflect how people (including screen reader users) find
things. In rough priority order:

1. `getByRole('button', { name: 'Save' })` — the accessible role and name.
2. `getByLabelText('Email')` — form fields by their label.
3. `getByPlaceholderText`, `getByText`, `getByDisplayValue`.
4. `getByTestId('...')` — last resort, when nothing user-visible identifies it.

Query variants:

| Prefix | No match | Use for |
|---|---|---|
| `getBy…` | throws | element should be there now |
| `queryBy…` | returns `null` | asserting something is **absent** |
| `findBy…` | rejects after timeout | element appears **asynchronously** (returns a promise) |

`getAllBy…`/`queryAllBy…`/`findAllBy…` return arrays.

If `getByRole` can't find your button, that's often a real accessibility bug (a clickable
`div`, an unlabeled icon button) — the test is doing double duty.

## `user-event` vs `fireEvent`

`userEvent` simulates full interactions: typing fires keydown, input and keyup for each
character; clicking fires pointer and mouse events and moves focus. `fireEvent` dispatches
a single DOM event. Prefer `userEvent`; its methods are async, so `await` them.

## Testing a form

```jsx
// LoginForm.test.jsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { vi } from 'vitest'
import LoginForm from './LoginForm'

test('shows validation and submits valid credentials', async () => {
  const user = userEvent.setup()
  const onSubmit = vi.fn()
  render(<LoginForm onSubmit={onSubmit} />)

  await user.click(screen.getByRole('button', { name: /sign in/i }))
  expect(screen.getByText(/email is required/i)).toBeInTheDocument()
  expect(onSubmit).not.toHaveBeenCalled()

  await user.type(screen.getByLabelText(/email/i), 'asha@example.com')
  await user.type(screen.getByLabelText(/password/i), 'correct-horse')
  await user.click(screen.getByRole('button', { name: /sign in/i }))

  expect(onSubmit).toHaveBeenCalledWith({ email: 'asha@example.com', password: 'correct-horse' })
})
```

`vi.fn()` creates a mock function that records calls. The test never inspects component
state — only what's rendered and what callbacks received.

## Worked example: testing async fetching

The component from lesson 6 calls `fetch`. In tests, replace it so tests are fast and
deterministic:

```jsx
// UserPosts.test.jsx
import { render, screen } from '@testing-library/react'
import { vi, afterEach } from 'vitest'
import UserPosts from './UserPosts'

afterEach(() => vi.unstubAllGlobals())

test('renders posts after loading', async () => {
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
    ok: true,
    json: async () => [{ id: 1, title: 'Hello world' }],
  }))

  render(<UserPosts userId={1} />)

  expect(screen.getByText(/loading posts/i)).toBeInTheDocument()
  expect(await screen.findByText('Hello world')).toBeInTheDocument()
  expect(screen.queryByText(/loading posts/i)).not.toBeInTheDocument()
})

test('shows an error for a failed response', async () => {
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue({ ok: false, status: 500 }))
  render(<UserPosts userId={1} />)
  expect(await screen.findByRole('alert')).toHaveTextContent(/500/)
})
```

For larger apps, **Mock Service Worker (MSW)** intercepts requests at the network level,
so the same mocks work in tests and in the browser, and your components' real `fetch`
code runs. It's worth adopting once you have more than a handful of mocked endpoints.

## Testing hooks and context

Test hooks through a component that uses them when you can. For reusable hooks,
`renderHook` exists:

```jsx
import { renderHook, act } from '@testing-library/react'
import { useToggle } from './useToggle'

test('useToggle flips', () => {
  const { result } = renderHook(() => useToggle(false))
  act(() => result.current[1]())
  expect(result.current[0]).toBe(true)
})
```

Components that need providers or a router get a `wrapper`:

```jsx
render(<Account />, {
  wrapper: ({ children }) => (
    <MemoryRouter initialEntries={['/account']}>
      <AuthProvider>{children}</AuthProvider>
    </MemoryRouter>
  ),
})
```

(`MemoryRouter` comes from `react-router`; it keeps history in memory instead of the
real URL.)

## How It Actually Works

Vitest runs each test file in Node using Vite's transform pipeline, so your JSX and
imports compile exactly as in the app. `environment: 'jsdom'` creates a `window` and
`document` implemented in JavaScript — enough DOM to render and query, but with no
layout engine: `getBoundingClientRect()` returns zeros, and CSS isn't applied. That's why
RTL tests check structure and text, not pixels.

`render()` calls `createRoot` on a fresh container attached to `document.body` and
renders your element inside React's `act()` helper. `act` tells React "we're in a test":
it flushes all pending state updates and effects before returning, so assertions see the
settled UI. RTL wraps `render`, `userEvent` and `fireEvent` in `act` for you, which is
why you rarely call it directly.

Asynchronous work (a mocked `fetch` resolving) happens *after* `act` returns. `findBy…`
handles that by polling: it re-runs the query on an interval and on DOM mutations
(default timeout 1000 ms) until it succeeds or times out. If you see the warning
"An update to X inside a test was not wrapped in act(...)", some state update happened
after your test stopped waiting — usually you forgot to `await` a `findBy` or a user
event.

## Common mistakes

- **Testing implementation details** — asserting on state variables, internal function
  calls or CSS class names. Refactors then break tests that should pass.
- **Forgetting `await`** on `user.click`/`user.type` or on `findBy…`.
- **Using `getBy` to assert absence** — it throws before your assertion runs. Use
  `queryBy…` with `.not.toBeInTheDocument()`.
- **Hitting real APIs in tests** → slow, flaky, and dependent on someone else's server.
- **Reaching for `getByTestId` first** instead of roles and labels.

## Exercise

For the expense tracker from Level 1 (or any app of yours):

1. Configure Vitest + RTL as above.
2. Write tests that: add an expense through the form; show a validation error for amount
   `0`; delete an expense; filter by category and check the total changes.
3. Stub `localStorage` data before rendering to test that saved expenses load on start
   (jsdom provides `localStorage`; clear it in `beforeEach`).
4. Break the app on purpose (e.g. make delete remove the wrong item) and confirm a test
   fails with a readable message. Then fix it.
