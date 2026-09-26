---
description: "React capstone: plan, build, test and deploy a production-grade team dashboard using feature architecture, a design system, server state, i18n and CI."
---

# 10 · Capstone — Team Dashboard

The capstone is a real application you design and build yourself, applying every level
of the course. Rather than a copy-along tutorial, this page gives you a specification,
the architecture decisions to make (with recommended defaults), milestones, the
key code seams, and a review checklist. Plan for it to take a few weeks of part-time
work; that's normal for something portfolio-worthy.

## The product

**"Standup"** — a dashboard for a small team to track work and check in daily.

Core features:

1. **Projects & tasks**: list projects; per project, a task table with status, assignee,
   priority and due date; create/edit/delete tasks; filter and sort (state in the URL).
2. **Daily check-in**: each member submits "yesterday / today / blockers"; the team view
   shows today's check-ins and highlights blockers.
3. **Insights**: charts for tasks completed per week and overdue tasks per assignee.
4. **Settings**: profile, language (at least two, including one RTL is a stretch goal),
   theme (light/dark).
5. **Auth**: a login screen; routes other than login require a session.

Non-functional requirements:

- Keyboard- and screen-reader-usable; zero axe violations on key screens.
- Initial JS for the login route under a budget you set and enforce in CI.
- INP "good" for filtering 1,000 tasks on a throttled CPU.
- Errors contained per widget; failed requests show retry.
- Strict TypeScript; CI runs lint, types, unit/integration tests, E2E, build.

## Backend options

You need an API. Choose one and **write down why** (an ADR):

- **Local mock server** — `json-server` or MSW in the browser (`setupWorker`) with seeded
  data. Easiest; fine for a portfolio as long as you're explicit that data is mocked.
- **Your own small API** — Node/Express, FastAPI, or similar, with a SQLite database.
  More work, much more realistic (real auth, validation, errors).
- **An RSC framework** — Server Components reading the database directly and Server
  Functions for mutations (Level 4 lesson 1). Changes the architecture significantly;
  good if you want framework experience.

The rest of this page assumes the SPA path (Vite + React Router + TanStack Query) against
a REST API; adapt if you chose the framework path.

## Architecture

```text
src/
  app/
    main.tsx              # providers, router, i18n bootstrap
    router.tsx            # route tree with lazy routes
    RequireAuth.tsx
    AppErrorBoundary.tsx
  features/
    auth/                 # session query, login form, useSession()
    projects/             # project list, project detail layout
    tasks/                # task table, filters (URL), task form, mutations
      model/              # status rules, overdue calculation, sorting
    checkins/             # daily form (RHF + Zod), team view
    insights/             # charts (lazy-loaded chart library)
    settings/
  shared/
    ui/                   # design system: tokens.css, Button, TextField, Dialog, Stack, Table
    api/                  # fetch client: base URL, JSON, errors, auth header/cookies
    i18n/
    lib/                  # formatters (Intl, cached), cx, dates
  test/                   # MSW handlers, renderWithProviders, setup
e2e/
```

Decisions to record as ADRs (recommended defaults in brackets):

1. Server state: [TanStack Query; query keys per feature in `features/*/api/keys.ts`].
2. Client state: [URL for filters; context for theme/locale; no global store unless a
   measured need appears].
