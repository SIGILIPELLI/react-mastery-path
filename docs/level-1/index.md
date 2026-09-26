---
description: "Level 1 of React Mastery Path: set up Vite, write JSX, build components with props and state, handle events and forms, and ship an expense tracker."
---

# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: go from an empty folder to a small, working React app you understand line by
line. By the end of this level you will know what a component really is (a function
that returns a description of UI), why state updates trigger a re-render, and how data
moves from parent to child through props.

## Modules

1. [Setup with Vite & Your First Component](01-setup-vite.md) — Node, Vite, the project layout, and what `createRoot` does
2. [JSX in Depth](02-jsx.md) — what JSX compiles to, expressions vs statements, fragments, attribute rules
3. [Components & Props](03-components-props.md) — splitting UI into functions, props as read-only inputs, defaults
4. [State with useState](04-state-usestate.md) — state as a snapshot, updater functions, immutable updates
5. [Handling Events](05-events.md) — synthetic events, passing handlers, `preventDefault`, event propagation
6. [Conditional Rendering, Lists & Keys](06-conditional-lists-keys.md) — `&&`, ternaries, `map`, and why keys matter
7. [Forms & Controlled Inputs](07-forms-controlled-inputs.md) — controlled vs uncontrolled, validation, submit handling
8. [useEffect Basics](08-useeffect-basics.md) — synchronizing with the outside world, dependencies, cleanup
9. [Styling React Apps](09-styling.md) — plain CSS, CSS Modules, inline styles, and utility classes compared
10. [Project — Expense Tracker](10-project-expense-tracker.md) — everything above, combined into one app with persistence

## What you need before starting

- Comfort with modern JavaScript: `const`/`let`, arrow functions, destructuring,
  spread (`...`), template literals, `Array.prototype.map`/`filter`, and ES modules.
- Node.js installed (an LTS release) and a code editor.
- A browser with the React Developer Tools extension is helpful but optional.

By the end of this level you will be able to build a single-page app with several
components, local state, user input, list rendering, and data that survives a page
refresh via `localStorage`.
