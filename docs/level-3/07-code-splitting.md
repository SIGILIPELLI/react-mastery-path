---
description: "Shrink initial JavaScript in React apps with lazy and dynamic import, split by route, preload chunks, and analyse bundles to find what to split."
---

# 07 · Code-Splitting & Lazy Loading

By default, a bundler puts your whole app into one JavaScript file. Every user downloads
and parses the admin panel, the chart library and the rich-text editor before seeing the
login page. **Code-splitting** breaks the bundle into chunks that load when needed.

## Dynamic `import()`

The foundation is a standard JavaScript feature:

```js
// static: included in the main bundle
import { formatReport } from './reports'

// dynamic: becomes a separate chunk, fetched when this line runs
const { formatReport } = await import('./reports')
```

Vite (Rollup under the hood for production builds) sees `import('./reports')` and emits
`reports-[hash].js` as a separate file.

Use it for heavy non-component code triggered by actions:

```jsx
async function handleExport() {
  const { utils, writeFile } = await import('xlsx')   // big library loaded only on click
  const sheet = utils.json_to_sheet(rows)
  const book = utils.book_new()
  utils.book_append_sheet(book, sheet, 'Report')
  writeFile(book, 'report.xlsx')
}
```

(`xlsx` is just an example of a large library; any heavy dependency works the same way.)

## `React.lazy` for components

```jsx
import { lazy, Suspense, useState } from 'react'

const MarkdownEditor = lazy(() => import('./MarkdownEditor'))

function PostForm() {
  const [editing, setEditing] = useState(false)
  return (
    <>
      <button onClick={() => setEditing(true)}>Write post</button>
      {editing && (
        <Suspense fallback={<p>Loading editor…</p>}>
          <MarkdownEditor />
        </Suspense>
      )}
    </>
  )
}
```

- `lazy` takes a function returning a promise of a module with a **default export**
  that is a component.
- Call `lazy` at **module level**, not inside a component (otherwise each render creates
  a new lazy type and remounts).
- The first render of a lazy component suspends while the chunk downloads, so it needs a
  `<Suspense>` above it.

For named exports:

```jsx
const Chart = lazy(() => import('./charts').then(m => ({ default: m.RevenueChart })))
```

## Route-level splitting

Routes are the natural seams: users visit one page at a time.

```jsx
import { lazy, Suspense } from 'react'
import { Routes, Route } from 'react-router'
import Layout from './Layout'
import Home from './pages/Home'                       // keep the landing page eager

const Reports = lazy(() => import('./pages/Reports'))
const Settings = lazy(() => import('./pages/Settings'))
const Admin = lazy(() => import('./pages/Admin'))

export default function App() {
  return (
    <Routes>
      <Route element={<Layout />}>
        <Route index element={<Home />} />
        <Route path="reports" element={<Suspense fallback={<PageSpinner />}><Reports /></Suspense>} />
        <Route path="settings" element={<Suspense fallback={<PageSpinner />}><Settings /></Suspense>} />
        <Route path="admin/*" element={<Suspense fallback={<PageSpinner />}><Admin /></Suspense>} />
      </Route>
    </Routes>
  )
}
```

A single `<Suspense>` inside `Layout` around `<Outlet />` works too and avoids repeating
the fallback. React Router's data/framework modes also support a route-level `lazy`
property; see its docs if you use those modes.

## Preloading

Waiting for a chunk after a click adds latency. Start the download earlier:

```jsx
const loadReports = () => import('./pages/Reports')
const Reports = lazy(loadReports)

<Link to="/reports" onMouseEnter={loadReports} onFocus={loadReports}>Reports</Link>
```

The browser caches modules, so the `import()` inside `lazy` reuses the already-started
request. Hover-to-preload typically gives a head start of a couple of hundred
milliseconds — often enough to skip the spinner entirely.

## Avoiding fallback flashes on navigation

When navigating to a lazy route, the old page would normally be replaced by the fallback
while the chunk loads. Wrapping the navigation update in a transition keeps the old page
visible instead (React Router's navigation already uses transitions in recent versions;
Level 4 lesson 2 explains the mechanism).

## Worked example: analysing the bundle

1. Build: `npm run build`. Vite prints each output file with its size and gzip size.
2. Visualise: install `rollup-plugin-visualizer` as a dev dependency and add it to the
   plugins:

```js
// vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { visualizer } from 'rollup-plugin-visualizer'

export default defineConfig({
  plugins: [react(), visualizer({ filename: 'stats.html', gzipSize: true })],
})
```

3. Open `stats.html` after building. Look for: large libraries used on one page only,
   duplicate copies of a dependency, and locale/icon packs imported wholesale.

Typical fixes, in order of payoff:

- Lazy-load routes and rarely used heavy components (editors, charts, maps).
- Import only what you use (`import debounce from 'lodash-es/debounce'` rather than all
  of lodash; per-icon imports).
- Replace heavy dependencies with lighter ones or native APIs (`Intl` instead of date
  formatting libraries for simple cases).

Measure the effect on real metrics (Lighthouse, or field data if you have it), not just
bundle size.

## How It Actually Works

At build time, Rollup builds a module graph. A static `import` makes two modules part of
the same chunk (or shared chunks); a dynamic `import()` marks a **split point**: the
imported module and anything only it depends on go into a new chunk. The `import()`
call is rewritten to load that chunk's URL (with a content hash in the file name, so it
can be cached forever). Vite also emits `<link rel="modulepreload">` hints for chunks
needed on startup and preloads a dynamic chunk's own dependencies in parallel when it's
requested, avoiding a waterfall of imports.

`lazy(load)` returns a special component type holding a status: uninitialised → pending
→ resolved/rejected. On first render, React calls `load()`, stores the promise, and
**throws** it — which is the Suspense mechanism from the previous lesson. When the
promise resolves, React stores the module's default export and retries; from then on the
lazy type renders the real component synchronously. If the import fails (network error,
or a chunk deleted by a new deployment), the rejection is thrown during render and
reaches the nearest error boundary.

That deployment case is real: users with an old tab open request `Reports-abc123.js`
after you've deployed `Reports-def456.js`. Mitigate by keeping old assets available for
a while after deploys, and by having an error boundary that offers to reload the page.

## Common mistakes

- **Calling `lazy` inside a component** → remounts and re-suspends every render.
- **No `Suspense` above a lazy component** → an error (or the nearest unrelated boundary
  catches it and blanks a large area).
- **Splitting tiny components** → more requests, no benefit. Split big, rarely-needed
  things.
- **Lazy-loading the landing page's own content** → you've just added a spinner to the
  most important screen.
- **No error boundary for failed chunk loads.**

## Exercise

1. Take a multi-route app (the recipe finder works) and run `npm run build`; record the
   main bundle size.
2. Lazy-load every route except the landing page, and one heavy component loaded on
   interaction. Rebuild and record the new sizes.
3. Add hover/focus preloading to navigation links and observe in the Network tab that the
   chunk loads before the click.
4. Simulate a failed chunk (block the request in dev tools' Network "Block request URL")
   and make sure an error boundary shows a "Reload" option.
5. Add the visualizer and identify the three largest dependencies; propose one change for
   each.
