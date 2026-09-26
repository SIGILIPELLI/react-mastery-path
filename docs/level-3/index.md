---
description: "Level 3 of React Mastery Path: reconciliation, Zustand vs Redux Toolkit, TanStack Query, accessibility, Suspense, code-splitting, TypeScript, animation."
---

# Level 3 · Advanced <span class="level-badge">Production</span>

Goal: understand React deeply enough to make architecture decisions, and build UIs that
hold up in production — accessible, resilient to errors, fast to load, and type-safe.
Level 3 starts by opening up the render model itself, because every advanced topic
after it (state libraries, Suspense, concurrency) is easier once you know exactly what
React does between `setState` and a DOM change.

## Modules

1. [The Render & Reconciliation Model](01-render-reconciliation.md) — fibers, the diffing heuristics, render vs commit, batching
2. [Client State: Zustand vs Redux Toolkit](02-state-management.md) — external stores, selectors, and honest trade-offs
3. [Server State with TanStack Query](03-tanstack-query.md) — caching, staleness, mutations and invalidation
4. [Accessibility in React](04-accessibility.md) — semantics first, focus management, ARIA done right, testing a11y
5. [Forms at Scale](05-forms-at-scale.md) — React Hook Form + Zod, and React 19 form actions
6. [Error Boundaries & Suspense](06-error-boundaries-suspense.md) — containing failures and declaring loading states
7. [Code-Splitting & Lazy Loading](07-code-splitting.md) — `lazy`, dynamic `import()`, route-level splitting, bundle analysis
8. [TypeScript with React](08-typescript-react.md) — typing props, hooks, events, generics and context
9. [Animation](09-animation.md) — CSS transitions, enter/exit animations, Motion, and reduced-motion
10. [Project — Kanban Board](10-project-kanban.md) — a typed, accessible, drag-and-drop board with a store and server-state caching

## Before you start

Be comfortable with everything in Level 2: custom hooks, context, reducers, effect-based
fetching and its limits, and RTL tests. From lesson 8 onward examples mix JavaScript and
TypeScript; the project is written in TypeScript.

Several lessons introduce third-party libraries (Zustand, Redux Toolkit, TanStack Query,
React Hook Form, Zod, Motion). Their APIs are stable in broad shape but change between
major versions, so each lesson focuses on the concepts and points out where to confirm
details in the library's own documentation.
