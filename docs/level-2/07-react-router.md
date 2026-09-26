---
description: "Add client-side routing with React Router: route definitions, nested layouts with Outlet, URL params, search params, navigation, and not-found pages."
---

# 07 · Routing with React Router

A single-page app still needs URLs: people bookmark pages, share links, and press Back.
**React Router** maps the browser's URL to components. This lesson uses React Router in
its classic client-side ("declarative"/"data") style with a Vite app. React Router can
also act as a full framework with server rendering (Level 4 touches on that).

Package naming changed between major versions. In React Router v7 the main package is
`react-router`; `react-router-dom` still exists and re-exports the same APIs, so older
tutorials using it still broadly apply. Check the version you install and its docs.

```bash
npm install react-router
```

## Defining routes

```jsx
// main.jsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { BrowserRouter, Routes, Route } from 'react-router'
import Layout from './Layout'
import Home from './pages/Home'
import Products from './pages/Products'
import ProductDetail from './pages/ProductDetail'
import NotFound from './pages/NotFound'

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <BrowserRouter>
      <Routes>
        <Route element={<Layout />}>
          <Route index element={<Home />} />
          <Route path="products" element={<Products />} />
          <Route path="products/:productId" element={<ProductDetail />} />
          <Route path="*" element={<NotFound />} />
        </Route>
      </Routes>
    </BrowserRouter>
  </StrictMode>,
)
```

- `BrowserRouter` uses the History API for clean URLs like `/products/42`.
- `index` marks the child route shown at the parent's exact path.
- `:productId` is a **dynamic segment**.
- `*` catches everything else — your 404 page.

## Layouts with `Outlet`

```jsx
// Layout.jsx
import { NavLink, Outlet } from 'react-router'

export default function Layout() {
  return (
    <>
      <header>
        <nav>
          <NavLink to="/" end>Home</NavLink>
          <NavLink to="/products">Products</NavLink>
        </nav>
      </header>
      <main>
        <Outlet />
      </main>
    </>
  )
}
```

`Outlet` is where the matching child route renders. The header stays mounted while
pages change underneath it. `NavLink` adds an `active` class (and `aria-current="page"`)
when its route matches; `end` stops `/` from matching every URL.

## Reading URL params

```jsx
import { useParams, Link } from 'react-router'

export default function ProductDetail() {
  const { productId } = useParams()        // always a string
  const product = products.find(p => p.id === Number(productId))

  if (!product) return <p>No product with id {productId}. <Link to="/products">Back to list</Link></p>
  return <h1>{product.name}</h1>
}
```

## Search params: state in the URL

Filters, sort order and pagination belong in the query string so they survive reloads
and can be shared:

```jsx
import { useSearchParams } from 'react-router'

export default function Products() {
  const [searchParams, setSearchParams] = useSearchParams()
  const category = searchParams.get('category') ?? 'all'
  const sort = searchParams.get('sort') ?? 'name'

  function update(key, value) {
    setSearchParams(prev => {
      const next = new URLSearchParams(prev)
      if (value === 'all') next.delete(key)
      else next.set(key, value)
      return next
    })
  }

  const visible = products
    .filter(p => category === 'all' || p.category === category)
    .toSorted((a, b) => (sort === 'price' ? a.price - b.price : a.name.localeCompare(b.name)))

  return (
    <>
      <select value={category} onChange={e => update('category', e.target.value)}>
        <option value="all">All</option>
        <option value="audio">Audio</option>
        <option value="storage">Storage</option>
      </select>
      <select value={sort} onChange={e => update('sort', e.target.value)}>
        <option value="name">Name</option>
        <option value="price">Price</option>
      </select>
      <ul>
        {visible.map(p => (
          <li key={p.id}><Link to={`/products/${p.id}`}>{p.name}</Link> — ₹{p.price}</li>
        ))}
      </ul>
    </>
  )
}
```

Here the URL **is** the state — no `useState` for filters at all. That's the single
source of truth principle applied to routing.

## Navigating in code

```jsx
import { useNavigate } from 'react-router'

function CheckoutButton({ cart }) {
  const navigate = useNavigate()
  async function handleCheckout() {
    const order = await placeOrder(cart)          // your API call
    navigate(`/orders/${order.id}`, { replace: true })
  }
  return <button onClick={handleCheckout}>Place order</button>
}
```

`replace: true` swaps the current history entry, so Back doesn't return to a checkout
that already completed. Use `<Link>` for anything the user clicks to go somewhere; use
`navigate` for redirects after actions.

## Worked example: protected routes

```jsx
import { Navigate, Outlet, useLocation } from 'react-router'
import { useAuth } from './auth-context' // from lesson 4

export function RequireAuth() {
  const { user } = useAuth()
  const location = useLocation()
  if (!user) return <Navigate to="/login" replace state={{ from: location }} />
  return <Outlet />
}

// in the route tree
<Route element={<RequireAuth />}>
  <Route path="account" element={<Account />} />
  <Route path="orders" element={<Orders />} />
</Route>
```

After login, read `location.state?.from?.pathname` and navigate back there. Remember
this only controls **what the UI shows**; the API must still reject unauthenticated
requests.

## Deploying a client-side router

With `BrowserRouter`, a user who reloads `/products/42` asks the **server** for that
path. Static hosts return 404 unless configured to serve `index.html` for unknown paths
(often called a "SPA fallback" or rewrite rule). Level 4's deployment lesson covers
this per host. GitHub Pages has no rewrite rules, which is why some apps use
`HashRouter` (`/#/products/42`) there.

## How It Actually Works

`BrowserRouter` subscribes to the browser's `popstate` event (fired on Back/Forward) and
keeps the current `location` in React state provided through context. `<Link>` renders a
real `<a href>` (so middle-click, "copy link" and accessibility work) but intercepts
normal clicks: it calls `event.preventDefault()` and then `history.pushState()` to change
the URL without a page load, and updates the router's location state.

That state change re-renders the `Routes` component, which **matches** the current path
against the route tree. Matching ranks routes by specificity — static segments beat
dynamic ones, which beat splats (`*`) — rather than by the order you wrote them. The
result is a list of matched routes from root to leaf; each level renders its `element`,
and each `Outlet` renders the next match down. `useParams` reads the params collected
during that match from context.

Because layouts are just parent routes, navigation between `/products` and
`/products/7` keeps `Layout` mounted (same type, same position) and only swaps what's
inside `Outlet` — its state survives, as reconciliation rules predict.

## Common mistakes

- **Using `<a href>` for internal links** → full page reloads that wipe all state. Use
  `<Link>`.
- **Forgetting params are strings**: `p.id === productId` fails when `id` is a number.
- **Duplicating URL state in `useState`** (filters in both the URL and state).
- **Missing the host rewrite** so deep links 404 in production.
- **Calling `navigate()` during render.** Use `<Navigate />` for render-time redirects
  and `navigate()` in handlers or effects.

## Exercise

Build a small "course catalog":

1. Routes: `/` (home), `/courses` (list), `/courses/:slug` (detail),
   `/courses/:slug/lessons/:n` (lesson, nested inside the course detail layout with a
   sidebar listing lessons), and a 404.
2. The course list supports `?level=beginner|advanced` and `?q=` search, both stored only
   in the URL.
3. The lesson page has Previous/Next links; Next on the last lesson navigates to a
   "course complete" page with `replace`.
4. Make `/account` protected with a fake login that stores the user in context, and
   return users to the page they came from after logging in.
5. Build with `npm run build`, serve with `npm run preview`, open a deep link directly
   and note what the preview server does for it.
