---
title: "React Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free React cheat sheet: 16 quick-reference tables from every level of the React Mastery Path, each linked to its full lesson."
---

# React Cheat Sheet

Quick-reference tables from **16 lessons** of the free React Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · Setup with Vite &amp; Your First Component](level-1/01-setup-vite.md)

*Useful scripts*

| Command | What it does |
|---|---|
| `npm run dev` | Dev server with Fast Refresh |
| `npm run build` | Production bundle into `dist/` (minified, dev checks stripped) |
| `npm run preview` | Serves `dist/` locally so you can test the real build |
| `npm run lint` | Runs ESLint, including the React Hooks rules |

### [02 · JSX in Depth](level-1/02-jsx.md)

*Attributes: camelCase and a few renames*

| HTML | JSX |
|---|---|
| `class` | `className` |
| `for` | `htmlFor` |
| `onclick` | `onClick` |
| `tabindex` | `tabIndex` |
| `style="color: red"` | `style={{ color: 'red' }}` |

### [04 · State with useState](level-1/04-state-usestate.md)

*Objects and arrays: replace, don't mutate*

| Goal | Avoid (mutates) | Use (returns new) |
|---|---|---|
| add | `push`, `unshift` | `[...arr, item]`, `[item, ...arr]` |
| remove | `splice` | `arr.filter(x => x.id !== id)` |
| replace | `arr[i] = x` | `arr.map(x => x.id === id ? {...x, done: true} : x)` |
| sort | `sort()` | `[...arr].sort()` or `arr.toSorted()` |

### [07 · Forms &amp; Controlled Inputs](level-1/07-forms-controlled-inputs.md)

*Other input types*

| Element | Controlled via | Read in handler |
|---|---|---|
| text, email, number, textarea | `value` | `e.target.value` (a string) |
| checkbox | `checked` | `e.target.checked` |
| radio group | `checked={choice === 'x'}` | `e.target.value` |
| select | `value` on `<select>` | `e.target.value` |
| multi-select | `value={array}` + `multiple` | `[...e.target.selectedOptions].map(o => o.value)` |

### [08 · useEffect Basics](level-1/08-useeffect-basics.md)

*Shape of an effect*

| Dependencies | Effect runs |
|---|---|
| `[a, b]` | after mount, and after any render where `a` or `b` changed |
| `[]` | after mount only (and again on remount) |
| omitted | after every render |

### [10 · Project — Expense Tracker](level-1/10-project-expense-tracker.md)

*Plan the components and state first*

| Data | Kind | Lives in |
|---|---|---|
| list of expenses | state | `App` (list, summary and form all need it) |
| category filter, month filter | state | `App` (list and summary both use them) |
| form field values | state | `ExpenseForm` (nobody else cares until submit) |
| filtered list, totals, breakdown | **derived** | computed in `App` during render |

## Level 2 · Intermediate

### [03 · useReducer](level-2/03-usereducer.md)

*When to choose `useReducer` over `useState`*

| Signal | Lean towards |
|---|---|
| One or two independent values | `useState` |
| Several values that change together | `useReducer` |
| Next state depends on previous state in complex ways | `useReducer` |
| Many handlers with similar update code | `useReducer` |
| You want to unit test state transitions without rendering | `useReducer` |

### [09 · Testing with Vitest &amp; Testing Library](level-2/09-testing-vitest-rtl.md)

*Querying like a user*

| Prefix | No match | Use for |
|---|---|---|
| `getBy…` | throws | element should be there now |
| `queryBy…` | returns `null` | asserting something is **absent** |
| `findBy…` | rejects after timeout | element appears **asynchronously** (returns a promise) |

## Level 3 · Advanced

### [02 · Client State: Zustand vs Redux Toolkit](level-3/02-state-management.md)

*An honest comparison*

