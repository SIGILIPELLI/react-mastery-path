---
description: "Design a React testing strategy: unit vs integration vs end-to-end tests, Playwright, MSW network mocking, visual tests, and fighting flaky tests."
---

# 07 · Testing Strategy

Level 2 taught *how* to write component tests. At scale, the harder question is *which*
tests to write: every test costs time to write, time to run and time to maintain, and a
suite nobody trusts is worse than a small one people do. This lesson is about spending
your testing effort where it buys confidence.

## The layers

| Layer | Tool (examples) | Tests | Speed | Confidence per test |
|---|---|---|---|---|
| Static | TypeScript, ESLint | typos, wrong types, hook rules, a11y lint | instant | narrow but free |
| Unit | Vitest | pure functions: reducers, domain rules, formatters | ms | high for logic |
| Integration (component) | Vitest + RTL + MSW | a screen or feature with real children, mocked network | tens–hundreds of ms | high for UI behaviour |
| End-to-end | Playwright (or Cypress) | real browser, real app, (often) real backend | seconds | highest, most expensive |
| Visual | Playwright screenshots, Chromatic | pixels | seconds | catches CSS regressions |

A useful shape for React apps is often described as a "testing trophy": a broad base of
static checks, a solid set of unit tests for logic, **most effort in integration tests**,
and a thin layer of E2E tests for critical journeys. Treat this as a heuristic, not a law.

## What to test at each layer

**Unit**: the `model/` functions from the architecture lesson, reducers, custom hooks
with tricky logic, formatting and validation schemas. Cheap, precise failures.

**Integration**: a feature as a user sees it — "filtering the invoice list by status
shows only overdue invoices and updates the total". Render the real feature component
with its real children and providers; mock only the network. Most bugs live in how
pieces connect, which is why this layer gives the best return.

**E2E**: a handful of journeys that must never break — sign up, log in, checkout, the
core workflow. They catch problems nothing else sees: routing, build config, real
browser APIs, the real backend contract.

**Don't test**: implementation details (state variable values, which hook was called),
third-party libraries' own behaviour, or trivial markup.

## Network mocking with MSW

Mock Service Worker intercepts requests at the network layer, so your components' real
`fetch` calls, TanStack Query setup and error handling all run.

```ts
// src/test/handlers.ts
import { http, HttpResponse } from 'msw'

export const handlers = [
  http.get('/api/invoices', ({ request }) => {
    const status = new URL(request.url).searchParams.get('status')
    const all = [
      { id: '1', customer: 'Acme', amount: 1200, status: 'overdue' },
      { id: '2', customer: 'Globex', amount: 800, status: 'paid' },
    ]
    return HttpResponse.json(status ? all.filter(i => i.status === status) : all)
  }),
]
```

```ts
// src/test/setup.ts
import '@testing-library/jest-dom/vitest'
import { setupServer } from 'msw/node'
import { afterAll, afterEach, beforeAll } from 'vitest'
import { handlers } from './handlers'

export const server = setupServer(...handlers)
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

Override per test for error cases:

```tsx
test('shows an error when invoices fail to load', async () => {
  server.use(http.get('/api/invoices', () => new HttpResponse(null, { status: 500 })))
  renderWithProviders(<InvoicesPage />)
  expect(await screen.findByRole('alert')).toHaveTextContent(/couldn't load/i)
})
```

This uses MSW v2's `http`/`HttpResponse` API; older examples use `rest` and `res(ctx…)`.
`onUnhandledRequest: 'error'` makes any un-mocked request fail loudly, so tests can't
accidentally hit real services. (With relative URLs like `/api/...` in Node, make sure
your fetch client resolves them against a base URL in tests.)

## A test-utils module

Centralise providers so every test renders like the app:

```tsx
// src/test/render.tsx
import { render } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { MemoryRouter } from 'react-router'

export function renderWithProviders(ui: React.ReactElement, { route = '/' } = {}) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } })
  return render(
    <QueryClientProvider client={client}>
      <MemoryRouter initialEntries={[route]}>{ui}</MemoryRouter>
    </QueryClientProvider>,
  )
}
```

A **fresh `QueryClient` per test** prevents cached data leaking between tests; `retry:
false` makes error tests fast.

## End-to-end with Playwright

```bash
npm init playwright@latest
```

```ts
// e2e/checkout.spec.ts
import { test, expect } from '@playwright/test'

