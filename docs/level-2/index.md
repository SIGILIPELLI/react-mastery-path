---
description: "Level 2 of React Mastery Path: composition, lifting state, useReducer, context, custom hooks, data fetching, routing, memoization, and testing."
---

# Level 2 · Intermediate <span class="level-badge">Real Apps</span>

Goal: move from single-screen demos to multi-page applications with shared state,
network data and tests. Level 1 taught you what a component is; Level 2 is about how
components **cooperate** — where state should live, how data travels, and how to keep
logic reusable and verified.

## Modules

1. [Composition & children](01-composition-children.md) — building flexible components from smaller ones instead of configuration props
2. [Lifting State & Data Flow](02-lifting-state.md) — finding the right owner for state, and avoiding duplicated or derived state
3. [useReducer](03-usereducer.md) — centralising complex update logic in a pure reducer
4. [Context](04-context.md) — passing values deep without prop drilling, and what it costs
5. [Custom Hooks](05-custom-hooks.md) — extracting reusable stateful logic, and the Rules of Hooks
6. [Data Fetching, Loading & Errors](06-data-fetching.md) — effects that fetch correctly, race conditions, abort, and UI states
7. [Routing with React Router](07-react-router.md) — URLs as state, nested layouts, params and navigation
8. [Performance Basics: memo, useMemo, useCallback](08-performance-memo.md) — what re-renders really cost, and when *not* to memoize
9. [Testing with Vitest & Testing Library](09-testing-vitest-rtl.md) — testing behaviour the way a user experiences it
10. [Project — Recipe Finder SPA](10-project-recipe-finder.md) — a routed app with search, fetching, favourites via context, and tests

## Before you start

You should be comfortable with everything in Level 1, especially: state as a snapshot,
immutable updates, keys, and effects with correct dependencies. Lesson 6 onward uses a
public, no-key-required API for examples; where an example needs a server you don't
have, the lesson says so and shows how to mock it.

By the end of this level you'll be able to structure a medium-sized app, share state
without chaos, fetch data safely, navigate between pages, and prove behaviour with
automated tests.