3. Forms: [React Hook Form + Zod; schemas shared with the backend if it's TypeScript].
4. Styling: [CSS Modules + tokens as CSS custom properties].
5. Auth storage: [HttpOnly session cookie set by the API; `useSession()` query for the
   current user].

## Key seams

### A typed API client

```ts
// shared/api/client.ts
export class ApiError extends Error {
  constructor(public status: number, message: string, public body?: unknown) {
    super(message)
  }
}

export async function api<T>(path: string, init: RequestInit = {}): Promise<T> {
  const res = await fetch(`${import.meta.env.VITE_API_URL}${path}`, {
    credentials: 'include',
    ...init,
    headers: { 'Content-Type': 'application/json', ...init.headers },
  })
  if (res.status === 204) return undefined as T
  const body = await res.json().catch(() => undefined)
  if (!res.ok) throw new ApiError(res.status, (body as { message?: string })?.message ?? res.statusText, body)
  return body as T
}
```

Validate responses with Zod schemas at the feature boundary (`features/tasks/api`)
rather than trusting the cast.

### Session and protected routes

```tsx
// features/auth/useSession.ts
export function useSession() {
  return useQuery({
    queryKey: ['session'],
    queryFn: () => api<User>('/session').catch(e => (e instanceof ApiError && e.status === 401 ? null : Promise.reject(e))),
    staleTime: 5 * 60_000,
  })
}

// app/RequireAuth.tsx
export function RequireAuth() {
  const { data: user, isPending } = useSession()
  const location = useLocation()
  if (isPending) return <FullPageSpinner />
  if (!user) return <Navigate to="/login" replace state={{ from: location }} />
  return <Outlet />
}
```

A 401 becomes `null` ("logged out") rather than an error; other failures still reach the
error boundary. Remember the API enforces authorisation — this only shapes the UI.

### Filters in the URL, work in a transition

```tsx
// features/tasks/useTaskFilters.ts
export function useTaskFilters() {
  const [params, setParams] = useSearchParams()
  const [isPending, startTransition] = useTransition()
  const filters = {
    status: params.get('status') ?? 'all',
    assignee: params.get('assignee') ?? 'all',
    q: params.get('q') ?? '',
    sort: params.get('sort') ?? 'due',
  }
  function update(key: keyof typeof filters, value: string) {
    startTransition(() => {
      setParams(prev => {
        const next = new URLSearchParams(prev)
        if (value === 'all' || value === '') next.delete(key)
        else next.set(key, value)
        return next
      }, { replace: true })
    })
  }
  return { filters, update, isPending }
}
```

Search-as-you-type needs the text input to stay urgent: keep the input's own value in
local state, and push it to the URL (inside the transition) — debounced or on each change.

### Pure domain rules

```ts
// features/tasks/model/tasks.ts
export function isOverdue(task: Task, today: string) {
  return task.status !== 'done' && task.dueDate !== null && task.dueDate < today
}

export function filterAndSortTasks(tasks: Task[], f: TaskFilters, today: string): Task[] {
  const q = f.q.toLowerCase()
  return tasks
    .filter(t => f.status === 'all' || t.status === f.status)
    .filter(t => f.assignee === 'all' || t.assigneeId === f.assignee)
    .filter(t => t.title.toLowerCase().includes(q))
    .toSorted(compareBy(f.sort, today))
}
```

`compareBy` is yours to write. Passing `today` in (instead of calling `new Date()` inside)
keeps the functions pure and makes tests deterministic.

### Lazy, resilient widgets

```tsx
// features/insights/InsightsPage.tsx
const CompletedChart = lazy(() => import('./CompletedChart'))
const OverdueChart = lazy(() => import('./OverdueChart'))

export default function InsightsPage() {
  const { t } = useTranslation()
  return (
    <Stack gap={4}>
      <Widget title={t('insights.completed')}><CompletedChart /></Widget>
      <Widget title={t('insights.overdue')}><OverdueChart /></Widget>
    </Stack>
  )
}
```

`Widget` is the error-boundary + Suspense wrapper from Level 3 lesson 6 (use `useId` for
its heading). The chart library loads only on this route.

## Milestones

1. **Skeleton** — repo, strict TS, lint (incl. hooks and jsx-a11y rules and import
   boundaries), Vitest + RTL + MSW, Playwright, CI workflow, design tokens, app shell with
   routes and a 404. *Deployable on day one.*
2. **Auth + projects** — login, session, protected routes, project list and detail.
3. **Tasks** — table, URL filters, create/edit dialog (RHF + Zod), optimistic status
   change, delete with undo toast.
4. **Check-ins** — form, team view, blockers highlighted, empty and error states.
5. **Insights** — lazy charts with accessible data tables as alternatives.
6. **Settings, i18n, theming** — language switch with lazy catalogs, dark theme via
   tokens, RTL stretch goal.
7. **Hardening** — performance pass with profiler numbers, a11y audit, security review
   (`dangerouslySetInnerHTML`, URLs, secrets), error monitoring hook, bundle budget in CI.
8. **Ship** — production deploy with correct cache headers and SPA fallback, README with
   architecture overview, ADRs, and known limitations.

## Testing plan (minimum)

- **Unit**: every `model/` function; reducers; the API client's error handling.
- **Integration**: task filtering via URL; create task (success + validation + server
  error); optimistic status change with rollback; check-in submission; language switch
  changes formatted dates.
- **E2E**: log in → create a task → mark it done → see it in insights; one run at a
  mobile viewport.
- **Accessibility**: axe checks on the task table, task dialog and check-in form.

## Review checklist

Use this before calling it done — ideally ask someone else to review with it:

- [ ] No component defined inside another; no index keys on dynamic lists.
- [ ] No derived data stored in state; no effects that only sync state to state.
- [ ] All server data through the query layer; keys include every parameter.
- [ ] URL holds filters/sort/pagination; reload and share work.
- [ ] Every async UI has loading, empty, error (with retry) and success states.
- [ ] Error boundaries at app, route and widget level.
- [ ] Keyboard-only walkthrough passes; focus visible; dialogs return focus.
- [ ] No `dangerouslySetInnerHTML` without sanitizing; URLs from data validated; no
      secrets in the bundle.
- [ ] Measured performance numbers recorded for the main interactions.
- [ ] All user-visible strings translated; `Intl` for every number and date.
- [ ] CI green: lint, types, tests, E2E, build, bundle budget.
- [ ] README explains how to run it, the architecture, and the trade-offs you made.

## How It Actually Works

The capstone is where the mechanisms from earlier lessons meet. When a user changes the
status filter:

1. The handler calls `update('status', 'blocked')`. Inside `startTransition`,
   `setParams` performs a `history.replaceState` and updates the router's location state
   at transition priority.
2. React renders the task route at low priority. `useTaskFilters` reads the new params;
   `filterAndSortTasks` recomputes (memoized on `[tasks, filters, today]`); the table
   reconciles rows by task id. If the user types meanwhile, that urgent update interrupts
   and the transition render restarts with the latest URL.
3. Nothing hits the network: the task list comes from the TanStack Query cache, and
   filtering is derived client-side. If filtering were server-side, the query key would
   include the filters, the new key would start a fetch, and — because the update is a
   transition — the old table would stay visible (dimmed via `isPending`) instead of
   flashing a skeleton.
4. On commit, only changed rows touch the DOM; the live region announcing
   "12 tasks shown" updates its text and screen readers read it politely.

Every decision in the review checklist exists to keep that pipeline predictable: pure
renders so interrupted work is harmless, stable keys so reconciliation keeps the right
state, one source of truth per piece of data so nothing drifts.

## Exercise

Build Standup to milestone 8, then write a one-page retrospective:

1. Which three decisions would you make again, and which one would you change?
2. Where did you reach for memoization, and what measurement justified each case?
3. Which bug took longest to find, and which mental model from this course explained it?
4. What would have to change to support 50 teams and 100,000 tasks?

Publish the code (without secrets) with a README, link a live demo if you deployed one,
and state clearly what is mocked.