test('a customer can buy a product', async ({ page }) => {
  await page.goto('/')
  await page.getByRole('link', { name: 'Desk lamp' }).click()
  await page.getByRole('button', { name: 'Add to cart' }).click()
  await page.getByRole('link', { name: /cart \(1\)/i }).click()
  await page.getByLabel('Email').fill('test@example.com')
  await page.getByRole('button', { name: 'Place order' }).click()
  await expect(page.getByRole('heading', { name: 'Order confirmed' })).toBeVisible()
})
```

Playwright's locators mirror Testing Library's role/label queries, and its assertions
**auto-wait** (retrying until the condition holds or times out), which removes most
manual waits. Configure `webServer` in `playwright.config.ts` to start your app
(`npm run preview`) automatically before tests. For E2E, use a dedicated test backend or
seeded database — never production data.

## Flakiness: the suite killer

A test that fails randomly teaches people to ignore failures. Common causes and fixes:

| Cause | Fix |
|---|---|
| Fixed sleeps (`waitForTimeout(2000)`) | auto-waiting assertions / `findBy` |
| Shared state between tests (cache, DB rows, `localStorage`) | isolate: fresh clients, per-test data, cleanup |
| Real time and dates | fake timers / inject "now" (`vi.useFakeTimers()`, `vi.setSystemTime`) |
| Real network in unit/integration tests | MSW with `onUnhandledRequest: 'error'` |
| Animations | reduced motion in tests, or disable animations in test config |
| Test order dependence | run in random order occasionally; each test sets up its own world |

Quarantine a flaky test (and fix it soon) rather than letting it erode trust.

## Worked example: a strategy for one feature

Feature: invoice list with filters, bulk "mark as paid", and export.

- **Unit**: `invoiceStatus()`, `outstandingBalance()`, the CSV builder, the bulk-selection
  reducer.
- **Integration (RTL + MSW)**: filter by status updates rows and total; select-all then
  "mark as paid" sends the right request and updates rows; server error on bulk update
  shows an alert and keeps selection; empty state.
- **E2E (Playwright)**: one journey — log in, filter overdue, mark one paid, see it move to
  "paid". That's the business-critical path.
- **Not tested**: the table's CSS classes, that TanStack Query caches (it's the library's
  job), internal state names.

## How It Actually Works

In Node, MSW's `setupServer` patches the request-making modules (the global `fetch`,
`http`/`https`, `XMLHttpRequest` polyfills) through interceptors, so a request made by your
code is handed to your handlers before any socket is opened. In the browser,
`setupWorker` registers a Service Worker that intercepts real network requests from the
page — the same handlers power both environments.

Playwright drives real browser engines (Chromium, Firefox, WebKit) over their automation
protocols. Each test gets a fresh **browser context** — an isolated profile with its own
cookies and storage — which is cheap to create, so tests can run in parallel without
sharing state. Locators are lazy: `page.getByRole('button', { name: 'Add' })` is a query
re-evaluated on each action, and actions wait for the element to be visible, stable,
enabled and receiving events before clicking. That "actionability" check is why
Playwright tests rarely need explicit waits.

Integration tests with jsdom are fast because there's no real browser: no layout, no
paint, no network. That's also their blind spot — CSS, real focus behaviour, and layout
bugs need E2E or visual tests.

## Common mistakes

- **Mostly E2E tests** → slow, flaky suite; or **mostly shallow unit tests of
  components** → green suite, broken app.
- **Mocking your own modules** heavily in integration tests, so the test verifies the
  mocks.
- **Sharing a `QueryClient`** across tests.
- **Arbitrary sleeps.**
- **Chasing a coverage percentage** instead of covering behaviour that matters.

## Exercise

For one of your projects:

1. Write down a test plan table like the worked example (unit / integration / E2E / not
   tested) for two features.
2. Add MSW with `onUnhandledRequest: 'error'` and a `renderWithProviders` helper; convert
   any `vi.stubGlobal('fetch')` tests to MSW.
3. Write two integration tests including one error path.
4. Add Playwright with a `webServer` config and one E2E test for the most critical journey.
5. Run the whole suite 10 times in a row (`npx vitest run` in a loop, and
   `npx playwright test --repeat-each=10`) and fix anything that fails even once.
