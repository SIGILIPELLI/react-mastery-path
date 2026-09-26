---
description: "Create a React project with Vite, understand every generated file, and see how createRoot mounts your first component into the page."
---

# 01 · Setup with Vite & Your First Component

React is a library for describing user interfaces as a function of data. You write
small functions called **components** that return what the UI should look like, and
React takes care of making the browser's DOM match that description. This lesson gets a
real project running and walks through every file so nothing feels like magic.

## What you need installed

1. **Node.js (LTS)** — check with `node --version`. Vite needs a reasonably recent
   Node; if the version check in step 2 complains, upgrade Node first.
2. **npm** — ships with Node (`npm --version`). `pnpm` or `yarn` also work; this
   course uses npm commands.
3. **An editor** — VS Code is common, but anything with JavaScript syntax
   highlighting is fine.

## Why Vite?

For years the default was Create React App (CRA). It is now deprecated and the React
team no longer recommends it. The two paths the React docs point to today are:

- **A framework** (Next.js, React Router in framework mode, etc.) when you need
  routing, server rendering and data loading built in. Level 4 covers these.
- **A build tool** such as Vite when you want a plain client-side React app and to
  learn React itself without framework conventions on top.

For learning, Vite is ideal: it starts in well under a second, has almost no config,
and the generated project is small enough to read completely.

## Create the project

```bash
npm create vite@latest react-lab -- --template react
cd react-lab
npm install
npm run dev
```

The `--` passes `--template react` through npm to the Vite scaffolder. Use
`--template react-ts` if you want TypeScript from day one (Level 3 lesson 8 covers
TypeScript with React; plain JavaScript is used until then).

`npm run dev` prints a local URL (by default `http://localhost:5173`). Open it and you
should see the Vite + React starter page with a counter button.

## Tour of the generated files

```text
react-lab/
├── index.html          # the single HTML page; Vite serves it as the entry point
├── package.json        # dependencies and scripts (dev, build, preview, lint)
├── vite.config.js      # enables the React plugin (JSX transform + Fast Refresh)
├── public/             # files copied as-is (favicon etc.)
└── src/
    ├── main.jsx        # boots React and mounts <App />
    ├── App.jsx         # the root component
    ├── App.css
    └── index.css
```

### `index.html`

Unlike older toolchains, Vite treats `index.html` as source code. The important parts:

```html
<body>
  <div id="root"></div>
  <script type="module" src="/src/main.jsx"></script>
</body>
```

There is exactly one empty `div`. React will own everything inside it.

### `src/main.jsx`

```jsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import './index.css'
import App from './App.jsx'

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

Three things happen here:

- `createRoot(domNode)` creates a React **root** attached to that DOM node.
- `.render(<App />)` tells the root what to display. React calls your `App`
  function, takes what it returns, and creates the matching DOM nodes.
- `<StrictMode>` is a development-only helper. It intentionally calls your component
  functions twice and runs effects setup → cleanup → setup once on mount, to surface
  bugs caused by impure code. It does nothing in production builds.

### `vite.config.js`

```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
})
```

The React plugin compiles JSX to plain JavaScript and wires up **Fast Refresh**, which
swaps edited components in the browser without losing their state.

## Write your first component

Replace everything in `src/App.jsx` with:

```jsx
function Greeting() {
  const hour = new Date().getHours()
  const part = hour < 12 ? 'morning' : hour < 18 ? 'afternoon' : 'evening'
  return <p>Good {part}! Welcome to React.</p>
}

export default function App() {
  return (
    <main>
      <h1>React Lab</h1>
      <Greeting />
      <Greeting />
    </main>
  )
}
```

Delete the `import './App.css'` line if you removed the file's usage, or keep it — it
does no harm. Save and the browser updates immediately.

Notice:

- A component is a **plain function whose name starts with a capital letter**. The
  capital letter is how JSX tells `<Greeting />` (your component) apart from `<p>`
  (a built-in HTML tag).
- It **returns JSX** — markup-like syntax that describes UI. Lesson 2 shows what that
  compiles to.
- You **use** a component by writing it as a tag. Each `<Greeting />` is a separate
  call to the function.

## Worked example: a component tree

Split a small page into components to see how the tree forms:

```jsx
function Header() {
  return (
    <header>
      <h1>Plant Care Log</h1>
    </header>
  )
}