| | Zustand | Redux Toolkit |
|---|---|---|
| Setup | one `create` call | store + slices + `Provider` |
| Structure | free-form; you decide conventions | opinionated: slices, actions, reducers |
| Update style | call actions directly | dispatch action objects |
| DevTools / time travel | via middleware | excellent, built in |
| Server data | pair with TanStack Query | RTK Query included |
| Best fit | small–medium apps, quick shared UI state | large teams wanting strict conventions and traceability |
| Cost | easy to create inconsistent patterns at scale | more concepts and ceremony up front |

### [05 · Forms at Scale](level-3/05-forms-at-scale.md)

*Choosing*

| Need | Good fit |
|---|---|
| A few fields, simple rules | plain controlled/uncontrolled inputs |
| Many fields, dynamic arrays, complex client validation | React Hook Form + Zod |
| Progressive enhancement, server-first framework | form actions (`useActionState`) |
| Shared validation client + server | a schema library (Zod or similar) regardless |

### [06 · Error Boundaries &amp; Suspense](level-3/06-error-boundaries-suspense.md)

*What boundaries catch — and don't*

| Caught | Not caught |
|---|---|
| errors thrown during rendering of children | errors in event handlers |
| errors in lifecycle methods / effects of children | errors in async code (`setTimeout`, promise callbacks) that isn't rethrown during render |
| errors in constructors of child class components | errors in the boundary itself |
| | server-side rendering errors (handled differently) |

## Level 4 · Master

### [01 · Server Components &amp; Frameworks](level-4/01-server-components.md)

*Two kinds of components*

| | Server Components | Client Components |
|---|---|---|
| Where they run | server / build time only | server (for the initial HTML) **and** browser |
| Can use state, effects, event handlers | no | yes |
| Can use browser APIs | no | yes (in effects/handlers) |
| Can read files, databases, secrets directly | yes | no |
| Code sent to browser | none | yes |
| Can be `async` | yes | no |

### [02 · Concurrent Rendering](level-4/02-concurrent-rendering.md)

*Worked example: choosing between them*

| Situation | Tool |
|---|---|
| You own the `setState` that triggers the slow render | `useTransition` |
| The slow part receives a value from a prop or a hook you don't control | `useDeferredValue` |
| You want a pending indicator for navigation or a tab switch | `useTransition` (`isPending`) |
| You want to show stale results dimmed while new ones render | either; compare value vs deferred value, or use `isPending` |
| The work is one huge synchronous computation | neither — optimise it, memoize it, or move it to a worker |

### [05 · Architecture for Large Apps](level-4/05-architecture.md)

*State placement at scale*

| Kind | Where |
|---|---|
| Server data | a server-state cache (TanStack Query) or framework loaders/RSC |
| URL-representable (filters, tabs, pagination, selected id) | the URL |
| Form drafts | the form (RHF or local state) |
| Local UI (open/closed, hover) | component state |
| Cross-cutting client state (theme, current workspace, feature flags) | context or a small store |

### [07 · Testing Strategy](level-4/07-testing-strategy.md)

*The layers*

| Layer | Tool (examples) | Tests | Speed | Confidence per test |
|---|---|---|---|---|
| Static | TypeScript, ESLint | typos, wrong types, hook rules, a11y lint | instant | narrow but free |
| Unit | Vitest | pure functions: reducers, domain rules, formatters | ms | high for logic |
| Integration (component) | Vitest + RTL + MSW | a screen or feature with real children, mocked network | tens–hundreds of ms | high for UI behaviour |
| End-to-end | Playwright (or Cypress) | real browser, real app, (often) real backend | seconds | highest, most expensive |
| Visual | Playwright screenshots, Chromatic | pixels | seconds | catches CSS regressions |

### [08 · Building &amp; Deploying](level-4/08-deployment.md)

*Caching strategy*

| File | Header | Why |
|---|---|---|
| `/assets/*-[hash].js`, `.css`, fonts, images | `Cache-Control: public, max-age=31536000, immutable` | the name changes whenever content changes, so caching "forever" is safe |
| `index.html` | `Cache-Control: no-cache` (revalidate every time) | it's the entry point that points to the current hashes |

---

Go deeper with the full [React Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