function Tip() {
  return <p>Water succulents only when the soil is fully dry.</p>
}

function Footer() {
  return <footer>Made while learning React</footer>
}

export default function App() {
  return (
    <>
      <Header />
      <Tip />
      <Footer />
    </>
  )
}
```

React sees this as a tree: `App` at the root with three children. When React renders
`App`, it gets back elements of type `Header`, `Tip` and `Footer`; it then calls each of
those functions in turn until everything bottoms out in real HTML tags. The `<>...</>`
is a **fragment**, which groups siblings without adding an extra DOM node.

## Useful scripts

| Command | What it does |
|---|---|
| `npm run dev` | Dev server with Fast Refresh |
| `npm run build` | Production bundle into `dist/` (minified, dev checks stripped) |
| `npm run preview` | Serves `dist/` locally so you can test the real build |
| `npm run lint` | Runs ESLint, including the React Hooks rules |

## How It Actually Works

When the browser loads `index.html`, it requests `/src/main.jsx` as an ES module. In
development, Vite does **not** bundle your app. Its dev server intercepts each module
request, transforms that one file on demand (JSX → JavaScript, rewriting bare imports
like `'react'` to pre-bundled dependency files), and returns it. That is why startup is
fast no matter how large the app grows: only files the browser actually asks for get
processed.

Once `main.jsx` runs, `createRoot` sets up React's internal bookkeeping for that DOM
container. Calling `root.render(<App />)` schedules a render:

1. **Render phase** — React calls `App()`. The returned JSX is a tree of plain
   objects ("React elements") like `{ type: Greeting, props: {} }`. For each element
   whose `type` is a function, React calls that function too, recursively, building an
   internal tree (the "fiber" tree) describing the whole UI.
2. **Commit phase** — React walks that tree and creates the actual DOM nodes
   (`document.createElement('main')`, etc.), then inserts them into `#root` in one go.

On later updates, React renders again, **compares** the new element tree with the
previous one, and only touches DOM nodes that changed. Level 3's reconciliation lesson
goes deep on that comparison.

Fast Refresh works because the Vite plugin injects registration code around each
component. When you save a file, Vite pushes the new module over a WebSocket; React
swaps the function implementation for existing component instances and re-renders them,
keeping their state as long as the hooks inside have the same order and types.

## Common mistakes

- **Lower-case component names.** `function greeting()` used as `<greeting />` is
  treated as an unknown HTML tag and renders nothing useful. Always capitalize.
- **Returning two siblings without a wrapper.** `return <h1/><p/>` is a syntax error;
  wrap them in a fragment `<>...</>` or an element.
- **Being surprised by double logs.** A `console.log` in a component prints twice in
  development under `StrictMode`. That is intentional, not a bug in your code.
- **Opening `index.html` directly from disk.** The file only works when served by Vite;
  use `npm run dev` or `npm run preview`.
- **Following old CRA tutorials.** `react-scripts`, `ReactDOM.render` and
  `serviceWorker.js` belong to older setups. `ReactDOM.render` was removed in React 19.

## Exercise

1. Create a new Vite React project called `profile-card`.
2. Build three components: `Avatar` (renders an `<img>` with any image URL and an
   `alt` text), `Bio` (a heading with a name and a paragraph about that person) and
   `ProfileCard` which renders both inside a `<section>`.
3. Render `ProfileCard` twice from `App`.
4. Add a `console.log('ProfileCard rendered')` inside `ProfileCard`. Count how many
   times it logs in dev, then run `npm run build && npm run preview` and count again.
   Explain the difference in one sentence.
